---
layout: default
title: 5.7.a-Mail-policies
nav_order: 1
parent: 5.7-Email-security
grand_parent: 5.0-Advanced-Threat-Protection
---

# 5.7.a Mail policies

Cisco Secure Email (旧 Email Security Appliance: ESA) における **Mail Policy** は、電子メールの「パイプライン」内で、特定の送信者または受信者のグループに対してどのようなセキュリティ処理（アンチスパム、アンチウイルス、コンテンツフィルタ等）を適用するかを決定する中核的なルールセットです。

---

## 📘 概要

*   **機能概要**: 送信元または送信先のメールアドレス、ドメイン、あるいは所属するグループ（LDAP）に基づいてメッセージを分類し、それぞれに異なるスキャンエンジンやフィルタを適用する機能です。
*   **利用目的**: 全社一律のポリシーではなく、特定の部署（例：人事部には機密情報漏洩防止、技術部には実行ファイル許可など）に応じた柔軟なセキュリティ管理を実現します。
*   **どのような場面で利用するか**:
    *   **Incoming (内向き)**: 外部からのメールに対し、VIP ユーザにはより厳しいスパム判定を適用し、一般ユーザには標準的な設定を適用する場合。
    *   **Outgoing (外向き)**: 社内からのメールに対し、特定のドメイン（パートナー企業）宛には暗号化を強制し、それ以外は通常通り送信する場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **分類の基準** | Incoming は受信者 (Recipient)、Outgoing は送信者 (Sender) のアドレスに基づきます。 |
| **評価順序** | ポリシーテーブルの **上から下へ** 順に評価され、最初に一致したものが適用されます (First Match)。 |
| **適用サービス** | Anti-Spam, Anti-Virus, Graymail, AMP, Content Filters, DLP。 |
| **デフォルトポリシー** | どの個別ポリシーにも一致しない場合、最下部の `Default Policy` が適用されます。 |
| **グループ連携** | LDAP 問い合わせを使用して、AD 上のセキュリティグループ単位でポリシーを適用可能です。 |

---

## 🏗 動作原理

Mail Policy は、SMTP 接続が受け入れられた後の「メッセージ処理フェーズ」で動作します。

```text
[ Incoming Connection ]
   ↓
[ HAT (IP Reputation Check) ] --> 接続自体の拒否/許可を決定
   ↓
[ RAT (Recipient Verification) ] --> 受信ドメインの妥当性確認
   ↓
[ Mail Policy Matching ]
   - 宛先/送信者アドレスをテーブルと照合
   - 一致したポリシーを選択
   ↓
[ Security Services Pipeline ]
   - Anti-Spam (判定: Spam/Suspected/Clean)
   - Anti-Virus (判定: Infected/Clean)
   - Advanced Malware Protection (ハッシュ照会)
   - Content Filters (条件ベースの操作)
   ↓
[ Delivery / Quarantine ] --> 最終的な配送または隔離
```

---

## ⚙ 動作シーケンス

1.  **アイデンティティの特定**: ESA はメッセージの `Envelope From` (Outgoing) または `Envelope To` (Incoming) を抽出します。
2.  **ポリシー照合**:
    *   Incoming の場合、宛先アドレスがポリシーの `Add Recipients` リストに含まれているか確認します。
    *   Outgoing の場合、送信者アドレスが `Add Senders` リストに含まれているか確認します。
3.  **アクションの継承**: 個別ポリシーで「Use Default」が選択されている項目は、Default Policy の設定を引き継ぎます。
4.  **例外処理**: 1 つのメッセージに複数の宛先があり、それぞれ異なるポリシーにマッチする場合、ESA はメッセージを内部的にコピー（スプリット）して個別に処理します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **一致順序の調整**: 「特定のドメイン `example.com` へのメールだけ特別扱いせよ」という問題では、Default Policy よりも上に専用ポリシーを作成し、順序を正しく配置する必要があります。
*   **LDAP Group との紐付け**: 受信者の指定にメールアドレスを直書きするのではなく、`LDAP Group` を参照させる設定が問われることがあります。
*   **スプリットメッセージの理解**: 1 通のメールが複数のポリシーによって「分割」される挙動を `mail_logs` から読み取る能力が重要です。
*   **サービスごとのアクション定義**: スパム判定された際の挙動（隔離するか、ヘッダーに `[SPAM]` と付与するか）をポリシーごとに正確に構成できる必要があります。
*   **Content Filter との連携**: Mail Policy 内で特定の Content Filter を有効化し忘れるミスが多いため、適用先のポリシー名を確認する癖をつけましょう。

