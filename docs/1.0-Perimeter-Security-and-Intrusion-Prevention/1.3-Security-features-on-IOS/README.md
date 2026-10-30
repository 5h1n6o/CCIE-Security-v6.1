---
layout: default
title: 1.3-Security-features-on-IOS
nav_order: 3
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.3 Security features on Cisco IOS/IOS XE

Cisco IOS および IOS XE ソフトウェアは、ルータおよびスイッチアプライアンス（ISR4000, Catalyst 8000, Catalyst 9000 シリーズ等）上でインフラストラクチャ自体とネットワークトラフィックを保護するための多層的セキュリティ機能を提供します。これらは Cisco の **Network Foundation Protection (NFP)** フレームワークに基づき、**Data Plane（データプレーン）**、**Control Plane（コントロールプレーン）**、**Management Plane（マネジメントプレーン）** の 3 つのプレーンに分類されて実装されます。

CCIE Security v6.1 Blueprint 1.3 では、データプレーン制御（Zone-Based Policy Firewall, uRPF, FPM, AVC/NBAR2）からコントロールプレーン防衛（CoPP）、マネジメントプレーン保護（MPP）まで、IOS/IOS XE 固有の高度なセキュリティ機構の完全な理解と実装能力が求められます。

---

## 📘 概要

* **機能概要**:
  * **Zone-Based Policy Firewall (ZBFW)**: インターフェイスをセキュリティゾーンにグループ化し、ゾーン間のトラフィックに対してクラスベースのステートフルパケットインスペクションを適用する次世代ゾーン型ファイアウォール。
  * **Control Plane Policing (CoPP)**: ルータ CPU（コントロールプレーン）へ向かう「Punt トラフィック」を特異的なクラスに分類し、QoS ポリシー（Rate Limiting / Policing / Drop）を用いて DoS 攻撃や制御パケット過多から CPU を保護する機能。
  * **Management Plane Protection (MPP)**: SSH、HTTPS、SNMP、NTP などのインバンド管理トラフィックを受信できる物理/論理インターフェイスを厳格に限定し、指定外インターフェイスからの管理アクセスをハードウェア/カーネルレベルで拒否する機能。
  * **Unicast Reverse Path Forwarding (uRPF)**: 着信パケットの送信元 IP アドレスと CEF（Cisco Express Forwarding）テーブルを照合し、IP スプーフィング（なりすまし）パケットをインgress ポートで即座にドロップする機能。
  * **Flexible Packet Matching (FPM)**: パケットの L2-L7 ペイロード内の任意のバイトオフセットおよびパターンパターン（ビットマスク/ポインタ演算）をパターンマッチングし、ACL では対応できないゼロデイ攻撃や特定シグネチャを破棄する高度ディープパケットフィルタリング機能。
  * **Application Visibility and Control (AVC) / NBAR2**: L7 アプリケーションをシグネチャベースで高精度に識別・分類（Next Generation Network-Based Application Recognition）し、QoS マーキングやドロップ、可視化（Flexible NetFlow）を行う機能。
* **利用目的**:
  * 専用アプライアンス（ASA/FTD）を配置できない拠点のルータ単体で、エンタープライズレベルのステートフルファイアウォール、DoS 防御、トラフィック制御を実現する。
  * ルータコントロールプレーンの枯渇（CPU 100% 張り付き）や不正アクセスによるネットワークダウンタイムを防止する。
* **どのような場面で利用するか**:
  * SD-WAN / 支社ルータ（Branch Office Router）におけるダイレクトインターネットアクセス（DIA）の保護。
  * コア/アグリゲーションルータにおける BGP / OSPF / EIGRP 制御通信の防御。
  * パブリック IP アドレス空間に露出している管理インターフェイスのロックダウン。

---

## 🔑 要点

| 機能 | 該当プレーン | 主な制御メカニズム | 設定コマンド体系 | メリット | 制限事項・注意点 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **ZBFW** | Data / Self Plane | CBM (Class-Based Policy) ＋ ステートフル検索 | <code>zone-member</code><br><code>policy-map type inspect</code> | インターフェイス非依存の柔軟なポリシー構成 | Pass ルールは双方向許可が必要（非ステートフル） |
| **CoPP** | Control Plane | MQC (Modular QoS CLI) ＋ CEF Punt パス制御 | <code>control-plane</code><br><code>service-policy input</code> | CPU リソース枯渇・DoS 攻撃からの絶対的防御 | 設定ミスにより BGP/OSPF 等の制御通信が切断されるリスク |
| **MPP** | Management Plane | Control Plane ホストサブコンテキストフィルター | <code>control-plane host</code><br><code>management-interface</code> | 未承認ポートからの管理プロトコル着信を完全遮断 | Inband 管理専用。Out-of-band(Gig0/0)と競合に留意 |
| **uRPF** | Data Plane | CEF テーブルの送信元 IP 逆引き検証 | <code>ip verify unicast source reachable-via</code> | IP スプーフィング攻撃を最前線で自動排除 | 非対称ルーティング環境では Loose モードが必須 |
| **FPM** | Data Plane | L2-L7 任意オフセットのパケットヘッダー/データ検索 | <code>load protocol</code><br><code>class-map type stack/access-control</code> | シグネチャ非依存のプロトコルフィールド検査 | CPU 負荷が高く、HW 転送（ASIC）から溢れる場合あり |
| **AVC / NBAR2** | Data Plane | ステートフル L7 アプリケーション識別エンジン | <code>ip nbar protocol-pack</code><br><code>match protocol</code> | 1000 以上の暗号化/非暗号化アプリを高精度識別 | 最新プロトコル検出には Protocol Pack 更新が必要 |

---

## 🏗 動作原理

Cisco IOS / IOS XE におけるパケット処理アーキテクチャは、データプレーンでの高速処理からコントロールプレーン保護まで一貫したパイプラインを持っています。

