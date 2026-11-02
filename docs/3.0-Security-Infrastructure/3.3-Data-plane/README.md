---
layout: default
title: 3.3-Data-plane
nav_order: 3
parent: 3.0-Security-Infrastructure
---

# 3.3 Data plane protection techniques

Cisco ネットワークセキュリティアーキテクチャにおける **データプレーン保護技術（Data Plane Protection Techniques）** は、ネットワークアプライアンス（Cisco IOS/IOS-XE ルーター、Cisco Catalyst スイッチ、Cisco ASA、Cisco FTD）を通過するユーザーデータトラフィック（Transit Traffic）の健全性、真正性、および可用性をリアルタイムに保障するための多層防御メカニズム群です。

CCIE Security v6.1 Blueprint **3.3 Data plane protection techniques** では、L2 から L4/L7 に至るデータプレーンの攻撃手法（IP 送信元偽装、MAC アドレス枯渇/スプーミング、DHCP 枯渇/Rogue Server、ARP ポイズニング、TCP SYN Flood、ブロードキャストストーム等）に対する防御技術の実装能力が多角的に評価されます。具体的には、**Unicast Reverse Path Forwarding (uRPF)**、**Layer 2 Infrastructure Protection (Port Security, DHCP Snooping, DAI, IPSG, Storm Control)**、**Access Control Lists (PACL, VACL, RACL, Time-based, Reflexive ACL)**、**TCP SYN Flood 防御（TCP Intercept, ASA Embryonic Connection Limits, Threat Detection）**、ならびに **Flexible Packet Matching (FPM)** および **NetFlow/IPFIX によるトラフィックモニタリング** の設計・設定・トラブルシューティングが対象となります。

---

## 📘 概要

* **機能概要**:
  * **Unicast Reverse Path Forwarding (uRPF)**: CEF (Cisco Express Forwarding) テーブルと連携し、受信パケットの送信元 IP アドレスが適切な着信インターフェイスのルーティングテーブルに存在するかを検証し、IP アドレス偽装パケット（IP Spoofing）をラインレートで即座にドロップする機能。
  * **Layer 2 Data Plane Security**:
    * **Port Security**: スイッチポートに接続を許可する MAC アドレスの数と固有値を制限し、MAC アドレスポイズニングや CAM テーブル枯渇攻撃を防ぐ。
    * **DHCP Snooping**: 信頼（Trusted）および非信頼（Untrusted）ポートを区別し、Rogue DHCP サーバーからの不正メッセージ排除と DHCP Binding Table（MAC, IP, VLAN, Port マッピング）の動的生成を行う。
    * **Dynamic ARP Inspection (DAI)**: DHCP Binding Table を参照して ARP パケットの IP/MAC 整合性を検証し、Man-in-the-Middle (MitM) 攻撃の原因となる ARP ポイズニングを阻止する。
    * **IP Source Guard (IPSG)**: DHCP Binding Table または静的バインディングに基づき、ポートごとの IP/MAC パケットフィルタリング（IP/MAC Source Binding）を自動適用して IP 偽装を防ぐ。
    * **Storm Control**: ブロードキャスト、マルチキャスト、または不明なユニキャスト（DLF）のトラフィックレベルを監視し、設定された閾値（pps、bps、またはパーセンテージ）を超えた場合にパケットドロップまたは Err-disable 化を行う。
  * **Access Control Lists (ACLs)**:
    * **PACL (Port ACL)**: L2 物理ポートまたは EtherChannel の着信（Ingress）パケットに対して直接適用される ACL。
    * **VACL (VLAN ACL / VLAN Access Map)**: 同一 VLAN 内の L2 ブリッジング通信を含む全トラフィックに対して適用される ACL。
    * **RACL (Router ACL)**: L3 インターフェイス（SVI、Routed Port）を通過するルーテッドトラフィックに対して適用される通常の ACL。
    * **Time-based / Reflexive ACL**: 時刻制限に基づく動的フィルタリング、または内部からのセッション戻り通信のみを動的に許可するステートフル風フィルタリング。
  * **TCP Connection Protection & DoS Mitigation**:
    * **TCP Intercept**: IOS/IOS-XE ルーター上で動作し、ハーフオープン TCP 接続の到達を監視・代理応答（Watch Mode / Intercept Mode）して SYN Flood 攻撃から背後サーバーを保護する。
    * **ASA/FTD Embryonic Connection Limits**: アプライアンス上で許可するハーフオープン（Embryonic）接続の最大数を制限し、閾値超過時に SYN Cookie や接続抑制を発動する。
* **利用目的**:
  * データプレーンを流れるトラフィックに対する不正アクセス、DoS/DDoS 攻撃、IP/MAC 偽装、Man-in-the-Middle 攻撃の排除。
  * 境界型ルーター/ファイアウォールおよびアクセス層スイッチにおける多層的なセキュリティポリシーの強制。
* **どのような場面で利用するか**:
  * エンタープライズのアクセス層スイッチ（Port Security, DHCP Snooping, DAI, IPSG の全社展開）。
  * 外部プロバイダ/インターネット接続境界（uRPF, iACL, TCP SYN Flood 限界値設定）。
  * 内部セグメント間 VLAN ルーティングの精密な制御（VACL, RACL）。

