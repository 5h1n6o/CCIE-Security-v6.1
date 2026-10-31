---
layout: default
title: 1.4.c-Reporting
nav_order: 3
parent: 1.4-FMC-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.4.c Reporting

Cisco Secure Firewall Management Center (FMC: 旧 Firepower Management Center) における **Reporting（レポーティング機能）** は、管理下の Secure Firewall Threat Defense (FTD) や NGIPS アプライアンスが生成した膨大なセキュリティイベント（接続、侵入、ファイル/マルウェア、Security Intelligence、ユーザー・ホスト情報、ネットワークディスカバリ、ヘルスモニタリング等）を集約し、用途や読者（経営層、SOC アナリスト、システム管理者、コンプライアンス監査人）に応じたドキュメント（PDF、HTML、CSV 形式）として動的生成・定期配信するための高度な可視化・分析レポート出力基盤です。

CCIE Security v6.1 ラボ試験では、単なる標準レポートの出力にとどまらず、**Event View / Dashboard からの Report Designer を使用したカスタムテンプレートの構築、入力パラメータ（Input Parameters）を用いた動的フィルタリング、Task Scheduler による自動生成と Mail Relay 統合による Email 配信、リモートストレージ（NFS/SMB/SSH）へのアーカイブ保存、および生成制限（PDF 結果上限数等）とパフォーマンス最適化** を含めた統合的運用能力が評価されます。

---

## 📘 概要

* **機能概要**:
  * **標準レポート (Standard Reports)**: 脅威分析、ネットワークアクティビティ、ホストプロファイル、Intrusion/Malware 概要などあらかじめ定義されたテンプレートに基づく多角的なレポート生成。
  * **リスクレポート (Risk Reports)**: ネットワーク内の高リスクアプリケーション、攻撃の影響度（Impact Level）、脆弱性、マルウェアの蔓延状況をエグゼクティブ向けにグラフィカルな HTML 形式で集約出力。
  * **レポートデザイナー (Report Designer)**: Unified Event Viewer、Dashboard ウィジェット、Context Explorer、検索クエリ（Saved Searches）から 1 クリックでカスタムレポートテンプレートをグラフィカルに自動デザインする強力な設計ツール。
  * **スケジュールレポート (Automated / Scheduled Reports)**: FMC の Task Scheduling (`System > Tools > Scheduling`) 機能を介して、日次/週次/月次などの周期でレポート生成を自動化し、指定された宛先リストへ Email 添付送信。
  * **リモートストレージ連携 (Remote Storage Management)**: 大量の過去レポートファイルを FMC ローカルディスクではなく外部ストレージ（NFS、SMB、SSH/SCP）へ自動移動・バックアップ保存。
* **利用目的**:
  * 経営陣およびセキュリティ責任者への週次・月次セキュリティ状況（Security Posture）および投資対効果（ROI）の可視化報告。
  * コンプライアンス規制（PCI-DSS、HIPAA、ISO 27001 等）で要求されるセキュリティインシデントおよびアクセスログの定期的な文書化・長期保存。
  * ネットワーク内で発生した特定のセキュリティ攻撃（特定 Attack SID、マルウェアハッシュ SHA-256、被害ホスト IP）に対するフォレンジック詳細調査資料の作成。
* **どのような場面で利用するか**:
  * 定常的な SOC 運用における「週次セキュリティ脅威サマリー」の作成と自動配信。
  * 特定ホストのマルウェア感染が疑われる際、該当ホストのすべての通信履歴・侵入アラート・ファイル転送記録を統合した「Host Detail Report」の即時作成。
  * FMC ローカルディスク容量が逼迫した環境において、古い生成済み PDF レポートを外部 NFS/SMB レポジトリへ安全にアーカイブ退避させる運用。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **出力フォーマット** | **PDF** (レイアウト重視), **HTML** (インタラクティブ表示), **CSV** (ローデータ処理・最大 100 行/テーブル) |
| **作成手法** | ① 空白テンプレートから作成<br>② 既存テンプレートの複製<br>③ **Report Designer** (Event View/Dashboard から直接作成)<br>④ Dashboard / Workflow のインポート |
| **動的パラメータ** | **Input Parameters** (`$<Parameter_Name>`) を使用し、レポート生成時に宛先 IP、ユーザー名、タイムウィンドウ等の条件を動的入力可能 |
| **自動化・スケジュール** | `System > Tools > Scheduling` でジョブタイプ `Report` を選択。Recurring (繰り返し) または Once (1 回のみ) |
| **配信方式** | オンデマンドダウンロード、Email 自動送信 (SMTP / Mail Relay 必須)、リモートストレージ (NFS / SMB / SSH) 自動保存 |
| **PDF 出力結果制限** | 各セクションごとに最大取得行数（例: Top 1,000 件等）の上限あり。超過時は設定画面に黄色三角の警告表示 |
| **マルチドメイン階層** | 祖先ドメイン (Ancestor Domain) のテンプレートは参照のみ可能。生成・編集には現在のドメインへコピーが必要 |

