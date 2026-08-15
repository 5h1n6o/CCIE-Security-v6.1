---
layout: default
title: 5.7.c-Quarantine
nav_order: 3
parent: 5.7-Email-security
grand_parent: 5.0-Advanced-Threat-Protection
---

# 5.7.c Quarantine

Cisco Secure Email (旧 Email Security Appliance: ESA) における **Quarantine（隔離）** は、セキュリティポリシー（アンチスパム、アンチウイルス、コンテンツフィルタ、DLPなど）に一致した疑わしいメッセージを、最終的な配送判断を下すまで安全な場所に一時的に保持する機能です。

---

## 📘 概要

*   **機能概要**: 脅威を含んでいる可能性がある、または企業のコンプライアンスに違反しているメッセージを、宛先の受信箱に届ける前にシステム上の隔離領域へ移動・保管します。
*   **利用目的**: 誤検知（False Positive）への対応機会を管理者に提供し、悪意のあるメールがユーザーの端末に直接届くのを防ぎます。
*   **どのような場面で利用するか**:
    *   **スパム判定**: 確実ではないがスパムの疑いがあるメールを保持し、ユーザー自身に判断させる（Spam Quarantine）。
    *   **コンテンツ違反**: 機密情報を含む可能性があるメールを保持し、セキュリティ管理者が承認後に送信する（Policy Quarantine）。
    *   **ウイルス/アウトブレイク**: 新種のウイルスが発生した際、パターンファイルが更新されるまで一時的に待機させる（Outbreak Quarantine）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **隔離の種類** | **Spam Quarantine** (スパム用) と **PVO Quarantine** (Policy, Virus, Outbreak用)。 |
| **保存場所** | **Local** (ESA本体) または **External** (Cisco Secure Email Management Appliance: SMA)。 |
| **管理アクション** | **Release** (配送), **Delete** (削除), **Deliver** (強制配送), **Modify** (編集)。 |
| **保持期間** | 各隔離設定ごとに設定可能な有効期限 (Retention Period)。 |
| **ユーザーアクセス** | エンドユーザーが自身のスパム隔離を管理できる「End-User Console」。 |
| **通知機能** | 隔離されたことを送信者、受信者、または管理者に通知する機能。 |

---

## 🏗 動作原理

メッセージが隔離されるまでのフロー：

```text
Incoming Email
   ↓
Email Pipeline (Security Services)
   ↓ (1) Policy Match (e.g., Content Filter "High Risk")
   ↓ (2) Action: Quarantine ("Policy_Q")
[ Quarantine Engine ]
   ↓ (3) Write to Disk/Database
   ↓ (4) Send Notification (Optional)
Isolated State (Waiting for Release/Expiration)
```


---

## ⚙ 動作シーケンス

1.  **判定**: メールパイプライン内の各エンジン（Anti-Spam, Content Filter等）がメッセージを検査します。
2.  **アクション実行**: フィルタールールで `Quarantine` アクションが指定されている場合、メッセージの配送プロセスが停止されます。
3.  **メタデータ記録**: メッセージのコピーが隔離データベースに保存され、受信者、送信者、理由、有効期限などの情報が紐付けられます。
4.  **メンテナンス**: 有効期限が切れたメッセージは、システムによって自動的に削除されます。
5.  **解放**: 管理者またはユーザーが「Release」を選択すると、メッセージは隔離を離れ、配送プロセス（残りのパイプラインまたは宛先MTAへ）を再開します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **PVO隔離の作成と適用**: 特定のコンテンツフィルタに一致したメールを、Default隔離ではなく**新規作成したカスタム隔離（Custom Quarantine）**へ入れる設定が頻出です。
*   **保持期間（Retention Period）の変更**: 試験要件で「隔離されたメールは48時間保持せよ」といった指定がある場合、`quarantineconfig` または GUI での時間設定が必要です。
*   **End-User Spam Quarantine (EUQ)**: ユーザーがセルフサービスで隔離を閲覧できるように、認証方式（LDAP連携等）を正しく構成する能力が問われます。
*   **Centralized Quarantine**: ESA で受信したメールを SMA (Security Management Appliance) へ転送して一括管理する構成は、実機試験での典型的なシナリオです。
*   **トラブルシュート**: `mail_logs` における `quarantined` ステータスの確認。メッセージがどの隔離に入ったか、なぜ消えたか（期限切れ等）を追跡します。

