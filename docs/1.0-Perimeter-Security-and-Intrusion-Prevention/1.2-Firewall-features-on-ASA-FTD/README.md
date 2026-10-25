---
layout: default
title: 1.2-Firewall-features-on-ASA-FTD
nav_order: 2
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.2 Firewall features on Cisco ASA and FTD

Cisco ASA (Adaptive Security Appliance) および Cisco FTD (Firepower Threat Defense) における **Firewall features（ファイアウォール機能群）** は、境界防御、トラフィック制御、脅威防止、セッション変換（NAT）、高度なアプリケーション/アイデンティティ識別を実現するコアテクノロジー群です。CCIE Security v6.1 ラボ試験では、単一のセキュリティポリシー構築にとどまらず、LINA（ASAデータプレーン）と Snort 3（NGIPSエンジン）の連携動作、パケット処理パイプラインの順序、複雑な NAT 設計、アイデンティティ連携、SSL/TLS 復号、およびサービスリダイレクトの実装・検証・トラブルシューティングが深く問われます。

---

## 📘 概要

* **機能概要**:
  * **NAT (Network Address Translation)**: プライベート/パブリック IP アドレス空間の変換、ポート番号変換（PAT）、アイデンティティ NAT（No-NAT）、ポリシーベース NAT。
  * **Application Inspection (MPF / AppID)**: L3/L4 セッション制御を超えた L7 プロトコル検証（HTTP, FTP, DNS, SIP 等）および FTD アプリケーション識別（AVC / AppID）。
  * **Traffic Zones（トラフィックゾーン）**: 複数の物理/論理インターフェイスを単一のゾーンオブジェクトとしてグループ化し、非対称ルーティングの許容や負荷分散を実現。
  * **Policy-Based Routing (PBR)**: 送信元 IP、宛先 IP、プロトコル、ポート番号等の条件に基づき、標準のルーティングテーブル（FIB）をバイパスしてネクストホップを変更。
  * **Traffic Redirection (MPF / WCCP)**: 内部トラフィックを FirePOWER モジュールや外部 Web Security Appliance (WSA) 等のサービスモジュールへ透過的・明示的にリダイレクト。
  * **Identity Firewall / User Control**: IP アドレス単位ではなく、Active Directory (AD) / Cisco ISE と連携した Active/Passive パッシブ認証による「ユーザー/グループ単位」のアクセス制御。
  * **Prefilter / Access Control / Security Intelligence / SSL Policy (FTD)**: ディープパケットインスペクション前に高速ドロップ/Fastpath を行う Prefilter、Talos フィードに基づく Threat Intelligence、SSL/TLS 復号制御。
* **利用目的**:
  * トラフィック処理の効率化（不要な CPU スキャンコストの削減）。
  * アプリケーション層に潜む脅威やプロトコル違反の検出・遮断。
  * ユーザーアイデンティティに基づく細粒度のコンテキスト対応セキュリティルールの適用。
* **どのような場面で利用するか**:
  * 企業境界における複数 ISP へのトラフィック分散および特定プロトコルの個別プロキシリダイレクト。
  * 暗号化トラフィック（HTTPS/TLS 1.3）の内部スキャンとデータ漏洩（DLP）防止。
  * IP アドレスが変動する BYOD / VDI 環境でのユーザーベースのアクセス制御。

---

## 🔑 要点

| 項目 | Cisco ASA (LINA Engine) | Cisco FTD (LINA + Snort 3 Engine) |
| :--- | :--- | :--- |
| **アクセス制御基盤** | ACL (Access Control List) ＋ MPF (Modular Policy Framework) | Prefilter Policy ＋ Access Control Policy (ACP) |
| **NAT アーキテクチャ** | Section 1 (Manual), Section 2 (Auto), Section 3 (Manual) | FMC/FDM による Section 1, Section 2, Section 3 構成 |
| **L7 アプリケーション制御** | MPF 内の `inspect` コマンド（限定的 L7） | Snort 3 / AVC (Application Visibility and Control) による 4000+ アプリ制御 |
| **脅威インテリジェンス** | IP/URL の静的ブラックリスト | **Security Intelligence (SI)**: IP, Domain, URL の動的 Talos フィード |
| **ユーザー認証制御** | CDA (Context Directory Agent) / Cut-through Proxy | **Identity Policy**: ISE pxGrid / Passive ID / Captive Portal (Active Auth) |
| **暗号化トラフィック検査** | ASA SSL Premium / AIP-SSM モジュール（旧式） | **SSL Policy**: Inbound Refresh / Outbound Decrypt (TLS 1.3 対応) |
| **トラフィックリダイレクト** | MPF `policy-map` / WCCPv2 (WSA 連携) | Prefilter Fastpath / FlexConfig WCCP |
| **ポリシー優先度** | Top-Down 行単位評価 | **Prefilter → SI → SSL → Identity → ACP Rules → Default Action** |

---

## 🏗 動作原理

### Cisco FTD / ASA 統合パケット処理パイプライン (Packet Processing Pipeline)

FTD では、パケットを受信すると **LINA データプレーン (Firewall Engine)** と **Snort 3 エンジン (NGFW Engine)** の 2 つのコンポーネントが相互に連携して多層処理を実行します。

