---
layout: default
title: 2.5-Infrastructure-segmentation
nav_order: 5
parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.5 Infrastructure segmentation methods

Cisco ネットワークおよびセキュリティインフラストラクチャにおける **Infrastructure Segmentation Methods（インフラストラクチャセグメンテーション技術）** は、単一の物理ネットワーク基盤上に論理的・ポリシーベースの分離環境を構築し、アクセス制御、マルウェアの横移動（Lateral Movement）防止、マルチテナンシー（Multi-Tenancy）、およびコンプライアンス維持を実現する包括的アーキテクチャです。

CCIE Security v6.1 Blueprint **2.5 Infrastructure segmentation methods** では、**Private VLANs (PVLANs)**、**VRF-Lite (Virtual Routing and Forwarding)**、**GRE / mGRE (Generic Routing Encapsulation)**、**L2TPv3 (Layer 2 Tunneling Protocol Version 3)**、**VXLAN / EVPN (Virtual Extensible LAN)**、および **Cisco TrustSec / SGT (Security Group Tagging ＆ SXP: SGT Exchange Protocol)** に至る、レイヤー 2・レイヤー 3・オーバーレイ・アイデンティティ（タグベース）の各階層におけるセグメンテーション技術の実装・統合・トラブルシューティング能力が深く問われます。

---

## 📘 概要

* **機能概要**:
  * **Private VLANs (PVLANs)**:
    * L2 VLAN 内部をさらに細分化し、同一サブネット/VLAN 内のホスト間直接通信を物理/論理スイッチレベルで強制遮断する技術。
    * **Primary VLAN**: 外部やルーター（Promiscuous Port）からのトラフィックを転送。
    * **Secondary VLANs**:
      * **Isolated VLAN**: Isolated ポート間に位置し、ホスト相互の L2 通信を完全拒否（Promiscuous ポートとのみ通信可能）。
      * **Community VLAN**: 同一 Community 内のホスト間通信を許可しつつ、他の Community や Isolated ホストとの通信を遮断。
  * **VRF-Lite (Virtual Routing and Forwarding)**:
    * L3 ルーティングテーブル、インターフェイス、およびルーティングプロセス（OSPF, EIGRP, BGP）を単一の物理機器内で仮想的に分離・独立させるマルチテナンシー技術（MPLS なしの L3 セグメンテーション）。
  * **GRE ＆ mGRE Tunnels**:
    * 任意の L3 パケット（IPsec, Multicast, Dynamic Routing）を標準 IP パケット（IP Protocol 47）でカプセル化し、非隣接 L3 ネットワーク間にポイントツーポイント（GRE）またはポイントツーマルチポイント（mGRE / DMVPN）の論理トンネルを構築する技術。
  * **L2TPv3 (Layer 2 Tunneling Protocol Version 3)**:
    * L2 パケット（Ethernet, HDLC, Frame Relay 等）を IP バックボーン（IP Protocol 115 または UDP 1701）経由で透過的にカプセル化・延伸する擬似線（Pseudowire）技術。
  * **VXLAN / EVPN (Virtual Extensible LAN / Ethernet VPN)**:
    * L2 イーサネットフレームを L3 UDP パケット（UDP Port 4789）でカプセル化する IP オーバーレイ技術。24 ビットの VNI (VXLAN Network Identifier) を用いて最大 1,600 万個の論理ネットワークを定義し、Cisco SD-Access や Data Center Fabric (ACI) のセグメンテーション基盤を形成。
  * **Cisco TrustSec / Security Group Tagging (SGT)**:
    * 従来の IP アドレスや VLAN に依存しない、16 ビットの **SGT (Security Group Tag)** に基づく役割ベース（Role-Based）の動的マイクロセグメンテーション技術。
    * ユーザー認証（802.1X / MAB via Cisco ISE）時に SGT を割り当て、**Inline Tagging (CMD Header / VXLAN-GPO)** または **SXP (SGT Exchange Protocol)** 経由でネットワーク全体へ伝播。スイッチ、ASA、FTD 上の **SGACL** や **SGT-based ACP** でアクセス制御を強制実行。
* **利用目的**:
  * 同一サブネット内の不要な L2 通信（ARP スプーフィング、ランサムウェアの横移動）を最小化する。
  * 物理インフラを追加することなく、異なる部門・テナント・セキュリティゾーンの L3 経路および L2 ドメインを分離する。
  * IP アドレスの変更やサブネット設計の肥大化を伴わずに、高度なアイデンティティベース（SGT）のアクセス制御ポリシーをネットワーク全体に統一適用する。
* **どのような場面で利用するか**:
  * 企業 DMZ やデータセンターサーバーラックにおける PVLAN 適用。
  * 共有 WAN エッジおよびファイアウォールにおける VRF-Lite による管理/データプレーン分離。
  * キャンパス LAN やデータセンター（Cisco SD-Access / ACI）における SGT ＋ VXLAN 統合マイクロセグメンテーション。

---

## 🔑 要点

