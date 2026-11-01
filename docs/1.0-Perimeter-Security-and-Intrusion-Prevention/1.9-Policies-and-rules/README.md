---
layout: default
title: 1.9-Policies-and-rules
nav_order: 9
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.9 Policies and rules for traffic control on Cisco ASA and Cisco FTD

Cisco ネットワークセキュリティ環境における **Traffic Control Policies ＆ Rules（トラフィック制御ポリシーとルール）** は、Cisco ASA（Adaptive Security Appliance）および Cisco FTD（Firepower Threat Defense / Secure Firewall）において、ネットワークを通過するトラフィックを通過（Allow / Trust / Fastpath）、遮断（Block / Drop / Reset）、制御（Rate-limiting / QoS）、あるいは深層検査（Application Inspection / Snort IPS / File / AMP）へ分岐・制御するための最重要制御メカニズムです。

CCIE Security v6.1 Blueprint **1.9 Policies and rules for traffic control on Cisco ASA and Cisco FTD** では、**ASA のインターフェイス Access Group、セキュリティレベル (Security Levels)、Object / Object Groups、Modular Policy Framework (MPF)** から、**FTD の Prefilter Policy (Fastpath / Analyze / Block)、Access Control Policy (ACP: Mandatory / Default Rules, Trust / Allow / Block / Interactive Block)、Identity Policy、QoS Policy、Network Analysis Policy (NAP)、Intrusion Policy (IPS)、および Object Group Search** に至るパケット処理パイプラインの順序・評価メカニズム、設計・構築、高度なトラブルシューティング能力が深く問われます。

---

## 📘 概要

* **機能概要**:
  * **Cisco ASA トラフィック制御**:
    * **Security Level (0–100)**: インターフェイスに割り当てられた信頼度。高セキュリティレベルから低セキュリティレベルへの通信はデフォルトで許可され、逆方向（低から高）および同一セキュリティレベル間（`same-security-traffic` 未設定時）はデフォルトで全ドロップされる。
    * **Access Group / ACL**: インターフェイス（Inbound / Outbound）またはグローバルにバインドされ、L3/L4 ヘッダー（IP、ポート、プロトコル）に基づいて明示的にトラフィックを制御する。
    * **Modular Policy Framework (MPF)**: `class-map` (トラフィック識別)、`policy-map` (アクション定義)、`service-policy` (適用対象) の 3 段階構造により、L7 アプリケーションインスペクション、接続数制限 (Embryonic Connections)、TCP 正常化 (TCP Normalization)、QoS (Policing) を適用する。
  * **Cisco FTD トラフィック制御**:
    * **Prefilter Policy**: パケットインスペクションパイプラインの最前段（LINA レベル）で動作し、トンネルカプセル化（GRE, IP-in-IP 等）の開梱や、5-Tuple（IP/ポート）に基づく **`Fastpath`** (Snort ディープパケットインスペクション回避による超低遅延転送) / **`Analyze`** / **`Block`** を実行する。
    * **Access Control Policy (ACP)**: FTD の中心となるマスターポリシー。`Security Intelligence` (IP/URL/DNS)、`SSL Policy`、`Identity Policy`、`L7 Access Control Rules` (AppID, URL, User, SGT)、`File / AMP Policy`、`Intrusion Policy (IPS)` を統合・順次コールする。
    * **QoS Policy**: Access Control で許可されたトラフィックに対し、インターフェイス単位またはアプリケーション/ユーザー単位で帯域制限（Rate Limiting / Policing）を適用する。
* **利用目的**:
  * 悪意あるトラフィックや不要なプロトコルのエッジでの早期破棄（Early Drop）。
  * 信頼されたトラフィック（例: インフラ管理通信、HFT 高周波取引等）に対するインスペクションバイパスによる遅延削減（Latency Optimization）。
  * アプリケーション識別（AppID）やアイデンティティ（ユーザー / SGT）に基づく高度なゼロトラスト型アクセス制御。
* **どのような場面で利用するか**:
  * 企業 LAN / DMZ / WAN インターネット境界でのアクセス制御。
  * Data Center 内の East-West マイクロセグメンテーションおよび高トラフィックフローの Fastpath 化。

---

## 🔑 要点

| 項目 | Cisco ASA Traffic Control | Cisco FTD Traffic Control |
| :--- | :--- | :--- |
| **主要制御コンポーネント** | Access-list (`access-group`)、Security Level、MPF (`class-map`/`policy-map`) | Prefilter Policy、Access Control Policy (ACP)、QoS Policy |
| **処理評価方式** | Top-down (First-match) の ACL 評価 ＋ MPF フロー評価 | Prefilter (LINA) ➔ Security Intelligence ➔ ACP (Top-down) ➔ Snort IPS |
| **インスペクション回避方式** | ACL permit (Stateful 許可) / MPF `sysopt connection permit-vpn` 等 | **Prefilter Fastpath** (Snort 完全バイパス) / **ACP Trust** (DPI のみバイパス) |
| **標準のデフォルトアクション** | 高 ➔ 低: Permit<br>低 ➔ 高: Implicit Deny | **Block All Traffic** (NGFW) または **Intrusion Prevention** (NGIPS) |
| **オブジェクト展開制御** | 手動 Object-Group 展開 | **`object-group-search`** (メモリ削減・ルール評価高速化) |
| **L7 アプリケーション識別** | MPF Application Inspection (特定プロトコルのみ) | **OpenAppID Engine** (数千種類のアプリを動的識別) |
| **階層型ポリシー** | 非対応 | 対応 (Base Policy ＋ Mandatory / Default Rules) |

