---
layout: default
title: 4.7-External-identity
nav_order: 7
parent: 4.0-Identity-Management
---

# 4.7 Cisco ISE integration with external identity sources

Cisco ISE (Identity Services Engine) の核心的な機能の一つは、内部データベースだけでなく、外部のアイデンティティソースと連携してユーザーやデバイスを認証・認可することです。CCIE Security v6.1 において、この項目は Active Directory (AD)、LDAP、SAML、ODBC、および外部 RADIUS サーバとの統合に関する深い理解と、それらを「Identity Source Sequence」を用いて柔軟に制御する能力を求めています。

---

## 📘 概要

*   **機能概要**: ISE を Microsoft Active Directory、LDAP ディレクトリ、SAML アイデンティティプロバイダー (IdP) などの外部リポジトリに接続し、資格情報の検証や属性情報の取得を行う機能です。
*   **利用目的**: 既存の企業内ユーザーデータベースをそのまま活用し、パスワードの一元管理やシングルサインオン (SSO) を実現します。
*   **どのような場面で利用するか**: 
    *   社内 PC の 802.1X 認証（AD ユーザーによるログイン）。
    *   ゲストポータルや My Devices ポータルでの SAML ベースの SSO 認証。
    *   マニュファクチャリング環境やレガシー環境での LDAP/ODBC データベース連携。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要ソース** | Active Directory, LDAP, SAML, ODBC, RADIUS Token (RSA/OTP), External RADIUS。 |
| **AD 連携方式** | ドメインへの「参加 (Join)」が必要。コンピュータアカウントを作成して対話する。 |
| **LDAP 連携方式** | 「クエリ (Query)」ベース。ISE はクライアントとして LDAP サーバーを検索する。 |
| **SAML 連携方式** | フェデレーション。ポータル認証（Guest, BYOD, Sponsor）に限定される。 |
| **Identity Source Sequence (ISS)** | 複数のソースをリスト化し、上から順に認証を試行するロジック。 |
| **属性の取得** | 認証後に AD/LDAP からグループ情報やカスタム属性を取得し、認可ポリシーの条件に使用する。 |

---

## 🏗 動作原理

ISE は外部ソースに対して異なるプロトコルを使用して対話します。

```text
  [ User/Supplicant ] 
          ↓ (RADIUS/HTTPS)
  [ Cisco ISE ]
          ↓
  +-------+-------+-------+
  ↓       ↓       ↓       ↓
[ AD ]  [LDAP]  [SAML]  [RADIUS]
(RPC)   (LDAP)  (HTTPS) (RADIUS)
```

1.  **Active Directory**: ISE は MS-RPC および NetLogon プロトコルを使用して AD ドメインコントローラと通信します。
2.  **LDAP**: ISE は標準的な LDAP ポート (389/636) を使用し、バインド操作によってユーザーを検索・検証します。
3.  **SAML**: ユーザーブラウザを介して IdP へリダイレクトし、アサーション（証明書付きの応答）を受け取って認可します。

---

## ⚙ 動作シーケンス (Active Directory Join)

1.  **DNS 解決**: ISE がドメイン名の SRV レコードを解決し、DC を特定します。
2.  **NTP 同期**: ISE と AD の時刻差が 5 分以内であることを確認します（Kerberos 認証に必須）。
3.  **ドメイン参加**: 管理者資格情報を使用して、AD 内に ISE のコンピュータアカウントを作成します。
4.  **信頼構築**: ISE と AD 間でセキュアなチャネルが確立されます。
5.  **属性取得**: 認証時、ISE は `ms-RPC` を通じてユーザーの `MemberOf` 属性などを取得します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **AD Join の前提条件**: ラボ試験では、DNS サーバの設定不足や NTP の同期ズレで AD Join に失敗するシナリオが想定されます。最初に CLI で `show ntp` および `nslookup` を確認してください。
*   **Identity Source Sequence (ISS) の優先順位**: 「特定の MAC アドレスは内部 DB で、それ以外は AD で認証せよ」といった要件に対し、正しい順序で ISS を構成する能力が問われます。
*   **SAML のメタデータ交換**: SAML IdP との間で XML メタデータファイルをインポート/エクスポートする手順を習得してください。
*   **Certificate Authentication Profile (CAP)**: 証明書認証において、どのフィールド（SAN, CN 等）を AD の `sAMAccountName` と比較するか（Binary Comparison）の設定は頻出です。
*   **LDAP スキーマのマッピング**: 外部 LDAP サーバーの属性名（例：`uid`, `mail`）を ISE の内部辞書に正しくマッピングする設定が必要です。