| セグメンテーション技術 | 動作レイヤー | カプセル化 / 識別子 | セグメンテーション範囲 | 主な用途 ＆ 特徴 |
| :--- | :--- | :--- | :--- | :--- |
| **Private VLAN (PVLAN)** | Layer 2 | 802.1Q Tag (Primary/Secondary) | 同一 L2 VLAN 内部 | 同一サブネット内の L2 横移動防止（Isolated / Community） |
| **VRF-Lite** | Layer 3 | VRF Table ID / RD / RT | L3 ネットワーク全域 | 独立した L3 ルーティングテーブル構築（マルチテナンシー） |
| **GRE / mGRE** | Layer 3 Overlay | IP Protocol 47 Header | L3 ピア間 | 異種プロトコル・マルチキャストカプセル化、DMVPN オーバーレイ |
| **L2TPv3** | Layer 2 Overlay | IP Protocol 115 / Session ID | L3 ネットワーク越しの L2 延伸 | L2 ポイントツーポイント擬似線（Pseudowire）延伸 |
| **VXLAN / EVPN** | Layer 2/3 Overlay | UDP 4789 ＋ 24-bit VNI | L3 オーバーレイファブリック | 大規模 L2 Over L3 延伸、SD-Access / ACI ファブリック |
| **Cisco TrustSec (SGT)** | Policy / Role (L2–L7) | 16-bit SGT Tag (CMD / VXLAN) | ネットワーク全体 | IP 依存のない動的役割ベースセグメンテーション (SGACL/FTD) |

---


## 🏗 動作原理

### 1. Private VLANs (PVLANs) トラフィックフロー構造

PVLAN は Primary VLAN と Secondary VLANs (Isolated / Community) のマッピングによって動作します。

```text
                                [ Default Gateway (Router/ASA) ]
                                               │
                                               │ Promiscuous Port (Primary VLAN 100)
                                               ▼
                                   [ L2 Switch (PVLAN Enabled) ]
                                    ├── Primary VLAN: 100
                                    ├── Secondary Isolated VLAN: 101
                                    └── Secondary Community VLAN: 102
                                       │                      │
                  ┌────────────────────┴────────┐            ┌┴──────────────────────────┐
                  │ Isolated Ports (VLAN 101)   │            │ Community Ports (VLAN 102)│
                  ▼                             ▼            ▼                           ▼
            [ Host A ]                    [ Host B ]    [ Host C ]                  [ Host D ]
            (Isolated)                    (Isolated)    (Community)                 (Community)

Traffic Rules:
 1. Host A (Isolated) ──► Host B (Isolated): BLOCKED (同一 Isolated 内通信不可)
 2. Host A (Isolated) ──► Router (Promiscuous): ALLOWED
 3. Host C (Community) ──► Host D (Community): ALLOWED (同一 Community 内通信許可)
 4. Host C (Community) ──► Host A (Isolated): BLOCKED (異言語 Secondary VLAN 通信不可)
 5. Router (Promiscuous) ──► ALL Hosts (A, B, C, D): ALLOWED
```

---

### 2. Cisco TrustSec SGT 伝播アーキテクチャ (Inline Tagging vs SXP)

Cisco TrustSec における SGT の割り当て・伝播・制御フロー：

```text
[ User / Device ]
       │ (1) 802.1X / MAB Authentication
       ▼
[ Cisco ISE (Policy Server) ] ─── Assigns SGT 10 (Finance) via RADIUS Access-Accept
       │
       ├─── (2a) SXP Peer Connection (TCP 64999) ─────────► [ ASA / FTD / Non-CMD Switch ]
       │    (Sends IP-to-SGT Binding: 10.1.1.50 ➔ SGT 10)         │
       │                                                         │ (Applies SGT-based ACP / Security Group Policy)
       │                                                         ▼
       └─── (2b) SGT Injected into Ingress Switch ───► [ Packet Pack with 802.1AE CMD Tag (SGT=10) ]
                                                                 │
                                                                 │ (3) Inline Tagging over L2/L3 Trunk
                                                                 ▼
                                                    [ Egress Switch / Firewall ]
                                                         │ (4) Enforces SGACL / Rule
                                                         │     SGT 10 ➔ SGT 20 (Server): PERMIT
                                                         ▼
                                                    [ Server (SGT 20) ]
```

---

## ⚙ 動作シーケンス

### 1. Private VLAN パケット処理シーケンス

1. **Ingress ポート着信 ＆ VLAN タグ付与**:
   * Isolated ポート（例: Host A）に入ったフレームは、スイッチ内部で Secondary Isolated VLAN ID (VLAN 101) が付与される。
2. **L2 転送テーブル (MAC Address Table) 評価**:
   * スイッチは MAC テーブルを検索。Secondary Isolated VLAN 101 上で許可されている宛先ポートは **Promiscuous ポートのみ**。
   * 同一 Isolated VLAN 内の別ポート（Host B）宛てのフレームは、L2 転送エンジンレベルで直ちにドロップされる。
3. **Egress Promiscuous ポート送信 ＆ Primary VLAN 変換**:
   * ルーター宛てのフレームは Promiscuous ポートへ送られ、Primary VLAN ID (VLAN 100) として送信される。
4. **ルーターによる L3 転送 ＆ 戻りパケット処理**:
   * ルーターが L3 処理して戻りパケットを Primary VLAN 100 として送信。スイッチは Primary VLAN 100 を全 Secondary VLAN (101, 102) へ転送可能なため、Host A へ正常に応答が到達する。

---

### 2. Cisco TrustSec (Inline Tagging ＆ SGACL) 処理シーケンス

