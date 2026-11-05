---
layout: default
title: 3.7.a-ISO27001
nav_order: 1
parent: 3.7-Compliance
grand_parent: 3.0-Security-Infrastructure
---

# 3.7.a ISO 27001

## 📘 概要

ISO/IEC 27001（最新版: ISO/IEC 27001:2022）は、組織における情報セキュリティマネジメントシステム（ISMS: Information Security Management System）の構築・運用・維持・継続的改善に関する国際標準規格です。ネットワークインフラストラクチャおよびセキュリティ設計において、ISO 27001規格が規定するセキュリティ管理策（Annex A / ISO 27002）の要求事項を技術的にどのように実装・運用・検証するかは、企業・組織のガバナンスおよび規制順守（Compliance）において不可欠な知識となります。

CCIE Security v6.1 試験における **3.7.a ISO 27001** の技術的範囲は、単なる組織マネジメントやポリシー文書の作成ではなく、Cisco Secure ネットワークインフラ機器（Cisco IOS-XE、Cisco ASA、Cisco Firepower / FTD / FMC、Cisco ISE、Cisco WSA / ESA 等）を用いて、ISO 27001 の付属書 A（Annex A）および ISO 27002 で求められるハードウェア・ソフトウェアベースのセキュリティコントロール（管理策）を具体的に実装・検証・トラブルシューティングする技術力を評価する領域です。

ISO 27001 のネットワークセキュリティに関連する主要管理策カテゴリーと、Cisco インフラストラクチャの対応技術は以下の通りです。

* **アクセス制御 (Access Control - ISO 27001:2022 8.2 / 旧 A.9)**: ネットワークアクセス制御（802.1X, MAB, Dynamic VLAN Assignment）、アイデンティティ管理（Cisco ISE, TACACS+, RADIUS）、ロールベースアクセス制御（RBAC, SGT / TrustSec）、特権アクセス管理（Privilege Level, Parser Option, Command Authorization）。
* **暗号化による保護 (Cryptography - ISO 27001:2022 8.24 / 旧 A.10)**: 転送中データの暗号化（IPsec VPN, FlexVPN, MACsec / 802.1AE, TLS/SSL）、次世代暗号アルゴリズム（CNSA / Suite B, AES-256-GCM, SHA-384, ECDSA）。
* **運用セキュリティと監視 (Operations Security & Monitoring - ISO 27001:2022 8.15, 8.16 / 旧 A.12, A.13)**: ログ収集・監視とタイムスタンプの完全性（Secure Syslog TLS 6514, NTP 認証 / PTP）、ネットワークトラフィック監視（NetFlow / IPFIX / NSEL, eStreamer, SNMPv3 USM/VACM）、脆弱性・構成管理（FMC Intrusion Prevention, Snort 3, Secure Endpoint / AMP）。
* **通信のセキュリティ (Communications Security - ISO 27001:2022 8.20, 8.21, 8.22 / 旧 A.13)**: ネットワークセグメンテーション（VLAN, PVLAN, VRF-Lite, SGT）、境界セキュリティ（iACL, CoPP, BCP 38 / uRPF, DAI, DHCP Snooping, RA Guard, VACL）、不正トラフィックの制御。
* **インシデント管理と自動対処 (Incident Management & Response - ISO 27001:2022 5.24-5.28 / 旧 A.16)**: リアルタイム脅威検知・相関分析（FMC Correlation Policy, Security Intelligence）、自動封じ込め（Rapid Threat Containment - RTC, ISE pxGrid ANC Dynamic Isolation）。

本学習メモは、CCIE Security ラボ試験においてISO 27001等の組織的セキュリティポリシー要件が出題された際に、要件を満たすCiscoインフラストラクチャ技術を的確に適用・検証できるレベルまで深く整理しています。

---

## 🔑 要点

ISO 27001 の主要管理策分野と、Cisco セキュリティインフラへの技術的マッピング、および設計・実装上のキーポイントを整理します。

| 管理策カテゴリー (ISO 27001) | Cisco 対応技術 / 機能 | 主な実装内容 | ラボ試験・設計上の注意点 |
| --- | --- | --- | --- |
| **アクセス制御 (Access Control)** | Cisco ISE, 802.1X, MAB, TACACS+, RADIUS | 端末/ユーザーの認証・認可・会計(AAA)。Cisco ISEによるコンテキストベース認可ポリシー。スイッチ/ルータのコマンド制限(Parser View/Privilege Level)。 | 認証失敗時のGuest/Auth-Fail VLAN設定。802.1Xタイムアウト値の最適化。ISEでのTACACS+ Command Authorizationの順序設定。 |
| **転送中データの暗号化 (Cryptography)** | MACsec (802.1AE), IPsec / FlexVPN, TLS/SSL, Suite B / CNSA | L2リンク間暗号化(MACsec MKA)、拠点間/リモートアクセス暗号化(FlexVPN/IPsec)、Web/管理通信のTLS 1.3保護。 | ネットワーク機器のハードウェア暗号エンジン(ASIC)制限。MACsec使用時のMTUオーバーヘッド(バッファ24/32バイト)とMSS調整。 |
| **ネットワークセグメンテーション (Network Segmentation)** | VLAN, Private VLAN (PVLAN), VRF-Lite, TrustSec (SGT) | L2/L3レベルでのセグメント分離。同セグメント内端末間隔離(PVLAN Isolated Port)。セキュリティグループタグ(SGT)によるロールベース動的マイクロセグメンテーション。 | PVLAN promiscuous/isolated/community ポートのマッピング誤り。SXP (SGT Exchange Protocol) のピアリングおよびタイマー調整。 |
| **境界防衛・スプーフィング防止 (Ingress/Egress Filtering)** | BCP 38 (uRPF), iACL, CoPP, DAI, DHCP Snooping, RA Guard | 送信元IPアドレス偽装の防止(uRPF Strict/Loose)。コントロールプレーン保護(CoPP)。L2インフラ保護(DAI/DHCP Snooping/RA Guard)。 | uRPFの非対称ルーティング環境におけるPacket Drop。DHCP Snooping Rate Limit超過による `err-disable` の自動復旧設定。 |
| **ログ管理・時刻同期・ログ改ざん防止 (Logging & Time Sync)** | Secure Syslog (TLS 6514), NTP with Authentication, SNMPv3 | 暗号化されたログ転送(Syslog over TLS)、NTP MD5/SHA認証による監査ログ時刻整合性保持、SNMPv3 (authPriv - SHA256/AES128) によるセキュアな監視。 | Syslog 転送先が不可達の場合のTCP Logging Bufferの枯渇およびCLIフリーズ防止 (`logging permit-down` / `logging queue-limit`)。 |
| **インシデント検知・自動封じ込め (Incident Response & Automation)** | FMC Security Intelligence, Snort 3, ISE pxGrid, ANC (Adaptive Network Control) | 高度侵入検知/防御(IPS)、IP/URL/File SIブラックリスト、pxGridを介したISEとの連携による感染端末の自動ポート遮断/隔離(ANC Quarantine)。 | FMC Correlation Rule のアクション設定漏れ。pxGrid 認証証明書のチェーンおよび TrustStore 登録ミスの回避。 |

---

## 🏗 動作原理

