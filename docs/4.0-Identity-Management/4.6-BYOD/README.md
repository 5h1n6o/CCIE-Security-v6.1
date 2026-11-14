---
layout: default
title: 4.6-BYOD
nav_order: 6
parent: 4.0-Identity-Management
---

# 4.6 BYOD on-boarding and network access flows

## 📘 概要

Cisco Identity Services Engine (ISE) における **BYOD (Bring Your Own Device) オンボーディングおよびネットワークアクセスフロー** は、従業員やユーザーが所有する個人のスマートフォン、タブレット、PC などのデバイスを企業ネットワークへ安全に接続・登録するための包括的なフレームワークです。

* **機能概要**: 未認証・未登録の個人デバイスに対し、Web ポータル（CWA: Central Web Authentication）を介してユーザー認証を行い、デバイス固有のデジタル証明書（X.509v3）の自動発行・インストール、および Network Setup Assistant (NSA) や Native Supplicant Profile (NSP) を通じた EAP-TLS 接続設定の自動構成を行います。
* **利用目的**: ユーザー自身にデバイス登録（Self-Service Onboarding）を行わせることで IT ヘルプデスクの負荷を激減させつつ、事前共有鍵 (PSK) やパスワードベース認証（PEAP-MSCHAPv2）から、盗聴や辞書攻撃に強い証明書ベース認証（EAP-TLS）へ安全に移行・昇格させます。
* **どのような場面で利用するか**: 企業や教育機関において、社有端末以外の多様な OS（iOS, Android, Windows, macOS）を社内 VLAN やセキュアリソース（社内 Web, DB, SGT アクセス制御）へ接続させる場面で導入されます。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **特徴** | ユーザーセルフサービスによる自動証明書発行（SCEP/EST）、Native Supplicant Profile (NSP) 自動適用、MAC アドレスとユーザーのバインディング（`RegisteredDevices` グループ追加）、ポータルベースのフロー遷移 |
| **用途** | 個人所有端末 (BYOD) の EAP-TLS ネットワーク認証への安全な自動登録、証明書管理、アクセス権限昇格 |
| **メリット** | ヘルプデスクの手動設定コスト削減、パスワード漏洩リスクの排除、デバイス固有証明書による厳格な個体識別とアクセス制御 |
| **デメリット** | 初期登録時のリダイレクト・証明書受入などユーザー操作が発生、OS バージョン（Android/iOS/Windows）ごとのサプリカント動作差異や証明書インストール制限 |
| **対応機種** | Cisco ISE (v2.x / v3.x), Cisco Catalyst スイッチ (IOS-XE), Cisco WLC (AireOS / Catalyst 9800), Cisco ASA / FTD (RA-VPN BYOD) |
| **制限事項** | モバイル OS のプライバシー保護機能（Randomized MAC Address / Private Wi-Fi Address）による MAC 追跡の妨害、一部 OS での外部 CA / SCEP 制限 |
| **設計上の注意点** | Single-SSID フローと Dual-SSID フローの選択、DHCP Option 43 / DNS リダイレクトの整合性、証明書有効期限および SAN (Subject Alternative Name) 拡張属性の設計 |

---

## 🏗 動作原理

Cisco ISE BYOD オンボーディングは、デバイスの登録状態および認証方式に応じてアクセス権限を動的に変更する **マルチステップ・ポリシー制御** に基づいて動作します。代表的なアプローチとして **Single-SSID フロー** と **Dual-SSID フロー** の 2 種類が存在します。

```
[ Unregistered Device ]
       │
       ▼ (802.1X PEAP / MAB)
[ Access-Accept with CWA Redirect ]
       │
       ▼ (HTTP Request intercept)
[ BYOD Portal / User Authentication ]
       │
       ▼ (Download NSP & NSA App)
[ Certificate Enrollment via SCEP/EST ]
       │
       ▼ (Install Cert & EAP-TLS Profile)
[ RADIUS CoA (Re-authenticate) ]
       │
       ▼ (802.1X EAP-TLS)
[ Full Network Access (Corporate VLAN/SGT) ]
```

1. **Single-SSID フロー**: 単一の SSID (例: `Corporate-BYOD`) で初期接続（PEAP-MSCHAPv2 または MAB）を行い、CWA リダイレクトにより BYOD ポータルへ誘導。証明書発行完了後、RADIUS CoA (Change of Authorization) を発行して同じ SSID のまま EAP-TLS へ透過的にセッションを再認証・昇格させます。
2. **Dual-SSID フロー**: オープンまたは PSK ネットワーク (例: `Guest-Onboarding`) で接続してオンボーディングポータルへ誘導し、証明書と EAP-TLS 設定を注入した後、ユーザーに暗号化されたセキュア SSID (例: `Secure-Corporate`) へ切り替えさせます。

---

## ⚙ 動作シーケンス

パケット処理およびセッション遷移の具体的なシーケンスは以下の通りです。

1. **初期接続とポータルリダイレクト (Initial Association & CWA)**
   * クライアントが WLC / スイッチへ接続（MAB または 802.1X PEAP）。
   * ISE は Authentication Policy および Authorization Policy により、未登録端末 (`BYODnotRegistered`) に対して **CWA リダイレクト URL** (`https://<ISE-PSN>:8443/portal/g?p=...`) と **Redirect ACL** をレスポンス（RADIUS Access-Accept）として返却します。
2. **Web リダイレクトとポータル認証 (Web Redirection & Authentication)**
   * クライアントが HTTP/HTTPS ブラウザリクエストを発行すると、スイッチ / WLC の Redirect ACL により TCP 80/443 パケットがトラップされ、ISE ポータルへ HTTP 302 リダイレクトされます。
   * ユーザーは BYOD ポータルで Active Directory / LDAP 資格情報を入力して認証されます。
