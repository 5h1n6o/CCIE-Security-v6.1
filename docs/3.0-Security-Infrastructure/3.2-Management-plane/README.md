---
layout: default
title: 3.2-Management-plane
nav_order: 2
parent: 3.0-Security-Infrastructure
---

# 3.2 Management plane protection techniques

Cisco ネットワークセキュリティ基盤における **Management Plane Protection（管理プレーン保護技術）** は、ルーター、スイッチ、ASA ファイアウォール、および FTD（Firepower Threat Defense）アプライアンスの管理インターフェイスやプロセッサ（CPU）を、不正アクセス、ブルートフォース攻撃、DoS/DDoS 攻撃、および盗聴から隔離・保護するための包括的なアーキテクチャおよびセキュリティハードニング手法です。

CCIE Security v6.1 Blueprint **3.2 Management plane protection techniques** では、**Cisco IOS/IOS-XE の Management Plane Protection (MPP)**、**Control Plane Protection (CPPr) の Host サブインターフェイス**、**ASA / FTD の管理プレーンアクセス制限（Management-only Interface、Platform Settings Access List、AAA 管理認証）**、**安全な管理プロトコル（SSHv2, HTTPS/TLS 1.2/1.3, SNMPv3 authPriv, SCP/SFTP）**、および **TACACS+/RADIUS コマンド認可 ＆ パスワード暗号化（Type 6/8/9）** に至る実装手順とトラブルシューティング能力が深く問われます。

---

## 📘 概要

* **機能概要**:
  * **Management Plane Protection (MPP)**: Cisco IOS/IOS-XE デバイスにおいて、SSH、TELNET、HTTP、HTTPS、SNMP などの管理トラフィックを受信できるインターフェイス（Out-of-Band または特定の In-Band インターフェイス）を明示的に指定・限定し、その他のインターフェイスで着信した管理パケットをコントロールプレーンで破棄する機能。
  * **ASA Management Security**: ASA において、`ssh`、`http` (ASDM/REST API)、`snmp-server` コマンドで接続元 IP とインターフェイスを紐付け、さらに `management-only` インターフェイス指定や管理用 AAA 認証 (`aaa authentication ssh/http`) を組み合わせることで管理プレーンを保護。
  * **FTD Platform Settings ＆ Access Control**: FMC（Firepower Management Center）から管理される FTD アプライアンスに対し、Platform Settings 経由で SSH/SNMP/HTTPS アクセス制御リスト（Access List）を適用し、管理インターフェイス (e0/0 または Dedicated Mgmt Port) へのアクセスを制御。
  * **安全なプロトコルと認証ハードニング**: 非暗号化プロトコル（TELNET, HTTP, FTP, TFTP, SNMPv1/v2c）を全面的に禁止し、SSHv2、HTTPS (TLS 1.2/1.3)、SNMPv3 (authPriv)、SCP/SFTP へ強制変更。また、TACACS+ によるコマンドレベルの認可 (Command Authorization) と Local Fallback 構成を適用。
* **利用目的**:
  * データプレーン（Transit Traffic）インターフェイス経由での不正な管理アクセス試行を物理的・論理的に遮断。
  * 管理セッション上の機密情報（管理者パスワード、コミュニティ名、コンフィグ情報）の盗聴および改ざん防止。
  * 管理インターフェイスへのブルートフォース攻撃や DoS 攻撃による CPU 高負荷の防止。
* **どのような場面で利用するか**:
  * OOB（Out-of-Band）専用管理ネットワークが存在するエンタープライズ Data Center および 拠点エッジ環境。
  * 外部（Outside）インターフェイス経由での管理プロトコル着信を一切拒否する必要がある境界ファイアウォール構築時。

---

## 🔑 要点

| 項目 | Cisco IOS / IOS-XE (MPP) | Cisco ASA | Cisco FTD (FMC Managed) |
| :--- | :--- | :--- | :--- |
| **主要保護メカニズム** | `control-plane host` 内の `management-interface` | `ssh`/`http` アクセスリスト ＋ `management-only` | Platform Settings 経由の Access List ＋ Diagnostic CLI |
| **制限対象プロトコル** | SSH, TELNET, HTTP, HTTPS, SNMP, BGP, BEEP | SSH, HTTP (ASDM), TELNET, SNMP | SSH, HTTPS (FDM/FMC), SNMP |
| **デフォルト動作** | 全インターフェイスで管理アクセス受容可能 | 明示的に許可指定した IP/IF のみ受容 | Platform Settings / FDM で許可した IP/IF のみ受容 |
| **OOB 隔離手法** | 専用 Management 0/0 インターフェイス ＋ MPP | `management-only` コマンド指定 | Dedicated Management Interface (br1 / eth0) |
| **認可・アクセス制御** | AAA TACACS+ ＋ Command Authorization | AAA TELNET/SSH/HTTP Console ＋ Local | FMC 統合 AAA (RADIUS/TACACS+/LDAP) |
| **ログ ＆ アラート** | Syslog, SNMP Traps, Audit Logs | Syslog (Message ID 111008/111010) | FMC Audit Logs ＆ FTD Diagnostic Logs |

