---
layout: default
title: 1.1.b-Transparent
nav_order: 2
parent: 1.1-Deployment-modes-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.1.b Transparent

Cisco ASA (Adaptive Security Appliance) および Cisco FTD (Firepower Threat Defense) における **Transparent Mode（トランスペアレントモード / レイヤ2モード）** は、デバイスをネットワーク上に Layer 2 ブリッジ（「Bump in the wire」または「ステルスファイアウォール」）として透過的に挿入し、既存の IP アドレス設計やルーティング構成を変更することなく L2〜L7 のセキュリティスキャン・アクセス制御を実行する機能です。

---

## 📘 概要

* **機能概要**:
  * 接続されたインサイドおよびアウトサイドの物理/論理インターフェイスを同一の L3 サブネット（VLAN 間ブリッジ）に配置し、デバイス自体をルータホップとして見せることなくトラフィックを透過転送・検査します。
  * **Bridge Group（ブリッジグループ）** と **BVI (Bridge Virtual Interface)** を使用して、L2 セグメント間のトラフィック分離とアプライアンス自体の管理 IP アドレスを保持します。
* **利用目的**:
  * 既存ネットワークの IP サブネット再設計やデフォルトゲートウェイ変更を行わずに、インラインセキュリティ検査（NGFW / NGIPS）を挿入・追加する。
  * 攻撃者からファイアウォール自体の IP アドレスを隠蔽し、直接の脅威攻撃対象となるリスクを低減（ステルス化）する。
  * ルーテッドモードでは通常遮断される非 IP トラフィック（EtherType / BPDU / MPLS / AppleTalk 等）の制御や、透過的なマルチキャスト / 冗長ルーティングプロトコル（OSPF, EIGRP, BFD 等）の通過を制御する。
* **どのような場面で利用するか**:
  * データセンター内のコアスイッチとサーバーファーム / データセンタールータ間へのインライン挿入。
  * 既存のネットワークトポロジが固化しており、IP アドレス変更のメンテナンスウィンドウが取れない環境へのセキュリティ強化。
  * 管理アクセス以外の通信で L3 インターフェイスを露出させたくない高セキュリティ領域。

---

## 🔑 要点

| 項目 | Cisco ASA (Transparent Mode) | Cisco FTD (Transparent Mode / FMC 管理) |
| :--- | :--- | :--- |
| **L2/L3 動作** | L2 ブリッジング（異なる VLAN 間の透過ブリッジ） | L2 ブリッジング（異なる VLAN 間の透過ブリッジ） |
| **IP アドレス割り当て** | インターフェイスには IP 不要。Bridge Group ごとに **BVI** へ管理 IP を割当 | インターフェイスには IP 不要。Bridge Group ごとに **BVI** へ管理 IP を割当 |
| **ブリッジグループ上限** | 最大 250 Bridge Group (1 グループあたり最大 4 インターフェイス) | 最大 250 Bridge Group (1 グループあたり最大 64 インターフェイス) |
| **MAC 学習 & 検索** | ダイナミック MAC 学習、Static MAC、MAC Learning Disable をサポート | ダイナミック MAC 学習、Static MAC、MAC Learning Disable をサポート |
| **EtherType ACL** | サポート（BPDU, IPX, MPLS 等の L2 制御） | サポート（FlexConfig オブジェクト経由で EtherType ACL 適用） |
| **ARP 制御** | ARP パケットはデフォルト通過。**ARP Inspection** で検証・ドロップ可 | ARP パケットはデフォルト通過。**ARP Inspection** で検証・ドロップ可 |
| **NAT サポート** | ASA 8.0+ より Bridge Group 内 NAT サポート | Bridge Group 内での Static / Dynamic NAT をサポート |
| **マルチテナント化** | Multi-Context モード（MMTF: Multimode Transparent）サポート | FXOS Multi-Instance (Container) によりハードウェアレベル分離 |

---

## 🏗 動作原理

Transparent Mode では、パケットは Layer 2 フレームとして受信され、宛先 MAC アドレスルックアップとステートフルインスペクション（LINA + Snort）を経て転送されます。

