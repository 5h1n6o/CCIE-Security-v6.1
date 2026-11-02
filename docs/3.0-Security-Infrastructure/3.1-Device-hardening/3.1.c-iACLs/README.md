---
layout: default
title: 3.1.c-iACLs
nav_order: 3
parent: 3.1-Device-hardening
grand_parent: 3.0-Security-Infrastructure
---

# 3.1.c iACLs

インフラストラクチャ・アクセスコントロールリスト（**iACLs: Infrastructure Access Control Lists**）は、ネットワークインフラストラクチャを構成するルーター、スイッチ、ファイアウォール自身の IP アドレス空間（ルーターのループバックインターフェイス、Point-to-Point インターフェイス IP、管理用 IP アドレス等）を、外部および内部の不正なトラフィックや分散サービス拒否（DDoS）攻撃から保護するための最も基本的かつ強力な第一線防御テクノロジーです。

CCIE Security v6.1 Blueprint **3.1 Device hardening techniques and control plane protection methods (3.1.c iACLs)** では、インフラストラクチャアドレス空間（Infrastructure Address Space）の分類・モデル化、4 階層型（Multi-Tier）iACL アーキテクチャの設計、通過トラフィック（Transit Traffic）とインフラ宛てトラフィック（Infrastructure-destined Traffic）の厳密な分離、ハードウェア ASIC / TCAM における高速パケットフィルタリング、CoPP（Control Plane Policing）や uRPF（Unicast Reverse Path Forwarding）との連携設計、および Cisco IOS/IOS-XE、Cisco ASA、Cisco FTD における高度な実装・トラブルシューティング能力が精緻に問われます。

---

## 📘 概要

* **機能概要**:
  * **iACL（Infrastructure ACL）**: ネットワークのエッジ（外縁）インターフェイスや内部コア境界インターフェイスの受信方向（`inbound`）にバインドされ、インフラ機器自体の IP アドレス空間（Infrastructure IP Space）宛てのアクセスを厳密に制限する拡張アクセスリスト（Extended ACL）。
  * **Transit Traffic と Infrastructure Traffic の分離**:
    * **Transit Traffic（通過トラフィック）**: 送信元および宛先がインフラ外部のユーザー/サーバーであり、ルーターを単に通過（フォワーディング）するデータプレーンパケット。iACL はこれを透過的に許可（`permit ip any any`）します。
    * **Infrastructure Traffic（インフラ宛てトラフィック）**: 宛先 IP がルーター自身のインターフェイス IP やループバック IP である制御/管理プレーンパケット。iACL は明示的に許可されたプロトコル（BGP, OSPF, EIGRP, SSH, SNMP, NTP, TACACS+ 等）および許可された送信元 IP のみをパスさせ、それ以外のインフラ宛てパケットを即座に破棄（`deny`）します。
* **利用目的**:
  * 外部インターネットや非信頼ネットワークからのインフラ機器に対する直接的な DoS / DDoS 攻撃、ポートスキャン、不正ログインの物理的遮断。
  * ネットワーク機器の CPU やコントロールプレーンに不要なトラフィックが到達するのをハードウェアラインカード（ASIC/TCAM）レベルで早期阻止（Early Drop）し、デバイスの安定稼働を維持する。
  * 自インフラストラクチャの IP アドレスを送信元とする IP スプーフィング攻撃（インサイドアウト/アウトサイドインのなりすまし）の排除。
* **どのような場面で利用するか**:
  * サービスプロバイダー境界（AS エッジルーター）。
  * 企業ネットワークのインターネットゲートウェイ（外部ルーター/ファイアウォールのアウトバウンド/インバウンド境界）。
  * データセンターのエッジおよびコア層のインターフェイス。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | インフラ機器自身の IP アドレスブロック（ループバック、P2P リンク IP）を保護するため、エッジインターフェイスの受信方向に適用されるマルチティアー（多層）拡張 ACL |
| **用途** | 未承認トラフィックによるコントロールプレーン負荷増大防止、インフラ宛て DoS/DDoS 攻撃遮断、IP スプーフィング防止 |
| **メリット** | **ハードウェア（ASIC/TCAM）で高速処理**されるため、CPU にパケットが到達する前にラインカードで即座にドロップ可能。デバイスの処理負荷を一切増加させない |
| **デメリット** | インフラアドレス空間の正確な集約（Aggregation）設計が必要。新規管理サーバーやルーティングピア追加時に ACL の更新漏れがあると通信障害が発生する |
| **対応機種** | Cisco IOS, Cisco IOS-XE ルーター, Cisco Catalyst スイッチ, Cisco ASA, Cisco FTD |
| **制限事項** | フラグメントパケット（Fragmented Packets）や非 IP プロトコルの評価ルールに配慮が必要。大量の `log` オプション設定は CPU 負荷を引き起こすリスクがある |
| **設計上の注意点** | インフラ IP アドレスのナンバリングプランを連続したブロック（例: `10.255.0.0/16`）に統一し、iACL ルール行数を最小限に抑えることが最重要 |

---

## 🏗 動作原理

iACL は、ルーターのイングラストラフィックを **「自機器のインフラ空間宛て」** と **「他宛ての通過トラフィック」** に論理的に分離して評価します。

