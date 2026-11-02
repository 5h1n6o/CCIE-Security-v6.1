---
layout: default
title: 2.6-TrustSec-SGT-SXP
nav_order: 6
parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.6 Microsegmentation with Cisco TrustSec using SGT and SXP

Cisco **TrustSec** アーキテクチャは、従来の IP アドレスやサブネットに依存した静的セキュリティポリシー（Classic ACL / Firewall Rule）の限界を克服し、ユーザーやデバイスの識別情報に基づく **セキュリティグループタグ（Security Group Tag: SGT）** を用いて高度なマイクロセグメンテーション（Microsegmentation）を実現する次世代セキュリティフレームワークです。

CCIE Security v6.1 Blueprint **2.6 Microsegmentation with Cisco TrustSec using SGT and SXP** では、**Cisco ISE（Identity Services Engine）** をコアとした SGT の割り当て（Ingress Classification）、**Inline Tagging（CMD/L2 Ethernet Frame タギング）** および **SXP (SGT Exchange Protocol)** による SGT 情報のネットワーク伝播（Propagation）、**SGACL (Security Group Access Control List)** や **ASA / FTD での SGT ベースアクセス制御**（Egress Enforcement）、さらにはネットワークスイッチ、ルーター、ASA、FTD (FMC) 間での TrustSec 統合設計とトラブルシューティングが極めて深く評価されます。

---

## 📘 概要

* **機能概要**:
  * **Cisco TrustSec**: ID ベースのネットワーク分離技術。ユーザーの認証結果（802.1X, MAB, PassiveID 等）に基づいて 16 ビットの **SGT（0 〜 65535）** を付与し、ネットワーク全体で同一のロールベースセキュリティポリシーを適用する。
  * **SGT Classification (Ingress)**: ネットワークへの侵入ポイント（スイッチ、WLC、VPN ゲートウェイ等）で、認証プロファイル、IP アドレス、サブネット、VLAN、ドメインなどの条件に基づいて SGT を動的または静的に割り当てる。
  * **SGT Propagation**: SGT 属性を送信元から宛先ポリシー適用ポイントまで透過的に転送する技術。
    * **Inline Tagging**: L2 イーサネットフレームの 802.1Q タグと EtherType の間に 8 ビットの CMD (Cisco MetaData) ヘッダーを挿入し、ハードウェア ASIC レベルで SGT（16bit）を直接カプセル化して対向機器へ伝播する。
    * **SXP (SGT Exchange Protocol)**: ハードウェア Inline Tagging をサポートしないネットワーク機器や L3 ネットワーク（WAN/Internet/レガシースイッチ）を越えて、TCP ポート 6499 上で `IP-to-SGT` のマッピングテーブルを TCP ピ アリング経由で伝播する制御プロトコル。
  * **SGT Enforcement (Egress)**: パケットが宛先サブネットを出力する最終ポイント（Egress Switch, ASA, FTD）において、送信元 SGT（Src SGT）と宛先 SGT（Dst SGT）の組み合わせマトリクスに基づいて **SGACL** や **Firewall Access Control Rule** を適用し、最小権限アクセスの制御（マイクロセグメンテーション）を実行する。
* **利用目的**:
  * IP アドレスの変動（DHCP、VDI、IP 移動、クラウド移行）に影響されない抽象化されたアクセス制御ポリシーの確立。
  * ポート単位・サーバー単位での Intra-VLAN / Inter-VLAN マイクロセグメンテーションの実現。
  * 複雑巨大化した Firewall ACL（数万行の IP ACL）の簡略化と運用コスト削減。
* **どのような場面で利用するか**:
  * エンタープライズ Campus ネットワークでの役割別アクセス制御（従業員、ゲスト、IoT、医療機器、ベンダーの分離）。
  * データセンター内の East-West トラフィックのマイクロセグメンテーション。
  * リモートアクセス VPN（AnyConnect）ユーザーに対する動的な SGT 付与と、社内リソース保護。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **評価対象技術** | **Cisco TrustSec (CTS)**、**Security Group Tag (SGT)**、**SGT Exchange Protocol (SXP)**、**SGACL**、**ASA/FTD SGT Integration** |
| **SGT 構造** | 16 ビット（2 バイト）整数値（`1` 〜 `65535`）。`0`: Unknown SGT, `2`: TrustSec Device, `3`: Unknown / Unassigned |
| **SGT 分類 (Ingress)** | Dynamic (802.1X/MAB/WebAuth/RADIUS Authorization via ISE), Static (IP-SGT, Subnet-SGT, VLAN-SGT, Port-SGT) |
| **SGT 伝播 (Propagation)** | **Inline Tagging** (L2 Frame CMD ヘッダー挿入 / 802.1AE MACsec 統合) / **SXP** (TCP 6499 Peer-to-Peer 接続) |
| **SXP 動作モード** | **Speaker** (マッピング送信側), **Listener** (マッピング受信側), **Both** (相互送受信) |
| **SXP バージョン** | **v1**: IPv4 のみ / **v2**: IPv4 ＋ Subnet-SGT / **v3**: IPv6 対応 / **v4**: Subnet/Range ＋ IPv6 ＋ 大規模メッシュ向上 |
| **Enforcement ポイント** | Egress Switch (SGACL), Cisco ASA (Security Group ACL), Cisco FTD/FMC (Access Control Policy SGT Condition) |
| **管理基盤** | **Cisco ISE** (Centralized TrustSec Matrix & SXP Connection Management), FMC (FTD SGT/SXP Integration) |