### 1. IOS XE パケット処理パイプラインとセキュリティ機能の適用順序

```text
[ Ingress Packet Arrival ]
            │
            ▼
┌──────────────────────────────────────────────────────────┐
│ 1. Ingress Layer 2 / Interface Checks                    │
│    └── Port Security / MAC ACL / Storm Control           │
└──────────────────────────┬───────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────┐
│ 2. Ingress Layer 3 Security Checks                       │
│    ├── Ingress IP ACL                                    │
│    └── Unicast RPF (uRPF Check vs CEF)                   │
│          ├── Strict Mode: Check Ingress Interface Match  │
│          └── Loose Mode: Check FIB Route Existence Only  │
└──────────────────────────┬───────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────┐
│ 3. Forwarding & Policy Engine                            │
│    ├── Flexible Packet Matching (FPM)                    │
│    ├── NBAR2 / AVC Classification                        │
│    └── Zone-Based Policy Firewall (ZBFW)                  │
│          ├── Check Session State Table (PAM)             │
│          └── Apply CBM Policy (Inspect / Pass / Drop)    │
└──────────────────────────┬───────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────┐
│ 4. Routing Lookup (CEF Decision)                         │
│    ├── Forward to Egress Interface? ──► [ Data Plane ]   │
│    └── Punt to CPU (Local Address)? ──► [ Control Plane ]│
└──────────┬────────────────────────────────┬──────────────┘
           │ (For Egress Transit)           │ (For Local CPU)
           ▼                                ▼
┌──────────────────────────┐   ┌───────────────────────────┐
│ Egress Processing        │   │ Control Plane Protection  │
│ ├── Egress IP ACL        │   │ ├── Management Plane (MPP)│
│ └── Egress QoS / NAT     │   │ └── CoPP (Control Plane)  │
└──────────┬───────────────┘   └────────────┬──────────────┘
           │                                │
           ▼                                ▼
[ Egress Interface Output ]    [ CPU Processing / Service ]
```

---

## ⚙ 動作シーケンス

### ZBFW (Zone-Based Policy Firewall) のパケット処理フロー

1. **Ingress / Egress ゾーンの判定**:
   * 着信パケットの入力ポート (`zone-member security Inside`) および CEF ルックアップで確定した出力ポート (`zone-member security Outside`) から、該当する `zone-pair security`（例: `Inside-to-Outside`）を割り出します。
2. **セッション状態テーブル（PAM: Port to Application Mapping）の検索**:
   * 既存のステートフルコネクションテーブルを照合します。
   * 復路パケット（Egress から Ingress）の場合、既存セッションにマッチすれば、ポリシーの評価をバイパスして透過転送されます。
3. **Class-Map / Policy-Map の評価**:
   * 新規セッション（SYN 等）の場合、Zone-Pair に割り当てられた `policy-map type inspect` が上から順に評価されます。
   * **Inspect**: セッション状態テーブルにエントリーを動的生成し、復路トラフィックを自動許可します。
   * **Pass**: 単方向でパケットを許可します（復路パケットの戻りを許可するには逆方向の Pass ポリシーが必須）。
   * **Drop**: パケットをログ記録（任意）のうえ静かに破棄します。

### CoPP (Control Plane Policing) のパケット処理フロー

1. **Punt トラフィックの捕捉**:
   * ルータの宛先 IP（自装置 IP）、BGP/OSPF/EIGRP ネイバー通信、ARP、ICMP、Telnet/SSH などの「CPU 処理を要するパケット」が CEF 送信パスからコントロールプレーンインターフェイスへ「Punt（引渡し）」されます。
2. **`control-plane` サブコンテキストでの分類**:
   * 入力パケットは、`control-plane` 配下に適用された `service-policy input` の Class-Map に順次マッチングされます。
3. **QoS ポリサー（Policer）の適用**:
   * 設定された帯域幅（bps / pps）を超過した制御パケットは、指定されたアクション（`drop` または `transmit`）に従って CPU へ到達する前にハードウェア/カーネルレベルで廃棄されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Self Zone (セルフゾーン) の特殊性と動作ルール**:
   * ルータ自装置（自身が送信元または宛先となる通信）は、明示的に作成しなくてもデフォルトで **`self` ゾーン** に属します。
   * **Inside ➔ Self**（例: 内部端末からルータへの SSH/SNMP）や **Self ➔ Outside**（例: ルータ自身からの BGP/NTP/Syslog）のトラフィックを ZBFW で制御する場合、`zone-pair security ZP_INSIDE_SELF source Inside destination self` を明示的に定義する必要があります。
   * **注意**: `self` ゾーンが絡まない Zone-Pair ポリシーは、ルータ自身から発生する通信には一切影響しません。
2. **CoPP と ZBFW の処理順序と相互干渉**:
   * コントロールプレーン宛てのトラフィックが入ってきた場合、**ZBFW (Self Zone) 判定が先に実行され、その後に CoPP が評価** されます。
   * ZBFW の Self Zone で Drop されたパケットは CoPP のカウンターには加算されません。問題文で「CoPP で ICMP をドロップさせ、統計カウントを確認せよ」と指示されている場合、Self Zone ポリシーで ICMP がブロックされていないか確認してください。
3. **uRPF Strict vs Loose の選択要件**:
   * **Strict Mode (`ip verify unicast source reachable-via rx`)**: パケットが着信した「まさにそのインターフェイス」が、CEF テーブルのベストパスと完全一致することを要求。単一 WAN リンクやアクセスレイヤで使用。
   * **Loose Mode (`ip verify unicast source reachable-via any`)**: 送信元 IP アドレスが CEF テーブル内に何らかのルート（Default Route 含む / 含まないの制御可）として存在すれば許可。非対称ルーティング（Asymmetric Routing）や BGP マルチホーム環境で必須。
   * **`allow-default` オプション**: デフォルトルート (`0.0.0.0/0`) を uRPF の到達可能ルートとして認めるかどうかの指定。指定しない場合、デフォルトルートしか存在しない送信元 IP はドロップされます。
