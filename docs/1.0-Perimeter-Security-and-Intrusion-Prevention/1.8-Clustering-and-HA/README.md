---
layout: default
title: 1.8-Clustering-and-HA
nav_order: 8
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.8 Clustering and high availability features on Cisco ASA and Cisco FTD

Cisco ネットワークセキュリティ環境における **Clustering（クラスタリング）** および **High Availability（HA / Failover）** 機能は、Cisco ASA（Adaptive Security Appliance）および Cisco FTD（Firepower Threat Defense / Secure Firewall）アプライアンスにおいて、ミッションクリティカルなネットワーク境界の可用性向上（Redundancy）とスループレット拡張（Throughput Scaling）を実現するコア技術です。

CCIE Security v6.1 Blueprint **1.8 Clustering and high availability features on Cisco ASA and Cisco FTD** では、**ASA Active/Standby** および **ASA Active/Active Failover**、**ASA/FTD の Clustering（Spanned EtherChannel モードおよび Individual Interface モード）**、**Cluster Control Link (CCL)** の設計・運用、**FTD Active/Standby HA**、ならびに **FXOS (Firepower Extensible Operating System) / FMC (Firepower Management Center)** 経由でのクラスタ構成、ヘルスモニタリング、ステート同期、非対称ルーティング（ASR）処理、トラブルシューティングに至るまで、実効性の高い設計・構築・検証能力が深く問われます。

---

## 📘 概要

* **機能概要**:
  * **ASA Failover (Active/Standby ＆ Active/Active)**:
    * **Active/Standby (A/S)**: 1台のアプライアンスがアクティブとして全トラフィックを処理し、もう1台がスタンバイ状態として待機する冗長モデル。Single Mode および Multi-Context Mode の両方でサポートされる。
    * **Active/Active (A/A)**: Multi-Context Mode が前提。複数のセキュリティコンテキストを「Failover Group 1」および「Failover Group 2」に分離し、両アプライアンスで同時にトラフィックをアクティブ処理させる負荷分散・冗長モデル。
  * **FTD High Availability (Active/Standby)**:
    * FTD アプライアンス（全モデル対応）において、2台の同一アプライアンスを Failover Link および Stateful Link で接続し、Active/Standby 冗長を構成する。※FTD では Active/Active モードは未サポート。
  * **Cisco ASA / FTD Clustering**:
    * 最大 16 台のアプライアンス（プラットフォーム依存）を1つの論理ファイアウォール（Single Logical Unit）としてグループ化し、N+1 冗長性と処理能力の線形拡張（Scale-out）を同時に実現する技術。
    * **Spanned EtherChannel Mode (L2/L3)**: 上流・下流スイッチと LACP（cLACP）を用いた1本の論理 Port-Channel で接続し、スイッチ側のハッシュアルゴリズムで全ノードへ均等ロードバランスさせる標準構成。
    * **Individual Interface Mode (L3 Only)**: 各クラスタノードが独自の IP/MAC アドレスを持ち、ルーティング（ECMP / PBR）や外部ロードバランサでトラフィックを分散させる構成（ASA のみサポート、FTD では Spanned モードが基本）。
* **利用目的**:
  * 機器故障（Hardware failure）、インターフェイス障害（Link failure）、ソフトウェアクラッシュ時における瞬時（1秒未満〜数秒）のフェールオーバーによる事業継続性の確保。
  * ディープパケットインスペクション（Snort IPS / AMP）処理におけるファイアウォール群のスループット拡張とボトルネックの解消。
* **どのような場面で利用するか**:
  * エンタープライズ Web エッジ、データセンターコア境界、クラウド相互接続点（DCI）、Service Provider エッジ。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **評価対象技術** | **ASA Active/Standby Failover**、**ASA Active/Active Failover**、**FTD Active/Standby HA**、**ASA/FTD Clustering** |
| **最大構成台数** | Failover: 2台ペア<br>Clustering: 最大 16台 (Firepower 4100/9300, ASA 5585-X 等) |
| **管理基盤** | ASA: CLI / ASDM / CSM<br>FTD: FMC (Firepower Management Center) / FDM / FXOS (FCM) |
| **クラスタ間通信軸** | **Cluster Control Link (CCL)**: 制御（選定・設定同期・ヘルスチェック）＋ データ（ステート複製・フォワードパケット転送） |
| **データインターフェイス** | **Spanned EtherChannel** (cLACP 利用、L2/L3 対応) / **Individual Interfaces** (ASA L3 Routed のみ) |
| **クラスタノード役割** | **Control Node (Master)**: 全体の設定同期・統括<br>**Data Node (Slave)**: データ処理担当<br>**Flow Role**: Owner, Director, Forwarder |
| **FTD 独自制限** | FTD は **Active/Active Failover 非対応**。また FTD クラスタでは **Remote Access VPN 非対応**。 |
| **必須前提条件** | 同一モデル、同一 RAM/Flash、同一インターフェイス構成、同一 OS バージョン、同一 Firewall Mode (Routed/Transparent) |

---

## 🏗 動作原理

### 1. ASA / FTD Clustering のトラフィックフロー (Owner / Director / Forwarder)

クラスタリングでは、非対称ルーティング環境下でも接続の整合性（Flow Symmetry）を保つため、フローごとに 3 つの役割が動的に割り当てられます。

