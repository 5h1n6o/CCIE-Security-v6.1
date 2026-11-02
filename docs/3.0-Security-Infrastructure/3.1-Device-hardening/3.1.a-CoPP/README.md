---
layout: default
title: 3.1.a-CoPP
nav_order: 1
parent: 3.1-Device-hardening
grand_parent: 3.0-Security-Infrastructure
---

# 3.1.a CoPP (Control Plane Policing)

Cisco IOS / IOS-XE ネットワーク機器における **Control Plane Policing (CoPP)** は、ルーターやスイッチのメイン CPU（Route Processor: RP / Control Plane）宛てに送信される制御面トラフィック（Control Plane Traffic）および管理面トラフィック（Management Plane Traffic）に対し、QoS（Modular QoS CLI: MQC）メカニズムを用いてレート制限（Policing）およびフィルタリングを適用する高度なデバイス・ハーデンニング（Device Hardening）技術です。

CCIE Security v6.1 Blueprint **3.1 Device hardening techniques and control plane protection methods** の核となる **3.1.a CoPP** では、コントロールプレーン宛てトラフィックの分類（Classification）、ACL パターン（Permit/Deny の特殊な意味合い）、MQC ポリシー（`class-map`, `policy-map`）、`control-plane` インターフェイスへの適用方法、プロトコル別レート調整（Routing Protocols, Management Protocols, Exception Traffic）、ならびにハードウェア（QFP / ASIC）プレーンでのトラブルシューティング・カウンター分析手法に至るまで、実戦的かつ高度な設計・構築・検証能力が問われます。

---

## 📘 概要

* **機能概要**:
  * **Control Plane の保護**: ルーターの CPU (Route Processor) に送られるトラフィック（Punt Packet）に対し、専用の論理インターフェイス `control-plane` 上で MQC ポリシーを適用し、CPU 枯渇を防ぐ。
  * **トラフィックの分類 (Classification)**: `access-list` や `class-map` を用いて、ルーティングプロトコル（BGP, OSPF, EIGRP）、管理通信（SSH, SNMP, NTP）、インフラ補助（ARP, ICMP, DHCP）、例外処理（IP Options, TTL Exceeded）等を厳密に識別。
  * **レート制限・ドロップ (Policing / Drop)**: 設定された閾値（CIR: Committed Information Rate, Burst Size）を超えるトラフィックを破棄（Drop）またはリマーキング（Set DSCP）することで、DDoS 攻撃や大量パケットによる CPU 100% 飽和を防ぐ。
* **利用目的**:
  * コントロールプレーンを標的とした DoS / DDoS 攻撃（SYN Flood, ICMP Flood, ARP Storm）からの CPU 保護。
  * 悪意あるトラフィックや不要なスキャニングパケットが大量に発生した場合でも、BGP / OSPF などのルーティングピアセッションの維持を保証する（Keepalive 喪失による隣接関係切断の防止）。
  * ネットワーク障害時の例外パケット（TTL 切れ、IP オプション付きパケット、MTU 超過等）による Process Switching 負荷の局所化。
* **どのような場面で利用するか**:
  * エンタープライズコア・アグリゲーションルーター、インターネット WAN エッジルーター、データセンターボーダールーター等、外部から直接制御面へアクセスされ得る全 Cisco IOS / IOS-XE デバイス。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **評価対象技術** | **Control Plane Policing (CoPP)** |
| **適用インターフェイス** | 論理インターフェイス `control-plane` (方向は `input` のみサポート) |
| **記述フレームワーク** | **MQC (Modular QoS CLI)**: `access-list` ➔ `class-map` ➔ `policy-map` ➔ `service-policy input` |
| **ACL `permit`/`deny` の意味** | **`permit`**: クラスにマッチし、ポリシングの評価対象とする。<br>**`deny`**: 当該クラスから除外（スキップ）し、後続の `class-default` で処理させる。※ACL で deny してもパケットはドロップされない点に最深の注意が必要！ |
| **ポリシングアクション** | `conform-action transmit`（許容内は透過）、`exceed-action drop`（超過分をドロップ）、`violate-action drop` |
| **処理アーキテクチャ** | IOS-XE (ASR1000 / ISR4000 等) では QFP (Quantum Flow Processor) ハードウェア microcode レベルで処理され、CPU 転送前に超高速ドロップが実行される。 |
| **対象トラフィック分類** | 1. **Control Plane Traffic**: BGP, OSPF, EIGRP, LDP, RSVP, BFD 等<br>2. **Management Plane Traffic**: SSH, HTTPS, SNMP, NTP, TACACS+/RADIUS 等<br>3. **Data Plane Exception / Sub-Control**: ARP, ICMP, IP Options, TTL Exceeded, Glean 等 |
| **主な制限事項** | 方向は `input`（CPU へ送られるトラフィック）のみ。CPU から送信される `output` トラフィックの制御は非サポート。 |

---

## 🏗 動作原理

### 1. Control Plane 宛てトラフィックパスと CoPP 適用ポイント

ルーターに到着したパケットは、データプレーン（FIB/CEF）で検索されます。宛先がルーター自身の IP アドレスである場合、または Process Switching が必要な例外パケットである場合、パケットは **Punt**（CPU 送信）と判断されます。CoPP は、Punt パケットが CPU（Route Processor）に到達する直前の論理ゲート（`control-plane` インターフェイス）で発動します。

