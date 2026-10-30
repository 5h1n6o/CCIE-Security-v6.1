---
layout: default
title: 1.2.e-Traffic-redirection
nav_order: 5
parent: 1.2-Firewall-features-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.2.e Traffic redirection to service modules

Cisco ASA (Adaptive Security Appliance) および Cisco FTD (Firepower Threat Defense) における **Traffic Redirection to Service Modules（サービスモジュールへのトラフィックリダイレクト）** は、ファイアウォールを通過またはファイアウォールに着信したトラフィックを、本体内部のハードウェア/ソフトウェアサービスモジュール（ASA FirePOWER / SFR、CX、IPS モジュールなど）や、ネットワーク上の外部セキュリティアプライアンス（Cisco WSA: Web Security Appliance など）へ透過的に転送・バイパスさせて高度なレイヤ 7 検査を実施する技術です。

リダイレクトの手法としては、主に **ASA MPF (Modular Policy Framework)** を用いた内部モジュール転送と、**WCCPv2 (Web Cache Communication Protocol Version 2)** を用いた外部アプライアンス転送（GRE / L2）が存在します。また、FTD の内部構造においては LINA データプレーンから Snort 3 / Snort 2 インスペクションエンジンへの DAQ (Data Acquisition) 経由のリダイレクト処理として実装されています。

---

## 📘 概要

* **機能概要**:
  * **MPF による内部モジュールリダイレクト**: ASA の Modular Policy Framework (`class-map`, `policy-map`, `service-policy`) を使用して、指定したトラフィックを内部の ASA FirePOWER (SFR) サービスモジュールへインライン (`sfr`) またはパッシブ (`sfr monitor-only` / `traffic-forward sfr monitor-only`) に転送します。
  * **WCCPv2 による外部アプライアンスリダイレクト**: WCCPv2 プロトコルを用いて、Web トラフィック（HTTP/HTTPS/FTP 等）をネットワーク上の外部 Cisco WSA (Web Security Appliance) へ透過的に迂回（Redirect）させ、URL フィルタリング、アンチマルウェアスキャン、プロキシ検査を実施させます。
  * **FTD インスペクションリダイレクト**: FTD 内で LINA コアエンジンが L2-L4 処理および NAT を完了した後、DAQ インターフェースを通じて Snort 脅威対策エンジンへトラフィックを引き渡します。
* **利用目的**:
  * 既存の ASA ファイアウォール構成を変更することなく、NGIPS、AVC (Application Visibility and Control)、AMP (Advanced Malware Protection)、URL Filtering などの次世代セキュリティ機能を追加する。
  * クライアント端末側のプロキシ設定（PAC ファイルやブラウザ設定）を変更せずに、透過的（Transparent）に Web トラフィックをプロキシ / WSA へ導引し、セキュリティポリシーを強制する。
  * 障害発生時（モジュールダウン時）に、トラフィックをドロップさせるか（`fail-close`）、検査をバイパスして通信を維持させるか（`fail-open`）を柔軟に制御する。
* **どのような場面で利用するか**:
  * ASA 5500-X シリーズにおいて ASA OS を維持したまま FirePOWER Services モジュールを追加運用する境界環境。
  * 企業ネットワークにおいて、Web 閲覧通信を一括して WSA へ転送し、脅威対策および帯域制御を行うインターネットエッジ。
  * 新規 IPS シグネチャのテスト導入時に、本番トラフィックに影響を与えないモニタ専用（Tap モード / Monitor-only）でのパッシブ評価環境。

---

## 🔑 要点

| 項目 | ASA MPF Redirection (SFR / FirePOWER) | WCCPv2 Redirection (WSA 連携) | FTD Internal DAQ Redirection |
| :--- | :--- | :--- | :--- |
| **リダイレクト対象** | ASA 内部 SFR (FirePOWER) モジュール | 外部 Cisco WSA (Web Security Appliance) | 内部 Snort Engine (Snort 2 / Snort 3) |
| **制御メカニズム** | MPF (`policy-map` 配下の `sfr` コマンド) | WCCPv2 (UDP 2048 のキープアライブ＆アナウンス) | DAQ (Data Acquisition Library) 内部バス |
| **転送方式** | 内部バックプレーン / メモリバス転送 | **GRE トラフィックカプセル化** (ASA は GRE のみ) | 内部プロセス間共有メモリ / DAQ インターフェース |
| **トラフィック方向** | インバウンド / アウトバウンド双方向可能 | **インバウンドのみサポート (ASA 制限)** | LINA から Snort へのインライン / タップ転送 |
| **障害時動作** | `fail-open` (通過) または `fail-close` (遮断) | `Open Mode` (通過) または `Closed Mode` (遮断) | Access Control Policy の Fail-Open 設定 |
| **バイパス指定** | MPF の `class-map` (ACL) で除外定義 | WCCP Redirect-List (ACL) で `deny` 定義 | ACP の `Trust` ルールまたは Prefilter Fastpath |
| **対応機種** | Cisco ASA 5500-X シリーズ | Cisco ASA, Cisco IOS/IOS-XE ルータ, FTD (FlexConfig) | Cisco FTD 全モデル (ASA 5500-X, FPR1000/2100/4100/9300) |
| **設計上の注意点** | レガシー CX / IPS モジュール設定との競合禁止 | WSA への GRE 送受信用 ACL 許可が必要 | Inline Set の Tap モード使用時はドロップ不可 |

