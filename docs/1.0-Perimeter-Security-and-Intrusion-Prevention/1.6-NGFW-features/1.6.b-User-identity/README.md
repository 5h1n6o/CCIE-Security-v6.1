---
layout: default
title: 1.6.b-User-identity
nav_order: 2
parent: 1.6-NGFW-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.6.b User identity

Cisco Secure Firewall Threat Defense (FTD) および Management Center (FMC) における **User Identity（ユーザーアイデンティティ統合・Identity Firewall）** は、伝統的な IP アドレスベースのセキュリティ制御から脱却し、「誰が（ユーザー名）」「どのグループに所属し（AD Group / LDAP Group）」「どのようなセキュリティ属性（Security Group Tag: SGT）を持って」通信しているかを識別し、アクセス制御ポリシー（Access Control Policy: ACP）や QoS、暗号化制御、脅威評価に動的に適用する高度なセキュリティ機能です。

CCIE Security v6.1 Blueprint **1.6.b User identity** では、Active Directory (AD) や LDAP との連携オブジェクトである **Realms（レルム）**、Cisco ISE / ISE-PIC との **pxGrid** 連携による **Passive Authentication（パッシブ認証）**、FTD 単体で駆動する Web リダイレクト型の **Active Authentication (Captive Portal: アクティブ認証)**、Terminal Services (TS) Agent、ならびに **Identity Policy（アイデンティティポリシー）** の適用シーケンスとトラブルシューティング能力が深く問われます。

---

## 📘 概要

* **機能概要**:
  * **Realm (レルム)**: Active Directory や LDAP サーバーに接続し、ユーザーおよびユーザーグループのディレクトリ構造（メタデータ）を FMC に同期・更新するオブジェクト。
  * **Passive Authentication (パッシブ認証)**: ユーザーにログイン画面を表示させることなく、Cisco ISE / ISE-PIC (pxGrid 経由) や AD User Agent / TS Agent、Syslog 等から「IP アドレス ↔ ユーザー名」のマッピング情報を裏で取得して透明に特定する方式。
  * **Active Authentication (アクティブ認証 / Captive Portal)**: パッシブ認証で特定できなかった通信やゲストユーザーに対し、HTTP/HTTPS トラフィックを FTD の Web 認証ポータルへ自動リダイレクトし、ユーザー名・パスワードの入力を求めてアクティブに認証する方式。
  * **Identity Policy (アイデンティティポリシー)**: どのトラフィックに対してどの Realm、認証方式（Passive / Active / No Authentication）、フォールバック動作を適用するかを定義するポリシー。
* **利用目的**:
  * 動的 IP 環境（DHCP、Wi-Fi、VDI 等）において、IP アドレスの変更に左右されない一貫したユーザー単位・部門単位のアクセス制御の確立。
  * FMC のイベントログ、接続履歴、脅威検知アラート（Intrusion Events、File/Malware Events）に対して具体的なユーザー名を紐付け、フォレンジックや監査の可視性を極限まで高めること。
* **どのような場面で利用するか**:
  * Active Directory ドメイン環境内の社内端末（Windows / Mac）に対する透明なアクセス制御（Passive Identity）。
  * ゲスト端末、BYOD 端末、またはドメイン非参加端末に対する Web 認証の強制（Active Identity / Captive Portal）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **ユーザー識別方式** | **Passive Authentication**（透明型マッピング）<br>**Active Authentication**（Captive Portal 認証）<br>**No Authentication**（IP のみで識別） |
| **アイデンティティ情報源** | **Cisco ISE / ISE-PIC**（pxGrid v1/v2 経由）<br>**FMC User Agent**（AD WMI キャプチャ）<br>**Terminal Services (TS) Agent**（マルチユーザー環境用） |
| **ディレクトリ基盤 (Realm)** | Active Directory (AD)、OpenLDAP (LDAP / LDAPS - TCP 389/636) |
| **制御対象プロトコル** | **Passive Auth**: 全 IP トラフィック<br>**Active Auth (Captive Portal)**: **HTTP (TCP 80) / HTTPS (TCP 443) のみ** |
| **フォールバック機能** | パッシブ認証失敗時に自走して Active Authentication (Captive Portal) へフォールバック可能 |
| **ライセンス要件** | **Base License**（機能利用に追加ライセンス費用は発生しないが Threat 等と組み合わせ可能） |
| **制限事項** | Active Authentication は **Routed Firewall Mode 専用**（Transparent Mode 非サポート）。HTTPS リダイレクトには **SSL Inspection Policy** が必要 |

---

## 🏗 動作原理

User Identity システムは、ディレクトリ情報を管理する **Realm**、認証ログを集約・配信する **Identity Source (ISE/Agent)**、通信とユーザーを評価する **Identity Policy**、および実際のデータパス制御を行う **Access Control Policy** の 4 つの要素で構築されます。