---

## 🏗 動作原理

### 1. Cisco FTD パケットインスペクション・パイプライン (LINA ＋ Snort Engine)

FTD において、パケットがインサイト着信してからアウトバウンド出力されるまでの、LINA（Classic Firewall Engine）と Snort（NGFW Engine）間でのトラフィック評価フロー：

```text
[ Ingress Packet ]
       │
       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ LINA Engine (Firewall Data Plane)                                                │
│                                                                                  │
│ 1. Existing Connection Lookup (FastPath State Check)                            │
│    ├── Match Existing Flow? ──► Forward Packet directly                        │
│    └── New Connection? ──► Continue Pipeline                                     │
│                                                                                  │
│ 2. Un-NAT / Egress Interface Lookup & Route Lookup                               │
│                                                                                  │
│ 3. Prefilter Policy Evaluation (L2-L4 Outer Header)                              │
│    ├── Action: Fastpath ──► Bypass ALL Snort DPI ──► Proceed to Egress / QoS    │
│    ├── Action: Block ────► DROP Packet Immediately                               │
│    └── Action: Analyze ──► Hand off to Snort Data Acquisition (DAQ)               │
└──────────────────────────────────────────────────────────────────────────────────┘
       │
       ▼ (Handed to Snort Engine via DAQ)
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Snort Engine (NGFW Inspection)                                                   │
│                                                                                  │
│ 4. Security Intelligence (IP / URL / DNS) Check ──► Blacklisted? ──► DROP       │
│                                                                                  │
│ 5. Identity Policy Check (Passive ID / Active Captive Portal)                    │
│                                                                                  │
│ 6. Access Control Policy (ACP) Rules Evaluation (Top-Down First-Match)           │
│    ├── Action: Block / Block with Reset ──► DROP Packet                          │
│    ├── Action: Trust ──► Bypass IPS/File DPI ──► Hand back to LINA               │
│    └── Action: Allow ──► Proceed to Deep Packet Inspection (DPI)                 │
│         ├── Application ID (AppID) & URL Filtering                               │
│         ├── File Policy & AMP Malware Analysis                                   │
│         └── Intrusion Policy (IPS Snort Rules)                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
       │
       ▼ (Handed back to LINA Engine)
┌──────────────────────────────────────────────────────────────────────────────────┐
│ LINA Engine (Post-Inspection Processing)                                         │
│                                                                                  │
│ 7. QoS Rate Limiting Enforcement (Policing)                                      │
│ 8. NAT IP/Port Translation Header Rewrite                                        │
│ 9. Egress Interface L2 Adjacency Lookup ──► [ Egress Packet ]                    │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

### 2. Cisco ASA パケットインスペクション・パイプライン (Security Level ＋ ACL ＋ MPF)

```text
[ Ingress Packet ] ──► [ 1. Interface ACL (`access-group`) Check ]
                             ├── Deny? ──► DROP & Log (Syslog 106023)
                             └── Permit? ──► Continue
                                      │
                                      ▼
                       [ 2. Security Level Check ]
                             ├── High ➔ Low: Allowed
                             └── Low ➔ High: Requires explicit ACL Permit
                                      │
                                      ▼
                       [ 3. Conn State Table / NAT Lookup ]
                                      │
                                      ▼
                       [ 4. Modular Policy Framework (MPF) Processing ]
                             ├── Connection Limits (Embryonic TCP Check)
                             ├── TCP Normalization / TCP Intercept
                             ├── Application Inspection Engines (HTTP, FTP, DNS, etc.)
                             └── QoS Policing / Priority Queuing
                                      │
                                      ▼
                       [ 5. Egress Route / L2 Adjacency Lookup ] ──► [ Egress Packet ]
