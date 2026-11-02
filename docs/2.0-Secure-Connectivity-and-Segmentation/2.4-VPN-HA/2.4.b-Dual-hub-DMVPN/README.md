---
layout: default
title: 2.4.b-Dual-hub-DMVPN
nav_order: 2
parent: 2.4-VPN-HA
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.4.b Dual-hub DMVPN deployments

Cisco インフラストラクチャにおける **Dual-hub DMVPN（Dynamic Multipoint VPN）デプロイメント** は、地理的に分散した多数の拠点（Spoke）と複数のセンタールーティング拠点（Hub）を動的 IPsec トンネルで相互接続し、単一障害点（Single Point of Failure）の排除、高い障害耐性（Resilience）、および負荷分散（Load Sharing）を実現するための大規模 WAN セキュリティアーキテクチャです。

CCIE Security v6.1 Blueprint **2.4.b Dual-hub DMVPN deployments** では、**Dual-Hub Single-Cloud** および **Dual-Hub Dual-Cloud** の論理トポロジ設計、**DMVPN Phase 1 / Phase 2 / Phase 3** における NHRP（Next Hop Resolution Protocol）の動作メカニズム（`ip nhrp redirect` および `ip nhrp shortcut`）、動的ルーティングプロトコル（EIGRP、OSPF、BGP）の次跳（Next-hop）処理とスプリットホライズン制御、IKEv1/IKEv2 ＋ IPsec トンネルの保護、NAT 回避、およびトラブルシューティング能力が深層レベルで問われます。

---

## 📘 概要

* **機能概要**:
  * **Dual-Hub DMVPN 構築モデル**:
    * **Dual-Hub Single-Cloud**: 全 Hub と Spoke が同一の IP サブネット（1つの Tunnel インターフェイス）および同一の mGRE トンネル空間を共有する構成。単一の mGRE ネットワーク上で NHRP サーバを冗長化し、ルーティングプロトコル（EIGRP / OSPF / BGP）のメトリックまたはアクティブ / スタンバイ制御によりハブ冗長を実現する。
    * **Dual-Hub Dual-Cloud**: 各 Hub が独立した IP サブネットおよび mGRE トンネル空間（Tunnel 0 用 Hub1、Tunnel 1 用 Hub2）を所有する構成。Spoke 側は 2 つの Tunnel インターフェイスを保持し、完全に物理・論理的に分離された 2 つの DMVPN クラウドへ並行接続する。高い対障害性と明確なトラフィック分離を提供できる。
  * **DMVPN フェーズ別制御メカニズム**:
    * **Phase 1**: 全トラフィック（Spoke-to-Spoke 含む）が常に Hub を経由する構成。Spoke 側は P2P GRE（または mGRE）を用い、動的 Spoke-to-Spoke トンネルは形成されない。
    * **Phase 2**: Spoke 間で直接 Spoke-to-Spoke IPsec トンネルを動的に確立。ルーティングプロトコルにおいて Hub が Next-Hop を変更せず（No Next-Hop-Self）、元送信元 Spoke の IP を保持する。全 Spoke でサブネットマスクが一致した Flat な L2/L3 アドレス空間が必要。
    * **Phase 3**: NHRP Redirect (`ip nhrp redirect`) と NHRP Shortcut (`ip nhrp shortcut`) を導入した最も洗練されたアーキテクチャ。ルーティングテーブル上は Hub 宛の集約ルート（Summary Route）やデフォルトルートで動作しつつ、データパケット着信時に Hub が NHRP Redirect を返送することで、Spoke が動的に最適な Spoke-to-Spoke パスを解決・ルーティングテーブルを書き換える（CEF Shortcut）。
* **利用目的**:
  * メイン Hub アプライアンスや上流 ISP 回線の障害時に、Spoke 拠点の通信を数秒以内にサブ Hub へ無瞬断（または最小限の切り替え時間）でフェールオーバーさせる。
  * Spoke-to-Spoke 通信における Hub の集中負荷（CPU・帯域ボトルネック）を解消し、マルチハブ構成でのトラフィックロードバランシングを実現する。
* **どのような場面で利用するか**:
  * 数百〜数千拠点を持つエンタープライズ WAN（金融機関、多店舗展開、グローバル拠点）。
  * 異なる ISP 回線（例: 専有線 + 商用ブロードバンド）を用いたマルチホーム WAN エッジ。

---

## 🔑 要点

| 項目 | Dual-Hub Single-Cloud | Dual-Hub Dual-Cloud |
| :--- | :--- | :--- |
| **Tunnel インターフェイス数** | Hub: 1, Spoke: **1** | Hub: 各 1, Spoke: **2 (Tunnel 0, Tunnel 1)** |
| **IP アドレスサブネット** | 全拠点同一の単一 IPv4/IPv6 サブネット | トンネルごとに独立した 2 つの IP サブネット |
| **NHRP サーバーアサイン** | Spoke 上で 1 つの Tunnel 内に複数の NHS を設定 | Spoke 上で Tunnel 0 ➔ Hub 1, Tunnel 1 ➔ Hub 2 |
| **冗長化・フェールオーバー** | ルーティングプロトコル（EIGRP / OSPF / BGP）に完全依存 | トンネル単位のルーティング優先度（Metric/AD）またはロードバランス |
| **設定の簡素性** | スポーク側のコンフィグが極めてシンプル | インターフェイス数が増えるが制御と分離が明確 |
| **障害分離性（Fault Domain）** | NHRP データベース障害が全拠点に波及するリスクあり | 1つのクラウド障害が他方のクラウドに影響しない |
| **推奨 DMVPN Phase** | Phase 3 (`ip nhrp redirect` / `shortcut`) | Phase 3 (`ip nhrp redirect` / `shortcut`) |

---

## 🏗 動作原理

### 1. Dual-Hub Dual-Cloud DMVPN Phase 3 通信アーキテクチャ

Dual-Cloud 構成において、Spoke1 が Spoke2 宛ての通信を開始した際、Hub1 経由で初期パケットが転送されると同時に、Hub1 から NHRP Redirect が返送され、動的 Spoke-to-Spoke トンネル（Direct Tunnel）が確立されるフロー：

