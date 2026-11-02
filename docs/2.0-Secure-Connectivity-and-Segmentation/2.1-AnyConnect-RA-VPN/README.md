---
layout: default
title: 2.1-AnyConnect-RA-VPN
nav_order: 1
parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.1 Cisco AnyConnect client-based, remote-access VPN technologies on Cisco ASA, Cisco FTD, and Cisco routers

Cisco セキュリティアーキテクチャにおける **Cisco AnyConnect Secure Mobility Client**（現在の Cisco Secure Client）を用いた Client-Based Remote Access VPN（RA-VPN）技術は、社外やリモート環境に存在するクライアント端末（Windows, macOS, Linux, iOS, Android 等）と組織の内部ネットワーク間において、SSL/TLS または IKEv2/IPsec 暗号化プロトコルを用いて安全な IP トンネルを確立する高度なリモートアクセスソリューションです。

CCIE Security v6.1 Blueprint **2.1 Cisco AnyConnect client-based, remote-access VPN technologies on Cisco ASA, Cisco FTD, and Cisco routers** では、**Cisco ASA（Adaptive Security Appliance）**、**Cisco FTD（Firepower Threat Defense / Secure Firewall）**、および **Cisco IOS-XE ルーター（FlexVPN Remote Access）** の 3 大プラットフォームにおける AnyConnect VPN ヘッドエンドの構築、高度な認証・認可基盤（AAA, PKI/証明書認証, SAML 2.0 SSO, SAML + Certificate 二元認証, ISE CoA/DACL/SGT 割り当て）、Dynamic Access Policies (DAP)、スプリットトンネリング (Split-Tunneling)、XML 接続プロファイル（Always-On, Trusted Network Detection [TND], HostScan / Posture）、およびトラブルシューティング能力が多角的に評価されます。

---

## 📘 概要

* **機能概要**:
  * **AnyConnect Secure Mobility Client (Cisco Secure Client)**: 端末上に常駐し、ヘッドエンドアプライアンスとの間で SSL/TLS (HTTPS Port 443) または IKEv2/IPsec (UDP Port 500 / 4500) 暗号化トンネルを確立してフル IP 接続を提供するクライアント型 VPN ソリューション。
  * **プラットフォーム別ヘッドエンド実装**:
    * **Cisco ASA**: WebVPN / SSL VPN および IKEv2 Remote Access をネイティブ CLI / ASDM で実装。最も機能が成熟しており、DAP (Dynamic Access Policies)、Custom Attributes、HostScan、Web-launch、Advanced XML Profiles をサポート。
    * **Cisco FTD (FMC / FDM 管理)**: Firepower Threat Defense 上で RAVPN ポータルおよび AnyConnect 機能を動作。FMC (Firepower Management Center) または FDM を介して SSL/IKEv2 VPN、SAML 2.0 認証、ISE Posture (CoA)、DACL/SGT 割り当てを構成。
    * **Cisco IOS / IOS-XE Router**: FlexVPN フレームワーク（IKEv2 / IPsec ＋ Virtual-Template / Virtual-Access インターフェイス）を用いて AnyConnect IKEv2 (および SSL WebVPN) 接続を処理。EAP-MSCHAPv2、EAP-TLS、RADIUS 属性アサインに対応。
  * **高度な認証・認可・ポスチャ機構**:
    * **認証 (Authentication)**: AAA (RADIUS/TACACS+), Local DB, Client Certificate (PKI), SAML 2.0 (Azure AD, Okta, Duo, Ping Identity), Multiple/Dual Authentication (Certificate + SAML/AAA)。
    * **認可 (Authorization)**: Group-Policy、Tunnel-Group / Connection Profile、DAP (Dynamic Access Policies)、ISE Authorization (DACL, VLAN ID, Cisco SGT)。
    * **ポスチャ (Posture/HostScan)**: AnyConnect HostScan モジュールまたは ISE Posture Agent (AnyConnect ISE Posture Module) による端末の安全衛生状態（OS パッチ、アンチウイルス、ファイル、レジストリ）の検証と動的アクセス制御。
* **利用目的**:
  * リモートワーカー、モバイルユーザー、サードパーティベンダーに対し、境界型ファイアウォールを透過して社内リソースへのセキュリティ保護されたリモートアクセスを提供する。
  * SAML 2.0 / MFA および ISE ポスチャ評価を組み合わせることで、Zero Trust Network Access (ZTNA) の第一歩となるアイデンティティ＆デバイス状態に基づく動的アクセス制御を実現する。
* **どのような場面で利用するか**:
  * 全社のリモートワークアクセス基盤。
  * クラウド IDP との統合によるシングルサインオン (SSO) ＆ 多要素認証 (MFA) の導入。
  * 端末のパッチ適用状態やセキュリティソフト稼働状態に応じた動的 VLAN / DACL 変更。

---

## 🔑 要点

| 項目 | Cisco ASA | Cisco FTD (FMC Managed) | Cisco IOS-XE Router (FlexVPN) |
| :--- | :--- | :--- | :--- |
| **サポートプロトコル** | **SSL/TLS (HTTPS 443)** ＆ **IKEv2/IPsec** | **SSL/TLS (HTTPS 443)** ＆ **IKEv2/IPsec** | **IKEv2/IPsec** (主) ＆ **SSL (WebVPN)** |
| **管理・設定インターフェイス** | CLI ＆ ASDM | FMC GUI (Objects / Devices RAVPN) | CLI (`crypto ikev2`, `crypto ipsec`, FlexVPN) |
| **高度制御機能** | **DAP (Dynamic Access Policies)**、HostScan、Lua Custom Attributes | ISE Posture / CoA、DACL、SGT、SAML 2.0 | RADIUS VSA 割当、FlexVPN AAA Authorization |
| **SAML 2.0 SSO 対応** | ASA 9.7(1) 以降対応 | FTD 6.7.0 以降 (FMC 統合) 対応 | 制限あり (FlexVPN 経由での制約) |
| **二元認証 (Dual Auth)** | 対応 (Certificate ＋ AAA / SAML) | 対応 (Certificate ＋ SAML / AAA) | 対応 (EAP-TLS / EAP-MSCHAPv2) |
| **IP アドレス割り当て** | Local Pool, RADIUS/ISE, DHCP | Local Pool, RADIUS/ISE, DHCP | Local Pool, RADIUS/ISE, DHCP |
| **スプリットトンネル** | TunnelAll, TunnelSpec, ExcludeSpec | TunnelAll, TunnelSpec, ExcludeSpec | ACL による Split-Tunnel 属性配布 |
| **主用途** | 伝統的・高度な機能要求を満たす大規模 RAVPN | FMC で一元管理される統合 NGFW/RAVPN エッジ | ルーター統合エッジ / FlexVPN インフラ |

