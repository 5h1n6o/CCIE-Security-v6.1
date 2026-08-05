---
layout: default
title: 1.4.c-Reporting
nav_order: 3
parent: 1.4-FMC-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.4.c Reporting

Cisco Secure Firewall Management Center (FMC) における**レポーティング（Reporting）**は、管理下のデバイスから収集した大量のイベントデータ（接続、侵入、マルウェア、SSLなど）を、管理者やエグゼクティブ向けに視覚化・要約する機能です。CCIE Security v6.1 ラボ試験では、特定の要件に基づいたレポートテンプレートの作成、フィルターの適用、およびスケジュールによる自動配布の設定能力が問われます。

---

## 📘 概要

*   **機能概要**: 収集したイベントをグラフや表形式でまとめ、PDF、HTML、またはCSV形式で出力します。
*   **利用目的**: ネットワークの脅威動向の把握、コンプライアンスの証明、週次・月次の運用報告、およびキャパシティプランニングに利用されます。
*   **利用場面**: 「過去24時間にブロックされた全マルウェアイベントの要約」を毎朝メールで受信する、あるいは「特定ユーザーの全Webアクセス履歴」を監査用に抽出する場合などに活用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **出力フォーマット** | PDF, HTML, CSV |
| **主要要素** | レポートテンプレート、セクション（チャート/テーブル）、スケジュール |
| **配布方法** | ローカル保存、Email（SMTP連携） |
| **メリット** | 複雑なイベントデータを一目で理解可能な「ビジネス言語」に変換できる |
| **デメリット** | 大規模なレポート生成時にFMCのリソース（CPU/メモリ）を一時的に消費する |
| **対応機種** | 全てのFMCモデル、ASDM管理のASA（限定的機能） |
| **設計上の注意点** | データベース内のイベント保持期間（Database Retention）がレポート期間に影響する |

---

## 🏗 動作原理

レポーティング機能は、FMCのデータベースエンジンと密接に連携して動作します。

```text
[ Managed Device (FTD) ]
   ↓ (Events: Connection/Intrusion)
[ Cisco FMC Database ] --- (PostgreSQL等に蓄積)
   ↓ (Report Engine Query)
[ Report Template ] --- (Layout/Filter apply)
   ↓
[ Output Generation ] --- (PDF/HTML/CSV)
   ↓
[ Distribution ] --- (Email Server / Local Storage)
```

---

## ⚙ 動作シーケンス

1.  **データ集約**: FTDがパケットを処理し、セキュリティイベントを生成してSFTunnel経由でFMCへ送信します。
2.  **クエリ実行**: レポート生成時（手動またはスケジュール）、レポートエンジンがテンプレートで定義された期間とフィルター条件に従ってデータベースを検索します。
3.  **セクションのレンダリング**: 抽出されたデータが、棒グラフ、円グラフ、またはテーブル（表）として視覚化されます。
4.  **フォーマット変換**: 指定された形式（例: PDF）にコンパイルされます。
5.  **スケジュール配布**: スケジューラが有効な場合、指定されたEmailアドレスへSMTP経由でレポートが添付送信されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **テンプレートのカスタマイズ**: 試験では「デフォルトのテンプレートではなく、独自のセクションを持つレポートを作成せよ」という指示が出ます。**Analysis > Reporting > Report Templates** からの新規作成手順が必須です。
*   **フィルターの適用**: 「特定の送信元ゾーンからの通信のみをレポートに含める」といった詳細なフィルター条件の構成が求められます。
*   **スケジュール設定**: レポート生成と配布を自動化するため、**System > Tools > Scheduling** でのタスク登録が問われます。
*   **SMTP連携**: レポートを送信するためのSMTPサーバー設定（System > Configuration > Email Notification）が事前条件として含まれる場合があります。

---

## 🛠 設定方法

### 1. レポートテンプレートの作成 (FMC GUI)
1.  **Analysis > Reporting > Report Templates** に移動し、**Create Report Template** をクリックします。
2.  **Report Title** を入力し、**Add Section** を選択します。
3.  **Table/Chart** を選び、対象となるデータ（例: Connection Events）を指定します。
4.  **Search/Filter** を設定して、特定のトラフィックに絞り込み、**Save** します。