---

## 🏗 動作原理

トラフィックリダイレクトは、ファイアウォール内部の処理エンジンまたはネットワーク接続されたアプライアンスと連携して動作します。

### 1. ASA MPF による FirePOWER (SFR) モジュールへのリダイレクト

```text
[ Client ] ───► 1. Ingress Packet (e.g., HTTP Request)
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Cisco ASA (Adaptive Security Appliance) LINA Engine         │
│                                                             │
│  2. L2-L4 Processing (ACL, Conn Table Lookup, NAT)          │
│                                                             │
│  3. MPF (Modular Policy Framework) Evaluation               │
│     └── Class-Map Match? ──► Action: 'sfr' / 'sfr fail-open'│
│                                                             │
│  4. Internal Backplane Redirection to SFR Module            │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ ASA FirePOWER (SFR) Service Module                          │
│                                                             │
│  5. L7 Deep Packet Inspection (Snort IPS, AVC, AMP, URL)    │
│  6. Inspection Decision:                                    │
│     ├── PERMIT ──► Return Packet to ASA LINA Engine         │
│     └── DROP   ──► Drop Packet & Send Alert to FMC          │
└──────────────────────────┬──────────────────────────────────┘
                           │ (If Permitted)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ Cisco ASA LINA Engine                                       │
│                                                             │
│  7. Egress Interface Processing & Forward Packet to Target │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
                      [ Server ]
```

### 2. WCCPv2 による Cisco WSA への透過的リダイレクト

```text
[ Client ] ───► 1. HTTP Request (Dst: Web Server 203.0.113.100:80)
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Cisco ASA / Router (WCCPv2 Service Router)                  │
│                                                             │
│  2. Ingress Interface WCCP Intercept (`wccp 90 redirect in`)│
│  3. WCCP Redirect ACL Match?                                │
│     ├── MATCH (Permit) ──► Encapsulate in GRE Tunnel        │
│     └── DENY (Bypass)  ──► Standard Routing to Internet     │
└──────────────────────────┬──────────────────────────────────┘
                           │ (GRE Encapsulated Packet)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ Cisco WSA (Web Security Appliance - WCCP Client)            │
│                                                             │
│  4. Decapsulate GRE & Perform Web Security Inspection       │
│  5. Initiate New Connection to Real Web Server (Proxy)      │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
                   [ Web Server ]
```

---

## ⚙ 動作シーケンス

### ASA MPF ➔ SFR モジュールリダイレクトシーケンス

1. **Ingress パケットの受信と L2-L4 処理**:
   * ASA は Ingress インターフェイスでパケットを受信し、インバウンド ACL、既存コネクションテーブル（xlate/conn）、および NAT ルールを評価します。
2. **MPF サービスポリシーの検索**:
   * `service-policy` が評価され、`class-map` に定義された条件（例: `match access-list`）にトラフィックがマッチするか判定します。
3. **SFR モジュールへのパケット引き渡し**:
   * マッチしたパケットは内部バックプレーン経由で SFR モジュールへ転送されます。
   * **Inline Mode (`sfr`)**: パケットのオリジナルは SFR モジュールに送られ、SFR の検査結果を待ちます。
   * **Monitor-only Mode (`sfr monitor-only`)**: パケットのコピーが SFR モジュールに送られ、オリジナルパケットは待機することなくそのまま Egress インターフェイスから送出されます。
4. **SFR モジュール内の検査と判定**:
   * SFR モジュール内の Snort エンジンが AVC、IPS シグネチャ、URL カテゴリ、AMP ファイル解析を実行します。
5. **後処理と送出**:
   * **Permit**: パケットは ASA LINA エンジンに戻され、Egress ルーティングに従って目的の宛先へ送信されます。
   * **Drop**: SFR モジュールがパケットを破棄し、FMC (Firepower Management Center) へイベントログを送信します。
   * **SFR モジュール障害時**:
     * `sfr fail-open`: 検査を行わずに ASA がパケットを透過させます。
     * `sfr fail-close`: ASA はパケットをドロップし通信を遮断します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **`sfr` コマンドのキーワード選択（Fail-Open vs Fail-Close）**:
   * 問題文の指示を厳密に読み分ける必要があります。
   * 「FirePOWER モジュールがダウンまたは再起動した場合でも、業務通信を停止させないこと」 ➔ **`sfr fail-open`**
   * 「セキュリティを最優先し、モジュール非稼働時は一切の通信を許可しないこと」 ➔ **`sfr fail-close`**
   * ※ 単に `sfr` とのみ指定した場合は、デフォルトで `sfr fail-open` として動作します。
2. **ASA におけるレガシーモジュールポリシーとの競合**:
   * 既に `inspect ips`（レガシー AIP-SSM モジュール用）や `cxsc`（ASA CX モジュール用）の設定が存在する場合、`sfr` コマンドを適用しようとすると設定エラーが発生します。
   * 事前に既存の `policy-map` からレガシーリダイレクトコマンドを削除する必要があります。