3. **デバイス登録と NSP ダウンロード (Device Registration & Profile Download)**
   * 認証成功後、ISE はデバイスの MAC アドレスを `RegisteredDevices` Identity Group に追加します。
   * ISE はクライアント OS（Windows, macOS, iOS, Android）を判定し、対応する **Native Supplicant Profile (NSP)** または **Network Setup Assistant (NSA)** アプリケーション（または iOS 設定プロファイル）を提供します。
4. **SCEP/EST 証明書プロビジョニング (Certificate Enrollment)**
   * NSP / NSA は ISE の内部 CA (ISE Local CA) または外部 PKI (Microsoft CA 等) に対して SCEP (Simple Certificate Enrollment Protocol) または EST (Enrollment over Secure Transport) を使用して PKCS#10 CSR (Certificate Signing Request) を送信します。
   * ISE は CSR を検証・署名し、デバイス固有のクライアント証明書（Subject: `CN=Username`, SAN: `MAC=xx-xx-xx-xx-xx-xx`）を発行・インストールします。
5. **CoA とセキュアアクセスへの遷移 (RADIUS CoA & EAP-TLS Access)**
   * 証明書インストール後、ISE はネットワーク機器（WLC / スイッチ）に対して **RADIUS CoA (Re-authenticate / Port Bounce)** を送信します。
   * クライアントは自動的に EAP-TLS 認証を開始し、インストールされたクライアント証明書を提示します。
   * ISE は EAP-TLS 認証を成功させ、社内アクセス用 Authorization Profile（正当な VLAN, dACL, SGT `BYOD_Secured` 等）を付与します。

---


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、BYOD オンボーディングは **Cisco ISE, WLC (Catalyst 9800/AireOS), Catalyst Switch** に跨る最もトラブルが発生しやすい総合評価セクションの一つです。

### Blueprint で重要なポイント
* **Single-SSID vs Dual-SSID フローの選択と実装**: 要件仕様（「SSID を追加作成せずに同一 SSID で昇格させること」か「Onboarding 専用 SSID から Secure SSID へ切り替えさせること」か）を正確に読み取ること。
* **NSP (Native Supplicant Profile) の構成**: EAP-TLS 用のキーサイズ (2048-bit)、Key Usage、内部 CA / 外部 CA のバインド、および SAN (Subject Alternative Name) への MAC アドレス埋め込み要件。
* **Client Certificate Template と SCEP/EST 連携**: ISE Internal CA または外部 SCEP RA (Registration Authority) ポートの設定と証明書チェーンの信頼関係構築。
* **ポータルリダイレクト ACL の定義**: DNS (UDP 53), DHCP (UDP 67/68), ISE PSN IP (TCP 8443/8905/8909) をバイパスし、その他の TCP 80/443 をトラップする ACL の構文。

### ラボ試験で設定させられそうな内容
1. **ISE 上での Native Supplicant Profile (NSP) 設定**: Wireless/Wired EAP-TLS プロファイル作成、SSID 名設定、SCEP/EST CA テンプレートのバインド。
2. **BYOD Client Certificate Template 設定**: Subject / SAN フィールドのカスタムマッピング (`CN=$user$`, `SAN:otherName=1.3.6.1.4.1.311.20.2.3;utf8:$user$@domain` や `MAC=$mac$`)。
3. **Authorization Policy の階層構築**:
   * **Rule 1 (BYOD Registered & EAP-TLS)**: `Network Access:AuthenticationMethod EQUALS EAP-TLS` AND `IdentityGroup EQUALS RegisteredDevices` ➔ **PermitAccess / Corporate_SGT**
   * **Rule 2 (BYOD Redirect Flow)**: `User-Identity-Group EQUALS Employee` AND `EndPoints:BYODRegistration EQUALS Unknown/NotRegistered` ➔ **BYOD_Portal_Redirect**
4. **WLC / Switch リダイレクト ACL コンフィグ**: Catalyst 9800 / AireOS での RADIUS Change of Authorization (CoA) サポート (`aaa server radius dynamic-author`) と Redirect ACL 設定。

### よくある設定ミス・落とし穴
* **CoA ポートおよび Secret の不一致**: ネットワーク機器側で `rfc3576` や `dynamic-author` の `server-key` が ISE の RADIUS 共有鍵と一致していない場合、証明書取得後に CoA が失敗し、セッションがリダイレクト状態のままスタックする。
* **Redirect ACL で ISE 通信を暗黙の DENY (バイパス) していない**: Redirect ACL で ISE 宛ての TCP 8443, 8905, 8909 (NSA/NSP 通信ポート) を `deny` (リダイレクト除外) していないため、リダイレクトループが発生する。
* **証明書信頼チェーンの不足**: ISE の Portal Authentication Certificate または Internal CA Sub-CA 証明書がクライアント端末（特に Android/iOS）に信頼されていないため、HTTPS 警告や SCEP 発行失敗が発生する。
* **Android/iOS の Private Wi-Fi MAC アドレス問題**: ランダム MAC アドレスが有効な場合、初期接続時と証明書取得後の再接続時で MAC アドレスが変化し、`RegisteredDevices` グループとマッチしなくなる。

---

## 🛠 設定方法

### 1. Cisco ISE 上での BYOD プロビジョニング設定

#### Step 1: SCEP / Certificate Template の構成
1. **Administration > Identity Management > Settings > PKI > Certificate Templates** に移動。
2. `BYOD_Client_Template` を作成。
   * **Key Size**: `2048`
   * **Subject Alternative Name (SAN)**: `Directory Address (emailAddress)` または `DNS Name` に `$user$` / `$mac$` を配置。
   * **Validity Period**: `365 Days`