```text
[ Client ] ───(1) Packet 1 (SYN) ───► [ Switch (Spanned EtherChannel) ]
                                                │ (LACP Hash sends to Node A)
                                                ▼
                                    [ Node A (Owner) ] ◄──────────────────────────────┐
                                       ├── Creates Connection State                   │
                                       ├── Hash (Conn ID) ──► Assigns Node B as Director │
                                       └── Sends State Sync via CCL ─────────────────┼─┐
                                                                                     │ │
┌────────────────────────────────────────────────────────────────────────────────────┼─┼┐
│ Cluster Control Link (CCL)                                                         │ ││
└────────────────────────────────────────────────────────────────────────────────────┼─┼┘
                                                                                     │ │
                                    [ Node B (Director) ] ◄──────────────────────────┘ │
                                       └── Stores Backup State & Owner Mapping (Node A)│
                                                                                       │
[ Server ] ◄─── (2) Forwarded SYN ────── [ Node A (Owner) ]                            │
    │                                                                                  │
    └─── (3) Packet 2 (SYN-ACK) ───► [ Switch ] ──► (Hash sends to Node C)             │
                                                        │                              │
                                                        ▼                              │
                                            [ Node C (Forwarder) ]                     │
                                               ├── Queries Director (Node B) via CCL ──┘
                                               └── Redirects Packet via CCL ──► [ Node A (Owner) ]
                                                                                       │
[ Client ] ◄─── (4) Forwarded SYN-ACK ───────────────────────────────────────── [ Node A (Owner) ]
```

---

### 2. ASA Active/Active Failover (Failover Group 1 ＆ 2) アーキテクチャ

Multi-Context Mode で 2 つの Failover Group を定義し、互いのコンテキストを相互クロスバックアップ（Active / Standby）させる構造：

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│ ASA-1 (Physical Appliance)                                                       │
│   ├── Context 1 (Sales)   ──► Failover Group 1 (Primary - ACTIVE)                 │
│   └── Context 2 (Finance) ──► Failover Group 2 (Secondary - STANDBY)             │
└──────────────────────────────────────────────────────────────────────────────────┘
            │                                                         │
            ├── Failover LAN Link (Heartbeat & Config Sync) ──────────┤
            └── Stateful Failover Link (Session State Sync) ──────────┤
                                                                      │
┌─────────────────────────────────────────────────────────────────────┴────────────┐
│ ASA-2 (Physical Appliance)                                                       │
│   ├── Context 1 (Sales)   ──► Failover Group 1 (Primary - STANDBY)               │
│   └── Context 2 (Finance) ──► Failover Group 2 (Secondary - ACTIVE)              │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

1. **クラスタブートストラップ ＆ Master (Control) 選定シーケンス**:
   * ノード起動時、Cluster Control Link (CCL) 上で既存の Control (Master) ノードが存在するかブロードキャスト検索（Keepalive 探索）を実行。
   * Master が不在の場合、45 秒間待機した後に最も優先度の高いノード（Priority 数値が最小のノード）が Master (Control) ノードとして選出される。
   * 既存 Master が存在する場合、起動ノードは Data (Slave) ノードとしてクラスタに参加し、Master からシステム設定を一括取得（Bulk Sync）する。
2. **新規フロー開始時における処理シーケンス (TCP SYN パケット着信)**:
   * スイッチの Spanned EtherChannel ハッシュ（または ECMP）により、パケットが特定ノード（例: Node A）に着信。
   * **Node A (Owner 化)**: パケットを直接受信した Node A が接続の **Owner** となり、自らのローカル接続テーブルにセッションを作成。
   * **Director の選定 ＆ 状態同期**: 5-Tuple ハッシュ値に基づき、クラスタ内の別のノード（例: Node B）が **Director** として決定される。
   * Node A は CCL 経由で Node B (Director) へ接続メタデータ（Owner は Node A である旨）とセッション状態をリアルタイム転送・登録する。
3. **戻りトラフィック非対称着信時における転送シーケンス (SYN-ACK パケット着信)**:
   * 戻りパケットがスイッチハッシュによって別のノード（例: Node C）に着信。
   * **Node C (Forwarder 化)**: Node C は自らのセッションテーブルに存在しないパケットを受信したため **Forwarder** となる。
   * Node C は CCL 経由で Director (Node B) へ問い合わせ（Query）を行うか、ハッシュから即座に Node B を特定。
   * Node C はパケットを破棄せず、CCL トンネル経由で本物の Owner である Node A へそのままパケットを転送（Redirect）する。
   * Node A がインスペクションおよび NAT 変換を完了させ、送信元へパケットを出力する。
4. **ノード障害発生時におけるフェールオーバーシーケンス**:
   * 既存の Owner (Node A) がハードウェアダウンまたは CCL 障害を起こす。
   * Master および他 Data ノードは、3秒間（デフォルト Holdtime）キープアライブを喪失した Node A をクラスタテーブルから除外（Evict）する。
   * 以前 Node A が Owner であったフローのパケットが Node C に着信した場合、Node C は Director (Node B) が保持していたバックアップステートを引き継ぎ、新たな Owner として透過的にセッションを処理し続ける。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **ASA Active/Standby ＆ Active/Active Failover 設定要件**:
   * **Failover LAN ＆ Stateful Link の設定**: `failover lan unit primary|secondary`、`failover lan interface <NAME> <IF>`、`failover link <NAME> <IF>`、`failover interface ip <NAME> <PRI_IP> <NETMASK> standby <STBY_IP>`。
   * **HTTP State Replication**: デフォルトでは HTTP セッションは同期されません。要件に「Ensure HTTP connections survive failover」とある場合、**`failover replication http`** コマンドの投入が必須です。
   * **Interface Health Monitoring**: 全物理ポートが監視されますが、特定のサブインターフェイスやバックアップポートを明示監視・非監視にする場合は **`monitor-interface <IF>`** / **`no monitor-interface <IF>`** を適切に切り替えます。
   * **Active/Active Preemption**: `failover group 1` / `failover group 2` 内で **`preempt [delay]`** を設定し、障害復旧後に優先ノードへアクティブ権限を自動書き戻し（Failback）させる設定が問われます。
