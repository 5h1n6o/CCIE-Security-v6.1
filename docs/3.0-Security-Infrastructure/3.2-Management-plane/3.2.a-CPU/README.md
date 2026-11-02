---
layout: default
title: 3.2.a-CPU
nav_order: 1
parent: 3.2-Management-plane
grand_parent: 3.0-Security-Infrastructure
---

# 3.2.a CPU

Cisco のネットワークインフラストラクチャおよびセキュリティアプライアンス（Cisco IOS/IOS-XE, Cisco ASA, Cisco FTD）において、**CPU（Central Processing Unit）** はコントロールプレーンのルーティング計算、セッション状態の管理、暗号化処理、シグナリング、およびデータプレーンでの例外パケット処理（Process Switching / Slow Path）を司る核心的コンポーネントです。

CCIE Security v6.1 Blueprint **3.2.a CPU** では、各プラットフォームにおける CPU アーキテクチャの相違（IOS/IOS-XE の Run-to-Completion & Quantum Scheduling、ASA の LINA マルチスレッドアーキテクチャ、FTD の Snort 2/3 ＋ LINA デュアルエンジン構造）、CPU 高負荷（High CPU Utilization）を引き起こす根本要因の特定手法、プロセス CPU（Process-level CPU）と割り込み CPU（Interrupt-level CPU）の挙動分析、CPU 保護メカニズム（Process Throttling, Rate Limiting, CoPP/CPPr, Threat Detection）、およびトラブルシューティング手順が多角的に評価されます。

---

## 📘 概要

* **機能概要**:
  * **CPU Protection & Hardening**: 不審なトラフィック暴走や Dos 攻撃、構成ミスによる制御パケット（ARP, ICMP, Routing, Management）の急増から CPU 資源を保護し、システム全体のクラッシュやフェールオーバー誤動作、ルーティングネイバーのダウンを防止する一連のハードニング技術。
  * **Cisco IOS/IOS-XE CPU Architecture**:
    * **Process-level CPU**: IOS プロセス（BGP, OSPF, Exec, IP Input 等）がコールドスタートや計算処理を行う際に消費する CPU。優先度ベース（Critical, High, Medium, Low）でスケジュールされる。
    * **Interrupt-level CPU**: ネットワークインターフェイスからの物理的なパケット受信（Rx/Tx DMA 割り込み）やハードウェアタイマー処理によって発生する割り込み。Process-level よりも優先度が高く、これが急増すると全プロセスがブロックされる。
  * **Cisco ASA CPU Architecture**:
    * **LINA (Firewall Core)**: 単一のモノリシックプロセスコアとして動作し、データパス（FastPath）とコントロールパスを統合スレッドモデルで処理。
    * **ASP (Accelerated Security Path)**: パケットを高速転送するハードウェア/ソフトウェアエンジン。NP (Network Processor) や FPGA と直接連携。
  * **Cisco FTD CPU Architecture**:
    * **Data Plane Process (LINA)**: L2-L4 インスペクション、NAT、ルーティング、VPN 処理を実行。
    * **Snort Engine (Snort 2 / Snort 3)**: L7 アプリケーション識別（AVC）、IPS、AMP、URL フィルタリングを実行するマルチスレッドプロセス。CPU コア数が LINA と Snort 間で動的/固定的に割り当てられる。
* **利用目的**:
  * コントロールプレーン（BGP, OSPF, EIGRP, LDP, IKEv2 等）の応答性を維持し、ネットワークのフラッピングを防ぐ。
  * 管理アクセス（SSH, HTTPS, SNMP, TACACS+）の応答性を確保し、緊急メンテナンス時のロックアウトを回避する。
  * 不審なトラフィック（ARP スプーミング、SYN フラッド、ICMP 爆撃、IP Option パケット）による CPU 枯渇を抑制する。
* **どのような場面で利用するか**:
  * 大規模データセンターや ISP エッジでの DDo S 攻撃対策およびリソース枯渇防止。
  * 高負荷な IPS 検査や SSL/TLS 復号を実行する FTD/ASA アプライアンスでのマルチコア最適化。
  * ラボ試験および実務における High CPU トラブルシューティング（原因プロセスの特定と動的サプレッション）。

---

## 🔑 要点

| 項目 | Cisco IOS / IOS-XE | Cisco ASA | Cisco FTD (FMC Managed) |
| :--- | :--- | :--- | :--- |
| **アーキテクチャ** | Process/Interrupt 割り込みモデル (Quantum) | LINA モノリシック ＋ 内部スレッド | FXOS / Linux OS 上で LINA ＋ Snort ディアル動作 |
| **主な CPU 消費要素** | Process Switching (CEF Miss), BGP/OSPF 計算, IP Input | ASP Rx/Tx, NP ハンドオフ, Deep Packet Inspection | Snort インスペクションスレッド, LINA データパス, FMC エージェント |
| **CPU 監視コマンド** | `show processes cpu sorted`, `show platform software process slot` | `show cpu`, `show process cpu-usage sorted`, `show asp table` | `top`, `system support diagnostic-cli` (`show cpu`), `cpu-hog` ログ |
| **保護・閾値メカニズム** | `process cpu-threshold`, CoPP, CPPr, `scheduler allocate` | Threat Detection (Basic/Advanced), `cpu queue` | Threat Detection, Snort Elephant Flow Bypass, Rate Limiting |
| **制限事項** | 割り込み CPU は CoPP/iACL なしではソフトウェア制御不可能 | LINA スレッドが 100% に達すると管理 GUI (ASDM) も応答停止 | Snort CPU 高騰時は Passive/Drop モード選択が必要 |
| **設計上の注意点** | CEF が有効であることを必ず確認 (CEF Disable は即死原因) | ASP Drop Counter を定期追跡し不審パケットを特定 | 物理コア数に応じた Snort インスタンス数の最適配分 |