```text
[ Incoming Packet at Ingress Interface ]
                   │
                   ▼
┌────────────────────────────────────────────────────────┐
│ Tier 1: Anti-Spoofing Check                            │
│  送信元 IP が自インフラ IP アドレス空間か？            │
│   ├── YES ──► [ DENY & DROP ] (外部からのスプーフィング) │
│   └── NO  ──► Continue                                 │
└────────────────────────┬───────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────┐
│ Tier 2: Explicit Permit for Authorized Protocols       │
│  宛先が自インフラ IP かつ 許可されたプロトコル/送信元か？  │
│  (例: BGP, OSPF, SSH from Mgt Net, ICMP Echo)         │
│   ├── YES ──► [ PERMIT ] ──► Processed by CPU/Control Plane│
│   └── NO  ──► Continue                                 │
└────────────────────────┬───────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────┐
│ Tier 3: Explicit Deny for Infrastructure Space         │
│  宛先が自インフラ IP アドレス空間か？                  │
│   ├── YES ──► [ DENY & DROP ] (未承認のインフラ宛て通信) │
│   └── NO  ──► Continue                                 │
└────────────────────────┬───────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────┐
│ Tier 4: Permit Transit Traffic                         │
│  宛先がユーザー/データプレーン宛ての通過通信か？        │
│   └── PERMIT ──► [ FORWARD ] (FIB/CEF 経由で転送)     │
└────────────────────────────────────────────────────────┘
```

### 階層型 (Multi-Tier) iACL 構成モデル

1. **Tier 1 (Anti-Spoofing Rules)**:
   * 外部から流入するパケットの送信元 IP アドレスが自組織のインフラ IP アドレスブロックである場合、なりすまし（IP Spoofing）として即座に破棄します。
2. **Tier 2 (Explicit Permit Rules for Control/Management)**:
   * 業務上およびネットワーク運用上必要な明示的トラフィックのみを許可します。
   * インフラ宛ての管理通信（指定管理者 IP からの SSH / SNMP / TACACS+ / NTP / Syslog）。
   * 隣接ルーターとの制御プロトコル（BGP ピア IP からの TCP 179、OSPF `224.0.0.5`/`224.0.0.6` または Unicast、EIGRP `224.0.0.10`、LDP TCP/UDP 646）。
   * 必要な ICMP メッセージ（Echo Request/Reply, Unreachable, Time Exceeded）。
3. **Tier 3 (Explicit Deny for Infrastructure Address Space)**:
   * Tier 2 で明示的に許可されなかった、インフラストラクチャ IP アドレス空間（例: `10.255.0.0/16`）宛てのすべてのパケットを拒否します（トラブルシューティング用に `log-input` または `log` オプションを付与）。
4. **Tier 4 (Permit Transit Traffic)**:
   * インフラ空間宛てではない、一般的な通過データトラフィック（Internet / Customer Transit Traffic）をすべて許可（`permit ip any any`）します。

---

## ⚙ 動作シーケンス

1. **パケット着信と ASIC/TCAM 検索**:
   * ルーターの物理または論理イングラストラフィックポートにパケットが着信。
   * インターフェイスに適用された `ip access-group <iACL_NAME> in` に基づき、ハードウェア ASIC / TCAM テーブルを検索（CPU 非介入）。
2. **Tier 1 (Anti-Spoofing) 判定**:
   * パケットの送信元 IP を照合。外部インターフェイスからのパケットでありながら、送信元が自社インフラ IP (`10.255.0.0/16`) である場合、即座に TCAM でドロップ。
3. **Tier 2 (Explicit Authorized Traffic) 判定**:
   * パケットの宛先 IP が自機器のループバックやインターフェイス IP (`10.255.1.1`) であり、かつプロトコル/ポートが許可リスト（例: 許可された NOC 端末 `192.168.1.50` からの SSH TCP 22）にマッチした場合、`PERMIT` 判定となりパケットはコントロールプレーン（CPU）へ転送。
4. **Tier 3 (Explicit Infrastructure Deny) 判定**:
   * パケットの宛先 IP が自社のインフラ IP アドレス空間 (`10.255.0.0/16`) に含まれ、Tier 2 の許可リストにマッチしなかった場合（例: 外部攻撃者からのループバック宛て Telnet または ICMP スキャン）、明示的 `deny ip any 10.255.0.0 0.0.255.255 log-input` にマッチして即座にドロップ。Syslog ログが生成。
5. **Tier 4 (Transit Forwarding) 判定**:
   * パケットの宛先 IP が自インフラ空間外（例: 内部 Web サーバー `172.16.10.100` 宛て）の場合、`permit ip any any` にマッチ。パケットはデータプレーン（CEF / FIB）で通常の高速スイッチング処理を受け、エグラストラフィックインターフェイスから転送。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **インフラ IP アドレスブロックの正確な要約 (Aggregation)**:
   * ラボ要件で「インフラ空間 `10.255.0.0/16` 宛ての未承認アクセスを拒否せよ」と指定された場合、ルーターの全ループバック（Loopback0, 1...）および P2P リンク IP を包含するワイルドカードマスク（`10.255.0.0 0.0.255.255`）を正確に計算する必要があります。
2. **制御プロトコル（Routing Protocols）の破断防止**:
   * iACL 適用時に最も多い誤りは、BGP (TCP 179)、OSPF (Protocol 89 / Multicast `224.0.0.5`/`224.0.0.6`)、EIGRP (Protocol 88)、LDP (TCP/UDP 646) の許可ルールを忘れて、**ルーティングネイバーシップを即死させてしまうこと**です。
   * 特に関数やネイバー IP がユニキャストかマルチキャストか（例: OSPF Point-to-Point ユニキャスト等）を考慮して `permit` ACE を記述しなければなりません。
3. **ICMP トラフィックの適切な許可ルール**:
   * `permit icmp any any echo`（Ping 応答）、`permit icmp any any echo-reply`、`permit icmp any any unreachable`（Path MTU Discovery 用）、`permit icmp any any ttl-exceeded`（Traceroute 用）を明示的に指定しないと、ネットワーク運用および Path MTU Discovery が破断します。