2. **ASA / FTD Clustering - Spanned EtherChannel ＆ cLACP の罠**:
   * **cLACP System ID / Priority**: スイッチ側から見て全クラスタノードが「1台の論理スイッチ」に見えるよう、Spanned EtherChannel では **`lacp system-id <MAC>`** または自動定義される cLACP ID を共有します。
   * **スイッチ側の設定要件**: スイッチ側ポートは必ず **1つの巨大な Port-Channel**（例: Port-channel 10 に全ノードのメンバーポートを入れる）に統合し、`channel-group 10 mode active` に設定します。※間違えてノードごとに別々の Port-Channel を作るとクラスタデータ面が不整合を起こします。
   * **CCL (Cluster Control Link) の設定ルール**: CCL 自体は**絶対 Spanned EtherChannel にしてはいけません**。各ノード個別の Local EtherChannel（例: ノードごとの Port-channel 48）として独立させ、スイッチ側もノードごとに個別の Port-Channel で終端させます。
3. **Cluster Control Link (CCL) MTU サイズ要件（必須落とし穴）**:
   * CCL 上では、データパケット（1500 バイト）にクラスタヘッダー（カプセル化オーバーヘッド）が付加されて転送されます。
   * **MTU ルール**: CCL の MTU は、データインターフェイスの最大 MTU より **最低 100 バイト以上大きく（推奨 9184 バイト / ジャンボフレーム）** 設定する必要があります。スイッチ側ポートも `system mtu jumbo 9000` や `mtu 9216` の設定が必要です。
4. **FTD Clustering の制限事項 ＆ Centralized Features**:
   * **Remote Access VPN**: FTD クラスタでは RA VPN (AnyConnect) は完全非サポートです。
   * **Site-to-Site VPN**: 中央制御機能（Centralized Feature）として Control Node 上でのみ動作します。
   * **FlexConfig**: GUI 未対応の ASA 特有コマンド（例: WCCP など）をクラスタ環境へ導入する場合、FlexConfig Policy 経由で流し込む必要があります。
5. **Failover / Cluster 状態判定・トラブルシュートコマンド**:
   * **`show failover` / `show failover state`**: Failover の同期状態（`Active`, `Standby Ready`, `Sync Done`）を確認。
   * **`show cluster info` / `show cluster info health`**: クラスタノードの選定状態（`MASTER`, `SLAVE`）、参加・脱落履歴を確認。
   * **`show cluster info transport asp`**: データプレーンの CCL 転送パケット・ドロップ統計を確認。
   * **`cluster exec <COMMAND>`**: Master から全 Slave ノードへ一括コマンド発行（例: `cluster exec show conn`）。

---

## 🛠 設定方法

### 1. ASA CLI: Active/Standby Stateful Failover 設定 (完全版)

```bash
! ===== Primary ASA (ASA-1) Configuration =====
failover lan unit primary
failover lan interface FO_LINK GigabitEthernet0/3
failover link STATE_LINK GigabitEthernet0/4

failover interface ip FO_LINK 192.168.254.1 255.255.255.252 standby 192.168.254.2
failover interface ip STATE_LINK 192.168.255.1 255.255.255.252 standby 192.168.255.2
failover key cisco123
failover replication http               ! HTTP セッションのステート同期を有効化

interface GigabitEthernet0/3
 no shutdown
interface GigabitEthernet0/4
 no shutdown

failover                                ! Failover 有効化

! ===== Secondary ASA (ASA-2) Bootstrap Configuration =====
! (※ Secondary 側は Failover インターフェイス基本定義のみで全設定が自動同期される)
failover lan unit secondary
failover lan interface FO_LINK GigabitEthernet0/3
failover link STATE_LINK GigabitEthernet0/4

failover interface ip FO_LINK 192.168.254.1 255.255.255.252 standby 192.168.254.2
failover interface ip STATE_LINK 192.168.255.1 255.255.255.252 standby 192.168.255.2
failover key cisco123

interface GigabitEthernet0/3
 no shutdown
interface GigabitEthernet0/4
 no shutdown

failover
```

---

### 2. ASA CLI: Active/Active Multi-Context Failover ＆ Preempt 設定

```bash
! ===== Primary ASA (ASA-1) Multi-Context Setup =====
mode multiple                           ! Multi-context モードへ変更 (要リロード)

! System Context にて Failover Group の設定
failover group 1
 primary
 preempt 30                             ! 復号時 30 秒後に Preempt (Failback)
failover group 2
 secondary
 preempt 30

context CTX_SALES
 allocate-interface GigabitEthernet0/0
 config-url disk0:/CTX_SALES.cfg
 join-failover-group 1                  ! Group 1 (ASA-1 で Active)

context CTX_FINANCE
 allocate-interface GigabitEthernet0/1
 config-url disk0:/CTX_FINANCE.cfg
 join-failover-group 2                  ! Group 2 (ASA-2 で Active)

failover lan unit primary
failover lan interface FO_LINK GigabitEthernet0/3
failover interface ip FO_LINK 10.10.10.1 255.255.255.0 standby 10.10.10.2
failover key cisco123
failover
```

---

### 3. ASA CLI: ASA Clustering (Spanned EtherChannel モード)

```bash
! ===== Master ASA (ASA-1) Configuration =====
cluster interface-mode spanned force    ! データ面を Spanned EtherChannel に指定

! Cluster Control Link (CCL) インターフェイス設定
interface Port-channel48
 description Cluster Control Link
 port-channel min-links 1
 member-interface GigabitEthernet0/2
 member-interface GigabitEthernet0/3

! Spanned Data EtherChannel インターフェイス設定
interface Port-channel1
 port-channel span-cluster
 nameif inside
 security-level 100
 ip address 10.1.1.1 255.255.255.0
 member-interface GigabitEthernet0/0
 member-interface GigabitEthernet0/1

! クラスタグループ設定
cluster group CLUSTER_ASA
 local-unit ASA-1
 cluster-interface Port-channel48 ip 192.168.100.1 255.255.255.0
 priority 1                             ! 最小値が最高優先度 (Master 優先)
 key cisco123
 enable

! ===== Slave ASA (ASA-2) Configuration =====
cluster interface-mode spanned force

interface Port-channel48
 member-interface GigabitEthernet0/2
 member-interface GigabitEthernet0/3

cluster group CLUSTER_ASA
 local-unit ASA-2
 cluster-interface Port-channel48 ip 192.168.100.2 255.255.255.0
 priority 10
 key cisco123
 enable
```

