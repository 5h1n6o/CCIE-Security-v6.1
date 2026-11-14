---
layout: default
title: 4.7-External-identity
nav_order: 7
parent: 4.0-Identity-Management
---

# 4.7 Cisco ISE integration with external identity sources

## 📘 概要

Cisco Identity Services Engine (ISE) における外部 ID ソース（External Identity Sources）統合は、ネットワークアクセス制御、VPN 接続、管理者ログイン、および BYOD / ポータル認証において、企業の既存認証基盤（Active Directory, LDAP, RSA SecurID, SAML IdP, ODBC 等）と連携してエンドユーザーおよびデバイスの識別・認証・認可を行うコア機能です。

単一のローカルデータベースのみに依存せず、Active Directory ドメインコントローラや LDAP ディレクトリ、SAML 2.0 Identity Provider (IdP) と動的に通信することで、ユーザー属性（所属グループ、役職、アカウント有効期限、パスワード変更フラグ等）を取得し、ISE の Policy Sets における動的認可ポリシー（Authorization Policy）へリアルタイムに反映します。

主な適用場面：
- **802.1X / MAB 企業ネットワークアクセス**: 企業所有デバイスおよび従業員アカウントを Active Directory / LDAP と照合して EAP-TLS / PEAP / EAP-FAST 認証を実施。
- **AnyConnect / FlexVPN / Remote Access VPN**: 二要素認証 (2FA / MFA) 用の RSA SecurID や SAML IdP (Azure AD, Okta) と連携したセキュアリモートアクセス。
- **Guest / BYOD 認証**: SAML 2.0 SSO ポータルまたは LDAP/AD 統合型 WebAuth による認証とプロビジョニング。
- **Identity Source Sequences (ISS)**: 複数の ID ソース（ローカル DB → AD → LDAP）を順次検索・フォールバックさせる多重認証チェーンの構築。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **対象機能** | Active Directory, LDAP, RSA SecurID / RADIUS Token, SAML 2.0 (IdP), ODBC, Identity Source Sequences (ISS) |
| **主要用途** | ユーザー/デバイス認証、属性（Group, MemberOf）取得、パスワード変更（Change Password）、MFA/SSO 連携 |
| **メリット** | 既存企業ディレクトリの再利用、リアルタイム属性グループ同期、シングルサインオン (SSO) 、高度なグループベース認可 |
| **デメリット** | 外部サーバ依存によるレイテンシ増加、ドメインコントローラ/LDAP 障害時の認証影響、証明書/時刻同期要求 |
| **対応プロトコル** | Kerberos, MS-RPC (SMB/NetBIOS), LDAP/LDAPS (TCP 389/636), RADIUS (UDP 1812), SAML 2.0 (HTTPS TCP 443) |
| **制限事項** | AD 複数フォレスト参加時の双方向信頼関係要件、EAP-PEAP/MSCHAPv2 認証時の暗号化パスワード要求（NTHash） |
| **設計上の注意点** | PSN ノード毎の AD Join 状態維持、DNS 逆引き/正引き完全性、NTP 時刻同期（許容差 5分以内）、ISS のフォールバックルール設計 |

---

## 🏗 動作原理

Cisco ISE は Identity Mapping エンジンを備えており、認証要求を受け取ると指定された Identity Source または Identity Source Sequence (ISS) に従って外部サーバへ照会を行います。

```
Client (Supplicant / AnyConnect)
   ↓ (EAP / RADIUS / HTTPS)
Network Access Device (Switch / WLC / ASA)
   ↓ (RADIUS / TACACS+)
Cisco ISE (Policy Service Node - PSN)
   │
   ├─► [Active Directory Engine] ─── (Kerberos / MS-RPC) ───► AD Domain Controller
   ├─► [LDAP Identity Store] ──── (LDAPS TCP 636) ────────► OpenLDAP / AD LDS
   ├─► [SAML IdP Service] ──────── (HTTPS TCP 443) ────────► Azure AD / Okta IdP
   └─► [RADIUS Token Store] ────── (RADIUS UDP 1812) ─────► RSA Authentication Manager
```

### 外部 ID ソース別の通信特性

1. **Active Directory (AD)**:
   - Cisco ISE PSN は Native AD Client として動作し、AD ドメインに直接バインド（Join）します。
   - 認証時、Kerberos 認証または MS-RPC による MSCHAPv2 資格情報検証を実施します。
   - 属性取得時、Global Catalog (TCP 3268/3269) または LDAP (TCP 389/636) を自動的にクエリします。

2. **LDAP ディレクトリ**:
   - スキーマ（OpenLDAP, AD LDS, Sun ONE 等）に合わせて検索ベース (Search Base) および属性マッピングを設定。
   - ユーザーパスワード検証は `Bind` (Simple Bind) または `Compare` 操作を使用。

