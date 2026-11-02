---
layout: default
title: 3.1-Device-hardening
nav_order: 1
parent: 3.0-Security-Infrastructure
---

# 3.1 Device hardening techniques and control plane protection methods

Cisco ネットワークセキュリティインフラストラクチャにおける **Device Hardening（デバイス構成強化）** および **Control Plane Protection（コントロールプレーン保護）** は、ルーター、スイッチ、ファイアウォール（ASA / FTD）、およびセキュリティ管理アプライアンス（FMC, ISE, WSA, ESA, SMA）自体の CPU、メモリ、管理インターフェイス、およびルーティング/制御プロトコル処理エンジンを、悪意ある DoS/DDoS 攻撃、偵察スキャン、未認証アクセス、およびリソース枯渇から防御するための必須セキュリティ技術群です。

CCIE Security v6.1 Blueprint **3.1 Device hardening techniques and control plane protection methods** では、**Cisco IOS / IOS-XE** 上での **Control Plane Policing (CoPP)**、**Control Plane Protection (CPPr: Host / Transit / CEF-Exception サブインターフェイス)**、**Infrastructure ACL (iACL)**、**Unicast Reverse Path Forwarding (uRPF: Strict / Loose / VRF-Aware)**、ならびに **Cisco ASA / FTD** 上での **Control-Plane ACL (`access-group control-plane`)**、**Threat Detection (Basic / Advanced / Scanning / Shun)**、管理プレーンセキュリティ（SSHv2, HTTPS, SNMPv3, TACACS+/RADIUS AAA, Password Encryption Type 6/8/9, NTP Authentication, Banner）の実装・運用・高度なトラブルシューティング能力が深く評価されます。

---

## 📘 概要

* **機能概要**:
  * **Control Plane Protection (CoPP / CPPr)**:
    * **CoPP (Control Plane Policing)**: ルーターのコントロールプレーンに到達するトラフィック（管理通信、ルーティングプロトコル、CEF例外パケット）を `class-map` で分類し、`policy-map` (QoS Police) を適用して CPU へのパケット投入レートを制限（Rate Limit）または即時破棄する機能。
    * **CPPr (Control Plane Protection)**: コントロールプレーンをさらに **Host サブインターフェイス**（自機宛て管理/インバンド通信: SSH/BGP/OSPF等）、**Transit サブインターフェイス**（自機を経由するデータ: GRE/IPsecトンネル終端パケット等）、および **CEF-Exception サブインターフェイス**（ARP, TTL過多, IP Options, ICMP unreachable等）の3つの集約パスに細分化し、各サブインターフェイスごとに独立した Policing ポリシーをバインドして精密制御する技術。
  * **Infrastructure ACL (iACL) ＆ uRPF**:
    * **iACL**: ネットワークインフラ機器（ルーター/スイッチの Loopback や物理インターフェイスのアドレス範囲）を外部および内部の未認証トラフィックから隔離・防護するため、ネットワークエッジに配置される精細なアクセスリスト。
    * **uRPF (Unicast Reverse Path Forwarding)**: パケットの送信元 IP アドレスが FIB (Forwarding Information Base) テーブルの戻りルートと一致するかをインターフェイス単位で検証し、IP スプーフィング（なりすまし）攻撃パケットをラインレートで未然破棄するセキュリティ機構。
  * **ASA / FTD Control Plane ＆ Threat Detection**:
    * **ASA Control-Plane ACL**: ASA 自身の IP (Management / Data interface) に宛てられた管理・制御パケットを、通常のデータプレーンアクセスルールを評価する前に即座にフィルタリングする専用 ACL 参照機能 (`access-group <ACL> in interface <IF> control-plane`)。
    * **Threat Detection**: DoS/DDoS 攻撃、ポートスキャン、ホストスキャン、不正なフラグ設定パケットを動的検知し、自動的に Syslog アラート出力または攻撃者 IP を一時遮断 (`shun`) する機能。
  * **Management Plane Hardening**:
    * 管理プロトコルの暗号化（SSHv2, HTTPS/TLS 1.2+）、SNMPv3 (AuthNoPriv / AuthPriv), AAA 認証・認可 (TACACS+ / RADIUS)、パスワードハッシュ強化 (Type 6 AES, Type 8 PBKDF2, Type 9 scrypt)、NTP 認証 ＆ access-group 制限。
* **利用目的**:
  * 大規模 DoS/DDoS 攻撃や不正な高負荷トラフィック（SYN Flood, ICMP Flood, ARP Storm）発生時における、ルーター/ファイアウォール CPU の枯渇・ハングアップ・ルーティングピア（BGP/OSPF）切断の防止。
  * インフラ機器自身の管理インターフェイス（SSH, SNMP, HTTPS）に対する非認可アクセスやパスワードクラック攻撃の排除。
  * ネットワーク境界における IP アドレススプーフィングパケットの遮断。
* **どのような場面で利用するか**:
  * エンタープライズコアルーター、ISP ピアリングエッジ、データセンター境界ルーター、ファイアウォール（ASA/FTD）の初期構築時およびセキュリティ強化（Hardening）要件の適用時。

---

## 🔑 要点

| 項目 | Cisco IOS / IOS-XE CoPP / CPPr | Cisco ASA Control-Plane ACL / Threat Detection |
| :--- | :--- | :--- |
| **保護対象** | ルーターの Route Processor (RP) / CPU 制御プレーン | ASA / FTD ノード自身の管理・制御プレーン (LINA Engine) |
| **制御単位** | **CoPP**: コントロールプレーン全体<br>**CPPr**: Host / Transit / CEF-Exception サブインターフェイス | **Control-Plane ACL**: インターフェイス宛て直接パケット<br>**Threat Detection**: 基本/拡張/スキャン脅威 |
| **トラフィック識別** | Modular QoS CLI (MQC: `class-map`, `policy-map`) | ACL (`access-list control-plane`) ＋ `threat-detection` コマンド |
| **アクション** | `police` (Rate Limiting) / `drop` / `transmit` | `deny` (Drop) / `permit` / 動的 `shun` (自動一時遮断) |
| **uRPF モード** | **Strict** (一致インターフェイス必須)<br>**Loose** (FIB 存在必須)<br>**VRF-Aware** | ASA インターフェイスレベル `ip verify reverse-path` (Strict 相当) |
| **管理プレーン保護** | SSHv2, HTTPS, SNMPv3, TACACS+, Password Type 6/8/9 | SSH, HTTPS (ASDM/FMC), SNMPv3, TACACS+/RADIUS |

