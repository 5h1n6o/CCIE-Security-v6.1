---
layout: default
title: 4.1-ISE-scalability
nav_order: 1
parent: 4.0-Identity-Management
---

# 4.1 Cisco ISE scalability using multiple nodes and personas

---

## 📘 概要

Cisco Identity Services Engine (Cisco ISE) は、大組織から小規模拠点まで柔軟に対応できるモジュール型の分散アーキテクチャを備えたエンタープライズ AAA・アイデンティティ管理プラットフォームです。単一ノード（Standalone）から最大 50 台の Policy Service Node (PSN) を備えた大規模分散クラスター（ISE Cube）まで拡張が可能です。

ISE の拡張性と高可用性（High Availability: HA）は、ノードに割り当てる **Persona（ペルソナ）** と **Service（サービス）** の分離・分散配置によって実現されます。各ノードは特定のペルソナ（PAN, MnT, PSN, pxGrid）を担当し、トラフィック負荷、データベースレプリケーション、ログ集約、外部セキュリティ製品との連携を分担します。

CCIE Security v6.1 ラボ試験では、各ペルソナの役割・制限・相互依存関係を正確に理解し、マルチノード環境におけるノード登録、ペルソナ変更、自動 PAN フェールオーバー設定、PSN Node Group によるセッション保護、ならびにネットワークロードバランサ（NLB）や Anycast を用いた HA 設計を正しく構築・検証・トラブルシューティングする能力が要求されます。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **主要ペルソナ** | Primary/Secondary Administration Node (PAN), Primary/Secondary Monitoring & Troubleshooting Node (MnT), Policy Services Node (PSN), pxGrid Controller |
| **展開モデル** | Standalone (単一ノード), Two-Node (2ノードHA/ミラー), Hybrid (PAN+MnT同居 + 独立PSN), Dedicated/Fully Distributed (全ペルソナ独立ノード) |
| **最大規模 (ISE 3.1/3.x)** | Fully Distributed 展開で最大 2,000,000 同時接続セッション、最大 50 台の Dedicated PSN、最大 4 台の Dedicated pxGrid ノード、最大 100,000 NAD |
| **データ同期機構** | 集中設定DB: PANから全ノードへTCP/443 (HTTPS/REST)・JGroupsによる自動リアルタイムレプリケーション<br>セッション同期: PSN間は TCP/7800 (Peer) および TCP/7802 (Heartbeat) |
| **高可用性 (HA)** | **PAN**: Active/Warm-Spare (手動昇格またはAuto-PAN Failover機能)<br>**MnT**: Active/Active ログ重複送信 (UDP/20514またはTCP/20514)<br>**PSN**: Active/Active (NAD側ロードバランシング / NLB / Node Group CoAリダイレクト) |
| **レイテンシ要件** | PANと他のすべてのノード（Secondary PAN, MnT, PSN）間の最大許容 RTT 遅延: **300 ms** 以下 |
| **主な制限事項** | 1つの ISE Cube 内で PAN は最大 2 台、MnT は最大 2 台。TC-NAC サービスは Cube 全体で最大 1 ノードのみアクティブ可能。Node Group 内推奨 PSN 数は 4 台以下 |
| **設計上の注意点** | ノード登録時には証明書信頼関係（Root CA / Admin Certificate）の事前確立が必須。時系列ログ整合性のため全ノードで同一 NTP サーバー同期が必須 |

---

## 🏗 動作原理

Cisco ISE クラスターは、役割に応じたペルソナを分散配置することで処理負荷と耐障害性を最適化します。

```
                       +-----------------------------------+
                       |   Primary PAN  /  Secondary PAN   |
                       |  (Administration & Policy Config) |
                       +-----------------+-----------------+
                                         |
            +----------------------------+----------------------------+
            | DB Replication (TCP/443)   | DB Replication (TCP/443)   |
            v                            v                            v
+-----------------------+    +-----------------------+    +-----------------------+
|  Primary MnT / S-MnT  |    |     PSN 1 (Node Group)|    |     PSN 2 (Node Group)|
|  (Central Log / Reporting) | (RADIUS/TACACS+/Portal)|    | (RADIUS/TACACS+/Portal)|
+-----------^-----------+    +-----------^-----------+    +-----------^-----------+
            |                            |                            |
            | Syslog (UDP/TCP 20514)     | RADIUS Auth / Acct         | RADIUS Auth / Acct
            +----------------------------+----------------------------+
                                         |
                               +---------+---------+
                               | Network Access    |
                               | Devices (NADs)    |
                               +-------------------+
```

### 1. ペルソナ（Persona）の定義と機能

1. **Policy Administration Node (PAN)**:
   - クラスター全体の「マスターデータベース（Single Source of Truth）」および GUI 管理コンソールを提供。
   - すべてのポリシー変更、ネットワークデバイス登録、ユーザー定義、証明書発行（Internal CA Root）を集中管理。
   - Primary PAN がダウンした場合でも既存の PSN による認証処理は継続するが、ポリシーの変更や新規ノードの追加・レプリケーションは停止する（Warm-Spare 構成）。