---

## 🏗 動作原理

### 1. Cisco TrustSec エンドツーエンド・トラフィックフロー (Ingress ➔ Propagation ➔ Egress)

```text
[ Client A (User) ]
       │ (1. 802.1X Authentication)
       ▼
[ Ingress Switch ] ◄─────── 2. RADIUS Access-Accept (Assigns SGT 10) ──────── [ Cisco ISE ]
       │                                                                            │
       ├── 3. Packet Ingress (Tagged with SGT 10)                                   │
       │                                                                            │
       ├── 4a. Inline Tagging (CMD Header via L2 Trunk) ──► [ Core Network ]         │
       │                                                          │                 │
       └── 4b. SXP Connection (TCP 6499: IP-SGT Mapping) ─────────┼─────────────────┤
                                                                  ▼                 │
                                                        [ Egress Switch ] ◄─────────┘
                                                                  │ (5. Downloads SGACL Matrix)
                                                                  ├── 6. Evaluates (Src SGT 10 ➔ Dst SGT 20)
                                                                  │      Action: DENY
                                                                  ▼
                                                        [ Server B (SGT 20) ] (Packet Dropped)
```

---

### 2. Inline Tagging (L2 CMD Frame) 構造 vs SXP Protocol

```text
[ L2 Ethernet Frame with Inline CMD Tagging ]
┌────────────┬────────────┬─────────────┬──────────┬───────────┬───────────────┬───────┐
│ Dst MAC    │ Src MAC    │ 802.1Q Tag  │ EtherType│ CMD Header│ Payload       │ FCS   │
│ (6 Bytes)  │ (6 Bytes)  │ (VLAN ID)   │ (0x8909) │ (SGT: 10) │ (IP Packet)   │ (4B)  │
└────────────┴────────────┴─────────────┴──────────┴───────────┴───────────────┴───────┘

[ SXP Connection Architecture (TCP Port 6499) ]
[ Ingress Device (SXP Speaker) ] ───────── TCP 6499 ─────────► [ Egress Device (SXP Listener) ]
  (Sends Binding Table: 10.1.1.50 ➔ SGT 10)                    (Stores in IP-SGT Binding DB)
```

---

## ⚙ 動作シーケンス

1. **認証 ＆ SGT 動的アサイン (Ingress Classification)**:
   * クライアント端末が 802.1X / MAB で認証要求を送信。
   * ISE は Authentication Policy および Authorization Policy を評価し、Access-Accept 応答内に **`cisco-av-pair = cts:security-group-tag=000A-10`** 属性を含めてスイッチへ返却。
   * イングレススイッチはポートに端末の IP アドレス (`10.1.1.50`) と SGT (`10`) のマッピングをローカル登録。
2. **SGT 情報の伝播 (Propagation Phase)**:
   * **Inline Tagging 選択時**: スイッチは出力インターフェイスが TrustSec 信頼インターフェイス（`cts manual` / `cts dot1x`）として構成されている場合、L2 イーサネットフレームの 802.1Q タグ直後に EtherType `0x8909` (CMD) を挿入し、SGT `10` をカプセル化して送信。
   * **SXP 選択時**: Inline Tagging を非サポートする L3 ルーターや WAN 経由の場合、SXP Speaker ノードは TCP 6499 上で SXP Binding Update メッセージを発行し、`IP: 10.1.1.50 <-> SGT: 10` のバインディング情報を SXP Listener ノードへ転送。
3. **ポリシー照合 ＆ 制御 (Egress Enforcement Phase)**:
   * エグレススイッチ（または ASA / FTD）にパケットが着信。
   * 受信装置は、パケットの送信元 SGT（Src SGT = `10`）および宛先 IP に割り当てられている宛先 SGT（Dst SGT = `20`）を特定。
   * ISE から事前取得（Environment Data Sync）している **SGACL マトリクス** を参照し、`Src SGT 10` から `Dst SGT 20` に対するルール（例: `deny ip`）をハードウェア TCAM で照合し、パケットを破棄または許可。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **ISE と CTS 機器間の TrustSec 認証・環境データ同期 (Environment Data)**:
   * ネットワーク機器（IOS-XE / ASA / FTD）が ISE と TrustSec 連携を行うには、RADIUS サーバー定義に加え、AAA Server Group で **`cts credentials`** や PAC (Protected Access Credential) の取得・更新が必要です。
   * **`cts refresh environment-data`**: ISE から SGT 名、SGT 値（SGT Table）、および SGACL マトリクスの同期を取得するコマンド。
