---
layout: default
title: 3.7.c-PCI-DSS
nav_order: 3
parent: 3.7-Compliance
grand_parent: 3.0-Security-Infrastructure
---

# 3.7.c PCI-DSS (Payment Card Industry Data Security Standard)

## 📘 概要

Payment Card Industry Data Security Standard (PCI-DSS) は、クレジットカード情報およびカード会員データ (Cardholder Data: CHD) / 敏感な認証データ (Sensitive Authentication Data: SAD) を取り扱う事業者、イシュア、アクワイアラ、サービスプロバイダー、および加盟店が遵守すべきグローバルなセキュリティ基準です。PCI Security Standards Council (PCI SSC) によって策定・管理されており、最新の主要バージョンである PCI-DSS v4.0（および従来基準 v3.2.1）では、技術的・運用的な12の主要要件 (12 Principal Requirements) が定義されています。

CCIE Security v6.1 試験における PCI-DSS (3.7.c) の位置づけは、単なる規律・合規チェックリストの暗記にとどまりません。Cisco セキュリティインフラストラクチャ（Cisco Secure Firewall / FTD / FMC、Cisco ASA、Cisco ISE、Cisco Secure Endpoint / AMP、Cisco WSA/ESA、Cisco IOS-XE スイッチ・ルータ、Cisco TrustSec / MACsec、Cisco Secure Network Analytics / Stealthwatch）を組み合わせ、カード会員データ環境 (Cardholder Data Environment: CDE) を非CDEネットワークから厳格に隔離（セグメンテーション）・保護・監視・監査するための「技術的実装手法とトラブルシューティング」が問われます。

### 主な利用目的と導入背景
* **カード会員データ環境 (CDE) の孤立化とスコープ縮小**: マイクロセグメンテーション、ファイアウォールルール、Private VLAN、VACL、VRF-Lite、および Cisco TrustSec (SGT) を活用してCDEの対象範囲（スコープ）を限定し、監査・運用コストを大幅に削減。
* **多層防御 (Defense in Depth) によるCDEの保護**: 次世代ファイアウォール (NGFW / FTD) によるL3-L7インスペクション、IPS / Snort3、マルウェア防御 (AMP/Secure Endpoint)、URLフィルタリングによるCDE端末の直接インターネットワーク通信の遮断。
* **厳格なアイデンティティ管理とアクセス制御**: Cisco ISE による802.1X/MAB、TACACS+ Administrative Access (RBAC)、多要素認証 (MFA) 連携、および特権セッションの最小権限原則の徹底。
* **データ転送時・保存時の強力な暗号化**: MACsec (802.1AE)、IPsec VPN (FlexVPN, GETVPN, Site-to-Site)、TLS 1.2/1.3 の強固な暗号スイート（Diffie-Hellman Group 14以上、AES-256-GCM 等）による伝送路保護。
* **監査ログの集約と時刻同期**: NTP (HMAC-SHA256認証) による高精度な時刻同期と、Secure Syslog (TLS/TCP 6514) / eStreamer による改ざん不可能な統合ログ管理。

---

## 🔑 要点

PCI-DSS要件とCiscoセキュリティソリューションとの統合マッピング、および設計・運用上の注意点を下表に示します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | クレジットカード業界における12の主要セキュリティ要件。Ciscoインフラ技術を用いてCDEのセグメンテーション、アクセス制御、暗号化、ログ監査を技術的に具現化する。 |
| **用途** | 決済システム、POS端末、ECサイト、バックエンドDB、カード決済トークナイゼーション基盤のセキュリティ設計・構築・運用およびCCIEラボ試験での要件準拠設定。 |
| **メリット** | CDEスコープの縮小による監査負担の軽減、未知・既知の脅威に対する多層防御の確立、不正アクセスおよびデータ漏洩の防止、法的・合規リスクの低減。 |
| **デメリット** | 暗号化・解読処理（SSL/TLS Inspection）やログ高頻度転送によるデバイス負荷増大、厳格なACL/Policyによる運用複雑化、トラブルシューティング時の多角的な検証の必要性。 |
| **対応機種** | Cisco Firepower Threat Defense (FTD) / FMC / FDM, Cisco ASA, Cisco ISE, Catalyst 9000 シリーズ (IOS-XE), Cisco WSA / ESA, Cisco Stealthwatch / Secure Network Analytics |
| **制限事項** | 非暗号化プロトコル（Telnet, HTTP, SNMPv1/v2c, FTP, Early TLS 1.0/1.1）の完全使用禁止。weak ciphers（DES, 3DES, RC4, MD5）の排除が必須。 |
| **設計上の注意点** | CDEと非CDE（Corporate / Guest / Internet）を明示的に分離すること。暗号鍵管理（KMS/Key Rotation）、NTP同期、Syslog over TLS、RBAC/MFAの漏れがないようコンフィグを徹底すること。 |


---

## 🏗 動作原理

PCI-DSS 準拠の CDE アーキテクチャは、「境界防衛」「アイデンティティ制御」「データ暗号化」「連続的監視」の4つの柱に基づいて動作します。

```
[ External / Internet / Untrusted ]
               │
               ▼ (Inbound TLS 1.2/1.3 Only)
┌──────────────────────────────────────────────┐
│  Cisco FTD / NGFW (Requirement 1 & 6)       │
│  - L3/L4 Access Control                      │
│  - Snort3 IPS Inspection                     │
│  - Malware Defense (AMP)                     │
└──────────────────────┬───────────────────────┘
                       │
                       ▼ (DMZ: Web/App Tier)
┌──────────────────────────────────────────────┐
│  DMZ Segment (Requirement 1.3)               │
│  - Direct DB connections forbidden           │
│  - Web Application Proxy / WAF               │
└──────────────────────┬───────────────────────┘
                       │
                       ▼ (Deep Packet Inspection / SGT Enforcement)
┌──────────────────────────────────────────────┐
│  Internal Core / ISE / TrustSec (Req 1 & 7)  │
│  - SGT = 2 (CDE_Server), SGT = 10 (Non_CDE)  │
│  - SGACL Enforcement: Deny Non_CDE to CDE    │
│  - Cisco ISE 802.1X / TACACS+ MFA            │
└──────────────────────┬───────────────────────┘
                       │
                       ▼ (Encrypted Channel: MACsec / IPsec)
┌──────────────────────────────────────────────┐
│  Cardholder Data Environment (CDE)           │
│  - Database / POS Processing                 │
│  - Strict Isolation via PVLAN / VACL          │
│  - Local Data Encryption (Req 3)             │
└──────────────────────────────────────────────┘
                       │
                       ▼ (Audit & Monitoring: TCP 6514 / eStreamer)
┌──────────────────────────────────────────────┐
│  Central Logging & SIEM (Requirement 10)     │
│  - Secure Syslog (TLS)                       │
│  - NTP Server with Authentication            │
└──────────────────────────────────────────────┘
```

### PCI-DSS 12大要件と Cisco 技術アーキテクチャの対比原理

1. **要件 1: ネットワークセキュリティ構造の導入と維持 (Build and Maintain a Secure Network)**
   * FTD / ASA によるゾーンベース・ファイアウォール（Inside, DMZ, Outside, CDE）。
   * DMZの配置により、インターネットからCDE内データベースへの直接アクセスを遮断。
   * 入出力トラフィックに対する明示的な `permit` ルールと末尾の暗黙ドロップ (`deny ip any any`)。
