---
layout: default
title: 4.3-Admin-access
nav_order: 3
parent: 4.0-Identity-Management
---

# 4.3 Cisco devices for administrative access with Cisco ISE

## 📘 概要

* **機能概要**: Cisco Identity Services Engine (ISE) を Centralized TACACS+ (TACACS+ Security Daemon) および RADIUS サーバーとして活用し、Cisco IOS-XE スイッチ/ルータ、Cisco ASA ファイアウォール、Cisco FTD (Firepower Threat Defense) / FMC (Firepower Management Center)、WLC、および Secure Email/Web Gateway 等のネットワーク機器に対する管理アクセス（Administrative Access / Device Administration）の「認証 (Authentication)」「認可 (Authorization)」「事後監査・ログ記録 (Accounting)」を一元管理する機能。
* **利用目的**: ネットワーク機器へのログイン時において、個別ローカルアカウントの分散管理を廃止し、Active Directory (AD) や LDAP 等の中央 ID プロバイダと連携した RBAC (Role-Based Access Control) を実現する。ユーザー権限レベル（Privilege Level 0〜15、または特定の CLI コマンドの実行許可/拒否）を中央の Policy Set で制御し、誰が・いつ・どの機器で・どのコマンドを実行したかを TACACS+ Accounting ログとして記録・監査する。
* **どのような場面で利用するか**: 
  - 大規模企業やエンタープライズネットワークにおいて、管理者の離着任に伴うローカルアカウント更新の手間やセキュリティリスク（暗号化されていない共通パスワードの使い回し）を削減する場合。
  - 特権管理者（Privilege Level 15）、運用監視担当者（Read-Only Privilege 1）、ネットワークセキュリティオペレータ（特定 show コマンドおよび Interface コマンドのみ許可）など、職務に応じた最小権限原則（Least Privilege Principle）を適用する場合。
  - PCI-DSS や ISO 27001 などのセキュリティ要件に準拠し、すべての特権 CLI 操作ログの不改ざん追跡（Command Accounting）を義務付ける環境。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **特徴** | TACACS+ プロトコル (TCP 49) を主軸とし、ヘッダーを除くパケット全体（ボディ）を暗号化。認証・認可・会計処理が完全に分離されており、コマンド単位のリアルタイム認可 (Command Authorization) が可能。 |
| **用途** | ネットワーク機器（IOS-XE, ASA, FTD/FMC, WLC 等）への CLI (SSH/Console) および Web GUI (HTTPS) 管理アクセスの統合認証・制御・監査。 |
| **メリット** | コマンド単位での厳格なアクセスコントロール、Active Directory/LDAP/ISE Local Users とのシームレスな連携、詳細な操作ログの集約、Privilege Level や Cisco AV-Pair による柔軟な権限付与。 |
| **デメリット** | ISE で TACACS+ 機能を有効化するためには Device Admin ライセンスが必要。WAN 障害時や ISE 停止時に管理者アクセスがロックアウトされるリスクがあるため、fallback（ローカル認証）の適切な設計が不可欠。 |
| **対応機種** | Cisco Catalyst スイッチ, ISR/ASR/Catalyst ルータ (IOS/IOS-XE), Cisco ASA (9.x以降), Cisco FTD/FMC (6.x/7.x以降), Cisco WLC (AireOS / Catalyst 9800), ESA/WSA/SMA。 |
| **制限事項** | RADIUS は UDP (1812/1813) を使用しパスワードのみ暗号化するため、コマンド単位の認可処理（Command Authorization）には不向き。RADIUS による管理アクセスはシェルレベル（Privilege 15 等）の付与に限定される。 |
| **設計上の注意点** | ネットワーク障害時のデッドロックを防ぐため、`aaa authentication login default group group-name local` や ASA の `LOCAL` フォールバック、および Tacacs+ Server Timeout/Retransmit のパラメータ最適化が必須。 |

---

## 🏗 動作原理

### 通信フロー

Cisco ネットワーク機器における SSH CLI ログインおよびコマンド実行時の TACACS+ / ISE 連携フローを図示します。

```text
  [ Admin User ]
        │
        │ 1. SSH / Console Access (Username/Password)
        ▼
┌──────────────────┐
│ Cisco Device     │
│ (IOS-XE/ASA/FTD) │
└────────┬─────────┘
         │
         │ 2. TACACS+ Authentication Request (TCP 49) [Encrypted Body]
         ▼
┌──────────────────┐
│  Cisco ISE       │ 3. External Identity Lookup (Kerberos / LDAP / MS-RPC)
│  (Device Admin)  ├───────────────────────────────────────────────────────► ┌──────────────┐
└────────┬─────────┘                                                         │ Active       │
         │ 4. Authentication SUCCESS + Challenge/Response                    │ Directory    │
         │                                                                   └──────────────┘
         │ 5. TACACS+ Authorization Request (Privilege Level / Command)
         ├───────────────────────────────────────────────────────┐
         │ 6. TACACS+ Authorization PASS (Shell Profile / Command Set)   │
         ▼                                                       │
┌──────────────────┐                                             │
│ Cisco Device     │◄────────────────────────────────────────────┘
│ (Execute Command)│
└────────┬─────────┘
         │ 7. TACACS+ Accounting Request (Command logged with timestamp)
         ▼
┌──────────────────┐
│  Cisco ISE (MnT) │ (Stored in MnT Database & Forwarded to Syslog Server)
└──────────────────┘
```

---

## ⚙ 動作シーケンス

1. **TCP セッション確立 (TCP Port 49)**:
   - 管理者が SSH または Console 経由で Cisco デバイスにログインを試みると、デバイスは指定された Cisco ISE ノードの IP アドレスに対し、TCP ポート 49 で 3-way ハンドシェイクを完了します。

