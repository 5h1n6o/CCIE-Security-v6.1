---
layout: default
title: 3.2.c-Securing-device-access
nav_order: 3
parent: 3.2-Management-plane
grand_parent: 3.0-Security-Infrastructure
---

# 3.2.c Securing device access

Cisco インフラストラクチャにおけるデバイスアクセス保護（Securing Device Access）は、ネットワークアプライアンス（Cisco IOS-XE ルーター/スイッチ、Cisco ASA ファイアウォール、Cisco FTD / FMC）に対する管理アクセス（Console、VTY/SSH、HTTPS/ASDM/FMC GUI）の認証・認可・監査（AAA: Authentication, Authorization, Accounting）、パスワード保護、暗号化通信、およびセッション制御を網羅するセキュリティ標準です。

CCIE Security v6.1 試験において、Securing Device Access は単に「SSHを有効化する」「ローカルユーザーを作成する」といった基礎的な設定にとどまらず、RADIUS / TACACS+ サーバ（Cisco ISE等）を組み合わせた高度な AAA 制御、ローカルフォールバック機構、特権レベル (Privilege Levels) および RBAC (Role-Based Access Control / CLI Views / Parser Views)、暗号強度を高めたパスワードハッシュアルゴリズム（Type 6 AES, Type 8 PBKDF2, Type 9 scrypt）、VTY / Console のセッション制御、ならびに ASA や FTD における管理プレーン制御の相互関係を正確に実装・検証・トラブルシューティングする能力が要求されます。

---

## 📘 概要

* **機能概要**:
  * **AAA (Authentication, Authorization, Accounting)**: TACACS+ (TCP 49) および RADIUS (UDP 1812/1813) プロトコルを用いて中央集権的に認証（ユーザー確認）、認可（実行可能コマンド・アクセス権限の決定）、監査（ログ記録）を実施。
  * **ローカル認証 ＆ フォールバック (Local Fallback)**: 外部 AAA サーバ群が応答不能（Timeout / Dead）となった際に、自動的にローカルデータベース（`username` コマンド）に切り替えて管理者のロックアウトを回避する冗長化機構。
  * **強固な暗号化（Password & Key Security）**: `service password-encryption` (Type 7 - 可逆的) の脱却、`enable algorithm-type scrypt` / `sha256` (Type 9 / Type 8)、および `password encryption aes` (Type 6) を活用した認証情報の保護。
  * **安全なセキュアトランスポート**: Telnet や HTTP などの平文プロトコルを完全に排除し、SSHv2 (RSA 2048bit+ / ECDSA) および HTTPS/TLS 1.2+ を限定許可。
  * **セッション制御とバナー**: `exec-timeout` によるアイドルセッション自動切断、ログイン警告バナー (`banner motd`, `banner exec`, `banner login`)、およびログイン試行回数制限 (`login block-for`, `aaa authentication attempt lockout`)。
* **利用目的**:
  * 不正アクセスや中間者攻撃 (MitM)、パスワード盗聴、辞書攻撃からネットワーク機器の管理プレーンを隔離・保護する。
  * 運用担当者の職務権限に応じた最小特権の原則 (Principle of Least Privilege) を CLI / GUI レベルで適用する。
* **どのような場面で利用するか**:
  * エンタープライズネットワークおよびデータセンターに配置されたすべてのインフラ機器の初期構築および運用セキュリティハードニング。
  * PCI-DSS、NIST SP 800-53、CIS Benchmarks などのセキュリティコンプライアンス要件に基づくアクセス制御の実装。

---

## 🔑 要点

| 項目 | Cisco IOS / IOS-XE Router | Cisco ASA Firewall | Cisco FTD / FMC |
| :--- | :--- | :--- | :--- |
| **主要認証プロトコル** | TACACS+ / RADIUS / Local | TACACS+ / RADIUS / Local | TACACS+ / RADIUS / Local / LDAP / SAML 2.0 |
| **コマンド認可 (Authorization)** | TACACS+ Command Authorization / Parser Views | TACACS+ Command Authorization / Command Authorization Sets | FMC User Roles (Administrator, Network Admin, Security Admin等) |
| **SSH/HTTPS アクセス制限** | `access-class` (VTY) / `ip http access-class` | `ssh <ip> <mask> <interface>` / `http <ip> <mask> <interface>` | Platform Settings > Management Access (SSH / Access List) |
| **パスワードハッシュ** | Type 9 (scrypt), Type 8 (PBKDF2-SHA256), Type 6 (AES) | Type 5 (salted MD5) / PBKDF2 (最新ASA) | Linux shadow (SHA-512) |
| **フォールバック指定** | `aaa authentication login default group TACACS_GROUP local` | `aaa authentication ssh console TACACS_GROUP LOCAL` | FMC: External Authentication + Local Default |
| **セッションタイムアウト** | `exec-timeout <min> [sec]` (Line VTY / Console) | `console timeout <min>` / `ssh timeout <min>` | FMC System Settings > Shell Timeout / Web Timeout |

