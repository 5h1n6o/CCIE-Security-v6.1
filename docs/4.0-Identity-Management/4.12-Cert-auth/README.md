---
layout: default
title: 4.12-Cert-auth
nav_order: 12
parent: 4.0-Identity-Management
---

# 4.12 Certification-based authentication using Cisco ISE

## 📘 概要

証明書ベース認証（Certification-based authentication）は、Cisco Identity Services Engine（ISE）を中心としたネットワークアクセス制御において、最も堅牢で信頼性の高い認証メカニズムです。パスワードベース認証（EAP-PEAP/MSCHAPv2など）で懸念されるパスワードの盗聴・ブルートフォース攻撃・中間者攻撃（Man-in-the-Middle: MitM）のリスクを根本的に排除し、X.509 デジタル証明書を用いた相互認証（Mutual Authentication）を提供します。

### 主な利用目的と適用場面
* **エンタープライズ有線・無線LANアクセス制御 (IEEE 802.1X / EAP-TLS)**: 企業所有のPC（Windows, macOS）およびモバイル端末に対して、暗号強度が高くユーザ操作を必要としないシームレスなネットワークアクセスを提供します。
* **Zero Trust ネットワークアクセス (ZTNA) / BYOD デバイスオンボーディング**: SCEP（Simple Certificate Enrollment Protocol）や EST（Enrollment over Secure Transport）と連携し、個人所有端末や企業端末に個別のクライアント証明書を動的に発行・自動配置してアクセス制御を行います。
* **Remote Access VPN (Cisco AnyConnect / Secure Client)**: Cisco ASA / Firepower Threat Defense (FTD) 経由でのリモート接続時に、証明書認証（または証明書＋パスワードの二要素認証）を実施し、許可された正規端末のみを社内ネットワークに接続させます。
* **pxGrid 連携および ISE ノード間通信 (ISE Cube)**: Policy Administration Node (PAN)、Policy Services Node (PSN)、Monitoring Node (MnT) 間の内部通信や、Cisco DNA Center (Catalyst Center) / Cisco FMC との Context Sharing（pxGrid）における厳格な相互認証に暗号証明書が利用されます。

---

## 🔑 要点

証明書ベース認証の主要な要点と実装設計における要件を下表に整理します。

| 項目 | 内容 |
| --- | --- |
| **主要対応プロトコル** | Native EAP-TLS、Tunneled EAP (PEAP-TLS, TEAP with EAP-TLS, EAP-FAST with EAP-TLS) |
| **主要証明書規格** | X.509 v3 デジタル証明書（RSA 2048/4096 bit、ECDSA P-256/P-384） |
| **ISE 認証オブジェクト** | Certificate Authentication Profile (CAP)、Trusted Certificate Store |
| **識別名抽出 (Use Identity From)** | Subject Common Name (CN)、Subject Alternative Name (SAN: UPN, Email, DNS, Directory Name) |
| **外部 Identity Store 比較** | Active Directory / LDAP バイナリ比較 (Match Client Certificate Against Certificate in Identity Store)、または属性ルックアップのみ（No Binary Comparison） |
| **失効検証メカニズム** | OCSP (Online Certificate Status Protocol: TCP/HTTP 2560)、CRL (Certificate Revocation List: HTTP/LDAP) |
| **必要信頼設定 (Trust Flags)** | `Trust for client authentication and Syslog`、`Trust for authentication within ISE` |
| **主な制限事項** | 端末側でのルート/中間CA証明書の事前配置が必要。失効検証サーバ（OCSP/CRL）へのネットワーク到達性が不可欠。 |
| **設計上の注意点** | AD側の `userCertificate` 属性を用いたバイナリ比較を実施する場合、ドメインコントローラへの通信負荷とレイテンシの増加に留意すること。 |

---

## 🏗 動作原理

証明書ベース認証（EAP-TLS）の基本構造は、サプリカント（端末）、アセンティケータ（スイッチ/WLC/ASA）、および認証サーバ（Cisco ISE）の3者間で展開される相互TLSハンドシェイクに基づきます。

### 全体アーキテクチャと通信フロー

```
+------------------+         +------------------+         +------------------+         +--------------------+
| Client Supplicant|         | Authenticator    |         | RADIUS Server    |         | External Identity  |
| (PC / Mobile)    |         | (Switch / WLC)   |         | (Cisco ISE PSN)  |         | (AD / LDAP / OCSP) |
+------------------+         +------------------+         +------------------+         +--------------------+
         |                            |                            |                            |
         |------ EAP-Start ---------->|                            |                            |
         |<----- EAP-Request/ID ------|                            |                            |
         |------ EAP-Response/ID ---->|                            |                            |
         |                            |--- RADIUS Access-Req(ID) ->|                            |
         |                            |<-- RADIUS Access-Chal(TLS)-|                            |
         |<----- EAP-Req(TLS Start)---|                            |                            |
         |                            |                            |                            |
         |====================== TLS Handshake Start ======================|                            |
         |------ Client Hello ------------------------------------>|                            |
         |<----- Server Hello, Server Certificate, Cert Request ---|                            |
         |------ Client Certificate, Certificate Verify ------------>|                            |
         |       [Finished]                                        |                            |
         |                                                         |--- 1. CA Signature Check -|
         |                                                         |--- 2. Date Validity Check -|
         |                                                         |--- 3. OCSP/CRL Revocation->|
         |                                                         |<-- Revocation Status Good -|
         |                                                         |--- 4. CAP Identity Extract-|
         |                                                         |--- 5. AD/LDAP AuthZ Lookup>|
         |                                                         |<-- Group Membership Data---|
         |<----- TLS Finished -------------------------------------|                            |
         |====================== TLS Tunnel Established ===================|                            |
         |                            |                            |                            |
         |                            |<-- RADIUS Access-Accept ---|                            |
         |                            |    (VLAN/dACL/SGT/CoA)     |                            |
         |<----- EAP-Success ---------|                            |                            |
         |                            |                            |                            |
```

### 認証・認可判定の5段階評価ステップ
1. **Issuer Signature Validation**: クライアント証明書を発行したCAのパブリックキーが、ISEの Trusted Certificate Store に登録されており、`Trust for client authentication` フラグが有効化されているかを検証。
2. **Validity Period Check**: 現在のシステム時刻が、証明書の `Valid From` から `Valid To` の有効期限内であることを確認。
3. **Revocation Check**: CRL（証明書失効リスト）または OCSP Responder を呼び出し、証明書シリアル番号が失効していないかを確認。
4. **Proof of Possession (Digital Signature Verification)**: TLS ハンドシェイク中にクライアントが送信した `Certificate Verify` メッセージ内のデジタル署名をクライアント証明書の公開鍵で検証し、クライアントが秘密鍵を所持していることを証明。
5. **CAP Mapping & Authorization Lookup**: Certificate Authentication Profile (CAP) に従い、証明書の Subject CN や SAN(UPN) から Principal Identity を抽出し、Active Directory または Internal Identity Store に対してグループ属性・認可ポリシーの検索を実施。