```text
[ Ingress Packet Received ]
            │
            ▼
┌────────────────────────────────────────────────────────────────────────┐
│ LINA Layer 3 / Layer 4 Engine (Firewall Engine)                        │
│  ├── 1. Ingress Interface / VLAN Decapsulation                         │
│  ├── 2. Existing Connection Table Lookup (Conn Table Match?)           │
│  │      ├── YES ───► Fastpath / Snort Stateful Inspection              │
│  │      └── NO  ───► New Connection Flow Evaluation                    │
│  ├── 3. VPN Decryption (IPsec / SSL VPN)                               │
│  ├── 4. UN-NAT / Egress Interface Determination (Route Lookup)         │
│  ├── 5. Prefilter Policy Evaluation                                    │
│  │      ├── Fastpath ──► Bypass Snort Inspection (Forward to NAT/Route) │
│  │      ├── Block    ──► Drop Packet Immediately                       │
│  │      └── Analyze  ──► Forward to Snort 3 (Data Acquisition - DAQ)   │
│  └── 6. L3/L4 Access Control List (LINA ACL)                           │
└────────────────────────────────────────────────────────────────────────┘
            │ (If Analyze)
            ▼
┌────────────────────────────────────────────────────────────────────────┐
│ Snort 3 Engine (NGFW Inspection Engine)                                │
│  ├── 7. Security Intelligence (IP Level Blacklist/Whitelist)           │
│  ├── 8. Transport / Application Layer Preprocessors (TCP Stream/DNS)  │
│  ├── 9. SSL/TLS Decryption Policy (Decrypt / Do Not Decrypt / Block)   │
│  ├── 10. Security Intelligence (DNS Domain & URL Level)                │
│  ├── 11. Identity Policy (Passive ID Match / Active Captive Portal)    │
│  ├── 12. Access Control Policy (ACP Rules: L7 App / SGT / User / URL) │
│  ├── 13. Quality of Service (QoS Rate Limiting)                        │
│  ├── 14. File Policy & AMP (SHA-256 Analysis & Sandbox)                 │
│  └── 15. Intrusion Policy (Snort IPS Rule Inspection)                 │
└────────────────────────────────────────────────────────────────────────┘
            │ (If Allowed)
            ▼
┌────────────────────────────────────────────────────────────────────────┐
│ LINA Egress Forwarding Engine                                          │
│  ├── 16. NAT Translation (IP/Port Header Rewrite)                      │
│  ├── 17. Egress Routing Lookup & PBR Evaluation                        │
│  ├── 18. VPN Encryption (If Tunnel Target)                             │
│  └── 19. L2 MAC Rewrite & Egress Port Transmission                     │
└────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

FTD および ASA の内部アルゴリズムにおける機能評価順序は、トラブルシューティングおよびポリシー設計において極めて重要です。

1. **既存セッション検索 (Connection Table Search)**:
   * 5 プレット（送信元 IP, 宛先 IP, 送信元 Port, 宛先 Port, Protocol）に一致するセッションが存在する場合、セッションテーブルのフラグに従って高速処理されます。
2. **UN-NAT (Untranslate / Reverse NAT) & Egress 判定**:
   * パケットの真の宛先 IP を確定させ、どの Egress インターフェイスから出力されるべきかをルーティングテーブルで事前評価します。
3. **Prefilter Policy 評価 (FTD)**:
   * **Fastpath**: Snort エンジン（DPI）を完全にバイパスし、LINA レベルで即座に許可。
   * **Block**: LINA レベルで即座にドロップ。
   * **Analyze**: DAQ を介して Snort エンジンへトラフィックを引き渡す。
4. **Security Intelligence (SI) 評価 (IP レベル)**:
   * Talos の動的フィードまたはカスタムリストに基づき、悪意のある送信元/宛先 IP を早期ドロップ。
5. **SSL/TLS 復号ポリシー評価**:
   * 暗号化通信の場合、証明書情報（SNI / Subject Alt Name）に基づき、復号（Decrypt - Known Key / Resign）または非復号（Do Not Decrypt）を判定。
6. **Identity Policy 評価**:
   * 送信元 IP からユーザーが未割り当ての場合、Captive Portal（Active Auth）へリダイレクトして認証を要求。
7. **Access Control Policy (ACP) ルール評価**:
   * Top-Down 順でルールを評価。`Trust`, `Allow`, `Block`, `Block with Reset`, `Interactive Block` のアクションを決定。
   * `Allow` アクションの場合、紐付けられた **File/AMP Policy** および **Intrusion Policy (IPS)** へパケットを渡して深層スキャンを実施。
8. **NAT 変換 & 送出**:
   * 許可されたトラフィックに対し、最終的な NAT 変換（送信元/宛先 IP・ポートの書き換え）を行い、L2 ヘッダを再構築して送出。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

* **NAT オーダリング (NAT Ordering) の厳密な理解**:
  * ASA / FTD の NAT ルールは、画面上の表示順序ではなく以下のルールで評価されます。
    1. **Section 1: Manual Twice NAT** (設定順または明示的ライン順)
    2. **Section 2: Auto NAT (Object NAT)** (Static が Dynamic より優先。同じ場合は IP サブネット長が長い順)
    3. **Section 3: Manual Twice NAT After Auto**
  * VPN トラフィックの例外処理（Identity NAT / No-NAT）は、必ず **Section 1 の最優先（Before Auto）** に配置する必要があります。
* **Prefilter Policy vs Access Control Policy の使い分け**:
  * 大容量データ転送（バックアップ、SAN レプリケーション、VoIP RTP 等）で CPU 負荷を下げたい場合、ACP ルールで `Trust` にするのではなく、**Prefilter Policy で `Fastpath` を適用** しなければ Snort へのパケット転送コスト（DAQ 処理）が発生してしまいます。
* **PBR (Policy-Based Routing) でのローカル通信除外設定**:
  * FMC 上で PBR を適用する際、ネクストホップへ曲げる対象の Match ACL で `permit ip any any` を指定すると、同一 FTD に接続された他のローカルサブネット宛通信までネクストホップへ飛んでしまい通信不能になります。
  * PBR の Match ACL の先頭には必ず **`deny ip any <ローカルサブネット>`** を配置しなければなりません。
* **Identity Policy と Captive Portal (Active Auth) の構成**:
  * パッシブ認証（ISE pxGrid / Passive ID）でユーザーが検出できない場合にフォールバックとして Active 認証を強制する場合、Identity Policy 内で `Use Active Authentication if Passive Identity cannot be established` にチェックを入れ、HTTP/HTTPS を Captive Portal ポート (8890) へリダイレクトする構成が問われます。
* **SSL Decryption Policy と Enterprise CA 証明書バインド**:
  * Outbound SSL Decryption (SSL Forward Proxy) を構成する場合、FTD 内の Internal CA として **エンタープライズ CA から発行された Sub-CA 証明書および秘密鍵** を FMC の `Objects > PKI > Internal CAs` へインポートし、クライアント端末側でその Root CA が信頼されている必要があります。

---

## 🛠 設定方法

### 1. Cisco ASA: MPF による Application Inspection & 接続制限構成 (CLI)

```bash
! 1. L7 HTTP Inspection Policy の作成 (不審な User-Agent や特定拡張子のブロック)
class-map type inspect http match-any CM_HTTP_BLOCK
 match request header user-agent regex ".*EvilBot.*"
 match request header uri regex ".*\.exe$"

