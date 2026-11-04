---
layout: default
title: 3.6.a-NetFlow-IPFIX-NSEL
nav_order: 1
parent: 3.6-Monitoring
grand_parent: 3.0-Security-Infrastructure
---

# 3.6.a NetFlow/IPFIX/NSEL

## 📘 概要

ネットワークインフラストラクチャにおけるトラフィックの可視化、脅威検出、およびセキュリティアナリティクスを実現する上で、**NetFlow**、**IPFIX (IP Flow Information Export)**、および **NSEL (NetFlow Secure Event Logging)** は中核となるモニタリング技術です。

従来のトラフィック統計（SNMPなどによるインターフェイス単位のインターバル集計）では、パケットの送信元・送信先IPアドレス、ポート番号、アプリケーション識別などの詳細なプロトコル挙動を把握することはできませんでした。これに対し、フローモニタリング技術はネットワーク上を通過するパケットを「フロー（Flow）」単位で識別・記録し、セキュリティインシデント解析やアノマリ検出（DoS/DDoS攻撃、マルウェアの側面的移動）、ネットワーク容量計画に不可欠なテレメトリデータを提供します。

### 主な利用目的と適用シーン

* **ネットワークトラフィックの可視化**: 5-tuple (送信元IP, 宛先IP, 送信元ポート, 宛先ポート, L4プロトコル) に基づくアプリケーションレベルでの帯域利用状況の追跡。
* **セキュリティインシデント解析と脅威検知**: Cisco Secure Network Analytics (旧 Stealthwatch) 等のNDR (Network Detection and Response) ソリューションと連携し、異常なデータ転送や不正アクセスを検知。
* **ステートフルファイアウォールイベントの高速ログ出力**: Cisco ASA および Firepower Threat Defense (FTD) において、従来の Syslog の処理限界を超える大量のアクセスログを低負荷・高スループットで出力（NSEL）。
* **マルチベンダー環境での標準化モニタリング**: IETF 標準規格である IPFIX を使用し、ベンダー固有の制約を受けずにフローデータをコレクタへ送信。


---

## 🔑 要点

NetFlow、IPFIX、NSEL の主要な技術的特徴を下表にまとめます。

| 項目 | 内容 |
| --- | --- |
| **特徴** | パケットのヘッダー情報からフロー（共通属性を持つパケット群）を形成し、統計情報またはイベント情報をコレクタへ UDP/SCTP 出力 |
| **用途** | ネットワークトラフィック可視化、セキュリティアナリティクス (NDR連携)、DoS/DDoS検知、ファイアウォール高速セッションログ出力 |
| **メリット** | パケットキャプチャと比べて転送データ量が極めて小さく低負荷。NSELではセッション生成/破棄のイベント単位出力により高スループットを実現 |
| **デメリット** | パケットのペイロード内容は保持しない。従来の NetFlow (v5/v9) ではタイマー周期のキャッシュ出力によるタイムラグが発生する可能性あり |
| **対応機種** | Cisco Catalyst スイッチ、ISR/ASR/Catalyst 8000 ルータ (FNF/IPFIX)、Cisco ASA (NSEL)、Cisco FTD/FMC (NSEL/FNF) |
| **制限事項** | サンプリング（Sampled NetFlow）適用時は微小な攻撃トラフィックを見落とすリスクあり。NSELはセッションイベントに限定（統計キャッシュなし） |
| **設計上の注意点** | コレクタ側との UDP ポート番号整合性（デフォルト: 2055 または 9996）、MTU サイズ超過によるフラグメンテーションの回避 |


---

## 🏗 動作原理

フローモニタリング技術は、ハードウェア/OS実装に応じて **Flexible NetFlow (FNF)**、**IPFIX**、および **NSEL** の3つのアーキテクチャに分類されます。

### 1. Flexible NetFlow (FNF) アーキテクチャ (Cisco IOS / IOS-XE)

Flexible NetFlow は、「Record」「Exporter」「Monitor」という3つの独立したオブジェクトで構成されるモジュール構造を持っています。

```
[ インターフェイス ] (ip flow monitor <MONITOR_NAME> input/output)
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ Flow Monitor                                                │
│                                                             │
│  ┌───────────────────────────┐  ┌─────────────────────────┐  │
│  │ Flow Record               │  │ Flow Exporter           │  │
│  │ (Match & Collect Keys)    │  │ (Destination IP/Port,   │  │
│  │  - match ipv4 src/dst     │  │  Source Interface,      │  │
│  │  - match transport port   │  │  Export Protocol)       │  │
│  │  - collect bytes/packets  │  └────────────┬────────────┘  │
│  └───────────────────────────┘               │              │
└──────────────────────────────────────────────┼──────────────┘
                                               │ (NetFlow v9 / IPFIX UDP Packet)
                                               ▼
                                   [ NetFlow Collector ]
```

* **Flow Record**: フローを識別するための「Match キー」（送信元/宛先IP、L4ポート、VLAN IDなど）と、抽出する「Collect キー」（パケット数、バイト数、タイムスタンプなど）を定義。
* **Flow Exporter**: キャッシュされたフローデータを送信するターゲット（コレクタのIPアドレス、UDPポート番号、送信元インターフェイス、エクスポートプロトコル: `netflow-v9` や `ipfix`）を定義。
* **Flow Monitor**: Flow Record と Flow Exporter を紐付け、最終的にルータ/スイッチの物理インターフェイスや SVI に適用（`ip flow monitor`）。