1. **認証 ＆ IP-to-SGT バインディング生成**:
   * クライアントが 802.1X 認証に成功。Cisco ISE が RADIUS `cisco-av-pair = CTS-SGT=000a-10` (SGT 10: Sales) をスイッチへ返却。
   * イングレススイッチはクライアントの IP アドレス（例: `10.1.1.50`）と SGT 10 のバインディングをローカル IP-SGT テーブルに登録。
2. **インラインタグ挿入 (CMD Header Insertion)**:
   * ホストからパケットが送信されると、インギレススイッチはイーサネットヘッダーと IP ヘッダーの間に **Cisco Metadata (CMD) ヘッダー**（EtherType `0x8909`）を挿入し、SGT フィールドに `10` を書き込む。
3. **中継 ＆ トラフィック伝播**:
   * タグ付きフレームは、TrustSec サポートされたトランクポート（Inline Tagging 有効）を経由してネットワーク内を透過。
4. **エグレススイッチ / ファイアウォールでの SGACL / ACP ポリシー評価**:
   * エグレススイッチは着信パケットから SGT `10` を抽出し、宛先ホストの SGT `20` (DataCenter Servers) を確認。
   * **SGACL マトリクス** (`Role-Based Access List`) を評価: `SGT 10 ➔ SGT 20: Permit TCP 80, Deny IP`。条件に合致した場合のみパケットをアウトバウンド送信。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Private VLANs (PVLAN) の設定手順 ＆ トラップ**:
   * **VTP モード要件**: PVLAN を設定するスイッチは **`vtp mode transparent`** (または Off) に設定されている必要がある（VTPv1/v2 では client/server モードで PVLAN を伝播できない。VTPv3 のみ対応）。
   * **Secondary ➔ Primary マッピングの必須投入**: `vlan 101` (isolated), `vlan 102` (community) を作成後、`vlan 100` (primary) の設定下で **`private-vlan association 101,102`** を指定しないと機能しない。
   * **Port 割り当てコマンド**:
     * Promiscuous: `switchport mode private-vlan promiscuous` ＋ **`switchport private-vlan mapping 100 101,102`**
     * Host: `switchport mode private-vlan host` ＋ **`switchport private-vlan host-association 100 101`**
2. **VRF-Lite 設定 ＆ BGP / OSPF 統合**:
   * **RD (Route Distinguisher) の必須指定**: `ip vrf <NAME>` (旧) または `vrf definition <NAME>` (新) で **`rd <ASN:NN>`** を必ず定義する。
   * **インターフェイスバインド時の IP 消失罠**: 既存インターフェイスに `vrf forwarding <NAME>` を適用すると、**設定されていた IP アドレスが消去される**。必ず VRF バインド後に `ip address` コマンドを再投入すること。
   * **OSPF per VRF**: `router ospf <PROCESS_ID> vrf <NAME>` または `router ospfv3 <PROCESS_ID>` 内で `vrf <NAME>` をアサインする。
3. **GRE / mGRE ＆ MTU/MSS Clamping**:
   * GRE オーバーレイでは 24 バイト（IP 20B ＋ GRE 4B）、IPsec 統合時は最大 50–70 バイトのヘッダーが追加される。
   * トラネルインターフェイス上で **`ip mtu 1400`** および **`ip tcp adjust-mss 1360`** を適切に計算して投入しないと、大きなパケットが断片化・ドロップする。
4. **Cisco TrustSec / SXP (SGT Exchange Protocol) の設定**:
   * **SXP Peer パスワード ＆ 役割**: SXP スピーカー (Speaker: バインディング送信側) と リスナー (Listener: 受信側) の関係を定義。`cts sxp peer <IP> password <KEY> mode local|peer <speaker|listener>`。
   * **SXP 有効化**: **`cts sxp enable`** をグローバルで投入することを忘れない。
   * **ASA / FTD での SGT 制御**: ASA では `object-group security-group` を用いて ACL で参照。FTD では FMC の Security Intelligence または Access Control Policy で SGT オブジェクトを条件として指定。

---


## 🛠 設定方法

### 1. Catalyst Switch: Private VLAN (PVLAN) 完全構成例

```bash
! ===== Step 1: VTP Mode を Transparent に変更 =====
vtp mode transparent

! ===== Step 2: Secondary VLANs の定義 =====
vlan 101
 private-vlan isolated
 name PVLAN_ISO_101

vlan 102
 private-vlan community
 name PVLAN_COM_102

! ===== Step 3: Primary VLAN の定義 ＆ Association =====
vlan 100
 private-vlan primary
 private-vlan association 101,102
 name PVLAN_PRI_100

! ===== Step 4: Promiscuous Port (ルーター/ゲートウェイ接続ポート) 設定 =====
interface GigabitEthernet1/0/1
 description Connected to Default Gateway Router
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 100 101,102

! ===== Step 5: Isolated Host Port 設定 =====
interface GigabitEthernet1/0/2
 description Connected to Server-A (Isolated)
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101

! ===== Step 6: Community Host Port 設定 =====
interface GigabitEthernet1/0/3
 description Connected to Server-B (Community)
 switchport mode private-vlan host
 switchport private-vlan host-association 100 102
```

---

### 2. Cisco IOS-XE: VRF-Lite ＆ BGP / OSPF 多重化構成

