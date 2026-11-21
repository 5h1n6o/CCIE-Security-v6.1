---
layout: default
title: 4.7.c-RADIUS
nav_order: 3
parent: 4.7-External-identity
grand_parent: 4.0-Identity-Management
---

# 4.7.c External RADIUS

## 📘 概要

External RADIUS (外部 RADIUS サーバー統合) は、Cisco Identity Services Engine (ISE) がネットワークアクセス認証・認可の過程において、自身のアカウントデータベースや直接接続された LDAP/AD ではなく、外部の RADIUS サーバー（例: RSA SecurID Access/Authentication Manager、Microsoft Network Policy Server (NPS)、他社製 RADIUS サーバー、あるいは別の ISE クラスタ）へ認証要求をプロキシ（中継/転送）する機能である。

Cisco ISE はポリシーサーバーおよび RADIUS プロキシ/サーバーの両方の役割を担うことができる。外部 RADIUS サーバー統合により、既存のワンタイムパスワード (OTP) 二要素認証基盤や外部ベンダーの AAA インフラストラクチャを廃棄・変更することなく、Cisco ISE の高度なコンテキストベース認可ポリシー (dACL、Dynamic VLAN、SGT、CoA) とシームレスに結合させることが可能となる。

主に以下のようなユースケースで利用される。
*   **OTP / 2FA 認証の統合**: RSA SecurID や SafeNet などの外部 2FA/MFA トークンサーバーへ RADIUS 経由で認証要求を転送し、認証成功後に ISE がグループ情報や属性に基づいてネットワークアクセスを制御する場面。
*   **マルチテナント / 組織間連携 (eduroam 等)**: 外部ドメインのユーザー（来訪者や提携組織のユーザー）から送られた EAP リクエストを、外部のホーム RADIUS サーバーへプロキシ転送する場面。
*   **既存 AAA インフラとの共存・段階的移行**: 既存の RADIUS サーバー (Microsoft NPS 等) に認証データベースが依存している環境で、アクセススイッチや VPN 機器の認証先を ISE に統合しつつ、実際の認証判断は外部 RADIUS へ委任する場面。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **特徴** | ISE が RADIUS クライアント（プロキシ）として動作し、ネットワーク機器（NAS）から受信した Access-Request を外部 RADIUS サーバーへ転送。戻り値（Access-Accept/Reject）および属性をパースして ISE の認可ポリシーに反映する。 |
| **用途** | 外部 OTP サーバー (RSA SecurID 等) 連携、外部 NPS/RADIUS 統合、eduroam 等のダイレクトプロキシ、複数 AAA ドメインの集約。 |
| **メリット** | 既存の認証データベースや 2FA トークンシステムを変更せずに Cisco ISE の高度な Posture/Profiling/TrustSec 機能と組み合わせ可能。 |
| **デメリット** | 外部 RADIUS サーバーへのネットワーク往復遅延 (RTT) が加算される。MS-CHAPv2 や EAP 認証プロトコルの暗号化パスワード検証において、外部 RADIUS が暗号鍵（NT ハッシュ等）を保持していない場合は認証方式が制限される。 |
| **対応機種** | Cisco ISE 2.x / 3.x 全モデル (PAN/PSN ペルソナ)。外部 RADIUS サーバーとしては RFC 2865 / RFC 2866 準拠の任意の RADIUS サーバーに対応。 |
| **制限事項** | 外部 RADIUS 経由の認証では、ISE が直接パスワード平文や NT ハッシュを検証できないため、EAP 認証方式（EAP-TTLS, PEAP-GTC, PAP）と外部 RADIUS 側の対応プロトコルの組み合わせに注意が必要。 |
| **設計上の注意点** | 外部 RADIUS サーバーのタイムアウト・リトライ設定、プロキシ共有鍵 (Shared Secret) の不一致防止、RADIUS 属性 (AVP) のパススルー vs ISE 内部属性マッピングの設計が極めて重要。 |

---

## 🏗 動作原理

Cisco ISE が Access-Request を受信した際、設定された External RADIUS Server Groups および Policy Sets (Identity Policy / Authentication Policy) の判定ロジックに従い、認証リクエストを外部 RADIUS サーバーへとプロキシ転送する。

### 通信フロー図

```
[ Supplicant / Client ]
       │
       │ 1. EAPOL / Access-Request (EAP-Response)
       ▼
[ Network Access Device (NAS / Switch / VPN) ]
       │
       │ 2. RADIUS Access-Request (UDP 1812)
       ▼
[ Cisco ISE (PSN / RADIUS Proxy) ]
       │
       │ 3. Identity Policy / Authentication Rule Match
       │ 4. Proxy RADIUS Access-Request (UDP 1812)
       ▼
[ External RADIUS Server (RSA SecurID / NPS) ]
       │
       │ 5. Validate Credentials / Token
       │ 6. RADIUS Access-Accept (with Class/Group AVP) (UDP 1812)
       ▼
[ Cisco ISE (PSN / Policy Server) ]
       │
       │ 7. Process External Attributes & Evaluate Authorization Policy
       │ 8. Send RADIUS Access-Accept + dACL / VLAN / SGT (UDP 1812)
       ▼
[ Network Access Device (NAS) ]
       │
       │ 9. Apply Authorization (Port Unlocked / VLAN Assigned)
       ▼
[ Supplicant / Client ]
```

---

## ⚙ 動作シーケンス

1.  **クライアント認証開始**: サプリカントが NAD (Switch/WLC/VPN) に対して EAPOL-Start または直接 Access-Request を送出し、NAD から ISE (PSN) へ UDP 1812 で RADIUS Access-Request が送信される。
2.  **ISE ポリシー評価 (Authentication Policy)**:
    *   ISE は受信した RADIUS Access-Request の属性（User-Name, Calling-Station-ID, NAS-IP-Address 等）をチェックする。
    *   Authentication Policy の条件（例: `User-Name ENDS_WITH @external.com` または `Device-Type EQUALS VPN-Gateways`）に一致するルールが評価され、Identity Source として「External RADIUS Server Group」が選択される。
3.  **プロキシリクエストパケットの生成**:
    *   ISE はクライアントからの Access-Request パケットを外部 RADIUS サーバー宛てに再構築する。
    *   必要に応じて、User-Name のプレフィックス/サフィックス削除（Strip Realm）や追加属性の付与を行う。
    *   プロキシ用の Shared Secret で HMAC-MD5 Authenticator を再計算する。
