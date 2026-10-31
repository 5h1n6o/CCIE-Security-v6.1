---
layout: default
title: 1.7.e-Botnet
nav_order: 5
parent: 1.7-Attack-detection
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.7.e Botnet

Cisco ネットワークインフラストラクチャおよびセキュリティアプライアンス（Cisco Secure Firewall FTD / FMC、Cisco ASA、Cisco Web Security Appliance [WSA]、Cisco Umbrella）における **Botnet（ボットネット）** 対策は、マルウェアに感染した内部ホスト（ボット/ゾンビ端末）が外部の C2（Command and Control：指令制御）サーバーと確立しようとする悪意ある通信（C2 チャネル、DNS トンネリング、DGA ドメイン生成アルゴリズム通信、Data Exfiltration）を、ネットワークの初期フェーズ（L3/L4 IP レベル、L7 URL レベル、および DNS レイヤー）で検知・検束・自動遮断するための包括的多層防御アーキテクチャです。

CCIE Security v6.1 Blueprint **1.7.e Botnet** では、従来の **Cisco ASA Botnet Traffic Filter (BTF)** のコンフィグレーション（動的データベース更新、DNS Snooping、Drop/Blacklist ポリシー）、**Cisco Secure Firewall Threat Defense (FTD) / FMC における Security Intelligence (SI)**（IP / URL / DNS ベースのレピュテーション制御、Custom Feeds/Lists、DNS Policy と DNS Sinkhole オブジェクト）、**Threat Intelligence Director (TID)**（STIX/TAXII 脅威インジケーター統合）、および **Cisco Umbrella（DNS Layer Security / Investigate）** を用いた Botnet C2 接続の検知・緩和・トラブルシューティング能力が多角的に評価されます。

---

## 📘 概要

* **機能概要**:
  * **Botnet（ボットネット）の定義と脅威メカニズム**: マルウェアに感染した多数の端末（ボット）が、攻撃者（ボットマスター）の意図に従って中央制御サーバー（C2 サーバー）からの命令を受信し、DDoS 攻撃、スパム配信、機密情報の窃取（Data Exfiltration）、横移動（Lateral Movement）を行うネットワーク構造。
  * **Cisco ASA Botnet Traffic Filter (BTF)**:
    * Cisco Talos / Threat Intelligence データベースから既知の悪意ある IP / ドメイン一覧を動的にダウンロード。
    * **DNS Snooping** 機能を併用し、内部クライアントが送信した DNS クエリから獲得した IP アドレスと C2 ブラックリストを照合して、C2 宛ての TCP/UDP 通信を直ちにドロップ・ドロップログ出力する。
  * **Cisco FTD / FMC Security Intelligence (SI)**:
    * アクセス制御ポリシー（ACP）の最前段（Snort ディープパケットインスペクションの前段階）で動作し、CPU / メモリリソースの浪費を防ぎながら大容量の悪意ある IP、URL、DNS ドメインへの接続を即時遮断する。
    * **DNS Policy**: 不審な DNS クエリに対し、`Block`、`Domain Not Found (NXDOMAIN)`、または **`Sinkhole`**（ボットからの通信をダミーの Sinkhole IP に強制リダイレクトして追跡・安全破棄する）アクションを適用する。
    * **Threat Intelligence Director (TID)**: 外部の脅威情報ソース（STIX/TAXII フォーマット等）から IOC（Indicators of Compromise）を直接 FMC へ取り込み、FTD へ自動展開してブラックリスト登録する。
  * **Cisco Umbrella (DNS Layer Security)**:
    * インターネット接続の手前（Recursive DNS レイヤー）で Botnet C2 ドメイン、Fast Flux ドメイン（IP を目まぐるしく変更して隠蔽する技術）、DGA（Domain Generation Algorithm）生成ドメインを機械学習アルゴリズムでリアルタイムにブロックする。
* **利用目的**:
  * 内部で既にマルウェア感染した端末が存在する場合でも、C2 サーバーとのコマンド送受信および追加のペロードダウンロード（Secondary Payload）を阻止し、実質的な被害（データ漏洩・外部攻撃の踏み台化）を防ぐ。
  * ネットワーク境界で悪意ある IP/URL/DNS 宛てのトラフィックを早期にドロップし、ファイアウォールおよび IPS エンジンのインスペクション負荷を大幅に削減する。
* **どのような場面で利用するか**:
  * 全社の外部インターネット境界（Outbound ユーザー通信および Server DMZ）。
  * 内部 LAN セグメントにおけるマルウェア感染端末の特定と隔離（DNS Sinkhole ログをセキュリティ監視システム [SIEM/SOC] に連携）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **評価コンポーネント (ASA)** | **Botnet Traffic Filter (BTF)** (`dynamic-filter database`, `dynamic-filter dns-snoop`, `dynamic-filter drop`) |