```bash
! ===== Step 1: VRF Definition の作成 (RD ＆ RT) =====
vrf definition RED
 rd 65000:10
 address-family ipv4
  route-target export 65000:10
  route-target import 65000:10
 exit-address-family

vrf definition BLUE
 rd 65000:20
 address-family ipv4
  route-target export 65000:20
  route-target import 65000:20
 exit-address-family

! ===== Step 2: インターフェイスへの VRF アサイン =====
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 vrf forwarding RED
 ip address 10.10.10.1 255.255.255.0

interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 vrf forwarding BLUE
 ip address 10.20.20.1 255.255.255.0

! ===== Step 3: OSPF per VRF 設定 =====
router ospf 10 vrf RED
 router-id 1.1.1.1
 network 10.10.10.0 0.0.0.255 area 0

! ===== Step 4: MP-BGP per VRF 設定 =====
router bgp 65000
 bgp router-id 1.1.1.1
 address-family ipv4 vrf BLUE
  redistribute connected
  neighbor 10.20.20.2 remote-as 65001
 exit-address-family
```

---

### 3. Cisco IOS-XE: TrustSec SXP (SGT Exchange Protocol) ＆ SGT 設定

```bash
! ===== Step 1: TrustSec 有効化 ＆ SXP パラメータ設定 =====
cts sxp enable
cts sxp default source-interface Loopback0
cts sxp key cisco123

! ===== Step 2: SXP Peer 定義 (IOS = Speaker, ASA/ISE = Listener) =====
cts sxp peer 192.168.100.50 password cisco123 mode local speaker

! ===== Step 3: 静的 IP-to-SGT バインディング作成 (代替設定) =====
cts role-based ip-to-gt mapping 10.1.1.100 sgt 15

! ===== Step 4: SGACL (Role-Based ACL) の定義 ＆ 適用 =====
ip access-list role-based PERMIT_WEB_ONLY
 permit tcp destination port eq 80
 permit tcp destination port eq 443
 deny ip

cts role-based permissions src-sgt 10 dst-sgt 20 PERMIT_WEB_ONLY
```

---

### 4. Cisco ASA: TrustSec SXP ＆ Security Group ベース Access Control

```bash
! ===== ASA TrustSec ＆ SXP 設定 =====
cts server-group ISE_TACACS
cts sxp enable
cts sxp default source-interface management
cts sxp connection peer 192.168.100.1 password cisco123 mode local listener

! ===== Security Group オブジェクトの作成 =====
object-group security-group OG_SGT_FINANCE
 security-group-tag 10

object-group security-group OG_SGT_SERVERS
 security-group-tag 20

! ===== SGT ベース Access List ＆ Access Group =====
access-list ACL_SGT_CONTROL extended permit ip object-group-security-group OG_SGT_FINANCE object-group-security-group OG_SGT_SERVERS
access-group ACL_SGT_CONTROL in interface inside
```

---

## 🔍 検証コマンド

| 目的 | コマンド (IOS / ASA / Switch) | 表示確認ポイント |
| :--- | :--- | :--- |
| **PVLAN 設定 ＆ マッピング確認** | <code>show interface private-vlan mapping</code><br><code>show vlan private-vlan</code> | Primary と Secondary (Isolated/Community) のペアリング、ポート割り当て |
| **VRF インターフェイス ＆ ルーティングテーブル確認** | <code>show ip vrf</code><br><code>show ip route vrf <NAME></code> | アサインされた RD、インターフェイス一覧、VRF ごとの独立ルート |
| **L2TPv3 トンネル ＆ Pseudowire セッション確認** | <code>show l2tun session all</code><br><code>show xconnect interface <IF></code> | Pseudowire 状態 (UP)、Local/Remote Session ID、パケットカウンター |
| **VXLAN NVT / VNI / ピア確認** | <code>show nve interface nve1</code><br><code>show nve peers</code> | VNI バインド状態、Encap Type (VXLAN)、VTEP ピア IP ＆ Status |
| **TrustSec SXP ピア接続確認** | <code>show cts sxp connections</code> | Peer IP、Mode (Speaker/Listener)、SXP Version、Status (**`On`**) |
| **IP-to-SGT バインディングテーブル確認** | <code>show cts role-based device-tracking all</code><br><code>show cts ip-sgt-mapping</code> | IP アドレス、SGT タグ値、Source (SXP / Local / RADIUS) |
| **SGACL 適用 ＆ ドロップカウンター確認** | <code>show cts role-based permissions</code><br><code>show cts role-based counters</code> | Src SGT ➔ Dst SGT のパーミッションマトリクス、Hardware Hit Multipliers |
| **SXP デバッグログ追跡** | <code>debug cts sxp messages</code><br><code>debug cts sxp connection</code> | SXP パケット（Open, Update, Keepalive）の送受信ログ、バインディングパブリッシュ |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **Isolated ポート間の通信が遮断されず、Ping が通ってしまう** | スイッチの VTP モードが Client/Server のまま、あるいは Primary VLAN と Secondary VLAN の Association が未設定 | <code>show vlan private-vlan</code><br><code>show vtp status</code> | `vtp mode transparent` に変更し、`vlan <PRI>` 下で `private-vlan association <SEC>` を正しく投入する。 |
| **インターフェイスへ `vrf forwarding` を適用したら IP や通信が消えた** | `vrf forwarding` 設定時に既存の `ip address` が自動削除された | <code>show ip interface brief</code> | VRF アサイン直後に `ip address <IP> <MASK>` を再設定する。 |
| **TrustSec SXP Connection ステータスが `Pending-On` から `On` に移行しない** | スピーカー/リスナーの Mode 不一致 (Speaker–Speaker 等)、SXP 共有キーの不一致、または TCP 64999 ポートが FW でドロップ | <code>show cts sxp connections</code> | 両端の SXP Mode (一方が Speaker, 他方が Listener)、SXP 共有キー、およびファイアウォールでの **TCP 64999** 通信許可を確認する。 |
| **SGT タグは伝播しているが SGACL による制御が効かない** | エグレススイッチ上で Role-Based Enforcement が未有効化 (`cts role-based enforcement`) | <code>show cts role-based permissions</code> | 対象インターフェイスおよびグローバルで `cts role-based enforcement` を有効化する。 |
| **GRE / VXLAN トンネル経由の大きな TCP パケットがドロップ・遅延する** | ヘッダー追加による PMTUD 失敗および断片化ドロップ | <code>show interface tunnel</code> | トンネルインターフェイスに `ip mtu 1400` および `ip tcp adjust-mss 1360` を適用する。 |

