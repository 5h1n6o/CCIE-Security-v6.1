---
layout: default
title: 3.6.d-RMON
nav_order: 4
parent: 3.6-Monitoring
grand_parent: 3.0-Security-Infrastructure
---

# 3.6.d RMON

## 📘 概要

Remote Network Monitoring (RMON) は、IETF (RFC 2819 / RFC 4502) によって標準化されたネットワークモニタリング用の標準 MIB (Management Information Base) 拡張仕様およびモニタリングプロトコルです。標準の SNMP (Simple Network Management Protocol) がマネージャー (NMS) から定期的にPolling (ポーリング) を行って MIB 変数を取得する構成に依存しているのに対し、RMON はネットワーク機器（ルータ、スイッチ、セキュリティアプライアンス等）内部に RMON プローブ (Agent) 機能を持たせることで、ローカル環境で自律的な統計収集、閾値監視、およびイベント通知を実行します。

RMON1 (RFC 2819) は Layer 2 (MAC層) レベルのトラフィック解析および MIB オブジェクト監視に特化しており、主に以下の 9 つのグループで構成されます：
1. **Statistics Group**: インターフェイスの統計情報 (パケット数、バイト数、ドロップ数、エラー数等)
2. **History Group**: 一定間隔で統計データをサンプリングして保持
3. **Alarm Group**: 特定の MIB 変数に対する上昇閾値 (Rising Threshold) および下降閾値 (Falling Threshold) の設定と監視
4. **Event Group**: Alarm 発動時に実行するアクション (SNMP Trap 送信、ローカルログ記録等) の定義
5. **Host Group**: ネットワーク上の個々のホストの統計収集
6. **HostTopN Group**: トラフィック量の多い上位 N ホストの統計ランキング抽出
7. **Matrix Group**: ホスト間のペア間通信トラフィックマトリクス
8. **Filter Group**: 特定パターンにマッチするパケットのフィルタリング定義
9. **Packet Capture Group**: フィルタリングされたパケットのキャプチャバッファ管理

CCIE Security v6.1 ラボ試験においては、特に **RMON Alarm (Group 3)** および **RMON Event (Group 9)** の組み合わせて構成されるオンデバイス自律監視機能（Cisco IOS / IOS-XE CLI 上での `rmon alarm` および `rmon event` コマンド）の理解と実践的な設定が強く問われます。これにより、NMS の定常的なポーリングトラフィックを削減し、CPU 使用率高騰、メモリ枯渇、インターフェイスエラー率の上昇、パケットドロップといったネットワーク障害の予兆やセキュリティイベントを、デバイス単体でリアルタイム検知して即座に Trap や Syslog を自動発生させることが可能となります。


---

## 🔑 要点

RMON (特に CLI ベースの RMON Alarm / Event) の主要な技術的特徴、制限事項、設計上の注意点を下表に整理します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | SNMP MIB オブジェクトをデバイス内部で自律監視し、閾値超過時に動的に Trap 送信やログ記録を発生させる技術 |
| **用途** | CPU/メモリ使用率監視、インターフェイスエラー/CRC率監視、BGP/OSPF ネイバーフラップ検知、帯域枯渇アラート |
| **メリット** | NMS からのポーリングトラフィックを不要化。ネットワーク分離環境や WAN リンク越しのポーリング遅延・負荷を大幅軽減 |
| **デメリット** | 監視対象 MIB 変数ごとに OID の正確な把握が必要。サンプリング周期が短すぎるとデバイス自体の CPU 負荷が増加 |
| **対応機種** | Cisco IOS / IOS-XE ルータ・スイッチ (Catalyst 9000等)、Cisco ASA (一部制限あり)、Cisco FTD/FMC |
| **制限事項** | Cisco IOS / IOS-XE では主に Alarm Group と Event Group をサポート。全 9 グループのフル実装はプラットフォームに依存 |
| **設計上の注意点** | Rising/Falling 閾値の「ヒステリシス (Hysteresis)」構造を理解しないと、閾値境界線付近で大量のアラームが連続発生する (Trap 嵐) |


---

## 🏗 動作原理

RMON Alarm および Event は、デバイス内部でタイマー駆動のバックグラウンドプロセスとして動作し、ヒステリシス (Hysteresis) 原理に基づいて動作します。

### ヒステリシス (Hysteresis) 制御機構

ヒステリシスとは、アラームの乱発を防ぐための二重閾値（上昇閾値と下降閾値）制御メカニズムです。

```
[ MIB 値の変動パターンと RMON アラームの発生順序 ]

 MIB 値
  ▲
  │                          (A) Rising Event 発動!
  │                             ┌───┐
高│── ── ── ── ── ── ── ── ── ─/─── ──\── ── ── ── ── ── [ Rising Threshold ] (例: 80%)
  │                           /     \
  │                          /       \       (B) 無視 (すでに Rising 状態のため)
  │                         /         \       ┌───┐
  │                        /           \── ──/─── ──\── ─ [ Rising Threshold ]
  │                       /             \   /     \
  │                      /               \ /       \
低│── ── ── ── ── ── ──/── ── ── ── ── ── ─X─ ── ── ───\─── [ Falling Threshold ] (例: 60%)
  │                   /                   \             \
  │                  /                     \───┐         \
  │                 /                          └─── ── ── └── [ Falling Event 発動!]
  └─────────────────────────────────────────────────────────────► 時間 (Time)
```

1. **初期状態 (Reset State)**:
   - システム起動時、アラーム状態はリセットされています。
   - `sample-type` (Absolute または Delta) に従って、指定したサンプリング周期 (`interval`) ごとに指定した OID の MIB 変数を読み取ります。

