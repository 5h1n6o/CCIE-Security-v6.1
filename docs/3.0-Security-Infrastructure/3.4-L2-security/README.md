---
layout: default
title: 3.4-L2-security
nav_order: 4
parent: 3.0-Security-Infrastructure
---

# 3.4 Layer 2 security techniques

---

## 📘 概要

レイヤ2（データリンク層）は、ローカルエリアネットワーク（LAN）におけるエンドポイント間の直接対話およびスイッチング処理を担う基礎インフラストラクチャです。ネットワークセキュリティ設計において、レイヤ2はセキュリティ境界の最前線（First Hop）として機能し、攻撃者がネットワークアクセスを獲得した直後に試みる偵察行為、なりすまし（Spoofing）、中間者攻撃（Man-in-the-Middle: MitM）、サービス拒否（DoS）攻撃を阻止する極めて重要な役割を果たします。

Layer 2 Security Techniques（レイヤ2セキュリティ技術）は、スイッチングファブリックにおけるMACアドレス学習、ARPパケット処理、DHCPアドレス割り当て、スパニングツリープロトコル（STP）計算、およびブロードキャスト/マルチキャスト転送の信頼性を担保するための統合的保護機構群です。これには以下の主要技術が含まれます。

1. **Port Security**: スイッチポートに接続を許可するMACアドレスの数を制限・固定し、不正なエンドポイントの接続やMACアドレスファンプッディング攻撃（MAC Address Flooding Attack）を防止します。
2. **DHCP Snooping**: 信頼（Trusted）ポートと非信頼（Untrusted）ポートの境界を定義し、偽DHCPサーバー（Rogue DHCP Server）による不正なIP割当やデフォルトゲートウェイのなりすましを遮断すると同時に、バインディングデータベース（DHCP Snooping Binding Database）を動的に構築します。
3. **Dynamic ARP Inspection (DAI)**: DHCP Snoopingバインディングデータベースおよび静的ARP ACLを参照し、悪意のあるARP応答（ARP Cache Poisoning / ARP Spoofing）をハードウェアレベルで検証・破棄します。
4. **IP Source Guard (IPSG)**: DHCP Snoopingバインディング（IP/MAC/VLAN/Port）に基づき、ポートに入力されるIPパケットの送信元IPアドレスおよび送信元MACアドレスをフィルタリングし、IPアドレスなりすまし（IP Spoofing）を阻止します。
5. **Storm Control**: ブロードキャスト、マルチキャスト、および不明なユニキャスト（Unknown Unicast）の受信トラフィックレートを監視し、設定閾値を超行した異常トラフィックを抑制（Suppress）またはポートを無効化（Err-disable）することで、ループやDoS攻撃による帯域枯渇を防ぎます。
6. **STP Security (BPDU Guard / BPDU Filter / Root Guard / Loop Guard)**: スパニングツリー計算の改ざん、不正なRoot Switchの選出、トポロジー変更攻撃（TC Attack）、および片方向リンク障害によるループ発生を防御します。
7. **Private VLANs (PVLAN) & VACL (VLAN ACL)**: 同一VLAN（同一サブネット）内におけるL2分離（Microsegmentation）を提供し、マルウェアの横移動（Lateral Movement）を防止します。

CCIE Security v6.1ラボ試験において、これらのレイヤ2セキュリティ機能は個別の単独設定にとどまらず、Cisco ISEによる動的ポート制御、802.1X/MAB認証、DAI+IPSG+DHCP Snoopingの三位一体連携、および802.1Q TrunkやPVLANとの高度な複合構成として出題されます。

---

## 🔑 要点

Layer 2 Security Techniquesの主要機能と仕様を以下の表に整理します。

| 技術要素 | 機能特性 | 主な保護対象・用途 | メリット | デメリット・オーバーヘッド | 対応機種・適用ポート | 主な制限事項・設計上の注意点 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Port Security** | MACアドレス数制限・固定学習（Static/Dynamic/Sticky） | 不正PC接続防止、MAC Flooding遮断 | 設定が簡潔、即座に違反ポートを遮断可能 | 多数端末の動的移動環境では運用負担増 | Access Port, Trunk Port (Native VLAN/Allowed VLAN) | Dynamic/Sticky MACはスイッチ再起動時に保存が必要（`copy run start`） |
| **DHCP Snooping** | UntrustedポートでのDHCP OFFER/ACK破棄、バインディング構築 | Rogue DHCP防御、IPSG/DAIの基盤データベース構築 | 後段のIPSG/DAIと密結合し包括的L2セキュリティを提供 | データベース永続化用にTFTP/FTP/Flash保存設計が必須 | L2 Switch (Access/Trunk) | Option 82の挿入ルール（Untrusted入力を基本Drop）に注意 |
| **DAI (Dynamic ARP Inspection)** | UntrustedポートでのARPパケットのIP/MAC整合性検証 | ARP Poisoning/MitM攻撃防止、IP/MACなりすまし遮断 | DHCP Snooping DBと連動し自動的にARPを保護 | CPUでのARP検証負荷（Rate Limit設定でCoPP的保護が必要） | L2/L3 Switch (VLAN単位で有効化) | 静的IP環境ではARP ACLの手動定義・バインディング追加が必要 |
| **IP Source Guard (IPSG)** | 入力IPパケットの送信元IP/MACアドレスバインディング検証 | IP Spoofing防止、盗聴・横取り攻撃の無効化 | ポート単位で厳格な送信元検証を自動適用 | 物理ハードウェア（TCAM）リソースの消費 | L2 Access Port (DHCP Snooping有効環境) | Trunkポートでの適用制限、Port Securityとの併用要件 |
| **Storm Control** | 受信レート監視（%/pps/bps）、閾値超過トラフィック抑制 | Broadcast/Multicast/Unknown Unicast Storm防止 | スイッチ全体の帯域枯渇・CPU高負荷を阻止 | 正常なマルチキャスト/ブロードキャストの意図しない破棄リスク | 物理Access/Trunk Port, EtherChannel | 閾値設定（Rising/Falling）の最適化が必要 |
| **STP Security** | BPDU Guard/Filter, Root Guard, Loop Guard | 不正Root選出、BPDUインジェクション、トポロジー改ざん防止 | STPトポロジーの安定性と信頼性を維持 | 設定箇所の誤りによるSTPループ発生リスク | Edge Port (BPDU Guard), Designated Port (Root Guard) | PortFast（Edge）との正しい組み合わせが不可欠 |
| **PVLAN / VACL** | Primary/Secondary VLAN構造によるL2セグメンテーション | 同一VLAN内端末隔離、マルウェア横移動防止 | L3サブネット割り当てを節約しつつ隔離を実現 | スイッチ間トランクでのVLANマッピング設計の複雑化 | L2/L3 Access/Trunk Port | VTP v1/v2ではTransparent Mode必須（VTP v3でサポート） |