---

## ⚠ 制限事項

* **Private VLANs (PVLAN)**:
  * PVLAN ポートは EtherChannel メンバーポートとして直接アサインできません（EtherChannel 全体として設定することは可能）。また、SPAN の Destination ポートに PVLAN ポートを指定することはサポートされません。
* **Cisco TrustSec / SGT**:
  * ASA / FTD では SGT を含んだインライン CMD タグの直接デコード（Hardware Tagging）にモデル制限があります。非対応アプライアンスでは **SXP 経由での IP-to-SGT バインディング取得** が必須となります。
* **L2TPv3 / VXLAN**:
  * L2TPv3 は標準で暗号化機能を持たないため、暗号化が必要な場合は IPsec との併用が必須となります。

---

## 🔄 他技術との関連

* **Cisco ISE (Identity Services Engine)**: Centralized Policy Server として 802.1X/MAB 認証時に SGT を動的割り当てし、SXP バインディングをネットワーク全体へ配信。
* **Cisco SD-Access (SDA) / ACI**: 制御プレーンに LISP / BGP-EVPN、データプレーンに VXLAN (GPO Header: SGT 埋め込み) を使用した次世代統合ファブリック。
* **Cisco ASA / FTD Access Control**: FTD Access Control Policy 内で SGT オブジェクトを識別条件として指定し、アイデンティティベースのファイアウォールルールを実行。

---

## 🧩 比較表

### 1. インフラストラクチャセグメンテーション技術比較

| 比較項目 | Private VLAN (PVLAN) | VRF-Lite | VXLAN / EVPN | Cisco TrustSec (SGT) |
| :--- | :--- | :--- | :--- | :--- |
| **主要レイヤー** | Layer 2 | Layer 3 | L2/L3 Overlay | **Policy / Identity (L2–L7)** |
| **識別子** | Secondary VLAN ID | Route Distinguisher (RD) | 24-bit VNI | **16-bit SGT Tag** |
| **IP 依存度** | IP 不依存 (L2 マックテーブル) | IP サブネット依存 | IP アンダーレイ依存 | **IP 依存なし (役割ベース)** |
| **スケーラビリティ**| 小〜中 (単一スイッチ/L2 域) | 中 (ルーティング分離) | 極めて大 (1,600 万 VNI) | **極めて大 (65,535 SGT)** |
| **制御ポイント** | スイッチ L2 転送エンジン | L3 ルーティングテーブル | VTEP / EVPN ルート | **SGACL / Firewall ACP** |

### 2. TrustSec SGT 伝播方式: Inline Tagging (CMD) vs SXP (SGT Exchange Protocol)

| 比較項目 | Inline Tagging (802.1AE CMD / VXLAN) | SXP (SGT Exchange Protocol) |
| :--- | :--- | :--- |
| **データプレーンカプセル化**| **パケットヘッダー内に SGT を直接挿入** (CMD / VXLAN-GPO) | **コントロールプレーンで IP-to-SGT マッピングを転送** (TCP 64999) |
| **ハードウェア要件** | 全中継スイッチで TrustSec ASIC (Hardware Tagging) 必須 | **標準 L3/L2 スイッチ・ファイアウォールで動作可能** |
| **推奨適用領域** | TrustSec 対応キャンパススイッチ間、SD-Access | 非 TrustSec スイッチ混在環境、ASA / FTD / WAN |

---

## 💡 ベストプラクティス

1. **IP 依存から分離した SGT ベースマイクロセグメンテーションの導入**:
   * アドレス設計（Subnet / IP）変更に影響されないアクセス制御を実現するため、Cisco ISE と統合した SGT / TrustSec アーキテクチャを標準採用し、エッジファイアウォール（FTD/ASA）での SGT ベース ACP 制御を構築します。
2. **SXP ピアの冗長化 ＆ ソースインターフェイス固定**:
   * SXP ピアリング（TCP 64999）を設定する際は、必ず Stable な Loopback インターフェイス（`cts sxp default source-interface Loopback0`）を指定し、複数の ISE PSN / SXP ピアと接続して冗長化を図ります。

---


## 📝 ラボ学習・設定サンプル例

### 例1: Catalyst スイッチにおける Private VLAN (PVLAN) 基本設定

#### 問題
スイッチ SW1 において、Primary VLAN `100`、Secondary Isolated VLAN `101`、Secondary Community VLAN `102` を作成し、`Gi1/0/1` を Promiscuous ポート、`Gi1/0/2` を Isolated ポート、`Gi1/0/3` を Community ポートとして構成せよ。

