---
layout: default
title: 3.4.f-RA-Guard
nav_order: 6
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.4.f RA Guard

## 📘 概要

IPv6 Router Advertisement Guard（RA Guard）は、Layer 2 アクセスネットワークにおいて不正なルータから送信される IPv6 Router Advertisement（RA）パケットおよび ICMPv6 Router Redirect パケットを識別し、スイッチポートレベルで破棄・ブロックするファーストホップセキュリティ（IPv6 First Hop Security: FHS）技術である（RFC 6105 に準拠）。

IPv6 ネットワークにおいて、ホストは Stateless Address Autoconfiguration（SLAAC）を利用してデフォルトゲートウェイ情報や IPv6 プレフィックス、DDR（Default Router Preferences）等の網構成情報を RA パケットを通じて動的に取得する。この際、悪意のある攻撃者や不適切に接続された顧客用ルータ・PC（テザリング機能や仮想化ハイパーバイザのルーティング機能が有効化された端末）が野良ルータ（Rogue IPv6 Router）として動作し、偽の RA パケットを送出すると、ネットワーク上の正常なホストのデフォルトゲートウェイが書き換わり、中間者攻撃（Man-in-the-Middle: MitM）やサービス拒否（Denial of Service: DoS）が発生する危険性がある。

RA Guard は、スイッチポートの役割（Device Role: Host または Router）や、RA パケットに含まれるポリシー条件（プレフィックス、プレファレンス、Hop Limit、M/O フラグ等）に基づいてリアルタイムでインスペクションを実施し、未認証のポートから送信された有害な RA / ICMPv6 Redirect パケットをインラインでブロックすることで、IPv6 インフラのデータプレーンおよび制御プレーンの健全性を保護する。

---

## 🔑 要点

RA Guard の主要な特徴、用途、利点、制限事項、および設計上の考慮事項を以下の表にまとめる。

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | スイッチポートに入力される ICMPv6 RA（Type 134）および Redirect（Type 137）を検証し、未承認ポートからの送出をハードウェア（TCAM）またはソフトウェアレベルでドロップする。 |
| **用途** | エンタープライズ Access スイッチ、キャンパス LAN、データセンター Access 層、Wi-Fi アクセスポイント接続ポートにおける Rogue IPv6 Router 防御。 |
| **メリット** | SLAAC 環境における MitM / DoS 攻撃の完全遮断、Cisco FHS / SISF 統合による統合管理、ポリシーステートレス／ステートフル検証の柔軟な運用。 |
| **デメリット** | 誤設定（アップリンクポートへの Host ロール適用等）による正常な IPv6 通信の遮断リスク、IPv6 フラグメンテーション攻撃による初期ハードウェアTCAMバイパス脆弱性への考慮が必要。 |
| **対応機種** | Catalyst 9000 シリーズ、Catalyst 3650/3850、Catalyst 4500E/6800、Nexus スイッチ（Nexus 9000等）、Cisco IOS / IOS-XE 搭載スイッチ。 |
| **制限事項** | トランクポートや EtherChannel でのポリシー適用制限、VLAN 単位およびポート単位の適用競合、ハードウェア TCAM リソースの上限。 |
| **設計上の注意点** | 正常なコア/ディストリビューションルータが接続されるポートには `device-role router` を設定し、エンドホスト接続ポートには `device-role host` を厳格に適用すること。 |

---

## 🏗 動作原理

RA Guard は、L2 スイッチングエンジンに入力されたすべての ICMPv6 パケットをスヌーピングし、ICMPv6 Type 134（Router Advertisement）および ICMPv6 Type 137（Redirect）に対して定義された FHS ポリシーをマッチングさせる。

### 通信フロー図（正常パケットと Rogue RA パケットの比較）

```text
[ Authorized IPv6 Router ] (Fa0/1: Device Role = Router)
          │
          │ ICMPv6 RA (Type 134) - Allowed
          ▼
    ┌───────────┐
    │ L2 Switch │ ───( Forward RA )───► [ Legitimate Host ] (Fa0/3)
    └───────────┘
          ▲
          │ ICMPv6 RA (Type 134) - BLOCKED & DROPPED
          │
[ Rogue IPv6 Host / Router ] (Fa0/2: Device Role = Host)
```

1. **アクセスポートからの RA 送信検出**: Fa0/2（`device-role host`）に接続された端末が ICMPv6 RA パケットを送出。
2. **ポリシールール照合**: スイッチは Fa0/2 に適用された `ipv6 nd raguard policy` を評価し、Device Role が `host` であることを確認。
3. **パケット破棄とログ記録**: パケットを即座に破棄（Drop）し、必要に応じて Syslog 警告および FHS ドロップカウンタをインクリメント。
4. **正規ルータからの通信維持**: Fa0/1（`device-role router`）からの RA パケットは照合を通過し、VLAN 内の端末（Fa0/3）へ正常に転送される。

