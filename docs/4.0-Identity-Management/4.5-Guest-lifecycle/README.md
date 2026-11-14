---
layout: default
title: 4.5-Guest-lifecycle
nav_order: 5
parent: 4.0-Identity-Management
---

# 4.5 Guest lifecycle management using Cisco ISE and Cisco WLC

## 📘 概要

ゲストライフサイクル管理（Guest Lifecycle Management）は、企業ネットワークにアクセスする外部訪問者（ゲストユーザー）に対して、一貫した認証・認可・ポータル提示・アカウント有効期限制御・利用監査を提供するセキュリティ機構である。Cisco Identity Services Engine (ISE) と Cisco Wireless LAN Controller (WLC: AireOS または Catalyst 9800) の統合により、無承認デバイスのアクセス遮断から、セルフ登録ポータル（Self-Registration Portal）、スポンサー認証ポータル（Sponsor Portal）、アクセスタイムアウトに伴う自動アカウント無効化・パージまでの一連のライフサイクルを自動化する。

ラボ試験および実務設計における導入目的は、ゲストアクセスの利便性を担保しつつ、企業内部リソースへの不正アクセスやマルウェア拡散を防ぐことにある。具体的には、Central Web Authentication (CWA) や Foreign/Anchor WLC 構成、BYOD WebAuth リダイレクトと連携し、アクセス試行時に動的に外部Webポータルへ誘導し、ゲスト種別（Hotspot, Self-Reg Guest, Sponsored Guest）に応じた柔軟なアクセス制御ポリシー（dACL, Dynamic VLAN, SGT）を適用する。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **特徴** | RADIUS CoA (Change of Authorization)、CWA (Central Web Authentication)、Guest Portals (ISE)、Sponsor Portals の統合管理 |
| **用途** | 来訪者（Guest）に対する一時的Wi-Fi/有線アクセスの提供、セルフ登録・スポンサー承認ベースの認証、アクセス時間・利用期間の自動制限 |
| **メリット** | ネットワーク管理者の手動アカウント発行負荷の削減、 guest アカウントの自動有効期限切れ・パージ機能によるセキュリティ向上、詳細な監査ログ保持 |
| **デメリット** | WLC と ISE 間の HTTPS リダイレクト・RADIUS CoA 連携の設定複雑性、証明書（Public/Private CA）の適切な配置が必須 |
| **対応機種** | Cisco ISE 2.x / 3.x, Cisco Catalyst 9800 シリーズ WLC, Cisco AireOS WLC (5520/8540/3504), Catalyst スイッチ (IOS-XE) |
| **制限事項** | CoA ポート (UDP 1700 / RFC 5176 UDP 3799) の到達性が必須。WLC 側での URL Redirect ACL 名と ISE 認可プロファイルの設定名の一致が厳格 |
| **設計上の注意点** | ゲストアクセス用ポータル証明書には、クライアント端末で警告が出ない公的証明書（Public CA）またはワイルドカード証明書の使用を推進。DNS 解消の確保 |

---

## 🏗 動作原理

ゲストライフサイクル管理の動作メカニズムは、初回のアクセス試行（未認証状態）から、ポータルリダイレクト、ユーザー認証/アカウント登録、CoA による認可更新、アクセス終了（パージ）までのフェーズで構成される。

```
Client / Guest Device
   │
   │  1. Wi-Fi Association / 802.1X (MAB)
   ▼
Cisco WLC (NAS) ═══════════════════════════════════ Authorization Request ═══► Cisco ISE (RADIUS)
   │                                                                               │
   │  2. RADIUS Access-Accept (Redirect-URL & ACL)                                 │
   ◄───────────────────────────────────────────────────────────────────────────────┘
   │
   │  3. HTTP/HTTPS Traffic Init
   ▼
Cisco WLC
   │  4. Intercept & Redirect (HTTP 302 to ISE Guest Portal)
   ▼
Guest Client Browser ══════════════════════════════ HTTP/HTTPS Portal ═══════► Cisco ISE Guest Portal
   │                                                                               │
   │  5. Self-Registration / Sponsor Approval / Login                              │
   │──────────────────────────────────────────────────────────────────────────────►│
   │                                                                               │
   │                                              6. RADIUS CoA (Reauth/Disconnect) │
   │ Cisco WLC ◄───────────────────────────────────────────────────────────────────│
   │     │                                                                         │
   │     │  7. New RADIUS Access-Request (Authenticated)                           │
   │     └════════════════════════════════════════════════════════════════════════►│
   │                                                                               │
   │  8. RADIUS Access-Accept (PermitAccess / dACL / Dynamic VLAN)                 │
   ◄───────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

1. **初期接続・認証試行 (Initial Association & MAB/802.1X)**:
   - ゲストクライアントが Guest SSID (例: `Guest-Wi-Fi`) に接続。
   - WLC は MAC Authentication Bypass (MAB) または Open Authentication を使用して RADIUS Access-Request を ISE に送信。

2. **CWA リダイレクト認可の返却 (CWA Redirect Authorization)**:
   - ISE は認証ポリシーで未認証ゲストと判定し、認可プロファイル `Guest_Redirect` を適用。
   - ISE は Access-Accept 内で `cisco-av-pair = url-redirect-acl=REDIRECT_GUEST_ACL` および `cisco-av-pair = url-redirect=https://ise.example.com:8443/portal/gateway?sessionId=...` を WLC に返却。