```text
1. Passive Authentication Flow (via Cisco ISE & pxGrid)

   [ Client PC ] ───(1) Domain Logon (Kerberos) ───► [ Active Directory ]
         │                                                    │
         │                                            (2) WMI / Syslog / Audit Log
         │                                                    ▼
         │                                           [ Cisco ISE / ISE-PIC ]
         │                                                    │
         │                                            (3) pxGrid Pub/Sub Bus
         │                                                (IP-to-User Mapping)
         │                                                    ▼
   [ Transit Data ] ───────────────────────────────► [ FMC / FTD Engine ]
   (Matching ACP User Rules based on Active IP-User Cache)

2. Active Authentication Flow (Captive Portal)

   [ Unmapped Client ] ───(1) HTTP/HTTPS Request ───► [ FTD (Identity Policy) ]
            ▲                                                  │
            │                                         (2) HTTP 302 Redirect
            │                                         (Port 885 / Custom Port)
            │                                                  ▼
            └──────────(3) Submit AD Credentials ──────────────┘
                                  │
                          (4) LDAP Check against AD Realm
                                  │
                                  ▼
                   [ User Mapped & Session Cached ]
```

---

## ⚙ 動作シーケンス

1. **Realm によるディレクトリ同期**:
   * FMC は Realm 設定に従い、AD/LDAP サーバーと LDAPS 通信を定期的に行い、ユーザー、グループ構造、OU (Organizational Unit) メタデータをローカルデータベースへ同期・保管します。
2. **パッシブアイデンティティの取得 (Passive Identity Mapping)**:
   * ユーザーが Windows 端末から AD へログオンすると、ISE/ISE-PIC（または AD Agent）が Domain Controller のイベントログを検出。
   * ISE は pxGrid トピック (`SessionDirectory`) を介して「IP: `10.1.1.50` ＝ User: `corp\jdoe`」というセッションバインディング情報を FMC へリアルタイム配信します。
   * FMC はこの情報を管理下の全 FTD アプライアンスの内部ユーザーキャッシュ (User Identity Memory Cache) へプッシュします。
3. **パケット着信と Identity Policy 評価**:
   * トラフィックが Ingress インターフェイスに着信。FTD はパケットの送信元 IP アドレスを自身のユーザーキャッシュと照合します。
   * **キャッシング済みの場合**: 該当パケットに `corp\jdoe` 属性を内部付与し、Access Control Policy の評価へ移行します。
   * **未キャッシングかつ Active Auth 対象の場合**: トラフィックが HTTP/HTTPS であれば、FTD は Captive Portal エンジンへルーティングし、ユーザーのブラウザへ HTTP 302 リダイレクトを返答して認証ポータルを表示させます。
4. **Access Control Policy の適用**:
   * Access Control Rules の **`Users` タブ** に定義された条件（例: `Realm: AD_CORP / Group: Sales_Dept`）と、パケットに付与されたユーザー属性が照合されます。
   * 条件が一致した場合、定義されたアクション（Allow, Block, IPS/File Inspection バインド等）が実行されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Realm 設定でのディレクトリエラー回避**:
   * **AD バインド用アカウント**: ドメイン管理者権限は必須ではないが、`Domain Users` 以上の読み取り権限を持ったアカウントが必要。
   * **Base DN / Group DN の指定**: ディレクトリツリー全体（`DC=corp,DC=local`）を指定すると同期に数十分かかる場合がある。試験では指定された OU (例: `OU=Employees,DC=corp,DC=local`) に限定して Base DN を設定することが推奨されます。
   * **User Download の実行**: Realm を作成・保存しただけでは FMC 上で Access Control Rule の条件としてユーザー/グループを選択できません。**`User Download` を手動実行 (Download Now) またはスケジュール化** する必要があります。
2. **pxGrid ピアリングの完全確立手順 (ISE ↔ FMC)**:
   * FMC と ISE 間の pxGrid 接続においては、以下の PKI 条件を満たす必要があります：
     1. FMC と ISE の相互信頼（ISE Root CA を FMC の `pxGrid Server CA` および `MNT Server CA` に登録）。
     2. FMC 用の pxGrid クライアント証明書を ISE 内部 CA 等で発行し、FMC の `Identity Sources > Identity Services Engine` にアサイン。
     3. ISE の `Administration > pxGrid Services > Clients` 画面で FMC の接続リクエスト (`ia-fmc-...` および `st-fmc-...`) を **`Approve`（承認）** する。
3. **Captive Portal (Active Auth) 設定時の 3 大ハマりポイント**:
   * **Firewall Mode 要件**: Captive Portal は **Routed Firewall Mode 専用**。Transparent Mode では機能しません。
   * **HTTPS リダイレクト時の SSL Decryption バインド**: HTTPS トラフィックを Captive Portal でリダイレクト・アクティブ認証する場合、**SSL Inspection Policy で該当通信を Decrypt（復号）** していなければ、FTD は TLS ハンドシェイクを割り込んで HTTP リダイレクトを発行できません。
   * **Platform Settings での Captive Portal ポート有効化**: FMC の `Devices > Platform Settings > Interfaces` で、Captive Portal 応答ポート（デフォルト: TCP 885 または 8890）が対象インターフェイス上で許可・バインドされているか確認する。