2. **要件 2: すべてのシステムコンポーネントへの安全な設定の適用 (Apply Secure Configurations)**
   * デフォルトパスワードの変更、未使用サービス（CDP, HTTP, Finger, Source Routing, IP Directed-Broadcast）の無効化。
   * 管理アクセスの暗号化（SSHv2, HTTPS / TLS 1.2/1.3, SNMPv3 AES-128/256）。
3. **要件 3: 保存されたアカウントデータの保護 (Protect Stored Account Data)**
   * カード会員データ (PAN) の不可逆的暗号化およびトークナイゼーション。
   * Cisco Key Management 連携およびデバイスストレージの暗号化。
4. **要件 4: 開かれたパブリックネットワーク上での送信時におけるアカウントデータの保護 (Protect Cardholder Data in Transit)**
   * パブリック/非信頼ネットワークを経由する通信の暗号化。
   * FlexVPN / GETVPN / Site-to-Site IPsec (AES-GCM, DH Group 14/19/20/21) および MACsec (802.1AE) によるL2/L3暗号化。
5. **要件 5: すべてのシステムおよびソフトウェアの悪意のあるソフトウェアからの保護 (Protect Systems from Malware)**
   * Cisco Secure Endpoint (旧 AMP for Endpoints) および FTD File Policy によるリアルタイム・マルウェア検出および自動隔離。
6. **要件 6: 安全なシステムおよびソフトウェアの開発と維持 (Develop and Maintain Secure Systems)**
   * FTD Snort3 IPS による仮想パッチ適用（シグネチャ自動更新）。
   * Web アプリケーションファイアウォール (WAF) 機能および不審なパケット構造の検出。
7. **要件 7: 業務上の必要性に応じたシステムコンポーネントおよびデータへのアクセスの制限 (Restrict Access to System Data by Business Need-to-Know)**
   * RBAC (Role-Based Access Control) および TrustSec SGT/SGACL による属性ベースのアクセス制御。
8. **要件 8: ユーザーの識別とシステムコンポーネントへのアクセスの認証 (Identify Users and Authenticate Access)**
   * Cisco ISE による 802.1X, MAB, RADIUS, TACACS+。
   * パスワードポリシー強化（複雑さ、最小8文字以上/v4.0では12文字推奨、90日変更、過去4回再利用禁止）。
   * 多要素認証 (MFA: Duo 連携) による管理者およびCDEアクセス端末の認証。
9. **要件 9: アカウントデータへの物理アクセスの制限 (Restrict Physical Access to Data)**
   * スイッチポートセキュリティ（Port Security, MAB, 802.1X）による未承認物理デバイスの接続遮断。
10. **要件 10: ネットワークリソースおよびアカウントデータへのすべてのアクセスの記録と統合 (Log and Monitor All Access)**
    * NTP (NTPv4 / HMAC-SHA256) による全インフラデバイスの時刻同期。
    * Secure Syslog (TLS/TCP 6514) および eStreamer (TCP 8302) による監査ログの改ざん防止・中央集約。
11. **要件 11: システムおよびネットワークのセキュリティの定期的なテスト (Test Security Systems and Processes)**
    * NetFlow / NSEL / IPFIX および Cisco Secure Network Analytics (Stealthwatch) による異常トラフィック・ポートスキャン検出。
12. **要件 12: 情報セキュリティ対応のポリシーおよびプログラムの維持 (Maintain Information Security Policies)**
    * 統合セキュリティポリシーの設計と定期的なコンプライアンス監査。

---

## ⚙ 動作シーケンス

PCI-DSS 準拠ネットワークにおける CDE 通信要求および管理者アクセス処理のシーケンスは以下のようになります。

### CDE Web/DB 多層アクセス処理シーケンス

```
[ External Client ]       [ FTD / NGFW ]         [ DMZ Web Server ]      [ CDE DB Server ]
        │                       │                        │                       │
        │─── 1. HTTPS (TLS1.3) ─►│                        │                       │
        │                       │─── 2. Access Control ──►│                       │
        │                       │    - IPS/Snort Check   │                       │
        │                       │    - Malware Check     │                       │
        │                       │─── 3. Permit to DMZ ──►│                       │
        │                       │                        │─── 4. Query (SQL) ───►│
        │                       │◄───────────────────────┼─── 5. FTD Check ──────│
        │                       │    - Strict SGT/VACL   │                       │
        │                       │    - No direct External│                       │
        │                       │─── 6. Permit DB Query ────────────────────────►│
        │                       │                        │◄── 7. Encrypted Data ─│
        │◄── 8. TLS Response ───│◄── Response ───────────│                       │
```

1. **外部クライアント要求の受信**: External クライアントは、TLS 1.2/1.3（強固な暗号スイート限定）を用いて FTD の Ingress インターフェイスにアクセス。
2. **境界 NGFW インスペクション**: FTD は Access Control Policy (ACP) により L3/L4 アドレスチェック、Snort3 IPS ルール検証、および AMP ファイル解析を実施。
3. **DMZ Web サーバーへの転送**: 正常なトラフィックのみが DMZ 上の Web/App サーバーへ転送される。インターネットから CDE 内部 DB への直接アクセスは ACP およびルーティングで完全に遮断。
4. **DMZ から CDE へのクエリ要求**: DMZ Web サーバーが CDE データベースに対して通信を試みる。
5. **内部セグメンテーション制御 (FTD / TrustSec)**: 内部 FTD または Catalyst スイッチは、送信元 SGT (DMZ) と宛先 SGT (CDE_DB) を照合。SQL ポート (TCP 1433/1521/5432 等) のみ許可し、他は即座にドロップ。
6. **CDE DB の応答と監査生成**: CDE DB が応答を返すと同時に、全通信アクティビティが Syslog (TLS/TCP 6514) 経由で SIEM へ即座に送信される。

### 管理者アクセスおよび MFA 認証シーケンス (Requirement 8)

```
[ Admin Workstation ]    [ Cisco Switch/FTD ]       [ Cisco ISE ]          [ Duo MFA Cloud ]
          │                       │                      │                        │
          │─── 1. SSHv2 Request ─►│                      │                        │
          │                       │─── 2. TACACS+ Req ──►│                        │
          │                       │    (User/Pass)       │─── 3. RADIUS/REST ────►│
          │                       │                      │    (MFA Push Req)      │
          │                       │                      │◄── 4. MFA Approved ────│
          │                       │◄── 5. TACACS+ Pass ──│                        │
          │                       │    (Privilege 15)    │                        │
          │◄── 6. SSH Granted ────│                      │                        │
          │                       │                      │                        │
          │─── 7. Exec Command ──►│                      │                        │
          │    (e.g., "conf t")   │─── 8. TACACS+ Cmd ──►│                        │
          │                       │    Authorization     │                        │
          │                       │◄── 9. Cmd Permitted ─│                        │
```

