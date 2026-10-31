---
layout: default
title: 1.5.b-Passive
nav_order: 2
parent: 1.5-NGIPS-deployment-modes
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.5.b Passive

Cisco Secure Firewall Threat Defense (FTD) および Firepower NGIPS における **Passive Mode（パッシブモード / アラート・検知専用モード）** は、ネットワークのインライン（通信直列線上）に装置を配置するのではなく、スイッチの SPAN (Switched Port Analyzer)、RSPAN、ERSPAN、あるいは物理 Network TAP デバイス等からパケットの「コピー」を受信し、プロミスキャスモード（Promiscuous mode）で動作するインターフェイス上で Snort 侵入検知エンジンを駆動させて帯域外（Out-of-band）で脅威監視・可視化を行う導入モードです。

CCIE Security v6.1 ラボ試験では、通常の **Standard Passive Mode (SPAN/TAP 経由)** と **ERSPAN Passive Mode (GRE トンネル集約経由)** の動作仕様の違い、ファイアウォールモード（Routed Mode vs Transparent Mode）の前提条件、レイヤ 3〜7 における制限事項、スイッチ側の SPAN/ERSPAN 構成との連携、およびイベント検知時のトラブルシューティングが厳格に問われます。

---

## 📘 概要

* **機能概要**:
  * スイッチのポートミラーリング（SPAN/RSPAN）、GRE トンネルによるカプセル化転送（ERSPAN）、または物理 TAP デバイスからネットワークパケットのコピーを取得し、プロミスキャスモードに設定された受信用物理ポート（または EtherChannel）で受信して Snort ディープパケットインスペクションを実行する検知専用モード。
  * 受信したパケットは FTD 内部で処理・解析のみが行われ、対向ポートへ再送信（Retransmit）やルーティング出力されることは一切ありません。
* **利用目的**:
  * 既存のネットワークトポロジ、IP アドレッシング、ルーティング設計、およびスイッチング構成に変更を加えることなく、ネットワーク内の通信の可視化（アプリケーション、ユーザー、ホスト、脆弱性）と脅威検知（IDS）を実現する。
  * 通信遅延（Latency）の追加や、機器障害・処理遅延による通信遮断（Packet Drop）リスクをゼロに抑えたセキュリティモニタリング基盤の構築。
* **どのような場面で利用するか**:
  * わずかな通信遅延やパケットドロップも許容されないミッションクリティカルな環境（金融市場取引、医療機器ネットワーク、産業用制御システム: ICS/SCADA 等）。
  * NGIPS 導入時の初期フェーズにおいて、シグネチャの誤検知（False Positive）リスクやトラフィック特性、FMC での可視化効果を事前評価・チューニングする評価フェーズ。
  * 複数拠点・分散スイッチのトラフィックを L3 ネットワーク越しに 1 台の FTD へ集約して統合監視する環境（ERSPAN モード）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **配置方式** | 帯域外 (Out-of-band / Promiscuous Mode) |
| **データ供給源** | Switch Local SPAN, RSPAN, ERSPAN (GRE Tunnel), 物理 Network TAP |
| **パケット制御能力** | **ドロップ不可** (コピーパケットを解析するため通信の直接遮断・帯域制御・QoS は不可) |
| **TCP Reset (能動応答)** | オプションで TCP Reset 送信可能 (ただし対向スイッチの受信用ポート構造や応答速度に依存し完全遮断は非保証) |
| **IP アドレス設定** | **Standard Passive**: 不要 (IP アドレス非依存)<br>**ERSPAN Passive**: **GRE 終端用 IP アドレス設定が必須** |
| **Firewall Mode 要件** | **Standard Passive**: Routed Mode / Transparent Mode 双方でサポート<br>**ERSPAN Passive**: **Routed Mode 専用** (Transparent Mode では利用不可) |
| **インターフェイス制限** | 物理ポートおよび EtherChannel のみサポート。Subinterface、Redundant Interface、FXOS Subinterface は使用不可 |
| **無効化される L4-L7 機能** | Routing, NAT, VPN, DHCP (Server/Relay/Client), TCP Intercept, QoS, NetFlow, Application Inspection |

---

## 🏗 動作原理

Passive モードでは、スイッチや TAP から送信されたトラフィックのコピーが受信用 Passive インターフェイスに着信し、LINA エンジンの最小限のチェックを経て Snort エンジンへ回送されます。

