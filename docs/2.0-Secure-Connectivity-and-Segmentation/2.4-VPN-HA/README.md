---
layout: default
title: 2.4-VPN-HA
nav_order: 4
parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.4 VPN high availability methods

Cisco ネットワークおよびセキュリティインフラストラクチャにおける **VPN High Availability（VPN 高可用性・冗長化）** 技術は、Site-to-Site VPN（IPsec L2L、DMVPN、FlexVPN）および Remote Access VPN（AnyConnect / Cisco Secure Client）において、回線障害、アプライアンス障害、あるいは拠点障害が発生した場合でも通信を停止させず、無中断または最小限の切り替え時間でセッションを維持・復旧させるための包括的アーキテクチャ設計群です。

CCIE Security v6.1 Blueprint **2.4 VPN high availability methods** では、**Cisco ASA**、**Cisco FTD (Firepower Threat Defense / Secure Firewall)**、および **Cisco IOS / IOS-XE ルーター** における多様な高可用性メカニズムの深い理解と、要求仕様に応じた最適な冗長化デザインの実装・検証・トラブルシューティング能力が厳格に評価されます。

---

## 📘 概要

* **機能概要**:
  * **Remote Access VPN 高可用性**:
    * **ASA / FTD Active/Standby Stateful Failover**: 2台のアプライアンス間で Stateful Link を介して AnyConnect セッション情報（IKE/IPsec SA、SSL SA、IP アドレス割り当て、認証ステート）をリアルタイム同期し、一次機障害時にクライアントの再接続なしで無中断切替を実現する。
    * **ASA AnyConnect VPN Load Balancing Cluster**: 複数台（最大 10 台等）の ASA を仮想クラスタ（Virtual Cluster IP）としてまとめ、マスターアプライアンスが最も負荷の低いスレーブへ IKEv2 / SSL リダイレクトを行うことで、負荷分散と障害時自動再接続を両立する。
  * **Site-to-Site VPN 高可用性**:
    * **Multiple Crypto Peers (IPsec Peer Redundancy)**: Crypto Map 内に複数のピア IP アドレス (`set peer IP1 IP2`) を優先順に定義し、DPD (Dead Peer Detection) や ISAKMP/IKEv2 キープアライブにより一次ピアの障害を検知した際、自動的にセカンダリピアへ SA 再確立を試みる。
    * **IP SLA ＆ Floating Static Route Failover**: IP SLA オブジェクトで対向ピアの到達性をモニタリングし、トラッキング状態に応じてルーティングテーブルのディスタンス値（Floating Static Route）を動的に切り替えてバックアップトンネルを起動する。
    * **HSRP ＋ RRI ＋ IPsec Stateful / Stateless HA**: HSRP の Virtual IP (VIP) を Tunnel Endpoint として共有し、Reverse Route Injection (RRI) で上流ルーターへの戻りルートを動的追従させる。SSO (Stateful Switchover) 統合環境では SCTP プロトコル経由で SA ステートを動的同期する。
  * **マルチハブ VPN 高可用性 (DMVPN / FlexVPN)**:
    * **DMVPN Dual-Hub (Single Cloud vs Dual Cloud)**:
      * **Single Cloud (DHSD / MHSD)**: 単一の mGRE サブネット（Overlay）内に複数の Hub を NHS (Next Hop Server) として配置。シンプルだがルーティングの優先制御に制約がある。
      * **Dual Cloud (DHDD)**: 独立した 2 つの mGRE サブネット（Cloud 1 & Cloud 2）を構築し、Spoke に 2 つの Tunnel インターフェイスを配置。EIGRP/OSPF/BGP のメトリック（Delay、Cost）を用いてプライマリ/セカンダリの完全制御が可能。
    * **FlexVPN High Availability (Dual-Hub FlexVPN ＆ FlexVPN Load Balancer)**:
      * **Backup Peers ＆ Backup Group**: Client 側の FlexVPN Profile に複数の Peer を優先順でバインドし、IP SLA Track や `backup-group` によりアクティブ接続の維持とフォールバックを自動制御。
      * **FlexVPN Load Balancer**: IKEv2 Redirect (RFC 5685) と HSRP VIP を統合し、Master ルーターが IKE_SA_INIT / IKE_AUTH 時にクライアントを最も負荷の低い Cluster Slave へ直接リダイレクトする。
* **利用目的**:
  * ネットワーク機器の故障、SFP/リンクダウン、ISP 回線障害時における VPN 通信途絶の防止。
  * 大規模 RAVPN 環境における暗号化処理負荷の平坦化と処理能力の水平拡張 (Scale-out)。
  * メイン拠点（HQ / DC）の完全二重化による災害対策 (DR: Disaster Recovery) 対応。
* **どのような場面で利用するか**:
  * 企業の本社・データセンター境界での AnyConnect RAVPN 収容基盤。
  * 全国の拠点（Branch）と 2 箇所のデータセンター間を接続する DMVPN / FlexVPN バックボーン。
  * 異種キャリア（MPLS ＋ インターネット）を利用したマルチホーミング Site-to-Site VPN。

---

## 🔑 要点

| 冗長化テクノロジー | 対象モデル / OS | セッション切替時間 | ステートフル同期 | スループレット拡張 | 特徴 / 主な用途 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **ASA Active/Standby Stateful Failover** | ASA / FTD (HA) | 1秒未満 | **あり** (State Link 同期) | なし | IPsec / SSL RAVPN の完全無中断フェールオーバー |
| **ASA VPN Load Balancing Cluster** | ASA (Single Mode) | 数秒 (再接続) | なし | **あり** (最大10台) | AnyConnect セッションの自動負荷分散と集約 |
| **Multiple Crypto Peers (`set peer`)** | ASA / FTD / IOS | DPD タイマー依存 (10-30秒) | なし | なし | 定番の Site-to-Site バックアップピア設定 |
| **IP SLA Track ＋ Floating Static Route** | ASA / FTD / IOS | IP SLA タイマー依存 | なし | なし | 回線障害検知に基づく静的ルート切替 |
| **HSRP ＋ RRI ＋ IPsec HA (IOS)** | IOS / IOS-XE Router | HSRP ＋ SA 再確立 | なし (SSO時あり) | なし | 共通 VIP を対向に提示するヘッドエンド冗長化 |
| **DMVPN Dual-Hub Single-Cloud** | IOS / IOS-XE Router | IGP コンバージェンス時間 | なし | あり (アクティブ分散) | 単一 Overlay 構成のマルチハブ冗長 |
| **DMVPN Dual-Hub Dual-Cloud** | IOS / IOS-XE Router | IGP コンバージェンス時間 | なし | あり (ポリシー分離) | 完全分離 Overlay による確実なトラフィック制御 |
| **FlexVPN Backup Peers / Backup Group** | IOS / IOS-XE Router | DPD / SLA タイマー依存 | なし | なし | IKEv2 IKE_SA_INIT レベルでのバックアップ切替 |
| **FlexVPN Load Balancer (RFC 5685)** | IOS / IOS-XE Router | 数秒 (リダイレクト) | なし | **あり** (HSRP VIP) | IKEv2 Redirect を用いたヘッドエンドクラスタリング |