---

## 🔑 要点

| 技術要素 | 主な特徴 | 主な用途 | メリット | デメリット / 制限事項 | 設計上の注意点 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **uRPF (Strict)** | 送信元 IP が受信用 CEF インターフェイスと完全一致するか検証 | 外部境界ルーターでの IP 偽装防止 | ラインレートでの IP Spoofing パケット排除 | 非対称ルーティング（Asymmetric Routing）環境で正常パケットがドロップ | マルチホーム構成では Strict モード使用不可（Loose モードを選択） |
| **uRPF (Loose)** | 送信元 IP が FIB（FIB テーブル）上に存在するかのみ検証 | ISP 境界や非対称経路が存在するネットワーク | 非対称経路を許容しつつ火消し（Null0 宛先）パケットを排除 | インターネット全域からの偽装 IP（有効 IP の偽装）は防げない | `allow-default` オプションの有無でデフォルトルートの評価挙動が変化 |
| **Port Security** | ポートごとの学習 MAC 数制限と違反（Violation）アクション | アクセスポートの保護 | 未許可スイッチや大量 MAC の接続防止 | IP phone 併用時の Voice/Data VLAN 設計の複雑化 | `protect`, `restrict`, `shutdown` の挙動差異と Err-disable 自動復旧設計 |
| **DHCP Snooping** | Non-DHCP サーバーの着信阻止と Binding Table の作成 | L2 防御の基盤（DAI/IPSG の前提条件） | Rogue DHCP 排除と後続 L2 セキュリティのアンカー | スイッチ再起動時の Binding Table 消失リスク | データベースエージェント（Flash/TFTP/FTP）への Binding 保存が必須 |
| **DAI** | DHCP Binding Table による ARP パケット検証 | ARP ポイズニング / MitM 攻撃防止 | L2 での ARP 偽装の完全阻止 | Binding Table にない静的 IP 端末の通信遮断 | 静的 IP 端末用には `ip arp inspection filter`（ARP ACL）を事前設定 |
| **IPSG** | Binding Table に基づく動的 IP/MAC フィルタリング | IP スプーミング防止 | 自端末以外の IP アドレス利用防止 | Option 82 不整合や静的 IP 環境での設定負荷 | Port Security と併用する場合の `ip verify source port-security` 仕様確認 |
| **Storm Control** | Broadcast/Multicast/DLF の量監視 | ネットワークループや暴走機器の隔離 | L2 ネットワーク全体の崩壊防止 | 正常なマルチキャスト/ブロードキャストのドロップリスク | 閾値（Rising/Falling Threshold）の適切な選定 |
| **VACL** | VLAN 内の同一サブネット間（L2 ブリッジ通信）のアクセス制御 | データセンター/DMZ 内の横移動防止 | L3 ルーターを経由しない通信も捕捉・制御可能 | 設定誤りによるセグメント全体の通信断障害リスク | アクションは `forward`, `drop`, `redirect` のいずれかを指定 |
| **TCP Intercept** | ハーフオープン TCP 接続の監視と代理応答（IOS） | TCP SYN Flood 攻撃からのサーバー防衛 | サーバー側のリソース枯渇を防止 | ルーターの CPU/メモリリソース消費の増大 | ハイスループット環境では ASA/FTD の Embryonic Limit を推奨 |

---

## 🏗 動作原理

### 1. uRPF (Unicast Reverse Path Forwarding) 動作フロー

```
[ Incoming Packet ] (Source: 192.168.1.50, Dest: 10.1.1.1)
        │
        ▼
[ Ingress Interface (e.g., Gi0/1) ]
        │
        ▼
[ CEF Table (FIB) Lookup for Source IP (192.168.1.50) ]
        │
        ├──────────── Strict Mode ────────────┐
        │                                     │
        ▼                                     ▼
[ Best Outgoing Interface == Gi0/1? ]   [ Route Exists in FIB? ] (Loose Mode)
        │                                     │
   Yes  │  No                            Yes  │  No
  ┌─────┴─────┐                         ┌─────┴─────┐
  ▼           ▼                         ▼           ▼
[ Forward ] [ Drop ]                  [ Forward ] [ Drop ]
```

### 2. Layer 2 Integrated Protection Flow (DHCP Snooping -> DAI -> IPSG)

```
1. Client Sends DHCP Request
   ↓
[ Access Switch Port (Untrusted) ] ──(DHCP Snooping)──> [ DHCP Server (Trusted Port) ]
   ↓
[ DHCP Ack Returned ] ──> [ DHCP Snooping Creates Binding Entry ]
                          (MAC: AAAA.BBBB.CCCC, IP: 10.1.1.100, VLAN: 10, Port: Gi0/1)
   ↓
2. Subsequent Traffic Operations:
   ├─ ARP Packet Received ──> [ DAI Checks Binding Table ] ──> Valid? ─Yes─> [ Forward ]
   │                                                               └─No──> [ Drop ]
   └─ IP Packet Received  ──> [ IPSG Checks ACL/Binding ] ──> Valid? ─Yes─> [ Forward ]
                                                                   └─No──> [ Drop ]
```

---

## ⚙ 動作シーケンス

### L2 データプレーン保護パケット処理フロー

