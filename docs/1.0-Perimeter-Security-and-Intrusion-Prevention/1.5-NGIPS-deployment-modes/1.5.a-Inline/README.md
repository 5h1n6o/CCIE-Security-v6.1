---
layout: default
title: 1.5.a-Inline
nav_order: 1
parent: 1.5-NGIPS-deployment-modes
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.5 Cisco NGIPS deployment modes
# 1.5.a In-line

Cisco Next-Generation IPS (NGIPS) における**インライン（In-line）展開モード**は、デバイスをネットワークのトラフィックパスに直接挿入し、全てのパケットが物理的または論理的にデバイスを通過するように構成するモードです。このモードの最大の特徴は、脅威を検知するだけでなく、**リアルタイムで悪意のあるトラフィックを遮断（Drop/Block）できる**点にあります。

---

## 📘 概要

*   **機能概要**: ネットワークデバイス（Firepower センサーまたは FTD）を 2 つのインターフェイス（またはペア）の間に配置し、ブリッジのように動作させます。通過するトラフィックは Snort インスペクションエンジンによって精査されます。
*   **利用目的**: 不正侵入、マルウェア、脆弱性を突く攻撃を動的に阻止し、内部ネットワークを保護します。
*   **どのような場面で利用するか**: インターネット境界、データセンターのセグメント間、機密情報が存在する VLAN 間など、**能動的な防御が必要な箇所**で利用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な特徴** | トラフィックパス上に配置され、攻撃パケットを破棄可能。 |
| **用途** | 侵入防止 (Prevention)、リアルタイム防御、コンプライアンス維持。 |
| **メリット** | 脅威の即時遮断。外部遮断デバイス（ACL等）との連携なしで自己完結。 |
| **デメリット** | デバイス故障が通信断に直結する。インスペクションによる遅延が発生。 |
| **対応機種** | Cisco Firepower (FTD) センサー、ASA with Firepower Services。 |
| **構成要素** | **Inline Set**（インターフェイスペア）、バイパス回路、タップモード。 |
| **設計上の注意** | **Fail-Safe**（故障時の挙動）と、スループット制限の考慮が必須。 |

---

## 🏗 動作原理

インラインモードでは、パケットはイングレス（着信）インターフェイスから入り、セキュリティエンジンで検査された後、エグレス（発信）インターフェイスから送出されます。

```text
[ External Network ]
         ↓
[ Ingress Interface (Eth1/1) ]
         ↓
[ Snort Inspection Engine ] ↔ [ IPS Policy / Signatures ]
         ↓ (Verdict: Allow)
[ Egress Interface (Eth1/2) ]
         ↓
[ Internal Network ]
```

もし Snort エンジンが攻撃を検知した場合、そのパケットはエグレス側に転送されずに破棄されます。

---

## ⚙ 動作シーケンス

1.  **パケット受信**: 物理層でパケットを受信し、データ収集 (DAQ) レイヤに渡されます。
2.  **前処理 (Pre-processing)**: 正規化、デフラグメンテーション、プロトコルデコードが行われます。
3.  **インスペクション**: 定義された **Intrusion Policy** に基づき、Snort シグネチャとの照合が行われます。
4.  **判定 (Verdict)**:
    *   **Permit**: パケットをペアとなるインターフェイスへ送出。
    *   **Drop**: パケットを破棄し、FMC へイベントを送信。
5.  **Fail-Open / Close 評価**: ハードウェア障害や Snort プロセスのハング時、あらかじめ設定されたバイパス挙動に従います。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### Blueprint で重要なポイント
*   **Inline Set の構成**: FMC 上で 2 つの物理インターフェイスをペアリングし、`Inline Set` オブジェクトを作成する手順。
*   **Fail-Safe オプション**: `Propagate Link State` や `Bypass` 設定の有無による挙動の違いの理解。
*   **Action の書き分け**: ラボ要件で「検知のみ（IDS）」か「遮断（IPS）」かを読み取り、ポリシーの Action を `Alert` または `Drop and Generate Events` に設定する。

### ラボ試験で設定させられそうな内容
*   **物理インターフェイスのインライン化**: FTD を Routed/Transparent ではなく、L2 的な IPS センサーとして動作させる。
*   **タップモードの併用**: インライン構成でありながら、パケットのコピーを検査し遮断を行わない設定（デバッグや移行期の設定）。
*   **シグネチャのカスタマイズ**: 特定の文字列（例: `% Bad passwords`）を含む通信をインラインでブロックする独自シグネチャの作成。

### よくある設定ミス
*   **インターフェイスモードの不一致**: インターフェイスが `None` モード以外（Routed 等）になっていると Inline Set に追加できません。
*   **バイパス設定の漏れ**: 電源断時に通信を維持する（Fail-to-wire）設定を行わず、ネットワークが遮断される。

