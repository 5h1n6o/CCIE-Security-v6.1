---
layout: default
title: 5.7.e-Encryption
nav_order: 5
parent: 5.7-Email-security
grand_parent: 5.0-Advanced-Threat-Protection
---

# 5.7.e Encryption

Cisco Secure Email (旧 Email Security Appliance: ESA) における **Encryption（暗号化）** は、組織外へ送信される電子メールの機密性を確保するための重要な機能です。主に **Cisco Secure Email Encryption Service** (旧 CRES: Cisco Registered Envelope Service) を利用し、受信者が特別なソフトウェアを持っていなくても、安全に暗号化されたメッセージを閲覧できる仕組みを提供します。

---

## 📘 概要

*   **機能概要**: 特定のポリシー（DLPやコンテンツフィルタ）に合致したメールを自動的に暗号化し、意図した受信者のみが復号して閲覧できるようにする機能です。
*   **利用目的**: 輸送中のデータの盗聴防止、機密情報の保護、および法規制（PCI-DSS, HIPAA等）への準拠。
*   **どのような場面で利用するか**:
    *   **機密情報の送信**: 契約書、財務データ、個人情報を含むメールを外部へ送る際。
    *   **B2B/B2C 通信**: 相手先が TLS をサポートしていない、あるいはより強固な認証が必要な場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要技術** | **PXE (PostX Envelope)** 方式、TLS (Transport Layer Security)。 |
| **暗号化方式** | **Envelope Encryption** (メールをHTMLファイルとしてカプセル化)。 |
| **認証基盤** | Cisco Secure Email Encryption Service (クラウド上のキー管理サーバ)。 |
| **トリガー** | コンテンツフィルタ、DLP ポリシー、または件名のタグ（例: `[SECURE]`）。 |
| **メリット** | 受信者側の事前準備が不要。パスワード保護や閲覧期限の設定が可能。 |
| **デメリット** | クラウドサービス (CRES) への到達性が必要。 |
| **設計上の注意点** | ファイアウォールで HTTPS (TCP/443) のアウトバウンド許可が必須。 |

---

## 🏗 動作原理

Cisco Secure Email Encryption は、クラウドベースのキー管理システムを利用した「エンベロープ（封筒）方式」を採用しています。

```text
[ Sender ]
    ↓ (1) 送信メール
[ ESA (Outgoing Pipeline) ]
    ↓ (2) ポリシー一致 → 暗号化実行
    ↓ (3) CRES へ暗号鍵を登録 (HTTPS)
    ↓ (4) HTML エンベロープを作成
[ Recipient ]
    ↓ (5) HTML 添付ファイルを受信
    ↓ (6) ブラウザで開き、CRES で認証
    ↓ (7) CRES から鍵を取得し、ブラウザ上で復号
[ Read Message ]
```

---

## ⚙ 動作シーケンス

1.  **ポリシーマッチング**: Outgoing Mail Policy 内のコンテンツフィルタまたは DLP が、暗号化が必要なメッセージを特定します。
2.  **プロファイル適用**: 指定された **Encryption Profile**（鍵サイズ、認証方式、ロゴ等の外観）が適用されます。
3.  **キーの生成と登録**: ESA はメッセージごとにユニークな対称鍵を生成し、その鍵をクラウド上の Cisco Encryption Service (CRES) へセキュアに送信・保存します。
4.  **エンベロープ生成**: 元のメール本文と添付ファイルは暗号化され、暗号化データを含む特殊な HTML ファイル（HTML Envelope）として再構築されます。
5.  **配送**: ESA はこの HTML ファイルを添付したメールを受信者へ送信します。
6.  **復号プロセス**: 受信者が HTML を開くと、Cisco のポータルへリダイレクトされ、自身のメールアドレスでログイン（初回は登録）することで、CRES から復号鍵がブラウザに渡されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Encryption Profile の作成**: 暗号化設定の土台となるプロファイルの作成手順を正確に把握する必要があります。
*   **Content Filter との紐付け**: プロファイルを作っただけでは動作しません。「Action: Encrypt」をコンテンツフィルタで定義し、適切なプロファイルを選択するプロセスが重要です。
*   **Commit Changes の徹底**: ESA の設定変更は **Commit** しない限り反映されません。ラボ試験での「設定したのに動かない」の典型原因です。
*   **TCP/443 通信の確認**: ESA がクラウドサービスと通信できないと暗号化に失敗します。ネットワーク疎通のトラブルシュートが問われる可能性があります。
*   **件名タグによるトリガー**: ユーザーが件名に特定の文字列を入れた場合に暗号化を実行するルール作成は頻出です。

