---
layout: default
title: 4.7.b-AD
nav_order: 2
parent: 4.7-External-identity
grand_parent: 4.0-Identity-Management
---

# 4.7.b AD (Active Directory)

Cisco ISE (Identity Services Engine) における **Active Directory (AD)** 統合は、エンタープライズネットワークにおいて最も一般的かつ重要な外部アイデンティティソース連携です。ISE は AD ドメインに「参加（Join）」することで、AD を単なる読み取り専用データベースとしてだけでなく、ネイティブな認証プロトコル（MS-CHAPv2 や Kerberos）をサポートする強力な認証基盤として利用します。

---

## 📘 概要

*   **機能概要**: Cisco ISE ノードを Microsoft Active Directory ドメインの「コンピュータオブジェクト」として登録し、AD サーバーと通信してユーザーおよびマシン情報の検証、グループ属性の取得を行う機能です。
*   **利用目的**: 既存の Windows 資産（ユーザー/マシンアカウント）をそのままネットワークアクセスポリシーに再利用し、シングルサインオン (SSO) や透過的な認証を実現します。
*   **場面**: 企業内 LAN/WLAN での 802.1X 認証、VPN 接続、BYOD オンボーディング、およびデバイス管理アクセス (TACACS+)。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **接続方式** | **ネイティブ参加 (Domain Join)**。ISE がドメイン内のコンピュータとして振る舞う。 |
| **通信プロトコル** | MS-RPC, NetLogon, LDAP (属性取得用), Kerberos (認証用)。 |
| **認証方式** | MS-CHAPv2, PEAP, EAP-FAST, EAP-TLS (Binary Comparison)。 |
| **前提条件** | 正確な **DNS 解決** と **NTP 同期** (最大許容誤差 5分)。 |
| **マルチドメイン** | 双方向の信頼関係があるフォレスト/ドメインをサポート。信頼関係がない場合は個別に Join が必要。 |
| **属性の利用** | `MemberOf` (グループ), `UserAccountControl`, カスタム属性を認可条件に使用可能。 |

---

## 🏗 動作原理

ISE は AD と対話するために **MS-RPC** を使用します。LDAP 連携とは異なり、ISE はドメインコントローラ (DC) を自動的に検出し、セキュアなチャネルを確立します。

```text
[ Endpoint ] ──── (802.1X/RADIUS) ───→ [ Cisco ISE ]
                                           ↓
                 (1) DNS SRV Query (Locate DC)
                                           ↓
                 (2) MS-RPC / NetLogon (Authentication)
                                           ↓
                 (3) LDAP Query (Fetch memberOf / Attributes)
                                           ↓
[ Endpoint ] ←─── (Access-Accept) ──── [ Cisco ISE ]
```

---

## ⚙ 動作シーケンス (ISE Join Process)

1.  **DNS 解決**: ISE がドメイン名から SRV レコードを検索し、利用可能なドメインコントローラを特定します。
2.  **時刻同期**: NTP を使用して ISE と AD の時刻を同期させます（Kerberos の制約）。
3.  **ドメイン参加**: AD 管理者の資格情報を使用して、AD 内の `Computers` コンテナ（または指定 OU）に ISE のコンピュータアカウントを作成します。
4.  **サービス接続**: Join 完了後、ISE の `Runtime` プロセスが AD サービスと対話を開始します。
5.  **グループ取得**: 管理者はポリシーに使用する特定の AD グループを選択して ISE 内に取り込みます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **前提条件の確認**: ラボ環境では **NTP の同期** が崩れているケースが非常に多いです。最初に `show ntp` を確認し、AD と同じソースを参照しているか見てください。
*   **DNS 構成**: ISE の構成で正しい DNS サーバー（通常は AD DC）が設定されているか、`nslookup` でドメイン名が引けるかを確認します。
*   **Identity Source Sequence (ISS)**: 「AD で認証し、失敗したら内部 DB を見に行け」という順序構成は頻出です。
*   **Binary Comparison**: 証明書認証 (EAP-TLS) において、証明書内の ID が AD に存在するかを厳密にチェックする設定の有無が問われます。
*   **AD 属性の抽出**: デフォルトのグループ情報以外に、特定の属性（例：`department`）を認可ポリシーの条件として追加する手順を習得してください。

---

## 🛠 設定方法