4. **`log` / `log-input` オプションの制限とリスク**:
   * ラボ問題で「拒否されたインフラ宛てパケットの送信元 MAC/Interface 情報をログに記録せよ」と要求された場合、`deny` 行の末尾に **`log-input`** を付与します。
   * ただし、大量の DoS パケット着信時に `log-input` が CPU 負荷を引き起こすため、`ip access-list logging interval` の設定が必要になる場合があります。
5. **iACL と CoPP の組み合わせ問題**:
   * **iACL**: エッジインターフェイスで全インフラ空間宛てをドロップ（Transit パケットは通す）。
   * **CoPP**: 実際に CPU（Control Plane）に到達したパケットの帯域制限。
   * 試験で「CoPP と iACL の両方を用いてインフラを保護せよ」と出題された場合、処理順序（iACL が先、CoPP が後）と役割分担を意識してコンフィグを作成します。

---
## 🛠 設定方法

### 1. Cisco IOS / IOS-XE: Multi-Tier iACL 標準構成

```bash
! ===== Step 1: インフラストラクチャ IP アドレス空間のオブジェクトグループ/定義 =====
! インフラストラクチャ空間: 10.255.0.0/16 (ループバックおよび P2P リンク)
! 許可された管理ネットワーク: 192.168.100.0/24 (NOC セグメント)
! 外部 BGP ピア IP: 198.51.100.1

! ===== Step 2: iACL (拡張命名付きアクセスリスト) の作成 =====
ip access-list extended ACL_INFRASTRUCTURE_PROTECTION

 ! --- Tier 1: Anti-Spoofing (自社インフラ IP の外部からの流入をブロック) ---
 deny ip 10.255.0.0 0.0.255.255 any log-input

 ! --- Tier 2: Explicit Permits for Management & Control Plane ---
 ! 1. ルーティングプロトコル (BGP, OSPF, EIGRP, LDP)
 permit tcp host 198.51.100.1 host 10.255.0.1 eq bgp
 permit tcp host 198.51.100.1 eq bgp host 10.255.0.1
 permit ospf any any
 permit eigrp any any
 permit tcp host 198.51.100.1 host 10.255.0.1 eq 646
 permit udp host 198.51.100.1 host 10.255.0.1 eq 646

 ! 2. 許可された NOC からの管理アクセス (SSH, SNMP, TACACS+)
 permit tcp 192.168.100.0 0.0.0.255 10.255.0.0 0.0.255.255 eq 22
 permit udp 192.168.100.0 0.0.0.255 10.255.0.0 0.0.255.255 eq snmp
 permit udp host 172.16.1.10 10.255.0.0 0.0.255.255 eq ntp

 ! 3. 必要な ICMP トラフィック (Ping, PMTUD, Traceroute)
 permit icmp any 10.255.0.0 0.0.255.255 echo
 permit icmp any 10.255.0.0 0.0.255.255 echo-reply
 permit icmp any 10.255.0.0 0.0.255.255 unreachable
 permit icmp any 10.255.0.0 0.0.255.255 packet-too-big
 permit icmp any 10.255.0.0 0.0.255.255 time-exceeded

 ! --- Tier 3: Explicit Deny for Infrastructure Space ---
 deny ip any 10.255.0.0 0.0.255.255 log-input

 ! --- Tier 4: Permit Transit Traffic ---
 permit ip any any

! ===== Step 3: エッジ（イングラストラフィック）インターフェイスへの適用 =====
interface GigabitEthernet0/0/0
 description ## External/Internet Edge Interface ##
 ip access-group ACL_INFRASTRUCTURE_PROTECTION in
```

---

### 2. Cisco ASA: Control-Plane ACL によるインフラ保護

Cisco ASA においては、`access-group <ACL> in interface <IF>`（データプレーン用）とは別に、**`access-group <ACL> control-plane`** を用いて ASA 自身宛ての To-Box トラフィック（管理通信・VPN・BGP/OSPF等）に対する iACL を適用します。

```bash
! ===== ASA Control-Plane iACL 設定 =====
access-list ACL_ASA_CP_IACL extended permit tcp 192.168.100.0 255.255.255.0 host 172.16.1.1 eq ssh
access-list ACL_ASA_CP_IACL extended permit ospf any any
access-list ACL_ASA_CP_IACL extended permit icmp 192.168.100.0 255.255.255.0 host 172.16.1.1 echo
access-list ACL_ASA_CP_IACL extended deny ip any host 172.16.1.1 log default
access-list ACL_ASA_CP_IACL extended permit ip any any

! グローバル・コントロールプレーンにバインド
access-group ACL_ASA_CP_IACL control-plane
```

---

### 3. Cisco FTD (FMC Managed): FlexConfig による iACL 適用

FMC 管理下の FTD において、LINA エンジンレベルで直接インフラストラクチャ宛てパケットを遮断するために **FlexConfig** を用いて Control-Plane ACL を挿入します。

```text
[ FMC GUI Steps ]
1. Objects > Object Management > FlexConfig > Text Location ➔ "Add Text Location":
   - Name: FlexConfig_iACL_Obj
   - Insert Code:
     access-list ACL_FTD_IACL extended permit tcp 192.168.100.0 255.255.255.0 any eq 22
     access-list ACL_FTD_IACL extended permit ospf any any
     access-list ACL_FTD_IACL extended deny ip any 10.255.0.0 255.255.0.0 log
     access-list ACL_FTD_IACL extended permit ip any any
     access-group ACL_FTD_IACL control-plane
2. Devices > FlexConfig ➔ 対象 FTD の FlexConfig Policy 編集。
3. "FlexConfig_iACL_Obj" をアサインして Save ＆ Deploy。
```