```

---

## ⚙ 動作シーケンス

1. **パケット着信と既存セッション確認**:
   * アプライアンスの物理/論理ポートにパケットが着信。既存のステートフルセッションテーブルにマッチした場合、後続のアクセスルール評価をスキップして高速転送処理へ移行。
2. **新規セッション：Prefilter 評価フェーズ (FTD)**:
   * パケットが新規セッションの場合、LINA 上の **Prefilter Policy** を評価。
   * **`Fastpath`** ヒット時: パケットは Snort エンジンへ引き渡されず、直ちに LINA の QoS / NAT / 経路解決フェーズへジャンプ。
   * **`Analyze`** ヒット時: DAQ (Data Acquisition) インターフェイスを介して Snort エンジンへリダイレクト。
3. **Snort インスペクション ＆ Security Intelligence フェーズ (FTD)**:
   * Snort プレプロセッサが IP / MAC 解読を実行。
   * **Security Intelligence (SI)** が発動し、送信元/宛先 IP、DNS ドメイン、URL を Cisco Talos フィードおよびブラックリストと動的照合。ヒットした場合は即座に破棄。
4. **Identity ＆ Access Control Rule 評価フェーズ (FTD)**:
   * パケットの L3/L4 ヘッダー、Identity Policy (ユーザー / SGT 属性)、および最初の数パケット（AppID の判定完了まで）を評価。
   * ACP ルールを **Top-Down (Mandatory ➔ Default)** の順番で評価。
   * **`Trust`** アクション: DPI (IPS / File) をバイパスして転送。
   * **`Allow`** アクション: アプリケーション識別、URL カテゴリ、File / AMP スキャン、Snort IPS ルールセットを順次適用。
5. **ASA MPF ＆ アプリケーションインスペクション処理フェーズ (ASA)**:
   * ACL を通過したパケットに対し、`class-map` の `match access-list` または `match default-inspection-traffic` が照合。
   * `policy-map` 内のアクション（`inspect <protocol>`, `set connection timeout`, `police`）に従い、動的ピンホールの作成やヘッダー書き換えを実行。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **FTD Prefilter Policy vs Access Control Policy (ACP) の使い分け**:
   * **Prefilter Policy**: L2–L4 条件（IP、Port、VLAN、Interface）のみ対応。**Fastpath** アクションにより Snort エンジン全体の CPU 負荷を排除可能。高周波取引（HFT）、バックアップ通信、インフラ管理通信（SSH/SNMP）のバイパスに最適。
   * **ACP Trust Action**: ACP ルール内で **Trust** を選択した場合、Snort DPI (IPS/File) や Network Discovery はバイパスされるが、**Security Intelligence や Rate Limiting (QoS) は引き続き適用**される。また、ACP Trust は L7 条件（AppID, URL）やユーザー属性の照合が可能。
2. **FTD Access Control Rule の並び順（Rule Ordering）と最適化**:
   * **Block ルールを最上位に配置**: 無効なトラフィックを早期破棄（Early Drop）させ、アプライアンスのリソース浪費を防ぐ。
   * **具体的条件 (Precise) を広範な条件 (Broader) より前に配置**: 上位の曖昧なルールに誤ヒットする過誤を防ぐ。
   * **L3/L4 のみの Block / Trust は Prefilter への追い出しを検討**: 5-Tuple のみのフィルタリングは Prefilter Policy へ移動させるのが Cisco ベストプラクティス。
3. **ASA / FTD での `Object Group Search` の無効・有効化**:
   * 数千件の ACE（Access Control Entry）が存在する場合、デフォルトでは FTD が全オブジェクトを展開して大量のメモリ（RAM）を消費します。
   * **有効化**: FMC CLI / Device Settings で **`object-group-search`** を有効化すると、メモリ消費量を劇的に削減しコンパイル時間を短縮できます（※CCIE ラボ試験の要件で指定されるケースあり）。
4. **ASA Security Level ＆ `same-security-traffic` の罠**:
   * **`same-security-traffic permit inter-interface`**: 異なるインターフェイス間だが同一セキュリティレベル（例: DMZ1=50, DMZ2=50）間の通信を許可する。
   * **`same-security-traffic permit intra-interface`**: 同一インターフェイスからの入出力を許可する（ヘアピン NAT や Single-Interface VPN の必須コマンド）。
5. **ASA MPF (Modular Policy Framework) の設定順序**:
   * `access-list` ➔ `class-map` ➔ `policy-map` ➔ `service-policy` の 4 段階ステップ。
   * グローバルポリシー (`global_policy`) とインターフェイス固有ポリシーのバインド順序に注意（インターフェイス固有ポリシーが同一機能に対して優先適用される）。

---

## 🛠 設定方法

### 1. ASA CLI: Extended ACL, Object Group ＆ MPF インスペクション・接続制御

```bash
! ===== Step 1: Object Group の作成 =====
object-group network OG_INSIDE_SUBNETS
 network-object 10.1.1.0 255.255.255.0
 network-object 10.1.2.0 255.255.255.0

object-group service OG_WEB_SERVICES tcp
 port-object eq www
 port-object eq https

! ===== Step 2: Access List ＆ Access Group の適用 =====
access-list ACL_INSIDE_IN extended permit tcp object-group OG_INSIDE_SUBNETS any object-group OG_WEB_SERVICES
access-list ACL_INSIDE_IN extended deny ip any any log-input

access-group ACL_INSIDE_IN in interface inside

! 同一セキュリティレベル間通信の許可設定
same-security-traffic permit inter-interface
same-security-traffic permit intra-interface

! ===== Step 3: MPF による TCP 接続数制限 ＆ ICMP Stateful Inspection =====
access-list ACL_MPF_HTTP extended permit tcp any host 172.16.1.100 eq www

class-map CM_MPF_HTTP
 match access-list ACL_MPF_HTTP

policy-map PM_CUSTOM_POLICY
 class CM_MPF_HTTP
  set connection conn-max 1000 embryonic-conn-max 100 per-client-max 50
  inspect http
 class inspection_default
  inspect icmp                           ! ICMP ステートフルインスペクション有効化