```text
[ Client / Switch 1: VLAN 10 ]
         │
         ▼ (Ingress Port: GigabitEthernet0/0 - Bridge Group 1 / VLAN 10)
┌────────────────────────────────────────────────────────────────────────┐
│ Cisco FTD / ASA (Transparent Mode)                                     │
│                                                                        │
│ 1. L2 MAC Table Lookup & Ingress Inspection                            │
│    ├── Frame Decapsulation & Ingress Interface Check                   │
│    ├── EtherType ACL / BPDU Check                                      │
│    └── Destination MAC Address Lookup in Bridge Group MAC Table        │
│                                                                        │
│ 2. LINA & Snort Stateful Inspection Engine                             │
│    ├── Connection Table Lookup (5-tuple Match)                         │
│    ├── Access Control Policy (ACP) / L3-L7 Inspection                  │
│    ├── Security Intelligence / IPS / File & AMP (FTD)                  │
│    └── Stateful Flow Creation & TCP Normalization                      │
│                                                                        │
│ 3. L2 Egress Forwarding                                                │
│    ├── Egress Interface Determination via MAC Table                    │
│    └── Frame Encapsulation & Forwarding out Bridge Group Pair          │
│                                                                        │
│ * BVI 1 (IP: 10.1.1.254/24) - 管理通信・SYSLOG・AAA 送信元専用         │
└────────────────────────────────────────────────────────────────────────┘
         │
         ▼ (Egress Port: GigabitEthernet0/1 - Bridge Group 1 / VLAN 20)
[ Default Gateway Router / Switch 2: VLAN 20 (Same 10.1.1.0/24 Subnet) ]
```

---

## ⚙ 動作シーケンス

Transparent Mode におけるパケット処理シーケンスは以下のロジックで実行されます。

1. **Layer 2 フレーム受信**:
   * 入力インターフェイスで L2 フレームを受信。送信元 MAC アドレスをブリッジグループの MAC アドレス表に動的学習（有効時）。
2. **EtherType ルール & ARP 評価**:
   * 非 IP パケットの場合、EtherType ACL（BPDU, MPLS 等）を照合。
   * ARP パケットの場合、デフォルトで全バインドポートへ通過。ARP Inspection 有効時は Static ARP テーブルとの整合性を検証。
3. **ステートフルセッション照会 (Conn Table Lookup)**:
   * IP パケットの場合、既存の L3/L4 セッションテーブルを検索。既存フローに一致する場合は ACL チェックをスキップ。
4. **Access Control Policy (ACP) / ACL 判定**:
   * 新規セッションの場合、L3/L4/L7 のアクセスルールを評価。
   * インサイド（高セキュリティ）からアウトサイド（低セキュリティ）へのユニキャスト IP 通信はデフォルト許可（ASA）または ACP ルールに従い処理。
5. **Snort インスペクション (FTD の場合)**:
   * スキャン対象トラフィックは Snort 3 エンジンへ渡され、IPS・ファイル・URL・セキュリティインテリジェンスの検証を実施。
6. **Egress ポート判定と転送 (MAC Address Lookup)**:
   * ルート検索（Route Lookup）ではなく、**宛先 MAC アドレスルックアップ** によって同一ブリッジグループ内の出力ポートを決定。
   * 宛先 MAC アドレスが未知の場合、正常なスイッチのように無条件フラッディングするのではなく、アプライアンスから **ターゲット IP への ARP 要求（直結機器宛て）** または **ICMP Ping（リモート宛て）** を自身で生成して学習を試み、元の未認知パケットはドロップします。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

* **FMC / CLI でのモード変更に伴う「全設定消去」**:
  * ASA CLI や FTD CLI で `configure firewall transparent` を実行すると、**すべてのインターフェイス構成・IP・ポリシーが即座に消去** されます。FTD の場合は FMC からの登録解除・再登録が必要です。
* **BVI サブネット設計と /30, /32 サブネットの禁止**:
  * BVI の IP アドレスはブリッジグループメンバーと同じ L3 サブネットに配置する必要があります。
  * **ホスト数 3 未満のサブネット（/30 や /32）は BVI に使用不可** です。アプライアンスはサブネットのネットワークアドレスおよびブロードキャストアドレス宛の ARP をドロップするため、アップストリーム/ダウンストリームルータとの ARP 解決に失敗します（最低でも 3 つのホスト IP が必要: 上流ルータ, 下流ルータ, BVI）。
* **ルート検索が必要となる特殊条件**:
  * Transparent Mode では基本 MAC ルックアップで転送されますが、以下の場合は **L3 ルーティングテーブル（スタティックルート）の検索が必要** になります。
    1. アプライアンス自身から発生するトラフィック（Syslog, AAA, SNMP, FMC 通信等）の戻りリスボンス。
    2. 音声（VoIP）や TFTP などのインスペクションが有効で、エンドポイントが 1 ホップ以上先にある場合のピンホール作成。
    3. TLS 1.3 や SSL プロキシ機能動作時、アプライアンス自体が TCP ACK キープアライブを生成する場合。
