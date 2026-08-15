---
layout: default
title: 5.4.d-CASB
nav_order: 4
parent: 5.4-Cloud-security
grand_parent: 5.0-Advanced-Threat-Protection
---

# 5.4.d CASB policies in Cisco Umbrella

Cisco Umbrella における **CASB (Cloud Access Security Broker)** 機能は、組織内で利用されているクラウドアプリケーション（SaaS）を可視化し、それらに対する詳細な制御（許可、ブロック、特定アクションの制限）を行うためのソリューションです。従来のファイアウォールや DNS フィルタリングでは困難だった「誰が」「どのアプリで」「何をしているか（アップロード等）」を制御し、**シャドー IT (Shadow IT)** 対策とデータ漏洩防止を実現します。

---

## 📘 概要

*   **機能概要**: ネットワーク内を流れるトラフィックを分析し、数千種類のクラウドアプリの使用状況を特定。リスクスコアに基づいたアプリの評価と、Web ポリシーを用いた詳細なアクセス制御を提供します。
*   **利用目的**: 許可されていないクラウドストレージへのデータ持ち出し防止、高リスクな SaaS アプリの利用抑制、およびコンプライアンスの維持。
*   **どのような場面で利用するか**:
    *   **シャドー IT の可視化**: 従業員が勝手に使用しているクラウドアプリ（個人用 Box、未承認の SNS 等）を特定したい場合。
    *   **きめ細やかな制御**: 「Microsoft 365 は許可するが、個人用の Dropbox へのファイルアップロードは禁止する」といった高度な制御。
    *   **リスク管理**: Talos インテリジェンスが提供するリスクスコアに基づき、安全性が低いと判断されたアプリを一括ブロックする場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要コンポーネント** | App Discovery, Cloud App Control (Web Policy), DLP 連携。 |
| **可視化手法** | SWG ログ分析、DNS クエリ、またはファイアウォール/プロキシからのログアップロード。 |
| **判定基準** | アプリのカテゴリ、ビジネスリスク、セキュリティ特性（認証強度、暗号化等）。 |
| **アクション** | Allow（許可）、Block（遮断）、Warn（警告）、Isolate（隔離）。 |
| **詳細制御** | 一部のアプリにおいて「閲覧は OK、アップロードは NG」といった操作単位の制御が可能。 |
| **対応範囲** | 数千種類以上の SaaS アプリケーション。 |

---

## 🏗 動作原理

Umbrella CASB は、主に **App Discovery（発見）** と **Enforcement（強制）** の 2 つのフェーズで動作します。

```text
[ Traffic ] --- (1) SWG Proxy / DNS ---> [ Cisco Umbrella ]
                                              ↓
[ Phase 1: App Discovery ] <----------------- (2) Identify App via Signature/URL
   - Discover unsanctioned apps
   - Risk Analysis (Talos)
                                              ↓
[ Phase 2: CASB Policy Enforcement ] <------- (3) Match Cloud App Control Rules
   - User Identity check
   - App Category / Individual App check
   - Action (Block Upload / Block All)
                                              ↓
[ Result ] <--------------------------------- (4) Log event / Block Page
```

---

## ⚙ 動作シーケンス

1.  **トラフィック捕捉**: ユーザーの Web トラフィックが Umbrella SIG (SWG) を通過します。
2.  **シグネチャ照合**: Umbrella は HTTP/HTTPS ヘッダーや URL を解析し、それがどのクラウドアプリ（例：Slack, Google Drive）であるかを特定します。
3.  **App Discovery への登録**: 初めて検知されたアプリは「App Discovery」ダッシュボードにリストアップされ、管理者に通知されます。
4.  **ポリシー評価**: 管理者が作成した **Cloud App Control Policy** に基づき、該当ユーザーによるそのアプリの利用が許可されているか照合します。
5.  **詳細インスペクション**: 必要に応じて、HTTPS 通信を復号し、POST メソッド（アップロード操作）などを特定して制限をかけます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **App Discovery の活用**: 試験で「未承認のストレージアプリを特定し、リスクスコアが 5 以下のものをブロックせよ」といった要件が出る可能性があります。ダッシュボードでのフィルタリング操作を確認してください。
*   **HTTPS Decryption の重要性**: CASB による詳細な操作制御（アップロード制限など）を行うには、**SSL Decryption が有効**であることが絶対条件です。
*   **Cloud App Control ルールの作成**: Web Policy の中で、`Application Settings` ではなく、`Cloud App Control` セクションを使用してルールを構成する手順を習得してください。
*   **Tenant Restrictions**: 「会社の Google アカウントは許可し、個人の Gmail は禁止する」といった **テナント制限 (Tenant Restrictions)** 設定が CASB 関連で問われることがあります。
*   **ログの確認**: **Activity Search** で `Content Type: Cloud App Control` を指定して、意図したアプリがブロックされているかを確認する能力が求められます。

