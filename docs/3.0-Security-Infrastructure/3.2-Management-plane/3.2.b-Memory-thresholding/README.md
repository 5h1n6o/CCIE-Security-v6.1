---
layout: default
title: 3.2.b-Memory-thresholding
nav_order: 2
parent: 3.2-Management-plane
grand_parent: 3.0-Security-Infrastructure
---

# 3.2.b Memory thresholding

## 📘 概要

メモリ閾値管理（Memory Thresholding）は、Cisco IOS/IOS-XE ルーター、Cisco ASA ファイアウォール、および Cisco FTD（Firepower Threat Defense）/ FMC（Firepower Management Center）において、利用可能なシステムメモリ（RAM）が枯渇（Out Of Memory: OOM）することを防止し、コントロールプレーンおよび管理プレーンの安定稼働を維持するための保護メカニズムです。

ネットワークデバイスに予期せぬトラフィックサージ、BGPフルルートの収束遅延、メモリリーク、または DoS / DDoS 攻撃が発生した場合、RAM の空き容量が急激に低下することがあります。メモリが完全に枯渇すると、デバイスのカーネルクラッシュ、意図しない再起動（Kernel Panic / OOM Panic）、あるいは新しい接続の処理不能（Drop）や管理アクセス（SSH / HTTPS / Console）の不能に陥ります。

Memory Thresholding 技術を適切に実装することで、メモリ使用量が設定された閾値（Warning / Critical 等）に達した段階で Syslog 通知、SNMP トラップ送信、自動プロセス制御（低優先度プロセスの制限や新しいセッションの拒否）、および EEM（Embedded Event Manager）スクリプトによる修復アクションを起動させ、システム全体のダウンタイムを未然に回避することが可能になります。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | システムメモリ（RAM）の空き容量または使用率を常時監視し、設定した閾値に応じて Syslog/SNMP 通知、接続制限、または緊急メモリ領域の保護を実行する機能。 |
| **用途** | メモリ枯渇（OOM）によるデバイスの突発的クラッシュの回避、DoS/DDoS 攻撃時のコントロールプレーン保護、メモリリークの早期検知。 |
| **メリット** | メモリ異常発生時に管理者へ即座に通知でき、完全にメモリが枯渇する前に不要なセッションの削除やプロセス制御を実行してシステム応答性を維持できる。 |
| **デメリット** | 閾値が不適切に低く設定されている場合、正常なトラフィックバーストや一時的なテーブル拡張で誤アラートが発生する可能性がある。 |
| **対応機種** | Cisco IOS / IOS-XE ルーター（CSR1000v, ISR4000, Catalyst 8000）、Cisco ASA（5500-X, ASAv）、Cisco FTD / FMC（Secure Firewall 1000/2100/3100/4200/9300, FTDv）。 |
| **制限事項** | 一部のローエンド機器やモデルでは `memory free low-watermark` のカスタマイズ設定に制限がある場合がある。FTD では LINA モニタと Linux/Snort プロセスのメモリ割り当てバランスの考慮が必要。 |
| **設計上の注意点** | IOS-XE では Processor メモリと I/O メモリの両方に閾値を設定すること。FTD では FMC の Health Policy と連携して適切な警告閾値（80%）および深刻閾値（90%）を定義する。 |

---

## 🏗 動作原理

Cisco デバイスのメモリ領域は、オペレーティングシステム（IOS-XE、ASA OS、LINA/Linux）によって管理され、動的にプロセスやパケットバッファに割り当てられます。Memory Thresholding の基本的なアーキテクチャと動作概念は以下の通りです。

```
[ メモリ使用量の推移 ]
 100% |----------------------------------------| クラッシュ / OOM Panic
      |                                        |
  90% |========================================| Critical 閾値 (制限動作 / Drop / EEM 実行)
      |                                        |
  80% |----------------------------------------| Warning 閾値 (Syslog / SNMP Trap 送信)
      |                                        |
   0% |----------------------------------------| 正常運用領域
```