2. **Monitoring & Troubleshooting Node (MnT)**:
   - 全ノード（PAN, PSN）から送出される Syslog (UDP/TCP 20514) を集約・相関分析し、リアルタイム Live Logs、レポート、アラートを生成。
   - 2 台の MnT を配置することで Active/Active 冗長化（全ノードが両 MnT にログを二重送信）を実現。
3. **Policy Services Node (PSN)**:
   - 実際の通信処理を担当する「Workhorse（労働馬）」。RADIUS/TACACS+ 認証・認可、ゲストポータル、ポスチャ評価、プロファイリング、SXP、TC-NAC 等を実行。
   - 変更された設定データは PAN から即座に同期（Replication）され、ローカルメモリのデータベースで処理するため、PAN が停止しても独立して認証を維持。
4. **pxGrid (Platform Exchange Grid) Controller**:
   - 異種セキュリティ製品（Cisco Firepower/FMC, Stealthwatch/Secure Network Analytics, ISE-PIC, サードパーティ SIEM）間でアイデンティティコンテキスト（IP-SGT マッピング、セッション情報、脅威ステータス）を Publish/Subscribe モデルで相互共有するための高速バス。
   - pxGrid 2.0 では WebSockets / REST API (TCP/8910) を使用。

### 2. ノードグループ（Node Groups）の動作メカニズム

- 同一 LAN または近接した L3 ネットワーク上に配置された複数の PSN を **Node Group** にグループ化します。
- PSN 相互間で TCP/7800（ピア通信）および TCP/7802（ヘルスチェック検出）を使用してピア状態を監視します。
- ゲストポータル、ポスチャ、MDM リダイレクト等の Web リダイレクトセッション中に特定の PSN が障害停止した場合、Node Group 内の別の健全な PSN がそれを検出し、該当 NAD に対して **Change of Authorization (CoA)** を送出してセッションを再初期化・代替 PSN へ誘導します。
- また、Node Group 内では JGroups フレームワークによるプロファイリング情報などのローカルレプリケーションが最適化されます。

---

## ⚙ 動作シーケンス

### A. ノード登録およびデータレプリケーション・シーケンス

```
  Secondary Node                  Primary PAN
       |                              |
       |--- 1. Register Node (GUI) -->|
       |    (HTTPS Admin Auth)        |
       |                              |
       |<-- 2. Establish Trust -------|
       |    (Certificate Exchange)    |
       |                              |
       |<-- 3. Full DB Sync ----------|
       |    (Schema & Initial Data)   |
       |                              |
       |==  4. Node Restart =========|  (App Server restarts on Secondary)
       |                              |
       |<== 5. Incremental Sync ======|  (Triggered on any Policy/Config Change)
```

1. **ノード登録の開始**: 管理者が Primary PAN GUI からセカンダリノードの FQDN/IP アドレスおよび管理者資格情報を入力して登録を実行。
2. **証明書トラスト確立**: 両ノード間で Admin 証明書および Root CA 証明書が交換・検証され、相互 TLS 暗号化チャネルが形成される。
3. **フルレプリケーション（Full Sync）**: Primary PAN からセカンダリノードへデータベース構造（スキーマ）および全データが転送される。
4. **ノード再起動**: セカンダリノード上でアプリケーションサービスが自動再起動し、指定されたペルソナ（PSN または Secondary PAN/MnT）として立ち上がる。
5. **増分レプリケーション（Incremental Sync）**: Primary PAN で設定変更（ポリシー追加、NAD追加等）が行われるたびに、変更差分がリアルタイムで全セカンダリノードへ自動プッシュされる。

### B. Auto-PAN Failover 動作シーケンス

1. **ヘルスチェック監視**: 構成された Health Check Node（例: 専用 PSN ノード）が Primary PAN に対して一定間隔（例: 120 秒）でポーリングを実施。
2. **障害検出**: Primary PAN からの応答が連続指定回数（例: 5 回連続失敗）絶たれた場合、Health Check Node は Primary PAN のダウンと判定。
3. **自動昇格（Promotion）**: Secondary PAN に対して昇格命令を発行し、Secondary PAN が自動的に Primary PAN へロール変更されて再起動。
4. **クラスター復旧**: 新Primary PAN がアクティブとなり、各 PSN とのデータベース同期関係を再確立。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、Cisco ISE のマルチノード展開およびペルソナ設計に関する出題ポイントとトラブルシューティングの要点を以下に整理します。

### 1. Blueprintで重要なポイント

- **ノード登録の前提要件**:
  - 全ノードの FQDN が DNS で正しく正引き・逆引き（Forward / Reverse DNS lookup）できること。
  - 全ノードの時刻が同一の NTP サーバーに同期しており、タイムゾーン・時刻のずれが数秒以内であること。
  - 登録前のセカンダリノードは「Standalone」状態であること。
- **Auto-PAN Failover の設定要件**:
  - Auto-PAN Failover を有効化するには、Primary PAN、Secondary PAN に加え、**第3のノード（Health Check Node として機能する PSN など）** が最低 1 台必要。
  - PAN 同士が相互監視するのではなく、独立した Health Check Node が判定を行う仕組みを理解しておくこと。