---

## 🏗 動作原理

### 1. ASA AnyConnect VPN Load Balancing Cluster の動作フロー

AnyConnect クライアントが仮想クラスタ IP (Virtual Cluster IP) に接続し、Master から Least-Loaded Slave へリダイレクトされるアーキテクチャ：

```text
[ AnyConnect Client ] ───(1) Initial SSL/IKEv2 Connection ───► [ Virtual Cluster IP: 203.0.113.100 ]
                                                                        │
                                                                        ▼
                                                         [ ASA-1 (Cluster Master) ]
                                                            ├── Checks CPU/RAM & Session Load
                                                            └── Identifies ASA-2 as Least-Loaded Node
                                                                        │
[ AnyConnect Client ] ◄──(2) HTTP 302 / IKEv2 Redirect (ASA-2 IP) ──────┘
       │
       └───(3) Establishment of VPN Tunnel ───────────────────────────► [ ASA-2 (Cluster Slave: 203.0.113.102) ]
                                                                             └── Assigned IP, Encrypted Data Flow
```

---

### 2. FlexVPN Load Balancer (HSRP VIP ＋ RFC 5685 IKEv2 Redirect) 動作フロー

```text
[ FlexVPN Client ] ───(1) IKE_SA_INIT (REDIRECT_SUPPORTED) ───► [ HSRP VIP: 192.168.10.1 (Master) ]
                                                                        │
                                                                        ├── Queries Cluster Member Loads
                                                                        └── Selects Router 3 (10.10.10.3)
                                                                        │
[ FlexVPN Client ] ◄──(2) IKE_SA_INIT Reply (NOTIFY: REDIRECT 10.10.10.3) ┘
       │
       └───(3) New IKE_SA_INIT / IKE_AUTH ────────────────────────────► [ FlexVPN Server 3 (10.10.10.3) ]
                                                                             └── Virtual-Access Tunnel Created
```

---

### 3. DMVPN Dual-Hub Dual-Cloud (DHDD) vs Single-Cloud (DHSD) 構成フロー

```text
[ Dual-Hub Dual-Cloud (DHDD) Topology ]
                          ┌─── [ Hub 1 (172.16.1.1) ] ─── (DMVPN Cloud 1: 172.16.1.0/24)
  [ Spoke Router ] ───────┤
                          └─── [ Hub 2 (172.16.2.1) ] ─── (DMVPN Cloud 2: 172.16.2.0/24)

  * Spoke has Tunnel0 (Cloud 1, EIGRP Delay 1000) and Tunnel1 (Cloud 2, EIGRP Delay 2000).
  * Independent NHRP Mapping & Routing metrics allow precise active/backup path selection.

[ Dual-Hub Single-Cloud (DHSD) Topology ]
  [ Spoke Router ] ─────── (Tunnel0: 172.16.1.0/24) ──────┬─── [ Hub 1 (172.16.1.1) - Primary NHS ]
                                                          └─── [ Hub 2 (172.16.1.2) - Secondary NHS ]

  * Single mGRE subnet. Both Hubs share the same IP network.
```

---

## ⚙ 動作シーケンス

1. **ASA Active/Standby Stateful VPN Failover シーケンス**:
   * **アクティブ機（ASA-1）での SA 確立**: クライアント（または対向 Peer）との間で IKE SA および IPsec SA（あるいは SSL VPN セッション）が生成される。
   * **Stateful Link 経由の同期**: ASA-1 は生成された SA 鍵情報、SPI、Sequence Number、IP Pool アロケーション情報を Stateful Link 経由で Standby 機（ASA-2）へ即座にレプリケーション。
   * **障害発生 ＆ フェールオーバー判定**: ASA-1 のリンクダウンまたは Keepalive 喪失が発生。
   * **無中断切替**: ASA-2 が Active へ昇格。共通の Virtual IP / MAC アドレスを Gratuitous ARP で通知。クライアントは再ハンドシェイクなしで暗号化通信を継続。
2. **ASA VPN Load Balancing Cluster シーケンス**:
   * **Heartbeat ＆ 負荷状態共有**: クラスタに参加する ASA（Master 1台、Slave 複数台）は、UDP 5555（UDP 内部通信）を用いて全ノードの現在のセッション数、CPU/メモリロード情報を 1 秒周期で共有。
   * **Initial Connection 到着**: クライアントが DNS で解決した Virtual Cluster IP（Master ノードが応答）へアクセス。
   * **Redirect 応答**: Master ノードは送信元クライアントに対し、最小負荷の Slave ノードの Public IP アドレスを書き込んだリダイレクト応答（HTTP 302 または IKEv2 Notify）を送信。
   * **Direct SA 確立**: クライアントは指定された Slave ノードの IP アドレスへ直接接続し、VPN セッションを確立。