---

## 🏗 動作原理

### 1. AnyConnect SSL vs IKEv2 トンネル構造と接続フロー

AnyConnect は端末とヘッドエンド間で **Control Channel（制御チャネル）** と **Data Channel（データチャネル）** を確立します。

* **SSL/TLS モード**:
  * **Control/Data Channel**: 最初は TCP 443 (TLS 1.2/1.3) 上で制御チャネルとデータチャネルを確立。
  * **DTLS (Datagram TLS)**: 通信効率向上のため、バックグラウンドで UDP 443 (DTLS v1.2) トンネルの確立を試行。成功した場合、データトラフィックは低遅延な DTLS 経由に自動昇格。DTLS がブロックされた場合は TCP/TLS にフォールバック。
* **IKEv2/IPsec モード**:
  * UDP 500 / UDP 4500 (NAT-Traversal) を使用し、IKEv2 SA (Security Association) および IPsec ESP SA を確立。次世代暗号化 (Suite B / NSA Suite: AES-GCM, SHA-384, ECDH) をサポート。

```text
[ Client (AnyConnect) ] ────(1) TLS / IKEv2 Handshake (Port 443 / 500) ────► [ Headend (ASA / FTD / Router) ]
           │                                                                               │
           ├────(2) Authentication Phase (SAML / Certificate / AAA) ───────────────────────┤
           │                                                                               │
           │ ◄──(3) Group-Policy & DAP / ISE Push (IP Address, Split-Tunnel, XML Profile) ─┤
           │                                                                               │
           ├─── (4) Data Tunnel Established ───────────────────────────────────────────────┤
           │    ├── Primary: DTLS (UDP 443) or IPsec ESP (UDP 4500)                      │
           │    └── Fallback: TLS (TCP 443)                                                │
           ▼                                                                               ▼
  [ Encrypted IP Virtual NIC Adapter ] ══════════════════════════════════════ [ Internal Corporate Network ]
```

---

### 2. SAML 2.0 シングルサインオン (SSO) ＆ MFA 動作シーケンス

AnyConnect における SAML 認証の動作フロー：

```text
[ Client PC ]                          [ AnyConnect Headend ]                 [ Identity Provider (IdP) ]
     │                                           │                                       │
     ├──────(1) Initiate VPN Connect ───────────►│                                       │
     │                                           │                                       │
     │◄─────(2) SAML Request (Redirect) ─────────┤                                       │
     │                                                                                   │
     ├──────(3) HTTP GET / Auth Request (SAML Request) ─────────────────────────────────►│
     │                                                                                   │
     │◄─────(4) Display Login Screen / Challenge MFA ────────────────────────────────────┤
     │                                                                                   │
     ├──────(5) Submit Credentials & Complete MFA ──────────────────────────────────────►│
     │                                                                                   │
     │◄─────(6) Return Signed SAML Assertion (Response) ─────────────────────────────────┤
     │                                                                                   │
     ├──────(7) Post SAML Assertion ────────────►│                                       │
     │                                           │ ── Validate SAML Signature
     │                                           │ ── Map Group-Policy & DAP
     │◄─────(8) VPN Session Established ─────────┤
```

---

## ⚙ 動作シーケンス

1. **フェーズ 1: トンネルネゴシエーション ＆ SSL/IKEv2 確立**:
   * クライアントがヘッドエンドの IP/FQDN へ接続を試行。
   * SSL モード: TLS ハンドシェイク（サーバー証明書の検証、暗号スイートのネゴシエーション）。
   * IKEv2 モード: IKE_SA_INIT 交換（Diffie-Hellman、暗号化アルゴリズム決定）。
2. **フェーズ 2: ユーザー認証 (Authentication)**:
   * 設定された Auth メソッドに基づき認証を実行。
   * **AAA/RADIUS**: クライアントから送信された ID/Password をヘッドエンドが RADIUS (ISE) へ転送。
   * **Client Certificate**: Client Hello または IKE_AUTH でクライアント証明書を提示。ヘッドエンドが CA トラストポイントで署名検証。
   * **SAML 2.0**: Embedded Browser または External Browser を起動し、IdP ポータルへリダイレクト。ユーザーが IdP 上で認証 ＋ MFA を完了し、IdP 署名済み SAML Assertion をヘッドエンドへポスト。
3. **フェーズ 3: 認可 ＆ ポリシー適用 (Authorization ＆ Policy Assignment)**:
   * ヘッドエンドは接続プロファイル (Tunnel-Group) および Group-Policy をアサイン。
   * RADIUS / ISE から返却された IETF 属性 (Class, Framed-IP-Address) や Cisco VSA (cvpn3000:acl, cvpn3000:av-pair) をアプライ。
   * **DAP (Dynamic Access Policies)**（ASA のみ）: クライアントの Endpoint 属性（OS、MAC、ファイル、HostScan 結果、AAA/SAML 属性）を評価し、最終的な Group-Policy、DACL、WebVPN 制限を動的オーバーライド。
   * **ISE Posture / CoA**（FTD/ASA）: 接続後、ISE Posture Agent が ISE PSN と通信し、コンプライアンス評価。不合格時は ISE が CoA (Change of Authorization) を発行して Restricted DACL / Quarantine VLAN へ変更。