---

## 🏗 動作原理

Cisco 機器における CPU 処理アーキテクチャおよび制御フローは、コントロールプレーンとデータプレーンの分離構造に依存します。

### 1. Cisco IOS/IOS-XE パケット処理フロー

```
Ingress Interface (Rx DMA)
       ↓
[Interrupt Handling] ── (CEF Hit) ──→ Egress Interface (Hardware/FastPath)
       ↓ (CEF Miss / Glean / Local IP / IP Options)
[IP Input Process Queue]
       ↓
[Process-level Scheduler] (Quantum-based Priority Allocation)
       ↓
[Control Plane / Management Process] (BGP, OSPF, SSH, etc.)
```

1. **Hardware FastPath (CEF)**: パケットがインターフェイスに到着すると、CEF (Cisco Express Forwarding) テーブルが参照されます。ヒューリスティックにマッチした標準的な通過トラフィックは CPU (Process-level) を介入させずに ASIC/NPU で転送されます。
2. **CEF Miss / Exception**: パケットが自機器宛て（Local IP）、ARP 未解決（Glean）、IP Option 付与、または TTL 1 の場合、割り込み処理が発生し、パケットは **IP Input** プロセスバッファへキューイングされます。
3. **Process Scheduler**: CPU スケジューラはタスクの優先度に従って CPU 時間（Quantum）を割り当てます。IP Input や BGP などの重要プロセスが優先されますが、IP Input にパケットが溢れると BGP 等のプロセスが CPU を獲得できなくなります。

### 2. Cisco ASA / FTD 処理フロー

```
Ingress Packet
       ↓
[ASP Rx Handling] (NPU / NIC Ring Buffer)
       ↓
[LINA Thread Dispatcher]
       ├── FastPath (Existing Session) ──→ Egress
       └── SlowPath (New Session / Inspection Required)
              ↓
       [Snort Engine] (FTD の場合: DAQ 経由で共有メモリ転送)
              ↓
       [Security Policy / IPS Rule Match]
              ↓
       [LINA Egress Processing] ──→ Egress Interface
```

* **LINA スレッド構造**: ASA/FTD の LINA プロセスはマルチスレッド化されており、ポーリングスレッド（Polling Thread）が NIC バッファを常時チェックします。トラフィック量が増加するとポーリングが CPU を 100% 占有する設計となっていますが、これは「ビジーウェイト（Busy Wait）」による正常動作の場合もあります。
* **Snort DAQ (Data Acquisition Layer)**: FTD では LINA と Snort エンジン間を DAQ を介して共有メモリでパケット受け渡しを行います。特定フロー（Elephant Flow）が巨大な場合、1 つの Snort スレッドが CPU コアを 100% 飽和させる現象が発生します。

---

## ⚙ 動作シーケンス

パケット受信から CPU 処理・サプレッションに至るシーケンスは以下のステップで実行されます。

```
1. 物理パケット受信 (NIC Ring Buffer)
       ↓
2. 割り込み発起 (Interrupt Level CPU 上昇)
       ↓
3. CEF Lookup / ASP Match
       ├── [CEF Hit] → データプレーン転送 (Process CPU 消費なし)
       └── [CEF Miss / Exception / Local Destination]
              ↓
4. Rate Limiting / Threat Detection / CoPP 評価
       ├── [閾値超え (Over-Limit)] → HW/SW Drop (CPU 負荷抑制)
       └── [許容範囲内 (Permitted)]
              ↓
5. IP Input Queue / Packet Ring にバッファリング
       ↓
6. プロセススケジューラによる CPU タスク割当
       ↓
7. 対象プロセス (BGP/OSPF/SSH) によるパケット処理
```

---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security ラボ試験において、CPU 保護およびトラブルシューティングに関して問われる重要ポイントは以下の通りです。

### 1. Blueprintで重要なポイント

* **CPU 高負荷の一次切り分け**:
  * `Interrupt` による高負荷か、特定 `Process` による高負荷かの判別。
  * `show processes cpu sorted` の 1 行目: `CPU utilization for five seconds: 98%/85%; one minute: 95%; five minutes: 90%` の場合、全 98% のうち **85% が Interrupt（割り込み）** で消費されていることを示します。この場合、Process-level の最適化ではなく、CoPP/iACL や CEF の状態確認（CEF が Disable になっていないか）が正解となります。