2. **Rising Threshold (上昇閾値) の超過**:
   - 監視対象の MIB 変数が上昇閾値 (`rising-threshold`) を下から上に突破した瞬間、**Rising Event** が一度だけ発動します (上図 A)。
   - イベントが発動すると、定義されたアクション (`log`, `trap`, または `log-trap`) が実行されます。
   - **重要**: 一度 Rising Event が発動すると、MIB 値が Rising Threshold を超え続けている間、または変動して再度 Rising Threshold を上回っても、**Falling Threshold を一度下回るまでは次の Rising Event は絶対に発生しません** (上図 B)。

3. **Falling Threshold (下降閾値) の割り込み**:
   - MIB 変数が低下し、下降閾値 (`falling-threshold`) を上から下に下回った瞬間、**Falling Event** が一度だけ発動します。
   - Falling Event が発動することにより、Alarm の状態が再度リセットされ、次の Rising Event を検知できる状態に戻ります。


---

## ⚙ 動作シーケンス

RMON Alarm および Event によるパケットおよび内部イベントの処理シーケンスは以下の通りです。

```
[ デバイス内部 RMON プロセス ]                               [ 管理サーバー / Syslog / NMS ]
           │                                                              │
           │ 1. サンプリングタイマー起動 (例: interval 30秒)                │
           │─────────────────────────────────────────────────────────────►│
           │                                                              │
           │ 2. OID オブジェクト値の取得 (例: .1.3.6.1.4.1.9.2.1.56.0)     │
           │    Absolute (絶対値) または Delta (前回差分) を算出          │
           │                                                              │
           │ 3. 閾値判定ロジックの実行:                                   │
           │    ┌──────────────────────────────────────────────────┐      │
           │    │ IF (Current_Value >= Rising_Threshold)           │      │
           │    │    AND (Alarm_State == RESET) THEN               │      │
           │    │      Alarm_State = RISING_SET                   │      │
           │    │      Execute Rising Event Action                 │      │
           │    │ ELSE IF (Current_Value <= Falling_Threshold)     │      │
           │    │    AND (Alarm_State == RISING_SET) THEN          │      │
           │    │      Alarm_State = RESET                         │      │
           │    │      Execute Falling Event Action                │      │
           │    └──────────────────────────────────────────────────┘      │
           │                                                              │
           │ 4. Event アクションの実行:                                   │
           │    - Event Type = log ──> RMON 内部ログテーブルに記録        │
           │    - Event Type = trap ──> SNMP Trap パケットを送出 ────────►│ (UDP 162)
           │    - Event Type = log-trap ──> ログ記録 & Trap 送信 ─────────►│ (Syslog/Trap)
           │                                                              │
```


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security ラボ試験における RMON の出題ポイント、設定上の必須留意事項、およびトラブルシューティングの落とし穴を解説します。

### 1. ラボ試験での主な出題シナリオ
* **CPU 使用率監視**: CPU 5分平均使用率 (または 5秒平均使用率) が 80% を超えた場合に SNMP Trap および Syslog ログを出力し、60% 以下に低下した際にクリアアラームを送信する設定。
* **メモリ使用率・残量監視**: メモリ空き容量 (Free Memory) が指定バイト数を下回った場合の即時アラート通知。
* **インターフェイスエラー・ドロップ監視**: ギガビットイーサネットインターフェイスにおける CRC エラー数 (`ifInErrors`) やドロップ数 (`ifInDiscards`) の「デルタ値 (Delta Rate)」監視。
* **SNMP 連携設定**: RMON Event 発動時に送出される Trap を受信するために必要な `snmp-server enable traps rmon` および `snmp-server host` の必須コマンド追加。

### 2. 試験で狙われやすい設定の落とし穴
* **`sample-type` の選択ミス (Absolute vs Delta)**:
  * **Absolute (絶対値)**: CPU 使用率やメモリ使用量など、オブジェクトの値自体がそのまま状態を表す MIB 変数に使用します。
  * **Delta (差分値)**: インターフェイスの受信バイト数 (`ifInOctets`) やエラー数 (`ifInErrors`) のように、カウントが累積していくカウンタ変数に使用します。カウンタ変数に対して Absolute を指定すると、累積値が即座に Rising Threshold を超過し続けて誤動作します。
* **OID の指定方法とインスタンス番号 (`.0` 等) の欠落**:
  * Cisco IOS MIB 変数 (例: `cpmCPUTotal5minRev` 等) の指定時に、末尾の `.0` (スカラーインスタンス) やインターフェイス index (`ifIndex`) をつけ忘れると、RMON プロセスが MIB オブジェクトを参照できずアラームが全く発動しません。
* **RMON Trap の有効化漏れ**:
  * `rmon event 1 trap public` を設定しても、グローバルコンフィグで `snmp-server enable traps rmon` を設定していないと、実際には UDP 162 番ポートから Trap パケットがネットワークへ送出されません。
* **`startup-alarm` の初期状態指定**:
  * `rmon alarm` コマンドの末尾に指定する `startup-alarm` (例: `rising-alarm`, `falling-alarm`, `rising-or-falling-alarm`) の選択。デフォルトは `rising-or-falling-alarm` ですが、問題文で「起動時に値が閾値を超えていた場合のみ発生させること」などの条件がある場合に意識する必要があります。


---

## 🛠 設定方法

Cisco IOS / IOS-XE における RMON Alarm および Event の標準的なコマンド構文および設定例を以下に示します。

### 構文解説

#### 1. RMON Event の定義
```text
rmon event <event-id> [log] [trap <community-string>] [description <string>] [owner <string>]
```
* `<event-id>`: イベント識別番号 (例: 1〜65535)。
* `log`: デバイスのローカル RMON ログテーブルに記録。
* `trap <community-string>`: 指定したコミュニティ名で SNMP Trap を送信。
* `log-trap <community-string>`: ログ記録と Trap 送信の両方を実行。