policy-map type inspect http PM_HTTP_INSPECT
 parameters
  match CM_HTTP_BLOCK
   drop log

! 2. L3/L4 Traffic Class の作成 (DMZ Web サーバー宛通信)
class-map CM_WEB_TRAFFIC
 match access-list ACL_WEB

access-list ACL_WEB extended permit tcp any host 192.168.10.100 eq 80

! 3. MPF Policy-Map での結合と TCP SYN/Conn 制限設定
policy-map POLICY_GLOBAL
 class CM_WEB_TRAFFIC
  inspect http PM_HTTP_INSPECT
  set connection timeout embryonic 0:00:30
  set connection conn-max 1000 embryonic-conn-max 100

! 4. インターフェイスへの適用
service-policy POLICY_GLOBAL interface dmz
```

### 2. Cisco ASA: WCCPv2 による WSA への Web トラフィック自動リダイレクト構成 (CLI)

```bash
! WCCP サービスの定義 (Service ID 90: Web Traffic)
wccp 90 redirect-list ACL_WCCP_REDIRECT

access-list ACL_WCCP_REDIRECT extended permit tcp 10.1.0.0 255.255.0.0 any eq www
access-list ACL_WCCP_REDIRECT extended deny ip any any

! inside インターフェイスに入力される HTTP トラフィックをリダイレクト
wccp interface inside 90 redirect in
```

### 3. Cisco FTD (FMC GUI): Prefilter Policy の作成と ACP へのバインド

1. **Prefilter Policy の作成**:
   * **Policies > Access Control > Prefilter** へ移動し **New Policy** をクリック。
   * **Name**: `PF_FASTPATH_POLICY`。
2. **Fastpath ルールの追加**:
   * **Add Prefilter Rule** をクリック。
   * **Name**: `Fastpath_Backup_Traffic`, **Action**: `Fastpath`。
   * **Protocol**: `TCP`, **Source Network**: `10.1.100.0/24`, **Destination Port**: `445` (SMB) または `902` (vMotion)。
3. **ACP へのバインド**:
   * **Policies > Access Control** で対象の ACP を開く。
   * **More > Prefilter Policy** にて `PF_FASTPATH_POLICY` を選択して保存・Deploy。

### 4. Cisco FTD (FMC GUI): Security Intelligence (IP & Domain Feed) 構成

1. **Custom IP List オブジェクトの作成**:
   * **Objects > Object Management > Security Intelligence > Network Lists and Feeds > Add Network List/Feed**。
   * **Name**: `Custom_Blacklist_IPs`, **Type**: `List`, IP アドレスを個別に登録。
2. **ACP 内での Security Intelligence 適用**:
   * **Policies > Access Control** で対象 ACP 編集。
   * **Security Intelligence** タブを選択。
   * **Networks** サブタブ: `Custom_Blacklist_IPs` および `Cisco-IP-Block-List` を **Blacklist** カラムへ移動。
   * **Logging**: `Enable Logging` にチェックを入れて保存。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **ASA MPF ポリシー適用状態およびドロップ数確認** | ASA CLI | <code>show service-policy</code> / <code>show service-policy inspect http</code> |
| **ASA/FTD 内部セッションテーブル詳細表示** | ASA / FTD CLI | <code>show conn detail</code> / <code>show conn count</code> |
| **NAT ルール評価順序およびヒット数確認** | ASA / FTD CLI | <code>show nat detail</code> / <code>show xlate</code> |
| **WCCP リダイレクトステータスと WSA ピア確認** | ASA CLI | <code>show wccp 90 detail</code> / <code>show wccp interfaces</code> |
| **FTD Prefilter ルールヒット数確認** | FTD CLI | <code>show access-list \| include PREFILTER</code> |
| **FTD Snort 3 プロセス状態・リソース確認** | FTD CLI | <code>show process summary \| include snort</code> / <code>top</code> |
| **FTD アイデンティティキャッシュ (User-to-IP) 確認** | FTD CLI | <code>show user-identity active-user</code> / <code>show user-identity user</code> |
| **パケット処理パスシミュレーション (パケットトレーサ)** | ASA / FTD CLI | <code>packet-tracer input inside tcp 10.1.1.10 12345 8.8.8.8 80 detail</code> |
| **LINA/Snort リアルタイムデバッグ** | ASA / FTD CLI | <code>debug policy-map type inspect</code> / <code>system support firewall-engine-debug</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **特定の L7 プロトコル（SIP/FTP等）でセッションが切れる** | MPF や FTD の Default Inspection によるプロトコル違反誤検知 | ASA: <code>show service-policy inspect</code> | 該当プロトコルの `inspect` を MPF から削除するか、FTD ACP の Preprocessor パラメータを調整する。 |
| **VPN トラフィックが NAT に引っかかり対向へ届かない** | Section 1 (Manual Twice NAT) の Identity NAT が未設定 | ASA/FTD: <code>show nat</code> | Section 1 の先頭に `nat (inside,outside) source static NET_INSIDE NET_INSIDE destination static NET_REMOTE NET_REMOTE` を追加する。 |
| **PBR 設定後、ローカルサブネット宛ての通信も失敗する** | PBR Match ACL で内部宛トラフィックを除外していない | FTD: <code>show running-config route-map</code> | PBR ACL の先頭に `deny ip any 10.0.0.0 255.0.0.0` ルールを追加して内部宛通信を除外する。 |
| **FMC で設定した Security Intelligence が適用されない** | FTD に Threat ライセンスがバインドされていない | FMC: `Devices > Device Management > License` | FTD デバイスに対して `Threat` ライセンスを有効化して Deploy する。 |
| **Captive Portal の Web 認証画面が表示されない** | 8890 ポート宛のリダイレクト ACL や HTTPS 証明書の未構成 | FTD CLI: <code>show user-identity active-user</code> | FMC の `Platform Settings` にて Captive Portal ポートを有効化し、証明書を割り当てる。 |

---

## ⚠ 制限事項

* **WCCP (Web Cache Communication Protocol) の制限**:
  * ASA / FTD で WCCP を使用する場合、インバウンドリダイレクト (`wccp interface <name> <id> redirect in`) のみがサポートされ、アウトバウンドリダイレクトは非サポートです。
* **Prefilter Policy での条件制約**:
  * Prefilter ルールでは、L7 アプリケーション、URL カテゴリ、ユーザー/グループ、SGT (Security Group Tag) を条件として指定することはできません（L2/L3/L4 条件のみ）。
* **SSL/TLS 1.3 復号の制限**:
  * TLS 1.3 通信において Encrypted Client Hello (ECH) が使用されている場合、SNI や証明書情報の事前スキャンが不可能なため、フォールバックアクション（Do Not Decrypt または Block）が適用されます。

---

## 🔄 他技術との関連

* **1.1 Deployment Modes**: Routed Mode または Transparent Mode に応じて、NAT や Prefilter ルールでのゾーン/BVI インターフェイス指定方法が変化します。
* **1.3 High Availability & Clustering**: HA フェイルオーバー発生時、接続テーブル（Conn Table）および Dynamic NAT テーブル（xlate）は Standby 機器へリアルタイム同期されます。
* **2.0 Threat Defense & Security Services**: Access Control Policy から呼び出される Snort IPS ルールセットや AMP (Advanced Malware Protection) ファイルポリシーと密接に連携します。

---

## 🧩 比較表

### 1. ASA MPF Inspection vs FTD Snort 3 Inspection

| 比較項目 | Cisco ASA (MPF Inspection) | Cisco FTD (Snort 3 Inspection) |
| :--- | :--- | :--- |
| **アーキテクチャ** | LINA OS 内の C 言語ベースモジュール | 独立したマルチスレッド Snort 3 プロセス |
| **定義方法** | `class-map` ＋ `policy-map` ＋ `service-policy` | FMC GUI (ACP, Intrusion Policy, AppID) |
| **検知シグネチャ更新** | ASA OS アップグレードが必要 | Talos 脅威フィードによる **自動・日次ディープ更新** |
| **コンテキストアウェア性** | IP/Port ベース（限定的 L7） | **User, SGT, Application, Vulnerability, SSL Context** |

### 2. Prefilter Fastpath vs ACP Trust Action

| 比較項目 | Prefilter Fastpath | ACP Trust Action |
| :--- | :--- | :--- |
| **評価層** | **LINA データプレーン (L2-L4)** | **Snort 3 エンジン (DAQ 経由)** |
| **CPU 負荷** | **極めて低い** (Snort を完全バイパス) | 低い (Snort 判定後に DPI をスキップ) |
| **指定可能な条件** | L2-L4 (VLAN, Zone, IP, Port) | **L2-L7 (User, SGT, App, URL, Network)** |
| **Security Intelligence** | バイパスされる | **適用される** (IP/DNS/URL SI は評価) |

---

## 💡 ベストプラクティス

1. **Prefilter ポリシーによる低遅延処理**:
   * バックアップ通信、音声/動画 RTP ストリーム、信頼されたサイト間 GRE トンネル通信は、ACP ではなく **Prefilter Fastpath ルール** を使用して処理を高速化します。
2. **Security Intelligence の早期ブロック活用**:
   * 既知の悪意ある IP/ドメインからの攻撃は ACP ルール（IPS スキャン）に到達する前に **Security Intelligence で Blacklist 破棄** し、スキャンリソースを節約します。
3. **NAT ポリシーにおける Identity NAT の先頭配置**:
   * 拠点間 IPsec VPN 通信で使用するサブネット間の通信は、必ず **Manual Twice NAT (Section 1)** の最上部に Identity NAT (No-NAT) として配置します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA: Auto NAT (Section 2) & Manual Twice NAT (Section 1) 構成

#### 問題
ASA1 において、内部サブネット `10.1.1.0/24` から外部への通信を `outside` ポート IP で動的 PAT 変換せよ（Auto NAT）。ただし、内部 `10.1.1.0/24` から Remote 拠点 `192.168.200.0/24` 宛ての VPN 通信のみは NAT 変換を行わない（Identity NAT）設定を Section 1 に作成せよ。

#### 要件
* ASA CLI を使用すること。

#### 設定例

```bash
! 1. オブジェクトの定義
object network OBJ_INSIDE_NET
 subnet 10.1.1.0 255.255.255.0

