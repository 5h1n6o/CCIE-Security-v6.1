---
layout: default
title: 1.6.d-AVC
nav_order: 4
parent: 1.6-NGFW-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.6.d AVC (Application Visibility and Control)

Cisco Secure Firewall Threat Defense (FTD) および Management Center (FMC) における **AVC（Application Visibility and Control：アプリケーションの可視化と制御）** は、伝統的な L3/L4 ポート番号（TCP 80 や 443 等）に依存せず、ディープパケットインスペクション（DPI: Deep Packet Inspection）や状態判定エンジン（NBAR2 / Cisco System-Provided Application Detectors / OpenAppID / Custom Detectors）を用いて、トラフィックの実際のアプリケーションプロトコル、クライアントアプリケーション、および Web アプリケーションの動作を精密に識別・制御する核心機能です。

CCIE Security v6.1 Blueprint **1.6.d AVC** では、Access Control Policy (ACP) におけるアプリケーション条件のマッチングロジック、Network Discovery Policy によるアプリケーション自動検出、カスタムアプリケーションディテクター（Custom Application Detectors / OpenAppID）の定義と適用、SSL Inspection との連携動作、非標準ポートでのアプリケーション識別、および Snort エンジンによるリアルタイムデバッグ手法など、高度な構築・検証・トラブルシューティング能力が厳格に評価されます。

---

## 📘 概要

* **機能概要**:
  * **アプリケーションプロトコル（Application Protocols）**: HTTP、SSH、DNS、BitTorrent、TLS など通信のベースとなる L4〜L7 プロトコルを識別。
  * **クライアントアプリケーション（Client Applications）**: Chrome、Firefox、PuTTY、MS Outlook など、通信を発生させているクライアントソフトウェアの種類やバージョンを識別。
  * **Web アプリケーション（Web Applications / Micro-Applications）**: Facebook、YouTube、Office365、Google Drive などの特定 Web サービスおよびその内部機能（例: Facebook Games、YouTube Upload、BitTorrent File Transfer 等のサブ機能）を識別して精細に制御。
  * **Custom Application Detectors / OpenAppID**: システム標準で定義されていない独自の社内アプリや新種の P2P アプリ等を、ポート、文字列、パターングループを定義してカスタム追跡・制御する拡張機能。
* **利用目的**:
  * ポート番号を回避・ステルス通信を行うアプリケーション（例: 80/443 番ポートを悪用する P2P、匿名プロキシ、ウイルストラフィック等）の完全な制御とブロック。
  * 業務上必要な Web サービス（例: Facebook や YouTube）の閲覧は許可しつつ、特定の危険機能（例: Facebook Games や動画アップロード）のみをピンポイントで禁止する「マイクロアプリケーション制御（Granular Control）」。
  * アプリケーションのビジネス関連度（Business Relevance）やリスクレベル（Risk Level）に基づく帯域制御（QoS Policy 連携）およびアクセス制御。
* **どのような場面で利用するか**:
  * 企業ネットワーク境界における業務外 SaaS / クラウドストレージ（例: Personal Dropbox, MegaUpload）の利用禁止とシャドー IT の撲滅。
  * 高リスクな P2P ファイル共有アプリ（BitTorrent、eMule 等）やリモートデスクトップツール（TeamViewer、AnyDesk 等）による情報漏洩防止。
  * 暗号化通信内における精密な Web アプリケーション識別の維持（SSL Inspection Policy との統合）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **識別エンジン** | **Snort 2 / Snort 3 Data Acquisition (DAQ) / NBAR2 / System Detectors / OpenAppID** |
| **検出要素** | **Application Protocol** (例: HTTP), **Client App** (例: Chrome), **Web App** (例: Facebook Games) |
| **評価コンポーネント** | **Access Control Policy (ACP)**: Rules の `Applications` タブ<br>**Network Discovery Policy**: アプリケーション自動発見・インベントリ化<br>**Custom Detectors / OpenAppID**: カスタム識別パターン定義 |
| **識別シーケンス** | パケット着信 ➔ L3/L4 判定 ➔ **最初の数パケット（通常 3〜5 パケット）で Snort がアプリ判定** ➔ ACP ルール判定 |
| **未識別状態 (Unclassified)** | **Pending**（データ不足で判定保留）、**Unknown**（一致パターンなし）、**Blank**（生データ・非アプリ） |
| **復号依存性** | HTTPS などの暗号化トラフィック内の Web アプリケーション（例: Facebook Upload）制御には **SSL Inspection (Decrypt) が必須** |
| **ライセンス要件** | **Base License**（基本的 AVC 機能は Base に含まれる。ただし SI や URL カテゴリ、IPS 連携時は各種サブスクリプションが必要） |
| **ポータビリティ** | **VDB (Vulnerability Database) パック** の更新により、FTD イメージ本体をアップグレードせずに新しいアプリディテクターを追加可能 |