### 1. Cisco IOS / IOS-XE におけるメモリ管理構造
* **Processor Memory**: ルーティングプロトコル（BGP, OSPF）、SSH/TELNET 管理プロセス、EEM、コントロールプレーン全体の動作に使用されるメインメモリ。
* **I/O Memory (Input/Output Memory)**: 物理インターフェイスや仮想インターフェイス間でのパケットバッファリングに使用されるメモリ。
* **Low-Watermark (低水準点)**: 利用可能な空きメモリが指定した閾値（Kbytes）を下回った場合、システムは緊急警告メッセージを発行し、非必須のメモリ割り当て要求を拒否します。
* **Reserved Memory**: システムが完全にメモリ不能状態に陥るのを防ぐため、緊急CLI操作や障害解析ログ生成用に事前に固定確保しておく予約領域。

### 2. Cisco ASA におけるメモリ管理構造
* **Block Memory (1550 / 2048 / 8192 bytes)**: パケットの受信、ステートフル接続テーブル（Session Table）、AnyConnect VPN トンネルバッファ等に割り当てられる固定ブロック構造。
* **Syslog Alerting (Syslog ID: 201011 / 201012)**: メモリ使用量が一定レベルに達すると、Syslog メッセージ（`201011: Memory allocation error` または `201012: Memory threshold exceeded`）を生成。

### 3. Cisco FTD / FMC におけるメモリ管理構造
* **Multi-Process Memory Allocation**: FTD 上では、ファイアウォールエンジン（LINA/ASA Core）、次世代 IPS エンジン（Snort 2 / Snort 3）、およびベース Linux OS が物理 RAM を分割共有しています。
* **FMC Health Policy モニタリング**: FMC は FTD デバイスの RAM 使用率を常時ポーリングし、メモリ使用率が 80%（Warning）および 90%（Critical）を超えた際に Health Alert を発行します。
* **OOM Killer (Out-Of-Memory Killer)**: Linux カーネルレベルでメモリが限界に達した場合、優先度の低いプロセスを自動終了してカーネルクラッシュを阻止します。

---

## ⚙ 動作シーケンス

Cisco デバイスにおいてメモリ閾値検出が発生し、保護・通知アクションが起動するまでの一般的な処理シーケンスは以下の通りです。

```
[ システム運用中 ]
       ↓
[ メモリ使用量の増加 ] (BGP ルート増加 / パケットバースト / メモリリーク)
       ↓
[ 閾値チェック (定期ポーリング/割り当て時) ]
       ↓
   < 空きメモリ < Warning 閾値 ? >
       ├─ YES ──> [ Syslog / SNMP Trap 発行 ] ──> [ 管理者通知 / FMC Health Alert ]
       └─ NO
       ↓
   < 空きメモリ < Critical 閾値 / Low-Watermark ? >
       ├─ YES ──> [ 高負荷プロセスのメモリ割当拒否 ]
       │          [ 新規セッション拒否 / パケットドロップ ]
       │          [ EEM スクリプト起動 (CLI 収集 / サービス再起動) ]
       └─ NO
```

1. **メモリ消費の進行**:
   大規模なルーティング更新、接続テーブルの爆発的増加、あるいは不具合によるメモリリークが発生し、利用可能なフリーメモリが徐々に減少します。
2. **第一段階: Warning 閾値の突破**:
   フリーメモリが Warning 閾値（例: 全体の 20% 以下、または特定の Kbytes 以下）を下回ると、システムは Syslog（`%SYS-2-MALLOCFAIL` や `%RESOURCE-4-LOW` 等）を生成し、SNMP トラップを外部監視サーバへ送信します。
3. **第二段階: Critical 閾値 / Low-Watermark の突破**:
   フリーメモリが Low-Watermark（例: 20,000 KB 以下）に達すると、OS は不要なバッファ割り当てや低優先度プロセスのメモリ要求を明示的に拒否（Malloc Failure）します。
4. **緊急保護と補正アクション**:
   EEM（Embedded Event Manager）が組み込まれている場合、Syslog パターンを検知して自動的に不要なプロセスのデバッグログをオフにしたり、現在のメモリマップ（`show memory allocation terminating` 等）を Flash 内に保存して障害解析用ログを保持します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、Memory Thresholding は単独の機能として設定を求められるだけでなく、**Device Hardening**、**Management Plane Protection**、**Syslog/SNMP 監視**、および **EEM（Embedded Event Manager）** と組み合わせた複合課題として出展されます。

### 1. ラボ試験で頻出の設定要求・ポイント
* **IOS-XE における明示的なメモリ低水準点（Low-Watermark）の定義**:
  * Processor メモリおよび I/O メモリに対して指定された Kbytes の低水準点を設定する。
  * `memory free low-watermark processor <Kbytes>`
  * `memory free low-watermark io <Kbytes>`