- **PSN Node Group の制約と機能**:
  - Node Group は Web リダイレクトセッション（Guest/Posture/MDM）のフェールオーバー時に CoA リダイレクトを発行する。RADIUS 認証自体の負荷分散は NAD 側の設定（Priority/Load Balancing）または NLB に依存する。
  - TCP/7800 (Peer) および TCP/7802 (Failure Detection) がノード間で通信可能であること。

### 2. ラボ試験で設定させられそうな内容

1. **Standalone ノードのマルチノード化**:
   - 初期状態の ISE-1 を Primary PAN/MnT に指定し、ISE-2 を Secondary PAN/MnT、ISE-3 / ISE-4 を Dedicated PSN として登録する設定。
2. **Auto-PAN Failover の構成**:
   - Polling Interval (30-300秒) および Failure Poll Count (2-60回) を要件通りに設定し、Health Check Node を指定。
3. **PSN Node Group の作成とメンバー割り当て**:
   - 特定のロケーション（例: Datacenter-1）に所属する PSN 群を同一 Node Group に追加。
4. **pxGrid Controller の有効化と証明書設定**:
   - PSN 上で pxGrid ペルソナを有効化し、FMC や Stealthwatch と連携させるための Auto-Registration または Client Approval 構成。

### 3. よくある設定ミス

- **DNS / Hostname の不一致**:
  - ノード登録時に IP アドレスではなく FQDN で登録しようとした際、DNS 逆引きが設定されていないと登録処理（Sync）が 80% や 99% で永久にハングする。
- **証明書トラスト（Root CA）の欠如**:
  - 各ノードが独自の Self-Signed Certificate を使用している場合、登録時に警告を無視するか、事前に PAN の Trusted Certificates にセカンダリノードの証明書/CA をインポートしておかないと TLS ハンドシェイクエラーで失敗する。
- **時刻非同期 (NTP Offset)**:
  - ノード間の時刻がずれていると、データベースレプリケーション（JGroups / DB Sync）が失敗し、「Out of Sync」状態に陥る。

### 4. コンフィグ読解・showコマンド判定

- CLI コマンド `show application status ise` で各プロセスの動作状態を確認：
  - `Database Listener` および `Database Server` が running であること。
  - `Application Server` が running であること（ノード登録直後は restarting や initializing になる）。
- `show logging application replication/config-replica.log` でレプリケーションエラーのログを確認。

---

## 🛠 設定方法

### 1. GUI によるノード登録とペルソナ変更手順

#### Step 1: Primary PAN の有効化準備
1. Primary PAN とする ISE ノードの Admin GUI にログイン。
2. **Administration > System > Deployment** に移動。
3. 自ノード（例: `ise-pan1`）を選択し、**Register** ボタンが使用可能（自ノードが Primary 権限を持っている状態）であることを確認。

#### Step 2: セカンダリノードの登録（Secondary PAN / Dedicated PSN）
1. **Administration > System > Deployment** にて **Register** ボタンをクリック。
2. **Register a Node** 画面で以下を入力：
   - **FQDN**: `ise-psn1.lab.local`（登録対象のセカンダリノード）
   - **Username**: `admin`
   - **Password**: `Cisco12345!`
3. **Next** をクリック。セカンダリノードの Self-Signed 証明書の警告が表示された場合は、証明書フィンガープリントを確認のうえ **Import Certificate and Proceed** を選択。

#### Step 3: ノードペルソナとサービスの設定
登録画面で該当ノードの役割を指定します：
- **Secondary PAN / Secondary MnT の場合**:
  - **Role**: `SECONDARY` を選択。
  - **Administration**: チェックを入れる。
  - **Monitoring**: チェックを入れ、Role を `SECONDARY` に設定。
- **Dedicated PSN の場合**:
  - **Administration**: チェックを外す。
  - **Monitoring**: チェックを外す。
  - **Policy Service**: チェックを入れる。
    - `Enable Session Services`（RADIUS/TACACS+ 認証）
    - `Enable Profiling Service`（プロファイリング機能）
    - `Enable Passive Identity Service`（必要に応じて）
    - `Include Node in Node Group`: 作成済みの Node Group 名を選択（任意）
4. **Save** をクリック。
5. セカンダリノード上でデータレプリケーションが開始され、アプリケーションサーバーが自動再起動します（所要時間: 10〜15分）。

---

### 2. Auto-PAN Failover の設定手順

1. **Administration > System > Deployment** に移動。
2. 左パネルのメニューから **PAN Failover Configuration** を選択。
3. **Enable PAN Auto Failover** チェックボックスをオンにする。
4. 以下の項目を設定：
   - **Primary Health Check Node**: `ise-psn1.lab.local`（第3のノードを選択）
   - **Secondary Health Check Node**: `ise-psn2.lab.local`（予備のヘルスチェックノードを選択）
   - **Polling Interval**: `120` 秒（範囲: 30〜300秒）
   - **Number of Failure Polls Before Failover**: `5` 回（範囲: 2〜60回）
5. **Save** をクリック。

---

### 3. PSN Node Group の作成と割り当て