---

## 🏗 動作原理

Cisco FTD の AVC アーキテクチャでは、データプレーンを通過するトラフィックに対して、Snort インスペクションエンジン内のアプリケーションディテクターがパケットの初期フェーズ（ハンドシェイク直後のペイロード）をリアルタイムに解析します。

```text
[ Client Ingress Traffic ] (e.g., TCP 443 to Facebook IP)
       │
       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Firepower Threat Defense (FTD) Packet Processing Architecture                     │
│                                                                                  │
│  [ LINA Firewall Engine ]                                                        │
│   └── Existing Connection / Fastpath / Prefilter / L3-L4 Access List             │
│            │                                                                     │
│            ▼                                                                     │
│  [ Snort Next-Gen Engine ]                                                       │
│   ├── 1. SSL Inspection Policy (Decrypt Encrypted Payload)                       │
│   │        └── Decrypt HTTPS Stream ──► Expose Plaintext HTTP Request            │
│   │                                                                              │
│   ├── 2. Application Inspector (App Detectors / VDB Engine)                      │
│   │        ├── Packet 1-3: TLS Handshake / TCP Syn ──► App State: "Pending"     │
│   │        ├── Packet 4: GET /facebook.com/games ──► App State: "Facebook Games"│
│   │        └── Attribute Tagging: Risk=High, Business Relevance=Low          │
│   │                                                                              │
│   ├── 3. Access Control Policy (ACP) Evaluation                                  │
│   │        └── Match Rule "BLOCK_GAMES":                                         │
│   │                 ├── Applications Condition: Facebook Games                   │
│   │                 └── Action: Block                                            │
│   │                                                                              │
│   └── 4. Action Enforcement & Event Generation                                   │
│            ├── Drop Packet & Send TCP Reset                                      │
│            └── Connection Event Log generated with App Name "Facebook Games" ──► FMC│
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

1. **コネクションの確立と初期パケット通過 (Pending State)**:
   * トラフィック（TCP SYN, SYN-ACK, ACK）が着信した段階では、Snort エンジンはアプリケーション名を確定できません。この状態を **`Pending`** と呼びます。
   * FTD は初期ハンドシェイクを通過させつつ、ペイロードデータの着信を待ちます。
2. **ディープパケットインスペクションとシグネチャ照合**:
   * クライアントが HTTP GET や TLS ClientHello (SNI)、プロトコル特有のヘッダー・シグネチャを送信した時点で、Snort のアプリケーション検出モジュール（VDB パックに収録されたディテクター群）がパターンマッチングを実行します。
3. **アプリケーション属性の確定とタグ付け**:
   * パターンが判定されると、セッションに対して「Application: `Facebook Games`」、「Risk: `High`」、「Category: `Social Networking`」などのメタデータ属性が内部キャッシュ（Flow State）にタグ付けされます。
4. **Access Control Policy (ACP) のルールの再評価**:
   * アプリケーション属性が確定した瞬間、FTD は ACP のルールリストを上位（Top）から再評価します。
   * `Applications` タブの条件（例: `Facebook Games`）に合致するルールが見つかった場合、そのルールに定義されたアクション（`Block`, `Allow`, `QoS Limit`）を執行します。
5. **接続のドロップまたは継続インスペクション**:
   * アクションが `Block` の場合、本線通信は即座にドロップ（または TCP Reset 送信）されます。
   * `Allow` の場合、後続の File Policy / AMP や Intrusion Policy (IPS) 検査を経て Egress インターフェイスから送出されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **暗号化トラフィック (HTTPS) に対する AVC 制限と SSL Inspection の必要性**:
   * **試験の罠**: 「Block Facebook Games inside HTTPS without blocking base Facebook browsing」という要件が出題された場合、**SSL Inspection Policy (Decrypt - Resign) を構成してバインドしていなければ設定は成功しません**。
   * SSL 復号なしの場合、FTD は TLS SNI (`facebook.com`) までしか認識できず、URI パス (`/games`) を識別できないため、ベースの `Facebook` と `Facebook Games` を切り分けることができません。
2. **Network Discovery Policy での「Application Detection」有効化**:
   * FMC の `Policies > Network Discovery` において、対象ネットワーク範囲（例: `10.0.0.0/8`）に対して **`Applications` の自動検出が有効** になっていないと、Context Explorer や Dashboard にアプリケーションデータが表示されず、VDB による動的検出精度が低下します。
3. **Custom Application Detector / OpenAppID の作成と構成**:
   * システム標準で存在しない特定のポート・パターン（例: TCP ポート 9999 で最初に `SPECIAL_APP_V1` という文字列を送出する社内アプリ）のブロックを指示された場合、**`Objects > Object Management > Application Detectors`** からカスタムディテクターを作成する手順が問われます。
4. **AVC ルール配置の優先順位（First Match Algorithm）**:
   * 「Allow Facebook base browsing, but block Facebook Games」を構成する場合、**上位（Top）ルールに `Block: Facebook Games`** を配置し、**下位ルールに `Allow: Facebook`** を配置しなければなりません。順序を逆にすると、下位の Block ルールに到達する前に全 Facebook 通信が許可されてしまいます。
5. **非標準ポート（Non-Standard Ports）でのアプリケーション識別**:
   * AVC はポート固定型 ACL ではなく、実際のデータペイロードからアプリを検出するため、HTTP が TCP 8080 や 8888 などの非標準ポートで通信していても正確に識別・ブロック可能です。
6. **CLI デバッグツールによる AVC マッチ確認**:
   * FTD CLI で **`system support firewall-engine-debug`** を起動し、クライアント IP をフィルターに指定して通信を発生させます。
   * ログ出力内の `svc: <app_id>, payload: <app_id>, client: <app_id>, match rule order <N>, action <Allow/Block>` を確認して AVC 判定プロセスを追跡・証明します。

---

## 🛠 設定方法

### 1. FMC GUI: AVC 条件を含む Access Control Rule の作成

```text
[ Step 1: ACP への AVC ルールの追加 ]
1. FMC UI から Policies > Access Control を開き対象の ACP を編集。
2. "Add Rule" をクリック:
   - Name: BLOCK_HIGH_RISK_APPS
   - Action: Block with reset
   - Zones タブ: Source = INSIDE_ZONE, Destination = OUTSIDE_ZONE
   - Applications タブを選択:
     - Search by Name: "BitTorrent", "TeamViewer", "Facebook Games" を検索。
     - 検索結果から対象アプリを選択し "Add to Rule" をクリック。
     - (または Filters タブから Risk: Very High / High を一括アサイン)
