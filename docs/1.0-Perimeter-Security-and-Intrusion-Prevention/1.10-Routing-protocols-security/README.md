---
layout: default
title: 1.10-Routing-protocols-security
nav_order: 10
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.10 Routing protocols security on Cisco IOS, Cisco ASA, and Cisco FTD

Cisco ネットワークインフラストラクチャおよびセキュリティアプライアンス（Cisco IOS/IOS-XE, Cisco ASA, Cisco Firepower Threat Defense [FTD] / FMC）における **Routing Protocols Security（ルーティングプロトコルセキュリティ）** は、コントロールプレーン（Control Plane）の信頼性・完全性・可用性を保護するための核となるセキュリティアーキテクチャです。

ルーティングプロトコル（OSPFv2/OSPFv3, EIGRP/EIGRPv6, BGP, RIP/RIPng）に対する攻撃（不正ルーターの挿入、ルートポイズニング、Man-in-the-Middle 盗聴、BGP セッションハイジャック、DoS/Resource Exhaustion）を防ぐため、**ネイバー認証（MD5, HMAC-SHA-256, TCP MD5, IPsec AH/ESP）**、**GTSM (Generalized TTL Security Mechanism)**、**ルートフィルタリング (Distribute-list, Prefix-list, Route-map)**、**Passive-interface 制御**、および **ASA/FTD トランジット環境における TCP Normalization（TCP オプション 19 の不透過回避・シーケンス番号ランダム化制御）** を適切に設計・実装・検証する能力が評価されます。

---

## 📘 概要

* **機能概要**:
  * **ルーティングプロトコル認証 (Routing Protocol Authentication)**:
    * **OSPFv2**: プレーンテキスト (Type 1)、MD5 ハッシュ (Type 2)、および HMAC-SHA (RFC 5709 / Key-chain) による OSPF パケットの改ざん・偽造防止。
    * **OSPFv3**: IPv6 ネイティブ仕様に基づき、プロトコル独自の認証ヘッダーを持たず、**IPsec AH (Authentication Header) または ESP (Encapsulating Security Payload)** を用いてパケット単位の暗号化・認証を実現。
    * **EIGRP / EIGRPv6**: MD5 および Key-chain を用いた **HMAC-SHA-256** 認証。キーロールオーバー（順次キー変更）をサポート。
    * **BGP**: TCP オプション 19（RFC 2385）を利用した **TCP MD5 シグネチャ** による BGP ピアセッションの偽造・改ざん防止。
  * **BGP GTSM (Generalized TTL Security Mechanism / RFC 5082 / BTSH)**:
    * eBGP ピア間のパケット TTL（Time-To-Live）が送信時に `255` に設定され、受信側で `255 - Hops` 以上の TTL を持つパケットのみを受理することで、複数ホップ離れた外部からの BGP SYN フラッドやセッションスプーフィング攻撃をドロップするメカニズム。
  * **ASA / FTD トランジット保護と TCP Normalization**:
    * ASA / FTD 経由で BGP MD5 ピアが通信する場合、デフォルトの **TCP シーケンス番号ランダム化 (Sequence Number Randomization)** および **TCP オプション 19 (MD5 Digest) の削除** によって BGP 認証が失敗する。MPF (Modular Policy Framework) および `tcp-map` を用いてこれをバイパス・カプセル化通過させる。
  * **コントロールプレーンハードニング (Control Plane Hardening)**:
    * **Passive-interface**: 不必要なインターフェイスでの Hello パケット送受信を停止し、不正ルーターの隣接関係樹立を防止。
    * **Route Filtering**: Prefix-list、Route-map、Distribute-list による不正ルート・ボゴン（Bogon）プレフィックスの伝播防止。
    * **CoPP (Control Plane Policing)**: ルーティングプロトコル（TCP 179, IP Protocol 89 [OSPF], IP Protocol 88 [EIGRP]）宛てトラフィックの帯域制御・レート制限。
* **利用目的**:
  * 不正なルーターや悪意あるエンドポイントによる偽ルーティング情報の注入（Route Injection / Route Poisoning）を遮断する。
  * ルーティングループ、トラフィックの不正迂回（Traffic Hijacking / MitM）、DoS（Denial of Service）の発生を防ぐ。
  * 防火壁（ASA / FTD）を通過する制御通信や、ファイアウォール自体が参加する動的ルーティングの整合性を維持する。
* **どのような場面で利用するか**:
  * 企業 WAN 境界、データセンター相互接続 (DCI)、SP ピアリングポイント (eBGP)。
  * 内部 LAN/DMZ セグメントにおけるコアスイッチ・ファイアウォール間の動的ルーティング確立時。

---

## 🔑 要点

| 項目 | Cisco IOS / IOS-XE | Cisco ASA | Cisco FTD (FMC / FDM) |
| :--- | :--- | :--- | :--- |
| **OSPFv2 認証** | インターフェイス/エリア単位 (Clear-text, MD5, HMAC-SHA) | インターフェイス/エリア単位 (Clear-text, MD5) | FMC GUI / FDM 経由での OSPFv2 MD5 認証 |
| **OSPFv3 認証** | IPsec AH / ESP (`ipv6 ospf authentication/encryption ipsec`) | IPsec AH / ESP (`ipv6 ospf authentication/encryption ipsec`) | FMC 経由での OSPFv3 IPsec 構成 |
| **EIGRP 認証** | Key-chain による MD5 / HMAC-SHA-256 | インターフェイス単位の MD5 認証 (`authentication mode/key`) | FMC OSPF/BGP ネイティブ, EIGRP は FlexConfig 経由 |
| **BGP TCP MD5** | `neighbor <IP> password <key>` | `neighbor <IP> password <key>` | FMC GUI (BGP Neighbor Password) |
| **BGP GTSM** | `neighbor <IP> ttl-security hops <n>` | `neighbor <IP> ttl-security hops <n>` | FMC / FlexConfig 経由 |
| **BGP トランジット時の注意点** | N/A (ルーター処理) | **TCP Randomization 無効化 ＋ TCP Option 19 許可が必要** | **TCP Normalization 制御が必要** |
| **キーローテーション** | Key-chain (`accept-lifetime`, `send-lifetime`) | 手動キー変更 (ID 重複定義) | Key Chain オブジェクト (FTD 7.0+) |

