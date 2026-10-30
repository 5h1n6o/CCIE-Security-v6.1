---
layout: default
title: 1.2.c-Traffic-zones
nav_order: 3
parent: 1.2-Firewall-features-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.2.c Traffic zones

Cisco ASA (Adaptive Security Appliance) および Cisco FTD (Firepower Threat Defense) における **Traffic Zones（トラフィックゾーン）** は、複数の物理または論理インターフェイスを 1 つの論理的な「ゾーン（Zone）」としてグループ化する技術です。

従来のステートフルファイアウォール（LINA Engine）は、セッションの送信元・宛先インターフェイスを固着して管理するため、インバウンドパケットとアウトバウンドパケットが異なるインターフェイスを通過する **非対称ルーティング（Asymmetric Routing）** が発生すると、接続テーブル（Conn Table）の不一致によりパケットを自動的にドロップします。Traffic Zones は、この接続テーブル管理をインターフェイス単位から「ゾーン単位」に拡張することで、**非対称ルーティングの自動許容**、**複数インターフェイス間に跨がる ECMP (Equal-Cost Multi-Path) ロードバランシング**、および **リンク障害時の透過的なルート冗長化** を実現します。

---

## 蓝 概要

* **機能概要**:
  * **ゾーンレベルのステートフルセッション管理**: 接続テーブル（xlate / conn）を個別の `nameif` インターフェイスではなく、複数のインターフェイスが所属する `zone` 全体で共有・追跡します。
  * **マルチインターフェイス ECMP ルーティング**: ASA / FTD で従来 1 つの物理ポート上に制限されていた等コストマルチパス（ECMP）ルーティングを、最大 8 つの異なる物理/論理ポート間に拡張します（ゾーンあたり最大 8 ルート/8 インターフェイス）。
  * **非対称フローの正常処理**: 往路が `Outside1` インターフェイスから送出され、復路が `Outside2` インターフェイスから戻ってきた場合でも、両インターフェイスが同一の Traffic Zone（例: `Zone-Outside`）に属していれば、セッションテーブルにマッチして正常に通過・処理されます。
* **利用目的**:
  * マルチホーム（Dual-ISP / Multi-ISP）環境における非対称トラフィックのドロップ防止。
  * 複数 WAN リンク間での動的/静的 ECMP パケット分散（ロードバランシング）。
  * 障害発生時の即時経路切替（コンバージェンス）に伴う既存アクティブセッションの維持。
* **どのような場面で利用するか**:
  * 2 社以上の ISP から異なるパピック IP 帯域の割り当てを受け、BGP や OSPF 等で等コストマルチパスを構成しているインターネット境界。
  * クラスタリング構成（Cisco ASA Cluster / FTD Cluster）において、個別インターフェイスモード（Individual Interface Mode）運用時に複数ルータとのアジャセンシーによる非対称通信が発生する環境。

---

## 🔑 要点

| 項目 | Cisco ASA (CLI / ASDM) | Cisco FTD (FMC / FDM) |
| :--- | :--- | :--- |
| **制御コンポーネント** | ASA LINA Traffic Zone (`zone`) | Virtual Router ECMP Zone ＆ FMC Security Zone |
| **上限数** | デバイス/コンテキストあたり最大 **256 ゾーン** | 仮想ルータあたり最大 **256 ECMP ゾーン** |
| **所属可能ポート数** | 1 ゾーンあたり最大 **8 インターフェイス** | 1 ゾーンあたり最大 **8 インターフェイス** |
| **ポート重複制限** | 1 ポートは 1 ゾーンにのみ所属可能 | 1 ポートは 1 ECMP ゾーンにのみ所属可能 |
| **セキュリティポリシーの適用単位** | **インターフェイス単位 (`nameif`)** (※ゾーン宛ではない) | **Security Zone 単位** (FMC Access Control Policy) |
| **ECMP ハッシュアルゴリズム** | L3/L4 5-tuple (Src/Dst IP, Src/Dst Port, Proto) | L3/L4 5-tuple (Src/Dst IP, Src/Dst Port, Proto) |
| **非対応ポート** | Management-only, Failover/State Link, CCL, BVI | Management-only, Failover/State Link, CCL, BVI, VNI, Loopback |
| **ASR Group との関係** | **併用不可** (ASR Group と Traffic Zone は共存不能) | **N/A** (FXOS / Multi-Instance が代替) |

---

## 🏗 動作原理

### 1. 非ゾーン環境 (Interface-based) vs ゾーン環境 (Zone-based) の接続テーブル処理

通常のステートフルファイアウォールでは、最初の SYN パケットが入ってきたインターフェイス（Primary Interface）と出参するインターフェイス（Current Interface）がセッションテーブルに固定的に書き込まれます。

