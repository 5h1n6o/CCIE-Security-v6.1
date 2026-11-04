---
layout: default
title: 3.5.a-WPA
nav_order: 1
parent: 3.5-Wireless-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.5.a WPA

## 📘 概要

* **機能概要**: Wi-Fi Protected Access (WPA) シリーズ（WPA, WPA2, WPA3）は、IEEE 802.11 無線 LAN 規格においてデータフレームの暗号化、送信元認証、およびメッセージ改ざん防止（完全性保護）を提供するセキュリティフレームワーク群である。WPA（IEEE 802.11i 下書き準拠）は TKIP（Temporal Key Integrity Protocol）を採用し、初期の WEP の脆弱性を迅速に解消するために制定された。WPA2（IEEE 802.11i 正式規格）は AES-CCMP（Counter Mode with Cipher Block Chaining Message Authentication Code Protocol）を採用し、強固な機密性と完全性を提供する。WPA3 は、SAE（Simultaneous Authentication of Equals: Dragonfly ハンドシェイク）による辞書攻撃耐性の向上、Protected Management Frames (PMF/802.11w) の必須化、192 ビット暗号化スイート（Enterprise 192-bit / CNSA 互換モード）の導入により、現代の無線の高度な脅威に対応する。
* **利用目的**: 物理的に電波が拡散する無線 LAN 空間において、未認証ユーザーの不正接続（Unauthorized Access）を防ぎ、盗聴（Eavesdropping）、改ざん（Tampering）、中間者攻撃（MitM）、Man-in-the-Middle 辞書攻撃、および管理フレーム偽装（Deauthentication Flood / Disassociation Attack）から通信を保護する。
* **どのような場面で利用するか**:
  * **WPA2/WPA3 Enterprise (802.1X/EAP)**: 企業内ネットワークにおいて Cisco ISE（Identity Services Engine）などの RADIUS サーバーと連携し、ユーザー/端末固有の証明書（EAP-TLS）や Active Directory 認証（PEAP-MSCHAPv2）を用いてアクセス制御を行う。
  * **WPA3 Personal (SAE)**: 共有鍵（PSK）運用が求められる拠点で、事前共有鍵が漏洩した場合でも過去の通信トラフィックが解読されない「前方秘匿性（Forward Secrecy）」を確保し、ブルートフォース/オフライン辞書攻撃を防ぐ。
  * **WPA3 Enterprise 192-bit Mode**: 政府機関、金融機関、医療機関など超高セキュリティ要件（Suite B / CNSA 適合）が求められる環境で、GCMP-256 暗号化と SHA-384 署名、ECDHE-P-384 鍵交換を強制する。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **特徴** | WPA（TKIP/RC4）、WPA2（AES-CCMP-128）、WPA3（SAE/AES-CCMP-128/GCMP-256）。Enterprise モードでは 802.1X/EAP による動的鍵生成（PMK）を行い、Personal モードでは Pre-Shared Key (PSK) または SAE から PMK を生成する。WPA3 では PMF (802.11w) が完全必須化。 |
| **用途** | 企業内セキュア無線アクセス（Enterprise 802.1X）、ゲスト/拠点アクセス（Personal / OWE）、高度セキュリティ要件拠点（Enterprise 192-bit）。 |
| **メリット** | WPA2/WPA3 により強固な暗号化（AES/GCMP）と認証を提供。WPA3 SAE により PSK の弱点（オフライン辞書攻撃・前方秘匿性の欠如）を克服。802.11w PMF により DoS 攻撃（Deauth 攻撃）を防御。 |
| **デメリット** | WPA3 / 802.11w 未対応のレガシー端末（IoT 機器や古くからのバーコードスキャナ等）が接続不可となるリスク。WPA3 Transition Mode の導入により相互運用性問題が発生する可能性。 |
| **対応機種** | Catalyst 9800 シリーズ WLC（IOS-XE 16.12/17.x 以降）、AireOS WLC（8.10 以降で WPA3 サポート）、Cisco Embedded Wireless Controller (EWC)、Cisco Catalyst / Aironet AP シリーズ。 |
| **制限事項** | WPA3 Enterprise 192-bit モードでは、GCMP-256 および SHA-384/EAP-TLS（ECDHE/RSA-384）非対応の端末は接続不能。Fast Transition (802.11r) や PMK Caching との併用時にクライアント実装による互換性問題が発生しやすい。 |
| **設計上の注意点** | WPA2 から WPA3 への移行時は Transition Mode (WPA2-PSK + WPA3-SAE または WPA2-Enterprise + WPA3-Enterprise) を一時的に利用し、PMF を `Optional` または `Required` に適切に設定する。Cisco 9800 WLC では WLAN Profile と Policy Profile の分離設計に注意する。 |

---

## 🏗 動作原理

WPA/WPA2/WPA3 のセキュリティは、**マスターキー（PMK: Pairwise Master Key）の生成・導出**と、それに基づく**4-Way Handshake（PTK / GTK / IGTK の生成・配布）**によって構成される。

### 1. 鍵階層（Key Hierarchy）アーキテクチャ

無線セッションの暗号化鍵は、以下の階層構造に従って動的に生成される。

```
                    [ 認証フェーズ ]
     +--------------------------------------------+
     | Enterprise: 802.1X/EAP (RADIUS/ISE)       |
     | Personal  : PSK (WPA2) または SAE (WPA3)   |
     +--------------------------------------------+
                           |
                           v
              +--------------------------+
              | PMK (Pairwise Master Key)|
              +--------------------------+
                           |
            +--------------+--------------+
            | 4-Way Handshake (EAPOL-Key) |
            +--------------+--------------+
                           |
       +-------------------+-------------------+
       |                                       |
       v                                       v
+-------------------------------+   +-------------------------------+
| PTK (Pairwise Transit Key)    |   | GTK (Group Temporal Key)      |
| (ユニキャスト通信の暗号化/認証)  |   | (マルチキャスト/ブロードキャスト) |
+-------------------------------+   +-------------------------------+
       |                                       |
       +--> KCK (EAPOL Key Confirmation Key)   +--> IGTK (Integrity GTK)
       +--> KEK (EAPOL Key Encryption Key)          (WPA3/802.11w Management
       +--> TK (Temporal Encryption Key)             Frame Integrity)
```

