---
layout: default
title: 4.4-802.1X-MAB
nav_order: 4
parent: 4.0-Identity-Management
---

# 4.4 AAA for network access with 802.1X and MAB using Cisco ISE

---

## 📘 概要

Cisco Identity Services Engine (ISE) を認証・認可サーバーとして利用するネットワークアクセス制御 (Network Access Control: NAC) において、**IEEE 802.1X** および **MAC Authentication Bypass (MAB)** は、ポートベースのアイデンティティ管理を実現する中核技術である。有線LANおよび無線LAN環境において、接続端末の身元（アイデンティティ）を検証し、企業ポリシーに基づいた動的なアクセス権限（VLAN、dACL、SGT、リダイレクトURL等）を付与するために使用される。

### 主な利用目的と適用シーン

1. **802.1X Authentication (IEEE 802.1X)**:
   * **概要**: EAP (Extensible Authentication Protocol) パケットを Ethernet フレーム (EAPoL: EAP over LAN) 上で透過転送し、サプリカント（端末）、オーセンティケータ（スイッチ/WLC）、認証サーバー（Cisco ISE）間で相互認証を実施する標準プロトコル。
   * **目的**: EAP-TLS（証明書ベースの強力な相互認証）、PEAP-MSCHAPv2（ユーザー名/パスワード認証）、EAP-FAST（PACベースの高速認証）を用いて、企業管理端末や従業員の厳格なアイデンティティ検証を行う。
   * **適用場面**: 企業ドメイン参加PC、BYOD端末、スマートフォンのセキュアアクセス制御。

2. **MAC Authentication Bypass (MAB)**:
   * **概要**: 802.1X サプリカントを実装していないネットワーク機器（プリンタ、IPカメラ、VoIP電話、IoTデバイスなど）に対し、MACアドレスをユーザー名およびパスワードとして ISE に送信して認証を行うフォールバック機能。
   * **目的**: 802.1X 非対応デバイスの自動識別と最低限必要なアクセス権限（特定VLANや制限されたACL）の自動割り当て。
   * **適用場面**: オフィス機器、ビル管理IoTデバイス、医療機器等のレイヤ2アクセス制御。

3. **統合アクセス制御モデル (IBNS 2.0 / C3PL)**:
   * **概要**: Cisco Catalyst スイッチにおける Control Plane Concurrent Architecture (C3PL) および IBNS 2.0 を使用し、802.1X と MAB の評価順序 (Order) および優先度 (Priority) を柔軟に定義。
   * **目的**: 802.1X タイムアウトの短縮（Flex-Auth）、MAB への迅速なフォールバック、および複数端末の同一ポート接続（Multi-Auth / Multi-Domain）制御。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **主要認証方式** | 802.1X (EAP-TLS, PEAP-MSCHAPv2, EAP-FAST, EAP-TTLS), MAB (MACアドレス認証), WebAuth (CWA/LWA) |
| **主要プロトコル** | EAPoL (IEEE 802.1X), RADIUS (UDP 1812/1813), RADIUS CoA (RFC 5176 / UDP 1700), RADSEC (TLS/TCP 2083) |
| **スイッチ動作モード** | Single-Host, Multi-Host, Multi-Domain (MDA), Multi-Auth (IBNS 2.0 / C3PL) |
| **制御コンポーネント** | Supplicant (端末), Authenticator (スイッチ/WLC), Authentication Server (Cisco ISE PSN) |
| **動的制御属性** | Dynamic VLAN (`Tunnel-Private-Group-ID`), dACL (`ip:inacl`), SGT (`cts:security-group-tag`), CWA (`url-redirect` / `url-redirect-acl`) |
| **ISE ポリシー構造** | Policy Set -> Authentication Policy (Allowed Protocols, Identity Source) -> Authorization Policy (Conditions, Authorization Profiles) |
| **制限事項・注意点** | MAB は MAC アドレス偽装のリスクがあるため Profiling または TrustSec/dACL による追加防御が推奨される。EAP-TLS では ISE 信頼 CA 証明書および CRL/OCSP 設定が必須。 |

---

## 🏗 動作原理

### 802.1X / MAB 認証の概念図とアーキテクチャ

```text
+------------------+         +----------------------+         +----------------------+
|    Supplicant    |         |    Authenticator     |         | Authentication Server|
| (PC / IP Camera) |         | (Catalyst Switch/WLC)|         |     (Cisco ISE)      |
+------------------+         +----------------------+         +----------------------+
         |                              |                              |
         | <======= EAPoL ============> |                              |
         |    (EtherType 0x888E)        |                              |
         |                              | <====== RADIUS (UDP 1812) => |
         |                              |    (EAP-Message / Access)    |
         |                              |                              |
         |                              | <==== RADIUS CoA (UDP 1700) =|
         |                              |    (Re-auth / Disconnect)    |
```

### 1. EAPoL と RADIUS のカプセル化転送
802.1X では、端末とスイッチ間は EAPoL (EAP over LAN) フレームでやり取りされ、スイッチは EAP パケットをそのまま RADIUS パケットの `EAP-Message` 属性 (Attribute 79) に格納して ISE に転送する。スイッチ自身は EAP の内部認証ペイロードを復号・処理せず、純粋な中継者（Authenticator）として動作する。