service-policy PM_CUSTOM_POLICY interface inside
```

---

### 2. FMC GUI: FTD Prefilter Policy (Fastpath ＆ Tunnel Rules)

```text
[ Step 1: Custom Prefilter Policy の作成 ]
1. FMC UI で Policies > Access Control > Prefilter に移動。
2. "New Policy" をクリック (Name: PREFILTER_POLICY_CUSTOM)。
3. Default Action: "Analyze all tunnel traffic" (または Block)。

[ Step 2: Prefilter Rule (Fastpath) の追加 ]
1. "Add Prefilter Rule" をクリック:
   - Name: Fastpath_Management_SSH
   - Action: Fastpath
   - Protocol: TCP
   - Source Network: 10.1.100.0/24 (Management Subnet)
   - Destination Port: 22 (SSH)
2. "Add" して Save。

[ Step 3: Access Control Policy へのインボーク ＆ Deploy ]
1. Policies > Access Control で対象 ACP (例: AC_POLICY_MAIN) を編集。
2. "Advanced" タブを開き "Prefilter Policy Settings" を編集。
3. Prefilter Policy ドロップダウンから "PREFILTER_POLICY_CUSTOM" を選択。
4. Save して Deploy を実行。
```

---

### 3. FMC GUI: Access Control Policy (Mandatory Rules, AppID, URL, SGT)

```text
[ FMC Access Control Policy 構造とルール追加手順 ]
1. Policies > Access Control > Access Control を開き、対象 ACP を編集。
2. "Mandatory" セクション下に Rule 1 (Block High Risk App) を作成:
   - Name: BLOCK_HIGH_RISK_APPS
   - Action: Block with Reset
   - Applications: Filter by Risk: "High Risk" ➔ Select All ➔ Add
3. Rule 2 (Allow Web Traffic with IPS/File Inspection):
   - Name: ALLOW_CORPORATE_WEB
   - Action: Allow
   - Source Zones: Inside_Zone, Destination Zones: Outside_Zone
   - Application: HTTP, HTTPS
   - Inspection タブ:
     - Intrusion Policy: Balanced Security and Connectivity
     - File Policy: Block Malware Policy
4. ACP の Default Action: "Block All Traffic" に設定の上、Save ＆ Deploy。
```

---

### 4. FTD CLI / FMC: Object Group Search の有効化

```bash
! ===== FMC GUI からの設定 =====
! Devices > Device Management > Edit Device > Device Settings > Object Group Search: Enable

