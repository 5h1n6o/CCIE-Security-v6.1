---
layout: default
title: 5.7.b-DLP
nav_order: 2
parent: 5.7-Email-security
grand_parent: 5.0-Advanced-Threat-Protection
---

# 5.7.b DLP (Data Loss Prevention)

Cisco Secure Email (旧 Email Security Appliance: ESA) における **DLP (データ損失防止)** 機能は、組織外へ送信される電子メールをスキャンし、機密情報（クレジットカード番号、個人情報、知的財産など）の意図しない流出を特定・阻止するための高度なセキュリティコンポーネントです。RSA Enterprise DLP エンジンを統合しており、業界標準（PCI-DSS, HIPAA 等）に基づいた包括的な保護を提供します。

---

## 📘 概要

*   **機能概要**: 送信メールの本文および添付ファイルをリアルタイムで解析し、定義された機密情報のパターン（Classifiers）に一致するかを判定して、適切なアクション（遮断、暗号化、隔離等）を実行する機能です。
*   **利用目的**: 法規制（GDPR、個人情報保護法等）の遵守、企業秘密の保護、および不注意による情報漏洩リスクの低減。
*   **どのような場面で利用するか**:
    *   **コンプライアンス保護**: 財務部門が外部へ送るメールにクレジットカード番号が含まれている場合、自動的に暗号化して送信する。
    *   **機密ドキュメント管理**: 特定のキーワード（「社外秘」「極秘」）や文書フィンガープリントに一致するファイルを検知し、管理者へ通知する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **エンジン** | RSA Enterprise DLP エンジンを搭載。 |
| **適用場所** | **Outgoing Mail Pipeline**（外向きのメール処理プロセス）に適用。 |
| **判定基準** | **Classifiers**（クレジットカード、住所、特定コード等）の組み合わせ。 |
| **感度設定** | 判定の「しきい値」を調整可能。 |
| **アクション** | Deliver, Drop, Quarantine, Encrypt, Notify, Tag。 |
| **暗号化連携** | **Cisco Secure Email Encryption (旧 CRES)** と連携し、自動暗号化が可能。 |

---

## 🏗 動作原理

DLP は電子メールパイプラインの後半、コンテンツフィルタリングの前後で動作します。

```text
Internal Mail Server
   ↓
ESA Private Listener (Incoming Connection)
   ↓
Outgoing Mail Policies
   ↓
[ DLP Scanning Engine ]
   ├── Content Analysis (Body + Attachments)
   ├── Policy Matching (RSA Classifiers)
   └── Severity Evaluation
   ↓
[ Action Execution ]
   ├── Secure (Encrypt)
   ├── Block (Drop)
   └── Deliver (Allow)
```

---

## ⚙ 動作シーケンス

1.  **メッセージ受信**: ESA が内部メールサーバからメッセージを受信し、送信者が Outgoing Mail Policy の対象であることを確認します。
2.  **コンテンツ抽出**: DLP エンジンがメール本文および添付ファイル（PDF, Word, Excel, ZIP等）からテキスト情報を抽出します。
3.  **シグネチャ照合**: 抽出されたテキストを「データ分類子（Classifiers）」と照合します。
4.  **ポリシー判定**: 複数のルールを評価し、最も高い重要度（Severity）に合致するポリシーを特定します。
5.  **アクション適用**: 合致したポリシーに基づき、メッセージをそのまま送るか、隔離するか、あるいは暗号化して保護するかを決定します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **DLP 有効化の手順**: グローバル設定（Security Services > DLP）でエンジンを有効化し、その後 **Outgoing Mail Policy** で DLP を `Enabled` に設定する流れが重要です。
*   **既定ポリシーの活用**: `PCI-DSS` や `HIPAA` などの組み込みポリシーテンプレートを正確に選択できること。
*   **暗号化 (Encryption) との組み合わせ**: DLP で検知した際に「暗号化アクション（Encrypt）」を選択し、外部にセキュアな形式で配送する設定は頻出です。
*   **しきい値（Severity）の調整**: 「3つ以上のカード番号が含まれる場合のみブロック」といった細かな要件に対応するため、ルールの `Occurrences` 設定を調整するスキルが求められます。
*   **トラブルシュート（mail_logs）**: ログ内の `DLP match` エントリを確認し、どのポリシー（Policy Name）によって処理されたかを特定する能力が必要です。

