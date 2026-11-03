---
layout: default
title: 3.4.c-STP-security
nav_order: 3
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.4.c STP security

## 📘 概要

スパニングツリープロトコル（STP: Spanning Tree Protocol）は、レイヤー2ネットワークにおけるループ構成を自動的に検知・排除し、冗長性を確保するための標準規格です。しかし、標準のSTP（IEEE 802.1D / 802.1w Rapid STP / 802.1s Multiple STP）はデフォルトで相互認証メカニズムを持たないため、不当なBPDU（Bridge Protocol Data Unit）の注入や物理層・データリンク層の片方向障害に対して脆弱性を抱えています。

STP Security（STP保護技術）は、不当なスイッチ接続によるRoot Bridge権限の奪取、アクセスポートからのループ発生、片方向リンク障害によるフォワーディングループ、およびBPDU枯渇・過多攻撃からレイヤー2インフラストラクチャを防御するための標準的セキュリティメカニズム群です。

主な利用場面および防御機能：
* **BPDU Guard**: エンドポイント（PC、サーバー等）が接続されるアクセスポートにBPDUが着信した際、即座にポートを `err-disable` 状態にしてアクセス層からの意図しないスイッチ接続やループを遮断します。
* **BPDU Filter**: 特定ポートでのBPDU送受信を完全に停止、またはPortFastポートでの不正BPDU受信時にPortFast属性を解除して通常のSTP動作へ移行させます。
* **Root Guard**: 指定したポート（通常は配布層・コア層からアクセス層に向けたダウンリンク）が上位のBPDUを受信した際、該当ポートを `root-inconsistent` 状態に移行させ、意図しない外部スイッチがRoot Bridgeになることを防ぎます。
* **Loop Guard**: BPDUの受信が不図示の理由（ソフトウェアビジー、片方向通信障害等）で途絶えた際、通常であればBlockingポートがDesignatedポートへ移行してループが発生するのを防ぐため、ポートを `loop-inconsistent` 状態に遷移させてブロック状態を維持します。
* **UDLD (Unidirectional Link Detection)**: 光ファイバーやツイストペアケーブルの物理的・論理的な片方向リンク障害をレイヤー2レベルで検知し、対向ポートをシャットダウン（`err-disable`）してルータやスイッチ間のブラックホールやSTPループを防ぎます。
* **Dispute Mechanism**: 802.1w (RSTP) / 802.1s (MST) において、Duplexミスマッチや一方的なDesignated状態の宣言といった矛盾（Dispute）を検出した際、ポートを即座にDispute/Blocking状態にしてループを防止します。

---

## 🔑 要点

STP Securityを構成する主要機能の比較と要約は以下の通りです。

| 機能 | 特徴 | 主な用途 | メリット | デメリット / 注意点 | 対応機種 | 制限事項 / 設計上の注意点 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **BPDU Guard** | PortFast有効ポートでBPDUを受信するとポートを `err-disable` 化 | アクセスポート（エンドユーザー接続ポート） | 不正スイッチやハブ接続によるループを即座に完全遮断 | 設定ミスにより正規スイッチ間リンクが遮断されるリスク | Catalyst全般, Nexus, Cisco IOS/IOS-XE, NX-OS | PortFastと組み合わせて使用。グローバル設定とインターフェイス設定で挙動が異なる |
| **BPDU Filter** | ポートでのBPDU送信および受信の抑止 | サーバ直結ポートや特定テスト環境 | 不要なBPDUの送信を抑止しCPU負荷軽減 | BPDUを無視するためループ検出不能になり大惨事（ループ発生）を招く危険性 | Catalyst全般, Nexus, Cisco IOS/IOS-XE, NX-OS | インターフェイス直接設定時はBPDU送受信を完全停止するため原則非推奨。グローバル設定を推奨 |
| **Root Guard** | 該当ポート経由でのRoot Bridge昇格を拒否 (`root-inconsistent`) | コア/分布層からアクセス層への配下ポート | ネットワークのRoot Bridgeトポロジの予期せぬ変動を保護 | 下位スイッチがRootになれなくなるため適切な階層設計が必須 | Catalyst全般, Nexus, Cisco IOS/IOS-XE, NX-OS | ポート単位で設定。Designated Portに設定する。Root PortやAlternate Portには設定不可 |
| **Loop Guard** | BPDU途絶時に Blocking 状態から Forwarding 状態への誤遷移を防止 | スイッチ間のTrunkポートおよびAlternate/Root Port | 片方向リンク障害やBPDU処理遅延によるSTPループを防止 | 単体では物理層の故障を検知できない（UDLDとの併用が推奨される） | Catalyst全般, Nexus, Cisco IOS/IOS-XE, NX-OS | Designated Portには作用しない。PortFast有効ポートやSharedリンクでは使用不可 |
| **UDLD** | Layer 2キープアライブにより物理/論理的な片方向リンク障害を検出 | 光ファイバー接続やメディアコンバータを挟むTrunkリンク | 物理リンクがUp状態のまま受信用芯線のみ断線した障害を正確に検知 | パケット送受信のオーバーヘッド（デフォルト15秒周期） | Catalyst全般, Nexus, Cisco IOS/IOS-XE, NX-OS | Normal ModeとAggressive Modeが存在。Aggressive Modeは8回再送失敗で `err-disable` |
| **Dispute Mechanism** | 802.1w/1sでInferior BPDUかつDesignatedフラグを受信時にポートをブロック | 全RSTP/MSTP動作スイッチ（自動有効） | Duplexミスマッチや一方向障害によるループをプロトコルレベルで自動即時遮断 | 手動無効化不可（プロトコル標準組み込み動作） | RSTP/MSTP実装機種全般 | 相手がSTP/RSTP BPDUを誤送信・改ざんした場合にもトリガーされる |