3. **SAML 2.0 Identity Provider**:
   - Web 認証（CWA, Guest Portal, Admin Login, AnyConnect SAML）で使用。
   - ユーザーは IdP ポータルへリダイレクトされ、IdP で認証後、SAML Assertion (HTTPS POST) が ISE へ送信されます。

4. **Identity Source Sequence (ISS)**:
   - 複数の Identity Store をグループ化し、優先順位を決定。
   - 照会結果（`Authentication Failed`, `User Not Found`, `Process Error`）に応じて「Advance to next store」「Treat as Authentication Failed」「Continue」等の動作を細かく制御。

---

## ⚙ 動作シーケンス

例：Active Directory を用いた 802.1X (PEAP-MSCHAPv2) 認証および認可フロー

```
Supplicant              Authenticator (Switch)             Cisco ISE (PSN)            Active Directory (DC)
    │                             │                              │                             │
    │─── EAPOL-Start ────────────►│                              │                             │
    │◄── EAP-Request/Identity ────│                              │                             │
    │─── EAP-Response/Identity ──►│─── RADIUS Access-Request ───►│                             │
    │                             │    (User: alice)             │                             │
    │◄── EAP-TLS Tunnel Est. ─────┼─────────────────────────────►│                             │
    │    (TLS Session Established)│                              │                             │
    │                             │                              │                             │
    │─── MSCHAPv2 Challenge ─────┼─────────────────────────────►│                             │
    │    Response                 │                              │                             │
    │                             │                              │── Netlogon / MS-RPC ───────►│
    │                             │                              │   Credential Validation     │
    │                             │                              │◄── Auth Success (NTHash OK)─│
    │                             │                              │                             │
    │                             │                              │── LDAP / GC Query ─────────►│
    │                             │                              │   Query MemberOf Attributes │
    │                             │                              │◄── MemberOf: Domain Admins ─│
    │                             │                              │                             │
    │                             │◄── RADIUS Access-Accept ─────│                             │
    │                             │    (dACL / SGT = 4 / VLAN 10)│                             │
    │◄── EAP-Success ─────────────│                              │                             │
```

1. **Supplicant 接続と EAP セッション確立**: クライアントと ISE 間で PEAP 用のアウター TLS トンネルを確立。
2. **インナー MSCHAPv2 認証**: クライアントから送信された MSCHAPv2 チャレンジレスポンスを ISE が受信。
3. **AD ドメインコントローラ照会**: ISE PSN は Netlogon / MS-RPC アクションを通じて AD DC へ資格情報検証を要求。
4. **属性・グループマッピングクエリ**: 認証成功後、ISE は AD の Global Catalog / LDAP プロトコルを用いて対象アカウントの `memberOf` 属性を検索。
5. **Policy Set 評価と Access-Accept 返答**: 取得した `ActiveDirectory:ExternalGroups` 属性が Policy Set の Authorization Rule 条件に合致した場合、dACL / SGT / VLAN を含む RADIUS Access-Accept をスイッチへ送出。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Securityラボ試験において、ISE と外部 ID ソースの統合は、単独の課題にとどまらず、802.1X/MAB, TACACS+ 管理者認証, AnyConnect SAML VPN, CWA ゲスト認証の前提となる極めて重要な技術領域です。

### ラボ試験における頻出要件とポイント

1. **Active Directory Join & Domain Health**:
   - すべての PSN ノードが指定された AD ドメイン（例: `corp.local`）に正常に Join していること。
   - DNS サーバー（正引き・逆引き）が正しく設定されていない場合、AD Join や 認証処理がタイムアウトするため、まず DNS 設定を CLI (`show running-config`) で確認する。
   - NTP の同期状態（`show ntp`）の確認。時刻が 5分以上ズレると Kerberos 認証および AD Join に失敗する。

2. **Identity Source Sequences (ISS) の例外処理制御**:
   - 試験要件例：「ローカルデータベースにユーザーが存在する場合はローカルで認証し、存在しない場合は Active Directory を参照せよ。AD で認証失敗した場合は即座に Reject し、AD 障害時にはローカル DB をフォールバック利用せよ。」
   - この場合、ISS の設定画面で：
     - `User Not Found` → `Advance to next store`
     - `Authentication Failed` → `Treat as Authentication Failed` (Stop)
     - `Process Error` → `Advance to next store` または `Continue`
   を正しく選択・設定する必要がある。

3. **SAML 2.0 IdP 統合**:
   - AnyConnect リモートアクセス VPN や CWA ポータル認証における SAML 連携。
   - Single Sign-On (SSO) 属性マッピングの適合。
   - IdP 側と ISE 側の Signing Certificate / Encryption Certificate のインポートおよびメタデータ XML の相互登録。

4. **LDAP 属性マッピングと Simple Bind**:
   - Secure LDAP (LDAPS) TCP 636 の利用と CA 証明書の ISE Trusted Certificates への登録。
   - LDAP ユーザーオブジェクトの `sAMAccountName` または `uid` の検索ベース指定。