| **評価コンポーネント (FTD)** | **Security Intelligence (SI)** (IP / URL / DNS)、**DNS Policy**、**Sinkhole Object**、**Threat Intelligence Director (TID)** |
| **評価コンポーネント (クラウド/プロキシ)** | **Cisco Umbrella** (DNS Layer Security / Fast Flux / DGA 判定)、**Cisco WSA** (WBRS / CTA: Cognitive Threat Analytics) |
| **情報ソース (Feeds)** | **Cisco Talos Intelligence Feed** (自動更新)、**Custom Feeds/Lists** (HTTP/HTTPS 経由の `.txt` フラットファイル取り込み) |
| **DNS Policy アクション** | **`Allow`**、**`Block/Drop`**、**`Domain Not Found` (NXDOMAIN)**、**`Sinkhole`** (偽装 IP 返答 ＋ 接続ログ保存) |
| **動作フェーズ (FTD)** | Access Control Policy (ACP) の最前列 (Snort DPI より前段の LINA / Preprocessor レベルで高速ドロップ) |
| **ライセンス要件** | ASA: **Botnet Traffic Filter License**<br>FTD: **Threat License** (旧 Protection License)<br>Umbrella: **Umbrella SIG / DNS Security License** |
| **制限事項** | 暗号化された C2 トンネル (DoH: DNS over HTTPS 等) や直打 IP (IP-based C2) 通信に対しては、SSL Inspection や IP ベース SI の併用が必須 |

---

## 🏗 動作原理

### 1. FTD Security Intelligence ＆ DNS Sinkhole 動作フロー

FTD において、マルウェア感染した内部端末（Bot）が C2 サーバー宛てに DNS 問い合わせを行った際、Security Intelligence DNS Policy が介入して Sinkhole IP を返答し、C2 通信を安全なシンクホールに引き込んで遮断・検知するアーキテクチャ：

```text
[ Infected Internal Host (Bot) ] ───(1) DNS Query: "[suspicious link removed]" ───► [ Local/Recursive DNS Server ]
                                                                                         │
                                                                                         ├───(2) DNS Query Forwarded ───► [ FTD Firewall ]
                                                                                                                                │
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┴────────────────┐
│ FTD Security Intelligence Processing Pipeline                                                                                                  │
│                                                                                                                                                │
│ 1. DNS Policy Evaluation (Matches Botnet C2 List)                                                                                              │
│    ├── Action: Sinkhole (Sinkhole Object IP: 192.168.1.91 / ::1)                                                                              │
│    └── FTD generates a Spoofed DNS Response with IP = 192.168.1.91 (Does NOT forward query to external Internet DNS)                            │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                                                                                                │
[ Infected Internal Host (Bot) ] ◄──(3) DNS Response: "192.168.1.91" ───────────────────────────────────────────────────────────┘
       │
       ├───(4) C2 Connection Attempt (TCP 80/443 to 192.168.1.91) ───► [ FTD Firewall ]
                                                                                │
                                                                                ├── Matching Sinkhole Connection Rule
                                                                                └── Action: Block & Log (Generates Security Intelligence Event)
```

---

### 2. ASA Botnet Traffic Filter (BTF) 動作フロー

ASA BTF が Cisco サーバーから動的データベースを取得し、DNS Snooping によって不審な IP とドメインのマッピングをキャッシュして判定する動作：

```text
[ ASA Firewall ] ───(1) Dynamic DB Download (HTTP/HTTPS) ───► [ Cisco Talos Dynamic Filter Server ]
       │ (Updates Blacklist / Whitelist / Category Databases in RAM)
       │
[ Client PC ] ───(2) DNS Query: "botnet-c2.net" ───► [ ASA (DNS Snooping Enabled) ] ───► [ External DNS Server ]
                                                            │                                         │
                                                            │ ◄──(3) DNS Reply: "198.51.100.50" ──────┘
                                                            │ (Snoops Reply & Maps "198.51.100.50" to Blacklist)
       │
       ├───(4) TCP SYN to 198.51.100.50 ─────────────────► [ ASA (Dynamic Filter Policy) ]
                                                                   │
                                                                   ├── Matches Blacklisted C2 IP (Malicious)
                                                                   └── Action: DROP Packet & Generate Syslog 722012
```

---

## ⚙ 動作シーケンス

1. **脅威インテリジェンス更新フェーズ (Threat Feed Sync)**:
   * ASA / FMC は一定周期（デフォルト 1 時間等）で Cisco Talos クラウドまたはカスタム Web サーバーにアクセスし、最新の Botnet C2 IP/Domain/URL 一覧（Dynamic Database / Feeds）をダウンロード。
   * メモリ上の Fast-Lookup ハッシュテーブル（ShmDB / DNS List）に自動展開。
2. **DNS クエリ捕捉・検知フェーズ (DNS Snooping / DNS Policy)**:
   * **ASA BTF**: クライアントの DNS クエリ/レスポンスを **DNS Snooping** で受動的（Passive）に監視し、レスポンスに含まれる A レコード（IP アドレス）を Botnet Blacklist DB と動的紐付け。
   * **FTD DNS Policy**: クライアントからの DNS UDP/TCP 53 パケットをインラインインスペクション。
     * ルール条件が Botnet C2 ドメインにマッチした場合、直ちに介入。
3. **DNS レスポンス改変・遮断フェーズ (DNS Action Execution)**:
   * **Domain Not Found (NXDOMAIN)**: FTD は上位 DNS へ問合せを転送せず、クライアントに対して「存在しないドメイン」を表す RCODE 3 (NXDOMAIN) レスポンスを即座に送信。
   * **Sinkhole**: FTD はあらかじめ定義された **Sinkhole Object IP**（例: `192.168.1.91`）を A レコード応答として返却。