---

### 4. FMC GUI / FXOS: FTD High Availability (Active/Standby) ＆ Clustering 構成

```text
[ Step 1: FMC GUI での FTD Active/Standby HA ペア作成 ]
1. FMC UI から Devices > Device Management に移動。
2. 右上の "Add" ドロップダウンから "Add High Availability" を選択。
3. パラメータ入力:
   - Name: FTD_HA_PAIR
   - Device Type: Firepower Threat Defense
   - Primary Device: FTD-Primary
   - Secondary Device: FTD-Secondary
4. "Continue" をクリックし、HA リンクを設定:
   - High Availability Interface: GigabitEthernet0/3 (IP: 192.168.200.1 / Standby: 192.168.200.2)
   - Stateful Link Interface: GigabitEthernet0/4 (IP: 192.168.201.1 / Standby: 192.168.201.2)
   - Encryption Key: cisco123
5. "Build HA" を押してペアを創出 ➔ Deploy 実行。

[ Step 2: FXOS / FMC での FTD Inter-Chassis Clustering 構成 ]
1. FXOS (Firepower Chassis Manager) にログイン:
   - Logical Devices ➔ "Add Device"
   - Device Name: FTD_CLUSTER_NODE1
   - Template: Firepower Threat Defense
   - Enable Clustering: チェックを入れる (Cluster Name: FTD_CLUSTER)
   - Cluster Control Link: Port-channel 48 を指定 (Local EtherChannel)
   - Data Interfaces: Spanned Mode として Port-channel 1 (Inside), Port-channel 2 (Outside) を追加。
2. FMC UI に移動:
   - Devices > Device Management ➔ "Add Cluster"
   - Cluster Name: FTD_CLUSTER
   - Control Node を登録後、Data Node 2, 3 をグループへ追加。
   - MTU Config: CCL Interface (Port-channel 48) の MTU を 9184 バイトに手動変更 ➔ Deploy。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (ASA / FTD CLI) | 表示確認ポイント |
| :--- | :--- | :--- |
| **Failover 動作状態 ＆ モニターポート確認** | <code>show failover</code> / <code>show failover state</code> | `This host: Primary - Active`, `Other host: Secondary - Standby Ready`, Monitored Interfaces 状態 |
| **Failover インターフェイス ＆ IP アドレス確認** | <code>show failover interface</code> | LAN Link および Stateful Link の My IP / Other IP アドレスの対比 |
| **クラスタノード一覧 ＆ 選定役割確認** | <code>show cluster info</code> | `This is "node1" in state MASTER`, 他メンバーノードの `SLAVE` 状態と CCL IP |
| **クラスタ全体のヘルスチェック状態確認** | <code>show cluster info health</code> | ユニット、物理ポート、CCL、cLACP のヘルスモニタリングステータス |
| **クラスタ間データ転送・ASP ドロップ統計確認** | <code>show cluster info transport asp</code><br><code>show asp cluster counter</code> | CCL 経由で転送された Forwarded / Redirected パケット数およびエラー |
| **全クラスタノードでの接続一括表示** | <code>cluster exec show conn</code> | 各ノードでのフロー役割（`Y`: Director, `z`: Forwarder, 無し: Owner） |
| **Spanned EtherChannel ＆ cLACP 状態確認** | <code>show port-channel summary</code><br><code>show lacp cluster system-mac</code> | ポートチャネルの `SPAN-CLUSTER` フラグ、cLACP System ID の整合性 |
| **Failover ＆ Cluster デバッグログ追跡** | <code>debug failover event</code><br><code>debug cluster ccp</code><br><code>debug cluster datapath</code> | 制御プレーン（CCP）のメッセージ交換、ノード参加/離脱メッセージ |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **Failover ペアを作成しようとしたが、`Version Mismatch` や `Hardware Mismatch` エラーで失敗する** | 2台の ASA/FTD 間で OS バージョン、VDB、RAM 容量、またはインターフェイスモジュール構成が不一致 | <code>show version</code><br><code>show failover state</code> | 2台のアプライアンスの OS バージョン、パッチ、VDB、およびハードウェア拡張カードを完全に同一に揃える。 |
| **新しくクラスタに参加させようとした Slave ノードが `Cluster Join Failed` となり離脱（Evict）する** | Cluster Control Link (CCL) の **MTU サイズが不足** している（データ MTU ＋ オーバーヘッド未考慮） | <code>show cluster info trace</code><br><code>show interface</code> | CCL (Port-channel 48) の MTU を **9184**（データ MTU より +100 バイト以上）に変更し、スイッチ側ポートも Jumbo Frame を許可する。 |
| **Spanned EtherChannel のデータポートが Up にならず、トラフィックが通過しない** | 上流/下流スイッチ側で全ノードのメンバーポートを **1つの同一 Port-Channel** として構成していない | <code>show port-channel summary</code><br><code>show lacp cluster</code> | スイッチ側で全ノードの接続ポートを1つの EtherChannel（`channel-group X mode active`）にまとめ、cLACP を合致させる。 |
| **クラスタ環境で大きな TCP パケットや HTTP トラフィックの応答が極端に遅延・ドロップする** | CCL 上での非対称フラグメントパケットの順序逆転（Out-of-order）または CCL バンド幅不足 | <code>show asp cluster counter</code><br><code>show cluster info transport asp</code> | CCL を 10Gbps/40Gbps の Port-Channel（複数物理ポートの冗長化）で帯域拡張し、低遅延（<20ms RTT）な直結線で構成する。 |
| **Failover 発生時、TCP セッションは維持されるが HTTP/Web 通信のみ切断される** | Stateful Failover において HTTP ステート同期（`failover replication http`）がデフォルト無効のままになっている | <code>show running-config failover</code> | `failover replication http` コマンドを投入し、HTTP セッションテーブルの動的同期を有効化する。 |
| **Active/Active Failover で特定コンテキストの Failback が発生しない** | Failover Group で `preempt` オプションが設定されていない | <code>show failover group</code> | 各 Failover Group に `preempt <delay>` を追加定義する。 |

---

## ⚠ 制限事項

* **FTD における Active/Active Failover の完全非サポート**:
  * FTD ソフトウェア（全モデル）は **Active/Standby 冗長のみをサポート** します。Active/Active モードは ASA Multiple Context Mode のみの機能です。
* **FTD Clustering における Remote Access VPN (AnyConnect) 非対応**:
  * FTD クラスタ環境では Remote Access VPN 機能は構成できません（Site-to-Site IPsec VPN は Control Node 上で Centralized 機能として利用可能）。
* **暗号化 TLS/SSL セッションのステート未同期**:
  * SSL Inspection (Managed MitM 復号) を行っている暗号化セッションは、ノード障害時に復号コンテキスト（TLS 鍵情報）が別ノードへ同期されません。ノードフェールオーバー発生時、既存の復号 TLS 通信はリセットされ再ハンドシェイクが必要となります（※非復号トラフィックは正常にステート同期されます）。

---

## 🔄 他技術との関連

* **LACP / cLACP (Cluster LACP)**: Spanned EtherChannel を構成し、スイッチと複数クラスタノード間で動的に Port-Channel を維持するための拡張プロトコル。
* **Multi-Context Mode (ASA)**: 1台の ASA を仮想的に複数分割し、Active/Active Failover (Failover Group 1 ＆ 2) を構築するための前提アーキテクチャ。
* **FXOS (Firepower Extensible OS)**: Firepower 4100 / 9300 シャーシ上で、FTD インスタンスを Logical Device として割り当て、シャーシレベルの CCL や Spanned インターフェイスを割り当てる基盤 OS。
* **FlexConfig Policy (FMC)**: FMC GUI で直接設定できない ASA 特有のクラスタコマンドやパラメータを補填・適用するための設定メカニズム。

---

## 🧩 比較表

### 1. ASA Failover A/S vs ASA Failover A/A vs ASA/FTD Clustering

| 比較項目 | ASA Active/Standby | ASA Active/Active | ASA / FTD Clustering |
| :--- | :--- | :--- | :--- |
| **対応動作モード** | Single / Multi-Context | **Multi-Context モード必須** | Routed / Transparent Mode |
| **最大ノード数** | 2台 | 2台 | **最大 16台** |
| **スループレット** | 1台分 (Active 側のみ) | 最大 2台分 (アクティブ分配時) | **N台分の線形拡張 (Scale-out)** |
| **FTD 対応** | **対応** (FTD 唯一の HA 形態) | **非対応** (FTD 未サポート) | **対応** (Firepower 4100/9300 等) |
| **インターフェイス構成**| Active / Standby 共有 IP | Context 別 Active / Standby IP | **Spanned EtherChannel** / Individual |
| **フロー役割概念** | Active / Standby | Active / Standby (Group 毎) | **Owner / Director / Forwarder** |

### 2. Spanned EtherChannel Mode vs Individual Interface Mode (Clustering)

| 比較項目 | Spanned EtherChannel Mode | Individual Interface Mode |
| :--- | :--- | :--- |
| **動作レイヤー** | Layer 2 (Bridged) / Layer 3 (Routed) | **Layer 3 (Routed) のみ** |
| **IP/MAC アドレス** | 全ノードで **1つの IP / MAC** を論理共有 | ノードごとに **個別の IP / MAC** を保有 |
| **外部スイッチ構成** | スイッチ側で全ノードを1つの EtherChannel に統合 | ノードごとに独立した Routed L3 ポート接続 |
| **ロードバランス方式**| スイッチの LACP (cLACP) ハッシュアルゴリズム | 外部ルーターの **ECMP** または **PBR** |
| **推奨運用** | **Cisco 推奨の標準アーキテクチャ** | 地理的分散（Site-to-Site Cluster）等での特殊用途 |

---

## 💡 ベストプラクティス

1. **Cluster Control Link (CCL) の設計と帯域・MTU 確保**:
   * CCL はクラスタの命綱（Heartbeat ＋ 状態同期 ＋ パケット転送）です。物理的に独立した複数リンクを Port-Channel（例: Port-channel 48）として冗長化し、MTU はジャンボフレーム（**9184 バイト**）を明示指定してパケット断片化（Fragmentation）を防止します。
2. **Prompt 表示のカスタムによる誤操作防止**:
   * Failover や Cluster の CLI にログインした際、自機が Active/Standby、あるいは Master/Slave のどちらであるかを一目で認識できるよう、プロンプト表示を変更します：
     * ASA Failover: `prompt hostname state priority`
     * ASA/FTD Cluster: `prompt cluster-unit state`
3. **HTTP ステート複製と Interface Health Monitoring の個別最適化**:
   * Stateful Failover では `failover replication http` を追加投入して Web セッション切断を防ぎます。また、管理専用ポート（Management）などの非データポートに対しては `no monitor-interface management` を設定し、管理回線のトラブルによる誤フェールオーバーを排除します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA Active/Standby Stateful Failover (CLI 完全設定)

#### 問題
ASA1 (Primary) と ASA2 (Secondary) 間で Stateful Failover を構成せよ。`Gi0/3` を Failover LAN リンク、`Gi0/4` を Stateful リンクとして割り当て、暗号化キー `cisco123` を設定せよ。また HTTP セッションの同期を有効化せよ。

#### 要件
* Primary LAN IP: `10.254.254.1/30`, Secondary LAN IP: `10.254.254.2/30`
* Primary State IP: `10.255.255.1/30`, Secondary State IP: `10.255.255.2/30`

#### 設定例

```bash
! ===== ASA1 Configuration =====
failover lan unit primary
failover lan interface FO_LAN GigabitEthernet0/3
failover link FO_STATE GigabitEthernet0/4
failover interface ip FO_LAN 10.254.254.1 255.255.255.252 standby 10.254.254.2
failover interface ip FO_STATE 10.255.255.1 255.255.255.252 standby 10.255.255.2
failover key cisco123
failover replication http

