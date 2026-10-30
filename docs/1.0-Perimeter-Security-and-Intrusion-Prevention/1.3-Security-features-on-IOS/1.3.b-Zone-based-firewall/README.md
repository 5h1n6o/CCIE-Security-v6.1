---
layout: default
title: 1.3.b-Zone-based-firewall
nav_order: 2
parent: 1.3-Security-features-on-IOS
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.3.b Zone-based firewall

Cisco IOS および Cisco IOS XE ソフトウェアにおける **Zone-Based Policy Firewall (ZBFW / ZFW: ゾーンベースポリシーファイアウォール)** は、従来のインターフェイス単位でポリシーを適用していた CBAC (Content-Based Access Control) に代わる、直感的かつ柔軟な次世代ステートフルファイアウォールアーキテクチャです。

ネットワークインターフェイスを「セキュリティゾーン (Security Zone)」と呼ばれる論理的なグループに分類し、ゾーン間のトラフィック方向（Zone-Pair）ごとに Cisco Policy Language (CPL) を用いた一貫したセキュリティポリシー（ステートフルインスペクション、L7 深層検査、URL フィルタリング、DoS 対策）を適用します。

---

## 📘 概要

* **機能概要**:
  * **ゾーン（Zone）ベースのグループ化**: ルータの物理/論理（Subinterface, VLAN, Tunnel, BVI 等）インターフェイスを 1 つ以上のセキュリティゾーン（例: `INSIDE`, `OUTSIDE`, `DMZ`）に割り当てます。
  * **ゾーンペア（Zone-Pair）方向制御**: 送信元ゾーン（Source Zone）と宛先ゾーン（Destination Zone）の組み合わせ（Zone-Pair）ごとに一方向のポリシーを定義します。
  * **Cisco Policy Language (CPL)**: MQC (Modular QoS CLI) に類似した `class-map type inspect`（トラフィック分類）と `policy-map type inspect`（アクション定義）の階層的構造により、L3〜L7 レベルで詳細なステートフル判定を実施します。
  * **Self Zone (セルフゾーン)**: ルータ自身（Control/Management Plane）を発着するトラフィック用にシステム定義された特殊なゾーン (`self`) を保持します。
* **利用目的**:
  * ネットワークのセグメンテーション（社内 LAN、DMZ、ゲスト Wi-Fi、WAN 境界の明確な分離）と動的なセッション管理。
  * アプリケーションレイヤ（HTTP, SMTP, FTP, SIP, DNS 等）のディープパケットインスペクション (DPI) によるプロトコル違反パケットや不正ペイロードの自動排除。
  * DoS / DDoS 攻撃（TCP SYN Flood, UDP/ICMP Flood 等）に対するセッション制限（Embryonic Limits）と自動遮断。
* **どのような場面で利用するか**:
  * 専用ファイアウォールアプライアンス（Cisco ASA や Secure Firewall / FTD）を個別配置できない拠点ルータ（Cisco ISR 1000/4000, ASR 1000, Catalyst 8000 等）での境界セキュリティ構築。
  * DMVPN, FlexVPN, SD-WAN 環境におけるハブ/スポーク間、または Direct Internet Access (DIA) ポイントでの動的トラフィック制御。
  * 透明型（Transparent / L2 Bridge）ルータ構成におけるインライン高度セキュリティ検査。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | インターフェイスを論理ゾーンにまとめ、CPL (Class/Policy Map) で方向性を持ったステートフル検査を定義 |
| **デフォルト動作 (Inter-Zone)** | 異なるゾーン間のトラフィックは **暗黙の拒否 (Implicit Deny / Drop)** |
| **デフォルト動作 (Intra-Zone)** | 同一ゾーン内のインターフェイス間トラフィックは **デフォルト許可 (Pass)**（変更可能） |
| **デフォルト動作 (Non-Zone)** | ゾーン非所属インターフェイスとゾーン所属インターフェイス間の通信は **完全ドロップ** |
| **Self Zone デフォルト動作** | ルータ自装置 (`self`) とゾーン間通信は **デフォルト許可**（Zone-Pair 適用で明示制御に移行） |
| **主なアクション** | **Inspect** (ステートフル検査と戻り通信動的許可), **Pass** (単方向通過), **Drop** (破棄/ログ) |
| **対応機種** | Cisco ISR 800/1000/4000, ASR 1000, CSR 1000v, Catalyst 8000, Cisco IOS 12.4(6)T+, IOS XE |
| **設計上の注意点** | 1 インターフェイスは 1 ゾーンにのみ所属可能。NAT 処理は ZBFW インスペクションと相互作用する |

---

## 🏗 動作原理

ZBFW は、ルータのパケット転送パイプラインと深く統合されたステートフルセッションテーブル（PAM: Port-to-Application Mapping 含む）に基づいてトラフィックを追跡・制御します。

