---
layout: default
title: 1.5.a-Inline
nav_order: 1
parent: 1.5-NGIPS-deployment-modes
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.5.a In-line

Cisco Secure Firewall Threat Defense (FTD) および NGIPS アプライアンスにおける **In-line Mode（インラインモード / インライン配置）** は、セキュリティアプライアンスを通信経路に物理的・論理的に直接挿入（"Bump in the wire"）し、ネットワークを通過するトラフィックに対してリアルタイムでインスペクション（深層パケット解析）と動的なアクセス制御・脅威遮断（Drop / Block / Reset）を即座に実行するインパケット型のセキュリティ導入モードです。

CCIE Security v6.1 ラボ試験において、インラインモードは単に 2 つのポートをペアとしてバインドする設定にとどまらず、**Inline Set（インラインセット）と Inline Pair（インラインペア）の論理構造、LINA と Snort インスペクションエンジンのパケット処理パイプライン、非同期ルーティング（Asynchronous Routing）の対処、障害回避メカニズム（Propagate Link State、Snort Fail Open、Hardware Bypass）、および BFD/STP/Q-in-Q 等のインフラプロトコルとの相互作用** まで含めた高度なトラブルシューティングおよび設計能力が厳格に問われます。

---

## 📘 概要

* **機能概要**:
  * 装置の 2 つのデータインターフェイスを組にして論理的な **Inline Pair** を形成し、それらを束ねた **Inline Set** として運用することで、ブリッジやルーターの IP 設定を伴わずにパケットを透過的に通過させながらリアルタイムで Snort IPS によるディープパケットインスペクションを実施するモード。
  * 通信ポリシー（Access Control Policy / Intrusion Policy）に合致した悪意あるトラフィックや未承認プロトコルを、データプレーン上で即座にドロップ（Drop）または接続リセット（Reset）可能。
* **利用目的**:
  * 悪意あるマルウェアの侵入、ゼロデイ攻撃、Botnet C2 通信、脆弱性攻撃シグネチャの即時遮断（Pre-infection Mitigation）。
  * インフラの IP アドレッシングや既存のルーティング設計を変更することなく、透過的かつインラインで高度なセキュリティインスペクションレイヤーを追加。
* **どのような場面で利用するか**:
  * インターネット境界、DMZ 境界、データセンター内部セグメント間など、脅威の検知だけでなく「確実な通信遮断」が要求されるすべてのセキュリティ境界。
  * スイッチとルーター間、またはスイッチとファイアウォール間の物理リンクに透過的に挿入する構成。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **透過性 (Agnostic)** | インターフェイスに IP アドレスの設定は不要（IP アドレス非依存） |
| **構成要素** | **Inline Pair**（2 ポートのペア）と、それをまとめる論理グループ **Inline Set** |
| **脅威遮断能力** | **完全対応**（Drop, Block, Block with reset, Interactive Block） |
| **必要インターフェイス** | 最低 **2 ポート**（物理ポートまたは EtherChannel） |
| **非対応インターフェイス** | **Redundant Interface、VLAN Subinterface (FXOS レベル等)** は非サポート |
| **障害回避機能** | **Propagate Link State**、**Snort Fail Open (Busy/Down)**、**Hardware Bypass (Fail-to-Wire)** |
| **L7/L3 動作関係** | Routed Mode / Transparent Mode のどちらの Firewall Mode 上でも IPS-Only として共存可能 |
| **制限事項** | インラインペア単体での NAT 設定は非サポート。BFD Echo パケットは LAND 攻撃としてドロップされる |

---

## 🏗 動作原理

インラインモードでは、一方の物理ポートに入力されたパケットが最小限の LINA チェックを受けた後、直接 Snort エンジンへ引き渡され、ポリシールールに従って処理された後にペアとなるもう一方の物理ポートから出力されます。

