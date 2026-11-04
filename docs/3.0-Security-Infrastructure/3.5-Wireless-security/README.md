---
layout: default
title: 3.5-Wireless-security
nav_order: 5
parent: 3.0-Security-Infrastructure
---

# 3.5 Wireless security technologies

## 📘 概要

ワイヤレスセキュリティ（Wireless Security）は、電波という物理的境界が存在しない共有メディアにおいて、通信の機密性（Confidentiality）、完全性（Integrity）、認証（Authentication）、および可用性（Availability）を確保するための技術体系である。CCIE Security v6.1 Blueprint 3.5におけるワイヤレスセキュリティは、WPA2/WPA3によるリンク層暗号化・認証、802.1X/EAP（EAP-TLS, PEAP, EAP-FAST）による高度なエンタープライズ認証、Cisco ISEと連携した Central Web Authentication (CWA) や Identity PSK (iPSK)、Management Frame Protection (MFP / PMF)、Rogue AP/Client の検出・抑制（WIDS/WIPS / RLDP）、ならびに Anchor/Foreign WLC 構成によるゲストアクセス制御まで多岐にわたる。

ネットワークインフラにおいて、キャンパスLAN、リモート拠点（FlexConnect）、SD-Access (Fabric) 環境ごとに適切なセキュリティモデルを選択・実装し、不正アクセスや中間者攻撃（MitM）、DoS攻撃からワイヤレスインフラを防御することが本技術の主目的である。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **特徴** | 物理層・データリンク層における電波暗号化（AES-CCMP/GCMP）、802.1X/EAP動的鍵交換、制御・管理フレーム保護（PMF/MFP）、および分散/集中管理型ポリシー制御。 |
| **用途** | 企業内LAN端末の安全なアクセス（Enterprise 802.1X）、ゲスト端末の保護・隔離（CWA/Anchor WLC）、IoT端末の個別に識別可能な接続（iPSK）、シャドーIT/Rogue APの自動排除。 |
| **メリット** | 物理線に依存しない柔軟な接続性と、有線LAN同等以上のゼロトラストセキュリティ（Dynamic VLAN, SGT, dACL）の実現。WPA3による辞書攻撃耐性およびOWEによる暗号化パブリックWi-Fi。 |
| **デメリット** | 802.1X/EAPやPKI（証明書）の運用負荷、管理フレーム攻撃への脆弱性（PMF未実装時）、Rogue Containmentによる電波法・隣接AP妨害のリスク。 |
| **対応機種** | Cisco Catalyst 9800 シリーズ WLC、Cisco AireOS WLC (8540/5520/3504)、Cisco Catalyst 9100/900 AP (CAPWAP/FlexConnect Mode)、Cisco ISE (Identity Services Engine)。 |
| **制限事項** | FlexConnect Local Switching時のアンカー mobility 設定における制限、WPA3-Enterprise 192-bit モードでの特定暗号スイート（GCMP-256）必須化、古いクライアントにおけるPMF不整合。 |
| **設計上の注意点** | ゲストトラフィックのAnchor WLCへのDMZ集約、ISEとのRADIUS CoA (Change of Authorization) 導通確保（UDP 1700/3799）、WPA3 Transition Modeにおけるクライアント互換性検証。 |

---

## 🏗 動作原理

ワイヤレスセキュリティは、CAPWAP (Control and Provisioning of Wireless Access Points) トンネルによる制御・データ分離、および 802.11i / WPA3 の 4-Way Handshake / SAE (Simultaneous Authentication of Equals) による動的暗号鍵生成を中心に構成される。

### 1. 集中型 (Centralized Mode) vs FlexConnect Mode のデータフロー
集中型アーキテクチャでは、APとWLC間で CAPWAP Control (UDP 5246) および CAPWAP Data (UDP 5247) トンネルが形成され、全ユーザー通信が WLC にカプセル化転送される。一方、FlexConnect モードでは、制御信号のみ CAPWAP Control で WLC へ送られ、データ通信は AP 直下のスイッチポートへローカルス switching される（Local Switching）。

```
[ Centralized Data Path ]
Wireless Client
   ↓ (802.11 Encrypted)
Access Point (CAPWAP Encapsulation)
   ↓ (CAPWAP Data Tunnel: UDP 5247)
WLC (Central Authentication & Policy Enforcer)
   ↓ (802.3 Ethernet)
Core / Distribution Switch / Internet

[ FlexConnect Local Switching Data Path ]
Wireless Client
   ↓ (802.11 Encrypted)
Access Point (FlexConnect AP)
   ↓ (802.3 Tagged VLAN)
Local Switch Port (Local Switching)
   ↓
Local Gateway / Internet
```

### 2. 802.1X / EAP 認証と 4-Way Handshake フロー
Enterprise 認証では、EAP (Extensible Authentication Protocol) メッセージが 802.11 (EAPOL) ─ CAPWAP ─ RADIUS (UDP 1812) とカプセル化変換されながら ISE とクライアント間を行き来する。

```
Client               AP                  WLC                  Cisco ISE
  |                   |                   |                       |
  |--- 802.11 Assoc ->|                   |                       |
  |<-- Assoc Resp ----|                   |                       |
  |                   |                   |                       |
  |<=================== EAP-Request/Identity ===================|
  |==================== EAP-Response/Identity ==================>|
  |                   |                   |                       |
  |<======== EAP-TLS / PEAP Tunnel Negotiation (TLS) ===========>|
  |                   |                   |                       |
  |==================== EAP-Response (Credentials) ==============>|
  |                   |                   |                       |
  |                   |                   |<--- Access-Accept ----|
  |                   |                   |    (MSK derived)      |
  |<=================== EAP-Success ==============================|
  |                   |                   |                       |
  |<--------------- 4-Way Handshake (EAPOL-Key) ----------------->|
  |  (Derive PTK from PMK, exchange GTK for Multicast/Broadcast)  |
  |                   |                   |                       |
  |=================== Encrypted Data Traffic ====================>|
```

