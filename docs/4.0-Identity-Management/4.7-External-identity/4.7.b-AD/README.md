---
layout: default
title: 4.7.b-AD
nav_order: 2
parent: 4.7-External-identity
grand_parent: 4.0-Identity-Management
---

# 4.7.b AD (Active Directory Integration with Cisco ISE)

## 📘 概要

Cisco Identity Services Engine (ISE) における **Active Directory (AD) 統合**（Native AD Join）は、エンタープライズネットワークアクセス制御（802.1X, MAB, CWA, TACACS+ Device Admin 等）において、Microsoft Active Directory を信頼できる統一的な外部アイデンティティリポジトリとして直接活用するための基盤技術です。

Cisco ISE は、Samba/Winbind 相当の高度なクライアントエンジンを内部に実装しており、Active Directory ドメインに対して「ドメインメンバーコンピュータ（Domain-Joined Computer Account）」としてネイティブ参加（Native Join）します。これにより、単なる LDAP ルックアップを超えた、Kerberos 認証、NTLMv2 認証、MS-RPC (Microsoft Remote Procedure Call) 経由のパスワード変更、マシン認証、およびネストされた Active Directory グループ（Nested Group）の動的解決を実現します。

### 利用目的

* **集中型ユーザーおよびマシン認証**: Active Directory 内のドメインユーザーアカウントおよびドメインコンピュータアカウントを直接参照し、802.1X (EAP-TLS, PEAP-MSCHAPv2, EAP-FAST 等) や MAB、Web 認証（CWA/LWA）の認証を実行する。
* **属性・グループベースのアクセスポリシー認可**: Active Directory 内のグループメンバーシップ（Security Group）、OU (Organizational Unit)、ユーザー/コンピュータ属性（Department, EmployeeID, UserPrincipalName, SAMAccountName 等）をリアルタイムに取得し、ISE Authorization Policy の条件として利用する。
* **パスワード変更機能 (Password Change)**: EAP-MSCHAPv2 や WebAuth (CWA) 認証時に期限切れパスワードを判定し、ネットワーク接続シーケンスの中で Active Directory のパスワードを直接更新・変更させる。
* **クロスドメイン・マルチフォレスト認証**: 双方向または片方向のドメイン間信頼関係（Forest Trust, External Trust, Shortcut Trust）を通過し、自ドメインだけでなく信頼された外部フォレストのユーザー/コンピュータアカウントを認証・認可する。
* **ネットワーク管理アクセス制御 (TACACS+ / RADIUS Device Admin)**: Cisco 機器への CLI / Web 管理ログイン時に、ISE が Active Directory 内の特権管理者グループを参照して Shell Profile や Command Set を動的に割り当てる。

---

## 🔑 要点

Cisco ISE と Active Directory 統合における技術的要点および設計パラメータは下表の通りです。

| 項目 | 内容 |
| --- | --- |
| **特徴** | ISE ノードが AD ドメインメンバー（Computer Account）としてネイティブ参加し、Kerberos / MS-RPC / SMB / LDAP プロトコル群を統合して認証・属性取得を行う。 |
| **用途** | 有線/無線 802.1X 認証、MAB、ゲスト/BYOD CWA 認証、パスワード更新、TACACS+ 管理者アクセス制御、Passive Identity (IP-to-User マッピング)。 |
| **メリット** | ・MSCHAPv2 (EAP-PEAP) 認証時のパスワードハッシュ直接照合が可能。<br>・ネストされた AD グループの自動解決。<br>・ドメインコントローラ (DC) の自動発見 (DC Locator) および Active Directory Site Affinity。<br>・マルチフォレスト信頼関係のサポート。 |
| **デメリット / 課題** | ・ISE ノードと AD DC 間で多数の IP ポート（DNS, Kerberos, LDAP/LDAPS, SMB, Dynamic RPC）開通が必要。<br>・NTP による高精度な時刻同期（許容誤差 5 分以内）が必須。<br>・大規模環境での AD グループ検索に伴う認証遅延リスク。 |
| **対応機種 / ノード** | Cisco ISE 2.x, 3.x 全バージョン。Standalone, Primary PAN, Secondary PAN, PSN の各ノード（全 PSN が AD ドメインへ個別に Join する必要あり）。 |
| **制限事項** | ・1 つの ISE Deployment で複数の AD 参加が構成可能（Multi-Domain/Multi-Forest）だが、1 ノードあたり最大 50 AD Join Groups。<br>・AD ジョインアカウントには「Create Computer Objects」権限が必要。<br>・EAP-MSCHAPv2 認証は NTLMv2/Kerberos 暗号化要件に依拠。 |
| **設計上の注意点** | ・すべての ISE ノード（特に PSN）で正確な DNS (SRV/A レコード) 解決と NTP 同期を保証する。<br>・Active Directory Site 構成を正しく反映し、近接する Local DC にリクエストを誘導する。<br>・大量のグループが割り当てられたユーザーの検索負荷を軽減するため、必要な Group/Attribute のみを選択的にインポートする。 |

---

## 🏗 動作原理

Cisco ISE と Active Directory の連携は、「**1. ドメイン参加 (Domain Join)**」「**2. DC 発見 (DC Locator)**」「**3. 認証・属性取得 (Authentication & Authorization)**」の 3 つのフェーズで動作します。

### 1. Active Directory Join および DC Locator メカニズム

ISE が AD ドメインに参加する際、以下のステップでドメインコントローラを発見し、コンピューターアカウントを作成・バインドします。

```text
Cisco ISE (PSN / PAN)                 DNS Server                  Active Directory DC
       │                                  │                                │
       │ 1. DNS SRV Lookup                │                                │
       │ ────────────────────────────────>│                                │
       │ _ldap._tcp.dc._msdcs.<domain>    │                                │
       │ _kerberos._tcp.<domain>          │                                │
       │                                  │                                │
       │ 2. SRV Record Response (DC IPs)  │                                │
       │ <────────────────────────────────│                                │
       │                                                                   │
       │ 3. CLDAP Ping (UDP 389) for DC Site Affinity                      │
       │ ─────────────────────────────────────────────────────────────────>│
       │                                                                   │
       │ 4. CLDAP Netlogon Response (DC Capabilities & Site Name)          │
       │ <─────────────────────────────────────────────────────────────────│
       │                                                                   │
       │ 5. Kerberos AS-REQ / TGT Request (TCP/UDP 88)                     │
       │ ─────────────────────────────────────────────────────────────────>│
       │                                                                   │
       │ 6. Kerberos AS-REP (TGT Received)                                 │
       │ <─────────────────────────────────────────────────────────────────│
       │                                                                   │
       │ 7. SMB / Dynamic RPC (TCP 445 / TCP 135+Dynamic RPC)              │
       │    Computer Account Creation & Trust Key Exchange                 │
       │ <────────────────────────────────────────────────────────────────>│
       │                                                                   │
```