```text
[ Ingress Packet ]
       │
       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Firepower Threat Defense (FTD) - Inline Pair (Gi0/0 <-> Gi0/1)                   │
│                                                                                  │
│  Ingress Interface (Gi0/0)                                                       │
│         │                                                                        │
│         ▼                                                                        │
│  ┌──────────────┐   (Minimal LINA Engine Sanity Checks)                          │
│  │ LINA Engine  │                                                                │
│  └──────┬───────┘                                                                │
│         │                                                                        │
│         ▼                                                                        │
│  ┌──────────────┐   (Snort Inspection Pipeline)                                  │
│  │ Snort Engine ├───────────────────────┐                                        │
│  └──────┬───────┘                       │                                        │
│         │                               │ Match Drop / Reset Rule                │
│         │ Permitted / Clean             ▼                                        │
│         ▼                       ┌──────────────┐                                 │
│  ┌──────────────┐               │ Packet Drop  │ (Generate Intrusion Event)     │
│  │ LINA Engine  │               │ / Send RST   │                                 │
│  └──────┬───────┘               └──────────────┘                                 │
│         │                                                                        │
│         ▼                                                                        │
│  Egress Interface (Gi0/1)                                                        │
└─────────┬────────────────────────────────────────────────────────────────────────┘
          │
          ▼
[ Egress Packet Transmitted ]
```

---

## ⚙ 動作シーケンス

1. **パケット受信とインターフェイスの受容**:
   * パケットが Ingress ポート（例: `Gi0/0`）に着信すると、物理層および MAC レイヤーでの入力エラーを検証します。
2. **LINA パケット処理パイプライン (Partial LINA Checks)**:
   * インラインペアにアサインされたポートは、通常のファイアウォール処理（NAT、ルーティングテーブル検索等）の大部分をバイパスし、パケットを速やかに DAQ (Data Acquisition) レイヤー経由で Snort プロセスへ渡します。
3. **Snort インスペクションポリシー評価**:
   * ネットワーク解析ポリシー (NAP: Network Analysis Policy) によるパケットの正規化、TCP ストリームの再構築、IP フラグメントの結合を実行。
   * アクセス制御ポリシー (ACP) および侵入防御ポリシー (Intrusion Policy) のシグネチャ条件に合致するか検査。
4. **アクションの決定と実行**:
   * **Allow / Trust**: 検査を通過したパケットは対向ポート（`Gi0/1`）の送信キューへ直接回送されます。
   * **Block / Drop**: 侵入シグネチャにマッチした場合、Snort はパケットをその場で破棄し、FMC へ Intrusion Event（および必要に応じて Connection Event）を送出します。
   * **Block with Reset**: TCP 接続の場合、送信元・宛先の両方向へ偽装した TCP RST パケットを送出してセッションを強制終了させます。
5. **エグレス送信**:
   * 正常と判定されたパケットが対向ポートから同一の Layer 2/3 ヘッダー情報を保持したまま送出されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Inline Set と Inline Pair の確実な定義**:
   * **共通のミス**: 単にインターフェイスの Mode を `None` にして有効化（Enable）しただけではインライン動作しません。
   * **必須手順**:
     1. インターフェイス設定で `Mode: None` とし `Enabled` にチェック。
     2. `Devices > Device Management > Inline Sets` タブで **Inline Set** を新規作成。
     3. 利用可能なペアから `Available Interfaces Pairs` を選択し `Selected Interface Pair` へ追加して保存・Deploy します。
2. **非同期ルーティング (Asynchronous Routing) 対策**:
   * 往路（Client ➔ Server）と復路（Server ➔ Client）が異なる物理ポートペアを通過するトポロジにおいて、往路ペア（例: `Gi0/0-Gi0/1`）と復路ペア（例: `Gi0/2-Gi0/3`）を**同一の Inline Set に含める必要があります**。
   * 別々の Inline Set に分割すると、Snort が片方向のトラフィックしか見えず、セッション状態を喪失して通信がドロップします。
3. **Propagate Link State（リンク状態伝搬）の必須適用**:
   * インラインペアの一方の物理リンク（例: WAN 側）が Down した際、対向のポート（例: LAN 側）の物理リンクも自動的に Down させる機能です（伝搬時間: 最大 4 秒）。
   * 上流・下流スイッチの OSPF/EIGRP/BGP や HSRP/VRRP に物理リンクダウンを即座に検知させ、網の高速迂回収束を促すために試験では必ず有効化が要求されます。
4. **Snort Fail Open (Failsafe) の使い分け**:
   * **`Busy`**: バッファオーバーフローや高負荷時に未検査でパケットを通過させる。
   * **`Down`**: Snort プロセスが再起動・クラッシュした際に未検査でパケットを通過させる。
   * 高可用性要件（"Ensure network connectivity is maintained even during Snort process crash"）が出題された場合、両方のチェックを入れる必要があります。