3. **WCCPv2 の ASA 固有制限（重要）**:
   * **GRE 方式のみサポート**: ASA は L2 リダイレクト（Layer 2 Redirect）をサポートしません。必ず **GRE 転送 (Forwarding Method: GRE)** を使用します。
   * **インバウンドのみ**: ASA では `wccp interface <if_name> <service-id> redirect in` のみサポートされ、`redirect out` は設定できません。
   * **グループリスト (Group-List) ACL は Extended のみ**: ASA で WCCP クライアント（WSA）を制限する `group-list` ACL は、**Extended ACL で Source に WSA の IP アドレスを指定** しなければなりません（Cisco IOS の Standard ACL と異なる点に注意）。
   * **GRE 通信の ACL 許可**: ASA 自身の管理 IP やデータ IP 上で GRE パケットを受信できるよう、インバウンド ACL や Security Group 設定で **`permit gre host <WSA_IP> host <ASA_IP>`** が許可されているか確認が必要です。
4. **Service ID の番号分類**:
   * `web-cache` (Service ID 0): HTTP (TCP 80) 専用。
   * Dynamic Service ID (`90` 〜 `97` または `1` 〜 `255`): HTTP、HTTPS (443)、FTP (21) など任意のポート番号を組み合わせ可能。推奨は `90` 〜 `97`。
5. **FTD における WCCP の構成方法**:
   * FTD の Native GUI (FMC) には WCCP 専用の設定画面が存在しないため、FMC の **FlexConfig Policy** を用いて LINA コマンド (`wccp ...`) をインジェクト展開させます。

---

## 🛠 設定方法

### 1. Cisco ASA: MPF による FirePOWER (SFR) モジュールへのインライン/モニタ専用リダイレクト (CLI)

```bash
! ===== 1. リダイレクト対象トラフィックの分類 (Extended ACL) =====
access-list ACL_SFR_TRAFFIC extended permit ip 10.1.0.0 255.255.0.0 any

! ===== 2. MPF Class-Map の定義 =====
class-map CM_SFR_TRAFFIC
 match access-list ACL_SFR_TRAFFIC

! ===== 3. MPF Policy-Map の定義と SFR アクションの割り当て =====
! パターン A: 通常のインライン検査 (Fail-Open 設定)
policy-map PM_SERVICE_POLICY
 class CM_SFR_TRAFFIC
  sfr fail-open

! パターン B: 障害時遮断の厳格インライン検査 (Fail-Close 設定)
! policy-map PM_SERVICE_POLICY
!  class CM_SFR_TRAFFIC
!   sfr fail-close

! パターン C: パッシブモニタリング (Tap モード / Monitor-only 設定)
! policy-map PM_SERVICE_POLICY
!  class CM_SFR_TRAFFIC
!   sfr fail-open monitor-only

! ===== 4. グローバルまたはインターフェイスへの Service Policy 適用 =====
service-policy PM_SERVICE_POLICY interface inside
```

### 2. Cisco ASA: WCCPv2 による外部 Cisco WSA への HTTP/HTTPS トラフィック透過リダイレクト (CLI)

```bash
! ===== 1. WSA IP アドレスの定義 (Group-List 用 Extended ACL) =====
! ※ ASA では group-list に Extended ACL を使用し、Source に WSA IP を記述する
access-list ACL_WSA_CLIENTS extended permit ip host 172.16.6.100 any

! ===== 2. リダイレクト対象トラフィックの定義 (Redirect-List Extended ACL) =====
! 内部 Web サーバー宛ての通信はバイパス (deny) し、それ以外の Web アクセスをリダイレクト (permit)
access-list ACL_WCCP_REDIRECT_HTTP extended deny tcp any host 10.1.1.100 eq 80
access-list ACL_WCCP_REDIRECT_HTTP extended permit tcp 10.1.0.0 255.255.0.0 any eq 80

access-list ACL_WCCP_REDIRECT_HTTPS extended permit tcp 10.1.0.0 255.255.0.0 any eq 443

! ===== 3. WCCP Service Group の構成 =====
! Service ID 90: HTTP 用 (暗号化パスワード authentication 設定付き)
wccp 90 redirect-list ACL_WCCP_REDIRECT_HTTP group-list ACL_WSA_CLIENTS password wccpsecret

! Service ID 91: HTTPS 用
wccp 91 redirect-list ACL_WCCP_REDIRECT_HTTPS group-list ACL_WSA_CLIENTS password wccpsecret

! ===== 4. Ingress インターフェイスへの WCCP リダイレクト適用 =====
wccp interface inside 90 redirect in
wccp interface inside 91 redirect in
```

### 3. Cisco FTD (FMC FlexConfig): FlexConfig を用いた FTD での WCCPv2 WSA リダイレクト構成