---

## ⚙ 動作シーケンス

EAP-TLS による証明書ベース認証がネットワーク上で処理される詳細なパケット処理シーケンスを順を追って説明します。

1. **802.1X セッション確立フェーズ**:
   * 端末がリンクアップすると、スイッチ/WLC は `EAP-Request/Identity` を送信。
   * 端末は `EAP-Response/Identity`（例: `host/pc1.securitydemo.net` または `user1@securitydemo.net`）を返信。
   * スイッチ/WLC はこれを RADIUS `Access-Request` パケットにカプセル化して ISE PSN に転送。

2. **TLS トンネル要求・サーバ証明書提示フェーズ**:
   * ISE は `EAP-TLS Start` を返答。
   * 端末は `Client Hello`（サポートする暗号スイート、TLSバージョン、ランダム値）を送信。
   * ISE は `Server Hello` と共に自身の ISE EAP Server Certificate（システム証明書）および Intermediate CA チェーン、そして `Certificate Request` パケットを端末に提示。
   * 端末は ISE のサーバ証明書が自身の信頼するルートCAによって署名されているか、および FQDN/SAN が一致するかを検証。

3. **クライアント証明書提示・署名検証フェーズ**:
   * 端末は自身の `Client Certificate`、およびハンドシェイクデータのハッシュ値に対して秘密鍵で署名した `Certificate Verify` メッセージを ISE に送信。
   * ISE は提示されたクライアント証明書を受信し、Trusted Certificate Store と照合。

4. **失効チェック (CRL / OCSP) 処理シーケンス**:
   * ISE は設定された OCSP サービス（ポート TCP/2560 など）にリクエストを発行。
   * OCSP から `Good` が返却されれば検証成功。OCSP が利用不可で CRL が設定されている場合、ISE は CRL キャッシュからシリアル番号を検索（未ロードの場合は HTTP/LDAP で CRL を動的取得）。

5. **CAP 属性抽出および Identity Mapping**:
   * ISE は設定された Certificate Authentication Profile (CAP) を参照。
   * `Use Identity From` 設定に基づき、証明書の `Subject - Common Name` や `Subject Alternative Name - Principal Name` から文字列を抽出。
   * 抽出された Identity（例: `user1`）を用いて Identity Source Sequence (ISS) 内の Active Directory または Internal Database に対して認可ルックアップを実行。

6. **EAP-Success および 認可ポリシープッシュ**:
   * ISE は認可ルールにマッチしたプロファイル（dACL、Dynamic VLAN、SGT 属性など）を含む RADIUS `Access-Accept` パケットをスイッチ/WLC に返信。
   * スイッチ/WLC はポートのブロックを解除し、端末に `EAP-Success` を送信して通信を許可。

---


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、証明書ベース認証（4.12）は単独で出題されるだけでなく、802.1X (4.4)、BYOD (4.6)、Posture (4.9)、pxGrid (4.10/3.10) など多岐にわたるセクションの前提条件として組み込まれます。

### Blueprintで重要なポイント
* **Trusted Certificates Store の属性設定**: 外部CA（Microsoft AD CSなど）のルート/中間CA証明書を ISE にインポートする際、単にインポートするだけでなく `Trust for client authentication and Syslog` のチェックボックスを確実に有効化すること。これが漏れると EAP-TLS 認証は即座に `12514 EAP-TLS failed SSL/TLS handshake because the client certificate is not trusted` エラーで失敗します。
* **Certificate Authentication Profile (CAP) の設計**:
  * **Identity Principal Extraction**: ユーザ認証の場合、`Subject - Common Name` または `SAN - User Principal Name` のどちらから Identity を抽出するか。マシン認証の場合、`Subject - Common Name`（端末ホスト名 `HOST/pc1.securitydemo.net`）から抽出する設計が求められます。
  * **Match Client Certificate Against Certificate in Identity Store**: デフォルトは `Never`。AD上の `userCertificate` 属性とのバイナリ比較が指示されていない限り `Never` のままに維持します（`Always perform binary comparison` にすると不要な AD クエリが発生しタイムアウトの原因となります）。
* **Identity Source Sequences (ISS) への組み込み**:
  * 証明書認証では、認証ソースとして CAP（例: `Preloaded_Certificate_Profile`）を指定しますが、認可フェーズで Active Directory のグループ情報（例: `Domain Users`や `Domain Computers`）を使用する場合、ISS の Identity Store List に CAP と AD の両方を登録するか、認証ポリシーで CAP を指定しつつ認可ポリシーで AD 属性を直接参照できるように構成する必要があります。
* **Certificate Revocation Check (CRL / OCSP) の例外処理**:
  * ラボ環境において外部 OCSP / CRL サーバへの到達性が制限されている場合、Revocation Check が有効になっていると認証が拒否されます。試験要件で「CRL/OCSPが利用できない場合でも認証を継続せよ」とある場合は、`Bypass CRL Verification if CRL is not Received` をチェックするか、Revocation Check を `Disable` に設定します。
* **Supplicant (Windows / AnyConnect) の証明書検証要件**:
  * クライアント側（Windows Supplicant）で「サーバ証明書を検証する」が有効な場合、ISE の EAP 提示証明書がクライアントの信頼する Root CA から発行されているか、および Trusted Server Name（例: `ise01.securitydemo.net`）が一致している必要があります。

---

## 🛠 設定方法

証明書ベース認証を ISE、Cisco Catalyst スイッチ、および Cisco ASA / WLC で構築するための手順と設定例です。

### 1. Cisco ISE 上での設定

#### ステップ 1: ルート/中間 CA 証明書のインポート
1. ISE GUI で **Administration > System > Certificates > Trusted Certificates** に移動。
2. **Import** をクリックし、外部 CA 証明書（例: `AD_Root_CA.cer`）を選択。
3. **Usage** タブで以下をチェック:
   * `Trust for authentication within ISE`
   * `Trust for client authentication and Syslog`
   * `Trust for certificate-based admin authentication` (必要に応じて)
4. **Submit** をクリック。

#### ステップ 2: Certificate Authentication Profile (CAP) の作成
1. **Administration > Identity Management > External Identity Sources > Certificate Authentication Profile** に移動。
2. **Add** をクリックし、新しいプロファイルを作成（例: `CAP_AD_UPN`）。
3. 主なパラメータ設定:
   * **Name**: `CAP_AD_UPN`
   * **Use Identity From**: `Subject Alternative Name` > `User Principal Name` （または `Subject - Common Name`）
   * **Identity Store**: `AD-SEC DEMO` （必要に応じて指定）
   * **Match Client Certificate Against Certificate in Identity Store**: `Never` （バイナリ比較不要時）