5. **Hardware Bypass (Fail-to-Wire) の適用条件と制約**:
   * 対応ハードウェア（Firepower 2130/2140、4100、9300 の FTW ネットワークモジュール）でのみ利用可能。
   * **試験で狙われる制限事項**:
     * **HA (Active/Standby) との併用は不可**。
     * **EtherChannel 内での使用は不可**。
     * **`Propagate Link State` と同一 Inline Set 内で同時に有効化することは禁止**。
6. **BFD (Bidirectional Forwarding Detection) との競合**:
   * インラインペアを通過する BFD Echo パケットは、送信元 IP と宛先 IP が同一となるため、FTD の LINA エンジンによって **LAND 攻撃** と判定されて強制ドロップされます。
   * 対向ルーター間で `no bfd echo` を設定するか、BFD モードを調整する必要があります。
7. **Q-in-Q (Double 802.1Q Tagging) のサポート限界**:
   * FTD のインラインペアは最大 2 つまでの 802.1Q ヘッダー (Q-in-Q) を透過処理できますが、**Firepower 4100 / 9300 シリーズは 1 つの 802.1Q ヘッダーのみサポート** されます。

---

## 🛠 設定方法

### 1. FMC GUI: インラインペアおよびインラインセットの基本構築手順

```text
[ Step 1: 物理インターフェイスの有効化 ]
1. Devices > Device Management に移動し、対象 FTD の編集 (鉛筆アイコン) をクリック。
2. Interfaces タブを選択。
3. GigabitEthernet0/0 を編集:
   - Name: INLINE_INSIDE
   - Mode: None
   - Enabled: [✓] チェック
4. GigabitEthernet0/1 を編集:
   - Name: INLINE_OUTSIDE
   - Mode: None
   - Enabled: [✓] チェック
5. "Save" をクリック。

[ Step 2: Inline Set の作成と Interface Pair の割り当て ]
1. "Inline Sets" タブを選択し "Add Inline Set" をクリック。
2. General タブ:
   - Name: PROD_INLINE_SET
   - MTU: 1500
   - Available Interfaces Pairs リストから "INLINE_INSIDE <-> INLINE_OUTSIDE" を選択し "Add" をクリック。
3. Advanced タブ:
   - [ ] Tap Mode (チェックオフ確認)
   - [✓] Propagate Link State
   - Snort Fail Open: [✓] Busy, [✓] Down
4. "OK" をクリック (インターフェイス設定変更の警告ポップアップで "Yes" を選択)。
5. "Save" をクリックし、変更を FTD へ Deploy。
```

---

### 2. FTD CLI: インラインセット設定状態の直接編集・確認

FTD CLI では、`show inline-set` コマンドで構成および動作ステータスを詳細に確認できます。

