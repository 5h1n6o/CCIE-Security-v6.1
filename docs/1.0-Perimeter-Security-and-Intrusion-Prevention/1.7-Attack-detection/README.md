---
layout: default
title: 1.7-Attack-detection
nav_order: 7
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.7 Detect and mitigate common types of attacks

Cisco Secure Firewall Threat Defense (FTD)、Management Center (FMC)、Cisco ASA、および Cisco Catalyst スイッチ/ルーターにおける **Detect and mitigate common types of attacks（一般的な攻撃ベクトルの検知と緩和）** は、現代の高度化・多層化したサイバー攻撃（DoS/DDoS、IP/MAC/ARP/DHCP Spoofing、Reconnaissance/Scanning、Evasion Techniques、Man-in-the-Middle 等）に対し、ネットワークのレイヤー 2 (Data Link) からレイヤー 7 (Application)、およびデバイスの Control/Management Plane に至る全領域で包括的なセキュリティコントロールを統合・適用する核心的なフレームワークです。

CCIE Security v6.1 Blueprint **1.7** では、スイッチレベルでの Layer 2 セキュリティ機能（DHCP Snooping、Dynamic ARP Inspection、IP Source Guard、Port Security、IPv6 First Hop Security）、境界ルーターおよび ASA/FTD での Layer 3/4 セキュリティ機能（Unicast RPF、TCP Intercept、ASA Threat Detection、CoPP/CPPr）、ならびに FTD/Snort エンジンにおけるノーマライゼーション（Evasion 防御）や Security Intelligence / Umbrella 連携 まで含めた、多層防御（Defense-in-Depth）の設計・実装・検証・トラブルシューティング能力が厳格に評価されます。

---

## 📘 概要

* **機能概要**:
  * **DoS/DDoS 緩和**: 大量の不審パケットや接続試行（TCP SYN Flood、UDP/ICMP Flood、Reflected/Amplification 攻撃）に対し、TCP Intercept (Embryonic Limits)、ASA Threat Detection、Control Plane Policing (CoPP)、および Rate Limiting を用いてサーバーおよびネットワーク機器の可用性を保護する。
  * **Spoofing（成りすまし）防止**: 偽造された IP/MAC アドレスや不正な ARP/DHCP メッセージを検知・遮断するため、L2 スイッチでの DHCP Snooping、Dynamic ARP Inspection (DAI)、IP Source Guard (IPSG)、および L3 での Unicast Reverse Path Forwarding (uRPF) を適用する。
  * **Reconnaissance/Scanning（偵察・スキャン）検知**: 攻撃の初期段階である Ping Sweep や TCP/UDP ポートスキャンを ASA Threat Detection や FTD Snort Portscan Preprocessor で動的検知し、自動 Shun (Dynamic Drop) を適用する。
  * **Evasion（回避技術）対策**: IP フラグメンテーション、TCP ストリーム分割、Overlapping パケット等による IPS 回避試行を FTD Snort Stream / Inline Normalization プレプロセッサで再構築・正規化して解読する。
  * **Man-in-the-Middle (MitM) 防御**: ARP Cache Poisoning や Rogue DHCP サーバーによる通信盗聴・改ざんを L2 スイッチのバインドデータベース連携で阻止する。
* **利用目的**:
  * ネットワークインフラの 3 大要素である CIA（機密性・完全性・可用性）を脅かす攻撃ベクトルを各境界で無効化する。
  * 攻撃の各フェーズ（Reconnaissance ➔ Delivery ➔ Exploitation ➔ C2）に応じた多段階の阻止ポイント（Control Points）を確立する。
* **どのような場面で利用するか**:
  * アクセス層スイッチにおける社内 LAN の横移動（Lateral Movement）および L2 攻撃防御。
  * 企業ネットワーク境界（WAN エッジ / DMZ）での IP 成りすまし攻撃および DoS/DDoS 攻撃の早期水際遮断。
  * コア/ディストリビューションルーターにおける CPU 保護（Control Plane 保護）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **対象攻撃種別** | DoS/DDoS、IP/MAC/ARP/DHCP Spoofing、Reconnaissance (Scan)、Evasion、MitM、Buffer Overflow / Malware |
| **L2 防御機能 (Switches)** | **DHCP Snooping**、**Dynamic ARP Inspection (DAI)**、**IP Source Guard (IPSG)**、**Port Security**、**IPv6 FHS (RA Guard, DHCPv6 Guard)** |
| **L3 防御機能 (Routers/ASA)** | **Unicast RPF** (Strict/Loose Mode)、**TCP Intercept** (Watch/Intercept)、**CoPP/CPPr**、**ASA Threat Detection**、**BGP RTBH** |
| **L7/NGFW (FTD/FMC)** | **Snort Preprocessors** (Stream, Normalization, Portscan)、**Security Intelligence** (IP/DNS/URL)、**AMP/File Policy** |
| **DNS/Cloud 防御** | **Cisco Umbrella** (DNS Layer Security, Sinkhole, DGA / Co-occurrence Models) |
| **動作プレーン区分** | **Management Plane** (MPP)、**Control Plane** (CoPP/CPPr)、**Data Plane** (ACL, uRPF, DAI, IPS, Threat Detection) |
| **制限事項** | uRPF Strict Mode は非対称ルーティング環境で正常パケットを誤ドロップする。IPSG/DAI は DHCP Snooping データベースに完全依存する |