---

## 🏗 動作原理

### デバイスアクセス認証・認可・監査通信フロー

```
[ Admin Workstation ]
        │
        │  1. SSH / HTTPS Connection Request
        ▼
[ Cisco Device (IOS-XE / ASA / FTD) ]
        │
        │  2. AAA Request (TACACS+ TCP 49 / RADIUS UDP 1812)
        ▼
[ Primary AAA Server (Cisco ISE) ]
        │
        ├── (A) Authentication OK ──► 3. Challenge / Pass
        ├── (B) TACACS+ Authorization ──► 4. Command Executed / Denied
        └── (C) AAA Server Down / Unreachable
                │
                ▼
      [ Fallback to Local DB ] ──► 5. Authenticate with Local 'username'
```

1. **セッション確立**: 管理端末から SSHv2 または HTTPS 経由でデバイスに接続を開始。
2. **AAA サーバ問い合わせ**: デバイスは設定された `aaa group`（優先順位順）に従い、Primary TACACS+/RADIUS サーバへアクセス要求を送信。
3. **認証判定**: サーバ側で Identity Store（Active Directory, Internal DB）と照合し、Access-Accept または Access-Reject を返答。
4. **コマンドレベル認可（TACACS+ 特有）**: ユーザーが CLI コマンドを入力する度に、デバイスは TACACS+ サーバへ Command Authorization Request を送信し、許可されたコマンドのみ実行を可視化。
5. **ローカルフォールバック**: AAA サーバ群が応答拒否（Timeout / TCP RST）ではなく「接続不能 (SERVER DEAD)」となった場合のみ、ローカルデータベース（`local` / `LOCAL`）による認証にフォールバック。

---

## ⚙ 動作シーケンス

デバイスアクセス保護におけるパケット処理および認証処理の順序は以下の通りです。

```
1. IP Access Control (Management Access List)
   └── VTY access-class / ASA 'ssh' command / FMC Platform Settings により送信元 IP アドレスをチェック。不一致なら TCP RST / Drop。

2. Transport Security Handshake
   └── SSHv2 ディフィー・ヘルマン鍵交換 / TLS Handshake。暗号化セッションを保護。

3. User Authentication Phase
   ├── AAA Server Active? ──► TACACS+/RADIUS で認証。成功したらステップ 4 へ。
   └── AAA Server Dead?   ──► Local DB ('username') で認証。

4. User Authorization & Privilege Level Assignment
   ├── AAA Server / Local DB より Privilege Level (0-15) や Executive Group、FMC Role を割り当て。
   └── TACACS+ 有効時：入力された各コマンドの実行可否を TACACS+ サーバに問い合わせ。

5. Session Monitoring & Accounting
   ├── 入力コマンドおよびセッション開始・終了ログを Accounting サーバ (TACACS+/RADIUS) および Syslog へ送信。
   └── 非アクティブ時間が exec-timeout / timeout 値に達した場合、セッションを強制切断。
```

---

## 🎯 試験対策（CCIE Securityラボ試験）

* **Blueprintで重要なポイント**:
  * TACACS+ と RADIUS の動作違いの把握（TACACS+ はヘッダー以降全暗号化・TCP 49・認証と認可の分離、RADIUS はパスワードのみ暗号化・UDP 1812/1813・認証と認可の結合）。
  * AAA サーバーグループのデッドタイム判定とローカルフォールバック構成（`aaa new-model`、`aaa group server`）。
  * SSHv2 構築要件：ホスト名 (`hostname`)、ドメイン名 (`ip domain-name`)、RSA/ECDSA 鍵生成 (`crypto key generate rsa modulus 2048`)、SSHv2 限定 (`ip ssh version 2`)、および VTY の `transport input ssh` 設定。
  * 特権レベル (Privilege Level) と Parser Views (RBAC) の構成。
  * パスワード暗号化形式の強化：`enable algorithm-type scrypt secret <pass>`、`username <user> algorithm-type scrypt secret <pass>`、`password encryption aes`。
* **ラボ試験で設定させられそうな内容**:
  * 「ISE 経由で TACACS+ 認証を行い、ISE がダウンした場合のみローカル特権ユーザーでログインできるように構成せよ。ただし RADIUS は使用不可。」
  * 「VTY ラインにおいて SSHv2 のみを許可し、特定管理サブネット（192.168.10.0/24）以外からのアクセスを無効化せよ。さらに 5 分間非アクティブの場合自動ログアウトさせよ。」
  * 「ASA において Outside および Inside インターフェイスからの SSH アクセスを制御し、TACACS+ 認証＋LOCAL フォールバックを設定せよ。」
  * 「すべてのパスワード暗号化を可視性の高い Type 7 から高強度の Type 6 (AES) または Type 9 (scrypt) に移行せよ。」
