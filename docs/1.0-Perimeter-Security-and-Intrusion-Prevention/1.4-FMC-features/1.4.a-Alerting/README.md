---
layout: default
title: 1.4.a-Alerting
nav_order: 1
parent: 1.4-FMC-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.4.a Alerting

Cisco Secure Firewall Management Center (FMC) における **Alerting（アラート通知機能）** は、管理下の Secure Firewall Threat Defense (FTD) や NGIPS アプライアンス、および FMC 自身が検知・生成したセキュリティイベント（侵入、マルウェア、接続、ネットワークディスカバリ）やシステム正常性イベント（ヘルスモニター、障害）を、リアルタイムに外部管理システム（Syslog サーバー、SNMP トラップレシーバー、メールサーバー/SMTP）へ動的通知するための基盤機能です。

CCIE Security v6.1 Blueprint 1.4.a では、単にアラートを設定するだけでなく、**FMC 主体のアラート（Alert Responses）と FTD アプライアンス直出しのアラート（Platform Settings / Policy Logging）のアーキテクチャ上の明確な違い**、相関ルール（Correlation Rules & Policies）や影響度フラグ（Impact Flag: 1〜4）と連携した高度な条件付きアラート発行、ヘルスアラート（Health Monitor Alerts）、ならびにトラブルシューティング・ログ分析の実践的スキルが評価されます。

---

## 📘 概要

* **機能概要**:
  * **Alert Responses (FMC 主体アラート)**: FMC 上で中央定義される外部通知プロファイル（Syslog、SNMPv1/v2c/v3 トラップ、Email）。相関ポリシーの違反、ディスカバリエベント、AMP マルウェアイベント、および Impact Flag 条件に連動して FMC から外部へ直接送出されます。
  * **Managed Device Direct Alerting (FTD 直出しアラート)**: FTD の Platform Settings や Access Control Policy (ACP) / Intrusion Policy の Advanced Settings から、FTD アプライアンス自身（Data Plane / LINA / Snort）が直接外部 Syslog / SNMP サーバーへリアルタイム出力するアラート。
  * **Impact Flag Alerting**: ネットワークディスカバリによって得られたターゲットホストの脆弱性情報と侵入イベントのシグネチャを自動相関させ、攻撃の影響度（Impact Level 1: Vulnerable 〜 4: Unknown）に応じて選択的にアラートを生成する機能。
  * **Correlation Policy Alerting**: 複数のイベント（接続、侵入、マルウェア、ユーザーアクティビティ、トラフィックプロファイル変化）を条件ロジック（AND/OR）で組み合わせ、条件合致時にアラート出力や自動リカバリ（Remediation: ISE ANC 隔離、Nmap スキャン等）を起動。
  * **Health Monitor Alerting**: FMC および FTD のハードウェア/ソフトウェアコンポーネント状態（CPU、メモリ、ディスク、Snort プロセス、HA 同期ステータス、インターフェイス Down 等）が警告/致命的閾値に達した際に通知。
* **利用目的**:
  * SIEM (Security Information and Event Management: Splunk、QRadar 等) や SOC (Security Operations Center) 監視端末へのリアルタイムセキュリティイベント統合。
  * 業務クリティカルな脆弱性ホストに対する攻撃（Impact Level 1）発生時の即時管理者メール通知。
  * システム異常（ディスク満杯、CPU 高負荷、HA 切断）の早期検知とダウンタイムの防止。
* **どのような場面で利用するか**:
  * 24 時間 365 日体制のセキュリティ監視基盤（SOC）へのイベントストリーミング。
  * セキュリティポリシー違反（例: 未承認の P2P 通信、大容量データ転送、マルウェア検知）発生時のセキュリティチームへの自動チケット発行・通知。
  * Cisco ISE と連携した Rapid Threat Containment (RTC / ANC) による感染端末の動的ネットワーク隔離。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **アラート方式** | Syslog、SNMP Trap (v1/v2c/v3)、Email (SMTP)、Remediation Execution、eStreamer API |
| **設定メニュー位置 (FMC)** | <code>Policies > Actions > Alerts</code> (Alert Responses / Impact / Discovery / AMP)<br><code>Policies > Correlation</code> (Correlation Rules/Policies)<br><code>System > Health > Monitor Alerts</code> (Health Alerts) |
| **送出元の違い** | **FMC 送出**: Correlation, Impact Flag, Discovery, AMP, Health, Email<br>**FTD 直出し**: ACP / Intrusion Policy 内の Syslog/SNMP (Platform Settings 経由) |
| **Impact Level** | Impact 1 (Vulnerable - 赤), Impact 2 (Potentially Vulnerable - 黄), Impact 3 (Currently Not Vulnerable - 青), Impact 4 (Unknown Target - グレー), Impact 0 (Unknown) |
| **SNMPv3 要件** | FTD 送出の SNMPv3 トラップ解析には Engine ID (管理 IP + "01") の登録が必須 |
| **制限事項** | Email アラートは Connection Event (接続ログ) に対しては利用不可。大量イベント発生時の FMC 処理負荷・帯域圧迫に注意 |

---

## 🏗 動作原理