3. "Save" をクリック。
4. 作成したルールを ACP リストの上位（Top）へ配置し、FTD へ Deploy。
```

---

### 2. FMC GUI: Custom Application Detector (独自アプリ検出) の定義

```text
[ Step 1: カスタムディテクターの追加 ]
1. FMC UI から Objects > Object Management に移動。
2. 左側メニューから "Application Detectors" を選択し "Add Custom Application Detector" をクリック:
   - Name: CUSTOM_PROPRIETARY_APP
   - Description: Custom Internal App on TCP 9999
3. Application Protocol 欄:
   - Category: Business and Economy
   - Risk: Low / Business Relevance: High
4. "Add Pattern" または Detection Criteria を設定:
   - Transport Parameter: TCP / Port: 9999
   - Pattern / String: "MY_COMPANY_SECRET_HEADER"
5. "OK" ➔ "Save" をクリック。

[ Step 2: ACP ルールでのカスタムディテクターの使用 ]
1. Access Control Policy の Applications タブを開く。
2. 作成した "CUSTOM_PROPRIETARY_APP" を検索・指定して Allow または Block に設定し Deploy。
```

---

### 3. FMC GUI: Network Discovery Policy での Application Detection 有効化

```text
1. FMC UI から Policies > Network Discovery に移動。
2. デフォルトルールまたは新規ルールを編集:
   - Subnet: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 (RFC 1918)
   - Action: Discover
   - Track: [✓] Applications にチェックを入れる。
