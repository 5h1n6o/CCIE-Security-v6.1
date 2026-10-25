---
layout: default
title: 1.1.a-Routed
nav_order: 1
parent: 1.1-Deployment-modes-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.1.a Routed Mode

Cisco ASA (Adaptive Security Appliance) および Cisco FTD (Firepower Threat Defense) における **Routed Mode（ルーテッドモード）** は、デバイスがネットワーク上で Layer 3 ルータホップ（ルータ段）として動作する標準的なデプロイメントモードです。トラフィックのルーティング、ステートフルファイアウォールインスペクション、NGIPS スキャン、NAT 変換、および VPN 終端を統合して実行する、境界防御の中核技術です。

---

## 📘 概要

* **機能概要**:
  * ファイアウォールの各物理・論理インターフェイスに異なる L3 サブネットの IP アドレスを割り当て、異なるネットワーク（Inside, Outside, DMZ 等）間で IP パケットのルーティングおよびアクセス制御を行う動作モードです。
  * Cisco ASA（LINA エンジン）および Cisco FTD（LINA + Snort 3 / Snort 2 エンジン）の双方でデフォルトのデプロイメントモードとして定義されています。
* **利用目的**:
  * 異なるセキュリティゾーン間（信頼ネットワーク、非信頼ネットワーク、DMZ）の L3 境界を確立し、アクセスコントロールリスト (ACL) や Access Control Policy (ACP) によるトラフィック制御を実施する。
  * NAT (Network Address Translation) / PAT (Port Address Translation) を使用して内部プライベート IP を隠蔽・変換する。
  * 動的ルーティングプロトコル (OSPF, BGP, EIGRP, RIP) や Policy-Based Routing (PBR) を使用して、ネットワークトポロジーの変化に適応したトラフィック転送を行う。
  * Site-to-Site IPsec VPN や Remote Access VPN (Cisco AnyConnect / Secure Client) のゲートウェイとして暗号化セッションを終端する。
* **どのような場面で利用するか**:
  * 企業ネットワークとインターネットとの境界（Internet Edge）。
  * データセンターと社内バックボーンとの境界、あるいは複数 DMZ セグメントの分離・制御。
  * 拠点間 VPN 接続やリモートワーカーのセキュアアクセス終端ポイント。

---

## 🔑 要点

| 項目 | Cisco ASA (Routed Mode) | Cisco FTD (Routed Mode / FMC 管理) |
| :--- | :--- | :--- |
| **デフォルトモード** | デフォルト（`show firewall` で確認） | デフォルト（シリアルアップ・初期セットアップ時） |
| **IP アドレス要件** | 各 Nameif インターフェイスに固有の L3 IP が必須 | 各 Routed ゾーンインターフェイスに固有の L3 IP が必須 |
| **ルーティング機能** | OSPFv2/v3, BGP, EIGRP, RIP, Static, PBR, SLA Monitor | OSPFv2/v3, BGP, EIGRP, RIP, Static, PBR, ECMP |
| **NAT 機能** | Auto NAT (Section 2), Manual NAT (Section 1 & 3) | Auto NAT, Manual NAT (FMC 画面で構成) |
| **VPN 終端** | IPsec Site-to-Site, Remote Access (AnyConnect) | IPsec Site-to-Site, Remote Access (AnyConnect/Secure Client) |
| **L2/L3 混合機能** | BVI (Bridge Virtual Interface) による IRB をサポート | BVI (Bridge Virtual Interface) による IRB をサポート |
| **マルチテナント化** | Multi-Context モードをサポート | FXOS Multi-Instance (Container) によりハードウェアレベル分離 |
| **管理アクセス** | Dedicated Management ポート、または Data インターフェイス (ASDM/SSH) | Dedicated Management ポート、または Data インターフェイス (FMC Access) |

---

## 🏗 動作原理

Routed Mode では、パケットは Layer 3 ルータホップとして受信・処理・送信されます。ASA では LINA エンジンが、FTD では LINA（データプレーン・接続追跡）と Snort 3（ディープパケット・IPS・AMP スキャン）が密接に連携して処理を行います。

```text
[ Client: 10.1.1.100 ]
         │
         ▼ (Ingress: GigabitEthernet0/0 - Inside: 10.1.1.1)
┌────────────────────────────────────────────────────────────────────────┐
│ Cisco FTD / ASA (Routed Mode)                                          │
│                                                                        │
│ 1. LINA Engine (Layer 3 / Layer 4 Check)                               │
│    ├── Ingress VLAN / L2 Header Decapsulation                          │
│    ├── Connection Table Lookup (Existing Session?)                     │
│    ├── Pre-filter Policy (Fastpath / Drop / Analyze Check)             │
│    ├── Input ACL / Access Control Evaluation                           │
│    └── NAT / UN-NAT Architecture Evaluation                            │
│                                                                        │
│ 2. Snort 3 Engine (L4-L7 Deep Inspection - FTD Only)                  │
│    ├── Security Intelligence (IP / Domain / URL Reputation)            │
│    ├── SSL / TLS Decryption Policy Evaluation                          │
│    ├── Application Visibility and Control (AVC / AppID)                │
│    ├── Intrusion Prevention System (IPS Signature Match)               │
│    └── File Policy / AMP (SHA-256 Analysis / Sandbox Submission)       │
│                                                                        │
│ 3. LINA Egress Routing & Forwarding                                    │
│    ├── Routing Table Lookup (FIB / Static / Dynamic / PBR)             │
│    ├── Egress Interface Selection                                      │
│    └── L2 Header Encapsulation                                         │
└────────────────────────────────────────────────────────────────────────┘
         │
         ▼ (Egress: GigabitEthernet0/1 - Outside: 209.165.200.225)
[ Internet Router / Destination Server ]
```