#### 要件
* VTP モードを `transparent` に変更すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
vtp mode transparent

vlan 101
 private-vlan isolated
vlan 102
 private-vlan community
vlan 100
 private-vlan primary
 private-vlan association 101,102

interface GigabitEthernet1/0/1
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 100 101,102

interface GigabitEthernet1/0/2
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101

interface GigabitEthernet1/0/3
 switchport mode private-vlan host
 switchport private-vlan host-association 100 102
```

#### 検証方法
* `show vlan private-vlan` を実行し、VLAN 100 に 101, 102 がマッピングされ、対象ポートが正しくアサインされていることを確認。

---

### 例2: Cisco IOS-XE における VRF-Lite ＆ 独立 OSPF プロセスの設定

#### 問題
ルーター R1 において、VRF `SEC_A` (RD `100:1`) および VRF `SEC_B` (RD `100:2`) を作成し、それぞれ `Gi0/1.10` および `Gi0/1.20` へバインドの上、独立した OSPF プロセス (Process 10 ＆ 20) を設定せよ。

#### 要件
* `vrf definition` を使用すること。

#### 設定例

```bash
! ===== R1 Configuration =====
vrf definition SEC_A
 rd 100:1
 address-family ipv4
 exit-address-family

vrf definition SEC_B
 rd 100:2
 address-family ipv4
 exit-address-family

interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 vrf forwarding SEC_A
 ip address 10.1.10.1 255.255.255.0

interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 vrf forwarding SEC_B
 ip address 10.1.20.1 255.255.255.0

router ospf 10 vrf SEC_A
 router-id 1.1.1.1
 network 10.1.10.0 0.0.0.255 area 0

router ospf 20 vrf SEC_B
 router-id 2.2.2.2
 network 10.1.20.0 0.0.0.255 area 0
```

#### 検証方法
* `show ip route vrf SEC_A` および `show ip route vrf SEC_B` を実行し、ルーティングテーブルが完全に分離されていることを確認。

---

### 例3: GRE トンネルでの MTU ＆ TCP MSS Clamping 設定

#### 問題
ルーター R1 と R2 間の GRE トンネル `Tunnel0` において、オーバーヘッド考慮によるフラグメントを防ぐため、IP MTU を `1400`、TCP MSS を `1360` に設定せよ。

#### 要件
* `ip mtu` および `ip tcp adjust-mss` コマンドを適用すること。

#### 設定例

```bash
! ===== R1 Configuration =====
interface Tunnel0
 ip address 172.16.1.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 192.168.1.2
 ip mtu 1400
 ip tcp adjust-mss 1360
```

#### 検証方法
* `show interface Tunnel0` を実行し、`MTU 1400 bytes` が反映され、大型 SYN パケットの MSS が 1360 に書き換えられることを確認。

---

### 例4: L2TPv3 による IP バックアップ経由の L2 擬似線 (Pseudowire) 延伸

#### 問題
R1 と R2 間で L2TPv3 トンネルを設定し、`Gi0/2` ポート（L2 VLAN 50）を L3 IP バックボーン越しに擬似線延伸せよ。

#### 要件
* `pseudowire-class` を使用すること。

#### 設定例

```bash
! ===== R1 Configuration =====
pseudowire-class L2TPV3_PW
 encapsulation l2tpv3
 ip local interface Loopback0

interface GigabitEthernet0/2.50
 encapsulation dot1Q 50
 xconnect 192.168.2.2 100 pw-class L2TPV3_PW
```

#### 検証方法
* `show l2tun session all` および `show xconnect interface GigabitEthernet0/2.50` を実行し、Session State が `UP` であることを確認。

---

### 例5: Cisco IOS-XE での TrustSec SXP Speaker 構築

#### 問題
ルーター R1 において SXP を有効化し、ソースを `Loopback0`、共有キー `cisco123` を用いて ISE (`192.168.100.50`) に対して SXP Speaker モードで接続せよ。

#### 要件
* `cts sxp` コマンドを使用すること。

#### 設定例

```bash
! ===== R1 Configuration =====
cts sxp enable
cts sxp default source-interface Loopback0
cts sxp key cisco123
cts sxp peer 192.168.100.50 password cisco123 mode local speaker
```

#### 検証方法
* `show cts sxp connections` を実行し、Status が `On` となっていることを確認。

---

### 例6: ASA での SXP Listener ＆ SGT ベース ACL 設定

#### 問題
ASA1 において SXP Listener モードを構成し、ISE (`192.168.100.50`) から取得した SGT タグ `10` (Finance) から SGT タグ `20` (Servers) 宛ての通信を許可する ACL を適用せよ。

#### 要件
* `object-group security-group` を参照すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
cts sxp enable
cts sxp default source-interface management
cts sxp connection peer 192.168.100.50 password cisco123 mode local listener

object-group security-group OG_FINANCE
 security-group-tag 10

object-group security-group OG_SERVERS
 security-group-tag 20

access-list ACL_INSIDE extended permit ip object-group-security-group OG_FINANCE object-group-security-group OG_SERVERS
access-group ACL_INSIDE in interface inside
```

#### 検証方法
* `show cts sxp connections` および `show access-list ACL_INSIDE` を実行し、SGT バインディングが反映されヒットカウントが増加することを確認。