---

## 🛠 設定方法

### 1. Active Directory への参加 (GUI)
1.  **Administration > Identity Management > External Identity Sources > Active Directory** に移動します。
2.  **Add** をクリックし、`Join Point Name` と `Active Directory Domain` を入力します。
3.  **Submit** すると、ドメイン参加のための資格情報を求められます。
4.  参加成功後、**Groups** タブで、ポリシーに使用したい AD グループを選択して追加します。

### 2. Identity Source Sequence (ISS) の作成
1.  **Administration > Identity Management > Identity Source Sequences** に移動します。
2.  **Add** をクリックし、認証対象のソース（Internal Endpoints, AD, LDAP等）を選択して順番を決定します。
3.  `Select Certificate Authentication Profile` で証明書ベースの認証設定を紐付けます。

---

## 🔍 検証コマンド

| 目的 | コマンド / 手法 |
| :--- | :--- |
| **AD との接続テスト** | ISE GUI の Active Directory 設定画面にある **Test User** ボタン。 |
| **AD Join 状態の CLI 確認** | <code>show application status ise</code> (AD Connector プロセスを確認)。 |
| **DNS 解決の確認** | <code>show tech-support</code> 内の resolv.conf または CLI で <code>nslookup [Domain]</code>。 |
| **認証ライブログ** | **Operations > RADIUS > Live Logs** でどのソースで成功/失敗したかを確認。 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| AD Join が "Clock skew" で失敗 | ISE と AD の時刻同期失敗 | <code>show ntp</code> で同期を確認し、AD と同じ NTP を参照させる。 |
| AD Join が "Access Denied" | AD 側の管理者権限不足 | コンピュータアカウント作成権限を持つユーザーを使用する。 |
| LDAP 認証が "User not found" | 検索ベース DN の誤り | `Search Base` 設定がユーザーの存在する階層をカバーしているか確認。 |
| SAML リダイレクトが失敗 | 証明書の信頼欠如 | IdP の署名用証明書を ISE の **Trusted Certificates** に追加する。 |

---

## ⚠ 制限事項

*   **AD Multi-Forest**: 双方向の信頼関係がないフォレストを複数追加する場合、フォレストごとに個別の Join Point が必要です。
*   **SAML の用途制限**: SAML は RADIUS 認証（802.1X）には使用できず、Web ベースのポータル認証のみをサポートします。
*   **LDAP 書き込み**: ISE は LDAP に対しては読み取り専用 (Read-only) であり、パスワード変更などの書き込み操作は制限されます。

---

## 🔄 他技術との関連

*   **4.2 Network Access AAA**: スイッチや WLC からの RADIUS 要求を、外部ソースに中継して判定します。
*   **4.6 BYOD Flow**: オンボーディング時の初回認証に AD の資格情報を使用します。
*   **4.5 Guest Lifecycle**: ゲストアカウントの承認を行う「スポンサー」の認証に AD グループを使用します。

---

## 🧩 比較表

### AD vs LDAP (ISE Integration)

| 特徴 | Active Directory (Join) | LDAP (Query) |
| :--- | :--- | :--- |
| **接続性** | ネイティブ参加 (MS-RPC) | 標準 LDAP クエリ (389/636) |
| **複雑さ** | 高（DNS, NTP, 権限が必要） | 低（単なる検索） |
| **パスワード変更** | サポートされる | 通常はサポートされない |
| **複数ドメイン** | 信頼関係があれば容易 | 複雑なマッピングが必要 |
| **主な用途** | Windows 企業環境 | サードパーティ/レガシー DB |

---

## 💡 ベストプラクティス

1.  **冗長 DC の指定**: ISE の AD 設定で、特定の DC だけでなくドメイン全体を指定し、DC 障害時の自動切り替えを有効にします。
2.  **Identity Source Sequence の最適化**: 認証の失敗時間を短縮するため、最も利用頻度の高いソースを ISS の先頭に配置します。
3.  **バイナリ比較の有効化**: EAP-TLS 認証時、証明書の `CN` が AD の `sAMAccountName` と一致するか確認する `Binary Comparison` を有効にし、セキュリティを高めます。
4.  **AD グループのフィルタリング**: 不要な AD グループをインポートせず、ポリシーに必要なものだけを選択してパフォーマンスを維持します。

