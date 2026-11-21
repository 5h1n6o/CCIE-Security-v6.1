---
layout: default
title: 4.13-Auth-methods
nav_order: 13
parent: 4.0-Identity-Management
---

# 4.13 Authentication methods

CCIE Security v6.1 Blueprintの「4.13 Authentication methods」では、企業ネットワークにおける有線・無線・VPN・ネットワーク機器管理アクセス（Administrative Access）での各種認証プロトコルおよび認証フレームワークの仕組み、パケットシーケンス、設定、検証、トラブルシューティングを扱います。本学習メモでは、IEEE 802.1X、MAB (MAC Authentication Bypass)、Web Authentication (CWA/LWA)、各種 EAP タイプ（EAP-TLS, PEAP, EAP-FAST, EAP-TTLS, EAP-MSCHAPv2, LEAP, EAP-MD5, EAP-GTC）、EAP Chaining、および AAA プロトコル（RADIUS, TACACS+）の実践的実装手法を包括的に整理します。

---

## 📘 概要

* **機能概要**:
  ネットワーク接続時にユーザーやデバイスの正当性を確認するセキュリティメカニズムです。レイヤ2ポートアクセスセキュリティ（802.1X, MAB, WebAuth）、Extensible Authentication Protocol (EAP) 認証フレームワーク、ネットワーク機器の管理アクセス保護（TACACS+ / RADIUS AAA）を含みます。Cisco Identity Services Engine (ISE) を Central Authentication Server として統合し、アイデンティティに応じた動的アクセス制御を実施します。
* **利用目的**:
  * 未許可デバイスや不正ユーザーによるネットワーク侵入の防止。
  * 端末種別（社内端末、BYOD、ゲスト、IoT）およびユーザー権限に応じた最小権限アクセス制御（Dynamic VLAN, dACL, SGT）の提供。
  * ネットワーク機器（Catalyst, WLC, ASA, FTD）の CLI/GUI 管理者アクセスにおける暗号化・詳細なログ監査（Command Accounting）。
* **どのような場面で利用するか**:
  * **キャンパス LAN / WLAN アクセス**: 社内 PC の 802.1X (EAP-TLS/PEAP) 認証、プリンタや IP 電話の MAB 認証、来訪者用 Guest WebAuth。
  * **リモートアクセス VPN**: AnyConnect クライアント認証における EAP-HI (EAP-TLS/PEAP) や SAML 2.0 / AAA 認証。
  * **インフラストラクチャ管理アクセス**: ネットワークエンジニアの IOS-XE/ASA SSH ログイン時の TACACS+ コマンド単位認可およびログ記録。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **特徴** | 認証（Authentication）、認可（Authorization）、会計（Accounting）を分離・統合管理。EAP 構造による柔軟な認証プロトコル選択。 |
| **用途** | 802.1X (証明書/パスワード), MAB (MACアドレス), WebAuth (キャプティブポータル), TACACS+ (機器管理AAA). |
| **メリット** | 不正接続の防止、アイデンティティベースの動的アクセスポリシー適用、全操作ログの中央化・改ざん防止。 |
| **デメリット** | 認証サーバー（Cisco ISE）への依存、証明書インフラ (PKI) やクライアント Supplicant 設定の運用オーバーヘッド。 |
| **対応機種** | Cisco Catalyst スイッチ (IOS-XE)、Cisco WLC (Catalyst 9800 / AireOS)、Cisco ASA / FTD、Cisco ISE. |
| **制限事項** | MAB は MAC アドレス偽装に脆弱。EAP-MSCHAPv2 は MS-CHAPv2 脆弱性のため TLS トンネル保護 (PEAP/EAP-FAST) が必須。 |
| **設計上の注意点** | ISE 障害時の Critical Authentication (Inaccessible Authentication Bypass) および RADIUS/TACACS+ タイムアウト設計が必須。 |

---

## 🏗 動作原理

ネットワーク認証は「Supplicant（クライアント）」「Authenticator（スイッチ/WLC/ASA）」「Authentication Server（Cisco ISE）」の 3 要素で構成されます。

### 1. 802.1X / EAP 認証アーキテクチャ通信フロー

```
[ Supplicant ]                [ Authenticator (Switch) ]                [ Auth Server (ISE) ]
      |                                   |                                        |
      |<------ EAP-Request/Identity ------|                                        |
      |------- EAP-Response/Identity ---->|                                        |
      |                                   |--- RADIUS Access-Request (EAP-Resp) -->|
      |                                   |<-- RADIUS Access-Challenge (EAP-Req) --|
      |<------ EAP-Request (TLS Init) ----|                                        |
      |                                   |                                        |
      |======================== TLS Tunnel Establishment =======================|
      |                                   |                                        |
      |------- EAP-Response (Auth Data) ->|                                        |
      |                                   |--- RADIUS Access-Request (Auth Data) ->|
      |                                   |<-- RADIUS Access-Accept (vlan/dACL/SGT)|
      |<------ EAP-Success ---------------|                                        |
      |                                   |                                        |
```

### 2. MAB (MAC Authentication Bypass) 通信フロー