```text
[ Ingress Physical / Logical Interface (e.g. Gi0/0/0) ]
                        │
                        ▼
   ┌─────────────────────────────────────────┐
   │ Data Plane (CEF / FIB Lookup)           │
   └─────────────────────────────────────────┘
        │                               │
        │ (Transit Traffic)             │ (Punt Traffic: Local IP / Exceptions)
        ▼                               ▼
[ Egress Interface ]       ┌────────────────────────────────────────────────────────┐
 (Data Plane Forwarding)   │ Control Plane Logical Interface (`control-plane`)       │
                           │                                                        │
                           │  ┌──────────────────────────────────────────────────┐  │
                           │  │ MQC Service Policy (CoPP)                        │  │
                           │  │                                                  │  │
                           │  │  Class: CRITICAL_ROUTING (BGP/OSPF)            │  │
                           │  │   ├── Conform ──► Transmit to CPU                │  │
                           │  │   └── Exceed  ──► Drop / Limit Rate              │  │
                           │  │                                                  │  │
                           │  │  Class: MANAGEMENT (SSH/SNMP)                    │  │
                           │  │   ├── Conform ──► Transmit to CPU                │  │
                           │  │   └── Exceed  ──► Drop                           │  │
                           │  │                                                  │  │
                           │  │  Class: EXCEPTION_TRAFFIC (ICMP/TTL-0)           │  │
                           │  │   ├── Conform ──► Transmit to CPU                │  │
                           │  │   └── Exceed  ──► Drop                           │  │
                           │  │                                                  │  │
                           │  │  Class: class-default                            │  │
                           │  │   └── Default Rate Limit / Drop                  │  │
                           │  └──────────────────────────────────────────────────┘  │
                           └────────────────────────────────────────────────────────┘
                                                        │
                                                        ▼ (Policed & Permitted Packets)
                                           ┌─────────────────────────┐
                                           │ Route Processor (CPU)   │
                                           │ - BGP/OSPF Process      │
                                           │ - SSH/SNMP Daemon       │
                                           │ - ICMP Generator        │
                                           └─────────────────────────┘
```

---

## ⚙ 動作シーケンス

1. **パケットの受信と CEF/FIB 照合**:
   * 物理インターフェイスでパケットを受信後、L1/L2 ヘッダーを剥ぎ取り L3 ヘッダー（IP）を解析。
   * CEF テーブル（FIB）を検索。パケットが「自機器の IP アドレス宛（Local Delivery）」、または「IP オプション付き」、「TTL=1/0 (TTL Exceeded)」、「ARP 未解決 (Glean)」、「CEF 非対応プロトコル」である場合、CEF 判定制御により **Punt** フラグが立てられる。
2. **Control Plane インターフェイスへのリダイレクト**:
   * Punt パケットはデータプレーンのフォワーディングパスから外され、CPU 宛ての内部キュー（Punt Path）を経由して論理 `control-plane` インターフェイスへ流れる。
3. **MQC クラス・マップによるトラフィック分類 (Class-Map Matching)**:
   * `control-plane` にバインドされた `service-policy input` 内の `class-map` が上から順に評価される。
   * クラスマップ内で参照されている `access-list` を評価：
     * **`permit` にマッチ**: パケットがそのクラスに該当したと判定し、対応する `policy-map` 内のポリシングアクションへ進む。
     * **`deny` にマッチ**: パケットはそのクラスに「マッチしなかった」とみなされ、次の `class-map`（最終的には `class-default`）の評価へ回される。※パケットの破棄ではない！
4. **ポリシング処理 (Police Action) の実行**:
   * マッチしたクラスの `police` 命令（Token Bucket アルゴリズム）に基づいて、トラフィックレートを計測。
   * **Conform (許容範囲内)**: `conform-action transmit` に基づき、パケットを CPU の該当プロセス（BGP、SSH、ICMP 等）の入力キューに渡す。
   * **Exceed / Violate (閾値超過)**: `exceed-action drop`（または `set-dscp-transmit`）に基づき、CPU に到達させることなく**その場で即座にパケットを破棄**する。
5. **CPU でのパケット処理**:
   * ポリシングを通過した健全なパケットのみが CPU ソフトウェアプロセスへ引き渡され、正常に安全処理される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **CoPP における Access-List `permit` / `deny` の「超厳格なルール」**:
   * **一般的なアクセスリスト**: `permit` = 通過、`deny` = 遮断（ドロップ）。
   * **CoPP でのアクセスリスト**: `permit` = **「この Class-Map の対象として抽出する」**、`deny` = **「この Class-Map の対象外とし、次の Class-Map (または `class-default`) に評価を譲る」**。
   * ❌ **最悪のミス例**: BGP を保護しようとして `access-list 100 deny tcp any any eq 179` と記述した場合、BGP パケットはドロップされるのではなく、ポリシングの入っていない `class-default` に落ちてしまい、最悪の場合ポリシング対象から外れるか、`class-default` の低帯域制限に巻き込まれて BGP セッションが切断されます。
2. **クリティカルプロトコルの個別クラス分離**:
   * CCIE ラボ試験では、「ICMP トラフィックを 8 kbps に制限しつつ、BGP および OSPF トラフィックを一切ドロップさせるな」、「SSH 管理通信は特定ホストからのみ許可し、それ以外の管理通信はレート制限せよ」といった複合要件が出題されます。
   * 以下のカテゴリごとに明確にクラス分けを実施します：
     * **Class 1 (Routing Protocols)**: BGP (TCP 179), OSPF (IP Prot 89), EIGRP (IP Prot 88), LDP (TCP/UDP 646), BFD (UDP 3784)
     * **Class 2 (Management Protocols)**: SSH (TCP 22), SNMP (UDP 161), NTP (UDP 123), TACACS+ (TCP 49), RADIUS (UDP 1812/1813)
     * **Class 3 (Infrastructure / Helper)**: ARP, ICMP, DHCP (UDP 67/68), DNS (UDP/TCP 53)
     * **Class 4 (Undesirable / Exception)**: IP Options, TTL Exceeded, Invalid packets