ISO 27001 準拠のネットワークインフラ防衛アーキテクチャは、**「Defense-in-Depth (多層防御)」** と **「Zero Trust Architecture (ゼロトラスト建築)」** の原則に基づいて動作します。

単一の防御壁（境界ファイアウォールなど）に頼るのではなく、アクセス層（Access Layer）、ディストリビューション/コア層（Distribution/Core Layer）、管理層（Management Plane）、運用・監視層（Monitoring/Log Analytics）の全レイヤーでISO 27001の管理策を多重に適用します。

```
[ エッジ / 接続端末 (Host / User / IoT) ]
          │
          ▼  <-- ISO 27001 A.9 アクセス制御 (802.1X / MAB / MACsec)
[ Access Layer Switch (Cisco Catalyst) ]
          │   ├── L2 Security: DAI, DHCP Snooping, RA Guard, Port Security
          │   └── Encryption: MACsec (802.1AE MKA)
          ▼
[ Distribution / Core Switch & Router ]
          │   ├── L3 Security: uRPF (BCP 38), iACL, CoPP, VRF-Lite
          │   └── Authorization & Tagging: SGT (TrustSec) / SXP
          ▼
[ Next-Generation Firewall / IPS (Cisco FTD / FMC) ]
          │   ├── Access Control Policy, SSL Decryption, Snort 3 IPS
          │   └── Security Intelligence, Malware Analysis (AMP)
          ▼
[ 内部データセンター / クラウド / インターネット ]

=========================== [ 管理 & 監査基盤 ] ===========================
  ├── Cisco ISE (AAA, 802.1X, Policy Engine, pxGrid ANC Quarantine)
  ├── Centralized Log Collector / SIEM (Secure Syslog over TLS 6514)
  ├── Network Telemetry / Monitoring (NetFlow/IPFIX, eStreamer, SNMPv3 authPriv)
  └── NTP Server with SHA/MD5 Authentication (時刻監査証明)
```

### コンプライアンス維持のための多層防御動作原理

1. **認可なき接続の拒否 (Default Deny Access Control)**:
   * ポートレベルで 802.1X / MAB が有効化されており、Cisco ISE による認証・認可が成功するまで、すべてのデータトラフィックをブロック（802.1X Port Unauthorized）。
   * 認証成功後、ISE から動的に Access-List (dACL) または Security Group Tag (SGT) がダウンロード・適用され、最小権限の原則（Principle of Least Privilege）に従って通信を許可。
2. **通信経路の完全性と機密性保護 (Data Confidentiality & Integrity)**:
   * スイッチ間および端末-スイッチ間通信は MACsec により L2 で暗号化・改ざん検知。
   * WAN および拠点間通信は FlexVPN / IPsec (IKEv2 / AES-GCM) により暗号化され、ISO 27001 暗号化要件を満たす。
3. **境界防衛とスプーフィング・攻撃遮断 (Perimeter Protection & Anti-Spoofing)**:
   * L2 では DHCP Snooping Binding Table を元に DAI (Dynamic ARP Inspection) や IP Source Guard (IPSG) が不正な ARP / IP スプーフィングを物理ポートレベルで廃棄。
   * L3 インターフェースでは uRPF (Unicast Reverse Path Forwarding / BCP 38) が送信元 IP アドレスのルーティングテーブル逆引きチェックを行い、偽装パケットを即座に破棄。
   * コントロールプレーンでは CoPP (Control Plane Policing) がルーティングプロトコルや管理通信以外のパケットレートを制御し、DoS/DDoS 攻撃からルータ/スイッチの CPU を保護。
4. **監査追跡可能性と時刻非否認性 (Audit Traceability & Timestamp Integrity)**:
   * すべての管理操作、認証イベント、トラフィックログ、セキュリティアラートは、認証済み NTP サーバーで同期された高精度タイムスタンプ（精度ミリ秒）を付与。
   * Syslog 通信は TLS（TCP 6514）で暗号化されて SIEM に転送され、中間者によるログ改ざん・盗聴を防止。
5. **インシデントの自動検出と動的隔離 (Automated Incident Containment)**:
   * FMC (Firepower Management Center) が Snort 3 や AMP でマルウェア感染や攻撃動作を検知すると、Correlation Rule が発火。
   * FMC は pxGrid 経由で Cisco ISE に ANC (Adaptive Network Control) 隔離リクエストを送信。
   * ISE は即座に CoA (Change of Authorization) をスイッチに送出し、対象端末のポート VLAN を隔離 VLAN に変更するか、SGT を `Quarantine_SGT` に動的変更してネットワークから完全隔離。

---

## ⚙ 動作シーケンス

ISO 27001 統合セキュリティインフラにおける「アクセス認証 ──► 通信監視 ──► 侵入検知 ──► 自動隔離 ──► ログ監査」の完全なパケット・イベント処理シーケンスは以下の通りです。

```
[ クライアント/端末 ]      [ Access Switch ]         [ Cisco ISE ]          [ Cisco FTD/FMC ]       [ Syslog/SIEM ]
       │                         │                        │                        │                       │
       │─── 1. EAPoL-Start ────>│                        │                        │                       │
       │<── 2. EAP-Request/ID ──│                        │                        │                       │
       │─── 3. EAP-Response ───>│─── 4. RADIUS Req ────>│                        │                       │
       │                         │    (EAP-Message)       │                        │                       │
       │                         │                        │── 5. Policy Check ──┐  │                       │
       │                         │                        │   (Identity, Posture│  │                       │
       │                         │                        │    Group, SGT)      │  │                       │
       │                         │<── 6. RADIUS Accept ───│<────────────────────┘  │                       │
       │                         │    (dACL / SGT / VLAN) │                        │                       │
       │<── 7. EAP-Success ──────│                        │                        │                       │
       │                         │                        │                        │                       │
       │── 8. Data Traffic (SGT 認証済み通信) ───────────>│                        │                       │
       │                         │                        │─── 9. RADIUS Accounting ──────────────────────>│
       │                         │                        │    (Start / Interim Log)                       │
       │                         │                        │                        │                       │
       │── 10. 不正攻撃パケット (マルウェア/C2通信/攻撃) ─────────────────────────>│                       │
       │                         │                        │                        │── 11. IPS Alert ────┐ │
       │                         │                        │                        │   (Snort 3 / SI)    │ │
       │                         │                        │                        │<────────────────────┘ │
       │                         │                        │<── 12. pxGrid ANC ─────│                       │
       │                         │                        │    Quarantine Req      │                       │
       │                         │<── 13. RADIUS CoA ─────│                        │                       │
       │                         │    (Re-Auth / Dynamic) │                        │                       │
       │<── 14. EAPoL-Reauth ────│    Isolation SGT / VLAN│                        │                       │
       │    (または Port Block)  │                        │                        │                       │
       │                         │                        │                        │── 15. Secure Syslog ─>│
       │                         │                        │                        │   (Audit Log / TLS)   │
```

### 処理ステップの詳細説明

1. **アクセス要求と認証 (Steps 1–4)**:
   * 端末が物理ポートに接続すると、802.1X EAPoL パケットがスイッチに送出されます。スイッチは Access-Request パケットを生成し、Cisco ISE（RADIUS サーバー）に送出します。