#### 2. RMON Alarm の定義
```text
rmon alarm <alarm-id> <variable-oid> <interval-seconds> {absolute | delta} rising-threshold <value> <rising-event-id> falling-threshold <value> <falling-event-id> [owner <string>]
```
* `<alarm-id>`: アラーム識別番号。
* `<variable-oid>`: 監視対象の SNMP MIB OID (ドット付き数値記法または標準名)。
* `<interval-seconds>`: サンプリング間隔 (秒)。
* `absolute`: 絶対値比較 (CPU/メモリ等)。
* `delta`: 前回サンプリング時からの増加量比較 (トラフィック/エラーカウンタ等)。
* `rising-threshold <value> <rising-event-id>`: 上昇閾値と到達時に発動する Event ID。
* `falling-threshold <value> <falling-event-id>`: 下降閾値と到達時に発動する Event ID。

---

### 設定例 1: CPU 使用率 5分平均の監視 (IOS / IOS-XE)

CPU 使用率 5分平均 (`cpmCPUTotal5minRev`: OID `.1.3.6.1.4.1.9.9.109.1.1.1.1.8.1`) を 30 秒ごとに監視し、80% 超過で Rising Event (ID 10)、60% 低下で Falling Event (ID 11) を発動。

```bash
! --- 1. RMON Event の定義 ---
rmon event 10 log-trap public description "CPU_Utilization_High_Alert" owner admin
rmon event 11 log-trap public description "CPU_Utilization_Normal_Clear" owner admin

! --- 2. RMON Alarm の定義 ---
! OID: .1.3.6.1.4.1.9.9.109.1.1.1.1.8.1 (5分CPU使用率, インスタンス .1)
! サンプリング間隔: 30秒, サンプルタイプ: absolute
rmon alarm 1 .1.3.6.1.4.1.9.9.109.1.1.1.1.8.1 30 absolute rising-threshold 80 10 falling-threshold 60 11 owner admin

! --- 3. SNMP サーバーおよび Trap 送信の有効化 ---
snmp-server community public RO
snmp-server enable traps rmon
snmp-server host 192.168.100.50 version 2c public
```

---

### 設定例 2: インターフェイス CRC / 入力エラー率の監視 (IOS / IOS-XE)

GigabitEthernet0/1 (`ifIndex` = 2) の入力エラー数 (`ifInErrors`: OID `.1.3.6.1.2.1.2.2.1.14.2`) を 60 秒ごとの Delta (増加量) で監視し、60 秒間に 10 エラー以上発生で Rising Event (ID 20)、1 エラー以下で Falling Event (ID 21) を発動。

```bash
! --- 1. RMON Event の定義 ---
rmon event 20 log-trap public description "Gi0/1_Input_Error_Rate_High" owner admin
rmon event 21 log-trap public description "Gi0/1_Input_Error_Rate_Normal" owner admin

! --- 2. RMON Alarm の定義 ---
! OID: .1.3.6.1.2.1.2.2.1.14.2 (ifInErrors.2)
! サンプリング間隔: 60秒, サンプルタイプ: delta
rmon alarm 2 .1.3.6.1.2.1.2.2.1.14.2 60 delta rising-threshold 10 20 falling-threshold 1 21 owner admin

! --- 3. SNMP Trap 設定 ---
snmp-server enable traps rmon
snmp-server host 192.168.100.50 version 2c public
```


---

## 🔍 検証コマンド

RMON の動作状態確認、アラーム定義の検証、および発動ログの確認に使用する主要コマンド一覧です。

| 目的 | コマンド | 主な確認ポイント |
| --- | --- | --- |
| RMON アラーム設定一覧確認 | `show rmon alarms` | 現在アクティブな Alarm ID、監視 OID、サンプリング間隔、現在値、閾値、次回判定タイマー |
| RMON イベント設定一覧確認 | `show rmon events` | 定義されている Event ID、アクションタイプ (log/trap)、発動回数 (Last time sent) |
| RMON ログテーブル確認 | `show rmon logs` | Event アクションによりローカルに記録されたログエントリ、発動タイムスタンプ |
| RMON 統計情報確認 | `show rmon statistics` | インターフェイスレベルの RMON Group 1 統計情報 |
| SNMP 宛先・Trap 設定確認 | `show snmp host` | RMON Trap パケットの送信先 IP、コミュニティ名、バージョン |
| デバッグ確認 | `debug rmon alarm` | RMON Alarm のサンプリング処理、閾値判定、Event 発動プロセスのリアルタイムログ |
| デバッグ確認 | `debug snmp packets` | RMON Event 発動時に送出される SNMP Trap パケットのデバッグ |


---

## 🚨 トラブルシュート

ラボ試験や実運用のトラブルシューティングで頻出する症状、原因、確認コマンド、および対処方法を下表にまとめます。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| アラーム条件を満たしても SNMP Trap が送信されない | `snmp-server enable traps rmon` または `snmp-server host` の設定漏れ | `show snmp host`<br>`show running-config \| include snmp` | グローバルコンフィグで `snmp-server enable traps rmon` および `snmp-server host <IP> version 2c <community>` を追加 |
| RMON Alarm の現在値 (`Current Value`) が常に 0 または Err | 指定した OID の末尾にインスタンス ID (`.0` や `.ifIndex`) が欠落している | `show rmon alarms`<br>`debug rmon alarm` | 対象 MIB オブジェクトの完全な OID (スカラー変数なら末尾に `.0`) を正しく再指定する |
| アラームが意図せず連続発生する (Trap 嵐) | `rising-threshold` と `falling-threshold` の差が小さすぎる、または同一 | `show rmon alarms` | ヒステリシス幅を拡大し、変動幅を考慮した `falling-threshold` を正しく設定する |
| カウンタ変数でサンプリング直後に爆発的な値が検出される | `sample-type` に `absolute` を指定している | `show rmon alarms` | インターフェイスバイト数やエラー数等のカウンタ変数には必ず `delta` を指定する |
| `show rmon logs` にログが一切残らない | `rmon event` に `log` または `log-trap` オプションが含まれていない (`trap` のみ) | `show rmon events` | `rmon event <id> log-trap <community>` に修正設定する |