---

## 🏗 動作原理

### 1. BPDU Guard の動作原理
* アクセスポートにPCやIP Phoneを接続する際、立ち上がり時間を高速化するために `spanning-tree portfast` を設定します。
* このPortFastポートに対して、攻撃者やユーザーが誤ってスイッチやルータを接続し、BPDUパケットがポートに入力された場合、BPDU Guard機能が即座にこれを検知します。
* スイッチは該当ポートのデータ転送および制御フレーム受信を停止し、インターフェイスのステータスを `err-disable`（Operational Down）へ遷移させます。

```
[Attacker / Rogue Switch]
        │ (Sends BPDU)
        ▼
 [Switch Access Port] ──(PortFast Enabled + BPDU Guard)
        │
        └─► [Detects BPDU] ──► [Transition to err-disable] (Port Shutdown)
```

### 2. Root Guard の動作原理
* ルートブリッジ選出では、最も低い Bridge ID (Priority + MAC Address) を持つスイッチが Root Bridge となります。
* ルートブリッジとして維持したい Core/Distribution スイッチの指定ポート（Designated Port）に Root Guard を設定します。
* アクセススイッチ側でより優先度の高い（ブリッジプライオリティが低い）BPDUが送信されてきた場合、Root Guardが設定されたポートはその上位BPDU（Superior BPDU）を無視し、ポート状態を `root-inconsistent` に変更します。
* `root-inconsistent` 状態の間、該当ポートではデータトラフィックの転送がブロックされます（BPDUの受信監視は継続）。上位BPDUの送信が停止すると、タイマー経過後に自動的に正常なForwarding状態に復帰します。

```
[Unauthorized Superior Switch (Priority 0)]
        │ (Sends Superior BPDU)
        ▼
 [Core/Dist Switch Designated Port] ──(Root Guard Enabled)
        │
        └─► [Intercepts Superior BPDU] ──► [Set State: root-inconsistent] (Block Data)
```

### 3. Loop Guard の動作原理
* STPの正常動作では、Root Port または Alternate Port は対向の Designated Port から定期的に BPDU を受信しています。
* 単方向リンク障害（送信芯線は正常だが受信芯線が切断）や対向スイッチのCPU高負荷によりBPDUが届かなくなると、STPタイマー（Max Age）がタイムアウトし、本来ブロックされるべき Alternate Port が Forwarding 状態に遷移してしまい、レイヤー2ループが発生します。
* Loop Guard が有効な場合、BPDUの受信がタイムアウトしても Forwarding 状態には遷移させず、ポートを `loop-inconsistent` 状態（Blocking同等）に維持します。BPDUの受信が再開されれば自動的に正常状態へ戻ります。

```
[Neighbor Switch] ──X (BPDU lost / Unidirectional fail)
        │
        ▼
 [Local Switch Alternate/Root Port] ──(Loop Guard Enabled)
        │
        └─► [Max Age Timeout] ──► [Set State: loop-inconsistent] (Prevent Forwarding)
```

### 4. UDLD (Unidirectional Link Detection) の動作原理
* UDLDは、隣接機器間で定期的にUDLDプロトコルフレーム（MAC: 01-00-0c-cc-cc-cc, EtherType: 0x0111）を交換し、双方向通信が維持されているかを監視します。
* 各スイッチは自機器の Device ID と Port ID、および検出した隣接機器の Device ID / Port ID リストをEchoパケットとして送信します。
* **Normal Mode**: 対向からのUDLDフレームが途絶えた場合、またはEchoに自機器情報が含まれない場合、リンクの通知状態を `Unknown` に変更しますが、ポートは閉鎖しません（Syslog警告のみ）。
* **Aggressive Mode**: 対向からのUDLDパケットが8回連続（デフォルト約64秒）で受信できない場合、対向機器との双方向接続が喪失したと判断し、ポートを即座に `err-disable` 状態にしてトラフィックを遮断します。

```
[Switch A] ───(TX)───► [Fiber / Cable] ───X (Rx Cut) ───► [Switch B]
   │                                                         │
   └◄───UDLD Echo (Contains Switch A ID) ────────────────────┘
        If Switch A stops receiving Echo:
        Aggressive Mode ──► [Set Port to err-disable]
```

---

## ⚙ 動作シーケンス

### 1. BPDU Guard 処理シーケンス
1. アクセスポート（PortFast有効）に端末またはスイッチが接続され、物理リンクが `Up` になる。
2. 対向機器からBPDUフレーム（DST MAC: `01:80:C2:00:00:00` または `01:00:0C:CC:CC:CD`）がスイッチポートに入力される。
3. スイッチのSTPエンジンがBPDUを受信し、BPDU Guardポリシーの違反を検知。
4. スイッチは該当ポートのMAC学習テーブルをクリアし、ポートを即座に `err-disable` に設定。
5. Syslogメッセージを発行：`%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port Fa0/1 with BPDU Guard enabled. Disabling port.`
6. 管理者による `shutdown` / `no shutdown` または `errdisable recovery cause bpduguard` タイマー処理によって復旧するまでポートは孤立状態を維持。