---

## 🏗 動作原理

FMC の Reporting アーキテクチャは、FMC 内のイベントデータベース（MySQL/Sybase）、Dashboard / Workflow 構成要素、Report Templates、および Task Scheduler が相互に連携して動作します。

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│                      Firepower Management Center (FMC)                          │
│                                                                                  │
│  [ FMC Event Database ] (Connection, Intrusion, File/Malware, Discovery, Health)  │
│         │                                                                        │
│         │ Query / Search Constraints                                             │
│         ▼                                                                        │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │ [ Report Engine / Report Designer ]                                        │  │
│  │   ├── Report Templates (Section 1: Pie Chart, Section 2: Table View, etc.) │  │
│  │   ├── Input Parameters ($<Host_IP>, $<Time_Window>, $<Customer_Name>)       │  │
│  │   └── Document Attributes (Cover Page, Table of Contents, Header/Footer)   │  │
│  └────────────────────────────────────┬───────────────────────────────────────┘  │
│                                       │                                          │
│        ┌──────────────────────────────┼──────────────────────────────┐           │
│        │ On-Demand Generation         │ Scheduled Job (Task Scheduler│           │
│        ▼                              ▼                              ▼           │
│  ┌──────────────┐            ┌─────────────────┐            ┌─────────────────┐  │
│  │ Rendered File│            │ Mail Relay Host │            │ Remote Storage  │  │
│  │ (PDF/HTML/   │            │ (SMTP: Port 25) │            │ (NFS/SMB/SSH)   │  │
│  │  CSV)        │            └────────┬────────┘            └────────┬────────┘  │
│  └──────┬───────┘                     │                              │           │
└─────────┼─────────────────────────────┼──────────────────────────────┼───────────┘
          │                             │                              │            
          ▼                             ▼                              ▼            
  [ Local Browser ]             [ Email Recipients ]         [ External File Server ]
  (Direct Download)             (Attachment Notification)    (Long-Term Archiving)  
```

---

## ⚙ 動作シーケンス

レポートの定義から生成、外部配信、およびリモート保存までの処理順序は以下の通りです。

1. **レポートテンプレートの定義・カスタマイズ**:
   * 管理者は `Overview > Reporting > Report Templates` で手動作成、または Event View / Dashboard 上で `Report Designer` ボタンをクリック。
   * データベーステーブル（Intrusion Events、Connection Events、Application Statistics 等）、表示フォーマット（Table、Pie Chart、Bar Chart、Line Chart、Text）、取得制限数（Maximum Results）、検索フィルターを指定。
   * 必要に応じて動的プロンプトである **Input Parameters**（例: `$ Destination IP`）を追加。
2. **レポートの生成リクエスト (On-Demand または Scheduled)**:
   * **オンデマンド生成**: `Generate` ボタンをクリック。フォーマット (PDF/HTML/CSV)、グローバルタイムウィンドウ、Input Parameters の値を手動入力。
   * **スケジュール生成**: `System > Tools > Scheduling` でスケジュールジョブ（Job Type: `Report`）を作成し、実行周期（日次/週次/月次等）、パラメータ値、Email 送信先を登録。
3. **データベースクエリとレンダリング**:
   * FMC のレポーティングエンジンが各セクションに定義された SQL / データベースクエリを順次並列実行。
   * クエリ結果が指定フォーマット（PDF、HTML、または CSV 構造）にレンダリングされます。PDF の場合はヘッダー/フッター、表紙、目次が合成されます。
4. **出力ファイルの保存と配信**:
   * 生成されたレポートは FMC のローカルデータベース/ストレージ（`Overview > Reporting > Reports`）に格納。
   * Email 送信が有効な場合、設定された Mail Relay Host (SMTP) を経由して PDF/HTML/CSV 添付ファイル付きメールが送信。
   * リモートストレージが有効な場合、生成完了と同時に NFS/SMB/SSH サーバーへバックグラウンドで転送。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Report Designer を用いた迅速なカスタムテンプレート構築**:
   * 試験で「特定のイベント表示画面（例: アプリケーション別拒否通信）をレポート化せよ」と指示された場合、空白のテンプレートから手動でセクションを組み立てるのではなく、**該当する Event View や Context Explorer 画面の右上にある `Report Designer` ボタンをクリックしてテンプレート化する** のが最も高速でミスがありません。
2. **PDF 生成時の結果制限数（Maximum Results）と黄色三角警告の解決**:
   * レポートテンプレートのセクションで取得件数（`Maximum Results`）が多すぎる場合、PDF レポート生成ボタンがグレーアウト（非活性化）するか、生成時にエラーが発生します。
   * テンプレート編集画面でセクション名横に **黄色三角アイコン (⚠️)** が表示されている場合は、マウスオーバーして許容上限値を確認し、**取得制限数（例: Top 1000 や Top 500）を削減** して保存する必要があります。
3. **Scheduled Report における Mail Relay (SMTP) 統合と設定の順番**:
   * スケジュールレポートを Email 送信させる課題では、以下の順序を厳守します:
     1. **Mail Relay Host 設定**: `System > Configuration > Email Notification` で SMTP サーバー IP、Port (25)、From アドレスを設定。
     2. **Report Template 側の Email オプション設定**: テンプレート生成画面またはスケジュール作成画面で「Send Email」を選択し、宛先を記述。
     3. **Task Scheduler 登録**: `System > Tools > Scheduling` で Task を作成。
4. **Input Parameters (動的入力変数) の構文と活用**:
   * 1 つのレポートテンプレートで複数の対象（特定ホスト、特定ユーザー）に対応させるため、セクションの Search Constraints やタイトルに **`$<Parameter_Name>`** 形式の入力パラメータを組み込む問題が出題されます。
   * 生成時（Generate 時）または Task Scheduling 設定時に、パラメータの具体例（例: `10.1.1.50`）を入力して実行させます。
5. **CSV フォーマットの制限事項**:
   * レポート形式として CSV を選択した場合、グラフィック要素（Pie Chart、Bar Chart）やドキュメント属性（表紙、目次、ヘッダー/フッター）は一切適用されず、**各セクションの表データ（最大 100 行/テーブル）のみがテキスト形式で出力** されます。

---

## 🛠 設定方法

### 1. FMC GUI: Report Designer による Event View からのレポート作成

```text
[ Step 1: 分析画面でのフィルター設定 ]
1. Analysis > Connections > Events (または Intrusions > Events) に移動。
2. 画面上の Search 機能を使用して、目的のトラフィックをフィルタリング (例: Action = Block)。
3. 画面右上の "Report Designer" ボタンをクリック。

[ Step 2: テンプレートの調整と保存 ]
1. 自動生成された Report Template 画面が開く。
2. Report Title: "Blocked_Connection_Summary_Report" に変更。
3. 必要に応じて不要な Section を削除 (右側のゴミ箱アイコン)。
4. "Advanced" をクリックし、Document Attributes (Cover Page, Table of Contents, Page Header/Footer) を設定。
5. "Save" をクリック。
```

---

### 2. FMC GUI: 新規カスタム Report Template の手動作成と Input Parameter 設定

```text
1. Overview > Reporting > Report Templates タブを開く。
2. "Create Report Template" をクリック。
3. Report Title: "Host Threat Analysis - $<Host_IP>" を入力。
   - タイトル内の $<Host_IP> は Insert Input Parameter アイコンから追加。

[ Section 1 の追加: Intrusion Events Table ]
  - Add Table View アイコンをクリック。
  - Table: Intrusion Events
  - Format: Table View
  - Section Description: "Intrusion Events for $<Host_IP>"
  - Search: Edit をクリックし、"Destination IP" ＝ "$<Host_IP>" 条件を追加。
  - Maximum Results: Top 500

[ Section 2 の追加: Traffic by Application Pie Chart ]
  - Add Pie Chart アイコンをクリック。
  - Table: Connection Events
  - Format: Pie Chart
  - X-Axis: Application Name, Y-Axis: Total Bytes
  - Search: Edit をクリックし、"Initiator IP" ＝ "$<Host_IP>" 条件を追加。

4. "Save" をクリック。
```

---

### 3. FMC GUI: Task Scheduling によるレポート自動生成 ＆ Email 配信の構成

```text
[ Step 1: Mail Relay Host の事前設定 ]
1. System > Configuration > Email Notification を開く。
2. Mail Relay Host: 192.168.10.25 (SMTP サーバー IP)
3. Port: 25
4. From Address: fmc-reports@inelab.local
5. "Save" をクリック。

[ Step 2: Task Scheduler での定期ジョブ登録 ]
1. System > Tools > Scheduling を開く。
2. "Add Task" をクリック。
3. Job Type: "Report" を選択。
4. Schedule task to run: "Recurring" を選択。
5. Start On: 開始日時を指定。
6. Repeat Every: "1 Weeks" (毎週月曜日 08:00 AM に実行など)。
7. Job Name: "Weekly_Security_Digest_Job"
8. Report Template: "Blocked_Connection_Summary_Report" を選択。
9. Format: PDF
10. Email Status To: soc-team@inelab.local
11. "Save" をクリック。
```

---

### 4. FMC GUI: 生成済みレポートのリモートストレージ (NFS / SMB) 移動構成

```text
1. System > Configuration > Remote Storage Management を開く。
2. Storage Type: NFS (または SMB / SSH) を選択。
3. NFS Server IP: 192.168.10.200
4. Remote Path: /exports/fmc_reports
5. "Test Connection" を実行し、成功を確認して "Save"。
6. Overview > Reporting > Reports タブを開く。
7. 移動させたいレポートを選択し、"Move to Remote Storage" をクリック。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI / FMC Bash) | 表示確認ポイント |
| :--- | :--- | :--- |
| **FMC 内のレポート生成タスク処理状態追跡** | `sudo tail -f /var/log/messages` | `sfreporter` および `cron` プロセスの実行ログ |
| **FMC レポート生成エンジンのプロセス状態確認** | `sudo pmtool status \| grep -i report` | レポーティングプロセスの動作ステータス |
| **生成されたレポートファイル群のローカルパス確認** | `ls -la /var/sf/reports/` | FMC ディスク上の生成済み PDF/HTML/CSV 実ファイル |
| **Mail Relay 経由の SMTP 送信ログ確認** | `sudo tail -f /var/log/maillog` | FMC から SMTP サーバーへの 25 番ポート接続・送信ログ |
| **Task Scheduler の登録・実行履歴表示** | `System > Tools > Scheduling` (GUI) | スケジュールされた Task の次回実行時刻と過去の Job Status |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **PDF フォーマットでのレポート生成ボタンが選択できない (グレーアウト)** | テンプレート内の一部のセクションで `Maximum Results` の上限を超過している | `Overview > Reporting > Report Templates` | テンプレートを編集し、黄色三角アイコン (⚠️) がついたセクションの取得結果数を削減して保存。 |
| **スケジュールレポートが指定時刻に実行されない/メールが届かない** | Mail Relay Host が未設定、または Task Scheduler の開始日時/タイムゾーン設定ミス | `System > Tools > Scheduling`<br>`sudo tail -f /var/log/maillog` | `System > Configuration > Email` で Mail Relay IP を設定し、Task のタイムゾーンを確認。 |
| **レポートの入力パラメータ (\$<IP>) に値が入らず空で生成される** | Generate 時または Scheduled Task 作成時に Input Parameter の値を入力していない | レポート生成ポップアップ画面 | `Generate` ボタン押下時に表示される「Input Parameters」フィールドに目的の値を入力。 |
| **CSV レポートを出力したのにグラフや表紙が表示されない** | CSV フォーマットはテキスト要素のローデータ出力のみをサポートする仕様 | 該当なし (仕様) | アニメーションやグラフ、表紙が必要な場合は **PDF** または **HTML** フォーマットを選択する。 |
| **リモートストレージへのレポート移動が失敗する (`Storage Error`)** | NFS/SMB サーバーのアクセス権限不足、またはネットワークポート遮断 | `System > Configuration > Remote Storage Management` | リモートストレージの書込権限を確認し、`Test Connection` を再実行。 |