## ⚠ 制限事項

RMON の実装および運用における主要な制限事項は以下の通りです。

* **プラットフォーム制限**:
  * Cisco IOS / IOS-XE において CLI から直接設定可能な機能は、主に **Group 3 (Alarm)** および **Group 9 (Event)** に限定されます。Statistics や History 等の他のグループは SNMP NMS 側からの MIB 設定（`rmonOnPort` 等）によってのみアクティブ化されるプラットフォームが多く存在します。
  * Cisco ASA ソフトウェアでは、一般的な RMON Alarm / Event CLI サポートが制限されており、EEM (Embedded Event Manager) や SNMP Trap 設定による代替が推奨されます。
* **OID の指定制限**:
  * RMON Alarm で指定できる MIB 変数は、原則として Integer (整数型)、Counter (カウンタ型)、または Gauge (ゲージ型) のデータ型を持つスカラー MIB オブジェクトまたはテーブルの特定行インスタンスに限られます。Octet String や DisplayString などの文字列型変数は指定できません。
* **サンプリングオーバーヘッド**:
  * サンプリング間隔 (`interval`) を極端に短く (例: 1〜2秒) 設定し、かつ多数の RMON Alarm を同時稼働させると、RMON バックグラウンドプロセスによる CPU 使用率増加およびメモリ使用量増大を招くリスクがあります。


---

## 🔄 他技術との関連

RMON は独立したプロトコルではなく、他の監視・制御技術と深く連携します。

* **SNMP (v1 / v2c / v3)**:
  * RMON の基盤プロトコル。RMON Event で発生する Trap は SNMP の Trap 処理エンジン（`snmp-server host`）を介して送出されます。SNMPv3 と組み合わせることで、RMON Trap の暗号化（AuthPriv）および改ざん防止が実現します。
* **Syslog**:
  * RMON Event のアクションとしてログ記録を指定した場合、デバイス内部の RMON ログテーブルに記録されるとともに、`logging on` が有効であれば Syslog メッセージとして外部 Syslog サーバーにも転送可能です。
* **EEM (Embedded Event Manager)**:
  * Cisco IOS / IOS-XE において、RMON は EEM のイベントトリガー源 (Event Detector) として活用できます。RMON Alarm が発生した際、単に Trap を送信するだけでなく、EEM スクリプトを呼び出してインターフェイスの `shutdown` や設定の自動変更（Auto-remediation）を実行できます。
* **Control Plane Policing (CoPP)**:
  * RMON 処理自体はコントロールプレーンのバックグラウンドタスクとして動作します。大量のトラフィック増大時に RMON プロセスが妨げられないよう、CoPP で管理通信や内部分析プロセスに必要な制御リソースを保護します。


---

## 🧩 比較表

ネットワーク監視手法としての RMON、標準 SNMP Polling、および Telemetry / NetFlow の比較を下表に示します。

| 比較項目 | RMON (Alarm / Event) | 標準 SNMP Polling | Model-Driven Telemetry / NetFlow |
| --- | --- | --- | --- |
| **動作モデル** | **オンデバイス自律検知 (Event-Driven)** | **NMS 定期照会 (Polling-based)** | **ストリーミング / フロー収集** |
| **NMS 帯域負荷** | **極めて低い** (異常発生時の Trap 送信のみ) | **高** (定常的な定期リクエスト通信) | **中〜高** (データ量・サンプリング依存) |
| **障害検知スピード** | **リアルタイム** (サンプリング周期依存) | **遅延あり** (次回 Polling 周期まで不明) | **超リアルタイム** (ミリ秒〜秒単位) |
| **設定の場所** | **ネットワーク機器の CLI** | **NMS サーバー側** | **機器 CLI ＆ 収集サーバー** |
| **適した用途** | **閾値超過アラート、即時ログ通知** | **長期的な傾向分析・グラフ化** | **高度なトラフィック解析、詳細パフォーマンス** |
| **CPU 負荷** | **低〜中** (ローカルサンプリングのみ) | **低** (リクエスト応答処理のみ) | **中** (データエンコード/ストリーミング処理) |


---

## 💡 ベストプラクティス

1. **適切なサンプリング間隔 (`interval`) の設計**:
   * 急激な変化を監視する項目 (CPU やトラフィック急増) には **10〜30秒**、緩やかな変化を監視する項目 (メモリ使用量やディスク容量) には **60〜300秒** のサンプリング間隔を設定します。
2. **十分なヒステリシス幅の確保**:
   * Rising Threshold と Falling Threshold の間には、少なくとも 15〜20% 以上のマージン（差分）を持たせます。例えば、CPU 監視では Rising 80% / Falling 60% のように設定し、境界値でのアラーム連続発生（チャタリング）を防止します。
3. **正確な OID とインスタンスの事前検証**:
   * RMON Alarm 設定前に、対象機器の CLI 上で `show snmp mib` または NMS 上の MIB Browser で OID とそのインスタンス番号 (末尾 `.0` や `.ifIndex`) を確実に確認します。
4. **SNMPv3 との統合**:
   * 監視データの改ざんや平文 Trap の傍受を防ぐため、RMON Event の Trap 通知には SNMPv3 AuthPriv（`authPriv`）セキュリティレベルを適用可能な構文を組み合わせます。


## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験レベルの実践的な RMON 設定・演習シナリオ 11 選です。すべての設定例は省略せず、最後まで完全なコンフィグレーション形式で記載しています。

