---
layout: default
title: 2.3-FlexVPN-DMVPN-IPsec
nav_order: 3
parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.3 FlexVPN, DMVPN, and IPsec L2L tunnels

Cisco ネットワークにおける **FlexVPN**、**DMVPN (Dynamic Multipoint VPN)**、および **IPsec L2L (LAN-to-LAN / Site-to-Site) トンネル** は、拠点間（Site-to-Site）、本社・拠点間（Hub-and-Spoke）、および拠点間ダイレクト通信（Spoke-to-Spoke）を暗号化・保護し、安全なオーバーレイネットワークを構築するための集大成技術です。

CCIE Security v6.1 Blueprint **2.3 FlexVPN, DMVPN, and IPsec L2L tunnels** では、Cisco IOS / IOS-XE ルーター、Cisco ASA、および Cisco FTD (Firepower Threat Defense / Secure Firewall) における各 VPN アーキテクチャの動作原理、シグナリング手順（IKEv1 / IKEv2、NHRP）、トンネルインターフェイス形態（Static VTI, Dynamic VTI, mGRE, Crypto Map）、認可・ポリシー制御（AAA, FlexVPN Authorization Policy, Config Exchange）、VRF 統合（FVRF / IVRF）、ルーティングプロトコル連携（EIGRP, OSPF, BGP）、およびトラブルシューティング能力が多角的かつ徹底的に評価されます。

---

## 📘 概要

* **機能概要**:
  * **FlexVPN (Cisco Unified IKEv2 Architecture)**:
    * IKEv2 (RFC 7296) を標準ベースとし、従来の Site-to-Site Crypto Map、DMVPN、EzVPN、および Remote Access VPN の設定パラメータとインフラを統合した Cisco 次世代 VPN 構成フレームワーク。
    * 点対点（P2P）、Hub-and-Spoke、Spoke-to-Spoke（NHRP 併用）、Remote Access を単一の CLI 体系（`crypto ikev2`）で実現。
    * **Virtual-Template** から動的に複製（Clone）される **Dynamic Virtual Tunnel Interface (dVTI / Virtual-Access)** を使用し、AAA や IKEv2 Configuration Exchange (CP) 経由でユーザー/グループ単位のポリシー（IP アドレス割り当て、ルート注入、QoS、ACL）を動的プッシュ。
  * **DMVPN (Dynamic Multipoint VPN)**:
    * **mGRE (Multipoint GRE)**、**NHRP (Next Hop Resolution Protocol: RFC 2332)**、**IPsec (Tunnel Protection)**、および **動的ルーティングプロトコル (EIGRP / OSPF / BGP)** を組み合わせ、ハブを経由せずに拠点間（Spoke-to-Spoke）の暗号化トンネルを動的に自動確立するメッシュ型 VPN ソリューション。
    * **Phase 1**: Hub-and-Spoke 構成。全トラフィックが Hub を通過（Spoke は P2P GRE または mGRE、Hub は mGRE）。
    * **Phase 2**: Single-Subnet モード。Spoke 間でダイレクトに NHRP 解決を実施して Spoke-to-Spoke トンネルを確立。ルーティングプロトコルの Next-Hop 保持（`no ip next-hop-self`）が必須。
    * **Phase 3**: **Hierarchical / Summarization 対応モード**。Hub が `ip nhrp redirect` を送信し、Spoke が `ip nhrp shortcut` を受信して動的 NHRP 概略ルート（`H` ルート）を RIB/FIB に挿入。大規模環境における集約ルーティングに対応。
  * **IPsec L2L (LAN-to-LAN) Tunnels**:
    * 2 つの対向拠点（ファイアウォールまたはルーター）間を 1 対 1 で恒久接続する標準的な IPsec トンネル。
    * **Policy-Based VPN (Crypto Map)**: 通信トラフィック（Interesting Traffic）を Extended ACL で定義し、一致したパケットが発生した時点で IKE / IPsec SA を動的にネゴシエーションして暗号化。マルチキャストや動的ルーティングプロトコルは非サポート。
    * **Route-Based VPN (Static VTI: sVTI)**: 仮想トンネルインターフェイス（`interface Tunnel`）に IP アドレスおよび IPsec Profile (`tunnel protection ipsec profile`) を割り当て、マルチキャスト通信や動的ルーティングプロトコルを透過処理。パケットの暗号化判断はルーティングテーブル（RIB）に依存。
* **利用目的**:
  * 公衆インターネットや MPLS 網上で、企業拠点間通信の機密性 (Confidentiality)、完全性 (Integrity)、認証 (Authentication)、および抗否認性 (Anti-Replay) を保証する。
  * 多数の拠点が存在するエンタープライズにおいて、手動の 1 対 1 メッシュ設定を排除し、自動的な Spoke-to-Spoke メッシュ接続を実現する。
* **どのような場面で利用するか**:
  * **FlexVPN**: 最新の IOS-XE WAN エッジにおける統合型 Hub-and-Spoke / Remote Access / Spoke-to-Spoke 展開。
  * **DMVPN Phase 3**: 大規模企業 WAN 網（数百〜数千拠点）で、ルート集約（Summarization）を維持しながら拠人間直接通信を行う環境。
  * **ASA / FTD IPsec L2L**: クラウド（AWS / Azure）境界や他社ファイアウォール（Palo Alto, Fortinet）との 1 対 1 拠点間暗号化接続。

---

## 🔑 要点

| 項目 | FlexVPN | DMVPN (Phase 1 / 2 / 3) | IPsec L2L (Crypto Map) | IPsec L2L (sVTI) |
| :--- | :--- | :--- | :--- | :--- |
| **鍵交換プロトコル** | **IKEv2 限定** | IKEv1 / IKEv2 | IKEv1 / IKEv2 | IKEv1 / IKEv2 |
| **トンネル形態** | P2P VTI, **Virtual-Template / dVTI** | **mGRE** (Hub/Spoke), P2P GRE | なし (Policy-Based) | **Static VTI (sVTI)** |
| **Spoke-to-Spoke 動的確立** | **対応** (NHRP ＋ dVTI) | **対応** (Phase 2 / Phase 3) | 非対応 (全メッシュ設定が必要) | 非対応 (手動固定トンネル) |
| **ルーティングプロトコル** | EIGRP, OSPF, BGP, **IKEv2 Routing** | EIGRP, OSPF, BGP | **非対応** (Static / RRI のみ) | **対応** (EIGRP, OSPF, BGP) |
| **AAA / Policy 統合** | **完全統合** (RADIUS / AAA / CP) | 限定的 | 制限あり (ASA Group-Policy 等) | 制限あり |
| **ルート集約の可否** | **対応** (IKEv2 Routing / BGP) | Phase 1: 可<br>Phase 2: **不可**<br>Phase 3: **可能** | N/A | **対応** (通常のルーティング) |
| **対応アプライアンス** | IOS / IOS-XE, ASA (VTI 接続) | IOS / IOS-XE (ASA/FTD は非対応) | IOS, ASA, FTD (FMC/FDM) | IOS, ASA (9.7+), FTD (6.7+) |