---

## 🏗 動作原理

Layer 2 Security Techniquesは、各層の脅威に対して相互に連動する防御アーキテクチャを形成します。最も代表的な統合通信フローである **DHCP Snooping → IP Source Guard → Dynamic ARP Inspection (DAI)** の多層検証アーキテクチャの動作原理を以下に示します。

```
[ Client (Access Port / Untrusted) ]
                 │
                 │ 1. DHCP Request (Broadcast)
                 ▼
[ Switch Access Port ] ───(DHCP Snooping: Option 82追加 & Untrusted検査)───► [ DHCP Server (Trusted Port) ]
                 │                                                                        │
                 │ 2. DHCP Snooping Binding Table 登録                                   │ 2. DHCP Offer / ACK
                 │    (MAC: 0011.2233.4455, IP: 10.1.1.50, VLAN: 10, Port: Gi1/0/1)        │ (IP/Lease/Gateway情報)
                 ◄────────────────────────────────────────────────────────────────────────┘
                 │
                 │ 3. データパケット送信 (IP: 10.1.1.50, MAC: 0011.2233.4455)
                 ▼
[ IP Source Guard (IPSG) 検証 ]
    ├─ (Match DB)   ───► [ CEF/FIB L2/L3 Switching 許可 ]
    └─ (Mismatch)  ───► [ Hardware Drop ]
                 │
                 │ 4. ARP Request / Reply 送信
                 ▼
[ Dynamic ARP Inspection (DAI) 検証 ]
    ├─ (Match DB)   ───► [ CPU/ASIC ARP Forwarding 許可 ]
    └─ (Mismatch)  ───► [ Drop & Syslog Logging ]
```

### 1. DHCP Snooping の動的バインディング構築機構
- **Trusted Interface**: DHCP サーバーまたは上位スイッチが接続されるポート。DHCP OFFER、DHCP ACK、DHCP LEASEQUERY 等のサーバーメッセージの受信を許可します。
- **Untrusted Interface**: エンドポイント（PC、IP Phone、プリンタ等）が接続されるポート。DHCP DISCOVER、DHCP REQUEST のみの受信を許可し、DHCP OFFER/ACK を受信した場合は即座にパケットを破棄し、ポートセキュリティ違反ログを出力します。
- **Binding Database**: MACアドレス、IPアドレス、IP Lease時間、Binding Type（dhcp-snooping/static）、VLAN ID、Port ID を記録したメモリテーブルです。

### 2. IP Source Guard (IPSG) のTCAMバインディングフィルター
- Accessポートに入力されるIPデータパケットの送信元IPおよびMACアドレスを、DHCP Snoopingバインディングデータベースと照合します。
- 送信元IPのみを検証するモード（`ip verify source`）と、送信元IPおよび送信元MACアドレスの両方を検証するモード（`ip verify source port-security`）が存在し、TCAM（Ternary Content Addressable Memory）に自動生成されたアクセスリスト（Port ACL）をロードして低遅延でパケットを破棄します。

### 3. Dynamic ARP Inspection (DAI) のARPパケット検証
- 入力されたARPパケット（ARP Request / ARP Response）の Sender MAC / Sender IP アドレスペアを、DHCP Snoopingバインディングデータベース（または手動定義の ARP ACL）と照合します。
- 不一致のパケットはCPUに転送された段階（またはHardware ASIC）で即座にドロップされ、ARP キャッシュポイズニング攻撃を無効化します。

---

## ⚙ 動作シーケンス

各レイヤ2セキュリティ機能がパケットを受信した際の段階的な処理シーケンスを詳細に解説します。

### パケット受信時の処理判定順序（Packet Processing Pipeline）

スイッチの物理ポートまたはVLANインターフェイスでフレームを受信した際、レイヤ2ファブリックは以下の順序でセキュリティ検証を実行します。

```
[ Frame Ingress on Switch Port ]
              │
              ▼
    1. Port Security Check (MAC Count & Violation Policy)
              │ (Pass)
              ▼
    2. Storm Control Threshold Check (Broadcast/Multicast Rate)
              │ (Pass)
              ▼
    3. STP Frame Check (BPDU Guard / BPDU Filter)
              │ (Pass)
              ▼
    4. Is Frame ARP? ───(Yes)───► Dynamic ARP Inspection (DAI) Verification
              │ (No - IP Frame)              ├─ Matches DHCP Snooping / ARP ACL ──► Permit
              ▼                              └─ Mismatch ──► Drop & Log
    5. IP Source Guard (IPSG) Check
              ├─ Sender IP/MAC Matches Binding DB ──► Permit
              └─ Mismatch ──► Drop
              │ (Pass)
              ▼
    6. Port ACL / VACL / RACL Processing
              │ (Permit)
              ▼
    7. Layer 2 Forwarding / Private VLAN Egress Filtering
```

#### 段階 1: Port Security のパケット処理シーケンス
1. ポートに入力されたフレームの送信元MACアドレス（Source MAC）を抽出。
2. 当該ポートのSecure MACテーブルを参照。
3. 未学習のMACアドレスであり、かつ最大学習数（`maximum`）に達している場合、設定されたViolation Modeに応じて以下の処理を実行：
   - **Protect**: 学習上限を超えた不承認MACからのパケットをサイレントドロップ。カウンタ増加なし、Syslogなし。
   - **Restrict**: 不承認パケットをドロップし、`SecurityViolation` カウンタをインクリメントし、Syslog/SNMP Trapを送信。
   - **Shutdown**: ポートを即座に `err-disable` 状態へ移行させ、リンクダウン。Syslog/SNMP Trapを送信。