#### Step 2: Native Supplicant Profile (NSP) の作成
1. **Work Centers > BYOD > Native Supplicant Profiles** に移動し、新規追加。
2. **Operating System**: `Windows / macOS / iOS / Android` を選択。
3. **Wireless Connection**:
   * **SSID**: `Corporate-BYOD` (Single-SSID の場合) または `Secure-Corporate` (Dual-SSID の場合)
   * **Allowed Protocol**: `EAP-TLS`
   * **Certificate Template**: `BYOD_Client_Template`

#### Step 3: Client Provisioning Policy の作成
1. **Policy > Client Provisioning** に移動。
2. OS ごとにルールを追加:
   * **Rule 1 (iOS/Android)**: `Operating System EQUALS Android / iOS` ➔ Profile: `BYOD_NSP_Profile`
   * **Rule 2 (Windows/macOS)**: `Operating System EQUALS Windows All` ➔ Profile: `BYOD_NSP_Profile` (NSA Wizard)

#### Step 4: Authorization Policy の作成
1. **Policy > Policy Sets > Default** に移動。
2. **Authentication Policy**: `Dot1x` または `MAB` を配置。
3. **Authorization Policy**:
   * **Rule 1 (BYOD_Secured)**:
     * Condition: `Network Access:AuthenticationMethod EQUALS EAP-TLS` AND `IdentityGroup EQUALS RegisteredDevices`
     * Profile: `PermitAccess` (VLAN 10, dACL `PERMIT_ALL`, SGT `BYOD_Employees`)
   * **Rule 2 (BYOD_Onboarding_Redirect)**:
     * Condition: `Network Access:UseCase EQUALS Wireless Access` AND `BYODnotRegistered EQUALS true`
     * Profile: `BYOD_Redirect_Profile` (Web Redirection: `BYOD Portal`, ACL: `ACL_BYOD_REDIRECT`)

---

### 2. Cisco Catalyst 9800 WLC 設定例

```text
! RADIUS & Dynamic Authorization (CoA) 設定
aaa new-model
aaa authentication login default group radius-group
aaa authorization network default group radius-group
aaa accounting identity default start-stop group radius-group

radius server ISE_PSN_1
 address ipv4 192.168.10.50 auth-port 1812 acct-port 1813
 key Cisco123!

aaa group server radius radius-group
 server name ISE_PSN_1

aaa server radius dynamic-author
 client 192.168.10.50 server-key Cisco123!

! リダイレクト ACL の定義 (トラップしない通信を deny、その他 HTTP/HTTPS を permit)
ip access-list extended ACL_BYOD_REDIRECT
 deny udp any any eq domain
 deny udp any eq domain any
 deny udp any any eq bootps
 deny udp any any eq bootpc
 deny ip any host 192.168.10.50
 permit tcp any any eq www
 permit tcp any any eq 443

! WLAN および Policy Profile の設定
wlan Corporate-BYOD 1 Corporate-BYOD
 client association limit 0

wireless profile policy BYOD_Policy_Profile
 aaa-override
 nac
 radius-profiling dhcp
 vlan 100
 no shutdown

wlan profile-name Corporate-BYOD policy BYOD_Policy_Profile
```

---

### 3. Cisco Catalyst Switch (IOS-XE IBNS 2.0) 設定例

```text
! AAA & RADIUS CoA 設定
aaa new-model
aaa authentication dot1x default group ISE_RADIUS
aaa authorization network default group ISE_RADIUS
aaa accounting dot1x default start-stop group ISE_RADIUS

aaa server radius dynamic-author
 client 192.168.10.50 server-key Cisco123!

radius server ISE_PSN_1
 address ipv4 192.168.10.50 auth-port 1812 acct-port 1813
 key Cisco123!

aaa group server radius ISE_RADIUS
 server name ISE_PSN_1

! Switch Redirect ACL
ip access-list extended ACL_BYOD_REDIRECT
 deny udp any any eq domain
 deny udp any any eq bootps
 deny udp any any eq bootpc
 deny ip any host 192.168.10.50
 permit tcp any any eq www
 permit tcp any any eq 443

! IBNS 2.0 (C3PL) 認証ポリシー
class-map match-all C_DOT1X
 match authentication-method dot1x

class-map match-all C_MAB
 match authentication-method mab

policy-map PM_PORT_AUTH
 class C_DOT1X
  authenticate
 class C_MAB
  authenticate

! インターフェース適用
interface GigabitEthernet1/0/11
 switchport mode access
 switchport access vlan 100
 authentication periodic
 authentication timer reauthenticate server
 access-session port-control auto
 service-policy input PM_PORT_AUTH
```

---

## 🔍 検証コマンド

### スイッチ / WLC 検証コマンド表

| 目的 | コマンド |
| --- | --- |
| 認証セッションおよび リダイレクト状態の確認 | <code>show access-session interface GigabitEthernet1/0/11 details</code> |
| WLC クライアント認証・リダイレクト状態確認 | <code>show wireless client mac-address xxxx.xxxx.xxxx detail</code> |
| Dynamic ACL / Redirect ACL 適用状態の確認 | <code>show ip access-lists ACL_BYOD_REDIRECT</code> |
| CoA パケット受信および処理状態の確認 | <code>show radius statistics</code> |
| 認証イベントログのリアルタイム追跡 | <code>debug access-session status</code> |
| EAPOL および 802.1X パケットデバッグ | <code>debug dot1x all</code> |
| RADIUS リクエスト / CoA レスポンスのデバッグ | <code>debug radius all</code> |

### Cisco ISE ログ・ダッシュボード確認箇所
* **Operations > RADIUS > Live Logs**: 認証試行ログ、Authentication Policy / Authorization Policy のマッチング結果、付与された Authorization Profile (Redirect URL, Redirect ACL) の確認。
* **Operations > BYOD > Live Logs / BYOD Monitoring**: SCEP 証明書申請ログ、NSP ダウンロード状態、Device Registration (`RegisteredDevices` 追加) のステータス確認。