3. **トラフィックのリダイレクト (Traffic Intercept & HTTP 302)**:
   - WLC はクライアントのセッション状態を `Webauth Pending` に設定。
   - クライアントが Web ブラウザを起動し HTTP/HTTPS リクエストを発行すると、WLC の リダイレクト ACL が捕捉し、ISE ポータル URL への HTTP 302 Redirect を応答。

4. **ポータル処理と登録/承認 (Portal Processing & Self-Reg / Sponsor)**:
   - クライアントが ISE の Guest Portal (TCP 8443 または TCP 8905) に接続。
   - Hotspot ポータル: 利用規約 (AUP) 同意のみでアクセス許可。
   - Self-Registration ポータル: ゲストが自身の情報（氏名、SMS/Email）を入力してアカウント作成。スポンサー承認が必要な場合は Sponsor ポータルへ通知メールが飛ぶ。
   - Sponsored Guest ポータル: スポンサー（社内社員）が Sponsor Portal (TCP 8443) から事前に発行したクレデンシャルを入力。

5. **RADIUS CoA の発行 (Change of Authorization)**:
   - 認証/登録完了後、ISE は WLC に対答して RADIUS CoA (Reauth または Disconnect) を送信 (UDP 1700 または UDP 3799)。

6. **再認証とネットワークアクセスの許可 (Re-authentication & Permit)**:
   - WLC は CoA を受け取り、クライアントの RADIUS セッションを再評価。新たな RADIUS Access-Request を ISE に送信。
   - ISE はゲストの MAC アドレスが `Endpoints:GuestEndpoints` グループに登録されたことを確認し、アクセス許可（PermitAccess、dACL、VLAN 変更など）を含む Access-Accept を返却。

7. **ライフサイクル管理とパージ (Account Expiration & Purge)**:
   - アカウント定義（例: 8 時間有効、7 日間有効）に基づき、有効期限が切れると ISE の Guest Purge Policy が起動。
   - アカウントおよび Endpoints データベースから MAC アドレスが削除され、次回接続時は再度ポータルリダイレクトが発生。



## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Securityラボ試験において、ゲストアクセスおよび CWA (Central Web Authentication) は頻出項目である。試験での主なポイントと実装時の留意事項を以下に整理する。

* **Blueprintでの重要ポイント**:
  - **CWA Redirect ACLの正確な定義**: WLC 側の Redirect ACL は「リダイレクトさせたいトラフィックを deny、リダイレクトさせないトラフィックを permit」とする（通常のアクセス制御ACLとは論理が逆転する）。特に DNS (UDP 53) や ISE IP (TCP 8443) への通信を `permit` しないと、DNS 解決不能やポータル非表示に陥る。
  - **AAA / RADIUS CoA の疎通**: WLC 側で `aaa server radius dynamic-author` (IOS-XE / Cat 9800) または RFC 3576 / RFC 5176 設定が正しく投入され、UDP 1700 / 3799 が到達可能であること。
  - **Guest Access Typeの使い分け**: Hotspot (AUPのみ)、Self-Registration (セルフ登録・承認フローあり/なし)、Sponsored Guest (スポンサーが事前発行/Sponsor Portalで承認) の設定要件の違いを理解すること。

* **よくある設定ミス**:
  1. **ACL 名の不一致**: ISE の Authorization Profile 内で指定する `url-redirect-acl` の文字列と、WLC 側で定義した IPv4 Access List 名が完全一致していない（大文字・小文字の区別）。
  2. **RADIUS Change of Authorization (CoA) 失敗**: WLC の RADIUS 設定で `nac` キーワードを有効化し忘れ、CoA Reauth が処理されない。
  3. **WLC Web-Auth Parameter Map / Virtual IP の未設定**: Catalyst 9800 で `parameter-map type webauth` を WLAN に紐付け忘れる。
  4. **DNS プレリダイレクト阻害**: クライアントがポータル FQDN を解決できない（DNS トラフィックが ACL でブロックされている）。

* **showコマンド / debugログ解読の着眼点**:
  - WLC 側: `show wireless client mac-address <MAC> detail` で `Client State: Associated` / `Webauth Pending` / `RUN` の推移を確認。
  - ISE 側: `Operations > RADIUS > Live Logs` で Auth-Status (Access-Accept), Authorization Policy (Guest_Redirect vs Guest_Access), CoA Status (Success) を確認。

---

## 🛠 設定方法

### 1. Cisco Catalyst 9800 WLC (IOS-XE) での設定例

#### Step 1: RADIUS サーバーおよび CoA の設定
```text
radius server ISE_NODE_1
 address ipv4 192.168.10.50 auth-port 1812 acct-port 1813
 key Cisco123!
exit

aaa group server radius ISE_RADIUS_GROUP
 server name ISE_NODE_1
exit

aaa authentication login default local
aaa authorization network default group ISE_RADIUS_GROUP
aaa accounting network default start-stop group ISE_RADIUS_GROUP

aaa server radius dynamic-author
 client 192.168.10.50 server-key Cisco123!
 port 1700
exit
```

#### Step 2: リダイレクト用 ACL の定義 (Redirect ACL)
```text
ip access-list extended REDIRECT_GUEST_ACL
 10 deny udp any any eq domain
 20 deny tcp any host 192.168.10.50 eq 8443
 30 deny tcp any host 192.168.10.50 eq 8905
 40 permit tcp any any eq www
 50 permit tcp any any eq 443
exit
```