5. **よくある設定ミス・落とし穴**:
   - **AD 外部グループ（External Groups）の事前取得漏れ**: Policy Set の条件で `ActiveDirectory:ExternalGroups EQUALS corp.local/Users/Domain Admins` を使用する際、ISE の `Active Directory > Groups` タブで該当グループを明示的に Add/Fetch していないと、ポリシー条件が成立しない。
   - **証明書信頼チェーンの不足**: LDAPS や SAML で利用する外部サーバのルート CA / 中間 CA 証明書が ISE の 「Trusted Certificates」 にインポートされておらず、かつ「Trust for authentication within ISE」チェックボックスがオフになっていることによる接続エラー。

---

## 🛠 設定方法

### 1. Active Directory 統合手順（GUI設定）

1. **DNS 及び NTP 設定の確認（CLI/GUI）**:
   - `Administration > System > Network Resources > IP Sockets` または CLI にて DNS 参照先および NTP 同期状態を確認。

2. **Active Directory Join**:
   - **ナビゲーション**: `Administration > Identity Management > External Identity Sources > Active Directory`
   - 「Add」をクリックし、**Join Point Name**（例: `Corp_AD`）と **Active Directory Domain Name**（例: `corp.local`）を入力。
   - 「Submit」後、表示されるダイアログでドメイン管理者資格情報を入力し、PSN ノードを Join させる。
   - 状態が `Completed` / `Operational` になることを確認。

3. **AD グループの取得 (Fetch Groups)**:
   - 作成した AD Join Point を開き、**Groups** タブを選択。
   - 「Add > Select Groups from Directory」をクリックし、「Retrieve Groups」を実行。
   - 認可ポリシーで使用するグループ（例: `Domain Admins`, `Sales-Group`）にチェックを入れ、「Save」する。

4. **属性の同期 (Attributes)**:
   - **Attributes** タブにて、必要に応じて追加属性（例: `department`, `title`, `telephoneNumber`）を選択してインポート。

---

### 2. Identity Source Sequences (ISS) の作成手順

1. **ナビゲーション**: `Administration > Identity Management > Identity Source Sequences` に移動し「Add」をクリック。
2. **基本設定**: Name に `Corp_ISS_Sequence` と入力。
3. **ID Store 順序設定**:
   - 「Certificate Based Authentication」または「Password Based Authentication」リストにおいて、検索順序を設定：
     1. `Internal Users`
     2. `Corp_AD`
     3. `Corp_LDAP`
4. **Advanced Search Rules (例外ハンドリング)**:
   - **If User Not Found**: `Search next store`
   - **If Authentication Failed**: `Do not access next store and process as failure`
   - **If Process Error**: `Search next store`

---

### 3. SAML 2.0 Identity Provider (External IdP) 統合

1. **ナビゲーション**: `Administration > Identity Management > External Identity Sources > SAML IdPs`
2. **IdP 追加**: 「Add」をクリックし、Name に `External_IdP` を指定。
3. **メタデータ インポート**:
   - IdP からダウンロードした XML メタデータファイルを「Import Service Provider Metadata」または「IdP Metadata」からアップロード。
4. **Assertion Attributes マッピング**:
   - Attribute Name: `http_schema_groups` (例: group claim)
   - ISE Attribute: `External Groups`


## 🔍 検証コマンド

ISE CLI および GUI (Live Logs / Diagnostics) における検証コマンド一覧：

| 目的 | コマンド / UI 操作 |
| --- | --- |
| **AD 参加状態・診断確認** | `show ad-domain-controller` <br> `test ad-connector corp.local` |
| **NTP 同期状態確認** | `show ntp` |
| **DNS 解決テスト** | `nslookup dc01.corp.local` |
| **AD Join ログ（CLI）** | `show logging application active | include ad` |
| **AD 認証デバッグ** | `debug logging component active-directory level debug` |
| **LDAP 接続デバッグ** | `debug logging component runtime-ldap level debug` |
| **RADIUS / 認証エンジンログ** | `debug logging component prt-runtime level debug` |
| **ISE RADIUS Live Logs** | GUI: `Operations > RADIUS > Live Logs` |
| **AD 接続診断 (GUI)** | GUI: `Administration > Identity Management > External Identity Sources > Active Directory > Diagnostic Tools` |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **AD Join 失敗 (Clock skew)** | ISE と AD DC 間の時刻同期ズレが 5分以上 | <code>show ntp</code> | NTP サーバー設定を見直し、時刻を同期させる |
| **AD Join 失敗 (DNS lookup failure)** | ISE から AD DC の SRV レコード / A レコードが解決できない | <code>nslookup _ldap._tcp.dc._msdcs.corp.local</code> | ISE の DNS 設定に AD 統合 DNS サーバーを優先登録 |
| **ユーザー認証成功も認可ポリシー不一致** | AD の External Groups が ISE 側で Fetch / 登録されていない | GUI: `RADIUS Live Logs > Auth Details` | ISE の AD Join Point > Groups タブで対象グループを追加 |
| **LDAPS 接続エラー (TCP 636)** | LDAPS 用 CA 証明書が ISE Trusted Certificates に登録されていない | GUI: `Trusted Certificates` | ルート/中間 CA 証明書をインポートし Authentication 用に Trust 設定 |
| **ISS で不要なフォールバック発生** | `User Not Found` や `Auth Failed` の処理ルール設定ミス | GUI: `Identity Source Sequences` | 「If Auth Failed → Treat as Failure」等のルールを正しく再定義 |
| **SAML SSO 認証後に ISE でエラー** | SAML Assertion の NameID / Group Claim 名の不一致 | GUI: `SAML IdPs > Attribute Mapping` | IdP 側の Claim 名と ISE 側の Mapping 属性を合致させる |