interface GigabitEthernet0/3
 no shutdown
interface GigabitEthernet0/4
 no shutdown

failover

! ===== ASA2 Configuration =====
failover lan unit secondary
failover lan interface FO_LAN GigabitEthernet0/3
failover link FO_STATE GigabitEthernet0/4
failover interface ip FO_LAN 10.254.254.1 255.255.255.252 standby 10.254.254.2
failover interface ip FO_STATE 10.255.255.1 255.255.255.252 standby 10.255.255.2
failover key cisco123

interface GigabitEthernet0/3
 no shutdown
interface GigabitEthernet0/4
 no shutdown

failover
```

#### 検証方法
* `show failover` を実行し、`This host: Primary - Active` および `Other host: Secondary - Standby Ready` と表示され、`http` レプリケーションが有効になっていることを確認する。

---

### 例2: ASA Active/Active Multi-Context Failover と Failover Group の設定

#### 問題
ASA1 と ASA2 において Multi-Context モードで Active/Active Failover を構成せよ。`CTX_1` を `failover group 1`（ASA1 が Active）、`CTX_2` を `failover group 2`（ASA2 が Active）に割り当て、双方に 30 秒の `preempt` を設定せよ。

#### 要件
* `failover group 1` / `2` を使用すること。

#### 設定例

```bash
! ===== ASA1 System Context Configuration =====
mode multiple