```text
[ Spoke 1 ] ───(1) Initial Data Packet (to Spoke 2) ───► [ Hub 1 (Primary NHS) ]
    │                                                            │
    │ ◄─── (2) NHRP Redirect Message ────────────────────────────┤ (Forwards Data to Spoke 2)
    │                                                            │
    ├───(3) NHRP Resolution Request (via Hub 1) ─────────────────┼───► [ Spoke 2 ]
    │                                                            │        │
    │ ◄─── (4) NHRP Resolution Reply (Direct) ───────────────────┴────────┘
    │
    └───(5) Direct IPsec Tunnel Established ───► [ Spoke 2 ] (Bypasses both Hubs)
```

---

### 2. Dual-Hub Single-Cloud DMVPN における NHRP 登録とルーティングモデル

1 つの Tunnel 空間（例: `10.254.0.0/24`）に 2 台の Hub (`10.254.0.1`, `10.254.0.2`) が存在し、Spoke が両方の Hub へ同時に NHRP 登録（NHS 冗長）を行うモデル：

```text
                               ┌───► [ Hub 1 (NHS 1: 10.254.0.1) ] (Primary Route)
                               │
[ Spoke 1 (Tunnel 0) ] ────────┤
  (IP: 10.254.0.11)            │
                               └───► [ Hub 2 (NHS 2: 10.254.0.2) ] (Secondary Route)

- NHRP Mapping (Spoke 1):
  ip nhrp nhs 10.254.0.1 nbma <Hub1_Public_IP>
  ip nhrp nhs 10.254.0.2 nbma <Hub2_Public_IP>
```

---

## ⚙ 動作シーケンス

1. **初期フェーズ：NHRP Registration Sequence (NHS 登録)**:
   * Spoke が起動すると、設定された `ip nhrp nhs` 指示に基づき、Hub 1（NHS 1）および Hub 2（NHS 2）宛てに **NHRP Registration Request** パケット（NAT Keepalive 含む）を同時送信。
   * 各 Hub は自らの NHRP キャッシュテーブルに Spoke のトンネル IP と NBMA（Public IP）アドレスの紐付けを保存し、**NHRP Registration Reply** を返送。
2. **ルーティング確立フェーズ (Routing Protocol Adjacency)**:
   * **EIGRP**: Spoke は Hub 1 および Hub 2 と個別または共通トンネル経由で EIGRP ネイバーを確立。Hub 側で `no ip split-horizon eigrp` および `no ip next-hop-self eigrp` (Phase 2) またはデフォルト/サマリールート配信 (Phase 3) を送信。
   * **OSPF**: Network Type が `point-to-multipoint`（Phase 3 推奨）または `non-broadcast`（Phase 1/2）で動作。Hub が DR/BDR を保持。
   * **BGP**: eBGP / iBGP (Route Reflector 構成) にて、`listen range` を用いた動的ネイバー構成。
3. **データ通信 ＆ NHRP Redirect 発動フェーズ (Phase 3)**:
   * Spoke 1 から Spoke 2 宛ての最初のパケットが発信。Spoke 1 のルーティングテーブルでは Hub 宛ての集約ルート（例: `10.0.0.0/8` via Hub 1）にマッチするため、パケットを Hub 1 へ送信。
   * Hub 1 はパケットを受信し、同一の Tunnel インターフェイスから Spoke 2 へパケットをルーティング（In-and-Out on same interface）。
   * Hub 1 はこの「同一インターフェイスでの折り返し」を検知し、Spoke 1 に対して **NHRP Redirect (Indication)** メッセージを送信。
4. **Shortcut 解決 ＆ ダイレクトトンネル確立フェーズ**:
   * NHRP Redirect を受信した Spoke 1 は、Spoke 2 の IP アドレスに対する **NHRP Resolution Request** を構築し、Hub 経由で Spoke 2 へ送信。
   * Spoke 2 は Resolution Request を受信すると、自らの NBMA アドレスを含めた **NHRP Resolution Reply** を Spoke 1 へ直接（Direct）返送。
   * Spoke 1 と Spoke 2 の間で IKE/IPsec 鍵交換が行われ、動的な Spoke-to-Spoke トンネルが形成される。
   * Spoke 1 の CEF (Cisco Express Forwarding) テーブルに **NHRP Shortcut コマンド由来の超高速特異ルート（`H` フラグ付きルート）** が動的注入され、以降のデータは Hub をバイパスして直接通信される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Dual-Hub Single-Cloud vs Dual-Cloud の設定・構成差分**:
   * **Single-Cloud**: Spoke 側で 1 つの `interface Tunnel0` 内に `ip nhrp nhs <Hub1_IP> nbma <Hub1_Public_IP>` と `ip nhrp nhs <Hub2_IP> nbma <Hub2_Public_IP>` の 2 行を記述。
   * **Dual-Cloud**: Spoke 側で `interface Tunnel0` (Hub 1 宛て) と `interface Tunnel1` (Hub 2 宛て) の 2 つのトンネルインターフェイスを作成し、異なる IP サブネットをアサインする。
2. **DMVPN Phase 3 固有コマンドの定義漏れ（ラボ試験で最も多い失点理由）**:
   * **Hub 側必須コマンド**: **`ip nhrp redirect`**（インターフェイス上でパケット折り返し時に Redirect メッセージを発行させる）。
   * **Spoke 側必須コマンド**: **`ip nhrp shortcut`**（Hub からの Redirect メッセージを受け取り、CEF テーブルに動的ショートカットルートを挿入させる）。
   * ※注意: Hub 側で `ip nhrp redirect` を忘れると Spoke-to-Spoke トンネルが一切形成されず、全トラフィックが Hub 経由（Phase 1 動作）に固定されてしまいます。