---

## ⚠ 制限事項

* **Kerberos / AD クロスフォレスト信頼の制約**: 外部フォレストとの信頼関係は双方向信頼 (Two-way trust) が推奨され、一方向信頼 (One-way trust) の場合は制約が発生する。
* **EAP-PEAP / MSCHAPv2 の認証暗号制約**: MSCHAPv2 認証を行う場合、AD 側に暗号化されたパスワード（NTHash）が必要。明細テキストや可逆暗号が必要な認証方式（EAP-GTC 等）では注意が必要。
* **PSN ノード毎の独立 Join**: 各 PSN は個別に AD に参加するため、特定 PSN のみ Join 障害が発生するケースがある。

---

## 🔄 他技術との関連

* **Policy Sets (Authorization Policies)**: 外部 ID ソースから取得した `ExternalGroups` や属性（`department` 等）を直接ポリシー条件として使用。
* **802.1X / MAB**: エンドポイント認証時に EAP-TLS（証明書 SAN / UPN を AD/LDAP と照合）や PEAP-MSCHAPv2 を外部照合。
* **AnyConnect VPN / SAML**: SAML 2.0 連携により Azure AD / Okta で MFA 認証を行い、ISE へ RADIUS/TACACS+ 認可を反映。
* **TACACS+ Device Administration**: 管理者アカウントを Active Directory の `Domain Admins` / `Network-Admins` に照合して Command Sets / Shell Profiles を割り当て。

---

## 🧩 比較表

### Active Directory vs LDAP vs SAML 2.0

| 比較項目 | Active Directory (Native) | LDAP / LDAPS | SAML 2.0 (IdP) |
| --- | --- | --- | --- |
| **通信プロトコル** | Kerberos, MS-RPC, LDAP | LDAP (TCP 389) / LDAPS (TCP 636) | HTTPS (TCP 443) |
| **パスワード検証** | MSCHAPv2, Kerberos | Simple Bind, Compare | IdP 側で認証 (SSO/MFA) |
| **グループ取得** | 自動クエリ (Global Catalog) | 検索ベース & memberOf 属性 | SAML Assertion Claim |
| **主用途** | 社内 802.1X, TACACS+, PEAP | 汎用ディレクトリ, 非AD環境 | WebAuth, Guest, VPN MFA |
| **パスワード変更** | サポート (Password Change) | ディレクトリ設定による | IdP 側で処理 |

---

## 💡 ベストプラクティス

1. **DNS 冗長化と SRV レコード依存**: ISE の DNS 設定には必ず AD 統合 DNS サーバーを主・副で指定し、AD DC の自動検出（SRV レコード）を機能させる。
2. **グループ参照の最小化**: AD から必要以上のグループを Fetch せず、認可ポリシーで使用するグループのみを選択してインポートすることで、クエリ処理速度を向上させる。
3. **Identity Source Sequence (ISS) の明確な分離**: 802.1X 用、TACACS+ 用、VPN 用で別々の ISS を定義し、無駄な外部問い合わせを削減する。


## 📝 ラボ学習・設定サンプル例

### 演習 1: ISE の Active Directory Join とドメイン診断構成

#### 問題
ISE PSN ノードを企業 Active Directory ドメイン (`corp.local`) に参加させ、ドメイン接続状態を検証せよ。

#### 要件
- AD ドメイン名: `corp.local`
- AD Join Point Name: `Corp_AD_Join`
- 事前に DNS 正引き・逆引きおよび NTP 同期が完了していること。
- CLI から AD コネクタの健全性をテストすること。

#### 設定例

1. **DNS 及び NTP 設定の確認 (ISE CLI)**:
```bash
# ISE CLI にて DNS および NTP を確認
show running-config | include nameserver
show ntp
```

2. **Active Directory Join (ISE GUI)**:
- ナビゲーション: `Administration > Identity Management > External Identity Sources > Active Directory`
- 「Add」をクリック:
  - Join Point Name: `Corp_AD_Join`
  - Active Directory Domain Name: `corp.local`
- 「Submit」をクリック。
- ダイアログにてドメイン管理者資格情報を入力し、「Join」を実行。