---

## 🛠 設定方法

### 1. DLP エンジンの有効化 (GUI)
1.  **Security Services > DLP** に移動。
2.  **Enable DLP** をクリックし、RSA エンジンをアクティブにします。

### 2. DLP ポリシーの作成
1.  **Mail Policies > DLP Policy Manager** を選択。
2.  **Add Policy** > `Predefined Policy` から `PCI-DSS` などを選択。
3.  ルールを設定（例: クレジットカード番号の検知）。
4.  アクションを設定（例: `Quarantine` または `Deliver with Encryption`）。

### 3. 送信ポリシーへの適用
1.  **Mail Policies > Outgoing Mail Policies** に移動。
2.  該当するポリシーの `DLP` 列をクリックし、作成した DLP ポリシーを選択して `Enabled` にします。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **リアルタイムログ監視** | <code>tail mail_logs</code> |
| **DLP エンジン状態確認** | <code>dlpstatus</code> |
| **DLP ポリシー設定の表示** | <code>dlpconfig</code> |
| **特定メッセージの検索** | <code>grep "DLP match" mail_logs</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 検知されない | DLP が無効 | `dlpstatus` | Security Services で DLP を Enable にする。 |
| 検知されない | 適用ポリシーの誤り | `mail_logs` | Outgoing Mail Policy で DLP が Enable か確認。 |
| 添付ファイルが漏れる | パスワード保護 | `mail_logs` | 暗号化された添付ファイルはスキャン不可。 |
| 誤検知が多い | しきい値が低すぎる | GUI ルール設定 | `Occurrence` 数を増やして精度を調整する。 |

---

## ⚠ 制限事項

*   **暗号化ファイル**: パスワード付き ZIP や暗号化された PDF の中身は、ESA で復号できない限りスキャンできません。
*   **リソース負荷**: 非常に複雑な正規表現や大規模な添付ファイルのスキャンは、メールの処理遅延（Latency）を招く可能性があります。
*   **ライセンス**: DLP 機能の利用には、有効な **Content Security** ライセンスが必要です。

---

## 🔄 他技術との関連

*   **5.7.a Mail Policies**: DLP は Outgoing Mail Policy を通じて個別の送信者グループに適用されます。
*   **1.0 VPN (Secure Email Encryption)**: 検知した機密情報を保護して配送するために、暗号化サービス (PXE) が不可欠です。
*   **4.7 AD/LDAP**: 送信者の所属部署に基づいて、異なる DLP 強度を適用するために連携します。

---

## 🧩 比較表

### DLP vs コンテンツフィルタ (Content Filters)

| 特徴 | DLP (Data Loss Prevention) | コンテンツフィルタ |
| :--- | :--- | :--- |
| **エンジンの専門性** | 高 (RSA エンジンによるパターン解析) | 中 (正規表現や辞書ベース) |
| **コンプライアンス** | 強 (PCI, HIPAA テンプレート完備) | 弱 (手動構築が必要) |
| **スキャン対象** | **機密情報のパターン** に特化 | ヘッダー、添付ファイル名、容量等 |
| **主な用途** | データの流出防止 | 運用の自動化、脅威の排除 |

---

## 💡 ベストプラクティス

1.  **段階的な導入**: 最初はアクションを `Deliver` にし、ログを収集して正常な通信を誤検知していないか確認する「フェーズイン」アプローチを推奨します。
2.  **フィンガープリントの使用**: 重要な設計図や文書そのものを登録する「ドキュメント・フィンガープリント」により、キーワード回避を目的とした改変も検知可能にします。
3.  **ユーザ通知**: 遮断時に送信者へ通知メールを送ることで、セキュリティポリシーの周知と教育に繋げます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的なクレジットカード検知
*   **要件**: 送信メールにクレジットカード番号が含まれる場合、隔離（Quarantine）せよ。
*   **設定**: DLP Policy Manager で `Credit Card` Classifier を使用し、Action を `Quarantine` に設定。