---


## 🏗 動作原理

### 1. Cisco IOS/IOS-XE Management Plane Protection (MPP) パイプライン

MPP は CoPP (Control Plane Policing) や CPPr (Control Plane Protection) の Host サブインターフェイスの上位レイヤーとして機能し、パケットが Control Plane の CPU 処理キューに入る前段でインターフェイス条件とプロトコル条件を検査します。

```text
[ Incoming Packet on Interface ]
               │
               ▼
[ 1. L1/L2 MAC & IP Header Sanity Check ]
               │
               ▼
[ 2. Destination IP = Router Local IP Address? ]
               ├── NO ──► Proceed to Data Plane Forwarding (FIB / CEF)
               └── YES ─► Management Plane / Control Plane Flow
                              │
                              ▼
               [ 3. Is Packet a Management Protocol? ] (SSH, HTTPS, SNMP, etc.)
                              ├── NO ──► Hand off to Control Plane (CoPP / Routing Protocols)
                              └── YES ─► Evaluate MPP Rules (`control-plane host`)
                                             │
                                             ├── Match Allowed Ingress Interface?
                                             │     ├── YES ──► Pass to Application Daemon (SSHd, HTTPd, SNMPd)
                                             │     └── NO  ──► DROP Packet Immediately & Increment Counter
```

---

### 2. Cisco ASA / FTD 管理アクセス制御パイプライン

ASA および FTD Lina エンジンにおける管理トラフィック制御処理フロー：

```text
[ Ingress Management Packet (e.g., SSH Port 22) ]
               │
               ▼
[ 1. Management-Only Interface Filter ]
               ├── Ingress IF is Mgmt-Only & Dest IP is Data Interface? ──► DROP
               └── Ingress IF matches Destination IP ─────────────────────► Continue
                              │
                              ▼
[ 2. Management Access List Check (`ssh`, `http`, `snmp-server`) ]
               ├── Sender IP & Ingress IF in Allow List?
               │     ├── NO  ──► DROP & Log (Syslog 106015 / 106023)
               │     └── YES ──► Continue
                              │
                              ▼
[ 3. Management AAA Authentication (`aaa authentication ssh/http`) ]
               ├── AAA TACACS+/RADIUS Server Reachable?
               │     ├── Pass ──► Grant Access (Apply Privilege Level / Command Authorization)
               │     └── Fail/Timeout ──► Local Database Fallback Check
```

---

## ⚙ 動作シーケンス

1. **管理パケット着信とインターフェイス識別**:
   * 管理者クライアント（例: `10.1.100.50`）からルーター / ASA / FTD の管理 IP アドレス宛に SSH（TCP 22）または HTTPS（TCP 443）パケットが送出されます。
2. **MPP 照合（Cisco IOS/IOS-XE）**:
   * ルーターは着信パケットの「着信物理/論理インターフェイス」と「宛先ポート番号」を取得します。
   * `control-plane host` 下に設定された `management-interface <IF> allow <protocols>` 規則を参照。
   * 着信インターフェイスが許可リストに含まれていない場合、パケットはプロセッサのアプリケーションデーモンに渡される前に即座にドロップされます。
3. **管理 ACL ＆ Management-Only 検証（ASA / FTD）**:
   * ASA / FTD は着信ポートに対応する管理アクセス設定（`ssh 10.1.100.0 255.255.255.0 inside` や Platform Settings Access List）を照合します。
   * 該当インターフェイスに `management-only` が設定されている場合、そのインターフェイスを経由した Data Plane（Transit）トラフィックの通過は一切遮断され、自身の管理宛てトラフィックのみが受容されます。
4. **AAA 認証 ＆ 認可処理**:
   * プロトコルデーモン（SSHd / Web Engine）が起動し、AAA 認証が発動。TACACS+（Port 49）または RADIUS（Port 1812/1812）サーバーへ `Access-Request` を送信。
   * 認証成功後、ユーザーの権限レベル（Privilege Level 15 等）および実行可能コマンドが TACACS+ Command Authorization 規則に従って動的に制御されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Cisco IOS MPP の設定コマンドと適用制約**:
   * `control-plane host` 設定モードに入り、`management-interface <interface-name> allow <protocols>` で定義します。
   * **指定可能プロトコル**: `ssh`, `telnet`, `http`, `https`, `snmp`, `beep`, `snmp` 等。
   * **非常に重要**: MPP を設定すると、**明示的に指定しなかった全インターフェイス上の該当管理プロトコルが自動的に拒否**されます。例えば `management-interface GigabitEthernet0/0 allow ssh` を投入すると、`GigabitEthernet0/1` 経由の SSH 接続は即座に拒否されます。自身が接続している SSH セクションを遮断しないよう、適用順序に細心の注意を払う必要があります。