2. **TACACS+ Authentication (認証段階)**:
   - デバイスは 共有鍵（Shared Secret）を用いて暗号化した `TAC_PLUS_AUTHEN_START` パケットを ISE に送信します。
   - ISE は認証ポリシー（Authentication Policy）に基づき、入力されたユーザー名・パスワードを ISE 内部データベースまたは Active Directory / LDAP と照合します。
   - 認証成功時、ISE は `TAC_PLUS_AUTHEN_STATUS_PASS` をデバイスに返答します。

3. **TACACS+ Shell Authorization (初期シェル認可段階)**:
   - 認証が成功すると、デバイスは直ちに `TAC_PLUS_AUTHOR_REQUEST` (service=shell) を送信します。
   - ISE は認可ポリシー（Authorization Policy Set）を評価し、対応する **Shell Profile**（例: Privilege Level 15 や privilege level 1）を決定して `TAC_PLUS_AUTHOR_STATUS_PASS_ADD` とともに属性（cisco-av-pair="priv-lvl=15" 等）を返却します。

4. **TACACS+ Command Authorization (コマンド単位の認可段階)**:
   - 管理者が CLI で特定のコマンド（例: `configure terminal`, `reload`, `show running-config`）を入力するたびに、デバイスは `TAC_PLUS_AUTHOR_REQUEST` (service=shell, cmd=configure, cmd-arg=terminal) を ISE にリアルタイム送信します。
   - ISE は構成された **Command Set**（許可コマンドリスト・正規表現ルール）と照合し、許可 (`PASS`) または拒否 (`FAIL`) を返します。拒否された場合、CLI 上には `% Command authorization failed.` と表示されます。

5. **TACACS+ Accounting (事後監査・ログ記録段階)**:
   - コマンド実行が完了すると、デバイスは `TAC_PLUS_ACCT_REQUEST` パケットを ISE の MnT (Monitoring and Troubleshooting) ペルソナに送信します。
   - パケットには、ユーザー名、端末 IP、実行日時、対象デバイス、および実行された完全なコマンド文字列が含まれ、ISE の Audit Report に記録されます。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Securityラボ試験における 「Cisco devices for administrative access with Cisco ISE」 の重点ポイントおよび落とし穴を以下に整理します。

### Blueprintで重要なポイント
1. **TACACS+ vs RADIUS の使い分け**:
   - Device Administration（機器管理アクセス）には原則 **TACACS+** を使用します。試験問題で「各コマンドの実行を中央で検証・許可し、すべての操作ログを記録すること」と指示された場合、Command Authorization が可能な TACACS+ 構成が必須となります。
   - RADIUS を使用した場合、ログイン時の一括 Shell 権限付与（priv-lvl=15）は可能ですが、個別の CLI コマンド単位の認可制御（Command Authorization）は不可能です。

2. **Cisco ISE での Device Admin Service (TACACS+) 有効化**:
   - ISE 側で `Administration > System > Deployment` から対象ノードを開き、**Enable Device Admin Service** チェックボックスをオンにする必要があります。これを忘れると、ISE は TCP 49 リクエストを受信しても応答せず、認証がタイムアウトします。

3. **ISE Policy Sets (Device Admin Policy Sets) の階層設計**:
   - `Work Centers > Device Administration > Device Admin Policy Sets` にて設定します。
   - Outer Policy（Match 条件: Network Device Group、Device Type、IP アドレス等）と Inner Policy (Authentication Policy / Authorization Policy) の評価順序と条件分岐を正確に構築する必要があります。

4. **ローカルフォールバック (Fallback to Local) の確実な実装**:
   - ラボ試験では、「ISE が疎通不能になった場合でも Console/SSH からローカルアカウントで緊急ログインできること」という条件が付与されます。
   - IOS-XE では `aaa authentication login default group ISE_TACACS local` や `aaa authentication enable default group ISE_TACACS enable` の順序設定。
   - ASA では `aaa-server` の timeout/retransmits 調整および `aaa authentication ssh console ISE_TACACS LOCAL`。

---

### よくある設定ミスと試験での狙われ目
* **IOS-XE で `aaa authorization exec` や `aaa authorization commands` の設定漏れ**:
  - `aaa authentication login` のみを設定し、`aaa authorization exec default group ISE_TACACS local` や `aaa authorization commands 15 default group ISE_TACACS local` を忘れると、ログインは成功するものの Privilege Level 1 の一般権限に降格したり、コマンド実行時の認可要求が ISE に飛ばずローカル実行されて要件違反になります。
* **`aaa authorization console` の指定忘れ**:
  - デフォルトでは Cisco IOS-XE は Console ポートに対する Authorization を免除（Bypass）します。試験問題で「Console 接続においても ISE によるコマンド認可を強制すること」と書かれている場合、グローバル設定で `aaa authorization console` を追加しなければなりません。
* **ISE TACACS+ Shared Secret のミスマッチ**:
  - Network Device の設定値と ISE Network Resources > Network Devices で登録した Shared Secret が一致しない場合、ISE ログに `15013 TACACS+ Shared Secret mismatch` エラーが記録されます。
* **ASA における `aaa authorization command` の構文誤り**:
  - ASA では命令体系が異なり、`aaa authorization command LOCAL` または `aaa authorization command TACACS_GROUP` を使用してインターフェースやプロトコルごとにバインドします。

---

## 🛠 設定方法

### 1. Cisco ISE 側での TACACS+ 設定手順 (GUI)

#### Step 1: Device Admin Service の有効化
1. ISE Web GUI にて `Administration > System > Deployment` に移動。
2. 対象ノードを選択し、`Edit` をクリック。
3. `General Settings` タブの **Enable Device Admin Service** にチェックを入れ、`Save` をクリック。