### 1. ZBFW トラフィックフローと Zone-Pair アーキテクチャ

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Router Running Cisco IOS XE ZBFW                                                 │
│                                                                                  │
│   [ INSIDE Zone ]                  [ Zone-Pair: IN-TO-OUT ]    [ OUTSIDE Zone ]  │
│   ┌───────────────┐               ┌───────────────────────┐   ┌───────────────┐ │
│   │ Gi0/0/1       │               │ Class-Map: CM_IN_OUT  │   │ Gi0/0/2       │ │
│   │ (10.1.1.0/24) │ ────────────► │   Action: INSPECT     │ ─►│ (Public IP)   │ │
│   └───────────────┘               └───────────────────────┘   └───────────────┘ │
│           │                                   │                       │         │
│           │                                   ▼                       │         │
│           │                       ┌───────────────────────┐           │         │
│           │                       │  State Table (PAM)    │           │         │
│           │                       │  Session Entry Created│           │         │
│           │                       └───────────────────────┘           │         │
│           │                                   ▲                       │         │
│           │                                   │                       │         │
│           │ ◄─────────────────────────────────┴───────────────────────┘         │
│             Return Traffic Automatically Permitted by Stateful Engine            │
│                                                                                  │
│   ─────────────────────────────────────────────────────────────────────────────  │
│   [ Self Zone: 'self' ]                                                          │
│   ルータ自装置 (BGP, OSPF, SSH, SNMP, Ping 等) 宛て / 発の制御・管理通信            │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

パケットが ZBFW 適用ルータに到着した際の評価シーケンスは以下の順番で実行されます。

1. **Ingress ポートおよび Egress ポートのゾーン判定**:
   * パケット着信インターフェイスのゾーン（Source Zone）と、CEF (Cisco Express Forwarding) ルックアップによって決定された出力インターフェイスのゾーン（Destination Zone）を識別します。
2. **既存セッションテーブル（State Table）の検索**:
   * 送信元 IP、宛先 IP、送信元ポート、宛先ポート、プロトコルの 5 チュートプルによりアクティブセッションを検索します。
   * **既存セッションにマッチ（戻りパケット等）**: ポリシー評価をスキップし、インスペクションエンジンにより即座に通過（Pass）処理されます。
3. **Zone-Pair の照合**:
   * 新規セッション（TCP SYN 等）の場合、該当する `zone-pair security`（例: `source INSIDE destination OUTSIDE`）が存在するか確認します。
   * **Zone-Pair 未存在**: パケットは即座に暗黙のドロップ（Implicit Drop）となります。
4. **Policy-Map (CPL) の上から順（Top-Down）評価**:
   * Zone-Pair にバインドされた `policy-map type inspect` 内の `class-map` を順次評価します。
   * **Inspect**: パケットの L4〜L7 ヘッダーを解析し、セッションテーブルに動的エントリー（コントロール/データチャネル含む）を作成して通過させます。
   * **Pass**: パケットを無検査で通過させます（※ 戻りパケット用の動的セッションは作成されません）。
   * **Drop**: パケットをログ出力（`drop log`）または静かに破棄します。
5. **Egress インターフェイスからの送出**:
   * 許可またはインスペクションされたパケットは、NAT や Egress QoS 処理を経て送信されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Pass vs Inspect の動的セッション作成の違い**:
   * **`inspect`**: 往路のパケットを検査し、ステートテーブルにセッションを自動生成します。復路（逆方向）の Zone-Pair や明示的な許可ルールがなくても、戻りパケットは自動的に許可されます。
   * **`pass`**: 単方向のみの許可です。**戻りパケット用のセッションテーブルは生成されない**ため、双方向で通信させるには逆方向の Zone-Pair にも明示的な Pass または Inspect ポリシーが必要になります。
2. **Self Zone (`self`) の小文字指定と特殊ルール**:
   * 自装置宛て/自装置発の通信を制御するゾーン名は、**すべて小文字の `self`** でなければなりません（大文字 `SELF` で作成すると単なる通常のユーザー定義ゾーンとして扱われます）。
   * 一度でも `source <ZONE> destination self` や `source self destination <ZONE>` の Zone-Pair を作成すると、当該ゾーンとルータ間の通信は暗黙許可から **暗黙拒否（Implicit Deny）** へ移行します。SSH、SNMP、NTP、BGP、OSPF などの必要な管理/制御プロトコルを明示的に許可 (`pass` または `inspect`) しないと、ルータへのアクセスやルーティングアジャセンシーが全滅します。
3. **`class-map` の `match-all` と `match-any` の厳格な使い分け**:
   * **`match-any`**: 指定したいずれかの条件（例: `match protocol tcp`, `match protocol udp`）にマッチすれば許可。
   * **`match-all`**: 指定したすべての条件に同時に合致する必要があるため、複数の `match protocol`（例: `match protocol http` と `match protocol smtp`）を並べると、1 つのパケットが両方であることは不可能なため**永久にマッチしません**。
4. **PAM (Port-to-Application Mapping) の変更と非標準ポートインスペクション**:
   * Web サーバーやプロキシが非標準ポート（例: TCP 8080, 8443）を使用している場合、単に `match protocol http` だけでは NBAR / PAM エンジンが 8080 を HTTP と認識しない場合があります。
   * 事前に **`ip port-map http port tcp 8080`** を設定して PAM テーブルを更新する必要があります。
5. **Intra-Zone トラフィックの明示的ブロック要件**:
   * 同一ゾーン内のインターフェイス間通信はデフォルトで全許可となります。試験問題で「同一ゾーン内の通信を拒否せよ」と指示された場合は、送信元と宛先に同じゾーンを指定した Zone-Pair（例: `zone-pair security ZP_SAME source INSIDE destination INSIDE`）を作成し、`drop` ポリシーをバインドします。
6. **Transparent Mode (BVI) における DHCP パススルー**:
   * ZBFW を L2 透過型（Transparent Mode / BVI）で構成する場合、デフォルトでは DHCP パケットがドロップされることがあります。
   * **`parameter-map type inspect global`** モード配下で **`l2-transparent dhcp-passthrough`** を有効化する必要があるか常に確認してください。