* **IOS/IOS-XE の Scheduler 調整**:
  * `scheduler allocate <4000> <500>`: プロセス処理（4000 マイクロ秒）と割り込み処理（500 マイクロ秒）の CPU 時間配分を調整し、割り込み暴走時にもプロセスが最低限の CPU 時間を獲得できるように設定します。
* **ASA/FTD Threat Detection による CPU 保護**:
  * Scanning Threat, Rate Threat, Basic Threat Detection を有効化し、攻撃ホストを動的に `shun` (ドロップ) して CPU 負荷を軽減します。
* **FTD Snort CPU 高負荷対策**:
  * Fastpath (Trust Rule) や Pre-filter Rule を使用して、バックアップ通信や大容量データ転送（Elephant Flow）を Snort の L7 検査からバイパスさせます。

### 2. ラボ試験で設定させられそうな内容

* 指定された CPU 閾値（例: 80% 超えが 5 秒継続）で Syslog および SNMP トラップを発行するコンフィグ作成。
* 割り込み処理による CPU 枯渇を防ぐための CoPP ルール（ARP, ICMP, Routing 制限）の構築。
* ASA/FTD における CPU 保護機能（Threat Detection, Concurrent Connection Limits）の構築。
* 特定プロセス（例: Exec, IP Input）が CPU を占有した際の処理制限設定。

### 3. よくある設定ミス

* **CEF の誤無効化**: インターフェイス単位で `no ip route-cache` や `no ip cef` を設定してしまい、全パケットが Process Switching（IP Input）へフォールバックして CPU が 100% に張り付く。
* **Logging デバッグの有効化**: `logging console debug` や `terminal monitor` を有効にしたまま大容量トラフィックを流し、`Console Output` プロセスが CPU を 100% 占有する。
* **不適切な CoPP Rate**: 正常なルーティングプロトコル（BGP/OSPF）のレートを低く設定しすぎてネイバーがダウンする。

---

## 🛠 設定方法

Cisco 各プラットフォームにおける CPU 保護・閾値監視・スケジューリング調整の実装例です。

### 1. Cisco IOS / IOS-XE における CPU 監視とスケジューリング調整

#### CPU 閾値通知（Syslog & SNMP Trap）設定
CPU 利用率が一定の閾値を超えた場合に Syslog および SNMP トラップを発行する設定です。

```bash
! CPU 閾値イベントの定義 (5秒平均が 85% を超えたら Rising、60% 未満に落ちたら Falling)
process cpu-threshold type total rising 85 interval 5 falling 60 interval 5

! 特定プロセスが CPU を占有した場合の通知
process cpu-threshold type process rising 70 interval 5 falling 40 interval 5

! SNMP トラップの有効化
snmp-server enable traps cpu threshold
snmp-server host 192.168.100.50 public cpu
```

#### プロセススケジューリング調整（Scheduler Allocate / Interval）
割り込み処理（Interrupt）が多発した際にもコントロールプレーンプロセスが CPU 実行時間を確保できるように設定します。

```bash
! 割り込み処理に割り当てる最大時間を 5000 マイクロ秒、プロセス処理に最低 1000 マイクロ秒を確保
scheduler allocate 5000 1000

! プロセススケジューラのタイムスライス間隔を調整
scheduler interval 500
```

---

### 2. Cisco ASA における CPU 保護設定

ASA では Threat Detection やコネクション制限を用いて、CPU を消費する攻撃トラフィックを早期ドロップします。

```bash
! 基本的な Threat Detection の有効化
threat-detection basic-threat

! 走査攻撃 (Scanning) およびレート超過攻撃の自動ドロップ・Shun 有効化
threat-detection scanning-threat threat-level high action-ge-level high
threat-detection rate host-scan-rate 200 dump 600
threat-detection rate-interval 600 average-rate 1000 burst-rate 3000

! CPU 負荷を抑えるための最大アクティブコネクション数の制限
policy-map global_policy
 class class-default
  set connection timeout embryonic 0:00:30 half-closed 0:10:00
  set connection conn-max 10000 embryonic-conn-max 2000
```

---

### 3. Cisco FTD (FMC Managed) における CPU 保護設定

FMC を介して FTD の CPU 負荷を抑制する設定戦略です。

1. **Pre-filter Rule (L3/L4 Fastpath)**:
   * **設定箇所**: `Policies` > `Access Control` > `Pre-filter`
   * **動作**: 大容量のバックアップ通信や信頼されたインサイド通信に対し `Fastpath` アクションを指定。Snort エンジンの CPU インスペクションをバイパスさせる。
2. **Snort Elephant Flow Bypass**:
   * **設定箇所**: `FlexConfig` または FMC プラットフォーム設定。
   * 大容量単一フロー（例: 10Gbps データ転送）が単一 Snort コアを飽和させるのを防止。

---

## 🔍 検証コマンド