4. **Special Identities (Guest, Unknown, Failed Authentication) の設計**:
   * アイデンティティマッピングに失敗した通信に対して、全拒否するのではなく限定アクセス（ゲストゾーン宛てのみ等）を与える場合、Access Control Rule の Users タブで **`Special Identities`** (`Guest`, `Unknown`, `Failed Authentication`) を指定します。
5. **Passive Authentication から Active Authentication への自動フォールバック**:
   * Identity Policy のルールにおいて、Action を `Passive Authentication` に設定し、**`Use active authentication if passive or VPN identity cannot be established`** にチェックを入れることで、未登録端末に対してのみ Captive Portal を透過的に発動させることができます。

---

## 🛠 設定方法

### 1. FMC GUI: Realm (Active Directory) の設定手順

```text
[ Step 1: AD Realm オブジェクトの作成 ]
1. FMC UI から System > Integration > Realms に移動。
2. "Add Realm" をクリック:
   - Name: AD_CORP_REALM
   - Type: AD
   - AD Primary Domain: corp.local
   - Directory Active Directory Username: join_account@corp.local
   - Directory Active Directory Password: Password123
   - Base DN: DC=corp,DC=local
   - Group DN: OU=Groups,DC=corp,DC=local
3. "Directory" タブで AD サーバーの IP アドレス (例: 10.1.1.10) と Port (389 または LDAPS 636) を入力。
4. "Test" をクリックし "Success" を確認。

[ Step 2: ユーザー/グループメタデータの自動ダウンロード ]
1. "User Download" タブを選択。
2. [✓] Download users and groups にチェック。
3. Available Groups から対象グループ (例: Sales, Engineering) を選択し "Add to Include" を押下。
4. "Save" をクリック。
5. Realm 一覧画面で "AD_CORP_REALM" のステートを "State: Enabled" (トグルスイッチ ON) に変更。
6. "Download Now" (ダウンロードアイコン) を押下して同期を完了させる。
```

---

### 2. FMC GUI: Cisco ISE pxGrid 連携による Passive Identity Source の追加

```text
[ Step 1: FMC への ISE 信頼証明書の登録 ]
1. Objects > Object Management > PKI > Trusted CAs に移動し "Add Trusted CAs" を選択。
2. ISE の Root CA 証明書 (PEM 形式) をインポート (オブジェクト名: ISE_ROOT_CA)。

[ Step 2: FMC pxGrid クライアント証明書の作成 ]
1. ISE GUI の Administration > pxGrid Services > Certificates タブで FMC 用の pxGrid 証明書/秘密鍵 PKCS12 または PEM を生成してダウンロード。
2. FMC UI の Objects > Object Management > PKI > Internal Certs で生成した証明書と鍵をインポート (オブジェクト名: FMC_PXGRID_CERT)。

[ Step 3: Identity Source (ISE) の追加 ]
1. FMC UI の System > Integration > Identity Sources に移動。
2. "Identity Services Engine" タブを選択:
   - Primary Host Name/IP Address: ise01.corp.local (10.1.1.20)
   - pxGrid Server CA: ISE_ROOT_CA
   - MNT Server CA: ISE_ROOT_CA
   - FMC Server Certificate: FMC_PXGRID_CERT
3. "Save" をクリック。
4. ISE GUI の Administration > pxGrid Services > Clients 画面を開き、FMC からの未承認クライアントを選択して "Approve" をクリック。
5. FMC UI に戻り "Test" をクリックして "Success" となることを確認。
```

---

### 3. FMC GUI: Identity Policy ＆ Captive Portal (Active Auth) の統合構築

