---
layout: default
title: 4.7.a-LDAP
nav_order: 1
parent: 4.7-External-identity
grand_parent: 4.0-Identity-Management
---

# 4.7.a LDAP

Cisco ISE (Identity Services Engine) における **LDAP (Lightweight Directory Access Protocol)** 連携は、Active Directory (AD) 以外のディレクトリサービス（OpenLDAP, Sun Directory, Novell eDirectory 等）や、AD を「ドメイン参加（Join）」させずにクエリベースで利用する場合の主要な外部アイデンティティソース統合手段です。

---

## 📘 概要

*   **機能概要**: ISE が LDAP クライアントとして動作し、外部の LDAP サーバーに対してユーザー検索やパスワード検証のクエリを送信する機能です。
*   **利用目的**: 既存の LDAP ディレクトリに格納されているユーザー情報を使用して、ネットワークアクセスの認証・認可を実現します。
*   **どのような場面で利用するか**: 
    *   非 Windows ベースのディレクトリサービスを使用している環境。
    *   AD への参加権限がない、あるいは運用上の理由でドメイン参加を避けたい場合。
    *   複数の異なる LDAP データベースを併用して認証を行う必要がある場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **接続方式** | **クエリベース**（ドメイン参加は不要）。ISE はディレクトリに対して読み取り専用のアクセスを行う。 |
| **プロトコル / ポート** | LDAP (TCP 389), **LDAPS** (TCP 636 / StartTLS)。 |
| **認証方式** | Simple Bind（ユーザー名とパスワードによるバインド）。 |
| **主なスキーマ** | Active Directory, Sun Java System, OpenLDAP, Novell eDirectory 等。 |
| **属性マッピング** | LDAP 側の属性名（`uid`, `cn` 等）を ISE の辞書にマッピングして認可ポリシーで使用する。 |
| **制限事項** | 通常、パスワード変更やマシン認証（AD特有の機能）のサポートが制限される。 |

---

## 🏗 動作原理

ISE は設定された **Search Base (DN)** を起点として、ツリー構造を検索します。

```text
[ Endpoint ] ──(802.1X/RADIUS)──→ [ Cisco ISE ]
                                     ↓
                 (1) LDAP Bind (Admin/Proxy Account)
                                     ↓
                 (2) Search for User (e.g., uid=jdoe)
                                     ↓
                 (3) Authenticate User (Password Verify)
                                     ↓
                 (4) Fetch Attributes (Groups, Department)
                                     ↓
[ Endpoint ] ←──(Access-Accept)─── [ Cisco ISE ]
```

---

## ⚙ 動作シーケンス

1.  **Connection**: ISE が LDAP サーバーへ TCP 接続を確立します（LDAPS の場合は SSL ハンドシェイクを実施）。
2.  **Binding**: 管理者が指定した **Admin DN** とパスワードを使用して、ディレクトリへのアクセス権限を取得します。
3.  **Search**: サプリカントから送られてきた ID を基に、`Search Base` 以下でユーザーオブジェクトを検索します。
4.  **Verification**: 見つかったユーザーの DN とパスワードを使用して再度バインドを試みるか、ディレクトリ固有の方法で認証を行います。
5.  **Attribute Retrieval**: 認証成功後、認可ポリシーで使用するためにグループ所属情報や特定の属性を抽出します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Search Base の指定**: ユーザーが所属する階層を正しく DN (Distinguished Name) 形式で指定できることが必須です。
*   **Secure LDAP (LDAPS)**: ラボ試験ではセキュリティ向上のため、ポート 636 と **CA 証明書のインポート** をセットで要求される可能性が高いです。
*   **Subject Name Attribute**: `uid` を使うのか `sAMAccountName` を使うのか、ディレクトリの種類に応じた正しいキー選択が求められます。
*   **Identity Source Sequence (ISS)**: 複数の LDAP や内部 DB を組み合わせる際、`Treat as user not found and continue` のフラグ設定による挙動の違いを理解してください。
*   **Group Mapping**: LDAP のグループ（`memberOf` 等）を ISE 側にインポートし、認可ポリシーの条件として適用する手順を確実にしてください。