---

### シナリオ 1: CPU 5分平均使用率の高騰および収束アラート

#### 問題
ルータ R1 において、CPU 5分平均使用率が 80% を超過した場合に SNMP Trap およびローカルログを発生させ、50% 以下に低下した際にクリアアラートを送信するように設定しなさい。

#### 要件
* 監視対象 OID: `.1.3.6.1.4.1.9.9.109.1.1.1.1.8.1` (`cpmCPUTotal5minRev.1`)
* サンプリング間隔: 20 秒
* サンプルタイプ: Absolute (絶対値)
* 上昇閾値 (Rising): 80% (Event ID: 101)
* 下降閾値 (Falling): 50% (Event ID: 102)
* Community 名: `SecLabComm`
* NMS 宛先: `192.168.10.100`

#### 設定例
```bash
! --- R1 Configuration ---
rmon event 101 log-trap SecLabComm description "CPU_5min_High_Threshold_Exceeded" owner Admin
rmon event 102 log-trap SecLabComm description "CPU_5min_Normal_Threshold_Recovered" owner Admin

rmon alarm 101 .1.3.6.1.4.1.9.9.109.1.1.1.1.8.1 20 absolute rising-threshold 80 101 falling-threshold 50 102 owner Admin

snmp-server community SecLabComm RO
snmp-server enable traps rmon
snmp-server host 192.168.10.100 version 2c SecLabComm
```

#### 検証方法
```bash
show rmon alarms
show rmon events
show rmon logs
```

---

### シナリオ 2: システムメモリ空き容量 (Free Memory) 枯渇アラート

#### 問題
ルータ R1 のプロセスメモリ空き容量 (`ciscoMemoryPoolFree`) を監視し、空き容量が 50MB (52,428,800 バイト) を下回った場合にアラームを発生させ、100MB (104,857,600 バイト) 以上に回復した際に通知しなさい。

#### 要件
* 監視対象 OID: `.1.3.6.1.4.1.9.9.48.1.1.1.6.1` (`ciscoMemoryPoolFree.1`)
* サンプリング間隔: 30 秒
* サンプルタイプ: Absolute
* 下降閾値 (Falling - メモリ低下): 52,428,800 バイト (Event ID: 201)
* 上昇閾値 (Rising - メモリ回復): 104,857,600 バイト (Event ID: 202)
* 初期スタートアップアラーム: `falling-alarm`

#### 設定例
```bash
! --- R1 Configuration ---
rmon event 201 log-trap SecLabComm description "Memory_Free_Space_Critically_Low" owner Admin
rmon event 202 log-trap SecLabComm description "Memory_Free_Space_Restored" owner Admin

! Falling Threshold を低数値 (メモリ低下時) に設定し、Rising Threshold を高数値 (メモリ回復時) に割り当てる応用構成
rmon alarm 201 .1.3.6.1.4.1.9.9.48.1.1.1.6.1 30 absolute rising-threshold 104857600 202 falling-threshold 52428800 201 startup-alarm falling-alarm owner Admin

snmp-server community SecLabComm RO
snmp-server enable traps rmon
snmp-server host 192.168.10.100 version 2c SecLabComm
```

#### 検証方法
```bash
show rmon alarms
```

---

### シナリオ 3: GigabitEthernet0/1 インターフェイスの CRC / 入力エラー率監視

#### 問題
GigabitEthernet0/1 (`ifIndex` = 2) における 60 秒間の入力エラー増加量 (`ifInErrors`) を監視し、60 秒間に 15 エラー以上発生した場合にアラームを出力しなさい。

#### 要件
* 監視対象 OID: `.1.3.6.1.2.1.2.2.1.14.2` (`ifInErrors.2`)
* サンプリング間隔: 60 秒
* サンプルタイプ: Delta (差分値)
* 上昇閾値 (Rising): 15 エラー (Event ID: 301)
* 下降閾値 (Falling): 2 エラー (Event ID: 302)

#### 設定例
```bash
! --- R1 Configuration ---
rmon event 301 log-trap SecLabComm description "Gi0/1_Input_Errors_Delta_High" owner Admin
rmon event 302 log-trap SecLabComm description "Gi0/1_Input_Errors_Delta_Cleared" owner Admin

rmon alarm 301 .1.3.6.1.2.1.2.2.1.14.2 60 delta rising-threshold 15 301 falling-threshold 2 302 owner Admin

snmp-server community SecLabComm RO
snmp-server enable traps rmon
snmp-server host 192.168.10.100 version 2c SecLabComm
```

#### 検証方法
```bash
show rmon alarms
show rmon events
```

---

### シナリオ 4: WAN 出力パケットドロップ (Output Discards) 監視

#### 問題
WAN インターフェイス GigabitEthernet0/0 (`ifIndex` = 1) での 30 秒間のパケットドロップ数 (`ifOutDiscards`) を監視し、帯域輻輳に伴うドロップ発生を即座に検知しなさい。

#### 要件
* 監視対象 OID: `.1.3.6.1.2.1.2.2.1.19.1` (`ifOutDiscards.1`)
* サンプリング間隔: 30 秒
* サンプルタイプ: Delta
* 上昇閾値 (Rising): 50 パケットドロップ (Event ID: 401)
* 下降閾値 (Falling): 0 パケットドロップ (Event ID: 402)

#### 設定例
```bash
! --- R1 Configuration ---
rmon event 401 log-trap SecLabComm description "Gi0/0_Output_Discards_High" owner Admin
rmon event 402 log-trap SecLabComm description "Gi0/0_Output_Discards_Normal" owner Admin

rmon alarm 401 .1.3.6.1.2.1.2.2.1.19.1 30 delta rising-threshold 50 401 falling-threshold 0 402 owner Admin

snmp-server community SecLabComm RO
snmp-server enable traps rmon
snmp-server host 192.168.10.100 version 2c SecLabComm
```