1. **DNS SRV レコード参照**: ISE は設定された Primary DNS サーバーに対し、`_ldap._tcp.dc._msdcs.<Domain>` および `_kerberos._tcp.<Domain>` の SRV レコードを問い合わせ、利用可能な DC の IP アドレスとポート、プライオリティ、ウェイトを取得します。
2. **CLDAP Ping による Site Affinity 判定**: ISE は返却された DC のリストに対し、UDP 389 ポートで CLDAP (Connectionless LDAP) Ping を送信します。DC からの応答に含まれる Site 情報（例: `Default-First-Site-Name` や拠点サイト名）を比較し、ISE ノード自身の IP サブネットに最も近い Local DC を選定（Site Affinity）します。
3. **Kerberos 認証と TGT 取得**: 取得した Local DC に対して Kerberos TGT (Ticket Granting Ticket) を要求し、管理者の認証情報を検証します。
4. **コンピューターアカウント作成と暗号化キー交換**: SMB (TCP 445) および RPC を介して Active Directory の指定された OU（デフォルトは `CN=Computers`）内に ISE のコンピューターアカウント（例: `ISE-PSN-01$`）を作成し、マシンパスワード（Trust Secret）を確立・保存します。

---

## ⚙ 動作シーケンス

ネットワーク端末が 802.1X (EAP-PEAP / MSCHAPv2) で接続し、ISE が Active Directory を参照して認証・認可を完了するまでの全パケット処理シーケンスを以下に示します。

```text
Supplicant             Authenticator (Switch/WLC)              Cisco ISE (PSN)                Active Directory DC
    │                              │                                 │                                │
    │ 1. EAPOL-Start               │                                 │                                │
    │ ───────────────────────────> │                                 │                                │
    │                              │ 2. RADIUS Access-Request        │                                │
    │                              │    (EAP-Response/Identity)      │                                │
    │                              │ ──────────────────────────────> │                                │
    │                              │                                 │ 3. TLS Tunnel Negotiation      │
    │                              │ <─────────────────────────────> │    (PEAP Phase 1)              │
    │                              │                                 │                                │
    │ 4. Inner EAP MSCHAPv2 Challenge (PEAP Phase 2)                 │                                │
    │ <─────────────────────────────────────────────────────────────>│                                │
    │    (Client sends MSCHAPv2 Response: Username, Challenge, Response)│                                │
    │                              │                                 │ 5. NETLOGON / MS-RPC Request   │
    │                              │                                 │    (Pass-through Auth)        │
    │                              │                                 │ ─────────────────────────────> │
    │                              │                                 │    Validate MSCHAPv2 Challenge │
    │                              │                                 │    & NT-Response               │
    │                              │                                 │                                │
    │                              │                                 │ 6. NETLOGON Auth Success Response│
    │                              │                                 │ <─────────────────────────────│
    │                              │                                 │                                │
    │                              │                                 │ 7. LDAP / Kerberos Query       │
    │                              │                                 │    Fetch Groups, OU, Attributes│
    │                              │                                 │ ─────────────────────────────> │
    │                              │                                 │                                │
    │                              │                                 │ 8. LDAP Response (User Groups) │
    │                              │                                 │ <─────────────────────────────│
    │                              │                                 │                                │
    │                              │                                 │ 9. Evaluate Policy Set         │
    │                              │                                 │    Compute dACL / SGT / VLAN   │
    │                              │ 10. RADIUS Access-Accept        │                                │
    │                              │     (EAP-Success + Attrs)       │                                │
    │                              │ <────────────────────────────── │                                │
    │ 11. Port Authorized          │                                 │                                │
    │ <─────────────────────────── │                                 │                                │
```

### ステップ詳細

1. **PEAP Phase 1 (TLS トンネル構築)**: Supplicant と ISE 間で EAP-PEAP の外側 TLS トンネルを確立します。ISE はサーバー証明書を提示し、Supplicant が検証します。
2. **PEAP Phase 2 (Inner MSCHAPv2 認証)**: TLS トンネル内部で EAP-MSCHAPv2 認証が開始されます。ISE はランダムな 16 バイトの Authenticator Challenge を生成して端末に送信します。端末はユーザーパスワードの NT ハッシュと Challenge を元に 24 バイトの Response を計算して返信します。
3. **Active Directory への Pass-Through 認証**: ISE は受け取った Challenge と Response を自前で暗号解読するのではなく、MS-RPC (NETLOGON プロトコル, `SamLogonEx`) を利用して AD ドメインコントローラへ送信し、認証結果の判定を委任（Pass-Through Auth）します。
4. **AD グループおよび属性の取得**: 認証が成功すると、ISE は Kerberos または LDAP (TCP 389/636) を用いて対象ユーザーアカウントのトークン（Security Identifier: SID）、所属するネストされた AD グループ（`memberOf` 属性）、その他の設定された属性（例: `department`, `title`, `userAccountControl` 等）をクエリします。
5. **Policy Set 評価および 認可レスポンス**: ISE は取得した AD グループ情報や属性をルール条件と照合し、適用する Access Permission (VLAN, dACL, Security Group Tag, Reauthentication Timer 等) を含んだ RADIUS Access-Accept を Authenticator (スイッチ/WLC) へ送出します。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験における Active Directory 統合に関する重要ポイント、トラップ、設定要件、およびトラブルシューティング技法を整理します。

### Blueprintにおける超重要ポイント

1. **DNS および NTP の完全な事前検証**:
   - ISE が AD ドメインに正常に参加・維持するための絶対条件は、**すべての ISE ノードにおける高精度な DNS 逆引き/正引き**と**NTP 時刻同期（AD DC との時刻差が 5 分以内）**です。
   - ラボ試験で AD ジョインエラーが発生した場合、最初に疑うべきは AD 自体ではなく、ISE 側の DNS 設定 (`/etc/resolv.conf` 相当の IP Name Server) または NTP オフセットです。

2. **マルチフォレスト (Multi-Forest) と信頼関係 (Trust Relationships)**:
   - 試験で 2 つ以上の AD フォレスト（例: `corp.local` と `partner.local`）が存在し、双方のユーザーを ISE で認証させる問題が出題される場合があります。
   - ISE は、ドメイン参加したプライマリフォレスト（例: `corp.local`）と双方向トラスト（Two-Way Forest Trust）または片方向トラスト（One-Way Trust）にある外部フォレストを自動検出（Discover Trusts）します。
   - **トラスト未検出時の対処**: ドメイン参加設定の「Trusts」タブで「Refresh Trusts」を実行するか、Forest Name を手動追加する必要があります。