! ===== FTD CLI での有効化コマンド =====
> system support diagnostic-cli
ciscoftd(config)# object-group-search
```

---

## 🔍 検証コマンド

| 目的 | コマンド (ASA / FTD CLI) | 表示確認ポイント |
| :--- | :--- | :--- |
| **ASA ACL 適用状態 ＆ Hitcount 確認** | <code>show access-list</code> | 各 ACE の `hitcnt` 件数、コンパイル後の Hex Hash 値 |
| **FTD 内部展開済み ACL ＆ Hitcount 確認** | <code>show access-list</code> (FTD Diagnostic CLI) | `CSM_FW_ACL_` のライン番号、Rule ID、`hitcnt` 件数 |
| **Prefilter Rules 展開状態確認** | <code>show access-list \| inc PREFILTER</code> | Prefilter 由来の `trust` / `deny` ルール行 |
| **パケットフローのシミュレーション検証** | <code>packet-tracer input inside tcp 10.1.1.10 5000 172.16.1.100 80</code> | `Phase 1: FLOW-LOOKUP`, `Phase 4: ACCESS-LIST`, `Phase 13: SNORT` 判定 |
| **Snort 3 エンジンのリアルタイムルールデバッグ** | <code>system support firewall-engine-debug</code> | `Starting AC rule matching`, `Matched AC Rule ID`, `Action Allow/Block` |
| **ASA MPF サービスポリシー統計確認** | <code>show service-policy</code><br><code>show service-policy inspect http</code> | クラスマップごとのマッチパケット数、ドロップ件数、接続制限判定数 |
| **FTD QoS パケットシェーピング/制限確認** | <code>show service-policy police</code> | Rate-limited (Policed) パケット数、コンフォーム/違反判定数 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **FTD で Prefilter Fastpath ルールを作成・Deploy したが、QoS Policy や Identity Policy が効かない** | Fastpath アクションは LINA レベルでパケットを直ちに出力処理へ回すため、**QoS、Identity、SI、Snort DPI が一切適用されない仕様**である | <code>show access-list</code><br><code>packet-tracer</code> | QoS や Identity の適用が必要な通信には、Prefilter Fastpath ではなく ACP **`Trust`** アクションを使用する。 |
| **FTD ACP で特定アプリを Allow したが、接続の最初の数パケットがドロップされたり遅延する** | AppID 識別には TLS ClientHello や HTTP GET 等の最初の数パケットの観察が必要であり、初期 SYN パケット通過時に一時的な待機が発生する | <code>system support firewall-engine-debug</code> | 5-Tuple（IP / ポート）で事前に確定可能な通信は、L3/L4 ルールとして ACP の上位に配置するか Prefilter を利用する。 |
| **ASA で同一セキュリティレベルに割り当てた 2 つの DMZ 間で Ping が通らない** | `same-security-traffic permit inter-interface` がグローバルで未設定 | <code>show running-config same-security-traffic</code> | グローバル設定で **`same-security-traffic permit inter-interface`** を追加投入する。 |
| **FTD に大量の ACL ルールを配備したところ、Deploy に非常に時間がかかりメモリ（RAM）不足警告が出る** | オブジェクトグループがすべてフラットな ACE へ自動拡張展開されている | <code>show memory detail</code> | **`object-group-search`** 機能を有効化し、オブジェクトグループの自動展開を防止してメモリ消費を抑える。 |
| **ASA MPF で `inspect icmp` を追加したが、戻りの Ping 応答が破棄される** | `service-policy` が対象インターフェイスまたはグローバルに適用されていない | <code>show service-policy</code> | `service-policy <PMAP> global` または `interface <IF>` を投入してアクティベートする。 |

---

## ⚠ 制限事項

* **FTD Prefilter Policy の条件制限**:
  * Prefilter Policy は L2–L4 ヘッダー情報（IP、ポート、VLAN、Interface、Tunnel Type）のみを照合条件としてサポートします。アプリケーション名 (AppID)、URL カテゴリ、ユーザー名、SGT 名、ファイルタイプは条件として使用できません。
* **ASA MPF と FTD Service Policies の互換性**:
  * ASA CLI で記述できた複雑な MPF コマンドのすべてが FMC GUI に対応しているわけではありません。FMC GUI 未対応の ASA 機能（例: WCCP や特殊な Inspect）を FTD で適用する場合は **FlexConfig Policy** を経由する必要があります。
* **QoS Policy のアサイン制限**:
  * FTD における QoS Policy は**ルーテッドモード (Routed Mode) インターフェイスのみサポート**されます（Inline Set や Passive インターフェイス、VTI / VPN インターフェイスでは非サポート）。

---

## 🔄 他技術との関連

* **Cisco ISE (Identity Services Engine)**: pxGrid / Passive ID 経由で SGT (Security Group Tag) やユーザーアイデンティティ情報を FTD Identity Policy へ供給。
* **Snort 3 IPS Engine**: ACP ルールで `Allow` アクションが選択されたトラフィックに対し、インラインで脆弱性攻撃シグネチャ（Exploit Rules）を評価。
* **Cisco Threat Intelligence (Talos)**: Security Intelligence (SI) データベースへリアルタイムの悪意ある IP/URL/DNS 脅威フィードを供給。

---

## 🧩 比較表

### 1. FTD Prefilter Fastpath vs ACP Trust vs ACP Allow

| 比較項目 | Prefilter Fastpath | ACP Trust | ACP Allow |
| :--- | :--- | :--- | :--- |
| **評価フェーズ** | **LINA レベル** (最先頭) | Snort ACP 評価フェーズ | Snort ACP 評価フェーズ |
| **Snort Engine 通過** | **完全バイパス** (手前で分岐) | Snort に入るが DPI をバイパス | **Snort に入り全 DPI を実行** |
| **Security Intelligence**| **不適用** | **適用される** | **適用される** |
| **QoS (Rate Limiting)** | **不適用** (シェーピング不能) | **適用される** | **適用される** |
| **照合可能条件** | L2–L4 (IP, Port, VLAN) | L2–L7 (IP, AppID, URL, User, SGT) | L2–L7 (IP, AppID, URL, User, SGT) |
| **推奨用途** | HFT 高周波取引、バックアップ | 信頼された App/User 通信 | 一般的な Web・データ通信 |

### 2. ASA Security Model vs FTD Access Control Model

| 比較項目 | Cisco ASA Security Model | Cisco FTD Access Control Model |
| :--- | :--- | :--- |
| **インターフェイス信頼** | **Security Level (0–100)** による暗黙許可 | **Zone ベース** (明示的な ACP ルールが必須) |
| **アクセス制御記述** | `access-list` ＋ `access-group` | **Access Control Policy (ACP)** 階層化 |
| **L7 インスペクション** | MPF (`inspect <protocol>`) | Snort 3 Engine (AppID, URL, File, IPS) |
| **デフォルト動作** | 高 ➔ 低は標準許可、低 ➔ 高は暗黙の拒否 | デフォルトは **Block All Traffic** (全遮断) |

---

## 💡 ベストプラクティス

1. **トラフィックバイパス（DPI 回避）の最適な選定**:
   * バックアップトラフィックや大容量の DB レプリケーションなど、5-Tuple（IP/ポート）で完全に特定できる大容量通信は、ACP の Trust ではなく **Prefilter Policy の Fastpath** を使用して Snort エンジンの CPU 負荷を最小化します。
2. **`object-group-search` の常時有効化推奨**:
   * FTD のメモリ使用量を大幅に削減し、FMC からの Deploy 時間を短縮するため、新規 FTD 構築時には `object-group-search` を標準有効化します。
3. **ACP ルール整理と Block ルールの最上位配置**:
   * 不要なパケットに AppID や IPS スキャンが走るのを防ぐため、明確な拒否ルール（Block Rules）は ACP Mandatory セクションの最上部に配置してアーリー・ドロップを徹底します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA Extended ACL ＆ Object Group と Object Group Search の有効化

#### 問題
ASA1 において、ネットワークオブジェクトグループ `OG_INSIDE` (`10.1.0.0/16`) から `OG_DMZ_SERVERS` (`172.16.1.10`–`172.16.1.20`) 宛ての HTTP/HTTPS 通信を許可する ACL `ACL_INSIDE` を作成・適用せよ。また、メモリ節約のため Object Group Search を有効化せよ。

#### 要件
* `object-group-search` コマンドを投入すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
object-group network OG_INSIDE
 network-object 10.1.0.0 255.255.0.0

object-group network OG_DMZ_SERVERS
 network-object host 172.16.1.10
 network-object host 172.16.1.20

object-group service OG_WEB tcp
 port-object eq www
 port-object eq https

access-list ACL_INSIDE extended permit tcp object-group OG_INSIDE object-group OG_DMZ_SERVERS object-group OG_WEB
access-group ACL_INSIDE in interface inside

object-group-search
```