---

## 🏗 動作原理

### 1. FlexVPN Spoke-to-Spoke 動的接続 ＆ IKEv2 Configuration Exchange フロー

FlexVPN では、Spoke 1 と Spoke 2 間のダイレクト通信が発生すると、NHRP Redirect メッセージをトリガーとして IKEv2 セッションがネゴシエーションされ、Dynamic VTI (dVTI) がオンデマンドで生成されます。

```text
[ Spoke 1 (Client) ] ───(1) Packet to Spoke 2 (via Hub) ───► [ FlexVPN Hub (Server) ]
        │                                                           │
        │                                                           ├── Forward Packet to Spoke 2
        │                                                           └── Sends NHRP Redirect to Spoke 1
        │
        ├───(2) Receives NHRP Redirect ──────────────────────────────┘
        │
        ├───(3) Sends NHRP Resolution Request (via Hub) ───────────► [ Spoke 2 ]
        │                                                           │
        │ ◄──(4) Receives NHRP Resolution Reply (with NBMA IP) ─────┘
        │
        ├───(5) Direct IKEv2 SA Negotiation (IKE_SA_INIT / IKE_AUTH) ─► [ Spoke 2 ]
        │        ├── Auths via Pre-Shared Key / PKI
        │        └── Exchanges IKEv2 Configuration Payload (CP)
        │
        ├───(6) Dynamic VTI (Virtual-Access) Cloned from Virtual-Template
        │
        ▼
[ Spoke 1 Dynamic VTI ] ◄═══════════════ (7) Direct IPsec Tunnel ═══════════════► [ Spoke 2 Dynamic VTI ]
```

---

### 2. DMVPN Phase 3 (NHRP Redirect ＆ Shortcut) トラフィック制御フロー

DMVPN Phase 3 では、Hub でルート集約を行っていても、Hub がデータパケットの着信・送信ポート（共に Tunnel0）を検知して `NHRP Redirect` を送信し、Spoke 側に動的ショートカットルート（`H` ルート）を学習させます。

```text
[ Spoke 1 ] ───(1) Packet for 10.2.0.0/24 (Matches Hub Summary Route 10.0.0.0/8) ───► [ Hub ]
                                                                                         │
                                                                                         ├── Forwards Packet to Spoke 2
                                                                                         └── Sends NHRP Traffic Indication (Redirect)
                                                                                               to Spoke 1
                                                                                                 │
[ Spoke 1 ] ◄──(2) Receives NHRP Redirect Message ───────────────────────────────────────┘
    │
    ├───(3) Sends NHRP Resolution Request (asks NBMA IP for 10.2.0.1) ───► [ Hub ] ───► [ Spoke 2 ]
    │                                                                                        │
    │ ◄──(4) Direct NHRP Resolution Reply (contains Spoke 2 NBMA IP) ────────────────────────┘
    │
    ├───(5) Establishes Direct IPsec SA with Spoke 2 NBMA IP
    │
    └───(6) Installs NHRP Shortcut Route into RIB/FIB:
              10.2.0.1/32 [250/1] via 10.0.0.2 (Tunnel0) % Next Hop Override
```

---

## ⚙ 動作シーケンス

1. **初期アンダーレイ接続 ＆ トンネル登録フェーズ**:
   * **FlexVPN / DMVPN Spoke**: 起動時、静的に定義された Hub の NBMA (Public) IP 宛てに IKEv1/IKEv2 SA ネゴシエーションを開始。
   * トンネルが保護された後、Spoke は NHRP Registration Request を Hub (NHS: Next Hop Server) へ送信し、自機の Tunnel IP と NBMA IP のマッピング情報を Hub の NHRP キャッシュテーブルに登録。
2. **ルーティング情報伝播 ＆ ポリシー適用フェーズ**:
   * **FlexVPN**: IKEv2 Configuration Exchange (CP) により、Hub から Spoke へ IP アドレス、DNS、および保護対象サブネット（`route set interface` または `route set remote`）が動的にプッシュされる。また、必要に応じて BGP / EIGRP が Overlay 上でネイバー確立。
   * **DMVPN Phase 3**: Hub は各 Spoke へ親ルート（例: `10.0.0.0/8` またはデフォルトルート `0.0.0.0/0`）のみをルーティング更新（EIGRP / OSPF）で広報。
3. **データ通信発起 ＆ 動的ショートカット解決フェーズ**:
   * Spoke 1 の LAN クライアントから Spoke 2 の LAN クライアント宛てにパケットが送信される。
   * 初期パケットは Hub 経由で転送される。
   * **Hub 側の判断**: Hub はパケットを Tunnel0 から受信し、同一の Tunnel0 から送信することを検知。Hub は **`ip nhrp redirect`**（または FlexVPN の Traffic Indication）メッセージを Spoke 1 へ送出。
4. **ダイレクト Tunnel 成立 ＆ RIB カウントオーバーライド**:
   * Spoke 1 は Redirect を受けると、Spoke 2 の宛先 IP に対する **NHRP Resolution Request** を発信。
   * Spoke 2 から Direct Reply を受信すると、Spoke 1 と Spoke 2 間に即座に直接 IKEv2 / IPsec SA が確立される。
   * Spoke 1 の RIB/FIB に `/32`（または正確なサブネット）の **NHRP Shortcut Route (`H`)** が動的挿入され、以降のデータパケットは Hub をバイパスして Spoke 間で直接高速転送される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **FlexVPN の必須コンポーネント構成順序**:
   * **IKEv2 Proposal / Policy**: `crypto ikev2 proposal` (Encryption, Integrity, PRF, Group) ➔ `crypto ikev2 policy`。
   * **IKEv2 Keyring**: `crypto ikev2 keyring` (Peer 条件と PSK)。
   * **IKEv2 Profile**: `crypto ikev2 profile` ➔ `match identity remote` ➔ `authentication local/remote` ➔ `keyring local` ➔ `aaa authorization group` ➔ **`virtual-template <N>`**。
   * **IPsec Profile**: `crypto ipsec profile` ➔ `set transform-set` (または Smart Default) ➔ `set ikev2-profile`。
   * **Virtual-Template**: `interface Virtual-Template1 type tunnel` ➔ `ip unnumbered` ➔ `tunnel mode ipsec ipv4` (または `mode auto`) ➔ `tunnel protection ipsec profile`。
   * **AAA 宣言要件**: FlexVPN で Group Authorization を使用する場合、**`aaa new-model`** および **`aaa authorization network default local`**（または RADIUS）の事前設定が**絶対必須**。これを怠ると dVTI が生成されずトンネルが確立しません。