---

## ⚠ 制限事項

* **PDF 出力における件数制限**:
  * PDF レポートのレンダリング処理（メモリ使用量および生成処理時間）を保護するため、各セクションのテーブル表示件数には厳格な上限値（例: セクションあたり最大 1,000〜10,000 行程度）が設定されています。
* **マルチドメイン（Multi-Domain）環境での制約**:
  * 祖先ドメイン（上位ドメイン）で作成された Report Template は下位ドメインで直接編集・生成できません。下位ドメインで利用するには **コピー (Copy)** を作成して自ドメインに配置する必要があります。
  * スケジュールレポートは現在アクティブなドメインのコンテキストでのみ設定可能です。
* **CSV 形式におけるレイアウト要素の非サポート**:
  * CSV フォーマットでは、表紙（Cover Page）、目次（Table of Contents）、ヘッダー/フッター、グラフ（Pie/Bar/Line Chart）はすべて削除され、純粋なテキストテーブルデータのみが出力されます。

---

## 🔄 他技術との関連

* **1.4.a Alerting / Email Notification**: スケジュールレポートの自動配信には、FMC の Mail Relay Host (SMTP) 設定が必須の前提コンポーネントとなります。
* **1.4.b Logging / Database Event Limits**: レポートのソースデータは FMC のイベントデータベース（MySQL/Sybase）から抽出されるため、Database Event Limits でイベント保持期間が短く設定されている場合、過去のレポート生成時にデータが取得できません。
* **Task Scheduling (`System > Tools > Scheduling`)**: レポート生成の完全自動化（日次、週次、月次）を司るジョブ実行エンジン。
* **Cisco Security Analytics and Logging (SAL) / Remote Data Storage**: FMC ローカル DB ではなく外部の Secure Network Analytics / SAL アプライアンスに接続ログが保存されている場合、レポート生成時に SAL リモートボリュームをデータソースとして指定可能。