---

## 📝 ラボ学習・設定サンプル例

### 1. Active Directory Join 構成
*   **問題**: ISE を `corp.local` ドメインに参加させよ。
*   **要件**:
    1. NTP サーバ 10.1.1.1 を使用すること。
    2. ドメイン管理者の資格情報 `admin / Cisco123` を使用すること。
*   **設定例 (GUI)**:
    - **System > Settings > NTP**: 10.1.1.1 を追加し、Sync 状態を確認。
    - **Identity Management > Active Directory**: ドメイン名を入力し、`Join` ボタンから admin でログイン。

### 2. LDAP アイデンティティソースの追加
*   **問題**: 10.1.1.50 の LDAP サーバーをソースとして追加せよ。
*   **要件**: 
    - スキーマは `Active Directory` 互換を使用。
    - `Subject Name Attribute` を `sAMAccountName` に設定。

### 3. Identity Source Sequence (ISS) の複合要件
*   **問題**: 以下の順序で認証を行う ISS を作成せよ。
    1. Internal Users
    2. Active Directory
*   **要件**: 1 でユーザーが見つからない場合のみ 2 を実行すること。

### 4. 証明書認証プロファイル (CAP)
*   **要件**: クライアント証明書の `Subject Alternative Name (SAN)` フィールドを使用して AD 検索を行え。

### 5. SAML SSO ポータル連携
*   **要件**: ゲストポータルの認証に外部 IdP (Azure AD 等) の SAML を使用せよ。

### 6. ODBC ソースによる外部 DB 連携
*   **操作**: 外部 SQL Server に接続するための DSN 構成と属性マッピングを行え。

### 7. AD 属性取得 (Attribute Fetching)
*   **要件**: AD の `department` 属性を取得し、認可ポリシーで使用可能にせよ。

### 8. パッシブ ID (PassiveID) の構成
*   **問題**: AD の WMI ログを監視し、認証なしでユーザーの IP-User マッピングを学習せよ。

### 9. 外部 RADIUS プロキシの構成
*   **要件**: 特定のユーザー要求を外部 RADIUS サーバー (192.168.1.100) に転送せよ。

### 10. AD 接続障害のシミュレーション
*   **課題**: AD Join Point が `Down` 状態になった際、ISS のフォールバックにより内部 DB 認証が機能することを確認せよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: ISE が AD ドメインに参加しようとすると "DNS Error" が発生する。最初に確認すべき CLI コマンドは？
    *   **回答**: `nslookup [ドメイン名]`。SRV レコードが引けるか確認する。
2.  **Design**: RADIUS 認証（802.1X）において、SAML IdP を認証ソースとして直接使用できるか？
    *   **回答**: できない。SAML は Web ベースのポータル認証（HTTPS）にのみ対応している。
3.  **コンフィグ読解**: ISS 設定で `Treat as user not found and continue to next identity store` オプションの役割は？
    *   **回答**: 最初のソースにユーザーが存在しなかった場合（Rejected ではなく Not Found の場合）、次のソースで認証を継続する。
4.  **Design**: Kerberos 認証が正常に動作するために、ISE と AD 間の許容される最大時刻差は？
    *   **回答**: **5 分**。これを超えると Kerberos チケットが無効になる。
5.  **実装**: 証明書認証において、AD 内のユーザーオブジェクトと証明書を厳密に 1 対 1 で照合する設定項目は何か？
    *   **回答**: Certificate Authentication Profile 内の **Binary Comparison**。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [外部アイデンティティソースの設定](https://www.cisco.com/c/en/us/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31/b_ise_admin_guide_31_chapter_010.html)
*   **Cisco Live (BRKSEC-2041)**: [Active Directory Integration with Cisco ISE](https://www.ciscolive.com/)
*   **Technical Note**: [Troubleshoot Active Directory Join Issues in Cisco ISE](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/118837-technote-ise-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「AD は Join が必要、LDAP は Query のみ」という基本姿勢を徹底してください。
*   **図解**: 
    - **Identity Source**: 名簿（AD, LDAP, SAML）。
    - **Identity Source Sequence**: 名簿を調べる「順番」。
*   **注意点**: ラボ試験では AD のコンピュータアカウントを作成する十分な権限がユーザーに付与されていないことがあります。その場合は、事前に AD 側でアカウントを手動作成し、ISE から「既存アカウントの使用」を試みる必要があります。