* **緊急時用メモリリザーブの設定**:
  * 管理アクセス不能（Lockout）を防ぐため、クリティカルプロセス用に事前にメモリを保留する。
  * `memory reserve <bytes>`
  * `memory reserve critical <bytes>`
* **EEM スクリプトによる自動障害ログ収集アサイン**:
  * メモリ枯渇アラートメッセージ（`%SYS-2-MALLOCFAIL` や `%SYS-4-FREEMEM`）が発生した際に、自動的に `show process memory sorted` や `show memory summary` を実行し、Flash 上のテキストファイルへ出力させる設定。
* **ASA / FTD における接続制限およびメモリ通知設定**:
  * ASA CLI での Threat Detection または Service Policy によるメモリ枯渇攻撃対策。
  * FMC GUI における Health Policy の調整（Memory Usage 閾値の設定と Syslog/Email 通知連携）。

### 2. よくある設定ミス・落とし穴
* **単位の勘違い（Bytes vs Kbytes）**:
  * `memory free low-watermark` コマンドの引数は **Kbytes** 単位ですが、`memory reserve` コマンドの引数は **Bytes** 単位です。単位を誤ると意図しない巨大なメモリが固定確保され、起動直後にメモリ不足に陥る危険があります。
* **閾値が高すぎる（または低すぎる）設定**:
  * 低水準点を高く設定しすぎると、正常なトラフィック運用時にも常に警告ログが出力されたり、新規プロセスの起動が阻害されます。逆に低すぎると警告なしに一気に OOM クラッシュが発生します。
* **EEM イベントの循環発生**:
  * メモリ不足の EEM スクリプト内で重い CLI コマンド（`show tech-support` など）を実行させると、スクリプト自体が多大なメモリを消費してさらにメモリ枯渇を悪化させるトラブルが発生します。

### 3. トラブルシューティングにおける確認ポイント
* `show memory free` / `show memory summary`:
  * 現在の Total, Used, Free メモリ量、および Lowest（過去最低空き容量）を確認する。
* `show processes memory sorted`:
  * メモリを最も消費しているプロセス（BGP Router, IP Input, Web Executive 等）を特定する。
* `show log` / `show logging`:
  * メモリ割り当て違反や低水準点突破の Syslog メッセージが出力されていないか確認する。

---

## 🛠 設定方法

Cisco IOS-XE、Cisco ASA、および Cisco FTD / FMC における Memory Thresholding およびメモリ保護の設定手順です。

### 1. Cisco IOS / IOS-XE (CLI) 設定例

IOS-XE では、空きメモリが指定値を下回った際に警告を発行する `memory free low-watermark` と、クリティカル操作用にメモリ領域を維持する `memory reserve` を設定します。

```bash
! --- 1. フリーメモリ低水準点（Low-Watermark）の設定 ---
! Processor メモリの空き容量が 50MB (51200 KB) を下回ったら警告ログを出力
memory free low-watermark processor 51200

! I/O メモリの空き容量が 20MB (20480 KB) を下回ったら警告ログを出力
memory free low-watermark io 20480

! --- 2. クリティカル用途・システム運用保全用の予約メモリ設定 ---
! システム全体の緊急用予約メモリとして 10MB (10485760 Bytes) を割り当て
memory reserve 10485760

! クリティカルプロセス専用の予約メモリとして 5MB (5242880 Bytes) を割り当て
memory reserve critical 5242880

! 一時的・バースト処理用メモリの予約設定
memory transient reserved 10240

! --- 3. EEM (Embedded Event Manager) によるメモリ監視と自動ログ収集 ---
event manager applet MEMORY_THRESHOLD_ALERT
 event syslog pattern "%SYS-2-MALLOCFAIL|%SYS-4-FREEMEM"
 action 1.0 syslog msg "CRITICAL: Memory threshold reached! Capturing diagnostic state."
 action 2.0 cli command "enable"
 action 3.0 cli command "show memory summary | append flash:memory_dump.txt"
 action 4.0 cli command "show processes memory sorted | append flash:memory_dump.txt"
```

### 2. Cisco ASA (CLI) 設定例

Cisco ASA では、接続数上限（Connection Limits）の設定や、メモリ超過時の Threat Detection 通知、Syslog の調整によってメモリ枯渇を防ぎます。