```text
1. Standard Passive Deployment (Local SPAN / TAP)
   [ Client ] ──────────────────────────────────────────────► [ Server ]
                       │
                       │ (Mirrored Packet Copy via Switch SPAN / TAP)
                       ▼
         ┌───────────────────────────┐
         │ FTD Passive Interface     │ (Promiscuous Mode: Recv Only)
         └─────────────┬─────────────┘
                       │
                       ▼ (Partial LINA & DAQ Engine)
         ┌───────────────────────────┐
         │ Snort Engine              │ (Protocol Normalization, Stream Reassembly, IPS Rules)
         └─────────────┬─────────────┘
                       │
                       ▼
         ┌───────────────────────────┐
         │ Event Generation          │ (Intrusion Alerts, Connection Events, Host Discovery)
         └───────────────────────────┘

2. ERSPAN Passive Deployment (Remote L3 Aggregation)
   [ Branch Client ] ──► [ Switch (ERSPAN Source) ] ──► ( L3 GRE Network ) ──► [ FTD (ERSPAN Dest) ]
                                                                                   │ (Decapsulate GRE)
                                                                                   ▼
                                                                            [ Snort Engine ]
```

---

## ⚙ 動作シーケンス

1. **トラフィックのミラーリング (Traffic Mirroring)**:
   * スイッチ（Catalyst / Nexus 等）が指定されたアクセスポートや Trunk ポート、VLAN の着信・発信パケットをコピーし、受信用 SPAN ポート（または ERSPAN GRE トンネル）へ送出します。
2. **プロミスキャス受容 (Promiscuous Ingress)**:
   * FTD の Passive モード用物理ポートがプロミスキャスモードで作動し、自身の MAC アドレス宛てでない全フレームを無条件でドロップせずにキャプチャします。
3. **LINA バイパスおよび DAQ 転送**:
   * パケットは通常のファイアウォール処理（コネクション状態追跡、NAT 変換、ACL 評価、ルーティング検索）をバイパスし、LINA 経由で DAQ (Data Acquisition) レイヤーを通って直接 Snort インスペクションエンジンへ渡されます。
4. **Snort インスペクションパイプライン**:
   * **デフラグ ＆ 再構築**: IP フラグメントの結合および TCP ストリームの再構築を実施。
   * **プロトコル正規化 (Normalization)**: HTTP、DNS、TLS、FTP などのアプリケーションレイヤのデコード。
   * **シグネチャ照合**: 設定された Access Control Policy (ACP) 配下の Intrusion Policy（侵入防御ルール）とパケットを照合。
5. **アラート出力 (Event Output)**:
   * シグネチャの定義が `Drop and Generate Events` や `Block` であっても、対象パケットは本線上の通信ではないコピーデータであるためドロップ処理は物理的に発生せず、FMC へ `Intrusion Event` および `Connection Event` が出力・記録されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **ERSPAN Passive モードと Firewall Mode の不整合トラブル**:
   * **よくある不合格パターン**: 出題要件で「Configure ERSPAN Passive interface to monitor remote branch traffic」と指定されているにもかかわらず、FTD の Firewall Mode がデフォルトの `Transparent Mode` のまま設定を進めようとしてエラーとなる。
   * **対策**: **ERSPAN モードは `Routed Firewall Mode` でのみサポート** される点を暗記し、設定前に FTD の Firewall Mode を確認・構成します。
2. **Passive インターフェイスにおける Block ルールのアクション理解**:
   * Access Control Policy や Intrusion Policy でルールアクションを `Block` または `Drop and Generate Events` にアサインして Deploy しても、実際のオリジナルパケットは遮断されません。
   * 試験で「Ensure suspicious traffic is logged without disrupting user traffic」と要求された場合、Passive モードを選択するのが模範解答です。
3. **サポートされる物理/論理ポートタイプの識別**:
   * Passive モードに割り当て可能なのは **「物理ポート」** および **「EtherChannel」** のみです。
   * **不可**: Subinterface (802.1Q ポート)、Redundant Interface、FXOS Logical Device の Subinterface ポートはアサインできません。
4. **スイッチ側 (Catalyst/Nexus) の SPAN / ERSPAN 設定の完全性**:
   * FTD 側で Passive ポートを適切に構成しても、対向スイッチ側で `monitor session` の設定が誤っているとパケットが届きません。
   * スイッチ側で `destination interface` の指定漏れ、あるいは `encapsulation replicate` オプションの失念により VLAN タグ情報が喪失する問題が出題されます。
