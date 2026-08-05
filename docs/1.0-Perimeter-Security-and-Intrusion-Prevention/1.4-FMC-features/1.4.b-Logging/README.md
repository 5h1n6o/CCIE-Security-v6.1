---
layout: default
title: 1.4.b-Logging
nav_order: 2
parent: 1.4-FMC-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.4.b Logging

Cisco Secure Firewall Management Center (FMC) における**ロギング（Logging）**は、ネットワーク上で発生するあらゆるイベント（接続、侵入、マルウェア、システムステータスなど）を記録・管理する中核機能です。CCIE Security v6.1ラボ試験では、適切なログ生成タイミングの選択、外部SIEMへの転送設定、および大量のログデータによるパフォーマンス低下を防ぐ設計能力が厳しく問われます。

---

## 📘 概要

*   **機能概要**: 管理下のデバイス（FTD）で発生したイベントを収集し、FMCのデータベースに保存、または外部サーバ（Syslog/SIEM）へ転送します。
*   **利用目的**: セキュリティ侵害の事後分析、コンプライアンスの遵守、トラフィック傾向の把握、トラブルシューティング。
*   **利用場面**: 不審な通信の特定、攻撃シグネチャのマッチング確認、ネットワーク利用率の統計作成。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主なログの種類** | 接続（Connection）、侵入（Intrusion）、マルウェア（Malware）、SSL、システムヘルス |
| **転送プロトコル** | Syslog, SNMP Trap, eStreamer (Binary), Webhook (HTTP) |
| **保存場所** | FMC内部データベース または 外部ロギングサーバ |
| **制御単位** | アクセスコントロールポリシー（ACP）の各ルール単位で設定 |
| **分析ツール** | FMC Event Viewer, Dashboard, Analysis Reports |
| **設計上の注意** | 全ての通信を「開始時」にログ出力すると、FMC/FTDの負荷が激増する |

---

## 🏗 動作原理

ロギングは、データパスを処理するFTDと、それを集約するFMCの連携によって動作します。

```text
[ Traffic Flow ]
      ↓
[ Managed Device (FTD) ]
      ├─ Security Engine (Snort) がパケットを検査
      ├─ 設定されたポリシー（ACP）に基づきログを生成
      └─ SFTunnel を介して FMC へイベントを送信（または外部へ直接）
              ↓
[ Cisco FMC ]
      ├─ イベントの受信と正規化
      ├─ 内部データベース（PostgreSQL）への格納
      └─ 外部レスポンスのトリガー（Alerting 連携）
              ↓
[ External SIEM / Syslog ]
```

---

## ⚙ 動作シーケンス

1.  **ルール合致**: パケットがアクセスコントロールルールの条件にマッチします。
2.  **ログ生成の判定**: ルールの「Logging」タブで設定されたオプション（開始時/終了時）を確認します。
3.  **イベント作成**: FTDがメモリ上に接続レコードを作成し、定義された属性（IP, Port, App, User等）を記録します。
4.  **データ転送**: 
    *   **FMCロギング**: デフォルトでは管理トンネル経由でFMCに送られます。
    *   **外部転送**: FTDから直接Syslogサーバへ送出するように設定することも可能です（FMCの負荷軽減）。
5.  **ビュー表示**: 管理者がFMCの「Analysis > Connections」画面でイベントを確認します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **開始時ログ vs 終了時ログ**: ラボでは「トラフィックの総量（バイト数）を記録せよ」という要件が出ます。この場合、**Log at End of Connection** を選択する必要があります（開始時にはバイト数が確定していないため）。
*   **Default Action のロギング**: 全てのルールにマッチしなかったパケットをログ出力するために、ACPの「Default Action」設定内にあるロギングを有効化することを忘れないでください。
*   **外部Syslog設定**: 「FTDから直接外部へSyslogを送れ」という要件に対し、ACPルールのロギング設定で **Syslog Config** を指定する手順が重要です。
*   **eStreamerの構成**: 高度なSIEM連携として、FMCでの証明書発行とクライアントIPの登録が問われる可能性があります。

---

## 🛠 設定方法

### 1. アクセスコントロールルールでのロギング有効化
1.  **Policies > Access Control** を編集。
2.  対象ルールの **Logging** タブをクリック。
3.  **Log at End of Connection** をチェック。
4.  **Log to Management Center** を選択（必要に応じて Syslog Server オブジェクトも追加）。

### 2. ファイル/マルウェアイベントの記録
1.  ACPルールの **Inspection** タブで **File Policy** を紐付け。
2.  **Logging** タブで **Log Files** をチェックし、マルウェアの試行を可視化する。

---

## 🔍 検証コマンド