---

## 🏗 動作原理

各レイヤーおよびセキュリティデバイスにおける攻撃検知・緩和の統合アーキテクチャ：

```text
[ Attacker / Untrusted Zone ]
       │
       ├── 1. Spoofed IP/MAC & Rogue Services ──► [ Switch L2 Boundary ]
       │                                            ├── DHCP Snooping (Validate Rogue DHCP / Option 82)
       │                                            ├── Port Security (Prevent CAM Flood / MAC Spoofing)
       │                                            ├── DAI (Validate ARP vs Snooping DB)
       │                                            ├── IPSG (Filter Non-Matching IP/MAC on Access Port)
       │                                            └── IPv6 FHS (RA Guard & DHCPv6 Guard)
       │
       ├── 2. Spoofed L3 Packets & SYN Floods ─► [ Edge Router / Firewall Boundary ]
       │                                            ├── uRPF (Strict/Loose: Validate Source IP in FIB)
       │                                            ├── TCP Intercept / ASA Threat Detection (SYN Proxy)
       │                                            └── CoPP / CPPr (Protect Router CPU & Control Protocols)
       │
       └── 3. Evasion / Exploits / C2 ─────────► [ FTD NGFW & Umbrella Cloud Security ]
                                                    ├── Preprocessors (Packet Normalization / Defrag / Portscan)
                                                    ├── Security Intelligence (IP/URL/DNS Sinkhole)
                                                    ├── Snort IPS Rules (Exploit & Buffer Overflow Signatures)
                                                    └── AMP / File Policy (Sandbox & Malware Disposition)
```

---

## ⚙ 動作シーケンス

1. **Reconnaissance ＆ Scanning Phase（偵察・スキャン段階）**:
   * 攻撃者が Target ネットワークに対し ICMP Ping Sweep や TCP/UDP ポートスキャンを実施。
   * ASA Threat Detection または FTD Portscan Preprocessor が短時間での接続失敗数・ポート接続率の急増（Burst/Average Threshold）を監視・検知。
   * 閾値を超過した攻撃元 IP アドレスを動的 ACL (Shun) へ自動追加し、以降のアクセスを一定時間即座にドロップ。
2. **Spoofing ＆ L2/L3 Access Phase（成りすまし・L2/L3 侵入段階）**:
   * 攻撃者が偽造 MAC/IP アドレスや Gratuitous ARP/Rogue DHCP オファーを送信し、MITM や盗聴を試行。
   * Switch の **DHCP Snooping** が Untrusted ポートからの Rogue DHCP 応答を遮断。**DAI** が ARP パケットを Snooping Binding DB と照合し、偽造 ARP をドロップ。**IPSG** がバインド情報と一致しない IP パケットをポートの PACL で拒否。
   * 境界ルーターで **uRPF** がパケットの送信元 IP を Forwarding Information Base (FIB) と照合し、着信ポート不一致（Strict Mode）またはルート不在（Loose Mode）の偽造パケットをドロップ。
3. **DoS/DDoS ＆ Exhaustion Phase（サービス不能攻撃段階）**:
   * 攻撃者が大量の TCP SYN パケットを特定 Web サーバーへ送信し、ハーフオープン接続キューを短時間で全消費させようと試みる（SYN Flood）。
   * ASA/FTD の **TCP Intercept (Embryonic Limits)** が動作を開始し、保護対象サーバーの代わりに ASA/FTD 自体が SYN Proxy（または Watch モード）としてクライアントへ SYN-ACK を応答。完全に 3-Way Handshake が完了した正常セッションのみを背後のサーバーへ引き渡し、サーバーのリソース枯渇を防止。
4. **Evasion ＆ Exploit Delivery Phase（回避・攻撃ペイロード送信段階）**:
   * 攻撃者が IP フラグメンテーションや TCP ストリーム分割、Overlapping データ等を用いて IPS のシグネチャ検知を回避（Evasion）しようと試みる。
   * FTD Snort **Stream / Inline Normalization** プレプロセッサが、パケットをデータプレーン上で正しく再構築 (Reassembly) および正規化 (Normalization) し、隠蔽された Buffer Overflow や脆弱性攻撃シグネチャを正確に検知してインライン遮断。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **L2 Switch Security の完全なバインド依存順序**:
   * **DHCP Snooping ➔ DAI / IPSG の依存関係**: Dynamic ARP Inspection (DAI) や IP Source Guard (IPSG) は、原則として **DHCP Snooping Binding Database** を参照して動作します。DAI/IPSG を設定する前に、必ずグローバル (`ip dhcp snooping`) および対象 VLAN (`ip dhcp snooping vlan <X>`) で DHCP Snooping を有効化しておく必要があります。
   * **Option 82 の問題 (Option 82 Issue)**: スイッチで DHCP Snooping を有効にすると、デフォルトで Option 82 (Relay Agent Information) が挿入されます。直接接続された Cisco IOS DHCP サーバーは `giaddr=0` かつ Option 82 付きの DHCP パケットを拒否するため、サーバー側で `ip dhcp relay information trust-all` を投入するか、スイッチ側で **`no ip dhcp snooping information option`** を投入して Option 82 挿入を停止させる必要があります。