5. **検証コマンドにおける Rx カウントと Tx カウントの判定**:
   * 正常に動作している Passive インターフェイスでは、`show interface <Name>` コマンドで **`rx_packets` のみが連続的に増加し、`tx_packets` は 0（または Pause フレームなどのフロー制御のみ）** となります。

---

## 🛠 設定方法

### 1. FMC GUI: Standard Passive Interface (SPAN 用) の構成

```text
[ Step 1: 物理インターフェイスの Passive 化 ]
1. FMC UI から Devices > Device Management に移動し、対象 FTD の編集 (鉛筆アイコン) を選択。
2. Interfaces タブを開き、対象の物理ポート (例: GigabitEthernet0/2) を編集。
3. General タブ:
   - Name: PASSIVE_SPAN_IN
   - Enabled: [✓] チェック (インターフェイスの有効化)
   - Mode: "Passive" を選択
   - Security Zone: PASSIVE_ZONE (新規作成または既存アサイン)
   - MTU: 1500
4. "OK" をクリック。
5. "Save" をクリックし、変更を対象 FTD へ Deploy。
```

---

### 2. FMC GUI: ERSPAN Passive Interface (GRE 終端用) の構成

```text
[ Step 1: 事前確認 ]
1. FTD が Routed Firewall Mode であることを確認 (Devices > Device Management の Mode 列)。

[ Step 2: ERSPAN インターフェイスの構成 ]
1. Devices > Device Management > Interfaces タブを開く。
2. 対象の物理ポート (例: GigabitEthernet0/3) を編集:
   - Name: ERSPAN_MONITOR_IN
   - Enabled: [✓] チェック
   - Mode: "Erspan" を選択
   - Security Zone: ERSPAN_ZONE を選択
3. Erspan タブ (または設定項目):
   - ERSPAN ID: 100 (対向スイッチの Flow ID と一致させる)
   - IP Address: 172.16.10.50 /24 (GRE トンネル終端用 IP アドレス)
4. "OK" ➔ "Save" をクリックし、対象 FTD へ Deploy。
```

---

### 3. 対向 Catalyst スイッチ側 (IOS-XE) の SPAN / ERSPAN 設定例

#### (A) Local SPAN 設定例 (Standard Passive 接続用)

```bash
! ===== Local SPAN 構成 (Gi1/0/1 と Gi1/0/2 の通信を FTD 接続ポート Gi1/0/10 へコピー) =====
monitor session 1 source interface GigabitEthernet1/0/1 both
monitor session 1 source interface GigabitEthernet1/0/2 both
monitor session 1 destination interface GigabitEthernet1/0/10 encapsulation replicate
```

#### (B) ERSPAN Source 設定例 (ERSPAN Passive 接続用)