---

## ⚙ 動作シーケンス

FTD および ASA の Routed Mode におけるパケット処理アルゴリズムは以下の順番で厳密に実行されます。

1. **パケット受信とデカプセル化**:
   * 物理ポートまたは VLAN サブインターフェイスで L2 フレームを受信。MAC アドレスの検証と VLAN タグのストリップを実施。
2. **既存接続テーブル照会 (Conn Table Lookup)**:
   * 5 プレット（送信元 IP, 宛先 IP, 送信元ポート, 宛先ポート, プロトコル）に基づき既存のフロー（接続）が存在するか検索。
   * 既存接続に一致する場合、ACL 評価をスキップして高速パス処理（Fastpath）に移行。
3. **新規セッション処理（最初の SYN パケット等）**:
   * **Pre-filter Policy 評価**: トラフィックの高速バイパス（Fastpath）、即時ドロップ（Block）、または Snort スキャン（Analyze）の振り分け。
   * **Input ACL / Access Control Policy (ACP) 評価**: ネットワークオブジェクト、ポート、ID、ゾーンに基づく許可・拒否判定。
   * **NAT 処理 (Twice / Auto NAT)**: ポリシーに基づく送信元・宛先 IP/ポートの変換ロジック適用。
4. **Snort インスペクション（FTD で Analyze 判定の場合）**:
   * DAQ (Data Acquisition) 層を介してパケットが Snort プロセスへ渡される。
   * Security Intelligence 照合 → SSL 復号 → AppID 識別 → IPS 検査 → File / AMP スキャンの順で多層スキャンを実行。
5. **ルーティングテーブル検索 (Route Lookup)**:
   * 送信元 PBR、あるいは宛先 IP に基づく FIB (Forwarding Information Base) / FIB ルートテーブルを検索。
   * Egress インターフェイスとネクストホップ IP を決定。
6. **L2 アドレス解決と送出**:
   * ネクストホップの ARP キャッシュを照会し、宛先 MAC アドレスをセットして Egress インターフェイスからパケットを出力。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

* **FMC Access on Data Interface (データポート経由の FMC 管理)**:
  * 通常 FTD の管理（sftunnel）は Dedicated Management 0/0 ポートで行いますが、ラボ問題で「データポート（例: Outside インターフェイス）経由で FMC から管理させよ」と指示されるケースがあります。
  * FTD CLI で `configure network {ipv4|ipv6} manual <IP> <Netmask> data-interfaces` を実行後、FMC の `Devices > Device Management > Interfaces` で該当データポートの **FMC Access** を有効化します。
  * FMC 側の DB と FTD の設定に不一致（Discrepancies）が生じるため、FMC の **FMC Access Details** 画面で **Acknowledge** を実行しないとポリシーのデプロイがブロックされます。
* **同じ Security Level 間の通信 (ASA)**:
  * ASA ではデフォルトで同一 Security Level を持つインターフェイス間の通信はドロップされます。同一 Security Level 間でルーティングさせる場合、`same-security-traffic permit inter-interface` の有効化が必要です。
* **サブインターフェイスと VLAN 802.1Q 構成**:
  * 物理ポート（例: `GigabitEthernet0/0`）を親として、`GigabitEthernet0/0.10`（VLAN 10）や `GigabitEthernet0/0.20`（VLAN 20）を作成する場合、親物理ポートは `no shutdown` 状態にして IP アドレスは割り当てず、サブインターフェイス側に `vlan <ID>` と IP アドレス、Security Zone / Nameif を割り当てます。
* **PBR (Policy-Based Routing) と PBR Bypass**:
  * FMC 7.0+ では PBR を GUI 上で設定可能です（`Devices > Device Management > Routing > Policy Based Routing`）。
  * 宛先がローカルサブネット宛の通信まで PBR で ISP へ曲げてしまわないよう、Extended ACL でローカル通信を除外（否定 / DENY または明示的 ALLOW ルールの順序調整）する必要があります。
* **Integrated Routing and Bridging (IRB) on Routed Mode**:
  * Routed Mode であっても Bridge Group (BVI) を追加構成できます。スイッチポートのように複数ポートを同一 L2 セグメントとしてまとめつつ、他の Routed インターフェイスとの間を L3 ルーティングさせたい場合に利用します。

---

## 🛠 設定方法

### 1. Cisco ASA: Command Line (CLI) による Routed Mode 基本構成

```bash
! 1. Firewall Mode の確認
show firewall
! (Firewall mode: Router と表示されることを確認)

! 2. 物理インターフェイスおよびサブインターフェイスの設定
interface GigabitEthernet0/0
 description Inside Network
 nameif inside
 security-level 100
 ip address 10.1.1.254 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/1
 description Outside Internet
 nameif outside
 security-level 0
 ip address 209.165.200.226 255.255.255.248
 no shutdown
!
! サブインターフェイス (VLAN Tagging) 構成例
interface GigabitEthernet0/2
 description Trunk to DMZ Switch
 no shutdown
!
interface GigabitEthernet0/2.50
 description DMZ Web Servers
 vlan 50
 nameif dmz
 security-level 50
 ip address 172.16.50.254 255.255.255.0
!
! 3. 同一セキュリティレベル間通信および同一インターフェイス折り返し許可
same-security-traffic permit inter-interface
same-security-traffic permit intra-interface

! 4. スタティックデフォルトルートの設定
route outside 0.0.0.0 0.0.0.0 209.165.200.225 1
```