1. **SSHv2 接続確立**: 管理者が SSHv2（AES-256-CTR, SHA-2）を用いてネットワークデバイスに接続。
2. **AAA TACACS+ 認証要領**: デバイスは資格情報を Cisco ISE へ送信。
3. **ISE と MFA (Duo) 連携**: ISE は基本認証確認後、Duo API 経由で管理者のモバイル端末に Push 承認要求を送信。
4. **MFA 承認**: 管理者がモバイル端末で承認後、ISE はデバイスに TACACS+ 認証成功 (PASS) と権限レベル (Privilege 15) を返却。
5. **コマンド承認 (Accounting & Authorization)**: 管理者がコマンドを実行するごとに、ISE がリアルタイムでコマンド認可を行い、全ての入力コマンドを改ざん不可能なログとして保存（Requirement 10 準拠）。

---


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、「PCI-DSS要件に準拠したネットワークの設計・構築・トラブルシューティング」が出題された場合、単に機能単体（例: FTDのACL設定やISEのTACACS+設定）を設定するだけでなく、**「PCI-DSSのどの要件をみたすために、どの組み合わせ技術を設定しているか」**を正確に理解して解く必要があります。

### Blueprintで重要なポイント

* **要件 1 & 7 (CDE セグメンテーション & アクセス制御)**:
  * FTD / ASA におけるゾーン間ポリシー設定（DMZからCDEへは最小限のTCPポートのみ許可）。
  * TrustSec (SGT/SGACL) または Dynamic VACL / PVLAN による CDE 内・CDE 間通信の制御。
  * 暗黙のドロップ (`deny ip any any log`) の徹底とロギングの有効化。
* **要件 2 (安全なシステム設定 & プロトコル無効化)**:
  * 平文プロトコルの無効化（`no ip http server`, `no cdp run` / `no cdp enable`, SSHv2 の強固な暗号・MAC アルゴリズム指定）。
  * Control Plane Policing (CoPP) および Management Plane Protection (MPP) による管理面保護。
* **要件 4 (送信時暗号化)**:
  * Site-to-Site IPsec / FlexVPN における AES-256-GCM / SHA-384 / DH Group 19/20/21 (ECDH) の使用。
  * Switch 間通信における MACsec (802.1AE) MKA 鍵交換の設定。
* **要件 8 (厳格な AAA & MFA)**:
  * Cisco ISE における TACACS+ コマンド認可（Command Authorization）と全コマンドのアカウンティング設定。
  * 管理者パスワードポリシー（最小桁数、複雑さ、有効期限）の適用。
* **要件 10 (完全な監査ロギング & 高精度時刻同期)**:
  * 全デバイスにおける NTP 認証（NTPv4 HMAC-SHA256 または MD5 認証キー）の設定。
  * Syslog over TLS (TCP 6514) による信頼性の高い暗号化ログ送信。
  * FTD / FMC における eStreamer / NetFlow (NSEL) の有効化。

---

### ラボ試験で設定させられそうな内容

1. **CDE 隔離ネットワークの構築 (FTD ACP + TrustSec SGT)**:
   * 「CDE サーバー群 (SGT=2) は、DMZ アプリケーションサーバー (SGT=10) からの TCP 1433 (MS-SQL) のみ許可し、それ以外の非 CDE ネットワークからの通信を遮断せよ。また、遮断された通信はログを生成すること。」
2. **管理アクセスの強化 (IOS-XE / ASA / FTD)**:
   * 「すべてのネットワーク機器において、Telnet/HTTP/SNMPv1/v2c を無効化し、SSHv2 (AES-256-CTR, HMAC-SHA2-256) および SNMPv3 (AuthNoPriv ではなく AuthPriv: SHA-256 / AES-128 以上) のみを許可せよ。管理アクセスは Out-of-Band (OOB) VRF `Management` からのみ許可すること。」
3. **安全な時刻同期と暗号化ログ転送 (NTP + Secure Syslog)**:
   * 「全ルータ・スイッチにおいて、NTP サーバー 192.168.10.100 と HMAC-SHA256 キー `PCI_NTP_KEY` を用いて同期させよ。また、Syslog ログを TLS (TCP 6514) 経由で SIEM サーバー 192.168.10.200 へ送信するよう設定せよ。」
4. **TACACS+ RBAC & コマンド認可 (Cisco ISE + IOS-XE)**:
   * 「ネットワーク管理者グループ `CDE-Admins` にのみ Full Privilege 15 を与え、入力された全コマンドを ISE で個別に認可・ログ記録せよ。」

---

### よくある設定ミス

* **NTP 認証の不整合**: `ntp authenticate` や `ntp trusted-key` の設定を漏らし、NTP サーバーと同期しない（`show ntp status` で `unsynchronized` のままになる）。
* **Syslog TLS 証明書エラー**: 認証局 (CA) トラストポイントの不整合や Common Name (CN) の不一致により、Syslog over TLS セッションが確立しない。
* **暗黙のドロップログのログ漏れ**: PCI-DSS 要件では拒否されたトラフィックの記録が必須であるため、ACL の末尾に `deny ip any any log` を明示的に記載し忘れる。
* **Weak Cipher の残存**: SSH や IPsec、TLS の設定で `3DES` や `DES`、`SHA-1`、`DH Group 2` などを有効にしたままにし、PCI-DSS 基準違反となる。
* **CoPP / MPP の設定漏れ**: インターフェイス個別にはアクセス制限をかけていても、Control Plane への直接攻撃に対する保護（CoPP）が適用されていない。

---

### 試験で狙われやすい制限事項

* **FMC / FTD の管理ポートにおける Syslog 設定制限**: FMC からの eStreamer / Syslog 転送時のソース IP アドレス指定（Management インターフェイス vs Data インターフェイス）。
* **Switch の MACsec サポート制限**: 物理ポートおよび SFP モジュールが MACsec (IEEE 802.1AE) ハードウェア暗号化エンジンをサポートしているかどうかの確認。
* **SNMPv3 グループおよびビュー制限**: VACM (View-based Access Control Model) で PCI 準拠に必要な MIB 枝（`iso` 全体または特定コンポーネント）を正しく許可しているか。

---

### コンフィグの読み取り問題対策

以下のようなコンフィグが提示された場合、PCI-DSS 上の問題点を即座に指摘できるように準備してください。

```bash
! 【問題のあるコンフィグ例】
ip http server
ip http secure-server
snmp-server community PUBLIC RO
ntp server 192.168.1.1
line vty 0 4
 transport input telnet ssh
 login local
```

* **判定・修正ポイント**:
  1. `ip http server`: 平文 HTTP サーバーが有効化されている（PCI-DSS 要件 2 違反） ──► `no ip http server` で無効化。
  2. `snmp-server community PUBLIC RO`: SNMPv1/v2c 平文コミュニティ名が設定されている ──► `no snmp-server` で削除し、SNMPv3 AuthPriv を設定。
  3. `ntp server 192.168.1.1`: NTP 認証キーが設定されていない ──► `ntp authenticate` および `ntp authentication-key` を適用。
  4. `transport input telnet ssh`: Telnet アクセスが許可されている ──► `transport input ssh` のみに限定。

---

### show コマンドから状態を判断する問題

#### 1. `show ntp status` による時刻同期状態の検証

