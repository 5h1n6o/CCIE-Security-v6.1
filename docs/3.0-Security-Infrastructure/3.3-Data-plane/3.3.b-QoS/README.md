---
layout: default
title: 3.3.b-QoS
nav_order: 2
parent: 3.3-Data-plane
grand_parent: 3.0-Security-Infrastructure
---

# 3.3.b QoS

## 📘 概要

Quality of Service（QoS）は、ネットワークインフラストラクチャおよびデータプレーンにおいて、特定のトラフィッククラス（音声、ビデオ、ミッションクリティカルデータなど）に対して適切な帯域幅、遅延（Latency）、ジッター（Jitter）、およびパケット損失率（Packet Loss）の目標値を保証するための技術的体系です。CCIE Security v6.1におけるQoSは、単なる帯域制限にとどまらず、暗号化VPN（IPsec / DMVPN / FlexVPN）環境におけるパケット分類の維持、ファイアウォール（ASA / FTD）でのバースト攻撃や帯域枯渇に対するデータプレーン保護、および管理・制御トラフィックの優先処理というセキュリティインフラの可用性担保において極めて重要視されます。

主な利用目的および適用場面は以下の通りです。

- **セキュリティインフラのデータプレーン保護**: 帯域枯渇攻撃（DDoSや大量の非承認トラフィック）発生時に、IPsec VPN制御パケット、管理通信（SSH, SNMP, TACACS+）、および音声・リアルタイム通信を保護します。
- **暗号化トンネル（IPsec / DMVPN / FlexVPN）でのQoS継承**: カプセル化前のToS/DSCP値を外側IPヘッダーにコピー（ToS Reflection）または`qos pre-classify`を適用し、暗号化パケットに対しても適切な優先制御およびシェーピング（HQoS）を実行します。
- **マルチタスクファイアウォールでの帯域制御**: Cisco ASA（MPF: Modular Policy Framework）や Cisco FTD（FMC QoS Policy）にて、特定のアプリケーションやインターフェイスごトラフィックを帯域制限（Policing）し、リソース枯渇を防止します。

---

## 🔑 要点

QoSの全体像をセキュリティインフラ観点から表形式で整理します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | ネットワークトラフィックの分類（Classification）、マーキング（Marking）、輻輳管理（Congestion Management）、輻輳回避（Congestion Avoidance）、帯域調整（Policing & Shaping）を提供し、データプレーンの品質と可用性を維持する |
| **用途** | 音声/ビデオ（EF/AF41）の優先転送、DDoS/大量トラフィック時の重要通信の帯域保護、暗号化VPNトンネルでのQoSポリシー適用（HQoS）、ファイアウォールでの帯域制御 |
| **メリット** | トラフィックの重要度に応じた帯域保証、遅延・ジッターの極小化、帯域枯渇時でも制御・セキュリティ通信の疎通を維持可能 |
| **デメリット** | 設定の複雑化、CPUおよびスイッチングASIC/QFP（Quantum Flow Processor）へのオーバーヘッド加重、暗号化通信での分類難易度上昇 |
| **対応機種** | Cisco IOS-XE ルーター（ASR1000, ISR4000, Catalyst 8000）、Catalyst スイッチ、Cisco ASA 5500-X、Cisco Secure Firewall (FTD / FMC) |
| **制限事項** | トンネル暗号化後はペイロードのL4/L7情報が隠蔽されるため、ToSバンプや`qos pre-classify`が必要。ASA/FTDではIOS-XEに比べ高度なキューイング（WRED等）が非対応 |
| **設計上の注意点** | 信頼境界（Trust Boundary）の明確化、IPsecアンチリプレイウィンドウと優先キューイングの順序逆転問題の回避、HQoSにおける親ポリシー（Shape）と子ポリシー（Priority/CBWFQ）の適合性 |

---

## 🏗 動作原理

QoSは「分類・マーキング」「ポリシング・シェーピング」「キューイング・輻輳回避」の3つの主要コンポーネントで構成され、データプレーン上でパケットを処理します。

### 1. QoSアーキテクチャモデル
QoS処理は、ルーター/スイッチのパケットパスにおいて以下のフローで実行されます。

```
Ingress Interface
   ↓
[ Classification & Marking ] (ACL, CoS/DSCP, NBAR2)
   ↓
[ Ingress Policing ] (srTCM / trTCM Token Bucket)
   ↓
[ Routing / Switching & Pre-classify ] (FIB Look-up, IPsec Encapsulation, ToS Byte Copy)
   ↓
[ Congestion Avoidance ] (WRED / Tail Drop)
   ↓
[ Congestion Management (Queuing) ] (LLQ Priority Queue + CBWFQ Bandwidth Queues)
   ↓
[ Egress Shaping / Policing ] (Token Bucket Rate Limiter)
   ↓
Egress Interface
```

### 2. 暗号化VPNにおけるQoS動作原理 (IPsec / DMVPN / FlexVPN)
IPsecでパケットが暗号化されると、元のL3 (IP Header) および L4 (TCP/UDP Header) はカプセル化され、外側からは参照不能になります。Cisco IOS-XEでは、デフォルトでインナーIPヘッダーのToS/DSCPバイトがアウターIPヘッダーにコピー（ToS Reflection）されます。

しかし、L4ポートやACL、NBARによる高度な分類をカプセル化後（Egress）で実施したい場合、`qos pre-classify`機能を利用します。これにより、暗号化処理前に内部パケットに基づいてQoS分類用メタデータが保持され、トンネルインターフェイスまたは物理インターフェイスでのQoS適用が可能となります。

DMVPN / FlexVPN環境におけるハブ・スポーク型設計では、スポーク対ハブの帯域不整合（Hub-to-Spoke Bandwidth Mismatch）に対処するため、親ポリシーでシェーピング（`shape average`）、子ポリシーで優先キューイング（`priority` / `bandwidth`）を行う階層型QoS（HQoS - Hierarchical QoS）を適用します。

```
[ Inner Packet: Src 10.1.1.10, Dst 10.2.2.20, Port 80 (DSCP EF) ]
   ↓ (IPsec Encapsulation)
[ Outer Packet: Src 192.168.1.1 (Hub), Dst 192.168.1.2 (Spoke) (Outer DSCP EF) ]
   ↓ (HQoS Parent Policy: Shape to Spoke Rate e.g., 2Mbps)
   ↓ (HQoS Child Policy: LLQ Priority Queue for EF, CBWFQ for Class-Data)
Egress Interface (Physical WAN)
```