```bash
! --- 1. ステートフル接続テーブル過多によるメモリ枯渇防止 (Service Policy) ---
access-list CONNS_LIMIT_ACL extended permit ip any any

class-map CONNS_LIMIT_CLASS
 match access-list CONNS_LIMIT_ACL

policy-map global_policy
 class CONNS_LIMIT_CLASS
  ! 最大ハーフオープン接続数を制限し、embryonic 接続によるメモリ消費を阻止
  set connection timeout embryonic 0:01:00 half-closed 0:02:00
  set connection conn-max 100000 embryonic-conn-max 5000

service-policy global_policy global

! --- 2. Threat Detection による DoS メモリ保護 ---
threat-detection basic-threat
threat-detection scanning-threat
threat-detection statistics host

! --- 3. メモリ超過 Syslog メッセージの即時送出レベル有効化 ---
logging enable
logging host inside 192.168.1.100
logging list MEM_ALERTS message 201011
logging list MEM_ALERTS message 201012
logging filter-list MEM_ALERTS trap warnings
```

### 3. Cisco FTD / FMC (GUI) 設定例

FMC において FTD の Memory Threshold モニタリングおよびアラートを有効化・調整する手順です。

1. **FMC GUI ヘルスプロファイルの編集**:
   * FMC GUI にログインし、**System** > **Health** > **Policy** に移動します。
   * 編集対象の Health Policy（例: `Default Health Policy`）を選択またはコピーして作成します。
2. **Memory Usage モジュール項目の設定**:
   * **Memory Usage** モジュールを選択します。
   * **Warning Threshold**: デフォルトの `80%` に設定（必要に応じて変更）。
   * **Critical Threshold**: デフォルトの `90%` に設定。
   * **Status**: `Enabled` に設定します。
3. **ヘルスアラート（Syslog / Email / SNMP）連携**:
   * **System** > **Health** > **Monitor** にて対象の FTD デバイスを選択。
   * **Alert Responses** 設定にて、Memory Warning / Critical 検知時に指定の Syslog サーバまたは SNMP トラップ宛てに通知を送信するようマッピングします。
4. **ポリシーのデプロイ**:
   * 設定後、**Save** をクリックし、対象 FTD デバイスへ Deploy します。

---

## 🔍 検証コマンド

Memory Thresholding およびメモリ使用状態の検証で使用する主要コマンド一覧です。

| 目的 | コマンド | 主な確認項目 |
| :--- | :--- | :--- |
| **IOS-XE メモリ要約表示** | `show memory summary` | Total, Used, Free, Lowest（過去最小空き容量）の確認 |
| **IOS-XE プロセス別メモリ確認** | `show processes memory sorted` | メモリを最も多く消費しているプロセスの特定 |
| **IOS-XE 予約メモリ確認** | `show memory reserve` | 予約済みメモリ容量および利用状況の確認 |
| **IOS-XE 低水準点閾値設定確認** | `show running-config \| include memory` | 設定された `memory free low-watermark` の確認 |
| **IOS-XE アロケーション履歴確認** | `show memory allocation terminating` | 終了したプロセスのメモリ解放状態とリークの追跡 |
| **ASA メモリ使用率確認** | `show memory` | ASA 全体のメモリ使用量とフリー領域の確認 |
| **ASA ブロックバッファ確認** | `show blocks` | 1550/2048/8192 バイトブロックの割り当てと枯渇状態 |
| **FTD (LINA) メモリ確認** | `system support diagnostic-cli` > `show memory` | FTD LINA エンジンのメモリ使用状況の確認 |
| **FTD (Linux) OS メモリ確認** | `expert` > `free -m` または `top` | Linux カーネルおよび Snort プロセスのメモリ消費確認 |

---

## 🚨 トラブルシュート