* **BPDUs と Spanning Tree 制御**:
  * スパニングツリーのループ防止のため、BPDU はデフォルトで透過転送されます。
  * Failover 構成においてトポロジ変化時にスイッチポートが Blocking 状態になるのを防ぎたい場合、EtherType ACL で BPDU を Block するか、FlexConfig を使用して BPDU を信頼・インスペクション除外（Trust BPDU）にする設定が求められます。
* **無効な MAC Learning と Dynamic ARP Validation**:
  * CAM テーブル溢れ攻撃（CAM Overflow Attack）から保護するため、`mac-learn <interface> disable` コマンドで MAC 学習を停止させ、Static MAC テーブルを登録する問題が出題されます。

---

## 🛠 設定方法

### 1. Cisco ASA: Transparent Mode の基本設定と Bridge Group / BVI 構成 (CLI)

```bash
! 1. Transparent Mode への変更 (既存設定クリア注意)
configure firewall transparent

! 2. BVI インターフェイスの作成と IP アドレス設定
interface BVI1
 description Management BVI for Bridge Group 1
 ip address 10.1.1.254 255.255.255.0
 ipv6 enable

! 3. インサイドおよびアウトサイドポートの Bridge Group バインド
interface GigabitEthernet0/0
 description Inside Segment (VLAN 10)
 nameif inside
 security-level 100
 bridge-group 1
 no shutdown
!
interface GigabitEthernet0/1
 description Outside Segment (VLAN 20)
 nameif outside
 security-level 0
 bridge-group 1
 no shutdown

! 4. アプライアンス自身の管理通信用デフォルトルート設定 (上流ルータ宛て)
route outside 0.0.0.0 0.0.0.0 10.1.1.1
```

### 2. Cisco ASA: EtherType ACL による BPDU および MPLS の通過許可 (CLI)

```bash
! EtherType ACL の作成 (BPDU および MPLS を許可)
access-list ETH_ALLOW ethertype permit bpdu
access-list ETH_ALLOW ethertype permit mpls-unicast

! インターフェイスへの適用
access-group ETH_ALLOW in interface outside
access-group ETH_ALLOW in interface inside
```

### 3. Cisco FTD: CLI での Transparent Mode 変更と FMC 登録手順

```bash
! 1. FTD コンソール CLI で既存マネージャ削除
> configure manager delete

! 2. Firewall Mode を Transparent に変更
> configure firewall transparent
This will destroy the current interface configurations, are you sure that you want to proceed? [y/N] y
The firewall mode was changed successfully.

! 3. FMC へ再登録
> configure manager add 10.1.1.50 RegistrationKey123
```

### 4. Cisco FTD (FMC GUI): Bridge Group および BVI の構成

1. **Bridge Group の追加**:
   * **Devices > Device Management** で対象 FTD の編集（鉛筆アイコン）をクリック。
   * **Interfaces** タブで **Add Interfaces > Bridge Group Interface** を選択。
2. **メンバーポートおよび BVI のパラメタ設定**:
   * **Bridge Group ID**: `1` (1 〜 250)。
   * **Interfaces** タブ: 利用可能なポート（例: `GigabitEthernet0/0`, `GigabitEthernet0/1`）を選択し **Selected Interfaces** に移動。
   * **IPv4** タブ: `10.1.1.254/24` (BVI IP アドレス) を入力。
3. **メンバーポートの Security Zone 設定**:
   * 各物理ポートの編集を開き、`inside-z` (Transparent タイプ) および `outside-z` (Transparent タイプ) の Security Zone を割り当て。
4. **Save & Deploy**:
   * 設定を保存し、FTD へ Deploy を実行。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **ファイアウォールモード確認** | ASA / FTD CLI | <code>show firewall</code> |