---

## 🏗 動作原理

### 1. BGP TCP MD5 認証（RFC 2385）および ASA/FTD トランジットの仕組み

BGP セッションは TCP ポート 179 上で確立されます。BGP TCP MD5 認証では、TCP ヘッダーの **TCP Option 19 (MD5 Signature Option)** を利用して、TCP 疑似ヘッダー、TCP ヘッダー、TCP セグメントデータ、および事前共有キー（Password）から 128 ビットの MD5 ダイジェストを計算します。

```text
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│ TCP Header with MD5 Signature Option (RFC 2385)                                                  │
│                                                                                                  │
│ [ Source IP | Dest IP | Src Port | Dst Port (179) | Sequence Number | TCP Option 19 (18-byte Hash) ]│
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                 │
                                                 ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│ ASA / FTD Firewall Transit Processing                                                            │
│                                                                                                  │
│ 1. Default TCP Normalization:                                                                    │
│    └── ASA strips TCP Option 19 (Clears Unknown/MD5 Options) ──► Breaks BGP Auth!                 │
│ 2. Default Sequence Number Randomization:                                                        │
│    └── ASA changes Initial Sequence Number (ISN) ──────────────► Breaks MD5 Hash Check!          │
│                                                                                                  │
│ Required ASA/FTD Solution:                                                                       │
│ ├── tcp-map BGPMAP ➔ tcp-options range 19 19 allow                                               │
│ └── policy-map ➔ set connection random-sequence-number disable + set connection advanced-options │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

### 2. BGP GTSM (Generalized TTL Security Mechanism / BTSH) の動作原理

eBGP ピア間（直結 1 ホップ想定）で、攻撃者が遠隔（複数ホップ離れたネットワーク）から偽造 BGP パケット（TCP 179）を送信し、ルーターの CPU を枯渇させる攻撃を防ぐアルゴリズム：

```text
[ Attacker (5 Hops Away) ] ─── TCP SYN (TTL = 255) ───► [ Router 1 ] ──► [ Router 2 ] ──► [ Router 3 ] ──► [ Target BGP Router ]
                                                                                                               │
                                                                                                               ▼
                                                                                                 Received TTL = 250 (255 - 5)
                                                                                                               │
                                                                                                 GTSM Check:
                                                                                                 `neighbor ttl-security hops 1`
                                                                                                 Min Incoming TTL = 254 (255 - 1)
                                                                                                               │
                                                                                                 250 < 254 ──► DROP Packet!
```

---

## ⚙ 動作シーケンス

1. **OSPFv2 MD5 認証シーケンス**:
   * OSPF パケット送信側は、パケット本体 ＋ OSPF ヘッダー ＋ メッセージダイジェストキー（Password） ＋ Key ID を元に MD5 ハッシュ値を生成。
   * OSPF パケットの末尾に 16 バイトの MD5 ダイジェストを付与し、ヘッダーの `Auth Type` を `2`（Message Digest）、`Key ID` を指定して送信。
   * 受信側は自らに設定された同一 Key ID のパスワードでハッシュを再計算。ダイジェストが一致した場合のみ OSPF パケットを受理し、隣接関係（Adjacency）を維持。
2. **EIGRP HMAC-SHA-256 Key-Chain 評価シーケンス**:
   * EIGRP プロセスは Key-chain で定義された `send-lifetime` を参照し、現在時刻に合致するアクティブな Key ID と Key-string を選択。
   * パケットに HMAC-SHA-256 ダイジェストを付与して送信。
   * 受信側は `accept-lifetime` に合致するキーで検証。キー交代時期に送信・受信ライフタイムにオーバーラップ（重複時間帯）を持たせることで、**ノンストップ（Hitless）キーローテーション** を実現。
3. **ASA / FTD Transit BGP MD5 パケット処理シーケンス**:
   * クライアントルーター R1 が BGP TCP SYN（Option 19 付き）を送信。
   * ASA / FTD がパケットを受信。MPF の `tcp-map` ルールに基づき、TCP Option 19 をクリアせずにそのまま透過。
   * ASA / FTD は BGP トラフィッククラスに対して **TCP シーケンス番号のランダム化（ISN Rewrite）を無効化** して転送。
   * 宛先ルーター R2 は無修正の TCP ヘッダーと Option 19 ダイジェストを受信し、MD5 認証を正しく完了して BGP エスタブリッシュ状態へ移行。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **OSPF 認証：グローバル (Area-level) vs インターフェイスレベルの優先関係**:
   * `router ospf 1` 配下での `area 0 authentication message-digest`（エリア全体有効化）と、`interface` 配下での `ip ospf authentication message-digest` / `ip ospf authentication null` の優先順位。
   * インターフェイスレベルの設定は、エリア単位の設定をオーバーライドします。「特定インターフェイスのみ認証を無効化（Null 認証）せよ」という要件では、対象インターフェイスで **`ip ospf authentication null`** を設定します。
2. **ASA / FTD 経由の BGP MD5 通信における TCP Normalization の罠**:
   * **問題背景**: BGP ピア同士が ASA/FTD を挟んで BGP MD5 認証を構成している場合、接続が Active 状態（`show ip bgp summary` で Active）のまま Established に移行しません。ルーター側には `%TCP-6-BADAUTH: No MD5 digest` や `Invalid MD5 digest` ログが出力されます。
   * **必須解決コンフィグ (ASA)**:
     ```bash
     tcp-map BGPMAP
      tcp-options range 19 19 allow
     class-map C_BGP
      match port tcp eq 179
     policy-map global_policy
      class C_BGP
       set connection random-sequence-number disable
       set connection advanced-options BGPMAP
     ```
3. **OSPFv3 IPsec 認証 / 暗号化の要件設定**:
   * OSPFv3 は AH (Authentication Header) または ESP (Encapsulating Security Payload) をインターフェイスまたはエリア配下で直接定義します。
   * SPI (Security Parameter Index) 値と 40 文字の Hex キー（SHA1 / MD5 / AES）の手動設定が求められます。
   * 例: `ipv6 ospf authentication ipsec spi 1000 sha1 <40-HEX-KEY>`
4. **EIGRP Key-Chain キーローテーション（Hitless Key Change）**:
   * 要件に「キー変更時に隣接関係を切断させないこと」とある場合、既存キーと新規キーの **`accept-lifetime`** および **`send-lifetime`** にオーバーラップ時間帯（例: 1時間重複）を設けます。
5. **ASA / FTD 自身がルーティングに参加する場合の注意点**:
   * ASA で OSPF / EIGRP / BGP を動かす場合、マルチキャストパケット（OSPF `224.0.0.5`/`224.0.0.6`、EIGRP `224.0.0.10`）やユニキャストパケットを許可する ACL がインバウンドインターフェイスに必要な場合があります。
   * ASA では `ospf authentication message-digest` および `ospf message-digest-key 1 md5 <KEY>` をインターフェイス配下で設定します。

---

## 🛠 設定方法

### 1. Cisco IOS / IOS-XE: OSPFv2, OSPFv3, EIGRP, BGP, GTSM 完全設定

```bash
! ===== 1. OSPFv2 MD5 認証 (Interface ＆ Area Level) =====
interface GigabitEthernet0/0/1
 ip address 10.1.12.1 255.255.255.0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CISCO_OSPF_KEY
 ip ospf 1 area 0

