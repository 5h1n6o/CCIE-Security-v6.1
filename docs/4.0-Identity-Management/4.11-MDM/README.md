---
layout: default
title: 4.11-MDM
nav_order: 11
parent: 4.0-Identity-Management
---

# 4.11 Integration of MDM with Cisco ISE

Cisco Identity Services Engine (ISE) と Mobile Device Management (MDM) / Enterprise Mobility Management (EMM) ソリューションの統合は、企業ネットワークにおける BYOD (Bring Your Own Device) および コーポレート所有モバイルデバイス（iOS、Android、macOS、Windows）のセキュリティ ガバナンスを確立するためのコア技術です。

---

## 📘 概要

Cisco ISE と MDM / EMM プラットフォーム（Cisco Meraki Systems Manager, VMware Workspace ONE / AirWatch, MobileIron, Microsoft Intune, Jamf Pro など）の連携は、ネットワークアクセス制御（RADIUS/802.1X/MAB）とエンドポイント管理システム（MDM）をリアルタイム API で接続する統合セキュリティ アーキテクチャです。

### 機能概要

* **MDM Onboarding Redirect (オンボーディングリダイレクト)**: ネットワークに接続した未登録モバイルデバイスを自動的に感知し、Cisco ISE の Web Redirection (MDM Redirect) 機能を用いて MDM ポータルへ誘導し、プロファイル・アプリインストールを強制します。
* **MDM Compliance Enforcement (ポスチャ・コンプライアンス検証)**: RADIUS 認証および認可のタイミングで、ISE が REST API を経由して MDM サーバーに問い合わせを入れ、デバイスの登録状態 (`DeviceRegisterStatus`) やコンプライアンス状態 (`DeviceComplianceStatus`)、脱獄有無 (`JailBrokenStatus`)、暗号化状態 (`DiskEncryptionStatus`)、PIN ロック設定 (`PinLockStatus`) を検証します。
* **Dynamic Access Control (動的アクセス制御)**: コンプライアンス違反が発生した場合、ISE は RADIUS CoA (Change of Authorization) を発行してネットワークアクセス権限を即座に制限（クアランティン VLAN や Dedicated Internet-Only dACL への移行）します。
* **Context Visibility & Remote Management Actions**: ISE UI (Context Visibility) 上から管理者操作により、MDM 経由で対象デバイスの PIN ロック (`PinLock`)、コーポレートワイプ (`Corporate Wipe`)、フルワイプ (`Full Wipe`)、リフレッシュ (`Refresh MDM Partner Endpoint`) を直接発動できます。

### 利用目的

1. **アクセス制限とモバイルガバナンスの融合**: 従来の 802.1X/MAB 単体では判別できない「OS の暗号化状態」や「脱獄（Jailbreak/Rooting）の有無」を網羅したゼロトラスト アクセス制御を実現します。
2. **自動化されたコンプライアンス維持**: ユーザーが接続中に MDM 側でコンプライアンス違反（例: 必修セキュリティアプリの削除やパスコード解除）を起こした際、即座にネットワークから遮断または制限付きアクセスへ降格させます。
3. **マルチベンダー MDM の一元管理**: Cisco 標準 MDM API により、異種 MDM プラットフォームが混在するマルチクラウド環境でも、統一された Policy Set 条件で管理できます。

### どのような場面で利用するか

* **BYOD (個人所有端末の業務利用)**: 従業員の個人端末に対し、企業リソースへのアクセス条件として MDM 登録とコンプライアンス遵守を強制する場合。
* **コーポレートモバイル端末のセキュリティ維持**: 役員や営業担当者が持ち歩くモバイル端末に対し、フルディスク暗号化や PIN ロックが有効であることを接続毎にチェックする場合。
* **MDM 未登録端末の隔離**: 社内 Wi-Fi に接続を試みる未登録端末を検知し、自動的に MDM 登録ポータルへリダイレクトしてプロビジョニングを完了させる場合。

---

## 🔑 要点

Cisco ISE と MDM 連携における主要項目と設計上のポイントを表形式で整理します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | Cisco 共通 MDM API による HTTPS/RESTful 双方向連携。リアルタイム API ルックアップおよび 4 時間毎のバルク同期。 |
| **用途** | モバイル端末のオンボーディングリダイレクト、コンプライアンス評価に基づく動的アクセス制御、遠隔クアランティン/ワイプ。 |
| **メリット** | AnyConnect Posture Agent を導入できない iOS/Android 端末でも高度なポスチャ評価が可能。ネットワーク遮断と MDM 制御の完全連動。 |
| **デメリット** | 外部 MDM サーバーの障害・通信遅延が 802.1X 認可処理に影響するリスク。API ポーリングによるタイムラグ。 |
| **対応 MDM** | Cisco Meraki SM, VMware Workspace ONE (AirWatch), MobileIron, Microsoft Intune, Jamf Pro, IBM MaaS360, Citrix XenMobile など。 |
| **制限事項** | モバイル OS の仕様変更（iOS 14+ Private MAC Address / Randomized MAC）により、MAC アドレスをキーとする MDM 照会が失敗する問題。 |
| **設計上の注意点** | `MDMServerReachable` 属性を活用した Fail-Open / Fail-Closed 設計。リダイレクト ACL における Google Play / Apple APNs / MDM FQDN のバイパス設定。 |

---

## 🏗 動作原理

Cisco ISE と MDM の連携は、ネットワークアクセス制御（RADIUS）とバックエンド API 連携（HTTPS REST API）の二層構造で動作します。

```text
+------------------+         +----------------------+         +---------------------+         +-------------------+
| Mobile Endpoint  |         | Network Access Device|         |  Cisco ISE (PSN)    |         | MDM/EMM Cloud     |
| (iOS / Android)  |         | (Switch / WLC)       |         |                     |         | (Meraki/AirWatch) |
+--------+---------+         +----------+-----------+         +----------+----------+         +---------+---------+
         |                              |                            |                              |
         | --- 1. Wi-Fi / 802.1X/MAB -->|                            |                              |
         |                              | --- 2. RADIUS Access-Req ->|                              |
         |                              |                            | --- 3. Rest API Query ------>|
         |                              |                            |     (Mac / Serial / UDID)    |
         |                              |                            |<--- 4. Return Attributes ----|
         |                              |                            |     (Status, Compliance)     |
         |                              |<-- 5. RADIUS Access-Accept-|                              |
         |                              |    (URL-Redirect + ACL)    |                              |
         |<-- 6. HTTP Web Traffic ------|                            |                              |
         | --- 7. HTTP Redirect ------------------------------------>|                              |
         |                              |                            | --- 8. Redirect to MDM ----->|
         | <================= 9. Complete MDM Enrollment / Compliance Update =====================> |
         |                              |                            |                              |
         |                              |                            |<=== 10. API Push Notification|
         |                              |<-- 11. RADIUS CoA-ReAuth --|     (Status: Compliant)      |
         | --- 12. Re-Authenticate ---->|                            |                              |
         |                              | --- 13. RADIUS Access-Req->|                              |
         |                              |<-- 14. RADIUS Access-Accept|                              |
         |                              |    (PermitAccess / dACL)   |                              |
```