```bash
! ===== ERSPAN Source 構成 (キャプチャ側スイッチ) =====
monitor session 2 type erspan-source
 source interface GigabitEthernet1/0/5 both
 destination
  erspan-id 100
  ip precision 6
  ip address 172.16.10.50
  origin ip address 172.16.10.1
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI) | 表示確認ポイント |
| :--- | :--- | :--- |
| **Passive インターフェイスの動作モード・ステータス確認** | <code>show interface GigabitEthernet 0/2</code> | `IPS Interface-Mode: Passive`、`Enabled`、`Promiscuous Mode` の確認 |
| **受信パケットカウントの増加確認** | <code>show interface GigabitEthernet 0/2 \| include packets</code> | `rx_packets` の推移（増加）と `tx_packets` が 0（送信なし）であること |
| **ERSPAN GRE トンネルの着信確認** | <code>show interface GigabitEthernet 0/3</code> | `IPS Interface-Mode: Erspan`、割り当てられた IP アドレスおよび `ERSPAN ID: 100` |
| **LINA データプレーンでの受容パケットキャプチャ** | <code>capture CAP_PASSIVE interface PASSIVE_SPAN_IN detail</code> | スイッチから送られてくるコピーパケットが正常に着信しているか表示 |
| **Snort エンジンへのパケット渡りとイベント評価デバッグ** | <code>system support firewall-engine-debug</code> | 受信パケットが Snort ルールと照合され、Intrusion Alert が生成されるログ |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **FMC 上で Mode に "Erspan" が選択できない/グレーアウトする** | FTD アプライアンスが `Transparent Firewall Mode` で稼働している | <code>show running-config firewall</code> | ERSPAN は Routed Mode 専用機能。FTD の Mode を Routed Mode へ変更する。 |
| **Passive インターフェイスを有効化したがイベントが一切検出されない** | ① 対向スイッチの SPAN 設定漏れ<br>② スイッチポートの物理 Cable 不良<br>③ インターフェイスの Enabled チェック漏れ | <code>show interface <IF></code><br>Switch: <code>show monitor</code> | スイッチ側で `show monitor session` を確認し、FTD 側で `Enabled` にチェックを入れて Deploy。 |
| **VLAN タグ付きパケットが Passive ポートに着信しても Snort で解読されない** | スイッチ側の SPAN 設定で `encapsulation replicate` が設定されておらず、802.1Q ヘッダーが剥がされている | Switch: <code>show monitor session</code> | スイッチの SPAN destination 設定に `encapsulation replicate` を追加。 |
| **Passive インターフェイスに NAT や Routing ポリシーを適用しようとして Deploy エラーになる** | IPS-Only ポート (Passive/Inline) では L3/L4 ファイアウォール機能はサポートされない | FMC Deploy Status Log | Passive インターフェイスを NAT/Routing/VPN/QoS などのポリシー設定オブジェクトから除外する。 |
| **高トラフィック環境で FTD の CPU 使用率が急上昇しパケットドロップが発生する (ERSPAN 時)** | FTD は GRE カプセル化パケットを複数 Rx リングにロードバランスできない仕様であるため | <code>show cpu</code><br><code>show asp drop</code> | 1 台の FTD 宛ての ERSPAN トラフィック量を制限するか、フィルタリングで対象を最小化する。 |

---

## ⚠ 制限事項

* **非サポートのファイアウォール機能 (IPS-Only 制限)**:
  * Passive インターフェイス（および ERSPAN Passive）上では、以下のファイアウォール機能およびネットワークサービスは一切機能せず、適用しようとするとエラーとなります。
    * Routing (Static/Dynamic)、NAT、VPN (Site-to-Site/Remote Access)、DHCP (Server/Relay/Client)、TCP Intercept、QoS、NetFlow、VXLAN、Application Inspection。
* **物理/論理ポートの制限**:
  * Passive モードをアサインできるのは **単一の物理ポート** および **EtherChannel** のみです。
  * Redundant Interface、VLAN Subinterface、FXOS Logical Device の Subinterface 等は使用できません。
* **802.1Q ヘッダー (VLAN Tag) 処理の限界**:
  * FTD 2100/1000 シリーズ等では最大 2 つまでの 802.1Q ヘッダー (Q-in-Q) をパケット内で保持・検査できますが、**Firepower 4100 / 9300 シリーズは 1 つの 802.1Q ヘッダーのみサポート** されます。
* **ERSPAN 固有の制限事項**:
  * **Routed Firewall Mode でのみ使用可能**（Transparent Mode 不可）。
  * FTD は ERSPAN (GRE) トラフィックを複数 CPU / Rx リングに負荷分散（Load-balance）できないため、過大な ERSPAN パケット着信は FTD のパフォーマンス低下およびドロップの原因となります。

---

## 🔄 他技術との関連

* **Switched Port Analyzer (SPAN / RSPAN / ERSPAN)**: 対向スイッチ側でトラフィックをコピーして受信用 Passive ポートへ届けるための L2/L3 送信基盤技術。
* **Access Control Policy (ACP) ＆ Intrusion Policy**: Passive インターフェイスに着信したパケットに対して適用するセキュリティポリシー。アクションを Block にしてもアラートログ生成のみが行われる。
* **Network Discovery (Host / User / Application Profiling)**: Passive ポートから着信した通信パターンを解析し、FMC 上で自動的にネットワーク内の OS、サービス、アプリケーション、ユーザー情報をマッピング・データベース化する機能。

---

## 🧩 比較表

### 1. Standard Passive vs ERSPAN Passive

| 比較項目 | Standard Passive Mode | ERSPAN Passive Mode |
| :--- | :--- | :--- |
| **通信物理距離** | 同一スイッチ内 (Local SPAN) または物理 TAP 直結 | L3 ネットワーク越しの遠隔地スイッチから集約可能 |
| **カプセル化方式** | 生のイーサネットフレーム (Raw Frame) | **GRE (Generic Routing Encapsulation) トンネル** |
| **IP アドレス要件** | **不要** (IP なし) | **GRE 終端用 IP アドレスの設定が必須** |
| **Firewall Mode** | **Routed / Transparent 両対応** | **Routed Mode 専用** (Transparent 不可) |
| **処理負荷・制限** | 標準的な Snort 処理負荷 | GRE デカプセル化処理が必要。Rx 負荷分散非対応による制限あり |

### 2. Passive Mode vs In-line Mode vs Inline Tap Mode

| 比較項目 | Passive Mode | In-line Mode | Inline Tap Mode |
| :--- | :--- | :--- | :--- |
| **配置位置** | 帯域外 (Out-of-band) | インライン (直列本線) | 物理インライン / 内部コピー処理 |
| **必要ポート数** | 最低 **1 ポート** | 最低 **2 ポート** (Pair) | 最低 **2 ポート** (Pair) |
| **脅威の直接遮断** | **不可** (検知・アラートのみ) | **可能** (Drop, Reset) | **不可** (`Would Have Dropped` 表示) |
| **本線遅延・影響** | **完全ゼロ** (本線非関与) | 微小なインスペクション遅延発生 | **完全ゼロ** (即時転送) |
| **スイッチ側要件** | **SPAN / TAP 設定が必要** | 不要 (配線のみ) | 不要 (配線のみ) |

---

## 💡 ベストプラクティス

1. **スイッチ SPAN ポートの帯域設計とドロップ対策**:
   * スイッチ側で複数の GbE ポートを 1 つの Passive モード用 1GbE ポートに SPAN すると、トラフィックの合計が 1Gbps を超えた段階でスイッチの SPAN ポート上でパケットドロップが発生し、FTD での可視性が阻害されます。過密環境では **EtherChannel による Passive ポートの帯域拡張** または **物理 Network TAP** を採用します。
2. **ERSPAN 適用時の GRE 負荷低減策**:
   * ERSPAN を使用する場合、FTD 側の GRE 処理負荷を抑えるため、送信側スイッチ（ERSPAN Source）で `ip access-group` 等を用いて不必要なトラフィック（例: バックアップデータ等）を事前にフィルタリングした上で FTD へ送出します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Standard Passive Interface (SPAN) の基本設定

#### 問題
FTD の `GigabitEthernet0/2` ポートを Standard Passive モードとして構成し、Security Zone `ZONE_PASSIVE` にアサインせよ。

#### 要件
* FMC を使用して設定すること。
* IP アドレスはアサインしないこと。

#### 設定・操作手順

```text
1. FMC UI で Devices > Device Management を開く。
2. 対象 FTD の Interfaces タブを選択し GigabitEthernet0/2 を編集:
   - Name: Gi0/2_PASSIVE
   - Mode: Passive
   - Enabled: [✓] チェック
   - Security Zone: ZONE_PASSIVE (新規作成)