| 目的 | Cisco IOS / IOS-XE | Cisco ASA | Cisco FTD (CLI / Diagnostic) |
| :--- | :--- | :--- | :--- |
| **CPU 利用率サマリー確認** | <code>show processes cpu</code> | <code>show cpu</code> | <code>show cpu</code> / <code>top</code> |
| **高 CPU 消費プロセスのソート表示** | <code>show processes cpu sorted</code> | <code>show process cpu-usage sorted</code> | <code>system support diagnostic-cli</code> 経由で <code>show process cpu-usage sorted</code> |
| **特定プロセスの詳細確認** | <code>show processes cpu detailed</code> | <code>show process <process_id></code> | <code>top -H -p <pid></code> |
| **割り込み・ASIC 統計確認** | <code>show platform software status control-processor</code> | <code>show asp table</code> / <code>show asp drop</code> | <code>show asp drop</code> / <code>show sftarget-statistics</code> |
| **CPU 閾値設定・履歴確認** | <code>show process cpu-threshold</code> | <code>show threat-detection rate</code> | <code>show threat-detection</code> |
| **デバッグ出力制御 (CPU保護)** | <code>undebug all</code> / <code>no logging console</code> | <code>no logging console</code> | <code>system support trace</code> のフィルタ適用 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **IOS で CPU 100% (Interrupt 割合が極めて高い)** | CEF が無効化されているか、大量のパケットが Process Switching されている | <code>show processes cpu sorted</code><br><code>show ip cef</code><br><code>show interfaces switching</code> | `ip cef` を全グローバル・インターフェイスで有効化。`no ip route-cache` を除去。 |
| **IOS で `IP Input` プロセスが CPU を占有** | ARP スプーミング、大量の ICMP/IP Options、Glean CEF エントリ多発 | <code>show processes cpu sorted</code><br><code>show ip traffic</code><br><code>show buffers input-interface <int> failures</code> | CoPP を導入して自機器宛て不審パケットを警察・制限。`ip options drop` を設定。 |
| **IOS で `Exec` または `Console Output` が CPU 占有** | `logging console debug` の有効化や、SSH/Console 経由での大量テキスト出力 | <code>show processes cpu sorted \| include Exec</code><br><code>show logging</code> | `no logging console` を設定。`logging buffered` の利用に変更。 |
| **ASA で `DATAPATH` スレッドが 100%** | トラフィック高負荷、またはポーリングスレッドのビジーウェイト (正常な動作の場合あり) | <code>show cpu</code><br><code>show cpu usage datapath</code><br><code>show asp drop</code> | `show asp drop` で非正常なドロップを確認。Threat Detection で不審トラフィックをサプレッション。 |
| **FTD で Snort プロセスが 100% 占有** | Elephant Flow (単一の大容量TCP/UDPフロー) が単一 Snort コアに集中 | <code>top</code> (Expert モード)<br><code>show conn detail</code> | Pre-filter Rule で当該フローを `Fastpath` 化し、Snort のインスペクションを回避。 |

---

## ⚠ 制限事項

* **IOS-XE Quantum Scheduling**: IOS-XE のコントロールプレーン（IOSd）は Linux 仮想プロセス（Container/VM）として動作するため、IOSd 内部の CPU 表示と Linux OS (`show platform software process`) の CPU 表示が異なる場合がある。
* **ASA Single-core vs Multi-core**: ASA 5505/5510 などの単一コアモデルと、5585-X / Firepower 9300 などのマルチコアモデルでは、`show cpu` のマルチスレッド表示結果の解釈（Core ごとの平滑化）が異なる。
* **FTD Snort 2 vs Snort 3**: Snort 2 はマルチプロセスモデル（Snort インスタンスがコアごとに独立）、Snort 3 はマルチスレッドモデル（単一 Snort3 プロセス内に複数スレッド）。CPU 割り当てメカニズムおよび `top` コマンドでの表示形式が異なる。

---

## 🔄 他技術との関連

* **CEF (Cisco Express Forwarding)**: CPU 保護の第一防衛線。CEF が無効になると全トラフィックが Process Switching となり CPU が枯渇する。
* **CoPP (Control Plane Policing)**: CPU 直前のソフトウェアポリサー。不審なコントロールプレーンパケットをドロップ/シェーピングして CPU を保護する。
* **Threat Detection (ASA/FTD)**: 攻撃トラフィックを監視し、CPU 資源を消費する前に接続を強制拒否（Shun）する。
* **Logging & SNMP**: 過度なログ出力（`logging console`）は直接的に CPU を高騰させるため、運用設計でのログレベル制御が必須。

---

## 🧩 比較表

| 比較項目 | Cisco IOS/IOS-XE (Control Plane) | Cisco ASA (LINA) | Cisco FTD (Snort 3 Engine) |
| :--- | :--- | :--- | :--- |
| **アーキテクチャ** | Process / Interrupt 分割型 | モノリシック ＋ マルチスレッド型 | マルチスレッド（LINA ＋ Snort3） |
| **主な CPU 枯渇要因** | CEF Miss, IP Input, SSH/BGP 処理 | ASP Drop 処理, SYN Flood, Polling | L7 Deep Inspection, Elephant Flow |
| **直接的な CPU 保護機能** | CoPP, CPPr, `process cpu-threshold` | Threat Detection, Connection Limit | Pre-filter Fastpath, Threat Detection |
| **スケジューラ調整コマンド** | `scheduler allocate`, `scheduler interval` | `cpu queue` (内部最適化) | FXOS リソース割り当て最適化 |