---

## ⚙ 動作シーケンス

RA Guard がパケットを受信してから転送または破棄を決定する内部処理シーケンスは以下の通りである。

```text
[ ICMPv6 Packet Ingress ]
           │
           ▼
[ ICMPv6 Type Check ] ──( Not Type 134/137 )──► [ Normal L2/L3 Forwarding ]
           │
     ( Type 134 / 137 )
           │
           ▼
[ Check RA Guard Policy Attached to Port/VLAN ]
           │
           ├─► [ Match Device-Role ]
           │        ├─► ( Role = Router ) ──► [ Pass to Validation Checks ]
           │        └─► ( Role = Host )   ──► [ DROP PACKET & Log ]
           │
           ▼
[ Access List (ACL) & Prefix-List Check ]
           │
           ├─► ( Source IPv6 / Prefix Match Deny ) ──► [ DROP PACKET ]
           └─► ( Match Permit ) ─────────────────────► [ Pass to Option Checks ]
           │
           ▼
[ Managed / Other Config Flag & Priority Checks ]
           │
           ├─► ( M/O Flag, Hop-Limit, Preference Discrepancy ) ──► [ DROP PACKET ]
           └─► ( All Checks Passed ) ─────────────────────────────► [ FORWARD PACKET ]
```

1. **パケットインプレスと ICMPv6 タイプ判定**: インターフェイスに入力された Ethernet フレームの EtherType（`0x86DD`）を識別し、IPv6 ヘッダー内の Next Header が ICMPv6（`58`）であり、ICMPv6 Type が 134（RA）または 137（Redirect）であるかを判断。
2. **ポート/VLAN ポリシーの参照**: 受信ポートまたは所属 VLAN にバインドされた `ipv6 nd raguard policy` を検索。
3. **デバイスロール（Device Role）の検証**:
   - `device-role host`: 即座にパケットを破棄（Rogue RA の判定）。
   - `device-role router`: 後続の属性検証処理へ移行。
4. **IPv6 Access-List および Prefix-List による検証**: 送信元 Link-Local アドレス（`FE80::/10`）や RA に含まれる Prefix Option が許可リストと合致するかを照合。
5. **RA 属性パラメータ検証（ステートフル/高度検証）**:
   - **Managed Address Config Flag (M-Flag)** / **Other Config Flag (O-Flag)** の検証（`on` / `off` の不一致チェック）。
   - **Router Preference**: `maximum` または `minimum` の範囲内であるか。
   - **Hop Limit**: 設定された最小/最大 Hop Limit 値を充足しているか。
6. **最終転送制御**: すべての検証をパスしたパケットのみが同一 VLAN 内の他ポートへフラッディング/転送される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security ラボ試験において、RA Guard は IPv6 Security および First Hop Security（FHS）の必須要件として頻出する。単なるコマンド入力だけでなく、制約条件に応じたポリシー設計とトラブルシューティングが要求される。

### 1. Blueprintで重要なポイント
- **ステートレス（Stateless）とステートフル（Stateful）RA Guard の違い**:
  - ステートレス: 単純に `device-role host` または `device-role router` に基づいて RA をドロップ/許可。
  - ステートフル: 受信した RA パケット内の Prefix、Preference、Managed/Other Flag、Hop Limit をポリシーと照合。
- **SISF (Switch Integrated Security Features) 構文への移行**:
  - レガシー IOS 構文: `ipv6 nd raguard policy <NAME>`
  - 最新 IOS-XE / SISF 構文: `device-tracking policy <NAME>` または `ipv6 nd raguard policy <NAME>` の適用方法。
- **VLAN 単位 vs ポート単位の適用優先度**:
  - ポート単位に直接アタッチされたポリシー（`ipv6 nd raguard attach-policy`）は、VLAN 単位のポリシーよりも優先して評価される。

### 2. ラボ試験で設定させられそうなシナリオ・要件例
- **基本要件**: 「アクセスポート Fa0/1 - 10 において、ユーザー端末からの不正な IPv6 RA 送信をブロックせよ。ただし正規ルータからの RA は拒否してはならない。」
- **高度属性フィルタリング**: 「RA Guard を設定し、Prefix `2001:DB8:1:10::/64` 以外のプレフィックスを広報する RA パケット、および Default Router Preference が `High` に設定された RA パケットをドロップせよ。」
- **ACL 連動**: 「IPv6 アクセスリスト `RA_AUTHOR_ACL` で許可された送信元 Link-Local アドレスからの RA のみを受信許可せよ。」