---

## 🛠 設定方法

### 1. Cisco IOS XE: 基本的な Routed Mode 3-Zone ZBFW 構成 (CLI)

```bash
! ===== 1. セキュリティゾーンの作成 =====
zone security INSIDE
zone security OUTSIDE
zone security DMZ

! ===== 2. ゾーンペアの定義 =====
zone-pair security ZP_IN_OUT source INSIDE destination OUTSIDE
zone-pair security ZP_IN_DMZ source INSIDE destination DMZ
zone-pair security ZP_OUT_DMZ source OUTSIDE destination DMZ
zone-pair security ZP_IN_SELF source INSIDE destination self

! ===== 3. トラフィック分類 (Class-Map) の定義 =====
ip access-list extended ACL_ADMIN_HOSTS
 permit ip host 10.1.1.50 any

class-map type inspect match-any CM_OUTBOUND_TRAFFIC
 match protocol tcp
 match protocol udp
 match protocol icmp

class-map type inspect match-any CM_PUBLIC_SERVICES
 match protocol http
 match protocol https
 match protocol dns

class-map type inspect match-all CM_ADMIN_SELF
 match access-group name ACL_ADMIN_HOSTS
 match protocol ssh

! ===== 4. ポリシーマップ (Policy-Map) の定義 =====
policy-map type inspect PM_IN_OUT
 class CM_OUTBOUND_TRAFFIC
  inspect
 class class-default
  drop log

policy-map type inspect PM_OUT_DMZ
 class CM_PUBLIC_SERVICES
  inspect
 class class-default
  drop log

policy-map type inspect PM_IN_SELF
 class CM_ADMIN_SELF
  pass
 class class-default
  drop

! ===== 5. ゾーンペアへのポリシーバインド =====
zone-pair security ZP_IN_OUT
 service-policy type inspect PM_IN_OUT

zone-pair security ZP_OUT_DMZ
 service-policy type inspect PM_OUT_DMZ

zone-pair security ZP_IN_SELF
 service-policy type inspect PM_IN_SELF

! ===== 6. インターフェイスへのゾーン割り当て =====
interface GigabitEthernet0/0/1
 description LAN
 zone-member security INSIDE

interface GigabitEthernet0/0/2
 description WAN
 zone-member security OUTSIDE

interface GigabitEthernet0/0/3
 description DMZ
 zone-member security DMZ
```

### 2. Cisco IOS XE: Parameter-Map によるセッションタイマー/Embryonic 制限のチューニング

