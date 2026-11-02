---
layout: default
title: 2.5.c-GRE
nav_order: 3
parent: 2.5-Infrastructure-segmentation
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.5.c GRE

Generic Routing Encapsulation (GRE) は、RFC 1701 および RFC 2784 で標準化された汎用トンネリングプロトコルであり、任意のネットワーク層プロトコル（IPv4、IPv6、IPX、AppleTalk 等）およびマルチキャストパケット（OSPF/EIGRP Hello、PIM パケット等）を IP トンネル内にカプセル化して、異種ネットワークやアンダーレイ IP インフラストラクチャを超えて透過的に伝送する技術です。

CCIE Security v6.1 Blueprint **2.5.c GRE** では、**Point-to-Point GRE** および **Multipoint GRE (mGRE)** の基本動作、**GRE over IPsec** (Transport Mode vs Tunnel Mode, Dynamic IPsec Protection)、**GRE Keepalive** 機構、**VRF-Aware GRE** (Front-Door VRF [FVRF] 認証・トランスポート分離および Inside VRF [IVRF] 統合)、**MTU / TCP MSS パス調製**、ならびに **Cisco ASA** および **Cisco FTD (Firepower Threat Defense / FMC FlexConfig)** における GRE 実装制限と回避構成に至るまで、実効性の高い設計・構築・検証能力が深く評価されます。

---

## 📘 概要

* **機能概要**:
  * **Generic Routing Encapsulation (GRE - IP Protocol 47)**: 送信元ルーター（GRE Tunnel Ingress）において、元の IP パケット（Payload）の前に GRE ヘッダー（4～16 バイト）および新しいアウター IP ヘッダー（20 バイト）を付加し、既存の IP ネットワークを通過させるカプセル化技術。
  * **Point-to-Point GRE**: トンネル送信元 (Tunnel Source) とトンネル宛先 (Tunnel Destination) が 1 対 1 で静的に固定定義される標準的な GRE トンネル。
  * **Multipoint GRE (mGRE)**: 単一の Tunnel インターフェイスで複数の動的ピア（Spoke 等）と接続可能な 1 対多トンネル。NHRP (Next Hop Resolution Protocol) や DMVPN と組み合わせて動的メッシュ接続を実現する。
  * **GRE over IPsec**: 単体では暗号化および認証機能を持たない GRE パケットに対し、IPsec (ESP Protocol 50) による暗号化と完全性保護を組み合わせる構成。
  * **VRF-Aware GRE / FVRF**: トンネルのアンダーレイ（物理トランスポート層: `tunnel vrf <FVRF>`）とオーバーレイ（トンネル内部のルーティング層: `ip vrf forwarding <IVRF>`）を異なる VRF に分離し、サービスプロバイダ網と社内ルーティングテーブルの衝突を防止する構成。
* **利用目的**:
  * IPsec 単体では直接伝送できないマルチキャストトラフィック（ルーティングプロトコル Hello/Update パケット、PIM 等）を IPsec トンネル経由で拠点間伝送する。
  * 非ルーティング型プロトコルや IPv6 トラフィックを、既存の IPv4 インフラ上でカプセル化して対向拠点へ延伸する。
  * オーバーレイネットワークの構築および SD-WAN / 拠点間オーバーレイにおける柔軟なトポロジー形成。
* **どのような場面で利用するか**:
  * 企業拠点間（Headquarters ↔ Branch）での OSPF / EIGRP / BGP 動的ルーティング情報の保護伝送（GRE over IPsec）。
  * サービスプロバイダ網やマルチテナント環境において、アンダーレイ IP 網と顧客オーバーレイ IP 網をマルチ VRF（FVRF/IVRF）で分離する構成。
  * クラウドプロバイダ（AWS Direct Connect, Azure ExpressRoute 等）やサードパーティプロキシへの動的 GRE 接続。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **評価対象技術** | **Point-to-Point GRE**、**Multipoint GRE (mGRE)**、**GRE over IPsec**、**GRE Keepalive**、**VRF-Aware GRE (FVRF/IVRF)**、**ASA / FTD GRE** |
| **IP プロトコル番号** | **IP Protocol 47** (GRE) |
| **オーバーヘッド** | 標準 GRE ヘッダー: **4 バイト**（Key/Sequence 付加時は最大 16 バイト）＋ アウター IP ヘッダー: **20 バイト** ＝ **合計 24 バイト**（IPsec 併用時はさらに約 52～72 バイト加算） |
| **暗号化・セキュリティ** | GRE 単体では **暗号化・改ざん検知・送信元認証機能は一切無し**（IPsec ESP/AH との組み合わせが必須） |
| **マルチキャスト対応** | **完全対応**（OSPF, EIGRP, BGP, PIM, RIPv2 等のマルチキャスト/ブロードキャストパケットを透過伝送） |
| **キープアライブ機構** | **GRE Keepalive**（送信元ルーターが自らを宛先とする外側 IP パケットを生成し、対向でループバック処理させて片方向リンク障害を検知） |
| **Cisco ASA サポート** | ASA 9.0(1) 以降で Point-to-Point GRE をサポート（※ただし ASA では GRE インターフェイス上での動的ルーティングプロトコル [OSPF/EIGRP] 稼働は制限あり） |
| **Cisco FTD サポート** | FTD 7.1 以降で Native GRE / VTI 制御をサポート。旧バージョンでは FMC FlexConfig を使用して CLI 補填 |

---

## 🏗 動作原理

### 1. GRE パケット構造 ＆ ヘッダーオーバレイ