---


## 🚨 トラブルシュート

BYOD オンボーディングに関する代表的なトラブル症例、根本原因、確認コマンド、および対処方法を下表に整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **Web ポータルへリダイレクトされない** | WLC / スイッチの Redirect ACL で HTTP (80) / HTTPS (443) が Permit されていない、または ISE 宛て通信が Deny されていない。 | <code>show ip access-lists ACL_BYOD_REDIRECT</code>, <code>show access-session interface G1/0/11 details</code> | Redirect ACL の条件を見直し、ISE IP 宛てを `deny`、TCP 80/443 を `permit` に設定する。 |
| **ポータル表示時に HTTPS 警告が表示され接続が中断する** | ISE の Portal Authentication Certificate がパブリック CA 署名でない、またはルート CA 証明書が端末にインストールされていない。 | ISE: Admin > System > Certificates, Client: ブラウザ証明書詳細 | ISE ポータル証明書に信頼された CA 署名証明書を適用するか、手動で ルート CA を端末に配布する。 |
| **証明書取得後にセッションが EAP-TLS へ切り替わらない** | RADIUS CoA がネットワーク機器（WLC / スイッチ）に到達していない、または CoA 共有鍵が不一致。 | <code>show radius statistics</code>, ISE Live Logs: CoA Failure | WLC/スイッチで dynamic-author と server-key を確認し、UDP 1700/3799 通信を許可する。 |
| **SCEP 証明書発行エラー** | ISE SCEP RA プロファイル設定ミス、または Microsoft CA (NDES) との通信不可。 | ISE: BYOD Live Logs, ISE: prt-server.log | ISE SCEP 設定、NDES / SCEP URL、および CA サービス状態を確認する。 |
| **Android 端末で NSP アプリが取得できない** | Android 端末がアプリストアや外部パスにアクセスできない（Redirect ACL でブロックされている）。 | WLC Redirect ACL, 端末ブラウザログ | Redirect ACL で Android 用プロビジョニング URL を `deny` に追加する。 |
| **証明書適用後にアクセス拒否される** | Authorization Policy で RegisteredDevices または EAP-TLS の条件が正しく定義されていない。 | ISE Operations > RADIUS > Live Logs | Authorization Policy のマッチング条件に AuthenticationMethod EQUALS EAP-TLS と IdentityGroup EQUALS RegisteredDevices を正しく指定する。 |

---


## ⚠ 制限事項

* **ランダム MAC アドレス (Private Wi-Fi Address)**: iOS 14 以降および Android 10 以降では、Wi-Fi 接続時にランダム MAC アドレスがデフォルトで有効化されます。CWA リダイレクト時と証明書インストール後の EAP-TLS 再接続時で MAC アドレスが変更されると、`RegisteredDevices` グループのマッチングに失敗し、再度オンボーディングポータルへリダイレクトされる現象が発生します。（対処: ユーザーに該当 SSID でプライベート MAC アドレスを無効化するよう周知するか、証明書の CN/SAN ベースのポリシー設計を行う）。
* **OS ごとの Native Supplicant Profile (NSP) サポート差異**:
  * **Windows / macOS**: Network Setup Assistant (NSA) アプリケーションを通じて自動的に Wi-Fi プロファイルと証明書がインストールされます。
  * **iOS (iPhone/iPad)**: Mobile Configuration Profile (`.mobileconfig`) をダウンロードしてユーザーが手動で設定画面からプロファイルインストールを承認する必要があります。
  * **Android**: Android 11 以降ではセキュリティ制限が強化されており、NSA アプリケーションの導入または Chrome ブラウザ経由での証明書インストールが必要となります。
* **ライセンス依存**: Cisco ISE で BYOD オンボーディング（NSP, SCEP/EST 証明書発行）を運用するには、**ISE Advantage ライセンス**（または旧 Apex ライセンス）が必要です。

---

## 🔄 他技術との関連

* **802.1X & EAP-TLS**: オンボーディング完了後のセキュアな認証方式。クライアント証明書と ISE CA を用いて相互認証を行います。
* **CWA (Central Web Authentication)**: 未登録端末を検知し、HTTP 302 リダイレクトによって BYOD ポータルへ誘導する認証基盤。
* **CA & SCEP / EST**: デバイス証明書の発行プロトコル。ISE Internal CA または外部 Microsoft CA (NDES) と連携して証明書を自動プロビジョニングします。
* **Cisco TrustSec (SGT)**: BYOD 登録端末に対して、アクセス層で Security Group Tag (`BYOD_Secured` 等) を動的付与し、マトリクスベースのアクセス制御を実施します。
* **Dynamic ACL (dACL) / Dynamic VLAN**: BYOD 登録状態に応じて、未登録時はオンボーディング専用 VLAN/dACL、登録後は社内 VLAN/PERMIT_ALL dACL へ推移させます。

---


## 🧩 比較表

### Single-SSID vs Dual-SSID BYOD フロー比較

| 評価項目 | Single-SSID BYOD フロー | Dual-SSID BYOD フロー |
| --- | --- | --- |
| **SSID 構成** | 単一の SSID (`Corporate-BYOD`) | 2つの SSID (`Guest-Onboard` + `Secure-Corp`) |
| **初期接続方式** | PEAP-MSCHAPv2 または MAB | Open または PSK (Web Auth) |
| **最終認証方式** | EAP-TLS | EAP-TLS |
| **ユーザー体験 (UX)** | 同一 SSID のまま背景で CoA が発行され自動昇格 | 登録完了後、手動で暗号化 SSID へ切り替える必要あり |
| **設定の複雑さ** | 高（CoA レスポンス、セッション状態保持の精密な設計が必要） | 中（ポータルとプロファイル設定は独立） |
| **セキュリティ** | 最初から PEAP/MAB で暗号化保護 | 初期接続時と登録後の接続が別 SSID |