### 2. RADIUS 認証属性と MAB
MAB の場合、スイッチは端末の送信元 MAC アドレスを検出し、EAPoL を送出せずに即座に RADIUS `Access-Request` を生成する。
* `User-Name`: MAC アドレス
* `User-Password`: MAC アドレス
* `Service-Type`: `Call Check` (Attribute 6 = 10)
* `NAS-Port-Type`: `Ethernet` (Attribute 61 = 15) または `Wireless - IEEE 802.11` (19)

### 3. 動的アクセスコントロール (Authorization Payload)
ISE は認証に成功すると `Access-Accept` に以下のコントロール属性を付与して返信する。
* **Dynamic VLAN**: `Tunnel-Type = VLAN (13)`, `Tunnel-Medium-Type = 802 (6)`, `Tunnel-Private-Group-ID = VLAN_ID/VLAN_NAME`
* **dACL (Downloadable ACL)**: `Cisco-AVPair = "ip:inacl#1=permit ip any any"`
* **Security Group Tag (SGT)**: `Cisco-AVPair = "cts:security-group-tag=0004-00"`
* **Central Web Authentication (CWA)**: `Cisco-AVPair = "url-redirect=https://..."`, `Cisco-AVPair = "url-redirect-acl=REDIRECT_ACL"`

---

## ⚙ 動作シーケンス

### シーケンス 1: 802.1X (PEAP-MSCHAPv2 / EAP-TLS) 正常認証フロー

```text
Supplicant                   Authenticator (Switch)                 Cisco ISE (PSN)
    |                                   |                                   |
    |<-------- EAP-Request/Identity ----|                                   | (Periodic Request or Link-Up)
    |--------- EAP-Response/Identity -->|                                   |
    |                                   |------ RADIUS Access-Request ----->| (Contains EAP-Identity)
    |                                   |<----- RADIUS Access-Challenge ----| (EAP-Start TLS / PEAP)
    |<-------- EAP-Request (TLS Start) -|                                   |
    |                                   |                                   |
    |========== TLS Handshake (Server Cert Verification & Key Ex) ==========|
    |                                   |                                   |
    |<-- EAP-Request (MSCHAPv2/Cert) ---|                                   | (Encapsulated in RADIUS Challenge)
    |--- EAP-Response (MSCHAPv2/Cert) ->|                                   |
    |                                   |------ RADIUS Access-Request ----->|
    |                                   |<----- RADIUS Access-Accept -------| (Contains dACL/VLAN/SGT)
    |<-------- EAP-Success -------------|                                   |
    |                                   |                                   |
    | [Port Authorized / Dynamic Policies Applied on Switch Interface]      |
```

1. **Link Up / EAPoL Initiation**: 物理リンクが確立すると、スイッチは `EAP-Request/Identity` を送出。端末が `EAP-Response/Identity` を返答。
2. **RADIUS Encapsulation**: スイッチは `EAP-Response/Identity` を RADIUS `Access-Request` にカプセル化して ISE に送信。
3. **EAP Negotiation & TLS Tunnel**: ISE は対応する Allowed Protocols に基づき `EAP-TLS` または `PEAP` の開始を提案。TLS ハンドシェイクを実施し、暗号化トンネルを確立。
4. **Credential Verification**:
   * EAP-TLS: クライアント証明書を検証。
   * PEAP-MSCHAPv2: TLS トンネル内部で MSCHAPv2 認証を実行。
5. **Authorization & Enforce**: 認証成功後、ISE は合致した Authorization Profile の属性を返信。スイッチはポートを `Authorized` 状態に変更。

---

### シーケンス 2: MAB 認証および CWA リダイレクトフロー

```text
Unauthenticated Device            Authenticator (Switch)                 Cisco ISE (PSN)
    |                                   |                                   |
    |--- (Traffic / Link-Up) ---------->|                                   |
    | (No EAPoL Response -> Timeout)    |                                   |
    |                                   |--- MAB Trigger (MAC Lookup) ----->| (RADIUS Access-Request: User=MAC)
    |                                   |<-- Access-Accept (CWA Profile) ---| (url-redirect & url-redirect-acl)
    |                                   |                                   |
    | [Port Authorized with Redirect ACL / Session ID Generated]           |
    |                                   |                                   |
    |--- HTTP GET request ------------->|                                   |
    |                                   |-- Intercept & HTTP Redirect ----->| (Redirect to ISE Guest Portal URL)
    |                                   |                                   |
    |========== HTTP(S) Web Portal Authentication with ISE Client ==========|
    |                                   |                                   |
    |                                   |                                   |<-- Guest Auth Success
    |                                   |<-- RADIUS CoA (Re-authenticate) --| (CoA-Request)
    |                                   |--- RADIUS CoA ACK --------------->|
    |                                   |                                   |
    |                                   |--- New RADIUS Access-Request ---->|
    |                                   |<-- RADIUS Access-Accept ----------| (Final Authz: PermitAccess / VLAN / dACL)
    |                                   |                                   |
    | [Switch Clears Redirect ACL / Applies Full Access Policy]             |
```