### 3. よくある設定ミスと試験で狙われやすい落とし穴
- **アップリンクポートへの `device-role host` 設定**: コアスイッチや上流ルータが接続された Trunk/Routed ポートに誤って `device-role host` ポリシーを適用し、VLAN 全体の IPv6 デフォルトゲートウェイ自動学習が不全となるトラブル。
- **IPv6 ACL の `permit` / `deny` ロジックの混同**: RA Guard ポリシー内で指定する IPv6 ACL は、**`permit` が「RA を許可する対象（正規ルータ）」**であり、`deny` は拒否対象となる。標準のパケットフィルタ ACL と混同しやすい。
- **IPv6 Fragmentation 攻撃（Extension Header）によるバイパス**: 攻撃者が IPv6 Extension Header（Fragment Header）を使用して ICMPv6 Type 134 を分断して送出する場合、古い IOS ハードウェアで検知を回避される問題。対策として `ipv6 snooping capture policy` や PACL を組み合わせる要件が出題される場合がある。

### 4. コンフィグ読解・show コマンドからの状態判断
- `show ipv6 nd raguard policy <NAME>`: ポリシー内のパラメータ設定（Device Role、ACL、Prefix-List、Hop Limit 等）を確認。
- `show ipv6 snooping counters`: RA Guard によってドロップされたパケット数およびドロップ理由をカウンターで確認。


## 🛠 設定方法

Cisco IOS / IOS-XE における RA Guard の設定手順と設定例を示す。

### 1. 基本設定手順（Stateless RA Guard）

ホスト接続ポートからの RA をすべてブロックする最も典型的な設定例。

```text
! Step 1: RA Guard ポリシーの定義
ipv6 nd raguard policy RAGUARD_HOST_POLICY
 device-role host
!
! Step 2: インターフェイスへのポリシー適用
interface GigabitEthernet1/0/1
 description Access Port connected to IPv6 Host
 switchport mode access
 switchport access vlan 10
 ipv6 nd raguard attach-policy RAGUARD_HOST_POLICY
```

正規ルータが接続されるポートに対する設定例。

```text
! 正規ルータ接続ポート用ポリシー
ipv6 nd raguard policy RAGUARD_ROUTER_POLICY
 device-role router
!
interface GigabitEthernet1/0/24
 description Trunk/Uplink Port to IPv6 Core Router
 switchport mode trunk
 ipv6 nd raguard attach-policy RAGUARD_ROUTER_POLICY
```

---

### 2. 高度な設定例（Stateful / Attribute-based RA Guard）

送信元 IPv6 アドレス制限、Prefix-List、Router Preference、Hop-Limit 条件を組み込んだ高度なポリシー設定。

```text
! 送信元ルータの Link-Local アドレス許可 ACL
ipv6 access-list PERMIT_GW_LL
 permit ipv6 host FE80::21A:4BFF:FE12:3456 any
!
! 広報を許可する IPv6 プレフィックス
ipv6 prefix-list PERMIT_RA_PREFIX seq 5 permit 2001:DB8:10:20::/64
!
! 高度な RA Guard ポリシー定義
ipv6 nd raguard policy ADVANCED_RAGUARD
 device-role router
 access-list PERMIT_GW_LL
 prefix-list PERMIT_RA_PREFIX
 managed-address-flag off
 other-address-flag off
 router-preference maximum medium
 hop-limit minimum 64
!
! VLAN またはポートへの適用
interface GigabitEthernet1/0/10
 switchport mode access
 switchport access vlan 20
 ipv6 nd raguard attach-policy ADVANCED_RAGUARD
```

---

## 🔍 検証コマンド

RA Guard の動作状態、ポリシー適用状態、ドロップカウンターを確認するためのコマンドを以下の表にまとめる。

| 目的 | コマンド |
| :--- | :--- |
| ポリシー詳細の確認 | <code>show ipv6 nd raguard policy [policy-name]</code> |
| インターフェイス適用状態確認 | <code>show ipv6 nd raguard interface [interface-id]</code> |
| FHS ドロップカウンターの確認 | <code>show ipv6 snooping counters [interface-id]</code> |
| SISF / Device Tracking 状態確認 | <code>show device-tracking database</code> |
| トラブルシューティング用デバッグ | <code>debug ipv6 snooping raguard</code> |
| FHS イベントデバッグ | <code>debug ipv6 nd</code> |

---

## 🚨 トラブルシュート