2. **DMVPN Phase 1 vs Phase 2 vs Phase 3 の設定差分（超高精度照合）**:
   * **Phase 1**:
     * Hub: `tunnel mode gre multipoint`, `ip nhrp map multicast dynamic`
     * Spoke: `tunnel mode gre ip` (P2P GRE), `tunnel destination <HUB_NBMA_IP>`
   * **Phase 2**:
     * Hub: `tunnel mode gre multipoint`, `no ip next-hop-self` (EIGRP), `no ip split-horizon eigrp <AS>`
     * Spoke: `tunnel mode gre multipoint`, `ip nhrp map <HUB_TUNNEL_IP> <HUB_NBMA_IP>`, `ip nhrp map multicast <HUB_NBMA_IP>`, `ip nhrp nhs <HUB_TUNNEL_IP>`
   * **Phase 3 (最頻出)**:
     * Hub: `tunnel mode gre multipoint`, **`ip nhrp redirect`**
     * Spoke: `tunnel mode gre multipoint`, **`ip nhrp shortcut`**, `ip nhrp nhs <HUB_TUNNEL_IP> nbma <HUB_NBMA_IP> multicast`
3. **ASA / FTD IPsec L2L ＆ Route-Based VTI (sVTI) のポイント**:
   * **Cisco ASA sVTI**: ASA 9.7(1) 以降対応。`interface Tunnel0` ➔ `nameif vti` ➔ `ip address 10.99.1.1 255.255.255.0` ➔ `tunnel source interface outside` ➔ `tunnel destination <PEER_IP>` ➔ `tunnel mode ipsec ipv4` ➔ `tunnel protection ipsec profile <PROF>`.
   * **Crypto Map からの移行**: ACL (`match address`) が不要となり、ルーティングテーブル（Static Route や BGP）によって暗号化対象を決定。
   * **FTD (FMC) Prefilter Policy 要件**: FTD で S2S VPN 通信を処理する際、FMC の Access Control Policy (ACP) でインスペクションされる前に、**Prefilter Policy で IKE (UDP 500/4500) および ESP (Protocol 50) を Fastpath 通過させる** 設定が問われます（これがないとパケットドロップや Latency が増加する）。
4. **VRF Aware VPN (FVRF vs IVRF)**:
   * **Front-Door VRF (FVRF)**: アンダーレイ（パブリックインターネット側）を独立した VRF に分離。`interface Tunnel` 上で **`tunnel vrf <FVRF_NAME>`** を指定（`ip vrf forwarding` は指定しない）。
   * **Inside VRF (IVRF)**: トンネル内部のオーバーレイ通信を特定 VRF に割り当て。`interface Tunnel` 上で **`ip vrf forwarding <IVRF_NAME>`** を指定。

---

## 🛠 設定方法

### 1. FlexVPN Hub-and-Spoke ＋ Spoke-to-Spoke (IOS-XE 完全構成)

```bash
! =========================================================
! FlexVPN Hub Configuration (IOS-XE)
! =========================================================
aaa new-model
aaa authorization network default local

! IKEv2 Authorization Policy (IP Pool & Route Push)
ip local pool FLEX_POOL 10.200.1.100 10.200.1.200

crypto ikev2 authorization policy FLEX_AUTHOR_POLICY
 pool FLEX_POOL
 route set interface

! IKEv2 Keyring & Profile
crypto ikev2 keyring FLEX_KEYRING
 peer SPOKES
  address 0.0.0.0 0.0.0.0
  pre-shared-key local cisco123
  pre-shared-key remote cisco123

crypto ikev2 profile FLEX_HUB_PROFILE
 match identity remote any
 authentication local pre-share
 authentication remote pre-share
 keyring local FLEX_KEYRING
 aaa authorization group psk list default FLEX_AUTHOR_POLICY
 virtual-template 1

! IPsec Profile
crypto ipsec profile FLEX_IPSEC_PROFILE
 set ikev2-profile FLEX_HUB_PROFILE

! Virtual-Template Interface (dVTI)
interface Virtual-Template1 type tunnel
 ip unnumbered Loopback0
 ip nhrp network-id 100
 ip nhrp redirect                       ! Spoke-to-Spoke 誘導用
 tunnel protection ipsec profile FLEX_IPSEC_PROFILE

! =========================================================
! FlexVPN Spoke Configuration (IOS-XE)
! =========================================================
aaa new-model
aaa authorization network default local

crypto ikev2 authorization policy default
 route set interface

crypto ikev2 keyring FLEX_SPOKE_KEYRING
 peer HUB
  address 192.168.100.1
  pre-shared-key local cisco123
  pre-shared-key remote cisco123

crypto ikev2 profile FLEX_SPOKE_PROFILE
 match identity remote address 192.168.100.1 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local FLEX_SPOKE_KEYRING
 aaa authorization group default default

crypto ipsec profile FLEX_SPOKE_IPSEC
 set ikev2-profile FLEX_SPOKE_PROFILE

! Spoke Static Tunnel Interface
interface Tunnel0
 ip address negotiated
 ip nhrp network-id 100
 ip nhrp shortcut Virtual-Template1     ! Spoke-to-Spoke dVTI トリガー
 tunnel source GigabitEthernet0/0
 tunnel destination 192.168.100.1
 tunnel protection ipsec profile FLEX_SPOKE_IPSEC

! Spoke Dynamic Virtual-Template (Spoke-to-Spoke 用)
interface Virtual-Template1 type tunnel
 ip unnumbered Loopback0
 ip nhrp network-id 100
 ip nhrp shortcut Virtual-Template1
 tunnel protection ipsec profile FLEX_SPOKE_IPSEC
```

---

### 2. DMVPN Phase 3 (EIGRP / IPsec Profile 完全構成)