```bash
! ===== 1. カスタム Inspector Parameter-Map の作成 =====
parameter-map type inspect PMAP_TUNE_TCP
 tcp idle-time 120
 udp idle-time 15
 max-incomplete high 100
 max-incomplete low 80
 one-minute high 1000
 one-minute low 800

! ===== 2. Policy-Map への Parameter-Map の適用 =====
policy-map type inspect PM_STRICT_INSPECT
 class CM_OUTBOUND_TRAFFIC
  inspect PMAP_TUNE_TCP
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **アクティブな ZBFW セッションテーブル一覧の確認** | <code>show policy-firewall session</code> / <code>show policy-map type inspect zone-pair sessions</code> |
| **ゾーンペアごとの詳細統計（Match パケット数、Drop 数）確認** | <code>show policy-map type inspect zone-pair</code> |
| **設定済みゾーン一覧および所属インターフェイス表示** | <code>show zone security</code> / <code>show policy-firewall config zone</code> |
| **設定済みゾーンペアおよび適用ポリシーマップの統合表示** | <code>show zone-pair security</code> |
| **PAM (Port-to-Application Mapping) データベースの表示** | <code>show ip port-map</code> / <code>show ip port-map http</code> |
| **ドロップカウンターおよびエラー統計の表示** | <code>show policy-firewall stats drop-counters</code> |
| **ZBFW セッション生成・削除のリアルタイムデバッグ** | <code>debug policy-firewall detailed</code> / <code>debug ip firewall detailed</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **戻りパケットがドロップされ双方向通信が成立しない** | `inspect` ではなく `pass` を使用している | <code>show zone-pair security</code> | 往路ポリシーで `inspect` を使用するか、復路の Zone-Pair にも Pass を追加する。 |
| **`self` ゾーンペア設定後、BGP/OSPF ネイバーが Down した** | 制御プロトコルが Class-Map で許可されていない | <code>show policy-map type inspect zone-pair</code> | Self Zone ペアに OSPF (`match protocol ospf`) や BGP の Pass/Inspect ルールを追加する。 |
| **非標準ポート (8080) の Web 通信が検査でドロップされる** | PAM テーブルに非標準ポートが登録されていない | <code>show ip port-map http</code> | `ip port-map http port tcp 8080` を投入して PAM を動的追加する。 |
| **同一ゾーンに所属する端末同士の通信が切断される** | 明示的に Intra-Zone の Zone-Pair で Drop を設定してしまっている | <code>show zone-pair security</code> | Intra-Zone の Drop ルールを修正・削除する。 |
| **Transparent モードで端末が IP アドレスを取得できない** | ZBFW で DHCP トラフィックがブロックされている | <code>show policy-firewall stats drop-counters</code> | `parameter-map type inspect global` で `l2-transparent dhcp-passthrough` を有効化する。 |

---

## ⚠ 制限事項

* **Cisco IOS/IOS XE ZBFW の制限事項**:
  * **CBAC との同時併用不可**: 1 つのインターフェイス上で古典的 CBAC (`ip inspect`) と ZBFW (`zone-member security`) を同時に設定することはできません（ルータ内で混在させる場合はインターフェイスごとに分離が必要）。
  * **1 インターフェイス 1 ゾーン制約**: 1 つの物理/論理インターフェイスは複数のセキュリティゾーンに同時に所属することはできません。
  * **Multi-VRF 環境での Zone-Pair**: VRF を跨ぐ Zone-Pair の設定には適切なルーツターゲット（RT）インポートまたは Inter-VRF 定義が必要です。

---

## 🔄 他技術との関連

* **1.2.a NAT (Network Address Translation)**: ZBFW と NAT を同時適用する場合、**Ingress パイプラインでは NAT が先 (Real IP ➔ Mapped IP) 、ZBFW が後**、**Egress パイプラインでは ZBFW が先、NAT が後** となります。ZBFW の ACL や Class-Map は **送信元/宛先の「Real IP (変換前の実 IP)」** を基準に作成する必要があります。
* **VPN (IPsec / FlexVPN / DMVPN)**: IPsec トンネル通信（ESP, UDP 500/4500）をルータ自身で終端する場合、`self` ゾーンに対する IKE/ESP 許可ルール（Self Zone Policy）が必須となります。また、Tunnel インターフェイス自体を専用ゾーン（例: `VPN_ZONE`）に割り当てて明示的に制御します。
* **Control Plane Policing (CoPP)**: ルータ自装置宛てのトラフィックに対しては、ZBFW (Self Zone) の処理が CoPP よりも前に実行されます。

---

## 🧩 比較表

### 1. ZBFW (Zone-Based Firewall) vs CBAC (Classic IOS Firewall)

| 比較項目 | ZBFW (Zone-Based Policy Firewall) | CBAC (Content-Based Access Control) |
| :--- | :--- | :--- |
| **適用単位** | **論理セキュリティゾーン (Zone / Zone-Pair)** | **個別物理/論理インターフェイス** |
| **設定構文** | Cisco Policy Language (**CPL**: Class/Policy-Map) | レガシー CLI (`ip inspect name ...`) |
| **デフォルト動作** | ゾーン間は **デフォルト拒否 (Deny All)** | インターフェイス間は **デフォルト許可 (Permit All)** |
| **自装置保護** | **Self Zone (`self`)** による一元管理 | インターフェイス ACL に依存 |
| **拡張性・管理性** | インターフェイス数が増えてもゾーン追加で対応可能 | インターフェイス毎に ACL/Inspect 適用が必要で過重 |

### 2. ZBFW アクションの比較 (Inspect vs Pass vs Drop)

| アクション | 動作概要 | ステートテーブル生成 | 復路パケットの扱い |
| :--- | :--- | :--- | :--- |
| **Inspect** | パケットをディープ検査し動的セッション作成 | **生成する** | **自動的に許可 (Pass)** |
| **Pass** | パケットを無検査で単方向通過 | **生成しない** | **逆方向ポリシーがない限りドロップ** |
| **Drop** | パケットを静かに破棄（または `drop log`） | 生成しない | 該当なし |

---

## 💡 ベストプラクティス

1. **Self Zone ポリシーの事前設計と管理ポートの保護**:
   * ZBFW 構成を開始する際、最初に `self` ゾーン関連の Zone-Pair を設計し、SSH, ICMP, SNMP, BGP/OSPF を明示的に許可 (`pass` または `inspect`) してから他のゾーンルールを設定します。
2. **`class-default` に対する明示的な `drop log` のバインド**:
   * すべての `policy-map type inspect` の `class class-default` 配下に **`drop log`** を設定しておくことで、セキュリティ違反や不審な拒否トラフィックを Syslog 経由で即座に追跡可能にします。
3. **`parameter-map type inspect global` でのドロップログ有効化**:
   * ポリシーマッピングの不一致以外の理由（パケットフォーマット違反、セッション超過等）でドロップされたパケットを記録するため、**`log dropped-packets enable`** をグローバルパラメータマップに設定します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Routed Mode 2-Zone (INSIDE ↔ OUTSIDE) 基本ステートフルインスペクション

#### 問題
ルータ R1 の `Gi0/1.12` (`INSIDE`) から `Gi0/1.24` (`OUTSIDE`) 宛ての ICMP, TCP, UDP, TELNET 通信をステートフルインスペクション許可せよ。OUTSIDE から INSIDE への新規通信は拒否せよ。

#### 要件
* ゾーン名: `INSIDE`, `OUTSIDE`
* ゾーンペア名: `INSIDE_TO_OUTSIDE`
* CPL を使用して構築すること。

#### 設定例

```bash
! 1. ゾーンの定義
zone security INSIDE
zone security OUTSIDE

! 2. Class-Map の定義
class-map type inspect match-any CM_INSIDE_TO_OUTSIDE
 match protocol icmp
 match protocol tcp
 match protocol udp
 match protocol telnet

! 3. Policy-Map の定義
policy-map type inspect PM_INSIDE_TO_OUTSIDE
 class CM_INSIDE_TO_OUTSIDE
  inspect
 class class-default
  drop log

! 4. Zone-Pair の定義と適用
zone-pair security INSIDE_TO_OUTSIDE source INSIDE destination OUTSIDE
 service-policy type inspect PM_INSIDE_TO_OUTSIDE