3. **EAP-MSCHAPv2 (PEAP) vs EAP-TLS の認証メカニズムの違い**:
   - **EAP-MSCHAPv2**: パスワードのハッシュを検証するため、ISE は AD DC に対して MS-RPC (Netlogon) 経由の認証委任（Pass-Through Authentication）を行います。ユーザー名フォーマットは `domain\user` または `user@domain` が推奨されます。
   - **EAP-TLS (証明書認証)**: クライアント証明書（X.509）の公認ルート CA 検証は ISE 内部で行われます。AD に対する処理は**属性照会・アカウント状態検証（Identity Mapping & Lookup）のみ**となり、証明書の SAN (Subject Alternative Name) や UPN / CN から抽出した文字列を AD の `sAMAccountName` や `userPrincipalName` とマッチングさせます。

4. **Machine Authentication (マシン認証) と User Authentication (ユーザー認証)**:
   - ドメイン参加している Windows PC が起動時に送る `host/PC-NAME.domain.com` 認証（マシン認証）と、ユーザーがログインした後のユーザー認証を組み合わせた「MAR (Machine Access Restriction)」または「Identity Services Engine Compound Condition」の出題に注意してください。
   - AD 内のコンピューターアカウントは `Domain Computers` グループや特定 OU に存在するため、ISE 側で Machine Authentication 用の AD Groups を正しく指定しておく必要があります。

5. **パスワード変更機能 (Password Change On EAP-MSCHAPv2 / CWA)**:
   - 「期限切れパスワードの次回ログイン時変更を有効化せよ」という要件が出題されることがあります。
   - AD ジョイン設定で **Enable Password Change** を有効にし、かつ Allowed Protocols (例: `Default PEAP Settings`) で **Allow MS-CHAPv2 Password Change** にチェックを入れる必要があります。

---

## 🛠 設定方法

### 1. Cisco ISE GUI における Active Directory Native Join 設定手順

#### Step 1: AD 接続パラメータの設定と Join 実行
1. ISE 管理コンソールにログインし、**Administration > Identity Management > External Identity Sources > Active Directory** に移動します。
2. **Add** をクリックし、以下を指定します。
   - **Join Point Name**: `AD_Corp`（識別用任意名）
   - **Active Directory Domain Name**: `corp.local`（Fully Qualified Domain Name）
3. **Save** をクリックした後、画面上部の **Join** ボタンをクリックします。
4. ドメイン参加ダイアログで以下を入力します。
   - **ISE Node**: 参加させる PSN / PAN ノードを選択（「Select All Nodes」を推奨）
   - **AD Username**: `Administrator`（Domain Join 権限を持つアカウント）
   - **AD Password**: 対象アカウントのパスワード
   - **Organizational Unit (OU)**:（任意、指定しない場合はデフォルトの `CN=Computers` に作成される）
5. **Join** を実行し、全ノードの Status が `Operational` になることを確認します。

#### Step 2: AD Groups および Attributes のインポート
1. AD ジョインポイントの編集画面で **Groups** タブを開きます。
2. **Add > Select Groups from Directory** をクリックします。
3. 検索フィルタ（例: `Domain Users`, `Domain Admins`, `HR-Group`, `Sales-Group`）を入力して **Search** を実行します。
4. 認可ポリシーで使用する対象グループにチェックを入れ、**OK** をクリックします。
5. **Attributes** タブを開き、認可条件や CWA カスタマイズ画面で参照したい属性（例: `department`, `mail`, `title`, `userAccountControl`, `msDS-UserPasswordExpiryTimeComputed`）を同様に検索・インポートします。
6. 画面下部の **Save** をクリックします。

#### Step 3: Identity Source Sequences (ISS) への組み込み
1. **Administration > Identity Management > Identity Source Sequences** に移動します。
2. 既存の `All_User_ID_Stores` を編集するか、新しい Sequence（例: `AD_Local_Sequence`）を作成します。
3. **Authentication Search List** に上記で作成した `AD_Corp` を追加（優先順位を最上位に配置）します。
4. **Advanced Search Option** で例外処理（User Not Found, Auth Failed, Process Error）をビジネス要件に合わせて設定します（例: User Not Found 時に Local Users へフォールバックさせる場合は `Continue` を選択）。

---

### 2. Cisco ISE CLI による AD ジョインおよび診断コマンド設定

ISE CLI（SSH 接続）からドメイン参加状態を確認し、手動で Join / Leave を行うコマンド例です。

```bash
# ISE CLI にログイン後、AD ジョイン状態の確認
ise-psn01/admin# show ad-status AD_Corp

# AD ドメインテストおよび DC の疎通確認
ise-psn01/admin# test ad-connectivity domain corp.local

# CLI からのドメイン手動参加 (必要時)
ise-psn01/admin# identity-server ad join corp.local user Administrator

# CLI からのドメイン離脱 (必要時)
ise-psn01/admin# identity-server ad leave corp.local user Administrator
```

---

### 3. Cisco ISE Policy Set (Authorization Policy) の条件記述例

ISE GUI (**Policy > Policy Sets**) における Active Directory グループおよび属性条件の定義例：

```text
Policy Set: Wired_802.1X_Access

[Authentication Policy]
Rule 1: AD_User_Auth
  Condition: Network Access : AccessType EQUALS Wired EAP
  Use: AD_Corp (または All_User_ID_Stores)

[Authorization Policy]
Rule 1: Domain_Admins_FullAccess
  Condition: AD_Corp:ExternalGroups EQUALS corp.local/Users/Domain Admins
  Permissions: PermitAccess + DACL_FULL_ACCESS + SGT: Network_Admin (Tag 2)

Rule 2: HR_Employees_Restricted
  Condition: AD_Corp:ExternalGroups EQUALS corp.local/OU_HR/HR_Users
             AND AD_Corp:department EQUALS Human Resources
  Permissions: PermitAccess + DACL_HR_ONLY + VLAN: 102

Rule 3: Machine_Authenticated_Only
  Condition: Network Access : UseCase EQUALS Wired 802.1X
             AND AD_Corp:ExternalGroups EQUALS corp.local/Computers/Domain Computers
  Permissions: PermitAccess + DACL_LIMITED_BOOTSTRAP
```


## 🔍 検証コマンド

ISE GUI および CLI で Active Directory 連携状態を確認・検証するための代表的コマンド一覧です。