```
[ Endpoint ]                  [ Authenticator (Switch) ]                [ Auth Server (ISE) ]
      |                                   |                                        |
      |--- (Link Up / Traffic Sent) ---->|                                        |
      | (802.1X Timeout or MAB Priority)  |                                        |
      |                                   |--- RADIUS Access-Request --------------|
      |                                   |    (User-Name=MAC, User-Password=MAC)  |
      |                                   |<-- RADIUS Access-Accept ---------------|
      |                                   |    (Dynamic VLAN / dACL / SGT)         |
```

### 3. Central Web Authentication (CWA) 通信フロー

```
[ Client ]           [ Switch / WLC ]              [ Cisco ISE ]             [ Internal / Internet ]
    |                      |                             |                             |
    |-- 1. MAB Auth ------>|                             |                             |
    |                      |-- 2. RADIUS Access-Req ---->|                             |
    |                      |<-- 3. RADIUS Access-Accept--|                             |
    |                      |      (Redirect-URL & ACL)   |                             |
    |-- 4. HTTP Request -->|                             |                             |
    |   (Intercepted)      |-- 5. HTTP Redirect -------->|                             |
    |-- 6. HTTPS Access Portal ------------------------->|                             |
    |-- 7. Submit Credentials -------------------------->|                             |
    |                      |<-- 8. RADIUS CoA (Re-auth)--|                             |
    |                      |--- 9. CoA ACK ------------->|                             |
    |                      |-- 10. Re-MAB Auth --------->|                             |
    |                      |<-- 11. RADIUS Access-Accept-| (PermitAccess)              |
    |-- 12. Full Network Access ------------------------------------------------------>|
```

---

## ⚙ 動作シーケンス

### 1. EAP-TLS 認証シーケンス
1. **EAPOL-Start / Identity**: Authenticator が EAP-Request/Identity を送信し、Supplicant が User/Machine 識別子を返答。
2. **TLS Handshake 開始**: ISE が Client Hello 要求を含んだ EAP-TLS Start を送信。Supplicant が Client Hello を返答。
3. **Server Certificate**: ISE が自身の Server Certificate を提示。Supplicant は Trusted Certificate Authority (CA) を検証。
4. **Client Certificate**: Supplicant が自身の Client Certificate を提示。ISE は Trusted CA 検索・CRL/OCSP 失効確認を実施。
5. **Key Exchange & TLS Tunnel**: Diffie-Hellman 鍵交換によりマスターセッションキー (MSK) を生成。
6. **Access-Accept & EAP-Success**: ISE が RADIUS Access-Accept（MSK、Dynamic VLAN/dACL/SGT 属性同封）をスイッチへ返送。スイッチが EAP-Success を送信しポートを Authorize。

### 2. PEAP-MSCHAPv2 認証シーケンス
1. **Phase 1 (TLS Tunnel)**: ISE のサーバー証明書のみを使用して暗号化 TLS トンネルを確立。クライアント証明書は不要。
2. **Phase 2 (Inner EAP)**: TLS トンネル内で EAP-MSCHAPv2 認証を実行。ISE と Supplicant 間で Challenge / Response による相互認証を実施。
3. **Identity Exchange**: 内部 EAP でのユーザー名認証および Active Directory (AD) / LDAP ドメインコントローラへの認証要求照会。
4. **Access-Accept**: 認証成功後、EAP-Success およびネットワーク権限属性をスイッチに送信。

### 3. EAP-FAST v2 (EAP Chaining) 認証シーケンス
1. **PAC (Protected Access Credential) Provisioning**: インバウンド/アウトオブバンドで PAC を配布（または In-band Automatic Provisioning）。
2. **Phase 1 Outer Tunnel**: PAC を用いて高速な TLS トンネルを確立。
3. **Phase 2 Inner EAP (Chaining)**:
   * **Machine Authentication**: ドメイン参加端末の証明書またはマシンアカウント認証。
   * **User Authentication**: ユーザーアカウントのパスワード/証明書認証。
4. **Compound Identity**: ISE はマシン認証とユーザー認証の両方が成功したことを確認し、"Machine + User Both Authenticated" ポリシーを適用。

---


## 🎯 試験対策（CCIE Securityラボ試験）

### 1. Blueprintで重要なポイント
* **認証メカニズムの評価順序（Order）と優先度（Priority）**:
  * スイッチのインターフェースレベルで `authentication order dot1x mab` および `authentication priority dot1x mab` を正確に設定できるか。
  * 802.1X に失敗した際に MAB へフォールバックさせる設定（`authentication event fail action next-method`）。
* **EAP タイプとアイデンティティソースの組み合わせ制約**:
  * **EAP-MSCHAPv2 / PEAP-MSCHAPv2**: Active Directory (AD) のみ対応。LDAP ドメインでは MSCHAPv2 の認証ハッシュ（NT-Hash）照会が不可能なため不可（LDAP では EAP-GTC または EAP-TLS を使用）。
  * **EAP Chaining**: EAP-FAST v2 のみでサポートされ、Cisco AnyConnect NAM (Network Access Manager) クライアントが必須。Windows 標準 Supplicant は非対応。
* **RADIUS / TACACS+ キーおよびタイムアウト設定**:
  * RADIUS 共有鍵の不一致（RADIUS Authentication Failed / Silence Drop）。
  * TACACS+ における Single-Connection オプション (`tacacs server <NAME> / single-connection`) の動作理解と設定。