* **PMK (Pairwise Master Key)**:
  * Enterprise: 802.1X/EAP-TLS 等の成功後、MSK (Master Session Key) から導出され、RADIUS サーバーから Access-Accept 経由で WLC/AP に送信される。
  * Personal (WPA2): `PBKDF2(SSID, Passphrase, 4096, 256)` により生成。
  * Personal (WPA3): SAE (Dragonfly 楕円曲線ディフィー・ヘルマン鍵交換) により、セッションごとに固有の要素（Scalar/Element）を交わして動的に導出（前方秘匿性確保）。
* **PTK (Pairwise Transit Key)**:
  * `PTK = PRF-384/512 (PMK, "Pairwise key expansion", Min/Max(SNonce, ANonce) || Min/Max(SPA, AA))`
  * PTK は以下のサブキーで構成される:
    1. **KCK (Key Confirmation Key)**: EAPOL-Key フレームの MIC（メッセージ整合性コード）計算・検証用。
    2. **KEK (Key Encryption Key)**: 4-Way Handshake 中に GTK や IGTK を暗号化してクライアントへ安全に送信するための鍵。
    3. **TK (Temporal Key)**: 実際にクライアントと AP 間で送信されるデータフレームの暗号化（AES-CCMP / GCMP）に使用される鍵。
* **GTK (Group Temporal Key)**: AP が生成し、接続中の全クライアントに安全に配布されるマルチキャスト/ブロードキャスト暗号化鍵。
* **IGTK (Integrity Group Temporal Key)**: 802.11w PMF で使用され、Deauthentication / Disassociation などの管理フレームの改ざん防止・送信元検証に使用される鍵。

---

## ⚙ 動作シーケンス

WPA/WPA2/WPA3 Enterprise (802.1X) および Personal (WPA3 SAE) におけるパケット処理・認証・鍵交換のシーケンスを以下に示す。

### WPA2/WPA3 Enterprise (802.1X/EAP-TLS) 認証 & 鍵交換シーケンス

```
 Client (STA)             Access Point / WLC               Cisco ISE (RADIUS)
      |                           |                                |
      |--- 802.11 Auth Req ------>|                                |
      |<-- 802.11 Auth Resp ------|                                |
      |                           |                                |
      |--- 802.11 Assoc Req ----->| (RSN IE: WPA2/WPA3 Capable)    |
      |<-- 802.11 Assoc Resp -----|                                |
      |                           |                                |
      |=================== 802.1X / EAP Phase ====================|
      |<-- EAP-Request/Identity --|                                |
      |--- EAP-Response/Identity->|--- RADIUS Access-Request ----->|
      |                           |<-- RADIUS Access-Challenge ----|
      |<-- EAP-TLS Start ---------|                                |
      |--- EAP-TLS Client Hello ->|--- RADIUS Access-Request ----->|
      |                           |    (TLS Handshake Exchange)    |
      |<-- EAP-Success -----------|<-- RADIUS Access-Accept -------|
      |                           |    (Contains MSK / PMK)        |
      |                           |                                |
      |=================== 4-Way Handshake Phase ==================|
      |<-- EAPOL-Key (Msg 1: ANonce) ------------------------------|
      |    (STA derives PTK)      |                                |
      |--- EAPOL-Key (Msg 2: SNonce + MIC) ----------------------->|
      |                           | (AP verifies MIC & derives PTK)|
      |<-- EAPOL-Key (Msg 3: Install + Encrypted GTK + MIC) --------|
      |    (STA installs PTK/GTK) |                                |
      |--- EAPOL-Key (Msg 4: MIC Ack) ---------------------------->|
      |                           | (AP installs PTK)              |
      |                           |                                |
      |================ Secure Data Encrypted Transmission ========|
```

### WPA3 Personal (SAE: Simultaneous Authentication of Equals) 動作シーケンス

WPA3 Personal では、従来の PSK 認証で用いられたプレーンな 4-Way Handshake 前処理に代わり、Open Authentication ステージ中に Dragonfly ハンドシェイク（Commit / Confirm）を実行する。

```
 Client (STA)                     Access Point / WLC
      |                                  |
      |====== 802.11 Open Auth / SAE Dragonfly Handshake ======|
      |--- 802.11 Auth (SAE Commit) ---->| (Exchanges Scalar & Element)
      |<-- 802.11 Auth (SAE Commit) -----| 
      |--- 802.11 Auth (SAE Confirm) --->| (Exchanges Field / Verification Hash)
      |<-- 802.11 Auth (SAE Confirm) ----| 
      |   [Both STA and AP derive identical PMK with Forward Secrecy]
      |                                  |
      |====== 802.11 Association Phase ========================|
      |--- 802.11 Association Request -->| (RSN IE: WPA3 SAE, PMF Required)
      |<-- 802.11 Association Response --|
      |                                  |
      |====== 4-Way Handshake Phase ============================|
      |<-- EAPOL-Key (Msg 1: ANonce) ----|
      |--- EAPOL-Key (Msg 2: SNonce) --->|
      |<-- EAPOL-Key (Msg 3: GTK/IGTK) --|
      |--- EAPOL-Key (Msg 4: Ack) ------->|
      |                                  |
```


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、無線セキュリティ（WPA2/WPA3）は Cisco Catalyst 9800 WLC (IOS-XE)、Cisco ISE、および 802.1X/EAP 認証基盤の統合問題として頻出する。単に WPA2/WPA3 を有効化するだけでなく、試験要件に提示される厳格な暗号指定や相互運用性制約を正確にコンフィグへ落とし込む必要がある。

### 1. Blueprintで重要なポイント