Memory Thresholding に関連する一般的な障害症状と対処方法です。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **`%SYS-2-MALLOCFAIL` メッセージが頻発する** | RAM 容量の枯渇、大規模テーブル収束、または特定プロセスのメモリリーク。 | <code>show processes memory sorted</code><br><code>show memory summary</code> | メモリ消費量の多いプロセスを特定。不要なデバッグや重いプロトコル設定を解除し、OS バグの場合は修正バージョンへアップグレード。 |
| **`%RESOURCE-4-LOW` ログが出力される** | 設定した `memory free low-watermark` 閾値をフリーメモリが下回った。 | <code>show running-config \| include memory</code><br><code>show memory free</code> | 閾値がトラフィック規模に対して適切か確認。不適切な場合は値を再調整するか、物理 RAM を増設。 |
| **FMC で FTD の Memory Health Alert（Critical 90%）が発生** | Snort エンジンのインスペクションバッファ肥大化、または接続数の急増。 | FMC **Health Monitor** > **Memory Usage**<br>FTD <code>top</code> (expert mode) | FTD の Snort メモリ設定（Stream Inspection メモリ上限等）を調整。不要な接続セッションを無効化。 |
| **EEM スクリプトが自動起動しない** | Syslog パターンマッチの記述ミス、または EEM の実行権限不足。 | <code>show event manager history events</code><br><code>show running-config \| section event manager</code> | Syslog の正規表現パターン（`pattern`）を確認・修正。EEM スクリプト内で `enable` モードへの遷移が含まれているか検証。 |
| **SSH / TELNET ログイン時に `Out of Memory` で拒否される** | CLI 管理プロセス用メモリ領域が枯渇。 | <code>show memory reserve</code> | `memory reserve critical` を設定してクリティカル操作用のメモリ領域を事前に固定確保。 |

---

## ⚠ 制限事項

* **プラットフォーム依存の機能差**:
  * IOS-XE、ASA OS、および FTD（Linux/Snort）では、メモリ割り当て構造とコマンド体系が全く異なります。CLI コマンドの互換性はありません。
* **物理 RAM 容量の制限**:
  * Memory Thresholding は既存メモリの保護と警告通知を行う機能であり、物理的なメモリ不足そのものを根本解決するものではありません。
* **予約メモリのトレードオフ**:
  * `memory reserve` や `memory reserve critical` で大きなメモリ容量を固定確保すると、通常運用時に使用できる一般メモリ容量がその分減少します。

---

## 🔄 他技術との関連

Memory Thresholding は、CCIE Security Blueprint における以下のセキュリティ基盤技術と深く関連しています。

* **CoPP (Control Plane Policing)**:
  * CoPP で管理プレーン/コントロールプレーン宛ての不審なパケット（ICMP, SYN Flood, BGP 制御パケット等）をドロップすることで、CPU とメモリの双方向の枯渇を防止します。
* **Threat Detection (ASA / FTD)**:
  * DoS 攻撃やスキャニング攻撃を検知して自動遮断（Shun）することで、ステートフル接続テーブルの爆発によるメモリ圧迫を阻止します。
* **Embedded Event Manager (EEM)**:
  * メモリ閾値オーバーの Syslog メッセージをトリガーとして、デバッグログの自動停止や診断情報の出力スクリプトを実行します。
* **FMC Health Policy**:
  * ネットワーク全体の FTD / FMC デバイスの RAM 使用率を統合モニタリングし、閾値超過時に管理者へ Syslog / Email 通知を行います。

---

## 🧩 比較表

| 項目 | Cisco IOS-XE | Cisco ASA | Cisco FTD (FMC Managed) |
| :--- | :--- | :--- | :--- |
| **設定方式** | CLI (`memory free low-watermark`) | CLI (Syslog & Threat Detection) | FMC GUI (Health Policy) |
| **保護対象** | Processor RAM / IO RAM | ASA Block Memory / Session Table | LINA Engine / Snort / Linux OS |
| **緊急防御機構** | `memory reserve critical` | TCP Intercept / Connection Limit | Linux OOM Killer / Snort Stream Cap |
| **自動修復連携** | EEM Applet 連携 | Syslog & Shun 連携 | FMC Health Policy & Alert Response |

---

## 💡 ベストプラクティス

1. **適切な低水準点（Low-Watermark）の定義**:
   * IOS-XE デバイスでは、全物理 RAM の 10〜15% 程度に相当する容量を `memory free low-watermark processor` に設定することを推奨します。
2. **クリティカル予約メモリの常時確保**:
   * メモリ枯渇時でも緊急 SSH 接続や障害解析コマンド（`show memory` 等）を確実に実行できるよう、`memory reserve critical` を必ず事前定義しておきます。
3. **FMC Health Policy の二段階閾値運用**:
   * FMC Health Policy で Warning（80%）と Critical（90%）の二段階アラートを設定し、Warning 発生段階で容量調査を開始する運用を徹底します。