```bash
> show inline-set
Inline-set PROD_INLINE_SET
Mtu is 1500 bytes
Fail-open for snort down is on
Fail-open for snort busy is on
Tap mode is off
Propagate-link-state option is on
hardware-bypass mode is disabled
Interface-Pair:
Interface: GigabitEthernet0/0 "INLINE_INSIDE"
Current-Status: UP
Interface: GigabitEthernet0/1 "INLINE_OUTSIDE"
Current-Status: UP
Bridge Group ID: 501
>
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI) | 表示確認ポイント |
| :--- | :--- | :--- |
| **Inline Set 全体の動作設定・ステータス確認** | <code>show inline-set</code> | `Fail-open`、`Tap mode`、`Propagate-link-state`、各ポートの `Current-Status` |
| **個々の物理ポートの IPS モード確認** | <code>show interface GigabitEthernet 0/0</code> | `IPS Interface-Mode: Inline` であること、ドロップカウントの推移 |
| **パケット処理パイプラインのリアルタイムトレース** | <code>packet-tracer input INLINE_INSIDE icmp 10.1.1.10 8 8 10.2.2.10</code> | LINA の処理フェーズを通過し、結果が ALLOW となること |
| **Snort インスペクションでのリアルタイムドロップ確認** | <code>system support firewall-engine-debug</code> | 指定した条件のパケットに対して `Intrusion Block` や `Drop` が発生しているか |
| **LINA データプレーンでのドロップ理由特定** | <code>show asp drop</code> | LAND 攻撃判定（BFD Echo 等）やバッファフルによるドロップ有無 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **インラインペアの片側ポートを抜いても対向ポートが Down にならない** | Inline Set の `Propagate Link State` が未設定、または設定後 Deploy されていない | <code>show inline-set</code> | FMC の Inline Set 設定画面（Advanced タブ）で `Propagate Link State` を有効化して Deploy。 |
| **インラインペアを通過する BFD (Bidirectional Forwarding Detection) セッションが確立しない** | BFD Echo パケット（送信元/宛先 IP が同一）を LINA が LAND 攻撃と判定してドロップする | <code>show asp drop</code> | 対向ルーターで `no bfd echo` を設定し、非 Echo モードで BFD を動作させる。 |
| **非同期通信時に特定のセッションが意図せずドロップする** | 往路ペアと復路ペアが別々の Inline Set に分割されているため Snort が片方向フローしか認識できない | <code>show inline-set</code> | 往復双方の Interface Pair を同一の Inline Set に統合アサインする。 |
| **Snort クラッシュ時にトラフィックが全遮断される** | `Snort Fail Open (Down)` がオフになっており、Fail-Close 動作となっている | <code>show inline-set</code> | Inline Set の Advanced 設定で `Snort Fail Open (Down)` にチェックを入れて Deploy。 |
| **FTD 再起動直後に外部ルーターが即座にパケットを送信してドロップが発生する** | FXOS と FTD アプリケーション間の Operational Link State が同期していない | FXOS: <code>show interface expand detail</code> | FXOS Chassis Manager にて Logical Device の `Enable Link State` を有効化する。 |

---

## ⚠ 制限事項

* **未サポートのインターフェイスタイプ**:
  * Inline Set にアサインできるのは **物理物理ポート** および **EtherChannel** のみです。
  * Redundant Interface、VLAN Subinterface、FXOS シャーシ定義の Subinterface はアサインできません。
* **Hardware Bypass の厳格な制限**:
  * Hardware Bypass (FTW) 機能は、**High Availability (HA) モードとは併用不可**。
  * **EtherChannel のメンバーポートとしては使用不可**。
  * **`Propagate Link State` と同一 Inline Set 内で同時設定することは不可**。
* **NAT 機能の制限**:
  * インラインペアを通過するフローに対して、FTD 上で直接 NAT 変換（Network Address Translation）を適用することはできません。

---

## 🔄 他技術との関連

* **Routed vs Transparent Firewall Mode**: FTD の基本動作モード。インラインペア（IPS-Only インターフェイス）は、FTD が Routed モードか Transparent モードのどちらで配置されていても完全に共存・動作可能です。
* **Asynchronous Routing（非同期ルーティング）**: 非同期通信が発生するトポロジでは、関連するすべてのインラインペアを 1 つの Inline Set に含めることがセッション保持の絶対要件です。
* **Spanning Tree Protocol (STP)**: インラインペアに接続される外部スイッチポートでは、Hardware Bypass 切り替え時やリンクダウン復旧時の即時転送移行を図るため、**STP PortFast を有効化** することが Cisco 推奨です。

---

## 🧩 比較表

### 1. In-line Mode vs Passive Mode vs Inline Tap Mode

| 比較項目 | In-line Mode | Passive Mode | Inline Tap Mode |
| :--- | :--- | :--- | :--- |
| **パケット位置** | インライン（帯域内） | 帯域外（SPAN / TAP 由来のコピー） | 物理インライン / 処理はコピー |
| **脅威遮断 (Drop)** | **可能**（リアルタイム破棄・RST） | **不可**（検知・アラートのみ） | **不可**（`Would Have Dropped` 記録） |
| **IP アドレス設定** | 不要 | 不要 (ERSPAN 送受信時除く) | 不要 |
| **必要ポート数** | 最低 **2 ポート** | 最低 **1 ポート** | 最低 **2 ポート** |
| **インライン移行** | 最初からインライン | ケーブルの物理再配線が必要 | **GUI 設定の変更のみ（配線変更不要）** |

### 2. Snort Fail Open vs Hardware Bypass (Fail-to-Wire)

| 比較項目 | Snort Fail Open | Hardware Bypass |
| :--- | :--- | :--- |
| **障害保護レイヤー** | ソフトウェア層（Snort プロセス） | 物理ハードウェア層（物理リレー回路） |
| **トリガー要因** | Snort プロセスの High CPU (Busy) や Crash (Down) | 装置の完全電源断、ハードウェア故障、シャーシクラッシュ |
| **対応機種** | 全モデル（FTDv、ASA-FirePOWER、FTD アプライアンス） | 特定モデルの FTW ネットワークモジュール（2130/2140, 4100, 9300） |
| **併用可能機能** | HA、EtherChannel、Propagate Link State | **HA 不可、EtherChannel 不可、Propagate Link State 不可** |

---

## 💡 ベストプラクティス

1. **すべての Inline Set で Propagate Link State を有効化**:
   * ネットワークの冗長ルーティング（OSPF/EIGRP/BGP）や HSRP/VRRP と連携させるため、例外なく `Propagate Link State` を有効にしてリンク障害時の迅速な迂回を確立します。
2. **対向スイッチポートへの STP PortFast 設定**:
   * インラインペアが接続されるスイッチポートでは `spanning-tree portfast trunk`（または `portfast`）を有効化し、リンクアップ時の STP Listening/Learning による 30 秒間のトラフィック停止を回避します。
3. **段階的な Inline 移行アプローチ (Inline Tap の活用)**:
   * 本番環境への初期導入時は、まず **`Inline Tap Mode`**（または Intrusion Policy の `Detection` モード）で運用を開始して偽陽性（False Positive）ルールをチューニングし、問題がないことを確認した後に GUI 操作のみで **`In-line Prevention`** へ移行します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: インラインペアおよびインラインセットの基本構成

#### 問題
FTD の 2 つの物理ポート (`GigabitEthernet0/0` と `GigabitEthernet0/1`) を使用して、透過的なインライン防御モード (In-line Mode) を構成せよ。

#### 要件
* インターフェイス名: `Gi0/0` ➔ `LAN_IN`、`Gi0/1` ➔ `WAN_OUT`
* インラインセット名: `SET_INLINE_BASE`
* IP アドレスは設定しないこと。

#### 設定・操作手順

```text
1. FMC UI で Devices > Device Management を開き対象 FTD を選択。
2. Interfaces タブで GigabitEthernet0/0 を編集:
   - Name: LAN_IN
   - Mode: None
   - Enabled: [✓]