3. **動的ルーティングプロトコルの要件とトラブル制御**:
   * **EIGRP**:
     * Hub 側で `no ip split-horizon eigrp <AS>` を設定（Single-Cloud で必須）。
     * Phase 2 では Hub 側で `no ip next-hop-self eigrp <AS>` を設定し、Spoke 間のパケットが相手 Spoke の Next-Hop IP を参照できるようにする。
     * Phase 3 では Hub 側で `ip next-hop-self eigrp <AS>`（デフォルト）または集約ルート (`ip summary-address eigrp`) を配信しても、NHRP Redirect/Shortcut によって自動的に Spoke-to-Spoke パスが構築される。
   * **OSPF**:
     * Phase 1/2 では `ip ospf network non-broadcast` または `broadcast` を使用し、Hub の OSPF 優先度（Priority）を高く（例: 255）、Spoke 側を 0 (`ip ospf priority 0`) に設定して Hub を DR/BDR に固定する。
     * Phase 3 では **`ip ospf network point-to-multipoint`** を使用するのがベストプラクティス。DR/BDR の選定が不要になり、サマリールート配信との親和性が極めて高い。
   * **BGP**:
     * Hub 側で Route Reflector (`neighbor <GROUP> route-reflector-client`) を設定。
     * 動的ネイバー (`bgp listen range <SUBNET> peer-group <GROUP>`) を活用して Spoke の個別設定を排除。
4. **IPsec Profile ＆ MTU / MSS の適切なチューニング**:
   * GRE ＋ IPsec カプセル化によるオーバーヘッド（約 52〜72 バイト）を考慮し、Tunnel インターフェイスで **`ip mtu 1400`** および **`ip tcp adjust-mss 1360`** を設定。
   * これらを怠ると、大きなデータパケットや HTTP 通信が PMTU ドロップまたはブラックホール化します。

---

## 🛠 設定方法

### 1. Dual-Hub Single-Cloud DMVPN Phase 3 設定例 (Cisco IOS-XE / Router)

```bash
! ==========================================================
! Hub 1 (Primary NHS) Configuration
! ==========================================================
interface Loopback0
 ip address 10.254.0.1 255.255.255.255

crypto ikev2 keyring KR_DMVPN
 peer SPOKES
  address 0.0.0.0 0.0.0.0
  pre-shared-key cisco123

crypto ikev2 profile IK2_PROF
 match identity remote any
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_DMVPN

crypto ipsec profile IPSEC_PROF
 set ikev2-profile IK2_PROF

interface Tunnel0
 ip address 10.254.0.1 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication DMVPN_KEY
 ip nhrp map multicast dynamic
 ip nhrp network-id 100
 ip nhrp redirect                          ! Phase 3 必須: Redirect 発行
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC_PROF

router eigrp 100
 network 10.254.0.0 0.0.255.255
 no ip split-horizon eigrp 100             ! Split-Horizon 無効化

! ==========================================================
! Hub 2 (Secondary NHS) Configuration
! ==========================================================
interface Loopback0
 ip address 10.254.0.2 255.255.255.255

crypto ikev2 keyring KR_DMVPN
 peer SPOKES
  address 0.0.0.0 0.0.0.0
  pre-shared-key cisco123

crypto ikev2 profile IK2_PROF
 match identity remote any
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_DMVPN

crypto ipsec profile IPSEC_PROF
 set ikev2-profile IK2_PROF

interface Tunnel0
 ip address 10.254.0.2 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication DMVPN_KEY
 ip nhrp map multicast dynamic
 ip nhrp network-id 100
 ip nhrp redirect                          ! Phase 3 必須
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC_PROF

router eigrp 100
 network 10.254.0.0 0.0.255.255
 no ip split-horizon eigrp 100

! ==========================================================
! Spoke 1 Configuration (Dual NHS in Single Tunnel)
! ==========================================================
interface Loopback0
 ip address 10.254.0.11 255.255.255.255

crypto ikev2 keyring KR_DMVPN
 peer HUBS
  address 0.0.0.0 0.0.0.0
  pre-shared-key cisco123

crypto ikev2 profile IK2_PROF
 match identity remote any
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_DMVPN

crypto ipsec profile IPSEC_PROF
 set ikev2-profile IK2_PROF

interface Tunnel0
 ip address 10.254.0.11 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication DMVPN_KEY
 ip nhrp network-id 100
 ! Hub 1 NHS Mapping
 ip nhrp nhs 10.254.0.1 nbma 198.51.100.1 mult
 ! Hub 2 NHS Mapping
 ip nhrp nhs 10.254.0.2 nbma 198.51.100.2 mult
 ip nhrp shortcut                          ! Phase 3 必須: Shortcut 受信・CEF 挿入
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC_PROF

router eigrp 100
 network 10.254.0.0 0.0.255.255
```

---

### 2. Dual-Hub Dual-Cloud DMVPN Phase 3 設定例 (Cisco IOS-XE / Router)

```bash
! ==========================================================
! Spoke 1 Configuration (Dual Tunnel / Dual Cloud)
! ==========================================================
! --- Cloud 1 (Hub 1 接続用 Tunnel 0) ---
interface Tunnel0
 ip address 10.254.1.11 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication KEY_CLOUD1
 ip nhrp network-id 101
 ip nhrp nhs 10.254.1.1 nbma 198.51.100.1 mult
 ip nhrp shortcut
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC_PROF

! --- Cloud 2 (Hub 2 接続用 Tunnel 1) ---
interface Tunnel1
 ip address 10.254.2.11 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication KEY_CLOUD2
 ip nhrp network-id 102
 ip nhrp nhs 10.254.2.1 nbma 198.51.100.2 mult
 ip nhrp shortcut
 tunnel source GigabitEthernet2
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC_PROF

router eigrp 100
 network 10.254.1.0 0.0.0.255
 network 10.254.2.0 0.0.0.255
```

---

## 🔍 検証コマンド