4. **FPM (Flexible Packet Matching) の構成手順とファイル読み込み**:
   * FPM では、IP / TCP / UDP ヘッダーの定義ファイル（`ip.cdl`, `tcp.cdl` 等）を CLI でロードする必要があります（`load protocol flash:ip.cdl`）。
   * `class-map type stack`（プロトコルスタック順序の定義）と `class-map type access-control`（詳細フィールド/オフセットの検索）の 2 段階の Class-Map 構成を正確に組み立てる必要があります。
5. **MPP による特定管理ポートのロックダウン**:
   * `control-plane host` 配下で `management-interface GigabitEthernet0/0 allow ssh snmp` を設定すると、指定されていないその他のすべての物理インターフェイス（例: Gi0/1, Gi0/2）における SSH / SNMP の受信が **暗黙のドロップ（Implicit Deny）** となります。実機試験で自身の管理セッション（SSH）を切断しないよう極めて慎重に設定してください。

---

## 🛠 設定方法

### 1. Cisco IOS XE: Zone-Based Policy Firewall (ZBFW) の完全構成

```bash
! ===== 1. セキュリティゾーンの定義 =====
zone security INSIDE
zone security OUTSIDE
zone security DMZ

! ===== 2. ゾーンペアの定義 =====
zone-pair security ZP_IN_OUT source INSIDE destination OUTSIDE
zone-pair security ZP_IN_DMZ source INSIDE destination DMZ
zone-pair security ZP_OUT_SELF source OUTSIDE destination self

! ===== 3. トラフィック分類 (Class-Map) の構成 =====
ip access-list extended ACL_INSIDE_TRAFFIC
 permit ip 10.1.0.0 255.255.0.0 any

class-map type inspect match-any CM_IN_OUT
 match access-group name ACL_INSIDE_TRAFFIC
 match protocol http
 match protocol https
 match protocol dns

class-map type inspect match-all CM_MANAGEMENT_SELF
 match protocol ssh
 match access-group name ACL_ADMIN_HOSTS

! ===== 4. ポリシーマップ (Policy-Map) の構成 =====
policy-map type inspect PM_IN_OUT
 class CM_IN_OUT
  inspect
 class class-default
  drop log

policy-map type inspect PM_OUT_SELF
 class CM_MANAGEMENT_SELF
  pass
 class class-default
  drop

! ===== 5. ゾーンペアへのポリシーマップ適用 =====
zone-pair security ZP_IN_OUT
 service-policy type inspect PM_IN_OUT

zone-pair security ZP_OUT_SELF
 service-policy type inspect PM_OUT_SELF

! ===== 6. インターフェイスのゾーンバインド =====
interface GigabitEthernet0/0/1
 zone-member security INSIDE

interface GigabitEthernet0/0/2
 zone-member security OUTSIDE

interface GigabitEthernet0/0/3
 zone-member security DMZ
```

### 2. Cisco IOS XE: Control Plane Policing (CoPP) の構成

```bash
! ===== 1. コントロールプレーン向けトラフィックの分類 =====
ip access-list extended ACL_COPP_CRITICAL
 permit ospf any any
 permit bgp any any
 permit bgp host 192.168.1.1 any

ip access-list extended ACL_COPP_MANAGEMENT
 permit tcp 10.254.0.0 255.255.0.0 any eq 22
 permit udp 10.254.0.0 255.255.0.0 any eq 161

ip access-list extended ACL_COPP_BAD_TRAFFIC
 permit icmp any any
 permit tcp any any eq www

! ===== 2. Class-Map の定義 =====
class-map match-any CM_COPP_CRITICAL
 match access-group name ACL_COPP_CRITICAL

class-map match-any CM_COPP_MANAGEMENT
 match access-group name ACL_COPP_MANAGEMENT

class-map match-any CM_COPP_BAD
 match access-group name ACL_COPP_BAD_TRAFFIC

! ===== 3. Policy-Map とポリサー (Policing / Drop) の構成 =====
policy-map PM_COPP
 class CM_COPP_CRITICAL
  police 512000 conform-action transmit exceed-action transmit
 class CM_COPP_MANAGEMENT
  police 128000 conform-action transmit exceed-action drop
 class CM_COPP_BAD
  police 8000 conform-action transmit exceed-action drop
 class class-default
  police 64000 conform-action transmit exceed-action drop

! ===== 4. Control Plane へのポリシー適用 =====
control-plane
 service-policy input PM_COPP
```

### 3. Cisco IOS XE: Management Plane Protection (MPP) & uRPF 構成

```bash
! ===== 1. Management Plane Protection (MPP) の設定 =====
control-plane host
 management-interface GigabitEthernet0/0/0 allow ssh https snmp
 management-interface GigabitEthernet0/0/1 allow ssh

! ===== 2. Unicast RPF (uRPF) の設定 =====
! 物理インターフェイスにおける Strict モード (Default Route 除外)
interface GigabitEthernet0/0/1
 ip verify unicast source reachable-via rx

! 外部 WAN ポートにおける Loose モード (Default Route 許可 ＆ ACL ログ記録)
ip access-list extended ACL_URPF_DROP_LOG
 permit ip any any

interface GigabitEthernet0/0/2
 ip verify unicast source reachable-via any allow-default ACL_URPF_DROP_LOG
```

---

## 🔍 検証コマンド