4. **フェーズ 4: アドレス割り当て ＆ XML プロファイル展開**:
   * IP アドレスプール（Local / RADIUS / DHCP）から Virtual NIC アドレスを割り当て。
   * スプリットトンネルルールの適用。
   * XML Client Profile（Always-On, TND, HostScan, NVM, Umbrella Roaming 等）をクライアントへプッシュ更新。
5. **フェーズ 5: データ通信確立**:
   * DTLS (UDP 443) または IPsec ESP SA を確立し、暗号化通信を開始。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **ASA CLI vs FTD FMC GUI vs IOS-XE CLI の表現差異**:
   * **ASA**: `tunnel-group`, `group-policy`, `webvpn`, `dap.xml` を直接編集。
   * **FTD (FMC)**: `Devices > Remote Access` から Wizard または手動で Connection Profile、Group Policy、AnyConnect Package、Address Pool を統合定義。
   * **IOS-XE**: `crypto ikev2 profile`, `crypto ipsec profile`, `interface Virtual-Template`, `webvpn gateway` / `webvpn context` を使用。
2. **SAML 2.0 SSO 設定の落とし穴**:
   * **Single Logout (SLO) ＆ Certificate**: Headend 自体が SAML SP (Service Provider) となるため、SP Signing Certificate を自作またはインポートし、IdP へエクスポートする。
   * **Metadata のインポート/エクスポート**: FMC または ASA CLI で IdP の Entity ID、SSO URL、IdP Certificate を正確に登録する。
   * **AnyConnect Embedded Browser 要件**: 最新の SAML 認証に対応するため、Group Policy 内で `saml-external-browser enable` または AnyConnect Client Profile で `EnableScripting` を許可させる要件が頻出。
3. **証明書 ＋ AAA (Certificate + AAA) 二元認証の構成要件**:
   * `tunnel-group <NAME> webvpn-attributes` 内で `authentication aaa certificate` を設定。
   * 証明書のマッチング要件（`certificate-group-map`）を用いて、特定の Subject CN または SAN (Subject Alternative Name) に基づき自動的に Tunnel-Group を割り当てる設定が問われます。
4. **スプリットトンネリング (Split Tunneling) の型とリスト指定**:
   * **`tunnel-all`**: 全トラフィックを VPN トンネルへ流す。
   * **`sp-tunnel-policy tunnelspecified`**: 指定した Network Object/ACL の宛先のみ VPN トンネルへ流す。
   * **`sp-tunnel-policy excludespecified`**: 指定した 宛先以外 を VPN トンネルへ流す。
5. **DAP (Dynamic Access Policies) の優先度とマッチング原理**:
   * ASA の DAP は複数ルールが同時ヒットした場合、**DACL はアペンド（結合）** され、最も制限の厳しい設定が適用される。
   * `dap.xml` の生成・適用手順および ASDM / CLI での確認方法。

---

## 🛠 設定方法

### 1. ASA CLI: Certificate ＋ RADIUS (ISE) Dual Authentication ＆ Split Tunneling

```bash
! ===== Step 1: AnyConnect パッケージのロード ＆ WebVPN 有効化 =====
webvpn
 enable outside
 anyconnect image disk0:/anyconnect-pkg.pkg 1
 anyconnect enable

! ===== Step 2: IP アドレスプール ＆ Split ACL 定義 =====
ip local pool VPN_POOL 192.168.100.1-192.168.100.254 mask 255.255.255.0

access-list SPLIT_ACL standard permit 10.1.0.0 255.255.0.0
access-list SPLIT_ACL standard permit 10.2.0.0 255.255.0.0

! ===== Step 3: AAA RADIUS サーバーグループ定義 =====
aaa-server ISE_GROUP protocol radius
aaa-server ISE_GROUP (inside) host 10.1.1.50
 key cisco123

! ===== Step 4: Group Policy 定義 =====
group-policy GP_DUAL_AUTH internal
group-policy GP_DUAL_AUTH attributes
 dns-server value 10.1.1.10 10.1.1.11
 default-domain value corp.local
 split-tunnel-policy tunnelspecified
 split-tunnel-network-list value SPLIT_ACL
 vpn-tunnel-protocol ssl-client ikev2

! ===== Step 5: Tunnel Group (Connection Profile) ＆ Dual Auth 設定 =====
tunnel-group TG_DUAL_AUTH type remote-access
tunnel-group TG_DUAL_AUTH general-attributes
 address-pool VPN_POOL
 default-group-policy GP_DUAL_AUTH
 authentication-server-group ISE_GROUP
tunnel-group TG_DUAL_AUTH webvpn-attributes
 authentication certificate aaa                 ! Certificate + AAA 二元認証
 group-alias "Dual_Auth_VPN" enable

! ===== Step 6: Certificate Group Mapping (証明書による TG 自動選択) =====
crypto ca trustpoint CA_ROOT
 enrollment terminal
 crypto ca authenticate CA_ROOT

certificate-map MAP_CORP 10
 subject-name attr cn co corp.local

tunnel-group-map MAP_CORP 10 TG_DUAL_AUTH
```

---

### 2. FMC GUI: FTD AnyConnect SSL VPN ＋ SAML 2.0 構成手順