---

## ⚙ 動作シーケンス

パケットがQoS対応のネットワーク機器を通過する際の詳細な処理手順は以下の通りです。

```
1. パケット受信 (Ingress)
   ↓
2. 分類 (Classification)
   - L2 CoS (802.1p), L3 DSCP/IP Precedence, IP ACL, または NBAR2 によるL7アプリケーション識別
   ↓
3. マーキング / 書き換え (Marking / Remarking)
   - 入力トラフィックのDSCP/CoS値を指定クラスの値（例: DSCP EF = 46, AF31 = 26）へ変更
   ↓
4. 入力ポリシング (Ingress Policing)
   - トークンバケットアルゴリズム（Single/Dual Rate）に基づき、規定レート（CIR/PIR）を超過するトラフィックのドロップまたはマーキングダウン（Remarking）
   ↓
5. ルーティング & 暗号化処理 (Routing & Encapsulation)
   - L3ルーティング処理。IPsecトンネル通過時、インナーDSCPがアウターIPヘッダーへ転写（ToS Copy）
   - `qos pre-classify`有効時、暗号化前の元パケット情報に基づくQoS分類タグが保持される
   ↓
6. 輻輳回避 (Congestion Avoidance)
   - キューの蓄積状態の監視。WREDにより、DSCP/Precedenceに基づいて優先度の低いパケットを閾値超過時に早期ドロップ（Tail Drop防止）
   ↓
7. 輻輳管理 / キューイング (Congestion Management)
   - 出力キューへの振り分け（LLQ: Priority Queue, CBWFQ: Bandwidth Queues）
   - LLQ（Priority Queue）に入ったパケットは他のキューより最優先で送信
   ↓
8. 出力シェーピング / ポリシング (Egress Shaping / Policing)
   - WAN帯域に合わせて送信レートを平滑化（Shaping: トークン待ちバッファリング）または超過制限（Policing: ドロップ）
   ↓
9. パケット送信 (Egress)
```

---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Securityラボ試験においてQoSが問われるポイントと、設定時の重要な注意点を網羅します。

### 1. Blueprintで重要なポイント
- **MQC (Modular QoS CLI) の正確な構築構造**: `class-map` -> `policy-map` -> `service-policy` の階層構造。
- **暗号化トンネル上のQoS設定 (`qos pre-classify`)**: Crypto Map、Tunnel インターフェイス、または Virtual-Template への正しい適用。
- **DMVPN / FlexVPN での HQoS (Hierarchical QoS)**: 親ポリシーに `shape average`、子ポリシーに `priority` や `bandwidth` を定義する階層構造。
- **Cisco ASA での MPF (Modular Policy Framework) QoS**: `class-map` -> `policy-map` -> `service-policy` を用いた `priority-queue` および `police` 設定。
- **Cisco FTD / FMC での QoS Policy**: インターフェイス単位のレート制限およびアプリケーション別ポリシング設定。

### 2. ラボ試験で設定させられそうな内容・要求パターン
- **要件例1**: 「音声トラフィック（UDP 16384-32767）を DSCP EF（46）にマーキングし、WAN出力量として最大 512Kbps の優先帯域（LLQ）を割り当てよ。帯域超過時はドロップすること。」
- **要件例2**: 「DMVPN スポーク向けの通信に対し、物理WAN出力をスポーク回線速度（2Mbps）にシェーピングし、その内部で管理通信（SSH/TACACS+）用に20%の帯域を保証せよ。」
- **要件例3**: 「IPsec トンネルインターフェイスにおいて、暗号化前の内部IP/Port情報に基づいてQoS分類を行うよう設定せよ (`qos pre-classify`)。」
- **要件例4**: 「Cisco ASA において、Management トラフィック用の優先キューを構成し、特定グループのダウンロード帯域を 5Mbps に制限せよ。」

### 3. よくある設定ミス・落とし穴
- **`qos pre-classify` の適用場所ミス**: `qos pre-classify` コマンドは、Physical インターフェイスではなく **Tunnel インターフェイス** または **Crypto Map** に設定する必要があります。
- **LLQ (Priority Queue) での制限値未指定**: `priority` コマンド単体（パラメータなし）で使用すると、他クラスが酷い輻輳に陥った際にPriority Queueが全帯域を占有し、制御通信（BGP/EIGRP）がダウンする危険があります。ラボでは必ず帯域（例: `priority 512` や `priority percent 20`）を指定します。
- **HQoSでの親/子ポリシーの逆指定**: HQoSでは、親ポリシー（Outer）に `shape` や `police` を配置し、子ポリシー（Inner）に `priority` や `bandwidth` を配置しなければなりません（逆にするとコンフィグエラーになります）。
- **ASA MPF での `priority-queue` の有効化忘れ**: ASAでポリシーマップ内に `priority` を設定しても、対象インターフェイス上でグローバルに `priority-queue <interface-name>` を有効化しないと優先キューイングが動作しません。

### 4. コンフィグ・debug・showコマンドの確認ポイント
- `show policy-map interface <interface>`: パケット数、バイト数、ドロップ数（Policed drops, Priority drops）が実際にカウントアップしているかを確認します。
- `show crypto ipsec sa`: ToS Copyが有効になっているか、および暗号化パケットのQoSマーキングが維持されているかを確認します。



---

## 🛠 設定方法

各種プラットフォーム（Cisco IOS-XE, Cisco ASA, Cisco FTD）におけるQoSの標準的なコンフィグレーション例を解説します。

### 1. Cisco IOS-XE MQC (Modular QoS CLI) 基本設定
標準的な分類（Classification）、マーキング（Marking）、LLQ（Low Latency Queuing）、および CBWFQ（Class-Based Weighted Fair Queuing）の設定です。