FMC の Alerting アーキテクチャは、「イベントの検出・生成場所」と「アラートの相関評価・送出場所」によって 2 つのデータパスに分かれます。

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│                      Firepower Management Center (FMC)                          │
│                                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐   │
│   │ [ FMC Alert Engine ]                                                     │   │
│   │  ├── Correlation Engine (SFDataCorrelator) ──► Correlation Policy Alerts  │   │
│   │  ├── Impact Assessment Engine            ──► Impact Flag Alerts         │   │
│   │  ├── Network Discovery Engine            ──► Discovery Event Alerts     │   │
│   │  ├── AMP Cloud Analysis                  ──► AMP Malware Alerts         │   │
│   │  └── Health Monitor                      ──► Health Alerts              │   │
│   └────────────────────────────────────┬─────────────────────────────────────┘   │
│                                        │                                         │
│                        Alert Responses │ (Syslog / SNMP Trap / Email)            │
│                                        ▼                                         │
│                       ┌────────────────────────────────┐                         │
│                       │ External Monitoring Systems    │                         │
│                       │ (SIEM / Syslog / NMS / Mail)   │                         │
│                       └────────────────────────────────┘                         │
│                                        ▲                                         │
│                                        │ Direct Security Syslog / SNMP Traps     │
│   ─────────────────────────────────────┼───────────────────────────────────────  │
│                                        │ (Configured via Platform Settings)      │
│   [ Managed Device: FTD ]              │                                         │
│   ┌────────────────────────────────────┴─────────────────────────────────────┐   │
│   │ Data Plane (Snort 2/3 & LINA Engine)                                     │   │
│   │  ├── Intrusion Events (Syslog / SNMP)                                    │   │
│   │  └── Connection Events (Syslog / EMBLEM Format)                          │   │
│   └──────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

1. **イベント発生と FTD 側の一次処理**:
   * FTD 上の Snort インスペクションエンジンまたは LINA データプレーンにおいて、アクセス制御ルール、侵入防御シグネチャ、マルウェアポリシーにマッチするパケットを検出。
   * **FTD 直出し設定の場合**: FTD は Platform Settings で定義された Syslog/SNMP サーバー宛てに直接アラートを送出。
2. **`sftunnel` (TCP 8305) 経由での FMC へのイベント転送**:
   * FTD は検出したバイナライザーイベント（Intrusion, Connection, File, Discovery）を暗号化 `sftunnel` を通じて FMC の `SFDataCorrelator` プロセスへ送信。
3. **FMC での相関評価と Impact Assessment (影響度自動判定)**:
   * FMC は受信した侵入イベントと、`Network Discovery` データベースに保持されているターゲットホストの OS/サービス/脆弱性情報を自動照合。
   * ターゲットが実際にその攻撃に対して脆弱である場合、**Impact Level 1 (Vulnerable)** フラグを付与。
4. **Correlation Policy および Alert Response のトリガー**:
   * イベント条件が `Correlation Policy` のルールや `Impact Flag Alerting` 設定に合致した場合、バインドされた `Alert Response`（Syslog、SNMP Trap、Email）プロファイルが起動。
5. **外部通知システムへの送出**:
   * FMC から外部 SIEM、SNMP トラップレシーバー、または SMTP メールサーバーへアラートが送出される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **FMC Alert Response と FTD Platform Settings の使い分け**:
   * **問題の指示を正確に読み解くこと**:
     * 「Send alerts **from FMC** when Impact Level 1 events occur」➔ `Policies > Actions > Alerts` で Syslog/Email/SNMP Alert Response を作成し、`Impact Flag Alerts` タブで設定。
     * 「Configure **FTD devices to send syslog** for security events directly」➔ `Devices > Platform Settings > Threat Defense Settings > Syslog` で Syslog Server を追加し、ACP の `Logging` タブで「Use Threat Defense Platform Settings」を選択。
2. **Email Alert 設定時の 2 大前提要件**:
   * Email アラートを動作させるには、事前に以下 2 点の設定が不可欠です。
     1. **Mail Relay Host 設定**: `Policies > Actions > Alerts` (または `System > Configuration > Email Notifications`) で有効な SMTP サーバー IP/Port を定義。
     2. **FMC 自身の Reverse DNS 解決**: FMC が自身の IP アドレスを逆引き解決できる状態（またはホスト名設定）であること。
   * **注意**: Email アラートは Connection Events (接続ログ) に対してはサポートされていません。
3. **Correlation Policy の構成とアクティベーション**:
   * Correlation Rule（`Policies > Correlation > Rule Management`）を作成しただけでは機能しません。必ず **Correlation Policy（`Policies > Correlation > Policy Management`）にルールを追加し、レスポンス（Alert / Remediation）を割り当てた上で、ポリシーを「Active (ON)」にする** 必要があります。
4. **SNMPv3 Engine ID の計算手法 (FTD 直出し時)**:
   * Managed Device (FTD) から直接 SNMPv3 トラップを送信する場合、トラップレシーバー側（NMS）でデコードするための Engine ID が必要です。
   * **Engine ID の構造**: FTD 管理インターフェイス IP アドレスの 16 進数表記の末尾に `01` を付与。
     * 例: FTD 管理 IP `172.16.1.50` ➔ 16進数: `AC.10.01.32` ➔ Engine ID: `0xAC10013201`。