| 目的 | コマンド | 表示確認ポイント |
| :--- | :--- | :--- |
| **ZBFW アクティブセッションテーブルの確認** | <code>show policy-map type inspect zone-pair sessions</code> | 確立されたセッションの 5 プレット、プロトコル、データ量 |
| **ZBFW ポリシー適用統計・ドロップ数の確認** | <code>show policy-map type inspect zone-pair</code> | 各 Class-Map のパケット一致数、Drop / Inspect カウント |
| **CoPP ポリシー適用状態およびパケットドロップカウント確認** | <code>show policy-map control-plane</code> | Conform / Exceed パケット数および Drop カウント |
| **MPP 設定インターフェイスおよび許可プロトコル表示** | <code>show control-plane host open-ports</code> | 指定ポート上で LISTEN 状態にあるプロトコルと IP 制限 |
| **uRPF グローバルおよびインターフェイスドロップ統計表示** | <code>show cef interface GigabitEthernet0/0/1</code><br><code>show ip traffic</code> | uRPF 判定結果、Drop カウント、RPF 違反統計 |
| **NBAR2 プロトコルパックおよび識別トラフィック統計確認** | <code>show ip nbar protocol-pack active</code><br><code>show ip nbar stats top-traffic</code> | 適用プロトコルパックバージョン、L7 判別上位アプリ |
| **ZBFW 判定シーケンスのリアルタイムデバッグ** | <code>debug policy-firewall detailed</code> | パケットの Zone-Pair 判定および PAM ドロップ理由 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **ZBFW 構成後、戻りパケットがドロップされて通信が成立しない** | `inspect` ではなく単なる `pass` が指定されている | <code>show policy-map type inspect zone-pair</code> | 双方向許可が必要な `pass` をやめ、ステートフルな `inspect` に変更する。 |
| **ルータ宛ての BGP / OSPF ネイバーが Down する** | CoPP ポリサーのレートが低すぎる、または ZBFW Self Zone で切断 | <code>show policy-map control-plane</code> | 制御プロトコル用 Class-Map の CIR 値を引き上げるか、Self Zone ポリシーを修正する。 |
| **MPP 設定後、別のポートからルータへ SSH 接続できなくなった** | MPP により未指定インターフェイス上の SSH が暗黙拒否された | <code>show control-plane host open-ports</code> | `management-interface` に該当インターフェイスを追加定義する。 |
| **非対称ルーティング環境で有効なトラフィックが uRPF ドロップする** | Strict モードが有効化されている | <code>show cef interface <IF></code> | `reachable-via rx` から Loose モード (`reachable-via any`) へ構成変更する。 |
| **NBAR2 で特定アプリ (例: WebEx) が分類できず class-default に入る** | Protocol Pack が古く最新シグネチャに対応していない | <code>show ip nbar protocol-pack active</code> | 最新の `.pack` ファイルを [suspicious link removed] から入手し `ip nbar protocol-pack` で更新。 |

---

## ⚠ 制限事項

* **ZBFW の制限事項**:
  * 同一ゾーンに属するインターフェイス間のトラフィック（Intra-Zone）は、デフォルトで全許可となります（`zone-member` を設定した時点で異ゾーン間はデフォルト拒否）。
  * 1 つの物理/論理インターフェイスは 1 つの Security Zone にのみ所属可能です。
* **uRPF の制限事項**:
  * uRPF は **CEF (Cisco Express Forwarding) が有効化されている環境でのみ動作** します（`ip cef` が必須）。
  * トンネルインターフェイス（GRE, VTI, IPsec）上の uRPF は、カプセル化解除後のインナー IP ヘッダーに対して評価されます。
* **CoPP の制限事項**:
  * ソフトウェア転送ルータ（IOS）とハードウェア/ASIC 転送スイッチ（Catalyst 9000 シリーズ等）では、CoPP のハードウェアポリシーマップの制限やサポートコマンド（`police` アクションのみ対応等）が異なります。

---

## 🔄 他技術との関連

* **1.2.a NAT**: ZBFW パイプラインにおいて、**Ingress NAT は ZBFW の前**、**Egress NAT は ZBFW の後** に処理されます。ZBFW の ACL / Class-Map は **Real (変換前) IP アドレス** を基準に作成します。
* **IPsec / FlexVPN**: FlexVPN や GRE/IPsec トンネルを ZBFW の保護対象とする場合、トンネル物理インターフェイスと Virtual-Template / Tunnel インターフェイスのどちらを Zone にバインドするか明確に設計する必要があります。
* **QoS (MQC)**: CoPP や AVC は MQC (Modular QoS CLI) 構造を直接使用するため、`class-map` および `policy-map` の設定文法をそのまま共有します。

---

## 🧩 比較表

### 1. Control Plane Policing (CoPP) vs Control Plane Protection (CPPr) vs Management Plane Protection (MPP)

| 比較項目 | Control Plane Policing (CoPP) | Control Plane Protection (CPPr) | Management Plane Protection (MPP) |
| :--- | :--- | :--- | :--- |
| **制御対象** | コントロールプレーン全体（単一パス） | コントロールプレーンを **3 サブサブコンテキスト** に分割 | **インバンド管理プロトコル (SSH/SNMP 等)** |
| **細分化レベル** | 粗い (All Punt Traffic) | 細かい (Host, Transit, CEF-Receive) | ポート/プロトコル限定 (Interface-specific) |
| **設定場所** | `control-plane` | `control-plane host / transit / cef-receive` | `control-plane host` |
| **用途** | ルータ CPU 全体の DoS / Rate Limit 保護 | 宛先別 (自 IP 宛 / 転送中 / 局所パケット) の精密制御 | 管理アクセスインターフェイスの完全限定 |

### 2. uRPF Strict Mode vs Loose Mode

| 比較項目 | uRPF Strict Mode | uRPF Loose Mode |
| :--- | :--- | :--- |
| **検証コマンド** | `ip verify unicast source reachable-via rx` | `ip verify unicast source reachable-via any` |
| **ドロップ条件** | 着信ポート ＝ CEF ベストパス出力ポート でない場合 | CEF テーブル内に送信元 IP へのルートが存在しない場合 |
| **非対称ルート対応** | **不可**（パケットがドロップされる） | **可能**（到達可能な別経路があれば許可） |
| **推奨適用場所** | 単一アップリンク WAN, ISP エッジ, アクセスポート | BGP マルチホーム, 非対称メッシュ網 |