### 1. ディクショナリ属性と情報モデル

ISE は MDM サーバーを追加設定すると、動的に `MDM` ディクショナリ属性を有効化します。認可ポリシー（Authorization Policy）では以下の主要属性を組み合わせて条件ルールを評価します。

* `MDM:DeviceRegisterStatus`: デバイスの MDM 登録状態 (`Registered` / `Unregistered`)
* `MDM:DeviceComplianceStatus`: MDM 側のセキュリティポリシー準拠状態 (`Compliant` / `NonCompliant`)
* `MDM:MDMServerReachable`: ISE から MDM サーバーへの API 到達性 (`Reachable` / `Unreachable`)
* `MDM:DiskEncryptionStatus`: 暗号化有効状態 (`On` / `Off`)
* `MDM:JailBrokenStatus`: 脱獄/ルート化検知 (`Unbroken` / `Broken`)
* `MDM:PinLockStatus`: 自動画面ロック設定 (`On` / `Off`)
* `MDM:MDMServerName`: 評価対象の MDM サーバー名

### 2. ポーリング (Polling) とリアルタイム問い合わせ

* **On-Demand Query (リアルタイム問い合わせ)**: エンドポイントがネットワーク認証（RADIUS Access-Request）を発生させた際、ISE はセッション情報に含まれる Calling-Station-ID (MAC アドレス) や証明書の SAN (Subject Alternative Name) をキーとして、MDM サーバーに API リクエストを送信します。
* **Bulk Download (定期バルク同期)**: デフォルトで 4 時間 (240 分) ごとに、ISE は MDM サーバーから全登録デバイスのステータスを一括取得し、内部データベースを同期します。定期同期中にコンプライアンス状態の変更が検出された場合、ISE は自動的に RADIUS CoA をトリガーします。

---

## ⚙ 動作シーケンス

パケット処理とデータフローの時系列シーケンスを詳細に解説します。

### フェーズ 1: 初回接続と MDM 未登録の検知

1. **接続試行**: モバイル端末が SSIDs に接続し、802.1X (EAP-TLS または PEAP) または MAB 認証を開始します。
2. **RADIUS 処理**: NAD (WLC / スイッチ) が ISE に RADIUS Access-Request を送信します。
3. **MDM API 問い合わせ**: ISE は MDM サーバーへ REST API 問い合わせを行い、対象デバイスが `DeviceRegisterStatus = Unregistered` であることを確認します。
4. **リダイレクト認可の返却**: ISE は認可プロファイル `MDM_Onboard` を適用し、NAD に対し `url-redirect-acl` および `url-redirect` (MDM Portal URL) を含む RADIUS Access-Accept を返却します。

### フェーズ 2: Web リダイレクトと MDM オンボーディング

1. **Web トラフィックの捕捉**: ユーザーが端末のブラウザを開いて HTTP トラフィックを発生させると、NAD の Redirect ACL にヒットし、ISE の MDM ポータルへリダイレクトされます。
2. **MDM アプリ / プロファイルへの誘導**: ISE MDM ポータルは、登録されている MDM サーバー (例: Meraki SM) の登録 URL / プロファイルダウンロードリンクへブラウザを自動リダイレクトします。
3. **MDM エージェントのインストールと登録**: 端末は Apple App Store / Google Play / MDM サーバーからプロファイルをインストールし、MDM への登録を完了します。この際、リダイレクト ACL では DNS (UDP 53) および Apple/Google/MDM の FQDN 通信が許可されている必要があります。

### フェーズ 3: コンプライアンス同期と RADIUS CoA によるアクセス昇格

1. **ステータス更新**: MDM サーバー上でデバイスが `Registered` かつ `Compliant` に変更されます。
2. **CoA 発行**: MDM から ISE への API 通知、または ISE の定期ポーリング / ユーザーアクションにより、ISE は NAD へ RADIUS CoA (CoA-ReAuth または CoA-Terminate) を送信します。
3. **再認証の実行**: NAD はセッション ID を維持したまま、エンドポイントに対して再認証（Re-Authentication）を実施します。
4. **フルアクセスの付与**: ISE は評価時に `MDM:DeviceRegisterStatus = Registered` かつ `MDM:DeviceComplianceStatus = Compliant` の条件にマッチし、`PermitAccess` や社内用 dACL / Dynamic VLAN / SGT を含む最終認可を返却します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、MDM 統合タスクは単なる GUI 設定にとどまらず、証明書トラスト、リダイレクト ACL 設計、API 認証、Policy Set の評価順序、Fail-Open 設定など多角的なトラブルシューティングと正しい実装能力が問われます。

### Blueprint で重要なポイント

* **信頼関係の構築 (CA Certificate Trust)**: ISE が HTTPS 経由で外部 MDM サーバー（例: `n180.meraki.com` や AirWatch FQDN）と通信するために、MDM サーバーのルート CA / 中間 CA 証明書を `Administration > System > Certificates > Trusted Certificates` に登録し、`Trust for authentication within ISE` を有効にする必要があります。
* **MDM サーバーの登録パラメータ**: `Administration > Network Resources > External MDM` での Hostname/IP, Port (443), API Key/Token, Basic Auth 認証情報、Polling Interval (4時間)、Time Interval for Compliance ReAuth の正確な設定。
* **認可ルールの評価順序 (Rule Order)**: MDM 評価ルールは、必ず「一般的な社内アクセスルール (PermitAccess)」よりも**上位**に配置しなければなりません。
  1. **MDM Redirect Rule**: `MDM:DeviceRegisterStatus = Unregistered` → Result: `MDM_Redirect_Profile`
  2. **MDM NonCompliant Quarantine Rule**: `MDM:DeviceComplianceStatus = NonCompliant` → Result: `Quarantine_Profile` (Internet Only / Deny)
  3. **MDM Compliant Rule**: `MDM:DeviceRegisterStatus = Registered` AND `MDM:DeviceComplianceStatus = Compliant` → Result: `PermitAccess`
* **NAD 側の リダイレクト ACL 設計**: IOS-XE スイッチや Catalyst 9800 WLC における Redirect ACL の記述論理。
  * リダイレクト**させない**通信（DNS、DHCP、ISE IP、MDM サーバー FQDN / Google Play / Apple APNs）を `deny`
  * リダイレクト**させる** HTTP/HTTPS トラフィックを `permit`