3. **Multiple Crypto Peers (IPsec Backup Peer) シーケンス**:
   * **Primary Peer への接続確立**: Initiator は `set peer IP1 IP2` の順序に従い、IP1 宛てに IKE_SA_INIT を送信。正常であれば IP1 との間で Phase 1/Phase 2 SA を確立。
   * **DPD (Dead Peer Detection) 監視**: 両端で DPD (`crypto isakmp keepalive` または `crypto ikev2 dpd`) を有効化。通信途絶時に R-U-THERE メッセージを送受信。
   * **DPD Timeout ＆ Active Peer 変更**: DPD 応答が途絶した場合、Initiator はローカルの SA を削除し、Crypto Map の次順ピア（IP2）宛てに新たな IKE ネゴシエーションを開始。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **ASA VPN Load Balancing Cluster の要件 ＆ 設定手順**:
   * **クラスタ設定プロファイル**: `vpn load-balancing` モードに入り、`cluster ip address <VIP>`、`priority <NUM>`、`redirect-fqdn`、`cluster port <PORT>` を定義。
   * **Cluster Encryption Key**: 全ノードで `cluster encryption key <KEY>` を一致させる。
   * **Group URL の組み合わせ要件**: AnyConnect グループ URL を使用する場合、**Virtual Cluster IP 用の Group URL** と **各ノード独自の Public IP 用の Group URL** の両方を Connection Profile にアサインする必要がある（片方だけだとリダイレクト後に 404 エラーとなる）。
2. **ASA Stateful VPN Failover ＆ HTTP レプリケーションの注意点**:
   * **Stateful Link の独立**: `failover link STATE_LINK <IF>` でデータポートとは別、または LAN Failover と共用のインターフェイスを指定。
   * **HTTP Connection Replication**: デフォルトでは HTTP セッションはレプリケーション対象外。要件で「Ensure HTTP connections remain active during failover」と指示された場合、**`failover replication http`** の投入が必須。
3. **Router HSRP ＋ RRI ＋ IPsec Stateful HA (SSO / SCTP) の設定要件**:
   * **HSRP 側の準備**: `standby 1 ip <VIP>`、`standby 1 name VPNHA`、`standby 1 track <IF>`。
   * **Crypto Map への HSRP 紐付け**: **`crypto map <NAME> <SEQ> ipsec-isakmp`** の下で **`set peer <REMOTE_IP>`** ＋ **`reverse-route`** ＋ **`redundancy VPNHA`** を設定。
   * **SCTP / IPC による Stateful Switchover**: `crypto ha` 構成により、IKE / IPsec SA を HSRP Active 機から Standby 機へ同期させ、無中断切り替えを実現する。
4. **DMVPN Dual-Hub (Single Cloud vs Dual Cloud) の選定とルーティング制御**:
   * **Single Cloud (DHSD)**: Spoke は 1 つの `interface Tunnel0` のみ設定。NHRP 設定で `ip nhrp nhs <HUB1_TUNNEL_IP>` と `ip nhrp nhs <HUB2_TUNNEL_IP>` を並列定義。EIGRP などのハブ側 `no ip split-horizon` に注意。
   * **Dual Cloud (DHDD)**: Spoke に `Tunnel0` (Cloud 1 -> Hub1) と `Tunnel1` (Cloud 2 -> Hub2) を作成。`Tunnel0` のメトリック（`delay 1000`）を `Tunnel1`（`delay 2000`）より優先させ、確実な Primary / Backup アクティブ・スタンバイ制御を構成。
5. **FlexVPN Backup Peers ＆ Backup Group 構成**:
   * **Backup Peers**: FlexVPN Client Profile 内で `match parent backhaul` ＋ `peer 1 <IP1> pref 1`, `peer 2 <IP2> pref 2` を定義。
   * **Backup Group**: `backup group <1-255>` を設定し、他 Client Profile が既に同じ Server に接続している場合に重複接続を回避するグループ制御。
6. **FlexVPN Load Balancer (RFC 5685 IKEv2 Redirect)**:
   * **HSRP 統合**: HSRP Active ルーターが Master として VIP を所有し、クライアントから到着した IKE_SA_INIT を `crypto ikev2 cluster` 設定に基づき Least-Loaded Slave ルーターへリダイレクト。

---

## 🛠 設定方法

### 1. ASA CLI: AnyConnect VPN Load Balancing Cluster 完全設定

```bash
! ===== ASA-1 (Cluster Master) Configuration =====
vpn load-balancing
 priority 10                            ! 数値が高いほど Master 優先度が高い
 cluster ip address 203.0.113.100       ! Virtual Cluster IP
 cluster port 5555
 cluster interface-group outside
 cluster encryption key cisco123
 redirect-fqdn vpn.example.com
 participate-in-load-balancing
 enable

! Connection Profile / Group URL 設定 (Virtual IP & Native IP 両対応)
webvpn
 enable outside

tunnel-group TG_EMPLOYEES type remote-access
tunnel-group TG_EMPLOYEES general-attributes
 address-pool POOL_RA_VPN
 default-group-policy GP_EMPLOYEES
tunnel-group TG_EMPLOYEES webvpn-attributes
 group-url https://vpn.example.com/employees enable

! ===== ASA-2 (Cluster Slave) Configuration =====
vpn load-balancing
 priority 5
 cluster ip address 203.0.113.100
 cluster port 5555
 cluster interface-group outside
 cluster encryption key cisco123
 redirect-fqdn vpn.example.com
 participate-in-load-balancing
 enable
```

---

### 2. Router CLI: Multiple Crypto Map Peers ＋ DPD ＋ RRI 設定

```bash
! ===== Branch Router Site-to-Site High Availability =====
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14

crypto isakmp key PrimaryKey123 address 198.51.100.1
crypto isakmp key SecondaryKey123 address 198.51.100.2
crypto isakmp keepalive 10 3 periodic   ! DPD 有効化 (10秒間隔, 3回再試行)

crypto ipsec transform-set TSET_AES_SHA esp-aes 256 esp-sha256-hmac
 mode tunnel

crypto map CMAP_TO_HQ 10 ipsec-isakmp
 set peer 198.51.100.1 198.51.100.2    ! 198.51.100.1 (Primary), 198.51.100.2 (Backup)
 set transform-set TSET_AES_SHA
 match address ACL_VPN_TRAFFIC
 reverse-route                         ! Dynamic RRI 有効化

interface GigabitEthernet0/0
 ip address 203.0.113.10 255.255.255.0
 crypto map CMAP_TO_HQ
```