---

## 💡 ベストプラクティス

1. **SAN (Subject Alternative Name) への属性埋め込み**:
   * クライアント証明書を発行する際、SAN フィールドに `$mac$` (MAC アドレス) および `$user$` (ユーザー名) の両方を埋め込むよう Certificate Template を設計してください。これにより、ランダム MAC アドレス環境下でも証明書自体の属性情報を用いて ISE 上でデバイスとユーザーを正確にマッピングできます。
2. **証明書有効期限と自動更新の設計**:
   * BYOD 用クライアント証明書の有効期限は通常 365 日に設定します。期限切れ前の更新通知ポータルまたは自動再プロビジョニングルールを事前構成してください。
3. **Redirect ACL の厳密な最適化**:
   * Redirect ACL 内で、DNS (53), DHCP (67/68), ISE PSN IP (8443/8905/8909) を明示的に `deny` (バイパス) し、必要通信以外を `permit` (トラップ) するよう設計してください。

---


## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルの実践的な設定・検証サンプル例です。

### 演習 1: Single-SSID BYOD (EAP-PEAP から EAP-TLS への CoA 昇格)

* **問題**: 同一の SSID `Corporate-BYOD` 上で、未登録端末の初回接続を PEAP-MSCHAPv2 で認証し、BYOD ポータルへリダイレクトして証明書を発行した後、RADIUS CoA によって同じ SSID のまま EAP-TLS へ自動昇格させよ。
* **要件**:
  1. 初期 PEAP 認証成功時、Authorization Profile `BYOD_Redirect` (CWA Redirect & ACL `ACL_BYOD_REDIRECT`) を返却すること。
  2. ポータル上でデバイスが登録されたら `RegisteredDevices` Identity Group へ自動追加すること。
  3. 証明書取得後の EAP-TLS 再認証時は `PermitAccess` と SGT `BYOD_Secured` (Tag 20) を付与すること。
* **設定例 (Cisco ISE)**:
  ```text
  ! 1. Certificate Template
  Administration > Identity Management > Settings > PKI > Certificate Templates
  Name: BYOD_Client_Template
  Key Size: 2048
  Subject: CN=$user$
  SAN: Email=$user$@corp.local, MAC=$mac$

  ! 2. Native Supplicant Profile
  Work Centers > BYOD > Native Supplicant Profiles
  Name: NSP_Single_SSID
  Operating System: Windows / macOS / iOS / Android
  SSID: Corporate-BYOD
  Allowed Protocol: EAP-TLS
  Certificate Template: BYOD_Client_Template

  ! 3. Client Provisioning Policy
  Policy > Client Provisioning
  Rule 1: OS EQUALS All Profile: NSP_Single_SSID

  ! 4. Authorization Policy
  Policy > Policy Sets > Default > Authorization Policy
  Rule 1 (Secured): 
    Condition: Network Access:AuthenticationMethod EQUALS EAP-TLS AND IdentityGroup EQUALS RegisteredDevices
    Permissions: PermitAccess, Security Group: BYOD_Secured (20)
  Rule 2 (Redirect): 
    Condition: Network Access:AuthenticationMethod EQUALS PEAP-MSCHAPv2 AND BYODnotRegistered EQUALS true
    Permissions: BYOD_Redirect_Profile (Web Redirection: BYOD Portal, ACL: ACL_BYOD_REDIRECT)
  ```
* **検証方法**:
  * `show access-session interface G1/0/11 details` を実行し、初期 PEAP 時に Status Authorized, Redirect ACL `ACL_BYOD_REDIRECT` が適用されていることを確認。
  * ポータル完了後、ISE Operations > RADIUS > Live Logs で `CoA Type: Re-authenticate` が送信され、再接続時に AuthenticationMethod: EAP-TLS, SGT: BYOD_Secured (20) に変化したことを検証。

---

### 演習 2: Dual-SSID BYOD (オープン SSID からセキュア SSID への遷移)

* **問題**: オンボーディング専用のオープン SSID `Guest-Onboard` で接続し、証明書およびセキュア SSID `Secure-Corporate` 用の EAP-TLS 設定を注入した後、自動的に `Secure-Corporate` へ接続を切り替えさせよ。
* **要件**:
  1. `Guest-Onboard` 接続時は CWA リダイレクトにより BYOD ポータルを表示し、証明書をプロビジョニングすること。
  2. NSP プロファイル内でターゲット SSID を `Secure-Corporate` に指定すること。
  3. `Secure-Corporate` 上では EAP-TLS 認証のみを許可し、内線 VLAN 10 を動的割当すること。
* **設定例 (ISE & Catalyst 9800 WLC)**:
  ```text
  ! ISE Native Supplicant Profile (Dual-SSID)
  Work Centers > BYOD > Native Supplicant Profiles
  Name: NSP_Dual_SSID
  Target SSID: Secure-Corporate
  Allowed Protocol: EAP-TLS
  Certificate Template: BYOD_Client_Template

  ! Catalyst 9800 WLC 設定
  wlan Guest-Onboard 1 Guest-Onboard
   security open
  wlan Secure-Corporate 2 Secure-Corporate
   security wpa wpa2
   security wpa wpa2 ciphers aes
   security wpa akm 802.1x

  wireless profile policy Policy_Secure_Corp
   aaa-override
   vlan 10
   no shutdown
  wlan profile-name Secure-Corporate policy Policy_Secure_Corp
  ```
* **検証方法**:
  * 端末で `Guest-Onboard` に接続し、ブラウザでポータル画面から登録実施。
  * プロファイルインストール後、端末の Wi-Fi 接続先が自動的に `Secure-Corporate` に変更され、`show wireless client mac-address xxxx.xxxx.xxxx detail` で VLAN 10 および EAP-TLS 状態であることを確認。

