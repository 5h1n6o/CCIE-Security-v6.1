---
layout: default
title: 1.3.a-Application-awareness
nav_order: 1
parent: 1.3-Security-features-on-IOS
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.3.a Application awareness

Cisco IOS および Cisco IOS XE ソフトウェアにおける **Application Awareness（アプリケーション識別・可視化技術）** は、伝統的な L3/L4（IP アドレスおよび TCP/UDP ポート番号）に依拠した静的アクセスポリシーの限界を克服し、パケットの L4〜L7 ペイロード内容をディープパケットインスペクション（DPI: Deep Packet Inspection）することによって、動的ポート通信や Web トンネリング（HTTP/HTTPS 暗号化通信含む）を利用するアプリケーションを正確に特定・分類・制御・可視化するコア技術群です。

CCIE Security v6.1 Blueprint 1.3.a では、**NBAR2 (Next Generation Network-Based Application Recognition)**、**Cisco AVC (Application Visibility and Control)**、**Flexible NetFlow (FNF) / IPFIX** との高度な連携、カスタムプロトコル（Custom Application）定義、プロトコルパック（Protocol Pack）管理、および QoS（CBWFQ/Policing/Marking）や ZBFW (Zone-Based Policy Firewall) との統合実装に関する実践的スキルが厳格に評価されます。

---

## 📘 概要

* **機能概要**:
  * **NBAR2 (Next Generation NBAR)**: L3〜L7 のパケット属性、ヒューリスティクス解析、ソケットキャッシュ、ホスト名/URL パターン、およびサブポート分類（HTTP Host, URL, User-Agent, Server Name Indication: SNI 等）を駆使し、1,400 種類以上のアプリケーションおよびプロトコルを高精度に動的識別する識別エンジン。
  * **Cisco AVC (Application Visibility and Control)**: NBAR2（識別）、Flexible NetFlow / IPFIX（可視化・メトリクス出力）、MQC QoS（制御・帯域保証・ドロップ）、および Cisco DNA Center / Stealthwatch（管理・解析）の 4 つの要素をシームレスに統合したトラフィック制御・可視化ソリューション。
  * **Protocol Pack (PP)**: IOS/IOS XE アプリケーションライブラリを OS のリロード（再起動）なしで動的に更新・拡張可能にする独立シグネチャファイル（`.pack`）。
* **利用目的**:
  * ポート 80/443（HTTP/HTTPS）に隠蔽・集約された SaaS アプリケーション（Microsoft 365, Salesforce, Zoom, YouTube 等）や P2P/ファイル共有通信の動的分類。
  * 業務クリティカルなアプリケーションに対する帯域幅の動的保証（CBWFQ）および非業務通信（Web ゲイム、P2P 等）の帯域制限（Policing）や完全ブロック（Drop）。
  * IPFIX / NetFlow v9 経由によるアプリケーション別帯域消費量、応答時間、ジッター、パケットロスの可視化とセキュリティアノマリ検知。
* **どのような場面で利用するか**:
  * SD-WAN / WAN エッジにおける Direct Internet Access (DIA) のトラフィック制御とアプリケーションベースルーティング。
  * 企業 LAN / WLAN エッジにおけるシャドー IT（未承認 SaaS 利用）の把握および帯域圧迫原因の特定。
  * 暗号化 Web トラフィック環境下での SNI / FQDN に基づく精緻な QoS マーキングおよびセキュリティポリシー適用。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **コアエンジン** | NBAR2 (Next Generation Network-Based Application Recognition) |
| **識別レイヤ** | Layer 3 〜 Layer 7 (IP, TCP/UDP, HTTP Header, Host/URL, SSL/TLS SNI, SIP 等) |
| **識別手法** | DPI、シグネチャマッチング、ヒューリスティクス、ソケットキャッシュ、オートラーニング、DNS-AS |
| **可視化方式** | Protocol Discovery (`ip nbar protocol-discovery`)、Flexible NetFlow (FNF) / IPFIX エクスポート |
| **制御方式** | MQC (Modular QoS CLI: Class-Map / Policy-Map)、ZBFW (Zone-Based Policy Firewall) |
| **シグネチャ更新** | NBAR2 Protocol Pack (`ip nbar protocol-pack`) によるノンストップ動的更新 |
| **カスタムプロトコル** | `ip nbar custom <NAME>` コマンドによる独自プロトコル/URL/ポート定義 |
| **主な対応プラットフォーム** | Cisco ISR 1000/4000, ASR 1000, CSR 1000v, Catalyst 8000/9000 シリーズ (IOS XE) |
| **設計上の制限事項** | SSL/TLS 完全暗号化ペイロードの内部探索限界（SNI/IP/Port 依存）、ハードウェア ASIC 転送オフロードとのトレードオフ |

---

## 🏗 動作原理

Cisco IOS XE における Application Awareness は、データプレーンのパケットパッシング処理の中で動作し、パケットの先頭数個（通常 10〜20 パケット）を詳細解析することでフローの識別情報を確定・キャッシュします。