| **Bridge Group 構成・ステータス表示** | ASA / FTD CLI | <code>show bridge-group</code> |
| **MAC アドレス表（CAM Table）確認** | ASA / FTD CLI | <code>show mac-address-table</code> |
| **BVI インターフェイス詳細確認** | ASA / FTD CLI | <code>show interface bvi 1</code> |
| **ARP キャッシュ一覧確認** | ASA / FTD CLI | <code>show arp</code> |
| **ARP Inspection 設定および統計確認** | ASA / FTD CLI | <code>show running-config arp-inspection</code> |
| **EtherType ACL 状態確認** | ASA CLI | <code>show access-list</code> |
| **ARP / MAC テーブルのデバッグ** | ASA CLI | <code>debug arp-inspection</code> / <code>debug mac-address-table</code> |
| **パケットフロー追跡** | FTD / ASA CLI | <code>packet-tracer input inside tcp 10.1.1.10 12345 10.1.1.20 80 detail</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **通過トラフィックが一切導通しない** | メンバーポートが同一 Bridge Group にバインドされていない、または BVI IP 未設定 | ASA/FTD: <code>show bridge-group</code> | 全データポートが正しい Bridge Group ID に属し、BVI に有効な IP (/24等) が設定されているか確認。 |
| **対向ルータ間で ARP 解決が失敗する** | BVI の IP サブネットに /30 や /32 が使用されている | ASA/FTD: <code>show interface bvi 1</code> | /30 サブネットはネットワーク/ブロードキャストアドレス処理の仕様上不可。/29 以上のサブネットに変更する。 |
| **Syslog や AAA などの管理トラフィックが不達** | L3 スタティックルート（デフォルトルート）の未設定 | ASA/FTD: <code>show route</code> | アプライアンス自身の送信トラフィック用に、メンバーポートを指定した L3 スタティックルートを設定する。 |
| **スイッチ間で BPDU が通過せず STP ループが起きる** | EtherType ACL または Access Policy で非 IP がブロックされている | ASA: <code>show access-list</code> | <code>access-list ETH ethertype permit bpdu</code> を作成し、インバウンドに適用する。 |
| **スイッチポートが "UDLD Neighbor Mismatch" で落ちる** | VSS/vPC スイッチ間に配置され UDLD パケットが重複受信 | スイッチ側: <code>show interfaces status err-disabled</code> | FTD を挟む EtherChannel ポートで UDLD を無効化（disable）する。 |

---

## ⚠ 制限事項

* **非サポート機能**:
  * **DHCP Relay**: 透過モードではアプライアンス自体が DHCP Relay になれません（ただし ACL で DHCP トラフィックの透過通過は可能）。
  * **Remote Access VPN (AnyConnect) の終端**: 透過モードの BVI やデータポートで Remote Access VPN を終端することはできません（Site-to-Site VPN は管理目的のみ、または透過通過のみ）。
  * **動的ルーティングプロトコルの終端**: BVI 自体が OSPF/BGP 隣接関係のピア（Peer）になることは不可（透過通過は ACL 許可で可能）。
  * **マルチキャストルーティング**: デバイス自体による PIM/IGMP ルーティング処理は不可（透過通過は可能）。
* **BVI への Access Policy 非適用**:
  * FMC や ASA において、BVI インターフェイス自体を Security Zone に追加して ACP や ACL を直接適用することはできません。ポリシーは常に Bridge Group のメンバーポートに対して適用します。

---

## 🔄 他技術との関連

* **1.2 Access Control & Pre-filter Policies**: Bridge Group メンバーポートの Security Zone を評価条件として、透過 IP トラフィックに L3-L7 ポリシーを適用。
* **3.1 NAT**: Transparent Mode 内であっても、同一 Bridge Group のメンバーポート間を通過するトラフィックに対して Static / Dynamic NAT を適用可能。
* **1.3 High Availability & Clustering**:
  * ASA Transparent Mode では Active/Standby Failover および Multimode Transparent (MMTF) がサポートされます。
* **FlexConfig Policy (FTD)**: FMC の標準 UI でサポートされない EtherType ACL や BPDU 除外コマンドを FTD へ投入する際に利用。

---

## 🧩 比較表

### 1. Routed Mode vs Transparent Mode

| 項目 | Routed Mode | Transparent Mode |
| :--- | :--- | :--- |
| **ネットワーク配置** | L3 ルータホップ（サブネット境界） | L2 ブリッジ（Bump in the wire） |
| **IP サブネット設計** | 各ポートが異なる L3 サブネット | メンバーポートおよび BVI が同一 L3 サブネット |
| **宛先決定アルゴリズム** | L3 ルーティングテーブル（FIB）検索 | **宛先 MAC アドレス表** 検索 |
| **非 IP パケット制御** | ドロップ（IP パケットのみ処理） | **EtherType ACL** により BPDU / MPLS 等通過可能 |
| **VPN 終端** | Remote Access / IPsec VPN フルサポート | 管理用 Site-to-Site のみ（Through 通信は透過のみ） |

### 2. ASA Transparent vs FTD Transparent