```text
Router# show ntp status
Clock is synchronized, stratum 2, reference is 192.168.10.100
nominal freq is 250.0000 Hz, actual freq is 250.0001 Hz, precision is 2**21
ntp uptime is 1234500 (100ths of secs), resolution is 4000
reference time is E8D2A1B0.12345678 (08:15:30.123 UTC Mon Nov 2 2026)
clock offset is 0.1234 msec, root delay is 1.23 msec
root dispersion is 2.34 msec, peer dispersion is 0.12 msec
loopfilter state is 'CTRL' (Normal Filter Code), drift is 0.000000123 s/s
system poll interval is 64, last update was 32 sec ago.
```
* **分析**: `Clock is synchronized` であり、ストラタム値および同期先 IP (`192.168.10.100`) が正しく表示されているため、要件 10.4（正確な時刻同期）を満たしている。

#### 2. `show logging` による Secure Syslog 接続状態の検証

```text
Router# show logging
Syslog logging: enabled (0 messages dropped, 0 messages discarded, 0 sample messages)
    Console logging: disabled
    Monitor logging: level debugging, 0 messages logged, xml disabled
    Buffer logging: level debugging, 120 messages logged, xml disabled
    Logging Exception size (4096 bytes)
    Count and timestamp logging messages: enabled
    Persistent logging: disabled
    Trap logging: level informational, 150 messages logged
        Logging to 192.168.10.200 (tcp vrf MGMT port 6514)
        Logging TLS profile: SYS_TLS_PROFILE
        TLS connection status: CONNECTED
        Messages logged: 150, Messages dropped: 0
```
* **分析**: `Logging to 192.168.10.200 (tcp vrf MGMT port 6514)` であり、`TLS connection status: CONNECTED` であるため、要件 10.5 (Secure Syslog over TLS) が正常動作している。

---

### debug ログの読み取り問題対策

#### NTP 認証失敗の debug ログ例

```text
*Nov  2 08:20:15.123: NTP: packet from 192.168.10.100 failed authentication
*Nov  2 08:20:15.124: NTP: key_id 1 not found or key mismatch
```
* **分析**: NTP サーバーから送信されたパケットの Key ID または認証パスワードがクライアント（ルータ）側の `ntp authentication-key` と一致していない。要件 10 準拠のため鍵設定を見直す必要がある。

#### MACsec MKA セッション失敗の debug ログ例

```text
*Nov  2 08:22:00.456: MKA-ERROR: MKA ICK/CAK key generation failed, PSK mismatch on interface GigabitEthernet1/0/1
*Nov  2 08:22:00.457: MKA-EVENT: Session creation failed for peer 0011.2233.4455
```
* **分析**: MACsec MKA (MACsec Key Agreement) 共有鍵 (CAK/CKN) が対向スイッチと不一致。物理リンクレベルの要件 4 (Transmission Encryption) 違反が発生している。

---


## 🛠 設定方法

PCI-DSS コンプライアンス達成のための各種 Cisco デバイス（Cisco IOS-XE, Cisco ASA, Cisco FTD/FMC）における設定例を提示します。

### 1. Cisco IOS-XE デバイスにおける PCI-DSS 硬化設定 (Hardening)

```bash
! --- 1. デフォルトアカウント・不要プロトコルの無効化 (Req 2) ---
no ip http server
no ip http secure-server
no cdp run
no ip source-route
no ip directed-broadcast

! --- 2. 厳格な SSHv2 設定 (Req 2 & Req 4) ---
ip domain name pci-domain.local
crypto key generate rsa modulus 3072 label PCI_SSH_KEY
ip ssh version 2
ip ssh dh min size 2048
ip ssh algorithm encryption aes256-ctr aes192-ctr
ip ssh algorithm mac hmac-sha2-256 hmac-sha2-512
ip ssh time-out 60
ip ssh authentication-retries 3

! --- 3. 信頼できる NTP 認証の設定 (Req 10.4) ---
ntp authenticate
ntp authentication-key 1 sha256 Cde_Ntp_P@ssw0rd_2026
ntp trusted-key 1
ntp server vrf MGMT 192.168.10.100 key 1 prefer

! --- 4. Secure Syslog over TLS (TCP 6514) の設定 (Req 10.5) ---
crypto pki trustpoint CA_SYS_LOG
 enrollment pkcs12
 revocation-check crl
!
crypto pki profile profile-tls SYS_TLS_PROFILE
 trustpoint CA_SYS_LOG
!
service timestamps log datetime msec show-timezone localtime
logging host 192.168.10.200 transport tcp port 6514 tls profile SYS_TLS_PROFILE vrf MGMT
logging trap informational
logging facility local7

! --- 5. 管理面アクセスの制御 (CoPP / VTY) (Req 1 & Req 2) ---
line vty 0 15
 exec-timeout 10 0
 transport input ssh
 access-class ADMIN_NET_ACL in
```

---

### 2. Cisco FTD / FMC における CDE セグメンテーション & 暗号化ポリシー

FMC (Firepower Management Center) Web GUI における設定手順:

1. **Access Control Policy (ACP) の設定 (Req 1 & Req 7)**:
   * **Rule 1 (Allow DMZ to CDE DB)**:
     * Source Zone: `DMZ_Zone` / SGT: `DMZ_App_Server`
     * Destination Zone: `CDE_Zone` / SGT: `CDE_DB_Server`
     * Port: `TCP 1433 (MS-SQL)`
     * Action: `Allow`
     * Inspection: `Snort3 IPS Policy (Connectivity Over Security or Balanced)` + `File Policy (Block Malware)`
     * Logging: `Log at End of Connection` -> `Send to FMC & Syslog Server`
   * **Rule 2 (Implicit Deny CDE)**:
     * Destination Zone: `CDE_Zone`
     * Action: `Block with reset`
     * Logging: `Log at Start of Connection`
2. **Platform Settings (Req 2 & Req 10)**:
   * **NTP**: NTP Server `192.168.10.100` と SHA-256 認証キーを設定。
   * **Syslog**: Secure Syslog (TLS) を有効化し、Syslog サーバー `192.168.10.200:6514` を指定。
   * **SSH**: 許可クライアントサブネットを `10.254.1.0/24` (MGMT_NET) のみに制限。

---

## 🔍 検証コマンド

PCI-DSS 要件の動作状態を確認するための主要コマンド一覧を下表にまとめます。

| 目的 | コマンド | 確認・判断ポイント |
| --- | --- | --- |
| **NTP 同期検証** | `show ntp status` | `Clock is synchronized` および正しく Stratum 値が表示されるか |
| **NTP アソシエーション確認** | `show ntp associations` | `*` (配置同期先) が付き、`refid` が正しく表示されているか |
| **Secure Syslog 接続状態** | `show logging` | `TLS connection status: CONNECTED` およびメッセージ送信カウントの確認 |
| **SSH 状態および暗号スイート確認** | `show ip ssh` | SSHv2 が有効化され、弱い暗号化アルゴリズムが除外されているか |
| **MACsec 接続状態確認** | `show macsec summary` | ポート上で MACsec 暗号化（AES-GCM-128/256）が `Active` か |
| **TrustSec SGT 制御確認** | `show cts role-based permissions` | 送信元 SGT と宛先 SGT 間の SGACL 許可・拒否マトリクス |
| **FMC eStreamer 接続状態** | `show eStreamer status` (FMC CLI) | SIEM / eStreamer クライアントと TCP 8302 セッションが確立しているか |
| **CoPP ドロップ統計** | `show policy-map control-plane` | 不正な管理プレーン通信が CoPP でドロップされているか |
| **NTP / AAA デバッグ** | `debug ntp packet` / `debug aaa authentication` | 認証エラー、パケット不一致ログのリアルタイム追跡 |