---

## 💡 ベストプラクティス

1. **Self Zone ポリシーの明示的定義**:
   * ZBFW 導入時、ルータ自装置宛ての ICMP/SSH や、ルータ発信の NTP/DNS/RADIUS 通信が遮断されないよう、`self` ゾーンを組み込んだ Zone-Pair (`Inside-to-Self`, `Self-to-Outside`) を必ず明示的に定義します。
2. **CoPP における `class-default` のドロップ回避と慎重なポリシング**:
   * CoPP の `class-default` で一律 `drop` を設定すると、未知の制御通信や ARP パケットが廃棄されルータが分離されます。`class-default` は十分な帯域幅を与えた `police` のみとし、`drop` は明確に分類された悪意ある Class-Map にのみ適用します。
3. **uRPF における `allow-default` の意図的な制御**:
   * デフォルトルートを持つルータで Loose モードを設定する際、`allow-default` を付けるとあらゆる IP アドレスが uRPF を通過してしまいます。インターネットエッジで偽装 IP を確実に止めるには `allow-default` を除外して評価します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ZBFW: インサイドからアウトサイドへのステートフル HTTP/HTTPS/DNS インスペクション構成

#### 問題
ルータ R1 の `GigabitEthernet0/0/1` (`INSIDE`) から `GigabitEthernet0/0/2` (`OUTSIDE`) へのトラフィックに対し、HTTP、HTTPS、DNS 通信のみをステートフルインスペクション許可せよ。その他のトラフィックはドロップしてログを記録せよ。

#### 要件
* Zone-Pair 名: `IN_TO_OUT`
* ZBFW CBM (Class-Based Policy) を使用すること。

#### 設定例

```bash
! 1. セキュリティゾーンの作成
zone security INSIDE
zone security OUTSIDE

! 2. ゾーンペアの定義
zone-pair security IN_TO_OUT source INSIDE destination OUTSIDE

! 3. Class-Map の定義
class-map type inspect match-any CM_PERMITTED_SERVICES
 match protocol http
 match protocol https
 match protocol dns

! 4. Policy-Map の定義
policy-map type inspect PM_IN_TO_OUT
 class CM_PERMITTED_SERVICES
  inspect
 class class-default
  drop log

! 5. ゾーンペアへの適用とインターフェイス割り当て
zone-pair security IN_TO_OUT
 service-policy type inspect PM_IN_TO_OUT

interface GigabitEthernet0/0/1
 zone-member security INSIDE

interface GigabitEthernet0/0/2
 zone-member security OUTSIDE
```

#### 検証方法
* INSIDE 端末 (`10.1.1.50`) から OUTSIDE Web サーバー (`203.0.113.100`) へ HTTP リクエストを送信。
* R1 で `show policy-map type inspect zone-pair sessions` を実行し、ステートテーブルにセッションが自動生成され、戻りパケットが許可されることを確認。

---

### 例2: ZBFW: Self Zone を使用した外部からのインバンド SSH 管理アクセスの限定許可

#### 問題
外部ネットワーク (`OUTSIDE`: `GigabitEthernet0/0/2`) からルータ R1 自身 (`self` ゾーン) 宛ての通信において、特定管理ホスト `198.51.100.50` からの SSH アクセスのみを許可し、その他の外部からのアクセスを全て拒否せよ。

#### 要件
* Self Zone を使用すること。

#### 設定例

```bash
zone security OUTSIDE

zone-pair security OUT_TO_SELF source OUTSIDE destination self

ip access-list extended ACL_ADMIN_HOST
 permit tcp host 198.51.100.50 any eq 22

class-map type inspect match-all CM_SSH_ADMIN
 match access-group name ACL_ADMIN_HOST
 match protocol ssh

policy-map type inspect PM_OUT_TO_SELF
 class CM_SSH_ADMIN
  pass
 class class-default
  drop

zone-pair security OUT_TO_SELF
 service-policy type inspect PM_OUT_TO_SELF

interface GigabitEthernet0/0/2
 zone-member security OUTSIDE
```

#### 検証方法
* 管理ホスト `198.51.100.50` から R1 宛てに SSH 接続を試み疎通することを確認。
* 未許可ホスト (`198.51.100.99`) から SSH を試み、`show policy-map type inspect zone-pair` で Drop カウントが増加することを確認。

---

### 例3: ZBFW: HTTP パラメータマップによる不審なヘッダー・ポートのチューニング

#### 問題
ZBFW において、標準以外のポート (TCP 8080) を使用する HTTP トラフィックを検査可能とし、かつ HTTP リクエストヘッダー長が 2048 バイトを超える不正パケットをドロップするパラメータマップを構成せよ。

#### 要件
* `parameter-map type inspect http` を使用すること。

#### 設定例

```bash
! 1. HTTP パラメータマップの構成
parameter-map type inspect http PMAP_HTTP_TUNING
 request-length max-header 2048
 port-misbehave log

! 2. PAM (Port to Application Mapping) の拡張 (8080 を HTTP として定義)
ip port-map http port tcp 8080

! 3. Class-Map および Policy-Map へのパラメータマップ適用
class-map type inspect match-all CM_HTTP_CUSTOM
 match protocol http

policy-map type inspect PM_HTTP_STRICT
 class CM_HTTP_CUSTOM
  inspect PMAP_HTTP_TUNING
```

#### 検証方法
* `show ip port-map http` を実行し、8080 ポートが HTTP にマップされていることを確認。
* 大きなヘッダーを持つ HTTP 通信を発生させ、パケットが制限違反でドロップされることを確認。

---

