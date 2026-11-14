---
layout: default
title: 4.7.a-LDAP
nav_order: 1
parent: 4.7-External-identity
grand_parent: 4.0-Identity-Management
---

# 4.7.a LDAP

## 📘 概要

Cisco Identity Services Engine (Cisco ISE) における LDAP (Lightweight Directory Access Protocol) 統合は、Active Directory ネイティブドメイン参加が適用できない環境（Linux/Unix 基盤の OpenLDAP、FreeIPA、Novell eDirectory、またはマルチフォレスト環境や特殊な Active Directory スキーマ構成）において、外部アイデンティティソースとしてユーザー認証および属性検索を行うための標準プロトコル機能です。

本機能は、802.1X（EAP-TLS, PEAP-GTC, EAP-TTLS）、WebAuth（CWA/LWA）、TACACS+ デバイス管理アクセス、および VPN 接続におけるユーザー認証と、グループ所属情報や端末属性に基づく認可ポリシー（dACL, VLAN, Dynamic SGT）の動的割り当てを実現するために利用されます。

* **機能概要**: RFC 4511/4513 に準拠した LDAP プロトコルを用い、ISE が LDAP クライアント（Directory Consumer）として外部 LDAP サーバーにバインドし、ユーザー認証（Bind / Password Verify）およびディレクトリ属性の検索（Search Query）を実行する機能。
* **利用目的**: 異種 OS 環境やサードパーティ製ディレクトリサービスに登録されたユーザー情報の一元参照、証明書認証（EAP-TLS）時の証明書失効・属性検証、およびマルチアイデンティティソース構造における柔軟な認証統合。
* **利用場面**:
  1. OpenLDAP や FreeIPA 等の Linux ディレクトリ基盤を保持するエンタープライズ環境での 802.1X / TACACS+ 統合認証。
  2. Active Directory ドメインに ISE ノードを直接参加（Join）させない/参加できないセキュリティポリシー配下でのユーザー認証。
  3. EAP-TLS 認証において、証明書内の SAN (Subject Alternative Name) や UPN (User Principal Name) をキーとして LDAP ディレクトリから属性（グループ情報、役職、SGT タグ等）を動的に検索・付与する構成。
  4. 複数の独立した LDAP ディレクトリを Identity Source Sequences (ISS) に組み込み、順次検索を行うマルチディレクトリ統合認証。

---

## 🔑 要点

Cisco ISE と LDAP サーバー連携に関する主要な技術要件、制限事項、および設計上の考慮点を下表に示します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | 標準 LDAPv3 (RFC 4511) に対応。TCP 389 (Plain LDAP/StartTLS) および TCP 636 (LDAPS over SSL) をサポート。パスワード認証（Bind / Compare）と属性取得（Search）を同時または段階的に実行。 |
| **用途** | 802.1X ネットワークアクセス認証、MAB 補助検証、ゲスト/WebAuth 認証、TACACS+ 管理者アクセス認可、VPN ユーザー認証、認可ポリシー用属性（Group/Attribute）抽出。 |
| **メリット** | AD ドメイン参加（Join）が不要。OS やディレクトリ製品を問わないマルチベンダー対応。スキーマ属性マッピングが自由自在（Custom Schema 対応）。証明書認証 (EAP-TLS) との組み合わせで高いセキュリティを実現。 |
| **デメリット** | NTLM / MSCHAPv2 認証が直接サポートされない（パスワードハッシュの直接検証不可）。そのため PEAP-MSCHAPv2 は利用不可（PEAP-GTC, EAP-TTLS-PAP, EAP-TLS の使用が必須）。AD Native Join に比べ機能制限あり。 |
| **対応機種** | Cisco ISE 全モデル (3.x / 2.x)、全 Deployment Mode (Standalone, Distributed PAN/MnT/PSN)。LDAP サーバー側は OpenLDAP, Active Directory, FreeIPA, Oracle Directory Server, Sun ONE 等に対応。 |
| **制限事項** | 1. MSCHAPv2 / LEAP 認証不可（平文パスワード検証が必要なプロトコル、または EAP-TLS / GTC のみ対応）。<br>2. パスワード変更機能（Change Password）は一部ポータル/プロトコル制限あり。<br>3. 大規模環境では検索レスポンスタイムの遅延（Latency）が ISE の認証タイムアウトを引き起こすリスクあり。 |
| **設計上の注意点** | 1. 通信暗号化（TCP 636 LDAPS）のため、LDAP サーバーの Root/Intermediate CA 証明書を ISE Trusted Certificates に事前登録が必須。<br>2. 検索ベース DN (Search Base DN) および Admin Bind DN の最適化とインデックス作成。<br>3. Identity Source Sequence (ISS) でのフェールオーバー条件（User Not Found vs Process Error）の設定。 |

---

## 🏗 動作原理

Cisco ISE における LDAP 統合動作は、大きく分けて「バインド認証 (Admin Bind / User Bind)」、「属性検索 (Attribute Query)」、「認可判定 (Authorization)」の 3 段階で構成されます。

### 通信フロー（ネットワークアクセス認証：EAP-TLS + LDAP 属性取得）

```
Supplicant (Client)
   │
   │ 1. EAP-TLS Authentication (Client Cert)
   ▼
Authenticator (Switch / WLC)
   │
   │ 2. RADIUS Access-Request (EAP-Response)
   ▼
Cisco ISE (PSN)
   │
   │ 3. Validate Certificate (CA Trust & Revocation Check)
   │ 4. Extract Identity (e.g., SAN:email or UPN)
   │
   │ 5. LDAPS Bind Request (Admin DN + Secret) [TCP 636]
   ├──────────────────────────────────────────────────────►
   │ 6. LDAPS Bind Success                                LDAP Server
   │◄──────────────────────────────────────────────────────┤
   │ 7. Search Request (BaseDN, Filter: mail=user@domain)  │
   ├──────────────────────────────────────────────────────►
   │ 8. Search Result (Attributes: memberOf, employeeType)│
   │◄──────────────────────────────────────────────────────┤
   │
   │ 9. Evaluate Policy Set (Rule matching by memberOf/SGT)
   │
   ▼
Authenticator (Switch / WLC)
   │
   │ 10. RADIUS Access-Accept (dACL / Dynamic VLAN / SGT)
   ▼
Supplicant (Client) ── Access Granted
```

### パケット/処理メカニズムの分類

1. **Admin Bind（管理者バインド）**:
   * ISE は設定された Admin DN とパスワードを使用して LDAP サーバーに事前接続（バインド）します。
   * 以降のユーザー検索や属性照会はこの管理者権限のセッション上で行われます。
2. **User Search & Attribute Extraction（ユーザー検索と属性取得）**:
   * サンプリカントから送信された ID（例: `username` または証明書の SAN/Subject）をキーとして、設定された Search Base DN 配下を検索フィルター（例: `(&(objectClass=user)(sAMAccountName={0}))`）で照会します。
   * 該当するユーザーオブジェクトから `memberOf`、`mail`、`department` などの属性（Attributes）を抽出します。