### 2. IPFIX (IP Flow Information Export - RFC 5101)

IPFIX は NetFlow v9 をベースに IETF で標準化されたプロトコルです。

* **Information Elements (IEs)**: IANA によって標準化された 16 ビットの要素 ID を使用してフローフィールドを記述。
* **Enterprise Bit (E-bit)**: ベンダー固有のカスタムフィールド（Cisco 固有のメタデータなど）を識別するために 32 ビットの Enterprise Number を拡張可能。
* **可変長フィールド (Variable-length Fields)**: URL や HTTP User-Agent などの動的な文字列データをエクスポート可能。

### 3. NSEL (NetFlow Secure Event Logging) アーキテクチャ (Cisco ASA / FTD)

Cisco ASA および Firepower Threat Defense (FTD) における NSEL は、通常のパケットサンプルベースの統計出力とは異なり、**ステートフルセッションのイベント発生時**に即時メッセージを出力するイベント駆動型メカニズムです。

```
[ クライアント ] ── (SYN パケット) ──> [ ASA / FTD (LINA Engine) ]
                                              │
                                              ├─► セッション作成 (Flow Created Event)
                                              │    └─► NSEL (v9) ──> [ Collector ]
                                              │
                                       (通信終了 / FIN・RST)
                                              │
                                              └─► セッション破棄 (Flow Teardown Event)
                                                   └─► NSEL (v9) ──> [ Collector ]
```

* **イベント駆動出力**:
  * `flow-create`: 新しいステートフル接続が確立された際に出力。
  * `flow-denied`: ACL や ASP Drop、Threat Detection 等でパケットが拒否された際に出力。
  * `flow-teardown`: 接続が終了（タイムアウトまたは FIN/RST）した際に出力（転送バイト数や継続時間を含む）。
* **Modular Policy Framework (MPF) による統合**: ASA では `class-map`、`policy-map`、`service-policy` を用いて、特定のトラフィックフローに対して選択的に NSEL を適用。


---

## ⚙ 動作シーケンス

フローデータの作成からコレクタへのエクスポートに至るパケット処理シーケンスは以下の通りです。

```
[ 入力パケット到着 ]
       │
       ▼
1. フローキャッシュルックアップ (5-tuple キー比較)
       │
       ├─► [ キャッシュヒット (既存フロー) ]
       │        │
       │        └─► パケット数・バイト数・最終アクセス時刻のカウンタを更新
       │
       └─► [ キャッシュミス (新規フロー) ]
                │
                └─► 新規フローキャッシュエントリーの作成 (Match/Collect フィールド保持)
       │
       ▼
2. フローパージ・タイマー評価 (NetFlow/FNF)
       │
       ├─► [ Active Timeout (デフォルト 1800秒/30分) 超過 ] ──► フローを固定化してエクスポート＆エントリー更新
       ├─► [ Inactive Timeout (デフォルト 15秒) 超過 ] ────► 通信終了とみなしエクスポート＆エントリー削除
       └─► [ TCP FIN / RST 検出 ] ─────────────────────────► 即座にエクスポート＆エントリー削除
       │
       ▼
3. パケット構築 & エクスポート
       │
       ├─► Template Flow Set (テンプレートパケット) の定期送出 (コレクタへフィールド構造を通知)
       └─► Data Flow Set (実際のフロー統計/イベントデータ) を UDP パケットとしてコレクタへ出力
```


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、NetFlow / IPFIX / NSEL は単体での設定問題だけでなく、**Cisco Secure Network Analytics (Stealthwatch) との連携**や**トラブルシューティング問題**として頻繁に出題されます。

### Blueprintで重要なポイント

1. **Flexible NetFlow (FNF) の構築プロセスの正確性**:
   * `flow record` ──► `flow exporter` ──► `flow monitor` ──► `interface` への適用という4段階の順序をミスなく構築できるか。
   * Match キーと Collect キーの適切な選択（例: 送信元/宛先 IPv4 アドレス、L4 ポート番号、入力インターフェイスを `match` とし、カウンタやタイムスタンプを `collect` とする）。
2. **ASA における NSEL (MPF) の適用パターン**:
   * `flow-export event-type` の適切な定義（`all` や `flow-creation` / `flow-teardown` / `flow-denied` の個別指定）。
   * `policy-map` 内での `set connection flow-export event-log <group-name>` の正しいクラス適用。
3. **FMC (Firepower Management Center) での NSEL 設定手段の使い分け**:
   * **Platform Settings**: FMC 上で FTD デバイスに対して NSEL コレクタ (IP, UDPポート) およびフローエクスポートイベントを設定する標準的な方法。
   * **FlexConfig**: 特殊な FNF / IPFIX カスタム設定や、Advanced NSEL パラメータを適用する際に使用。
4. **テンプレート送出タイマー (Template Retransmit Interval)**:
   * UDP は接続レスであるため、コレクタがフローデータを解釈するには定期的に「テンプレート」を受信する必要があります。`option template-retransmit` や `template data-template interval` の設定を見落とすと、コレクタ側で「Unknown Template」エラーとなりフローが解析不能になります。