### 2. よくある設定ミスと試験での落とし穴
* **CWA リダイレクト ACL の記述論理ミス**:
  * IOS-XE の IP リダイレクト ACL では、**ISE 宛て通信および DNS/DHCP 通信を `deny`（リダイレクト除外）**し、**その他の HTTP 通信を `permit`（リダイレクト対象）**にする必要があります。
  * `permit ip any any` にすると DNS や ISE 宛てリダイレクト通信自体がドロップしポータルが表示されません。
* **802.1X Host Mode の設定ミス**:
  * 1 ポートに複数端末（PC + IP Phone）を接続する場合に `host-mode multi-domain` または `multi-auth` を設定し忘れて `single-host` のままになり、2台目の端末接続時に Security Violation が発生する。
* **RADIUS CoA 関連の要件漏れ**:
  * スイッチ側で `aaa server radius dynamic-author` 設定および `client <ISE-IP> secret <KEY>` を入れ忘れ、ISE からの CoA Request (UDP 1700) がドロップする。

---

## 🛠 設定方法

### 1. Cisco Catalyst スイッチ (IOS-XE IBNS 2.0 / C3PL) 設定

```text
! AAA グローバル設定
aaa new-model
aaa authentication dot1x default group ISE_RADIUS
aaa authorization network default group ISE_RADIUS
aaa accounting dot1x default start-stop group ISE_RADIUS

! RADIUS サーバーグループおよび Dynamic Authorization (CoA)
radius server ISE_NODE1
 address ipv4 192.168.10.10 auth-port 1812 acct-port 1813
 key Cisco123!
aaa group server radius ISE_RADIUS
 server name ISE_NODE1

aaa server radius dynamic-author
 client 192.168.10.10 server-key Cisco123!
 server-key Cisco123!

! C3PL 802.1X / MAB クラスマップ & ポリシーマップ設定
class-map type authc match-all DOT1X_FAILED
 match authentication-failed
class-map type authc match-all DOT1X_NO_RESP
 match result dot1x no-response

policy-map type control subscriber ACCESS_PORT_POLICY
 event session-started trigger discovery
  10 class type address-pool MAB_POOL
  20 authenticate using dot1x priority 10
 event authentication-failure trigger authenticate
  10 class type authc DOT1X_FAILED action next-method
  20 class type authc DOT1X_NO_RESP action next-method
  30 class type authc default action terminate-session
 event agent-found trigger discovery
  10 evaluate

! インターフェース適用
interface GigabitEthernet1/0/1
 description 802.1X / MAB Access Port
 switchport mode access
 switchport access vlan 10
 mab
 dot1x pae authenticator
 dot1x timeout tx-period 7
 service-policy type control subscriber ACCESS_PORT_POLICY
```

### 2. Cisco Wireless LAN Controller (Catalyst 9800) 802.1X WLAN 設定

```text
! AAA RADIUS 設定
aaa new-model
aaa authentication login ISE_RADIUS_AUTH group ISE_RADIUS
aaa authorization network ISE_RADIUS_AUTHOR group ISE_RADIUS
aaa accounting network ISE_RADIUS_ACCT start-stop group ISE_RADIUS

radius server ISE_NODE1
 address ipv4 192.168.10.10 auth-port 1812 acct-port 1813
 secret Cisco123!

aaa group server radius ISE_RADIUS
 server name ISE_NODE1

! WLAN プロファイル設定
wlan CORPORATE_WLAN 1 Corp-SSID
 security ft
 security wpa akm ft dot1x
 security wpa wpa2 ciphers aes
 no shutdown

! Policy Profile 設定
wireless profile policy CORP_POLICY
 aaa-override
 nac
 vlan Corporate_VLAN
 no shutdown

! WLAN と Policy Profile のバインド
wireless tag policy DEFAULT_POLICY_TAG
 wlan CORPORATE_WLAN policy CORP_POLICY
```

### 3. TACACS+ 機器管理アクセス設定 (IOS-XE)