2. **SXP 接続（Speaker / Listener）と認証 MD5 キーの不一致**:
   * SXP ピアリング（TCP 6499）はデフォルトで TCP MD5 認証をサポートします。両エンドで `password`（または `key-chain`）が不一致の場合、SXP セッションは `ON` 状態にならず `Pending-On` や `Disconnect` となります。
   * **SXP Role**: 誰が Speaker（バインディング保持・提供側）で、誰が Listener（受信・利用側）か、要件通りに割り当てること。
3. **ASA ＆ FTD (FMC) における SGT アクセス制御の実装要件**:
   * **ASA**: SGT を用いた ACL 設定。`access-list ACL_SGT extended permit ip security-group name Sales security-group name Finance`。`security-group` オプションを用いたポリシー記述。
   * **FTD / FMC**: FMC の `Objects > Security Group Tags` で ISE から SGT を pxGrid 経由で動的同期し、Access Control Policy の **`Security Intelligence`** または **`Access Control Rule` の Source / Destination SGT タブ** で制御。
4. **SXP ノードバインディング優先順位 (Binding Source Priority)**:
   * 単一の IP アドレスに対して複数のソースから SGT バインディング情報が報告された場合、以下のデフォルト優先順位で処理されます：
     1. **Local Dynamic (802.1X/MAB)** (最高優先)
     2. **Static IP-SGT / Subnet-SGT Mapping**
     3. **SXP Connection** (外部 SXP ピアからの共有)
     4. **Internal / Unknown** (最低)
5. **CTS / SXP トラブルシューティングコマンド群**:
   * **`show cts environment-data`**: ISE との同期状態（SGT テーブル、PAC 有効期限）を確認。
   * **`show cts ip-sgt-map` / `show device-sensor cache`**: スイッチ上の IP-to-SGT バインディングテーブルを確認。
   * **`show cts sxp connections` / `show cts sxp db`**: SXP ピアの接続ステータスおよび SXP 経由で取得した SGT マッピングDBを確認。
   * **`show cts role-based permissions`**: 適用されている SGACL マトリクスと Hardware TCAM プログラミング状態を確認。

---

## 🛠 設定方法

### 1. Cisco IOS-XE: TrustSec / SXP / Inline Tagging / SGACL 基本設定

```bash
! ===== Step 1: AAA ＆ RADIUS / ISE 定義 =====
aaa new-model
aaa authentication dot1x default group radius
aaa authorization network default group radius
aaa accounting dot1x default start-stop group radius

radius server ISE_NODE
 address ipv4 10.1.100.50 auth-port 1812 acct-port 1813
 key cisco123

aaa group server radius RADIUS_SERVERS
 server name ISE_NODE

! ===== Step 2: Cisco TrustSec (CTS) 認証 ＆ PAC 設定 =====
cts credentials id SWITCH_DC password cisco123
cts authorization list default

! ===== Step 3: SXP (SGT Exchange Protocol) 設定 =====
cts sxp enable
cts sxp default source-interface Loopback0
cts sxp default password cisco_sxp_key

! SXP Peer 定義 (この機器が Listener の場合)
cts sxp connection peer 10.1.100.50 password cisco_sxp_key mode local listener

! 静的 IP-to-SGT マッピングの追加 (テスト/非 802.1X 機器用)
cts ip-sgt-map 10.200.1.100 sgt 20

! ===== Step 4: インターフェイスでの Inline Tagging 有効化 =====
interface GigabitEthernet1/0/10
 description Trunk to Core Switch
 switchport mode trunk
 cts manual
  policy static sgt 2 trusted              ! トランクポートでの SGT Inline Tagging 信頼設定
```

---

### 2. Cisco ASA: SGT アクセス制御 ＆ SXP 設定

```bash
! ===== ASA SXP ピアリング定義 =====
cts sxp enable
cts sxp default source-interface management
cts sxp connection peer 10.1.100.50 password cisco_sxp_key mode local listener

! ===== SGT オブジェクトグループ ＆ SGT ベース ACL 定義 =====
object-group security-group OG_SGT_SALES
 security-group-tag 10

object-group security-group OG_SGT_SERVERS
 security-group-tag 20

! SGT を使用した Extended ACL 設定
access-list ACL_OUTSIDE extended permit ip object-group-security OG_SGT_SALES object-group-security OG_SGT_SERVERS
access-list ACL_OUTSIDE extended deny ip any any

access-group ACL_OUTSIDE in interface inside
```

---

### 3. FMC / FTD: pxGrid 統合 ＆ SGT ベース Access Control Rule 設定

