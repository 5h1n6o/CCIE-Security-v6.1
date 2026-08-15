---
layout: default
title: 5.4.e-DLP
nav_order: 5
parent: 5.4-Cloud-security
grand_parent: 5.0-Advanced-Threat-Protection
---

# 5.4.e DLP policies in Cisco Umbrella

**DLP (Data Loss Prevention: データ損失防止)** ポリシーは、Cisco Umbrella の SIG (Secure Internet Gateway) 機能の中核であり、組織外へ転送される機密情報（クレジットカード番号、個人情報、機密ドキュメント等）を検知・遮断するために使用されます。クラウド経由の Web トラフィック（HTTP/HTTPS）をリアルタイムで検査し、コンプライアンス要件（PCI-DSS, GDPR 等）の遵守を支援します。

---

## 📘 概要

*   **機能概要**: ユーザーが Web サイトやクラウドストレージへファイルをアップロードしたり、テキストを入力したりする際に、その内容をスキャンして機密データの流出を阻止する機能です。
*   **利用目的**: 意図的または不注意による機密情報の漏洩防止、および PCI-DSS などの業界標準への適合。
*   **どのような場面で利用するか**:
    *   **機密ファイルのアップロード制限**: 個人用 Google Drive や Dropbox へ社内資料をアップロードしようとした際に遮断。
    *   **Web フォームへの入力監視**: SNS や掲示板にクレジットカード番号やマイナンバーを入力して送信する行為を検知。
    *   **シャドー IT 制御**: 許可されていない（Unsanctioned）SaaS アプリケーションへのデータ転送をきめ細かく制御。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **検査対象** | HTTP/HTTPS トラフィック (Web ポスト、ファイルアップロード)。 |
| **識別子 (Classifiers)** | **Built-in** (クレジットカード、住所、マイナンバー等) および **Custom** (正規表現、キーワード)。 |
| **アクション** | **Monitor Only** (ログ記録のみ) または **Block** (遮断)。 |
| **要件** | **SIG Advantage** ライセンス、**SSL Decryption** の有効化。 |
| **データ状態** | Data-in-Motion (移動中のデータ) に特化。 |
| **除外設定** | 信頼されたドメインや特定のユーザーを検査対象から除外可能。 |

---

## 🏗 動作原理

Umbrella DLP は、プロキシベースのインスペクションエンジンとして動作します。

```text
[ Client ] --(1) Upload Data / File --> [ Umbrella SWG (Proxy) ]
                                              ↓
                                      (2) SSL Decryption
                                              ↓
                                      (3) DLP Inspection Engine
                                              ↓
                                      (4) Pattern Matching
                                      (Regex / Dictionaries / File Types)
                                              ↓
[ Enforcement ] <--------------------- (5) Match Result
      ├── [Block] --> Show Block Page & Log Event
      └── [Allow] --> Data forwarded to Internet Destination
```

---

## ⚙ 動作シーケンス

1.  **トラフィック捕捉**: トラフィックが Umbrella SIG (SWG) に転送されます（AnyConnect, IPsec トンネル, または PAC ファイル経由）。
2.  **SSL 復号**: HTTPS 通信の場合、DLP エンジンが中身をスキャンするために Umbrella がトラフィックを復号します。
3.  **コンテンツ抽出**: 送信されたファイルや Web ポストデータからテキスト情報を抽出します。
4.  **ポリシー照合**: 定義された「データ分類（Data Classifiers）」と照合します。
    *   例：16 桁の数字が Luhn アルゴリズムでクレジットカード番号と判定されるか。
5.  **判定と実行**: ポリシーに一致した場合、通信を即座にドロップし、ユーザーに遮断画面を表示します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **SSL Decryption の前提**: ラボ試験で DLP が動作しない最大の原因は、**SSL 復号が有効になっていない**ことです。DLP 設定の前に Web ポリシーで HTTPS Inspection を有効化する手順を確認してください。
*   **Data Classifiers の構成**: 「クレジットカード番号が含まれる場合に遮断せよ」という要件に対し、組み込みの分類子を正しく選択できる必要があります。
*   **DLP ポリシーの優先順位**: Umbrella ダッシュボードでは、DLP ポリシーは専用のセクションで管理されます。他の Web ポリシー（アクセス制御）との評価順序を理解してください。
*   **コンプライアンス要件 (PCI-DSS)**: 試験問題で PCI-DSS 準拠が求められた場合、DLP ポリシーで「Credit Card Number」の検知を有効にすることが正解となります。
*   **トラブルシュート（ログ読解）**: **Activity Search** で `Response Code: Blocked (DLP)` をフィルタリングし、どのルールがトリガーされたか特定するスキルが求められます。