3. **User Authentication (Direct Bind / Password Compare)**:
   * **Direct Bind Mode**: ユーザーが入力したパスワードを用いて、ISE がそのユーザーの DN（Distinguished Name）で直接 LDAP サーバーに Bind を試みます。認証成功でパスワード一致と判断。
   * **Password Compare Mode**: ユーザーのパスワードハッシュを LDAP から取得し、ISE 側で比較（一般に Cleartext または一部の非可逆ハッシュで制限あり）。通常は Direct Bind が使用されます。

---

## ⚙ 動作シーケンス

Cisco ISE が LDAP 外部アイデンティティソースを参照してアクセス制御を実施する際の段階的なシーケンスは以下の通りです。

```
[Phase 1: 認証リクエスト受信]
  1. サンプリカントが 802.1X / WebAuth / TACACS+ 接続を開始。
  2. Authenticator (NAS) が RADIUS / TACACS+ リクエストを ISE (PSN) に転送。

[Phase 2: アイデンティティ抽出と外部ソース選定]
  3. ISE の Policy Set (Authentication Policy) にて、対象の Identity Source または Identity Source Sequence (ISS) が評価される。
  4. 選択された LDAP アイデンティティソースの設定に基づき、抽出キー（例: Username または Certificate Attribute）が決定される。

[Phase 3: LDAP サーバーとのセッション確立・バインド]
  5. ISE は接続プールから既存の LDAP セッションを取得するか、新規に TCP 636 (LDAPS) / TCP 389 (StartTLS) 接続を確立。
  6. LDAPS の場合、TLS ハンドシェイクが実行され、サーバー証明書の検証（Trusted Certificate Store 照合）が行われる。
  7. ISE は Admin Bind DN および Password を使って BindRequest を送信。
  8. LDAP サーバーから BindResponse (Success) を受信。

[Phase 4: ユーザー検索およびパスワード検証/属性取得]
  9. ISE は設定された Search Base および Filter 条件を用いて SearchRequest を送信。
 10. LDAP サーバーは一致するユーザーオブジェクト（DN, 属性リスト）を SearchResultEntry として返却。
 11. (パスワード認証の場合) ISE は取得した ユーザー DN とユーザー入力パスワードで User BindRequest を送信し、合致を確認。
 12. (証明書認証 EAP-TLS の場合) ユーザー Bind はスキップし、取得したグループ情報 (memberOf 等) を ISE 内部セッションキャッシュに保存。

[Phase 5: 認可ポリシー評価とレスポンス送信]
 13. Policy Set (Authorization Policy) にて、LDAP から取得した属性条件（例: LDAP:memberOf EQUALS cn=Domain Admins,ou=Groups,dc=example,dc=com）を評価。
 14. 合致した Authorization Profile に基づき、RADIUS Access-Accept (dACL / Dynamic VLAN / SGT) または TACACS+ PASS を NAS に返却。
```

---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、Cisco ISE と LDAP の統合は「Active Directory ネイティブ参加が許可されていないシナリオ」や「特定Linuxユーザー基盤との802.1X/TACACS+連携シナリオ」で極めて頻繁に出題されます。試験で要求される実装精度と狙われやすいポイントを整理します。

### 1. Blueprint における最重要ポイント
* **認証プロトコルの制限と選択**:
  * LDAP は NTLM チャレンジ/レスポンスに必要なパスワードハッシュ構造を直接提供しないため、**PEAP-MSCHAPv2 は使用不可**。
  * 試験で「PEAP を使用しつつ外部 OpenLDAP サーバーで認証せよ」と指定された場合、**PEAP-GTC** (Generic Token Card) または **EAP-TTLS (with PAP)** を使用する設定が必要。
  * 端末証明書を用いた **EAP-TLS** 認証において、証明書の SAN/UPN からユーザーを特定し、LDAP から `memberOf` 属性を取得して Authorization Policy に適用する構成。
* **LDAP Over SSL (LDAPS - TCP 636) と CA 証明書バインド**:
  * 実務・試験ともに暗号化（LDAPS）が強く推奨/強制される。
  * ISE の `Administration > System > Certificates > Trusted Certificates` に LDAP サーバーの発行元 CA 証明書がインポートされ、`Trust for authentication within ISE` および `Trust for client authentication and Syslog` にチェックが入っていないと LDAPS バインドが失敗する。
* **カスタムスキーマ・属性マッピング (Schema & Attribute Mapping)**:
  * OpenLDAP や Sun Directory では、Active Directory の `sAMAccountName` ではなく `uid` や `cn` がユーザー識別子として使用される。
  * 属性名（例: `memberOf` vs `isMemberOf` vs `groupMembership`）の正確な設定と、ISE 上でのデータタイプ（String, Enum, Int）のミスマッチ回避。

### 2. よくある設定ミスと試験での落とし穴
* **検索ベース DN (Search Base DN) の指定ミス**:
  * Search Base DN を曖昧（例: `dc=example,dc=com`）にしすぎると、オブジェクト数が多い環境で ISE の LDAP クエリがタイムアウト（デフォルト 2〜5 秒）を起こし、認証拒否（Process Error）となる。正確な OU（例: `ou=Users,dc=example,dc=com`）を指定するか、検索スコープを調整すること。
* **Admin Bind Credentials の権限不足/ロックアウト**:
  * ISE が最初に LDAP に接続する Admin Bind DN (例: `cn=iseadmin,ou=ServiceAccounts,dc=example,dc=com`) のパスワード誤りやアカウントロックアウト。
* **PEAP-MSCHAPv2 を選択してしまう設定ミス**:
  * 外部 ID ソースに LDAP を指定した Policy Set で PEAP-MSCHAPv2 を許可したままにすると、クライアント接続時に `11506 MSCHAPv2 password change is not supported` または `24401 LDAP authentication failed` エラーが発生する。Allowed Protocols で `Allow EAP-GTC` や `Allow EAP-TLS` を明示的に許可すること。
* **Identity Source Sequence (ISS) での認証失敗時動作未設定**:
  * AD -> LDAP の順で検索する ISS を構築した際、`User Not Found` 時に次のソースに移行する設定（`Treat as User Not Found and continue`）が入っていないと、最初のソースで拒否されてセッションが終了する。

---

## 🛠 設定方法

Cisco ISE GUI における LDAP 外部アイデンティティソースの設定手順および手順例を示します。

### GUI 設定手順（Cisco ISE 3.x）

#### Step 1: Trusted Certificate のインポート (LDAPS 利用時)
1. GUI ナビゲーション: `Administration > System > Certificates > Trusted Certificates` に移動。
2. `Import` をクリックし、LDAP サーバーの Root/Intermediate CA 証明書（PEM/DER 形式）を選択。
3. チェックボックス: `Trust for authentication within ISE` および `Trust for client authentication and Syslog` を有効化して `Save`。

#### Step 2: LDAP External Identity Source の作成
1. GUI ナビゲーション: `Administration > Identity Management > External Identity Sources > LDAP` に移動。
2. `Add` をクリックし、基本情報を入力。
   * **Name**: `OpenLDAP_Server`
   * **Description**: `Corporate OpenLDAP Directory`