```text
[ Step 1: SAML Single Sign-On (SSO) オブジェクトの作成 ]
1. FMC UI で Objects > Object Management > AAA Server > Single Sign-On Server に移動。
2. "Add Single Sign-On Server" をクリック:
   - Name: SAML_IdP_Server
   - Request URL: https://saml-idp.example.com/sso
   - Issuer URL (Entity ID): https://vpn.corp.local/saml/sp/metadata
   - Certificate: SAML Token Signing Certificate をインポート。
   - Reload Attribute: Enabled

[ Step 2: Connection Profile (Tunnel Group) の作成 ]
1. Devices > Remote Access に移動し "Add" をクリック。
2. Name: AnyConnect_SAML_RAVPN, Topology: Routed
3. Targeted Devices: 対象 FTD を選択。
4. Connection Profile 設定:
   - Name: SAML_VPN_Profile
   - Authentication Method: SAML
   - Authentication Server: SAML_IdP_Server
   - Client Address Assignment: Local Pool "VPN_POOL_10.200.0.0/24"
   - Group Policy: "GP_SAML_USERS" をアサイン。

[ Step 3: Group Policy ＆ Split Tunneling の構成 ]
1. Group Policy タブ内で "GP_SAML_USERS" を編集:
   - Secure Client (AnyConnect) Settings > Split Tunneling:
     - Policy: Allow Specified
     - Network List: "OBJ_INTERNAL_NETWORKS" (10.0.0.0/8)
   - Protocol Support: SSL Enabled, IPsec IKEv2 Enabled.
   - AnyConnect Client Profile: XML Profile "ANYCONNECT_PROFILE.xml" をバインド。

[ Step 4: AnyConnect Client Images の割り当て ＆ Deploy ]
1. Client Images タブで Windows/macOS 用 AnyConnect パッケージを登録。
2. Save して対象 FTD へ Deploy を実行。
```

---

### 3. Cisco IOS-XE CLI: FlexVPN AnyConnect IKEv2 Remote Access Server

```bash
! ===== Step 1: IKEv2 Proposal ＆ Policy =====
crypto ikev2 proposal IKE2_PROP
 encryption aes-gcm-256
 integrity sha384
 group 19

crypto ikev2 policy IKE2_POL
 proposal IKE2_PROP

! ===== Step 2: Local IP Pool ＆ AAA =====
ip local pool FLEX_POOL 192.168.150.1 192.168.150.254

aaa new-model
aaa authentication login FLEX_AAA_AUTH local
aaa authorization network FLEX_AAA_AUTHOR local
username vpnuser secret Cisco123!

! ===== Step 3: FlexVPN Authorization Policy =====
crypto ikev2 authorization policy FLEX_AUTHOR_POL
 pool FLEX_POOL
 dns 10.1.1.10
 def-domain corp.local
 net-unsplit 10.0.0.0 255.0.0.0                ! Split-Tunnel ACL

! ===== Step 4: IKEv2 Profile =====
crypto ikev2 profile FLEX_IKE2_PROFILE
 match match-any any
 identity local fqdn vpn.corp.local
 authentication local rsa-sig
 authentication remote eap query-identity
 pki trustpoint ROUTER_ROOT_CA
 aaa authentication eap FLEX_AAA_AUTH
 aaa authorization network eap list FLEX_AAA_AUTHOR FLEX_AUTHOR_POL
 virtual-template 1

! ===== Step 5: IPsec Transform Set ＆ Profile =====
crypto ipsec transform-set TS_AES256_SHA256 esp-gcm 256

crypto ipsec profile FLEX_IPSEC_PROFILE
 set transform-set TS_AES256_SHA256
 set ikev2-profile FLEX_IKE2_PROFILE

! ===== Step 6: Virtual-Template Interface =====
interface Virtual-Template1 type tunnel
 ip unnumbered GigabitEthernet1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile FLEX_IPSEC_PROFILE
```

---

## 🔍 検証コマンド

| 目的 | コマンド (ASA / FTD CLI / IOS-XE) | 表示確認ポイント |
| :--- | :--- | :--- |
| **ASA アクティブ AnyConnect セッション確認** | <code>show vpn-sessiondb anyconnect</code> | セッション数、ユーザー名、アサイン IP、プロトコル (SSL/DTLS/IKEv2)、Group Policy |
| **ASA 詳細セッション情報・統計表示** | <code>show vpn-sessiondb detail anyconnect</code> | スプリットトンネル ACL、バイト送信/受信、暗号スイート、DAP マッチルール |
| **FTD (LINA) VPN セッション確認** | <code>show vpn-sessiondb anyconnect</code> (Diagnostic CLI) | FMC からアプライされた AnyConnect 接続のリアルタイム状態 |
| **IOS-XE FlexVPN IKEv2 SA 確認** | <code>show crypto ikev2 sa</code> | IKEv2 ネゴシエーション状態 (`ESTABLISHED`)、Local/Remote IP |
| **IOS-XE FlexVPN IPsec SA ＆ カウンター確認** | <code>show crypto ipsec sa</code> | encaps/decaps パケット数、PKT drop 数、SA 暗号パラメータ |
| **ASA/FTD WebVPN リアルタイムデバッグ** | <code>debug webvpn anyconnect 255</code> | クライアント接続処理、認証やり取り、プロファイル配信ログ |
| **ASA DAP (Dynamic Access Policy) デバッグ** | <code>debug dap trace</code> / <code>debug dap status</code> | DAP ルールの評価プロセス、属性評価結果、DACL アプライ情報 |
| **SAML 認証リアルタイムデバッグ** | <code>debug webvpn saml 255</code> | SAML Request 生成、IdP レスポンス、Assertion 署名検証結果 |
| **IOS-XE FlexVPN IKEv2 デバッグ** | <code>debug crypto ikev2 internal</code> | IKEv2 EAP 認証、AAA Authorization ポリシー割り当てのデバッグ |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **AnyConnect 接続後、Web トラフィックは通るが UDP / 通信（音声・データ）が極端に遅延・ドロップする** | DTLS (UDP 443) トンネルの確立に失敗し、TCP/TLS 上へフォールバックしている | <code>show vpn-sessiondb detail anyconnect</code> | 外部ファイアウォールで **UDP 443** が許可されているか確認。`anyconnect-enable` および `webvpn` 内で `dtls enable` を有効化する。 |
| **SAML 認証時、AnyConnect クライアント上に「Page cannot be displayed」または JavaScript エラーが表示される** | IdP が要求するモダン認証 (HTML5/JS) に Embedded Browser が対応できていない | AnyConnect Client Profile / <code>debug webvpn saml 255</code> | Group Policy で **`saml-external-browser enable`** を設定し、OS の標準デフォルトブラウザ (Chrome/Edge) で SAML 認証を行わせる。 |
| **証明書 ＋ AAA 二元認証で、証明書はパスするが Tunnel-Group にアサインされない** | `certificate-group-map` または Subject CN の文字列マッチング条件が不一致 | <code>show running-config certificate-map</code><br><code>debug webvpn anyconnect 255</code> | 証明書の Subject / SAN 属性を確認し、`certificate-map` の正規表現および `tunnel-group-map` を修正する。 |
| **AnyConnect 接続後、社内サブネット宛ての通信が VPN トンネルへ入らずローカルへ抜けてしまう** | Group Policy の Split-Tunnel ACL のネットワークマスク指定ミス、または `tunnelspecified` が未設定 | <code>show vpn-sessiondb detail anyconnect</code> | Group Policy の `split-tunnel-policy tunnelspecified` および Standard/Extended ACL の宛先定義を見直す。 |
| **IOS-XE FlexVPN で AnyConnect IKEv2 接続時、`AUTHENTICATION-FAILED` で拒否される** | IKEv2 Profile の EAP 認証タイプ (EAP-MSCHAPv2 / EAP-GTC) が AAA サーバー側の許可設定と不一致 | <code>debug crypto ikev2 internal</code> | RADIUS サーバー (ISE) の Authentication Policy で EAP-MSCHAPv2 を許可し、`aaa authentication eap` を調整する。 |