3. **`class-default` の扱い**:
   * 明示的に定義しなかったすべての Punt パケット（未定義のユニキャスト/マルチキャスト、ICMP、未知のプロトコル等）は `class-default` に分類されます。
   * `class-default` に対しても適切なレートリミット（例: `police rate 8000 pps exceed-action drop` または `police 64000 conform-action transmit exceed-action drop`）を設定しておくことが、未知の DoS 攻撃を防ぐ要件となります。
4. **IOS-XE (QFP) における CoPP 表示・検証の特殊コマンド**:
   * IOS-XE (ASR1000, ISR4000, Catalyst 9000 等) では、CoPP はデータプレーンである Quantum Flow Processor (QFP) 上で動作します。
   * 通常の `show policy-map control-plane` に加え、QFP ハードウェアレベルのリアルタイムドロップカウンターを確認する **`show platform hardware qfp active feature copp client`** コマンドの理解が必須です。

---

## 🛠 設定方法

### 1. Cisco IOS / IOS-XE: 包括的 Control Plane Policing (CoPP) 完全設定例

以下は、CCIE Security ラボ試験で求められる要件（ルーティングプロトコル優先保護、管理プロトコル制限、インフラストラクチャ保護、例外トラフィック制限）を満たす標準コンフィグレーションです。

```bash
! ===== Step 1: トラフィック分類用 Extended Access-List の定義 =====

! 1. ルーティングプロトコル識別用 ACL
ip access-list extended ACL_COPP_ROUTING
 permit tcp any any eq bgp
 permit tcp any eq bgp any
 permit ospf any any
 permit eigrp any any
 permit udp any any eq 3784                ! BFD
 permit tcp any any eq LDP
 permit udp any any eq LDP

! 2. 許可された管理端末からの SSH / SNMP 識別用 ACL
ip access-list extended ACL_COPP_MGMT_ALLOWED
 permit tcp 10.1.100.0 0.0.0.255 any eq ssh
 permit udp 10.1.100.0 0.0.0.255 any eq snmp
 permit udp any any eq ntp

! 3. ICMP / インフラ用 ACL
ip access-list extended ACL_COPP_INFRA
 permit icmp any any
 permit udp any eq bootps any eq bootpc   ! DHCP

! 4. 例外トラフィック (IP Options, TTL 切れ等) 用 ACL
ip access-list extended ACL_COPP_EXCEPTIONS
 permit ip any any option any

! ===== Step 2: Class-Map の定義 =====

class-map match-all CM_COPP_ROUTING
 match access-group name ACL_COPP_ROUTING

class-map match-all CM_COPP_MGMT_ALLOWED
 match access-group name ACL_COPP_MGMT_ALLOWED

class-map match-all CM_COPP_INFRA
 match access-group name ACL_COPP_INFRA

class-map match-all CM_COPP_EXCEPTIONS
 match access-group name ACL_COPP_EXCEPTIONS

! ===== Step 3: Policy-Map の定義とポリシングアクションのバインド =====

policy-map PM_COPP_POLICY
 ! ルーティングプロトコル: 高帯域確保 (1 Mbps), 超過分はドロップ
 class CM_COPP_ROUTING
  police 1000000 bc 31250
   conform-action transmit
   exceed-action drop

 ! 許可管理通信: 帯域確保 (512 kbps)
 class CM_COPP_MGMT_ALLOWED
  police 512000 bc 16000
   conform-action transmit
   exceed-action drop

 ! ICMP / DHCP インフラ通信: 厳格な制限 (64 kbps)
 class CM_COPP_INFRA
  police 64000 bc 2000
   conform-action transmit
   exceed-action drop

 ! 例外トラフィック (IP Options 等): 極小制限 (32 kbps)
 class CM_COPP_EXCEPTIONS
  police 32000 bc 1000
   conform-action transmit
   exceed-action drop

 ! 未定義の全 Punt トラフィック (`class-default`): 基本保護 (128 kbps)
 class class-default
  police 128000 bc 4000
   conform-action transmit
   exceed-action drop

! ===== Step 4: `control-plane` インターフェイスへの適用 =====

control-plane
 service-policy input PM_COPP_POLICY
```

---

## 🔍 検証コマンド