### 2. Cisco FTD: FMC GUI による Routed Mode インターフェイスおよびルーティング構成

1. **物理 / サブインターフェイスの設定**:
   * **Devices > Device Management** へ移動し、対象 FTD の編集（鉛筆アイコン）をクリック。
   * **Interfaces** タブで対象ポート（例: `GigabitEthernet0/0`）の編集を開く。
   * **Name**: `inside`, **Enabled**: チェックオン。
   * **Mode**: `Routed`（デフォルト）。
   * **Security Zone**: 新規作成または既存の `Inside-Zone` (Routed タイプ) を選択。
   * **IPv4** タブ: `Use Static IPv4` を選択し `10.1.1.254/24` を入力して **OK**。
2. **VLAN サブインターフェイスの追加**:
   * **Add Subinterface** をクリック。
   * **Interface**: `GigabitEthernet0/2`, **Subinterface ID**: `50`, **VLAN ID**: `50`。
   * **Name**: `dmz`, **Security Zone**: `DMZ-Zone` (Routed)。
   * **IPv4**: `172.16.50.254/24` を指定。
3. **スタティックルートの設定**:
   * **Routing > Static Route** タブを選択し **Add Route** をクリック。
   * **Type**: `IPv4`, **Interface**: `outside`。
   * **Available Network**: `any-ipv4` (0.0.0.0/0)。
   * **Gateway**: `Gateway_Outside_ISP` (IP: 209.165.200.225 を指す Network Object)。
   * **Metric**: `1` として **OK**。
4. **Deploy の実行**:
   * 画面右上の **Deploy** ボタンを押して対象 FTD に変更を適用。

### 3. Cisco FTD: CLI によるデータポート経由の FMC アクセス (FMC Access on Data Interface) 構成

```bash
! 1. FTD コンソール CLI に接続し、管理ポートのデフォルトゲートウェイを data-interfaces に設定
> configure network ipv4 manual 192.168.45.46 255.255.255.0 data-interfaces

! 2. FMC GUI 上で管理アクセスの有効化
! Devices > Device Management > Interfaces > Data インターフェイス (例: outside) を編集
! 「Enable FMC Access」のチェックボックスをオンにして保存

! 3. FMC 側で不一致の認識 (Acknowledge)
! Devices > Device Management > Device > Management > FMC Access Details
! 表示される設定差異を確認し「Acknowledge」をクリックしてデプロイブロックを解除

! 4. FMC から FTD へ Deploy を実行
```

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **ファイアウォールモード確認** | ASA / FTD CLI | <code>show firewall</code> |
| **インターフェイス IP/ステータス一覧** | ASA / FTD CLI | <code>show interface ip brief</code> |
| **インターフェイス詳細・カウンタ確認** | ASA / FTD CLI | <code>show interface GigabitEthernet0/0</code> |
| **L3 ルーティングテーブル確認** | ASA / FTD CLI | <code>show route</code> |
| **ARP テーブル確認** | ASA / FTD CLI | <code>show arp</code> |
| **アクティブ接続（Conn Table）表示** | ASA / FTD CLI | <code>show conn</code> / <code>show conn detail</code> |
| **NAT テーブルおよび変換ヒット数表示** | ASA / FTD CLI | <code>show nat</code> / <code>show xlate</code> |
| **パケットフロー処理シミュレーション** | FTD / ASA CLI | <code>packet-tracer input inside tcp 10.1.1.100 12345 209.165.201.1 80 detail</code> |
| **FMC 管理セッション状態確認** | FTD CLI | <code>show managers</code> / <code>sftunnel-status-brief</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **サブネット間通信がルーティングされない** | インターフェイスが `no shutdown` になっていない、または Security Zone の未割り当て | ASA/FTD: <code>show interface ip brief</code> | インターフェイスを Enable にし、FMC 上で正しく Security Zone (Routed) を紐付けて Deploy する。 |
| **同一 Security Level インターフェイス間で通信不能 (ASA)** | コマンドの未設定 | ASA: <code>show running-config same-security-traffic</code> | <code>same-security-traffic permit inter-interface</code> を設定する。 |
| **データポート管理アクセス設定後に FMC デプロイが失敗** | FMC と FTD ローカル CLI 間の DB 不一致 | FMC: `FMC Access Details` バナー | FMC GUI の `FMC Access Details` で不一致項目を確認の上、**Acknowledge** を実行してから再 Deploy する。 |
| **サブインターフェイスでタグ付きパケットが落ちる** | 親物理ポートが Down 状態、または VLAN ID の齟齬 | ASA/FTD: <code>show interface GigabitEthernet0/2.50</code> | 親物理ポートを `no shutdown` に設定し、対向スイッチの Trunk カプセル化 VLAN ID と一致させる。 |
| **PBR を設定したらローカル通信も外部へ飛んでしまう** | PBR ACL で内部宛トラフィックを除外していない | FTD: <code>show running-config route-map</code> | PBR に使用する Extended ACL の先頭に、内部宛通信を拒否（`deny ip any 10.0.0.0 255.0.0.0`）するルールを追加する。 |