元の IP パケット（Inner IP Header ＋ Payload）の先頭に GRE ヘッダー（4 バイト）が追加され、さらにそれをアンダーレイでルーティングするための Outer IP Header（20 バイト）がカプセル化されます。

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│ GRE Encapsulated Packet Structure                                                │
├───────────────────┬───────────────────┬───────────────────┬──────────────────────┤
│ Outer IP Header   │ GRE Header        │ Inner IP Header   │ Payload Data         │
│ (20 Bytes)        │ (4 Bytes)         │ (20 Bytes)        │ (TCP/UDP/OSPF/etc.)  │
├───────────────────┼───────────────────┼───────────────────┼──────────────────────┤
│ Src: 203.0.113.1  │ Flags (16-bits)   │ Src: 10.1.1.10    │ Data...              │
│ Dst: 198.51.100.1 │ Protocol Type     │ Dst: 10.2.2.20    │                      │
│ Protocol: 47      │ (0x0800 = IPv4)   │ Protocol: TCP/80  │                      │
└───────────────────┴───────────────────┴───────────────────┴──────────────────────┘
```

---

### 2. GRE over IPsec (Transport Mode vs Tunnel Mode) のヘッダー比較

GRE トラフィックを IPsec ESP (Protocol 50) で保護する場合、**Transport Mode** と **Tunnel Mode** でカプセル化効率が大きく異なります。

```text
[ 1. GRE over IPsec - Transport Mode (推奨: 20バイトの冗長 IP ヘッダーを削減) ]
┌───────────────────┬──────────────┬────────────┬───────────────────┬──────────────┬────────────────┬──────────────┐
│ Outer IP Header   │ ESP Header   │ GRE Header │ Inner IP Header   │ Payload Data │ ESP Trailer    │ ESP Auth     │
│ (20 Bytes)        │ (8 Bytes)    │ (4 Bytes)  │ (20 Bytes)        │ (TCP/UDP)    │ (Pad/NextHdr)  │ (ICV 12B)    │
└───────────────────┴──────────────┴────────────┴───────────────────┴──────────────┴────────────────┴──────────────┘
                    │◄──────────────────────── Encrypted Payload ─────────────────►│
                    │◄──────────────────────── Authenticated ──────────────────────────────────────►│

[ 2. GRE over IPsec - Tunnel Mode (二重 IP ヘッダーオーバーヘッドが発生) ]
┌───────────────────┬──────────────┬───────────────────┬────────────┬───────────────────┬──────────────┬─────────────┐
│ New IP Header     │ ESP Header   │ Outer IP Header   │ GRE Header │ Inner IP Header   │ Payload Data │ ESP Trailer │
│ (20 Bytes)        │ (8 Bytes)    │ (20 Bytes)        │ (4 Bytes)  │ (20 Bytes)        │ (TCP/UDP)    │ (Pad/Next)  │
└───────────────────┴──────────────┴───────────────────┴────────────┴───────────────────┴──────────────┴─────────────┘
                    │◄──────────────────────── Encrypted Payload ─────────────────────────►│
```

---

### 3. GRE Keepalive の動作フロー (送信元ルーティング・ループバック機構)

GRE トラフィックが発生しない状態でも、対向ノードのダウンやアンダーレイ障害による GRE インターフェイス状態（Up/Down）を検知するための固有メカニズム：

```text
[ Router A (Tunnel Source: 1.1.1.1) ] ─────────────────────────► [ Router B (Tunnel Dst: 2.2.2.2) ]
        │                                                                │
        ├── Generates Keepalive Packet:                                  │
        │     Outer IP Header: Src=1.1.1.1, Dst=2.2.2.2 (Protocol 47)    │
        │     GRE Header 1: Protocol = 0x0800                            │
        │     Inner IP Header: Src=2.2.2.2, Dst=1.1.1.1 (Reversed!)       │
        │     GRE Header 2: Protocol = 0x0000 (Keepalive Response)       │
        │                                                                │
        │───────────── Send Outer Packets via Underlay ─────────────────►│
        │                                                                │
        │                                                                ├── Receives GRE Packet.
        │                                                                ├── Decapsulates Outer IP & GRE 1.
        │                                                                ├── Sees Inner IP Header: Dst=1.1.1.1.
        │                                                                ├── Router B simply ROUTES the Inner Packet
        │                                                                │   back to Router A (No GRE processing needed on B!).
        │◄──────────── Inner Packet Transmitted back to Router A ────────┤
        │                                                                │
        ├── Router A receives its own inner packet (GRE 2 response).     │
        └── Keepalive counter reset -> Tunnel0 remains UP/UP.            │