#### 検証方法
```bash
# ISE CLI にて AD ドメインコントローラ接続状態の確認
show ad-domain-controller

# AD コネクタ診断テストの実行
test ad-connector corp.local
```

---

### 演習 2: AD 外部グループを使用した 802.1X 認可ポリシー設定

#### 問題
Active Directory から取得した `Domain Admins` グループのユーザーが 802.1X 認証に成功した場合、SGT 4 (Enterprise-Admin) および VLAN 10 を動的に割り当てる認可ルールを作成せよ。

#### 要件
- Identity Source: `Corp_AD_Join`
- AD Group: `corp.local/Users/Domain Admins`
- Rule Name: `Rule_AD_Domain_Admins`
- Action: SGT `Enterprise-Admin` (Tag 4) + Dynamic VLAN `10_Admin`

#### 設定例

1. **AD グループの取得 (ISE GUI)**:
- `Administration > Identity Management > External Identity Sources > Active Directory > Corp_AD_Join`
- **Groups** タブ > 「Add > Select Groups from Directory」
- 「Retrieve Groups」をクリックし、`Domain Admins` にチェックを入れて保存。

2. **Authorization Policy の作成**:
- ナビゲーション: `Policy > Policy Sets > Default (または対象 Policy Set)`
- Authorization Policy に新規ルールを追加:
  - Rule Name: `Rule_AD_Domain_Admins`
  - Condition: `Corp_AD_Join:ExternalGroups EQUALS corp.local/Users/Domain Admins`
  - Security Group: `Enterprise-Admin`
  - Profiles: `VLAN10_Profile` (VLAN 10)

#### 検証方法
- テスト端末から 802.1X PEAP 認証を実行。
- ISE GUI: `Operations > RADIUS > Live Logs` を確認。
- 該当認証ログの 「Auth Details」 を開き、`Matched Rule = Rule_AD_Domain_Admins` および `SelectedSecurityGroup = Enterprise-Admin` であることを確認。

---

### 演習 3: LDAPS (TCP 636) 外部 ID ストアの追加と証明書信頼

#### 問題
外部 OpenLDAP サーバー (`ldap.corp.local`, IP: `192.168.10.50`) を LDAPS (TCP 636) で ISE に登録せよ。

#### 要件
- Identity Store Name: `OpenLDAP_Store`
- Host Domain / IP: `192.168.10.50`, Port: `636` (Use SSL/TLS)
- Root CA 証明書を Trusted Certificates にインポートし、Authentication 用に信頼させること。
- Base DN: `ou=users,dc=corp,dc=local`
- User Identity Attribute: `uid`

#### 設定例

1. **Root CA 証明書のインポート (ISE GUI)**:
- `Administration > System > Certificates > Trusted Certificates`
- 「Import」をクリックし、OpenLDAP の Root CA 証明書を選択。
- 「Trust for authentication within ISE」にチェックを入れて保存。

2. **LDAP Store の構成**:
- `Administration > Identity Management > External Identity Sources > LDAP`
- 「Add」をクリック:
  - General タブ: Name = `OpenLDAP_Store`
  - Connection タブ:
    - Primary Server: `192.168.10.50`, Port: `636`, Check 「Use SSL」
    - Admin DN: `cn=admin,dc=corp,dc=local`
    - Base DN: `ou=users,dc=corp,dc=local`
  - Directory Organization タブ:
    - Naming Attribute: `uid`
    - Object Class: `inetOrgPerson`

#### 検証方法
- Connection タブの下部にある 「Test Connection」 をクリックし、`Test Successful` と表示されることを確認。

---

### 演習 4: Identity Source Sequence (ISS) によるローカル DB と AD の優先度制御

#### 問題
ユーザー認証時に、まず ISE ローカルデータベースを検索し、存在しない場合は Active Directory (`Corp_AD_Join`) を検索する Identity Source Sequence を作成せよ。

#### 要件
- ISS Name: `ISS_Local_then_AD`
- Authentication Order:
  1. `Internal Users`
  2. `Corp_AD_Join`

#### 設定例

1. **ISS の作成 (ISE GUI)**:
- `Administration > Identity Management > Identity Source Sequences`
- 「Add」をクリック:
  - Name: `ISS_Local_then_AD`
  - 「Select the Identity Sources...」 リストから以下を右側に移動し順序設定:
    - `Internal Users` (最上位)
    - `Corp_AD_Join` (2番目)

2. **Policy Set への適用**:
- `Policy > Policy Sets` を開き、対象 Policy Set の Authentication Policy を展開。
- Use / Authentication Source として `ISS_Local_then_AD` を選択して保存。

#### 検証方法
- ローカルユーザーで認証し、成功することを確認。
- ローカルに存在しない AD ユーザーで認証し、AD 側で成功することを確認。

---

### 演習 5: ISS における Exceptions / Advanced Search Rules の制御