```text
[ Step 1: Identity Policy の作成 ]
1. Policies > Access Control > Identity に移動し "New Identity Policy" を作成 (名前: CORP_IDENTITY_POLICY)。
2. "Add Rule" (Rule 1: Passive Auth with Fallback to Active Auth):
   - Name: PASSIVE_WITH_FALLBACK
   - Action: Passive Authentication
   - Realm: AD_CORP_REALM
   - [✓] Use active authentication if passive or VPN identity cannot be established にチェック。
   - Active Authentication Response Page: System-provided (または Custom)
3. Save をクリック。

[ Step 2: Access Control Policy へのバインドと User Rule の作成 ]
1. Policies > Access Control で対象の ACP を編集。
2. 画面上部の "Identity Policy: None" をクリックし "CORP_IDENTITY_POLICY" をバインド。
3. "Add Rule" (Rule 1: Allow Sales Group Internet Access):
   - Name: ALLOW_SALES_WEB
   - Action: Allow
   - Users タブ: Realm = AD_CORP_REALM ➔ Selected Users/Groups = "Sales" を追加。
   - Ports タブ: HTTP (80), HTTPS (443)
4. Save して対象 FTD へ Deploy。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI / FMC) | 表示確認ポイント |
| :--- | :--- | :--- |
| **FTD 内部ユーザーキャッシュの即時確認** | <code>show user-identity user</code> | IP アドレス ↔ ユーザー名 (例: `10.1.1.50 -> corp\jdoe`) の対応表 |
| **ISE pxGrid セッション接続ステータス確認** | <code>show user-identity status</code> / <code>show dynamic-objects</code> | `pxGrid State: CONNECTED`、受領セッション数 |
| **Realm の同期ステータス・エラー確認** | <code>show user-identity errors</code> | AD 接続エラー、ユーザー属性同期エラーログ |
| **Captive Portal アクティブ認証セッション確認** | <code>show user-identity active-authentication-sessions</code> | Web 認証成功済みのアクティブユーザーセッション情報 |
| **Snort 3 エンジンにおけるユーザー属性マッチ追跡** | <code>system support firewall-engine-debug</code> | 通信パケットに対して識別された `User: corp\jdoe`、`Group: Sales` の評価表示 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **FMC で Access Control Rule を作成する際、Users タブにグループが表示されない** | Realm 設定後に `User Download` を手動実行していない、または AD の Base DN 設定不備 | FMC Realm 設定画面 | Realm 編集画面の "User Download" タブで Download Now を実行し、完了後に ACP 編集画面を再読み込みする。 |
| **pxGrid の接続テストを実行すると「Connection Refused / Unauthorized」エラーになる** | ① ISE 上で FMC クライアントが `Pending`（未承認）状態<br>② FMC/ISE 間の NTP 時刻ズレ<br>③ 証明書の SAN / CN 不一致 | FTD CLI: <code>show user-identity status</code><br>ISE: `pxGrid Services > Clients` | ISE GUI で FMC の pxGrid クライアントを Approve する。FMC と ISE の NTP 時間を完全に同期させる。 |
| **ユーザーが AD ログインしても FTD 上でパッシブ特定されず IP のまま通過する** | Domain Controller 側でセキュリティ監査ログ (Event ID 4624) の出力が無効化されている | AD: `eventvwr.msc`<br>FTD CLI: <code>show user-identity user</code> | AD の Group Policy で `Audit Logon Events (Success)` を有効化し、Event ID 4624 が生成されるようにする。 |
| **HTTPS サイトへアクセスした際、Captive Portal ログイン画面へリダイレクトされない** | SSL Inspection Policy が設定されておらず、FTD が HTTPS ハンドシェイク内に介入・リダイレクトできない | FMC SSL Policy | 対象 HTTPS 通信を復号（Decrypt - Resign 等）する SSL Inspection Policy を作成し ACP へバインドする。 |
| **Captive Portal で正しい AD パスワードを入力しても認証失敗する** | FMC から AD への LDAPS/LDAP 認証ポート (389/636) が途中 FW で遮断されている、または Realm Bind アカウントのロックアウト | <code>show user-identity errors</code> | AD サーバー宛ての LDAP (389/636) 到達性を確認し、Realm 編集画面の "Test" ボタンで認証成功を確認する。 |

---

## ⚠ 制限事項

* **Captive Portal の動作モード制限**:
  * Captive Portal (Active Authentication) は **Routed Firewall Mode でのみサポート** されます。Transparent Mode では機能しません。
* **プロトコル制限 (Active Auth)**:
  * Active Authentication (Captive Portal) がトリガー・作動するのは **HTTP (TCP 80) および HTTPS (TCP 443) トラフィックのみ** です。SSH や FTP、DNS 通信では Web 認証ポータルを表示できません。
* **ユーザーマッピングの容量上限**:
  * 1 台の FTD アプライアンスが保持できる「IP ↔ ユーザー名」のマッピング保持件数はモデルのメモリサイズに依存します（例: FTD 2100 シリーズで最大数十万セッション）。上限を超えた場合、古いマッピングからドロップされます。

---

## 🔄 他技術との関連

* **Cisco ISE (Identity Services Engine)**: pxGrid バスパブリッシャーとして、802.1X / MAB / AD ログオンに基づくユーザーアイデンティティおよび Security Group Tag (SGT) を FTD へ自動共有。
* **SSL Inspection Policy**: Active Authentication (Captive Portal) で HTTPS 通信をリダイレクト処理するために必須となる事前復号コンポーネント。
* **Access Control Policy (ACP)**: Identity Policy によって特定されたユーザー属性（User / Group / Special Identities）を条件として、アクセス許可/拒否/IPS 検査を決定する最終評価エンジン。

---

## 🧩 比較表

### 1. Passive Authentication vs Active Authentication (Captive Portal)

| 比較項目 | Passive Authentication | Active Authentication (Captive Portal) |
| :--- | :--- | :--- |
| **ユーザー体験** | **完全透明 (ユーザー操作不要)** | Web ブラウザ上に認証画面が表示される |
| **トラフィック条件** | 全 IP プロトコル・ポートに対応 | **HTTP (TCP 80) / HTTPS (TCP 443) 専用** |
| **対応端末** | AD ドメイン参加済み Windows / Mac | ゲスト端末、BYOD、未登録端末、Linux 等全般 |
| **必要な連携構成** | Cisco ISE (pxGrid) または AD User Agent | FTD 上の Captive Portal ポートおよび SSL 復号設定 |
| **Firewall Mode** | Routed Mode / Transparent Mode 両対応 | **Routed Firewall Mode 専用** |

### 2. ISE pxGrid 連携 vs AD User Agent (MDA) 連携

| 比較項目 | Cisco ISE pxGrid 連携 | AD User Agent (MDA) 連携 |
| :--- | :--- | :--- |
| **データ共有方式** | REST / WebSocket ベースの **pxGrid Pub/Sub Bus** | AD 上の Windows サービス経由の Direct Push |
| **取得可能属性** | ユーザー名、グループ、**SGT (TrustSec)**、**端末タイプ** | ユーザー名、IP アドレス、AD グループのみ |
| **拡張性・冗長性** | 高高度 (ISE PSN クラスタによる冗長化) | 限定的 (User Agent サーバー単体障害リスク) |
| **推奨アーキテクチャ** | **Cisco 推奨アーキテクチャ** | 小規模環境またはレガシー移行過渡期向け |

---

## 💡 ベストプラクティス

1. **Passive Auth から Active Auth へのハイブリッド構成 (Fallback)**:
   * Identity Policy で基本アクションを `Passive Authentication` とし、**`Use active authentication if passive...` を有効化** します。これにより、ドメイン参加端末は透過的に通過させつつ、未知の端末やゲスト端末のみをピンポイントで Captive Portal 認証へリダイレクトさせるシームレスな運用を実現します。
2. **Realm のグループダウンロード対象絞り込み**:
   * 大規模 AD 環境において、AD 内の全ユーザー/グループを FMC にダウンロードするとパフォーマンス低下の原因となります。Realm の `User Download` タブで、実際に Access Control Policy で使用する特定の OU やセキュリティグループのみを選択してダウンロード対象（Include List）に指定します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Active Directory Realm の構築とグループ同期

#### 問題
AD ドメイン `corp.local` (DC IP: `10.1.1.10`) に接続する Realm `REALM_CORP` を作成し、グループ `Engineering` のメタデータを同期させよ。

#### 要件
* FMC GUI を使用して構成すること。
* バインドユーザー: `fmc_bind@corp.local` / パスワード: `Cisco123!`

#### 設定・操作手順

```text
1. System > Integration > Realms を開き "Add Realm" をクリック。
2. General タブ:
   - Name: REALM_CORP
   - Type: AD
   - AD Primary Domain: corp.local
   - Directory Active Directory Username: fmc_bind@corp.local
   - Password: Cisco123!
   - Base DN: DC=corp,DC=local