4. **EEM による障害エビデンス自動取得**:
   * MALLOCFAIL 発生時に自動的に `show processes memory sorted` をファイル出力する EEM アプレットを組み込み、原因追求の証拠を保持します。

---

## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルの実践的な設定サンプル例 10 選です。

### サンプル例 1: IOS-XE における Processor / IO メモリ低水準点の設定

#### 問題
R1 (IOS-XE) において、RAM 容量の低下を早期検知するため、Processor メモリおよび I/O メモリの低水準点閾値を定義してください。

#### 要件
* Processor メモリの空き容量が 64MB (65536 KB) を下回った場合に警告ログを出力すること。
* I/O メモリの空き容量が 16MB (16384 KB) を下回った場合に警告ログを出力すること。

#### 設定例
```bash
! R1 Configuration
memory free low-watermark processor 65536
memory free low-watermark io 16384
```

#### 検証方法
```bash
R1# show running-config | include memory free
R1# show memory free
```


### サンプル例 2: クリティカル操作・緊急管理用予約メモリの確保

#### 問題
R1 (IOS-XE) において、メモリ枯渇が発生した場合でも管理者が SSH 経由でログインしてトラブルシューティングを実行できるように予約メモリを設定してください。

#### 要件
* システム全体で 16MB (16777216 Bytes) の予約メモリを確保すること。
* クリティカルプロセス用に 8MB (8388608 Bytes) の予約メモリを確保すること。

#### 設定例
```bash
! R1 Configuration
memory reserve 16777216
memory reserve critical 8388608
```

#### 検証方法
```bash
R1# show memory reserve
```

---

### サンプル例 3: EEM を用いた Memory Threshold 発生時の自動診断ログ取得

#### 問題
R1 において、フリーメモリが閾値を下回った際に、自動的にメモリ使用状況を Flash 上の `flash:mem_alert.log` へ保存する EEM スクリプトを作成してください。

#### 要件
* Syslog パターン `%SYS-2-MALLOCFAIL` または `%SYS-4-FREEMEM` を検出条件とすること。
* 発生時に Syslog で `[ALERT] Memory threshold reached!` を出力すること。
* `show memory summary` および `show processes memory sorted` の結果を `flash:mem_alert.log` へ追記出力すること。

#### 設定例
```bash
! R1 Configuration
event manager applet AUTO_MEM_DIAG
 event syslog pattern "%SYS-2-MALLOCFAIL|%SYS-4-FREEMEM"
 action 1.0 syslog msg "[ALERT] Memory threshold reached!"
 action 2.0 cli command "enable"
 action 3.0 cli command "show memory summary | append flash:mem_alert.log"
 action 4.0 cli command "show processes memory sorted | append flash:mem_alert.log"
```

#### 検証方法
```bash
R1# show event manager history events
R1# show flash:mem_alert.log
```

---

### サンプル例 4: ASA における embryonic 接続制限によるメモリ枯渇防止

#### 問題
ASA1 において、TCP SYN 大量接続によるステートフルセッションテーブル肥大化とメモリ枯渇を防ぐため、接続上限ポリシーを適用してください。

#### 要件
* クライアントからの同時最大接続数を 50,000 に制限すること。
* ハーフオープン（embryonic）接続数を最大 2,000 に制限し、タイムアウトを 30 秒に設定すること。

#### 設定例
```bash
! ASA1 Configuration
access-list EMBRYONIC_PROTECT_ACL extended permit ip any any

class-map EMBRYONIC_CLASS
 match access-list EMBRYONIC_PROTECT_ACL

policy-map global_policy
 class EMBRYONIC_CLASS
  set connection conn-max 50000 embryonic-conn-max 2000
  set connection timeout embryonic 0:00:30

service-policy global_policy global
```

#### 検証方法
```bash
ASA1# show service-policy global inspect
ASA1# show local-host
```

---

### サンプル例 5: ASA メモリ超過 Syslog メッセージの専用ログトラップ定義

#### 問題
ASA1 において、メモリ割り当てエラー（Syslog ID 201011, 201012）を検知した際に、外部 Syslog サーバへ即座に警告ログを送信するように設定してください。

#### 要件
* Syslog サーバ IP: `10.1.1.250`（inside インターフェイス経由）。
* メモリ関連 Syslog ID `201011` および `201012` のみを集約した Custom Logging List を作成し送信すること。