#### Step 2: ネットワーク機器 (Network Devices) の登録
1. `Work Centers > Device Administration > Network Resources > Network Devices` に移動。
2. `+ Add` をクリックし、名前（例: `SW-Core-01`）、IP アドレス（例: `10.1.1.1/32` またはサブネット）を入力。
3. `TACACS+ Authentication Settings` にチェックを入れ、`Shared Secret`（例: `TacacsKey123`）を入力して保存。

#### Step 3: TACACS+ Command Sets の作成
1. `Work Centers > Device Administration > Policy Elements > Results > TACACS Command Sets` に移動。
2. `+ Add` をクリックし、名前（例: `Operator_Permitted_Commands`）を入力。
3. `Permit any command not listed below` のチェックを外し、個別許可ルールを設定:
   - Command: `show`, Arguments: `*` (Permit)
   - Command: `ping`, Arguments: `*` (Permit)
   - Command: `traceroute`, Arguments: `*` (Permit)
4. 管理者用（`Full_Access_Command_Set`）の場合は `Permit any command not listed below` にチェックを入れ保存。

#### Step 4: TACACS+ Shell Profiles の作成
1. `Work Centers > Device Administration > Policy Elements > Results > TACACS Shell Profiles` に移動。
2. `+ Add` をクリックし、名前（例: `Priv15_Shell_Profile`）を入力。
3. `General Attributes` タブにて `Default Privilege` を `15`、`Maximum Privilege` を `15` に設定し保存。
4. 監視オペレータ用（`Priv1_Shell_Profile`）の場合は `Default Privilege` を `1` に設定。

#### Step 5: Device Admin Policy Sets の構成
1. `Work Centers > Device Administration > Device Admin Policy Sets` に移動。
2. `+` をクリックして新しい Policy Set（例: `Device_Admin_Policy`）を作成。
3. Conditions: `Device Context:Device Type EQUALS All Device Types`
4. Policy Set を展開し、`Authentication Policy` を設定:
   - Rule 1: `All_Users` -> Use `Active_Directory` (or `Internal Users`).
5. `Authorization Policy` を設定:
   - Rule 1 (Network Admins): If `AD:ExternalGroups EQUALS corp.local/Network_Admins` -> Shell Profile: `Priv15_Shell_Profile`, Command Sets: `Full_Access_Command_Set`.
   - Rule 2 (Operators): If `AD:ExternalGroups EQUALS corp.local/Operators` -> Shell Profile: `Priv1_Shell_Profile` (Priv 1), Command Sets: `Operator_Permitted_Commands`.

---

### 2. Cisco IOS-XE ルータ / スイッチにおける CLI 設定例

```bash
! --- 1. AAAの有効化とTACACS+サーバーグループの定義 ---
aaa new-model

tacacs server ISE-PRIMARY
 address ipv4 192.168.1.100
 key 0 TacacsKey123
 timeout 5
 single-connection

tacacs server ISE-SECONDARY
 address ipv4 192.168.1.101
 key 0 TacacsKey123
 timeout 5
 single-connection

aaa group server tacacs+ ISE_TACACS_GROUP
 server name ISE-PRIMARY
 server name ISE-SECONDARY
 ip tacacs source-interface GigabitEthernet0/0

! --- 2. AAA Authentication (ログイン認証 & Enable認証) ---
aaa authentication login default group ISE_TACACS_GROUP local
aaa authentication enable default group ISE_TACACS_GROUP enable

! --- 3. AAA Authorization (Execシェル認可 & コマンド認可) ---
aaa authorization exec default group ISE_TACACS_GROUP local 
aaa authorization commands 15 default group ISE_TACACS_GROUP local
aaa authorization commands 1 default group ISE_TACACS_GROUP local
aaa authorization config-commands
aaa authorization console

! --- 4. AAA Accounting (ログイン会計 & コマンド実行会計) ---
aaa accounting exec default start-stop group ISE_TACACS_GROUP
aaa accounting commands 15 default start-stop group ISE_TACACS_GROUP

! --- 5. VTYおよびConsoleラインへの適用 ---
line console 0
 login authentication default
 authorization exec default
 authorization commands 15 default
line vty 0 15
 exec-timeout 15 0
 login authentication default
 authorization exec default
 authorization commands 15 default
 transport input ssh
```

---

### 3. Cisco ASA (Adaptive Security Appliance) における CLI 設定例

```bash
! --- 1. TACACS+ サーバーグループの設定 ---
aaa-server ISE_TACACS_GROUP protocol tacacs+
aaa-server ISE_TACACS_GROUP (inside) host 192.168.1.100
 key TacacsKey123
 timeout 5
aaa-server ISE_TACACS_GROUP (inside) host 192.168.1.101
 key TacacsKey123
 timeout 5

! --- 2. SSH / TELNET / CONSOLE 認証の設定 ---
aaa authentication ssh console ISE_TACACS_GROUP LOCAL
aaa authentication serial console ISE_TACACS_GROUP LOCAL
aaa authentication enable console ISE_TACACS_GROUP LOCAL

! --- 3. Exec シェルおよび Privilege Level 認可の設定 ---
aaa authorization exec authentication-server

! --- 4. コマンド認可 (Command Authorization) の設定 ---
aaa authorization command ISE_TACACS_GROUP LOCAL

! --- 5. Accounting (会計) の設定 ---
aaa accounting ssh ISE_TACACS_GROUP
aaa accounting command ISE_TACACS_GROUP
```

---

### 4. Cisco FTD (Firepower Threat Defense) / FMC 管理アクセス設定 (FMC Platform Settings)

FMC から FTD へ TACACS+ 管理アクセス設定を配布する手順:

1. FMC GUI にて `Devices > Platform Settings` に移動。
2. 対象の FTD ポリシーを編集（または新規作成）。
3. 左メニューの `External Authentication` を選択。
4. `+ Add` をクリックし、`Server Group` を定義:
   - Name: `ISE_TACACS_FTD`
   - Protocol: `TACACS+`
   - Primary Server: `192.168.1.100`, Secret: `TacacsKey123`
   - Secondary Server: `192.168.1.101`, Secret: `TacacsKey123`
