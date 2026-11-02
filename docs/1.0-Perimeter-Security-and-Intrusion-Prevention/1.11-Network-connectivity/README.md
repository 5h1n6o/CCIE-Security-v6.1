---
layout: default
title: 1.11-Network-connectivity
nav_order: 11
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.11 Network connectivity through Cisco ASA and Cisco FTD

Cisco ネットワークセキュリティ環境における **Network Connectivity（ネットワーク接続性）** は、Cisco ASA（Adaptive Security Appliance）および Cisco FTD（Firepower Threat Defense / Secure Firewall）がネットワークインフラ内でトラフィックを受信・通過・送信・監視するための物理および論理インターフェイスアーキテクチャです。

CCIE Security v6.1 Blueprint **1.11 Network connectivity through Cisco ASA and Cisco FTD** では、**Routed Mode (L3)** と **Transparent Mode (L2)**、**Integrated Routing and Bridging (IRB)**、物理インターフェイス、**802.1Q Subinterfaces**、**Redundant Interfaces**、**EtherChannel (LACP/Static)**、**Bridge Groups / BVI (Bridge Virtual Interface)**、FTD 独自の IPS インターフェイスモード（**Inline Set**、**Inline Tap Mode**、**Passive Mode**、**ERSPAN Passive Mode**）、**Link State Propagation**、**Hardware Bypass**、**IPv4/IPv6 デュアルスタック接続 (SLAAC, DHCPv4/v6, ICMPv6 Inspection)**、**Proxy ARP / MAC テーブル処理**、**MTU / ジャンボフレーム / TCP MSS Clamping**、ならびに **Virtual Routers (VRF-Lite)** および **FXOS Multi-instance** に至る接続仕様の実装・検証・トラブルシューティング能力が多角的に評価されます。

---

## 📘 概要

* **機能概要**:
  * **デプロイメントモード (Firewall Operating Modes)**:
    * **Routed Mode (Layer 3)**: ファイアウォールがネットワーク上のルーターホップとして動作し、各インターフェイスが異なる IP サブネットに所属する標準構成。
    * **Transparent Mode (Layer 2)**: ファイアウォールが Layer 2 の「ステルスファイアウォール（Bump in the Wire）」として動作し、同一 IP サブネット内の VLAN 間をブリッジングしながらステートフルアクセス制御を実施する。
    * **Integrated Routing and Bridging (IRB)**: FTD/ASA の Routed Mode 内でブリッジグループ (BVI) を定義し、Layer 2 ブリッジングと Layer 3 ルーティングを同一機器内で統合処理する高度な接続モード。
  * **FTD 固有の IPS インターフェイスモード**:
    * **Inline Set**: 2 つのインターフェイスをインラインペアとして結合し、パケットをインラインで通過させながら Snort エンジンでディープパケットインスペクション (DPI) を実施する。**Tap Mode** を有効にすると、インライン接続を維持したままコピーパケットのみを検査してドロップ動作を無効化（パッシブ化）できる。
    * **Passive / ERSPAN Passive Mode**: スイッチの SPAN / RSPAN / ERSPAN（GRE トンネル）から複製されたパケットを受信して脅威検知のみを行う（インラインドロップ不可）。※ERSPAN は Routed Mode のみでサポートされる。
  * **インターフェイスタイプ ＆ カプセル化**:
    * **Physical, Subinterfaces (802.1Q VLAN Tagging), Redundant Interfaces (Active/Standby ペア), EtherChannel (Port-Channel)** をサポート。
    * **MTU ＆ Jumbo Frames**: 最大 9000 バイト（プラットフォームにより 9184/9198 バイト）のジャンボフレームをサポート。
    * **TCP MSS Clamping**: GRE、IPsec VPN、VLAN タグなどのヘッダー追加による TCP パケットの断片化（Fragmentation）を防止するため、TCP SYN パケット内の MSS 値を最適値へ制限（Clamp）する。
* **利用目的**:
  * IP アドレス体系を変更することなく、既存ネットワークに透過的にファイアウォール/IPS を挿入する（Transparent Mode / Inline Set）。
  * 物理ポートの冗長化（Redundant / EtherChannel）および 802.1Q トランキングによる高密度セグメンテーション（Subinterfaces）を実現する。
  * IPv6 ネットワークへの移行に伴うデュアルスタック接続、SLAAC / DHCPv6 端末のアドレス割り当ておよび ICMPv6 PMTUD の維持。
* **どのような場面で利用するか**:
  * データセンターコア / DMZ 境界での L2 透過型セキュリティ挿入。
  * エンタープライズ WAN / インターネット境界での L3 ルーテッド接続および NAT / VPN 終端。
  * アウトオブバンド脅威監視環境における SPAN / ERSPAN パッシブセンシング。

---

## 🔑 要点