5. **Syslog フォーマット (Cisco EMBLEM vs RFC 5424)**:
   * FTD の Platform Settings で **`Send syslogs in EMBLEM format`** を有効にすると、送信プロトコルは **UDP 限定** となります（TCP は使用不可）。
   * EMBLEM フォーマットでは Syslog ヘッダーに Priority (PRI) 値が含まれますが、Device ID は出力されなくなります。

---

## 🛠 設定方法

### 1. FMC GUI: Alert Responses (Syslog / SNMP / Email) の統合作成プロファイル

#### A. Syslog Alert Response の作成
1. **`Policies > Actions > Alerts`** に移動。
2. `Create Alert` ドロップダウンから **`Create Syslog Alert`** を選択。
3. パラメータを入力:
   * **Name**: `Syslog_SIEM_Server`
   * **Host**: `192.168.10.100` (Syslog サーバー IP)
   * **Port**: `514`
   * **Facility**: `ALERT` (または `LOCAL4`)
   * **Severity**: `ALERT` (または `CRITICAL`)
   * **Tag**: `CiscoSecureFirewall`
4. **`Save`** をクリック。

#### B. SNMP Trap Alert Response の作成
1. **`Policies > Actions > Alerts`** を開く。
2. `Create Alert` ドロップダウンから **`Create SNMP Alert`** を選択。
3. パラメータを入力:
   * **Name**: `SNMP_NMS_Trap`
   * **Trap Server**: `192.168.10.200`
   * **Version**: `Version 3`
   * **Username**: `snmp_sec_user`
   * **Authentication Password / Private Password**: 該当パスワードを入力
   * **Engine ID**: （必要に応じて入力）
4. **`Save`** をクリック。

#### C. Email Alert Response および Mail Relay の作成
1. **`System > Configuration > Email Notifications`** (または `Policies > Actions > Alerts`) を開く。
2. **Mail Relay Host**: `192.168.10.25` (SMTP サーバー IP)、Port: `25` を設定。
3. **`Policies > Actions > Alerts`** で `Create Email Alert` を選択。
   * **Name**: `Email_Security_Team`
   * **To**: `soc-team@inelab.local`
   * **From**: `fmc-alert@inelab.local`
4. **`Save`** をクリック。

---

### 2. FMC GUI: Impact Flag Alerting (攻撃影響度に応じた通知) の構成

1. **`Policies > Actions > Alerts`** を開き、**`Impact Flag Alerts`** タブを選択。
2. 最上部のドロップダウンプロファイルを選択:
   * **Syslog**: `Syslog_SIEM_Server`
   * **Email**: `Email_Security_Team`
3. **Impact Flag Configuration** マトリクスで通知条件にチェックを入れる:
   * **Impact 1 (Vulnerable)**: `Syslog` [✓], `Email` [✓]
   * **Impact 2 (Potentially Vulnerable)**: `Syslog` [✓]
   * **Impact 4 (Unknown Target)**: `Syslog` [✓]
4. **`Save`** をクリック。

---

### 3. FMC GUI: Correlation Policy ＆ Rule による特定攻撃の自動アラート設定

#### Step 1: Correlation Rule の定義
1. **`Policies > Correlation > Rule Management`** に移動し、**`Create Rule`** をクリック。
2. **Rule Name**: `CR_Brute_Force_SSH`
3. **If**: `an intrusion event occurs`
4. **Conditions**: `Add condition` をクリックし、`Signature ID` `is` `1985` (SSH Login Failed 等) を指定。
5. **`Save`** をクリック。

#### Step 2: Correlation Policy の作成とアクティベート
1. **`Policies > Correlation > Policy Management`** に移動し、**`Create Policy`** をクリック。
2. **Policy Name**: `CP_High_Severity_Events`
3. **`Add Rules`** をクリックし、`CR_Brute_Force_SSH` を選択して追加。
4. ルール横の **Response アイコン (吹き出しマーク)** をクリック。
5. **Unassigned Responses** から `Syslog_SIEM_Server` および `Email_Security_Team` を **Assigned Responses** へ移動し、`Update` をクリック。
6. ポリシー保存後、Policy Management 一覧画面でトグルスイッチを **`On (Active)`** に変更。

---

### 4. FMC GUI: Health Monitor Alert (システム正常性アラート) の構成

1. **`System > Health > Monitor Alerts`** に移動。
2. **`Add`** をクリック。
3. パラメータ設定:
   * **Health Alert Name**: `HA_Critical_Hardware_Alert`
   * **Severity**: `Critical`
   * **Alert**: `Syslog_SIEM_Server`