1. **物理ポート着信時 (Port Security & Storm Control)**:
   * スイッチポートにパケットが到着すると、まず **Storm Control** により直近のトラフィックレートが監視され、閾値を超えている場合はドロップされます。
   * 次に **Port Security** が送信元 MAC アドレスを評価します。未学習の MAC アドレスであり、最大学習数（`maximum`）を超えている場合、設定された Violation モード（`protect`, `restrict`, `shutdown`）に従って処理されます。

2. **パケット種別判定 (DHCP / ARP / IP)**:
   * **DHCP パケットの場合**:
     * 着信ポートが Untrusted の場合、DHCP OFFER や DHCP ACK などのサーバー応答メッセージであれば即座にドロップされます。
     * DHCP REQUEST / DISCOVER の場合、Option 82 フィールドの挿入検証が行われ、Trusted ポート上の DHCP サーバーへ転送されます。応答メッセージ受信後、**DHCP Snooping Binding Table** にエントリが追加されます。
   * **ARP パケットの場合 (DAI 有効時)**:
     * Untrusted ポート上の ARP パケットは **DAI** により捕捉され、パケット内の Target/Sender IP-MAC ペアが DHCP Snooping Binding Table（または ARP ACL）と照合されます。不一致の場合はドロップおよび Logging されます。
   * **IP パケットの場合 (IPSG 有効時)**:
     * Untrusted ポート上の IP パケットは **IP Source Guard** のハードウェア TCAM フィルターにより検証されます。送信元 IP アドレス（および MAC）が Binding エントリと一致しない場合、パケットはスイッチハードウェア上でドロップされます。

3. **VLAN 内フィルタリング (VACL)**:
   * VLAN 内をブリッジングされる、または L3 ルーティングされるすべてのパケットに対して **VLAN Access Map (VACL)** が評価されます。指定された IP/MAC ACL にマッチしたトラフィックは `forward`、`drop`、または `redirect` されます。

4. **L3 ルーティング & uRPF 検証**:
   * ルーテッドポートまたは SVI を通過する際、**RACL** の評価が行われます。
   * 着信インターフェイスで **uRPF** が有効化されている場合、パケットの送信元 IP アドレスに対して CEF テーブルルックアップが実行されます。
     * **Strict Mode**: FIB 上の最適出口インターフェイスと着信インターフェイスが一致しない場合、パケットをドロップします。
     * **Loose Mode**: FIB 上に到達可能ルート（有効なネクストホップ）が存在しない場合、パケットをドロップします。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、データプレーン保護技術は「制限事項付きの実装」や「複数技術の組み合わせシナリオ」として頻出します。

### 1. 試験で問われるポイントと重要トピック
* **uRPF のモード選定と ACL 連携**:
  * 非対称ルーティングが存在する環境での uRPF 配置（Strict vs Loose）。
  * `allow-default` オプションの動作理解（デフォルトルート `0.0.0.0/0` しか存在しない環境で `allow-default` を設定しないと全パケットがドロップされる）。
  * uRPF と Drop ACL (Exception ACL) の連携（特定の送信元 IP だけ uRPF チェックをバイパスさせる設定）。
* **Layer 2 ポートセキュリティ & DHCP Snooping & DAI & IPSG の依存関係**:
  * DHCP Snooping が有効化されていないと DAI および IPSG は動作しない（Binding Table が空になるため全通信障害が発生する）。
  * 静的 IP 端末が存在する環境で DAI/IPSG を正常動作させるための ARP ACL (`ip arp inspection filter`) および Static IP Source Binding (`ip source binding`) の追加設定。
  * Switch 間の Trunk ポートに対する `ip dhcp snooping trust` および `ip arp inspection trust` の設定漏れチェック。
* **PACL vs VACL vs RACL の優先順位と処理順序**:
  * 同一ポート/VLAN 上で PACL、VACL、RACL が重複した場合のパケット評価順序の理解。
  * 着信処理順序: **PACL -> VACL -> Ingress RACL**
  * 送信処理順序: **Egress RACL -> VACL**（※PACL は Ingress のみサポート）
* **TCP Intercept と ASA Embryonic Connection Limits**:
  * IOS/IOS-XE の `ip tcp intercept mode` (intercept vs watch)。
  * ASA / FTD (FMC Policy) における Threat Detection および Class-Map / Policy-Map を使用した `set connection timeout embryonic` 設定。

---

## 🛠 設定方法

### 1. Cisco IOS/IOS-XE: uRPF 設定

```text
! Strict Mode uRPF 設定 (インターフェイス単位)
interface GigabitEthernet0/1
 description OUTSIDE_INTERFACE
 ip address 203.0.113.1 255.255.255.0
 ip verify unicast source reachable-via rx

! Loose Mode uRPF + デフォルトルート許容 + 例外 ACL 設定
access-list 100 permit ip host 192.0.2.50 any   ! この送信元は uRPF チェック例外
interface GigabitEthernet0/2
 description MULTI_HOMED_INTERFACEE
 ip address 198.51.100.1 255.255.255.0
 ip verify unicast source reachable-via any allow-default 100
```

### 2. Cisco Catalyst Switch: Layer 2 統合データプレーン保護設定