---

## 🛠 設定方法

### 1. 暗号化プロファイルの作成 (GUI)
1.  **Security Services > Cisco Email Encryption** に移動。
2.  **Add Encryption Profile** をクリック。
3.  `Profile Name` を入力（例: `Secure_Internal`）。
4.  `Key Provider` で **Cisco Secure Email Encryption Service** を選択。
5.  必要に応じて通知テキストやロゴをカスタマイズし、**Submit**。

### 2. コンテンツフィルタでの有効化
1.  **Mail Policies > Outgoing Content Filters**。
2.  **Add Filter** をクリック。
3.  `Conditions`: `Subject Header` が `[SECURE]` で始まる、などを設定。
4.  `Actions`: **Encrypt** を選択し、上記で作成したプロファイルを指定。
5.  **Submit** および **Commit Changes**。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **暗号化エンジン状態確認** | <code>encryptionstatus</code> |
| **設定済みプロファイル表示** | <code>encryptionconfig</code> |
| **リアルタイムログ監視** | <code>tail mail_logs</code> |
| **特定メッセージの暗号化確認** | <code>grep "Encryption match" mail_logs</code> |
| **CRES への接続テスト** | <code>telnet res.cisco.com 443</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 暗号化されない | ポリシー順位の誤り | コンテンツフィルタの優先順位を上げ、他のルールに食われていないか確認。 |
| 暗号化に失敗する | ライセンス未適用 | <code>featurekey</code> コマンドで Encryption ライセンスの状態を確認。 |
| 暗号化に失敗する | クラウド通信不可 | <code>mail_logs</code> で "connection error to CRES" を確認。FW設定を修正。 |
| 受信者が開けない | CRES アカウント未登録 | 受信者に CRES への登録を案内。または認証不要な方式への変更検討。 |

---

## ⚠ 制限事項

*   **インターネット依存**: 鍵の管理に CRES を使用するため、ESA と CRES 間のインターネット接続が断たれると、新規の暗号化は行えません。
*   **ファイルサイズ**: 暗号化によってメールサイズが増大するため、最大メッセージサイズ制限に抵触する可能性があります。
*   **サポート形式**: 非常に古いブラウザや、JavaScript を完全に無効化している環境では復号ポータルが動作しません。

---

## 🔄 他技術との関連

*   **5.7.b DLP**: DLP で機密情報（カード番号等）を検知した際の「アクション」として暗号化が多用されます。
*   **5.7.a Mail Policies**: 送信者グループ（例: 役員）ごとに異なる暗号化プロファイルを適用します。
*   **3.6 Monitoring**: 暗号化されたメッセージの統計情報をモニタリングし、レポートを生成します。

---

## 🧩 比較表

### Cisco Secure Email Encryption vs 一般的な PGP/S-MIME

| 特徴 | Cisco Secure Email Encryption | PGP / S-MIME |
| :--- | :--- | :--- |
| **鍵管理** | クラウドで一括管理 (CRES) | ユーザー間での公開鍵交換が必要 |
| **受信側の負担** | 低 (ブラウザのみ) | 高 (証明書のインストール等が必要) |
| **導入速度** | 即時 (ESAの設定のみ) | 緩やか (全ユーザーの鍵準備が必要) |
| **標準化** | 独自方式 (Envelope) | RFC 準拠 |

---

## 💡 ベストプラクティス

1.  **デフォルト TLS の併用**: まずは Opportunistic TLS を設定し、TLS が確立できない場合や、内容が特に重要な場合のみ CRES 暗号化をトリガーする階層化設計を推奨します。
2.  **件名タグの周知**: ユーザーが明示的に暗号化したい場合のために、特定のキーワード（`#encrypt#`など）をあらかじめ定義し、周知します。
3.  **期限設定**: 機密情報の性質に応じて、メッセージの有効期限（例: 30日間）を設定し、不要な情報の露出期間を短縮します。

---

## 📝 ラボ学習・設定サンプル例

### 1. キーワードベースの暗号化
*   **要件**: 件名に「Confidential」が含まれる場合、プロファイル `Level_1` で暗号化せよ。
*   **設定**: Content Filter > `Condition: Header contains "Confidential"` > `Action: Encrypt (Level_1)`。