4.  **外部 RADIUS サーバーへの転送と認証処理**:
    *   外部 RADIUS サーバー (例: RSA Authentication Manager) がリクエストを受信し、ユーザー資格情報 (Passcode / OTP) を検証する。
    *   検証が成功すると、Access-Accept パケットを ISE 返信用 IP アドレスへ UDP 1812 で返却する。このパケットにはグループ属性 (Class AVP または Vendor-Specific Attribute: VSA) が含まれる場合がある。
5.  **ISE 認可ポリシー評価 (Authorization Policy)**:
    *   ISE は外部 RADIUS から受け取った Access-Accept および含まれる AVP (例: `ExternalRADIUS:Class`) を内部属性としてパースする。
    *   Authorization Policy ルールを上から順に評価し、一致したルールで定義された Authorization Profile (dACL, Dynamic VLAN, SGT, Security Group Name) を適用する。
6.  **最終アクセス許可の応答**:
    *   ISE は NAD に対して最終的な Access-Accept を返送する。NAD は指示された VLAN / ACL / SGT をポートまたはセッションに適用し、クライアントのネットワークアクセスを完了させる。



## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、External RADIUS 統合は単純な RADIUS 転送設定にとどまらず、**認可ポリシーとの属性連携**や**EAP トンネル内プロキシの特殊動作**、**ヘッダー/レルムの書き換え設定**が頻繁に出題される。

### Blueprintで重要なポイント
*   **External RADIUS Server / Server Group の定義**: IP アドレス、UDP 1812/1813、Shared Secret、Timeout (デフォルト 5 秒)、Retries (デフォルト 2 回) の設定。
*   **Identity Source / Authentication Policy への組み込み**: Policy Sets 内で External RADIUS Server Group を Identity Source として指定する構成。
*   **Attribute Mapping / Pass-through 制御**: 外部 RADIUS が返す RADIUS 属性 (Class AVP 8 や VSA) を ISE がどのように取り込み、Authorization Rule の Match 条件（例: `ExternalRADIUS:Class EQUALS OTP_Users`）として利用するか。
*   **Single-Sign-On (SSO) / EAP-TTLS / PEAP 認証の終端構成**: ISE が EAP トンネル（Outer EAP）を終端し、内部の PAP/GTC 資格情報のみを外部 RADIUS サーバーへ転送する構成。

### ラボ試験で設定させられそうな内容
1.  **特定ドメインのプロキシルーティング**: ユーザー名が `@partner.com` で終わる場合のみ、パートナー企業用 External RADIUS Server Group へリクエストをプロキシし、`@partner.com` のサフィックスを削除（Strip Realm）して転送する設定。
2.  **RSA SecurID OTP 連携**: 外部 RADIUS サーバーグループを登録し、外部 RADIUS から返される `Class` 属性値を条件として、特定 Dynamic VLAN (VLAN 20) および dACL (`PERMIT_OTP_ONLY`) を返却する認可ポリシールールを設定。
3.  **フェイルオーバー / ロードバランス構成**: 2 台の外部 RADIUS サーバーに対して Failover (Priority ベース) または Load Balance (Round Robin ベース) を適用し、デッドタイムや検出条件をチューニングする設定。

### よくある設定ミス・狙われやすい制限事項
*   **Shared Secret の不一致**: NAD-ISE 間の Shared Secret と、ISE-External RADIUS 間の Shared Secret は全く別物である。混同して設定ミスを起こすケースが多い。
*   **ネットワークインターフェース指定ミス**: ISE が多重 IP (Gigabit0 等) を持つ場合、外部 RADIUS サーバー側で登録されている ISE の IP アドレスと、ISE がプロキシ送信時に使用する送信元 IP インターフェースが一致していないと、外部 RADIUS 側で `Unknown Network Access Device` としてドロップされる。
*   **MS-CHAPv2 パススルー制限**: ISE が EAP-PEAP (MS-CHAPv2) のリクエストを受け取った際、外部 RADIUS サーバーが Active Directory や NT ハッシュに直接アクセスできない場合、MS-CHAPv2 チャレンジ/レスポンスのプロキシ検証が失敗する。外部 RADIUS で MS-CHAPv2 を処理するには、外部 RADIUS 自身が AD 参加しているか、適切な EAP カプセル化プロキシ動作が必要。
*   **Timeout / Retry の合計値と NAD タイムアウトの関係**: ISE から外部 RADIUS への Timeout (5 秒) × Retry (3 回) = 15 秒 と設定している場合、NAD (Switch) 側の RADIUS サーバータイムアウトが 5 秒に設定されていると、ISE が応答を返す前に NAD 側でタイムアウトし、認証が失敗する。

---

## 🛠 設定方法

Cisco ISE GUI での External RADIUS 設定手順および要点を以下に示す。

### 1. External RADIUS Server の作成
*   **ナビゲーション**: **Administration > Identity Management > External Identity Sources > RADIUS Token** (または **External RADIUS Server**)
*   **設定項目**:
    *   `Name`: `External_OTP_Server_1`
    *   `Host IP`: `192.168.100.50`
    *   `Shared Secret`: `Cisco123Secret!`
    *   `Authentication Port`: `1812` (UDP)
    *   `Accounting Port`: `1813` (UDP)
    *   `Timeout`: `5` seconds
    *   `Retries`: `2`

### 2. External RADIUS Server Group の作成
*   **ナビゲーション**: **Administration > Identity Management > External Identity Sources > RADIUS Token > Server Groups**
*   **設定項目**:
    *   `Group Name`: `OTP_Server_Group`
    *   `Selected Servers`: `External_OTP_Server_1`, `External_OTP_Server_2`
    *   `Load Balancing Algorithm`: `Round Robin` または `Failover`
    *   `Accounting Load Balancing`: チェック（必要に応じて）

### 3. Policy Set における Identity Source / Authentication Rule の設定
*   **ナビゲーション**: **Work Centers > Network Access > Policy Sets > [Target Policy Set] > Authentication Policy**
*   **設定手順**:
    1.  新規 Authentication Rule を追加。
    2.  `Conditions`: `RADIUS:User-Name ENDS_WITH @otp-domain.com`
    3.  `Use`: Identity Source として `OTP_Server_Group` を選択。
    4.  `Advanced Options`:
        *   `If User Not Found`: `Reject`
        *   `If Process Error`: `Drop`