---

## 🚨 トラブルシュート

PCI-DSS 関連の設定・運用時に発生しやすい障害、原因、確認コマンド、および対処方法を表形式で整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **NTP 同期が確立せず `unsynchronized` のまま** | NTP 認証キー不一致、UDP 123 ポートが FTD/ACL で遮断、VRF 指定漏れ | `show ntp status`<br><code>show ntp associations</code><br><code>debug ntp packet</code> | 認証キー（SHA256/MD5）を再確認し、`ntp server vrf <VRF>` 設定およびルータ/FW 間での UDP 123 通信を許可 |
| **Secure Syslog (TLS) セッションが開かない** | CA 証明書未登録、PKI トラストポイント不整合、ポート 6514 閉鎖 | `show logging`<br><code>show crypto pki trustpoints</code> | PKI Trustpoint に正しい CA 証明書を組み込み、両エンドで TLS 1.2/1.3 暗号スイートが一致しているか確認 |
| **CDE 通信が意図せず拒否される** | SGACL または FTD ACP の SGT マッピング不一致、暗黙のドロップに到達 | `show cts environment-data`<br><code>show logging</code> (FTD/FMC) | ISE から SGT IP-to-SGT バインディングが最新でスイッチ/FTD にダウンロードされているか確認 |
| **SSH 接続が「No matching cipher」で拒否される** | クライアントとルータ間で許可された SSH 暗号アルゴリズムの不一致 | `show ip ssh`<br><code>debug ip ssh</code> | `ip ssh algorithm encryption` にクライアント側がサポートする強固な暗号（AES-256-CTR等）を追加 |
| **MACsec MKA セッションが Up しない** | Pre-Shared Key (CAK/CKN) の不一致、MACsec 非対応ポート/SFP | `show mka sessions`<br><code>show macsec summary</code> | スイッチ両端で CKN/CAK 文字列・長さを一致させ、インターフェイス機能を確認 |

---

## ⚠ 制限事項

* **非標準プロトコルの禁止**: HTTP (80), Telnet (23), FTP (21), SNMPv1/v2c, RSH/RLOGIN などの平文プロトコルは CDE 内および管理ネットワーク上で一切使用できません。
* **弱い暗号方式の禁止**: DES, 3DES, RC4, MD5, SHA-1, DH Group 1/2/5, TLS 1.0/1.1 は PCI-DSS 監査で即座に非準拠と判定されます（AES-128/256, SHA-256/512, DH Group 14 以上が必須）。
* **パスワード変更期間**: 管理者アカウントおよびローカルアカウントのパスワードは最長 90 日ごとの変更が強制されます（v4.0 では認証要素の多角化・MFA 必須化が強調）。

---

## 🔄 他技術との関連

* **Cisco TrustSec (SGT / SGACL)**: タグベースのマイクロセグメンテーションを実現し、物理トポロジに変更を加えずに CDE を独立分離。
* **Cisco ISE (Identity Services Engine)**: 802.1X/MAB による CDE 端末のデバイス認証、TACACS+ による管理者 RBAC & コマンド認可、および Duo MFA 統合。
* **Cisco Firepower / Snort3 (NGFW/NGIPS)**: 要件 1 (Firewall), 要件 5 (Malware Defense), 要件 6 (IPS), 要件 10 (Logging) を網羅する包括的中核コンポーネント。
* **MACsec (IEEE 802.1AE)**: レイヤ 2 イーサネットフレーム全体（MAC ヘッダー除く）を暗号化し、スイッチ間幹線の物理的盗聴を完全に防止。

---

## 🧩 比較表

### PCI-DSS v3.2.1 vs PCI-DSS v4.0 主な技術的差異

| 項目 | PCI-DSS v3.2.1 | PCI-DSS v4.0 | CCIE ラボ試験における影響 |
| --- | --- | --- | --- |
| **管理用パスワード要件** | 最小 7 文字以上 | 最小 12 文字以上（英大文字・小文字・数字・記号） | デバイスのローカルパスワード設定時の桁数条件に注意 |
| **多要素認証 (MFA)** | CDE への非ローカル管理アクセスに必須 | CDE 内の**すべての**管理アクセス・特権アクセスに MFA 必須 | Cisco ISE + Duo 連携設定の重要度増大 |
| **E-Commerce WAF要件** | 定期的な脆弱性診断または WAF | Automated Technical Solution (WAF / IPS) の常時有効化が必須 | FTD における HTTP/Web インスペクションと Snort3 必須化 |
| **暗号スイート評価** | TLS 1.1 廃止（TLS 1.2 推奨） | TLS 1.2 / TLS 1.3 強制（弱い Cipher の完全自動排除） | SSH/TLS/IPsec 暗号プロファイルの厳格化 |

---

## 💡 ベストプラクティス

1. **CDE スコープの最少化 (Scope Reduction)**: VLAN、VRF、TrustSec SGT、Private VLAN を駆使し、非 CDE コンポーネントが CDE に接触しない設計を徹底すること。CDE スコープが狭いほど監査および防御コストが激減します。
2. **監査ログの非同期ドロップ防止**: syslog 送信において、ネットワーク障害時にデバイスが通信をブロックしないよう `logging queue-limit` やフォールバックロジックを設計しつつ、TCP 6514 (TLS) で完全性を保障すること。
3. **時刻管理の単一信頼源 (Single Source of Time Truth)**: NTP 認証を有効化した冗長化 NTP サーバー構成を維持し、全セキュリティログのタイムスタンプがミリ秒精度で完全一致するように設定すること。

---


## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験レベルを意識した、PCI-DSS 要件準拠のための実践的な設定演習サンプル 11 選です。各例において問題、要件、完全な CLI/GUI 設定例、および検証方法を示します。

---

### サンプル 1: CDE ネットワーク孤立化のための FTD アクセスコントロールポリシー

#### 問題
FTD1 において、DMZ 上の Web サーバー群から CDE 内の Database サーバーへのアクセスを必要最小限の通信のみ許可し、それ以外のすべての未承認通信をブロックおよびログ生成するアクセスコントロールポリシーを構築しなさい。

#### 要件
1. DMZ Zone (`DMZ_Zone`) から CDE Zone (`CDE_Zone`) への通信は、CDE Database サーバー `10.100.1.50` に対する TCP Port `1433 (MS-SQL)` のみ許可すること。
2. 許可ルールにおいて Snort3 IPS ポリシー `Balanced Security and Connectivity` を適用し、パケットインスペクションを行うこと。
3. 許可および拒否されたすべての接続イベントを記録（ログ生成）すること。

#### 設定例 (FMC / FTD CLI & Policy)