3. GigabitEthernet0/1 を編集:
   - Name: WAN_OUT
   - Mode: None
   - Enabled: [✓]
4. Inline Sets タブに移動し "Add Inline Set" をクリック:
   - Name: SET_INLINE_BASE
   - Available Interfaces Pairs リストから "LAN_IN <-> WAN_OUT" を追加。
5. Save をクリックし、対象 FTD へ Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`Interface-Pair` に `LAN_IN` と `WAN_OUT` が `Current-Status: UP` で表示されることを確認。

---

### 例2: Propagate Link State（リンク状態伝搬）の構成

#### 問題
インラインセット `SET_INLINE_BASE` において、一方の物理ケーブルが抜けた際に対向の物理ポートも自動的に Down させるよう設定せよ。

#### 要件
* `Propagate Link State` オプションを有効化すること。

#### 設定・操作手順

```text
1. Devices > Device Management > Inline Sets タブを開く。
2. "SET_INLINE_BASE" の編集 (鉛筆アイコン) をクリック。
3. "Advanced" タブを選択。
4. [✓] Propagate Link State にチェックを入れる。
5. OK ➔ Save をクリックし、対象 FTD へ Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`Propagate-link-state option is on` と表示されることを確認。

---

### 例3: Snort 障害時の高可用性を担保する Snort Fail Open 設定

#### 問題
Snort インスペクションエンジンが高負荷 (Busy) に達した場合、またはプロセス障害 (Down) で停止した場合に、通信を落とさず未検査のまま通過（Bypass）させるよう構成せよ。

#### 要件
* `Snort Fail Open` の `Busy` および `Down` の両方を有効化すること。

#### 設定・操作手順

```text
1. Devices > Device Management > Inline Sets を開く。
2. "SET_INLINE_BASE" を編集し "Advanced" タブを選択。
3. Snort Fail Open セクションで以下をチェック:
   - [✓] Busy
   - [✓] Down
4. OK ➔ Save をクリックして Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`Fail-open for snort down is on` および `Fail-open for snort busy is on` を確認。

---

### 例4: 非同期ルーティング環境における複数インラインペアの統合

#### 問題
往路トラフィック（`Gi0/0 <-> Gi0/1`）と復路トラフィック（`Gi0/2 <-> Gi0/3`）が異なる物理ポートを通過する非同期ルーティングトポロジにおいて、Snort が同一フローとして正しくセッションを認識できるよう設定せよ。