---

## 🏗 動作原理

### 1. Control Plane Protection (CPPr) のサブインターフェイス アーキテクチャ

ルーターに着信したパケットは、宛先および処理タイプに応じて 3 つの内部 Control Plane サブインターフェイスへ自動的に振分けられます。

```text
[ Ingress Packet 着信 ]
       │
       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Cisco Router Data Plane (CEF Forwarding Engine)                                  │
│                                                                                  │
│ 1. パケットの宛先・処理要件の判定                                                │
│                                                                                  │
│ ├── (A) 自機 IP 宛ての制御・管理パケット (SSH, SNMP, BGP, OSPF, LDP)              │
│ │     └───► [ Control-Plane Host Subinterface ] ──► (Host CoPP Policy)          │
│ │                                                                                │
│ ├── (B) 自機を終端とするトンネル data または通過制御パケット (GRE/IPsecトンネル) │
│ │     └───► [ Control-Plane Transit Subinterface ] ──► (Transit CoPP Policy)    │
│ │                                                                                │
│ └── (C) CEF で処理できず CPU 介入が必要なパケット (ARP, TTL=1, IP Options, ICMP)   │
│       └───► [ Control-Plane CEF-Exception Subinterface ] ──► (CEF-Except Policy)│
└──────────────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Route Processor (RP) / CPU (Control Plane)                                       │
│   - BGP/OSPF ルーティングプロセス                                                 │
│   - SSH/SNMP 管理デーモン                                                         │
│   - ARP / ICMP パケット処理エンジン                                              │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

### 2. Unicast Reverse Path Forwarding (uRPF) Strict vs Loose モード動作比較

```text
[ Strict Mode uRPF ]
  Ingress Packet (Src: 10.1.1.5, Dst: 192.168.1.100) 着信 ➔ Interface Gi0/0
       │
       ▼
  FIB テーブル検索: "10.1.1.0/24 宛ての最適出力ポートは Gi0/0 か？"
       ├── YES ➔ 【PASS】 パケット転送を継続
       └── NO  ➔ 【DROP】 スプーフィングと判定し破棄 (uRPF Drop カウンタ増分)

[ Loose Mode uRPF ]
  Ingress Packet (Src: 10.1.1.5, Dst: 192.168.1.100) 着信 ➔ Interface Gi0/0
       │
       ▼
  FIB テーブル検索: "10.1.1.5 宛てのルートが FIB 内に存在する（ポート不問）か？"
       ├── YES ➔ 【PASS】 パケット転送を継続
       └── NO  ➔ 【DROP】 存在しない宛先/非ルーティング IP として破棄
```

---

## ⚙ 動作シーケンス

1. **パケット着信と uRPF 評価フェーズ**:
   * パケットが物理インターフェイスに着信。uRPF (`ip verify unicast source reachable-via`) が設定されている場合、送信元 IP を FIB と照合。設定モード (Strict / Loose) の基準を満たさないパケットは、ACL や QoS を評価する前に即座にラインレートで破棄。
2. **ASA Control-Plane ACL 評価フェーズ**:
   * ASA の物理/論理インターフェイスにパケットが着信。宛先が ASA 自身の IP アドレスである場合、インターフェイスにバインドされた **Control-Plane ACL** (`access-group <ACL> in interface <IF> control-plane`) が通常のデータプレーン ACL より**優先**して判定される。
   * Control-Plane ACL で `deny` された管理パケット（例: 非許可送信元からの SSH/SNMP）は即座にドロップされ、接続要求は拒否される。
3. **IOS CoPP / CPPr 分類・Policing 評価フェーズ**:
   * パケットがルーターの Route Processor (RP) へ転送される直前、バインドされた CoPP / CPPr ポリシーマップが発動。
   * パケットが `class-map` 条件（例: BGP, OSPF, ICMP, SSH）に一致するか評価。
   * `police` コマンドで定義された CIR (Committed Information Rate) や Burst サイズを超過したパケットは、設定されたアクション（`drop`）に従って破棄され、CPU への過負荷を防止する。
4. **Threat Detection ＆ Shun 自動遮断フェーズ (ASA/FTD)**:
   * ASA の Threat Detection エンジンがリアルタイムで接続率、失敗率、およびポートスキャンパターンを監視。
   * 閾値を超えた攻撃バーストを検知した場合、システムログを出力し、Scanning Threat Detection が有効な場合は該当攻撃者 IP に対する `shun` (動的ドロップルール) を内部生成して後続パケットを自動遮断する。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **IOS / IOS-XE CoPP / CPPr のバインドと構造要件**:
   * **CoPP**: `control-plane` グローバル設定モード配下で `service-policy input <POLICY_NAME>` を適用。
   * **CPPr**: `control-plane host`、`control-plane transit`、`control-plane cef-exception` モード配下でそれぞれ個別に `service-policy input <POLICY_NAME>` を適用。
   * **注意**: CPPr では、特定のサブインターフェイス（例: Host）のみに適用可能なクラス（管理プロトコル等）と、他のサブインターフェイス（例: CEF-Exception）にのみ適用可能なクラス（ARP等）が存在します。誤ったサブインターフェイスにクラスをアサインするとポリシー評価が意図通り動作しません。
2. **iACL (Infrastructure ACL) の設計と暗黙の Deny**:
   * **目的**: インフラ機器のアドレス空間（ルーターのループバックおよび P2P リンク IP）への直接アクセスを遮断しつつ、通過（Transit）トラフィックを全許可する。
   * **構文順序**:
     1. 許可された管理通信（SSH, SNMP, NTP, TACACS+）をインフラ IP 宛てに許可。
     2. 許可された制御プロトコル（BGP, OSPF, EIGRP, LDP）をインフラ IP 宛てに許可。
     3. **インフラ IP 宛てのその他全通信を明示拒否** (`deny ip any <INFRA_SUBNET> <MASK> log-input`)。
     4. **通過トラフィック（データプレーン）を全許可** (`permit ip any any`)。
3. **ASA Control-Plane ACL vs Normal Access Group**:
   * 通常の `access-group <ACL> in interface <IF>` は、ASA を通過するデータパケットを制御します。
   * ASA 自身への管理アクセス（SSH, ASDM/HTTPS, SNMP）をインターフェイス単位で制限するには、**`access-group <ACL> in interface <IF> control-plane`** コマンドの指定が必須です（※普通の `access-group` では ASA 宛ての管理パケットの制御順序が異なります）。
4. **uRPF (Unicast Reverse Path Forwarding) の各種オプション要件**:
   * **`allow-default` オプション**: デフォルトルート (`0.0.0.0/0`) を正規の戻りルートとして認めるか否か。インターネットエッジで `allow-default` を付けると、全パケットがマッチしてしまい uRPF の効果が失われるため注意が必要。
   * **`allow-self-ping` オプション**: ルーター自機の IP 宛ての Self-Ping を許可する。
5. **パスワード暗号化高度化 (Type 6, Type 8, Type 9)**:
   * `service password-encryption` (Type 7 - 脆弱) はラボ試験では不十分とされる場合があります。
   * **Type 6 (AES)**: `key config-key password-encrypt <KEY>` ＋ `password encryption aes`。
   * **Type 8 (PBKDF2 SHA-256)** / **Type 9 (scrypt)**: `username <NAME> algorithm-type sha256|scrypt secret <PASSWORD>`。

---


## 🛠 設定方法

### 1. Cisco IOS / IOS-XE: CoPP (Control Plane Policing) 設定

```bash
! ===== Step 1: クラスマップによる制御プレーン・トラフィックの分類 =====
access-list 120 permit tcp any any eq bgp
access-list 120 permit tcp any eq bgp any
access-list 121 permit ospf any any
access-list 122 permit icmp any any
access-list 123 permit tcp any any eq 22 ! SSH

