---
layout: default
title: 1.2.d-PBR
nav_order: 4
parent: 1.2-Firewall-features-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.2.d Policy-based routing

Cisco ASA (Adaptive Security Appliance) および Cisco FTD (Firepower Threat Defense) における **Policy-Based Routing (PBR: ポリシーベースルーティング)** は、従来のデスティネーション IP アドレスのみに基づく標準のルーティングテーブル (FIB: Forwarding Information Base) をバイパスし、パケットの送信元 IP アドレス、宛先 IP アドレス、L4 プロトコル、宛先/送信元ポート番号、さらには L7 アプリケーション識別（FTD FMC 7.1+）などの拡張条件に基づいて動的にネクストホップや送信インターフェイスを変更する技術です。

ASA では OS 9.4(1) で導入され、CLI の `route-map` および `policy-route` コマンドにより実装されます。FTD では FMC 7.1 以降で GUI によるネイティブ機能としてサポートされ、それ以前のバージョンでは FlexConfig を用いて LINA データプレーンに展開します。デュアル ISP 環境でのロードシェアリング、特定トラフィックの Direct Internet Access (DIA) / クラウドブレイクアウト、SLA トラッキングに基づく動的経路切り替え、および QoS DSCP マーキングなどで幅広く利用されます。

---

## 📘 概要

* **機能概要**:
  * パケットがファイアウォールのインgress（入力）インターフェイスに到着した際、拡張 Access Control List (ACL) やアプリケーションオブジェクトとマッチングを行い、一致したトラフィックに対して指定されたネクストホップ IP アドレス、インターフェイス、または IP ヘッダー属性（DSCP/DF ビット）を適用して転送します。
  * **ASA**: CLI にて `access-list`、`route-map`（`match ip address` / `set ip next-hop` 等）、および インターフェイス配下の `policy-route route-map` を使用して構成します。
  * **FTD (FMC 7.1+)**: GUI の `Devices > Device Management > Routing > Policy Based Routing` 画面にて、Ingress インターフェイス、Match ACL（ネットワーク/ポート/アプリケーション）、および Forwarding Actions（Egress インターフェイス優先度または Next-Hop IP）を一括定義します。