---

## 🛠 設定方法

### 1. LDAP アイデンティティソースの追加 (GUI)
1.  **Administration > Identity Management > External Identity Sources > LDAP** へ移動。
2.  **General**: 名前を定義し、ディレクトリタイプを選択。
3.  **Connection**: サーバー IP、ポート（389 or 636）、Admin DN、パスワードを入力。
4.  **Directory Organization**: `Search Base`（例: `ou=users,dc=corp,dc=com`）を指定。
5.  **Groups**: LDAP 側からグループを取得し、ISE の内部名と紐付け。
6.  **Attributes**: `department`, `title` などの追加属性を選択。

### 2. 認可ポリシーでの利用
*   認可ポリシーの Condition で `LDAP:ExternalGroups` を選択し、インポートしたグループを指定します。

---

## 🔍 検証コマンド

| 目的 | 手法 / コマンド |
| :--- | :--- |
| **接続とバインドのテスト** | LDAP 設定画面の **Test Connection** ボタンを使用。 |
| **ユーザー検索テスト** | 設定画面の **Test User** 機能で特定の ID を検索。 |
| **ISE 認証ログの確認** | **Operations > RADIUS > Live Logs** でソースを確認。 |
| **CLIでのプロセス確認** | <code>show application status ise</code> (Runtime 等の確認)。 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| Bind 失敗 (Invalid Credentials) | Admin DN またはパスワードの誤り | DN がフルパスで正しく記述されているか再確認。 |
| User Not Found | Search Base が間違っている | ユーザーが存在する最上位の OU を指定しているか確認。 |
| LDAPS 接続不可 | 証明書が信頼されていない | LDAP サーバーの署名 CA を **Trusted Certificates** に入れる。 |
| 属性が取得できない | 属性名のマッピングミス | LDAP スキーマと ISE 側の設定値が一致しているか確認。 |

---

## ⚠ 制限事項

*   **読み取り専用**: ISE から LDAP 上のパスワードをリセットしたり、アカウントをロックしたりすることはできません。
*   **プロトコル制限**: LDAP ソースは Web 認証や 802.1X（PAP/GTC 等）で使用されますが、MS-CHAPv2 は AD 連携時のような高度な互換性がない場合があります。
*   **冗長性**: サーバーリストに複数を登録可能ですが、フェイルオーバーのタイムアウト設定に注意が必要です。

---

## 🔄 他技術との関連

*   **4.1 ISE Personas**: PSN (Policy Service Node) が直接 LDAP サーバーへクエリを投げます。
*   **4.7 Cisco ISE Integration**: AD と LDAP を ISS で組み合わせ、階層的な認証環境を構築します。
*   **2.6 Microsegmentation**: LDAP 属性（部署等）に基づいて SGT を動的に割り当てます。

---

## 🧩 比較表

### LDAP vs Active Directory (Join)

| 特徴 | LDAP (External Identity Source) | Active Directory (Join) |
| :--- | :--- | :--- |
| **実装の容易さ** | 設定がシンプルで早い | DNS, NTP, 管理者権限が必要 |
| **認証プロトコル** | PAP, GTC, EAP-TLS | MS-CHAPv2, PEAP, EAP-TLS |
| **マシン認証** | サポートが困難 | ネイティブにサポート |
| **可用性** | 複数の IP を登録して冗長化 | サイト/ドメインコントローラの自動検出 |

---

## 💡 ベストプラクティス

1.  **専用のサービスアカウント**: LDAP 検索用に、最小権限（読み取りのみ）を持つ専用の DN アカウントを使用します。
2.  **セキュアな通信**: パスワード情報が流れるため、可能な限り **LDAPS (ポート 636)** または StartTLS を使用します。
3.  **タイムアウトの最適化**: ネットワーク遅延がある場合、デフォルトの接続タイムアウト値を調整して認証のタイムアウトを防ぎます。
4.  **Identity Source Sequence の活用**: LDAP サーバーがダウンした場合に備え、内部ユーザー DB をバックアップとしてリストに含めることを検討します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な LDAP 統合
*   **要件**: 10.1.1.50 の LDAP サーバーを Simple Bind で登録せよ。
*   **設定**: Admin DN を `cn=admin,dc=lab,dc=local` に設定。