| 項目 | Cisco ASA (Transparent Mode) | Cisco FTD (Transparent Mode / FMC) |
| :--- | :--- | :--- |
| **設定インターフェース** | CLI (Config t) / ASDM | FMC (Firepower Management Center) / FDM |
| **スキャンエンジン** | LINA エンジン（＋モジュールで FirePOWER） | LINA + Snort 3 / Snort 2 が完全統合 |
| **1グループあたりのポート数** | 最大 4 インターフェイス | **最大 64 インターフェイス** |
| **EtherType ACL 設定** | コマンドで直接適用 (`access-list ethertype`) | **FlexConfig オブジェクト** を介して適用 |

---

## 💡 ベストプラクティス

1. **BPDU のインスペクション除外 (FlexConfig)**:
   * スパニングツリーの安定性を確保するため、FlexConfig を使用して EtherType ACL で BPDU を信頼し、Snort の高度なインスペクションから免除（Exempt）します。
2. **BVI サブネットの広さ確保**:
   * BVI アドレスには必ず /29 以上のサブネット（例: /24）を使用し、アップストリーム/ダウンストリームルータとの ARP 競合や不達事故を防ぎます。
3. **管理トラフィック専用ルートの明記**:
   * 複数の Bridge Group が存在する場合、デフォルトルートは 1 つの Bridge Group にしか適用できないため、その他の管理通信送信元（Syslog/AAA サーバ等）に対して明示的な L3 スタティックルート（メンバーポート指定）を配置します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA: Single-Mode Transparent Mode の初期構成と Bridge Group / BVI 設定

#### 問題
ASA1 を Transparent Mode に変更し、`GigabitEthernet0/0` (inside, Security Level 100) と `GigabitEthernet0/1` (outside, Security Level 0) を Bridge Group 10 に設定せよ。管理用 IP アドレスとして BVI10 に `172.16.10.254/24` を割り当てよ。

#### 要件
* モード変更時の初期化に留意すること。

#### 設定例

```bash
! モード変更
configure firewall transparent

! BVI の作成
interface BVI10
 description Management BVI
 ip address 172.16.10.254 255.255.255.0
 no shutdown

! インターフェイス割り当て
interface GigabitEthernet0/0
 nameif inside
 security-level 100
 bridge-group 10
 no shutdown
!
interface GigabitEthernet0/1
 nameif outside
 security-level 0
 bridge-group 10
 no shutdown
```

#### 検証方法
* `show firewall` を実行し `Transparent` を確認。
* `show bridge-group` を実行し、`GigabitEthernet0/0` と `GigabitEthernet0/1` が Bridge Group 10 に所属していることを確認。

---

### 例2: ASA: EtherType ACL による BPDU および MPLS トラフィックの通過許可

#### 問題
ASA1 において、通常の IP トラフィックに加えて Spanning Tree BPDU パケットおよび MPLS Unicast トラフィックの通過を許可する EtherType ACL を作成し、`outside` インターフェイスに適用せよ。

#### 要件
* Access List 名は `ACL_ETHERTYPE` とすること。

#### 設定例

```bash
! EtherType ACL の定義
access-list ACL_ETHERTYPE ethertype permit bpdu
access-list ACL_ETHERTYPE ethertype permit mpls-unicast

! インターフェイスへのバインド
access-group ACL_ETHERTYPE in interface outside
access-group ACL_ETHERTYPE in interface inside
```

#### 検証方法
* `show access-list ACL_ETHERTYPE` を実行し、BPDU および mpls-unicast ルールのヒットカウンタ増加を確認。

---

### 例3: ASA: MAC スプーフィング防止のための ARP Inspection と Static ARP 構成

#### 問題
ASA1 の Bridge Group 10 において、`inside` インターフェイス側での ARP スプーフィング（中間者攻撃）を防止するため、ARP Inspection を有効化せよ。ルータ R1 (`172.16.10.1`, MAC: `0011.2233.4455`) の Static ARP エントリを作成し、一致しない ARP パケットをドロップ（no-flood）せよ。

#### 要件
* Static ARP テーブルの照合を強制すること。

#### 設定例

```bash
! 1. Static ARP エントリの作成
arp inside 172.16.10.1 0011.2233.4455

! 2. ARP Inspection の有効化 (no-flood オプションで不一致ドロップ)
arp-inspection inside enable no-flood
```

#### 検証方法
* `show running-config arp-inspection` を確認。
* `debug arp-inspection` を有効化し、不一致な ARP レスポンスがドロップされるログを確認。

