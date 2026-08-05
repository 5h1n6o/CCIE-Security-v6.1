---
layout: default
title: 1.2.e-Traffic-redirection
nav_order: 5
parent: 1.2-Firewall-features-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.2.e Traffic redirection to service modules

Cisco ASAおよびFirepower Threat Defense (FTD) における**サービスモジュールへのトラフィックリダイレクション**は、ファイアウォールを通過するパケットを、高度な検査（IPS、AMP、URLフィルタリング等）を行う専用エンジンやハードウェアモジュールへ転送する技術です。CCIE Securityラボ試験では、特にASAとFirepowerサービスモジュール（SFR）間の連携、およびFTD内部でのLINAエンジンからSnortエンジンへのパケット受け渡し（リダイレクション）の理解と設定が問われます。

---

# 📘 概要

*   **機能概要**: ファイアウォールエンジン（L3/L4処理）が受信したトラフィックを、特定のルールに基づいて次世代セキュリティサービス（L7処理）を担当するサービスモジュールへ「リダイレクト」します。
*   **利用目的**: 
    *   **ASA with Firepower Services**: 既存のASAにFirepowerの次世代機能を統合するため。
    *   **インスペクションの分離**: 高負荷なディープパケットインスペクション（DPI）を別プロセスまたは別モジュールで処理させ、デバイスの安定性を確保。
*   **主な方式**: ASAではModular Policy Framework (MPF) を使用して `sfr` モジュールへリダイレクトします。FTDでは、アーキテクチャ上、LINA（ASAベースエンジン）からSnortへ自動的に、あるいはポリシーに基づきリダイレクトされます,。

---

# 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **リダイレクト方式** | ASA: MPF (`sfr` コマンド)。FTD: 内部的なデータ収集 (DAQ) レイヤによる転送。 |
| **動作モード** | **Inline (Fail-open/Fail-close)** および **Monitor-only**。 |
| **主なサービス** | IPS, AVC (Application Visibility and Control), URL Filtering, AMP (Advanced Malware Protection)。 |
| **転送判定** | ACL（クラスマップ）に一致するトラフィックのみをモジュールへ送る。 |
| **制限事項** | モジュールがダウンしている際の振る舞い（通過させるか遮断するか）の設計。 |
| **対応ハードウェア** | ASA 5500-X (SFRモジュール内蔵)、Firepower 4100/9300 (セキュリティモジュール)。 |

---

# 🏗 動作原理

ASAを例にとると、パケットはまずASAの基本エンジン（LINA）で処理され、その後Firepowerモジュールへ「横流し」されます。

```text
Incoming Packet
      ↓
[ ASA LINA Engine ] --- (L3/L4 ACL, NAT, Conn Lookup)
      ↓
[ Modular Policy Framework (MPF) ]
      ↓  (Redirect via 'sfr' command)
[ Firepower Service Module (SFR) ] --- (Snort Engine: IPS, AVC, URL, AMP)
      ↓  (Verdict: Permit/Drop)
[ ASA LINA Engine ] --- (Verdictに従い転送または破棄)
      ↓
Outgoing Packet
```

パケットトレーサーで検証すると、フェーズの中に **「Module information for forward flow: packet dispatched to next module」** というログが表示されます。

---

# ⚙ 動作シーケンス

1.  **パケット受信**: 物理インターフェイスでパケットを受信。
2.  **基本検査**: ASA/LINAエンジンが既存コネクションの確認、ACL、NATを適用。
3.  **クラス分類**: `class-map` に基づき、Firepowerで検査すべきトラフィックか判断。
4.  **リダイレクション評価**: `policy-map` 内の `sfr` 命令を実行。
5.  **モジュール内インスペクション**: SFRモジュール（Snort）がL7レベルでスキャン。
6.  **判定の返却**:
    *   **Inline (Fail-close)**: モジュールがダウンまたは破棄判定ならパケットをドロップ。
    *   **Inline (Fail-open)**: モジュールがダウンしていてもパケットを通過させる。
    *   **Monitor-only**: 判定に関わらずコピーのみを送り、ASA側は即座に転送。

---

# 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **リダイレクト対象の絞り込み**: 全トラフィックをリダイレクトするとパフォーマンスが低下するため、「HTTP/HTTPSのみ」や「特定のサブネットのみ」を `class-map` で指定する能力が問われます。
*   **Fail-modeの設定**: 「モジュール故障時でも通信を維持せよ（Fail-open）」または「セキュリティを優先せよ（Fail-close）」という要件の書き分け。

### ラボ試験で設定させられそうな内容
*   **ASAでのFirepowerリダイレクト**: `sfr {fail-open | fail-close} [monitor-only]` のCLI設定。
*   **FTDでのインスペクションバイパス**: 高負荷時に特定のトラフィックをSnortに送らずバイパスさせる（Intelligent Traffic Management）概念。