---

## 🧩 比較表

### 1. レポート出力フォーマット (PDF vs HTML vs CSV) の比較

| 比較項目 | PDF | HTML | CSV |
| :--- | :--- | :--- | :--- |
| **見た目・レイアウト** | **高度 (表紙・目次・ヘッダー完全対応)** | 標準 (ブラウザ表示最適化) | **テキストのみ (装飾なし)** |
| **グラフ/チャート表示** | **完全対応** (カラーチャート出力) | **完全対応** (動的グラフ表示) | **非対応** (データテーブルのみ) |
| **用途** | **経営層向け報告、コンプライアンス監査提出** | Web 上での簡易閲覧、社内共有 | **Excel 等での二次加工、データ解析** |
| **データ取得量制限** | **あり** (PDF メモリ保護のため件数制限) | 比較的ゆるい | 制限なし (テーブル最大 100 行/CSV) |

### 2. Standard Reports vs Risk Reports

| 比較項目 | Standard Reports | Risk Reports |
| :--- | :--- | :--- |
| **テンプレートカスタマイズ** | **自由自在** (Report Designer, クエリ編集) | 固定テンプレート (カスタム不可) |
| **出力形式** | PDF, HTML, CSV | **HTML 形式メイン** |
| **主対象データ** | 接続ログ、IPS イベント、ファイル、ユーザー、ヘルス | ネットワーク全体の**総合リスク・攻撃影響度 (Impact 1)** |
| **用途** | SOC 詳細分析、セキュリティ運用レポート | **CISO / CIO 向けエグゼクティブサマリー** |