#### 段階 2: DHCP Snooping パケット処理シーケンス
1. 入力パケットが UDP Port 67/68 (DHCP) か判定。
2. インターフェイスの Trust 状態を確認。
3. **Untrusted ポートの場合**:
   - パケットタイプが DHCP OFFER, ACK, NAK 等のサーバーレスポンスであればパケットを破棄（Rogue DHCP 遮断）。
   - パケットタイプが DHCP REQUEST / DISCOVER の場合、GiADDR（Gateway IP Address）フィールドが `0.0.0.0` でない場合は破棄。
   - Option 82（Relay Agent Information）が既に挿入されている場合、`ip dhcp snooping information option allow-untrusted` が未設定であればパケットを破棄。
   - 検証を通過した場合、Option 82 を付与して Trusted ポートへ転送。
4. **DHCP ACK 受信時（Trusted ポートからの応答）**:
   - クライアントの MAC, 割り当て IP, Lease 時間, VLAN, 物理ポート情報を `DHCP Snooping Binding Table` に動的登録。

---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験における Layer 2 Security の出題傾向、トラップ、および重点検証ポイントを解説します。

### 1. Blueprintにおける重要ポイントと出題パターン
- **DHCP Snooping Option 82 と Cisco スイッチの不整合（最頻出トラップ）**:
  - Cisco スイッチはデフォルトで Untrusted ポートからの DHCP 転送時に Option 82 を自動付与します。
  - 後続のアクセススイッチ（DHCP Snooping が有効な別のスイッチ）は、Untrusted ポートから Option 82 付きの DHCP REQUEST を受信すると「改ざんパケット」とみなしてデフォルトで**即座にドロップ**します。
  - **対策**: スイッチ間で Trunk 接続する場合は Trunk ポートを `ip dhcp snooping trust` に設定するか、`ip dhcp snooping information option allow-untrusted` または `no ip dhcp snooping information option` を設定する必要があります。

- **DAI (Dynamic ARP Inspection) と静的 IP 端末の統合**:
  - 試験要件で「DHCP を使用しない静的 IP サーバ環境に対して DAI を適用せよ」と指示されるケース。
  - **対策**: `arp access-list` を作成して静的な IP/MAC バインディングを定義し、`ip arp inspection filter <ACL_NAME> vlan <VLAN_ID> [static]` コマンドで適用します。`static` オプションを付けない場合、ACL にマッチしなかったパケットは DHCP Snooping データベースの検索にフォールバックしますが、`static` を付けると ACL にマッチしないものは即座にドロップされます。

- **IP Source Guard と Port Security の併用**:
  - 送信元 IP と送信元 MAC の両方を検証する要件（`ip verify source port-security`）。
  - この設定を有効化する場合、当該ポートで **Port Security があらかじめ有効（`switchport port-security`）になっていなければ命令が正しく機能しません**。

- **STP Security の組み合わせ問題**:
  - 「アクセスポートにスイッチが不正接続された場合、即座にポートを遮断せよ」→ `spanning-tree portfast` + `spanning-tree bpduguard enable`（またはグローバルコマンド `spanning-tree portfast bpduguard default`）。
  - 「上位スイッチから不正な Root BPDU が送信された場合、該当ポートの指定ポート（Designated Port）役割を維持しつつトラフィックをブロックせよ」→ `spanning-tree guard root`（Root Guard）。

- **Private VLANs (PVLAN) のマッピングと SVI 設定**:
  - Primary VLAN と Secondary VLAN (Isolated / Community) の適切なバインド。
  - ルーティングを行う場合、SVI インターフェイス（`interface Vlan <Primary_VLAN>`）において `private-vlan mapping <Secondary_VLAN_List>` を忘れると、Secondary VLAN 端末からの外部ルーティング通信が完全に途絶します。

### 2. debug ログと show コマンドの読み取り

#### (1) Dynamic ARP Inspection 違反の確認
```text
%SW_DAI-4-DHCP_REFUSED: 1 packts refused by DAI. Rcvd packet type: request, src MAC: 0011.2233.4455, src IP: 10.1.1.99, dst MAC: 0000.0000.0000, dst IP: 10.1.1.1 on Gi1/0/2, vlan 10
```
- **判定**: Gi1/0/2 (VLAN 10) で受信した ARP Request (IP: 10.1.1.99, MAC: 0011.2233.4455) が DHCP Snooping Binding Table に存在しないため破棄された。

#### (2) Port Security Violation 違反の確認
```text
%PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred, caused by MAC address 00aa.bbcc.ddee on port GigabitEthernet1/0/5.
%LINK-3-UPDOWN: Interface GigabitEthernet1/0/5, changed state to down
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet1/0/5, changed state to down
```
- **判定**: Gi1/0/5 で未許可の MAC アドレスが検出され、Violation Mode が Shutdown に設定されていたため、インターフェイスが `err-disable` 化された。

---


## 🛠 設定方法

Layer 2 Security の主要技術におけるコンフィグレーション例を示します。

### 1. Port Security の設定（Sticky MAC & Restrict / Shutdown）

```bash
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security aging time 10
 switchport port-security aging type inactivity
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### 2. DHCP Snooping / DAI / IP Source Guard の統合設定

```bash
! --- 1. グローバルで DHCP Snooping を有効化 ---
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option

! --- 2. DAI (Dynamic ARP Inspection) の有効化 ---
ip arp inspection vlan 10,20
ip arp inspection validate src-mac dst-mac ip

! --- 3. Trusted ポート（DHCPサーバー・アップリンク方向）の設定 ---
interface GigabitEthernet1/0/24
 description *** Trunk Uplink to Core/DHCP Server ***
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 ip dhcp snooping trust
 ip arp inspection trust

! --- 4. Untrusted アクセスポート（IPSG & DAI 適用）の設定 ---
interface Range GigabitEthernet1/0/1 - 10
 switchport mode access
 switchport access vlan 10
 ip verify source port-security
 ip arp inspection limit rate 15 burst interval 1
 switchport port-security
 switchport port-security maximum 1
```

### 3. 静的 IP 端末用 ARP ACL と DAI の連携設定

```bash
! --- 静的 IP アドレス端末用の ARP ACL 定義 ---
arp access-list STATIC_SERVER_ARP
 permit ip host 10.1.1.100 mac host 0050.56a1.2233