---

### 例7: Cisco IOS-XE 静的 IP-to-SGT マッピング ＆ SGACL の定義

#### 問題
スイッチ SW1 において、特定サーバー IP `10.50.1.100` に対し静的に SGT `50` を定義し、SGT `10` から SGT `50` 宛ての HTTP トラフィックのみを許可する SGACL を設定せよ。

#### 要件
* `cts role-based ip-to-sgt` および `cts role-based permissions` を使用すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
cts role-based ip-to-sgt mapping 10.50.1.100 sgt 50

ip access-list role-based SGACL_HTTP_ONLY
 permit tcp destination port eq 80
 deny ip

cts role-based permissions src-sgt 10 dst-sgt 50 SGACL_HTTP_ONLY
cts role-based enforcement
```

#### 検証方法
* `show cts role-based device-tracking all` および `show cts role-based permissions` を実行し、SGACL マトリクスにルールが反映されていることを確認。

---

### 例8: FTD FMC における SGT オブジェクトベース Access Control Policy

#### 問題
FMC 管理下の FTD において、SGT タグ `10` (Sales) から SGT タグ `30` (DB_Servers) 宛ての SQL トラフィック (TCP 1433) を許可する ACP ルールを作成せよ。

#### 要件
* Security Group Object を使用すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Objects > Object Management > Security Group Objects ➔ "Add Security Group"
   - Name: SGT_Sales (Tag: 10)
   - Name: SGT_DB_Servers (Tag: 30)
2. Policies > Access Control > ACP 編集 ➔ "Add Rule"
   - Name: ALLOW_SALES_TO_DB
   - Action: Allow
   - Users / Identity ➔ Source Security Group: "SGT_Sales", Destination Security Group: "SGT_DB_Servers"
   - Port: MS SQL (TCP 1433)
3. Save して Deploy。
```

#### 検証方法
* FTD Diagnostic CLI で `show access-list` を実行し、内部 ACL に SGT タグ条件が追加されていることを確認。

---

### 例9: VRF 属性を保持した mGRE / DMVPN トンネルの設定

#### 問題
ルーター R1 において、アンダーレイ WAN を VRF `OUTSIDE_WAN` に分離しつつ、オーバーレイ mGRE トンネルをグローバルルーティングテーブルで動かす Front-Door VRF (FVRF) を構成せよ。

#### 要件
* `tunnel vrf OUTSIDE_WAN` コマンドを使用すること。

#### 設定例

```bash
! ===== R1 Configuration =====
vrf definition OUTSIDE_WAN
 rd 65000:999
 address-family ipv4
 exit-address-family

interface GigabitEthernet0/0
 vrf forwarding OUTSIDE_WAN
 ip address 198.51.100.1 255.255.255.0

interface Tunnel0
 ip address 10.255.255.1 255.255.255.0
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel vrf OUTSIDE_WAN
```

#### 検証方法
* `show ip route vrf OUTSIDE_WAN` で物理インターフェイスが隔離され、`show ip route` でトンネルオーバーレイ経路が確立していることを確認。

---

### 例10: VXLAN NVE インターフェイス ＆ VNI の基本設定 (IOS-XE)

#### 問題
スイッチ SW1 において、VXLAN 仮想インターフェイス `nve1` を作成し、VNI `10010` を L2VLAN 10 へバインドの上、VTEP ソースを `Loopback0` に指定せよ。

#### 要件
* `interface nve1` を使用すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
vlan 10
 nv overlay evpn

interface nve1
 no shutdown
 source-interface Loopback0
 member vni 10010 mcast-group 239.1.1.10
```

#### 検証方法
* `show nve interface nve1` を実行し、NVE1 が `Up` 状態であり、VNI 10010 が正しくバインドされていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: スイッチ SW1 上で以下の Private VLAN コンフィグが適用されている。

```bash
vlan 101
 private-vlan isolated
vlan 100
 private-vlan primary
 private-vlan association 101