```text
! 1. DHCP Snooping グローバルおよび VLAN 有効化
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option   ! ※Option 82 挿入でスイッチ/ルーター間不整合が出る場合の回避策

! DHCP Binding データベースの不揮発性保存設定
ip dhcp snooping database flash:/dhcp_snooping_db.txt

! 上位ルーター/スイッチ向き Trunk ポートを Trusted に設定
interface GigabitEthernet0/1
 description TRUNK_TO_CORE
 switchport mode trunk
 ip dhcp snooping trust
 ip arp inspection trust

! 2. アクセスポートの設定 (Port Security + DAI + IPSG + Storm Control)
interface Range GigabitEthernet1/0/1 - 24
 switchport mode access
 switchport access vlan 10
 
 ! Port Security
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security aging time 10
 switchport port-security aging type inactivity
 switchport port-security mac-address sticky

 ! Dynamic ARP Inspection (DAI)
 ip arp inspection limit rate 15 burst interval 1

 ! IP Source Guard (IPSG)
 ip verify source tracking port-security

 ! Storm Control (Rising 80%, Falling 60%)
 storm-control broadcast level 80.0 60.0
 storm-control multicast level 80.0 60.0
 storm-control action shutdown

! 3. DAI の VLAN 有効化および静的 IP 端末用 ARP ACL 設定
ip arp inspection vlan 10
ip access-list extended ARP_ACL_STATIC_SERVER
 permit ip host 10.1.1.200 mac host aaaa.bbbb.cccc
ip arp inspection filter ARP_ACL_STATIC_SERVER vlan 10 static
```

### 3. Cisco Catalyst Switch: VACL (VLAN Access Map) 設定

```text
! VLAN 10 内の 10.1.1.50 から 10.1.1.100 への Web 通信を遮断し、他は許可
ip access-list extended ACL_VACL_BLOCK
 permit tcp host 10.1.1.50 host 10.1.1.100 eq 80

vlan access-map MAP_VACL_SEC 10
 match ip address ACL_VACL_BLOCK
 action drop
vlan access-map MAP_VACL_SEC 20
 action forward

vlan filter MAP_VACL_SEC vlan-list 10
```

### 4. Cisco ASA: Embryonic Connection Limit & Threat Detection 設定

```text
! WEB_SERVER クラスに対する Embryonic Connection 制限
access-list ACL_EMBRYONIC extended permit tcp any host 192.168.1.100 eq 80

class-map CM_EMBRYONIC
 match access-list ACL_EMBRYONIC

policy-map PM_DATA_PLANE
 class CM_EMBRYONIC
  set connection timeout embryonic 0:00:30
  set connection conn-max 1000 embryonic 100

service-policy PM_DATA_PLANE interface outside

! Threat Detection (Basic & Advanced) の有効化
threat-detection basic-threat
threat-detection statistics host
threat-detection scanning-threat
```

---

## 🔍 検証コマンド

| 目的 | コマンド | 期待される確認項目 |
| :--- | :--- | :--- |
| **uRPF 確認** | `show cef interface <int>` | `unicast RPF: rx (Strict)` または `any (Loose)` の動作状態 |
| **uRPF ドロップ確認** | `show ip traffic` | `unicast RPF drops` カウンターの増加状態 |
| **Port Security 確認** | `show port-security interface <int>` | Status、CurrentAddrCount、SecurityViolation カウンター |
| **DHCP Binding 表確認**| `show ip dhcp snooping binding` | MAC アドレス、IP アドレス、VLAN、Interface バインディング一覧 |
| **DAI 動作状態確認** | `show ip arp inspection vlan <vlan>` | DAI 有効化状態、Pakets Processed / Dropped カウンター |
| **IPSG 適用状態確認** | `show ip verify source` | Filter-type (IP/MAC)、Filter-mode (active)、IP-address |
| **VACL 適用状態確認** | `show vlan filter` | VLAN に適用されている Access-Map 名と対象 VLAN 一覧 |
| **Storm Control 確認** | `show storm-control <int>` | Rising/Falling 閾値、Current Rate、Suppressed カウンター |
| **ASA Embryonic 確認** | `show local-host` / `show conn detail` | Embryonic 接続数、ハーフオープンセッションの制限超過状態 |

---

## 🚨 トラブルシュート

| 症状 | 主な原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **uRPF 有効化後に正常通信がドロップされる** | 非対称経路の存在、またはデフォルトルート依存時の `allow-default` 漏れ | `show ip route` <br> `show ip traffic` | Strict モードから Loose モードへ変更するか、`allow-default` や例外 ACL を適用 |
| **DHCP Snooping 有効化後、クライアントが IP 取得不可** | 上位 Trunk ポートに対する `ip dhcp snooping trust` 設定漏れ | `show ip dhcp snooping` | 上位ルーター/DHCP サーバー向きのポートに `ip dhcp snooping trust` を追加 |
| **DAI 有効化後、静的 IP 端末同士が通信不可** | DHCP Binding Table に静的 IP 端末のエントリが存在しない | `show ip dhcp snooping binding` <br> `show ip arp inspection` | 静的 IP 端末用に ARP ACL (`ip arp inspection filter`) を定義して VLAN に適用 |
| **IPSG 有効化後、ポート上の全 IP 通信が遮断される** | Binding Table の非存在、または Option 82 不整合による Binding 失敗 | `show ip verify source` | Binding Table の生成状態を確認し、手動で `ip source binding` を追加設定 |
| **Port Security でポートが Err-disable 化する** | ポート移動（Sticky/Static MAC）や未許可機器の接続（Violation） | `show interfaces status err-disabled` | `errdisable recovery cause psecure-violation` を設定するかポートを `shutdown`/`no shutdown` |