#### 検証方法
* `show access-list` を実行し、ACL 内でオブジェクトが未展開のまま保持され、`packet-tracer` で正しく `ALLOW` と判定されることを確認。

---

### 例2: ASA 同一セキュリティレベル間通信の許可設定

#### 問題
ASA1 の `DMZ1` (Security Level 50) と `DMZ2` (Security Level 50) の間、および `INSIDE` インターフェイス単一ポートでのヘアピン通信を許可せよ。

#### 要件
* `same-security-traffic` コマンドを使用すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
same-security-traffic permit inter-interface
same-security-traffic permit intra-interface
```

#### 検証方法
* `show running-config same-security-traffic` で両コマンドが設定されていることを確認。

---

### 例3: ASA MPF による HTTP アプリケーション・インスペクション ＆ 接続数制限

#### 問題
ASA1 において、Web サーバー `172.16.1.100` 宛ての HTTP トラフィックに対し、最大接続数を 500、最大未確立（Embryonic）接続数を 50 に制限し、HTTP インスペクションを適用せよ。

#### 要件
* MPF (`class-map`, `policy-map`, `service-policy`) を使用すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
access-list ACL_WEB_SERVER extended permit tcp any host 172.16.1.100 eq www

class-map CM_WEB_SERVER
 match access-list ACL_WEB_SERVER

policy-map PM_INSIDE_POLICY
 class CM_WEB_SERVER
  set connection conn-max 500 embryonic-conn-max 50
  inspect http

service-policy PM_INSIDE_POLICY interface inside
```

#### 検証方法
* `show service-policy interface inside` を実行し、接続制限パラメータおよび HTTP インスペクションのマッチ件数を確認。

---

### 例4: FTD Prefilter Policy によるバックアップトラフィックの Fastpath 化

#### 問題
FMC 管理下の FTD において、バックアップサーバー `10.200.1.50` 宛ての TCP 445 (SMB) トラフィックに対し、Prefilter Policy で Snort DPI を完全バイパスする Fastpath ルール `Fastpath_Backup` を構成せよ。

#### 要件
* Prefilter Action: `Fastpath`

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Access Control > Prefilter ➔ "New Policy" (Name: PF_POLICY_DC)
2. Add Prefilter Rule:
   - Name: Fastpath_Backup
   - Action: Fastpath
   - Destination Network: 10.200.1.50
   - Destination Port: 445 (TCP)
3. Save ➔ Policies > Access Control > ACP 編集 ➔ Advanced ➔ Prefilter Policy に "PF_POLICY_DC" をバインド ➔ Deploy。
```

#### 検証方法
* FTD Diagnostic CLI で `show access-list | inc PREFILTER` を実行し、`trust tcp any host 10.200.1.50 eq 445` ルールが挿入されていることを確認。

---

### 例5: FTD Access Control Policy による High-Risk アプリケーションのブロック

#### 問題
FMC ACP において、リスクレベル `High Risk` および `Very High Risk` に分類される全 P2P/ファイル共有アプリケーションを拒否 (`Block with Reset`) する Mandatory ルールを作成せよ。

#### 要件
* ACP Mandatory セクションの最上位に配置すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Access Control > ACP 編集。
2. Mandatory セクションで "Add Rule":
   - Name: BLOCK_HIGH_RISK_P2P
   - Action: Block with Reset
   - Applications: Filter by Risk ("High", "Very High") ➔ Add All
3. Save して Deploy。
```

#### 検証方法
* 該当アプリの通信を発生させ、FMC `Analysis > Connections > Events` で `Action: Block with Reset` として記録されることを確認。

---

### 例6: FTD Identity Policy ＆ Captive Portal による未認証ユーザーの強制認証

#### 問題
Inside ネットワークからの Web アクセスに対し、未認証ユーザーを Captive Portal 認証画面へリダイレクトさせる Identity Policy `ID_POLICY_CP` を作成し ACP へ適用せよ。

#### 要件
* Action: `Active Authentication`

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Access Control > Identity ➔ "Add Identity Policy" (Name: ID_POLICY_CP)
2. Add Rule:
   - Name: RULE_ACTIVE_AUTH
   - Action: Active Authentication
   - Source Zone: Inside_Zone