4. **C2 トラフィックブロック ＆ イベントログ生成フェーズ (Connection Drop)**:
   * 感染端末は指定された Sinkhole IP (`192.168.1.91`) へ C2 トラフィック（HTTP/HTTPS/Custom TCP）を送信。
   * FTD の Security Intelligence IP ルール（または ASA BTF `dynamic-filter drop`）が作動し、パケットをデータプレーン上で即時ドロップ。
   * FMC / ASA Syslog へ `Security Intelligence Event`（または Syslog 722012）が出力され、SOC アナリストは感染端末の IP アドレスを即座に特定・孤立化（Quarantine）する。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **ASA Botnet Traffic Filter (BTF) の完全コマンド体系**:
   * **`dynamic-filter database`**: データベースダウンロードの有効化（`fetch-interval` 調整）。
   * **`dynamic-filter use-database`**: ダウンロードした DB をフィルタリング評価に使用。
   * **`dynamic-filter dns-snoop detail`**: インターフェイス上での DNS スヌーピングの有効化。
   * **`dynamic-filter blacklist` / `whitelist`**: カスタム IP / ネットワークオブジェクトグループのブラックリスト/ホワイトリスト登録。
   * **`dynamic-filter drop port tcp eq 80 classify threat-level-range high high`**: MPF (`policy-map` / `class-map`) またはインターフェイス/グローバルでのドロップポリシー適用。
2. **FTD FMC DNS Policy と Sinkhole オブジェクトの構成手順**:
   * **Sinkhole Object の作成**: `Objects > Object Management > Sinkhole` ➔ IPv4 アドレス（例: `192.168.1.91`）および IPv6 アドレス（例: `::1`）を指定。**`Block and Log Connections to Sinkhole`** にチェックを入れる（実在する Sinkhole サーバーがない場合の必須要件）。
   * **DNS Policy の作成とルールアサイン**: `Policies > Access Control > DNS` ➔ ルール追加で `Botnet` / `C2` / `Malware` カテガリを選択し、Action を **`Sinkhole`**（作成した Sinkhole オブジェクトを指定）または **`Domain Not Found`** に設定。
   * **Access Control Policy へのアサイン**: ACP の Header にある **`DNS Policy`** ドロップダウンで作成した DNS Policy をバインドして Deploy。
3. **FTD Security Intelligence (SI) Custom Lists / Feeds の作成手順**:
   * **Custom Network / URL / DNS List**: フラットな `.txt` ファイル（1 行に 1 つの IP / Subnet / Domain）を作成し、`Objects > Object Management > Security Intelligence` でアップロード。
   * **Custom Feed**: 外部 HTTP/HTTPS サーバー上の `.txt` ファイル URL を登録し、更新間隔 (Update Interval) を指定。
   * **ACP の Security Intelligence タブ**: `Networks` / `URLs` タブで作成した List/Feed を `Blacklist` 列へアサイン。
4. **Threat Intelligence Director (TID) の要点**:
   * FMC を TID ポータルとして有効化し、STIX (Structured Threat Information eXpression) や TAXII (Trusted Automated eXchange of Indicator Information) 規格の脅威フィードを統合。
   * FTD へインジケーター（IP / Domain / URL / SHA-256）を配布して自動ブロック。
5. **Cisco Umbrella との役割分担・試験における問われ方**:
   * 「Block C2 communication at the DNS layer before packets leave the network without deploying local appliances」➔ **Cisco Umbrella**。
   * 「Redirect malicious C2 DNS queries to an internal non-routable IP on FTD and log the connection」➔ **FTD DNS Policy (Sinkhole Action)**。

---

## 🛠 設定方法

### 1. ASA CLI: Botnet Traffic Filter (BTF) 完全構成

```text
! ===== Step 1: DNS サーバーおよびドメイン検索の有効化 =====
dns domain-lookup OUTSIDE
dns name-server 8.8.8.8

! ===== Step 2: BTF データベースダウンロード設定 =====
dynamic-filter database
 fetch-interval 60                     ! 60 分間隔で更新
 use-database

! ===== Step 3: カスタム White/Blacklist オブジェクトグループ作成 =====
object-group network OG_BOTNET_CUSTOM_BLACKLIST
 network-object host 198.51.100.99
 network-object 203.0.113.0 255.255.255.0

object-group network OG_BOTNET_CUSTOM_WHITELIST
 network-object host 8.8.8.8

dynamic-filter blacklist
 name-group OG_BOTNET_CUSTOM_BLACKLIST

dynamic-filter whitelist
 name-group OG_BOTNET_CUSTOM_WHITELIST

! ===== Step 4: インターフェイスで DNS Snooping を有効化 =====
dynamic-filter dns-snoop detail

! ===== Step 5: 全インターフェイスで Drop ポリシーの適用 =====
dynamic-filter drop threat-level-range high high interface OUTSIDE
dynamic-filter drop threat-level-range high high interface INSIDE
```

---

### 2. FMC GUI: FTD Security Intelligence (SI) ＆ DNS Sinkhole 構成