#### 設定例
```bash
! ASA1 Configuration
logging enable
logging host inside 10.1.1.250

logging list MEM_CRIT_LIST message 201011
logging list MEM_CRIT_LIST message 201012

logging filter-list MEM_CRIT_LIST trap warnings
```

#### 検証方法
```bash
ASA1# show logging
ASA1# show logging queue
```


---

### サンプル例 6: FTD / FMC における Memory Health Policy 閾値設定

#### 問題
FMC において、管理対象の FTD1 デバイスのメモリ使用率モニタリング閾値をカスタマイズしてください。

#### 要件
* Warning 閾値を `75%` に設定すること。
* Critical 閾値を `85%` に設定すること。
* 閾値を超えた場合に Syslog サーバ `10.1.1.200` へアラートを送信すること。

#### 設定手順（FMC GUI）
1. **System** > **Health** > **Policy** へ移動し、ポリシーを編集。
2. **Memory Usage** モジュールを選択。
3. **Warning Threshold** を `75%`、**Critical Threshold** を `85%` に変更。
4. **Alert Responses** 設定にて Syslog アラート応答（Server: `10.1.1.200`）を紐付け。
5. 設定を保存し、FTD1 へ Deploy。

#### 検証方法
* FMC GUI **System** > **Health** > **Monitor** にて FTD1 の Memory Status を確認。

---

### サンプル例 7: IOS-XE における BGP メモリ枯渇対策（BGP Route Limit）

#### 問題
R1 (IOS-XE) において、対向 ISP から過剰な BGP プレフィックスを受信して RAM が枯渇するのを防ぐため、受信用最大プレフィックス制限を設定してください。

#### 要件
* BGP ピア `192.168.12.2` からの受信プレフィックス数を最大 10,000 に制限すること。
* 80%（8,000 プレフィックス）に達した時点で警告ログを出力すること。

#### 設定例
```bash
! R1 Configuration
router bgp 65001
 neighbor 192.168.12.2 remote-as 65002
 neighbor 192.168.12.2 maximum-prefix 10000 80
```

#### 検証方法
```bash
R1# show ip bgp neighbors 192.168.12.2 | include Prefix
```

---

### サンプル例 8: IOS-XE における Transient メモリ予約とバッファ保護

#### 問題
R1 において、急激なトラフィックバースト時の短時間メモリ割り当て要求に応答できるよう、Transient メモリ領域を確保してください。

#### 要件
* Transient メモリ領域として 20MB (20480 KB) を予約すること。

#### 設定例
```bash
! R1 Configuration
memory transient reserved 20480
```

#### 検証方法
```bash
R1# show running-config | include memory transient
```

---

### サンプル例 9: FTD エキスパートモードにおける Linux OS メモリ状態の調査

#### 問題
FTD1 の CLI エキスパートモードを使用して、システム全体のメモリ使用状況およびプロセスメモリ分布を分析してください。

#### 要件
* Linux カーネルの `free -m` コマンドで物理 RAM、スワップメモリの消費量を確認すること。

#### 設定・検証手順
```bash
# FTD CLI にログイン
> expert
$ free -m
$ top -b -n 1 | head -n 20
```

#### 検証方法
* FTD Linux エキスパートモードにて `free -m` の結果を確認。

---

### サンプル例 10: EEM と SNMP トラップを組み合わせた高度なメモリ緊急通知

#### 問題
R1 において、フリーメモリが閾値を下回った際に SNMP トラップを送信する EEM アプレットを構成してください。

#### 要件
* `%SYS-4-FREEMEM` の出力時に、Syslog 通知と同時に SNMP トラップを発行すること。

#### 設定例
```bash
! R1 Configuration
snmp-server enable traps syslog
snmp-server host 10.1.1.100 community public syslog

event manager applet MEM_SNMP_ALERT
 event syslog pattern "%SYS-4-FREEMEM"
 action 1.0 syslog msg "SNMP Event: Free memory dropped below threshold!"
 action 2.0 snmp-trap strdata "CRITICAL_MEMORY_LOW"
```

#### 検証方法
```bash
R1# show event manager history events
```


---

## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験を想定した試験問題と詳細解説です。