3. ACP 編集 ➔ Header の "Identity Policy" で "ID_POLICY_CP" をバインド ➔ Deploy。
```

#### 検証方法
* 未認証クライアントから Web ブラウザを起動し、Captive Portal ログイン画面が表示されることを確認。

---

### 例7: FTD QoS Policy による特定ネットワークの帯域制限 (Rate Limiting)

#### 問題
ゲストネットワーク `192.168.200.0/24` からの Outbound トラフィックに対し、ダウンロード最大 10 Mbps、アップロード最大 2 Mbps に制限する QoS Policy `QOS_GUEST` を構成せよ。

#### 要件
* Devices > QoS から設定すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Devices > QoS ➔ "New Policy" (Name: QOS_GUEST)
2. Add Rule:
   - Name: LIMIT_GUEST_BANDWIDTH
   - Source Network: 192.168.200.0/24
   - Download Limit: 10 Mbits/sec
   - Upload Limit: 2 Mbits/sec
3. Target Devices に対象 FTD を指定して Deploy。
```

#### 検証方法
* FTD CLI で `show service-policy police` を実行し、制限値超えパケットが Policed ドロップされていることを確認。

---

### 例8: ASA TCP Intercept による DoS (SYN Flood) 攻撃の緩和

#### 問題
ASA1 において、DMZ サーバー `172.16.1.100` に対する SYN Flood 攻撃を防ぐため、未確立接続数が 100 を超えた場合に ASA が応答を代行する TCP Intercept を有効化せよ。

#### 要件
* MPF `set connection embryonic-conn-max` を使用すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
access-list ACL_DMZ_SERVER extended permit tcp any host 172.16.1.100

class-map CM_DMZ_SERVER
 match access-list ACL_DMZ_SERVER

policy-map PM_PROTECT_SERVER
 class CM_DMZ_SERVER
  set connection embryonic-conn-max 100

service-policy PM_PROTECT_SERVER interface outside
```

#### 検証方法
* 接続負荷テストを実施し、`show service-policy` で Embryonic Connection 制限が発動していることを確認。

---

### 例9: FTD Custom Network Analysis Policy (NAP) のアサイン

#### 問題
特定のセンシティブなネットワークセグメント (`10.50.0.0/16`) 宛てのトラフィックに対し、より厳格な TCP 正規化とプレプロセッサ設定を行うカスタム NAP `NAP_STRICT` をアサインせよ。

#### 要件
* ACP の Custom NAP 割り当て機能を使用すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Access Control > Advanced ➔ "Network Analysis and Intrusion Policies" 編集。
2. "Custom Analysis Policy Rules" ➔ Add:
   - Network Analysis Policy: NAP_STRICT
   - Source/Destination Network: 10.50.0.0/16
3. Save して Deploy。
```

#### 検証方法
* FTD CLI で `show running-config inline` を確認し、対象セグメントに対し NAP_STRICT のプレプロセッサ定義がバインドされていることを確認。

---

### 例10: FTD FlexConfig による ASA 特有 MPF 機能 (WCCP Redirect) の実装

#### 問題
FMC GUI で直接設定できない ASA WCCP トラフィックリダイレクト機能を、FlexConfig Policy を用いて FTD へ流し込め。

#### 要件
* FlexConfig Text Object および FlexConfig Policy を使用すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Objects > Object Management > FlexConfig > Text Location ➔ "Add Text Location":
   - Name: WCCP_CONFIG
   - Insert Code:
     wccp web-cache
     wccp interface inside web-cache redirect in