| 項目 | Cisco ASA Network Connectivity | Cisco FTD Network Connectivity |
| :--- | :--- | :--- |
| **デプロイモード** | Routed Mode, Transparent Mode (SMTF / MMTF) | Routed Mode, Transparent Mode, **IPS-Only Mode** |
| **L2 ブリッジ構造** | **Bridge Group** ＋ **BVI (Bridge Virtual Interface)** (最大 250 グループ) | **Bridge Group** ＋ **BVI** (最大 250 グループ) |
| **IPS インターフェイス** | ASA FirePOWER モジュール経由 | **Inline Set (Tap Mode/Link State Prop/HW Bypass)**, **Passive**, **ERSPAN** |
| **冗長構成** | **Redundant Interface** (Active/Standby ペア), **EtherChannel** | **Redundant Interface** (ASA プラットフォーム/FDM/FMC), **EtherChannel** |
| **VLAN サブインターフェイス**| 802.1Q Tagging (`interface <IF>.<ID>`) | 802.1Q Tagging (FMC アプリケーション定義 / **FXOS シャーシ定義**) |
| **Jumbo Frames 要件** | `mtu` 変更後、**アプライアンスの再起動 (Reload) が必須** | FMC から `MTU > 1500` 設定後 Deploy (ASA ベースモデルは再起動必須) |
| **TCP MSS 制御** | `sysopt connection tcpmss <bytes>` | FMC Interface Settings ➔ TCP MSS Clamping |
| **仮想ルーター (VRF)** | **Security Contexts** (Multi-Context Mode) | **Virtual Routers (VRF-Lite)**, **FXOS Multi-instance** |
| **IPv6 サポート** | Dual-stack, Static, SLAAC, DHCPv6, ICMPv6 Inspection | Dual-stack, Static, SLAAC, DHCPv6, ICMPv6 Inspection |

---

## 🏗 動作原理

### 1. FTD インターフェイスアーキテクチャとパケット処理モデル

FTD では、要求されるセキュリティレベルやネットワークトポロジに応じて、異なるインターフェイス処理パイプラインが適用されます。

```text
[ Ingress Packet ]
       │
       ├────────────────────────────────────────────────────────────────────────┐
       │                                                                        │
       ▼ (Firewall-Type Interfaces: Routed / Transparent BVI)                   ▼ (IPS-Only Interfaces: Inline Set / Passive)
┌────────────────────────────────────────────────────────┐             ┌────────────────────────────────────────────────────────┐
│ LINA Data Plane Processing                             │             │ IPS-Only Data Plane Processing                         │
│                                                        │             │                                                        │
│ 1. L2 / L3 Un-NAT, Stateful Conn Table Lookup          │             │ 1. Bypass Traditional Firewall Checks                  │
│ 2. Security Intelligence (SI) Check                    │             │    (No Stateful Conn Table, No NAT, No Routing)       │
│ 3. Access Control Policy (ACP) Rules Evaluation        │             │                                                        │
│ 4. Route Lookup / MAC Table Lookup (BVI)               │             │ 2. Direct Hand-off to Snort 3 Engine                   │
│                                                        │             │    ├── Inspects Intrusion Rules / Malware / File       │
│ 5. Hand-off to Snort 3 Engine for DPI (If Allowed)     │             │    └── Inline Pair: Transmit / Drop                    │
│                                                        │             │        Tap Mode: Inspect Copy & Always Transmit        │
│ 6. LINA Post-Processing (QoS / NAT / Egress Rewrite)   │             │        Link State Propagation: Sync Link Status        │
└────────────────────────────────────────────────────────┘             └────────────────────────────────────────────────────────┘
       │                                                                        │
       ▼                                                                        ▼
[ Egress Interface ]                                                     [ Egress Interface / Dropped ]
```

---

### 2. Transparent Mode ＆ Integrated Routing and Bridging (IRB) トラフィックフロー

Routed Mode 内で Bridge Group (BVI) を定義し、Layer 2 ブリッジングと Layer 3 ルーティングを統合処理する IRB の通信フロー：

```text
[ Client A (10.1.1.10) ]                [ Server B (10.1.1.20) ]             [ Server C (10.2.2.30) ]
          │                                        │                                    │
          │ (VLAN 10)                              │ (VLAN 10)                          │ (VLAN 20)
          ▼                                        ▼                                    ▼
┌────────────────────────────────────────────────────────────────────────────────────────────────┐
│ FTD / ASA Firewall (Integrated Routing and Bridging: IRB Enabled)                              │
│                                                                                                │
│  ┌───────────────────────────────────────────────────┐      ┌───────────────────────────────┐  │
│  │ Bridge Group 1 (VLAN 10 Subnet: 10.1.1.0/24)      │      │ Routed Interface (Outside)    │  │
│  │                                                   │      │ (Subnet: 10.2.2.0/24)         │  │
│  │ Member IF: Gi0/1 (Inside1), Gi0/2 (Inside2)       │      │                               │  │
│  │ Gateway:  BVI 1 (IP: 10.1.1.1/24)                 │      │ Interface: Gi0/0 (Outside)    │  │
│  └─────────────────────────┬─────────────────────────┘      │ IP Address: 10.2.2.1/24       │  │
│                            │                                └──────────────┬────────────────┘  │
│                            │                                               │                   │
│                            └─────────────── Internal L3 Router ────────────┘                   │
│                                            (Routes between BVI 1 and Outside)                  │
└────────────────────────────────────────────────────────────────────────────────────────────────┘

1. Client A ➔ Server B (Intra-Bridge Group L2 Communication):
   - MAC Destination Lookup in Bridge Group 1 Table.
   - Passes through Gi0/1 ➔ Gi0/2 without TTL decrement (Layer 2 Inspection).

2. Client A ➔ Server C (Inter-Subnet L3 Routed Communication):
   - Client A sends packet to Default Gateway MAC (BVI 1 MAC: 10.1.1.1).
   - FTD terminates L2 frame at BVI 1, performs L3 Route Lookup, decrements TTL, and forwards out Gi0/0.
```

---

## ⚙ 動作シーケンス

1. **Layer 2 MAC アドレス学習と転送シーケンス (Transparent Mode / Bridge Group)**:
   * インターフェイスに着信したパケットの L2 イーサネットヘッダー（Source MAC）を読み取り、Bridge Group 固有の MAC アドレスハッシュテーブル（`show mac-address-table`）に「MAC - 物理ポート」マッピングを自動登録。
   * 宛先 MAC アドレスが既知の場合、対象ポートへ直接フレームをフォワード。未知（Unknown Unicast / Broadcast / Multicast）の場合、送信元以外の Bridge Group メンバーポートへフラッディング。