4. 該当するモジュール（例: `CPU Usage`, `Disk Usage`, `Appliance Heartbeat`, `Snort Process`）の Critical 行に割り当てて保存。

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI / FMC Bash) | 表示確認ポイント |
| :--- | :--- | :--- |
| **FMC 内部システムログのリアルタイム追跡** | <code>sudo tail -f /var/log/messages</code> | FMC からの Syslog 生成、SMTP 接続、エラーログの確認 |
| **Data Correlator プロセスの動作状態確認** | <code>sudo perfstats -Cq</code> | Correlation エンジンのアクティブ状態、イベント処理レート |
| **FTD 側の Syslog サーバー設定および統計確認** | <code>show logging</code> | データプレーン Syslog 送信先 IP、ポート、送出メッセージ数 |
| **FTD 側の SNMP サーバーおよびトラップ設定確認** | <code>show snmp-server statistics</code> | 送信済み SNMP トラップ数、ドロップ数、レシーバー IP |
| **FMC 側プロセスの統合ステータス表示** | <code>sudo pmtool status \| grep -i correlator</code> | `SFDataCorrelator` および `SFRemediateD` が Running か |
| **FMC 内 MIB ファイルのパス確認** | <code>ls -l /etc/sf/DCEALERT.MIB</code> | FMC Alert 用 MIB ファイルの存在確認 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **Correlation Rule を作成したのにアラートが飛ばない** | Rule が Correlation Policy に追加されていない、または Policy が `On (Active)` になっていない | <code>Policies > Correlation > Policy Management</code> | Correlation Policy に Rule を紐付け、Response を割り当てて Policy を **On** にする。 |
| **FMC から Email アラートが送信されない** | Mail Relay Host が未設定、または FMC が自身の IP を逆引き DNS 解決できない | <code>sudo tail -f /var/log/messages</code><br><code>System > Configuration > Email</code> | Mail Relay IP を正しく設定し、DNS サーバーで FMC の A/PTR レコードを登録する。 |
| **Impact 1 アラートが発生しない** | Network Discovery Policy が無効化されているためホストの脆弱性が特定できていない | <code>Policies > Network Discovery</code> | Network Discovery Policy を有効化し、対象サブネットのホスト/OS/サービス発見を実行させる。 |
| **FTD から Syslog が届かない** | Platform Settings の Deploy 忘れ、または出力インターフェイスの指定ミス | <code>show logging</code> | `Devices > Platform Settings` で Syslog サーバーを設定し、対象 FTD へ Deploy を実行。 |
| **SNMPv3 トラップが NMS 側でデコードエラーになる** | FTD の Engine ID が NMS 側に正しく登録されていない | <code>show snmp-server</code> | FTD 管理 IP から算出した Engine ID (`0x<HEX_IP>01`) を NMS 側に登録する。 |

---

## ⚠ 制限事項

* **イベント送信レートと帯域制限**:
  * FMC 側で全接続イベント（Connection Events）に対して Syslog Alert Response を有効化すると、1 秒あたり数万件の Syslog が発生し、FMC の Disk Manager および `sftunnel` 帯域を枯渇させます。**接続ログの外部出力は FTD Platform Settings 経由で行う** のが原則です。
* **eStreamer と Syslog の機能差**:
  * パケットログ（Packet Header/Payload）やディスカバリイベントの全詳細データは Syslog では送信できず、**eStreamer (Event Streamer API)** でのみストリーミング可能です。

---

## 🔄 他技術との関連

* **Network Discovery Policy**: Impact Flag Alerting を機能させるための前提技術。ターゲットホストの OS やサービス脆弱性をあらかじめ FMC が把握しておく必要があります。
* **Cisco ISE (Identity Services Engine)**: Correlation Policy のレスポンスとして `Remediation` (ISE ANC / Rapid Threat Containment) を指定することで、攻撃を検出した瞬間に ISE 経由でスイッチポートを Quarantine (隔離) 状態へ変更可能。
* **Intrusion Policy (IPS)**: Intrusion Policy 内の `Advanced Settings > Syslog / SNMP Alerting` で、個別シグネチャ単位でのリアルタイムアラート出力を制御。

---

## 🧩 比較表

### 1. FMC Alert Responses vs Managed Device (FTD) Direct Syslog

| 比較項目 | FMC Alert Responses | FTD Direct Syslog (Platform Settings) |
| :--- | :--- | :--- |
| **送信元アプライアンス** | **FMC (Management Center)** | **FTD (Managed Device / Data Plane)** |
| **処理エンジン** | `SFDataCorrelator` (相関・影響度解析後) | LINA / Snort Engine (検知と同時) |
| **サポートイベント** | Impact Flag, Correlation, Discovery, AMP, Health | Connection, Intrusion, File/Malware, SI |
| **リアルタイム性** | わずかな転送・解析遅延あり | **極めて高い（リアルタイム直出し）** |
| **設定場所** | `Policies > Actions > Alerts` | `Devices > Platform Settings > Syslog` |

### 2. Impact Flag (影響度レベル) マトリクス

| Impact Level | フラグ名称 | 判定基準 | 推奨アクション |
| :--- | :--- | :--- | :--- |
| **Impact 1** | **Vulnerable (赤)** | ターゲットホストが存在し、**攻撃に対して実際に脆弱** | **即時 Email / Syslog アラートおよび自動隔離** |
| **Impact 2** | **Potentially Vulnerable (黄)** | ターゲットホストは存在するが、OS/サービス詳細が一部不明 | 注意監視・追跡調査 |
| **Impact 3** | **Currently Not Vulnerable (青)** | ターゲットホストは存在するが、**該当の脆弱性を持たない** | ログ記録のみ（低優先度） |
| **Impact 4** | **Unknown Target (グレー)** | ターゲット IP のホスト情報が FMC データベースに未登録 | ディスカバリ範囲の確認 |
| **Impact 0** | **Unknown** | 影響度判定対象外のイベント | 通常ログ処理 |