---

## 🛠 設定方法

### 1. 新規 Incoming Mail Policy の作成 (GUI)
1.  **Mail Policies > Incoming Mail Policies** を選択。
2.  **Add Policy** をクリック。
3.  `Policy Name` を入力（例：`VIP_Users`）。
4.  `Add Recipients` セクションで、対象となるメールアドレスやドメイン、または LDAP グループを追加。
5.  **Submit** をクリック。

### 2. セキュリティサービスのカスタマイズ
1.  作成したポリシーの `Anti-Spam` 列をクリック。
2.  `Use Default` のチェックを外し、独自の判定閾値やアクション（例：Quarantine）を設定。
3.  **Commit Changes** を実行して反映。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **リアルタイムログの表示** | <code>tail mail_logs</code> |
| **特定メッセージのポリシー適用確認** | <code>grep "MID 123" mail_logs</code> |
| **設定済みのポリシー一覧表示** | <code>policyconfig</code> |
| **LDAP グループ解決のテスト** | <code>ldaptest</code> |
| **送信元レピュテーションの確認** | <code>reputation [IP]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 設定したポリシーにマッチしない | ポリシーの優先順位が低い | <code>policyconfig</code> で順序を確認し、上位へ移動させる。 |
| 宛先が複数あると挙動が変わる | メッセージのスプリット発生 | `mail_logs` で `splitting message` の記録を確認する。 |
| スパムが素通りする | `Anti-Spam` が `Disabled` | ポリシー内の該当サービスが `Enabled` になっているか確認。 |
| LDAP 連携したポリシーが動かない | ユーザ名の解決失敗 | `ldaptest` を実行し、ESA が正しく AD 情報を引けているか確認。 |

---

## ⚠ 制限事項

*   **アドレス形式の依存**: Envelope アドレス（SMTP 通信上のアドレス）を使用するため、メールヘッダー（本文内）の `From/To` と一致しない場合があります。
*   **リソース消費**: 個別ポリシーを増やしすぎると、メッセージのスプリットが多発し、ESA の処理パフォーマンス（スループット）が低下する可能性があります。
*   **順序の固定**: `Default Policy` は常に最下部であり、削除や順序変更はできません。

---

## 🔄 他技術との関連

*   **4.7 Active Directory 連携**: ポリシーの対象者を動的に決定するために LDAP クエリを使用します。
*   **5.7.b Content filters**: Mail Policy 内で、より詳細な「条件（Condition）」に基づいた処理を実行するために呼び出されます。
*   **5.1 Cisco AMP**: ファイルスキャンの詳細設定は Mail Policy レベルでオン/オフを制御します。

---

## 🧩 比較表

### Incoming Policy vs Outgoing Policy

| 特徴 | Incoming Mail Policy | Outgoing Mail Policy |
| :--- | :--- | :--- |
| **主要な判定基準** | **Recipients** (受信者) | **Senders** (送信者) |
| **一般的な目的** | スパム・ウイルスからの保護 | DLP、暗号化、免責事項の付与 |
| **適用インターフェイス** | Public Listener | Private Listener |
| **DLP 機能** | 通常は使用しない | **必須コンポーネント** |

---

## 💡 ベストプラクティス

1.  **Default は厳しめに**: 特殊な要件がない限り、Default Policy は安全側に倒した（スパムは隔離、ウイルスは削除）設定にします。
2.  **ドメインベースの優先**: 特定パートナー企業との通信用ポリシーは、汎用的なポリシーよりも上位に配置します。
3.  **コメントの活用**: ポリシー作成時は `Description` 欄に作成目的と日付を記載し、後日のトラブルシュートを容易にします。

---

## 📝 ラボ学習・設定サンプル例

### 1. 特定ドメインのスパムバイパス
*   **要件**: `partner.com` からのメールはスパム検査をスキップせよ。
*   **設定**: `Incoming Mail Policy` を作成し、宛先を `All` に、送信元（HAT 連携）または Content Filter で `partner.com` を除外し、Anti-Spam を `Disabled` にする。