### 2. レポートのスケジュール配布設定
1.  **System > Tools > Scheduling** に移動します。
2.  **Add Task** をクリックし、**Job Type** で「Report」を選択します。
3.  作成したテンプレートを選択し、**Recurrence**（頻度）を設定します。
4.  **Email To** 欄に宛先アドレスを入力し、保存・有効化します。

---

## 🔍 検証コマンド

FMCのレポート機能は主にGUIベースですが、バックエンドの確認が可能です。

| 目的 | コマンド（FMC CLI） |
| :--- | :--- |
| **レポート生成プロセスの確認** | <code>tail -f /var/log/messages \| grep -i report</code> |
| **Email送信ステータスの確認** | <code>tail -f /var/log/maillog</code> |
| **データベース接続状態の確認** | <code>omniquery.pl "SELECT * FROM connection_events LIMIT 1"</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| レポートの中身が「空」 | 期間指定ミスまたはデータ未着 | フィルターで指定した期間に実際にイベントが発生しているか <code>Analysis > Connections</code> で確認。 |
| スケジュール送信されない | SMTP設定またはスケジューラ無効 | <code>System > Configuration > Email Notification</code> のテスト送信を実行。 |
| 生成中にエラーが出る | リソース不足またはDB破損 | FMCのディスク容量を確認し、一度に生成するレポートのデータ量を減らす。 |
| グラフが正しく表示されない | ブラウザの互換性またはプラグイン | PDF形式で出力し、FMC GUI外での表示を試みる。 |

---

## ⚠ 制限事項

*   **データ保持**: FMCのデータベース設定でイベントが早期にパージされるよう設定されている場合、古い期間のレポートは生成できません。
*   **最大サイズ**: 生成されるレポートファイル（特にPDF）のサイズには上限があり、大量の生データを含めると生成に失敗することがあります。
*   **ライセンス**: 基本的なロギングとレポーティングにライセンスは不要ですが、マルウェアや侵入データのレポートには各サブスクリプションが必要です。

---

## 🔄 他技術との関連

*   **Logging**: レポートのソースとなるデータそのものです。
*   **Dashboards**: レポートは静的なドキュメントですが、ダッシュボードはFMC上でリアルタイムに更新される動的なレポートとして機能します。
*   **Search Filters**: 保存された検索条件（Search Objects）をレポートテンプレートの条件として流用できます。

---

## 🧩 比較表

### FMC Reports vs FMC Dashboards

| 特徴 | Reports | Dashboards |
| :--- | :--- | :--- |
| **性質** | 静的（特定の時点の要約） | 動的（常に最新データを表示） |
| **配布** | メールやファイルとして外部配布可能 | FMC GUI内での閲覧に限定 |
| **形式** | PDF, HTML, CSV | ウィジェット、チャート |
| **用途** | 監査、定期報告、長期分析 | リアルタイム監視、即時対応 |

---

## 💡 ベストプラクティス

*   **段階的セクション**: レポートの最初はエグゼクティブサマリー（円グラフなど）から始め、後ろのセクションで詳細なテーブル（IPアドレス一覧など）を配置します。
*   **オフピーク生成**: 大規模なレポートはネットワーク負荷の低い夜間にスケジュール生成するよう構成します。
*   **CSVの活用**: サードパーティの分析ツール（ExcelやBIツール）で再加工する必要がある場合は、PDFではなくCSV形式を選択します.

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な脅威サマリーレポート
*   **要件**: 全デバイスの「重大な侵入イベント」のサマリーを作成せよ。
*   **設定**: Analysis > Reporting > Template作成。Intrusion Eventsをソースにし、Severity: Highでフィルター。

### 2. 特定ユーザーのトラフィック追跡
*   **要件**: ユーザー `John_Doe` の過去1週間のWebアクセス履歴をCSVで出力せよ。
*   **設定**: Connection Eventsをソースにし、User: John_Doe でフィルター。出力形式をCSVに指定。

### 3. 週次マルウェア通知の自動化
*   **要件**: 毎週月曜午前8時に、前週のマルウェア検知結果を管理者にメールせよ。
*   **設定**: Schedulingタスクで「Weekly」を指定。Email Toを設定し、Malwareテンプレートを紐付け。