3. "Save" を押下し、対象 FTD へ Deploy。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI / FMC) | 表示確認ポイント |
| :--- | :--- | :--- |
| **Snort 3 でのリアルタイム AVC マッチ追跡** | <code>system support firewall-engine-debug</code> | `svc: <app_name>`, `payload: <app_name>`, `match rule order <N>` のデバッグ出力 |
| **LINA データプレーンでの ACL 展開状態確認** | <code>show access-list</code> | Snort へ転送するための LINA ACL (`CSM_FW_ACL_`) の Hit Count 増加 |
| **VDB (Vulnerability Database) バージョン確認** | <code>show version</code> / FMC Help > About | アプライアンスに適用されている VDB バージョンおよびディテクター数 |
| **LINA パケットトレーサーによる AVC 評価フェーズ確認** | <code>packet-tracer input INSIDE tcp 10.1.1.50 50000 192.0.2.100 80</code> | `Phase: ACCESS-LIST` ➔ `Action: ALLOW/DROP` および Snort 転送判定 |
| **FMC 上でのアプリケーション検出インベントリ確認** | `Analysis > Network Discovery > Applications` | ネットワーク内で検出されたアクティブなアプリケーション一覧テーブル |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **Facebook Games などのマイクロアプリを Block 設定したのに通過してしまう** | HTTPS トラフィックに対して **SSL Inspection Policy (Decrypt) が設定されていない** | <code>system support firewall-engine-debug</code> | SSL Policy で対象通信を `Decrypt - Resign` にアサインし、平文 HTTP URI を Snort に解読させる。 |
| **AVC ルールを配置したが、全トラフィックが後続の Allow ルールで通過してしまう** | AVC ルールの配置順序が下位（Bottom）になっており、上位のポートベース Allow ルールに先にマッチしている | FMC ACP Rule Order Editor | ACP ルールリストで `Block: Application` ルールを、広範な `Allow` ルールよりも上（Top）へ移動させて Deploy。 |
| **カスタムアプリを作成したが FTD 上で一切認識されない** | ポート番号や判定文字列（Pattern）の指定不備、または最小パケットサイズに満たない | FTD CLI: <code>system support firewall-engine-debug</code> | ディテクターの Transport Parameter (TCP/UDP ポート) および文字列のケースセンシティブ（大文字小文字）を修正。 |
| **特定アプリの通信で FTD の CPU 使用率が跳ね上がる** | P2P や大容量ストリーミングの全パケットを過剰に深層検査（DPI）し続けている | <code>show cpu</code><br><code>show access-list</code> | 該当トラフィックに対して Prefilter Policy (`Fastpath`) を構成するか、信頼できるアプリを ACP の上位で `Trust` アサインする。 |
| **VDB アラートが出力され新しいアプリが認識できない** | FMC/FTD 上の VDB (Vulnerability Database) が古く最新アプリディテクターが含まれていない | <code>show version</code> | `System > Updates > Product Updates` から最新の VDB パックをダウンロードし、FMC/FTD へ手動適用する。 |

---

## ⚠ 制限事項

* **復号なしにおける Web アプリケーション識別の限界**:
  * SSL 復号を適用しない場合、FTD は TLS ClientHello の **SNI (Server Name Indication)** や IP レピュテーション情報までしか参照できません。そのため、同一ドメイン配下の詳細サービス（例: `YouTube` と `YouTube Upload`）を識別することは不可能です。
* **セッション初期パケットでのドロップ不可 (Pending State)**:
  * AVC はパケットのペイロードを一定数参照してアプリを確定するため、TCP 3 Way Handshake（1〜3 パケット目）の段階で即座に拒否（Drop）することはできません。アプリが確定した時点（4 パケット目以降）で接続がドロップ/RST されます。
* **カスタムディテクターのネスト制限**:
  * FMC のカスタムアプリケーションディテクターでは、非常に複雑な正規表現（Regex）や複数プロトコルの連動条件を無限にネストさせることはできず、一定のシグネチャ長制限が存在します。

---

## 🔄 他技術との関連

* **SSL Inspection Policy**: HTTPS トラフィック内の暗号化ペイロードを復号し、AVC によるマイクロアプリケーション識別を成立させる必須前処理。
* **QoS Policy**: AVC で特定したアプリケーション（例: `YouTube` や `Netflix`）に対して、帯域制限（Rate Limiting / Police）を動的適用。
* **URL Filtering**: サイトの URL/ドメインカテゴリ制御と、AVC によるアプリケーション機能制御を組み合わせた二重防御アーキテクチャ。
* **Network Discovery Policy**: ネットワーク内を流れるアプリケーションやプロトコルを自動学習し、ネットワークマップおよび Context Explorer へインベントリ展開。

---

## 🧩 比較表

### 1. Application Protocol vs Client Application vs Web Application

| 比較項目 | Application Protocol | Client Application | Web Application |
| :--- | :--- | :--- | :--- |
| **識別対象** | 通信の基盤となる L4〜L7 プロトコル | トラフィックを生成する端末側ソフト | ブラウザ上で動作する特定の Web サービス |
| **具体例** | `HTTP`, `HTTPS`, `SSH`, `DNS`, `FTP` | `Chrome`, `Firefox`, `PuTTY`, `Outlook` | `Facebook`, `YouTube`, `Office365`, `Dropbox` |
| **識別箇所** | L4 ポート ＋ ヘッダー構造 | User-Agent ヘッダー ＋ TLS フィンガープリント | HTTP Host / URI パス / TLS SNI |
| **主な用途** | 標準通信ポート以外の不正通信拒否 | 特定の未許可ブラウザ/クライアント遮断 | シャドー IT 排除・機能別ピンポイント制御 |

### 2. Standard Application Detectors vs Custom Application Detectors (OpenAppID)

