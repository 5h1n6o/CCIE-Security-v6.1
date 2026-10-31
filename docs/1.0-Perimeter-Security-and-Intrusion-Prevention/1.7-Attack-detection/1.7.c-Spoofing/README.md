---
layout: default
title: 1.7.c-Spoofing
nav_order: 3
parent: 1.7-Attack-detection
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.7.c Spoofing

Cisco ネットワークインフラストラクチャにおける **Spoofing（成りすまし・アイデンティティ偽装）対策** は、レイヤー 2 (Data Link Layer) および レイヤー 3 (Network Layer) において、攻撃者が IP アドレス、MAC アドレス、ARP 応答、DHCP メッセージ、ならびに IPv6 NDP/RA/DHCPv6 などの制御メッセージの送信元情報を偽装して行う攻撃（Man-in-the-Middle、セッションハイジャック、DoS/Poisoning、IP 盗用など）を検知・緩和するための包括的なセキュリティアーキテクチャです。

CCIE Security v6.1 Blueprint **1.7.c Spoofing** では、スイッチャブル LAN 環境における L2 攻撃対策（DHCP Snooping、Dynamic ARP Inspection [DAI]、IP Source Guard [IPSG]、Port Security）、L3 パケットバウンダリにおける IP 反転経路検証（Unicast RPF / uRPF: Strict Mode ＆ Loose Mode、ASA/FTD Anti-Spoofing）、および IPv6 特有のファーストホップセキュリティ（IPv6 FHS: RA Guard、DHCPv6 Guard、IPv6 Source Guard、IPv6 Destination Guard）の設計・設定・検証・トラブルシューティング能力が多角的に問われます。

---

## 📘 概要

* **機能概要**:
  * **L2 Spoofing 対策 (IPv4)**:
    * **DHCP Snooping**: 信頼できないポート（Untrusted Ports）からの不正な DHCP サーバー応答 (Rogue DHCP) を遮断し、正常に割り当てられた IP/MAC/Port/VLAN バインディングデータベース（DHCP Snooping Binding Table）を動的に生成・保持する。
    * **Dynamic ARP Inspection (DAI)**: DHCP Snooping バインド DB または静的 ARP ACL を参照し、不正な ARP 応答（ARP Cache Poisoning / MitM 攻撃）を検知・ドロップする。
    * **IP Source Guard (IPSG)**: アクセスポートにおいて、DHCP Snooping バインド DB または静的 IP Source Binding と一致しない送信元 IP/MAC アドレスを持つデータパケットを L2 ハードウェアレベルで直接フィルタリングする。
    * **Port Security**: ポート上の送出 MAC アドレス数を制限し、CAM テーブル枯渇攻撃や MAC アドレス偽装を防止する。
  * **L3 Spoofing 対策**:
    * **Unicast RPF (uRPF)**: ルーターおよびファイアウォール（ASA/FTD）において、パケットの送信元 IP アドレスを CEF (Forwarding Information Base: FIB) またはルーティングテーブルと照合し、経路の反転ルックアップ（Reverse Lookup）を行うことで、スプーフィングされた IP パケットを境界で水際破棄する。
  * **IPv6 FHS (First-Hop Security) Spoofing 対策**:
    * **IPv6 RA Guard**: 不正なルーター広告 (Rogue RA) パケットを L2 スイッチポート上で検査・ブロックし、偽のデフォルトゲートウェイ設定や不正プレフィックス割り当てによる DoS/MitM を阻止する。
    * **IPv6 DHCPv6 Guard**: 未承認の DHCPv6 サーバーからの Reply/Advertise パケットをドロップする。
    * **IPv6 Source Guard / SISF**: IPv6 Neighbor Binding Table（NDP インスペクション / DHCPv6 ガードで作成）を元に、偽装された送信元 IPv6 アドレスを持つデータパケットをポートレベルで破棄する。
    * **IPv6 Destination Guard**: 未知の宛先 IPv6 アドレスに対する大量スキャン (Neighbor Cache Starvation) 攻撃からルーターの Neighbor Cache を保護する。
* **利用目的**:
  * 攻撃者がネットワーク内で正常な端末、ルーター、または各種インフラサーバーになりすますことで発生するデータ盗聴、通信の改ざん、およびサービス不能状態を根本から排除する。
* **どのような場面で利用するか**:
  * 企業 LAN のアクセス層スイッチ（ユーザー端末が接続する全 Access Ports）。
  * 外部インターネットや他社接続境界（WAN Edge / DMZ / Service Provider Peering）。
  * IPv6 アドレスが自動割り当て（SLAAC / Stateful DHCPv6）されるデュアルスタック LAN 環境。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **評価コンポーネント (L2)** | **DHCP Snooping**、**Dynamic ARP Inspection (DAI)**、**IP Source Guard (IPSG)**、**Port Security** |
| **評価コンポーネント (L3)** | **Unicast RPF (uRPF)** (Strict Mode: `reachable-via rx`, Loose Mode: `reachable-via any`)、**ASA/FTD Anti-Spoofing** (`ip verify reverse-path`) |
| **評価コンポーネント (IPv6 FHS)** | **RA Guard**、**DHCPv6 Guard**、**IPv6 Source Guard**、**IPv6 Destination Guard**、**SISF (Secure Infrastructure Security Framework)** |
| **基盤データベース** | **DHCP Snooping Binding Table**（IPv4）、**Static IP/ARP/MAC Bindings**、**CEF FIB Table**（L3）、**IPv6 Neighbor Binding Table**（IPv6 SISF） |
| **前提要件** | DAI および IPSG は原則として **DHCP Snooping** の有効化に依存する（静的環境では Static Binding / ARP ACL が必須） |
| **Option 82 注意点** | スイッチで DHCP Snooping を有効化するとデフォルトで Option 82 が挿入されるため、IOS DHCP サーバー側またはスイッチ側で不整合の回避処理が必要 |
| **非対称ルーティング** | uRPF Strict Mode は非対称ルーティング環境で正常パケットを誤ドロップするため、**Loose Mode** を使用する |
| **ライセンス要件** | スイッチおよびルーターの基本的 Spoofing 対策機能は Base / LAN Base / IP Base ライセンスで利用可能 |