### 2. Root Guard 処理シーケンス
1. 指定ポート（Designated Port）で定期的にBPDUを送信中、対向からBPDUを受信する。
2. 受信したBPDUの Bridge ID / Root Cost を評価し、自ネットワークのCurrent Root BPDUよりも「優先度が高い（Superior BPDU）」か判断する。
3. Superior BPDU であると判断した場合、Root Bridgeの選出変更（Root Migration）を阻止する。
4. ポートのSTP状態を `Forwarding` から `root-inconsistent` に変更。データフレームの送信および受信をブロック。
5. Syslogを発行：`%SPANTREE-2-ROOTGUARNS: Root guard blocking port GigabitEthernet1/0/1 on VLAN0001.`
6. 対向からの Superior BPDU 受信が停止すると、Forward-Delay タイマー（約15秒）経過後に自動的に `Forwarding` 状態へ復帰。

### 3. UDLD Aggressive Mode 処理シーケンス
1. 隣接スイッチ間でUDLDピアシップが確立（15秒間隔でProbeパケット交換）。
2. 物理障害等により、Switch AからSwitch Bへの送信線は活きているが、Switch BからSwitch Aへの受信線が切断される。
3. Switch AはSwitch BからのUDLDパケットを受信できなくなる。
4. Switch AはUDLDタイマーを再起動し、1秒間隔で8回のProbeパケットを連続送信して再試行。
5. 8回連続で応答が得られない場合（計8秒間）、UDLDはリンク状態を `Unidirectional` と確定。
6. Switch Aはポートを `err-disable` 状態へ遷移させ、Syslogを発行：`%UDLD-3-DISABLE: Unidirectional link detected on port Gi1/0/1. Disabling port.`


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Securityラボ試験におけるSTP Securityは、単独での設定問題だけでなく、アクセス層のセキュリティ（Port Security, 802.1X, DHCP Snooping, DAI）やインフラ保護（CoPP, VACL, PVLAN）と組み合わせて出題される頻度が高い分野です。

### ラボ試験で狙われやすいポイントと実装上の注意点

1. **BPDU Guard: Global vs Interface 設定の挙動差**
   * **Global設定 (`spanning-tree portfast bpduguard default`)**: `portfast` が有効化されているポートでのみBPDU Guardが動作します。PortFastが無効なポートでは動作しません。
   * **Interface設定 (`spanning-tree bpduguard enable`)**: PortFastの有効/無効に関わらず、そのポートでBPDUを受信すると強制的に `err-disable` 化されます。

2. **BPDU Filter: Global vs Interface 設定の破滅的差異**
   * **Global設定 (`spanning-tree portfast bpdufilter default`)**: PortFastポートで起動時に数個のBPDUを送信。BPDUを受信すると即座にPortFast属性とBPDU Filter状態を解除し、通常のSTPポートとして動作（安全な挙動）。
   * **Interface設定 (`spanning-tree bpdufilter enable`)**: ポートでのBPDU送信を停止し、受信したBPDUもすべて無視（Drop）します。対向にスイッチが接続されてもSTPループを検知できなくなるため、ラボ試験で指定されない限りインターフェイス単位での有効化は厳禁です。

3. **Root Guard の設定対象ポート**
   * Root Guardは必ず **Designated Port（指定ポート）** に設定します。Root Port や Alternate Port に設定すると、正常なRoot BridgeからのBPDUをブロックしてしまい、全ネットワークのSTPトポロジが崩壊します。

4. **Loop Guard と Root Guard の相互適用**
   * Loop Guard と Root Guard は同一ポートに同時設定可能ですが、動作メカニズムが異なります。 Root Guard は「強すぎるBPDU（Superior BPDU）」をブロックし、Loop Guard は「BPDUの途絶（Lack of BPDU）」をブロックします。

5. **Err-disable 自動復旧（Errdisable Recovery）の設定**
   * BPDU Guard や UDLD により `err-disable` になったポートを自動復旧させる問題が出題されます。
   * コマンド：`errdisable recovery cause bpduguard` / `errdisable recovery cause udld` / `errdisable recovery interval <30-86400>`

6. **show コマンドからの状態判定**
   * `show spanning-tree inconsistentports`: `root-inconsistent` や `loop-inconsistent` になっているポート一覧を確認。
   * `show interfaces status err-disabled`: BPDU Guard や UDLD によって `err-disable` になったポートを確認。
   * `show udld neighbors`: UDLDのピアシップ確立状態を確認。

---

## 🛠 設定方法

### 1. Cisco IOS / IOS-XE での設定例

#### グローバル設定（推奨の基本ハードニング）
```bash
! 全アクセスポート（PortFast有効ポート）でBPDU Guardを自動有効化
spanning-tree portfast bpduguard default

! 全アクセスポート（PortFast有効ポート）で安全なBPDU Filterを自動有効化
spanning-tree portfast bpdufilter default

! 全Point-to-PointリンクでLoop Guardを有効化
spanning-tree loopguard default

! 光ファイバーインターフェイス全般でUDLD Aggressive Modeを有効化
udld enable aggressive

! err-disable状態からの自動復旧タイマー設定（30秒間隔）
errdisable recovery cause bpduguard
errdisable recovery cause udld
errdisable recovery cause loopguard
errdisable recovery interval 30
```