! --- VLAN 10 への適用 (Static オプションで ACL 不一致を即破棄) ---
ip arp inspection filter STATIC_SERVER_ARP vlan 10 static
```

### 4. Storm Control の設定

```bash
interface GigabitEthernet1/0/2
 switchport mode access
 switchport access vlan 10
 storm-control broadcast level 10.0 5.0
 storm-control multicast level pps 1k 500
 storm-control action shutdown
 storm-control action trap
```

---

## 🔍 検証コマンド

Layer 2 Security 各機能の状態確認およびデバッグ用コマンド一覧です。

| 目的 | コマンド | 主な確認項目・注目パラメータ |
| :--- | :--- | :--- |
| **Port Security 状態確認** | `show port-security interface <INT>` | Violation Mode, Security Violation Count, Max/Current MAC |
| **Secure MAC アドレス一覧** | `show port-security address` | 学習された MAC アドレス、VLAN、Type (Sticky/Dynamic/Static) |
| **DHCP Snooping 概要** | `show ip dhcp snooping` | 有効 VLAN, Option 82 設定, インターフェイス Trust 状態 |
| **DHCP バインディング DB** | `show ip dhcp snooping binding` | MAC, IP Address, Lease Time, Type, VLAN, Interface |
| **DAI 動作状態** | `show ip arp inspection vlan <VLAN>` | DAI 有効 VLAN, Forwarded/Dropped パケットカウンタ |
| **DAI インターフェイス制限** | `show ip arp inspection interfaces` | Rate Limit (pps), Trust 状態 |
| **IP Source Guard バインディング** | `show ip verify source` | Filter Type (ip / ip-mac), Filter Mode (Active), IP/MAC/VLAN |
| **Storm Control 状態** | `show storm-control` | 物理ポートごとの Rising/Falling 閾値, 現在の利用率, Action |
| **Err-disable 原因確認** | `show interfaces status err-disabled` | Port Security / Storm Control / BPDU Guard による物理ダウン原因 |
| **DAI パケットデバッグ** | `debug ip arp inspection` | 破棄された ARP パケットの詳細ログ出力 |
| **DHCP Snooping イベント** | `debug ip dhcp snooping event` | バインディング登録・破棄イベントログ |

---

## 🚨 トラブルシュート

ラボ試験や実運用で頻出する Layer 2 Security 障害パターンと対処方法のガイドです。

| 症状・障害内容 | 推定原因 | 確認コマンド | 対処方法・ソリューション |
| :--- | :--- | :--- | :--- |
| **DHCPクライアントがIPアドレスを取得できない** | アクセススイッチ間の Trunk ポートが DHCP Snooping Untrusted になっている | `show ip dhcp snooping` | 上位スイッチ/DHCPサーバー方向の Trunk ポートに `ip dhcp snooping trust` を設定 |
| **Option 82 ドロップによる DHCP 失敗** | Untrusted ポートから Option 82 付き DHCP パケットが流入 | `show ip dhcp snooping statistics` | `no ip dhcp snooping information option` または `ip dhcp snooping information option allow-untrusted` を設定 |
| **静的 IP 端末の通信が完全不可** | DAI または IPSG が有効な VLAN でバインディング DB に登録がない | `show ip arp inspection vlan` <br> `show ip verify source` | 静的 IP 端末用に `arp access-list` を作成し DAI に適用、または `ip source binding` を静的追加 |
| **正規ユーザー端末が err-disable 化する** | ARP パケット送信数が DAI Rate Limit (15pps) を超過 | `show interfaces status err-disabled` | 影響ポートの DAI レート制限を緩和 (`ip arp inspection limit rate 50`) し、`errdisable recovery cause arp-inspection` を有効化 |
| **Port Security で Sticky MAC が永続化しない** | スイッチ再起動時に NVRAM へコンフィグが保存されていない | `show running-config interface` | `copy running-config startup-config` を実行して Sticky MAC アドレスを startup-config に保存 |
| **IPSG で送信元 MAC 検証が機能しない** | `ip verify source port-security` を設定したが Port Security が未有効 | `show ip verify source` | 当該インターフェイスで `switchport port-security` を明示的に有効化 |

---

## ⚠ 制限事項

Layer 2 Security 機能の仕様上の制限事項およびプラットフォーム依存事項です。

- **Port Security の制限**:
  - EtherChannel ポートチャネルインターフェイスに対する Dynamic/Sticky MAC 設定の制限。
  - SPAN 宛先ポート（Destination Port）や Dynamic Voice VLAN 設定時の制限。
- **DHCP Snooping の制限**:
  - 内部メモリのみにバインディング DB を保持している場合、スイッチ再起動時に全動的バインディングが消滅。外部 TFTP/FTP/Flash へのデータベースバックアップ（`ip dhcp snooping database`）設定が推奨されます。
- **IP Source Guard の制限**:
  - IPSG は IPv4 トラフィックを対象とします。IPv6 環境では **IPv6 First Hop Security (FHS)** （Binding Integrity Guard, RA Guard）を個別に使用する必要があります。
  - Trunk ポートに対する IPSG 適用はプラットフォーム固有の制限が存在します。

---

## 🔄 他技術との関連

 Layer 2 Security は、インフラストラクチャ全体の他のセキュリティ機構と密接に連携します。

- **Cisco ISE / 802.1X / MAB 統合**:
  - 802.1X 認証成功後、Cisco ISE は Dynamic VLAN Assignment や Dynamic ACL (dACL) を適用します。認証完了後に分配された VLAN 上で DHCP Snooping や DAI が動的にパケット検証を実行します。
- **Control Plane Policing (CoPP) / CPU Protection**:
  - DAI や DHCP Snooping はパケットを CPU で処理・検証するため、悪意のある高レート ARP/DHCP 攻撃に対して CoPP や `ip arp inspection limit rate` で CPU 資源を保護する必要があります。
- **Private VLANs (PVLAN)**:
  - 同一 Secondary VLAN 内の Isolated Port 間通信を L2 レベルで完全に防止し、IPSG/DAI と組み合わせてサブネット内マイクロセグメンテーションを強化します。

---

## 🧩 比較表

L2 セキュリティにおける類似機能の比較表です。

| 比較項目 | DHCP Snooping | Dynamic ARP Inspection (DAI) | IP Source Guard (IPSG) |
| :--- | :--- | :--- | :--- |
| **保護レイヤ** | Application / L7 (DHCP) | Data Link / L2 (ARP) | Network / L3 (IP Header) |
| **検証対象パケット** | UDP 67/68 (DHCP Messages) | ARP Request / ARP Response | 全入出力 IP パケット |
| **依存データソース** | 独自（Trust 判定ルール） | DHCP Snooping DB / ARP ACL | DHCP Snooping DB / Static Binding |
| **主な攻撃防御** | Rogue DHCP Server, DHCP Starvation | ARP Cache Poisoning, MitM | IP Address Spoofing |
| **ドロップ場所** | Control Plane / L7 Parser | Control Plane / ASIC | Hardware TCAM (Port ACL) |

---

## 💡 ベストプラクティス

1. **DHCP Snooping と Option 82 の標準化**: スイッチ間接続（Trunk）においては `ip dhcp snooping trust` を徹底し、アクセススイッチ単体運用時は `no ip dhcp snooping information option` で不要なヘッダー付加によるトラブルを防ぐ。
2. **DAI レート制限と errdisable 自動復旧**: アクセスポートには `ip arp inspection limit rate 15 burst interval 1` を設定し、不意のスパイクによるポートダウン対策として `errdisable recovery cause arp-inspection` および `errdisable recovery interval 300` をグローバル設定する。
3. **バインディング DB の永続化**: スイッチ再起動時の大規模通信障害を防ぐため、`ip dhcp snooping database flash:/dhcp_snoop.db` または外部 TFTP サーバーへの自動同期を構成する。

---


## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルの実践的な設定サンプル例を 10 個掲載します。設定例は省略せず最後まで出力しています。

### サンプル例 1: Port Security (Sticky MAC + Restrict モード + Errdisable Recovery)

#### 問題
SW1 の GigabitEthernet1/0/5 ポートにおいて、最初に接続された 2 台のエンドポイントのみ接続を許可し、未承認の MAC アドレスが検出された場合は通信をドロップしつつ Syslog を発行し、ポートダウンは発生させない設定を行え。また、BPDU を受信した場合はポートを err-disable にし、300秒後に自動復旧させよ。

#### 要件
- 最大接続 MAC アドレス数: 2
- Violation Mode: Restrict
- MAC アドレス学習方式: Sticky MAC
- BPDU 受信時: 即座に PortFast BPDU Guard により err-disable 化
- Errdisable 復旧: BPDU Guard 原因に対して 300 秒で自動リカバリ

#### 設定例

```bash
! --- SW1 グローバル設定 ---
errdisable recovery cause bpduguard
errdisable recovery interval 300

