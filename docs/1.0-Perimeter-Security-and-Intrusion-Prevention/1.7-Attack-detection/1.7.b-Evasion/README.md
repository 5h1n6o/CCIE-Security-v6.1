---
layout: default
title: 1.7.b-Evasion
nav_order: 2
parent: 1.7-Attack-detection
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.7.b Evasion techniques

**回避技術（Evasion techniques）**とは、攻撃者がファイアウォール（FW）や侵入防止システム（IPS）などのセキュリティ対策を「検知されずに」すり抜けるために用いる手法の総称です。CCIE Security v6.1 ラボ試験では、これらの回避策を無効化するための**トラフィック正規化（Normalization）**、プロトコルインスペクション、およびパケット再構成の設定能力が重要となります。

---

## 📘 概要

*   **機能概要**: パケットの断片化（Fragmentation）、プロトコルの難読化（Obfuscation）、重複パケットの送出などを通じて、セキュリティデバイスのシグネチャマッチングを回避しようとする試みを特定し、阻止します。
*   **利用目的**: セキュリティポリシーの有効性を維持し、攻撃者が「検査エンジンには無害に見え、ターゲットホストでは有害に動作する」パケットを送り込むことを防ぎます。
*   **どのような場面で利用するか**: インターネット境界のNGFW、データセンター内のセグメント間、および重要なプロトコル（HTTP, DNS, SMTP）の保護。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な回避手法** | IP断片化、TCPセグメンテーション、IPオプション利用、プロトコルアノマリ。 |
| **防御の要** | **トラフィック正規化（Traffic Normalization）** と **パケット再構成**。 |
| **主な防御機能** | Fragment Reassembly、TCP State Check、Protocol Inspection、SSL Decryption。 |
| **メリット** | シグネチャベースの検知精度を極限まで高め、偽陰性（漏れ）を防ぐ。 |
| **デメリット** | 検査プロセスが増えるため、スループットの低下や遅延の要因となる。 |
| **対応機種** | Firepower (FTD), ASA, IOS-XE (ZBFW)。 |
| **設計上の注意** | ターゲットOSによってスタックの実装（パケット再構成の順序など）が異なる点。 |

---

## 🏗 動作原理

回避技術の多くは、セキュリティデバイスとターゲットホストの間での「解釈の差」を利用します。

```text
[ Attacker ] ── (Evaded Traffic: Overlapping Fragments) ──→ [ Security Device (FTD/ASA) ]
                                                                    │
    [ Mitigation Phase ]                                            │
    1. Fragment Reassembly: 断片を結合し、矛盾（重複）を排除。       │
    2. TCP Normalizer: シーケンス番号やウィンドウを正規化。             │
    3. Application Inspection: プロトコル準拠性を確認。                │
                                                                    ↓
[ Target Host ] <────── (Clean & Reconstructed Traffic) ────────────┘
```

---

## ⚙ 動作シーケンス

1.  **パケット着信**: 意図的に断片化された、またはIPオプションが付与されたパケットを受信します。
2.  **LINAエンジンによる正規化**: FTD/ASAの基盤エンジンが、IPフラグメントのオフセットを計算し、重複がある場合は先行するパケットを優先する等の処理を行います。
3.  **Snortによるディープインスペクション**: 正規化された（＝組み立て直された）ペイロードに対して、IPSシグネチャのスキャンを実行します。
4.  **プロトコル検証**: 不正なHTTPヘッダーや、標準外のコマンドシーケンスを検知し、アノマリとして処理します。
5.  **アクション適用**: 正規化に失敗したパケットや、組み立て後に攻撃が判明したパケットをドロップ（遮断）します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Fragmentation Attackの緩和**: ASA/FTDにおいて、フラグメントパケットの最大数やタイムアウト値を調整する設定（`fragment chain`等）。
*   **TCP Normalization**: 不要なTCPオプション（Window Scalingの悪用等）を削除したり、シーケンス番号の整合性をチェックする設定。
*   **IP Optionsの制御**: ラボ要件で「Security Policyを回避するために使用されるIPオプション付きパケットを破棄せよ」といった指示に対し、`inspect ip-options` や `access-list` での `options` 指定による対応。
*   **Custom Signatureの作成**: 標準のシグネチャでは検知できない特殊な文字列（例: `% Bad passwords`）を含む通信に対し、IPSで対応する設定。
*   **IPv6アノマリ対策**: `ipv6 spd mode aggressive` を使用し、不正なIPv6ヘッダーを持つパケットを積極的に破棄する設定。

---

## 🛠 設定方法