---

## ⚠ 制限事項

* **FTD における AnyConnect 機能制限**:
  * ASA CLI で利用可能な高度な Dynamic Access Policies (DAP) の一部複雑な条件評価や、Lua スクリプトによる Custom Attributes 設定は、FMC GUI ではネイティブ直接サポートされません（ISE Dynamic Authorization / CoA や SGT で代用）。
* **SAML 認証時の Session Timeout ＆ Re-Authentication**:
  * SAML 認証では、Rekey 時や瞬断再接続時に IdP の Cookie 有効期限が切れていると、再度ブラウザポップアップによる再認証が必要となる場合があります。
* **IOS-XE FlexVPN での SSL WebVPN 制限**:
  * Cisco IOS-XE ルーター上の AnyConnect サポートは **IKEv2/IPsec プロトコルがメイン** であり、ASA/FTD のような豊富な WebVPN ポータル機能や HostScan 動的モジュール配布には制限があります。

---

## 🔄 他技術との関連

* **Cisco ISE (Identity Services Engine)**: RADIUS / TACACS+ / SAML IdP 連携、DACL (Downloadable ACL)、VLAN/SGT 動的アサイン、および **ISE Posture Agent** 経由での Compliance Check ＆ CoA 発行。
* **PKI / Certificate Authority (CA)**: SSL/TLS 証明書、Client Certificate 認証、SAML Assertion 署名検証用 Trustpoint。
* **Cisco Umbrella Roaming Module**: AnyConnect クライアントプラグインとして統合され、VPN トンネル外の直接インターネット接続 (Local Breakout) 時にも DNS レイヤーセキュリティを強制。

---

## 🧩 比較表

### 1. SSL/TLS (HTTPS/DTLS) AnyConnect vs IKEv2/IPsec AnyConnect

| 比較項目 | SSL/TLS (HTTPS / DTLS) AnyConnect | IKEv2 / IPsec AnyConnect |
| :--- | :--- | :--- |
| **使用ポート** | TCP 443 (TLS) ＋ UDP 443 (DTLS) | UDP 500 (IKE) ＋ UDP 4500 (IPsec NAT-T) |
| **ファイアウォール透過性** | **極めて高い** (通常の HTTPS トラフィックに擬態) | 高い (ただし UDP 500/4500 の通過許可が必要) |
| **認証方式** | AAA, Client Cert, **SAML 2.0 SSO**, Dual Auth | AAA (EAP), Client Cert, IKEv2 Auth |
| **モバイル切断耐性** | 中程度 (DTLS 復帰処理) | **極めて高い** (IKEv2 Mobility and Multihoming: MOBIKE) |
| **暗号強度** | TLS 1.2/1.3, AES-GCM | IKEv2 / IPsec Suite B, AES-GCM-256 |

### 2. ASA vs FTD (FMC) vs IOS-XE FlexVPN (AnyConnect Headend)

| 比較項目 | ASA (Adaptive Security Appliance) | FTD (Firepower Threat Defense) | Cisco IOS-XE FlexVPN Router |
| :--- | :--- | :--- | :--- |
| **プライマリプロトコル** | SSL ＆ IKEv2 | SSL ＆ IKEv2 | **IKEv2 / IPsec** |
| **高度アクセス制御** | **DAP (Dynamic Access Policies)** / HostScan | **ISE Posture (CoA)** / DACL / SGT | RADIUS VSA / FlexVPN Authorization |
| **SAML 2.0 サポート** | 完全対応 (ASA CLI / ASDM) | 完全対応 (FMC 6.7+ GUI) | 制限あり |
| **統合 NGFW 機能** | ASA Firepower Services モジュール | **Snort 3 IPS / AMP / URL 一体化** | Zone-Based Firewall (ZBFW) 併用 |

---

## 💡 ベストプラクティス

1. **DTLS (Datagram TLS) の常時有効化**:
   * SSL AnyConnect 構成時は、必ず UDP 443 の DTLS トンネルを有効化します。音声（VoIP）やビデオ会議、TCP In-TCP カプセル化による崩壊（TCP Collapse）を防止し、パフォーマンスを最大化します。
2. **SAML 外部ブラウザ (`saml-external-browser enable`) の活用**:
   * クラウド IdP と統合する際は、OS のデフォルトブラウザを利用させることで、FIDO2 WebAuthn パスキー、Windows Hello、ハードウェアセキュリティキーなどの最新 MFA 認証を完全に透過利用させることができます。