---

## 💡 ベストプラクティス

1. **Report Designer による迅速なテンプレート構築**:
   * イベント調査中に有益なデータ表示が得られたら、画面右上の `Report Designer` ボタンを使用して即座にレポートテンプレート化して保存します。手動でゼロからクエリを構成する手間の 90% を削減できます。
2. **Scheduled Report の分散配置（夜間/早朝実行）**:
   * 週次・月次の大容量 PDF レポート生成は CPU および DB 読み出し負荷が高いため、業務時間帯（9:00〜18:00）を避け、**夜間または早朝（例: 毎週月曜 03:00 AM）に Scheduling 設定** します。
3. **リモートストレージ (NFS/SMB) へのアーカイブ自動移動**:
   * FMC のローカルディスク容量（`/var` パーティション）枯渇を防ぐため、生成された過去レポートは定期的に外部 NFS/SMB ストレージへ移動（Move）させる運用を自動化・標準化します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Report Designer によるブロックイベント画面のテンプレート化

#### 問題
FMC の Unified Event Viewer (接続イベント画面) において、アクションが `Block` または `Block with reset` となっている過去 24 時間のトラフィックを表示させた後、この表示内容をベースにしたレポートテンプレート `Blocked_Traffic_Template` を構築・保存せよ。

#### 要件
* `Report Designer` 機能を使用すること。

#### 設定・操作手順

```text
1. Analysis > Connections > Events に移動。
2. "Edit Search" をクリックし、Action = "Block", "Block with reset" を指定して Apply。
3. 画面右上の "Report Designer" ボタンをクリック。
4. 開いた Report Template 編集画面で:
   - Report Title: "Blocked_Traffic_Template"
   - Section Title: "Summary of Blocked Connections"
5. "Save" をクリック。
```

#### 検証方法
* `Overview > Reporting > Report Templates` 一覧に `Blocked_Traffic_Template` が正常に登録されていることを確認。

---

### 例2: Input Parameter (\$<Destination_IP>) を組み込んだ動的ホスト分析レポート

#### 問題
特定の宛先 IP アドレスを生成時に動的指定できるレポートテンプレート `Host_Specific_Threat_Report` を作成せよ。特定ホスト宛ての Intrusion Events テーブルおよび Connection Events 円グラフを含めよ。

#### 要件
* Input Parameter `$ Destination IP` を使用すること。

#### 設定・操作手順

```text
1. Overview > Reporting > Report Templates を開き "Create Report Template" をクリック。
2. Report Title: "Host Threat Report for $<Destination_IP>"
3. Section 1 (Intrusion Table):
   - Table: Intrusion Events / Format: Table View
   - Search: "Destination IP" ＝ "$<Destination_IP>" を設定。
4. Section 2 (Connection Pie Chart):
   - Table: Connection Events / Format: Pie Chart (X: Application Name, Y: Total Bytes)
   - Search: "Responder IP" ＝ "$<Destination_IP>" を設定。
5. "Save" をクリック。
```

#### 検証方法
* テンプレートの `Generate` をクリックし、表示された Input Parameter フィールドに `10.1.1.100` を入力してレポート生成。`10.1.1.100` 専用のレポートが生成されることを確認。

---

### 例3: Mail Relay Host の構成とレポートの 即時 Email 送信

#### 問題
FMC 上で SMTP サーバー `192.168.10.25:25` (From: `fmc@inelab.local`) を設定し、既存のレポートテンプレート `Executive_Summary` から PDF レポートを即時生成して `ciso@inelab.local` 宛てに Email 添付送信せよ。

#### 要件
* `System > Configuration > Email Notification` および `Generate` 画面を使用すること。

#### 設定・操作手順

```text
1. System > Configuration > Email Notification を開く。
   - Mail Relay Host: 192.168.10.25
   - Port: 25
   - From Address: fmc@inelab.local
   - Save をクリック。
2. Overview > Reporting > Report Templates を開く。
3. "Executive_Summary" テンプレート横の "Report" (Generate) アイコンをクリック。
4. Format: PDF を選択。
5. "Email" セクションを展開:
   - Email Options: Send Email
   - Recipient List: ciso@inelab.local
   - Subject: "Executive Security Summary Report"
6. "Generate" をクリック。
```