---

### 3. Router CLI: HSRP ＋ RRI ＋ IPsec Stateful HA (SSO / SCTP) 設定

```bash
! ===== HQ Primary Gateway Router (R1) =====
crypto ha
 ike VIP 172.16.100.1
 ipsec VIP 172.16.100.1

interface GigabitEthernet0/0
 description OUTSIDE_INTERFACE
 ip address 172.16.100.2 255.255.255.0
 standby 1 ip 172.16.100.1              ! HSRP Virtual IP
 standby 1 priority 110
 standby 1 preempt
 standby 1 name VPN_HA_GROUP
 standby 1 track GigabitEthernet0/1
 crypto map CMAP_HQ redundancy VPN_HA_GROUP

crypto map CMAP_HQ 10 ipsec-isakmp
 set peer 203.0.113.10
 set transform-set TSET_AES_SHA
 match address ACL_VPN_TRAFFIC
 reverse-route

! ===== HQ Standby Gateway Router (R2) =====
crypto ha
 ike VIP 172.16.100.1
 ipsec VIP 172.16.100.1

interface GigabitEthernet0/0
 ip address 172.16.100.3 255.255.255.0
 standby 1 ip 172.16.100.1
 standby 1 priority 100
 standby 1 preempt
 standby 1 name VPN_HA_GROUP
 crypto map CMAP_HQ redundancy VPN_HA_GROUP
```

---

### 4. Router CLI: DMVPN Dual-Hub Dual-Cloud (DHDD) Phase 3 設定

```bash
! ===== Spoke Router Configuration (Dual Cloud DHDD) =====
! ----- Cloud 1 (Hub 1 接続用) -----
interface Tunnel0
 ip address 172.16.1.10 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication DMVPN_KEY1
 ip nhrp network-id 100
 ip nhrp holdtime 300
 ip nhrp nhs 172.16.1.1 nbma 198.51.100.1  ! Hub 1 NHS/NBMA
 ip nhrp shortcut
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel key 100
 tunnel protection ipsec profile IPSEC_PROF_DMVPN
 delay 1000                            ! Primary Cloud (低メトリック)

! ----- Cloud 2 (Hub 2 接続用) -----
interface Tunnel1
 ip address 172.16.2.10 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication DMVPN_KEY2
 ip nhrp network-id 200
 ip nhrp holdtime 300
 ip nhrp nhs 172.16.2.1 nbma 198.51.100.2  ! Hub 2 NHS/NBMA
 ip nhrp shortcut
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel key 200
 tunnel protection ipsec profile IPSEC_PROF_DMVPN
 delay 2000                            ! Backup Cloud (高メトリック)
```

---

### 5. Router CLI: FlexVPN Load Balancer (RFC 5685 IKEv2 Redirect ＋ HSRP) 設定

```bash
! ===== FlexVPN Master Server Configuration =====
crypto ikev2 cluster
 standby-group Group1
 master
 max-session 1000
 re-redirect-val 10
 no shutdown

interface GigabitEthernet0/0
 ip address 192.168.10.2 255.255.255.0
 standby 1 ip 192.168.10.1              ! HSRP VIP (Cluster Endpoint)
 standby 1 name Group1
 standby 1 priority 110
 standby 1 preempt

crypto ikev2 profile PROF_FLEX_SERVER
 match identity remote any
 authentication local pre-share key Cisco123
 authentication remote pre-share key Cisco123
 aaa authorization group psk list default default
 virtual-template 1
```

---

## 🔍 検証コマンド

| 目的 | コマンド (ASA / IOS / FTD CLI) | 表示確認ポイント |
| :--- | :--- | :--- |
| **ASA RAVPN Cluster 状態 ＆ メンバー確認** | <code>show vpn-sessiondb cluster</code> | Master/Slave 状態、接続クライアント数、Public/Private IP |
| **ASA Active/Standby Stateful 状態確認** | <code>show failover</code> / <code>show failover state</code> | Stateful Link 状態、SA 同期オブジェクト統計 |
| **IPsec SA / Peer 状態確認** | <code>show crypto ipsec sa</code> / <code>show crypto session</code> | `current_peer`, Encaps/Decaps パケット数, DPD 状態 |
| **IKEv2 / FlexVPN SA ＆ Redirect 状態確認** | <code>show crypto ikev2 sa detailed</code> | `IKEv2 Redirect Supported`, Redirect 先 IP |
| **FlexVPN Cluster (Load Balancer) 状態確認** | <code>show crypto ikev2 cluster</code> | Master/Slave ロール、メンバー負荷率、リダイレクト数 |
| **IPsec Stateful HA (Crypto HA) 同期状態** | <code>show crypto ha</code> | IKE VIP / IPsec VIP 状態、SA 同期スタンプ |
| **DMVPN NHRP ピア ＆ NHS トラッキング確認** | <code>show ip nhrp</code> / <code>show ip nhrp nhs</code> | NHS ピア（Hub）の状態 (`UP`), Priority, Cluster |
| **IP SLA トラッキングステータス確認** | <code>show track</code> / <code>show ip sla summary</code> | Track オブジェクト状態 (`UP` / `DOWN`), RTT |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **ASA AnyConnect VPN Cluster でリダイレクト後にクライアントが接続エラーになる** | Connection Profile の Group URL に Master VIP 用 URL はあるが、**Slave 独自の Public IP/FQDN 用 Group URL が未設定** | <code>show vpn-sessiondb cluster</code><br><code>show running-config tunnel-group</code> | 全 Cluster Member の固有 Public IP / FQDN に対する Group URL を Connection Profile に追加定義する。 |
| **ASA Active/Standby Failover 切り替え時に AnyConnect RAVPN セッションが切断され再認証を要求される** | Stateful Failover の **`failover link` が未有効**、またはライセンス/モード不一致 | <code>show failover</code> | `failover link` の設定および `failover interface ip` の正常性を確認する。 |
| **Multiple Crypto Peers (`set peer IP1 IP2`) で Primary 障害時に切替が極めて遅い (数分かかる)** | **DPD (Dead Peer Detection) が未設定**のまま、デフォルトの ISAKMP/IPsec タイムアウトを待機している | <code>show crypto isakmp keepalive</code><br><code>show crypto ikev2 sa detailed</code> | `crypto isakmp keepalive 10 3 periodic` または IKEv2 DPD を有効化し、短時間で障害検知させる。 |
| **Multiple Crypto Peers で Primary 回線が復旧しても Backup Peer から Primary に切り替わらない** | IPsec Crypto Map の仕様上、**確立済みの Active SA は期限切れ (Lifetime) または DPD 破棄まで維持**される | <code>show crypto ipsec sa</code> | 正常な動作仕様。即時復旧が必要な場合は IP SLA ＋ Dynamic Routing / VTI の構成を検討する。 |
| **Router HSRP ＋ RRI ＋ IPsec HA で切替後に戻りトラフィックが対向ルーターに届かない** | **Reverse Route Injection (RRI)** が未設定のため、上流ルーターのルーティングテーブルに VIP 宛てのルートが動的生成されていない | <code>show ip route</code><br><code>show crypto map</code> | Crypto Map 内で `reverse-route` を有効化し、上流の IGP (EIGRP/OSPF) で再配布する。 |
| **DMVPN Dual-Hub で Spoke から Hub2 へのトンネルが Up にならない** | Spoke の `Tunnel1` で **`ip nhrp network-id` や `tunnel key` が Hub2 と不一致** | <code>show ip nhrp</code><br><code>show crypto session</code> | NHRP Network ID、Tunnel Key、および IPsec Profile 設定を Hub2 と一致させる。 |

