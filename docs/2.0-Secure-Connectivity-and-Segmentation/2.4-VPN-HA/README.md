---
layout: default
title: 2.4-VPN-HA
nav_order: 4
parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.4.a Cisco ASA VPN clustering

Cisco セキュリティアプライアンス（Cisco ASA 5500-X シリーズ、ISA 3000、Firepower 4100/9300 シリーズ上の ASA / FTD）における **ASA VPN Clustering（VPN クラスタリング）** は、複数台の ASA / FTD アプライアンスを 1 つの論理ファイアウォール（Single Logical Unit）として透過的に統合し、Site-to-Site IPsec VPN および AnyConnect Remote Access VPN (RA-VPN) セッションのスループレット線形拡張（Scale-out）と N+1 高可用性（High Availability / Resiliency）を同時に実現するエンタープライズ統合アーキテクチャです。

CCIE Security v6.1 Blueprint **2.4.a Cisco ASA VPN clustering** では、**Spanned EtherChannel Mode** および **Individual Interface Mode** における VPN トラフィック処理、**Control Node (Master)** と **Data Node (Slave)** の選定と役割分担、**Cluster Control Link (CCL)** 上での VPN ステート同期・ダイレクト転送、**Centralized VPN Features**（IPsec S2S / Remote Access VPN 終端における制御プレーンとデータプレーンの振る舞い）、**ASA VPN Load Balancing Cluster**（IPsec/SSL VPN 固有のディスパッチャ・リダイレクト技術）とのアーキテクチャ上の差異、および高度な障害復旧・トラブルシューティング能力が深く問われます。

---

## 📘 概要

* **機能概要**:
  * **ASA Clustering (システムレベルのクラスタリング)**:
    * 最大 16 台（プラットフォーム依存）の ASA / FTD アプライアンス群を 1 つの IP アドレス・仮想アプライアンスとして統合。
    * **Spanned EtherChannel モード**（L2/L3）または **Individual Interface モード**（L3 のみ）で動作し、全ノードでアクティブにトラフィックを分散・処理する。
    * VPN 機能においては、**Centralized Features（中央集権型機能）** として Control Node (Master) が IKEv1/IKEv2 ネゴシエーションや Dynamic Tunnel 割り当てを統括し、データプレーンまたはセッション情報を Data Node へ動的配信する。
  * **ASA VPN Load Balancing Cluster (VPN 固有のロードバランシングクラスタ)**:
    * 複数台の ASA（独立した個別のファイアウォール）を **Virtual Cluster IP (VIP)** の下にグループ化（VPN Cluster Group）し、Incoming AnyConnect / IPsec クライアント接続要求を受信した Master (Virtual Master) が、最も負荷の低い Slave ASA の物理 IP アドレスへ HTTP 302 リダイレクトまたは IKEv2 Redirect 応答を返して接続を分散する分散負荷制御技術。
* **利用目的**:
  * 数万人規模の Remote Access AnyConnect ユーザーおよび多数の拠点 Site-to-Site IPsec トンネルを単一のエンドポイント（FQDN / IP）で収容し、単一アプライアンスの暗号化処理能力（Crypto Engine / Throughput）の限界を破る。
  * クラスタ内の 1 台のアプライアンスがハードウェア障害や回線障害を起こした場合でも、残りのノードがアクティブに VPN セッションを引き継ぎ、サービス停止時間をゼロまたは極小化する。
* **どのような場面で利用するか**:
  * 大規模エンタープライズのグローバル RAVPN 集約ヘッドエンド。
  * ミッションクリティカルな Data Center 相互接続（DCI）における大容量 Site-to-Site IPsec VPN 終端。

## 🔑 要点

| 項目 | ASA Clustering (System Cluster) | ASA VPN Load Balancing Cluster |
| :--- | :--- | :--- |
| **アーキテクチャ** | 複数台の ASA を **1台の論理 ASA** に統合 (Single IP/MAC) | **独立した複数台の ASA** をグループ化 (VIP 共有) |
| **インターフェイス構成** | Spanned EtherChannel Mode / Individual Interface Mode | 各 ASA が独立したインターフェイス・IP アドレスを保持 |
| **専用相互接続線** | **Cluster Control Link (CCL)** が必須 (MTU 9184 推奨) | 不要 (LAN / UDP 902 ポートでクラスタキープアライブ交換) |
| **VPN セッション処理** | **Centralized / Distributed**: Control Node が制御、Data Node 処理 | **Fully Distributed**: 各 ASA が独立して TLS/DTLS/IPsec を完全終端 |
| **対応 VPN プロトコル** | Site-to-Site IPsec (VTI/Crypto Map), AnyConnect Remote Access | AnyConnect SSL/TLS/DTLS, Clientless SSL VPN, IKEv2 RA |
| **リダイレクト方式** | LACP ハッシュ / ECMP / CCL リダイレクト処理 | HTTP 302 Redirect, IKEv2 Redirect (RFC 5685) |
| **ノード障害時挙動** | ノード脱落時、他ノードがセッション・IP を即時引継ぎ | 障害ノードのユーザーは切断、VIP 経由で別ノードへ自動再接続 |
| **最大構成台数** | 最大 16 台 (Firepower 4100/9300, ASA 5585-X 等) | 最大 10 台 |
| **ライセンス要件** | ASA Cluster License (各ノードに必要) | Base License 内包 (追加ライセンス不要) |