router ospf 1
 router-id 1.1.1.1
 area 0 authentication message-digest  ! エリア全域有効化
 passive-interface default
 no passive-interface GigabitEthernet0/0/1

! ===== 2. OSPFv3 IPsec AH / ESP 認証・暗号化 =====
interface GigabitEthernet0/0/2
 ipv6 address 2001:db8:12::1/64
 ipv6 ospf 10 area 0
 ipv6 ospf authentication ipsec spi 1000 sha1 0123456789012345678901234567890123456789
 ipv6 ospf encryption ipsec spi 2000 esp aes-cbc 256 0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF md5 0123456789ABCDEF0123456789ABCDEF

! ===== 3. EIGRP HMAC-SHA-256 Key-Chain 認証 =====
key chain EIGRP_KEYCHAIN
 key 1
  key-string EIGRP_SECRET_KEY
  accept-lifetime 00:00:00 Jan 1 2026 infinite
  send-lifetime 00:00:00 Jan 1 2026 infinite

interface GigabitEthernet0/0/3
 ip address 10.1.23.1 255.255.255.0
 ip authentication mode eigrp 100 hmac-sha-256
 ip authentication key-chain eigrp 100 EIGRP_KEYCHAIN

! ===== 4. BGP TCP MD5 認証 ＆ BGP GTSM (TTL Security) =====
router bgp 65001
 bgp router-id 1.1.1.1
 neighbor 192.168.12.2 remote-as 65002
 neighbor 192.168.12.2 password BGP_SECRET_PASS
 neighbor 192.168.12.2 ttl-security hops 1   ! GTSM: 1 ホップ直結 (Min incoming TTL = 254)
```

---

### 2. Cisco ASA: OSPFv2 MD5, EIGRP 認証 ＆ Transit BGP TCP Normalization

```bash
! ===== 1. ASA 自体が参加する OSPFv2 MD5 認証 =====
interface GigabitEthernet0/0
 nameif OUTSIDE
 security-level 0
 ip address 192.168.100.1 255.255.255.0
 ospf authentication message-digest
 ospf message-digest-key 1 md5 ASA_OSPF_PASS

router ospf 1
 router-id 10.10.10.10
 network 192.168.100.0 255.255.255.0 area 0

! ===== 2. ASA 自体が参加する EIGRP 認証 =====
interface GigabitEthernet0/1
 nameif INSIDE
 security-level 100
 ip address 10.1.1.1 255.255.255.0
 authentication mode eigrp 100 md5
 authentication key eigrp 100 ASA_EIGRP_PASS key-id 1

router eigrp 100
 network 10.1.1.0 255.255.255.0

! ===== 3. ASA トランジット BGP MD5 パススルー制御 (必須) =====
tcp-map BGP_MD5_MAP
 tcp-options range 19 19 allow         ! TCP Option 19 (MD5) の通過を許可

class-map C_BGP_TRAFFIC
 match port tcp eq 179

policy-map global_policy
 class C_BGP_TRAFFIC
  set connection random-sequence-number disable  ! シーケンス番号の書き換えを無効化
  set connection advanced-options BGP_MD5_MAP   ! tcp-map のバインド

service-policy global_policy global
```

---

### 3. Cisco FTD / FMC: OSPF / BGP 認証の設定

```text
[ FMC GUI: FTD OSPFv2 MD5 認証設定 ]
1. Devices > Device Management > Routing > OSPF > Interface を開く。
2. 対象インターフェイス (例: Outside) を編集。
3. "Authentication" タブを選択:
   - Authentication Type: MD5
   - Key ID: 1
   - Key / MD5 Secret: FTD_OSPF_PASS
4. Save して Deploy を実行。

