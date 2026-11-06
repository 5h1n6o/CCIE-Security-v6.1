---
layout: default
title: 4.2-Network-access-AAA
nav_order: 2
parent: 4.0-Identity-Management
---

# 4.2 Cisco switches and Cisco Wireless LAN Controllers for network access AAA with Cisco ISE

## 📘 概要

ネットワークアクセス制御におけるAAA（Authentication, Authorization, Accounting）は、エンタープライズインフラセキュリティの中核をなす技術です。Cisco ISE（Identity Services Engine）を中心に、Cisco Catalyst スイッチ（IOS-XE）および Cisco Wireless LAN Controller（AireOS / Catalyst 9800 シリーズ WLC）をネットワークアクセス学習・制御ポイント（Network Access Device: NAD）として統合することにより、有線・無線の両環境において一貫した識別・認証・認可・ポスチャ検証・動的ポリシー適用を実現します。

本学習メモは、CCIE Security v6.1 ラボ試験および実務設計において、Cisco スイッチおよび WLC と Cisco ISE 間の RADIUS / TACACS+ / RADIUS CoA（Change of Authorization）連携、802.1X、MAB（MAC Authentication Bypass）、WebAuth（Web Authentication）、C3PL（Cisco Common Classification Policy Language / IBNS 2.0）、FlexConnect、Central/Local Authentication、および各種動的制御属性（Dynamic VLAN, DACL, SGT, CWA Redirection）の実装・検証・トラブルシューティングを網羅的に解説します。

---

## 🔑 要点

Cisco スイッチおよび WLC と Cisco ISE の AAA 連携における主要要点を下表に示します。

| 項目 | 内容 |
| --- | --- |
| **主要プロトコル** | RADIUS (UDP 1812/1813, 1645/1646), RADIUS CoA (UDP 1700 / 3799), TACACS+ (TCP 49), EAP (EAP-TLS, PEAP, EAP-FAST, EAP-TTLS) |
| **対応機器** | Cisco Catalyst 9000 シリーズ (IOS-XE), Catalyst 3850/3650, AireOS WLC (5520/8540, 3504), Catalyst 9800 シリーズ WLC |
| **認証方式** | IEEE 802.1X（証明書・ID/Pass）, MAB（非802.1X機器のMACベース認証）, WebAuth / CWA（Central Web Authentication） |
| **認可制御方式** | Dynamic VLAN, DACL (Dynamic Access List), SGT (Security Group Tag), QoS / Bandwidth Rate-Limit, Guest Redirect URL |
| **アーキテクチャ** | IBNS 1.0 (Legacy 802.1X commands) vs IBNS 2.0 / C3PL (Control-Policy based) |
| **ワイヤレス認証モデル** | Central Authentication / Central Switching, Central Authentication / Local Switching (FlexConnect), Local Auth / Local Switching |
| **プロファイリングデータ収集** | Device Sensor (DHCP Options, CDP, LLDP, HTTP-User-Agent) を RADIUS Accounting / EAPoL パケット経由で ISE に伝送 |
| **制限事項・注意点** | CoA Re-auth vs Port Bounce / Shutdown の使い分け、WLC における RADIUS MAC Filtering と 802.1X の同時構成順序、FlexConnect ACL のローカル定義要件 |

---

## 🏗 動作原理

### 1. 有線アクセス制御（IBNS 2.0 / C3PL ベース 802.1X / MAB / CWA）

有線スイッチポートにおいて端末が接続された際、スイッチは 802.1X サプリカントからの EAPoL-Start を待機しつつ、構成されたタイマーに従って 802.1X 認証を試行します。802.1X が応答しない場合は MAB（MAC アドレスをユーザー名・パスワードとして RADIUS 送信）へフォールバックし、未登録端末に対しては CWA（Web リダイレクト認可）を適用します。

```text
End Host (Client)            Switch (NAD / IOS-XE)                 Cisco ISE (RADIUS Server)
   │                                 │                                        │
   ├─────── EAPoL-Start ────────────>│                                        │
   │<────── EAP-Request/Identity ────┤                                        │
   ├─────── EAP-Response/Identity ──>│                                        │
   │                                 ├────── RADIUS Access-Request ──────────>│
   │                                 │       (User-Name, Calling-Station-ID)  │
   │                                 │<───── RADIUS Access-Challenge ─────────┤
   │<────── EAP-Request (TLS/PEAP) ──┤                                        │
   │─────── EAP-Response ───────────>│                                        │
   │                                 ├────── RADIUS Access-Request ──────────>│
   │                                 │<───── RADIUS Access-Accept ────────────┤
   │                                 │       (dACL, SGT, Dynamic VLAN, CoA)   │
   │<────── EAP-Success ─────────────┤                                        │
   │                                 │                                        │
   │ [データ通信開始]                 │                                        │
   ├═════════════════════════════════╪═══════════════════════════════════════>│
```

### 2. 無線アクセス制御（WLC 802.1X / CWA / Central vs Local Switching）

WLC 環境では、AP が受信したクライアントの認証トラフィック（EAPoL や 802.11 認証）を WLC が CAPWAP 制御プレーン経由で集約し、RADIUS プロトコルで ISE に中継します。

```text
Wireless Client               Access Point (AP)              WLC (C9800 / AireOS)           Cisco ISE
   │                                 │                                │                         │
   ├────── 802.11 Assoc / EAPoL ────>│                                │                         │
   │                                 ├────── CAPWAP Control ─────────>│                         │
   │                                 │                                ├─ RADIUS Access-Request >│
   │                                 │                                │<─ RADIUS Access-Accept ─┤
   │                                 │                                │  (SGT, VLAN-ID, Redir)  │
   │<───── 802.11 EAP-Success ───────┼<───── CAPWAP Control ──────────┤                         │
   │                                 │                                │                         │
   │ [Central Switching: データパケットを CAPWAP データで WLC へカプセル化中継]                    │
   │ [Local Switching:   AP がローカルスイッチ VLAN に直結ブリッジング]                             │
```

---

## ⚙ 動作シーケンス