2. **Proxy ARP 応答シーケンス (Static NAT / Mapped Addresses)**:
   * 外部ルーターが Static NAT 宛先（Mapped IP: `209.165.201.10`）へパケットを送信するため ARP Request を送出。
   * ASA / FTD はデータプレーン上で Proxy ARP エンジンを起動しており、NAT テーブルに適合する Mapped IP に対する ARP Request を検知すると、自身のインターフェイス MAC アドレスを Source MAC とした ARP Reply を即座に代行返答。
3. **FTD Inline Set ＆ Link State Propagation 動作シーケンス**:
   * Inline Set ペア (`Gi0/1` – `Gi0/2`) 上で `Link State Propagation` が有効化されている環境。
   * 上流スイッチの障害により `Gi0/1` の物理リンクが Down。
   * FTD (LINA / FXOS) は最大 4 秒以内にリンク状態の変化をセンシングし、対となる `Gi0/2` のリンク状態をプログラム的に Down 状態へ変更。
   * 隣接の動的ルーティングルーターは対向リンクの Down を検知し、即座に予備経路（Reroute）へ切替を実行。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Transparent Mode / Bridge Group (BVI) の制約事項と設計要件**:
   * **BVI IP アドレスの要件**: IPv4/IPv6 管理通信およびデータ通過のため、BVI には接続サブネットと同一の IP アドレスが必須です。ただし、**/30 や host (/32) サブネットは使用不可**（最小 3 つのアドレス範囲が必要）。また、対向機器のデフォルトゲートウェイに BVI IP アドレスを指定してはいけません（対向ルーターの IP を指定する）。
   * **BVI の Naming 挙動 (Routed Mode IRB)**:
     * **BVI に Name (`nameif`) を指定した場合**: BVI が L3 インターフェイスとして動作し、他の Routed インターフェイスや Bridge Group との間でルーティング可能になる。
     * **BVI に Name を指定しない場合**: Bridge Group は他から完全孤立（Isolate）され、純粋な Layer 2 ブリッジとして動作する。
   * **BVI に対する Security Zone アサイン**: FMC では **BVI 自体を Security Zone に追加することはできません**。セキュリティポリシー (ACP) は Bridge Group のメンバー物理ポート（`Gi0/1`, `Gi0/2` 等）が属する Security Zone に対して定義します。
2. **FTD IPS インターフェイスの罠 (Inline Set vs Passive vs ERSPAN)**:
   * **Subinterface の非サポート**: Inline Set および Passive インターフェイスでは **802.1Q Subinterfaces は使用不可**（物理ポートまたは EtherChannel のみ）。
   * **Q-in-Q (Double 802.1Q Tagging)**: Inline Set / Passive インターフェイスでは最大 2 つの 802.1Q タグ（Q-in-Q）を透過スキャン可能（※Firepower 4100/9300 は 1 タグのみ）。一方、通常の Firewall インターフェイスは 1 タグのみ対応。
   * **ERSPAN モードの前提条件**: ERSPAN インターフェイスは **Routed Firewall Mode でのみ構成可能**（Transparent Mode では構成不能）。
3. **Jumbo Frames ＆ TCP MSS Clamping の有効化要件**:
   * **ASA ジャンボフレーム有効化**: `mtu <IF> 9000` を設定後、**`reload` コマンドでアプライアンスを再起動**しなければジャンボフレームメモリバッファが割り当てられません。
   * **TCP MSS Clamping 設定**: ネットワーク境界で IPsec や GRE ヘッダーが付加されて PMTUD ブラックホールが発生するのを防ぐため、ASA では **`sysopt connection tcpmss 1360`**（デフォルト 1380）、FTD では FMC デバイスインターフェイス設定の **TCP MSS Clamping** を適切にチューニングします。
4. **Subinterface の MAC アドレス重複回避と IPv6 Link-Local**:
   * ASA / FTD の Subinterface はデフォルトで親物理ポートの Burned-in MAC アドレスを共有します。
   * **IPv6 の問題点**: MAC アドレスが同一だと EUI-64 により生成される IPv6 Link-Local アドレス (`FE80::`) が全サブインターフェイスで重複し、隣接ルーターで不整合が発生します。要件に応じて **`mac-address <MAC>`** コマンドでサブインターフェイスごとに一意の MAC アドレスを手動アサインします。
5. **Virtual Routers (VRF-Lite) ＆ Multi-Instance (FXOS) の構成**:
   * FTD 6.3+ では **Virtual Routers (VRF-Lite)** をサポート。複数の独立したルーティングテーブル (RIB) を保持可能。
   * Firepower 4100/9300 では FXOS 上で **Multi-instance (Container Instances)** を作成し、物理ポート/サブインターフェイスをインスタンス間でハードウェア共有/分離します。

---

## 🛠 設定方法

### 1. ASA CLI: Transparent Firewall Mode ＆ Bridge Group (BVI) 設定

```bash
! ===== Step 1: Transparent Mode への変更 (コンフィグ消去に注意) =====
firewall transparent

! ===== Step 2: Bridge Group ＆ BVI インターフェイス設定 =====
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 bridge-group 1
 no shutdown

interface GigabitEthernet0/1
 nameif inside
 security-level 100
 bridge-group 1
 no shutdown

interface BVI1
 ip address 192.168.1.254 255.255.255.0
 ipv6 address 2001:db8:1::254/64

! ===== Step 3: 管理トラフィック用の Static Route 構成 =====
! (※ Transparent Mode ではゲートウェイ IP と出力メンバーポートの両方を指定)
route outside 0.0.0.0 0.0.0.0 192.168.1.1
```