1. **Administration > System > Deployment** に移動。
2. 左パネルの **Deployment** ルートツリーアイコン横のギアマークをクリックし、**Create Node Group** を選択。
3. **Node Group Name**: `HQ-PSN-Group` を入力し **Save**。
4. 割り当てたい PSN ノード（例: `ise-psn1`）の編集画面を開き、`Policy Service` セクションの `Include Node in Node Group` ドロップダウンから `HQ-PSN-Group` を選択して保存。

---

### 4. CLI による確認・復旧手順

#### CLI でのノード状態確認
```bash
# ISE アプリケーションプロセスの動作状況確認
show application status ise

# クラスター展開のノード設定確認
show running-config
```

#### CLI による手動 Primary 昇格（Secondary PAN 上で実行）
Primary PAN が完全に壊れた場合、GUI または CLI から Secondary PAN を Primary に手動昇格させます：

```bash
# Secondary PAN の CLI にログインし、Primary への昇格コマンドを実行
application configure ise
# 対話メニューから Promotion オプションを選択
# または GUI: Administration > System > Deployment > [自ノード選択] > Promote to Primary
```


## 🔍 検証コマンド

Cisco ISE ノードの状態確認、ペルソナステータス、データ同期、およびトラブルシューティングで使用する主要な検証コマンドを以下に示します。

| 目的 | コマンド | 説明・確認ポイント |
| --- | --- | --- |
| ノードプロセス状態確認 | <code>show application status ise</code> | Application Server, Database Server, Database Listener, ISE Messaging Service 等のプロセス状態（running）を確認 |
| システムリソース確認 | <code>show tech-support</code> | CPU、メモリ、ディスク使用量、NIC ステータス等の詳細システムログを出力 |
| ノードのバージョン・パッチ確認 | <code>show version</code> | インストールされている ISE バージョンおよび適用済み Hotfix / Patch レベルを確認 |
| ネットワーク接続確認 | <code>show ports status</code> | リッスンしているポート番号（TCP 443, 8443, 7800, 7802, 8910 等）の開口状況を確認 |
| NTP 動期状態確認 | <code>show ntp</code> | NTP サーバーとの同期状態および Offset/Jitter 値を確認 |
| DNS 解決確認 | <code>nslookup &lt;FQDN&gt;</code> | 他ノードの FQDN に対する正引き・逆引き（PTR レコード）解決を確認 |
| データベース同期ログ確認 | <code>show logging application config-replica.log</code> | PAN からの DB レプリケーション処理、データ同期の完了ステータス、エラーメッセージを確認 |
| ADE-OS システムログ確認 | <code>show logging application ade/ade.log</code> | アプリケーション開発・展開エンジン (ADE-OS) の管理ログおよびノード登録エラーを確認 |
| リアルタイムログ監視 | <code>tail-log ade/ade.log</code> | ノード登録・同期実行中のログをリアルタイムで追跡 |

---

## 🚨 トラブルシュート

ISE マルチノード構成で発生しやすい典型的な症状、根本原因、確認コマンド、および対処方法を一覧表で整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| ノード登録が 80% または 99% で停止（Progress Hang） | DNS 逆引き (PTR) 失敗、またはノード間の NTP 時刻非同期 | <code>nslookup &lt;IP&gt;</code><br><code>show ntp</code> | DNS サーバーに PTR レコードを追加。全ノードで同一 NTP サーバーを設定し <code>ntp server &lt;IP&gt;</code> で同期させる。 |
| ノード登録時に TLS/Certificate Error で失敗 | 相手ノードの Self-Signed 証明書または Root CA を信頼していない | <code>show logging application ade/ade.log</code> | Primary PAN の Trusted Certificates に登録対象ノードの Admin/Root CA 証明書をあらかじめインポートする。 |
| Secondary PAN への自動フェールオーバーが発動しない | Health Check Node が設定されていない、またはヘルスチェック間隔に達していない | GUI: Administration > System > Deployment > PAN Failover | PAN Failover 構成画面で PSN を Health Check Node に指定し、Enable PAN Auto Failover にチェックを入れて保存する。 |
| セカンダリノードが "Out of Sync" ステータスになる | WAN 回線の遅延超過（>300ms）またはパケットドロップ、DB レプリケーションチャネル切断 | GUI: System Deployment<br><code>show logging application config-replica.log</code> | GUI の Deployment 画面で該当ノードを選択し **Syncup** ボタンを実行してフル同期を再試行。ネットワーク遅延（<300ms RTT）を確認。 |
| PSN 障害時に Guest ポータルの URL リダイレクトが引き継がれない | PSN が Node Group に未所属、または TCP/7800-7802 がファイアウォールでブロック | <code>show ports status</code><br>GUI: Node Group 画面 | 該当 PSN を同一 Node Group に割り当てる。中間ファイアウォールで TCP/7800 および TCP/7802 を許可する。 |
| Active-Active MnT 構成で片方の MnT にログが届かない | PSN から MnT への Syslog トラフィック（UDP/TCP 20514）が遮断されている | GUI: Operations > Troubleshoot > Diagnostic Tools | ネットワーク途中の ACL / FW で UDP 20514 および TCP 20514 を許可する。 |

---

## ⚠ 制限事項

Cisco ISE マルチノード展開における機能的・パフォーマンス上の主な制限事項です。