---

## 🛠 設定方法

### 1. App Discovery による現状把握 (GUI)
1.  Umbrella Dashboard の **Reporting > App Discovery** に移動。
2.  `Unsanctioned`（未承認）アプリのリストを確認し、リスクが高いものを特定。

### 2. CASB ポリシー (Cloud App Control) の作成
1.  **Policies > Web Policies** に移動。
2.  既存のポリシーを編集するか、新規作成。
3.  **Rules** タブで **Add Rule**。
4.  **Identity**: 対象のユーザー/グループを選択。
5.  **Destination**: `Cloud App Categories`（カテゴリ単位）または `Cloud Apps`（個別アプリ）を選択。
    *   例：`Cloud Storage` カテゴリを選択。
6.  **Action**: `Block` を選択。
    *   特定のアプリ（例：Dropbox）で **Edit** をクリックし、`Upload` のみにチェックを入れて保存することで、アップロード制限が可能。

---

## 🔍 検証コマンド

CASB はクラウド管理のため、ブラウザおよびダッシュボードでの確認が中心です。

| 目的 | 確認方法 |
| :--- | :--- |
| **ブロックの挙動確認** | ブラウザで該当アプリ（例：dropbox.com）にアクセスし、Umbrella ブロックページを確認。 |
| **ポリシー適用ログの確認** | **Reporting > Activity Search** で `Cloud App Control` フィルタを適用。 |
| **アプリ識別状態の確認** | **Reporting > App Discovery** で対象アプリの `Status`（Under Review/Approved/Unsanctioned）を確認。 |
| **HTTPS 復号の確認** | ブラウザの証明書情報が `Cisco Umbrella Root CA` になっていることを確認。 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| アプリをブロックできない | DNS ポリシーのみ使用している | CASB (Cloud App Control) は **Web ポリシー (SIG/SWG)** でのみ動作します。 |
| アップロード制限が効かない | SSL 復号がオフ | Web ポリシーの `HTTPS Inspection` 設定をオンにする。 |
| 全く新しいアプリが検知されない | トラフィックが SWG を通っていない | PAC ファイルや AnyConnect SWG トンネルの設定を再確認。 |
| 偽陽性（正常なアプリが止まる） | カテゴリ設定が広すぎる | 個別アプリを `Allow` リスト（例外）に追加する。 |

---

## ⚠ 制限事項

*   **ライセンス**: CASB 機能を利用するには、**Umbrella SIG Essentials/Advantage** 以上のライセンスが必要です。
*   **アプリの網羅性**: Umbrella のシグネチャに登録されていないマイナーなアプリは識別できない場合があります。
*   **モバイルアプリ**: PC ブラウザ版は詳細制御しやすい一方、モバイルアプリ版は証明書ピンニング（Certificate Pinning）の影響で SSL 復号ができず、詳細制御が制限されることがあります。

---

## 🔄 他技術との関連

*   **5.4.b DNS security policies**: DNS レイヤでは「ドメイン全体」のブロックしかできませんが、CASB は Web レイヤで「ドメイン内の一部のアクション」を制御します。
*   **3.7.c PCI-DSS / DLP**: CASB と **DLP (Data Loss Prevention)** を組み合わせることで、クラウドへの機密情報（カード番号等）の送信を自動検知・遮断します。
*   **5.4.c RBI (Remote Browser Isolation)**: 不審なアプリを直接ブロックせず、隔離されたブラウザ環境で実行させることで安全性を確保します。

---

## 🧩 比較表

### DNS Policy vs Web CASB Policy

| 特徴 | DNS Policy | Web CASB Policy (SIG) |
| :--- | :--- | :--- |
| **制御の粒度** | 低（ドメイン単位） | **高（アプリ操作・テナント単位）** |
| **SSL 復号** | 不要 | **必須（詳細制御の場合）** |
| **シャドー IT 対策** | 簡易的（既知の悪意あるドメイン） | **専門的（数千のアプリシグネチャ）** |
| **用途** | 最初の防御ライン | 詳細なガバナンスとコンプライアンス |

---

## 💡 ベストプラクティス