| 比較項目 | Standard Application Detectors | Custom Application Detectors (OpenAppID) |
| :--- | :--- | :--- |
| **提供元** | **Cisco System (VDB パックとして提供)** | **管理者によるカスタム定義 (オープンソース OpenAppID)** |
| **更新方法** | FMC / FTD の VDB アップデートにより自動更新 | 管理者が GUI や Lua スクリプトで作成・手動管理 |
| **対象アプリ** | メジャーなグローバル App 1,000 以上 | 社内独自開発 Web アプリ、局所的な P2P ツール等 |
| **メンテナンス** | **Cisco サポートが完全メンテナンス** | 組織内での自前メンテナンスが必要 |

---

## 💡 ベストプラクティス

1. **マイクロアプリケーション制御時の SSL Decryption 併用**:
   * 同一 Web サービス内の特定のサブライン機能（例: `Facebook Games` や `Google Drive Upload`）をブロック対象とする場合は、必ず SSL Inspection Policy で該当通信を **`Decrypt - Resign`** に設定します。
2. **リスク/ビジネス関連度に基づくアグリゲート制御 (Risk/Relevance Filters)**:
   * 個別のアプリ名を何百個も手動指定するのではなく、ACP ルールで **Risk: `Very High`** や **Business Relevance: `Low`** のフィルター条件を指定してルールを作成することで、将来 VDB 更新で追加される新種の高リスクアプリも自動ブロック対象に含める効率的運用を実現します。
3. **AVC ルールを ACL ルールの上位（Top）へ配置**:
   * ポート番号ベースの全許可ルール（例: `Allow Any Port 80/443`）が存在する場合、AVC ルール（例: `Block BitTorrent`）を必ずその上位に配置して、アプリ判定が優先実行されるように設計します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: P2P アプリケーション (BitTorrent / eMule) の完全ブロック

#### 問題
社内 LAN (`10.1.1.0/24`) から外部宛ての全 P2P ファイル共有アプリケーション (`BitTorrent`, `eMule`) 通信を FTD 上で検知し、即座に遮断（Block with reset）せよ。

#### 要件
* ACP ルールの `Applications` タブ条件を使用すること。

#### 設定・操作手順

```text
1. FMC UI で Policies > Access Control を開き対象 ACP を編集。
2. "Add Rule" を押下:
   - Name: BLOCK_P2P_TRAFFIC
   - Action: Block with reset
   - Source Zones: INSIDE_ZONE
   - Applications タブを選択:
     - Search 欄に "BitTorrent", "eMule" を入力して選択。
     - "Add to Rule" をクリック。
3. Save をクリックし、ルールを上位に配置して Deploy。
```

#### 検証方法
* テスト端末から BitTorrent クライアントを起動して接続試行し、通信が Reset されることを確認。FTD CLI で `system support firewall-engine-debug` を実行し、`svc: BitTorrent, action Block` が出力されることを検証。

---

### 例2: Facebook 全体の閲覧は許可しつつ Facebook Games のみを遮断

#### 問題
社員の `Facebook` サイトの閲覧は許可しつつ、Facebook 内のゲーム機能 (`Facebook Games`) のみをピンポイントでブロックせよ。

#### 要件
* HTTPS 通信を復号する SSL Inspection Policy が有効であること。
* ルールの優先順位を考慮して構成すること。

#### 設定・操作手順

```text
1. Policies > Access Control > SSL で Outbound 通信に対する Decrypt - Resign ルールが有効であることを確認。
2. 対象 ACP を編集し、ルールを 2 つ追加:
   - Rule 1 (Top):
     - Name: BLOCK_FB_GAMES
     - Action: Block
     - Applications: "Facebook Games"
   - Rule 2 (Below Rule 1):
     - Name: ALLOW_FB_BASE
     - Action: Allow
     - Applications: "Facebook"
3. Save をクリックして Deploy。
```

#### 検証方法
* クライアントから `[suspicious link removed]` にアクセスし正常閲覧できることを確認。その後 `[suspicious link removed]` にアクセスし、通信が遮断されることを確認。

---

### 例3: 高リスク (Risk: Very High) アプリケーションの一括ブロック

#### 問題
Cisco VDB によって Risk レベルが `Very High` と定義されているすべてのアプリケーションを自動識別し、一括でブロックせよ。

#### 要件
* Applications タブの Filter 機能を使用すること。

#### 設定・操作手順

```text
1. ACP 編集画面で "Add Rule" を選択:
   - Name: BLOCK_VERY_HIGH_RISK_APPS
   - Action: Block
   - Applications タブを選択:
     - "Filter BY" セクションを開き "Risk" ➔ "Very High" を選択。
     - 表示された全アプリを一括選択して "Add to Rule" を押下。
2. Save して Deploy。
```