---

### 演習 3: 外部 PKI (SCEP) 連携によるクライアント証明書発行

* **問題**: Cisco ISE 内部 CA ではなく、外部証明書機関 (SCEP サーバー) と連携して端末証明書を発行せよ。
* **要件**:
  1. ISE に SCEP プロファイルを設定し、証明書サーバーに接続すること。
  2. CA 証明書チェーンを ISE にインポートすること。
* **設定例 (Cisco ISE)**:
  ```text
  ! ISE SCEP Profile 設定
  Administration > Identity Management > External Identity Sources > SCEP Profiles
  Name: External_SCEP_Profile
  Root CA Certificate: Select External_Root_CA

  ! Certificate Template で SCEP Profile を指定
  Administration > Identity Management > Settings > PKI > Certificate Templates
  BYOD_Client_Template > CA Configuration: External SCEP > Select External_SCEP_Profile
  ```
* **検証方法**:
  * オンボーディング実行時に、ISE のシステムログを確認し、証明書発行要求が外部 CA サーバーへ正常に応答されたことを検証。

---

### 演習 4: Android 端末向け NSP プロビジョニングおよびリダイレクト例外設定

* **問題**: Android 端末のオンボーディングにおいて、キャプティブポータル検出通信が Redirect ACL でブロックされないよう設定せよ。
* **要件**:
  1. Redirect ACL にてキャプティブポータル検証サーバー宛て通信をリダイレクト除外 (`deny`) とすること。
  2. ISE ポータル上で Android 向けプロビジョニング設定を有効化すること。
* **設定例 (WLC / Switch Redirect ACL)**:
  ```text
  ip access-list extended ACL_BYOD_REDIRECT
   deny udp any any eq domain
   deny udp any any eq bootps
   deny udp any any eq bootpc
   deny ip any host 192.168.10.50
   permit tcp any any eq www
   permit tcp any any eq 443
  ```
* **検証方法**:
  * Android 端末接続時、キャプティブポータルポップアップが正常に起動し、NSP アプリケーションが中断することなく実行できることを確認。

---

### 演習 5: iOS (.mobileconfig) 端末向けプロファイル自動インストール

* **問題**: iPhone/iPad (iOS) デバイス接続時、Cisco ISE から Apple Mobile Configuration Profile (`.mobileconfig`) を配信し、Wi-Fi EAP-TLS 設定と SCEP 証明書を自動注入せよ。
* **要件**:
  1. iOS デバイス接続時、Safari ブラウザ経由でプロファイルダウンロードダイアログを表示させること。
  2. プロファイル内に EAP-TLS Payload, Identity Payload, Wi-Fi Payload (SSID: `Corporate-BYOD`) を含めること。
* **設定例 (ISE Policy & NSP)**:
  ```text
  ! Native Supplicant Profile (iOS)
  Work Centers > BYOD > Native Supplicant Profiles > NSP_iOS
  Target OS: iOS
  Payload Type: Mobile Config Profile
  Wi-Fi Security Type: WPA2-Enterprise
  EAP Type: EAP-TLS
  SCEP Association: Internal_CA_Template

  ! Client Provisioning Policy
  Rule_iOS: Identity Group EQUALS Any AND OS EQUALS iOS ➔ Profile: NSP_iOS
  ```
* **検証方法**:
  * iOS 端末で接続時、Safari にてプロファイルダウンロード画面が表示され、設定アプリから承認・インストールされた後、EAP-TLS で再接続されることを確認。

---


### 演習 6: Windows 端末向け NSA (Network Setup Assistant) プロビジョニング

* **問題**: Windows 10/11 ドメイン未参加端末接続時、ISE ポータルから Network Setup Assistant (NSA) ウィザードを実行し、EAP-TLS Wi-Fi 設定およびクライアント証明書を自動作成せよ。
* **要件**:
  1. Client Provisioning Policy で Windows OS 検出時に NSA executable を配信すること。
  2. NSA アプリ実行により、Windows 証明書ストア (Personal) へ証明書を格納し、Wireless Profile を EAP-TLS へ自動構成すること。
* **設定例 (ISE Client Provisioning)**:
  ```text
  ! Client Provisioning Resources
  Work Centers > BYOD > Client Provisioning > Resources
  Add > Network Setup Assistant Application (Cisco-NSA-Setup.exe)
  Native Supplicant Profile: NSP_Windows_EAPTLS

  ! Client Provisioning Policy
  Rule_Win: OS EQUALS Windows All ➔ Profile: NSP_Windows_EAPTLS
  ```
* **検証方法**:
  * Windows 端末でブラウザからポータルへアクセスし、NSA をダウンロード・実行。プロビジョニング成功メッセージの表示と `certmgr.msc` で Personal Store に証明書が追加されたことを確認。

---

### 演習 7: Catalyst 9800 WLC における CWA Redirect ACL および CoA 構成

* **問題**: Catalyst 9800 WLC にて ISE BYOD 用 RADIUS Dynamic Authorization (CoA) および CWA Redirect ACL を設定せよ。
* **要件**:
  1. RADIUS CoA (UDP 1700/3799) を受信可能にし、共有鍵 `Cisco123!` を適用すること。
  2. Redirect ACL `ACL_BYOD_REDIRECT` で DNS, DHCP, ISE 通信を除外すること。
* **設定例 (Catalyst 9800 CLI)**:
  ```text
  aaa server radius dynamic-author
   client 192.168.10.50 server-key Cisco123!

  ip access-list extended ACL_BYOD_REDIRECT
   deny udp any any eq domain
   deny udp any eq domain any
   deny udp any any eq bootps
   deny udp any any eq bootpc
   deny ip any host 192.168.10.50
   permit tcp any any eq www
   permit tcp any any eq 443
  ```