```

---

## ⚙ 動作シーケンス

1. **パケット生成 ＆ トンネルインターフェイス着信**:
   * ホスト A (10.1.1.10) が ホスト B (10.2.2.20) へ TCP パケットを送信。
   * ルーター A のルーティングテーブルで、10.2.2.0/24 へのネクストホップが `Tunnel0` インターフェイスにヒット。
2. **GRE カプセル化処理**:
   * ルーター A は元の IP パケット（Inner IP Header）の先頭に GRE ヘッダー（Protocol Type = `0x0800` [IPv4]）を付加。
   * インターフェイスの `tunnel source` (203.0.113.1) および `tunnel destination` (198.51.100.1) に基づいて、新しい Outer IP Header（IP Protocol = 47）を付加。
3. **IPsec 処理 (GRE over IPsec 構成時)**:
   * ルーター A の Crypto Map または IPsec Profile (Protection) が発動。
   * Outer IP Header (203.0.113.1 ➔ 198.51.100.1) の Protocol 47 パケットに対し、IPsec ESP トランスフォーム（AES-GCM-256 等）を適用し、暗号化 Outer パケットを物理インターフェイスへ転送。
4. **アンダーレイ IP 網経由の転送**:
   * インターネット / サービスプロバイダ網のルーター群は、アウター IP ヘッダー (203.0.113.1 ➔ 198.51.100.1) のみに基づいて通常通りパケットをルーティング。
5. **GRE デカプセル化 ＆ 宛先転送**:
   * ルーター B (198.51.100.1) がパケットを受信し、IPsec 復号を実施。
   * GRE ヘッダーを取り外し、内部 IP パケット（Inner IP Header: 10.1.1.10 ➔ 10.2.2.20）を抽出。
   * ローカルルーティングテーブル（または Inside VRF）に従い、LAN 側の物理インターフェイス経由でホスト B へパケットを出力。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **GRE over IPsec における Transport Mode の指定 (必須最適化)**:
   * デフォルトの IPsec Transform Set は `mode tunnel` に設定されています。GRE トラフィックに対して Tunnel Mode IPsec を適用すると、Outer IP Header が二重に付加され無駄な 20 バイトのオーバーヘッドが発生します。
   * **要件**: ラボ試験で「Minimize IPsec encapsulation overhead for GRE traffic」と指定された場合、Transform Set 内で **`mode transport`** を明示指定する必要があります：
     * `crypto ipsec transform-set TS_GRE esp-aes 256 esp-sha256-hmac`
     * `mode transport`
2. **GRE Keepalive の仕組みと片方向設定**:
   * `keepalive <period> <retries>` コマンド（例: `keepalive 10 3`）を投入すると、ルーターは送信元と宛先を逆転させた二重 GRE ヘッダーパケットを生成します。
   * **重要機能**: 対向ルーター（ルーター B）側で `keepalive` コマンドが設定されていなくても、ルーター B がパケットを通常ルーティングでルーター A に返送できる状態であれば、ルーター A 側の Keepalive は正常に成立します（片側のみの設定でも動作可能）。
3. **VRF-Aware GRE (FVRF vs IVRF) の構築手順**:
   * アンダーレイ（物理 WAN ポート）が VRF `WAN_PROVIDER` に属し、オーバーレイ（社内通信）が Global Routing Table または VRF `INTERNAL` に属している場合：
     * インターフェイス設定:
       ```bash
       interface Tunnel0
        ip vrf forwarding INTERNAL       ! Inner VRF (IP パケットルーティング層)
        ip address 10.255.255.1 255.255.255.252
        tunnel source GigabitEthernet0/0
        tunnel destination 198.51.100.1
        tunnel vrf WAN_PROVIDER          ! Front-Door VRF (アンダーレイ層)
       ```
   * **注意**: `tunnel vrf` を指定し忘れると、トンネルルーターは Global Routing Table から `tunnel destination` を検索して失敗するか、アンダーレイ接続が確立しません。
4. **MTU ＆ TCP MSS Clamping 設定 (フラグメントによるドロップ防止)**:
   * GRE (24 バイト) ＋ IPsec ESP (約 56 バイト) のオーバーヘッドにより、通常の 1500 バイトパケットは 1580 バイト超となり、DF (Don't Fragment) ビットがセットされた TCP パケットが破棄されます。
   * **標準設定ルール**:
     * `interface Tunnel0` 内で:
       * `ip mtu 1400`
       * `ip tcp adjust-mss 1360` (MTU 1400 - 40 バイト IP/TCP ヘッダー)
5. **ASA / FTD における GRE パケットのバイパス ＆ インスペクションルール**:
   * ASA / FTD 経由で GRE パケットを通過させる場合、IP Protocol 47 が Access Control Policy (ACP) や ASA ACL で許可されていることを確認します。また、ASA の Modular Policy Framework (MPF) で GRE 状態チェックが走る場合、`inspect gre` またはステートフルパススルーの動作を理解しておく必要があります。

---

## 🛠 設定方法

### 1. Cisco IOS / IOS-XE: Point-to-Point GRE ＋ IPsec Protection (Transport Mode)

```bash
! ===== Step 1: IPsec Profile ＆ Transport Mode Transform Set の定義 =====
crypto ikev2 proposal PROP_IKEV2
 encryption aes-gcm-256
 group 19 20
 integrity sha256

crypto ikev2 policy POL_IKEV2
 proposal PROP_IKEV2

crypto ikev2 keyring KR_GRE
 peer SPOKE_HUB
  address 198.51.100.1
  pre-shared-key cisco123

crypto ikev2 profile IKE2_PROF_GRE
 match identity remote address 198.51.100.1 255.255.255.255
 identity local address 203.0.113.1
 authentication remote pre-share
 authentication local pre-share
 keyring local KR_GRE

crypto ipsec transform-set TS_GRE_TRANSPORT esp-aes 256 esp-sha256-hmac
 mode transport                         ! GRE over IPsec で必須の Transport Mode 指定

crypto ipsec profile IPSEC_PROF_GRE
 set transform-set TS_GRE_TRANSPORT
 set ikev2-profile IKE2_PROF_GRE

! ===== Step 2: GRE Tunnel インターフェイスの設定 =====
interface Tunnel0
 description GRE_OVER_IPSEC_TO_HQ
 ip address 10.255.255.1 255.255.255.252
 ip mtu 1400                            ! MTU 調整
 ip tcp adjust-mss 1360                 ! TCP MSS Clamping
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.1
 tunnel protection ipsec profile IPSEC_PROF_GRE  ! IPsec 暗号化のバインド
 keepalive 10 3                         ! GRE Keepalive (10秒間隔, 3回不達でDown)
```

---

### 2. Cisco IOS / IOS-XE: VRF-Aware GRE (Front-Door VRF [FVRF] ＋ Inside VRF [IVRF])

```bash
! ===== Step 1: Front-Door VRF (FVRF) ＆ Inside VRF (IVRF) の定義 =====
ip vrf FVRF_INTERNET
 description Underlay Transport VRF
ip vrf IVRF_CORP
 description Overlay Internal VRF