#### 検証方法
* 非常に対象リスクが高いプロキシツールや未承認 VPN ツールからの通信を発生させ、本ルールにマッチして拒否されることを確認。

---

### 例4: 非標準ポート (TCP 8080) で動作する HTTP 通信の AVC 識別と制御

#### 問題
非標準ポート `TCP 8080` を使用して外部へアクセスする Web ブラウザ通信を AVC で識別し、`Allow` 処理せよ。

#### 要件
* L4 Port 条件（8080）ではなく Applications 条件（HTTP/Web-browsing）を使用すること。

#### 設定・操作手順

```text
1. ACP 編集画面で "Add Rule" を選択:
   - Name: ALLOW_HTTP_NON_STD_PORT
   - Action: Allow
   - Applications タブ: "HTTP" を追加。
   - Ports タブ: 条件なし (Any) のまま保持。
2. Save して Deploy。
```

#### 検証方法
* クライアントから `[suspicious link removed]` へアクセスし、L4 ポート 8080 であっても AVC エンジンが `HTTP` プロトコルとして正常認識し、ルールにヒットして疎通することを確認。

---

### 例5: 特定クライアントアプリケーション (PuTTY / SSH Client) のブロック

#### 問題
社内 LAN から外部管理サーバー宛ての SSH 通信において、`PuTTY` クライアントソフトウェアを使用した接続のみをブロックし、他の標準 Terminal アプリは許可せよ。

#### 要件
* Client Application 条件を使用すること。

#### 設定・操作手順

```text
1. ACP ルールを作成:
   - Name: BLOCK_PUTTY_CLIENT
   - Action: Block
   - Applications タブ: "PuTTY" (Client Application) を選択して追加。
2. Save して Deploy。
```

#### 検証方法
* PuTTY 端末から SSH 接続を試行してブロックされること、Linux 標準ターミナルからの SSH 接続は通過することを検証。

---

### 例6: カスタムアプリケーションディテクター `APP_INTERNAL_CRM` の定義とブロック

#### 問題
社内独自アプリ (`TCP 9999` / ペイロード先頭文字列 `CRM_SECRET_CONNECT`) を識別するカスタムディテクター `APP_INTERNAL_CRM` を作成し、外部宛て通信をブロックせよ。

#### 要件
* Objects > Application Detectors から作成すること。

#### 設定・操作手順

```text
1. Objects > Object Management > Application Detectors を開く。
2. "Add Custom Application Detector" を押下:
   - Name: APP_INTERNAL_CRM
   - Category: Business and Economy
3. Detection Criteria タブ:
   - Transport Parameter: TCP / Port: 9999
   - Pattern: "CRM_SECRET_CONNECT"
4. Save を押下。
5. ACP にルールを追加し、Applications 条件で "APP_INTERNAL_CRM" を指定して Block アサインの上 Deploy。
```

#### 検証方法
* 疑似クライアントから TCP 9999 番ポートで `CRM_SECRET_CONNECT` 文字列を含むパケットを送信し、FTD 上でカスタムディテクターとして識別・遮断されることを確認。

---

### 例7: Remote Desktop / TeamViewer の動的ブロックルール

#### 問題
リモートデスクトップツールである `TeamViewer` および `AnyDesk` 通信を FTD 上で識別し、ポート番号に関わらず自動ブロックせよ。

#### 要件
* Applications 条件で指定すること。

#### 設定・操作手順

```text
1. ACP ルールを作成:
   - Name: BLOCK_REMOTE_DESKTOP_TOOLS
   - Action: Block with reset
   - Applications タブ: "TeamViewer", "AnyDesk" を指定。
2. Save して Deploy。
```

#### 検証方法
* クライアント上で TeamViewer を起動し、サーバーへのハンドシェイク段階で FTD によって通信がリセットされセッションが確立しないことを確認。

---

### 例8: QoS Policy と AVC の連携による YouTube 帯域制限

#### 問題
社内端末からの `YouTube` 動画視聴を完全にブロックするのではなく、QoS Policy と連携して最大帯域幅を `1 Mbps` に制限（Policing）せよ。

#### 要件
* QoS Policy の Applications 条件を使用すること。

#### 設定・操作手順

```text
1. Policies > Access Control > QoS を開き "Add QoS Rule" を選択。
2. Rule 編集画面:
   - Name: LIMIT_YOUTUBE_BANDWIDTH
   - Applications タブ: "YouTube" を指定。
   - Rate Limit: Download = 1024 Kbps (1 Mbps)
3. Save をクリックし ACP 経由で Deploy。
```

#### 検証方法
* クライアントから YouTube で高画質 4K 動画を再生し、ダウンロード速度が 1 Mbps 付近で頭打ちになることを確認。