---

## ⚠ 制限事項

* **FMC Access on Data Interface の制約**:
  * 1 台の FTD に対して FMC Access を有効化できるデータポートは **1 つの物理ポートのみ** です（サブインターフェイスや EtherChannel 上では不可）。
  * Routed Mode のみでサポートされ、Transparent Mode や Inline Pair モードでは使用できません。
  * 管理ポートが DHCP 設定の場合、データポート経由の FMC 管理アクセスは設定できません（マニュアル IP 必須）。
* **モード変更時の設定初期化**:
  * ASA / FTD でモードを Routed から Transparent、あるいはその逆に変更すると、**既存のすべてのインターフェイス設定およびポリシー構成が完全に削除・初期化** されます。
* **ルーティングテーブルの分離 (VRF / Management)**:
  * Dedicated Management ポート（`Diagnostic` ポート含む）は、データプレーンのルーティングテーブルとは独立した管理用 Linux ルーティングテーブル（`management-only`）を使用します。データポート経由のトラフィックと動的ルーティングを共有することはできません。

---

## 🔄 他技術との関連

* **1.2 Access Control & Pre-filter Policies**: Routed インターフェイスに割り当てられた Security Zone（Inside, Outside 等）を評価条件として ACP ルールを定義します。
* **3.1 NAT (Network Address Translation)**:
  * Routed Mode は Auto NAT (Section 2) および Twice/Manual NAT (Section 1 & 3) のフル機能をサポートします。
* **1.3 High Availability & Clustering**:
  * Active/Standby HA や Cluster 環境において、Routed インターフェイスは仮想 IP (IP) および Standby IP を持ち、フェイルオーバー時に Gratuitous ARP (GARP) を送出して L2/L3 スイッチのテーブルを更新します。
* **1.1.b Transparent Mode**:
  * Transparent Mode が L2 ブリッジ（Bump in the wire）動作であるのに対し、Routed Mode は L3 ホップ動作です。ただし、Routed Mode 内に Bridge Group (BVI) を含めることで L2/L3 統合運用（IRB）が可能です。

---

## 🧩 比較表

### 1. ASA Routed Mode vs FTD Routed Mode

| 比較項目 | Cisco ASA (Routed Mode) | Cisco FTD (Routed Mode / FMC) |
| :--- | :--- | :--- |
| **セキュリティレベル概念** | `security-level 0 - 100` を直接使用 | 内部表現のみ（ポリシー評価は Security Zone で実施） |
| **スキャンエンジン** | LINA エンジン（＋モジュールで FirePOWER） | LINA + Snort 3 / Snort 2 が完全統合 |
| **設定インターフェース** | CLI (Config t) / ASDM | FMC (Firepower Management Center) / FDM |
| **PBR (Policy Based Routing)** | CLI で `route-map` および `ip policy route-map` | FMC 7.0+ GUI 上でオブジェクトベース設定可能 |
| **マルチテナント化** | Multi-Context モード（単一 OS 分割） | FXOS Multi-Instance（コンテナレベルの完全独立） |

### 2. Single-Context Routed vs Multi-Context Routed (ASA)

| 比較項目 | Single-Context Routed Mode | Multi-Context Routed Mode |
| :--- | :--- | :--- |
| **インスタンス数** | 1 台の物理/仮想 ASA に 1 つのコンフィグ | 1 台の ASA を複数の仮想 ASA (Context) に分割 |
| **管理モデル** | 単一の管理者グループ | Admin Context ＋ 顧客ごとの Data Context |
| **Dynamic Routing** | OSPF, BGP, EIGRP, RIP すべてサポート | 制限あり（古いバージョンでは Static ルートメイン） |
| **VPN 機能** | Remote Access VPN / IPsec VPN フルサポート | Remote Access VPN 非サポート（Site-to-Site は制限付き可） |

---

## 💡 ベストプラクティス

1. **インターフェイスゾーン命名と整理**:
   * Security Zone 名はインターフェイスの役割（例: `CZ_Inside_Routed`, `CZ_Outside_Routed`, `CZ_DMZ_Routed`）が即座に識別できる命名規則を採用します。
2. **Anti-Spoofing (Unicast RPF) の有効化**:
   * 外部に面した Outside インターフェイスには、送信元 IP 詐称攻撃を防止するために FMC の `Interfaces > Advanced > Security Configuration` にて **Enable Anti Spoofing (uRPF)** を有効化します。
3. **TCP MSS Adjustment の構成**:
   * PPPoE や IPsec VPN を通過するトラフィックで MTU 超過によるフラグメント/ドロップを防ぐため、`sysopt connection tcpmss 1380`（または FMC の Platform Settings）で TCP MSS 値を適切にクランプします。
4. **Path Monitoring / SLA Monitor による静的ルート冗長化**:
   * デフォルトルートを複数用意する場合、IP SLA (SLA Monitor) でネクストホップの ICMP 応答を監視し、ISP 障害時に自動でバックアップルート（Floating Static Route）へ切り替わる構成を標準とします。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA: サブインターフェイス (802.1Q) を使用した Routed Mode DMZ 分離構成

#### 問題
ASA1 の `GigabitEthernet0/1` インターフェイス上に 2 つの VLAN サブインターフェイスを作成せよ。VLAN 10 宛てを `inside`（IP: `10.1.10.254/24`, Security Level: 100）、VLAN 20 宛てを `dmz`（IP: `172.16.20.254/24`, Security Level: 50）として構成せよ。

