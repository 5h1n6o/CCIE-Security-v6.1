---
layout: default
title: 1.5.c-TAP
nav_order: 3
parent: 1.5-NGIPS-deployment-modes
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.5.c TAP

Cisco Next-Generation IPS (NGIPS) における**TAPモード（Inline Tap）**は、インライン（In-line）構成の利点（トラフィックパスへの直接配置）と、パッシブ（Passive）構成の利点（通信への影響ゼロ）を組み合わせた特殊な展開モードです。デバイスは物理的にネットワークパス上に配置されますが、パケットをコピーして検査エンジン（Snort）に送る一方で、オリジナルのパケットはそのまま通過させます,。

---

## 📘 概要

*   **機能概要**: インラインインターフェイスペア（Inline Set）において、パケットのコピーを作成して分析を行い、脅威を検知しても**パケットの破棄（ドロップ）を行わない**モードです,。
*   **利用目的**: 本番環境のネットワークにおいて、IPSポリシーが実際の通信にどのような影響を与えるかを、通信断のリスクなしに評価（テスト）するために使用されます,。
*   **利用場所**: 新規IPSシグネチャの導入時、IPSの試験運用期間、またはネットワークの遅延や可用性が最優先され、検知のみが求められる環境で利用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | 物理的にはインラインだが、論理的にはパッシブ（検知のみ）として動作する。 |
| **用途** | ポリシーの検証（チューニング）、IPS導入前のベースライン確認。 |
| **メリット** | インライン構成への移行が容易（チェックボックス一つで変更可能）。誤検知による遮断リスクがない。 |
| **デメリット** | 攻撃をリアルタイムに阻止できない。物理故障時は通信断の原因になり得る（バイパスなしの場合）。 |
| **対応機種** | Firepower Threat Defense (FTD), Firepower センサー。 |
| **制限事項** | **Tap Mode** は Inline Set オブジェクト内でのみ有効化できる。 |
| **設計上の注意点** | インラインパスにあるため、デバイスの最大スループット制限の影響を受ける。 |

---

## 🏗 動作原理

TAPモードでは、イングレスインターフェイスに到着したパケットは即座にエグレスインターフェイスへ転送されると同時に、内部でコピーが作成されます。

```text
[ External Network ]
         ↓
[ Ingress Interface ] ── (Original Packet) ─→ [ Egress Interface ] ─→ [ Internal ]
         │                                            ↑
         └─ (Copy of Packet) ──→ [ Snort Engine ] ────┘
                                        ↓
                         [ Verdict: Drop? -> Alert only ]
```

Snortエンジンが「Drop」と判定した場合でも、オリジナルのパケットは既に通過しているため、FMCには「Would have dropped（ドロップしたであろう）」というイベントが記録されるのみです,。

---

## ⚙ 動作シーケンス

1.  **物理受信**: パケットがインラインペアの一方のインターフェイスに到着します。
2.  **ハードウェアバイパス/転送**: パケットは即座に対向のインターフェイスへ転送（Forward）されます。
3.  **パケットの複製**: 内部バスレベルでパケットがコピーされ、Snort プロセスに渡されます。
4.  **IPS インスペクション**: 定義された **Intrusion Policy** に基づき、シグネチャとの照合が行われます,。
5.  **判定の生成**: 
    *   攻撃を検知した場合、アクションが `Drop` に設定されていても、パケットは既に通過済みであるため、破棄は行われません。
    *   FMCに対して「Would have dropped」フラグが付いたアラートを送信します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Blueprintで重要なポイント**: ラボ試験では「トラフィックをインラインで配置しつつ、本番通信を遮断せずに攻撃を可視化せよ」という要件で出題されます。これは **Passive モード（SPAN）ではなく、Inline Set 内の Tap Mode** を指しています,。
*   **ラボ試験で設定させられそうな内容**:
    *   FMC GUI で **Inline Set** を作成し、そのプロパティで `Tap Mode` を有効にする手順。
    *   TAPモードで出力されたイベント（Analysis > Intrusion > Events）を確認し、どのルールがヒットしたかを特定する問題。
*   **よくある設定ミス**:
    *   インターフェイスを `None` モードに設定し忘れる（Routed や Transparent モードでは Inline Set を作成できません）,。
    *   Inline Set を作成したが、**Access Control Policy (ACP)** でそのゾーンを使用し、かつ `Intrusion Policy` を適用した `Allow` ルールを作成していない。