- **最大ノード数と接続スケール (ISE 3.1/3.x)**:
  - クラスター全体の最大同時接続セッション数: **2,000,000** エンドポイント（Dedicated PAN/MnT、SNS-3695 / 大容量 VM 使用時）。
  - クラスター全体でサポートされる最大 Dedicated PSN 数: **50 台**。
  - クラスター全体でサポートされる最大 Dedicated pxGrid ノード数: **4 台**（すべて Active）。
  - クラスター全体でサポートされる最大 NAD 数: **100,000 台**。
- **ペルソナ制限**:
  - 1 つの ISE クラスター（Cube）内に配置可能な Administration Node (PAN) は最大 **2 台**（Primary 1 台、Secondary 1 台）。
  - 1 つの ISE クラスター内に配置可能な Monitoring Node (MnT) は最大 **2 台**（Primary 1 台、Secondary 1 台）。
  - **TC-NAC (Threat-Centric NAC)** サービスは、クラスター全体で **1 台のノードのみ** 有効化可能。
- **ハイブリッド展開の制限**:
  - PAN と MnT を同居（PAN+MnT）させたハイブリッド構成の場合、サポートされる Dedicated PSN の上限は最大 **5 台**、最大セッション数は **50,000** エンドポイントに制限される。
- **遅延（Latency）要件**:
  - PAN とすべてのセカンダリノード（Secondary PAN, MnT, PSN）間の最大許容 Round Trip Time (RTT) 遅延は **300 ms**。これを超えるとレプリケーションタイムアウトおよびデータベース非同期が発生する。


## 🔄 他技術との関連

Cisco ISE のマルチノード・ペルソナアーキテクチャは、インフラの他の主要技術と以下のように密接に関連しています。

- **Cisco TrustSec (SGT / SXP)**:
  - PAN 上で定義された Security Group Tag (SGT) および Egress SGACL ポリシーは、データレプリケーションによって全 PSN に自動同期されます。また、特定 PSN で SXP サービスを有効化することで、スイッチやルータと IP-SGT マッピング情報をピアリング伝送します。
- **Network Access Control (802.1X / MAB / WebAuth)**:
  - スイッチや WLC（NAD）は RADIUS サーバーグループとして複数の ISE PSN を登録し、ラウンドロビンまたは優先順位指定で負荷分散・障害迂回を行います。
- **Network Load Balancers (NLB) & Anycast**:
  - 多数の PSN を束ねる際、F5 や Cisco VIP などの外付けロードバランサを配置して RADIUS/TACACS+ トラフィックを均等分散します。また、Anycast BGP/EIGRP ルーティングを用いて NAD から最も近い PSN へ自動誘導する設計も一般的です。
- **pxGrid & Ecosystem Integration (FMC / Stealthwatch / DNA Center)**:
  - pxGrid ペルソナを介して、ISE が認証したセッション情報（ユーザー名、IP、MAC、SGT、デバイスタイプ）を Cisco Firepower Management Center (FMC) や Secure Network Analytics (Stealthwatch) に即時パブリッシュし、アイデンティティベースのセキュリティポリシー適用や Rapid Threat Containment (RTC / ANC) を実現します。

---

## 🧩 比較表

### 1. 展開モデル比較

| 展開モデル | 構成内容 | 最大接続セッション数 | 最大 PSN 数 | 主な用途 |
| --- | --- | --- | --- | --- |
| **Standalone** | 1台のノードに全ペルソナ (PAN+MnT+PSN) 同居 | 10,000 〜 50,000 | 0（同居のみ） | ラボ検証、テスト環境、小規模単一拠点 |
| **Small / Two-Node** | 2台のノードそれぞれに全ペルソナ同居 (Active/Active 相互ミラー) | 10,000 〜 50,000 | 0（同居のみ） | 小中規模企業、標準的な2ノード冗長構成 |
| **Hybrid** | 2台の PAN+MnT 複合ノード + 独立した PSN | 最大 50,000 | 最大 5 台 | 中規模多拠点、PSN の物理的・地理的分散 |
| **Fully Distributed** | 独立した PAN(2台)、MnT(2台)、PSN(最大50台)、pxGrid(最大4台) | 最大 2,000,000 | 最大 50 台 | 大規模エンタープライズ、グローバル展開 |

---

### 2. ISE 各ペルソナの役割・冗長化比較

| ペルソナ | 主な役割 | 許容ノード数 | 冗長化メカニズム | 障害時の影響 |
| --- | --- | --- | --- | --- |
| **PAN (Admin)** | 設定管理、マスターDB保持、証明書発行 | 最大 2 台 | Primary/Secondary (Warm-Spare) | 認証処理は継続するが、ポリシー設定変更・新規ノード追加・同期が不可 |
| **MnT (Monitoring)** | Syslog集約、Live Logs、レポート、アラート | 最大 2 台 | Primary/Secondary (Active/Active ログ二重受信) | 片系障害時も他方でログ収集継続。全停止時はレポート・監査ログ記録停止 |
| **PSN (Policy Service)** | RADIUS/TACACS+ 認証、ポータル、プロファイリング | 最大 50 台 | Active/Active (NAD側切替 / NLB / Node Group) | 該当 PSN への接続断。NAD または NLB が別 PSN へ自動切替 |
| **pxGrid** | 異種製品間コンテキスト共有パブリッシュバス | 最大 4 台 | Active/Active (全ノードアクティブ) | 一部障害時も他ノードへ再接続。全停止時は外部連携（FMC/Stealthwatch）停止 |