---

## 🏗 動作原理

### 1. ASA Clustering における Centralized VPN フロー (Control Node ＆ Data Node)

ASA System Clustering において、VPN 制御（Site-to-Site IPsec / AnyConnect）は **Centralized Feature（中央集権型機能）** として管理されます。外部からのトンネル確立ネゴシエーション（IKEv1/IKEv2 / SSL TLS）は、常に **Control Node (Master)** が主導して制御します。

```text
[ AnyConnect Client / Remote IPsec Peer ]
                   │
                   ├─── (1) Initial VPN Connection Request (TLS SYN / IKE_SA_INIT)
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Switch Infrastructure (Spanned EtherChannel / cLACP)                             │
└──────────────────────────────────────────────────────────────────────────────────┘
                   │
                   ├─── Hash routes packet to Data Node B ─────────────────────────┐
                                                                                   │
┌──────────────────────────────────────────────────────────────────────────────────┼┐
│ ASA Cluster                                                                      ││
│                                                                                  ││
│  [ Data Node B (Slave) ] ◄───────────────────────────────────────────────────────┘│
│     ├── Centralized VPN Traffic Inspection                                        │
│     └── Encapsulates & Redirects Packet via CCL ───┐                              │
│                                                    │                              │
│ ┌──────────────────────────────────────────────────┼────────────────────────────┐ │
│ │ Cluster Control Link (CCL - MTU 9184 Jumbo Frame)│                            │ │
│ └──────────────────────────────────────────────────┼────────────────────────────┐ │
│                                                    │                              │
│  [ Control Node A (Master) ] ◄─────────────────────┘                              │
│     ├── Executes IKE / TLS Handshake & Authentication                             │
│     ├── Allocates Inner IP Address (Local Pool / RADIUS)                          │
│     └── Replicates IPsec/TLS SA Keys & Session State to Data Nodes via CCL        │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

### 2. ASA VPN Load Balancing Cluster (Redirect Flow) 動作原理

独立した複数台の ASA で構成される VPN Load Balancing Cluster において、Virtual Cluster IP (VIP) を使用してクライアントを最少負荷のノードへ振る流し構造：

```text
[ AnyConnect Client ] ───(1) HTTPS GET / IKE_SA_INIT to VIP (192.168.1.100) ───► [ Master ASA (Virtual VIP Owner) ]
                                                                                   │
                                                                                   ├── Evaluates Load Status of All Members
                                                                                   └── Selects Lowest-Load Slave ASA 2
                                                                                   │
[ AnyConnect Client ] ◄──(2) HTTP 302 Redirect / IKEv2 Redirect (to 192.168.1.102) ┘
       │
       └───(3) Direct TLS/DTLS / IKEv2 Connection ───────────────────────────────► [ Slave ASA 2 (192.168.1.102) ]
                                                                                   └── Terminates Tunnel & Forwards Traffic
```

---

## ⚙ 動作シーケンス

### 1. ASA System Clustering での VPN セッション処理シーケンス

1. **クラスタブート ＆ Control Node 選定**:
   * ASA クラスタグループが起動し、`priority` 数値が最も小さいアプライアンスが **Control Node (Master)** として選出される。
   * Control Node は VPN 仮想インターフェイスおよび Cluster Virtual IP の制御権限をアサートする。
2. **インバウンド VPN 接続要求の受信 ＆ CCL 転送**:
   * AnyConnect クライアントまたは対向 IPsec ピアからのトンネル確立パケット（TCP 443 / UDP 500 / UDP 4500）が着信。
   * スイッチの LACP ハッシュによって Data Node (Slave) に着信した場合、Data Node はセントラライズドプロトコル（VPN）であることを認識し、パケットを **Cluster Control Link (CCL)** を経由して Control Node へリダイレクトする。
3. **Control Node によるトンネル処理 ＆ セッションレプリケーション**:
   * Control Node が IKE 認証、SAML 2.0 / RADIUS 認証、AnyConnect グループポリシー適用、および IP アドレス割り当てを完了させる。
   * トンネル確立後、Control Node は暗号化キー（IPsec SA / TLS Session Key）および接続状態情報を CCL 経由で全 Data Node（または指定された Director Node）へ一括同期（Replicate）する。
4. **データプレーンのフォワーディング処理**:
   * トンネル確立後の暗号化データパケット（ESP / DTLS）は、Data Node 上で直接復号処理され、内部ネットワークへ転送される。

---

### 2. ASA VPN Load Balancing Cluster の処理シーケンス

1. **クラスタ状態ハートビート交換**:
   * クラスタ内の全 ASA（Master および Slave）は、UDP 902 ポート（デフォルト）を使用して LAN 上でハートビートメッセージおよび現在のアクティブ接続数・CPU 使用率データを定期交換する。
2. **仮想マスター (Virtual Master) による初回受信 ＆ 振分判定**:
   * クライアントは Cluster VIP (`192.168.1.100`) 宛てに接続。VIP を保持する Virtual Master がリクエストを受信。
   * Virtual Master は自機を含む全メンバーの負荷状態を比較し、最も負荷の低いノード（例: Slave 2 `192.168.1.102`）を選択。
3. **リダイレクト応答の送信**:
   * **SSL / HTTPS の場合**: Master は `HTTP/1.1 302 Redirect` レスポンスを送信 (`Location: https://...`)。
   * **IKEv2 の場合**: Master は `IKE_SA_INIT` 応答内で `N(REDIRECT_SUPPORTED)` 通知ヘッダーを解釈し、`IKEv2 Redirect` 応答メッセージを送信。