```text
! 1. アクセスリストによるトラフィック分類の定義
ip access-list extended ACL-VOICE
 permit udp any any range 16384 32767
ip access-list extended ACL-CRITICAL-MGMT
 permit tcp any any eq 22
 permit tcp any any eq 49
 permit udp any any eq 123

! 2. Class-Map の作成
class-map match-any CM-VOICE
 match access-group name ACL-VOICE
 match dscp ef
class-map match-any CM-MGMT
 match access-group name ACL-CRITICAL-MGMT
 match dscp cs6

! 3. Policy-Map の定義 (LLQ + CBWFQ + Policing)
policy-map PM-WAN-OUT
 class CM-VOICE
  set dscp ef
  priority 512
 class CM-MGMT
  set dscp cs6
  bandwidth percent 15
 class class-default
  fair-queue
  random-detect dscp-prec-based

! 4. 物理インターフェイスへの適用
interface GigabitEthernet0/0/1
 service-policy output PM-WAN-OUT
```

### 2. IPsec / DMVPN / FlexVPN 環境における QoS (`qos pre-classify` & HQoS)
暗号化前のインナーパケットに基づいてQoS分類を行い、かつハブ側でスポーク帯域（2Mbps）に合わせて階層型シェーピングを行う設定です。

```text
! --- スポーク用階層型QoS (HQoS) ポリシー ---
policy-map PM-CHILD-SPOKE
 class CM-VOICE
  priority 256
 class CM-MGMT
  bandwidth 128
 class class-default
  fair-queue

policy-map PM-PARENT-SPOKE-2M
 class class-default
  shape average 2000000
  service-policy PM-CHILD-SPOKE

! --- Tunnel インターフェイス設定 (DMVPN / FlexVPN) ---
interface Tunnel0
 ip address 172.16.1.1 255.255.255.0
 tunnel source GigabitEthernet0/0/0
 tunnel mode gre multipoint
 ! 暗号化前のパケットでQoS分類を可能にする
 qos pre-classify
 service-policy output PM-PARENT-SPOKE-2M
```

### 3. Cisco ASA MPF (Modular Policy Framework) QoS 設定
ASAにおいて、管理トラフィックの優先キューイングおよび特定トラフィックのポリシング（帯域制限）を構成します。

```text
! 1. トラフィック識別用 Access-list
access-list ACL-ASA-VOICE extended permit udp any any range 16384 32767
access-list ACL-GUEST-LIMIT extended permit ip 192.168.100.0 255.255.255.0 any

! 2. Class-Map の定義
class-map CM-ASA-VOICE
 match access-list ACL-ASA-VOICE
class-map CM-GUEST-LIMIT
 match access-list ACL-GUEST-LIMIT

! 3. Policy-Map の定義
policy-map PM-ASA-QOS
 class CM-ASA-VOICE
  priority
 class CM-GUEST-LIMIT
  ! 2Mbps にポリシング、超過分はドロップ
  police output 2000000 37500

! 4. インターフェイスへの適用と Priority Queue の有効化
policy-map global_policy
 class class-default
  ! 他のグローバルポリシー

service-policy PM-ASA-QOS interface outside

! 優先キューをインターフェイス上で物理的に有効化
priority-queue outside
```

### 4. Cisco FTD (FMC 管理) QoS Policy 設定
FMC（Firepower Management Center）のGUIから設定するQoSポリシーの概念構成です。

1. **FMC GUI** にログイン: **Devices** -> **QoS** に移動。
2. **Add QoS Policy** をクリックし、名前を指定（例: `FTD-QoS-Policy`）。
3. **Add Rule** を選択:
   - **Target**: 対象の FTD デバイスおよび Interface（例: `outside`）を選択。
   - **Rate Limiting**: `Download / Upload Rate Limit`（Kbps または Mbps）を指定。
   - **Conditions**: Source/Destination Network、Protocol/Port、または Application（例: `YouTube`, `Bittorrent`）を指定してポリシングルールを定義。
4. **Deploy**: 変更内容を対象の FTD デバイスへデプロイ。

---

## 🔍 検証コマンド

QoSの動作状態、パケットカウント、ドロップ状況を確認するためのコマンド一覧です。

| 目的 | コマンド | 補足・確認ポイント |
| --- | --- | --- |
| **QoSポリシー動作確認** | `show policy-map interface <interface>` | 各クラスのパケット数、バイト数、優先キュー（Priority）のドロップ数、ポリサーのドロップ数を確認 |
| **Class-Map 設定確認** | `show class-map` | 定義されたMatch条件（ACL, DSCP, CoS等）の一覧確認 |
| **Policy-Map 設定確認** | `show policy-map` | ポリシーマップの構成およびネストされた子ポリシー（HQoS）の階層確認 |
| **NBAR2 統計確認** | `show ip nbar protocol-discovery` | リアルタイムで検出されているアプリケーション別トラフィック量の確認 |
| **トンネル QoS 確認** | `show tunnel interface tunnel <id>` | `qos pre-classify` の有効化状態の確認 |
| **ASA QoS 状態確認** | `show service-policy interface <if-name>` | ASAのMPF QoSポリシーによる優先処理数・ポリシングドロップ数の確認 |
| **ASA 優先キュー状態確認** | `show priority-queue statistics <if-name>` | ASAのインプレース優先キューのキュー長、オーバーフロー・ドロップ確認 |
| **QoSデバッグ (IOS-XE)** | `debug priority` | Priority Queueのリアルタイム処理およびドロップ状態のデバッグ |
| **QoSデバッグ (QFP)** | `show platform hardware qfp active feature qos interface <if>` | ASR1000/ISR4000におけるハードウェアQFP処理レベルでのQoS統計・ドロップ確認 |

---

## 🚨 トラブルシュート