3. Directory タブ:
   - Primary Node: 10.1.1.10, Port: 389
4. "Test" をクリックして成功を確認後、"User Download" タブへ移動:
   - Available Groups から "Engineering" を選択し "Add to Include" を押下。
5. Save をクリックし、Realm 一覧で REALM_CORP を Enabled に変更。"Download Now" を押下。
```

#### 検証方法
* FMC の `Analysis > Users > Users` 画面を開き、`Engineering` グループに所属する AD ユーザー一覧が正常にロードされていることを確認。

---

### 例2: Cisco ISE pxGrid による Passive Identity Source 構成

#### 問題
Cisco ISE (`10.1.1.20`) と FMC 間に pxGrid 接続を構築し、パッシブユーザーマッピングの動的受信を有効化せよ。

#### 要件
* オブジェクト名: `ISE_PRIMARY_SOURCE`

#### 設定・操作手順

```text
1. Objects > Object Management > PKI > Trusted CAs で ISE Root CA をインポート (`ISE_ROOT_CA`)。
2. System > Integration > Identity Sources > Identity Services Engine タブを選択:
   - Primary Host Name/IP Address: 10.1.1.20
   - pxGrid Server CA: ISE_ROOT_CA
   - MNT Server CA: ISE_ROOT_CA
   - FMC Server Certificate: (FMC 用 pxGrid 証明書を選択)
