---
layout: default
title: 4.17-DUO
nav_order: 17
parent: 4.0-Identity-Management
---

# 4.17 Access control and single sign-on using Cisco DUO security technology

Cisco Duo は、クラウドベースのセキュリティプラットフォームであり、**多要素認証 (MFA)**、**シングルサインオン (SSO)**、および**アクセス制御**を提供します。CCIE Security v6.1 において、項目 4.17 は単なる MFA ではなく、Duo を Identity Provider (IdP) として使用した SAML ベースの SSO 実装や、デバイスの健全性（Device Health）に基づいた動的なアクセス制御ポリシーの構築に焦点を当てています。

---

## 📘 概要

*   **機能概要**: Duo SSO は、クラウド上の Duo を IdP とし、SAML 2.0 または OpenID Connect (OIDC) を使用して複数のアプリケーションへのログインを一本化します。これに Duo の強力なポリシーエンジンを組み合わせることで、ユーザー、デバイス、および場所に基づいた詳細なアクセス制限を適用します。
*   **利用目的**: パスワード管理の簡素化（SSO）、ゼロトラストアクセス（信頼されたデバイスのみ許可）、および一貫したログイン体験の提供。
*   **どのような場面で利用するか**:
    *   **リモートアクセス VPN**: AnyConnect の認証を SAML で Duo SSO へリダイレクトし、ログインと MFA、デバイスチェックを一度に行う。
    *   **管理プレーンの保護**: FMC, ISE, Cisco DNA Center へのログインを Duo SSO で保護し、管理者権限の濫用を防ぐ。
    *   **クラウド・オンプレアプリの統合**: Office 365, AWS, 内部 Web サーバーへのアクセスを Duo 中央ポータル（Duo Central）から提供。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **Duo SSO (IdP)** | Duo が SAML 応答を生成する、フルマネージドな SSO サービス。 |
| **Duo Central** | ユーザーが全アプリにアクセスできるカスタマイズ可能なランチャーポータル。 |
| **Access Policy** | ユーザーグループ、OS バージョン、ブラウザ、場所に基づく制御。 |
| **Trusted Endpoints** | Duo デバイス証明書や管理状態を確認し、未管理デバイスをブロックする機能。 |
| **Duo Device Health** | 端末の暗号化状態、OS パッチ、ファイアウォール有効化などをチェック。 |
| **Identity Store** | Active Directory (オンプレ) または Azure AD, ISE を認証ソースとして利用。 |

---

## 🏗 動作原理

Duo SSO を使用した標準的な SAML 認証フロー（Service Provider 始動）を以下に示します。

```text
[ User ]        [ ASA/FMC (SP) ]      [ Duo SSO (IdP) ]      [ Identity Store ]
    |                 |                    |                     |
    |--- (1) Access ->|                    |                     |
    |                 |-- (2) SAML Req --->|                     |
    |                 |      (Redirect)    |                     |
    |                 |                    |--- (3) Auth Check ->|
    |                 |                    |<-- (4) Success -----|
    |                 |                    |                     |
    |                 |<-- (5) MFA & Device Health Check --------|
    |--- (6) Approve --------------------->|                     |
    |                 |                    |                     |
    |                 |<-- (7) SAML Resp --|                     |
    |--- (8) Granted ->|      (Token)       |                     |
```

---

## ⚙ 動作シーケンス

1.  **SP へのアクセス**: ユーザーが VPN や管理 GUI にアクセス。
2.  **リダイレクト**: デバイス（ASA 等）がユーザーを Duo SSO ポータルにリダイレクト。
3.  **プライマリ認証**: Duo が既存の AD や ISE に問い合わせ、資格情報を検証。
4.  **ポリシー評価**: Duo がアクセスルール（場所は日本か？デバイスは会社支給か？）を評価。
5.  **MFA & ヘルスチェック**: スマホプッシュ通知の承認と、Duo Device Health アプリによる端末スキャンを実行。
6.  **SAML 応答**: すべての条件を満たした場合、Duo が署名済み SAML アサーションをデバイスに発行。
7.  **アクセス許可**: デバイスがアサーションを検証し、セッションを確立。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **AnyConnect SAML Auth**: ASA 上で `tunnel-group` の認証方式を `saml` に設定し、Duo SSO のメタデータをインポートする設定は最重要です。
*   **FMC SSO 管理**: FMC の `System > Configuration > Management SSO` で Duo を登録する手順。
*   **証明書の管理**: Duo SSO とデバイス間での信頼関係構築のため、Duo からダウンロードした IdP 証明書を ASA/FMC に正しくインポートする必要があります。
*   **Entity ID の不一致**: ラボでの典型的なミスとして、Duo 管理画面と ASA/FMC 側で Entity ID や ACS URL が 1 文字でも違うと認証がループまたは失敗します。
*   **Duo Connect (DAG/SSO) 移行**: レガシーな Duo Access Gateway (DAG) ではなく、最新の **Duo SSO (Cloud IdP)** を使用する構成が求められます。