### 3. WPA3 SAE (Simultaneous Authentication of Equals) 動作原理
WPA3-Personal で採用されている SAE（Dragonfly Key Exchange）は、事前共有鍵 (PSK) から Diffie-Hellman 鍵交換を実行し、セッションごとに固有の暗号鍵（PMK）を生成する。これにより、通信の盗聴ログに対する事後解読攻撃（オフライン辞書攻撃）を完全に無効化し、前方秘匿性（Forward Secrecy）を提供する。

```
Client                                                  Access Point / WLC
  |                                                              |
  |------ SAE Commit (Scalar & Element derived from Password) -->|
  |<----- SAE Commit (Scalar & Element derived from Password) ---|
  |                                                              |
  |  [ Both sides derive a shared secret & calculate PMK ]       |
  |                                                              |
  |------ SAE Confirm (Send Verification Hash) ----------------->|
  |<----- SAE Confirm (Send Verification Hash) ------------------|
  |                                                              |
  |============= Proceed to 4-Way Handshake (EAPOL) =============|
```

### 4. OWE (Opportunistic Wireless Encryption) 動作原理
OWE（RFC 8110）は、ユーザーパスワードや事前共有鍵を一切要求しないオープン型 Wi-Fi でありながら、Diffie-Hellman 鍵交換（Group 19 / 256-bit ECDH）を Association ハンドシェイク内で実行し、個々のクライアント通信を AES-CCMP/GCMP で暗号化する技術である。

---

## ⚙ 動作シーケンス

ワイヤレスクライアントが WPA3-Enterprise または CWA (Central Web Authentication) で接続する際の詳細処理シーケンスを以下に示す。

### WPA3 / 802.1X 接続シーケンス
1. **802.11 Open Authentication & Association**:
   クライアントが AP に対して Probe Request / Association Request を送信。802.11w (PMF) のサポート状況（Required / Capable）および WPA2/WPA3 ケーパビリティをハンドシェイク。
2. **EAPOL-Start & EAP Exchange**:
   AP/WLC は EAPOL-Start を受信後、EAP-Request/Identity を送出。クライアントのレスポンスを RADIUS Access-Request として ISE へ転送。
3. **EAP 認証および PMK 生成**:
   ISE とクライアント間で EAP-TLS（相互証明書検証）または PEAP/EAP-FAST 認証が完了。認証成功時、ISE は RADIUS Access-Accept とともに MSK (Master Session Key) を WLC に送信。WLC とクライアントは MSK から PMK (Pairwise Master Key) を誘導。
4. **4-Way Handshake**:
   WLC（または FlexConnect AP）とクライアント間で EAPOL-Key パケットを4回交換。
   - **Msg 1**: AP -> Client (ANonce)
   - **Msg 2**: Client -> AP (SNonce + MIC) [PMKからPTK: Pairwise Transient Key を算出]
   - **Msg 3**: AP -> Client (GTK + MIC) [GTK: Group Temporal Key を暗号化送出]
   - **Msg 4**: Client -> AP (ACK)
5. **802.11w PMF IGTK 交換**:
   PMF が有効な場合、管理フレーム用暗号鍵（IGTK: Integrity Group Temporal Key）が 4-Way Handshake 内で安全に配布される。
6. **Data Transfer**:
   AES-CCMP / GCMP によるデータフレーム暗号化が開始。

### CWA (Central Web Authentication) 動作シーケンス
1. **MAB (MAC Authentication Bypass) 認証**:
   SSID に接続したクライアントの MAC アドレスで MAB を実施。
2. **ISE の Authorization Policy 適用**:
   ISE は未認証端末と判断し、`CWA_Redirect` 属性（Redirect URL + CWA ACL）を含む RADIUS Access-Accept を返却。
