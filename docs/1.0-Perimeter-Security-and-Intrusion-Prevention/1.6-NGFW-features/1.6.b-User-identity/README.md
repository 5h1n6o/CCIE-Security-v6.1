---
layout: default
title: 1.6.b-User-identity
nav_order: 2
parent: 1.6-NGFW-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.6.b User identity

Cisco Secure Firewall (FTD) における**ユーザーアイデンティティ（User Identity）**機能は、従来の「IPアドレス」ベースの制御から、Active Directory (AD) や Cisco ISE と連携した「ユーザー/グループ」ベースの制御へとセキュリティポリシーを昇華させる技術です。これにより、BYODや動的なIP割り当て環境下でも、一貫したアクセス制御と可視性を実現します。

---

## 📘 概要

*   **機能概要**: ネットワーク上のトラフィックを特定のユーザー名や所属グループに紐付け、それに基づいたフィルタリングやロギングを行います。
*   **利用目的**: 「人事部のユーザーのみ給与サーバーにアクセス可能にする」といった、業務ロールに基づいた直感的なポリシー定義を実現します。
*   **利用場面**: コンプライアンス遵守のためのユーザー別ログ取得、内部不正の抑止、および複雑なセグメンテーションの簡素化。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **認証方式** | **パッシブ（Passive）**：ユーザーに意識させず情報を取得。 **アクティブ（Active）**：認証画面を表示。 |
| **主要コンポーネント** | Realm（AD設定）, Identity Policy, Identity Source（ISE/AD）。 |
| **メリット** | IPアドレスの変化に影響されない制御、詳細なユーザー別監査ログの生成。 |
| **デメリット** | ADやISEとの連携設定が必要。認証プロセスのオーバーヘッド。 |
| **対応機種** | Firepower Threat Defense (FTD), FMC。 |
| **制限事項** | 1つのルールに多数のユーザーを含めるとパフォーマンスに影響する場合がある。 |
| **設計上の注意点** | **User-IP マッピングの正確性**。共有PC（マルチユーザーホスト）での識別。 |

---

## 🏗 動作原理

ユーザーアイデンティティは、FMCがディレクトリサーバー（AD等）からユーザー情報を取得し、それをFTDのデータプレーンでIPアドレスと照合することで動作します。

```text
[ Active Directory ] <--- (Security Logs) --- [ Cisco AD Agent / ISE ]
          |                                          |
          | (User-IP Mapping)                        | (pxGrid / Syslog)
          ↓                                          ↓
    [ Cisco FMC ] <--------------------------- [ Identity Source ]
          |
          | (Deploy Policy with User Groups)
          ↓
    [ Cisco FTD ] <--- (Traffic Flow: 10.1.1.5) --- [ Client PC (User: Bob) ]
          |
    [ Policy Lookup: User "Bob" is in group "HR" -> Allow ]
```

---

## ⚙ 動作シーケンス

1.  **アイデンティティ情報の収集**: AD Agent または ISE (pxGrid) がドメインコントローラのログを監視し、ユーザー名とログインIPのペアを FMC に通知します。
2.  **Realm の照合**: FMC は受信した情報を、あらかじめ設定された **Realm**（AD連携設定）と照合し、ユーザーがどのグループに属するかを確認します。
3.  **ポリシーへのマッピング**: FMC は FTD に対し、「Group HR = {10.1.1.5, 10.1.2.10...}」という動的なマッピング情報を配布します。
4.  **トラフィック判定**: FTD は着信パケットの送信元IPを確認し、メモリ内のマッピング情報を参照してユーザー/グループを特定後、アクセスコントロールポリシーを適用します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Realm の作成**: ADサーバーのIP、ドメイン名、接続用アカウント（通常は参照権限）の設定。**Test Connection** での疎通確認が必須です。
*   **パッシブ認証 vs アクティブ認証**: 
    *   要件に「Transparent（透過的）」とあればパッシブ（ISE pxGrid/AD Agent）を選択。
    *   要件に「Prompt credentials（入力を求める）」や「Guest access」とあれば、Captive Portal によるアクティブ認証を選択。
*   **Identity Policy の構成**: 特定のネットワークに対して「どの Realm を使用して認証するか」を定義し、ACP に紐付ける手順。
*   **ISE pxGrid 連携**: 証明書のインポート、pxGrid 接続の承認、および SXP 等を使用した TrustSec 連携との違いの理解。
*   **debugコマンド**: FTD CLI から `system support firewall-engine-debug` を使用して、ユーザー情報が正しく Snort に渡っているかを確認するスキル。