4. **クライアントの物理ノード直接接続**:
   * AnyConnect クライアントは切断を受け入れ、指定された Slave 2 の物理 IP アドレス (`192.168.1.102`) へ直接再接続を実施し、トンネルを完全終端する。


---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **SAML 2.0 認証を伴う VPN Load Balancing での FQDN リダイレクト (`vpn load-balancing send-fqdn`)**:
   * **試験の罠**: ASA VPN Load Balancing Cluster 環境で SAML 2.0 (Azure AD / Duo 等) シングルサインオンを導入する場合、デフォルト（IP アドレスでのリダイレクト）ではクライアントが Redirect された際、URL が IP アドレス表記に変化し、IdP（Identity Provider）側の Redirect URI 検証および SSL 証明書エラーで認証が破綻します。
   * **必須対策**: `vpn load-balancing` モード配下で **`send-fqdn`** コマンドを投入し、Redirect 応答メッセージ内に IP ではなく FQDN（例: `asa1-vpn.corp.local`）を含めさせることが強く要求されます。
2. **Cluster Control Link (CCL) の MTU サイズ設定 (9184 Bytes)**:
   * **必須要件**: ASA System Clustering 上で VPN データがノード間転送される際、元の IPsec / DTLS パケットにクラスタヘッダーがカプセル化（オーバーヘッド約 100 バイト）されて CCL を通過します。
   * CCL インターフェイスの MTU は **`9184`** バイト（Jumbo Frame）に指定し、スイッチ側の物理ポートも `system mtu jumbo 9000` 等でジャンボフレームを許可する必要があります。MTU が 1500 のままだと、大容量 VPN パケットが CCL 上で断片化・ドロップし、AnyConnect トンネルが著しく低速化または切断されます。
3. **ASA VPN Load Balancing の UDP 902 通信許可 ＆ グループ共有キー**:
   * **キープアライブポート**: クラスタノード間は UDP 902 ポートで制御メッセージ（負荷情報）を定期送信します。ノード間にファイアウォールが存在する場合は UDP 902 を許可しなければなりません。
   * **Shared Secret**: 全ノードで `shared-secret <KEY>` を完全に同一に揃える必要があります。
4. **AnyConnect IP Local Pool ＆ Subnet の注意点**:
   * System Clustering 環境では、IP アドレスプール（`ip local pool`）は Control Node で一元定義されます。クラスタ全体の同時接続ユーザー数に対応できる十分な CIDR ブロック（例: `/20` や `/21`）をあらかじめ確保する必要があります。

---

## 🛠 設定方法

### 1. ASA System Clustering (Spanned EtherChannel Mode) 設定

```bash
! ===== Step 1: Cluster Control Link (CCL) ＆ MTU 設定 =====
interface Port-channel48
 description Cluster Control Link
 port-channel min-links 1
 member-interface GigabitEthernet0/2
 member-interface GigabitEthernet0/3

mtu cluster 9184                        ! CCL の MTU を 9184 (Jumbo) に拡張

! ===== Step 2: Spanned Data EtherChannel 設定 =====
interface Port-channel1
 port-channel span-cluster
 nameif outside
 security-level 0
 ip address 192.168.10.100 255.255.255.0
 member-interface GigabitEthernet0/0
 member-interface GigabitEthernet0/1

! ===== Step 3: Cluster Group 有効化 (Master ASA-1) =====
cluster group VPN_CLUSTER
 local-unit ASA-1
 cluster-interface Port-channel48 ip 10.255.255.1 255.255.255.0
 priority 1                             ! 最少数値が Control Node 優先
 key cisco123
 enable
```

---

### 2. ASA VPN Load Balancing Cluster 設定 (SAML / AnyConnect 対応)