```text
【非ゾーン環境 (Non-Zoned) での非対称通信ドロップ】

Client (Inside) ───► Ingress: Inside ───► ASA ───► Egress: Outside1 ───► ISP 1 ───► Server
                                           │
                                 (Conn Table Registered: Inside <-> Outside1)
                                           │
Server ───────────► ISP 2 ───────────────► Ingress: Outside2
                                           │
                                           ▼
                            [ Connection Lookup Failed! ]
               (Outside2 != Outside1 -> Unexpected Ingress Interface)
                                           │
                                           ▼
                                    DROP (TCP Reset)
```

```text
【Traffic Zone 環境での非対称通信許容】

Client (Inside) ───► Ingress: Inside ───► ASA ───► Egress: Outside1 ───► ISP 1 ───► Server
                                           │
                    (Zone Conn Table: Inside <-> Zone-Outside [Outside1, Outside2])
                                           │
Server ───────────► ISP 2 ───────────────► Ingress: Outside2
                                           │
                                           ▼
                            [ Connection Lookup Success! ]
                 (Outside2 belongs to Zone-Outside -> Session Matched)
                                           │
                                           ▼
                          FORWARD (Current Interface Updated)
```

---

## ⚙ 動作シーケンス

Traffic Zone 内におけるパケット処理パイプラインおよびインターフェイス移行順序は以下の通りです。

1. **Ingress パケットの受信とゾーン識別**:
   * 入力物理/論理ポートから、所属する Traffic Zone（例: `Zone-Outside`）を識別します。
2. **ゾーンレベル接続テーブル（Zone Connection Table）の検索**:
   * 受信パケットの 5 プレット（送信元 IP, 宛先 IP, 送信元 Port, 宛先 Port, Protocol）を照合します。
   * 入力ポートが前回のプライマリポート（`Outside1`）と異なっていても、同一ゾーン内のポート（`Outside2`）であればマッチ判定が下ります。
3. **Current Interface（現在のインターフェイス）の動的更新**:
   * 非対称ルートにより通信パスが変化した場合、ASA / FTD は該当セッションの「Current Interface」を `Outside2` に動的に更新します（「Primary Interface」は最初にセッションが作られたポートとして保持されます）。
4. **セキュリティポリシーの評価 (ASA と FTD の決定的な違い)**:
   * **ASA**: セキュリティポリシー（ACL や NAT）は **ゾーン名ではなく各ポートの `nameif`（例: `outside1`, `outside2`）に対して個別に評価** されます。このため、ゾーン内のすべてのポートに同等の ACL / NAT 設定を入れておく必要があります。
   * **FTD**: FMC 上の Access Control Policy (ACP) は **Security Zone（例: `SZ-Outside`）に対して一括適用** されるため、ゾーン内全ポートに同一ポリシーが自動適用されます。
5. **ECMP 転送とハッシュ算出**:
   * 新規セッション（SYN パケット）が発生した場合、ゾーン内の利用可能な全等コストルートから、5 プレットをハッシュ関数に投入して出力ポートを自動選択・分散します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **ASA における「ポリシーはインターフェイス単位」という罠**:
   * ASA で `zone OutsideZone` を作成し、`GigabitEthernet0/1` (`outside1`) と `GigabitEthernet0/2` (`outside2`) をバインドしても、**`access-group` や `nat` はゾーン名に対して定義できません**。
   * 試験要件で「Traffic Zone を使用してロードバランシングせよ」と指示された場合、`outside1` と `outside2` の両インターフェイスに対して、**完全に平行（Parallel）な ACL および NAT ルールを投入** しなければ、非対称トラフィックが返ってきた際に向き先のポートで ACL ドロップや NAT 漏れが発生します。
2. **`same-security-permit` コマンドの設定要件 (Intra-Zone / Inter-Interface)**:
   * トラフィックがゾーン内のポートに入り、同じゾーン内の別のポートから抜ける構成（ゾーン内ルーティング / U ターン通信）が必要な場合、以下を有効化しなければドロップされます。
     * `same-security-permit inter-interface`: 異なるインターフェイス間での通信許可。
     * `same-security-permit intra-interface`: 同一インターフェイス出入り通信許可。
3. **ASR Group (Active/Active Failover) との排他条件**:
   * A/A Failover で使用される非対称ルーティング補正機能 `asr-group` と Traffic Zone は **同一コンテキスト内で共存できません**。試験で `asr-group` が設定されているコンテキストに対して Traffic Zone の追加を求められた場合、先に `asr-group` 設定を削除する必要があります。
4. **FTD における ECMP Zone と Security Zone の役割分担**:
   * FTD では、ルーティング/非対称通信を司る **ECMP Zone**（`Devices > Device Management > Routing > ECMP`）と、セキュリティルールを適用する **Security Zone**（`Objects > Object Management > Security Zone`）の 2 種類が存在します。
   * ECMP Zone に所属するポート群を、FMC 上の同一 Security Zone オブジェクトに所属させることで、正しいロードバランシングとポリシー適用が両立します。