3. **DTLS / IPsec 用の Maximum Transmission Unit (MTU) 調整**:
   * トンネルオーバーヘッドによるパケット断片化を防ぐため、Group Policy 内で `anyconnect mtu 1400` を明示設定し、TCP MSS Clamping (例: 1360 バイト) を併用します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA: SSL AnyConnect ＋ 証明書 ＆ RADIUS 二元認証 (Dual Auth)

#### 問題
ASA1 において、AnyConnect SSL VPN を構成せよ。クライアントは有効な社内 CA 発行の証明書を提示し、かつ RADIUS サーバー (`10.1.1.50`) でのユーザー名/パスワード認証の両方に成功しなければ接続を許可しないこと。

#### 要件
* Connection Profile: `TG_DUAL`
* Group Policy: `GP_DUAL`
* Split Tunnel: 内部ネットワーク `10.50.0.0/16` のみトンネル通過させること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
ip local pool POOL_DUAL 192.168.201.1-192.168.201.254 mask 255.255.255.0

access-list ACL_SPLIT_DUAL standard permit 10.50.0.0 255.255.0.0

aaa-server ISE_RADIUS protocol radius
aaa-server ISE_RADIUS (inside) host 10.1.1.50
 key cisco123

group-policy GP_DUAL internal
group-policy GP_DUAL attributes
 dns-server value 10.1.1.10
 split-tunnel-policy tunnelspecified
 split-tunnel-network-list value ACL_SPLIT_DUAL
 vpn-tunnel-protocol ssl-client

tunnel-group TG_DUAL type remote-access
tunnel-group TG_DUAL general-attributes
 address-pool POOL_DUAL
 default-group-policy GP_DUAL
 authentication-server-group ISE_RADIUS
tunnel-group TG_DUAL webvpn-attributes
 authentication certificate aaa
 group-alias "Dual_Auth" enable

webvpn
 enable outside
 anyconnect image disk0:/anyconnect-pkg.pkg 1
 anyconnect enable
```

#### 検証方法
* `show vpn-sessiondb detail anyconnect` を実行し、`Authentication: Certificate + AAA` およびアサインされた IP アドレスを確認。

---

### 例2: ASA: AnyConnect IKEv2 ＋ Dynamic Access Policies (DAP)

#### 問題
ASA1 において AnyConnect IKEv2 VPN を構成せよ。さらに DAP を作成し、端末の OS が Windows かつ特定アンチウイルスソフトが検知された場合のみ特定 DACL をアサインせよ。

#### 要件
* IKEv2 有効化、DAP ルール `DAP_SECURE_WIN` を適用すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
crypto ikev2 policy 10
 encryption aes-256
 integrity sha256
 group 19

crypto ikev2 enable outside client-services

group-policy GP_IKE2 internal
group-policy GP_IKE2 attributes
 vpn-tunnel-protocol ikev2

tunnel-group TG_IKE2 type remote-access
tunnel-group TG_IKE2 general-attributes
 address-pool POOL_DUAL
 default-group-policy GP_IKE2
 authentication-server-group ISE_RADIUS
tunnel-group TG_IKE2 webvpn-attributes
 group-alias "IKEv2_VPN" enable

! ===== Dynamic Access Policy (dap.xml 設定イメージ) =====
! match: target.os.which == "Windows" AND aaa.cisco.grouppolicy == "GP_IKE2"
! action: dacl = "permit ip any 10.0.0.0 255.0.0.0"
```

#### 検証方法
* `show vpn-sessiondb detail anyconnect` を実行し、`Session Type: IKEv2` および `DAP Record Applied: DAP_SECURE_WIN` を確認。

---

### 例3: FTD (FMC): AnyConnect SSL VPN ＋ SAML 2.0 (External Browser)

#### 問題
FMC を使用して FTD 上に AnyConnect SSL VPN を構築せよ。認証には SAML 2.0 を使用し、認証画面は OS 標準の外部ブラウザで起動させよ。

#### 要件
* Group Policy で External Browser を有効化すること。

#### 設定例

```text
[ FMC GUI Configuration Steps ]
1. Objects > Single Sign-On Server ➔ SAML_IdP 追加 (Request URL, Entity ID, Certificate 登録)。
2. Devices > Remote Access ➔ "Add RAVPN":
   - Name: RAVPN_SAML
   - Connection Profile: Profile_SAML (Auth: SAML ➔ SAML_IdP)
   - Group Policy: GP_SAML
3. GP_SAML 編集 ➔ Advanced ➔ SAML Settings:
   - [✓] Enable External Browser for SAML Authentication
4. Client Images で AnyConnect Package 指定 ➔ Deploy。
```

#### 検証方法
* クライアントから接続試行時、既定の Web ブラウザが起動して SAML ログインプロンプトが表示されることを確認。

---

### 例4: FTD (FMC): AnyConnect IKEv2 Remote Access ＋ Local Pool

#### 問題
FMC 上で FTD を AnyConnect IKEv2 専用の Remote Access VPN ヘッドエンドとして構築せよ。

#### 要件
* Protocol: IKEv2 Only
* Local Address Pool: `192.168.210.0/24`

#### 設定例

```text
[ FMC GUI Configuration Steps ]
1. Objects > Address Pools ➔ "POOL_IKE2" (192.168.210.1 - 192.168.210.254) 作成。
2. Devices > Remote Access ➔ RAVPN 編集:
   - Connection Profile ➔ General Attributes ➔ Address Pool: POOL_IKE2
   - Group Policy ➔ General ➔ Protocols: [ ] SSL, [✓] IPSec (IKEv2)
3. IPsec ➔ Native IPsec (IKEv2) Proposal ＆ Transform Set を指定 ➔ Deploy。
```

#### 検証方法
* クライアント AnyConnect から IKEv2 モードで接続し、`show vpn-sessiondb anyconnect` で IPsec IKEv2 トンネルが確立されていることを確認。

---

### 例5: Cisco IOS-XE: FlexVPN AnyConnect Server ＋ EAP-MSCHAPv2

#### 問題
Cisco IOS-XE ルーター R1 上で、FlexVPN フレームワークを用いて AnyConnect IKEv2 リモートアクセスサーバーを構築せよ。