---

## ⚠ 制限事項

* **uRPF 制限事項**:
  * uRPF は **Cisco Express Forwarding (CEF)** が有効化されている環境でのみ動作します (`ip cef`)。
  * GRE トンネルや IPsec 仮想インターフェイス上の uRPF 評価にはプラットフォーム制限が存在する場合があります。
* **Layer 2 統合保護の制限事項**:
  * **DHCP Snooping**: データベースエージェント（Flash などのストレージ）を設定しない場合、スイッチ再起動時に Binding Table が全消去され、DAI/IPSG により全通信が遮断される重篤な障害が発生します。
  * **IPSG**: PACL や EtherChannel 構成時のハードウェア TCAM 制限に注意が必要です。
* **ASA/FTD Embryonic Limits**:
  * 大小さまざまな TCP セッションを処理する環境で設定値を小さくしすぎると、正規の大量アクセスユーザーが遮断されます。

---

## 🔄 他技術との関連

* **Control Plane Protection (CoPP / CPPr)**:
  * データプレーン保護（uRPF や L2 Security）はデータトラフィックをフィルタリングしますが、コントロールプレーン保護（CoPP）はルーター CPU 自体の保護を担い、両者は相互補完的に動作します。
* **Infrastructure ACL (iACL)**:
  * iACL はインフラストラクチャ IP アドレス宛てのデータプレーン通信を制限し、uRPF と組み合わせることで送信元 IP 偽装攻撃を完璧に防御します。
* **Cisco Identity Services Engine (ISE) & TrustSec (SGT)**:
  * 認証後端末のアクセス制限において、L2 セキュリティ（Port Security / DAI / IPSG）はネットワーク進入時の物理的な整合性を担保し、ISE/SGT は動的かつ論理的なグループベースアクセス制御（SGACL）を提供します。


## 🧩 比較表

### 1. PACL vs VACL vs RACL の機能比較

| 項目 | PACL (Port ACL) | VACL (VLAN ACL) | RACL (Router ACL) |
| :--- | :--- | :--- | :--- |
| **適用場所** | 物理ポート、EtherChannel | VLAN 全体（VLAN Access Map） | L3 インターフェイス（SVI、Routed Port） |
| **処理レイヤー** | Layer 2（着信のみ） | Layer 2 & Layer 3 | Layer 3（Ingress & Egress） |
| **同一 VLAN 内 L2 通信** | 捕捉可能（ポート着信時） | 捕捉可能（ブリッジング通信を含む） | 捕捉不可（L2 内で完結するため） |
| **方向** | Ingress のみ | VLAN 入出力全方向（Implicit Any） | Ingress ＆ Egress |
| **動作アクション** | `permit`, `deny` | `forward`, `drop`, `redirect` | `permit`, `deny` |

### 2. uRPF Strict Mode vs Loose Mode 比較

| 項目 | Strict Mode (`reachable-via rx`) | Loose Mode (`reachable-via any`) |
| :--- | :--- | :--- |
| **検証条件** | 送信元 IP が FIB に存在し、かつ**着信ポートと一致** | 送信元 IP が FIB に存在すること（着信ポート不問） |
| **非対称経路 (Asymmetric)**| 不可（パケットが即座にドロップされる） | 許可（別ポートからの着信でも FIB にあれば通過） |
| **デフォルトルート** | `allow-default` がなければデフォルトルートでの一致を無視 | `allow-default` がなければデフォルトルートでの一致を無視 |
| **推奨適用場所** | エンタープライズの単一接続境界ポート | ISP ピアリング、マルチホーム BGP 境界 |

---

## 💡 ベストプラクティス

1. **Layer 2 ファーストホップセキュリティの全域展開**:
   * アクセス層スイッチでは、DHCP Snooping、Dynamic ARP Inspection (DAI)、IP Source Guard (IPSG)、Port Security、Storm Control を組み合わせた「First Hop Security (FHS)」を標準化します。
2. **DHCP Snooping Binding 表の長期的保護**:
   * `ip dhcp snooping database flash:/dhcp_snooping.db` コマンド等で不揮発性ストレージまたは外部 TFTP サーバーに定期保存し、スイッチ再起動時の全通信停止トラブルを防ぎます。
3. **uRPF モードの適切な選定**:
   * 単一構成のアクセスポートやエッジ境界には Strict モード、BGP マルチホーム接続や非対称経路が存在する構成には Loose モードを適用します。
4. **VACL による Intra-VLAN 横移動（East-West）通信の遮断**:
   * データセンターや DMZ セグメントにおいて、同一 VLAN 内のサーバー間不正通信を防御するために VACL または Private VLANs (PVLAN) を適用します。


## 📝 ラボ学習・設定サンプル例