### 1. 802.1X / MAB / CWA フォールバック処理シーケンス

1. **リンクアップ & セッション開始**: スイッチポートが UP すると、ポートセキュリティ / C3PL テンプレートに従いセッション（MAC アドレス単位）が生成されます。
2. **EAPoL 認証要求**: スイッチから `EAP-Request/Identity` をマルチキャスト（`01:80:c2:00:00:03`）送信します。
3. **802.1X タイムアウト & MAB フォールバック**: 端末が EAPoL に応答しない場合、タイマー切れ（`tx-period`）後にスイッチは端末のソース MAC アドレスを捉え、`Access-Request`（User-Name = MAC, Password = MAC）を ISE に送信します。
4. **MAB 認可 & CWA 適用**: ISE はプロファイリングデータベースを参照し、未知の端末である場合は「CWA 認可ポリシー」を返答します。
   - レスポンス属性: `ACS:Cisco-AVPair = url-redirect=https://ise.example.com:8443/portal/...`
   - `ACS:Cisco-AVPair = url-redirect-acl=REDIRECT_ACL`
5. **HTTP リダイレクト処理**: 端末が Web ブラウザで HTTP 通信を開始すると、スイッチのデータプレーンは `REDIRECT_ACL` にマッチする TCP 80 通信をキャプチャし、内部の HTTP 代理サーバー機能によって ISE ゲストポータル URL へ HTTP 302 Redirect を返します。
6. **Web ポータル認証 & CoA 実行**: ユーザーが ISE ゲストポータルでログインを完了すると、ISE はスイッチに対して RADIUS CoA（CoA-Request: Type 43 / Re-authenticate）を発行します。
7. **Re-authentication & 最終認可適用**: スイッチはセッションを再認証し、ISE から最終的な認可結果（Full Access DACL, SGT 4, VLAN 10 など）を受信してデータプレーンに適用します。

---


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security ラボ試験において、Cisco ISE と Switch / WLC の AAA 連携は **Section 4 (Identity Management & Access Control)** の配点の大半を占める最重要項目です。

### 1. Blueprintで重要なポイント
* **IBNS 2.0 / C3PL (Cisco Common Classification Policy Language)**: 従来の `dot1x pae authenticator` などの Legacy コマンドではなく、`class-map type control subscriber` や `policy-map type control subscriber` を用いたセッション状態遷移プログラミングが課されます。
* **RADIUS CoA (Change of Authorization)**: DACL 変更、SGT 変更、WebAuth ポスチャ完了後の属性動的変更に伴う CoA （Port Bounce / Re-auth）の正しい動作確認。
* **Device Sensor Integration**: スイッチ側で `device-sensor filter-list` および `device-sensor notify group-listening` を構成し、ISE 側での正確なデバイスプロファイリング（IP Phone, IP Camera, BYOD）を実現する設定。
* **WLC AAA & FlexConnect**: Central Authentication / Central Switching と FlexConnect Local Switching における RADIUS リダイレクトおよび ACL (FlexConnect ACL vs WLC IPv4 ACL) の違いの理解。
* **RADIUS Key Wrap & TACACS+ Command Authorization**: RADIUS 通信の暗号化（RFC 3579 Key Wrap）および管理者のコマンド認可設定。

### 2. ラボ試験で設定させられそうな内容・シナリオ
1. **Multi-Domain / Multi-Auth の切り替え**: 音声 VLAN（IP Phone）とデータ VLAN（PC）が同小ポートに接続される環境での `host-mode multi-domain` / `multi-auth` の厳密な設定。
2. **Open Authentication (Low-Impact Mode)**: 認証前であっても基本通信（DHCP, DNS, PXE）を許可し、認証完了後に DACL で通信を絞る設定。
3. **CWA (Central Web Authentication) & Guest Access**: スイッチおよび WLC でのリダイレクト ACL の作成と ISE 側認可ポリシーの紐付け。
4. **WLC 9800 AAA Advanced Options**: `aaa accounting identity default start-stop group RADIUS-GROUP` および `radius-server attribute 31 send mac-in-dn` 等の無線固有パラメータ構成。

### 3. よくある設定ミス・トラップ
* **RADIUS CoA ポート不一致**: スイッチはデフォルトで UDP 1700 または 3799 を使用。ISE 側の NAD（Network Access Device）設定で CoA Port が一致していないと CoA がサイレントドロップする。
* **リダイレクト ACL の方向誤り**: スイッチのリダイレクト ACL では、**ISE への通信および DNS/DHCP 通信を `deny`（リダイレクト除外）** し、**一般的な HTTP/HTTPS 通信を `permit`（リダイレクト対象）** に指定しなければならない（通常の Permit/Deny と意味が逆になる点に注意）。
* **WLC FlexConnect ACL の未定義**: FlexConnect Local Switching を使用する場合、リダイレクト ACL や DACL 名が AP ローカル側で保持・定義されていないと、クライアントのトラフィックがブロックされる。
* **RADIUS Accounting の未設定**: `aaa accounting network default start-stop group RADIUS-GROUP` が設定されていないと、ISE が IP アドレスと MAC アドレスのバインディング（Session-ID）を把握できず、CoA や Profiling が正常機能しない。

---

## 🛠 設定方法

### 1. Cisco Catalyst Switch (IOS-XE) - IBNS 2.0 / C3PL 完全設定例