### 例4: CoPP: BGP/OSPF 制御通信の保護と SYN Flood 攻撃緩和ポリシング

#### 問題
ルータ R1 のコントロールプレーンにおいて、BGP (TCP 179) および OSPF 通信を最優先 (1Mbps まで許容) で保護し、ICMP 通信を 64kbps に制限し、未定義の SYN トラフィック過多（SYN Flood）を 32kbps でドロップせよ。

#### 要件
* `control-plane` に `service-policy input` を適用すること。

#### 設定例

```bash
! 1. 分類用 ACL の作成
ip access-list extended ACL_CRITICAL_ROUTING
 permit ospf any any
 permit tcp any any eq bgp
 permit tcp any eq bgp any

ip access-list extended ACL_ICMP_TRAFFIC
 permit icmp any any

ip access-list extended ACL_SYN_FLOOD
 permit tcp any any match-all +syn

! 2. Class-Map の定義
class-map match-any CM_ROUTING
 match access-group name ACL_CRITICAL_ROUTING

class-map match-any CM_ICMP
 match access-group name ACL_ICMP_TRAFFIC

class-map match-any CM_SYN
 match access-group name ACL_SYN_FLOOD

! 3. Policy-Map の定義
policy-map PM_COPP_PROTECT
 class CM_ROUTING
  police 1000000 conform-action transmit exceed-action transmit
 class CM_ICMP
  police 64000 conform-action transmit exceed-action drop
 class CM_SYN
  police 32000 conform-action transmit exceed-action drop
 class class-default
  police 128000 conform-action transmit exceed-action drop

! 4. Control Plane への適用
control-plane
 service-policy input PM_COPP_PROTECT
```

#### 検証方法
* `show policy-map control-plane` を実行し、各クラスでの Conform / Exceed パケット数を確認。
* 大量の ICMP パケット（`ping -f` 等）を打った際、`CM_ICMP` クラスで Exceed パケットがドロップされることを検証。

---

### 例5: MPP: インバンド管理インターフェイス制限 (Gig0/0/0 のみ SSH/SNMP 許可)

#### 問題
ルータ R1 において、アウトオブバンド管理ポート `GigabitEthernet0/0/0` からのみ SSH および SNMP アクセスを受信許可し、他の全インターフェイスからの管理通信を拒否せよ。

#### 要件
* `control-plane host` を使用すること。

#### 設定例

```bash
control-plane host
 management-interface GigabitEthernet0/0/0 allow ssh snmp
```

#### 検証方法
* `show control-plane host open-ports` を実行し、GigabitEthernet0/0/0 上でのみ TCP 22 および UDP 161 が OPEN 状態であることを確認。
* `GigabitEthernet0/0/1` の IP 宛てに SSH 接続を試み、接続がタイムアウト（拒否）されることを確認。

---

### 例6: uRPF: アクセスポートにおける Strict モードとドロップログ記録

#### 問題
ユーザーアクセスポート `GigabitEthernet0/0/1` 上で uRPF Strict モードを有効化し、逆引きルーティング不一致でドロップされたパケットを ACL ログ経由で記録せよ。

#### 要件
* `ip verify unicast source reachable-via rx` を使用すること。

#### 設定例

```bash
ip access-list extended ACL_URPF_LOG
 permit ip any any log

interface GigabitEthernet0/0/1
 ip verify unicast source reachable-via rx ACL_URPF_LOG
```

#### 検証方法
* 送信元 IP アドレスを偽装したパケットを `GigabitEthernet0/0/1` に注入。
* `show cef interface GigabitEthernet0/0/1` および Syslog (`%SEC-6-IPACCESSLOGP`) で uRPF ドロップログが出力されることを確認。

---

### 例7: uRPF: BGP マルチホーム環境における Loose モード (Default Route 除外) 構成

#### 問題
デュアル ISP に接続された WAN インターフェイス `GigabitEthernet0/0/2` において、非対称ルーティングに対応するため uRPF Loose モードを構成せよ。ただし、デフォルトルートのみで一致する偽装 IP アドレスは到達不可能としてドロップさせよ。

#### 要件
* `allow-default` を除外した Loose モードを構成すること。

#### 設定例

```bash
interface GigabitEthernet0/0/2
 ip verify unicast source reachable-via any
```

#### 検証方法
* FIB テーブルに明示的ルートが存在しない送信元 IP を持つパケットが到達した際、Loose モードによって正しくドロップされることを確認。

---

### 例8: FPM: TCP SYN パケットのヘッダーフラグ検出とカスタムパターンフィルタリング

#### 問題
FPM (Flexible Packet Matching) を使用して、TCP ヘッダー内の SYN-FIN フラグが同時にセットされている異常なパケット（Xmas 攻撃スキャン）を検出してドロップせよ。

#### 要件
* FPM パラメータファイル（`ip.cdl`, `tcp.cdl`）をロードして使用すること。

#### 設定例

```bash
! 1. プロトコル定義ファイルのロード
load protocol flash:ip.cdl
load protocol flash:tcp.cdl

! 2. Stack Class-Map の定義 (IP ➔ TCP の順序定義)
class-map type stack match-all CM_STACK_TCP
 match start ip version 4
 match next-header tcp

! 3. Access-Control Class-Map の定義 (TCP フラグオフセット検索)
class-map type access-control match-all CM_FPM_SYN_FIN
 match field TCP flags eq 0x03  ! SYN (0x02) + FIN (0x01) = 0x03

! 4. Policy-Map の定義
policy-map type access-control PM_FPM_DROP
 class CM_FPM_SYN_FIN
  drop

policy-map type service PM_FPM_SERVICE
 class CM_STACK_TCP
  service-policy PM_FPM_DROP

! 5. インターフェイス適用
interface GigabitEthernet0/0/1
 service-policy type service input PM_FPM_SERVICE
```