| 目的 | コマンド (IOS / IOS-XE CLI) | 表示確認ポイント |
| :--- | :--- | :--- |
| **CoPP 全体およびクラス別統計確認** | `show policy-map control-plane` | 各クラスのマッチパケット数、`conformed` / `exceeded` バイト数、ドロップ件数 |
| **特定クラスに絞った CoPP 統計確認** | `show policy-map control-plane input class <CLASS_NAME>` | 特定クラス（例: `CM_COPP_ROUTING`）でのドロップ発生有無のピンポイント確認 |
| **Control Plane インターフェイス設定確認** | `show running-config section control-plane` | `control-plane` 下に `service-policy input <POLICY_NAME>` が正しくバインドされているか |
| **IOS-XE (QFP) ハードウェア側ドロップ統計** | `show platform hardware qfp active feature copp client` | QFP ハードウェア ASIC レベルで CoPP によりドロップされたパケット統計 |
| **Punt パケットの原因別詳細表示 (IOS-XE)** | `show platform hardware qfp active infrastructure punt statistics` | なぜパケットが CPU へ送られたか（Punt 理由: ARP, BGP, ICMP, IP Options 等）の統計 |
| **CPU 使用率とプロセス確認** | `show process cpu sorted | exclude 0.00` | CoPP 適用前後での CPU 負荷軽減効果、`IP Input` プロセスの負荷割合 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **CoPP 適用後、BGP や OSPF のフラッピング（セッション切断）が頻発する** | ルーティングプロトコルの CIR (Committed Information Rate) や Burst Size (`bc`) の設定値が低すぎるか、`class-default` に誤って分類されて破棄されている | `show policy-map control-plane input class CM_COPP_ROUTING`<br>`show ip bgp summary` | BGP/OSPF クラスの `police` レートを増大させるか、`bc` パラメータを拡大する。また、ACL で BGP/OSPF が `permit` で正しくキャッチされているか再点検する。 |
| **管理端末からの SSH 接続が頻繁にタイムアウトまたは拒否される** | SSH トラフィックのポリサー上限を超過したか、非許可 IP からの SSH が `class-default` でドロップされている | `show policy-map control-plane input class CM_COPP_MGMT` | 管理用 ACL に送信元 IP を正しく追加するか、`police` 帯域幅の上限（Rate）を引き上げる。 |
| **ACL で `deny` を記述したのに、パケットがドロップされず CPU に到達する** | CoPP における ACL の `deny` は「ドロップ」ではなく「ポリシング非対象（`class-default` へ転送）」を意味する誤解 | `show access-list`<br>`show policy-map control-plane` | CoPP ポリシーでドロップさせたいトラフィックは、ACL で **`permit`** 記述し、対応する `policy-map` 内で **`exceed-action drop`** を設定する。 |
| **`show policy-map control-plane` のドロップカウンターが増加しないがパケットが落とされる** | IOS-XE でハードウェア (QFP) 側でポリシングドロップが発生しているが、ソフトウェアのカウンター同期が遅延している | `show platform hardware qfp active feature copp client` | QFP ハードウェアコマンドを用いてデータプレーン層でのドロップ状況を直接検証する。 |

---

## ⚠ 制限事項

* **適用方向の制限**:
  * CoPP は方向として **`input`（ルーターの CPU へ向かうトラフィック）のみサポート** します。CPU から外部へ送信される `output` トラフィックに対する CoPP ポリシーの適用は不可能です。
* **出力アクションの制限**:
  * CoPP 内の `policy-map` では、一般的な QoS で使用される `queue-limit`、`priority`、`bandwidth`（シェーピング/キュートリー）は使用できず、**`police`（ポリシング）のみ** がサポートされます。
* **マルチキャスト・ブロードキャストの扱い**:
  * コントロールプレーンへ向かうマルチキャスト/ブロードキャストパケット（OSPF Hello や ARP 等）は全ラインカードから集約されるため、バーストサイズ (`bc`) を過小に設定すると正常な隣接関係の維持に支障をきたします。

---

## 🔄 他技術との関連

* **Control Plane Protection (CPPr)**: CoPP の拡張機能。`control-plane` を 3 つのサブインターフェイス（`host`, `transit`, `cef-exception`）に細分化してポリシングを適用する。
* **Infrastructure ACL (iACL)**: 物理インターフェイスの入口（`interface Gi0/0` の `ip access-group in`）に適用し、ネットワークインフラ機器の IP アドレス（Loopback や P2P サブネット）宛ての不正パケットをデータプレーンの最前線でドロップする。
* **Management Plane Protection (MPP)**: 管理プロトコル（SSH, SNMP, HTTPS 等）を受信できる「物理インターフェイス」を限定する機能。
* **Unicast Reverse Path Forwarding (uRPF)**: 送信元 IP アドレスの偽装（Spoofing）パケットをデータプレーンで早期破棄し、CoPP や CPU への負荷到達を防止する。

---

## 🧩 比較表

| 比較項目 | Control Plane Policing (CoPP) | Control Plane Protection (CPPr) | Infrastructure ACL (iACL) | Management Plane Protection (MPP) |
| :--- | :--- | :--- | :--- | :--- |
| **保護対象** | Control / Management / Exception | Control / Management / Exception (個別) | 機器の内部 IP アドレス全般 | 管理プレーン (SSH/SNMP/HTTPS) |
| **適用場所** | `control-plane` | `control-plane host / transit / cef-exception` | 物理/論理データインターフェイス | `control-plane host` (MPP モード) |
| **制御方式** | MQC Rate Limiting (`police`) | MQC Rate Limiting (`police`) | Explicit Packet Drop (`access-group`) | Interface/Protocol Whitelisting |
| **粒度** | ルーター全体で一括分類 | 宛先タイプ（自機/通過/例外）別 | L3/L4 5-Tuple フィルタリング | インターフェイス×プロトコル限定 |
| **設定の複雑さ**| 中程度 | 高い (IOS のみ対応、IOS-XE 非推奨) | 低〜中 | 非常に低い (シンプル) |

---

## 💡 ベストプラクティス

1. **`class-default` に対する明示的なポリシングの適用**:
   * 未分類の Punt トラフィックによる DoS 攻撃を防ぐため、`class-default` にも必ず必要最小限の `police`（例: 128 kbps 〜 512 kbps, `exceed-action drop`）を設定します。
2. **バーストサイズ (`bc`) の適切な設定**:
   * ルーティングプロトコル（BGP の大量更新や OSPF Hello バースト）では、単なる CIR（bps）だけでなく、十分な **`bc` (Committed Burst)** を確保しないと、バースト発生時に正常なパケットが誤ってドロップされます。
3. **段階的な導入とログモニタリング**:
   * 初期構築時は `exceed-action drop` の代わりに一時的に `exceed-action transmit`（またはポリサーカウンターの観察のみ）でデプロイし、`show policy-map control-plane` でドロップが発生しないか数日間観察した後に `drop` へ切り替えます。

## 📝 ラボ学習・設定サンプル例

### 例1: 基本的な CoPP 設定 (ICMP 制限と Telnet 破棄)