! --- インターフェイス設定 ---
interface GigabitEthernet1/0/5
 description *** Branch Access Port ***
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
```

#### 検証方法
1. `show port-security interface GigabitEthernet1/0/5` を実行し、`Port Security: Enabled`、`Violation Mode: Restrict`、`Maximum MAC Addresses: 2` を確認。
2. 3台目の端末からフレームを送信し、`SecurityViolation Count` が増加すること、および `%PORT_SECURITY-2-PSECURE_VIOLATION` Syslog が出力されることを確認。

---

### サンプル例 2: DHCP Snooping と Option 82 設定の最適化

#### 問題
VLAN 10 および VLAN 20 において DHCP Snooping を有効化せよ。SW1 はコアスイッチ SW2（Gi1/0/24）経由で DHCP サーバーに接続している。スイッチ間の Trunk リンクで Option 82 が付与された DHCP REQUEST パケットが破棄されないよう適切な設定を行え。

#### 要件
- 有効 VLAN: VLAN 10, VLAN 20
- Uplink インターフェイス (Gi1/0/24): Trusted
- Option 82 不整合によるドロップ防止のため、Option 82 挿入を無効化するか Untrusted からの受信を許可

#### 設定例

```bash
! --- SW1 グローバル設定 ---
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option

! --- アップリンクインターフェイス設定 ---
interface GigabitEthernet1/0/24
 description *** Trunk to Core Switch & DHCP Server ***
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 ip dhcp snooping trust
```

#### 検証方法
1. クライアント端末から DHCP DISCOVER を送信し、IP アドレスが正常に割り当てられるか確認。
2. `show ip dhcp snooping` を実行し、Gi1/0/24 が `trusted: yes` であること、および Option 82 処理状態を確認。
3. `show ip dhcp snooping binding` を実行し、割り当てられた IP/MAC/VLAN/Port ログが登録されていることを確認。

---

### サンプル例 3: Dynamic ARP Inspection (DAI) の基本実装とレート制限

#### 問題
VLAN 10 において DAI を有効化し、ARP Cache Poisoning 攻撃を防止せよ。また、アクセスポートにおいて大量の ARP パケットによる CPU 高負荷を防ぐため、1秒間に 20 パケットを超える ARP レートを制限せよ。

#### 要件
- DAI 有効化: VLAN 10
- Uplink (Gi1/0/24): DAI Trust ポート
- Access Ports (Gi1/0/1 - 10): Untrusted, ARP Rate Limit = 20 pps

#### 設定例

```bash
! --- SW1 グローバル設定 ---
ip arp inspection vlan 10

! --- アップリンク設定 ---
interface GigabitEthernet1/0/24
 ip arp inspection trust

! --- アクセスポート設定 ---
interface range GigabitEthernet1/0/1 - 10
 ip arp inspection limit rate 20 burst interval 1
```

#### 検証方法
1. `show ip arp inspection vlan 10` を実行し、`DHCP Logging: Deny`、`Forwarded` / `Dropped` カウンタを確認。
2. 承認されていない IP/MAC ペアで ARP Reply を送信し、スイッチがパケットをドロップして Syslog `%SW_DAI-4-DHCP_REFUSED` を出力することを確認。

---

### サンプル例 4: 静的 IP サーバー環境に対する ARP ACL と DAI の適用

#### 問題
SW1 の VLAN 30 に配置された静的 IP サーバー（IP: 10.1.30.50, MAC: 0050.5688.1122）は DHCP を使用しない。DAI が原因でこのサーバーの通信がブロックされないよう ARP ACL を作成・適用せよ。ACL に合致しないパケットは即座にドロップさせよ。

#### 要件
- ARP ACL 名: `DAI_STATIC_SERVER`
- 許可対象: IP `10.1.30.50` / MAC `0050.5688.1122`
- 適用方式: VLAN 30 に対し `static` キーワード付きでバインド

#### 設定例

```bash
! --- ARP ACL の作成 ---
arp access-list DAI_STATIC_SERVER
 permit ip host 10.1.30.50 mac host 0050.5688.1122