---

## 🛠 設定方法

### 1. Realm の作成 (FMC GUI)
1.  **Objects > Object Management > Realms** に移動。
2.  **Add Realm** をクリックし、AD情報を入力。
    *   **Directory**: ADサーバーのIP、ポート 389 (LDAP) または 636 (LDAPS)。
    *   **User/Password**: 参照用アカウント。
3.  **User Download** タブで、ポリシーで使用する特定のグループを選択し、ダウンロードスケジュールを設定。

### 2. Identity Policy の作成
1.  **Policies > Identity** に移動し、**Add Identity Policy** をクリック。
2.  ルールを追加：
    *   **Action**: `Passive Authentication`。
    *   **Realm**: 作成した Realm を選択。
3.  保存してデプロイ。

### 3. Access Control Policy への適用
1.  ACP 編集画面の上部にある **Identity Policy** リンクをクリックし、作成したポリシーを選択。
2.  ACP ルール内で、**Users** タブから Realm 内の特定のユーザーやグループをソースとして指定。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **FTD上のUser-IPマップ確認** | <code>show user-identity-map [active\|all]</code> |
| **Realmの状態確認** | <code>show realm</code> |
| **Snortでの認証判定デバッグ** | <code>system support firewall-engine-debug</code> |
| **LINAでのユーザーコンテキスト確認** | <code>show uauth</code> (Legacy) |
| **FMCでのイベント確認** | Analysis > Users > **Active Sessions** |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| ユーザー名が「Unknown」になる | User-IP マッピングの欠落 | FMCの **Identity Sources** でイベントが受信できているか確認。 |
| AD連携が失敗する (Connection fail) | 認証エラーまたはLDAPポート遮断 | Realm設定のパスワード再入力、または中間FWでの 389/636 ポート許可。 |
| グループ名がリストに現れない | User Download未実行 | Realm設定の **User Download** タブで手動ダウンロードを実行。 |
| アクティブ認証画面が出ない | 証明書またはHTTPリダイレクト不備 | ブラウザのHTTPリダイレクト許可と、FTDのインターフェイス設定を確認。 |

---

## ⚠ 制限事項

*   **サポート対象**: ユーザー名に基づいた制御は、TCP/UDP トラフィックに対してのみ有効です（ICMP等には限定的）。
*   **マルチユーザーホスト**: 1つのIPを複数人で共有する VDI（Citrix/Terminal Server）環境では、専用の TS Agent が必要です。
*   **ラグ**: ユーザーがログインしてからポリシーが適用されるまで、数秒〜数十秒の同期ラグが発生する可能性があります。

---

## 🔄 他技術との関連

*   **ISE (Identity Services Engine)**: pxGrid を介した高度なアイデンティティ共有のメインソース。
*   **SSL Inspection**: アクティブ認証（Captive Portal）を利用する場合、HTTPS トラフィックのリダイレクトのために SSL 復号が必要になる場合があります。
*   **Remote Access VPN**: VPN 接続時に確立されたユーザーセッション情報を、そのままアイデンティティベースの ACP に利用可能です。

---

## 🧩 比較表

### Passive Authentication vs Active Authentication

| 特徴 | Passive Auth (推奨) | Active Auth (Captive Portal) |
| :--- | :--- | :--- |
| **ユーザー体験** | シームレス（気づかない） | ログイン画面での入力が必要 |
| **実装難易度** | 低（AD/ISE連携のみ） | 中（証明書管理、HTTPリダイレクト設定） |
| **情報の正確性** | ADログに依存（ログアウト検知に弱点） | **確実（その場で認証）** |
| **主な用途** | 社内ドメイン環境 | ゲスト、非ドメイン端末、共有PC |

---

## 💡 ベストプラクティス

1.  **冗長ディレクトリ**: Realm 設定で複数の AD ドメインコントローラを登録し、単一障害点を排除します。
2.  **グループベースのルール**: 個別のユーザー名ではなく、AD グループ単位でルールを作成することで、運用負荷を最小限に抑えます。
3.  **アイデンティティタイムアウトの調整**: モバイル端末の頻繁な移動に合わせて、FMC上のセッションタイムアウト値を適切に設定します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的なAD Realmの同期
*   **要件**: AD (10.1.1.100) と連携し、「Sales」グループのユーザーリストを取得せよ。
*   **設定**: Objects > Realms > Add AD Realm。User Download で「Sales」をチェック。