### よくある設定ミス
*   **クラスマップの不一致**: `match any` を忘れて、意図したトラフィックがFirepowerに届かない。
*   **管理通信の欠如**: モジュール自体がFMCと通信できていないため、リダイレクトは成功しても検査が行われない。

### showコマンド/debugログの読み取り
*   `show module sfr details`: SFRモジュールの動作ステータス（Up/Down）を確認。
*   `show service-policy sfr`: どれだけのパケットがリダイレクトされたかの統計を確認。
*   `packet-tracer`: モジュールへのディスパッチが発生しているかを確認。

---

# 🛠 設定方法

### ASA (CLI) - Firepowerモジュールへのリダイレクト設定
```bash
# 1. 検査対象の定義
access-list FIREPOWER_TRAFFIC extended permit ip 192.168.1.0 255.255.255.0 any

# 2. クラスマップの作成
class-map SFR_CLASS
 match access-list FIREPOWER_TRAFFIC

# 3. ポリシーマップでリダイレクトを指示
policy-map global_policy
 class SFR_CLASS
  sfr fail-open  # モジュールが死んでも通信を維持

# 4. サービスポリシーの適用
service-policy global_policy global
```

### FTD (FMC管理)
FTDではSnortエンジンへのリダイレクトは「Access Control Policy」と「Intrusion Policy」の紐付けによって自動的に行われます。
1.  **Policies > Access Control**: 対象ルールを編集。
2.  **Inspectionタブ**: Intrusion Policy (IPS) や Variable Set を選択。
3.  **Loggingタブ**: ログを有効にすることで Snort へのリダイレクトを確認可能。

---

# 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **モジュール状態の確認** | <code>show module sfr</code> |
| **統計情報の表示** | <code>show service-policy sfr</code> |
| **パケットフローの検証** | <code>packet-tracer input inside tcp 192.168.1.1 1234 8.8.8.8 80</code> |
| **Snortプロセス確認(FTD)** | <code>system support engine-status</code> |
| **リダイレクト状況確認** | <code>show asp table classify</code> |

---

# 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 通信が一切通らなくなった | モジュールがダウンし Fail-close 設定 | <code>show module sfr</code> で確認。一時的に <code>fail-open</code> へ変更。 |
| SFRモジュールの統計が増えない | MPFのクラスマップが一致していない | ACLの設定と、<code>match</code> 文の整合性を <code>show run class-map</code> で確認。 |
| Firepower側のイベントが空 | モジュールとFMCの通信断 | SFRモジュールの管理IP疎通を <code>session sfr</code> から確認。 |
| パケットがバイパスされる | 特定のプロトコルが非サポート | <code>show service-policy sfr</code> のドロップ/バイパスカウンタを確認。 |

---

# ⚠ 制限事項

*   **ASAシングルモード限定**: 旧バージョンではマルチコンテキストモードでのSFRリダイレクトに厳しい制限がありました（現在は一部緩和されていますが、RA-VPNとの併用などに注意が必要です）。
*   **パフォーマンス制限**: サービスモジュールへリダイレクトするパケット量に応じて、デバイス全体の最大スループットが低下します。
*   **EtherType**: 非IPトラフィックのリダイレクトはサポートされません。
*   **暗号化**: SSL/TLSトラフィックは、リダイレクト前にASAまたはFirepower側で復号（SSL Decryption）しない限り、ディープインスペクションができません。

---

# 🔄 他技術との関連

*   **Access Control**: ACLで許可されたパケットのみが、次のフェーズとしてサービスモジュールへリダイレクトされます。
*   **IPS**: リダイレクトされた先のSnortエンジンで実行される主要なセキュリティ機能です,。
*   **Modular Policy Framework (MPF)**: ASAにおけるリダイレクションのインフラストラクチャそのものです。
*   **Management Plane Protection**: サービスモジュール自体の管理用通信を保護する必要があります。

---

# 🧩 比較表

### Inline vs Monitor-only (SFR)

| 機能 | Inline Mode (デフォルト) | Monitor-only |
| :--- | :--- | :--- |
| **パケット処理** | パケットをモジュールへ転送し、結果を待つ。 | パケットのコピーをモジュールへ送る。 |
| **遮断（Drop）** | 可能。 | **不可（検知のみ）**。 |
| **レイテンシ** | 検査時間分、わずかに増加。 | 最小限（ASAは即座にパケットを送出）。 |
| **用途** | 本番の保護（IPS）。 | 導入初期の動作テスト、IDS。 |

---

# 💡 ベストプラクティス

*   **段階的な導入**: 最初は `monitor-only` でリダイレクトを開始し、誤検知（False Positive）がないことを確認してから Inline に切り替えます。
*   **重要トラフィックの選別**: OSPFやBGPなどのルーティングプロトコルは、サービスモジュールへリダイレクトせずにバイパスさせることで、モジュール負荷によるネイバー断を防ぎます。
*   **Fail-open の検討**: 可用性が最優先のネットワークでは、モジュール障害による全遮断を避けるため、原則 `fail-open` を設定します。