3. **HTTP リダイレクト**:
   端末が Web ブラウザで HTTP/HTTPS リクエストを発行すると、WLC/AP の CWA ACL が捕獲し、ISE ゲストポータル (https://ise.example.com:8443/portal/...) にリダイレクト。
4. **ポータル認証 & RADIUS CoA**:
   ユーザーがポータルで認証成功。ISE は WLC に対して RADIUS CoA (Reauthenticate / Disconnect) を送信（UDP 1700 または 3799）。
5. **再認証とフルアクセス権与**:
   WLC はクライアントの再認証（MAB）を実施。ISE は認証済み状態として、PermitAny または動的 SGT/dACL を返却。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security ラボ試験において、Wireless セキュリティは単体設定だけでなく、**Cisco ISE との複合連携**や **Cisco Catalyst 9800 WLC (IOS-XE)** の CLI/GUI 設定として頻出する。

### 1. 試験で問われる最重要ポイント
* **WPA3 / PMF (Protected Management Frames)**:
  - WPA3-Enterprise (128-bit / 192-bit) および WPA3-Personal (SAE) の構成。
  - 802.11w PMF の要求条件（WPA3 では PMF `Required` が必須。WPA2/WPA3 Transition Mode では `Optional/Capable`）。
  - PMF SA Query Timeout や IGTK 鍵更新の設定。
* **CWA (Central Web Authentication) と ISE 連携**:
  - AAA サーバーグループ、RADIUS Accounting、CoA (Change of Authorization) の定義。
  - WLC 側の Redirect ACL の作成ルール（`deny` = リダイレクト除外通信, `permit` = リダイレクト対象 HTTP/HTTPS）。
  - CLI (Cat 9800) での `parameter-map type webauth` および `policy profile` へのバインド。
* **FlexConnect Local Switching + Local Auth**:
  - WAN 障害時における FlexConnect AP の独立動作（Local Authentication / Local EAP）。
  - FlexConnect Group への RADIUS VLAN Mapping、WLAN-VLAN Mapping 設定。
* **Rogue AP Detection & RLDP (Rogue Location Discovery Protocol)**:
  - Managed AP による Rogue 検出、RLDP による有線ネットワーク結合確認、Containment (Deauth 攻撃抑制) の自動・手動設定。
* **Anchor / Foreign Mobility Tunnel**:
  - DMZ 上の Anchor WLC と内部 Foreign WLC 間における CAPWAP/EoIP Mobility Tunnel 構築とゲスト VLAN 隔離。

### 2. よくある設定ミス・落とし穴
* **Redirect ACL の論理逆転**:
  IOS-XE (Cat 9800) の Redirect ACL では、**ISE / DNS / DHCP / Gateway への通信を `deny`** にし、それ以外の Web 通信 (`permit ip any any`) をリダイレクト対象とする。これを逆に設定すると無限リダイレクトまたは通信不能に陥る。
* **RADIUS CoA ポート不一致**:
  Cat 9800 では既定の CoA ポートが `1700` または `3799` である。`aaa server radius dynamic-author` 設定で `client <ISE-IP> server-key <KEY>` を定義し忘れると、CoA がドロップされる。
* **PMF 不整合によるクライアント切断**:
  古い IoT 端末や Legacy 端末が存在する SSID で PMF を `Required` にすると、端末が Association できない。Transition Mode (`pmf optional`) を使用する必要がある。
* **FlexConnect AP の WLAN-VLAN マッピング欠落**:
  WLAN 上で Local Switching を有効にしても、FlexConnect AP または FlexConnect Group で VLAN マッピングが正しく定義されていない場合、デフォルト VLAN (VLAN 1) にフォールバックするかパケットがドロップする。

---

## 🛠 設定方法

### Catalyst 9800 WLC (IOS-XE) CLI 完全設定例

#### 1. AAA サーバー & RADIUS Dynamic Authorization (CoA) 設定
```bash
! AAA 認証・認可・課金グループ定義
radius server ISE_NODE_1
 address ipv4 192.168.10.50 auth-port 1812 acct-port 1813
 key 0 CiscoKey123

aaa group server radius RADIUS_ISE_GRP
 server name ISE_NODE_1

aaa authentication dot1x ISE_8021X_LIST group RADIUS_ISE_GRP
aaa authorization network ISE_NET_AUTHZ group RADIUS_ISE_GRP
aaa accounting network ISE_ACCT_LIST start-stop group RADIUS_ISE_GRP

! RADIUS Dynamic Authorization (CoA) 受信ポートおよび Shared Key 定義
aaa server radius dynamic-author
 client 192.168.10.50 server-key 0 CiscoKey123
 port 1700
```

#### 2. CWA Redirect ACL および WebAuth Parameter Map 設定
```bash
! Redirect ACL (DNS, DHCP, ISE, Gateway 宛ては deny で除外)
ip access-list extended CWA_REDIRECT_ACL
 10 deny udp any any eq domain
 20 deny udp any any eq bootps
 30 deny udp any any eq bootpc
 40 deny ip any host 192.168.10.50
 50 permit tcp any any eq www
 60 permit tcp any any eq 443

! WebAuth Parameter Map 定義
parameter-map type webauth global
 type consent
 redirect for-login-only
```

#### 3. WPA3-Enterprise & PMF 対応 WLAN ポリシー構成
```bash
! WPA3-Enterprise WLAN 定義
wlan EMP_WPA3_SSID 10 EMP_WPA3_SSID
 security ft
 security wpa wpa3
 security wpa akm dot1x
 security wpa wpa3 ciphers aes-gcm256 aes-ccm128
 security wpa pmf mandatory
 no shutdown

! Policy Profile 定義
wireless profile policy EMP_POLICY_PROFILE
 aaa-override
 accounting-list ISE_ACCT_LIST
 central switching
 central authentication
 central dhcp
 vlan 200
 no shutdown

! Policy Tag へのマッピング
wireless tag policy DEFAULT_POLICY_TAG
 wlan EMP_WPA3_SSID policy EMP_POLICY_PROFILE
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| --- | --- |
| **アクティブワイヤレスクライアント一覧** | <code>show wireless client summary</code> |
| **クライアント詳細（Auth, VLAN, SGT, PMF）** | <code>show wireless client mac-address &lt;MAC&gt; detail</code> |
| **WLAN 詳細セキュリティプロファイル確認** | <code>show wlan name &lt;WLAN_NAME&gt;</code> |
| **802.11w PMF 動作状態確認** | <code>show wireless client mac-address &lt;MAC&gt; pmf</code> |
| **AAA RADIUS サーバー応答・統計確認** | <code>show aaa servers</code> |
| **FlexConnect 状態および VLAN マッピング** | <code>show wireless profile flex detailed &lt;FLEX_PROFILE&gt;</code> |
| **Rogue AP 検出要約確認** | <code>show wireless rogue ap summary</code> |
| **リアルタイム Client デバッグ・トレース** | <code>debug wireless mac &lt;MAC&gt; monitor-time 60</code> |
| **AAA / RADIUS ライブデバッグ** | <code>debug radius verbose</code> / <code>debug aaa authentication</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **WPA3-Enterprise 端末が接続不能** | 端末が PMF (802.11w) に非対応、または WPA3 192-bit モード暗号不一致。 | <code>show wlan name &lt;WLAN&gt;</code><br><code>show wireless client mac-address &lt;MAC&gt; detail</code> | PMF を `mandatory` から `optional` に変更し、Transition Mode を有効化。 |
| **CWA でポータルにリダイレクトされない** | Redirect ACL の論理ミス、または DNS 解決不能。 | <code>show ip access-lists CWA_REDIRECT_ACL</code><br><code>show wireless client mac &lt;MAC&gt; detail</code> | Redirect ACL で DNS/ISE 宛てを `deny` にし、HTTP/HTTPS を `permit` に修正。 |
| **ISE での CoA 適用失敗 (VLAN/SGT未変更)** | RADIUS Dynamic-Author 設定漏れ、または UDP ポート不一致。 | <code>show aaa servers</code><br><code>debug radius verbose</code> | `aaa server radius dynamic-author` で ISE IP と Shared Key、Port (1700/3799) を設定。 |
| **FlexConnect Local Switching 時に通信不可** | AP のスイッチポート Trunk 設定不備、または VLAN マッピング未定義。 | <code>show wireless profile flex detailed &lt;PROFILE&gt;</code> | Switch Port を Trunk 化し、Native VLAN および WLAN-VLAN Mapping を正しく設定。 |
| **Rogue AP が Containment (抑制) されない** | Rogue AP 分類ポリシーが Unclassified、または RLDP 未有効化。 | <code>show wireless rogue ap summary</code> | Rogue Rule で Malicious 定義を設定し、手動/自動 Containment を有効化。 |


## ⚠ 制限事項

### 1. WPA3 / PMF (Protected Management Frames) の制限事項
* **WPA3-Personal (SAE) Transition Mode**: WPA2/WPA3 移行モードでは PMF は `Capable` (Optional) となるが、WPA3 専用モードでは PMF `Required` が必須である。
* **WPA3 192-bit Security Mode**: 192-bit モード（CNSA 適合）では、EAP-TLS (TLS 1.2/1.3) と Suite B 暗号（GCMP-256、ECDHE-ECDSA）が必須となり、非対応クライアントは接続を拒否される。

### 2. FlexConnect Mode の制限事項
* **Central Web Authentication (CWA)**: FlexConnect Local Switching 環境での CWA は、AP ローカルで Redirect ACL が処理されるため、WLC と AP 間で ACL 名が完全に一致している必要がある。
* **Anchor/Foreign Mobility**: FlexConnect Local Switching WLAN に対する Guest Anchor トンネリングはサポートされない。ゲストトラフィックを Anchor WLC へ集約する場合は、Central Switching モードを使用する必要がある。

### 3. Rogue Containment (抑制) の法的・技術的制限
* **802.11 Deauthentication 攻撃による抑制**: 隣接組織の正常な AP に対する Containment は電波法違反や通信妨害となる危険性がある。CCIE ラボ試験では指定された Rogue AP のみに Containment を適用すること。

---

## 🔄 他技術との関連

ワイヤレスセキュリティは、ネットワークインフラ全体のセキュアアーキテクチャと密接に連携する。

* **Cisco ISE (Identity Services Engine)**:
  ワイヤレス 802.1X 認証、MAB 認証、CWA ゲストポータル、Posture 健全性チェック、BYOD オンボーディングの集中制御ポイント。
* **Cisco TrustSec / Security Group Tag (SGT)**:
  802.1X 認証成功時に ISE から WLC/AP へ SGT (Security Group Tag) を動的割り当て。AP またはインフラ側で SXP (SGT Exchange Protocol) または 802.1Q Inline Tagging を用いてポリシー（SGACL）を適用。
* **FlexConnect / SD-Access (Fabric Wireless)**:
  SD-Access Fabric 環境では、AP が Control Plane (LISP) および Data Plane (VXLAN) に統合され、ANY3 (Any-subnet, Any-location, Any-device) の一貫したセキュリティポリシー（VRF + SGT）を適用。
* **802.11w PMF & 802.11r FT (Fast Transition)**:
  ローミング時の再認証時間を短縮する Fast BSS Transition (FT) と PMF の組み合わせにより、セキュアかつシームレスな音声・リアルタイム通信を保護。

---

## 🧩 比較表

### 1. ワイヤレスセキュリティ規格の比較

| 項目 | WPA2-Enterprise | WPA3-Enterprise (128-bit) | WPA3-Enterprise (192-bit Mode) | WPA3-Personal (SAE) | OWE (Enhanced Open) |
| --- | --- | --- | --- | --- | --- |
| **認証方式** | 802.1X / EAP | 802.1X / EAP | 802.1X / EAP-TLS (Suite B) | SAE (Simultaneous Auth of Equals) | Opportunistic Encryption (DH) |
| **暗号アルゴリズム** | AES-CCMP-128 | AES-CCMP-128 / GCMP-256 | AES-GCMP-256 | AES-CCMP-128 | AES-CCMP-128 |
| **PMF (802.11w)** | オプション (Capable) | **必須 (Mandatory)** | **必須 (Mandatory)** | **必須 (Mandatory)** | **必須 (Mandatory)** |
| **辞書攻撃耐性** | 低い (PSK時) | 高い | 極めて高い | **極めて高い (Diffie-Hellman)** | N/A (パスワード不要) |
| **主な用途** | 従来の企業LAN | 現代の企業LAN | 政府・金融高セキュリティ | 小規模拠点・IoT | パブリックゲストWi-Fi |

### 2. ゲストアクセス方式の比較

| 項目 | Local WebAuth | Central Web Authentication (CWA) | Foreign / Anchor Mobility |
| --- | --- | --- | --- |
| **認証制御** | WLC 内部ローカルWebポータル | Cisco ISE 外部ポータル | DMZ 上の Anchor WLC + ISE |
| **ポリシー柔軟性** | 限定的（固定VLAN/ローカルID） | **極めて高い（動的VLAN, SGT, dACL, 期限管理）** | 高い（物理/論理ネットワーク完全隔離） |
| **CoA (Re-auth) 対応** | 非対応 | **必須対応 (UDP 1700/3799)** | 対応 |
| **推奨用途** | 簡易的な拠点ゲスト | 統合エンタープライズゲスト | DMZ分離が必要な高度セキュリティ環境 |

---

## 💡 ベストプラクティス

1. **WPA3-Enterprise への段階的移行**:
   WPA2/WPA3 Transition Mode を活用し、古い端末の互換性を維持しながら段階的に WPA3 および 802.11w PMF を強制化する。
2. **CWA における Redirect ACL の最適化**:
   Redirect ACL には不必要な `permit` ルールを含めず、DNS (UDP 53), DHCP (UDP 67/68), ISE Node IP のみを `deny`（除外）し、残りの HTTP/HTTPS を最小限で捕獲する。
3. **管理フレーム保護 (PMF / Infrastructure MFP) の全系適用**:
   Deauthentication 攻撃や Rogue AP スプーフィングを防ぐため、全 WLAN で 802.11w PMF を有効化する。
4. **ゲストトラフィックの Anchor WLC 集約**:
   ゲストユーザーの通信を有線内部ネットワークから物理的・論理的に遮断するため、DMZ ゾーンに配置した Guest Anchor WLC へ CAPWAP/Mobility トンネルで直接カプセル化転送する。


## 📝 ラボ学習・設定サンプル例

実際の CCIE Security ラボ試験レベルを意識した、高度なワイヤレスセキュリティ設定・演習シナリオ 10 選を以下に示す。

---

### シナリオ 1: WPA3-Enterprise 802.1X と PMF (802.11w) の厳格設定

#### 問題
Catalyst 9800 WLC 上で、企業幹部用 SSID `CORP_SECURE` を構成し、802.1X 認証および WPA3-Enterprise 暗号化を適用せよ。

#### 要件
1. SSID 名: `CORP_SECURE`, WLAN ID: `15`
2. L2 セキュリティ: WPA3-Enterprise (AKM: `dot1x`), 暗号アルゴリズム: `aes-ccm128`
3. 802.11w Protected Management Frames (PMF): `Mandatory` (必須)
4. RADIUS サーバーグループ `ISE_GRP` (192.168.10.50, Key: `CiscoKey123`) を使用
5. クライアント通信は VLAN `200` にアサイン

#### 設定例 (Cat 9800 CLI)
```bash
! RADIUS サーバーおよびグループ設定
radius server ISE_RADIUS
 address ipv4 192.168.10.50 auth-port 1812 acct-port 1813
 key 0 CiscoKey123

aaa group server radius ISE_GRP
 server name ISE_RADIUS

aaa authentication dot1x CORP_DOT1X_LIST group ISE_GRP

! WLAN 設定
wlan CORP_SECURE 15 CORP_SECURE
 security wpa wpa3
 security wpa akm dot1x
 security wpa wpa3 ciphers aes-ccm128
 security wpa pmf mandatory
 no shutdown

! Policy Profile 設定
wireless profile policy CORP_POLICY
 aaa-override
 authentication-list CORP_DOT1X_LIST
 vlan 200
 central switching
 central authentication
 no shutdown

! Policy Tag へのバインド
wireless tag policy DEFAULT_POLICY_TAG
 wlan CORP_SECURE policy CORP_POLICY
```

#### 検証方法
```bash
# WLAN のセキュリティ設定および PMF 設定の確認
show wlan name CORP_SECURE

# 接続クライアントの PMF ネゴシエーション状態の確認
show wireless client mac-address aaaa.bbbb.cccc pmf
```

---

### シナリオ 2: WPA2/WPA3-Personal Transition Mode (SAE) の構成

#### 問題
レガシー端末と最新端末が混在する拠点向けに、WPA2-PSK と WPA3-Personal (SAE) が共存する Transition Mode SSID `GUEST_SAE` を構成せよ。

#### 要件
1. SSID: `GUEST_SAE`, WLAN ID: `16`
2. WPA2-PSK (AKM: `psk`) および WPA3-Personal (AKM: `sae`) の双方を許可
3. プレシェアードキー: `WirelessPass123!`
4. 802.11w PMF: `optional` (Capable)

#### 設定例 (Cat 9800 CLI)
```bash
wlan GUEST_SAE 16 GUEST_SAE
 security wpa wpa2 ciphers aes-ccm128
 security wpa wpa3 ciphers aes-ccm128
 security wpa akm psk set-key ascii 0 WirelessPass123!
 security wpa akm sae
 security wpa pmf optional
 no shutdown
```

#### 検証方法
```bash
show wlan name GUEST_SAE
# AKM に psk と sae が表示され、PMF が Optional であることを確認
```

---

### シナリオ 3: OWE (Opportunistic Wireless Encryption) による保護されたオープンWi-Fi

#### 問題
パスワード入力不要でありながら、空中ログを盗聴不可能な暗号化オープン SSID `PUBLIC_OWE` を構成せよ。

#### 要件
1. SSID: `PUBLIC_OWE`, WLAN ID: `17`
2. セキュリティ: OWE (AKM: `owe`)
3. PMF (802.11w): `mandatory` (OWE では規格上必須)

#### 設定例 (Cat 9800 CLI)
```bash
wlan PUBLIC_OWE 17 PUBLIC_OWE
 security wpa wpa3
 security wpa akm owe
 security wpa wpa3 ciphers aes-ccm128
 security wpa pmf mandatory
 no shutdown
```

#### 検証方法
```bash
show wlan name PUBLIC_OWE
# AKM が owe、PMF が mandatory となっていることを確認
```

---

### シナリオ 4: Central Web Authentication (CWA) と Cisco ISE 連携

#### 問題
Cisco ISE ポータルへ自動リダイレクトする Central Web Authentication (CWA) SSID `GUEST_CWA` を構成せよ。

#### 要件
1. L2 認証: MAB (MAC Authentication Bypass)
2. RADIUS Dynamic Authorization (CoA) を有効化 (Port: `1700`, Key: `CiscoKey123`)
3. リダイレクト ACL: `CWA_REDIRECT` (DNS, DHCP, ISE 192.168.10.50 は除外)

#### 設定例 (Cat 9800 CLI)
```bash
! AAA Dynamic Authorization (CoA)
aaa server radius dynamic-author
 client 192.168.10.50 server-key 0 CiscoKey123
 port 1700

! Redirect ACL
ip access-list extended CWA_REDIRECT
 10 deny udp any any eq domain
 20 deny udp any any eq bootps
 30 deny ip any host 192.168.10.50
 40 permit tcp any any eq www
 50 permit tcp any any eq 443

! Parameter Map
parameter-map type webauth global
 type consent
 redirect for-login-only

! WLAN & Policy Profile
wlan GUEST_CWA 20 GUEST_CWA
 mac-filtering ISE_GRP
 no security wpa
 no shutdown

wireless profile policy CWA_POLICY
 aaa-override
 accounting-list ISE_ACCT_LIST
 central switching
 central authentication
 central dhcp
 vlan 300
 no shutdown
```

#### 検証方法
```bash
show wireless client mac-address aaaa.bbbb.cccc detail
# URL-Redirect-ACL : CWA_REDIRECT
# URL-Redirect     : https://ise01.example.com:8443/portal/...
```

---

### シナリオ 5: Foreign / Anchor Mobility トンネルによるゲストトラフィック隔離

#### 問題
内部 WLC (Foreign) 上のゲスト通信を、DMZ 上の Guest Anchor WLC へ CAPWAP Mobility トンネル経由で転送せよ。

#### 要件
1. Foreign WLC IP: `10.1.1.10`, Anchor WLC IP: `172.16.10.10`
2. Mobility Group 名: `MOB_GRP_1`
3. Mobility Keepalive: `3` 秒, Retry: `3` 回

#### 設定例 (Foreign WLC CLI)
```bash
wireless mobility group name MOB_GRP_1
wireless mobility group member mac aaaa.0000.0001 ip 172.16.10.10 public-ip 172.16.10.10 group MOB_GRP_1

! WLAN 上で Anchor Peer を指定
wlan GUEST_ANCHOR_SSID 25 GUEST_ANCHOR_SSID
 mobility anchor 172.16.10.10
 no shutdown
```

#### 検証方法
```bash
show wireless mobility summary
show wireless mobility peer brief
```

---

### シナリオ 6: FlexConnect Local Switching & Local Authentication

#### 問題
リモート拠点の FlexConnect AP において、WAN 障害時にも拠点の RADIUS サーバーまたは AP 内で 802.1X 認証が完結し、データが直下スイッチへ Local Switching される構成を作出せよ。

#### 要件
1. Flex Profile 名: `BRANCH_FLEX_PROFILE`
2. Native VLAN: `10`, WLAN 30 ──► Local VLAN `30` へマッピング
3. FlexConnect Local Switching 及び Local Authentication を有効化

#### 設定例 (Cat 9800 CLI)
```bash
wireless profile flex BRANCH_FLEX_PROFILE
 native-vlan-id 10
 vlan-name LOCAL_GUEST
  vlan-id 30
 wlan-vlan-mapping 30 vlan-id 30
 local-authentication-radius-server ISE_LOCAL
  address ipv4 10.20.1.50 auth-port 1812 acct-port 1813 key CiscoKey123
```

#### 検証方法
```bash
show wireless profile flex detailed BRANCH_FLEX_PROFILE
```

---

### シナリオ 7: Wireless Management Frame Protection (Client / Infrastructure MFP)

#### 問題
偽装 AP による Deauthentication 攻撃を防ぐため、WLC 全体で Infrastructure MFP を有効化し、対象 WLAN で Client MFP を要求せよ。

#### 要件
1. Infrastructure MFP: AP 間で管理フレーム検証を有効化
2. WLAN `SECURE_WLAN` 上で Client MFP (802.11w) を `mandatory` に設定

#### 設定例 (Cat 9800 CLI)
```bash
! Global Infrastructure MFP
ap mfp

! WLAN Client MFP
wlan SECURE_WLAN 35 SECURE_WLAN
 security wpa pmf mandatory
 no shutdown
```

#### 検証方法
```bash
show ap mfp summary
show wlan name SECURE_WLAN
```

---

### シナリオ 8: Rogue AP 検出・RLDP・自動抑制 (Containment)

#### 問題
有線ネットワークに不正接続された Rogue AP を検知し、有線接続を判別する RLDP (Rogue Location Discovery Protocol) を実行後、自動抑制せよ。

#### 要件
1. Rogue AP の脅威判定ルール: SSID 名が `Corporate_Fake` の場合、Malicious に分類
2. Malicious Rogue AP に対して自動 Containment (1 Radio) を適用

#### 設定例 (Cat 9800 CLI)
```bash
wireless rogue policy
 rogue-rule R_RULE_1
  match ssid Corporate_Fake
  classify malicious
  containment 1
```

#### 検証方法
```bash
show wireless rogue ap summary
show wireless rogue ap detail aaaa.bbbb.cccc
```

---

### シナリオ 9: Identity PSK (iPSK) と Cisco ISE 動的 VLAN/dACL 割り当て

#### 問題
同一の PSK 用 SSID `IOT_WLAN` において、接続する IoT 機器の MAC アドレスごとに異なる PSK を適用し、ISE から動的に VLAN を変更せよ。

#### 要件
1. WLAN ID: `40`, SSID: `IOT_WLAN`
2. MAC Filtering を ISE_GRP に対して有効化
3. ISE より VLAN オーバーライドおよび dACL を適用

#### 設定例 (Cat 9800 CLI)
```bash
wlan IOT_WLAN 40 IOT_WLAN
 mac-filtering ISE_GRP
 security wpa wpa2 ciphers aes-ccm128
 security wpa akm psk
 no shutdown

wireless profile policy IOT_POLICY
 aaa-override
 central switching
 central authentication
 vlan 1
 no shutdown
```

#### 検証方法
```bash
show wireless client mac-address aaaa.bbbb.cccc detail
# AAA-Override により正しく特定の VLAN に再割り当てされていることを確認
```

---

### シナリオ 10: Catalyst 9800 WLC における Dynamic SGT 割り当てと SXP エクスポート

#### 問題
無線 802.1X 認証成功時に ISE から動的に返却された Security Group Tag (SGT) を WLC 上で保持し、SXP プロトコルでコアスイッチにピアリング送出せよ。

#### 要件
1. SXP Connection: ピア IP `10.1.1.254`, Shared Secret: `SxpKey123`
2. SXP Mode: `Speaker`
3. Policy Profile で SGT Override (`aaa-override`) を許可

#### 設定例 (Cat 9800 CLI)
```bash
! SXP 有効化
cts sxp enable
cts sxp default source-ip 10.1.1.10
cts sxp connection peer 10.1.1.254 password default 0 SxpKey123 mode local speaker holdtime 120 180

! Policy Profile 設定
wireless profile policy CORP_POLICY
 aaa-override
 no shutdown
```

#### 検証方法
```bash
show cts sxp connections
show cts sxp sgt-map
```


## ❓ 想定試験問題

CCIE Security ラボ試験・記述問題（Design/Troubleshooting）を意識した想定問題 7 問と詳細解説を以下に示す。

---

### 問題 1 (コンフィグ読解)
以下の Catalyst 9800 WLC のコンフィグレーションを読み、クライアントが HTTP にアクセスした際の動作を説明せよ。また、設定上の重大な不備が存在する場合は指摘せよ。

```bash
ip access-list extended REDIRECT_ACL
 10 permit udp any any eq domain
 20 permit udp any any eq bootps
 30 permit ip any host 192.168.10.50
 40 permit tcp any any eq www
 50 permit tcp any any eq 443
```

#### 正解・解説
**動作と問題点**:
本 ACL 設定は重大な設計ミスである。IOS-XE の Redirect ACL では、`permit` された通信が Web リダイレクト（捕獲）の対象となり、`deny` された通信がリダイレクトをバイパスして直接通信可能となる。
本設定では DNS (UDP 53), DHCP (UDP 67/68), ISE (192.168.10.50) が `permit` されているため、DNS や DHCP の応答パケットまで Web ポータルへリダイレクトされ、クライアントは IP アドレスの取得や FQDN 解決ができず、Web リダイレクト自体が正常に動作しない。

**正しい修正案**:
```bash
ip access-list extended REDIRECT_ACL
 10 deny udp any any eq domain
 20 deny udp any any eq bootps
 30 deny ip any host 192.168.10.50
 40 permit tcp any any eq www
 50 permit tcp any any eq 443
```



---

### 問題 2 (トラブルシュート)
新設した WPA3-Enterprise SSID において、一部の最新端末は問題なく接続できるが、古いノート PC や IoT 端末が一切接続を試行できず Association エラーとなる。WLAN 設定は以下の通りである。原因と対処方法を述べよ。

```bash
wlan CORP_WPA3 10 CORP_WPA3
 security wpa wpa3
 security wpa akm dot1x
 security wpa pmf mandatory
 no shutdown
```

#### 正解・解説
**原因**:
`security wpa pmf mandatory` により 802.11w PMF が必須に設定されているため、PMF 未対応の古い端末や IoT クライアントは Association Request 段階で接続を拒否される。

**対処方法**:
PMF を `optional` (Capable) に設定し、WPA2/WPA3 Transition Mode として動作させる。
```bash
wlan CORP_WPA3 10 CORP_WPA3
 security wpa wpa2 ciphers aes-ccm128
 security wpa wpa3 ciphers aes-ccm128
 security wpa akm dot1x
 security wpa pmf optional
 no shutdown
```

---

### 問題 3 (Design)
リモート拠点（FlexConnect 構成）において、WAN 回線切断時にも拠点内の無線端末がローカルスイッチポート経由で通信を継続し、かつ新規接続端末の認証を拠点内の RADIUS サーバーで処理するためのベストプラクティス構成要素を 3 点挙げよ。

#### 正解・解説
1. **FlexConnect Local Switching の有効化**:
   Policy Profile において `central switching` を無効化（`no central switching`）し、データ通信を AP 直下のスイッチへローカルブリッジングする。
2. **FlexConnect Local Authentication の構成**:
   Flex Profile 内で `local-authentication-radius-server` を定義し、拠点ローカルの RADIUS サーバーを指定する。
3. **FlexConnect AP VLAN-WLAN マッピングの設定**:
   AP または Flex Profile 上で各 WLAN に割り当てるローカル VLAN ID（`wlan-vlan-mapping`）を明示的に事前定義する。

---

### 問題 4 (実装・CLI)
DMZ 内に配置した Anchor WLC (`172.16.1.100`) と内部ネットワークの Foreign WLC (`10.1.1.10`) 間で、ゲスト用 SSID `GUEST_NET` のトラフィックをカプセル化転送するための Foreign WLC 側の Mobility グループおよび WLAN 設定を記述せよ。

#### 正解・解説
**Foreign WLC 側設定**:
```bash
! Mobility グループと Anchor Peer の定義
wireless mobility group name GUEST_MOB_GRP
wireless mobility group member mac aaaa.1111.2222 ip 172.16.1.100 public-ip 172.16.1.100 group GUEST_MOB_GRP

! WLAN で Anchor Peer を指定
wlan GUEST_NET 50 GUEST_NET
 mobility anchor 172.16.1.100
 no shutdown
```



---

### 問題 5 (コンフィグ読解)
以下の `show wireless client mac-address aaaa.bbbb.cccc detail` の出力結果から、このクライアントの現在の認証状態および適用されているセキュリティポリシーを分析せよ。

```text
Client MAC Address: aaaa.bbbb.cccc
Client State: Associated
Policy Profile: GUEST_POLICY
VLAN: 99
Security Group Tag: 0015-Guest
URL-Redirect-ACL: CWA_REDIRECT
URL-Redirect: https://ise01.example.com:8443/portal
```

#### 正解・解説
* **分析結果**:
  1. クライアントは現在 **CWA (Central Web Authentication) のリダイレクト待機状態**（未 Web 認証状態）である。
  2. L2/MAB 認証は通過しており、ISE から初期 SGT `15` (Guest) および VLAN `99` が割り当てられている。
  3. Web ブラウザで HTTP 通信を発生させると、`CWA_REDIRECT` ACL により捕捉され、ISE のゲストポータルへ誘導される。Web 認証完了後に RADIUS CoA が発行され、フルアクセス用ポリシーへ遷移する。

---

### 問題 6 (トラブルシュート)
ISE 上で Web 認証を完了したゲストユーザーのポリシーが変更されず、WLC 側で動的 VLAN や SGT の更新が行われない。`show aaa log` では RADIUS CoA パケットを受信したログが記録されていない。確認すべき WLC コンフィグと対処手順を述べよ。

#### 正解・解説
**原因**:
WLC 上で RADIUS Dynamic Authorization (CoA) 受信機能 (`aaa server radius dynamic-author`) が未設定であるか、Shared Key または UDP 受信ポート番号（既定: `1700` または `3799`）が ISE 側の送信設定と不一致である。

**確認・対処コマンド**:
```bash
! CoA 設定の確認
show aaa server radius dynamic-author

! 正しい設定の追加
aaa server radius dynamic-author
 client 192.168.10.50 server-key 0 CiscoKey123
 port 1700
```

---

### 問題 7 (実装・トラブルシュート)
Rogue AP に対する RLDP (Rogue Location Discovery Protocol) テストを実行したところ、「RLDP Failed - No Response from Wired Network」と表示された。RLDP が有線ネットワーク結合を判別する技術的仕組みと、失敗する主な原因を述べよ。

#### 正解・解説
**動作原理**:
RLDP を実行すると、WLC は Managed AP の 802.11 Radio を使用して疑似クライアントとして Rogue AP に接続を試みる。接続後、Managed AP は Rogue AP 経由で有線ネットワークに向けて特殊な UDP テストパケットを送出する。WLC が有線側インターフェイスでこのパケットを直接受信できた場合、「Rogue AP は内部有線ネットワークに物理接続されている（Malicious Rogue）」と判定する。

**失敗の主な原因**:
1. Rogue AP 上で PSK / 802.1X 認証が有効化されており、Managed AP が Rogue AP に Association できない。
2. Rogue AP 直下のスイッチポートで DHCP または IP 通信が疎通しない。
3. 有線側スイッチポートの ACL や Firewall で RLDP テストパケットがドロップされた。

---

## 🔗 参考リソース

Cisco 公式ドキュメント、Design Guide (CVD)、および Cisco Live 資料一覧。

* Cisco Catalyst 9800 Series Wireless Controller Software Configuration Guide (Cisco Official Documentation)
* Cisco Catalyst 9800 Command Reference Guide (Cisco Official Documentation)
* Cisco ISE Central Web Authentication (CWA) on Catalyst 9800 WLC Configuration Example (Cisco Technical Notes)
* WPA3 Deployment Guide - Cisco Enterprise Wireless Architecture (Cisco TechNotes)
* Cisco FlexConnect Deployment Guide (Cisco Deployment Guides)
* Cisco Live BRKSEC-3032 - Advanced Wireless Security Architecture (Cisco Live On-Demand)
* Cisco Live BRKCRS-2811 - Cisco Catalyst 9800 Wireless Controller Deployment & Best Practices (Cisco Live On-Demand)
* Cisco Validated Design (CVD) - Secure Campus Wireless Network Design (Cisco Design Zone)

---

## 📝 補足（Notes）

### 802.11w Protected Management Frames (PMF) の枠組み
* Action Frames, Disassociation, Deauthentication の各管理フレームを AES-128-CMAC で完全性保護。
* スプーフィングされた Deauth パケット受信時、PMF 有効端末は直ちに切断せず、SA Query プロシージャを実行して接続を検証する。

### WPA3 192-bit Security Mode (CNSA Suite)
* AKM: 802.1X
* Cipher: GCMP-256 (Data) + GCMP-256 (PMF IGTK)
* EAP Method: EAP-TLS v1.2/1.3 with ECDHE-ECDSA