#### インターフェイス単位での個別設定
```bash
! アクセスポート（エンドポイント接続用）
interface GigabitEthernet1/0/1
 description Access_Port_To_End_Host
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
exit

! 配布層スイッチへのアップリンク（Designated Port側でRoot Guardを有効化）
interface GigabitEthernet1/0/24
 description Downlink_To_Access_Switch
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 spanning-tree guard root
 udld port aggressive
exit

! ブロッキング候補ポート（Alternate / Backup Port側でLoop Guardを有効化）
interface GigabitEthernet1/0/23
 description Uplink_To_Secondary_Distribution
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 spanning-tree guard loop
 udld port aggressive
exit
```

---

## 🔍 検証コマンド

STP Securityの動作状態や障害判定に使用する確認コマンド一覧：

| 目的 | コマンド | 確認・注目ポイント |
| :--- | :--- | :--- |
| **STP不整合状態の一覧表示** | `show spanning-tree inconsistentports` | `root-inconsistent` や `loop-inconsistent` に陥っているポートとVLANの特定 |
| **Err-disableポートの確認** | `show interfaces status err-disabled` | Cause（`bpduguard`, `udld` 等）および自動復旧までの残り時間を判定 |
| **Err-disable復旧設定の確認** | `show errdisable recovery` | 自動復旧が有効な要素（Causes）とタイマー値（Interval）の確認 |
| **STP要約・グローバル設定の確認** | `show spanning-tree summary` | BPDU Guard/Filter default, Loopguard default の有効化状態を確認 |
| **インターフェイス別STP詳細** | `show spanning-tree interface <id> detail` | `BPDU guard is enabled`, `Root guard is enabled`, `Loop guard is enabled` の表示 |
| **UDLDピア状態・リンク監視確認** | `show udld` / `show udld <id>` | Operational mode (Aggressive/Normal), Link state (Bidirectional/Unidirectional) |
| **UDLD隣接機器の確認** | `show udld neighbors` | 対向機器のDevice ID, Port ID, ピアシップの正常性を確認 |
| **STPイベントのリアルタイムデバッグ** | `debug spanning-tree events` | BPDU受信時の保護機能発動や状態遷移ログの取得 |
| **UDLDパケットデバッグ** | `debug udld events` / `debug udld packets` | UDLD Probe/Echoパケットの送受信トレース |

---

## 🚨 トラブルシュート

STP Security環境で発生する代表的な障害パターンと対処方法：

| 症状 | 主な原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| ポート接続直後に `err-disable` 化する | PortFast有効ポートにスイッチやハブが接続されBPDU Guardが発動 | `show interfaces status err-disabled`<br>`show log \| inc BPDUGUARD` | 端末側の不正スイッチ撤去、または正規スイッチ接続ポートの場合はPortFast/BPDU Guardの設定を解除 |
| ポートが `root-inconsistent` になり通信不能 | 指定ポート（Designated Port）配下に優先度の高い（Priority値の小さい）スイッチが接続された | `show spanning-tree inconsistentports`<br>`show log \| inc ROOTGUARD` | 配下スイッチのブリッジプライオリティを正しく調整（大きな値に変更）、または不正スイッチの切断 |
| ポートが `loop-inconsistent` になりブロックされる | 片方向リンク障害、または対向スイッチのCPU高負荷によりBPDU受信が中断した | `show spanning-tree inconsistentports`<br>`show log \| inc LOOPGUARD` | 物理ケーブル・SFPの交換、対向スイッチのCPU負荷（CoPP/プロセス）調査、UDLDログの確認 |
| UDLDによりポートが `err-disable` 化する | 光ファイバーの片芯断線、Rx/Tx逆刺し、メディアコンバータによるL1パススルー不具合 | `show udld <interface>`<br>`show interfaces status err-disabled` | 物理層パッチコード・SFPの交換、光パッチパネルの結線確認、対向機器のUDLD設定の一致確認 |
| BPDU Filter設定後にネットワーク全体でループ発生 | インターフェイス単位で `spanning-tree bpdufilter enable` を設定し、BPDU受信が完全に無視された | `show running-config interface <id>`<br>`show mac address-table count` | インターフェイス直接の `bpdufilter enable` を削除し、グローバルの `spanning-tree portfast bpdufilter default` に変更 |

---

## ⚠ 制限事項

1. **Root Guard と Loop Guard の排他性**
   * 同一インターフェイスにおいて、Root Guard と Loop Guard を同時に有効化した場合、Root Guard の設定が優先されます（Root Guard は Designated Port、Loop Guard は Root/Alternate Port 用のため概念的にも競合します）。
2. **Loop Guard と PortFast の非互換性**
   * `spanning-tree portfast` が有効なポートでは Loop Guard は機能しません。
3. **BPDU Filter (Interface設定) のリスク**
   * インターフェイスレベルで `spanning-tree bpdufilter enable` を設定すると、該当ポートは一切のBPDUを送信せず、受信したBPDUも破棄します。対向にスイッチが接続された場合にSTPループが確実に発生します。
4. **UDLD と EtherChannel (LACP/PAgP)**
   * EtherChannelのメンバポートでUDLDが片方向リンクを検知した場合、UDLDは個別のメンバポートのみを `err-disable` 化します。チャネル全体は残りの正常ポートで通信を継続します。

---

## 🔄 他技術との関連