#### Step 3: General 属性と Connection 設定
1. **General Tab**:
   * **Subject Name Attribute**: `cn` または `uid` (環境に応じて選択)
   * **Time Out (seconds)**: `5`
   * **Max Connections**: `50`
2. **Connection Tab**:
   * **Primary Server**: LDAP サーバーの IP アドレスまたは FQDN (例: `192.168.10.50`)
   * **Port**: `636` (LDAPS の場合) または `389` (Plain LDAP/StartTLS)
   * **Server Encryption Method**: `LDAPS` (SSL/TLS) を選択
   * **Admin DN**: `cn=admin,dc=example,dc=com`
   * **Password**: 管理者パスワードを入力
   * `Test Connection` をクリックして、ステータスが OK（緑色）になることを確認。

#### Step 4: Directory Organization (Search Base & Filter) 設定
1. **Directory Organization Tab**:
   * **Search Base**: `dc=example,dc=com` (ユーザーおよびグループが存在する最上位ノード)
   * **All User Organizations**: ユーザー検索範囲の OU (例: `ou=People,dc=example,dc=com`)
   * **User Object Class**: `inetOrgPerson` または `posixAccount` または `user`
   * **User Search Filter**: `(&(objectClass=inetOrgPerson)(uid={0}))`

#### Step 5: Groups および Attributes の追加
1. **Groups Tab**:
   * `Add` をクリックし、検索条件に基づいてグループを取得（例: `cn=NetworkAdmins,ou=Groups,dc=example,dc=com`）。
   * **Group Membership Attribute**: `memberOf` または `groupMembership`
2. **Attributes Tab**:
   * 必要に応じて動的認可に使用する属性（例: `department`, `employeeType`, `telephoneNumber`）を追加。

#### Step 6: Allowed Protocols および Policy Sets への適用
1. `Policy > Policy Elements > Results > Authentication > Allowed Protocols` に移動。
2. 新規プロトコルセットを作成（例: `Default_LDAP_Protocols`）。
3. `Allow EAP-TLS`, `Allow EAP-GTC`, `Allow EAP-TTLS` (PAP) を有効化（`Allow PEAP` の下の `MSCHAPv2` はオフにする）。
4. `Policy > Policy Sets` に移動し、新規 Policy Set を作成。
   * **Authentication Policy**: Identity Source として `OpenLDAP_Server` または該当する `ISS` を指定。
   * **Authorization Policy**: 条件文に `OpenLDAP_Server:ExternalGroups EQUALS cn=NetworkAdmins,ou=Groups,dc=example,dc=com` を設定し、`PermitAccess` + `Network_Admin_dACL` を割り当て。

---

## 🔍 検証コマンド

Cisco ISE CLI、ISE GUI Diagnostics、および外部ツールの検証コマンド一覧です。

| 目的 | コマンド / 操作画面 | 詳細解説 |
| --- | --- | --- |
| **ISE ネットワーク接続確認** | <code>ping 192.168.10.50</code> | PSN ノードから LDAP サーバーへの IP 到達性確認。 |
| **LDAP ポート到達性確認** | <code>tech-support</code> または <code>show network-status</code> | TCP 636 / 389 ポートの通信状態確認。 |
| **ISE ライブログリアルタイム確認** | `Operations > RADIUS > Live Logs` | 認証リクエストの成否、使用された Policy Set、参照された Identity Source、取得された External Group 名を即座に確認。 |
| **LDAP 接続テスト (GUI)** | `Administration > Identity Management > External Identity Sources > LDAP > [Server] > Connection` | `Test Connection` ボタンを押下。Admin Bind、TLS ハンドシェイク、Base DN 検索の成否を診断。 |
| **ISE 内部認証ログ詳細デバッグ** | <code>debug logging component runtime level debug</code> | PSN ノードの CLI にて、`prt-runtime.log` に詳細な LDAP 接続および検索ログを出力。 |
| **LDAP ログ抽出** | <code>show logging application prt-runtime.log tail 100</code> | 直近の LDAP バインド・レスポンスメッセージを CLI 上で確認。 |

---

## 🚨 トラブルシュート

Cisco ISE と LDAP 統合環境で遭遇する典型的な障害症状、根本原因、確認コマンド/画面、および対処方法を下表に整理します。

| 症状 | 根本原因 | 確認コマンド / 画面 | 対処方法 |
| --- | --- | --- | --- |
| **`Test Connection` が Failed となる (LDAPS 636)** | ISE が LDAP サーバーの SSL/TLS 証明書を発行した CA を信頼していない。 | Operations > RADIUS > Live Logs<br>`prt-runtime.log` ("PKIX path building failed") | `Administration > Trusted Certificates` に LDAP CA 証明書をインポートし、Trust アプリケーションを有効化する。 |
| **`Test Connection` が Port Unreachable となる** | ISE と LDAP サーバー間のファイアウォールで TCP 389/636 が遮断されている。 | ISE CLI: <code>ping 192.168.10.50</code><br>ネットワークスイッチの ACL | ファイアウォールで PSN IP から LDAP サーバー IP への TCP 636 (LDAPS) / 389 を許可する。 |
| **`Admin Bind Failed` エラーが発生する** | Admin Bind DN の記法ミスまたはパスワード不一致。 | ISE GUI Test Connection ログ<br>LDAP サーバー syslog (`slapd.log`) | Admin DN のフルパス (例: `cn=admin,dc=example,dc=com`) とパスワードを正確に再設定する。 |
| **PEAP 認証が失敗する (`24401 LDAP authentication failed`)** | クライアントが PEAP-MSCHAPv2 を要求しているが、LDAP は MSCHAPv2 を非サポート。 | ISE RADIUS Live Logs: Failure Code `11506` または `24401` | Supplicant および Allowed Protocols を PEAP-GTC または EAP-TTLS (PAP) / EAP-TLS に変更する。 |
| **ユーザー認証成功だが認可ルールがマッチしない** | LDAP グループ名 (DN) の不一致、または Group Membership Attribute の誤り。 | ISE RADIUS Live Logs > Detailed Report ("Other Attributes" 内のグループ確認) | ISE の `External Identity Sources > LDAP > Groups` で取得された DN と Authorization Policy の条件文を完全に一致させる。 |
| **証明書認証 (EAP-TLS) で User Not Found となる** | Certificate Authentication Profile (CAP) で抽出された Identity 属性が LDAP 内の検索キーと一致しない。 | Policy > Identity Source Sequences<br>RADIUS Live Logs | CAP の Principal Identity Attribute (例: SAN Email) と LDAP 検索属性 (例: `mail`) をマッピングする。 |
| **LDAP タイムアウトによる認証遅延/ドロップ** | LDAP サーバーの検索インデックス不足、または Search Base が広大すぎる。 | `prt-runtime.log` ("Search operation timed out") | Search Base DN を特定の OU に絞り込み、LDAP サーバー側で `uid` や `sAMAccountName` に Index を作成する。 |
| **複数の LDAP サーバー構成で片方障害時に認証失敗** | Identity Source Sequence (ISS) でフェールオーバー条件が適切に設定されていない。 | Policy > Identity Source Sequences | ISS 設定で `On Process Error` -> `Attempt Next Authority` に設定変更する。 |