* **利用目的**:
  * 送信元ネットワーク（例: 部門別・VLAN 別）に応じた特定 ISP へのトラフィックの優先振分け（Equal-Access / Source-Sensitive Routing）。
  * トラフィック種別（Web 閲覧、ストレージバックアップ、VoIP 等）に応じた低遅延リンク/広帯域リンクの使い分け。
  * 拠点間 IPsec VPN トンネル経由のバックホール通信から、特定の SaaS アプリケーション（Microsoft 365, YouTube 等）のみをローカルの WAN インターフェイスから直接インターネットへ抜く Direct Internet Access (DIA) の実現。
  * IP ヘッダー内の Precedence / DSCP フィールドの書き換えや DF (Don't Fragment) ビットの制御による帯域制御連携。
* **どのような場面で利用するか**:
  * 複数 ISP 接続を持つマルチホーム環境（Internet Edge）。
  * 拠点 FTD から本社 VPN 経由とインターネット直接接続を組み合わせる SD-WAN / 支社構成（Branch Office Deployment）。
  * クラスター構成における複数物理アプライアンス間でのトラフィック静的分散。

---

## 🔑 要点

| 項目 | Cisco ASA (CLI / ASDM) | Cisco FTD (FMC 7.1+ Native) | Cisco FTD (Pre-7.1 FlexConfig) |
| :--- | :--- | :--- | :--- |
| **導入バージョン** | ASA OS 9.4(1) 以降 | FMC / FTD Version 7.1.0 以降 | FTD 6.x 〜 7.0.x |
| **設定インターフェース** | CLI (`route-map`) / ASDM GUI | FMC GUI (`Routing > Policy Based Routing`) | FMC FlexConfig Policy (ASA CLI 展開) |
| **サポートモード** | Routed Firewall Mode 専用 | Routed Firewall Mode 専用 | Routed Firewall Mode 専用 |
| **識別条件 (Match Criteria)** | 拡張 ACL (L3/L4 Src/Dst IP, Port) | 拡張 ACL (L3/L4 ＋ **L7 アプリケーション / NSG**) | 拡張 ACL (L3/L4 Src/Dst IP, Port) |
| **適用場所** | **Ingress インターフェイス専用** (入力パケットのみ) | **Ingress インターフェイス専用** (入力パケットのみ) | **Ingress インターフェイス専用** (入力パケットのみ) |
| **SLA トラッキング** | サポート (`set ip next-hop verify-availability`) | サポート (SLA Monitor オブジェクト) | サポート (ASA CLI オブジェクトトラック) |
| **アクション優先度** | `set` アクションの厳格な評価順序に従う | インターフェイス優先度 (Priority) または Order | `set` アクションの厳格な評価順序に従う |
| **uRPF (Reverse Path) 連携** | **uRPF 評価は PBR を無視** (ドロップに注意) | **uRPF 評価は PBR を無視** (ドロップに注意) | **uRPF 評価は PBR を無視** (ドロップに注意) |
| **IPv6 サポート** | 非サポート（IPv4 のみ） | 非サポート（IPv4 のみ） | 非サポート（IPv4 のみ） |
| **クラスタリング制限** | サポート | **アプリベース PBR はクラスタ非サポート** | サポート |

---

## 🏗 動作原理

PBR はファイアウォールのパケット処理パイプラインにおいて、従来のルーティングテーブル検索（FIB Lookup）手順を上書き（Override）します。

```text
[ Ingress Packet Arrives (e.g., Inside Interface) ]
                        │
                        ▼
┌────────────────────────────────────────────────────────┐
│ LINA Fastpath / Packet Flow Engine                     │
│                                                        │
│ 1. Existing Session Lookup (Conn Table Match?)          │
│    ├── YES ──► Route based on saved connection flow    │
│    └── NO  ──► New Connection Flow Processing          │
│                                                        │
│ 2. Access Control List (ACL / ACP Check)               │
│    └── Allowed? ──► Proceed to Routing Stage           │
│                                                        │
│ 3. Policy-Based Routing (PBR) Evaluation               │
│    ├── Ingress Interface configured with PBR?          │
│    │     ├── YES ──► Evaluate Route-Map / PBR Policy   │
│    │     └── NO  ──► Standard Route Lookup (FIB)       │
│    │                                                   │
│    ├── Matches PBR Match ACL / Rules?                  │
│    │     ├── Match Permit ──► Apply PBR Action (Set)   │
│    │     │                   (Egress Ifc / Next-Hop)   │
│    │     └── Match Deny / Fallthrough                  │
│    │                         └── Standard Route Lookup │
└────────────────────────────────────────────────────────┘
                        │
                        ▼
[ Forward Packet to Selected Egress Interface & Next-Hop ]
```

### PBR 評価ルールとルートマップの論理マトリクス

ルートマップにおける `permit` / `deny` 句と、Match ACL 内の `permit` / `deny` エントリの組み合せによる動作結果は以下の通りです。

| Route-Map アクション | ACL エントリ | PBR 処理結果 | 理由・動作 |
| :--- | :--- | :--- | :--- |
| **Permit** | **Permit** | **PBR 適用 (Success)** | トラフィックがマッチし、`set` アクション（ネクストホップ書き換え等）が実行される。 |
| **Permit** | **Deny** | **次の Route-Map 句へスキップ** | 当該句のマッチ失敗。次のシーケンス番号の `route-map` 句の評価へ進む。 |
| **Deny** | **Permit** | **PBR 終了 ➔ 通常ルーティング** | PBR によるポリシールーティングを明示的に拒否。通常の FIB ルーティングを実行。 |
| **Deny** | **Deny** | **次の Route-Map 句へスキップ** | 当該句のマッチ失敗。次のシーケンス番号の `route-map` 句の評価へ進む。 |
| **(末尾到達)** | **(No Match)** | **通常ルーティング** | どの `route-map` 句にもマッチしなかった場合、通常の FIB ルーティングテーブルを使用。 |

---

## ⚙ 動作シーケンス

パケットが ASA / FTD の Ingress インターフェイスに到着してから PBR が適用されるまでの詳細シーケンスは以下の通りです。

1. **新規セッション（First Packet）の受信**:
   * 着信パケットに対し、既存のコネクションテーブル（Conn Table）の検索が行われます。
   * 既存接続が存在する場合は、そのフローに登録された出力インターフェイスから即座に転送されます（PBR は評価されません）。
2. **L3/L4 アクセス制御 (ACL / ACP) 判定**:
   * セキュリティポリシー（ASA ACL または FTD Access Control Policy）が評価されます。許可されたトラフィックのみが PBR 処理へ進みます。
3. **PBR ポリシーの呼び出し**:
   * 入力インターフェイスにバインドされた PBR ルートマップが呼び出されます。
4. **Match 条件の Top-Down 評価**:
   * ルートマップのシーケンス番号順（例: seq 10, seq 20）に、`match ip address <ACL>` が評価されます。
   * **FTD (FMC 7.1+)** の場合、ファーストパケットのアプリケーション分類（First-Packet Application Classification / Network Service Groups: NSG）が行われ、指定された L7 アプリケーション（例: Office365, YouTube）が照合されます。
5. **Set アクションの評価と順序決定**:
   * マッチした `permit` 句内に複数の `set` アクションが設定されている場合、以下の**厳格な評価優先順位**に従って最初に利用可能な設定が適用されます。
     1. `set ip next-hop verify-availability` (SLA トラックで Reachable と判定されたネクストホップ)
     2. `set ip next-hop` (直接接続されたネクストホップ IP)
     3. `set ip next-hop recursive` (ルーティングテーブルで再帰検索可能なネクストホップ IP)
     4. `set interface` (指定された出力インターフェイス)
     5. `set ip default next-hop` (通常の FIB ルックアップが失敗した場合のみ使用するデフォルトネクストホップ)
     6. `set default interface` (通常の FIB ルックアップが失敗した場合のみ使用するデフォルト出力ポート)
6. **セッションテーブルへのフロー保存**:
   * PBR によって決定された出力インターフェイスおよびネクストホップ情報が接続フロー（xlate / conn）に書き込まれます。同一セッションの後続パケットは PBR を再評価することなく高速転送されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **`set` アクションの優先順位とフォールバックの動作順序**:
   * ASA / FTD の PBR では、1 つの `route-map` 句の中に複数の `set` コマンドが記述されている場合、ロードバランシングは行われず、**あらかじめ定義された優先順位に従って上から順番に判定** されます。
   * 例えば `set ip next-hop 1.1.1.1` と `set ip default next-hop 2.2.2.2` が同時に存在する場合、通常のルーティングテーブルで宛先ルートが存在すれば `1.1.1.1` が使われ、宛先ルートが存在しない（デフォルトルートのみ等）場合に初めて `2.2.2.2` が評価されます。
2. **SLA トラッキング (`verify-availability`) の必須構成**:
   * 問題文で「ISP 障害時には自動的にバックアップ ISP または標準ルーティングへ切り替えさせよ」と指示された場合、単なる `set ip next-hop` ではなく、必ず **`set ip next-hop verify-availability <IP> <seq> track <id>`** と **`sla monitor` / `track` オブジェクト** を組み合わせて設定しなければなりません。
   * SLA トラックなしで `set ip next-hop` のみを設定すると、対向ゲートウェイがダウンして ARP が引けなくなっても、PBR はパケットをドロップし続け、通常のルーティングに切り替わりません。
3. **Unicast Reverse Path Forwarding (uRPF) との致命的な競合**:
   * インターフェイス上で **uRPF (`ip verify reverse-path`)** が有効化されている場合、uRPF はパケットの送信元 IP アドレスの妥当性を **標準のルーティングテーブル（FIB）** を参照して検証します（PBR のルートマップは無視されます）。
   * 送信元 IP に対する逆引きルートが着信ポートに存在しない場合、PBR 設定が正しくても **uRPF によってパケットが即座にドロップ** されます。試験で「PBR を設定したが通信が疎通しない」というトラブルシューティングが出題された場合、対象ポートの `ip verify reverse-path` を確認することが鉄則です。
4. **PBR が適用されるトラフィックの範囲制限**:
   * **Inbound Only**: PBR は Ingress インターフェイスから入ってくる着信パケット（Through-the-box traffic）にのみ適用されます。
   * **To-the-box / Local Traffic 不可**: ファイアウォール自身から発生するトラフィック（例: Syslog, RADIUS/TACACS+, SNMP, FMC 管理通信）には PBR は適用されません。
   * **既存接続には非適用**: 接続テーブルが既に確立している場合、PBR 設定を追加しても既存セッションの経路は変わりません（検証時は `clear conn` が必須）。
5. **FTD (FMC 7.1+) における Direct Internet Access (DIA) 設定**:
   * FMC 7.1 以降の GUI では、Match ACL に `Network Service` または `Application` を指定することで、特定の SaaS アプリケーション（例: Microsoft 365, Salesforce）を識別してローカルの WAN インターフェイスから直接ブレイクアウトさせる PBR ルールを直感的に作成できます。
   * **注意**: クラスター構成（FTD Cluster）では、アプリケーションベースの PBR ポリシーはサポートされません。

---

## 🛠 設定方法

### 1. Cisco ASA: CLI による PBR 構成 (SLA Tracking ＆ ロードシェアリング)

```bash
! ===== 1. ICMP SLA Monitor と Track オブジェクトの構成 =====
sla monitor 100
 type echo server-monitoring ipaddr 192.168.6.6 interface outside-1
 frequency 5
sla monitor schedule 100 life forever start-time now

track 1 rsl policy-list
 sla 100 reachability

sla monitor 200
 type echo server-monitoring ipaddr 172.16.7.7 interface outside-2
 frequency 5
sla monitor schedule 200 life forever start-time now

track 2 rsl policy-list
 sla 200 reachability

! ===== 2. トラフィック分類用 Extended ACL の構成 =====
! 10.1.0.0/16 (HR Dept) -> ISP-A (192.168.6.6)
access-list ACL_PBR_HR extended permit ip 10.1.0.0 255.255.0.0 any

! 10.2.0.0/16 (Eng Dept) -> ISP-B (172.16.7.7)
access-list ACL_PBR_ENG extended permit ip 10.2.0.0 255.255.0.0 any

! ===== 3. Route-Map の定義 =====
route-map PBR_POLICY permit 10
 match ip address ACL_PBR_HR
 set ip next-hop verify-availability 192.168.6.6 1 track 1
 set ip next-hop verify-availability 172.16.7.7 2 track 2

route-map PBR_POLICY permit 20
 match ip address ACL_PBR_ENG
 set ip next-hop verify-availability 172.16.7.7 1 track 2
 set ip next-hop verify-availability 192.168.6.6 2 track 1

! ===== 4. Ingress インターフェイスへの PBR 適用 =====
interface GigabitEthernet0/0
 nameif inside
 security-level 100
 ip address 10.0.0.1 255.255.0.0
 policy-route route-map PBR_POLICY
```

### 2. Cisco FTD (FMC 7.1+ GUI): ネイティブ PBR ポリシーおよび アプリケーションブレイクアウト構成

1. **Extended Access Control List (Match ACL) オブジェクトの作成**:
   * **Objects > Object Management > Access List > Extended** へ移動し **Add Extended Access List** をクリック。
   * **Name**: `ACL_DIA_APP_BREAKOUT`。
   * **Add Entry**: Action: `Allow`, **Application** タブを選択。
   * Available Applications から `Office 365`, `YouTube`, `WebEx` を選択して **Add to Rule** ➔ **Save**。
2. **Policy Based Routing ポリシーの構成**:
   * **Devices > Device Management** で対象 FTD デバイスを編集 ➔ **Routing** タブ ➔ **Policy Based Routing** を選択。
   * **Add** をクリック。
   * **Ingress Interface**: `Inside1_Zone` (または `Inside1` ポート) を選択。
   * **Forwarding Actions**:
     * **Match ACL**: `ACL_DIA_APP_BREAKOUT` を選択。
     * **Send To**: `Egress Interfaces` を選択。
     * **Interface Ordering**: `By Priority` を選択。
     * **Selected Egress Interfaces**: `WAN1` (Priority: 10), `WAN2` (Priority: 20) を選択して追加 ➔ **Save**。
3. **Deploy の実行**:
   * FMC 上で保存後、対象 FTD デバイスへ Deploy を適用。

### 3. Cisco FTD (Pre-7.1): FlexConfig による PBR 展開

```text
! FMC GUI: Objects > FlexConfig > FlexConfig Objects > Add FlexConfig Object
! Name: FlexConfig_PBR_ASA
! Deployment: Every Time, Type: Append

access-list ACL_PBR_FLEX extended permit ip 10.50.0.0 255.255.0.0 any
route-map MAP_PBR_FLEX permit 10
 match ip address ACL_PBR_FLEX
 set ip next-hop 192.168.100.254
interface GigabitEthernet0/1
 policy-route route-map MAP_PBR_FLEX

! FMC GUI: Devices > FlexConfig > 対象 FTD に割り当てて Deploy 実行。
```

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **PBR 適用インターフェイスとマップ名の一覧表示** | ASA / FTD CLI | <code>show policy-route</code> |
| **定義済み Route-Map エントリおよび Match/Set 条件確認** | ASA / FTD CLI | <code>show route-map</code> |
| **SLA モニタートラッキングオブジェクトの状態確認** | ASA / FTD CLI | <code>show track</code> / <code>show sla monitor operational-state</code> |
| **リアルタイム PBR 処理ログのデバッグ** | ASA CLI | <code>debug policy-route</code> |
| **パケットフロー追跡 (PBR ルックアップの擬似検証)** | ASA / FTD CLI | <code>packet-tracer input inside tcp 10.1.1.100 12345 8.8.8.8 80 detail</code> |
| **FTD (FMC) PBR の LINA 展開状態確認** | FTD CLI | <code>system support diagnostic-cli</code> ➔ <code>show policy-route</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **PBR を設定しても特定のパケットがネクストホップへ曲がらない** | uRPF (`ip verify reverse-path`) によるドロップ | ASA: <code>show running-config interface</code> | 着信ポートの `ip verify reverse-path` を無効化するか、送信元 IP に対する適正ルートを FIB に追加する。 |
| **SLA トラックで Primary GW ダウン時に Backup GW に切り替わらない** | `set ip next-hop` に `verify-availability` が未指定 | ASA: <code>show route-map</code> | `set ip next-hop verify-availability <IP> <seq> track <ID>` オプション形式に変更する。 |
| **新規設定した PBR が既存端末の通信に即座に反映されない** | コネクションテーブルに旧フローがキャッシュされている | ASA/FTD: <code>show conn detail</code> | `clear conn address <IP>` を実行して既存セッションをクリアする。 |
| **FTD 7.1 へのバージョンアップ後、PBR が動作しなくなった** | FlexConfig PBR がアップグレード時に全削除された | FMC: `Devices > Routing > Policy Based Routing` | FMC 7.1+ のネイティブ PBR 画面で同等設定を再構築する。 |
| **ローカル内部セグメント宛ての通信まで外部 GW に曲がってしまう** | Match ACL で内部ネットワーク宛てを除外していない | ASA: <code>show access-list</code> | Match ACL の先頭に `deny ip <内部網> <内部網>` ルールを追加して PBR 評価から除外する。 |

---

## ⚠ 制限事項

* **非サポートアーキテクチャ・プラットフォーム**:
  * **Transparent Firewall Mode**: PBR は Routed Mode 専用であり、透過モードでは一切使用不可。
  * **IPv6 PBR**: IPv6 アドレスに対する PBR（`set ipv6 next-hop` 等）は ASA/FTD ともに非サポート。
  * **Firepower 1010 / ISA 3000**: FMC 7.1+ ネイティブ PBR のプラットフォーム制限事項（一部小型モデル非対応）。
* **設定上の制限事項**:
  * **Egress Route-Map 不可**: PBR は Ingress（着信）ポート専用であり、Egress ポートに割り当てることはできません。
  * **To-the-box パケット不可**: ファイアウォール自身から発信されるローカル管理通信には適用されません。
  * **Cluster 制限**: アプリケーションベースの PBR ポリシーは FTD クラスタ環境では非サポートです。

---

## 🔄 他技術との関連

* **1.2.a NAT**: PBR によるネクストホップ決定（Egress インターフェイス決定）の前に、UN-NAT 処理が行われます。PBR の Match ACL は **UN-NAT 後のアドレス** を基準に評価されます。
* **1.2.c Traffic Zones / ECMP**: PBR の `set interface` アクションで Traffic Zone や ECMP Zone を指定することで、指定ゾーン内の複数ポートへ動的に負荷分散させることが可能です。
* **SLA Monitor & Tracking**: PBR の可用性を担保するため、ICMP Echo ベースの `sla monitor` / `track` オブジェクトと密接に連携します。
* **QoS (DSCP / Precedence / DF-Bit)**: PBR の `set` アクションで IP DSCP や DF ビットを直接書き換えることで、上流ルータの QoS ポリシーと連携できます。

---

## 🧩 比較表

### 1. 通常ルーティング (FIB) vs Policy-Based Routing (PBR)

| 比較項目 | 通常ルーティング (FIB / Route Table) | Policy-Based Routing (PBR) |
| :--- | :--- | :--- |
| **判定基準** | **宛先 IP アドレスのみ (Destination IP)** | **Src IP, Dst IP, Protocol, Port, App (L3-L7)** |
| **優先度** | 低（PBR ポリシーがない場合のみ評価） | **高（FIB ルーティングテーブルを上書き）** |
| **適用場所** | デバイス全体 (Global RIB/FIB) | **Ingress インターフェイス単位** |
| **制御コマンド** | `route <ifc> <prefix> <mask> <gw>` | `policy-route route-map <NAME>` |
| **用途** | 基本的なパケット転送 | DIA / クラウドブレイクアウト, ソースベース ISP 選択 |

### 2. ASA PBR (CLI) vs FTD Native PBR (FMC 7.1+)

| 比較項目 | Cisco ASA PBR (CLI) | Cisco FTD Native PBR (FMC 7.1+) |
| :--- | :--- | :--- |
| **設定手段** | ASA CLI (`route-map`, `policy-route`) | FMC GUI (`Routing > Policy Based Routing`) |
| **識別オブジェクト** | 拡張 ACL (L3/L4 IP, Port) | 拡張 ACL ＋ **Network Service Groups / Apps** |
| **SLA トラッキング** | `verify-availability` ＋ `track` | SLA Monitor オブジェクト連携 |
| **設定の簡便性** | CLI の複雑な階層設定が必要 | GUI ウィザードでインgress/Match/Egressを一括定義 |

---

## 💡 ベストプラクティス

1. **Match ACL における内部宛トラフィックの事前拒否 (`deny`)**:
   * PBR Match ACL で `permit ip 10.1.0.0/16 any` のように宛先を `any` に指定すると、別の内部サブネット（例: `10.2.0.0/16`）宛の通信まで外部 ISP ゲートウェイへ曲げられて不達になります。
   * 必ず Match ACL の先頭に **`deny ip 10.1.0.0 255.255.0.0 10.0.0.0 255.0.0.0`**（内部宛通信の除外）を明記します。
2. **SLA モニタートラッキングの常時結合**:
   * ネクストホップを指定する際は、単一障害点（SPOF）化を防ぐため、必ず `sla monitor` による ICMP ポーリングと `track` オブジェクトを紐付けた `verify-availability` を設定します。
3. **uRPF との併用回避（または適切なバイパス）**:
   * PBR を運用する Ingress ポートでは、uRPF による意図しないパケットドロップを防ぐため、`ip verify reverse-path` を無効化するか、ルータ側で適切な非対称対策を講じます。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA: 内部部門別 (HR / Engineering) ソースベース Dual-ISP PBR 構成

#### 問題
ASA1 の `inside` インターフェイス（`10.0.0.1`）において、HR 部門（`10.1.0.0/16`）からのトラフィックを ISP-A ゲートウェイ（`192.168.6.6`）へ、Engineering 部門（`10.2.0.0/16`）からのトラフィックを ISP-B ゲートウェイ（`172.16.7.7`）へポリシールーティングせよ。内部ネットワーク相互間の通信（`10.0.0.0/8` 宛て）は通常のルーティングを行わせよ。

#### 要件
* ASA CLI を使用し、`route-map` および `policy-route` で構成すること。

#### 設定例

```bash
! 1. Match ACL の作成 (内部宛通信の deny を忘れないこと)
access-list ACL_PBR_HR extended deny ip 10.1.0.0 255.255.0.0 10.0.0.0 255.0.0.0
access-list ACL_PBR_HR extended permit ip 10.1.0.0 255.255.0.0 any

access-list ACL_PBR_ENG extended deny ip 10.2.0.0 255.255.0.0 10.0.0.0 255.0.0.0
access-list ACL_PBR_ENG extended permit ip 10.2.0.0 255.255.0.0 any

! 2. Route-Map の定義
route-map PBR_DEPT_POLICY permit 10
 match ip address ACL_PBR_HR
 set ip next-hop 192.168.6.6

route-map PBR_DEPT_POLICY permit 20
 match ip address ACL_PBR_ENG
 set ip next-hop 172.16.7.7

! 3. Ingress インターフェイスへの割り当て
interface GigabitEthernet0/0
 nameif inside
 security-level 100
 ip address 10.0.0.1 255.255.0.0
 policy-route route-map PBR_DEPT_POLICY
```

#### 検証方法
* `show policy-route` を実行し、`inside` に `PBR_DEPT_POLICY` がバインドされていることを確認。
* HR クライアント (`10.1.1.10`) から `ping 8.8.8.8` を送信し、`debug policy-route` で `next_hop = 192.168.6.6` と判定されることを確認。

---

### 例2: ASA: SLA モニタートラッキングと `verify-availability` を用いた自動フォールバック PBR

#### 問題
ASA1 において、内部サブネット `10.10.0.0/16` からの外部通信をプライマリ GW `192.168.1.254`（Track 10）へ曲げよ。プライマリ GW の ICMP 応答が途絶えた場合は、セカンダリ GW `172.16.1.254`（Track 20）へ自動迂回させよ。両方ともダウンした場合は通常のルーティングテーブルへフォールバックさせよ。

#### 要件
* `set ip next-hop verify-availability` を使用すること。

#### 設定例

```bash
! 1. SLA Monitor と Track オブジェクトの設定
sla monitor 10
 type echo server-monitoring ipaddr 192.168.1.254 interface outside1
 frequency 5
sla monitor schedule 10 life forever start-time now

track 10 rsl policy-list
 sla 10 reachability

sla monitor 20
 type echo server-monitoring ipaddr 172.16.1.254 interface outside2
 frequency 5
sla monitor schedule 20 life forever start-time now

track 20 rsl policy-list
 sla 20 reachability

! 2. ACL と Route-Map の作成
access-list ACL_PBR_SLA extended permit ip 10.10.0.0 255.255.0.0 any

route-map PBR_SLA_MAP permit 10
 match ip address ACL_PBR_SLA
 set ip next-hop verify-availability 192.168.1.254 1 track 10
 set ip next-hop verify-availability 172.16.1.254 2 track 20

! 3. インターフェイス適用
interface GigabitEthernet0/0
 nameif inside
 policy-route route-map PBR_SLA_MAP
```

#### 検証方法
* `show track` を実行し、`Track 10` が `State is Up` であることを確認。
* Primary GW の回線を切断し、`Track 10` が `Down` に遷移後、`debug policy-route` でネクストホップが `172.16.1.254` へ動的切替されることを検証。

---

### 例3: ASA: 再帰的ルックアップ (`set ip next-hop recursive`) を用いた間接ネクストホップ PBR

#### 問題
ASA1 において、特定 Web アクセストラフィック（TCP 80）を、直接接続されていないリモートの専用プロキシサーバ IP (`10.250.250.254`) へ転送せよ。ファイアウォールは FIB ルーティングテーブルを参照して当該 IP への最適なパスを再帰検索して転送しなければならない。

#### 要件
* `set ip next-hop recursive` を使用すること。

#### 設定例

```bash
access-list ACL_PBR_PROXY extended permit tcp 10.0.0.0 255.0.0.0 any eq 80

route-map PBR_RECURSIVE_MAP permit 10
 match ip address ACL_PBR_PROXY
 set ip next-hop recursive 10.250.250.254

interface GigabitEthernet0/0
 nameif inside
 policy-route route-map PBR_RECURSIVE_MAP
```

#### 検証方法
* `show route-map` を実行し、`set ip next-hop recursive 10.250.250.254` が反映されていることを確認。
* HTTP トラフィックを発生させ、PBR ログで FIB ルックアップ経由でネクストホップが解決されて送出されることを検証。

---

### 例4: ASA: デフォルトネクストホップ (`set ip default next-hop`) による既存ルート優先型 PBR

#### 問題
内部ホスト群からの通信において、通常のルーティングテーブルに明示的な宛先ルート（/24 や /16 等）が存在する場合はそのルートを最優先で使用させ、明示的ルートが存在せずデフォルトルートにフォールバックする通信のみ、特定のバックアップゲートウェイ `192.168.99.1` へ転送せよ。

#### 要件
* `set ip default next-hop` を使用すること。

#### 設定例

```bash
access-list ACL_PBR_DEFAULT_FALLBACK extended permit ip 10.0.0.0 255.0.0.0 any

route-map PBR_DEFAULT_MAP permit 10
 match ip address ACL_PBR_DEFAULT_FALLBACK
 set ip default next-hop 192.168.99.1

interface GigabitEthernet0/0
 nameif inside
 policy-route route-map PBR_DEFAULT_MAP
```

#### 検証方法
* FIB テーブルに登録されている内部宛先（例: `10.5.5.5`）宛通信は通常のインターフェイスから抜け、未登録のインターネット宛通信（`8.8.8.8`）のみが `192.168.99.1` へ送出されることを確認。

---

### 例5: ASA: ネクストホップ変更なしの QoS DSCP / DF ビット書き換え PBR

#### 問題
ASA1 の `inside` から入るビデオ会議トラフィック（UDP 16384-32767）に対し、ネクストホップアドレスを変更することなく、IP ヘッダーの DSCP 値を `EF` (Expedited Forwarding: 46) に変更し、DF (Don't Fragment) ビットを `1` (Set) に設定せよ。

#### 要件
* `set ip dscp` および `set ip df` を使用すること。

#### 設定例

```bash
access-list ACL_VOIP_VIDEO extended permit udp 10.0.0.0 255.0.0.0 any range 16384 32767

route-map PBR_QOS_MARKING permit 10
 match ip address ACL_VOIP_VIDEO
 set ip dscp ef
 set ip df 1

interface GigabitEthernet0/0
 nameif inside
 policy-route route-map PBR_QOS_MARKING
```

#### 検証方法
* 対象ビデオパケットを送信し、対向ルータのキャプチャで DSCP 値が `ef` (0xb8) かつ DF Bit が `1` に書き換わっていることを検証。

---

### 例6: FTD (FMC 7.1+): Extended ACL を用いた Direct Internet Access (DIA) ネイティブ PBR 構成

#### 問題
FMC 7.1 上の FTD1 において、内部ネットワーク `10.100.0.0/16` からの通信のうち、社内データセンター `172.16.0.0/12` 宛て以外のアウトバウンドトラフィックを、ローカルの WAN インターフェイス (`WAN_ISP1`) へ優先度 10 で PBR 転送せよ。

#### 要件
* FMC GUI の Policy Based Routing 画面より設定すること。

#### 設定例

```text
1. FMC: Objects > Object Management > Access List > Extended > Add Extended Access List:
   - Name: ACL_FTD_DIA_PBR
   - Entry 1: Action: Deny, Source: 10.100.0.0/16, Destination: 172.16.0.0/12
   - Entry 2: Action: Allow, Source: 10.100.0.0/16, Destination: Any -> Save.
2. Devices > Device Management > FTD1 編集 > Routing > Policy Based Routing > Add:
   - Ingress Interface: Inside_Zone
   - Forwarding Actions:
     - Match ACL: ACL_FTD_DIA_PBR
     - Send To: Egress Interfaces
     - Interface Ordering: By Priority
     - Selected Egress Interfaces: WAN_ISP1 (Priority: 10) -> Save.
3. FMC から Deploy 実行。
```

#### 検証方法
* FTD CLI (Diagnostic CLI) で `show policy-route` を実行し、`Inside_Zone` ポートに PBR ルートマップが自動生成・展開されていることを確認。

---

### 例7: FTD (FMC 7.1+): SaaS アプリケーション (YouTube/WebEx) のブレイクアウト PBR 構成

#### 問題
FMC 7.1 を使用して FTD1 の `Branch_Inside` ポート上で Application-Aware PBR を構成せよ。クライアントからの `YouTube` および `WebEx` アプリケーション通信を自動検出（First-Packet Classification）し、ダイレクトインターネットブレイクアウト用インターフェイス `WAN_DIRECT` へ転送せよ。

#### 要件
* Application 条件を含む Extended ACL オブジェクトを作成すること。

#### 設定例

```text
1. FMC: Objects > Object Management > Access List > Extended > Add Extended Access List:
   - Name: ACL_PBR_SAAS_BREAKOUT
   - Entry 1: Action: Allow, Source: Any, Destination: Any
     - Applications タブ: `YouTube`, `WebEx` を選択して Add -> Save.
2. Devices > Device Management > FTD1 > Routing > Policy Based Routing > Add:
   - Ingress Interface: Branch_Inside
   - Forwarding Actions:
     - Match ACL: ACL_PBR_SAAS_BREAKOUT
     - Send To: Egress Interfaces
     - Selected Egress Interfaces: WAN_DIRECT (Priority: 10) -> Save.
3. Save & Deploy 実行。
```

#### 検証方法
* クライアントから WebEx 会議に接続し、`show policy-route` および FTD の `system support firewall-engine-debug` で WebEx トラフィックが `WAN_DIRECT` インターフェイスへ転送されることを確認。

---

### 例8: FTD (FMC 7.1+): 複数 Egress インターフェイス間での Priority 指定 PBR

#### 問題
FTD1 の `LAN_Interface` に到着するインターネット宛て通信に対し、プライマリ WAN (`WAN1`, Priority 10) へ優先転送し、`WAN1` 障害時はセカンダリ WAN (`WAN2`, Priority 20) へ自動フォールバックさせる PBR ポリシーを FMC 上で作成せよ。

#### 要件
* Interface Priority Ordering を使用すること。

#### 設定例

```text
1. FMC: Devices > Device Management > FTD1 > Interfaces 編集:
   - WAN1 インターフェイスの Priority を `10` に設定。
   - WAN2 インターフェイスの Priority を `20` に設定。
2. Routing > Policy Based Routing > Add:
   - Ingress Interface: LAN_Interface
   - Match ACL: ACL_PBR_INTERNET_ALL (permit ip any any)
   - Send To: Egress Interfaces
   - Interface Ordering: By Priority
   - Selected Egress Interfaces: WAN1, WAN2 を追加して Save。
3. Deploy 実行。
```

#### 検証方法
* FTD CLI で `show policy-route` を実行し、`set interface WAN1 WAN2` の優先度順が反映されていることを検証。

---

### 例9: FTD (Pre-7.1): FlexConfig を使用した CLI 互換 PBR ポリシーの定義

#### 問題
Version 7.0 以下の FTD1 において、特定サブネット `10.200.1.0/24` からの通信をプロキシ Gateway `192.168.200.254` へ転送する PBR ポリシーを FlexConfig で展開せよ。

#### 要件
* FlexConfig Policy オブジェクトを使用すること。

#### 設定例

```bash
! 1. FlexConfig Text オブジェクトの作成 (FMC: Objects > FlexConfig > Text Topics)
! Name: Text_PBR_Cmds

access-list ACL_FLEX_PBR extended permit ip 10.200.1.0 255.255.255.0 any
route-map MAP_FLEX_PBR permit 10
 match ip address ACL_FLEX_PBR
 set ip next-hop 192.168.200.254
interface GigabitEthernet0/0
 policy-route route-map MAP_FLEX_PBR

! 2. FlexConfig Object の作成 (Objects > FlexConfig > FlexConfig Objects)
! Name: Object_PBR_Deploy
! Deployment: Every Time, Type: Append
! 内容に Text_PBR_Cmds を挿入。

! 3. FMC: Devices > FlexConfig > FTD1 に Object_PBR_Deploy を追加して Deploy 実行。
```

#### 検証方法
* FTD コンソールで `show policy-route` を実行し、`GigabitEthernet0/0` に `MAP_FLEX_PBR` が適用されていることを確認。

---

### 例10: ASA: uRPF (Unicast Reverse Path Forwarding) 共存環境での PBR 正常化設定

#### 問題
ASA1 の `inside` インターフェイス上で uRPF (`ip verify reverse-path`) が有効化されている環境で PBR を構成したところ、送信元 `10.88.1.0/24` からの通信が PBR 評価前に uRPF ドロップした。uRPF ドロップを回避し、PBR 経由で GW `192.168.88.254` へ正しく転送させるための設定修正を行え。

#### 要件
* uRPF と FIB ルーティングテーブルの整合性を考慮すること。

#### 設定例

```bash
! 現状のドロップ原因: FIB ルーティングテーブルに 10.88.1.0/24 への逆引きルートが存在しない

! 解決策 A: inside ポートの uRPF を無効化
interface GigabitEthernet0/0
 nameif inside
 no ip verify reverse-path interface inside

! 解決策 B (uRPF を維持する場合): 該当送信元サブネットへの Null0 または Null ルートを FIB に追加
ip route inside 10.88.1.0 255.255.255.0 10.0.0.254

! PBR 側の設定
access-list ACL_PBR_URPF extended permit ip 10.88.1.0 255.255.255.0 any
route-map MAP_PBR_URPF permit 10
 match ip address ACL_PBR_URPF
 set ip next-hop 192.168.88.254

interface GigabitEthernet0/0
 policy-route route-map MAP_PBR_URPF
```

#### 検証方法
* `packet-tracer input inside tcp 10.88.1.100 12345 8.8.8.8 80 detail` を実行し、uRPF フェーズを通過して `POLICY-ROUTE` フェーズで `192.168.88.254` へルーティングされることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ASA1 上で以下の PBR コンフィグが適用されている。内部端末 `10.1.1.50` から外部サーバー `8.8.8.8` 宛てに発信されたパケットが `inside` インターフェイスに到着した際、適用されるネクストホップ IP アドレスはどれか。なお、`Track 1` は `UP` 状態、`Track 2` は `DOWN` 状態である。

```bash
access-list ACL_PBR extended permit ip 10.1.1.0 255.255.255.0 any

route-map MAP_PBR permit 10
 match ip address ACL_PBR
 set ip next-hop verify-availability 172.16.1.1 1 track 2
 set ip next-hop verify-availability 192.168.1.1 2 track 1
 set ip next-hop 10.254.254.1
```

* A) `172.16.1.1`
* B) `192.168.1.1`
* C) `10.254.254.1`
* D) 通常の FIB ルーティングテーブルのネクストホップ