2. **uRPF (Unicast RPF) Strict vs Loose Mode の使い分けと例外**:
   * **Strict Mode (`reachable-via rx`)**: 送信元 IP が FIB 内に存在し、かつ**パケットが着信したインターフェイスと CEF のベストリターンポートが完全一致** することを確認します。非対称ルーティング（In Ingress ≠ Out Egress）環境では正常パケットを誤ドロップするため使用不可です。
   * **Loose Mode (`reachable-via any`)**: 送信元 IP が FIB（ルーティングテーブル）内に存在するかのみを確認します（Null0 宛て等以外は通過）。ISP 境界や BGP RTBH (Remote Triggered Black Hole) で利用されます。
   * **`allow-default` キーワード**: uRPF でデフォルトルート (`0.0.0.0/0`) を到達可能なルートとして認めるか否かの設定。インターネット境界での Strict uRPF 適用時に `allow-default` を忘れると、外部からの全パケットがドロップされる危険があります。
3. **ASA Threat Detection ＆ Embryonic Connection Limits (MPF)**:
   * **ASA Threat Detection**: `threat-detection basic-threat`（デフォルト有効）、`threat-detection scanning-threat shun`（ポートスキャン自動検出・shun）、`threat-detection statistics tcp-intercept`（統計保持）。
   * **Embryonic Connection Limits (TCP Intercept)**: MPF (`policy-map`) 内で **`set connection embryonic-conn-max <N>`** を指定し、ハーフオープン接続数が閾値を超えた場合に ASA が SYN Proxy 動作に移行してサーバーを保護します。
4. **CoPP (Control Plane Policing) ＆ CPPr (Control Plane Protection)**:
   * **CoPP**: `control-plane` インターフェイスに `service-policy input <policy>` をアサインし、ルーター CPU に着信・処理されるトラフィック（BGP, OSPF, SSH, ICMP 等）を Rate Limit (Policing) または Drop して DoS 防御します。
   * **CPPr**: `control-plane` を 3 つのサブインターフェイス（`host`, `transit`, `cef-exception`）に細分化して、さらに厳密な制御と Port Filtering を実現します。

---

## 🛠 設定方法

### 1. Switch L2 Attack Mitigation (DHCP Snooping + DAI + IPSG + Port Security)

```bash
! ===== Step 1: DHCP Snooping の設定 =====
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option   ! Option 82 による DHCP サーバー拒否の防止

interface GigabitEthernet1/0/24          ! Uplink / DHCP Server Port
 ip dhcp snooping trust                  !
 ip dhcp snooping limit rate 100         !

! ===== Step 2: Dynamic ARP Inspection (DAI) の設定 =====
ip arp inspection vlan 10,20             !
ip arp inspection validate src-mac dst-mac ip   ! サニティチェックの有効化

interface GigabitEthernet1/0/24
 ip arp inspection trust                 !

! ===== Step 3: IP Source Guard (IPSG) ＆ Port Security の設定 =====
interface range GigabitEthernet1/0/1 - 10   ! Untrusted Access Ports
 switchport mode access
 switchport access vlan 10
 switchport port-security                !
 switchport port-security maximum 2       !
 switchport port-security violation shutdown !
 ip verify source port-security          ! IP ＋ MAC の同時検証
```

---

### 2. Router uRPF ＆ CoPP (Control Plane Protection)

```bash
! ===== uRPF Strict Mode with ACL Exemption =====
ip access-list extended ACL_URPF_EXEMPT
 permit ip 192.168.99.0 0.0.0.255 any   ! 例外許可

interface GigabitEthernet0/0
 ip verify unicast source reachable-via rx allow-default 101 !

! ===== CoPP (Control Plane Policing) =====
ip access-list extended ACL_ROUTER_MGMT
 permit tcp any host 10.1.1.1 eq 22     ! SSH Management

ip access-list extended ACL_ROUTING_PROTOS
 permit ospf any any
 permit bgp any any

class-map match-all CM_MGMT
 match access-group name ACL_ROUTER_MGMT

class-map match-all CM_ROUTING
 match access-group name ACL_ROUTING_PROTOS

policy-map PM_COPP
 class CM_ROUTING
  police 800000 10000 conform-action transmit exceed-action transmit
 class CM_MGMT
  police 64000 8000 conform-action transmit exceed-action drop
 class class-default
  police 8000 1000 conform-action transmit exceed-action drop

control-plane                            !
 service-policy input PM_COPP            !
```