* **検証方法**:
  * WLC で `show aaa servers` および `show radius statistics` を実行し、CoA ポートが開放され受信統計カウンターが増加することを確認。

---

### 演習 8: Catalyst IOS-XE スイッチにおける有線 (Wired) BYOD オンボーディング

* **問題**: Catalyst 9300 スイッチの有線ポートにて IBNS 2.0 (C3PL) を使用し、未登録端末の MAB 接続を CWA ポータルへリダイレクトして EAP-TLS へ昇格させよ。
* **要件**:
  1. 有線ポートで MAB 認証を実行し、未登録時は CWA リダイレクト ACL および URL を適用すること。
  2. ポータル登録完了後、ISE から RADIUS CoA Port Bounce を送信し、有線ポートの再リンク・EAP-TLS 認証を行わせること。
* **設定例 (Catalyst 9300 CLI)**:
  ```text
  aaa server radius dynamic-author
   client 192.168.10.50 server-key Cisco123!

  ip access-list extended ACL_BYOD_REDIRECT
   deny udp any any eq domain
   deny udp any any eq bootps
   deny udp any any eq bootpc
   deny ip any host 192.168.10.50
   permit tcp any any eq www
   permit tcp any any eq 443

  interface GigabitEthernet1/0/15
   switchport mode access
   switchport access vlan 100
   authentication periodic
   authentication timer reauthenticate server
   access-session port-control auto
   service-policy input PM_PORT_AUTH
  ```
* **検証方法**:
  * `show access-session interface G1/0/15 details` で Status `Authorized`, CWA Redirect URL 適用を確認。
  * 端末登録完了時、Port Bounce が発生し、再接続後に EAP-TLS 認証成功・dACL `PERMIT_ALL` 適用を確認。

---

### 演習 9: Dynamic SGT 移行 (BYOD_Unregistered から BYOD_Secured への変更)

* **問題**: Cisco TrustSec 環境において、端末の BYOD 登録状態に応じて SGT (Security Group Tag) を動的に変更せよ。
* **要件**:
  1. 未登録状態での接続時: SGT `BYOD_Unregistered` (Tag 15) を付与すること。
  2. 証明書プロビジョニング完了後の EAP-TLS 接続時: SGT `BYOD_Secured` (Tag 20) を付与すること。
* **設定例 (Cisco ISE Authorization Policy)**:
  ```text
  Policy > Policy Sets > Default > Authorization Policy

  ! Rule 1: EAP-TLS 登録済み端末
  Rule: EAP-TLS_Registered
    Condition: Network Access:AuthenticationMethod EQUALS EAP-TLS AND IdentityGroup EQUALS RegisteredDevices
    Permissions: PermitAccess, Security Group: BYOD_Secured (Tag 20)

  ! Rule 2: 初期 PEAP/MAB 未登録端末
  Rule: PEAP_Unregistered
    Condition: BYODnotRegistered EQUALS true
    Permissions: BYOD_Redirect_Profile, Security Group: BYOD_Unregistered (Tag 15)
  ```
* **検証方法**:
  * スイッチ / WLC で `show access-session interface <int> details` を実行し、初期接続時に `SGT: 15`、オンボーディング完了後に `SGT: 20` へ更新されることを確認。

---

### 演習 10: EST (Enrollment over Secure Transport) プロトコルによる証明書発行

* **問題**: SCEP に代わり、RFC 7030 準拠の EST (Enrollment over Secure Transport) プロトコルを使用して TLS 上で証明書プロビジョニングを実行せよ。
* **要件**:
  1. ISE の Certificate Template にて EST エンロールメントを有効化すること。
  2. TLS ポート 8443 上で HTTPS/REST ベースの証明書要求を受け付けること。
* **設定例 (Cisco ISE)**:
  ```text
  Administration > Identity Management > Settings > PKI > Certificate Templates
  Name: EST_BYOD_Client_Template
  Protocol Support: Enable EST (Enrollment over Secure Transport)
  Key Size: 2048
  Authentication Method: Client Certificate / Basic Auth (TLS Session)
  ```
* **検証方法**:
  * EST クライアント接続時、`prt-server.log` にて `.well-known/est/simpleenroll` への HTTPS POST リクエストおよび EST 証明書応答ログを確認。

---


## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験および記述試験を想定した実践問題集です。

### 問題 1 (コンフィグ読解)
* **質問**: 以下の Cisco ISE Authorization Policy 設定において、未登録ユーザーが初回 PEAP 認証を行った際、ポータルへリダイレクトされず `PermitAccess` が適用されてしまいました。コンフィグ上の原因を指摘し、正しいルール順序を記述しなさい。
  ```text
  [Rule 1]: Condition: Network Access:AuthenticationMethod EQUALS PEAP-MSCHAPv2 -> Result: PermitAccess
  [Rule 2]: Condition: BYODnotRegistered EQUALS true -> Result: BYOD_Redirect_Profile
  ```
* **解答・解説**:
  * **原因**: ISE の Policy Evaluation は最上位ルールから順に評価されるため、Rule 1 の `PEAP-MSCHAPv2` が先にマッチし、Rule 2 の未登録チェック (`BYODnotRegistered EQUALS true`) に到達しません。
  * **修正案**: 未登録チェックおよびリダイレクトルールを上位に配置するか、Rule 1 に「登録済みであること」を条件として追加します。
    ```text
    [Correct Rule 1]: Condition: BYODnotRegistered EQUALS true AND Network Access:AuthenticationMethod EQUALS PEAP-MSCHAPv2 -> Result: BYOD_Redirect_Profile
    [Correct Rule 2]: Condition: Network Access:AuthenticationMethod EQUALS EAP-TLS AND IdentityGroup EQUALS RegisteredDevices -> Result: PermitAccess
    ```