2. **アイデンティティ判定とポリシー適用 (Steps 5–7)**:
   * ISE は証明書（EAP-TLS）やユーザー認証情報、Posture 状態、デバイスタイプを判定し、属性（dACL、VLAN、SGT: Security Group Tag）を含む `RADIUS Access-Accept` を返します。スイッチはポートをオープンし、属性をハードウェア TCAM にロードします。
3. **データ通信とセグメンテーション/境界防御 (Steps 8–9)**:
   * 端末のデータパケットは、SGT / Access Control Policy や L2/L3 防御（DAI, uRPF）を通過してルーティング/スイッチングされます。同時期に RADIUS Accounting が発行され、監査ログとして SIEM に記録されます。
4. **脅威発生とリアルタイム検知 (Steps 10–11)**:
   * 端末がマルウェアに感染し、C2 通信や内部スキャン等の不正通信を試みると、Cisco FTD / Snort 3 IPS エンジンがこれを即座に検知し、ブロック（Drop）します。
5. **自動追跡と動的遮断/隔離 (Steps 12–14)**:
   * FMC は Correlation Rule に基づき、pxGrid ポータルを介して Cisco ISE に `ANC Quarantine`（適応型ネットワーク制御による隔離）メッセージを発行します。
   * ISE は即座に `RADIUS CoA (Change of Authorization)` パケットをスイッチに送出し、スイッチは該当ポートに対して再認証要求を行うか、SGT を `Quarantine_SGT`（すべての内部アクセス拒否）へ動的変更、またはポートをシャットダウンします。
6. **監査ログの記録と非否認性の確立 (Step 15)**:
   * 認証イベント、検知イベント、CoA 実行結果、管理者の操作ログがすべて Secure Syslog (TLS) 経由で暗号化され、タイムスタンプ付きで中央 SIEM サーバーへ自動転送・蓄積されます。


---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、「ISO 27001 コンプライアンス」「組織のセキュリティ要件に基づくインフラ保護」が出題された場合、受験者は単にコマンドを入力するだけでなく、**「要求されたISO 27001管理策（アクセス制御、ログ暗号化、IPスプーフィング防止、認証付き時刻同期など）をどの技術で実現すべきか」**を正しく判断し、正しく相互連携させる必要があります。

### Blueprintで重要なポイント

1. **アクセス制御と最小権限（ISO 27001 A.9 / 8.2）**:
   * **AAA & TACACS+ Command Authorization**: 特権レベル（Privilege 15）のユーザーであっても、特定の非許可コマンドの実行をブロックする TACACS+ コマンド認可要件。
   * **Parser Views & Executive Views**: TACACS+ サーバ非依存でローカルでコマンド実行権限を制限する IOS-XE の `parser view` 設定。
   * **802.1X / MAB & Dynamic Authorization**: Cisco ISE と連携した dACL (Dynamic ACL) および SGT (Security Group Tag) のダウンロード。
2. **ログの完全性と非否認性（ISO 27001 A.12 / 8.15）**:
   * **Secure Syslog over TLS**: UDP 514 の非暗号化 Syslog ではなく、TLS（TCP 6514）を使用した証明書ベースの暗号化 Syslog 転送。
   * **NTP Authentication**: ログのタイムスタンプ偽装を防ぐための NTP MD5/SHA 認証および自作 NTP ピアの制限。
3. **通信のセキュリティとスプーフィング対策（ISO 27001 A.13 / 8.20）**:
   * **BCP 38 / uRPF**: ネットワーク境界（WAN / Internet / Access）における Unicast RPF の Strict vs Loose モード適用。
   * **L2 防御群の完全実装**: DHCP Snooping、DAI (Dynamic ARP Inspection)、IP Source Guard (IPSG)、IPv6 RA Guard の統合適用。
4. **暗号化要件（ISO 27001 A.10 / 8.24）**:
   * **MACsec (802.1AE MKA)**: スイッチ間（Trunk リンク）または端末-スイッチ間のデータプレーン暗号化。
   * **FlexVPN / IKEv2 Suite B**: 拠点間 VPN における CNSA / Suite B 暗号仕様（AES-GCM-256、SHA-384、ECDSA P-384）の強制。

### ラボ試験で狙われやすい設定ミス・落とし穴

* **Syslog TCP ブロッキング問題**: `logging host <IP> transport tcp port 6514` を設定した際、Syslog サーバーへの接続がダウンすると、ルータ/スイッチの CLI 入力がフリーズまたは著しく遅延する。対策として `logging permit-down` や `logging queue-limit` を忘れず設定する。
* **uRPF 非対称ルーティングでのデータドロップ**: 送信ルートと受信ルートが異なる環境で `ip verify unicast source reachable-via rx` (Strict Mode) を設定し、正当なトラフィックを大量ドロップさせてしまう。非対称経路では `reachable-via any` (Loose Mode) または ACL による例外許可を行う。
* **MACsec MTU オーバーヘッドの見落とし**: MACsec（MKA）を有効にすると、802.1AE ヘッダー（8〜32 バイト）が付与され、標準の 1500 バイトパケットがフラグメントまたはドロップする。インターフェース MTU の拡大 (`mtu 1530` や `jumbo frame`) を合わせて設定すること。
* **TACACS+ コマンド認可のバイパス要件**: 試験要件で「コンソール接続時は AAA サーバがダウンしていてもアクセス可能にすること」と指示されている場合、`aaa authentication login default group tacacs+ local` および `aaa authorization exec default group tacacs+ local` のバックアップ記述漏れに注意。

---

## 🛠 設定方法

ISO 27001 管理策要件を Cisco IOS-XE、Cisco ISE、および Cisco FTD/FMC で実装するための完全な設定例を示します。

### 1. アクセス制御・特権分離 (ISO 27001 A.9 / 8.2 適合)

#### Cisco IOS-XE: TACACS+ コマンド認可 & ローカル Parser View 設定

```bash
! --- 1.1 TACACS+ サーバー定義と AAA 有効化 ---
aaa new-model
!
tacacs server ISE-TACACS-1
 address ipv4 192.168.10.100
 key 6 Cisc0Secur1tyKey!
!
tacacs server ISE-TACACS-2
 address ipv4 192.168.10.101
 key 6 Cisc0Secur1tyKey!
!
aaa group server tacacs+ TAC-GROUP
 server name ISE-TACACS-1
 server name ISE-TACACS-2
!
! --- 1.2 認証・認可・会計のポリシー定義 ---
aaa authentication login default group TAC-GROUP local
aaa authentication enable default group TAC-GROUP enable
aaa authorization exec default group TAC-GROUP local 
aaa authorization commands 15 default group TAC-GROUP local 
aaa accounting exec default start-stop group TAC-GROUP
aaa accounting commands 15 default start-stop group TAC-GROUP
!
! --- 1.3 ローカルバックアップ管理者と Parser View 設定 ---
username admin privilege 15 secret 9 $9$x8zK9L...
!
! TACACS+ サーバ非依存のロールベースビュー (ISO 27001 役割分離要件)
parser view AUDITOR_VIEW
 secret 9 $9$mKw1...
 commands exec include show running-config
 commands exec include show logging
 commands exec include show tech-support
 commands exec include show ip route
!
line vty 0 15
 transport input ssh
 exec-timeout 10 0
 logging synchronisation
```