### よくある設定ミスとハマりポイント

* **送信元インターフェイス指定漏れ (`option export-protocol` / `source <interface>`)**:
  * Exporter に `source` インターフェイス（Loopback や Inside など）を指定しない場合、ルーティングテーブルに従って出力インターフェイスのアドレスが使用されます。コレクタ側のアクセス制御リストや FQDN/IP 認証でブロックされる原因となります。
* **NSEL における `flow-denied` の設定漏れ**:
  * ASA / FTD で「拒否されたトラフィック（ドロップログ）も収集せよ」という要件に対して、`flow-export event-type flow-denied` を設定し忘れるケース。
* **ASA MPF の `global_policy` 適用重複/漏れ**:
  * ASA のデフォルト `global_policy` にクラスを追加するか、インターフェイス固有の `service-policy` を適用するかの指示を厳密に読み取る必要があります。

---

## 🛠 設定方法

IOS-XE (FNF / IPFIX)、Cisco ASA (NSEL CLI)、および Cisco FTD/FMC (NSEL Platform Settings) の設定例を具体的に解説します。

### 1. Cisco IOS-XE Flexible NetFlow (FNF) / IPFIX 設定例

以下は、ルータの GigabitEthernet1 インターフェイスで全 IPv4 トラフィックのフローを収集し、UDP 2055 でコレクタ (192.168.100.50) へ送出する設定です。

```text
! 1. Flow Record の定義
flow record REC-SECURITY-ANALYSIS
 match ipv4 source address
 match ipv4 destination address
 match ipv4 protocol
 match transport source-port
 match transport destination-port
 match interface input
 collect counter bytes long
 collect counter packets long
 collect timestamp sys-uptime first
 collect timestamp sys-uptime last

! 2. Flow Exporter の定義 (IPFIX プロトコル使用)
flow exporter EXP-STEALTHWATCH
 destination 192.168.100.50
 source Loopback0
 transport udp 2055
 option template-retransmit 60
 export-protocol ipfix

! 3. Flow Monitor の定義
flow monitor MON-SECURITY
 record REC-SECURITY-ANALYSIS
 exporter EXP-STEALTHWATCH
 cache timeout active 60
 cache timeout inactive 15

! 4. インターフェイスへの適用
interface GigabitEthernet1
 ip flow monitor MON-SECURITY input
 ip flow monitor MON-SECURITY output
```

### 2. Cisco ASA NSEL 設定例 (CLI)

以下は、ASA の全トラフィック（Creation, Teardown, Denied）を NSEL コレクタ (192.168.100.50:2055) へ送出する MPF 設定です。

```text
! 1. NSEL コレクタの定義
flow-export destination inside 192.168.100.50 2055
flow-export template timeout-rate 1
flow-export delay flow-create 0

! 2. NSEL クラスマップの定義
class-map NETFLOW-TRAFFIC
 match any

! 3. ポリシーマップへの適用
policy-map global_policy
 class NETFLOW-TRAFFIC
  flow-export event-type all destination 192.168.100.50

! 4. グローバル サービスポリシーの確認
service-policy global_policy global
```

### 4. Cisco FTD / FMC での NSEL 設定手順 (GUI)

FMC から FTD へ NSEL を設定する手順は以下の通りです。

1. **FMC 管理画面にログイン**: `Devices` ──► `Platform Settings` へ移動。
2. **Platform Settings Policy の作成/編集**: 対象の FTD デバイスが割り当てられているポリシーを選択。
3. **NetFlow (NSEL) セクションの設定**:
   * **NetFlow Collectors**: コレクタ IP アドレス（例: `192.168.100.50`）、UDP ポート（例: `2055`）、および送信元インターフェイス（例: `inside`）を定義。
   * **NetFlow Rules**: イベントタイプ (`All` / `Flow Creation` / `Flow Teardown` / `Flow Denied`) を選択し、適用対象クラスを指定。
4. **ポリシーの保存と Deploy**: 変更をターゲット FTD デバイスへデプロイ。


## 🔍 検証コマンド

NetFlow / FNF / NSEL の動作状態や統計情報を確認するための主要コマンドを下表にまとめます。

| 目的 | コマンド (IOS-XE / ASA) |
| --- | --- |
| FNF キャッシュエントリーの確認 | <code>show flow monitor <MONITOR_NAME> cache</code> |
| FNF モニタの統計・適用状態確認 | <code>show flow monitor <MONITOR_NAME></code> |
| FNF エクスポートパケット統計確認 | <code>show flow exporter <EXPORTER_NAME> statistics</code> |
| FNF レコード構造の確認 | <code>show flow record <RECORD_NAME></code> |
| ASA NSEL エクスポート統計の確認 | <code>show flow-export statistics</code> |
| ASA MPF 統合状態の確認 | <code>show service-policy flow-export</code> |
| FNF 動作デバッグログ | <code>debug flow monitor</code> / <code>debug flow exporter</code> |
| ASA NSEL イベントデバッグログ | <code>debug flow-export event</code> |


---

## 🚨 トラブルシュート