#### Step 3: Web-Auth パラメータマップと WLAN / Policy Profile の設定
```text
parameter-map type webauth global
 type consent
exit

parameter-map type webauth GUEST_WEBAUTH_PARAM
 type redirection
 redirect portal ipv4 192.168.10.50
exit

wlan GUEST_SSID 2 Guest-Wi-Fi
 security wpa psk set-key ascii 0 GuestPass123
 no security wpa wpa2 ft
 client association limit 0
 no shutdown
exit

wireless tag policy GUEST_POLICY_TAG
wlan GUEST_SSID policy GUEST_POLICY_PROFILE
exit

wireless profile policy GUEST_POLICY_PROFILE
 aaa-override
 nac
 aaa-policy default
 vlan 100
 central association
 central switching
 central dhcp
 parameter-map type webauth GUEST_WEBAUTH_PARAM
 no shutdown
exit
```

---

### 2. Cisco ISE (3.1/3.2) での設定例

#### Step 1: ゲストポータルの作成・設定
1. **Work Centers > Guest Access > Configure > Portals & Components**:
   - **Self-Registration Portal** または **Sponsored Guest Portal** を選択。
   - ポータルページの設定（AUP 表記、アカウント有効期間（例: 8時間）、パスワード通知方法（SMS/Email/画面表示））。

#### Step 2: 認可プロファイル (Authorization Profile) の作成
1. **Policy > Policy Elements > Results > Authorization > Authorization Profiles**:
   - 名前: `Guest_Redirect`
   - Access Type: `ACCESS_ACCEPT`
   - Common Tasks:
     - `Web Redirection (CWA / MDM / NSP / CPP)` をチェック。
     - タイプ: `Central Web Authentication`
     - ACL: `REDIRECT_GUEST_ACL`
     - Value: 作成した Guest Portal を選択。

2. **追加の認可プロファイル**:
   - 名前: `Guest_PermitAccess`
   - Access Type: `ACCESS_ACCEPT`
   - Service Template / dACL: 必要に応じて `PERMIT_ALL_TRAFFIC` や制限付き dACL を適用。

#### Step 3: 認証・認可ポリシー (Policy Sets) の定義
1. **Policy > Policy Sets > GUEST_POLICY_SET**:
   - **Authentication Policy**:
     - Rule 1: `MAB_Auth` -> Condition: `Network Access : AccessType EQUALS Wired_MAB OR Wireless_MAB` -> Use: `Internal Endpoints` (Continue if User/Device not found).
   - **Authorization Policy**:
     - Rule 1 (初回接続時): `Condition: Network Access : UseCase EQUALS Guest Access AND Endpoints : Endpoints:GuestEndpoints NOT_IN_GROUP` -> Result: `Guest_Redirect`
     - Rule 2 (認証完了後): `Condition: Network Access : UseCase EQUALS Guest Access AND Endpoints : Endpoints:GuestEndpoints IN_GROUP` -> Result: `Guest_PermitAccess`

---

## 🔍 検証コマンド

ゲストライフサイクル管理の動作状態、リダイレクト状態、CoA 処理の検証に使用する主要コマンド一覧：

| 目的 | コマンド |
| --- | --- |
| **クライアントセッション詳細確認** | <code>show wireless client mac-address &lt;MAC&gt; detail</code> |
| **WLCリダイレクト状態・ACL確認** | <code>show wireless client mac-address &lt;MAC&gt; policy</code> |
| **RADIUS サーバー状態・統計** | <code>show aaa servers</code> |
| **CoA 受信ログの確認** | <code>show aaa dynamic-author server statistics</code> |
| **リダイレクト ACL 設定確認** | <code>show ip access-lists REDIRECT_GUEST_ACL</code> |
| **ISE RADIUS ライブログ追跡** | ISE GUI: <code>Operations &gt; RADIUS &gt; Live Logs</code> |
| **ISE ゲストセッション追跡** | ISE GUI: <code>Operations &gt; RADIUS &gt; Live Sessions</code> |
| **WLC リアルタイムデバッグ (IOS-XE)** | <code>debug wireless mac &lt;MAC&gt;</code> / <code>trace on-demand wireless-system-trace</code> |
| **WLC RADIUS パケットデバッグ** | <code>debug aaa authentication</code> / <code>debug aaa authorization</code> |



## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **クライアントがポータル画面にリダイレクトされない** | WLC 側の Redirect ACL で DNS (UDP 53) または ISE Web ポータル (TCP 8443) が `deny` されていない、あるいは HTTPS トラフィックの捕捉が不完全 | <code>show wireless client mac-address &lt;MAC&gt; detail</code><br><code>show ip access-lists REDIRECT_GUEST_ACL</code> | Redirect ACL で DNS / ISE Port トラフィックを `deny` し、その他の HTTP/HTTPS を `permit` する。 |
| **ブラウザで HTTPS 警告 (NET::ERR_CERT_COMMON_NAME_INVALID) が表示される** | ISE ゲストポータル証明書が Private CA 発行であり、クライアント端末にルート証明書がインストールされていない | ISE GUI: <code>Administration &gt; System &gt; Certificates</code> | 信頼されたパブリック CA (Public CA) 発行の証明書を ISE ポータル用 (Portal Tag) にインポートする。 |
| **ポータルでログイン後もネットワークアクセスが許可されない (Webauth Pending から変わらない)** | ISE から WLC への RADIUS CoA (Reauth/Disconnect) が失敗している（UDP 1700/3799 到達不能または Shared Secret 不一致） | <code>show aaa dynamic-author server statistics</code><br>ISE Live Logs: <code>CoA Failure</code> | WLC 側で <code>aaa server radius dynamic-author</code> の IP・Shared Secret を確認し、ファイアウォールで UDP 1700 を開放する。 |
| **CWA リダイレクトのループが発生する** | ISE 認可ポリシーで、ポータル認証成功後のエンドポイントグループ (`GuestEndpoints`) への登録判定ルールが機能していない | ISE Live Logs: <code>Authorization Policy match</code> | 認可ポリシーの条件に <code>Endpoints:GuestEndpoints IN_GROUP</code> が正しく設定されているか確認する。 |
| **スポンサー承認メールが届かない** | ISE の SMTP サーバー設定エラー、またはスポンサーのメールアドレス記述ミス | ISE GUI: <code>Administration &gt; System &gt; Settings &gt; SMTP Server</code> | SMTP サーバーの IP/認証情報およびテストメール送信機能で到達性を確認する。 |