---

### 例4: ASA: Dynamic MAC Address Learning の無効化と Static MAC 登録

#### 問題
CAM テーブル溢れ攻撃を防止するため、ASA1 の `outside` インターフェイスにおいて動的な MAC アドレス学習（MAC Learning）を無効化せよ。対向ルータ R2 の MAC アドレス（`00aa.bbcc.dd00`）を Bridge Group 10 の Static MAC として登録せよ。

#### 要件
* インターフェイス `outside` の MAC 学習を停止すること。

#### 設定例

```bash
! MAC 学習の停止
mac-learn outside disable

! Static MAC エントリの定義
mac-address-table static outside 00aa.bbcc.dd00 bridge-group 10
```

#### 検証方法
* `show mac-address-table` を実行し、`outside` の学習タイプが `static` であることを確認。

---

### 例5: FTD (FMC): Transparent Mode での Bridge Group および BVI 構成

#### 問題
FMC を使用して、Transparent Mode の FTD1 上に Bridge Group 1 (`GigabitEthernet0/0` と `GigabitEthernet0/1`) を作成し、BVI1 に IP アドレス `192.168.1.254/24` を割り当てよ。

#### 要件
* 物理ポートの Security Zone 名は `SZ_IN_TRANS` および `SZ_OUT_TRANS` (Transparent タイプ) とすること。

#### 設定例

```text
1. FMC: Objects > Object Management > Security Zones > Add Security Zone.
   - SZ_IN_TRANS (Type: Transparent)
   - SZ_OUT_TRANS (Type: Transparent)
2. FMC: Devices > Device Management > FTD1 の編集 > Interfaces > Add Interfaces > Bridge Group Interface.
   - Bridge Group ID: 1
   - Selected Interfaces: GigabitEthernet0/0, GigabitEthernet0/1
   - IPv4 Tab: Static -> 192.168.1.254 /24 -> OK
3. GigabitEthernet0/0 を編集: Security Zone -> SZ_IN_TRANS
4. GigabitEthernet0/1 を編集: Security Zone -> SZ_OUT_TRANS
5. Save & Deploy.
```

#### 検証方法
* FTD CLI にて `show running-config interface` を実行し、`interface BVI1` およびメンバーポートの `bridge-group 1` 構成を確認。

---

### 例6: FTD (FlexConfig): EtherType ACL による BPDU Trust オブジェクトの適用

#### 問題
Transparent Mode の FTD1 において、Spanning Tree BPDU パケットが Snort による誤検知でブロックされるのを防止するため、FlexConfig を使用して BPDU 信頼 EtherType ACL を作成し、メンバーポート `GigabitEthernet0/0` に適用せよ。

#### 要件
* FlexConfig Text Object を使用すること。

#### 設定例

```text
1. FMC: Objects > Object Management > FlexConfig > Text Object > Add Text Object.
   - Name: txt_BPDU_ACL
   - Variable Value: access-list permit-bpdu ethertype trust bpdu
2. FlexConfig Object > Add FlexConfig Object:
   - Name: Flex_BPDU_Trust
   - Deployment: Every Deploy
   - Content:
     access-list permit-bpdu ethertype trust bpdu
     access-group permit-bpdu in interface GigabitEthernet0/0
3. Devices > FlexConfig > FTD1 の FlexConfig Policy に上記オブジェクトを追加して Deploy。
```

#### 検証方法
* FTD CLI にて `show access-list` を実行し、`permit-bpdu` エントリが存在することを確認。

---

### 例7: FTD (FMC): Transparent Mode における Access Control Policy の構成

#### 問題
Transparent Mode の FTD1 において、`SZ_IN_TRANS` から `SZ_OUT_TRANS` への HTTP/HTTPS 通信のみを許可し、それ以外の通信をブロックする Access Control ルールを作成せよ。

#### 要件
* アプリケーションルール（AVC）で HTTP および HTTPS を指定すること。

#### 設定例

```text
1. FMC: Policies > Access Control > New Policy (または既存ポリシー編集).
2. Add Rule:
   - Name: Allow_Web_Transparent
   - Action: Allow
   - Source Zone: SZ_IN_TRANS
   - Destination Zone: SZ_OUT_TRANS
   - Applications: HTTP, HTTPS を追加
   - Inspection: Intrusion Policy (Balanced Security and Connectivity) をバインド -> OK
3. Deploy を実行。
```