---

## 💡 ベストプラクティス

1. **二重アラート発生の回避（FMC vs FTD）**:
   * 侵入イベントに対して、FTD からの直接 Syslog と FMC からの Syslog Alert Response を両方有効化すると、全く同じイベントログが外部 SIEM に重複送信されます。セキュリティイベントは FTD 直出し、高度な相関/Impact アラートは FMC から送出するよう役割を分離します。
2. **Correlation Policy 内の Snooze Period（無応答期間）の設定**:
   * 同一の警告が短時間に大量発生してメールサーバーや通知先をパンク（メールループ）させないよう、Correlation Rule 設定画面で **`Snooze` (例: 1 hour)** を構成します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Impact Level 1 (Vulnerable) 侵入イベントの Syslog Alert Response 設定

#### 問題
FMC (`192.168.10.50`) において、Impact Level 1 (Vulnerable) と判定された重大な侵入イベントが発生した場合のみ、外部 Syslog サーバー (`192.168.10.100:514`, Facility: `ALERT`) へアラートを送信する設定を構成せよ。

#### 要件
* `Policies > Actions > Alerts` を使用すること。
* Syslog Alert プロファイル名: `Impact1_Syslog`

#### 設定・操作手順

```text
1. Policies > Actions > Alerts を開く。
2. "Create Alert" > "Create Syslog Alert" を選択。
   - Name: Impact1_Syslog
   - Host: 192.168.10.100
   - Port: 514
   - Facility: ALERT
   - Severity: ALERT
   - Tag: FMC_Impact1
   - Save をクリック。
3. "Impact Flag Alerts" タブを選択。
4. 最上部の Syslog ドロップダウンで "Impact1_Syslog" を選択。
5. Impact Flag Configuration テーブルで "1 Vulnerable" の Syslog Notification にのみチェックを入れる。
6. "Save" をクリック。
```

#### 検証方法
* FMC Bash で `sudo tail -f /var/log/messages` を実行。
* ターゲットホスト宛てに攻撃パケットを送信し、Impact 1 と判定されたイベント発生時に `192.168.10.100` 宛てに Syslog メッセージが生成されることを確認。

---

### 例2: SNMPv3 Trap Alert Response の構成

#### 問題
FMC 上で SNMPv3 トラップ通知プロファイル `SNMPv3_Alert_Response` を作成せよ。NMS サーバー IP は `192.168.10.200`、ユーザー名は `nms_admin`、暗号化アルゴリズムは `SHA` / `AES` とせよ。

#### 要件
* `Policies > Actions > Alerts` を使用すること。

#### 設定・操作手順

```text
1. Policies > Actions > Alerts を開く。
2. "Create Alert" > "Create SNMP Alert" を選択。
   - Name: SNMPv3_Alert_Response
   - Trap Server: 192.168.10.200
   - Version: Version 3
   - Username: nms_admin
   - Authentication Protocol: SHA
   - Authentication Password: AuthPassword123
   - Privacy Protocol: AES
   - Privacy Password: PrivPassword123
3. "Save" をクリック。
```

#### 検証方法
* 設定した `SNMPv3_Alert_Response` プロファイルが Alert 一覧で `Enabled` 表示になっていることを確認。

---

### 例3: Email Alert Response と Mail Relay ホストの設定

#### 問題
FMC において、Mail Relay サーバー `192.168.10.25` を登録し、重大アラート発生時に `soc@inelab.local` 宛てに送信元 `fmc@inelab.local` でメール通知するプロファイル `Email_SOC_Alert` を作成せよ。

#### 要件
* Mail Relay 設定および Email Alert Response 設定を完了させること。

#### 設定・操作手順

```text
1. System > Configuration > Email Notifications を開く。
   - Mail Relay Host: 192.168.10.25
   - Port: 25
   - Save をクリック。
2. Policies > Actions > Alerts を開く。
3. "Create Alert" > "Create Email Alert" を選択。
   - Name: Email_SOC_Alert
   - To: soc@inelab.local
   - From: fmc@inelab.local
4. "Save" をクリック。
```

#### 検証方法
* FMC CLI / Bash より Mail Relay サーバー `192.168.10.25:25` への TCP ポート疎通を確認。

---

### 例4: Correlation Rule ＆ Policy による SSH 失敗ログの Syslog アラート化

#### 問題
特定の SSH ログイン失敗侵入シグネチャ (SID: `1985`) が発生した際、これを検知して Syslog アラートを送信する Correlation Policy `CP_SSH_Attack` を作成し、有効化（Active）せよ。

#### 要件
* Correlation Rule 名: `CR_SSH_Failed`
* Correlation Policy 名: `CP_SSH_Attack`