```text
! TACACS+ サーバー登録
tacacs server ISE_TACACS
 address ipv4 192.168.10.10
 key CiscoTacacs123!
 single-connection

aaa group server tacacs+ ISE_TACACS_GROUP
 server name ISE_TACACS

! TACACS+ AAA 設定
aaa authentication login default group ISE_TACACS_GROUP local
aaa authorization exec default group ISE_TACACS_GROUP local
aaa authorization commands 15 default group ISE_TACACS_GROUP local
aaa accounting exec default start-stop group ISE_TACACS_GROUP
aaa accounting commands 15 default start-stop group ISE_TACACS_GROUP
aaa authorization console
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| --- | --- |
| 認証セッション詳細確認 | `<code>show access-session interface Gi1/0/1 details</code>` |
| 全認証セッション概要一覧 | `<code>show access-session</code>` |
| RADIUS サーバー状態・統計 | `<code>show radius server-group ISE_RADIUS</code>` |
| TACACS+ サーバー状態確認 | `<code>show tacacs</code>` |
| Dot1x インターフェース状態 | `<code>show dot1x interface Gi1/0/1 details</code>` |
| リアルタイム AAA デバッグ | `<code>debug aaa authentication</code>` / `<code>debug aaa authorization</code>` |
| 802.1X 状態遷移デバッグ | `<code>debug dot1x all</code>` |
| RADIUS パケットデバッグ | `<code>debug radius verbose</code>` |
| C3PL セッションデバッグ | `<code>debug subscriber policy peer mac <MAC></code>` |

---


## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **802.1X タイムアウトになり MAB に移行しない** | `authentication event fail action next-method` がポリシーに設定されていない、または EAPOL 応答がサイレントドロップされている。 | `<code>show access-session interface Gi1/0/1 details</code>`, `<code>debug dot1x all</code>` | C3PL ポリシーマップに `event authentication-failure` および `no-response` で `action next-method` を追加。 |
| **EAP-MSCHAPv2 認証が失敗する (Error 5440)** | クライアントと ISE 間で MS-CHAPv2 パスワードハッシュが不一致、または AD アカウントがロックアウトされている。 | ISE Live Logs: Failure Reason `12104 MS-CHAPv2 Auth Failed`, `<code>debug aaa authentication</code>` | AD アカウント状態確認。ISE AD Join 状態 (`Test User` ボタン) を検証しドメインコントローラ通信を確認。 |
| **EAP-TLS 認証で証明書検証エラー (Error 5400)** | ISE 信頼ストアにクライアント証明書を発行した Intermediate/Root CA 証明書が登録されていない、または CRL/OCSP 失効。 | ISE Live Logs: Failure Reason `12514 Certificate chain untrusted`, `<code>show crypto pki certs</code>` | ISE の Trusted Certificates に上位 CA 証明書をインポートし `Trust for client authentication` を有効化。 |
| **RADIUS CoA が拒否される / タイムアウト** | スイッチに `aaa server radius dynamic-author` が設定されていない、または UDP 1700 が Firewall で遮断されている。 | `<code>show radius server-group ALL</code>`, スイッチ側 `<code>debug radius verbose</code>` | スイッチに `aaa server radius dynamic-author` と `client <ISE-IP> secret <KEY>` を定義。FW で UDP 1700 を許可。 |
| **CWA でポータル画面にリダイレクトされない** | リダイレクト ACL で ISE IP / DNS / DHCP が Permit になっている（Permit された通信はリダイレクトされない）。 | `<code>show ip access-lists <ACL_NAME></code>`, `<code>show access-session interface Gi1/0/1 details</code>` | リダイレクト ACL で ISE / DNS / DHCP 通信を `deny` に変更し、その他の HTTP (TCP 80) 通信を `permit` に設定。 |
| **TACACS+ コマンド認可が拒否される** | ISE Command Set に対象コマンドが定義されていない、または `aaa authorization console` が未設定。 | `<code>show running-config \| section aaa</code>`, ISE Live Logs TACACS Livelogs | ISE Command Sets で当該コマンドを許可。Console 認可が必要な場合は `aaa authorization console` を追加。 |

---

## ⚠ 制限事項

* **MAB セキュリティ制限**:
  * MAB は送信元 MAC アドレスのみに依存するため、MAC アドレス偽装（MAC Spoofing）に対して無力です。802.1X が使用できない IoT/プリンタ端末専用とし、802.1X ポートでの自動フォールバック運用を推奨します。
* **EAP-MSCHAPv2 / LDAP 制限**:
  * LDAP ディレクトリは平文パスワードまたは可逆暗号化パスワードを格納していない限り、EAP-MSCHAPv2 認証に必要な NT ハッシュを計算できません。そのため、LDAP 外部 ID ソースでは EAP-TLS または EAP-GTC (要 Supplicant プラグイン) のみ使用可能です。
* **EAP Chaining の Supplicant 制限**:
  * EAP Chaining (EAP-FAST v2) は、Cisco AnyConnect NAM (Network Access Manager) でのみ動作します。Windows 10/11 や macOS の標準 802.1X Supplicant では EAP Chaining はサポートされません。

---

## 🔄 他技術との関連

* **Dynamic Authorization (RADIUS CoA)**:
  * 認証完了後に属性の変更（Guest CWA ログイン後の PermitAccess 昇格、Posture パス後の Quarantine 解除、CoA Re-auth/Port-Bounce）を実行。
* **Cisco TrustSec (SGT / SXP)**:
  * RADIUS Access-Accept に Security Group Tag (SGT: cisco-av-pair `option:sgt-tag=xxxx`) を付与し、スイッチのインポートでインラインタグ付けまたは SXP 配信を実行。
* **Profiling & Posture Assessment**:
  * MAB 認証時に Device Sensor や Profiling Probe（DHCP/HTTP）で端末属性を収集し、プロファイル識別後に CoA を発行して認証・認可ポリシーを動的更新。
* **High Availability & Critical Auth**:
  * ISE 全ノードダウン時にポートを停止させず特定 VLAN（Critical VLAN）へ割り当てる Local Critical Authentication (Inaccessible Authentication Bypass) 機能。

---

## 🧩 比較表

### 1. 各種 EAP 認証プロトコルの比較

| 認証プロトコル | クライアント証明書 | サーバー証明書 | トンネル保護 | アイデンティティソース | 主な用途・特徴 |
| --- | --- | --- | --- | --- | --- |
| **EAP-TLS** | **必須** | **必須** | なし (TLS直接) | AD / LDAP / Internal / Certificate | 最高レベルのセキュリティ。証明書ベースの相互認証。 |
| **PEAP-MSCHAPv2** | 不要 | **必須** | あり (PEAP Phase 1) | Active Directory のみ | パスワード認証の業界標準。Windows 標準 Supplicant 対応。 |
| **EAP-FAST v2** | 任意 (Chaining) | 任意 (PAC/Cert) | あり (FAST Phase 1) | Active Directory | EAP Chaining (マシン + ユーザー同時認証) に対応。AnyConnect NAM 必須。 |
| **EAP-TTLS** | 不要 (または任意) | **必須** | あり (TTLS Phase 1) | AD / LDAP / Internal | レガシー非 AD (LDAP/PAP/CHAP) での暗号化パスワード認証。 |
| **EAP-MD5** | 不要 | 不要 | なし (平文同等) | Internal / Cleartext | 暗号化なしのチャレンジ/レスポンス。現代では非推奨。 |

### 2. ネットワークアクセス認証方式の比較

| 項目 | IEEE 802.1X | MAB (MAC Auth Bypass) | Web Authentication (CWA) |
| --- | --- | --- | --- |
| **対象端末** | Supplicant 搭載 PC / スマートフォン | プリンタ / IP カメラ / IoT 機器 | ゲスト / ブラウザ搭載未登録端末 |
| **認証情報** | X.509 証明書 / ドメインパスワード | エンドポイントの MAC アドレス | ブラウザで入力するユーザー名/パスワード |
| **セキュリティ強度** | **最高** (EAP 暗号化 / 相互認証) | **低** (MAC 偽装が容易) | **中** (HTTPS ポータル暗号化) |
| **プロトコル** | EAPOL (Layer 2) / RADIUS | Layer 2 MAC Lookup / RADIUS | HTTP/HTTPS Redirect / RADIUS / CoA |

### 3. RADIUS vs TACACS+ 比較表

| 項目 | RADIUS (RFC 2865 / 2866) | TACACS+ (RFC 8907) |
| --- | --- | --- |
| **プロトコル / ポート** | UDP 1812 (Auth), UDP 1813 (Acct) | TCP 49 (Single Connection 可) |
| **AAA の分離** | Authentication と Authorization が結合 | Authentication, Authorization, Accounting が完全分離 |
| **パケット暗号化** | パスワードフィールドのみ暗号化 (MD5) | ヘッダーを除く**パケット全体を暗号化** |
| **コマンド単位認可** | 非対応 | **対応** (入力コマンドごとに ISE へ認可照会) |
| **主な用途** | ネットワークアクセス (802.1X, MAB, VPN) | ネットワーク機器の管理アクセス (SSH/Console) |

---

## 💡 ベストプラクティス

1. **802.1X + MAB ハイブリッドポート設計**:
   * キャンパスポートでは `dot1x` を優先度 1 (`priority 10`)、`mab` を優先度 2 (`priority 20`) とし、802.1X 失敗時に無駄なタイムアウトを発生させないよう `tx-period 7` / `max-reauth-req 2` に短縮設定する。
2. **EAP-TLS と CRL / OCSP の信頼性設計**:
   * EAP-TLS 認証において CRL / OCSP 失効チェックサーバーにアクセスできない場合のフォールバックポリシー（`Ignore / Continue` または `Fail`）を要件に応じて正しく設計する。
3. **TACACS+ Single Connection の活用**:
   * ネットワーク機器数が多く大量の CLI コマンドを実行する環境では、`single-connection` オプションを有効化し TCP ハンドシェイクのオーバヘッドを削減する。
4. **RADIUS 認証サーバー障害対策 (Critical Auth)**:
   * ISE 全台との通信断時に全ポートが遮断されないよう、スイッチ側で `event server dead action authorize vlan <CRITICAL_VLAN>` を設定する。

---


## 📝 ラボ学習・設定サンプル例

### 演習例 1: Catalyst スイッチでの 802.1X (EAP-TLS/PEAP) 認証と MAB フォールバック設定

* **問題**:
  Catalyst スイッチ Gi1/0/1 に接続する社内 PC に対して 802.1X 認証（EAP-TLS または PEAP）を実行し、非 802.1X 端末（プリンタ等）は MAB 認証へ自動フォールバックさせる C3PL ポリシーを構築せよ。
* **要件**:
  1. 802.1X 認証を優先（Priority 10）、MAB をセカンダリ（Priority 20）とする。
  2. 802.1X の応答がない場合（EAPOL タイムアウト）または失敗した場合は即座に MAB を試行すること。
  3. EAPOL タイムアウト時間をデフォルトの 30 秒から 7 秒へ短縮すること。
* **設定例**:

```text
aaa new-model
aaa authentication dot1x default group ISE_RADIUS_GROUP
aaa authorization network default group ISE_RADIUS_GROUP
aaa accounting dot1x default start-stop group ISE_RADIUS_GROUP