! ===== Step 2: 物理アンダーレイポートを FVRF へ割当 =====
interface GigabitEthernet0/0
 description WAN Physical Interface
 ip vrf forwarding FVRF_INTERNET
 ip address 203.0.113.1 255.255.255.0

! ===== Step 3: VRF-Aware GRE Tunnel インターフェイスの設定 =====
interface Tunnel0
 description VRF_AWARE_GRE_TUNNEL
 ip vrf forwarding IVRF_CORP            ! Inner VRF (ルーティング層)
 ip address 10.255.255.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.1
 tunnel vrf FVRF_INTERNET               ! Front-Door VRF (アンダーレイ検索層)
```

---

### 3. Cisco ASA CLI: Point-to-Point GRE トンネル設定

```bash
! ===== ASA GRE Tunnel Configuration =====
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 203.0.113.10 255.255.255.0

interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 10.1.1.1 255.255.255.0

! GRE トンネルインターフェイスの定義 (ASA 9.0+ 対応)
interface Tunnel1
 nameif gre_tunnel
 security-level 50
 ip address 10.255.255.1 255.255.255.252
 tunnel source interface outside
 tunnel destination 198.51.100.10
 tunnel mode gre ip

! トラフィック用スタティックルーティング
route gre_tunnel 10.2.0.0 255.255.255.0 10.255.255.2
```

---

### 4. Cisco FTD (FMC Managed): FlexConfig による GRE トンネルの追加

```text
[ FMC GUI Steps for GRE FlexConfig ]
1. Objects > Object Management > FlexConfig > FlexConfig Objects ➔ "Add FlexConfig Object":
   - Name: Flex_GRE_Tunnel
   - Deployment: Every Time
   - Insert Code:
     interface Tunnel100
      nameif gre_branch
      security-level 50
      ip address 10.255.255.1 255.255.255.252
      tunnel source GigabitEthernet0/0
      tunnel destination 198.51.100.1
      tunnel mode gre ip
     !
     route gre_branch 10.50.0.0 255.255.0.0 10.255.255.2
2. Devices > FlexConfig ➔ 対象 FTD の FlexConfig Policy 編集 ➔ "Flex_GRE_Tunnel" を追加して Deploy。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (IOS / ASA / FTD) | 表示確認ポイント |
| :--- | :--- | :--- |
| **GRE トンネルステータス確認** | <code>show interface Tunnel0</code> | `Tunnel0 is up, line protocol is up`, `Tunnel source/destination`, MTU/MSS 値 |
| **GRE トンネルサマリー表示** | <code>show ip interface brief \| inc Tunnel</code> | トンネルインターフェイスの IP 割当および UP/UP 状態 |
| **IPsec SA 結合状態の確認 (GRE over IPsec)** | <code>show crypto ipsec sa interface Tunnel0</code> | `inbound esp sas`, `outbound esp sas`, `mode: transport`, 暗号化/復号化パケットカウント |
| **IKEv2 セッション確認** | <code>show crypto ikev2 sa</code> | `Status: READY`, 対向ピア IP アドレス |
| **VRF マッピング確認 (FVRF/IVRF)** | <code>show vrf brief</code><br><code>show ip route vrf IVRF_CORP</code> | 物理ポートと Tunnel ポートの属する VRF 一覧およびオーバーレイ経路 |
| **ASA GRE インターフェイス確認** | <code>show interface tunnel 1</code> (ASA CLI) | ASA 上の GRE トンネル統計・パケットカウンター |
| **GRE トラフィックのデバッグ** | <code>debug tunnel</code><br><code>debug ip packet detail</code> | カプセル化/デカプセル化エラー、ミスマッチパラメータの追跡 |

---

## 🚨 トラ活（トラブルシューティング）

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **`Tunnel0 is up, line protocol is down` と表示され通電しない** | アンダーレイ IP 網で `tunnel destination` への L3 ルーティング経路が存在しない、または ACL で IP Protocol 47 がブロックされている | <code>ping <TUNNEL_DEST></code><br><code>show ip route <TUNNEL_DEST></code> | アンダーレイ経路（Default Route や Static Route）を確認し、中継ルーター/FW で **IP Protocol 47 (GRE)** を許可する。 |
| **GRE Over IPsec でトンネルは UP するが、内部で通信を開始するとトンネルがループ（Flapping）する** | トンネル内部の動的ルーティング（OSPF/EIGRP）で、`tunnel destination` のプレフィックスが GRE トンネル経由で学習されてしまう（Recursive Routing） | <code>show ip route</code><br><code>show log</code> (`RECURCONFIG` エラー) | 動的ルーティング（OSPF/EIGRP）の `network` 宣言からアンダーレイ物理 IP を除外するか、`tunnel destination` 宛ての /32 ホストルートを静的スタティック経路で明示固定する。 |
| **小サイズパケット (Ping) は通るが、Web (HTTP/HTTPS) や大容量通信がタイムアウトする** | GRE (24B) ＋ IPsec (56B) のオーバーヘッドによる **IP フラグメンテーション（MTU/MSS ミスマッチ）** | <code>ping <IP> size 1450 df-bit</code> | トンネルインターフェイス上で **`ip mtu 1400`** および **`ip tcp adjust-mss 1360`** を投入する。 |
| **GRE Keepalive を設定したのに、対向をシャットダウンしても Tunnel が Down にならない** | 送信元ルーターから送出された Keepalive パケット（内側 IP Header: Dst=自IP）が、アンダーレイ網でルーティングされずに破棄されている | <code>debug tunnel keepalive</code> | アンダーレイ網および対向ルーターで、内側 IP アドレス宛てのルーティングが成立していることを確認する。 |
| **VRF-Aware GRE で `tunnel vrf` を設定した途端にトンネルが Down する** | `tunnel source` で指定した物理ポートが属する VRF と、`tunnel vrf` で指定した FVRF 名が不一致 | <code>show vrf</code><br><code>show interface <PHYSICAL_IF></code> | 物理インターフェイスの `ip vrf forwarding <FVRF>` と `tunnel vrf <FVRF>` の記述名を完全に一致させる。 |