QoS運用時およびラボ試験で発生しやすい障害症状と原因・対処方法です。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **Priority Queue でパケットドロップが発生する** | 設定された `priority` 帯域幅の上限を超過したトラフィックが流入している | `show policy-map interface` (Priority drop コールアウト確認) | 実際の音声/リアルタイム通信量に合わせて `priority` 帯域幅を拡大するか、マーキング設定を見直す |
| **IPsecトンネル通過後にQoS分類が機能しない** | 暗号化によりL4ポート情報が隠蔽され、EgressでACL/NBARマッチに失敗している | `show policy-map interface` (対象クラスのパケットが0のまま) | Tunnel インターフェイスに `qos pre-classify` を追加し、暗号化前のメタデータでマッチングを行う |
| **DMVPN/FlexVPN で他通信が切断される** | スポーク側の回線速度を超えた通信がハブから送信され、プロバイダでTail Dropが発生している | `show policy-map interface` | ハブ側で対象スポーク向けに親ポリシーで `shape average` を設定した HQoS を適用する |
| **ASA で Priority Queue が機能しない** | ポリシーマップ内で `priority` を定義したが、物理インターフェイスで優先キューが有効化されていない | `show service-policy interface` | グローバルコンフィグで `priority-queue <interface-name>` コマンドを実行して物理キューを有効化する |
| **WRED が動作せずパケットが一括ドロップされる** | パケットがWREDの最小閾値（Min-threshold）に達する前にテールドロップが発生している、または帯域保証が未設定 | `show policy-map interface` (WRED statistics) | ポリシーマップ内で対象クラスに `bandwidth` または `shape` を設定し、WREDのキュー長を適切に調整する |

---

## ⚠ 制限事項

QoSを実装する際の制限事項およびプラットフォーム固有の依存性です。

- **IPsec Anti-Replay ウィンドウとの干渉**: LLQ（Priority Queue）を使用すると、音声パケットが他のデータパケットを追い越して送信されます。追越しの幅（Packet Reordering）が大きすぎると、受信側のIPsec SAのAnti-Replay Window（デフォルト64パケット）から外れ、正当なパケットがAnti-Replayによって破棄される場合があります。対策として、Anti-Replay ウィンドウサイズを拡大（`crypto ipsec security-association replay window-size 1024`）します。
- **ASA/FTD での機能制限**: ASA および FTD は、IOS-XE のような高度な WRED や NBAR2 に基づく動的 HQoS をフルサポートしていません。ASA の QoS は主に単純な Priority Queuing および Rate Limiting (Policing) に限定されます。
- **物理インターフェイスとサブインターフェイスの適用制限**: プラットフォームによっては、HQoS 親ポリシーのシェーピングをサブインターフェイスやトンネルに適用する場合、ハードウェアASICのキューイング仕様に制約を受ける場合があります。

---

## 🔄 他技術との関連

QoSと他のCCIE Security関連技術との相互作用です。

- **IPsec / GRE / DMVPN / FlexVPN**: カプセル化によるヘッダーの隠蔽に対処するため、`qos pre-classify` および ToS Reflection（DSCP Copy）が必須となります。DMVPN HQoSではNHRPやPer-Tunnel QoS機能と連携して動的に帯域を制御します。
- **Access Control / Firewall (ASA / FTD)**: データプレーン保護として、非認可アプリケーション（P2P, 動画ストリーミング等）に対する帯域制限（Policing）や、制御・管理トラフィックの優先度維持をファイアウォールレベルで実行します。
- **Control Plane Policing (CoPP)**: データプレーンのQoS（MQC）がユーザー通信の帯域を制御するのに対し、CoPPはCPUに流入するコントロールプレーン通信（BGP, OSPF, SSH等）をMQC形式でポリシング・保護します。

---

## 🧩 比較表

主要なQoS技術概念の比較です。

### 1. ポリシング (Policing) vs シェーピング (Shaping)

| 項目 | ポリシング (Policing) | シェーピング (Shaping) |
| --- | --- | --- |
| **動作メカニズム** | 規定レートを超過したパケットを**即座にドロップ**またはリマーク | 規定レートを超過したパケットを**バッファ（キュー）に保持**し平滑化して送信 |
| **適用方向** | Ingress（受信）および Egress（送信）の両方 | Egress（送信）のみ |
| **パケット遅延** | 遅延を増加させない（バッファリングなし） | バッファリングにより遅延・ジッターが増加する可能性あり |
| **主な用途** | 契約帯域の強制制限、DDoS攻撃/特定アプリの帯域上限切り捨て | 回線速度の不整合（Bandwidth Mismatch）回避、WANルーター出力制御 |
| **使用トークンバケット** | Single-Rate / Dual-Rate Token Bucket ($CIR, PIR, Bc, Be$) | Single-Rate Token Bucket ($CIR, Bc, Be, Tc$) |

### 2. L2 CoS vs L3 DSCP

| 項目 | CoS (Class of Service) | DSCP (Differentiated Services Code Point) |
| --- | --- | --- |
| **定義レイヤー** | Layer 2（IEEE 802.1Q VLAN Tag 内 802.1p 3ビット） | Layer 3（IPv4 ToS バイト / IPv6 Traffic Class 内 6ビット） |
| **値の範囲** | 0 〜 7（8段階） | 0 〜 63（64段階: EF, AF11-AF43, CS0-CS7, Default） |
| **透過性** | ルーター（L3ホップ）を越えると802.1Qタグ削除により消失 | ルーターを越えてエンドツーエンドでIPヘッダー内に保持可能 |
| **主な標準クラス** | CoS 5 (Voice), CoS 3 (Call Signaling), CoS 0 (Best Effort) | DSCP 46 (EF), DSCP 26 (AF31), DSCP 48 (CS6) |

---

## 💡 ベストプラクティス

セキュリティインフラにおけるQoS設計・運用の推奨標準です。

1. **RFC 4594（DiffServ アセスメントガイドライン）の準拠**:
   - Voice: DSCP EF (46) - LLQ (Priority Queue)
   - Video: DSCP AF41 (34) - Guaranteed Bandwidth
   - Network Control / Mgmt: DSCP CS6 (48) / CS2 (16) - Guaranteed Bandwidth
   - Best Effort: DSCP 0 - Fair Queue
2. **信頼境界（Trust Boundary）の早期定義**: アクセススイッチのポートレベルでエンドポイントからのDSCP/CoSを信頼（Trust）するか書き換える（Untrust/Remark）かを決定し、コア/WANで不当なDSCP EF通信が優先されないように保護します。
3. **LLQ 占有率の制限（33% ルール）**: Priority Queue に割り当てる帯域は、回線全体帯域の 33% 以下に抑え、輻輳時に他のミッションクリティカルデータやルーティング制御通信（CS6）が餓死（Starvation）しないように設計します。
4. **IPsec Replay Window の拡張**: LLQを併用する暗号化VPN環境では、Anti-Replay Window を 1024 パケット以上に拡張し、優先パケット追越しによる誤ドロップを防止します。