### 1. FTD/ASA: IPフラグメントの正規化設定
```bash
! ASA CLI例 (FTDではPlatform Settings経由)
fragment chain 24 outside
fragment timeout 5 outside
! 不正なパケットを組み立てずに破棄
fragment lookahead 100 outside
```

### 2. IOS-XE: ZBFWによるTCP正規化
```bash
parameter-map type inspect global
 max-incomplete low 80
 max-incomplete high 100
! TCPシーケンス番号のチェック
 tcp max-incomplete host 10 block-time 10
```

### 3. FTD: IPオプションパケットのドロップ (FlexConfig利用)
```bash
! ASA設定を模したFlexConfig
policy-map global_policy
 class inspection_default
  inspect ip-options
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **正規化によるドロップ確認** | <code>show asp drop</code> (原因コード: <code>tcp-normalizer</code> 等) |
| **フラグメント統計の表示** | <code>show fragment</code> |
| **Snortインスペクション状況** | <code>show snort statistics</code> |
| **パケット処理パスの追跡** | <code>packet-tracer input Inside tcp ... detailed</code> |
| **IPv6 CPU保護状態** | <code>show ipv6 spd</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| 正常なアプリ（NFS等）が断片化で落ちる | <code>fragment chain</code> の閾値不足 | 正常な通信の最大フラグメント数を確認し、閾値を引き上げる。 |
| 特定のWebサイトが表示されない | TCP MSS不一致または正規化の誤検知 | <code>sysopt connection tcpmss</code> の調整またはログで原因特定。 |
| 回避パケットが通過してしまう | Inspectionが有効になっていない | <code>service-policy</code> 下で該当プロトコルの <code>inspect</code> を有効化。 |
| IPSイベントに「Would have dropped」 | TAPモードまたはモニタ設定 | <code>Inline Set</code> の <code>Tap Mode</code> を解除する。 |

---

## ⚠ 制限事項

*   **パフォーマンスへの影響**: すべてのパケットを完全に再構成（Full Reassembly）する場合、メモリとCPUの消費が激しくなります。
*   **SSLの壁**: ペイロードを難読化（Obfuscation）して暗号化通信に隠された場合、SSL復号ポリシーがなければ正規化も検査も不可能です。
*   **非IPトラフィック**: レイヤー2レベルでの回避（VLAN Hopping等）は、L3正規化機能では防げず、ポートセキュリティ等のL2機能が必要です。

---

## 🔄 他技術との関連

*   **IPS (NGIPS)**: シグネチャマッチングの前に「デフラグメンテーション」や「プリプロセッサ」によってデータを整形します。
*   **uRPF**: 送信元IPのなりすまし（Spoofing）という回避技術に対するData Planeレベルの防御です。
*   **Advanced Malware Protection (AMP)**: ファイルのサンドボックス実行により、パケットレベルではなく「振る舞い」による回避検知を担当します。

---

## 🧩 比較表

### Evasion Method vs Mitigation Feature

| 回避手法 | 防御メカニズム | 設定箇所 |
| :--- | :--- | :--- |
| **Tiny Fragments** | IP Fragment Reassembly | Platform Settings / <code>fragment</code> |
| **Overlapping segments** | TCP Normalizer | <code>parameter-map</code> / Inspection |
| **Obfuscation (HTTP)** | Application Inspection | <code>inspect http</code> |
| **Payload Encryption** | SSL Decryption | SSL Policy (FMC) |
| **IP Source Routing** | Drop Options / Disable Routing | <code>no ip source-route</code> |

---

## 💡 ベストプラクティス

1.  **Strict Normalization**: デフォルト設定を過信せず、ターゲット環境に合わせてTCPシーケンス番号チェックやMSS固定を厳格化します。
2.  **IP Optionsを全破棄**: 現代のインターネットでIPオプションが必要なケースは極めて稀なため、セキュリティ境界では一括破棄（Drop IP Options）が推奨されます。
3.  **時刻同期（NTP）**: 回避試行があった際のログの順序性を保証するため、すべてのデバイスでNTPを同期させます。
4.  **L2/L3の併用**: ARP SpoofingやVLAN HoppingにはDAIやPort Securityを、フラグメントにはNGFWの正規化を用いる多層防御を構成します。

---

## 📝 ラボ学習・設定サンプル例

### 1. IPフラグメントパケットの再構成制限
*   **要件**: 外部からのフラグメントチェーンを最大10個、タイムアウト3秒に制限せよ。
*   **設定**: `fragment chain 10 outside`, `fragment timeout 3 outside`。

### 2. TCPシーケンス番号のランダム化無効化（デバッグ用）
*   **要件**: 特定のトラブルシュートのため、正規化機能の一部であるシーケンス番号ランダム化を無効にせよ。
*   **設定**: `policy-map` -> `class` -> `set connection random-sequence-number disable`。

### 3. 不正なIPオプションパケットの遮断
*   **要件**: IP Source Routingを悪用した通信を防御せよ。
*   **設定**: `access-list 101 permit ip any any options` -> `access-list 101 deny...` または `inspect ip-options`。

### 4. HTTP回避（難読化）のIPS検知
*   **要件**: HTTPヘッダーの異常なエンコーディングを検知せよ。
*   **設定**: FMCの `Intrusion Policy` 内で `HTTP Inspect` プリプロセッサを有効化。

### 5. IPv6 SPD Aggressiveモードの有効化
*   **要件**: 不正なIPv6パケットを高負荷時にアグレッシブにドロップせよ。
*   **設定**: `ipv6 spd mode aggressive`。

### 6. ASA脅威検知 (Threat Detection) によるスキャン阻止
*   **要件**: 回避を試みるポートスキャナを自動遮断せよ。
*   **設定**: `threat-detection rate scanning-threat threshold 500`。

### 7. DAI (Dynamic ARP Inspection) によるMiM防止
*   **要件**: ARP Spoofingによる回避を防止せよ。
*   **設定**: `ip arp inspection vlan [ID]`。

### 8. DHCP Snooping による偽DHCP防止
*   **要件**: Rogue DHCPサーバーによる通信リダイレクトを防げ。
*   **設定**: `ip dhcp snooping`。

### 9. 特定のTelnet文字列に対するIPSシグネチャ
*   **要件**: `cisco` という文字列を含む不正なTelnet応答を遮断せよ。
*   **設定**: 独自署名 `content: "cisco"` の作成。

### 10. ASA shun による動的ブロック
*   **要件**: 攻撃者のIPを即座に無効化せよ。
*   **コマンド**: `shun [Attacker_IP]`。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: 以下の設定がある環境で、攻撃者がIPフラグメントを用いてシグネチャマッチングを回避しようとした。この設定で防御可能か？
    `fragment chain 1 outside`
    *   **回答**: はい。チェーンを1に制限することで、断片化されたパケット自体を実質的に拒否（または極めて厳しく制限）するため、回避は困難になる。
2.  **トラブルシュート**: IPSを導入したが、フラグメントされたパケット内の攻撃を検知できない。FTDのインターフェイス設定で確認すべきことは？
    *   **回答**: `Inline Set` 設定においてパケット再構成が有効か、または `Pre-filter Policy` で `Fast-path` されていないか。
3.  **Design**: セキュリティデバイスの負荷を抑えつつ、TCPセグメンテーションによる回避を防ぐ最適な手法は？
    *   **回答**: TCP MSS（Maximum Segment Size）の調整により、極端に小さなセグメントを排除する。
4.  **実装**: 特定のプロトコル（例: HTTP）のみに対して、詳細な正規化とアノマリ検知を有効にする手順を述べよ。
5.  **コンフィグ読解**: `show asp drop` において `tcp-seq-syn-fin` でのドロップが増えている。これはどのような回避攻撃を阻止しているか？
    *   **回答**: TCPフラグの異常な組み合わせ（SYNとFINが同時など）を用いた、プロトコルスタックのアノマリ攻撃。

---

## 🔗 参考リソース

*   **Cisco Configuration Guides**
    *   [Cisco ASA Series Firewall Directory - Inspection of IP Options](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/firewall/asa-94-firewall-config/inspect-basic.html#ID-2043-0000030a)
*   **Cisco Live (BRKSEC-3020)**
    *   [Troubleshooting Firewall Threat Defense (FTD)](https://www.ciscolive.com/on-demand/on-demand-library.html)
*   **Technical Notes**
    *   [Understanding TCP Normalization on Cisco ASA](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/113685-asa-threat-detection-00.html)

---

## 📝 **補足（Notes）**  

*   **学習メモ**: 回避技術への対策は「防御（Prevent）」ではなく「正規化（Normalize）」という単語で検索・理解するとスムーズです。
*   **図解**: パケットが「パズルのピース」のようにバラバラに届くのを、セキュリティデバイスが「一度完成させてから中身を見る」イメージを持ってください。
*   **注意点**: ラボ試験では、正規化機能を強化しすぎると正規の試験用端末（Test PC）の通信まで止めてしまうことがあります。不必要なドロップが発生していないか、常に `show asp drop` を確認する癖をつけましょう。