---

## 🛠 設定方法

### 1. PVO隔離の作成 (GUI)
1.  **Monitor > Quarantines** に移動し、**Add Policy Quarantine** を選択。
2.  `Quarantine Name` を入力（例: `Confidential_Leak`）。
3.  `Retention Period`（保持期間）を設定。
4.  `Overflow Action`（容量超過時の動作）を `Deliver` または `Drop` から選択。

### 2. コンテンツフィルターへの適用
1.  **Mail Policies > Incoming/Outgoing Content Filters**。
2.  アクションの `Add Action` で **Quarantine** を選択し、上記で作成した隔離領域を指定します。

### 3. CLI による状態確認
```bash
# 隔離全体のステータス確認
esa.lab# quarantinestatus

# 特定の隔離内のメッセージ一覧表示
esa.lab# showmessage -q Confidential_Leak
```


---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **隔離の統計情報表示** | <code>quarantinestatus</code> |
| **隔離エンジンの構成表示** | <code>quarantineconfig</code> |
| **特定の隔離内のメッセージ検索** | <code>showmessage -q [Quarantine_Name]</code> |
| **メッセージの強制解放** | <code>releasemessage [MID]</code> |
| **リアルタイムログ監視** | <code>tail mail_logs</code> ( "quarantined to" を検索) |



---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 隔離されたはずのメールが見当たらない | 有効期限切れによる自動削除 | <code>mail_logs</code> で "deleted by system" を確認。 |
| 隔離データベースが満杯 | ディスク容量不足または設定値上限 | <code>status detail</code> で容量確認。不要なメールの削除。 |
| ユーザーが EUQ にログインできない | LDAP連携またはURL設定ミス | <code>testauth</code> コマンドで認証をテスト。SMA側のアクセス設定確認。 |
| 解放(Release)したのに届かない | 下流のメールサーバでの拒否 | <code>mail_logs</code> で DCID を追跡し、配送エラーを確認。 |



---

## ⚠ 制限事項

*   **ストレージ制限**: 隔離メッセージはディスク（または一部メモリ）を消費するため、無制限に保存することはできません。
*   **外部隔離への遅延**: SMAへメッセージを転送する場合、ネットワーク状況によりダッシュボードへの反映にタイムラグが生じることがあります。
*   **暗号化**: 隔離されたメッセージはESA/SMAに生データ（または内部形式）で保存されるため、物理的なセキュリティ保護が必要です。

---

## 🔄 他技術との関連

*   **5.7.a Mail Policies**: 隔離はメールポリシーのアクションとして定義されます。
*   **5.7.b DLP**: 機密情報漏洩検知時の主要なアクションは「暗号化」または「隔離」です。
*   **4.7 Active Directory**: スパム隔離へのエンドユーザーアクセスや、管理者通知の宛先解決に使用されます。

---

## 🧩 比較表

### Policy Quarantine vs Spam Quarantine

| 特徴 | Policy Quarantine (PVO) | Spam Quarantine |
| :--- | :--- | :--- |
| **管理対象** | ポリシー、ウイルス、アウトブレイク | スパムメッセージ |
| **主な管理者** | ITセキュリティ担当者 | エンドユーザーおよび管理者 |
| **トリガー** | Content Filter, DLP, Anti-Virus | Anti-Spam (CASE) |
| **構成単位** | 複数のカスタム隔離を作成可能 | 通常はシステムで単一の領域 |

---

## 💡 ベストプラクティス

1.  **通知の有効化**: ポリシー隔離時は、受信者（または管理者）に「メッセージが保留されている」旨の通知を送り、業務遅延を最小限にします。
2.  **容量警告の設定**: 隔離容量が 80% を超えた際に SNMP トラップや Eメールで管理者に警告が飛ぶように構成します。
3.  **期限の最適化**: スパムは 7 日間、ポリシー違反は 30 日間など、データの重要度に応じた保持期間を設定し、リソースを節約します。