```text
! --- 1. Global AAA & RADIUS Configuration ---
aaa new-model
aaa authentication dot1x default group RADIUS-ISE-GROUP
aaa authorization network default group RADIUS-ISE-GROUP
aaa accounting update periodic 5
aaa accounting network default start-stop group RADIUS-ISE-GROUP

! RADIUS Server Definition (RFC 3579 Key Wrap Supported)
radius server ISE-PRIMARY
 address ipv4 192.168.10.100 auth-port 1812 acct-port 1813
 key 7 0822455D0A16
radius server ISE-SECONDARY
 address ipv4 192.168.10.101 auth-port 1812 acct-port 1813
 key 7 0822455D0A16

aaa group server radius RADIUS-ISE-GROUP
 server name ISE-PRIMARY
 server name ISE-SECONDARY
 ip radius source-interface GigabitEthernet1/0/1

! Enable Dynamic Authorization (CoA)
aaa server radius dynamic-author
 client 192.168.10.100 server-key Cisco123!
 client 192.168.10.101 server-key Cisco123!
 port 1700
 auth-type any

! Global Dot1x & CoA System Control
dot1x system-auth-control
radius-server attribute 6 on-for-auth
radius-server attribute 8 include-in-access-req
radius-server attribute 25 access-request include
radius-server attribute 31 send mac-format ietf upper-case

! --- 2. Redirect ACL Configuration ---
! NOTE: 'deny' means DO NOT REDIRECT, 'permit' means REDIRECT TO ISE
ip access-list extended ACL_CWA_REDIRECT
 deny   udp any any eq domain
 deny   udp any any eq bootps
 deny   udp any any eq bootpc
 deny   ip any host 192.168.10.100
 deny   ip any host 192.168.10.101
 permit tcp any any eq www
 permit tcp any any eq 443

! --- 3. IBNS 2.0 / C3PL Control Class & Policy Maps ---
class-map type control subscriber match-all IN_CRITICAL_AUTH
 match authorization-status unauthorized
 match result-type method dot1x agent-not-found

class-map type control subscriber match-all DOT1X_FAILED
 match method dot1x priority 10
 match result-type method dot1x failed

class-map type control subscriber match-all MAB_FAILED
 match method mab priority 20
 match result-type method mab failed

policy-map type control subscriber DOT1X_MAB_POLICY
 event session-started action number 10
  execute factory-control-plan
 event authentication-failure action number 10
  check-first-authorization
 event authentication-failure class DOT1X_FAILED action number 20
  authenticate using mab priority 20
 event authentication-failure class MAB_FAILED action number 30
  authorize
  set-variable $AUTHZ_PROFILE_NAME = 'CRITICAL_AUTH_PROFILE'
 event agent-found action number 10
  authenticate using dot1x priority 10

! --- 4. Service Templates ---
template DEFAULT_LINK_TEMPLATE
 dot1x max-req 3
 dot1x max-reauth-req 2
 dot1x timeout tx-period 7
 mab
 access-session port-control auto
 access-session host-mode multi-auth
 service-policy type control subscriber DOT1X_MAB_POLICY

! --- 5. Interface Application & Device Sensor ---
interface GigabitEthernet1/0/10
 description Access_Port_with_IBNS2.0
 switchport mode access
 switchport access vlan 10
 source template DEFAULT_LINK_TEMPLATE
 spanning-tree portfast

! Device Sensor Profiling Configuration
device-sensor filter-spec dhcp include list DHCP_LIST
device-sensor filter-spec cdp include list CDP_LIST
device-sensor filter-spec lldp include list LLDP_LIST
device-sensor notify group-listening
```

---

### 2. Cisco Catalyst 9800 WLC (IOS-XE Wireless) - AAA & 802.1X / CWA 設定

Catalyst 9800 WLC では、従来の AireOS と異なり、AAA 設定・WLAN 構成・Policy Profile・AAA Policy を分離してモジュール設計します。

```text
! --- 1. Global AAA Configuration on C9800 ---
aaa new-model
aaa authentication login default local
aaa authentication dot1x ISE_DOT1X_GROUP group RADIUS-ISE-GROUP
aaa authorization network ISE_AUTHZ_GROUP group RADIUS-ISE-GROUP
aaa accounting network ISE_ACCT_GROUP start-stop group RADIUS-ISE-GROUP

aaa server radius dynamic-author
 client 192.168.10.100 server-key Cisco123!
 client 192.168.10.101 server-key Cisco123!
 auth-type any

! --- 2. C9800 Redirect ACL ---
ip access-list extended ACL_WLC_CWA_REDIRECT
 deny   udp any any eq domain
 deny   udp any any eq bootps
 deny   udp any any eq bootpc
 deny   ip any host 192.168.10.100
 deny   ip any host 192.168.10.101
 permit tcp any any eq www

! --- 3. AAA Policy & Authorization Profile ---
wireless aaa policy ISE_AAA_POLICY
 aaa-override
 nac
 aaa-accounting ISE_ACCT_GROUP

! --- 4. WLAN Configuration (802.1X Enterprise) ---
wlan Corp_Dot1x 2 Corp_SSID
 security wpa wpa2 ciphers aes
 security wpa akm dot1x
 no shutdown

! --- 5. Policy Profile Configuration ---
wireless tag policy DEFAULT_POLICY_TAG
 wlan Corp_Dot1x policy Corp_Policy_Profile

wireless profile policy Corp_Policy_Profile
 aaa-policy ISE_AAA_POLICY
 vlan 20
 central switching
 central authentication
 central dhcp
 no shutdown
```

---


## 🔍 検証コマンド

Cisco スイッチおよび WLC における AAA 連携、セッション状態、RADIUS パケットの検証コマンドを下表にまとめます。

| 目的 | コマンド | 備考 |
| --- | --- | --- |
| 認証セッション状態の確認 | `show access-session interface <int> details` | MAC, IP, 認証方式(Dot1x/MAB), 認可属性(dACL, SGT, VLAN)の網羅的確認 |
| セッションサマリーの確認 | `show access-session summary` | 全ポートのセッションステータス一覧 |
| 適用済み dACL の表示 | `show ip access-lists dynamic` | RADIUS 経由で ISE からダウンロードされた DACL の適用状態確認 |
| RADIUS サーバー統計確認 | `show radius server-group RADIUS-ISE-GROUP` | RTT, Access-Requests, Access-Accepts, Retries, Dead 状態の確認 |
| 802.1X 統計情報の確認 | `show dot1x statistics interface <int>` | EAPoL パケット送信数・受信数・タイムアウト数の確認 |
| Device Sensor 収集状態 | `show device-sensor cache interface <int>` | DHCP/CDP/LLDP から収集した TLV 情報の確認 |
| WLC クライアント詳細 | `show wireless client mac-address <MAC> detail` | C9800 における RADIUS 認可属性・SGT・リダイレクト状態確認 |
| AireOS クライアント詳細 | `show client detail <MAC>` | AireOS における AAA 設定・VLAN・リダイレクト URL 確認 |
| 802.1X デバッグ (スイッチ) | `debug dot1x all` | EAPoL チャレンジ/レスポンスのリアルタイムデバッグ |
| MAB デバッグ (スイッチ) | `debug mab all` | MAB 状態遷移と RADIUS 送信処理のログ出力 |
| 認証セッションデバッグ | `debug access-session all` | IBNS 2.0 C3PL ポリシー実行および CoA 処理のデバッグ |
| RADIUS パケットデバッグ | `debug radius verbose` | RADIUS アトリビュート (AV-Pair) の送受信解読 |
| WLC 9800 追跡デバッグ | `debug wireless mac <MAC>` | C9800 におけるクライアント接続・AAA 認可処理の集中デバッグ |