### 4. Authorization Policy における属性評価とプロファイル設定
*   **ナビゲーション**: **Work Centers > Network Access > Policy Sets > [Target Policy Set] > Authorization Policy**
*   **設定手順**:
    1.  新規 Authorization Rule を追加（例: `OTP_Access_Rule`）。
    2.  `Conditions`: `ExternalRADIUS:Class EQUALS OTP_ALLOWED_GROUP` (外部 RADIUS から戻却された Class AVP 8 を参照)
    3.  `Profiles`: `PermitAccess` + `dACL_Corporate_Access`

---

## 🔍 検証コマンド

ISE CLI および Cisco Catalyst スイッチ (NAD) での検証コマンド一覧。

| 目的 | コマンド / 確認場所 | 詳細・ポイント |
| --- | --- | --- |
| **ISE Live Logs 確認** | **Operations > RADIUS > Live Logs** | 認証リクエストの概要。`Identity Server` 列に外部 RADIUS サーバー名が表示されているか確認。 |
| **ISE Detailed Authentication Report** | **Operations > RADIUS > Live Logs > [Details Icon 🔍]** | `Authentication Result` 内の `Identity Generation Sequence`, `Proxy RADIUS Details`, `External RADIUS Response Time`, `Attributes Received from External RADIUS` を確認。 |
| **ISE CLI プロキシ通信デバッグ** | <code>debug logging component runtime level debug</code> | ISE CLI にて `prrt-management.log` および `runtime.log` をリアルタイム監視し、External RADIUS パケットの送受信を確認。 |
| **ISE ログファイル閲覧 (CLI)** | <code>show logging application runtime.log tail 100</code> | プロキシ要求のタイムアウトや Shared Secret 不一致ログメッセージを監視。 |
| **NAD RADIUS サーバー状態確認** | <code>show radius server-groups</code> | スイッチから見た ISE (PSN) の応答状態、RTT、デッドカウンターを確認。 |
| **NAD 認証セッション確認** | <code>show authentication sessions interface Gig1/0/1 detail</code> | ISE から最終的に降ってきた Dynamic VLAN, dACL, SGT が適用されているか検証。 |

---

## 🚨 トラブルシュート

External RADIUS 連携における代表的な障害パターンと対処方法。

| 症状 | 原因 | 確認コマンド / ログ確認場所 | 対処方法 |
| --- | --- | --- | --- |
| **ISE ログに `External RADIUS Server Timeout` が記録される** | ISE から外部 RADIUS への IP 到達性がない、または外部 RADIUS 側で ISE の IP が許可されていない（ドロップ）。 | ISE Live Logs の Details パネル。`ping 192.168.100.50` (ISE CLI), 外部 RADIUS ログ。 | 外部 RADIUS 側で ISE の Management/PSN IP アドレスを CLIENT として登録し、ファイアウォールで UDP 1812 を許可する。 |
| **ISE ログに `Shared Secret Mismatch with External RADIUS` が記録される** | ISE と外部 RADIUS サーバー間で設定された共有鍵が不一致。 | ISE Detailed Log: `RADIUS Message Authenticator failed validation`. 外部 RADIUS ログ。 | ISE の External RADIUS Server 設定と外部 RADIUS 側の Shared Secret を完全に一致させる。 |
| **外部 RADIUS は Access-Accept を返しているが、ISE で `Authorization Failed` になる** | 外部 RADIUS が返す Class 属性や Group 属性が ISE の Authorization Policy ルール条件と合致していない。 | ISE Detailed Log: `Attributes Received` セクションの `Class (8)` の値を確認。 | Authorization Policy 条件の `ExternalRADIUS:Class` または属性定義を外部 RADIUS が返す文字列と一致させる。 |
| **NAD 側で `RADIUS server is not responding` になり認証が失敗する** | ISE の外部 RADIUS タイムアウト処理時間が NAD の RADIUS Timeout 設定値を超過している。 | NAD: <code>show radius statistics</code><br>ISE: `Proxy Timeout` ログ | NAD 側の RADIUS タイムアウト値を延長（例: <code>radius-server retransmit 3 timeout 10</code>）するか、ISE 側の Timeout を短縮する。 |
| **ユーザー名のレルム除去が動作せず外部 RADIUS で認証不可** | ISE の Advanced Strip Realm / Prefix-Suffix 設定が効いていないため `user@domain.com` そのまま外部 RADIUS へ送信されている。 | ISE Detailed Log: `User-Name Sent to External RADIUS` 属性を確認。 | ISE External RADIUS Server Group 設定にて `Strip Realm` オプションを有効化する。 |



## ⚠ 制限事項

*   **認証プロトコル制限**: EAP-PEAP (MS-CHAPv2) や EAP-TTLS (MS-CHAPv2) を使用する場合、外部 RADIUS サーバー自身がユーザーのパスワードハッシュ（NT ハッシュ等）にアクセスできないと認証を完了できない。外部 RADIUS が OTP 専用サーバー (PAP 必須) の場合、サプリカントと ISE 間の Outer/Inner EAP 設定（EAP-GTC または PAP リダイレクト）に制約が生じる。
*   **Change of Authorization (CoA) 制限**: 外部 RADIUS サーバーから ISE に対して直接 RADIUS CoA (RFC 3576 / RFC 5176) を要求する構成はサポートされない。CoA のトリガーは常に ISE 内部（Posture 状態変更、Identity 変更等）から NAD に対して送信される必要がある。
*   **アカウントデータベースの検索不可**: Cisco ISE GUI 上で External RADIUS サーバーグループ内の個々のユーザーアカウント一覧をブラウズ・参照することはできない。認証要求が発生した時点で初めてオンデマンドでプロキシ要求が送信される。
*   **ライセンス要件**: Cisco ISE 3.x 以降において、External RADIUS プロキシ機能および高度な認可制御 (dACL, SGT) を利用するには `Advantage` または `Premier` ライセンスが必要となる場合がある。

---

## 🔄 他技術との関連