**正解**: **B**
**解説**: `set` アクションは定義されたシーケンス順に評価されます。最初の `172.16.1.1` に紐づく `Track 2` は `DOWN` 状態であるためスキップされます。次に評価される `192.168.1.1` の `Track 1` は `UP` 状態であるため、このネクストホップが選択されます。

---

### 2. トラブルシュート問題

**問題**: 管理者が FTD1（Version 7.1）において、特定クライアント `10.5.5.5` からの Web アクセス（TCP 80）を PBR で代理プロキシ `192.168.5.254` へ転送するポリシーを構成した。しかし、クライアントからアクセスを発生させてもプロキシへ送信されず、通常のデフォルトゲートウェイへ送出されてしまう。`show access-list` コマンドで確認した Match ACL の内容は以下の通りである。原因として正しいものはどれか。

```text
access-list ACL_PBR line 1 extended permit tcp host 10.5.5.5 any eq 80
```

* A) Match ACL で送信元ポートが指定されていないため。
* B) クライアントの既存コネクション（Conn Table）が残っており、PBR 評価がスキップされているため。
* C) PBR は UDP トラフィック専用であり、TCP には適用できないため。
* D) `set ip default next-hop` コマンドが欠落しているため。

**正解**: **B**
**解説**: ASA/FTD の PBR は新しい着信接続のファーストパケット（Flow Creation）に対してのみ評価されます。すでに確立済みのコネクションテーブル（Conn Table）が存在する場合、PBR ポリシーは評価されず、以前のフロー情報に従って転送されます。検証時は `clear conn` で既存セッションを削除する必要があります。