* **よくある設定ミス**:
  * `aaa authentication login default group TACACS_GROUP local` の末尾に `local` を付け忘れ、ISE がダウンした際に機器に一切アクセスできなくなる（ロックアウト）。
  * `ip domain-name` を設定せずに `crypto key generate rsa` を実行し、キー名がデフォルトになって再生成時に混乱する。
  * VTY ラインで `login local` を設定している状態で `aaa new-model` を有効化し、既存のログイン動作と AAA ポリシーが干渉する。
  * ASA で `aaa-server` に `host` や `secret-key` を指定した際、`timeout` 値が短すぎて一時的なネットワーク遅延で即座に SERVER DEAD と判定される。


## 🛠 設定方法

### 1. Cisco IOS-XE: 完全な デバイスアクセス保護 (AAA + SSHv2 + RBAC + Pass Hardening)

```bash
! --- 1. 基本識別子とドメイン設定 ---
hostname R1-Core
ip domain-name lab.ccie.local

! --- 2. 高強度暗号キー生成と SSHv2 有効化 ---
crypto key generate rsa modulus 2048 label SSH-RSA-KEY
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
ip ssh source-interface GigabitEthernet0/0

! --- 3. パスワード保護の強化 (Type 9 scrypt / Type 6 AES) ---
! Master Key の設定 (Type 6 暗号化用)
key config-key password-encrypt MySecretMasterKey123!
password encryption aes

! 特権パスワードおよびローカルフォールバックユーザー (Type 9 scrypt)
enable algorithm-type scrypt secret SecretEnable123!
username admin algorithm-type scrypt secret AdminPass123!
username admin privilege 15

! --- 4. AAA サーバグループおよびポリシー構成 ---
aaa new-model

! TACACS+ サーバ定義
tacacs server ISE-TACACS-1
 address ipv4 10.1.1.100
 key SecretTacacsKey123!
 timeout 5

tacacs server ISE-TACACS-2
 address ipv4 10.1.1.101
 key SecretTacacsKey123!
 timeout 5

! サーバグループ定義
aaa group server tacacs+ TACACS-GROUP
 server name ISE-TACACS-1
 server name ISE-TACACS-2
 ip tacacs source-interface GigabitEthernet0/0

! 認証・認可・監査ポリシー
aaa authentication login default group TACACS-GROUP local
aaa authentication login CONSOLE-KEY local
aaa authorization exec default group TACACS-GROUP local
aaa authorization commands 15 default group TACACS-GROUP local
aaa accounting exec default start-stop group TACACS-GROUP
aaa accounting commands 15 default start-stop group TACACS-GROUP

! --- 5. 管理アクセスコントロールリスト (iACL / VTY ACL) ---
ip access-list extended ACL-MGMT-ALLOW
 permit tcp 192.168.10.0 0.0.0.255 any eq 22
 deny   ip any any log-input

! --- 6. ライン制御 (Console / VTY) ---
line console 0
 exec-timeout 5 0
 login authentication CONSOLE-KEY
 logging synchronous

line vty 0 15
 exec-timeout 5 0
 access-class ACL-MGMT-ALLOW in
 transport input ssh
 transport output none
```

---

### 2. Cisco ASA Firewall: AAA 認証・SSH / HTTPS アクセス制限

```text
! --- 1. ホスト名・ドメイン・RSAキー ---
hostname ASA-FW1
domain-name lab.ccie.local
crypto key generate rsa modulus 2048

! --- 2. ローカルユーザーおよび Enable パスワード ---
enable password SecretEnable123!
username admin password AdminPass123! privilege 15

! --- 3. TACACS+ サーバーグループおよび AAA 設定 ---
aaa-server TACACS-GROUP protocol tacacs+
aaa-server TACACS-GROUP (mgmt) host 10.1.1.100
 key SecretTacacsKey123!
 timeout 5

aaa authentication ssh console TACACS-GROUP LOCAL
aaa authentication http console TACACS-GROUP LOCAL
aaa authorization command TACACS-GROUP LOCAL
aaa accounting ssh TACACS-GROUP
aaa accounting enable TACACS-GROUP

! --- 4. 管理アクセス許可 (SSH / ASDM HTTPS) ---
ssh 192.168.10.0 255.255.255.0 inside
ssh 10.1.1.0 255.255.255.0 mgmt
ssh timeout 5
ssh version 2

http server enable
http 192.168.10.0 255.255.255.0 inside
http timeout 5

console timeout 5
```

---

### 3. Cisco FTD / FMC: FMC 管理インターフェイスおよびプラットフォーム設定

1. **FMC GUI > Objects > Object Management > RADIUS / TACACS+ External Server**:
   * TACACS+ サーバ（ISE）の IP アドレス、Shared Secret、ポート 49 を追加。
2. **FMC GUI > System > Users > External Authentication**:
   * Authentication Method: `TACACS+`
   * Primary Server: 定義した ISE サーバオブジェクトを指定。
   * Shell Access Filter / Role Mapping: ISE から返される TACACS+ Attribute（例: `cisco-av-pair=shell:roles="Administrator"`）を FMC の Administrator ロールへマッピング。
   * Enable Local Fallback: チェックを入れて外部 AAA サーバ疎通不能時にローカル `admin` アカウントでのログインを保証。