1. **WPA3 SAE (Personal) と Transition Mode の制御**:
   * SAE（Simultaneous Authentication of Equals）のみを強制する場合は、WPA2-PSK を無効化し、AKM（Authentication Key Management）に `sae` のみを指定する。
   * **Transition Mode** 要件の場合は、AKM に `psk` と `sae` の両方を設定し、PMF（Protected Management Frames）を `Optional`（IOS-XE では `pmf-optional`）に設定する。WPA3 専用（SAE Only）の場合は PMF を `Required`（`pmf-mandatory`）にする必要がある。
2. **WPA3 Enterprise 192-bit Mode (CNSA / Suite B 適合)**:
   * 暗号化方式に **GCMP-256**（`gcmp256`）を指定し、AKM に `suiteb-192` を設定する。
   * PMF に **BIP-GMAC-256**（`pmf-mandatory` + `gmac-256`）を指定する必要がある。
   * 802.1X 認証サーバー（Cisco ISE）側でも TLS 1.2/1.3 上で ECDHE-ECDSA-AES256-GCM-SHA384 等の 192 ビットスイートが選択されるよう EAP-TLS ポリシーを設定する。
3. **802.11w Protected Management Frames (PMF)**:
   * WPA3 では PMF が標準規格上必須である。WLC 上での PMF 設定誤り（例: WPA3 SAE なのに PMF を `Disabled` にする）は、WLAN の起動エラー（`Disabled` 状態のまま変化しない）やクライアント接続不可の原因となる。
4. **802.11r Fast Transition (FT) と AKM の組み合わせ**:
   * FT-PSK (`ft-psk`)、FT-SAE (`ft-sae`)、FT-802.1X (`ft-dot1x`)、FT-192dot1x (`ft-suiteb-192`) など、適切な FT AKM を既存の AKM と共に併記指定する必要がある。

---

### 2. コンフィグ読解・状態診断（show / debug コマンドの読み取り）

#### Catalyst 9800 WLC 上での WLAN / AKM 設定確認
`show wlan name <WLAN-NAME>` コマンドの出力から、暗号化スイートと AKM が要件通り設定されているか診断する。

```text
WLAN Profile Name     : CORP-WLAN
==================================================
Identifier            : 1
Network Name (SSID)   : CORP-SECURE
Status                : Enabled
802.11 Security
  802.11 Authentication : Open System
  Static WEP Keys       : Disabled
  Wi-Fi Protected Access (WPA/WPA2/WPA3):
    WPA               : Disabled
    WPA2              : Enabled
    WPA3              : Enabled
    WPA2 Ciphers      : CCMP128
    WPA3 Ciphers      : CCMP128, GCMP256
    PMF Option        : Mandatory
    PMF Dual Binding  : Disabled
    AKM List          : sae, ft-sae, suiteb-192
```

#### 接続クライアントのセキュリティステータス確認
`show wireless client mac-address <MAC> detail` コマンドで、接続クライアントが実際どの AKM / 暗号化方式でアソシエートしているか確認する。

```text
Client MAC Address : a483.e712.3456
Client IPv4 Address : 10.1.100.150
AP Name : AP-FLOOR1-01
WLAN ID : 1
WLAN Profile Name : CORP-WLAN
Security Group Tag : 0
Client State : Associated
Policy Profile : CORP-POLICY-PROFILE
Security Vector :
  WPA Version : WPA3
  Authentication Key Management : SAE (AKM 8)
  Cipher Pairwise : CCMP-128
  Cipher Group : CCMP-128
  PMF Status : In-Use (BIP-CMAC-128)
```

---

## 🛠 設定方法

### 1. Cisco Catalyst 9800 WLC (IOS-XE) CLI 設定例

#### Scenario A: WPA3 Personal (SAE Only) + 802.11w Mandatory 設定
WPA2 を完全に排除し、WPA3 SAE 専用 WLAN を構築する設定。

```bash
! --- 1. WLAN プロファイルの定義 ---
wlan WPA3-PERSONAL-WLAN 10 WPA3-PERSONAL-SSID
 security wpa
  no security wpa wpa2
  security wpa wpa3
  security wpa akm sae
  security wpa akm ft-sae
  security wpa pmf mandatory
 no shutdown

! --- 2. Policy Profile の定義と VLAN マッピング ---
wireless profile policy CORP-POLICY-PROFILE
 vlan 100
 central switching
 central dhcp
 no shutdown

! --- 3. Policy Tag の作成とプロファイルバインド ---
wireless tag policy CORP-POLICY-TAG
 wlan WPA3-PERSONAL-WLAN policy CORP-POLICY-PROFILE

! --- 4. AP への Policy Tag 割り当て ---
ap e4c7.2211.5588
 policy-tag CORP-POLICY-TAG
```

#### Scenario B: WPA2/WPA3 Transition Mode (Personal: PSK + SAE) 設定
既存の WPA2-PSK 端末と新しい WPA3-SAE 端末の混在を許可する設定。

```bash
wlan TRANSITION-WLAN 11 TRANSITION-SSID
 security wpa
  security wpa wpa2
  security wpa wpa2 ciphers aes
  security wpa wpa3
  security wpa akm psk
  security wpa akm sae
  security wpa akm ft-psk
  security wpa akm ft-sae
  security wpa psk set-key ascii 0 Cisco12345!Key
  security wpa pmf optional
 no shutdown
```

#### Scenario C: WPA3 Enterprise 192-bit Mode (CNSA 適合) + ISE 802.1X 設定
高セキュリティ 192 ビット暗号モードの設定。

```bash
! --- RADIUS サーバー (Cisco ISE) の定義 ---
radius server ISE-PRI
 address ipv4 10.1.10.50 auth-port 1812 acct-port 1813
 key 0 RADIUSSecretPass123!

aaa group server radius ISE-RADIUS-GROUP
 server name ISE-PRI

aaa authentication dot1x ISE-DOT1X-AUTH group ISE-RADIUS-GROUP

! --- 192-bit WPA3 Enterprise WLAN の作成 ---
wlan WPA3-ENT-192BIT 12 WPA3-ENTERPRISE-192BIT
 security wpa
  no security wpa wpa2
  security wpa wpa3
  security wpa wpa3 ciphers gcmp256
  security wpa akm suiteb-192
  security wpa akm ft-suiteb-192
  security wpa pmf mandatory
  security wpa pmf gmac-256
  security dot1x authentication ISE-DOT1X-AUTH
 no shutdown
```