2. **ASA `management-only` と `same-security-traffic` の組み合わせ**:
   * インターフェイス設定で `management-only` を投入すると、そのポートはルーティングやファイアウォール通過（Transit）に使用できなくなります。
   * ラボ要件で「GigabitEthernet0/0 は管理専用とし、ルーティングトラフィックを通過させてはならない」と指示された場合、必ず `management-only` を設定します。
3. **FTD Platform Settings 経由の SSH / HTTPS / SNMP アクセス制御**:
   * FMC GUI の `Devices > Platform Settings` で Custom SSH / HTTP / SNMP ポリシーを作成し、ターゲット FTD にアサインします。
   * Diagnostic CLI (Lina Engine) に対する SSH アクセスと、FTD OS (Linux Shell / FXOS) に対する SSH アクセスのポート指定・許可 IP の差異を理解しておく必要があります。
4. **AAA Management Command Authorization と Local Fallback**:
   * TACACS+ サーバーを用いたコマンド認可設定：
     `aaa authorization commands 15 default group tacacs+ local`
   * TACACS+ サーバーがダウンしている場合に全コマンドがブロックされないよう、必ず `local` または `none` のフォールバックを設定します。
5. **暗号化パスワード（Password Encryption）の要件順守**:
   * 古い `service password-encryption` (Type 7 Vigenère) や `enable password` (Type 0 Plaintext) は不可。
   * **Type 6 (AES)**: `password encryption aes` を有効化し、`key config-key password-encryption` を設定。
   * **Type 8 (PBKDF2-SHA256)** / **Type 9 (scrypt)**: `username admin secret 9 <hash>` または `enable secret 9 <hash>` を使用。

---


## 🛠 設定方法

### 1. Cisco IOS/IOS-XE: Management Plane Protection (MPP) ＆ SSHv2 ハードニング

```bash
! ===== Step 1: ホスト名 ＆ ドメイン名 ＆ RSA 鍵生成 =====
hostname R1-EDGE
ip domain-name ccie.lab
crypto key generate rsa general-keys modulus 2048 label SSH_KEY

! ===== Step 2: SSHv2 制限 ＆ VTY / Console ハードニング =====
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3

line con 0
 exec-timeout 10 0
 logging synchronous
 login local
line vty 0 4
 exec-timeout 10 0
 transport input ssh
 login local

! ===== Step 3: Management Plane Protection (MPP) 設定 =====
control-plane host
 management-interface GigabitEthernet0/0 allow ssh https snmp
 management-interface Loopback0 allow ssh
```

---

### 2. Cisco ASA: Management Security, AAA 認証 ＆ Management-Only IF

```bash
! ===== Step 1: インターフェイスへの Management-Only アサイン =====
interface GigabitEthernet0/0
 nameif management
 security-level 100
 ip address 10.1.100.1 255.255.255.0
 management-only

! ===== Step 2: SSH / HTTPS アクセス許可 =====
ssh 10.1.100.0 255.255.255.0 management
http server enable
http 10.1.100.0 255.255.255.0 management
ssh version 2
ssh timeout 10

! ===== Step 3: 管理 AAA 認証 (TACACS+ ＋ Local Fallback) =====
aaa-server TACACS_GROUP protocol tacacs+
aaa-server TACACS_GROUP (management) host 10.1.100.100
 key SecretKey123!

aaa authentication ssh console TACACS_GROUP LOCAL
aaa authentication http console TACACS_GROUP LOCAL
aaa authorization command TACACS_GROUP LOCAL
```

---

### 3. Cisco FTD (FMC GUI): Platform Settings による管理アクセス制限