4. **Submit** をクリック。

#### ステップ 3: Certificate Status Validation (CRL / OCSP) の調整
1. **Administration > System > Certificates > Trusted Certificates** で対象の CA 証明書を選択して **Edit**。
2. ページ下部の **Certificate Status Validation** セクションで設定:
   * **OCSP Configuration**: 必要に応じて `Validate against OCSP Service` をチェック。
   * **CRL Configuration**: `Download CRL` をチェックし、CRL Distribution URL を設定。
   * ラボ試験等で失効サーバが不在の場合は `Bypass CRL Verification if CRL is not Received` を有効化。

#### ステップ 4: Policy Sets（認証ルール・認可ルール）の設定
1. **Policy > Policy Sets** に移動し、対象の Policy Set（例: `Wired_EAP-TLS_Policy`）を選択。
2. **Authentication Policy**:
   * **Rule Name**: `EAP-TLS_Auth`
   * **Condition**: `Network Access:EapAuthentication EQUALS EAP-TLS`
   * **Use**: `CAP_AD_UPN` (作成した CAP プロファイル)
3. **Authorization Policy**:
   * **Rule Name**: `Corporate_Domain_Users`
   * **Conditions**: `AD-SECDEMO:ExternalGroups EQUALS securitydemo.net/Users/Domain Users` AND `CERTIFICATE:Issuer - Common Name EQUALS AD-Root-CA`
   * **Profiles**: `PermitAccess`

---

### 2. Cisco Catalyst スイッチ（IOS-XE IBNS 2.0 / C3PL）コンフィグ例

```bash
! RADIUS サーバおよび AAA 設定
aaa new-model
aaa group server radius ISE_GROUP
 server-private 10.1.100.21 auth-port 1812 acct-port 1813 key Cisco123!
 ip radius source-interface GigabitEthernet0/0
!
aaa authentication dot1x default group ISE_GROUP
aaa authorization network default group ISE_GROUP
aaa accounting dot1x default start-stop group ISE_GROUP
!
radius-server attribute 6 on-for-login-auth
radius-server attribute 8 include-in-access-req
radius-server attribute 25 access-accept include
radius-server attribute 31 send mac-format ietf upper-case
!
! C3PL 802.1X ポート設定
policy-map type control subscriber DOT1X_POLICY
 event session-started match-all
  10 class always do-until-failure
   10 authenticate using dot1x priority 10
 event authentication-failure match-all
  10 class always do-until-failure
   10 terminate dot1x
   20 authorize format default
!
interface GigabitEthernet1/0/10
 description 802.1X EAP-TLS Client Port
 switchport mode access
 switchport access vlan 10
 access-session port-control auto
 access-session host-mode multi-auth
 service-policy type control DOT1X_POLICY
 dot1x pae authenticator
 spanning-tree portfast
```

---

## 🔍 検証コマンド

証明書ベース認証の正常性確認およびトラブルシューティングに使用する主要コマンドです。

| 目的 | コマンド / 確認場所 | 期待される出力 / 確認ポイント |
| --- | --- | --- |
| **スイッチ側セッション確認** | <code>show access-session interface G1/0/10 details</code> | `Status: Authz Success`, `Method: dot1x`, `Authenticator: Dot1x`, `User-Name` に証明書抽出識別子が表示されていること |
| **802.1X 統計情報確認** | <code>show dot1x statistics interface G1/0/10</code> | `EAP-TLS` パケットの正常送受信カウント、Tx/Rx EAPOL フレーム数 |
| **ISE 認証ライブログ確認** | **Operations > RADIUS > Live Logs** | `Authentication Result: Success`, `Authentication Protocol: EAP-TLS`, `Identity Store: CAP_AD_UPN` |
| **ISE 信任証明書ストア確認** | **Administration > System > Certificates > Trusted Certificates** | 対象 Root/Intermediate CA の Status が `Enabled` かつ `Trust for client authentication` がチェックされていること |
| **ISE リアルタイムデバッグログ** | <code>show logging application ise-psc.log tail</code> | `EAP-TLS handshake succeeded`, `Certificate validation passed` |
| **IOS-XE トラブルシューティング** | <code>debug dot1x packets / debug radius verbose</code> | TLS ハンドシェイク中の EAPOL-Payload 交換（Client Hello, Server Hello, Certificate）の遷移確認 |

---

## 🚨 トラブルシュート

証明書ベース認証で発生する代表的な障害症状、原因、確認方法および解決策を下表にまとめます。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **ISE ライブログに Failure Code `12514` (Client cert not trusted) が記録される** | ISE の Trusted Certificate Store にクライアント証明書の発行元 CA（Root/Intermediate）がインポートされていない、または `Trust for client authentication` が無効。 | ISE Live Logs: Failure Code `12514`, **Administration > Trusted Certificates** | 発行元 CA 証明書をインポートし、`Trust for client authentication` のチェックボックスを有効化する。 |
| **ISE ライブログに Failure Code `12520` (Certificate expired / not valid yet) が記録される** | クライアント証明書または ISE 自身の証明書の有効期限が切れている、あるいは ISE/端末の時刻（NTP）がズレている。 | <code>show ntp status</code>, ISE Live Logs: Failure Code `12520` | ISE および端末の NTP 同期状態を確認し、期限切れ証明書を再発行・更新する。 |
| **ISE ライブログに Failure Code `12509` / `12517` (CRL/OCSP check failed) が記録される** | OCSP Responder または CRL Distribution Point URL に ISE から到達できない、または失効サーバがダウンしている。 | ISE Live Logs: `CRL/OCSP server unreachable`, <code>tech-support</code> | ISE から CRL/OCSP サーバ（HTTP/LDAP ポート）へのネットワーク導通を確認する。緊急時は `Bypass CRL Verification` を有効化。 |
| **端末側で「証明書が無効です / サーバを確認できません」と表示され認証が停止する** | クライアントサプリカント側で ISE の EAP 提示証明書のルート CA が「信頼されたルート証明書機関」に登録されていない。 | Windows イベントビューア (`CAPI2`, `WLAN-AutoConfig`), AnyConnect Logs | ISE の EAP 提示証明書を発行した CA のパブリック証明書を、端末の「信頼されたルート証明書機関」ストアに配布・インポートする。 |
| **認証は成功するが認可ポリシーで失敗しデフォルト拒否される** | CAP で抽出した Identity 属性（CN など）が Active Directory のオブジェクト名と一致せず、AD グループ属性が取得できていない。 | ISE Live Logs > Authentication Details > `Identity Selection` & `Attributes` | CAP の `Use Identity From` 設定を `Subject - Common Name` から `SAN - User Principal Name` など実際の AD 属性に合わせて修正する。 |