! --- VLAN 30 への DAI フィルタ適用 ---
ip arp inspection vlan 30
ip arp inspection filter DAI_STATIC_SERVER vlan 30 static
```

#### 検証方法
1. `show ip arp inspection interfaces` および `show ip arp inspection vlan 30` で ACL フィルタが適用されているか確認。
2. サーバーからゲートウェイへ Ping を送信し、ARP 応答が正常に転送されることを確認。

---

### サンプル例 5: IP Source Guard (IPSG) と Port Security の同時有効化

#### 問題
SW1 の GigabitEthernet1/0/3 ポートにおいて、盗聴や IP アドレスのなりすましを防止するため、送信元 IP アドレスおよび送信元 MAC アドレスの両方を検証する IP Source Guard を構築せよ。

#### 要件
- バインディングソース: DHCP Snooping Binding Table
- 検証方式: IP + MAC アドレス検証 (`ip verify source port-security`)
- 事前必須設定: Port Security の有効化

#### 設定例

```bash
interface GigabitEthernet1/0/3
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 1
 ip verify source port-security
```

#### 検証方法
1. `show ip verify source` を実行し、Filter type が `ip-mac`、Filter mode が `active` であることを確認。
2. 手動で静的 IP（10.1.1.200）を端末に設定して通信を試み、IPSG により ASIC レベルでパケットが破棄されることを確認。

---

### サンプル例 6: Storm Control による多重閾値帯域制御

#### 問題
SW1 の GigabitEthernet1/0/8 において、ブロードキャストトラフィックが帯域の 10% を超えた場合に制御を開始し、5% まで低下した際に制御を解除せよ。また、マルチキャストパケットが 500 pps を超えた場合はポートを err-disable 化せよ。

#### 要件
- ブロードキャスト: Rising Threshold = 10.0%, Falling Threshold = 5.0%
- マルチキャスト: Threshold = 500 pps, Action = Shutdown & Trap

#### 設定例

```bash
interface GigabitEthernet1/0/8
 switchport mode access
 switchport access vlan 10
 storm-control broadcast level 10.0 5.0
 storm-control multicast level pps 500
 storm-control action shutdown
 storm-control action trap
```

#### 検証方法
1. `show storm-control GigabitEthernet1/0/8` を実行し、設定された Rising/Falling 閾値およびアクションを確認。
2. トラフィックジェネレータで大量のマルチキャストを送信し、ポートが `err-disable` 状態へ移行することを確認。

---

### サンプル例 7: STP Root Guard と BPDU Guard の統合設計

#### 問題
SW1 のアクセスポート群（Gi1/0/1 - 10）には BPDU Guard を適用し、コアスイッチとのアップリンクポート（Gi1/0/23, Gi1/0/24）のうち上位から優位な BPDU が受信された場合に Root の座を保護する Root Guard を適用せよ。

#### 要件
- アクセスポート: BPDU Guard 有効化
- アップリンク指定ポート: Root Guard 有効化

#### 設定例

```bash
! --- アクセスポート群 ---
interface range GigabitEthernet1/0/1 - 10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable

! --- コア/ディストリビューション向けアップリンク ---
interface range GigabitEthernet1/0/23 - 24
 switchport mode trunk
 spanning-tree guard root
```

#### 検証方法
1. Gi1/0/23 の対向で優先度の高い Priority 0 の BPDU を送信し、`root-inconsistent` 状態に移行してブロックされることを確認（`show spanning-tree inconsistentports`）。

---

### サンプル例 8: Private VLAN (PVLAN) と SVI ルーティング設定

#### 問題
SW1 上で VLAN 100 を Primary VLAN、VLAN 101 を Isolated VLAN、VLAN 102 を Community VLAN として構成せよ。また、Primary VLAN の SVI（Vlan 100）を作成し、外部との L3 ルーティングを行えるよう設定せよ。

#### 要件
- Primary VLAN: 100
- Secondary Isolated VLAN: 101
- Secondary Community VLAN: 102
- Promiscuous Port: Gi1/0/24（ルーター接続）
- Isolated Port: Gi1/0/1
- Community Port: Gi1/0/2, Gi1/0/3

#### 設定例

```bash
! --- VLAN 定義 ---
vlan 101
 private-vlan isolated
vlan 102
 private-vlan community
vlan 100
 private-vlan primary
 private-vlan association 101,102

! --- インターフェイスアサイン ---
interface GigabitEthernet1/0/24
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 100 101,102

interface GigabitEthernet1/0/1
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101

interface range GigabitEthernet1/0/2 - 3
 switchport mode private-vlan host
 switchport private-vlan host-association 100 102

! --- SVI ルーティング設定 ---
interface Vlan100
 ip address 10.1.100.1 255.255.255.0
 private-vlan mapping 101,102
```

#### 検証方法
1. `show vlan private-vlan` を実行し、Primary と Secondary のマッピング関係およびポートアサインを確認。
2. Gi1/0/1（Isolated）から Gi1/0/2（Community）への Ping が不成立であり、SVI への Ping が成功することを確認。

---

### サンプル例 9: VACL (VLAN Access Map) による同一 VLAN 内特定通信の制御

#### 問題
VLAN 10 内の全端末間通信において、特定の管理端末（10.1.1.200）以外の端末から内部 Web サーバー（10.1.1.80）への TCP 80 (HTTP) 通信を拒否し、その他の通信は許可する VACL を作成・適用せよ。

#### 要件
- 対象 VLAN: VLAN 10
- 許可通信: 送信元 10.1.1.200 -> 宛先 10.1.1.80 (TCP 80)
- 破棄通信: その他全端末 -> 宛先 10.1.1.80 (TCP 80)

#### 設定例

```bash
! --- 1. IP Access List の定義 ---
ip access-list extended ACL_PERMIT_ADMIN_HTTP
 permit tcp host 10.1.1.200 host 10.1.1.80 eq 80

ip access-list extended ACL_DENY_OTHER_HTTP
 permit tcp any host 10.1.1.80 eq 80