---

## 🚨 トラブルシューティング

AAA 連携およびアクセス制御において発生する代表的な障害パターン、原因、確認コマンド、および解決策を下表に整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **802.1X 認証が常にタイムアウトし MAB に落ちる** | 端末の EAPoL サプリカントが無効、またはスイッチの `tx-period` タイマーが長すぎる | `show access-session int <int> details`<br>`show dot1x statistics int <int>` | 端末の WLAN AutoConfig / 802.1X サービスを有効化。スイッチ側で `dot1x timeout tx-period 7` や `max-req 2` に短縮調整 |
| **ISE で Access-Accept が出ているが端末が IP を取得できない** | 動的 VLAN 割り当て後の Port Re-auth または DHCP Release/Renew の不具合 | `show access-session int <int> details`<br>`show ip interface brief` | スイッチポートの Native VLAN / Access VLAN 設定を確認。必要に応じて ISE の Authz Profile で 「Send Access-Accept with VLAN」 とともに DHCP Renew 制御を構成 |
| **CWA ポータルへリダイレクトされない** | リダイレクト ACL の `permit/deny` 論理逆転、または DNS 通信のブロック | `show access-session int <int> details`<br>`show ip access-lists ACL_CWA_REDIRECT` | リダイレクト ACL 内で DNS (UDP 53) および ISE IP への通信を `deny`（リダイレクト除外）にし、HTTP (TCP 80) を `permit` に修正 |
| **RADIUS CoA 実行時に "CoA Failed" エラー** | CoA ポート番号の不一致 (1700 vs 3799)、または RADIUS 共有鍵の不一致 | `show aaa server radius dynamic-author`<br>`debug radius verbose` | ISE の NAD 設定とスイッチの `aaa server radius dynamic-author` 内の `port` 番号および Shared Secret を完全一致させる |
| **dACL 適用時に "dACL Apply Failed"** | スイッチに `ip device tracking` や IPDT/SISF が未構成、または dACL 文法エラー | `show ip access-lists dynamic`<br>`show logging` | `device-tracking tracking` を有効化。ISE 上の dACL 定義で `permit ip any any` 等の標準記法に従い構文チェックを実行 |
| **WLC FlexConnect で CWA 失敗** | FlexConnect AP 側にリダイレクト ACL 名がローカル同期・定義されていない | `show wireless client mac <MAC> detail`<br>(AP CLI) `show flexconnect acl` | FlexConnect Group にリダイレクト ACL 名を追加登録し、AP に設定をプッシュ |

---

## ⚠ 制限事項

* **dACL のハードウェア TCAM 制限**: スイッチの TCAM エントリ数上限を超える複雑な dACL を大量のポートに適用すると、dACL の適用が失敗し、セッションがドロップまたはデフォルト拒否状態になります。
* **IBNS 1.0 と IBNS 2.0 の非互換性**: 同一スイッチ内で IBNS 1.0 (Legacy `dot1x` コマンド) と IBNS 2.0 (`policy-map type control subscriber`) を混在させることは推奨されず、グローバルな変換コマンド（`ibns integrate`）が必要です。
* **FlexConnect Local Switching の DACL 制限**: FlexConnect Local Switching モードでは、ISE から動的に注入される dACL がサポートされない場合があり、原則として AP 側にローカル ACL（FlexConnect ACL）を事前定義して名前で参照する必要があります。

---

## 🔄 他技術との関連

* **Cisco TrustSec (SGT / SXP)**: ISE は RADIUS Access-Accept 内で Security Group Tag (SGT: 16-bit 数値) を NAD に割り当てます。スイッチはパケットの CMD / Inline Tagging または SXP 経由で SGT を伝送し、SGACL によるセグメンテーションを実行します。
* **802.1X / EAP-TLS & PKI**: 802.1X EAP-TLS 認証では、クライアントと ISE 間の双方向証明書検証が必要です。Cisco IOS CA または外部 Active Directory CS と ISE の証明書チェーン統合が前提となります。
* **Guest Portal & WebAuth**: CWA 動作時、スイッチは HTTP 302 を返して ISE の FQDN（例: `ise.example.com`）へ誘導するため、クライアントが ISE の FQDN を正しく名前解決できる DNS 設定が必須です。

---

## 🧩 比較表

### 1. IBNS 1.0 (Legacy) vs IBNS 2.0 (C3PL / Control Policy)

| 比較項目 | IBNS 1.0 (Legacy) | IBNS 2.0 (C3PL) |
| --- | --- | --- |
| **設定体系** | インターフェース単位の個別コマンド（`dot1x host-mode`, `mab`） | モジュール化された Class-Map / Policy-Map 構造 |
| **柔軟性** | 限定的（固定の認証順序: Dot1x -> MAB） | 極めて高い（イベント・条件に応じたカスタム状態遷移） |
| **マルチセッション** | 設定が複雑（`host-mode multi-domain` 等） | `access-session host-mode` と Template による一括管理 |
| **Cisco 推奨** | 非推奨 (Legacy) | Cisco IOS-XE 標準推奨アーキテクチャ |

### 2. WLC 無線制御モデル (Central vs FlexConnect Local Switching)