---

## 🛠 設定方法

### 1. インターフェイスモードの変更 (FMC)
1.  **Devices > Device Management** で対象 FTD を編集。
2.  使用する 2 つのインターフェイスの **Mode** を `None` に設定し、`Enabled` にチェックを入れます。

### 2. Inline Set の作成
1.  **Devices > Device Management > Interfaces** タブから **Inline Sets** を選択。
2.  **Add Inline Set** をクリックし、名前を付けます。
3.  **Interfaces** セクションで、先ほど `None` にした 2 つのインターフェイスをペアとして選択します。
4.  (オプション) **Propagate Link State** にチェックを入れ、片側のリンクダウンを対向に伝搬させます。

### 3. アクセスコントロールポリシーへの適用
1.  **Policies > Access Control** でルールを作成。
2.  **Inspection** タブで **Intrusion Policy** を選択。
3.  ルールの **Action** が `Allow` であることを確認（IPS がその中で動作するため）。

---

## 🔍 検証コマンド

| 目的 | コマンド（FTD CLI） |
| :--- | :--- |
| **Inline Set の状態確認** | <code>show inline-set</code> |
| **Snort 統計とドロップ数** | <code>show snort statistics</code> |
| **インターフェイスの統計** | <code>show interface</code> |
| **リアルタイムの Snort 動作** | <code>system support firewall-engine-debug</code> |
| **バイパス状態の確認** | <code>show external-bypass status</code> (HW依存) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 通信が全く通らない | インターフェイスがペアになっていない | <code>show inline-set</code> | FMC でインターフェイスペアを正しく作成しデプロイ。 |
| 攻撃を検知するが遮断しない | ルールの Action が Drop ではない | FMC Intrusion Policy | ルールアクションを <code>Drop and Generate</code> に変更。 |
| 片側のリンクが落ちても対向が Up のまま | Link State Propagation が無効 | FMC Inline Set 設定 | <code>Propagate Link State</code> を有効化。 |
| Snort 過負荷で遅延発生 | 検査対象が多すぎる | <code>show cpu</code> | 不要なトラフィックをインスペクションから除外 (Trust)。 |

---

## ⚠ 制限事項

*   **インターフェイスモード**: インラインペアに使用するインターフェイスは、L3 設定や VLAN タグ設定を持っていない `None` モードである必要があります。
*   **ハードウェアバイパス**: 物理的なバイパス（Fail-to-wire）は、特定のネットワークモジュール（NetMod）でのみサポートされます。
*   **SSL**: 暗号化されたトラフィックは、SSL 復号ポリシーを適用しない限り、Snort エンジンで内容を検査できません。

---

## 🔄 他技術との関連

*   **Access Control**: ACP の Allow ルール内で IPS 検査を呼び出すことで、インライン防御が成立します。
*   **High Availability**: フェイルオーバー時、スタンバイ側のインラインペアがアクティブ化され、通信を継続します。
*   **Quality of Service (QoS)**: インラインでのディープパケットインスペクションはレイテンシを生むため、優先制御とのバランス設計が重要です。

---

## 🧩 比較表

### In-line vs Passive (Passive Mode)

| 比較項目 | In-line Mode | Passive Mode |
| :--- | :--- | :--- |
| **トラフィックパス** | **パス上（直列）** | パス外（並列/SPAN） |
| **リアルタイム遮断** | **可能** | 不可（Shunning等が必要） |
| **ネットワークへの影響** | 故障時に通信断のリスクあり | 故障時も通信に影響なし |
| **導入の容易さ** | ネットワーク構成の変更が必要 | 既存環境に容易に追加可能 |

---

## 💡 ベストプラクティス

*   **タップモードからの開始**: 本番環境への投入時は、まず Inline Set 内で **Tap Mode** を有効にしてデプロイし、誤検知による通信遮断が発生しないことを確認します。
*   **Link State Propagation の利用**: IPS センサーをルータやスイッチの間に配置する場合、この機能を有効にすることで、障害発生時のルーティング切り替えを高速化できます。
*   **重要プロトコルのバイパス**: 制御通信（BGP, OSPF等）は、Intrusion Policy で `Trust` するか、Fast-path で Snort をバイパスさせることで、IPS 負荷によるネイバー断を防ぎます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な物理インターフェイスペアの設定
*   **要件**: Eth1/1 と Eth1/2 をインラインペアとして構成せよ。
*   **設定**: `Devices > Device Management > Edit > Interfaces`。両 IF を `None` にし、`Inline Sets > Add Inline Set` でペアにする。

### 2. リンク状態伝搬 (Link State Propagation)
*   **要件**: 片側のインターフェイスがダウンした場合、即座に対向もダウンさせよ。
*   **設定**: Inline Set オブジェクト編集画面で `Propagate Link State` を ON にする。

