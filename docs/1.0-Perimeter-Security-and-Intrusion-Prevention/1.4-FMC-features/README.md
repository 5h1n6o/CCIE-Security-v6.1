---
layout: default
title: 1.4-FMC-features
nav_order: 4
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.4 Cisco FMC features

Cisco Secure Firewall Management Center (FMC: 旧 Firepower Management Center) は、Cisco Secure Firewall (FTD: Firepower Threat Defense) アプライアンス、ASA with FirePOWER Services、および NGIPS モジュールを一元的に集中管理・制御・監視するための高度なセキュリティ管理プラットフォームです。

CCIE Security v6.1 Blueprint 1.4 では、単なる管理 GUI の操作にとどまらず、FMC が提供する多層的ポリシーモデル（Access Control, Prefilter, Intrusion, File/AMP, SSL, Identity, Network Discovery, NAT, FlexConfig）、オブジェクト管理、ネットワークコンテキスト分析（Network Discovery / IOC / Impact Level）、セキュリティインテリジェンス（SI / TID）、FMC High Availability (HA)、およびヘルスモニタリングの実践的設定・運用・トラブルシューティング能力が厳格に評価されます。

---

## 📘 概要

* **機能概要**:
  * **集中ポリシー・オブジェクト管理**: アクセスコントロールポリシー (ACP) を親とし、Prefilter、Intrusion、File/AMP、SSL、Identity、QoS、NAT などのサブポリシーを一元管理・バインド。
  * **コンテキスト認識型セキュリティ分析**: ネットワークトラフィックから自動的にホスト、OS、サービス、アプリケーション、アクティブユーザー（ISE pxGrid / Passive Identity 連携）を検出し、ネットワークマップ（Network Map）を構築。
  * ** threat インテリジェンスと自動リスク評価**: Security Intelligence (SI) フィードおよび Threat Intelligence Director (TID) による悪意ある IP/Domain/URL の即時ブロック、ならびに脆弱性情報と攻撃の整合性を評価する **Impact Level (1〜4)** による自動リスク算出。
  * **高可用性（FMC High Availability）**: Active/Standby 冗長構成による管理プレーンの可用性確保とリアルタイムの構成データベース同期。
* **利用目的**:
  * エンタープライズ全域に分散配置された多数の FTD アプライアンスに対するセキュリティポリシーの一括作成・検証・配布（Deploy）。
  * ネットワーク内で発生した通信イベント（Connection Events）、セキュリティ侵害イベント（Intrusion/File/Malware Events）の統合相関分析およびレトロスペクティブ分析。
* **どのような場面で利用するか**:
  * 境界セキュリティ（Perimeter）、データセンター、支社（Branch）の FTD アプライアンス集中管理。
  * Cisco ISE との pxGrid 連携による Identity-Based Access Control（SGT / Active Directory ユーザー単位のアクセス制御）の実装。
  * 複合脅威（ゼロデイ攻撃、マルウェア感染、C2 通信）の迅速な検知および Rapid Threat Containment (RTC / ANC) による感染端末の自動隔離。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **役割** | 多層セキュリティポリシー定義、FTD デバイス制御、イベントログ集約、脅威相関分析 |
| **アーキテクチャ** | 物理アプライアンス (FMC 1600/2600/4600 等) または 仮想アプライアンス (FMCv: VMware, KVM, AWS, Azure) |
| **管理プロトコル** | `sftunnel` (TCP 8305: 双方向 TLS 暗号化トンネル) による FTD ↔ FMC 間制御・イベント伝送 |
| **主要ポリシー** | Prefilter, Access Control (ACP), Intrusion (IPS), File/AMP, SSL, Identity, Network Discovery, NAT, FlexConfig |
| **冗長化構成** | FMC High Availability (Active/Standby データベース同期、手動プロモート方式) |
| **ライセンス体系** | Cisco Smart Licensing (Base, Threat, Malware, URL Filtering) |
| **設計上の注意点** | FMC と FTD 間の管理通信断（`sftunnel` ダウン）時も FTD のデータプレーン転送および Snort 処理は継続する |

---

## 🏗 動作原理

FMC はデータプレーンパケット処理に直接関与せず、`sftunnel` (TCP 8305) 経由で FTD へ設定ポリシーをプッシュ（Deploy）し、FTD から発生する各種イベント（Connection, Intrusion, Malware, Discovery, Health）を受信・データベース化して解析します。

### 1. FMC ポリシー階層アーキテクチャと FTD パケット処理パイプライン