```bash
! ===== Master ASA (ASA-1) Configuration =====
ip local pool VPN_POOL 10.100.1.1-10.100.1.254 mask 255.255.255.0

vpn load-balancing
 cluster ip address 192.168.10.200       ! Virtual Cluster IP (VIP)
 priority 10                            ! 割り当て優先度
 public-ip 192.168.10.101               ! ASA-1 自身の物理 Outside IP
 fqdn asa1-vpn.corp.local               ! 自身の物理 FQDN
 send-fqdn                              ! SAML / Cert 認証破綻防止の FQDN リダイレクト
 shared-secret ClusterKey123
 master-encryption
 participate-in-load-distribution

! ===== Slave ASA (ASA-2) Configuration =====
vpn load-balancing
 cluster ip address 192.168.10.200       ! 同一の Virtual Cluster IP
 priority 5
 public-ip 192.168.10.102               ! ASA-2 自身の物理 Outside IP
 fqdn asa2-vpn.corp.local
 send-fqdn
 shared-secret ClusterKey123
 participate-in-load-distribution
```

---

### 3. ASA Clustering 上での AnyConnect RA-VPN トンネルグループ設定

```bash
! ===== Tunnel Group ＆ Group Policy 定義 =====
group-policy GP_CORPORATE internal
group-policy GP_CORPORATE attributes
 vpn-tunnel-protocol ssl-client ikev2
 default-domain corp.local
 split-tunnel-policy tunnelall

tunnel-group TG_ANYCONNECT type remote-access
tunnel-group TG_ANYCONNECT general-attributes
 address-pool VPN_POOL
 default-group-policy GP_CORPORATE
tunnel-group TG_ANYCONNECT webvpn-attributes
 group-alias Corporate_VPN enable

webvpn
 enable outside
 anyconnect image disk0:/anyconnect-win-4.10.05095-webdeploy-k9.pkg 1
 anyconnect enable
```

---

## 🔍 検証コマンド

| 目的 | コマンド (ASA / FTD CLI) | 表示確認ポイント |
| :--- | :--- | :--- |
| **System Cluster ノード選定・状態確認** | <code>show cluster info</code> | `This is "node-1" in state MASTER`, `Data Node` の一覧 |
| **Cluster Health ＆ CCL MTU ステータス確認** | <code>show cluster info health</code> | `Unit Health: PASSED`, `Interface Health: PASSED`, `CCL MTU: 9184` |
| **VPN Load Balancing Cluster の動作状態確認** | <code>show vpn load-balancing</code> | `Role: Master/Slave`, `Cluster IP`, `Participating: Yes`, メンバー一覧と負荷度 |
| **VPN クラスタ全体のセッションDB一括表示** | <code>show vpn-sessiondb cluster</code> / <code>cluster exec show vpn-sessiondb anyconnect</code> | 各ノードでアクティブに収容されている AnyConnect ユーザーセッション一覧 |
| **VPN Load Balancing デバッグログ追跡** | <code>debug vpn load-balancing code</code><br><code>debug vpn load-balancing cluster</code> | リダイレクト判定、UDP 902 パケット交換、VIP アナウンスのログ |


---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **VPN Load Balancing でクライアントが特定の Master ノードにしか接続されず、負荷分散されない** | クラスタノード間の UDP 902 ポートが遮断されている、または `shared-secret` キーが不一致 | <code>show vpn load-balancing</code><br><code>debug vpn load-balancing cluster</code> | ノード間で UDP 902 ポートの疎通を確保し、全 ASA で `shared-secret` 設定を完全に一致させる。 |
| **SAML 2.0 認証と VPN Load Balancing を併用時、Redirect 後に SSL 証明書警告または IdP 拒否エラーが発生** | デフォルトのリダイレクト動作で物理 IP アドレスが指定され、IdP の Redirect URI 条件と不一致を起こしている | <code>show vpn load-balancing</code> | `vpn load-balancing` モード配下で **`send-fqdn`** コマンドを有効化し、FQDN でリダイレクトさせる。 |
| **System Clustering 上で AnyConnect 通信の遅延・パケットドロップが激しい** | Cluster Control Link (CCL) の **MTU サイズが 1500** のままで、カプセル化パケットが断片化・ドロップしている | <code>show cluster info health</code><br><code>show interface Port-channel48</code> | `mtu cluster 9184` コマンドを投入し、スイッチ側ポートも Jumbo Frame (9000 以上) を許可する。 |
| **Master ノードダウン時、AnyConnect セッションが一斉に切断される** | SSL/TLS セッションキーの即時同期が行われず、クライアント側 Auto-Reconnect タイマーが失効した | <code>show vpn-sessiondb anyconnect</code> | AnyConnect プロファイルで **`Auto Reconnect`** を有効化し、再接続間隔を最適化する。 |

---

## ⚠ 制限事項

* **FTD Clustering における Remote Access VPN (AnyConnect) 非対応**:
  * Cisco FTD (Firepower Threat Defense) クラスタ環境では、Site-to-Site VPN のみがサポートされ、**AnyConnect RA-VPN は完全非対応** です（※RA-VPN 負荷分散には ASA VPN Load Balancing Cluster または AnyConnect TND/FQDN DNS 振分けを利用）。
* **ASA VPN Load Balancing Cluster の構成上限**:
  * VPN Load Balancing Cluster グループに所属できる ASA ノード数は **最大 10 台** に制限されます。