```bash
! --- FTD CLI (確認用 Network Object / Port Object 構造) ---
! FMC GUI にて設定後、FTD にデプロイされるコンフィグ要素

object network obj-CDE-DB-Server
 host 10.100.1.50
!
object service obj-TCP-1433
 service tcp destination eq 1433
!
access-control-policy PCI_CDE_ACP
 rule 10_ALLOW_DMZ_TO_CDE_DB
  source-zone DMZ_Zone
  destination-zone CDE_Zone
  destination-network obj-CDE-DB-Server
  destination-port obj-TCP-1433
  action allow
  ips-policy Balanced_Security_and_Connectivity
  log-end
 !
 rule 999_IMPLICIT_DENY_CDE
  destination-zone CDE_Zone
  action block
  log-start
```

#### 検証方法
```bash
! FTD CLI にてアクセス制御ルールのヒット数およびコネクションを検証
FTD1# show access-list
FTD1# show conn detail
```

---

### サンプル 2: Cisco IOS-XE における Secure Syslog over TLS (TCP 6514) 設定

#### 問題
Catalyst 9300 スイッチ (SW1) において、発生するすべてのログを暗号化された Secure Syslog (TLS) 経由で SIEM サーバー (`192.168.10.200`) の TCP Port `6514` へ送信しなさい。

#### 要件
1. トラストポイント `TP_SYSLOG_CA` を作成し、証明書検証を有効化すること。
2. TLS プロファイル `SYSLOG_TLS_PROF` を定義し、トラストポイントを適用すること。
3. ログ送信は Out-of-Band 管理 VRF `MGMT` を経由し、ファシリティを `local7`、レベルを `informational` に設定すること。

#### 設定例

```bash
SW1(config)# crypto pki trustpoint TP_SYSLOG_CA
SW1(ca-trustpoint)# enrollment pkcs12
SW1(ca-trustpoint)# revocation-check crl
SW1(ca-trustpoint)# exit

SW1(config)# crypto pki profile profile-tls SYSLOG_TLS_PROF
SW1(pki-profile-tls)# trustpoint TP_SYSLOG_CA
SW1(pki-profile-tls)# exit

SW1(config)# service timestamps log datetime msec show-timezone localtime
SW1(config)# logging facility local7
SW1(config)# logging trap informational
SW1(config)# logging host 192.168.10.200 transport tcp port 6514 tls profile SYSLOG_TLS_PROF vrf MGMT
```

#### 検証方法
```bash
SW1# show logging
! "TLS connection status: CONNECTED" および送信メッセージ数が増加していることを確認
```

---

### サンプル 3: NTPv4 SHA-256 認証による高精度時刻同期

#### 問題
ルータ R1 において、監査ログの不整合を防止するため、NTP サーバー (`192.168.10.100`) との完全な NTP 認証および時刻同期を設定しなさい。

#### 要件
1. NTP 認証を有効化し、Key ID `10`、アルゴリズム `sha256`、パスワード `Ntp_Pci_Sec_S3cur3Key!` を使用すること。
2. 送信元インターフェイスを `Loopback0` に指定すること。

#### 設定例

```bash
R1(config)# ntp authenticate
R1(config)# ntp authentication-key 10 sha256 Ntp_Pci_Sec_S3cur3Key!
R1(config)# ntp trusted-key 10
R1(config)# ntp source Loopback0
R1(config)# ntp server 192.168.10.100 key 10 prefer
```

#### 検証方法
```bash
R1# show ntp status
R1# show ntp associations
```

---

### サンプル 4: SSHv2 強化 & 不要サービス無効化 (PCI-DSS Hardening)

#### 問題
ルータ R1 において、PCI-DSS 要件 2 に従い不必要な平文プロトコルを完全に削除し、暗号化アクセスプロトコルを最高レベルに強化しなさい。

#### 要件
1. HTTP, HTTPS, CDP, IP Source Routing, Direct Broadcast を無効化すること。
2. SSHv2 のみを許可し、DH グループ長を最小 2048 ビット、暗号化アルゴリズムを `aes256-ctr`、MAC アルゴリズムを `hmac-sha2-256` に限定すること。

#### 設定例

```bash
R1(config)# no ip http server
R1(config)# no ip http secure-server
R1(config)# no cdp run
R1(config)# no ip source-route
R1(config)# no ip directed-broadcast

R1(config)# ip domain name pci-audit.internal
R1(config)# crypto key generate rsa modulus 3072 label PCI_SSH_KEY
R1(config)# ip ssh version 2
R1(config)# ip ssh dh min size 2048
R1(config)# ip ssh algorithm encryption aes256-ctr
R1(config)# ip ssh algorithm mac hmac-sha2-256
R1(config)# ip ssh time-out 60
R1(config)# ip ssh authentication-retries 3

R1(config)# line vty 0 15
R1(config-line)# exec-timeout 10 0
R1(config-line)# transport input ssh
```

#### 検証方法
```bash
R1# show ip ssh
! 許可された Encryption / MAC アルゴリズムが限定されていることを確認
```

---

### サンプル 5: Cisco ISE TACACS+ による RBAC & 個別コマンド認可・監査

#### 問題
スイッチ SW1 において、すべての管理者コマンドを Cisco ISE (`192.168.10.50`) 経由で照会し、個別に認可・ログ保持（Requirement 10）を行いなさい。

#### 要件
1. TACACS+ サーバーグループ `ISE_TACACS` を作成し、暗号キー `Tacacs_Pci_Key2026` を使用すること。
2. Privilege Level 15 の全コマンドに対して認可（Authorization）を実施し、ISE から拒否されたコマンドは実行させないこと。
3. 実行されたすべてのコマンドを即座にアカウンティングログ（Accounting）として ISE に転送すること。

#### 設定例

```bash
SW1(config)# aaa new-model
SW1(config)# aaa group server tacacs+ ISE_TACACS
SW1(config-sg-tacacs+)# server-private 192.168.10.50 key Tacacs_Pci_Key2026
SW1(config-sg-tacacs+)# exit

SW1(config)# aaa authentication login default group ISE_TACACS local
SW1(config)# aaa authorization exec default group ISE_TACACS local
SW1(config)# aaa authorization commands 15 default group ISE_TACACS local
SW1(config)# aaa accounting exec default start-stop group ISE_TACACS
SW1(config)# aaa accounting commands 15 default start-stop group ISE_TACACS
```

#### 検証方法
```bash
SW1# show aaa sessions
! Cisco ISE 側 Live Logs の TACACS LPP (Live Authorization/Accounting) 画面でコマンドログを確認
```

---

### サンプル 6: 幹線の物理盗聴を防止する MACsec (IEEE 802.1AE) MKA 設定

#### 問題
スイッチ SW1 と SW2 間のトランクポート (`GigabitEthernet1/0/1`) において、CDE データが混在する物理リンク上の盗聴を防止するため、MACsec による全フレーム暗号化を設定しなさい。

#### 要件
1. MKA (MACsec Key Agreement) 共有鍵ポリシーを作成し、CKN `0123456789ABCDEF0123456789ABCDEF` および CAK `0123456789ABCDEF0123456789ABCDEF` を使用すること。
2. 暗号化アルゴリズムとして `GCM-AES-256` を指定すること。

#### 設定例