* **PortFast**: エンドポイント接続ポートでSTPの Forward-Delay（30秒）をスキップして即座に Forwarding 遷移させる機能。BPDU Guard / BPDU Filter (Global) の前提条件となります。
* **Private VLAN (PVLAN)**: PVLAN Trunk ポートや Isolated ポートで STP Security（BPDU Guard / Root Guard）を適用し、テナント間のL2非分離や不正スイッチ接続を防止します。
* **DHCP Snooping & DAI (Dynamic ARP Inspection)**: アクセス層のセキュリティパッケージとして、BPDU Guard と同時に設定することで、L2フレーム（STP）、L3 IP割り当て（DHCP）、L2/L3マッピング（ARP）の改ざんを包括的に防止します。
* **Flex Links**: STPを使わずにL2冗長化を行う機能。STP BPDUは送信されないため、Flex Linksポート上ではSTP Security設定は無効化されます。

---

## 🧩 比較表

### 1. Loop Guard vs Root Guard

| 比較項目 | Loop Guard | Root Guard |
| :--- | :--- | :--- |
| **主な目的** | BPDU途絶による誤った Forwarding 遷移（ループ）の防止 | 意図しないスイッチによる Root Bridge 権限奪取の防止 |
| **適用推奨ポート** | Root Port, Alternate Port (スイッチ間リンク) | Designated Port (指定ポート, 配下スイッチへのポート) |
| **トリガー条件** | BPDUの「喪失（Lack of BPDU）」 | 「優先度の高いBPDU（Superior BPDU）」の受信 |
| **遷移する状態** | `loop-inconsistent` (Blockingと同等) | `root-inconsistent` (Blockingと同等) |
| **自動復旧条件** | BPDUの受信が再開されたとき | Superior BPDU の受信が停止したとき |

### 2. BPDU Guard vs BPDU Filter

| 比較項目 | BPDU Guard | BPDU Filter |
| :--- | :--- | :--- |
| **BPDU受信時の挙動** | ポートを即座に `err-disable` 化して閉鎖 | 【Global】PortFast解除して通常のSTPポート化<br>【Interface】受信したBPDUを完全無視（Drop） |
| **BPDU送信の有無** | 送信する（対向へBPDUは送信される） | 【Global】起動時に数個送信後停止<br>【Interface】一切送信しない |
| **安全性の評価** | **極めて高い**（不正接続を確実に遮断） | 【Global】高い<br>【Interface】**極めて危険**（ループ誘発のリスク） |

---

## 💡 ベストプラクティス

Cisco Validated Design (CVD) に基づくSTP Security設計の標準指針：

1. **アクセスポート (Access Ports)**:
   * `spanning-tree mode rapid-pvst` (または mst)
   * `spanning-tree portfast`
   * `spanning-tree portfast bpduguard default` (グローバル有効化)
   * `errdisable recovery cause bpduguard` および `errdisable recovery interval 30` を設定。
2. **分布層/コア層からアクセス層へのダウンリンク (Designated Ports)**:
   * `spanning-tree guard root` を設定し、アクセス層スイッチからの不正なRoot昇格を排除。
   * `udld port aggressive` を設定し、光/銅線リンクの片方向障害を防御。
3. **スイッチ間冗長リンク (Root / Alternate Ports)**:
   * `spanning-tree loopguard default` をグローバル有効化。
   * `udld port aggressive` を全Trunkリンクで有効化。


## 📝 ラボ学習・設定サンプル例

実際のCCIE Securityラボ試験レベルを意識した10個の実践演習シナリオです。

### 1. アクセスポートにおけるBPDU Guardと自動リカバリの設定

#### 問題
SW1のポート Gig1/0/1〜Gig1/0/10 はエンドユーザーPCが接続されるアクセスポートです。これらのポートで不正なスイッチが接続された際にネットワークを保護し、トラブル解消後に管理者の手動介入なしで自動復旧する構成を実装してください。

#### 要件
* Gig1/0/1〜10 をアクセスモード（VLAN 10）に設定し、PortFastを有効化すること。
* 不正なスイッチが接続されBPDUを受信した場合、ポートを即座にシャットダウンさせること。
* シャットダウンされたポートは、30秒後に自動的に復旧（`err-disable recovery`）を試みること。

#### 設定例
```bash
! グローバルでBPDU Guardと自動リカバリを設定
spanning-tree portfast bpduguard default
errdisable recovery cause bpduguard
errdisable recovery interval 30

! アクセスポートの個別設定
interface range GigabitEthernet1/0/1 - 10
 description Access_Ports_VLAN10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
exit
```

#### 検証方法
* `show spanning-tree summary` を実行し、`BPDU Guard is enabled by default` が `Enabled` になっていることを確認。
* 該当ポートにスイッチを接続してBPDUを送信させ、`show interfaces status err-disabled` でポートが `bpduguard` 原因で `err-disabled` に移行し、30秒後に自動復旧することを確認。

---

### 2. グローバル BPDU Filter の動作と安全な運用

#### 問題
SW1の全PortFast有効ポートに対して、無用なBPDU送信を抑制しつつ、誤ってスイッチが接続された場合には自動的に通常STPポートへ戻す安全なBPDU Filter構成を適用してください。

#### 要件
* PortFastが有効なポートでのみBPDU Filterをグローバルに動作させること。
* インターフェイス直接の `bpdufilter enable` コマンドは使用しないこと。