#### 要件
* インラインセット名: `ASYNC_INLINE_SET`
* 2 つのインラインペアを単一の インラインセットへアサインすること。

#### 設定・操作手順

```text
1. Gi0/0 (IN_1), Gi0/1 (OUT_1), Gi0/2 (IN_2), Gi0/3 (OUT_2) の 4 ポートの Mode を None にして有効化。
2. Inline Sets タブで "Add Inline Set" を追加:
   - Name: ASYNC_INLINE_SET
   - Selected Interface Pairs リストに "IN_1 <-> OUT_1" および "IN_2 <-> OUT_2" の両方を追加。
3. Save をクリックして Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`Interface-Pair` と `Interface-Pair` の 2 つのペアが同一 Inline Set 内に含まれていることを確認。

---

### 例5: EtherChannel を用いたバンドルインラインペアの構成

#### 問題
FTD の 2 つの物理ポート (`Gi0/4` と `Gi0/5`) で EtherChannel (`Port-Channel 1`) を形成し、対向の EtherChannel (`Port-Channel 2`) と組にして大容量インラインペアを構築せよ。

#### 要件
* インラインセット名: `BUNDLE_INLINE_SET`

#### 設定・操作手順

```text
1. Devices > Device Management > Interfaces で Add EtherChannel を作成 (Port-Channel 1: Gi0/4, Gi0/5 / Mode: None)。
2. 別途 Port-Channel 2 を作成。
3. Inline Sets タブで "Add Inline Set" を作成:
   - Name: BUNDLE_INLINE_SET
   - Selected Interface Pair: "Port-Channel1 <-> Port-Channel2" をアサイン。
4. Save をクリックして Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`Interface-Pair` の構成メンバーが `Port-Channel1` と `Port-Channel2` になっていることを確認。

---

### 例6: Firepower 2130 / 4100 ハードウェアバイパス (Fail-to-Wire) の構成

#### 問題
FTD (Firepower 2140) の FTW ネットワークモジュールポートペア (`Gi1/1 <-> Gi1/2`) において、装置の完全電源断時に物理リレーを直接ショートさせて通信を通過させる `Hardware Bypass` を設定せよ。

#### 要件
* Bypass Mode: `Standby`

#### 設定・操作手順

```text
1. Devices > Device Management > Interfaces を開く。
2. GigabitEthernet1/1 と GigabitEthernet1/2 の Mode を None にして有効化。
3. Inline Sets タブで "Add Inline Set" を作成:
   - Name: HW_BYPASS_SET
   - Bypass: Standby を選択。
   - Selected Interface Pair: "Gi1/1 <-> Gi1/2" を追加。
4. Advanced タブで Propagate Link State がオフ（非チェック）であることを確認。
5. OK ➔ Save をクリックして Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`hardware-bypass mode is standby` (または `enabled`) と表示されることを確認。

---

### 例7: BFD セッション切断トラブルの修正 (LAND Attack Drop 回避)

#### 問題
インラインペアを通過する対向ルーター間の BFD セッションが FTD 挿入後に切断された。通信を復旧させるための設定変更を行え。

#### 要件
* FTD で BFD Echo パケットが LAND 攻撃としてドロップされる問題を回避すること。

#### 設定・操作手順

```text
[ 対向ルーター側の設定変更 ]
1. 対向する両ルーターの BFD 設定配下に移動。
2. "no bfd echo" コマンドを投入し、BFD Control パケット（非 Echo）のみでネイバーを維持するよう変更。
```

#### 検証方法
* FTD CLI で `show asp drop` を実行し、LAND 攻撃カウンター (`land-attack`) の増加が停止し、ルーター間で BFD ネイバーが `UP` となることを確認。

---

### 例8: Inline Tap Mode への安全な移行

#### 問題
インラインセット `SET_INLINE_BASE` において、物理配線を変更することなく、一時的に通信を一切遮断しない検出専用モード (Inline Tap) へ移行させよ。

#### 要件
* `Tap Mode` を有効化すること。

#### 設定・操作手順

```text
1. Devices > Device Management > Inline Sets を開く。
2. "SET_INLINE_BASE" を編集。
3. "Advanced" タブで [✓] Tap Mode にチェックを入れる。
4. OK ➔ Save をクリックし、対象 FTD へ Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`Tap mode is on` と表示されることを確認。

---