---

### 2. Cisco AireOS WLC CLI 設定例

AireOS WLC（8.10 以降）における WPA3 SAE / Enterprise 設定。

```text
# --- WLAN の作成と WPA3 SAE 有効化 ---
config wlan create 20 WPA3-AireOS WPA3-SSID
config wlan security wpa enable 20
config wlan security wpa wpa2 disable 20
config wlan security wpa wpa3 enable 20
config wlan security wpa akm sae enable 20
config wlan security wpa akm psk disable 20
config wlan security pmf mandatory 20
config wlan enable 20
```

---

### 3. Cisco ISE 3.x 認証・認可ポリシー設定（WPA2/WPA3 Enterprise 802.1X 統合）

1. **Authentication Policy**:
   * **Rule Name**: Wireless_Dot1x_Auth
   * **Condition**: `Wireless_Access:NAS-Port-Type EQUALS Wireless - IEEE 802.11`
   * **Allowed Protocols**: `Default Network Access`（EAP-TLS, PEAP 必須有効化）
   * **Use Identity Source**: `Internal Endpoints` -> `Active Directory`

2. **Authorization Policy**:
   * **Rule Name**: Wireless_Employee_Access
   * **Conditions**: `Wireless_Access:NAS-Port-Type EQUALS Wireless` AND `Network Access:AuthenticationMethod EQUALS dot1x` AND `AD-USER-GROUPS EQUALS Domain Users`
   * **Permissions (Result Profile)**: `AccessAccept` + `dACL: PERMIT_ALL_TRAFFIC` + `SGT: Employees (Tag 10)`

---


## 🔍 検証コマンド

Cisco Catalyst 9800 WLC (IOS-XE) および AP 上での確認・トラブルシューティングコマンド一覧。

| 目的 | コマンド |
| --- | --- |
| **WLAN 設定確認** | <code>show wlan name &lt;WLAN-NAME&gt;</code> |
| **全 WLAN のステータス一覧確認** | <code>show wlan summary</code> |
| **接続クライアントの詳細状態（AKM/Cipher/PMF/IP）** | <code>show wireless client mac-address &lt;MAC&gt; detail</code> |
| **クライアント接続サマリー** | <code>show wireless client summary</code> |
| **4-Way Handshake / EAPOL 鍵交換デバッグ** | <code>debug wireless mac &lt;MAC&gt;</code> / <code>show logging profile wireless filter mac &lt;MAC&gt;</code> |
| **802.1X / RADIUS 認証デバッグ** | <code>debug aaa authentication</code> / <code>debug aaa authorization</code> |
| **WLC ワイヤレス トレースログ収集（Always-on Trace）** | <code>show logging profile wireless internal filter mac &lt;MAC&gt;</code> |
| **AP 側でのクライアント結合ログ確認** | <code>show controllers dot11Radio 0 client</code> (AP Console) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **WPA3-SAE クライアントがアソシエートできない** | PMF（802.11w）の設定不整合、または WLC の AKM 設定で `sae` が未設定。 | <code>show wlan name &lt;WLAN&gt;</code> | WPA3 単体運用の場合は <code>security wpa pmf mandatory</code> および <code>security wpa akm sae</code> を設定。Transition モード時は <code>pmf optional</code> を設定する。 |
| **WPA3 Enterprise 192-bit モードで端末が拒否される** | クライアント端末が GCMP-256 暗号化または BIP-GMAC-256 に非対応。 | <code>show wireless client mac-address &lt;MAC&gt; detail</code> | 端末の NIC ドライバおよび OS バージョンを確認し、CNSA/Suite B 対応端末（Intel AX200 以降等）を使用する。 |
| **WPA2/WPA3 Transition Mode で旧型端末が切断される** | 旧型レガシー端末が 802.11w PMF (IE) 宣言を正しく解釈できずアソシエート拒否。 | <code>show logging profile wireless internal filter mac &lt;MAC&gt;</code> | 旧型端末専用に WPA2-PSK 専用 SSID（WPA3/PMF Disabled）を分離定義して切り分ける。 |
| **4-Way Handshake タイムアウト（EAPOL Msg 2/4 未受信）** | クライアント側で設定された Pre-Shared Key / SAE Passphrase の不一致、または EAPOL フレームのドロップ。 | <code>show logging profile wireless filter mac &lt;MAC&gt;</code> | WLC 上の PSK 設定とクライアント入力パスワードを再確認。EAPOL Key タイムアウト値を微調整する。 |
| **802.1X EAP-TLS 認証失敗（Radius Reject）** | Cisco ISE 側の Allowed Protocols ポリシーで TLS 1.2/1.3 暗号スイートが不許可、または証明書有効期限切れ。 | ISE Live Logs (`Operations` -> `RADIUS` -> `Livelogs`) | ISE の Trusted Certificates および Client Certificate の有効性・CRL/OCSP を確認する。 |

---

## ⚠ 制限事項

1. **WPA3 と PMF (802.11w) の依存関係**:
   * WPA3 規格（SAE / Enterprise 192-bit / OWE）では、Protected Management Frames (PMF) の有効化が必須条件（Mandatory）である。WLC 上で PMF を `Disabled` に設定した状態で WPA3 を有効化することはできない。
2. **WPA3 Enterprise 192-bit Mode の互換性制約**:
   * 192-bit モード（CNSA / Suite B 適合）では、暗号化に GCMP-256、管理フレーム保護に BIP-GMAC-256、EAP-TLS の署名/鍵交換に SHA-384 および ECDHE-P-384 が要求されるため、大多数のレガシー端末（スマートフォン、一般 PC、IoT 端末）はアソシエート不能となる。