| 目的 | コマンド / 操作手順 |
| --- | --- |
| **AD ジョイン状態の確認 (CLI)** | <code>show ad-status &lt;Join_Point_Name&gt;</code> |
| **AD ドメインコントローラ疎通テスト** | <code>test ad-connectivity domain &lt;domain_name&gt;</code> |
| **ISE ノードの DNS 解決確認** | <code>show name-server</code> / <code>nslookup &lt;dc_fqdn&gt;</code> |
| **時刻同期 (NTP) 状態の確認** | <code>show ntp</code> |
| **AD 診断テストの実行 (GUI)** | ISE GUI: **Administration > Identity Management > External Identity Sources > Active Directory > Diagnostic Tools** |
| **AD リアルタイム接続・認証デバッグ** | <code>debug logging component active-directory level debug</code> |
| **AD ドメインジョイン実行 (CLI)** | <code>identity-server ad join &lt;domain&gt; user &lt;admin_user&gt;</code> |
| **AD ドメイン離脱実行 (CLI)** | <code>identity-server ad leave &lt;domain&gt; user &lt;admin_user&gt;</code> |
| **ISE サポートバンドル/ログ抽出** | <code>show logging application active-directory.log</code> |

---

## 🚨 トラブルシュート

Active Directory 統合環境で頻発する障害症状、根本原因、確認手順および対処方法を下表にまとめます。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **AD Domain Join が `Failed to join domain` で失敗する** | 1. ISE の時刻が AD DC と 5 分以上乖離している。<br>2. DNS SRV レコードが解決できない。<br>3. 指定した AD ユーザーに Computer Account 作成権限がない。 | <code>show ntp</code><br><code>nslookup _ldap._tcp.dc._msdcs.&lt;domain&gt;</code><br>ISE GUI: Diagnostic Tools | 1. NTP サーバーの設定を見直し、時刻差を 5 分未満（推奨は秒単位）にする。<br>2. ISE の Primary DNS サーバーを AD DNS サーバの IP に変更する。<br>3. `Domain Admins` 権限を持つアカウントを使用するか、指定 OU に事前アカウントを作成（Pre-stage）する。 |
| **特定ノード（PSN）のみ AD ジョイン状態が `Unjoined` または `Not Operational` になる** | 1. 該当 PSN ノードと AD DC 間の Firewall で TCP 445 / UDP 88 等が遮断されている。<br>2. 該当 PSN の Machine Secret (Trust Key) の同期が不完全。 | <code>show ad-status &lt;Join_Point&gt;</code><br><code>test ad-connectivity domain &lt;domain&gt;</code> | 1. 該当 PSN と DC 間の通信（DNS, Kerberos, LDAP, SMB, Dynamic RPC）のパス上ファイアウォールルールを確認・開放する。<br>2. ISE GUI から対象 PSN のみ「Leave」を実施し、再度「Join」をやり直す。 |
| **EAP-MSCHAPv2 (PEAP) 認証が `11002 Authentication Failed` で拒否される** | 1. ユーザーパスワードが間違っている。<br>2. アカウントが AD 上で Lockout / Disabled / Password Expired になっている。<br>3. NTLM パススルー認証で DC がタイムアウトした。 | ISE Operations > RADIUS > Live Logs<br>`active-directory.log` | 1. ISE Live Logs の Details 画面で Failure Reason コード（例: `11007 Account Disabled`, `11008 Account Expired`）を確認する。<br>2. AD 側で対象アカウントのロック解除・有効化を行う。<br>3. ISE AD ジョイン設定の Timeout 値を調整するか、負荷の低い DC へリダイレクトさせる。 |
| **ネストされた AD グループ（Nested Group）が ISE 側で認識されない** | 1. AD グループが正しく External Identity Source にインポートされていない。<br>2. ネスト検索の深度上限に達している。 | ISE GUI: Active Directory > Groups タブ<br>Live Logs の Identity Details | 1. 親グループだけでなく子グループも明示的に AD Groups 一覧に追加する。<br>2. ISE の AD 設定で `Nested Group Search` オプションが有効になっているか確認する。 |
| **パスワード変更 (Password Change) 処理が `11054 Password change failed` になる** | 1. Allowed Protocols で `Allow MS-CHAPv2 Password Change` が未有効。<br>2. 新パスワードが AD のパスワードポリシー（文字数・複雑さ・履歴）を満たしていない。 | ISE Operations > RADIUS > Live Logs<br>AD イベントログ (Event ID 4723/4724) | 1. Allowed Protocols (Default PEAP) で MS-CHAPv2 Password Change を有効化する。<br>2. 端末側で入力した新パスワードが AD のポリシーを満たしているか確認する。 |
| **マルチフォレスト環境で外部フォレストのユーザーが `User Not Found` になる** | 1. フォレスト間の Trust タイプが非互換、または Selective Authentication が有効になっている。<br>2. UPN サフィックスのルックアップ設定が不適切。 | ISE GUI: Active Directory > Trusts タブ<br><code>show ad-status</code> | 1. ISE GUI の Trusts タブで「Refresh Trusts」を実行し、該当 External Forest が `Operational` となっているか確認する。<br>2. Identity Source Sequence の Advanced Settings で「Search All Joined Domains/Forests」を有効にする。 |

---

## ⚠ 制限事項

* **時刻同期 (Clock Skew)**: ISE ノードと AD ドメインコントローラ間の時刻差が **5 分 (300 秒)** を超えると、Kerberos 認証チケットが無効化され、すべての AD 認証およびドメイン参加状態が失敗します。
* **1 ノードあたりの AD Join Group 数制限**: 1 つの ISE ノード群（Deployment）は最大 50 の AD Join Groups (Domain 参加グループ) を構成可能ですが、大規模多国籍企業などでドメイン数が極端に多い場合は設計上の注意が必要です。
* **暗号化プロトコル制限**: EAP-MSCHAPv2 は NTLMv2 ハッシュ検証を内部で行うため、Active Directory 側で NTLMv2 が完全に無効化（「Network security: Restrict NTLM: NTLM authentication in this domain」＝ Deny All）された環境では動作しません。その場合は EAP-TLS または Kerberos 認証への移行が必要です。

---

## 🔄 他技術との関連

* **IEEE 802.1X / MAB / CWA**: ISE が Active Directory から取得した `ExternalGroups` や属性を基に、スイッチや WLC に対して Dynamic VLAN, dACL, SGT を指示する RADIUS Access-Accept を返します。
* **Cisco TrustSec (SGT)**: AD の「Security Group」や「OU」を ISE 側で「Security Group Tag (SGT)」にマッピング（例: `Domain Admins` -> `SGT: Admin (Tag 2)`）し、ネットワーク全体のグループベースマイクロセグメンテーションを実現します。
* **Passive Identity (pxGrid-Direct / AD Agent / WMI)**: ISE は 802.1X だけでなく、AD DC のセキュリティイベントログ（Event ID 4624: ログオン）を WMI や AD Agent 経由で受信し、認証を経ていない端末の IP アドレスと AD ユーザー名のマッピング（IP-to-User Context）情報を保持して pxGrid 経由で FMC や WSA へ共有します。

---

## 🧩 比較表

### 外部 ID ソースとしての AD 連携方式比較