```text
[ Step 1: Sinkhole オブジェクトの作成 ]
1. FMC UI で Objects > Object Management > Sinkhole に移動し "Add Sinkhole" をクリック。
2. パラメータ入力:
   - Name: Sinkhole_Botnet_C2
   - IPv4 Policy: 192.168.1.91
   - IPv6 Policy: ::1
   - [✓] Block and Log Connections to Sinkhole
   - Type: Botnet / Malware
3. "Save" をクリック。

[ Step 2: Custom DNS List の作成 ]
1. 端末上で "Custom_C2_Domains.txt" (1行に1ドメイン: e.g., [suspicious link removed]) を作成。
2. Objects > Object Management > Security Intelligence > DNS Lists and Feeds を開く。
3. "Add DNS List/Feed" を選択:
   - Name: Custom_Botnet_C2_List
   - Type: List
   - File: Custom_C2_Domains.txt をアップロードして Save。

[ Step 3: DNS Policy の作成とルール定義 ]
1. Policies > Access Control > DNS を開き "Add DNS Policy" (Name: DNS_POLICY_BOTNET)。
2. Rule 1 (Sinkhole Action):
   - Name: RULE_SINKHOLE_C2
   - Action: Sinkhole
   - Sinkhole Object: "Sinkhole_Botnet_C2" を選択
   - DNS Intelligence Feeds: "Cisco-DNS-Malware-Feed", "Cisco-DNS-C2-Feed" を追加
   - Custom Lists: "Custom_Botnet_C2_List" を追加
3. Rule 2 (Domain Not Found Action):
   - Name: RULE_NXDOMAIN_SUSPICIOUS
   - Action: Domain Not Found
   - Category: "Cisco-DNS-High-Risk-Domains" を追加
4. Save をクリック。

[ Step 4: Access Control Policy へのバインド ＆ Deploy ]
1. Policies > Access Control で対象 ACP を編集。
2. 上部 "DNS Policy:" フィールドで "DNS_POLICY_BOTNET" をアサイン。
3. "Security Intelligence" タブを開き、Networks / URLs 列で "Cisco-IP-Botnet-Feed" 等を Blacklist へ追加。
4. Save して Deploy を実行。
```

---

### 3. FMC GUI: Threat Intelligence Director (TID) 構成

```text
[ FMC UI 操作手順 ]
1. Intelligence > Threat Intelligence Director に移動。
2. "Sources" タブ ➔ "Add Source":
   - Name: TAXII_Botnet_Feed
   - Type: TAXII / STIX
   - URL: [suspicious link removed]
   - Poll Frequency: Every 1 Hour
3. "Elements" タブで対象 FTD アプライアンスが Publish 先として選択されていることを確認。
4. Action: "Block" に設定し、自動ルールインジェクションを完了。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (ASA / FTD CLI / FMC) | 表示確認ポイント |
| :--- | :--- | :--- |
| **ASA BTF データベース更新状態確認** | <code>show dynamic-filter database</code> | DB バージョン、エントリ数 (Malicious/Ambiguous)、最終更新日時 |
| **ASA BTF 統計 ＆ ドロップカウンター確認** | <code>show dynamic-filter statistics</code> | Classify カウンター、Drop パケット数、DNS Snooping キャッシュ数 |
| **ASA BTF ドロップログ追跡** | <code>show logging \| inc 722012</code> | `722012: Group <G> User <U> IP <IP> IPv6 <IP> Drop <TCP/UDP> connection...` |
| **FTD (Snort 3) SI カウンター確認** | <code>show statistics security-intelligence</code> | IP/URL/DNS の Block 処理件数、メモリ使用率 |
| **FTD 適用中の DNS Policy ルール確認** | <code>expert</code> ➔ <code>cat /var/sf/sidns_download/dns.rules</code> | `sinkhole`, `nxdomain`, `block` ルールの定義内容 |
| **FTD Sinkhole 接続確認 (ASP Drop)** | <code>show asp drop</code> / FMC SI Events | `the SI preprocessed`, Sinkhole IP 宛て接続のドロップ判定 |
| **クライアントからの DNS 解決テスト** | <code>nslookup botnet-c2.net</code> | **Sinkhole**: `192.168.1.91` 返答<br>**NXDOMAIN**: `*** UnKnown can't find botnet-c2.net: Non-existent domain` |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **ASA BTF で Dynamic DB のダウンロードが `Update failed` で失敗する** | ASA 上で DNS 解決 (`dns domain-lookup`) が未設定、または ASA 自体の外部 Route / HTTPS 通信がブロックされている | <code>show dynamic-filter database</code><br><code>ping www.cisco.com</code> | ASA 上で `dns domain-lookup OUTSIDE` および `dns name-server <IP>` を構成し、外部 Route を確保する。 |
| **FTD で Sinkhole ルールを構成したのに、Bot 端末がプロキシ等を経由して外部 C2 と直接通信できてしまう** | 内部 DNS サーバーがクライアントからの DNS クエリを外部へそのままフォワードしており、クライアントが直で FTD 経由して外部 DNS へ抜けている | <code>show statistics security-intelligence</code> | クライアントからの外部 DNS 直接通信 (UDP/TCP 53) を FTD ACP ルールで遮断し、正規の内部 DNS サーバー経由に強制する。 |
| **`Sinkhole` アクションを設定したが、クライアントに正常な外部 IP が返却されてしまう** | Access Control Policy のヘッダーで **DNS Policy がバインドされていない**、または Deploy が完了していない | FMC ACP 設定画面 | ACP の "DNS Policy" フィールドで対象ポリシーを確実にアサインして再 Deploy する。 |
| **FTD カスタム SI List (.txt) を更新したが、ブロックが反映されない** | SI List オブジェクトはテキストアップロード後に **ACP の Deploy 処理**（または SI 自動再適用）を行わないとデータプレーンへ反映されない | FMC Deployment 状態 | オブジェクト変更後、対象 FTD アプライアンスへ Deploy を実行する。 |
| **特定の業務 Web サイトが Botnet SI Blacklist に誤検知されて閲覧できない** | Cisco Talos フィード内での誤検知 (False Positive) | FMC SI Events | FMC の `Security Intelligence` タブで該当 IP/URL/Domain を **`Global Do-Not-Block List` (Whitelist)** へ追加する。 |