### 3. タップモードによる IDS シミュレーション
*   **要件**: インライン構成のまま、トラフィックを遮断せずにイベントのみ記録せよ。
*   **設定**: Inline Set 設定で `Tap Mode` を有効化する。

### 4. 独自シグネチャによるインラインブロック
*   **要件**: Telnet 応答に含まれる "Bad passwords" 文字列を検知して遮断せよ。
*   **設定**: `Policies > Intrusion > Rule Editor`。独自ルール作成、`content: "% Bad passwords"`、Action: `Drop and Generate`。

### 5. バイパス機能 (Fail-open) の確認
*   **要件**: センサー故障時に物理的に通信をバイパスさせよ。
*   **設定**: ハードウェアバイパス対応 NetMod を使用し、Inline Set でバイパス設定を `Enabled` にする。

### 6. 特定 VLAN トラフィックのインライン検査
*   **要件**: 802.1Q タグが付いたトラフィックを保持したままインライン検査せよ。
*   **設定**: Inline Set のインターフェイスペアで VLAN タグの透過を許可する。

### 7. Snort 3 へのアップグレードとインライン動作
*   **要件**: 次世代エンジン Snort 3 を使用してインラインスループットを向上させよ。
*   **設定**: `Devices > Device Management > Edit`。Snort エンジン設定を Snort 3 に切り替え。

### 8. FTD 透明モード（Transparent）でのインライン IPS
*   **要件**: FTD を L2 ファイアウォールとして動作させ、かつインライン IPS を実行せよ。
*   **設定**: デバイス自体を `Transparent` モードにし、BVI インターフェイスを構成した上で IPS ポリシーを適用。

### 9. インラインドロップイベントの確認
*   **課題**: 実際に遮断されたパケットを FMC で特定せよ。
*   **確認**: `Analysis > Intrusion > Events`。`Inline Result` カラムが `Dropped` になっていることを確認。

### 10. インラインペアの統計リセット
*   **要件**: 統計情報をクリアし、新規トラフィックのマッチングを検証せよ。
*   **コマンド**: `clear snort statistics`。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: Inline Set 設定で `Propagate Link State` が有効になっている。Eth1/1 に接続されたスイッチポートをシャットダウンしたとき、Eth1/2 はどうなるか？
    *   **正解**: FTD 自身が Eth1/2 のリンクステータスを強制的にダウンさせる。
2.  **トラブルシュート**: インライン構成にしたが、IPS ポリシーのイベントが発生しない。ACP で Action は `Allow` になっており、Intrusion Policy も紐付いている。他に確認すべき点は？
    *   **正解**: インターフェイスペアが物理的に正しく結線されているか、または Inline Set 設定で `Tap Mode` が誤って有効になっていないか確認する。
3.  **Design**: 冗長性のない単一パスに NGIPS を導入する際、デバイス故障による全断を避けるために必須のハードウェア要件は？
    *   **正解**: ハードウェアバイパス（Fail-to-wire）機能を備えたネットワークモジュールとインターフェイス。
4.  **実装**: インラインモードにおいて、Snort エンジンがパケットをドロップするようにするためには、Intrusion Policy 内のシグネチャアクションを何に設定すべきか？
    *   **正解**: `Drop and Generate Events`。
5.  **動作シーケンス**: インラインモードの FTD で、NAT 処理と IPS 検査はどちらが先に行われるか？
    *   **正解**: Routed/Transparent の設定に依存するが、一般的に Snort エンジン（IPS）へのリダイレクトは LINA エンジンの前処理の後に行われる。

---

## 🔗 参考リソース

*   **Configuration Guide**:
    *   [Cisco Secure Firewall Management Center Administration Guide, 7.1 - Inline Sets](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/interfaces-settings-inline-sets.html)
*   **Cisco Live**:
    *   [BRKSEC-2021: Firepower Threat Defense - Packet Flow and Troubleshooting](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-2021)
*   **Technical Notes**:
    *   [Understanding Firepower Deployment Modes](https://www.cisco.com/c/en/us/support/docs/security/firepower-management-center/212356-understand-firepower-deployment-modes.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: インラインモードは CCIE ラボで「IPS としての機能」を試すための大前提の設定です。ここを間違えると、その後のシグネチャ調整や遮断テストが全て失敗するため、最初に行うインターフェイス設定が最も重要です。
*   **図解**: 常に「パケットがデバイスを通り抜ける」イメージを持ち、物理的なペア（Set）を意識してください。
*   **注意点**: ラボ試験では、インターフェイスを `None` モードに設定した後、**一度 Save してからでないと Inline Set のドロップダウンに表示されない**ことが多いため、操作手順の正確さが求められます。  

---

📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

---