```text
[ Ingress Packet (e.g., TCP Port 443 / SSL Session) ]
                         │
                         ▼
┌────────────────────────────────────────────────────────┐
│ 1. Fastpath / Session Lookup                           │
│    └── Conn Table Check (Existing Session Cache?)      │
│          ├── YES ──► Use Cached Application ID         │
│          └── NO  ──► Send First Packets to NBAR2 Engine│
└────────────────────────┬───────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────┐
│ 2. NBAR2 Deep Packet Inspection (DPI)                  │
│    ├── L3/L4 Header Evaluation (IP/Port/Protocol)     │
│    ├── SSL/TLS Client Hello Inspection (SNI Extract)   │
│    ├── HTTP Header Inspection (Host, URL, User-Agent)  │
│    └── Subport / Pattern Matching against Protocol Pack│
└────────────────────────┬───────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────┐
│ 3. Application Classification & Cache Update           │
│    └── Bind Application ID (e.g., "office365") to Flow │
└────────────────────────┬───────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────┐
│ 4. Policy Enforcement & Telemetry                      │
│    ├── MQC QoS Action (Marking / CBWFQ / Policing)     │
│    ├── ZBFW Action (Inspect / Drop / Pass)             │
│    └── Flexible NetFlow / IPFIX Telemetry Export       │
└────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

NBAR2 によるアプリケーション識別と AVC ポリシー適用の完全シーケンスは以下の通りです。

1. **ファーストパケットの着信とフロー生成**:
   * クライアントが通信を開始すると、ルータは 5 チュートプル（Src IP, Dst IP, Src Port, Dst Port, Protocol）に基づいて新しいフローエントリーをセッションテーブルに登録します。
2. **NBAR2 DPI エンジンへのインスペクション**:
   * フローの初期パケット群（TCP 3-way handshake および初期データ交換）が NBAR2 DPI エンジンに引き渡されます。
   * **SSL/TLS トラフィックの場合**: クライアントから送信される `Client Hello` パケット内の **Server Name Indication (SNI)** 拡張フィールドを自動抽出します。
   * **HTTP トラフィックの場合**: `GET / POST` リクエストヘッダーから `Host:`、`URL`、`User-Agent` フィールドを解析します。
3. **プロトコルパック（Protocol Pack）とのパターン照合**:
   * 抽出されたメタデータおよびヘッダー情報が、メモリ上にロードされている NBAR2 Protocol Pack 内のルールベースとマッチングされます。
4. **アプリケーション ID の決定とソケットキャッシュ登録**:
   * アプリケーション名（例: `webex-media`, `youtube`, `bittorrent`）が確定すると、その判定結果がセッションのフローキャッシュに記録されます。後続のパケットは DPI をスキップしてキャッシュされた Application ID に基づき高速処理されます。
5. **コントロールポリシー（QoS / ZBFW / FNF）の実行**:
   * `class-map` 内の `match protocol <app>` に合致したトラフィックに対し、設定されたポリシー（例: DSCP マーキング `ef`、帯域制限 `police`、ドロップ `drop`）が適用されます。
   * 同時に Flexible NetFlow モニター経由で、Application Name フィールドを含んだ IPFIX パケットがコレクターへエクスポートされます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **`ip nbar protocol-discovery` の設定場所と役割**:
   * トラフィックを可視化（統計取得）するには、対象の全インターフェイス配下で **`ip nbar protocol-discovery`** を明示的に有効化する必要があります。
   * コマンド未投入状態でも MQC `service-policy` 内で NBAR2 アプリケーションが指定されていれば分類自体は行われますが、`show ip nbar protocol-discovery` によるインターフェイス別プロトコル統計は表示されません。
2. **MQC Class-Map における `match-all` と `match-any` の使い分け**:
   * NBAR2 アプリケーションの複数指定において、**`class-map match-any`** を使用すると「指定したいずれかのアプリにマッチ」しますが、誤って **`class-map match-all`** を使用して複数の `match protocol` を並べると、「単一パケットが同時にすべてのプロトコルであること」を条件としてしまい、永久にマッチしなくなります。
3. **HTTP Sub-Classification (URL, Host, MIME) のワイルドカード構文**:
   * NBAR2 で特定の URL パターンや拡張子（例: `.exe`, `.bin`）をマッチさせる場合、特殊文字のエスケープやワイルドカード指定に厳格な文法ルールが存在します。
   * 例: `match protocol http url *.exe|*.bin|*.com`（`|` は OR 条件）。
   * パターン内に `?` を含める場合は、CLI エスケープ（`Ctrl-V` 押下後に `?`）を入力する必要があります。
4. **Custom Protocol の優先度と分類ルール**:
   * 既存の NBAR シグネチャで分類できない独自アプリや社内 Web サーバーは、`ip nbar custom <NAME>` で定義します。
   * カスタムプロトコル定義では、`http` (URL/Host/Header)、`ssl` (SNI)、`ip` (IP/Port)、`dns` (Domain) などのキーワードを指定し、Selector ID（数値）を正しく割り当てる必要があります。
5. **Flexible NetFlow (FNF) における NBAR アプリケーションフィールドの組み込み**:
   * FNF Record 内で NBAR2 アプリケーション名を出力・追跡するには、`flow record` 設定モード配下で **`match application name`** または **`collect application name`** を明示的に指定しなければなりません。
6. **NBAR2 Protocol Pack (PP) の読み込み手順**:
   * 新しい `.pack` ファイルを flash から適用するコマンドは **`ip nbar protocol-pack flash:<filename>.pack`** です。
   * 投入時にデフォルトプロトコルパックを上書き・優先化する際の挙動（`force` オプション等）および確認コマンド (`show ip nbar protocol-pack active`) を確実に把握しておきます。

---

## 🛠 設定方法

### 1. Cisco IOS XE: NBAR2 プロトコルディスカバリと MQC による P2P アプリ制限

```bash
! ===== 1. インターフェイスでの Protocol Discovery 有効化 =====
interface GigabitEthernet0/0/1
 description LAN Interface
 ip nbar protocol-discovery