---

### 3. ASA Threat Detection ＆ TCP Intercept (MPF)

```text
! ===== ASA Threat Detection の設定 =====
threat-detection basic-threat            !
threat-detection scanning-threat shun   ! ポートスキャン検出と自動 Shun
threat-detection statistics tcp-intercept !

! ===== ASA MPF による Embryonic Connection Limit (TCP Intercept) =====
access-list ACL_WEB_SERVER extended permit tcp any host 192.168.100.10 eq 80

class-map CM_WEB_SERVER
 match access-list ACL_WEB_SERVER

policy-map PM_PROTECT_SERVER
 class CM_WEB_SERVER
  set connection embryonic-conn-max 100 per-client-embryonic-max 10 !

service-policy PM_PROTECT_SERVER interface OUTSIDE !
```

---

## 🔍 検証コマンド

| 目的 | コマンド (IOS / ASA / FTD) | 表示確認ポイント |
| :--- | :--- | :--- |
| **DHCP Snooping バインド DB 確認** | <code>show ip dhcp snooping binding</code> | MAC アドレス、割り当て IP、VLAN、Interface |
| **DAI 統計および拒否パケット確認** | <code>show ip arp inspection vlan 10</code> | Validated パケット数、Drop 判定件数 |
| **IPSG フィルター状態確認** | <code>show ip verify source</code> | Filter-type (ip / ip-mac)、Filter-mode (active) |
| **uRPF 状態・ドロップカウンター確認** | <code>show ip interface Gi0/0</code> / <code>show cef interface Gi0/0</code> | `IP verify source reachable-via RX`、`verification drops` |
| **CoPP ポリシーカウンター確認** | <code>show policy-map control-plane</code> | Class ごとの conform/exceed パケット数 |
| **ASA Threat Detection 情報確認** | <code>show threat-detection rate</code> / <code>show threat-detection scanning-threat</code> | ポートスキャン検出履歴、Shun された IP アドレス |
| **ASA TCP Intercept セッション確認** | <code>show service-policy interface OUTSIDE</code> | `current embryonic conns`, `drop` 件数 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **DHCP Snooping 有効化後、クライアントが IP アドレスを取得できない** | Uplink インターフェイスで `ip dhcp snooping trust` が投入されていない、または Option 82 拒否 | <code>show ip dhcp snooping</code> | DHCP サーバー宛て Uplink ポートを Trust 設定する。IOS DHCP サーバーの場合は Option 82 無効化 (`no ip dhcp snooping information option`)。 |
| **DAI 有効化後、スタティック IP 端末間の通信が完全に切断される** | DHCP Snooping DB にエントリが存在せず、全 ARP パケットが Drop される | <code>show ip arp inspection log</code> | ARP Access List を作成して `ip arp inspection filter <ACL> vlan <VLAN>` で手動バインドを適用する。 |
| **uRPF Strict Mode 設定後、特定拠点からの通信がランダムに切断される** | 非対称ルーティング (Asymmetric Routing) により、戻りトラフィックと異なるポートに着信している | <code>show ip interface</code><br><code>show ip route</code> | 非対称環境では uRPF を **Loose Mode (`reachable-via any`)** に変更するか、ACL Exemption を追加する。 |
| **ASA Threat Detection で管理者端末が誤って Shun されてしまう** | スキャン検出閾値が低すぎる、または管理端末の正常な大量接続がスキャンと判定された | <code>show threat-detection scanning-threat</code> | `threat-detection scanning-threat` の閾値を調整するか、管理端末 IP を例外 ACL / Threat Detection 除外設定に追加する。 |
| **CoPP 適用後に OSPF / BGP ネイバーがフラッピングする** | CoPP ルールでルーティングプロトコルパケットが `class-default` にヒットしドロップされた | <code>show policy-map control-plane</code> | クラスマップで OSPF/BGP 用 ACL を明示定義し、`police` の conform レートに十分な帯域を割り振る。 |

---

## ⚠ 制限事項

* **uRPF のマルチホーム環境での制限**:
  * uRPF Strict Mode は、複数の外部プロバイダ（マルチホーム）を持つ環境や、非対称ルーティング（Ingress ポートと Egress ポートが別）が存在する環境では使用できません（Loose Mode を使用）。
* **IPSG ＆ DAI の DHCP 依存性**:
  * Dynamic ARP Inspection および IP Source Guard は、原則として Dynamic DHCP Snooping Database に依存します。静的 IP アドレス環境では手動の ARP ACL や Static Source Binding の登録運用負荷が発生します。
* **CoPP での出力トラフィック非制御**:
  * CoPP (Control Plane Policing) は、ルーター CPU に**着信（Ingress）するトラフィックのみ** を制御します。ルーター自身が生成する自発トラフィック（Egress）の制御はできません。