1. **802.1X Timeout -> MAB Fallback**: 端末が EAPoL に応答しないため MAB に移行。
2. **MAB Request**: 端末の MAC アドレスを資格情報として RADIUS `Access-Request` を送出。
3. **CWA Authorization**: ISE は未認証ゲストとして判定し、`url-redirect-acl` と `url-redirect` を含む `Access-Accept` を返答。
4. **HTTP Intercept**: スイッチは Web 通信を横取りし、ISE ポータルヘ 302 Redirect。
5. **Web Login & CoA Trigger**: ポータル認証成功後、ISE は RADIUS CoA を送信して再認証を要求。
6. **Final Policy Enforce**: スイッチが新たな Access-Request を送信し、正規アクセス制限が適用される。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、802.1X および MAB の設定・トラブルシューティングは最も配点が高く、確実な動作検証が要求される主要 Blueprint 項目である。スイッチ側の IBNS 2.0 (C3PL) コンフィグと ISE 側の Policy Sets の不一致による認証失敗を迅速に特定する能力が問われる。

### Blueprintにおける超重要ポイント

1. **Authentication Order と Priority の精緻な制御**:
   * **Order (評価順序)**: ポートで最初に試行する認証方式（例: 802.1X を最初に試行し、タイムアウト後に MAB を試行）。
   * **Priority (優先度)**: 複数の認証結果が存在する場合にどちらを優先するか（例: 802.1X 認証成功は MAB 認証成功よりも常に優先）。
   * ラボ要件で「端末が802.1X非対応の場合のみMABに移行し、かつ802.1Xが成功した場合はMABを即座に上書きすること」といった指示が出される。

2. **Host Mode (ホストモード) の選択**:
   * **Single-Host**: 1ポートに1端末のみ許可。
   * **Multi-Host**: 最初に認証された端末が許可されると、同一ポートに接続された後続端末が無認証で通信可能。
   * **Multi-Domain (MDA)**: Data VLAN (PC) と Voice VLAN (IP Phone) を個別に認証・許可。
   * **Multi-Auth**: 同一ポートに接続されたすべての端末が個別に802.1X/MAB認証を実施。

3. **IBNS 2.0 / C3PL コンフィグ**:
   * `class-map` / `policy-map type control subscriber` を使用した現代的コンフィグ。

4. **Dynamic Authorization (CoA / dACL / Dynamic VLAN)**:
   * スイッチ側で `aaa server radius dynamic-author` が正しく設定されていないと CWA ポータル認証後の CoA Re-auth が失敗する。

5. **ISE Policy Set 構成**:
   * Authentication Policy と Authorization Policy の正しい評価ロジック。

---

### ラボ試験で狙われる設定ミスと落とし穴

1. **RADIUS Key (Shared Secret) の不一致**:
   * スイッチ側と ISE 側の Shared Secret が異なると認証不可。

2. **IBNS 2.0 タイマー設定不備による遅延**:
   * `dot1x timeout tx-period 3`, `dot1x max-reauth-req 2` 等でタイムアウトを短縮しないと MAB 移行に時間がかかる。

3. **CoA ポート (UDP 1700 vs 3799) の不一致**:
   * IOS-XE のデフォルト RADIUS CoA ポートは UDP 1700。

4. **WebAuth / CWA 用 Redirect ACL の誤り**:
   * ISE への通信 (DNS, DHCP, ISE IP) を **`deny`** (リダイレクト除外) し、Web 通信を **`permit`** (リダイレクト対象) に設定する。

---

## 🛠 設定方法

### 1. Cisco Catalyst スイッチ (IOS-XE) - IBNS 2.0 / C3PL コンフィグ例

802.1X を優先試行し、タイムアウト後に MAB にフォールバックする構成（Multi-Auth モード）。

```bash
! AAA & RADIUS Global Configuration
aaa new-model
aaa authentication dot1x default group ISE_RADIUS_GROUP
aaa authorization network default group ISE_RADIUS_GROUP
aaa accounting dot1x default start-stop group ISE_RADIUS_GROUP

aaa group server radius ISE_RADIUS_GROUP
 server name ISE_PSN1
 server name ISE_PSN2

radius server ISE_PSN1
 address ipv4 10.1.100.21 auth-port 1812 acct-port 1813
 key SecretKey123
radius server ISE_PSN2
 address ipv4 10.1.100.22 auth-port 1812 acct-port 1813
 key SecretKey123

! Dynamic Authorization (CoA)
aaa server radius dynamic-author
 client 10.1.100.21 server-key SecretKey123
 client 10.1.100.22 server-key SecretKey123
 port 1700

dot1x system-auth-control
device-tracking policy IPDT_POLICY
 tracking enable

! C3PL (IBNS 2.0) Policy Configuration
class-map type control subscriber match-all DOT1X_FAILED
 match authentication-status failure
 match authenticated-using dot1x

class-map type control subscriber match-all DOT1X_NO_RESP
 match result-type method dot1x agent-not-found

policy-map type control subscriber DOT1X_MAB_POLICY
 event session-started
  10 authorize
  20 authenticate using dot1x priority 10
 event authentication-failure
  10 class DOT1X_NO_RESP
   10 terminate dot1x
   20 authenticate using mab priority 20
  20 class DOT1X_FAILED
   10 terminate dot1x
   20 authenticate using mab priority 20
 event agent-found
  10 authenticate using dot1x priority 10

! Interface Configuration
interface GigabitEthernet1/0/11
 description === 802.1X & MAB Access Port (IBNS 2.0) ===
 switchport mode access
 switchport access vlan 999
 access-session host-mode multi-auth
 access-session port-control auto
 service-policy type control subscriber DOT1X_MAB_POLICY
 device-tracking attach-policy IPDT_POLICY
 ip access-group DEFAULT_RESTRICT_ACL in
 mab
 dot1x pae authenticator
 dot1x timeout tx-period 3
 dot1x max-reauth-req 2
```