| 項目 | AD Native Join (本技術) | LDAP Integration | Passive Identity (AD Agent / WMI) |
| --- | --- | --- | --- |
| **接続プロトコル** | Kerberos, MS-RPC, SMB, LDAP, Dynamic RPC | LDAP / LDAPS (TCP 389 / 636) | WMI (RPC), AD Agent, pxGrid-Direct |
| **802.1X MSCHAPv2 対応** | **可能** (Pass-through NTLM/RPC 検証) | **不可** (クリアテキストまたは EAP-GTC/TTLS のみ) | N/A (認証プロトコルではない) |
| **パスワード変更機能** | **対応** (MS-RPC 経由) | 不可（読み取り専用） | N/A |
| **ネストグループ解決** | **自動** (AD の Token-Groups 構造を参照) | 手動フィルタ設定が必要 | N/A |
| **主な用途** | 802.1X 認証、MAB、TACACS+、認可ポリシー | 属性検索、EAP-TLS/GTC、証明書マッピング | IP アドレスと AD ユーザー名の透過的マッピング |

---

## 💡 ベストプラクティス

1. **DNS / NTP の二重化と信頼性向上**:
   - 各 ISE ノードには、Active Directory 統合型 DNS サーバーを最優先 (Primary) として登録してください。
   - NTP サーバーは AD ドメインの PDC Emulator または信頼できる上位 NTP 階層 (Stratum 1/2) を指定し、全 PSN ノードで同期を徹底してください。
2. **Active Directory Site Affinity の活用**:
   - 地理的に分散した ISE 展開では、ISE PSN ノードが配置されたサブネットを AD の「Active Directory Sites and Services」で正しく サイト定義してください。これにより、PSN はWAN を越えた遠隔地の DC ではなく、必ずローカル拠点の DC へ認証リクエストを送信します。
3. **必要なグループ・属性のみのインポート**:
   - AD には数千のグループが存在することがありますが、ISE にインポートするのは**ポリシー判定に直接使用するグループ（通常 10〜30 個程度）のみ**に絞ってください。多すぎるグループ検索は RADIUS 応答遅延（Timeout）の原因となります。


## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験の実践的タスクを想定した 10 個の演習シナリオです。各問題の要件、コンフィグ・GUI 設定手順、および検証手順を網羅しています。

---

### 演習 1: 単一 Active Directory ドメインへの Native Join と 802.1X EAP-PEAP 認証構成

#### 【問題】
ISE ノードを企業 Active Directory ドメイン `corp.local` にネイティブジョインさせ、有線 802.1X クライアントの EAP-PEAP (MSCHAPv2) 認証を確立せよ。

#### 【要件】
1. AD ドメイン名: `corp.local`
2. ISE ジョインポイント名: `AD_CORP_JOIN`
3. AD ドメイングループ `Domain Admins` および `Domain Users` を ISE 外部グループとしてインポートすること。
4. 有線 802.1X 認証ポリシーにて、`Domain Admins` 所属ユーザーには `PermitAccess` および `DACL_FULL` を割り当て、`Domain Users` には `VLAN 100` を割り当てること。

#### 【設定例】

##### ISE GUI 設定手順
1. **Administration > Identity Management > External Identity Sources > Active Directory** に移動し、**Add** をクリック。
   - **Join Point Name**: `AD_CORP_JOIN`
   - **Active Directory Domain Name**: `corp.local`
   - **Save** をクリック。
2. 画面上部 **Join** をクリックし、管理者クレデンシャル（User: `Administrator`, Pass: `Cisco123!`）を入力して全ノードを Join させる。
3. **Groups** タブで **Add > Select Groups from Directory** を選択。
   - 検索フィルタに `Domain Admins` および `Domain Users` を指定して検索し、チェックを入れて **OK** > **Save**。
4. **Policy > Policy Sets > Wired_PEAP** を作成・編集。
   - **Authentication Policy**:
     - Rule Name: `AD_PEAP_Auth`
     - Condition: `Network Access : AccessType EQUALS Wired EAP`
     - Use: `AD_CORP_JOIN`
   - **Authorization Policy**:
     - Rule 1: `Rule_Admin`
       - Condition: `AD_CORP_JOIN:ExternalGroups EQUALS corp.local/Users/Domain Admins`
       - Profiles: `PermitAccess`, `DACL_FULL`
     - Rule 2: `Rule_User`
       - Condition: `AD_CORP_JOIN:ExternalGroups EQUALS corp.local/Users/Domain Users`
       - Profiles: `VLAN100_Profile` (RADIUS Attribute: `Tunnel-Private-Group-ID = 100`)

##### スイッチ側 AAA 設定例
```text
aaa new-model
aaa authentication dot1x default group ISE_GROUP
aaa authorization network default group ISE_GROUP
aaa accounting dot1x default start-stop group ISE_GROUP

radius server ISE-PSN1
 address ipv4 192.168.10.5 auth-port 1812 acct-port 1813
 key Cisco123!

interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 99
 authentication port-control auto
 dot1x pae authenticator
```

#### 【検証方法】
1. Supplicant 端末から ドメインユーザー `user01@corp.local` で 802.1X ログインを実行。
2. スイッチでセッション状態を確認:
   ```bash
   Switch# show authentication sessions interface Gi1/0/1 details
   ```
3. ISE コンソールの **Operations > RADIUS > Live Logs** で該当認証ログを開き、Identity Store に `AD_CORP_JOIN` が表示され、正しく Authorization Policy `Rule_User` にマッチしていることを確認。

---

### 演習 2: クロストラスト・マルチフォレスト (Two-Way Forest Trust) 環境での 802.1X 認証

#### 【問題】
ISE を `headquarters.local` ドメインにジョインさせた状態で、双方向信頼関係（Two-Way Forest Trust）にある別フォレスト `subsidiary.local` のユーザーアカウントを認証・認可せよ。

#### 【要件】
1. プライマリ AD ジョインポイント: `HQ_AD` (`headquarters.local`)
2. 信受フォレスト `subsidiary.local` の信頼関係を自動検出し、`subsidiary.local/Users/Contractors` グループをインポートすること。
3. `subsidiary.local` の契約社員ユーザーに対しては、アクセスを限定した `DACL_CONTRACTOR` を適用すること。

#### 【設定例】

##### ISE GUI 設定手順
1. **Administration > Identity Management > External Identity Sources > Active Directory** で `HQ_AD` を選択。
2. **Trusts** タブを開き、**Refresh Trusts** ボタンをクリック。
3. 一覧に `subsidiary.local` が表示され、Trust Type が `Forest` / Status が `Operational` になることを確認。
4. **Groups** タブを開き、**Add > Select Groups from Directory** で Domain ドロップダウンから `subsidiary.local` を選択。
5. `Contractors` グループを検索してインポートし、保存。
6. **Policy > Authorization Policy**:
   - Rule Name: `Contractor_Access`
   - Condition: `HQ_AD:ExternalGroups EQUALS subsidiary.local/Users/Contractors`
   - Permissions: `PermitAccess` + `DACL_CONTRACTOR`