---

## 💡 ベストプラクティス

1. **CEF の常時有効化**: `ip cef` および `ipv6 cef` を全ルーティングデバイスで必ず有効化する。
2. **コンソールログの無効化**: 本番・試験環境を問わず `no logging console` を基本設定とし、ログは `logging buffered` または SYSLOG サーバーへ転送する。
3. **CoPP の標準適用**: 全てのルーティングデバイスに BGP/OSPF/SSH/ICMP/ARP の帯域を定義した CoPP ポリシーを適用する。
4. **FTD Pre-filter の活用**: 大容量かつ事前検証済みの通信（データバックアップ、ビデオストリーミング、ハイパフォーマンス DB 間通信）は Pre-filter で `Fastpath` に指定し、Snort CPU を節約する。
5. **CPU 閾値監視の自動化**: `process cpu-threshold` を設定し、CPU が 85% を超えた際に即座に Syslog / SNMP Trap を送信させる。

## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験を意識した、CPU 保護・監視・トラブルシューティングに関する 10 個の実践的な演習シナリオです。

---

### シナリオ 1: IOS-XE における CPU 閾値監視と Syslog / SNMP アラートの設定

#### 問題
ルーター R1 において、コントロールプレーンの CPU 負荷を常時監視し、過負荷状態が発生した際に自動的にネットワーク管理システム (NMS) へ通知する仕組みを構築しなさい。

#### 要件
1. R1 全体の CPU 5秒平均利用率が **80%** を超えた場合、5秒後に Rising アラートの Syslog および SNMP トラップを発行すること。
2. CPU 利用率が **50%** 未満に低下した場合、5秒後に Falling 通知を発行すること。
3. いずれかの単一プロセス（Process-level）が CPU の **60%** 以上を占有した場合にもアラートを発行すること。
4. SNMP トラップの送信先として `10.1.100.50` (Community: `SecNET`) を指定すること。

#### 設定例

```bash
R1(config)# ! CPU 全体の閾値設定
R1(config)# process cpu-threshold type total rising 80 interval 5 falling 50 interval 5

R1(config)# ! 単一プロセスの閾値設定
R1(config)# process cpu-threshold type process rising 60 interval 5 falling 30 interval 5

R1(config)# ! SNMP トラップの設定
R1(config)# snmp-server enable traps cpu threshold
R1(config)# snmp-server host 10.1.100.50 SecNET cpu
```

#### 検証方法
1. `show process cpu-threshold` を実行し、配置された Rising/Falling 閾値およびインターバルを確認する。
2. `show running-config | include process cpu-threshold` で設定の正確性を検証する。
3. ループ処理等で意図的に CPU 負荷を発生させ、Syslog (`%SYS-1-CPURISINGTHRESHOLD`) が出力されることを確認する。

---

### シナリオ 2: 割り込み処理暴走に対する Scheduler Allocate 調整

#### 問題
R2 において、大量の割り込みパケット（Interrupt-level）が流入した際にも、BGP および SSH プロセスが制御不能に陥るのを防ぐため、CPU スケジューラのリソース割当を再定義しなさい。

#### 要件
1. 割り込み処理に割り当てる最大時間を **4000 マイクロ秒** とする。
2. プロセスレベル処理に最低限割り当てる時間を **1000 マイクロ秒** とする。
3. スケジューラのタイムスライス間隔を **500 マイクロ秒** に調整すること。

#### 設定例

```bash
R2(config)# scheduler allocate 4000 1000
R2(config)# scheduler interval 500
```

#### 検証方法
1. `show running-config | include scheduler` を実行し、コマンドが正しく反映されていることを確認する。
2. コントロールプレーン負荷試験時に、`show processes cpu` でプロセス処理が完全に短絶（0%）しないことを確認する。

---

### シナリオ 3: CEF Miss による IP Input CPU 高騰の防護（CoPP 統合）

#### 問題
R3 において、CEF で処理できず CPU にフォールバックする「IP Input」プロセスの高騰を防ぐため、自機器宛ての不要なトラフィックを制限しなさい。

#### 要件
1. 自機器宛ての ICMP トラフィックを **128 kbps** にポリシング（警察制御）すること。
2. 許容帯域を超えた ICMP トラフィックは即座にドロップすること。
3. BGP (TCP 179) 通信は優先的に許可し、帯域制限を設けないこと。

#### 設定例

```bash
R3(config)# ip access-list extended ACL-ICMP-CP
R3(config)#  permit icmp any any

R3(config)# ip access-list extended ACL-BGP-CP
R3(config)#  permit tcp any any eq bgp
R3(config)#  permit tcp any eq bgp any

R3(config)# class-map match-all CM-ICMP-CP
R3(config)#  match access-group name ACL-ICMP-CP

R3(config)# class-map match-all CM-BGP-CP
R3(config)#  match access-group name ACL-BGP-CP

R3(config)# policy-map PM-COPP-CPU
R3(config)#  class CM-BGP-CP
R3(config)#   police 10000000 conform-action transmit exceed-action transmit
R3(config)#  class CM-ICMP-CP
R3(config)#   police 128000 conform-action transmit exceed-action drop

R3(config)# control-plane
R3(config-cp)# service-policy input PM-COPP-CPU
```