---

## 🔄 他技術との関連

* **Cisco Umbrella / DNS Layer Security**: 端末の初期アクセス（Phishing、C2 コールバック、DGA ドメイン）を DNS レベルで Sinkhole 遮断。
* **Cisco Secure Endpoint (AMP)**: 端末（Endpoint）におけるファイルレピュテーション、サンドボックス、および Retrospective (追跡調査) 分析。
* **Snort IPS ＆ Preprocessors**: トラフィックの正規化（Defragmentation/Stream Reassembly）による Normalization および脆弱性攻撃（Buffer Overflow、SQLi）の阻止。

---

## 🧩 比較表

### 1. uRPF Strict Mode vs uRPF Loose Mode

| 比較項目 | uRPF Strict Mode (`reachable-via rx`) | uRPF Loose Mode (`reachable-via any`) |
| :--- | :--- | :--- |
| **検証条件** | 送信元 IP が FIB に存在し、かつ**着信ポートがベストリターンポートと一致** | 送信元 IP が FIB（ルーティングテーブル）内に存在するかのみ |
| **主な用途** | 企業エッジ、シングルホーム接続、LAN 境界 | ISP 境界、BGP RTBH (Remote Triggered Black Hole) |
| **非対称ルーティング** | **非対応**（正常パケットを誤ドロップする） | **対応**（複数パスや非対称ルーティング環境でも利用可） |
| **デフォルトルート照合** | デフォルトで無効（`allow-default` で有効化） | デフォルトで照合対象 |

### 2. ASA Threat Detection vs ASA TCP Intercept (Embryonic Limit)

| 比較項目 | ASA Threat Detection | ASA TCP Intercept (MPF) |
| :--- | :--- | :--- |
| **主な目的** | ポートスキャン、異常ドロップ率、DoS 統計の自動検知 | TCP SYN Flood 攻撃から特定サーバーを直接保護 |
| **動作方式** | トラフィックの統計・パターンの監視および自動 Shun | SYN Proxy 動作（ASA が代わりに SYN-ACK を応答） |
| **設定コンポーネント** | グローバルコマンド `threat-detection ...` | MPF (`class-map` / `policy-map` / `set connection`) |
| **アクティブ防御** | スキャン元 IP を動的 ACL (Shun) で一定時間ドロップ | ハーフオープン接続数が閾値を超えた時点で動的プロキシ |

---

## 💡 ベストプラクティス

1. **L2 Switched Domain での多層防御バインド**:
   * アクセススイッチにおいて、`Port Security` + `DHCP Snooping` + `DAI` + `IPSG` を全アクセスポートに標準展開し、MAC/IP/ARP 攻撃をポート着信段階で全ブロックします。
2. **CoPP による Control Plane の徹底保護**:
   * ルーター/スイッチにおいて、管理系 (SSH/SNMP)、ルーティング系 (BGP/OSPF)、インフラ系 (ICMP/ARP) のクラス分けを行い、未知の未定義トラフィック (`class-default`) を厳格にレート制限して CPU 枯渇を防止します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: DHCP Snooping ＆ Option 82 制御の設定

#### 問題
VLAN 20 において Rogue DHCP サーバー攻撃を防止するため DHCP Snooping を設定せよ。

#### 要件
* Uplink ポート `Gi1/0/24` のみを Trusted とすること。
* 直接接続された IOS DHCP サーバーでエラーが発生しないよう Option 82 の挿入を無効化せよ。

#### 設定・操作手順

```bash
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 20
SW1(config)# no ip dhcp snooping information option

SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# ip dhcp snooping trust
```

#### 検証方法
* `show ip dhcp snooping` を実行し、VLAN 20 で有効、`Gi1/0/24` が Trusted であることを確認。

---

### 例2: Dynamic ARP Inspection (DAI) による ARP Poisoning 防御

#### 問題
VLAN 20 において DAI を有効化し、MAC アドレスと IP アドレスの不整合（Man-in-the-Middle 攻撃）を検知・ドロップせよ。

#### 要件
* Uplink ポート `Gi1/0/24` を Trusted とすること。
* 源泉 MAC・宛先 MAC・IP アドレスのサニティチェックを有効化すること。

#### 設定・操作手順

```bash
SW1(config)# ip arp inspection vlan 20
SW1(config)# ip arp inspection validate src-mac dst-mac ip

SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# ip arp inspection trust
```

#### 検証方法
* 偽造 ARP パケットを送出し、`show ip arp inspection vlan 20` の Invalid ARP カウンターが増加することを確認。

---

### 例3: IP Source Guard (IPSG) による IP/MAC Spoofing 防御

#### 問題
アクセスポート `Gi1/0/1` において、DHCP で割り当てられた IP/MAC アドレス以外のパケット送出をハードウェアレベルで遮断せよ。