### 演習 1: uRPF Strict モードと例外 ACL の実装
* **問題**: 外出先からのリモートアクセス拠点との境界ルーター Gi0/1 において、IP 偽装パケットをドロップする uRPF Strict モードを設定しなさい。ただし、特定のテスト用送信元 IP `198.51.100.50` からの通信のみ、uRPF チェックに失敗しても通信を許可しなさい。
* **要件**:
  * Gi0/1 インターフェイスで Strict uRPF を有効化する。
  * ACL 10 を作成し、`198.51.100.50` を許可して uRPF 例外として指定する。
* **設定例**:
  ```text
  access-list 10 permit host 198.51.100.50
  
  interface GigabitEthernet0/1
   ip address 203.0.113.1 255.255.255.0
   ip verify unicast source reachable-via rx 10
  ```
* **検証方法**:
  * `show cef interface GigabitEthernet0/1` を実行し、`unicast RPF: rx with ACL 10` と表示されることを確認する。
  * 送信元を偽装したパケットを送信し、`show ip traffic` で `unicast RPF drops` カウンターが増加することを確認する。

### 演習 2: BGP マルチホーム境界における uRPF Loose モードとデフォルトルート許容
* **問題**: 2 社の ISP と BGP ピアリングを行っている境界ルーター Gi0/2 において、非対称経路を許容しつつ IP 偽装パケットを排除する uRPF Loose モードを構成しなさい。なお、デフォルトルート経由の送信元も到達可能とみなすこと。
* **要件**:
  * Loose モード (`reachable-via any`) を設定する。
  * `allow-default` オプションを追加する。
* **設定例**:
  ```text
  interface GigabitEthernet0/2
   ip address 198.51.100.1 255.255.255.0
   ip verify unicast source reachable-via any allow-default
  ```
* **検証方法**:
  * `show cef interface GigabitEthernet0/2` で `unicast RPF: any (allow default)` を確認する。

### 演習 3: DHCP Snooping の不揮発性データベース保存と Trunk トラスト設定
* **問題**: スイッチ SW1 において VLAN 10, 20 に対する DHCP Snooping を有効化しなさい。DHCP サーバーは Gi0/1 (Trunk) の先に存在します。また、スイッチ再起動時に Binding Table が消失しないよう Flash ドライブへ保存しなさい。
* **要件**:
  * VLAN 10, 20 で DHCP Snooping を有効化する。
  * Gi0/1 を Trusted ポートに設定する。
  * データベースを `flash:/dhcp_db.txt` に書き出すよう設定する。
* **設定例**:
  ```text
  ip dhcp snooping
  ip dhcp snooping vlan 10,20
  ip dhcp snooping database flash:/dhcp_db.txt
  
  interface GigabitEthernet0/1
   switchport mode trunk
   ip dhcp snooping trust
  ```
* **検証方法**:
  * `show ip dhcp snooping` で有効化状態および Trusted ポートを確認する。
  * `show ip dhcp snooping binding` でクライアントの Binding 情報が生成されることを確認する。

### 演習 4: Dynamic ARP Inspection (DAI) と静的 IP 端末用 ARP ACL
* **問題**: SW1 の VLAN 10 で DAI を有効化しなさい。VLAN 10 内には DHCP を利用しない静的 IP サーバー（IP: `10.1.1.200`, MAC: `aaaa.bbbb.cccc`）が存在するため、このサーバーの ARP 通信がドロップされないよう対策を講じなさい。
* **要件**:
  * DAI を VLAN 10 で有効化する。
  * ARP ACL `ARP_STATIC` を作成し、サーバーの IP/MAC ペアを許可する。
  * `ip arp inspection filter` コマンドで VLAN 10 に適用する。
* **設定例**:
  ```text
  ip access-list extended ARP_STATIC
   permit ip host 10.1.1.200 mac host aaaa.bbbb.cccc
  
  ip arp inspection filter ARP_STATIC vlan 10 static
  ip arp inspection vlan 10
  ```
* **検証方法**:
  * `show ip arp inspection vlan 10` を実行し、Filter 名と設定状態を確認する。

### 演習 5: IP Source Guard (IPSG) と Port Security の統合
* **問題**: アクセスポート Gi1/0/5 において、DHCP Binding に基づく IP アドレス偽装防止（IPSG）および MAC アドレス最大学習数制限（Port Security）を統合設定しなさい。
* **要件**:
  * ポートモードを access（VLAN 10）にする。
  * Port Security を有効化し、最大学習数を 2、Violation を restrict に設定する。
  * IPSG を有効化し、Port Security と連携させる。
* **設定例**:
  ```text
  interface GigabitEthernet1/0/5
   switchport mode access
   switchport access vlan 10
   switchport port-security
   switchport port-security maximum 2
   switchport port-security violation restrict
   ip verify source tracking port-security
  ```
* **検証方法**:
  * `show ip verify source` を実行し、Filter-type が `IP option 82` または `IP/MAC` で active になっているか確認する。


### 演習 6: Port Security の Sticky MAC と Err-disable 自動復旧
* **問題**: 端末接続ポート Gi1/0/10 において、動的に学習した MAC アドレスを Running-config に自動登録（Sticky）させ、違反発生時はポートを Shutdown (Err-disable) にしなさい。また、5 分（300 秒）後に Err-disable から自動復旧させなさい。
* **要件**:
  * Sticky MAC を有効化する。
  * Violation モードを `shutdown` に設定する。
  * Err-disable の自動復旧原因 `psecure-violation` とタイマー 300 秒を設定する。