* **SSL/TLS トンネル暗号化キーの非動的同期**:
  * SSL/TLS トンネルは本質的に状態フルなプロトコルであり、アプライアンス障害発生時は AnyConnect クライアントが別ノードへ自動再接続（Auto-Reconnect）を試行して新トンネルを再確立します。

---

## 🔄 他技術との関連

* **SAML 2.0 / RADIUS 認証**: AnyConnect ユーザーの認証・認可。VPN Load Balancing 時は FQDN リダイレクト (`send-fqdn`) が必須。
* **Cisco ISE (Identity Services Engine)**: RADIUS CoA、SGT (Security Group Tag)、および DACL 動的割り当て。
* **Spanned EtherChannel / cLACP**: System Clustering において複数ノードを 1 つの Port-Channel に束ねるデータ面技術。

---

## 🧩 比較表

### ASA System Clustering vs ASA VPN Load Balancing Cluster vs ASA Failover (A/S ＆ A/A)

| 比較項目 | ASA System Clustering | ASA VPN Load Balancing Cluster | ASA Active/Standby Failover | ASA Active/Active Failover |
| :--- | :--- | :--- | :--- | :--- |
| **最大ノード数** | **最大 16 台** | **最大 10 台** | 2 台 | 2 台 |
| **IP / MAC 表現** | 単一の論理 IP / MAC | 個別の物理 IP ＋ 共有 VIP | 共有 Virtual IP / Active-Standby MAC | Context 別の共有 IP / MAC |
| **スループレット** | **N台分の線形拡張 (Scale-out)**| **N台分の線形拡張 (Scale-out)**| 1台分 (Standby は待機) | 最大 2台分 (分散時) |
| **専用相互接続** | **CCL (MTU 9184)** | 不要 (LAN / UDP 902) | Failover LAN ＋ Stateful Link | Failover LAN ＋ Stateful Link |
| **AnyConnect RA-VPN**| 対応 (Centralized 制御) | **最適 (HTTP/IKEv2 Redirect)**| 対応 (Stateful Failover) | 非対応 (RA-VPN は A/A 不可) |

---

## 💡 ベストプラクティス

1. **SAML / PKI 認証環境での `send-fqdn` 標準適用**:
   * VPN Load Balancing を構成する際は、将来の SAML / MFA / 証明書認証導入を見越し、最初から `send-fqdn` および各ノードの物理 FQDN / パブリック証明書をアサインします。
2. **Cluster Control Link (CCL) 9184 Jumbo Frame 確保**:
   * System Clustering を採用する場合、データ面パケットのカプセル化ドロップを防ぐため、CCL の MTU を 9184 バイトに設定することを設計原則とします。
3. **独立した AnyConnect IP Address Pool の設計**:
   * 各ノードが独自に IP を割り当てる構成では、ノードごとに重複しないサブネットブロックをアサインして内部ルーティングの不整合を防ぎます。


---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA System Clustering (Spanned EtherChannel ＆ CCL 9184) の基本構築

#### 問題
2台の ASA (ASA-1, ASA-2) 間で Spanned EtherChannel モードのクラスタ `VPN_CLUSTER` を構築せよ。`Port-channel48` を Cluster Control Link (CCL) としてアサインし、CCL の MTU を 9184 バイトに設定せよ。

#### 要件
* ASA-1 の Priority を `1` (Master 優先)、ASA-2 を `10` に設定すること。
* データ面インターフェイスは `Port-channel1` (Outside) とすること。

#### 設定例

```bash
! ===== ASA-1 (Master) Configuration =====
interface Port-channel48
 description Cluster Control Link
 port-channel min-links 1
 member-interface GigabitEthernet0/2
 member-interface GigabitEthernet0/3

mtu cluster 9184

interface Port-channel1
 port-channel span-cluster
 nameif outside
 security-level 0
 ip address 192.168.10.100 255.255.255.0
 member-interface GigabitEthernet0/0
 member-interface GigabitEthernet0/1

cluster group VPN_CLUSTER
 local-unit ASA-1
 cluster-interface Port-channel48 ip 10.255.255.1 255.255.255.0
 priority 1
 key cisco123
 enable

! ===== ASA-2 (Slave) Configuration =====
interface Port-channel48
 description Cluster Control Link
 member-interface GigabitEthernet0/2
 member-interface GigabitEthernet0/3

interface Port-channel1
 port-channel span-cluster

cluster group VPN_CLUSTER
 local-unit ASA-2
 cluster-interface Port-channel48 ip 10.255.255.2 255.255.255.0
 priority 10
 key cisco123
 enable
```

#### 検証方法
* `show cluster info` を実行し、`ASA-1 in state MASTER`、`ASA-2 in state SLAVE` と表示され、`show cluster info health` で `CCL MTU: 9184` であることを確認。

---

### 例2: ASA VPN Load Balancing Cluster (2台構成) の基本設定

#### 問題
2台の独立した ASA (ASA1, ASA2) 間で VPN Load Balancing Cluster を構築せよ。Virtual Cluster IP として `192.168.10.200` を使用し、負荷分散を有効化せよ。