3. **FMC GUI > Devices > Platform Settings**:
   * **SSH**: 許可するクライアント IP ネットワーク（例: `192.168.10.0/24`）、セッションタイムアウト（5分）を設定。
   * **HTTP Access**: 許可するネットワーク（例: `192.168.10.0/24`）を設定。

---

## 🔍 検証コマンド

| 目的 | コマンド (IOS-XE) | コマンド (ASA) |
| :--- | :--- | :--- |
| **AAA 状態・グループ確認** | <code>show aaa servers</code><br><code>show aaa sessions</code> | <code>show aaa-server</code><br><code>test aaa-server ...</code> |
| **SSH 接続状態確認** | <code>show ip ssh</code><br><code>show ssh</code> | <code>show ssh</code><br><code>show ssh sessions</code> |
| **VTY / Console 状態確認** | <code>show users</code><br><code>show line vty 0 4</code> | <code>show users</code><br><code>show console-timeout</code> |
| **暗号キー確認** | <code>show crypto key mypubkey rsa</code> | <code>show crypto key mypubkey rsa</code> |
| **デバッグ追跡** | <code>debug aaa authentication</code><br><code>debug aaa authorization</code><br><code>debug ip ssh</code> | <code>debug aaa-server authentication</code><br><code>debug ssh</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **AAA サーバ疎通不能でログイン不可** | AAA グループに `local` / `LOCAL` フォールバック指定が欠落。 | <code>show aaa servers</code> | コンソール接続または ROMMON から `no aaa new-model` で復旧し、`group <name> local` を正しく再設定。 |
| **SSH 接続時に "Connection Refused"** | RSA 鍵未生成、VTY の `transport input ssh` 欠落、または ACL で拒否。 | <code>show ip ssh</code><br><code>show access-lists</code> | `crypto key generate rsa` を実行し、VTY の `access-class` 設定を再確認。 |
| **TACACS+ コマンド認可で全コマンド失敗** | ISE 側に該当ユーザーの Command Authorization Policy が存在しない。 | <code>debug aaa authorization</code> | ISE の TACACS+ Command Sets ポリシーに許可ルールを追加するか、緊急時は `no aaa authorization commands` を設定。 |
| **パスワード解読リスク (Type 7)** | `service password-encryption` による弱い Vigenère 暗号の使用。 | <code>show running-config</code> | `enable algorithm-type scrypt secret` および `password encryption aes` を設定。 |

---

## ⚠ 制限事項

* **SSH キー長**: RSA キー長は 2048 bit 以上を必須とし、1024 bit 以下の鍵はモダンな SSH クライアント（OpenSSH 8.8+）で拒否される。
* **RADIUS によるコマンド認可の非対応**: RADIUS プロトコルはセッション単位の認証・認可（Exec Shell の許可等）のみをサポートし、入力コマンド単位の認可・制御（Command Authorization）は不可（TACACS+ が必須）。
* **ASA における Parser Views 非対応**: IOS/IOS-XE のような `parser view` による柔軟なローカル CLI ロール分離は ASA では非対応（ASA では Privilege Level 0-15 または TACACS+ 認可を使用）。

---

## 🔄 他技術との関連

* **Cisco ISE (Identity Services Engine)**: デバイスアクセスの中心となる TACACS+ (Device Admin) および RADIUS サーバ。
* **Control Plane Protection (CoPP / iACL)**: 許可された管理 IP アドレス以外の SSH/HTTPS/SNMP パケットをコントロールプレーン到達前にドロップする多層防御。
* **Syslog / SNMP Traps**: デバイスアクセス失敗、ログイン成功、コマンド実行ログの監査ログ収集基盤。

---

## 🧩 比較表

| 比較項目 | TACACS+ (Device Admin) | RADIUS (Network Access) |
| :--- | :--- | :--- |
| **トランスポート** | **TCP 49** (信頼性・再送制御あり) | **UDP 1812 (Auth), 1813 (Acct)** |
| **暗号化範囲** | **パケット全体** (ヘッダー除く全ペイロード暗号化) | **パスワード属性のみ** (他属性は平文) |
| **機能分離** | **認証 (AuthN) と認可 (AuthZ) を完全に分離** | **認証と認可が結合** (Access-Accept で属性返却) |
| **コマンド単位認可** | **対応** (CLI コマンド毎に判定可能) | **非対応** (シェルアクセス権限のみ) |
| **主な用途** | ネットワーク機器の管理アクセス保護 (Device Administration) | ネットワーク参入認証 (802.1X, Remote Access VPN) |

---

## 💡 ベストプラクティス