---

## 🏗 動作原理

### 1. L2 IPv4 Anti-Spoofing 統合アーキテクチャ (DHCP Snooping + DAI + IPSG)

アクセススイッチにおいて、DHCP Snooping が核（信頼できる情報源）となり、動的に作成されたバインド情報を DAI および IPSG が参照してパケットをハードウェアレベルで検証します。

```text
[ Client / Potential Attacker ] ──( Ingress Traffic )──► [ Access Port (Untrusted) ]
                                                                 │
┌────────────────────────────────────────────────────────────────┴──────────────────┐
│ Switch Security Inspection Pipeline                                               │
│                                                                                   │
│ 1. DHCP Control Traffic Processing                                                │
│    └── DHCP Snooping: Inspects DHCP Replies ──► Block Rogue DHCP Offers            │
│         └── If Valid Lease: Populate DHCP Snooping Binding Table                  │
│              (MAC: 0011.2233.4455, IP: 10.1.1.50, VLAN: 10, Port: Gi1/0/1)        │
│                                                                                   │
│ 2. ARP Traffic Processing (Dynamic ARP Inspection - DAI)                          │
│    └── Check ARP Packet (Sender IP & MAC) against DHCP Snooping Binding Table     │
│         ├── Match? ──► Forward ARP Packet                                         │
│         └── Mismatch / Spoofed ARP? ──► DROP ARP Packet & Log Event               │
│                                                                                   │
│ 3. IP Data Traffic Processing (IP Source Guard - IPSG)                            │
│    └── Apply PACL based on Binding Table (IP + MAC + Port)                        │
│         ├── Src IP 10.1.1.50 & MAC 0011.2233.4455? ──► Forward Packet             │
│         └── Spoofed Source IP / MAC? ──► DROP Data Packet at Hardware ASIC        │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

### 2. L3 Unicast RPF (uRPF) 反転経路検証原理

ルーターおよびファイアウォール（ASA/FTD）において、パケットの「宛先 IP」ではなく「送信元 IP」を CEF FIB テーブルと照合します。

```text
[ Ingress Packet ] (Src IP: 203.0.113.10, Dst IP: 10.1.1.100) on Interface GigabitEthernet0/0
       │
       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Unicast Reverse Path Forwarding (uRPF) Inspection                                │
│                                                                                  │
│  [ CEF Forwarding Information Base (FIB) Lookup for Source IP 203.0.113.10 ]     │
│                                                                                  │
│  Mode 1: Strict Mode (`ip verify unicast source reachable-via rx`)               │
│     ├── Is 203.0.113.10 in FIB? AND Is Best Egress Interface = Gi0/0?             │
│     │      ├── YES ──► Pass Packet                                               │
│     │      └── NO  ──► DROP Packet (uRPF Verification Drop)                      │
│                                                                                  │
│  Mode 2: Loose Mode (`ip verify unicast source reachable-via any`)               │
│     ├── Is 203.0.113.10 in FIB? (Interface match NOT required, Null0 excluded)   │
│     │      ├── YES ──► Pass Packet                                               │
│     │      └── NO  ──► DROP Packet (Used in BGP RTBH & Asymmetric Routing)       │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

1. **DHCP アドレス取得とバインド登録フェーズ (IPv4 L2)**:
   * 端末が `DHCPDISCOVER` を送信し、信頼されたポート（`ip dhcp snooping trust`）に接続された正規 DHCP サーバーから `DHCPACK` を受信。
   * スイッチは `DHCPACK` をスヌーピングし、`MAC アドレス`、`IP アドレス`、`VLAN`、`Port`、`Lease Time` を **DHCP Snooping Binding Table** に即座に動的登録する。
2. **ARP パケット着信と DAI 検証フェーズ**:
   * Untrusted ポートに ARP パケット（Request / Reply）が着信した際、DAI エンジンがパケットをインターセプト。
   * パケット内部の Sender IP / Sender MAC を DHCP Snooping DB（または `arp access-list`）と照合。
   * 情報が一致しない偽造 ARP（例: 他人の IP を自分の MAC として宣言）を発見した場合、パケットをドロップして Syslog アラートを生成する。
3. **データパケット着信と IPSG 検証フェーズ**:
   * Untrusted ポートから IP データパケットが通過しようとする際、IPSG がポートに動的アサインした PACL (Port Access Control List) がハードウェア ASIC 上で評価される。
   * 送信元 IP（`ip verify source`）および送信元 MAC（`ip verify source port-security`）がバインドテーブルと一致しない場合、パケットはスイッチポートの入口で破棄される。
4. **L3 バウンダリでの uRPF 検証フェーズ**:
   * パケットがルーターまたは ASA/FTD の L3 インターフェイスに着信。
   * パケットの送信元 IP アドレスを用いて CEF FIB テーブルの反転照合を実行。
   * **Strict Mode**: パケットの着信ポートが、CEF が判定する対象送信元 IP への「最良戻りルート (Best Return Path)」のインターフェイスと異なる場合はドロップ。
   * **Loose Mode**: 送信元 IP が FIB 内に存在しない（または Null0 ルートを指している）場合にドロップ。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **DHCP Snooping と Option 82 の問題（試験の罠）**:
   * スイッチで `ip dhcp snooping` を有効化すると、デフォルトで DHCP パケットに Option 82（Relay Agent Information）が自動挿入されます。
   * **問題発生パターン**: 直結された Cisco IOS DHCP サーバーは、`giaddr=0` かつ Option 82 が付与された DHCP パケットを不正とみなしてデフォルトで拒否（ドロップ）します。
   * **回避策**:
     * スイッチ側: **`no ip dhcp snooping information option`** コマンドを投入して Option 82 挿入を停止させる。
     * サーバー側: **`ip dhcp relay information trust-all`** を設定して Option 82 付きの直結パケットを容認させる。