---

## 🛠 設定方法

### 1. SSL 復号の有効化 (前提)
1.  **Policies > Web Policies** に移動。
2.  該当するポリシーの **HTTPS Inspection** を `On` に設定。
3.  Umbrella Root CA 証明書をクライアント端末にインストール。

### 2. DLP ポリシーの作成 (Umbrella Dashboard)
1.  **Policies > DLP Policies** に移動。
2.  **Add Policy** をクリック。
3.  **Identity**: 対象の Network, User, Roaming Computer を選択。
4.  **Content**:
    *   **Built-in Classifiers**: `Financial > Credit Card` などを選択。
    *   **Severity**: 低・中・高の閾値を設定。
5.  **Destination**: すべてのサイト、または特定のアプリカテゴリを選択。
6.  **Action**: `Block` または `Monitor` を選択。

---

## 🔍 検証コマンド

DLP はクラウド上で動作するため、端末やルータの CLI よりもダッシュボード上での確認が不可欠です。

| 目的 | 確認方法 |
| :--- | :--- |
| **ポリシー適用ログの確認** | Umbrella Dashboard > **Reporting > Activity Search** で `Content Type: DLP` を確認。 |
| **テストデータの送信** | `https://dlptest.com` などのテストサイトを使用し、ダミーのカード番号をアップロード。 |
| **SWG 経由の確認** | クライアント側で `http://proxy.umbrella.com/debug/` にアクセスし SWG が Enabled か確認。 |
| **詳細レポートの表示** | **Reporting > DLP Report** で、どの個人情報が最も多く検知されているかを確認。 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| **機密データが遮断されない** | SSL 復号が未設定 | Web ポリシーで HTTPS Inspection が有効か再確認。 |
| **遮断ログが残らない** | アクションが `Allow` | アクションが `Monitor Only` になっていないか確認。 |
| **誤検知 (False Positive)** | 閾値が低すぎる | `Occurrences` (発生回数) の閾値を上げて調整する。 |
| **特定のサイトで DLP が効かない** | SSL Decryption のバイパス | **Domain Bypass** リストにそのサイトが含まれていないか確認。 |

---

## ⚠ 制限事項

*   **パスワード保護ファイル**: 暗号化された ZIP や PDF の中身は、Umbrella DLP ではスキャンできません。
*   **ファイルサイズの制限**: 極端に大きなファイル（数GB以上）は、リアルタイムスキャンの対象外となる場合があります。
*   **非 Web プロトコル**: DLP ポリシーは Web (HTTP/HTTPS) プロキシを介した通信にのみ適用されます。SSH や FTP（非 Web）での流出防止には対応しません。

---

## 🔄 他技術との関連

*   **3.7.c PCI-DSS**: DLP はクレジットカード情報の流出を防ぐことで、このコンプライアンス要件を満たします。
*   **5.4.d CASB**: CASB は「どのアプリ」を使わせるかを制御し、DLP はそのアプリの中で「何を」送るかを制御します。
*   **1.0 VPN (AnyConnect)**: ユーザーが外出先でも SIG (SWG) を強制経由するように AnyConnect Umbrella モジュールを構成することで、DLP 保護を維持します。

---

## 🧩 比較表

### Umbrella DLP vs オンプレミス DLP (ESA 等)

| 特徴 | Umbrella DLP (Cloud) | On-Premise DLP (ESA/WSA) |
| :--- | :--- | :--- |
| **主な経路** | **Web (HTTPS), クラウドアプリ** | **Eメール (SMTP)**, ローカルWeb |
| **展開の速さ** | 即時（クラウド設定のみ） | アプライアンス導入が必要 |
| **オフネット保護** | どこにいても適用可能 | 社内ネットワーク内のみ |
| **管理** | Umbrella Dashboard 一括管理 | 個別デバイス管理 |

---

## 💡 ベストプラクティス

1.  **Monitor Mode から開始**: 導入初期は `Action: Monitor` に設定し、正常な業務通信を誤って遮断していないか 1〜2 週間ログを分析します。
2.  **Occurrences (発生回数) の活用**: 1 回の検知で遮断するのではなく、「1 ファイル内に 5 個以上のカード番号がある場合のみ遮断」とすることで精度を高めます。
3.  **機密情報キーワードの定義**: 自社固有のプロジェクトコード名などを `Custom Identifier` として登録し、意図的な持ち出しを防止します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的なクレジットカード遮断
*   **要件**: 全社員に対し、外部 Web サイトへのクレジットカード番号の送信をブロックせよ。
*   **設定**: DLP Policy > Built-in Classifiers > `Credit Card`.