radius server ISE_PRIMARY
 address ipv4 192.168.10.10 auth-port 1812 acct-port 1813
 key Cisco123!

aaa group server radius ISE_RADIUS_GROUP
 server name ISE_PRIMARY

aaa server radius dynamic-author
 client 192.168.10.10 server-key Cisco123!

class-map type authc match-all DOT1X_FAILED
 match authentication-failed
class-map type authc match-all DOT1X_NO_RESP
 match result dot1x no-response

policy-map type control subscriber PORT_POLICY
 event session-started trigger discovery
  10 class type address-pool MAB_POOL
  20 authenticate using dot1x priority 10
 event authentication-failure trigger authenticate
  10 class type authc DOT1X_FAILED action next-method
  20 class type authc DOT1X_NO_RESP action next-method
  30 class type authc default action terminate-session

interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
 mab
 dot1x pae authenticator
 dot1x timeout tx-period 7
 dot1x max-req 2
 service-policy type control subscriber PORT_POLICY
```

* **検証方法**:
  * 802.1X クライアント接続時: `show access-session interface Gi1/0/1 details` で Method が `dot1x`、Status が `Authorized` であることを確認。
  * 非 802.1X 端末接続時: 7 秒経過後に Method が `mab` に切り替わり認証成功することを確認。

---

### 演習例 2: EAP-FAST v2 (EAP Chaining) によるマシン・ユーザー同時認証設定

* **問題**:
  Cisco AnyConnect NAM を使用するドメイン参加 PC に対して、マシン認証とユーザー認証の両方を同時に要求する EAP Chaining (EAP-FAST) を構成せよ。
* **要件**:
  1. EAP-FAST 内部でマシン認証（Computer Certificate / Active Directory Account）とユーザー認証（AD Password）を要求する。
  2. ISE にて 両方が成功した場合にのみ フルアクセス（VLAN 10 + SGT 4）を付与し、片方のみの場合は制限付き VLAN (VLAN 99) を付与すること。
* **設定例**:
  * **ISE Side Configuration**:
    1. `Allowed Protocols` で `EAP-FAST` を有効化し、`Enable EAP Chaining` にチェックを入れる。
    2. Policy Set の Authorization Rule を以下のように作成:
       * Rule 1: `EAP-FAST:CompoundIdentityResult EQUALS Machine and user authenticated` -> Authz Result: `VLAN10_SGT4`
       * Rule 2: `EAP-FAST:CompoundIdentityResult EQUALS Machine authenticated only` -> Authz Result: `VLAN99_Restricted`
  * **Switch Configuration**:

```text
interface GigabitEthernet1/0/2
 switchport mode access
 switchport access vlan 10
 dot1x pae authenticator
 service-policy type control subscriber PORT_POLICY