---

## ⚠ 制限事項

* **ASA Multi-Context Mode における制限**:
  * ASA Multi-Context モードでは、**VPN Load Balancing Cluster および Remote Access VPN 機能全般（AnyConnect / Clientless SSL）がサポートされません**。RAVPN 高可用性には Single Context モードでの Active/Standby Failover または VPN Load Balancing Cluster を採用する必要があります。
* **FTD Clustering における Remote Access VPN 非対応**:
  * FTD Cluster（Multi-node Cluster）では Remote Access VPN は一切サポートされません（Site-to-Site VPN のみ Control Node 上で Centralized 機能として動作）。
* **Multiple Crypto Map Peers の非プリエンプション（Non-Preemptive）動作**:
  * Crypto Map で `set peer IP1 IP2` を設定した場合、IP1 障害時に IP2 へ切替わりますが、IP1 が復旧しても既存セッションは自動的に IP1 へ戻りません（SA の Lifetime 満了またはクリア時まで IP2 を維持）。

---

## 🔄 他技術との関連

* **Dynamic Routing Protocols (EIGRP, OSPF, BGP)**: DMVPN / FlexVPN バックボーン上でマルチハブ間の最適な経路選定とアクティブ・バックアップ動的切替を制御。
* **HSRP / VRRP (First Hop Redundancy Protocol)**: ルーター上の IPsec VIP エンドポイントおよび FlexVPN Load Balancer の Virtual Master 宛てロードバランス基盤を提供。
* **IP SLA ＆ Track**: 外線 ICMP / TCP ポートプローブによる回線状態を常時モニタリングし、Static Route や FlexVPN Profile のソースを動的判定。
* **Reverse Route Injection (RRI)**: IPsec SA の確立状態と連動して動的にスタティックルートを生成し、上流ルーターへ戻りトラフィックの正しい Next-hop を開示。

---

## 🧩 比較表

### 1. DMVPN Dual-Hub Single Cloud (DHSD) vs Dual-Hub Dual Cloud (DHDD)

| 比較項目 | DMVPN Single Cloud (DHSD) | DMVPN Dual Cloud (DHDD) |
| :--- | :--- | :--- |
| **mGRE サブネット数** | **単一サブネット** (例: `172.16.1.0/24`) | **2つの独立サブネット** (例: `172.16.1.0/24`, `172.16.2.0/24`) |
| **Spoke 側 Tunnel 数** | 1つの `interface Tunnel0` | **2つの Tunnel インターフェイス** (`Tunnel0`, `Tunnel1`) |
| **トラフィック制御** | ルーティングプロトコルのメトリック制御に制約 | **インターフェイス単位の Delay / Cost で完全制御** |
| **設定の容易性** | 設定がシンプルで展開が容易 | 設定箇所が増えるが、確実な冗長設計が可能 |
| **CCIE 試験推奨** | 基本的な冗長化 | **トラフィック制御・アクティブ/スタンバイ指定の推奨構成** |

### 2. AnyConnect VPN Load Balancing Cluster vs ASA Active/Standby Failover

| 比較項目 | VPN Load Balancing Cluster | Active/Standby Stateful Failover |
| :--- | :--- | :--- |
| **主な目的** | **負荷分散 (Load Distribution)** ＋ 冗長 | **セッション維持 (Stateful Redundancy)** |
| **切替時ユーザー影響** | セッション切断 ➔ **自動再接続** (再認証) | **完全無中断** (再認証なし・透明切替) |
| **最大ノード数** | 最大 10 台 | 2 台 (Active / Standby) |
| **IP アドレス形態** | Virtual Cluster IP ＋ 各ノード固有 IP | Active / Standby 共有 IP |
| **動作モード** | Single Context Mode のみ | Single / Multi-Context Mode |

---

## 💡 ベストプラクティス

1. **DPD (Dead Peer Detection) の標準有効化**:
   * Site-to-Site VPN や RAVPN の全冗長構成において、`crypto isakmp keepalive 10 3 periodic` または IKEv2 DPD を標準化し、障害発生時に無駄な SA を保持せず 30 秒以内にバックアップピアへ切り替えを促します。
2. **HSRP ＋ RRI ＋ DPD のセット運用**:
   * IOS ルーターで IPsec 冗長化を行う場合、HSRP VIP のみでは片側リンク障害時に即座に戻りルートが変更されないリスクがあります。必ず **Reverse Route Injection (`reverse-route`)** を併用し、IGP へルート再配布させて上流ネットワークの対称性を保証します。