#### 検証方法
```bash
show rmon alarms
```

---

### シナリオ 5: BGP エスタブリッシュ状態ネイバー数の監視

#### 問題
BGP 確立済ネイバー数 (`bgpPeerState`) を監視し、確立状態であるネイバー数 (`Established` = 6) の変調または切断を迅速にアラート検知しなさい。

#### 要件
* 監視対象 OID: `.1.3.6.1.2.1.15.3.1.2.192.168.1.2` (`bgpPeerState.192.168.1.2`)
* サンプリング間隔: 15 秒
* サンプルタイプ: Absolute
* 下降閾値 (Falling - ネイバーダウン): 5 (Established=6 未満) (Event ID: 501)
* 上昇閾値 (Rising - ネイバー復旧): 6 (Established) (Event ID: 502)

#### 設定例
```bash
! --- R1 Configuration ---
rmon event 501 log-trap SecLabComm description "BGP_Peer_192.168.1.2_Down_Alert" owner Admin
rmon event 502 log-trap SecLabComm description "BGP_Peer_192.168.1.2_Established_Restored" owner Admin

rmon alarm 501 .1.3.6.1.2.1.15.3.1.2.192.168.1.2 15 absolute rising-threshold 6 502 falling-threshold 5 501 startup-alarm falling-alarm owner Admin

snmp-server community SecLabComm RO
snmp-server enable traps rmon
snmp-server host 192.168.10.100 version 2c SecLabComm
```

#### 検証方法
```bash
show rmon alarms
```

---

### シナリオ 6: ルーティングテーブル変動 (Route Flap) の監視

#### 問題
ルータ内の活性 IPv4 ルート数 (`ipCidrRouteNumber`) または経路変更カウンターを監視し、30 秒間に 20 回以上のルート変更が発生した場合に経路フラッピングアラートを出力しなさい。

#### 要件
* 監視対象 OID: `.1.3.6.1.2.1.4.24.3.0` (`ipCidrRouteNumber.0`)
* サンプリング間隔: 30 秒
* サンプルタイプ: Delta
* 上昇閾値 (Rising): 20 変更 (Event ID: 601)
* 下降閾値 (Falling): 2 変更 (Event ID: 602)

#### 設定例
```bash
! --- R1 Configuration ---
rmon event 601 log-trap SecLabComm description "IP_Route_Flap_Storm_Detected" owner Admin
rmon event 602 log-trap SecLabComm description "IP_Route_Flap_Stabilized" owner Admin

rmon alarm 601 .1.3.6.1.2.1.4.24.3.0 30 delta rising-threshold 20 601 falling-threshold 2 602 owner Admin

snmp-server community SecLabComm RO
snmp-server enable traps rmon
snmp-server host 192.168.10.100 version 2c SecLabComm
```

#### 検証方法
```bash
show rmon alarms
```

---

### シナリオ 7: ブロードキャストストーム発生検知

#### 問題
スイッチ SW1 の VLAN 10 インターフェイス (Vlan10, `ifIndex` = 15) における 10 秒間の受信ブロードキャストパケット数 (`ifInBroadcastPkts`) を監視し、ストーム発生を即座に通知しなさい。

#### 要件
* 監視対象 OID: `.1.3.6.1.2.1.31.1.1.1.3.15` (`ifInBroadcastPkts.15`)
* サンプリング間隔: 10 秒
* サンプルタイプ: Delta
* 上昇閾値 (Rising): 5000 パケット (Event ID: 701)
* 下降閾値 (Falling): 500 パケット (Event ID: 702)

#### 設定例
```bash
! --- SW1 Configuration ---
rmon event 701 log-trap SecLabComm description "Vlan10_Broadcast_Storm_Detected" owner Admin
rmon event 702 log-trap SecLabComm description "Vlan10_Broadcast_Traffic_Normal" owner Admin

rmon alarm 701 .1.3.6.1.2.1.31.1.1.1.3.15 10 delta rising-threshold 5000 701 falling-threshold 500 702 owner Admin

snmp-server community SecLabComm RO
snmp-server enable traps rmon
snmp-server host 192.168.10.100 version 2c SecLabComm
```

#### 検証方法
```bash
show rmon alarms
```

---

### シナリオ 8: IPsec VPN アクティブトンネル数の閾値監視

#### 問題
Cisco 機器上のアクティブ IPsec 思想トンネル数 (`ciscoIpsecFlowActiveTunnels`) を監視し、50 トンネルを超えた場合に高負荷アラートを出力しなさい。

#### 要件
* 監視対象 OID: `.1.3.6.1.4.1.9.9.171.1.2.1.1.0` (`cpmIPsecFlowActiveTunnels.0`)
* サンプリング間隔: 60 秒
* サンプルタイプ: Absolute
* 上昇閾値 (Rising): 50 トンネル (Event ID: 801)
* 下降閾値 (Falling): 30 トンネル (Event ID: 802)

#### 設定例
```bash
! --- R1 Configuration ---
rmon event 801 log-trap SecLabComm description "IPsec_Active_Tunnels_High" owner Admin
rmon event 802 log-trap SecLabComm description "IPsec_Active_Tunnels_Normal" owner Admin

rmon alarm 801 .1.3.6.1.4.1.9.9.171.1.2.1.1.0 60 absolute rising-threshold 50 801 falling-threshold 30 802 owner Admin

snmp-server community SecLabComm RO
snmp-server enable traps rmon
snmp-server host 192.168.10.100 version 2c SecLabComm
```

#### 検証方法
```bash
show rmon alarms
```

---

### シナリオ 9: RMON Alarm と EEM (Embedded Event Manager) の自動復旧連携