class-map match-any CM_COPP_ROUTING
 match access-group 120
 match access-group 121

class-map match-all CM_COPP_MGMT
 match access-group 123

class-map match-all CM_COPP_ICMP
 match access-group 122

! ===== Step 2: ポリシーマップによる帯域制限 (Policing) の適用 =====
policy-map PM_COPP_POLICY
 class CM_COPP_ROUTING
  police 1000000 32000 conform-action transmit exceed-action transmit
 class CM_COPP_MGMT
  police 500000 16000 conform-action transmit exceed-action drop
 class CM_COPP_ICMP
  police 64000 8000 conform-action transmit exceed-action drop
 class class-default
  police 32000 4000 conform-action transmit exceed-action drop

! ===== Step 3: コントロールプレーンへポリシーバインド =====
control-plane
 service-policy input PM_COPP_POLICY
```

---

### 2. Cisco IOS / IOS-XE: CPPr (Control Plane Protection) 設定

```bash
! ===== Host, Transit, CEF-Exception 各サブインターフェイス用クラス定義 =====
class-map match-all CM_CPPR_HOST_SSH
 match access-group 130                    ! permit tcp any any eq 22

class-map match-all CM_CPPR_CEF_EXCEPT_ARP
 match protocol arp

! ===== ポリシーマップ作成 =====
policy-map PM_CPPR_HOST
 class CM_CPPR_HOST_SSH
  police 256000 8000 conform-action transmit exceed-action drop

policy-map PM_CPPR_CEF_EXCEPT
 class CM_CPPR_CEF_EXCEPT_ARP
  police 128000 4000 conform-action transmit exceed-action drop

! ===== 各サブインターフェイスへ適用 =====
control-plane host
 service-policy input PM_CPPR_HOST

control-plane cef-exception
 service-policy input PM_CPPR_CEF_EXCEPT
```

---

### 3. Cisco IOS / IOS-XE: Infrastructure ACL (iACL) ＆ uRPF 設定

```bash
! ===== Infrastructure ACL (iACL) 設定例 =====
! インフラストラクチャアドレス空間: 10.255.0.0/16 (ループバック ＆ 内部リンク)
ip access-list extended ACL_INFRASTRUCTURE_PROTECT
 ! 1. 管理通信の許可 (NOC 送信元 192.168.100.0/24 ➔ インフラ宛て)
 permit tcp 192.168.100.0 0.0.0.255 10.255.0.0 0.0.255.255 eq 22
 permit udp 192.168.100.0 0.0.0.255 10.255.0.0 0.0.255.255 eq snmp
 ! 2. 制御プロトコルの許可 (BGP ピア ➔ インフラ IP)
 permit tcp host 198.51.100.1 10.255.0.0 0.0.255.255 eq bgp
 permit tcp host 198.51.100.1 eq bgp 10.255.0.0 0.0.255.255
 permit ospf any 10.255.0.0 0.0.255.255
 ! 3. インフラ空間宛てのその他全通信の明示破棄 ＆ ログ出力
 deny ip any 10.255.0.0 0.0.255.255 log-input
 ! 4. 外部への通過データ (Transit Traffic) の全許可
 permit ip any any

! 外部エッジインターフェイスへのバインド
interface GigabitEthernet0/0
 ip access-group ACL_INFRASTRUCTURE_PROTECT in

! ===== uRPF (Strict Mode ＆ Loose Mode) 設定 =====
! Strict Mode (インターフェイス一致必須)
interface GigabitEthernet0/1
 ip verify unicast source reachable-via rx

! Loose Mode (FIB 存在必須 ＋ デフォルトルート無視)
interface GigabitEthernet0/2
 ip verify unicast source reachable-via any
```

---

### 4. Cisco ASA: Control-Plane ACL ＆ Threat Detection 設定

```bash
! ===== ASA Control-Plane ACL 設定 =====
! 管理専用端末 192.168.1.50 のみを ASA の outside IP 宛て SSH/SNMP 接続許可
access-list ACL_ASA_CONTROL_PLANE extended permit tcp host 192.168.1.50 host 203.0.113.1 eq ssh
access-list ACL_ASA_CONTROL_PLANE extended permit udp host 192.168.1.50 host 203.0.113.1 eq snmp
access-list ACL_ASA_CONTROL_PLANE extended deny ip any any

access-group ACL_ASA_CONTROL_PLANE in interface outside control-plane