---

### 例9: FTD CLI デバッグコマンドによる AVC アプリケーション識別のリアルタイム追跡

#### 問題
FTD CLI のデバッグツールを使用して、特定 IP (`10.1.1.100`) から発生した Web アクセスに対する AVC アプリケーション識別処理をリアルタイム追跡せよ。

#### 要件
* `system support firewall-engine-debug` コマンドを使用すること。

#### 設定・操作手順

```bash
> system support firewall-engine-debug

Please specify an IP protocol: tcp
Please specify a client IP address: 10.1.1.100
Please specify a server IP address:
Please specify a server port: 80
```

#### 検証方法
* クライアントからアクセス発生時、CLI に `svc: 676 (HTTP), payload: 629 (Facebook), match rule order 1, action Allow/Block` などのリアルタイムログが出力されることを確認。

---

### 例10: FMC REST API による Application Detector リストの GET 取得

#### 問題
FMC REST API を使用して、システムおよびカスタムで登録されている全 Application Detector オブジェクトの UUID 一覧を Curl コマンドで取得せよ。

#### 要件
* `/api/fmc_config/v1/domain/{domainUUID}/object/applicationdetectors` エンドポイントを使用すること。

#### 設定・操作手順

```bash
# ===== Auth Token 取得後、Application Detectors を GET =====
curl -k -X GET "https://fmc.corp.local/api/fmc_config/v1/domain/e276abec-e0f2-11e3-8169-6d9ed13b206f/object/applicationdetectors" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json"
```

#### 検証方法
* 返却された JSON レスポンス内に、アプリケーションディテクターの名称および UUID リストが含まれていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で Access Control Policy ルールを作成し、`Applications` 条件に `Facebook Games` を指定して `Block` アクションを設定した。しかし、クライアントから HTTPS (`[suspicious link removed]`) でアクセスしたところ、ゲーム画面が正常に開いてしまいブロックされなかった。原因として最も可能性が高いものはどれか。

* A) FTD アプライアンスのライセンスが Base ライセンスにダウングレードしているため。
* B) 対象の HTTPS トラフィックを復号する SSL Inspection Policy (Decrypt - Resign) が設定・バインドされておらず、FTD が暗号化ペイロード内の URI パスを識別できないため。
* C) FTD が Transparent Firewall Mode で動作しているため。
* D) FMC 上で Threat License が無効化されているため。

**正解**: **B**  
**解説**: 暗号化された HTTPS 通信（TCP 443）において、SSL 復号が適用されていない場合、FTD は TLS ClientHello の SNI (`facebook.com`) までしか認識できません。ベースの Facebook サイトと内部のサブライン機能である `Facebook Games`（`/games`）を精細に識別して遮断するには、**SSL Inspection Policy による復号（Decrypt）が必須要件** となります。

---

### 2. トラブルシュート問題

**問題**: 社内クライアントからのインターネット接続制御において、上位ルールに `Allow Any HTTP/HTTPS Traffic` を配置し、下位ルールに `Block BitTorrent Application` を配置した。クライアントから BitTorrent による通信を発生させたところ、下位の Block ルールにヒットせず通信が許可されてしまった。正しい修正手順はどれか。

* A) FTD CLI で `configure application-bypass disable` を実行する。
* B) Access Control Policy のルール順序を変更し、`Block BitTorrent Application` ルールを `Allow Any HTTP/HTTPS Traffic` ルールよりも上位（Top）へ移動させる。
* C) FTD の Firewall Mode を Routed Mode から Transparent Mode へ変更する。
* D) Network Discovery Policy で IP アドレス指定を解除する。

**正解**: **B**  
**解説**: FTD の Access Control Policy は **First Match Algorithm（最初に合致したルールを適用）** で評価されます。広範な `Allow` ルールが上位にある場合、通信はアプリ判定が完全に確定する前、またはポート判定の段階で上位ルールにマッチして許可されます。特定のアプリケーションを遮断したい場合は、該当の **Block ルールを Allow ルールよりも上位へ配置** する必要があります。

---

### 3. Design 問題

**問題**: 企業ネットワークにおいて、新しく登場する未知のハイリスクアプリケーション（未承認 P2P や匿名プロキシ等）を管理者が手動で個別にルール追加することなく、将来にわたって動的に自動ブロックする最適な Access Control Policy ルール設計手法はどれか。

* A) 毎週末に FTD のアクセスリストを CLI から更新する。
* B) ACP ルールの Applications 条件において、個別のアプリ名ではなく `Filter BY` 機能で **`Risk: Very High`** の属性フィルター条件を指定した Block ルールを作成する。
* C) 全トラフィックに対して Prefilter Fastpath を設定する。
* D) SSL Policy の Default Action を Block に変更する。