failover group 1
 primary
 preempt 30
failover group 2
 secondary
 preempt 30

context CTX_1
 allocate-interface GigabitEthernet0/0
 config-url disk0:/CTX_1.cfg
 join-failover-group 1

context CTX_2
 allocate-interface GigabitEthernet0/1
 config-url disk0:/CTX_2.cfg
 join-failover-group 2

failover lan unit primary
failover lan interface FO_LAN GigabitEthernet0/2
failover interface ip FO_LAN 172.16.254.1 255.255.255.0 standby 172.16.254.2
failover key cisco123
failover
```

#### 検証方法
* `show failover group` を実行し、Group 1 が ASA1 で Active、Group 2 が ASA2 で Active となっていることを確認する。

---

### 例3: ASA Clustering - Spanned EtherChannel モードの構築

#### 問題
2台の ASA（`ASA-1`, `ASA-2`）で Spanned EtherChannel モードのクラスタ `CLUSTER_LAB` を構築せよ。CCL に `Port-channel 48`（IP: `192.168.10.1` / `10.2`）、データ面に `Port-channel 1`（Inside）を割り当てよ。

#### 要件
* ASA-1 の Priority を `1` (Master 優先)、ASA-2 を `10` に設定すること。

#### 設定例

```bash
! ===== ASA-1 (Master) =====
cluster interface-mode spanned force

interface Port-channel48
 member-interface GigabitEthernet0/2
 member-interface GigabitEthernet0/3

interface Port-channel1
 port-channel span-cluster
 nameif inside
 security-level 100
 ip address 10.10.10.1 255.255.255.0
 member-interface GigabitEthernet0/0
 member-interface GigabitEthernet0/1

cluster group CLUSTER_LAB
 local-unit ASA-1
 cluster-interface Port-channel48 ip 192.168.10.1 255.255.255.0
 priority 1
 key cisco123
 enable

! ===== ASA-2 (Slave) =====
cluster interface-mode spanned force

interface Port-channel48
 member-interface GigabitEthernet0/2
 member-interface GigabitEthernet0/3

cluster group CLUSTER_LAB
 local-unit ASA-2
 cluster-interface Port-channel48 ip 192.168.10.2 255.255.255.0
 priority 10
 key cisco123
 enable