---

## ⚠ 制限事項

* **WLC と ISE 間の Redirect ACL 名の完全一致必須**:
  - ISE 認可プロファイルの `url-redirect-acl` に入力する文字列は、WLC 側で定義した IP Access List 名と大文字・小文字を含めて完全に一致しなければならない。
* **クライアントブラウザの HSTS / DoH 制限**:
  - HSTS (HTTP Strict Transport Security) を強制する外部 Web サイトを最初に開いた場合、証明書警告のバイパスが不可能となる。対策として未認証時は非 HSTS HTTP サイトの利用を推奨。
  - DNS over HTTPS (DoH) が有効なブラウザでは WLC の DNS インターセプトが機能せず、ポータル誘導が失敗する場合がある。
* **RADIUS CoA 到達性・ポート要件**:
  - NAT 越えの構成では RADIUS CoA パケットが WLC の外側 IP に到達しない場合があるため、直接的な L3 到達性を確保するか VPN / Dynamic Port マッピングが必要。

---

## 🔄 他技術との関連

* **FlexConnect Local Switching / Anchor WLC**:
  - FlexConnect 構成では、リダイレクト ACL が各アクセスポイント (AP) 側に分散定義・適用されるため、AP への ACL ディスパッチ設定が必須。
  - Foreign / Anchor WLC 構成では、Guest トラフィックは Anchor WLC で Egress されるため、CWA リダイレクトおよび RADIUS 認証要求は Anchor WLC から ISE に対して実行される。
* **Cisco TrustSec (SGT / SXP)**:
  - 認証完了後、ISE から WLC / スイッチに対して Security Group Tag (SGT: 例: `SGT 15 - Guest`) を動的割り当て可能。他部署ネットワーク（社内リソース）とのセグメンテーションを SGT ベースの SGACL / SFW で透過的に実現。
* **BYOD / Onboarding (ISE NSP)**:
  - セルフ登録ポータルから、社内端末としてのオンボーディング（証明書発行・プロファイル適用）へのシームレスな移行フローマッピングが可能。

---

## 🧩 比較表

### ゲストアクセス認証方式の比較

| 項目 | Hotspot (AUP Only) | Self-Registration Guest | Sponsored Guest |
| --- | --- | --- | --- |
| **認証強度** | 低 (誰でも接続可能) | 中 (メール/SMS本人確認) | 高 (社内社員による承認必須) |
| **アカウント管理** | 不要 (MACアドレス登録のみ) | 自動生成 (自己登録) | スポンサーが事前発行またはポータル承認 |
| **利用場面** | パブリックエリア、店舗Wi-Fi | 訪問者・セミナー参加者向けアクセス | 業務ベンダー・常駐協力会社員向けアクセス |
| **ISEの設定要素** | Hotspot Portal | Self-Reg Portal, Email/SMS Gateway | Sponsored Guest Portal, Sponsor Groups |
| **アカウント有効期間** | セッション単位 (数時間) | 1日〜数日 | 数日〜数ヶ月 (ポリシー依存) |

---

## 💡 ベストプラクティス

1. **ポータル専用公的証明書 (Public Certificate) の適用**:
   - ユーザー体験（UX）向上とセッション切断を防ぐため、ISE ゲストポータルには信頼された Public CA 発行証明書をバインドする。
2. **適切なキャプティブポータル検出 (Captive Portal Assist) の制御**:
   - OS の自動ポータル検知メカニズムが正常に動作するよう、ISE ポータル設定で CNA サポートを適切に有効化する。
3. **定期的な Endpoint / Guest Account パージの設定**:
   - 期限切れのゲストアカウントおよび Endpoints データベースを定期的に掃除（`Guest Purge Policy`）し、ISE 内部データベースの膨張を回避する。
4. **ゲスト分離用 VLAN / Direct Internet Access の確保**:
   - ゲスト Wi-Fi の VLAN トラフィックは社内 VLAN から隔離し、DMZ または Direct Internet Access (DIA) 回線へ直接ルーティングする構成を基本とする。



## 📝 ラボ学習・設定サンプル例

### サンプル例 1: CWA with Self-Registration on Catalyst 9800 WLC

#### 問題
Catalyst 9800 WLC および Cisco ISE を使用して、来訪者が自らアカウント登録（Self-Registration）できるゲスト Wi-Fi 環境を構築せよ。