---

### 2. FMC GUI: FTD Integrated Routing and Bridging (IRB) 構成

```text
[ Step 1: Bridge Group Interface (BVI) の作成 ]
1. FMC UI で Devices > Device Management ➔ 対象 FTD を編集。
2. "Add Interfaces" ドロップダウンから "Bridge Group Interface" を選択。
3. General タブ:
   - Name: BVI1
   - Bridge Group ID: 1
   - Interface Members: GigabitEthernet0/1 (Inside1), GigabitEthernet0/2 (Inside2) チェック
4. IPv4 タブ:
   - IP Address: 10.1.1.1 / 24
5. Save をクリック。

[ Step 2: Routed Interface の作成と Security Zone アサイン ]
1. 既存の GigabitEthernet0/0 (Outside) を Routed Interface として設定 (IP: 203.0.113.1/24)。
2. Member Interfaces (Gi0/1, Gi0/2) にそれぞれ "Zone_Inside_1", "Zone_Inside_2" をアサイン。
3. Outside ポートに "Zone_Outside" をアサイン。
4. Access Control Policy (ACP) で Zone_Inside_1/2 ➔ Zone_Outside への通信許可ルールを作成の上 Deploy。
```

---

### 3. FMC GUI: FTD Inline Set (Tap Mode, Link State Prop) 構成

```text
[ Step 1: 物理ポートのモード変更 ]
1. Devices > Device Management ➔ 対象 FTD 編集 ➔ Interfaces タブ。
2. GigabitEthernet0/3 および 0/4 を編集 ➔ Mode: "None" (または Inline パス用) に設定し Enable。

[ Step 2: Inline Set の作成 ＆ パラメータ調整 ]
1. Devices > Device Management > Inline Sets タブを開く。
2. "Add Inline Set" をクリック:
   - Name: INLINE_SET_DC
   - Add Inline Pair: GigabitEthernet0/3 ＆ GigabitEthernet0/4 を指定
   - [✓] Enable Tap Mode (テスト運用時のみチェック)
   - [✓] Propagate Link State (障害伝播の有効化)
3. Save を押し、Deploy を実行。
```

---

### 4. ASA CLI: EtherChannel (LACP) ＆ 802.1Q Subinterface ＆ TCP MSS Clamping

```bash
! ===== Step 1: LACP Port-Channel の作成 =====
interface GigabitEthernet0/0
 channel-group 1 mode active
 no shutdown

interface GigabitEthernet0/1
 channel-group 1 mode active
 no shutdown

interface Port-channel1
 no shutdown

! ===== Step 2: 802.1Q サブインターフェイスの定義 =====
interface Port-channel1.10
 vlan 10
 nameif INSIDE_VLAN10
 security-level 100
 ip address 10.10.10.1 255.255.255.0
 mac-address 0050.56a1.0010              ! EUI-64 重複防止用の手動 MAC

interface Port-channel1.20
 vlan 20
 nameif DMZ_VLAN20
 security-level 50
 ip address 10.20.20.1 255.255.255.0
 mac-address 0050.56a1.0020

! ===== Step 3: MTU ＆ TCP MSS Clamping チューニング =====
mtu INSIDE_VLAN10 9000
mtu DMZ_VLAN20 9000
sysopt connection tcpmss 1360
sysopt connection tcpmss minimum 1200
```

---

## 🔍 検証コマンド