#### 要件
* 物理ポート `GigabitEthernet0/1` は IP アドレスを持たせず有効化すること。
* サブインターフェイス識別子は VLAN ID と一致させること。

#### 設定例

```bash
! 物理ポートの有効化
interface GigabitEthernet0/1
 description Trunk Link to Core Switch
 no ip address
 no shutdown
!
! Inside VLAN サブインターフェイス
interface GigabitEthernet0/1.10
 description Inside Trusted Subnet
 vlan 10
 nameif inside
 security-level 100
 ip address 10.1.10.254 255.255.255.0
 no shutdown
!
! DMZ VLAN サブインターフェイス
interface GigabitEthernet0/1.20
 description DMZ Web Server Subnet
 vlan 20
 nameif dmz
 security-level 50
 ip address 172.16.20.254 255.255.255.0
 no shutdown
```

#### 検証方法
* `show interface ip brief` を実行し、`Gi0/1.10` および `Gi0/1.20` が `YES CONFIG manual up up` であることを確認。
* `show route` を実行し、`10.1.10.0/24` および `172.16.20.0/24` が Directly Connected としてルーティングテーブルに存在することを確認。

---

### 例2: FTD (FMC): Routed Mode インターフェイスおよび Security Zone 構成

#### 問題
FMC を使用して FTD1 の `GigabitEthernet0/0` を `inside-routed` (IP: `10.0.1.254/24`)、`GigabitEthernet0/1` を `outside-routed` (IP: `203.0.113.2/24`) として構成し、それぞれの Security Zone に紐付けよ。

#### 要件
* Security Zone 名は `SZ_INSIDE_R` および `SZ_OUTSIDE_R` とすること。

#### 設定例

```text
1. FMC: Objects > Object Management > Security Zones > Add Security Zone.
   - Name: SZ_INSIDE_R, Type: Routed -> Save
   - Name: SZ_OUTSIDE_R, Type: Routed -> Save
2. FMC: Devices > Device Management > FTD1 の編集 (鉛筆アイコン).
3. Interfaces > GigabitEthernet0/0 を編集:
   - Name: inside-routed
   - Enabled: Checked
   - Security Zone: SZ_INSIDE_R
   - Mode: Routed
   - IPv4: Static -> 10.0.1.254 /24 -> OK
4. Interfaces > GigabitEthernet0/1 を編集:
   - Name: outside-routed
   - Enabled: Checked
   - Security Zone: SZ_OUTSIDE_R
   - Mode: Routed
   - IPv4: Static -> 203.0.113.2 /24 -> OK
5. Save & Deploy を実行。
```

#### 検証方法
* FTD CLI にて `show interface ip brief` を実行し、IP アドレス割り当てを確認。
* FTD CLI にて `show zone` を実行し、`inside-routed` が `SZ_INSIDE_R` に属していることを確認。

---

### 例3: ASA: Multi-Context モードでの Routed Context 作成とインターフェイス割り当て

#### 問題
ASA1 上で Multi-Context モードを有効化し、`CTX_CORP` という名前の Data Context を作成せよ。物理ポート `Gi0/1` および `Gi0/2` を `CTX_CORP` に割り当て、コンフィグ URL を `disk0:/CTX_CORP.cfg` とせよ。

#### 要件
* システムコンテキストから Context を切り替えて、`CTX_CORP` 内で IP アドレスを設定すること。

#### 設定例

```bash
! [System Context]
mode multiple
! (再起動を促された場合は指示に従う)

context CTX_CORP
 allocate-interface GigabitEthernet0/1
 allocate-interface GigabitEthernet0/2
 config-url disk0:/CTX_CORP.cfg
!
! Context へ切り替え
changeto context CTX_CORP
!
! [CTX_CORP Context]
config t
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 10.100.1.254 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/2
 nameif outside
 security-level 0
 ip address 198.51.100.2 255.255.255.0
 no shutdown
```

#### 検証方法
* システムコンテキストで `show context` を実行し、`CTX_CORP` のステータスが `Active` であることを確認。
* `changeto context CTX_CORP` 後に `show interface ip brief` を実行し、コンテキスト固有のポートと IP を確認。

---

### 例4: FTD (FMC): OSPFv2 による動的ルーティングとデフォルトルート注入

#### 問題
FTD1 の `inside-routed` インターフェイスにおいて OSPF Process ID `1`、Area `0` を有効化し、対向ルータと隣接関係（Adjacency）を確立せよ。また、外部へのデフォルトルートを OSPF 内へ条件なし（`Always`）で再配布（Originate）せよ。

#### 要件
* Router ID は `10.0.1.254` に手動固定すること。

#### 設定例

```text
1. FMC: Devices > Device Management > FTD1 の編集 > Routing > OSPF.
2. Process アプリケーション:
   - Process ID: 1
   - Router ID: 10.0.1.254
3. Area タブ > Add Area:
   - Area ID: 0 (or 0.0.0.0)
   - Interface: inside-routed を追加 -> OK
4. Default Route Advertise タブ:
   - Offer Default Route: Checked
   - Always: Checked
   - Metric: 20, Metric Type: Type 2 -> OK
5. Save & Deploy.
```

#### 検証方法
* FTD CLI にて `show ospf neighbor` を実行し、対向ルータとの State が `FULL/DR` または `FULL/BDR` であることを確認。
* FTD CLI にて `show route ospf` を確認。