3. Save をクリック。
4. ISE GUI で FMC の接続申請を Approve し、FMC 上で "Test" をクリックして SUCCESS を確認。
```

#### 検証方法
* FTD CLI で `show user-identity status` を実行し、`pxGrid State: CONNECTED` と表示されることを確認。

---

### 例3: Identity Policy による Passive Authentication の適用

#### 問題
社内 LAN インターフェイス (`INSIDE_ZONE`) からの全トラフィックに対して Realm `REALM_CORP` を使用したパッシブ認証を適用する Identity Policy `POL_ID_PASSIVE` を構築せよ。

#### 要件
* アクション: `Passive Authentication`

#### 設定・操作手順

```text
1. Policies > Access Control > Identity で "New Identity Policy" を作成 (名前: POL_ID_PASSIVE)。
2. Add Rule:
   - Name: PASSIVE_RULE
   - Action: Passive Authentication
   - Zones: Source = INSIDE_ZONE
   - Realm: REALM_CORP
3. Save をクリック。
4. ACP の編集画面を開き、上部の Identity Policy に "POL_ID_PASSIVE" をバインドして Deploy。
```

#### 検証方法
* 内部 PC から AD ログイン後、外部へ通信を発生させ、FTD CLI で `show user-identity user` を実行して「PC IP ➔ AD ユーザー名」のマッピングが表示されることを確認。

---

### 例4: Captive Portal (Active Authentication) の基本構成

#### 問題
ドメイン非参加端末向けに、HTTP 通信発生時に Web 認証ポータルを表示させて AD 認証を求める Captive Portal ポリシーを構築せよ。

#### 要件
* 応答ポート: TCP `885`
* Identity Policy Rule Action: `Active Authentication` (Protocol: HTTP Basic)

#### 設定・操作手順

```text
1. Devices > Platform Settings で FTD Platform Settings を編集し、Captive Portal ポート 885 を有効化。
2. Identity Policy にルールを追加:
   - Name: ACTIVE_AUTH_RULE
   - Action: Active Authentication
   - Realm: REALM_CORP
   - Authentication Protocol: HTTP Basic
3. ACP にバインドして Deploy。
```

#### 検証方法
* 未登録端末から Web ブラウザで `[suspicious link removed]` へアクセスし、FTD の Captive Portal 認証画面へリダイレクトされることを確認。

---

### 例5: Passive Auth 失敗時の Captive Portal フォールバック設定

#### 問題
パッシブ認証で「IP ↔ ユーザー」の特定に失敗した通信について、自動的に Captive Portal アクティブ認証へリダイレクトさせるフォールバック規則を構成せよ。

#### 要件
* `Use active authentication if passive identity cannot be established` オプションを使用すること。

#### 設定・操作手順

```text
1. Identity Policy 内の Passive Auth ルールを編集。
2. [✓] Use active authentication if passive or VPN identity cannot be established にチェックを入れる。
3. Active Authentication Response Page: System-provided を選択。
4. Save をクリックして Deploy。
```

#### 検証方法
* パッシブマッピングが存在しないテスト端末から Web アクセスを実行し、自動的に Captive Portal ポータル画面が表示されることを確認。

---

### 例6: Access Control Policy でのユーザーグループに基づくアクセス制限

#### 問題
AD の `Sales` グループに属するユーザーのみ、インターネット宛ての HTTP/HTTPS アクセスを許可し、それ以外のユーザーをブロックせよ。

#### 要件
* ACP ルールの Users タブ条件を使用すること。

#### 設定・操作手順

```text
1. Policies > Access Control で対象 ACP を編集し、新規ルールを追加。
2. Add Rule:
   - Name: ALLOW_SALES_ONLY
   - Action: Allow
   - Users タブ: Realm = REALM_CORP ➔ Groups = "Sales" を追加。
   - Ports タブ: HTTP, HTTPS
3. デフォルトアクションを Block に設定し、Deploy を実行。
```

#### 検証方法
* `Sales` グループのユーザーでログインした端末から Web 閲覧（成功）し、`Marketing` グループのユーザー端末からアクセス試行してブロックされることを確認。

---

### 例7: Special Identities (Guest / Unknown) を利用した未認証端末の制御

#### 問題
認証に失敗した端末 (`Failed Authentication`) およびユーザー不詳の端末 (`Unknown`) からの通信を、隔離 VLAN 宛てのみ許可する安全ルールを作成せよ。

#### 要件
* Users タブの `Special Identities` オブジェクトを使用すること。

#### 設定・操作手順

```text
1. ACP ルールを作成:
   - Name: ISOLATE_UNAUTHENTICATED
   - Action: Allow
   - Users タブ: "Special Identities" から `Failed Authentication` および `Unknown` を追加。
   - Destination Networks: QUARANTINE_NET
2. Save して Deploy。
```

#### 検証方法
* 認証失敗した端末からの通信が隔離ネットワーク (`QUARANTINE_NET`) 宛てのみ正常に疎通し、主要基幹サーバー宛ては拒否されることを検証。

---

### 例8: HTTPS トラフィックに対する Captive Portal リダイレクトと SSL Inspection 連携

#### 問題
HTTPS アクセス (`https://...`) 発生時にも Captive Portal リダイレクト画面を正常表示させるため、SSL Inspection Policy とのバインドを行え。

#### 要件
* SSL Policy アクション: `Decrypt - Resign`