#### 要件
1. SSID: `Guest-SelfReg`
2. 初回接続時は ISE の Self-Registration Portal へリダイレクトすること。
3. リダイレクト ACL 名: `ACL_CWA_REDIRECT`
4. ポータルで自己登録完了後、自動的にネットワークアクセスを許可し、速度制限用 dACL を適用すること。

#### 設定例

**Cat 9800 WLC Config:**
```text
ip access-list extended ACL_CWA_REDIRECT
 10 deny udp any any eq domain
 20 deny tcp any host 192.168.10.50 eq 8443
 30 deny tcp any host 192.168.10.50 eq 8905
 40 permit tcp any any eq www
 50 permit tcp any any eq 443
exit

aaa server radius dynamic-author
 client 192.168.10.50 server-key Cisco123!
 port 1700
exit

wlan GUEST_SELFREG 10 Guest-SelfReg
 no security wpa
 no security wpa wpa2
 no security wpa ft
 client association limit 0
 no shutdown
exit

wireless profile policy PROFILE_GUEST_SELFREG
 aaa-override
 nac
 aaa-policy default
 vlan 100
 central association
 central switching
 central dhcp
 parameter-map type webauth global
 no shutdown
exit

wireless tag policy TAG_GUEST_POLICY
 wlan GUEST_SELFREG policy PROFILE_GUEST_SELFREG
exit
```

**Cisco ISE Authorization Profile Config:**
- Name: `Authz_Guest_CWA`
- Access Type: `ACCESS_ACCEPT`
- Web Redirection: `Central Web Authentication`
- Redirect ACL: `ACL_CWA_REDIRECT`
- Value: `Self-Registration Portal`

#### 検証方法
1. クライアント端末から `Guest-SelfReg` SSID に接続する。
2. ブラウザを起動し、任意の HTTP サイトを開いて ISE の Self-Registration Portal に誘導されることを確認する。
3. アカウント登録およびログインを実行し、WLC 上で `show wireless client mac-address <MAC> detail` を実行。Client State が `RUN` に更新され、CoA が正常に受信用処理されたことを確認する。

---

### サンプル例 2: Hotspot Portal (AUP Accept Only) on Cat 9800 WLC

#### 問題
ログイン情報の入力なしで利用規約 (AUP) に同意するだけで利用可能な Hotspot ゲストアクセスを構築せよ。

#### 要件
1. SSID: `Guest-Hotspot`
2. ポータル種類: Hotspot Portal (AUP 画面のみ表示)
3. 同意ボタン押下後、セッションが確立し 4 時間有効とすること。

#### 設定例

**Cisco ISE Configuration:**
1. **Work Centers > Guest Access > Portals & Components > Hotspot Portals**:
   - `Hotspot Portal (default)` を編集。
   - AUP (Acceptable Use Policy) ページを有効化。
2. **Authorization Profile**:
   - Name: `Authz_Hotspot_Redirect`
   - Redirection: Central Web Authentication (`ACL_CWA_REDIRECT`, `Hotspot Portal`)
3. **Authorization Policy**:
   - Rule: `If Wireless_MAB AND Endpoints:GuestEndpoints NOT_IN_GROUP -> Authz_Hotspot_Redirect`
   - Rule: `If Wireless_MAB AND Endpoints:GuestEndpoints IN_GROUP -> PermitAccess`

#### 検証方法
1. クライアントが `Guest-Hotspot` SSID に接続。
2. AUP 規約同意画面が表示され、「Accept」をクリック。
3. ISE Live Logs で CoA Success および `PermitAccess` ポリシーへの遷移を確認。

---

### サンプル例 3: Sponsored Guest Access with Sponsor Portal Approval

#### 問題
外部ベンダー接続用に社内社員（Sponsor）による承認が必要な Sponsored Guest アクセスを構築せよ。

#### 要件
1. ゲストが Self-Registration 画面で社内社員の Email アドレスを入力。
2. スポンサー（社員）宛てに承認依頼メールが届き、Sponsor Portal より承認ボタンを押すことでアカウントが有効化されること。

#### 設定例

**Cisco ISE Configuration:**
1. **Sponsor Groups**:
   - `ALL_EMPLOYEES` グループに Sponsor 権限を割り当て。
2. **Self-Registration Portal Settings**:
   - `Require guests to be approved` を有効化。
   - `Sponsor Email List` 条件を設定。
3. **Authorization Policy**:
   - ゲスト状態が `Pending Approval` の間はネットワークアクセスを遮断。承認完了（`Active`）時に CoA を発行して接続許可。

#### 検証方法
1. ゲストが接続して登録申請を提出。ポータルに「承認待ち」画面が表示されることを確認。
2. スポンサーが `https://ise.example.com:8443/sponsorportal/` にログインし、該当ゲストの申請を「Approve」。
3. ゲスト端末の画面が更新され、インターネット接続が確立されることを検証。

---

### サンプル例 4: AireOS WLC Central Web Authentication (CWA) Configuration

#### 問題
AireOS WLC (8.5/8.10) 環境において、ISE CWA リダイレクトを設定せよ。

#### 要件
1. WLAN ID 3: `Guest-AireOS`
2. Layer 2 Security: None, Layer 3 Security: None (ISE CWA)
3. RADIUS NAC および AAA Override を有効化すること。

#### 設定例