---

## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験レベルの実践的なQoS構築・検証シナリオ10選です。設定例は省略せず最後まで記載しています。

### シナリオ 1: MQC 分類とマーキング (Classification & Marking)

#### 問題
企業エッジルーター R1 において、社内端末からの入出力トラフィックを特定し、適切な DSCP 値にマーキングする MQC ポリシーを構築してください。

#### 要件
1. UDP ポート 16384-32767 の音声トラフィックをクラス `CM-VOICE` として識別し、DSCP `ef` (46) にマーキングすること。
2. SSH (TCP 22) および TACACS+ (TCP 49) 通信をクラス `CM-SEC-MGMT` として識別し、DSCP `cs6` (48) にマーキングすること。
3. その他の通信は `class-default` として扱い、DSCP `default` (0) にリマークすること。
4. インターフェイス `Gi0/0/1` の Ingress 方向にポリシーを適用すること。

#### 設定例

```text
! 1. アクセスリストの定義
ip access-list extended ACL-VOICE-RTP
 permit udp any any range 16384 32767
ip access-list extended ACL-SEC-MGMT
 permit tcp any any eq 22
 permit tcp any any eq 49

! 2. Class-Map の定義
class-map match-all CM-VOICE
 match access-group name ACL-VOICE-RTP

class-map match-all CM-SEC-MGMT
 match access-group name ACL-SEC-MGMT

! 3. Policy-Map の定義 (Marking)
policy-map PM-MARKING-INGRESS
 class CM-VOICE
  set dscp ef
 class CM-SEC-MGMT
  set dscp cs6
 class class-default
  set dscp default

! 4. インターフェイスへの適用
interface GigabitEthernet0/0/1
 service-policy input PM-MARKING-INGRESS
```

#### 検証方法
`show policy-map interface GigabitEthernet0/0/1 input` を実行し、該当ポート経由のパケットに対して `CM-VOICE` および `CM-SEC-MGMT` のパケットカウントが増加し、DSCP値が期待通り書き換わっていることを確認します。

---

### シナリオ 2: LLQ (Low Latency Queuing) & CBWFQ による WAN 出力制御

#### 問題
WANルーター R2 の出力インターフェイスにおいて、音声トラフィックの極小遅延転送（LLQ）およびミッションクリティカル通信の帯域保証（CBWFQ）を設定してください。

#### 要件
1. DSCP `ef` のパケットに対して、512Kbps の優先帯域（Priority Queue）を割り当てること。
2. DSCP `cs6` の管理通信に対して、最低 256Kbps の帯域を保証すること。
3. DSCP `af31` の Web データ通信に対して、最低 1024Kbps の帯域を保証すること。
4. 物理インターフェイス `Gi0/0/2` の Egress 方向に適用すること。

#### 設定例

```text
! 1. Class-Map の定義
class-map match-any CM-EF-VOICE
 match dscp ef
class-map match-any CM-CS6-MGMT
 match dscp cs6
class-map match-any CM-AF31-DATA
 match dscp af31

! 2. Policy-Map の定義 (LLQ + CBWFQ)
policy-map PM-Egress-Queuing
 class CM-EF-VOICE
  priority 512
 class CM-CS6-MGMT
  bandwidth 256
 class CM-AF31-DATA
  bandwidth 1024
 class class-default
  fair-queue

! 3. インターフェイスへの適用
interface GigabitEthernet0/0/2
 service-policy output PM-Egress-Queuing
```

#### 検証方法
`show policy-map interface GigabitEthernet0/0/2 output` を実行し、`CM-EF-VOICE` クラス配下で Priority Queue の動作状態および Priority Drop が発生していないか確認します。

---

### シナリオ 3: 入力ポリシング (srTCM 2-Color Policing) による帯域制限

#### 問題
アクセスルーター R3 において、特定のゲストネットワーク（192.168.99.0/24）からの受信トラフィックを 1Mbps に制限するポリシングを設定してください。

#### 要件
1. 192.168.99.0/24 からの受信通信をクラス `CM-GUEST` として識別すること。
2. 平均レート（CIR）1Mbps、バーストサイズ（Bc）31250 バイトの単一レート2カラーポリサー（srTCM）を構成すること。
3. 規定レート内の通信は送信（`transmit`）し、超えた通信はドロップ（`drop`）すること。
4. `Gi0/0/1` の Ingress に適用すること。

#### 設定例

```text
! 1. ACLおよびClass-Map定義
ip access-list extended ACL-GUEST-NET
 permit ip 192.168.99.0 0.0.0.255 any

class-map match-all CM-GUEST
 match access-group name ACL-GUEST-NET

! 2. Policy-Map（Policing）定義
policy-map PM-INGRESS-POLICE
 class CM-GUEST
  police 1000000 31250 conform-action transmit exceed-action drop

! 3. インターフェイス適用
interface GigabitEthernet0/0/1
 service-policy input PM-INGRESS-POLICE
```

#### 検証方法
ゲストセグメントから 2Mbps 相当の iperf パケットを送出し、`show policy-map interface GigabitEthernet0/0/1 input` にて `conformed` が約 1Mbps、`exceeded` がドロップされていることを確認します。

---

### シナリオ 4: クラスベース・シェーピング (Class-Based Shaping) と階層型 QoS

#### 問題
本社ルーター R4 から拠点向け 10Mbps 専用線への送信において、回線契約帯域（10Mbps）へのシェーピングと内部での優先キューイングを組み合わせた HQoS を設定してください。

#### 要件
1. 子ポリシー `PM-CHILD-QUEUING` を作成し、DSCP `ef` に 1Mbps の LLQ、DSCP `cs6` に 512Kbps の CBWFQ を割り当てること。
2. 親ポリシー `PM-PARENT-SHAPE` を作成し、全体通信を 10Mbps (`shape average 10000000`) にシェーピングすること。
3. 親ポリシー内に子ポリシーを組み込み、`Gi0/0/0` の Egress に適用すること。

#### 設定例