#### 検証方法
* 接続端末から Web 閲覧を発生させ、FMC の `Analysis > Unified Events` にてアクションが `Allow` かつ Zone が判定されていることを確認。

---

### 例8: ASA/FTD: Transparent Firewall を通過する動的ルーティング (OSPF) 隣接関係の許可

#### 問題
Transparent Mode の ASA1 を挟んだ対向ルータ R1 (`10.1.1.1`) と R2 (`10.1.1.2`) の間で OSPF ピアリングを確立させよ。

#### 要件
* OSPF マルチキャストトラフィック（`224.0.0.5`）を ACL で明示的許可すること。

#### 設定例

```bash
! OSPF マルチキャストおよびユニキャストパケットを許可する ACL
access-list ACL_INBOUND extended permit ospf host 10.1.1.1 host 224.0.0.5
access-list ACL_INBOUND extended permit ospf host 10.1.1.1 host 10.1.1.2

! ACL のバインド
access-group ACL_INBOUND in interface inside
```

#### 検証方法
* 対向ルータ R1/R2 で `show ip ospf neighbor` を実行し、State が `FULL` になることを確認。

---

### 例9: ASA: 複数 Bridge Group 環境における管理トラフィック用 L3 ルート配置

#### 問題
ASA1 上に 2 つの Bridge Group (`BVI1`: `10.1.1.254/24`, `BVI2`: `10.2.2.254/24`) が存在する。Syslog サーバ (`192.168.100.50`) 宛の管理ログを `BVI2` 側のメンバーポート `outside2` から送信するための L3 スタティックルートを構成せよ。

#### 要件
* ゲートウェイ IP として `10.2.2.1` を指定すること。

#### 設定例

```bash
! 管理トラフィック用スタティックルート (ゲートウェイインターフェイスとしてメンバーポートを指定)
route outside2 192.168.100.50 255.255.255.255 10.2.2.1
```

#### 検証方法
* ASA1 から `ping 192.168.100.50` を実行し、`outside2` から ICMP パケットが送出されることを確認。

---

### 例10: ASA: Multimode Transparent Firewall (MMTF) の Context 作成と BVI 割り当て

#### 問題
ASA1 上で Multi-Context モードを有効化し、`CTX_L2_A` コンテキストを作成せよ。Transparent Mode で動作させ、`GigabitEthernet0/1` と `Gi0/2` を Bridge Group 1 にバインドし、BVI1 に `10.10.10.254/24` を割り当てよ。

#### 要件
* Multimode Transparent (MMTF) として構成すること。

#### 設定例

```bash
! [System Context]
mode multiple

context CTX_L2_A
 allocate-interface GigabitEthernet0/1
 allocate-interface GigabitEthernet0/2
 config-url disk0:/CTX_L2_A.cfg

! Context へ切替後、Transparent モード設定
changeto context CTX_L2_A
config t
firewall transparent

interface BVI1
 ip address 10.10.10.254 255.255.255.0
 no shutdown

interface GigabitEthernet0/1
 nameif inside
 bridge-group 1
 security-level 100
 no shutdown

interface GigabitEthernet0/2
 nameif outside
 bridge-group 1
 security-level 0
 no shutdown
```

#### 検証方法
* コンテキスト内で `show firewall` および `show bridge-group` を確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ASA の設定で以下のコマンドが出力されている。このアプライアンスの動作に関する正しい説明はどれか。

```bash
firewall transparent
!
interface BVI1
 ip address 10.1.1.254 255.255.255.252
!
interface GigabitEthernet0/0
 nameif inside
 bridge-group 1
!
interface GigabitEthernet0/1
 nameif outside
 bridge-group 1
```

* A) 正常に動作し、`10.1.1.254` が対向ルータのデフォルトゲートウェイとしてルーティングを行う。
* B) BVI に `/30` サブネット（255.255.255.252）が割り当てられているため、ホスト数が不足し、対向機器間の ARP パケット処理に失敗する。
* C) インターフェイス `inside` と `outside` の Security Level が未指定のため、即座にシャットダウンする。
* D) BVI に IP アドレスが設定されているため、自動的に Routed Mode へ変更される。

**正解**: **B**
**解説**: Transparent Mode の BVI に `/30` や `/32` などのホストアドレスが 3 つ未満のサブネットを割り当てると、予約アドレス宛ての ARP パケット処理仕様により対向ルータ間の ARP 解決が失敗します。

---

### 2. トラブルシュート問題