#### 検証方法
* FMC Bash で `sudo tail -f /var/log/maillog` を確認し、`192.168.10.25:25` 宛てにメールが正常送信されたことを確認。

---

### 例4: Task Scheduler による週次自動レポート生成 ＆ Email 配信

#### 問題
毎週月曜日の朝 07:00 AM に、レポートテンプレート `Weekly_Network_Digest` を使用して PDF レポートを自動生成し、`soc@inelab.local` 宛てに送信する自動タスク `Task_Weekly_Report` を構成せよ。

#### 要件
* `System > Tools > Scheduling` を使用すること。

#### 設定・操作手順

```text
1. System > Tools > Scheduling を開く。
2. "Add Task" をクリック:
   - Job Type: Report
   - Schedule task to run: Recurring
   - Start On: 該当の月曜日を選択
   - Repeat Every: 1 Weeks
   - Repeat On: Monday
   - Run At: 07:00 AM
   - Job Name: Task_Weekly_Report
   - Report Template: Weekly_Network_Digest
   - Format: PDF
   - Email Status To: soc@inelab.local
3. "Save" をクリック。
```

#### 検証方法
* `System > Tools > Scheduling` のタスク一覧で `Task_Weekly_Report` が `Enabled` で登録され、次回実行時刻が正しく計算されていることを確認。

---

### 例5: PDF 出力制限オーバーの警告解除 (Yellow Triangle Resolution)

#### 問題
レポートテンプレート `Detailed_IPS_Report` で PDF 出力を選択しようとしたが、ボタンが活性化しない。テンプレート内の各セクションの表示結果数（`Maximum Results`）を調整して PDF 生成を可能にせよ。

#### 要件
* 警告マーク (⚠️) のついたセクションを修正すること。

#### 設定・操作手順

```text
1. Overview > Reporting > Report Templates を開き "Detailed_IPS_Report" を編集。
2. セクション一覧で Results 数横に黄色三角アイコン (⚠️) が表示されているセクションを特定。
3. 黄色三角にマウスオーバーして制限件数 (例: Max 1000) を確認。
4. "Maximum Results" ドロップダウンで "Top 1000" (または制限以下の数値) を選択。
5. 全セクションの黄色三角が消去されたことを確認して "Save"。
```

#### 検証方法
* `Generate` をクリックし、PDF 出力形式ラジオボタンが選択可能になっていることを確認。

---

### 例6: Dashboard ウィジェットのインポートによる急速レポート作成

#### 問題
FMC の「Connection Summary Dashboard」に含まれる全ウィジェットのグラフを一括インポートして、新しいレポートテンプレート `Dashboard_Imported_Report` を構成せよ。

#### 要件
* "Import Section" 機能を使用すること。

#### 設定・操作手順

```text
1. Overview > Reporting > Report Templates を開き "Create Report Template" をクリック。
2. Report Title: "Dashboard_Imported_Report"
3. "Save" をクリック。
4. "Import Section" アイコンをクリック。
5. Data Source: Dashboard を選択。
6. Dashboard ドロップダウンから "Connection Summary Dashboard" を選択。
7. "Import" をクリック。
8. 自動追加された各ウィジェットセクションを確認し "Save"。
```

#### 検証方法
* `Dashboard_Imported_Report` からレポートを生成し、Dashboard と同じグラフ群（Connections Over Time 等）がレポート内に描画されていることを確認。

---

### 例7: 監査ログ (Audit Log) のレポート出力

#### 問題
FMC 上で管理者が行った過去 7 日間の設定変更およびログイン操作履歴（Audit Log）を抽出し、PDF レポート `FMC_Audit_Log_Report` として生成せよ。

#### 要件
* Table: Audit Records を使用すること。

#### 設定・操作手順

```text
1. Overview > Reporting > Report Templates で "Create Report Template" を選択。
2. Report Title: "FMC_Audit_Log_Report"
3. "Add Table View" を追加:
   - Table: Audit Records
   - Format: Table View
   - Time Window: Last 7 days
   - Fields: Time, User, Subsystem, Message
   - Maximum Results: Top 500
4. "Save" をクリックし "Generate" で PDF 出力。
```

#### 検証方法
* 生成された PDF ファイルを開き、FMC 管理者の操作履歴（`admin@192.168.10.50 deployed policy...` 等）が表形式で出力されていることを確認。

---

### 例8: Risk Report のオンデマンド生成と印刷用設定

#### 問題
FMC に標準搭載されているエグゼクティブ向け Risk Report (`Network Risk Report`) をオンデマンド生成し、HTML 形式で閲覧・出力せよ。

#### 要件
* `Overview > Reporting` 内の Risk Reports を使用すること。

#### 設定・操作手順