### 4. 許可されたアプリケーションのトップ10
*   **要件**: 帯域を消費しているアプリケーション上位10位を円グラフで表示せよ。
*   **設定**: Connection Events。Chart形式で「Application」をカテゴリーに選択し、Countでソート。

### 5. 監査ログレポート（変更履歴）
*   **要件**: FMCの設定変更を誰がいつ行ったかのリストを作成せよ。
*   **設定**: Audit Logsをソースとしたテンプレートを作成。

### 6. SSL復号化トラフィックの統計
*   **要件**: SSLポリシーで復号されたトラフィックの割合をレポートせよ。
*   **設定**: SSL Eventsをソースに、Action: Decrypt を条件に含める。

### 7. レポートへのロゴ追加（ブランディング）
*   **要件**: 生成されるPDFのヘッダーに会社のロゴを表示せよ。
*   **設定**: Report TemplateのGlobal設定で画像をアップロード。

### 8. インパクトレベル1のイベント抽出
*   **要件**: 最も危険な攻撃（Impact 1）のみを抽出した技術詳細レポートを作成せよ。
*   **設定**: Intrusion Eventsフィルターで Impact: 1 を指定。

### 9. 特定サブネット間の接続レポート
*   **要件**: 10.1.1.0/24 から DMZ への全通信を記録せよ。
*   **設定**: Source Network と Destination Network オブジェクトをフィルターに使用。

### 10. レポート生成失敗の確認
*   **課題**: スケジュールタスクが失敗した際の履歴を確認せよ。
*   **実行**: System > Tools > Scheduling の **Task Status** ページを確認。

---

## ❓ 想定試験問題

1.  **実装**: FMCにおいて、毎日深夜1時に過去24時間の「拒否された接続（Blocked Connections）」をPDFで作成し、指定されたEmailへ送信するスケジューラを構成しなさい。
2.  **トラブルシュート**: レポートテンプレートに「Application」セクションを追加したが、出力結果にアプリケーション名が表示されない。考えられる原因は何か？
    *   **回答**: アクセスコントロールルールでアプリケーション識別（AVC）が有効になっていない、あるいはロギングが「開始時」に設定されており情報が欠落している。
3.  **Design**: 経営層向けに、セキュリティ投資の正当性を証明するためのレポートに含めるべき最適なデータは何か？
    *   **回答**: 「ブロックされた脅威の総数」、「Impact 1イベントの減少傾向」、「高リスクアプリケーションの使用状況」などの要約グラフ。
4.  **実装**: レポートの特定のセクションにのみ、特定の宛先IPアドレスを除外するフィルターを適用する方法を述べなさい。
5.  **コンフィグ読解**: スケジューラ設定画面のスクリーンショットが提示された際、次回のレポート実行日時と宛先を特定しなさい。

---

## 🔗 参考リソース

*   **Cisco Configuration Guides**
    *   [Cisco Secure Firewall Management Center Administration Guide, 7.1 - Reporting](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/reports.html)
*   **Cisco Live (Videos & Slides)**
    *   [BRKSEC-3020: Troubleshooting Firewall Threat Defense (FTD)](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-3020)
*   **Technical Notes**
    *   [Configuring and Troubleshooting FMC Scheduled Reports](https://www.cisco.com/c/en/us/support/docs/security/firepower-management-center/215354-configure-syslog-on-firepower-firewall-m.html)

---

## 📝 **補足（Notes）**  

*   **学習メモ**: CCIEラボでは、レポーティング自体が主目的になることは少ないですが、他の機能（マルウェア対策やIDS）の設定後に「正しく動作していることをレポートで証明せよ」という形式で組み合わされることがあります。
*   **図解**: 常に「データ源（DB）→ フィルター（クエリ）→ テンプレート（ガワ）→ 配布（SMTP）」の流れを意識して設定してください。
*   **注意点**: ラボ環境ではSMTPサーバーの到達性に制限がある場合があるため、Email通知の要件があるときは、まずSMTP設定が正しいかを真っ先に確認するのが鉄則です。