[ FMC GUI: FTD BGP Neighbor Password 設定 ]
1. Devices > Device Management > Routing > BGP > IPv4 > Neighbor を開く。
2. 対象 Neighbor (例: 192.168.12.2) を編集。
3. "Security" または "Advanced" タブを選択:
   - Password: FTD_BGP_PASS
4. Save して Deploy を実行。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (IOS / ASA / FTD) | 表示確認ポイント |
| :--- | :--- | :--- |
| **OSPFv2 インターフェイス認証確認** | <code>show ip ospf interface <IF></code> | `Message digest authentication enabled`, `Youngest key id is 1` |
| **OSPFv2 エリア全体認証確認** | <code>show ip ospf \| section Area</code> | `Area has message digest authentication` |
| **OSPFv3 IPsec SA 確立確認** | <code>show crypto ipsec sa</code> / <code>show ipv6 ospf interface</code> | `IPsecv6 policy name: OSPFv3-`, `AH/ESP packets encaps/decaps` |
| **EIGRP ネイバー ＆ 認証確認** | <code>show ip/ipv6 eigrp neighbors</code><br><code>show key chain</code> | ネイバーが Up 状態であること、キーの `accept/send-lifetime` 状態 |
| **BGP セッション ＆ 認証確認** | <code>show ip bgp summary</code><br><code>show ip bgp neighbors</code> | `BGP state = Established`, `Option Flags: md5` |
| **BGP GTSM (TTL Security) 確認** | <code>show ip bgp neighbors \| i TTL\|hop</code> | `External BGP neighbor may be up to 1 hops away`, `Minimum incoming TTL 254` |
| **ASA MPF TCP Normalization 確認** | <code>show service-policy inspect scopy</code> / <code>show conn</code> | BGP セッションの `r` フラグ (Random sequence number disabled) |
| **ルーティングプロトコル認証デバッグ** | <code>debug ip ospf adj</code><br><code>debug ip eigrp packet</code><br><code>debug ip bgp updates</code> | `Mismatch Authentication type`, `Invalid MD5 digest`, `Mismatched Key` ログ |

---

## 🚨 トラブルシュート

| 症状 | Cause (原因) | Check Command (確認コマンド) | Action (対処方法) |
| :--- | :--- | :--- | :--- |
| **OSPFv2 ネイバーが `DOWN` / `INIT` のまま確立しない** | OSPF 認証タイプ不一致 (Type 1 vs Type 2)、または Password / Key ID 不一致 | <code>debug ip ospf adj</code> | `Mismatch Authentication type` または `Mismatched Authentication Key - Message Digest Key X` を確認し、Key ID と Key-string を統一する。 |
| **ASA / FTD を挟んだ BGP ピア間で MD5 認証が成功せず `Active` のままになる** | ASA が TCP Option 19 を削除、または TCP シーケンス番号をランダム化している | <code>show log \| inc BADAUTH</code> (ルーター側) | ASA で `tcp-map` を作成し `tcp-options range 19 19 allow` を定義、MPF で `set connection random-sequence-number disable` を適用する。 |
| **BGP GTSM 設定後、正当な eBGP ピアとのセッションが切断される** | 対面ルーター側で GTSM が未設定（通常の eBGP TTL 1 で送信されている）、または Hops 数指定が誤っている | <code>show ip bgp neighbors</code> | 対面ルーターにも `neighbor <IP> ttl-security hops <n>` を設定し、送信パケットの TTL を 255 に統一させる。 |
| **EIGRP キーローテーション時に瞬断（フラッピング）が発生する** | 新旧キーの `send-lifetime` と `accept-lifetime` にオーバーラップ期間（重複時間）が存在しない | <code>show key chain</code> | 旧キーの有効期限終了前に新キーの `accept-lifetime` を先行有効化させる設計に変更する。 |
| **OSPFv3 IPsec 認証で `show ipv6 ospf neighbor` にネイバーが表示されない** | SPI 値または Manual IPsec 鍵 (Hex 40文字) が対面と不一致 | <code>show crypto ipsec sa</code> | 両端の SPI 値 (例: 1000) と Hex 暗号鍵を完全に一致させる。 |

---

## ⚠ 制限事項

* **BGP GTSM と eBGP Multihop の排他制御**:
  * `neighbor ttl-security` (GTSM) と `neighbor ebgp-multihop` コマンドは**相互排他 (Mutually Exclusive)** です。同一 BGP ネイバーに対して両方を同時に設定することはできません（GTSM が自動的に TTL 255 送信および受信 TTL チェックを包括するため）。
* **OSPFv3 における ISAKMP 動的鍵不可**:
  * OSPFv3 の IPsec セキュリティ機能は IKE/ISAKMP による動的鍵交換をサポートせず、**手動設定キー (Manual Keying / SPI)** のみがサポートされます。
* **ASA / FTD EIGRP 認証の制限**:
  * ASA / FTD の EIGRP 認証は Key-chain による高度なアルゴリズム切替が制限され、直接インターフェイス配下での MD5 設定が基本となります。

---

## 🔄 他技術との関連

* **Control Plane Policing (CoPP)**: 不正なルーティングプロトコルパケット（TCP 179, OSPF IP 89, EIGRP IP 88）をコントロールプレーン手前でレート制限・ドロップする保護フレームワーク。
* **ASA Modular Policy Framework (MPF)**: ASA を通過する BGP MD5 通信に対し、`tcp-map` と組み合わせて TCP シーケンス番号ランダム化を無効化し、TCP Option 19 を透過。
* **Unicast Reverse Path Forwarding (uRPF)**: 動的ルーティングテーブル（FIB）と連動し、送信元 IP スプーフィングパケットを水際でドロップ。
* **Resource Public Key Infrastructure (RPKI)**: BGP ルートインジェクション対策として、BGP プレフィックスの所有権 (ROA: Route Origin Authorization) を暗号学的に検証する広域セキュリティ機構。