### 1. Active Directory への参加 (GUI)
1.  **Administration > Identity Management > External Identity Sources > Active Directory** に移動。
2.  **Add** をクリックし、`Join Point Name` と `Active Directory Domain` を入力。
3.  **Submit** をクリック。
4.  ポップアップが表示されたら、ドメイン管理者のユーザー名とパスワードを入力して **OK**。
5.  ステータスが `Operational` になったことを確認。

### 2. グループのインポート
1.  AD 設定画面の **Groups** タブを開く。
2.  **Add > Select Groups from Directory** をクリック。
3.  フィルターを使用して必要なグループ（例: `Domain Users`）を検索し、チェックを入れて **OK**。
4.  **Save** をクリック。

---

## 🔍 検証コマンド

| 目的 | 手法 / コマンド |
| :--- | :--- |
| **AD との接続テスト** | GUI の AD 設定画面にある **Test User** ボタン。 |
| **AD Join 状態の確認 (CLI)** | <code>show application status ise</code> (Active Directory Connector が running か確認) |
| **認証詳細の確認** | **Operations > RADIUS > Live Logs** で `Identity Store` が AD になっているか確認。 |
| **DNS/ドメイン情報の確認** | <code>show tech-support</code> で `/etc/resolv.conf` や Join ログを確認。 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| AD Join 時に Clock Skew エラー | 時刻同期の失敗 | <code>show ntp</code> を実行。AD との差を 5分以内に調整する。 |
| Domain Not Found | DNS 設定の不備 | <code>nslookup [domain_name]</code> を実行し、DC の IP が返るか確認。 |
| 認証が "Internal Error" で失敗 | AD サービスアカウントのロック | AD 側で ISE のコンピュータアカウントが有効か確認。 |
| 特定ユーザーのみ認証失敗 | 権限または属性不一致 | Live Log の `Steps` を開き、どの AD 属性取得で失敗したか特定する。 |

---

## ⚠ 制限事項

*   **コンピュータアカウントの削除**: AD 側で ISE のコンピュータアカウントを削除すると、ISE からの認証は即座に失敗します。
*   **読み取り専用**: ISE は AD に対してデータの書き込み（パスワード変更やアカウント作成）は行いません。
*   **ネットワーク遅延**: ISE と DC 間の遅延が 100ms を超えると、MS-RPC の通信が不安定になり認証タイムアウトが発生しやすくなります。

---

## 🔄 他技術との関連

*   **4.1 ISE Scalability**: 分散環境では、PAN が Join を管理し、全 PSN ノードがそれぞれの AD 接続情報を保持します。
*   **4.4 802.1X / MAB**: エンドポイント（Windows クライアント）の 802.1X 認証バックエンドとして AD を使用。
*   **4.6 BYOD**: オンボーディング前の「人」の認証に AD の資格情報を使用。
*   **Cisco FMC**: pxGrid 経由で、AD ユーザー名と IP のマッピング情報を FMC に転送し、ユーザーベースの Firewall ポリシーを実現。

---

## 🧩 比較表

### Active Directory (Native Join) vs LDAP (Query to AD)

| 特徴 | AD Native Join | LDAP Query |
| :--- | :--- | :--- |
| **プロトコルサポート** | **MS-CHAPv2, Kerberos 含む全種** | PAP, EAP-GTC, EAP-TLS に限定 |
| **マシン認証** | **ネイティブサポート** | 実装が非常に困難 |
| **設定の複雑さ** | 高 (Domain Admin 権限、NTP、DNS) | 低 (単なるクエリ用アカウント) |
| **推奨用途** | **通常の Windows ドメイン環境** | 信頼関係のないレガシー/サードパーティ DB |

---

## 💡 ベストプラクティス

1.  **複数のドメインコントローラの指定**: 冗長性を確保するため、サイト内で利用可能な複数の DC を ISE が参照できるようにします。
2.  **専用 OU の作成**: ISE のコンピュータアカウントを管理しやすいよう、AD 内に `Cisco_ISE` などの専用 OU を作成して Join させます。
3.  **Global Catalog の利用**: フォレストをまたぐグループ検索を行う場合は、Global Catalog (ポート 3268) の利用設定を確認します。
4.  **ISS (Identity Source Sequence) の末尾に Internal**: 外部ソースがダウンした場合に備え、管理者がログインできるよう ISS の最後に Internal DB を含めることが推奨されます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な AD Join
*   **問題**: ISE-1 を `ccie.local` ドメインに `administrator` で参加させよ。
*   **要件**: 事前に DNS 10.1.1.10 が設定されていること。