! 5. インターフェイスへの割当
interface GigabitEthernet0/1.12
 zone-member security INSIDE

interface GigabitEthernet0/1.24
 zone-member security OUTSIDE
```

#### 検証方法
* INSIDE 側の端末から OUTSIDE 宛てに Ping および Telnet を実行。
* R1 で `show policy-map type inspect zone-pair sessions` を実行し、`SIS_OPEN` セッションが確立されることを確認。
* OUTSIDE から INSIDE 宛ての Ping がタイムアウト（ドロップ）し、`show policy-firewall stats drop-counters` のカウントが増加することを確認。

---

### 例2: Self-Zone 制御 (INSIDE からの SSH/ICMP 許可および BGP 通信の維持)

#### 問題
ルータ R1 自身 (`self` ゾーン) に対し、`INSIDE` (`Gi0/1.12`) からの ICMP Ping および SSH (TCP 22) 通信のみを許可せよ。また、`OUTSIDE` ピア (`190.1.24.4`) との BGP (TCP 179) セッションが遮断されないよう設定せよ。

#### 要件
* Self Zone 用の Zone-Pair を正しく作成すること。

#### 設定例

```bash
! 1. Class-Map の作成
ip access-list extended ACL_MGMT_IN
 permit tcp any any eq 22
 permit icmp any any

ip access-list extended ACL_BGP
 permit tcp host 190.1.24.4 any eq 179
 permit tcp any host 190.1.24.4 eq 179

class-map type inspect match-all CM_INSIDE_TO_SELF
 match access-group name ACL_MGMT_IN

class-map type inspect match-all CM_BGP_PASS
 match access-group name ACL_BGP

! 2. Policy-Map の作成
policy-map type inspect PM_INSIDE_TO_SELF
 class CM_INSIDE_TO_SELF
  pass
 class class-default
  drop

policy-map type inspect PM_OUTSIDE_TO_SELF
 class CM_BGP_PASS
  pass
 class class-default
  drop

! 3. Zone-Pair の定義
zone-pair security ZP_IN_SELF source INSIDE destination self
 service-policy type inspect PM_INSIDE_TO_SELF

zone-pair security ZP_OUT_SELF source OUTSIDE destination self
 service-policy type inspect PM_OUTSIDE_TO_SELF
```

#### 検証方法
* `INSIDE` 端末から R1 の IP アドレス宛てに SSH 接続を試し疎通することを確認。
* `show bgp summary` で BGP ピアリングが `Established` のまま維持されることを確認。

---

### 例3: L7 深層インスペクション (HTTP/SMTP Parameter-Map / AIC)

#### 問題
`OUTSIDE` から `INSIDE` の DMZ メールサーバー宛て SMTP トラフィックに対し、深層検査（AIC）を適用せよ。SMTP コマンドのデータ長制限（最大 10MB）を適用し、不審なコマンドをリセット遮断せよ。

#### 要件
* `policy-map type inspect smtp` を使用すること。

#### 設定例

```bash
! 1. L7 SMTP Policy-Map / Parameter-Map の定義
policy-map type inspect smtp PMAP_L7_SMTP
 match data-length gt 10000000
  reset log

! 2. L3/L4 Class-Map および Policy-Map の定義
class-map type inspect match-any CM_SMTP
 match protocol smtp

policy-map type inspect PM_OUT_TO_INSIDE
 class CM_SMTP
  inspect
  service-policy smtp PMAP_L7_SMTP

! 3. Zone-Pair 適用
zone-pair security OUT_TO_IN source OUTSIDE destination INSIDE
 service-policy type inspect PM_OUT_TO_INSIDE
```

#### 検証方法
* `show policy-map type inspect zone-pair` を実行し、`Service Policy: smtp PMAP_L7_SMTP` が正常にネストされていることを確認。

---

### 例4: PAM (Port-to-Application Mapping) による非標準ポート (TCP 8080) インスペクション

#### 問題
`INSIDE` から `OUTSIDE` 宛ての Web 通信において、標準ポート (TCP 80) に加え、非標準ポート **TCP 8080** を使用する HTTP 通信も NBAR / ZBFW インスペクションエンジンで動的検査・許可せよ。

#### 要件
* `ip port-map` コマンドを使用すること。

#### 設定例

```bash
! 1. PAM テーブルへの非標準ポートの追加
ip port-map http port tcp 8080

! 2. Class-Map / Policy-Map の定義
class-map type inspect match-any CM_HTTP_ALL
 match protocol http

policy-map type inspect PM_HTTP_ALL
 class CM_HTTP_ALL
  inspect

zone-pair security IN_TO_OUT source INSIDE destination OUTSIDE
 service-policy type inspect PM_HTTP_ALL
```

#### 検証方法
* クライアントから `http://<OUTSIDE_IP>:8080` へ接続。
* `show policy-map type inspect zone-pair sessions` で 8080 ポートのセッションが HTTP として判定・許可されていることを確認。

---

### 例5: 透明型 (Transparent Mode / BVI) ZBFW と DHCP Passthrough 構成

#### 問題
ルータ R7 を透明型（L2 Bridge / BVI17）ファイアウォールとして構成し、`VLAN76` ゾーンから `VLAN17` ゾーンへの TCP/UDP/ICMP をインスペクション許可せよ。また、Transparent モードで DHCP トラフィックがドロップされないようグローバル設定を追加せよ。