| 目的 | コマンド (Cisco IOS / IOS-XE) | 表示確認ポイント |
| :--- | :--- | :--- |
| **NHRP キャッシュテーブル・マッピング確認** | <code>show ip nhrp</code> | `Flags: static, dynamic, router, redirected` の確認 |
| **NHRP トラフィック・統計カウンター確認** | <code>show ip nhrp traffic</code> | Registration/Resolution Request/Reply, Redirect 送受信数 |
| **NHRP ショートカット（CEF）ルート確認** | <code>show ip route nhrp</code> / <code>show ip route</code> | コード `H` (NHRP generated dynamic route) の表示確認 |
| **Crypto Session 動作状態確認** | <code>show crypto session detail</code> | `Session status: UP-ACTIVE`, Peer NBMA IP, Encaps/Decaps カウンター |
| **IKEv2 SA ネゴシエーション確認** | <code>show crypto ikev2 sa</code> | `Status: READY`, Local/Remote SPI, ESTABLISHED 状態 |
| **IPsec SA ネゴシエーション確認** | <code>show crypto ipsec sa</code> | `#pkts encrypt/decrypt`, SPI 値、Transform Set 一致確認 |
| **NHRP パケットリアルタイムデバッグ** | <code>debug nhrp packet</code> / <code>debug nhrp condition</code> | Redirect メッセージ発行、Resolution Request/Reply シグナリング追跡 |
| **IKEv2 / IPsec デバッグ** | <code>debug crypto ikev2 packet</code> / <code>debug crypto ipsec</code> | 鍵交換失敗原因、Transform Set 不一致、認証失敗ログの取得 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **Spoke が Hub へ NHRP 登録できず、`show ip nhrp` に Hub が表示されない** | `ip nhrp nhs` の NBMA IP 誤り、IPsec 認証鍵 (IKEv2 Pre-shared Key / Certificate) の不一致、または UDP 500/4500 ブロック | <code>show crypto ikev2 sa</code><br><code>show ip nhrp</code><br><code>debug nhrp packet</code> | PSK 鍵および `ip nhrp nhs` / `map` 設定を再確認。アウターネットワーク上で UDP 500/4500 が疎通可能か確認。 |
| **Spoke-to-Spoke 通信は可能だが、常に Hub を経由（Phase 1 動作）してしまい直トンネルが張られない** | Hub 側で **`ip nhrp redirect`** が未設定、または Spoke 側で **`ip nhrp shortcut`** が未設定 | <code>show ip nhrp</code><br><code>show ip route</code> | Hub インターフェイスに `ip nhrp redirect` を追加し、Spoke 側に `ip nhrp shortcut` を追加する。 |
| **Spoke-to-Spoke トンネルの成立時に NHRP 解消要求（Resolution Request）が失敗・タイムアウトする** | 途中のアクセスリスト（ACL）やファイアウォールで NHRP (IP Protocol 54) や GRE (IP Protocol 47) がドロップされている | <code>debug nhrp packet</code><br><code>show ip nhrp traffic</code> | インターフェイス ACL で GRE (47) および ESP (50) / UDP (500/4500) を明示的に許可（Permit）する。 |
| **EIGRP ネイバーが Hub-Spoke 間で頻繁にフラッピング（Flap）する** | Holdtime 内に EIGRP Hello パケットが届かない、または IP MTU 不一致による Hello/Update パケットの断片化・ドロップ | <code>show eigrp neighbors</code><br><code>show interface tunnel 0</code> | Tunnel インターフェイス上で `ip mtu 1400` および `ip tcp adjust-mss 1360` を設定。EIGRP コマンド `ip hello-interval` / `hold-time` を適切に調整。 |
| **DMVPN Phase 3 で Spoke 側ルーティングテーブルに `H` ルート（Shortcut）が注入されない** | Spoke 側で `ip nhrp shortcut` が無効、または CEF (Cisco Express Forwarding) がグローバルで無効化されている | <code>show ip route nhrp</code><br><code>show ip cef</code> | グローバルで `ip cef` を有効化し、Tunnel 上で `ip nhrp shortcut` を投入する。 |
| **OSPF ネイバーが Hub-Spoke 間で `FULL` にならず `ATTEMPT` や `INIT` で停止する** | OSPF Network Type の不一致（例: Hub が `broadcast` で Spoke が `point-to-point`）、または OSPF Priority 設定ミスによる DR 誤選定 | <code>show ip ospf neighbor</code><br><code>show ip ospf interface</code> | 全拠点で OSPF Network Type を統一（Phase 3 では `point-to-multipoint` 推奨）。Hub の Priority を 255、Spoke を 0 に設定。 |

---

## ⚠ 制限事項

* **FMC / FDM 管理下 FTD における DMVPN サポート非対応**:
  * Cisco FTD (Firepower Threat Defense) ソフトウェアは、現行バージョンにおいて **DMVPN (mGRE + NHRP) 機能をネイティブサポートしていません**。DMVPN の Hub および Spoke 機能は Cisco IOS / IOS-XE ルーター（ISR, ASR, Catalyst 8000, CSR1000v）で構築する必要があります。
* **ASA における DMVPN 非対応**:
  * Cisco ASA アプライアンスは mGRE および NHRP プロトコルをサポートしないため、DMVPN Hub / Spoke として配置することはできません（※ASA は Site-to-Site IPsec L2L または AnyConnect RAVPN のみ対応）。
* **Phase 2 と Phase 3 の混在制限**:
  * 同一 Tunnel インターフェイス内で Phase 2 (`ip next-hop-self` なし EIGRP) と Phase 3 (`ip nhrp redirect/shortcut`) を混在させると、NHRP キャッシュおよびルーティングテーブルで競合（Conflict）が発生します。ネットワーク全体で同一フェーズ仕様へ統一することが推奨されます。

---

## 🔄 他技術との関連

* **IKEv2 / IPsec**: DMVPN の mGRE トンネル保護に標準利用。FlexVPN と同様の高度なプロファイル・キーリング構造を活用可能。
* **EIGRP / OSPF / BGP**: DMVPN 上でのオーバーレイ動的ルーティング。Dual-Hub 環境でのパス選定（Equal/Unequal Cost Load Balancing）、スプリットホライズン制御、Route Reflector 設計の中核。
* **NAT / Front-Door VRF (FVRF)**: アンダーレイ（パブリック側）とオーバーレイ（プライベート側）のルーティングテーブルを分離する VRF-Aware DMVPN 設計。
* **QoS (Per-Tunnel QoS)**: DMVPN Hub 側で各 Spoke への帯域制限（Rate Limiting）や優先制御を行う機能（`shape average`, `bandwidth` 属性）。

---

## 🧩 比較表

### 1. Dual-Hub Single-Cloud vs Dual-Hub Dual-Cloud