```bash
! =========================================================
! DMVPN Phase 3 Hub Configuration
! =========================================================
crypto ikev2 proposal DMVPN_PROP
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy DMVPN_IKE2_POLICY
 proposal DMVPN_PROP

crypto ikev2 keyring DMVPN_KEYRING
 peer SPOKES
  address 0.0.0.0 0.0.0.0
  pre-shared-key local dmvpnkey123
  pre-shared-key remote dmvpnkey123

crypto ikev2 profile DMVPN_IKE2_PROFILE
 match identity remote any
 authentication local pre-share
 authentication remote pre-share
 keyring local DMVPN_KEYRING

crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport                         ! GRE over IPsec には transport モードが最適

crypto ipsec profile DMVPN_IPSEC_PROFILE
 set transform-set TS_DMVPN
 set ikev2-profile DMVPN_IKE2_PROFILE

interface Tunnel0
 ip address 10.100.1.1 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 1000
 ip nhrp redirect                       ! Phase 3 必須
 ip nhrp map multicast dynamic
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN_IPSEC_PROFILE

router eigrp 100
 network 10.100.1.0 0.0.0.255
 no auto-summary

! =========================================================
! DMVPN Phase 3 Spoke Configuration
! =========================================================
crypto ikev2 keyring DMVPN_SPOKE_KEYRING
 peer HUB
  address 192.168.100.1
  pre-shared-key local dmvpnkey123
  pre-shared-key remote dmvpnkey123

crypto ikev2 profile DMVPN_SPOKE_IKE2_PROFILE
 match identity remote address 192.168.100.1 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local DMVPN_SPOKE_KEYRING

crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile DMVPN_SPOKE_IPSEC
 set transform-set TS_DMVPN
 set ikev2-profile DMVPN_SPOKE_IKE2_PROFILE

interface Tunnel0
 ip address 10.100.1.2 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 1000
 ip nhrp shortcut                       ! Phase 3 必須
 ip nhrp nhs 10.100.1.1 nbma 192.168.100.1 multicast
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN_SPOKE_IPSEC

router eigrp 100
 network 10.100.1.0 0.0.0.255
 no auto-summary
```

---

### 3. Cisco ASA Route-Based IPsec VTI (sVTI) 構成

```bash
! ===== ASA sVTI Configuration =====
crypto ikev2 policy 10
 encryption aes-256
 integrity sha256
 group 14
 prf sha256
 lifetime 86400

crypto ikev2 enable outside

group-policy VTI_GP internal
group-policy VTI_GP attributes
 vpn-tunnel-protocol ikev2

tunnel-group 198.51.100.2 type ipsec-l2l
tunnel-group 198.51.100.2 general-attributes
 default-group-policy VTI_GP
tunnel-group 198.51.100.2 ipsec-attributes
 ikev2 local-authentication pre-shared-key cisco123
 ikev2 remote-authentication pre-shared-key cisco123

crypto ipsec ikev2 ipsec-proposal VTI_PROPOSAL
 protocol esp encryption aes-256
 protocol esp integrity sha-256

crypto ipsec profile VTI_PROFILE
 set ikev2 ipsec-proposal VTI_PROPOSAL

interface Tunnel1
 nameif vti-branch
 security-level 50
 ip address 10.250.1.1 255.255.255.252
 tunnel source interface outside
 tunnel destination 198.51.100.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile VTI_PROFILE

! ルーティングにより暗号化対象を指定
route vti-branch 172.16.20.0 255.255.255.0 10.250.1.2
```

---

## 🔍 検証コマンド

| 目的 | コマンド (IOS / ASA / FTD) | 表示確認ポイント |
| :--- | :--- | :--- |
| **IKEv2 SA 確立状態の確認** | <code>show crypto ikev2 sa</code> / <code>show crypto ikev2 sa detail</code> | `Status: READY`, `ENCR: AES-CBC`, `Hash: SHA256`, `Local/Remote Identity` |
| **IPsec SA 統計・暗号化カウンター確認** | <code>show crypto ipsec sa</code> / <code>show crypto ipsec sa peer <IP></code> | `#pkts encaps/encrypt`, `#pkts decaps/decrypt`, `inbound/outbound esp sas` |
| **NHRP キャッシュ ＆ ショートカット情報** | <code>show ip nhrp</code> / <code>show ip nhrp shortcut</code> | `Type: dynamic / shortcut`, `NBMA Address`, `Flags: used router valid` |
| **DMVPN トンネルセッション詳細** | <code>show dmvpn</code> / <code>show dmvpn detail</code> | `Type: mGRE`, `State: UP`, `NBMA Address`, `Tunnel IP` |
| **FlexVPN セッション ＆ 動的 VTI 確認** | <code>show crypto session detail</code> | `Interface: Virtual-Access1`, `Session Status: UP-ACTIVE`, `Pushed Routes` |
| **ASA / FTD VTI インターフェイス確認** | <code>show interface Tunnel <N></code> / <code>show crypto ikev2 sa</code> | `Tunnel source/destination`, `Line protocol is up`, `IPsec Profile Name` |
| **IKEv2 ネゴシエーションデバッグ** | <code>debug crypto ikev2</code> / <code>debug crypto ikev2 internal</code> | Phase 1/Phase 2 AUTH 交換、CP Payload、Proposal 一致 |
| **NHRP パケット交換デバッグ** | <code>debug nhrp packet</code> / <code>debug nhrp condition</code> | Redirect, Resolution Request/Reply 送受信 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **FlexVPN で Spoke の dVTI が生成されずトンネルが UP しない** | AAA モデルが不完全、または `aaa authorization network default local` が未設定 | <code>show crypto ikev2 profile</code><br><code>show aaa session</code> | `aaa new-model` および `aaa authorization network default local` を投入する。 |
| **DMVPN Phase 3 で Spoke 間ダイレクト通信が発生せず全パケットが Hub を通過する** | Hub で `ip nhrp redirect` が未設定、または Spoke で `ip nhrp shortcut` が未設定 | <code>show ip nhrp shortcut</code><br><code>show dmvpn</code> | Hub の Tunnel インターフェイスで `ip nhrp redirect`、Spoke で `ip nhrp shortcut` を有効化する。 |
| **mGRE / VTI 上で EIGRP ネイバーがフラッピング（Up/Down）を繰り返す** | IPsec トンネル通信による MTU パケットの断片化（Fragmentation） | <code>show ip interface Tunnel<N></code> | Tunnel インターフェイスで `ip mtu 1400` および `ip tcp adjust-mss 1360` を設定する。 |
| **ASA sVTI トンネル経由で対向ルーターとの通信が暗号化されない** | `tunnel destination` の不一致、または IKEv2 Proposal / PSK の不一致 | <code>show crypto ikev2 sa</code><br><code>show crypto ipsec sa</code> | 両端で `tunnel destination`、IKEv2 Proposal、および PSK を完全に一致させる。 |
| **FTD FMC で構成した S2S VPN 経由で最初の通信で大幅な遅延・パケットドロップが起きる** | FTD の Access Control Policy で IKE/ESP 通信が Snort DPI に引き渡されている | <code>show access-list</code> | FMC の Prefilter Policy で IKE (UDP 500/4500) および ESP (50) を Fastpath 設定する。 |

---

## ⚠ 制限事項