---

## ⚠ 制限事項

* **DoH (DNS over HTTPS) / DoT (DNS over TLS) による DNS 制御回避**:
  * クライアントやマルウェアが DoH (TCP 443) や DoT (TCP 853) を使用して暗号化 DNS クエリを外部公衆 DNS（`1.1.1.1` や `8.8.8.8`）へ直接送信した場合、L3/L4 の DNS インスペクションや DNS Policy は平文のドメイン名を識別できません。
  * **対策**: FTD で **SSL Inspection (Decrypt)** を有効化するか、DoH/DoT 宛て IP アドレス/ポートを ACP で直接ブロックする構成が必要です。
* **ASA BTF のメモリ制限**:
  * ASA のハードウェアモデル（特に旧型の 5505 / 5510 等）では、Botnet データベース全体のロードが RAM 容量制限により一部パージ・制限される場合があります。

---

## 🔄 他技術との関連

* **Cisco Umbrella**: クラウドベースの Recursive DNS セキュリティとして、Botnet C2、DGA、Fast Flux をエッジ手前で遮断。
* **AMP for Endpoints (Secure Endpoint)**: EDR 端末エージェントとして、Botnet マルウェア本体のプロセス起動（SHA-256）や悪意ある IP 接続（DFC: Device Flow Correlation）を端末レベルで破棄。
* **Cisco WSA (Web Security Appliance) / CTA (Cognitive Threat Analytics)**: アノマリ検知（機械学習）により、暗号化通信に隠れた Botnet C2 トラフィックの行動パターンを抽出・遮断。

---

## 🧩 比較表

### 1. ASA Botnet Traffic Filter (BTF) vs FTD Security Intelligence (SI) vs Cisco Umbrella

| 比較項目 | ASA Botnet Traffic Filter (BTF) | FTD Security Intelligence (SI) | Cisco Umbrella |
| :--- | :--- | :--- | :--- |
| **防御レイヤー** | Layer 3/4 ＋ DNS Snooping (Passive) | Layer 3/4, L7 (URL), **DNS Policy** (Inline) | **Recursive DNS Layer** (Cloud) |
| **プロキシ/介入** | 受動的 DNS スヌーピング ＋ L3/L4 Drop | **インライン DNS 応答改変** (Sinkhole/NXDOMAIN) | **クラウド DNS 解決インターセプト** |
| **Sinkhole 機能** | 非対応（L3/L4 ドロップのみ） | **対応**（Sinkhole Object による IP 偽装返答） | **対応**（Umbrella Block Page / IP リダイレクト） |
| **脅威フィード** | Cisco Dynamic Database (BTF DB) | **Cisco Talos Feed** ＋ Custom Feeds ＋ **TID** | **Cisco Umbrella Global Network / Investigate** |
| **設定ポイント** | ASA CLI: `dynamic-filter` コマンド群 | FMC GUI: `DNS Policy` / `ACP SI Tab` | Umbrella Dashboard / AnyConnect Roaming Module |

### 2. FTD DNS Policy Action: Block/Drop vs Domain Not Found vs Sinkhole

| 比較項目 | Block / Drop | Domain Not Found (NXDOMAIN) | Sinkhole |
| :--- | :--- | :--- | :--- |
| **応答メッセージ** | DNS クエリパケットを無言で破棄（タイムアウト） | DNS 応答 `RCODE 3 (NXDOMAIN)` を即座に返却 | **偽装 Sinkhole IP (例: `192.168.1.91`) を返答** |
| **クライアント挙動** | リトライを繰り返し、タイムアウトまで待機 | 「ドメインが存在しない」として即座に接続諦め | **Sinkhole IP へ接続を試行** |
| **追跡可能性 (SOC)** | 低い（DNS タイムアウトログのみ） | 中程度（DNS 拒否イベント） | **極めて高い**（Sinkhole 宛て C2 通信を IP レベルで完全捕捉） |

---

## 💡 ベストプラクティス

1. **FTD DNS Policy での `Sinkhole` アクションの標準採用**:
   * 単なる `Drop` や `NXDOMAIN` では、マルウェアが別プロトコルやフォールバック IP へ移行してしまう可能性があります。**`Sinkhole`** アクションを設定し、非ルーティング可能 IP（例: `192.168.1.91`）へ誘引することで、感染端末からの二次接続（C2 接続試行）を確実に SI イベントとして捕捉し、SOC での感染端末特定を容易にします。
2. **多層防御 (Defense-in-Depth) ラインの構築**:
   * **第 1 防御線**: Cisco Umbrella（DNS レイヤーで C2 解決を遮断）。
   * **第 2 防御線**: FTD Security Intelligence (SI) ＆ DNS Policy（境界ファイアウォールでの IP/URL/DNS 高速ブロック ＆ Sinkhole）。
   * **第 3 防御線**: AMP for Endpoints（端末レベルでの C2 ポート接続およびマルウェアプロセスブロック）。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA Botnet Traffic Filter (BTF) の基本設定と動的 DB ダウンロード