#### 問題
ルーター R1 の CPU を保護するため、CoPP を構成せよ。ルーター宛ての ICMP トラフィックを 64 kbps に制限し、Telnet トラフィック（TCP Port 23）は一切 CPU に到達させず即座にドロップさせよ。その他のトラフィックはポリシングの影響を受けないように処理せよ。

#### 要件
* Policy-Map 名: `PM_COPP_BASIC`
* Class-Map 名: `CM_ICMP`, `CM_TELNET`
* ICMP 帯域制限: 64,000 bps (`bc 2000`), `exceed-action drop`
* Telnet: レート 8000 bps, `conform-action drop`, `exceed-action drop`
* `control-plane` に `input` 適用すること。

#### 設定例

```bash
! ===== ACL 定義 =====
ip access-list extended ACL_ICMP
 permit icmp any any

ip access-list extended ACL_TELNET
 permit tcp any any eq telnet

! ===== Class-Map 定義 =====
class-map match-all CM_ICMP
 match access-group name ACL_ICMP

class-map match-all CM_TELNET
 match access-group name ACL_TELNET

! ===== Policy-Map 定義 =====
policy-map PM_COPP_BASIC
 class CM_TELNET
  police 8000
   conform-action drop
   exceed-action drop

 class CM_ICMP
  police 64000 bc 2000
   conform-action transmit
   exceed-action drop

! ===== Control-Plane 適用 =====
control-plane
 service-policy input PM_COPP_BASIC
```

#### 検証方法
1. 対面機器から R1 宛てに Telnet 接続を試行し、接続が即座にドロップされることを確認。
2. `ping <R1_IP> repeat 1000 size 1400` を実行し、大量 Ping 送信時に一部パケットがドロップされることを確認。
3. R1 上で `show policy-map control-plane input class CM_TELNET` および `show policy-map control-plane input class CM_ICMP` を実行し、`drop` カウンターの増加を確認。

---

### 例2: BGP ＆ OSPF ルーティングプロトコルの優先保護設定

#### 問題
R1 上で動作する BGP および OSPF ルーティングセッションの維持を最優先するため、BGP (TCP 179) と OSPF (IP Protocol 89) を含む専用クラス `CM_ROUTING` を作成し、1 Mbps の専用帯域と十分なバーストサイズ（`bc 31250`）を確保せよ。また未定義トラフィック (`class-default`) は 128 kbps に制限せよ。

#### 要件
* ルーティングプロトコルのパケット損失によるセッションフラップを防止すること。

#### 設定例

```bash
ip access-list extended ACL_ROUTING
 permit tcp any any eq bgp
 permit tcp any eq bgp any
 permit ospf any any

class-map match-all CM_ROUTING
 match access-group name ACL_ROUTING

policy-map PM_COPP_ROUTING_PROTECT
 class CM_ROUTING
  police 1000000 bc 31250
   conform-action transmit
   exceed-action drop

 class class-default
  police 128000 bc 4000
   conform-action transmit
   exceed-action drop

control-plane
 service-policy input PM_COPP_ROUTING_PROTECT
```

#### 検証方法
* `show policy-map control-plane input class CM_ROUTING` を実行し、BGP キープアライブや OSPF Hello パケットが `conformed` として正常に処理され、ドロップが発生していないことを確認。

---

### 例3: 管理通信 (SSH / SNMP) の送信元 IP ホワイトリスト型 CoPP

#### 問題
R1 に対する SSH (TCP 22) および SNMP (UDP 161) アクセスを、管理サブネット `192.168.10.0/24` からの通信のみ 512 kbps で許可せよ。それ以外の送信元からの SSH / SNMP トラフィックは一切 CPU に到達させずドロップせよ。

#### 要件
* 許可された管理送信元: `192.168.10.0/24`
* 違反アクセスは明示的なドロップクラスで即時遮断すること。

#### 設定例

```bash
! 許可管理送信元用 ACL
ip access-list extended ACL_MGMT_ALLOW
 permit tcp 192.168.10.0 0.0.0.255 any eq ssh
 permit udp 192.168.10.0 0.0.0.255 any eq snmp

! 非許可 SSH/SNMP 攻撃検知用 ACL
ip access-list extended ACL_MGMT_BLOCK
 permit tcp any any eq ssh
 permit udp any any eq snmp

class-map match-all CM_MGMT_ALLOW
 match access-group name ACL_MGMT_ALLOW

class-map match-all CM_MGMT_BLOCK
 match access-group name ACL_MGMT_BLOCK

policy-map PM_COPP_MGMT_STRICT
 class CM_MGMT_ALLOW
  police 512000 bc 16000
   conform-action transmit
   exceed-action drop

 class CM_MGMT_BLOCK
  police 8000
   conform-action drop
   exceed-action drop

control-plane
 service-policy input PM_COPP_MGMT_STRICT
```

#### 検証方法
1. `192.168.10.5` から R1 へ SSH を行い、正常にログインできることを確認。
2. `172.16.1.5` から R1 へ SSH を試行し、即座に接続がドロップされることを確認。
3. `show policy-map control-plane input class CM_MGMT_BLOCK` で `drop` パケット数が増加していることを検証。

---

### 例4: BFD ＆ LDP インフラストラクチャプロトコルの超低遅延ポリシング

#### 問題
MPLS 網のエッジルーター R1 において、LDP (TCP/UDP 646) および BFD (UDP 3784) トラフィックのパケット破棄を防ぐため、CoPP 内で専用クラス `CM_MPLS_INFRA` を定義し、2 Mbps の帯域を確保せよ。

#### 要件
* BFD パケットのドロップは Fast Reroute / LFA の誤発動を引き起こすため、高い CIR をアサインすること。

#### 設定例