| 比較項目 | Central Authentication / Central Switching | FlexConnect Central Auth / Local Switching |
| --- | --- | --- |
| **データプレーン** | 全クライアント通信を WLC へ CAPWAP トンネル伝送 | AP がローカルスイッチの VLAN へ直結ブリッジング |
| **コントロールプレーン** | WLC が集中管理し ISE と RADIUS 通信 | 認証は WLC 経由で ISE 通信、データは AP 側で処理 |
| **dACL サポート** | 完全サポート (WLC 上に適用) | 制限あり (FlexConnect ACL による名前参照推奨) |
| **WAN 障害耐性** | WAN 切断時に通信全停止 | Local Auth / Local Switching モードへフォールバック可能 |

---

## 💡 ベストプラクティス

1. **RADIUS タイムアウトとサーバーデッドタイマーの最適化**:
   `radius-server retransmit 3`, `radius-server timeout 5`, `aaa-server radius dynamic-author` 内の共有鍵設定を標準化し、ISE のメンテナンス時にも迅速にセカンダリ ISE へ切り替わるよう設計します。
2. **Device Sensor の必須構成**:
   プロファイリング精度向上のため、スイッチの全アクセスポートで `device-sensor filter-spec` を有効化し、RADIUS Accounting パケットに DHCP Option 55/60 や CDP/LLDP TLV を付与して ISE に送信します。
3. **CWA リダイレクト ACL の厳格な設計**:
   リダイレクト ACL において、DNS, DHCP, ISE 通信を確実に `deny`（スルー）させ、不要なリダイレクトループやポータル接続不能障害を防止します。

---


## 📝 ラボ学習・設定サンプル例

### 演習1: スイッチにおける IBNS 2.0 (C3PL) 802.1X / MAB フォールバック構成

#### 【問題】
Catalyst 9300 スイッチのポート GigabitEthernet1/0/1 において、IBNS 2.0 (C3PL) を使用して 802.1X 認証を第一優先、MAB 認証を第二優先とする認証ポリシーを構成し、ISE サーバー (192.168.10.100) と連携しなさい。

#### 【要件】
1. EAPoL タイムアウト時間は 7 秒、最大リクエスト回数は 2 回に設定すること。
2. 端末が 802.1X に応答しない場合、自動的に MAB 認証へ移行すること。
3. RADIUS Accounting を有効化し、5分周期で Periodic Update を送信すること。

#### 【設定例】
```text
! --- 1. Global AAA & RADIUS Configuration ---
aaa new-model
aaa authentication dot1x ISE_GROUP group RADIUS-ISE
aaa authorization network ISE_GROUP group RADIUS-ISE
aaa accounting update periodic 5
aaa accounting network ISE_GROUP start-stop group RADIUS-ISE

radius server ISE-PRIMARY
 address ipv4 192.168.10.100 auth-port 1812 acct-port 1813
 key Cisco123!

aaa group server radius RADIUS-ISE
 server name ISE-PRIMARY

aaa server radius dynamic-author
 client 192.168.10.100 server-key Cisco123!
 port 1700

dot1x system-auth-control

! --- 2. C3PL Class-Maps & Policy-Map ---
class-map type control subscriber match-all DOT1X_FAILED
 match method dot1x priority 10
 match result-type method dot1x failed

policy-map type control subscriber IBNS2_POLICY
 event session-started action number 10
  execute factory-control-plan
 event authentication-failure class DOT1X_FAILED action number 20
  authenticate using mab priority 20
 event agent-found action number 10
  authenticate using dot1x priority 10

! --- 3. Template Definition ---
template C3PL_ACCESS_TEMPLATE
 dot1x max-req 2
 dot1x timeout tx-period 7
 mab
 access-session port-control auto
 access-session host-mode single-host
 service-policy type control subscriber IBNS2_POLICY

! --- 4. Interface Application ---
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
 source template C3PL_ACCESS_TEMPLATE
 spanning-tree portfast
```

#### 【検証方法】
1. 端末を接続し、`show access-session interface GigabitEthernet1/0/1 details` を実行して、認証方式が `dot1x` または `mab` で Status が `Authorized` となっていることを確認。
2. `show radius server-group RADIUS-ISE` で RADIUS Access-Request および Access-Accept のカウント増加を確認。

---

### 演習2: スイッチにおける CWA (Central Web Authentication) とリダイレクト ACL 構成

#### 【問題】
未認証クライアントに対して ISE のゲストポータルへリダイレクトを行う CWA (Central Web Authentication) を実装しなさい。

#### 【要件】
1. リダイレクト ACL 名は `ACL_CWA_REDIRECT` とすること。
2. DNS (UDP 53)、DHCP (UDP 67/68)、および ISE (192.168.10.100) への通信はリダイレクトさせず通過させること。
3. HTTP (TCP 80) 通信のみを ISE ポータルへリダイレクトさせること。

#### 【設定例】
```text
! --- Redirect ACL ---
ip access-list extended ACL_CWA_REDIRECT
 deny   udp any any eq domain
 deny   udp any any eq bootps
 deny   udp any any eq bootpc
 deny   ip any host 192.168.10.100
 permit tcp any any eq www

! --- AAA Dynamic Authorization (CoA) ---
aaa server radius dynamic-author
 client 192.168.10.100 server-key Cisco123!
 port 1700
 auth-type any
```

#### 【検証方法】
1. CWA 認可プロファイルが適用されたセッションで `show access-session interface GigabitEthernet1/0/2 details` を実行。
2. `URL Redirect ACL: ACL_CWA_REDIRECT` および `URL Redirect: https://ise.example.com:8443/...` が正しくアタッチされていることを確認。
3. クライアントで ブラウザを開き HTTP リクエストを発生させ、ISE ログイン画面が表示されることを確認。

---

### 演習3: Multi-Domain Authentication (IP Phone + PC) 構成

#### 【問題】
同一スイッチポートに Cisco IP Phone と PC がインライン接続される環境において、音声とデータを分離認証する Multi-Domain Authentication を構成しなさい。