5. `Shell Access Authorization` を有効化し、認可失敗時の Fallback として `Local Authentication` を選択。
6. 設定を保存し、対象 FTD デバイスへ **Deploy** を実行。


## 🔍 検証コマンド

ネットワーク機器および Cisco ISE 上で AAA/TACACS+ 状態・ログを確認するための主要コマンド一覧です。

| 目的 | コマンド |
| --- | --- |
| **TACACS+ サーバー状態確認** | `<code>show tacacs</code>` |
| **AAA 統計およびリクエスト確認** | `<code>show aaa servers</code>` |
| **アクティブなユーザーセッション表示** | `<code>show users</code>` または `<code>show ssh</code>` |
| **ASA TACACS+ サーバー応答確認** | `<code>show aaa-server ISE_TACACS_GROUP</code>` |
| **ASA 認証テスト実行** | `<code>test aaa-server authentication ISE_TACACS_GROUP host 192.168.1.100 username admin password Ciscopass123</code>` |
| **IOS-XE TACACS+ パケットデバッグ** | `<code>debug tacacs authentication</code>`, `<code>debug tacacs authorization</code>`, `<code>debug tacacs accounting</code>` |
| **IOS-XE AAA トラブルシュート** | `<code>debug aaa authentication</code>`, `<code>debug aaa authorization</code>` |
| **ASA AAA デバッグ** | `<code>debug aaa authentication</code>`, `<code>debug aaa authorization</code>`, `<code>debug aaa-server active</code>` |
| **ISE リアルタイム監査ログ** | Web GUI: `Operations > TACACS > Live Logs` |
| **ISE 詳細レポーティング** | Web GUI: `Operations > Reports > Reports > TACACS Authentication` |

---

## 🚨 トラブルシュート

管理アクセスにおける一般的な障害症状、原因、確認コマンド、および対処方法を下表に整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **ユーザー名・パスワードを入力しても認証がタイムアウトする** | ネットワーク機器と ISE 間の IP 疎通不良、または ISE の Device Admin Service (TACACS+) 未有効化。 | `<code>show tacacs</code>`, `<code>ping 192.168.1.100</code>`, ISE: `Deployment Settings` | ISE で Device Admin Service を有効化し、ファイヤーウォール等で TCP ポート 49 が許可されているか確認する。 |
| **`TACACS+ Shared Secret mismatch` エラーが記録される** | デバイスの `key` と ISE `Network Devices` で登録した Shared Secret の不一致。 | `<code>show running-config \| section tacacs</code>`, ISE Live Logs: Failure Code `15013` | デバイス側 `key` と ISE 側の共有鍵を完全に一致させる。 |
| **ログインは成功するが Privilege Level 1 で入ってしまう** | ISE 認可ポリシーの Shell Profile で `Default Privilege 15` が指定されていない。 | `<code>show privilege</code>`, ISE TACACS Shell Profiles | Shell Profile の `Default Privilege` および `Maximum Privilege` を `15` に修正する。 |
| **コマンドを実行すると `% Command authorization failed` となる** | ISE の Command Set で該当コマンドが許可されていない、または `Permit unlisted commands` がオフ。 | `<code>show running-config \| inc aaa authorization</code>`, ISE TACACS Command Sets | Command Set で対象コマンド（例: `show *` や `configure terminal`）を明示的に Permit ルールに追加する。 |
| **ISE 停止時に Console/SSH ログインが拒否されロックアウトされる** | AAA リストで `local` または `LOCAL` フォールバックが指定されていない。 | `<code>show running-config \| inc aaa authentication</code>` | `aaa authentication login default group ISE_TACACS_GROUP local` のように末尾に `local` を追記する。 |
| **Console ポートでコマンド認可が機能しない** | IOS-XE で `aaa authorization console` が設定されていない。 | `<code>show running-config \| inc aaa authorization console</code>` | グローバルコンフィグで `aaa authorization console` を投入する。 |

---

## ⚠ 制限事項

* **ライセンス要件**: Cisco ISE で TACACS+ Device Administration 機能を利用するには、ノード数に応じた **ISE Device Admin License** (旧 TACACS+ License) が別途必要です。Base/Essential ライセンスのみでは TACACS+ 機能は有効化できません。
* **RADIUS プロトコルの認可制限**: RADIUS ではセッション開始時に認証・認可が同時に行われるため、ログイン後の個別コマンド認可 (Command Authorization) やリアルタイムコマンド制限は不可能です。
* **暗号化範囲の差異**: RADIUS は Access-Request パケット内の User-Password 属性のみを MD5 シークレットで暗号化しますが、TACACS+ はパケットヘッダー（8バイト）を除く全ボディ（ユーザー名、属性、パケットペイロード）を暗号化します。

---

## 🔄 他技術との関連

* **Active Directory / LDAP 統合**: ISE を AD または LDAP サーバーと連携させることで、管理者の人事異動や退職に伴うアカウント無効化が ISE およびネットワーク機器へ即座に反映されます。
* **Syslog / SIEM 連携**: ISE MnT から送出される TACACS+ Accounting ログ（TCP 49 で受信されたコマンドログ）を Splunk や Radar 等の SIEM に転送し、操作監査や特権乱用検知に利用します。
* **Management Plane Protection (MPP)**: IOS-XE の MPP 機能を併用し、特定管理用インターフェース（例: GigabitEthernet0/0）のみ SSH および TACACS+ 通信を許可することで制御プレーンを防衛します。

---

## 🧩 比較表

### 1. TACACS+ vs RADIUS (管理アクセス観点)