---

## 🔍 検証コマンド

| 目的 | コマンド | 確認ポイント |
| :--- | :--- | :--- |
| **iACL の設定 ＆ Hitcount 確認** | <code>show ip access-lists ACL_INFRASTRUCTURE_PROTECTION</code> | 各 ACE のヒット件数（`matches`）、特に Tier 3 `deny` 行および Tier 4 `permit ip any any` の数値推移 |
| **インターフェイスへの ACL バインド確認** | <code>show ip interface GigabitEthernet0/0/0 \| inc Inbound</code> | `Inbound access list is ACL_INFRASTRUCTURE_PROTECTION` の表示 |
| **ASA コントロールプレーン ACL 確認** | <code>show access-list ACL_ASA_CP_IACL</code> | ASA To-Box パケットにマッチした ACE とヒットカウント |
| **iACL ドロップパケットのログ確認** | <code>show logging \| inc %SEC-6-IPACCESSLOGP</code> | `deny` 行でログ記録された送信元 IP、宛先 IP、プロトコル、インプットインターフェイス、MAC アドレス |
| **ハードウェア (TCAM) ドロップ統計確認** | <code>show platform hardware qfp active feature acl stats</code> (IOS-XE) | QFP ハードウェアレベルでの ACL ドロップ件数 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **iACL 適用直後に BGP または OSPF ネイバーシップが Down した** | Tier 2（Explicit Permit）で BGP (TCP 179) や OSPF (IP Protocol 89) の許可ルールが漏れていたため、Tier 3 で遮断された | <code>show ip bgp summary</code><br><code>show ip access-list</code> | BGP ピア IP 宛ての TCP 179 および OSPF マルチキャスト/ユニキャストの `permit` ACE を Tier 2 の最上部に追加する。 |
| **NOC からルーターへの SSH アクセスが拒否される** | Tier 2 で NOC の送信元 IP サブネット指定ミス、またはループバック IP 宛て許可の指定漏れ | <code>show logging</code><br><code>show ip access-list</code> | Syslog でドロップログを確認し、NOC IP からインフラ IP 宛ての `permit tcp <NOC_SUBNET> <INFRA_SUBNET> eq 22` を投入する。 |
| **iACL 適用後、外部からの Traceroute や PMTUD が機能しない** | ICMP Time Exceeded (Type 11) や Unreachable / Packet Too Big (Type 3) が iACL でブロックされている | <code>show ip access-list</code> | Tier 2 に `permit icmp any <INFRA_SUBNET> time-exceeded` および `unreachable` を追加する。 |
| **`deny ... log-input` を設定したらルーターの CPU 使用率が 100% に高騰した** | 大量の DoS パケットが Tier 3 ドロップにマッチし、Syslog 生成処理が CPU を圧迫した | <code>show processes cpu history</code> | `ip access-list logging interval` を調整するか、過度な DoS パケットに対しては `log-input` なしの `deny` に切り替える。 |

---
## ⚠ 制限事項

* **フラグメントパケット (Fragmented Packets) の評価**:
  * FO (Fragment Offset) > 0 の 2 通目以降の IP フラグメントパケットは L4 ポート情報（TCP 22 や BGP 179）を持たないため、L4 ポート条件付きの ACE をスルーして Tier 3 または Tier 4 にマッチする危険性があります。
  * 必要に応じて `permit tcp any any eq bgp fragments` などのフラグメント評価 ACE を考慮する必要があります。
* **ASA コントロールプレーン ACL の制限**:
  * ASA の `access-group <ACL> control-plane` は、**ASA 自身を宛先とする通信 (To-Box)** のみに作用します。ASA を通過するトラフィック (Through-Box) には影響しません（Through-Box には通常の `access-group <ACL> in interface <IF>` を使用）。
* **VACL / CoPP との階層関係**:
  * iACL はレイヤ 3 インターフェイスの受信で評価されるため、レイヤ 2 物理ポートで VACL が設定されている場合、VACL が先（L2）、iACL が次（L3 Ingress）、CoPP が最後（CPU 投入手前）の順で評価されます。

---

## 🔄 他技術との関連

* **CoPP (Control Plane Policing)**: iACL で集約ドロップされなかったパケットがコントロールプレーンに達した際、レート制限（Policing）を行う相補的防御機能。
* **uRPF (Unicast Reverse Path Forwarding)**: 送信元 IP アドレスのなりすまし（IP Spoofing）をルーティングテーブルの逆引きで排除する機能。iACL の Tier 1 (Anti-Spoofing) と組み合わせて第一線防御を固める。
* **Infrastructure IP Aggregation (IP ナンバリング設計)**: ルーターの Loopback や P2P リンクを `10.255.0.0/16` などの連続したブロックに設計することで、iACL の Tier 3 拒否ルールを 1 行に要約可能。

---

## 🧩 比較表

### 1. iACL vs CoPP vs uRPF vs VACL

| 比較項目 | iACL (Infrastructure ACL) | CoPP (Control Plane Policing) | uRPF (Unicast RPF) | VACL (VLAN ACL) |
| :--- | :--- | :--- | :--- | :--- |
| **目的** | インフラ IP 空間全体の保護・アクセス制限 | CPU/コントロールプレーンの帯域制限 | 送信元 IP スプーフィングの排除 | Intra-VLAN トラフィックの L2 フィルタリング |
| **処理ポイント** | L3 インターフェイス Ingress | CPU 投入直前の Control-Plane | L3 インターフェイス Ingress | VLAN スイッチング ASIC (L2) |
| **対象トラフィック** | インフラ IP 宛て ＆ 通過トラフィック | CPU 宛て (To-Box) トラフィックのみ | 全 Ingress トラフィック | VLAN 内全 L2/L3 トラフィック |
| **ハードウェアオフロード**| **可能** (ASIC/TCAM で高速処理) | 一部機能 (QFP/ASIC ポリサー) | **可能** (FIB 検索) | **可能** (VLAN TCAM) |