---


## ⚠ 制限事項

* **サプリカントおよびOSの制限事項**:
  * **Microsoft Windows Native Supplicant**: ワイルドカード証明書（例: `*.securitydemo.net`）を EAP サーバ証明書として使用する場合、Windows の標準 802.1X サプリカントは安全上の理由から接続を拒否します。ISE の EAP 証明書には明示的な FQDN（例: `ise01.securitydemo.net`）を指定し、Subject Alternative Name (SAN) に必要な FQDN を列挙する必要があります。
* **バージョン依存およびライセンス要件**:
  * **Cisco ISE 3.x ライセンスモデル**: 証明書ベースの EAP-TLS 認証自体は **Premier / Advantage / Essentials** ライセンス（または旧 Base ライセンス）で動作しますが、証明書を伴う BYOD 自動オンボーディング（ISE Internal CA による SCEP 発行）を実施する場合は **Premier / Advantage**（旧 Apex）ライセンスが必要となります。
  * **TEAP (Tunnel Extensible Authentication Protocol - RFC 7170)**: Windows 10 (2004以降) および Cisco ISE 2.7 以降でサポート。EAP-TLS を Inner Method として使用した EAP Chaining（マシン証明書とユーザ証明書の同時検証）には TEAP が必須です。
* **PKI・ネットワークインフラ制限**:
  * **OCSP Timeout / Fail-Closed**: OCSP リプライのタイムアウト値（デフォルト5秒）および `Reject the request if OCSP Responder is unreachable` 設定が有効な場合、ネットワーク障害で OCSP にアクセスできないと全 EAP-TLS ユーザーが遮断されます。

---

## 🔄 他技術との関連

証明書ベース認証は、他の CCIE Security Blueprint 項目と以下のように深く関連しています。

* **Cisco Identity Services Engine (ISE) Policy Sets (4.1/4.4)**:
  * 認証ポリシーにおいて CAP（Certificate Authentication Profile）を選択することで、後続の認可ポリシーで証明書属性（Issuer CN, Key Length, Extended Key Usage）および AD グループ属性を組み合わせてアクセス権限を判定します。
* **BYOD Self-Service Onboarding & SCEP/EST (4.6)**:
  * クライアントが未登録状態（MAB / Open WebAuth）で接続した際、ISE の BYOD ポータル経由で SCEP/EST プロトコルを呼び出し、端末上で鍵ペアを生成させてクライアント証明書を自動自動発行・配置します。
* **Cisco AnyConnect / Secure Client Remote Access VPN (2.1/4.8)**:
  * AnyConnect VPN パケットは ASA/FTD を通過する際、EAP-TLS または Machine/User Certificate 認証を用いて二要素認証を実施します。
* **Cisco TrustSec & Security Group Tag (SGT) (2.6)**:
  * 証明書認証によりクライアントの役割（例: `Corporate-Laptop` や `Executive-User`）が確定すると、ISE はそのセッションに対して動的に SGT（例: SGT 10）を割り当て、スイッチの SGACL や ASA/FTD の SGT ベースフィルタリングを適用します。

---

## 🧩 比較表

### 1. EAP 認証プロトコルの比較表

| 比較項目 | Native EAP-TLS | PEAP-MSCHAPv2 | TEAP (EAP-TLS Inner) |
| --- | --- | --- | --- |
| **認証方式** | X.509 デジタル証明書による相互認証 | サーバ証明書 ＋ ユーザ名/パスワード | サーバ証明書 ＋ マシン/ユーザ証明書（EAP Chaining） |
| **セキュリティ強度** | 極めて高い（MitM/フィッシング不可） | 中程度（MitM/MSCHAPv2ハッシュ解読リスクあり） | 最高（トンネル保護＋相互証明書認証＋Chaining） |
| **クライアント証明書** | **必要** | 不要 | **必要**（マシン・ユーザ両方） |
| **資格情報変更対応** | パスワード概念がないため変更不要 | ADパスワード有効期限切れ変更対応可能 | トンネル内で認証制御 |
| **運用の複雑さ** | PKI（CA/SCEP/証明書配布）基盤が必要 | AD連携のみで導入が容易 | PKIおよび特定OS/サプリカント設定が必要 |

### 2. 証明書失効チェック方式の比較（CRL vs OCSP）

| 比較項目 | CRL (Certificate Revocation List) | OCSP (Online Certificate Status Protocol) |
| --- | --- | --- |
| **通信メカニズム** | ISE が CA から失効リストファイル（.crl）を定期取得・キャッシュ | ISE が OCSP Responder にリアルタイム HTTP クエリを送信 |
| **帯域および処理負荷** | 失効リストが巨大化するとファイル転送・解析負荷が増大 | リクエスト単位の軽量 HTTP 伝送で帯域効率が高い |
| **失効情報のリアルタイム性** | 定期更新（数時間〜数日単位）のため遅延あり | リアルタイム（即座に失効状態を反映可能） |
| **推奨運用形態** | 大規模環境での二次フォールバックとして利用 | 高セキュリティ環境およびプライマリ検証方式として推薦 |

---

## 💡 ベストプラクティス

1. **二重証明書の分離運用（EAP vs Portal vs Admin）**:
   * ISE のシステム証明書において、同一の証明書を Admin GUI, EAP 認証, Guest Portal で使い回す（Multi-Use）ことは避け、用途ごとに個別の証明書を発行・バインドすることが推奨されます。特に EAP 証明書には明示的な FQDN と適切な EKU (`Server Authentication`) を割り当ててください。
2. **SAN (Subject Alternative Name) の完全定義**:
   * EAP サーバ証明書およびクライアント証明書には、必ず Common Name (CN) と同じ値を `SAN:DNS` に含めてください（RFC 6125 準拠）。また、マルチノード展開（PSN クラスタ）では、各 PSN の個別 FQDN を SAN に追加配置します。
3. **CRL/OCSP のハイブリッド構成と Fail-Safe 設計**:
   * ISE の Trusted CA 設定では、OCSP と CRL の両方を有効化し、優先度を `OCSP First` に設定します。さらに、ネットワーク障害時に認証全体がダウンするのを防ぐため、環境のセキュリティポリシーに応じて `Bypass CRL Verification if CRL is not Received` を適切に評価・適用してください。
4. **Active Directory 連携時の `userCertificate` バイナリ比較の最適化**:
   * AD 連携時の CAP プロファイルでは、`Match Client Certificate Against Certificate in Identity Store` を原則 `Never` に設定します。AD 側でアカウントが無効化された場合のアクセス拒否は、ISE が自動的に行う「AD Courtesy Check（アカウント状態確認）」によって担保されるため、重いバイナリ比較を行わなくても十分なセキュリティが維持されます。

---


## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルの実践的な設定サンプル例を10個作成しました。各設定例は省略せず最後まで完全に記述しています。