---

### 2. Cisco ISE GUI 設定手順 (Policy Sets)

#### A. Network Device の追加
1. **Administration > Network Resources > Network Devices** に移動。
2. `Add` をクリックし、Name (`SW-CAMPUS-01`), IP Address (`10.1.10.10/32`) を入力。
3. `Authentication Settings` を有効化し、RADIUS Shared Secret (`SecretKey123`) を設定。

#### B. Authorization Profiles の作成
1. **Policy > Policy Elements > Results > Authorization > Authorization Profiles** に移動。
2. **従業員用 (EAP-TLS/PEAP)**:
   * Name: `Authz_Employee_Secured`
   * Dynamic VLAN: Tag ID `10`
   * Advanced Attributes: `cts:security-group-tag = 0004-4`
3. **MAB デバイス用 (Printers/IoT)**:
   * Name: `Authz_Printer_VLAN20`
   * Dynamic VLAN: Tag ID `20`
   * Downloadable ACL (dACL): `PERMIT_PRINTER_ONLY_DACL`
4. **CWA ゲスト用**:
   * Name: `Authz_CWA_Redirect`
   * Web Redirection (CWA): Central Web Auth, ACL: `REDIRECT_ACL`, Portal: `Sponsored Guest Portal`

#### C. Policy Set の構築
1. **Policy > Policy Sets** に移動し、`Campus_Access` Policy Set を作成。
2. **Authentication Policy**:
   * Rule 1 (802.1X): `Network Access:AuthenticationMethod EQUALS dot1x` -> `Active_Directory`
   * Rule 2 (MAB): `Network Access:AuthenticationMethod EQUALS mab` -> `Internal Endpoints`
3. **Authorization Policy**:
   * Rule 1: `IdentityGroup EQUALS Endpoint Identity Groups:Printers` -> `Authz_Printer_VLAN20`
   * Rule 2: `Network Access:AuthenticationMethod EQUALS dot1x` -> `Authz_Employee_Secured`
   * Default Rule: `DenyAccess`

---

## 🔍 検証コマンド

| 目的 | コマンド |
| --- | --- |
| **アクセスセッション状態確認** | <code>show access-session interface GigabitEthernet1/0/11</code> |
| **セッション詳細確認 (dACL/VLAN/SGT)** | <code>show access-session interface GigabitEthernet1/0/11 details</code> |
| **全ポート認証状態要約** | <code>show access-session summary</code> |
| **802.1X インターフェース状態** | <code>show dot1x interface GigabitEthernet1/0/11 details</code> |
| **認証統計情報** | <code>show dot1x statistics interface GigabitEthernet1/0/11</code> |
| **MAB 詳細状態確認** | <code>show mab interface GigabitEthernet1/0/11</code> |
| **ダウンロード済み dACL 確認** | <code>show ip access-lists dynamic</code> |
| **Device Tracking Database** | <code>show device-tracking database interface GigabitEthernet1/0/11</code> |
| **RADIUS サーバー応答状態** | <code>show radius server-group ISE_RADIUS_GROUP</code> |
| **RADIUS トラブルシュート** | <code>debug radius verbose</code> |
| **アクセスセッション デバッグ** | <code>debug access-session all</code> / <code>debug dot1x all</code> / <code>debug mab all</code> |
| **IBNS 2.0 C3PL デバッグ** | <code>debug subscriber control all</code> |


## 🚨 トラブルシュート

802.1X および MAB 導入時に発生しやすい障害現象、原因、確認コマンド、および解決策を下表にまとめます。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **802.1X 認証がタイムアウトし MAB に失敗する** | 端末が EAPoL に応答せず、かつスイッチの `tx-period` / `max-reauth-req` が長すぎてタイムアウトまでに時間がかかりすぎている。 | <code>show dot1x interface Gig1/0/11 details</code> | `dot1x timeout tx-period 3` および `dot1x max-reauth-req 2` を設定し 802.1X タイムアウトを短縮する。 |
| **MAB 認証が `15039 Selected Identity Source not found` で失敗する** | ISE の Authentication Policy で MAB 用の Identity Source として Endpoints または AD が正しく紐付けられていない。 | ISE Live Logs (Failure Reason 15039) | ISE Authentication Policy で `If User not found` を `Continue` に設定し、Internal Endpoints を参照させる。 |
| **EAP-TLS 認証で `12514 EAP-TLS failed SSL/TLS handshake` が発生する** | 端末のクライアント証明書を検証する Root CA 証明書が ISE Trusted Certificates に登録されていないか、有効期限切れ。 | ISE Live Logs (Failure Reason 12514 / 12508) | ISE Administration > Certificates にて Root/Intermediate CA 証明書をインポートし `Trust for client authentication` を有効化。 |
| **CWA リダイレクト後、ポータルログイン成功してもアクセス制限が解除されない** | スイッチ側で RADIUS CoA (UDP 1700) が拒否されているか、Shared Secret が不一致で Re-authentication CoA が失敗している。 | <code>show access-session interface Gig1/0/11 details</code> (Status が Redirect のまま), <code>debug radius verbose</code> | `aaa server radius dynamic-author` 設定で ISE PSN IP と Shared Secret を一致させ、Port 1700 を確認する。 |
| **ISE から配布された dACL の適用にスイッチ側で失敗する (%ACL-3-ACLNOTFOUND)** | ISE 上の dACL 構文エラー（例: `permit ip any any` の行頭スペースや無効な構文）またはスイッチのメモリ不足。 | <code>show ip access-lists dynamic</code>, <code>show logging \| inc ACL</code> | ISE の Authorization Elements > dACL の記述文法を確認し、IP アドレスやポートの誤記を修正する。 |
| **Voice VLAN (IP Phone) と PC の双方接続時に IP Phone の通信が遮断される** | スイッチポートの Host Mode が `single-host` のままになっており、2つ目の MAC アドレス検出で Violation が発生。 | <code>show access-session interface Gig1/0/11</code> | インターフェースの Host Mode を `multi-domain` (MDA) または `multi-auth` に変更する。 |
| **Windows PC 起動時に 802.1X が実行されず MAB で認証されてしまう** | Windows の `Wired AutoConfig` サービスが停止しているか、NIC プロパティで 802.1X が有効化されていない。 | 端末側 `services.msc` (dot3svc サービス状態) | `Wired AutoConfig` サービスを自動起動に設定し、ネットワークアダプタのプロパティで 802.1X を有効化。 |