#### 要件
* IP アドレスおよび MAC アドレスの両方を検証すること。

#### 設定・操作手順

```bash
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
SW1(config-if)# switchport port-security
SW1(config-if)# ip verify source port-security
```

#### 検証方法
* `show ip verify source` を実行し、Filter-type が `ip-mac`、Filter-mode が `active` であることを確認。

---

### 例4: Router uRPF (Strict Mode) による IP Spoofing 防御

#### 問題
ルーターの WAN インターフェイス `Gi0/0` に着信すIP パケットに対し、Strict uRPF を設定して Spoofed IP パケットを早期ドロップせよ。

#### 要件
* デフォルトルート (`0.0.0.0/0`) も到達可能ルートとして許容すること。

#### 設定・操作手順

```bash
R1(config)# ip cef
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip verify unicast source reachable-via rx allow-default
```

#### 検証方法
* `show ip interface GigabitEthernet0/0` を実行し、`IP verify source reachable-via RX, allow default` および verification drops 件数を確認。

---

### 例5: Router CoPP による Control Plane DoS 防御

#### 問題
ルーター CPU に対する SSH 接続および OSPF パケットの帯域を制限し、それ以外の不審トラフィックを厳格に Police / Drop せよ。

#### 要件
* OSPF: 500 Kbps, SSH: 100 Kbps, Default: 10 Kbps

#### 設定・操作手順

```bash
R1(config)# ip access-list extended ACL_OSPF
R1(config-ext-nacl)# permit ospf any any
R1(config)# ip access-list extended ACL_SSH
R1(config-ext-nacl)# permit tcp any host 10.1.1.1 eq 22

R1(config)# class-map CM_OSPF
R1(config-cmap)# match access-group name ACL_OSPF
R1(config)# class-map CM_SSH
R1(config-cmap)# match access-group name ACL_SSH

R1(config)# policy-map PM_COPP
R1(config-pmap)# class CM_OSPF
R1(config-pmap-c)# police 500000 conform-action transmit exceed-action transmit
R1(config-pmap)# class CM_SSH
R1(config-pmap-c)# police 100000 conform-action transmit exceed-action drop
R1(config-pmap)# class class-default
R1(config-pmap-c)# police 10000 conform-action transmit exceed-action drop

R1(config)# control-plane
R1(config-cp)# service-policy input PM_COPP
```

#### 検証方法
* `show policy-map control-plane` を実行し、各クラスでの conform / exceed パケット数を確認。

---

### 例6: ASA Threat Detection によるポートスキャン自動 Shun 設定

#### 問題
ASA においてポートスキャン（Reconnaissance）を自動検知し、攻撃元 IP アドレスを一定時間自動的に Shun（遮断）せよ。

#### 要件
* 基本脅威検知およびスキャン検知を有効化すること。

#### 設定・操作手順

```text
ASA1(config)# threat-detection basic-threat
ASA1(config)# threat-detection scanning-threat shun
ASA1(config)# threat-detection statistics tcp-intercept
```

#### 検証方法
* 外部から Nmap ポートスキャンを実行し、`show threat-detection scanning-threat` および `show shun` で攻撃元 IP が自動登録されることを確認。

---

### 例7: ASA MPF による TCP Intercept (SYN Flood 防御)

#### 問題
内部 Web サーバー (`192.168.100.10:80`) に対する TCP SYN Flood 攻撃を防ぐため、ハーフオープン接続（Embryonic Connection）の上限を 50 に制限せよ。

#### 要件
* MPF を使用して構成すること。

#### 設定・操作手順

```text
ASA1(config)# access-list ACL_HTTP_SERVER extended permit tcp any host 192.168.100.10 eq 80
ASA1(config)# class-map CM_HTTP_SERVER
ASA1(config-cmap)# match access-list ACL_HTTP_SERVER

ASA1(config)# policy-map PM_PROTECT
ASA1(config-pmap)# class CM_HTTP_SERVER
ASA1(config-pmap-c)# set connection embryonic-conn-max 50

ASA1(config)# service-policy PM_PROTECT interface OUTSIDE
```

#### 検証方法
* 大量の SYN パケットを送信し、`show service-policy interface OUTSIDE` で `current embryonic conns` および `drop` カウンターが増加することを確認。

---

### 例8: IPv6 RA Guard による Rogue Router Advertisement 遮断

#### 問題
VLAN 11 において不正な IPv6 Router Advertisement (RA) メッセージを遮断するため、IPv6 RA Guard をスイッチ上で構成せよ。

#### 要件
* ルーターポート `Gi1/0/24` のみを Router モードとし、アクセスポートを Host モードに設定すること。

#### 設定・操作手順

```bash
SW1(config)# ipv6 nd raguard policy POLICY_HOST
SW1(config-raguard)# device-role host

SW1(config)# ipv6 nd raguard policy POLICY_ROUTER
SW1(config-raguard)# device-role router

SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# ipv6 nd raguard attach-policy POLICY_HOST

SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# ipv6 nd raguard attach-policy POLICY_ROUTER
```