3. OK ➔ Save をクリックし、FTD へ Deploy。
```

#### 検証方法
* FTD CLI で `show interface GigabitEthernet0/2` を実行し、`IPS Interface-Mode: Passive` と表示され、対向スイッチから SPAN パケット送信時に `rx_packets` のみが増加することを確認。

---

### 例2: 対向 Catalyst スイッチでの Local SPAN 設定

#### 問題
Catalyst スイッチ (IOS-XE) において、`GigabitEthernet1/0/1` (Client ポート) の全通信（Ingress/Egress）をコピーし、FTD の Passive ポートに接続されている `GigabitEthernet1/0/10` へ送信する SPAN セッションを構成せよ。

#### 要件
* 802.1Q VLAN タグを保持して送出すること (`encapsulation replicate`)。

#### 設定・操作手順

```bash
Switch(config)# monitor session 1 source interface GigabitEthernet1/0/1 both
Switch(config)# monitor session 1 destination interface GigabitEthernet1/0/10 encapsulation replicate
```

#### 検証方法
* スイッチで `show monitor session 1` を実行し、Source Ports、Destination Port、および `Encapsulation: Replicate` が正常に反映されていることを確認。

---

### 例3: ERSPAN Passive Interface (GRE 終端) の基本構成

#### 問題
FTD (`GigabitEthernet0/3`) 上に ERSPAN 受信用インターフェイスを構成せよ。

#### 要件
* ERSPAN ID: `200`
* GRE 終端用 IP アドレス: `192.168.200.50/24`
* Security Zone: `ZONE_ERSPAN`

#### 設定・操作手順

```text
1. FTD が Routed Firewall Mode であることを確認。
2. Devices > Device Management > Interfaces で GigabitEthernet0/3 を編集:
   - Name: Gi0/3_ERSPAN
   - Mode: Erspan
   - Enabled: [✓]
   - Security Zone: ZONE_ERSPAN