5. **インターフェイス追加・削除時のセッション切断仕様**:
   * インターフェイスを Traffic Zone に割り当てた瞬間、**そのインターフェイス上のすべての既存接続（Conn Table）は即座にフラッシュ（削除）** されます。稼働中環境での設定変更時にはダウンタイムが発生します。

---

## 🛠 設定方法

### 1. Cisco ASA: CLI による Traffic Zone 構成 (Dual-ISP ECMP & 非対称ルーティング)

```bash
! 1. Traffic Zone オブジェクトの定義
zone Zone-Outside

! 2. 物理ポートの作成と Zone へのバインド (outside1)
interface GigabitEthernet0/1
 nameif outside1
 security-level 0
 zone-member Zone-Outside
 ip address 203.0.113.1 255.255.255.248

! 3. 物理ポートの作成と Zone へのバインド (outside2)
interface GigabitEthernet0/2
 nameif outside2
 security-level 0
 zone-member Zone-Outside
 ip address 198.51.100.1 255.255.255.248

! 4. インターフェイス間通信の許可 (Zone 内通信用)
same-security-permit inter-interface
same-security-permit intra-interface

! 5. ECMP デフォルトルートの設定 (Zone ポートを跨ぐ等コストルート)
route outside1 0.0.0.0 0.0.0.0 203.0.113.6
route outside2 0.0.0.0 0.0.0.0 198.51.100.6

! 6. パラレルセキュリティポリシー (ACL) の作成と適用
access-list OUTSIDE1_IN extended permit tcp any host 10.1.1.100 eq 80
access-list OUTSIDE2_IN extended permit tcp any host 10.1.1.100 eq 80

access-group OUTSIDE1_IN in interface outside1
access-group OUTSIDE2_IN in interface outside2

! 7. パラレル Auto NAT の作成
object network OBJ_INSIDE_NET
 subnet 10.1.0.0 255.255.0.0
 nat (inside,outside1) dynamic interface
 nat (inside,outside2) dynamic interface
```

### 2. Cisco FTD (FMC GUI): ECMP Zone および Security Zone 構成

1. **Security Zone の作成 (ACP 用)**:
   * **Objects > Object Management > Interface Objects > Security Zone**。
   * **Add Security Zone**: Name: `SZ-WAN-ZONE`, Type: `Routed`。
   * `WAN1` (eth1/1) および `WAN2` (eth1/2) インターフェイスを選択して追加し保存。
2. **ECMP Zone の作成 (ルーティング用)**:
   * **Devices > Device Management** で対象 FTD を編集 ➔ **Routing** タブ ➔ **ECMP**。
   * **Add**: Name: `ECMP-WAN-ZONE`。
   * インターフェイスリストから `WAN1` と `WAN2` をチェックして割り当て。
3. **等コスト Static Route の追加**:
   * **Routing > Static Route** ➔ **Add Route**。
   * Route 1: Interface: `WAN1`, Available Network: `any-ipv4`, Gateway: `203.0.113.6`, Metric: `1`。
   * Route 2: Interface: `WAN2`, Available Network: `any-ipv4`, Gateway: `198.51.100.6`, Metric: `1` (同一 Metric)。
4. **Deploy の実行**:
   * FMC 上で保存後、対象 FTD デバイスへ Deploy を実行。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **定義済み Traffic Zone 一覧および所属ポート確認** | ASA / FTD CLI | <code>show zone</code> / <code>show running-config zone</code> |
| **ゾーンレベルの接続テーブル（xlate/conn）表示** | ASA / FTD CLI | <code>show conn long zone</code> / <code>show conn zone \<NAME\></code> |
| **ゾーン対応の高速パケットパス (ASP) テーブル確認** | ASA / FTD CLI | <code>show asp table zone</code> |
| **ゾーン対応ルーティングテーブル表示** | ASA / FTD CLI | <code>show route zone</code> / <code>show asp table routing</code> |
| **ゾーン内ローカルホスト接続状態確認** | ASA / FTD CLI | <code>show local-host zone \<NAME\></code> |
| **FTD Virtual Router 内の ECMP Zone 設定確認** | FTD CLI | <code>system support diagnostic-cli</code> ➔ <code>show zone</code> |
| **パケットフロー追跡 (非対称ゾーン判定の確認)** | ASA / FTD CLI | <code>packet-tracer input outside1 tcp 1.1.1.1 12345 10.1.1.10 80 detail</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **戻りパケットが受信ポートでドロップされる** | リターンポートが送信ポートと異なる Zone に属している | ASA: <code>show zone</code><br>ASA: <code>show conn long zone</code> | 往路ポートと復路ポートの両方を同一の Traffic Zone にバインドする。 |
| **ASA で片方のポートからのアクセスが ACL で拒否される** | Traffic Zone 内のポート間で ACL 設定がパラレルになっていない | ASA: <code>show access-group</code> | ゾーンに属するすべての `nameif` ポートに同等の ACL を適用する。 |
| **ゾーン内ポート間（Same Zone）の通信が届かない** | `same-security-permit` 設定の欠落 | ASA: <code>show running-config same-security-permit</code> | `same-security-permit inter-interface` および `intra-interface` を有効化する。 |
| **「Cannot configure zone: ASR group exists」エラー** | 該当ポートに `asr-group` が設定されている | ASA: <code>show running-config interface</code> | 該当ポートから `no asr-group` を実行して ASR 設定を削除してからゾーンを適用する。 |
| **ECMP ルートがルーティングテーブルに反映されない** | ゾーン内ポートの Metric が異なっている | ASA/FTD: <code>show route zone</code> | 対象ポートに対するスタティック/動的ルートの Metric / Distance を完全に一致させる。 |
| **ゾーン追加後に接続中の全セッションが切断された** | ゾーン割り当て時の仕様によるセッションクリア | ASA: <code>show conn count</code> | 正常な動作仕様。ゾーン構成変更はメンテナンスウィンドウ中に実施する。 |