* **Cisco ASA / FTD での DMVPN 非サポート**:
  * ASA および FTD アプライアンスは mGRE および NHRP プロトコルをサポートしないため、**DMVPN の Hub や Spoke として動作させることはできません**（※L2L IPsec または VTI 接続のみ対応）。
* **FlexVPN IKEv1 非サポート**:
  * FlexVPN は IKEv2 専用のアーキテクチャであり、IKEv1 での動作は完全に非サポートです。

---

## 🔄 他技術との関連

* **NAT-Traversal (NAT-T)**: アンダーレイに NAT アプライアンスが存在する場合、ESP (Protocol 50) パケットを UDP 4500 で自動カプセル化。
* **VRF-Aware IPsec (FVRF / IVRF)**: 前面 VRF (FVRF) でアンダーレイを隔離し、内部 VRF (IVRF) でマルチテナント・オーバーレイを分離。
* **Cisco FMC Prefilter Policy**: FTD において S2S VPN トラフィック（IKE/ESP）を Fastpath 化して Snort インスペクション負荷を排除。

---

## 🧩 比較表

### 1. DMVPN Phase 1 vs Phase 2 vs Phase 3

| 比較項目 | DMVPN Phase 1 | DMVPN Phase 2 | DMVPN Phase 3 |
| :--- | :--- | :--- | :--- |
| **Spoke トンネルモード** | Point-to-Point GRE | **Multipoint GRE (mGRE)** | **Multipoint GRE (mGRE)** |
| **Spoke-to-Spoke トラフィック** | **Hub 経由** (ダイレクト不可) | **ダイレクト確立** | **ダイレクト確立** |
| **Hub ルート集約 (Summarization)**| **可能** | **不可** (全詳細ルートが必要) | **完全対応** (NHRP Redirect/Shortcut) |
| **EIGRP Next-Hop** | Hub が自身に書き換え (`next-hop-self`) | **そのまま保持** (`no ip next-hop-self`) | **Hub 保持** (Redirect で別ルート挿入) |
| **必須 NHRP コマンド** | 基本 NHRP | `ip nhrp map / nhs` | **`ip nhrp redirect` (Hub) / `shortcut` (Spoke)** |

### 2. FlexVPN vs DMVPN

| 比較項目 | FlexVPN | DMVPN |
| :--- | :--- | :--- |
| **キー管理プロトコル** | **IKEv2 専用** | IKEv1 ＆ IKEv2 |
| **中心的なポリシー定義** | **AAA ＆ IKEv2 Authorization Policy** | 手動 GRE / NHRP 定義 |
| **インターフェイス実装** | P2P VTI, **Virtual-Template / dVTI** | **mGRE** |
| **構成の柔軟性** | S2S, RA, Hub-Spoke を**単一 CLI で統合** | 拠点間接続・メッシュに特化 |

---

## 💡 ベストプラクティス

1. **IPsec Transport Mode の採用**:
   * GRE や mGRE、VTI トンネル上で IPsec を適用する場合、カプセル化の重複（Outer IP Header の二重化）を防ぐため、Transform Set にて **`mode transport`** を指定してオーバーヘッドを削減します。
2. **MTU ＆ MSS の明確な調整**:
   * GRE (24 bytes) ＋ IPsec ESP (約 50-60 bytes) のヘッダー付加によるフラグメンテーションを防ぐため、Tunnel インターフェイス上で **`ip mtu 1400`** および **`ip tcp adjust-mss 1360`** を標準構成とします。

---

## 📝 ラボ学習・設定サンプル例

### 例1: IOS-XE FlexVPN Hub-and-Spoke 構成（IKEv2 ＋ PSK ＋ Local Pool）

#### 問題
Hub1 と Spoke1 / Spoke2 間で FlexVPN Hub-and-Spoke を構築せよ。Spoke には Local Pool `10.200.1.0/24` より IP アドレスを動的割当し、IKEv2 Authorization Policy を用いて保護対象ネットワーク情報をプッシュせよ。

#### 要件
* 共通 PSK: `cisco_flex_123`
* Hub 側は `Virtual-Template1 type tunnel` を使用し、`aaa authorization network default local` を有効化すること。

#### 設定例

```bash
! ===== Hub1 Configuration =====
aaa new-model
aaa authorization network default local

ip local pool FLEX_POOL 10.200.1.10 10.200.1.200

crypto ikev2 authorization policy FLEX_AUTHOR
 pool FLEX_POOL
 route set interface

crypto ikev2 keyring FLEX_KEYRING
 peer SPOKES
  address 0.0.0.0 0.0.0.0
  pre-shared-key local cisco_flex_123
  pre-shared-key remote cisco_flex_123

crypto ikev2 profile FLEX_PROFILE
 match identity remote any
 authentication local pre-share
 authentication remote pre-share
 keyring local FLEX_KEYRING
 aaa authorization group psk list default FLEX_AUTHOR
 virtual-template 1

crypto ipsec profile FLEX_IPSEC
 set ikev2-profile FLEX_PROFILE

interface Loopback0
 ip address 10.100.1.1 255.255.255.255

interface Virtual-Template1 type tunnel
 ip unnumbered Loopback0
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile FLEX_IPSEC

! ===== Spoke1 Configuration =====
aaa new-model
aaa authorization network default local

crypto ikev2 authorization policy default
 route set interface

crypto ikev2 keyring FLEX_KEYRING
 peer HUB
  address 192.168.100.1
  pre-shared-key local cisco_flex_123
  pre-shared-key remote cisco_flex_123

crypto ikev2 profile FLEX_SPOKE_PROFILE
 match identity remote address 192.168.100.1 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local FLEX_KEYRING
 aaa authorization group default default

crypto ipsec profile FLEX_SPOKE_IPSEC
 set ikev2-profile FLEX_SPOKE_PROFILE

interface Tunnel0
 ip address negotiated
 tunnel source GigabitEthernet0/0
 tunnel destination 192.168.100.1
 tunnel protection ipsec profile FLEX_SPOKE_IPSEC
```

#### 検証方法
* Spoke1 で `show crypto ikev2 sa` および `show ip route static` を実行し、`READY` ステートおよび Hub から動的に受領したスタティックルートが存在することを確認。

---

### 例2: IOS-XE FlexVPN Spoke-to-Spoke 構成（NHRP ＋ Dynamic VTI）

#### 問題
例1 の FlexVPN 構成を拡張し、Spoke1 と Spoke2 間で Hub を経由しない直接暗号化トンネル（Spoke-to-Spoke dVTI）が自動形成されるように設定せよ。

#### 要件
* Hub 側で `ip nhrp redirect` を有効化すること。
* Spoke 側で `ip nhrp shortcut Virtual-Template1` を適用すること。