### 2. 時刻同期トラブルの修正
*   **要件**: AD との時刻差が 10 分ある。CLI で NTP サーバー 10.1.1.10 を構成して同期せよ。
*   **設定**: `ntp server 10.1.1.10`

### 3. AD グループベースの認可ポリシー
*   **問題**: `Domain Users` グループのユーザーに SGT "Employee" を割り当てよ。
*   **設定**: Condition `ccie.local:ExternalGroups EQUALS Domain Users` -> Result `SGT_Employee`.

### 4. 複数フォレストの Join
*   **要件**: 信頼関係のない `sales.local` と `engineering.local` の両方に Join せよ。

### 5. バイナリ比較 (Binary Comparison) の有効化
*   **要件**: EAP-TLS 認証において、証明書の CN と AD の `sAMAccountName` が一致することを確認せよ。
*   **設定**: Certificate Authentication Profile で AD をソースに選び、Binary Comparison を有効化。

### 6. AD 属性の動的抽出
*   **操作**: `title` 属性を AD から取得し、認可ポリシーで使用できるように構成せよ。

### 7. Identity Source Sequence の作成
*   **要件**: まず AD を確認し、ユーザーが見つからない場合のみ内部ゲスト DB を参照する ISS を作成せよ。

### 8. マシン認証の構成
*   **問題**: AD 内に登録されている Windows PC のみの接続を許可せよ。
*   **条件**: `Network Access:WasMachineAuthenticated EQUALS True`.

### 9. 特定のドメインコントローラの除外
*   **操作**: 応答の遅い特定の DC (10.1.1.20) を、ISE が使用しないように設定でブラックリスト化せよ。

### 10. パッシブ ID (PassiveID) の統合
*   **問題**: AD のセキュリティログを ISE が WMI で購読し、ユーザー/IP 情報を取得せよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: AD Join は成功しているが、802.1X 認証 (PEAP) が失敗し、Live Log に `24408 User not found` と出る。何が原因か？
    *   **回答**: ISE に必要な **AD グループがインポートされていない**、または **Identity Source Sequence** に該当の AD Join Point が含まれていない。
2.  **Design**: ISE と AD 間で Kerberos 認証を正常に行うための最大許容時間誤差は何分か？
    *   **回答**: **5 分**。
3.  **コンフィグ読解**: 認可条件に `ccie.local:ExternalGroups EQUALS HR-Dept` とある。この `ccie.local` は何を指しているか？
    *   **回答**: ISE 上で定義された **Active Directory Join Point の名称**。
4.  **Design**: 信頼関係のない 2 つの AD フォレストを ISE で統合したい。どのように実装すべきか？
    *   **回答**: 各フォレストに対して個別の **Join Point** を作成し、それぞれを Identity Source Sequence に追加する。
5.  **実装**: 証明書認証において、AD 側でアカウントが「無効 (Disabled)」になっている場合にアクセスを拒否する機能は？
    *   **回答**: **Binary Comparison** による AD アカウントステータスチェック。

---

## 🔗 参考リソース

*   **Cisco Configuration Guide**: [Cisco ISE 3.1 管理者ガイド - Active Directory の設定](https://www.cisco.com/c/ja_jp/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31.html)
*   **Technical Note**: [Cisco ISE への Active Directory 統合に関するベストプラクティス](https://www.cisco.com/c/ja_jp/support/docs/security/identity-services-engine/118837-technote-ise-00.html)
*   **Cisco Live (BRKSEC-2041)**: [Active Directory Integration with Cisco ISE Deep Dive](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「AD は Join するもの、LDAP は検索するもの」という違いを叩き込みましょう。Join することで ISE は AD の「身内」になり、暗号化された高度な認証が可能になります。
*   **注意点**: ラボ試験では、AD 管理者のパスワードが間違っている、またはパスワードの有効期限が切れているという「初歩的なトラップ」が仕込まれることもあります。
*   **図解**: 
    - **Control-Plane**: MS-RPC (TCP 445 等)
    - **Data-Plane**: RADIUS/EAP over RADIUS.