```text
[ Step 1: FMC と ISE 間での pxGrid 統合 ]
1. FMC UI で System > Integration > Identity Services Engine に移動。
2. ISE Primary Node IP (10.1.100.50) および pxGrid Client Certificate をアサインして接続確立。
3. FMC が ISE から SGT オブジェクト（Name ↔ Tag ID マッピング）を自動取得。

[ Step 2: Access Control Rule での SGT 条件指定 ]
1. Policies > Access Control で対象 ACP を編集 ➔ Add Rule。
2. Rule Name: BLOCK_GUEST_TO_DC
3. Users / SGT タブを開く:
   - Security Group Tags 選択画面から:
     - Source SGT: "Guest_SGT" (Tag 30)
     - Destination SGT: "DataCenter_SGT" (Tag 20)
4. Action: Block with Reset ➔ Save ＆ Deploy。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (IOS-XE / ASA / FTD) | 表示確認ポイント |
| :--- | :--- | :--- |
| **TrustSec 環境データ ＆ SGT 同期確認** | <code>show cts environment-data</code> | ISE から取得した `SGT Table`、`PAC` の有効性、同期タイムスタンプ |
| **IP-to-SGT バインディングテーブル表示** | <code>show cts ip-sgt-map</code><br><code>show cts role-based device-capability</code> | 各 IP アドレスに対応する SGT 値および登録ソース (`Local`, `SXP`, `Static`) |
| **SXP 接続ピア状態確認** | <code>show cts sxp connections</code> | `Status: On` (正常確立)、SXP Mode (`Speaker`/`Listener`)、SXP Version |
| **SXP バインディングデータベース確認** | <code>show cts sxp db</code> | SXP ピア経由で受信・提供した `IP-SGT` バインディングエントリー |
| **SGACL マトリクス ＆ 許可・拒否判定確認** | <code>show cts role-based permissions</code> | `Src SGT -> Dst SGT` に対する SGACL 名およびアクティブ許可ルール |
| **ハードウェア TCAM プログラミング確認** | <code>show cts role-based counters</code> | ハードウェア処理された SGACL マッチパケット数（Permit / Deny カウント） |
| **SXP デバッグログの追跡** | <code>debug cts sxp messages</code><br><code>debug cts sxp events</code> | SXP 接続ネゴシエーション、TCP 6499 ハンドシェイク、Binding Update 詳細 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **SXP ピアリングが確立せず `Pending-On` や `Disconnect` のままになる** | 1. SXP パスワード（MD5 キー）の不一致<br>2. SXP モード（Speaker / Listener）の不整合<br>3. TCP 6499 ポートが途中の FW で遮断されている | <code>show cts sxp connections</code><br><code>debug cts sxp events</code> | 両エンドで SXP MD5 パスワードを合致させ、一方が Speaker、他方が Listener となるよう設定。TCP 6499 通信を許可する。 |
| **端末が 802.1X 認証に成功したが、正しい SGT が付与されない** | ISE の Authorization Policy で割り当てる SGT 属性名または Tag ID の定義ミス | <code>show cts ip-sgt-map</code><br><code>show access-session detail</code> | ISE の Authorization Profile で **`Security Group`** 割当定義を再確認し、スイッチへ RADIUS `cisco-av-pair = cts:security-group-tag` が届いているか確認する。 |
| **SGACL を設定・同期したが、パケットが制限されず通過してしまう** | 1. エグレススイッチで `cts role-based enforcement` がグローバル無効<br>2. パケットに送信元 SGT が付与されていない | <code>show cts role-based permissions</code><br><code>show cts environment-data</code> | スイッチ上で **`cts role-based enforcement`** コマンドを投入してハードウェア適用を有効化する。 |
| **SXP 経由で取得した SGT マッピングが、スイッチローカルの静的マッピングで上書きされる** | TrustSec の Binding Source Priority 仕様により、Static / Local バインディングが SXP より優先されるため | <code>show cts ip-sgt-map</code> | 重複する静的 `cts ip-sgt-map` 定義を削除するか、Binding Priority 設定を調整する。 |

---

## ⚠ 制限事項

* **SXP バージョンと伝播属性の互換性**:
  * SXP v1/v2 は IPv4 アドレスのみをサポートします。IPv6 アドレスや Subnet/Range の SXP 伝播を行うには、SXP v3 または **SXP v4** の有効化が必須です。
* **ASA / FTD での Inline Tagging インターフェイス制約**:
  * 一部の Cisco ASA および FTD アプライアンス（モデル依存）では、物理ポートでの L2 CMD Inline Tagging カプセル化を直接サポートせず、SXP または pxGrid 経由での IP-SGT 受信のみに対応する場合があります。
* **SGACL のハードウェア TCAM 容量制限**:
  * スイッチの ASIC TCAM 容量を超過する膨大な SGACL エントリーがアサインされた場合、TCAM エラーが発生してソフトウェア処理へフォールバックし、パケット遅延やドロップの原因となります。

---

## 🔄 他技術との関連

* **Cisco ISE (Identity Services Engine)**: TrustSec アーキテクチャのセントラルブレイン。SGT 定義、SGACL マトリクス、Environment Data の配布、pxGrid / SXP の集中管理。
* **802.1X / MAB / PassiveID**: イングレスアクセスポイントでユーザーや端末を識別し、ISE からスイッチへ動的 SGT をプッシュするための一次認証機構。
* **MACsec (802.1AE)**: リンク間暗号化と組み合わせて SGT CMD ヘッダーを安全に保護・伝播（CTS-MACsec）。
* **Cisco ASA / FTD (FMC)**: マイクロセグメンテーション境界での SGT ベース Firewall Access Control Rule 適用。

---

## 🧩 比較表

### 1. Inline Tagging vs SXP (SGT Exchange Protocol)

| 比較項目 | Inline Tagging (CMD Header) | SXP (SGT Exchange Protocol) |
| :--- | :--- | :--- |
| **データ伝播方式** | L2 Frame の CMD ヘッダー（`0x8909`）に SGT をインライン直接埋め込み | **TCP ポート 6499** 経由の Out-of-Band `IP-to-SGT` バインディング制御転送 |
| **対象レイヤー** | **Layer 2 (Ethernet) トランク接続** | **Layer 3 (IP ネットワーク / WAN / Internet)** |
| **ハードウェア要求**| スイッチ ASIC の CMD ヘッダー処理対応が必要 | **ソフトウェア処理** (既存の汎用 L3 機器で動作可能) |
| **帯域オーバーヘッド**| 各フレームに 8 バイト（CMD）追加 | 制御プレーン（TCP 6499）でのマッピング更新時のみ |
| **スケーラビリティ**| 高い（状態保持不要のデータプレーンカプセル化） | 大規模ネットワークでは SXP Peer メッシュの設計管理が必要 |

### 2. Classic Access Control List (ACL) vs Security Group ACL (SGACL)

| 比較項目 | Classic IP Access List (ACL) | Security Group Access List (SGACL) |
| :--- | :--- | :--- |
| **識別条件** | 送信元 / 宛先 IP アドレス、サブネット、Port | **送信元 SGT / 宛先 SGT** (役割ベース) |
| **メンテナンス性** | 機器追加・IP 変更ごとに全ルールの変更・拡張が必要 | **IP アドレス依存なし**。新機器追加時もルール変更ゼロ |
| **ルール数** | IP アドレスごとの定義により数千〜数万行へ膨張 | **Role-to-Role（マトリクス形式）** の極めてシンプルなルール |
| **管理手法** | 各機器への個別 ACL 配布または CLI 管理 | **Cisco ISE による中央一元集中管理 ＆ 自動配布** |

---

## 💡 ベストプラクティス

1. **SXP ピアリング構造の設計（Speaker/Listener 構成の最適化）**:
   * 大規模環境では、すべてのスイッチ間で Full-Mesh SXP ピアリングを構築するのではなく、コアスイッチや ISE を **SXP Peer Reflector / Hub** として中央配置し、スター型拓撲で SXP バインディングを伝播させます。
2. **SXP MD5 認証キーの常時適用**:
   * SXP は TCP ポート 6499 上で動作する制御プロトコルです。不正な SXP ノードからの偽造 `IP-to-SGT` バインディングの注入を防ぐため、すべての SXP ピアリングで必ず `password` (TCP MD5 認証) を設定します。
3. **Egress Switch での `cts role-based enforcement` 有効化確認**:
   * SGACL マトリクスを ISE から正常同期していても、スイッチ上で `cts role-based enforcement` コマンドが設定されていないとポリシーが評価されません。設定初期化時の漏れを防ぐようチェックリスト化します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Cisco IOS-XE における ISE AAA 統合と SGT Dynamic Assignment 設定

#### 問題
スイッチ `SW-1` において、Cisco ISE (`10.1.100.50`) との RADIUS 連携を構成し、802.1X 認証成功時に ISE からプッシュされる動的 SGT（`Security-Group-Tag`）を受信してアクセスセッションへ適用できるようにせよ。

#### 要件
* RADIUS Key: `cisco123`
* `aaa authorization network default group radius` を定義すること。
* `cts authorization list default` を有効化すること。

#### 設定例

```bash
! ===== SW-1 RADIUS & AAA Configuration =====
aaa new-model
aaa authentication dot1x default group radius
aaa authorization network default group radius