---

## 💡 ベストプラクティス

1. **インフラストラクチャ IP アドレスの集約化 (IP Address Structuring)**:
   * 全ルーターの Loopback IP および P2P リンク IP を特定の集約プレフィックス（例: `10.255.0.0/16`）に配置します。これにより、iACL の Tier 3 拒否ルールを `deny ip any 10.255.0.0 0.0.255.255 log-input` 1 行で簡潔に記述できます。
2. **すべての外部境界インターフェイスへの適用 (Apply to All Edge Interfaces)**:
   * iACL はインターネット接続ポートだけでなく、サードパーティ接続、WAN エッジ、データセンター境界など、外部からパケットが流入するすべてのインターフェイスの `inbound` にバインドします。
3. **明示的な `deny` 行と `log-input` の活用**:
   * ACL 末尾の暗黙の拒否 (`deny ip any any`) に頼らず、Tier 3 で `deny ip any <INFRA_BLOCK> log-input` を明示的に記述し、拒否されたインフラ攻撃パケットの送信元 IP・MAC・着信ポートを特定できるようにします。

---

## 📝 ラボ学習・設定サンプル例

### 例1: 基本的な Multi-Tier iACL の構築 (IOS-XE)

#### 問題
ルーター R1 の外部インターフェイス G0/0/0 に対し、自社インフラ空間 `10.255.0.0/16` を保護する iACL `iACL_IN` を適用せよ。

#### 要件
* 外部からの送信元 IP `10.255.0.0/16` のパケットを拒否 (`log-input`)。
* BGP ピア `198.51.100.2` から Loopback0 (`10.255.0.1`) 宛ての BGP (TCP 179) を許可。
* 管理セグメント `192.168.10.0/24` からインフラ空間宛ての SSH (TCP 22) および ICMP Echo を許可。
* それ以外のインフラ空間宛てパケットを拒否 (`log-input`)。
* 通過（Transit）トラフィックはすべて許可。

#### 設定例

```bash
! ===== R1 Configuration =====
ip access-list extended iACL_IN
 ! Tier 1: Anti-Spoofing
 deny ip 10.255.0.0 0.0.255.255 any log-input
 ! Tier 2: Explicit Permits
 permit tcp host 198.51.100.2 host 10.255.0.1 eq bgp
 permit tcp host 198.51.100.2 eq bgp host 10.255.0.1
 permit tcp 192.168.10.0 0.0.0.255 10.255.0.0 0.0.255.255 eq 22
 permit icmp 192.168.10.0 0.0.0.255 10.255.0.0 0.0.255.255 echo
 permit icmp any 10.255.0.0 0.0.255.255 unreachable
 permit icmp any 10.255.0.0 0.0.255.255 time-exceeded
 ! Tier 3: Explicit Infrastructure Deny
 deny ip any 10.255.0.0 0.0.255.255 log-input
 ! Tier 4: Permit Transit
 permit ip any any

interface GigabitEthernet0/0/0
 ip access-group iACL_IN in
```

#### 検証方法
* `show ip access-lists iACL_IN` を実行し、BGP および SSH 通信発生時に該当 ACE のヒットカウントが増加すること、および非許可端末からの Loopback 宛て Ping が Tier 3 でドロップされログ出力されることを確認。

---

### 例2: OSPF / EIGRP 制御プロトコルを含む iACL の実装

#### 問題
R2 において、インフラ空間 `172.16.255.0/24` を保護しつつ、直接接続された OSPF ネイバーおよび EIGRP ネイバーとの通信を阻害しない iACL を作成せよ。

#### 要件
* OSPF (Protocol 89) および EIGRP (Protocol 88) をインフラ空間宛てに許可。
* 管理端末 `10.0.0.50/32` からの SNMP (UDP 161) を許可。
* その他のインフラ空間宛てパケットを拒否。

#### 設定例

```bash
! ===== R2 Configuration =====
ip access-list extended ACL_INFRA_ROUTING
 permit ospf any any
 permit eigrp any any
 permit udp host 10.0.0.50 172.16.255.0 0.0.0.255 eq snmp
 permit icmp any 172.16.255.0 0.0.0.255 echo
 deny ip any 172.16.255.0 0.0.0.255 log-input
 permit ip any any

interface GigabitEthernet0/0/1
 ip access-group ACL_INFRA_ROUTING in
```

#### 検証方法
* `show ip ospf neighbor` および `show ip eigrp neighbors` を実行し、iACL 適用後もネイバー関係が UP を維持していることを確認。

---

### 例3: Cisco ASA Control-Plane iACL の構築

#### 問題
ASA1 において、外向きインターフェイス `outside` から ASA 自身宛ての管理トラフィックを制御する Control-Plane iACL を構成せよ。

#### 要件
* 管理サブネット `192.168.50.0/24` から ASA Outside IP (`198.51.100.10`) 宛ての SSH および HTTPS (ASDM) を許可。
* それ以外の Outside から ASA 自身宛ての全 IP トラフィックを拒否。

#### 設定例