### ラボ試験で設定させられそうな内容

1. **特定 MDM サーバーの追加と API トークン設定**: Meraki SM や AirWatch サーバーを ISE に登録し、`Test Connection` をパスさせる。
2. **MDM リダイレクト用 Authorization Profile の作成**: `Access Type = ACCESS_ACCEPT`、`Web Redirection = MDM Redirect`、`ACL = ACL-MDM-REDIRECT`、`MDM Server = <MDM_Name>` を指定。
3. **Fail-Open / Fail-Closed ポリシーの実装**: 万が一 MDM サーバー障害や API 通信途絶が発生した場合に、全拒否にせず `MDM:MDMServerReachable = Unreachable` を検知して `Internet-Only` アクセスを付与するフォールバックルールの作成。
4. **Context Visibility からの遠隔操作要件**: 違反端末に対して Context Visibility 画面から `Corporate Wipe` または `PIN Lock` を発行するトラブルシュートタスク。

### よくある設定ミス

* **証明書トラストの漏れ**: MDM サーバーの SSL 証明書を発行した Root CA が ISE の Trusted Certificates に未登録で、API 接続テスト時に `Connection Failure` または `SSL Handshake Failed` となる。
* **リダイレクト ACL で MDM サーバー通信を拒否（Permit 漏れ）**: NAD の Redirect ACL で MDM サーバーへの FQDN / IP へのアクセスを `deny` (バイパス) し忘れたため、ユーザーが MDM ポータルにアクセスできずループが発生する。
* **MDM ディクショナリの非表示**: External MDM サーバーの設定が完了して有効化 (`Enabled`) されるまで、Policy Elements の Condition Editor 上で `MDM` ディクショナリが隠蔽されたままでルールが作成できない。
* **Policy Set 順序ミス**: 登録状態を評価する前にデフォルトの 802.1X 許可ルールにヒットしてしまい、未登録デバイスにフルアクセスが与えられてしまう。

---

## 🛠 設定方法

Cisco ISE GUI および NAD (Catalyst 9800 WLC / IOS-XE Switch) における標準的な設定手順を解説します。

### 1. MDM CA 証明書のインポート (ISE GUI)

1. `Administration > System > Certificates > Certificate Management > Trusted Certificates` に移動します。
2. `Import` をクリックし、MDM サーバーの Root / Intermediate CA 証明書ファイルを選択します。
3. `Friendly Name` に識別名（例: `Meraki_MDM_Root_CA`）を入力します。
4. `Trust for authentication within ISE` チェックボックスを有効にします。
5. `Submit` をクリックして保存します。

### 2. External MDM サーバーの追加 (ISE GUI)

1. `Administration > Network Resources > External MDM` に移動します。
2. `Add` をクリックします。
3. パラメータを設定します:
   * **Name**: `Meraki-MDM`
   * **Server Type**: `Mobile Device Manager`
   * **Authentication Type**: `Basic` (または OAuth)
   * **Host Name / IP Address**: `n180.meraki.com` (または専用 FQDN)
   * **Port**: `443`
   * **Username / Password**: MDM 管理画面から発行した API キー / トークン認証情報
   * **Polling Interval**: `240` ( minutes )
   * **Time Interval For Compliance Device ReAuth Query**: `1` ( minutes )
   * **Status**: `Enabled`
4. `Test Connection` をクリックし、`Connection Successful` を確認した後に `Save` をクリックします。

### 3. MDM Onboard 用 Authorization Profile の作成 (ISE GUI)

1. `Policy > Policy Elements > Results > Authorization > Authorization Profiles` に移動します。
2. `Add` をクリックします。
3. 名前を `MDM_Onboard_Profile` と指定します。
4. `Access Type`: `ACCESS_ACCEPT` を選択します。
5. `Common Tasks` で `Web Redirection (CWA, MDM, NSP, CPP)` をチェックします。
6. ドロップダウンから `MDM Redirect` を選択します。
7. **ACL**: NAD 側に定義する ACL 名（例: `ACL-MDM-REDIRECT`）を入力します。
8. **Value**: `MDM Portal (default)` を選択します。
9. **MDM Server**: 追加した `Meraki-MDM` を選択します。
10. `Save` をクリックします。

### 4. Authorization Policy (Policy Sets) の構成 (ISE GUI)

`Policy > Policy Sets` を開き、対象の Policy Set 内で以下の順序で Authorization Rule を作成します。

```text
Rule 1: MDM_NotReachable_Fallback
  Condition: MDM:MDMServerReachable EQUALS UnReachable
  Result: AuthorizationProfile -> Internet-Only

Rule 2: MDM_Onboarding_Redirect
  Condition: MDM:DeviceRegisterStatus EQUALS UnRegistered
  Result: AuthorizationProfile -> MDM_Onboard_Profile

Rule 3: MDM_NonCompliant_Quarantine
  Condition: MDM:DeviceComplianceStatus EQUALS NonCompliant
  Result: AuthorizationProfile -> DenyAccess (または Internet-Only)

Rule 4: MDM_Compliant_FullAccess
  Condition: MDM:DeviceRegisterStatus EQUALS Registered AND MDM:DeviceComplianceStatus EQUALS Compliant
  Result: AuthorizationProfile -> PermitAccess
```

### 5. NAD 側 (Catalyst 9800 WLC / Switch) の Redirect ACL コンフィグ

#### WLC / Catalyst 9800 リダイレクト ACL 設定例 (`ACL-MDM-REDIRECT`)

```bash
! DNS, DHCP, ISE, MDM 通信をバイパス (deny)
ip access-list extended ACL-MDM-REDIRECT
 10 deny udp any any eq domain
 20 deny udp any eq domain
 30 deny udp any any eq bootps
 40 deny udp any any eq bootpc
 50 deny ip any host 10.1.100.41      ! ISE PSN IP
 60 deny ip any host 204.8.168.0 0.0.0.255 ! MDM Subnet / FQDN IP
 ! HTTP / HTTPS Web トラフィックをリダイレクト (permit)
 70 permit tcp any any eq www
 80 permit tcp any any eq 443
!
```

#### Catalyst 9800 WLC の WLAN 設定 (AAA Override & ISE NAC 有効化)

```bash
wlan Corp-BYOD 1 Corp-BYOD
 aaa-override
 security ft
 security wpa psk set-key ascii 0 Cisco12345!
 security wpa akm ft-psk
 radio-policy all
 no shutdown
!
wlan-profile Corp-BYOD
 nac
!
```

---

## 🔍 検証コマンド