RA Guard 導入時に発生しやすい障害現象、原因、確認コマンド、および対処方法を以下の表にまとめる。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| ホストが IPv6 デフォルトゲートウェイを取得できない | 正規ルータ接続ポートに `device-role host` または誤ったポリシーが適用されている。 | <code>show ipv6 nd raguard interface</code> | 上流ルータポートのポリシーを `device-role router` に変更するか、ポリシー適用を削除する。 |
| RA Guard 設定にもかかわらず Rogue RA が通過する | インターフェイスにポリシーがアタッチされていないか、VLAN ポリシーとポートポリシーが矛盾している。 | <code>show ipv6 nd raguard policy</code> | `ipv6 nd raguard attach-policy` の設定状態を確認し、適切にポートへアタッチする。 |
| 正規ルータからの RA がドロップされる | ACL または Prefix-List の記述ミス、または Router Preference / Hop Limit 条件の不一致。 | <code>show ipv6 snooping counters</code> | ドロップカウンターの理由（Prefix Mismatch, ACL Deny 等）を確認し、ポリシー条件を修正する。 |
| IPv6 フラグメント化された Rogue RA が透過する | ハードウェア TCAM が Fragmented ICMPv6 パケットを正常にインスペクションできていない。 | <code>debug ipv6 snooping raguard</code> | PACL や `ipv6 snooping capture policy` を併用し、フラグメントパケットを CPU 処理へリダイレクトしてインスペクションする。 |

---

## ⚠ 制限事項

1. **トランクポートおよび EtherChannel での動作制限**:
   - EtherChannel 構成時、ポリシーは物理メンバーポートではなく Port-Channel インターフェイスに適用する必要がある。
2. **ハードウェア TCAM リソースの制限**:
   - 大量のバインドポリシーや複雑な IPv6 ACL を定義した場合、スイッチの Ternary Content Addressable Memory（TCAM）の上限を超過し、ソフトウェアフォールバック（CPU 処理）またはポリシー適用失敗が発生する。
3. **IPv6 Extension Header による回避脆弱性**:
   - 複数の IPv6 拡張ヘッダー（Destination Options、Routing Header、Fragment Header）が重なったパケットにおいて、ハードウェア解析深度を超過して RA Guard を回避されるリスクが存在する（最新 IOS-XE では修正済み）。

---

## 🔄 他技術との関連

RA Guard と他のセキュリティ・ネットワーク技術との統合・関連性を整理する。

- **DHCPv6 Guard**: DHCPv6 Reply / Rebind パケットによる不正な DHCPv6 サーバー（Rogue DHCPv6）の遮断技術。RA Guard と併用することで SLAAC と Stateful DHCPv6 の双方のインフラ保護を実現。
- **IPv6 Snooping (FHS Binding Table)**: IPv6 の IP-MAC-Port バインディングデータベースを動的に構築する基盤。RA Guard、IPv6 Source Guard、IPv6 DAI（ND Inspection）と共通のデータベースを参照する。
- **SeND (Secure Neighbor Discovery - RFC 3971)**: Cryptographically Generated Addresses（CGA）と RSA 署名を用いて ND パケットを認証する高度な技術。RA Guard は証明書インフラを必要としない L2 スイッチレベルの軽量防御を提供する。
- **Private VLANs (PVLAN)**: Isolated ポートからの RA 送信を二重で遮断可能。

---

## 🧩 比較表

IPv6 ファーストホップセキュリティ技術および類似機能の比較表。

| 比較項目 | IPv6 RA Guard | IPv6 ND Inspection (IPv6 DAI) | DHCPv6 Guard | IPv6 Source Guard |
| :--- | :--- | :--- | :--- | :--- |
| **対象プロトコル/パケット** | ICMPv6 RA (Type 134) / Redirect (Type 137) | ICMPv6 NS (135) / NA (136) | DHCPv6 Reply / Advertise | IPv6 Data Packets (IP / Non-ICMP) |
| **主な攻撃対象** | Rogue Router / SLAAC 偽装 | DAD 妨害 / Neighbor Spoofing | Rogue DHCPv6 Server | IP Address Spoofing |
| **評価基準** | Device Role, Prefix, ACL, Hop Limit | Binding Table (IP-MAC-Port) | Device Role, Server Preference | Binding Table (IP-MAC-Port) |
| **適用位置** | L2 Access Ports / VLAN | L2 Access Ports / VLAN | L2 Access Ports / VLAN | L2 Access Ports |

---

## 💡 ベストプラクティス

1. **Access ポートへの例外なき `device-role host` 適用**: エンドポイント（PC、IP Phone、プリンタ、カメラ等）が接続されるすべての L2 アクセスポートにデフォルトで `device-role host` の RA Guard ポリシーを適用する。
2. **アップリンクポートの明示的共有ポリシー運用**: 上流のコア/ディストリビューションスイッチ、またはルータが接続されるポートには、送信元 Link-Local アドレス ACL および標準 Prefix-List を組み込んだ `device-role router` ポリシーをバインドする。
3. **新規ポート設定の標準化（Template 化）**: ポート共有コンフィグテンプレートに `ipv6 nd raguard attach-policy` を組み込み、設定漏れを防止する。


## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験レベルを意識した実践的な設定シナリオを 10 個作成する。

---