### 2. ログの完全性・暗号化と認証付き時刻同期 (ISO 27001 A.12 / 8.15 適合)

#### Cisco IOS-XE: Secure Syslog (TLS 6514) & NTP SHA-1/MD5 認証設定

```bash
! --- 2.1 認証付き NTP 設定 (時刻の完全性非否認性) ---
ntp authenticate
ntp authentication-key 1 md5 NtpAuthKey2026!
ntp trusted-key 1
ntp server 192.168.1.50 key 1 prefer
ntp source Loopback0
!
! --- 2.2 ログのタイムスタンプミリ秒設定 ---
service timestamps log datetime msec show-timezone localtime
service timestamps debug datetime msec show-timezone localtime
!
! --- 2.3 Secure Syslog (TLS 6514) 設定 ---
crypto pki trustpoint SYSLOG-CA
 enrollment pkcs12
 revocation-check crl none
!
logging host 192.168.10.50 transport tcp port 6514 target-vrf Mgmt-vrf
logging source-interface Loopback0
logging trap informations
logging facility local7
! Syslog サーバー障害時の CLI フリーズ防止設定 (重要)
logging permit-down
logging queue-limit 1000
```

### 3. 転送中データの暗号化 (ISO 27001 A.10 / 8.24 適合)

#### Cisco IOS-XE: 802.1AE MACsec (MKA) スイッチ間暗号化設定

```bash
! --- 3.1 MACsec Key Chain と Pre-Shared Key の定義 ---
key chain MACSEC-KEYCHAIN macsec
 key 0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF
  cryptographic-algorithm aes-256-cmac
  key-string 1234567890ABCDEF1234567890ABCDEF1234567890ABCDEF1234567890ABCDEF
!
! --- 3.2 MACsec CKN/CAK ポリシー定義 ---
mka policy MACSEC-POLICY
 macsec-cipher-suite gcm-aes-256
 key-server-priority 16
 confidentiality-offset 0
!
! --- 3.3 トランクインターフェースへの MACsec 適用 ---
interface GigabitEthernet1/0/48
 description Trunk Link with MACsec Encryption
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,100
 mtu 1530
 macsec
 macsec network-link
 mka policy MACSEC-POLICY
 mka pre-shared-key key-chain MACSEC-KEYCHAIN
```

### 4. 境界防衛・IPスプーフィング防止 (ISO 27001 A.13 / 8.20 適合)

#### Cisco IOS-XE: BCP 38 / uRPF & L2 Security 設定

```bash
! --- 4.1 L3 インターフェースでの uRPF (BCP 38) 設定 ---
interface GigabitEthernet1/0/1
 description External ISP / WAN Edge Interface
 ip address 203.0.113.2 255.255.255.252
 ! 送信元IPの逆引きルーティング検証 (Strict Mode)
 ip verify unicast source reachable-via rx allow-default
!
! --- 4.2 L2 アクセスポートセキュリティ統合 ---
interface Range GigabitEthernet1/0/1 - 24
 switchport mode access
 switchport access vlan 10
 ! Port Security
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security aging time 10
 switchport port-security aging type inactivity
 ! DHCP Snooping & DAI 適用
 ip dhcp snooping limit rate 15
 ip arp inspection limit rate 20
 ip verify source
```


---

## 🔍 検証コマンド

ISO 27001 関連コントロールの実装状態、統計情報、およびデバッグを検証するための代表的なコマンド一覧です。

| 目的 | コマンド | 確認・評価項目 |
| --- | --- | --- |
| **TACACS+ / AAA 認可状態確認** | <code>show aaa sessions</code> | アクティブな AAA セッションおよび認可済レベル |
| **Parser View 権限確認** | <code>show parser view</code> | 現在のログインユーザーが属する View と利用可能コマンド |
| **NTP 認証・同期状態確認** | <code>show ntp status</code> / <code>show ntp associations</code> | NTP サーバーとの同期状態 (Synchronized)、Stratum、認証失敗数 |
| **Syslog 転送・バッファ確認** | <code>show logging</code> | Syslog サーバー宛ての送信メッセージ数、ドロップ数、TLS 送信状態 |
| **MACsec (MKA) 暗号化確認** | <code>show mka sessions</code> / <code>show macsec interface</code> | MKA セッション確立状態、使用暗号スイート (GCM-AES-256)、Encrypted Packets |
| **uRPF (BCP 38) ドロップ確認** | <code>show ip traffic | include Drop</code> | uRPF 逆引き失敗によりドロップされたパケット統計 |
| **uRPF インターフェース詳細** | <code>show cef interface <interface></code> | CEF で有効化されている uRPF モード (Strict / Loose) |
| **DHCP Snooping / DAI 状態** | <code>show ip dhcp snooping binding</code> / <code>show ip arp inspection</code> | バインディングテーブル、DAI ドロップカウンタ、Rate Limit ドロップ |
| **AAA / EAP デバッグ** | <code>debug radius</code> / <code>debug aaa authorization</code> | RADIUS/TACACS+ 通信および認可レスポンスのデバッグ |
| **MKA / MACsec デバッグ** | <code>debug mka events</code> / <code>debug macsec errors</code> | MACsec 鍵交換 (MKA) および暗号化処理のエラーログ |

---

## 🚨 トラブルシュート

ISO 27001 関連の技術実装において発生しやすい障害、その原因、確認コマンド、および解決手順をまとめます。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **Syslog 転送設定後に CLI 入力がフリーズする** | Syslog over TCP (Port 6514) 設定時、宛先サーバー不可達により TCP 出力バッファが枯渇し CLI がブロックされた。 | <code>show logging</code> でバッファフルとドロップログを確認 | `logging permit-down` を設定し、Syslog ダウン時でも CLI 出力を維持させる。`logging queue-limit 1000` を追加。 |
| **uRPF 有効化後に正常トラフィックがドロップする** | 非対称ルーティング環境で uRPF Strict Mode を設定したため、戻り経路の I/F と受信 I/F が不一致となり破棄された。 | <code>show cef interface <interface></code> / <code>show ip traffic</code> | uRPF モードを Loose Mode (`reachable-via any`) に変更するか、ACL による例外許可を追加。 |
| **MACsec (MKA) セッションが UP しない** | 両対向の Key Chain / Key-String の不一致、または MKA Policy の暗号スイート不一致。 | <code>show mka sessions</code> / <code>debug mka events</code> | Pre-Shared Key (CKN / CAK) の文字列および暗号化アルゴリズム (AES-256-CMAC 等) を一致させる。 |
| **NTP 同期が失敗しログの時刻がずれる** | NTP 認証キーの不一致、または Firewall/ACL で UDP 123 パケットが遮断されている。 | <code>show ntp associations detail</code> | `ntp authentication-key` および `ntp trusted-key` の整合性を確認。中間ルータで UDP 123 を許可。 |
| **ISE CoA 実行時に端末が隔離されない** | スイッチ側で RADIUS Dynamic Authorization (CoA) の受信用 Port 1700 が開放されていない、または Shared Secret 不一致。 | <code>show aaa server</code> / <code>debug radius</code> | スイッチ上で `aaa server radius dynamic-author` を定義し、`client <ISE_IP> server-key` を正しく設定。 |