#### 問題
ASA1 において Botnet Traffic Filter を有効化せよ。Cisco サーバーから 60 分間隔で動的データベースを取得し、`OUTSIDE` インターフェイスで Threat Level `High` の C2 通信を自動ドロップせよ。

#### 要件
* DNS 解決に `8.8.8.8` を使用すること。
* `dynamic-filter` コマンドを使用すること。

#### 設定例

```text
! ===== ASA1 Configuration =====
dns domain-lookup OUTSIDE
dns name-server 8.8.8.8

dynamic-filter database
 fetch-interval 60
 use-database

dynamic-filter dns-snoop detail
dynamic-filter drop threat-level-range high high interface OUTSIDE
```

#### 検証方法
* `show dynamic-filter database` を実行し、`Last update status: successful` およびデータベースのエントリ数がロードされていることを確認。

---

### 例2: ASA BTF カスタム・ブラックリスト Network Object Group の追加

#### 問題
ASA1 BTF において、特定攻撃元 IP `198.51.100.77` および サブネット `203.0.113.0/24` を動的データベースに加えて即座にブラックリスト登録せよ。

#### 要件
* オブジェクトグループ `OG_BTF_BLACK` を作成して適用すること。

#### 設定例

```text
! ===== ASA1 Configuration =====
object-group network OG_BTF_BLACK
 network-object host 198.51.100.77
 network-object 203.0.113.0 255.255.255.0

dynamic-filter blacklist
 name-group OG_BTF_BLACK
```

#### 検証方法
* `show dynamic-filter statistics` を確認し、`Blacklist Classified` カウンターに反映されていることを確認。

---

### 例3: FMC での DNS Sinkhole オブジェクトの定義

#### 問題
FMC 上で Botnet C2 トラフィック誘引・ログ記録用の Sinkhole オブジェクト `OBJ_SINKHOLE_C2` を作成せよ。

#### 要件
* IPv4: `192.168.1.254`
* IPv6: `::1`
* 動作: `Block and Log Connections to Sinkhole`

#### 設定例

```text
[ FMC GUI Configuration Steps ]
1. Objects > Object Management > Sinkhole ➔ "Add Sinkhole"
   - Name: OBJ_SINKHOLE_C2
   - IPv4 Address: 192.168.1.254
   - IPv6 Address: ::1
   - Radio Button: Block and Log Connections to Sinkhole
   - Type: Botnet
2. "Save" をクリック。
```

#### 検証方法
* FMC の Object Management 一覧で `OBJ_SINKHOLE_C2` が正常登録されていることを確認。

---

### 例4: FTD DNS Policy の作成と Botnet C2 ドメインに対する Sinkhole 適用

#### 問題
FMC 上で DNS Policy `POLICY_DNS_BOTNET` を作成し、Cisco 提供の Botnet/Malware DNS フィードにヒットした通信に対し、`OBJ_SINKHOLE_C2` への Sinkhole アクションをアサインせよ。

#### 要件
* ACP にバインドして Deploy すること。

#### 設定例

```text
[ FMC GUI Configuration Steps ]
1. Policies > Access Control > DNS ➔ "Add DNS Policy" (Name: POLICY_DNS_BOTNET)
2. Add Rule:
   - Name: RULE_SINKHOLE_BOTNET
   - Action: Sinkhole
   - Sinkhole Object: OBJ_SINKHOLE_C2
   - DNS Intelligence Feeds: Cisco-DNS-Botnet-Feed, Cisco-DNS-Malware-Feed
3. Policies > Access Control > 対象 ACP を編集 ➔ Header の "DNS Policy" で "POLICY_DNS_BOTNET" を選択して Save ＆ Deploy。
```

#### 検証方法
* クライアント PC から該当 C2 ドメインを `nslookup` で引いた際、応答 IP として `192.168.1.254` が返却されることを確認。

---

### 例5: FTD DNS Policy での Domain Not Found (NXDOMAIN) アクション構成

#### 問題
高リスクドメイン (`Cisco-DNS-High-Risk-Domains`) 宛ての DNS クエリに対し、クライアントへ即座に `Domain Not Found (NXDOMAIN)` を応答させる DNS ルールを追加せよ。

#### 要件
* Action: `Domain Not Found`

#### 設定例

```text
[ FMC GUI Configuration Steps ]
1. POLICY_DNS_BOTNET を編集 ➔ "Add Rule"
   - Name: RULE_NXDOMAIN_RISKY
   - Action: Domain Not Found
   - DNS Intelligence Feeds: Cisco-DNS-High-Risk-Domains
2. Save して Deploy を実行。
```

#### 検証方法
* クライアント PC から対象ドメインを `nslookup` で引き、`Non-existent domain (NXDOMAIN)` 応答を受信することを確認。

---

### 例6: カスタム Botnet C2 URL リスト (.txt) の作成と Security Intelligence 適用

#### 問題
独自に特定した C2 ドメイン/URL 一覧（`c2-badsite.com`）を記載した `.txt` ファイルを FMC へアップロードし、Access Control Policy の Security Intelligence タブでブラックリスト適用せよ。

#### 要件
* Custom URL List オブジェクトを使用すること。

#### 設定例

```text
[ FMC GUI Configuration Steps ]
1. "custom_c2_urls.txt" (内容: [suspicious link removed]) を作成。
2. Objects > Object Management > Security Intelligence > URL Lists and Feeds ➔ "Add List/Feed"
   - Name: LIST_CUSTOM_C2_URLS
   - Type: List
   - File: custom_c2_urls.txt をアップロードして Save。
3. Policies > Access Control ➔ 対象 ACP 編集 ➔ "Security Intelligence" タブ ➔ "URLs"
   - Available Objects から "LIST_CUSTOM_C2_URLS" を選択し "Blacklist" 列へ追加 ➔ Deploy。
```