#### 【要件】
1. PC は VLAN 10（データ）、IP Phone は VLAN 100（音声）に配属すること。
2. IP Phone は MAB、PC は 802.1X でそれぞれ個別に ISE 認証させること。
3. CDP による Voice VLAN 割り当てを正常動作させること。

#### 【設定例】
```text
interface GigabitEthernet1/0/3
 description Voice_and_Data_Port
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 100
 access-session host-mode multi-domain
 access-session port-control auto
 mab
 dot1x pae authenticator
 spanning-tree portfast
```

#### 【検証方法】
1. `show access-session interface GigabitEthernet1/0/3 details` を実行し、同一ポート上で 2 つの MAC アドレスが認識され、一方が `Domain: VOICE` (VLAN 100)、他方が `Domain: DATA` (VLAN 10) として `Authorized` になっていることを確認。

---

### 演習4: Catalyst 9800 WLC 802.1X Enterprise (Central Switching) 構成

#### 【問題】
Catalyst 9800 WLC において、SSID `Corp_Secure` に対して 802.1X 認証および Central Switching を行う WLAN を構成しなさい。

#### 【要件】
1. WPA2-Enterprise (AES) 方式を使用すること。
2. RADIUS 認証および Accounting を ISE サーバーグループ `ISE_RADIUS_GRP` 経由で行うこと。
3. クライアント VLAN は VLAN 20 とすること。

#### 【設定例】
```text
! RADIUS Group & Server
radius server ISE_1
 address ipv4 192.168.10.100 auth-port 1812 acct-port 1813
 key Cisco123!

aaa group server radius ISE_RADIUS_GRP
 server name ISE_1

aaa authentication dot1x ISE_DOT1X_AAA group ISE_RADIUS_GRP
aaa accounting network ISE_ACCT_AAA start-stop group ISE_RADIUS_GRP

! AAA Policy
wireless aaa policy ISE_WIRELESS_AAA_POLICY
 aaa-override
 nac
 aaa-accounting ISE_ACCT_AAA

! WLAN Configuration
wlan Corp_Secure_WLAN 1 Corp_Secure
 security wpa wpa2 ciphers aes
 security wpa akm dot1x
 no shutdown

! Policy Profile
wireless profile policy Corp_Policy_Profile
 aaa-policy ISE_WIRELESS_AAA_POLICY
 vlan 20
 central switching
 central authentication
 no shutdown

! Policy Tag
wireless tag policy Corp_Policy_Tag
 wlan Corp_Secure_WLAN policy Corp_Policy_Profile
```

#### 【検証方法】
1. C9800 CLI で `show wireless client summary` を実行し、端末の MAC アドレスが表示されていることを確認。
2. `show wireless client mac-address <MAC> detail` で `Authentication: 802.1X`, `State: Associated`, `VLAN ID: 20` を確認。

---

### 演習5: Catalyst 9800 WLC FlexConnect Local Switching & MAB 認証構成

#### 【問題】
拠点 AP 向けに SSID `IoT_Flex` を作成し、FlexConnect Local Switching および MAB 認証を構成しなさい。

#### 【要件】
1. MAB 認証は C9800 経由で ISE にて実施すること。
2. データパケットは AP ローカルスイッチ側 VLAN 30 にブリッジングすること。

#### 【設定例】
```text
! WLAN Config
wlan IoT_Flex_WLAN 2 IoT_Flex
 mac-filtering ISE_DOT1X_AAA
 no security wpa
 no shutdown

! Flex Profile
wireless profile flex Flex_Profile_Branch
 vlan-name Flex_VLAN30
  vlan-id 30
 native-vlan-id 1

! Policy Profile
wireless profile policy Flex_Policy_Profile
 aaa-policy ISE_WIRELESS_AAA_POLICY
 vlan Flex_VLAN30
 no central switching
 central authentication
 flex-profile Flex_Profile_Branch
 no shutdown
```

#### 【検証方法】
1. C9800 CLI で `show wireless client mac-address <MAC> detail` を実行し、`Central Switching: Disabled`, `FlexConnect Data Switching: Local` であることを検証。

---


### 演習6: スイッチにおける Device Sensor (DHCP / CDP / LLDP) 構成

#### 【問題】
ISE でのデバイスプロファイリング精度を高めるため、スイッチポートで DHCP, CDP, LLDP の属性情報をキャプチャし、RADIUS Accounting パケットに含めて ISE に送信する Device Sensor を構成しなさい。

#### 【要件】
1. DHCP Option 55 および Option 60 を収集対象とすること。
2. CDP および LLDP のシステム名・デバイスタイプ情報を収集対象とすること。
3. 属性変化時に即時 RADIUS Accounting Notification を発効すること。

#### 【設定例】
```text
! --- 1. Define Filter Lists ---
device-sensor filter-list dhcp list DHCP_LIST
 option name host-name
 option name parameter-request-list
 option name class-identifier

device-sensor filter-list cdp list CDP_LIST
 tlv name device-name
 tlv name capabilities-address

device-sensor filter-list lldp list LLDP_LIST
 tlv name system-name
 tlv name system-description

! --- 2. Apply Filters to Device Sensor ---
device-sensor filter-spec dhcp include list DHCP_LIST
device-sensor filter-spec cdp include list CDP_LIST
device-sensor filter-spec lldp include list LLDP_LIST

! --- 3. Enable Accounting Notification ---
device-sensor notify group-listening
```

#### 【検証方法】
1. 端末接続後に `show device-sensor cache interface GigabitEthernet1/0/1` を実行し、DHCP Option 60 や CDP/LLDP TLV がキャッシュされていることを確認。
2. `debug radius verbose` を有効化し、送信される RADIUS Accounting-Request 内に `cisco-av-pair = dhcp-class-identifier=...` が含まれていることを確認。

---

### 演習7: RADIUS Change of Authorization (CoA) 構成と検証

#### 【問題】
ISE からの発行命令に応じて、アクセスセッションの再認証（Re-authentication）およびポートバウンス（Port Bounce）を実行できるよう、スイッチ側で Dynamic Authorization (CoA) を正しく有効化しなさい。