1. **AAA ローカルフォールバックの徹底**: 必ず `aaa authentication login default group <GROUP> local` の形式で定義し、AAA サーバ障害時の機器ロックアウトを防止する。
2. **暗号化強度の最大化**: `service password-encryption` (Type 7) を廃止し、`enable algorithm-type scrypt` (Type 9) または `password encryption aes` (Type 6) を適用する。
3. **明示的な SSH トランスポート制限**: VTY ラインに `transport input ssh` および `access-class` を適用し、不要なプロトコル (Telnet) や不審な IP からのアクセスを拒否する。
4. **Console/VTY タイムアウトの設定**: `exec-timeout 5 0` を全ラインに適用し、離席時の未操作セッション放置リスクを排除する。


## 📝 ラボ学習・設定サンプル例

### 例1: TACACS+ 認証・認可およびローカルフォールバックの設定 (IOS-XE)

* **問題**:
  外部 TACACS+ サーバ群（10.1.1.100, 10.1.1.101）を用いたログイン認証および特権 Exec 認可を構成し、サーバ疎通不能時のみローカルユーザー `admin` にフォールバックさせよ。
* **要件**:
  * サーバーグループ名: `TACACS-SERVERS`
  * Shared Secret: `CiscoLabKey123!`
  * TACACS+ サーバタイムアウト: 3秒
  * 外部サーバ疎通不能時のみローカル認証へフォールバックすること。
* **設定例**:
```bash
tacacs server ISE-1
 address ipv4 10.1.1.100
 key CiscoLabKey123!
 timeout 3

tacacs server ISE-2
 address ipv4 10.1.1.101
 key CiscoLabKey123!
 timeout 3

aaa group server tacacs+ TACACS-SERVERS
 server name ISE-1
 server name ISE-2

aaa new-model
username admin algorithm-type scrypt secret AdminLocalPass123!
username admin privilege 15

aaa authentication login default group TACACS-SERVERS local
aaa authorization exec default group TACACS-SERVERS local
```
* **検証方法**:
  `test aaa group TACACS-SERVERS user1 Pass123! legacy` を実行して TACACS+ 応答を確認。TACACS+ サーバの IP を一時的に不達のアラートアドレスに変更し、VTY ログインを試行して `admin` でログイン可能か確認。`show aaa servers` で DEAD カウントを検証。

---

### 例2: SSHv2 の強固な構築と VTY アクセス制限 (IOS-XE)

* **問題**:
  R1 に対する遠隔管理アクセスを SSHv2 に限定し、指定管理ネットワーク（192.168.100.0/24）以外からのアクセスを無効化せよ。
* **要件**:
  * ドメイン名: `ccie.lab`
  * RSA 鍵長: 2048 bit
  * SSH タイムアウト: 60秒、認証試行回数: 3回
  * 非アクティブ 10 分で自動ログアウトさせること。
* **設定例**:
```bash
hostname R1
ip domain-name ccie.lab

crypto key generate rsa modulus 2048 label SSH-KEY
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3

ip access-list extended ACL-VTY-ALLOW
 permit tcp 192.168.100.0 0.0.0.255 any eq 22
 deny   ip any any log-input

line vty 0 15
 exec-timeout 10 0
 access-class ACL-VTY-ALLOW in
 transport input ssh
 transport output none
```
* **検証方法**:
  `192.168.100.50` から SSH 接続を試行し成功することを確認。それ以外の IP（例: `10.1.1.1`）から `ssh -v 2 R1` を実行し、接続が拒否（Connection Refused / ACL Drop）されることを検証。`show ip ssh` および `show access-lists ACL-VTY-ALLOW` を実行。

---

### 例3: Type 6 (AES) および Type 9 (scrypt) パスワード暗号化の適用 (IOS-XE)

* **問題**:
  IOS-XE の設定ファイル内に保存されるすべてのプレーンテキストおよび Type 7 パスワードを廃止し、Type 6 (AES) および Type 9 (scrypt) にアップグレードせよ。
* **要件**:
  * マスターキーを設定し、`password` コマンドで指定されたパスワードを Type 6 で暗号化すること。
  * `enable` および `username` パスワードには Type 9 (scrypt) を使用すること。
* **設定例**:
```bash
key config-key password-encrypt MasterSecurityKey2026!
password encryption aes

enable algorithm-type scrypt secret EnableScryptSecret123!
username secadmin algorithm-type scrypt secret SecAdminPass123!
```
* **検証方法**:
  `show running-config | include (username|enable|password)` を実行し、ユーザーパスワードが `algorithm-type scrypt` または `6` で暗号化されていることを確認。Type 7（`7` で始まる文字列）が存在しないことを検証。

---

### 例4: Parser Views (RBAC) による限定的 CLI ビューの作成 (IOS-XE)

* **問題**:
  オペレーター用として、インターフェイスの状態確認（`show ip interface brief`）および `ping` コマンドのみが実行可能な限定 CLI ビュー `MONITOR-VIEW` を作成せよ。
* **要件**:
  * ビュー名: `MONITOR-VIEW`
  * ビューパスワード: `ViewPass123!`
  * 許可コマンド: `show ip interface brief`, `ping`