```text
! 1. 子ポリシー用 Class-Map
class-map match-any CM-VOICE
 match dscp ef
class-map match-any CM-MGMT
 match dscp cs6

! 2. 子ポリシー (Queuing)
policy-map PM-CHILD-QUEUING
 class CM-VOICE
  priority 1000
 class CM-MGMT
  bandwidth 512
 class class-default
  fair-queue

! 3. 親ポリシー (Shaping + Child Policy)
policy-map PM-PARENT-SHAPE
 class class-default
  shape average 10000000
  service-policy PM-CHILD-QUEUING

! 4. 物理インターフェイスへの適用
interface GigabitEthernet0/0/0
 service-policy output PM-PARENT-SHAPE
```

#### 検証方法
`show policy-map interface GigabitEthernet0/0/0 output` を実行し、親クラスで Shape 統計が表示され、子クラスで Priority / Bandwidth のキューイング統計が正しくネスト表示されていることを確認します。

---

### シナリオ 5: IPsec / DMVPN での `qos pre-classify` と ToS 保持の確認

#### 問題
R5（Hub）と R6（Spoke）間の IPsec / DMVPN トンネルにおいて、暗号化前の元パケットの L4 ポート情報に基づいて Egress QoS を正しく適用できるように設定してください。

#### 要件
1. トンネルインターフェイス `Tunnel10` において、`qos pre-classify` を有効化すること。
2. 暗号化前のパケットで SSH (Port 22) を識別し、優先帯域を割り当てる QoS ポリシーを `Tunnel10` に適用すること。

#### 設定例

```text
! 1. SSH識別用 Class-Map
ip access-list extended ACL-SSH
 permit tcp any any eq 22

class-map match-all CM-SSH
 match access-group name ACL-SSH

! 2. QoS Policy
policy-map PM-TUNNEL-QOS
 class CM-SSH
  bandwidth percent 20
 class class-default
  fair-queue

! 3. DMVPN Tunnel インターフェイス設定
interface Tunnel10
 ip address 172.16.10.1 255.255.255.0
 tunnel source GigabitEthernet0/0/0
 tunnel mode gre multipoint
 qos pre-classify
 service-policy output PM-TUNNEL-QOS
```

#### 検証方法
Tunnel10 経由で SSH 通信を発生させ、`show policy-map interface Tunnel10` を実行して `CM-SSH` クラスでパケットが正しくマッチ・処理されていることを確認します。

---

### シナリオ 6: DMVPN Hub での Per-Spoke 階層型 QoS (HQoS) 構築

#### 問題
DMVPN Hub ルーター R7 において、低速拠点 Spoke1 (契約帯域 2Mbps) 向けの送信トラフィックがプロバイダで一括ドロップされるのを防ぐため、Spoke1 宛て通信を 2Mbps にシェーピングする HQoS を設定してください。

#### 要件
1. Spoke1 のトンネル IP (172.16.1.101) 宛て通信を識別する Class-Map `CM-SPOKE1` を作成すること。
2. 子ポリシー `PM-SPOKE-CHILD` で DSCP `ef` に 256Kbps の LLQ を構成すること。
3. 親ポリシー `PM-DMVPN-HUB-HQOS` 内で `CM-SPOKE1` に対し 2Mbps のシェーピングと子ポリシーを適用すること。
4. WAN 物理インターフェイス `Gi0/0/0` に適用すること。

#### 設定例

```text
! 1. Spoke1 識別用 Access-List & Class-Map
ip access-list extended ACL-SPOKE1-TRAFFIC
 permit ip any host 172.16.1.101

class-map match-all CM-SPOKE1
 match access-group name ACL-SPOKE1-TRAFFIC

class-map match-any CM-VOICE
 match dscp ef

! 2. 子ポリシー
policy-map PM-SPOKE-CHILD
 class CM-VOICE
  priority 256
 class class-default
  fair-queue

! 3. 親ポリシー
policy-map PM-DMVPN-HUB-HQOS
 class CM-SPOKE1
  shape average 2000000
  service-policy PM-SPOKE-CHILD
 class class-default
  fair-queue

! 4. 物理インターフェイスへ適用
interface GigabitEthernet0/0/0
 service-policy output PM-DMVPN-HUB-HQOS
```

#### 検証方法
`show policy-map interface GigabitEthernet0/0/0` を実行し、Spoke1 宛ての通信のみが 2Mbps にシェーピングされ、かつ音声パケットが Priority Queue で正しく処理されているか確認します。

---

### シナリオ 7: Cisco ASA MPF QoS (Priority Queuing & Policing)

#### 問題
Cisco ASA5516 Firewall において、音声通信の優先キュー処理および特定セグメントからの通信に対する帯域制限（Policing）を構成してください。

#### 要件
1. UDP 16384-32767 通信を `CM-ASA-VOICE` として識別し、優先キュー（`priority`）で処理すること。
2. 10.100.0.0/24 セグメントの通信を `CM-ASA-LIMIT` として識別し、最大 5Mbps (`police 5000000`) に制限すること。
3. `outside` インターフェイス上で Priority Queue を物理的に有効化すること。

#### 設定例

```text
! 1. Access-List 定義
access-list ACL-ASA-VOICE extended permit udp any any range 16384 32767
access-list ACL-ASA-LIMIT extended permit ip 10.100.0.0 255.255.255.0 any

! 2. Class-Map 定義
class-map CM-ASA-VOICE
 match access-list ACL-ASA-VOICE
class-map CM-ASA-LIMIT
 match access-list ACL-ASA-LIMIT

! 3. Policy-Map 定義
policy-map PM-ASA-OUTSIDE-QOS
 class CM-ASA-VOICE
  priority
 class CM-ASA-LIMIT
  police output 5000000 93750

! 4. インターフェイス適用 & Priority Queue 有効化
service-policy PM-ASA-OUTSIDE-QOS interface outside
priority-queue outside
```

#### 検証方法
ASA にて `show service-policy interface outside` を実行し、`CM-ASA-VOICE` で priority 処理数が増加し、`CM-ASA-LIMIT` でポリサーによるドロップ/送信数が記録されていることを確認します。さらに `show priority-queue statistics outside` でインプレースキューの健全性を確認します。

---

### シナリオ 8: NBAR2 による P2P アプリケーションの帯域規制

#### 問題
ルーター R8 において、NBAR2（Network-Based Application Recognition）を利用して P2P（BitTorrent）および動画ストリーミング通信を動的に検出し、帯域を遮断または 64Kbps に強力ポリシングしてください。