3. **Transition Mode におけるセキュリティダウングレードリスク**:
   * WPA2-PSK / WPA3-SAE Transition Mode では、攻撃者が偽装 AP を用いてクライアントに WPA2-PSK での接続を強制し、辞書攻撃を試みるダウングレード攻撃（Hunted-and-Pecked 攻撃等）を受ける可能性がある。

---

## 🔄 他技術との関連

* **Cisco ISE (RADIUS / TACACS+)**:
  * Enterprise モード（802.1X/EAP）において、ユーザー/端末の識別、Cert 認証、Active Directory / LDAP 連携、Dynamic VLAN 割り当て、dACL（Dynamic Access Control List）、SGT（Security Group Tag / TrustSec）の付与を担当する。
* **Protected Management Frames (PMF / IEEE 802.11w)**:
  * Deauthentication Flood や Disassociation Attack などの管理フレーム偽装 DoS 攻撃を防ぐ技術。WPA3 ではインフラとして不可分である。
* **Fast Transition (IEEE 802.11r)**:
  * AP 間ローミング時に 802.1X / SAE 認証をスキップし、事前計算された Fast Key（FT-802.1X / FT-SAE）を用いてハンドオーバー遅延を 50ms 未満に抑える技術。
* **Enhanced Open (OWE / IEEE 802.11ew)**:
  * パスワード認証を行わないオープン Wi-Fi において、Diffie-Hellman 鍵交換（Unauthenticated ECDH）を用いて個別データ通信を暗号化する技術。

---

## 🧩 比較表

### 1. WPA / WPA2 / WPA3 技術比較表

| 項目 | WPA | WPA2 | WPA3 Personal | WPA3 Enterprise 192-bit |
| --- | --- | --- | --- | --- |
| **標準規格** | IEEE 802.11i Draft | IEEE 802.11i Final | WPA3 Specification (Wi-Fi Alliance) | WPA3 CNSA / Suite B Mode |
| **暗号化アルゴリズム** | TKIP (RC4) | AES-CCMP-128 | AES-CCMP-128 / GCMP-128 | GCMP-256 |
| **データ完全性 (MIC)** | Michael (32-bit) | CBC-MAC (128-bit) | CBC-MAC / GMAC | GMAC-256 |
| **鍵交換 / 認証** | PSK / 802.1X | PSK / 802.1X | SAE (Dragonfly ECDH) | 802.1X / EAP-TLS (SHA-384) |
| **前方秘匿性 (Forward Secrecy)** | なし | なし | **あり**（セッション固有鍵生成） | **あり** |
| **辞書攻撃耐性** | 低い（オフライン解読可） | 低い（オフライン解読可） | **極めて高い**（オンラインのみ制限） | **極めて高い** |
| **802.11w PMF** | 非対応 | オプション (`Optional`) | **必須 (`Mandatory`)** | **必須 (`Mandatory` / GMAC-256)** |

### 2. Personal (PSK/SAE) vs Enterprise (802.1X) 比較表

| 項目 | Personal (PSK / SAE) | Enterprise (802.1X / EAP) |
| --- | --- | --- |
| **認証方式** | 事前共有鍵（Passphrase / Dragonfly） | ユーザー名/パスワード（PEAP）または証明書（EAP-TLS） |
| **認証サーバー** | 不要（AP / WLC 内部処理） | 必須（Cisco ISE / RADIUS サーバー） |
| **ユーザー固有鍵** | PSK: 全端末で同一鍵 / SAE: 端末ごとに固有セッション鍵 | ユーザー/端末ごとに固有の PMK / PTK を生成 |
| **一括失効・アクセス制限** | 不可（PSK 変更時は全端末の再設定が必要） | 容易（ISE 側でアカウント無効化・証明書失効・CoA 実行） |
| **適用場面** | 拠点オフィス、ゲスト用、IoT 機器 | 企業内正規社員端末、高セキュリティ端末 |

---

## 💡 ベストプラクティス

1. **WPA2 から WPA3 への段階的移行計画**:
   * 一括全切り替えが困難な環境では、WPA2/WPA3 Transition Mode を一時的に展開し、接続ログ（`show wireless client summary`）から WPA3 対応端末の割合をモニタリングする。移行完了後に SAE Only / PMF Mandatory に設定を強化する。
2. **PMF (802.11w) 設定の最適化**:
   * WPA2 Enterprise / PSK WLAN においても、セキュリティ強化のため PMF を `Optional` で有効化しておく。WPA3 移行時の互換性検証が容易になる。
3. **Cisco ISE 連携における EAP-TLS 推進**:
   * パスワード認証（PEAP-MSCHAPv2）は MS-CHAPv2 の脆弱性や証明書検証不備によるフィッシングリスクがあるため、WPA2/WPA3 Enterprise では EAP-TLS（クライアント証明書認証）を第一推奨とする。
4. **802.11r FT の併用設定**:
   * 音声端末（Cisco IP Phone）やモバイル端末の高速ローミングを担保するため、WPA3 SAE 時には `ft-sae`、802.1X 時には `ft-dot1x` を必ず併記設定する。


## 📝 ラボ学習・設定サンプル例

実際の CCIE Security v6.1 ラボ試験レベルを意識した、10 個の実践的な演習シナリオと完全な設定例・検証手順。

---

### サンプル 1: WPA3 Personal (SAE Only) & PMF Mandatory 構築

#### 問題
Catalyst 9800 WLC において、WPA2 を一切許可せず、WPA3 SAE（Simultaneous Authentication of Equals）のみを強制する高セキュリティ WLAN を構築しなさい。

#### 要件
* SSID: `SECURE-WPA3-ONLY`
* WLAN ID: `101`
* セキュリティ: WPA3 Personal (SAE Only)、WPA2 完全無効化
* パスフレーズ: `WPA3SecretPass2026!`
* PMF (802.11w): `Mandatory` 強制
* VLAN: VLAN `101` (Policy Profile: `POLICY-VLAN101`)