NAD (Catalyst Switch / WLC) および Cisco ISE CLI / Operations 画面で MDM 統合状態を確認・検証するための主要コマンドおよびビューです。

| 目的 | コマンド / 確認画面 |
| --- | --- |
| NAD セッション状態確認 | <code>show authentication sessions interface <int> details</code> |
| WLC クライアント状態確認 | <code>show wireless client mac-address <MAC> detail</code> |
| リダイレクト ACL ヒット確認 | <code>show ip access-lists ACL-MDM-REDIRECT</code> |
| ISE 認証・認可リアルタイムログ | ISE GUI: `Operations > RADIUS > Live Logs` |
| MDM API 通信・コンプライアンス状態 | ISE GUI: `Operations > Context Visibility > Endpoints (Compliance Tab)` |
| MDM API 接続ステータス確認 | ISE GUI: `Administration > Network Resources > External MDM` (`Test Connection`) |
| ISE デバッグログ有効化 | ISE CLI / GUI: `prt-server` および `mdm` コンポーネントの DEBUG レベル変更 |
| ISE ポスチャ/MDM ログファイル | ISE CLI: <code>show logging application prt-server.log</code> |

---

## 🚨 トラブルシュート

MDM 統合環境で発生しやすいトラブル症状、根本原因、確認方法、および解決策の一覧です。

| 症状 | 原因 | 確認コマンド / 画面 | 対処方法 |
| --- | --- | --- | --- |
| **`Test Connection` で MDM 接続テストが失敗する** | ISE 側に MDM サーバーの CA 証明書がインポートされていない、または API キー/トークンが不正。 | `External MDM` 画面の `Test Connection` エラー、<code>prt-server.log</code> | `Trusted Certificates` に MDM サーバーの Root/Intermediate CA を追加し `Trust for auth within ISE` をチェック。API Key を再発行。 |
| **未登録デバイスが MDM ポータルへリダイレクトされない** | NAD 側に Redirect ACL が存在しない、または 802.1X 許可ルールが上位にあり先にマッチしている。 | <code>show authentication sessions</code>, ISE Live Logs の Matched Rule | ISE Policy Set で `MDM:DeviceRegisterStatus = Unregistered` ルールを最上位付近に配置。NAD 側に同一名の Redirect ACL を作成。 |
| **MDM オンボーディング中にブラウザが接続エラー (DNS/IP 到達不能)** | Redirect ACL で DNS や App Store / MDM サーバーへの通信が `deny` (バイパス) されていない。 | <code>show ip access-lists ACL-MDM-REDIRECT</code> | Redirect ACL で DNS (UDP 53)、Apple APNs、Google Play、MDM FQDN への通信を `deny` (許可バイパス) に設定。 |
| **MDM 登録完了後もフルアクセスに切り替わらない (クアランティン継続)** | ISE から NAD への RADIUS CoA (UDP 1700) がドロップされている、または MDM ポーリング遅延。 | <code>show radius statistics</code>, ISE Live Logs の CoA 状態 | ISE と NAD 間の UDP 1700 通信（ファイアウォール含む）を許可。WLC 側で `Support for CoA` が有効化されているか確認。 |
| **`MDMServerReachable` が `Unreachable` となり接続拒否される** | MDM クラウドサービス側障害、または ISE のプロキシ/DNS 設定不備。 | ISE Live Logs の `MDMServerReachable` 属性値, ISE CLI: <code>ping <MDM_FQDN></code> | ISE Policy Set に `MDMServerReachable = Unreachable` 時の Fail-Open (Internet-Only) フォールバックルールを追加。 |
| **Context Visibility からの Corporate Wipe / Lock 操作が失敗する** | ISE と MDM サーバー間の ERS / REST API 権限不足。 | `Context Visibility > Endpoints > MDM Actions` | MDM サーバー側の API トークンアカウントにデバイス管理/ワイプ権限を付与。 |

---

## ⚠ 制限事項

* **iOS 14+ / Android 10+ Private MAC Address (ランダム MAC) 制限**: モバイル OS が Wi-Fi 接続ごとにランダム MAC アドレスを使用すると、ISE が RADIUS セッション MAC と MDM 側登録 MAC を紐付けられず、`DeviceRegisterStatus = Unregistered` のままリダイレクトループに陥るケースがあります。対策として MDM 側でランダム MAC を無効化する構成プロファイルを配布するか、証明書 (SAN/UPN) ベースで識別します。
* **API ポーリング間隔によるタイムラグ**: 定期バルク同期 (`Polling Interval`) の標準は 4 時間であり、MDM 側でコンプライアンス違反が検知されてから ISE がそれを認識して CoA を打つまでにタイムラグが生じる可能性があります（MDM からのリアルタイム webhook 推奨）。
* **ライセンス依存**: ISE 3.x において、MDM 統合機能を利用するには **ISE Premier License** (旧 Apex License) がエンドポイント数分必要となります。

---

## 🔄 他技術との関連

MDM 統合機能は、Cisco ISE の他のセキュリティコンポーネントおよびネットワークインフラと密接に連携します。

* **BYOD (Native Supplicant Provisioning)**: ISE 独自の EST/SCEP 証明書プロビジョニングと MDM プロビジョニングを連続的なステップとして組み合わせ、証明書発行と MDM エージェント導入を単一フローで完了させます。
* **CWA (Centralized Web Authentication)**: MDM リダイレクト機能は CWA と同一の URL-Redirection エンジンおよび RADIUS CoA メカニズムを共有します。
* **TrustSec (SGT)**: MDM コンプライアンス結果に基づき、準拠端末には `SGT 4 (Employees)`、違反端末には `SGT 15 (Quarantined_Systems)` を動的に割り当てて SGACL でマイクロセグメンテーションを実施します。
* **pxGrid (Platform Exchange Grid)**: MDM から取得したコンテキスト情報（端末モデル、OS バージョン、所有者、コンプライアンス状態）を、pxGrid を介して Cisco FMC (Firepower) や Stealthwatch (Secure Network Analytics) と共有します。

---

## 🧩 比較表

各種エンドポイントポスチャ・評価方式の比較です。

| 比較項目 | ISE MDM 統合ポスチャ | AnyConnect Posture Agent | Agentless Posture |
| --- | --- | --- | --- |
| **対象 OS** | iOS, Android, macOS, Windows | Windows, macOS, Linux | Windows (WMI/WinRM) |
| **エージェント** | MDM プロファイル / アプリ | AnyConnect ISE Posture Module | 不要 (エージェントレス) |
| **評価項目** | 暗号化, PIN ロック, 脱獄, 管理状態 | ファイル, レジストリ, サービス, AV/FW | WMI 属性, サービス状態 |
| **網羅性** | モバイル端末に最適 | PC/デスクトップに最適 | 簡易チェック |
| **ネットワーク接続要件**| MDM クラウド/サーバーへの API 通信 | ISE PSN への Port 8905 直通信 | ISE から PC への WMI 通信 |
| **ライセンス** | Premier (Apex) | Premier (Apex) | Advantage (Plus) |