```text
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      Firepower Management Center (FMC)                          │
│                                                                                 │
│   [ Access Control Policy (ACP) - Parent Policy ]                               │
│   ├── Prefilter Policy (Fastpath / Tunnel / Block)                              │
│   ├── Security Intelligence (IP / DNS / URL Feeds)                              │
│   ├── Identity Policy (Active/Passive Auth, ISE pxGrid, SGT)                    │
│   ├── SSL Policy (Decrypt-Resign, Decrypt-Known Key, Block, Bypass)             │
│   ├── Access Control Rules (L3-L7, App, User, SGT, URL)                         │
│   │     ├── Intrusion Policy (Snort 2 / Snort 3, SRU Rules)                     │
│   │     └── File / AMP Policy (File Control, Malware Cloud Lookup)              │
│   └── Default Action (Block / Trust / Network Discovery / Intrusion)           │
└─────────────────────────────────────┬───────────────────────────────────────────┘
                                      │ Policy Deployment (sftunnel: TCP 8305)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    Firepower Threat Defense (FTD) Data Plane                    │
│                                                                                 │
│  [ Ingress Packet ]                                                             │
│         │                                                                       │
│         ▼                                                                       │
│  ┌──────────────┐    Fastpath    ┌───────────────────────────────────────────┐  │
│  │ Prefilter    ├───────────────►│ L2/L3 Routing & Egress Forwarding         │  │
│  └──────┬───────┘                └─────────────────────▲─────────────────────┘  │
│         │ Analyze                                      │                        │
│         ▼                                              │                        │
│  ┌──────────────┐   Block/Drop   ┌───────────────────┐ │ Trust / Pass           │
│  │ Sec Intel    ├───────────────►│ Drop Packet & Log │ │                        │
│  └──────┬───────┘                └───────────────────┘ │                        │
│         │ Pass                                         │                        │
│         ▼                                              │                        │
│  ┌──────────────┐                                      │                        │
│  │ Identity/SSL ├────────┐                             │                        │
│  └──────┬───────┘        │                             │                        │
│         │                ▼                             │                        │
│         │        ┌──────────────┐                      │                        │
│         │        │ App / URL    │                      │                        │
│         │        └──────┬───────┘                      │                        │
│         │               │                              │                        │
│         ▼               ▼                              │                        │
│  ┌──────────────────────────────┐                      │                        │
│  │ File / AMP & Snort IPS Engine├──────────────────────┘                        │
│  └──────────────────────────────┘                                               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

FMC を介したポリシー適用およびイベント相関解析の処理シーケンスは以下の通りです。

1. **管理通信の確立 (`sftunnel`)**:
   * FTD 上で `configure manager add <FMC_IP> <REG_KEY>`、FMC 上で `Devices > Add Device` を実行。
   * ポート TCP 8305 上で双方向 TLS 証明書ハンドシェイクが完了し、`sftunnel` が確立。
2. **オブジェクトおよびポリシーの定義**:
   * FMC 上でオブジェクト（Network, Port, URL, SGT, Dynamic Object 等）を作成し、ACP およびサブポリシー（Prefilter, Intrusion, File, SSL 等）を構成。
3. **ポリシー展開 (Deployment)**:
   * FMC から FTD へポリシーを展開。FTD 側で LINA コンフィグおよび Snort ルールセット（C3PL / AC ACL）に変換されてアクティブ化（必要に応じて Snort プロセスがリロード）。
4. **イベント収集と相関・影響度解析**:
   * FTD がトラフィックを検出し、発生したイベント（Intrusion, Connection, File/Malware）を FMC へ送出。
   * FMC は Network Discovery データベース（ホストの OS/サービス/脆弱性）と照合し、攻撃の有効性を評価して **Impact Level (1: Victim Host Vulnerable 〜 4: Unknown)** を算出。
5. **ヘルスモニタリング**:
   * FMC は定義された Health Policy（CPU, Memory, Disk, Snort Statistics, HA Status 等）に基づき定期的に FTD の状態を監視。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Access Control Policy (ACP) 内のルール評価順序と処理アクション**:
   * ACP ルールは上から順（Top-Down）に最初のマッチ（First-Match）で評価されます。
   * アクション種別の完全な理解が不可欠です。
     * **`Trust`**: Snort IPS、File/AMP、Network Discovery をバイパスして高速通過（ただし Security Intelligence、Identity、QoS は適用）。
     * **`Allow`**: 後続の Inspection（Intrusion Policy、File/Malware Policy）へパケットを引き渡し。
     * **`Block` / `Block with reset`**: 即座にドロップ（TCP RST を送信）。
     * **`Interactive Block`**: Web ブラウザに警告画面を表示し、ユーザーが「続行」を選択した場合に通信を許可。
     * **`Monitor`**: パケットをログ記録するのみで、ルールのマッチング処理を停止させず後続のルール評価を継続。
2. **Prefilter Policy の設定と用途**:
   * 高負荷な大容量通信（Elephant Flow）やカプセル化（GRE, IP-in-IP, Tunnel）パケットを、Snort (DPI) エンジンに渡す前に L2/L4 レベルで高速バイパス（`Fastpath`）または遮断（`Block`）するためのポリシー。
   * 設定場所: `Policies > Access Control > Prefilter` で定義し、ACP の `Advanced` タブでバインド。
3. **Identity Policy と ISE pxGrid 連携**:
   * パッシブ認証（ISE pxGrid 経由の IP-to-User / IP-to-SGT マッピング）とアクティブ認証（Captive Portal / HTTP Response）の構成。
   * Active Directory レルム (Realm) を `Integration > Realms` で登録・同期させておくことが前提。
4. **File Policy / AMP for Networks の構成**:
   * ファイル転送（HTTP, FTP, SMTP, POP3, IMAP, SMB）における拡張子ブロック（File Control）および SHA-256 ハッシュによる AMP クラウド照会（Malware Cloud Lookup）。
   * サポートされるアクション: `Block Files`, `Block Malware`, `Detect Files`, `Malware Cloud Lookup`。
5. **FlexConfig Policy の活用と限界**:
   * FMC GUI でネイティブサポートされていない ASA / LINA CLI コマンド（例: WCCP, EIGRP, BFD, 特定の MPF コマンド、検査ポリシー拡張等）を FTD に追加投入するためのテキストベースポリシー。
   * **注意**: FlexConfig 内でシークレットキー（Secret Key）変数を使用しているポリシーを Export/Import した場合、暗号キー値がクリアされるため、Import 後に手動で再設定が必要。
6. **FMC High Availability (HA) の同期挙動と障害復旧**:
   * FMC HA は **Active/Standby** 方式。アクティブ側のみが FTD の管理および Deploy を実行可能。
   * データベース（ポリシー、オブジェクト、イベント）は同期されますが、アクティブ機ダウン時のスタンバイ機切り替えは **自動フェイルオーバーではなく、手動での Standby ➔ Active プロモート操作** が必要。
   * HA ペアを組む両 FMC は、同一モデル、同一メジャー/マイナー/メンテナンスバージョン、同一 SRU（Intrusion Rule）、同一 VDB（Vulnerability Database）バージョンである必要があります。

---

## 🛠 設定方法

### 1. FMC GUI 設定手順フロー

```text
[ Step 1: Objects 作成 ]
  Objects > Object Management
  ├── Network Objects / Groups
  ├── Port Objects / Groups
  ├── Security Group Tag (SGT) Objects
  └── Variable Sets (例: default-set の $HOME_NET 調整)