### シナリオ 1: 基本的な Stateless RA Guard の実装

#### 問題
スイッチ SW1 のアクセスポート Gi1/0/1 - Gi1/0/10 に接続されたユーザー端末からの不正な IPv6 Router Advertisement パケットをブロックせよ。

#### 要件
- ポリシー名 `RAGUARD_HOST_ONLY` を作成すること。
- デバイスロールを `host` に設定すること。
- 対象インターフェイス（Gi1/0/1 - 10）にポリシーをバインドすること。

#### 設定例
```text
!
ipv6 nd raguard policy RAGUARD_HOST_ONLY
 device-role host
!
interface range GigabitEthernet1/0/1 - 10
 switchport mode access
 switchport access vlan 10
 ipv6 nd raguard attach-policy RAGUARD_HOST_ONLY
!
```

#### 検証方法
1. Gi1/0/1 に接続された端末から scapy または 擬似ルータで ICMPv6 RA（Type 134）を送出する。
2. SW1 にてポリシー適用状態を確認する:
   `show ipv6 nd raguard policy RAGUARD_HOST_ONLY`
   `show ipv6 nd raguard interface GigabitEthernet1/0/1`
3. ドロップカウンターを確認する:
   `show ipv6 snooping counters GigabitEthernet1/0/1`

---

### シナリオ 2: 正規ルータ接続ポートへの Router Role RA Guard 適用

#### 問題
スイッチ SW1 の Gi1/0/24 は正規ルータ R1 に接続されている。R1 からの RA 通信のみを許可し、他の属性変更を抑止せよ。

#### 要件
- ポリシー名 `RAGUARD_ROUTER_PORT` を作成すること。
- デバイスロールを `router` に設定すること。
- Gi1/0/24 にポリアシをバインドすること。

#### 設定例
```text
!
ipv6 nd raguard policy RAGUARD_ROUTER_PORT
 device-role router
!
interface GigabitEthernet1/0/24
 switchport mode trunk
 ipv6 nd raguard attach-policy RAGUARD_ROUTER_PORT
!
```

#### 検証方法
1. R1 から定期的に送信される RA パケットが SW1 の Gi1/0/24 を通過し、VLAN 10 内の端末に到達することを確認。
2. `show ipv6 nd raguard interface GigabitEthernet1/0/24` で Device Role が `router` になっていることを確認。

---

### シナリオ 3: 送信元 IPv6 制限 ACL を伴う RA Guard

#### 問題
ルータ R1（Link-Local: `FE80::1111:2222`）以外の送信元 IPv6 アドレスから送出される RA パケットを全ポートで拒否せよ。

#### 要件
- IPv6 ACL `ACL_PERMIT_R1_LL` を作成し、`FE80::1111:2222` からの通信を permit とすること。
- RA Guard ポリシー `RAGUARD_ACL_FILTER` にこの ACL をバインドすること。
- VLAN 10 全体にポリシーを適用すること。

#### 設定例
```text
!
ipv6 access-list ACL_PERMIT_R1_LL
 permit ipv6 host FE80::1111:2222 any
!
ipv6 nd raguard policy RAGUARD_ACL_FILTER
 device-role router
 access-list ACL_PERMIT_R1_LL
!
vlan configuration 10
 ipv6 nd raguard attach-policy RAGUARD_ACL_FILTER
!
```

#### 検証方法
1. 送信元 `FE80::9999:9999` からの RA パケットを送信し、ドロップされることを確認。
2. `show ipv6 snooping counters` で ACL ドロップが発生していることを確認。

---

### シナリオ 4: 広報 Prefix 制限を行う Prefix-List 連携 RA Guard

#### 問題
VLAN 20 内において、正規プレフィックス `2001:DB8:ABCD:20::/64` 以外の IPv6 プレフィックスを広報する RA をブロックせよ。

#### 要件
- IPv6 Prefix-List `PERMIT_V20_PREFIX` を作成すること。
- RA Guard ポリシー `RAGUARD_PREFIX_CHECK` 内で Prefix-List を指定すること。
- Gi1/0/24 ポートに適用すること。

#### 設定例
```text
!
ipv6 prefix-list PERMIT_V20_PREFIX seq 5 permit 2001:DB8:ABCD:20::/64
!
ipv6 nd raguard policy RAGUARD_PREFIX_CHECK
 device-role router
 prefix-list PERMIT_V20_PREFIX
!
interface GigabitEthernet1/0/24
 ipv6 nd raguard attach-policy RAGUARD_PREFIX_CHECK
!
```

#### 検証方法
1. ルータ R1 から `2001:DB8:FFFF:99::/64` を含める偽 RA を送信し、スイッチでドロップされることを確認。
2. `show ipv6 snooping counters GigabitEthernet1/0/24` で Prefix 不一致ドロップを確認。