radius server ISE_PRIMARY
 address ipv4 10.1.100.50 auth-port 1812 acct-port 1813
 key cisco123

aaa group server radius ISE_GROUP
 server name ISE_PRIMARY

! ===== TrustSec Authorization Definition =====
cts authorization list default
```

#### 検証方法
* 端末接続後に `show access-session interface Gi1/0/1 detail` を実行し、`Security Group Tag: 10` のように RADIUS より動的割り当てされていることを確認。

---

### 例2: SXP Speaker / Listener 接続と TCP MD5 認証設定 (IOS-XE)

#### 問題
ディストリビューションスイッチ `SW-DIST` (Speaker / `10.1.1.1`) とアクセススイッチ `SW-ACC` (Listener / `10.1.1.2`) の間で、SXP ピアリングを構築せよ。

#### 要件
* ソースインターフェイス: `Loopback0`
* MD5 認証パスワード: `sxp_pass123`
* SXP バージョン: `2`

#### 設定例

```bash
! ===== SW-DIST (SXP Speaker) =====
cts sxp enable
cts sxp default source-interface Loopback0
cts sxp default password sxp_pass123
cts sxp connection peer 10.1.1.2 password sxp_pass123 mode local speaker version 2

! ===== SW-ACC (SXP Listener) =====
cts sxp enable
cts sxp default source-interface Loopback0
cts sxp default password sxp_pass123
cts sxp connection peer 10.1.1.1 password sxp_pass123 mode local listener version 2
```

#### 検証方法
* `show cts sxp connections` を実行し、`Status: On` および Mode が正しく配置されていることを確認。

---

### 例3: 静的 IP-to-SGT マッピング (Static IP-SGT) 設定

#### 問題
スイッチ `SW-1` において、802.1X 非対応の静的 IP 機器（プリンター `10.200.1.100`）に対し、手動で SGT `15` (Printers) を割り当てよ。

#### 要件
* `cts ip-sgt-map` コマンドを使用すること。

#### 設定例

```bash
! ===== SW-1 Static Binding Configuration =====
cts ip-sgt-map 10.200.1.100 sgt 15
```

#### 検証方法
* `show cts ip-sgt-map` を実行し、`10.200.1.100` が SGT `15` として `Static` ソースで登録されていることを確認。

---

### 例4: インターフェイスでの CTS Manual Inline Tagging 設定

#### 問題
`SW-1` と `SW-2` 間のトランクポート `GigabitEthernet1/0/48` において、CTS Manual モードを有効化し、L2 イーサネットフレームに CMD タグ（Inline SGT）を付与してカプセル化転送せよ。

#### 要件
* トランク上のトラフィックに対して対向機器を信頼 (`trusted`) すること。

#### 設定例

```bash
! ===== SW-1 Interface Configuration =====
interface GigabitEthernet1/0/48
 description Trunk to SW-2 with TrustSec Inline Tagging
 switchport mode trunk
 cts manual
  policy static sgt 2 trusted