トラブルシューティング時の典型的な症状、原因、確認コマンド、および対処方法を下表に整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| コレクタ側でフローデータが「Unknown Template」としてドロップされる | テンプレートパケットが定期送信されていない、または初回パケット受信前 | <code>show flow exporter statistics</code> | エクスポートタイマー (`template-retransmit`) を短縮 (例: 60秒)。ASA では `flow-export template timeout-rate 1` を設定 |
| コレクタ側で一切のフローデータが検出されない | Exporter の宛先 IP/Port 誤り、またはルーティング/ACL による遮断 | <code>show flow exporter statistics</code><br><code>show ip route <COLLECTOR_IP></code> | 宛先 IP/UDP ポート番号を確認。コレクタ経路上の ACL およびファイアウォールで UDP 2055/9996 の通過を許可 |
| ASA / FTD から拒否ログ (Denied Event) が出力されない | MPF で `flow-export event-type flow-denied` が指定されていない | <code>show service-policy flow-export</code> | MPF の policy-map 内で `event-type all` または `event-type flow-denied` を明示的に設定 |
| フロー統計の更新が遅延しリアルタイム性が損なわれる | FNF の `cache timeout active` の値が大きすぎる (デフォルト 1800秒) | <code>show flow monitor <MONITOR></code> | `cache timeout active 60` (1分) など、要求されるモニタリング解像度に合わせてアクティブタイマーを短縮 |
| FNF キャッシュが溢れて新しいフローが記録されない (`Cache full`) | キャッシュエントリー上限の到達、または大量のDoS攻撃による新規フロー急増 | <code>show flow monitor <MONITOR> cache format table</code> | `cache entries <1024-524288>` コマンドでキャッシュサイズを拡張。または `cache type immediate` (サンプル出力) を検討 |


---

## ⚠ 制限事項

* **パケットサンプリングのトレードオフ**: サンプルモード (Sampled NetFlow) を有効にした場合、CPU/メモリ負荷は著しく低減しますが、単発の攻撃パケットやスキャン攻撃などの微小なトラフィックイベントが見落とされるリスクがあります。
* **NSEL における非イベントログの不保持**: ASA / FTD の NSEL は「セッション生成/破棄/拒否」イベントのみを転送します。従来の NetFlow のように中間経過のパケットカウンタを一定周期でエクスポートする挙動とは異なるため、コレクタ側の仕様適合性を確認する必要があります。
* **ハードウェア ASIC 制限**: スイッチプラットフォーム (Catalyst 9300/9400/9500 など) では、TCAM リソースの制約により同時にアクティブ化できる FNF モニター数や Match キーの組み合わせに制約が存在します。


---

## 🔄 他技術との関連

* **Cisco Secure Network Analytics (Stealthwatch)**: FNF / IPFIX / NSEL からエクスポートされたフローデータをコレクタ (Flow Collector) で受信し、機械学習と行動分析 (EBA: Entity Behavior Analytics) によってアノマリや内部脅威を自動検出します。
* **Cisco TrustSec / SGT (Security Group Tag)**: FNF の Match キーに `match metadata security-group source` および `destination` を組み込むことで、SGT ベースのセグメンテーション通信ログをグラフィカルに可視化可能です。
* **Modular Policy Framework (MPF)**: ASA / FTD における NSEL は MPF の一部として実行されるため、アクセス制御ポリシーや QOS ポリシーと同一の分類エンジンを共有します。


---

## 🧩 比較表

フローモニタリング技術の各標準およびプロトコル比較を下表に示します。

| 比較項目 | Traditional NetFlow v5 | NetFlow v9 | IPFIX (RFC 5101) | NSEL (ASA / FTD) |
| --- | --- | --- | --- | --- |
| **標準規格** | Cisco 独自 (固定フォーマット) | Cisco 独自 (テンプレートベース) | IETF 標準規格 | Cisco 独自 (NetFlow v9 拡張) |
| **可変フィールド** | 非対応 | 対応 (Template 依存) | 対応 (Variable-length IEs) | 定形イベントテンプレート |
| **主目的** | パケット統計・帯域集計 | パケット統計・IPv6対応 | 汎用フローエクスポート | 高速ファイアウォールイベント出力 |
| **出力トリガー** | タイマー判定 (Active/Inactive) | タイマー判定 (Active/Inactive) | タイマー判定 / イベント | セッションイベント (Create/Teardown/Deny) |
| **トランスポート** | UDP (固定) | UDP (固定) | UDP / TCP / SCTP | UDP (デフォルト) |
| **ベンダー拡張** | 不可 | 限定的 | 可能 (Enterprise Bit) | 特定ベンダーフィールド埋め込み |


---

## 💡 ベストプラクティス

1. **Active Timeout の最適化**: セキュリティアナリティクス (Stealthwatch 等) と連携する場合は、長時間のセッション (SSH, HTTPS等) のフローをタイムリーに可視化するため、`cache timeout active` を `60` 秒（1分）に設定することを推奨します。
2. **テンプレート送信周期の調整**: コレクタの再起動やネットワーク障害からの迅速な復旧のため、`option template-retransmit 60` (1分周期) を設定し、テンプレート情報を高頻度で提供します。
3. **NSEL での不要イベント除外**: ログ負荷が高い環境では、ドロップログが不要な場合は `flow-export event-type flow-creation` および `flow-teardown` のみに絞り込むことで、コレクタの処理負荷を最小化します。


## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験レベルの実践的な設定演習サンプルを 10 個掲載します。

### サンプル 1: Cisco IOS-XE 上での Flexible NetFlow (FNF) 基本構成

#### 問題
ルータ R1 の GigabitEthernet1 インターフェイスを通過するすべての IPv4 トラフィックに対して Flexible NetFlow を設定し、送信元・宛先 IP アドレス、L4 プロトコル、および L4 ポート番号を収集してコレクタへ送信してください。

#### 要件
* Flow Record 名: `REC-LAB-BASIC`
* Flow Exporter 名: `EXP-LAB-COLLECTOR`
  * 宛先 IP アドレス: `10.1.100.50`
  * UDP ポート: `2055`
  * 送信元: `Loopback0`
* Flow Monitor 名: `MON-LAB-BASIC`
* インターフェイス `GigabitEthernet1` の Ingress および Egress 方向でモニタを有効化すること。

#### 設定例
```text
! 1. Flow Record の定義
flow record REC-LAB-BASIC
 match ipv4 source address
 match ipv4 destination address
 match ipv4 protocol
 match transport source-port
 match transport destination-port
 collect counter bytes long
 collect counter packets long
 collect timestamp sys-uptime first
 collect timestamp sys-uptime last

! 2. Flow Exporter の定義
flow exporter EXP-LAB-COLLECTOR
 destination 10.1.100.50
 source Loopback0
 transport udp 2055
 option template-retransmit 60

! 3. Flow Monitor の定義
flow monitor MON-LAB-BASIC
 record REC-LAB-BASIC
 exporter EXP-LAB-COLLECTOR
 cache timeout active 60
 cache timeout inactive 15

! 4. インターフェイスへの適用
interface GigabitEthernet1
 ip flow monitor MON-LAB-BASIC input
 ip flow monitor MON-LAB-BASIC output
```

#### 検証方法
* `show flow monitor MON-LAB-BASIC cache format table` を実行し、キャプチャされた 5-tuple フローが正しく表示されるか確認。
* `show flow exporter EXP-LAB-COLLECTOR statistics` を実行し、送信パケット数が増加していることを確認。


---

### サンプル 2: Cisco TrustSec (SGT) メタデータを含めた FNF 構成

#### 問題
Cisco TrustSec 環境において、ロールベースアクセスコントロールの通信状況を可視化するため、送信元および宛先の Security Group Tag (SGT) を含めた FNF を設定してください。

#### 要件
* Flow Record 名: `REC-SGT-MONITOR`
* 送信元 IP、宛先 IP、L4 プロトコル、送信元 SGT、宛先 SGT を Match キーとすること。
* Flow Exporter 名: `EXP-STEALTHWATCH`
  * 宛先 IP: `10.1.100.50` (UDP 2055)
  * 送信元: `GigabitEthernet0/0`
* Flow Monitor 名: `MON-SGT` を `GigabitEthernet1` に適用すること。

#### 設定例
```text
! 1. SGT メタデータを含む Flow Record の定義
flow record REC-SGT-MONITOR
 match ipv4 source address
 match ipv4 destination address
 match ipv4 protocol
 match transport source-port
 match transport destination-port
 match metadata security-group source
 match metadata security-group destination
 collect counter bytes long
 collect counter packets long

! 2. Flow Exporter の定義
flow exporter EXP-STEALTHWATCH
 destination 10.1.100.50
 source GigabitEthernet0/0
 transport udp 2055
 option template-retransmit 60

! 3. Flow Monitor の定義
flow monitor MON-SGT
 record REC-SGT-MONITOR
 exporter EXP-STEALTHWATCH
 cache timeout active 60

! 4. インターフェイス適用
interface GigabitEthernet1
 ip flow monitor MON-SGT input
```

#### 検証方法
* `show flow monitor MON-SGT cache` を実行し、`src sgt` および `dst sgt` のフィールドに正しい SGT 値（例: 2, 3 等）が記録されていることを確認。


---

### サンプル 3: IPFIX プロトコルを使用した FNF 構成

#### 問題
マルチベンダーコレクタとの相互運用性を確保するため、エクスポートプロトコルに IETF 標準の IPFIX を指定した Flexible NetFlow を構成してください。

#### 要件
* Flow Exporter 名: `EXP-IPFIX-STD`
  * 宛先 IP: `172.16.10.100` (UDP 4739)
  * プロトコル: `ipfix`
  * テンプレート送信周期: `30` 秒
* Flow Monitor 名: `MON-IPFIX`
* `GigabitEthernet2` の Ingress に適用すること。

#### 設定例
```text
flow record REC-IPFIX-DEFAULT
 match ipv4 source address
 match ipv4 destination address
 match ipv4 protocol
 match transport source-port
 match transport destination-port
 collect counter bytes long
 collect counter packets long

flow exporter EXP-IPFIX-STD
 destination 172.16.10.100
 source Loopback0
 transport udp 4739
 export-protocol ipfix
 option template-retransmit 30

flow monitor MON-IPFIX
 record REC-IPFIX-DEFAULT
 exporter EXP-IPFIX-STD

interface GigabitEthernet2
 ip flow monitor MON-IPFIX input
```