! --- 2. VLAN Access Map の構築 ---
vlan access-map VACL_HTTP_BLOCK 10
 match ip address ACL_PERMIT_ADMIN_HTTP
 action forward

vlan access-map VACL_HTTP_BLOCK 20
 match ip address ACL_DENY_OTHER_HTTP
 action drop

vlan access-map VACL_HTTP_BLOCK 30
 action forward

! --- 3. VLAN 10 への適用 ---
vlan filter VACL_HTTP_BLOCK vlan-list 10
```

#### 検証方法
1. `show vlan filter` および `show vlan access-map` で適用状態を確認。
2. 10.1.1.50 から 10.1.1.80:80 へ接続を試みドロップされること、10.1.1.200 からは接続可能であることを確認。

---

### サンプル例 10: DAI + IPSG + DHCP Snooping + Port Security 完全統合構築

#### 問題
SW1 のアクセスポート GigabitEthernet1/0/10 に対し、Layer 2 脅威を防御するための統合セキュリティプロファイル（Port Security + DHCP Snooping + DAI + IPSG）を実装せよ。

#### 要件
- DHCP Snooping / DAI 有効化 (VLAN 10)
- Port Security: Maximum 1, Violation Restrict, Sticky MAC
- IPSG: IP + MAC 検証
- DAI: Untrusted, Rate limit 15pps

#### 設定例

```bash
! --- グローバル基本設定 ---
ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option
ip arp inspection vlan 10

! --- インターフェイス統合設定 ---
interface GigabitEthernet1/0/10
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 ip verify source port-security
 ip arp inspection limit rate 15
 spanning-tree portfast
 spanning-tree bpduguard enable
```

#### 検証方法
1. `show port-security interface Gi1/0/10`、`show ip verify source`、`show ip arp inspection interfaces Gi1/0/10` の各出力を確認し、すべての防御層がアクティブであることを検証。

---


## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験およびテクノロジー評価を想定した実践的な試験問題群です。

---

### 質問 1 (コンフィグ読解)

以下のスイッチコンフィグが適用された環境において、クライアント PC（Gi1/0/1 接続）が DHCP サーバーから IP アドレスを取得できません。原因として最も適切なものを選択し、その理由を述べよ。

```bash
ip dhcp snooping
ip dhcp snooping vlan 10

interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10

interface GigabitEthernet1/0/24
 switchport mode trunk
 switchport trunk allowed vlan 10
```

**【選択肢】**
A. `ip dhcp snooping trust` が Gi1/0/1 に設定されていないため。
B. Gi1/0/24（アップリンク）に `ip dhcp snooping trust` が設定されていないため。
C. `ip dhcp snooping vlan 10` の設定順序が誤っているため。
D. `ip verify source` が適用されていないため。

#### 正解
**B**

#### 解説
DHCP Snooping が有効な環境では、すべてのインターフェイスがデフォルトで **Untrusted** 状態となります。DHCP サーバーから送信される応答パケット（DHCP OFFER, DHCP ACK など）は Untrusted ポート上でドロップされます。DHCP サーバーが接続されている方向（Gi1/0/24）に対して `ip dhcp snooping trust` を明示的に設定しない限り、クライアントは DHCP レスポンスを受信できません。

---

### 質問 2 (トラブルシュート)

スイッチ SW1 において `ip arp inspection vlan 20` を有効化した直後、VLAN 20 に接続された静的 IP 構成のファイルサーバー（10.1.20.100）がネットワークから不通となりました。DHCP Snooping バインディングテーブルを確認したところ、このサーバーのレコードは存在しませんでした。通信を最小限のコンフィグ変更で復旧させるための最も適切なアプローチはどれか。

**【選択肢】**
A. VLAN 20 全体で DAI を無効化する。
B. サーバーが接続されたポートを `switchport mode trunk` に変更する。
C. サーバーの IP/MAC バインディングを定義した `arp access-list` を作成し、`ip arp inspection filter` で VLAN 20 に適用する。
D. `ip dhcp snooping trust` をサーバー接続ポートに設定する。

#### 正解
**C**

#### 解説
DAI はデフォルトで DHCP Snooping バインディングテーブルを参照して ARP パケットを検証します。静的 IP アドレス構成の端末は DHCP を使用しないためバインディング DB に登録されず、DAI によって ARP がすべて拒否されます。これを解消するには、静的 IP/MAC のペアを定義した `arp access-list` を作成し、`ip arp inspection filter <ACL_NAME> vlan 20` コマンドでバインドする必要があります。

---

### 質問 3 (Design)

同一サブネット（192.168.10.0/24）内に配置された複数のWebサーバー間における直接のレイヤ2通信を全面的に禁止しつつ、既存のルーターやサブネット設定を変更せずにセグメンテーションを実現したい。最も設計効率が高く、管理オーバーヘッドが少ない技術はどれか。

**【選択肢】**
A. 各サーバーを異なる VLAN と IP サブネットに分割する。
B. Private VLAN (PVLAN) を導入し、サーバーポートを Isolated Port にアサインする。
C. すべてのアクセスポートに Standard ACL をインバウンド適用する。
D. Storm Control を適用してトラフィックを絞り込む。

#### 正解
**B**

#### 解説
Private VLANs (PVLAN) は、単一の L3 サブネット（IP 空間）を変更することなく、Layer 2 レベルでの端末間完全隔離（Isolated Port）を提供します。L3 サブネットの切り直しやルーティング設定の変更が不要であるため、最も効率的な設計ソリューションとなります。

---

### 質問 4 (実装)

スイッチ SW1 の GigabitEthernet1/0/2 ポートに対し、`ip verify source port-security` を設定しました。しかし、`show ip verify source` コマンドを実行したところ、Filter Mode が Active にならず、MAC アドレスの検証が正しく行われていません。原因と適切な修正手順を説明せよ。

#### 解答・解説
**原因**: `ip verify source port-security`（IP+MAC 検証）を正常に動作させるためには、対象インターフェイス上で **Port Security (`switchport port-security`) が有効化されていることが必須要件**となります。Port Security が無効な状態では、IP アドレスのみの検証にフォールバックするか、フィルタが正常に機能しません。

**修正手順**:
```bash
interface GigabitEthernet1/0/2
 switchport mode access
 switchport port-security
 ip verify source port-security