```text
[ Step 1: Platform Settings ポリシーの作成 ]
1. FMC UI で Devices > Platform Settings に移動。
2. "New Policy" をクリック (Type: FTD Platform Settings, Name: FTD_HARDENING_POLICY)。

[ Step 2: SSH アクセス制限の設定 ]
1. Platform Settings 編集画面で "SSH" タブを選択。
2. "Add" をクリック:
   - IP Address / Network: 10.1.100.0/24 (Management Admin Subnet)
   - Interface: mgmt (または Diagnostic)
3. Save して Deploy。

[ Step 3: SNMPv3 認証設定 ]
1. "SNMP" タブを選択 ➔ SNMPv3 Users タブを開く:
   - User Name: snmpadmin
   - Security Level: Auth-Priv
   - Authentication Algorithm: SHA-256 (Password: AuthPass123!)
   - Encryption Algorithm: AES-128 (Password: PrivPass123!)
2. Save して Deploy。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (IOS / ASA / FTD) | 表示・確認ポイント |
| :--- | :--- | :--- |
| **IOS MPP 適用状態の確認** | <code>show control-plane host open-ports</code> | MPP で許可されたインバウンドポートとバインドインターフェイス一覧 |
| **IOS SSH セッション ＆ バージョン確認** | <code>show ip ssh</code><br><code>show ssh</code> | SSHv2 の有効化状態、暗号化アルゴリズム、アクティブ接続一覧 |
| **ASA 管理アクセス設定の確認** | <code>show running-config ssh</code><br><code>show running-config http</code> | 許可されている送信元 IP、マスク、インターフェイスバインド |
| **ASA 管理 SSH アクティブ接続確認** | <code>show ssh</code> | 接続中の管理者 IP、ユーザー名、確立時間 |
| **ASA Management-Only インターフェイス確認**| <code>show interface management</code> | `Management-only interface` の有効化フラグ状態 |
| **FTD Diagnostic CLI 管理設定確認** | <code>show running-config ssh</code> (Diagnostic CLI) | Lina レベルでの SSH アクセス許可エントリー |
| **TACACS+ AAA サーバー到達性 ＆ 認証確認** | <code>test aaa-server authentication TACACS_GROUP user admin password Pass123</code> | AAA サーバーからの `SERVER RESPONSE: SUCCESS` 応答 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **IOS で MPP 設定追加直後、別ポートからの SSH セッションが一切拒否される** | MPP は「明示的に許可したインターフェイス以外を全拒否」するため、現在接続中のポートが含まれていなかった | <code>show control-plane host open-ports</code> | コンソールポートからログインし、`management-interface` に正しい管理インターフェイスを追加する。 |
| **ASA の `management-only` インターフェイスに接続したクライアントから社内 サーバーへ通信が通らない** | `management-only` インターフェイスは通過（Transit）トラフィックを一切フォワーディングしない仕様 | <code>show interface</code> | 通過トラフィック用のデータインターフェイス（Data Interface）を別途割り当てる。 |
| **TACACS+ サーバー障害時、ルーターに SSH ログインできなくなる** | `aaa authentication login default group tacacs+` の末尾に `local` フォールバックが記述されていなかった | <code>show running-config \| section aaa</code> | コンソールから復旧し、`aaa authentication login default group tacacs+ local` へコンフィグ変更する。 |
| **SNMPv3 で GetRequest を送信してもタイムアウトする** | SNMPv3 ユーザーの `auth` / `priv` パスワード誤り、または ACL で UDP 161 がブロックされている | <code>debug snmp packets</code><br><code>show snmp user</code> | SNMPv3 パスワードおよびグループの `authPriv` セキュリティレベルを再確認・修正する。 |

---

## ⚠ 制限事項

* **IOS MPP の対象プロトコル制限**:
  * MPP は IP サービスプロトコル（SSH, TELNET, HTTP, HTTPS, SNMP, BGP）のみをサポートします。ICMP（Ping / Traceroute）や OSPF, EIGRP などの L3 ルーティングプロトコルは MPP の直接の制御対象外であり、これらは CoPP または iACL で制御する必要があります。
* **ASA Management-Only インターフェイスのルーティング制限**:
  * `management-only` インターフェイスは、自身が生成・応答する管理トラフィック専用の独立したルーティングテーブル（Management Routing Table）を持つため、データプレーンのルーティングテーブル（Global Route Table）とルートを共有・再配布することはできません。

---

## 🔄 他技術との関連

* **Control Plane Policing (CoPP) / CPPr**: MPP が「どの物理/論理インターフェイスで受容するか」を定義するのに対し、CoPP/CPPr は「どの程度のレート（帯域）でCPUに渡すか」を制御。両者を組み合わせて多層防御を構築。
* **Infrastructure ACL (iACL)**: ネットワーク境界（Edge）で管理プレーン宛てトラフィックを遮断するのに対し、MPP はルーター自身（Local Device）の入力ポイントで遮断。
* **Cisco ISE / TACACS+ (ISE Device Admin)**: 管理者のログイン認証、シェルアクセスの権限昇格 (Privilege Level 15)、および個別コマンドの認可ログ記録を提供。

---

## 🧩 比較表

### IOS MPP vs ASA Management Security vs FTD Platform Settings

| 比較項目 | Cisco IOS / IOS-XE (MPP) | Cisco ASA | Cisco FTD (FMC Managed) |
| :--- | :--- | :--- | :--- |
| **制御記述場所** | `control-plane host` | グローバルコンフィグ (`ssh`/`http`) | FMC Platform Settings |
| **指定軸** | **インターフェイス単位** ＋ プロトコル | **送信元 IP / マスク** ＋ インターフェイス | **送信元 IP / マスク** ＋ インターフェイス |
| **通過トラフィック影響** | なし（Local IP 宛てのみ影響） | なし（Local IP 宛てのみ影響） | なし（Local IP 宛てのみ影響） |
| **デフォルト拒否動作** | MPP 設定投入時に未指定 IF を全拒否 | コンフィグ未定義の IP/IF は全拒否 | Platform Settings 未定義の IP/IF は全拒否 |

---

## 💡 ベストプラクティス

1. **OOB 専用管理インターフェイス ＋ MPP の徹底**:
   * エンタープライズ環境では Dedicated Management Port (Gi0/0 または Mgmt0) に OOB 管理 IP を割り当て、MPP で SSH / HTTPS / SNMP を該当ポートのみに限定します。
2. **AAA TACACS+ コマンド認可 ＋ Local Database Fallback**:
   * 管理者の特権昇格コマンド（`configure terminal`, `reload` 等）は TACACS+ サーバーで集中監査・認可し、障害時の救済処置として Local Console ログインのフォールバックを用意します。
3. **安全な暗号化アルゴリズム ＆ Type 9 パスワードの標準化**:
   * SSHv2 強制 (`ip ssh version 2`)、RSA 2048 ビット以上の鍵、HTTPS TLS 1.2/1.3 の固定、および `secret 9` (scrypt) ハッシュパスワードを標準構成テンプレートとします。

---


## 📝 ラボ学習・設定サンプル例

### 例1: Cisco IOS MPP による Out-of-Band (Gi0/0) 管理ポートの保護

#### 問題
R1 において、管理プロトコル（SSH、HTTPS、SNMP）の着信を OOB 管理インターフェイス `GigabitEthernet0/0` のみに限定し、データプレーンポート（`GigabitEthernet0/1`）経由の管理アクセスをすべて拒否せよ。

#### 要件
* MPP (`control-plane host`) を使用すること。
* `GigabitEthernet0/0` で `ssh`, `https`, `snmp` を許可すること。

#### 設定例

```bash
! ===== R1 Configuration =====
hostname R1
ip domain-name ccie.lab
crypto key generate rsa general-keys modulus 2048 label SSH_KEY