#### 要件
1. NBAR2 で `bittorrent` アプリケーションを識別する Class-Map `CM-P2P` を作成すること。
2. `CM-P2P` トラフィックに対し、帯域上限 64Kbps のポリシングを設定し、超過パケットを即座にドロップすること。
3. `Gi0/0/1` の Ingress に適用すること。

#### 設定例

```text
! 1. NBAR2 Class-Map 定義
class-map match-any CM-P2P
 match protocol bittorrent

! 2. Policy-Map 定義
policy-map PM-NBAR-RESTRICT
 class CM-P2P
  police 64000 2000 conform-action transmit exceed-action drop

! 3. インターフェイス設定
interface GigabitEthernet0/0/1
 ip nbar protocol-discovery
 service-policy input PM-NBAR-RESTRICT
```

#### 検証方法
`show ip nbar protocol-discovery interface GigabitEthernet0/0/1` でアプリ識別統計を確認し、`show policy-map interface GigabitEthernet0/0/1 input` にて BitTorrent 通信の超過分がドロップされていることを確認します。

---

### シナリオ 9: DSCP ベースの WRED (Weighted Random Early Detection) による輻輳回避

#### 問題
ルーター R9 の出力ポートにおいて、ネットワーク輻輳時に低優先度パケットを段階的に事前破棄し、TCP グローバル同期を防ぐ WRED を設定してください。

#### 要件
1. クラス `class-default` に対し、DSCP ベースの WRED (`random-detect dscp-prec-based`) を有効化すること。
2. DSCP `af11`（優先度高）および DSCP `af13`（優先度低）の破棄閾値を設定し、`af13` が先にドロップ開始されるよう設定すること。

#### 設定例

```text
! 1. Policy-Map 定義
policy-map PM-WRED-ENG
 class class-default
  bandwidth percent 70
  random-detect dscp-prec-based
  ! dscp af11: min-threshold 32, max-threshold 40, mark-probability 10
  random-detect dscp 10 32 40 10
  ! dscp af13: min-threshold 20, max-threshold 40, mark-probability 10 (早期ドロップ)
  random-detect dscp 14 20 40 10

! 2. インターフェイス適用
interface GigabitEthernet0/0/2
 service-policy output PM-WRED-ENG
```

#### 検証方法
`show policy-map interface GigabitEthernet0/0/2` を実行し、`WRED explicit drop counts` セクションで DSCP 14 (AF13) のパケットが DSCP 10 (AF11) よりも低いバッファ閾値でドロップ開始されていることを確認します。

---

### シナリオ 10: IPsec Anti-Replay ウィンドウ調整による QoS 優先パケット保護

#### 問題
IPsec トンネルで LLQ（Priority Queue）を導入した際、優先音声パケットがデータパケットを追い越して到着することで発生する Anti-Replay 誤ドロップを防ぐ設定を行ってください。

#### 要件
1. IPsec SA の Anti-Replay Window サイズを標準の 64 から 1024 パケットへ拡大すること。
2. 対象 Crypto Map または IPsec Profile に適用すること。

#### 設定例

```text
! 1. IPsec Anti-Replay Window のグローバル/プロファイル設定
crypto ipsec security-association replay window-size 1024

! または FlexVPN / VTI 用 IPsec Profile での設定
crypto ipsec profile PROF-FLEXVPN-QOS
 set transform-set TS-AES256-SHA256
 set ipsec-proposal PROP-AES256
 ! プロファイル単位のウィンドウサイズ適用
 set replay window-size 1024
```

#### 検証方法
`show crypto ipsec sa` を実行し、`replay window size: 1024` と表示されていること、および `anti-replay error` のカウントが増加していないことを確認します。



---

## ❓ 想定試験問題

CCIE Security v6.1 ラボ・デザイン試験を想定したシナリオ問題と詳細な解説です。

### 問題 1 (コンフィグ読解・トポロジ推論)
以下の Cisco IOS-XE コンフィグレーションを分析し、問いに答えてください。

```text
class-map match-all CM-APP
 match access-group name ACL-APP

policy-map PM-CHILD
 class CM-APP
  priority percent 20

policy-map PM-PARENT
 class class-default
  shape average 5000000
  service-policy PM-CHILD

interface Tunnel0
 ip address 10.255.1.1 255.255.255.0
 tunnel source GigabitEthernet0/0/0
 tunnel mode gre multipoint
 service-policy output PM-PARENT
```

**質問**:
1. この設定において、暗号化/GREトンネル経由で送信される通信に対して `CM-APP` クラスの優先キュー処理（`priority`）が正常に機能しない技術的理由を説明してください。
2. この問題を修正するために追加すべき必須コマンドとその設定箇所を答えてください。

**解答・解説**:
1. **理由**: `Tunnel0` インターフェイスに `qos pre-classify` コマンドが設定されていません。GRE または IPsec カプセル化が行われると、元の IP ヘッダーおよび L4 ポート情報（`ACL-APP` で指定された情報）がカプセル化ヘッダーで隠蔽されます。Egress の QoS ポリシー評価時点でパケットはすでにカプセル化されているため、アウターヘッダーに対して `ACL-APP` のマッチングが失敗し、すべて `class-default` として処理されてしまいます。
2. **修正コマンド**: `Tunnel0` インターフェイス配下に `qos pre-classify` を追加設定します。これにより、パケットのカプセル化前に内部情報に基づくQoS分類タグがアタッチされ、トンネル出力時に `CM-APP` へ正しくマッチングされます。

---

### 問題 2 (トラブルシューティング)
DMVPN スポーク拠点から本社ハブへの音声通話（VoIP）において、ネットワーク帯域が十分に空いているにもかかわらず、高頻度で音声パケットが破棄され、音切れが発生しています。管理者が出力コマンドを実行したところ、以下の結果が得られました。

```text
R1-Spoke# show policy-map interface GigabitEthernet0/0/0
 GigabitEthernet0/0/0 
  Service-policy output: PM-WAN-OUT

    Class-map: CM-VOICE (match-any)
      124500 packets, 24900000 bytes
      5 minute offered rate 320000 bps, drop rate 85000 bps
      Match: dscp ef (46)
      Priority: 256 kbps, burst bytes 6400, Explicit Drop: 4250000 bytes
```