```

* **検証方法**:
  * AnyConnect NAM 接続ログおよび ISE Live Logs にて `EAP-FAST (EAP Chaining)` が適用され、`CompoundIdentityResult = Machine and user authenticated` となっていることを確認。

---

### 演習例 3: Central Web Authentication (CWA) と Dynamic RADIUS CoA 連携設定

* **問題**:
  未登録端末接続時に MAB を実行し、Cisco ISE の Web ポータルへ強制リダイレクトさせる CWA 構成を構築せよ。
* **要件**:
  1. 初期 MAB 認証成功時、ISE から Redirect URL および Redirect ACL 名 `REDIRECT_CWA` を返送する。
  2. スイッチで HTTP 通信を捕捉して ISE ポータルへ誘導し、ポータルログイン成功後に ISE から RADIUS CoA Re-auth を受信して通信を全解放（PermitAccess）すること。
* **設定例**:

```text
! リダイレクト ACL の作成（ISE / DNS / DHCP 通信は deny して除外）
ip access-list extended REDIRECT_CWA
 deny udp any any eq domain
 deny udp any eq bootps any
 deny udp any any eq bootpc
 deny ip any host 192.168.10.10
 permit tcp any any eq www
 permit tcp any any eq 443

! Dynamic Authorization (CoA) 設定
aaa server radius dynamic-author
 client 192.168.10.10 server-key Cisco123!

interface GigabitEthernet1/0/3
 switchport mode access
 switchport access vlan 20
 mab
 dot1x pae authenticator
 service-policy type control subscriber PORT_POLICY
```

* **検証方法**:
  * 端末接続時に `show access-session interface Gi1/0/3 details` で Status が `Authz Success` かつ `URL Redirect` および `Redirect ACL` が適用されていることを確認。
  * ポータルログイン後、`debug radius verbose` にて RADIUS CoA-Request (Code 40) を受信し、セッションが更新されてリダイレクトが解除されることを確認。

---

### 演習例 4: RADIUS Change of Authorization (CoA) 設定と Port-Bounce / Re-Auth 検証

* **問題**:
  Cisco ISE からスイッチへの Dynamic CoA 要求（Re-Auth および Port-Bounce）を正しく受信・処理できるようにスイッチを設定せよ。
* **要件**:
  1. RADIUS Dynamic Author ポートとして標準 UDP 1700 を使用する。
  2. CoA パケットの送信元 IP（ISE PSN）および共有鍵を厳格に照合すること。
* **設定例**:

```text
aaa server radius dynamic-author
 client 192.168.10.10 server-key Cisco123!
 port 1700
 auth-type any

interface GigabitEthernet1/0/4
 switchport mode access
 service-policy type control subscriber PORT_POLICY
```

* **検証方法**:
  * ISE Context Visibility ディレクトリで対象端末を選択し、`CoA -> Re-authenticate` または `Port-Bounce` を実行。
  * スイッチ側で `show logging` または `debug radius verbose` を監視し、`CoA-ACK` が即座に返信されることを確認。

---

### 演習例 5: TACACS+ による IOS-XE SSH 管理者認証およびコマンド認可設定

* **問題**:
  IOS-XE スイッチへの SSH ログイン時に ISE TACACS+ による個人アカウント認証を行い、特権モード（Privilege Level 15）におけるコマンド実行を認可制御せよ。
* **要件**:
  1. SSH ログイン認証、EXEC 認可、および Level 15 コマンド認可を TACACS+ で実施する。
  2. ISE 障害時はローカルローカルデータベースへフォールバックすること。
  3. 実行された全コマンドを TACACS+ 会計ログとして記録すること。
* **設定例**:

```text
username admin-local secret CiscoLocal123!