interface GigabitEthernet0/0/2
 description WAN Interface
 ip nbar protocol-discovery

! ===== 2. NBAR2 アプリケーション分類 Class-Map の定義 =====
class-map match-any CM_P2P_BLOCK
 match protocol bittorrent
 match protocol edonkey
 match protocol kazaa2
 match protocol gnutella

class-map match-any CM_STREAMING_POLICE
 match protocol youtube
 match protocol netflix

! ===== 3. Policy-Map の定義 (Drop ＆ Policing) =====
policy-map PM_AVC_CONTROL
 class CM_P2P_BLOCK
  drop
 class CM_STREAMING_POLICE
  police cir 1000000 bc 31250 pir 2000000 be 62500
   conform-action transmit
   exceed-action set-dscp-transmit cs1
   violate-action drop
 class class-default
  fair-queue

! ===== 4. インターフェイスへの適用 =====
interface GigabitEthernet0/0/1
 service-policy input PM_AVC_CONTROL
```

### 2. Cisco IOS XE: NBAR2 カスタムプロトコル (Custom Protocol) の定義 (HTTP Host / SSL SNI / TCP Port)

```bash
! ===== 1. Custom HTTP Host ベースプロトコルの定義 =====
! 社内勤怠システム (Host: internal-portal.inelab.local) を "INTERNAL_PORTAL" として定義
ip nbar custom INTERNAL_PORTAL http host "internal-portal.inelab.local"

! ===== 2. Custom SSL SNI ベースプロトコルの定義 =====
! 開発クラウド (SNI: *.devcloud.cisco.com) を "DEV_CLOUD" として定義
ip nbar custom DEV_CLOUD ssl unique-name "*[suspicious link removed]"

! ===== 3. Custom TCP/UDP 複合プロトコルの定義 =====
! 独自社内アプリ (TCP ポート 10050-10060, Payload 先頭に "INELAB") を定義
ip nbar custom CUSTOM_APP_LAB tcp range 10050 10060 selector 10

! ===== 4. 作成したカスタムプロトコルの MQC ポリシー化 =====
class-map match-any CM_BUSINESS_APPS
 match protocol INTERNAL_PORTAL
 match protocol DEV_CLOUD
 match protocol CUSTOM_APP_LAB

policy-map PM_MARK_BUSINESS
 class CM_BUSINESS_APPS
  set dscp af31

interface GigabitEthernet0/0/1
 service-policy input PM_MARK_BUSINESS
```

### 3. Cisco IOS XE: Flexible NetFlow (FNF) への NBAR2 アプリケーション名組み込みと IPFIX エクスポート

```bash
! ===== 1. NBAR2 アプリケーション名を含む Flow Record の作成 =====
flow record REC_AVC_TELEMETRY
 match ipv4 source address
 match ipv4 destination address
 match transport source-port
 match transport destination-port
 match ipv4 protocol
 match application name
 collect counter bytes long
 collect counter packets long
 collect timestamp sys-uptime first
 collect timestamp sys-uptime last

! ===== 2. IPFIX Flow Exporter の作成 =====
flow exporter EXP_IPFIX_COLLECTOR
 destination 172.16.10.100
 transport udp 2055
 export-protocol ipfix
 option application-table

! ===== 3. Flow Monitor の作成とバインド =====
flow monitor MON_AVC_TELEMETRY
 record REC_AVC_TELEMETRY
 exporter EXP_IPFIX_COLLECTOR
 cache timeout active 60
 cache timeout inactive 15

! ===== 4. インターフェイスへの適用 =====
interface GigabitEthernet0/0/1
 ip flow monitor MON_AVC_TELEMETRY input
 ip flow monitor MON_AVC_TELEMETRY output
```

### 4. Cisco IOS XE: NBAR2 Protocol Pack の動的アップデート

```bash
! ===== 1. 現在適用中の Protocol Pack の確認 =====
! show ip nbar protocol-pack active

! ===== 2. Flash 上の新しい Protocol Pack ファイルの適用 =====
ip nbar protocol-pack flash:pp-VX.Y.Z-16.12-39.0.0.pack