```

#### 検証方法
* `show cts interface GigabitEthernet1/0/48` を実行し、`CTS is enabled`, `Inline Tagging is enabled` と表示されることを確認。

---

### 例5: Egress Switch での SGACL マトリクスと Enforcement 有効化

#### 問題
エグレススイッチ `SW-EGRESS` において、Role-Based Enforcement（SGACL 適用）をグローバルおよび VLAN 10 インターフェイス上で有効化せよ。

#### 要件
* `cts role-based enforcement` コマンドを使用すること。

#### 設定例

```bash
! ===== SW-EGRESS Configuration =====
cts role-based enforcement

interface Vlan10
 description DataCenter Subnet
 ip address 10.10.10.1 255.255.255.0
 cts role-based enforcement
```

#### 検証方法
* `show cts role-based permissions` を実行し、現在適用されている SGACL マトリクスルールが表示されることを確認。

---

### 例6: ASA における SXP Listener 接続と SGT ベース ACL 設定

#### 問題
ASA1 において、ISE (`10.1.100.50`) から SXP 経由で IP-SGT バインディングを受信（Listener）し、SGT `10` (Sales) から SGT `20` (DataCenter) 宛ての通信のみを許可する ACL `ACL_CTS_FILTER` を構成せよ。

#### 要件
* SXP Password: `sxp_pass123`
* `object-group security-group` を使用すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
cts sxp enable
cts sxp default source-interface management
cts sxp connection peer 10.1.100.50 password sxp_pass123 mode local listener

object-group security-group OG_SALES
 security-group-tag 10

object-group security-group OG_DC
 security-group-tag 20

access-list ACL_CTS_FILTER extended permit ip object-group-security OG_SALES object-group-security OG_DC
access-list ACL_CTS_FILTER extended deny ip any any

access-group ACL_CTS_FILTER in interface inside
```

#### 検証方法
* ASA で `show cts sxp connections` および `show cts sxp db` を実行し、SXP バインディングが正常取得されていることを確認。

---

### 例7: FTD / FMC における ISE pxGrid 統合と SGT ベース Access Control Rule 設定

#### 問題
FMC 管理下の FTD において、SGT `30` (Guest) から SGT `20` (PCI_Server) 宛ての通信を拒否 (`Block with Reset`) する Access Control Rule を構成せよ。

#### 要件
* FMC の Access Control Rule 内の Users / SGT タブを使用すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Access Control で対象 ACP を編集 ➔ Add Rule。
2. Rule Name: BLOCK_GUEST_TO_PCI
3. Users / SGT タブを開く:
   - Security Group Tags:
     - Source SGT: Select "Guest" (Tag 30)
     - Destination SGT: Select "PCI_Server" (Tag 20)