```

#### 検証方法
* `show cluster info` を実行し、`ASA-1 in state MASTER`、`ASA-2 in state SLAVE` と表示されることを確認する。

---

### 例4: Cluster Control Link (CCL) の MTU サイズ調整とモニタリング

#### 問題
クラスタ `CLUSTER_LAB` において、データパケットのカプセル化オーバーヘッドによるパケットドロップを防ぐため、CCL (Port-channel 48) の MTU を `9184` バイトに変更せよ。

#### 要件
* `mtu cluster` コマンドを使用すること。

#### 設定例

```bash
! ===== ASA Master Console =====
ciscoasa(config)# mtu cluster 9184
```

#### 検証方法
* `show cluster info health` および `show interface Port-channel48` を実行し、MTU が `9184` バイトに反映され、パケットドロップが発生していないことを確認する。

---

### 例5: FTD High Availability (Active/Standby) の FMC GUI 設定

#### 問題
FMC 管理下の 2台の FTD アプライアンス (`FTD-1`, `FTD-2`) において、FMC GUI から Active/Standby HA ペア `FTD_HA_PAIR` を作成せよ。

#### 要件
* HA Link: `Gi0/3` (`192.168.100.1` / `100.2`), Stateful Link: `Gi0/4` (`192.168.101.1` / `101.2`)。

#### 設定例

```text
[ FMC GUI Steps ]
1. FMC UI ➔ Devices > Device Management ➔ Add > Add High Availability
2. Name: FTD_HA_PAIR
3. Primary Device: FTD-1, Secondary Device: FTD-2
4. HA Interface: GigabitEthernet0/3 (Primary IP: 192.168.100.1, Secondary IP: 192.168.100.2, Netmask: 255.255.255.252)
5. State Interface: GigabitEthernet0/4 (Primary IP: 192.168.101.1, Secondary IP: 192.168.101.2, Netmask: 255.255.255.252)
6. Encryption Key: cisco123 ➔ Build HA ➔ Deploy.
```

#### 検証方法
* FMC の Device Management 画面で `FTD_HA_PAIR` が緑色の「In Sync」状態で表示され、Primary が Active となっていることを確認する。

---

### 例6: ASA Cluster における特定非監視ポート (Management) の設定

#### 問題
ASA クラスタ環境において、管理専用ポート `Management0/0` のリンクダウンが原因でクラスタ脱落（Eviction）が発生しないよう、当該ポートのヘルスモニタリングを無効化せよ。

#### 要件
* `no monitor-interface` コマンドを使用すること。

#### 設定例

```bash
ciscoasa(config)# interface Management0/0
ciscoasa(config-if)# no monitor-interface
```

#### 検証方法
* `show failover` または `show cluster info health` を実行し、`Monitored Interfaces` の一覧から `Management0/0` が除外されていることを確認する。

---

### 例7: FTD クラスタノード脱落時の自動再参加 (Auto-Rejoin) パラメータ設定

#### 問題
FTD クラスタにおいて、データインターフェイスの障害によってクラスタから除外されたノードが、自動再参加を試みる回数・間隔のパラメータを確認・チューニングせよ。

#### 要件
* `show cluster info auto-join` コマンドでデフォルト動作を確認すること。

#### 設定例

```bash
> show cluster info auto-join
```

#### 検証方法
* データインターフェイス障害時の再参加試行が `5分, 10分, 20分`（計3回）に設定されていることを確認する。

---

### 例8: ASA Cluster での全ノード接続状態一括確認 (cluster exec)

#### 問題
Master ノードの CLI から `cluster exec` コマンドを発行し、全クラスタノード上のアクティブ接続テーブルを一覧表示させよ。

#### 要件
* `cluster exec show conn` コマンドを使用すること。

#### 設定例

```bash
ciscoasa# cluster exec show conn
```

#### 検証方法
* 出力結果において、接続ごとに `Owner`、`Director (Y)`、`Forwarder (z)` のフラグが各ノードに正しく割り当てられていることを確認する。

---

### 例9: ASA Active/Active における Asymmetric Routing Support (ASR) Group 設定

#### 問題
ASA Active/Active Failover 環境において、非対称ルーティングによって戻りパケットが別コンテキストへ着信した場合にドロップされないよう、ASR Group `1` を定義してインターフェイスへ適用せよ。

#### 要件
* `asr-group` コマンドを使用すること。

#### 設定例

```bash
! ===== Context 1 Configuration =====
interface GigabitEthernet0/0.10
 nameif inside
 ip address 10.1.1.1 255.255.255.0
 asr-group 1

! ===== Context 2 Configuration =====
interface GigabitEthernet0/1.10
 nameif inside2
 ip address 10.1.1.2 255.255.255.0
 asr-group 1
```

#### 検証方法
* `show asr-group` を実行し、非対称フローのデータが対面コンテキストへ正常リダイレクトされていることを確認する。

---

### 例10: FMC REST API を用いた FTD HA ペア状態の取得

#### 問題
FMC REST API を使用して、登録されている FTD High Availability ペアのステータス情報を CURL コマンドで取得せよ。

#### 要件
* エンドポイント `/api/fmc_config/v1/domain/{domainUUID}/deviceha/ftddevicehas` を使用すること。

#### 設定例

```bash
curl -k -X GET "https://fmc.corp.local/api/fmc_config/v1/domain/e276abec-e0f2-11e3-8169-6d9ed13b206f/deviceha/ftddevicehas" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json"
```

#### 検証方法
* JSON レスポンス内に `primary` および `secondary` ノードの UUID と `In Sync` ステータスが含まれていることを確認する。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 以下のコンフィグが投入された ASA1 の状態に関する記述として最も適切なものはどれか。

```bash
failover group 1
 primary
 preempt 30
failover group 2
 secondary
 preempt 30

context CTX_A
 join-failover-group 1
context CTX_B
 join-failover-group 2