#### 要件
* `parameter-map type inspect global` を使用すること。

#### 設定例

```bash
! 1. グローバルパラメータマップでの DHCP パススルー有効化
parameter-map type inspect global
 l2-transparent dhcp-passthrough

! 2. Class-Map / Policy-Map / Zone 設定
class-map type inspect match-any CM_IP_TRAFFIC
 match protocol tcp
 match protocol udp
 match protocol icmp

policy-map type inspect PM_VLAN76_TO_17
 class CM_IP_TRAFFIC
  inspect

zone security VLAN17
zone security VLAN76

zone-pair security VLAN76_TO_17 source VLAN76 destination VLAN17
 service-policy type inspect PM_VLAN76_TO_17

interface FastEthernet0/0.17
 zone-member security VLAN17

interface FastEthernet0/0.76
 zone-member security VLAN76

interface Bvi17
 zone-member security VLAN17
```

#### 検証方法
* `show policy-firewall config parameter-map global` を実行し、`l2-transparent dhcp-passthrough enabled` になっていることを確認。
* VLAN76 のクライアントが DHCP 経由で IP アドレスを正常取得できることを確認。

---

### 例6: High Availability (HA) / Redundancy Application Group (RAG) 統合アクティブ/スタンバイ ZBFW

#### 問題
ルータ R1 (Active, Priority 150) と R2 (Standby) 間で Redundancy Application Group (RAG) 1 を作成し、ZBFW のステートフルセッション情報を同期（Active/Standby HA）させよ。データ/コントロールポートとして `Gi0/1` を使用せよ。

#### 要件
* `application redundancy` を使用すること。

#### 設定例

```bash
! ===== R1 (Active) 構成 =====
redundancy
 application redundancy
  group 1
   name ZBPF_HA
   priority 150
   preempt
   control GigabitEthernet0/1 protocol 1
   data GigabitEthernet0/1

interface GigabitEthernet0/1.12
 redundancy rii 12
 redundancy group 1 ip 190.1.12.254 exclusive decrement 10

interface GigabitEthernet0/1.24
 redundancy rii 24
 redundancy group 1 ip 190.1.24.254 exclusive decrement 10
```

#### 検証方法
* `show redundancy application group 1` を実行し、R1 が `Master` (Active)、R2 が `Standby` であることを確認。
* R1 でセッション生成後、`show policy-firewall session` で `HA State: ACTIVE` のセッションが表示され、フェイルオーバー後もセッションが維持されることを確認。

---

### 例7: URL Filtering (Local Glob Match) と ZBFW の統合設定

#### 問題
`UNTRUSTED` から `TRUSTED` 宛ての HTTP アクセスにおいて、特定ドメイン（`*.facebook.com`, `*.reddit.com`）へのアクセスをローカル URL フィルタリングで遮断（Reset & Log）し、ブロック画面メッセージを出力せよ。

#### 要件
* `parameter-map type urlf-glob` および `policy-map type inspect urlfilter` を使用すること。

#### 設定例

```bash
! 1. URL パターンマップの作成
parameter-map type urlf-glob BLOCK_URL
 pattern *.facebook.com
 pattern *.reddit.com

parameter-map type urlf-glob ALLOW_URL
 pattern *

! 2. URL フィルタ Class-Map の作成
class-map type urlfilter match-any BLOCK_MAP
 match server-domain urlf-glob BLOCK_URL

class-map type urlfilter match-any ALLOW_MAP
 match server-domain urlf-glob ALLOW_URL

! 3. ブロックメッセージ用 Parameter-Map
parameter-map type urlfpolicy local BANNER
 block-page message "Access to this content has been restricted by Corporate Policy."

! 4. URL フィルタ Policy-Map の作成
policy-map type inspect urlfilter HTTP_URL_POLICY
 parameter type urlfpolicy local BANNER
 class type urlfilter BLOCK_MAP
  reset
  log
 class type urlfilter ALLOW_MAP
  allow

! 5. 親 ZBFW Policy-Map へのバインド
class-map type inspect match-any CM_HTTP_INSPECT
 match protocol http

policy-map type inspect PM_UNTRUSTED_TO_TRUSTED
 class CM_HTTP_INSPECT
  inspect
  service-policy urlfilter HTTP_URL_POLICY

zone-pair security ZP_UN_TR source UNTRUSTED destination TRUSTED
 service-policy type inspect PM_UNTRUSTED_TO_TRUSTED
```

#### 検証方法
* クライアントから `[suspicious link removed]` へアクセスを試み、カスタムブロックページが表示されて接続が切断されることを確認。

---

### 例8: Dynamic VPN (DMVPN / FlexVPN) と Self-Zone / ZBFW 統合構成

#### 問題
ルータ R4 (DMVPN Hub) において、スポークからの IKE (UDP 500) および ESP (IP proto 50) パケットを Self Zone ポリシーで許可せよ。また、DMVPN トンネルインターフェイス (`Tunnel124`) を `TRUSTED` ゾーンにアサインしてデータ通信を検査せよ。

#### 要件
* Self Zone ACL に UDP 500 および ESP を追加すること。

#### 設定例