#### 設定例
```bash
! グローバルでPortFastポート用BPDU Filterを有効化
spanning-tree portfast bpdufilter default

! インターフェイス設定（PortFastのみ有効化）
interface GigabitEthernet1/0/11
 description Server_Port
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
exit
```

#### 検証方法
* `show spanning-tree summary` で `PortFast BPDU Filter is enabled by default` を確認。
* `debug spanning-tree events` を有効にし、ポート起動直後に数個のBPDUが送信された後、送信が停止することを確認。対向にスイッチを接続した際、PortFast属性が解除され通常のSTPリスニング/ラーニング状態へ移行することを確認。

---

### 3. ディストリビューションスイッチでの Root Guard 設定

#### 問題
Core-SW1 はネットワーク内のRoot Bridgeです。Access-SW1 が誤ってブリッジプライオリティ `0` に変更された場合でも、Core-SW1 が Root Bridge の地位を維持し続けるよう配下スイッチ接続ポートを保護してください。

#### 要件
* Core-SW1 の Gig1/0/24（Access-SW1 への指定ポート）で Root Guard を有効化すること。
* Access-SW1 から上位BPDUを受信した際、Gig1/0/24 をデータ遮断状態に移行させること。

#### 設定例
```bash
! Core-SW1 の設定
interface GigabitEthernet1/0/24
 description Downlink_To_Access-SW1
 switchport mode trunk
 switchport trunk allowed vlan all
 spanning-tree guard root
exit
```

#### 検証方法
* Access-SW1 側で `spanning-tree vlan 1 priority 0` を設定。
* Core-SW1 で `show spanning-tree inconsistentports` を実行し、Gig1/0/24 が `ROOT-Inconsistent` 状態としてブロックされていることを確認。
* Access-SW1 のプライオリティを元に戻すと、Core-SW1 のポートが自動的に Forwarding 状態に復帰することを確認。

---

### 4. スイッチ間 Trunk リンクにおける Loop Guard の設定

#### 問題
SW1 と SW2 間の Trunk リンクで、片方向のBPDU受信障害が発生した際に、ブロッキングポート（Alternate Port）が誤って Forwarding へ遷移してループが発生するのを防ぐ構成を構築してください。

#### 要件
* SW1 の配下にある全 Trunk インターフェイスに Loop Guard を適用すること（グローバル設定およびインターフェイス設定の両方）。

#### 設定例
```bash
! グローバルでのLoop Guard有効化（Point-to-Pointリンク全体に適用）
spanning-tree loopguard default

! 個別インターフェイスでの確認・明示設定
interface GigabitEthernet1/0/23
 description Trunk_To_SW2
 switchport mode trunk
 spanning-tree guard loop
exit
```

#### 検証方法
* `show spanning-tree interface GigabitEthernet1/0/23 detail` を実行し、`Loop guard is enabled` と表示されることを確認。
* 疑似的に対向からのBPDU送信を停止させ（ACL等で拒否）、タイマータイムアウト時に `show spanning-tree inconsistentports` で該当ポートが `LOOP-Inconsistent` 状態になることを確認。

---

### 5. 光ファイバー Uplink ポートでの UDLD Aggressive Mode 設定

#### 問題
SW1 と SW2 を接続する光ファイバーリンク（Gig1/1/1）において、物理的な片芯断線やSFPモジュール障害による片方向通信を検知し、リンクを即座に閉鎖する構成を設定してください。

#### 要件
* Gig1/1/1 で UDLD Aggressive Mode を有効化すること。
* 対向からのUDLD応答が連続して途絶した場合、ポートを `err-disable` 状態に遷移させること。

#### 設定例
```bash
! インターフェイス単位でのUDLD Aggressive Mode有効化
interface GigabitEthernet1/1/1
 description Fiber_Uplink_To_SW2
 switchport mode trunk
 udld port aggressive
exit

! グローバルでの自動復旧設定
errdisable recovery cause udld
errdisable recovery interval 60
```

#### 検証方法
* `show udld GigabitEthernet1/1/1` を実行し、`Mode: Aggressive` および `Bidirectional` 状態を確認。
* 光ファイバーの受信用芯線を抜いて片方向障害を発生させ、`show interfaces status err-disabled` でポートが `udld` 原因で `err-disable` 化することを確認。

---

### 6. アクセスポートにおける L2 セキュリティ複合設定 (BPDU Guard + Port Security + DHCP Snooping)

#### 問題
アクセスポート Gig1/0/2 に、STP攻撃保護、MACアドレス偽装防止、および不正DHCPサーバー対策を組み合わせた包括的なアクセス層セキュリティを設定してください。

#### 要件
* ポートをアクセスモード（VLAN 100）とし、PortFastとBPDU Guardを有効化すること。
* 最大MACアドレス数を 2 に制限し、違反時はポートを `err-disable`（`violation shutdown`）とすること。
* DHCP Snooping を有効化し、Untrusted ポートとして動作させること。

#### 設定例
```bash
! グローバルDHCP Snooping有効化
ip dhcp snooping
ip dhcp snooping vlan 100

! アクセスポート設定
interface GigabitEthernet1/0/2
 description Secure_Access_Port
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation shutdown
 switchport port-security aging time 2
 switchport port-security aging type inactivity
exit
```

#### 検証方法
* `show running-config interface GigabitEthernet1/0/2` で全セキュリティ機能の設定内容をスクリーニング。
* スイッチ接続（BPDU送信）、MAC上限突破、不正DHCP OFFER送信の各試験を行い、それぞれ適切な保護動作（`err-disable` またはドロップ）が発動することを確認。