! ===== 3. 適用状態およびバージョンの検証 =====
! show ip nbar protocol-pack active
```

---

## 🔍 検証コマンド

| 目的 | コマンド | 表示確認ポイント |
| :--- | :--- | :--- |
| **NBAR2 アクティブプロトコルパックのバージョン確認** | <code>show ip nbar protocol-pack active</code> | 適用中プロトコルパック名、バージョン番号、発行元 |
| **インターフェイスごとのリアルタイム識別統計表示** | <code>show ip nbar protocol-discovery interface <IF></code> | 受信/送信プロトコル名、パケット数、バイト数、ビットレート |
| **特定の NBAR2 プロトコル定義の詳細情報確認** | <code>show ip nbar protocol <PROTOCOL></code> | 対象プロトコルの記述、デフォルトポート番号、分類カテゴリ |
| **定義済みカスタムプロトコル一覧表示** | <code>show ip nbar custom</code> | カスタムプロトコル名、一致条件 (Host/URL/Port/SNI) |
| **MQC ポリシーにおける NBAR2 マッチパケット・ドロップ数確認** | <code>show policy-map interface <IF></code> | Class-Map 内の `Match: protocol` ごとのパケット/バイト数 |
| **Flexible NetFlow キャッシュ内の NBAR アプリケーション名表示** | <code>show flow monitor <MON_NAME> cache format table</code> | 検出された IPv4 アドレス、ポート、および `APPLICATION NAME` |
| **NBAR2 デバッグログの有効化** | <code>debug ip nbar classification</code><br><code>debug ip nbar custom</code> | アプリケーションマッチングのリアルタイムシグナル |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **`show ip nbar protocol-discovery` が何も表示されない** | 対象インターフェイス配下でコマンドが未投入 | <code>show running-config interface</code> | 該当インターフェイスで `ip nbar protocol-discovery` を有効化する。 |
| **`class-map` で `match protocol` を複数追加したが全くマッチしない** | `class-map match-all` を使用している | <code>show running-config class-map</code> | `class-map match-any` に変更して、いずれかのプロトコルでマッチするように修正する。 |
| **HTTPS Web サイト（例: YouTube）のアプリ制御が効かない** | SNI 解析が失敗しているか、QUIC (UDP 443) トラフィックを見落としている | <code>show policy-map interface</code> | UDP 443 通信を行う `quic` プロトコルも Class-Map の Match 条件に追加する。 |
| **Custom Protocol を設定したのに識別が `unknown` になる** | Selector ID の重複、または URL/Host の正規表現ミス | <code>show ip nbar custom</code> | `ip nbar custom` の構文を見直し、文字列にワイルドカード `*` を正しく付与する。 |
| **Flexible NetFlow でコレクター側にアプリ名が表示されない** | Flow Record に `match application name` が欠落、または Exporter に `option application-table` が未設定 | <code>show flow record</code><br><code>show flow exporter</code> | Flow Record に `match application name` を追加し、Exporter に `option application-table` を設定する。 |
| **Protocol Pack 適用時に「Incompatible version」エラーが発生する** | IOS XE ソフトウェアバージョンと `.pack` バージョンの不一致 | <code>show version</code><br><code>show ip nbar protocol-pack active</code> | 対象 IOS XE リリースに適合する正確な Protocol Pack ファイルを [suspicious link removed] からダウンロードして適用する。 |

---

## ⚠ 制限事項

* **暗号化トラフィック (TLS 1.3 / ESNI / Encrypted Client Hello) の制限**:
  * TLS 1.3 や Encrypted SNI (ECH) が使用されている環境では、SNI フィールドが暗号化されるため、NBAR2 は Host/SNI 解析から IP アドレス/ポート/ヒューリスティクス解析（または ETA: Encrypted Traffic Analytics）へフォールバックします。
* **パフォーマンスと CPU リソースの制限**:
  * NBAR2 はパケットのディープインスペクション（DPI）を行うため、大量の新規セッションが急増した際に Control/Data Plane CPU に大きな負荷がかかる場合があります。ハードウェア ASIC 転送メインのプラットフォームでは、NBAR 処理パケットが Punt（CPU 処理）されることに注意が必要です。
* **トンネルカプセル化パケットの制限**:
  * IPsec, GRE, VXLAN 等で二重カプセル化されたパケットに対してインナーヘッダーの NBAR2 解析を行う場合、トンネルインターフェイス側で NBAR / QoS を構成する必要があります。

---

## 🔄 他技術との関連

* **1.3.b Zone-Based Policy Firewall (ZBFW)**: ZBFW の `class-map type inspect` 内で `match protocol <app>` を指定することにより、ポート番号ではなく NBAR2 で識別した L7 アプリケーションレベルでのファイアウォール遮断/許可ポリシーを適用できます。
* **QoS (MQC: Modular QoS CLI)**: NBAR2 で分類されたトラフィッククラスに対して、帯域保証（`bandwidth`）、優先制御（`priority`）、帯域制限（`police`）、DSCP / CoS 書き換え（`set dscp`）、WRED ドロップポリシーを適用します。
* **Flexible NetFlow (FNF) / IPFIX**: FNF のフィールドとして NBAR2 アプリケーション名を埋め込み、Cisco Stealthwatch や Cisco DNA Center へテレメトリ出力します。
* **Cisco DNA Center (CBAR: Controller-Based Application Recognition)**: DNA Center から一括でデバイスの CBAR 機能を有効化し、ネットワーク全体のアプリケーションレジストリと NBAR2 プロトコルパックを自動同期させます。

---

## 🧩 比較表

### 1. NBARv1 (Legacy) vs NBAR2 (Next Generation NBAR)

| 比較項目 | NBARv1 (Legacy IOS) | NBAR2 (Cisco IOS XE) |
| :--- | :--- | :--- |
| **シグネチャファイル** | PDLM (Packet Description Language Module) | **Protocol Pack (`.pack`)** |
| **更新方式** | 個別プロトコルごとの PDLM ロード | 独立したシステム全体シグネチャパックの統合更新 |
| **識別可能アプリ数** | 数百程度 | **1,400 以上**（SaaS, Web 2.0, P2P, 暗号アプリ対応） |
| **SSL/TLS 識別** | 限定的（IP/Port 依存） | **SNI (Server Name Indication) 抽出および証明書解析** |
| **Subport 抽出** | 基本的な HTTP URL / Host | HTTP Host, URL, User-Agent, MIME, Client App, Version |
| **NetFlow 連携** | 非対応（QoS MQC のみ） | **Flexible NetFlow / IPFIX 直接連携 (`match application name`)** |

### 2. Traditional NetFlow vs Flexible NetFlow (FNF) with NBAR2

| 比較項目 | Traditional NetFlow (v5/v9) | Flexible NetFlow (FNF) with NBAR2 |
| :--- | :--- | :--- |
| **フロー識別項目** | 5 チュートプル（L3/L4 IP・Port・Protocol） | **ユーザー定義キーフィールド（L2-L7 任意オフセット）** |
| **アプリケーション識別** | TCP/UDP ポート番号からの推測のみ | **NBAR2 DPI エンジンによる完全な L7 アプリケーション名** |
| **キャッシュ構造** | 単一の標準キャッシュ | **目的別マルチキャッシュ（モニターごとに独立）** |
| **エクスポート形式** | NetFlow v5 / v9 | **NetFlow v9 / IPFIX (IETF オープン規格)** |

---

## 💡 ベストプラクティス

1. **`ip nbar protocol-discovery` のアクセス/DS レイヤでの早期適用**:
   * アプリケーション識別およびテレメトリ収集は、可能な限りネットワークのエッジ（ユーザーアクセスレイヤまたは WAN エッジ）で実行し、フローの早い段階で識別結果を固定します。
2. **`option application-table` の FNF Exporter への追加**:
   * FNF で `match application name` をコレクターへエクスポートする際、コレクター側が Application ID（数値）とアプリケーション名（文字列）のマッピングテーブルを理解できるよう、`flow exporter` 内で **`option application-table`** を必ず指定します。
3. **定期的な Protocol Pack のアップグレード管理**:
   * 新規 SaaS や Web サービスは頻繁に IP アドレスやドメインを変更します。[suspicious link removed] から定期的に最新の NBAR2 Protocol Pack（`.pack`）を取得し、無停止で更新します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: NBAR2 による基本プロトコルディスカバリと表示確認

#### 問題
ルータ R1 の `GigabitEthernet0/0/1` (LAN) および `GigabitEthernet0/0/2` (WAN) インターフェイスにおいて、NBAR2 Protocol Discovery を有効化し、双方向のトラフィックプロトコル統計を収集・表示できるように構成せよ。

#### 要件
* インターフェイスレベルで `ip nbar protocol-discovery` を設定すること。
* 設定後、リアルタイムのトッププロトコル統計を確認すること。

#### 設定例

```bash
! 1. インターフェイス配下での Protocol Discovery 有効化
interface GigabitEthernet0/0/1
 ip nbar protocol-discovery