```bash
SW1(config)# mka policy MACSEC_PCI_POLICY
SW1(config-mka-policy)# cipher-suite GCM-AES-256
SW1(config-mka-policy)# exit

SW1(config)# key chain MACSEC_KEYS macsec
SW1(config-keychain)# key 0123456789ABCDEF0123456789ABCDEF
SW1(config-keychain-key)# cryptographic-algorithm AES-256
SW1(config-keychain-key)# key-string 0123456789ABCDEF0123456789ABCDEF
SW1(config-keychain-key)# exit

SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# macsec
SW1(config-if)# macsec mka policy MACSEC_PCI_POLICY
SW1(config-if)# mka pre-shared-key key-chain MACSEC_KEYS
```

#### 検証方法
```bash
SW1# show macsec summary
SW1# show mka sessions
```

---

### サンプル 7: FlexVPN Site-to-Site IPsec (AES-256-GCM / ECDH Group 19) 設定

#### 問題
ルータ R1 (本社) と R2 (CDE 拠点) 間において、パブリックネットワークを通過するカード会員データを強固に保護するため、FlexVPN IPsec トンネルを構築しなさい。

#### 要件
1. IKEv2 Proposal において `aes-gcm-256` / `prf-sha384` / `group 19 (ECP-256)` を使用すること。
2. IPsec Transform-set において `esp-gcm 256` を使用すること。

#### 設定例

```bash
! --- R1 Configuration ---
crypto ikev2 proposal PCI_IKEv2_PROP
 encryption aes-gcm-256
 prf sha384
 group 19
!
crypto ikev2 policy PCI_IKEv2_POL
 proposal PCI_IKEv2_PROP
!
crypto ikev2 keyring PCI_KEYRING
 peer R2
  address 203.0.113.2
  pre-shared-key Pci_FlexVPN_S3cur3_2026!
!
crypto ikev2 profile PCI_IKEv2_PROF
 match identity remote address 203.0.113.2 255.255.255.255
 identity local address 203.0.113.1
 authentication remote pre-share
 authentication local pre-share
 keyring local PCI_KEYRING
!
crypto ipsec transform-set PCI_TS esp-gcm 256
!
crypto ipsec profile PCI_IPSEC_PROF
 set transform-set PCI_TS
 set ikev2-profile PCI_IKEv2_PROF
!
interface Tunnel10
 ip address 172.16.1.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 203.0.113.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PCI_IPSEC_PROF
```

#### 検証方法
```bash
R1# show crypto ikev2 sa
R1# show crypto ipsec sa
```

---

### サンプル 8: Private VLAN (PVLAN) による CDE 内サーバー間アイソレーション

#### 問題
スイッチ SW1 の VLAN 100 (CDE サーバーセグメント) において、同一 VLAN 内の CDE Web サーバー同士の直接通信を遮断し、GW 兼セキュリティ機器（SVI / FW）との通信のみを許可する Private VLAN を構築しなさい。

#### 要件
1. Primary VLAN を `100`、Isolated Secondary VLAN を `101` に指定すること。
2. ポート `GigabitEthernet1/0/1` から `1/0/10` を Isolated ポートに設定し、`GigabitEthernet1/0/24` (FW 接続ポート) を Promiscuous ポートに設定すること。

#### 設定例

```bash
SW1(config)# vlan 101
SW1(config-vlan)# private-vlan community-or-isolated isolated
!
SW1(config)# vlan 100
SW1(config-vlan)# private-vlan primary
SW1(config-vlan)# private-vlan association 101

SW1(config)# interface range GigabitEthernet1/0/1 - 10
SW1(config-if-range)# switchport mode private-vlan host
SW1(config-if-range)# switchport private-vlan host-association 100 101

SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode private-vlan promiscuous
SW1(config-if)# switchport private-vlan mapping 100 101
```

#### 検証方法
```bash
SW1# show vlan private-vlan
```

---

### サンプル 9: Cisco ISE 802.1X / MAB による CDE 端末の動的 SGT 割り当て

#### 問題
Cisco ISE において、CDE 内の POS 端末および CDE 管理端末がスイッチの 802.1X ポートに接続した際、認証成功時に動的に SGT (Security Group Tag) を割り当ててネットワークを分離しなさい。

#### 要件
1. POS 端末群には `SGT 5 (POS_Devices)` を割り当てること。
2. CDE 管理端末群には `SGT 2 (CDE_Admin)` を割り当てること。
3. スイッチ側で Radius CoA (Change of Authorization) および Cisco TrustSec サポートを有効化すること。

#### 設定例 (スイッチ側設定)

```bash
SW1(config)# cts status
SW1(config)# aaa authorization network default group ISE_RADIUS
SW1(config)# radius-server attribute 6 on-for-signer
SW1(config)# radius-server attribute 8 on-for-signer
SW1(config)# radius-server attribute 25 access-accept

SW1(config)# interface GigabitEthernet1/0/5
SW1(config-if)# switchport mode access
SW1(config-if)# authentication port-control auto
SW1(config-if)# dot1x pae authenticator
SW1(config-if)# cts manual
SW1(config-if)# cts role-based enforcement
```

#### 検証方法
```bash
SW1# show cts interface GigabitEthernet1/0/5
SW1# show cts environment-data
```

---

### サンプル 10: Cisco FTD による Snort3 IPS & Malware Defense ポリシー適用

#### 問題
Cisco FTD1 において、CDE ゾーンに通過する全トラフィックに対して Snort3 インプルーブド IPS ポリシーおよび AMP ファイルポリシーを適用しなさい。

#### 要件
1. 侵入禁止ポリシー（IPS Policy）として `Security Over Connectivity` を全 CDE ルールに適用すること。
2. 実行可能ファイル (.exe, .dll, .bat) および圧縮ファイル (.zip, .rar) のファイル転送時に マルウェア検査 (Malware Cloud Lookup) および Block Malware ルールを適用すること。

#### 設定例 (FMC 内定義概念 & CLI 表示)

```bash
! FMC ACP Rule Configuration:
! Rule Name: CDE_INSPECTION_POLICY
! Src Zone: ANY -> Dst Zone: CDE_Zone
! Action: Allow
! Inspection Tab:
!   - IPS Policy: Security Over Connectivity
!   - Variable Set: Default-Set
!   - File Policy: Block_Malware_Executables
! Logging: Log at End of Connection -> Send to FMC

! FTD CLI 検証用
FTD1# show service-policy inspect ip
```

#### 検証方法
```bash
FTD1# show sftunnel status
! FMC への脅威イベント転送ステータスを確認
```

---

### サンプル 11: Cisco ASA における NSEL (NetFlow Secure Event Logging) 監査設定

#### 問題
Cisco ASA5585-X (ASA1) において、CDE インターフェイス上の全セッション発生・終了イベントを NSEL (NetFlow v9) 経由で Secure Analytics コレクター (`192.168.10.150:2055`) へ出力しなさい。

#### 要件
1. Flow コレクター `192.168.10.150` UDP Port `2055` を指定すること。
2. コネクション生成 (flow-creation) および削除 (flow-teardown) イベントを出力すること。

#### 設定例

```bash
ASA1(config)# flow-export destination inside 192.168.10.150 2055
ASA1(config)# flow-export template timeout-rate 1
ASA1(config)# flow-export delay flow-create 15

ASA1(config)# class-map NSEL_CLASS
ASA1(config-cmap)# match any

ASA1(config)# policy-map NSEL_POLICY
ASA1(config-pmap)# class NSEL_CLASS
ASA1(config-pmap-c)# flow-export event-type all destination 192.168.10.150

ASA1(config)# service-policy NSEL_POLICY global
```