3. **DMVPN では Dual-Cloud 構成を採用**:
   * マルチハブ DMVPN では、トラブルシュートとパス制御が容易な Dual-Cloud 構成を推奨します。EIGRP の `delay` や OSPF の `ip ospf cost` を `Tunnel0` と `Tunnel1` で変えるだけで、明快な Priority 制御が成立します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA AnyConnect VPN Load Balancing Cluster の構築

#### 問題
ASA1 (Master) と ASA2 (Slave) において、仮想 IP `203.0.113.100` を用いた AnyConnect VPN ロードバランシングクラスタを構築せよ。

#### 要件
* クラスタポート: `5555`
* 暗号化キー: `cisco123`
* ASA1 Priority: `10`, ASA2 Priority: `5`

#### 設定例

```bash
! ===== ASA1 (Master) =====
vpn load-balancing
 priority 10
 cluster ip address 203.0.113.100
 cluster port 5555
 cluster interface-group outside
 cluster encryption key cisco123
 participate-in-load-balancing
 enable

! ===== ASA2 (Slave) =====
vpn load-balancing
 priority 5
 cluster ip address 203.0.113.100
 cluster port 5555
 cluster interface-group outside
 cluster encryption key cisco123
 participate-in-load-balancing
 enable
```

#### 検証方法
* ASA1 で `show vpn-sessiondb cluster` を実行し、ASA1 が `Master`、ASA2 が `Slave` として表示され、VIP `203.0.113.100` がバインドされていることを確認する。

---

### 例2: ASA Active/Standby Stateful Failover ＆ HTTP レプリケーション

#### 問題
2台の ASA 間で Stateful Failover を構成し、障害切替時にも HTTP セッションステートが維持されるよう設定せよ。

#### 要件
* `failover replication http` を使用すること。

#### 設定例

```bash
! ===== ASA Primary Configuration =====
failover lan unit primary
failover lan interface FO_LAN GigabitEthernet0/3
failover link FO_STATE GigabitEthernet0/4
failover interface ip FO_LAN 10.255.255.1 255.255.255.252 standby 10.255.255.2
failover interface ip FO_STATE 10.255.255.5 255.255.255.252 standby 10.255.255.6
failover key cisco123
failover replication http
failover
```

#### 検証方法
* `show failover` を実行し、`failover replication http` が有効化され、Stateful updates が送受信されていることを確認する。

---

### 例3: Router Site-to-Site Multiple Crypto Map Peers ＋ DPD

#### 問題
ルーター R1 において、対向 HQ のプライマリ IP `198.51.100.1` およびセカンダリ IP `198.51.100.2` に対する 2 重化 Site-to-Site IPsec トンネルを構成せよ。

#### 要件
* DPD 周期: 10 秒, 再試行: 3 回。

#### 設定例

```bash
! ===== R1 Configuration =====
crypto isakmp keepalive 10 3 periodic

crypto map CMAP_HQ 10 ipsec-isakmp
 set peer 198.51.100.1 198.51.100.2
 set transform-set TSET_AES_SHA
 match address ACL_VPN
```

#### 検証方法
* `show crypto map` を実行し、`Peer` に `198.51.100.1` および `198.51.100.2` が順序通り登録されていることを確認する。

---

### 例4: Router HSRP ＋ RRI ＋ IPsec Stateful High Availability (SSO/SCTP)

#### 問題
ルーター R1 (Primary) と R2 (Secondary) において、HSRP VIP `172.16.100.1` を用い、IPsec SA ステートを動的同期させる Stateful IPsec HA を構成せよ。

#### 要件
* HSRP グループ名 `VPN_HA_GROUP`
* RRI (`reverse-route`) を設定すること。

#### 設定例

```bash
! ===== R1 (Primary Router) =====
crypto ha
 ike VIP 172.16.100.1
 ipsec VIP 172.16.100.1

interface GigabitEthernet0/0
 ip address 172.16.100.2 255.255.255.0
 standby 1 ip 172.16.100.1
 standby 1 priority 110
 standby 1 preempt
 standby 1 name VPN_HA_GROUP
 crypto map CMAP_HQ redundancy VPN_HA_GROUP

crypto map CMAP_HQ 10 ipsec-isakmp
 set peer 203.0.113.10
 set transform-set TSET_AES_SHA
 match address ACL_VPN
 reverse-route
```

#### 検証方法
* R1 および R2 で `show crypto ha` を実行し、IKE/IPsec VIP が定義され、Sync 状態が正常であることを確認する。

---

### 例5: IP SLA Tracking に基づく Floating Static Route Failover

#### 問題
ルーター R1 において、対向 IP `198.51.100.1` への ICMP プローブを IP SLA 1 で監視し、Primary 経路断絶時に Backup 経路へ切り替えよ。

#### 要件
* Primary Route AD: `1`, Backup Route AD: `10`

#### 設定例

```bash
! ===== R1 Configuration =====
ip sla 1
 icmp-echo 198.51.100.1 source-interface GigabitEthernet0/0
 frequency 5

ip sla schedule 1 start-time now life forever

track 1 ip sla 1 reachability

ip route 10.100.0.0 255.255.0.0 198.51.100.1 track 1   ! Primary Route
ip route 10.100.0.0 255.255.0.0 198.51.200.1 10        ! Backup Route
```

#### 検証方法
* `show track 1` および `show ip route` を実行し、`198.51.100.1` ダウン時にルーティングテーブルがバックアップアドレスへ切り替わることを確認する。

---

### 例6: DMVPN Dual-Hub Single-Cloud (DHSD) Phase 3 構成

#### 問題
Spoke ルーターにおいて、単一の GRE トラフィック（`Tunnel0`）内で Hub1 (`172.16.1.1`) および Hub2 (`172.16.1.2`) を NHS として並列登録せよ。

#### 要件
* Single Subnet: `172.16.1.0/24`

#### 設定例

```bash
! ===== DMVPN Spoke Configuration =====
interface Tunnel0
 ip address 172.16.1.10 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication DMVPN_KEY
 ip nhrp network-id 100
 ip nhrp holdtime 300
 ip nhrp nhs 172.16.1.1 nbma 198.51.100.1
 ip nhrp nhs 172.16.1.2 nbma 198.51.100.2
 ip nhrp shortcut
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC_PROF_DMVPN
```