---

## ⚠ 制限事項

* **非サポートインターフェイス**:
  * Management-only / Management-access インターフェイス。
  * Failover リンクおよび Stateful Failover リンク。
  * Cluster Control Link (CCL)。
  * BVI (Bridge Virtual Interface) および Bridge Group メンバーポート。
  * VNI (Virtual Network Interface) および NVE ポート。
  * Loopback インターフェイスおよび dVTI (FTD)。
* **機能制限**:
  * ゾーン内ポートでの **DHCP Relay は非サポート**。
  * トラフィックポリシング (QoS Policing) はゾーンレベルでは適用不可（帯域制限のみ）。
  * IPsec VPN 通信において配送経路に NAT が存在する環境での ECMP / Traffic Zone 併用は非サポート。

---

## 🔄 他技術との関連

* **1.1 Deployment Modes**: Traffic Zone は **Routed Mode でのみサポート** され、Transparent Mode では使用できません。
* **1.2.a NAT**: ASA では各ポートの `nameif` に対して Auto NAT / Manual NAT を定義します。FTD では FMC の Security Zone を使用して NAT ルールを一括適用します。
* **1.3 High Availability & Clustering**:
  * Failover ペア構成時、スタンバイ機器にはプライマリポートの情報のみが同期されます（Current Port は同期されず、フェールオーバー時に新しいアクティブ機器が自動再割り当て）。
  * ASA / FTD Cluster の Individual Interface Mode において、複数ルータとのマルチアジャセンシー通信時の非対称ドロップを防ぐために Traffic Zone が不可欠です。

---

## 🧩 比較表

### 1. Interface-based Routing vs Traffic Zone (ECMP)

| 比較項目 | 従来型 (Interface-based) | Traffic Zone (Zone-based) |
| :--- | :--- | :--- |
| **セッションテーブル識別** | 送信元/宛先 `nameif`（固定ポート） | **送信元/宛先 `zone`（ゾーン内複数ポート）** |
| **非対称ルーティング** | **ドロップ** (別ポートからの復路は拒否) | **許容** (同一ゾーン内ポートであれば正常処理) |
| **ECMP サポート** | **単一ポート内のみ** (同一 Gateways) | **最大 8 個の異なる物理/論理ポート間 ECMP** |
| **障害切替 (Link Failure)** | 既存セッション切断 (Re-SYN 必要) | ゾーン内別ポートへ **無停止透過引き継ぎ** |
| **ASR Group との共存** | 共存可能 | **共存不可 (排他利用)** |

### 2. ASA Traffic Zone vs FTD Security Zone vs FTD ECMP Zone

| 比較項目 | Cisco ASA Traffic Zone | Cisco FTD Security Zone | Cisco FTD ECMP Zone |
| :--- | :--- | :--- | :--- |
| **主な目的** | 非対称許容 ＆ ECMP ルーティング | **Access Control Policy (ACP) 一括適用** | 非対称許容 ＆ ECMP ルーティング |
| **設定場所** | CLI (`zone <NAME>`) | FMC (`Objects > Security Zone`) | FMC (`Devices > Routing > ECMP`) |
| **ポリシー適用** | **インターフェイス単位 (`nameif`)** | **ゾーン単位 (ACP Rule)** | ルーティング設定（Static/Dynamic） |
| **対応モード** | Routed Mode 専用 | Routed / Transparent / Inline | Routed Mode 専用 |

---

## 💡 ベストプラクティス

1. **同一ゾーン内ポートに対する設定の完全パラレル化 (ASA)**:
   * ASA で Traffic Zone を運用する場合、ゾーン内の全ポート (`outside1`, `outside2` 等) に対し、Access Group、NAT、QoS、Service Policy を完全に同等に設定します。
2. **`same-security-permit` の標準有効化**:
   * ゾーン内インターフェイスを通過する U ターン通信や迂回フローでのドロップを防ぐため、`same-security-permit inter-interface` および `intra-interface` を事前に有効化します。