2. **DAI (Dynamic ARP Inspection) での非 DHCP / 静的 IP 端末の保護**:
   * 静的 IP アドレス（Static IP）が割り当てられたサーバーやプリンターが存在する VLAN で DAI を有効化すると、DHCP Snooping DB にエントリが存在しないため通信が全遮断されます。
   * **解決手順**:
     1. `arp access-list <ACL_NAME>` を定義し、`permit ip host <IP> mac host <MAC>` エントリを作成。
     2. `ip arp inspection filter <ACL_NAME> vlan <VLAN_ID> [static]` を適用。
     3. **`static` キーワードの有無の差**: `static` を付与すると ARP ACL のみを参照し DHCP DB を検索しません。付与しない場合は ARP ACL で一致しないパケットを続けて DHCP DB と照合します。
3. **IPSG (IP Source Guard) の適用手順と Static Binding**:
   * **設定コマンド**: インターフェイス上で **`ip verify source`**（IP のみ検証）または **`ip verify source port-security`**（IP ＋ MAC 同時検証）を投入。
   * **前提条件**: `port-security` オプションを使用する場合は、対象ポートで事前に **`switchport port-security`** を有効化しておく必要があります。
   * **静的バインド**: 静的 IP 端末に対しては `ip source binding <MAC> vlan <VLAN> <IP> interface <INT>` をグローバルで定義します。
4. **uRPF (Unicast RPF) Strict vs Loose Mode ＆ ACL 例外設定**:
   * **Strict Mode (`ip verify unicast source reachable-via rx`)**: 送信元 IP が FIB に存在し、かつ着信ポートが CEF のベストリターンポートと完全一致することを要求。
   * **`allow-default` オプション**: デフォルトルート (`0.0.0.0/0`) を到達可能経路として評価に含める場合は **`allow-default`** を付与します。
   * **`allow-self-ping` オプション**: ルーター自身の IP を送信元とするセルフパケットを許可する場合は **`allow-self-ping`** を付与します。
   * **ACL 例外設定**: RPF チェックに失敗したパケットを即時破棄せず、ACL で条件判定させる場合は `ip verify unicast source reachable-via rx <ACL_NUM>` を指定します（ACL で `permit` されたパケットは RPF 違反でも通過を許可され、`log` オプションで追跡可能）。
5. **ASA / FTD での Reverse-Path (Anti-Spoofing) 構成**:
   * **ASA LINA**: インターフェイス単位で **`ip verify reverse-path interface <nameif>`** を投入。
   * **FMC / FTD**: `Devices > Platform Settings` または FMC Interface Configuration 内の Anti-Spoofing チェックボックスを有効化して Deploy。
6. **IPv6 First Hop Security (FHS) ポリシー構成**:
   * **RA Guard**: `ipv6 nd raguard policy <NAME>` を定義 ➔ `device-role {host|router}` ➔ ポートまたは VLAN で `ipv6 nd raguard attach-policy <NAME>` をバインド。
   * **DHCPv6 Guard**: `ipv6 dhcp guard policy <NAME>` を定義 ➔ `device-role {client|server}` ➔ ポート/VLAN にアサイン。
   * **IPv6 Source Guard**: `ipv6 source guard` または `ipv6 snooping policy`（SISF）を構成し、未登録の IPv6 送信元パケットをブロック。

---

## 🛠 設定方法

### 1. Switch L2 IPv4 Anti-Spoofing (DHCP Snooping + DAI + IPSG + Port Security)

```bash
! ===== Step 1: DHCP Snooping の有効化と Option 82 の調整 =====
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option   ! IOS DHCP サーバーとの不整合防止

interface GigabitEthernet1/0/24          ! DHCP Server / Uplink Port
 ip dhcp snooping trust                  !

! ===== Step 2: Dynamic ARP Inspection (DAI) の設定 =====
ip arp inspection vlan 10,20
ip arp inspection validate src-mac dst-mac ip   ! 詳細サニティチェック

! 静的 IP サーバー用の ARP ACL 定義と適用
arp access-list DAI_STATIC_SERVERS
 permit ip host 10.1.1.10 mac host 0050.56aa.bbcc log

ip arp inspection filter DAI_STATIC_SERVERS vlan 10

interface GigabitEthernet1/0/24          ! Uplink Port
 ip arp inspection trust                 !

! ===== Step 3: IP Source Guard (IPSG) ＆ Static Binding =====
ip source binding 0050.56aa.bbcc vlan 10 10.1.1.10 interface GigabitEthernet1/0/10

interface range GigabitEthernet1/0/1 - 10   ! User Access Ports
 switchport mode access
 switchport access vlan 10
 switchport port-security                !
 switchport port-security maximum 2
 switchport port-security violation shutdown
 ip verify source port-security          ! IP + MAC の同時検証
```

---

### 2. Router uRPF (Strict Mode ＆ Loose Mode ＆ ACL Exemption)

```bash
! ===== 共通前提: CEF の有効化 =====
ip cef

! ===== uRPF Strict Mode (ACL 例外 ＆ デフォルトルート許容) =====
ip access-list extended ACL_URPF_EXEMPT
 permit ip 192.168.99.0 0.0.0.255 any log-input   ! 特定スプーフィングの調査ログ許可
 deny ip any any log-input

interface GigabitEthernet0/0/0           ! Single-homed LAN / WAN Boundary
 ip verify unicast source reachable-via rx allow-default allow-self-ping ACL_URPF_EXEMPT

! ===== uRPF Loose Mode (BGP RTBH / 非対称ルーティング環境向け) =====
interface GigabitEthernet0/0/1           ! Multi-homed ISP Interface
 ip verify unicast source reachable-via any
```