### 2. 人事部向けの DLP 強制
*   **要件**: 送信者が `hr@example.com` の場合、DLP スキャンを強制せよ。
*   **設定**: `Outgoing Mail Policy` で人事部用ポリシーを作成し、DLP 列を `Enabled` にする。

### 3. スパム判定時の件名書き換え
*   **要件**: スパムの疑いがあるメールの件名に `[SUSPECT]` を付与せよ。
*   **設定**: Policy の `Anti-Spam` 設定で `Suspected Spam` のアクションを `Deliver` にし、`Subject Tag` を設定。

### 4. 実行ファイル添付の隔離
*   **要件**: 外部からの `.exe` 添付ファイルを隔離せよ。
*   **設定**: `Incoming Mail Policy` に `Content Filter` を適用し、拡張子一致で `Quarantine` アクションを実行。

### 5. 大容量メールの拒否
*   **要件**: 20MB を超えるメールは送信元に差し戻せ。
*   **設定**: `Content Filter` で `Message Size > 20M` を条件にし、`Bounce` アクションを設定。

### 6. LDAP グループポリシーの実装
*   **要件**: AD の `Marketing` グループのみ SNS 通知メールを許可せよ。

### 7. ウイルス検知時の管理者通知
*   **要件**: ウイルスを検知した際、IT 管理者に MID 情報を通知せよ。

### 8. Outgoing 免責事項の自動付与
*   **要件**: 全外部送信メールの末尾に「本メールは機密情報を含みます...」を追記せよ。

### 9. 添付ファイル内のキーワード検知
*   **要件**: 添付の PDF 内に「極秘」という文字があれば、そのメールを暗号化せよ。

### 10. メッセージスプリットのログ確認
*   **要件**: 複数宛先のメールが個別のポリシーで処理される様子をログで確認せよ。
*   **確認**: `mail_logs` で同じ `ICID` から複数の `MID` が生成されることを確認。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: 特定の受信者に対してスパム設定を変更したが反映されない。`mail_logs` を見ると `Default Policy` が適用されている。なぜか？
    *   **回答**: その受信者のアドレスが **作成した個別ポリシーの `Add Recipients` に正しく登録されていない** か、あるいは **上位のポリシーに先にマッチしている** 可能性があります。
2.  **Design**: 全社員に対し、特定の外部ドメイン `malicious.org` へのメール送信を禁止したい。どのポリシーで設定すべきか？
    *   **回答**: **Outgoing Mail Policies** において、送信者を `All` に、Content Filter の条件で宛先（Envelope To）を `malicious.org` に指定して `Drop` します。
3.  **コンフィグ読解**: `policyconfig` の出力で、あるポリシーの Anti-Spam 列が `(Inherited)` となっている意味は？
    *   **回答**: そのポリシーでは独自のスパム設定を行わず、**Default Policy の設定をそのまま使用している** ことを意味します。
4.  **実装**: 1 通のメールに 2 人の受信者（A さん：標準ポリシー、B さん：厳しいポリシー）が含まれる場合、ESA はどのように処理するか？
    *   **回答**: メッセージを **スプリット (Split)** し、それぞれに適用されるポリシーに従って別々の MID として処理を継続します。
5.  **Design**: メールの方向性（Incoming/Outgoing）を ESA が判断する基準は？
    *   **回答**: メッセージを受信した **リスナー (Listener)** の種類（Public か Private か）によって判断されます。

---

## 🔗 参考リソース

*   **Cisco Secure Email Configuration Guide**: [Mail Policies and Pipeline](https://www.cisco.com/c/en/us/td/docs/security/esa/esa11-1/user_guide/b_ESA_Admin_Guide_11_1.html)
*   **Cisco Live (BRKSEC-2041)**: [Email Security Content Filtering Best Practices](https://www.ciscolive.com/)
*   **Technical Note**: [Understanding Message Splitting in ESA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117970-technote-esa-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: ESA の Mail Policy は「誰に対するルールか」を決める場所であり、実際の「何をするか」という細かいロジックは Content Filter に記述されることが多いです。
*   **注意点**: ラボ試験では、**設定変更後に `Commit Changes` を忘れると一切反映されない** ため、操作の最後には必ず Commit を行うよう徹底してください。