4. Action: Block with Reset
5. Save ➔ Deploy 実行。
```

#### 検証方法
* ゲスト端末から PCI サーバーへ通信を発生させ、FMC `Analysis > Connections > Events` で `Action: Block with Reset` としてドロップ記録されることを確認。

---

### 例8: SXP v4 構成による Subnet-to-SGT マッピング伝播設定

#### 問題
ルーター `R1` (SXP Speaker) と `R2` (SXP Listener) 間で SXP v4 接続を構成し、サブネット `192.168.50.0/24` 全体に SGT `50` を割り当てて SXP 伝播させよ。

#### 要件
* SXP Version `4` を明示指定すること。

#### 設定例

```bash
! ===== R1 (Speaker) Configuration =====
cts sxp enable
cts sxp default source-interface Loopback0
cts sxp default password cisco123
cts sxp connection peer 10.2.2.2 password cisco123 mode local speaker version 4

! サブネット単位での Static SGT 割り当て
cts ip-sgt-map 192.168.50.0/24 sgt 50

! ===== R2 (Listener) Configuration =====
cts sxp enable
cts sxp default source-interface Loopback0
cts sxp default password cisco123
cts sxp connection peer 10.1.1.1 password cisco123 mode local listener version 4
```

#### 検証方法
* `R2` で `show cts sxp db` を実行し、`192.168.50.0/24 -> SGT 50` が受信・登録されていることを確認。

---

### 例9: CTS Environment Data の手動同期と PAC 管理

#### 問題
スイッチ `SW-1` において、ISE から最新の SGT テーブルおよび PAC (Protected Access Credential) を手動で再取得・リフレッシュせよ。

#### 要件
* `cts refresh environment-data` コマンドを使用すること。

#### 設定例

```bash
! ===== SW-1 Exec Command =====
SW-1# cts refresh environment-data
```

#### 検証方法
* `show cts environment-data` を実行し、`Status: LAST REFRESH SUCCESSFUL` および取得された SGT 名一覧を確認。

---

### 例10: SXP ピア切断時の Binding Expiration Timeout パラメータ調整

#### 問題
`SW-1` において、SXP ピア（Speaker）との TCP 接続が一時的に切断された際、受信済みの IP-SGT バインディングテーブルを即座に破棄せず、`3600` 秒間キャッシュとして維持（Hold）するように設定せよ。

#### 要件
* `cts sxp default holdtime` / `reconcile-period` パラメータを調整すること。

#### 設定例

```bash
! ===== SW-1 Configuration =====
cts sxp enable
cts sxp default holdtime min 120 max 3600
```

#### 検証方法
* `show cts sxp connections` を実行し、`Holdtime` 値が `3600` 秒に設定されていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 以下のコンフィグが設定された 2 台のスイッチ (`SW-1`, `SW-2`) 間の SXP 接続ステータスに関する記述として、最も適切なものはどれか。

```bash
! ===== SW-1 Configuration =====
cts sxp enable
cts sxp connection peer 10.1.1.2 password cisco123 mode local speaker version 2