### 質問 1 (コンフィグ読解)
以下の IOS-XE コンフィグが設定されたルーターの動作として**正しいもの**を 1 つ選んでください。

```bash
memory free low-watermark processor 40960
memory reserve 10485760
```

* A) Processor メモリの空き容量が 10,485,760 KB を下回ると Syslog ログが出力される。
* B) Processor メモリの空き容量が 40,960 KB (約 40MB) を下回ると警告が出力され、緊急用に 10,485,760 Bytes (10MB) が確保される。
* C) フリーメモリが 40,960 Bytes を下回ると自動的にルーターが再起動する。
* D) `memory reserve` コマンドの単位は Kbytes であるため、10GB のメモリが予約される。

**正解**: **B**
**解説**: `memory free low-watermark processor 40960` の引数は Kbytes 単位（40,960 KB = 40MB）であり、この値を下回ると警告が発行されます。一方、`memory reserve` の引数は Bytes 単位（10,485,760 Bytes = 10MB）であり、緊急時の管理操作用に固定確保されます。

---

### 質問 2 (トラブルシューティング)
Cisco ASA において、大量の不審な TCP 接続試行が発生し、ステートフル接続テーブルが拡大して RAM が枯渇し始めました。接続テーブルによるメモリ枯渇を防止するための最善の初期対応はどれですか？

* A) `memory free low-watermark` コマンドを ASA CLI で実行する。
* B) Service Policy において `set connection embryonic-conn-max` を設定し、ハーフオープン接続数を制限する。
* C) `no logging enable` を実行して Syslog プロセスを停止する。
* D) ASA をマルチコンテキストモードに変換する。

**正解**: **B**
**解説**: ASA において SYN 大量接続によるメモリ枯渇を防ぐには、Service Policy で `set connection embryonic-conn-max`（および `conn-max`）を設定してハーフオープン接続数を制限するのが最も効果的です。


---

### 質問 3 (Design / 運用)
Cisco FTD / FMC 環境において、FTD デバイスのメモリ使用率が 90% を超えた際のアクションとして正しいものはどれですか？

* A) FTD は自動的に Transparent Mode へ移行する。
* B) FMC Health Policy が Critical アラートを発行し、プロセスメモリ消費を抑止する。
* C) FTD はすべての Access Control Policy を変更する。
* D) FTD は全インターフェイスを無効化する。

**正解**: **B**
**解説**: FTD では RAM 使用率が 90%（Critical 閾値）を超えると FMC Health Alert が発生します。

---

### 質問 4 (実装)
IOS-XE ルーターにおいて、メモリ割り当て違反（`%SYS-2-MALLOCFAIL`）が発生した際に、自動的に診断コマンドを実行してログを Flash 内に書き込むための正しい EEM 設定はどれですか？

* A)
```bash
event manager applet MEM_LOG
 event syslog pattern "%SYS-2-MALLOCFAIL"
 action 1.0 cli command "show processes memory sorted | append flash:mem.txt"
```
* B)
```bash
event manager applet MEM_LOG
 event syslog pattern "%SYS-2-MALLOCFAIL"
 action 1.0 cli command "show memory summary"
```

**正解**: **A**
**解説**: EEM アプレットで `event syslog pattern "%SYS-2-MALLOCFAIL"` を指定し、`action` で `cli command "show processes memory sorted | append flash:mem.txt"` を呼び出します。


---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術資料へのリンク一覧です。

* [Cisco IOS-XE Memory Management Configuration Guide](https://www.cisco.com)
* [Cisco Command Reference: memory free low-watermark Command](https://www.cisco.com)
* [Cisco ASA Configuration Guide: Managing System Memory and Connections](https://www.cisco.com)
* [Cisco Secure Firewall Management Center Configuration Guide](https://www.cisco.com)
* [Cisco Live Technical Resources](https://www.ciscolive.com)

---

## 📝 補足（Notes）

* **メモリ使用量監視の定期自動化**:
  * 実務およびラボ環境において、`show memory summary` の出力を定期的に取得しておくことで、メモリリーク傾向（時間経過とともに Used が増え続け Free が戻らない状態）を早期に把握できます。
* **Cisco EEM との連携による自動復旧**:
  * メモリ不足の根本原因が特定のデバッグログ出力や未解放セッションにある場合、EEM アプレットでデバッグログの無効化やセッションクリアを自動実行させることで、管理者の手動介入なしに危機を回避可能です。