```text
! FMC GUI: Objects > FlexConfig > FlexConfig Objects > Add FlexConfig Object
! Name: FlexConfig_WCCP_WSA
! Deployment: Every Time, Type: Append

access-list ACL_FTD_WSA extended permit ip host 172.16.6.100 any
access-list ACL_FTD_REDIRECT extended permit tcp 10.20.0.0 255.255.0.0 any eq 80

wccp 90 redirect-list ACL_FTD_REDIRECT group-list ACL_FTD_WSA password wccpsecret
wccp interface inside 90 redirect in

! FMC GUI: Devices > FlexConfig > 対象 FTD に FlexConfig_WCCP_WSA を割り当てて Deploy 実行。
```

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **ASA FirePOWER (SFR) モジュールの物理/動作ステータス確認** | ASA CLI | <code>show module sfr</code> / <code>show module sfr details</code> |
| **MPF による SFR リダイレクト統計およびパケットカウント確認** | ASA CLI | <code>show service-policy sfr</code> / <code>show service-policy inspect sfr</code> |
| **WCCPv2 グローバル設定・登録済みクライアント(WSA)一覧表示** | ASA / Router CLI | <code>show wccp</code> / <code>show wccp 90</code> |
| **特定 WCCP Service ID の詳細情報・パケットリダイレクト数表示** | ASA / Router CLI | <code>show wccp 90 detail</code> |
| **WCCP アソシエーション・クライアント接続状態確認** | ASA / Router CLI | <code>show wccp 90 clients</code> |
| **WCCP イベントおよびキープアライブメッセージのデバッグ** | ASA CLI | <code>debug wccp events</code> / <code>debug wccp packets</code> |
| **WSA 側の WCCP 接続状態ログ確認 (WSA CLI)** | WSA CLI | <code>tail track_services</code> / <code>sh wccp</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **`sfr` コマンド投入時に「Service policy error」が発生する** | 既存の `cxsc` または `inspect ips` 設定との競合 | ASA: <code>show running-config policy-map</code> | 競合するレガシーモジュールコマンドを `policy-map` から削除した後に再実行する。 |
| **SFR モジュールダウン時に全通信が停止する** | `sfr fail-close` が設定されている | ASA: <code>show service-policy sfr</code> | 業務要件に応じて `sfr fail-open` キーワードに変更する。 |
| **WSA を追加したのに WCCP ネゴシエーション（Here I am）が成功しない** | パスワード不一致、または GRE パケットが ACL でドロップされている | ASA: <code>show wccp 90 detail</code><br>ASA: <code>show log</code> | ASA/WSA 間で MD5 パスワードを一致させ、`permit gre` をインバウンド ACL に追加する。 |
| **ASA で `wccp 90 group-list` 設定が拒否される** | Standard ACL を指定している (ASA は Extended ACL 必須) | ASA: <code>show access-list</code> | `access-list <NAME> extended permit ip host <WSA_IP> any` 形式で作成し直す。 |
| **内部 Web サーバー宛ての通信まで WSA に曲げられてアクセス不能になる** | WCCP Redirect-List ACL で内部宛てを除外していない | ASA: <code>show access-list</code> | Redirect-List ACL の先頭に `deny tcp any host <内部WebIP> eq 80` を追加する。 |
| **WSA からクライアントへの戻り通信が到達しない** | WSA 側のデフォルトルート / ルーティングテーブルの設定不備 | WSA CLI: <code>netstat -rn</code> | WSA の Management/Data インターフェイスの標準ルーティングテーブルを正しく設定する。 |

---

## ⚠ 制限事項

* **Cisco ASA MPF / SFR モジュールの制限**:
  * ASA 5500-X シリーズのモデル（5506-X 〜 5585-X）でのみサポート。
  * Multiple Context Mode（マルチコンテキストモード）では、Context ごとに Service Policy を構成する必要があります。
* **Cisco ASA WCCPv2 の制限**:
  * **GRE モード専用**: ASA は Layer 2 (L2) リダイレクトをサポートしません。
  * **インバウンド専用**: `redirect in` のみサポート（`redirect out` は不可）。
  * **マルチキャスト非対応**: ASA は WCCP のマルチキャストルータグループ定義に非対応（ユニキャスト 2048 のみ）。
  * **IPv6 WCCP 非サポート**: WCCP による IPv6 トラフィックリダイレクトは非対応。

---

## 🔄 他技術との関連

* **1.2 Firewall features (MPF)**: トラフィックの抽出および SFR モジュールへの転送指示は、MPF (`class-map`, `policy-map`, `service-policy`) のフレームワーク上で動作します。
* **1.2.a NAT**: WCCP リダイレクトは NAT 処理の前にインバウンドでキャプチャされるか、NAT 処理後に透過転送されるため、WSA 側で見えるクライアント IP アドレスの維持（IP Spoofing モード）に影響を与えます。
* **2.0 Intrusion Prevention (IPS / Snort)**: SFR モジュールおよび FTD 内の Snort エンジンがリダイレクトされたパケットに対して実際の IPS シグネチャマッチングを実行します。
* **WSA (Web Security Appliance)**: WCCPv2 の対向クライアントとして動作し、リダイレクトされた HTTP/HTTPS/FTP 通信に対して URL フィルタリングや AVC を実行します。

---

## 🧩 比較表

### 1. MPF Redirection (SFR) vs WCCPv2 Redirection (WSA) vs FTD Native DAQ

| 比較項目 | ASA MPF Redirection (SFR) | WCCPv2 Redirection (WSA) | FTD Native DAQ Redirection |
| :--- | :--- | :--- | :--- |
| **目的** | 内部 FirePOWER モジュールでの NGIPS/AMP 検査 | 外部 WSA での Web 透過プロキシ/URL 検査 | ASA LINA と Snort エンジン間のパケット引渡し |
| **転送先** | ASA 内部 SFR モジュール | ネットワーク上の外部 WSA アプライアンス | FTD 内部 Snort プロセス (Snort 2 / Snort 3) |
| **カプセル化** | 内部 PCI/バス転送 (カプセル化なし) | **GRE トンネルカプセル化** | 内部共有メモリ / DAQ インターフェース |
| **障害切替** | `sfr fail-open` / `sfr fail-close` | `Open Mode` / `Closed Mode` | ACP Fail-Open 設定 / Snort Safe Mode |
| **透過性** | 完全透過 | 透過（クライアントからプロキシは不可視） | 完全透過 |