#### 問題
演習 4 の ISS において、Active Directory で認証失敗（パスワード間違い）した場合は他のストアを検索せず即座に Reject し、AD サーバー障害（Process Error）の場合のみ次へ進むルールを設定せよ。

#### 要件
- If User Not Found: `Search next store`
- If Authentication Failed: `Do not access next store and process as failure`
- If Process Error: `Search next store`

#### 設定例

1. **Advanced Search Rules の調整 (ISE GUI)**:
- `Administration > Identity Management > Identity Source Sequences > ISS_Local_then_AD`
- **Advanced Search Rules** セクションを展開:
  - `If User Not Found`: **Search next store**
  - `If Authentication Failed`: **Do not access next store and process as failure**
  - `If Process Error`: **Search next store**
- 設定を保存。

#### 検証方法
- 誤ったパスワードで AD ユーザーの認証を実行。
- RADIUS Live Logs にて、認証失敗理由が `15039 EAP-MSCHAPv2 Bad password` で即座に Reject され、後続ストアの照会が行われていないことを確認。


### 演習 6: SAML 2.0 Identity Provider 統合と CWA ポータル連携

#### 問題
Cisco ISE CWA (Central Web Authentication) ポータル認証において、外部 SAML IdP を Identity Source として統合せよ。

#### 要件
- IdP Name: `SAML_AzureAD`
- XML メタデータを用いて SAML IdP を構成。
- Guest / CWA Portal の認証ソースとして `SAML_AzureAD` を割り当てる。

#### 設定例

1. **SAML IdP の登録**:
- `Administration > Identity Management > External Identity Sources > SAML IdPs`
- 「Add」をクリック:
  - Name: `SAML_AzureAD`
  - Import Identity Provider Metadata: IdP の XML メタデータファイルをアップロード。
- 「Save」をクリック。

2. **CWA ポータルの構成**:
- `Work Centers > Guest Pages and Portals > Self-Service Sponsored Guest Portal`
- **Portal Settings > Authentication Options**:
  - Identity source sequence / store: `SAML_AzureAD` を選択。
- 設定を保存。

#### 検証方法
- WebAuth リダイレクト後、クライアントブラウザが SAML IdP ログイン画面にリダイレクトされることを確認。
- 認証成功後、ポータルからネットワークアクセスが許可されることを確認。

---

### 演習 7: TACACS+ 管理者アクセスにおける AD グループ認可

#### 問題
スイッチへの CLI ログインにおいて、AD の `Network-Admins` グループ所属ユーザーに Privilege Level 15、`Network-Operators` グループ所属ユーザーに Privilege Level 1 を割り当てる TACACS+ 認可ルールを構成せよ。

#### 要件
- AD Join Point: `Corp_AD_Join`
- Admin Group 1: `corp.local/Users/Network-Admins` → Privilege 15 Profile
- Admin Group 2: `corp.local/Users/Network-Operators` → Privilege 1 Profile

#### 設定例

1. **TACACS Policy Set の作成**:
- `Work Centers > Device Administration > Device Admin Policy Sets`
- Policy Set Name: `TACACS_Device_Admin`
- Authentication Policy: Use = `Corp_AD_Join`

2. **Authorization Policy Rules の作成**:
- **Rule 1**:
  - Name: `Rule_NetAdmin`
  - Condition: `Corp_AD_Join:ExternalGroups EQUALS corp.local/Users/Network-Admins`
  - Command Sets: `PermitAllCommands`
  - Shell Profile: `Privilege15_Profile`
- **Rule 2**:
  - Name: `Rule_NetOperator`
  - Condition: `Corp_AD_Join:ExternalGroups EQUALS corp.local/Users/Network-Operators`
  - Command Sets: `ShowOnly_CommandSet`
  - Shell Profile: `Privilege1_Profile`

#### 検証方法
- スイッチ CLI から SSH ログインテストを実施：
```bash
# Network-Admin ユーザーで SSH
ssh netadmin@192.168.1.1
# ログイン後、enable なしで privilege 15 (PROMPT#) になることを確認
show privilege

# Network-Operator ユーザーで SSH
ssh netop@192.168.1.1
# ログイン後、privilege 1 (PROMPT>) になることを確認
show privilege
```

---

### 演習 8: AD ユーザー属性（department）を利用した動的認可ルール

#### 問題
Active Directory ユーザーの `department` 属性が `Sales` である場合、専用の dACL (`DACL_SALES`) を適用する認可ルールを構築せよ。

#### 要件
- AD Join Point: `Corp_AD_Join`
- Fetch Attribute: `department`
- Rule Condition: `Corp_AD_Join:department EQUALS Sales`
- Profile Action: `DACL_SALES`

#### 設定例

1. **AD 属性の有効化**:
- `Administration > Identity Management > External Identity Sources > Active Directory > Corp_AD_Join`
- **Attributes** タブを開き、「Add > Select Attributes from Directory」から `department` を選択・追加して保存。