### 例9: FXOS 物理リンク同期 (Enable Link State) の有効化

#### 問題
Firepower 4100 シャーシにおいて、FTD アプライアンスプロセスが完全に起動するまで物理リンクの Up 状態を保留し、起動途中のパケットドロップを防ぐ `Link State Sync` を有効化せよ。

#### 要件
* FXOS Chassis Manager または FXOS CLI で設定すること。

#### 設定・操作手順

```bash
# ===== FXOS CLI にて設定する場合 =====
FPR4K-A# scope system
FPR4K-A /system# scope logical-device FTD_Standalone
FPR4K-A /system/logical-device# set link-state-sync enabled
FPR4K-A /system/logical-device*# commit-buffer
```

#### 検証方法
* FXOS CLI で `show interface expand detail` を実行し、`Link State Sync: Enabled` であることを確認。

---

### 例10: Inline Pair における 802.1Q Q-in-Q タグ透過検証

#### 問題
VLAN 100 の外側に VLAN 200 の S-Tag が付与された Double-tagged (Q-in-Q) パケットをインラインペアで透過インスペクションさせよ。

#### 要件
* 2 レイヤーの 802.1Q ヘッダーを維持したまま転送すること（FTD 2100 シリーズ使用）。

#### 設定・操作手順

```text
1. インラインペア (Gi0/0 <-> Gi0/1) が正常に UP していることを確認。
2. 接続スイッチ側で Q-in-Q ケーブリングを構成。
3. ACP ルールで対象の内部 VLAN トラフィックのインスペクションを有効化して Deploy。
```

#### 検証方法
* スイッチから Q-in-Q パケットを送出し、FTD を通過後も対向スイッチで外側・内側の両 VLAN タグが保持されたまま受信されることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で FTD の物理ポート `Gi0/0` と `Gi0/1` を使用して Inline Set を構成した。保守テストとして `Gi0/0` の LAN ケーブルを引き抜いたが、`Gi0/1` 側の対向ルーターのポートは UP 状態のままであり、ネットワークのルーティング迂回が発動しなかった。FMC 上で不足している設定はどれか。

* A) Snort Fail Open が無効化されているため。
* B) Inline Set の Advanced 設定で `Propagate Link State` にチェックが入っていないため。
* C) FTD が Transparent Firewall Mode で動作しているため。
* D) `Strict TCP Enforcement` が有効化されているため。

**正解**: **B**  
**解説**: **Propagate Link State（リンク状態伝搬）** が有効化されていない場合、インラインペアの一方の物理リンクが Down しても対向ポートのリンク状態は維持されます。これを有効化することで、対向ポートも自動的に Down させ、外部ルーターの動的ルーティングプロトコルや HSRP/VRRP に即時迂回を発生させることができます。

---

### 2. トラブルシュート問題

**問題**: FTD のインラインペアを通過する 2 台のルーター間で BFD (Bidirectional Forwarding Detection) ピアを確立しようとしたが、BFD セッションが `DOWN` のまま追従しない。FTD の CLI で `show asp drop` を実行したところ `land-attack` カウンターが増加していた。適切な対処方法はどれか。

* A) FTD 上で BFD 用の Static Route を追加する。
* B) 対向ルーター上で `no bfd echo` を設定し、送信元・宛先 IP が同一となる BFD Echo パケットの送出を停止させる。
* C) FTD の Firewall Mode を Routed Mode から Transparent Mode へ変更する。
* D) Inline Set の `Tap Mode` を有効化する。

**正解**: **B**  
**解説**: BFD Echo パケットは自 IP アドレス宛てに送出されてループバックしてくるため、送信元 IP と宛先 IP が同一になります。FTD の LINA データプレーンはこれを **LAND 攻撃** と判定してドロップします。対向ルーターで `no bfd echo` を設定して非 Echo (Control) モードへ変更することで解決します。

---

### 3. Design 問題

**問題**: 企業境界に配置する FTD において、FTD アプライアンスの完全に予期せぬ全電源断（Power Outage）時でも物理層で通信を機械的に導通（Fail-to-Wire）させ、ネットワークダウンタイムをゼロに抑えたい。設計上留意すべき制約事項として正しいものはどれか。