### 2. コンプライアンス暗号化 (PCI-DSS)
*   **要件**: PCI-DSS 違反が検知された場合、自動的に暗号化して送信せよ。
*   **設定**: Action に `Encrypt` プロファイルを指定。

### 3. しきい値ベースの制限
*   **要件**: 個人情報（住所等）が 5 つ以上含まれる場合のみ、管理者へ通知せよ。
*   **設定**: ルール設定で `Occurrences > 4` に設定。

### 4. 特定部署（人事部）専用 DLP
*   **要件**: 人事部からのメールのみ、マイナンバー検知を有効にせよ。
*   **設定**: `Outgoing Mail Policy` で人事部用ポリシーを作成し、専用の DLP ポリシーを紐付け。

### 5. 文書フィンガープリントの構成
*   **要件**: 特定の機密 PDF ファイルが送出されるのを阻止せよ。

### 6. カスタムキーワード辞書の作成
*   **要件**: 独自のプロジェクトコード名「Project-CCIE」が含まれるメールをドロップせよ。

### 7. 添付ファイル内のパターン検知
*   **要件**: Excel ファイル内の 10 桁の数字パターンを検知せよ。

### 8. スキャン例外設定
*   **要件**: 役員からのメールは DLP スキャンをバイパスせよ。

### 9. 暗号化メールのブロック
*   **要件**: DLP スキャンが不可能な（暗号化された）添付ファイルを持つメールを拒否せよ。

### 10. DLP ログの外部転送
*   **要件**: DLP 検知イベントを Syslog で監視サーバへ送信せよ。

---

## ❓ 想定試験問題

1.  **実装**: 内部ユーザが外部へ機密情報を送る際、ESA を通じて自動的に暗号化を適用したい。最低限必要な設定要素は？
    *   **回答**: **DLP ポリシーの作成**、**Encryption プロファイルの作成**、および **Outgoing Mail Policy への適用**。
2.  **トラブルシュート**: DLP ポリシーを作成したが、テストメールが検知されずに送信された。`mail_logs` で何を確認すべきか？
    *   **回答**: メッセージが **どの Outgoing Mail Policy にマッチしたか**、およびそのポリシーで DLP が `Enabled` になっていたか。
3.  **コンフィグ読解**: DLP ルールで `Severity: High` が設定されている意味は？
    *   **回答**: 一致の確信度や情報の重要度が高いことを示し、より厳しいアクション（Drop 等）をトリガーするために使用される。
4.  **Design**: 拠点間で機密情報をやり取りするが、VPN がない。ESA で対応する方法は？
    *   **回答**: **DLP と Secure Email Encryption (CRES)** を組み合わせ、インターネット経由でも暗号化された状態で配送する。
5.  **制限**: ESA DLP で検知できないファイル形式は？
    *   **回答**: **パスワードで保護された暗号化ファイル**。

---

## 🔗 参考リソース

*   **Cisco Secure Email Configuration Guide**: [Data Loss Prevention](https://www.cisco.com/c/en/us/td/docs/security/esa/esa11-1/user_guide/b_ESA_Admin_Guide_11_1/b_ESA_Admin_Guide_11_1_chapter_01001.html)
*   **Cisco Live (BRKSEC-2041)**: [Email Security Best Practices and DLP](https://www.ciscolive.com/)
*   **Technical Note**: [Troubleshooting DLP on ESA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117970-technote-esa-00.html)

---

## 📝 **補足（Notes）**  

- **学習メモ**: DLP は「内容の警察」です。パケットのヘッダーではなく、中身（ペイロード）を深く掘り下げてスキャンすることを意識してください。
- **図解**: パイプライン内で DLP は「改札口」のような役割を果たし、許可証（ポリシー）がないデータを通しません。
- **注意点**: ラボ試験では、**Outgoing** ポリシーに設定することを忘れないでください（Incoming に DLP は通常適用されません）。