---

## ⚠ 制限事項

* **暗号化機能の欠如**:
  * GRE 単体ではパケットの平文カプセル化のみを行うため、盗聴や改ざん、なりすましを防止できません。セキュリティ要件のある境界通信では必ず **IPsec (Tunnel Protection / Crypto Map)** を併用する必要があります。
* **ASA における GRE 制限**:
  * ASA 上で作成した GRE トンネルインターフェイス上では、**OSPF / EIGRP などの動的ルーティングプロトコルは稼働できません**（スタティックルーティングまたは BGP のみサポート）。
* **マルチキャストおよび DiffServ/QoS コピー**:
  * トンネルカプセル化時、デフォルトでは Inner IP の QoS (ToS/DSCP) バイナリ値は Outer IP へコピーされますが、明示的な QoS ポリシーを適用する場合は `qos pre-classify` コマンドの設定が必要です。

---

## 🔄 他技術との関連

* **IPsec (ESP Protocol 50)**: GRE トラフィックに対し、暗号化（AES-GCM/CBC）および認証（HMAC-SHA2）を付与して安全な GRE over IPsec トンネルを構築。
* **DMVPN (Dynamic Multipoint VPN)**: mGRE (Multipoint GRE) ＋ NHRP ＋ IPsec を組み合わせ、拠点の動的フルメッシュ接続を実現するアーキテクチャ。
* **FlexVPN**: Cisco 統合 IKEv2 / IPsec フレームワーク。mGRE / VTI (Virtual Tunnel Interface) を使用して Point-to-Point および Multipoint リモートアクセス・S2S VPN を統一構築。
* **VRF-Lite / MPLS**: Front-Door VRF (FVRF) と Inside VRF (IVRF) を分離し、アンダーレイのトランスポート層とオーバーレイの顧客ネットワーク層をセグメンテーション。

---

## 🧩 比較表

### 1. GRE vs IPsec VTI vs DMVPN (mGRE)

| 比較項目 | GRE (Point-to-Point) | IPsec VTI (Virtual Tunnel Interface) | DMVPN (mGRE) |
| :--- | :--- | :--- | :--- |
| **トポロジー** | **Point-to-Point (1対1)** | **Point-to-Point (1対1)** | **Point-to-Multipoint (1対多/フルメッシュ)** |
| **プロトコル** | IP Protocol 47 | IP Protocol 50 (ESP Native) | mGRE ＋ NHRP ＋ IPsec |
| **マルチキャスト伝送** | **対応** | **対応** | **対応** |
| **暗号化機能** | なし（IPsec Protection が別途必要） | **標準組み込み**（IPsec 統合） | **標準組み込み**（IPsec Profile） |
| **ヘッダーオーバーヘッド**| 約 24 バイト (GRE) ＋ IPsec | 約 52～72 バイト (ESP のみ) | 約 24 バイト (mGRE) ＋ IPsec |
| **動的メッシュ接続** | 非対応（固定 1対1） | 非対応（固定 1対1） | **対応**（NHRP による Spoke-to-Spoke 確立） |

---

## 💡 ベストプラクティス

1. **GRE over IPsec での Transport Mode 明示指定**:
   * 不要な 20 バイトの Outer IP ヘッダー重複を排除するため、IPsec Transform Set では必ず `mode transport` を使用します。
2. **MTU 1400 / TCP MSS 1360 の一律適用**:
   * 全ての GRE トンネルインターフェイスにおいて、`ip mtu 1400` と `ip tcp adjust-mss 1360` を適用し、Path MTU Discovery (PMTUD) ドロップ障害を予防します。
3. **Recursive Routing 対策の徹底**:
   * OSPF / EIGRP の `network` 設定において、アンダーレイ物理 IP (`tunnel source` / `tunnel destination`) が誤ってトンネル経由で広報されないよう、フィルタリング（Distribute-list / Prefix-list）または明示的な /32 スタティック経路を常設します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Cisco IOS Point-to-Point GRE over IPsec (IKEv2 / Transport Mode)

#### 問題
Router A (HQ: `203.0.113.1`) と Router B (Branch: `198.51.100.1`) の間で、IKEv2 事前共有鍵 (`cisco123`) および IPsec Transport Mode を使用した GRE over IPsec トンネル (`Tunnel0`) を構築せよ。

#### 要件
* Tunnel Subnet: `10.255.255.0/30` (Router A: `.1`, Router B: `.2`)
* IPsec Transform Set は `mode transport` を明示指定すること。
* MTU を `1400`、TCP MSS を `1360` に調整すること。

#### 設定例