### 2. LDAPS (ポート 636) の実装
*   **要件**: 証明書を使用して暗号化された LDAP 通信を構成せよ。
*   **操作**: サーバー証明書を ISE にインポート後、ポートを 636 に変更。

### 3. 特定の属性取得
*   **要件**: `employeeType` 属性を抽出し、認可ポリシーで使用可能にせよ。

### 4. Search Base の階層指定
*   **要件**: `OU=Contractors` 以下のみを検索対象とするように DN を設定せよ。

### 5. Subject Name Attribute の変更
*   **要件**: ログイン ID としてメールアドレス (`mail`) を使用するように構成せよ。

### 6. Identity Source Sequence (ISS) での優先順位設定
*   **要件**: 最初に内部 DB、失敗した場合のみ LDAP を参照するようにせよ。

### 7. LDAP グループを用いた VLAN 割り当て
*   **要件**: LDAP グループ "Engineering" に所属するユーザーに VLAN 100 を付与せよ。

### 8. 冗長構成の設定
*   **要件**: 10.1.1.50 (Primary) と 10.1.1.51 (Secondary) の LDAP サーバーを登録せよ。

### 9. バインドパスワードの秘匿性確認
*   **操作**: 設定を保存し、パケットキャプチャを行わずに Test Connection で接続成功を確認。

### 10. 外部 ID ストアとしての認証なしでの利用
*   **要件**: 認証は証明書で行い、属性（グループ）取得のみ LDAP を使用せよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: LDAP サーバーとの接続テストで "Identity Store Unavailable" となる。原因として考えられる設定項目は？
    *   **回答**: LDAP サーバーの IP 誤り、ポート遮断（Firewall）、あるいは Admin DN のパスワード誤り。
2.  **Design**: AD をドメイン参加させずに LDAP として利用する際の欠点は？
    *   **回答**: MS-CHAPv2 などのプロトコルが制限され、マシン認証との統合が複雑になる点。
3.  **実装**: ユーザー ID に `jdoe@corp.com` のような形式を許可するために、LDAP 設定で変更すべき項目は？
    *   **回答**: **Subject Name Attribute** を `mail` または `userPrincipalName` に変更する。
4.  **コンフィグ読解**: `cn=ISE_Service,ou=Apps,dc=example,dc=com` という記述は何を指すか？
    *   **回答**: ISE がディレクトリに接続（Bind）するために使用する **Admin DN**。
5.  **Design**: 大規模環境で LDAP クエリの応答を速めるための ISE 側の設計ポイントは？
    *   **回答**: **Search Base** を可能な限り詳細（深い OU）に設定し、検索範囲を絞り込む。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [外部アイデンティティソース - LDAP の設定](https://www.cisco.com/c/ja_jp/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31.html)
*   **Technical Note**: [Cisco ISE への LDAP アイデンティティ ストアの追加](https://www.cisco.com/c/ja_jp/support/docs/security/identity-services-engine/119143-configure-ise-00.html)
*   **Cisco Live (BRKSEC-3432)**: [Identity Services Engine - Advanced Deployments](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「AD は 1 つの Join、LDAP は複数の Query」とイメージすると、試験での使い分けが明確になります。
*   **図解**: ツリー構造を常に意識してください。`dc=corp,dc=com` は「根」であり、`ou=users` は「枝」です。ISE がどこから「枝探し」を始めるかが Search Base です。
*   **注意点**: ラボ試験で LDAPS を設定する場合、サーバーの FQDN が証明書の CN と一致していないとエラーになるため、DNS と証明書の整合性に注意してください。