| 比較項目 | TACACS+ (RFC 8907) | RADIUS (RFC 2865/2866) |
| --- | --- | --- |
| **トランスポート** | TCP ポート 49 (信頼性の高い接続) | UDP ポート 1812 (認証) / 1813 (会計) |
| **暗号化** | パケットボディ全体（ヘッダー以外）を暗号化 | パスワード属性のみ暗号化（全体は平文） |
| ** AAA の分離** | 認証・認可・会計が完全に独立 | 認証と認可が結合 (Combined) |
| **コマンド認可** | コマンド単位のリアルタイム認可が可能 | コマンド認可不可（ログイン時一括権限のみ） |
| **主な用途** | ネットワーク機器の管理アクセス (Device Admin) | 端末のネットワーク接続認証 (802.1X / Dot1x / VPN) |

### 2. IOS-XE vs ASA vs FTD (TACACS+ 設定体系の比較)

| プラットフォーム | TACACS+ サーバー定義 | 認証・認可バインド | コマンド認可の指定方法 |
| --- | --- | --- | --- |
| **Cisco IOS-XE** | `tacacs server <name>` & `aaa group server tacacs+` | `aaa authentication login default group <name> local` | `aaa authorization commands 15 default group <name> local` |
| **Cisco ASA** | `aaa-server <group> protocol tacacs+` | `aaa authentication ssh console <group> LOCAL` | `aaa authorization command <group> LOCAL` |
| **Cisco FTD** | FMC Platform Settings (`External Authentication`) | Platform Settings Web GUI または FlexConfig | Platform Settings GUI (`Shell Access Authorization`) |

---

## 💡 ベストプラクティス

1. **Single-Connection モードの活用**: Cisco IOS-XE では TACACS+ サーバー設定に `single-connection` オプションを追加します。これにより、コマンド認可のたびに TCP セッションを再確立するオーバーヘッドを削減し、スイッチ・ISE 双方の CPU 負荷を軽減します。
2. **ローカル緊急アカウント（Fallback Account）の厳格管理**: `local` フォールバック用に機器ローカルに作成する非常用アカウント（例: `emergency-admin`）は、無作為生成された複雑なパスワード（Secret Type 9 等）で保護し、金庫管理等で厳重に保管します。
3. **ソースインターフェースの固定**: `ip tacacs source-interface <Interface>` を設定し、ルーティング変更によって TACACS+ パケットの送信元 IP が変動し ISE 側で Network Device 登録エラーが発生するのを防止します。


## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルを意識した、Cisco 機器における ISE 管理アクセス（TACACS+ / AAA）の10個の実践演習シナリオです。

### シナリオ 1: Cisco IOS-XE における基本 TACACS+ ログイン認証と Priv 15 シェル認可（ローカルフォールバック付き）

#### 問題
スイッチ SW-CORE-1 において、SSH ログイン時に Cisco ISE による TACACS+ 認証および Privilege Level 15 の Exec シェル認可を実装しなさい。ISE 障害時にはローカルデータベースで緊急ログインできるように設定しなさい。

#### 要件
- TACACS+ サーバー IP: Primary `192.168.10.50`, Secondary `192.168.10.51`
- TACACS+ 共有鍵: `TacacsKeyPass123`
- ソースインターフェース: `Loopback0`
- 認証・認可失敗時のフォールバックとしてローカルデータベース (`local`) を使用する。

#### 設定例

```bash
! --- IOS-XE (SW-CORE-1) --- 
aaa new-model

tacacs server ISE-TACACS-1
 address ipv4 192.168.10.50
 key 0 TacacsKeyPass123
 timeout 5
 single-connection

tacacs server ISE-TACACS-2
 address ipv4 192.168.10.51
 key 0 TacacsKeyPass123
 timeout 5
 single-connection

aaa group server tacacs+ ISE-TACACS-GROUP
 server name ISE-TACACS-1
 server name ISE-TACACS-2
 ip tacacs source-interface Loopback0

aaa authentication login default group ISE-TACACS-GROUP local
aaa authorization exec default group ISE-TACACS-GROUP local

line vty 0 15
 login authentication default
 authorization exec default
 transport input ssh
```

#### 検証方法
- `show tacacs` を実行し、両 ISE サーバーがアクティブであることを確認する。
- SSH クライアントから SW-CORE-1 へログインを試み、`show privilege` で即座に Privilege Level 15 が付与されていることを確認する。
- ISE Live Logs (`Operations > TACACS > Live Logs`) で `Authentication PASS` および `Authorization PASS` (priv-lvl=15) を確認する。

---

### シナリオ 2: 監視オペレータ用制限アクセス（Privilege Level 1 + コマンド認可制限）

#### 問題
オペレータユーザー（`operator1`）に対して Privilege Level 1 を割り当て、実行可能なコマンドを `show` / `ping` / `traceroute` のみに限定するリアルタイムコマンド認可を導入しなさい。

#### 要件
- コマンド認可リスト名: `default`
- ISE 上で Command Set: `Operator_Cmd_Set` を構成し、`show`, `ping`, `traceroute` のみ Permit とする。
- IOS-XE 側で Privilege Level 1 のコマンド認可を強制する。

#### 設定例

```bash
! --- IOS-XE --- 
aaa authorization commands 1 default group ISE-TACACS-GROUP local

line vty 0 15
 authorization commands 1 default
```

```text
# --- ISE Web GUI (TACACS Command Set) ---
Name: Operator_Cmd_Set
Permit any command not listed below: UNCHECKED
Command Grants:
  1. Grant: PERMIT, Command: show, Arguments: *
  2. Grant: PERMIT, Command: ping, Arguments: *
  3. Grant: PERMIT, Command: traceroute, Arguments: *
```

#### 検証方法
- `operator1` でログイン後、`show ip interface brief` や `ping 8.8.8.8` が正常実行できることを確認する。
- `configure terminal` や `reload` を実行し、`% Command authorization failed.` と拒否されることを確認する。

---