*   **showコマンドから状態を判断**: 
    *   `show inline-set` コマンドで、対象のペアに `tap-mode: enabled` が表示されているかを確認します。

---

## 🛠 設定方法

### 1. インターフェイスの準備 (FMC)
1.  **Devices > Device Management** で対象 FTD を編集。
2.  2つのインターフェイス（例: Eth1/1, Eth1/2）を選択し、**Mode** を `None` に設定。
3.  `Security Zone` を作成し、両方のインターフェイスを同じゾーン（例: `ZONE-TAP`）に所属させます。

### 2. Inline Set の作成と TAP モード有効化
1.  同じ編集画面の **Interfaces** タブから **Inline Sets** セクションへ移動。
2.  **Add Inline Set** をクリック。
    *   **Name**: `TAP-SET-01`
    *   **Interfaces**: `Eth1/1` と `Eth1/2` を選択。
    *   **Tap Mode**: **チェックを入れる**。
3.  `Propagate Link State` を任意で有効にし、保存します。

### 3. ポリシーのデプロイ
1.  **Policies > Access Control** で、`ZONE-TAP` からの通信を許可するルールを作成。
2.  **Inspection** タブで `Intrusion Policy`（例: `Balanced Security and Connectivity`）を適用。
3.  **Deploy** を実行します。

---

## 🔍 検証コマンド

| 目的 | コマンド（FTD CLI） |
| :--- | :--- |
| **Inline Set の状態確認** | <code>show inline-set</code> |
| **TAPモードの有効化確認** | <code>show running-config inline-set</code> |
| **Snort による検知統計** | <code>show snort statistics</code> |
| **パケット処理プロセスの確認** | <code>system support firewall-engine-debug</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 攻撃が検知されない | ACPでIPSが適用されていない | FMCのACPルール設定で <code>Intrusion Policy</code> が紐付いているか確認。 |
| 攻撃を検知するが遮断される | TAPモードが未有効 | <code>show inline-set</code> で <code>Tap Mode: disabled</code> になっていないか確認。 |
| 通信が全く通らない | 物理結線の不備 | インターフェイスの物理ステータスと、Inline Set のペア設定が正しいか確認。 |
| アラートがFMCに届かない | SFTunnel の異常 | <code>show sftunnel status</code> で管理トンネルの稼働状況を確認。 |

---

## ⚠ 制限事項

*   **遮断機能の喪失**: TAPモードが有効な限り、いかなる重大な攻撃もシステムは自動阻止できません。
*   **ハードウェア依存**: インライン配置であるため、デバイスがハングした場合、ハードウェアバイパス回路（Fail-to-wire）がないモデルでは通信が止まります。
*   **SSL復号**: 暗号化トラフィックは SSL Policy で復号しない限り、Snort による中身の検査は行えません。

---

## 🔄 他技術との関連

*   **Access Control**: ルールのアクションを `Allow` に設定し、IPS検査へトラフィックを誘導する必要があります,。
*   **Intrusion Policy**: TAPモードで実行される実際の検査ルール（シグネチャ）を定義します。
*   **High Availability**: フェイルオーバー構成でも TAP モードの設定は引き継がれます。
*   **Bypass**: 物理的な障害時に通信を維持するためのバイパス機能と組み合わせて設計されます。

---

## 🧩 比較表

### In-line vs Passive vs TAP

| 比較項目 | In-line (Normal) | Passive (SPAN) | Inline TAP (推奨テスト用) |
| :--- | :--- | :--- | :--- |
| **配置** | パス上（直列） | パス外（並列） | パス上（直列） |
| **遮断能力** | あり | なし（TCP Resetのみ） | なし |
| **導入リスク** | 高（通信断の可能性） | 低（影響なし） | 中（故障時リスクあり） |
| **検知イベント** | Drop / Alert | Alert | Would have dropped / Alert |

---

## 💡 ベストプラクティス

1.  **移行ステップとしての活用**: 運用開始時はまず TAP モードで数週間稼働させ、誤検知（False Positive）を排除してから TAP モードを解除（インライン遮断へ移行）します,。
2.  **バイパスインターフェイスの利用**: TAPモードであっても物理的な単一障害点になるため、可能な限りハードウェアバイパス機能を持つ NIC を使用します。
3.  **イベントのフィルタリング**: 「Would have dropped」イベントを重点的に監視し、遮断に切り替えた際の影響範囲を正確に特定します。

---

## 📝 ラボ学習・設定サンプル例

※以下の例は FMC 7.x 環境を想定しています。