---

### シナリオ 5: Managed/Other Address Config Flag (M/O-Flag) 検証

#### 問題
ネットワーク管理者の方針に従い、SLAAC（M=0, O=0）のみを許可し、Stateful DHCPv6（M=1）または DHCPv6 他情報（O=1）を広報する RA パケットをドロップせよ。

#### 要件
- ポリシー名 `RAGUARD_SLAAC_ONLY` を作成すること。
- Managed Address Flag を `off` に指定すること。
- Other Address Flag を `off` に指定すること。

#### 設定例
```text
!
ipv6 nd raguard policy RAGUARD_SLAAC_ONLY
 device-role router
 managed-address-flag off
 other-address-flag off
!
interface GigabitEthernet1/0/24
 ipv6 nd raguard attach-policy RAGUARD_SLAAC_ONLY
!
```

#### 検証方法
1. M フラグ（`1`）がセットされた RA パケットを送信し、SW1 で遮断されることを確認。
2. `show ipv6 nd raguard policy RAGUARD_SLAAC_ONLY` で Managed/Other Flag が `off` と表示されることを確認。


### シナリオ 6: Router Preference および Hop-Limit の高度な制御

#### 問題
Rogue ルータが Default Router Preference を `High` に設定して優先ゲートウェイになろうとする攻撃、および Hop-Limit を `32` など不適切な値に変更する攻撃を防御せよ。

#### 要件
- Router Preference の上限を `medium` に制限すること（`High` の RA は拒否）。
- Minimum Hop-Limit を `64` に制限すること（`64` 未満の RA は拒否）。
- ポリシー名 `RAGUARD_LIMIT_PARAM` とすること。

#### 設定例
```text
!
ipv6 nd raguard policy RAGUARD_LIMIT_PARAM
 device-role router
 router-preference maximum medium
 hop-limit minimum 64
!
interface GigabitEthernet1/0/24
 ipv6 nd raguard attach-policy RAGUARD_LIMIT_PARAM
!
```

#### 検証方法
1. Router Preference = `High` の RA パケットを送出し、SW1 にて拒否されることを確認。
2. Hop-Limit = `32` の RA パケットを送出し、同様にドロップされることを確認。
3. `show ipv6 snooping counters` で属性違反ドロップを確認。

---

### シナリオ 7: VLAN Configuration レベルでの一括 RA Guard 適用

#### 問題
VLAN 100 および VLAN 200 に属する全アクセスポートに対し、一括で Host ロールの RA Guard ポリシーを適用せよ。ただし個別ポートにアタッチされたポリシーが存在する場合は、個別ポートのポリシーを優先させよ。

#### 要件
- ポリシー名 `RAGUARD_VLAN_HOST` を作成。
- VLAN 100 および 200 にポリシーをアタッチすること。

#### 設定例
```text
!
ipv6 nd raguard policy RAGUARD_VLAN_HOST
 device-role host
!
vlan configuration 100,200
 ipv6 nd raguard attach-policy RAGUARD_VLAN_HOST
!
```

#### 検証方法
1. `show ipv6 nd raguard policy RAGUARD_VLAN_HOST` を実行し、VLAN 100, 200 にバインドされていることを確認。
2. VLAN 100 のアクセスポート Gi1/0/5 から送信された RA がドロップされることを確認。

---

### シナリオ 8: EtherChannel（LACP）環境における RA Guard 設定

#### 問題
ルータ接続用の EtherChannel（Port-channel 1: Gi1/0/21, Gi1/0/22）において、RA Guard を正しく適用せよ。

#### 要件
- ポリシー名 `RAGUARD_PO1_POLICY` を作成し、`device-role router` に指定。
- 物理メンバーポートではなく、Port-channel 1 インターフェイスに適用すること。

#### 設定例
```text
!
ipv6 nd raguard policy RAGUARD_PO1_POLICY
 device-role router
!
interface Port-channel1
 switchport mode trunk
 ipv6 nd raguard attach-policy RAGUARD_PO1_POLICY
!
interface GigabitEthernet1/0/21
 switchport mode trunk
 channel-group 1 mode active
!
interface GigabitEthernet1/0/22
 switchport mode trunk
 channel-group 1 mode active
!
```

#### 検証方法
1. Port-channel 1 経由で受信した RA パケットが正しく検証され転送されることを確認。
2. `show ipv6 nd raguard interface Port-channel1` でアタッチ状態を確認。

---

### シナリオ 9: IPv6 Snooping と RA Guard の統合運用

#### 問題
スイッチ全体で IPv6 Snooping を有効化し、FHS バインディングデータベースを正常に機能させた上で、アクセスポートからの RA 送信をブロックせよ。