---

## 🧩 比較表

### 1. OSPFv2 認証 vs OSPFv3 認証

| 比較項目 | OSPFv2 (IPv4) | OSPFv3 (IPv6) |
| :--- | :--- | :--- |
| **認証メカニズム** | OSPF ヘッダー内組み込み (`Auth Type`, `Auth Data`) | **IPv6 ネイティブ IPsec (AH / ESP)** |
| **暗号化対応** | 非対応 (認証・整合性のみ) | **対応** (ESP によるパケット暗号化) |
| **対応アルゴリズム** | Clear-text, MD5, HMAC-SHA (RFC 5709) | AH: SHA1-HMAC, MD5-HMAC / ESP: AES-CBC, 3DES |
| **鍵管理** | Password / Key ID または Key-chain | **SPI (Security Parameter Index)** ＋ Manual Hex Key |

### 2. BGP GTSM (TTL Security) vs Traditional eBGP Multihop

| 比較項目 | BGP GTSM (RFC 5082) | Traditional eBGP Multihop |
| :--- | :--- | :--- |
| **送信パケット TTL** | **常に 255** で送信 | 設定された Hops 数 (例: 2 ➔ TTL=2) で送信 |
| **受信パケット検証** | `255 - Hops` 以上の TTL (例: 254) のみ受理 | 受信パケットの TTL 検証は行わない |
| **DoS 攻撃耐性** | **極めて高い** (遠隔スプーフィングパケットを即時ドロップ) | **低い** (遠隔からの TCP 179 SYN フラッドを CPU 処理してしまう) |

---

## 💡 ベストプラクティス

1. **OSPF / EIGRP の Passive-Interface Default 運用**:
   * ルーティングプロセス内で **`passive-interface default`** を設定し、隣接関係を形成する明示的なインターフェイスのみ **`no passive-interface <IF>`** で開口させることで、意図しないポートからの Hello 送出と不正ルーター接続を完全に封じ込めます。
2. **BGP ピアリングにおける GTSM ＋ TCP MD5 の二重バインド**:
   * 直結 eBGP ピアおよび Multi-hop eBGP ピアにおいて、`neighbor password` (TCP MD5) によるデータ改ざん防止と、`neighbor ttl-security` (GTSM) による遠隔 CPU 枯渇攻撃防止を併用します。
3. **Key-Chain での重複生存期間 (Overlapping Lifetime) 設計**:
   * EIGRP や OSPF HMAC-SHA で Key-chain を利用する際、新しいキーの `send-lifetime` / `accept-lifetime` の開始時刻を旧キーの終了時刻より 1〜2 時間前に設定し、鍵交換時のセッション切断を防止します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: IOS OSPFv2 MD5 認証とエリア単位・インターフェイス単位のオーバーライド構成

#### 問題
R1 と R3 間で OSPFv2 Area 0 を構成せよ。Area 0 全体で MD5 認証を有効化し、キー `AREA0_KEY` (Key ID 1) を使用せよ。ただし、R3 のテスト用インターフェイス `Gi0/2` のみ OSPF 認証を無効化 (Null 認証) せよ。

#### 要件
* グローバル Area 認証コマンドを使用すること。
* 特定インターフェイスで `ip ospf authentication null` を使用すること。

#### 設定例

```bash
! ===== R1 Configuration =====
interface GigabitEthernet0/1
 ip address 10.1.13.1 255.255.255.0
 ip ospf message-digest-key 1 md5 AREA0_KEY
 ip ospf 1 area 0

router ospf 1
 router-id 1.1.1.1
 area 0 authentication message-digest

! ===== R3 Configuration =====
interface GigabitEthernet0/1
 ip address 10.1.13.3 255.255.255.0
 ip ospf message-digest-key 1 md5 AREA0_KEY
 ip ospf 1 area 0

interface GigabitEthernet0/2
 ip address 10.3.3.3 255.255.255.0
 ip ospf authentication null            ! エリア認証のオーバーライド (Null 認証)
 ip ospf 1 area 0

router ospf 1
 router-id 3.3.3.3
 area 0 authentication message-digest
```

#### 検証方法
* `show ip ospf interface GigabitEthernet0/1` で `Message digest authentication enabled` を確認。
* `show ip ospf interface GigabitEthernet0/2` で `No authentication enabled` を確認し、R1-R3 間で OSPF ネイバーが FULL となることを確認。

---

### 例2: IOS OSPFv3 IPsec AH / ESP 認証および暗号化設定

#### 問題
R1 と R2 の OSPFv3 Area 1 において、パケットの改ざん防止と暗号化を実現するため、IPsec ESP 認証・暗号化を設定せよ。

#### 要件
* SPI: `2000`
* 暗号化: `AES-CBC 256` (Key: `0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF`)
* 認証: `MD5` (Key: `0123456789ABCDEF0123456789ABCDEF`)

#### 設定例

```bash
! ===== R1 Configuration =====
interface GigabitEthernet0/0/1
 ipv6 address 2001:db8:12::1/64
 ipv6 ospf 1 area 1
 ipv6 ospf encryption ipsec spi 2000 esp aes-cbc 256 0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF md5 0123456789ABCDEF0123456789ABCDEF

router ospfv3 1
 router-id 1.1.1.1
!
! ===== R2 Configuration =====
interface GigabitEthernet0/0/1
 ipv6 address 2001:db8:12::2/64
 ipv6 ospf 1 area 1
 ipv6 ospf encryption ipsec spi 2000 esp aes-cbc 256 0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF md5 0123456789ABCDEF0123456789ABCDEF

router ospfv3 1
 router-id 2.2.2.2
```

#### 検証方法
* `show crypto ipsec sa` を実行し、`IPsecv6 policy name: OSPFv3-1-2000` および `#pkts encrypt / #pkts decrypt` カウンターが増加していることを確認。