2. **Authorization Policy の構成**:
- `Policy > Policy Sets > Default`
- Authorization Policy にルール追加:
  - Name: `Rule_Sales_Department`
  - Condition: `Corp_AD_Join:department EQUALS Sales`
  - Permissions: `DACL_SALES` (dACL)

#### 検証方法
- `department=Sales` の AD アカウントで 802.1X 認証を実行。
- `Operations > RADIUS > Live Logs` で Auth Details を開き、`Matched Attribute: department = Sales` および `Applied dACL = DACL_SALES` を確認。

---

### 演習 9: ISE と RSA SecurID (RADIUS Token) 連携の構成

#### 問題
二要素認証用に外部 RADIUS Token サーバー (`192.168.10.100`) を RADIUS Token Identity Source として登録せよ。

#### 要件
- Identity Store Name: `RSA_SecurID_Token`
- Token Server IP: `192.168.10.100`, Shared Secret: `cisco123`
- Authentication Protocol: RADIUS (UDP 1812)

#### 設定例

1. **RADIUS Token Store の登録**:
- `Administration > Identity Management > External Identity Sources > RADIUS Token`
- 「Add」をクリック:
  - Name: `RSA_SecurID_Token`
  - Connection タブ:
    - Host IP: `192.168.10.100`
    - Shared Secret: `cisco123`
    - Auth Port: `1812`
- 「Save」をクリック。

2. **ISS への組み込み**:
- `Administration > Identity Management > Identity Source Sequences > ISS_VPN_MFA`
- Identity Sources に `RSA_SecurID_Token` を追加し、VPN 認可ポリシーで使用。

#### 検証方法
- VPN クライアントから OTP (Passcode + Token) を用いて接続要求を発行し、RADIUS Token サーバー側で認証が通過することを確認。

---

### 演習 10: 外部 ID ソース障害時のローカルフォールバック動作の検証

#### 問題
Active Directory 障害（ネットワーク断や全 DC 停止）発生時に、管理者ユーザーが ISE ローカルデータベースに定義されたフォールバックアカウントで認証できるように設定せよ。

#### 要件
- Identity Source Sequence: `ISS_Admin_Fallback`
- Primary Store: `Corp_AD_Join`
- Fallback Store: `Internal Users`
- Exception Rule: `If Process Error` → `Advance to next store`

#### 設定例

1. **ISS の構成**:
- `Administration > Identity Management > Identity Source Sequences > ISS_Admin_Fallback`
- Order:
  1. `Corp_AD_Join`
  2. `Internal Users`
- **Advanced Search Rules**:
  - `If User Not Found`: `Do not access next store and process as failure`
  - `If Authentication Failed`: `Do not access next store and process as failure`
  - `If Process Error`: **Search next store** (ここがフォールバックのカギ)

#### 検証方法
- AD サーバーとの通信を遮断（または ISE 側で不整合を発生）させる。
- ローカルユーザー `fallback_admin` で TACACS+ ログインを試し、ISE が AD エラー（Process Error）検知後に `Internal Users` を参照して認証成功することを確認。


## ❓ 想定試験問題

### 問 1 (コンフィグ・要件読解)
Cisco ISE の Identity Source Sequence (ISS) において、以下の要件を満たす設定として最も適切な選択肢を選びなさい。

**要件**:
- ユーザー認証要求を受信した際、まずローカル DB を参照し、存在しない場合は Active Directory を参照する。
- ローカル DB にユーザーが存在しパスワードが一致しない場合、即座に認証拒否とし、Active Directory への照会を行ってはならない。
- Active Directory との通信障害（Process Error）が発生した場合は、認証拒否すること。

A. `If User Not Found` = `Advance to next store` / `If Auth Failed` = `Advance to next store` / `If Process Error` = `Advance to next store`  
B. `If User Not Found` = `Advance to next store` / `If Auth Failed` = `Treat as Auth Failed` / `If Process Error` = `Treat as Auth Failed`  
C. `If User Not Found` = `Treat as Auth Failed` / `If Auth Failed` = `Advance to next store` / `If Process Error` = `Treat as Auth Failed`  
D. `If User Not Found` = `Advance to next store` / `If Auth Failed` = `Treat as Auth Failed` / `If Process Error` = `Advance to next store`

**正解**: **B**  
**解説**: `If User Not Found` で `Advance to next store` を選択することで、ローカル DB にユーザーがいない場合のみ AD を照会します。`If Auth Failed`（パスワード不一致）で `Treat as Auth Failed` を選択することで即座に Reject され AD への無駄な照会を防ぎます。`If Process Error` で `Treat as Auth Failed` を選択することで、AD 障害時のアクセスを遮断します。

---

### 問 2 (トラブルシューティング)
ISE PSN ノードを Active Directory ドメイン `corp.local` に Join させようとした際、以下のエラーが発生して Join に失敗しました。

`Error Name: Clock skew too great`

この問題の原因と確認コマンドの組み合わせとして正しいものを選択しなさい。