! ===== SW-2 Configuration =====
cts sxp enable
cts sxp connection peer 10.1.1.1 password cisco123 mode local speaker version 2
```

* A) SXP セッションは正常に `On` 状態となり、双方向で SGT バインディングが同期される。
* B) 両スイッチが `speaker` モードとして設定されているため、Listener ノードが存在せず、SXP 接続は確立しない（`Pending-On` または `Disconnect` のままとなる）。
* C) SXP Password の指定方法が誤っているため、コンパイルエラーとなる。
* D) SXP v2 は同一 IP の指定をサポートしないためドロップされる。

**正解**: **B**  
**解説**: SXP ピアリング（TCP 6499）の確立には、一方のノードが **Speaker**（バインディング送信側）、他方のノードが **Listener**（バインディング受信側）である必要があります（または双方 `both`）。両ノードが `speaker` の場合、バインディングを受信する側が存在しないため接続が完成しません。

---

### 2. トラブルシュート問題

**問題**: エンタープライズスイッチにおいて、Cisco ISE から最新の SGACL マトリクスルール（`Src SGT 10 -> Dst SGT 20: Deny IP`）が正常に同期されていることを `show cts role-based permissions` で確認した。しかし、実際には SGT 10 の端末から SGT 20 のサーバーへパケットがドロップされず通過してしまう。考えられる原因として最も可能性が高いものはどれか。

* A) スイッチ上で `cts role-based enforcement` コマンドがグローバルまたは対象インターフェイスで設定されていない。
* B) SXP v4 が無効化されているため。
* C) スイッチの 802.1X ポートが Control Direction In になっているため。
* D) ISE の PAC が期限切れになっているため。

**正解**: **A**  
**解説**: スイッチが ISE から SGACL マトリクス情報を取得・保持していても、**`cts role-based enforcement`** コマンドがグローバル（または特定インターフェイス/VLAN）で有効化されていない場合、ハードウェア TCAM への SGACL ルール適用・ドロップ処理が実行されません。

---

### 3. Design 問題

**問題**: 全国に複数の拠点（Branch）を持つ企業において、中心となる Data Center (DC) の Cisco FTD ファイアウォール上で、拠点のユーザー端末の SGT タグに基づいたマイクロセグメンテーション（SGACL / ACP 制御）を実施したい。拠点スイッチと DC ファイアウォール間の WAN 回線は汎用 ISP ルーター（L3 WAN）経由であり、L2 CMD Inline Tagging のカプセル化フレームを透過できない。最も拡張性が高く適切な SGT 伝播設計はどれか。

* A) 全 WAN ルーターを MACsec 構成に変更して CMD ヘッダーを暗号化する。
* B) 拠点スイッチと DC ファイアウォール（または FMC/ISE）間で **SXP (SGT Exchange Protocol)** ピアリングを構築し、Out-of-Band で `IP-to-SGT` マッピングを伝播させる。
* C) WAN 回線全体に Private VLAN をオーバーレイ構築する。
* D) 拠点スイッチの全ポートを Trunk モードに変更する。

**正解**: **B**  
**解説**: L2 CMD ヘッダーを透過できない L3 ネットワークや WAN / Internet 越えで SGT 属性を伝播させる標準ソリューションは **SXP (SGT Exchange Protocol)** です。TCP ポート 6499 上で `IP-to-SGT` バインディングテーブルを伝播させることで、L3 境界を越えてエグレス側で SGT 制御を実行できます。

---

### 4. 実装問題

**問題**: Cisco ASA において、SGT 属性を用いたアクセス制御リスト（Extended ACL）を記述する際、正しい構文はどれか。

* A) `access-list ACL1 extended permit ip sgt 10 sgt 20`
* B) `access-list ACL1 extended permit ip object-group-security OG_SRC object-group-security OG_DST`
* C) `access-list ACL1 extended permit ip user-group Sales user-group Finance`
* D) `access-list ACL1 extended permit ip tag 10 tag 20`

**正解**: **B**  
**解説**: ASA で SGT を使用した ACL を作成する場合、あらかじめ `object-group security-group <NAME>` で `security-group-tag <ID>` を定義し、ACL 内で **`object-group-security <NAME>`** キーワードを指定して参照・記述します。

---

### 5. コンフィグ読解問題

**問題**: FTD / FMC において、ISE から pxGrid 経由で取得した SGT オブジェクトを Access Control Rule の条件として使用する場合、正しい適用場所（タブ）はどれか。

* A) Network タブ
* B) Port タブ
* C) Users / SGT タブ
* D) Inspection タブ

**正解**: **C**  
**解説**: FMC の Access Control Rule 編集画面において、SGT（Security Group Tag）条件は **Users / SGT タブ** 内の `Security Group Tags`（Source SGT / Destination SGT）で指定します。

---

## 🔗 参考リソース

* [Cisco TrustSec Configuration Guide, Cisco IOS Release 15.2(4)E](https://www.cisco.com/)
* [Cisco ASA Series General Operations Configuration Guide: Cisco TrustSec Integration](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Device Configuration Guide: Security Group Tags in Access Control](https://www.cisco.com/)
* [Cisco Technical Notes: Troubleshoot Cisco TrustSec SXP Connections](https://www.cisco.com/)
* [Cisco Validated Design (CVD): Cisco TrustSec Software-Defined Segmentation Design Guide](https://www.cisco.com/)
* [Cisco Live: Enterprise Microsegmentation using Cisco TrustSec (BRKSEC-2020)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で TrustSec ＆ SXP が出題された場合の迅速な判断・構築フローチャート：
    * **「Propagate SGT over L2 Trunk between CTS-capable switches」** ➔ Interface ➔ `cts manual` ➔ `policy static sgt X trusted` (Inline Tagging)。
    * **「Propagate SGT over L3 / WAN / Non-CTS switches」** ➔ `cts sxp enable` ➔ `cts sxp connection peer <IP> password <KEY> mode local speaker|listener` (SXP)。
    * **「Enforce SGACL on Egress Switch」** ➔ Global ➔ `cts role-based enforcement` ＋ `cts refresh environment-data` (ISE 保持)。
    * **「Enforce SGT on ASA / FTD」** ➔ ASA: `object-group security-group` ➔ FMC: ACP Rule ➔ Users / SGT Tab。
* **図解（SGT 伝播と SGACL パイプライン）**:
  ```text
  [ Client (802.1X) ]
          │ Authentication
          ▼
  [ Ingress Switch ] ◄── RADIUS (Assign SGT 10) ── [ Cisco ISE ]
          │
          ├── Option A: Inline CMD Tag (L2 Frame) ──► [ Core ]
          │
          └── Option B: SXP TCP 6499 Binding ───────► [ Core / Egress ]
                                                             │
                                                             ▼
                                                    [ Egress Switch / FW ]
                                                         │ Evaluates SGACL
                                                         │ (Src SGT 10 -> Dst SGT 20)
                                                         ▼
                                                    [ Target Server (SGT 20) ]
  ```
* **注意点**:
  * ラボ試験で SXP や SGACL を設定した後は、必ず `show cts sxp connections`、`show cts ip-sgt-map`、および `show cts role-based permissions` を実行し、接続ステータスが `On` であり、マッピングテーブルおよび SGACL エントリーがプログラミングされていることを確認・証明してください。