---

## ⚠ 制限事項

Cisco ISE の LDAP 統合機能に関するサポートされない構成、バージョン依存性、およびライセンス依存性をまとめます。

### 1. サポートされない構成・プロトコル制限
* **MSCHAPv2 直接検証の不可能性**: LDAP プロトコルおよび一般的な LDAP データベース（OpenLDAP 等）はパスワードハッシュ（NTLM hash）を検証するインターフェースを提供しないため、**EAP-MSCHAPv2、PEAP-MSCHAPv2、LEAP 認証は使用不可**。
* **Password Change Portal の制約**: ISE のセルフサービスポータル経由での LDAP ユーザーパスワード変更は、LDAP サーバー側の特定パスワード拡張（例: Password Modify Extended Operation - RFC 3062）や LDAPS 接続が必須であり、標準設定ではサポートされない場合がある。
* **Kerberos 認証の未サポート**: Active Directory Native Join で利用できる Kerberos 認証チケットの発行や利用は不可（LDAP 単体通信のみ）。

### 2. バージョン依存性
* **Cisco ISE 3.x / 2.x**: LDAP 属性を用いた動的 SGT 割当や Security Group Tag 属性マッピング機能は ISE 2.6 以降で大幅強化。
* **TLS バージョン**: LDAPS 通信時、ISE 3.1 以降ではデフォルトで TLS 1.2 / TLS 1.3 が強制され、古い SSLv3 / TLS 1.0 の暗号スイートしか持たないレガシー LDAP サーバーとは接続不能となる。

### 3. ライセンス依存性
* **ISE Base / Premier / Advantage License**:
  * 802.1X / MAB / TACACS+ における基本 LDAP 認証・グループ参照: **Essentials / Base License** で利用可能。
  * LDAP 属性に基づく Dynamic SGT (TrustSec) 割り当てや Posture / BYOD 連携: **Advantage / Premier License** が必要。

---

## 🔄 他技術との関連

Cisco ISE の LDAP 統合は、他のセキュリティ技術およびネットワークインフラと密接に連携します。

* **Cisco TrustSec (SGT)**:
  * LDAP から取得した `department` や `memberOf` 属性条件に基づき、認可プロファイル（Authorization Profile）経由で動的に Security Group Tag (SGT: 例 SGT 10 = Sec_Engineers) を端末に付与。
* **Identity Source Sequences (ISS)**:
  * Active Directory、Internal Users、LDAP を統合する優先順位付きリスト（ISS）を構成。AD で検索失敗したユーザーを OpenLDAP で二次検索する等のハイブリッド構成を実現。
* **TACACS+ Device Administration (Cisco ISE TACACS+)**:
  * ネットワーク機器（Catalyst, ASA, FTD）の CLI 管理者アクセス認証時、ISE が LDAP を参照して Privilege Level 15 や Command Sets（許可コマンドリスト）を動的に決定。
* **WebAuth / CWA (Central Web Authentication)**:
  * ゲストポータルや MyDevices ポータルにおいて、内部ユーザー DB ではなく外部 LDAP ディレクトリに登録された一般社員・パートナーアカウントでログインを許可する構成。
* **AnyConnect / FlexVPN (Remote Access VPN)**:
  * Cisco ASA / FTD / CSR1000v VPN Gateway が RADIUS/SAML 経由で ISE に問い合わせた際、ISE がバックエンドの LDAP サーバーを検索して VPN グループポリシーを決定。

---

## 🧩 比較表

Cisco ISE における外部アイデンティティソース連携方式および LDAP 接続モードの比較表です。

### 比較表 1: ISE 外部アイデンティティソース方式の比較

| 項目 | LDAP Integration | Active Directory Native Join | RADIUS External Source |
| --- | --- | --- | --- |
| **対応プロトコル** | EAP-TLS, PEAP-GTC, EAP-TTLS-PAP | EAP-TLS, PEAP-MSCHAPv2, EAP-FAST | EAP-TTLS, PAP, Proxy RADIUS |
| **AD ドメイン参加** | **不要** (IP/Port 接続のみ) | **必須** (ISE を AD に Join) | **不要** (Proxy 送信) |
| **マルチベンダー対応** | **極めて高い** (OpenLDAP, FreeIPA 等) | Windows Active Directory 限定 | 高い (サードパーティ RADIUS) |
| **MSCHAPv2 サポート** | ✕ 非対応 | ◯ 完全対応 | △ 送信先 RADIUS に依存 |
| **属性ベース認可** | ◯ 可能 (`memberOf`, 任意属性) | ◯ 可能 (AD Groups, SID) | △ RADIUS VSA に依存 |
| **構築難易度** | 中程度 (スキーマ定義が必要) | 低〜中 (GUI Wizard で完結) | 中程度 |

### 🧩 比較表 2: LDAP 接続セキュリティモードの比較

| モード | ポート番号 | 暗号化有無 | 証明書検証 (CA Trust) | 特徴・推奨用途 |
| --- | --- | --- | --- | --- |
| **Plain LDAP** | TCP 389 | ✕ なし (平文) | 不要 | 非推奨。検証環境のみ。パスワードが平文露出。 |
| **LDAP StartTLS** | TCP 389 | ◯ あり (明示的 TLS 昇格) | 必須 | 389 ポート上で TLS ハンドシェイクを開始。標準的。 |
| **LDAPS over SSL** | TCP 636 | ◯ あり (暗号化トンネル) | **必須** (ISE Trusted CA) | **最推奨**。接続確立時から TLS 暗号化が強制される。 |

---

## 💡 ベストプラクティス

Cisco エンタープライズネットワーク設計における LDAP 統合の推奨事項です。

1. **暗号化通信 (LDAPS - TCP 636) の徹底**:
   * セキュリティおよびコンプライアンス要件（PCI-DSS, ISO 27001）を満たすため、必ず LDAPS (Port 636) を採用し、ISE に適切な Enterprise Root CA 証明書を登録する。
2. **Search Base DN の最適化とインデックス作成**:
   * 検索ルートをディレクトリ全体のルート (`dc=example,dc=com`) に設定せず、対象ユーザー・グループが格納されている最小単位の OU (`ou=Users,ou=Japan,dc=example,dc=com`) に限定する。
   * LDAP サーバー側で検索キー（`uid`, `sAMAccountName`, `mail`, `memberOf`）に対して属性インデックス（Attribute Indexing）を作成し、レスポンス時間を 10ms 以下に抑える。
3. **冗長化とロードバランシングの設計**:
   * ISE の LDAP 設定画面で Primary LDAP Server に加え、Secondary LDAP Server を必ず登録する。
   * PSN ノードごとに最寄りの LDAP レプリカサーバーを参照するよう地域分散設計を行う。