[ Step 2: サブポリシーの作成 ]
  Policies > Intrusion > Create Policy (Balanced Security and Connectivity)
  Policies > Access Control > Malware & File > Create File Policy
  Policies > Access Control > Prefilter > Create Prefilter Policy

[ Step 3: Access Control Policy (ACP) の定義 ]
  Policies > Access Control > Add Policy
  ├── Mandatory Section (上位優先ルール)
  ├── Category 追加によるルールの構造化
  ├── Rule 条件設定 (Zones, Networks, Ports, Applications, Users, SGT, URLs)
  ├── Rule Inspection (Intrusion Policy & File Policy のバインド)
  ├── Rule Logging (Log at End of Connection ➔ FMC / Syslog 指定)
  └── Advanced Tab (Prefilter Policy, SSL Policy, Identity Policy の適用)

[ Step 4: ポリシー展開 (Deployment) ]
  Deploy > Deployment > Select Devices ➔ Click "Deploy"
```

### 2. FTD CLI: FMC 管理サーバーの登録および初期化コマンド

```bash
! ===== 1. FTD 上で FMC を管理サーバーとして登録 =====
configure manager add 192.168.10.50 C1sco12345

! NAT 経由で FMC に登録する場合の Unique ID 指定
configure manager add 192.168.10.50 C1sco12345 DEREK_FTD01

! ===== 2. 管理接続状態の確認 =====
show manager

! ===== 3. FMC 側の登録変更コマンド (FMC IP 変更時) =====
configure manager edit
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI / FMC Bash) | 表示確認ポイント |
| :--- | :--- | :--- |
| **FTD 側の FMC 登録状態・接続ステータス確認** | `show manager` | 管理サーバー IP、登録状態 (`Completed`)、`sftunnel` 接続状態 |
| **`sftunnel` 通信セッションおよび証明書検証** | `show sftunnel status` | TCP 8305 トンネルのアップ/ダウン、暗号化ステータス |
| **展開された Access Control Rules (LINA 展開 ACL) 確認** | `show access-list` | FMC から展開された `CSM_FW_ACL_` の ACE エントリと hitcnt |
| **Prefilter ルールの展開状態確認** | `show access-list \| include PREFILTER` | Prefilter ポリシーから変換された LINA Fastpath ルール |
| **Snort 2 / Snort 3 プロセス状態確認** | `show summary` / `show snort status` | Snort インスタンス数、PID、ルールロード状態 |
| **Hit Count 統計のリアルタイムクリア** | `clear access-list CSM_FW_ACL_ counters` | 展開済み ACL のヒットカウントの初期化 |
| **FMC プロセス状態の確認 (FMC Bash)** | `sudo pmtool status` | FMC 内部 daemon (Correlator, Database, HTTPD 等) の動作 |
| **FMC イベントレート・パフォーマンス統計確認 (FMC Bash)** | `sudo perfstats -Cq` | FMC が受信している 1 秒あたりのイベント処理レート |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **FMC に FTD を追加登録できない (`sftunnel` 失敗)** | TCP ポート 8305 が途中 FW でブロック、または Registration Key / NAT ID の不一致 | `show manager`<br>`show sftunnel status` | ポート 8305 の到達性を確認し、`configure manager add` で Key / NAT ID を再設定。 |
| **FMC から FTD への Deploy が失敗する** | FTD 側のディスク容量不足、またはオブジェクト参照エラー/構文競合 | `show disk`<br>FMC `Task Status` ログ | 不正な FlexConfig や未定義オブジェクトを修正し、FTD の `/var/log` をクリーンアップ。 |
| **Deploy 時に「Inspect Interruption: Yes」と表示される** | Snort プロセスの再起動を伴う変更（VDB / SRU 更新、NAP 変更等） | FMC `Deploy Preview` 画面 | 業務影響を避けるため、メンテナンスウィンドウに展開を実行する。 |
| **ISE pxGrid から SGT / ユーザー情報が FMC に同期されない** | FMC ↔ ISE 間の pxGrid 信頼証明書エラー、または pxGrid クライアント未承認 | FMC `Integration > Other Integrations` | ISE 上で FMC pxGrid クライアントの Status を `Approved` に変更し、CA 証明書を正しくインポート。 |
| **FMC HA が「Degraded」または「Split-Brain」状態になる** | FMC 間ハートビート途絶、または SRU / VDB バージョン不一致 | `Policies > System > Configuration > High Availability` | 両 FMC の SRU/VDB バージョンを一致させ、アクティブ機を手動プロモートする。 |
| **接続イベント (Connection Events) が FMC に記録されない** | ACP ルールの Logging タブで `Log at End of Connection` が未選択 | FMC `Policies > Access Control` (Logging Tab) | 該当ルールまたは Default Action で `Log at End of Connection` を有効化して Deploy。 |