| 比較項目 | Dual-Hub Single-Cloud | Dual-Hub Dual-Cloud |
| :--- | :--- | :--- |
| **Hub / Spoke 論理構造** | 1 サブネット / 1 mGRE Tunnel | 2 サブネット / 2 mGRE Tunnels |
| **障害分離性 (Fault Domain)**| 中（1つの NHRP キャッシュトラブルが全拠点に影響）| **高**（クラウドごとに完全分離） |
| **設定・運用負荷** | **低**（Spoke の設定が極めてシンプル） | やや高（Spoke に 2 つの Tunnel 定義が必要） |
| **パス制御 (Traffic Engineering)**| ルーティングプロトコルのメトリック/コストで制御 | **インターフェイス単位のルーティング優先度で明快に制御** |
| **推奨用途** | 構成の簡素性を優先する中規模 DMVPN | 最高の可用性と障害分離を求める大規模エンタープライズ |

### 2. DMVPN Phase 1 vs Phase 2 vs Phase 3

| 比較項目 | Phase 1 | Phase 2 | Phase 3 |
| :--- | :--- | :--- | :--- |
| **Spoke-to-Spoke トンネル** | 非対応 (常に Hub 経由) | **対応** (動的確立) | **対応** (動的確立) |
| **Hub 負荷** | 極めて高い (全データ中継) | 低い (制御のみ) | **極めて低い** (制御のみ + CEF 変換) |
| **ルーティング設計** | Hub で Next-Hop-Self & サマリー可 | **Hub で Next-Hop 保持必須 & サマリー不可** | **Hub で Next-Hop-Self & サマリー配信可能** |
| **推奨 OSPF Network Type** | point-to-multipoint | non-broadcast / broadcast | **point-to-multipoint** |
| **NHRP 核心コマンド** | `ip nhrp map` | `ip nhrp map` | **`ip nhrp redirect` (Hub) / `shortcut` (Spoke)** |

---

## 💡 ベストプラクティス

1. **DMVPN Phase 3 ＋ EIGRP/OSPF サマリールート配信の標準採用**:
   * 大規模 DMVPN では、Hub から Spoke へサマリールート（例: `10.0.0.0/8` またはデフォルトルート `0.0.0.0/0`）のみを配信します。Phase 3 の `ip nhrp redirect` / `shortcut` 機構により、Spoke 側はサマリールートしか保持していなくても、対向 Spoke への通信発生時に動的に特異ショートカットルート (`H` コード) を CEF に生成するため、ルーティングテーブルの肥大化を完璧に防止できます。
2. **Front-Door VRF (FVRF) によるアンダーレイ・オーバーレイの完全分離**:
   * 物理インターフェイス（WAN 側）を `ip vrf forwarding FVRF` にアサインし、Tunnel インターフェイスを Global VRF（または Data VRF）で動作させます。これにより、デフォルトルートの競合やルーティングループを根本的に排除します。
3. **PMTU ドロップを防ぐ MTU / MSS の均一適用**:
   * 全 Hub および Spoke の Tunnel インターフェイスで一律 `ip mtu 1400` および `ip tcp adjust-mss 1360` を設定します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Dual-Hub Single-Cloud DMVPN Phase 3 (IKEv2 ＋ EIGRP)

#### 問題
Hub1, Hub2 および Spoke1 間で Single-Cloud DMVPN Phase 3 構成を構築せよ。IKEv2 事前共有鍵 `cisco123` を使用し、EIGRP AS 100 で全拠点を接続せよ。

#### 要件
* Single-Cloud トンネルサブネット: `10.254.0.0/24`
* Hub1: `10.254.0.1`, Hub2: `10.254.0.2`, Spoke1: `10.254.0.11`
* Phase 3 Redirect / Shortcut を有効化すること。

#### 設定例

```bash
! ===== Hub 1 =====
crypto ikev2 keyring KR_DMVPN
 peer SPOKES
  address 0.0.0.0 0.0.0.0
  pre-shared-key cisco123

crypto ikev2 profile IK2_PROF
 match identity remote any
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_DMVPN

crypto ipsec profile IPSEC_PROF
 set ikev2-profile IK2_PROF

interface Tunnel0
 ip address 10.254.0.1 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 100
 ip nhrp redirect
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC_PROF

router eigrp 100
 network 10.254.0.0 0.0.255.255
 no ip split-horizon eigrp 100

! ===== Hub 2 =====
crypto ikev2 keyring KR_DMVPN
 peer SPOKES
  address 0.0.0.0 0.0.0.0
  pre-shared-key cisco123

crypto ikev2 profile IK2_PROF
 match identity remote any
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_DMVPN

crypto ipsec profile IPSEC_PROF
 set ikev2-profile IK2_PROF

interface Tunnel0
 ip address 10.254.0.2 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 100
 ip nhrp redirect
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC_PROF

router eigrp 100
 network 10.254.0.0 0.0.255.255
 no ip split-horizon eigrp 100

! ===== Spoke 1 =====
crypto ikev2 keyring KR_DMVPN
 peer HUBS
  address 0.0.0.0 0.0.0.0
  pre-shared-key cisco123

crypto ikev2 profile IK2_PROF
 match identity remote any
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_DMVPN

crypto ipsec profile IPSEC_PROF
 set ikev2-profile IK2_PROF

interface Tunnel0
 ip address 10.254.0.11 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 100
 ip nhrp nhs 10.254.0.1 nbma 198.51.100.1 mult
 ip nhrp nhs 10.254.0.2 nbma 198.51.100.2 mult
 ip nhrp shortcut
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC_PROF

router eigrp 100
 network 10.254.0.0 0.0.255.255
```

#### 検証方法
* Spoke1 で `show ip nhrp` を実行し、`10.254.0.1` および `10.254.0.2` が `static` マッピングとして登録され、`show crypto session` で両 Hub との IPsec セッションが UP になっていることを確認。

---

### 例2: Dual-Hub Dual-Cloud DMVPN Phase 3 (Dual Tunnel インターフェイス構成)

#### 問題
Hub1 (Cloud 1: `10.254.1.0/24`) および Hub2 (Cloud 2: `10.254.2.0/24`) に対して Spoke1 に 2 つの Tunnel インターフェイス (`Tunnel0`, `Tunnel1`) を作成し、Dual-Cloud DMVPN 構成を構築せよ。