interface Gi1/0/2
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101
```

ポート `Gi1/0/2` に接続された Host-A が、同じ VLAN 101 に割り当てられた `Gi1/0/3` の Host-B 宛てに ICMP Ping を送信した際の挙動として正しいものはどれか。

* A) Host-A と Host-B は同一 Secondary VLAN 内にあるため Ping は正常に通る。
* B) VLAN 101 は Isolated VLAN であり、同一 Isolated VLAN 内のホスト間通信は L2 エンジンレベルでドロップされるため Ping は失敗する。
* C) Primary VLAN 100 経由でルーターへ送られ、ルーターがリダイレクトして通信が成功する。
* D) VTP モードが Server になっていないためスイッチがリロードされる。

**正解**: **B**  
**解説**: **Isolated VLAN** 内のホストポート（Host Ports）間通信は、L2 スイッチング処理の最初で完全に遮断されます。Isolated ポートは **Promiscuous ポートとのみ通信が可能** です。

---

### 2. トラブルシュート問題

**問題**: 管理者が IOS ルーター R1 と Cisco ISE 間で SXP Connection を設定した。`show cts sxp connections` を確認したところ、ステータスが **`Pending-On`** のまま変化せず `On` にならない。対向の SXP Key は一致している。主な原因として最も適切なものはどれか。

* A) SXP は UDP ポート 500 を使用するが、途中の FW でドロップされている。
* B) R1 と ISE の間で SXP モードがともに `Speaker`（またはともに `Listener`）に設定されており、ミスマッチが発生している。
* C) SGT タグが 65535 を超えているため。
* D) 802.1X 認証が未有効のため。

**正解**: **B**  
**解説**: SXP ピアリング（TCP Port 64999）を確立するには、一方が **Speaker** (マッピング送信側)、他方が **Listener** (受信側) として正しくペアリングされている必要があります。双方のモードが同一の場合、TCP 接続は確立せず `Pending-On` で停止します。

---

### 3. Design 問題

**問題**: 企業ネットワークにおいて、既存の IP アドレス設計（サブネット構造）を変更することなく、役職（営業部、財務部、役員）に応じた動的なマイクロセグメンテーションアクセス制御をネットワーク全域で実現したい。最も推奨される技術構成はどれか。

* A) 全スイッチポートを Isolated PVLAN に変更する。
* B) 各役職ごとに新規 VLAN と VRF-Lite を作成して再IP化する。
* C) **Cisco ISE ＋ Cisco TrustSec (SGT)** を導入し、16 ビットの SGT タグを用いたロールベース SGACL / FTD ACP 制御を行う。
* D) L2TPv3 トンネルを全端末間にフルメッシュ構築する。

**正解**: **C**  
**解説**: **Cisco TrustSec (SGT)** は、IP アドレスやサブネット設計から完全に独立したアイデンティティベースのセグメンテーションソリューションです。IP アドレスの再設計なしに高度な動的アクセス制御を実現します。

---

### 4. 実装問題

**問題**: Cisco IOS-XE ルーターにおいて、既存の物理インターフェイス `Gi0/1` に `vrf forwarding RED` を適用した直後に発生する注意すべき挙動はどれか。

* A) ルーターが自動的に再起動する。
* B) インターフェイスに設定されていた IP アドレスおよび IPv6 アドレスが自動的に削除される。
* C) OSPF プロセスが自動生成される。
* D) MAC アドレスが変更される。

**正解**: **B**  
**解説**: IOS/IOS-XE では、インターフェイスへ `vrf forwarding` (または `ip vrf forwarding`) をアサインした瞬間、**設定済みの IP アドレスが削除されます**。そのため、VRF アサイン後に IP アドレスを再設定する必要があります。

---

### 5. コンフィグ読解問題

**問題**: 以下の SGT アクセス制御コンフィグに関する説明として正しいものはどれか。

```bash
cts role-based permissions src-sgt 10 dst-sgt 20 SGACL_RESTRICT
```

* A) SGT 20 から SGT 10 宛ての通信のみを制限する。
* B) 送信元 SGT `10`、送信先 SGT `20` のトラフィックに対し、アクセスマスク `SGACL_RESTRICT` で定義されたロールベースアクセス制御を適用する。
* C) SGT タグを 10 から 20 へ書き換える。
* D) SXP プロトコル経由で SGT 10 を暗号化送信する。

**正解**: **B**  
**解説**: **`cts role-based permissions`** コマンドは、指定された送信元 SGT (`src-sgt`) と送信先 SGT (`dst-sgt`) のマトリクス交差点に対して SGACL (`role-based ACL`) をアサイン・実行するコマンドです。

---

## 🔗 参考リソース

* [Cisco TrustSec Configuration Guide: Cisco TrustSec Switch Configuration](https://www.cisco.com/)
* [Cisco Catalyst 9300 Switch Configuration Guide: Configuring Private VLANs](https://www.cisco.com/)
* [Cisco IOS-XE VRF Configuration Guide: VRF-Lite Overview](https://www.cisco.com/)
* [Cisco Technical Notes: Configure SXP on Cisco IOS/IOS-XE and ASA](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Cisco TrustSec Architecture and Deployment (BRKSEC-2030)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で Infrastructure Segmentation が出題された場合の迅速な構成判断フローチャート：
    * **「Prevent L2 communication within the same subnet/VLAN」** ➔ Switch ➔ Private VLAN (PVLAN) (`isolated` / `community` / `promiscuous`)。
    * **「Separate L3 routing tables on a single router/firewall without MPLS」** ➔ VRF-Lite (`vrf definition` ＋ `vrf forwarding` ＋ `ip address` 再設定)。
    * **「Identity/Role-based access control independent of IP addresses」** ➔ Cisco TrustSec (SGT) ＋ SXP (`cts sxp enable` ＋ `cts role-based permissions`)。
    * **「Extend L2 domain over an IP backbone」** ➔ L2TPv3 (`pseudowire-class` ＋ `xconnect`) または VXLAN/EVPN。
* **図解（SGT 伝播 ＆ エンフォースメントマトリクス）**:
  ```text
  [ Ingress (Authentication) ] ──► Assign SGT 10 (Finance)
                                       │
                         SXP (TCP 64999) / CMD Tag
                                       │
                                       ▼
  [ Egress Firewall / Switch ] ──► Checks Matrix: Src SGT 10 ➔ Dst SGT 20 (Server)
                                       │
                                       ├── Match SGACL: Permit TCP 80/443, Deny IP
                                       └── Action: Forward / Drop
  ```
* **注意点**:
  * ラボ試験で PVLAN、VRF-Lite、SGT を設定した後は、`show vlan private-vlan`、`show ip route vrf <NAME>`、`show cts sxp connections`、`show cts role-based permissions` を確実に実行し、状態が正常であることを確認してください。