### シナリオ 3: Console ポート制御および設定モードコマンド認可の強化

#### 問題
SW-CORE-1 の Console 接続においても ISE によるコマンド認可を回避できないようにし、かつ Global Configuration モード内のすべての設定変更コマンドを ISE にて承認・監査させなさい。

#### 要件
- `aaa authorization console` を有効化する。
- `aaa authorization config-commands` を有効化する。
- Console 0 ラインにも CLI 認可設定を適用する。

#### 設定例

```bash
! --- IOS-XE --- 
aaa authorization console
aaa authorization config-commands
aaa authorization commands 15 default group ISE-TACACS-GROUP local

line console 0
 login authentication default
 authorization exec default
 authorization commands 15 default
```

#### 検証方法
- Console 経由でログインし、`configure terminal` から `interface GigabitEthernet0/1` -> `shutdown` を実行する。
- ISE の `Live Logs` にて、`service=shell`, `cmd=interface`, `cmd-arg=GigabitEthernet0/1` および `cmd=shutdown` が個別に Authorization PASS されていることを確認する。

---

### シナリオ 4: TACACS+ Single-Connection および Source Interface の最適化

#### 問題
マルチホーム接続されたコアスイッチ SW-CORE-2 において、ISE ノードとの TACACS+ 通信効率を高めるため Single-Connection を設定し、送信元 IP を安定化させなさい。

#### 要件
- TACACS+ サーバー `192.168.10.50` および `192.168.10.51` に対して `single-connection` を指定する。
- サーバーグループレベルで `ip tacacs source-interface Loopback0` を指定する。

#### 設定例

```bash
! --- IOS-XE --- 
tacacs server ISE-1
 address ipv4 192.168.10.50
 key 0 TacacsKeyPass123
 single-connection

tacacs server ISE-2
 address ipv4 192.168.10.51
 key 0 TacacsKeyPass123
 single-connection

aaa group server tacacs+ TACACS-HQ
 server name ISE-1
 server name ISE-2
 ip tacacs source-interface Loopback0
```

#### 検証方法
- コマンドを連続実行した際、`show tacacs` の出力にて `Socket handle` が維持され、接続再確立のオーバーヘッドなしに単一の TCP セッションで処理されていることを確認する。

---

### シナリオ 5: Cisco ASA における SSH ログイン認証および Privilege 15 コマンド認可

#### 問題
Cisco ASA ファイアウォールにおいて、SSH ログイン時に ISE TACACS+ で認証し、コマンド実行時の認可・会計を記録しなさい。

#### 要件
- TACACS+ サーバーグループ名: `ISE_TACACS_ASA`
- 認証失敗または障害時はローカル DB (`LOCAL`) を使用する。
- コマンド認可 (`aaa authorization command`) を有効化する。

#### 設定例

```bash
! --- Cisco ASA --- 
aaa-server ISE_TACACS_ASA protocol tacacs+
aaa-server ISE_TACACS_ASA (inside) host 192.168.10.50
 key TacacsKeyPass123
 timeout 5
aaa-server ISE_TACACS_ASA (inside) host 192.168.10.51
 key TacacsKeyPass123
 timeout 5

aaa authentication ssh console ISE_TACACS_ASA LOCAL
aaa authentication serial console ISE_TACACS_ASA LOCAL
aaa authentication enable console ISE_TACACS_ASA LOCAL
aaa authorization exec authentication-server
aaa authorization command ISE_TACACS_ASA LOCAL
aaa accounting ssh ISE_TACACS_ASA
aaa accounting command ISE_TACACS_ASA
```

#### 検証方法
- ASA CLI にて `test aaa-server authentication ISE_TACACS_ASA host 192.168.10.50 username admin password AdminPass123` を実行し、`INFO: Authentication Successful` を確認する。


### シナリオ 6: Cisco FTD / FMC における Platform Settings 経由の TACACS+ 外部認証連携

#### 問題
FMC によって管理される Cisco FTD において、CLI 接続時の管理アクセス認証を ISE TACACS+ に統合しなさい。

#### 要件
- FMC Platform Settings ポリシーを作成し、FTD デバイスに割り当てる。
- TACACS+ サーバーグループ `FTD_TACACS_GRP` を作成し、`192.168.10.50` および `192.168.10.51` を登録する。
- 障害時は `Local Authentication` にフォールバックする。

#### 設定手順 (FMC Web GUI & Generated CLI)

1. FMC GUI にて `Devices > Platform Settings` に移動。
2. 新規 FTD Policy を作成し、`External Authentication` タブを開く。
3. `Add Server Group` を選択:
   - Name: `FTD_TACACS_GRP`
   - Protocol: `TACACS+`
   - Primary Server: `192.168.10.50`, Key: `TacacsKeyPass123`
   - Secondary Server: `192.168.10.51`, Key: `TacacsKeyPass123`
4. `Shell Access Authorization` にチェックを入れ、Fallback 用に `Local` を指定して **Save & Deploy**。

```bash
! --- FTD 側で生成される内部 CLI コンフィグ確認 --- 
aaa-server FTD_TACACS_GRP protocol tacacs+
aaa-server FTD_TACACS_GRP host 192.168.10.50
 key TacacsKeyPass123
aaa-server FTD_TACACS_GRP host 192.168.10.51
 key TacacsKeyPass123
aaa authentication ssh console FTD_TACACS_GRP LOCAL
```

#### 検証方法
- SSH クライアントから FTD の管理 IP アドレスに接続し、ISE ユーザーアカウントで正常ログインできることおよび ISE 監査ログに成功履歴が残ることを確認する。

---

### シナリオ 7: ISE Device Admin Policy Set における Active Directory グループ連携（Role-Based Access Control）

#### 問題
Active Directory (AD) のグループ情報に基づいて、`Network_Admins` グループ所属ユーザーには Privilege Level 15 (無制限)、`Sec_Auditors` グループ所属ユーザーには Privilege Level 1 (Read-Only 許可コマンド限定) を動的割り当てしなさい。