---

## ⚠ 制限事項

* **プラットフォームによる MACsec ハードウェア非対応**: 古いラインカードや一部の Catalyst スイッチ（例: 階層の低いアクセススイッチ）では MACsec (802.1AE) の ASIC 処理がサポートされておらず、設定コマンドが存在しないか CPU ソフトウェア処理となり帯域が著しく低下します。
* **uRPF と MPLS / VRF の組み合わせ制約**: VRF 構成において uRPF を適用する場合、`ip verify unicast source reachable-via rx` に VRF 認識オプションが含まれていないと正しくルーティングテーブルを参照できません。
* **FMC / FTD と Syslog TLS の証明書制限**: FMC から暗号化 Syslog (TLS) を送信する場合、FMC の Trusted CA ストアに対象 Syslog サーバーのルート CA 証明書がインポートされていないと TLS ハンドシェイクが失敗します。

---

## 🔄 他技術との関連

ISO 27001 準拠設計は、Blueprint の他の多くの技術と密接に相互連携します。

* **3.1 Control Plane Security (CoPP / iACL)**: 制御プレーンを DoS 攻撃から保護し、ISO 27001 A.12/A.13 の可用性・セキュリティ運用要件を満たします。
* **3.3 Data Plane Security (uRPF / RTBH)**: BCP 38 / uRPF を用いて送出・流入パケットの送信元偽装を遮断します。
* **3.4 Layer 2 Security (DAI / DHCP Snooping / RA Guard / Port Security)**: L2 アクセス層における不正デバイス接続・スプーフィング・MITM 攻撃を防ぎ、ISO 27001 A.9 / A.13 に適合します。
* **2.6 Cisco TrustSec (SGT / SXP)**: 従来のアドレスベース ACL に代わり、ロールベースの動的タグ（SGT）を付与してネットワーク全体で最小権限認可ポリシーを維持します。

---

## 🧩 比較表

各種セキュリティフレームワークおよび主要管理手法の比較です。

### 1. ガバナンス・セキュリティフレームワークの比較

| 項目 | ISO/IEC 27001:2022 | NIST SP 800-53 (Rev. 5) | PCI-DSS v4.0 |
| --- | --- | --- | --- |
| **主な対象** | 全業種の総合的 ISMS（情報セキュリティ管理体制） | 米国連邦政府・政府機関および関連サプライヤー | クレジットカード情報を扱うすべての事業者 |
| **規格の性質** | 承認・認証制度のある国際標準規格 (ISO) | 米国標準技術研究所 (NIST) のガイドライン | カードブランド5社が共同制定した民間セキュリティ基準 |
| **技術管理策の特徴** | Annex A (93の管理策) による組織・人的・物理・技術の包括的コントロール | 20 のセキュリティコントロールファミリーによる極めて詳細な技術的要件 | 12 の明確な要求事項（ログ保存期間1年、パスワード長12文字以上など具体的数字規定） |
| **Cisco インフラでの位置付け** | ISMS 全体の中でアクセス制御・暗号化・監視・ログ保持を全般的に実装 | ハイセキュリティ環境（FIPS 140-3, Suite B / CNSA 1.0）での高度な暗号化実装 | ポートセキュリティ、暗号化(TLS1.3/MACsec)、厳格なSyslog保持・アクセス制限の強制 |

### 2. アクセス制御認可方式の比較 (ISO 27001 A.9 適合技術)

| 認可方式 | 制御ポイント | 柔軟性・拡張性 | 運用負荷 | ISO 27001 適合評価 |
| --- | --- | --- | --- | --- |
| **dACL (Dynamic ACL)** | IP アドレス / サブネット | 低（IP 変更に弱い） | 高（大量の ACL 管理が必要） | 中（アクセス制限は達成可能だが複雑化しやすい） |
| **VLAN Assignment** | サブネット / 物理 VLAN | 低（VLAN スケーラビリティ限界） | 中（VLAN ID 管理とトランク設定が必要） | 中（セグメント分離は明確だが粒度が粗い） |
| **SGT (TrustSec)** | 65,535 のグループタグ | 極めて高（位置・IP に非依存） | 低（ISE で中央一元管理） | **極めて高**（最小権限の動的マイクロセグメンテーションを実現） |

---

## 💡 ベストプラクティス

ISO 27001 準拠の Cisco ネットワークインフラを構築・運用する際の推奨設計指針です。

1. **暗号化通信のデフォルト化 (Encryption by Default)**:
   * 管理通信は HTTP / Telnet / SNMPv1/v2c を全廃し、HTTPS (TLS 1.2/1.3)、SSHv2 (RSA 3048bit / ECDSA)、SNMPv3 (authPriv) に限定。
   * スイッチ間トランクリンクには原則として MACsec (802.1AE) を適用し、物理層タップ攻撃から保護。
2. **監査ログの非否認性および暗号化転送**:
   * NTP は必ず HMAC-MD5 または SHA-1 認証を有効化し、信頼された内部 NTP サーバーとのみ同期。
   * Syslog 転送には TLS (TCP 6514) を使用し、転送路上の改ざん・盗聴を防止。ログバッファ枯渇対策として `logging permit-down` を明示的に設定。
3. **アクセス制御の一元化と動的隔離 (Centralized AAA & Automated RTC)**:
   * Cisco ISE を核とした TACACS+ 管理者アクセス制御および 802.1X / MAB ユーザーアクセス制御を導入。
   * FMC と ISE を pxGrid で統合し、IPS / マルウェア検知時に自動で ANC Quarantine 隔離を発生させる自動レスポンスを確立。


---

## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験レベルの実践的な設定演習サンプルを 10 個掲載します。各演習には「問題」「要件」「完全な設定例」「検証方法」を含めています。

---

### 演習サンプル 1: ISO 27001 適合 TACACS+ 管理者アクセス制御 & コマンド認可

#### 【問題】
ルータ R1 において、ISO 27001 A.9（アクセス制御）要件に基づき、TACACS+ を用いた管理者認証・コマンド認可・ログ会計を設定し、ローカルバックアップアクセスを確保しなさい。

#### 【要件】
1. TACACS+ サーバー `192.168.10.100` （Shared Key: `Cisc0Secur1tyKey!`）を使用すること。
2. 特権レベル 15 への昇格時およびコマンド実行時に TACACS+ サーバーによる認可を行うこと。
3. TACACS+ サーバーがダウンしている場合のみ、ローカルユーザー `admin`（Secret: `Cisco123!`）でのログインを許可すること。
4. SSH 通信のみを許可し、Exec Timeout を 10 分に設定すること。

#### 【設定例】
```bash
! R1 Configuration
aaa new-model
!
tacacs server ISE-TACACS
 address ipv4 192.168.10.100
 key 6 Cisc0Secur1tyKey!
!
aaa group server tacacs+ TAC-GROUP
 server name ISE-TACACS
!
aaa authentication login default group TAC-GROUP local
aaa authentication enable default group TAC-GROUP enable
aaa authorization exec default group TAC-GROUP local 
aaa authorization commands 15 default group TAC-GROUP local 
aaa accounting exec default start-stop group TAC-GROUP
aaa accounting commands 15 default start-stop group TAC-GROUP
!
username admin privilege 15 secret Cisco123!
!
line vty 0 15
 transport input ssh
 exec-timeout 10 0
 logging synchronisation
```