#### 設定・操作手順

```text
1. Policies > Correlation > Rule Management > "Create Rule"
   - Rule Name: CR_SSH_Failed
   - If: an intrusion event occurs
   - Condition: Signature ID is 1985
   - Save をクリック。
2. Policies > Correlation > Policy Management > "Create Policy"
   - Policy Name: CP_SSH_Attack
   - "Add Rules" をクリックし、CR_SSH_Failed を選択。
   - Rule 右側の Response アイコン (吹き出し) をクリック。
   - Assigned Responses に "Impact1_Syslog" を追加し Update。
   - Save をクリック。
3. Policy Management 一覧で CP_SSH_Attack のステータススイッチを "On" に切り替える。
```

#### 検証方法
* 対象端末へ不適切なパスワードで SSH 接続を連打し、`Analysis > Correlation > Events` に Correlation Event が記録され、Syslog アラートが出力されることを確認。

---

### 例5: Cisco ISE ANC (Adaptive Network Control) と連携した自動端末隔離 Remediation

#### 問題
Correlation Rule で高度な脅威（Impact Level 1 または Malware 検知）を検出した際、Cisco ISE へ pxGrid 経由で Remediation アクション（ANC Policy: `Quarantine`）を発行して感染端末を自動隔離せよ。

#### 要件
* `Policies > Actions > Remediations` を使用すること。

#### 設定・操作手順

```text
1. Policies > Actions > Remediations > Modules を開く。
2. "Cisco ISE ANC Remediation" モジュールが有効であることを確認。
3. "Instances" タブで "Add Instance" をクリック。
   - Name: ISE_Quarantine_Instance
   - ISE pxGrid Server: 登録済み ISE を選択
   - ANC Policy: Quarantine
   - Save をクリック。
4. Policies > Correlation > Policy Management で該当ポリシーを開く。
5. Rule の Response アイコンをクリックし、"ISE_Quarantine_Instance" を割り当てて Policy を Save & On。
```

#### 検証方法
* マルウェア検知イベントを発生させ、ISE 側で該当クライアントの MAC アドレスに対して `Quarantine` 状態が適用されることを確認。

---

### 例6: Health Monitor Alert (CPU / Disk 閾値超え) の設定

#### 問題
FMC または FTD アプライアンスの CPU 使用率またはディスク使用率が **Critical** レベルに達した際、即座に Syslog アラートを発生させる Health Alert `HA_Resource_Critical` を構成せよ。

#### 要件
* `System > Health > Monitor Alerts` を使用すること。

#### 設定・操作手順

```text
1. System > Health > Monitor Alerts を開く。
2. "Add" をクリック。
   - Health Alert Name: HA_Resource_Critical
   - Severity: Critical
   - Alert: Impact1_Syslog (または定義済み Syslog)
   - Save をクリック。
3. CPU Usage および Disk Usage モジュールの Critical 列に作成した Alert がアサインされていることを確認。
```

#### 検証方法
* `System > Health > Health Monitor` 画面でテスト警告を発生させ、Syslog サーバーへ通知が届くことを確認。

---

### 例7: FTD アプライアンス直出し (Platform Settings) Syslog アラート設定

#### 問題
FTD アプライアンスからのセキュリティイベントログ（Connection / Intrusion Events）を、FMC を経由させず FTD のデータプレーンから直接 Syslog サーバー `192.168.10.100` (UDP 514, EMBLEM フォーマット) へ送信せよ。

#### 要件
* `Devices > Platform Settings` を使用すること。

#### 設定・操作手順

```text
1. Devices > Platform Settings を開き、対象 FTD の Threat Defense Policy を編集。
2. 左メニューから "Syslog" > "Syslog Servers" を選択。
3. "Add" をクリック:
   - Allow Syslog Accross Data Interfaces: チェック
   - IP Address: 192.168.10.100
   - Protocol: UDP / Port: 514
4. "Syslog Settings" を選択:
   - "Send syslogs in EMBLEM format" にチェック。
5. Policy を保存し、FTD へ Deploy。
6. Policies > Access Control > Logging タブで "Use Threat Defense Platform Settings" を選択して Deploy。
```

#### 検証方法
* FTD CLI で `show logging` を実行し、`Logging to 192.168.10.100` および `EMBLEM` フォーマットが有効になっていることを確認。

---

### 例8: Intrusion Policy 個別高度設定における Syslog Alerting の無効化/有効化

#### 問題
特定グループ向け Intrusion Policy (`DMZ_IPS_POLICY`) 内の全侵入シグネチャに対し、個別に設定された Syslog 宛先 (`192.168.10.105`, Facility: `LOCAL4`) へアラートを出力するようポリシーの Advanced Settings を構成せよ。

#### 要件
* `Policies > Intrusion > Intrusion Policy` 内で設定すること。

#### 設定・操作手順

```text
1. Policies > Intrusion > Intrusion Policy を開き "DMZ_IPS_POLICY" を編集。
2. 左ナビゲーションペインで "Advanced Settings" を選択。
3. "Syslog Alerting" の項目の Edit をクリック。
   - Syslog Alerting: Enabled
   - Logging Hosts: 192.168.10.105
   - Facility: LOCAL4
   - Priority: ALERT
4. Policy Information を開き "Commit Changes" を実行。
5. 該当 ACP にバインドして FTD へ Deploy。
```