* A) Hardware Bypass 機能を使用し、該当ポートペアを FTD の Active/Standby HA 構成に組み込む。
* B) Hardware Bypass 機能を使用するが、この機能は HA 構成、EtherChannel、および Propagate Link State とは同時に併用できない制限を許容する。
* C) Snort Fail Open 機能のみを使用して物理リレーを制御する。
* D) FTD を Transparent Mode に変更すれば全電源断時も自動で通過する。

**正解**: **B**  
**解説**: **Hardware Bypass (Fail-to-Wire)** は物理リレー回路を用いて電源断時にも通信を物理ショートさせて通過させる強力な機能ですが、**HA (High Availability) モード、EtherChannel、および Propagate Link State との併用は不可** という明確な設計制限が存在します。

---

### 4. 実装問題

**問題**: 非同期ルーティング（往路と復路が異なるインターフェイスペアを通過する）環境において、FTD のインラインペアで通信が正常に検査・許可されるようにするための正しい FMC 上の設定構成はどれか。

* A) 往路ペアと復路ペアをそれぞれ別々の Inline Set に作成し、異なる Security Zone をアサインする。
* B) 往路ペアと復路ペアの両方を単一の Inline Set に含めてアサインする。
* C) FMC の Advanced 設定で `Asynchronous Routing Bypass` オプションを有効化する。
* D) FTD を Active/Active クラスタリング構成に変更する。

**正解**: **B**  
**解説**: 非同期ルーティング環境では、往路と復路の双方の Interface Pair を**同一の Inline Set に含める** ことで、FMC/FTD は両方向のトラフィックを同一セッションフローとして正しく認識・追跡できるようになります。

---

### 5. コンフィグ読解問題

**問題**: FTD CLI で `show inline-set` を実行したところ、以下の出力が得られた。この表示から読み取れる FTD の動作状態として正しいものはどれか。

```text
> show inline-set
Inline-set INLINE_SET_1
Mtu is 1500 bytes
Fail-open for snort down is on
Fail-open for snort busy is on
Tap mode is off
Propagate-link-state option is on
hardware-bypass mode is disabled
```

* A) 通信はコピーパケットに対してのみ検知処理が行われている (Tap Mode)。
* B) Snort プロセスが停止または高負荷になった場合でも、トラフィックは未検査で通過し、片側の物理リンクダウン時に対向ポートも Down する。
* C) 電源切断時に物理リレーが閉じてハードウェアバイパスが作動する。
* D) 非同期トラフィックはすべてドロップされる。

**正解**: **B**  
**解説**: `Fail-open for snort down/busy is on`（Snort 停止/高負荷時に通過）および `Propagate-link-state option is on`（リンク状態伝搬有効）が表示されているため、ソフトウェア障害時にも通信を維持しつつ、リンク障害を対向へ伝搬させる高可用性インライン構成となっています。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Threat Defense Configuration Guide - Inline Sets and Passive Interfaces]([suspicious link removed])
* [Cisco Secure Firewall Threat Defense Command Reference]([suspicious link removed])
* [Cisco Technical Notes: Configure and Troubleshoot Inline Sets and Tap Mode on FTD]([suspicious link removed])
* [Cisco Live: Deep Dive into Cisco Secure Firewall Architecture and Deployment Modes (BRKSEC-3021)]([suspicious link removed])

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で「Configure FTD to transparently inspect traffic without changing IP addresses and ensure link failure is communicated to adjacent routers」と要求された場合、構築すべきは **`Inline Set` + `Propagate Link State`** です。
* **図解（In-line Mode における障害制御メカニズム）**:
  ```text
  [ Physical Link Failure ] ────────► Propagate Link State ────────► Brings Down Paired Interface (Reroute Traffic)
  [ Snort Process Crash ]   ────────► Snort Fail Open (Down) ──────► Passes Traffic Uninspected (Keep Service)
  [ Traffic Overload ]      ────────► Snort Fail Open (Busy) ──────► Passes Traffic Uninspected (Avoid Buffer Drop)
  [ Chassis Power Loss ]    ────────► Hardware Bypass (FTW)  ──────► Physical Relay Closes Wire (Passes Signal)
  ```
* **注意点**:
  * ESXi などの仮想環境（FTDv / NGIPSv）でインラインペアを構築する際は、vSwitch の Port Group 設定において **`Promiscuous Mode: Accept`**、**`MAC Address Changes: Accept`**、**`Forged Transmits: Accept`** の 3 つの項目を必ず Accept に変更してください。これを怠ると、インラインペアポートがパケットを正しく受信・フォワーディングできません。