#### 検証方法
* `show policy-map type service interface GigabitEthernet0/0/1` を実行し、FPM マッチ数およびドロップカウントを確認。

---

### 例9: AVC / NBAR2: P2P トラフィックの検出・QoS マーキングおよび帯域制限

#### 問題
NBAR2 を使用して `BitTorrent` および `eDonkey` アプリケーションを識別し、該当トラフィックの DSCP 値を `CS1` (Scavenger) に書き換えた上で 128kbps に帯域制限（Police）せよ。

#### 要件
* NBAR2 (`match protocol`) と MQC を使用すること。

#### 設定例

```bash
! 1. NBAR2 による P2P アプリの分類
class-map match-any CM_P2P_TRAFFIC
 match protocol bittorrent
 match protocol edonkey

! 2. Policy-Map の構成 (DSCP マーキング ＆ ポリシング)
policy-map PM_AVC_POLICE
 class CM_P2P_TRAFFIC
  set dscp cs1
  police 128000 conform-action transmit exceed-action drop

! 3. インターフェイス適用
interface GigabitEthernet0/0/1
 ip nbar protocol-discovery
 service-policy input PM_AVC_POLICE
```

#### 検証方法
* `show ip nbar stats top-traffic` で P2P トラフィックが検出されていることを確認。
* `show policy-map interface GigabitEthernet0/0/1` で DSCP 書き換えおよび Exceed ドロップ数を確認。

---

### 例10: 統合セキュリティ構成: ZBFW + CoPP + uRPF + MPP の一括実装

#### 問題
エッジルータ R1 において、以下の要件を満たす包括的セキュリティポリシーを定義せよ。
1. `INSIDE` から `OUTSIDE` への全 IP 通信を ZBFW でステートフルインスペクション許可。
2. インターフェイス `Gig0/0/0` 上で uRPF Strict モードを有効化。
3. ルータ管理（SSH）は `Gig0/0/0` からのみ許可 (MPP)。
4. OSPF 制御通信を CoPP で保護。

#### 要件
* 全機能を矛盾なく統合構成すること。

#### 設定例

```bash
! ===== 1. ZBFW 構成 =====
zone security INSIDE
zone security OUTSIDE

zone-pair security ZP_IN_OUT source INSIDE destination OUTSIDE

class-map type inspect match-any CM_ALL_IP
 match protocol tcp
 match protocol udp
 match protocol icmp

policy-map type inspect PM_IN_OUT
 class CM_ALL_IP
  inspect
 class class-default
  drop

zone-pair security ZP_IN_OUT
 service-policy type inspect PM_IN_OUT

interface GigabitEthernet0/0/0
 zone-member security INSIDE
 ip verify unicast source reachable-via rx  ! ===== 2. uRPF Strict =====

interface GigabitEthernet0/0/1
 zone-member security OUTSIDE

! ===== 3. MPP 構成 =====
control-plane host
 management-interface GigabitEthernet0/0/0 allow ssh

! ===== 4. CoPP 構成 =====
ip access-list extended ACL_OSPF
 permit ospf any any

class-map match-all CM_COPP_OSPF
 match access-group name ACL_OSPF

policy-map PM_COPP_INTEGRATED
 class CM_COPP_OSPF
  police 256000 conform-action transmit exceed-action drop

control-plane
 service-policy input PM_COPP_INTEGRATED
```

#### 検証方法
* `show policy-map type inspect zone-pair sessions` で ZBFW セッションを確認。
* `show control-plane host open-ports` で MPP のオープンポートを確認。
* `show policy-map control-plane` で OSPF コントロールプレーンのポリシングを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ルータ R1 上で以下の ZBFW および ACL コンフィグが適用されている。INSIDE ゾーン（`10.1.1.0/24`）のクライアントから OUTSIDE ゾーンの HTTP サーバー宛てに接続を開始した。パケット処理動作として正しいものはどれか。

```bash
zone security INSIDE
zone security OUTSIDE
zone-pair security ZP_IN_OUT source INSIDE destination OUTSIDE

class-map type inspect match-all CM_HTTP
 match protocol http

policy-map type inspect PM_IN_OUT
 class CM_HTTP
  pass

zone-pair security ZP_IN_OUT
 service-policy type inspect PM_IN_OUT
```

* A) パケットはステートフルインスペクションされ、戻りの Web レスポンスも自動的に許可される。
* B) 往路の HTTP リクエストは通過するが、復路のレスポンスパケットは ZBFW のステートテーブルが作られないため戻りポートでドロップされる。
* C) `pass` アクションが指定されているため、暗黙のステートテーブルが生成されて双方向通信が可能となる。
* D) `CM_HTTP` に ACL が定義されていないため、パケットは即座にドロップされる。

**正解**: **B**
**解説**: ZBFW の `pass` アクションは、ステートテーブル（Connection State）を生成しない**単方向の許可（Unidirectional Pass）**として動作します。そのため、往路のリクエストは通過しますが、戻り（OUTSIDE ➔ INSIDE）のゾーンペアポリシーが存在しないか許可されていない場合、復路のパケットはドロップされます。ステートフルに復路を許可するには `inspect` アクションを使用する必要があります。

---

### 2. トラブルシュート問題

**問題**: 管理者が R1 上で CoPP (Control Plane Policing) を適用した後、BGP ピアリング（対向: `192.168.12.2`）が突然 `ACTIVE` 状態と `IDLE` 状態を繰り返し、最終的に Down した。`show policy-map control-plane` の出力は以下の通りである。解決策として最も適切なものはどれか。

```text
Class-map: CM_COPP_BGP (match-all)
  50 packets, 3200 bytes
  5-minute offered rate 0000 bps, drop rate 1000 bps
  Match: access-group name ACL_BGP
  police:
      cir 8000 bps, bc 1500 bytes
      conformed 10 packets, 640 bytes; actions: transmit
      exceeded 40 packets, 2560 bytes; actions: drop
```