*   **Cisco TrustSec (SGT)**: 外部 RADIUS サーバーでユーザー認証が成功した後、ISE は外部 RADIUS から受け取ったグループ属性 (Class AVP 等) をトリガーとして動的に Security Group Tag (SGT) を割り当て、NAD へ降ろすことができる。
*   **Posture / Compliance**: クライアントが 802.1X / EAP 認証を外部 RADIUS 経由で完了した後、ISE の AnyConnect ISE Posture Agent が起動し、ISE PSN と直接 Posture 評価を実施して最終的なアクセス権限をリダイレクト CoA で切り替える。
*   **BYOD / Onboarding**: 外部 RADIUS で一時認証（ゲスト/暫定アクセス）を実施し、その後 ISE の BYOD セルフ登録ポータルへ誘導して証明書 (EAP-TLS) をプロビジョニングする連動設計が可能。
*   **Cisco Firepower (FTD / FMC)**: Remote Access VPN (AnyConnect) の認証先を ISE に設定し、ISE が要求を External RADIUS (RSA OTP) へ転送。認証成功後に ISE が FMC 宛てに Dynamic Authorization Profile や SGT 情報を返し、FTD 上で Access Control Policy を制御する。

---

## 🧩 比較表

Cisco ISE における各種外部 Identity Source の比較。

| 比較項目 | Active Directory (Native Join) | OpenLDAP / Direct LDAP | External RADIUS Server |
| --- | --- | --- | --- |
| **接続プロトコル** | Kerberos, MS-RPC, SMB (TCP 445, 88, 389, 3268) | LDAP / LDAPS (TCP 389, 636) | RADIUS / UDP 1812 |
| **支持する EAP 認証** | EAP-TLS, PEAP-MSCHAPv2, EAP-FAST, EAP-TTLS | EAP-TLS, PEAP-GTC, EAP-TTLS (PAP/GTC) | PEAP-GTC, EAP-TTLS (PAP), PAP/ASCII |
| **ドメイン参加手続** | 必須 (ISE 機器自体を AD ドメインに参加させる) | 不要 (Bind DN と接続設定のみ) | 不要 (Shared Secret と IP 設定のみ) |
| **OTP / 2FA 連携** | 不可 (AD 単体では不可。サードパーティプラグイン必要) | 不可 (基本はパスワード照合のみ) | **最適 (RSA SecurID / SafeNet 等とネイティブ連携)** |
| **ユーザー属性の検索** | フルサポート (GUI でグループ検索・参照可能) | フルサポート (Base DN から検索可能) | 不可 (プロキシ応答に含まれる AVP のみ利用可能) |
| **冗長化方式** | DC Site Affinity / DNS SRV による自動検出 | ISE で複数 LDAP サーバー優先度指定 | ISE で RADIUS Server Group (Failover / Round Robin) |

---

## 💡 ベストプラクティス

1.  **外部 RADIUS 専用の Server Group を定義する**: 1 台の外部 RADIUS サーバー設定であっても、必ず Server Group を作成して管理する。将来の冗長化追加やロードバランス設定変更が容易になる。
2.  **Strip Realm (属性整形) の活用**: ユーザーが `user@otp-domain.local` と入力してログインする場合、外部 RADIUS (RSA 等) が `user` のみのユーザー名を期待しているケースが多い。Server Group 設定で `Strip Prefix/Suffix` を適切に設定する。
3.  **タイムアウト値の連鎖設計**:
    *   **NAD (Switch/VPN)**: Timeout 10 秒, Retries 2 回
    *   **ISE (Proxy)**: Timeout 4 秒, Retries 2 回
    *   **External RADIUS (RSA)**: レスポンス処理時間 < 2 秒
    *   このように「NAD > ISE > External RADIUS」となるようタイムアウト値を設計し、NAD 側の不必要なフェイルオーバーを防止する。
4.  **Fallback Policy の明確化**: 外部 RADIUS サーバー群がすべて応答しない (`Process Error`) 場面において、要求をドロップするのか、制限付き VLAN に隔離するのかを Authentication Policy の `Advanced Options` で厳格に定義する。



## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルを想定した実践的な設定演習シナリオ（シナリオ 1 〜 5）。

---

### シナリオ 1: RSA SecurID 外部 RADIUS サーバーグループの登録と基本プロキシ設定

#### 問題
企業ネットワークにおいて、リモート Access VPN ユーザーの 2FA 認証を外部の RSA Authentication Manager (IP: 10.1.100.50) へ委任する設計を行う。ISE を RADIUS プロキシとして動作させよ。

#### 要件
1. ISE GUI にて 外部 RADIUS サーバー `RSA_SecurID_Primary` (IP: 10.1.100.50, Shared Secret: `CiscoRSA123!`, Timeout: 4s, Retry: 2) を作成すること。
2. 外部 RADIUS サーバーグループ `RSA_OTP_Group` を作成し、上記サーバーを組み込むこと。
3. ユーザー名が `@otp.corp` で終わるリクエストについて、`@otp.corp` 領域を削除（Strip Realm）した上で `RSA_OTP_Group` へ転送すること。

#### 設定例

**1. External RADIUS Server 作成 (ISE GUI)**
*   **Path**: Administration > Identity Management > External Identity Sources > RADIUS Token > RADIUS Token Servers
*   **Name**: `RSA_SecurID_Primary`
*   **IP Address**: `10.1.100.50`
*   **Shared Secret**: `CiscoRSA123!`
*   **Authentication Port**: `1812`
*   **Accounting Port**: `1813`
*   **Timeout**: `4`
*   **Retries**: `2`

**2. External RADIUS Server Group 作成 (ISE GUI)**
*   **Path**: Administration > Identity Management > External Identity Sources > RADIUS Token > Server Groups
*   **Group Name**: `RSA_OTP_Group`
*   **Selected Servers**: `RSA_SecurID_Primary`
*   **Strip Suffix**: `@otp.corp` (有効化)

**3. Policy Set Authentication Policy (ISE GUI)**
*   **Path**: Work Centers > Network Access > Policy Sets > `VPN_Policy_Set` > Authentication Policy
*   **Rule Name**: `OTP_Proxy_Rule`
*   **Conditions**: `RADIUS:User-Name ENDS_WITH @otp.corp`
*   **Use**: `RSA_OTP_Group`

#### 検証方法
*   サプリカント/VPN クライアントから ユーザー名 `testuser@otp.corp` および OTP パスコードを入力して接続試行。
*   ISE CLI でログ確認: <code>show logging application runtime.log tail 50</code>
*   `User-Name` が `testuser` に書き換えられて IP `10.1.100.50` へ UDP 1812 送信されていることを確認。