---

### 例3: IOS EIGRP HMAC-SHA-256 キーローテーション付き Key-Chain 構成

#### 問題
R1 と R2 間で EIGRP AS 100 を構成し、Key-chain `EIGRP_KEYS` を使用して HMAC-SHA-256 認証を設定せよ。2 つのキー（Key 1 および Key 2）を定義し、キー切り替え時に通信が瞬断しないようライフタイムを重複させよ。

#### 要件
* Key 1: `KEY_OLD` (2026年1月1日〜2026年12月31日)
* Key 2: `KEY_NEW` (2026年12月1日〜2027年12月31日)

#### 設定例

```bash
! ===== R1 Configuration =====
key chain EIGRP_KEYS
 key 1
  key-string KEY_OLD
  accept-lifetime 00:00:00 Jan 1 2026 23:59:59 Dec 31 2026
  send-lifetime 00:00:00 Jan 1 2026 23:59:59 Dec 31 2026
 key 2
  key-string KEY_NEW
  accept-lifetime 00:00:00 Dec 1 2026 23:59:59 Dec 31 2027
  send-lifetime 00:00:00 Dec 1 2026 23:59:59 Dec 31 2027

interface GigabitEthernet0/1
 ip address 10.1.12.1 255.255.255.0
 ip authentication mode eigrp 100 hmac-sha-256
 ip authentication key-chain eigrp 100 EIGRP_KEYS

router eigrp 100
 net 10.1.12.0 0.0.0.255
```

#### 検証方法
* `show key chain` を実行し、現在アクティブなキーを確認。`show ip eigrp neighbors` でネイバー関係が切り替え時間中も維持されることを確認。

---

### 例4: BGP TCP MD5 認証と BGP GTSM (TTL Security) 併用設定

#### 問題
R1 (AS 65001) と R2 (AS 65002) 間で eBGP ピアを確立せよ。BGP MD5 認証キー `BGP_SECRET` を設定し、さらに GTSM (TTL Security) を有効化して直結 1 ホップ離れたピアからの接続のみを許可せよ。

#### 要件
* `neighbor password` および `neighbor ttl-security hops 1` を使用すること。

#### 設定例

```bash
! ===== R1 Configuration =====
router bgp 65001
 bgp router-id 1.1.1.1
 neighbor 192.168.12.2 remote-as 65002
 neighbor 192.168.12.2 password BGP_SECRET
 neighbor 192.168.12.2 ttl-security hops 1

! ===== R2 Configuration =====
router bgp 65002
 bgp router-id 2.2.2.2
 neighbor 192.168.12.1 remote-as 65001
 neighbor 192.168.12.1 password BGP_SECRET
 neighbor 192.168.12.1 ttl-security hops 1
```

#### 検証方法
* `show ip bgp neighbors | i TTL|hop|Option` を実行し、`External BGP neighbor may be up to 1 hops away` および `Minimum incoming TTL 254` を確認。

---

### 例5: ASA トランジット BGP MD5 認証通過のための MPF ＆ TCP Normalization 設定

#### 問題
R1 と R2 の目で BGP MD5 認証セッションを確立するが、両ルーター間に ASA1 がインラインで存在している。ASA1 によって TCP Option 19 が削除され BGP 認証が失敗するのを防ぐため、ASA1 上で MPF および `tcp-map` を定義して BGP MD5 トラフィックを正常透過させよ。

#### 要件
* `tcp-map` 名: `MAP_BGP_MD5`
* シーケンス番号のランダム化を無効化すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
tcp-map MAP_BGP_MD5
 tcp-options range 19 19 allow

class-map C_BGP_TRAFFIC
 match port tcp eq 179

policy-map global_policy
 class C_BGP_TRAFFIC
  set connection random-sequence-number disable
  set connection advanced-options MAP_BGP_MD5

service-policy global_policy global
```

#### 検証方法
* R1/R2 で `show ip bgp summary` を実行し、BGP セッションが `Established` 状態になることを確認。ASA1 で `show conn` を実行し `r` フラグが付与されていることを確認。

---

### 例6: ASA 自体が参加する OSPFv2 MD5 認証とルートフィルタリング

#### 問題
ASA1 において `OUTSIDE` インターフェイスで OSPFv2 エリア 0 を動かし、隣接ルーター R2 と MD5 認証（Key ID 1, Password `ASA_PASS`）を構成せよ。また、R2 から学習するルートのうち `10.99.0.0/16` を除外する Prefix-list フィルタをアサインせよ。

#### 要件
* `ospf authentication message-digest` コマンドを使用すること。
* `prefix-list` を使用すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
prefix-list FILTER_IN deny 10.99.0.0/16
prefix-list FILTER_IN permit 0.0.0.0/0 le 32

interface GigabitEthernet0/0
 nameif OUTSIDE
 security-level 0
 ip address 192.168.10.1 255.255.255.0
 ospf authentication message-digest
 ospf message-digest-key 1 md5 ASA_PASS

router ospf 1
 router-id 10.10.10.10
 network 192.168.10.0 255.255.255.0 area 0
 filter-list prefix FILTER_IN in
```

#### 検証方法
* `show ospf neighbor` でネイバー状態が FULL であることを確認。`show route` で `10.99.0.0/16` が受信ルーティングテーブルから除外されていることを確認。

---

### 例7: FTD (FMC Managed) OSPFv2 MD5 認証と Passive-Interface 設定

#### 問題
FMC 管理下の FTD1 において、`Inside` インターフェイスで OSPFv2 MD5 認証（Key ID 1, Key `FTD_OSPF_KEY`）を有効化し、`DMZ` インターフェイスを Passive-interface に設定せよ。