**AireOS CLI Config:**
```text
config wlan create 3 Guest-AireOS Guest-AireOS
config wlan security parameter-map 3 default
config wlan aaa-override enable 3
config wlan nac-state enable 3
config wlan radius-fallback-mode Dynamic 3
config wlan security wpa disable 3
config wlan security web-auth disable 3
config wlan enable 3

config acl create REDIRECT_GUEST_ACL
config acl rule add REDIRECT_GUEST_ACL 1
config acl rule action REDIRECT_GUEST_ACL 1 permit
config acl rule protocol REDIRECT_GUEST_ACL 1 udp
config acl rule destination port range REDIRECT_GUEST_ACL 1 53 53
config acl rule apply REDIRECT_GUEST_ACL

config acl rule add REDIRECT_GUEST_ACL 2
config acl rule action REDIRECT_GUEST_ACL 2 deny
config acl rule protocol REDIRECT_GUEST_ACL 2 tcp
config acl rule destination port range REDIRECT_GUEST_ACL 2 8443 8443
config acl rule apply REDIRECT_GUEST_ACL
```

#### 検証方法
1. `show wlan 3` で AAA Override および RADIUS NAC が `Enabled` であることを確認。
2. クライアント接続時に `show client detail <MAC>` で `WebAuth Pending: Yes` および URL Redirect ACL が適用されていることを検証。

---

### サンプル例 5: FlexConnect Local Switching with CWA on Cat 9800 WLC

#### 問題
拠点の FlexConnect AP において、データ通信をローカルブレイクアウト (Local Switching) しつつ ISE CWA 認証を実行せよ。

#### 要件
1. Policy Profile で `Central Switching` を無効化し `FlexConnect` を有効化。
2. Webauth Redirect ACL を FlexConnect AP 側に配信・分散適用すること。

#### 設定例

**Cat 9800 WLC Config:**
```text
wireless profile policy FLEX_GUEST_POLICY
 no central switching
 no central dhcp
 aaa-override
 nac
 vlan 100
 parameter-map type webauth global
 no shutdown
exit

wireless profile flex FLEX_PROFILE_BRANCH
 acl-map REDIRECT_GUEST_ACL
exit
```

#### 検証方法
1. FlexConnect AP 配下のクライアントから接続。
2. AP の CLI で `show controllers dot11Radio 0` または AP 内部 ACL テーブルを確認し、リダイレクト ACL が AP ローカルで透過処理されていることを検証。



### サンプル例 6: Foreign / Anchor WLC Architecture for Guest CWA

#### 問題
DMZ に配置された Anchor WLC と内部網の Foreign WLC を使用し、ゲストトラフィックを DMZ へカプセル化（EoIP / CAPWAP Tunnel）して排出しつつ CWA 認証を行え。

#### 要件
1. Foreign WLC と Anchor WLC 間に Guest Mobility Anchor を構成。
2. RADIUS CWA 処理およびアンカーからの直接インターネット排出の実現。

#### 設定例

**Foreign WLC Config:**
```text
wireless mobility group member ip 192.168.200.10 public-ip 192.168.200.10 group DEFAULT_GROUP
wlan GUEST_ANCHOR 15 Guest-Anchor
 mobility anchor 192.168.200.10
exit
```

**Anchor WLC Config:**
```text
wireless mobility group member ip 192.168.100.10 public-ip 192.168.100.10 group DEFAULT_GROUP
wlan GUEST_ANCHOR 15 Guest-Anchor
 client association limit 0
 no shutdown
exit

wireless profile policy ANCHOR_GUEST_POLICY
 aaa-override
 nac
 vlan 999
 central switching
 central dhcp
 parameter-map type webauth global
 no shutdown
exit
```

#### 検証方法
1. `show wireless mobility summary` で Foreign - Anchor 間の Mobility Tunnel パスが `Up` であることを確認。
2. ゲスト接続時、Anchor WLC 上で RADIUS Access-Request が生成され ISE へ送信されることを確認。

---

### サンプル例 7: Dynamic VLAN Assignment & dACL for Authenticated Guests

#### 問題
ポータル認証成功後、ゲスト端末に対してアクセス許可 VLAN (VLAN 200) および通信宛先を限定する dACL を適用せよ。

#### 要件
1. dACL 名: `dACL_GUEST_RESTRICTED`
2. 許可範囲: Web 閲覧 (HTTP/HTTPS) および DNS のみ。社内サブネット (10.0.0.0/8) へのアクセスは遮断。

#### 設定例

**Cisco ISE Authorization Profile Config:**
- Name: `Authz_Guest_Restricted`
- Access Type: `ACCESS_ACCEPT`
- Dynamic VLAN: `200`
- Downloadable ACL (dACL):
```text
permit udp any any eq domain
deny ip any 10.0.0.0 0.255.255.255
permit tcp any any eq www
permit tcp any any eq 443
deny ip any any
```

#### 検証方法
1. ゲストログイン成功後、WLC 上で `show wireless client mac-address <MAC> detail` を実行。
2. Assigned VLAN ID が `200` に変化し、Apply dACL に `dACL_GUEST_RESTRICTED` が表示されることを確認。

---

### サンプル例 8: RADIUS CoA (Reauth/Port Bounce) for Guest Session State Transition

#### 問題
ISE 上でゲストのセッション状態が変化した際（例: ログイン成功、アカウント即時停止）、WLC に対して RADIUS CoA Reauth / Disconnect を正常に処理させる設定を行え。

#### 要件
1. WLC 側で UDP 1700 での Dynamic Authorization 応答を許可。
2. CoA パケット受信時の統計ログを生成すること。

#### 設定例