### 2. ASA SFR モジュール動作モード比較 (Inline vs Fail-Close vs Monitor-Only)

| 動作モード | コマンド | パケットの影響 | モジュールダウン時の動作 | 主な用途 |
| :--- | :--- | :--- | :--- | :--- |
| **Inline (Fail-Open)** | `sfr` / `sfr fail-open` | インラインブロック・ドロップ可能 | **通信をバイパス許可 (通過)** | 標準的な本番インライン運用 |
| **Inline (Fail-Close)** | `sfr fail-close` | インラインブロック・ドロップ可能 | **通信をドロップ (全遮断)** | 高セキュリティ要求環境 |
| **Monitor-Only (Tap)** | `sfr monitor-only` | パケットコピーのみ検査 (ドロップ不可) | 通信に影響なし | シグネチャ評価・事前検証環境 |

---

## 💡 ベストプラクティス

1. **インライン導入前の `sfr monitor-only` による事前検証**:
   * ASA FirePOWER を本番環境へ導入する際は、いきなり `sfr fail-close` でインライン化せず、まず `sfr monitor-only`（Tap モード）で運用して意図しないドロップや誤検知（False Positive）が発生しないか確認します。
2. **WCCP Redirect-List における Bypass ルールの明記**:
   * 社内 LAN や DMZ 上の Web サーバー宛て通信が誤って WSA へリダイレクトされないよう、Redirect-List ACL の先頭に **`deny tcp any <内部サブネット> eq 80`** を必ず配置します。
3. **ASA WCCP 用 GRE 通信の明示的許可**:
   * インバウンド ACL を運用している ASA インターフェイス上で WCCP を有効化する場合、WSA からの GRE レスポンスを受け取るため **`permit gre host <WSA_IP> host <ASA_IP>`** を追加しておきます。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA MPF: ASA FirePOWER (SFR) モジュールへの HTTP トラフィックのインラインリダイレクト (`fail-open`)

#### 問題
ASA1 において、`inside` インターフェイスから発信される外部 Web アクセストラフィック（TCP ポート 80 および 443）を ASA FirePOWER (SFR) モジュールへインライン転送せよ。万が一 SFR モジュールが障害またはメンテナンス等で停止した場合でも、Web アクセス通信を遮断せずに通過（`fail-open`）させよ。

#### 要件
* Extended ACL と MPF を使用すること。
* モジュール障害時は通信を維持すること。

#### 設定例

```bash
! 1. リダイレクト対象の HTTP/HTTPS トラフィックを定義
access-list ACL_WEB_TRAFFIC extended permit tcp 10.1.0.0 255.255.0.0 any eq 80
access-list ACL_WEB_TRAFFIC extended permit tcp 10.1.0.0 255.255.0.0 any eq 443

! 2. MPF Class-Map の作成
class-map CM_WEB_TRAFFIC
 match access-list ACL_WEB_TRAFFIC

! 3. MPF Policy-Map の定義と sfr fail-open の指定
policy-map PM_INSIDE_POLICY
 class CM_WEB_TRAFFIC
  sfr fail-open

! 4. inside インターフェイスへ適用
service-policy PM_INSIDE_POLICY interface inside
```

#### 検証方法
* `show service-policy sfr` を実行し、`Class-map: CM_WEB_TRAFFIC` 配下で `sfr fail-open` がアクティブであり、パケット数がカウントされていることを確認。
* モジュールをシャットダウン (`sw-module module sfr shutdown`) しても、内部クライアントからの Web アクセスが切断されないことを検証。

---

### 例2: ASA MPF: ASA FirePOWER (SFR) モジュールへの厳格なインラインリダイレクト (`fail-close`)

#### 問題
高セキュリティセグメント（`dmz`: `192.168.10.0/24`）からの全 IP トラフィックを ASA FirePOWER モジュールへ検査リダイレクトせよ。SFR モジュールが利用不能状態になった場合は、未検査のパケットを一切透過させず即座にドロップ（`fail-close`）させよ。

#### 要件
* `sfr fail-close` キーワードを使用すること。

#### 設定例

```bash
access-list ACL_DMZ_ALL extended permit ip 192.168.10.0 255.255.255.0 any

class-map CM_DMZ_ALL
 match access-list ACL_DMZ_ALL

policy-map PM_DMZ_POLICY
 class CM_DMZ_ALL
  sfr fail-close

service-policy PM_DMZ_POLICY interface dmz
```

#### 検証方法
* `show service-policy sfr` を実行し、`sfr fail-close` が反映されていることを確認。
* モジュール停止時に `192.168.10.0/24` からの通信が完全にブロックされることを検証。

---

### 例3: ASA CLI: `traffic-forward sfr monitor-only` によるパッシブモニタリング (Tap モード) リダイレクト

#### 問題
ASA1 を通過するインターネット宛ての全トラフィックに対し、本番通信に影響を与えないようパケットのコピーのみを SFR モジュールへ転送（Monitor-only / Tap モード）して評価せよ。

#### 要件
* `sfr monitor-only` または CLI 専用コマンドを使用すること。