#### 要件
* FMC GUI の Routing 設定画面より構成すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Devices > Device Management > Routing > OSPF > Interface タブを開く。
2. "Inside" インターフェイスを編集 ➔ "Authentication" タブ:
   - Type: MD5
   - Key ID: 1, Key: FTD_OSPF_KEY
3. "Passive Interfaces" タブを開く:
   - "DMZ" インターフェイスを Passive Interface 一覧に追加。
4. Save して Deploy を実行。
```

#### 検証方法
* FTD CLI で `show ospf interface Inside` を実行し `Message digest authentication enabled` を確認。`show ospf interface DMZ` で `No Hello Inbound/Outbound` (Passive) を確認。

---

### 例8: IOS IPv6 EIGRP 認証と Key-Chain 設定

#### 問題
R1 と R2 間で IPv6 EIGRP AS 200 を構成し、Key-chain `EIGRP6_KEYS` (Key ID 1, Password `IPV6_PASS`) を使用した MD5 認証を有効化せよ。

#### 要件
* `ipv6 authentication mode eigrp` および `ipv6 authentication key-chain eigrp` を使用すること。

#### 設定例

```bash
! ===== R1 Configuration =====
key chain EIGRP6_KEYS
 key 1
  key-string IPV6_PASS

interface GigabitEthernet0/0/1
 ipv6 address 2001:db8:100::1/64
 ipv6 eigrp 200
 ipv6 authentication mode eigrp 200 md5
 ipv6 authentication key-chain eigrp 200 EIGRP6_KEYS

ipv6 router eigrp 200
 router-id 1.1.1.1
 no shutdown
```

#### 検証方法
* `show ipv6 eigrp neighbors` を実行し、ネイバーの IPv6 Link-Local アドレスが表示され Up 状態であることを確認。

---

### 例9: IOS Route Filtering (Prefix-List, Route-Map, Passive-Interface Default)

#### 問題
R1 において OSPF プロセスの制御プレーンを強化せよ。全インターフェイスをデフォルトで Passive 化し、`Gi0/1` のみ開口せよ。また外部からのデフォルトルート受信を遮断する Prefix-list を適用せよ。

#### 要件
* `passive-interface default` を使用すること。

#### 設定例

```bash
! ===== R1 Configuration =====
ip prefix-list DENY_DEFAULT deny 0.0.0.0/0
ip prefix-list DENY_DEFAULT permit 0.0.0.0/0 le 32

router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface GigabitEthernet0/1
 distribute-list prefix DENY_DEFAULT in
```

#### 検証方法
* `show ip ospf` で `Passive interface default enabled` を確認。`show ip route` でデフォルトルートが注入されていないことを確認。

---

### 例10: Control Plane Policing (CoPP) による OSPF, EIGRP, BGP トラフィックの保護

#### 問題
ルーター R1 に対し、Control Plane Policing (CoPP) を構成せよ。OSPF (IP Protocol 89), EIGRP (IP Protocol 88), BGP (TCP 179) 宛てのコントロールプレーンパケットを 1 Mbps にレート制限（Police）し、溢れた悪意あるトラフィックをドロップせよ。

#### 要件
* `control-plane` 設定モードを使用すること。

#### 設定例

```bash
! ===== R1 Configuration =====
ip access-list extended ACL_ROUTING_PROTOCOLS
 permit ospf any any
 permit eigrp any any
 permit tcp any any eq 179
 permit tcp any eq 179 any

class-map CM_ROUTING_CP
 match access-list ACL_ROUTING_PROTOCOLS

policy-map PM_COPP_ROUTING
 class CM_ROUTING_CP
  police 1000000 conform-action transmit exceed-action drop

control-plane
 service-policy input PM_COPP_ROUTING