#### 問題
RMON Alarm で CRC エラー急増やトラフィック異常を検知した際、EEM アクションを連動させてトラブルの起因となっているポート GigabitEthernet0/2 を自動的に `shutdown` しなさい。

#### 要件
* RMON Event 901 発動時に Syslog メッセージ `%RMON-5-RISING_THRESHOLD` を出力。
* EEM Applet がこの Syslog を検知し、GigabitEthernet0/2 を自動 `shutdown`。

#### 設定例
```bash
! --- 1. RMON Event & Alarm 設定 ---
rmon event 901 log description "CRC_Storm_On_Gi0_2" owner Admin
rmon alarm 901 .1.3.6.1.2.1.2.2.1.14.3 10 delta rising-threshold 100 901 falling-threshold 5 901 owner Admin

! --- 2. EEM Applet 設定 ---
event manager applet AUTO_SHUT_GI0_2
 event syslog pattern "RMON-5-RISING_THRESHOLD.*alarm 901"
 action 1.0 cli command "enable"
 action 2.0 cli command "configure terminal"
 action 3.0 cli command "interface GigabitEthernet0/2"
 action 4.0 cli command "shutdown"
 action 5.0 cli command "description AUTO_SHUTDOWN_BY_RMON_CRC_STORM"
 action 6.0 syslog msg "EEM Action Executed: GigabitEthernet0/2 shutdown due to RMON Alarm 901"
```

#### 検証方法
```bash
show event manager history events
show interface GigabitEthernet0/2
```

---

### シナリオ 10: SNMPv3 セキュア環境における RMON Trap 送信統合

#### 問題
RMON Event 1001 の通知先として、暗号化 (AuthPriv) された SNMPv3 ユーザー (`v3admin`) を指定し、安全に Trap を送信しなさい。

#### 要件
* SNMPv3 Group: `ADMIN_GRP` (v3 priv)
* SNMPv3 User: `v3admin` (Auth: SHA / Pass: `AuthPass123`, Priv: AES128 / Pass: `PrivPass123`)
* RMON Event コミュニティ引数に SNMPv3 ユーザー名を適用。

#### 設定例
```bash
! --- 1. SNMPv3 基本設定 ---
snmp-server group ADMIN_GRP v3 priv
snmp-server user v3admin ADMIN_GRP v3 auth sha AuthPass123 priv aes 128 PrivPass123
snmp-server host 192.168.10.100 version 3 priv v3admin

! --- 2. RMON Event & Alarm 設定 ---
! trap 引数に v3admin ユーザー名を指定
rmon event 1001 log-trap v3admin description "Secure_SNMPv3_RMON_Event" owner Admin
rmon alarm 1001 .1.3.6.1.4.1.9.9.109.1.1.1.1.8.1 30 absolute rising-threshold 85 1001 falling-threshold 55 1001 owner Admin

snmp-server enable traps rmon
```

#### 検証方法
```bash
show snmp user
show rmon events
```

---

### シナリオ 11: 筐体内部温度 MIB の監視

#### 問題
Cisco ルータの環境温度 MIB (`ciscoEnvMonTemperatureStatusValue`) を 60 秒ごとに監視し、内部温度が 55 ℃ を超えた場合にあらかじめ定義した緊急イベントを発動させなさい。

#### 要件
* 監視対象 OID: `.1.3.6.1.4.1.9.9.13.1.3.1.3.1` (`ciscoEnvMonTemperatureStatusValue.1`)
* サンプリング間隔: 60 秒
* サンプルタイプ: Absolute
* 上昇閾値 (Rising): 55 ℃ (Event ID: 1101)
* 下降閾値 (Falling): 40 ℃ (Event ID: 1102)

#### 設定例
```bash
! --- R1 Configuration ---
rmon event 1101 log-trap SecLabComm description "High_Chassis_Temperature_Critical" owner Admin
rmon event 1102 log-trap SecLabComm description "Chassis_Temperature_Normal" owner Admin

rmon alarm 1101 .1.3.6.1.4.1.9.9.13.1.3.1.3.1 60 absolute rising-threshold 55 1101 falling-threshold 40 1102 owner Admin

snmp-server community SecLabComm RO
snmp-server enable traps rmon
snmp-server host 192.168.10.100 version 2c SecLabComm
```

#### 検証方法
```bash
show rmon alarms
```


## ❓ 想定試験問題

CCIE Security ラボ試験および記述試験を想定した実践的な問題集です。

---

### 問題 1 (コンフィグ読解・トラブルシュート)

以下のコンフィグレーションが設定されているルータ R1 において、ギガビットイーサネット Gi0/1 の入力エラー数が 10 秒間で 50 エラー発生したにもかかわらず、NMS (`10.1.1.50`) に SNMP Trap が届きませんでした。

```bash
rmon event 1 log-trap NmsGroup description "Error_High"
rmon alarm 1 .1.3.6.1.2.1.2.2.1.14.2 10 absolute rising-threshold 20 1 falling-threshold 5 1

snmp-server community NmsGroup RO
snmp-server host 10.1.1.50 version 2c NmsGroup
```

この問題の主な技術的原因を 2 つ指摘し、正しい修正コンフィグを記述しなさい。

#### 解答・解説
**不具合の原因**:
1. **`sample-type` の指定ミス**: カウンタ変数である `ifInErrors` (.1.3.6.1.2.1.2.2.1.14.2) に対し、`absolute` (絶対値) が指定されています。累積カウンタに対して `absolute` を指定すると、累積エラー数が一度でも 20 を超えた後は常に閾値超過状態となり、増加率に応じたイベントが正しく発生しません。`delta` (差分値) を指定する必要があります。
2. **`snmp-server enable traps rmon` の欠落**: グローバルコンフィグで RMON Trap の送信許可コマンドが設定されていません。