#### 要件
* グループ共有キーに `ClusterKey123` を設定すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
vpn load-balancing
 cluster ip address 192.168.10.200
 priority 10
 public-ip 192.168.10.101
 shared-secret ClusterKey123
 master-encryption
 participate-in-load-distribution

! ===== ASA2 Configuration =====
vpn load-balancing
 cluster ip address 192.168.10.200
 priority 5
 public-ip 192.168.10.102
 shared-secret ClusterKey123
 participate-in-load-distribution
```

#### 検証方法
* `show vpn load-balancing` を実行し、ASA1 が `Role: Master`、ASA2 が `Role: Slave` で `Participating: Yes` と表示されることを確認。

---

### 例3: SAML 認証対応の FQDN リダイレクト (`send-fqdn`) 設定

#### 問題
ASA VPN Load Balancing Cluster において、SAML シングルサインオン時の証明書不一致および Redirect 失敗を防ぐため、リダイレクト時に IP ではなく物理 FQDN をクライアントへ通知するよう設定せよ。

#### 要件
* ASA1 FQDN: `asa1-vpn.corp.local`, ASA2 FQDN: `asa2-vpn.corp.local`

#### 設定例

```bash
! ===== ASA1 Configuration =====
vpn load-balancing
 fqdn asa1-vpn.corp.local
 send-fqdn

! ===== ASA2 Configuration =====
vpn load-balancing
 fqdn asa2-vpn.corp.local
 send-fqdn
```

#### 検証方法
* クライアント接続時に `debug vpn load-balancing code` を有効化し、HTTP 302 Redirect 応答ヘッダー内に `asa2-vpn.corp.local` が指定されていることを確認。

---

### 例4: ASA Clustering 上での AnyConnect トンネルグループと Address Pool

#### 問題
ASA クラスタ環境において、AnyConnect クライアント用のアドレスプール `POOL_ANYCONNECT` (`10.200.1.1`–`10.200.1.254`) を一元定義し、トンネルグループ `TG_VPN` にアサインせよ。

#### 要件
* アドレスプールは `/24` マスクとすること。

#### 設定例

```bash
! ===== ASA Master Console Configuration =====
ip local pool POOL_ANYCONNECT 10.200.1.1-10.200.1.254 mask 255.255.255.0

group-policy GP_VPN internal
group-policy GP_VPN attributes
 vpn-tunnel-protocol ssl-client ikev2
 split-tunnel-policy tunnelall

tunnel-group TG_VPN type remote-access
tunnel-group TG_VPN general-attributes
 address-pool POOL_ANYCONNECT
 default-group-policy GP_VPN
tunnel-group TG_VPN webvpn-attributes
 group-alias Corporate_RA_VPN enable
```

#### 検証方法
* `show running-config tunnel-group` を確認し、クラスタ全体でアドレスプールが正常に認識されていることを確認。

---

### 例5: IKEv2 / DTLS リダイレクト対応の VPN Load Balancing 設定

#### 問題
AnyConnect クライアントが IKEv2 または DTLS トンネルを確立する際、VPN Load Balancing が正常に動作するよう IKEv2 リダイレクト機能を有効化せよ。

#### 要件
* `vpn load-balancing` モード内で設定すること。

#### 設定例

```bash
! ===== ASA1 / ASA2 Configuration =====
vpn load-balancing
 group-url https://vpn.corp.local/anyconnect enable
 participate-in-load-distribution
```

#### 検証方法
* AnyConnect IKEv2 接続時に `debug ikev2 platform` を実行し、`IKEv2 Redirect` 通知パケットがクライアントへ返却されることを確認。

---

### 例6: ASA Clustering での Site-to-Site IPsec VPN (Centralized) の設定

#### 問題
ASA クラスタ環境において、対向ルーター `203.0.113.50` との Site-to-Site IKEv2 IPsec VPN を Centralized 機能として構成せよ。

#### 要件
* Pre-shared-key: `CiscoIPsecKey123`

#### 設定例

```bash
! ===== ASA Master Configuration =====
crypto ikev2 policy 10
 proposal aes-256 integrity sha256 group 14
exit

crypto ikev2 enable outside

tunnel-group 203.0.113.50 type ipsec-l2l
tunnel-group 203.0.113.50 ipsec-attributes
 ikev2 local-authentication pre-shared-key CiscoIPsecKey123
 ikev2 remote-authentication pre-shared-key CiscoIPsecKey123

access-list ACL_S2S extended permit ip 10.1.0.0 255.255.0.0 10.2.0.0 255.255.0.0

crypto ipsec ikev2 ipsec-proposal PROPOSAL_AES256
 protocol esp encryption aes-256 integrity sha-256