---

## ⚠ 制限事項

* **FMC High Availability の制約**:
  * スタンバイ FMC はアクティブ FMC から常時データベース同期を受け取りますが、管理 UI は読み取り専用（または限定状態）となり、**FTD へのポリシー展開コマンドを実行できるのはアクティブ FMC のみ** です。
  * アクティブ FMC 障害時、自動フェイルオーバー（Auto-Failover）は発生しません。管理者が手動でスタンバイ FMC をアクティブへ昇格（Promote）させる必要があります。
* **イベント保存数とディスク容量制限**:
  * FMC のモデルごとに保存可能なイベント上限数（例: FMC 2600 は最大 1 億イベント）が規定されており、上限を超過した古いイベントから自動的に上書き消去されます。
* **Multi-Instance (FTD 容器化) における FMC 管理制限**:
  * Firepower 4100 / 9300 シャーシ上で FTD コンテナインスタンスを運用する場合、**同一セキュリティモジュール上のすべての FTD インスタンスは単一の FMC で管理しなければならない** という制約が存在します。

---

## 🔄 他技術との関連

* **Cisco ISE (Identity Services Engine)**: pxGrid (Platform Exchange Grid) 経由で FMC と統合し、ユーザー ID、IP アドレス、Security Group Tag (SGT) のコンテキストマッピングを取得して Identity-Aware ACP を実現。
* **Cisco SecureX / Cisco Analytics (Stealthwatch)**: FMC は eStreamer (Event Streamer API) または Syslog 経由でセキュリティイベントおよび接続イベントを外部 SIEM や XDR プラットフォームへリアルタイム出力。
* **AMP for Endpoints (Cisco Secure Endpoint)**: FMC 上で Endpoint Malware イベントを同期表示し、ネットワーク側 (AMP for Networks) と端末側 (AMP for Endpoints) のクロスプラットフォーム脅威追跡 (Device Trajectory) を提供。

---

## 🧩 比較表

### 1. FMC (Firepower Management Center) vs FDM (Firepower Device Management)

| 比較項目 | FMC (Firepower Management Center) | FDM (Firepower Device Management) |
| :--- | :--- | :--- |
| **管理方式** | **中央集中型（マルチデバイス管理）** | **オンボックス型（単一 FTD Web UI）** |
| **管理スケール** | 多数の FTD アプライアンス（最大数百台） | 単一アプライアンス（ローカル管理） |
| **高度機能サポート** | **完全対応** (Prefilter, Advanced Malware, TID, Custom Workflows, Correlation) | 限定的 (基本的な ACP, NAT, IPS, Simple File Policy) |
| **ISE 連携** | **pxGrid 経由での全機能統合** (SGT, Passive Identity, Dynamic ANC) | パッシブ Identity のみ（pxGrid 高度機能不可） |
| **運用シナリオ** | エンタープライズ、データセンター、多拠点統合管理 | 小規模拠点、スタンドアロン FTD 運用 |

### 2. ACP Rule Actions (Allow vs Trust vs Fastpath)

| アクション | 定義場所 | スキャン対象 | 用途・メリット |
| :--- | :--- | :--- | :--- |
| **`Allow`** | Access Control Policy (ACP) | 全エンジン（SI, Identity, App, File/AMP, Snort IPS） | **標準的なセキュリティ保護通信** |
| **`Trust`** | Access Control Policy (ACP) | L3/L4 ACL, SI, Identity（**Snort IPS と File/AMP はバイパス**） | 信頼された内部通信の Snort CPU 負荷削減 |
| **`Fastpath`** | Prefilter Policy | L2-L4 ヘッダーのみ（**すべての DPI / Snort を最前線で除外**） | バックアップ通信、大容量 Elephant Flow の超高速転送 |

---

## 💡 ベストプラクティス

1. **ログ出力設定の最適化 (`Log at End of Connection`)**:
   * 高トラフィック環境において、接続の開始（`Log at Beginning of Connection`）と終了（`Log at End of Connection`）の両方をログ記録すると、FMC のデータベースおよびネットワーク帯域を過剰に消費します。**接続終了時のみ (`Log at End of Connection`) 記録する** のが Cisco 推奨構成です。
2. **Variable Set (\$HOME_NET) の正確なチューニング**:
   * デフォルトの `$HOME_NET` (`any`) のまま Snort IPS を運用すると、ルールの評価範囲が広がり誤検知（False Positive）や CPU 負荷増大の原因となります。`Objects > Object Management > Variable Set` で `$HOME_NET` を自社の実際の内部 IP サブネットにカスタマイズします。