---

## ⚠ 制限事項

### 1. 802.1X / MAB のプロトコル制限
* **MAB のセキュリティ限界**: MAB は MAC アドレスのみを認証情報とするため、攻撃者による MAC アドレス偽装 (MAC Spoofing) を防ぐことができない。Profiling や ISE pxGrid ANC、TrustSec との組み合わせが必須。
* **EAP-TLS と MTU / パケット断片化**: EAP-TLS で巨大な証明書チェーンを送受信する際、EAPoL パケットが PMTU を超えてフラグメント化され、途中のネットワーク機器でドロップされる可能性がある。

### 2. スイッチ / Hardware 制限
* **Host Mode の挙動差異**: `multi-host` モードでは最初の1台しか認証されないため、後続デバイスのアイデンティティ追跡や dACL / SGT 適用が不可能になる。
* **IBNS 1.0 vs 2.0 の互換性**: IOS-XE 16.9 以降では IBNS 2.0 (C3PL) が標準。レガシーコマンド (`authentication order`) と C3PL コマンドの混在は設定の矛盾や予期せぬ挙動を引き起こす。

---

## 🔄 他技術との関連

| 関連技術 | 相互作用・関係性 |
| --- | --- |
| **Cisco TrustSec (SGT)** | 802.1X / MAB 認証成功時に ISE から `cts:security-group-tag` 属性を返却し、スイッチポートに SGT (Security Group Tag) を動的割り当て。SGACL によるマイクロセグメンテーションを実現。 |
| **Device Tracking (IPDT)** | 認証成功後の端末 IP/MAC アドレスマッピングを維持・監視。CWA リダイレクトや ISE Profiling (DHCP/HTTP Probe) の動作基盤。 |
| **DHCP Snooping & DAI** | MAB や 802.1X でポートが `Authorized` になった後、MAC アドレス偽装や不正 IP 設定を防ぐための L2 防御策。 |
| **Cisco ISE Profiling** | MAB デバイスの属性（DHCP OUI, CDP/LLDP, MAC OUI）を収集し、単なる MAC 認証を超えた端末種別判定を行う。 |
| **pxGrid / ANC (Adaptive Network Control)** | ISE 側で異常な挙動を検知した際、RADIUS CoA を通じて対象ポートを Quarantine VLAN や Shutdown 状態へ動的に変更。 |

---

## 🧩 比較表

### 認証方式の比較 (802.1X vs MAB vs CWA)

| 比較項目 | 802.1X (EAP-TLS / PEAP) | MAB (MAC Auth Bypass) | CWA (Central Web Auth) |
| --- | --- | --- | --- |
| **対象端末** | PC, スマホ, 企業管理端末 | プリンタ, IPカメラ, IoT, IP Phone | ゲストユーザー, BYOD 未登録端末 |
| **認証強度** | 最高（暗号化 / 証明書 / 資格情報） | 低（MAC アドレスのみ） | 中（Web ユーザー名 / パスワード） |
| **サプリカント要否** | **必要** (Windows Wired AutoConfig 等) | **不要** | **不要** (Web ブラウザが必要) |
| **プロトコル** | EAPoL (L2) + RADIUS (UDP 1812) | RADIUS (UDP 1812) | MAB (L2) + HTTP/HTTPS (L7) + RADIUS CoA |
| **認証情報** | クライアント証明書 / AD 認証情報 | MAC アドレス | ゲストポータルアカウント credentials |
| **主な用途** | 従業員デバイスのセキュアアクセス | 802.1X 非対応デバイスの自動アクセス | 一時ゲスト・来訪者のインターネット接続 |

---

## 💡 ベストプラクティス

1. **802.1X タイマー最適化**:
   * ディフォルトの 802.1X タイムアウト (30秒 × 3回) では MAB 移行に 90 秒以上かかるため、`dot1x timeout tx-period 3` および `dot1x max-reauth-req 2` を設定し、MAB 移行を 6〜10 秒以内に完了させる。
2. **IBNS 2.0 (C3PL) への統一**:
   * IOS-XE 搭載スイッチではレガシー `authentication` コマンドを避け、全拠点でIBNS 2.0 テンプレートポリシーを標準化する。