4. **Identity Source Sequences (ISS) の例外処理明示**:
   * ISS 内で LDAP を使用する場合、`User Not Found` 時は即時拒否（Reject）せず次のソースへ継続（`Continue`）、`Process Error` (タイムアウト) 時は代替サーバーへ再試行（`Attempt Next Authority`）するようポリシーを明記する。
5. **PEAP-GTC クライアントプロファイルの配信**:
   * 端末側（Windows / macOS / AnyConnect NAM）に対し、PEAP-GTC または EAP-TLS の認証プロファイルを GPO や MDM 経由で事前配布し、ユーザーの誤設定を防ぐ。

---

## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルの実践的な設定サンプルおよび検証手順を 10 個作成しました。コンフィグおよび設定手順は省略せず最後まで記載しています。

---

### 演習例 1: LDAPS (TCP 636) を用いた OpenLDAP 接続と CA 証明書信頼構築

#### 問題
スイッチ Sw-1 配下の端末認証のため、Cisco ISE と外部 OpenLDAP サーバー (`192.168.10.50`) を TCP 636 (LDAPS) で接続し、管理者バインドおよび接続テストを正常に完了させよ。

#### 要件
1. LDAP サーバーの Root CA 証明書を ISE Trusted Certificates にインポートし、認証目的で信頼すること。
2. Admin DN: `cn=admin,dc=example,dc=com` / Password: `Cisco123Secret`
3. ポート `636` (LDAPS) を使用し、タイムアウトは `5` 秒とすること。

#### 設定例

**1. CA 証明書のインポート (ISE GUI)**:
* `Administration > System > Certificates > Trusted Certificates` -> `Import`
* File: `OpenLDAP_RootCA.pem`
* Friendly Name: `OpenLDAP_Root_CA`
* Checkbox:
  * [x] `Trust for authentication within ISE`
  * [x] `Trust for client authentication and Syslog`
* `Submit` をクリック。

**2. LDAP External Identity Source 設定 (ISE GUI)**:
* `Administration > Identity Management > External Identity Sources > LDAP` -> `Add`
* **Name**: `OpenLDAP_LDAPS`
* **General Tab**:
  * Subject Name Attribute: `uid`
  * Time Out: `5`
  * Max Connections: `50`
* **Connection Tab**:
  * Primary Server Hostname/IP: `192.168.10.50`
  * Port: `636`
  * Server Encryption Method: `LDAPS`
  * Admin DN: `cn=admin,dc=example,dc=com`
  * Password: `Cisco123Secret`
* `Save` をクリック後、`Test Connection` を押下。

#### 検証方法
1. ISE GUI にて `Test Connection` の結果が「Admin Bind Success」および「TLS Handshake Successful」となることを確認。
2. ISE CLI にてログを確認:
```bash
show logging application prt-runtime.log tail 50
```
ログ内に `LDAP bind successful for cn=admin,dc=example,dc=com` が出力されていることを検証。

---

### 演習例 2: PEAP-GTC を用いた OpenLDAP ユーザーの 802.1X 認証と Dynamic VLAN 割当

#### 問題
Sw-1 GigabitEthernet1/0/1 に接続された PC からの 802.1X 認証において、外部 OpenLDAP ユーザーによる PEAP-GTC 認証を構成し、認証成功時に VLAN 20 を動的割り当てよ。

#### 要件
1. 外部 ID ソースとして `OpenLDAP_LDAPS` を使用。
2. PEAP-MSCHAPv2 は非対応のため、Allowed Protocols で `PEAP-GTC` を許可すること。
3. 認証成功時、RADIUS レスポンスで VLAN ID 20 (`Tunnel-Private-Group-ID = 20`) を返却すること。

#### 設定例

**1. Allowed Protocols の作成 (ISE GUI)**:
* `Policy > Policy Elements > Results > Authentication > Allowed Protocols` -> `Add`
* Name: `PEAP_GTC_Only`
* [x] `Allow PEAP`
  * [x] `Allow EAP-GTC` (PEAP 内)
  * [ ] `Allow MSCHAPv2` (チェックを外す)
* `Save`

**2. Authorization Profile の作成 (ISE GUI)**:
* `Policy > Policy Elements > Results > Authorization > Authorization Profiles` -> `Add`
* Name: `VLAN20_Profile`
* Common Tasks:
  * VLAN: ID/Name = `20`
* `Save`

**3. Policy Set の構築 (ISE GUI)**:
* `Policy > Policy Sets` -> 新規作成 `OpenLDAP_8021X`
* Condition: `RADIUS:NAS-Port-Type EQUALS Ethernet`
* Allowed Protocols: `PEAP_GTC_Only`
* **Authentication Policy**:
  * Rule 1: `Use OpenLDAP_LDAPS`
* **Authorization Policy**:
  * Rule 1: Condition: `OpenLDAP_LDAPS:Authentication Status EQUALS AuthenticationPassed` -> Action: `VLAN20_Profile`

**4. スイッチ側 C3PL コンフィグ (Sw-1 CLI)**:
```text
aaa new-model
aaa authentication dot1x default group radius
aaa authorization network default group radius

interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
 authentication port-control auto
 dot1x pae authenticator
 access-session host-mode single-host
 access-session port-control auto
```

#### 検証方法
1. クライアントから PEAP-GTC でユーザー `john` / パスワードを入力して接続。
2. Sw-1 で認証状態を確認:
```bash
show access-session interface GigabitEthernet1/0/1 details
```
`Status: Authorized` かつ `VLAN: 20` が適用されていることを確認。
3. ISE RADIUS Live Logs で `Authentication Method: PEAP (EAP-GTC)` および `Identity Source: OpenLDAP_LDAPS` と表示されることを確認。

---

### 演習例 3: EAP-TLS 端末証明書認証 + LDAP SAN/UPN 属性抽出および dACL 動的付与

#### 問題
EAP-TLS 証明書認証を行うクライアントに対し、証明書内の SAN (Subject Alternative Name) から Email アドレスを抽出し、LDAP から属性を取得して dACL `PERMIT_ENG_DACL` を付与せよ。

#### 要件
1. EAP-TLS 証明書検証を行い、SAN:Email からユーザー識別子を取得。
2. LDAP から抽出した `department` 属性が `Engineering` の場合、dACL `PERMIT_ENG_DACL` を適用。

#### 設定例

**1. Certificate Authentication Profile (CAP) の作成 (ISE GUI)**:
* `Administration > Identity Management > Certificate Authentication Profile` -> `Add`
* Name: `CAP_SAN_Email`
* Principal Identity Attribute: `Subject Alternative Name - Email Address`
* `Save`

**2. LDAP Attribute Mapping の設定 (ISE GUI)**:
* `Administration > Identity Management > External Identity Sources > LDAP > OpenLDAP_LDAPS`
* **Attributes Tab**:
  * Attribute Name: `department` -> Schema Attribute: `department` -> DataType: `STRING`
* `Save`

**3. dACL の作成 (ISE GUI)**:
* `Policy > Policy Elements > Results > Authorization > Downloadable ACLs` -> `Add`
* Name: `PERMIT_ENG_DACL`
* DACL Content:
```text
permit ip any host 10.100.1.50
permit ip any host 10.100.1.51
deny ip any 10.0.0.0 255.0.0.0
permit ip any any
```