#### 要件
* ユーザー認証に EAP-MSCHAPv2 を使用すること。

#### 設定例

```bash
! ===== R1 Configuration =====
ip local pool FLEX_POOL 192.168.220.1 192.168.220.254

aaa new-model
aaa authentication login RADIUS_AUTH group radius
aaa authorization network RADIUS_AUTHOR group radius

crypto ikev2 authorization policy AUTHOR_POL
 pool FLEX_POOL
 dns 10.1.1.10

crypto ikev2 profile IKE2_ANYCONNECT_PROFILE
 match match-any any
 identity local fqdn vpn.example.com
 authentication local rsa-sig
 authentication remote eap query-identity
 pki trustpoint ROUTER_TP
 aaa authentication eap RADIUS_AUTH
 aaa authorization network eap list RADIUS_AUTHOR AUTHOR_POL
 virtual-template 1

crypto ipsec profile IPSEC_ANYCONNECT_PROFILE
 set ikev2-profile IKE2_ANYCONNECT_PROFILE

interface Virtual-Template1 type tunnel
 ip unnumbered GigabitEthernet1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC_ANYCONNECT_PROFILE
```

#### 検証方法
* `show crypto ikev2 sa` および `show crypto ipsec sa` で `ESTABLISHED` 状態と EAP 認証成功を確認。

---

### 例6: ASA: AnyConnect HostScan ＋ DAP ポスチャ評価

#### 問題
ASA1 において HostScan モジュールを有効化し、AnyConnect 接続時に端末のウイルス対策ソフト稼働状態を収集して DAP でアクセス制限せよ。

#### 要件
* `hostscan image` コマンドを使用すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
webvpn
 hostscan image disk0:/hostscan-pkg.pkg
 hostscan enable
 anyconnect enable
```

#### 検証方法
* クライアント接続時に HostScan 走査が実行され、ASA CLI の `debug dap trace` で収集された AV 属性が出力されることを確認。

---

### 例7: FTD (FMC): AnyConnect ＋ ISE Posture Agent 統合 ＆ CoA (Change of Authorization)

#### 問題
FTD において、AnyConnect 接続クライアントに対し Cisco ISE Posture Agent を介したポスチャチェックを構成せよ。

#### 要件
* RADIUS CoA (Port 1700) を許可すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Objects > AAA Server ➔ ISE RADIUS Group 定義 (Enable CoA: Enabled, Port 1700)。
2. RAVPN Connection Profile ➔ AAA 認証・認可サーバーとして ISE RADIUS Group を指定。
3. Access Control Policy ➔ ISE Posture 未完了時は Redirect ACL をアサインするアイデンティティルールを配置。
```

#### 検証方法
* クライアント接続後、ISE パネルで `Posture Status: Compliant` に変化し、ISE から CoA が発行されて完全アクセス ACL に切り替わることを確認。

---

### 例8: ASA: AnyConnect Web-Launch ＆ XML Profile (Always-On / TND) 配信

#### 問題
ASA1 において、AnyConnect クライアントプロファイル `Profile.xml` を配信し、社内 LAN 接続時は自動的に VPN を無効化する TND (Trusted Network Detection) を構成せよ。

#### 要件
* `webvpn` セクションで Client Profile をアサインすること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
webvpn
 anyconnect profiles PROFILE_TND disk0:/Profile.xml

group-policy GP_TND internal
group-policy GP_TND attributes
 webvpn
  anyconnect profiles value PROFILE_TND type user
```

#### 検証方法
* AnyConnect 接続完了後、クライアント PC の Profile ディレクトリに `Profile.xml` がダウンロードされることを確認。

---

### 例9: Cisco IOS-XE: FlexVPN AnyConnect ＋ RADIUS DACL 動的適用

#### 問題
IOS-XE ルーター R1 において、AnyConnect 接続成功時に RADIUS (ISE) から返却される Downloadable ACL (DACL) を適用せよ。

#### 要件
* AAA Authorization ポリシーで RADIUS 属性を許可すること。

#### 設定例

```bash
! ===== R1 Configuration =====
aaa authorization network FLEX_AUTHOR_AAA group radius

crypto ikev2 authorization policy AUTHOR_POL
 aaa-attribute type author-acl
```

#### 検証方法
* `show ip access-lists` を実行し、動的に生成された `per-user` DACL 項目を確認。

---

### 例10: ASA/FTD: AnyConnect Split-Tunneling (Exclude-Specified モード)

#### 問題
AnyConnect VPN 接続時、デフォルトで全トラフィックを VPN トンネルへ送るが、特定宛て（`192.0.2.0/24`）のみ VPN トンネルから除外 (Local Breakout) せよ。

#### 要件
* `excludespecified` スプリットトンネルポリシーを使用すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
access-list ACL_EXCLUDE standard permit 192.0.2.0 255.255.255.0

group-policy GP_EXCLUDE internal
group-policy GP_EXCLUDE attributes
 split-tunnel-policy excludespecified
 split-tunnel-network-list value ACL_EXCLUDE
```

#### 検証方法
* クライアントの AnyConnect Route Details 画面で、`192.0.2.0/24` が Non-Secured Routes (Excluded) に登録されていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ASA1 上で以下のコンフィグが投入されている。

```bash
tunnel-group TG_CORP type remote-access
tunnel-group TG_CORP webvpn-attributes
 authentication certificate aaa
```

クライアントが AnyConnect 接続を試みた際の認証動作として正しいものはどれか。

* A) 証明書認証に成功すれば、AAA 認証はスキップされて即座に接続が確立される。
* B) クライアントは有効なクライアント証明書を提示し、かつ提示されたユーザー名/パスワードによる AAA 認証の両方に成功しなければ接続が許可されない。
* C) AAA 認証が失敗した場合でも、証明書が有効であればバックアップとして接続が許可される。
* D) SAML 2.0 認証へ自動リダイレクトされる。