! ===== ASA Threat Detection 設定 =====
threat-detection basic-threat                          ! 基本脅威検出の有効化
threat-detection statistics host                        ! ホスト統計の収集
threat-detection scanning-threat shun                   ! スキャン脅威検出時に攻撃者 IP を自動 shun
```

---

### 5. Cisco IOS / IOS-XE: Management Plane Security Hardening

```bash
! ===== SSHv2 ＆ AAA / Local Auth 設定 =====
ip domain-name corp.local
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3

! ===== 高度パスワード暗号化 (Type 9 scrypt / Type 8 SHA-256) =====
username admin algorithm-type scrypt secret Cisco12345!

! ===== VTY / Line Hardening =====
line vty 0 15
 exec-timeout 5 0
 transport input ssh
 access-class ACL_ADMIN_VTY in
 logging synchronous

! ===== NTP 認証 ＆ Access Group 制御 =====
ntp authentication-key 1 md5 CiscoNtpKey123
ntp trusted-key 1
ntp authenticate
ntp server 192.168.10.1 key 1

access-list 10 permit 192.168.10.1                     ! NTP サーバー IP
ip access-list extended ACL_NTP_SERVERS
 permit ip host 192.168.10.1 any

ntp access-group peer 10
```

---

## 🔍 検証コマンド

| 目的 | コマンド (IOS / ASA / FTD CLI) | 表示確認ポイント |
| :--- | :--- | :--- |
| **CoPP / CPPr 統計 ＆ ドロップ確認** | <code>show policy-map control-plane</code> | クラスごとの `conformed` / `exceeded` パケット数および `drop` カウンタ |
| **CPPr サブインターフェイス別統計確認** | <code>show policy-map control-plane host</code><br><code>show policy-map control-plane cef-exception</code> | Host / CEF-Exception それぞれにおける Policing ドロップ数 |
| **uRPF ドロップパケット統計確認** | <code>show ip traffic \| sec Drop</code><br><code>show interface GigabitEthernet0/0</code> | `unicast RPF drops` 件数、インターフェイス別の RPF ドロップ統計 |
| **ASA Control-Plane ACL ヒット数確認** | <code>show access-list ACL_ASA_CONTROL_PLANE</code> | `control-plane` ACL 内の各行における `hitcnt` 件数 |
| **ASA Threat Detection 検出・Shun 状態確認** | <code>show threat-detection statistics</code><br><code>show threat-detection scanning-threat</code><br><code>show shun</code> | ポートスキャン検知数、動的遮断 (`shun`) された攻撃者 IP 一覧 |
| **SSHv2 接続 ＆ 暗号セッション確認** | <code>show ip ssh</code><br><code>show ssh</code> | SSH バージョン (2.0)、接続中クライアント IP、使用 Cipher |
| **NTP 同期 ＆ 認証状態確認** | <code>show ntp associations</code><br><code>show ntp status</code> | `sys.peer` 状態 (`*`)、`authenticated` フラグの付与確認 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **CoPP 適用後、BGP ピアリングや OSPF ネイバーが定期的に Down (Flap) する** | CoPP のルーティングプロトコル用 `class-map` で定義した `police` レート（CIR / Burst）が小さすぎ、ルーター間 Keepalive / LSA アップデートがドロップしている | <code>show policy-map control-plane</code><br><code>show log</code> | 対象クラスの `police` レートを引き上げるか、`conform-action transmit exceed-action transmit` に一時変更して検証する。 |
| **uRPF 適用後、正常な非対称ルーティング通信（Asymmetric Routing）が破棄される** | Strict モード uRPF を設定しているため、戻りポートと異なるポートからの着信パケットがドロップされた | <code>show ip traffic</code><br><code>show ip route <SRC_IP></code> | 非対称パスが存在するインターフェイスでは uRPF を **Loose モード** (`reachable-via any`) に変更する。 |
| **ASA に `access-group control-plane` をバインドしたら管理者からの ASDM / SSH アクセスが遮断された** | Control-Plane ACL の末尾（暗黙の Deny）により、管理端末の IP アドレスがブロックされた | <code>show access-list</code><br><code>show log</code> | Control-Plane ACL の最上部に管理端末 IP から ASA IP 宛ての `permit tcp host <ADMIN_IP> host <ASA_IP> eq 22/443` を追記する。 |
| **CPPr で CEF-Exception クラスのパケットがドロップされず CPU 高負荷が続く** | ARP や ICMP 等の CEF 例外パケットを `control-plane host` に適用しようとしている | <code>show policy-map control-plane</code> | ARP などの CEF 例外パケットに対する CPPr ポリシーは必ず `control-plane cef-exception` サブインターフェイス配下にバインドする。 |
| **SSHv2 接続時 `Algorithm negotiation failed` エラーが発生してログインできない** | SSH クライアント側が要求する暗号化アルゴリズム/Cipher が、ルーター側で非有効化されている | <code>show ip ssh</code> | ルーター側で `ip ssh algorithm encryption` または `ip ssh server algorithm mac/kex` を更新し適合させる。 |

---

## ⚠ 制限事項

* **CPPr のプラットフォーム依存性**:
  * CPPr (Control Plane Protection - サブインターフェイス分割) は、一部の Cisco IOS / IOS-XE プラットフォームおよび分散型アーキテクチャでのみサポートされます。機能非サポート環境では一般的な **CoPP** を使用する必要があります。
* **ASA Control-Plane ACL の適用対象制限**:
  * ASA の `access-group control-plane` は ASA 自体を宛先とするパケット（管理アクセス、VPN 制御等）のみに作用し、ASA を「通過」するデータプレーンパケットには影響を及ぼしません。
* **uRPF と Default Route (`0.0.0.0/0`) の関係**:
  * `allow-default` オプションなしの Strict / Loose モード uRPF では、FIB テーブル上でデフォルトルートのみにマッチする送信元 IP はドロップ対象となる場合があります。

---

## 🔄 他技術との関連

* **Cisco ISE / TACACS+ (Device Administration)**: ルーター/ASA への CLI アクセスに対する中央集中型 AAA 認証およびコマンド認可 (Command Authorization)。
* **QoS (MQC - Modular QoS CLI)**: CoPP / CPPr のパケット分類 (`class-map`) およびPolicing (`policy-map`) の基盤となる機構。
* **CEF (Cisco Express Forwarding)**: uRPF の高速テーブル照合および CEF-Exception パケット分岐の前提となる転送エンジン。

---

## 🧩 比較表

### 1. CoPP vs CPPr (Host / Transit / CEF-Exception)

| 比較項目 | CoPP (Control Plane Policing) | CPPr (Control Plane Protection) |
| :--- | :--- | :--- |
| **制御構造** | コントロールプレーンを **単一の論理ポート** として扱う | コントロールプレーンを **3 つのサブインターフェイス** に分割 |
| **適用レベル** | `control-plane` 直下 | `control-plane host` / `transit` / `cef-exception` |
| **粒度** | 全制御パケットが一括で同一ポリシーを通過 | 管理・トンネル・CEF例外パケットを完全独立制御 |
| **トラブルシューティング**| どの種類のトラフィックがドロップしたか特定が難解 | 各サブインターフェイスごとの個別ドロップ統計により迅速特定 |

### 2. uRPF Strict Mode vs Loose Mode

| 比較項目 | uRPF Strict Mode | uRPF Loose Mode |
| :--- | :--- | :--- |
| **検証条件** | 送信元 IP の FIB 戻りルートが **着信インターフェイスと一致** すること | 送信元 IP の FIB ルートが **テーブル内に存在** すること（ポート不問） |
| **非対称パス対応** | **不可** (ドロップ発生) | **対応** (別ポートからの着信も許可) |
| **デフォルトルート** | `allow-default` 明示指定で許可 | `allow-default` 明示指定で許可 |
| **適用推奨箇所** | アクセスレイヤー、シングルホーム WAN エッジ | マルチホーム BGP ピアリングエッジ、ISP 境界 |

---

## 💡 ベストプラクティス

1. **CoPP 導入時の「Dry-Run」検証手順**:
   * 新しく CoPP ポリシーを導入する際は、いきなり `exceed-action drop` を設定せず、`exceed-action transmit` または `police` せずに `class-map` でヒット数のみを測定し、必要なルーティングプロトコルや管理通信が意図せずドロップしないかを事前検証します。
2. **iACL (Infrastructure ACL) でのログ出力設定**:
   * インフラ空間への不正アクセスを拒否する行には `log-input` オプションを付与し、攻撃の送信元 MAC アドレスおよび着信インターフェイス情報を Syslog へ記録・追跡できるようにします。
3. **ASA での Control-Plane ACL 設定時の管理アクセス保護**:
   * `access-group <ACL> in interface <IF> control-plane` を適用する前に、現在自身が接続している SSH/HTTPS セッションの IP が `permit` 行に含まれていることを必ずダブルチェックします（※誤ると即座にセッションが遮断されます）。


## 📝 ラボ学習・設定サンプル例

### 例1: IOS CoPP による BGP/OSPF 優先処理 ＆ ICMP Rate-Limiting

#### 問題
R1 (Cisco IOS) において、コントロールプレーンを攻撃から保護するため CoPP を構成せよ。BGP (TCP 179) および OSPF トラフィックには 1 Mbps の帯域を確保し、ICMP トラフィックは 64 kbps に制限して超過パケットを破棄せよ。その他の未定義トラフィックは 32 kbps に制限せよ。

#### 要件
* クラスマップ名: `CM_ROUTING`, `CM_ICMP`
* ポリシーマップ名: `PM_COPP`
* `control-plane` 配下に適用すること。

#### 設定例

```bash
! ===== Step 1: トラフィック分類 ACL ＆ クラスマップ =====
ip access-list extended ACL_ROUTING_PROTO
 permit tcp any any eq bgp
 permit tcp any eq bgp any
 permit ospf any any