**4. Authorization Profile & Policy Set 設定**:
* Policy Set Authentication Policy:
  * Identity Source: `OpenLDAP_LDAPS` (Identity Certificate Profile: `CAP_SAN_Email`)
* Authorization Policy:
  * Condition: `OpenLDAP_LDAPS:department EQUALS Engineering`
  * Action: `PERMIT_ENG_Profile` (DACL: `PERMIT_ENG_DACL`)

#### 検証方法
1. Sw-1 で dACL 適用確認:
```bash
show ip access-lists dynamic
show access-session interface GigabitEthernet1/0/1 details
```
2. ISE RADIUS Live Logs で `Authentication Method: EAP-TLS` かつ `Matched Rule` が正しく指定されたルールであることを検証。

---

### 演習例 4: LDAP memberOf グループ属性に基づく TACACS+ CLI 管理アクセス認可

#### 問題
ルータ R-1 への SSH 管理アクセスにおいて、Cisco ISE を TACACS+ サーバーとし、OpenLDAP の `memberOf` 属性に `cn=NetAdmins` が含まれるユーザーには Privilege Level 15 を与えよ。

#### 要件
1. ISE TACACS+ Device Admin 機能を使用。
2. OpenLDAP の `memberOf` 属性から `cn=NetAdmins,ou=Groups,dc=example,dc=com` を取得。
3. 一致する場合、Shell Profile で `Privilege 15` を返却。

#### 設定例

**1. ISE Shell Profile の作成 (ISE GUI)**:
* `Work Centers > Device Admin > Policy Elements > Results > TACACS Profiles` -> `Add`
* Name: `Piv15_Shell_Profile`
* Default Privilege: `15`
* Maximum Privilege: `15`

**2. ISE TACACS+ Policy Set の作成 (ISE GUI)**:
* `Work Centers > Device Admin > Device Admin Policy Sets` -> 新規作成 `Network_Device_Admin`
* Condition: `DEVICE:Device Type EQUALS Network_Routers`
* **Authentication Policy**:
  * Identity Source: `OpenLDAP_LDAPS`
* **Authorization Policy**:
  * Rule 1: Condition: `OpenLDAP_LDAPS:ExternalGroups EQUALS cn=NetAdmins,ou=Groups,dc=example,dc=com`
  * Action: Shell Profile = `Piv15_Shell_Profile`

**3. ルータ R-1 TACACS+ コンフィグ (R-1 CLI)**:
```text
tacacs server ISE_TACACS
 address ipv4 192.168.10.21
 key Cisco123Secret

aaa group server tacacs+ ISE_TAC_GROUP
 server name ISE_TACACS

aaa authentication login default group ISE_TAC_GROUP local
aaa authorization exec default group ISE_TAC_GROUP local
```

#### 検証方法
1. SSH で R-1 にユーザー `adminuser` でログイン。
2. R-1 でプロンプトが `R-1#` (Privilege 15) に直接昇格することを確認。
3. ISE GUI `Operations > TACACS > Live Logs` で Privilege Level 15 の付与ログを確認。

---

### 演習例 5: Active Directory 優先、OpenLDAP フォールバックの Identity Source Sequence (ISS) 構築

#### 問題
802.1X 認証において、最初に Active Directory (`AD_Domain`) を参照し、ユーザーが存在しない場合 (`User Not Found`) は OpenLDAP (`OpenLDAP_LDAPS`) を参照する ISS を構築せよ。

#### 要件
1. Identity Source Sequence 名: `AD_then_OpenLDAP_ISS`
2. `AD_Domain` 検索時に `User Not Found` が返された場合、処理を中断せず `OpenLDAP_LDAPS` へ継続すること。

#### 設定例

**1. ISS の作成 (ISE GUI)**:
* `Administration > Identity Management > Identity Source Sequences` -> `Add`
* Name: `AD_then_OpenLDAP_ISS`
* **Authentication Search List**:
  * Selected List:
    1. `AD_Domain`
    2. `OpenLDAP_LDAPS`
* **Advanced Search Option (Do not access next storage in sequence...)**:
  * If User Not Found: [ ] (チェックを外し、`Treat as User Not Found and continue to next source` を選択)
  * If Process Error: [x] `Attempt next authority`
* `Save`

**2. Policy Set への適用**:
* Policy Set の Authentication Policy で Identity Source として `AD_then_OpenLDAP_ISS` を割り当て。

#### 検証方法
1. OpenLDAP にのみ存在するユーザー `ldapuser1` で 802.1X 接続を開始。
2. RADIUS Live Logs で、最初 `AD_Domain` で `User Not Found` となり、次に `OpenLDAP_LDAPS` で認証成功（AuthenticationPassed）に移行したことをログ詳細（Steps）で確認。

---

### 演習例 6: Custom LDAP Schema (uid / posixAccount) マッピングと認可制御

#### 問題
Linux 環境固有の OpenLDAP スキーマ (`posixAccount` / `gidNumber`) を使用する環境で、`gidNumber` が `5001` (Engineers) のユーザーに 特定の dACL を割り当てる認可ポリシーを構築せよ。

#### 要件
1. LDAP 属性 `gidNumber` を ISE 内部属性としてタイプ `INTEGER` でマッピング。
2. `gidNumber EQUALS 5001` の場合、dACL `DACL_ENG_ONLY` を適用。

#### 設定例

**1. LDAP 属性マッピング (ISE GUI)**:
* `Administration > Identity Management > External Identity Sources > LDAP > OpenLDAP_LDAPS`
* **Attributes Tab**:
  * Name: `gidNumber` / Schema Attribute: `gidNumber` / Type: `INTEGER`
* `Save`

**2. Authorization Policy 設定 (ISE GUI)**:
* `Policy > Policy Sets > [Default Policy Set]`
* Authorization Rule:
  * Name: `Eng_Gid_Rule`
  * Condition: `OpenLDAP_LDAPS:gidNumber EQUALS 5001`
  * Action: `Profile_DACL_ENG_ONLY`

#### 検証方法
1. 対象ユーザーでログイン後、ISE RADIUS Live Logs の `Other Attributes` 内で `gidNumber = 5001` と認識されていることを確認。
2. スイッチ側で動的 ACL が期待通り適用されていることを検証。

---

### 演習例 7: Primary / Secondary LDAP Server 冗長化とタイムアウトチューニング

#### 問題
LDAP 認証の可用性を高めるため、Primary LDAP (`192.168.10.50`) および Secondary LDAP (`192.168.10.51`) を登録し、1 次サーバー応答停止時に自動切り替えを行わせよ。

#### 要件
1. Primary: `192.168.10.50` (Port 636) / Secondary: `192.168.10.51` (Port 636)
2. タイムアウト: `3` 秒 / 再試行回数: `2`

#### 設定例

**ISE GUI 設定**:
* `Administration > Identity Management > External Identity Sources > LDAP > OpenLDAP_LDAPS`
* **General Tab**:
  * Time Out: `3`
  * Max Connections: `50`