---

## 💡 ベストプラクティス

1. **DNS 正引き・逆引きの完全整備**:
   - ISE クラスターを構築する前に、全 ISE ノードの A レコードおよび PTR レコードを内部 DNS サーバーに正しく登録してください。
2. **厳格な NTP 時刻同期**:
   - すべての ISE ノードが信頼できる同一の NTP サーバーを参照するよう構成してください。時刻偏差が大きくなるとデータベースレプリケーションが破綻します。
3. **Dedicated ペルソナの推奨**:
   - セッション数が 20,000 を超える環境、または大規模環境では、PAN、MnT、PSN のペルソナを独立したノードに分離してください。特に MnT のログ書き込み負荷は高いため、専用ノード化が推奨されます。
4. **証明書管理と SAN (Subject Alternative Name)**:
   - マルチノード構成および pxGrid / ロードバランサ構成では、ワイルドカード証明書または全ノードの FQDN/VIP を含めた SAN 属性を持つ内部 CA 証明書を使用してください。
5. **Node Group の活用**:
   - 同一データセンターまたは同一 L2/L3 ドメイン内の PSN 群には Node Group を設定し、Web リダイレクトセッションの自動 CoA 切替およびレプリケーション最適化を図ってください。

---

## 📝 ラボ学習・設定サンプル例

### 演習1: ISE Primary PAN の展開初期化とノード設定
- **要件**:
  - ISE-1 (`ise-pan1.lab.local`, IP: 10.1.100.11) を Primary PAN および Primary MnT として初期設定する。
- **設定例**:
  - ISE-1 にログイン後、**Administration > System > Deployment** を開き、`ise-pan1` の Role が `PRIMARY`（Administration および Monitoring）になっていることを確認。
- **検証方法**:
  - CLI にて <code>show application status ise</code> を実行し、全主要プロセスが `running` であることを確認。

---

### 演習2: Secondary PAN / Secondary MnT ノードの登録
- **要件**:
  - Standalone 状態の ISE-2 (`ise-pan2.lab.local`, IP: 10.1.100.12) を Secondary PAN / Secondary MnT としてクラスターに追加する。
- **設定例**:
  1. Primary PAN GUI の **Administration > System > Deployment** より **Register** をクリック。
  2. FQDN: `ise-pan2.lab.local`, Credentials: `admin` / `Cisco12345!` を入力。
  3. Role: `SECONDARY`, Administration にチェック、Monitoring にチェック（Role: SECONDARY）を入れて保存。
- **検証方法**:
  - Primary PAN の Deployment 画面で `ise-pan2` の Status が緑アイコン（Connected / In Sync）になることを確認。
  - CLI (ISE-2): <code>show logging application config-replica.log</code> で Sync 完了を確認。

---

### 演習3: Dedicated PSN の追加とプロファイリングサービスの有効化
- **要件**:
  - ISE-3 (`ise-psn1.lab.local`, IP: 10.1.100.13) を Dedicated PSN として登録し、Session Services および Profiling Service を有効化する。
- **設定例**:
  1. Primary PAN より Register をクリック。
  2. FQDN: `ise-psn1.lab.local` を指定。
  3. Administration および Monitoring のチェックを外し、`Policy Service` にチェック。
  4. `Enable Session Services` および `Enable Profiling Service` にチェックを入れて保存。
- **検証方法**:
  - Deployment 画面で `ise-psn1` が `Policy Service (ALL)` として正しく表示されることを確認。

---

### 演習4: PSN Node Group の構成
- **要件**:
  - ISE-3 (`ise-psn1`) と ISE-4 (`ise-psn2`) を含む Node Group `DC1-PSN-Group` を作成する。
- **設定例**:
  1. **Administration > System > Deployment** のツリーから **Create Node Group** をクリック。
  2. Name: `DC1-PSN-Group` を入力し作成。
  3. `ise-psn1` および `ise-psn2` の編集画面にて `Include Node in Node Group` から `DC1-PSN-Group` を選択して保存。
- **検証方法**:
  - Deployment 左ツリーで `DC1-PSN-Group` の配下に両 PSN が表示されていることを確認。

---

### 演習5: Auto-PAN Failover の構成
- **要件**:
  - Primary PAN (`ise-pan1`) 障害時に Secondary PAN (`ise-pan2`) が自動昇格するよう設定。Health Check Node として `ise-psn1` を指定。
- **設定例**:
  1. **Administration > System > Deployment > PAN Failover Configuration** を開く。
  2. `Enable PAN Auto Failover` にチェック。
  3. `Primary Health Check Node`: `ise-psn1.lab.local`
  4. `Secondary Health Check Node`: `ise-psn2.lab.local`
  5. `Polling Interval`: 120, `Failure Polls`: 5 に設定し保存。
- **検証方法**:
  - 画面上に Failover 構成が正常保存されたメッセージを確認。