* **設定例**:
  ```text
  errdisable recovery cause psecure-violation
  errdisable recovery interval 300
  
  interface GigabitEthernet1/0/10
   switchport mode access
   switchport access vlan 10
   switchport port-security
   switchport port-security mac-address sticky
   switchport port-security violation shutdown
  ```
* **検証方法**:
  * `show port-security interface GigabitEthernet1/0/10` で Sticky エントリと Shutdown モードを確認する。
  * `show errdisable recovery` でタイマーと対象機能を確認する。

### 演習 7: Storm Control によるブロードキャスト/マルチキャスト制限
* **問題**: スイッチポート Gi1/0/15 において、ブロードキャストおよびマルチキャストの帯域圧迫を防止するため、帯域使用率 70% 超過でドロップを開始し、50% 以下で解除する Storm Control を設定しなさい。
* **要件**:
  * Broadcast および Multicast の Rising 70.0%、Falling 50.0% を設定する。
* **設定例**:
  ```text
  interface GigabitEthernet1/0/15
   storm-control broadcast level 70.0 50.0
   storm-control multicast level 70.0 50.0
  ```
* **検証方法**:
  * `show storm-control GigabitEthernet1/0/15` で設定閾値と現在のトラフィックレベルを確認する。

### 演習 8: VACL による同一 VLAN 内の不審通信ドロップ
* **問題**: VLAN 20 内の全端末からセキュリティ監査サーバー（`10.2.2.99`）に対する Telnet 通信（TCP 23）のみを L2 レベルでドロップし、その他のすべての通信は許可する VACL を構成しなさい。
* **要件**:
  * IP ACL で TELNET 通信を定義する。
  * VLAN Access Map `VACL_TELNET_BLOCK` を作成し、Sequence 10 で Drop、Sequence 20 で Forward を設定する。
  * VLAN 20 に適用する。
* **設定例**:
  ```text
  ip access-list extended ACL_TELNET
   permit tcp any host 10.2.2.99 eq 23
  
  vlan access-map VACL_TELNET_BLOCK 10
   match ip address ACL_TELNET
   action drop
  vlan access-map VACL_TELNET_BLOCK 20
   action forward
  
  vlan filter VACL_TELNET_BLOCK vlan-list 20
  ```
* **検証方法**:
  * `show vlan filter` で VLAN 20 へのマッピングを確認する。
  * VLAN 20 上の端末から `10.2.2.99:23` へ接続テストを行い、遮断されることを確認する。

### 演習 9: Cisco IOS TCP Intercept (Watch Mode) の実装
* **問題**: 内部 Web サーバー（`192.168.10.50`）宛ての TCP SYN Flood 攻撃を防ぐため、ルーター上で TCP Intercept を Watch モードで構成しなさい。ハーフオープン閾値を 300 接続、タイムアウトを 15 秒に設定しなさい。
* **要件**:
  * 対象サーバーを指定する ACL 101 を作成する。
  * Intercept モードを `watch` に設定する。
  * `ip tcp intercept one-minute high 300` および `ip tcp intercept watch-timeout 15` を設定する。
* **設定例**:
  ```text
  access-list 101 permit tcp any host 192.168.10.50
  
  ip tcp intercept list 101
  ip tcp intercept mode watch
  ip tcp intercept one-minute high 300
  ip tcp intercept watch-timeout 15
  ```
* **検証方法**:
  * `show tcp intercept connections` で進行中の監視セッションを確認する。
  * `show tcp intercept statistics` でドロップ/応答統計を確認する。

### 演習 10: Cisco ASA Embryonic Connection Limits と Threat Detection
* **問題**: Cisco ASA の outside インターフェイスにおいて、DMZ ゾーン内の Database サーバー（`172.16.1.100`）宛てのハーフオープン接続が 200 を超えた場合に SYN Cookie を発動させ、Basic Threat Detection を有効化しなさい。
* **要件**:
  * ACL、Class-Map、Policy-Map を作成して Embryonic Limit 200 を適用する。
  * `threat-detection basic-threat` を有効化する。
* **設定例**:
  ```text
  access-list ACL_DB_EMBRYONIC extended permit tcp any host 172.16.1.100 eq 3306
  
  class-map CM_DB
   match access-list ACL_DB_EMBRYONIC
  
  policy-map PM_PROTECT
   class CM_DB
    set connection embryonic 200
  
  service-policy PM_PROTECT interface outside
  threat-detection basic-threat
  ```
* **検証方法**:
  * `show service-policy interface outside` を実行し、Embryonic 限度超過時のドロップ数を確認する。
  * `show threat-detection` で検出された脅威イベントを確認する。


## ❓ 想定試験問題

### 問題 1 (コンフィグ読解)
以下の Catalyst スイッチのコンフィグが適用されています。

```text
ip dhcp snooping
ip dhcp snooping vlan 10
ip arp inspection vlan 10

interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10

interface GigabitEthernet1/0/24
 switchport mode trunk
```