```

#### 検証方法
* `show policy-map control-plane` を実行し、`CM_ROUTING_CP` クラスで OSPF/BGP パケットがバインド・ポリス処理されていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ルーター R1 と R2 間で BGP MD5 認証を構成したが、間に存在する Cisco ASA ファイアウォールを通過する BGP 通信が Established に移行せず、R1 に以下のエラーが出力された。

`%TCP-6-BADAUTH: No MD5 digest from 192.168.2.2(179) to 192.168.1.1(49152)`

ASA 側で投入すべき正しい設定はどれか。

* A) `sysopt connection permit-vpn`
* B) `tcp-map BGP_MAP` で `tcp-options range 19 19 allow` を設定し、MPF で BGP トラフィックの `random-sequence-number` を `disable` に設定する。
* C) `inspect bgp` を policy-map に追加する。
* D) `same-security-traffic permit inter-interface` を設定する。

**正解**: **B**  
**解説**: BGP MD5 認証（TCP Option 19）は、TCP ヘッダーのオプションフィールドとシーケンス番号を使用してハッシュを計算します。ASA はデフォルトで未知の TCP オプション（Option 19 を含む）を削除し、TCP シーケンス番号をランダム化するため、ハッシュ検証が失敗します。`tcp-map` で Option 19 の透過を許可し、MPF で `set connection random-sequence-number disable` を設定することが必要です。

---

### 2. トラブルシュート問題

**問題**: ルーター R1 で `neighbor 192.168.12.2 ttl-security hops 1` を投入したところ、直ちに eBGP ピアとのセッションが切断された。対面ルーター R2 側では `neighbor 192.168.12.1 ebgp-multihop 2` が投入されていた。切断の直接原因として正しいものはどれか。

* A) GTSM (TTL Security) と eBGP Multihop は共に BGP パケットの送信 TTL 処理が異なり（GTSM は TTL 255、Multihop は設定された TTL で送信）、R1 が要求する Minimum TTL 254 を R2 の送信パケットが満たせなかったため。
* B) BGP パスワードが一致していないため。
* C) R1 の OSPF プロセスがダウンしたため。
* D) TCP ポート 179 がブロックされたため。

**正解**: **A**  
**解説**: **`ttl-security hops 1`** を設定した R1 は、受信パケットの TTL が **254 以上 (255 - 1)** であることを要求します。しかし、対面の R2 が `ebgp-multihop 2` を設定している場合、R2 は TTL 2 でパケットを送信するため、R1 到着時の TTL は 2 となり、GTSM 検証（TTL >= 254）に失敗して即座にドロップされます。両端で GTSM 設定を合致させる必要があります。

---

### 3. Design 問題

**問題**: データセンターの OSPF ネットワーク設計において、新規ルーター追加時の誤設定や悪意ある端末接続による不正な OSPF ネイバー形成をネットワーク全体で最小化したい。最も効果的な設計アプローチはどれか。

* A) OSPF 領域で ACL を適用し、すべての IP トラフィックをブロックする。
* B) OSPF プロセス内で **`passive-interface default`** を設定して全ポートをパッシブ化し、正規のネイバー対面ポートのみ **`no passive-interface`** で個別に開口させ、さらにエリア全域で **MD5 / HMAC-SHA 認証** を強制する。
* C) OSPF の Hello タイマーを 1 秒に変更する。
* D) OSPF の代わりに Static Route のみを使用する。

**正解**: **B**  
**解説**: **`passive-interface default`** により不必要なインターフェイスからの OSPF Hello 送出・受信を根本から止め、正規ポートのみを個別に許可＋MD5/HMAC-SHA 認証をバインドする設計が、Cisco の推奨するコントロールプレーンハードニングのベストプラクティスです。

---

### 4. 実装問題

**問題**: Cisco IOS において OSPFv3 (IPv6) のネイバー認証および暗号化を構成する際、適切なプロトコル記述はどれか。

* A) OSPFv3 は OSPFv2 と同様に `ip ospf message-digest-key` コマンドでパスワードを設定する。
* B) OSPFv3 は独自の認証ヘッダーを持たず、IPv6 ネイティブの **IPsec (AH または ESP)** を使用して `ipv6 ospf authentication ipsec` または `ipv6 ospf encryption ipsec` コマンドで SPI と鍵を指定する。
* C) OSPFv3 は TACACS+ サーバーによるユーザー認証が必須である。
* D) OSPFv3 は SSL/TLS トンネル上でしか動作しない。

**正解**: **B**  
**解説**: OSPFv3 はプロトコル仕様として独自のアセンブリ認証フィールドを持たず、**IPv6 の IPsec AH / ESP アーキテクチャを直接利用** してネイバー間の認証・暗号化を行います。

---

### 5. コンフィグ読解問題

**問題**: EIGRP においてキーローテーション（鍵変更）を行う際、旧キー（Key 1）から新キー（Key 2）へ無断絶で移行するための Key-chain 設定として正しい記述はどれか。

* A) Key 1 を即座に削除し、Key 2 のみを設定する。
* B) Key 1 の `send-lifetime` 終了時刻より前に、Key 2 の `accept-lifetime` および `send-lifetime` の開始時刻をオーバーラップ（重複）して設定する。
* C) Key-chain 名を毎回変更する。
* D) EIGRP プロセスを一度 `no router eigrp` で削除して再作成する。

**正解**: **B**  
**解説**: Key-chain によるキーローテーションでは、受領・送信ライフタイム（`accept-lifetime` / `send-lifetime`）を新旧キー間で意図的に重ね合わせる（Overlapping）ことで、隣接ルーター間でのキー非同期状態によるセッション切断（フラッピング）を防止できます。

---

## 🔗 参考リソース

* Cisco IOS IP Routing: OSPF Configuration Guide - OSPF Authentication
* Cisco ASA Series General Operations Configuration Guide: Routing Protocols
* Cisco Secure Firewall Management Center Device Configuration Guide: OSPF ＆ BGP
* RFC 5082: The Generalized TTL Security Mechanism (GTSM)
* Cisco Technical Notes: BGP MD5 Authentication through ASA Configuration Example
* Cisco Live: Securing the Control Plane in Enterprise Networks (BRKSEC-2001)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で Routing Protocol Security が出題された場合の構成選択フローチャート：
    * **「Configure OSPF MD5 authentication on Area 0, but disable on Gi0/2」** ➔ `router ospf` ➔ `area 0 authentication message-digest` ➔ `interface Gi0/2` ➔ `ip ospf authentication null`。
    * **「Allow BGP MD5 authentication through ASA」** ➔ `tcp-map` ➔ `tcp-options range 19 19 allow` ➔ MPF `set connection random-sequence-number disable` ＋ `set connection advanced-options`。
    * **「Protect BGP against remote CPU attacks using TTL」** ➔ `neighbor <IP> ttl-security hops <n>`。
    * **「Secure OSPFv3 with encryption」** ➔ `ipv6 ospf encryption ipsec spi <SPI> esp aes-cbc 256 <HEX> md5 <HEX>`。
* **図解（ルーティングプロトコル保護レイヤー マトリクス）**:
  ```text
  [ Incoming Routing Packet (OSPF / EIGRP / BGP) ]
                          │
                          ├── 1. Control Plane Policing (CoPP) ──► Rate-limit / Drop Flooding
                          │
                          ├── 2. BGP GTSM Check ───────────────► Drop Packets with TTL < Min_Incoming_TTL
                          │
                          ├── 3. Protocol Auth Check ──────────► Verify MD5 / HMAC-SHA / IPsec SPI
                          │
                          └── 4. Route Filtering ──────────────► Distribute-list / Prefix-list Validation
  ```
* **注意点**:
  * ラボ試験で OSPF/EIGRP/BGP 認証や ASA Normalization を設定した後は、必ず `show ip ospf interface`、`show ip eigrp neighbors`、`show ip bgp summary`、`show conn` を実行し、ネイバーが正常にエスタブリッシュされることを実機確認してください。