```bash
! ===== ASA1 Configuration =====
access-list ACL_ASA_CP extended permit tcp 192.168.50.0 255.255.255.0 host 198.51.100.10 eq ssh
access-list ACL_ASA_CP extended permit tcp 192.168.50.0 255.255.255.0 host 198.51.100.10 eq https
access-list ACL_ASA_CP extended deny ip any host 198.51.100.10 log default
access-list ACL_ASA_CP extended permit ip any any

access-group ACL_ASA_CP control-plane
```

#### 検証方法
* `show access-list ACL_ASA_CP` でヒットカウントを確認し、未承認 IP から ASA Outside IP 宛ての通信が `control-plane` ACL でドロップされることを確認。

---

### 例4: TACACS+ / RADIUS / NTP 認証インフラを保護する iACL

#### 問題
R3 において、インフラ IP アドレス群 `10.100.0.0/16` に対し、特定の AAA サーバー (`10.1.1.10`) および NTP サーバー (`10.1.1.20`) との双方向通信のみを許可する iACL を構築せよ。

#### 要件
* TACACS+ (TCP 49) および NTP (UDP 123) 通信を許可。
* 承認された外部対向 IP 以外からのインフラ空間宛てパケットを拒否。

#### 設定例

```bash
! ===== R3 Configuration =====
ip access-list extended ACL_INFRA_AAA
 permit tcp host 10.1.1.10 10.100.0.0 0.0.255.255 eq tacacs
 permit tcp host 10.1.1.10 eq tacacs 10.100.0.0 0.0.255.255
 permit udp host 10.1.1.20 eq ntp 10.100.0.0 0.0.255.255
 permit icmp host 10.1.1.10 10.100.0.0 0.0.255.255
 deny ip any 10.100.0.0 0.0.255.255 log-input
 permit ip any any

interface TenGigabitEthernet0/1/0
 ip access-group ACL_INFRA_AAA in
```

#### 検証方法
* `test aaa group tacacs+ admin Password123 legacy` を実行して認証が正常に成功し、iACL の TACACS+ 許可行のカウントが増加することを確認。

---

### 例5: LDP / MPLS インフラを保護する iACL の構築

#### 問題
MPLS コアルーター R4 において、LDP ネイバー `10.255.0.2` との LDP セッション (TCP/UDP 646) を許可しつつ、インフラ空間 `10.255.0.0/24` を保護する iACL を設定せよ。

#### 要件
* LDP Discovery (UDP 646) および LDP Session (TCP 646) を許可。

#### 設定例

```bash
! ===== R4 Configuration =====
ip access-list extended ACL_MPLS_INFRA
 permit tcp host 10.255.0.2 host 10.255.0.1 eq 646
 permit udp host 10.255.0.2 host 10.255.0.1 eq 646
 permit ospf host 10.255.0.2 host 10.255.0.1
 deny ip any 10.255.0.0 0.0.0.255 log-input
 permit ip any any

interface GigabitEthernet0/0/2
 ip access-group ACL_MPLS_INFRA in
```

#### 検証方法
* `show mpls ldp neighbor` を実行し、LDP セッションが `OPERATIONAL` 状態を維持していることを確認。

---

### 例6: IP Option ドロップと iACL の統合保護

#### 問題
R5 において、インフラ保護 ACL に加えて IP Option（LSRR/SSRR 等）を含むパケットをハードウェアレベルで一括ドロップする設定を追加せよ。

#### 要件
* `ip options drop` コマンドを使用。
* インフラ空間 `10.200.0.0/16` への未承認アクセスを拒否。

#### 設定例

```bash
! ===== R5 Configuration =====
ip options drop

ip access-list extended ACL_INFRA_OPT
 deny ip 10.200.0.0 0.0.255.255 any log-input
 permit tcp 10.10.0.0 0.0.255.255 10.200.0.0 0.0.255.255 eq 22
 deny ip any 10.200.0.0 0.0.255.255 log-input
 permit ip any any

interface GigabitEthernet0/0/0
 ip access-group ACL_INFRA_OPT in
```

#### 検証方法
* `show ip traffic` を実行し、IP Option ドロップ件数が増加することを確認。

---

### 例7: uRPF Strict Mode と iACL の併用構成

#### 問題
R6 の WAN インターフェイス G0/0/0 において、uRPF Strict Mode と iACL を同時に適用し、送信元スプーフィングとインフラ攻撃を多層防御せよ。

#### 要件
* uRPF Strict Mode を適用。
* iACL でインフラ空間 `10.250.0.0/16` を保護。

#### 設定例

```bash
! ===== R6 Configuration =====
ip access-list extended ACL_DUAL_DEFENSE
 permit tcp host 203.0.113.5 host 10.250.0.1 eq bgp
 permit tcp 192.168.1.0 0.0.0.255 10.250.0.0 0.0.255.255 eq 22
 permit icmp any 10.250.0.0 0.0.255.255 echo
 deny ip any 10.250.0.0 0.0.255.255 log-input
 permit ip any any

interface GigabitEthernet0/0/0
 ip verify unicast source reachable-via rx
 ip access-group ACL_DUAL_DEFENSE in
```

#### 検証方法
* `show cef interface GigabitEthernet0/0/0` を実行し、uRPF がアクティブであることを確認の上、非対称ルートパケットが uRPF で、未承認インフラパケットが iACL でドロップされることを検証。

---

### 例8: CoPP と iACL の連動構成

#### 問題
R7 において、iACL でインフラ空間宛てアクセスを制限し、かつ iACL を通過して CPU に到達する ICMP パケットを CoPP で 64 kbps に制限せよ。

#### 要件
* iACL で Ping (ICMP Echo) を許可。
* CoPP で ICMP トラフィックを 64000 bps にポリシング。

#### 設定例