ip ssh version 2

interface GigabitEthernet0/0
 description OOB-MGMT-INTERFACE
 ip address 10.1.100.1 255.255.255.0
 no shutdown

interface GigabitEthernet0/1
 description DATA-PLANE-INTERFACE
 ip address 172.16.1.1 255.255.255.0
 no shutdown

control-plane host
 management-interface GigabitEthernet0/0 allow ssh https snmp
```

#### 検証方法
* `show control-plane host open-ports` を実行し、Gi0/0 のみに SSH(22), HTTPS(443), SNMP(161) が開いていることを確認。
* `172.16.1.1` (Gi0/1) 宛ての SSH 接続が拒否されることを確認。

---

### 例2: Cisco ASA Management-Only インターフェイス ＆ AAA SSH/HTTP 認証

#### 問題
ASA1 において、`GigabitEthernet0/0` を `management-only` インターフェイスとして構成し、IP サブネット `10.1.100.0/24` からの SSH および ASDM (HTTP) 接続を許可せよ。また、管理アクセス認証に TACACS+ サーバー `10.1.100.100` と Local Database フォールバックを設定せよ。

#### 要件
* `management-only` インターフェイス指定。
* `aaa authentication ssh console` および `aaa authentication http console` の構成。

#### 設定例

```bash
! ===== ASA1 Configuration =====
interface GigabitEthernet0/0
 nameif management
 security-level 100
 ip address 10.1.100.1 255.255.255.0
 management-only

ssh 10.1.100.0 255.255.255.0 management
http server enable
http 10.1.100.0 255.255.255.0 management
ssh version 2

username admin password LocalPass123! privilege 15

aaa-server TACACS_GROUP protocol tacacs+
aaa-server TACACS_GROUP (management) host 10.1.100.100
 key TACACS_Secret_Key123

aaa authentication ssh console TACACS_GROUP LOCAL
aaa authentication http console TACACS_GROUP LOCAL
```

#### 検証方法
* `show interface management` で `Management-only interface` が有効であることを確認。
* `10.1.100.50` から SSH 接続し、TACACS+ 認証ログが記録されることを確認。

---

### 例3: Cisco IOS SSHv2 ハードニング ＆ RSA Keypair / Timeout 指定

#### 問題
R1 において、SSH アクセスを厳格化せよ。SSHv2 のみを許可し、暗号鍵として 2048 ビットの RSA 鍵 `R1_SSH_KEYS` を使用せよ。また、タイムアウトを 60 秒、認証試行回数を 2 回に制限せよ。

#### 要件
* `ip ssh version 2`
* `ip ssh rsa keypair-name R1_SSH_KEYS`
* `ip ssh time-out 60`, `ip ssh authentication-retries 2`

#### 設定例

```bash
! ===== R1 Configuration =====
hostname R1
ip domain-name ccie.lab