```bash
! 1. Self Zone 用 ACL の定義
ip access-list extended ACL_DMVPN_SELF
 permit udp any host 150.1.4.4 eq 500
 permit udp any host 150.1.4.4 eq 4500
 permit esp any host 150.1.4.4

class-map type inspect match-all CM_SELF_DMVPN
 match access-group name ACL_DMVPN_SELF

policy-map type inspect PM_UNTRUSTED_TO_SELF
 class CM_SELF_DMVPN
  pass
 class class-default
  drop

zone-pair security ZP_UN_SELF source UNTRUSTED destination self
 service-policy type inspect PM_UNTRUSTED_TO_SELF

! 2. DMVPN トンネルのゾーンバインド
interface Tunnel124
 zone-member security TRUSTED
```

#### 検証方法
* スポークルータから IPsec / DMVPN トンネルを確立させ、`show crypto ipsec sa` で SA が正常生成されることを確認。
* `show policy-map type inspect zone-pair` で Self Zone の Pass パケット数が増加していることを確認。

---

### 例9: Intra-Zone Traffic の拒否と Multi-Zone ポリシー

#### 問題
ルータ R1 の `INSIDE` ゾーンに 2 つのサブインターフェイス (`Gi0/1.10`, `Gi0/1.20`) が所属している。デフォルトで許可されている同一ゾーン内 (Intra-Zone) の通信を明示的に遮断 (Drop) せよ。

#### 要件
* 送信元・宛先に同一ゾーンを指定した Zone-Pair を作成すること。

#### 設定例

```bash
! 1. 同一ゾーンを指定する Zone-Pair の作成
zone-pair security ZP_INTRA_INSIDE source INSIDE destination INSIDE

! 2. ドロップ用 Policy-Map の定義
policy-map type inspect PM_BLOCK_INTRA
 class class-default
  drop log

! 3. Zone-Pair への適用
zone-pair security ZP_INTRA_INSIDE
 service-policy type inspect PM_BLOCK_INTRA
```

#### 検証方法
* `Gi0/1.10` 上の端末から `Gi0/1.20` 上の端末宛てに Ping を実行し、通信が遮断され Syslog にドロップログが出力されることを確認。

---

### 例10: ZBFW と Dynamic Static NAT (Ingress/Egress) 連携

#### 問題
内部 Web サーバー (`10.1.1.100`) が Dynamic Static NAT によりパブリック IP `203.0.113.100` へ変換されている。外部 (`OUTSIDE`) から内部 (`INSIDE`) の Web サーバー宛ての HTTP アクセスを許可する ZBFW ポリシーを構築せよ。

#### 要件
* ZBFW の ACL には **変換前の実 IP (Real IP: 10.1.1.100)** を指定すること。

#### 設定例

```bash
! 1. NAT の設定
ip nat inside source static 10.1.1.100 203.0.113.100

! 2. ZBFW ACL の定義 (Real IP アドレス 10.1.1.100 を指定)
ip access-list extended ACL_REAL_WEB_SERVER
 permit tcp any host 10.1.1.100 eq 80

class-map type inspect match-all CM_WEB_REAL
 match access-group name ACL_REAL_WEB_SERVER
 match protocol http

policy-map type inspect PM_OUT_TO_IN
 class CM_WEB_REAL
  inspect

zone-pair security OUT_TO_IN source OUTSIDE destination INSIDE
 service-policy type inspect PM_OUT_TO_IN
```

#### 検証方法
* 外部クライアントから `http://203.0.113.100` へアクセス。
* ZBFW が Ingress NAT 後の Real IP (`10.1.1.100`) で正常にパケットを照合・許可し、Web ページが表示されることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ルータ R1 上で以下の ZBFW コンフィグが適用されている。`INSIDE` ゾーンのホスト `10.1.1.10` から `OUTSIDE` ゾーンのサーバー `203.0.113.50` 宛てに Telnet 接続を開始した。パケットの処理結果として正しいものはどれか。

```bash
class-map type inspect match-all CM_TELNET
 match protocol telnet

policy-map type inspect PM_IN_OUT
 class CM_TELNET
  pass

zone-pair security ZP_IN_OUT source INSIDE destination OUTSIDE
 service-policy type inspect PM_IN_OUT
```

* A) 往路の Telnet SYN パケットが通過し、戻りの SYN-ACK パケットも自動的に許可されて Telnet 接続が完了する。
* B) 往路の Telnet SYN パケットは通過するが、ステートテーブルが生成されない（`pass` アクション）ため、サーバーからの戻りパケットが OUTSIDE インターフェイスでドロップされ接続がタイムアウトする。
* C) `CM_TELNET` に ACL が指定されていないため、すべてのパケットが即座にドロップされる。
* D) `pass` アクションにより自動的に UDP トラフィックにカプセル化される。

**正解**: **B**
**解説**: `pass` アクションは単方向のみの通過許可であり、ステートテーブルを生成しません。そのため、戻り方向（`OUTSIDE_TO_INSIDE`）の Zone-Pair ポリシーが存在しない場合、復路のレスポンスパケットが暗黙のドロップによって拒否され接続が失敗します。ステートフルに戻り通信を許可するには `inspect` を使用する必要があります。

---

### 2. トラブルシュート問題

**問題**: 管理者が R1 上で `INSIDE` ゾーンと `self` ゾーン間に Zone-Pair を構成し、SSH のみを許可するポリシーを適用した直後、`OUTSIDE` インターフェイス経由で確立していた BGP ネイバーセッション (`190.1.24.4`) が Down してしまった。原因と適切な対処方法はどれか。