#### 設定例 (Catalyst 9800 WLC CLI)
```bash
! --- WLAN Profile 作成 ---
wlan WPA3-SAE-ONLY-WLAN 101 SECURE-WPA3-ONLY
 security wpa
  no security wpa wpa2
  security wpa wpa3
  security wpa akm sae
  security wpa sae psk set-key ascii 0 WPA3SecretPass2026!
  security wpa pmf mandatory
 no shutdown

! --- Policy Profile 作成 ---
wireless profile policy POLICY-VLAN101
 vlan 101
 central switching
 central dhcp
 no shutdown

! --- Policy Tag 作成と結び付け ---
wireless tag policy POLICY-TAG-101
 wlan WPA3-SAE-ONLY-WLAN policy POLICY-VLAN101

! --- AP への適用 ---
ap name AP-01
 policy-tag POLICY-TAG-101
```

#### 検証方法
```bash
# WLAN のセキュリティステータス確認
show wlan name WPA3-SAE-ONLY-WLAN

# クライアント接続後の SAE & PMF ステータス検証
show wireless client mac-address a483.e712.3456 detail | include Security|AKM|Cipher|PMF
```

---

### サンプル 2: WPA2/WPA3 Transition Mode (PSK + SAE) 構築

#### 問題
既存の WPA2-PSK 端末と最新の WPA3-SAE 端末が同一の SSID に接続可能な Transition Mode WLAN を構築しなさい。

#### 要件
* SSID: `CORP-TRANSITION-SSID`
* WLAN ID: `102`
* セキュリティ: WPA2-PSK (AES-CCMP) および WPA3-SAE の両方を有効化
* 共通パスフレーズ: `SharedCiscoPass123!`
* PMF (802.11w): `Optional`（WPA2 端末用）
* Fast Transition: `ft-psk` および `ft-sae` を併記指定

#### 設定例 (Catalyst 9800 WLC CLI)
```bash
wlan TRANSITION-WLAN 102 CORP-TRANSITION-SSID
 security wpa
  security wpa wpa2
  security wpa wpa2 ciphers aes
  security wpa wpa3
  security wpa akm psk
  security wpa akm sae
  security wpa akm ft-psk
  security wpa akm ft-sae
  security wpa psk set-key ascii 0 SharedCiscoPass123!
  security wpa pmf optional
 no shutdown
```

#### 検証方法
```bash
show wlan name TRANSITION-WLAN
# 出力で AKM に psk, sae, ft-psk, ft-sae が含まれ、PMF が Optional であることを確認
```

---

### サンプル 3: WPA3 Enterprise (802.1X/EAP-TLS) & Cisco ISE 統合構築

#### 問題
企業内正規端末向けに Cisco ISE 認証サーバーと連携する WPA3 Enterprise 802.1X WLAN を構築しなさい。

#### 要件
* SSID: `CORP-ENTERPRISE-WPA3`
* WLAN ID: `103`
* 認証方式: 802.1X (RADIUS サーバー群 `ISE-GROUP`)
* 暗号化: WPA3 Enterprise (AES-CCMP-128)、AKM `dot1x`
* PMF: `Mandatory`
* VLAN: VLAN `103`

#### 設定例 (Catalyst 9800 WLC CLI)
```bash
! --- AAA / RADIUS 設定 ---
radius server ISE-01
 address ipv4 10.1.10.50 auth-port 1812 acct-port 1813
 key 0 RadiusSharedKey123!

aaa group server radius ISE-GROUP
 server name ISE-01

aaa authentication dot1x DOT1X-AUTH-METHOD group ISE-GROUP

! --- WLAN 設定 ---
wlan WPA3-ENTERPRISE-WLAN 103 CORP-ENTERPRISE-WPA3
 security wpa
  no security wpa wpa2
  security wpa wpa3
  security wpa akm dot1x
  security wpa pmf mandatory
  security dot1x authentication DOT1X-AUTH-METHOD
 no shutdown
```

#### 検証方法
```bash
show wlan name WPA3-ENTERPRISE-WLAN
show wireless client summary
```

---

### サンプル 4: WPA3 Enterprise 192-bit Mode (CNSA 適合) 高信頼構築

#### 問題
政府機関・金融システム向けに CNSA (Suite B) 192 ビット暗号規格に準拠した WPA3 Enterprise WLAN を構築しなさい。

#### 要件
* SSID: `CNSA-192BIT-SECURE`
* WLAN ID: `104`
* ペアワイズ暗号: `GCMP-256`
* AKM: `suiteb-192` および `ft-suiteb-192`
* PMF / バイト整合性: `Mandatory` + `BIP-GMAC-256` (`gmac-256`)

#### 設定例 (Catalyst 9800 WLC CLI)
```bash
wlan WPA3-CNSA-192-WLAN 104 CNSA-192BIT-SECURE
 security wpa
  no security wpa wpa2
  security wpa wpa3
  security wpa wpa3 ciphers gcmp256
  security wpa akm suiteb-192
  security wpa akm ft-suiteb-192
  security wpa pmf mandatory
  security wpa pmf gmac-256
  security dot1x authentication DOT1X-AUTH-METHOD
 no shutdown
```

#### 検証方法
```bash
show wlan name WPA3-CNSA-192-WLAN
# Cipher Pairwise が GCMP256、PMF が Mandatory (GMAC-256) であることを確認
```

---

### サンプル 5: WPA3 SAE + Fast Transition (802.11r FT-SAE) 高速ローミング

#### 問題
WPA3 SAE 環境において、AP 間移動時のハンドオーバー遅延を極小化するため 802.11r FT-SAE を構築しなさい。

#### 要件
* SSID: `VOICE-WPA3-FT`
* WLAN ID: `105`
* AKM: `sae` および `ft-sae`
* Over-the-Air ローミング有効化
* Reassociation Timeout: `20` 秒

#### 設定例 (Catalyst 9800 WLC CLI)
```bash
wlan WPA3-FT-WLAN 105 VOICE-WPA3-FT
 security wpa
  no security wpa wpa2
  security wpa wpa3
  security wpa akm sae
  security wpa akm ft-sae
  security wpa ft reassociation-timeout 20
  security wpa ft over-the-air
  security wpa pmf mandatory
 no shutdown
```