3. **Prefilter Policy による非検査トラフィックの早期バイパス**:
   * データセンター間の大容量バックアップ通信や IP-in-IP / GRE トンネルは、Prefilter Policy で `Fastpath` ルールを作成し、Snort DPI エンジンを完全通過させて FTD のスループット性能を最大化します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: FMC への FTD デバイス登録と初期ポリシー適用

#### 問題
FMC (`192.168.10.50`) において、新規 FTD アプライアンス (`192.168.10.101`) を登録キー `C1sco12345` を使用して追加し、アクセス制御ポリシー `DMZ_PROTECT_ACP` を適用して展開せよ。

#### 要件
* FTD CLI で登録コマンドを実行すること。
* FMC GUI 上で Access Control Policy をバインドして Deploy すること。

#### 設定・操作手順

```bash
! ===== 1. FTD CLI での FMC 登録コマンド =====
configure manager add 192.168.10.50 C1sco12345
```

```text
! ===== 2. FMC GUI でのデバイス追加操作 =====
1. Devices > Device Management に移動。
2. "Add" > "Add Device" を選択。
3. Host: 192.168.10.101, Display Name: FTD-01, Registration Key: C1sco12345 を入力。
4. Access Control Policy: "DMZ_PROTECT_ACP" を選択し "Register" をクリック。
5. 右上の "Deploy" ボタンをクリックし、FTD-01 を選択して "Deploy" を実行。
```

#### 検証方法
* FTD CLI で `show manager` を実行し、`Manager: 192.168.10.50` および `Status: Completed` と表示されることを確認。

---

### 例2: Access Control Policy (ACP) における L7 アプリケーションおよび URL カテゴリブロック

#### 問題
FMC 上で ACP ルールを作成し、内部ネットワーク (`INSIDE_NET: 10.1.1.0/24`) から外部への通信のうち、P2P アプリケーション (`BitTorrent`) および URL カテゴリ `Gambling` 宛ての通信を遮断（Block with reset）し、ログを記録せよ。

#### 要件
* `Block with reset` アクションを使用すること。
* `Log at End of Connection` を有効化すること。

#### 設定・操作手順

```text
1. Policies > Access Control > "DMZ_PROTECT_ACP" を編集。
2. "Add Rule" をクリック。
   - Name: BLOCK_P2P_GAMBLING
   - Action: Block with reset
   - Zone: Source = INSIDE_ZONE, Destination = OUTSIDE_ZONE
   - Networks: Source = INSIDE_NET (10.1.1.0/24)
   - Applications: Applications タブで "BitTorrent" を検索して追加
   - URLs: Category タブで "Gambling" を選択して追加
   - Logging: "Log at End of Connection" にチェック
3. ルールを保存し、FTD へ Deploy。
```

#### 検証方法
* クライアントから BitTorrent 接続またはギャンブルサイトへアクセスを試み、接続がリセット拒否されることを確認。
* FMC 上で `Analysis > Connections > Events` を開き、Action が `Block with reset` のイベントログを確認。

---

### 例3: Prefilter Policy による大容量バックアップ通信の Fastpath 化

#### 問題
内部バックアップサーバー (`10.1.1.200`) と外部バックアップストレージ (`198.51.100.200`) 間の TCP `445` (SMB) 通信を、Snort DPI エンジンに引き渡さず最前線で高速通過（Fastpath）させる Prefilter Rule を作成せよ。

#### 要件
* `Policies > Access Control > Prefilter` で構成すること。
* ACP の `Advanced` タブで適用すること。

#### 設定・操作手順

```text
1. Policies > Access Control > Prefilter > "Add Prefilter Policy" (Name: CUSTOM_PREFILTER)
2. "Add Prefilter Rule" を追加。
   - Name: FASTPATH_BACKUP
   - Action: Fastpath
   - Source IP: 10.1.1.200
   - Destination IP: 198.51.100.200
   - Service: TCP / Port 445
3. Policies > Access Control > "DMZ_PROTECT_ACP" を編集。
4. "Advanced" タブ > "Prefilter Policy Options" で "CUSTOM_PREFILTER" を選択して保存。
5. FTD へ Deploy。
```

#### 検証方法
* FTD CLI で `show access-list | include PREFILTER` を実行し、該当 IP/Port に対する Fastpath ルールが LINA ACL として展開されていることを確認。

---

### 例4: File Policy による実行可能ファイル (.exe) のブロックおよび AMP Malware Cloud Lookup

#### 問題
Web 閲覧 (HTTP/HTTPS) におけるファイルのダウンロード通信に対し、すべての `.exe` ファイルのダウンロードを即座にブロック（Block Files）し、PDF ファイルについては SHA-256 マルウェアクラウド照会（Malware Cloud Lookup）を行う File Policy を構成せよ。

#### 要件
* File Policy 名: `WEB_FILE_POLICY`
* ACP ルールの Inspection タブで適用すること。

#### 設定・操作手順

```text
1. Policies > Access Control > Malware & File > "Add File Policy" (Name: WEB_FILE_POLICY)
2. "Add Rule" を追加 (Rule 1):
   - Application: HTTP, Inline FTP
   - Direction of Transfer: Download
   - File Types: Executable (EXE)
   - Action: Block Files
3. "Add Rule" を追加 (Rule 2):
   - Application: HTTP
   - Direction of Transfer: Download
   - File Types: PDF
   - Action: Malware Cloud Lookup
4. ACP の WEB 許可ルール (`ALLOW_WEB`) の "Inspection" タブを開き、File Policy として "WEB_FILE_POLICY" をバインド。
5. FTD へ Deploy。
```