object network OBJ_REMOTE_VPN
 subnet 192.168.200.0 255.255.255.0

! 2. Auto NAT (Section 2) の構成
object network OBJ_INSIDE_NET
 nat (inside,outside) dynamic interface

! 3. Section 1 (Manual Twice NAT) に Identity NAT (No-NAT) を配置
nat (inside,outside) section 1 source static OBJ_INSIDE_NET OBJ_INSIDE_NET destination static OBJ_REMOTE_VPN OBJ_REMOTE_VPN no-proxy-arp route-lookup
```

#### 検証方法
* `show nat` を実行し、Section 1 の Identity NAT が Section 2 の Auto NAT より上に配置されていることを確認。
* `show xlate` で VPN 通信時に PAT 変換が行われないことを確認。

---

### 例2: FTD (FMC): Auto NAT (Object NAT) & Manual NAT (Before/After Auto) 構成

#### 問題
FMC を使用して FTD1 に NAT ポリシーを作成せよ。内部 DMZ Web サーバー (`172.16.50.10`) を外部パブリック IP (`203.0.113.50`) へ Static NAT 変換（Auto NAT）せよ。また、外部特定クライアント (`198.51.100.5`) からのアクセスのみ、宛先ポート 8080 を 80 にポート変換する Twice NAT ルールを Section 1 (Before Auto) に作成せよ。

#### 要件
* FMC の Threat Defense NAT Policy 画面で設定すること。

#### 設定例

```text
1. FMC: Objects > Object Management:
   - Host Object: HOST_DMZ_WEB = 172.16.50.10
   - Host Object: HOST_PUB_WEB = 203.0.113.50
   - Host Object: HOST_EXT_CLI = 198.51.100.5
   - Port Objects: TCP_8080 (tcp/8080), TCP_80 (tcp/80)