---

### 3. ASA / FTD Anti-Spoofing (Reverse-Path Verification)

```text
! ===== ASA CLI 設定例 =====
ip verify reverse-path interface OUTSIDE
ip verify reverse-path interface INSIDE
ip verify reverse-path interface DMZ

! ===== FTD FMC GUI 操作手順 =====
1. FMC UI から Devices > Platform Settings に移動。
2. 対象 FTD の Platform Settings ポリシーを編集。
3. "Threat Defense Settings" ➔ "Anti-Spoofing" を選択。
4. 対象インターフェイス (例: OUTSIDE, INSIDE) の "Enable Anti-Spoofing (uRPF)" にチェックを入れる。
5. Save して Deploy を実行。
```

---

### 4. IPv6 First Hop Security (RA Guard + DHCPv6 Guard + IPv6 Source Guard)

```bash
! ===== Step 1: IPv6 RA Guard の構成 =====
ipv6 prefix-list RA_APPROVED_PREFIX permit 2001:db8:10::/64
ipv6 access-list RA_APPROVED_SOURCE
 permit ipv6 host FE80::1 any

ipv6 nd raguard policy POLICY_RA_ROUTER
 device-role router
 match ra prefix-list RA_APPROVED_PREFIX
 match ipv6 access-list RA_APPROVED_SOURCE

ipv6 nd raguard policy POLICY_RA_HOST
 device-role host

interface GigabitEthernet1/0/24          ! Router Port
 ipv6 nd raguard attach-policy POLICY_RA_ROUTER

vlan configuration 10                    ! User Host Access VLAN
 ipv6 nd raguard attach-policy POLICY_RA_HOST

! ===== Step 2: IPv6 DHCPv6 Guard の構成 =====
ipv6 dhcp guard policy POLICY_DHCPV6_SERVER
 device-role server

interface GigabitEthernet1/0/24          ! DHCPv6 Server Port
 ipv6 dhcp guard attach-policy POLICY_DHCPV6_SERVER

! ===== Step 3: IPv6 Source Guard / SISF Snooping の構成 =====
ipv6 snooping policy POLICY_IPV6_SNOOPING
 device-role node
 protocol dhcp
 protocol ndp

vlan configuration 10
 ipv6 snooping attach-policy POLICY_IPV6_SNOOPING

interface range GigabitEthernet1/0/1 - 10
 ipv6 source guard
```

---

## 🔍 検証コマンド

| 目的 | コマンド (IOS / ASA / FTD) | 表示確認ポイント |
| :--- | :--- | :--- |
| **DHCP Snooping ステータス ＆ バインドテーブル確認** | <code>show ip dhcp snooping</code><br><code>show ip dhcp snooping binding</code> | 有効 VLAN、Trust インターフェイス、動的学習された MAC/IP/Port/VLAN バインド一覧 |
| **DAI VLAN 適用状態 ＆ バインドログ確認** | <code>show ip arp inspection vlan 10</code><br><code>show ip arp inspection log</code> | Active 状態、ACL Match 設定、ドロップ判定件数、ログバッファ内容 |
| **IPSG ポートフィルター適用状態確認** | <code>show ip verify source</code><br><code>show ip source binding</code> | Filter-type (ip / ip-mac)、Filter-mode (active / deny-all / inactive)、Static Bindings |
| **uRPF 設定 ＆ ドロップカウンター確認 (Router)** | <code>show ip interface Gi0/0</code><br><code>show cef interface Gi0/0</code> | `IP verify source reachable-via RX/ANY`、`verification drops` カウンター |
| **ASA Anti-Spoofing (uRPF) 統計確認** | <code>show ip verify statistics</code> | インターフェイスごとの `unicast rpf drops` 件数 |
| **IPv6 RA Guard / DHCPv6 Guard ポリシー確認** | <code>show ipv6 snooping features</code><br><code>show ipv6 snooping policies</code> | READY 状態の機能、Target (Port/VLAN) へのポリシーアサイン状態 |
| **IPv6 FHS ドロップメッセージ追跡** | <code>show ipv6 snooping counters</code> | Protocol ごとの Bridged / Dropped メッセージカウンター |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **DHCP Snooping 有効化後、全クライアントが IP アドレスを取得できなくなる** | DHCP サーバー接続ポートで `ip dhcp snooping trust` が未設定、または Option 82 が原因で IOS サーバーが拒否 | <code>show ip dhcp snooping</code> | DHCP サーバー宛てポートを Trust 設定する。IOS DHCP サーバー環境では `no ip dhcp snooping information option` を投入。 |
| **DAI 有効化後、静的 IP サーバーへのアクセスが完全に途絶する** | DHCP Snooping DB にサーバー情報が存在せず、すべての ARP パケットが Invalid として拒否されている | <code>show ip arp inspection log</code> | `arp access-list` を作成し、`ip arp inspection filter <ACL> vlan <VLAN>` で静的バインドを追加定義する。 |
| **IPSG を投入したポートで `Filter-mode: inactive-no-snooping-vlan` と表示され機能しない** | 対象 VLAN で DHCP Snooping (`ip dhcp snooping vlan <VLAN>`) が有効化されていない | <code>show ip verify source</code> | グローバルで `ip dhcp snooping` および対象 VLAN で `ip dhcp snooping vlan <X>` を有効化する。 |
| **`ip verify source port-security` 設定時、データパケットが全て拒否される** | 対象ポートで Port Security (`switchport port-security`) が有効化されていない | <code>show ip verify source</code><br><code>show port-security interface</code> | 対象インターフェイスで `switchport port-security` コマンドを投入する。 |
| **uRPF Strict Mode 設定後、マルチホーム接続の一部の拠点通信が切断される** | 非対称ルーティングにより、パケット着信ポートと CEF ベストリターンポートが不一致を起こしている | <code>show ip interface</code><br><code>show ip route</code> | 非対称ルーティング環境では uRPF を **Loose Mode (`reachable-via any`)** に変更する。 |
| **uRPF 投入後、デフォルトルート経由の外部通信がすべてドロップされる** | uRPF 設定時に `allow-default` オプションが付与されていないため、FIB の `0.0.0.0/0` 照合が拒否されている | <code>show ip interface</code> | コマンド末尾に **`allow-default`** キーワードを追加する (`ip verify unicast source reachable-via rx allow-default`)。 |
| **IPv6 RA Guard 設定後、正規ルーターからの RA もドロップされ IPv6 クライアントが通信不能になる** | ルーター接続ポートで `device-role router` ポリシーがアサインされていない | <code>show ipv6 snooping policies</code> | ルーター宛てポートに `device-role router` を設定した RA Guard ポリシーをアサインする。 |