---

### 演習6: pxGrid Controller ペルソナの有効化
- **要件**:
  - `ise-psn1` 上で pxGrid Controller ペルソナを有効化する。
- **設定例**:
  1. **Administration > System > Deployment** にて `ise-psn1` を選択。
  2. `pxGrid` チェックボックスをオンにして保存。
- **検証方法**:
  - **Administration > pxGrid Services** 画面が開けるようになり、Internal ノードが Online であることを確認。

---

### 演習7: Dedicated TACACS+ PSN の分離設定
- **要件**:
  - `ise-psn2` をネットワークデバイス管理（Device Admin / TACACS+）専用 PSN として設定する。
- **設定例**:
  1. Deployment 画面にて `ise-psn2` の編集を開く。
  2. `Policy Service` 配下の `Enable Device Admin Service` にチェック。
  3. 不要なプロファイリングやポスチャ等のチェックを外し保存。
- **検証方法**:
  - **Work Centers > Device Administration** 画面の Deployment で `ise-psn2` が Device Admin 有効として認識されていることを確認。

---

### 演習8: NTP 同期障害の検知と復旧
- **要件**:
  - `ise-psn1` の NTP 同期が外れて Sync エラーが発生した際の障害追跡と復旧。
- **設定例 / 復旧手順**:
  1. CLI にて <code>show ntp</code> を実行し、unsynchronized 状態を確認。
  2. CLI: <code>config t</code> -> <code>ntp server 10.1.1.254 interpolate</code> を再設定。
  3. GUI Deployment にて `ise-psn1` を選択し **Syncup** を実行。
- **検証方法**:
  - <code>show ntp</code> で `synchronised to NTP server` を確認し、GUI の Node Status が緑色に復旧することを確認。

---

### 演習9: NAD（スイッチ）側での複数 PSN RADIUS 冗長設定
- **要件**:
  - Cisco IOS-XE スイッチ上で `ise-psn1` (10.1.100.13) をプライマリ、`ise-psn2` (10.1.100.14) をセカンダリ RADIUS サーバーとして設定。
- **設定例**:
```text
radius server ISE-PSN1
 address ipv4 10.1.100.13 auth-port 1812 acct-port 1813
 key Cisco123secret

radius server ISE-PSN2
 address ipv4 10.1.100.14 auth-port 1812 acct-port 1813
 key Cisco123secret

aaa group server radius ISE-CLUSTER
 server name ISE-PSN1
 server name ISE-PSN2
```
- **検証方法**:
  - スイッチ上で <code>show aaa servers</code> を実行し、両 PSN が STATE UP であることを確認。

---

### 演習10: Manual Primary PAN Promotion（手動昇格テスト）
- **要件**:
  - Primary PAN メンテナンスに伴い、Secondary PAN (`ise-pan2`) を手動で Primary に昇格させる。
- **設定例**:
  1. Secondary PAN (`ise-pan2`) の GUI にログイン。
  2. **Administration > System > Deployment** を開き、自ノードを選択。
  3. **Promote to Primary** ボタンをクリック。
- **検証方法**:
  - 両ノードでアプリケーションが再起動し、`ise-pan2` が Primary PAN として認識されることを確認。


## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験および筆記試験を意識した実践的な想定問題と詳細解説です。

### 質問 1 (コンフィグ/設計)
**問題**:
ある企業が Cisco ISE 3.1 を展開しており、現在 PAN+MnT が同居した 2 台のノードと 5 台の Dedicated PSN を運用しています。今後、エンドポイント数が 100,000 に増加する見込みです。クラスターの再設計として最も適切な対処はどれですか？

A. 単に新しい PSN を 5 台追加し、合計 10 台の PSN に拡張する。<br>
B. PAN と MnT を別々の専用ノード（Dedicated PAN, Dedicated MnT）に分離・再構成し、PSN を追加する。<br>
C. Auto-PAN Failover の間隔を短縮してデータベース処理能力を向上させる。<br>
D. pxGrid ペルソナをすべての PSN 上で有効化して負荷分散する。

**正解**: **B**<br>
**解説**:
PAN と MnT が同居するハイブリッド展開モデルでは、サポートされる最大セッション数は 50,000 エンドポイント、PSN 上限は 5 台に制限されます。50,000 セッションを超えて 100,000 規模（最大 2,000,000）へ拡張するには、PAN と MnT をそれぞれ独立した専用ノード（Dedicated PAN, Dedicated MnT）に分離する Fully Distributed 展開モデルへリファクタリングする必要があります。

---

### 質問 2 (トラブルシューティング)
**問題**:
新しいセカンダリノード `ise-psn3.lab.local` を Primary PAN から登録しようとしましたが、プログレスバーが 80% で停止し、最終的に "Node Registration Failed" となりました。CLI の `config-replica.log` を確認したところ、TLS ハンドシェイクおよびデータベース同期接続がタイムアウトしていました。ネットワーク接続に問題がない場合、原因として最も可能性が高いものはどれですか？