---

### 3. Design 問題

**問題**: 大規模企業の拠点（Branch）ネットワークにおいて、Cisco FTD デバイスを使用して拠点からのトラフィックを「社内データセンター宛ての通信（VPN 経由）」と「Microsoft 365 / Salesforce などの特定クラウドサービス宛ての通信（ローカル WAN からの直接インターネットアクセス: DIA）」に動的に分離したい。最も推奨される FTD / FMC 設計要件はどれか。

* A) FMC 7.1 以降のネイティブ PBR 画面で、アプリケーションオブジェクトを含む Match ACL を指定した Ingress PBR ポリシーを作成する。
* B) 全トラフィックを IPsec VPN トンネルへバックホールし、本社側で AVC 制御を行う。
* C) FTD 上で uRPF を有効化し、Static Route の Metric 比較で自動分離させる。
* D) ASA CLI の `asr-group` コマンドを FlexConfig で投入する。

**正解**: **A**
**解説**: FMC 7.1 以降のネイティブ PBR 機能では、アプリケーション（Network Service Groups）に基づく Match ACL を指定することで、特定のクラウドアプリ（Microsoft 365 等）のみをローカルの WAN インターフェイスからダイレクトにインターネットへ抜けさせる Direct Internet Access (DIA) アーキテクチャを最もスマートかつ直感的に構成できます。