```text
1. Overview > Reporting > Report Templates タブを開く。
2. Standard テンプレートの下部に配置されている Risk Report セクションを表示。
3. "Network Risk Report" の "Generate Report" アイコンをクリック。
4. Title Page 用の Input Parameters (Customer Name 等) を入力。
5. "Generate" をクリック。
6. 表示された HTML 画面からブラウザの印刷設定 (Background Graphics 有効) で確認。
```

#### 検証方法
* ブラウザ上でグラフィカルなリスク評価サマリー（高リスクアプリ、感染ホスト一覧等）が表示されることを確認。

---

### 例9: リモート NFS ストレージへの生成済みレポート自動バックアップ

#### 問題
FMC 上で生成されたすべてのレポートを、ローカルディスクに保持せず外部 NFS サーバー (`192.168.10.200:/exports/fmc_reports`) へ自動的に移動・保存する構成を完了させよ。

#### 要件
* `Remote Storage Management` を使用すること。

#### 設定・操作手順

```text
1. System > Configuration > Remote Storage Management を開く。
2. Storage Type: NFS
3. Host: 192.168.10.200
4. Path: /exports/fmc_reports
5. "Test Connection" をクリックし成功を確認後、"Save"。
6. Overview > Reporting > Reports タブで対象レポートを選択し "Move to Remote Storage" を実行。
```

#### 検証方法
* NFS サーバー側の `/exports/fmc_reports/` ディレクトリ配下に、FMC から転送された `.pdf` ファイルが存在することを確認。

---

### 例10: マルウェア感染追跡 (File/Malware) 専用詳細レポートの構築

#### 問題
過去 30 日間にネットワーク内で検知されたマルウェア感染通信（AMP Malware Lookup で Malware 判定となったファイル転送）の送信元・宛先 IP、ファイル名、SHA-256 をまとめたレポート `Malware_Incident_Report` を作成せよ。

#### 要件
* Table: File/Malware Events を使用すること。

#### 設定・操作手順

```text
1. Overview > Reporting > Report Templates で "Create Report Template" を選択。
2. Report Title: "Malware Incident Report - Last 30 Days"
3. "Add Table View" を追加:
   - Table: File & Malware Events
   - Format: Table View
   - Time Window: Last 30 days
   - Search: Disposition ＝ "Malware"
   - Fields: Time, File Name, SHA256, Threat Score, Source IP, Destination IP, Action
   - Maximum Results: Top 1000
4. "Save" をクリックして PDF 生成。
```

#### 検証方法
* 生成された PDF 内に、マルウェアファイルのハッシュ値および感染疑惑端末の IP アドレスが整理されて出力されていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で特定のカスタムレポートテンプレートを作成し、PDF フォーマットでレポートを出力しようとしたが、`Generate` ポップアップ画面において「PDF」ラジオボタンが灰色に無効化（グレーアウト）されていた。テンプレート編集画面を確認したところ、あるセクションの横に黄色三角アイコン (⚠️) が表示されていた。このエラーの原因と適切な対処方法はどれか。

* A) FMC のライセンスが Base にダウングレードしているため。Smart Licensing を再登録する。
* B) 該当セクションの `Maximum Results` (取得結果件数) が PDF レンダリングの制限数を超えているため。黄色三角アイコンにマウスオーバーして上限値を確認し、件数を上限以下に削減して保存する。
* C) FMC の Mail Relay Host が設定されていないため。
* D) FTD 上で `ip nbar protocol-discovery` が停止しているため。

**正解**: **B**
**解説**: FMC のレポーティングエンジンでは、PDF 生成時のメモリ枯渇を防ぐため、セクションごとに取得可能な最大件数上限が定められています。上限を超過すると黄色三角アイコン (⚠️) が表示され、PDF 生成機能が制限されます。結果件数を上限値以下（例: Top 1,000 件等）に減らすことで解除されます。

---

### 2. トラブルシュート問題

**問題**: 管理者は Task Scheduler (`System > Tools > Scheduling`) を使用して、毎週月曜日にセキュリティレポートを自動生成し `soc@inelab.local` 宛てに Email 送信するジョブを設定した。指定時刻にタスクは正常終了（Completed）と表示されたが、管理者宛てにメールが一切届かなかった。考えられる主な原因はどれか。

* A) FMC の `System > Configuration > Email Notification` で Mail Relay Host (SMTP サーバー IP) が未設定、または SMTP ポート 25 の通信が途中でブロックされているため。
* B) レポート形式として CSV を選択したため。
* C) FTD アプライアンス上で `sftunnel` が切断されていたため。
* D) FMC の SSH ポートが閉じられているため。

**正解**: **A**
**解説**: スケジュールレポートの Email 自動配信を行うには、あらかじめ **`System > Configuration > Email Notification` で有効な Mail Relay Host (SMTP サーバー IP) と From アドレスを設定しておく必要があります**。未設定の場合、レポート生成自体は成功しますがメール送信段階で失敗します。

---

### 3. Design 問題