#### 検証方法
* クライアントから `.exe` ファイルをダウンロードし、遮断されることを確認。
* FMC 上で `Analysis > Files > Events` および `Malware Events` を確認。

---

### 例5: Security Intelligence (SI) による悪意ある IP / Domain の自動ブロック

#### 問題
Cisco が提供する動的脅威フィード (Security Intelligence) を使用して、既知の悪意ある IP アドレス (`Cisco-Intelligence-Feed`) および不審なドメイン宛ての通信を ACP の前段で自動ブロック（Block）せよ。

#### 要件
* ACP の `Security Intelligence` タブで設定すること。

#### 設定・操作手順

```text
1. Policies > Access Control > "DMZ_PROTECT_ACP" を編集。
2. "Security Intelligence" タブを選択。
3. "Networks" セクションで、Available Networks 内の "Cisco-Intelligence-Feed" を "Blacklist" に移動。
4. "URL" セクションで、Available URLs 内の "Cisco-URL-Feed" を "Blacklist" に移動。
5. Logging アイコンをクリックし、"Log at Beginning of Connection" にチェックを入れて保存。
6. FTD へ Deploy。
```

#### 検証方法
* FMC の `Analysis > Connections > Security Intelligence Events` を確認し、ブラックリスト IP/Domain 宛ての通信が即座にブロックされていることを検証。

---

### 例6: Identity Policy と Cisco ISE pxGrid 連携による SGT (Security Group Tag) ベースアクセス制御

#### 問題
Cisco ISE から pxGrid 経由で取得した Security Group Tag (`Employees`: SGT 10) から `PCI_Servers` (SGT 20) 宛ての通信のみを許可し、その他の SGT からのアクセスを拒否する ACP ルールを構成せよ。

#### 要件
* SGT オブジェクトを使用して ACP ルールを作成すること。

#### 設定・操作手順

```text
1. Objects > Object Management > Security Group Tag を開き、ISE から同期された SGT (Employees: 10, PCI_Servers: 20) を確認。
2. Policies > Access Control > "DMZ_PROTECT_ACP" を編集。
3. "Add Rule" を追加:
   - Name: PERMIT_EMPLOYEE_TO_PCI
   - Action: Allow
   - Source SGT: Employees (10)
   - Destination SGT: PCI_Servers (20)
   - Logging: Log at End of Connection
4. デフォルトアクションまたは後続ルールでその他の通信を Block。
5. FTD へ Deploy。
```

#### 検証方法
* FTD CLI で `show access-list` を実行し、SGT 属性 (`security-group source/destination`) を含んだ ACE が正しく展開されていることを確認。

---

### 例7: Variable Set (\$HOME_NET) のカスタマイズと Intrusion Policy の適用

#### 問題
FMC のデフォルト変数セット `default-set` における `$HOME_NET` を自社ネットワーク `10.1.0.0/16` に変更し、Intrusion Policy `Balanced Security and Connectivity` を ACP のデフォルトアクションに適用せよ。

#### 要件
* Variable Set の編集を行うこと。

#### 設定・操作手順

```text
1. Objects > Object Management > Variable Set を開き、"default-set" を編集。
2. Variable "$HOME_NET" の値を "any" から "10.1.0.0/16" に変更して保存。
3. Policies > Access Control > "DMZ_PROTECT_ACP" を編集。
4. 画面最下部の "Default Action" を "BLOCK ALL TRAFFIC" から "INTRUSION PREVENTION" に変更。
5. Intrusion Policy に "Balanced Security and Connectivity"、Variable Set に "default-set" を選択して保存。
6. FTD へ Deploy。
```

#### 検証方法
* `Analysis > Intrusions > Events` を開き、検出された IPS イベントの宛先 IP が `$HOME_NET` (`10.1.0.0/16`) 内のホストとして正しく識別されていることを確認。

---

### 例8: FlexConfig Policy による FTD への WCCP (Web Cache Communication Protocol) 追加投入

#### 問題
FMC GUI で未サポートの LINA コマンド（WCCP グループ設定）を FTD に追加適用するため、FlexConfig Text Object および FlexConfig Policy を作成して展開せよ。

#### 要件
* `FlexConfig Object` と `FlexConfig Policy` を使用すること。

#### 設定・操作手順

```text
1. Objects > Object Management > FlexConfig > FlexConfig Object を作成。
   - Name: WCCP_CONFIG_OBJ
   - Deployment: Once / Every Deploy
   - Content (CLI):
     wccp web-cache group-address 239.255.0.1 redirect-list ACL_WCCP
2. Policies > Access Control > FlexConfig > "Add FlexConfig Policy" を作成。
3. Left Pane から "WCCP_CONFIG_OBJ" を選択し、Selected Appended FlexConfigs に追加して保存。
4. 対象 FTD にポリシーをアサインし、Deploy。
```

#### 検証方法
* FTD CLI で `show running-config wccp` を実行し、FlexConfig で指定した WCCP コマンドが反省されていることを確認。

---