2. Devices > NAT > New Policy > Threat Defense NAT Policy (FTD1 に割当).
3. Auto NAT (Section 2) の追加:
   - Edit Host Object: HOST_DMZ_WEB -> NAT -> Static -> Translated Addr: HOST_PUB_WEB
4. Manual NAT (Section 1) の追加:
   - Position: Before Auto NAT, Type: Static
   - Source: SZ_OUTSIDE, Destination: SZ_DMZ
   - Original Source: HOST_EXT_CLI, Translated Source: HOST_EXT_CLI (Identity)
   - Original Dest: HOST_PUB_WEB, Translated Dest: HOST_DMZ_WEB
   - Original Service: TCP_8080, Translated Service: TCP_80 -> OK
5. Save & Deploy.
```

#### 検証方法
* FTD CLI で `show nat` を実行し、Section 1 の Port Forwarding ルールと Section 2 の Object NAT ルールを確認。

---

### 例3: ASA: Modular Policy Framework (MPF) による L7 Application Inspection チュートリアル

#### 問題
ASA1 において、HTTP 通信内の URI に `.zip` または `.iso` ファイルが含まれるダウンロードリクエストを検知してドロップ・ログ出力する MPF L7 検査ポリシーを構成せよ。

#### 要件
* Regular Expression (Regex) オブジェクトを使用すること。

#### 設定例

```bash
! Regex オブジェクトの定義
regex REGEX_BLOCKED_FILES ".*\. (zip|iso)$"

! L7 Class-Map の構成
class-map type inspect http CM_BLOCK_FILE_EXT
 match request header uri regex REGEX_BLOCKED_FILES

! L7 Policy-Map の構成
policy-map type inspect http PM_BLOCK_FILE_EXT
 parameters
  match CM_BLOCK_FILE_EXT
   drop log