#### 要件
* Cloud 1 Network ID: 101, Cloud 2 Network ID: 102
* 各 Tunnel で `ip nhrp shortcut` を有効化すること。

#### 設定例

```bash
! ===== Spoke 1 =====
interface Tunnel0
 description DMVPN Cloud 1 (Hub 1)
 ip address 10.254.1.11 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 101
 ip nhrp nhs 10.254.1.1 nbma 198.51.100.1 mult
 ip nhrp shortcut
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC_PROF

interface Tunnel1
 description DMVPN Cloud 2 (Hub 2)
 ip address 10.254.2.11 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 102
 ip nhrp nhs 10.254.2.1 nbma 198.51.100.2 mult
 ip nhrp shortcut
 tunnel source GigabitEthernet2
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC_PROF

router eigrp 100
 network 10.254.1.0 0.0.0.255
 network 10.254.2.0 0.0.0.255
```

#### 検証方法
* `show ip nhrp` を実行し、Tunnel0 および Tunnel1 でそれぞれの Hub に対する NHS マッピングが確立されていることを確認。

---

### 例3: Dual-Hub DMVPN Phase 3 ＋ OSPF Point-to-Multipoint 構成

#### 問題
Dual-Hub DMVPN 上で OSPF エリア 0 を構成せよ。DR/BDR 選定のトラブルを回避するため、OSPF Network Type に `point-to-multipoint` を使用せよ。

#### 要件
* Hub 1, Hub 2, Spoke 1 の Tunnel0 上で `ip ospf network point-to-multipoint` を設定すること。

#### 設定例

```bash
! ===== Hub 1 / Hub 2 / Spoke 1 Tunnel Interface =====
interface Tunnel0
 ip ospf network point-to-multipoint
 ip ospf 1 area 0

router ospf 1
 router-id 10.254.0.1                       ! 各ルーター固有の ID
```

#### 検証方法
* `show ip ospf neighbor` を確認し、DR/BDR 表示がハイフン (`-`) となり、全ネイバーが `FULL` ステータスで安定確立されていることを確認。

---

### 例4: Dual-Hub DMVPN ＋ BGP Route Reflector ＆ 動的ネイバー構成

#### 問題
Hub 1 および Hub 2 を BGP Route Reflector として構成し、Spoke 群からの iBGP 接続（AS 65000）を `bgp listen range` で動的収容せよ。

#### 要件
* トンネルサブネット: `10.254.0.0/24`
* Hub 側で `bgp listen range 10.254.0.0/24 peer-group SPOKES` を使用すること。

#### 設定例

```bash
! ===== Hub 1 (Route Reflector) =====
router bgp 65000
 bgp router-id 10.254.0.1
 bgp listen range 10.254.0.0/24 peer-group SPOKES
 neighbor SPOKES peer-group
 neighbor SPOKES remote-as 65000
 neighbor SPOKES route-reflector-client
 neighbor SPOKES cluster-id 10.254.0.1
 address-family ipv4
  neighbor SPOKES activate
 exit-address-family

! ===== Spoke 1 =====
router bgp 65000
 bgp router-id 10.254.0.11
 neighbor 10.254.0.1 remote-as 65000
 neighbor 10.254.0.2 remote-as 65000
 address-family ipv4
  neighbor 10.254.0.1 activate
  neighbor 10.254.0.2 activate
 exit-address-family
```

#### 検証方法
* Hub 1 で `show bgp ipv4 unicast summary` を実行し、Spoke 1 からの動的 BGP ネイバーが確立されていることを確認。

---

### 例5: Front-Door VRF (FVRF) を用いた VRF-Aware DMVPN 構成

#### 問題
WAN 側インターフェイス `Gi1` を `VRF_WAN` に割り当て、DMVPN トンネル `Tunnel0`（Global 領域）を構築する VRF-Aware DMVPN を構成せよ。

#### 要件
* `tunnel vrf VRF_WAN` コマンドを使用すること。

#### 設定例

```bash
! ===== Spoke 1 =====
vrf definition VRF_WAN
 address-family ipv4
 exit-address-family

interface GigabitEthernet1
 vrf forwarding VRF_WAN
 ip address 198.51.100.11 255.255.255.0

interface Tunnel0
 ip address 10.254.0.11 255.255.255.0
 tunnel source GigabitEthernet1
 tunnel vrf VRF_WAN                         ! FVRF アサイン
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC_PROF

ip route vrf VRF_WAN 0.0.0.0 0.0.0.0 198.51.100.254
```

#### 検証方法
* `show ip route vrf VRF_WAN` でデフォルトルートを確認し、`show crypto session` で IPsec セッションが `VRF: VRF_WAN` 上で正しく確立していることを確認。

---

### 例6: EIGRP メトリックチューニングによる Dual-Hub アクティブ / スタンバイ制御

#### 問題
Dual-Hub Single-Cloud DMVPN において、平常時はすべての外部トラフィックを Hub 1 経由とし、Hub 1 障害時のみ Hub 2 へ迂回させるよう、Hub 2 側で EIGRP ディレイ（Delay）を大きく調整せよ。

#### 要件
* Hub 2 の Tunnel0 で `delay 200` を設定すること（Hub 1 デフォルト `delay 10`）。

#### 設定例

```bash
! ===== Hub 2 =====
interface Tunnel0
 delay 200                                 ! メトリックを増大させてバックアップ化
```

#### 検証方法
* Spoke1 で `show ip route eigrp` を実行し、集約ルート/デフォルトルートの Next-Hop が Hub 1 (`10.254.0.1`) のみとなっていることを確認。

---

### 例7: DMVPN Phase 3 における EIGRP サマリールート配信と NHRP Shortcut 検証

#### 問題
Hub 1 から Spoke 群へ全社集約ルート `10.0.0.0/8` を配信せよ。Spoke1 から Spoke2 (`10.2.0.50`) への Ping 発生時に動的ショートカットルート (`H` コード) が注入されることを確認せよ。

#### 要件
* Hub 1 側で `ip summary-address eigrp 100 10.0.0.0 255.0.0.0` を投入すること。

#### 設定例