#### 検証方法
* FTD CLI で `show access-list` および Snort 構成を確認し、侵入イベント発生時に `192.168.10.105` 宛てに LOCAL4 ファシリティでログが出力されることを確認。

---

### 例9: Discovery Event Alerting (新規ホスト / OS 検出時アラート) の設定

#### 問題
Network Discovery ポリシーによってネットワーク上で新規ホスト (New Host) または新規 OS (New OS) が検出された際、システム管理者に Syslog アラートを出力するよう構成せよ。

#### 要件
* `Policies > Actions > Alerts > Discovery Event Alerts` を使用すること。

#### 設定・操作手順

```text
1. Policies > Actions > Alerts を開き "Discovery Event Alerts" タブを選択。
2. 最上部の Syslog ドロップダウンで "Impact1_Syslog" を選択。
3. Discovery Event Configuration リストにおいて:
   - New Host: [✓]
   - New OS: [✓]
4. "Save" をクリック。
```

#### 検証方法
* 未知の IP / MAC アドレスを持つ新規端末から通信を発生させ、FMC の `Analysis > Discovery > Events` に記録されると同時に Syslog アラートが送信されることを確認。

---

### 例10: AMP for Networks (マルウェア検知) の外部アラート設定

#### 問題
FTD がネットワーク上で悪意あるファイル (Malware) を検知した場合、ならびに判定が後から変更された場合 (Retrospective Malware Event) に、セキュリティチーム宛てに Email アラートを自動送信せよ。

#### 要件
* `Policies > Actions > Alerts > Advanced Malware Protection Alerts` を使用すること。

#### 設定・操作手順

```text
1. Policies > Actions > Alerts を開き "Advanced Malware Protection Alerts" タブを選択。
2. Email ドロップダウンで "Email_SOC_Alert" を選択。
3. AMP Event Configuration において:
   - Malware Event: [✓]
   - Retrospective Malware Event: [✓]
4. "Save" をクリック。
```

#### 検証方法
* テスト用 EICAR ファイル等をダウンロードさせ、AMP クラウド照会によってマルウェア判定が下された際、定義したメールアドレスに通知が届くことを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で Correlation Rule `CR_1` を作成し、条件として「Impact Level が 1 の侵入イベントが発生した場合」を定義した。しかし、該当する侵入イベントが多数発生しているにもかかわらず、外部 Syslog サーバーに Correlation アラートが一切送信されない。考えられる原因として正しいものはどれか。

* A) Correlation Rule は Impact Level 1 を条件として指定できないため。
* B) Correlation Rule が Correlation Policy に追加されていない、または Correlation Policy のステータスが `On (Active)` に設定されていないため。
* C) FMC のライセンスが Threat から Base にダウングレードしているため。
* D) FTD アプライアンス側で `ip nbar protocol-discovery` が無効化されているため。

**正解**: **B**
**解説**: Correlation Rule（ルール）は単体で作成しただけでは機能しません。必ず **Correlation Policy（ポリシー）を作成してルールを追加し、レスポンス（Alert Response 等）を割り当てた上で、ポリシーのステートを `On (Active)` に変更する** 必要があります。

---

### 2. トラブルシュート問題

**問題**: FMC からセキュリティチーム宛てに Email Alert Response を設定したが、テストイベントを発生させてもメールが一切届かない。FMC Bash にログインし `/var/log/messages` を確認したところ、SMTP リレーサーバーへの接続失敗ログが記録されていた。要因の確認・解決策として適切なものはどれか。

* A) FTD の Platform Settings で `no-alias` コマンドを設定する。
* B) `System > Configuration > Email Notifications` (または Alert 設定) で指定している Mail Relay Host の IP アドレスおよび Port 25 の到達性を確認し、FMC 自身の IP アドレスが DNS 逆引き可能か確認する。
* C) FTD 上で `configure manager add` を再実行する。
* D) ACP の Default Action を Trust に変更する。

**正解**: **B**
**解説**: FMC からの Email アラート送信には、**有効な Mail Relay Host (SMTP サーバー) の指定** と **FMC 自身のアドレスの Reverse DNS 解決 (または正当な Hostname 保持)** が必須条件です。送信エラーが発生した場合は SMTP サーバーとの疎通および DNS 応答を確認します。

---

### 3. Design 問題

**問題**: 大規模エンタープライズネットワークにおいて、毎秒 50,000 件以上の接続イベント (Connection Events) が発生している。外部 SIEM サーバーへ接続ログを出力したいが、FMC アプライアンスの CPU 負荷と `sftunnel` 管理帯域の枯渇を防止したい。最も適切な設計・実装手法はどれか。

* A) FMC の `Policies > Actions > Alerts` で Connection Events 用の Syslog Alert Response を作成して送信する。
* B) FTD の `Devices > Platform Settings > Syslog` で Syslog サーバーを直接定義し、ACP の Logging タブで「Use Threat Defense Platform Settings」を選択して FTD のデータプレーンから直接 SIEM へ送出させる。
* C) Connection Events をすべて Email アラートで送信する。
* D) FMC の Disk Manager サービスを停止する。