---

## 💡 ベストプラクティス

1. **Fail-Open ポリシーの必携**: 外部 MDM クラウドサービスとの通信途絶時に社内業務が全停止しないよう、`MDMServerReachable = Unreachable` 条件を条件分岐の先頭に置き、一時的な `Internet-Only` アクセスを許可するポリシーを定義します。
2. **証明書 SAN (Subject Alternative Name) の活用**: 802.1X EAP-TLS 認証で使用するクライアント証明書の SAN フィールドに MAC アドレス (`SAN:otherName` または `SAN:DNS`) を埋め込むことで、OS のランダム MAC 環境下でも確実に MDM 登録情報と合致させます。
3. **DNS Snooping ACL の活用**: Catalyst 9800 や AireOS WLC において、MDM サーバーの動的 IP 変更に対応するため、`ACL-MDM-REDIRECT` で URL ベースの DNS スヌーピングバイパスを設定します。

---

## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルの実践的な設定演習 10 例です。各問題について要件、設定例（コンフィグおよび GUI 設定手順）、および検証方法を網羅しています。

---

### サンプル例 1: External MDM サーバー (Cisco Meraki SM) の登録と証明書信頼構成

#### 問題
Cisco ISE において、外部 MDM プラットフォームとして Cisco Meraki Systems Manager (`n180.meraki.com`) を追加し、API 連携を確立せよ。

#### 要件
* Meraki MDM の Root CA 証明書を Trusted Certificates に追加し、ISE 認証信頼を有効化すること。
* API 接続タイプは `Basic`、ポートは `443`、API トークンアカウント情報を使用して通信すること。
* 定期ポーリング間隔は `240` 分に設定し、`Test Connection` が成功することを確認すること。

#### 設定例

##### 1. CA 証明書のインポート (ISE GUI)
* `Administration > System > Certificates > Certificate Management > Trusted Certificates`
* `Import` -> `Meraki_Root_CA.crt` を選択
* `Friendly Name`: `Meraki_EMM_CA`
* `Trust for authentication within ISE`: チェックオン
* `Submit`

##### 2. MDM サーバーの追加 (ISE GUI)
* `Administration > Network Resources > External MDM` -> `Add`
* `Name`: `Meraki-MDM`
* `Server Type`: `Mobile Device Manager`
* `Authentication Type`: `Basic`
* `Host Name / IP Address`: `n180.meraki.com`
* `Port`: `443`
* `Username`: `0b49e939531a040eeedc93b8543736fb342230de` (API Token)
* `Password`: `********************`
* `Polling Interval`: `240`
* `Status`: `Enabled`
* `Test Connection` ボタンをクリック -> `Success` を確認後 `Save`

#### 検証方法
ISE GUI の `External MDM` 画面で `Meraki-MDM` の Status が `Enabled` であり、`Test Connection` で緑色の成功通知が表示されることを確認します。

---

### サンプル例 2: 未登録端末向け MDM Onboarding Redirect 認可プロファイルと ACL 設定

#### 問題
MDM 未登録端末が Wi-Fi (SSID: `Corp-BYOD`) に接続した際、MDM ポータルへ自動誘導する認可プロファイルおよび NAD 側リダイレクト ACL を構成せよ。

#### 要件
* Authorization Profile 名: `MDM_Onboard_Profile`
* Web Redirection: `MDM Redirect` を選択し、リダイレクト ACL に `ACL-MDM-REDIRECT` を指定すること。
* MDM サーバーとして `Meraki-MDM` を紐付けること。
* Catalyst スイッチおよび WLC 側に `ACL-MDM-REDIRECT` を定義し、DNS/DHCP/ISE/Meraki FQDN 通信をバイパスすること。

#### 設定例

##### 1. Authorization Profile 設定 (ISE GUI)
* `Policy > Policy Elements > Results > Authorization > Authorization Profiles` -> `Add`
* `Name`: `MDM_Onboard_Profile`
* `Access Type`: `ACCESS_ACCEPT`
* `Web Redirection (CWA, MDM, NSP, CPP)`: チェックオン
  * タイプ: `MDM Redirect`
  * ACL: `ACL-MDM-REDIRECT`
  * Value: `MDM Portal (default)`
  * MDM Server: `Meraki-MDM`
* `Save`

##### 2. NAD (Catalyst 9800 WLC) ACL コンフィグ

```bash
ip access-list extended ACL-MDM-REDIRECT
 10 deny udp any any eq domain
 20 deny udp any eq domain
 30 deny udp any any eq bootps
 40 deny udp any any eq bootpc
 50 deny ip any host 10.1.100.41
 60 deny ip any 204.8.168.0 0.0.0.255
 70 permit tcp any any eq www
 80 permit tcp any any eq 443
!
```

#### 検証方法
未登録端末接続時に `show authentication sessions interface <int> details` を実行し、`URL Redirect ACL: ACL-MDM-REDIRECT` および `URL Redirect: https://ISE:8443/mdmportal/...` が適用されていることを確認します。

---

### サンプル例 3: 802.1X EAP-TLS + MDM 登録・コンプライアンス順守によるフルアクセス許可ルール

#### 問題
EAP-TLS 証明書認証をパスし、かつ MDM 上で登録済みかつコンプライアンス準拠であるデバイスにのみフルアクセス (`PermitAccess`) を許可する Policy Set ルールを作成せよ。

#### 要件
* Rule 名: `MDM_Compliant_Permit`
* 認証方法: `EAP-TLS`
* ユーザーグループ: AD `Domain Users`
* MDM 条件: `MDM:DeviceRegisterStatus EQUALS Registered` AND `MDM:DeviceComplianceStatus EQUALS Compliant`
* 認可結果: `PermitAccess`

#### 設定例

##### ISE Policy Set 認可ルール構成 (ISE GUI)
* `Policy > Policy Sets > Corp_PolicySet > Authorization Policy`
* `Insert New Rule Above` -> Rule 名: `MDM_Compliant_Permit`
* **Conditions**:
  * `Network Access:AuthenticationMethod EQUALS x509_PKI`
  * `AND` `ad1:ExternalGroups EQUALS securitydemo.net/Users/Domain Users`
  * `AND` `MDM:DeviceRegisterStatus EQUALS Registered`
  * `AND` `MDM:DeviceComplianceStatus EQUALS Compliant`