---

### シナリオ 2: 外部 RADIUS から返される Class 属性に基づく動的 dACL 制御

#### 問題
外部 RADIUS サーバー認証成功時に返却される RADIUS 属性 `Class (8)` の値に応じ、Cisco ISE 上で動的 Access Control List (dACL) を適用せよ。

#### 要件
1. 外部 RADIUS サーバーから `Class = "ADMIN_GROUP"` が返された場合、ISE は `PERMIT_ALL_DACL` を適用すること。
2. 外部 RADIUS サーバーから `Class = "CONTRACTOR_GROUP"` が返された場合、ISE は `RESTRICTED_DACL` を適用すること。

#### 設定例

**1. Authorization Profile の定義 (ISE GUI)**
*   **Profile 1**: `AuthZ_Admin_Profile`
    *   dACL Name: `PERMIT_ALL_DACL` (内容: `permit ip any any`)
*   **Profile 2**: `AuthZ_Contractor_Profile`
    *   dACL Name: `RESTRICTED_DACL` (内容: `permit icmp any any`, `permit tcp any any eq 80`, `deny ip any any`)

**2. Authorization Policy の設定 (ISE GUI)**
*   **Rule 1 Name**: `Rule_Admin_Class`
    *   **Conditions**: `ExternalRADIUS:Class EQUALS ADMIN_GROUP`
    *   **Permissions**: `AuthZ_Admin_Profile`
*   **Rule 2 Name**: `Rule_Contractor_Class`
    *   **Conditions**: `ExternalRADIUS:Class EQUALS CONTRACTOR_GROUP`
    *   **Permissions**: `AuthZ_Contractor_Profile`

#### 検証方法
*   外部 RADIUS 側で `CONTRACTOR_GROUP` を返すテストユーザーでログイン。
*   スイッチ側で dACL 適用状態を確認:
    <code>show authentication sessions interface Gig1/0/2 detail</code>
*   `ACS ACL: RESTRICTED_DACL` が正しくダウンローダブル ACL として適用されているか検証。

---

### シナリオ 3: 複数外部 RADIUS サーバーのロードバランシングと自動フェイルオーバー

#### 問題
高可用性を確保するため、2 台の外部 RADIUS サーバー (10.1.100.51, 10.1.100.52) に対する Round Robin ロードバランシング構成を設定せよ。

#### 要件
1. 外部 RADIUS サーバー `Ext_RADIUS_1` (10.1.100.51) および `Ext_RADIUS_2` (10.1.100.52) を登録すること (Shared Secret: `RadiusPass123!`)。
2. サーバーグループ `Ext_RADIUS_LB_Group` を作成し、Round Robin ロードバランシングを有効化すること。
3. サーバー無応答時の Dead Time を 5 分に設定すること。

#### 設定例

**1. RADIUS Server 登録 (ISE GUI)**
*   `Ext_RADIUS_1`: `10.1.100.51`, Shared Secret: `RadiusPass123!`, Timeout: 3, Retries: 2
*   `Ext_RADIUS_2`: `10.1.100.52`, Shared Secret: `RadiusPass123!`, Timeout: 3, Retries: 2

**2. Server Group 設定 (ISE GUI)**
*   **Group Name**: `Ext_RADIUS_LB_Group`
*   **Selected Servers**: `Ext_RADIUS_1`, `Ext_RADIUS_2`
*   **Load Balancing Algorithm**: `Round Robin`
*   **Dead Time**: `5` minutes

#### 検証方法
*   連続して複数回の認証リクエストを投入する。
*   **Operations > RADIUS > Live Logs** の Details パネルにて、リクエストが 10.1.100.51 と 10.1.100.52 に交互に振り分けられていることを確認。
*   `Ext_RADIUS_1` 宛てのインターフェースをダウンさせ、自動的に `Ext_RADIUS_2` へ要求がフェイルオーバーすることを確認。

---

### シナリオ 4: EAP-TTLS (PAP) トンネル内プロキシによる OTP 認証統合

#### 問題
Windows / Mac サプリカントからの 802.1X EAP 認証において、EAP-TTLS トンネル内部で PAP 資格情報を外部 RADIUS (OTP サーバー) へ安全にプロキシ転送する構成を構築せよ。

#### 要件
1. ISE は Outer EAP トンネルとして EAP-TTLS を終端し、ISE サーバー証明書を提示すること。
2. Inner EAP メカニズムとして PAP / ASCII を使用し、内部資格情報を平文抽出して外部 RADIUS サーバーグループ `OTP_Server_Group` へプロキシ送信すること。
3. Allowed Protocols 設定で `EAP-TTLS` を許可し、内部 PAP 認可を有効化すること。

#### 設定例

**1. Allowed Protocols 調整 (ISE GUI)**
*   **Path**: Policy > Policy Elements > Results > Authentication > Allowed Protocols
*   **Name**: `Default_EAP_TTLS_OTP_Allowed_Protocols`
*   **Allow EAP-TTLS**: チェック
    *   **Inner Attributes**: `PAP / ASCII` チェック有効化

**2. Authentication Policy 設定 (ISE GUI)**
*   **Rule Name**: `8021X_TTLS_OTP_Rule`
*   **Conditions**: `Network Access:EapTunnel EQUALS EAP-TTLS`
*   **Use**: `OTP_Server_Group`

#### 検証方法
*   PC サプリカントで 802.1X 認証プロトコルとして EAP-TTLS / PAP を選択して接続。
*   ISE Live Logs にて `EAP-TTLS (PAP)` として受け取った資格情報がプロキシ送信され、Access-Accept が返ることを確認。

---

### シナリオ 5: 外部 RADIUS 障害時のローカル認証フォールバック (Identity Source Sequence)

#### 問題
外部 RADIUS サーバー障害（Process Error / Timeout）発生時に、ローカル ISE ユーザーデータベースへの緊急認証フォールバックを実装せよ。

#### 要件
1. Identity Source Sequence `ISS_External_then_Local` を作成すること。
2. 最初の優先順位に `Ext_RADIUS_Group`、2 番目に `Internal Users` を配置すること。
3. `If Process Error` (サーバー無応答時) の場合は `Search Next` を選択すること。

#### 設定例