#### 設定例

```bash
! ===== Hub1 追加設定 =====
interface Virtual-Template1 type tunnel
 ip nhrp network-id 100
 ip nhrp redirect

! ===== Spoke1 / Spoke2 追加設定 =====
interface Tunnel0
 ip nhrp network-id 100
 ip nhrp shortcut Virtual-Template1

interface Virtual-Template1 type tunnel
 ip unnumbered Loopback0
 ip nhrp network-id 100
 ip nhrp shortcut Virtual-Template1
 tunnel protection ipsec profile FLEX_SPOKE_IPSEC
```

#### 検証方法
* Spoke1 から Spoke2 の Loopback へ Ping を実行し、`show crypto session` を確認して `Virtual-Access1` 経由の直接 IPsec SA が確立されることを確認。

---

### 例3: DMVPN Phase 3 (EIGRP ＋ IKEv2 ＋ IPsec Profile)

#### 問題
Hub1、Spoke1、Spoke2 間で DMVPN Phase 3 を構築せよ。IKEv2 認証を使用し、EIGRP AS 100 でオーバーレイのフルメッシュ動的ルーティングを実現せよ。

#### 要件
* 共通 PSK: `dmvpn_ike2_key`
* Hub 側で `ip nhrp redirect`、Spoke 側で `ip nhrp shortcut` を有効化すること。

#### 設定例

```bash
! ===== Hub1 Configuration =====
crypto ikev2 keyring DMVPN_KEYRING
 peer SPOKES
  address 0.0.0.0 0.0.0.0
  pre-shared-key local dmvpn_ike2_key
  pre-shared-key remote dmvpn_ike2_key

crypto ikev2 profile DMVPN_PROFILE
 match identity remote any
 authentication local pre-share
 authentication remote pre-share
 keyring local DMVPN_KEYRING

crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile DMVPN_IPSEC
 set transform-set TS_DMVPN
 set ikev2-profile DMVPN_PROFILE

interface Tunnel0
 ip address 10.100.1.1 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 100
 ip nhrp redirect
 ip nhrp map multicast dynamic
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN_IPSEC

router eigrp 100
 network 10.100.1.0 0.0.0.255
 no auto-summary

! ===== Spoke1 Configuration =====
crypto ikev2 keyring DMVPN_KEYRING
 peer HUB
  address 192.168.100.1
  pre-shared-key local dmvpn_ike2_key
  pre-shared-key remote dmvpn_ike2_key

crypto ikev2 profile DMVPN_PROFILE
 match identity remote address 192.168.100.1 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local DMVPN_KEYRING

crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile DMVPN_IPSEC
 set transform-set TS_DMVPN
 set ikev2-profile DMVPN_PROFILE

interface Tunnel0
 ip address 10.100.1.2 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 100
 ip nhrp shortcut
 ip nhrp nhs 10.100.1.1 nbma 192.168.100.1 multicast
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN_IPSEC

router eigrp 100
 network 10.100.1.0 0.0.0.255
 no auto-summary
```

#### 検証方法
* Spoke1 で `show ip nhrp shortcut` および `show ip route` を実行し、Spoke2 宛てのルートが `H` (NHRP) として FIB にロードされることを確認。

---

### 例4: DMVPN Front-Door VRF (FVRF) によるアンダーレイ分離

#### 問題
DMVPN Hub1 において、パブリックアンダーレイインターフェイス `Gi0/0` を VRF `OUTSIDE`（FVRF）に分離し、オーバーレイ Tunnel0 をグローバル VRF で動作させよ。

#### 要件
* `tunnel vrf OUTSIDE` コマンドを使用すること。

#### 設定例

```bash
! ===== Hub1 Configuration =====
vrf definition OUTSIDE
 address-family ipv4
 exit-address-family

interface GigabitEthernet0/0
 vrf forwarding OUTSIDE
 ip address 192.168.100.1 255.255.255.0

ip route vrf OUTSIDE 0.0.0.0 0.0.0.0 192.168.100.254

crypto ikev2 keyring DMVPN_KEYRING vrf OUTSIDE
 peer SPOKES
  address 0.0.0.0 0.0.0.0
  pre-shared-key local dmvpn_ike2_key
  pre-shared-key remote dmvpn_ike2_key

crypto ikev2 profile DMVPN_PROFILE
 match fvrf OUTSIDE
 match identity remote any
 authentication local pre-share
 authentication remote pre-share
 keyring local DMVPN_KEYRING

interface Tunnel0
 ip address 10.100.1.1 255.255.255.0
 ip nhrp network-id 100
 ip nhrp redirect
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel vrf OUTSIDE                     ! FVRF バインド
 tunnel protection ipsec profile DMVPN_IPSEC
```

#### 検証方法
* `show crypto ikev2 sa` を実行し、`fvrf: OUTSIDE` / `ivrf: in0:f0`（Global）として SA が正しく管理されていることを確認。

---

### 例5: Cisco ASA Static VTI (sVTI) 構成 (IKEv2)

#### 問題
ASA1 と 対向ルーター R1 の目で IKEv2 を使用した Route-Based Static VTI (sVTI) トンネルを構築せよ。

#### 要件
* ASA 側 VTI IP: `10.250.1.1/30`, R1 側 VTI IP: `10.250.1.2/30`
* 共通 PSK: `asa_svti_key`

#### 設定例

```bash
! ===== ASA1 Configuration =====
crypto ikev2 policy 10
 encryption aes-256
 integrity sha256
 group 14
 prf sha256

crypto ikev2 enable outside

group-policy VTI_GP internal
group-policy VTI_GP attributes
 vpn-tunnel-protocol ikev2

tunnel-group 198.51.100.2 type ipsec-l2l
tunnel-group 198.51.100.2 general-attributes
 default-group-policy VTI_GP
tunnel-group 198.51.100.2 ipsec-attributes
 ikev2 local-authentication pre-shared-key asa_svti_key
 ikev2 remote-authentication pre-shared-key asa_svti_key

crypto ipsec ikev2 ipsec-proposal VTI_PROP
 protocol esp encryption aes-256
 protocol esp integrity sha-256

crypto ipsec profile VTI_PROF
 set ikev2 ipsec-proposal VTI_PROP

interface Tunnel1
 nameif vti-r1
 security-level 100
 ip address 10.250.1.1 255.255.255.252
 tunnel source interface outside
 tunnel destination 198.51.100.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile VTI_PROF

route vti-r1 172.16.100.0 255.255.255.0 10.250.1.2
```

#### 検証方法
* ASA1 で `show interface Tunnel1` および `show crypto ikev2 sa` を実行し、VTI が UP しトラフィックが透過することを確認。

---