```bash
! ===== Router A (HQ) Configuration =====
crypto ikev2 proposal PROP_AES256
 encryption aes-gcm-256
 group 19
 integrity sha256

crypto ikev2 policy POL_GRE
 proposal PROP_AES256

crypto ikev2 keyring KR_BRANCH
 peer BRANCH
  address 198.51.100.1
  pre-shared-key cisco123

crypto ikev2 profile PROF_GRE
 match identity remote address 198.51.100.1 255.255.255.255
 identity local address 203.0.113.1
 authentication remote pre-share
 authentication local pre-share
 keyring local KR_BRANCH

crypto ipsec transform-set TS_GRE_TRANS esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile IPSEC_PROF_GRE
 set transform-set TS_GRE_TRANS
 set ikev2-profile PROF_GRE

interface Tunnel0
 description GRE_OVER_IPSEC_TO_BRANCH
 ip address 10.255.255.1 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.1
 tunnel protection ipsec profile IPSEC_PROF_GRE

! ===== Router B (Branch) Configuration =====
crypto ikev2 proposal PROP_AES256
 encryption aes-gcm-256
 group 19
 integrity sha256

crypto ikev2 policy POL_GRE
 proposal PROP_AES256

crypto ikev2 keyring KR_HQ
 peer HQ
  address 203.0.113.1
  pre-shared-key cisco123

crypto ikev2 profile PROF_GRE
 match identity remote address 203.0.113.1 255.255.255.255
 identity local address 198.51.100.1
 authentication remote pre-share
 authentication local pre-share
 keyring local KR_HQ

crypto ipsec transform-set TS_GRE_TRANS esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile IPSEC_PROF_GRE
 set transform-set TS_GRE_TRANS
 set ikev2-profile PROF_GRE

interface Tunnel0
 description GRE_OVER_IPSEC_TO_HQ
 ip address 10.255.255.2 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source GigabitEthernet0/0
 tunnel destination 203.0.113.1
 tunnel protection ipsec profile IPSEC_PROF_GRE
```

#### 検証方法
* Router A で `show interface Tunnel0` を実行し `Tunnel0 is up, line protocol is up` を確認。
* `show crypto ipsec sa interface Tunnel0` を実行し、`mode: transport` および暗号化パケットの増加を確認する。

---

### 例2: VRF-Aware GRE (FVRF トランスポート ＋ IVRF 内部ルーティング)

#### 問題
Router A において、物理ポート `Gi0/0` をアンダーレイ VRF `FVRF_WAN` にアサインし、GRE トンネル内部のトラフィックをオーバーレイ VRF `IVRF_CORP` に独立分離してルーティングせよ。

#### 要件
* アンダーレイ IP: `203.0.113.1/24`, 対向 IP: `198.51.100.1`
* トンネル IP: `10.255.255.1/30`
* `tunnel vrf FVRF_WAN` および `ip vrf forwarding IVRF_CORP` を適用すること。

#### 設定例

```bash
! ===== Router A Configuration =====
ip vrf FVRF_WAN
 description Underlay Transport
ip vrf IVRF_CORP
 description Overlay Corporate

interface GigabitEthernet0/0
 description WAN Interface
 ip vrf forwarding FVRF_WAN
 ip address 203.0.113.1 255.255.255.0

ip route vrf FVRF_WAN 0.0.0.0 0.0.0.0 203.0.113.254

interface Tunnel0
 description VRF_AWARE_GRE
 ip vrf forwarding IVRF_CORP
 ip address 10.255.255.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.1
 tunnel vrf FVRF_WAN
```

#### 検証方法
* `show vrf brief` を実行し、`Gi0/0` が `FVRF_WAN`、`Tunnel0` が `IVRF_CORP` にアサインされていることを確認。
* `ping vrf IVRF_CORP 10.255.255.2` を実行し、トンネル内対向 IP への導通を確認する。

---

### 例3: GRE Keepalive による片方向リンク障害検出

#### 問題
Router A から Router B 宛ての GRE トンネル (`Tunnel0`) に対し、10秒間隔・3回不達（30秒）でトンネルを Down 状態へ遷移させる GRE Keepalive を構成せよ。

#### 要件
* `keepalive 10 3` コマンドを使用すること。

#### 設定例

```bash
! ===== Router A Configuration =====
interface Tunnel0
 ip address 10.255.255.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.1
 keepalive 10 3
```

#### 検証方法
* Router B 側でインターフェイスをシャットダウン後、Router A で `show interface Tunnel0` を実行し、30秒後に `line protocol is down (monitoring)` に変化することを確認。

---

### 例4: GRE トンネル経由の OSPFv2 動的ルーティングの構成

#### 問題
Router A と Router B 間の GRE トンネル (`Tunnel0`: `10.255.255.0/30`) 上で OSPFv2 Area 0 を動作させ、各ローカル LAN ルート (`10.1.0.0/16` ↔ `10.2.0.0/16`) を自動広報せよ。

#### 要件
* GRE トンネル上で OSPF Hello パケット（マルチキャスト `224.0.0.5`）を通すこと。

#### 設定例

```bash
! ===== Router A Configuration =====
interface Tunnel0
 ip address 10.255.255.1 255.255.255.252
 ip ospf 1 area 0

interface GigabitEthernet0/1
 ip address 10.1.1.1 255.255.255.0
 ip ospf 1 area 0

router ospf 1
 router-id 1.1.1.1
```

#### 検証方法
* `show ip ospf neighbor` を実行し、`Tunnel0` 経由で対向ルーターが `FULL/BDR` または `FULL/DR` になっていることを確認。

---

### 例5: ASA 9.x CLI での GRE トンネル構成と静的ルート

#### 問題
ASA1 (`outside`: `203.0.113.10`) において、対向ルーター (`198.51.100.10`) 宛ての GRE トンネル `Tunnel1` を作成し、リモート LAN 宛て (`10.50.0.0/16`) のスタティックルートを設定せよ。

#### 要件
* `tunnel mode gre ip` を設定すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
interface Tunnel1
 nameif gre_hq
 security-level 50
 ip address 10.255.255.1 255.255.255.252
 tunnel source interface outside
 tunnel destination 198.51.100.10
 tunnel mode gre ip

route gre_hq 10.50.0.0 255.255.0.0 10.255.255.2
```

#### 検証方法
* ASA CLI で `show interface tunnel 1` を実行し、GRE トンネルステータスおよび送受信パケット数を確認する。

---

### 例6: GRE Over IPsec での Path MTU / TCP MSS 調整

#### 問題
GRE over IPsec トンネル経由の Web 通信でパケット破棄が発生しないよう、`Tunnel0` インターフェイスの MTU を `1400`、TCP MSS を `1360` に設定せよ。

#### 要件
* `ip mtu 1400` および `ip tcp adjust-mss 1360` コマンドを適用すること。

#### 設定例

```bash
! ===== Router A Configuration =====
interface Tunnel0
 ip mtu 1400
 ip tcp adjust-mss 1360