3. **MAB と Profiling / dACL の併用**:
   * MAB 単体では MAC 偽装のリスクがあるため、ISE Profiling 判定（例: `Printer` であること）と dACL（必要最小限の通信のみ許可）を必ず組み合わせる。
4. **RADIUS 高可用性 (HA) 設計**:
   * スイッチに複数の ISE PSN を登録し、`radius-server deadtime 15` を設定して不応答 PSN を迅速に切り離す。


## 📝 ラボ学習・設定サンプル例

### シナリオ 1: 基本的な 802.1X (PEAP) 認証と Dynamic VLAN 割り当て (C3PL)

#### 問題
Catalyst スイッチ GigabitEthernet1/0/1 上で 802.1X 認証を有効化し、認証成功時に ISE から動的に VLAN 10 (Employee VLAN) を割り当てるよう設定しなさい。

#### 要件
* C3PL (IBNS 2.0) 構文を使用すること。
* RADIUS サーバーとして ISE PSN (10.1.100.21) を使用し、共通鍵は `SecretKey123` とする。
* デフォルト VLAN は VLAN 999 (Unauthenticated) とする。

#### 設定例

```bash
! RADIUS Configuration
aaa new-model
aaa authentication dot1x default group ISE_GROUP
aaa authorization network default group ISE_GROUP

aaa group server radius ISE_GROUP
 server name ISE_PSN1

radius server ISE_PSN1
 address ipv4 10.1.100.21 auth-port 1812 acct-port 1813
 key SecretKey123

dot1x system-auth-control

! C3PL Policy Configuration
policy-map type control subscriber DOT1X_ONLY_POLICY
 event session-started
  10 authorize
  20 authenticate using dot1x priority 10

! Interface Configuration
interface GigabitEthernet1/0/1
 description === 802.1X Dynamic VLAN Port ===
 switchport mode access
 switchport access vlan 999
 access-session host-mode single-host
 access-session port-control auto
 service-policy type control subscriber DOT1X_ONLY_POLICY
 dot1x pae authenticator
```

#### 検証方法
```bash
show access-session interface GigabitEthernet1/0/1 details
show vlan id 10
```

---

### シナリオ 2: MAB (MAC Authentication Bypass) と dACL 動的適用

#### 問題
ポート GigabitEthernet1/0/2 にて MAB 認証を構成し、プリンター端末が認証された際、ISE から動的 ACL (dACL) をダウンロードして適用しなさい。

#### 要件
* MAB 認証のみを有効化。
* ISE 上で dACL `PERMIT_PRINT_DACL` を定義して割り当てること。

#### 設定例

```bash
! AAA & MAB Global Configuration
aaa new-model
aaa authentication dot1x default group ISE_GROUP
aaa authorization network default group ISE_GROUP

aaa group server radius ISE_GROUP
 server name ISE_PSN1

radius server ISE_PSN1
 address ipv4 10.1.100.21 auth-port 1812 acct-port 1813
 key SecretKey123

! C3PL Policy Configuration for MAB
policy-map type control subscriber MAB_ONLY_POLICY
 event session-started
  10 authorize
  20 authenticate using mab priority 10

! Interface Configuration
interface GigabitEthernet1/0/2
 description === MAB Printer Port ===
 switchport mode access
 switchport access vlan 20
 access-session host-mode single-host
 access-session port-control auto
 service-policy type control subscriber MAB_ONLY_POLICY
 mab
```

#### 検証方法
```bash
show access-session interface GigabitEthernet1/0/2 details
show ip access-lists dynamic
```

---

### シナリオ 3: 802.1X から MAB への自動フォールバックとタイマー最適化

#### 問題
ポート GigabitEthernet1/0/3 にて 802.1X 認証を優先し、応答がない場合は 10 秒以内に MAB 認証へ自動フォールバックするよう構成しなさい。

#### 要件
* 802.1X タイムアウト時間を短縮すること (`tx-period 3`, `max-reauth-req 2`)。
* C3PL を使用して認証失敗/無応答イベントを検知し MAB を開始すること。

#### 設定例

```bash
class-map type control subscriber match-all DOT1X_NO_RESP
 match result-type method dot1x agent-not-found

class-map type control subscriber match-all DOT1X_FAILED
 match authentication-status failure
 match authenticated-using dot1x

policy-map type control subscriber DOT1X_MAB_FALLBACK_POLICY
 event session-started
  10 authorize
  20 authenticate using dot1x priority 10
 event authentication-failure
  10 class DOT1X_NO_RESP
   10 terminate dot1x
   20 authenticate using mab priority 20
  20 class DOT1X_FAILED
   10 terminate dot1x
   20 authenticate using mab priority 20
 event agent-found
  10 authenticate using dot1x priority 10

interface GigabitEthernet1/0/3
 switchport mode access
 switchport access vlan 999
 access-session host-mode single-host
 access-session port-control auto
 service-policy type control subscriber DOT1X_MAB_FALLBACK_POLICY
 mab
 dot1x pae authenticator
 dot1x timeout tx-period 3
 dot1x max-reauth-req 2
```

#### 検証方法
```bash
show access-session interface GigabitEthernet1/0/3 details
```

---

### シナリオ 4: Multi-Domain (MDA) モードによる IP Phone と PC の個別認証

#### 問題
ポート GigabitEthernet1/0/4 に IP Phone と PC が接続されている。IP Phone は Voice VLAN 100 で MAB 認証、PC は Data VLAN 10 で 802.1X 認証を個別に行うよう設定しなさい。