#### 検証方法
```bash
ASA1# show flow-export counters
```

---


## ❓ 想定試験問題

CCIE Security ラボ・テクノロジー試験を模した、PCI-DSS に関する実践的な想定問題と詳細解説です。

---

### 問題 1 (コンフィグ読解・合規判定)

#### 【問題】
以下のスイッチコンフィグが提示されました。PCI-DSS v4.0 要件に照らし合わせて**違反している箇所を3つ**指摘し、それぞれの修正コンフィグを記述しなさい。

```bash
line vty 0 4
 transport input ssh telnet
 exec-timeout 0 0
 login local
!
ip http server
ip http path flash:/html
```

#### 【回答・解説】
1. **違反 1 (要件 2.2.7): Telnet の許可 (`transport input ssh telnet`)**
   * **理由**: Telnet は暗号化されない平文プロトコルであり、管理者認証情報が盗聴されるリスクがある。
   * **修正**: `transport input ssh`
2. **違反 2 (要件 8.2.8): セッションタイムアウトの無効化 (`exec-timeout 0 0`)**
   * **理由**: PCI-DSS では、無人端末の乗っ取りを防ぐため、15 分以下の非アクティブタイムアウト設定が義務付けられている。
   * **修正**: `exec-timeout 10 0`
3. **違反 3 (要件 2.2.1): 平文 HTTP サーバーの有効化 (`ip http server`)**
   * **理由**: HTTP 管理インターフェイスは平文通信のため禁止。
   * **修正**: `no ip http server`

---

### 問題 2 (トラブルシューティング)

#### 【問題】
管理者アカウントが Cisco ISE 経由でルータ R1 に SSH 接続しようとした際、認証は成功しますが、コマンド実行時に一切の標準コマンドが拒否（`Command authorization failed`）されてしまいます。考えられる設定原因と修正コマンドを述べなさい。

#### 【回答・解説】
* **原因**: ルータ側で `aaa authorization commands 15 default group <TACACS_GROUP>` が設定されているにもかかわらず、Cisco ISE 側の TACACS+ Command Authorization Policy において該当ユーザーグループに対するコマンドの許可ルールが設定されていない。
* **修正方法 (ルータ側での緊急確認/一時的ローカルフォールバック指定)**:
```bash
R1(config)# aaa authorization commands 15 default group ISE_TACACS local
```
Cisco ISE 側の Policy Sets 内 TACACS Authorization Rules にて、管理者グループに `Permit All Commands` Shell Profile が割り当てられているか確認・修正する。

---

### 問題 3 (Design / アーキテクチャ)

#### 【問題】
ある加盟店において、カード会員データ (CDE) を処理する DB サーバーと、社内一般ユーザーがアクセスする OA 端末群が同一の L3 サブネット (`10.20.30.0/24`) 内に混在しています。PCI-DSS 監査人より「CDE のスコープ隔離が不十分である」と指摘されました。物理的なネットワーク再構成やルータのインターフェイス追加を行わずに、最速かつ安全に CDE サーバーを論理隔離するための Cisco ソリューションを 2 つ挙げ、それぞれの特徴を説明しなさい。

#### 【回答・解説】
1. **Cisco TrustSec (SGT / SGACL)**:
   * **特徴**: 既存の L3 サブネットを変更することなく、Cisco ISE 経由で CDE サーバーに `SGT 2`、OA 端末に `SGT 10` を定義。スイッチおよび FTD 上で `SGACL` (SGT 10 -> SGT 2: Deny) を適用することで、IP アドレス非依存の高度なマイクロセグメンテーションを実現する。
2. **Private VLANs (PVLAN)**:
   * **特徴**: 同一 VLAN (`10.20.30.0/24`) 内で、CDE サーバーが接続するポートを `Isolated Port`、一般端末やゲートウェイポートを `Promiscuous Port` に設定し、同一 L2 ドメイン内の端末間直接通信を物理/論理レベルで遮断する。

---

### 問題 4 (実装・設定)

#### 【問題】
ルータ R1 において、PCI-DSS 要件 10.4 に基づき、NTP サーバー `192.168.1.1` との間で HMAC-SHA256 認証を用いた時刻同期を設定しなさい。キー ID は `5`、キー文字列は `PciNtpPass2026` とすること。

#### 【回答・解説】
```bash
R1(config)# ntp authenticate
R1(config)# ntp authentication-key 5 sha256 PciNtpPass2026
R1(config)# ntp trusted-key 5
R1(config)# ntp server 192.168.1.1 key 5 prefer
```

---

### 問題 5 (トラブルシューティング / Syslog over TLS)

#### 【問題】
スイッチ SW1 から Syslog サーバー `10.1.1.100` への Secure Syslog (TLS) 接続が確率せず、`show logging` で `TLS connection status: DOWN` と表示されています。トラブルシューティング手順として確認すべき項目を 3 つ挙げなさい。

#### 【回答・解説】
1. **PKI トラストポイントおよび CA 証明書の妥当性**: `show crypto pki trustpoints` を実行し、Syslog サーバーの証明書を署名した Root/Intermediate CA 証明書が正しくインストールされ、有効期限内であるか確認する。
2. **ポートおよび TCP/TLS 疎通性**: `test tcp 10.1.1.100 6514` 等で、途中経路（FTD/ACL）で Port `6514` がドロップされていないか確認する。
3. **証明書の Common Name (CN) / SAN 不一致**: Syslog サーバーから提示された TLS 証明書の CN/SAN が、ルータ側で指定した IP アドレスまたは FQDN と一致しているか確認する。

---


## 🔗 参考リソース

Cisco 公式ドキュメント、デザインガイド、テクニカルノート等のリソース一覧です。

* Cisco Technical White Paper: Payment Card Industry Data Security Standard (PCI-DSS) v4.0 Design Guide
* Cisco Live Presentation: Securing Cardholder Data Environments with Cisco TrustSec and Firepower
* Cisco IOS-XE Security Configuration Guide: Securing the Control Plane and Management Plane
* Cisco Secure Firewall Management Center Administration Guide 7.2: Access Control and Snort3 IPS Configuration
* Cisco ISE Admin Guide 3.2: TACACS+ Command Authorization and Duo MFA Integration
* Cisco Configuration Example: Configuring Secure Syslog (TLS) on Cisco IOS-XE Devices
* Cisco MACsec Switch-to-Switch Configuration Guide for Catalyst 9000 Switches

---

## 📝 補足（Notes）

* **PCI-DSS 適合性のための運用チェックリスト**:
  - [ ] 全デバイスで `no ip http server` / `no telnet` が徹底されているか
  - [ ] NTP が認証（SHA-256）付きで同一タイムソースに同期しているか
  - [ ] Syslog が TLS (TCP 6514) で暗号化送信され、ドロップが発生していないか
  - [ ] TACACS+ によりコマンド認可・ログ記録が全管理操作で機能しているか
  - [ ] CDE への全アクセス経路において弱小暗号（DES, 3DES, RC4, MD5, TLS 1.0/1.1）が排除されているか

---