**Cat 9800 WLC Config:**
```text
aaa server radius dynamic-author
 client 192.168.10.50 server-key Cisco123!
 port 1700
 auth-type any
exit
```

#### 検証方法
1. ISE の Live Sessions 画面から該当ゲストセッションを選択し、「Session Actions > Reauthenticate > Push」を実行。
2. WLC CLI で `show aaa dynamic-author server statistics` を確認し、`CoA-ACK` カウントがインクリメントされることを検証。

---

### サンプル例 9: Cisco ISE Guest Account Lifetime & Purge Policy Setup

#### 問題
ゲストアカウントの有効期限（例: 24時間）およびデータベース自動パージ（Purge Policy）を構成せよ。

#### 要件
1. 有効期限が切れたアカウントは即時無効化。
2. 7日経過した不活性ゲスト端末 MAC アドレスを ISE Endpoint DB から自動削除すること。

#### 設定例

**Cisco ISE Configuration:**
1. **Work Centers > Guest Access > Settings > Guest Account Purge Policy**:
   - `Purge Guest Accounts` を有効化。
   - 条件: `Time Since Account Expiration > 7 Days`
2. **Work Centers > Guest Access > Settings > Endpoint Purge Policy**:
   - Condition: `IdentityGroup EQUALS GuestEndpoints AND ElapsedDays > 7`
   - Action: `Purge`

#### 検証方法
1. テストアカウントを作成し、有効期限を 5 分後に設定。
2. 5 分経過後、ISE `Guest Access > Manage Accounts` で該当アカウントが `Expired` に変化し、再接続時にポータルへと再誘導されることを確認。

---

### サンプル例 10: Wired Guest CWA Access on Cisco Catalyst Switch (IOS-XE IBNS 2.0)

#### 問題
Cisco Catalyst スイッチの有線ポートにおいて、CWA ゲストアクセスを IBNS 2.0 (C3PL) 構文を用いて設定せよ。

#### 要件
1. ポートで MAB を優先実行し、ISE CWA リダイレクトを受け取ること。
2. RADIUS CoA によりポート認証状態を更新すること。

#### 設定例

**Catalyst Switch (IOS-XE) Config:**
```text
aaa new-model
aaa authentication dot1x default group RADIUS_ISE
aaa authorization network default group RADIUS_ISE
aaa accounting identity default start-stop group RADIUS_ISE

aaa server radius dynamic-author
 client 192.168.10.50 server-key Cisco123!
exit

ip access-list extended REDIRECT_GUEST_ACL
 10 deny udp any any eq domain
 20 deny tcp any host 192.168.10.50 eq 8443
 30 permit tcp any any eq www
 40 permit tcp any any eq 443
exit

class-map type control subscriber match-all CWA_REDIRECT_CLASS
 match authorization-status unauthorized
exit

policy-map type control subscriber WIRED_GUEST_POLICY
 event session-started
  10 authorize
  20 authenticate using mab
 event authentication-failure
  10 restart mab
exit

interface GigabitEthernet1/0/10
 switchport mode access
 switchport access vlan 100
 access-session port-control auto
 service-policy type control subscriber WIRED_GUEST_POLICY
ip access-group PERMIT_ALL in
exit
```

#### 検証方法
1. 有線端末を GigabitEthernet1/0/10 に接続。
2. `show access-session interface GigabitEthernet1/0/10 detail` を実行し、Status が `Authorized` かつ URL Redirect ACL 名が表示されることを確認。
3. ポータルログイン後、CoA により dACL または VLAN が更新されることを確認。



## ❓ 想定試験問題

### 問題 1 (コンフィグ読解)
ある Catalyst 9800 WLC において、ゲスト用 Redirect ACL を以下のように設定したが、ゲスト端末が ISE の CWA ポータル画面に一切リダイレクトされず、タイムアウトエラーとなった。原因として最も可能性が高いものを選べ。

```text
ip access-list extended REDIRECT_GUEST_ACL
 10 permit udp any any eq domain
 20 permit tcp any host 192.168.10.50 eq 8443
 30 deny tcp any any eq www
 40 deny tcp any any eq 443
```

- A. DNS トラフィックが permit されているため、DNS 解決が阻害されている。
- B. リダイレクト ACL のロジックが逆転しており、HTTP/HTTPS (www/443) トラフィックを deny しているため捕捉されない。
- C. ISE の IP アドレスへの通信が許可されているため、WLC がリダイレクトを停止した。
- D. ACL 構文に `ip classless` が含まれていない。

**正解**: B
**解説**: WLC の Redirect ACL では、「リダイレクトさせたくない通信（DNS や ISE 自身への通信）を deny」「リダイレクト対象とする Web 通信を permit」と定義する。本コンフィグでは www/443 が `deny` されているため、WLC は Web トラフィックをスルー（またはドロップ）し、ポータルへの 302 リダイレクトを生成できない。

---

### 問題 2 (トラブルシューティング)
ISE でゲストポータル認証（Self-Registration）が成功し、画面上には「ログイン完了」と表示されたが、ゲスト端末のネットワーク通信が引き続き遮断されたままである。ISE Live Logs を確認すると、`CoA-Request Failed` エラーが記録されていた。この障害の根本原因を特定するための確認手順として適切なものはどれか。

- A. WLC の `ip access-list` 設定で `permit ip any any` を追加する。
- B. WLC CLI で `show aaa dynamic-author server statistics` を実行し、CoA 受信ポート (UDP 1700) および Dynamic Authorization クライアント IP / Key 設定の一致を確認する。
- C. ISE の DHCP Server 設定で Gateway IP を再割り当てする。
- D. WLC の WLAN 設定で WPA2-Enterprise (802.1X) を有効化する。