#### 【要件】
1. ISE IP アドレス `192.168.10.100` からの CoA リクエストを許可すること。
2. CoA リクエストの送信先 UDP ポート番号を `1700` とすること。
3. 共有キーは `Cisco123!` とすること。

#### 【設定例】
```text
aaa server radius dynamic-author
 client 192.168.10.100 server-key Cisco123!
 port 1700
 auth-type any
```

#### 【検証方法】
1. ISE の Live Logs または CLI から対象端末セッションに対して "Re-authenticate" または "Port Bounce" CoA を発行。
2. スイッチで `debug access-session CoA` および `show logging` を確認し、`CoA RECEIVED` ログの出力とともにセッションが再認証 / ポートリセットされることを確認。

---

### 演習8: スイッチにおける Local WebAuth (LWA) フォールバック構成

#### 【問題】
802.1X および MAB に失敗した未認証クライアントに対し、スイッチ本体が Web ポータル画面を提供する Local WebAuth (LWA) を構成しなさい。

#### 【要件】
1. LWA 用のパラメータマップ `LWA_PARAMMAP` を作成し、Banner メッセージを設定すること。
2. MAB 失敗時に LWA 認証へとフォールバックさせること。

#### 【設定例】
```text
! --- Parameter Map ---
parameter-map type webauth LWA_PARAMMAP
 type consent
 banner text "Welcome to Corporate Guest Network. Click Accept to Continue."

! --- C3PL Policy Map with LWA Fallback ---
class-map type control subscriber match-all MAB_FAILED
 match method mab priority 20
 match result-type method mab failed

policy-map type control subscriber LWA_FALLBACK_POLICY
 event session-started action number 10
  execute factory-control-plan
 event authentication-failure class MAB_FAILED action number 30
  authenticate using webauth priority 30

! --- Interface Configuration ---
interface GigabitEthernet1/0/4
 switchport mode access
 access-session port-control auto
 service-policy type control subscriber LWA_FALLBACK_POLICY
 ip access-group ACL_DEFAULT_GUEST in
```

#### 【検証方法】
1. 未登録 PC を接続し、MAB タイムアウト後にブラウザで Web 閲覧を開始。
2. スイッチのローカル Web ポータル画面が表示され、「Accept」をクリック後にアクセスが許可されることを確認。

---

### 演習9: TACACS+ コマンド認可 (Command Authorization) 構成

#### 【問題】
スイッチの CLI において、特権レベル 15 のユーザーが実行するすべての設定コマンドを ISE (TACACS+) で認可・ログ収集（Accounting）するように構成しなさい。

#### 【要件】
1. TACACS+ サーバー `192.168.10.100` (Key: `TacacsKey123!`) を定義すること。
2. レベル 15 のコマンド実行時に TACACS+ 認可を必須とすること。
3. TACACS+ 不可視時はローカル特権ユーザーへフォールバックすること。

#### 【設定例】
```text
! --- TACACS+ Server Config ---
tacacs server ISE-TACACS
 address ipv4 192.168.10.100
 key TacacsKey123!

aaa group server tacacs+ TAC_GROUP
 server name ISE-TACACS

! --- AAA Authorization & Accounting ---
aaa authorization exec default group TAC_GROUP local
aaa authorization commands 15 default group TAC_GROUP local
aaa accounting commands 15 default start-stop group TAC_GROUP
```

#### 【検証方法】
1. スイッチに SSH ログインし、`configure terminal` や `interface GigabitEthernet1/0/1` を実行。
2. ISE の TACACS+ Live Logs を確認し、実行されたコマンド文がリアルタイムで Pass / Fail 記録されていることを確認。

---

### 演習10: AireOS WLC 802.1X & AAA Override 構成

#### 【問題】
AireOS WLC (例: 5520 / 3504) において、SSID `Enterprise_AireOS` を作成し、ISE からの Dynamic VLAN / SGT 割り当て（AAA Override）を有効化しなさい。

#### 【要件】
1. RADIUS Server `192.168.10.100` (Port 1812/1813) を設定すること。
2. WLAN の Advanced タブで `AAA Override` を有効にすること。
3. RADIUS 認証の MAC アドレスフォーマットを `Raw Hexadecimal` (小文字ハイフンなし) に設定すること。

#### 【設定例（CLI）】
```text
! --- RADIUS Server Configuration ---
config radius auth add 1 192.168.10.100 1812 ascii Cisco123!
config radius acct add 1 192.168.10.100 1813 ascii Cisco123!

! --- WLAN Configuration ---
config wlan create 3 Enterprise_AireOS Enterprise_SSID
config wlan security wpa wpa2 enable 3
config wlan security wpa wpa2 ciphers aes enable 3
config wlan security wpa wpa2 entitlement dot1x enable 3

! Enable AAA Override & RADIUS Auth/Acct
config wlan aaa-override enable 3
config wlan radius_server auth add 3 1
config wlan radius_server acct add 3 1
config wlan enable 3
```

#### 【検証方法】
1. クライアント接続後、WLC CLI で `show client detail <MAC>` を実行。
2. `AAA Override ACL/VLAN Name` 項目に ISE から注入された VLAN ID / Name が表示されていることを確認。

---


## ❓ 想定試験問題

### 【問題1：コンフィグ読解】
以下の Cisco Catalyst 9300 (IOS-XE) のコンフィグレーションを読み、クライアントが 802.1X 認証に失敗した際の挙動として正しい説明を選択しなさい。

```text
class-map type control subscriber match-all DOT1X_FAILED
 match method dot1x priority 10
 match result-type method dot1x failed

policy-map type control subscriber MAB_FALLBACK
 event session-started action number 10
  execute factory-control-plan
 event authentication-failure class DOT1X_FAILED action number 20
  authenticate using mab priority 20
 event agent-found action number 10
  authenticate using dot1x priority 10
```

* **A**: 802.1X 認証に失敗した場合、即座にセッションがドロップされポートがシャットダウンする。
* **B**: 802.1X 認証に失敗（Timeout または Access-Reject）した場合、イベントクラス `DOT1X_FAILED` にマッチし、優先度 20 で MAB 認証が開始される。
* **C**: 端末から EAPoL パケットが送信されると、MAB 認証が中断されて WebAuth 認証が開始される。
* **D**: MAB 認証に失敗した場合、デフォルトで VLAN 999 に強制移動させられる。

