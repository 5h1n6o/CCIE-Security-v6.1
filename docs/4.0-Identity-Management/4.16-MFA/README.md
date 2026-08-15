---
layout: default
title: 4.16-MFA
nav_order: 16
parent: 4.0-Identity-Management
---

# 4.16 Integration of Cisco ISE with multifactor authentication

Cisco ISE (Identity Services Engine) における **多要素認証 (MFA: Multi-Factor Authentication)** 統合は、パスワードの漏洩に対する強力な防御策として機能します。ISE は主に **Cisco Duo** やその他のサードパーティ MFA プロバイダーと連携し、RADIUS プロキシ方式または SAML (Security Assertion Markup Language) 方式を用いて、ユーザーの「身元」を複数の要素で検証します,。

---

## 📘 概要

*   **機能概要**: 知識情報（パスワード）に加えて、所持情報（スマホアプリのプッシュ通知、ハードウェアトークン）や固有情報（生体認証）を組み合わせ、ネットワークアクセスやデバイス管理時の認証を強化する機能です,。
*   **利用目的**: 盗まれた資格情報による不正アクセスの防止、および規制コンプライアンス（PCI-DSS, HIPAA 等）の遵守。
*   **どのような場面で利用するか**:
    *   **リモートアクセス VPN**: AnyConnect 接続時にスマホプッシュで承認。
    *   **デバイス管理 (TACACS+)**: 特権ユーザーがネットワーク機器にログインする際の追加認証。
    *   **ゲストポータル/Web Auth**: 重要なリソースにアクセスする前の二次認証。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要パートナー** | **Cisco Duo** (ネイティブ統合/推奨)。 |
| **統合方式 (Protocol)** | **RADIUS (Proxy/Token)**, **SAML 2.0**, **OAuth2/OpenID Connect**,。 |
| **Duo 連携コンポーネント** | Duo Authentication Proxy (オンプレミス) または Duo Cloud。 |
| **認証フロー** | プライマリ認証 (AD等) 成功後に二次認証を ISE が要求する。 |
| **ユーザー体験** | Push通知 (Duo Push), 電話, 短信 (SMS), OTP, WebAuthン。 |
| **認可への影響** | MFA 成功時のみ特定の SGT や VLAN を付与可能。 |

---

## 🏗 動作原理

ISE と Duo を RADIUS プロキシ方式で連携させた場合の標準的なフローです。

```text
[ Client ]        [ NAD/ASA ]        [ Cisco ISE ]        [ Duo Proxy ]       [ Duo Cloud ]
     |                 |                  |                    |                   |
     |-- (1) Login --->|                  |                    |                   |
     |                 |-- (2) RADIUS --->|                    |                   |
     |                 |       Request    |-- (3) Primary Auth |                   |
     |                 |                  |       (AD/LDAP)    |                   |
     |                 |                  |                    |                   |
     |                 |                  |-- (4) RADIUS Proxy |                   |
     |                 |                  |       to Duo Proxy |                   |
     |                 |                  |------------------->|-- (5) MFA Req --->|
     |                 |                  |                    |                   |
     | <---------------------------------------------------------------- (6) Push --|
     |                 |                  |                    |        to Phone   |
     |-- (7) Approve ------------------------------------------------------------->|
     |                 |                  |                    |                   |
     |                 |                  |<-- (8) RADIUS Acc --|<-- (9) Success ---|
     |                 |<-- (10) Access --|                    |                   |
     |                 |    Accept        |                    |                   |
```

---

## ⚙ 動作シーケンス

1.  **プライマリ認証**: ISE はまず Active Directory 等を使用してユーザー名とパスワードを検証します。
2.  **MFA トリガー**: プライマリ認証成功後、ISE の **Identity Source Sequence (ISS)** または **RADIUS Token** 設定に基づき、MFA リクエストを Duo Proxy 等へ転送します。
3.  **チャレンジ応答**: Duo Proxy は Duo Cloud と連携し、ユーザーのデバイスにプッシュ通知を送ります。
4.  **結果の待機**: ISE は設定されたタイムアウト時間（通常 30〜60秒）応答を待ちます。
5.  **認可の完了**: 二次認証が成功すると、ISE は NAD (Network Access Device) に対して最終的な `Access-Accept` を送信し、通信を許可します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **RADIUS Token Identity Source**: ISE で `RADIUS Token` ソースを作成し、Duo Proxy の IP と共有キーを正しく設定する必要があります。
*   **Identity Source Sequence (ISS)**: MFA を含む ISS を作成する際、プライマリソース（AD等）の後に MFA ソースを配置する順序が重要です。
*   **Timeout 設定**: MFA のプッシュ通知はユーザーの操作を待つため、ISE 側の RADIUS タイムアウト値をデフォルト（5秒）から **30〜60秒** へ大幅に引き上げる必要があります。
*   **SAML 統合**: ポータル認証において SAML IdP として Duo を登録する手順、および証明書のインポートを確認してください。
*   **トラブルシュート（Live Logs）**: MFA 待ちの間にタイムアウトが発生していないか、Live Log の `Steps` で確認する能力が問われます。