### 1. 基本的な Inline TAP セットの設定
*   **要件**: Eth1/1 と Eth1/2 を使用し、通信を遮断せずに検知のみを行う構成にせよ。
*   **設定**: 
    1. インターフェイス Mode を `None` に設定。
    2. **Inline Set** を作成し、`Tap Mode` を ON。

### 2. TAPモードでの IPS アラート確認
*   **課題**: TAPモード稼働中に、`Analysis > Intrusion > Events` を確認し、`Inline Result` カラムに `Would have dropped` と表示されることを確認せよ。

### 3. 特定プロトコルのみの監視
*   **要件**: Web (HTTP) トラフィックのみを TAP モードで監視し、他は検査なしで通過させよ。
*   **設定**: ACP ルールで HTTP トラフィックのみを `Allow` + `Intrusion Policy` とし、他は `Trust` アクションを設定。

### 4. リンクダウンの伝搬設定
*   **要件**: TAPモード構成において、片側のスイッチ障害をもう一方のスイッチに伝えよ。
*   **設定**: Inline Set プロパティで `Propagate Link State` を有効化。

### 5. 独自シグネチャのテスト
*   **課題**: 自作のシグネチャを TAP モードで適用し、実際の通信を止めずに検知できるか検証せよ。

### 6. Snort 3 への切り替えと動作確認
*   **要件**: Snort 3 エンジンを使用して TAP モードを構成せよ。

### 7. MTU サイズの調整
*   **要件**: ジャンボフレームを扱う環境で TAP モードを構成せよ。
*   **設定**: インターフェイス設定で MTU を `9000` 等に変更。

### 8. VTI トンネルとの併用
*   **構成**: 仮想トンネルインターフェイスを通るパケットを TAP モードで検査する論理構成。

### 9. デバッグによるパス確認
*   **コマンド**: `system support firewall-engine-debug` を実行し、パケットが Snort に複製されている様子を追跡。

### 10. TAPモードからインライン遮断への切り替え
*   **課題**: 検証完了後、設定変更なしで遮断モードへ移行せよ。
*   **操作**: Inline Set の `Tap Mode` チェックを外して再デプロイ。

---

## ❓ 想定試験問題

1.  **Design**: IPSを導入したいが、既存のルーティング構成を変更できず、かつ誤検知による通信断が許されない。最適な展開モードは？
    *   **回答**: TAP モード（Inline Tap）。
2.  **Troubleshoot**: TAPモードを使用しているのに `Drop` イベントが発生している。原因は？
    *   **回答**: 設定ミスにより `Tap Mode` が有効になっていないか、あるいは `Pre-Filter Policy` など IPS 以外の箇所でドロップされている。
3.  **コンフィグ読解**: `show inline-set` の出力で、特定のペアにのみ `tap: enabled` がある場合、そのペアを通過する攻撃トラフィックはどうなるか？
    *   **回答**: 遮断されずに通過するが、FMC に「ドロップされたはず」の記録が残る。
4.  **実装**: Inline Set を作成する際、インターフェイスが選択肢に現れない理由を述べよ。
    *   **回答**: インターフェイスのモードが `Routed` や `Transparent` になっており、`None` に設定されていないため。
5.  **動作シーケンス**: TAPモードにおけるパケットの複製は、どのエンジンの段階で行われるか？
    *   **回答**: LINA エンジンがパケットを受け取った直後のデータ収集（DAQ）レイヤ周辺。

---

## 🔗 参考リソース

*   **Cisco Configuration Guides**
    *   [Cisco Secure Firewall Management Center Administration Guide, 7.1 - Inline Sets](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/interfaces-settings-inline-sets.html)
*   **Technical Notes**
    *   [Firepower Deployment Modes: In-Line, Passive, and Tap](https://www.cisco.com/c/en/us/support/docs/security/firepower-management-center/212356-understand-firepower-deployment-modes.html)
*   **Cisco Live**
    *   BRKSEC-2021: Firepower Threat Defense - Packet Flow and Troubleshooting

---

## 📝 **補足（Notes）**

*   **学習メモ**: インライン「TAP」は、物理的に回線を切断してデバイスを挟むため、SPANを用いた「パッシブ」とは物理トポロジが全く異なります。ラボ試験の要件で「Physically in-path」か「Out-of-band」かを読み分けることが重要です。
*   **注意点**: TAPモードを解除してインライン遮断に移行する際、それまで検知のみだったルールが突然パケットを捨て始めるため、移行直後は注意深い監視が必要です。