#### 要件
* Host Mode を `multi-domain` に設定すること。

#### 設定例

```bash
interface GigabitEthernet1/0/4
 description === IP Phone & PC MDA Port ===
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 100
 access-session host-mode multi-domain
 access-session port-control auto
 service-policy type control subscriber DOT1X_MAB_FALLBACK_POLICY
 mab
 dot1x pae authenticator
```

#### 検証方法
```bash
show access-session interface GigabitEthernet1/0/4 details
```

---

### シナリオ 5: Multi-Auth モードによるハブ配下複数端末の個別認証

#### 問題
ポート GigabitEthernet1/0/5 の配下にハブが接続され、複数の PC が接続されている。各端末が個別に 802.1X 認証を実施できるように構成しなさい。

#### 要件
* Host Mode を `multi-auth` に設定すること。

#### 設定例

```bash
interface GigabitEthernet1/0/5
 description === Multi-Auth Hub Port ===
 switchport mode access
 switchport access vlan 10
 access-session host-mode multi-auth
 access-session port-control auto
 service-policy type control subscriber DOT1X_ONLY_POLICY
 dot1x pae authenticator
```

#### 検証方法
```bash
show access-session interface GigabitEthernet1/0/5 summary
```

---

### シナリオ 6: Central Web Authentication (CWA) と Dynamic Authorization (CoA)

#### 問題
未登録端末に対し MAB 実行後、CWA ポータルへリダイレクトし、ユーザー認証成功後に CoA を受託してアクセス許可に変更するコンフィグを作成しなさい。

#### 要件
* スイッチ側で `aaa server radius dynamic-author` (Port 1700) を有効化すること。
* Redirect ACL `REDIRECT_ACL` を定義すること。

#### 設定例

```bash
! CoA Config
aaa server radius dynamic-author
 client 10.1.100.21 server-key SecretKey123
 port 1700

! Redirect ACL Definition
ip access-list extended REDIRECT_ACL
 deny udp any any eq domain
 deny udp any any eq bootps
 deny udp any any eq bootpc
 deny ip any host 10.1.100.21
 permit tcp any any eq www
 permit tcp any any eq 443

interface GigabitEthernet1/0/6
 switchport mode access
 switchport access vlan 999
 access-session host-mode single-host
 access-session port-control auto
 service-policy type control subscriber DOT1X_MAB_FALLBACK_POLICY
 ip access-group REDIRECT_ACL in
 mab
 dot1x pae authenticator
```

#### 検証方法
```bash
show access-session interface GigabitEthernet1/0/6 details
```

---

### シナリオ 7: EAP-TLS 認証と TrustSec SGT 動的割り当て

#### 問題
ポート GigabitEthernet1/0/7 にて EAP-TLS 認証を実行し、ISE から SGT 4 (Employees) を動的に割り当てるよう設定しなさい。

#### 要件
* スイッチ側で TrustSec を有効化すること。

#### 設定例

```bash
cts credentials id SW-CAMPUS-01 password SecretKey123
cts role-based sgacl monitor

interface GigabitEthernet1/0/7
 switchport mode access
 switchport access vlan 10
 access-session host-mode single-host
 access-session port-control auto
 service-policy type control subscriber DOT1X_ONLY_POLICY
 dot1x pae authenticator
```

#### 検証方法
```bash
show access-session interface GigabitEthernet1/0/7 details
show cts interface GigabitEthernet1/0/7
```

---

### シナリオ 8: ISE 障害時のローカルフォールバック (Inaccessible Authentication Bypass)

#### 問題
RADIUS サーバーが不応答の場合、アクセスポートを自動的に Critical VLAN 99 に割り当てて通信を救済しなさい。

#### 要件
* C3PL イベント `server-unreachable` を利用すること。

#### 設定例

```bash
radius-server deadtime 15

class-map type control subscriber match-all RADIUS_DEAD
 match result-type method dot1x server-unreachable

policy-map type control subscriber CRITICAL_POLICY
 event server-unreachable
  10 class RADIUS_DEAD
   10 authorize vlan 99

interface GigabitEthernet1/0/8
 switchport mode access
 switchport access vlan 10
 service-policy type control subscriber CRITICAL_POLICY
```

#### 検証方法
```bash
show radius server-group ISE_GROUP
show access-session interface GigabitEthernet1/0/8 details
```

---

### シナリオ 9: Device Sensor (CDP/LLDP/DHCP) と ISE Profiling 連動 MAB

#### 問題
スイッチポート上で Device Sensor を有効化し、端末の属性情報を ISE へ送信しなさい。

#### 設定例

```bash
device-sensor filter-list cdp tlv name CDP_LIST
 tlv name device-name
 tlv name platform-type

device-sensor filter-spec cdp include list CDP_LIST
device-sensor accounting
device-sensor notify group-channel
```

#### 検証方法
```bash
show device-sensor cache
```

---

### シナリオ 10: RADSEC (TLS over TCP 2083) による RADIUS 通信の暗号化

#### 問題
Catalyst スイッチと ISE 間の RADIUS 通信を TLS (RADSEC) で暗号化しなさい。

#### 設定例

```bash
crypto pki trustpoint RADSEC_TP
 enrollment url https://ca.example.com
 revocation-check crl

radius server RADSEC_ISE
 address ipv4 10.1.100.21 tls port 2083
 trustpoint RADSEC_TP
 key SecretKey123
```