3. **ASR Group からの計画的移行**:
   * Active/Active Failover 環境で従来の `asr-group` から Traffic Zone へ移行する場合は、`asr-group` を削除してから `zone` を構成する順番を厳守します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA: Dual-ISP 非対称ルーティングを許可する Traffic Zone の基本構成

#### 問題
ASA1 において、2 つの外部物理ポート `GigabitEthernet0/1` (`outside1`: `203.0.113.1/29`) と `GigabitEthernet0/2` (`outside2`: `198.51.100.1/29`) を単一の Traffic Zone `ZONE-WAN` にグループ化せよ。ISP1 経由で開始された通信の戻りパケットが ISP2 経由で到着した場合でもドロップされないように構成せよ。

#### 要件
* Traffic Zone 名: `ZONE-WAN`
* ASA CLI で設定すること。

#### 設定例

```bash
! 1. Traffic Zone の作成
zone ZONE-WAN

! 2. outside1 ポートの設定と Zone バインド
interface GigabitEthernet0/1
 nameif outside1
 security-level 0
 zone-member ZONE-WAN
 ip address 203.0.113.1 255.255.255.248

! 3. outside2 ポートの設定と Zone バインド
interface GigabitEthernet0/2
 nameif outside2
 security-level 0
 zone-member ZONE-WAN
 ip address 198.51.100.1 255.255.255.248

! 4. 同一 Zone 内およびインターフェイス間通信の許可
same-security-permit inter-interface
same-security-permit intra-interface
```

#### 検証方法
* `show zone` を実行し、`ZONE-WAN` に `outside1` と `outside2` が登録されていることを確認。
* 外部から `outside1` IP にセッションを開始し、戻りパケットを `outside2` 側から入力させ、`show conn long zone` でセッションがドロップされず疎通することを確認。

---

### 例2: ASA: 3 つの WAN リンク間における 4 ウェイ ECMP デフォルトルーティング構成

#### 問題
ASA1 において、`outside1` (`203.0.113.1`), `outside2` (`198.51.100.1`), `outside3` (`192.0.2.1`) の 3 ポートを Traffic Zone `ZONE-INTERNET` に割り当て、各 ISP ルータ宛ての等コストデフォルトルートを構成してパケット単位ロードバランシングを行わせよ。

#### 要件
* 各 ISP GW: ISP1 (`203.0.113.6`), ISP2 (`198.51.100.6`), ISP3 (`192.0.2.6`)

#### 設定例

```bash
zone ZONE-INTERNET

interface GigabitEthernet0/1
 nameif outside1
 security-level 0
 zone-member ZONE-INTERNET
 ip address 203.0.113.1 255.255.255.0

interface GigabitEthernet0/2
 nameif outside2
 security-level 0
 zone-member ZONE-INTERNET
 ip address 198.51.100.1 255.255.255.0

interface GigabitEthernet0/3
 nameif outside3
 security-level 0
 zone-member ZONE-INTERNET
 ip address 192.0.2.1 255.255.255.0

! 等コストデフォルトルートの登録
route outside1 0.0.0.0 0.0.0.0 203.0.113.6
route outside2 0.0.0.0 0.0.0.0 198.51.100.6
route outside3 0.0.0.0 0.0.0.0 192.0.2.6
```

#### 検証方法
* `show route zone` を実行し、`0.0.0.0/0` 宛てのルートが 3 つのポートに分散して登録されていることを確認。
* 内部から様々な宛先 IP 宛てに通信を発生させ、`show asp table routing` でハッシュ分散を確認。

---

### 例3: ASA: Traffic Zone 内におけるパラレル ACL および パラレル Auto NAT 構成

#### 問題
Traffic Zone `ZONE-WAN`（メンバー: `outside1`, `outside2`）が構成されている ASA1 において、内部 `10.1.0.0/16` から外部への通信を両ポートで動的 PAT 変換せよ。また外部から内部 Web サーバー (`10.1.1.100`) の HTTP アクセスを両ポートで許可せよ。

#### 要件
* ASA では ACL/NAT がインターフェイス単位で評価される点に留意して設定すること。

#### 設定例

```bash
! パラレル ACL の作成
access-list OUTSIDE1_IN extended permit tcp any host 10.1.1.100 eq 80
access-list OUTSIDE2_IN extended permit tcp any host 10.1.1.100 eq 80

access-group OUTSIDE1_IN in interface outside1
access-group OUTSIDE2_IN in interface outside2

! パラレル Auto NAT の定義
object network OBJ_INSIDE_SUBNET
 subnet 10.1.0.0 255.255.0.0
 nat (inside,outside1) dynamic interface
 nat (inside,outside2) dynamic interface
```

#### 検証方法
* `show running-config nat` および `show running-config access-group` で両ポートに同等ルールが適用されていることを確認。

---

### 例4: ASA: ASR Group から Traffic Zone への構成移行手続き