#### 【検証方法】
```bash
! 1. TACACS+ サーバー接続および認可状態の確認
show aaa sessions
show aaa server

! 2. VTY ラインへの SSH ログインおよびコマンド実行のテスト
ssh -l admin 192.168.10.1
! (認可失敗・成功イベントが ISE TACACS+ Live Logs に記録されることを確認)
```

---

### 演習サンプル 2: 暗号化 Syslog (TLS 6514) と認証付き NTP による監査トレイル保護

#### 【問題】
スイッチ SW1 において、ISO 27001 A.12（ログ管理）要件を満たすため、認証付き NTP による時刻同期と Secure Syslog (TLS) によるログ転送を設定しなさい。

#### 【要件】
1. NTP サーバー `192.168.1.50` と MD5 認証（Key ID: 1, Key: `NtpKey2026!`）で時刻同期すること。
2. ログメッセージにはミリ秒精度のタイムスタンプを付与すること。
3. Syslog サーバー `192.168.10.50` に対し、TCP ポート 6514 (TLS) でログを転送すること。
4. Syslog サーバー停止時にも CLI がフリーズしないようフリーズ防止対策を施すこと。

#### 【設定例】
```bash
! SW1 Configuration
service timestamps log datetime msec show-timezone localtime
service timestamps debug datetime msec show-timezone localtime
!
ntp authenticate
ntp authentication-key 1 md5 NtpKey2026!
ntp trusted-key 1
ntp server 192.168.1.50 key 1 prefer
!
logging host 192.168.10.50 transport tcp port 6514
logging source-interface Loopback0
logging trap information
logging facility local7
logging permit-down
logging queue-limit 1000
```

#### 【検証方法】
```bash
! 1. NTP 同期状態および認証結果の検証
show ntp status
show ntp associations detail

! 2. Logging 設定および Syslog 送信バッファ状態の確認
show logging
```

---

### 演習サンプル 3: L2 境界防衛 (DHCP Snooping & DAI & IP Source Guard)

#### 【問題】
アクセススイッチ SW2 の VLAN 10 において、ISO 27001 A.13 に従い、ARP スプーフィングおよび IP 偽装攻撃を完全に阻止する L2 セキュリティを設定しなさい。

#### 【要件】
1. VLAN 10 で DHCP Snooping を有効化し、ポート `GigabitEthernet1/0/24` を Trusted ポートに設定すること。
2. アクセスポート `GigabitEthernet1/0/1 - 10` で DHCP パケットの Rate Limit を 15 pps に制限し、超過時はポートを `err-disable` にすること。
3. VLAN 10 で Dynamic ARP Inspection (DAI) を有効化し、`GigabitEthernet1/0/24` を Trusted ポートに設定すること。
4. アクセスポートで IP Source Guard を有効化し、IP/MAC 偽装パケットをドロップすること。

#### 【設定例】
```bash
! SW2 Configuration
ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option
!
ip arp inspection vlan 10
!
interface GigabitEthernet1/0/24
 description Uplink to Distribution Router
 switchport mode trunk
 ip dhcp snooping trust
 ip arp inspection trust
!
interface range GigabitEthernet1/0/1 - 10
 switchport mode access
 switchport access vlan 10
 ip dhcp snooping limit rate 15
 ip arp inspection limit rate 20
 ip verify source
!
! err-disable 自動復旧の設定
errdisable recovery cause dhcp-rate-limit
errdisable recovery cause arp-inspection
errdisable recovery interval 60
```

#### 【検証方法】
```bash
! 1. DHCP Snooping バインディングの確認
show ip dhcp snooping binding

! 2. DAI 統計およびドロップ数の確認
show ip arp inspection vlan 10

! 3. IP Source Guard テーブルの確認
show ip verify source
```

---

### 演習サンプル 4: 拠点間トランクの 802.1AE MACsec (MKA) 暗号化

#### 【問題】
スイッチ SW1 と SW2 間のトランクリンク (`GigabitEthernet1/0/48`) において、盗聴・改ざんを防ぐため 802.1AE MACsec (MKA) による暗号化を設定しなさい。

#### 【要件】
1. Pre-Shared Key (PSK) 方式の MKA を使用し、キーチェーン名 `MACSEC-KC`、暗号化アルゴリズム `aes-256-cmac` を使用すること。
2. MKA ポリシー `SEC-MKA` を作成し、Cipher Suite に `gcm-aes-256` を指定すること。
3. MTU オーバーヘッドに対応するため、該当ポートの MTU を 1530 バイトに変更すること。

#### 【設定例】
```bash
! SW1 / SW2 Configuration
key chain MACSEC-KC macsec
 key 0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF
  cryptographic-algorithm aes-256-cmac
  key-string 1234567890ABCDEF1234567890ABCDEF1234567890ABCDEF1234567890ABCDEF
!
mka policy SEC-MKA
 macsec-cipher-suite gcm-aes-256
 key-server-priority 16
!
interface GigabitEthernet1/0/48
 description MACsec Protected Trunk Link
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 mtu 1530
 macsec
 macsec network-link
 mka policy SEC-MKA
 mka pre-shared-key key-chain MACSEC-KC
```

#### {検証方法】
```bash
! 1. MKA セッション確立の確認
show mka sessions

! 2. MACsec 暗号化統計の確認
show macsec interface GigabitEthernet1/0/48
```

---

### 演習サンプル 5: WAN エッジにおける BCP 38 / uRPF スプーフィング遮断

#### 【問題】
ルータ R1 の WAN インターフェース (`GigabitEthernet0/0`) において、ISO 27001 A.13 の境界セキュリティ要件に従い、送信元 IP アドレス偽装パケットをドロップする uRPF (Strict Mode) を設定しなさい。

#### 【要件】
1. `GigabitEthernet0/0` で uRPF Strict Mode を有効化すること。
2. デフォルトルート経由のルーティング判定も許可する (`allow-default`) オプションを付与すること。
3. ドロップされたパケット数を統計情報で確認可能にすること。

#### 【設定例】
```bash
! R1 Configuration
ip cef
!
interface GigabitEthernet0/0
 description WAN Edge Interface
 ip address 203.0.113.2 255.255.255.252
 ip verify unicast source reachable-via rx allow-default
```

#### 【検証方法】
```bash
! 1. CEF インターフェースでの uRPF 設定状態確認
show cef interface GigabitEthernet0/0

! 2. uRPF ドロップ統計の確認
show ip traffic | include Drop
```

---

### 演習サンプル 6: Private VLAN による同セグメント内端末間アクセス隔離

#### 【问题】
スイッチ SW1 の VLAN 100（DMZ）において、同一セグメント内のサーバー同士の直接通信を禁止し、ルータ経由でのみ通信を許可する Private VLAN を設定しなさい。

#### 【要件】
1. プライマリ VLAN 100、アイソレーテッド VLAN 101 を作成しマッピングすること。
2. ルータ接続ポート `GigabitEthernet1/0/24` を Promiscuous ポートに設定すること。
3. サーバー接続ポート `GigabitEthernet1/0/1 - 10` を Isolated ポートに設定すること。