! L3/L4 Class-Map & Policy-Map へのバインド
class-map CM_HTTP_ALL
 match access-list ACL_HTTP_ALL

access-list ACL_HTTP_ALL extended permit tcp any any eq www

policy-map PM_GLOBAL_INSPECT
 class CM_HTTP_ALL
  inspect http PM_BLOCK_FILE_EXT

service-policy PM_GLOBAL_INSPECT global
```

#### 検証方法
* クライアントから `.zip` ファイルをダウンロード要求し、`show service-policy inspect http` でドロップカウンタの増加を確認。

---

### 例4: ASA: MPF を使用した FirePOWER モジュール / WCCP 経由 WSA トラフィックリダイレクト構成

#### 問題
ASA1 の `inside` インターフェイスを通過する全 HTTP/HTTPS 通信を、内部 WSA (`10.1.1.200`) へ WCCPv2 (Service ID 90/91) で透過リダイレクトせよ。

#### 要件
* ASA CLI で構成すること。

#### 設定例

```bash
! WCCP リダイレクト用 ACL の作成
access-list ACL_WCCP_WEB extended permit tcp 10.1.0.0 255.255.0.0 any eq www
access-list ACL_WCCP_WEB extended permit tcp 10.1.0.0 255.255.0.0 any eq 443

! WCCP グループおよびサービスの有効化
wccp 90 redirect-list ACL_WCCP_WEB
wccp 91 redirect-list ACL_WCCP_WEB

! インターフェイスへの適用
wccp interface inside 90 redirect in
wccp interface inside 91 redirect in
```

#### 検証方法
* `show wccp 90 detail` および `show wccp interfaces` を実行し、WSA との通信（Active）およびパケットリダイレクト数を確認。

---

### 例5: ASA: Traffic Zones (ゾーン間でのロードバランシングと非対称ルーティング許可)

#### 問題
ASA1 において、2 つの Outside ポート (`Gi0/1` と `Gi0/2`) を単一の Zone `ZONE_OUTSIDE` に配置せよ。同一ゾーン内の複数ポートをまたぐ非対称ルーティング（Asymmetric Routing）通信を許可せよ。

#### 要件
* Zone 割り当てを行うこと。

#### 設定例

```bash
! Traffic Zone の作成と非対称ルーティング許可
zone ZONE_OUTSIDE
 asymmetric-routing permit

! インターフェイスの Zone バインド
interface GigabitEthernet0/1
 nameif outside1
 security-level 0
 zone-member ZONE_OUTSIDE
 ip address 203.0.113.1 255.255.255.0
!
interface GigabitEthernet0/2
 nameif outside2
 security-level 0
 zone-member ZONE_OUTSIDE
 ip address 198.51.100.1 255.255.255.0
```

#### 検証方法
* `show zone` を実行し、`outside1` と `outside2` が `ZONE_OUTSIDE` に属していることを確認。
* 往路と復路が異なるポートを通過した通信のセッション（`show conn`）が維持されることを検証。

---

### 例6: FTD (FMC): Prefilter Policy と プレーンテキストトンネル (GRE/IPinIP) 再ゾーン化構成

#### 問題
FTD1 に流入する GRE トンネル通信（IP Protocol 47）を Prefilter Policy で検知し、アウターヘッダーを解凍したインナーパケットを新しい Security Zone `SZ_GRE_INNER` にタグ付け（Re-zoning）して ACP で二次判定させよ。

#### 要件
* FMC Prefilter Policy の Tunnel Rule を使用すること。

#### 設定例

```text
1. FMC: Objects > Security Zones > Add: SZ_GRE_INNER (Routed Type).
2. Policies > Access Control > Prefilter > Edit Prefilter Policy.
3. Tunnel Rules > Add Tunnel Rule:
   - Name: Rezone_GRE_Tunnel
   - Action: Analyze
   - Protocol: GRE (47)
   - Rezone To: SZ_GRE_INNER -> OK
4. ACP 内で Source Zone に SZ_GRE_INNER を指定したルールを作成してスキャン実施。
5. Save & Deploy.
```

#### 検証方法
* FTD CLI にて `show access-list` を実行し、Prefilter トンネルルールのヒットカウンタを確認。

---

### 例7: FTD (FMC): Access Control Policy (ACP) と Security Intelligence 構成

#### 問題
FTD1 の ACP において、Talos から動的配信される `Malware` および `C&C` ドメイン宛ての DNS 問い合わせを即座にドロップし、カスタム IP Blacklist (`192.0.2.0/24`) を評価する Security Intelligence ルールを有効化せよ。

#### 要件
* DNS Policy / SI タブで構成すること。

#### 設定例

```text
1. FMC: Objects > Object Management > Security Intelligence > Network Lists > Add:
   - Name: LIST_CUSTOM_BLOCK_IP, Network: 192.0.2.0/24
2. Policies > Access Control > ACP 編集 > Security Intelligence タブ:
   - Networks: LIST_CUSTOM_BLOCK_IP を Blacklist へ追加
   - DNS Policy: System-provided DNS Policy を選択 (Malware / C&C ドメインを Block アクションに指定)