**正解**: **B**
**解説**: 高トラフィック環境において、接続イベント（Connection Events）を FMC 経由のアラート（Alert Response）で処理させると、FMC への処理負荷および `sftunnel` 帯域が過剰に消費されます。**FTD の Platform Settings 経由で FTD データプレーンから直接 SIEM 宛てに Syslog 送出する** のが標準的な推奨設計です。

---

### 4. 実装問題

**問題**: Managed Device (FTD) から外部 NMS 宛てに直接 SNMPv3 トラップを送信するよう構成した。NMS アプライアンス側で FTD からの SNMPv3 トラップメッセージをデコードするために必要な FTD の Engine ID のフォーマットとして正しいものはどれか。

* A) FTD のシリアル番号の ASCII コード表記。
* B) FTD 管理インターフェイス IP アドレスの 16 進数表記の末尾に "01" を追加した値 (例: IP `172.16.1.50` ➔ `0xAC10013201`)。
* C) FMC の MAC アドレス。
* D) 常に `0x1234567890` 固定。

**正解**: **B**
**解説**: FTD アプライアンスが送出する SNMPv3 トラップの Engine ID は、送信元 FTD の管理インターフェイス IP アドレスを 16 進数（Hex）変換し、その末尾に **`01`** を付与した形式（例: `172.16.1.50` ➔ `AC.10.01.32` + `01` = `0xAC10013201`）となります。

---

### 5. コンフィグ読解問題

**問題**: FMC の `Impact Flag Alerts` 設定画面において、以下の図のように `Impact 1 (Vulnerable)` のみ Syslog Notification が有効化されている。この設定に関する説明として正しいものはどれか。

```text
[ Impact Flag Configuration ]
Impact Flag                  Syslog Notification
------------------------------------------------
0 Unknown                           [ ]
4 Unknown Target                    [ ]
3 Currently Not Vulnerable          [ ]
2 Potentially Vulnerable            [ ]
1 Vulnerable                        [✓]
```

* A) ターゲットホストの脆弱性に関係なく、すべての侵入イベントで Syslog アラートが発生する。
* B) Network Discovery によって把握されたターゲットホストが、検知された攻撃シグネチャに対して「実際に脆弱である (Vulnerable)」と FMC が自動相関判定した場合のみ、Syslog アラートが送信される。
* C) 外部攻撃者の IP アドレスが特定できなかった場合のみ Syslog アラートが送信される。
* D) この設定を有効にすると、FTD のデータプレーン転送が自動的に停止する。

**正解**: **B**
**解説**: **Impact 1 (Vulnerable)** は、FMC がネットワークディスカバリ情報（ホストの OS、サービス、ポート）と侵入イベントを相関分析し、**攻撃の対象ホストが実際にその攻撃に対して脆弱である** と判定した場合に付与される最高警戒レベルのフラグです。無駄なアラート（False Positive / 非脆弱ホストへの攻撃）を排除し、真に危険な攻撃のみを即座に Syslog アラート通知できます。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Administration Guide 7.1 - External Alerting with Alert Responses](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Administration Guide 7.1 - External Alerting for Intrusion Events](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot Syslog on Secure Firewall Management Center](https://www.cisco.com/)
* [Cisco Live: Advanced Threat Detection and Event Correlation with Cisco Secure Firewall (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * ラボ試験でアラート課題が出題された場合、最初に **「送出元は FMC か FTD か」「トリガーは Impact / Correlation / Health / Intrusion のどれか」「プロトコルは Syslog / SNMP / Email のどれか」** の 3 点を頭の中でマトリクス化整理してから設定画面へ移行してください。
* **図解（Impact Level 算定と Alert Response 判定モデル）**:
  ```text
  [ Intrusion Event Detected by FTD ]
                 │
                 ▼ (sftunnel: TCP 8305)
  [ FMC SFDataCorrelator Process ]
                 │
                 ├── Cross-check with Host Discovery Database
                 │   (Is Target Host IP in Network Map? Is Host OS/Service Vulnerable?)
                 │
                 ├── Match Result ➔ Impact Level Assigned
                 │   ├── Impact 1: Host IS Vulnerable (Red)
                 │   ├── Impact 2: Potentially Vulnerable (Yellow)
                 │   ├── Impact 3: Currently NOT Vulnerable (Blue)
                 │   └── Impact 4: Target Unknown (Gray)
                 │
                 ▼
  [ Check Impact Flag Alerting Rules / Correlation Policies ]
                 │
                 ├── If Condition Matched ──► Trigger Alert Response
                 │                               ├── Send Syslog to SIEM
                 │                               ├── Send SNMP Trap to NMS
                 │                               └── Send Email to Security Team
  ```
* **注意点**:
  * FMC の Web UI 上で Alert Response を新規作成した際、デフォルトで `Enabled` になっていますが、**Impact Flag Alerts や Correlation Policy への紐付けを行わない限り、アラートは一切送出されません**。作成後のバインド設定を絶対に忘れないでください。