---

### 7. Private VLAN (PVLAN) 環境における STP Security 統合

#### 問題
Private VLAN が設定されたアクセススイッチ SW1 において、Isolated ポートおよび Promiscuous Trunk ポートに対して最適な STP 保護を適用してください。

#### 要件
* Isolated アクセスポート（Gig1/0/5）には PortFast と BPDU Guard を適用すること。
* Promiscuous Trunk ポート（Gig1/0/24）には Root Guard を適用すること。

#### 設定例
```bash
! Isolated ポートの設定
interface GigabitEthernet1/0/5
 description PVLAN_Isolated_Host
 switchport mode private-vlan host
 switchport private-vlan host-association 500 501
 spanning-tree portfast
 spanning-tree bpduguard enable
exit

! Promiscuous Trunk ポートの設定
interface GigabitEthernet1/0/24
 description PVLAN_Promiscuous_Trunk_To_Router
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 500 add 501-502
 spanning-tree guard root
exit
```

#### 検証方法
* `show interface GigabitEthernet1/0/5 switchport` および `show interface GigabitEthernet1/0/24 switchport` でPVLAN設定を確認。
* `show spanning-tree interface GigabitEthernet1/0/5 detail` で BPDU Guard の有効化を確認し、`show spanning-tree interface GigabitEthernet1/0/24 detail` で Root Guard の有効化を確認。

---

### 8. EtherChannel メンバポートにおける UDLD Aggressive Mode の設定

#### 問題
SW1 と SW2 間の EtherChannel（Port-channel 1: Gig1/0/19, Gig1/0/20）において、一部の物理リンクで片方向障害が発生した際に、チャネル全体を倒さずに障害メンバポートのみを切り離す設定を行ってください。

#### 要件
* Port-channel 1 の各メンバ物理インターフェイス上で UDLD Aggressive Mode を有効化すること。

#### 設定例
```bash
! メンバ物理インターフェイスへの設定
interface range GigabitEthernet1/0/19 - 20
 description EtherChannel_Members_To_SW2
 switchport mode trunk
 channel-group 1 mode active
 udld port aggressive
exit

! Port-channel 論理インターフェイス
interface Port-channel 1
 switchport mode trunk
exit
```

#### 検証方法
* `show udld neighbor` を実行し、Gig1/0/19 と Gig1/0/20 がそれぞれ個別に対向とUDLDピアを確立していることを確認。
* 片方のメンバポートの受信線を物理的に切断し、該当物理ポートのみが `err-disable` 化され、Port-channel 1 自体は残り1本のメンバポートで通信を継続することを確認。

---

### 9. Errdisable Recovery タイマー調整と特定イベントの復旧設計

#### 問題
SW1 において、BPDU Guard および UDLD によって `err-disable` 化されたポートは 60 秒後に自動復旧させ、Port Security 違反による `err-disable` は手動復旧のみとする非対称な自動復旧ポリシーを設定してください。

#### 要件
* BPDU Guard と UDLD のみを自動復旧対象に登録すること。
* 復旧インターバルを 60 秒に指定すること。
* Port Security 違反（`psecure-violation`）は自動復旧対象外とすること。

#### 設定例
```bash
! 自動復旧設定の初期化と個別指定
no errdisable recovery cause all
errdisable recovery cause bpduguard
errdisable recovery cause udld
errdisable recovery interval 60
```

#### 検証方法
* `show errdisable recovery` を実行し、`Timer interval: 60 seconds` であること、および `bpduguard` と `udld` のみが `Enabled` で `psecure-violation` が `Disabled` であることを確認。

---

### 10. Rapid-PVST+ / MSTP Dispute Mechanism の理解とトラブルシューティング

#### 問題
SW1 と SW2 間のリンクで、Duplex ミスマッチ（片方が Half Duplex、片方が Full Duplex）が発生した結果、STPの Dispute Mechanism がトリガーされてポートがブロックされる現象をログから特定し、解決してください。

#### 要件
* Dispute Mechanism によるポートブロックの原因を突き止め、Duplex 設定を一致させて通信を復旧させること。

#### 設定例
```bash
! SW1 の設定（問題修正後）
interface GigabitEthernet1/0/15
 description Link_To_SW2
 switchport mode trunk
 duplex full
 speed 1000
exit

! SW2 の設定（問題修正後）
interface GigabitEthernet1/0/15
 description Link_To_SW1
 switchport mode trunk
 duplex full
 speed 1000
exit
```

#### 検証方法
* 障害発生時、`show logging` で `%SPANTREE-2-DISPUTE_FAIL` または Dispute 関連のイベントログを確認。
* `show interface GigabitEthernet1/0/15` で Duplex の不一致（Half vs Full）を確認。
* 両端の Duplex を `full` に固定後、`show spanning-tree interface GigabitEthernet1/0/15` でポートが即座に `Forwarding` 状態へ戻ることを確認。


## ❓ 想定試験問題

CCIE Security ラボ試験・筆記試験を想定した実践的な問題集です。

### 質問 1 (コンフィグ読解・挙動予測)
スイッチのグローバルコンフィグに以下の設定が追加されました。

```bash
spanning-tree portfast bpduguard default
spanning-tree portfast bpdufilter default
```

ポート GigabitEthernet1/0/5 には `spanning-tree portfast` のみが設定されています。このポートに外部からBPDUが入力された場合、ポートの動作はどうなりますか？