---

### サンプル 1: Catalyst Switch における EAP-TLS 単体認証と VLAN/dACL 動的割当

#### 問題
スイッチ `SW1` のポート `Gi1/0/1` に接続する PC に対して、ISE を用いた 802.1X EAP-TLS 認証を構成し、認証成功時に VLAN 10 および dACL を動的割り当ててください。

#### 要件
* 認証プロトコルは EAP-TLS のみ許可すること。
* 認証成功時、ISE は dACL `PERMIT_CORP_ACL` (IP全許可) を返却すること。
* スイッチ側は IBNS 2.0 (C3PL) 構文を使用すること。

#### 設定例

**SW1 (Cisco IOS-XE) コンフィグ:**
```bash
aaa new-model
aaa group server radius ISE_RADIUS
 server-private 10.1.100.21 auth-port 1812 acct-port 1813 key Cisco123!
 ip radius source-interface GigabitEthernet0/0
!
aaa authentication dot1x default group ISE_RADIUS
aaa authorization network default group ISE_RADIUS
aaa accounting dot1x default start-stop group ISE_RADIUS
!
radius-server attribute 6 on-for-login-auth
radius-server attribute 8 include-in-access-req
radius-server attribute 25 access-accept include
radius-server attribute 31 send mac-format ietf upper-case
!
ip device tracking
!
policy-map type control subscriber EAP_TLS_POLICY
 event session-started match-all
  10 class always do-until-failure
   10 authenticate using dot1x priority 10
 event authentication-failure match-all
  10 class always do-until-failure
   10 terminate dot1x
   20 authorize format default
!
interface GigabitEthernet1/0/1
 description 802.1X EAP-TLS Client Port
 switchport mode access
 switchport access vlan 99
 access-session port-control auto
 access-session host-mode single-host
 service-policy type control EAP_TLS_POLICY
 dot1x pae authenticator
 spanning-tree portfast
```

**ISE 設定内容:**
* **Allowed Protocols**: `EAP-TLS-Only`（EAP-TLS のみチェック、他をオフ）。
* **CAP Profile**: `Preloaded_Certificate_Profile` (`Subject - Common Name` 抽出)。
* **Authorization Profile**: `Corp_Access` -> VLAN `10`, DACL Name `PERMIT_CORP_ACL` (`ip permit ip any any`)。

#### 検証方法
1. スイッチ上でセッション状態を確認:
   ```bash
   show access-session interface GigabitEthernet1/0/1 details
   ```
   * `Status: Authz Success`, `Method: dot1x`, `VLAN: 10`, `Applied Server-Policies: DACL: PERMIT_CORP_ACL` を確認。
2. ISE ライブログで `Authentication Protocol: EAP-TLS` およびマッチした認可ルールを確認。

---

### サンプル 2: Cisco WLC (Catalyst 9800) における EAP-TLS 無線アクセスと SGT 割当

#### 問題
Catalyst 9800 WLC において、SSID `Corporate-WLAN` に対する 802.1X EAP-TLS 認証を設定し、ISE から SGT（Security Group Tag）`15 (Employees)` を動的割り当ててください。

#### 要件
* RADIUS AAA サーバとして ISE (10.1.100.21) を登録。
* WLAN は WPA2-Enterprise (802.1X) を使用。
* ISE は証明書検証後に SGT 15 を返却。

#### 設定例

**Catalyst 9800 WLC コンフィグ:**
```bash
aaa group server radius ISE_WLAN_GROUP
 server-private 10.1.100.21 auth-port 1812 acct-port 1813 key Cisco123!
 ip radius source-interface VLAN100
!
aaa authentication dot1x EAP_TLS_WLAN_AUTH group ISE_WLAN_GROUP
aaa authorization network EAP_TLS_WLAN_AUTHZ group ISE_WLAN_GROUP
!
wlan Corporate-WLAN 1 Corporate-WLAN
 security ft
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security wpa akm dot1x
 authentication-list EAP_TLS_WLAN_AUTH
 authorization-list EAP_TLS_WLAN_AUTHZ
 no shutdown
!
wireless tag policy CORP_POLICY_TAG
 wlan Corporate-WLAN policy CORP_POLICY
!
wireless profile policy CORP_POLICY
 aaa-override
 cts inline-tagging
 vlan 20
 no shutdown
```

**ISE 認可プロファイル設定:**
* **Authorization Profile**: `Corp_Wireless_Profile` -> Advanced Attributes: `Cisco:cisco-av-pair = cts:security-group-tag=000f-15` (SGT 15)。

#### 検証方法
1. WLC 上でクライアントの接続状態と SGT 割当を確認:
   ```bash
   show wireless client mac-address aaaa.bbbb.cccc detail
   ```
   * `Security Group Tag: 15`, `VLAN ID: 20` を確認。

---

### サンプル 3: CAP における SAN (UPN) 抽出と Active Directory グループ認可

#### 問題
クライアント証明書の `Subject Alternative Name (SAN)` フィールドに格納された `User Principal Name (UPN)`（例: `user1@securitydemo.net`）を抽出し、Active Directory の `Domain Users` グループに基づいて認可を行う CAP プロファイルを ISE に設定してください。

#### 要件
* CAP プロファイル名: `CAP_SAN_UPN`
* 抽出フィールド: `Subject Alternative Name -> User Principal Name`
* AD 連携: `AD-SECDEMO` ジョインポイントを使用。

#### 設定例

**ISE GUI 設定手順:**
1. **Administration > Identity Management > External Identity Sources > Certificate Authentication Profile**:
   * **Name**: `CAP_SAN_UPN`
   * **Use Identity From**: `Subject Alternative Name` -> `User Principal Name`
   * **Match Client Certificate Against Certificate in Identity Store**: `Never`
   * **Submit** をクリック。
2. **Policy > Policy Elements > Conditions > Authorization > Compound Conditions**:
   * **Name**: `Cond_AD_Domain_Users`
   * **Condition**: `AD-SECDEMO:ExternalGroups EQUALS securitydemo.net/Users/Domain Users`
3. **Policy > Policy Sets > Standard Policy Set > Authorization Policy**:
   * **Rule**: `AD_User_Cert_Rule`
   * **Condition**: `Cond_AD_Domain_Users` AND `Network Access:AuthenticationIdentityStore EQUALS CAP_SAN_UPN`
   * **Permissions**: `PermitAccess`

#### 検証方法
1. 端末から EAP-TLS 接続を実行。
2. ISE Live Logs で `Identity Unscoped: user1@securitydemo.net` および `Matched Rule: AD_User_Cert_Rule` を確認。

---

### サンプル 4: Active Directory `userCertificate` 属性を用いた証明書バイナリ比較