* **Profiles**: `PermitAccess`
* `Save`

#### 検証方法
対象デバイスで Wi-Fi 接続後、ISE `Operations > RADIUS > Live Logs` で該当セッションの Authorization Rule が `MDM_Compliant_Permit` にヒットし、`PermitAccess` が適用されていることを確認します。

---

### サンプル例 4: MDM サーバー障害に備えた Fail-Open フォールバックルールの構成

#### 問題
MDM クラウドサーバーが通信不能（障害やメンテナンス）になった際、社員の業務停止を防ぐため、一時的にインターネット接続のみを許可する Fail-Open ルールを最優先で適用せよ。

#### 要件
* Rule 名: `MDM_Server_Unreachable_Fallback`
* 評価位置: MDM 関連ルールの最上位
* 条件: `MDM:MDMServerReachable EQUALS UnReachable` AND AD `Domain Users`
* 認可結果: `Internet_Only_Profile` (社内セグメント遮断 dACL 適用)

#### 設定例

##### ISE Policy Set 認可ルール構成 (ISE GUI)
* `Policy > Policy Sets > Corp_PolicySet > Authorization Policy`
* `Insert New Rule Top` -> Rule 名: `MDM_Server_Unreachable_Fallback`
* **Conditions**:
  * `MDM:MDMServerReachable EQUALS UnReachable`
  * `AND` `ad1:ExternalGroups EQUALS securitydemo.net/Users/Domain Users`
* **Profiles**: `Internet_Only_Profile`
* `Save`

#### 検証方法
ISE から MDM サーバーの通信を遮断（または External MDM 画面でダミー IP に変更）し、端末再接続時に Live Logs で `MDMServerReachable = Unreachable` にヒットして `Internet_Only_Profile` が適用されることを確認します。

---

### サンプル例 5: 詳細ポスチャ属性 (Jailbreak / Disk Encryption / PIN Lock) に基づく微細制御

#### 問題
単なる `Compliant` ステータスだけでなく、具体的に「脱獄なし」「ディスク暗号化 ON」「PIN ロック ON」の 3 条件を個別に明示検証する認可ルールを構築せよ。

#### 要件
* Rule 名: `MDM_Detailed_Security_Check`
* 条件:
  * `MDM:DeviceRegisterStatus EQUALS Registered`
  * `MDM:JailBrokenStatus EQUALS Unbroken`
  * `MDM:DiskEncryptionStatus EQUALS On`
  * `MDM:PinLockStatus EQUALS On`
  * `MDM:DeviceComplianceStatus EQUALS Compliant`
* 認可結果: `PermitAccess` + `SGT_Employee` (Security Group Tag 4)

#### 設定例

##### 1. Condition Library 登録 (ISE GUI)
* `Policy > Policy Elements > Conditions > Library Conditions` -> `Add`
* `Name`: `MDM_Strict_Security_Condition`
* Attributes (AND 結合):
  * `MDM:DeviceRegisterStatus = Registered`
  * `MDM:JailBrokenStatus = Unbroken`
  * `MDM:DiskEncryptionStatus = On`
  * `MDM:PinLockStatus = On`
  * `MDM:DeviceComplianceStatus = Compliant`
* `Save`

##### 2. Authorization Policy への適用
* Authorization Rule `MDM_Detailed_Security_Check` に上記 `MDM_Strict_Security_Condition` を指定し、Result に `PermitAccess` および Security Group `Employees` を割当。

#### 検証方法
ISE Live Logs の `Authentication Details` セクションで、評価された属性値 `MDM:JailBrokenStatus = Unbroken`, `MDM:DiskEncryptionStatus = On`, `MDM:PinLockStatus = On` がすべて Passed であることを確認します。

---

### サンプル例 6: Catalyst 9800 WLC における DNS スヌーピング付き MDM Redirect ACL の設定

#### 問題
Catalyst 9800 WLC において、動的に IP が変化する Meraki MDM サーバー (`*.meraki.com`) への通信を確実にバイパスするための DNS Snooping 付き ACL を構成せよ。

#### 要件
* ACL 名: `ACL-MDM-REDIRECT`
* DNS (UDP 53) および FQDN `*.meraki.com` への通信をリダイレクトから除外 (`deny`) すること。
* Web トラフィック (TCP 80/443) をリダイレクトに対象 (`permit`) とすること。

#### 設定例

##### Catalyst 9800 WLC CLI コンフィグ

```bash
! URL リストの定義
parameter-map type webauth MDM_URL_MAP
 type regex
 pattern .*\.meraki\.com
 pattern .*\.apple\.com
!
! リダイレクト ACL の作成
ip access-list extended ACL-MDM-REDIRECT
 10 deny udp any any eq domain
 20 deny udp any eq domain
 30 deny tcp any any-host parameter-map MDM_URL_MAP
 40 permit tcp any any eq www
 50 permit tcp any any eq 443
!
```

#### 検証方法
WLC 上で <code>show ip access-lists ACL-MDM-REDIRECT</code> を実行し、クライアントが `*.meraki.com` にアクセスした際に `deny` エントリのカウンタが増加し、Web トラフィックのみが ISE へリダイレクトされていることを確認します。

---

### サンプル例 7: Catalyst 3850/9300 スイッチ IBNS 2.0 MDM リダイレクト設定

#### 問題
IOS-XE Catalyst スイッチの有線ポートにおいて、ISE からの MDM リダイレクト認可（`url-redirect` / `url-redirect-acl`）を受け入れるための IBNS 2.0 C3PL コンフィグを投入せよ。

#### 要件
* 有線ポート FastEthernet1/0/5 に 802.1X および MAB を有効化。
* RADIUS CoA (UDP 1700) を受信可能に設定。
* スイッチローカルに `ACL-MDM-REDIRECT` を定義。

#### 設定例

##### Catalyst スイッチ CLI コンフィグ

```bash
! RADIUS / CoA 設定
aaa new-model
aaa authentication dot1x default group radius
aaa authorization network default group radius
aaa accounting update periodic 5
aaa server radius dynamic-author
 client 10.1.100.41 server-key Cisco123!
!
! リダイレクト ACL
ip access-list extended ACL-MDM-REDIRECT
 deny   udp any any eq domain
 deny   udp any any eq bootps
 deny   ip any host 10.1.100.41
 permit tcp any any eq www
 permit tcp any any eq 443
!
! インターフェース設定 (IBNS 2.0 / C3PL)
interface FastEthernet1/0/5
 switchport mode access
 switchport access vlan 10
 authentication host-mode multi-auth
 authentication order dot1x mab
 authentication priority dot1x mab
 authentication port-control auto
 mab
 dot1x pae authenticator
!
```