### 例9: FMC High Availability (HA) の設定およびスタンバイ機の手動昇格 (Promote)

#### 問題
2 台の FMC (`FMC-01: 192.168.10.50` [Active] と `FMC-02: 192.168.10.51` [Standby]) 間で FMC HA を構成せよ。障害発生を想定し、FMC-02 をアクティブへ昇格させる手順を確認せよ。

#### 要件
* `Policies > System > Configuration > High Availability` から設定すること。

#### 設定・操作手順

```text
1. FMC-01 (Active) の UI にログイン。
2. System > Configuration > High Availability に移動。
3. "Register High Availability" をクリック。
   - Peer FMC IP: 192.168.10.51
   - Shared Secret: HAkey12345
4. データベース同期が完了し、Status が "Normal (Active / Standby)" になることを確認。
! ===== スタンバイ機の昇格操作 =====
5. 障害時、FMC-02 (Standby) UI の High Availability 画面を開く。
6. "Make Active" (Promote) ボタンをクリックし、FMC-02 を Active 状態へプロモート。
```

#### 検証方法
* FMC-02 の画面上で HA Status が `Active` に遷移し、FTD の管理および Deploy 操作が可能になったことを確認。

---

### 例10: Network Discovery Policy のカスタマイズと Impact Level 評価の検証

#### 問題
デフォルトで全ネットワーク対象となっている Network Discovery Policy を変更し、監視対象を社内サブネット `10.1.0.0/16` のみに限定せよ。発見されたホスト情報に基づき、IPS イベントの Impact Level がどう変化するか検証せよ。

#### 要件
* `Policies > Network Discovery` で範囲を限定すること。

#### 設定・操作手順

```text
1. Policies > Network Discovery を編集。
2. デフォルトルール "All-IP-Subnets" (0.0.0.0/0) のネットワーク範囲を "10.1.0.0/16" に修正。
3. Discovery 対象として "Hosts", "Applications", "Users" をすべて有効化して保存し、Deploy。
4. 分析: Analysis > Assets > Hosts で 10.1.0.0/16 内のホスト OS / サービスが自動登録されたことを確認。
```

#### 検証方法
* 脆弱性が登録された内部ホスト宛てに攻撃パケットを送信し、`Analysis > Intrusions > Events` で該当イベントの **Impact Level が 1 (Red: Host Vulnerable)** として優先表示されることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で Access Control Policy (ACP) を構成し、特定の社内サーバー宛てトラフィックに対してアクション `Trust` を指定したルールを定義した。このルールにマッチしたトラフィックのパケット処理動作として正しいものはどれか。

* A) パケットはあらゆる検査を受けずに最前線で Fastpath 処理され、QoS や Security Intelligence もバイパスされる。
* B) L3/L4 ACL および Security Intelligence、Identity Policy は適用されるが、Snort IPS エンジン、File/AMP Policy、Network Discovery のディープパケットインスペクション (DPI) はバイパスされる。
* C) パケットはログのみ記録され、後続の ACP ルールが上から順に継続評価される。
* D) Snort IPS エンジンによるインスペクションは実行されるが、File Policy のみがバイパスされる。

**正解**: **B**
**解説**: ACP における **`Trust`** アクションは、L3/L4 ACL や Security Intelligence、Identity、QoS ポリシーを適用して通信を許可しますが、CPU 負荷の高い **Snort IPS エンジン、File/AMP Policy、Network Discovery などの L7 ディープパケットインスペクション (DPI) をバイパス** させます。すべての検査（SI や QoS を含む）を完全バイパスさせるには Prefilter Policy の `Fastpath` を使用します。

---

### 2. トラブルシュート問題

**問題**: FTD アプライアンス上で `configure manager add 192.168.10.50 C1sco12345` を実行し、FMC 上で Add Device 操作を行ったが、登録がタイムアウトエラーで失敗した。FTD CLI で `show manager` を実行したところステートが `Pending` のままであった。要因のトラブルシューティングとして最初に確認すべき項目はどれか。

* A) FTD と FMC 間の TCP ポート `8305` 通信が途中のファイアウォールでドロップされていないか。
* B) FTD 上で SSH (TCP 22) ポートが開いているか。
* C) FMC 上で Active Directory レルムが未構成であるか。
* D) FTD 上で `ip nbar protocol-discovery` が無効化されているか。

**正解**: **A**
**解説**: FMC と FTD 間の制御・イベント通信チャネル (`sftunnel`) は **TCP ポート 8305** を使用して TLS 暗号化トンネルを確立します。登録が Pending で止まる場合、ポート 8305 の疎通不可、または Registration Key / NAT ID の不一致が主要因です。

---

### 3. Design 問題

**問題**: データセンター内の FTD において、毎秒数 10 Gbps に達するデータベースの非暗号化大容量バックアップ通信（10.10.10.0/24 ➔ 10.20.20.0/24）が通過しており、Snort IPS エンジンの CPU リソースを激しく消費して他の重要通信の遅延を引き起こしている。セキュリティ要件を満たしつつ、FTD の転送パフォーマンスを最も効果的に改善するための設計手法はどれか。

* A) ACP ルールで該当通信に `Allow` アクションを指定し、Intrusion Policy をドロップする。
* B) Prefilter Policy で該当の送信元/宛先 IP サブネットを指定した `Fastpath` ルールを作成し、ACP の Advanced タブで適用する。
* C) FMC の Health Policy で CPU 警告閾値を 99% に引き上げる。
* D) Dynamic PAT ルールを追加して IP アドレスを多重化する。