---

## ⚠ 制限事項

* **DHCP Snooping DB の不揮発性ストレージ保存要件**:
  * スイッチが再起動すると、メモリ上の DHCP Snooping Binding Table は消失します。再起動後の DAI/IPSG 誤ドロップを防ぐため、実稼働環境では `ip dhcp snooping database flash:/dhcp_db.txt` などの外部/ローカル保存設定が強く推奨されます。
* **uRPF の非対称ルーティング制限**:
  * uRPF Strict Mode は、Inbound パスと Outbound パスが異なる非対称ルーティング環境では使用できません（パケットが誤ドロップされます）。この場合は Loose Mode を使用します。
* **In-band Management と Anti-Spoofing の相互作用**:
  * ASA/FTD において、同一インターフェイス上で PBR (Policy-Based Routing) と Anti-Spoofing (uRPF) を同時有効化すると、PBR ルートマップではなく通常のルーティングテーブルで uRPF 検証が行われるため、パケットが不正ドロップされる制限があります。

---

## 🔄 他技術との関連

* **CEF (Cisco Express Forwarding)**: uRPF の反転照合処理（FIB テーブル検索）を実行するための必須前提コンポーネント。
* **DHCP / SLAAC / NDP**: L2 バインドデータベース（DHCP Snooping DB / SISF Neighbor Table）を生成するための制御プロトコル基盤。
* **Private VLANs (PVLAN)**: L2 セグメント内でのホスト間直接通信（横移動）を物理的に分離・遮断する補完セキュリティ技術。
* **BGP RTBH (Remotely Triggered Black Hole)**: Source-based RTBH 構築時に、エッジルーターの Ingress インターフェイスで **uRPF Loose Mode** を併用して攻撃パケットを破棄。

---

## 🧩 比較表

### 1. IPv4 L2 Spoofing Mitigation (DHCP Snooping vs DAI vs IPSG vs Port Security)

| 比較項目 | DHCP Snooping | Dynamic ARP Inspection (DAI) | IP Source Guard (IPSG) | Port Security |
| :--- | :--- | :--- | :--- | :--- |
| **保護レイヤー** | Layer 2 / 7 (DHCP) | Layer 2 (ARP) | Layer 2 / 3 (IP Data) | Layer 2 (MAC) |
| **主な対象攻撃** | Rogue DHCP Server, Starvation | ARP Poisoning, MitM 攻撃 | IP / MAC Spoofing | CAM Table Flood, MAC Spoofing |
| **検証データ** | DHCP Message (Offer/ACK) | ARP Request / Reply (IP/MAC) | Ingress IP Data Packets | Ingress Frame Source MAC |
| **基盤データベース** | 自ら動的生成・保持 | **DHCP Snooping DB** / ARP ACL | **DHCP Snooping DB** / Static | Secure MAC Table (CAM) |
| **設定単位** | Global / VLAN / Interface | Global / VLAN / Interface | Interface (Access Ports) | Interface (Access Ports) |

### 2. Unicast RPF Strict Mode vs Loose Mode

| 比較項目 | uRPF Strict Mode (`reachable-via rx`) | uRPF Loose Mode (`reachable-via any`) |
| :--- | :--- | :--- |
| **検証条件** | 送信元 IP が FIB に存在し、かつ**着信ポートがベストリターンポートと一致** | 送信元 IP が FIB に存在するかのみ（Null0 宛て以外は通過） |
| **主な用途** | シングルホーム LAN/WAN 境界、アクセスエッジ | マルチホーム ISP 境界、BGP RTBH 連携 |
| **非対称ルーティング** | **非対応**（正常トラフィックを誤ドロップする） | **対応**（パス不一致でも FIB に存在すれば通過） |
| **デフォルトルート照合**| デフォルトで照合対象外（`allow-default` で許可） | デフォルトで照合対象（FIB にデフォルトがあれば通過） |

---

## 💡 ベストプラクティス

1. **アクセス層における L2 統合セキュリティの標準展開順序**:
   * アクセススイッチにおいて、**`DHCP Snooping` ➔ `Port Security` ➔ `DAI` ➔ `IPSG`** の順番で段階的に有効化し、インフラポート（Uplink / Server Ports）に対して確実に `trust` アサインを適用します。
2. **uRPF 展開時の `allow-default` ＆ ACL ログ活用**:
   * ルーター境界で uRPF Strict Mode を展開する際は、デフォルトルート環境での全ドロップ事故を防ぐため `allow-default` を併用し、不整合パケットの調査・トラブルシューティング用に `log-input` 付き ACL 例外を一時配置します。
3. **IPv6 デュアルスタック環境における RA Guard の全アクセスポート必須バインド**:
   * Windows 等の主要 OS は IPv6 がデフォルトで有効化されているため、IPv4 のみの運用であっても Rogue RA による意図しない IPv6 トンネル・デフォルトゲートウェイ乗っ取りを防ぐため、アクセスポートへ `RA Guard (device-role host)` ポリシーを標準適用します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: DHCP Snooping と Option 82 制御の設定