**正解**: **B**  
**解説**: `authentication certificate aaa` は **Certificate ＋ AAA 二元認証 (Dual Authentication)** を意味します。クライアント証明書の検証と AAA (RADIUS/Local) 認証の両方が成功しなければ接続は許可されません。

---

### 2. トラブルシュート問題

**問題**: FTD 上で AnyConnect SSL VPN ＋ SAML 2.0 を構築した。クライアントが AnyConnect から接続を開始したところ、SAML ログイン画面が表示されず「SAML 認証のハンドシェイクに失敗しました」というエラーが発生した。FTD Diagnostic CLI で `debug webvpn saml 255` を実行したところ、以下のログが出力されていた。

`SAML assertion validation failed: Certificate signature mismatch`

原因および修正方法として適切なものはどれか。

* A) FTD の DTLS ポートがブロックされている。
* B) FMC に登録された **IdP Token Signing Certificate（IdP 証明書）が不一致または期限切れ** である。最新の証明書を再インポートする。
* C) AnyConnect クライアントのバージョンが古い。
* D) IP アドレスプールが枯渇している。

**正解**: **B**  
**解説**: SAML Response に付与された IdP 署名と、ヘッドエンド (FTD/FMC) に登録された IdP 証明書のパブリックキーが一致しない場合、`Certificate signature mismatch` となり SAML 検証が拒否されます。

---

### 3. Design 問題

**問題**: 大規模企業の AnyConnect Remote Access VPN 設計において、VPN トンネル内を通過する音声・ビデオ通信の低遅延化を図ると同時に、トンネル通過不能な厳しい環境下（一部公衆 Wi-Fi 等）でも通信を切断させず自動フォールバックさせたい。ヘッドエンドでの最適な設定はどれか。

* A) SSL/TLS (TCP 443) のみを有効化し、DTLS を明示的に無効化する。
* B) **SSL (TCP 443) と DTLS (UDP 443) の両方を有効化** する。通常は低遅延な DTLS を優先使用させ、UDP がブロックされた場合は自動的に SSL/TLS へフォールバックさせる。
* C) IKEv1 プレシェアードキー認証のみを構成する。
* D) スプリットトンネリングを無効化する。

**正解**: **B**  
**解説**: **DTLS (Datagram TLS)** を有効化することで、パケット再送遅延を防ぎリアルタイム通信の品質を向上させます。UDP 443 が通過できない環境では自動的に TLS (TCP 443) へフォールバックされるため可用性も維持されます。

---

### 4. 実装問題

**問題**: ASA1 において、AnyConnect 接続クライアントの SAML 2.0 認証時に、OS 標準の既定ブラウザを自動起動させて MFA パスキル認証を行わせたい。Group Policy で設定すべきコマンドはどれか。

* A) `saml-external-browser enable`
* B) `webvpn-sso force`
* C) `anyconnect script-enable`
* D) `browser-mode internal`

**正解**: **A**  
**解説**: **`saml-external-browser enable`** を設定することで、AnyConnect 内蔵の Embedded Browser ではなく、OS 標準の外部ブラウザが自動起動し、最新の MFA 認証が利用可能になります。

---

### 5. コンフィグ読解問題

**問題**: Cisco IOS-XE ルーター上で FlexVPN AnyConnect サーバーを構築している。クライアント側からの接続要求に対し、IKEv2 認証として EAP 認証を要求し、RADIUS サーバーへ問い合わせを行うための IKEv2 Profile 設定として正しいものはどれか。

* A) `authentication remote pre-share`
* B) `authentication remote eap query-identity`
* C) `authentication remote rsa-sig`
* D) `crypto map set transform-set`

**正解**: **B**  
**解説**: FlexVPN で AnyConnect クライアントからのユーザー名/パスワード (EAP) 認証をネゴシエートするには、IKEv2 Profile 内で **`authentication remote eap query-identity`** を設定します。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center RAVPN Configuration Guide](https://www.cisco.com/)
* [Cisco IOS-XE FlexVPN Remote Access AnyConnect Configuration Guide](https://www.cisco.com/)
* [Cisco Technical Notes: Configure SAML 2.0 SSO for AnyConnect on ASA and FTD](https://www.cisco.com/)
* [Cisco Live: Deep Dive into AnyConnect Architecture and Troubleshooting (BRKSEC-3011)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で AnyConnect RA-VPN が出題された場合の構成選択フローチャート：
    * **「Configure AnyConnect SSL with Certificate + Username/Password」** ➔ ASA/FTD Connection Profile ➔ `authentication certificate aaa`。
    * **「Configure SAML 2.0 SSO with External Browser」** ➔ FMC/ASA ➔ SSO Server オブジェクト作成 ➔ Group Policy: `saml-external-browser enable`。
    * **「Configure AnyConnect IKEv2 on IOS-XE Router」** ➔ FlexVPN ➔ `crypto ikev2 profile` ➔ `authentication remote eap query-identity` ➔ `interface Virtual-Template`。
    * **「Apply Dynamic Access Policies (DAP)」** ➔ ASA ➔ HostScan 有効化 ➔ `dap.xml` ルールアサイン。
* **図解（AnyConnect 認証 ＆ トンネル昇格 パイプライン）**:
  ```text
  [ Client Connect ] ──► [ TLS Handshake (TCP 443) ]
                              │
                              ├──► SAML / Cert / AAA Auth
                              │
                              ├──► DAP / Group Policy / XML Profile Push
                              │
                              ├──► DTLS (UDP 443) Negotiation
                              │        ├── Success? ──► [ DTLS Tunnel (Data) ]
                              │        └── Failed?  ──► [ TLS Tunnel (Data Fallback) ]
                              │
                              └──► Assign IP Address & Split-Tunnel Rules
  ```
* **注意点**:
  * ラボ試験で AnyConnect を設定した後は、実際にクライアント接続を試行するか、`show vpn-sessiondb anyconnect` / `show crypto ikev2 sa` を実行し、セッションが `Active` でアサインされた IP アドレスや Split-Tunnel 範囲が正しいことを必ず確認してください。