* A) `ACL_BGP` を削除して BGP トラフィックのポリシングを完全に停止する。
* B) BGP クラスの CIR（Commit Information Rate）値を引き上げるか、`exceed-action` を `transmit` に変更する。
* C) ルータ上で BGP プロセスを再起動する。
* D) `control-plane` を `control-plane host` に変更する。

**正解**: **B**
**解説**: 出力結果から、BGP トラフィック（`CM_COPP_BGP`）の offered rate が設定された CIR (8000 bps) を超過し、`exceeded` パケットが大量にドロップ（`actions: drop`）されていることがわかります。制御パケットが欠損したことで BGP Hold Timer がタイムアウトし、ピアが切断されました。CIR 値を適切な帯域幅へ増やすか、`exceed-action transmit` に調整する必要があります。

---

### 3. Design 問題

**問題**: エンタープライズのインターネット境界ルータにおいて、パブリック IP アドレス空間から送信される IP スプーフィング（偽装送信元 IP）パケットを最前線で排除したい。ルータは単一の ISP と BGP デフォルトルートのみで接続されている。最も効果的かつ最小の運用負荷で実装できるデータプレーンセキュリティ技術はどれか。

* A) 全パブリック IP レンジを網羅した Extended ACL をインバウンド適用する。
* B) WAN インターフェイス上で uRPF Strict モード (`ip verify unicast source reachable-via rx`) を有効化する。
* C) Control Plane Protection (CPPr) を有効化する。
* D) FPM (Flexible Packet Matching) で送信元 IP のマスクを作成する。

**正解**: **B**
**解説**: 単一アップリンク構成（非対称ルートが発生しない環境）において、IP スプーフィング対策として最も効果的なのは **uRPF Strict モード** です。CEF テーブルを参照し、インgress ポートと最適逆引き経路が一致しないパケットを動的・自動的にドロップできます。

---

### 4. 実装問題

**問題**: ルータ R1 の管理ポート `GigabitEthernet0/0/0` 上でのみ SSH および HTTPS 管理接続を許可し、他の全インターフェイスからの管理通信を拒否する MPP を構成したい。正しいコマンド記述はどれか。

* A) `control-plane host` 配下で `management-interface GigabitEthernet0/0/0 allow ssh https` を設定する。
* B) `interface GigabitEthernet0/0/0` 配下で `ip access-group` を設定する。
* C) `control-plane` 配下で `service-policy input MPP_POLICY` を適用する。
* D) `zone-pair security` で `management-interface` キーワードを指定する。

**正解**: **A**
**解説**: Management Plane Protection (MPP) は `control-plane host` サブコンテキスト配下で `management-interface <IF_NAME> allow <PROTOCOLS>` コマンドを使用して構成します。

---

### 5. コンフィグ読解問題

**問題**: 以下の uRPF コンフィグがインターフェイス `GigabitEthernet0/0/2` に適用されている。送信元 IP が `172.16.1.1` のパケットが到着した。CEF ルーティングテーブル上、`172.16.1.1` へのルートは存在しないが、デフォルトルート `0.0.0.0/0` のみが存在する。このパケットの処理結果として正しいものはどれか。

```bash
interface GigabitEthernet0/0/2
 ip verify unicast source reachable-via any
```

* A) Loose モードであり、デフォルトルートが存在するためパケットは許可される。
* B) `allow-default` キーワードが指定されていないため、デフォルトルートは到達可能パスとして認められず、パケットはドロップされる。
* C) Strict モードとして処理され、インプレースドロップされる。
* D) CEF が無効化され、プロセススイッチングにフォールバックする。

**正解**: **B**
**解説**: `ip verify unicast source reachable-via any` (Loose モード) において、`allow-default` キーワードを付与しない場合、uRPF は**明示的なプレフィックスルート（FIB エントリ）の存在のみを要求**します。デフォルトルート (`0.0.0.0/0`) しか存在しない送信元 IP は到達不能とみなされてドロップされます。

---

## 🔗 参考リソース

* [Cisco IOS XE 17.x Security Configuration Guide - Zone-Based Policy Firewall]([suspicious link removed])
* [Cisco IOS XE 17.x Security Configuration Guide - Control Plane Policing (CoPP)]([suspicious link removed])
* [Cisco Technical Notes: Understanding and Configuring Management Plane Protection (MPP)]([suspicious link removed])
* [Cisco Technical Notes: Unicast Reverse Path Forwarding (uRPF) Enhancement]([suspicious link removed])
* [Cisco Live: Network Foundation Protection (NFP) Architecture Deep Dive (BRKSEC-2003)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * IOS/IOS XE のセキュリティ機能群は、**「どのプレーン（Data / Control / Management）を誰から守るか」** を頭の中でマッピングすることが合格への近道です。
  * 特に **CoPP** と **ZBFW Self Zone** は試験での併用シナリオが多く、両者の評価順序（ZBFW ➔ CoPP）を意識した設定設計が重要です。
* **図解（NFP フレームワークと IOS XE セキュリティ機能のマッピング）**:
  ```text
  ┌─────────────────────────────────────────────────────────────┐
  │              Cisco Network Foundation Protection            │
  ├─────────────────────────────────────────────────────────────┤
  │ [ Management Plane ]  ➔ MPP, SSHv2, TACACS+/RADIUS, SNMPv3   │
  │ [ Control Plane ]     ➔ CoPP, CPPr, BGP MD5, OSPF Auth      │
  │ [ Data Plane ]        ➔ ZBFW, uRPF, FPM, AVC/NBAR2, PortSec │
  └─────────────────────────────────────────────────────────────┘
  ```
* **注意点**:
  * 実機ラボ環境で CoPP を投入する際は、設定を保存（`copy run start`）する前に必ず別端末から Ping / SSH テストを行い、誤って制御アクセスを完全にシャットアウトしていないか検証してください。