!
interface GigabitEthernet0/0/2
 ip nbar protocol-discovery
```

#### 検証方法
* `show ip nbar protocol-discovery interface GigabitEthernet0/0/1` を実行し、Input/Output パケット数およびプロトコル一覧（`http`, `ssl`, `dns`, `icmp` 等）が表示されることを確認。

---

### 例2: NBAR2 Custom Protocol 定義 (HTTP Host ワイルドカードマッチ)

#### 問題
ルータ R1 において、社内グループウェア（ホスト名: `*.groupware.inelab.local`）への HTTP トラフィックを分類するためのカスタムプロトコル **`CUSTOM_GROUPWARE`** を定義せよ。

#### 要件
* `ip nbar custom` コマンドを使用すること。
* HTTP Host 条件でワイルドカード演算子を使用すること。

#### 設定例

```bash
! 1. HTTP Host ベースの Custom Protocol 定義
ip nbar custom CUSTOM_GROUPWARE http host "*groupware.inelab.local"

! 2. 作成したプロトコルの動作確認用 Class-Map
class-map match-all CM_CUSTOM_GW
 match protocol CUSTOM_GROUPWARE
```

#### 検証方法
* `show ip nbar custom` を実行し、`CUSTOM_GROUPWARE` が定義一覧に登録されていることを確認。
* クライアントから対象ホスト宛てに HTTP リクエストを送信し、`show ip nbar protocol-discovery` で `CUSTOM_GROUPWARE` がカウントされることを確認。

---

### 例3: NBAR2 Custom Protocol 定義 (TCP Port Range & Payload Pattern)

#### 問題
ルータ R1 において、TCP ポート `12000` 〜 `12010` を使用し、パケットペイロードの先頭に Hex パターン `0x494E45` ("INE") が含まれるカスタムアプリケーション **`LAB_APP_SEC`** を定義せよ。

#### 要件
* `selector` ID を指定して定義すること。

#### 設定例

```bash
! Custom TCP Protocol 定義
ip nbar custom LAB_APP_SEC tcp range 12000 12010 selector 15
```

#### 検証方法
* `show ip nbar protocol LAB_APP_SEC` を実行し、定義されたポート範囲と Selector ID が反映されていることを確認。

---

### 例4: NBAR2 と MQC による P2P トラフィックの完全ブロック (Drop)

#### 問題
ルータ R1 を通過する P2P アプリ通信（`bittorrent`, `edonkey`, `kazaa2`）を NBAR2 で分類し、全出入力インターフェイスで即座にドロップ（破棄）せよ。

#### 要件
* `class-map match-any` を使用すること。
* Policy-Map 内で `drop` アクションを構成すること。

#### 設定例

```bash
! 1. Class-Map の定義
class-map match-any CM_P2P_DROP
 match protocol bittorrent
 match protocol edonkey
 match protocol kazaa2