**1. Identity Source Sequence の作成 (ISE GUI)**
*   **Path**: Administration > Identity Management > Identity Source Sequences
*   **Name**: `ISS_External_then_Local`
*   **Authentication Search List**:
    1.  `Ext_RADIUS_Group`
    2.  `Internal Users`
*   **Advanced Search Rules Configuration**:
    *   `If Ext_RADIUS_Group returns User Not Found`: `Do Not Access Next Store` (ユーザー不在時は拒否)
    *   `If Ext_RADIUS_Group returns Process Error`: `Access Next Store` (サーバー障害時のみローカル検索へフォールバック)

**2. Authentication Policy 適用 (ISE GUI)**
*   **Authentication Rule**: `Use` 列で `ISS_External_then_Local` を指定。

#### 検証方法
*   外部 RADIUS サーバーを停止状態にする。
*   ISE ローカルデータベースに登録済みの緊急管理ユーザーでログイン試行。
*   Live Logs にて `Process Error` が検知され、自動的に `Internal Users` で認証成功することを確認。



### シナリオ 6: Cisco FTD / AnyConnect 認証用 External RADIUS 統合と Dynamic Access-Policy 適用

#### 問題
Cisco Firepower Threat Defense (FTD) で収容する AnyConnect リモートアクセス VPN ユーザーの認証を ISE 経由で 外部 RADIUS へ転送し、認証成功後に SGT (Security Group Tag) を付与せよ。

#### 要件
1. FTD からの RADIUS リクエストを ISE で受信し、外部 RADIUS `Ext_RADIUS_Group` へプロキシすること。
2. 認証成功時、ISE は SGT `Contractor_SGT (Tag 15)` を決定し、Access-Accept 内で FTD へ返却すること。
3. FTD 側で SGT に基づくアクセス制御ポリシーを適用可能とすること。

#### 設定例

**1. ISE Network Device 登録 (ISE GUI)**
*   **Name**: `FTD_Headend`
*   **IP Address**: `172.16.10.1`
*   **Shared Secret**: `CiscoVPNSecret123!`

**2. ISE Authorization Profile & Rule (ISE GUI)**
*   **Authorization Profile Name**: `Profile_SGT_Contractor`
    *   **Security Group (SGT)**: `Contractor (15)`
*   **Authorization Rule**:
    *   **Condition**: `ExternalRADIUS:Class EQUALS CONTRACTOR`
    *   **Permissions**: `Profile_SGT_Contractor`

#### 検証方法
*   AnyConnect クライアントから VPN 接続を実行。
*   ISE Live Logs にて `FTD_Headend` からのリクエストがプロキシされ、SGT 15 が割り当てられて Access-Accept が返却されていることを確認。
*   FTD CLI または FMC コマンドで VPN セッションの SGT 割当を確認:
    <code>show vpn-sessiondb detail anyconnect</code>

---

### シナリオ 7: eduroam 連携における Outgoing RADIUS Proxy と Vendor-Specific Attribute (VSA) 制御

#### 問題
大学キャンパスネットワークにおいて、他校からの訪問研究者の EAP リクエストを eduroam 外部 RADIUS プロキシサーバーへ転送し、特定の RADIUS VSA を付与して送信せよ。

#### 要件
1. ユーザー名が `@eduroam.edu` で終わる EAP-TTLS リクエストを外部 RADIUS `Eduroam_Proxy_Group` へプロキシすること。
2. ISE はプロキシ送信時に `Operator-Name` 属性 (RADIUS Attribute 126) をパケットに追加すること。

#### 設定例

**1. External RADIUS Server Group 設定 (ISE GUI)**
*   **Group Name**: `Eduroam_Proxy_Group`
*   **RADIUS Attributes Customization**:
    *   Add Attribute: `Operator-Name (126)` = `1myuniversity.edu`

**2. Authentication Policy (ISE GUI)**
*   **Condition**: `User-Name ENDS_WITH @eduroam.edu`
*   **Use**: `Eduroam_Proxy_Group`

#### 検証方法
*   訪問ユーザーが `user@eduroam.edu` で接続。
*   ISE でキャプチャ取得: <code>tech-support-bundle</code> または Wireshark/Tcpdump にて UDP 1812 パケットを確認。
*   送信 Access-Request パケットに `Operator-Name = 1myuniversity.edu` が含まれていることをパケットレベルで検証。

---

### シナリオ 8: Cisco WLC 802.1X + CWA (Central Web Authentication) + 外部 RADIUS 2FA 認証

#### 問題
無線 LAN 環境において、第1段階で MAC Authentication Bypass (MAB) を行い CWA ポータルへリダイレクトし、第2段階の Web ポータルログイン認証を外部 RADIUS 2FA サーバーへ転送せよ。

#### 要件
1. MAB 段階では CWA リダイレクト URL および Redirect ACL `REDIRECT_GUEST_ACL` を返却すること。
2. ユーザーが ISE ゲスト/Web ポータルで入力した 2FA ID/Password を 外部 RADIUS `OTP_Web_Group` へ送信すること。
3. Web 認証成功後、CoA (Change of Authorization) を発行してポートをフルアクセスに切り替えること。

#### 設定例

**1. Authentication Policy (ISE GUI)**
*   **Rule 1 (MAB / Initial)**:
    *   Condition: `Wireless_MAB`
    *   Use: `Internal Endpoints`
*   **Rule 2 (WebAuth / Portal)**:
    *   Condition: `Identity_Policy_WebAuth`
    *   Use: `OTP_Web_Group`

**2. Authorization Policy (ISE GUI)**
*   **Rule 1 (Redirect Rule)**:
    *   Condition: `AuthenticationStatus EQUALS AuthorizationPending`
    *   Permissions: `CWA_Profile` (Redirect URL + `REDIRECT_GUEST_ACL`)
*   **Rule 2 (Post WebAuth Success)**:
    *   Condition: `Network Access:UseCase EQUALS GuestFetch` AND `ExternalRADIUS:Class EQUALS OTP_SUCCESS`
    *   Permissions: `PermitAccess`

#### 検証方法
*   無線クライアントが SSID に接続し、ブラウザを開いてポータルにリダイレクトされるか検証。
*   OTP パスコードを入力してログイン完了後、WLC 上で CoA Re-authentication が発生し、リダイレクトが解除されることを確認。

---

### シナリオ 9: 外部 RADIUS Accounting Proxy (課金・ログ情報の外部転送)