```bash
ip access-list extended ACL_MPLS_INFRA
 permit tcp any any eq 646
 permit udp any any eq 646
 permit udp any any eq 3784

class-map match-all CM_MPLS_INFRA
 match access-group name ACL_MPLS_INFRA

policy-map PM_COPP_MPLS
 class CM_MPLS_INFRA
  police 2000000 bc 62500
   conform-action transmit
   exceed-action drop

control-plane
 service-policy input PM_COPP_MPLS
```

#### 検証方法
* `show bfd neighbors` で BFD セッションが Up 状態を維持していることを確認。
* `show policy-map control-plane input class CM_MPLS_INFRA` で `conformed` バイト数を確認。

---

### 例5: IP Options ＆ TTL Exceeded 例外パケットの保護・制限

#### 問題
Traceroute 攻撃や IP オプション付きパケットの大量送信による CPU 枯渇を防ぐため、IP Options パケットおよび TTL 切れ（TTL Exceeded）パケットをキャッチするクラス `CM_EXCEPTIONS` を作成し、32 kbps にレート制限せよ。

#### 要件
* `ip access-list` の `option` フラグおよび `ttl` 機能を活用すること。

#### 設定例

```bash
ip access-list extended ACL_EXCEPTIONS
 permit ip any any option any
 permit ip any any ttl eq 1

class-map match-all CM_EXCEPTIONS
 match access-group name ACL_EXCEPTIONS

policy-map PM_COPP_EXCEPTIONS
 class CM_EXCEPTIONS
  police 32000 bc 1000
   conform-action transmit
   exceed-action drop

control-plane
 service-policy input PM_COPP_EXCEPTIONS
```

#### 検証方法
* 隣接ルーターから `traceroute <R1_IP>` を大量実行し、32 kbps を超えた分の ICMP Time Exceeded 生成パケットがポリサーによりドロップされることを確認。

---

### 例6: Single-Rate Three-Color (srTCM) ポリシングによる段階的制御

#### 問題
ルーター R1 宛ての NTP 通信 (UDP 123) に対し、シングルレート・3 カラーポリサー (srTCM) を構成せよ。Cir 128 kbps, Normal Burst (`bc`) 4000 bytes, Excess Burst (`be`) 8000 bytes とし、Conform 時は transmit、Exceed 時は DSCP を `CS1` にリマーキングして transmit、Violate 時は drop とせよ。

#### 要件
* `police cir 128000 bc 4000 be 8000` コマンドを使用すること。

#### 設定例

```bash
ip access-list extended ACL_NTP
 permit udp any any eq ntp

class-map match-all CM_NTP
 match access-group name ACL_NTP

policy-map PM_COPP_SRTCM
 class CM_NTP
  police cir 128000 bc 4000 be 8000
   conform-action transmit
   exceed-action set-dscp-transmit cs1
   violate-action drop

control-plane
 service-policy input PM_COPP_SRTCM
```

#### 検証方法
* `show policy-map control-plane input class CM_NTP` を実行し、`conformed`, `exceeded (set-dscp)`, `violated (drop)` の各カウンターが条件に応じて増分することを確認。

---

### 例7: CoPP における ACL `deny` 動作の検証シナリオ

#### 問題
以下のコンフィグレーションにおいて、`10.1.1.1` から R1 宛ての Ping が送信された場合のパケットの挙動を説明し、意図通り 64 kbps で制限されるように修正せよ。

```bash
! 誤った設定例
ip access-list extended ACL_TEST
 deny icmp host 10.1.1.1 any
 permit icmp any any

class-map match-all CM_ICMP_TEST
 match access-group name ACL_TEST

policy-map PM_TEST
 class CM_ICMP_TEST
  police 64000
   conform-action transmit
   exceed-action drop

control-plane
 service-policy input PM_TEST
```

#### 要件
* `10.1.1.1` からの ICMP パケットが `deny` により `class-default` へ通過してしまう問題を解説し、適切な ACL に修正すること。

#### 設定例

```bash
! 修正後の正しいコンフィグ
ip access-list extended ACL_TEST_FIXED
 permit icmp any any                   ! すべての ICMP を CM_ICMP_TEST クラスへ一致させる

class-map match-all CM_ICMP_TEST
 match access-group name ACL_TEST_FIXED

policy-map PM_TEST
 class CM_ICMP_TEST
  police 64000 bc 2000
   conform-action transmit
   exceed-action drop

control-plane
 service-policy input PM_TEST
```

#### 検証方法
* `10.1.1.1` から Ping を送信し、`show policy-map control-plane input class CM_ICMP_TEST` にてマッチカウンターが増加し、64 kbps 超過分が正しくドロップされることを検証。

---

### 例8: ARP フラッディング攻撃からの CPU 保護 CoPP

#### 問題
スイッチ/ルーター R1 に対する ARP スプーフィングおよび大量 ARP リクエスト攻撃を防ぐため、ARP パケット（Punt 理由: ARP）をキャッチし、100 pps (Packets Per Second) に制限する CoPP を構築せよ。

#### 要件
* PPS (Packets Per Second) 単位でのポリシング（IOS-XE 対応機能）を使用すること。

#### 設定例

```bash
! ARP パケットを識別する Class-Map
class-map match-any CM_ARP
 match protocol arp

policy-map PM_COPP_ARP_PROTECT
 class CM_ARP
  police rate 100 pps
   conform-action transmit
   exceed-action drop

control-plane
 service-policy input PM_COPP_ARP_PROTECT
```

#### 検証方法
* 擬似的な ARP スキャンツール等で大量の ARP リクエストを発生させ、`show policy-map control-plane input class CM_ARP` で 100 pps 超過パケットが破棄されることを確認。

---