```

#### 検証方法
* クライアント端末から `ping 10.2.2.20 -f -l 1372` (ICMP パケット 1400B) を送信し、断片化されずに応答が返ることを確認。

---

### 例7: Multipoint GRE (mGRE) インターフェイスの基本設定 (DMVPN 準備)

#### 問題
Hub ルーターにおいて、複数の Spoke ルーターからの動的接続を受け入れる Multipoint GRE (mGRE) インターフェイス `Tunnel0` を設定せよ。

#### 要件
* `tunnel mode gre multipoint` を指定すること。

#### 設定例

```bash
! ===== Hub Router Configuration =====
interface Tunnel0
 description mGRE_HUB_INTERFACE
 ip address 10.255.255.1 255.255.255.0
 no ip split-horizon egrp 100
 ip nhrp authentication cisco_nhrp
 ip nhrp network-id 100
 ip nhrp redirect
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
```

#### 検証方法
* `show interface Tunnel0` を実行し、`Header encapsulation NO_TUNNEL_DESTINATION`, `Tunnel mode gre multipoint` と表示されることを確認。

---

### 例8: GRE トンネルでの Key および Sequence Number の付加

#### 問題
GRE ヘッダー内に識別キー `12345` およびパケット順序制御用シーケンス番号を付加してカプセル化セキュリティを高めよ。

#### 要件
* `tunnel key` および `tunnel sequence-datagrams` を設定すること。

#### 設定例

```bash
! ===== Router A Configuration =====
interface Tunnel0
 tunnel key 12345
 tunnel sequence-datagrams
```

#### 検証方法
* `show interface Tunnel0` を実行し、`Tunnel key 12345`, `Tunnel sequence datagrams enabled` が表示されていることを確認。

---

### 例9: IPv6 over IPv4 GRE トンネルの構築

#### 問題
IPv4 アンダーレイ ネットワーク（`203.0.113.1` ↔ `198.51.100.1`）を経由して、両拠点の IPv6 トラフィック (`2001:db8:1::/64` ↔ `2001:db8:2::/64`) をカプセル化伝送する GRE トンネルを構築せよ。

#### 要件
* トンネル上で IPv6 アドレス (`2001:db8:ffff::1/64`) を割り当てること。

#### 設定例

```bash
! ===== Router A Configuration =====
ipv6 unicast-routing

interface Tunnel0
 description GRE_IPV6_OVER_IPV4
 no ip address
 ipv6 address 2001:DB8:FFFF::1/64
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.1
 tunnel mode gre ip                      ! IPv4 アウターヘッダー内に IPv6 をカプセル化

ipv6 route 2001:DB8:2::/64 Tunnel0 2001:DB8:FFFF::2
```

#### 検証方法
* `ping ipv6 2001:DB8:FFFF::2` および `ping ipv6 2001:DB8:2::1` を実行し、IPv6 トラフィックが正常にカプセル化通過することを確認。

---

### 例10: FMC FlexConfig による FTD への Point-to-Point GRE 追加

#### 問題
FMC 管理下の FTD において、FlexConfig オブジェクトを使用して対向ルーター宛ての GRE トンネル `Tunnel10` を追加配置せよ。

#### 要件
* Tunnel IP: `10.255.255.1/30`, Remote IP: `198.51.100.1`

#### 設定例

```text
[ FMC FlexConfig Script ]
interface Tunnel10
 nameif gre_site2
 security-level 50
 ip address 10.255.255.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.1
 tunnel mode gre ip
!
route gre_site2 10.100.0.0 255.255.0.0 10.255.255.2
```

#### 検証方法
* FTD Diagnostic CLI で `show interface tunnel 10` を実行し、GRE インターフェイスが生成されていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 以下のコンフィグが設定されたルーターの動作に関する説明として正しいものはどれか。

```bash
crypto ipsec transform-set TS_GRE esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile IPSEC_PROF
 set transform-set TS_GRE

interface Tunnel0
 ip address 10.255.255.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.1
 tunnel protection ipsec profile IPSEC_PROF