#### 検証方法
* クライアントから `[suspicious link removed]` へアクセスを試み、FTD 上で Snort DPI 評価前の SI レベルで接続が即時拒否されることを確認。

---

### 例7: FTD Security Intelligence (SI) 手動 IP 即時ブラックリスト登録 (Blacklist IP Now)

#### 問題
FMC の Connection Events / Intrusion Events パネルから、攻撃を行っている IP アドレス `203.0.113.222` を再 Deploy なしで即座に Global Blacklist へ登録せよ。

#### 要件
* "Blacklist IP Now" コンテキストメニュー機能を使用すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Analysis > Connections > Events を開く。
2. イベント一覧から IP アドレス "203.0.113.222" を右クリック。
3. "Blacklist IP Now" を選択。
```

#### 検証方法
* FTD CLI で `show statistics security-intelligence` を確認し、即座にブロック対象エントリが追加され、該当 IP からのトラフィックがドロップされることを確認。

---

### 例8: Threat Intelligence Director (TID) での STIX/TAXII 脅威フィード統合

#### 問題
FMC TID において、外部 TAXII サーバーからの Botnet IOC フィードを 1 時間間隔で自動ポーリングし、FTD アプライアンスへ Block アクションとして自動配置せよ。

#### 要件
* TID 機能を使用すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Intelligence > Threat Intelligence Director ➔ "Sources" タブ ➔ "Add Source"
   - Name: TAXII_BOTNET_FEED
   - Type: TAXII
   - URL: [suspicious link removed]
   - Poll Frequency: 1 Hour
2. "Elements" タブで対象 FTD を選択し、Action を "Block" に設定して Save。
```

#### 検証方法
* TID の "Indicators" タブでロードされた C2 IP/Domain の数がカウントされていることを確認。

---

### 例9: ASA BTF での特定 C2 サーバー宛て TCP/80 トラフィックの明示ドロップ設定

#### 問題
ASA1 BTF において、Threat Level が `Ambiguous` または `Malicious` と判定された通信のうち、TCP 80 ポート (HTTP C2 通信) のみをピンポイントでドロップせよ。

#### 要件
* `dynamic-filter drop` コマンドでポート指定すること。

#### 設定例

```text
! ===== ASA1 Configuration =====
dynamic-filter drop port tcp eq 80 classify threat-level-range moderate high interface OUTSIDE
```

#### 検証方法
* `show running-config dynamic-filter` を実行し、`port tcp eq 80` のドロップ条件が設定されていることを確認。

---

### 例10: FTD CLI を使用した SI ＆ DNS Policy 適用ログ (`dns.rules`) の検証

#### 問題
FTD CLI の Expert モードにログインし、現在アプライアンスの Snort エンジンに展開されている DNS Policy (`dns.rules`) の内部記述内容を直接検証せよ。

#### 要件
* `cat /var/sf/sidns_download/dns.rules` コマンドを使用すること。

#### 設定例

```bash
# ===== FTD CLI Expert Mode =====
> expert
admin@FTD:~$ cat /var/sf/sidns_download/dns.rules
```

#### 検証方法
* 出力結果内に `sinkhole` ルールや `nxdomain` ルール、およびアサインされた Sinkhole IP (`192.168.1.254`) が正しく記述されていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: FTD において以下のように Sinkhole オブジェクトおよび DNS Policy ルールを構成し、ACP にバインドして Deploy した。

* **Sinkhole Object**: `IPv4: 192.168.1.91`, `Type: Botnet`, `Block and Log Connections to Sinkhole` 有効。
* **DNS Policy Rule**: `Action: Sinkhole`, `Feeds: Cisco-DNS-Botnet-Feed`。

マルウェアに感染した内部 PC (`10.1.1.50`) が `nslookup [suspicious link removed]` を実行した際、内部 PC が受信する DNS レスポンスの A レコード IP アドレスとして正しいものはどれか。

* A) 実際の Botnet C2 サーバーの公衆 IP アドレス
* B) `192.168.1.91` (Sinkhole Object IP)
* C) `0.0.0.0`
* D) DNS レスポンスが途中でタイムアウトして返ってこない

**正解**: **B**  
**解説**: **`Sinkhole`** アクションが適用された DNS Policy ルールにヒットすると、FTD は外部 DNS へクエリを転送せず、自らが偽装 DNS 応答として設定された **Sinkhole Object IP (`192.168.1.91`)** をクライアントへ返却します。

---

### 2. トラブルシュート問題

**問題**: 管理者が FTD 上で C2 ドメインに対する Sinkhole DNS Policy を作成して Deploy した。しかし、マルウェアに感染した PC から `botnet-c2.com` 宛てに Ping を送信すると、実際の外部悪意 IP (`203.0.113.66`) に解決されて通信が外部へ到達してしまう。原因として最も適切なものはどれか。

* A) FTD の Threat License が無効であるため。
* B) Access Control Policy (ACP) のヘッダー設定で、作成した **DNS Policy がバインドされていない**。
* C) FTD の CPU 使用率が 100% に達しているため。
* D) スイッチで DAI が設定されていないため。