### 例6: Cisco ASA Crypto Map (Policy-Based) 拠点間 L2L トンネル

#### 問題
ASA1 と ASA2 の間で、`10.1.0.0/16` と `10.2.0.0/16` 間のトラフィックを保護する標準的な IKEv2 Policy-Based L2L トンネルを構築せよ。

#### 要件
* 共通 PSK: `cisco_l2l_key`

#### 設定例

```bash
! ===== ASA1 Configuration =====
access-list L2L_ACL extended permit ip 10.1.0.0 255.255.0.0 10.2.0.0 255.255.0.0

crypto ikev2 policy 20
 encryption aes-256
 integrity sha256
 group 14
 prf sha256

crypto ikev2 enable outside

crypto ipsec ikev2 ipsec-proposal L2L_PROP
 protocol esp encryption aes-256
 protocol esp integrity sha-256

tunnel-group 203.0.113.2 type ipsec-l2l
tunnel-group 203.0.113.2 ipsec-attributes
 ikev2 local-authentication pre-shared-key cisco_l2l_key
 ikev2 remote-authentication pre-shared-key cisco_l2l_key

crypto map OUTSIDE_MAP 10 match address L2L_ACL
crypto map OUTSIDE_MAP 10 set peer 203.0.113.2
crypto map OUTSIDE_MAP 10 set ikev2 ipsec-proposal L2L_PROP
crypto map OUTSIDE_MAP interface outside
```

#### 検証方法
* `show crypto ipsec sa` を確認し、`#pkts encaps/encrypt` カウンターが増加することを確認。

---

### 例7: FTD FMC S2S VPN ＆ Prefilter Fastpath 構成

#### 問題
FMC 管理下の FTD1 と 対向ルーター R1 間で S2S IPsec VPN を構成し、IKE/ESP パケットを Prefilter Policy で Fastpath 設定せよ。

#### 要件
* Prefilter Policy にて UDP 500/4500 および Protocol 50 (ESP) を Fastpath 化すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Access Control > Prefilter ➔ "New Policy"
   - Rule 1: Name "Allow_IKE", Protocol UDP, Ports 500, 4500 ➔ Action: Fastpath
   - Rule 2: Name "Allow_ESP", Protocol 50 (ESP) ➔ Action: Fastpath
2. Devices > VPN > Site to Site ➔ "Add Tunnel"
   - Topology Name: FTD_R1_S2S
   - Protocol: IKEv2
   - Endpoint A: FTD1 (Outside Interface)
   - Endpoint B: R1 (IP: 198.51.100.2)
   - Protected Networks: FTD_LAN (10.10.0.0/16) <-> R1_LAN (172.16.0.0/16)
   - Authentication: Pre-shared Key (cisco123)
3. Save ＆ Deploy。
```

#### 検証方法
* FTD CLI で `show access-list | inc PREFILTER` を実行し、Fastpath ルールがアプライアンスに適用されていることを確認。

---

### 例8: IOS-XE GRE over IPsec (sVTI) 構成 (IKEv2 ＋ EIGRP)

#### 問題
R1 と R2 の間で IKEv2 を用いた IPsec 結合型 GRE トンネル (sVTI) を構築し、EIGRP AS 1 で動的ルート交換を行わせよ。

#### 要件
* `tunnel mode ipsec ipv4` を使用すること。

#### 設定例

```bash
! ===== R1 Configuration =====
crypto ikev2 keyring SVTI_KEYRING
 peer R2
  address 192.168.200.2
  pre-shared-key local svtikey123
  pre-shared-key remote svtikey123

crypto ikev2 profile SVTI_IKE2_PROFILE
 match identity remote address 192.168.200.2 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local SVTI_KEYRING

crypto ipsec transform-set TS_SVTI esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile SVTI_PROFILE
 set transform-set TS_SVTI
 set ikev2-profile SVTI_IKE2_PROFILE

interface Tunnel0
 ip address 10.200.1.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 192.168.200.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile SVTI_PROFILE

router eigrp 1
 network 10.200.1.0 0.0.0.3
```

#### 検証方法
* `show ip eigrp neighbors` および `show crypto ipsec sa` で EIGRP ネイバーが Established となり、SA が ACTIVE になることを確認。

---

### 例9: DMVPN Phase 3 ＋ PKI / 相互証明書認証 (IOS-XE)

#### 問題
DMVPN Phase 3 において、事前共有鍵 (PSK) の代わりに Cisco IOS CA より発行された RSA デジタル証明書を用いて相互認証を行わせよ。

#### 要件
* `crypto pki trustpoint CA_SERVER` を使用すること。

#### 設定例

```bash
! ===== Hub1 Configuration =====
crypto pki trustpoint MY_CA
 enrollment url http://ca.corp.local/
 subject-name CN=Hub1.corp.local
 revocation-check crl

crypto ikev2 profile DMVPN_PKI_PROFILE
 match identity remote any
 authentication local rsa-sig
 authentication remote rsa-sig
 pki trustpoint MY_CA

crypto ipsec profile DMVPN_PKI_IPSEC
 set ikev2-profile DMVPN_PKI_PROFILE

interface Tunnel0
 tunnel protection ipsec profile DMVPN_PKI_IPSEC
```

#### 検証方法
* `show crypto ikev2 sa detail` で Auth Method が `rsa-sig` と表記され、証明書で認証が成功していることを確認。

---

### 例10: FlexVPN Client (IOS-XE ➔ Hub 自動発信) 構成

#### 問題
Spoke1 ルーターにおいて `crypto ikev2 client flexvpn` を設定し、Hub1 (192.168.100.1) 宛てに自動で FlexVPN 接続を発信・維持させよ。

#### 要件
* `connect auto` コマンドを使用すること。

#### 設定例

```bash
! ===== Spoke1 Configuration =====
crypto ikev2 client flexvpn FLEX_CLIENT_CONFIG
 peer 1 192.168.100.1
 connect auto
 client connect Tunnel0

interface Tunnel0
 ip address negotiated
 tunnel source GigabitEthernet0/0
 tunnel destination dynamic
 tunnel protection ipsec profile FLEX_SPOKE_IPSEC