#### 【設定例】
```bash
! SW1 Configuration
vtp mode transparent
!
vlan 101
 private-vlan isolated
!
vlan 100
 private-vlan primary
 private-vlan association 101
!
interface GigabitEthernet1/0/24
 description Gateway Router Connection
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 100 101
!
interface range GigabitEthernet1/0/1 - 10
 description DMZ Server Connections
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101
```

#### 【検証方法】
```bash
! 1. Private VLAN 設定およびポートマッピングの検証
show vlan private-vlan
show interface private-vlan mapping
```

---

### 演習サンプル 7: Cisco TrustSec SGT による動的ロールベースマイクロセグメンテーション

#### 【問題】
コアスイッチ SW1 において、Cisco ISE からダウンロードされた SGT (Security Group Tag) に基づき、`Quarantine_Group` (SGT 15) から `Server_Group` (SGT 10) への全通信をブロックする SGACL を設定しなさい。

#### 【要件】
1. TrustSec を有効化し、SGT 割り当ておよび SGACL 制御を有効化すること。
2. SGT 15 から SGT 10 への IPv4 トラフィックを拒否 (Deny) し、それ以外の通信を許可 (Permit) する SGACL を定義・適用すること。

#### 【設定例】
```bash
! SW1 Configuration
cts status
cts role-based enforcement
!
ip access-list role-based SGACL-QUARANTINE-TO-SERVER
 10 deny ip
!
cts role-based permissions dst-sgt 10 src-sgt 15 SGACL-QUARANTINE-TO-SERVER
```

#### 【検証方法】
```bash
! 1. TrustSec ロールベース権限マッピングの確認
show cts role-based permissions

! 2. SGACL ハードウェアカウンタの確認
show cts role-based counters
```

---

### 演習サンプル 8: FMC 侵入検知から ISE pxGrid を介した自動感染端末隔離 (ANC Quarantine)

#### 【問題】
FMC (Firepower Management Center) において、Snort 3 IPS がマルウェア感染アクションを検知した際、pxGrid を経由して Cisco ISE に即座に ANC Quarantine (端末隔離) を要求する Correlation Policy を構築しなさい。

#### 【要件】
1. FMC の `pxGrid System Component` で ISE との pxGrid セッションを UP 状態に維持すること。
2. Correlation Rule `Rule-Malware-Detected` を定義し、侵入イベントの Priority が High の場合発火させること。
3. 該当ルール発火時、ISE 応答アクション `Apply ANC Policy: Quarantine` を実行させること。

#### 【設定例 (FMC CLI / API & Conceptガイド)】
```bash
! FMC CLI での pxGrid 接続ステータス確認
sudo Omniquery.pl -realm=sys -cmd="select * from pxgrid_status;"

! Correlation Rule 構造 (FMC Web UI 設定要件)
Rule Name: Block-Infected-Host
If: An intrusion event occurs where Impact Flag is Red / High
Action: Remediation -> Cisco ISE pxGrid -> Quarantine Host (by Source IP)
```

#### 【検証方法】
```bash
! 1. ISE 側での Adaptive Network Control (ANC) 適用状態確認
! (ISE Operations -> ANC -> Active Sessions に該当 MAC/IP が Quarantine として登録されていることを確認)

! 2. 対象スイッチでの CoA 実行ログ確認
show aaa server
show cts environment
```

---

### 演習サンプル 9: SNMPv3 (authPriv - SHA256/AES128) によるセキュア監視

#### 【問題】
ルータ R1 において、ISO 27001 A.12（監視）に基づき、暗号化・認証付きの SNMPv3 (authPriv) を設定し、MIB アクセス範囲を最小化しなさい。

#### 【要件】
1. SNMPv3 ビュー `RESTRICTED-VIEW` を作成し、`system` および `interfaces` 枝（OID: 1.3.6.1.2.1.1 および 1.3.6.1.2.1.2）のみ閲覧を許可すること。
2. SNMPv3 グループ `SEC-GROUP` を作成し、`RESTRICTED-VIEW` を割り当てること。
3. ユーザー `snmpadmin` を作成し、認証アルゴリズム `SHA-256`（Pass: `AuthPass2026!`）、暗号化アルゴリズム `AES 128`（Pass: `PrivPass2026!`）を設定すること。

#### 【設定例】
```bash
! R1 Configuration
snmp-server view RESTRICTED-VIEW system include
snmp-server view RESTRICTED-VIEW interfaces include
!
snmp-server group SEC-GROUP v3 priv read RESTRICTED-VIEW access 10
!
snmp-server user snmpadmin SEC-GROUP v3 auth sha AuthPass2026! priv aes 128 PrivPass2026!
!
access-list 10 permit 192.168.10.50
```

#### 【検証方法】
```bash
! 1. SNMPv3 ユーザーおよびグループ設定の確認
show snmp user
show snmp group

! 2. 外部 NMS からの snmpwalk テスト (authPriv)
```

---

### 演習サンプル 10: Control Plane Policing (CoPP) による DoS 攻撃耐性強化

#### 【問題】
ルータ R1 において、ISO 27001 A.13（可用性・インフラ防衛）に基づき、コントロールプレーン宛てトラフィックを制限する CoPP を構築しなさい。

#### 【要件】
1. SSH / TACACS+ / BGP 通信を `CP-CRITICAL` クラスに割り当て、帯域 1 Mbps で優先転送すること。
2. ICMP 通信を `CP-ICMP` クラスに割り当て、500 kbps でレート制限し、超過分をドロップすること。
3. その他の未定義パケットを 100 kbps にポリシングすること。

#### 【設定例】
```bash
! R1 Configuration
ip access-list extended ACL-CP-CRITICAL
 permit tcp any any eq 22
 permit tcp any any eq 49
 permit tcp any eq 49 any
 permit tcp any any eq 179
!
ip access-list extended ACL-CP-ICMP
 permit icmp any any
!
class-map match-any CLASS-CP-CRITICAL
 match access-group name ACL-CP-CRITICAL
!
class-map match-any CLASS-CP-ICMP
 match access-group name ACL-CP-ICMP
!
policy-map POLICY-COPP
 class CLASS-CP-CRITICAL
  police 1000000 conform-action transmit exceed-action transmit
 class CLASS-CP-ICMP
  police 500000 conform-action transmit exceed-action drop
 class class-default
  police 100000 conform-action transmit exceed-action drop
!
control-plane
 service-policy input POLICY-COPP
```

#### 【検証方法】
```bash
! 1. CoPP ポリシー適用およびドロップ統計の検証
show policy-map control-plane
```


---

## ❓ 想定試験問題

ISO 27001 関連の技術理解と実装能力を問う CCIE Security ラボ・筆記レベルの想定問題セットです。

---

### 質問 1 (コンフィグ読解・トラブルシュート)

#### 問題
ある企業で ISO 27001 監査に対応するため、IOS-XE スイッチ SW1 にて Syslog サーバー `10.1.1.50` 宛てに TCP ポート 6514 (TLS) によるログ転送を設定した。しかし、Syslog サーバーの定期メンテナンス中にスイッチのコンソールおよび VTY ログイン後の CLI 操作が完全にフリーズした。
コンフィグを確認したところ以下の記述があった。

```bash
logging host 10.1.1.50 transport tcp port 6514
logging trap informational
```