#### 【検証方法】
1. `subsidiary\contractor01` アカウントで 802.1X 接続を実行。
2. ISE CLI で AD ステータスを確認:
   ```bash
   ise-psn01/admin# show ad-status HQ_AD
   ```
   出力結果に `subsidiary.local` とのドメイン信頼状態が正常として表示されることを確認。

---

### 演習 3: EAP-MSCHAPv2 パスワード変更 (Password Change) 許可構成

#### 【問題】
Active Directory 上で「次回ログオン時にパスワード変更が必要」に設定されているユーザーが、802.1X 接続時に安全にパスワードを変更してネットワークアクセスを継続できるように構成せよ。

#### 【要件】
1. Active Directory ジョイン設定および Allowed Protocols ポリシーで MSCHAPv2 パスワード変更を有効化すること。
2. 期限切れユーザーが正しい旧パスワードと新パスワードを入力した際、認証を成功させて AD 上のパスワードを更新すること。

#### 【設定例】

##### ISE GUI 設定手順
1. **Administration > Identity Management > External Identity Sources > Active Directory > AD_CORP_JOIN** に移動。
2. **Advanced Settings** タブ（または Main タブの Configuration）で **Enable Password Change** にチェックを入れる。
3. **Policy > Policy Elements > Results > Authentication > Allowed Protocols** を開き、`Default PEAP Settings` を編集。
4. **Allow MS-CHAPv2 Password Change** にチェックを入れる。
5. 設定を保存する。

#### 【検証方法】
1. Active Directory Users and Computers (ADUC) で `testuser` のプロパティを開き、「User must change password at next logon」にチェックを入れる。
2. Windows Supplicant から `testuser` でログイン試行。パスワード変更プロンプトが表示されることを確認。
3. 新しいパスワードを入力し、認証が成功してネットワークに接続されることを確認。
4. ISE Live Logs で Failure 原因コードが発生せず、`Password Change Successful` イベントが記録されていることを確認。

---

### 演習 4: EAP-TLS 証明書 SAN/UPN 抽出と Active Directory ID マッピング

#### 【問題】
クライアント証明書を用いた 802.1X EAP-TLS 認証において、証明書の Subject Alternative Name (SAN) から UPN を抽出し、Active Directory アカウントと照合して認可グループを取得せよ。

#### 【要件】
1. Certificate Authentication Profile (CAP) `CAP_UPN_AD` を作成し、Identity Attribute に `Subject Alternative Name - Other Name` (UPN) を指定すること。
2. AD Identity Source で Identity Attribute Mapping に上記 UPN を指定し、AD 上の `userPrincipalName` 属性とマッチングさせること。

#### 【設定例】

##### ISE GUI 設定手順
1. **Administration > Identity Management > Certificate Authentication Profiles** に移動し、**Add**。
   - Name: `CAP_UPN_AD`
   - Principal Identity Attribute: `Subject Alternative Name - Other Name`
2. **Administration > Identity Management > External Identity Sources > Active Directory > AD_CORP_JOIN** に移動。
3. **Certificate** タブを開き、**Enable Certificate - Based Authentication** にチェック。
4. **Match Certificate Attribute to Active Directory Attribute** で:
   - Certificate Attribute: `Subject Alternative Name - Other Name`
   - AD Attribute: `userPrincipalName`
5. **Policy Sets** の Authentication Policy で Use 欄に `CAP_UPN_AD` と `AD_CORP_JOIN` の組み合わせを指定。

#### 【検証方法】
1. UPN (`user1@corp.local`) が埋め込まれた EAP-TLS 証明書を持つ端末から接続。
2. ISE Live Logs の Authentication Details で、Certificate 認証成功後に AD からグループ情報 (`ExternalGroups`) が正常に取得できていることを確認。

---

### 演習 5: AD 属性 (Department) に基づく ISE 認可ポリシー設計

#### 【問題】
Active Directory のユーザー属性 `department` が `Engineering` であるユーザーに対して、技術部専用 VLAN 200 と SGT `Eng_SGT (Tag 10)` を動的に割り当てよ。

#### 【要件】
1. AD ジョインポイント `AD_CORP_JOIN` にて、AD 属性 `department` をインポートすること。
2. 認可ポリシーの条件に `AD_CORP_JOIN:department EQUALS Engineering` を使用すること。

#### 【設定例】

##### ISE GUI 設定手順
1. **Administration > Identity Management > External Identity Sources > Active Directory > AD_CORP_JOIN**。
2. **Attributes** タブで **Add > Select Attributes from Computer/User**。
3. ユーザー `eng_user1` を検索し、属性リストから `department` にチェックを入れてインポート。
4. **Policy > Policy Sets > Authorization Policy**:
   - Rule Name: `Engineering_Rule`
   - Condition: `AD_CORP_JOIN:department EQUALS Engineering`
   - Permissions: `VLAN200_Profile` + `Security Group: Eng_SGT (Tag 10)`

#### 【検証方法】
1. `department` 属性が `Engineering` のユーザーでログイン。
2. スイッチ側で VLAN および SGT の割り当てを確認:
   ```bash
   Switch# show authentication sessions interface Gi1/0/2 details
   ```
   `VLAN: 200`, `Security Group Tag: 10` が正常に適用されていることを確認。

---

### 演習 6: AD 接続障害時の Identity Source Sequence (ISS) フォールバック構成

#### 【問題】
Active Directory サービスが停止・不可達（Process Error）になった場合でも、緊急アクセス用ローカルアカウント（ISE Local Users）で認証できるように Identity Source Sequence を構成せよ。

#### 【要件】
1. Sequence 名: `AD_Fallback_ISS`
2. プライマリ認証ソース: `AD_CORP_JOIN`
3. セカンダリ認証ソース: `Internal Users`
4. AD 障害時 (`On Process Error`) の挙動を `Continue` に設定すること。

#### 【設定例】

##### ISE GUI 設定手順
1. **Administration > Identity Management > Identity Source Sequences** で **Add**。
   - Name: `AD_Fallback_ISS`
2. **Authentication Search List** に以下を順番に追加:
   1. `AD_CORP_JOIN`
   2. `Internal Users`
3. 画面下部 **Advanced Search Option** を展開:
   - `If AD_CORP_JOIN returns Account Not Found`: `Continue`
   - `If AD_CORP_JOIN returns Process Error`: `Continue`
4. **Save** をクリックし、Authentication Policy の Use 欄で本 ISS を指定する。