---

## 🛠 設定方法

### 1. Duo Authentication Proxy の準備（概要）
*   Duo の管理画面で `RADIUS Auto` アプリケーションを作成し、`ikey`, `skey`, `host` を取得。
*   オンプレミスの Proxy サーバーで `authproxy.cfg` を設定。

### 2. Cisco ISE：RADIUS Token ソースの作成
1.  **Administration > Identity Management > External Identity Sources > RADIUS Token** に移動。
2.  **Add** をクリック。
3.  **General**: 名前（例：Duo_MFA）を入力。
4.  **Connection**: Duo Proxy の IP アドレス、共有シークレット、ポート (通常 1812) を設定。
5.  **Timeout**: サーバータイムアウトを 60 秒に設定。

### 3. Identity Source Sequence の構成
1.  **Administration > Identity Management > Identity Source Sequences** に移動。
2.  **Authentication Profiles**: `Certificate Based` ではなく `Password Based` を選択。
3.  **Identity Sources**: `Active Directory` を上位に、`Duo_MFA` を下位に追加。

---

## 🔍 検証コマンド

| 目的 | 手法 / コマンド |
| :--- | :--- |
| **ISE 認証ログの確認** | **Operations > RADIUS > Live Logs** |
| **MFA 属性の確認** | Live Logs 詳細の `IdentitySelection` および `Authentication` ステップ。 |
| **Duo Proxy ログ確認** | Proxy サーバー上の `authproxy.log` を確認。 |
| **NAD 側でのタイマー確認** | <code>show radius statistics</code> で再送回数を確認。 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 認証が即座に失敗する | 共有キーの不一致 | ISE と Duo Proxy のキーを再入力する。 |
| スマホに Push が届かない | Proxy から Cloud への HTTPS (443) 遮断 | Proxy サーバーから Duo Cloud への疎通を確認。 |
| **"Authentication Timeout"** | ISE のタイムアウト設定が短すぎる | RADIUS Token ソースの Timeout を 60 秒に増加させる。 |
| ユーザーが拒否される | Duo 側にユーザーが未登録 | Duo 管理画面でユーザーの登録・アクティベーション状態を確認。 |

---

## ⚠ 制限事項

*   **プロトコルの制限**: MS-CHAPv2 などの一部のプロトコルは、MFA チャレンジの透過的な処理に制限がある場合があります（PAP/ASCII が推奨されるケースが多い）。
*   **セッションの複雑化**: MFA を有効にすると、ユーザーが 2 回認証情報を入力する（または Push を承認する）必要があるため、利便性とセキュリティのトレードオフが発生します。
*   **オフライン認証**: Duo Cloud にアクセスできない場合、フェイルセーフ設定（Fail Open/Fail Close）の事前検討が不可欠です。

---

## 🔄 他技術との関連

*   **2.1.a FTD Routed Mode (VPN)**: AnyConnect VPN ユーザーに MFA を強制する設計が一般的です。
*   **4.4 802.1X**: 有線/無線の高度なセキュリティ要件として MFA を組み合わせます。
*   **3.10 Cisco DNAC**: SD-Access 環境での管理アクセスに MFA を導入。

---

## 🧩 比較表

### RADIUS Proxy 方式 vs SAML 方式

| 特徴 | RADIUS Proxy (Duo Auth Proxy) | SAML (Duo Cloud IdP) |
| :--- | :--- | :--- |
| **主な用途** | **VPN, CLI (SSH)**, 802.1X | **Web ポータル**, ゲスト, 管理 GUI |
| **複雑さ** | 中（Proxy サーバーが必要） | 高（証明書、XML 連携が必要） |
| **ユーザー体験** | 自動 Push が主流 | ブラウザ上の Duo 画面で選択 |
| **認証強度** | プライマリ認証と結合 | 完全に分離された認証フロー |

---

## 💡 ベストプラクティス