crypto map CMAP 10 match address ACL_S2S
crypto map CMAP 10 set peer 203.0.113.50
crypto map CMAP 10 set ikev2 ipsec-proposal PROPOSAL_AES256
crypto map CMAP interface outside
```

#### 検証方法
* `show crypto ikev2 sa` を実行し、Control Node (Master) 上で IKEv2 SA が確立されていることを確認。

---

### 例7: VPN Load Balancing ポートおよび Cluster Key 設定

#### 問題
ASA VPN Load Balancing Cluster において、クラスタ間キープアライブポートをカスタムポート `9050` に変更し、グループ共有キー `SecureKey999` を設定せよ。

#### 要件
* `cluster port` コマンドを使用すること。

#### 設定例

```bash
! ===== ASA1 / ASA2 Configuration =====
vpn load-balancing
 cluster port 9050
 shared-secret SecureKey999
```

#### 検証方法
* `show vpn load-balancing` を実行し、`Cluster Port: 9050` と表示されることを確認。

---

### 例8: Firepower 4100 FXOS 上での Inter-Chassis ASA VPN Clustering 構成

#### 問題
Firepower 4100 シャーシ 2台において、FXOS ディレクトリから ASA インスタンスのクラスタグループ `FXOS_CLUSTER` を構成せよ。

#### 要件
* CCL に `Port-channel 48` を割り振ること。

#### 設定例

```text
[ FXOS Chassis Manager Steps ]
1. FXOS UI ➔ Logical Devices ➔ Add Device
2. Name: ASA_NODE1, Template: ASA
3. [✓] Enable Clustering
   - Cluster Name: FXOS_CLUSTER
   - Cluster Control Link: Port-channel48
   - Local IP: 10.255.255.1/24
4. Save ＆ Deploy.
```

#### 検証方法
* FXOS CLI で `show cluster group` を実行し、両シャーシノードが `UP/ONLINE` であることを確認。

---

### 例9: ASA Clustering における AnyConnect Session Timeout/Reconnect 設定

#### 問題
ASA クラスタ環境において、ノード切替発生時に AnyConnect クライアントが切断されず 300 秒間透過的に再接続を試行するようグループポリシーを設定せよ。

#### 要件
* `auto-reconnect` タイマーを 300 秒に指定すること。

#### 設定例

```bash
! ===== ASA Configuration =====
group-policy GP_CORPORATE attributes
 anyconnect-auto-reconnect-timeout 300
 auto-reconnect enable
```

#### 検証方法
* `show running-config group-policy` を確認し、タイマーが 300 秒に設定されていることを確認。

---

### 例10: ASA VPN Load Balancing Master 障害時の Virtual Master 昇格設定

#### 問題
ASA VPN Load Balancing Cluster において、現在の Master (Priority 10) がダウンした際、即座に Slave (Priority 20) が Virtual Master へ昇格するよう Priority をチューニングせよ。

#### 要件
* ASA2 の Priority を `20` (最高値) に変更すること。

#### 設定例

```bash
! ===== ASA2 Configuration =====
vpn load-balancing
 priority 20
```

#### 検証方法
* ASA1 の外部ポートを `shutdown` にし、ASA2 上で `show vpn load-balancing` を実行して `Role: Master` へ変更されたことを確認。


---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 以下のコンフィグが設定された ASA1 および ASA2 の VPN Load Balancing クラスタ環境に関する記述として最も適切なものはどれか。

```bash
! ===== ASA1 =====
vpn load-balancing
 cluster ip address 192.168.10.200
 public-ip 192.168.10.101
 fqdn asa1-vpn.corp.local
 send-fqdn
 shared-secret Key123

! ===== ASA2 =====
vpn load-balancing
 cluster ip address 192.168.10.200
 public-ip 192.168.10.102
 fqdn asa2-vpn.corp.local
 send-fqdn
 shared-secret Key123