3. Erspan タブ:
   - ERSPAN ID: 200
   - IP Address: 192.168.200.50 /24
4. OK ➔ Save をクリックして Deploy。
```

#### 検証方法
* FTD CLI で `show interface GigabitEthernet0/3` を実行し、`IPS Interface-Mode: Erspan` および IP アドレス `192.168.200.50` が設定されていることを確認。

---

### 例4: 対向 Catalyst スイッチでの ERSPAN Source セッション設定

#### 問題
遠隔地 Catalyst スイッチにおいて、`VLAN 100` のトラフィックをコピーし、FTD の ERSPAN IP (`192.168.200.50`) 宛てに送信する ERSPAN Source セッションを構築せよ。

#### 要件
* ERSPAN ID: `200`
* Origin IP (送信元 IP): `192.168.200.1`

#### 設定・操作手順

```bash
Switch(config)# monitor session 2 type erspan-source
Switch(config-mon-erspan-src)# source vlan 100 both
Switch(config-mon-erspan-src)# destination
Switch(config-mon-erspan-src-dst)# erspan-id 200
Switch(config-mon-erspan-src-dst)# ip address 192.168.200.50
Switch(config-mon-erspan-src-dst)# origin ip address 192.168.200.1
```

#### 検証方法
* スイッチで `show monitor session 2` を確認し、`Status : Admin Enable` となっていることを検証。

---

### 例5: EtherChannel を用いた Passive インターフェイスの多重化

#### 問題
大容量トラフィックをキャプチャするため、FTD の 2 つの物理ポート (`Gi0/4`, `Gi0/5`) をバインドした EtherChannel (`Port-Channel 1`) を作成し、Passive モードとしてアサインせよ。

#### 要件
* インターフェイスモード: `Passive`

#### 設定・操作手順

```text
1. Devices > Device Management > Interfaces で "Add EtherChannel" を選択。
2. General タブ:
   - Name: PC1_PASSIVE
   - Interface ID: 1
   - Mode: Passive
   - Available Interfaces から GigabitEthernet0/4 と GigabitEthernet0/5 を選択して追加。
   - Security Zone: ZONE_PASSIVE_LAG
3. OK ➔ Save をクリックして Deploy。
```

#### 検証方法
* FTD CLI で `show interface Port-Channel1` を実行し、`IPS Interface-Mode: Passive` であることを確認。

---

### 例6: Passive インターフェイスでの Intrusion Policy アクション挙動の検証

#### 問題
Passive インターフェイス配下のトラフィックに対し、特定の侵入ルール（SID: `2000100`）のアクションを `Drop and Generate Events` に設定した際のログ出力を検証せよ。

#### 要件
* 通信自体はドロップされず、FMC にアラートが出力されることを確認すること。

#### 設定・操作手順

```text
1. FMC UI で Access Control Policy を編集し、PASSIVE_ZONE からのトラフィックに Intrusion Policy をバインド。
2. Intrusion Policy で Rule 2000100 を "Drop and Generate Events" に設定して Deploy。
3. クライアントから攻撃トラフィックを発生させる。
```

#### 検証方法
* クライアントの通信が遮断されずに正常完了することを確認。
* FMC の `Analysis > Intrusions > Events` 画面を開き、該当イベントが生成され、Reason に破棄されなかった旨が記録されていることを検証。

---

### 例7: Transparent Mode での ERSPAN 設定不可（エラー）の回避

#### 問題
Transparent Firewall Mode で動作している FTD において ERSPAN インターフェイスを設定しようとして発生したエラーを解決せよ。

#### 要件
* ERSPAN を使用するために適切な構成変更を行うこと。

#### 設定・操作手順

```text
1. Devices > Device Management に移動。
2. 対象 FTD の Mode カラムで Transparent を Routed Mode へ変更 (または初期セットアップで Routed を選択)。
3. その後、Interfaces で Mode: Erspan を設定して Deploy。
```

#### 検証方法
* FMC で Deploy がエラーなく正常完了することを確認。

---

### 例8: LINA キャプチャ機能を使用した Passive パケット着信のトラブルシューティング

#### 問題
FTD の Passive ポート `Gi0/2_PASSIVE` にスイッチからパケットが正しく届いているか、LINA CLI キャプチャツールを使用して確認せよ。

#### 要件
* FTD CLI で `capture` コマンドを使用すること。

#### 設定・操作手順

```bash
> capture CAP_TEST interface PASSIVE_SPAN_IN match ip any any
> show capture CAP_TEST
```

#### 検証方法
* `show capture CAP_TEST` の出力で、対向スイッチから送信された様々なパケットのフレームカウントが増加していることを確認。

---

### 例9: Snort リアルタイムデバッグによる Passive パケット処理追跡

#### 問題
FTD CLI のデバッグツールを使用して、Passive ポートから入力された特定 IP (`10.1.1.100`) のパケットが Snort エンジンで処理されているか追跡せよ。

#### 要件
* `system support firewall-engine-debug` を使用すること。

#### 設定・操作手順

```bash
> system support firewall-engine-debug