#### 問題
現在 Active/Active Failover コンテキスト内で非対称補正用に構成されている `asr-group 1` を撤去し、新しく Traffic Zone `ZONE-HA-WAN` へ移行せよ。

#### 要件
* 設定競合エラー（Cannot configure zone: ASR group exists）を回避する手順とすること。

#### 設定例

```bash
! 1. 既存の asr-group を物理ポートから削除
interface GigabitEthernet0/1
 no asr-group 1
interface GigabitEthernet0/2
 no asr-group 1

! 2. Traffic Zone の作成とバインド
zone ZONE-HA-WAN

interface GigabitEthernet0/1
 zone-member ZONE-HA-WAN

interface GigabitEthernet0/2
 zone-member ZONE-HA-WAN
```

#### 検証方法
* `show zone` を実行し、エラーなく `ZONE-HA-WAN` に両ポートがバインドされていることを確認。

---

### 例5: ASA: Intra-Zone 内部ルーター間 U ターン通信の許可設定

#### 問題
ASA1 の Traffic Zone `ZONE-INTERNAL` に属する `inside1` (`10.1.1.1/24`) と `inside2` (`10.1.2.1/24`) 間において、一方に入力されたトラフィックが他方のポートへ抜けられるよう Intra-Zone 通信を許可せよ。

#### 要件
* `same-security-permit` を正しく設定すること。

#### 設定例

```bash
zone ZONE-INTERNAL

interface GigabitEthernet0/3
 nameif inside1
 security-level 100
 zone-member ZONE-INTERNAL
 ip address 10.1.1.1 255.255.255.0

interface GigabitEthernet0/4
 nameif inside2
 security-level 100
 zone-member ZONE-INTERNAL
 ip address 10.1.2.1 255.255.255.0

! 同一 Zone 内ポート間通信の有効化
same-security-permit inter-interface
same-security-permit intra-interface
```

#### 検証方法
* `10.1.1.50` から `10.1.2.50` 宛てに ping を送信し、`show conn long zone` でセッションが維持されることを確認。

---

### 例6: FTD (FMC): Global Virtual Router における ECMP Zone の構成

#### 問題
FMC を使用して FTD1 の Global Virtual Router 上に ECMP Zone `ECMP-OUTSIDE` を作成し、`WAN1` (eth1/1) と `WAN2` (eth1/2) を割り当てよ。

#### 要件
* FMC GUI の Routing 設定より実施すること。

#### 設定例

```text
1. FMC: Devices > Device Management > FTD1 編集 > Routing タブ。
2. Manage Virtual Routers: `Global` を選択。
3. ECMP を選択し、`Add` をクリック:
   - Name: ECMP-OUTSIDE
   - Interfaces: WAN1, WAN2 にチェックを入れて OK。
4. Static Route タブで WAN1 (GW: 203.0.113.6, Metric: 1) と WAN2 (GW: 198.51.100.6, Metric: 1) の等コストルートを作成。
5. Save & Deploy 実行。
```

#### 検証方法
* FTD CLI (Diagnostic CLI) で `show zone` および `show route zone` を実行し、ECMP Zone およびルーティングテーブルを確認。

---

### 例7: FTD (FMC): Security Zone と ECMP Zone を組み合わせた Access Control Policy 適用

#### 問題
FTD1 において、ECMP Zone に属する `WAN1` と `WAN2` を包摂する Security Zone `SZ-OUTSIDE` を作成し、Access Control Policy で内部ネットワークから `SZ-OUTSIDE` 宛ての HTTP/HTTPS 通信を許可せよ。

#### 要件
* FMC の Security Zone オブジェクトを使用すること。

#### 設定例

```text
1. FMC: Objects > Object Management > Interface Objects > Security Zone:
   - Name: SZ-OUTSIDE, Type: Routed, Interfaces: WAN1, WAN2 を追加して Save。
2. Policies > Access Control > ACP 編集 > Add Rule:
   - Name: Allow_Internet_Access, Action: Allow
   - Source Zone: SZ-INSIDE, Destination Zone: SZ-OUTSIDE
   - Protocols: HTTP, HTTPS
3. Save & Deploy 実行。
```

#### 検証方法
* 内部端末から Web アクセスを発生させ、どちらの WAN ポートを通過した場合でも FMC の Connection Events で `SZ-OUTSIDE` 宛てとして許可されることを確認。

---

### 例8: FTD (FMC): VTI (Virtual Tunnel Interface) 複数経路に対する ECMP Zone 構成

#### 問題
FTD1 と対向拠点間で 2 本の IPsec VTI トンネル (`VTI01`, `VTI02`) が貼られている。両 VTI インターフェイスを ECMP Zone `ECMP-VTI` に割り当て、VTI 経由のトラフィックをロードバランシングせよ。

#### 要件
* Metric: `10` の等コスト Static Route を両 VTI に設定すること。

#### 設定例