! 2. Policy-Map の定義
policy-map PM_P2P_BLOCK
 class CM_P2P_DROP
  drop

! 3. インターフェイス適用
interface GigabitEthernet0/0/1
 service-policy input PM_P2P_BLOCK
```

#### 検証方法
* クライアントから BitTorrent セッションを開始させ、通信が遮断されることを確認。
* `show policy-map interface GigabitEthernet0/0/1` を実行し、`CM_P2P_DROP` クラスの `drop` パケットカウントが増加していることを検証。

---

### 例5: NBAR2 による Web 会議 / 音声トラフィックの DSCP マーキングと帯域保証 (CBWFQ)

#### 問題
ルータ R1 において、NBAR2 を使用して `webex-media` および `ms-teams` トラフィックを識別し、DSCP 値を `EF` に書き換えた上で、WAN ポート (`GigabitEthernet0/0/2`) において 5Mbps の優先帯域（Priority Bandwidth）を確保せよ。

#### 要件
* `set dscp ef` および `priority kbps` を使用すること。

#### 設定例

```bash
! 1. Class-Map の定義
class-map match-any CM_VOICE_VIDEO
 match protocol webex-media
 match protocol ms-teams

! 2. Policy-Map の定義
policy-map PM_QOS_MARK_PRIORITY
 class CM_VOICE_VIDEO
  set dscp ef
  priority 5000
 class class-default
  fair-queue

! 3. WAN インターフェイスへの適用
interface GigabitEthernet0/0/2
 service-policy output PM_QOS_MARK_PRIORITY
```

#### 検証方法
* WebEx 通信を発生させ、`show policy-map interface GigabitEthernet0/0/2` を実行して `CM_VOICE_VIDEO` クラスでパケットがマッチし、DSCP `ef` マーキングおよび Priority キューイングが機能していることを確認。

---

### 例6: NBAR2 による動画ストリーミング (YouTube / Netflix) の帯域制限 (Policing)

#### 問題
内部ネットワークからの `youtube` および `netflix` の閲覧トラフィックに対し、NBAR2 を用いて合計 2Mbps（CIR: 2,000,000 bps）に制限（Policing）し、超過分をドロップせよ。

#### 要件
* Single-Rate Two-Color Policer を使用すること。

#### 設定例

```bash
class-map match-any CM_VIDEO_STREAMING
 match protocol youtube
 match protocol netflix

policy-map PM_POLICE_STREAMING
 class CM_VIDEO_STREAMING
  police 2000000 62500 conform-action transmit exceed-action drop

interface GigabitEthernet0/0/1
 service-policy input PM_POLICE_STREAMING
```

#### 検証方法
* クライアントから YouTube で 4K 動画を再生し、帯域が約 2Mbps に抑制されることを確認。
* `show policy-map interface GigabitEthernet0/0/1` で `exceed-action drop` カウントの増加を検証。

---

### 例7: NBAR2 Protocol Pack の更新およびロールバック操作

#### 問題
ルータ R1 の flash にアップロードされた新しい NBAR2 Protocol Pack ファイル `pp-GS-16.12-42.0.0.pack` をシステムに適用し、アクティブ化せよ。

#### 要件
* `ip nbar protocol-pack` コマンドを使用すること。

#### 設定例

```bash
! 1. Protocol Pack のロード
ip nbar protocol-pack flash:pp-GS-16.12-42.0.0.pack

! 2. 適用状態の確認
! show ip nbar protocol-pack active
```

#### 検証方法
* `show ip nbar protocol-pack active` を実行し、`Active Protocol Pack` のバージョン番号およびファイル名が指定したものに更新されていることを確認。

---

### 例8: Flexible NetFlow (FNF) における NBAR2 アプリケーション名のキャプチャとエクスポート

#### 問題
ルータ R1 において、通過する IPv4 トラフィックの送信元 IP、宛先 IP、および **NBAR2 アプリケーション名** を記録する FNF 構成を作成し、NetFlow コレクター (`10.10.10.50:2055`) へ送信せよ。

#### 要件
* Flow Record 内で `match application name` を指定すること。

#### 設定例

```bash
! 1. Flow Record の構成
flow record REC_NBAR_FNF
 match ipv4 source address
 match ipv4 destination address
 match application name
 collect counter bytes long
 collect counter packets long

! 2. Flow Exporter の構成
flow exporter EXP_NETFLOW
 destination 10.10.10.50
 transport udp 2055
 option application-table

! 3. Flow Monitor の構成
flow monitor MON_NBAR_FNF
 record REC_NBAR_FNF
 exporter EXP_NETFLOW
 cache timeout active 60

! 4. インターフェイス適用
interface GigabitEthernet0/0/1
 ip flow monitor MON_NBAR_FNF input
```

#### 検証方法
* `show flow monitor MON_NBAR_FNF cache format table` を実行し、`APPLICATION NAME` カラムに `http`, `ssl`, `dns` 等の具体的なアプリ名が表示されていることを確認。

---

### 例9: NBAR2 と Zone-Based Policy Firewall (ZBFW) の統合による L7 アプリケーション遮断

#### 問題
ZBFW が動作するルータ R1 において、`INSIDE` ゾーンから `OUTSIDE` ゾーンへの通信のうち、NBAR2 で特定された `facebook` アプリ通信のみをファイアウォールルールで遮断（Drop）し、その他の HTTP/HTTPS 通信はステートフルインスペクション（Inspect）許可せよ。

#### 要件
* ZBFW の `class-map type inspect` で `match protocol facebook` を使用すること。

#### 設定例

```bash
! 1. ZBFW Class-Map の定義
class-map type inspect match-all CM_ZBFW_FACEBOOK
 match protocol facebook