ip access-list extended ACL_ICMP_TRAFFIC
 permit icmp any any

class-map match-all CM_ROUTING
 match access-group name ACL_ROUTING_PROTO

class-map match-all CM_ICMP
 match access-group name ACL_ICMP_TRAFFIC

! ===== Step 2: ポリシーマップ定義 =====
policy-map PM_COPP
 class CM_ROUTING
  police 1000000 32000 conform-action transmit exceed-action transmit
 class CM_ICMP
  police 64000 8000 conform-action transmit exceed-action drop
 class class-default
  police 32000 4000 conform-action transmit exceed-action drop

! ===== Step 3: コントロールプレーンへ適用 =====
control-plane
 service-policy input PM_COPP
```

#### 検証方法
* `show policy-map control-plane` を実行し、`CM_ROUTING` および `CM_ICMP` クラスでパケットが分類・Policing 処理されていることを確認する。
* 送信元から大容量 Ping (`ping <R1_IP> repeat 1000 size 1400`) を実行し、`CM_ICMP` の `exceeded` カウンタが増加しドロップされることを確認する。

---

### 例2: IOS CPPr による Host (SSH/SNMP) ＆ CEF-Exception (ARP) 分割制御

#### 問題
R1 において CPPr (Control Plane Protection) を構築せよ。自機宛ての SSH (TCP 22) および SNMP (UDP 161) を `host` サブインターフェイスで 256 kbps に制限し、ARP パケットを `cef-exception` サブインターフェイスで 128 kbps に制限せよ。

#### 要件
* `control-plane host` および `control-plane cef-exception` を使用すること。

#### 設定例

```bash
! ===== Step 1: クラスマップ定義 =====
ip access-list extended ACL_MGMT_HOST
 permit tcp any any eq 22
 permit udp any any eq 161

class-map match-all CM_CPPR_HOST
 match access-group name ACL_MGMT_HOST

class-map match-all CM_CPPR_ARP
 match protocol arp

! ===== Step 2: ポリシーマップ定義 =====
policy-map PM_CPPR_HOST_POL
 class CM_CPPR_HOST
  police 256000 8000 conform-action transmit exceed-action drop

policy-map PM_CPPR_CEF_EXCEPT_POL
 class CM_CPPR_ARP
  police 128000 4000 conform-action transmit exceed-action drop

! ===== Step 3: サブインターフェイスへの適用 =====
control-plane host
 service-policy input PM_CPPR_HOST_POL

control-plane cef-exception
 service-policy input PM_CPPR_CEF_EXCEPT_POL