#### 設定・操作手順

```text
1. SSL Policy で Outbound 通信に対する `Decrypt - Resign` ルールを構成。
2. ACP 編集画面で作成した SSL Policy をアサイン。
3. Identity Policy で Active Authentication ルールが構成されていることを確認して Deploy。
```

#### 検証方法
* 端末から `https://www.cisco.com` へアクセスした際、SSL 証明書エラーを伴わずに Captive Portal 認証画面が表示され、ログイン完了後に目的のページへ遷移することを確認。

---

### 例9: CLI による FTD ユーザーマッピングのリアルタイムデバッグ

#### 問題
FTD CLI にログインし、特定 IP (`10.1.1.100`) に対するパッシブユーザーマッピングの更新と削除をリアルタイムデバッグ追跡せよ。

#### 要件
* `debug user-identity` コマンドを使用すること。

#### 設定・操作手順

```bash
> debug user-identity enable
> debug user-identity trace
```

#### 検証方法
* 対象端末から AD ドメインへログオン/ログオフを発生させ、FTD CLI 上に `User mapping added: 10.1.1.100 -> corp\user01` のデバッグログが出力されることを確認。

---

### 例10: FMC REST API を使用した Active Identity Sessions の取得

#### 問題
FMC REST API を使用して、管理下の FTD アプライアンス上で現在アクティブになっているユーザーセッション一覧情報を JSON 形式で取得せよ。

#### 要件
* `/api/fmc_config/v1/domain/{domainUUID}/users/usersessions` エンドポイントを使用すること。

#### 設定・操作手順

```bash
# ===== Auth Token 取得後、User Sessions を GET =====
curl -k -X GET "https://fmc.corp.local/api/fmc_config/v1/domain/e276abec-e0f2-11e3-8169-6d9ed13b206f/users/usersessions" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json"
```

#### 検証方法
* 返却された JSON レスポンス内に、ログイン中の IP アドレス、ユーザー名、認証種別 (`PASSIVE` / `ACTIVE`) が表示されていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で Active Directory と接続する Realm を新規作成し、バインド設定を完了した。その後、Access Control Policy のルールで特定ユーザーグループを条件に指定しようとしたが、Users タブの Group 一覧に AD のグループ名が一切表示されなかった。原因として最も可能性が高いものはどれか。

* A) FTD アプライアンスのライセンスが Base ライセンスにダウングレードしているため。
* B) Realm 設定画面で "User Download" タブの同期処理 (Download Now) を完了させていないため。
* C) FTD が Transparent Firewall Mode で動作しているため。
* D) FMC ↔ FTD 間の sftunnel が切断されているため。

**正解**: **B**  
**解説**: FMC 上で Realm を追加・保存しただけでは、AD のユーザーおよびグループのメタデータは FMC データベースにロードされません。Realm 編集画面の **`User Download` タブ** で `Download Now` を実行して AD と同期させることで、初めて Access Control Rule の条件としてグループ名が選択可能になります。

---

### 2. トラブルシュート問題

**問題**: FMC と Cisco ISE 間で pxGrid 接続を設定し、テストボタンを押したが「Connection Refused」エラーが発生して接続できない。ISE GUI の `Administration > pxGrid Services > Clients` を確認したところ、FMC のクライアント名が `Pending` と表示されていた。適切な対処方法はどれか。

* A) FMC の FTD デバイス登録を解除して再登録する。
* B) ISE GUI 上で Pending 状態の FMC pxGrid クライアントを選択し、「Approve」ボタンを押して承認する。
* C) FTD の Firewall Mode を Routed Mode から Transparent Mode に変更する。
* D) Realm のプロトコルを LDAP から LDAPS へ変更する。

**正解**: **B**  
**解説**: ISE と外部コンシューマー (FMC 等) の pxGrid 接続では、セキュリティ上、初回接続時に ISE 側でクライアントアカウントの**手動承認 (Approve)** が必要となります（Auto-approve 設定時を除く）。Approve 処理を行うことで pxGrid ピアリングが完了します。

---

### 3. Design 問題

**問題**: 企業ネットワークにおいて、社内のドメイン参加 Windows PC（約 3,000 台）に対しては透過的で画面入力を求めない認証を適用し、来客用 Wi-Fi や持ち込みスマホ（約 500 台）に対してのみ Web ログイン画面を表示させてアクセス制限を行いたい。最も効率的な Identity Policy の設計要件はどれか。

* A) 全トラフィックに対して Active Authentication (Captive Portal) のみを一律適用する。
* B) Identity Policy で Action を `Passive Authentication` とし、`Use active authentication if passive or VPN identity cannot be established` オプションを有効化する。
* C) 全ポートを Passive Mode インターフェイスに変更する。
* D) FTD を Single Instance モードから Multi-Instance モードへ変更する。