#### 要件
- IPv6 Snooping ポリシー `FHS_SNOOPING_POLICY` を作成。
- アクセスポート Gi1/0/2 に Snooping ポリシーおよび RA Guard ポリシーの双方をアタッチせよ。

#### 設定例
```text
!
ipv6 snooping policy FHS_SNOOPING_POLICY
!
ipv6 nd raguard policy RAGUARD_ACCESS
 device-role host
!
interface GigabitEthernet1/0/2
 switchport mode access
 switchport access vlan 10
 ipv6 snooping attach-policy FHS_SNOOPING_POLICY
 ipv6 nd raguard attach-policy RAGUARD_ACCESS
!
```

#### 検証方法
1. `show device-tracking database` で端末の IPv6 バインディング情報が正しく記録されているか確認。
2. 対象ポートから RA を送信し、Snooping データベースに影響を与えることなく RA が即座にドロップされることを確認。

---

### シナリオ 10: RA Guard のトラブルシューティングとデバッグ解析

#### 問題
Gi1/0/3 からの正規 RA パケットがドロップされる問題が発生している。原因をログおよびデバッグコマンドで特定し復旧させよ。

#### 要件
- FHS カウンターおよび Debug コマンドを使用してドロップ要因（Prefix mismatch, ACL, Role mismatch 等）を診断すること。

#### 設定例（復旧コンフィグ）
```text
! トラブルシューティング手順
! 1. ドロップカウンターの確認
show ipv6 snooping counters GigabitEthernet1/0/3

! 2. デバッグの有効化
debug ipv6 snooping raguard

! 3. 原因が Prefix-List の指定不一致であった場合の復旧修正
no ipv6 prefix-list PERMIT_RA_PREFIX
ipv6 prefix-list PERMIT_RA_PREFIX seq 5 permit 2001:DB8:100:10::/64
!
```

#### 検証方法
1. `debug ipv6 snooping raguard` の出力で `RA Guard: Packet dropped due to Prefix Mismatch` ログが解消されたことを確認。
2. ホスト端末で IPv6 アドレスが正常に生成されることを確認。


## ❓ 想定試験問題

CCIE Security ラボ試験・筆記試験を想定した実践的な試験問題（5問）。

### 問題 1（コンフィグ読解・トラブルシューティング）
**【問題】**
以下のコンフィグが設定された Catalyst スイッチ SW1 のアクセスポート Gi1/0/5 に接続された PC が、IPv6 の自動アドレス生成（SLAAC）を実行できず、デフォルトゲートウェイを検出できない現象が発生している。スイッチの共有コンフィグおよびログ出力を分析し、原因と解決策を述べよ。

```text
! コンフィグ抜粋
ipv6 access-list ALLOW_RA_GW
 permit ipv6 host FE80::200:5EFF:FE00:1111 any
!
ipv6 nd raguard policy RAGUARD_POLICY_A
 device-role host
 access-list ALLOW_RA_GW
!
interface GigabitEthernet1/0/5
 switchport mode access
 switchport access vlan 10
 ipv6 nd raguard attach-policy RAGUARD_POLICY_A
!
```

**【解説・解答】**
- **原因**: `RAGUARD_POLICY_A` にて `device-role host` が設定されている。Device Role が `host` の場合、ポリシー内の ACL や Prefix-List 条件にかかわらず、ポートに入力される**すべての ICMPv6 RA（Type 134）および Redirect（Type 137）パケットが即座にドロップされる**ためである。
- **解決策**: Gi1/0/5 がルータ接続ポートである場合は `device-role router` に変更する。ユーザーホスト接続ポートである場合は上流ルータポート Gi1/0/24 等にポリシーを適用し直す。

---

### 問題 2（デザイン・仕様理解）
**【問題】**
IPv6 RA Guard と SeND（Secure Neighbor Discovery - RFC 3971）の比較に関する以下の記述のうち、正しいものを 2 つ選べ。

A. RA Guard は公開鍵暗号（RSA）および CGA（Cryptographically Generated Addresses）を使用して ICMPv6 メッセージ自体を電子署名する。
B. RA Guard は L2 スイッチポートレベルで動作し、証明書インフラ（PKI）やホスト側の特別なプロトコル実装を必要としない。
C. SeND は L2 スイッチの TCAM リソースのみを使用して RA パケットをハードウェアアクセラレーションでドロップする。
D. RA Guard は ICMPv6 Type 134（RA）および Type 137（Redirect）のポリシー制御を行うのに対し、SeND はホスト・ルータ間の暗号論的相互認証を提供する。

**【解説・解答】**
- **正解**: B, D
- **解説**: A および C は誤り。RSA 署名および CGA を使用してメッセージ保護を行うのは SeND であり、RA Guard はスイッチ側の FHS ポリシーによるフィルタリング技術である。RA Guard は PKI を不要とし、ホスト OS に依存しない利点を持つ。