**【正解】**: **B**
**【解説】**: C3PL の Policy Map 定義において、`authentication-failure` イベント発生時に `class DOT1X_FAILED` にマッチすると、アクション `authenticate using mab priority 20` が実行され、自動的に MAB 認証へ移行します。

---

### 【問題2：トラブルシューティング】
Cisco ISE からスイッチポートへ接続された端末に対して RADIUS CoA（Re-authenticate）を発行しましたが、スイッチ側で CoA が受け付けられず、`show logging` に何のアクティビティも記録されません。ISE 上では "CoA Timeout / No Response" エラーが記録されています。考えられる根本原因として最も適切なものを2つ選びなさい。

* **A**: スイッチポートで `dot1x system-auth-control` が有効化されていない。
* **B**: スイッチ側で `aaa server radius dynamic-author` が未設定、または ISE IP が client として定義されていない。
* **C**: スイッチと ISE 間で RADIUS CoA 用の UDP ポート番号（例: 1700 / 3799）が不一致である。
* **D**: 端末の IP アドレスが DHCP で取得できていない。
* **E**: スイッチの Interface で `ip access-group` が設定されていない。

**【正解】**: **B, C**
**【解説】**: Dynamic Authorization (CoA) を処理するためには、NAD（スイッチ）側で `aaa server radius dynamic-author` セクション内に ISE の IP アドレスと Shared Key、および一致する CoA ポート（一般に UDP 1700 または 3799）を構成する必要があります。

---

### 【問題3：Design / アーキテクチャ】
大規模エンタープライズの拠点（Branch Office）における Catalyst 9800 WLC 設計において、WAN 切断時であっても拠点のローカル無線端末同士の通信およびアクセス制御を維持したい。最も推奨される設計アーキテクチャはどれか。

* **A**: Central Authentication / Central Switching
* **B**: FlexConnect Local Authentication / Local Switching
* **C**: Mesh AP Deployment with Central Switching
* **D**: Autonomous AP Deployment with TACACS+

**【正解】**: **B**
**【解説】**: WAN 障害発生時に本社 Cisco ISE との通信が途絶しても、FlexConnect Local Authentication / Local Switching モードを構成しておくことで、AP 自信がローカルで EAP 認証・MAB 処理およびローカル VLAN ブリッジングを継続できます。

---

### 【問題4：実装】
CWA（Central Web Authentication）をスイッチポートで構成する際、リダイレクト ACL（`ip access-list extended ACL_CWA_REDIRECT`）のエントリ設計として正しいものはどれか。

* **A**: DNS、DHCP、および ISE への通信を `permit` し、その他の HTTP 通信を `deny` する。
* **B**: DNS、DHCP、および ISE への通信を `deny`（リダイレクト除外）し、標準 HTTP 通信（TCP 80）を `permit`（リダイレクト対象）にする。
* **C**: すべての IP トラフィックを `permit` に設定する。
* **D**: ISE への HTTPS (TCP 8443) 通信を `permit` に設定する。

**【正解】**: **B**
**【解説】**: Cisco スイッチのリダイレクト ACL の動作ロジックでは、`deny` が「リダイレクトせず通常通り通過（Bypass）」を意味し、`permit` が「キャプチャして ISE ポータル URL へリダイレクト」を意味します。そのため、DNS / DHCP / ISE IP への通信は必ず `deny` にする必要があります。

---

### 【問題5：コンフィグ読解】
以下の WLC 9800 の設定において、クライアントに ISE からの Dynamic VLAN や SGT を適用するために不足している設定項目はどれか。

```text
wireless profile policy Corp_Policy_Profile
 vlan 10
 central switching
 central authentication
 no shutdown
```

* **A**: `aaa-policy` の割り当ておよび `aaa-override` の有効化
* **B**: `security wpa akm psk` の設定
* **C**: `ip dhcp snooping` の有効化
* **D**: `device-tracking` の無効化

**【正解】**: **A**
**【解説】**: C9800 では、ISE から受信した RADIUS 属性（VLAN ID, SGT, dACL）を適用するために、`wireless aaa policy` 内で `aaa-override` を有効化し、それを `wireless profile policy` に `aaa-policy <NAME>` としてバインドする必要があります。

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび各種技術資料への参照リンク集です。

* Cisco ISE Administrator Guide - Access Control and Authentication
* Cisco Catalyst 9000 Series Switches Security Configuration Guide - IBNS 2.0 Deployment
* Cisco Catalyst 9800 Series Wireless Controller Software Configuration Guide - Wireless AAA and ISE Integration
* Cisco Technical Notes - Configure Central Web Authentication (CWA) with Cisco ISE and Catalyst Switches
* Cisco Technical Notes - Catalyst 9800 WLC Central Web Authentication (CWA) Configuration Example
* Cisco Validated Design (CVD) - Campus Network Unified Access Security Design Guide
* Cisco Live Presentation - Advanced IBNS 2.0 and C3PL Deployment in Campus Networks

---

## 📝 補足（Notes）

* **C3PL の状態遷移図メモ**:
  セッションスタート → 802.1X EAPoL 発行 → 応答なし (Timeout) → Class `DOT1X_FAILED` マッチ → MAB 実行 → ISE から Access-Accept (CWA 属性付き) 受信 → ポート状態: Authorized (CWA Redirection) → ユーザーポータルログイン → ISE から CoA-Request (Re-auth) 到着 → 再認証実行 → 最終 Access-Accept (Full Access DACL + SGT 4) 適用。
* **AireOS vs C9800 設定用語の対比**:
  - AireOS `AAA Override` ＝ C9800 `wireless aaa policy` -> `aaa-override`
  - AireOS `FlexConnect Local Switching` ＝ C9800 `wireless profile policy` -> `no central switching`
  - AireOS `FlexConnect Group` ＝ C9800 `wireless profile flex` + `flex-profile`