#### 検証方法
* `show flow exporter EXP-IPFIX-STD` を実行し、`Export protocol: IPFIX (RFC 5101)` と表示されることを確認。


---

### サンプル 4: Cisco ASA CLI による NSEL 基本構成 (Create / Teardown)

#### 問題
Cisco ASA 5585-X において、セッションの生成 (flow-creation) および破棄 (flow-teardown) イベントを NetFlow コレクタへ送信するように NSEL を設定してください。

#### 要件
* コレクタ IP: `192.168.10.250` (UDP 2055)
* 送信元インターフェイス: `inside`
* MPF を使用してすべてのトラフィックの Create / Teardown イベントを送信すること。

#### 設定例
```text
! 1. NSEL コレクタ宛先の設定
flow-export destination inside 192.168.10.250 2055

! 2. テンプレート送信レートの調整 (1分ごと)
flow-export template timeout-rate 1

! 3. NSEL 用クラスマップの定義
class-map CM-NSEL-EVENTS
 match any

! 4. ポリシーマップへのイベント適用
policy-map global_policy
 class CM-NSEL-EVENTS
  flow-export event-type flow-creation destination 192.168.10.250
  flow-export event-type flow-teardown destination 192.168.10.250

! 5. グローバル適用確認
service-policy global_policy global
```

#### 検証方法
* `show flow-export statistics` を実行し、`Packets sent` および `Flow-create events` のカウントが増加することを確認。


---

### サンプル 5: ASA NSEL における Denied イベント (拒否ログ) の追加構成

#### 問題
セキュリティ要件に基づき、ASA で ACL や ASP Drop によって拒否されたトラフィック（flow-denied）も含めて NSEL コレクタへ通知するように追加構成してください。

#### 要件
* 既存の NSEL コレクタ `192.168.10.250` に対し、`flow-denied` イベントを出力するように追加。
* 設定は既存の `global_policy` 内で完結させること。

#### 設定例
```text
policy-map global_policy
 class CM-NSEL-EVENTS
  flow-export event-type flow-denied destination 192.168.10.250
```

#### 検証方法
* 不許可のアクセス制御リストに該当するテストパケットを送信し、`show flow-export statistics` で `Flow-denied events` の数値が増加することを確認。


### サンプル 6: FMC Platform Settings による FTD NSEL 構成

#### 問題
FMC によって管理されている FTD デバイスに対し、Platform Settings ポリシーを使用して NSEL を構成し、ステートフル接続イベントを可視化してください。

#### 要件
* コレクタ IP: `10.200.1.50` (UDP 2055)
* エクスポートインターフェイス: `outside`
* すべての接続イベント (`All`: Creation, Teardown, Denied) をエクスポートすること。

#### 設定手順とコンフィグ表現 (FMC Generated / FDM FTD)
1. FMC 管理画面にて `Devices` ──► `Platform Settings` にアクセス。
2. 対象の FTD Platform Settings ポリシーを開き、`NetFlow` タブを選択。
3. `Add NetFlow Collector`:
   * Name: `NSEL-COLLECTOR-1`
   * IP Address: `10.200.1.50`
   * Port: `2055`
   * Interface: `outside`
4. `NetFlow Rules`:
   * Rule Name: `RULE-NSEL-ALL`
   * Collector: `NSEL-COLLECTOR-1`
   * Event Types: `Flow Creation`, `Flow Teardown`, `Flow Denied` にチェック。
5. 変更をデプロイ。FTD CLI 上で生成されるコンフィグ構造：

```text
! FTD LINA Engine に生成される CLI コンフィグ
flow-export destination outside 10.200.1.50 2055
class-map NSEL-class
 match any
policy-map platform_policy
 class NSEL-class
  flow-export event-type all destination 10.200.1.50
service-policy platform_policy global
```

#### 検証方法
* FTD CLI 上で `system support diagnostic-cli` に入り、`show flow-export statistics` を実行してエクスポート統計を確認。


---

### サンプル 7: FTD FlexConfig を使用したカスタム FNF 設定

#### 問題
Platform Settings ではサポートされない高度な Flexible NetFlow パラメータ（特定アプリケーション属性のインスペクション等）を FTD に適用するため、FlexConfig を使用してカスタム FNF を設定してください。

#### 要件
* FlexConfig Text Object 名: `obj_netflow_collector` (IP: `10.200.1.50`, Port: `2055`)
* FlexConfig Object 名: `flex_fnf_custom`
* ASA/FTD LINA エンジンに指定の FNF オブジェクトを流し込むこと。

#### 設定例 (FlexConfig Script)
```text
! FlexConfig Object: flex_fnf_custom
flow-export destination inside 10.200.1.50 2055
flow-export template timeout-rate 1
flow-export delay flow-create 0
```

#### 検証方法
* FMC の Deployment Preview で流し込まれるコマンドを確認し、デプロイ完了後 FTD CLI で `show running-config flow-export` を実行して反映を確認。


---

### サンプル 8: Cisco Secure Network Analytics (Stealthwatch) 向け FNF タイマー最適化

#### 問題
Cisco Secure Network Analytics (Stealthwatch) でリアルタイムのアノマリ検知を正確に行うため、ルータ R1 の FNF タイマーを最適化してください。