---

## 📝 ラボ学習・設定サンプル例

### 1. カスタムポリシー隔離の作成
*   **要件**: 名前「HR_Review」、保持期間 14 日間の隔離を作成せよ。

### 2. 特定キーワードによる隔離
*   **要件**: 本文に「Top Secret」が含まれる外部送信メールを「HR_Review」へ隔離せよ。

### 3. 送信者への通知設定
*   **要件**: メールが隔離された際、送信者に「ポリシー違反により審査中」と返信せよ。

### 4. スパム隔離の保持期間短縮
*   **要件**: デフォルト 14 日のスパム保持期間をリソース節約のため 3 日に変更せよ。

### 5. 隔離メールの解放 (CLI)
*   **要件**: CLI を使用して MID 105 のメッセージを「HR_Review」から解放せよ。

### 6. アウトブレイクフィルタとの連携
*   **要件**: VDB更新前の疑わしいファイルを最大 12 時間隔離せよ。

### 7. エンドユーザーコンソールの有効化
*   **要件**: ユーザーがブラウザから 10.1.1.5 へアクセスしてスパムを閲覧できるようにせよ。

### 8. 隔離オーバーフロー時の挙動設定
*   **要件**: 「HR_Review」が満杯になった場合、古いものから「削除(Drop)」するように設定せよ。

### 9. 受信者ごとの隔離サマリー送信
*   **要件**: 毎日 17 時に、隔離されたスパムのリストを各ユーザーにメール通知せよ。

### 10. SMAへの集中隔離 (Centralized)
*   **要件**: ESAの隔離サービスを停止し、すべての隔離を SMA (10.1.1.20) で行うように構成せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `quarantineconfig` の出力で `Internal Spam Quarantine: Disabled` となっている場合、スパムメールのアクションを `Quarantine` に設定するとどうなるか？
    *   **回答**: 隔離サービスが無効であるため、メッセージは通常配送されるか、構成によってはエラー（またはドロップ）となる。
2.  **Design**: 大規模なメール環境で、ESA 本体のパフォーマンスを維持しつつ、長期間の隔離保持が必要な場合の最適な設計は？
    *   **回答**: **Centralized Quarantine** を使用し、専用アプライアンスである **SMA** に隔離処理をオフロードする。
3.  **トラブルシュート**: 管理者が GUI からメッセージを Release したが、受信者に届かない。最初に確認すべきログは？
    *   **回答**: `mail_logs` を確認し、Release 後に生成された **DCID** (Delivery Connection ID) と、宛先サーバからの応答（250 OK 等）を確認する。
4.  **実装**: コンテンツフィルタで複数の隔離を順番に指定した場合、メッセージはどのように扱われるか？
    *   **回答**: ESA は最初にマッチした隔離アクションを実行する（メッセージは重複して隔離されない）。
5.  **Design**: エンドユーザーがスパム隔離内のメールを誤って「Release」してしまうのを防ぎたい。どのような制御が可能か？
    *   **回答**: **End-User Console の権限設定**により、Release（配送）を許可せず、閲覧または削除のみを許可するように構成する。

---

## 🔗 参考リソース

*   **Cisco Secure Email Configuration Guide**: [Chapter: Quarantines](https://www.cisco.com/c/en/us/td/docs/security/esa/esa11-1/user_guide/b_ESA_Admin_Guide_11_1.html)
*   **Cisco Live (BRKSEC-2041)**: [Managing Email Quarantines Effectively](https://www.ciscolive.com/)
*   **Technical Note**: [How to troubleshoot Centralized Spam Quarantine](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117970-technote-esa-00.html)

---

## 📝 **補足（Notes）**  
- **学習メモ**: 「隔離はゴミ箱ではなく、一時的な待合室」です。
- **注意点**: ラボ試験では、**PVO隔離**と**Spam隔離**の設定箇所が GUI 上で分かれている（Monitor 側か Mail Policies 側か）ことに注意してください。