### 2. PCI-DSS 準拠ポリシー
*   **要件**: 金融部門のユーザーがクラウドストレージへ財務データをアップロードするのを監視せよ。
*   **設定**: Identity: `Finance-Group`, Action: `Monitor Only`.

### 3. 日本固有の個人情報保護
*   **要件**: マイナンバー（My Number）が含まれるデータの流出をブロックせよ。
*   **設定**: Built-in Classifiers > `Japan > My Number`.

### 4. 特定ドメインの DLP 除外
*   **要件**: 自社のパートナーポータル (`partner.example.com`) へのデータ送信は DLP 検査から除外せよ。

### 5. ソーシャルメディアへの投稿制限
*   **要件**: Facebook や Twitter への投稿内容に機密キーワードが含まれる場合、遮断せよ。

### 6. カスタム正規表現による機密コード検知
*   **要件**: 社内独自の ID 形式 `PROJ-{4}` を検知するルールを作成せよ。

### 7. ファイルタイプベースの制御
*   **要件**: ソースコードファイル（.py, .c, .java）のアップロードを全般的にブロックせよ。

### 8. DLP Block Page のカスタマイズ
*   **要件**: 遮断時に「セキュリティポリシー違反です。IT 部門（内線 1234）へ連絡してください」と表示せよ。

### 9. 大規模アップロードの検知
*   **要件**: 100 件以上の個人情報が含まれるバルクデータの送信を「高重要度」イベントとして記録せよ。

### 10. Activity Search によるインシデント調査
*   **操作**: 過去 24 時間にブロックされた DLP イベントから、ファイルを送信しようとしたユーザー名と宛先 URL を特定せよ。

---

## ❓ 想定試験問題

1.  **Design**: リモートユーザーがスターバックスの Wi-Fi を使用している際にも DLP ポリシーを適用したい。最低限必要なコンポーネントは？
    *   **回答**: **AnyConnect Umbrella Roaming Module** (SWG 有効化) および **Umbrella SIG Advantage** ライセンス。
2.  **トラブルシュート**: DLP ポリシーで「Credit Card」をブロック設定したが、HTTPS サイトにカード番号を書き込めてしまう。原因は？
    *   **回答**: **HTTPS Inspection** が無効である、または当該サイトが **SSL Decryption の除外リスト**に含まれている。
3.  **コンフィグ読解**: DLP ポリシーの `Severity`（重要度）レベルを「High」に設定した場合、検知の挙動はどう変わるか？
    *   **回答**: より多くの機密データが検出された場合（例：1 ファイル内のカード番号数が多い等）にのみ、アラートや遮断が実行されるようになる（偽陽性を減らす設定）。
4.  **実装**: 会社の Google アカウントでの Drive 利用は許可しつつ、個人アカウントへのデータ転送のみ DLP をかけたい。併用すべき機能は？
    *   **回答**: **CASB (Tenant Restrictions)** と **DLP ポリシー**。
5.  **Design**: 特定の法規制（GDPR 等）に対応するための最も効率的な DLP 設定方法は？
    *   **回答**: Umbrella にあらかじめ用意されている **GDPR テンプレート（Compliance Rules）** を使用する。

---

## 🔗 参考リソース

*   **Cisco Umbrella Documentation**: [Data Loss Prevention (DLP)](https://docs.umbrella.com/deployment-umbrella/docs/dlp-policy)
*   **Cisco Live (BRKSEC-2041)**: [Cloud Security Architecture and SIG](https://www.ciscolive.com/)
*   **Design Guide**: [Implementing Cisco Umbrella SIG with DLP](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/umbrella-deployment-guide.html)
*   **Technical Note**: [Troubleshooting SSL Decryption for DLP](https://support.umbrella.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「DLP はコンテンツの警察」です。パケットの宛先（IP/URL）だけでなく、その中身（ペイロード）を読み取って判断します。
*   **図解**: `User --(Encrypted)--> SWG --(Decrypt/Scan)--> DLP Engine --(Decision)--> Internet`.
*   **注意点**: ラボ試験では、**証明書の信頼関係**が壊れていると Web ブラウジング自体ができなくなるため、DLP の検証前に必ず正常なプロキシ経由の通信を確認してください。