```

* A) GRE パケットに対し IPsec Tunnel Mode が適用され、アウター IP ヘッダーが 2 重に付加される。
* B) GRE パケットに対し IPsec Transport Mode が適用され、冗長なアウター IP ヘッダーを追加することなく GRE ヘッダー直前で ESP 暗号化が行われる。
* C) GRE トンネルは暗号化されず、平文のまま送出される。
* D) `tunnel protection` は Multipoint GRE (mGRE) 専用コマンドのためエラーとなる。

**正解**: **B**  
**解説**: Transform Set 内で `mode transport` が明示指定されており、GRE のアウター IP ヘッダーをそのまま IPsec のアウター IP ヘッダーとして再利用するため、二重 IP ヘッダーによるオーバーヘッドを発生させずに ESP 暗号化を実行します。

---

### 2. トラブルシュート問題

**問題**: Router A と Router B の間で GRE トンネル経由の OSPFv2 ネイバーを確立した。しかし、トンネルが確立した直後に OSPF ログに `RECURCONFIG` または Recursive Routing エラーが頻発し、OSPF ネイバーが DOWN/UP を繰り返してループ（Flapping）状態に陥った。原因として最も適切なものはどれか。

* A) GRE Keepalive のインターバルが短すぎるため。
* B) `tunnel destination` で指定されている物理 IP アドレスへのプレフィックスが、OSPF によって GRE トンネル経由で学習されてしまったため。
* C) GRE トンネル上で MTU サイズが大きすぎるため。
* D) OSPF の Router-ID が重複しているため。

**正解**: **B**  
**解説**: OSPF が GRE トンネル経由で `tunnel destination` 宛てのプレフィックスを学習すると、ルーターは「トンネル宛てパケットをトンネル内にカプセル化して送る」という再帰ループ（Recursive Routing）に陥り、トンネルが自己破壊（Down）と再確立（Up）を繰り返します。

---

### 3. Design 問題

**問題**: エンタープライズ WAN の設計において、本社ルーターと拠点ルーター間で動的ルーティングプロトコル (OSPF/EIGRP) のマルチキャストパケットを透過伝送しつつ、パケット暗号化オーバーヘッドを最小化したい。最も推奨される構成はどれか。

* A) IPsec VTI (Tunnel Mode)
* B) Point-to-Point GRE ＋ IPsec (Transport Mode)
* C) Raw GRE (暗号化なし)
* D) Pure IPsec L2L Tunnel (Crypto Map)

**正解**: **B**  
**解説**: GRE を使用することでマルチキャストパケットの通過が可能になり、さらに IPsec を **Transport Mode** で結合することで、二重 IP ヘッダーを発生させずオーバーヘッドを最小化した安全な動的ルーティング伝送が実現します。

---

### 4. 実装問題

**問題**: VRF-Aware GRE を構築する際、アンダーレイのトランスポート層（物理ポート）が属する VRF 名を指定するために `interface Tunnel` 内で設定すべきコマンドはどれか。

* A) `ip vrf forwarding <VRF>`
* B) `tunnel vrf <VRF>`
* C) `ip vrf receive <VRF>`
* D) `crypto vrf <VRF>`

**正解**: **B**  
**解説**: `tunnel vrf <VRF>` コマンドは、GRE カプセル化パケットを送出する際のアウター IP ヘッダー（`tunnel source` / `tunnel destination`）を検索するアンダーレイ (Front-Door VRF) を指定します。`ip vrf forwarding` はトンネル内部のインナー IP (Inside VRF) を指定します。

---

### 5. コンフィグ読解問題

**問題**: Cisco ASA 9.x において、GRE トンネルを構成する際のシステム上の制限事項として正しいものはどれか。

* A) ASA では GRE トンネルを一切サポートしない。
* B) ASA の GRE トンネル上では OSPF / EIGRP などの動的ルーティングプロトコルを動かすことができない。
* C) ASA では GRE over IPsec 構成が不可能ですべて平文伝送となる。
* D) ASA では GRE トンネルの宛先にドメイン名 (FQDN) しか指定できない。

**正解**: **B**  
**解説**: Cisco ASA は ASA 9.0(1) 以降で Point-to-Point GRE トンネルをサポートしますが、**GRE インターフェイス上での動的ルーティングプロトコル（OSPF / EIGRP）の動作はサポートされていません**（スタティックルーティングまたは BGP のみ設定可能）。

---

## 🔗 参考リソース

* [Cisco IOS Flexible Internal Encapsulation Configuration Guide: Generic Routing Encapsulation (GRE)](https://www.cisco.com/)
* [Cisco ASA Series General Operations Configuration Guide: GRE Tunnels](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Device Configuration Guide: FlexConfig Policies](https://www.cisco.com/)
* [Cisco Technical Notes: Troubleshoot GRE Tunnel Flapping and Recursive Routing Issues](https://www.cisco.com/)
* [Cisco Live: Advanced Site-to-Site VPN Design and Troubleshooting with GRE and IPsec (BRKSEC-3033)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で GRE が出題された場合の構成判断フローチャート：
    * **「Encapsulate multicast or non-IP traffic over IPsec」** ➔ Point-to-Point GRE ＋ IPsec Protection。
    * **「Minimize IPsec header overhead for GRE」** ➔ `mode transport` を Transform Set 内で指定。
    * **「Separate transport routing table from internal corporate routing」** ➔ VRF-Aware GRE (`tunnel vrf <FVRF>` ＋ `ip vrf forwarding <IVRF>`)。
    * **「Prevent TCP fragmentation/packet loss over GRE」** ➔ `ip mtu 1400` ＋ `ip tcp adjust-mss 1360`。
* **図解（VRF-Aware GRE トラフィック分離）**:
  ```text
  [ Corporate LAN (IVRF_CORP) ]
               │
               ▼
  [ Inner IP Packet (Src: 10.1.1.10, Dst: 10.2.2.20) ]
               │
               ▼  (Routed in IVRF_CORP -> Tunnel0)
  [ GRE Encapsulation Engine ]
               │
               ▼  (Encapsulates Outer Header using FVRF_WAN)
  [ Outer IP Packet (Src: 203.0.113.1, Dst: 198.51.100.1, Protocol: 47) ]
               │
               ▼  (Forwarded out via Physical Gi0/0 in FVRF_WAN)
  [ Public Internet / Provider Underlay ]
  ```
* **注意点**:
  * ラボ試験で GRE を構成した後は、必ず `show interface Tunnel0`、`show crypto ipsec sa`、および `ping`（大サイズ ＋ DF ビット付き）を実行し、パケットドロップやフラッピングが発生していないことを確認してください。


## 📝 **補足（Notes）**

*   **学習メモ**: 「GRE はただの筒（筒自体に鍵はない）」とイメージしてください。鍵（暗号化）をかけるのが IPsec の役割です。
*   **図解**: パケットがルータを通過するたびに、ヘッダーが「着替え」をしていく様子（Encapsulation/Decapsulation）を紙に書いて整理しましょう。
*   **注意点**: ラボ試験では `tunnel source` に指定した IP アドレスに到達性がない場合、インターフェイスは `up/down` になります。まず下位レイヤの導通を優先してください。