---

### 例5: FTD (FMC): Policy-Based Routing (PBR) による特定トラフィックの宛先制御

#### 問題
FTD1 の `inside-routed` インターフェイスに入力される HTTP トラフィック（TCP 80）のみ、デフォルトルートに関わらず別プロキシ gateway（`10.0.1.100`）へ PBR で転送せよ。ただし内部 LAN 宛通信（`10.0.0.0/8`）は PBR 対象外とせよ。

#### 要件
* Extended ACL オブジェクト `ACL_PBR_HTTP` を作成して利用すること。

#### 設定例

```text
1. FMC: Objects > Object Management > Access List > Extended > Add Extended Access List.
   - Name: ACL_PBR_HTTP
   - Rule 1: Action: Deny, Source: any, Destination: 10.0.0.0/8, Port: HTTP (tcp/80)
   - Rule 2: Action: Allow, Source: any, Destination: any, Port: HTTP (tcp/80)
2. FMC: Devices > Device Management > FTD1 > Routing > Policy Based Routing > Add.
   - Ingress Interface: inside-routed
   - Match ACL: ACL_PBR_HTTP
   - Send To (Next Hop): IPv4 Unicast Address -> 10.0.1.100
3. Save & Deploy.
```

#### 検証方法
* FTD CLI にて `show running-config route-map` を実行し、生成されたルートマップを確認。
* FTD CLI にて `show route-map` を実行し、PBR にマッチしたパケットポリシーカウンタの増加を確認。

---

### 例6: ASA/FTD: SLA Monitor による Floating Static Route 冗長構成

#### 問題
Outside 主回線 (GW: `203.0.113.1`, Priority 1) および 副回線 (GW: `198.51.100.1`, Priority 10) の 2 つのデフォルトルートを構成せよ。主回線のゲートウェイ `203.0.113.1` 宛に 3 秒間隔で ICMP Echo を送信する SLA Monitor を構成し、応答が途絶えたら自動的に副回線へ切り替えよ。

#### 要件
* SLA ID: `100`, Track ID: `1`

#### 設定例 (ASA CLI / FTD FlexConfig 適用例)

```bash
! SLA Monitor の定義
sla monitor 100
 type echo protocol ipIcmpEcho 203.0.113.1 interface outside
 timeout 1000
 frequency 3
!
sla monitor schedule 100 life forever start-time now
!
! オブジェクトトラッキングの定義
track 1 rtr 100 reachability
!
! トラッキング付き主デフォルトルート
route outside 0.0.0.0 0.0.0.0 203.0.113.1 1 track 1
!
! バックアップ用 Floating Static Route (Metric 10)
route backup-out 0.0.0.0 0.0.0.0 198.51.100.1 10
```

#### 検証方法
* `show track` を実行し、State が `Up` であることを確認。
* 主回線を物理切断または ICMP をドロップさせ、`show track` が `Down` に変化し、`show route` の `0.0.0.0/0` が `198.51.100.1` に切り替わることを確認。

---

### 例7: FTD (FMC): ECMP (Equal-Cost Multi-Path) を用いた複数 ISP への負荷分散

#### 問題
FTD1 において、2 つの Outside インターフェイス (`outside1`: GW `203.0.113.1`, `outside2`: GW `198.51.100.1`) を同一の Zone 内に配置し、Equal-Cost Multi-Path (ECMP) によるトラフィック分散を構成せよ。

#### 要件
* 両方のデフォルトルートの Metric を `1` とすること。

#### 設定例

```text
1. FMC: Devices > Device Management > FTD1 の編集 > Routing > ECMP.
2. Enable ECMP にチェックを入れる。
3. Add ECMP Zone:
   - Name: ECMP_Zone_Outside
   - Interfaces: outside1, outside2 を追加して OK。
4. Routing > Static Route にて 2 つのデフォルトルートを追加:
   - Route 1: Interface: outside1, Network: any-ipv4, GW: 203.0.113.1, Metric: 1
   - Route 2: Interface: outside2, Network: any-ipv4, GW: 198.51.100.1, Metric: 1
5. Save & Deploy.
```

#### 検証方法
* FTD CLI にて `show route` を実行し、`0.0.0.0/0` に対して 2 つの ECMP パス（`via 203.0.113.1, outside1` および `via 198.51.100.1, outside2`）が表示されることを確認。

---

### 例8: FTD (FMC): FMC Access on Data Interface の完全構成手続き

#### 問題
FTD1 の Dedicated Management ポート障害に備え、データポート `outside-routed` (IP: `203.0.113.2/24`) を使用して FMC (IP: `10.0.0.50`) との sftunnel 通信を行えるよう構成せよ。

#### 要件
* FTD コンソール CLI と FMC GUI の双方で必要な不一致解除（Acknowledge）を行うこと。

#### 設定例

```bash
! 1. FTD コンソール CLI での設定 (管理ゲートウェイを data-interfaces へ送出)
> configure network ipv4 manual 192.168.1.100 255.255.255.0 data-interfaces

! 2. FMC GUI でのインターフェイス設定
! Devices > Device Management > Interfaces > outside-routed を編集
! 「Enable FMC Access」にチェックを入れて保存

! 3. FMC GUI での Acknowledge 実行
! Devices > Device Management > Device > Management > FMC Access Details
! 「Acknowledge」をクリックして配備ブロック（Deployment Block）を解除

! 4. FMC から Deploy 実行
```