**正解**: **B**  
**解説**: FMC で作成した DNS Policy は、単に作成・編集しただけでは FTD アプライアンスへ適用されません。**Access Control Policy (ACP) のヘッダー設定画面にある "DNS Policy" フィールドで明示的にバインドし、Deploy を実行すること** が必須です。

---

### 3. Design 問題

**問題**: 企業ネットワークにおいて、マルウェア感染端末が外部の C2 サーバーへ送信する DNS 問い合わせを早期検知・遮断すると同時に、感染端末が C2 通信を試みた際の接続ログを SIEM（監視システム）で特定・追跡（Traceability）したい。FTD DNS Policy のアクションとして最適な選択肢はどれか。

* A) `Allow`
* B) `Domain Not Found (NXDOMAIN)`
* C) `Sinkhole` (ダミー IP 返答 ＋ 接続ブロックログ出力)
* D) `Do Not Block`

**正解**: **C**  
**解説**: **`Sinkhole`** アクションを使用すると、クライアントに特定のダミー IP（Sinkhole IP）が返却されるため、ボット端末はそのダミー IP 宛てに実際の接続を試みます。FTD はその二次接続を SI イベントとして確実にキャッチ・ログ保存するため、SOC アナリストは感染端末の IP アドレスを完璧に追跡・特定できます。

---

### 4. 実装問題

**問題**: ASA 上で Botnet Traffic Filter (BTF) を構成している。Cisco サーバーから動的データベースを取得・利用し、かつ外部へ送信される DNS レスポンスの IP アドレスとブラックリストを照合・マッピングさせるために設定すべき必須コマンドはどれか。

* A) `ip verify reverse-path`
* B) `dynamic-filter dns-snoop detail`
* C) `threat-detection scanning-threat shun`
* D) `ip tcp intercept`

**正解**: **B**  
**解説**: ASA BTF において、パッシングする DNS レスポンスの A レコードをリアルタイムで受動キャッチしてブラックリストと動的バインドさせるには、**`dynamic-filter dns-snoop detail`** の設定が不可欠です。

---

### 5. コンフィグ読解問題

**問題**: FTD の Security Intelligence (SI) 機能に関する説明として正しいものはどれか。

* A) SI ルールは Snort IPS ディープパケットインスペクション（DPI）の評価が完了した後に実行される。
* B) SI ルールは Access Control Policy の初期フェーズ（Snort DPI の前段）で評価されるため、悪意ある IP/URL/DNS 通信を低 CPU 負荷で高速ドロップできる。
* C) SI 機能を使用するには SSL Decryption が必須である。
* D) SI ではカスタムのテキストファイル (.txt) をブラックリストとして読み込むことはできない。

**正解**: **B**  
**解説**: **Security Intelligence (SI)** は、Access Control Policy (ACP) の処理フローにおいて **早期フェーズ (Early Phase)** で適用されます。Snort による高負荷な DPI インスペクションに入る手前で悪意ある IP/URL/DNS をドロップするため、アプライアンスの全体パフォーマンスを大幅に向上させます。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Configuration Guide: Security Intelligence Policies](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Configuration Guide: Domain Name System (DNS) Policies and Sinkholes](https://www.cisco.com/)
* [Cisco ASA Series Configuration Guide: Botnet Traffic Filter](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot Threat Intelligence Director (TID) on FMC](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Firepower Security Intelligence and Threat Defense (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で Botnet 対策・Security Intelligence が出題された場合の構成選択フローチャート：
    * **「Block Botnet C2 IP/URL using Talos dynamic feeds」** ➔ FMC ACP ➔ `Security Intelligence` タブ ➔ IP/URL を Blacklist アサイン。
    * **「Redirect Botnet DNS C2 queries to a non-routable IP and log the connection」** ➔ FMC `Objects > Sinkhole` 作成 ➔ FMC `Policies > DNS Policy` ➔ Rule Action: `Sinkhole` ➔ ACP へバインド ➔ Deploy。
    * **「Return "Domain Not Found" for high-risk domains」** ➔ FMC DNS Policy ➔ Rule Action: `Domain Not Found (NXDOMAIN)`。
    * **「Configure Botnet Traffic Filter on ASA CLI」** ➔ `dynamic-filter database` ➔ `use-database` ➔ `dynamic-filter dns-snoop detail` ➔ `dynamic-filter drop`。
* **図解（Botnet C2 防御レイヤー マトリクス）**:
  ```text
  [ Infected Host (Bot) ]
             │
             ├── 1. Recursive DNS Layer ──► Cisco Umbrella (Cloud DNS Block)
             │
             ├── 2. Firewall Ingress ─────► FTD Security Intelligence DNS Policy (Sinkhole / NXDOMAIN)
             │                             ASA BTF DNS Snooping
             │
             ├── 3. IP / URL C2 Layer ────► FTD SI IP/URL Blacklist (Early Drop before Snort DPI)
             │                             ASA BTF Drop Policy
             │
             └── 4. Endpoint Level ───────► AMP for Endpoints (DFC / Process Isolation)
  ```
* **注意点**:
  * ラボ試験で FTD の DNS Policy や SI Custom List を設定・更新した後は、必ず対象 FTD へ **Deploy** を実施し、`nslookup` テストや FTD CLI で `cat /var/sf/sidns_download/dns.rules` / `show statistics security-intelligence` を実行して、ルールが正しく展開され、ドロップカウンターが増加することを確認してください。

---