**正解**: **B**
**解説**: L2-L4 ヘッダーのみで特定可能な大容量バックアップ通信（Elephant Flow）等は、**Prefilter Policy の `Fastpath` ルール** を使用して Snort DPI エンジンに渡す前に最前線でバイパスさせるのが最も効果的です。これにより FTD の Snort CPU 負荷が極限まで削除され、スループットが大幅に向上します。

---

### 4. 実装問題

**問題**: FMC High Availability (HA) 環境において、プライマリ FMC (Active) にハードウェア障害が発生し完全にダウンした。スタンバイ FMC (Standby) を使用して FTD の管理および緊急のポリシー展開を再開するための正しい運用手順はどれか。

* A) スタンバイ FMC が自動的に Active へフェイルオーバーするため、何も操作せず Deploy を実行する。
* B) スタンバイ FMC の GUI にログインし、High Availability 画面から手動で "Make Active" (Promote) を実行してアクティブへ昇格させる。
* C) FTD CLI で `configure manager add` を実行し、スタンバイ FMC の IP アドレスを再登録する。
* D) スタンバイ FMC を再起動（Reboot）すると自動的に Active になる。

**正解**: **B**
**解説**: FMC HA は自動フェイルオーバーをサポートしていません。アクティブ FMC 障害時は、管理者がスタンバイ FMC の UI にログインし、**手動で Promte ("Make Active") 操作を実行** してスタンバイ機をアクティブ機へ昇格させる必要があります。

---

### 5. コンフィグ読解問題

**問題**: 以下の図は FMC の Intrusion Event 解析画面において、あるホスト宛ての IPS イベントの **Impact Level** が表示されたものである。Impact Level が **1 (Red: Host Vulnerable)** と判定される基準として正しいものはどれか。

* A) 攻撃の送信元 IP アドレスが FMC の Blacklist に登録されている場合。
* B) 攻撃のターゲット（宛先ホスト）が Network Discovery によって検出されており、そのホストの OS/サービス/脆弱性データベース（VDB）と一致する攻撃シグネチャが検知された場合。
* C) 攻撃パケットが FTD によって正常にドロップ（Block）された場合。
* D) ターゲットホストがネットワークマップ上に存在せず、脆弱性が不明な場合。

**正解**: **B**
**解説**: FMC の **Impact Level 1 (Red)** は、「ターゲットホストが Network Discovery で把握されており、そのホストが対象の攻撃に対して**実際に脆弱である（Vulnerable）**」と FMC が相関判定した場合に割り当てられます。セキュリティアナリストが最優先で対処すべき重大なアラートです。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Administration Guide 7.1]([suspicious link removed])
* [Cisco Secure Firewall Threat Defense Device Configuration Guide with FMC 7.1]([suspicious link removed])
* [Cisco Technical Notes: Troubleshoot FTD Registration to FMC with sftunnel]([suspicious link removed])
* [Cisco Technical Notes: Firepower Management Center High Availability Configuration and Troubleshooting]([suspicious link removed])
* [Cisco Live: Cisco Secure Firewall FMC Architecture and Policy Deep Dive (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験における FMC の操作では、**「オブジェクト作成 ➔ ポリシーバインド ➔ ACP ルール適用 ➔ Deploy」** の一連の流れを素早く迷わずに実行できる手戻りのないGUI操作熟練度が勝敗を分けます。
* **図解（FMC パケットインスペクション・パイプライン概要）**:
  ```text
  [ Ingress Packet ]
          │
          ▼
  ┌────────────────────────────────────────────────────────┐
  │ 1. Prefilter Policy (Fastpath / Analyze / Block)       │
  └────────────────────────┬───────────────────────────────┘
                           │ Analyze
                           ▼
  ┌────────────────────────────────────────────────────────┐
  │ 2. Security Intelligence (IP / Domain / URL Blacklist) │
  └────────────────────────┬───────────────────────────────┘
                           │ Pass
                           ▼
  ┌────────────────────────────────────────────────────────┐
  │ 3. Identity Policy & SSL Decryption Policy             │
  └────────────────────────┬───────────────────────────────┘
                           │ Decrypted / Identified
                           ▼
  ┌────────────────────────────────────────────────────────┐
  │ 4. Access Control Rules (App / URL / SGT / L3-L4 Match)│
  │    ├── Action: Block ──► Drop Packet                   │
  │    ├── Action: Trust ──► Pass Packet (Bypass Snort)    │
  │    └── Action: Allow ──► Send to L7 Inspection         │
  └────────────────────────┬───────────────────────────────┘
                           │ Allow
                           ▼
  ┌────────────────────────────────────────────────────────┐
  │ 5. Snort IPS Engine & File/AMP Malware Policy          │
  └────────────────────────┬───────────────────────────────┘
                           │ Clean
                           ▼
  [ Egress Forwarding & Connection Event Logging ]
  ```
* **注意点**:
  * 試験中に FMC の Deploy が失敗する場合は、まず `Deploy > Deployment History` で詳細エラーログを確認し、オブジェクトの重複、未割り当てのインターフェイスゾーン、あるいは不適切な FlexConfig 構文がないかをチェックしてください。