#### 検証方法
* `show ipv6 nd raguard policy` を実行し、ポリシーの適用状態を確認。

---

### 例9: IPv6 DHCPv6 Guard の設定

#### 問題
不正な DHCPv6 サーバーからの広告メッセージを遮断するため、スイッチ上で DHCPv6 Guard を構成せよ。

#### 要件
* ルーター宛てポート `Gi1/0/24` でのみ DHCPv6 Server 応答を許可すること。

#### 設定・操作手順

```bash
SW1(config)# ipv6 dhcp guard policy POLICY_DHCP_SERVER
SW1(config-dhcp-guard)# device-role server

SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# ipv6 dhcp guard attach-policy POLICY_DHCP_SERVER
```

#### 検証方法
* `show ipv6 dhcp guard policy` を実行し、ポリシー状態を確認。

---

### 例10: FTD Stream Preprocessor ＆ Packet Normalization による Evasion 防御

#### 問題
FTD (Snort 3) において、IP フラグメンテーションや TCP ストリーム分割による IPS 回避（Evasion）を防ぐため、Network Analysis Policy (NAP) で Normalization を構成せよ。

#### 要件
* FMC GUI から構成すること。

#### 設定・操作手順

```text
1. FMC UI で Policies > Access Control > Advanced に移動。
2. Network Analysis and Intrusion Policies セクションを開く。
3. Default Network Analysis Policy として "Balanced Security and Connectivity" を選択 (Inline Normalization 有効)。
4. Deploy を実行。
```

#### 検証方法
* FMC の Intrusion Events 画面で、TCP Normalization 関連のアラートおよびドロップログを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: スイッチ上で DHCP Snooping を有効化し、`Gi1/0/24` を Trusted ポートに設定した。しかし、直接接続された Cisco IOS DHCP サーバーからクライアントへ IP アドレスが割り当てられず、DHCP サーバーのデバッグログに `DHCPDISCOVER with option 82 on non-relay interface dropped` が出力されていた。適切な対処方法はどれか。

* A) `Gi1/0/24` ポートで Port Security を有効化する。
* B) スイッチ上で `no ip dhcp snooping information option` を実行するか、DHCP サーバー側で `ip dhcp relay information trust-all` を設定する。
* C) スイッチ上で `ip arp inspection vlan` を有効化する。
* D) スイッチの Firewall Mode を Transparent から Routed に変更する。

**正解**: **B**  
**解説**: スイッチで DHCP Snooping を有効化すると、デフォルトで DHCP パケットに Option 82（Relay Information）が挿入されます。直接接続された IOS DHCP サーバーは `giaddr=0` かつ Option 82 が付与されたパケットをデフォルトで拒否するため、Option 82 挿入を無効化するか、サーバー側で信託設定を追加する必要があります。

---

### 2. トラブルシュート問題

**問題**: ルーターの WAN インターフェイス `Gi0/0` で `ip verify unicast source reachable-via rx` を設定した。その後、外部の一部の拠点からの通信が拒否される障害が発生した。調査したところ、該当拠点からの着信パスと戻りパスが異なる非対称ルーティング (Asymmetric Routing) となっていた。通信を復旧させるための最適な変更はどれか。

* A) uRPF 設定を `ip verify unicast source reachable-via any` (Loose Mode) に変更する。
* B) インターフェイスで `no ip cef` を投入する。
* C) 該当ポートで Dynamic ARP Inspection を有効化する。
* D) CoPP ポリシーを削除する。

**正解**: **A**  
**解説**: **uRPF Strict Mode (`reachable-via rx`)** は、着信ポートと CEF のベストリターンポートの完全一致を要求します。非対称ルーティング環境ではチェックに失敗してパケットがドロップされるため、送信元 IP が FIB に存在するかのみを確認する **Loose Mode (`reachable-via any`)** へ変更する必要があります。

---

### 3. Design 問題

**問題**: 内部ネットワークの公開 Web サーバー (`10.1.1.100`) が大量の SYN Flood 攻撃を受け、TCP 接続キューが枯渇して正常ユーザーがアクセス不能となっている。Web サーバー側の設定や OS を変更することなく、ネットワーク境界の ASA ファイアウォールでこの DoS 攻撃を直接緩和するための最適な設計手法はどれか。

* A) ASA 上で uRPF Loose Mode を有効化する。
* B) MPF (Modular Policy Framework) を使用して、Web サーバー宛てトラフィックに対し `set connection embryonic-conn-max`（TCP Intercept）を設定する。
* C) Access Control List で ICMP を全拒否する。
* D) スイッチ上で Port Security を設定する。

**正解**: **B**  
**解説**: **ASA の TCP Intercept（Embryonic Connection Limit）** を MPF で設定することで、ASA が SYN Proxy として動作し、クライアントとの 3-way handshake が完了した正常セッションのみを背後の Web サーバーへ引き渡すため、Web サーバーの TCP 接続キュー枯渇を防ぐことができます。