| 目的 | コマンド（FTD CLI） |
| :--- | :--- |
| **ログ出力統計の確認** | <code>show statistics system flow-export</code> |
| **Snortによる判定ログ確認** | <code>system support firewall-engine-debug</code> |
| **外部Syslog設定の確認** | <code>show running-config logging</code> (LINA) |
| **イベント転送キューの確認** | <code>show perfmon</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 接続ログがFMCに表示されない | Logging設定が無効 | ACPルールのLoggingタブでチェックが入っているか確認。 |
| ログのバイト数が「0」 | 開始時ログのみ有効 | **Log at End of Connection** が選択されているか確認。 |
| FMCのディスクが枯渇 | ログ保持期間が長すぎる | **System > Configuration > Database** で最大保存イベント数を調整。 |
| 外部Syslogが届かない | ネットワーク疎通・ポート | FTD/FMCからサーバへの <code>ping</code> とUDP 514の許可を確認。 |

---

## ⚠ 制限事項

*   **パフォーマンスへの影響**: 全てのパケットをログ出力するように設定すると、FTDのデータプレーン性能が著しく低下します。
*   **データベース容量**: FMCモデルごとに保存可能なイベント数に上限があります。
*   **暗号化**: FTDから直接外部へ送るSyslogは、デフォルトではプレーンテキストです（セキュアSyslogには別途設定が必要）。

---

## 🔄 他技術との関連

*   **Access Control**: ログ生成のトリガーとなる主要なポリシー。
*   **Intrusion Policy**: 攻撃検知時の詳細なパケットログ（Packet Captures）を生成可能。
*   **Analysis & Reporting**: 収集したログを基に、日次や週次のエグゼクティブレポートを自動生成します。
*   **Health Monitoring**: デバイス自体の異常（ファン故障、CPU高負荷）をログとして記録。

---

## 🧩 比較表

### Log at Beginning vs Log at End

| 特徴 | Log at Beginning (開始時) | Log at End (終了時) |
| :--- | :--- | :--- |
| **タイミング** | 通信確立直後 | 通信切断（またはタイムアウト）直後 |
| **記録内容** | 基本属性（IP, Port, App） | **全属性 ＋ バイト数 ＋ 通信時間** |
| **推奨用途** | リアルタイム監視、拒否ルール | **全ての許可ルール（推奨）** |
| **負荷** | 高い（短期通信が多い場合） | 比較的低い |

---

## 💡 ベストプラクティス

1.  **原則「終了時」にログ**: 通常の許可トラフィックは「Log at End」で記録し、詳細な統計を確保します。
2.  **拒否ルールは「開始時」**: 攻撃をリアルタイムに検知するため、Denyルールは「Log at Beginning」で即座に出力します。
3.  **重要なルールのみに絞る**: 不要なDNSやICMPなどの頻発するトラフィックはロギングを無効化し、ストレージとパフォーマンスを節約します。
4.  **外部SIEMの活用**: FMCの負荷軽減のため、長期保存用のログは外部Syslogサーバへオフロードします。

---

## 📝 ラボ学習・設定サンプル例

※以下はFMC GUIのロジックに基づいた設定シナリオです。

### 1. 接続ログの基本設定（FMC）
*   **要件**: 社内からインターネットへの全Webトラフィックのバイト数を記録せよ。
*   **設定**: ACPルール > Logging > **Log at End of Connection** & **Log to Management Center** を選択。

### 2. 拒否された通信の監視
*   **要件**: ポリシーに合致せずドロップされたパケットをログに残せ。
*   **設定**: ACP > **Default Action** セクションのロギングボタン（封筒アイコン）を押し、**Log at Beginning** を有効化。

### 3. FTDから直接Syslog転送
*   **要件**: FMCの負荷を避けるため、FTDからSyslogサーバ `10.1.1.100` へ直接ログを送れ。
*   **設定**: ACPルールのロギング設定で、**Syslog Server** オブジェクトを作成して追加。

### 4. アプリケーション識別情報のロギング
*   **要件**: ログに「Facebook」や「BitTorrent」といったアプリ名を含めよ。
*   **設定**: ACPルールで **Application** を指定し、ロギングを有効にするだけで自動的に付与される。

### 5. マルウェアイベントの記録
*   **要件**: ファイル検知時のマルウェア判定イベントを記録せよ。
*   **設定**: ACPルール > Logging > **Log Files** をチェック。

### 6. eStreamer証明書の発行
*   **要件**: SIEMサーバからのデータ取得を許可せよ。
*   **手順**: System > Integration > **eStreamer** > Add Client でIP登録し、証明書をダウンロード。

### 7. 侵入イベントの詳細パケット保存
*   **要件**: IPS検知時にパケットキャプチャを最大10パケット分保存せよ。
*   **設定**: Intrusion Policy > ルール編集 > **External Responses** または **Logging** で Packet Log を有効化。

### 8. システムヘルスのアラートログ
*   **要件**: CPU使用率が90%を超えた際、イベントログに残せ。
*   **設定**: System > Health > Monitor で閾値を設定し、Health Policy に Alert を紐付け。