#### 設定例

```bash
access-list ACL_MONITOR_ALL extended permit ip any any

class-map CM_MONITOR_ALL
 match access-list ACL_MONITOR_ALL

policy-map PM_GLOBAL_MONITOR
 class CM_MONITOR_ALL
  sfr fail-open monitor-only

service-policy PM_GLOBAL_MONITOR global
```

#### 検証方法
* `show service-policy sfr` を実行し、`Monitor-only: Enabled` と表示され、SFR 上でイベントが検知されても通信がドロップされないことを確認。

---

### 例4: ASA CLI: WCCPv2 標準 Dynamic Service Group (90) を用いた Web トラフィック (TCP 80) の WSA リダイレクト

#### 問題
ASA1 の `inside` インターフェイスに接続されたクライアント (`10.1.0.0/16`) からの HTTP (TCP 80) トラフィックを、WCCPv2 Dynamic Service Group `90` を使用して外部 Cisco WSA (`172.16.6.100`) へ透過リダイレクトせよ。

#### 要件
* Dynamic Service ID `90` を使用すること。

#### 設定例

```bash
! リダイレクト対象 ACL
access-list ACL_WCCP_HTTP extended permit tcp 10.1.0.0 255.255.0.0 any eq 80

! 許可 WSA クライアント ACL (ASA は Extended 必須)
access-list ACL_WSA_IP extended permit ip host 172.16.6.100 any

! WCCP 定義
wccp 90 redirect-list ACL_WCCP_HTTP group-list ACL_WSA_IP

! インターフェイス適用
wccp interface inside 90 redirect in
```

#### 検証方法
* `show wccp 90 detail` を実行し、`Service Identifier: 90` の `Number of Cache Engines: 1` となり、WSA IP が登録されていることを確認。

---

### 例5: ASA CLI: WCCPv2 複数 Service Group (94: HTTP / 95: HTTPS) と MD5 パスワード認証構成

#### 問題
ASA1 において、HTTP (TCP 80) 用に Service Group `94`、HTTPS (TCP 443) 用に Service Group `95` を作成し、WSA (`172.16.6.100`) へリダイレクトせよ。WCCP セッションの保護のため MD5 パスワード `wccpasa` を設定せよ。

#### 要件
* Service ID `94` および `95` を使用し、パスワード認証を有効化すること。

#### 設定例

```bash
access-list ACL_HTTP extended permit tcp any any eq 80
access-list ACL_HTTPS extended permit tcp any any eq 443
access-list ACL_WSA_HOST extended permit ip host 172.16.6.100 any

wccp 94 redirect-list ACL_HTTP group-list ACL_WSA_HOST password wccpasa
wccp 95 redirect-list ACL_HTTPS group-list ACL_WSA_HOST password wccpasa

wccp interface inside 94 redirect in
wccp interface inside 95 redirect in
```

#### 検証方法
* `show wccp 94 detail` および `show wccp 95 detail` で `Total Authentication failures: 0` であることを確認。

---

### 例6: ASA CLI: WCCPv2 Group-List (Extended ACL) を使用した許可済み WSA IP の制限

#### 問題
ASA1 の WCCPv2 設定において、悪意ある未認証プロキシが WCCP クライアントとして登録されるのを防ぐため、Group-List ACL を使用して指定 IP (`172.16.6.100` および `172.16.6.101`) のみの登録を許可せよ。

#### 要件
* Extended ACL を使用すること。

#### 設定例

```bash
access-list ACL_APPROVED_WSAS extended permit ip host 172.16.6.100 any
access-list ACL_APPROVED_WSAS extended permit ip host 172.16.6.101 any
access-list ACL_APPROVED_WSAS extended deny ip any any

access-list ACL_REDIRECT_ALL_WEB extended permit tcp any any eq 80

wccp 90 redirect-list ACL_REDIRECT_ALL_WEB group-list ACL_APPROVED_WSAS
wccp interface inside 90 redirect in
```

#### 検証方法
* `show wccp 90 clients` を実行し、許可された 2 台の WSA のみがクライアント一覧に表示されることを確認。

---

### 例7: Router/IOS: WCCPv2 Closed Mode (`mode closed`) による WSA 非稼働時のトラフィック遮断

#### 問題
ルータ R4 において、WSA (`172.16.6.100`) 宛てに HTTP トラフィックを WCCP リダイレクトせよ。WSA がダウンして利用可能な WCCP クライアントが存在しなくなった場合は、リダイレクト対象トラフィックを外部へ直接バイパスさせず即座にドロップする **Closed Mode (`mode closed`)** を設定せよ。

#### 要件
* Cisco IOS CLI で `mode closed` を構成すること。

#### 設定例

```bash
ip access-list extended REDIRECTED_TRAFFIC
 permit tcp any any eq 80

ip access-list standard WSA_GROUP
 permit host 172.16.6.100

ip access-list extended WCCP_CLOSED_ACL
 permit tcp any any eq 80

ip wccp version 2
ip wccp 93 redirect-list REDIRECTED_TRAFFIC group-list WSA_GROUP mode closed service-list WCCP_CLOSED_ACL

interface FastEthernet0/1.32
 ip wccp 93 redirect out
```

#### 検証方法
* `show ip wccp 93` を実行し、`Service mode: Closed` であることを確認。WSA 停止時に `Total Packets Dropped Closed` のカウントが増加することを検証。