#### 検証方法
```bash
show radius server-group RADSEC_GROUP
```


## ❓ 想定試験問題

### 質問 1 (コンフィグ読解)
以下の IOS-XE C3PL コンフィグにおいて、端末が 802.1X に応答しなかった場合、MAB 認証が開始されるまでに何秒かかるか計算し、設定の改善点を説明せよ。

```bash
dot1x timeout tx-period 30
dot1x max-reauth-req 3
```

* **解答・解説**:
  * **計算**: 802.1X の再試行間隔 `tx-period` (30秒) × `max-reauth-req` (3回) = **90秒**。
  * **解説**: サプリカント非搭載端末（プリンター等）が接続された際、スイッチが 802.1X 応答を諦めて MAB に切り替わるまで 90 秒間通信不能となる。ラボ試験の迅速な認証要件を満たすには `dot1x timeout tx-period 3` および `dot1x max-reauth-req 2` (計 6 秒) 程度に最適化する必要がある。

---

### 質問 2 (トラブルシューティング)
CWA (Central Web Authentication) 構成において、端末が MAB 認証に成功しリダイレクト URL を受領したが、Web ポータルでログイン後にアクセス制限が解除されず依然としてリダイレクトされる。考えられるスイッチおよび ISE 側の原因を3つ挙げよ。

* **解答・解説**:
  1. **RADIUS CoA ポートの遮断または不一致**: スイッチ側の `aaa server radius dynamic-author` (Port 1700) 設定漏れ、またはファイアウォールによる UDP 1700 ドロップ。
  2. **CoA Shared Secret の不一致**: dynamic-author の `server-key` と ISE の Network Device 共通鍵の相違。
  3. **Redirect ACL の記述ミス**: ISE への HTTPS/HTTP 通信や DNS/DHCP が `deny` (リダイレクト除外) されておらず、CoA 後の再認証通信が阻害されている。

---

### 質問 3 (Design)
大規模企業ネットワークにおいて、IP Phone (Voice VLAN 100) と PC (Data VLAN 10) が同一スイッチポートに接続されている。セキュリティ要件として「PC のみ 802.1X EAP-TLS 認証を適用し、IP Phone は MAB 認証とする。また PC と IP Phone は異なる SGT を動的に割り当てる」設計を策定せよ。

* **解答・解説**:
  * **Host Mode**: `multi-domain` (MDA) モードをポートに適用。
  * **スイッチ側コンフィグ**: C3PL ポリシーで 802.1X と MAB の双方を有効化。
  * **ISE 側設計**:
    * Auth Policy: 802.1X (EAP-TLS) と MAB (Internal Endpoints) を定義。
    * Authz Policy Rule 1: MAB + IP Phone Profile -> Voice VLAN 100 + SGT_Voice (SGT 3).
    * Authz Policy Rule 2: 802.1X + Cert Domain -> Data VLAN 10 + SGT_Employee (SGT 4).

---

### 質問 4 (実装)
ISE の Authorization Profile で dACL を指定した際、スイッチ側で `%ACL-3-ACLNOTFOUND` エラーが発生し通信が全遮断された。原因とログ確認手順を述べよ。

* **解答・解説**:
  * **原因**: ISE で定義した dACL の構文にエラーがある（例: 先頭に不要なスペース、誤った IP アドレス構文、存在しないサービスポートなど）。スイッチは不完全な dACL のダウンロードを拒否し、安全のため全遮断に倒れる。
  * **手順**: スイッチ上で `show ip access-lists dynamic` を実行し dACL が展開されているか確認。`show logging | inc ACL` でスイッチのログを解析し、ISE 上の dACL 構文を正しく修正する。

---

### 質問 5 (Troubleshooting)
802.1X EAP-TLS 認証において、端末側で「サーバー証明書が信頼できません」と表示され認証が中断した。ISE 側で必要な設定を説明せよ。

* **解答・解説**:
  * **原因**: ISE の EAP 認証用サーバー証明書（EAP System Certificate）を発行した Root CA / Intermediate CA 証明書が、端末の信頼されたルート証明書ストアに登録されていないか、証明書チェーンが不完全。
  * **対処**: ISE の **Administration > Certificates > Trusted Certificates** にて中間CAを含む証明書チェーンを正しくインポートし、端末側にも該当 Root CA を GPO 等で事前配布する。

---

## 🔗 参考リソース

* Cisco Identity Services Engine Administrator Guide - Configure 802.1X and MAB
* Cisco Catalyst 9000 Series Switches Security Configuration Guide - Configuring IEEE 802.1X Access
* Cisco Technical Notes - IBNS 2.0 Deployment Guide for Enterprise Networks
* Cisco Live BRKSEC-2030 - Advanced Wired 802.1X and MAB Deployment with ISE
* Cisco Validated Design (CVD) - Software-Defined Access (SD-Access) Deployment Guide

---

## 📝 補足（Notes）

### IBNS 2.0 (C3PL) 状態遷移まとめ
* `session-started`: ポートリンクアップ時に最初に実行されるイベント。
* `authentication-failure`: 認証失敗時またはタイムアウト時。`class-map` により失敗原因を判定して分流。
* `agent-found`: 途中で EAPoL パケットを検出した際に 802.1X へ割り込むイベント。