1.  **Sanctioned アプリの定義**: 自社で公式に許可しているアプリ（例：Google Workspace）を `Approved` としてマークし、それ以外を `Under Review` または `Unsanctioned` として管理します。
2.  **カテゴリベースの初期制限**: まずは `P2P` や `Anonymizers` カテゴリを一括ブロックし、攻撃面を最小化します。
3.  **段階的な操作制限**: いきなりアプリ全体をブロックするのではなく、まずは「ファイルのアップロード」や「投稿」のみを制限し、業務への影響を確認します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 高リスクストレージアプリの特定
*   **要件**: リスクスコアが 1〜3 の Cloud Storage アプリを App Discovery で抽出せよ。

### 2. カテゴリ「Social Networking」のブロック
*   **要件**: 全社員に対し、SNS カテゴリへのアクセスを Web ポリシーで遮断せよ。

### 3. Dropbox アップロード制限の実装
*   **要件**: Dropbox の閲覧は許可するが、ファイルのアップロードのみをブロックせよ。
*   **前提**: SSL Decryption 有効化。

### 4. 未承認アプリのステータス変更
*   **操作**: 検知された「WeTransfer」を `Unsanctioned` に設定し、自動的にポリシーでブロックされるように構成せよ。

### 5. テナント制限 (Google)
*   **要件**: `example.com` ドメインの Google アカウントのみログインを許可し、個人アカウントを制限せよ。

### 6. App Discovery ログの外部転送
*   **要件**: S3 バケットを設定し、Umbrella の CASB ログを定期的にエクスポートせよ。

### 7. リスクスコアに基づく自動ブロック
*   **要件**: リスクスコア 4 以下のアプリが新しく検知された際、自動的にブロックするルールを作成せよ（※現在 Umbrella では動的ルール生成を組み合わせる設定）。

### 8. Web メールへの添付ファイル禁止
*   **要件**: Gmail でのメール送受信は許可するが、ファイルの添付（Attach）を制限せよ。

### 9. 特定 Identity への例外適用
*   **要件**: IT 管理者グループのみ、すべての Unsanctioned アプリへのアクセスを許可せよ。

### 10. Activity Search によるフォレンジック
*   **操作**: 過去 24 時間に Cloud App Control ルールによってブロックされた通信の送信元ホスト名を特定せよ。

---

## ❓ 想定試験問題

1.  **Design**: 従業員が個人用の Box を使ってファイルを外部に持ち出すのを防ぎたいが、会社用の Box は利用させたい。最適な設定は？
    *   **回答**: Web Policy の **Cloud App Control** において、Box アプリに対し **Tenant Restrictions** を構成する。
2.  **トラブルシュート**: 「Cloud App Control」で特定の SNS アプリをブロックしたが、ユーザーがアクセスできてしまう。真っ先に確認すべき Web ポリシーの設定は？
    *   **回答**: **HTTPS Inspection (SSL復号)** が有効になっているか、およびポリシーの **Identity** が正しくマッチしているか。
3.  **コンフィグ読解**: App Discovery 画面で、あるアプリのセキュリティスコアが低い理由として「Financial Viability」が挙げられている。これは何を意味するか？
    *   **回答**: そのアプリを提供している企業の **財務的な健全性** やビジネスの持続性に懸念があり、将来的なサービス停止やデータ紛失のリスクがあることを示している。
4.  **実装**: CASB ポリシーで「Warning」アクションを選択した場合、ユーザーの体験はどうなるか？
    *   **回答**: ユーザーには警告ページが表示されるが、ユーザー自身が「Continue」をクリックすればサイトへのアクセスを継続できる。
5.  **Design**: 組織全体のクラウドアプリ利用傾向を月次レポートで報告したい。どのダッシュボードを使用すべきか？
    *   **回答**: **Reporting > App Discovery** のサマリー画面、または **SaaS User Reports**。

---

## 🔗 参考リソース

*   **Cisco Umbrella Documentation**: [App Discovery and Control Guide](https://docs.umbrella.com/deployment-umbrella/docs/app-discovery)
*   **Cisco Umbrella SIG Guide**: [Manage Cloud App Control Policies](https://docs.umbrella.com/deployment-umbrella/docs/cloud-app-control-rules)
*   **Cisco Live (BRKSEC-2041)**: [Securing SaaS with Cisco Umbrella CASB](https://www.ciscolive.com/)
*   **Official Cert Guide (SCOR 350-701)**: [Chapter 12: Cloud Security and CASB](https://www.ciscopress.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「CASB はクラウド時代の検問所」です。誰が何を持ち出そうとしているかを細かくチェックします。
*   **図解**: 
    - **Visible**: App Discovery (光を当てる)
    - **Controllable**: Cloud App Control (門を閉める)
*   **注意点**: ラボ試験では、**Web Policy の順序（上位ルールが優先）**に注意してください。グローバルなブロックルールの下に個別の許可ルールを置いてしまうと、許可ルールが無視されます。