---

### 例8: FTD (FMC FlexConfig): FlexConfig を使用した FTD 上での WCCPv2 WSA トラフィックリダイレクト

#### 問題
FMC で管理される FTD1 において、`inside` インターフェイスからの HTTP 通信 (TCP 80) を WSA (`10.28.20.130`) へ WCCPv2 リダイレクトする設定を FlexConfig で展開せよ。

#### 要件
* Service ID `60` を使用すること。

#### 設定例

```text
1. FMC: Objects > FlexConfig > FlexConfig Objects > Add FlexConfig Object:
   - Name: FTD_WCCP_Service_60
   - Deployment: Every Time, Type: Append
   - Script:
     access-list ACL_WSA_HOST extended permit ip host 10.28.20.130 any
     access-list ACL_WCCP_HTTP extended permit tcp 10.28.0.0 255.255.0.0 any eq 80
     wccp 60 redirect-list ACL_WCCP_HTTP group-list ACL_WSA_HOST
     wccp interface inside 60 redirect in
2. Devices > FlexConfig > FTD1 にポリシーをバインドして Deploy。
```

#### 検証方法
* FTD CLI (Diagnostic CLI) にて `show wccp 60` を実行し、WCCP リダイレクトが動作していることを確認。

---

### 例9: ASA MPF: 特定ソース IP サブネット（VLAN 10）のみを SFR モジュールへ送る選択的 MPF リダイレクト

#### 問題
ASA1 において、開発部門サブネット `10.10.0.0/16` からのトラフィックのみを選択的に SFR モジュールへインライン転送し、他のセグメントからの通信は SFR 検査をバイパスさせよ。

#### 要件
* MPF の Class-Map 条件を限定すること。

#### 設定例

```bash
access-list ACL_DEV_ONLY extended permit ip 10.10.0.0 255.255.0.0 any

class-map CM_DEV_ONLY
 match access-list ACL_DEV_ONLY

policy-map PM_SELECTIVE_POLICY
 class CM_DEV_ONLY
  sfr fail-open

service-policy PM_SELECTIVE_POLICY interface inside
```

#### 検証方法
* `10.10.0.10` からの通信発生時にのみ `show service-policy sfr` のパケット数がカウントされ、他サブネットの通信ではカウントが増加しないことを確認。

---

### 例10: ASA CLI: WCCPv2 Redirect-List における内部 Web サーバー宛通信のバイパス (Bypass ACL) 設定

#### 問題
内部クライアント `10.1.0.0/16` からの Web 通信を WSA へ WCCP リダイレクトせよ。ただし、社内イントラネットサーバー `10.1.200.50` および `10.1.200.51` 宛ての HTTP アクセスは WSA へ送らずにバイパス（直接アクセス）させよ。

#### 要件
* Redirect-List ACL 内で `deny` エントリを先行定義すること。

#### 設定例

```bash
! 内部 Web サーバー宛ては deny (バイパス)
access-list ACL_WCCP_BYPASS extended deny tcp 10.1.0.0 255.255.0.0 host 10.1.200.50 eq 80
access-list ACL_WCCP_BYPASS extended deny tcp 10.1.0.0 255.255.0.0 host 10.1.200.51 eq 80
! その他の外部 Web アクセスは permit (リダイレクト)
access-list ACL_WCCP_BYPASS extended permit tcp 10.1.0.0 255.255.0.0 any eq 80

access-list ACL_WSA_SERVER extended permit ip host 172.16.6.100 any

wccp 90 redirect-list ACL_WCCP_BYPASS group-list ACL_WSA_SERVER
wccp interface inside 90 redirect in
```

#### 検証方法
* クライアントから `[suspicious link removed]` へアクセスし、`show wccp 90 detail` の `Total Packets Denied Redirect` カウントが増加することを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ASA1 上で以下の MPF コンフィグが適用されている。ASA FirePOWER (SFR) サービスモジュールのソフトウェア障害が発生してモジュールがダウンした場合、`10.1.1.50` から外部 Web サーバーへの通信はどのようになるか。

```bash
access-list ACL_HTTP extended permit tcp 10.1.1.0 255.255.255.0 any eq 80

class-map CM_HTTP
 match access-list ACL_HTTP

policy-map PM_POLICY
 class CM_HTTP
  sfr fail-close

service-policy PM_POLICY interface inside
```

* A) 検査なしで透過して外部へ正常に到達する。
* B) ASA によってパケットが即座にドロップされ、通信は遮断される。
* C) 自動的に monitor-only モードへ切り替わる。
* D) パケットは WSA へリダイレクトされる。

**正解**: **B**
**解説**: `sfr fail-close` が指定されているため、SFR モジュールが停止すると、ASA LINA エンジンは該当トラフィックを検査不能として即座にドロップします。

---

### 2. トラブルシュート問題

**問題**: 管理者が ASA1 上で外部 WSA (`172.16.6.100`) へ Web トラフィックを迂回させるために以下の WCCP 設定を行ったが、`show wccp 90` を実行しても `Number of Cache Engines: 0` のままでアソシエーションが確立しない。原因として最も適切なものはどれか。

```bash
access-list ACL_WSA_GROUP standard permit host 172.16.6.100
access-list ACL_REDIRECT extended permit tcp any any eq 80

wccp 90 redirect-list ACL_REDIRECT group-list ACL_WSA_GROUP
wccp interface inside 90 redirect in
```