#### 検証方法
1. 対向機器から `ping -f`（大容量高速 ICMP）を実行する。
2. `show policy-map control-plane` を実行し、`CM-ICMP-CP` クラスで `exceeded` パケットがドロップされ、`IP Input` プロセスの CPU が安定していることを確認する。

---

### シナリオ 4: IP Option 付与パケットによる CPU 攻撃の無効化

#### 問題
攻撃者が IP Option（LSRR, Record Route 等）を付与したパケットを送信し、R4 の CPU (Process Switching) を意図的に高騰させている。この攻撃を無効化しなさい。

#### 要件
1. IP Option が付与されたすべての IPv4 パケットを、CPU 処理へ送らずインターフェイスレベルで即座に破棄すること。
2. ICMP Unreachable 応答を送信して新たな CPU 負荷を発生させないこと。

#### 設定例

```bash
R4(config)# ip options drop
```

#### 検証方法
1. `show ip traffic` を実行し、`Header options` のドロップカウンターが増加していることを確認する。
2. `show processes cpu sorted` を確認し、`IP Input` プロセスの利用率が上昇しないことを検証する。

---

### シナリオ 5: ASA における Threat Detection を用いた CPU 保護

#### 問題
ASA1 において、外部ネットワークからのポートスキャンおよび DoS 攻撃による CPU 高負荷を防ぐため、Threat Detection を構築しなさい。

#### 要件
1. 基本的な Threat Detection を有効化すること。
2. スキャン攻撃 (Scanning Threat) を検知した場合、攻撃元ホストを **1時間 (3600秒)** 自動的に Shun（アクティブドロップ）すること。
3. 接続試行レートの監視間隔を 600 秒とし、平均レート 1000 回/秒を超えた場合に警告を出力すること。

#### 設定例

```bash
ASA1(config)# threat-detection basic-threat
ASA1(config)# threat-detection scanning-threat threat-level high action-ge-level high
threat-detection rate host-scan-rate 200 dump 3600
threat-detection rate-interval 600 average-rate 1000 burst-rate 3000
```

#### 検証方法
1. `show threat-detection` を実行し、機能が有効化されていることを確認する。
2. 疑似スキャンを実行し、`show shun` を確認して攻撃元 IP が動的に登録・ドロップされることを検証する。

---

### シナリオ 6: Console ログ出力による CPU 高騰の緊急トラブルシューティング

#### 問題
R5 において、大量のトラフィックが流れた際に `Console Output` プロセスが CPU の **95%** を占有し、管理アクセスが極めて重くなっている。この問題を解決しなさい。

#### 要件
1. コンソールポートへの即時ログ出力を無効化すること。
2. ログはメモリーバッファ (`buffered`) に保存し、サイズを **1MB (1048576 bytes)**、レベルを `debugging` とすること。

#### 設定例

```bash
R5(config)# no logging console
R5(config)# logging buffered 1048576 debugging
```

#### 検証方法
1. `show processes cpu sorted` を実行し、`Console Output` または `TTY Provider` プロセスの CPU 消費が 0% に低下したことを確認する。
2. `show logging` を実行し、バッファ内にログが記録されていることを確認する。

---

### シナリオ 7: ASA における ASP コントロールプレーンキューの調整

#### 問題
ASA2 において、コントロールプレーン宛ての突発的な管理トラフィック爆発時にパケットドロップが発生し、CPU へのバッファリングが破綻している。ASP キューを調整しなさい。

#### 要件
1. コントロールプレーン宛てバッファキューのサイズを拡張し、突発的なパケットスパイクによる破棄を抑制すること。

#### 設定例

```bash
ASA2(config)# cpu queue 1000
```

#### 検証方法
1. `show cpu queue` を実行し、設定されたキュー長およびドロップカウンターの推移を確認する。

---

### シナリオ 8: FTD (FMC) における Pre-filter Fastpath による Snort CPU 保護

#### 問題
FTD1 において、バックアップサーバー (`10.1.200.10`) とストレージ (`10.2.200.10`) 間の 10Gbps データ転送により、Snort エンジンの CPU 利用率が **100%** に張り付き、他の通信が遅延している。

#### 要件
1. 当該バックアップ通信（TCP Port 445 / 8500）を、Snort の L7 検査を通さず L3/L4 のみで高速転送 (Fastpath) させる Pre-filter ルールを作成しなさい。

#### 設定手順 (FMC GUI ＋ FlexConfig / CLI 表現)

1. **FMC GUI**:
   * `Policies` > `Access Control` > `Pre-filter` へ移動。
   * 新規 Pre-filter Rule を作成:
     * **Name**: `Fastpath-Backup-Traffic`
     * **Action**: `Fastpath`
     * **Source IP**: `10.1.200.10`
     * **Destination IP**: `10.2.200.10`
     * **Protocol**: `TCP` (Ports: `445`, `8500`)
   * Access Control Policy へ Pre-filter をバインドし、FTD1 へ Deploy。