#### 【検証方法】
1. AD DC と ISE 間の通信を一時的にルーター ACL でブロックし、AD 接続障害（Process Error）を発生させる。
2. ISE Local Users に登録された緊急用アカウントでログイン試行。
3. ISE Live Logs で `AD_CORP_JOIN` が Error となり、`Internal Users` に処理がフォールバックして認証が成功することを確認。

---

### 演習 7: ISE TACACS+ 管理者アクセスにおける AD グループに基づく Shell Profile 認可

#### 【問題】
Cisco ルーター/スイッチへの SSH 管理ログインにおいて、AD グループ `NetAdmin_Group` 所属ユーザーには Privilege Level 15 を与え、`NetOps_Group` 所属ユーザーには Privilege Level 7 を与えよ。

#### 【要件】
1. TACACS+ Device Admin 機能を有効化すること。
2. AD グループ `NetAdmin_Group` -> Shell Profile `Priv15_Profile`
3. AD グループ `NetOps_Group` -> Shell Profile `Priv7_Profile`

#### 【設定例】

##### ISE GUI 設定手順
1. **Work Centers > Device Administration > Policy Elements > Results > TACACS Shell Profiles**:
   - Profile 1: `Priv15_Profile` (Default Privilege: 15, Maximum Privilege: 15)
   - Profile 2: `Priv7_Profile` (Default Privilege: 7, Maximum Privilege: 7)
2. **Work Centers > Device Administration > Device Admin Policy Sets > Default**:
   - **Authentication Policy**: Use `AD_CORP_JOIN`
   - **Authorization Policy**:
     - Rule 1: `Admin_Rule`
       - Condition: `AD_CORP_JOIN:ExternalGroups EQUALS corp.local/Users/NetAdmin_Group`
       - Shell Profile: `Priv15_Profile`
     - Rule 2: `Ops_Rule`
       - Condition: `AD_CORP_JOIN:ExternalGroups EQUALS corp.local/Users/NetOps_Group`
       - Shell Profile: `Priv7_Profile`

#### 【検証方法】
1. ルーターへ SSH ログインを試行:
   ```bash
   ssh -l netadmin01 192.168.1.1
   ```
2. ログイン後、`show privilege` コマンドを実行し、`Current privilege level is 15` と表示されることを確認。

---

### 演習 8: Active Directory Site Affinity に基づく PSN と Local DC の優先バインド設定

#### 【問題】
東京拠点にある PSN (`ise-tokyo-psn`) が、WAN 経由の米国 DC ではなく、必ず東京のローカル DC (`tokyo-dc01.corp.local`) と優先して通信するように AD サイト設定を反映せよ。

#### 【要件】
1. AD 上の Site 名: `Tokyo-Site`
2. ISE ジョインポイントで Site 名を `Tokyo-Site` に明示的に関連付けるか、自動検出させること。

#### 【設定例】

##### ISE GUI / CLI 設定手順
1. Active Directory 内の「Active Directory Sites and Services」にて、`ise-tokyo-psn` の IP サブネット（例: `10.20.0.0/24`）が `Tokyo-Site` に割り当てられていることを確認。
2. ISE CLI で Site 検出テストを実行:
   ```bash
   ise-tokyo-psn/admin# test ad-connectivity domain corp.local
   ```
3. CLI 出力結果で `Discovered Site: Tokyo-Site` および選択された DC が `tokyo-dc01.corp.local` であることを確認。

#### 【検証方法】
1. ISE CLI で現在アクティブな DC 接続状態を表示:
   ```bash
   ise-tokyo-psn/admin# show ad-status AD_CORP_JOIN
   ```
2. 接続先 DC の IP アドレスが東京 DC の IP であることを確認。

---

### 演習 9: Active Directory 属性取得のパフォーマンス最適化構成

#### 【問題】
AD ドメイン内に 50,000 以上のユーザーが存在する環境で、ISE の認可判定における AD グループ検索オーバーヘッドを最小化せよ。

#### 【要件】
1. グループ検索のスコープを制限し、必要最小限の外部グループのみを同期対象に設定すること。
2. ネストされたグループ検索の無効化（または階層制限）を適用すること。

#### 【設定例】

##### ISE GUI 設定手順
1. **Administration > Identity Management > External Identity Sources > Active Directory > AD_CORP_JOIN**。
2. **Groups** タブを開き、使用していない未定義グループを全削除し、認可ポリシーに必要な 3 つのグループのみ残す。
3. **Advanced Settings** タブに移動:
   - **Optimize / Limit Nested Group Membership Lookup**: 有効化
4. 保存を実行。

#### 【検証方法】
1. ユーザー認証時の処理時間を比較。Live Logs の `Response Time` が大幅に短縮（数秒 -> 数百ミリ秒）されることを確認。

---

### 演習 10: Passive Identity (AD WMI / Agent 連携) による IP-to-User マッピング

#### 【問題】
Active Directory ドメインコントローラのセキュリティイベントログからドメインユーザーのログインイベント（Event ID 4624）を ISE が検知し、IP-to-User 識別情報を自動生成するように構成せよ。

#### 【要件】
1. WMI または PassiveID Agent を用いて AD DC `192.168.10.10` と連携すること。
2. 取得した IP-to-User セッション情報を ISE の Session Directory に登録すること。

#### 【設定例】

##### ISE GUI 設定手順
1. **Work Centers > PassiveID > Providers > Active Directory** に移動し、**Add**。
   - Domain: `corp.local`
   - IP Address: `192.168.10.10`
   - Connection Type: `WMI`
   - WMI Username: `corp\passiveid_svc`
   - WMI Password: `Password123!`
2. **Test Connection** を実行し、成功したら **Save**。
3. **Work Centers > PassiveID > Overview** で リアルタイムにユーザーログインイベントが受信されていることを確認。

#### 【検証方法】
1. クライアント PC から AD ドメインへログイン。
2. ISE GUI **Context Visibility > Users** または CLI コマンドで IP と ユーザー名のマッピングを確認:
   ```bash
   ise-psn01/admin# show application status ise
   ```


## ❓ 想定試験問題

CCIE Security ラボ試験および筆記試験を意識した実践的な想定問題と詳細解説です。

---

### 質問 1 (コンフィグ読解・トラブルシュート)

#### 【問題】
ISE ノードを Active Directory ドメイン `corp.local` へジョインさせようとしたところ、以下のエラーが出力されて失敗しました。

```text
Error: Clock skew too great (37) / Failed to join domain corp.local.
```

ISE CLI で `show ntp` を実行したところ、以下の出力が得られました。

```text
ise-psn01/admin# show ntp
Synchronized: yes
NTP Server: 10.1.1.10 (Stratum 2)
Time: Thu Nov 12 10:15:00 UTC 2026
```