#### 検証方法
* `show ip nhrp nhs` を実行し、両 Hub (`172.16.1.1`, `172.16.1.2`) が RE (Registered) 状態となっていることを確認する。

---

### 例7: DMVPN Dual-Hub Dual-Cloud (DHDD) Phase 3 ＋ EIGRP 優先度制御

#### 問題
Spoke ルーターにおいて、独立した 2 つの DMVPN クラウド (`Tunnel0`: Hub1, `Tunnel1`: Hub2) を定義し、EIGRP の `delay` 設定により Hub1 経由を優先せよ。

#### 要件
* Tunnel0 Delay: `1000`, Tunnel1 Delay: `2000`

#### 設定例

```bash
! ===== DMVPN Spoke Configuration =====
interface Tunnel0
 description Primary Cloud to Hub 1
 ip address 172.16.1.10 255.255.255.0
 ip nhrp network-id 100
 ip nhrp nhs 172.16.1.1 nbma 198.51.100.1
 delay 1000

interface Tunnel1
 description Backup Cloud to Hub 2
 ip address 172.16.2.10 255.255.255.0
 ip nhrp network-id 200
 ip nhrp nhs 172.16.2.1 nbma 198.51.100.2
 delay 2000

router eigrp 100
 network 172.16.1.0 0.0.0.255
 network 172.16.2.0 0.0.0.255
```

#### 検証方法
* `show ip route eigrp` を実行し、送信元トラフィックの Next-hop が `Tunnel0` (Hub1) 側になっていることを確認する。

---

### 例8: FlexVPN Dual-Hub Resiliency (Backup Peers ＋ IP SLA Track)

#### 問題
FlexVPN Client ルーターにおいて、Primary Hub (`172.16.1.1`) への到達性を IP SLA 1 で追跡し、障害時に Secondary Hub (`172.16.2.1`) へ切り替えよ。

#### 要件
* `match parent backhaul` 設定を使用すること。

#### 設定例

```bash
! ===== FlexVPN Client Configuration =====
track 1 ip sla 1 reachability

crypto ikev2 profile PROF_FLEX_CLIENT
 match identity remote fqdn hub.example.com
 identity local fqdn client.example.com
 authentication local pre-share key Cisco123
 authentication remote pre-share key Cisco123
 match parent backhaul track 1
  peer 1 172.16.1.1 pref 1
  peer 2 172.16.2.1 pref 2
```

#### 検証方法
* Track 1 を Down に変更し、`show crypto ikev2 sa` で Client の接続先が `172.16.2.1` にフォールバックされることを確認する。

---

### 例9: FlexVPN IKEv2 Load Balancer (HSRP VIP ＋ RFC 5685 IKEv2 Redirect)

#### 問題
FlexVPN Server 1 (Master) において HSRP VIP `192.168.10.1` を提示し、到着したクライアントを最大セッション数 1000 の負荷計算に基づきリダイレクトせよ。

#### 要件
* `crypto ikev2 cluster` を使用すること。

#### 設定例

```bash
! ===== FlexVPN Server 1 (Master) =====
crypto ikev2 cluster
 standby-group Group1
 master
 max-session 1000
 no shutdown

interface GigabitEthernet0/0
 ip address 192.168.10.2 255.255.255.0
 standby 1 ip 192.168.10.1
 standby 1 name Group1
 standby 1 priority 110
 standby 1 preempt
```

#### 検証方法
* Client から HSRP VIP に接続し、`show crypto ikev2 cluster` でクライアントが Slave ルーターへリダイレクトされることを確認する。

---

### 例10: FlexVPN Backup Group による動的トンネル重複回避

#### 問題
同一ルーター上の 2 つの FlexVPN Client Profile (`flex1`, `flex2`) に対し、同一 Peer への重複接続を防止する `backup group 10` を割り当てよ。

#### 要件
* `backup group 10` を設定すること。

#### 設定例

```bash
! ===== FlexVPN Client Profile 1 =====
crypto ikev2 profile flex1
 match identity remote fqdn hub1.example.com
 identity local fqdn client1.example.com
 backup group 10

! ===== FlexVPN Client Profile 2 =====
crypto ikev2 profile flex2
 match identity remote fqdn hub2.example.com
 identity local fqdn client2.example.com
 backup group 10
```

#### 検証方法
* `debug crypto ikev2 client flexvpn` を有効化し、一方のプロファイルがアクティブな場合に他方のプロファイルが同一ピア接続をスキップすることを確認する。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 以下のコンフィグが投入された ASA1 および ASA2 の VPN Load Balancing クラスタ環境において、AnyConnect クライアントが `https://vpn.example.com/employees` に接続した際、リダイレクト後に 404 エラーが発生した。原因として正しいものはどれか。

```bash
! ASA-1 (Master)
tunnel-group TG_EMP webvpn-attributes
 group-url https://vpn.example.com/employees enable

! ASA-2 (Slave, Public FQDN: asa2.example.com)
tunnel-group TG_EMP webvpn-attributes
 group-url https://vpn.example.com/employees enable
```

* A) ASA1 の Priority が指定されていないため。
* B) Slave ノード (ASA2) の Connection Profile に、**ASA2 独自の Public FQDN (`https://asa2.example.com/employees`) 用の Group URL** が追加設定されていないため。
* C) UDP 5555 ポートがブロックされているため。
* D) AnyConnect のライセンスが不足しているため。

**正解**: **B**  
**解説**: VPN Load Balancing Cluster では、Master ノードがクライアントを Slave ノードの固有 Public IP / FQDN アドレスへリダイレクトします。そのため、各 Slave ノードの Connection Profile には、**Virtual Cluster IP 用の Group URL と 自ノード固有 FQDN/IP 用の Group URL の双方** を登録しておく必要があります。

---

### 2. トラブルシュート問題

**問題**: ルーター間で `set peer 198.51.100.1 198.51.100.2` を用いた Crypto Map を構成している。Primary ピア (`198.51.100.1`) の物理回線が切断されたにもかかわらず、切り替えに 3 分以上かかった。原因および最も効果的な対処方法はどれか。