Please specify an IP protocol: icmp
Please specify a client IP address: 10.1.1.100
Please specify a server IP address:
```

#### 検証方法
* クライアントから ICMP パケットを送出させた際、FTD CLI 上に Snort の評価ログがリアルタイム表示されることを確認。

---

### 例10: Network Discovery (Host Profiling) と Passive インターフェイスの連携

#### 問題
Passive インターフェイス `Gi0/2_PASSIVE` から受信したトラフィックを自動解析し、FMC の Network Discovery データベースへホスト OS およびアプリケーション情報を自動登録させよ。

#### 要件
* `Policies > Network Discovery` 内でルールを有効化すること。

#### 設定・操作手順

```text
1. FMC UI で Policies > Network Discovery を開く。
2. ディスカバリルールで PASSIVE_ZONE からのトラフィックに対して "Discover Applications and Hosts" が有効になっていることを確認。
3. ACP を Deploy。
```

#### 検証方法
* FMC の `Analysis > Hosts > Discovered Hosts` 画面を開き、Passive ポート経由で検出されたネットワーク内の端末の OS、MAC アドレス、稼働サービスがマッピング表示されることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で FTD の物理ポート `Gi0/3` の Mode を `Erspan` に変更し、必要な IP アドレスと ERSPAN ID を設定して Deploy を試みたところ、FMC の Deployment 画面で「ERSPAN interfaces are only supported in routed firewall mode」というエラーメッセージが表示されて失敗した。原因として正しいものはどれか。

* A) ERSPAN インターフェイスにアサインした IP アドレスが重複しているため。
* B) 対象の FTD アプライアンスが `Transparent Firewall Mode` で構成されているため。
* C) FTD 上で Threat ライセンスが有効化されていないため。
* D) ERSPAN ID が 100 を超えているため。

**正解**: **B**  
**解説**: **ERSPAN Passive インターフェイスは `Routed Firewall Mode` で稼働している FTD でのみサポート** されます。Transparent Mode の FTD 上で ERSPAN ポートを構成することはできず、Deploy 時にバリデーションエラーが発生します。

---

### 2. トラブルシュート問題

**問題**: 新しく配置した FTD の Passive インターフェイスにおいて、Access Control Policy 内の特定の侵入ルールアクションを `Block` に設定した。しかし、対象の攻撃トラフィックを発生させても、ターゲットホストへのパケットは遮断されず通過してしまう。FMC のイベントビューアには該当の侵入アラートログが記録されている。この挙動の原因として正しいものはどれか。

* A) FMC ↔ FTD 間の sftunnel が切断されているため。
* B) FTD の Snort エンジンが停止しているため。
* C) Passive モードはスイッチから送信されたパケットの「コピー」を検査する帯域外モニタリングモードであり、本線パケットを直接破棄・遮断する能力を持たないため。
* D) FTD のライセンスが Base にダウングレードしているため。

**正解**: **C**  
**解説**: **Passive モード** は、スイッチの SPAN や TAP から送られてくる「パケットのコピー」を解析する検出専用モードです。そのため、ルールのアクションを `Block` や `Drop` に設定しても、本線上のオリジナルパケットをドロップすることはできず、アラートイベントのみが記録されます。

---

### 3. Design 問題

**問題**: 各地に分散する 5 つのブランチオフィスの Catalyst スイッチから、L3 ネットワークを越えてトラフィックのコピーを本社の FTD へ集約し、社内規定違反アプリケーションの利用状況を統合モニタリングしたい。本社の FTD で採用すべき適切なインターフェイスモードおよび設計要件の組み合わせはどれか。

* A) Standard Passive モード (Transparent Firewall Mode)
* B) ERSPAN Passive モード (Routed Firewall Mode)
* C) Inline Tap モード (Multi-Instance Mode)
* D) Standard Passive モード (EtherChannel Subinterface)

**正解**: **B**  
**解説**: 遠隔地のスイッチから L3 ネットワーク（GRE カプセル化）を介してパケットコピーを集約・受信する方式は **ERSPAN Passive モード** です。また、ERSPAN は **`Routed Firewall Mode`** でのみ動作します。

---

### 4. 実装問題

**問題**: FTD CLI にログインし、設定した Passive インターフェイス (`Gi0/2`) がプロミスキャスモードで作動し、正しく対向スイッチからパケットを受信できているか状態を検証するためのコマンドはどれか。

* A) <code>show access-list</code>
* B) <code>show interface GigabitEthernet0/2</code>
* C) <code>show inline-set</code>
* D) <code>show service-policy</code>

**正解**: **B**  
**解説**: FTD CLI で **`show interface <Interface_Name>`** を実行すると、該当インターフェイスの物理ステータス、`IPS Interface-Mode: Passive`、プロミスキャスモードの有効化状態、および受信パケットカウント (`rx_packets`) の推移を一括で確認できます。

---

### 5. コンフィグ読解問題

**問題**: 対向 Catalyst スイッチで以下のコンフィグを投入し、FTD の Passive ポートに接続した。FTD 側で受信用ポートを構成する際、適切なモードおよび注意事項の組み合わせはどれか。

```bash
Switch(config)# monitor session 1 source interface Gi1/0/1 both
Switch(config)# monitor session 1 destination interface Gi1/0/10 encapsulation replicate
```

* A) Mode: Erspan を選択し、GRE IP アドレスを設定する。
* B) Mode: Passive を選択し、IP アドレスは設定しない (SPAN による生フレーム受信のため)。
* C) Mode: Inline Pair を選択し、Gi1/0/10 とペアを形成する。
* D) Mode: Routed を選択し、Default Gateway を設定する。

**正解**: **B**  
**解説**: スイッチ側のコンフィグはローカル SPAN (Local SPAN) の設定です。したがって、FTD 側では **Standard Passive モード (Mode: Passive)** を選択し、IP アドレスを設定せずに構成します。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Threat Defense Configuration Guide - Passive Interfaces](https://www.cisco.com/)
* [CCNP Security Cisco Secure Firewall and Intrusion Prevention System Official Cert Guide (Chapter 7)](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot Passive and ERSPAN Interfaces on FTD](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Cisco Secure Firewall Deployment Modes (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * ラボ試験で「Monitor traffic from remote switches across Layer 3 without impacting production traffic」と指示された場合は、迷わず **`Routed Mode` + `ERSPAN Passive Interface`** を選択します。
* **図解（Passive Mode と他モードの選択決定ツリー）**:
  ```text
  [ Monitoring / Inspection Requirement ]
                     │
                     ├── Need to Block Threats In-line?
                     │         ├── YES ──► In-line Mode (Inline Set)
                     │         └── NO
                     │              │
                     │              ├── Traffic sourced across L3 Network?
                     │              │         ├── YES ──► ERSPAN Passive Mode (Routed Firewall Mode Required)
                     │              │         └── NO  ──► Standard Passive Mode (Local SPAN / TAP)
  ```
* **注意点**:
  * 仮想環境（FTDv / NGIPSv）で Standard Passive モードを構築する場合、ハイパーバイザ（ESXi 等）の Port Group 設定で **`Promiscuous Mode: Accept`** を有効化しないと、プロミスキャスフレームが FTD 仮想 NIC に到達しません。実機・仮想環境ともにプロミスキャス状態の事前確認を徹底してください。