#### 問題
ネットワーク機器からの RADIUS Accounting (課金/セッションログ) パケットを ISE で受信し、自組織の SIEM / RADIUS Accounting 集約サーバー宛てにそのままプロキシ転送せよ。

#### 要件
1. ISE PSN が NAD から受信した UDP 1813 Accounting-Request パケットを 外部 Accounting サーバー `SIEM_RADIUS_Acc` (192.168.200.10) へ転送すること。
2. Accounting データの送信用共有鍵は `AccSecret456!` とすること。

#### 設定例

**1. External RADIUS Accounting Server 設定 (ISE GUI)**
*   **Path**: Administration > Identity Management > External Identity Sources > RADIUS Token > RADIUS Token Servers
*   **Accounting Host**: `192.168.200.10`
*   **Accounting Port**: `1813`
*   **Shared Secret**: `AccSecret456!`

**2. Server Group の Accounting 有効化 (ISE GUI)**
*   `SIEM_Acc_Group` を作成し、`Enable Accounting Proxy` オプションを有効化。

#### 検証方法
*   クライアントが接続・切断した際、ISE および 192.168.200.10 側で Accounting Start / Stop パケットがパケットキャプチャで転送されていることを確認。

---

### シナリオ 10: 外部 RADIUS プロキシにおける MS-CHAPv2 パススルー制限の回避と EAP-GTC 統合

#### 問題
サプリカントが EAP-PEAP を使用する環境で、外部 RADIUS が MS-CHAPv2 を解読できない制約がある。サプリカント側で EAP-GTC を使用するように構成し、ISE 経由で外部 OTP サーバーへ平文パスワード認証を行わせよ。

#### 要件
1. ISE Allowed Protocols 設定にて `PEAP (EAP-GTC)` を有効化し、`PEAP (EAP-MSCHAPv2)` を無効化（または低優先度化）すること。
2. EAP-GTC トンネル内部のユーザー資格情報を外部 RADIUS `OTP_Server_Group` へプロキシ送信すること。

#### 設定例

**1. Allowed Protocols の編集 (ISE GUI)**
*   **Path**: Policy > Policy Elements > Results > Authentication > Allowed Protocols
*   **Name**: `PEAP_GTC_OTP_Only`
*   **Allow PEAP**: チェック
    *   `Allow EAP-MSCHAPv2`: チェック解除
    *   `Allow EAP-GTC`: チェック有効化

**2. Policy Set 設定 (ISE GUI)**
*   **Authentication Policy**: Identity Source として `OTP_Server_Group` を指定し、Allowed Protocols に `PEAP_GTC_OTP_Only` を割り当て。

#### 検証方法
*   AnyConnect NAM または Cisco EAP-GTC モジュールをインストールした PC から接続。
*   GTC プロンプトに OTP パスコードを入力。
*   ISE Live Logs にて EAP-GTC が正常にプロキシされ認証成功となることを検証。



## ❓ 想定試験問題

CCIE Security v6.1 ラボ・筆記試験レベルを意識した実践問題（全 6 問）。

---

### 問題 1 (コンフィグ読解・トラブルシュート)
Cisco ISE を使用して外部 OTP サーバー宛てに RADIUS プロキシを設定したが、サプリカントが認証試行を行った際、ISE Live Logs に `External RADIUS Server Unreachable` と表示され、外部 OTP サーバー側にもログが記録されない。スイッチ側の RADIUS 設定および ISE CLI の ping は疎通確認が取れている。この障害の最も可能性の高い原因を 2 つ選べ。

*   A. ISE とスイッチ間の RADIUS Shared Secret が不一致である。
*   B. 外部 RADIUS サーバー側で Cisco ISE の IP アドレスが RADIUS クライアントとして許可登録されていない。
*   C. 外部 RADIUS サーバー宛ての UDP 1812 通信がネットワーク途中のファイアウォールでブロックされている。
*   D. ISE の Allowed Protocols 設定で EAP-TLS が許可されていない。
*   E. ISE の Authorization Policy に PermitAccess プロファイルが設定されていない。

**正解**: B, C
**解説**:
*   `External RADIUS Server Unreachable` は、ISE から外部 RADIUS サーバーへ Access-Request を送信したものの応答（Access-Accept/Reject/Challenge）がタイムアウト時間内に返ってこない状態を示す。
*   原因としては、途中の FW で UDP 1812 がドロップされているケース (C)、または外部 RADIUS サーバー側で ISE の IP アドレスが Client として登録されていないため無応答ドロップされているケース (B) が該当する。
*   A は ISE-Switch 間の問題であり外部 RADIUS へのプロキシとは無関係。D や E はプロキシパケット送受信後の処理問題である。

---

### 問題 2 (Design・プロトコル制限)
企業が EAP-PEAP (MS-CHAPv2) を使用した 802.1X 認証を導入しており、認証バックエンドとして外部の RADIUS サーバー (RSA SecurID) を使用して OTP (One-Time Password) 認証を実施したいと考えている。設計上の考慮事項として正しいものはどれか。

*   A. EAP-PEAP (MS-CHAPv2) は外部 RADIUS サーバー宛てのプロキシ認証をネイティブにサポートするため、設定変更は不要である。
*   B. MS-CHAPv2 はチャレンジ/レスポンス方式でパスワードハッシュを検証するため、外部 RADIUS (OTP サーバー) がユーザーの平文パスワードやハッシュを保持していない限り直接認証できない。EAP-GTC や EAP-TTLS (PAP) への変更が必要である。
*   C. 外部 RADIUS サーバー統合を使用する場合、EAP トンネルは常に外部 RADIUS サーバー上で終端されなければならない。
*   D. ISE は MS-CHAPv2 チャレンジを平文パスワードに自動変換して外部 RADIUS サーバーへ転送する機能を備えている。

**正解**: B
**解説**:
*   MS-CHAPv2 はクライアントとサーバー間でチャレンジ/レスポンスを行い、NT ハッシュを用いて相互検証する。OTP のように認証ごとに変化するワンタイムパスワードは、MS-CHAPv2 のハッシュ計算メカニズムと互換性がない。
*   したがって、OTP サーバーへ RADIUS プロキシを行う場合は、サプリカントが平文（または暗号化トンネル内の平文）で資格情報を送信する EAP-GTC や EAP-TTLS (PAP) を採用する必要がある。

---