**正解**: B
**解説**: ゲストポータルログイン成功後、アクセス許可状態へ移行するには ISE から WLC への RADIUS CoA パケットの正常受信が必須である。CoA が失敗している場合、WLC 側の `aaa server radius dynamic-author` 設定（受信 IP, Port 1700, Shared Secret）または経路上のファイアウォールによる UDP 1700 ブロックが原因である。

---

### 問題 3 (Design)
大規模企業のマルチ拠点（本社＋50拠点）において、ゲスト アクセス用 SSID を展開する設計を行っている。本社に Anchor WLC を配置し、拠点には FlexConnect AP を配置する構成において、WAN 帯域の圧迫を回避しつつ、セキュリティを確保する最も推奨される構成はどれか。

- A. 拠点の FlexConnect AP で Central Switching と Central Authentication を強制し、すべてのトラフィックを本社の ISE および Anchor WLC へ集約する。
- B. 拠点の FlexConnect AP で FlexConnect Local Switching を使用し、CWA リダイレクト ACL を AP に配信・ローカルブレイクアウト (DIA) させ、認証処理のみ本社の ISE と連携させる。
- C. 各拠点に個別の Cisco ISE パノラマアプライアンスをローカル配置し、スタンドアロン運用する。
- D. ゲスト SSID の認証を 802.1X EAP-TLS に限定し、ポータルリダイレクトを無効化する。

**正解**: B
**解説**: FlexConnect Local Switching と CWA を組み合わせることで、データ通信（インターネットアクセス）は拠点から直接排出 (DIA) して WAN 帯域を保護しつつ、認証およびアクセス制御ポリシーの集中管理を本社の ISE で実現できる。

---

### 問題 4 (実装・設定)
Catalyst 9800 WLC で CWA ゲストアクセスを有効化する際、Policy Profile 内で絶対に有効化（Enable）しなければならない 2 つの設定項目を選べ。

- A. `aaa-override`
- B. `passive-client`
- C. `nac`
- D. `ip dhcp snooping`

**正解**: A, C
**解説**: RADIUS サーバー (ISE) から返却される Redirect URL / ACL 属性を反映するために `aaa-override` が必須であり、認可更新時の RADIUS CoA (Reauth) に対応するために `nac` (RADIUS NAC) が必須である。

---

### 問題 5 (コンフィグ・要件確認)
以下の ISE 認可ルールにおいて、未認証ゲストが無限リダイレクトループに陥っている。この問題を解消するための適切な設定変更はどれか。

* Rule 1: `If Wireless_MAB -> Result: Authz_Guest_Redirect`
* Rule 2: `If Wireless_MAB AND Endpoints:GuestEndpoints IN_GROUP -> Result: PermitAccess`

- A. Rule 1 と Rule 2 の評価順序を逆転させ、Rule 2 を上位に配置する。
- B. Rule 1 の条件に `Endpoints:GuestEndpoints NOT_IN_GROUP` を追加する。
- C. RADIUS CoA のタイプを Disconnect から Reauth に変更する。
- D. WLC 側の Redirect ACL を削除する。

**正解**: A (または B)
**解説**: ISE ポリシーセットは上から順に評価される。Rule 1 が全 Wireless_MAB にマッチする状態のままだと、ポータル登録完了後（`GuestEndpoints` に登録された後）の再認証でも Rule 1 にマッチしてしまい、再び `Authz_Guest_Redirect` が返却されて無限ループとなる。評価順序を逆転させて登録済みチェック（Rule 2）を先に評価させるか、Rule 1 に NOT_IN_GROUP 条件を追加する必要がある。

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術ガイド一覧：

* Cisco Identity Services Engine Admin Guide - Guest Access
* Configure Central Web Authentication (CWA) on Catalyst 9800 WLC and ISE
* Cisco ISE Guest Access Deployment Guide
* AireOS WLC Central Web Authentication with ISE Configuration Example
* Cisco Catalyst 9800 Series Wireless Controller Configuration Guide - Web Authentication
* Cisco Live BRKSEC-2040 - Advanced Guest and BYOD Architectures with Cisco ISE
* Cisco Live BRKSEC-3699 - Troubleshooting Cisco ISE Guest & Web Authentication

---

## 📝 **補足（Notes）**

* **CWA リダイレクトのデバッグコマンドまとめ (WLC / ISE)**:
  - Cat 9800 WLC CLI:
    ```text
    debug wireless mac <Client_MAC>
    show logging profile wireless filter mac <Client_MAC> to-file bootflash:guest_debug.txt
    ```
  - Cisco ISE: `Operations > Troubleshooting > Diagnostic Tools > RADIUS Live Logs` から Details を開き、評価された Policy Set, Authentication Policy, Authorization Policy 名を確認。
* **図解: CWA リダイレクト ACL 評価ロジック**:
  - `deny udp any any eq domain`: DNS はリダイレクトせず、通常通りプロキシ/解決させる。
  - `deny tcp any host <ISE_IP> eq 8443`: ISE ポータル自体への通信はリダイレクトさせず、そのままポータル Web ページを表示させる。
  - `permit tcp any any eq www`: 上記以外のすべての HTTP 通信を捕獲し、ISE ポータル URL へ 302 リダイレクト応答を返す。