**修正コンフィグ**:
```bash
no rmon alarm 1

! サンプルタイプを delta に変更
rmon alarm 1 .1.3.6.1.2.1.2.2.1.14.2 10 delta rising-threshold 20 1 falling-threshold 5 1

! RMON Trap 送信の有効化
snmp-server enable traps rmon
```

---

### 問題 2 (設計・ヒステリシス構造)

RMON のヒステリシス (Hysteresis) 原理に関する記述として、最も適切なものを 1 つ選びなさい。

A. 上昇閾値 (Rising Threshold) を超過して Rising Event が発動した後、値が上昇閾値を下回ればすぐに次の Rising Event が発動可能状態になる。  
B. 上昇閾値を超過して Rising Event が発動した後、値が下落して「下降閾値 (Falling Threshold)」を一度下回るまで、新たな Rising Event は発動しない。  
C. 下降閾値を下回った場合、イベント指定の有無にかかわらず常にシステムが自動再起動する。  
D. サンプリング間隔ごとに常に Rising Event と Falling Event が交互に発動する。  

#### 解答・解説
**正解: B**  
**解説**: RMON は閾値境界付近でのアラート乱発（チャタリング）を防ぐためヒステリシス構造を採用しています。Rising Event が発動した後、次の Rising Event が発動可能になるためには、値が一度 Falling Threshold を下回り、Alarm 状態がリセットされる必要があります。

---

### 問題 3 (実装・EEM 連携)

ネットワーク機器内部の 5分平均 CPU 使用率が 90% を超えた際、RMON Alarm 経由で Syslog ログを発生させ、その Syslog をトリガーとして EEM (Embedded Event Manager) によりトラブルシュート用のデバッグログ収集コマンド (`show process cpu sorted`) を自動実行する設定を作成しなさい。

#### 解答・解説
**設定コンフィグ**:
```bash
! 1. RMON Event および Alarm の定義
rmon event 900 log description "CPU_Critical_Alert" owner Admin
rmon alarm 900 .1.3.6.1.4.1.9.9.109.1.1.1.1.8.1 15 absolute rising-threshold 90 900 falling-threshold 60 900 owner Admin

! 2. EEM Applet の定義
event manager applet CAPTURE_CPU_DEBUG
 event syslog pattern "RMON-5-RISING_THRESHOLD.*alarm 900"
 action 1.0 cli command "enable"
 action 2.0 cli command "show process cpu sorted | redirect flash:cpu_spike_debug.txt"
 action 3.0 syslog msg "EEM: High CPU detected by RMON. CPU process list saved to flash:cpu_spike_debug.txt"
```

---

### 問題 4 (トラブルシューティング・OID 構文)

あるエンジニアが CPU 使用率を監視するために `rmon alarm 5 .1.3.6.1.4.1.9.9.109.1.1.1.1.8 30 absolute rising-threshold 70 5 falling-threshold 40 5` を設定しましたが、`show rmon alarms` を実行しても `Current Value` が常に `0` のままでアラームが発動しません。考えられる原因を説明しなさい。

#### 解答・解説
**原因**:
OID の末尾にインスタンス ID である `.1` (またはスカラーオブジェクトの `.0`) が不足しています。`cpmCPUTotal5minRev` (`.1.3.6.1.4.1.9.9.109.1.1.1.1.8`) はテーブルの列オブジェクトを指しており、個別の CPU コア/プロセッサを指定するためにはインスタンス指定 (例: `.1.3.6.1.4.1.9.9.109.1.1.1.1.8.1`) が必須です。

---

### 問題 5 (デザイン・比較)

RMON (Remote Network Monitoring) と SNMP Polling の比較において、RMON を採用すべき最も適切なシナリオはどれか。

A. WAN リンク越しの拠点ルータにおいて、回線帯域を圧迫せずにトラフィック急増や CPU 負荷の異常発生をリアルタイム検知したい場合。  
B. NMS 側で全ポートのトラフィックグラフを毎分リアルタイム表示し続けたい場合。  
C. 機器のシリアル番号や IOS バージョンなどの静的インベントリ情報を収集したい場合。  
D. ネットワーク機器の初期セットアップや IP アドレス割り当てを行う場合。  

#### 解答・解説
**正解: A**  
**解説**: RMON は機器ローカルで閾値判定を行うため、NMS からの定常的な Polling 通信が発生しません。低帯域の WAN リンクや遅延の大きい拠点の監視において、帯域を消費せずに異常発生時のみ通知を受ける目的で最適です。


## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術資料の一覧です。

* Cisco IOS Network Management Configuration Guide: Monitoring Network Performance with RMON
* Cisco Technical Support & Documentation: How to Configure RMON Alarms and Events
* RFC 2819: Remote Network Monitoring Management Information Base (RMON1)
* RFC 4502: Remote Network Monitoring Management Information Base Version 2 (RMON2)
* Cisco MIB Locator & Command Reference


---

## 📝 **補足（Notes）**

* **ヒステリシス閾値の設定テクニック**:
  * ネットワークの定常状態における MIB 値の「ベースライン」を測定しておくことが重要です。例えば、定常 CPU 使用率が 30% の環境で Rising=40% / Falling=35% に設定すると、正常なゆらぎでアラームが乱発します。
* **EEM との連携による自動修復**:
  * RMON と EEM を組み合わせることで、「セキュリティ攻撃による特定のインターフェイスへのパケット殺到」を RMON で検知し、EEM で即座に ACL を適用したり、ポートを `shutdown` させたりするセキュリティ自動応答メカニズムを構築できます。
* **OID の確認コマンド**:
  * Cisco IOS 機器では `show snmp mib` コマンドで内部でサポートされている MIB オブジェクトとその OID 番号を確認することができます。