PC1（`10.1.1.10`）が Gi1/0/1 に接続され、DHCP サーバーが Gi1/0/24 の先に存在します。PC1 が IP アドレスを取得しようとしましたが、DHCP OFFER レスポンスが届かず通信が失敗しました。この原因として最も適切なものを 1 つ選びなさい。

* **A**: DAI が有効になっているため、DHCP パケットが拒否された。
* **B**: Gi1/0/24（Trunk）ポートに `ip dhcp snooping trust` が設定されていないため、DHCP サーバーからの応答メッセージがドロップされた。
* **C**: Gi1/0/1 に `ip verify source` が設定されていないため。
* **D**: `ip dhcp snooping information option` が有効になっているため。

**正解**: **B**  
**解説**: DHCP Snooping が有効な場合、デフォルトで全ポートが Untrusted となります。Untrusted ポートから着信する DHCP サーバー応答メッセージ（DHCP OFFER / ACK）は即座にドロップされます。DHCP サーバーが存在する上位 Trunk ポート（Gi1/0/24）に `ip dhcp snooping trust` を設定しない限り、クライアントは IP アドレスを取得できません。

---

### 問題 2 (トラブルシューティング)
BGP マルチホーム構成の境界ルーターにおいて、外部インターフェイス Gi0/0 に `ip verify unicast source reachable-via rx`（Strict uRPF）を設定したところ、一部の外部 Web サイトからの戻りパケットが意図せずドロップされる事象が発生しました。この問題の原因と最適な対処方法の組み合わせとして正しいものはどれですか。

* **A**: CEF が無効化されているため。`ip cef` を有効化する。
* **B**: 非対称ルーティング（Asymmetric Routing）が発生しており、着信インターフェイスと FIB の最適出口ポートが一致しない。`reachable-via any`（Loose Mode）に変更する。
* **C**: デフォルトルートしか存在しないため。`allow-default` を削除する。
* **D**: TCP SYN パケットが拒否されているため。uRPF 例外 ACL に `permit tcp any any` を追加する。

**正解**: **B**  
**解説**: Strict Mode uRPF (`reachable-via rx`) は、送信元 IP 宛てのベストパスの出口インターフェイスと、実際のパケット着信インターフェイスが完全一致することを要求します。マルチホーム BGP 環境などで非対称ルーティングが発生している場合、パケットは正常であっても不一致となりドロップされます。この場合は着信ポートを問わない Loose Mode (`reachable-via any`) への変更が適切な対処となります。

---

### 問題 3 (Design / 実装)
アクセス層スイッチにおいて、静的 IP アドレスを使用しているプリンター端末が存在する VLAN 20 に対し、DAI（Dynamic ARP Inspection）を有効化することになりました。DHCP Binding Table に存在しないこのプリンター（IP: `192.168.20.50`, MAC: `0011.2233.4455`）の通信を正常に維持するために実施すべき正しい手順はどれですか。

* **A**: プリンターが接続されているアクセスポートに `ip arp inspection trust` を設定する。
* **B**: `ip access-list extended` で IP アドレスと MAC アドレスを許可する ARP ACL を作成し、`ip arp inspection filter <ACL名> vlan 20 static` コマンドを適用する。
* **C**: `ip verify source` コマンドをアクセスポートから削除する。
* **D**: DHCP Snooping を VLAN 20 で無効化する。

**正解**: **B**  
**解説**: DAI は DHCP Snooping Binding Table を参照して ARP パケットを検証します。静的 IP 端末は Binding Table に載らないため、そのままでは ARP パケットが拒否されます。これを回避するためには、IP と MAC のペアを定義した ARP ACL (`ip access-list extended`) を作成し、`ip arp inspection filter <ACL名> vlan <VLAN_ID> static` コマンドで適用する必要があります。アクセスポートを `trust` にすると、そのポートからの偽装 ARP もすべて許可されてしまうためセキュリティ上推奨されません。


## 🔗 参考リソース

* Cisco Configuration Guide: Unicast Reverse Path Forwarding (uRPF)
* Cisco Catalyst Configuration Guide: DHCP Features and DHCP Snooping
* Cisco Catalyst Configuration Guide: Dynamic ARP Inspection (DAI)
* Cisco Catalyst Configuration Guide: IP Source Guard (IPSG)
* Cisco ASA Configuration Guide: Connection Limits and Threat Detection
* Cisco Live Presentation: Layer 2 Security Mechanisms

---

## 📝 **補足（Notes）**

* **First Hop Security (FHS) 統合における適用順序のまとめ**:
  * スイッチポートにおける FHS 機能の処理優先順位:
    1. **Storm Control** (レート制限)
    2. **Port Security** (L2 接続許可数制限)
    3. **DHCP Snooping** (DHCP メッセージ検証 & Binding 登録)
    4. **DAI** (ARP パケットの Binding 表照合)
    5. **IPSG** (IP パケットの Binding 表照合)
* **uRPF と Access Control List の相互作用**:
  * uRPF に ACL を指定する場合（例: `ip verify unicast source reachable-via rx 10`）、ACL で `permit` されたパケットは **uRPF チェックをパス**（例外許可）し、`deny` されたパケットは通常の uRPF インターフェイス一致検証を受けます。標準的な Access Group と判定ロジックの意味合いが異なるため、ラボ試験でのコンフィグ読解時に注意してください。