### 2. パッシブ認証ポリシーの作成
*   **要件**: 全ての内部ネットワーク（Inside Zone）に対して、ユーザー名を意識した制御を有効化せよ。
*   **設定**: Identity Policy ルールを追加し、Source Zone: Inside, Action: Passive Auth に設定。

### 3. ISE pxGrid 連携の設定
*   **要件**: ISE から SGT 情報を取得し、ACP で使用可能にせよ。
*   **設定**: System > Integration > Identity Sources > **ISE**。pxGrid サーバーを登録。

### 4. 特定グループに対するWebアクセス許可
*   **要件**: 「IT_Admin」グループのみ、SSH ポートの使用を許可せよ。
*   **設定**: ACP ルール > Users タブ > IT_Admin を追加。Service タブ > SSH を指定。

### 5. キャプティブポータルの構成
*   **要件**: 非ドメイン端末に対して、Webブラウザを介した認証を強制せよ。
*   **設定**: Identity Policy の Action を `Active Authentication` に設定。FTD インターフェイスでリダイレクトを有効化。

### 6. ユーザー名のイベントログ確認
*   **課題**: 通信ログに IP アドレスではなく、実際のユーザー名が表示されていることを確認せよ。
*   **確認**: Analysis > Connections > Events。

### 7. ゲスト用アクティブ認証 (HTTP)
*   **要件**: ゲストゾーンからの全トラフィックに対し、一度だけ認証を求めよ。
*   **設定**: Action: `Active Auth`、ルールの `Identify as Guest` オプションを検討。

### 8. AD Agent との接続トラブル解決
*   **課題**: AD Agent の状態が「Red」になっている原因を特定し解決せよ。
*   **確認**: Agent上のサービス状態と、FMC への 443 疎通。

### 9. 同一IPのユーザー交代への対応
*   **要件**: ユーザーが交代した際に、古いマッピング情報を即座に無効化せよ。
*   **設定**: ISE pxGrid による Logout 通知連携を構成。

### 10. 除外リストの作成
*   **要件**: プリンタやサーバーなどのシステムデバイスは、認証処理から除外せよ。
*   **設定**: Identity Policy の一番上に `No Authentication` ルール（特定のIPアドレス対象）を作成。

---

## ❓ 想定試験問題

1.  **実装**: FMCにおいて、ADグループ「Manager」に属するユーザーのみが特定のURLカテゴリにアクセスできるACPルールを構成せよ。
2.  **トラブルシュート**: Identity Policy をデプロイしたが、Connection Events にユーザー名が表示されない。FTD CLI でマッピングテーブルが空である場合、まずどこを確認すべきか？
    *   **回答**: FMC と Identity Source (ISE/AD Agent) 間の同期状態、および Realm の疎通確認。
3.  **Design**: 透過的なユーザー識別に AD Agent ではなく ISE pxGrid を使用する最大のメリットは何か？
    *   **回答**: SGT (Scalable Group Tags) と連携したコンテキスト情報の共有が可能になり、ログアウト情報の更新も迅速である点。
4.  **実装**: アクティブ認証を使用する際、ユーザーに証明書警告を出さないようにするために FTD 側で必要な準備は？
    *   **回答**: 信頼された CA によって署名された証明書を FTD にインポートし、リダイレクト用サーバー証明書として指定する。
5.  **コンフィグ読解**: `show user-identity-map active` の出力において、複数のユーザー名が1つのIPに紐付いている場合、システムはどう挙動するか述べよ。

---

## 🔗 参考リソース

*   **Configuration Guide**:
    *   [Cisco Secure Firewall Management Center Administration Guide, 7.1 - Identity Policies](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/identity_policies.html)
    *   [Integrated Security Technologies and Solutions Volume II - Network Access Control](https://www.cisco.com/c/en/us/products/collateral/security/identity-services-engine/guide-c07-732448.html)
*   **Technical Notes**:
    *   [Troubleshoot Firepower User Identity (FTD/FMC)](https://www.cisco.com/c/en/us/support/docs/security/firepower-management-center/215354-configure-syslog-on-firepower-firewall-m.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「アイデンティティ」は単独では機能せず、常に **Realm -> Identity Policy -> ACP** という三段構えの紐付けが必要です。このチェーンのどこが欠けてもユーザーベースの制御は失敗します。
*   **図解**: パケットが FTD を通るたびに、Snort が「こいつのIPは誰だ？」とメモリ内の地図（User-IP Map）を引いている様子をイメージしてください。
*   **注意点**: ラボ試験では AD との時刻同期（NTP）が非常に重要です。時刻がずれていると証明書エラーや Kerberos 認証の失敗により Realm 同期が停止します。