---

### 4. 実装問題

**問題**: ASA1 において、特定サブネット `10.20.0.0/16` からの通信に対し、通常のルーティングテーブルに該当宛先の明示的なエントリ（ロンゲストマッチ）が存在する場合はそのルートに従わせ、通常のルックアップが失敗してデフォルトルートに落ちる通信のみ、特別ネクストホップ `192.168.200.1` へ PBR 転送させたい。ルートマップで使用すべき適切な `set` コマンドはどれか。

* A) `set ip next-hop 192.168.200.1`
* B) `set ip next-hop recursive 192.168.200.1`
* C) `set ip default next-hop 192.168.200.1`
* D) `set default interface Null0`

**正解**: **C**
**解説**: `set ip default next-hop` コマンドは、通常の FIB ルーティングテーブル上で宛先 IP に対する明示的なルート検索が失敗した場合にのみ評価・適用される特別な PBR アクションです。

---

### 5. コンフィグ読解問題

**問題**: ASA1 の Ingress ポートにおいて PBR ルートマップが適用されている環境で、受信パケットがルートマップ内の `route-map PBR_MAP deny 10`（Match ACL は permit）にマッチした。このパケットに対する ASA の処理動作として正しいものはどれか。

```bash
route-map PBR_MAP deny 10
 match ip address ACL_MATCH_ALL
```