### 例9: CoPP による Control Plane トラフィックの DSCP 優先度付け

#### 問題
R1 の `control-plane` に着信する BGP トラフィックに対し、ポリシング適用（2 Mbps 制限）と同時に、CPU 内部キューでの最優先処理のために DSCP 値を `CS6` に強制リマーキングせよ。

#### 要件
* `set dscp cs6` または `conform-action set-dscp-transmit cs6` を使用すること。

#### 設定例

```bash
ip access-list extended ACL_BGP
 permit tcp any any eq bgp
 permit tcp any eq bgp any

class-map match-all CM_BGP
 match access-group name ACL_BGP

policy-map PM_COPP_MARKING
 class CM_BGP
  police 2000000 bc 62500
   conform-action set-dscp-transmit cs6
   exceed-action drop

control-plane
 service-policy input PM_COPP_MARKING
```

#### 検証方法
* BGP セッション確立時、`show policy-map control-plane input class CM_BGP` を実行し、`conformed set dscp cs6` カウンターの増加を確認。

---

### 例10: IOS-XE (QFP) ハードウェアレベルでの CoPP ドロップカウンター分析

#### 問題
IOS-XE ルーター (ASR1000 / ISR4000) 上で動作する CoPP ポリシーにおいて、ソフトウェアカウンターと QFP ハードウェア ASIC カウンターの双方から、現在発生している CoPP ドロップ原因を分析・特定せよ。

#### 要件
* `show platform hardware qfp` 関連コマンドを網羅して検証すること。

#### 設定例

```bash
! 診断用確認コマンドの実行例
R1# show policy-map control-plane
R1# show platform hardware qfp active feature copp client
R1# show platform hardware qfp active infrastructure punt statistics
```

#### 検証方法
1. `show platform hardware qfp active feature copp client` の出力で、`CoPP Drop Packets` および `CoPP Drop Bytes` の数値を記録。
2. `show platform hardware qfp active infrastructure punt statistics` を実行し、どの Punt 原因（例: `PUNT_CAUSE_IP_OPTIONS`, `PUNT_CAUSE_ICMP_REDIRECT`）でドロップが発生しているかを一致照合する。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 以下のコンフィグレーションが設定された Cisco IOS ルーターにおいて、`10.200.1.5` からルーターの Loopback0 (`172.16.0.1`) 宛てに送信された ICMP Echo Request パケットに対する動作として最も適切なものはどれか。

```bash
ip access-list extended ACL_COPP_CHECK
 deny icmp host 10.200.1.5 any
 permit icmp any any

class-map match-all CM_COPP_ICMP
 match access-group name ACL_COPP_CHECK

policy-map PM_COPP_TEST
 class CM_COPP_ICMP
  police 8000
   conform-action transmit
   exceed-action drop

control-plane
 service-policy input PM_COPP_TEST
```

* A) `10.200.1.5` からの ICMP パケットは ACL の `deny` により即座にドロップされる。
* B) `10.200.1.5` からの ICMP パケットは `CM_COPP_ICMP` クラスにマッチせず、ポリシング（8 kbps 制限）の対象から外れて `class-default` として処理されるため、ドロップされずに CPU に到達する（`class-default` の条件による）。
* C) `10.200.1.5` からの ICMP パケットは 8 kbps のポリサーで処理される。
* D) CoPP は Loopback インターフェイス宛てのトラフィックには作用しない。

**正解**: **B**  
**解説**: CoPP 内で参照される ACL において、`deny` ステートメントは「パケットの破棄」ではなく「この Class-Map への不一致（ミスマッチ）」を意味します。したがって、`10.200.1.5` からの ICMP は `CM_COPP_ICMP` にはマッチせず、後続の `class-default` に送られます。ドロップさせたいパケットは ACL で **`permit`** 記述し、Policy-Map の **`exceed-action drop`** または **`conform-action drop`** で破棄する必要があります。

---

### 2. トラブルシュート問題

**問題**: ASR1002-X (IOS-XE) ルーターにおいて、CoPP ポリシーを導入した直後から、他拠点の BGP ネイバーとの間で 3 分ごとに BGP セッションが切断・再確立を繰り返す（Flapping）現象が発生した。`show policy-map control-plane` を確認したところ、BGP クラスでの `exceeded drop` カウンターは 0 であった。考えられる原因として最も適切なものはどれか。

* A) BGP パケットが ACL で `deny` されており、ポリシング無制限の `class-default` に分類されているため。
* B) BGP パケットのバーストにより、QFP ハードウェアレベル（データプレーン）でドロップが発生しているが、IOS ソフトウェア側の `show policy-map control-plane` カウンターへの同期が遅れている。
* C) CoPP は BGP プロトコルをサポートしていないため。
* D) BGP の Port 179 が CoPP によってブロックされているため。

**正解**: **B**  
**解説**: IOS-XE アーキテクチャでは、CoPP ポリシングは QFP (Quantum Flow Processor) ハードウェア側でマイクロコード処理されます。バーストパケット（BGP のルート更新等）が発生した際、QFP レベルで瞬間的にドロップが発生して BGP Keepalive が失われても、ソフトウェア側の `show policy-map` カウンターに即座に反映されない場合があります。この場合、`show platform hardware qfp active feature copp client` コマンドでハードウェアドロップを確認する必要があります。

---

### 3. Design 問題

**問題**: 大規模エンタープライズのコア統合ルーターにおいて、コントロールプレーン（CPU）の保護設計を行う。BGP / OSPF などのルーティングプロトコル、SSH / SNMP などの管理通信、および ICMP や例外パケットが混在する環境において、最も推奨される CoPP 設計方針はどれか。