#### 問題
高度なセキュリティが要求される端末に対して、提示されたクライアント証明書が Active Directory のユーザーオブジェクト属性（`userCertificate`）に登録されているバイナリデータと完全一致する場合のみアクセスを許可する CAP を設定してください。

#### 要件
* CAP プロファイル名: `CAP_Binary_Match`
* バイナリ比較モード: `Always perform binary comparison`
* Identity Store: `Active Directory (AD-SECDEMO)`

#### 設定例

**ISE GUI 設定手順:**
1. **Administration > Identity Management > External Identity Sources > Certificate Authentication Profile**:
   * **Name**: `CAP_Binary_Match`
   * **Identity Store**: `AD-SECDEMO`
   * **Use Identity From**: `Subject - Common Name`
   * **Match Client Certificate Against Certificate in Identity Store**: `Always perform binary comparison`
   * **Submit** をクリック。
2. **Policy > Policy Sets > Authentication Policy**:
   * **Use**: `CAP_Binary_Match`

#### 検証方法
1. AD 上の該当ユーザーオブジェクトの `userCertificate` 属性に証明書が登録されていることを確認。
2. 接続実行時、ISE ライブログの Details で `Binary Certificate Comparison Output: Passed` を確認。

---

### サンプル 5: ISE Internal CA を用いた AnyConnect VPN 用 EAP-TLS 証明書認証

#### 問題
Cisco ASA 経由の AnyConnect Remote Access VPN 接続において、ISE の Internal CA から発行された証明書を用いた EAP-TLS 認証を構成してください。

#### 要件
* ASA 側の Tunnel Group 名: `EAP_TLS_VPN`
* 認証方式: RADIUS 経由の EAP-TLS
* ISE 側で Client Provisioning によりAnyConnect 用証明書が発行済みであること。

#### 設定例

**Cisco ASA コンフィグ:**
```bash
tunnel-group EAP_TLS_VPN type remote-access
tunnel-group EAP_TLS_VPN general-attributes
 authentication-server-group ISE_RADIUS
 default-group-policy EAP_TLS_GP
tunnel-group EAP_TLS_VPN webvpn-attributes
 authentication aaa certificate
!
group-policy EAP_TLS_GP internal
group-policy EAP_TLS_GP attributes
 vpn-tunnel-protocol ssl-client
 split-tunnel-policy tunnelall
```

**ISE GUI 設定手順:**
1. **Administration > System > Certificates > Certificate Authority > Certificate Authority Certificates**:
   * ISE Root CA および Sub CA が `Enabled` であることを確認。
2. **Policy > Policy Sets**:
   * RADIUS `Service-Type EQUALS Framed (2)` かつ `NAS-Port-Type EQUALS Virtual (5)` の条件で AnyConnect 用認証・認可ルールを設定。

#### 検証方法
1. AnyConnect クライアントから接続し、ASA 上で確認:
   ```bash
   show vpn-sessiondb detail anyconnect
   ```
   * `Username: user1`, `Authentication: Certificate` を確認。

---


### サンプル 6: EAP-TLS 認証における OCSP/CRL 失効検証と Fail-Safe（Bypass）制御

#### 問題
ISE がクライアント証明書の検証を行う際、最初に OCSP Responder へ問い合わせ、応答がない場合は CRL を確認し、双方にアクセスできない場合でも認証を通過させる（Bypass）設定を行ってください。

#### 要件
* OCSP サービス名: `Corporate_OCSP` (URL: `http://ocsp.example.com:2560`)
* CRL 設定: `Bypass CRL Verification if CRL is not Received` を有効化。

#### 設定例

**ISE GUI 設定手順:**
1. **Administration > System > Certificates > Certificate Management > OCSP Client Profile**:
   * **Name**: `Corporate_OCSP`
   * **URL**: `http://ocsp.example.com:2560`
   * **Submit** をクリック。
2. **Administration > System > Certificates > Trusted Certificates**:
   * 対象の Root CA 証明書を選択して **Edit**。
   * **Certificate Status Validation**:
     * `Validate against OCSP Service` をチェックし、`Corporate_OCSP` を選択。
     * `Download CRL` をチェック。
     * `Bypass CRL Verification if CRL is not Received` をチェック。
   * **Save** をクリック。

#### 検証方法
1. OCSP サーバを停止させた状態で EAP-TLS 認証を実行。
2. ISE ライブログで `OCSP Status: Unreachable` および `CRL Check Bypassed` にもかかわらず `Authentication Result: Success` となることを確認。

---

### サンプル 7: TEAP (EAP-TLS Inner Method) を用いたマシン認証＋ユーザ認証の EAP Chaining

#### 問題
Windows 端末に対して TEAP (Tunnel EAP) プロトコルを使用し、アウター tunnel 内でマシン証明書とユーザ証明書の両方を EAP-TLS で認証する EAP Chaining ポリシーを ISE に作成してください。

#### 要件
* Allowed Protocols プロファイル: `TEAP_EAP-TLS_Chaining`
* 認可条件: マシン認証とユーザ認証の双方が成功した場合にフルアクセスを付与。

#### 設定例

**ISE GUI 設定手順:**
1. **Policy > Policy Elements > Results > Authentication > Allowed Protocols**:
   * **Name**: `TEAP_EAP-TLS_Chaining`
   * `Allow TEAP` を有効化。
   * Inner EAP Methods: `Allow EAP-TLS` をチェック。
   * `Enable EAP Chaining` をチェック。
2. **Policy > Policy Sets > Authorization Policy**:
   * **Rule**: `Both_Machine_And_User_Authenticated`
   * **Condition**: `Network Access:EapChainingResult EQUALS User and machine authenticated`
   * **Permissions**: `PermitAccess`
   * **Rule**: `Machine_Only_Authenticated`
   * **Condition**: `Network Access:EapChainingResult EQUALS Machine only authenticated`
   * **Permissions**: `Restricted_Access`

#### 検証方法
1. Windows 端末ログイン前（マシン認証のみ）の状態でスイッチのセッションを確認し、`Restricted_Access` の割り当てを確認。
2. ユーザーログイン後、セッションが自動再認証され `Both_Machine_And_User_Authenticated` (Full Access) に昇格することを確認。

---

### サンプル 8: Identity Source Sequence (ISS) を用いた EAP-TLS 証明書認証とローカルフォールバック

#### 問題
クライアント証明書による認証をプライマリとし、証明書の抽出 ID が Active Directory に存在しない場合に内部データベース（Internal Users）を参照する Identity Source Sequence (ISS) を構築してください。

#### 要件
* ISS 名: `ISS_Cert_Then_Internal`
* 認証ソース順序: 1. `CAP_AD_UPN`, 2. `Internal Users`
* 例外処理: `If User Not Found -> Continue`

#### 設定例