```

* A) SAML 2.0 認証を有効化した際、クライアントは Redirect 応答で IP アドレスを受け取り、証明書警告が表示される。
* B) `send-fqdn` が設定されているため、リダイレクト応答ヘッダーには IP アドレスではなく FQDN (`asa2-vpn.corp.local`) が挿入され、SAML SSO 認証や証明書検証の失敗を防止できる。
* C) この構成では IKEv2 トンネルのリダイレクトはサポートされない。
* D) ASA1 と ASA2 は ASA System Clustering モードで 1 台の論理アプライアンスとして統合されている。

**正解**: **B**  
**解説**: **`send-fqdn`** コマンドを定義することで、Master がクライアントへリダイレクトを指示する際、IP アドレスではなく FQDN (`asa2-vpn.corp.local`) を使用します。これにより、SAML IdP 側での Redirect URI 検証および SSL 証明書の Common Name / SAN 照合の破綻を防ぐことができます。

---

### 2. トラブルシュート問題

**問題**: ASA System Clustering モードで AnyConnect リモートアクセス VPN を構築した。トンネル自体は確立するが、大容量ファイル転送や Web 閲覧時にパケットの遅延およびドロップが著しく発生する。`show cluster info health` を確認したところ、CCL の MTU が `1500` バイトに指定されていた。最も適切な対処方法はどれか。

* A) ASA の Outside インターフェイスの MTU を 1400 に変更する。
* B) **`mtu cluster 9184`** コマンドを投入し、CCL の MTU をジャンボフレームに拡張するとともに、対向スイッチ側ポートも Jumbo Frame を許可する。
* C) `same-security-traffic permit intra-interface` を投入する。
* D) SSL VPN を全廃し IKEv2 IPsec に切り替える。

**正解**: **B**  
**解説**: ASA Clustering では、VPN パケットがノード間を転送される際にクラスタ用カプセル化ヘッダー（約 100 バイト）が付加されます。CCL の MTU が 1500 のままだとパケットが断片化・ドロップするため、**`mtu cluster 9184`** (Jumbo Frame) の設定が必須です。

---

### 3. Design 問題

**問題**: 新規マルチデータセンター設計において、Cisco FTD (Firepower Threat Defense) アプライアンス群による Inter-Chassis Clustering (Spanned Mode) を採用する予定である。この環境で 10,000 名規模の AnyConnect RA-VPN ユーザーを収容したい。設計上の判断として正しいものはどれか。

* A) FTD クラスタ上でそのまま AnyConnect RA-VPN トンネルグループを定義する。
* B) FTD クラスタは AnyConnect RA-VPN をサポートしないため、RA-VPN 用に**独立した ASA アプライアンス群による ASA VPN Load Balancing Cluster** を別個に設計・配置する。
* C) FTD を Transparent Mode に変更して AnyConnect を収容する。
* D) FTD 上で FlexConfig を用いて VPN Load Balancing を強制有効化する。

**正解**: **B**  
**解説**: **FTD Clustering 環境では Remote Access VPN (AnyConnect) は完全非サポート** です。そのため、RA-VPN ヘッドエンドとして独立した ASA による **ASA VPN Load Balancing Cluster** を配置するのが Cisco の標準推奨設計です。

---

### 4. 実装問題

**問題**: ASA VPN Load Balancing Cluster において、2台の ASA 間でハートビートメッセージおよび負荷度ステータスの動的交換に使用されるデフォルトのプロトコルおよび UDP ポート番号はどれか。

* A) TCP 443
* B) UDP 500
* C) **UDP 902**
* D) TCP 8443

**正解**: **C**  
**解説**: ASA VPN Load Balancing Cluster ノード間では、**UDP 902 ポート** を使用してクラスタキープアライブおよび負荷情報（現在のアクティブ接続数、CPU 使用率）を定期交換します。

---

### 5. コンフィグ読解問題

**問題**: ASA System Clustering における VPN (Site-to-Site IPsec ＆ Remote Access VPN) の機能カテゴリ分類（Feature Type）として正しいものはどれか。

* A) Distributed Feature (各ノードが完全に独立して処理)
* B) **Centralized Feature** (Control Node / Master が制御プレーン処理を管轄し、必要に応じて Data Node へ分散)
* C) Non-supported Feature (クラスタ上では一切利用不可)
* D) Asymmetric Feature (片方向のみ処理可能)

**正解**: **B**  
**解説**: VPN 機能は ASA Clustering において **Centralized Feature（中央集権型機能）** に指定されており、IKE / TLS ネゴシエーションやアドレス割り当てなどの制御プレーン処理は **Control Node (Master)** 上で集中的に実行されます。

---

## 🔗 参考リソース

* [Cisco ASA Series General Operations Configuration Guide: Clustering](https://www.cisco.com/)
* [Cisco ASA Series VPN Configuration Guide: Load Balancing VPN Cluster](https://www.cisco.com/)
* [Cisco Technical Notes: Configure SAML Single Sign-On with ASA VPN Load Balancing](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Cisco ASA Clustering Architecture and VPN Integration (BRKSEC-3032)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で VPN 高可用性 / クラスタリングが出題された場合の構成判断フローチャート：
    * **「Scale-out Remote Access VPN with SAML / FQDN redirect」** ➔ **ASA VPN Load Balancing Cluster** (`vpn load-balancing` ＋ `cluster ip` ＋ `send-fqdn` ＋ `fqdn <NODE_FQDN>`)。
    * **「Scale-out Firewall ＆ Site-to-Site IPsec VPN using single logical unit」** ➔ **ASA System Clustering** (`cluster group` ＋ `mtu cluster 9184` ＋ `port-channel span-cluster`)。
* **図解（VPN Load Balancing vs System Clustering）**:
  ```text
  [ ASA VPN Load Balancing Cluster ]
  - Client connects to VIP ──► Master ASA redirects to Slave ASA physical FQDN/IP via HTTP 302 / IKEv2 Redirect.
  - Each ASA is an independent firewall.

  [ ASA System Clustering ]
  - Client connects to Single Cluster IP ──► Switched via Spanned EtherChannel / cLACP.
  - Control Node (Master) handles VPN negotiation; Data Nodes handle decapsulation via CCL (Jumbo MTU 9184).
  ```
* **注意点**:
  * ラボ試験で VPN Load Balancing を設定した後は、必ず `show vpn load-balancing` を実行して各ノードの Role (Master / Slave) および `Participating: Yes` を確認し、SAML 環境では `send-fqdn` が投入されていることを二重チェックしてください。