* **Connection Tab**:
  * Primary Server: `192.168.10.50` / Port: `636` / LDAPS
  * Secondary Server: `192.168.10.51` / Port: `636` / LDAPS
* `Save`

#### 検証方法
1. Primary LDAP (`192.168.10.50`) のインターフェースをシャットダウンまたは iptables で 636 をブロック。
2. クライアントから認証を実行。
3. ISE RADIUS Live Logs および `prt-runtime.log` にて、`192.168.10.50` タイムアウト後、自動的に `192.168.10.51` へフェールオーバーし認証が成功することを確認。

---

### 演習例 8: ISE Central Web Authentication (CWA) での LDAP 外部 DB 認証

#### 問題
Cisco WLC 接続端末に対する Central Web Authentication (CWA) において、ゲストポータルログインの外部認証バックエンドとして OpenLDAP サーバーを使用せよ。

#### 要件
1. WLC と ISE 間で CWA (URL Redirect) を構成。
2. ゲスト/社員Webポータルの Identity Source として `OpenLDAP_LDAPS` を指定。

#### 設定例

**1. Guest Portal の Identity Source 設定 (ISE GUI)**:
* `Work Centers > Guest Access > Configure > Guest Portals > Self-Registration Guest Portal`
* **Portal Settings > Portal Customization / Authentication Options**:
  * Identity Source Sequence / Identity Source: `OpenLDAP_LDAPS` を選択
* `Save`

**2. Policy Set 認可ルール設定**:
* 最初の MAB 接続時: Action = `CWA_Redirect_Profile` (Redirect URL & ACL)
* ポータルWebログイン成功後 (RADIUS CoA 発生後): Action = `PermitAccess`

#### 検証方法
1. 端末を Wi-Fi に接続し、ブラウザを起動。
2. ISE Web ポータルにリダイレクトされ、OpenLDAP のユーザー名・パスワードを入力してログイン成功後、インターネットアクセスが許可されることを確認。

---

### 演習例 9: LDAP 属性 (department) に基づく Dynamic Security Group Tag (SGT) 割当

#### 問題
スイッチ接続端末の認証成功時、OpenLDAP の `department` 属性値に応じて Cisco TrustSec SGT を動的に付与せよ。

#### 要件
1. `department` が `Finance` の場合 -> SGT `Finance_SGT` (Tag 15) を付与。
2. `department` が `HR` の場合 -> SGT `HR_SGT` (Tag 16) を付与。

#### 設定例

**1. Authorization Profile の作成**:
* Profile 1 (`Finance_Auth_Profile`): Security Group Option -> SGT = `Finance_SGT (15)`
* Profile 2 (`HR_Auth_Profile`): Security Group Option -> SGT = `HR_SGT (16)`

**2. Authorization Policy 設定**:
* Rule 1: Condition: `OpenLDAP_LDAPS:department EQUALS Finance` -> Action: `Finance_Auth_Profile`
* Rule 2: Condition: `OpenLDAP_LDAPS:department EQUALS HR` -> Action: `HR_Auth_Profile`

#### 検証方法
1. `Finance` 部署のユーザーでログイン後、Sw-1 CLI にて SGT 確認:
```bash
show access-session interface GigabitEthernet1/0/1 details
```
`Security Group Tag: 15` がバインドされていることを確認。

---

### 演習例 10: EAP-TTLS (PAP) を用いた Linux Supplicant の LDAP 認証と VLAN 変革

#### 問題
Linux 端末 (wpa_supplicant) から EAP-TTLS (Inner: PAP) を使用して 802.1X 接続する際、OpenLDAP 認証を実施し VLAN 30 を割り当てよ。

#### 要件
1. 許容プロトコルに `EAP-TTLS` (Inner: PAP) を含めること。
2. 認証成功時、Dynamic VLAN 30 を NAS に送信すること。

#### 設定例

**1. ISE Allowed Protocols 設定**:
* Allowed Protocols リストで `Allow EAP-TTLS` -> Inner Authentication: `PAP` をチェック。

**2. Linux Client (`wpa_supplicant.conf`) 設定**:
```text
network={
    ssid="Corporate_Wired"
    key_mgmt=IEEE8021X
    eap=TTLS
    phase2="auth=PAP"
    identity="john"
    password="JohnPassword123"
}
```

#### 検証方法
1. Linux クライアントから 802.1X 接続を起動。
2. ISE RADIUS Live Logs で `Authentication Method: EAP-TTLS (PAP)` および `Identity Source: OpenLDAP_LDAPS` で認証成功し、VLAN 30 が返却されることを確認。

---

## ❓ 想定試験問題

CCIE Security ラボ試験を想定したコンフィグ読解、トラブルシューティング、および設計問題です。

---

### 問題 1 (コンフィグ読解・プロトコル制限)

ある企業で Cisco ISE と OpenLDAP サーバーを連携させ、新設計画の 802.1X ネットワークを構築しました。しかし、Windows 10 端末（標準 802.1X サプリカント）から接続を試みたところ、認証がすべて失敗します。以下は ISE の Allowed Protocols 設定および RADIUS Live Log のエラー出力です。

**[ISE Allowed Protocols 設定]**
* [x] Allow PEAP
  * [x] Allow MSCHAPv2
  * [ ] Allow EAP-GTC

**[RADIUS Live Log Failure Reason]**
`24401 LDAP authentication failed: MSCHAPv2 is not supported for LDAP identity source`

**【質問】**
1. この接続が失敗する根本的な技術理由を説明せよ。
2. Windows 標準サプリカントの変更を行わずに、または設定変更によってこの問題を解決するための技術的アプローチを 2 点挙げよ。

#### 解答解説
1. **根本理由**: LDAP プロトコルおよび一般的 LDAP サーバーは、MSCHAPv2 のチャレンジ/レスポンス計算に必要な NTLM パスワードハッシュの直接検証をサポートしていません。PEAP-MSCHAPv2 認証を実行するには Active Directory Native Join または特定タイプのデータベースが必要です。
2. **解決アプローチ**:
   * **アプローチ A (EAP-TLS への移行)**: 端末およびユーザーに PKI 証明書を発行し、802.1X 認証プロトコルを EAP-TLS に変更する。ISE は証明書で認証を行い、LDAP は属性取得（検索）のみに使用する（標準的かつ最も推奨される構成）。
   * **アプローチ B (EAP-GTC / EAP-TTLS への変更)**: ISE 側で PEAP-GTC または EAP-TTLS (PAP) を許可し、Windows 端末側に Cisco AnyConnect NAM などの EAP-GTC 対応サプリカントを導入して平文相当のパスワード検証を行わせる。

---

### 問題 2 (トラブルシューティング・証明書暗号化)

ネットワーク管理者が ISE GUI にて LDAPS (TCP 636) 接続の `Test Connection` を実行したところ、以下のエラー画面が表示されました。

**[エラーメッセージ]**
`Test Connection Failed: Unable to establish SSL/TLS connection to LDAP server 192.168.10.50:636. Certificate path validation failed.`