* A) `redirect out` を使用していないため。
* B) ASA では `group-list` に Standard ACL は使用できず、Source に WSA IP を指定した **Extended ACL** を指定しなければならないため。
* C) WCCPv1 モードが有効になっていないため。
* D) `wccp 90` は予約された Service ID であり使用できないため。

**正解**: **B**
**解説**: Cisco ASA の WCCP 仕様では、`group-list` に指定する ACL は Cisco IOS のような Standard ACL ではなく、**Extended ACL**（例: `access-list ACL_WSA_GROUP extended permit ip host 172.16.6.100 any`）で指定する必要があります。

---

### 3. Design 問題

**問題**: 既存の ASA 5516-X ファイアウォール構成を変更することなく、高度な Web 脅威対策（マルウェアスキャン、URL カテゴリ制御）を透過的に導入したい。WSA アプライアンスと ASA の連動設計として正しいアプローチはどれか。

* A) WSA を Routed モードで ASA とインラインに直列挿入する。
* B) ASA 上で WCCPv2 を有効化し、GRE トンネルを介して HTTP/HTTPS トラフィックを WSA へインバウンドリダイレクトする。
* C) FTD にアップグレードし、PBR で WSA に転送する。
* D) ASA 上で L2 転送（Layer 2 Redirection）を設定する。

**正解**: **B**
**解説**: 既存の ASA 構成や IP スキームに一切影響を与えずに透過型 Web セキュリティを導入するには、ASA 上で **WCCPv2 (GRE トンネル転送)** を構成して Web トラフィックのみを外部 WSA へダイナミックリダイレクトさせる構成が最も標準的です（ASA は L2 リダイレクト非対応）。

---

### 4. 実装問題

**問題**: ASA1 の `inside` インターフェイスに適用する WCCPv2 設定において、WSA (`172.16.6.100`) との間で平文のメッセージ改ざんを防ぐための MD5 認証キー `cisco123` を適用する正しいコマンド記述はどれか。

* A) `wccp 90 authentication md5 cisco123`
* B) `wccp 90 redirect-list ACL_REDIRECT group-list ACL_WSA password cisco123`
* C) `ip wccp password cisco123`
* D) `service-policy` 配下で `wccp password cisco123` を指定する

**正解**: **B**
**解説**: ASA における WCCP 認証パスワードは、`wccp <service-id> ... password <secret>` コマンドのオプションキーワードとして直接指定します。

---

### 5. コンフィグ読解問題

**問題**: 以下の WCCP Redirect-List ACL 設定が ASA1 に適用されている。内部ホスト `10.1.1.50` が `[suspicious link removed]` (社内サーバー) および `[suspicious link removed]` (外部サイト) へアクセスした際のリダイレクト動作の記述として正しいものはどれか。

```bash
access-list ACL_REDIRECT extended deny tcp any 10.1.100.0 255.255.255.0 eq 80
access-list ACL_REDIRECT extended permit tcp 10.1.0.0 255.255.0.0 any eq 80
```

* A) 両方のアクセスとも WSA へリダイレクトされる。
* B) 両方のアクセスともリダイレクトされず直接通信する。
* C) `10.1.100.1` 宛てのアクセスはリダイレクトがバイパスされ直接通信し、`www.cisco.com` 宛てのみ WSA へリダイレクトされる。
* D) ACL の `deny` 行により `10.1.100.1` 宛ての通信は ASA でドロップされる。

**正解**: **C**
**解説**: WCCP の Redirect-List ACL において、`deny` 行は「パケットをドロップする」意味ではなく、**「WCCP リダイレクトの対象からバイパス（除外）して通常のルーティングで直接通信させる」** ことを意味します。

---

## 🔗 参考リソース

* [Cisco ASA Series Firewall Configuration Guide - ASA FirePOWER Module Integration]([suspicious link removed])
* [Cisco ASA Series General Operations Configuration Guide - WCCP Redirection]([suspicious link removed])
* [Cisco Technical Notes: Configure WCCP on ASA with WSA]([suspicious link removed])
* [Cisco Live: Deep Dive into FirePOWER Module Redirection and WCCP Architectures (BRKSEC-3020)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * ASA のトラフィックリダイレクトは **「内部モジュール (MPF `sfr`)」** と **「外部アプライアンス (WCCPv2)」** の 2 大テーマで出題されます。
  * 特に WCCPv2 において **「ASA は GRE のみ・インバウンドのみ・group-list は Extended ACL のみ」** という 3 大制約はラボ試験の超頻出引っかけポイントです。
* **図解（WCCP Redirect-List の Permit / Deny の意味）**:
  ```text
  [ Incoming HTTP Packet ]
            │
            ▼
  [ WCCP Redirect-List ACL Check ]
            ├── PERMIT ──► Encapsulate in GRE ──► Send to WSA Proxy
            └── DENY   ──► Bypass WCCP        ──► Standard Routing to Real Server
  ```
* **注意点**:
  * ASA FirePOWER (SFR) モジュールを再起動する際は、MPF で `sfr fail-open` が設定されているか確認してください。`sfr fail-close` のままモジュールを再起動すると、起動完了までの約 5〜10 分間、該当トラフィックが完全に全断します。