| 目的 | コマンド (ASA / FTD CLI) | 表示確認ポイント |
| :--- | :--- | :--- |
| **インターフェイス概要 ＆ IP アドレス確認** | <code>show interface ip brief</code> | インターフェイス名、IP アドレス、OK?、Status、Protocol 状態 |
| **Bridge Group (BVI) 接続 ＆ 学習確認** | <code>show bridge-group</code> | Bridge Group ID、メンバーポート、BVI IP、伝送統計 |
| **Layer 2 MAC アドレステーブル確認** | <code>show mac-address-table</code> | 学習された MAC アドレス、VLAN/Bridge Group、ポートマッピング |
| **EtherChannel (Port-Channel) 状態確認** | <code>show port-channel summary</code> | メンバーポートのフラグ (`P`: bundled in port-channel, `I`: stand-alone) |
| **FTD Inline Set パケット処理統計確認** | <code>show inline-set</code> (FTD CLI) | Inline Pair メンバー、Tap Mode 状態、Link State Propagation 状態 |
| **MTU ＆ TCP MSS 設定値確認** | <code>show running-config mtu</code><br><code>show running-config sysopt</code> | 設定された MTU サイズ、`sysopt connection tcpmss` 値 |
| **IPv6 インターフェイス ＆ SLAAC / NDP 確認** | <code>show ipv6 interface</code> | Link-local アドレス、Global アドレス、ND 応答統計 |
| **パケットフローおよび接続処理シミュレーション** | <code>packet-tracer input inside tcp 10.1.1.10 12345 10.2.2.20 80</code> | L2 Forwarding / L3 Route Lookup / Access-list / NAT 評価結果 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **Transparent Mode で対向ルーター間の ARP / Ping が不通になる** | BVI に `/30` などの極小サブネットが割り当てられており、ASA/FTD がネットワーク/ブロードキャスト アドレス宛て ARP を自動ドロップしている | <code>show interface bvi1</code><br><code>show mac-address-table</code> | BVI のサブネットマスクを `/24` など 3 アドレス以上収容可能なプレフィックスに変更する。 |
| **FTD Inline Set を配置したが、Snort によるブロックが発生せず攻撃パケットが通過してしまう** | Inline Set の構成で **`Tap Mode` が有効化** されており、コピーパケットのみスキャンして実トラフィックがスルーしている | <code>show inline-set</code> | FMC UI で Inline Set の **`Tap Mode` チェックボックスをオフ** にして Deploy を再実行する。 |
| **ASA の MTU を 9000 バイトに変更したが、ジャンボフレームパケットがドロップされる** | ASA において `mtu` 変更後にアプライアンスの **Reload（再起動）** を実行していないため、ジャンボバッファが未有効 | <code>show running-config mtu</code> | メンテナンスウィンドウにて ASA を `reload` し、ジャンボフレームメモリ予約を確定させる。 |
| **サブインターフェイス上で IPv6 接続を構成したところ、対向ルーターで Duplicate Address (DAD) エラーが発生する** | 親物理ポートの MAC アドレスが共有されており、生成された IPv6 Link-Local アドレス (`FE80::`) が重複している | <code>show ipv6 interface</code> | 各サブインターフェイスの構成に **`mac-address <一意のMAC>`** コマンドを追加し、Link-Local アドレスを分離する。 |
| **FTD Transparent Mode で TLS 1.3 通信や Proxy 通信のセッションが即座に切断される** | FTD 自体が TCP Proxy としてパケットを生成（Keepalive 送信）する際、経路解決（Adjacency Lookup）用 Static Route が未存在 | <code>show route</code><br>Syslog 110003 | FMC で Bridge Group メンバーポートを経由する適切な Default / Static Route を追加定義する。 |
| **FTD ERSPAN パッシブインターフェイスでパケットが受信できない** | ERSPAN インターフェイスを Transparent Mode の FTD に追加しようとしている | <code>show interface</code> | ERSPAN 制御（GRE トンネル解読）は **Routed Firewall Mode でのみサポート** されるため、モードを再確認する。 |

---

## ⚠ 制限事項

* **FTD IPS インターフェイスにおける機能制限**:
  * Inline Set および Passive インターフェイスでは、標準ファイアウォール機能（NAT、Routing、DHCP Server/Relay、VPN 終端、TCP Intercept、QoS）は完全非サポートとなります（Snort IPS 評価専用）。
* **Bridge Group (BVI) での非サポート機能**:
  * BVI インターフェイス上では、動的ルーティングプロトコル (OSPF, BGP, EIGRP) のネイバー確立、マルチキャストルーティング、および VPN 接続の終端（VPN Termination）はサポートされません。
* **Cloud Platform における Transparent Mode 非サポート**:
  * FTDv / ASAv のパブリッククラウド環境（AWS, Azure, GCP, OCI）でのデプロイメントでは、クラウドルーティング構成の制約上、**Transparent Mode (Layer 2) は完全非サポート**（Routed Mode のみ対応）となります。

---

## 🔄 他技術との関連

* **Snort 3 IPS Engine**: FTD Inline Set や Passive インターフェイスから引き渡されたパケットストリームに対し、シグネチャインスペクションを実行。
* **FXOS (Firepower Extensible OS)**: Firepower 4100/9300 において、物理ポート、802.1Q サブインターフェイス、および Container Instances へのインターフェイス割当を制御する下位 OS。
* **Cisco ISE (Identity Services Engine)**: 802.1X 認証や SGT (Security Group Tag) インラインタギング情報をパケットヘッダー経由で FTD インターフェイスへ伝達。

---

## 🧩 比較表

### 1. Routed Mode vs Transparent Mode vs Integrated Routing and Bridging (IRB)

| 比較項目 | Routed Mode (L3) | Transparent Mode (L2) | Integrated Routing ＆ Bridging (IRB) |
| :--- | :--- | :--- | :--- |
| **動作レイヤー** | Layer 3 (Router Hop) | Layer 2 (Bump in the wire) | **Layer 2 ブリッジ ＋ Layer 3 ルーティング** |
| **IP アドレス割り当て** | 各インターフェイスに別サブネット IP | **BVI にのみ同サブネット IP をアサイン** | **BVI ＋ 独立した Routed インターフェイス** |
| **L3 ルーティング** | 対応 (OSPF, BGP, EIGRP, Static) | 非対応 (パススルーのみ) | **BVI と Routed インターフェイス間でルーティング対応** |
| **VPN / NAT 終端** | 完全対応 | 限定対応 (Management 通信のみ/BVI NAT) | **対応 (Routed インターフェイス側で終端)** |
| **ネットワーク挿入性** | IP 再設計・デフォルト GW 変更が必要 | **IP アドレス体系を変更せず既存構成へ挿入可能** | 外部スイッチを置かずに L2/L3 混合集約が可能 |

### 2. FTD IPS Interface: Inline Set vs Passive vs ERSPAN Passive

| 比較項目 | Inline Set | Passive Mode | ERSPAN Passive Mode |
| :--- | :--- | :--- | :--- |
| **トラフィック通過** | インライン通過 (実ライン上に挿入) | コピーパケット受信 (SPAN/Mirror) | **GRE トンネル経由のコピーパケット** |
| **アクティブ遮断 (Drop)**| **対応** (Snort インラインブロック) | 非対応 (検知・ログ記録のみ) | 非対応 (検知・ログ記録のみ) |
| **Tap Mode オプション** | **対応** (コピー処理でドロップ回避) | — | — |
| **対応 Firewall モード** | Routed Mode / Transparent Mode | Routed Mode / Transparent Mode | **Routed Firewall Mode のみ** |
| **インターフェイス制限** | 物理ポート / EtherChannel のみ | 物理ポート / EtherChannel のみ | 物理ポート / Subinterface / EtherChannel |