```

#### 検証方法
* `show policy-map control-plane host` および `show policy-map control-plane cef-exception` を実行し、各サブインターフェイスにそれぞれのポリシーが適用されていることを確認する。

---

### 例3: Infrastructure ACL (iACL) によるコアインフラ空間の防護

#### 問題
ルーター R1 の外部境界インターフェイス `Gi0/0` において、内部インフラ空間 (`10.255.0.0/16`) を保護する iACL `ACL_IACL_PROTECT` を適用せよ。NOC 管理端末 (`192.168.100.50`) からの SSH/SNMP アクセス、および BGP ピア (`198.51.100.1`) を除き、インフラ空間宛ての全パケットを拒否してログ出力せよ。通過データ通信は全て許可せよ。

#### 要件
* `log-input` オプションを使用すること。

#### 設定例

```bash
! ===== iACL 定義 =====
ip access-list extended ACL_IACL_PROTECT
 ! 1. 許可された管理端末からの通信
 permit tcp host 192.168.100.50 10.255.0.0 0.0.255.255 eq 22
 permit udp host 192.168.100.50 10.255.0.0 0.0.255.255 eq snmp
 ! 2. BGP 制御プロトコル
 permit tcp host 198.51.100.1 10.255.0.0 0.0.255.255 eq bgp
 permit tcp host 198.51.100.1 eq bgp 10.255.0.0 0.0.255.255
 ! 3. インフラ宛てその他通信の拒否 ＆ ログ
 deny ip any 10.255.0.0 0.0.255.255 log-input
 ! 4. 通過トラフィックの許可
 permit ip any any

! ===== インターフェイス適用 =====
interface GigabitEthernet0/0
 ip access-group ACL_IACL_PROTECT in
```

#### 検証方法
* 非許可ホストから `10.255.1.1` へ Ping / SSH を実行し、通信がブロックされ Syslog に `denied` ログと着信 MAC/インターフェイスが出力されることを確認する。

---

### 例4: uRPF Strict Mode と Loose Mode のインターフェイス別適用

#### 問題
ルーター R1 において、単一接続のアクセスインターフェイス `Gi0/1` に Strict モード uRPF を設定し、マルチホーム BGP ピアリングインターフェイス `Gi0/2` に Loose モード uRPF を設定せよ。デフォルトルートは RPF チェックの対象外とせよ。

#### 要件
* `ip verify unicast source` コマンドを使用すること。

#### 設定例

```bash
! ===== Access Interface (Strict Mode) =====
interface GigabitEthernet0/1
 ip verify unicast source reachable-via rx

! ===== Multihomed BGP Interface (Loose Mode) =====
interface GigabitEthernet0/2
 ip verify unicast source reachable-via any
```

#### 検証方法
* `show ip traffic` を実行し、`unicast RPF drops` カウンタが表示されることを確認する。
* `Gi0/1` に自ルートテーブルに存在しない、あるいは別ポートから戻る送信元 IP のパケットを入力し、破棄されることを確認する。

---

### 例5: ASA Control-Plane ACL による SSH/ASDM 管理アクセスの制限

#### 問題
Cisco ASA1 において、outside インターフェイス宛ての直接アクセス（SSH/ASDM/SNMP）を制御する Control-Plane ACL `ACL_ASA_CP` を作成せよ。管理者 IP `192.168.1.100` からの SSH (22) および HTTPS (443) のみを許可し、その他の ASA 自身宛て通信を拒否せよ。

#### 要件
* `access-group control-plane` コマンドを使用すること。

#### 設定例

```bash
! ===== ASA Control-Plane ACL 定義 =====
access-list ACL_ASA_CP extended permit tcp host 192.168.1.100 any eq ssh
access-list ACL_ASA_CP extended permit tcp host 192.168.1.100 any eq https
access-list ACL_ASA_CP extended deny ip any any

! ===== Control-Plane 適用 =====
access-group ACL_ASA_CP in interface outside control-plane
```

#### 検証方法
* `192.168.1.100` から ASA outside IP 宛てに SSH 接続を実施し成功することを確認。
* 別の IP (`192.168.1.200`) から SSH 接続を実施し、拒否され `show access-list ACL_ASA_CP` の `deny` 行の `hitcnt` が増加することを確認する。

---

### 例6: ASA Threat Detection による Scanning Threat 検知と動的 Shun

#### 問題
ASA1 において、Basic Threat Detection を有効化し、ポートスキャンやホストスキャン攻撃者を動的検知した場合に、攻撃者 IP を自動的に `shun` (一時遮断) するよう設定せよ。

#### 要件
* `threat-detection scanning-threat shun` コマンドを使用すること。

#### 設定例

```bash
! ===== ASA Threat Detection 設定 =====
threat-detection basic-threat
threat-detection statistics host
threat-detection scanning-threat shun
```

#### 検証方法
* `show threat-detection scanning-threat` を実行し、スキャン検知エンジンがアクティブであることを確認する。
* 外部からポートスキャンを発行し、検知後に `show shun` で攻撃者 IP が動的ブロックリストに追加されていることを確認する。

---

### 例7: IOS-XE Password Encryption (Type 9 scrypt / Type 8 SHA-256) ＆ SSHv2 構成

#### 問題
R1 (Cisco IOS-XE) において、ユーザー `admin` に Type 9 (scrypt) 暗号化パスワード `Cisco12345!` を設定し、ユーザー `operator` に Type 8 (SHA-256) パスワード `Operator123!` を設定せよ。また SSHv2 を強制有効化せよ。

#### 要件
* `algorithm-type scrypt` および `algorithm-type sha256` を使用すること。

#### 設定例

```bash
! ===== ドメイン名 ＆ 鍵生成 =====
ip domain-name corp.local
crypto key generate rsa modulus 2048

! ===== ユーザー定義 =====
username admin algorithm-type scrypt secret Cisco12345!
username operator algorithm-type sha256 secret Operator123!

! ===== SSHv2 制限 =====
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3

line vty 0 4
 login local
 transport input ssh