#### 要件
- ISE 上で AD ジョインを確立済みであること。
- 2つの TACACS Shell Profile (`Priv15_Profile`, `Priv1_Profile`) を作成。
- 2つの TACACS Command Set (`All_Permit_CmdSet`, `Auditor_Permit_CmdSet`) を作成。
- Policy Set 内で AD グループ条件により振り分ける。

#### 設定例 (ISE Policy Set 構成概要)

```text
Policy Set: HQ_Device_Admin_Policy
  Condition: Device Context:Device Type EQUALS Network_Devices
  
  Authentication Policy:
    Rule 1: Match All -> Use AD_Store (Active Directory)
  
  Authorization Policy:
    Rule 1 (Admins):
      Condition: AD:ExternalGroups EQUALS corp.local/Users/Network_Admins
      Command Sets: All_Permit_CmdSet
      Shell Profile: Priv15_Profile (priv-lvl=15)
    
    Rule 2 (Auditors):
      Condition: AD:ExternalGroups EQUALS corp.local/Users/Sec_Auditors
      Command Sets: Auditor_Permit_CmdSet (show only)
      Shell Profile: Priv1_Profile (priv-lvl=1)
```

#### 検証方法
- AD ユーザー `admin_user` でログインすると Priv 15 かつ任意コマンドが実行可能であることを確認する。
- AD ユーザー `auditor_user` でログインすると Priv 1 に固定され、`show` 以外のコマンドが失敗することを確認する。

---

### シナリオ 8: ISE サーバー疎通断時における非常用ローカルアカウント自動切替（Fallback 動作テスト）

#### 問題
すべての TACACS+ サーバー（ISE-1, ISE-2）がダウンした場合、スイッチ上の緊急アカウント `admin-local` で Console/VTY からログインできるように動作を検証しなさい。

#### 要件
- ローカルに Secret Type 9 で緊急アカウントを事前登録する。
- AAA リストの末尾に `local` を明記する。
- タスク検証用に TACACS+ サーバーのタイムアウトパラメータを短縮調整する。

#### 設定例

```bash
! --- IOS-XE --- 
username admin-local privilege 15 algorithm-type scrypt secret EmergencyPass999

tacacs server ISE-1
 address ipv4 192.168.10.50
 timeout 2
 retransmit 1

tacacs server ISE-2
 address ipv4 192.168.10.51
 timeout 2
 retransmit 1

aaa authentication login default group ISE-TACACS-GROUP local
aaa authorization exec default group ISE-TACACS-GROUP local
```

#### 検証方法
- ルータ/スイッチの出力を切り離すか ACL で TCP 49 をブロックする。
- SSH ログイン時に `admin-local` / `EmergencyPass999` を入力し、数秒のタイムアウト後に即座にログイン成功することを確認する。

---

### シナリオ 9: PCI-DSS 準拠のための TACACS+ コマンド会計ログの完全記録設定

#### 問題
スイッチ上で実行されたすべての Privilege 15 コマンドおよび Privilege 1 コマンドの開始・終了ログ（Start-Stop Accounting）を ISE へ送信しなさい。

#### 要件
- `aaa accounting commands 15 default start-stop group ISE-TACACS-GROUP`
- `aaa accounting commands 1 default start-stop group ISE-TACACS-GROUP`
- `aaa accounting exec default start-stop group ISE-TACACS-GROUP`

#### 設定例

```bash
! --- IOS-XE --- 
aaa accounting exec default start-stop group ISE-TACACS-GROUP
aaa accounting commands 1 default start-stop group ISE-TACACS-GROUP
aaa accounting commands 15 default start-stop group ISE-TACACS-GROUP
aaa accounting system default start-stop group ISE-TACACS-GROUP
```

#### 検証方法
- SSH ログイン後、`configure terminal` -> `interface vlan 100` -> `ip address 10.100.1.1 255.255.255.0` を実行する。
- ISE Web GUI の `Operations > TACACS > Live Logs` または `Reports > TACACS Accounting` を開き、各コマンドがミリ秒単位のタイムスタンプとともに記録されていることを確認する。

---

### シナリオ 10: Out-of-Band (OOB) 管理 VRF 統合による RADIUS 管理バックアップ構成

#### 問題
管理トラフィックが専用 VRF (`Mgmt-vrf`) に分離されたルータにおいて、VRF 経由の RADIUS 管理バックアップ認証を構築しなさい。

#### 要件
- RADIUS サーバー: `10.254.1.50`, Secret: `RadiusSecretKey1`
- VRF 名: `Mgmt-vrf`
- RADIUS 認可により `cisco-av-pair="shell:priv-lvl=15"` を受信してログインさせる。

#### 設定例

```bash
! --- IOS-XE --- 
radius server RADIUS-OOB
 address ipv4 10.254.1.50 auth-port 1812 acct-port 1813
 key RadiusSecretKey1

aaa group server radius RADIUS-OOB-GROUP
 server name RADIUS-OOB
 use-vrf Mgmt-vrf
 ip radius source-interface GigabitEthernet0/0

aaa authentication login OOB-LOGIN group RADIUS-OOB-GROUP local
aaa authorization exec OOB-EXEC group RADIUS-OOB-GROUP local

line vty 0 4
 login authentication OOB-LOGIN
 authorization exec OOB-EXEC
```

#### 検証方法
- `show aaa servers` を実行し、`RADIUS-OOB-GROUP` が `Mgmt-vrf` 内で正しくリクエストを送受信していることを確認する。


## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験およびテクノロジー評価を想定した多角的な問題集です。