**選択肢**:
1. ポートは即座に `err-disable` 状態になる。
2. ポートはBPDUを無視し、PortFast状態を維持する。
3. ポートはBPDU受信時にPortFast属性およびBPDU Filter状態を解除し、通常のSTPポートとして動作する。
4. ポートは `root-inconsistent` 状態に遷移する。

**正解**: 3
**解説**: グローバルで `bpdufilter default` と `bpduguard default` の両方が有効な場合、`bpdufilter` の動作が優先されます。PortFastポートがBPDUを受信すると、BPDU FilterはPortFast属性を自動的に無効化し、通常のSTPネゴシエーション（Listening -> Learning -> Forwarding）へ移行します。そのため、`err-disable` にはならず、通常ポートとして動作します。

---

### 質問 2 (トラブルシューティング)
管理者から「Switch-A の GigabitEthernet1/0/24 が `root-inconsistent` 状態になり、データ通信が遮断されている」との報告がありました。コンフィグを確認したところ、該当ポートには `spanning-tree guard root` が設定されていました。この障害の直接的な原因として最も可能性が高いものはどれですか？

**選択肢**:
1. 対向スイッチからBPDUが送信されなくなった（BPDUの途絶）。
2. 対向スイッチから Current Root Bridge よりも優先度の高い（Bridge IDの数値が小さい）BPDUを受信した。
3. 該当ポートでデュプレックスミスマッチが発生した。
4. 該当ポートでUDLDの片方向リンクが検出された。

**正解**: 2
**解説**: `Root Guard`（`spanning-tree guard root`）は、設定された指定ポート（Designated Port）において、現在のRoot Bridgeよりも優先度の高いBPDU（Superior BPDU）を受信した際に、ポートを `root-inconsistent` 状態にしてデータ転送をブロックします。原因は対向スイッチからの Superior BPDU 受信です。

---

### 質問 3 (設計・実装)
次のSTP保護機能のうち、**Designated Port（指定ポート）** に設定することが推奨され、Root Port や Alternate Port に設定してはならない機能はどれですか？

**選択肢**:
1. Loop Guard
2. BPDU Guard
3. Root Guard
4. UDLD Aggressive Mode

**正解**: 3
**解説**: `Root Guard` は、配下（ダウンストリーム）のスイッチが不正にRoot Bridgeになることを防ぐための機能であり、上位スイッチ側の **Designated Port** に設定します。Root Port や Alternate Port に設定すると、正規の Root Bridge からの BPDU をブロックしてしまい、STPトポロジが破壊されます。

---

### 質問 4 (トラブルシューティング・UDLD)
光ファイバーで接続された2台のスイッチ間で、一方の送信芯線のみが断線しました。UDLD Aggressive Mode が有効な場合、スイッチはどのようなシーケンスでポートを処理しますか？

**正解・解説**:
1. 送信芯線が活きている側のスイッチは、対向からのUDLD Echoパケットを受信できなくなります。
2. スイッチは1秒間隔で8回のProbeパケットを連続送信して再試行します。
3. 8回連続で応答が得られない場合（約8秒後）、リンクを `Unidirectional`（片方向）と確定します。
4. スイッチは該当ポートを即座に `err-disable` 状態へ遷移させ、リンクを物理的に遮断してL2ループやブラックホールを防止します。

---

### 質問 5 (コンフィグ実装・条件分岐)
以下の要件を満たすアクセスポート（GigabitEthernet1/0/1）の最小限かつ適切なコンフィグ記述を作成してください。

**要件**:
- ポートは VLAN 50 のアクセスモード。
- 端末接続時のSTP遷移時間を高速化すること。
- 不正スイッチ接続時に即座にポートを閉鎖（`err-disable`）すること。

**回答コンフィグ例**:
```bash
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 spanning-tree bpduguard enable
exit
```

---

## 🔗 参考リソース

Cisco公式ドキュメントおよび技術解説リソース一覧：

* Cisco Technical Note: Spanning Tree PortFast BPDU Guard Enhancement
* Cisco Technical Note: Understanding and Configuring the Unidirectional Link Detection Protocol (UDLD)
* Cisco Configuration Guide: Catalyst 9300 Switch Cisco IOS XE - Configuring STP Extensions
* Cisco Technical Note: Spanning Tree Protocol Root Guard Enhancement
* Cisco Technical Note: Enhancements to Spanning Tree Protocol using Loop Guard and BPDU Skew Detection
* Cisco Live Slide: BRKCRS-1500 - Advanced Spanning Tree Protocol Troubleshooting

---

## 📝 補足（Notes）

* **BPDU Skew Detection**: CPU高負荷等によりBPDUの到着間隔が不規則になる現象を検知し、Syslog警告を発行するCisco固有機能。
* **Dispute Mechanism の補足**: RSTP/MSTPにおいて、自ポートが Designated で対向からも Designated 属性の Inferior BPDU が届いた場合、Duplexミスマッチや片方向障害が発生していると判断し、即座にポートをブロックします（手動無効化不可の組み込み仕様）。
* **Err-disable 復旧のベストプラクティス**: 本番障害時は単に `errdisable recovery` で自動復旧させるだけでなく、`show logging` や `show interfaces status err-disabled` で根本原因（不正スイッチの接続や物理ケーブル断線）を特定・除去してから復旧させることが不可欠です。