class-map type inspect match-any CM_ZBFW_WEB
 match protocol http
 match protocol https

! 2. ZBFW Policy-Map の定義
policy-map type inspect PM_ZBFW_IN_OUT
 class CM_ZBFW_FACEBOOK
  drop log
 class CM_ZBFW_WEB
  inspect
 class class-default
  drop

! 3. Zone-Pair への適用
zone-pair security ZP_IN_OUT source INSIDE destination OUTSIDE
 service-policy type inspect PM_ZBFW_IN_OUT
```

#### 検証方法
* クライアントから Facebook へのアクセスを試し、ZBFW によって接続がドロップされることを確認。
* `show policy-map type inspect zone-pair` を実行し、`CM_ZBFW_FACEBOOK` クラスで Drop カウントが増加していることを確認。

---

### 例10: NBAR2 HTTP URL パターンマッチによる特定の危険な拡張子 (.exe, .bin) ダウンロードブロック

#### 問題
ルータ R1 を通過する HTTP トラフィックにおいて、URL の末尾が `.exe` または `.bin` で終わる危険なファイルダウンロード要求を NBAR2 でキャプチャし、完全にドロップせよ。

#### 要件
* `match protocol http url` コマンドを使用すること。
* 正規表現・パイプ (`|`) による複数パターン指定を行うこと。

#### 設定例

```bash
! 1. HTTP URL ワイルドカードパターンの定義
class-map match-all CM_DANGEROUS_EXT
 match protocol http url *.exe|*.bin

! 2. Policy-Map の定義
policy-map PM_BLOCK_EXT
 class CM_DANGEROUS_EXT
  drop

! 3. インターフェイス適用
interface GigabitEthernet0/0/1
 service-policy input PM_BLOCK_EXT
```

#### 検証方法
* クライアントから `[suspicious link removed]` へアクセスを試み、接続がドロップされることを確認。
* `show policy-map interface GigabitEthernet0/0/1` で `CM_DANGEROUS_EXT` クラスのマッチ数を確認。

---

### 例11: Cisco DNA Center 連携用 CBAR (Controller-Based Application Recognition) の手動有効化

#### 問題
ルータ R1 の SPAN ポート (`GigabitEthernet0/0/3`) において、Cisco DNA Center トラフィックテレメトリアプライアンスへ送出するための CBAR ディスカバリを有効化せよ。

#### 要件
* 対象ポートで `ip nbar protocol-discovery` を構成すること。

#### 設定例

```bash
interface GigabitEthernet0/0/3
 description SPAN_TO_TELEMETRY_APPLIANCE
 ip nbar protocol-discovery
```

#### 検証方法
* `show ip nbar protocol-discovery interface GigabitEthernet0/0/3` を実行し、SPAN 受信トラフィックのアプリケーション分類統計が表示されることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ルータ R1 上で以下の NBAR2 および MQC コンフィグが適用されている。内部クライアントから外部の Web サーバー宛てに HTTP 通信を発生させた際、パケット処理結果として正しいものはどれか。

```bash
class-map match-all CM_WEB_TRAFFIC
 match protocol http
 match protocol https

policy-map PM_QOS
 class CM_WEB_TRAFFIC
  set dscp af21

interface GigabitEthernet0/0/1
 service-policy input PM_QOS
```

* A) HTTP パケットおよび HTTPS パケットの両方がマッチし、DSCP `af21` に書き換えられる。
* B) 単一のパケットが同時に HTTP かつ HTTPS であることは不可能なため、`class-map match-all` により永久にマッチせず、`class-default` で処理される。
* C) HTTP パケットのみがマッチする。
* D) NBAR2 エンジンがエラーを吐き、インターフェイスが `err-disable` に遷移する。

**正解**: **B**
**解説**: `class-map match-all` は、指定されたすべての `match` 条件を **同時に満足するパケット** のみにマッチします。HTTP (TCP 80) と HTTPS (TCP 443) は異なるプロトコルであるため、単一のパケットが両方に合致することはあり得ません。複数プロトコルのいずれかにマッチさせたい場合は **`class-map match-any`** を使用する必要があります。

---

### 2. トラブルシュート問題

**問題**: 管理者が R1 上で Flexible NetFlow (FNF) を構成し、コレクター側でトラフィックの分析を開始したが、IPFIX パケット内に IP アドレスとポート番号は表示されるものの、NBAR2 の「アプリケーション名」が空白（Unknown / ID のみ）になってしまう。`flow record` の設定は以下の通りである。原因として正しいものはどれか。

```bash
flow record REC_NETFLOW
 match ipv4 source address
 match ipv4 destination address
 match transport source-port
 match transport destination-port
 collect counter bytes