#### 要件
* アクティブキャッシュタイムアウトを `60` 秒に設定すること（デフォルトの 1800 秒から変更）。
* インアクティブキャッシュタイムアウトを `15` 秒に設定すること。
* テンプレート送信周期を `1` 分（60秒）に設定すること。

#### 設定例
```text
flow exporter EXP-STEALTHWATCH-OPT
 destination 10.1.100.50
 transport udp 2055
 option template-retransmit 60

flow monitor MON-STEALTHWATCH-OPT
 record REC-SECURITY-ANALYSIS
 exporter EXP-STEALTHWATCH-OPT
 cache timeout active 60
 cache timeout inactive 15

interface GigabitEthernet1
 ip flow monitor MON-STEALTHWATCH-OPT input
```

#### 検証方法
* `show flow monitor MON-STEALTHWATCH-OPT` を実行し、`Cache timeout active: 60 secs` および `Cache timeout inactive: 15 secs` になっていることを確認。


---

### サンプル 9: ASA NSEL ディレイタイマーとテンプレートタイムアウトチューニング

#### 問題
大量のショートセッション（HTTP/DNS等）が発生する環境において、ASA の CPU 負荷と NSEL ログ量を削減するため、Flow Creation の送信遅延 (Delay) チューニングを設定してください。

#### 要件
* 短寿命セッション（1秒未満で終了するフロー）の `flow-creation` イベント出力を抑制するため、`delay flow-create` を `1` 秒に設定すること。
* テンプレート再送タイマーを `1` 分に設定すること。

#### 設定例
```text
! 1秒未満で終了する接続の flow-create ログを抑制
flow-export delay flow-create 1

! テンプレート再送インターバルを1分に指定
flow-export template timeout-rate 1
```

#### 検証方法
* 1秒未満で完結する ping 通信を実行した際、`flow-create` イベントが破棄され、`flow-teardown` のみが出力されることを `show flow-export statistics` で確認。


---

### サンプル 10: FNF トラブルシューティング演習 (Unknown Template / Cache Full)

#### 問題
ルータ R1 で FNF を設定したが、コレクタ側で「Unknown Template」エラーが発生し、かつルータ側で `Cache full` 警告が発生しています。原因を特定し、問題を修正してください。

#### 要件・原因解析
* **原因 1**: Exporter に `option template-retransmit` が設定されておらず、コレクタがテンプレートを更新できていない。
* **原因 2**: 大規模トラフィックに対してデフォルトのキャッシュサイズ (4096) が不足し、エントリーが溢れている。

#### 修正設定例
```text
! 1. キャッシュエントリー数の拡張
flow monitor MON-LAB-BASIC
 cache entries 65536
 cache timeout active 60
 cache timeout inactive 15

! 2. テンプレート再送設定の追加
flow exporter EXP-LAB-COLLECTOR
 option template-retransmit 60
```

#### 検証方法
* `show flow monitor MON-LAB-BASIC` を実行し、`Cache entries: 65536` と表示され、`Cache full` カウンタが増加しなくなったことを確認。


## ❓ 想定試験問題

CCIE Security ラボ試験・記述試験を想定した実践的な問題を 5 問掲載します。

### 問題 1 (コンフィグ読解・トラブルシュート)

#### 質問
以下の Cisco IOS-XE ルータのコンフィグレーションを読み、コレクタ側でフローデータが正しく解析されない原因として最も適切なものを1つ選び、その理由を説明しなさい。

```text
flow record REC-CUSTOM
 match ipv4 source address
 match ipv4 destination address
 collect counter bytes

flow exporter EXP-NETFLOW
 destination 10.1.100.50
 transport udp 2055

flow monitor MON-CUSTOM
 record REC-CUSTOM
 exporter EXP-NETFLOW

interface GigabitEthernet1
 ip flow monitor MON-CUSTOM input
```

#### 解答・解説
**【不備と原因】**:
1. **L4 情報 (Protocol / Port) の Match キー欠落**: `match ipv4 protocol` および `match transport source-port` / `destination-port` が定義されていないため、フローを識別する 5-tuple が不完全となり、コレクタ側で標準的なフローデータとして認識・分類できません。
2. **テンプレート再送設定の欠落**: `flow exporter` 内に `option template-retransmit` が設定されていないため、コレクタの再起動時やテンプレート保持期間切れの際に「Unknown Template」としてフローデータが破棄されます。

---

### 問題 2 (NSEL 設計・動作仕様)

#### 質問
Cisco ASA における NetFlow Secure Event Logging (NSEL) の挙動について、従来の Cisco IOS 機器における NetFlow (v5/v9) と比較した説明として**不適切**なものはどれか。

1. NSEL はステートフルファイアウォールのセッションイベント (Create / Teardown / Deny) に同期して即座に出力される。
2. NSEL はアクティブなセッションに対して、一定時間ごとに転送バイト数を中間エクスポートする。
3. NSEL は Modular Policy Framework (MPF) を用いて特定トラフィックのみを選択的に収集対象に指定できる。
4. NSEL のデータフォーマットは NetFlow v9 のテンプレート構文をベースに拡張されている。

#### 解答・解説
**正解: 2**