### 2. 特定ドメインへの暗号化強制
*   **要件**: パートナー企業 `partner.com` 宛のメールはすべて暗号化せよ。
*   **設定**: Content Filter > `Condition: Envelope To matches "partner.com"` > `Action: Encrypt`.

### 3. DLP 連携シナリオ
*   **要件**: PCI-DSS 違反（カード番号等）が検知された場合、自動的に暗号化を適用せよ。
*   **設定**: DLP Policy > `Action: Deliver with Encryption`.

### 4. 添付ファイル暗号化
*   **要件**: `.pdf` ファイルが添付されている場合のみ暗号化を実行せよ。

### 5. 役員用カスタムプロファイル
*   **要件**: 役員からのメールには会社のロゴが入った特別な暗号化通知を表示せよ。

### 6. メッセージ有効期限の構成
*   **要件**: 暗号化されたメールが 7 日後に閲覧できなくなるように設定せよ。
*   **設定**: Encryption Profile > `Message Expiration: 7 days`。

### 7. 件名からのタグ削除
*   **要件**: 暗号化トリガーに使用した `[SECURE]` タグを受信者に送る前に削除せよ。
*   **設定**: Action セクションで `Strip subject tag` をオンにする。

### 8. CLI によるプロファイル確認
*   **操作**: <code>encryptionconfig</code> を実行し、設定内容をテキストベースで確認せよ。

### 9. 接続障害のシミュレーション
*   **操作**: ESA の DNS 設定を壊し、CRES への名前解決失敗時の `mail_logs` の挙動を観察せよ。

### 10. 大容量ファイルの暗号化制限
*   **要件**: 5MB を超える暗号化メールは送信を拒否し、送信者に通知せよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: 暗号化プロファイルを作成し、コンテンツフィルタを適用したが、外部にプレーンテキストで届いてしまう。原因として考えられることは？
    *   **回答**: フィルタの **Commit Changes** が行われていない、または該当ポリシーが **Outgoing Mail Policy** で有効になっていない。
2.  **Design**: 受信者がインターネット接続のない環境にいる。Cisco Secure Email Encryption Service は利用可能か？
    *   **回答**: **不可**。復号鍵の取得と認証のために、受信者のブラウザが CRES クラウドにアクセスできる必要があります。
3.  **コンフィグ読解**: `mail_logs` に `Encryption: Policy match, profile 'Standard_Secure' triggered` とある。このメッセージは最終的にどうなったか？
    *   **回答**: 指定されたプロファイルで暗号化され、CRES への鍵登録が行われた後に配送された。
4.  **実装**: 件名に `[CRYPT]` とある場合のみ暗号化したい。フィルタの順位はどうすべきか？
    *   **回答**: 汎用的な許可ルールよりも **上位** に配置し、確実にキャッチできるようにする。
5.  **Design**: TLS 1.3 を強制したいが、相手側が対応していない場合でもセキュアに届けたい。最適な ESA の構成は？
    *   **回答**: **Destination Controls** で TLS を `Required` にし、失敗した場合のフォールバック先として **Encryption Profile** を指定する。

---

## 🔗 参考リソース

*   **Cisco Secure Email Configuration Guide**: [Encryption Settings](https://www.cisco.com/c/en/us/td/docs/security/esa/esa11-1/user_guide/b_ESA_Admin_Guide_11_1.html)
*   **Cisco Live (BRKSEC-2041)**: [Securing Data-in-Motion with Email Encryption](https://www.ciscolive.com/)
*   **Technical Note**: [Troubleshooting CRES/PXE Connectivity](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117970-technote-esa-00.html)
*   **Design Guide**: [Best Practices for Email Encryption Deployment](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/umbrella-deployment-guide.html)

---

## 📝 **補足（Notes）**  

*   **学習メモ**: 「暗号化は封筒を二重にするイメージ」です。外側の封筒（HTMLファイル）を開けるには、Ciscoという管理所（CRES）から合鍵（復号鍵）を貰う必要があります。
*   **図解**: `ESA --(鍵)--> CRES` / `ESA --(HTML)--> Recipient --(認証)--> CRES --(鍵)--> Recipient`.
*   **注意点**: ラボ試験では、**証明書エラー** が出ると検証が進まないため、ESA 自体の証明書管理や、ブラウザでの信頼設定にも気を配ってください。