#### 検証方法
```bash
show wlan name WPA3-FT-WLAN | include FT|AKM
```

---

### サンプル 6: WPA2-PSK WLAN に対する 802.11w PMF Mandatory 強制

#### 問題
既存の WPA2-PSK WLAN において、Deauth 攻撃を防ぐため 802.11w PMF を Mandatory に設定しなさい。

#### 要件
* SSID: `WPA2-PMF-MANDATORY`
* WLAN ID: `106`
* セキュリティ: WPA2-PSK (AES)
* PMF: `Mandatory`

#### 設定例 (Catalyst 9800 WLC CLI)
```bash
wlan WPA2-PMF-WLAN 106 WPA2-PMF-MANDATORY
 security wpa
  security wpa wpa2
  security wpa wpa2 ciphers aes
  security wpa akm psk
  security wpa psk set-key ascii 0 SecureKey9988!
  security wpa pmf mandatory
 no shutdown
```

#### 検証方法
```bash
show wlan name WPA2-PMF-WLAN
```

---

### サンプル 7: AireOS WLC における WPA3 SAE 設定

#### 問題
AireOS 8.10 WLC 上で WPA3 SAE 専用 WLAN を設定しなさい。

#### 要件
* WLAN ID: `30`
* SSID: `AireOS-WPA3-SAE`
* WPA3 SAE 有効、WPA2 無効、PMF Mandatory

#### 設定例 (AireOS WLC CLI)
```text
config wlan create 30 AireOS-WPA3-SAE AireOS-WPA3-SAE
config wlan security wpa enable 30
config wlan security wpa wpa2 disable 30
config wlan security wpa wpa3 enable 30
config wlan security wpa akm sae enable 30
config wlan security wpa akm psk disable 30
config wlan security pmf mandatory 30
config wlan enable 30
```

#### 検証方法
```text
show wlan 30
```

---

### サンプル 8: OWE (Enhanced Open / 802.11ew) 暗号化オープン WLAN 構築

#### 問題
キャプティブポータルやパスワード不要の公共/ゲスト Wi-Fi において、盗聴を防ぐため OWE（Opportunistic Wireless Encryption）を設定しなさい。

#### 要件
* SSID: `GUEST-ENHANCED-OPEN`
* WLAN ID: `108`
* AKM: `owe`
* PMF: `Mandatory`

#### 設定例 (Catalyst 9800 WLC CLI)
```bash
wlan OWE-GUEST-WLAN 108 GUEST-ENHANCED-OPEN
 security wpa
  no security wpa wpa2
  security wpa wpa3
  security wpa akm owe
  security wpa pmf mandatory
 no shutdown
```

#### 検証方法
```bash
show wlan name OWE-GUEST-WLAN
```

---

### サンプル 9: WPA3 Enterprise + Cisco ISE 認証（Dynamic SGT & dACL 連携）

#### 問題
WPA3 Enterprise 接続クライアントに対し、Cisco ISE から動的に Security Group Tag (SGT 15) と dACL を付与する設定をしなさい。

#### 要件
* WLAN ID: `109`
* SSID: `ISE-SECURE-WPA3`
* AAA 認証/認可/アカウンティング有効化
* AAA Override 有効化（Policy Profile 側）

#### 設定例 (Catalyst 9800 WLC CLI)
```bash
! --- AAA 設定 ---
aaa authorization network ISE-STAGE-AUTH group ISE-GROUP
aaa accounting network ISE-ACCT-GROUP start-stop group ISE-GROUP

! --- Policy Profile で AAA Override 有効化 ---
wireless profile policy ISE-POLICY-PROFILE
 aaa-override
 vlan 109
 central switching
 central dhcp
 accounting-list ISE-ACCT-GROUP
 no shutdown

! --- WLAN 設定 ---
wlan WPA3-ISE-WLAN 109 ISE-SECURE-WPA3
 security wpa
  no security wpa wpa2
  security wpa wpa3
  security wpa akm dot1x
  security wpa pmf mandatory
  security dot1x authentication DOT1X-AUTH-METHOD
 no shutdown
```

#### 検証方法
```bash
show wireless client mac-address a483.e712.3456 detail | include Security Group Tag|dACL
```

---

### サンプル 10: WPA2/WPA3 Transition Mode + FT (802.11r) ハイブリッド構成

#### 問題
WPA2-PSK、WPA3-SAE、FT-PSK、FT-SAE の 4 つの AKM を全て同時にサポートする統合ローミング WLAN を構築しなさい。

#### 要件
* SSID: `HYBRID-ROAMING-SSID`
* WLAN ID: `110`
* パスフレーズ: `HybridPassphrase2026!`
* PMF: `Optional`

#### 設定例 (Catalyst 9800 WLC CLI)
```bash
wlan HYBRID-WLAN 110 HYBRID-ROAMING-SSID
 security wpa
  security wpa wpa2
  security wpa wpa2 ciphers aes
  security wpa wpa3
  security wpa akm psk
  security wpa akm sae
  security wpa akm ft-psk
  security wpa akm ft-sae
  security wpa psk set-key ascii 0 HybridPassphrase2026!
  security wpa pmf optional
 no shutdown
```

#### 検証方法
```bash
show wlan name HYBRID-WLAN
```

---


## ❓ 想定試験問題

CCIE Security ラボ試験・実技判定を想定した実践的な問題群。

---

### 問題 1 (実装・トラブルシューティング)
**質問**: 新設した WPA3 SAE 専用 WLAN (`WLAN-ID: 15`) にクライアント端末が一切接続できず、アソシエート段階で切断される。WLC のコンフィグを確認したところ以下の状態であった。原因と適切な修正コマンドを答えなさい。

```text
wlan WPA3-TEST 15 WPA3-TEST-SSID
 security wpa
  no security wpa wpa2
  security wpa wpa3
  security wpa akm sae
  security wpa pmf disabled
 no shutdown
```