#### 検証方法
未登録端末接続時に <code>show authentication sessions interface FastEthernet1/0/5 details</code> を実行し、`ACS ACL` または `URL Redirect ACL: ACL-MDM-REDIRECT` が動的に適用されていることを検証します。

---

### サンプル例 8: Context Visibility からの遠隔 Corporate Wipe & RADIUS CoA 発行

#### 問題
紛失または規約違反が発覚したモバイル端末 `48:f8:b3:2e:24:23` に対し、ISE GUI から遠隔ワイプを指示し、ネットワークから即座に切断せよ。

#### 要件
* `Context Visibility > Endpoints` の `Compliance` タブから対象端末を選択。
* `MDM Actions > Corporate Wipe` を実行すること。
* コールバックにより ISE から NAD へ RADIUS CoA (`CoA-Terminate` または `CoA-ReAuth`) を自動送信させること。

#### 設定例 (ISE GUI 操作手順)

1. `Operations > Context Visibility > Endpoints` に移動します。
2. `Compliance` タブを開きます。
3. クライアント MAC アドレス `48:f8:b3:2e:24:23` のチェックボックスを選択します。
4. 上部アクションバーの `MDM Actions` ドロップダウンをクリックします。
5. `Corporate Wipe` を選択します。
6. 確認ダイアログで `Yes` をクリックします。
7. ISE が MDM API を呼び出してワイプ命令を送信し、同時に NAD に対して RADIUS CoA Session Terminate を発行します。

#### 検証方法
ISE Live Logs にて `CoA Type: Dynamic Authorization RADIUS Session Terminate` が `Success` で記録され、NAD 側の `show authentication sessions` から該当セッションが削除されることを確認します。

---

### サンプル例 9: マルチベンダー MDM (AirWatch + Intune) 優先度別認可ルール

#### 問題
社内で併用されている VMware AirWatch と Microsoft Intune の 2 つの MDM サーバーに対し、MDM サーバー名ごとに個別の認可プロファイルと制限ポリシーを割り当てよ。

#### 要件
* AirWatch 登録端末 (`MDM:MDMServerName EQUALS AirWatch-Server`): フルアクセス付与。
* Intune 登録端末 (`MDM:MDMServerName EQUALS Intune-Server`): Restricted アクセス付与。
* いずれの MDM にも未登録の端末: `MDM_Onboard_Profile` へリダイレクト。

#### 設定例

##### ISE Policy Set 認可ルール構成 (ISE GUI)

```text
Rule 1: AirWatch_Compliant_Permit
  Condition: MDM:MDMServerName EQUALS AirWatch-Server 
             AND MDM:DeviceRegisterStatus EQUALS Registered 
             AND MDM:DeviceComplianceStatus EQUALS Compliant
  Result: PermitAccess

Rule 2: Intune_Compliant_Restricted
  Condition: MDM:MDMServerName EQUALS Intune-Server 
             AND MDM:DeviceRegisterStatus EQUALS Registered 
             AND MDM:DeviceComplianceStatus EQUALS Compliant
  Result: Restricted_Access_Profile

Rule 3: MDM_Unregistered_Redirect
  Condition: MDM:DeviceRegisterStatus EQUALS UnRegistered
  Result: MDM_Onboard_Profile
```

#### 検証方法
AirWatch 端末と Intune 端末でそれぞれ接続テストを行い、Live Logs で意図した MDMServerName に対応する Authorization Rule および Profile がヒットすることを確認します。

---

### サンプル例 10: MDM API タイムアウト・認証エラーのトラブルシューティング

#### 問題
MDM 統合環境において、ユーザー接続時に ISE Live Logs で `MDM Server Unreachable` エラーが頻発し、認証が失敗する障害を調査・復旧せよ。

#### 要件
* ISE CLI から MDM サーバーへのネットワーク到達性と SSL ハンドシェイクを検証すること。
* `prt-server.log` から根本原因（証明書不一致 / API トークン期限切れ）を特定し、修正すること。

#### 設定例 & トラブルシュート手順

##### 1. ISE CLI からのネットワーク・SSL 到達性確認

```bash
# DNS 解決と Port 443 到達性の確認
tech-support ping n180.meraki.com
tech-support traceroute n180.meraki.com

# ISE アプリケーションログの追尾
show logging application prt-server.log tail
```

##### 2. ログ解析と原因特定
`prt-server.log` 内に以下のエラーメッセージを確認:
`ERROR [mdm-api-thread]: SSLHandshakeException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target`

##### 3. 修正措置
* MDM サーバーの証明書チェーンが更新されていたため、新しい中間 CA 証明書を ISE の `Trusted Certificates` に追加インポートし、`Trust for authentication within ISE` をチェックオンにして保存。
* `External MDM` 設定画面で `Test Connection` を再実行し `Success` を確認。

#### 検証方法
`Test Connection` の成功後、クライアントを再接続し、Live Logs で `MDMServerReachable = Reachable` となり正常にアクセス認可が行われることを確認します。

---

## ❓ 想定試験問題

CCIE Security ラボ試験および記述試験を想定した実践的なクイズ・演習問題です。

---

### 質問 1 (コンフィグ読解・評価順序)

Cisco ISE の Policy Set において、以下の認可ルールが上から順に定義されている。未登録の新しい iOS デバイスが 802.1X EAP-TLS 認証（AD 認証成功、有効な証明書保持）で接続した際、どの認可ルールにヒットするか？

```text
Rule A (Employee_Access): 
  If Network Access:AuthenticationMethod EQUALS x509_PKI 
  AND AD:ExternalGroups EQUALS Domain Users 
  Then PermitAccess

Rule B (MDM_Onboarding): 
  If Network Access:AuthenticationMethod EQUALS x509_PKI 
  AND MDM:DeviceRegisterStatus EQUALS UnRegistered 
  Then MDM_Onboard_Profile
```

* **A)** Rule B にヒットし、`MDM_Onboard_Profile` が適用される。
* **B)** Rule A にヒットし、`PermitAccess` が適用される。
* **C)** いずれにもヒットせず Default Rule が適用される。
* **D)** 認証エラーとなり接続が拒否される。

#### 正解
**B**

#### 解説
ISE の Authorization Policy は上から順にトップダウン評価 (`First-Match`) されます。未登録端末であっても EAP-TLS と AD グループ条件を満たしているため、より上位に配置されている Rule A に最初にマッチしてしまい、MDM 未登録状態のチェック（Rule B）が行われずに `PermitAccess` が付与されてしまいます。未登録デバイスを正しくリダイレクトするには、`MDM_Onboarding` ルールを `Employee_Access` ルールよりも**上位**に配置する必要があります。

---

### 質問 2 (トラブルシューティング・Redirect ACL)