### 9. 大量ログのフィルタリング
*   **要件**: 重大な脅威（Critical）のみをFMCに、他はSyslogへ。
*   **設定**: アラートオブジェクトとACPロギングを組み合わせて Severity ベースで振り分ける。

### 10. 接続イベントの検索（検証）
*   **要件**: 過去1時間の `192.168.1.50` からの通信を抽出せよ。
*   **手順**: Analysis > Connections > Events > **Search** で送信元IPを指定。

---

## ❓ 想定試験問題

1.  **Design**: 大規模な拠点に展開したFTDで、接続ログが多すぎてFMCのGUIが重くなっている。パフォーマンスを改善する最適なロギング構成を述べよ。
    *   **正解**: FTDから外部Syslogサーバへ直接イベントを送信するようACPで設定し、FMCへの送信（Log to Management Center）をオフにする。
2.  **Troubleshoot**: 特定のアクセスコントロールルールのログで、送信バイト数が常に 0 と表示される。原因と修正方法は？
    *   **正解**: ロギングタイミングが「Log at Beginning」になっている。ルールの編集画面で「Log at End of Connection」に変更する。
3.  **Implementation**: 外部SIEM(Splunk等)が、FMCからバイナリ形式で大量のイベントをプル型で取得するためのプロトコルは何か？
    *   **正解**: eStreamer。
4.  **Config Reading**: ACPのデフォルトアクションでロギングが無効な場合、どのような問題が発生するか？
    *   **正解**: どのルールにも合致せず、ファイアウォールによって自動的にドロップされた不明なトラフィックが可視化されない。
5.  **Implementation**: マルウェア分析機能を備えたFile Policyをルールに適用している。ファイルが転送された際のイベントを確実に出力するためにチェックが必要な項目は？
    *   **正解**: ACPルールのLoggingタブにある「Log Files」チェックボックス。

---

## 🔗 参考リソース

*   **Cisco Secure Firewall Management Center Administration Guide, 7.1**
    *   [Event Logging and Analysis](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/connection_logging.html)
*   **Cisco Secure Firewall Threat Defense Configuration Guide for FMC, 7.0**
    *   [Logging and System Monitoring](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/logging.html)
*   **Cisco Live 資料 (BRKSEC-3020)**
    *   [Troubleshooting Firewall Threat Defense (FTD)](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-3020)