* **設定例**:
```bash
enable view
! 最初に enable secret が必要
enable algorithm-type scrypt secret EnableRootPass123!

parser view MONITOR-VIEW
 secret ViewPass123!
 commands exec include show ip interface brief
 commands exec include show ip interface
 commands exec include show ip
 commands exec include show
 commands exec include ping
```
* **検証方法**:
  `enable view MONITOR-VIEW` を実行してビューに遷移。`show ip interface brief` および `ping 8.8.8.8` が成功することを確認。許可されていないコマンド（例: `show running-config`, `configure terminal`）を実行し、"Command authorization failed" または無効コマンドエラーになることを検証。

---

### 例5: ASA における TACACS+ 認証・認可および LOCAL フォールバック構成

* **問題**:
  Cisco ASA において SSH および Console アクセスに対して TACACS+ 認証・認可を構成せよ。
* **要件**:
  * TACACS+ サーバ: `10.1.1.200`
  * Shared Secret: `AsaTacacsSecret123!`
  * TACACS+ サーバダウン時のみローカルデータベースにフォールバックすること。
  * Management1/1 インターフェイス経由で通信すること。
* **設定例**:
```text
aaa-server TACACS-GROUP protocol tacacs+
aaa-server TACACS-GROUP (Management1/1) host 10.1.1.200
 key AsaTacacsSecret123!
 timeout 4

aaa authentication ssh console TACACS-GROUP LOCAL
aaa authentication serial console TACACS-GROUP LOCAL
aaa authorization command TACACS-GROUP LOCAL
aaa accounting ssh TACACS-GROUP
```
* **検証方法**:
  `test aaa-server authentication TACACS-GROUP host 10.1.1.200 username testuser password TestPass123!` を実行し成功を確認。`show aaa-server` でサーバ状態をログ確認。

---

### 例6: ASA SSH / ASDM HTTPS 管理アクセスの制限設定

* **問題**:
  Cisco ASA への SSH および ASDM (HTTP) 管理接続を特定ネットワーク（Inside: 10.10.10.0/24, Mgmt: 172.16.1.0/24）からのみ許可せよ。
* **要件**:
  * Inside インターフェイス: `10.10.10.0 255.255.255.0` からの SSH/HTTP を許可。
  * Management インターフェイス: `172.16.1.0 255.255.255.0` からの SSH を許可。
  * アイドルタイムアウト: 5分。
* **設定例**:
```text
ssh 10.10.10.0 255.255.255.0 inside
ssh 172.16.1.0 255.255.255.0 Management1/1
ssh timeout 5
ssh version 2

http server enable
http 10.10.10.0 255.255.255.0 inside
http timeout 5
```
* **検証方法**:
  `10.10.10.5` から ASA の Inside IP へ SSH および ASDM 接続を試行し成功することを確認。`10.10.20.5` からの接続が即座にドロップされることを確認。`show ssh` および `show http` を実行。

---

### 例7: IOS-XE ログイン試行制限と DoS 対策 (Login Block)

* **問題**:
  ブルートフォース攻撃からルーターを保護するため、1分間に 3 回ログインに失敗した場合、その IP からのログインを 5 分間ブロックせよ。ただし、指定管理ホスト（192.168.10.100）はブロック対象外とせよ。
* **要件**:
  * 失敗条件: 60秒間に 3 回の失敗。
  * ブロック期間: 300秒（5分）。
  * 例外リスト: `ACL-ADMIN-EXEMPT`
* **設定例**:
```bash
ip access-list standard ACL-ADMIN-EXEMPT
 permit host 192.168.10.100

login block-for 300 attempts 3 within 60
login quiet-mode access-class ACL-ADMIN-EXEMPT
login on-failure log
login on-success log
```
* **検証方法**:
  管理対象外の IP（例: `10.1.1.5`）から不正なパスワードで 3 回 SSH ログインを試行。ルーターで `show login` を実行し、Quiet Mode（ブロック状態）に移行したことを確認。192.168.10.100 からはブロック中も正常にログインできることを検証。

---

### 例8: Cisco FTD (FMC Managed) における SSH アクセス制限とタイムアウト設定

* **問題**:
  FMC で管理される FTD アプライアンスの CLI (SSH) アクセス範囲を制限し、セキュリティポリシーを強化せよ。
* **要件**:
  * 許可ネットワーク: `192.168.50.0/24`
  * CLI アイドルタイムアウト: 15分
* **設定例 (FMC GUI 設定手順)**:
  1. **FMC GUI > Devices > Platform Settings**: FTD 用の Platform Settings ポリシーを作成/編集。
  2. **SSH Section**:
     * 「Add」をクリックし、IP Address: `192.168.50.0/24`、Interface: `Diagnostic` / `Inside` を選択。
     * SSH Timeout を `15` 分に設定。
  3. **Deploy**: 対象 FTD へポリシーをデプロイ。
* **検証方法**:
  FTD CLI にログインし、`show running-config ssh` を実行して指定ネットワークの `ssh` コマンドが適用されているか確認。

---

### 例9: TACACS+ コマンド単位の認可 (Command Authorization) とローカルフォールバック (IOS-XE)