一方、Active Directory DC (`192.168.10.10`) で日時を確認したところ `Thu Nov 12 10:22:30 UTC 2026` でした。このエラーの原因と具体的な修正手順を述べよ。

#### 【解答・解説】
* **原因**: ISE ノードと Active Directory ドメインコントローラ間の時刻差（Clock Skew）が **7 分 30 秒** 存在しており、Kerberos 認証の最高許容誤差である **5 分 (300 秒)** を超えているため、ドメイン参加処理時の Kerberos TGT 発行が拒否されています。
* **修正手順**:
  1. ISE の NTP 設定を見直し、Active Directory DC が参照している NTP サーバー (`10.1.1.10`) または AD DC 自身 (`192.168.10.10`) と同等の高精度な時刻ソースに同期させます。
  2. CLI から手動で時刻を修正するか、NTP デーモンを再起動して同期を強制します。
  3. 時刻差が数秒以内に収まったことを確認後、再度 GUI または CLI から `identity-server ad join corp.local` を実行します。

---

### 質問 2 (実装・プロトコル動作)

#### 【問題】
802.1X EAP-PEAP (MSCHAPv2) 認証において、Cisco ISE が Active Directory に対して行う認証処理と、EAP-TLS 認証において行う処理の決定的な違いについて、ネットワークパケットおよびプロトコルレベルで説明せよ。

#### 【解答・解説】
* **EAP-PEAP (MSCHAPv2)**:
  - 端末から送られた チャレンジ/レスポンス (NT-Response) を ISE 内部で検証することはできません（パスワードハッシュを保持していないため）。
  - そのため、ISE は **MS-RPC (NETLOGON プロトコル)** を使用して AD ドメインコントローラへ認証リクエストを転送（Pass-Through Authentication）し、AD 側で NT-Response の整合性を直接検証させます。
* **EAP-TLS (証明書認証)**:
  - クライアント証明書（X.509）の暗号検証および署名検証は、ISE 内部で完結します（AD DC へのパススルー認証は発生しません）。
  - 認証完了後、ISE は証明書の Subject や SAN (UPN/email) から文字列を抽出し、**LDAP または Kerberos を用いて AD を検索（Identity Lookup）** して対象アカウントの有効性 (`userAccountControl`) および所属グループ (`memberOf`) を参照し、認可ポリシーを適用します。


---

### 質問 3 (Design・セキュリティ要件)

#### 【問題】
大規模企業環境において、Active Directory のフォレスト構造が `na.corp.local` (北米) と `eu.corp.local` (欧州) に分かれており、両者間に Two-way External Trust が設定されている。東京拠点に配置された ISE PSN が欧州ドメインのユーザーを認証する際、WAN リンクの遅延によって RADIUS Timeout が多発している。この問題を改善するための最適な設計策を 2 つ挙げよ。

#### 【解答・解説】
1. **各地域への ISE PSN の分散配置と Active Directory Site Affinity**:
   - 欧州拠点にローカル ISE PSN を配置し、欧州拠点のスイッチ/WLC からの RADIUS リクエストを欧州 PSN へ誘導する。各 PSN は AD Site 認識により近接するローカル DC (`eu.corp.local`) を直接参照させる。
2. **Identity Source Sequence (ISS) および Selective Group Import の最適化**:
   - 欧州ユーザー専用の Policy Set を分離し、認証サーチリストの先頭に `eu.corp.local` を配置することで無駄な `na.corp.local` への検索試行を排除する。
   - さらに、必要最小限の AD グループのみを ISE にインポートし、ネストグループの過度な再帰検索（Nested Group Lookup）を無効化・制限することで、レスポンスタイムを大幅に削減する。


---

### 質問 4 (トラブルシューティング・ログ解読)

#### 【問題】
ユーザーが EAP-PEAP でログインを試みた際、ISE Live Logs に以下の詳細イベントが記録され、アクセスが拒否されました。

```text
Event: 5400 Authentication failed
Failure Reason: 11007 Account disabled in Active Directory
Identity Store: AD_CORP_JOIN
```

このログが意味する状態と、管理者としての対応手順を説明せよ。

#### 【解答・解説】
* **状態の意味**: ユーザーが入力したアカウント情報自体は Active Directory 上で正しく確認されましたが、Active Directory のアカウント設定において対象ユーザーのオブジェクトが **「Disabled (無効化)」** に設定されているため、ISE が Access-Reject を返しています。
* **対応手順**:
  1. Active Directory Users and Computers (ADUC) または PowerShell で対象ユーザーアカウントを開く。
  2. アカウントのステータスを確認し、「Account is disabled」のチェックを外してアカウントを有効化する。
  3. ユーザーに再度接続を行わせ、認証が成功することを確認する。

---

### 質問 5 (コンフィグ設定・認可ポリシー)

#### 【問題】
Cisco ASA への SSH 管理アクセスにおいて、Active Directory 内の `Firewall_Admins` グループに所属するユーザーのみに Privilege Level 15 を与える ISE TACACS+ Policy Set を構成する手順の要点を説明せよ。

#### 【解答・解説】
1. **AD Group のインポート**: ISE の AD ジョインポイント設定で `corp.local/Groups/Firewall_Admins` を外部グループとしてインポートしておく。
2. **Shell Profile の作成**: **TACACS Shell Profiles** で、`Privilege 15` を指定した Shell Profile（例: `ASA_Priv15_Profile`）を作成する。
3. **TACACS+ Policy Set の作成**:
   - **Authentication Policy**: Use 欄に AD ジョインポイント（例: `AD_CORP_JOIN`）を指定。
   - **Authorization Policy**:
     - Condition: `AD_CORP_JOIN:ExternalGroups EQUALS corp.local/Groups/Firewall_Admins`
     - Command Sets: `PermitAllCommands`
     - Shell Profile: `ASA_Priv15_Profile`


---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび Cisco Live の参考資料一覧です。

* Cisco Identity Services Engine Administrator Guide - Active Directory Integration
* Cisco Technical Notes: Troubleshoot Active Directory Integration in Cisco ISE
* Cisco Live On-Demand Library: Advanced Cisco ISE Operations and Troubleshooting with Active Directory
* Cisco ISE Performance and Scalability Guide - External Identity Sources Best Practices
* Cisco Configuration Example: Configure ISE Native Active Directory Integration

---

## 📝 補足（Notes）

* **ドメイン参加時のコンピュータアカウント権限要件**:
  - ISE を AD ドメインに参加させるアカウントは、特定 OU に対するオブジェクト作成権限があれば十分です。
* **DC Locator の優先順位**:
  - ISE は DNS SRV レコードを尊重しますが、CLDAP Ping の結果得られる Subnet/Site Affinity が最も優先されます。
* **暗号化通信の保護**:
  - EAP-MSCHAPv2 認証は TLS トンネルによって保護されます。