crypto key generate rsa general-keys modulus 2048 label R1_SSH_KEYS

ip ssh version 2
ip ssh rsa keypair-name R1_SSH_KEYS
ip ssh time-out 60
ip ssh authentication-retries 2

line vty 0 15
 exec-timeout 5 0
 transport input ssh
 login local
```

#### 検証方法
* `show ip ssh` を実行し、SSHv2、Keypair 名、タイムアウト、リトライ回数が意図通り適用されていることを確認。

---

### 例4: TACACS+ による IOS EXEC コマンド認可 (Command Authorization) ＆ Local Fallback

#### 問題
R1 において、管理者（特権レベル 15）が実行する全コマンドに対し、TACACS+ サーバー `10.1.100.100` によるコマンド認可を要求せよ。TACACS+ サーバー不可達時はローカル認可へフォールバックさせよ。

#### 要件
* `aaa authorization commands 15 default group tacacs+ local`

#### 設定例

```bash
! ===== R1 Configuration =====
aaa new-model

tacacs server TACACS_SRV
 address ipv4 10.1.100.100
 key TacacsKey123!

aaa group server tacacs+ TACACS_GROUP
 server name TACACS_SRV

aaa authentication login default group TACACS_GROUP local
aaa authorization exec default group TACACS_GROUP local
aaa authorization commands 15 default group TACACS_GROUP local
```

#### 検証方法
* R1 にログインして `configure terminal` などを実行した際、TACACS+ サーバーに `Command Authorization` リクエストが送信されることを確認。

---

### 例5: Cisco IOS Password Encryption Type 9 (scrypt) ＆ AES Type 6 ハードニング

#### 問題
R1 において、ローカルユーザー `admin` のパスワードを Type 9 (scrypt) ハッシュで暗号化し、さらにコンフィグ内の Type 7 平文暗号化を強固な AES Type 6 暗号化へアップグレードせよ。

#### 要件
* `username admin secret 9 <password>`
* `password encryption aes` ＋ Master Key の設定

#### 設定例

```bash
! ===== R1 Configuration =====
! マスターキーの設定 (AES Type 6 暗号化用)
key config-key password-encryption MasterKey2026!
password encryption aes

! Type 9 (scrypt) ハッシュを用いたユーザー作成
username admin secret 9 AdminScryptPass123!
enable secret 9 EnableScryptPass123!
```

#### 検証方法
* `show running-config | inc username|secret` を実行し、パスワードが `$9$` で始まるハッシュ（Type 9）および `$6$` でコード化されていることを確認。

---

### 例6: Cisco FTD (FMC) Platform Settings による SSH アクセス制限

#### 問題
FMC 管理下の FTD1 に対し、Platform Settings ポリシーを用いて SSH 管理アクセスを管理サブネット `10.1.100.0/24` のみに制限せよ。

#### 要件
* FMC Platform Settings 経由でアサインすること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Devices > Platform Settings ➔ "New Policy" (Name: FTD_HARDENING).
2. SSH タブを選択 ➔ "Add":
   - IP Address / Network: 10.1.100.0/24
   - Interface: mgmt
3. Policy に FTD1 をアサインして Deploy。
```

#### 検証方法
* FTD Diagnostic CLI で `show running-config ssh` を実行し、`ssh 10.1.100.0 255.255.255.0 mgmt` が反映されていることを確認。

---

### 例7: Cisco IOS SNMPv3 (authPriv) ＋ VACM (View-based Access Control)

#### 問題
R1 において、SNMPv3 グループ `GRP_ADMIN` およびユーザー `snmpuser` を作成し、SHA-256 認証 ＆ AES-128 暗号化 (`authPriv`) を構成せよ。また、MIB ビュー `VIEW_LIMITED` で `system` および `interfaces` ツリーのみにアクセスを限定せよ。

#### 要件
* SNMPv3 `authPriv`
* MIB View による最小権限制御

#### 設定例

```bash
! ===== R1 Configuration =====
snmp-server view VIEW_LIMITED system include
snmp-server view VIEW_LIMITED interfaces include

snmp-server group GRP_ADMIN v3 priv read VIEW_LIMITED

snmp-server user snmpuser GRP_ADMIN v3 auth sha AuthPass123! priv aes 128 PrivPass123!
```

#### 検証方法
* `show snmp user` を実行し、ユーザー `snmpuser` が `authPriv` (SHA / AES128) かつ `VIEW_LIMITED` にバインドされていることを確認。

---