* A) `self` ゾーンにポリシーを設定したことで、すべての物理ポート上の BGP トラフィックがグローバルにドロップされた。`OUTSIDE` ➔ `self` の Zone-Pair を作成し、BGP (TCP 179) を明示的に許可 (`pass` / `inspect`) する。
* B) BGP は UDP プロトコルであるため、`inspect icmp` に変更する。
* C) ルータを再起動する。
* D) `control-plane host` で BGP を有効化する。

**正解**: **A**
**解説**: ZBFW において `self` ゾーンが絡む Zone-Pair を 1 つでも定義すると、ルータ自装置宛ての通信はデフォルト許可から**暗黙の拒否 (Implicit Deny)** へ変化します。OUTSIDE からの BGP セッション（TCP 179）を維持するには、`OUTSIDE` ➔ `self` の Zone-Pair に BGP 許可ルールを追加する必要があります。

---

### 3. Design 問題

**問題**: ルータを L2 透明型 (Transparent Bridge / BVI) ファイアウォールとして拠点に挿入し、ZBFW によるインスペクションを有効化した。しかし、配下の PC が DHCP サーバーから IP アドレスを取得できなくなった。ルータの設定を変更して端末が DHCP アドレスを取得できるようにするための正しいコマンドはどれか。

* A) `ip dhcp relay information option`
* B) `parameter-map type inspect global` モード配下で `l2-transparent dhcp-passthrough` を設定する。
* C) `interface BVI1` 配下で `no ip route-cache` を設定する。
* D) `bridge-group 1 subscriber-group` を適用する。

**正解**: **B**
**解説**: L2 透過型 ZBFW では、デフォルトで DHCP ブロードキャスト/ユニキャストパケットが遮断される場合があります。グローバルパラメータマップ内で `l2-transparent dhcp-passthrough` を有効化することで、DHCP パケットを透過させることができます。

---

### 4. 実装問題

**問題**: ルータ R1 において、非標準ポート TCP 8080 を使用する Web サーバー宛ての HTTP 通信を ZBFW の `match protocol http` でインスペクションできるようにするための正しい事前設定コマンドはどれか。

* A) `ip port-map http port tcp 8080`
* B) `class-map type inspect http 8080`
* C) `parameter-map type inspect port 8080`
* D) `service-policy http port 8080`

**正解**: **A**
**解説**: ZBFW の PAM (Port-to-Application Mapping) データベースに非標準ポートを追加・登録するには **`ip port-map <protocol> port <transport> <port_number>`** コマンドを使用します。

---

### 5. コンフィグ読解問題

**問題**: 以下の ZBFW コンフィグが適用されている。`INSIDE` ゾーンのホスト `10.1.1.50` から `OUTSIDE` の DNS サーバー (`8.8.8.8`) 宛てに UDP 53 パケットを送信した際、コンフィグ上の問題点として正しいものはどれか。

```bash
class-map type inspect match-all CM_DNS_HTTP
 match protocol dns
 match protocol http

policy-map type inspect PM_IN_OUT
 class CM_DNS_HTTP
  inspect

zone-pair security IN_OUT source INSIDE destination OUTSIDE
 service-policy type inspect PM_IN_OUT
```

* A) `inspect` アクションは UDP をサポートしていない。
* B) `class-map match-all` を使用しているため、単一パケットが DNS かつ HTTP である条件を満たせず、パケットは永久にマッチしない（`match-any` にすべき）。
* C) `zone-pair` の文法エラーが発生する。
* D) DNS トラフィックは Self Zone でしか検査できない。

**正解**: **B**
**解説**: `class-map match-all` は指定したすべての `match` 条件に同時に適合することを要求します。単一パケットが DNS (UDP 53) と HTTP (TCP 80) の両方であることは不可能なため、このクラスには一切マッチせず `class-default` (Drop) に落ちます。

---

## 🔗 参考リソース

* [Cisco Technical Notes: Zone-Based Policy Firewall Design and Application Guide](https://www.cisco.com/)
* [Cisco IOS XE 17.x Security Configuration Guide - Zone-Based Policy Firewall](https://www.cisco.com/)
* [Cisco Command Reference - zone-pair security](https://www.cisco.com/)
* [Cisco Live: Deep Dive into IOS Zone-Based Policy Firewall (BRKSEC-2003)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * ZBFW のトラブルの 9 割は **「`pass` による戻りパケットドロップ」「`self` ゾーン設定後の BGP/OSPF/SSH 遮断」「`match-all` と `match-any` の誤用」「NAT 処理順序と ACL 内 Real IP の誤指定」** に起因します。
* **図解（ZBFW パケット処理フローチャート）**:
  ```text
  [ Packet Ingress ]
          │
          ▼
  [ Existing Session in State Table? ] ─── YES ───► [ PERMIT / PASS ]
          │ NO
          ▼
  [ Source & Destination Zone Defined? ] ─ NO  ───► [ DROP ]
          │ YES
          ▼
  [ Zone-Pair Exists? ] ───────────────── NO  ───► [ DROP ]
          │ YES
          ▼
  [ Evaluate CPL Policy-Map (Top-Down) ]
          ├── Match 'inspect' ──► Create Session Entry & Pass
          ├── Match 'pass'    ──► Pass Only (No Session Entry)
          └── Class-Default   ──► DROP (Implicit Deny)
  ```
* **注意点**:
  * 実機試験で `self` ゾーンを構成する際は、自身のコンソール/SSH 接続が途絶しないよう、最初に `pass` ルールを記述した Self Zone ポリシーを適用してからインターフェイスをゾーンにバインドしてください。