#### 検証方法
* FTD CLI にて `sftunnel-status-brief` を実行し、`nlp_int_tap` インターフェイス経由で FMC とのステータスが `CONNECTED` であることを確認。

---

### 例9: ASA/FTD: Routed Mode における Bridge Group (BVI) による Integrated Routing and Bridging (IRB) 構成

#### 問題
Routed Mode で動作している ASA1 において、`GigabitEthernet0/1` と `GigabitEthernet0/2` を同一 L2 ブリッジグループ `1` に所属させ、BVI1 に IP アドレス `10.1.1.254/24` を割り当てよ。他の Routed ポート (`Gi0/0` - `outside`) との間で L3 ルーティングを行わせよ。

#### 要件
* Bridge Group 名を `inside-bridge` と定義すること。

#### 設定例

```bash
! メンバーポートの Bridge Group バインド
interface GigabitEthernet0/1
 description Switch 1 Connection
 nameif inside-p1
 bridge-group 1
 no shutdown
!
interface GigabitEthernet0/2
 description Switch 2 Connection
 nameif inside-p2
 bridge-group 1
 no shutdown
!
! Bridge Virtual Interface (BVI) の作成と L3 アドレス設定
interface BVI1
 description Inside Default Gateway
 nameif inside-bridge
 security-level 100
 ip address 10.1.1.254 255.255.255.0
 no shutdown
```

#### 検証方法
* `show interface bvi 1` を実行し、BVI1 が UP 状態であることを確認。
* `Gi0/1` 側の端末および `Gi0/2` 側の端末から `10.1.1.254` への ping 応答、および Outside 方面への L3 ルーティング疎通を確認。

---

### 例10: FTD (FMC): Routed Mode における Auto NAT (Section 2) および Manual Twice NAT (Section 1) の同時構成

#### 問題
FTD1 において、内部サブネット `10.1.1.0/24` から外部への通信を Outside インターフェイス IP で動的 PAT 変換する Auto NAT ルールを作成せよ。同時に、内部 DMZ Web サーバ（`172.16.50.10`）から外部 DNS サーバ（`8.8.8.8`）宛ての通信のみ、例外的に Static PAT（変換後 IP: `203.0.113.10`）を行う Manual Twice NAT ルールを Section 1（Before Auto）に作成せよ。

#### 要件
* FMC の NAT Policy 画面で設定すること。

#### 設定例

```text
1. FMC: Devices > NAT > New Policy > Threat Defense NAT Policy.
   - Name: FTD1_NAT_Policy, Assign Device: FTD1 -> Save
2. Auto NAT (Section 2) の追加:
   - Add NAT Rule > Auto NAT Rule
   - Type: Dynamic, Source Interface: SZ_INSIDE_R, Destination Interface: SZ_OUTSIDE_R
   - Original Source: Network Object "NET_INSIDE_10.1.1.0_24"
   - Translated Source: Destination Interface IP -> OK
3. Manual Twice NAT (Section 1 - Before Auto) の追加:
   - Add NAT Rule > Manual NAT Rule
   - Position: Before Auto NAT (Section 1)
   - Type: Static
   - Source Interface: SZ_DMZ_R, Destination Interface: SZ_OUTSIDE_R
   - Original Source: Host Object "HOST_WEB_172.16.50.10"
   - Translated Source: Host Object "HOST_PUB_203.0.113.10"
   - Original Destination: Host Object "HOST_DNS_8.8.8.8"
   - Translated Destination: Host Object "HOST_DNS_8.8.8.8" (Identity)
   - Original Service: Domain (udp/53), Translated Service: Domain (udp/53) -> OK
4. Save & Deploy.
```

#### 検証方法
* FTD CLI にて `show nat` を実行し、Section 1 に Manual NAT、Section 2 に Auto NAT が意図した順序で配置されていることを確認。
* 実際に通信を発生させ、`show xlate` で変換エントリを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: FTD CLI にて `show route` を実行したところ、以下の出力が得られた。このデバイスのルーティングテーブルに関する正しい説明はどれか。

```text
> show route

Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 

S*    0.0.0.0/0 [1/0] via 203.0.113.1, outside1
                [1/0] via 198.51.100.1, outside2
C     10.1.1.0/24 is directly connected, inside
L     10.1.1.254/32 is directly connected, inside
```

* A) 230 のアドミニストレーティブディスタンスを持つバックアップルートが設定されている。
* B) `outside1` と `outside2` 間で Equal-Cost Multi-Path (ECMP) が有効化されており、トラフィックが負荷分散されている。
* C) PBR (Policy-Based Routing) が適用されているため、このルーティングテーブルは無視される。
* D) `10.1.1.254` は BVI インターフェイスのアドレスであり、デバイスは Transparent Mode で動作している。

**正解**: **B**
**解説**: `0.0.0.0/0` に対し、同じメトリック `[1/0]` で 2 つの異なる Egress インターフェイス (`outside1`, `outside2`) とネクストホップが表示されているため、ECMP Zone による負荷分散が機能しています。

---

### 2. トラブルシュート問題

**問題**: FTD のデータポート（`outside`）経由で FMC 管理を行うため、FTD CLI で `configure network ipv4 manual 192.168.1.50 255.255.255.0 data-interfaces` を実行し、FMC 上で `Enable FMC Access` を有効化した。しかし、FMC からポリシーをデプロイしようとすると「Deployment Blocked due to interface configuration mismatch」エラーが発生してデプロイできない。正しい復旧手順はどれか。