* A) 単一の `class-default` に対して 1 Mbps のポリサーを設定し、すべての Punt パケットを一括制限する。
* B) プロトコルの重要度（Routing > Management > Infrastructure > Exceptions）に応じた複数の専用 Class-Map を作成し、BGP/OSPF には十分な CIR と大きなバーストサイズ (`bc`) を与え、ICMP や例外パケットには厳格な低レート制限を適用する。
* C) すべての Punt パケットを ACL の `deny` で除外し、ポリシングをバイパスさせる。
* D) CoPP の代わりにすべての物理インターフェイスに Ingress ACL を適用して代替する。

**正解**: **B**  
**解説**: CoPP の設計ベストプラクティスは、トラフィックの役割（ルーティング、管理、インフラ補助、例外）ごとにクラスを明確に分離し、クリティカルなルーティング通信には十分な帯域と大容量のバーストサイズ (`bc`) を確保することでセッション切断を防ぎ、攻撃に利用されやすい ICMP や IP Options 等には厳格なポリシングを課すことです。

---

### 4. 実装問題

**問題**: Cisco IOS ルーターにおいて、CoPP ポリシー `PM_COPP` を有効化するための正しい CLI コマンド手順はどれか。

* A) `interface control-plane` ➔ `service-policy input PM_COPP`
* B) `control-plane` ➔ `service-policy input PM_COPP`
* C) `global` ➔ `copp service-policy PM_COPP`
* D) `interface GigabitEthernet0/0` ➔ `service-policy control-plane PM_COPP`

**正解**: **B**  
**解説**: CoPP をアクティベートするには、グローバルコンフィグレーションモードから **`control-plane`** コマンドで Control-Plane サブモードに入り、**`service-policy input <POLICY_NAME>`** を適用します（方向は `input` のみ指定可能）。

---

### 5. コンフィグ読解問題

**問題**: 以下の Policy-Map 設定において、`CM_ICMP` クラスに到達した ICMP トラフィックが 128 kbps を超えた場合の動作に関する説明として正しいものはどれか。

```bash
policy-map PM_COPP_POLICY
 class CM_ICMP
  police 128000 bc 4000
   conform-action transmit
   exceed-action drop
```

* A) 128 kbps 以下の ICMP パケットは CPU へ転送され、128 kbps を超えた過剰なパケットはその場でドロップされる。
* B) 128 kbps を超えたパケットは DSCP `CS1` に書き換えられて CPU へ転送される。
* C) ICMP トラフィックは一切 CPU へ転送されずすべてドロップされる。
* D) 128 kbps を超えたパケットは `class-default` へ転送される。

**正解**: **A**  
**解説**: `police 128000` 設定により、Conform（128 kbps 以下）のトラフィックは `conform-action transmit` に従って CPU（Route Processor）へ透過転送され、Exceed（128 kbps 超過分）のトラフィックは `exceed-action drop` に従って即座に破棄されます。

---

## 🔗 参考リソース

* [Cisco IOS Quality of Service Solutions Configuration Guide: Control Plane Policing (CoPP)](https://www.cisco.com/)
* [Cisco Technical Notes: Control Plane Policing (CoPP) Configuration Guide and Best Practices](https://www.cisco.com/)
* [Cisco Technical Notes: Troubleshoot Control Plane Policing (CoPP) on Cisco IOS-XE Routers](https://www.cisco.com/)
* [Cisco Live: Control Plane Hardening and Protection Architecture (BRKSEC-2003)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で CoPP が要求された場合の構成確認チェックリスト：
    1. **ACL の `permit` 記述**: ポリシング・分類対象とするプロトコルは ACL で必ず `permit` で記述すること（`deny` にすると `class-default` へ落ちる）。
    2. **BGP / OSPF のバースト保証**: `bc` (Committed Burst) パラメータを小さくしすぎないこと（最小でも `bc 16000` 〜 `31250` 以上を推奨）。
    3. **`control-plane` への適用方向**: 必ず `service-policy input <POLICY_NAME>` とすること（`output` は無効エラーとなる）。
    4. **検証コマンド**: `show policy-map control-plane` および `show platform hardware qfp active feature copp client` (IOS-XE) を用いてカウンターの推移を自ら証明すること。
* **図解（CoPP 評価アルゴリズムフローチャート）**:
  ```text
  [ Ingress Packet to CPU (Punt) ]
                 │
                 ▼
     [ Match MQC Class-Map? ]
      ├── Class 1 (BGP/OSPF)  ──► [ Police 1 Mbps ] ──► Conform? ──► Transmit to CPU
      │                                                └── Exceed?  ──► DROP
      │
      ├── Class 2 (SSH/SNMP)  ──► [ Police 512k ] ──► Conform? ──► Transmit to CPU
      │                                                └── Exceed?  ──► DROP
      │
      ├── Class 3 (ICMP/ARP)  ──► [ Police 64k ]  ──► Conform? ──► Transmit to CPU
      │                                                └── Exceed?  ──► DROP
      │
      └── Unmatched (Default) ──► [ Class-Default ] ──► Police 128k ──► Transmit / DROP
  ```
* **注意点**:
  * CoPP の設定変更を行う際、誤って全トラフィックをドロップするポリシーを適用すると、自らの SSH セッションやルーティングが即座に切断されます。ラボ試験では `reload in 5` 等のロールバック設定を行ってから検証することを強く推奨します。


## 📝 **補足（Notes）**

*   **学習メモ**: 「CoPP はルータのボディガード」です。すべてのパケットを調べるのではなく、重要なパケットを「通し」、怪しいパケットを「制限」します。
*   **注意点**: ラボ試験で `police` コマンドを入れる際、単位が `bps` か `pps` かを慎重に確認してください。指定と異なると大幅にパケットがドロップされます。