```bash
! ===== R7 Configuration =====
! 1. iACL 設定
ip access-list extended ACL_IACL_COPP
 permit icmp any 10.255.0.0 0.0.255.255 echo
 deny ip any 10.255.0.0 0.0.255.255 log-input
 permit ip any any

interface GigabitEthernet0/0/0
 ip access-group ACL_IACL_COPP in

! 2. CoPP 設定
ip access-list extended ACL_COPP_ICMP
 permit icmp any any echo

class-map CM_COPP_ICMP
 match access-list ACL_COPP_ICMP

policy-map PM_COPP
 class CM_COPP_ICMP
  police 64000 conform-action transmit exceed-action drop

control-plane
 service-policy input PM_COPP
```

#### 検証方法
* 外部から大量の Ping を送信し、iACL は通過するが `show policy-map control-plane` で CoPP ポリサーの `exceed` カウントが増加することを確認。

---

### 例9: IPv6 インフラストラクチャ保護 ACL (iACLv6) の作成

#### 問題
R8 において、IPv6 インフラストラクチャ空間 `2001:db8:infra::/48` を保護する IPv6 iACL `iACLv6_IN` を作成・適用せよ。

#### 要件
* 送信元 `2001:db8:infra::/48` のスプーフィングパケットを拒否。
* ICMPv6 Neighbor Discovery (NS/NA/RS/RA) および Packet Too Big を許可。
* 管理サブネット `2001:db8:mgt::/64` からの SSH を許可。
* その他のインフラ空間宛て IPv6 通信を拒否。

#### 設定例

```bash
! ===== R8 Configuration =====
ipv6 access-list iACLv6_IN
 deny ipv6 2001:db8:infra::/48 any
 permit icmp any any neighbor-solicitation
 permit icmp any any neighbor-advertisement
 permit icmp any any router-solicitation
 permit icmp any any router-advertisement
 permit icmp any any packet-too-big
 permit tcp 2001:db8:mgt::/64 2001:db8:infra::/48 eq 22
 deny ipv6 any 2001:db8:infra::/48
 permit ipv6 any any

interface GigabitEthernet0/0/0
 ipv6 traffic-filter iACLv6_IN in
```

#### 検証方法
* `show ipv6 access-list iACLv6_IN` でヒットカウントを確認し、IPv6 ND 通信が阻害されずネイバーが正常維持されることを検証。

---

### 例10: FMC FlexConfig による FTD Control-Plane iACL の導入

#### 問題
FTD1 において、FMC FlexConfig を使用して Management ネットワーク `10.10.10.0/24` 以外の外部 IP から FTD への直接通信を拒否する Control-Plane ACL を定義せよ。

#### 要件
* `access-group <ACL> control-plane` コマンドを使用。

#### 設定例

```text
[ FMC FlexConfig Configuration ]
1. Text Location Object: "iACL_FTD_Obj"
   access-list ACL_CONTROL_PLANE extended permit tcp 10.10.10.0 255.255.255.0 any eq 22
   access-list ACL_CONTROL_PLANE extended permit ospf any any
   access-list ACL_CONTROL_PLANE extended deny ip any any log
   access-group ACL_CONTROL_PLANE control-plane

2. Assign Object to FTD Policy & Deploy.
```

#### 検証方法
* FTD Diagnostic CLI で `show access-list ACL_CONTROL_PLANE` を実行し、非許可 IP からの To-Box パケットがドロップされていることを確認。

---
## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 以下の iACL コンフィグが適用されたルーター R1 において、外部ルーター（IP: `198.51.100.2`）からの OSPF Hello パケットが破棄され、OSPF ネイバーシップが確立しない。原因として正しいものはどれか。

```bash
ip access-list extended ACL_INFRA
 deny ip 10.255.0.0 0.0.255.255 any
 permit tcp host 198.51.100.2 host 10.255.0.1 eq bgp
 permit tcp 192.168.1.0 0.0.0.255 10.255.0.0 0.0.255.255 eq 22
 deny ip any 10.255.0.0 0.0.255.255 log-input
 permit ip any any

interface GigabitEthernet0/0/0
 ip access-group ACL_INFRA in
```

* A) OSPF は TCP 179 を使用するため、BGP の許可ルールに統合する必要がある。
* B) OSPF パケット（IP Protocol 89）を許可する `permit ospf any any` ルールが Tier 2 に存在しないため、Tier 3 (`deny ip any 10.255.0.0 0.0.255.255`) にヒットして破棄されている。
* C) `deny ip 10.255.0.0 0.0.255.255 any` が OSPF パケットを破棄している。
* D) OSPF は `permit ip any any` にヒットして転送されるため、iACL に問題はない。

**正解**: **B**  
**解説**: OSPF パケットは宛先 IP としてマルチキャスト `224.0.0.5`/`224.0.0.6` やルーターのインターフェイス IP (`10.255.0.1`) を使用します。本 ACL には OSPF (IP Protocol 89) を許可する文が存在しないため、Tier 3 の `deny ip any 10.255.0.0 0.0.255.255`（または暗黙の拒否）にヒットして破棄されます。正しく動作させるには `permit ospf any any` を Tier 2 に挿入する必要があります。

---

### 2. トラブルシュート問題

**問題**: iACL をエッジルーターに適用したところ、ネットワークのトラブルシューティング時に外部からインフラ IP アドレス宛ての Traceroute が失敗し、途中経路のルーター IP が表示されなくなった。iACL に追加すべき最も適切な ACE（Access Control Entry）はどれか。

* A) `permit icmp any any echo-reply`
* B) `permit icmp any 10.255.0.0 0.0.255.255 time-exceeded` および `permit icmp any 10.255.0.0 0.0.255.255 unreachable`
* C) `permit udp any any eq 33434`
* D) `permit ip any any`