**正解**: **B**  
**解説**: **Passive Authentication に Active Auth フォールバックオプションを組み合わせる** 設計により、ドメイン端末は AD ログイン情報に基づいて完全に透過的に特定され、パッシブ特定できなかった未登録端末やゲスト端末のみが Captive Portal ポータル画面へ自動リダイレクトされます。

---

### 4. 実装問題

**問題**: FTD CLI にログインし、現在 FTD のメモリキャッシュ内に保持されている「IP アドレス ↔ ユーザー名」の対応テーブル（パッシブ/アクティブセッション一覧）を直接確認するための正確な CLI コマンドはどれか。

* A) <code>show access-list</code>
* B) <code>show user-identity user</code>
* C) <code>show running-config identity</code>
* D) <code>show service-policy</code>

**正解**: **B**  
**解説**: FTD CLI で **`show user-identity user`** を実行すると、管理下の FTD が保持している最新のユーザーバインディングテーブル（IP、ユーザー名、Realm、ドメイン、認証方式）を一括表示できます。

---

### 5. コンフィグ読解問題

**問題**: 未登録端末から `[suspicious link removed]` 宛てにアクセスを発生させた際、FTD の Captive Portal 認証画面へリダイレクトされず、ブラウザがタイムアウトエラーとなった。FMC 上では Identity Policy で Active Authentication が正しくアサインされていた。不足している設定として正しいものはどれか。

* A) 対象 HTTPS トラフィックを復号する SSL Inspection Policy（Decrypt - Resign）が設定・バインドされていない。
* B) FTD 上で BFD エコーが有効化されているため。
* C) Realm の Base DN が指定されていないため。
* D) FMC 上で Threat License が無効化されているため。

**正解**: **A**  
**解説**: HTTPS 通信（TCP 443）に対して Captive Portal リダイレクト（HTTP 302）を発行するには、FTD が TLS ハンドシェイクを復号・割り込み処理できる必要があります。**SSL Inspection Policy (Decrypt)** が設定されていない場合、暗号化ハンドシェイクを破綻させずにポータルへリダイレクトすることができません。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Device Configuration Guide - User Identity](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot ISE and FMC Integration via pxGrid](https://www.cisco.com/)
* [Cisco Technical Notes: Troubleshoot Identity Policies and Captive Portal on FTD](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Identity Firewall and pxGrid with Cisco Secure Firewall (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で「Configure FTD to transparently identify domain users via ISE pxGrid and prompt Captive Portal only for unmapped users accessing HTTP/HTTPS」と指示された場合の全体構築フロー：
    1. FMC に AD Realm を作成 ➔ ユーザー/グループの `User Download` を完了。
    2. ISE ↔ FMC 間で pxGrid ピアリングを構築 (ISE で Approve 忘れに注意)。
    3. Platform Settings で Captive Portal ポート (885) を有効化。
    4. Identity Policy を作成：Action = `Passive Authentication` + `Use active authentication if passive...` [✓]。
    5. SSL Policy で Outbound Decrypt を設定し、ACP で Identity Policy ＆ SSL Policy をバインドして Deploy。
* **図解（Identity 判定とポリシー評価フロー）**:
  ```text
  [ Ingress Packet Received ]
               │
               ▼
  [ Look up IP in FTD User Memory Cache ]
               │
               ├── Cached? ──► YES ──► [ Attach Username / Group Attributes ]
               └── NO (Unmapped)
                     │
                     ├── Identity Policy Rule = Active Auth (HTTP/HTTPS)?
                     │         ├── YES ──► [ Redirect to Captive Portal (Port 885) ]
                     │         └── NO  ──► [ Tag as "Unknown" / Special Identity ]
                     │
                     ▼
  [ Evaluate Access Control Policy (Users Tab Conditions) ]
               │
               ├── Match User/Group Rule ──► [ Allow / Apply IPS & File Inspection ]
               └── Match Default Action  ──► [ Block / Drop Packet ]
  ```
* **注意点**:
  * FMC と Active Directory サーバーおよび Cisco ISE 間の **NTP 時刻同期（Time Synchronization）** は絶対条件です。時刻が 5 分以上ズレていると、Kerberos 認証の拒否や pxGrid トークンエラーが発生し、パッシブ特定が完全に停止します。ラボ試験開始直後に各サーバーの NTP ステータスを必ず確認してください。

## 📝 **補足（Notes）**

*   **学習メモ**: 「アイデンティティ」は単独では機能せず、常に **Realm -> Identity Policy -> ACP** という三段構えの紐付けが必要です。このチェーンのどこが欠けてもユーザーベースの制御は失敗します。
*   **図解**: パケットが FTD を通るたびに、Snort が「こいつのIPは誰だ？」とメモリ内の地図（User-IP Map）を引いている様子をイメージしてください。
*   **注意点**: ラボ試験では AD との時刻同期（NTP）が非常に重要です。時刻がずれていると証明書エラーや Kerberos 認証の失敗により Realm 同期が停止します。