*   **Technical Notes**
    *   [Firepower Event Streamer (eStreamer) Guide](https://www.cisco.com/c/en/us/td/docs/security/firepower/60/api/estreamer/EventStreamerGuide.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: FMCのロギングは「情報の宝庫」ですが、ラボ試験では「何を見るべきか」以上に「設定をミスしてシステムを止めていないか」が重要です。特に、全てのログをManagement Centerに送る設定は、試験用リソースを圧迫するリスクがあるため、要件に従って慎重に選択してください。
*   **図解**: パケット処理（Snort/LINA）→ ログ生成（FTD）→ 転送（SFTunnel/Syslog）という物理的な流れを常に意識しましょう。
*   **注意点**: CCIEラボでは、ログが表示されるまで数分のラグがある場合があります。設定直後に表示されなくても慌てず、<code>show statistics system flow-export</code> 等でFTDが送出しているかを確認してください。

---

### FMCのロギングにおける「開始時」と「終了時」の使い分けについて

FMC（Firepower Management Center）の接続ロギングにおける**「開始時（Beginning）」**と**「終了時（End）」**は、取得できる情報の詳細度とシステムのパフォーマンスへの影響が大きく異なります。

主な違いと使い分けのガイドラインは以下の通りです。

#### 1. 「開始時」と「終了時」の比較

| 特徴 | 接続の開始時（Beginning） | 接続の終了時（End） |
| :--- | :--- | :--- |
| **生成タイミング** | システムが接続の開始を検出したとき（最初の数パケットでアプリケーションやURLを識別した後） | 接続の終了を検出したとき、またはタイムアウト、メモリ制限で追跡不能になったとき |
| **記録される情報** | 最初のパケットから判断できる情報のみ（IP, ポート, アプリ識別など） | **開始時の全情報** ＋ 通信時間（Duration）、総転送バイト数、最後のパケットのタイムスタンプ |
| **主な用途** | 拒否（Block）された通信の記録 | 許可（Allow）された通信の詳細分析、統計、グラフ作成、ベースライン（Traffic Profile）の作成 |

#### 2. 使い分けのガイドラインとベストプラクティス

*   **原則として「終了時」のみを記録する**: 許可された通信については、終了時のログに開始時の情報がすべて含まれているため、終了時のみを有効にすることが推奨されます。
*   **両方を同時に有効にしない**: パフォーマンスへの悪影響やログの冗長化、FMCへの過負荷（イベントのバックログ発生）を避けるため、1つのルールで「開始時」と「終了時」の両方をチェックすることは避けてください。
*   **拒否（Block）ルールは「開始時」**: 通信が即座に遮断されるため「終了」という概念がなく、通常は開始時イベントのみを記録できます。
*   **モニター（Monitor）ルールは「終了時」が必須**: 通信を継続させながら統計を取るためのルールの場合は、終了時のロギングが必要です。

#### 3. 特殊なケースと自動ロギング

*   **侵入・マルウェアイベント**: 侵入（Intrusion）やマルウェア（Malware）を検知した場合、ルールの設定に関わらず、システムはその接続の**終了時イベントを自動的に記録**します。
*   **暗号化通信のブロック**: SSLポリシーでブロックされた暗号化トラフィックの場合、最初のパケットだけでは判断できず数パケットを要するため、「開始時」ではなく「終了時」イベントとして記録されます。
*   **NetFlowデータ**: eStreamerなどを介してFMCに取り込まれるNetFlowレコードは、常に「終了時」イベントとして処理されます。

#### 4. 分析における重要性
トラフィックの傾向分析、レポート作成、相関ルール（Correlation Rules）の適用、またはダッシュボードでの可視化を行いたい場合は、必ず**終了時のロギング**を有効にする必要があります。開始時のみでは、通信量（バイト数）などの重要な統計データが収集できないためです。

---

### Cisco Secure Firewall Management Center (FMC) から外部syslogサーバーへログを転送する設定

Cisco Secure Firewall Management Center (FMC) から外部syslogサーバーへログを転送する設定は、主に**「監査ログ（Audit Log）」**の転送と、セキュリティイベントなどの通知を行う**「アラートレスポンス（Alert Response）」**の2つの仕組みがあります。

設定の詳細は以下の通りです。

#### 1. 監査ログ（Audit Log）の転送設定
FMC上でのユーザー操作や設定変更の履歴を転送する設定です。これによりFMCのディスク容量を節約できます。

*   **設定場所:** `System` > `Configuration` > `Audit Log`
*   **手順:**
    1.  **Send Audit Log to Syslog** を「Enabled」に設定します。
    2.  **Host** にsyslogサーバーのIPアドレスを入力します。
    3.  **Facility** および **Severity** を選択します。これらは受信側での分類に使用されます。
    4.  **Tag**（任意）を設定します。例として「FMC-AUDIT-LOG」などと入力すると、他のログと識別しやすくなります。
    5.  (任意) **Test Syslog Server** をクリックして疎通確認を行います。FMCはICMPやTCP SYNパケットを使用して到達性を確認します。
    6.  **Save** をクリックして保存します。

#### 2. アラートレスポンス（Alert Response）によるイベント転送
FMCが相関分析（Correlation）したイベント、インパクトフラグの変更、ディスカバリイベント、マルウェア検知などを転送するための設定です。

##### ステップA：syslogアラートオブジェクトの作成
*   **設定場所:** `Policies` > `Actions` > `Alerts`
*   **手順:**
    1.  「Create Alert」から **Create Syslog Alert** を選択します。
    2.  **Name**、**Host**（IPアドレス）、**Port**（デフォルトは514）を入力します。
    3.  **Facility**、**Severity**、**Tag** を指定します。
    4.  **Save** をクリックします。この設定は即座に有効になりますが、接続ログに使用する場合はデプロイが必要です。

##### ステップB：各ポリシーへの紐付け
作成したオブジェクトを、通知を受けたいイベントの種類に応じて紐付けます。

*   **インパクトフラグ/ディスカバリイベント:** `Policies` > `Correlation` > `Alerts` タブなどで、作成したsyslogアラートを選択し有効化します。
*   **接続ログ（Connection Logs）:** アクセスコントロールポリシーの各ルール、またはデフォルトアクションの `Logging` タブで「Syslog Server」として作成したオブジェクトを指定します。

#### 3. 設定上の注意点とベストプラクティス
*   **ネットワーク疎通:** FMCの管理インターフェイスからsyslogサーバーへ、UDP 514（または設定したポート）で通信できる必要があります。
*   **デバイスとの使い分け:** FTDデバイス自体からも直接外部syslogへ転送可能（Platform Settingsを使用）です。FMC経由の転送は、FMC側で相関処理された高度なイベントの通知に適していますが、大量の接続ログをすべてFMC経由で送るとFMCの負荷が高まるため注意が必要です。
*   **メッセージ形式:** FMCから送信されるメッセージには、設定した「Tag」やFMCのホスト名がヘッダーに含まれます。
*   **信頼性:** syslogサーバーにTCPを使用する場合、サーバーがダウンしてもトラフィックを通過させるオプション（Allow User Traffic to Pass When TCP Syslog Server Is Down）を検討してください。