**正解**: **B**  
**解説**: Traceroute ツールは、UDP/ICMP パケットの TTL を 1 ずつ増やして送信し、途中のルーターから返される `ICMP Time Exceeded (Type 11)` メッセージおよび目的地の `ICMP Port Unreachable (Type 3)` メッセージを受信することで経路を特定します。iACL でこれらの ICMP メッセージをブロックすると Traceroute が破断するため、明示的に `time-exceeded` と `unreachable` を許可する必要があります。

---

### 3. Design 問題

**問題**: サービスプロバイダーのマルチホーム境界ルーターにおいて、インフラストラクチャ IP アドレス空間 `10.255.0.0/16` を保護するための iACL 設計として、最も適切なアプローチはどれか。

* A) パケットの処理遅延を避けるため、ルーターの CPU (Control Plane) 上で CoPP のみを用いて全パケットを検査する。
* B) インフラアドレス空間を集約設計（Aggregation）し、エッジインターフェイスの Ingress に 4 階層型 (Multi-Tier) iACL を適用して、ハードウェア ASIC/TCAM レベルで未承認のインフラ宛てパケットを即座に破棄する。
* C) ルーターの全インターフェイスで `deny ip any any` を適用し、必要な通信のみを個別に静的ルーティングする。
* D) iACL を Outgress (Egress) 方向に適用し、外部へ流出するインフラパケットをブロックする。

**正解**: **B**  
**解説**: iACL の設計思想は、**インフラアドレス空間を集約**し、エッジインターフェイスの **Ingress** でハードウェア（ASIC/TCAM）処理させることで、CPU に到達させることなく未承認パケットを早期破棄（Early Drop）することです。

---

### 4. 実装問題 (ASA)

**問題**: Cisco ASA において、Outside インターフェイスを通過するトラフィック（Through-Box）には影響を与えず、ASA 自身の IP アドレス（To-Box）宛ての管理トラフィックや制御プロトコルのみを制限したい。適切な設定手順はどれか。

* A) Extended ACL を作成し、`access-group <ACL> in interface outside` コマンドでバインドする。
* B) Extended ACL を作成し、`access-group <ACL> control-plane` コマンドでバインドする。
* C) MPF で `class-map` を作成し、`inspect` コマンドで制限する。
* D) ASA では To-Box トラフィック専用の ACL はサポートされていない。

**正解**: **B**  
**解説**: Cisco ASA において ASA 自身を宛先とする通信 (To-Box Traffic) のみを評価・制限するには、`access-group <ACL> control-plane` コマンドを使用します。`in interface <IF>` は ASA を通過するトラフィック (Through-Box) のみに作用します。

---

### 5. コンフィグ読解問題

**問題**: 以下の iACL 設定において、外部ネットワークから内部サーバー `172.16.10.100` 宛てに送信された Web トラフィック（TCP 80）はどのように処理されるか。なお、インフラ IP 空間は `10.255.0.0/16` である。

```bash
ip access-list extended ACL_IACL_TEST
 deny ip 10.255.0.0 0.0.255.255 any
 permit tcp host 198.51.100.1 host 10.255.0.1 eq bgp
 deny ip any 10.255.0.0 0.0.255.255 log-input
 permit ip any any
```

* A) `deny ip any 10.255.0.0 0.0.255.255` にマッチして破棄される。
* B) `permit ip any any` にマッチして正常に通過・転送される。
* C) 暗黙の拒否 (`deny ip any any`) にヒットして破棄される。
* D) `deny ip 10.255.0.0 0.0.255.255 any` にマッチして破棄される。

**正解**: **B**  
**解説**: 宛先 IP は `172.16.10.100` であり、インフラ空間 `10.255.0.0/16` には含まれません。したがって、Tier 1, Tier 2, Tier 3 の各条件をスルーし、最終行の Tier 4 **`permit ip any any`** にマッチして正常に通過・転送されます。

---
## 🔗 参考リソース

* [Cisco Technical Report: Protecting Your Core - Infrastructure Protection Access Control Lists](https://www.cisco.com/)
* [Cisco IOS Guidance: Configuring Infrastructure Protection ACLs](https://www.cisco.com/)
* [Cisco ASA 9.4 Configuration Guide: Control Plane Access Groups](https://www.cisco.com/)
* [Cisco Live: Hardening Network Infrastructure Devices (BRKSEC-2003)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * iACL 構築時の黄金ルール:
    1. **インフラ IP 空間を事前に集約化 (`/16` や `/20`) しておくこと**。
    2. **`permit ip any any` を最後に入れ忘れると、全ての通過トラフィックが遮断され全社障害になる**。
    3. **BGP (TCP 179) / OSPF (IP 89) / EIGRP (IP 88) / LDP (TCP/UDP 646) を明示的に permit すること**。
* **図解（iACL 評価の 4-Tier 構造）**:
  ```text
  [ Ingress Packet ]
          │
          ├── [ Tier 1: Anti-Spoofing Check ] ────► Match? ──► [ DROP ]
          │
          ├── [ Tier 2: Explicit Permits ] ────────► Match? ──► [ ALLOW to CPU ]
          │
          ├── [ Tier 3: Infra Space Deny ] ────────► Match? ──► [ DROP & LOG ]
          │
          └── [ Tier 4: Transit Permit (`permit ip any any`) ] ──► [ FORWARD ]
  ```
* **注意点**:
  * ラボ試験で `deny ip any <INFRA_BLOCK> log-input` を設定した後は、必ず `show ip access-list` および `show logging` を実行し、意図しない管理通信や制御プロトコルがドロップされていないかを確認してください。