```

#### 検証方法
* `show running-config | inc username` を実行し、`username admin secret 9 $9$...` および `username operator secret 8 $8$...` と表示されていることを確認する。

---

### 例8: IOS-XE NTP 認証 ＆ NTP Access-Group による制御

#### 問題
R1 において、NTP サーバー `192.168.10.1` との同期を構成せよ。NTP 認証キー `1` (MD5: `NtpSecretKey123`) を使用し、NTP 応答を受信する相手を ACL `10` で `192.168.10.1` に限定せよ。

#### 要件
* `ntp authentication-key`, `ntp trusted-key`, `ntp access-group peer` を使用すること。

#### 設定例

```bash
! ===== NTP 認証設定 =====
ntp authenticate
ntp authentication-key 1 md5 NtpSecretKey123
ntp trusted-key 1
ntp server 192.168.10.1 key 1

! ===== NTP アクセス制御 =====
access-list 10 permit host 192.168.10.1
ntp access-group peer 10
```

#### 検証方法
* `show ntp status` および `show ntp associations` を実行し、`192.168.10.1` と正常に同期し、`authenticated` 状態になっていることを確認する。

---

### 例9: FTD/FMC Platform Settings による SSH/HTTPS 管理アクセス制限

#### 問題
FMC 管理下の FTD において、管理インターフェイスへの SSH (TCP 22) および HTTPS (TCP 443) アクセスを、社内 NOC サブネット `10.100.1.0/24` のみに制限する Platform Settings を構成せよ。

#### 要件
* FMC の Platform Settings > Secure Shell / HTTP 構成を使用すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Devices > Platform Settings に移動し、新規 FTD Policy を作成。
2. "Secure Shell (SSH)" タブを選択:
   - Add IP / Network: 10.100.1.0/24
   - Selected Interface: Management
3. "HTTP" タブを選択:
   - Add IP / Network: 10.100.1.0/24
   - Selected Interface: Management
4. Save して対象 FTD アプライアンスへ Deploy。
```

#### 検証方法
* FTD Diagnostic CLI で `show running-config http` および `show running-config ssh` を実行し、`http 10.100.1.0 255.255.255.0 management` が反映されていることを確認する。

---

### 例10: IOS Line VTY Hardening (Exec-Timeout, Transport Input SSH, Access-Class)

#### 問題
R1 の VTY ポート (0–15) において、タイムアウトを 5 分 0 秒に制限し、接続プロトコルを SSH のみに制限せよ。また管理端末セグメント `192.168.50.0/24` からの接続のみを許可する ACL `ACL_VTY_ACCESS` をバインドせよ。

#### 要件
* `exec-timeout 5 0`, `transport input ssh`, `access-class` を使用すること。

#### 設定例

```bash
! ===== VTY 制限 ACL =====
ip access-list extended ACL_VTY_ACCESS
 permit tcp 192.168.50.0 0.0.0.255 any eq 22
 deny ip any any log-input

! ===== Line VTY 設定 =====
line vty 0 15
 exec-timeout 5 0
 transport input ssh
 access-class ACL_VTY_ACCESS in
 logging synchronous
 login local
```

#### 検証方法
* `192.168.50.10` から SSH 接続し成功することを確認。非許可端末から Telnet および SSH を試み、即座に拒否されることを確認する。


## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 以下の CoPP コンフィグが適用されたルーター R1 において、192.168.1.50 から R1 のループバック IP 宛てに発生させた大容量 ICMP 通信が大量にドロップされたが、BGP ピアリングは正常に維持されている。ドロップが発生している理由として最も適切なものはどれか。

```bash
access-list 101 permit tcp any any eq bgp
access-list 102 permit icmp any any

class-map match-all CM_BGP
 match access-group 101

class-map match-all CM_ICMP
 match access-group 102

policy-map PM_COPP
 class CM_BGP
  police 1000000 32000 conform-action transmit exceed-action transmit
 class CM_ICMP
  police 32000 4000 conform-action transmit exceed-action drop

control-plane
 service-policy input PM_COPP
```

* A) BGP パケットの ACL 照合が失敗しているため。
* B) `CM_ICMP` クラスの Policing レートが 32 kbps (CIR) / 4000 bytes (Burst) に制限されており、超過した ICMP パケットが `exceed-action drop` により破棄されているため。
* C) `control-plane` にポリシーマップを適用すると全 ICMP が自動遮断されるため。
* D) `class-default` に明示的な `drop` アクションが記載されていないため。

**正解**: **B**  
**解説**: `PM_COPP` ポリシーマップ内で、`CM_ICMP` クラスに対し `police 32000 4000 conform-action transmit exceed-action drop` が定義されています。そのため、32 kbps を超える ICMP トラフィックは正しく Policing 判定を受け、超過分がドロップされています。BGP は 1 Mbps の高帯域が割り当てられているため正常に動作し続けています。

---

### 2. トラブルシュート問題

**問題**: ASA1 の管理者から「outside インターフェイス宛ての SSH アクセスを制御するために `access-group ACL_ASA_MGMT in interface outside control-plane` をバインドした直後、既存の SSH セッションが遮断され、管理者 IP `192.168.10.50` からの再接続も不可となった」との報告があった。`ACL_ASA_MGMT` の定義は以下の通りである。修復方法として正しいものはどれか。

```bash
access-list ACL_ASA_MGMT extended permit tcp host 192.168.10.50 any eq 22
```

* A) `access-group` コマンドで `control-plane` オプションを外し、普通の `access-group` に変更する。
* B) Control-Plane ACL では暗黙の Deny (`deny ip any any`) が末尾に存在し、さらに SSH の宛先 port 22 のみならず ASA 自体への応答を考慮して ACL 定義を再確認し、必要に応じて管理アクセス用 IP 指定を正しく見直す。
* C) `threat-detection basic-threat` を無効化する。
* D) `service-policy` を `inside` インターフェイスに移動する。

**正解**: **B**  
**解説**: Control-Plane ACL も通常の ACL と同様に末尾に「暗黙の Deny (deny ip any any)」が存在します。`access-list` の記述で送信元 IP、宛先 IP (ASA 自身の IP)、プロトコル (TCP 22) が正しく許可されているか、また管理端末の NAT 変換後の IP などと不整合がないか確認・修復する必要があります。

---

### 3. Design 問題

**問題**: エンタープライズコアルーターにおいて、制御プレーン（Route Processor）に引き渡されるパケットのうち、自機宛ての SSH/BGP 管理制御通信と、CEF 処理で自己完結できずに CPU 介入が発生する ARP パケットを、それぞれ独立した Policing ポリシーで分離・防護したい。採用すべき最も適切な技術はどれか。