**質問**:
1. `CM-VOICE` クラスでパケットドロップが発生している直接的な原因は何ですか。
2. 音声品質を維持しつつ、他通信を保護するための適切な対処方法を説明してください。

**解答・解説**:
1. **原因**: `CM-VOICE` クラスの優先帯域幅制限（`priority 256`）に対し、実際の流入トラフィック（320Kbps）が制限値を超加しているためです。LLQ の Priority Queue は規定帯域（256Kbps）を超過したパケットを警察作用（Policed drop）により強制破棄します。
2. **対処方法**:
   - `priority` 帯域幅を実際の最大音声トラフィック量（例: 384Kbps または 512Kbps）へ引き上げます。
   - 同時に、音声コーデック（G.711 から G.729 への変更）や過剰な音声セッションの抑制（CAC: Call Admission Control）を検討し、Priority Queue への過剰流入を防止します。

---

### 問題 3 (Design / アーキテクチャ)
本社（1Gbps 回線）と 100 箇所の拠点（各 2Mbps 回線）を DMVPN で接続するメッシュネットワークを設計します。ハブ側から各拠点への送信時、プロバイダ網でのパケット一括破棄（Tail Drop）を防ぎつつ、拠点ごとの音声通信（DSCP EF）を優先制御するための最適な QoS アーキテクチャを提案してください。

**解答・解説**:
- **提案アーキテクチャ**: DMVPN Hub における Per-Spoke 階層型 QoS (HQoS) および Per-Tunnel QoS 設定。
- **構成要素**:
  1. **親ポリシー (Outer)**: 各スポークの契約帯域（2Mbps）に合わせて `shape average 2000000` を設定。これにより、ハブからのバースト送信が 2Mbps に平滑化され、プロバイダ網でのドロップを回避。
  2. **子ポリシー (Inner)**: 親ポリシー内にネストし、DSCP EF（音声）に対して `priority 512` (LLQ)、管理通信に対して `bandwidth` (CBWFQ) を割り当てる。
  3. **QoS Pre-classify**: トンネルインターフェイスに `qos pre-classify` を適用し、暗号化前の IP/L4 情報に基づいて子ポリシーでの優先振分けを実行。



### 問題 4 (Cisco ASA MPF QoS 実装)
Cisco ASA において、Management 通信用の優先キューイングを設定しましたが、通信を発生させても優先処理が行われていないことが判明しました。設定されたコンフィグは以下の通りです。

```text
access-list ACL-MGMT extended permit tcp any any eq 22

class-map CM-ASA-MGMT
 match access-list ACL-MGMT

policy-map PM-ASA
 class CM-ASA-MGMT
  priority

service-policy PM-ASA interface mgmt
```

**質問**:
この ASA において優先キューイングを正常に動作させるために欠けているコマンドを指摘してください。

**解答・解説**:
- **欠けているコマンド**: グローバルコンフィグレーションモードでの `priority-queue mgmt` です。
- **理由**: Cisco ASA では、MPF（Policy-Map）内で `priority` を定義するだけでなく、対象の物理/論理インターフェイス上で明示的に `priority-queue <interface-name>` コマンドを実行してハードウェア/ソフトウェアのインプレース優先キューをアクティベートする必要があります。

---

### 問題 5 (Troubleshooting / Anti-Replay と QoS の相互作用)
IPsec FlexVPN トンネル上で LLQ（Priority Queue）を導入した直後から、受信側ルーターの Syslog に `%CRYPTO-4-RECVD_PKT_NOT_IN_SEQUENCE` エラーが大量に出力され、一部のデータパケットが破棄される現象が発生しました。原因と解決策を述べてください。

**解答・解説**:
- **原因**: LLQ により音声などの優先パケットが遅延中のデータパケットを追い越して送信されたため、受信側に届く IPsec パケットのシーケンス番号の順序が大きく逆転（Packet Reordering）しました。この逆転幅が IPsec 標準の Anti-Replay Window サイズ（64パケット）を超えたため、受信側で攻撃と判定されて破棄されました。
- **解決策**: IPsec SA の Anti-Replay ウィンドウサイズを拡大します。`crypto ipsec security-association replay window-size 1024` コマンドを実行してウィンドウ幅を 1024 パケットに広げることで、順序逆転による誤判定ドロップを防止します。

---

## 🔗 参考リソース

Cisco 公式の QoS Configuration Guide、Design Guide の参照情報です。

- Cisco IOS-XE Quality of Service Solutions Configuration Guide
- Cisco Quality of Service Design Overview (QoS Design Guide)
- Class-Based Weighted Fair Queueing (CBWFQ) Implementation Guide
- Configuring QoS Pre-Classify on IPsec VPN Tunnels
- Cisco ASA Series CLI Configuration Guide - Quality of Service (QoS)
- Cisco Secure Firewall Management Center QoS Policy Configuration Guide
- RFC 4594: Configuration Guidelines for DiffServ Service Classes
- RFC 2697: A Single Rate Three Color Marker (srTCM)
- RFC 2698: A Two Rate Three Color Marker (trTCM)

---

## 📝 補足（Notes）

### 図解: QoS 処理フローとバケット（Token Bucket）モデル

```text
[ Token Bucket (CIR) ]
   Tokens added at rate CIR
   │
   ▼
[ Packet Ingress ] ──▶ [ Check Token Bucket ]
                             │
            ┌────────────────┴────────────────┐
            ▼                                 ▼
    Tokens Available?                 No Tokens Available?
            │                                 │
            ▼                                 ▼
   [ Conform Action ]                [ Exceed Action ]
  (Transmit / Mark DSCP)            (Drop / Remark DSCP)
```

### 注意点メモ
- **Priority Percent と Priority KBPS の違い**: `priority percent 20` はインターフェイスの割り当て可能帯域に対する割合、`priority 512` は絶対帯域幅（Kbps）です。親ポリシーでシェーピング（`shape`）を行っている場合、`percent` はシェーピング設定後の帯域に対して計算されます。
- **ASA と IOS-XE の MQC 構文の違い**: ASA ではポリシーマップ内のキューイングオプションが `priority`（帯域値なし）のみとなり、詳細な `bandwidth` 指定や WRED はサポートされません。用途を明確に区別して解答する必要があります。