**回答・解説**:
* **原因**: WPA3 規格（SAE）では 802.11w Protected Management Frames (PMF) の有効化が必須（Mandatory）である。`security wpa pmf disabled` と設定されているため、WPA3 アソシエーション要件を満たせず通信が拒否される。
* **修正コマンド**:
  ```bash
  wlan WPA3-TEST 15
   security wpa pmf mandatory
  ```

---

### 問題 2 (Design・規格理解)
**質問**: 企業内で WPA3 Enterprise 192-bit モードを導入する際、WLC 上で設定すべきペアワイズ暗号化方式（Pairwise Cipher）および管理フレーム完全性保護アルゴリズム（Group Management Cipher）の正しい組み合わせを選びなさい。

1. AES-CCMP-128 / BIP-CMAC-128
2. GCMP-128 / BIP-CMAC-128
3. GCMP-256 / BIP-GMAC-256
4. TKIP / Michael

**回答・解説**:
* **正解**: 3
* **解説**: WPA3 Enterprise 192-bit Mode（CNSA / Suite B 適合）では、データパケット暗号化に **GCMP-256**、管理フレーム完全性保護に **BIP-GMAC-256**、および AKM `suiteb-192` の使用が規格上厳格に定められている。

---

### 問題 3 (コンフィグ読解・ローミング)
**質問**: 以下の WLC コンフィグが適用された WLAN において、クライアントが高速ローミング（Fast Transition）を行うためにサポートされている AKM の組み合わせをすべて挙げなさい。

```bash
wlan CORP-MOBILE 20 CORP-MOBILE-SSID
 security wpa
  security wpa wpa2
  security wpa wpa3
  security wpa akm psk
  security wpa akm sae
  security wpa akm ft-psk
  security wpa akm ft-sae
  security wpa pmf optional
 no shutdown
```

**回答・解説**:
* **正解**: `ft-psk` (WPA2 Fast Transition) および `ft-sae` (WPA3 Fast Transition)
* **解説**: コンフィグ内で `security wpa akm ft-psk` と `security wpa akm ft-sae` が明示的に有効化されているため、WPA2-PSK 接続端末は FT-PSK、WPA3-SAE 接続端末は FT-SAE を用いて 802.11r 高速ローミングを実行できる。

---

### 問題 4 (トラブルシューティング)
**質問**: WPA2/WPA3 Transition Mode を適用した環境で、一部の古いレガシー型 Wi-Fi 端末（バーコードスキャナ等）が SSID を検出できるものの接続を試みると即座にタイムアウトする。ネットワーク全体のアライメントを維持しつつ、原因を究明・対処するアプローチを説明しなさい。

**回答・解説**:
* **原因**: 一部の古い WPA2 専用端末は、Beacon / Probe Response 内に含まれる RSN IE (Information Element) 内の WPA3 / PMF (`pmf optional`) フラグを正しく処理できず、ドライバーレベルで接続動作を停止・拒否する。
* **対処方法**:
  1. トラブル端末の MAC アドレスに対して `show logging profile wireless internal filter mac <MAC>` を実行し、アソシエーションフレームの拒否理由コードを確認する。
  2. 互換性のないレガシー端末群に対しては、既存の Transition WLAN から分離し、PMF を完全無効化 (`security wpa pmf disabled`) した WPA2-PSK 専用のレガシー用 SSID を個別に割り当てる。

---

### 問題 5 (ISE 統合・認可ポリシー)
**質問**: WPA3 Enterprise (802.1X/EAP-TLS) 構成において、Cisco ISE が Access-Accept と共に送信する RADIUS 属性で、WLC 上での動的 VLAN / SGT 割り当てを可能にするために WLC の Policy Profile 側で必須となる設定コマンドを答えなさい。

**回答・解説**:
* **回答**: `aaa-override`
* **解説**: Cisco Catalyst 9800 WLC において、RADIUS サーバー（Cisco ISE）から返却される Dynamic VLAN（`Tunnel-Private-Group-ID`）や SGT（`cisco-av-pair: cts:security-group-tag`）を適用するには、該当する `wireless profile policy` 内で `aaa-override` コマンドが有効化されている必要がある。

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術資料。

* Cisco Catalyst 9800 Series Wireless Controller Software Configuration Guide - WPA3 Configuration
* Cisco Technical Notes - Deployment Guide: Wi-Fi Protected Access 3 (WPA3) Configuration on Catalyst 9800 WLC
* Cisco Live Slide - BRKWLN-2108: Deep Dive into WPA3 and Wi-Fi Security
* Cisco Live Video - BRKSEC-3055: Enterprise Wireless Security with WPA3 and Cisco ISE Integration
* Cisco Validated Design (CVD) - Wi-Fi 6 and WPA3 Security Design Guide
* Cisco Command Reference - Catalyst 9800 Wireless Controller Command Reference - wlan Command Group

---

## 📝 **補足（Notes）**

- **図解・キー階層マップ**:
  * PMK (802.1X/EAP または SAE/PSK) ──► 4-Way Handshake ──► PTK (KCK / KEK / TK) + GTK + IGTK
- **PMF (802.11w) 設定マトリクス**:
  * **WPA2 Only**: PMF `Disabled` または `Optional` または `Mandatory`
  * **WPA2/WPA3 Transition Mode**: PMF `Optional` 必須
  * **WPA3 SAE Only**: PMF `Mandatory` 必須
  * **WPA3 Enterprise 192-bit**: PMF `Mandatory` (`gmac-256`) 必須
- **注意点**:
  * Catalyst 9800 WLC (IOS-XE) では、WLAN 設定変更（AKM や Cipher 変更等）を行う前に `no shutdown` / `shutdown` によるプロファイルの再有効化が必要となる場合がある。設定反映後は必ず `show wlan name <WLAN>` で Status が `Enabled` になっているかテストすること。




## 📝 **補足（Notes）**
*   **学習メモ**: WPA(WPA1) は WPA2 の登場によりその役割を終えましたが、ラボ試験では「レガシー対応」や「設定ミス探し」として出題される可能性があります。
*   **注意点**: 実際の WLC GUI では `WPA Policy` のチェックボックスをオンにする必要があります。CLI と GUI の対応関係を整理しておきましょう。