---

# 📝 ラボ学習・設定サンプル例

### 1. 全IPトラフィックのリダイレクト (ASA)
*   **要件**: InsideからOutsideへの全パケットをFirepowerで検査せよ。モジュール停止時は通信を遮断せよ。
*   **設定**: `policy-map global_policy; class inspection_default; sfr fail-close`

### 2. 特定ポートのみのリダイレクト (ASA)
*   **要件**: HTTP (TCP 80) 通信のみを検査対象にせよ。
*   **設定**: `access-list SFR_HTTP permit tcp any any eq 80` → クラスマップでこれを指定。

### 3. モニターモードの設定
*   **要件**: 既存の通信に影響を与えず、Firepowerでトラフィックを分析せよ。
*   **設定**: `sfr fail-open monitor-only`

### 4. 故障時バイパス（Fail-open）の設定
*   **要件**: モジュールがリブート中もユーザー通信を維持せよ。
*   **設定**: `sfr fail-open`

### 5. インターフェイス個別ポリシー
*   **要件**: Insideインターフェイスからの着信トラフィックのみをリダイレクトせよ。
*   **設定**: `service-policy global_policy interface inside`

### 6. リダイレクトからの特定ホスト除外
*   **要件**: 管理者端末 (10.1.1.100) は検査を通さず高速に通信させよ。
*   **設定**: ACLの最初で `deny ip host 10.1.1.100 any` を入れ、その後に許可を入れる。

### 7. packet-tracer によるリダイレクト確認
*   **課題**: 設定がSFRにパケットを送っているか証明せよ。
*   **確認**: `packet-tracer` の出力で `sfr` フェーズが `ALLOW` かつモジュールへディスパッチされていることを確認。

### 8. FTD 内部Snortリダイレクトの確認
*   **課題**: Snortがパケットを処理しているかステータスを確認せよ。
*   **コマンド**: FTD CLIで `show snort statistics` を実行。

### 9. サービスプロセッサとの通信設定
*   **要件**: Firepowerモジュールの管理IPを設定せよ。
*   **コマンド**: `session sfr` でログインし、`setup` コマンドを実行。

### 10. クラスタリング環境でのリダイレクト
*   **注意**: 複数のASAユニットでサービスモジュールのステートを一貫させるためのHA設定。

---

# ❓ 想定試験問題

1.  **コンフィグ読解**: ASAのコンフィグに `sfr fail-open monitor-only` とある。モジュールがパケットを攻撃と判定した場合、パケットは破棄されるか？
    *   **正解**: 破棄されない。`monitor-only` モードはコピーを検査するだけであり、ASA本体はパケットをそのまま転送する。
2.  **トラブルシュート**: ASAの `show service-policy sfr` でパケットヒットが増えていない。ACLに誤りがない場合、次に確認すべきポイントは？
    *   **正解**: クラスマップがポリシーマップ内で正しく定義されているか、およびそのポリシーが適切なインターフェイス（または `global`）に `service-policy` コマンドで適用されているかを確認する。
3.  **Design**: 冗長化したASAにおいて、モジュール障害がネットワーク全体の障害（Black hole）になるのを防ぐためのコマンドは？
    *   **正解**: `sfr fail-open`。
4.  **実装**: ASAの特定のコンテキストでFirepowerサービスを有効にする手順は？
    *   **正解**: システム実行スペースからモジュールにインターフェイスを割り当て、各コンテキスト内でMPF設定を行う。
5.  **動作シーケンス**: ASAにおいてNATとSFRリダイレクションはどちらが先に行われるか？
    *   **正解**: 一般的にNAT（Untranslate/Translate）が先に行われ、パケットが転送される直前のMPFフェーズでリダイレクションが行われる。

---

# 🔗 参考リソース

*   **Cisco Live動画/スライド**:
    *   [BRKSEC-2021: Firepower Threat Defense - Packet Flow and Troubleshooting](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-2021)
*   **Configuration Guide**:
    *   [Cisco ASA Series Firewall CLI Configuration Guide, 9.4 - ASA Firepower Services](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/firewall/asa-94-firewall-config/modules-sfr.html)
*   **Command Reference**:
    *   [Cisco ASA Series Command Reference - sfr](https://www.cisco.com/c/en/us/td/docs/security/asa/command-reference/s/cmdref2/s7.html)
*   **Technical Notes**:
    *   [Cisco ASA Firepower Module Configuration and Troubleshooting (Cisco Support)](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/118121-config-asa-00.html)
*   **White Paper**:
    *   Cisco Next-Generation Security Solutions: Integrating Firepower with ASA  

---

📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

---