---

## 💡 ベストプラクティス

1. **Inline Set における Link State Propagation の必須有効化**:
   * 耐障害性（Resiliency）を備えたネットワークに FTD Inline Set を挿入する場合、必ず **Link State Propagation** を有効化します。片側リンク障害時に即座に対向ポートを Down させ、上流ルーターの高速迂回（Reroute）を誘発します。
2. **サブインターフェイスでの一意な MAC アドレス手動アサイン**:
   * IPv6 デュアルスタック環境で 802.1Q サブインターフェイスを複数展開する場合、全サブインターフェイスに一意の `mac-address` を手動設定し、IPv6 Link-Local アドレス (`FE80::`) の重複障害を根絶します。
3. **TCP MSS Clamping による PMTUD 障害防止**:
   * IPsec VPN、GRE トンネル、VLAN タグを多用する構成では、アクセス制御ポリシーやインターフェイス設定で **TCP MSS を 1360 バイト程度に Clamp** し、断片化不可 (DF bit) パケットのドロップ事故を防ぎます。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA Transparent Mode ＆ Bridge Group (BVI) の基本構築

#### 問題
ASA1 を Transparent Firewall Mode に変更し、`Gi0/0` (Outside) と `Gi0/1` (Inside) を Bridge Group 1 に割り当てよ。BVI1 の IP アドレスを `10.1.1.254/24` に設定し、管理デフォルトルートを `10.1.1.1` に向けて設定せよ。

#### 要件
* `firewall transparent` コマンドを使用すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
firewall transparent

interface GigabitEthernet0/0
 nameif outside
 security-level 0
 bridge-group 1
 no shutdown

interface GigabitEthernet0/1
 nameif inside
 security-level 100
 bridge-group 1
 no shutdown

interface BVI1
 ip address 10.1.1.254 255.255.255.0

route outside 0.0.0.0 0.0.0.0 10.1.1.1
```

#### 検証方法
* `show firewall` で `Transparent` モードであることを確認し、`show bridge-group` でメンバーポートと BVI1 の状態を確認する。

---

### 例2: FTD Integrated Routing and Bridging (IRB) の FMC 設定

#### 問題
FMC 管理下の FTD において、Bridge Group 1 (`Gi0/1`, `Gi0/2`) と Routed Interface `Gi0/0` の間でルーティングを可能にする IRB 構成を作成せよ。

#### 要件
* BVI 1 の IP: `172.16.10.1/24`, Gi0/0 の IP: `192.168.100.1/24`

#### 設定例

```text
[ FMC GUI Steps ]
1. Devices > Device Management ➔ 対象 FTD 編集 ➔ Interfaces タブ。
2. Add Interfaces > Bridge Group Interface:
   - Name: BVI1
   - Bridge Group ID: 1
   - Members: GigabitEthernet0/1, GigabitEthernet0/2
   - IPv4 Address: 172.16.10.1 / 24
3. Edit GigabitEthernet0/0:
   - Name: Outside
   - Mode: Routed
   - IPv4 Address: 192.168.100.1 / 24
4. メンバーポートに Security Zone をアサインし Deploy を実行。
```

#### 検証方法
* Inside 端末 (`172.16.10.50`) から Outside 宛て Ping を送信し、BVI1 経由でルーティング通過することを確認する。

---

### 例3: FTD Inline Set と Link State Propagation の設定

#### 問題
FTD の `Gi0/3` と `Gi0/4` を使用して Inline Set `INLINE_PAIR1` を構成し、Link State Propagation を有効化せよ。

#### 要件
* FMC の Inline Sets メニューから設定すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Devices > Device Management ➔ Inline Sets タブ。
2. Add Inline Set:
   - Name: INLINE_PAIR1
   - Add Pair: GigabitEthernet0/3 ＆ GigabitEthernet0/4
   - [✓] Propagate Link State
3. Save して Deploy を実行。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`Link State Propagation: Enabled` と表示されることを確認する。

---

### 例4: FTD Inline Set での Tap Mode 有効化

#### 問題
既存の Inline Set `INLINE_PAIR1` において、通信をドロップさせずにスキャンのみを行う Tap Mode を有効化せよ。

#### 要件
* `Tap Mode` を有効化すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Devices > Device Management ➔ Inline Sets タブ。
2. INLINE_PAIR1 を編集 ➔ [✓] Enable Tap Mode にチェック。
3. Save して Deploy を実行。
```

#### 検証方法
* 擬似攻撃パケットを送出し、通信自体は遮断されずに通過するが、FMC の Intrusion Events に検知ログ（Would have dropped）が記録されることを確認する。

---

### 例5: ASA LACP EtherChannel ＆ 802.1Q サブインターフェイス構成

#### 問題
ASA1 の `Gi0/0` と `Gi0/1` を用いて LACP 動作の `Port-channel 1` を構築し、VLAN 10 (`10.10.10.1/24`) および VLAN 20 (`10.20.20.1/24`) の 802.1Q サブインターフェイスを作成せよ。

#### 要件
* サブインターフェイスに一意の MAC アドレスを設定すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
interface GigabitEthernet0/0
 channel-group 1 mode active
 no shutdown

interface GigabitEthernet0/1
 channel-group 1 mode active
 no shutdown

interface Port-channel1.10
 vlan 10
 nameif INSIDE_V10
 security-level 100
 ip address 10.10.10.1 255.255.255.0
 mac-address 0011.2233.0010