**問題**: FTD を Transparent Mode に変更後、FMC から Access Control Policy をデプロイしたが、インサイド端末から外部 Syslog サーバ (`172.16.200.10`) 宛の管理ログが全く届かない。`mail` や `syslog` 通信はポリシーで許可されている。原因として最も可能性が高いものはどれか。

* A) FTD が Transparent Mode のため、アプライアンス自身が到達するための L3 スタティックルートが設定されていない。
* B) BVI インターフェイスが Access Control Policy の Source Zone に指定されていない。
* C) Transparent Mode では Syslog 送信機能が一切サポートされていない。
* D) BVI インターフェイスの MAC アドレスが対向スイッチでドロップされている。

**正解**: **A**
**解説**: アプライアンス自身から発出される通信（Syslog 等）の戻りトラフィックや送信先解決には、Transparent Mode であっても Bridge Group メンバーポートを指定した L3 スタティックルートが必要です。

---

### 3. Design 問題

**問題**: データセンター内の 2 台の Core スイッチ（VSS / vPC 構成）の間に FTD を Transparent Mode で挿入した。設置後、スイッチのポートが "UDLD Neighbor Mismatch" エラーで Err-disabled 状態になった。最も適切な設計上の回避策はどれか。

* A) FTD のモードを Routed Mode に変更する。
* B) スイッチ側の FTD 接続ポートにおいて、EtherChannel 上の UDLD 機能を無効化（disable）する。
* C) FTD 上で BVI の IP アドレスを削除する。
* D) ACP ルールで UDP ポート 12345 をブロックする。

**正解**: **B**
**解説**: Transparent Mode の FTD を VSS / vPC ペア間に配置すると、対向スイッチからの UDLD パケットが透過して重複受信され、"UDLD Neighbor Mismatch" を引き起こします。接続ポートで UDLD を無効化するのが標準的な回避策です。

---

### 4. 実装問題

**問題**: ASA1 の Transparent Mode において、攻撃者による CAM テーブル溢れ攻撃（CAM Overflow）を防止したい。取れる対策の組み合わせとして最も適切なものはどれか。

* A) `mac-learn <interface> disable` で動的 MAC 学習を無効化し、`mac-address-table static` で静的 MAC を登録する。
* B) BVI インターフェイスを shutdown する。
* C) `same-security-traffic permit inter-interface` を有効化する。
* D) OSPF を有効にして全 MAC アドレスを動的ルーティングで配信する。

**正解**: **A**
**解説**: 透過モードで MAC 学習を停止し、必要な宛先 MAC を静的登録（Static MAC）することで、CAM テーブル溢れ攻撃や MAC スプーフィングを防止できます。

---

### 5. コンフィグ読解問題

**問題**: FTD コンソール CLI で `configure firewall transparent` コマンドを実行した際に発生する即時イベントはどれか。

* A) デバイスが自動的に再起動し、既存のすべてのインターフェイス設定・ポリシーが初期化（消去）される。
* B) バックグラウンドでモード変換が行われ、既存の IP アドレスが BVI に自動変換される。
* C) FMC からの接続が自動遮断され、FDM (Firepower Device Manager) モードへ移行する。
* D) 物理ポートがすべて Passive モードに切り替わる。

**正解**: **A**
**解説**: CLI でファイアウォールモードを変更すると、警告メッセージが表示された後、**既存のインターフェイス構成が完全に破棄** されます。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Threat Defense Configuration Guide (FMC) - Transparent Mode](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/transparent_firewall_mode.html)
* [Cisco ASA Series General Operations Configuration Guide - Transparent Firewall Mode](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/general/asa-94-general-config/firewall-mode.html)
* [Cisco Technical Reference: Troubleshooting Transparent Mode Firewalls](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/118334-technote-asa-00.html)
* [Cisco Live: Advanced Layer 2 Security and Firewall Architecture (BRKSEC-3020)](https://www.ciscolive.com/)
* [CVD: Secure Firewall Deployment in Data Center Networks](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/cisco-secure-firewall-design-guide.html)

---

## 📝 補足（Notes）

* **学習メモ**:
  * Transparent Mode は **「IP アドレスを変更しない L2 透過挿入」** です。
  * 「パケット転送は MAC テーブル検索」、「アプライアンス自体の送信トラフィックには L3 スタティックルート検索が必要」という 2 つのルックアップ原則を整理しておきましょう。
* **注意点**:
  * ラボ試験で /30 サブネットを BVI に割り当ててしまう凡ミスや、FMC で Security Zone タイプ（Transparent）を間違えてインターフェイスにバインドできないトラブルに注意してください。