**ISE GUI 設定手順:**
1. **Administration > Identity Management > Identity Source Sequences**:
   * **Name**: `ISS_Cert_Then_Internal`
   * **Certificate Authentication Profile**: `CAP_AD_UPN` を選択。
   * **Authentication Search List**: `AD-SECDEMO`, `Internal Users` を配置。
   * **Advanced Search List Settings**:
     * `If User Not Found`: `Continue`
     * `If Process Error`: `Drop`
   * **Save** をクリック。
2. **Policy > Policy Sets > Authentication Policy**:
   * **Use**: `ISS_Cert_Then_Internal`

#### 検証方法
1. AD 未登録の証明書（Internal Users に存在するユーザー）で接続を試行し、ステップ2の `Internal Users` で認証が成功することを確認。

---

### サンプル 9: ASA Remote Access VPN における証明書マップ (Certificate Map) と ISE EAP-TLS 連携

#### 問題
Cisco ASA において、クライアント証明書の Issuer CN に `Corporate CA` が含まれる接続要求のみを特定の Tunnel Group `CORP_VPN` にマッピングし、ISE で EAP-TLS 認証を実施してください。

#### 要件
* ASA 側で Certificate Map `MAP_CORP_CA` を定義。
* `MAP_CORP_CA` を用いて Tunnel Group を動的選択。

#### 設定例

**Cisco ASA コンフィグ:**
```bash
crypto ca certmap MAP_CORP_CA 10
 issuer-name co corporate ca
!
webvpn
 certificate-group-map MAP_CORP_CA 10 CORP_VPN
!
tunnel-group CORP_VPN type remote-access
tunnel-group CORP_VPN general-attributes
 authentication-server-group ISE_RADIUS
 default-group-policy CORP_VPN_GP
tunnel-group CORP_VPN webvpn-attributes
 authentication aaa certificate
```

#### 検証方法
1. クライアント証明書を提示して VPN 接続を開始。
2. ASA 上で `show webvpn certificate-group-map` を実行し、正しく `CORP_VPN` トンネルグループにマッピングされたことを確認。

---

### サンプル 10: EAP-TLS 認証成功時における RADIUS Change of Authorization (CoA) によるアクセスタイムアウト・切替

#### 問題
証明書認証成功後、端末のポストゥア状態や属性変更に伴い、ISE から RADIUS Change of Authorization (CoA Re-Auth) を発行してセッションをリアルタイムに更新する構成を作成してください。

#### 要件
* スイッチ側で RADIUS Dynamic Authorization (CoA: UDP 1700) を受信可能に設定。
* ISE 側で Posture コマンドまたは手動 Change Status により CoA を送出。

#### 設定例

**Cisco IOS-XE スイッチコンフィグ:**
```bash
aaa server radius dynamic-author
 client 10.1.100.21 server-key Cisco123!
 auth-type any
!
interface GigabitEthernet1/0/1
 access-session control-direction in
```

**ISE 実行操作:**
1. **Operations > RADIUS > Live Sessions** で対象端末の MAC アドレスを選択。
2. **Change of Authorization (CoA)** ボタンを押し、`Re-Auth` または `Port Bounce` を選択。

#### 検証方法
1. スイッチ側で `debug radius` および `debug aaa coa` を有効化。
2. ISE から CoA 送信時、スイッチが `CoA-Request` を受信し、`EAP-TLS` の再認証が即座に開始されることを確認。

---


## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験および筆記試験（SCOR 350-701）を想定した総合問題です。

---

### 問題 1（コンフィグ読解・トラブルシューティング）

**質問**:
Windows PC がスイッチ経由で 802.1X EAP-TLS 認証を試みた際、アクセスが拒否されました。ISE の Live Logs を確認したところ、以下の詳細イベントが記録されていました。

```text
Event: 5400 Authentication failed
Failure Reason: 12514 EAP-TLS failed SSL/TLS handshake because the client certificate is not trusted
Radius Key-Value Pairs:
  User-Name = host/PC-CORP01.example.com
  Calling-Station-ID = 00-11-22-33-44-55
  Selected Allowed Protocol = Default Session
  Identity Store = Preloaded_Certificate_Profile
```

管理者として ISE の Trusted Certificate Store を確認したところ、クライアント証明書を発行した社内 Root CA (`Corp_Root_CA`) の証明書は既に登録済みで、Status は `Enabled` となっていました。この障害の根本原因と解決手順を説明してください。

**解答・解説**:
* **根本原因**:
  ISE の Trusted Certificate Store に Root CA が登録されていても、その証明書の利用目的（Usage / Trusted For）設定において `Trust for client authentication and Syslog`（クライアント認証信頼）のチェックボックスがオフになっているため、ISE は EAP-TLS ハンドシェイク中にクライアント証明書の署名検証を拒否しました。
* **解決手順**:
  1. ISE GUI で **Administration > System > Certificates > Trusted Certificates** に移動。
  2. ストア一覧から `Corp_Root_CA` を選択して **Edit** をクリック。
  3. **Usage** セクションで `Trust for client authentication and Syslog` チェックボックスを有効化。
  4. **Save** をクリックし、端末から再度 802.1X 接続を実行して認証が成功することを確認。

---

### 問題 2（Design・仕様理解）

**質問**:
企業が全社員のマルチOS端末（Windows, macOS, iOS, Android）に対して 802.1X EAP-TLS 認証を導入するにあたり、ISE の Certificate Authentication Profile (CAP) の設計を検討しています。すべての端末で共通して安全かつ確実に Identity を抽出するための CAP 設計として、最も適切なアプローチとその理由を説明してください。

**解答・解説**:
* **推奨設計**:
  CAP の `Use Identity From` 設定において、`Subject Alternative Name (SAN)` > `User Principal Name (UPN)` または `Subject Alternative Name (SAN)` > `Email Address` を主識別子として抽出し、フォールバックとして `Subject - Common Name (CN)` を使用する構成（または SAN 全般を走査する設定）が最善です。
* **理由**:
  * モバイル端末（iOS/Android）や Active Directory 参加端末では、ユーザーのログインIDが UPN または SAN 属性として発行されることが多く、マルチドメイン環境においても一意性が保証されます。
  * また、マシン認証（`HOST/computer.example.com`）とユーザー認証（`user1`）が混在する環境において、SAN を優先抽出対象とすることで、Active Directory に対するグループ検索クエリの曖昧性を排除できます。

---

### 問題 3（実装・コンフィグ問題）

**質問**:
Cisco Catalyst スイッチ上で 802.1X EAP-TLS 認証を有効化し、認証成功時に RADIUS サーバ（ISE）から受領した Dynamic VLAN (VLAN 20) および dACL を適用する C3PL コントロールポリシーを構成してください。また、認証サーバからの応答がない（Timeout）場合のフォールバックとして VLAN 999 に配置する設定を含めてください。

**解答・解説**:

```bash
aaa new-model
aaa group server radius ISE_CLUSTER
 server-private 10.1.100.21 auth-port 1812 acct-port 1813 key Cisco123!
!
aaa authentication dot1x default group ISE_CLUSTER
aaa authorization network default group ISE_CLUSTER
!
radius-server attribute 6 on-for-login-auth
radius-server attribute 8 include-in-access-req
radius-server attribute 25 access-accept include
!
policy-map type control subscriber POLICY_DOT1X_FALLBACK
 event session-started match-all
  10 class always do-until-failure
   10 authenticate using dot1x priority 10
 event server-dead match-all
  10 class always do-until-failure
   10 authorize format default
   20 set-vlan 999
 event authentication-failure match-all
  10 class always do-until-failure
   10 terminate dot1x
!
interface GigabitEthernet1/0/5
 switchport mode access
 access-session port-control auto
 access-session host-mode single-host
 service-policy type control POLICY_DOT1X_FALLBACK
 dot1x pae authenticator
```

---

### 問題 4（トラブルシューティング・ログ解読）

**質問**:
EAP-TLS 認証において、クライアントが TLS ハンドシェイクメッセージ `Certificate Verify` を送信した直後に ISE が RADIUS `Access-Reject` を返却しました。ISE ログには Failure Code `12509` (`CRL check failed`) が出力されていました。原因として考えられる要素を2つ挙げ、失効サーバがダウンしている状況下で緊急復旧させるための設定変更を説明してください。

**解答・解説**:
* **原因（2点）**:
  1. ISE から CRL Distribution Point (CDP) の HTTP/LDAP URL に対するネットワーク到達性が途絶している。
  2. キャッシュされた CRL ファイルの有効期限（Next Update）が切れており、新規の CRL ダウンロードが失敗した。
* **緊急復旧手順**:
  1. ISE GUI で **Administration > System > Certificates > Trusted Certificates** に移動。
  2. 当該 Root/Intermediate CA 証明書を開き、`Certificate Status Validation` セクションに移動。
  3. `Bypass CRL Verification if CRL is not Received` チェックボックスを有効化（または `Download CRL` のチェックを外す）。
  4. 設定を保存することで、CRL が受信できない状態でも失効検証をスキップして認証を許可します。

---

### 問題 5（Design・セキュリティ要求）

**質問**:
離職した社員の証明書が不正に悪用されるリスクを防ぐため、ISE 側で EAP-TLS 認証を行う際に「証明書自体の署名・有効期限検証」に加えて「Active Directory 上で該当ユーザーアカウントが有効（Enabled）であるか」をリアルタイムで確認する設計について述べてください。

**解答・解説**:
* **設計解剖**:
  ISE では、EAP-TLS 認証時に CAP（Certificate Authentication Profile）を通じて Active Directory と連携している場合、認証成功直後に自動的に **AD Courtesy Check**（礼儀チェック）が実行されます。
* **動作メカニズム**:
  1. クライアント証明書が PKI 的に有効（署名OK・有効期限内・未失効）であっても、CAP が抽出した Identity（例: `user1`）をもとに ISE が AD の `userAccountControl` 属性を参照します。
  2. 対象アカウントが `DISABLED` または `LOCKED_OUT` である場合、ISE は認証・認可を拒否（`DenyAccess`）し、RADIUS `Access-Reject` を返却します。
  3. これにより、証明書が物理的に端末に残っていたとしても、AD 上でアカウントを無効化するだけで即座にアクセスを断絶できます。

---

## 🔗 参考リソース

CCIE Security v6.1 ラボ試験および実務設計に役立つ Cisco 公式ドキュメントのリンク一覧です。

* [Cisco ISE Administration Guide - Managing Certificates](https://www.cisco.com/c/en/us/support/security/identity-services-engine/series.html)
* [Cisco ISE EAP-TLS Configuration Guide](https://www.cisco.com/c/en/us/support/security/identity-services-engine/series.html)
* [Cisco Catalyst 9000 Series Switches IBNS 2.0 Deployment Guide](https://www.cisco.com/c/en/us/support/switches/catalyst-9300-series-switches/series.html)
* [Cisco ASA 5500 Series Configuration Guide - Digital Certificates and PKI](https://www.cisco.com/c/en/us/support/security/asa-5500-series-next-generation-firewalls/series.html)

---

## 📝 **補足（Notes）**

### 1. 証明書認証におけるトラブルシューティングチェックリスト
* **端末側**:
  * クライアント証明書が「個人 (Personal)」ストアに配置され、秘密鍵がインポートされているか。
  * 発行元 CA 証明書が「信頼されたルート証明書機関」にインポートされているか。
  * Supplicant 設定で「サーバの証明書を検証する」がオンの場合、ISE の EAP 証明書の FQDN が正しく登録されているか。
* **ISE 側**:
  * Root/Intermediate CA 証明書が Trusted Certificate Store にインポートされ、`Trust for client authentication` がオンになっているか。
  * CAP プロファイルの抽出属性（CN/UPN）が、対向 Identity Store（AD/Internal）のユーザー識別子と一致しているか。
  * ISE のシステム時刻（NTP）が正しいか。

### 2. EAP-TLS ハンドシェイク構造図解
```text
Client (Supplicant)                      ISE (Authentication Server)
       |                                              |
       |-------------- Client Hello ----------------->|
       |<------------- Server Hello ------------------|
       |<------------- Certificate (ISE Cert) --------|
       |<------------- Certificate Request -----------|
       |<------------- Server Hello Done -------------|
       |                                              |
 [Verify Server Cert]                                 |
       |                                              |
       |-------------- Certificate (Client Cert) ---->|
       |-------------- Client Key Exchange ----------->|
       |-------------- Certificate Verify ----------->| (Signed with Private Key)
       |-------------- Change Cipher Spec ------------|
       |-------------- Finished --------------------->|
       |                                              |
       |                                       [Verify Client Cert,
       |                                        Check CRL/OCSP,
       |                                        CAP Identity Extract]
       |                                              |
       |<------------- Change Cipher Spec ------------|
       |<------------- Finished ----------------------|
```

証明書ベース認証は、構成要素の理解と正確な設計パラメーター設定（Trust Flags, CAP Profile, Revocation Rules）が揃うことで、初めて安全かつ強固に稼働します。本学習メモを復習ガイドとして活用してください。




## 📝 **補足（Notes）**

*   **学習メモ**: 「証明書がある＝許可」ではなく、「信頼された CA が出した有効な証明書があり、かつ ID ストアでその ID が有効であること」が証明書認証の完成形です。
*   **注意点**: ラボ試験では、**ISE のシステム時刻（NTP）** が狂っていると、証明書が「未来のもの」や「期限切れ」と判断されるため、必ず `show ntp` で同期を確認してください。