tacacs server ISE_TACACS_1
 address ipv4 192.168.10.10
 key CiscoTacacs123!
 single-connection

aaa group server tacacs+ ISE_TACACS_GROUP
 server name ISE_TACACS_1

aaa authentication login default group ISE_TACACS_GROUP local
aaa authorization exec default group ISE_TACACS_GROUP local
aaa authorization commands 15 default group ISE_TACACS_GROUP local
aaa accounting exec default start-stop group ISE_TACACS_GROUP
aaa accounting commands 15 default start-stop group ISE_TACACS_GROUP
aaa authorization console

line vty 0 4
 transport input ssh
 login authentication default
 authorization exec default
 authorization commands 15 default
```

* **検証方法**:
  * SSH 接続時: AD/ISE アカウントでログイン可能か検証。
  * コマンド実行時: ISE に未許可のコマンドを入力すると `% Command authorization failed.` となることを確認。
  * `show tacacs` で Single-Connection 通信状態を確認。

---

### 演習例 6: ASA Remote Access VPN における AnyConnect EAP-TLS / PEAP 認証設定

* **問題**:
  Cisco ASA 9.x において、AnyConnect リモートアクセス VPN クライアント認証を Cisco ISE へ RADIUS 経由で委譲する構成を作成せよ。
* **要件**:
  1. AAA サーバーグループとして ISE を登録し、認証・会計ポートを 1812/1813 とする。
  2. AnyConnect クライアントの 802.1X / EAP 認証プロキシ処理を実行すること。
* **設定例**:

```text
aaa-server ISE_GROUP protocol radius
aaa-server ISE_GROUP (inside) host 192.168.10.10
 key Cisco123!
 authentication-port 1812
 accounting-port 1813

tunnel-group ANYCONNECT_TG type remote-access
tunnel-group ANYCONNECT_TG general-attributes
 authentication-server-group ISE_GROUP
 accounting-server-group ISE_GROUP
 default-group-policy ANYCONNECT_GP

group-policy ANYCONNECT_GP internal
group-policy ANYCONNECT_GP attributes
 vpn-tunnel-protocol ssl-vpn
```

* **検証方法**:
  * ASA CLI にて `show aaa-server ISE_GROUP` を実行し、ISE サーバーが UP であることを確認。
  * AnyConnect クライアントから接続し、`show vpn-sessiondb anyconnect` で認証セッション状態を確認。

---

### 演習例 7: Catalyst 9800 WLC における 802.1X (WPA3-Enterprise) AAA 統合設定

* **秘密**:
  Catalyst 9800 WLC において、WPA3-Enterprise (802.1X) SSID を構築し、Cisco ISE による動的 AAA Override (VLAN 割当) を有効化せよ。
* **設定例**:

```text
radius server ISE_PSN1
 address ipv4 192.168.10.10 auth-port 1812 acct-port 1813
 secret Cisco123!

aaa group server radius ISE_WLAN_GROUP
 server name ISE_PSN1

aaa authentication login WLAN_AUTH_LIST group ISE_WLAN_GROUP
aaa authorization network WLAN_AUTHZ_LIST group ISE_WLAN_GROUP

wlan WPA3_CORP 2 WPA3-Corp-SSID
 security wpa akm dot1x
 security wpa wpa3 ciphers aes
 no shutdown

wireless profile policy WPA3_POLICY
 aaa-override
 nac
 vlan 10
 no shutdown

wireless tag policy GLOBAL_POLICY_TAG
 wlan WPA3_CORP policy WPA3_POLICY
```

* **検証方法**:
  * 9800 CLI にて `show wireless client summary` および `show wireless client mac <MAC> detail` を実行し、`AAA Override: Yes` かつ割り当て VLAN が適用されていることを確認。

---

### 演習例 8: OpenLDAP 外部 ID ソースを用いた EAP-TLS / EAP-TTLS 認証設定

* **問題**:
  Cisco ISE において 外部 OpenLDAP サーバーを Identity Source として登録し、LDAP 属性に基づく 802.1X EAP-TTLS 認証および EAP-TLS 認証を構成せよ。
* **要件**:
  1. LDAPS (TCP 636) 通信を保護するため ISE Trusted Store に OpenLDAP CA 証明書を登録する。
  2. LDAP 属性 `mail` および `groupMembership` を引き出し、認可ポリシーで使用すること。
* **設定例 (ISE 設定要件)**:
  1. `Administration -> Identity Management -> External Identity Sources -> LDAP` にて新規 LDAP ソース追加。
  2. Connection Tab: Host `10.0.0.50`, Port `636`, `Use Secure SSL` にチェック。
  3. Directory Organization Tab: Root DN `dc=example,dc=com`, User Directory Subtree `ou=users`.
  4. Attributes / Groups Tab: `memberOf` 属性をグループマッピングに登録。
* **検証方法**:
  * LDAP ソースページの `Test Connection` および `Fetch Groups` を実行し、エラーなくグループ情報が取得できることを確認。

---

### 演習例 9: Active Directory 統合環境での PEAP-MSCHAPv2 認証と Password Change 連携

* **問題**:
  Cisco ISE を Active Directory ドメイン `corp.local` に Join させ、PEAP-MSCHAPv2 認証およびパスワード期限切れ時の MS-CHAPv2 Password Change 機能を有効化せよ。
* **設定例 (ISE GUI & CLI)**:
  1. `Administration -> Identity Management -> External Identity Sources -> Active Directory` にて `corp.local` を追加し、Domain Admin 資格情報で Join 処理を実行。
  2. `Allowed Protocols -> PEAP` 設定にて `Enable Password Change` オプションにチェックを入れる。
* **検証方法**:
  * AD 側でテストユーザーの `User must change password at next logon` フラグを有効化。
  * クライアント Supplicant から接続を試み、パスワード変更ダイアログが表示され新パスワードで更新・認証成功することを確認。

---

### 演習例 10: RADIUS Critical Authentication (Inaccessible Authentication Bypass) 高可用性構成

* **秘密**:
  スイッチと Cisco ISE 間のネットワーク断線や ISE 全ノード障害発生時に、ポートを完全にブロックせず特定のエマージェンシー VLAN (VLAN 999) へ疎通させる Critical Auth 機能を構成せよ。
* **設定例**:

```text
aaa server radius dynamic-author
 client 192.168.10.10 server-key Cisco123!