interface Port-channel1.20
 vlan 20
 nameif DMZ_V20
 security-level 50
 ip address 10.20.20.1 255.255.255.0
 mac-address 0011.2233.0020
```

#### 検証方法
* `show port-channel summary` で EtherChannel 状態を確認し、`show interface` で各サブインターフェイスの MAC アドレスを確認する。

---

### 例6: ASA ジャンボフレームの有効化と MTU 9000 設定

#### 問題
ASA1 の全データインターフェイスにおいてジャンボフレームを有効化し、`inside` インターフェイスの MTU を `9000` バイトに設定せよ。

#### 要件
* 再起動要件を考慮すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
mtu inside 9000

! 設定反映のためアプライアンスをリロード
reload
```

#### 検証方法
* 再起動後、`show running-config mtu` を実行し、`mtu inside 9000` がアクティブになっていることを確認する。

---

### 例7: FTD での IPv6 SLAAC ＆ ICMPv6 Inspection 設定

#### 問題
FTD の `inside` インターフェイスにおいて IPv6 SLAAC アドレス自動設定を有効化し、PMTUD 維持のために ICMPv6 インスペクションを有効化せよ。

#### 要件
* FMC から構成すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Devices > Device Management ➔ 対象 FTD 編集 ➔ Inside インターフェイス編集。
2. IPv6 タブ:
   - Enable IPv6: チェック
   - Autoconfig (SLAAC): チェック
3. Objects > Service Actions / Policy Maps ➔ ICMPv6 Stateful Inspection を有効化の上 Deploy。
```

#### 検証方法
* FTD CLI で `show ipv6 interface inside` を実行し、SLAAC アドレスが自動生成され、`ICMP Inspection` が動作していることを確認する。

---

### 例8: ASA TCP MSS Clamping チューニング

#### 問題
ASA1 を通過する全 TCP 接続の最大セグメントサイズ (MSS) を 1360 バイトに制限し、最小値を 1200 バイトに制限せよ。

#### 要件
* `sysopt connection tcpmss` コマンドを使用すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
sysopt connection tcpmss 1360
sysopt connection tcpmss minimum 1200
```

#### 検証方法
* `show running-config sysopt` を実行し、設定値が反映されていることを確認する。

---

### 例9: FTD ERSPAN パッシブインターフェイスの構成

#### 問題
FTD の `Gi0/5` ポートを Routed Firewall Mode において ERSPAN パッシブモニタリングポート（Source IP: `192.168.200.1`）として設定せよ。

#### 要件
* FMC Interfaces から設定すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Devices > Device Management ➔ 対象 FTD 編集 ➔ GigabitEthernet0/5 編集。
2. Mode: "Erspan" を選択。
3. ERSPAN IP Address: 192.168.200.1 / 24 を指定して Save ＆ Deploy。
```

#### 検証方法
* FTD CLI で `show interface GigabitEthernet0/5` を実行し、`ERSPAN Monitored` 状態であることを確認する。

---

### 例10: FMC REST API を用いた FTD インターフェイス一覧の取得

#### 問題
FMC REST API を使用して、指定した FTD アプライアンスに定義されている全物理・論理インターフェイス情報を GET リクエストで取得せよ。

#### 要件
* エンドポイント `/api/fmc_config/v1/domain/{domainUUID}/devices/devicerecords/{containerUUID}/interfaces` を使用すること。

#### 設定例

```bash
curl -k -X GET "https://fmc.corp.local/api/fmc_config/v1/domain/e276abec-e0f2-11e3-8169-6d9ed13b206f/devices/devicerecords/6e432abc-1111-2222-3333-123456789def/interfaces" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json"
```

#### 検証方法
* レスポンスの JSON 配列内に各インターフェイスの Name、Mode (Routed/Inline/Bridge)、および IP 設定が正しく含まれていることを確認する。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: FTD において 2 つの物理ポート (`Gi0/1`, `Gi0/2`) を用いて Inline Set を構成した。ネットワーク管理者が障害試験として `Gi0/1` のケーブルを抜いたところ、直ちに対向の `Gi0/2` のポート LED が消灯し、対向スイッチ側でもリンク Down が検知された。この動作を実現している Inline Set の機能はどれか。

* A) Hardware Bypass
* B) Link State Propagation
* C) Tap Mode
* D) Inline Pair Redundancy

**正解**: **B**  
**解説**: **Link State Propagation** 機能は、Inline Set を構成する片側の物理ポートのリンク状態（Up/Down）が変化した際、最大 4 秒以内に他方のポートへリンク状態を同期・伝播させる機能です。これにより、上流ルーターは対向リンク Down を即座に認識して障害迂回（Reroute）を実行できます。

---

### 2. トラブルシュート問題

**問題**: 管理者は FTD 上で Inline Set を作成し、インラインでトラフィックを通過させている。Snort ルールにより悪意あるパケットが検出され、FMC イベントログには `Would have dropped` と記録されたが、実際には攻撃パケットが破棄されずにターゲットサーバーへ到達してしまった。原因として最も適切なものはどれか。

* A) FTD の Threat License が失効しているため。
* B) Inline Set の設定で **`Tap Mode` が有効化** されているため。
* C) FTD が Routed Mode で動作しているため。
* D) ポート上で Q-in-Q タグが除去されたため。

**正解**: **B**  
**解説**: **Inline Tap Mode** を有効化すると、FTD はインライン物理接続を維持しながらパケットのコピーを作成して Snort スキャンを行います。Snort ルールにヒットしても実トラフィックはドロップされずそのまま通過し、ログには `Would have dropped`（インラインであればドロップされていた）と記録されます。

---

### 3. Design 問題

**問題**: 企業ネットワークにおいて、既存のルーター間（同一 `10.1.1.0/24` サブネット）に FTD を挿入してアクセス制御を行いたい。また、将来的に FTD に直結された別サブネット (`192.168.100.0/24`) の Web サーバー群との間でも FTD 自体がルーティング（デフォルトゲートウェイ）を行いたい。最適なデプロイメントモードおよび設定構成はどれか。

* A) FTD を Transparent Mode に設定し、複数 Bridge Group を作成する。
* B) FTD を **Routed Mode** に設定し、既存サブネット用に **Bridge Group (BVI1)** を作成して **BVI に Name (`nameif`) を割り当てて IRB を有効化** する。
* C) FTD を IPS-Only Inline Set モードに設定する。
* D) FTD を Multi-Instance モードにして 2 つの容器を作成する。

**正解**: **B**  
**解説**: **Integrated Routing and Bridging (IRB)** を使用すると、FTD Routed Mode 内で Bridge Group (BVI) を定義して Layer 2 ブリッジングを行いながら、Named BVI を介して他の Routed インターフェイス (`192.168.100.0/24`) との間で Layer 3 ルーティングを統合実行できます。

---

### 4. 実装問題

**問題**: ASA において物理ポートの MTU をデフォルトの 1500 バイトから 9000 バイトに変更した。設定変更後、アプライアンスを再起動せずにジャンボフレームパケットを送信した際のアプライアンスの動作として正しいものはどれか。

* A) パケットは正常に 9000 バイトで送信される。
* B) ジャンボフレーム用のメモリバッファが割り当てられていないため、1500 バイトを超えるパケットはドロップされる（または要再起動状態となる）。
* C) 自動的に MSS が 1360 に書き換えられる。
* D) 物理ポートが Disable に変更される。

**正解**: **B**  
**解説**: ASA（および一部の専用ハードウェア）では、MTU を 1500 バイト以上に拡張してジャンボフレームを使用する場合、**アプライアンスの再起動 (`reload`)** を行ってジャンボバッファメモリ割り当てを確定させる必要があります。

---

### 5. コンフィグ読解問題

**問題**: ASA 上で以下のコンフィグを投入した。

```bash
interface Port-channel1.10
 vlan 10
 nameif INSIDE
 security-level 100
 ip address 10.1.1.1 255.255.255.0