---

### 問題 3（実装・属性フィルター）
**【問題】**
ネットワーク要件「RA パケット内の Managed Address Config Flag (M) が 1（Stateful DHCPv6）にセットされている RA を拒否し、SLAAC 通信のみを許可する」を満たす正しい RA Guard ポリシーコンフィグを選べ。

A.
```text
ipv6 nd raguard policy SLAAC_ONLY
 device-role router
 managed-address-flag on
```

B.
```text
ipv6 nd raguard policy SLAAC_ONLY
 device-role router
 managed-address-flag off
```

C.
```text
ipv6 nd raguard policy SLAAC_ONLY
 device-role host
 dhcp-snooping
```

D.
```text
ipv6 nd raguard policy SLAAC_ONLY
 device-role router
 dhcp-managed-override
```

**【解説・解答】**
- **正解**: B
- **解説**: `managed-address-flag off` を設定することで、M フラグが `1`（`on`）に設定された RA パケットをドロップし、M フラグが `0`（`off`）のパケットのみを許可することができる。

---

### 問題 4（トラブルシューティング・高度）
**【問題】**
スイッチ SW1 にて RA Guard を設定したが、未認証の送信元から IPv6 Extension Header（Fragment Header）を重畳させて分割送信された RA パケットが RA Guard を不当に通過してホストに到達してしまった。この事象に対処するための最善のスイッチ設定対策はどれか。

A. インターフェイスで `no ipv6 routing` を設定する。
B. `ipv6 snooping capture policy` を設定し、フラグメント化された IPv6 パケットを CPU にリダイレクトして完全検査させる。
C. RA Guard ポリシー内で `device-role host` を解除する。
D. `switchport port-security` を有効化する。

**【解説・解答】**
- **正解**: B
- **解説**: 古い ASIC や初期のハードウェア TCAM は、IPv6 Extension Header（特に Fragment Header）が挿入された ICMPv6 パケットの L4/ICMP タイプを正しく識別できず通過してしまう事象（RFC 7113）が存在する。`ipv6 snooping capture policy` を使用してパケットを CPU で再構築・検証させることで、この回避事象を防止できる。

---

### 問題 5（コンフィグ実装）
**【問題】**
GigabitEthernet1/0/10 インターフェイスに対し、以下の条件を満たす RA Guard ポリシーを作成・適用せよ。
1. ポリシー名: `RAGUARD_SECURE`
2. デバイスロール: `router`
3. 許可するプレフィックス: `2001:DB8:500::/64`（Prefix-List `PFL_SECURE` を使用）
4. Hop Limit の最小値: `64`

**【解答コンフィグ】**
```text
ipv6 prefix-list PFL_SECURE seq 5 permit 2001:DB8:500::/64
!
ipv6 nd raguard policy RAGUARD_SECURE
 device-role router
 prefix-list PFL_SECURE
 hop-limit minimum 64
!
interface GigabitEthernet1/0/10
 ipv6 nd raguard attach-policy RAGUARD_SECURE
!
```

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび Cisco Live 資料の参照リンク一覧。

- Cisco Catalyst 9300 Series Switches - IPv6 First Hop Security Configuration Guide
- Cisco Technical Support: IPv6 RA Guard Configuration Example
- IETF RFC 6105: IPv6 Router Advertisement Guard
- IETF RFC 7113: Implementation Advice for IPv6 Router Advertisement Guard (RA-Guard)
- Cisco Live On-Demand Library: IPv6 Security Threats and Mitigation Technologies

---

## 📝 補足（Notes）

### 補足 1: RFC 6105 と RFC 7113 (RA Guard 回避対策)
初期の RA Guard 実装では、IPv6 Extension Header（Destination Options、Fragment Header 等）を重ね合わせることで、ASIC の固定深度インスペクションによる検知を逃れ、Rogue RA パケットがスイッチを通過してしまう現象が報告された（RFC 7113）。
最新の Cisco Catalyst 9000 シリーズおよび Cisco IOS-XE（16.x / 17.x）では、フレキシブルインスペクションエンジンおよび UDF（User Defined Field）TCAM パターンの最適化により、拡張ヘッダーが重畳されたパケットに対しても高精度なデコードが行われ、安全にドロップされる。

### 補足 2: SISF (Switch Integrated Security Features) による統合
Cisco IOS-XE では、IPv6 FHS（RA Guard、DHCPv6 Guard、IPv6 Source Guard 等）と IPv4 FHS（DHCP Snooping、DAI、IPSG）が SISF アーキテクチャとして共通のバインディングデータベースおよびポリシープラットフォーム（`device-tracking policy`）へ統合されつつある。ラボ試験においては、従来の `ipv6 nd raguard policy` と SISF 構文の双方の表示・設定方法を理解しておくことが推奨される。