* **問題**:
  特権レベル 15 のユーザーが CLI コマンドを入力する度に TACACS+ サーバへ照合を行い、TACACS+ 応答不能時のみローカル認可へ切り替えよ。
* **要件**:
  * 特権レベル: 15
  * AAA 認可リスト: `default`
* **設定例**:
```bash
aaa new-model
aaa group server tacacs+ TACACS-CLUSTER
 server name ISE-1

aaa authorization exec default group TACACS-CLUSTER local
aaa authorization commands 15 default group TACACS-CLUSTER local
```
* **検証方法**:
  `debug aaa authorization` を有効化し、`configure terminal` などのコマンドを実行。デバッグログで `AAA/AUTHOR/CMD` の要求が TACACS+ サーバへ送信され、PASS/FAIL が返されるログを確認。

---

### 例10: Exec Timeout および Banner セキュリティの設定 (IOS-XE)

* **問題**:
  すべての Console および VTY ラインにおいて非アクティブ 5 分でセッションを切断し、ログイン前およびログイン後に適切なセキュリティ警告バナーを表示せよ。
* **要件**:
  * Console / VTY タイムアウト: 5分0秒
  * MOTD バナー: 未認証アクセスの禁止とログ監視の警告
* **設定例**:
```bash
banner motd ^C
======================================================================
 WARNING: Authorized Access Only!
 All activities on this system are logged and monitored.
 Unauthorized access will be prosecuted to the full extent of the law.
======================================================================
^C

line console 0
 exec-timeout 5 0
 logging synchronous

line vty 0 15
 exec-timeout 5 0
 logging synchronous
```
* **検証方法**:
  SSH 接続時に MOTD バナーが表示されることを確認。接続後、5 分間入力を停止し、セッションが自動的にクローズ（"Timeout expired!"）されることを検証。


## ❓ 想定試験問題

### 問1 (コンフィグ読解・トラブルシュート)
**問題**:
以下の IOS-XE ルーターの設定において、TACACS+ サーバ群（`10.1.1.10`, `10.1.1.11`）へのネットワーク経路が切断された際、管理者がコンソールおよび VTY からローカルユーザー `admin` でログインを試みたが認証に失敗し、機器からロックアウトされた。その原因として最も適切なものを選べ。

```bash
aaa new-model
username admin algorithm-type scrypt secret AdminPass123!

tacacs server ISE-1
 address ipv4 10.1.1.10
 key Key123!

tacacs server ISE-2
 address ipv4 10.1.1.11
 key Key123!

aaa group server tacacs+ TACACS-GRP
 server name ISE-1
 server name ISE-2

aaa authentication login default group TACACS-GRP
aaa authorization exec default group TACACS-GRP
```

* **選択肢**:
  * A. `ip domain-name` が設定されていないため RSA 鍵が生成されずログインできない。
  * B. `aaa authentication login default` に `local` フォールバック指定が存在しないため、TACACS+ サーバ応答不能時にローカルデータベースが参照されない。
  * C. `username admin` に `privilege 15` が設定されていないためログインが拒否された。
  * D. `tacacs-server host` の旧構文が使用されていないため TACACS+ グループが正常に動作しない。

* **正解**: B
* **解説**:
  `aaa authentication login default group TACACS-GRP` と記述されており、末尾に `local` キーワードが存在しません。このため、TACACS+ サーバ群がすべて DEAD になった際、ルーターは認証処理を中断して拒否を返します。正しくは `aaa authentication login default group TACACS-GRP local` と設定する必要があります。


---

### 問2 (実装・設計)
**問題**:
Cisco ASA ファイアウォールにおいて、SSH 経由での管理アクセスを制御する際、TACACS+ 認証を設定しつつ外部サーバ障害時のロックアウトを防ぎ、かつ管理ネットワーク `192.168.10.0/24` (Inside) からのみアクセスを許可するための正しい設定の組み合わせを選べ。

* **選択肢**:
  * A.
    ```text
    aaa-server TACACS-GRP protocol tacacs+
    aaa-server TACACS-GRP (inside) host 10.1.1.10
     key Secret123!
    aaa authentication ssh console TACACS-GRP LOCAL
    ssh 192.168.10.0 255.255.255.0 inside
    ```
  * B.
    ```text
    aaa-server TACACS-GRP protocol tacacs+
    aaa-server TACACS-GRP (inside) host 10.1.1.10
     key Secret123!
    aaa authentication ssh console TACACS-GRP
    ssh 192.168.10.0 255.255.255.0 inside
    ```
  * C.
    ```text
    aaa-server TACACS-GRP protocol radius
    aaa-server TACACS-GRP (inside) host 10.1.1.10
     key Secret123!
    aaa authentication ssh console TACACS-GRP LOCAL
    ssh access-group 101 inside
    ```
  * D.
    ```text
    aaa-server TACACS-GRP protocol tacacs+
    aaa-server TACACS-GRP (inside) host 10.1.1.10
     key Secret123!
    aaa authentication ssh console LOCAL TACACS-GRP
    ssh 192.168.10.0 255.255.255.0 inside
    ```