1.  **専用ポリシーセット**: MFA を適用するトラフィックを特定のポリシーセット（例：Admin_Access）に限定し、一般ユーザーの負荷を下げます。
2.  **フェイルモードの設定**: Duo Proxy 側で `failmode = safe` を設定することで、MFA サービス停止時にプライマリ認証のみでログインを許可する（可用性優先）か検討します。
3.  **AnyConnect プロファイル**: `Authentication Timeout` を VPN クライアント側でも適切に調整し、プッシュ承認中に接続が切れないようにします。

---

## 📝 ラボ学習・設定サンプル例

### 1. ISE RADIUS Token ソースの作成
*   **要件**: 10.1.1.50 の Duo Proxy を使用し、ポート 1812、キー `cisco123` で登録せよ。

### 2. MFA 用 ISS の構築
*   **要件**: `Internal Users` で認証後、失敗したら `Duo_MFA` を呼び出すシーケンスを作成せよ。

### 3. ASA VPN MFA の構成
*   **要件**: AnyConnect ユーザーに対し、ISE を介して Duo MFA を適用せよ。
*   **設定**: `aaa-server` に ISE を指定。

### 4. 特権ログイン (TACACS+) への MFA 追加
*   **要件**: スイッチへの `enable` ログイン時に Duo MFA を要求せよ。

### 5. ISE 管理者 GUI への SAML MFA
*   **要件**: ISE 管理画面ログインを Duo SAML IdP へリダイレクトせよ。

### 6. MFA タイムアウトの調整 (ISE)
*   **問題**: プッシュ承認中に ISE が Reject を出す。
*   **対処**: Server Timeout を 60 秒に変更。

### 7. 特定の AD グループのみ MFA 強制
*   **要件**: `Domain Admins` グループのみ MFA を含む ISS を使用せよ。

### 8. Duo Push 失敗時の OTP 入力
*   **シナリオ**: Push が届かないユーザーがパスワード末尾に `,123456` (OTP) を付与して認証する設定の確認。

### 9. 証明書認証 (L1) + Duo MFA (L2)
*   **要件**: EAP-TLS マシン認証成功後、ユーザーに MFA を要求せよ。

### 10. ライブログでの MFA ステップ詳細確認
*   **操作**: 正常な MFA 完了時の `Live Log` から、どの Identity Source が最終的にパスしたか読み取れ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: ISE の `RADIUS Token` 設定で `Server Timeout` が 5 秒のままになっている。プッシュ通知運用でどのような問題が起きるか？
    *   **回答**: ユーザーがスマホを操作する前に ISE がタイムアウトし、認証が失敗する。
2.  **トラブルシュート**: Live Log に `11001 RADIUS-Token-Server: No response from server` と表示される。確認すべき項目は？
    *   **回答**: ISE と Duo Proxy 間の IP 到達性、および Duo Proxy サービスが起動しているか。
3.  **Design**: セキュリティ要件として「証明書が正当でも、MFA 承認がなければアクセス不可」とするために必要な ISE の設定箇所は？
    *   **回答**: **Authorization Policy** の条件（Condition）に、MFA 成功を示す属性を含めるか、ISS で MFA を必須にする。
4.  **Design**: WAN 帯域が極端に狭い環境で MFA を導入する際の懸念事項は？
    *   **回答**: RADIUS パケットの遅延によるタイムアウト。再送タイマー（Retransmit）の調整が必要。
5.  **実装**: Duo Proxy 方式で、パスワード入力後に自動でプッシュ通知を送る設定を Duo Proxy 側でする場合、使用するアプリケーションタイプは？
    *   **回答**: `radius_auto`。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [RADIUSトークン外部アイデンティティソース](https://www.cisco.com/c/ja_jp/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31.html)
*   **Cisco Duo Documentation**: [Cisco ISE Duo Integration Guide](https://duo.com/docs/cisco-ise)
*   **Configuration Example**: [Configure ISE 2.x for Duo MFA](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/200632-Configure-ISE-2-0-and-Duo-Authentication.html)
*   **Cisco Live (BRKSEC-2041)**: [Advanced Identity and MFA with Cisco ISE](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「MFA は ISE の外部 ID ソースの一部」として捉えると、ポリシー作成がスムーズになります。
*   **図解**: `Client -(RADIUS)-> ISE -(RADIUS Proxy)-> Duo Proxy -(HTTPS)-> Duo Cloud` というプロトコルの変化を意識してください。
*   **注意点**: ラボ試験では、**AD 連携が正常であることが前提**となります。AD が壊れていると、その後の MFA プロセスまで到達しません。まずはプライマリ認証の Live Log を確認しましょう。