```

この設定に関する説明として **誤っているもの** はどれか。

* A) `Port-channel1.10` は 802.1Q タグ `10` を持つパケットを透過・処理する。
* B) 親インターフェイス `Port-channel1` は自動的に 802.1Q トランクとして動作する。
* C) サブインターフェイスには自動的に親物理ポートとは異なる一意の Burned-in MAC アドレスがアサインされる。
* D) `INSIDE` インターフェイスの Security Level は 100 である。

**正解**: **C**  
**解説**: ASA / FTD のサブインターフェイスは、デフォルトで**親物理ポート（または EtherChannel）の Burned-in MAC アドレスを共有**します。自動的に一意の MAC アドレスが割り当てられることはありません（必要に応じて `mac-address` コマンドで手動アサインします）。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Device Configuration Guide: Interface Overview](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Device Configuration Guide: Transparent or Routed Firewall Mode](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Device Configuration Guide: Inline Sets and Passive Interfaces](https://www.cisco.com/)
* [Cisco ASA Series General Operations Configuration Guide: Interfaces and Firewall Modes](https://www.cisco.com/)
* [Cisco Technical Notes: Configure Firepower Threat Defense Interfaces in Routed Mode and Transparent Mode](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Secure Firewall Connectivity and Architecture (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で Network Connectivity が出題された場合の構成選択フローチャート：
    * **「Insert FTD into existing subnet without IP change」** ➔ Transparent Mode または Integrated Routing and Bridging (IRB)。
    * **「Inspect live traffic inline with fail-open resiliency」** ➔ FTD Inline Set ＋ Hardware Bypass。
    * **「Propagate link failure to adjacent routers」** ➔ FTD Inline Set ➔ Enable `Link State Propagation`。
    * **「Audit attack traffic without inline dropping」** ➔ FTD Inline Set ➔ Enable `Tap Mode`（または Passive SPAN/ERSPAN）。
    * **「Configure Subinterfaces with unique IPv6 Link-Local」** ➔ `interface <IF>.<ID>` ➔ `vlan <ID>` ➔ 手動 `mac-address <MAC>` 設定。
* **図解（FTD インターフェイス処理トポロジ）**:
  ```text
  [ Network Interfaces ]
             │
             ├── 1. Routed Interfaces (L3 Router Hop) ──► Full LINA ＋ Snort DPI
             │
             ├── 2. Bridge Group / BVI (L2 Transparent / IRB) ──► L2 MAC Lookup / L3 Routing
             │
             ├── 3. Inline Set (IPS-Only) ──► Snort DPI Only (Bypass LINA Firewall)
             │        ├── Tap Mode: Inspect Copy Only
             │        └── Link State Propagation: Propagate Link Down
             │
             └── 4. Passive / ERSPAN (IDS) ──► Receive Copy Traffic Only
  ```
* **注意点**:
  * ラボ試験で ASA/FTD のインターフェイス設定やデプロイメントモードを変更した後は、必ず `show interface ip brief`、`show bridge-group`、`show inline-set`、および `packet-tracer` を実行し、データプレーン上でパケットが意図通りのインターフェイスモードで処理されていることを確認・証明してください。