```

#### 検証方法
* `show crypto ikev2 client flexvpn` を実行し、`Current state: ACTIVE` と表示されることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 以下の DMVPN 構成を持つ Hub ルーターがある。Spoke 1 から Spoke 2 宛てに ICMP トラフィックを送信したが、Spoke 間ダイレクト通信が確立されず全通信が Hub を経由した。原因として正しいものはどれか。

```bash
interface Tunnel0
 ip address 10.100.1.1 255.255.255.0
 ip nhrp network-id 100
 ip nhrp map multicast dynamic
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN_IPSEC
```

* A) Hub 側で `ip nhrp map` が未設定のため。
* B) Hub 側で **`ip nhrp redirect`** コマンドが設定されていないため、Phase 3 のショートカット誘導が行われない。
* C) Spoke 側で EIGRP が無効化されているため。
* D) `tunnel mode` が `ipsec ipv4` になっていないため。

**正解**: **B**  
**解説**: DMVPN Phase 3 において、Hub が Spoke 間のダイレクト通信をトリガーするには、Hub の Tunnel インターフェイス上に **`ip nhrp redirect`** コマンドを設定することが絶対必須です。

---

### 2. トラブルシュート問題

**問題**: IOS-XE 上で FlexVPN Hub を構築し、Spoke が接続を試みたが IKEv2 SA が `IN-NEGOTIATION` のまま完了せず、`Virtual-Access` インターフェイスが生成されなかった。ログには `AAA-6-USER_AUTH_FAIL` が出力されている。原因として最も適切なものはどれか。

* A) FlexVPN で PSK 認証が使用できないため。
* B) グローバル設定で **`aaa new-model`** および **`aaa authorization network default local`** が設定されていないため、IKEv2 Profile が dVTI をバインド・生成できなかった。
* C) Spoke 側で EIGRP が起動していないため。
* D) Virtual-Template の MTU が 1500 だから。

**正解**: **B**  
**解説**: FlexVPN の dVTI (Virtual-Access) 複製メカニズムは、Cisco IOS AAA インフラに完全に依存しています。**`aaa new-model`** および **`aaa authorization network ...`** の定義が欠落していると、グループポリシー処理が失敗し dVTI が生成されません。

---

### 3. Design 問題

**問題**: 全国の 500 拠点を超える企業 WAN において、ハブのルーティングテーブルサイズを抑制するために **ルート集約（Summarization / Default Route）** を実施しつつ、拠点間（Spoke-to-Spoke）での直接暗号化通信を実現したい。最適な VPN アーキテクチャはどれか。

* A) DMVPN Phase 1
* B) DMVPN Phase 2
* C) **DMVPN Phase 3**
* D) IPsec L2L Policy-Based Crypto Map

**正解**: **C**  
**解説**: DMVPN Phase 2 では Spoke 間で直接 NHRP 解決を行うため Hub でのルート集約が不可能（全詳細ルートが必要）ですが、**DMVPN Phase 3** は **`ip nhrp redirect` / `shortcut`** アーキテクチャにより、Hub で親ルート集約を行っていても Spoke 側に動的ショートカットルート（`H` ルート）をプログラム挿入できるため、大規模型集約 WAN に最適です。

---

### 4. 実装問題

**問題**: Cisco ASA (バージョン 9.18) において、従来の Policy-Based (Crypto Map) から Route-Based IPsec VPN へ移行することになった。ASA 上で利用すべきインターフェイス形式はどれか。

* A) mGRE Interface
* B) Dynamic VTI (dVTI)
* C) **Static Virtual Tunnel Interface (sVTI)** (`interface Tunnel`)
* D) Inline Set Interface

**正解**: **C**  
**解説**: Cisco ASA は 9.7(1) 以降で **Static VTI (sVTI)** (`interface Tunnel`) をサポートしており、Route-Based VPN をネイティブ実現できます（※ASA は mGRE や dVTI は非サポート）。

---

### 5. コンフィグ読解問題

**問題**: FTD アプライアンスにおいて S2S IPsec VPN を運用しているが、VPN セッションの初期確立時に大きな遅延が発生し、アプライアンスの CPU 使用率が高騰する。FMC 上で推奨されるパフォーマンス最適化設定はどれか。

* A) S2S VPN トンネルを IKEv1 にダウングレードする。
* B) FMC の **Prefilter Policy** にて、IKE (UDP 500/4500) および ESP (Protocol 50) トラフィックを **`Fastpath`** 化し、Snort DPI スキャンをバイパスさせる。
* C) FTD を Transparent モードに切り替える。
* D) Access Control Policy で Trust ルールを作成する。

**正解**: **B**  
**解説**: FTD を通過する S2S VPN カプセル化通信（IKE / ESP）を **Prefilter Policy で Fastpath 設定** することで、Snort インスペクションエンジンへの余計なパケット引き渡しを排除し、CPU 負荷軽減と遅延カットが達成されます。

---

## 🔗 参考リソース

* [Cisco IOS Flexible VPN Configuration Guide](https://www.cisco.com/)
* [Cisco Dynamic Multipoint VPN (DMVPN) Design Guide](https://www.cisco.com/)
* [Cisco ASA Series General Operations Configuration Guide: Virtual Tunnel Interfaces (VTI)](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Device Configuration Guide: Site-to-Site VPN](https://www.cisco.com/)
* [Cisco Live: Advanced FlexVPN Architecture and Troubleshooting (BRKSEC-3032)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で VPN 技術が問われた場合の構築判断チャート：
    * **「Configure IKEv2 Hub-and-Spoke with dVTI & Configuration Exchange」** ➔ **FlexVPN** (`crypto ikev2` + `Virtual-Template type tunnel` + `aaa authorization`).
    * **「Configure dynamic Spoke-to-Spoke VPN with route summarization on Hub」** ➔ **DMVPN Phase 3** (`ip nhrp redirect` / `shortcut`).
    * **「Configure Route-based IPsec VPN on Cisco ASA」** ➔ **ASA Static VTI (sVTI)** (`interface Tunnel` + `tunnel protection ipsec profile`).
    * **「Configure S2S VPN on FTD via FMC」** ➔ FMC S2S Wizard + **Prefilter Fastpath for IKE/ESP**.
* **図解（VPN 技術分類マトリクス）**:
  ```text
  [ Site-to-Site VPN Architectures ]
        │
        ├── Policy-Based (ACL-driven) ──────► Classic Crypto Map (ASA / IOS / FTD)
        │
        └── Route-Based (Interface-driven)
              ├── 1-to-1 Static ───────────► Static VTI (sVTI) / GRE over IPsec
              │
              └── Dynamic Multi-Site
                    ├── IKEv1/IKEv2 mGRE ──► DMVPN (Phase 1 / 2 / 3)
                    └── Pure IKEv2 dVTI ───► FlexVPN (Hub/Spoke, Spoke-to-Spoke)
  ```
* **注意点**:
  * ラボ試験で FlexVPN や DMVPN、sVTI を設定した後は、`show crypto ikev2 sa`、`show crypto ipsec sa`、`show ip nhrp`、`show dmvpn`、および `show crypto session` を必ず実行し、`Status: UP-ACTIVE` となり暗号化/復号パケットカウンターが増加していることを確認・証明してください。