**正解**: **B**  
**解説**: AVC の `Applications` タブにおいて **`Risk` や `Business Relevance` の属性フィルター** を条件として指定しておくことで、将来 Cisco VDB パックのアップデートによって新しく追加された `Very High Risk` アプリケーションも自動的に本ルールのブロック対象として包括・制御されます。

---

### 4. 実装問題

**問題**: FTD CLI にログインし、現在アプライアンス上を通るリアルタイムトラフィックに対して、Snort エンジンがどのようなアプリケーション名（`svc` や `payload`）として識別しているかを直接デバッグ確認するための正確な CLI コマンドはどれか。

* A) <code>show access-list</code>
* B) <code>system support firewall-engine-debug</code>
* C) <code>show running-config application</code>
* D) <code>show service-policy</code>

**正解**: **B**  
**解説**: FTD CLI で **`system support firewall-engine-debug`** を実行すると、指定した IP/ポートのリアルタイム通信に対して、Snort エンジンが判定したアプリケーション名（例: `svc: HTTP, payload: Facebook`）やマッチした ACP ルール番号を詳細ログとして出力・検証できます。

---

### 5. コンフィグ読解問題

**問題**: 管理者が `Objects > Object Management > Application Detectors` から、独自の社内 Web アプリ（TCP 8080 / ペイロード文字列 `INTERNAL_APP_HEADER`）を識別する Custom Application Detector を作成し Deploy した。コネクション発生時、FTD がこのカスタムアプリを確定・ルールアサインできる最小のタイミングとして正しいものはどれか。

* A) TCP 3 Way Handshake の最初の SYN パケットを受信した時点。
* B) 初期ハンドシェイク完了後、クライアントから `INTERNAL_APP_HEADER` 文字列を含むデータペイロードパケットを受信し、Snort がパターンマッチした時点。
* C) トラフィックのセッションが終了して FIN パケットが送出された時点。
* D) FTD アプライアンスを再起動した時点。

**正解**: **B**  
**解説**: AVC はディープパケットインスペクション（DPI）機能であるため、L4 の TCP SYN パケットのみ（Pending 状態）では判定できません。初期ハンドシェイク完了後、定義された文字列（`INTERNAL_APP_HEADER`）を含むデータペイロードを受信して初めてアプリケーション名が確定し、ACP ルールが執行されます。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Device Configuration Guide - Application Detection](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot Application Visibility and Control (AVC) on FTD](https://www.cisco.com/)
* [Cisco Technical Notes: Create and Deploy Custom Application Detectors and OpenAppID on FMC](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Application Visibility and Control with Cisco Secure Firewall (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で「Block Facebook Games, allow Facebook base, and rate limit YouTube to 1Mbps」と要求された場合の全体構成手順：
    1. SSL Policy で Outbound 通信に対する `Decrypt - Resign` を構成。
    2. QoS Policy で `YouTube` に対する Rate Limit (1Mbps) ルールを作成。
    3. ACP を編集：
       - Rule 1 (Top): Action = `Block`, Applications = `Facebook Games`
       - Rule 2 (Below): Action = `Allow`, Applications = `Facebook`
    4. ACP の上部で SSL Policy および QoS Policy をアサインし、FTD へ Deploy。
* **図解（AVC アプリケーション判定フロー）**:
  ```text
  [ Ingress Packet Received ]
               │
               ▼
  [ Snort DPI Engine Processing ]
               │
               ├── Packet 1-3 (TCP Handshake) ──► Application State: "Pending"
               │                                       │
               │                                       ▼
               │                          [ Pass Initial Packets ]
               │
               └── Packet 4+ (Payload / HTTP Request / TLS SNI)
                       │
                       ▼
            [ Match App Detector Signature / VDB ]
                       │
                       ▼
            [ Application Identified: "Facebook Games" ]
                       │
                       ▼
            [ Evaluate Access Control Policy (Top to Bottom) ]
                       │
                       ├── Match Rule "Block Facebook Games" ──► [ Drop / Send TCP Reset ]
                       └── Match Rule "Allow Facebook Base"  ──► [ Forward Packet ]
  ```
* **注意点**:
  * ラボ試験において AVC ルール作成・変更を行った後は、必ず FTD CLI で `system support firewall-engine-debug` を実行するか、FMC の `Analysis > Connections > Events`（または `Network Discovery > Applications`）を開き、対象通信の `Application Protocol` および `Client/Web Application` カラムに意図通りのアプリケーション名が記録されていることを客観的にテスト・証明してください。

---

ご不明な点や追加の解説が必要な場合は、お気軽にお知らせください。復習や実機演習の際にお役立ていただければ幸いです！