* A) 通常の Extended Access-List をデータポートに適用する。
* B) **Control Plane Protection (CPPr)** を使用し、`control-plane host` に SSH/BGP 用ポリシー、`control-plane cef-exception` に ARP 用ポリシーを独立バインドする。
* C) インフラストラクチャ ACL (iACL) のみを適用する。
* D) Unicast Reverse Path Forwarding (uRPF) Strict モードを適用する。

**正解**: **B**  
**解説**: コントロールプレーン宛てトラフィックを自機宛て管理通信 (`host`) と CEF 例外処理通信 (`cef-exception`) に分割して個別に Policing を適用できるのは **CPPr (Control Plane Protection)** の独自機能です。

---

### 4. 実装問題

**問題**: ルーター R1 のインターフェイス Gi0/0 において uRPF Strict モード (`ip verify unicast source reachable-via rx`) を有効化したところ、一部の外部顧客からの非対称ルーティング（戻りトラフィックが Gi0/1 経由で着信）トラフィックが全ドロップされた。通信を正常化しつつ IP スプーフィング攻撃を防止するための最適設定はどれか。

* A) uRPF を完全に無効化する。
* B) Gi0/0 の uRPF を **Loose モード** (`ip verify unicast source reachable-via any`) に変更する。
* C) `allow-default` オプションを追加して Strict モードのまま維持する。
* D) `ip proxy-arp` を有効化する。

**正解**: **B**  
**解説**: 非対称パスが存在する場合、Strict モード（着信ポートと FIB 戻りポートの一致要求）ではパケットが破棄されます。**Loose モード** (`reachable-via any`) に変更することで、送信元 IP が FIB テーブル内に存在すれば着信ポートに関わらず通過を許可できるため、非対称パス環境における防御策として最も適切です。

---

### 5. コンフィグ読解問題

**問題**: 以下の iACL (Infrastructure ACL) が外部境界インターフェイスのインバウンドに適用されている。内部インフラ空間 (`10.255.0.0/16`) 宛てではない一般インターネット通過トラフィック（データプレーン）の処理結果として正しいものはどれか。

```bash
ip access-list extended ACL_INFRA
 permit tcp host 192.168.1.1 10.255.0.0 0.0.255.255 eq 22
 deny ip any 10.255.0.0 0.0.255.255 log-input
 permit ip any any
```

* A) 全パケットが `deny` 行にマッチしてドロップされる。
* B) 宛先が `10.255.0.0/16` 以外の一般通過パケットは、`deny` 行を通過して 3 行目の `permit ip any any` にマッチし、正常に転送される。
* C) `log-input` オプションの影響でデータパケットが低速化する。
* D) SSH パケット以外のすべての通過パケットが拒否される。

**正解**: **B**  
**解説**: iACL の標準設計において、`deny ip any <INFRA_SUBNET>` 行は宛先がインフラストラクチャ空間 (`10.255.0.0/16`) であるパケットのみを捕捉します。宛先がインフラ空間以外の一般通過トラフィックは `deny` 行をスルーし、末尾の `permit ip any any` にマッチして正常に高速転送されます。

---

## 🔗 参考リソース

* [Cisco IOS-XE Security Configuration Guide: Control Plane Policing (CoPP)](https://www.cisco.com/)
* [Cisco IOS-XE Security Configuration Guide: Control Plane Protection (CPPr)](https://www.cisco.com/)
* [Cisco ASA Series General Operations Configuration Guide: Threat Detection and Control-Plane ACLs](https://www.cisco.com/)
* [Cisco Technical Notes: Infrastructure Protection ACLs (iACL) Design and Deployment Guide](https://www.cisco.com/)
* [Cisco Technical Notes: Understanding Unicast Reverse Path Forwarding (uRPF)](https://www.cisco.com/)
* [Cisco Live: Hardening Cisco IOS / IOS-XE Devices and Control Plane Architecture (BRKSEC-2001)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で Device Hardening / Control Plane が出題された場合の構成判断ルール：
    * **「Protect Router CPU using MQC for all control traffic」** ➔ CoPP (`control-plane` ➔ `service-policy input`).
    * **「Differentiate Host management from CEF-Exception traffic (ARP)」** ➔ CPPr (`control-plane host` ＆ `control-plane cef-exception`).
    * **「Block unauthorized SSH/SNMP direct to ASA IP」** ➔ ASA Control-Plane ACL (`access-group <ACL> in interface <IF> control-plane`).
    * **「Mitigate IP spoofing on asymmetric multi-homed links」** ➔ uRPF Loose Mode (`ip verify unicast source reachable-via any`).
    * **「Protect Infrastructure Loopback IPs at WAN edge」** ➔ iACL (`deny ip any <INFRA_SUBNET>` ＋ `permit ip any any`).
* **図解（CoPP / CPPr / ASA Control-Plane パイプライン）**:
  ```text
  [ Ingress Packet ]
          │
          ├── Dst: ASA Internal IP? ──► [ ASA Control-Plane ACL ] ──► Pass/Drop
          │
          └── Dst: Router CPU / RP?
                  │
                  ├── (CoPP Mode) ──► [ Global CoPP Policy ] ──► Police/Drop/Transmit
                  │
                  └── (CPPr Mode) ──► Classify Destination Subinterface
                                           ├── [ Host Subinterface ] (SSH, BGP, OSPF)
                                           ├── [ Transit Subinterface ] (GRE/IPsec Tunnel)
                                           └── [ CEF-Exception Subinterface ] (ARP, TTL=1)
  ```
* **注意点**:
  * ラボ試験で CoPP や iACL を設定した後は、必ず `show policy-map control-plane`、`show access-list`、`show ip traffic` を実行し、必要なルーティングプロトコルや管理通信が意図せずドロップされていないか確認・証明してください。


## 📝 **補足（Notes）**

*   **学習メモ**: CCIE Security において、ハードニングは「加点要素」というより「減点回避」の基本です。Telnet を残したままにするなどの凡ミスは避けましょう。
*   **注意点**: ラボ試験での CoPP 設定時は、必ず `permit` だけでなく、設定した Policing が実際に `show` コマンドでカウントアップされるかを、ルータ自身に `ping` を打つなどして確認してください。