この問題の根本原因を説明し、スイッチの運用可活性を維持しつつ ISO 27001 要件を満たすための正解コンフィグを記述しなさい。

#### 解答・解説
* **原因**: IOS-XE の Syslog over TCP 設定では、デフォルトで送信バッファがフルになり TCP 接続が未完了・切断状態になると、CLI プロセスがログの送出待ちのために同期ブロックされます。これにより Syslog サーバーが停止すると CLI 全体が応答不能になります。
* **正解コンフィグ**:
```bash
logging host 10.1.1.50 transport tcp port 6514
logging permit-down
logging queue-limit 1000
```
`logging permit-down` コマンドを追加することで、Syslog トランスポート層がダウンしても CLI 処理のブロックを解除し、ローカル運用を維持できます。

---

### 質問 2 (設計・uRPF / BCP 38)

#### 問題
組織の WAN エッジルータ R1 において、ISP-A と ISP-B の 2 社の回線を使用したマルチホームマルチパス構成（非対称ルーティング環境）を導入している。送信元 IP アドレス偽装攻撃（BCP 38 要件）を防止するため uRPF を導入したい。

1. `ip verify unicast source reachable-via rx` (Strict Mode) を適用した場合に発生する技術的問題を説明しなさい。
2. この非対称環境で正しく偽装パケットのみをブロックするための推奨設定コマンドを記述しなさい。

#### 解答・解説
1. **発生する問題**: Strict Mode (`reachable-via rx`) は、パケットを受信したインターフェースと、ルーティングテーブル（FIB）における対象送信元 IP への最適出力インターフェースが完全一致することを要求します。非対称ルーティングでは、ISP-A から入ってきたパケットの戻りルートが ISP-B 宛てになっていると、逆引きチェックに失敗し、正当なトラフィックがドロップします。
2. **推奨設定コマンド**:
```bash
interface GigabitEthernet0/0
 ip verify unicast source reachable-via any allow-default
```
`reachable-via any` (Loose Mode) を使用することで、FIB テーブル上に送信元 IP への有効な経路が存在すれば受信を許可し、ルーティングテーブルに存在しない偽装 IP からのパケットのみを破棄できます。


---

### 質問 3 (実装・802.1AE MACsec)

#### 問題
ISO 27001 暗号化要件を満たすため、スイッチ SW1 と SW2 間の 10G トランクリンクにおいて MACsec (MKA) を設定した。設定後、MACsec セッションは UP したが、クライアント間での Jumbo Frame 通信が途切れ、一部のアプリケーションで通信エラーが発生した。

この現象が発生する理由と、スイッチのインターフェースで実施すべき設定変更を記述しなさい。

#### 解答・解説
* **発生理由**: MACsec (802.1AE) は、L2 フレームに 802.1AE MACsec ヘッダーおよび ICV (Integrity Check Value) を付与するため、合計 24〜32 バイトのパケットオーバーヘッドが発生します。標準 MTU 1500 バイトでカプセル化されるとフレームサイズが 1524〜1532 バイトに膨らみ、MTU 超過によりドロップされます。
* **対処コンフィグ**:
```bash
interface GigabitEthernet1/0/48
 mtu 1530
```
トランクインターフェースの物理 MTU を 1530 バイト以上に拡張します。

---

### 質問 4 (マルチベンダー / 管理策マッピング)

#### 問題
ISO 27001 監査において、「ネットワーク内で重大なセキュリティインシデント（マルウェア感染、C2 通信等）が検知された場合、人手を介さずに数秒以内に該当端末のネットワークアクセスを自動遮断または隔離しなければならない」という要件が提示された。

Cisco Firepower (FTD/FMC) および Cisco ISE を使用してこの要件を満たすための連携メカニズムと技術要素を 3 つ挙げなさい。

#### 解答・解説
1. **Cisco pxGrid (Platform Exchange Grid)**: FMC と ISE を安全に証明書ベースで結合し、コンテキスト情報および脅威イベントを双方向にリアルタイム共有するプロトコル。
2. **FMC Correlation Policy & Remediation**: Snort 3 IPS や Security Intelligence がハイリスクイベントを検知した際に、自動で pxGrid リクエストをトリガーするルールエンジン。
3. **ISE ANC (Adaptive Network Control) & RADIUS CoA**: pxGrid 経由で要請を受けた ISE が、即座にスイッチに対して RADIUS CoA を発行し、該当端末の SGT を Quarantine SGT に変更、または VLAN / ACL を動的変更してポートレベルで遮断するメカニズム。

---

### 質問 5 (SNMPv3 Security)

#### 問題
ISO 27001 監査員より「ネットワーク機器の SNMP 監視において、コミュニティ文字列が暗号化されていない SNMPv2c は不適合である」との指摘を受けた。IOS-XE ルータにおいて SHA-256 認証および AES-128 暗号化を使用し、かつ system MIB オブジェクト（OID: 1.3.6.1.2.1.1）のみの参照を許可する最少権限の SNMPv3 設定を作成しなさい。

#### 解答・解説
```bash
! MIB ビュー制限
snmp-server view AUDIT-VIEW system include
!
! グループ作成 (authPriv + ビュー制限 + ACL)
snmp-server group AUDIT-GROUP v3 priv read AUDIT-VIEW access 10
!
! ユーザー作成 (SHA256 認証 + AES128 暗号化)
snmp-server user audituser AUDIT-GROUP v3 auth sha AuthKeyPass2026 priv aes 128 PrivKeyPass2026
!
access-list 10 permit 192.168.10.100
```

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび Cisco Live セッション資料です。

* Cisco Technical Report: Cisco Compliance for ISO/IEC 27001:2022 Framework
* Cisco Live BRKSEC-2022: Safeguarding Network Infrastructure with ISO 27001 Technical Controls
* Cisco IOS-XE Security Configuration Guide: Securing the Control Plane (CoPP)
* Cisco IOS-XE Configuration Guide: Unicast Reverse Path Forwarding (uRPF / BCP 38)
* Cisco Security Group Access Control List (SGACL) & TrustSec Architecture Guide
* Cisco ISE & Firepower Integration via pxGrid & ANC Guide
* Cisco MACsec Configuration Guide for Catalyst Switches

---

## 📝 補足（Notes）

### コンプライアンス実装チェックリスト

1. **管理プレーンの保護 (Management Plane Protection)**
   - SSHv2 強制・Telnet 完全無効化
   - TACACS+ による全コマンド認可 (Command Authorization)
   - HTTP/HTTPS サーバー制限と TLS 1.2/1.3 強制
   - Exec Timeout および Console / VTY のローカル認証フォールバック定義
2. **データプレーン・境界保護 (Data Plane & Ingress Protection)**
   - 外部接続インターフェースでの uRPF (BCP 38) 有効化
   - L2 ポートでの DHCP Snooping、DAI、IP Source Guard、RA Guard 適用
   - 承認端末以外の MAC アドレス接続制限 (Port Security)
3. **監査とトレイル保護 (Audit & Logging)**
   - NTP SHA/MD5 認証有効化
   - Secure Syslog (TLS 6514) 転送と `logging permit-down` 設定
   - SNMPv3 authPriv (SHA/AES) と VACM ビュー制限の適用