### 問題 3 (実装・レルム操作)
ユーザーが `john.doe@external-corp.com` の形式でユーザー名を入力する環境において、ISE から 外部 RADIUS サーバー宛てにリクエストを転送する際、外部 RADIUS サーバーには `john.doe` のみ（レルム削除状態）で送信したい。ISE GUI のどの場所で設定を行うべきか。

*   A. **Administration > System > Settings > RADIUS > Custom Attributes**
*   B. **Administration > Identity Management > External Identity Sources > RADIUS Token > Server Groups** の `Strip Suffix` オプション
*   C. **Work Centers > Network Access > Authorization Rules** の `Profile` 設定
*   D. **Network Devices > RADIUS Authentication Settings** の `Vendor Specific Attributes`

**正解**: B
**解説**:
*   Cisco ISE において、プロキシ送信時のユーザー名整形（Prefix/Suffix の削除や置換）は **External RADIUS Server Group** の設定プロパティ内で定義する。
*   `Strip Suffix` に `@external-corp.com` を指定することで、ISE は送信パケットの `User-Name` 属性からレルム部分を自動的に削除して外部 RADIUS へパケットを再構築する。


### 問題 4 (トラブルシューティング・ログ解読)
外部 RADIUS サーバーから Access-Accept が返却され、パケット内に Class (8) 属性が含まれているにもかかわらず、ISE Live Logs でユーザーがデフォルトの認可ルールに一致してしまった。原因として最も当てはまるものはどれか。

*   A. スイッチ側で RADIUS Accounting が無効になっているため。
*   B. ISE Authorization Policy の条件で `ExternalRADIUS:Class` の記述ミス（文字列不一致や属性参照誤り）がある。
*   C. 外部 RADIUS サーバーが UDP 1813 で Accounting を送信していないため。
*   D. ISE が RADIUS Token Server のライセンスを失効しているため。

**正解**: B
**解説**:
*   外部 RADIUS が返す AVP（例: Class 属性 8）を ISE の認可条件で使用する場合、ISE は内部属性 `ExternalRADIUS:Class` として参照する。
*   文字列の完全一致や属性参照の定義ミスがあると、認可ルールにヒットせず、フォールバックルールが適用されてしまう。

---

### 問題 5 (ハイアベイラビリティ設計)
ISE が 2 台の外部 RADIUS サーバーグループに対して Failover モードで接続している。Primary サーバーが無応答になったと判定され、Secondary サーバーへ切り替わる条件および調整パラメーターとして正しいものはどれか。

*   A. ISE が設定された `Timeout` × `Retries` の回数連続で応答を受信できなかった場合、Primary を Dead と判定し Secondary へ切り替える。
*   B. 外部 RADIUS サーバーの切替は RADIUS Dead Time に関係なく毎パケット交互に行われる。
*   C. Primary サーバーが復旧しても、ISE は手動で再起動するまで Primary へ自動復帰することはない。
*   D. 外部 RADIUS サーバーの死活監視は ICMP Ping のみによって行われる。

**正解**: A
**解説**:
*   ISE の External RADIUS Server Group が Failover モードの場合、Primary サーバー宛てに送った Access-Request が `Timeout` × `Retries` に達しても無応答である場合に Primary を Dead と判定し、即座に Secondary サーバーへ要求を送信する。
*   Dead Time 経過後、あるいはバックグラウンドの RADIUS ヘルスチェックによって Primary の復旧が確認されれば、次回以降のリクエストは Primary へ自動復帰する。

---

### 問題 6 (コンフィグ読解・CoA/RADIUS 動作)
外部 RADIUS 認証成功後、ユーザーの Posture 状態が Compliant に変化した。ISE がスイッチ宛てに RADIUS CoA を発行してアクセス権限を変更する際、プロトコルおよび属性の動きとして正しいものはどれか。

*   A. 外部 RADIUS サーバーがスイッチ宛てに直接 RADIUS CoA (UDP 3799) を送信する。
*   B. ISE (PSN) がスイッチ宛てに RADIUS CoA (UDP 1700 または 3799) を送信し、新しい dACL / VLAN 属性を適用する。
*   C. スイッチが外部 RADIUS サーバー宛てに RADIUS Accounting-Request を送信して認可を更新する。
*   D. 外部 RADIUS 統合を使用している場合、RADIUS CoA 機能は完全に無効化される。

**正解**: B
**解説**:
*   認証自体が外部 RADIUS で処理された場合であっても、セッションの動的認可制御 (CoA) のトリガーは Cisco ISE (PSN) が保持・管理する。
*   Posture 状態変更に伴う CoA パケットは、ISE から NAD 宛てに送信される。

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術資料情報。

*   Cisco ISE 3.1 Administrator Guide - Manage External Identity Sources (Cisco Official)
*   Cisco Technical Support Documentation - Configure External RADIUS Server on ISE (Cisco Official)
*   Cisco Live Session BRKSEC-3699 - Advanced ISE Troubleshooting and External Integrations (Cisco Live)
*   Cisco Validated Design - SD-Access Secure Campus Design Guide (CVD)
*   RFC 2865 - Remote Authentication Dial In User Service (RADIUS)
*   RFC 2866 - RADIUS Accounting
*   RFC 3576 - Dynamic Authorization Extensions to RADIUS

---

## 📝 **補足（Notes）**

### 1. ログ追跡時の重要チェック項目 (ISE Details Report)
外部 RADIUS 統合トラブルシューティング時、ISE Live Logs の Details レポートで特に注意して確認すべき属性項目:
*   `IdentitySourcePassed`: 実際に認証を処理した External RADIUS Server Group 名。
*   `Proxy-RADIUS-Response-Code`: `Access-Accept`, `Access-Reject`, または `Timeout`。
*   `Attributes-Received-From-External-RADIUS`: Class (8), State (24), VSA 等。

### 2. パケット構造と Authenticator 再計算メカニズム
ISE が NAD から Access-Request を受信し外部 RADIUS へプロキシする際、パケット内の Authenticator フィールドおよび User-Password 属性は、ISE と 外部 RADIUS 間の Shared Secret を用いて暗号化・HMAC 再計算が行われる。
そのため、NAD ↔ ISE 間の Shared Secret が正しくても、ISE ↔ 外部 RADIUS 間の Shared Secret が不一致であれば外部 RADIUS 側でデータが破棄される点に留意すること。