### 問題 1 (コンフィグ読解・認可制御)
**質問**: Cisco IOS-XE ルータにおいて以下の AAA コンフィグを投入した。VTY 経由で SSH ログインしたユーザーが `configure terminal` コマンドを実行しようとしたが、ISE への認可パケットが送出されず、ローカルで処理されて許可されてしまった。原因として最も適切なものを1つ選びなさい。

```bash
aaa new-model
aaa authentication login default group ISE_TACACS local
aaa authorization exec default group ISE_TACACS local
aaa authorization commands 15 default group ISE_TACACS local
line vty 0 15
 login authentication default
 authorization exec default
```

- A) `aaa authorization config-commands` が設定されていないため。
- B) `line vty` 配下に `authorization commands 15 default` が適用されていないため。
- C) `aaa authorization console` が設定されていないため。
- D) `tacacs server` に `single-connection` が設定されていないため。

**正解**: **B**
**解説**: Cisco IOS-XE では、グローバルで `aaa authorization commands 15 default ...` を定義するだけでなく、対象の `line` (VTY または Console) 配下に `authorization commands 15 default` を明示的に適用しなければ、VTY ライン上でコマンド認可要求は評価されません。なお、Global Configuration モード内のコマンドを対象とする場合は A の `aaa authorization config-commands` も必要ですが、そもそも VTY ラインにコマンド認可がバインドされていないことが根本原因です。

---

### 問題 2 (トラブルシューティング・ログ分析)
**質問**: ISE の Live Logs にて、新規に追加した Catalyst スイッチからの TACACS+ ログイン試行がすべて失敗し、Failure Code: `15013 TACACS+ Shared Secret mismatch` が記録された。デバイス側と ISE 側で設定を確認したところ、双方とも Shared Secret として `TacacsKeyPass123` を入力していた。この障害の原因として考えられるものはどれか。

- A) スイッチの `tacacs server` 設定で `key 7` の暗号化文字列指定と `key 0` の平文指定を誤って混同したため。
- B) ISE 側で Device Admin Service が無効化されているため。
- C) スイッチ側で `ip tacacs source-interface` が設定されておらず、異なる送信元 IP で ISE に到着したため。
- D) TACACS+ が UDP ポート 49 を使用しているためパケットがドロップされたため。

**正解**: **A**
**解説**: スイッチ CLI にて `key 7 TacacsKeyPass123` と入力すると、CLI は `TacacsKeyPass123` を既にタイプ7暗号化された文字列として解釈し、実際の平文鍵とは全く異なる値に変換して送信します。平文で鍵を入力する場合は `key 0 TacacsKeyPass123` または `key TacacsKeyPass123` と指定しなければなりません。

---

### 問題 3 (Design・セキュリティアーキテクチャ)
**質問**: エンタープライズネットワークにおいて、管理アクセスの認証方式として RADIUS ではなく TACACS+ を選択すべき決定的な要件はどれか。2つ選びなさい。

- A) ログイン後に管理者が実行するコマンドを1行ずつリアルタイムで ISE に問い合わせて可否を制御したい。
- B) 認証パケットの User-Password 属性のみを暗号化し、ヘッダーを透過させたい。
- C) ネットワーク障害発生時や外部攻撃時に、信頼性の高い TCP プロトコルで再送制御を行いたい。
- D) 802.1X ポートベースアクセス制御と管理アクセス認証を同一プロトコルで統一したい。

**正解**: **A, C**
**解説**: TACACS+ は TCP (Port 49) を使用してコネクション指向の信頼性を提供し、かつ認証・認可・会計が分離されているためコマンド単位の認可 (Command Authorization) が可能です。RADIUS は UDP ベースであり、認証と認可が統合されているためコマンド単位のリアルタイム制御はできません。

---

### 問題 4 (実装・ASA 管理アクセス)
**質問**: Cisco ASA ファイアウォールにおいて、TACACS+ サーバーグループ `ISE_GRP` を使用して SSH ログイン認証を設定したが、ISE 障害時にローカルユーザーでログインできない。正しいローカルフォールバック設定はどれか。

- A) `aaa authentication ssh console ISE_GRP local`
- B) `aaa authentication ssh console ISE_GRP LOCAL`
- C) `aaa authentication default group ISE_GRP fallback local`
- D) `aaa-server ISE_GRP protocol group-fallback LOCAL`

**正解**: **B**
**解説**: Cisco ASA におけるローカルフォールバックキーワードは、大文字の **`LOCAL`** です（IOS-XE では小文字の `local`）。小文字で入力した場合や構文を誤った場合、ASA は設定を受け付けないか正しくフォールバックしません。

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよびテクニカルガイド一覧:

* Cisco ISE Device Administration Configuration Guide
* Cisco IOS-XE AAA Configuration Guide
* Cisco ASA Series CLI Configuration Guide - AAA for System Administrators
* Cisco TACACS+ Protocol Specification (RFC 8907)
* Cisco Live Presentation: Advanced Cisco ISE Device Administration (TACACS+)

---

## 📝 **補足（Notes）**

### トラブルシューティングの極意 (CCIE Security Lab Tips)
1. **ISE Live Logs の判定ロジック**: `Operations > TACACS > Live Logs` を開き、認証要求が届いているか確認します。表示されない場合は、IP 疎通、Routing、および `ip tacacs source-interface` を真っ先に疑ってください。
2. **Debug コマンドの併用**: IOS-XE CLI にて `debug tacacs authentication` および `debug tacacs authorization` を実行した状態のまま SSH ログインを行うことで、バックグラウンドで交わされる TACACS+ パケットのステータスコード（`PASS` / `FAIL` / `ERROR`）がリアルタイム表示されます。
3. **Single-Connection の効果**: 大規模構成や自動化スクリプト等で短時間に大量のコマンドが送信される環境では、`single-connection` が有効になっていないと TCP ポート枯渇や ISE 側のスレッド枯渇が発生するため、ベストプラクティスとして常に適用することを推奨します。