```bash
! ===== Hub 1 =====
interface Tunnel0
 ip summary-address eigrp 100 10.0.0.0 255.0.0.0
```

#### 検証方法
* Spoke1 から Spoke2 宛に Ping を打ち、`show ip route nhrp` を実行して `H 10.2.0.0/24 [250/1] via 10.254.0.12, Tunnel0` という動的ショートカットルートが生成されることを確認。

---

### 例8: PKI 証明書認証を用いた DMVPN IPsec トンネルの保護

#### 問題
事前共有鍵の代わりに Cisco IOS Root CA から取得した X.509 証明書を用いて IKEv2 DMVPN トンネルを保護せよ。

#### 要件
* `crypto ikev2 profile` 内で `authentication local rsa-sig` および `authentication remote rsa-sig` を定義すること。

#### 設定例

```bash
! ===== All Routers (Hubs & Spokes) =====
crypto pki trustpoint TP_CA
 subject-name CN=ROUTER-DMVPN, O=Cisco
 revocation-check crl

crypto ikev2 profile IK2_CERT_PROF
 match identity remote any
 authentication local rsa-sig
 authentication remote rsa-sig
 pki trustpoint TP_CA

crypto ipsec profile IPSEC_CERT_PROF
 set ikev2-profile IK2_CERT_PROF
```

#### 検証方法
* `show crypto ikev2 sa` を確認し、`Auth-Method: RSA-SIG` で SA が ESTABLISHED 状態になっていることを確認。

---

### 例9: DMVPN Hub 側での Per-Tunnel QoS (Shape Average) の適用

#### 問題
Hub 1 において、全 Spoke 宛ての最大パケット送出帯域を 2 Mbps に制限（Policed / Shaped）する Per-Tunnel QoS を構成せよ。

#### 要件
* `policy-map PM_DMVPN_SHAPE` を定義し、Tunnel インターフェイスへアサインすること。

#### 設定例

```bash
! ===== Hub 1 =====
policy-map PM_DMVPN_SHAPE
 class class-default
  shape average 2000000

interface Tunnel0
 service-policy output PM_DMVPN_SHAPE
```

#### 検証方法
* `show policy-map interface Tunnel0` を実行し、トンネル出力トラフィックが 2 Mbps にシェーピングされていることを確認。

---

### 例10: DMVPN Phase 3 トラブルシューティング（NHRP Redirect 発行失敗の修正）

#### 問題
Spoke1 と Spoke2 の間で直接通信が発生した際、通信は可能であるものの常に Hub1 を経由してしまい、Spoke-to-Spoke トンネルが形成されない。Hub1 の設定を診断し問題を修正せよ。

#### 要件
* 原因を特定し、必要なコマンドを補充すること。

#### 設定例

```bash
! ===== Hub 1 Diagnostic & Resolution =====
! 診断: "show running-config interface Tunnel0" を確認したところ、"ip nhrp redirect" が欠落していた。

! 修正コマンド投入:
interface Tunnel0
 ip nhrp redirect                          ! パケット折り返し時の Redirect 発行を有効化
```

#### 検証方法
* Spoke1 から Spoke2 へ Ping を打ち、`show ip nhrp` を実行して Spoke2 の NBMA アドレスが `router` / `redirected` フラグ付きで動的作成されることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: Dual-Hub Single-Cloud DMVPN Phase 3 環境において、Hub 1 で以下のコンフィグが投入されている。

```bash
interface Tunnel0
 ip address 10.254.0.1 255.255.255.0
 ip nhrp network-id 100
 ip nhrp redirect
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
```

Spoke 1 (`10.254.0.11`) から Spoke 2 (`10.254.0.12`) 宛てに通信を発生させたが、Spoke 1 上で `show ip route` を確認すると Spoke 2 宛ての `H` ルート（Shortcut）が全く注入されず、常に Hub 1 経由で通信が行われていた。Spoke 1 側で設定が欠落しているコマンドとして最も適切なものはどれか。

* A) `ip nhrp redirect`
* B) `ip nhrp shortcut`
* C) `no ip split-horizon`
* D) `tunnel mode gre multipoint`

**正解**: **B**  
**解説**: DMVPN Phase 3 において、Hub 側には **`ip nhrp redirect`**（パケット折り返し時に Redirect メッセージを発行する）、Spoke 側には **`ip nhrp shortcut`**（Hub からの Redirect メッセージを解析し、CEF テーブルに動的ショートカット `H` ルートを生成・更新する）の双方が設定されている必要があります。Spoke 側で `ip nhrp shortcut` が欠落している場合、Shortcut ルートが生成されず、トラフィックは常に Hub 経由（Phase 1 同等の挙動）となります。

---

### 2. トラブルシュート問題

**問題**: Dual-Hub DMVPN 上で OSPF を動作させたところ、Spoke ルーター上で OSPF ネイバー状態が `FULL` にならず、`INIT/DROTHER` または `EXSTART` で停止した。さらに、`show ip ospf interface` を確認すると Hub が DR になっておらず、特定の Spoke が DR として選定されていた。原因および正解の対処方法はどれか。

* A) DMVPN トンネル上で MTU が不一致であるため、MTU チェックを無効化する。
* B) OSPF Network Type が `broadcast` になっており、Spoke 側で DR 選定が無効化されていない。全 Spoke の Tunnel インターフェイスで `ip ospf priority 0` を投入し、Hub 側の Priority を高める（または OSPF Network Type を `point-to-multipoint` に変更する）。
* C) NHRP の事前共有鍵が不一致であるため、`ip nhrp authentication` を変更する。
* D) OSPF Area 0 が NSSA エリアとして正しく宣言されていないため。

**正解**: **B**  
**解説**: DMVPN の mGRE インターフェイス上で OSPF Broadcast モードを使用する場合、すべての Spoke 相互間で L2 Broadcast / Multicast が直接到達できない（Hub 経由になる）ため、Spoke が DR に選定されると OSPF データベースの同期が破綻します。そのため、全 Spoke で **`ip ospf priority 0`** を設定して DR/BDR 選定を回避するか、DR/BDR 自体が存在しない **`ip ospf network point-to-multipoint`** へ変更することが Cisco のベストプラクティスです。

---