```text
1. FMC: Devices > Device Management > FTD1 > Routing > ECMP:
   - Add > Name: ECMP-VTI, Selected Interfaces: VTI01, VTI02 -> OK.
2. Static Route > Add Route:
   - Interface: VTI01, Network: 172.16.0.0/16, Gateway: 192.168.101.21, Metric: 10
   - Interface: VTI02, Network: 172.16.0.0/16, Gateway: 192.168.102.21, Metric: 10
3. Save & Deploy 実行。
```

#### 検証方法
* FTD CLI で `show route` を実行し、`172.16.0.0/16` 宛てのルートが `VTI01` と `VTI02` の両方へロードバランス設定されていることを確認。

---

### 例9: ASA: Traffic Zone 内における OSPF 動的等コストルートの適用

#### 問題
ASA1 の `outside1` と `outside2` ポート（Zone `ZONE-OSPF` 所属）上で OSPF プロセス 100 を有効化し、上流ルータ群から受信する OSPF 等コストルート（ECMP）を Traffic Zone 内で自動ロードバランスさせよ。

#### 要件
* OSPF Area 0 で構成すること。

#### 設定例

```bash
zone ZONE-OSPF

interface GigabitEthernet0/1
 nameif outside1
 security-level 0
 zone-member ZONE-OSPF
 ip address 203.0.113.1 255.255.255.0
 ospf cost 10

interface GigabitEthernet0/2
 nameif outside2
 security-level 0
 zone-member ZONE-OSPF
 ip address 198.51.100.1 255.255.255.0
 ospf cost 10

router ospf 100
 router-id 1.1.1.1
 network 203.0.113.0 255.255.255.0 area 0
 network 198.51.100.0 255.255.255.0 area 0
```

#### 検証方法
* `show ospf neighbor` で両ポートのネイバーが `FULL` であることを確認。
* `show route zone` で OSPF 学習ルートが両ポートに分散して登録されていることを確認。

---

### 例10: ASA/FTD: `packet-tracer` を使用した Traffic Zone 非対称フロー検証

#### 問題
`outside1` から開始された接続に対し、戻りパケットが `outside2` に到着するシナリオを `packet-tracer` コマンドでシミュレーションし、Traffic Zone の非対称判定ロジックを検証せよ。

#### 要件
* `packet-tracer` コマンドを使用すること。

#### 設定例

```bash
! 往路シミュレーション (Inside -> Outside1)
packet-tracer input inside tcp 10.1.1.50 12345 203.0.113.100 80 detail

! 復路シミュレーション (Outside2 -> Inside: 非対称パス)
packet-tracer input outside2 tcp 203.0.113.100 80 10.1.1.50 12345 detail
```

#### 検証方法
* 復路の `packet-tracer` 出力において、`Phase: CONN-LOOKUP` が `ALLOW` となり、`Zone-member: ZONE-WAN` により非対称パケットがドロップされず通過することを検証。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ASA1 において以下のコンフィグが適用されている。内部端末 `10.1.1.50` から外部へ発信された TCP 通信の戻りパケットが `outside2` インターフェイスに到着した。このパケットの処理結果として正しいものはどれか。

```bash
zone ZONE-INTERNET

interface GigabitEthernet0/1
 nameif outside1
 security-level 0
 zone-member ZONE-INTERNET
 ip address 203.0.113.1 255.255.255.0

interface GigabitEthernet0/2
 nameif outside2
 security-level 0
 ip address 198.51.100.1 255.255.255.0
```

* A) `ZONE-INTERNET` のセッションテーブルにマッチして正常に通過する。
* B) `outside2` ポートに `zone-member ZONE-INTERNET` が設定されていないため、接続テーブル不一致でドロップされる。
* C) `outside1` と `outside2` のセキュリティレベルが同じであるため自動的に通過する。
* D) `same-security-permit` が設定されていないためドロップされる。

**正解**: **B**
**解説**: `outside2` インターフェイスに `zone-member ZONE-INTERNET` コマンドが欠落しているため、`outside2` は Traffic Zone のメンバーではありません。そのため、`outside1` で作成されたゾーンセッションテーブルと照合されず、非対称パケットとしてドロップされます。

---

### 2. トラブルシュート問題

**問題**: 管理者が ASA1 上で 2 つの外部ポート (`outside1`, `outside2`) を Traffic Zone `ZONE-WAN` に追加しようと `zone-member ZONE-WAN` を入力したところ、以下のエラーが表示された。原因として正しいものはどれか。

```text
ERROR: Cannot configure zone: ASR group exists on interface GigabitEthernet0/1
```

* A) ASA のライセンスが Base ライセンスであり、Traffic Zone に未対応であるため。
* B) 該当インターフェイスに Active/Active Failover 用の `asr-group` が設定されており、Traffic Zone と排他関係にあるため。
* C) `outside1` のセキュリティレベルが 100 に設定されているため。
* D) モードが Multiple Context Mode であるため。