policy-map type control subscriber CRITICAL_PORT_POLICY
 event session-started trigger discovery
  10 class type address-pool MAB_POOL
  20 authenticate using dot1x priority 10
 event server-dead trigger server-dead
  10 authorize vlan 999
 event server-alive trigger server-alive
  10 action re-authenticate

interface GigabitEthernet1/0/10
 switchport mode access
 switchport access vlan 10
 mab
 dot1x pae authenticator
 service-policy type control subscriber CRITICAL_PORT_POLICY
```

* **検証方法**:
  * スイッチと RADIUS サーバー間の通信を遮断（または ACL で 1812 をドロップ）。
  * 端末接続時に `show access-session interface Gi1/0/10 details` で Status が `Authorized`、VLAN が `999` (Critical VLAN) となっていることを確認。

---


## ❓ 想定試験問題

### 問題 1 (コンフィグ読解)
IOS-XE C3PL コンフィグにおいて 802.1X 非対応端末が接続された場合の動作を選択せよ。

```text
policy-map type control subscriber MY_POLICY
 event session-started trigger discovery
  10 authenticate using dot1x priority 10
 event authentication-failure trigger authenticate
  10 class type authc DOT1X_FAILED action terminate-session
```

* A. 802.1X 認証失敗時、即座に MAB へ移行する。
* B. EAPOL 応答がない場合、セッションが終了し MAB は試行されない。
* C. デフォルト VLAN に割り当てられる。
* D. アクセスポートが開放される。

**正解**: B
**解説**: ポリシーマップで `action next-method` が設定されていないため、MAB へ切り替わりません。

--- 

### 問題 2 (トラブルシューティング)
802.1X (PEAP-MSCHAPv2) 認証で「12104 MS-CHAPv2 Auth Failed」のエラーログが出力された場合の根本原因はどれか。

* A. RADIUS Shared Key 不一致。
* B. サーバー証明書の非信頼。
* C. パスワード不一致または AD アカウントのロックアウト/無効化。
* D. AAA Override 未設定。

**正解**: C
**解説**: MS-CHAPv2 認証失敗または AD アカウント無効化を示します。

--- 

### 問題 3 (Design)
LDAP サーバーを外部 ID ソースとする環境で、追加プラグインなしに標準機能で最も安全に実装できる EAP 方式はどれか。

* A. PEAP-MSCHAPv2
* B. EAP-TLS
* C. LEAP
* D. EAP-MD5

**正解**: B
**解説**: LDAP では MSCHAPv2 ハッシュ計算不可のため EAP-TLS が最適です。

--- 

### 問題 4 (実装)
CWA 設定時の Redirect ACL の適切な記述はどれか。

* A. ISE 宛て通信を permit する。
* B. 除外したい通信 (DNS/DHCP/ISE) を deny し、HTTP 通信を permit する。
* C. permit ip any any にする。
* D. スイッチ側設定不要。

**正解**: B
**解説**: permit されたパケットが HTTP リダイレクト対象となるため、除外通信は deny にします。

--- 

### 問題 5 (コンフィグ読解)
TACACS+ サーバーが全台ダウンした場合の動作を選択せよ。

```text
aaa authentication login default group ISE_TACACS_GROUP local
```

* A. 即座にログイン拒否される。
* B. タイムアウト時、ローカルデータベース認証へフォールバックする。
* C. Access-Reject 時もローカル認証する。
* D. 無認証でログインできる。

**正解**: B
**解説**: TACACS+ 通信不可（サーバーダウン）時のみ local へフォールバックします。

--- 

## 🔗 参考リソース

* Cisco ISE Administrator Guide
* Cisco IOS-XE IBNS 2.0 Deployment Guide
* Cisco Catalyst 9800 WLC AAA Configuration Guide
* Cisco Live Presentation - BRKSEC-3699

--- 

## 📝 補足（Notes）

* **802.1X Timers Optimization**: `tx-period 7` と `max-req 2` によるタイマー最適化。
* **EAP-TLS Courtesy Check**: AD アカウント有効状態照会による失効チェック補完。
* **SAML 2.0 / IdP 連携**: Azure AD (Entra ID) や Okta との連携推進。