* A) RRI が設定されていない。`reverse-route` を追加する。
* B) DPD (Dead Peer Detection) が設定されていない。`crypto isakmp keepalive 10 3 periodic`（または IKEv2 DPD）を設定して短時間で障害検出させる。
* C) Static Route のディスタンス値が大きすぎる。
* D) HSRP が設定されていない。

**正解**: **B**  
**解説**: Multiple Crypto Peers において DPD が未設定の場合、ルーターは ISAKMP / IPsec タイムアウトまで相手の障害を認識できず、切替に長い時間を要します。DPD (Keepalive) を設定することで、数秒〜数十秒で故障を検知してセカンダリピアへ再接続できます。

---

### 3. Design 問題

**問題**: 全国 50 箇所の拠点（Spoke）を 2 箇所のデータセンター（Hub1, Hub2）へ DMVPN で収容する設計を行う。ハブ回線の完全分離、帯域に応じたプライマリ/バックアップの明快なパス制御、および暗号化オーバーレイの完全制御を実現するための最適構成はどれか。

* A) DMVPN Phase 1 Single Cloud
* B) DMVPN Phase 3 Dual-Hub Dual-Cloud (DHDD)
* C) DMVPN Phase 2 Single Cloud
* D) FlexVPN Single Hub

**正解**: **B**  
**解説**: **DMVPN Phase 3 Dual-Hub Dual-Cloud (DHDD)** は、2つの独立した mGRE サブネット（Cloud 1 ＆ Cloud 2）と Spoke 側の 2 つの Tunnel インターフェイスを使用するため、IGP メトリック（Delay / Cost）による明快かつ柔軟なトラフィック制御と障害分離が実現します。

---

### 4. 実装問題

**問題**: IOS ルーターで HSRP VIP を Tunnel Endpoint とした Site-to-Site IPsec VPN を構築する際、Crypto Map を HSRP グループにバインドするために使用するコマンドはどれか。

* A) `crypto map CMAP redundancy <HSRP_NAME>`
* B) `standby 1 ipsec-enable`
* C) `crypto ha active`
* D) `ipsec group hsrp`

**正解**: **A**  
**解説**: Crypto Map を HSRP トラッキングおよび Virtual IP と連動させるには、インターフェイス側で `standby 1 name <NAME>` を設定し、Crypto Map 内で **`crypto map <MAP_NAME> <SEQ> ipsec-isakmp`** の下で **`redundancy <NAME>`** を指定します。

---

### 5. コンフィグ読解問題

**問題**: FlexVPN Load Balancer (RFC 5685) において、クライアントが HSRP VIP 宛てに IKE_SA_INIT パケットを送信した際、リダイレクト機能を利用可能であることを示すために送信に含まれる IKEv2 Notify メッセージはどれか。

* A) `REDIRECT_SUPPORTED`
* B) `LOAD_BALANCING_OK`
* C) `CLUSTER_MEMBER_REQ`
* D) `COOKIE_CHALLENGE`

**正解**: **A**  
**解説**: RFC 5685 に基づき、IKEv2 クライアントがリダイレクト処理に対応していることをサーバーへ伝えるため、IKE_SA_INIT パケット内に **`REDIRECT_SUPPORTED`** 通知メッセージを格納します。

---

## 🔗 参考リソース

* [Cisco ASA Series General Operations Configuration Guide: VPN Load Balancing](https://www.cisco.com/)
* [Cisco ASA Series General Operations Configuration Guide: High Availability](https://www.cisco.com/)
* [Cisco IOS Flexible VPN Configuration Guide: FlexVPN Load Balancer and High Availability](https://www.cisco.com/)
* [Cisco Dynamic Multipoint VPN (DMVPN) Design Guide](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot IPsec VPN High Availability with HSRP and RRI](https://www.cisco.com/)
* [Cisco Live: Advanced Site-to-Site and Remote Access VPN High Availability Architecture (BRKSEC-3033)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で VPN High Availability が問われた場合の選択フローチャート：
    * **「Load balance AnyConnect users across multiple ASAs」** ➔ ASA VPN Load Balancing Cluster (`vpn load-balancing` ＋ Group URL 二重登録)。
    * **「Zero downtime failover for AnyConnect / S2S on ASA」** ➔ ASA Active/Standby Stateful Failover (`failover link` ＋ `failover replication http`)。
    * **「Backup peer for Site-to-Site IPsec map」** ➔ `set peer IP1 IP2` ＋ `crypto isakmp keepalive 10 3 periodic` (DPD)。
    * **「HSRP Router Redundancy with IPsec」** ➔ HSRP (`standby name`) ＋ Crypto Map (`redundancy`) ＋ `reverse-route` (RRI) ＋ `crypto ha` (SSO)。
    * **「Dual Hub DMVPN with strict active/backup pathing」** ➔ DMVPN Dual-Cloud (DHDD) Phase 3 ＋ `delay` 調整。
    * **「FlexVPN Server Cluster Load Balancing」** ➔ FlexVPN Load Balancer (`crypto ikev2 cluster`) ＋ HSRP VIP ＋ RFC 5685 Redirect。
* **図解（VPN HA パイプラインマトリクス）**:
  ```text
  [ Client / Initiator ]
            │
            ├── AnyConnect RAVPN Cluster ──► Master VIP ──► (HTTP 302 / IKEv2 Redirect) ──► Least-Loaded Slave ASA
            │
            ├── FlexVPN Load Balancer ─────► HSRP VIP ───► (RFC 5685 Redirect) ───────────► Least-Loaded FlexVPN Server
            │
            ├── S2S Multiple Peers ────────► Primary Peer (IP1) ──► (DPD Timeout) ────────► Secondary Peer (IP2)
            │
            └── DMVPN / FlexVPN Dual-Cloud ─► Cloud 1 (Low Delay) ──► (Link Down) ───────► Cloud 2 (High Delay)
  ```
* **注意点**:
  * ラボ試験で VPN HA を設定した後は、`show vpn-sessiondb cluster`、`show failover`、`show crypto session`、`show crypto ikev2 cluster`、`show ip nhrp nhs` を必ず実行し、フェールオーバー時やリダイレクト時に通信が維持されることを確認・証明してください。