#### 問題
スイッチ SW1 の VLAN 10, 20 において DHCP Snooping を有効化せよ。Uplink ポート `Gi1/0/24` のみを Trusted ポートとし、直接接続された Cisco IOS DHCP サーバーでエラーが発生しないよう Option 82 挿入を無効化せよ。

#### 要件
* グローバルおよび対象 VLAN で有効化すること。
* Option 82 の挿入をグローバルで停止させること。

#### 設定例

```bash
! ===== SW1 Configuration =====
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option

interface GigabitEthernet1/0/24
 ip dhcp snooping trust
```

#### 検証方法
* `show ip dhcp snooping` を実行し、`Switch DHCP snooping is enabled`、`Insertion of option 82 is disabled`、および `Gi1/0/24` が `Trusted: yes` であることを確認する。

---

### 例2: Dynamic ARP Inspection (DAI) と Static ARP ACL の組み合わせ設定

#### 問題
スイッチ SW1 の VLAN 10 で DAI を有効化せよ。DHCP を使用しない静的 IP サーバー (`10.1.1.50` / MAC: `0050.56aa.bbcc`) の ARP パケットを正しく通過させるため、Static ARP ACL を定義して適用せよ。

#### 要件
* ARP ACL `DAI_STATIC_ACL` を作成して適用すること。
* Uplink ポート `Gi1/0/24` を DAI Trust ポートに設定すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
arp access-list DAI_STATIC_ACL
 permit ip host 10.1.1.50 mac host 0050.56aa.bbcc log

ip arp inspection filter DAI_STATIC_ACL vlan 10
ip arp inspection vlan 10
ip arp inspection validate src-mac dst-mac ip

interface GigabitEthernet1/0/24
 ip arp inspection trust
```

#### 検証方法
* `show ip arp inspection vlan 10` を実行し、`ACL Match: DAI_STATIC_ACL` および `Operation: Active` であることを確認する。

---

### 例3: IP Source Guard (IPSG) と Port Security の統合適用

#### 問題
SW1 のアクセスポート `Gi1/0/1` 〜 `Gi1/0/10` (VLAN 10) において、DHCP Snooping DB に基づく IP アドレスおよび MAC アドレスの同時検証 (IPSG + Port Security) を有効化せよ。

#### 要件
* `ip verify source port-security` コマンドを使用すること。
* 対象ポートで Port Security を有効化すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
interface range GigabitEthernet1/0/1 - 10
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 ip verify source port-security
```

#### 検証方法
* `show ip verify source` を実行し、対象ポートの `Filter-type` が `ip-mac`、`Filter-mode` が `active` であることを確認する。

---

### 例4: 静的 IP 環境における Static IP Source Binding と IPSG の適用

#### 問題
SW1 のポート `Gi1/0/10` に接続された静的 IP サーバー (`10.1.1.100` / MAC: `0011.2233.4455` / VLAN 10) に対し、静的 IP Source Binding エントリを作成して IPSG を有効化せよ。

#### 要件
* `ip source binding` コマンドを使用すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
ip source binding 0011.2233.4455 vlan 10 10.1.1.100 interface GigabitEthernet1/0/10

interface GigabitEthernet1/0/10
 switchport mode access
 switchport access vlan 10
 ip verify source
```

#### 検証方法
* `show ip source binding` を実行し、Type が `static` としてアサインされていることを確認する。

---

### 例5: Router uRPF Strict Mode と ACL 例外許可の設定

#### 問題
ルーター R1 の WAN インターフェイス `Gi0/0/0` において、Strict Mode uRPF を有効化せよ。デフォルトルートでの照合および自発パケット (Self-ping) を許可し、テスト用サブネット (`192.168.99.0/24`) からの RPF 違反パケットのみを例外許可してログ記録せよ。

#### 要件
* `ip verify unicast source reachable-via rx` コマンドを使用すること。

#### 設定例

```bash
! ===== R1 Configuration =====
ip cef

ip access-list extended ACL_URPF_EXEMPTION
 permit ip 192.168.99.0 0.0.0.255 any log-input
 deny ip any any log-input

interface GigabitEthernet0/0/0
 ip verify unicast source reachable-via rx allow-default allow-self-ping ACL_URPF_EXEMPTION
```

#### 検証方法
* `show ip interface GigabitEthernet0/0/0` を実行し、`IP verify source reachable-via RX, allow default, allow self-ping, ACL ACL_URPF_EXEMPTION` および verification drops 件数を確認する。

---

### 例6: Router uRPF Loose Mode による BGP RTBH 連携アンチスプーフィング

#### 問題
マルチホームルーター R3 の WAN ポート `Gi0/0/1` において、非対称ルーティング環境に対応した uRPF Loose Mode を構成し、BGP RTBH (Next-Hop Null0) で指定された攻撃元 IP からのパケットを自動破棄せよ。

#### 要件
* `ip verify unicast source reachable-via any` コマンドを使用すること。

#### 設定例

```bash
! ===== R3 Configuration =====
ip cef
ip route 192.0.2.1 255.255.255.255 Null0

interface GigabitEthernet0/0/1
 ip verify unicast source reachable-via any