* A) FTD CLI で `configure firewall transparent` を実行してモードを初期化する。
* B) FTD CLI で `configure manager delete` を行い、FMC に再登録する。
* C) FMC GUI の `Devices > Device Management > Device > Management > FMC Access Details` 画面に移動し、構成の不一致を確認した上で `Acknowledge` ボタンをクリックしてから再 Deploy する。
* D) FTD を再起動して CLI の変更を破棄させる。

**正解**: **C**
**解説**: データポート管理設定を FTD ローカル CLI で変更すると FMC の DB と差分が生じ、誤設定による通信断を防ぐためにデプロイが自動ブロックされます。FMC GUI の `FMC Access Details` で差分を認識（Acknowledge）することでブロックが解除されます。

---

### 3. Design 問題

**問題**: 企業ネットワークの設計において、FTD を Routed Mode で導入予定である。内部ネットワーク宛ての通信（`10.0.0.0/8`）は直接内部ルータへルーティングし、インターネット宛ての Web 通信（TCP 80/443）のみをセキュリティプロキシ（`192.168.100.1`）へ強制的に転送したい。最も拡張性が高く適切な実装方法はどれか。

* A) PBR (Policy-Based Routing) を構成し、Extended ACL で `10.0.0.0/8` 宛て通信を `deny`（除外）し、それ以外の Web 通信を `permit` してプロキシ IP をネクストホップに指定する。
* B) デフォルトルートのネクストホップをプロキシ IP に変更し、`10.0.0.0/8` 宛てのスタティックルートを削除する。
* C) FTD を Transparent Mode に変更し、プロキシと内部ルータの間にインライン挿入する。
* D) Pre-filter Policy で Web 通信を Fastpath に指定する。

**正解**: **A**
**解説**: 特定のアプリケーションポート（TCP 80/443）のみを別のネクストホップへ曲げつつ、指定の内部宛先を除外するには、Extended ACL で拒否条件を入れた PBR (Policy-Based Routing) を適用するのが最も正確で標準的な設計です。

---

### 4. 実装問題

**問題**: ASA1 を Single-Context Routed Mode から Multi-Context モードへ移行するため、CLI で `mode multiple` コマンドを実行した。この操作によって発生する動作として正しいものはどれか。

* A) 既存の Single-Context 設定が自動的に Admin Context へ変換され、無停止で移行が完了する。
* B) デバイスの再起動が要求され、再起動後に既存の Single-Context 設定は消去（初期化）され、システム実行スペース（System Execution Space）で起動する。
* C) モードが Transparent Mode に自動変更される。
* D) すべての物理インターフェイスが Bridge Group に自動的に追加される。

**正解**: **B**
**解説**: ASA で `mode multiple` を実行すると再起動が必要となり、既存の単一コンフィグは消去され、システムコンテキストとデフォルトの admin コンテキストが生成された初期状態で起動します。

---

### 5. コンフィグ読解問題

**問題**: ASA の設定で以下のコマンドが定義されている。この設定の動作として正しいものはどれか。

```bash
same-security-traffic permit inter-interface
same-security-traffic permit intra-interface
```

* A) 異なる Security Level を持つインターフェイス間でのみ通信を許可する。
* B) 同じ Security Level を持つ異なるインターフェイス間（inter-interface）の通信、および同一インターフェイス内での折り返し通信（intra-interface）を許可する。
* C) ACL の評価を完全にバイパスしてすべてのトラフィックを通過させる。
* D) Transparent Mode でのみ動作し、BVI 間の通信を有効化する。

**正解**: **B**
**解説**: ASA のデフォルト制限を解除し、同じセキュリティレベル値（例: 50 と 50）を持つインターフェイス同士のルーティング（inter-interface）や、同一ポートに入って同じポートから出ていくヘアピン通信（intra-interface）を許可する設定です。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Threat Defense Configuration Guide (FMC) - Interfaces](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/routed_mode_interfaces.html)
* [Cisco ASA Series General Operations Configuration Guide - Firewall Mode](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/general/asa-94-general-config/firewall-mode.html)
* [Cisco TechNotes: Configuring Firepower Threat Defense Interfaces in Routed Mode](https://www.cisco.com/c/en/us/support/docs/security/firepower-ngfw/200908-configuring-firepower-threat-defense-int.html)
* [Cisco Live: Deep Dive into Secure Firewall Architecture and Routing (BRKSEC-3020)](https://www.ciscolive.com/)
* [Cisco Validated Design (CVD): Secure Firewall Deployment Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/cisco-secure-firewall-design-guide.html)

---

## 📝 補足（Notes）

* **学習メモ**:
  * Routed Mode は Cisco ファイアウォールの基本中の基本です。パケットが「1. Conn Table Lookup → 2. Pre-filter → 3. ACL/ACP → 4. NAT → 5. Snort 3 → 6. Route Lookup」の順で処理されるパイプラインを頭の中に描けるようにしてください。
  * 特に FTD FMC 環境での **Security Zone** の扱い（Routed Zone / Transparent Zone / Inline Zone 等のタイプ不一致による設定不可）に注意しましょう。
* **注意点**:
  * CCIE ラボ試験中に `configure firewall transparent` や `mode multiple` などのモード変更コマンドを実行すると、**既存設定がすべてクリアされる** ため、必ず試験開始直後の初期構成フェーズで実行してください。

## 📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