Catalyst 9800 WLC 環境で ISE MDM リダイレクトを設定した。未登録の Android 端末が接続した際、ユーザーのブラウザが MDM オンボーディングポータル画面に到達できず、「接続タイムアウト」エラーとなる。WLC に設定されている以下の Redirect ACL を確認した。根本原因として正しいものはどれか？

```bash
ip access-list extended ACL-MDM-REDIRECT
 10 deny udp any any eq bootps
 20 deny ip any host 10.1.100.41
 30 permit tcp any any eq www
 40 permit tcp any any eq 443
```

* **A)** DHCP トラフィックを `deny` しているため IP アドレスが取得できていない。
* **B)** ISE サーバー (`10.1.100.41`) への通信を `deny` しているためリダイレクトできない。
* **C)** DNS トラフィック (UDP 53) に対する `deny` ルールが欠落しており、DNS 解決自体がリダイレクトされて名前解決に失敗している。
* **D)** TCP 80/443 を `permit` しているため全 Web トラフィックが遮断されている。

#### 正解
**C**

#### 解説
NAD の Redirect ACL において、`permit` は「リダイレクト対象（ISE の Portal へ送る）」、`deny` は「リダイレクト除外（直接通過させる）」を意味します。端末が FQDN で Web ブラウジングを開始する際、まず DNS 解決を行う必要がありますが、DNS (UDP 53) を除外するルール (`deny udp any any eq domain`) が定義されていないため、DNS 要求自体が ISE へリダイレクトされ、端末が IP 名解決に失敗してタイムアウトとなります。

---

### 質問 3 (設計・Fail-Open 仕様)

ISE と 外部 MDM サーバーの連携設計において、クラウド MDM サーバーへの API 通信が不可視（障害またはネットワーク分断）になった場合でも、既に接続中の準拠端末の通信を即座に遮断せず、セキュリティを維持しつつフォールバックさせるための最善の設計手法はどれか？

* **A)** ISE の MDM 統合設定で `Status` を `Disabled` に変更する。
* **B)** Authorization Policy の先頭に `MDM:MDMServerReachable EQUALS UnReachable` 条件のルールを作成し、`Internet-Only` などの制限付きプロファイルを適用する。
* **C)** NAD 側の RADIUS サーバータイムアウト値を 0 に設定する。
* **D)** ISE の Global Profiler 設定で `No CoA` を選択する。

#### 正解
**B**

#### 解説
ISE の MDM ディクショナリには、API の到達性を保持する属性 `MDM:MDMServerReachable` (`Reachable` / `Unreachable`) が用意されています。外部 MDM サーバー障害時にこの属性が `Unreachable` と評価されるため、Policy Set の上位に `MDMServerReachable EQUALS UnReachable` を条件とするフォールバックルールを配置し、`Internet-Only` プロファイルを割り当てることで、Fail-Closed (完全全拒否) による全業務停止を回避し、安全な Fail-Open / Fallback 運用を実現できます。

---

### 質問 4 (マルチベンダー・属性)

Cisco ISE の MDM ディクショナリ属性において、MDM ベンダー（Meraki SM, AirWatch, Intune 等）を問わず、端末が MDM サーバーの定めるセキュリティポリシー（パスコード、暗号化、脱獄等の総合評価）に適合しているか否かを示すマクロレベルの属性名はどれか？

* **A)** `MDM:JailBrokenStatus`
* **B)** `MDM:DeviceRegisterStatus`
* **C)** `MDM:DeviceComplianceStatus`
* **D)** `MDM:DiskEncryptionStatus`

#### 正解
**C**

#### 解説
`MDM:DeviceComplianceStatus` は、各 MDM サーバーが独自に評価したデバイス全体のコンプライアンス適合結果（`Compliant` または `NonCompliant`）を ISE に引き渡すための共通マクロ属性です。個別の設定項目（`JailBrokenStatus` や `DiskEncryptionStatus` 等）を一つずつルールに書かなくても、この属性一つで簡潔にポスチャ準拠判定を行うことができます。

---

### 質問 5 (実装・証明書信頼)

ISE と 外部 MDM サーバーを HTTPS REST API で接続する際、ISE GUI の `External MDM` 設定で `Test Connection` を実行したところ SSL ハンドシェイクエラーが発生した。この問題を解決するために最初に行うべき作業はどれか？

* **A)** ISE の Admin 証明書を再発行する。
* **B)** MDM サーバーの Root / Intermediate CA 証明書を ISE の `Trusted Certificates` にインポートし、`Trust for authentication within ISE` を有効化する。
* **C)** NAD 側で `ip radius source-interface` を変更する。
* **D)** ISE CLI で設定初期化コマンドを実行する。

#### 正解
**B**

#### 解説
ISE が外部 MDM サーバーと HTTPS (Port 443) 経由で REST API 通信を行う際、相手方の SSL/TLS 証明書を検証します。MDM サーバーの証明書チェーンを発行した CA 証明書が ISE の `Trusted Certificates` にインポートされておらず、かつ認証信頼がチェックされていない場合、PKIX パス構築エラー（SSL ハンドシェイク失敗）となります。

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよびテクニカルリソースの一覧です。

* **Cisco ISE Administration Guide**: Mobile Device Management Integration & External MDM Setup Guide
* **Cisco Meraki Systems Manager Configuration Guide**: Integrating Meraki SM with Cisco ISE
* **Cisco Technical Support & Documentation**: Configuring Cisco ISE 3.x Integration with Microsoft Intune
* **Cisco Live Session BRKSEC-2044**: Enterprise Mobility & BYOD Integration with Cisco ISE Architecture
* **Cisco Validated Design (CVD)**: Campus Network Security Design and Implementation Guide with Identity Services Engine

---

## 📝 補足（Notes）

* **学習メモ**: CCIE Security ラボ試験では、MDM 連携は BYOD (NSP) や RADIUS CoA、SGT 割り当てと複合的に出題されるケースが多いため、単一のルール設定だけでなく、セッション全体の評価フローと RADIUS 属性の遷移を頭の中でシミュレーションできるようにトレーニングしてください。
* **図解の要点**:
  ```text
  [Endpoint] --(RADIUS)--> [NAD] --(RADIUS)--> [Cisco ISE] --(REST API/HTTPS 443)--> [MDM Cloud/Server]
     |                      ^                      |
     +--- (Web Redirect) ---+--- (RADIUS CoA) -----+
  ```
* **注意点**: 試験環境の MDM サーバーはダミーの API 応答を返すシミュレータとして実装されている場合があるため、`Test Connection` の結果メッセージおよび ISE `prt-server.log` のエラー出力パターンを正確に読み取れるように準備しておくことが重要です。