```

* A) ASA1 は Single Context Mode で動作しており、`CTX_A` のみがアクティブになる。
* B) ASA1 は Multi-Context Mode で動作しており、正常時は `CTX_A` が ASA1 で Active、`CTX_B` が Standby となる。ASA1 再起動後も 30 秒後に `CTX_A` のアクティブ権限を自動復旧 (Preempt) する。
* C) FTD アプライアンス上で Active/Active HA が構成されている。
* D) `preempt 30` は Failover LAN リンクのタイムアウト時間を表している。

**正解**: **B**  
**解説**: コンフィグは **ASA Active/Active Failover** の典型的な定義です。Multi-Context Mode 上で `Failover Group 1`（Primary 指定）に属する `CTX_A` は ASA1 で Active となり、`preempt 30` により障害復旧から 30 秒後にアクティブ権限が自動的に戻されます。

---

### 2. トラブルシュート問題

**問題**: Firepower 4100 上で FTD クラスタを構築し、FMC からノード 2 を追加しようとしたが、`Cluster Join Failed` エラーが発生してノード 2 がクラスタから即座に除外（Evict）された。FTD CLI で `show interface` を確認したところ、データポートの MTU は `9000` バイト、CCL (Port-channel 48) の MTU は `1500` バイトに設定されていた。原因および修正方法として正しいものはどれか。

* A) FTD クラスタでは Spanned EtherChannel が未サポートであるため。
* B) CCL (Cluster Control Link) の MTU がデータポートの MTU より小さいためパケットが断片化・ドロップした。CCL の MTU を `9184` バイトに引き上げ、スイッチ側ポートも Jumbo Frame を許可する。
* C) FMC のライセンスが Threat ライセンスのみであるため。
* D) FTD では Active/Standby 以外の HA 構成が不可能なため。

**正解**: **B**  
**解説**: クラスタ制御リンク (CCL) は、データパケット（最大 9000 バイト）をクラスタヘッダーでカプセル化して転送します。そのため **CCL の MTU はデータポートより最低 100 バイト以上大きく（推奨 9184 バイト）** 設定しなければクラスタ参加時のヘルスチェックおよびデータ転送に失敗して離脱します。

---

### 3. Design 問題

**問題**: データセンターの新規設計において、Cisco FTD アプライアンス群を用いてファイアウォール処理能力を段階的に 40 Gbps から 120 Gbps へ線形拡張 (Scale-out) しつつ、全ノードでアクティブにトラフィックを分散処理させたい。また、外部スイッチとは LACP を用いたシンプルな冗長構成としたい。最適なアーキテクチャはどれか。

* A) FTD Active/Active Failover ペア
* B) Firepower 9300 シャーシを用いた **FTD Clustering (Spanned EtherChannel モード)**
* C) FTDv を用いた ECMP ルーティング分散
* D) ASA Single Context Active/Standby HA

**正解**: **B**  
**解説**: FTD は Active/Active Failover をサポートしませんが、Firepower 4100/9300 上での **Clustering** を強力にサポートします。**Spanned EtherChannel モード** を採用することで、上流/下流スイッチからの LACP ハッシュによって複数ノードへのアクティブ分散と最大 16 台までのスループレット線形拡張（Scale-out）が実現します。

---

### 4. 実装問題

**問題**: FTD Active/Standby HA ペアを FMC 上で構成する際、システム要件として **誤っているもの** はどれか。

* A) 2台の FTD アプライアンスは完全に同一のハードウェアモデルである必要がある。
* B) 2台の FTD は同一の FTD ソフトウェアバージョンおよび VDB (Vulnerability Database) バージョンである必要がある。
* C) 2台の FTD は FMC 上の同一 Domain / Device Group に属している必要がある。
* D) HA 用の Failover Link および Stateful Link は、ユーザーデータパケットが通過する Data Subinterface と物理ポートを共有（兼任）しなければならない。

**正解**: **D**  
**解説**: FTD では、Failover Link および Stateful Link は**データインターフェイスと物理ポートを共有することは厳しく禁止されています**。独立した専用の物理インターフェイスまたは専用の Port-Channel を割り当てる必要があります。

---

### 5. コンフィグ読解問題

**問題**: ASA クラスタ環境において、Master ノードで `cluster exec show conn` を実行した。出力結果において、自ノード以外のノードに表示されている `Y` フラグおよび `z` フラグの意味として正しい組み合わせはどれか。

* A) `Y`: Owner, `z`: Standby
* B) `Y`: Director (セッション状態保持ノード), `z`: Forwarder (パケット受取・転送ノード)
* C) `Y`: Master, `z`: Slave
* D) `Y`: Blocked, `z`: Permitted

**正解**: **B**  
**解説**: `show conn` または `cluster exec show conn` の出力において、**`Y` は Director**（接続のバックアップ状態と Owner マッピングを保持するノード）、**`z` は Forwarder**（パケットを受信し CCL 経由で Owner へ転送するノード）を表します。

---

## 🔗 参考リソース

* [Cisco ASA Series General Operations Configuration Guide: High Availability and Failover]([suspicious link removed])
* [Cisco ASA Series General Operations Configuration Guide: Clustering]([suspicious link removed])
* [Cisco Secure Firewall Management Center Device Configuration Guide: High Availability]([suspicious link removed])
* [Cisco Secure Firewall Management Center Device Configuration Guide: Clustering]([suspicious link removed])
* [Cisco Technical Notes: Troubleshoot ASA and FTD Clustering Issues]([suspicious link removed])
* [Cisco Live: Advanced ASA and FTD Clustering Architecture and Troubleshooting (BRKSEC-3032)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で Clustering ＆ HA が出題された場合の迅速な構成判断フローチャート：
    * **「Configure Failover with 2 ASAs in Single Mode」** ➔ ASA Active/Standby Failover (`failover lan unit primary|secondary` ＋ `failover link` ＋ `failover replication http`)。
    * **「Configure Failover with 2 ASAs active simultaneously」** ➔ ASA Active/Active Multi-Context Failover (`mode multiple` ＋ `failover group 1|2` ＋ `join-failover-group`)。
    * **「Scale throughput across multiple ASA/FTD units」** ➔ ASA/FTD Clustering (Spanned EtherChannel Mode ＋ cLACP ＋ CCL MTU 9184)。
* **図解（Clustering パケット処理パイプライン）**:
  ```text
  [ Ingress Packet (SYN) ] ──► [ Switch LACP Hash ] ──► [ Node A (Owner) ]
                                                            │
                                              Creates State & Selects Director
                                                            │
                                             CCL Sync ──────┼──────► [ Node B (Director) ]
                                                            │           (Stores Backup State)
  [ Egress Packet ] ◄───────────────────────────────────────┘

  [ Return Packet (SYN-ACK) ] ──► [ Switch Hash ] ──► [ Node C (Forwarder) ]
                                                          │
                                           Redirect via CCL
                                                          │
                                                          ▼
                                                   [ Node A (Owner) ] ──► [ Egress to Client ]
  ```
* **注意点**:
  * ラボ試験で Clustering や HA を設定した後は、`show failover`、`show cluster info`、`show cluster info health`、`show interface` を必ず実行し、全モニタポートが `Normal`、ノード状態が `In Sync / MASTER / SLAVE` となっていることを確認してください。