A. PSN 上で TC-NAC サービスが有効化されていないため。<br>
B. 登録対象ノードの IP アドレスに対する DNS 逆引き（PTR レコード）が登録されていない、または時刻が非同期。<br>
C. Primary PAN の HDD 容量が 50% を超えているため。<br>
D. PSN Node Group の数が上限を超えているため。

**正解**: **B**<br>
**解説**:
ISE のノード登録およびレプリケーション（JGroups/HTTPS）処理では、相互の FQDN 解決（DNS 正引き・逆引き PTR）および厳格な NTP 時刻同期が不可欠です。DNS 逆引きが失敗するか、時刻に偏差があると、証明書・TLS 認証またはデータチャネルの確立に失敗し、80% 付近でハング・失敗します。


### 質問 3 (実装/機能)
**問題**:
Cisco ISE の Auto-PAN Failover 機能について正しく説明しているステートメントはどれですか？

A. Primary PAN と Secondary PAN がハートビート（TCP/7802）を直接送受信し、切断時に自動昇格する。<br>
B. ネットワークロードバランサ (NLB) が PAN の VIP をヘルスチェックし、アクティブノードを切り替える。<br>
C. クラスター内の独立した第3のノード（Health Check Node として指定された PSN 等）が Primary PAN をポーリング監視し、障害検知時に Secondary PAN を自動昇格させる。<br>
D. Auto-PAN Failover が完了すると、旧 Primary PAN が復旧した際に自動的に Primary 役に切り戻される（Preemption）。

**正解**: **C**<br>
**解説**:
ISE の Auto-PAN Failover は、PAN 同士ではなく、指定された Health Check Node（非 PAN ノード）が Primary PAN をポーリング監視することでスプリットブレインを回避します。また、旧 Primary PAN が復旧しても自動切戻し（Preemption）は行われず、手動で同期・昇格を行う必要があります。

---

### 質問 4 (トラブルシューティング)
**問題**:
ゲストアクセス運用中、特定の PSN がネットワーク障害でダウンしました。該当 PSN で WebAuth リダイレクト中だったエンドポイントが、別の健全な PSN へ自動的に誘導されず、リダイレクトエラーのまま取り残されました。原因として考えられる設定不足はどれですか？

A. 該当 PSN が PSN Node Group に追加されていなかった。<br>
B. 該当 PSN で pxGrid サービスが有効化されていなかった。<br>
C. Secondary PAN の REST API が無効化されていた。<br>
D. MnT ノードで Syslog UDP/20514 が受信できていなかった。

**正解**: **A**<br>
**解説**:
PSN Node Group は、グループ内の PSN 相互間でピア監視（TCP/7800/7802）を行い、メンバー PSN がダウンして孤立した Web リダイレクト（Guest/Posture/MDM）セッションを検知した際に、他メンバー PSN が NAD に CoA を送信して別 PSN へセッションをリダイレクト・リカバリさせる機能を提供します。Node Group 未設定の場合、この自動追従が行われません。

---

### 質問 5 (設計)
**問題**:
Cisco ISE クラスターにおける Monitoring & Troubleshooting (MnT) ノードの高可用性 (HA) 動作について正しい記述はどれですか？

A. Primary MnT のみがアクティブであり、Secondary MnT は待機（Warm-Spare）している。<br>
B. すべての PSN および PAN は、Primary MnT と Secondary MnT の両方に対して同時に Syslog (UDP/TCP 20514) を二重送信する（Active/Active ログ集約）。<br>
C. MnT ノード間のログ同期は TCP/7800 を使用してリアルタイムに双方向レプリケーションされる。<br>
D. MnT ノードの障害時は、PAN がログバッファを一時保持し、復旧後に一括転送する。

**正解**: **B**<br>
**解説**:
MnT ノードの HA は Active/Active 構成であり、全 ISE ノードは発生したログを Primary MnT と Secondary MnT の両方に同時に送信（二重書き込み）します。これにより、片方の MnT が障害停止してもログの欠損が発生せず、もう一方の MnT で即座に Live Logs やレポートを参照できます。

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術資料一覧です。

- Cisco ISE 3.1 Administration Guide - Multi-Node Distributed Deployments
- Cisco ISE Performance & Scalability Guide
- Cisco ISE Community Resources & Validated Design
- Cisco Live Presentation: BRKSEC-2042 - Architectural Approaches to ISE Scaling and Resilience
- Cisco Live Presentation: BRKSEC-3032 - Advanced ISE Deployment and Troubleshooting
- Cisco ISE Port Reference Guide

---

## 📝 **補足（Notes）**

### 試験直前のチェックリスト
- [ ] 全ノードの FQDN/IP、DNS 逆引き (PTR)、NTP 同期状態が正常か確認できるか？
- [ ] Primary PAN からのノード登録手順および証明書インポート（Root CA 信頼）の流れを把握しているか？
- [ ] Auto-PAN Failover の設定項目（Health Check Node, Polling Interval, Failure Poll Count）の役割を説明できるか？
- [ ] PSN Node Group の役割（WebAuth CoA 切替 & JGroups 最適化）と使用ポート（TCP/7800, TCP/7802）を暗記しているか？
- [ ] 各ペルソナの上限値（PAN: 2, MnT: 2, PSN: 50, pxGrid: 4, TC-NAC: 1）を即答できるか？