#### 検証方法
1. 大容量データ転送を開始する。
2. FTD1 CLI にて `system support diagnostic-cli` を実行し、`show access-list` および `show service-policy` で Pre-filter にヒットしていることを確認する。
3. Expert モードの `top` コマンドで、`snort` プロセスの CPU 利用率が上昇しないことを検証する。

---

### シナリオ 9: IOS-XE における ARP スプーミング爆撃による CPU 保護

#### 問題
スイッチング環境における不審端末からの ARP 爆撃により、R6 の CPU (`ARP Input` プロセス) が高騰している。これを防止しなさい。

#### 要件
1. CoPP を用いて、未承認の ARP 通信の CPU 到達レートを制限すること。

#### 設定例

```bash
R6(config)# ip access-list extended ACL-ARP-CP
R6(config)#  permit arp any any

R6(config)# class-map match-all CM-ARP-CP
R6(config)#  match access-group name ACL-ARP-CP

R6(config)# policy-map PM-COPP-PROTECT
R6(config)#  class CM-ARP-CP
R6(config)#   police 200 pps conform-action transmit exceed-action drop

R6(config)# control-plane
R6(config-cp)# service-policy input PM-COPP-PROTECT
```

#### 検証方法
1. `show policy-map control-plane` で ARP のレートポリシングが機能していることを確認する。
2. `show processes cpu sorted | include ARP` で `ARP Input` の CPU 利用率が安定していることを確認する。

---

### シナリオ 10: ASA コネクション数制限による SYN Flood CPU 枯渇防止

#### 問題
ASA3 の Outside インターフェイス宛てに大量の SYN Flood 攻撃が流入し、半開（Embryonic）コネクションが爆発して CPU 資源が枯渇している。これを防止しなさい。

#### 要件
1. Web サーバー (`172.16.1.100`) 宛ての最大未完了（Embryonic）コネクション数を **500** に制限すること。
2. 接続制限を超えた通信に対して SYN Cookie (TCP Intercept) を自動発動させ、CPU 負荷を抑えること。

#### 設定例

```bash
ASA3(config)# access-list ACL-WEBSERVER extended permit tcp any host 172.16.1.100 eq 80

ASA3(config)# class-map CM-WEBSERVER
ASA3(config)#  match access-list ACL-WEBSERVER

ASA3(config)# policy-map PM-PROTECT-CPU
ASA3(config)#  class CM-WEBSERVER
ASA3(config)#   set connection embryonic-conn-max 500

ASA3(config)# service-policy PM-PROTECT-CPU interface outside
```

#### 検証方法
1. 外部から大量の SYN パケットを送信する。
2. `show service-policy interface outside` を実行し、`Embryonic connection limit` のカウンターおよび TCP Intercept の作動状況を確認する。
3. `show cpu` で CPU 利用率が安全圏に維持されていることを確認する。

## ❓ 想定試験問題

CCIE Security ラボ・筆記試験レベルを想定した実践問題と解答解説です。

---

### 問題 1 (コンフィグ読解・トラブルシューティング)

ルーター R1 の CPU 利用率が 99% に達し、BGP ネイバーがダウンしました。管理者が `show processes cpu sorted` を実行したところ、以下の出力が得られました。

```text
CPU utilization for five seconds: 99%/92%; one minute: 95%; five minutes: 91%
 PID Runtime(ms)     Invoked      uSecs   5Sec   1Min   5Min TTY Process 
  90     1024850      241029       4251  0.50%  0.45%  0.40%   0 BGP Router
  12       84920       12001       7076  0.10%  0.08%  0.05%   0 IP Input
```

この出力から判断できる **CPU 高騰の直接的な原因** と、最優先で実行すべき対処コマンドとして最も適切な組み合わせを選びなさい。

1. **原因**: BGP プロセスによる計算処理の遅延。 **対処**: `scheduler allocate 5000 1000` を設定する。
2. **原因**: CEF が無効化されているか、パケットがハードウェアで処理されず大量の割り込み（Interrupt）が発生している。 **対処**: `ip cef` をグローバルおよびインターフェイスで有効化する。
3. **原因**: IP Input プロセスによる不審な ICMP パケットの過大処理。 **対処**: CoPP ポリシーを作成して ICMP をドロップする。
4. **原因**: コンソールポートへの大量ログ出力。 **対処**: `no logging console` を設定する。

#### 解答・解説
* **正解**: **2**
* **解説**:
  * 1 行目の `CPU utilization for five seconds: 99%/92%` において、`99%` が全体の CPU 利用率、スラッシュの後の `92%` が **割り込みレベル（Interrupt-level）** の CPU 利用率を示しています。
  * プロセス一覧の合計（BGP Router 0.50%, IP Input 0.10%）は僅かであり、CPU の大半（92%）はプロセススケジューラへ引き渡される前の物理割り込み処理（Rx/Tx DMA、CEF Miss）で消費されています。
  * これは CEF が無効化された際によく見られる典型的な挙動であり、`ip cef` を有効化してパケットを FastPath 転送に戻すことが最優先の対処となります。