```

#### 検証方法
* `show cef interface GigabitEthernet0/0/1` を実行し、`IP unicast RPF check is enabled` であることを確認する。

---

### 例7: ASA / FTD での Reverse-Path (Anti-Spoofing) 検証設定

#### 問題
ASA1 の全アクティブインターフェイス (`OUTSIDE`, `INSIDE`, `DMZ`) において、IP 送信元偽装防止のための Reverse-Path (uRPF) 検証を有効化せよ。

#### 要件
* `ip verify reverse-path` コマンドを使用すること。

#### 設定例

```text
! ===== ASA1 Configuration =====
ip verify reverse-path interface OUTSIDE
ip verify reverse-path interface INSIDE
ip verify reverse-path interface DMZ
```

#### 検証方法
* `show ip verify statistics` を実行し、各インターフェイスの `unicast rpf drops` カウンターが表示されることを確認する。

---

### 例8: IPv6 RA Guard ポリシー（Host / Router Role / Prefix Match）の構成

#### 問題
スイッチ SW3 の VLAN 11 において、不正な IPv6 Router Advertisement パケットを拒否するため RA Guard を構成せよ。ルーターポート `Gi1/0/1` からの RA のみ（送信元: `FE80::4` / プレフィックス: `2001:db8:11::/64`）を許可し、アクセスポートを Host モードにアサインせよ。

#### 要件
* `ipv6 nd raguard policy` コマンドを使用すること。

#### 設定例

```bash
! ===== SW3 Configuration =====
ipv6 prefix-list RA_PREFIX permit 2001:db8:11::/64
ipv6 access-list RA_SOURCE
 permit ipv6 host FE80::4 any

ipv6 nd raguard policy POLICY_ROUTER
 device-role router
 match ra prefix-list RA_PREFIX
 match ipv6 access-list RA_SOURCE

ipv6 nd raguard policy POLICY_HOST
 device-role host

interface GigabitEthernet1/0/1
 ipv6 nd raguard attach-policy POLICY_ROUTER

vlan configuration 11
 ipv6 nd raguard attach-policy POLICY_HOST
```

#### 検証方法
* `show ipv6 snooping policies` を実行し、`Gi1/0/1` に `ROUTER`、`vlan 11` に `HOST` ポリシーがアサインされていることを確認する。

---

### 例9: IPv6 DHCPv6 Guard ポリシーと Server / Scope 条件の設定

#### 問題
SW3 の VLAN 11 において、未承認の DHCPv6 サーバーからの広告メッセージを遮断する DHCPv6 Guard を構成せよ。正規 DHCPv6 サーバーポート `Gi1/0/24` のみを Server モードとして許可せよ。

#### 要件
* `ipv6 dhcp guard policy` コマンドを使用すること。

#### 設定例

```bash
! ===== SW3 Configuration =====
ipv6 dhcp guard policy POLICY_DHCPV6_SERVER
 device-role server

interface GigabitEthernet1/0/24
 ipv6 dhcp guard attach-policy POLICY_DHCPV6_SERVER
```

#### 検証方法
* `show ipv6 snooping features` を実行し、`DHCP Guard` が READY 状態であることを確認する。

---

### 例10: IPv6 Source Guard (SISF / IPv6 Snooping Policy) による SLAAC/NDP スプーフィング防止

#### 問題
SW3 の VLAN 11 およびアクセスポート `Gi1/0/1` 〜 `Gi1/0/10` において、IPv6 Neighbor Binding Table に存在しない送信元 IPv6 アドレスからのデータパケットをポートレベルで拒否する IPv6 Source Guard を有効化せよ。

#### 要件
* `ipv6 snooping policy` および `ipv6 source guard` を使用すること。

#### 設定例

```bash
! ===== SW3 Configuration =====
ipv6 snooping policy VLAN_SNOOPING_POLICY
 device-role node
 protocol dhcp
 protocol ndp

vlan configuration 11
 ipv6 snooping attach-policy VLAN_SNOOPING_POLICY

interface range GigabitEthernet1/0/1 - 10
 ipv6 source guard
```

#### 検証方法
* `show ipv6 snooping policies` を実行し、アクセスポートに `Source guard` ポリシーがアサインされていることを確認する。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: スイッチ SW1 上で以下のコンフィグを投入した。

```bash
ip dhcp snooping
ip dhcp snooping vlan 10
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
 ip verify source port-security