### 例8: Cisco ASA Service Policy による管理アクセス制限 (Management Policing)

#### 問題
ASA1 の `management` インターフェイス宛ての SSH トラフィックに対し、DoS 攻撃対策として 1 Mbps (1000000 bps) のレート制限 (Policing) を適用せよ。

#### 要件
* ASA Service Policy (MPF) を使用すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
access-list ACL_MGMT_SSH extended permit tcp any interface management eq ssh

class-map CM_MGMT_SSH
 match access-list ACL_MGMT_SSH

policy-map PM_MGMT_PROTECT
 class CM_MGMT_SSH
  police input 1000000 10000

service-policy PM_MGMT_PROTECT interface management
```

#### 検証方法
* `show service-policy interface management` を実行し、SSH トラフィックがクラスマップ `CM_MGMT_SSH` にマッチして Policing 評価されていることを確認。

---

### 例9: NTP 認証 (NTP MD5 Authentication) による時刻同期の保護

#### 問題
R1 において、NTP サーバー `10.1.100.123` との時刻同期に対し、MD5 認証キー（Key ID: 1、Key: `NtpSecretKey123`）を用いた認証を必須化せよ。

#### 要件
* `ntp authenticate`
* `ntp authentication-key 1 md5 NtpSecretKey123`
* `ntp trusted-key 1`

#### 設定例

```bash
! ===== R1 Configuration =====
ntp authenticate
ntp authentication-key 1 md5 NtpSecretKey123
ntp trusted-key 1
ntp server 10.1.100.123 key 1
```

#### 検証方法
* `show ntp associations detail` を実行し、`authenticated` フラグが立って正常に同期されていることを確認。

---

### 例10: Banner 警告メッセージ ＆ VTY / Console 自動ログアウトの設定

#### 問題
R1 において、不正アクセスに対する法的警告を表示する `banner login` を構成し、Console および VTY ラインのアイドルタイムアウトを 5 分に設定せよ。

#### 要件
* `banner login`
* `exec-timeout 5 0`

#### 設定例

```bash
! ===== R1 Configuration =====
banner login ^C
*****************************************************************
* WARNING: AUTHORIZED ACCESS ONLY. ALL ACTIVITIES ARE LOGGED.   *
* UNAPPROVED ACCESS ATTEMPTS WILL BE PROSECUTED TO THE LAW.    *
*****************************************************************
^C

line con 0
 exec-timeout 5 0
 logging synchronous

line vty 0 15
 exec-timeout 5 0
 transport input ssh
```

#### 検証方法
* SSH 接続時にバナーメッセージが表示され、5 分間無操作状態を維持するとセッションが自動切断されることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: R1 において以下の MPP コンフィグが投入されている。

```bash
control-plane host
 management-interface GigabitEthernet0/0 allow ssh https
```

この状態において、管理者が `GigabitEthernet0/1` (IP: `172.16.1.1`) 経由で R1 に SSH 接続を試みた場合、パケットはどのように処理されるか。最も適切な説明を選べ。

* A) CEF スイッチングにより正常に SSH 接続が確立する。
* B) CoPP のデフォルトクラスマップに引き渡されて処理される。
* C) MPP の規則により、`GigabitEthernet0/1` での SSH 着信はコントロールプレーンに到達する前に直ちにドロップされる。
* D) `GigabitEthernet0/0` 経由でリダイレクトされて処理される。

**正解**: **C**  
**解説**: Cisco IOS MPP (`control-plane host`) では、`management-interface` で許可されたインターフェイス（本例では Gi0/0）以外の全物理/論理ポートにおいて、指定された管理プロトコル（SSH, HTTPS）の着信が自動的に拒否・ドロップされます。

---

### 2. トラブルシュート問題

**問題**: ASA1 において、管理インターフェイス `GigabitEthernet0/0` に `management-only` コマンドを追加設定した直後、内部クライアント (`10.1.1.0/24`) から外部 Web サーバー (`192.168.1.10`) 宛ての通信が完全に停止した。原因として正しいものはどれか。

* A) `management-only` を設定したことで、ASA 全体のライセンスが無効化されたため。
* B) `GigabitEthernet0/0` がデータプレーンのルーティング/フォワーディング（Transit Traffic）から除外され、通過トラフィックを転送できなくなったため。
* C) ASA の ASDM サービスが停止したため。
* D) `management-only` により全 Access List が自動削除されたため。

**正解**: **B**  
**解説**: ASA の `management-only` インターフェイスは、自身宛ての管理トラフィック専用に隔離されます。Transit（通過）トラフィックの転送には使用できなくなるため、誤ってデータプレーン用ポートに設定すると通信障害を引き起こします。

---

### 3. Design 問題

**問題**: エンタープライズネットワークにおける管理プレーン（Management Plane）のセキュリティ設計として、Cisco ベストプラクティスに合致する構成の組み合わせはどれか。

* A) TELNET 接続を許可し、パスワードを Type 7 で暗号化する。
* B) データプレーンポート経由で SNMPv1 コミュニティ名を共有する。
* C) OOB (Out-of-Band) 専用管理ネットワークを配置し、MPP で SSHv2/HTTPS/SNMPv3 (authPriv) のみを許可し、TACACS+ コマンド認可を有効化する。
* D) HTTP プロトコルを平文で運用し、アクセスリストのみで保護する。

**正解**: **C**  
**解説**: OOB 管理の物理/論理隔離、MPP による受容ポート制限、暗号化プロトコル (SSHv2, HTTPS, SNMPv3 authPriv)、および TACACS+ コマンド認可の組み合わせが Cisco の推奨する管理プレーンハードニングアーキテクチャです。

---

### 4. 実装問題

**問題**: Cisco IOS において、平文パスワードや Type 7 暗号化よりも強固な PBKDF2-SHA256 (Type 8) または scrypt (Type 9) ハッシュを用いてユーザーを作成するコマンドとして正しいものはどれか。

* A) `username admin password 7 <hash>`
* B) `username admin secret 9 <password>`
* C) `username admin password aes <password>`
* D) `username admin privilege 15 password <password>`

**正解**: **B**  
**解説**: `username <name> secret 9 <password>` コマンドを使用すると、scrypt アルゴリズムに基づく不可逆で強固な Type 9 ハッシュ値でパスワードが保存されます (`secret 8` は PBKDF2-SHA256)。

---

### 5. コンフィグ読解問題

**問題**: ASA において以下の AAA 認証コンフィグが投入されている。

```bash
aaa-server TACACS_GROUP protocol tacacs+
aaa-server TACACS_GROUP (management) host 10.1.100.100
 key SecretKey123!