---

### 問題 2 (Design・実装)

Cisco ASA において、ポートスキャンやバースト通信による CPU 高負荷を防ぐため、相手ホストを即座に動的ドロップする仕組みを構築したい。これを実現するためのコマンドの組み合わせとして正しいものを選択しなさい。

1. `ip options drop` および `control-plane`
2. `threat-detection basic-threat` および `threat-detection scanning-threat threat-level high action-ge-level high`
3. `process cpu-threshold type total rising 85`
4. `access-group control-plane` および `service-policy`

#### 解答・解説
* **正解**: **2**
* **解説**:
  * ASA では `threat-detection` 機能を使用することで、スキャン行為や異常な接続レートを監視し、自動的に相手ホストを Shun（アクティブドロップ）することができます。

### 問題 3 (FTD アーキテクチャ・トラブルシューティング)

FTD において、大容量の単一 TCP フロー（Elephant Flow）が発生し、特定の Snort インスタンスコアが 100% に張り付き、他のユーザーの通信が遅延しています。この問題の恒久的な回避策として最も適切なものを選択しなさい。

1. `no logging console` を FMC から適用する。
2. 対象の通信を識別し、Pre-filter Rule で Action を `Fastpath` に指定する。
3. CoPP ポリシーを FMC から FTD へ配布する。
4. FTD を再起動して Snort インスタンス数を削減する。

#### 解答・解説
* **正解**: **2**
* **解説**:
  * Pre-filter Rule で `Fastpath` を指定すると、パケットは Snort エンジン（L7 インスペクション）へ渡されず、LINA データパス（L3/L4）のみで直接高速転送されます。これにより Snort CPU コアの飽和を回避できます。

---

### 問題 4 (IOS スケジューリング)

IOS において、大量の物理割り込みが発生している環境下でも、プロセスレベルのタスク（BGP, SSH 等）が最小限の CPU 時間を獲得できるように調整するコマンドはどれか。

1. `process cpu-threshold`
2. `scheduler allocate`
3. `threat-detection`
4. `service-policy input`

#### 解答・解説
* **正解**: **2**
* **解説**:
  * `scheduler allocate <interrupt_time> <process_time>` コマンドは、割り込み処理とプロセス処理の CPU タイムスライス配分を調整するコマンドです。

---

### 問題 5 (コンフィグ読解)

以下の IOS コマンド設定に関する記述として正しいものを選びなさい。

```text
process cpu-threshold type total rising 85 interval 5 falling 60 interval 5
```

1. CPU 利用率が 85% を超えた瞬間に、即座に CPU 限界動作として不要プロセスを終了（Kill）する。
2. 5秒間の平均 CPU 利用率が 85% を超えた場合に Rising アラートを発行し、その後 5秒平均が 60% 未満に落ちた際に Falling 通知を発行する。
3. 85秒間にわたって CPU 利用率が 5% を超えた場合にアラートを発行する。
4. 単一プロセスの利用率が 85% に達した場合にのみ動作する。

#### 解答・解説
* **正解**: **2**
* **解説**:
  * `type total` はシステム全体の CPU を監視し、`rising 85 interval 5` は 5秒平均で 85% 超えを条件とし、`falling 60 interval 5` は 5秒平均で 60% 未満への回復を条件として通知を行います。

## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術情報です。

* Cisco Technical Note: Troubleshooting High CPU Utilization on Cisco Routers
* Cisco Technical Note: High CPU Utilization on Cisco IOS Software-Based Routers
* Cisco ASA Configuration Guide: Threat Detection Overview
* Cisco Secure Firewall Threat Defense Configuration Guide: Pre-filter Policies
* Cisco Live Slide: BRKARC-2001 - Router CPU and Memory Architecture
* Cisco Command Reference: process cpu-threshold Command

---

## 📝 補足（Notes）

### CPU 監視における表示の違いと注意点
* **IOS `show processes cpu` の読み方**:
  * `CPU utilization for five seconds: A%/B%`
  * `A%`: 総 CPU 利用率。
  * `B%`: 割り込みレベル（Interrupt-level）の利用率。
  * `A - B`: プロセスレベル（Process-level）の利用率。
  * **例**: `95%/80%` → 割り込みが 80%、プロセスが 15%。原因は割り込みパケットや CEF の問題。
  * **例**: `95%/5%` → 割り込みが 5%、プロセスが 90%。原因は特定のプロセス（`show processes cpu sorted` の一覧で確認）。

### ASA / FTD の Datapath スレッドに関する注意点
* ASA/FTD では、トラフィックが全く流れていなくても `DATAPATH` スレッド（Polling Thread）が NIC を絶え間なくチェックするため、`show cpu` で特定のコアが 100% に見える場合があります。
* これがパケットバーストや障害によるものか正常なポーリングによるものかは、`show asp drop` や `show traffic`、および実際の転送遅延の有無で総合的に判断する必要があります。