```

---


### 質問 5 (トラブルシュート)

ネットワーク管理者がアクセスポート（Gi1/0/4）上で大量の ARP リクエストを発生させたところ、ログに以下のメッセージが出力され、インターフェイスがダウンしました。

```text
%SW_DAI-4-PACKET_RATE_EXCEEDED: 20 packets received in 1000ms on Gi1/0/4
%LINK-3-UPDOWN: Interface GigabitEthernet1/0/4, changed state to down
%PM-4-ERR_DISABLE: arp-inspection error detected on Gi1/0/4, putting Gi1/0/4 in err-disable state
```

この現象の原因を特定し、ポートの自動復旧および将来の誤検知を防ぐためのコンフィグ変更例を示せ。

#### 解答・解説
**原因**: DAI のデフォルトのパケットレート制限（15 pps）を超過する ARP パケットが 1 秒間に入力されたため、`arp-inspection` 違反としてポートが `err-disable` 化されました。

**対策コンフィグ**:
```bash
! --- レート制限の緩和 (例: 50 pps) ---
interface GigabitEthernet1/0/4
 ip arp inspection limit rate 50

! --- err-disable からの自動復旧有効化 ---
errdisable recovery cause arp-inspection
errdisable recovery interval 300
```

---

### 質問 6 (コンフィグ読解)

以下の Port Security コンフィグにおける動作仕様として正しいものを 2 つ選べ。

```bash
interface GigabitEthernet1/0/10
 switchport mode access
 switchport port-security
 switchport port-security maximum 3
 switchport port-security violation protect
 switchport port-security mac-address sticky
```

**【選択肢】**
A. 承認されていない 4 台目の MAC アドレスからパケットを受信した場合、ポートは即座に shutdown する。
B. 承認されていない 4 台目の MAC アドレスからパケットを受信した場合、そのパケットはサイレントドロップされ、Syslog は発行されない。
C. 学習された MAC アドレスは `copy running-config startup-config` を実行しなくても再起動後に保持される。
D. 自動学習された MAC アドレスは running-config 内に `switchport port-security mac-address sticky <MAC>` として追加される。

#### 正解
**B, D**

#### 解説
Violation Mode が `protect` に設定されているため、制限上限を超えた不承認パケットはサイレントドロップされ、違反カウンタの増加や Syslog/SNMP Trap の通知は行われません（選択肢 B が正しい）。また、`sticky` オプションが有効なため、動的に学習された MAC アドレスは running-config 内に `sticky` 行として追加されます（選択肢 D が正しい）。NVRAM に保存して再起動後も保持するには `copy run start` が必須です。

---

### 質問 7 (実装)

自作の小型スイッチが誤ってアクセスポート（Gi1/0/12）に接続された際、そのスイッチが優位な BPDU を送信してネットワーク全体の Root Switch になる事故を防止したい。スイッチポートを err-disable 化させずに、当該ポートが Root Port になることのみをブロックする設定コマンドを示せ。

#### 解答・解説
**設定コマンド**:
```bash
interface GigabitEthernet1/0/12
 spanning-tree guard root
```

**解説**: `spanning-tree guard root`（Root Guard）を有効にすると、当該ポートで優位な BPDU（Superior BPDU）を受信した際にポートを `root-inconsistent` 状態にし、トラフィック転送をブロックします。BPDU Guard と異なりポート全体を `err-disable` にしないため、上位 BPDU の受信が止まれば自動的に通常状態に復旧します。

---

### 質問 8 (トラブルシュート)

スイッチ SW1 において、DHCP Snooping と DAI を有効化したところ、ネットワーク上の端末間で ARP 応答が正しく処理されず通信不能になりました。デバッグログを確認したところ `src-mac mismatch` というエラーが多数出力されています。この問題の原因と対策を説明せよ。

#### 解答・解説
**原因**: DAI の検証オプションとして `ip arp inspection validate src-mac` が有効化されている環境において、ARP ヘッダー内の Sender MAC アドレスと、Ethernet フレームヘッダーの Source MAC アドレスが一致していないパケット（不正なフレームまたは一部のロードバランサ/仮想化環境の不整合）が検出されドロップされています。

**対策**:
1. 不正なパケットを送信している端末の不整合を修正する。
2. 検証オプションから `src-mac` チェックを除外する：
```bash
ip arp inspection validate dst-mac ip
```

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術資料の参照先一覧です。

- Cisco Catalyst 9300 Series Switches Security Configuration Guide - Port Security
- Cisco Catalyst 9300 Series Switches Security Configuration Guide - DHCP Snooping
- Cisco Catalyst 9300 Series Switches Security Configuration Guide - Dynamic ARP Inspection (DAI)
- Cisco Catalyst 9300 Series Switches Security Configuration Guide - IP Source Guard (IPSG)
- Cisco Live - BRKSEC-2009: Layer 2 Security Attacks and Mitigation Techniques
- Cisco Technical Support & Documentation - Understanding and Troubleshooting Dynamic ARP Inspection (DAI)
- Cisco Validated Design (CVD) - Campus Network Security Design Guide

---

## 📝 **補足（Notes）**

### Layer 2 Security 技術の依存関係マトリクス

| セキュリティ機能 | 必須の事前有効化機能 | 推奨併用機能 |
| :--- | :--- | :--- |
| **DHCP Snooping** | なし | DB External Backup |
| **DAI** | DHCP Snooping / ARP-ACL | CoPP / Rate Limit |
| **IP Source Guard** | DHCP Snooping | Port Security (IP+MAC) |
| **Port Security** | なし | BPDU Guard / PortFast |
| **PVLAN** | VACL / Private VLAN | SVI Mapping |

### 設計上のゴールデンルール
1. **信頼境界の定義**: トランクポート、アップリンクポート、DHCP サーバー接続ポートには常に `trust` を与え、アクセスポートには厳格な `untrusted` 検証を適用する。
2. **ログとアラートの可視化**: 違反発生時に即座にログを把握できるよう、`logging buffered` および Syslog サーバー連携を必須とする。
3. **ハードウェア TCAM リソースの意識**: IPSG や VACL の多用はスイッチの TCAM リソースを消費するため、大規模環境では定常的なリソースモニタリングを実施する。

---