```

* A) `ip nbar protocol-discovery` が WAN ポートで無効化されているため。
* B) Flow Record 内に **`match application name`**（または `collect application name`）が定義されていないため。
* C) FNF は NetFlow v5 でしかエクスポートできないため。
* D) `ip nbar protocol-pack` がロードされていないため。

**正解**: **B**
**解説**: FNF 経由でコレクターに NBAR2 アプリケーション名を送信するには、`flow record` 定義内で **`match application name`** または **`collect application name`** を明示的に指定しなければなりません。これが欠落していると、フローデータ内にアプリケーション識別子要素が含まれません。

---

### 3. Design 問題

**問題**: 企業の WAN エッジルータ（Cisco ASR 1000 シリーズ）において、SaaS アプリケーション（Microsoft 365, Zoom 等）の急激な利用増加に伴い、特定の未識別 Web 通信の帯域圧迫原因を究明したい。ルータの再起動や IOS XE ソフトウェア全体のアップグレードを行うことなく、最新の Web アプリケーションシグネチャを動的に追加・認識させるための最適な手法はどれか。

* A) `ip nbar custom` コマンドで全 SaaS アプリの IP アドレスを個別に手動追加する。
* B) [suspicious link removed] から最新の **NBAR2 Protocol Pack (`.pack`)** を入手し、`ip nbar protocol-pack` コマンドで動的適用する。
* C) IOS XE のイメージを最新の Maintenance Release へアップグレードする。
* D) 全 Web トラフィックのインスペクションを停止する。

**正解**: **B**
**解説**: NBAR2 の最大の特徴の一つは、ルータの OS 再起動やバージョンアップを行うことなく、最新のアプリケーションシグネチャをまとめた **Protocol Pack (`.pack` ファイル)** を動的にシステムへ追加・適用できる点です。

---

### 4. 実装問題

**問題**: ルータ R1 において、特定の社内ドメイン `http://internal-portal.inelab.local` への HTTP トラフィックを識別するカスタムプロトコル **`MY_PORTAL`** を定義したい。正しい CLI コマンド記述はどれか。

* A) `ip nbar custom MY_PORTAL http host "internal-portal.inelab.local"`
* B) `class-map MY_PORTAL match http host internal-portal.inelab.local`
* C) `ip nbar protocol internal-portal.inelab.local name MY_PORTAL`
* D) `custom-protocol MY_PORTAL url internal-portal.inelab.local`

**正解**: **A**
**解説**: NBAR2 カスタムプロトコルで HTTP Host ヘッダーをマッチさせる文法は **`ip nbar custom <NAME> http host "<HOST_NAME>"`** です。

---

### 5. コンフィグ読解問題

**問題**: 以下のコンフィグがルータ R1 に適用されている。クライアントが `[suspicious link removed]` からファイルをダウンロードしようとした際、NBAR2 ポリシーによって通信はどう処理されるか。

```bash
class-map match-all CM_BLOCK_FILE
 match protocol http url *.exe|*.bin

policy-map PM_SECURITY
 class CM_BLOCK_FILE
  drop

interface GigabitEthernet0/0/1
 service-policy input PM_SECURITY
```

* A) パケットは正常に通過し、ログのみが記録される。
* B) URL パスが `.bin` にマッチするため、NBAR2 ポリシーによりパケットは即座にドロップされる。
* C) 接続は許可されるが、帯域が 8kbps に制限される。
* D) SSL トンネルへ自動的にリダイレクトされる。

**正解**: **B**
**解説**: `match protocol http url *.exe|*.bin` は、HTTP リクエストの URL 末尾が `.exe` または `.bin` であるパケットにマッチします。該当するパケットは Policy-Map 内の `drop` アクションにより遮断されます。

---

## 🔗 参考リソース

* [Cisco IOS XE 17.x Quality of Service Solutions Configuration Guide - Classifying Network Traffic Using NBAR](https://www.cisco.com/)
* [Cisco NBAR2 Protocol Library and Protocol Pack Releases](https://www.cisco.com/)
* [Cisco Application Visibility and Control (AVC) Configuration Guide for IOS XE Routers](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot NBAR2 and Custom Protocols](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Application Visibility & Control (AVC) and NBAR2 Architecture (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * NBAR2 / AVC の試験対策では、**「`ip nbar protocol-discovery` の設定」「`class-map match-any` の絶対ルール」「`ip nbar custom` の構文」「FNF への `match application name` の追加」** が合格のための必須チェックポイントとなります。
* **図解（NBAR2 による MQC / FNF / ZBFW へのデータ連携フロー）**:
  ```text
  ┌────────────────────────────────────────────────────────┐
  │                 NBAR2 Engine (DPI)                     │
  │     (Analyzes SNI, HTTP Host, URL, Proto Pack)         │
  └────────────────────────┬───────────────────────────────┘
                           │ Identifies Application ID
                           ▼
  ┌────────────────────────────────────────────────────────┐
  │              Application Classification                │
  ├────────────────────────┬───────────────────────────────┤
  │                        │                               │
  ▼                        ▼                               ▼
  [ MQC QoS Policy ]       [ Flexible NetFlow (FNF) ]      [ ZBFW Firewall ]
  - Drop / Block           - Export App Name via IPFIX     - L7 Inspect / Drop
  - CBWFQ Bandwidth        - Telemetry to Stealthwatch     - Zone-Pair Rule
  - DSCP Marking           - Telemetry to DNAC             - Application Access
  ```
* **注意点**:
  * 実機ラボ環境で HTTP/HTTPS の Match ルールをテストする際は、ブラウザのキャッシュや HTTP/2, QUIC (UDP 443) トラフィックの挙動に注意してください。事前検証時には `clear ip nbar discovery` やセッションのクリアを行ってからパケットを生成します。