**【解説】**:
NSEL はイベント駆動型（Event-Driven）のログエクスポートプロトコルであり、接続の確立 (`flow-creation`)、破棄 (`flow-teardown`)、および拒否 (`flow-denied`) のイベント発生時にのみパケットを出力します。従来の NetFlow のように Active Timeout に基づいて中間統計を定期的にエクスポートする挙動は行わないため、選択肢 2 が不適切（誤り）です。

---

### 問題 3 (IPFIX 規格理解)

#### 質問
IPFIX (RFC 5101) が NetFlow v9 に対して持っている上位互換機能・拡張機能に関する説明として、最も適切な組み合わせを選びなさい。

* A. IPFIX はトランスポートプロトコルとして UDP のみに限定されている。
* B. IPFIX は Enterprise Bit (E-bit) を使用してベンダー固有の独自フィールドを定義できる。
* C. IPFIX は可変長フィールド (Variable-length Information Elements) をサポートし、URL 等の文字列を収集できる。
* D. IPFIX はトランスポート層として SCTP (Stream Control Transmission Protocol) や TCP もサポートする。

#### 解答・解説
**正解: B, C, D**

**【解説】**:
IPFIX は IETF によって標準化されたフローエクスポートプロトコルです。NetFlow v9 を拡張し、SCTP や TCP による信頼性の高いトランスポート、可変長フィールドによる動的文字列（URL等）の送信、および Enterprise Bit によるベンダー固有の属性拡張をサポートしています。UDP のみに限定されるという記述 (A) は誤りです。

---

### 問題 4 (FTD/FMC 実装・デザイン)

#### 質問
Firepower Threat Defense (FTD) において、FMC の Platform Settings を使用して NSEL を有効化する場合、正しい適用手順および注意事項を説明しなさい。

#### 解答・解説
1. **手順**: FMC の `Devices` ──► `Platform Settings` にて対象ポリシーを選択し、`NetFlow` セクションで NetFlow Collector (IP, Port, Interface) を定義した上で、`NetFlow Rules` で収集対象イベント (`All`, `Flow Creation`, `Flow Teardown`, `Flow Denied`) とインターフェイスクラスを関連付け、対象 FTD デバイスへ Deploy する。
2. **注意事項**: FTD のデータプレーン（LINA エンジン）で処理される前に Snort エンジンでドロップされた初期パケットや、高度な IPS インスペクション結果ログは NSEL ではなく eStreamer プロトコルを介して FMC へ送信されるため、可視化要件に応じて eStreamer と NSEL を組み合わせる必要があります。

---

### 問題 5 (トラブルシューティング)

#### 質問
ルータの FNF 設定において、大量のトラフィックが流入した際に `show flow monitor` の出力結果で `Cache full` の数値が急増し、新しい接続のフローが収集されなくなりました。この現象を解決するための設定変更を2つ挙げなさい。

#### 解答・解説
1. **フローキャッシュエントリー上限の拡張**: `flow monitor` 設定モードにおいて、`cache entries 65536`（環境に応じて 1024 〜 524288 の範囲）を設定し、メモリ上に保持できる最大フロー数を増やします。
2. **アクティブ・インアクティブタイマーの最適化**: `cache timeout active 60` および `cache timeout inactive 15` を設定し、終了したフローや長期滞留しているフローを迅速にキャッシュから追い出して空きエントリーを確保します。

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術資料の一覧です。

* Cisco IOS-XE Flexible NetFlow Configuration Guide
* Cisco ASA Series NetFlow Secure Event Logging (NSEL) Developer Guide
* Cisco Management Center Configuration Guide
* RFC 5101 - Specification of the IP Flow Information Export (IPFIX) Protocol
* Cisco Live - Network Visibility with NetFlow and IPFIX

---

## 📝 **補足（Notes）**

### NetFlow / NSEL アーキテクチャ構成図

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        NetFlow / IPFIX / NSEL                          │
├───────────────────────────────────┬────────────────────────────────────┤
│   Flexible NetFlow (FNF/IPFIX)    │ NetFlow Secure Event Logging (NSEL)│
│    (Cisco IOS-XE Router/Switch)   │         (Cisco ASA / FTD)          │
├───────────────────────────────────┼────────────────────────────────────┤
│ ・パケットサンプル＆キャッシュ集計 │ ・ステートフルセッションイベント出力│
│ ・Active/Inactive タイマー制御    │ ・Create / Teardown / Denied       │
│ ・フローテーブルの保持            │ ・MPF (Modular Policy Framework)   │
│ ・ネットワーク帯域・利用状況の分析 │ ・高速ファイアウォールログ転送      │
└───────────────────────────────────┴────────────────────────────────────┘
```

### ラボ試験直前チェックリスト

* [ ] FNF の 4 つの構成要素 (`record`, `exporter`, `monitor`, `interface`) の依存関係を暗記しているか？
* [ ] IPFIX の標準エクスポートプロトコル指定 (`export-protocol ipfix`) を記述できるか？
* [ ] ASA / FTD の NSEL で `flow-export destination <if> <ip> <port>` および MPF での `event-type` 指定手順を迷わず実行できるか？
* [ ] テンプレート送信タイマー (`option template-retransmit`) の指定漏れによる Unknown Template エラーを予防できるか？
* [ ] Stealthwatch (Secure Network Analytics) 連携時の推奨タイマー（Active Timeout 60秒）を即座に設定できるか？