---

## 🛠 設定方法

### 1. Cisco ASA：SAML 設定例 (CLI)
```bash
! Duo SSO 用の Trustpoint 作成
crypto ca trustpoint Duo_SSO_IdP
 enrollment terminal
 crypto ca authenticate Duo_SSO_IdP
 ! [Duoからダウンロードした証明書をペースト]

! SAML IdP の定義
webvpn
 saml idp https://sso-abc12345.duosecurity.com/saml2/sp/DI...
  url request sig-and-enc https://sso-abc12345.duosecurity.com/portal/saml/login
  url response sig-and-enc https://sso-abc12345.duosecurity.com/portal/saml/login
  trustpoint idp Duo_SSO_IdP
  trustpoint sp [ASA_CERT]
  no force re-authentication

! トネルグループへの適用
tunnel-group DUO_VPN_TG webvpn-attributes
 authentication saml
 saml identity-provider https://sso-abc12345.duosecurity.com/saml2/sp/DI...
```

### 2. Duo Admin Panel 側の構成 (GUI)
1.  **Applications**: `Cisco ASA VPN` (または Firepower) を追加。
2.  **Service Provider**: ASA の外部 FQDN と `Assertion Consumer Service (ACS) URL` を入力。
3.  **Policy**: `Device Health` を "Required" に設定。

---

## 🔍 検証コマンド

| 目的 | コマンド / 手法 |
| :--- | :--- |
| **SAML セッションの確認 (ASA)** | <code>show vpn-sessiondb detail saml</code> |
| **SAML デバッグ (ASA)** | <code>debug webvpn saml 255</code> |
| **Duo 認証ログの確認** | Duo Admin Panel > **Reports > Authentication Logs** |
| **FMC SSO 状態確認** | FMC CLI: <code>cat /var/log/httpd/httpsd_error_log</code> |
| **メタデータ確認** | ブラウザで IdP メタデータ URL にアクセスし XML が表示されるか確認。 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| **認証ループが発生する** | Entity ID の不一致 | Duo 管理画面と ASA 設定の Entity ID が完全一致しているか確認。 |
| **"Invalid SAML response"** | 時刻（NTP）のズレ | ASA/FMC と Duo Cloud の時刻誤差が 300 秒以内であることを確認。 |
| **Device Health 失敗** | アプリ未インストール | ユーザー端末に Duo Device Health アプリが起動しているか確認。 |
| **IdP 接続エラー** | DNS 解決不可 | ASA から Duo SSO の FQDN が名前解決できるか <code>ping</code> で確認。 |

---

## ⚠ 制限事項

*   **AnyConnect バージョン**: SAML 認証を完全にサポートするには、AnyConnect 4.6 以降が必要です。
*   **ブラウザ依存**: AnyConnect の外部ブラウザ認証を有効にする必要がある場合があります（`saml external-browser`）。
*   **オフライン認証**: Duo SSO はクラウドサービスのため、インターネット接続が失われると認証が不可となります（フェイルセーフ設定の検討が必要）。

---

## 🔄 他技術との関連

*   **2.1.a FTD Routed Mode**: VPN の認証バックエンドとして Duo SSO を利用。
*   **4.2 Network Access AAA**: ISE が Duo SSO のプライマリ認証ソースとなり、SAML クエリを処理する構成。
*   **2.6 Microsegmentation**: Duo が SAML 属性に SGT (Security Group Tag) 名を含め、デバイスがそれを認可に使用する高度な連携。

---

## 🧩 比較表

### Duo SSO vs Duo Authentication Proxy (RADIUS)

| 特徴 | Duo SSO (SAML) | Duo Auth Proxy (RADIUS) |
| :--- | :--- | :--- |
| **認証プロトコル** | **SAML 2.0 / OIDC** | RADIUS / LDAP |
| **ユーザー体験** | **インライン Web 画面**（モダン） | ポップアップまたはパスワード末尾入力 |
| **デバイスチェック** | **詳細なヘルスチェックが可能** | 限定的（MFAのみ） |
| **推奨用途** | **クラウド/Web アプリ, AnyConnect** | 旧来の CLI (SSH), 厚いクライアント |

---

## 💡 ベストプラクティス