3. Save & Deploy.
```

#### 検証方法
* クライアントから `192.0.2.1` 宛てに ping を送信し、FMC の `Analysis > Security Intelligence Events` でブロックイベントログを確認。

---

### 例8: FTD (FMC): Identity Policy (Passive Identity / Active Authentication) 構成

#### 問題
FTD1 において、ユーザー認証ポリシーを作成せよ。内部サブネット `10.1.1.0/24` からの HTTP アクセスに対し、まず ISE pxGrid による Passive Identity（IP-to-User ログオン情報）を照合し、不明なユーザーに対しては Captive Portal による Active Authentication（Web フォーム認証）を強制せよ。

#### 要件
* Realm 名: `AD_REALM`

#### 設定例

```text
1. FMC: Policies > Access Control > Identity > Add Identity Policy:
   - Name: ID_POLICY_CORP
2. Add Rule:
   - Action: Passive Authentication
   - Realm: AD_REALM
   - Checkbox: "Use active authentication if passive identity cannot be established" をオン
   - Active Auth Active Response: Captive Portal -> OK
3. Policies > Access Control > ACP 編集 > Identity Policy タブに `ID_POLICY_CORP` をセット。
4. Deploy を実行。
```

#### 検証方法
* ドメイン未参加端末から Web 閲覧を発生させ、Captive Portal 認証画面（8890 ポート）へリダイレクトされることを確認。
* FTD CLI で `show user-identity active-user` を実行しユーザーバインドを確認。

---

### 例9: FTD (FMC): SSL/TLS Decryption Policy (Outbound Decrypt) 構成

#### 問題
FTD1 を通過するアウトバウンド HTTPS 通信に対し、`Finance` カテゴリ以外の Web サイト閲覧トラフィックを復号スキャンする SSL Policy `SSL_OUTBOUND_DECRYPT` を作成・適用せよ。

#### 要件
* Action: `Decrypt - Resign` (Internal CA 使用)

#### 設定例

```text
1. FMC: Objects > Object Management > PKI > Internal CAs にエンタープライズ Sub-CA を登録。
2. Policies > Access Control > SSL > Add SSL Policy:
   - Name: SSL_OUTBOUND_DECRYPT, Default Action: Do Not Decrypt
3. Add Rule:
   - Name: Bypass_Finance, Action: Do Not Decrypt, Categories: Finance and Banking
4. Add Rule 2:
   - Name: Decrypt_All_Other, Action: Decrypt - Resign, Categories: Any
   - Decryption CA: インポートした Sub-CA を指定 -> OK
5. ACP 編集 > SSL Policy タブに `SSL_OUTBOUND_DECRYPT` を割り当てて Deploy。
```

#### 検証方法
* クライアント端末から一般 HTTPS サイトへアクセスし、ブラウザで証明書Issuerが FTD の Internal CA に置き換わっていることを確認。

---

### 例10: ASA & FTD: Policy-Based Routing (PBR) と SLA Track / Route Map 構成

#### 問題
FTD1 の `inside` ポートに入力される特定サーバ (`10.1.1.50`) からのパケットのみ、デフォルトルートを無視して次世代プロキシ (`10.1.1.250`) へ PBR で転送せよ。プロキシ障害時（ICMP 応答断）は通常の L3 ルーティングへフォールバックさせよ。

#### 要件
* PBR / SLA Track オブジェクトを利用すること。

#### 設定例

```text
1. FMC: Objects > Extended Access List > Add:
   - ACL_PBR_PROXY:
     - Rule 1: Deny ip host 10.1.1.50 10.0.0.0 255.0.0.0 (内部宛通信は除外)
     - Rule 2: Permit ip host 10.1.1.50 any
2. Devices > Device Management > FTD1 > Routing > Policy Based Routing > Add:
   - Ingress Interface: inside
   - Match ACL: ACL_PBR_PROXY
   - Send To (Next Hop): 10.1.1.250