### 3. Design 問題

**問題**: 1,000 拠点を有する大規模エンタープライズネットワークにおいて、Dual-Hub DMVPN アーキテクチャを設計している。Hub ルーターおよび Spoke ルーターの CPU・メモリ負担を最少化しつつ、拠点間の直接（Spoke-to-Spoke）通信と迅速な障害切替を実現したい。最も推奨される設計組み合わせはどれか。

* A) DMVPN Phase 1 ＋ EIGRP（Hub で `next-hop-self` 設定）
* B) DMVPN Phase 2 ＋ OSPF Broadcast（全 Spoke でフラットなルーティングテーブル保持）
* C) **DMVPN Phase 3 ＋ EIGRP（Hub で全社サマリールート配信 ＋ `ip nhrp redirect/shortcut`）**
* D) DMVPN Phase 2 ＋ BGP（全 Spoke 間に Full-Mesh iBGP ピア確立）

**正解**: **C**  
**解説**: **DMVPN Phase 3** では、Hub 側でサマリールート（例: `10.0.0.0/8` や `0.0.0.0/0`）のみを Spoke へ配信することが許可されています。Spoke は小さなルーティングテーブルしか保持しませんが、Spoke-to-Spoke 通信発生時には NHRP Redirect/Shortcut により動的 CEF ルート (`H` ルート) が最小限だけオンデマンド生成されるため、最もスケーラブルで CPU/メモリ消費の少ない優れたアーキテクチャとなります。

---

### 4. 実装問題

**問題**: Dual-Hub Dual-Cloud DMVPN において、アンダーレイ（パブリック WAN 回線）のデフォルトルートと、オーバーレイ（DMVPN トンネル内）の動的ルーティングプロトコルとの間でルーティングループや再配布トラブルが発生するのを根本的に防ぐための設定手法はどれか。

* A) `ip summary-address` コマンドによるルート集約
* B) **Front-Door VRF (FVRF) による物理 WAN インターフェイスと Tunnel インターフェイスの VRF 分離**
* C) `crypto map` の静的ピア定義
* D) OSPF Stub Area の定義

**正解**: **B**  
**解説**: **Front-Door VRF (FVRF)** を使用すると、パブリック WAN（アンダーレイ）のルーティングテーブルと DMVPN トンネル内（オーバーレイ）のルーティングテーブルが物理的・論理的に完全分離されます。これにより、トンネル通過後のパケットが誤ってパブリックデフォルトルートへ折り返されるようなルーティングループを完璧に遮断できます。

---

### 5. コンフィグ読解問題

**問題**: Spoke ルーターで `show ip route` を実行したところ、以下の出力が得られた。

```text
H        10.2.1.0/24 [250/1] via 10.254.0.12, 00:05:22, Tunnel0
```

プレフィックスの先頭にあるコード **`H`** が意味するものとして最も適切なものはどれか。

* A) Host route (静的な /32 ホストルート)
* B) High-availability route (HSRP / VRRP による冗長ルート)
* C) **NHRP generated dynamic shortcut route (DMVPN Phase 3 で生成された動的ショートカットルート)**
* D) Hardware accelerated route (ASIC で処理されるハードウェアルート)

**正解**: **C**  
**解説**: Cisco IOS / IOS-XE のルーティングテーブルにおいて、コード **`H`** は **NHRP によって動的に注入された Phase 3 ショートカットルート (NHRP generated dynamic route)** を表します。

---

## 🔗 参考リソース

* [Cisco Dynamic Multipoint VPN Configuration Guide: DMVPN Phase 3 Design and Implementation](https://www.cisco.com/)
* [Cisco Technical Notes: Troubleshoot DMVPN Phase 3 Redirect and Shortcut Issues](https://www.cisco.com/)
* [Cisco Live: Advanced DMVPN Architecture and Design (BRKSEC-3052)](https://www.ciscolive.com/)
* [Cisco Validated Design (CVD): Enterprise WAN DMVPN Deployment Guide](https://www.cisco.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で Dual-Hub DMVPN が出題された場合の迅速チェックリスト：
    1. **Topology Identification**: Single-Cloud (1 Tunnel subnet) か Dual-Cloud (2 Tunnel subnets) かを指示文から見極める。
    2. **Phase Determination**: 通常は Phase 3。Hub 側で `ip nhrp redirect`、Spoke 側で `ip nhrp shortcut` を即座に投入。
    3. **Routing Integration**:
       - EIGRP: Single-Cloud では Hub 側で `no ip split-horizon eigrp <AS>` が必須。
       - OSPF: `ip ospf network point-to-multipoint` を標準採用。
    4. **MTU / MSS**: `ip mtu 1400` ＆ `ip tcp adjust-mss 1360` を忘れない。
* **図解（DMVPN Phase 3 パケット処理・ショートカット自動生成フロー）**:
  ```text
  [ Spoke 1 ] ──── (1. Initial Packet to Spoke 2) ────► [ Hub 1 ] ────► [ Spoke 2 ]
       │                                                    │
       │ ◄─── (2. NHRP Redirect: "Use direct path!") ────────┘
       │
       ├─── (3. NHRP Resolution Request) ───────────────────────────► [ Spoke 2 ]
       │                                                                  │
       │ ◄─── (4. NHRP Resolution Reply with Public IP) ─────────────────┘
       │
  [ Spoke 1 ] ════ (5. Direct IPsec Tunnel & CEF 'H' Route) ═════════► [ Spoke 2 ]
  ```
* **注意点**:
  * ラボ試験で DMVPN を構成した後は、必ず `show ip nhrp` でマッピング状態（`dynamic` / `redirected`）を確認し、`show ip route` で `H` コードのショートカットルートが正しく注入されていることを実機で実証してください。


## 📝 **補足（Notes）**

*   **学習メモ**: デュアルハブは「設定の重複」ではなく「論理的な冗長」です。スポークのTunnel設定を書く際、2つ目のハブ向けの設定が漏れていないか指差し確認しましょう。
*   **注意点**: ラボ試験では、ハブをシャットダウンした後の「切り替わり時間」も採点基準になることがあります。`ip nhrp holdtime` やルーティングタイマーの短縮指示を見落とさないでください。