---

### 問題 2 (トラブルシューティング)
* **質問**: ユーザーが BYOD ポータルで証明書のダウンロードに成功した後、ネットワーク接続が切断され、EAP-TLS 再認証が行われませんでした。WLC で `show radius statistics` を実行したところ `CoA-Requests Received = 0` でした。考えられる根本原因を 2 つ挙げなさい。
* **解答・解説**:
  1. **原因 1: WLC 上で RADIUS Dynamic Authorization (CoA) が未設定または共有鍵不一致**
     * 確認方法: WLC CLI で `show running-config` を実行し、`aaa server radius dynamic-author` の設定および ISE IP / server-key を確認する。
  2. **原因 2: 途中経路のファイアウォールで UDP 1700 / 3799 通信がブロックされている**
     * 確認方法: ISE から WLC 宛ての UDP 1700/3799 通信を確認し、アクセスログを調査する。

---

### 問題 3 (Design)
* **質問**: モバイル端末 (iOS 14+ / Android 10+) がデフォルトで「プライベート MAC アドレス」を使用する環境において、Single-SSID BYOD フローを導入する際の設計上の懸念点と推奨される解決策を説明しなさい。
* **解答・解説**:
  * **懸念点**: CWA リダイレクト時（初期 MAC アドレス A）で登録した MAC アドレスが `RegisteredDevices` に追加されるが、証明書適用後の再接続時に端末が別のランダム MAC アドレス B を使用すると、ISE は未登録端末と判定し、再び CWA リダイレクトへループする。
  * **解決策**: 
    1. ユーザーに対して社内 BYOD 用 SSID で「プライベート MAC アドレス」を無効化する運用ガイドを提供する。
    2. 証明書ベース認証 (EAP-TLS) 時に、MAC アドレス条件ではなく証明書の SAN 属性 (Email/User) および ISE 登録済みユーザー・証明書検証のみで Authorization Policy をマッチングさせる設計に変更する。

---

### 問題 4 (コンフィグ読解)
* **質問**: 以下の Catalyst 9800 WLC の Redirect ACL コンフィグには重大な欠陥があります。ユーザーが Web ブラウザを開いた際にリダイレクトループが発生してポータルが表示されない理由を説明し、正しく修正しなさい。
  ```text
  ip access-list extended ACL_BYOD_REDIRECT
   permit tcp any any eq www
   permit tcp any any eq 443
  ```
* **解答・解説**:
  * **欠陥**: DNS (UDP 53) や DHCP、および ISE PSN 自体への通信がリダイレクト対象 (`permit`) となっているため、端末が名前解決を行えず、また ISE ポータル宛ての通信自体も WLC で再リダイレクトされてループします。
  * **修正案**: バイパスすべき通信 (DNS, DHCP, ISE IP 宛て) を `deny` に設定し、残りの HTTP/HTTPS を `permit` に設定します。
    ```text
    ip access-list extended ACL_BYOD_REDIRECT
     deny udp any any eq domain
     deny udp any any eq bootps
     deny udp any any eq bootpc
     deny ip any host 192.168.10.50
     permit tcp any any eq www
     permit tcp any any eq 443
    ```

---

### 問題 5 (実装・トラブル)
* **質問**: iPhone (iOS) 端末で BYOD ポータルからプロビジョニングを実行した際、プロファイルが適用されず EAP-TLS 設定が注入されませんでした。考えられる設定上の原因と対応手順を説明しなさい。
* **解答・解説**:
  * **原因**: iOS ではプロファイルをブラウザでダウンロードしただけでは適用されず、ユーザーが手動で iOS の設定画面からプロファイルインストールを承認する必要があります。また、ISE の Native Supplicant Profile (NSP) 内で iOS 向けの Mobile Configuration Profile 署名証明書が正しく設定されていない可能性があります。
  * **対処手順**: ISE 上で NSP プロファイルに有効な署名証明書を割り当て、ユーザー画面上に設定アプリからプロファイルを承認・インストールする手順を明示する案内を追加します。

---

## 🔗 参考リソース

Cisco 公式情報および技術ドキュメントの参照先一覧です。

* Cisco ISE BYOD Prescriptive Deployment Guide
* Cisco ISE BYOD Wireless Single-SSID Configuration Example
* Cisco ISE BYOD Dual-SSID Configuration Guide
* Configure Cisco ISE 3.x Native Supplicant Profile (NSP) for BYOD
* Cisco Catalyst 9800 Series Wireless Controller Configuration Guide - Central Web Authentication & BYOD
* Cisco Live BRKSEC-2040 - Deep Dive into Cisco ISE BYOD and Guest Access Deployment
* Cisco Live BRKSEC-3004 - Advanced ISE BYOD Implementation and Troubleshooting

---

## 📝 補足（Notes）

### BYOD トラブルシューティングのクイックフロー
1. **ポータルが開かない場合**:
   * クライアントの IP/DNS 取得を確認。
   * スイッチ / WLC で `show access-session` を実行し、Redirect ACL と Redirect URL が動的に割り当てられているか確認。
   * Redirect ACL で DNS / ISE IP が `deny` になっているか確認。
2. **証明書が発行されない場合**:
   * ISE の Operations > BYOD > Live Logs を確認。
   * SCEP / EST サーバーのステータスおよび ISE Internal CA サービス状態の確認。
3. **証明書取得後に EAP-TLS へ切り替わらない場合**:
   * ISE から WLC/スイッチ宛てに CoA (Re-authenticate) が正常送信されているか (`Operations > RADIUS > Live Logs`)。
   * WLC/スイッチで `show radius statistics` を実行し、CoA 受信エラーおよび共有鍵の不一致を調査。
   * 端末の Wi-Fi 設定で生成されたプロファイルが有効化されているか確認。

---