aaa authentication ssh console TACACS_GROUP LOCAL
```

TACACS+ サーバー `10.1.100.100` がネットワーク障害で応答しない場合、ASA はどのように SSH ログインを処理するか。

* A) SSH ログインを即座に拒否する。
* B) 認証なしで即座にログインを許可する。
* C) ASA ローカルユーザーデータベース (`LOCAL`) を照合してログイン認証を実行する。
* D) RADIUS サーバーにリダイレクトする。

**正解**: **C**  
**解説**: 末尾の `LOCAL` キーワードにより、第一優先の AAA サーバーグループ (`TACACS_GROUP`) がタイムアウトまたは不可達（Unreachable）となった場合に、ASA ローカルデータベースに自動フォールバックします。

---

## 🔗 参考リソース

* [Cisco IOS Management Plane Protection Configuration Guide](https://www.cisco.com/)
* [Cisco ASA Series General Operations Configuration Guide: Securing Management Access](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Device Configuration Guide: Platform Settings](https://www.cisco.com/)
* [Cisco Live: Hardening Cisco IOS Devices against Cyber Attacks (BRKSEC-2001)](https://www.ciscolive.com/)
* [Cisco Technical Notes: Guide to Cisco Infrastructure Security Architecture](https://www.cisco.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で Management Plane Protection が問われた場合のチェックリスト：
    1. **IOS MPP**: `control-plane host` ➔ `management-interface <IF> allow <protocols>` (必要のない IF が拒否されていないか確認)。
    2. **ASA Mgmt Security**: `ssh <net> <mask> <if>`, `http <net> <mask> <if>`, `management-only` の適切なバインド。
    3. **FTD Mgmt Security**: FMC Platform Settings Access List で SSH / SNMP 制限をデプロイ。
    4. **AAA Authorization**: `aaa authorization commands 15 default group tacacs+ local` (必ず `local` フォールバックを記述)。
    5. **Password Encryption**: `secret 9` (scrypt) または `password encryption aes` の指定を厳守。
* **図解（MPP vs CoPP vs iACL 役割分担）**:
  ```text
  [ Outside Internet / Untrusted Network ]
                 │
                 ▼
  [ 1. iACL (Edge Interface) ] ────► 外部からの管理 IP 宛てパケットを境界で破棄
                 │
                 ▼
  [ 2. MPP (Control Plane Host) ] ──► 許可されていない物理ポートでの管理受容を破棄
                 │
                 ▼
  [ 3. CoPP (Control Plane Rate) ] ─► 許容レート超過パケットをシェーピング / ドロップ
                 │
                 ▼
  [ Local Management Daemon (SSHd/HTTPd) ]
  ```
* **注意点**:
  * ラボ試験で MPP や ASA 管理アクセスルールを設定する際は、**「現在自分が接続している SSH 端末の IP / インターフェイス」を誤って拒否しないよう**、設定投入前に必ず自分の着信ポートと送信元 IP アドレスを確認してください。