**正解**: **B**
**解説**: Cisco ASA において、`asr-group`（非対称ルーティンググループ）と `zone`（Traffic Zone）は同一コンテキスト/インターフェイス上で共存できません。インターフェイスから `no asr-group` を実行して削除してからゾーンを割り当てる必要があります。

---

### 3. Design 問題

**問題**: 大規模マルチホーム環境において、Cisco FTD デバイスを使用して 4 つの異なる ISP リンクへ等コストマルチパス (ECMP) ルーティングを行い、非対称トラフィックのドロップを防止したい。FMC での実装設計として最も適切な組み合わせはどれか。

* A) FMC の `Objects > Security Zone` で 4 ポートを同一 Security Zone に追加し、各ポートに異なる Metric の Static Route を設定する。
* B) FMC の `Devices > Routing > ECMP` で 4 ポートを同一 ECMP Zone に追加し、同一 Metric の Static Route を各ポート宛てに設定する。
* C) Prefilter Policy で `Fastpath` ルールを作成し、各ポートに `asr-group` をバインドする。
* D) BVI インターフェイスを作成し、4 ポートを同一 Bridge Group に配置する。

**正解**: **B**
**解説**: FTD でマルチポート ECMP および非対称ルーティング制御を行うには、FMC の `Routing > ECMP` 画面で対象ポート群を **ECMP Zone** に割り当て、同一 Metric のルート（Static または Dynamic）を設定するのが正しい標準設計です。

---

### 4. 実装問題

**問題**: ASA1 において、Traffic Zone `ZONE-LAN` に属する `inside1` ポートから入ったパケットが、同じ `ZONE-LAN` 内の `inside2` ポートから抜ける U ターンルーティングを許可するために必要なグローバル設定コマンドの組み合わせはどれか。

* A) `same-security-permit inter-interface` および `same-security-permit intra-interface`
* B) `sysopt connection permit-override`
* C) `access-group ZONE-LAN in interface inside1`
* D) `asr-group 1`

**正解**: **A**
**解説**: ASA において同一ゾーン内や同一セキュリティレベルのポート間でトラフィックをルーティングさせるには、`same-security-permit inter-interface`（異ポート間）および `same-security-permit intra-interface`（同一ポート出入り）の両方の有効化が必要です。

---

### 5. コンフィグ読解問題

**問題**: FTD における ECMP Zone の制限事項に関する記述として**誤っているもの**はどれか。

* A) 1 つの ECMP Zone に割り当てられるインターフェイス数は最大 8 個である。
* B) BVI (Bridge Virtual Interface) や EtherChannel メンバーポートは ECMP Zone に追加できない。
* C) ECMP Zone 内のインターフェイス上で DHCP Relay 機能を使用することができる。
* D) 1 つのデバイス上に作成できる ECMP Zone の上限数は 256 個である。

**正解**: **C**
**解説**: Traffic Zone / ECMP Zone に属するインターフェイス上では、**DHCP Relay 機能は非サポート** です（Cisco 公式制限事項）。

---

## 🔗 参考リソース

* [Cisco ASA Series General Operations Configuration Guide - Traffic Zones]([suspicious link removed])
* [Cisco Secure Firewall Threat Defense Configuration Guide (FMC) - Equal-Cost Multi-Path (ECMP) Routing]([suspicious link removed])
* [Cisco Command Reference - zone / zone-member command on ASA]([suspicious link removed])
* [Cisco Technical Notes: ASA Traffic Zones Configuration Example and Asymmetric Routing Support]([suspicious link removed])
* [Cisco Live: Deep Dive into ASA/FTD Routing and Multi-WAN Architectures (BRKSEC-3020)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * ASA の Traffic Zone は **「ルーティングと接続テーブルをゆるく共有する仕組み」** であり、**「セキュリティポリシー（ACL/NAT）をまとめる仕組みではない」** 点が最大の引っかけポイントです。
  * FTD では **「ECMP Zone (ルーティング用)」** と **「Security Zone (ACP用)」** が明確に分離されているため、FMC GUI で両方を正しくセットで構成する手順を身につけておきましょう。
* **図解（ASA 接続テーブルと Zone の関係）**:
  ```text
  [ Non-Zone ASA ]
  Conn Table Entry: [ TCP | 10.1.1.10:1234 -> 203.0.113.100:80 | In: inside | Out: outside1 ]
  -> Return Packet on outside2: MISMATCH -> DROP

  [ Zoned ASA ]
  Conn Table Entry: [ TCP | 10.1.1.10:1234 -> 203.0.113.100:80 | In: inside | Out: Zone-Outside (outside1, outside2) ]
  -> Return Packet on outside2: MATCH -> ALLOW (Current interface updated to outside2)
  ```
* **注意点**:
  * 実機試験でインターフェイスをゾーンに追加すると、そのポート上の既存コネクションが即座にクリアされます。動作確認中に設定を追加する際はセッションが切断される点に注意してください。