1.  **外部ブラウザの使用**: AnyConnect 認証時に `external-browser` を使用することで、OS 標準のブラウザ（および保持されている認証セッション）を利用でき、ユーザー体験が向上します。
2.  **証明書ピンニングの回避**: IdP 証明書は期限があるため、自動更新が可能な構成か、監視体制を整えます。
3.  **信頼されたデバイスの構成**: 証明書ベースの `Trusted Endpoints` を構成し、個人の私物端末（BYOD）を完全にブロックするポリシーを適用します。

---

## 📝 ラボ学習・設定サンプル例

### 1. AnyConnect SAML 基本構成
*   **要件**: ASA VPN ログインに Duo SSO を使用せよ。
*   **設定**: `authentication saml` + `saml identity-provider`。

### 2. IdP 証明書のインポート
*   **要件**: Duo からエクスポートした PEM 証明書を ASA の Trustpoint に登録せよ。

### 3. FMC SSO 管理アクセスの有効化
*   **要件**: FMC GUI へのログインを Duo SSO で保護せよ。

### 4. Duo SSO と ISE 連携
*   **要件**: Duo SSO のプライマリ認証を、SAML IdP として動作する ISE に委譲せよ。

### 5. 特定の国からのアクセスブロック
*   **要件**: Duo Access Policy で、日本以外からの認証要求を Reject せよ。

### 6. デバイス健全性（Firewall）の強制
*   **要件**: 端末の Firewall がオフの場合、AnyConnect 接続を拒否せよ。

### 7. AnyConnect 外部ブラウザ認証の設定
*   **コマンド**: `tunnel-group [name] webvpn-attributes` > `saml external-browser enable`。

### 8. Duo Central アプリケーションランチャーの作成
*   **要件**: ユーザーが 1 クリックで FMC と ISE GUI にアクセスできるポータルを構築せよ。

### 9. SAML 属性マッピング (SGT)
*   **要件**: Duo から返される SAML 属性を、ISE で SGT に変換せよ。

### 10. パスワードレス認証の構成
*   **操作**: WebAuthn/FIDO2 認証器を使用して、パスワードなしで VPN 接続を確立せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: ASA の `saml identity-provider` 設定で `url response sig-and-enc` が指定されている場合、ASA は何を期待しているか？
    *   **回答**: Duo から送られる SAML レスポンスが、**署名され、かつ暗号化されていること**。
2.  **トラブルシュート**: Duo SSO で認証後、ASA に戻ると "Internal Error" になる。デバッグで "Signature verification failed" と出る。原因は？
    *   **回答**: ASA に登録した **IdP Trustpoint の証明書**が、現在の Duo SSO の署名鍵と一致していない。
3.  **Design**: AnyConnect ユーザーが 24 時間に一度だけ MFA を求められるようにしたい。どこで設定すべきか？
    *   **回答**: **Duo Access Policy** の `Remembered Devices` 設定、または ASA の `saml force-reauthentication` 無効化。
4.  **実装**: FMC で SSO を有効にした後、管理者がロックアウトされた場合の回避策は？
    *   **回答**: `https://[FMC_IP]/login.cgi?saml=false` (または類似のローカルログイン URL) を使用して、SSO をバイパスしてローカルアカウントで入る。
5.  **Design**: 信頼された会社支給デバイスのみを許可するための最も確実な Duo の機能は？
    *   **回答**: **Trusted Endpoints** (証明書ベースの検証)。

---

## 🔗 参考リソース

*   **Duo Documentation**: [Cisco ASA SSL VPN with Duo SSO](https://duo.com/docs/cisco-asa-sso)
*   **Cisco Config Guide**: [ASA SAML 2.0 Integration](https://www.cisco.com/c/en/us/td/docs/security/asa/asa914/configuration/vpn/asa-914-vpn-config/vpn-saml.html)
*   **Cisco Live (BRKSEC-2041)**: [Leveraging Duo for Zero Trust Access](https://www.ciscolive.com/)
*   **Design Guide**: [FMC Management Single Sign-On](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/management_sso.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「Duo SSO はゲートウェイ、Duo Admin Panel は脳」です。ASA や FMC は単に指示に従うだけの腕に過ぎません。
*   **図解**: 
    - Duo Admin Panel (Configure SP) <--> Metadata Exchange <--> ASA/FMC (Configure IdP).
*   **注意点**: ラボ試験では、**メタデータの XML を手動で修正する**（ASA の制約等でインポートできない場合）スキルが必要になることがあるため、SAML メタデータの構造（Location, Binding 等）を理解しておきましょう。