```

クライアント PC を `Gi1/0/1` に接続して DHCP 接続を試みたところ、IP アドレスを取得した直後から、一切の IP トラフィック（Ping や Web アクセス）がスイッチポートで即座にドロップされた。原因として正しいものはどれか。

* A) `Gi1/0/1` ポートで `switchport port-security` が設定されていないため、IPSG が不完全な状態 (`deny-all`) に陥っている。
* B) DHCP サーバーの IP アドレスが `ip source binding` に静的登録されていないため。
* C) uRPF がルーター上で有効化されていないため。
* D) ポートが Trunk モードになっていないため。

**正解**: **A**  
**解説**: `ip verify source port-security` コマンド（IP + MAC の同時検証）を使用する場合、対象ポートで **`switchport port-security` が有効化されていることが必須前提要件** となります。Port Security が未設定の場合、IPSG は MAC アドレス情報を取得できず、ポートのフィルタリングモードが `deny-all` に設定されて全トラフィックを破棄します。

---

### 2. トラブルシュート問題

**問題**: ルーター R1 の WAN ポート `Gi0/0/0` で `ip verify unicast source reachable-via rx` を設定した。直後から、インターネット宛てのすべての通信が切断された。R1 のルーティングテーブルにはデフォルトルート (`0.0.0.0/0 via 203.0.113.1`) のみが存在する。問題を修正するための正確な設定変更はどれか。

* A) CEF を無効化する (`no ip cef`)。
* B) インターフェイスコマンドを `ip verify unicast source reachable-via rx allow-default` に変更する。
* C) uRPF コマンドを削除し、DAI を有効化する。
* D) ポートを Trunk モードへ変更する。

**正解**: **B**  
**解説**: uRPF Strict Mode (`reachable-via rx`) は、デフォルトで FIB 内の特定ルート (Specific Routes) のみを到達可能パスとして評価し、デフォルトルート (`0.0.0.0/0`) を無視します。デフォルトルート経由の戻りトラフィックを通過させるには、コマンド末尾に **`allow-default`** キーワードを追加する必要があります。

---

### 3. Design 問題

**問題**: 企業 LAN 内において、ユーザーが個人の Wi-Fi ルーターやスマホのテザリング（Rogue Router）を LAN スイッチに接続し、不正な IPv6 Router Advertisement パケットを送出して社内 PC の IPv6 デフォルトゲートウェイを奪取する攻撃（Rogue RA Attack）が発生している。スイッチ層でこの攻撃を遮断するための最適な設計手法はどれか。

* A) 全アクセスポートに `ip verify source` を適用する。
* B) 全アクセスポートに IPv6 RA Guard ポリシー (`device-role host`) をバインドし、正規ルーター接続ポートのみ `device-role router` に設定する。
* C) ルーターで uRPF Loose Mode を設定する。
* D) スイッチの Spanning Tree を Portfast に変更する。

**正解**: **B**  
**解説**: **IPv6 RA Guard** は、L2 スイッチポートに着信した ICMPv6 Router Advertisement (Type 134) パケットをインスペクションし、`device-role host` が設定されたポートからの RA メッセージを即座に破棄することで、Rogue RA 攻撃を完全に遮断します。

---

### 4. 実装問題

**問題**: Catalyst スイッチ SW1 において、DHCP を使用しない静的 IP 端末の ARP 偽装（ARP Poisoning）を防ぐため、DAI が参照する Static ARP ACL を作成し、VLAN 10 に適用した。このとき、ARP ACL に一致しないパケットを DHCP Snooping DB と照合させず、完全に拒否させるために必要なキーワードはどれか。

* A) `ip arp inspection filter <ACL> vlan 10 static`
* B) `ip arp inspection trust`
* C) `ip arp inspection validate ip`
* D) `ip verify source port-security`

**正解**: **A**  
**解説**: `ip arp inspection filter <ACL> vlan <VLAN>` コマンドの末尾に **`static`** キーワードを付与すると、スイッチは指定した ARP ACL のみを厳格評価し、一致しないパケットを DHCP Snooping データベースへフォールバック照合させずに即座にドロップします。

---

### 5. コンフィグ読解問題

**問題**: ルーターの WAN インターフェイスで以下のコマンドが投入されている。

```bash
ip verify unicast source reachable-via any
```

この uRPF 設定の動作に関する説明として正しいものはどれか。

* A) 送信元 IP が FIB 内に存在し、かつ着信ポートが最良戻りルートと一致する場合のみ許可される。
* B) 送信元 IP が FIB (CEF) テーブル内に存在していれば、どの着信ポートから入ってきたパケットであっても通過を許可する（ただし Next-Hop が Null0 のルートはドロップされる）。
* C) パケットの宛先 IP アドレスのみが検証される。
* D) すべてのパケットが無条件でドロップされる。

**正解**: **B**  
**解説**: **`reachable-via any` (uRPF Loose Mode)** は、パケットの送信元 IP アドレスが CEF FIB テーブル内に存在するかのみを確認します。着信インターフェイスの一致要求を行わないため、非対称ルーティング環境や BGP RTBH (Source-based) 防御において威力を発揮します。

---

## 🔗 参考リソース

* [Cisco Catalyst Security Configuration Guide: Layer 2 Anti-Spoofing Features (DHCP Snooping, DAI, IPSG)](https://www.cisco.com/)
* [Cisco IOS Security Configuration Guide: Unicast Reverse Path Forwarding (uRPF)](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot IPv6 First-Hop Security (FHS)](https://www.cisco.com/)
* [Cisco ASA Series Configuration Guide: Anti-Spoofing Configuration](https://www.cisco.com/)
* [Cisco Live: Advanced Layer 2 and Layer 3 Threat Mitigation (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験において「Mitigate IPv4/IPv6 spoofing attacks on Access Switches and Edge Routers」が出題された場合の構成選択チャート：
    * **IPv4 L2 Access**: `ip dhcp snooping` ➔ `ip arp inspection vlan` ➔ `ip verify source [port-security]`（アクセスポート）＋ Uplink ポートで `trust`。
    * **IPv4 L3 Edge**: `ip verify unicast source reachable-via rx allow-default allow-self-ping`（ルーター WAN ポート）。
    * **ASA / FTD**: `ip verify reverse-path interface <nameif>`。
    * **IPv6 L2 Access**: `ipv6 nd raguard policy` ➔ `ipv6 dhcp guard policy` ➔ `ipv6 source guard` / `ipv6 snooping policy`。
* **図解（Anti-Spoofing トラフィック検証マトリクス）**:
  ```text
  [ Ingress Frame / Packet ]
             │
             ├── Is Layer 2 Frame (IPv4 ARP / Data)?
             │    ├── ARP Packet ──► DAI Engine ──► Validate against DHCP DB / ARP ACL ──► Pass/Drop
             │    └── IP Data Packet ──► IPSG Engine ──► Validate Src IP/MAC against Binding Table ──► Pass/Drop
             │
             ├── Is Layer 3 Packet (Routed Interface)?
             │    └── IP Packet ──► uRPF Engine ──► CEF FIB Reverse Lookup (Strict / Loose) ──► Pass/Drop
             │
             └── Is IPv6 Control Packet (NDP / RA / DHCPv6)?
                  ├── ICMPv6 RA ──► RA Guard ──► Validate Role (Host vs Router) ──► Pass/Drop
                  └── DHCPv6 Reply ──► DHCPv6 Guard ──► Validate Role (Server vs Client) ──► Pass/Drop
  ```
* **注意点**:
  * ラボ試験で DAI、IPSG、uRPF、RA Guard を設定した後は、`show ip dhcp snooping binding`、`show ip verify source`、`show ip interface <IF>`、`show ipv6 snooping policies` を必ず実行し、バインド DB が正しく反映され、検証ドロップカウンターが意図通り記録されていることを確認してください。

---