* **正解**: A
* **解説**:
  ASA において SSH 認証を TACACS+ で行い LOCAL にフォールバックさせる構文は `aaa authentication ssh console <group-name> LOCAL` です。また、管理許可 IP の指定は `ssh <network> <mask> <interface>` コマンドで行います。D は LOCAL が第一優先となってしまい、TACACS+ が使用されないため誤りです。


---

### 問3 (パスワードセキュリティ)
**問題**:
Cisco IOS-XE において、設定ファイル内のパスワード強度を最新のセキュリティ基準に高めるため、外部攻撃者に対する耐性が最も高いアルゴリズム（Type 9）を適用するコマンドとして正しいものを選べ。

* **選択肢**:
  * A. `enable secret Type9Pass123!`
  * B. `enable algorithm-type sha256 secret Type9Pass123!`
  * C. `enable algorithm-type scrypt secret Type9Pass123!`
  * D. `service password-encryption type 9`

* **正解**: C
* **解説**:
  Cisco IOS-XE における Type 9 暗号化ハッシュは scrypt アルゴリズムを使用します（`algorithm-type scrypt`）。`sha256` は Type 8 (PBKDF2-SHA256) です。`enable secret` のみ（引数なし）の場合はデフォルトで Type 5 (MD5) または Type 8/9 になりますが、明確に Type 9 を指定するには `algorithm-type scrypt` を指定します。

---

### 問4 (トラブルシューティング)
**問題**:
ルーター R1 で SSHv2 接続を構成したところ、クライアントから接続した際に "Algorithm negotiation failed" または "Host key verification failed" というエラーが発生して接続できない。`show ip ssh` を確認したところ、SSHv2 は有効化されていた。最も考えられる原因と対処法を選べ。

* **選択肢**:
  * A. `ip ssh version 1` に変更する必要がある。
  * B. RSA ホストキーが 1024 bit 以下の短い鍵で生成されており、モダンなクライアントの暗号スイートポリシで拒否された。`crypto key generate rsa modulus 2048` で再生成する。
  * C. `transport input telnet` が設定されているため SSH が受け付けられない。
  * D. `aaa new-model` が未設定のため SSH ハンドシェイクが失敗している。

* **正解**: B
* **解説**:
  近年（OpenSSH 8.8 以降等）の SSH クライアントは 1024 bit 以下の短い RSA 鍵や SHA-1 署名を無効化しています。そのため、デフォルト（512/1024 bit）で生成された鍵ではアルゴリズム交渉エラーが発生します。2048 bit 以上の RSA 鍵を再生成（`crypto key generate rsa modulus 2048`）することで解決します。

---

### 問5 (プロトコル比較)
**問題**:
管理アクセスの AAA 設計において、RADIUS ではなく TACACS+ を選択すべき決定的な理由として正しいものを選べ。

* **選択肢**:
  * A. TACACS+ は UDP を使用するため、レスポンス速度が RADIUS よりも高速である。
  * B. TACACS+ はユーザーが入力する CLI コマンド単位での認可 (Command Authorization) が可能であり、パケット全体（ヘッダー除く）が暗号化される。
  * C. TACACS+ は 802.1X 認証や Dynamic VLAN 割り当てに対応しているため。
  * D. RADIUS は認証と認可が完全に独立したプロトコル構造を持っているため。

* **正解**: B
* **解説**:
  TACACS+ は TCP 49 を使用し、ヘッダーを除く全パケット構造を暗号化します。さらに認証（Authentication）と認可（Authorization）が完全に分離しているため、CLI の各コマンド入力時に「そのコマンドを実行してよいか」を単位ごとに評価（Command Authorization）できます。RADIUS は UDP 1812/1813 を使い、パスワード部分のみ暗号化し、認証と認可が一体化しているためコマンド単位認可には適しません。


## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術ガイド一覧:

* [Cisco IOS Security Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_cfg/configuration/xe-16/sec-usr-cfg-xe-16-book.html)
* [Configuring Secure Shell (SSH) on Cisco IOS Devices](https://www.cisco.com/c/en/us/support/docs/security-vpn/secure-shell-ssh/24060-46.html)
* [Cisco ASA Series CLI Configuration Guide - AAA and Device Access](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/general/asa-94-general-config/access-mgmt.html)
* [Cisco Secure Firewall Management Center Administration Guide](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/external_authentication_servers.html)

---

## 📝 **補足（Notes）**

- **コンソールアクセス制限とフォールバック**:
  外部 AAA サーバ疎通不能時の機器管理を維持するため、認証リストの末尾に `local`（ASA の場合は `LOCAL`）を構成します。
- **ASA と IOS-XE の構文上の違い**:
  IOS/IOS-XE では `aaa authentication login default group TACACS-GRP local`（小文字）を使用しますが、ASA CLI では `aaa authentication ssh console TACACS-GRP LOCAL`（大文字）を使用します。