3. Save & Deploy.
```

#### 検証方法
* FTD CLI にて `show running-config route-map` および `show route-map` を実行し、PBR マッチカウンタの増加を確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: FTD CLI にて `packet-tracer` コマンドを実行した際、以下のフェーズでパケットがドロップされた。原因として正しい説明はどれか。

```text
Phase: 4
Type: ACCESS-LIST
Subtype: log
Result: DROP
Config:
access-list CSM_FW_ACL_ line 12 advanced deny ip any any rule-id 268434000
```

* A) LINA の L3/L4 接続テーブルでドロップされた。
* B) Prefilter Policy の Fastpath ルールにマッチしてドロップされた。
* C) Access Control Policy (ACP) の評価ルール（または Default Action）によって拒否された。
* D) Security Intelligence の IP Blacklist によりドロップされた。

**正解**: **C**
**解説**: FMC からデプロイされた Access Control Policy (ACP) のルールは、LINA 側では `CSM_FW_ACL_` という名前の高度な ACL として展開されます。Phase 4 で `CSM_FW_ACL_` にマッチして DROP しているため、ACP ルールによる遮断です。

---

### 2. トラブルシュート問題

**問題**: FTD1 において、内部端末から外部プロキシ経由の Web 閲覧通信に対し PBR (Policy-Based Routing) を設定したところ、内部 LAN（`10.2.2.0/24`）に存在するローカルファイルサーバーへのアクセスまで外部プロキシ宛へ送信されて通信不能になった。原因と適切な修正方法はどれか。

* A) PBR ルートマップのメトリック値が高すぎるため。Match ACL の先頭に `deny ip any 10.0.0.0 255.0.0.0` を追加する。
* B) PBR が Prefilter より前に評価されているため。Prefilter で Fastpath ルールを作成する。
* C) PBR の Match ACL で `permit ip any any` が指定されており、ローカル宛通信までマッチしているため。Match ACL の先頭にローカル宛通信を拒否（`deny ip`）するエントリを追加する。
* D) FTD が Transparent Mode で動作しているため。Routed Mode に変更する。

**正解**: **C**
**解説**: PBR は Match ACL で `permit` されたすべてのトラフィックをネクストホップへ強制転送します。内部宛通信の PBR 適用を防ぐには、Match ACL の先頭で内部宛通信を `deny` して PBR 評価から除外する必要があります。

---

### 3. Design 問題

**問題**: FTD において、大規模データセンターの 10 Gbps データベースバックアップ通信を通過させたい。NGIPS（Snort 3）によるディープパケットインスペクションの CPU 負荷を最小限に抑えつつ、最速でトラフィックを転送するための構成として最も適切なものはどれか。

* A) Access Control Policy で該当通信を `Allow` アクションにし、IPS Policy を `None` に設定する。
* B) Access Control Policy で該当通信を `Trust` アクションにする。
* C) Prefilter Policy で該当通信に対して `Fastpath` ルールを作成する。
* D) SSL Decryption Policy で `Do Not Decrypt` アクションを割り当てる。

**正解**: **C**
**解説**: ACP の `Trust` アクションであっても Snort エンジンへのパケット引き渡し（DAQ 処理）が発生します。Prefilter Policy で `Fastpath` を指定することで、LINA データプレーンのみで即座に処理され、Snort プロセスを完全にバイパスするため CPU 負荷と遅延を最小化できます。

---

### 4. 実装問題

**問題**: ASA1 において、内部サブネット `10.1.1.0/24` から外部への通信を Dynamic PAT 変換（`outside` IP）しつつ、特定 Web サーバー (`10.1.1.50`) 宛の HTTP ポート (80) への外部からのアクセスのみを Static PAT (ポート 8080 → 80) で公開したい。ASA の NAT 設定に関する正解はどれか。

* A) Section 2 (Auto NAT) のみに両方のルールを設定する。
* B) Section 1 (Manual Twice NAT) に Static PAT ルールを設定し、Section 2 (Auto NAT) に Dynamic PAT ルールを設定する。
* C) Section 3 (Manual NAT After Auto) に Static PAT を設定する。
* D) Dynamic PAT ルールを Section 1 の先頭に配置する。

**正解**: **B**
**解説**: 送信元と宛先の両方のポート/IP 変換を伴う Port Forwarding (Static PAT) は、Section 1 (Manual Twice NAT) で定義するのが一般的かつ安全です。通常の送信元 Dynamic PAT は Section 2 (Auto NAT) で定義することで、NAT オーダリングが正しく処理されます。

---

### 5. コンフィグ読解問題

**問題**: FTD FMC の SSL Policy において、以下のルール構成が定義されている。クライアントが未分類の未知の HTTPS サイトにアクセスした際の動作として正しい説明はどれか。

* Rule 1: Action: `Do Not Decrypt`, Category: `Financial Services`
* Rule 2: Action: `Decrypt - Resign`, Category: `Any`
* Default Action: `Do Not Decrypt`

* A) Rule 1 にマッチしないため、Default Action が適用され復号されない。
* B) Rule 2 の Category: `Any` にマッチし、通信は復号スキャンされる。
* C) SSL Policy の評価が拒否され、即座に TCP Reset が送信される。
* D) Rule 2 で復号された後、Prefilter Policy でドロップされる。

**正解**: **B**
**解説**: SSL Policy も Top-Down 順で最初にマッチしたルールが適用されます。Rule 1（Finance）にマッチしない場合、Rule 2 の `Category: Any` にマッチするため、`Decrypt - Resign`（復号）が適用されます。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Threat Defense Configuration Guide (FMC) - Access Control Policies](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/access_control_policies.html)
* [Cisco ASA Series Firewall Configuration Guide - Modular Policy Framework (MPF)](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/firewall/asa-94-firewall-config/policy-mpf.html)
* [Cisco ASA Series Firewall Configuration Guide - Network Address Translation (NAT)](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/firewall/asa-94-firewall-config/nat-overview.html)
* [Cisco Live: Deep Dive into Firepower Threat Defense Packet Flow (BRKSEC-3020)](https://www.ciscolive.com/)
* [Cisco Technical Notes: Troubleshooting FTD Access Control Policy and Engine Debug](https://www.cisco.com/c/en/us/support/docs/security/firepower-ngfw/214580-troubleshoot-ftd-access-control-policy.html)

---

## 📝 補足（Notes）

* **学習メモ**:
  * Blueprint 1.2 は CCIE Security ラボ試験の中で最も出題比重の高い領域の一つです。
  * 「LINA と Snort 3 のどっちのエンジンで評価されているか？」を常に意識して設定・トラブルシューティング（`packet-tracer` や `system support firewall-engine-debug`）を行えるように訓練してください。
* **注意点**:
  * FTD で `Trust` と `Fastpath` の違い（SI や Auth が適用されるか否か）を問う引っかけ問題に注意してください。