2. Devices > FlexConfig ➔ 対象 FTD の FlexConfig Policy 編集。
3. User Defined 追記部分に "WCCP_CONFIG" をアサインして Deploy。
```

#### 検証方法
* FTD Diagnostic CLI で `show wccp` を実行し、WCCP リダイレクトがアクティブになっていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: FTD において、特定の大容量データ転送通信（バックアップ）に対し Prefilter Policy で `Fastpath` アクションを設定した。しかし、FMC 上で当該通信に対する QoS Rate Limiting（帯域制限）が全く機能していない。原因として最も適切なものはどれか。

* A) FTD の QoS ライセンスが不足しているため。
* B) Prefilter の Fastpath アクションは LINA レベルで Snort エンジンおよび QoS モジュールを完全にバイパスして転送するため、QoS シェーピングは適用されない。
* C) QoS Policy が ACP の Default Action に設定されていないため。
* D) バックアップソフトが暗号化通信を行っているため。

**正解**: **B**  
**解説**: **Prefilter Fastpath** アクションが適用されたパケットは、Snort ディープパケットインスペクションだけでなく、**QoS Rate Limiting エンジンも完全にバイパス** されます。QoS 制限をかけたい場合は、Prefilter Fastpath ではなく ACP の **`Trust`** アクションを使用する必要があります。

---

### 2. トラブルシュート問題

**問題**: FTD を新規導入し、5,000 件以上の ACE を含む複雑な Access Control Policy を配置したところ、FMC からの Deploy に 30 分以上かかり、FTD アプライアンス上で Memory (RAM) 枯渇アラートが発生した。ルール条件を変更せずにメモリ消費量を削減し Deploy を高速化するための設定はどれか。

* A) 全ルールを ACP から Prefilter Policy へ移動する。
* B) FTD 上で **`object-group-search`** 機能を有効化する。
* C) FTD を Transparent Mode から Routed Mode に変更する。
* D) Snort 3 エンジンを Snort 2 へダウングレードする。

**正解**: **B**  
**解説**: FTD デバイス上で **`object-group-search`** を有効化すると、アプライアンスはオブジェクトグループ内の全 IP/ポートをフラットな ACE へ展開（Expand）せず、オブジェクトグループ構造のまま内部検索テーブルを検索するため、メモリ使用量が激減しコンパイルおよび Deploy 時間が大幅に短縮されます。

---

### 3. Design 問題

**問題**: 企業ネットワークにおいて、高周波取引 (HFT) システムの超低遅延（Sub-millisecond latency）通信を確保しつつ、同一ファイアウォール上で通常の 社内 Web アクセスに対しては Snort IPS ＋ AMP マルウェアインスペクションを適用したい。FTD 上での最適な設計構成はどれか。

* A) HFT 通信に対し ACP ルールで `Allow` アクションを設定する。
* B) HFT 通信に対し ACP ルールで `Trust` アクションを設定する。
* C) HFT 通信に対し **Prefilter Policy で `Fastpath` アクション** を設定し、Web アクセスは ACP ルールで `Allow` (IPS/AMP バインド) とする。
* D) 全トラフィックに対し QoS Policy で 100 Gbps 帯域をアサインする。

**正解**: **C**  
**解説**: 最低遅延が要求される通信には、Snort エンジンへの引き渡し自体をバイパスする **Prefilter Policy の `Fastpath`** が最適です。ACP Trust では Snort DAQ への受け渡しオーバーヘッドがわずかに残ります。

---

### 4. 実装問題

**問題**: ASA1 において、同一セキュリティレベル (Security Level 50) に設定された `DMZ_A` インターフェイスと `DMZ_B` インターフェイスの間で、アクセスリストを記述することなく直で IP パケットの往来を許可するグローバルコマンドはどれか。

* A) `same-security-traffic permit intra-interface`
* B) `same-security-traffic permit inter-interface`
* C) `sysopt connection permit-vpn`
* D) `no access-group`

**正解**: **B**  
**解説**: **`same-security-traffic permit inter-interface`** コマンドを設定することで、同一セキュリティレベルを持つ異なるインターフェイス間の通信がデフォルトで許可されます（`intra-interface` は同一ポート内での折返し通信用）。

---

### 5. コンフィグ読解問題

**問題**: FTD の Access Control Policy (ACP) におけるルール評価順序の説明として正しいものはどれか。

* A) ACP ルールは Bottom-Up（下から上）に評価される。
* B) ACP ルールは Mandatory セクションのルールが上から順に評価され、一致しない場合に Default セクションのルールが評価される。
* C) ルールのアクションが Block のものは、順序に関係なく Allow より優先評価される。
* D) ACP ルールは Prefilter Policy より前に評価される。

**正解**: **B**  
**解説**: FTD の ACP ルールは **Top-Down (First-Match)** で評価され、**Mandatory セクション** のルールがすべて先に評価された後、**Default セクション** のルールが順次評価されます。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Device Configuration Guide: Access Control Policies](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Device Configuration Guide: Prefilter Policies](https://www.cisco.com/)
* [Cisco ASA Series General Operations Configuration Guide: Modular Policy Framework (MPF)](https://www.cisco.com/)
* [Cisco Technical Notes: Optimize Access Control Policy Performance and Memory with Object Group Search](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Cisco Secure Firewall Architecture and Packet Flow (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で Traffic Control が出題された場合の構成判断チャート：
    * **「Bypass ALL inspection for ultra-low latency L3/L4 traffic」** ➔ Prefilter Policy ➔ Action: `Fastpath`。
    * **「Bypass IPS/File DPI but apply Security Intelligence and Rate Limiting」** ➔ ACP Rule ➔ Action: `Trust`。
    * **「Block traffic by Application / Risk Level」** ➔ ACP Rule ➔ Action: `Block` / `Block with Reset` ➔ App Condition。
    * **「Limit connection rate or embryonic connections on ASA」** ➔ ASA MPF ➔ `class-map` ➔ `policy-map` ➔ `set connection conn-max / embryonic-conn-max`。
    * **「Optimize FTD Memory / Deploy time」** ➔ Enable `object-group-search`。
* **図解（Prefilter vs ACP Trust vs ACP Allow パケット分岐）**:
  ```text
  [ Ingress Packet ]
          │
          ├── Matches Prefilter Fastpath? ──► YES ──► [ Direct Egress (No Snort) ]
          │
          └── No ──► [ Hand off to Snort Engine ]
                           │
                           ├── Matches ACP Block? ──► YES ──► [ DROP Packet ]
                           │
                           ├── Matches ACP Trust? ──► YES ──► [ Bypass IPS/File DPI ] ──► [ Egress ]
                           │
                           └── Matches ACP Allow? ──► YES ──► [ Run AppID, File, AMP, Snort IPS ] ──► [ Egress ]
  ```
* **注意点**:
  * ラボ試験で ACL や ACP、Prefilter を設定した後は、必ず `packet-tracer` や `show access-list`、`system support firewall-engine-debug` を実行し、意図したフェーズ・ルールでパケットがマッチし、処理されていることを確認・証明してください。