---

### 4. 実装問題

**問題**: Catalyst スイッチにおいて、未承認のホストが手動で静的 IP アドレスを設定し、他人の IP/MAC アドレスに成りすまして通信（IP Spoofing）を行うのを未然にブロックしたい。DHCP Snooping データベースと連携してアクセスポートでの IP/MAC 送信元をハードウェア検証するスイッチ機能はどれか。

* A) Dynamic ARP Inspection (DAI)
* B) IP Source Guard (IPSG)
* C) Control Plane Policing (CoPP)
* D) BPDU Guard

**正解**: **B**  
**解説**: **IP Source Guard (IPSG)** は、DHCP Snooping データベース（または Static Binding）を元にアクセスポートでの送信元 IP/MAC アドレスを検証し、不一致の IP トラフィックをポートレベルで直接フィルタリング・ブロックします。

---

### 5. コンフィグ読解問題

**問題**: ルーターで以下の CoPP 設定が投入されている。この設定の動作に関する説明として正しいものはどれか。

```text
ip access-list extended ACL_MGMT
 permit tcp any host 10.1.1.1 eq 22

class-map CM_MGMT
 match access-group name ACL_MGMT

policy-map PM_COPP
 class CM_MGMT
  police 64000 exceed-action drop

control-plane
 service-policy input PM_COPP
```

* A) ルーターを通過する全 SSH トラフィックが 64 Kbps に制限される。
* B) ルーター自身宛て（Control Plane）の SSH 接続（ポート 22）が 64 Kbps にレート制限され、超過パケットはドロップされる。
* C) ルーターの Egress 送信 SSH パケットがドロップされる。
* D) SSH 以外の全トラフィックが自動的に拒否される。

**正解**: **B**  
**解説**: `control-plane` セクションにアサインされた `service-policy input` は、ルーター自身の CPU（Control Plane）宛てに着信（Ingress）するトラフィックに対して適用されます。定義された `CM_MGMT`（SSH 宛て）クラスの超過トラフィックがドロップされます。

---

## 🔗 参考リソース

* [Cisco Technical Notes: Understand and Configure Unicast Reverse Path Forwarding (uRPF)](https://www.cisco.com/)
* [Cisco Catalyst Security Configuration Guide: Layer 2 Attack Mitigation (DHCP Snooping, DAI, IPSG)](https://www.cisco.com/)
* [Cisco ASA Series Configuration Guide: Threat Detection and TCP Intercept (MPF)](https://www.cisco.com/)
* [Cisco IOS Security Configuration Guide: Control Plane Policing (CoPP) and CPPr](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Configuration Guide: Snort 3 Preprocessors and Normalization](https://www.cisco.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験において「Mitigate Layer 2 spoofing and L3 DoS attacks on Edge Switch and Router」が出題された場合の構成手順まとめ：
    1. **Switch L2**: `ip dhcp snooping` + `ip dhcp snooping vlan` ➔ `ip arp inspection vlan` ➔ `ip verify source port-security`（アクセスポート）+ Uplink ポートで `trust` アサイン。
    2. **Router L3**: `ip verify unicast source reachable-via rx allow-default`（WAN インターフェイス）。
    3. **Router CPU**: `control-plane` ➔ `service-policy input <PM>` (CoPP 設定)。
    4. **ASA / FTD**: `threat-detection` 構成 ➔ MPF で `set connection embryonic-conn-max` アサイン。
* **図解（L2 ↔ L3 攻撃緩和連携図）**:
  ```text
  [ Untrusted Host ]
          │
          ├── (1) L2 Access Port Protection
          │        ├── DHCP Snooping: Block Rogue DHCP Reply
          │        ├── Port Security: Limit MAC Count
          │        ├── DAI: Validate ARP against Binding DB
          │        └── IPSG: Verify IP/MAC Matching
          │
          ├── (2) L3 Network Edge Protection
          │        ├── uRPF: Verify Source IP in FIB (Strict/Loose)
          │        └── RTBH: Blackhole Attack Sources via BGP
          │
          └── (3) Firewall / CPU Protection
                   ├── ASA Threat Detection: Auto-Shun Scanning Hosts
                   ├── TCP Intercept: SYN Proxy for Embryonic Limits
                   └── CoPP / CPPr: Police CPU Ingress Traffic
  ```
* **注意点**:
  * ラボ試験で DAI や IPSG を設定した後は、`show ip dhcp snooping binding` および `show ip verify source` / `show ip arp inspection` を実行し、バインド DB が正しく作成され、検証ドロップカウンターが意図通り動作していることを必ず検証してください。

---

💡 **次ステップのご提案**:
次回の Blueprint 項目（例: 2.0 シリーズや他のセキュリティ防御機能等）についての学習メモ作成も引き続き対応可能です。ご希望の項目がございましたらお知らせください！