* A) パケットは PBR 処理によって即座にドロップ（ICMP Unreachable）される。
* B) PBR による経路変更は行われず、通常の FIB ルーティングテーブルに従って転送される。
* C) 次の `route-map` 句（seq 20）へ進んで再評価される。
* D) パケットの TTL が 0 に書き換えられる。

**正解**: **B**
**解説**: ルートマップ文が `deny` であり、かつ Match ACL が `permit`（マッチ成功）した場合、PBR ポリシーの適用が拒否（スキップ）され、処理は通常のアドレスベースルーティングテーブル（FIB）へフォールバックします。

---

## 🔗 参考リソース

* [Cisco ASA Series General Operations Configuration Guide - Policy Based Routing]([suspicious link removed])
* [Cisco Secure Firewall Threat Defense Configuration Guide (FMC) - Policy Based Routing]([suspicious link removed])
* [Cisco Command Reference - policy-route and route-map on ASA]([suspicious link removed])
* [Cisco Technical Notes: Configure and Troubleshoot PBR on ASA and FTD]([suspicious link removed])
* [Cisco Live: Deep Dive into Secure Firewall Routing & DIA Architectures (BRKSEC-3020)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * PBR のコアは **「通常のルーティングテーブル（FIB）を上書きする強力なルール」** です。
  * `set` コマンドの優先度（`verify-availability` ➔ `next-hop` ➔ `recursive` ➔ `interface` ➔ `default next-hop`）と、Match ACL の `deny` / `permit` の関係をしっかり頭に叩き込んでおきましょう。
* **図解（PBR 評価とフォールバック処理の流れ）**:
  ```text
  Ingress Packet
       │
       ▼
  [ Matches PBR Match ACL? ]
       ├── NO  ──► Check Next Route-Map Sequence (or Standard Route Lookup)
       └── YES ──► [ Route-Map Action? ]
                       ├── DENY  ──► Standard Route Lookup (FIB)
                       └── PERMIT ──► Apply 'set' Actions in Priority Order
                                        1. set ip next-hop verify-availability (If Track UP)
                                        2. set ip next-hop
                                        3. set ip next-hop recursive
                                        4. set interface
                                        5. set ip default next-hop
                                        6. Standard Route Lookup (Fallback)
  ```
* **注意点**:
  * 実機検証時には、既存のコネクションテーブル（`show conn`）が残っていると PBR が再評価されません。設定変更後は必ず `clear conn address <IP>` を実行してセッションをクリアしてから通信テストを行ってください。

## 📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