**問題**: セキュリティ運用チームから、「特定のクライアント IP アドレスを入出力するだけで、その端末が発生させた過去 24 時間の侵入アラートと Web 閲覧履歴を一括でまとめた PDF レポートを即座に発行できるようにしたい」という要望を受けた。FMC 上で最も運用効率の高いレポート設計手法はどれか。

* A) 毎日手動で Unified Event Viewer から CSV ファイルをエクスポートする。
* B) テンプレート内に **Input Parameter** (`$<Client_IP>`) を定義した単一の Report Template を構築し、検索条件に `$<Client_IP>` をバインドしておき、生成時に対象 IP を動的入力して出力させる。
* C) クライアント IP ごとに個別の Report Template を数百個作成しておく。
* D) FMC の Disk Manager を手動停止する。

**正解**: **B**
**解説**: レポートテンプレート内に **Input Parameters**（例: `$<Client_IP>`）を定義しておくと、生成時（Generate ボタン押下時）に任意の IP アドレスをパラメータとして動的に割り当てることができます。これにより、1 つのテンプレートだけで全端末の個別の調査レポートを効率的に動的生成できます。

---

### 4. 実装問題

**問題**: イベントビューア（`Analysis > Connections > Events`）で特定の複雑な検索条件（特定国からの非標準ポートによる通信拒否）を絞り込んだ状態から、最も手数が少なく正確にレポートテンプレート化するための FMC Web UI 上の操作手順はどれか。

* A) 空白の Report Template を新規作成し、手動で同じ SQL クエリを打ち込む。
* B) イベント表示画面の右上にある **`Report Designer`** ボタンをクリックし、自動構築されたテンプレートに必要な修正を加えて保存する。
* C) FTD CLI で `show log` コマンドを実行してテキストコピーする。
* D) `System > Tools > Scheduling` から直接 Task を登録する。

**正解**: **B**
**解説**: イベント表示画面（Event View）や Context Explorer 画面の右上にある **`Report Designer`** ボタンをクリックすると、現在画面上に適用されている検索フィルターおよびカラムレイアウトをそのまま引き継いだ Report Template が自動的に構築されます。

---

### 5. コンフィグ読解問題

**問題**: レポートテンプレートの生成設定画面において、出力フォーマットとして **CSV** を選択してレポートを生成した。出力されたファイルに関する説明として正しいものはどれか。

* A) PDF と同様に表紙、目次、カラーの円グラフが含まれたファイルが出力される。
* B) 各セクションのグラフィック要素やドキュメント装飾は除外され、検索結果のデータテーブルのみがカンマ区切りのテキストファイル (CSV) として出力される。
* C) FMC の管理パスワードが暗号化されてファイル末尾に追加される。
* D) 生成された CSV ファイルは自動的に FTD データプレーンへ Deploy される。

**正解**: **B**
**解説**: **CSV** フォーマットは純粋なテキスト形式の表データ出力であり、表紙、目次、ヘッダー/フッター、グラフィックチャート（円グラフ・棒グラフ）等のレイアウト装飾要素はすべて除外されます。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Administration Guide 7.1 - Introduction to Reports](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Administration Guide 7.1 - About Designing Reports](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Administration Guide 7.1 - Automating Report Generation with Task Scheduler](https://www.cisco.com/)
* [Cisco Technical Notes: Troubleshoot Email Delivery and Scheduled Reports on FMC](https://www.cisco.com/)
* [Cisco Live: Enterprise Security Visibility and Reporting with Cisco Secure Firewall (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で Reporting 課題が出題された場合、**「レポート作成 (Report Designer or Template) ➔ メール設定 (Email Notification) ➔ タスク自動化 (Task Scheduling)」** の一連の依存関係を意識し、漏れなく順序立てて構成することが完答への鍵となります。
* **図解（Report Generation ＆ Distribution フロー概要）**:
  ```text
  [ Trigger Event ]
         │
         ├── On-Demand: Overview > Reporting > Generate Click
         └── Scheduled: System > Tools > Scheduling (Cron Execution)
                 │
                 ▼
  [ Fetch Report Template ]
         │
         ├── Evaluate Input Parameters ($<IP>, $<Time_Window>)
         │
         ├── Query Event Database (Connection, Intrusion, File, Audit)
         │
         ▼
  [ Render Format ]
         ├── PDF  ──► Cover Page + Table of Contents + Charts + Tables (Result Capped)
         ├── HTML ──► Interactive Web Display
         └── CSV  ──► Plain Text Tables Only
                 │
                 ▼
  [ Distribution Output ]
         ├── Save to Local Reports List (Overview > Reporting > Reports)
         ├── Send Email via Mail Relay Host (SMTP Port 25)
         └── Move to Remote Storage (NFS / SMB / SSH Repo)
  ```
* **注意点**:
  * スケジュールタスクで作成したレポートに Email 添付送信を設定する場合、FMC の `System > Configuration > Email Notification` で **Mail Relay Host が未設定だとタスクが正常終了してもメールは一切飛搬されません**。必ず事前に SMTP 設定の疎通を確認してください。