**【質問】**
このエラーを解消するために、Cisco ISE GUI 上で実施すべき具体的な手順を順番に述べよ。

#### 解答解説
1. LDAP サーバー (`192.168.10.50`) の SSL/TLS サーバー証明書を発行した Root CA（および中間 CA がある場合は Intermediate CA）の CA 証明書ファイル（.pem / .cer）を取得する。
2. ISE GUI `Administration > System > Certificates > Trusted Certificates` に移動し、`Import` をクリック。
3. 取得した CA 証明書ファイルをアップロードし、以下の用途チェックボックスを有効化する:
   * `Trust for authentication within ISE`
   * `Trust for client authentication and Syslog`
4. 証明書を保存後、再度 `External Identity Sources > LDAP > Connection` タブを開き、`Test Connection` を実行して「Admin Bind Success」および TLS 接続成功を確認する。

---

### 問題 3 (Identity Source Sequence 設計)

組織内で Active Directory (`AD_Corporate`) と OpenLDAP (`LDAP_Partners`) の 2 つのアイデンティティソースが存在します。エンジニアはこれらを統合した Identity Source Sequence (ISS) `Global_ISS` を作成しましたが、`AD_Corporate` に存在しないパートナー社員が接続した際、認証が拒否されてしまい、`LDAP_Partners` への検索が行われません。

**【質問】**
ISE の ISS 設定において、この問題を修正するために設定変更すべき項目とパラメータを具体的に説明せよ。

#### 解答解説
* **修正項目**: Identity Source Sequence の Advanced Search Options における **"If User Not Found" (ユーザーが見つからない場合)** の動作設定。
* **原因**: デフォルトまたは誤設定により、最初のソース (`AD_Corporate`) でユーザーが見つからない場合に「Do not access next storage in sequence and treat as User Not Found (即時認証拒否)」が選択されているため。
* **設定変更方法**:
  * `Administration > Identity Management > Identity Source Sequences > Global_ISS` を開く。
  * `Advanced Search Option` にて、`If User Not Found` の選択肢を **`Treat as User Not Found and continue to next source in sequence`** (チェックボックスを外し継続させる) に変更して保存する。

---

### 問題 4 (コンフィグ検証・dACL 属性)

以下は ISE から スイッチ Sw-1 に返却された RADIUS Access-Accept の属性情報およびスイッチ側の出力です。

**[ISE RADIUS Access-Accept Attributes]**
* `Tunnel-Type = 13 (VLAN)`
* `Tunnel-Medium-Type = 6 (802)`
* `Tunnel-Private-Group-ID = 30`
* `Filter-Id = DACL_LDAP_GUEST`

**[Sw-1 ログ]**
`%RADIUS-4-DACL_NOT_FOUND: Dynamic ACL DACL_LDAP_GUEST not found on device, denying access.`

**【質問】**
1. スイッチでログが発生し、通信が拒否された理由を説明せよ。
2. ISE 側の設定において、この問題を防止するための正しい設定方法を述べよ。

#### 解答解説
1. **拒否理由**: ISE から `Filter-Id` 属性としてテキスト名 `DACL_LDAP_GUEST` が送信されましたが、スイッチ Sw-1 のローカルコンフィグに `ip access-list extended DACL_LDAP_GUEST` が定義されていないため、スイッチが ACL を適用できずアクセスを拒否しました。
2. **正しい設定方法**:
   * ISE の Downloadable ACL (dACL) 機能を使用する場合、`Filter-Id` 属性ではなく、cisco-av-pair 属性の `acs:dacl=` (Downloadable ACL) を使用する。
   * ISE GUI の `Policy Elements > Results > Authorization > Downloadable ACLs` で ACL コンテンツを定義し、Authorization Profile 内で `Downloadable ACL Name` として選択して送信させる。これにより、スイッチは Access-Accept 受信時に ISE から動的に ACL 内容をダウンロードして適用する。

---

### 問題 5 (LDAP Schema & 認可マッピング)

OpenLDAP サーバー上のグループ情報 (`memberOf`) が以下の構造になっています:
`memberOf: cn=SecOps,ou=SecurityGroups,dc=company,dc=local`

ISE 上の Authorization Policy にて「SecOps グループに属するユーザー」を合致させるためのグループ検索設定および Policy 条件文の適切な作成手順を説明せよ。

#### 解答解説
1. **ISE LDAP Groups タブでのグループ抽出**:
   * `Administration > Identity Management > External Identity Sources > LDAP > [LDAP_Name] > Groups` タブを開く。
   * `Add` をクリックし、`Member Attribute` に `memberOf` を指定して検索を実行。
   * `cn=SecOps,ou=SecurityGroups,dc=company,dc=local` を選択してインポートし、ISE 内部グループとして保存する。
2. **Authorization Policy 条件文の設定**:
   * `Policy > Policy Sets` にて該当ルールを作成。
   * Condition 条件文: `[LDAP_Name]:ExternalGroups EQUALS cn=SecOps,ou=SecurityGroups,dc=company,dc=local`
   * 一致時の Action として適切な Authorization Profile を割り当てる。

---

## 🔗 参考リソース

Cisco 公式ドキュメント、Cisco Live 資料、およびテクニカルガイドのリソース一覧です。

* **Cisco Live セッション動画・スライド**:
  * Cisco Live BRKSEC-3432 - Cisco ISE Integration with External Identity Sources
  * Cisco Live BRKSEC-2041 - Advanced ISE Architecture and Deployment
* **Cisco Official Configuration Guides**:
  * Cisco Identity Services Engine Administrator Guide, Release 3.1 - Identity Sources
  * Cisco ISE LDAP Integration Guide
* **Configuration Examples & Technical Notes**:
  * Configure ISE 3.0 Integration with OpenLDAP Server
  * Configure EAP-TLS Authentication with ISE and LDAP Attribute Lookup
  * Troubleshoot ISE LDAP Connection and Authentication Failures
* **Design Guides & Cisco Validated Designs (CVD)**:
  * Cisco Secure Access Control System / ISE Design Guide

---

## 📝 **補足（Notes）**

### 試験直前チェックリスト
* [ ] LDAPS (Port 636) 接続のために Trusted Certificates へ CA 証明書を正しくインポートしているか？
* [ ] LDAP 認証で PEAP-MSCHAPv2 が使えない理由（パスワードハッシュ直接参照不可）を即答できるか？
* [ ] PEAP-GTC または EAP-TLS / EAP-TTLS-PAP への変更構成を迷わず設定できるか？
* [ ] OpenLDAP の `uid` / `cn` と AD の `sAMAccountName` のスキーマ相違点を理解し、ISE の General タブで正しく属性選択できるか？
* [ ] Identity Source Sequence (ISS) で `User Not Found` 発生時に次ソースへ継続（Continue）するアドバンスドオプションを設定できるか？
* [ ] LDAP の `memberOf` 完全 DN 文字列を Authorization Policy の条件に正しく設定できるか？
* [ ] ISE CLI の `prt-runtime.log` を用いた LDAP デバッグ方法を把握しているか？