A. ISE と AD DC 間の TCP 636 ポートがブロックされている。確認コマンド: `show ports`  
B. ISE の DNS 逆引き設定が不足している。確認コマンド: `nslookup dc01.corp.local`  
C. ISE と AD DC 間の時刻同期ズレが Kerberos 許容値（5分）を超えている。確認コマンド: `show ntp`  
D. ISE の Active Directory ライセンスが無効化されている。確認コマンド: `show license`

**正解**: **C**  
**解説**: Kerberos プロトコルはリプレイ攻撃を防ぐために厳格な時刻同期を要求します。ISE と AD DC の時刻ズレが 5分を超えると `Clock skew too great` エラーが発生します。CLI の `show ntp` で同期状態を確認します。

---

### 問 3 (デザイン)
大規模企業における Cisco ISE と Active Directory の統合設計において、認証レイテンシを最小化し安定性を確保するためのベストプラクティスとして不適切なものを 1 つ選ばなさい。

A. ISE の DNS 設定には、AD 統合 DNS サーバーを優先して登録し、DC の SRV レコード検出を可能にする。  
B. ISE の AD Join Point 設定において、フォレスト内のすべてのグループを取得（Fetch All Groups）してポリシー処理速度を向上させる。  
C. 各 PSN ノードは、地理的に最も近い AD ドメインコントローラ (Sites) に優先接続するよう設計する。  
D. 認証失敗（Auth Failed）時には無駄な Identity Store 照会を行わないよう ISS を設計する。

**正解**: **B**  
**解説**: フォレスト内の全グループを取得（Fetch All Groups）すると、ISE のメモリ消費と属性クエリの処理レイテンシが増大します。認可ポリシーで実際に使用する特定のグループのみをインポートするのがベストプラクティスです。


### 問 4 (実装・SAML)
Cisco ISE と External SAML IdP を統合し、AnyConnect SAML リモートアクセス VPN を実現する場合、ISE 側でインポート必須となる要素として正しいものを 2 つ選びなさい。

A. SAML IdP から発行された Federation Metadata XML ファイル  
B. AD DC の Netlogon 共有キー  
C. IdP の Signing Certificate (ルート/中間 CA 証明書) を ISE の Trusted Certificates にインポート  
D. RSA Authentication Manager サーバーの Node Secret

**正解**: **A, C**  
**解説**: SAML 2.0 統合では、IdP が発行した Federation Metadata XML をインポートしてエンドポイント URL や Entity ID を設定し、SAML Assertion の署名検証用に IdP の Signing Certificate を Trusted Certificates に登録する必要があります。

---

### 問 5 (トラブルシューティング)
802.1X PEAP 認証において、ユーザーが AD の `Domain Admins` に所属しているにもかかわらず、ISE Live Logs でデフォルト認可ルール (`Default Rule`) にマッチしてアクセスが拒否されました。Auth Details を確認したところ、以下が記録されていました。

`External Groups: corp.local/Users/Domain Admins` (未取得)

この症状に対する適切な対処方法はどれか。

A. ISE CLI で `test ad-connector` コマンドを再実行する。  
B. ISE の Active Directory Join Point 設定画面の「Groups」タブで `Domain Admins` を Retrieve して追加・保存する。  
C. スイッチ側で `aaa authorization network default` コマンドを削除する。  
D. ISE の Active Directory Join を解除し、再度 Re-join を行う。

**正解**: **B**  
**解説**: ISE の Policy Set 条件で AD 外部グループ（`ExternalGroups`）を使用するには、AD Join Point の Groups タブで対象グループが明示的に追加・インポートされている必要があります。グループが未登録の場合、属性として評価されずデフォルトルールにフォールバックします。

---

## 🔗 参考リソース

Cisco 公式ドキュメント：

* Cisco Identity Services Engine Administrator Guide - Active Directory Integration
* Cisco ISE with SAML 2.0 Identity Providers Configuration Guide
* Configuring External Identity Sources in Cisco ISE
* Cisco Live On-Demand Library - ISE Architecture and External Identity Integration
* Troubleshooting Active Directory Integration in Cisco ISE

---

## 📝 **補足（Notes）**

### AD Join と DNS / NTP の関係チェックリスト

1. **DNS 正引き・逆引き**:
   - ISE から `nslookup <DC_FQDN>` および `nslookup <DC_IP>` がともに正しく返ること。
   - SRV レコード `_ldap._tcp.dc._msdcs.<Domain>` が解決できること。

2. **NTP 時刻同期**:
   - `show ntp` コマンドで `synchronised` となっていること。
   - 許容時刻差は 300秒（5分）以内。

3. **証明書チェーンの確認**:
   - LDAPS (TCP 636) 利用時は、LDAP サーバー証明書を発行した Root CA および 中間 CA を ISE の `Trusted Certificates` に登録し、`Trust for authentication within ISE` フラグを有効化すること。


