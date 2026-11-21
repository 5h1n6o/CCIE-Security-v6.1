---
layout: default
title: 4.8-AnyConnect-provisioning
nav_order: 8
parent: 4.0-Identity-Management
---

# 4.8 Provisioning Cisco AnyConnect with Cisco ISE and Cisco ASA

## 📘 概要

Cisco Identity Services Engine (ISE) および Cisco Adaptive Security Appliance (ASA) を組み合わせた Cisco AnyConnect (Secure Client) プロビジョニングは、エンタープライズネットワークにおけるリモートアクセス VPN およびアクセス制御端末のクライアントソフトウェア、モジュール、プロファイルを自動配信・管理するための核心技術です。

従来の ASA 単体による Web-based (WebVPN) プロビジョニングとは異なり、Cisco ISE を統合したプロビジョニングモデル（Client Provisioning Policy: CPP）では、接続ユーザーの属性（Active Directory グループ、端末種別、ネットワークロケーション、認証方式など）に応じて動的に異なる AnyConnect ソフトウェアバージョン、モジュール（NAM, Posture/ISE Agent, Umbrella, AMP/Secure Endpoint, NVM, VPN）、および設定プロファイル（XML）を自動配信することが可能です。

### 利用目的および適用場面

* **ポスチャ検証 (Posture Compliance)**: 端末が VPN または 802.1X で接続した際、AnyConnect Posture Module (ISE Secure Client Posture) を自動インストール/更新し、OSパッチ・アンチウイルス定義・レジストリ・必須プロセスの遵守状況を検査します。
* **クライアントモジュールの一元化とライフサイクル管理**: VPN Client モジュールだけでなく、Network Access Manager (NAM)、Umbrella Roaming Security、AMP for Endpoints、Network Visibility Module (NVM) を単一の ISE ポータルから一括デプロイおよびバージョン統合管理を行います。
* **動的プロファイル切替**: 営業部、開発部、サードパーティベンダーなど、ユーザーロールに応じて異なる VPN 接続先（XML プロファイル）、802.1X セキュリティ設定（NAM プロファイル）、ポスチャ検査ポリシー（Posture プロファイル）を配信用ポリシールール（Client Provisioning Policy）で識別・適用します。
* **ASA ダイナミックアクセスポリシー (DAP) / RADIUS CoA 連携**: ASA 側で初期 VPN 接続を受け付けた後、ISE への RADIUS 認証/認可を通じて Client Provisioning ポータルへの HTTP/HTTPS リダイレクトを行い、プロビジョニング完了後に RADIUS CoA (Change of Authorization) を発行して完全なネットワークアクセス権限を付与します。

---

## 🔑 要点

Cisco ISE と Cisco ASA による AnyConnect プロビジョニングの主要コンポーネントおよび設計要件を下表に整理します。

| 項目 | 内容 |
| --- | --- |
| **主要コンポーネント** | Cisco ISE (Client Provisioning Engine / Posture Service)、Cisco ASA (VPN Gateway / Dynamic Access Policies / HTTP Redirect)、Cisco AnyConnect / Secure Client Package (`.pkg`)、Client Profiles (`.xml`) |
| **主な用途** | リモートアクセス VPN 接続時の自動クライアントソフトウェア更新、Posture モジュールデプロイ、NAM/Umbrella/AMP モジュール追加、プロファイル (`.xml`) 自動配信 |
| **配信メカニズム** | ASA 直接配信 (WebVPN Hostscan/AnyConnect package) または ISE ポスチャリダイレクト (CWA/CPP Web Portal + Compliance Module + AnyConnect Downloader) |
| **必要ライセンス** | **Cisco ISE**: Premier / Advantage License (Posture/Compliance 用)、Base/Essentials License (基本認証/認可 用)<br>**Cisco ASA/Secure Client**: Apex License (Posture/Apex 機能用)、Plus License (基本 VPN 用) |
| **プロトコル要件** | RADIUS (UDP 1812/1813)、RADIUS CoA (UDP 1700 または 3799)、HTTPS (TCP 443 / TCP 8443)、EAP-TLS/PEAP (UDP 1812)、HTTP Redirect (TCP 80/443) |
| **キーファイル種別** | AnyConnect Package (`anyconnect-win-x.x.x-webdeploy-k9.pkg`), AnyConnect Compliance Module (`anyconnect-compliance-x.x.x-k9.pkg`), ISE Posture Profile (`ISEPostureProfile.xml`), AnyConnect VPN Profile (`ACVPNProfile.xml`) |
| **設計上の注意点** | ① ASA 側の `group-policy` および `tunnel-group` での Redirect ACL 設定ミス防止<br>② ISE 側の Discovery Host (Default: `enrollment.cisco.com` または ISE IP) への DNS 解決要件<br>③ ASA トラフィックに対する CoA 受信ポート (UDP 1700/3799) の許可と AAA サーバー設定 |

---

## 🏗 動作原理

Cisco ISE と ASA を連携させた AnyConnect Client Provisioning および Posture の通信アーキテクチャは、**初期接続とリダイレクト**、**クライアントデプロイと評価**、**CoA によるアクセス権限更新**の3つのフェーズで構成されます。

### ネットワーク構成とデータフロー図

```
[ AnyConnect Client ]
       │
       │ 1. SSL/IPsec VPN Auth Request (HTTPS/UDP 4500)
       ▼
[ Cisco ASA (VPN GW) ] ◄───── 2. RADIUS Access-Accept with Redirect-ACL & CWA URL ─────► [ Cisco ISE (PAN/PSN) ]
       │                                                                                       │
       │ 3. Client HTTP Traffic Redirected to ISE Port 8443                                   │
       ├───────────────────────────────────────────────────────────────────────────────────────┤
       │ 4. Client Provisioning Portal / AnyConnect Downloader & Profile Provisioning (`.xml`)│
       │                                                                                       │
       │ 5. Posture Assessment & SW Compliance Check via AnyConnect Posture Agent              │
       │                                                                                       │
       │ 6. Posture Status (Compliant) Reported to ISE                                         │
       │                                                                                       │
       │ ◄──────────────── 7. RADIUS CoA (Re-auth / DACL / SGT Change) ──────────────────────┤
       │
       │ 8. Access Permit (Unrestricted VPN Tunnel Access)
       ▼
[ Corporate Network / Intranet ]
```

### 通信フローの補足解説

1. **初期接続認証**: AnyConnect クライアントが ASA に対して VPN 接続を開始します。ASA は ISE に対して RADIUS Access-Request を送信します。
2. **リダイレクト属性の返却**: ISE はユーザー/端末の初期状態（Posture 未検証）を認識し、RADIUS Access-Accept 内で Redirect URL および `url-redirect-acl=ACL-VPN-REDIRECT` を ASA に返却します。
3. **HTTP リダイレクト処理**: ASA はクライアントの Web ブラウザ通信（HTTP/80）を検知し、ISE の Client Provisioning Portal (TCP 8443) にリダイレクトします。
4. **AnyConnect モジュール・プロファイル配信**: ISE の Client Provisioning Engine が端末の OS (Windows/macOS) を判定し、AnyConnect Package (`.pkg`)、Compliance Module、および ISE Posture Profile (`ISEPostureProfile.xml`) をダウンロードさせます。
5. **ポスチャ評価と状態レポート**: インストールされた AnyConnect Posture Agent が起動し、ローカル端末のセキュリティ要件（アンチウイルス有効化、OS パッチなど）を評価した後、ISE (TCP 8443/8905) に Compliant 報告を送信します。
6. **RADIUS CoA によるアクセス昇格**: ISE は端末状態が "Compliant" に更新されたことをトリガーとして、ASA に対して RADIUS CoA (Re-evaluate / Push Policy) を送信します。ASA はリダイレクト ACL を解除し、通常のアクセス ACL (dACL) または SGT を適用して社内ネットワークへの完全なアクセスを許可します。

---

## ⚙ 動作シーケンス

AnyConnect クライアントが未インストールまたは旧バージョンの状態から、ISE と ASA を経由して自動デプロイおよび Posture 完了に至る詳細シーケンスを以下に示します。

```
Client (Host)                 Cisco ASA                      Cisco ISE                      DNS Server
  │                               │                              │                              │
  │─── 1. AnyConnect/Web VPN ───►│                              │                              │
  │    Authentication             │─── 2. RADIUS Access-Req ────►│                              │
  │                               │    (User/Pass, Tunnel-Grp)   │                              │
  │                               │                              │─── 3. Check Policy Set ───┐  │
  │                               │◄── 4. RADIUS Access-Accept ──│    (Posture Unknown)      │  │
  │                               │    (Redirect-ACL & CWA URL)  └──────────────────────────┘  │
  │                               │                                                             │
  │─── 5. Initial Web Traffic ───►│                                                             │
  │    (HTTP GET)                 │                                                             │
  │                               │─── 6. Intercept & Redirect ──►                              │
  │                               │    (HTTPS 302 to ISE 8443)   │                              │
  │                               │                              │                              │
  │─── 7. HTTPS Connect to ISE ─────────────────────────────────►│                              │
  │    (Client Provisioning Portal TCP 8443)                     │                              │
  │                                                              │─── 8. Evaluate CPP Rules ──┐ │
  │                                                              │    (Match OS, Group)      │ │
  │◄── 9. Download AC WebDeploy PKG, Posture Profile, Compliance └──────────────────────────┘ │
  │                                                                                             │
  │─── 10. AnyConnect Posture Agent Starts & Resolves Discovery Host ──────────────────────────►│
  │    (Queries 'enrollment.cisco.com' or ISE Discovery IP)                                             │
  │                                                                                             │
  │─── 11. Posture Assessment SW Check (TCP 8905/8443) ────────────────►│                              │
  │    (Sends AV/AS Status, Registry, OS Patch Data)             │                              │
  │                                                              │─── 12. Update Posture State─┐│
  │                                                              │    (Unknown -> Compliant)  ││
  │                                                              └────────────────────────────┘│
  │                               │◄── 13. RADIUS CoA (Push Policy / Disconnect-Reauth) ───────│
  │                               │                                                             │
  │◄── 14. Update VPN Filter / dACL (Unrestricted Access Permitted) ────────────────────────────│
```

### シーケンスの技術ポイント

* **Discovery Host の役割**: Posture Agent はインストール後、最初に `ISEPostureProfile.xml` に定義された Server IP または Discovery Host（デフォルト: `enrollment.cisco.com`）に対して HTTP/HTTPS リクエストを送信し、自分が ISE のリダイレクト状態にあるかを検出します。この DNS 解決が失敗すると、Posture Agent は ISE を発見できず "Server Not Found" で停止します。
* **Compliance Module の同期**: ISE にアップロードされている AnyConnect Compliance Module（OPSWAT エンジンベース）は、端末上の 350 種類以上のアンチウイルス・パーソナルファイアウォール製品のバージョンや定義ファイルを識別するために不可欠です。
* **RADIUS CoA ポートとセッションキー**: ASA と ISE 間で RADIUS CoA が正常に完了するためには、ASA 側の `aaa-server` 設定で `key` と CoA ポート（デフォルト UDP 1700）が一致しており、且つ Audit-Session-ID が正しく引き継がれている必要があります。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、Cisco ISE と ASA を使用した AnyConnect Client Provisioning および Posture 設定は、実機ララックおよび出題モデルで頻出する高難易度タスクの一つです。

### 1. Blueprintで重要なポイント

* **Client Provisioning Policy (CPP) の階層評価**: ISE 側で OS (Windows / macOS)、端末タイプ、ユーザーグループごとの条件（Conditions）に合致する AnyConnect ソフトウェアリソース（Package, Compliance Module, Posture Profile）を正しくマッピングできること。
* **リダイレクト ACL (Redirect ACL) の双方向定義**: ASA 上で設定する Web-Redirect ACL は、**「リダイレクト除外トラフィック（ISE 宛、DNS 宛、DHCP 宛など）を `deny`」**し、**「リダイレクト対象トラフィック（HTTP/HTTPS 宛）を `permit`」**に指定する逆転論理（Inverted Logic）を正確に記述すること。
* **Posture Policy と Remediation**: Posture 未評価端末に対する "Unknown" 状態の認可ルールと、評価完了後の "Compliant" 状態の認可ルール（SGT / dACL 切替）を正しく設計すること。

### 2. ラボ試験で設定させられそうな内容

* **ASA 側**:
  * `aaa-server` 設定での RADIUS Dynamic Authorization (CoA) 受信有効化 (`reactivation-mode`, `port 1700`)
  * `group-policy` 内の `redirect-acl` および `url-redirect-server` 設定、または ISE RADIUS 返却属性との連携
  * WebVPN / AnyConnect 基本イメージ保持設定 (`anyconnect image disk0:/...`)
* **ISE 側**:
  * AnyConnect WebDeploy Package (`.pkg`) および Compliance Module の Client Provisioning Resources への追加
  * AnyConnect Posture Profile の作成（Discovery Host または ISE IP、Server Port 8443/8905）
  * Client Provisioning Policy (CPP) の条件設定（Windows / macOS 条件、AnyConnect Configuration の割り当て）
  * Posture Requirement / Policy 設定（アンチウイルス有効化チェック、ファイル存在チェックなど）

### 3. よくある設定ミスとトラブルポイント

* **リダイレクト ACL の記述ミス**: `permit` と `deny` を逆にしてしまい、DNS トラフィックがリダイレクトされて名前解決が失敗する、または ISE 宛通信がリダイレクトされてループが発生する。
* **DNS 解決の失敗**: クライアントが Posture Discovery Host を解決できず、Posture Agent が ISE に接続できない。
* **CoA ポートの不一致**: ASA 側の Default CoA ポート (UDP 1700) と ISE の Network Device 設定（UDP 1700）の不一致。
* **ASA `sysopt connection permit-vpn` の影響**: IPsec/SSL VPN トラフィックがインターフェース ACL をバイパスするため、認可 dACL や VPN Filter の挙動を誤解すること。

---

## 🛠 設定方法

### 1. Cisco ASA の CLI 設定例

```bash
! --- AAA サーバーおよび RADIUS CoA 設定 ---
aaa-server ISE_RADIUS_GROUP protocol radius
aaa-server ISE_RADIUS_GROUP (inside) host 192.168.10.100
 key C!sc0_S3cur1ty_Secret
 authentication-port 1812
 accounting-port 1813
 dynamic-authorization port 1700

! --- リダイレクト ACL の作成 ---
! 注: deny はリダイレクト除外（直接通信許可）、permit は ISE 8443 へのリダイレクト対象
access-list ACL_VPN_REDIRECT extended deny udp any any eq domain
access-list ACL_VPN_REDIRECT extended deny ip any host 192.168.10.100
access-list ACL_VPN_REDIRECT extended deny ip host 192.168.10.100 any
access-list ACL_VPN_REDIRECT extended permit tcp any any eq www
access-list ACL_VPN_REDIRECT extended permit tcp any any eq https

! --- グループポリシーおよびトンネルグループ設定 ---
group-policy GP_ANYCONNECT_POSTURE internal
group-policy GP_ANYCONNECT_POSTURE attributes
 vpn-tunnel-protocol ssl-client
 default-domain example.com

tunnel-group TG_EMPLOYEE_VPN type remote-access
tunnel-group TG_EMPLOYEE_VPN general-attributes
 authentication-server-group ISE_RADIUS_GROUP
 accounting-server-group ISE_RADIUS_GROUP
 default-group-policy GP_ANYCONNECT_POSTURE
tunnel-group TG_EMPLOYEE_VPN webvpn-attributes
 group-alias "Employee VPN" enable
```

### 2. Cisco ISE の GUI 設定手順

#### Step 1: AnyConnect リソースの追加 (Work Centers > Posture > Client Provisioning > Resources)
1. **AnyConnect Software Package**: `anyconnect-win-4.10.x-webdeploy-k9.pkg` をアップロード。
2. **AnyConnect Compliance Module**: `anyconnect-compliance-4.3.x-k9.pkg` をアップロード。
3. **ISE Posture Profile**:
   * Name: `ISE_Posture_Profile`
   * Server Name Rules: `192.168.10.100` または `*.example.com`
   * Discovery Host: `enrollment` + `.` + `cisco` + `.` + `com` または `192.168.10.100`
   * Call Home List: `192.168.10.100:8443`
4. **AnyConnect Configuration**:
   * Name: `AC_Provisioning_Config`
   * Select AnyConnect Package, Compliance Module, and Posture Profile.

#### Step 2: Client Provisioning Policy (CPP) の設定 (Policy > Client Provisioning)
* Rule Name: `Windows_Posture_CPP`
* Identity Groups: `Any` (または `Employee_Group`)
* Operating System: `Windows All`
* Results: `AC_Provisioning_Config`

#### Step 3: Authorization Policy の設定 (Policy > Authorization)
* **Rule 1 (Unknown Posture)**:
  * Condition: `Session:PostureStatus EQUALS Unknown` AND `Network Access:AuthenticationStatus EQUALS Authenticated`
  * Authorization Result: `VPN_Posture_Redirect` (RADIUS 返却属性: `url-redirect-acl=ACL_VPN_REDIRECT`, `url-redirect=https://192.168.10.100:8443/portal/gateway`)
* **Rule 2 (Compliant Posture)**:
  * Condition: `Session:PostureStatus EQUALS Compliant`
  * Authorization Result: `PermitAccess` (または特定 dACL `dACL_EMPLOYEE_PERMIT`)
* **Rule 3 (Non-Compliant Posture)**:
  * Condition: `Session:PostureStatus EQUALS NonCompliant`
  * Authorization Result: `Quarantine_Access` (リフレッシュポータルへの限定アクセス)

---

## 🔍 検証コマンド

Cisco ASA CLI および Cisco ISE Live Logs での状態検証コマンドを下表に整理します。

### ASA CLI 検証コマンド表

| 目的 | コマンド |
| --- | --- |
| AnyConnect VPN セッション一覧表示 | `<code>show vpn-sessiondb detail anyconnect</code>` |
| リダイレクト ACL 適用状況の確認 | `<code>show access-list ACL_VPN_REDIRECT</code>` |
| 適用中の DAP / Dynamic Access Policy 確認 | `<code>show dynamic-access-policy stats</code>` |
| RADIUS サーバーおよび CoA ステータス確認 | `<code>show aaa-server ISE_RADIUS_GROUP</code>` |
| 特定ユーザーの認可 ACL / Session ID 確認 | `<code>show vpn-sessiondb anyconnect filter name username</code>` |
| デバッグ: RADIUS 認証 / CoA の詳細ログ | `<code>debug radius flag all</code>` |
| デバッグ: WebVPN リダイレクト処理 | `<code>debug webvpn raw-http 255</code>` |

### ISE Live Logs / CLI 検証コマンド表

| 目的 | 確認項目 / コマンド |
| --- | --- |
| **ISE Operations > RADIUS > Live Logs** | 認証サクセス、返却 RADIUS 属性 (`url-redirect-acl`, `url-redirect`) の確認 |
| **ISE Operations > Live Sessions** | Posture Status (`Unknown`, `In Progress`, `Compliant`, `NonCompliant`) のリアルタイム更新 |
| **ISE Work Centers > Posture > Reports** | ポスチャ評価詳細レポート（不合格要件、OS/AV ディレクトリ評価結果） |
| **ISE CLI サービス状態確認** | `<code>show application status ise</code>` (Application Server, Posture Engine) |


## 🚨 トラブルシュート

Cisco AnyConnect と ISE / ASA 連携における代表的な障害症状、発生要因、確認コマンドおよび解決策を下表にまとめます。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **VPN 接続直後にリダイレクトが機能せず、直接社内アクセスできてしまう** | ASA 側のグループポリシーで `sysopt connection permit-vpn` が有効化されている、または Authorization Rule で Redirect ACL が正しく返却されていない。 | ASA: `<code>show running-config sysopt</code>`<br>ISE: Live Logs の Authorization Result 属性確認 | RADIUS Authorization Profile で `url-redirect-acl` および `url-redirect` が正しく設定されているか確認する。 |
| **クライアントブラウザで Web リダイレクト時に TLS 証明書警告 / ループが発生する** | ASA の Redirect ACL で ISE IP (Port 8443) 宛のトラフィックを `deny` に指定していないため、ISE 宛通信が自分自身に再リダイレクトされる。 | ASA: `<code>show access-list ACL_VPN_REDIRECT</code>` | Redirect ACL の先頭に `deny ip any host <ISE-IP>` を追加し、ISE 宛通信をリダイレクトから除外する。 |
| **AnyConnect Posture Agent が "Server Not Found" で失敗する** | クライアントが Posture Discovery Host (`enrollment.cisco.com`) の IP アドレスを DNS 解決できない。 | Client: `<code>nslookup enrollment.cisco.com</code>`<br>ASA: Redirect ACL で DNS (UDP 53) が `deny` になっているか確認 | Redirect ACL に `deny udp any any eq 53` を設定し、リダイレクト状態でも DNS 解決を直接許可する。 |
| **Posture Assessment 完了後もアクセス権限が昇格しない (Compliant に変わらない)** | ① ISE から ASA への RADIUS CoA ポート (UDP 1700) がファイアウォールでブロックされている。<br>② ASA 側の `aaa-server` で `dynamic-authorization` が未設定。 | ASA: `<code>show aaa-server</code>`<br>ISE: Operations > RADIUS > Live Logs (CoA Failure 理由) | ASA の `aaa-server host` 設定内に `dynamic-authorization port 1700` を追加し、Shared Secret を一致させる。 |
| **Client Provisioning Portal で AnyConnect Package のダウンロードが拒否される** | ISE 側の Client Provisioning Policy (CPP) 条件にユーザーの OS やグループがマッチしていない。 | ISE: Operations > Reports > Client Provisioning | CPP ポリシールールを見直し、対象の OS (Windows All / macOS All) および Identity Group を正確に割り当てる。 |

---

## ⚠ 制限事項

AnyConnect プロビジョニングおよび Posture 運用における主要な制限事項および互換性条件は以下の通りです。

* **モバイル OS の Posture 制限**: AnyConnect Posture Agent (ISE Secure Client Posture) は Windows および macOS のみをサポートします。iOS および Android 端末に対しては、Cisco Endpoint Management (MDM) または Clientless Posture (Web Agent) を利用する必要があります。
* **証明書信頼チェーン**: AnyConnect ソフトウェアやプロファイルプロビジョニングを HTTPS 経由で実行する際、クライアント端末が ISE の HTTPS ポータル証明書（または発行元 Root/Intermediate CA）を信頼していない場合、プロビジョニングプロセスがセキュリティ警告で中断されます。
* **Split Tunneling 設定との干渉**: ASA 側で VPN Split Tunneling が設定されている場合、ISE の Discovery Host (`enrollment.cisco.com`) または ISE IP アドレスへのトラフィックが VPN トンネル内に正しくルーティングされるよう `split-tunnel-network-list` に含める必要があります。
* **バージョン依存関係**: AnyConnect Package、AnyConnect Compliance Module、および ISE システムのバージョンには厳格な互換性マトリクスが存在します。古い Compliance Module では最新の Windows 11 パッチや最新 AV ソフトウェアを正しく判定できない場合があります。

---

## 🔄 他技術との関連

AnyConnect プロビジョニング機能が他インフラ・セキュリティ技術とどのように連動するかを整理します。

* **Cisco TrustSec (SGT / SXP)**: Posture 判定が "Compliant" に移行した際、RADIUS CoA を通じて ASA 上の VPN セッションに Security Group Tag (SGT: 例 SGT=15 Employee) を動的割り当てし、インサイドネットワークでのマイクロセグメンテーションを実施します。
* **Cisco Umbrella Roaming Security**: ISE Client Provisioning (CPP) を介して AnyConnect Umbrella Roaming Security モジュールおよび `OrgInfo.json` プロファイルを自動配信し、VPN 切断時でも端末の DNS セキュリティを維持します。
* **Cisco Secure Endpoint (formerly AMP for Endpoints)**: AnyConnect AMP モジュールを CPP で同時デプロイし、EDR コネクタの自動セットアップを実現します。
* **DNS / DHCP & IPAM (DDI)**: VPN セッション確立直後の未認証（リダイレクト）状態で、DNS/DHCP トラフィックのみを解放するために ASA ACL および ISE ポータル通信経路が密接に関係します。

---

## 🧩 比較表

AnyConnect デプロイ手法および Posture モジュールの特徴比較を下表に示します。

| 比較項目 | ASA 直接配信 (WebVPN AnyConnect Image) | ISE Client Provisioning (CPP) デプロイ |
| --- | --- | --- |
| **主目的** | 基本的な VPN クライアントソフトウェアのインストール・バージョン維持 | ユーザー属性に応じた各種モジュール・プロファイル・Posture の動的配布 |
| **モジュール柔軟性** | VPN コア機能＋指定追加モジュールの一律配布 | AD グループ・OS・接続ロケーション別の動的モジュール/XML 選択 |
| **Posture 統合** | ASA Hostscan モジュールによる簡易点検 (DAP 連携) | ISE Posture Module (OPSWAT 連携) による詳細コンプライアンス検証 |
| **制御トリガー** | VPN 接続確立時の ASA ローカル判定 | RADIUS 認可 ＋ HTTPS Web リダイレクト ＋ RADIUS CoA |
| **管理ポイント** | ASA CLI / ASDM / FMC Platform Settings | Cisco ISE Admin GUI (Work Centers > Posture) |

---

## 💡 ベストプラクティス

Cisco が推奨する AnyConnect プロビジョニングおよび Posture 設計のベストプラクティスです。

1. **Discovery Host に専用 FQDN を使用**: デフォルトの `enrollment.cisco.com` を使用する代わりに、社内 DNS で自社ドメインの FQDN (例: `ise-discovery.example.com`) を作成し、`ISEPostureProfile.xml` に設定して ISE IP アドレスへマッピングします。
2. **Grace Period (猶予期間) の活用**: Posture コンプライアンス違反が発生した場合、即座にアクセスを断ち切るのではなく、特定要件（OS パッチ等）に対して 1〜3 日間の猶予期間 (Grace Period) を設定して業務影響を最小化します。
3. **プロファイルの段階的デプロイ**: AnyConnect ソフトウェアやプロファイル更新を行う際は、まず一部のテストグループ（IT 部門）に CPP ルールを適用し、安定性を確認した後に全体へ展開します。
4. **Compliance Module の定期更新**: ISE の Auto-Update 機能を有効化し、新しい AV 製品や OS バージョンに対応した Compliance Module を常に自動取得・更新します。


## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルの実践的な AnyConnect プロビジョニングおよび Posture 演習シナリオ 10 選です。全設定例は省略せず網羅しています。

### シナリオ 1: ASA と ISE 間における基本 AnyConnect リダイレクトと AAA 連携設定

#### 問題
Cisco ASA において AnyConnect VPN クライアント接続を受け付け、ISE による認証後に未認証（Posture Unknown）端末を ISE Client Provisioning Portal へ自動リダイレクトする構成を構築せよ。

#### 要件
* RADIUS サーバーグループ `ISE_AAA` を作成し、IP `192.168.10.100`、Key `C!sc012345`、CoA Port `1700` を設定すること。
* ASA 側に Redirect ACL `ACL_POSTURE_REDIRECT` を定義し、DNS (UDP 53) および ISE IP (`192.168.10.100`) 宛トラフィックをリダイレクトから除外すること。
* Tunnel Group `TG_REMOTE_VPN` を設定し、ISE から返却される `url-redirect-acl` および `url-redirect` 属性を正常に受信・処理させること。

#### 設定例

```bash
! ASA CLI Configuration
aaa-server ISE_AAA protocol radius
aaa-server ISE_AAA (inside) host 192.168.10.100
 key C!sc012345
 authentication-port 1812
 accounting-port 1813
 dynamic-authorization port 1700

access-list ACL_POSTURE_REDIRECT extended deny udp any any eq domain
access-list ACL_POSTURE_REDIRECT extended deny ip any host 192.168.10.100
access-list ACL_POSTURE_REDIRECT extended deny ip host 192.168.10.100 any
access-list ACL_POSTURE_REDIRECT extended permit tcp any any eq www
access-list ACL_POSTURE_REDIRECT extended permit tcp any any eq https

group-policy GP_REMOTE_VPN internal
group-policy GP_REMOTE_VPN attributes
 vpn-tunnel-protocol ssl-client
 default-domain ccie.local

tunnel-group TG_REMOTE_VPN type remote-access
tunnel-group TG_REMOTE_VPN general-attributes
 authentication-server-group ISE_AAA
 accounting-server-group ISE_AAA
 default-group-policy GP_REMOTE_VPN
tunnel-group TG_REMOTE_VPN webvpn-attributes
 group-alias "Remote VPN" enable
```

#### 検証方法
* ASA にて `show vpn-sessiondb detail anyconnect` を実行し、クライアントが接続した際に `Redirect ACL : ACL_POSTURE_REDIRECT` および `Redirect URL : https://192.168.10.100:8443/...` が適用されていることを確認する。

---

### シナリオ 2: リダイレクト除外 ACL の完全定義とループ防止構成

#### 問題
Web リダイレクト通信において、クライアントの Web ブラウザが無限ループや接続エラーを起こさないよう、ASA 上で完全な逆転論理 (Inverted Logic) ACL を構築せよ。

#### 要件
* DNS サーバー (`192.168.10.2`) 宛 UDP/TCP 53 トラフィックを許可 (deny) すること。
* DHCP サーバー宛 (UDP 67/68) トラフィックを許可 (deny) すること。
* ISE ポスチャノード (`192.168.10.100`) 宛の全 IP 通信を許可 (deny) すること。
* 上記以外の全 HTTP (TCP 80) / HTTPS (TCP 443) 通信をリダイレクト対象 (permit) に設定すること。

#### 設定例

```bash
access-list ACL_CWA_REDIRECT extended deny udp any host 192.168.10.2 eq domain
access-list ACL_CWA_REDIRECT extended deny tcp any host 192.168.10.2 eq domain
access-list ACL_CWA_REDIRECT extended deny udp any eq bootps any eq bootpc
access-list ACL_CWA_REDIRECT extended deny ip any host 192.168.10.100
access-list ACL_CWA_REDIRECT extended deny ip host 192.168.10.100 any
access-list ACL_CWA_REDIRECT extended permit tcp any any eq www
access-list ACL_CWA_REDIRECT extended permit tcp any any eq https
```

#### 検証方法
* ASA にて `show access-list ACL_CWA_REDIRECT` を実行し、端末が HTTP リクエストを送信した際に `permit` の hitcnt が増加し、DNS や ISE 宛通信の際に `deny` の hitcnt が増加することを確認する。

---

### シナリオ 3: ISE Client Provisioning Policy (CPP) と Posture Profile のマッピング

#### 問題
Cisco ISE において、Windows 10/11 端末接続時に AnyConnect VPN + Posture モジュールおよび専用 Posture Profile を配布する CPP ポリシーを構築せよ。

#### 要件
* AnyConnect Software Package (`anyconnect-win-webdeploy.pkg`) および Compliance Module をリソースに追加すること。
* ISE Posture Profile の Discovery Host に `enrollment.cisco.com` を定義し、Call Home 宛先ポートに `8443` を指定すること。
* Client Provisioning Policy ルール名 `CPP_Windows_Employees` を作成し、条件 `Windows All` に対して作成した AnyConnect Configuration を割り当てること。

#### 設定例 (ISE GUI 手順要約)

1. **Resources 作成**:
   * Work Centers > Posture > Client Provisioning > Resources > Add > Agent resources from local disk
   * アップロード: AnyConnect Package & Compliance Module
2. **Posture Profile 作成**:
   * Work Centers > Posture > Client Provisioning > Resources > Add > AnyConnect Posture Profile
   * Name: `AC_Posture_Profile_Sec`
   * Discovery Host: `enrollment.cisco.com`
   * Server Name Rules: `*.ccie.local`
3. **Client Provisioning Policy 定義**:
   * Policy > Client Provisioning > Insert Rule Above
   * Rule Name: `CPP_Windows_Employees` | Identity Groups: `Employee_Group` | Operating System: `Windows All` | Results: `AnyConnect Configuration (AC_Posture_Profile_Sec 含む)`

#### 検証方法
* 接続端末でブラウザを開き、ISE Client Provisioning Portal が表示され、AnyConnect Downloader 経由で Posture Profile (`ISEPostureProfile.xml`) が `C:\ProgramData\Cisco\Cisco AnyConnect Secure Mobility Client\Posture\` に配置されることを確認する。

---

### シナリオ 4: RADIUS CoA による Posture 評価完了後の動的アクセス権限昇格

#### 問題
AnyConnect Posture Agent の評価結果が "Compliant" となった際、ISE が RADIUS CoA をトリガーして ASA 上のセッション認可を昇格させる設定を構築せよ。

#### 要件
* ISE Authorization Policy で `Session:PostureStatus EQUALS Compliant` 条件を作成すること。
* 認可結果として完全アクセスを許可する `PermitAccess` プロファイルを返却させること。
* 状態遷移時に ISE から ASA へ CoA Re-evaluate / Push Policy パケットが送信されること。

#### 設定例 (ISE Policy & ASA Config)

```bash
! ASA CLI - RADIUS Dynamic Authorization 有効化確認
aaa-server ISE_AAA protocol radius
aaa-server ISE_AAA (inside) host 192.168.10.100
 key C!sc012345
 dynamic-authorization port 1700
```

```text
# ISE Authorization Policy Rules
Rule 1: Name: Posture_Compliant_Rule
        Condition: Session:PostureStatus EQUALS Compliant
        Permissions: PermitAccess

Rule 2: Name: Posture_Unknown_Rule
        Condition: Session:PostureStatus EQUALS Unknown
        Permissions: VPN_Redirect_Profile (url-redirect-acl=ACL_POSTURE_REDIRECT)
```

#### 検証方法
* クライアントの Posture 判定完了時、ISE Operations > Live Logs にて `CoA Status: Success` および `Posture Status: Compliant` のイベントが発生し、ASA DB 上の `Redirect ACL` 表示が消失して通信制限が解除されたことを確認する。

---

### シナリオ 5: 認可結果としての Dynamic ACL (dACL) の動的適用

#### 問題
Posture 評価完了後、端末に対して全許可ではなく、特定の社内サーバー（Web/DB）への通信のみを許可する Dynamic ACL (dACL) を ISE から ASA へ動的送信せよ。

#### 要件
* ISE ダウンロード可能 ACL `dACL_VPN_EMPLOYEE` を作成し、IP `10.1.1.0/24` へのアクセスのみを許可すること。
* Posture Compliant 条件に当該 dACL をマッピングすること。

#### 設定例

```text
# ISE Downloadable ACL Content (dACL_VPN_EMPLOYEE)
permit ip any 10.1.1.0 255.255.255.0
deny ip any any
```

```bash
! ASA CLI で適用結果の確認
show vpn-sessiondb detail anyconnect
! 表示出力例:
! Dynamic Access Filter : #ACSACL#-IP-dACL_VPN_EMPLOYEE-xxxxxx
```

#### 検証方法
* VPN 端末から `10.1.1.50` への ping が通る一方、`172.16.1.1` への通信が拒否されることを確認する。ASA 上で `show access-list #ACSACL#...` によりパケットドロップ数を確認する。
### シナリオ 6: AnyConnect NAM (Network Access Manager) モジュールおよび 802.1X プロファイルのプロビジョニング

#### 問題
社内無線/有線 LAN の 802.1X 接続を一元管理するため、VPN 接続時に AnyConnect NAM モジュールおよび 802.1X プロファイル (`configuration.xml`) を ISE から自動配信せよ。

#### 要件
* NAM Profile Editor で作成した EAP-TLS / PEAP プロファイルを含む XML を ISE にアップロードすること。
* Client Provisioning Policy で NAM モジュールを AnyConnect デプロイ構成要素に含めること。

#### 設定例 (ISE GUI & AnyConnect Profile)

1. **NAM Profile 作成**: Network Access Manager Profile Editor にて SSIDs `Corporate_WIFI` および EAP 設定 (PEAP-MSCHAPv2 / EAP-TLS) を定義し `NAM_Profile.xml` として保存。
2. **ISE リソース追加**: Work Centers > Posture > Client Provisioning > Resources > Add > AnyConnect NAM Profile > `NAM_Profile.xml` アップロード。
3. **CPP ルールバインド**: Client Provisioning Policy で `AC_Provisioning_Config` 内の NAM Profile に `NAM_Profile.xml` をマッピング。

#### 検証方法
* クライアント端末の AnyConnect UI に 「Network Access Manager」 タブが追加され、自動的に `Corporate_WIFI` への接続プロファイルが読み込まれていることを確認する。

---

### シナリオ 7: AnyConnect Umbrella Roaming Security モジュールおよび OrgInfo.json のデプロイ

#### 問題
VPN 切断時でも端末のクラウド DNS セキュリティを保護するため、ISE CPP を通じて AnyConnect Umbrella モジュールおよび組織識別情報 (`OrgInfo.json`) をデプロイせよ。

#### 要件
* Cisco Umbrella Dashboard からダウンロードした `OrgInfo.json` を ISE リソースに登録すること。
* Client Provisioning Policy で Umbrella モジュールを選択し、`OrgInfo.json` を紐付けること。

#### 設定例

1. **Umbrella Profile 登録**:
   * ISE Work Centers > Posture > Client Provisioning > Resources > Add > AnyConnect Umbrella Profile
   * ファイル名: `OrgInfo.json` を指定して保存。
2. **Provisioning 設定**:
   * AnyConnect Configuration にて `Umbrella Roaming Security` モジュールをチェックし、上記 `OrgInfo.json` を選択。

#### 検証方法
* 端末の `C:\ProgramData\Cisco\Cisco AnyConnect Secure Mobility Client\Umbrella\` に `OrgInfo.json` が配置され、Umbrella サービスが Roaming Protection 状態（Active/Protected）になっていることを確認する。

---

### シナリオ 8: ASA 側ローカル WebVPN イメージと ISE リダイレクトプロビジョニングの併用設計

#### 問題
AnyConnect 未導入端末は ASA の WebVPN ポータルから初回インストールを行わせ、既存端末の更新・モジュール追加は ISE CPP リダイレクトから実行させる複合デプロイモデルを構成せよ。

#### 要件
* ASA 側に AnyConnect イメージ `anyconnect-win-4.10.pkg` を保持し、`webvpn` 設定で有効化すること。
* ISE 側では CPP による動的プロファイル/モジュール配信を構成し、ASA と ISE の二重更新競合を防止すること。

#### 設定例

```bash
! ASA CLI Configuration
webvpn
 enable inside
 anyconnect image disk0:/anyconnect-win-4.10.05095-webdeploy-k9.pkg 1
 anyconnect enable
 tunnel-group-list enable
```

```text
# ISE Client Provisioning Policy
CPP Rule: AC_Deploy_Version_Check
Condition: AnyConnect Version LESS THAN 4.10.05095
Action: Push AC 4.10.05095 Package + Posture Engine
```

#### 検証方法
* ブラウザで ASA WebVPN Portal (`https://vpn.ccie.local`) に接続して初回 AnyConnect インストールが完了した後、VPN 確立時に ISE への HTTPS リダイレクトが走り、追加プロファイルが正常適用されることを確認する。

---

### シナリオ 9: Posture Non-Compliant 端末に対する Remediation ACL および隔離ポータル割り当て

#### 問題
Posture 評価で不合格（Non-Compliant）となった端末に対し、社内ネットワークへのアクセスを遮断し、アンチウイルス定義更新サーバーおよび ISE Remediation ポータルのみ通信を許可せよ。

#### 要件
* ASA 上に Remediation ACL `ACL_REMEDIATION` を作成し、定義更新サーバー (`10.2.2.100`) および ISE ポータル宛以外の通信を拒否すること。
* ISE Authorization Policy で `PostureStatus EQUALS NonCompliant` 時に当該 ACL および Remediation Redirect を適用すること。

#### 設定例

```bash
! ASA CLI - Remediation ACL
access-list ACL_REMEDIATION extended permit udp any any eq domain
access-list ACL_REMEDIATION extended permit ip any host 10.2.2.100
access-list ACL_REMEDIATION extended permit ip any host 192.168.10.100
access-list ACL_REMEDIATION extended deny ip any any
```

```text
# ISE Authorization Policy Rule
Rule Name: Posture_NonCompliant_Quarantine
Condition: Session:PostureStatus EQUALS NonCompliant
Result: AuthzProfile_Quarantine (dACL=ACL_REMEDIATION または url-redirect-acl=ACL_REMEDIATION)
```

#### 検証方法
* アンチウイルスが無効なテスト端末で VPN 接続した際、ステータスが NonCompliant に変化し、`10.2.2.100` 以外の全社内サーバー宛通信が拒否されることを確認する。

---

### シナリオ 10: 冗長 ASA / ISE 構成における RADIUS Dynamic Authorization (CoA) およびキー共有のチューニング

#### 問題
複数の ISE PSN ノード (`192.168.10.100`, `192.168.10.101`) および Active/Standby ASA クラスタ間における RADIUS CoA セッションキー引き継ぎとタイムアウトパラメータを最適化せよ。

#### 要件
* 両 PSN ノードを ASA `aaa-server` グループに登録し、CoA 受信ポート `1700` を両ノードで一致させること。
* RADIUS タイムアウトを 5 秒、再試行回数を 3 回に設定し、障害時のノード切り替えを迅速化すること。

#### 設定例

```bash
! ASA CLI Configuration
aaa-server ISE_CLUSTER protocol radius
aaa-server ISE_CLUSTER max-failed-attempts 3
aaa-server ISE_CLUSTER deadtime 10

aaa-server ISE_CLUSTER (inside) host 192.168.10.100
 key C!sc0_HA_Secret1
 timeout 5
 authentication-port 1812
 accounting-port 1813
 dynamic-authorization port 1700

aaa-server ISE_CLUSTER (inside) host 192.168.10.101
 key C!sc0_HA_Secret1
 timeout 5
 authentication-port 1812
 accounting-port 1813
 dynamic-authorization port 1700
```

#### 検証方法
* 主系 PSN (`192.168.10.100`) のインターフェースをダウンさせた際、ASA が副系 PSN にフェイルオーバーし、副系 PSN から送信された RADIUS CoA パケットが正常に処理されることを `show aaa-server` で確認する。


## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験および筆記試験（SCOR/CCIE）を想定した実践的なオリジナル問題 5 問です。

### 問題 1 (トラブルシューティング - リダイレクトループ)
**質問**:
エンジニアが ASA 上で AnyConnect Posture リダイレクトを設定したところ、VPN 接続端末のブラウザで Web リダイレクトポータルを開こうとすると「リダイレクトの回数が多すぎます」という TLS エラーが発生した。以下の ASA ACL 設定の不備を指摘し、正しいコンフィグを記述せよ。

```bash
access-list ACL_REDIRECT extended permit tcp any any eq www
access-list ACL_REDIRECT extended permit tcp any any eq https
```

**解答・解説**:
* **不備の理由**: ASA の Web リダイレクト ACL は、`deny` で指定した通信をリダイレクトから除外（直接通信許可）し、`permit` で指定した通信を ISE 宛にリダイレクトする逆転論理である。上記の ACL では ISE 自体の IP アドレス (`192.168.10.100`) 宛の TCP 8443/443 通信までリダイレクト対象になってしまうため、ISE から ISE への無限リダイレクトループが発生する。また DNS 通信も除外されていないため名前解決が失敗する。
* **修正コンフィグ**:
```bash
access-list ACL_REDIRECT extended deny udp any any eq domain
access-list ACL_REDIRECT extended deny ip any host 192.168.10.100
access-list ACL_REDIRECT extended deny ip host 192.168.10.100 any
access-list ACL_REDIRECT extended permit tcp any any eq www
access-list ACL_REDIRECT extended permit tcp any any eq https
```

---

### 問題 2 (実装 - Posture Agent Discovery 失敗)
**質問**:
AnyConnect Posture Agent を導入した端末で、VPN 接続後に Posture Agent が起動するものの「ISE Server Not Found」と表示されて評価が開始されない。ネットワーク側およびプロファイル側で確認すべき 2 つの主要な原因と対策を述べよ。

**解答・解説**:
1. **原因 1: DNS 解決不可**
   * **詳細**: Posture Agent は `ISEPostureProfile.xml` 内の Discovery Host (`enrollment.cisco.com`) または指定 FQDN へ HTTP 通信を行い、リダイレクト応答の有無で ISE を検出する。VPN 接続時の Redirect ACL で DNS 通信が遮断されているか、社内 DNS に該当 A レコードが存在しない場合、探索が失敗する。
   * **対策**: Redirect ACL に `deny udp any any eq domain` を追加し、DNS に Discovery FQDN のレコードを登録する。
2. **原因 2: Split Tunneling によるパケットドロップ**
   * **詳細**: ASA の Split Tunneling ACL から Discovery Host IP または ISE IP アドレスが除外されているため、Posture Discovery パケットが VPN トンネル外（ローカルインターネット）へ送信されて落とされている。
   * **対策**: ASA の `split-tunnel-network-list` に ISE IP アドレスブロックを含める。

---

### 問題 3 (コンフィグ読解 - RADIUS CoA ポートの不一致)
**質問**:
以下の ASA コンフィグおよび ISE Live Logs エラーメッセージを読み取り、Posture 完了後にアクセス権限が昇格しない原因を説明せよ。

```bash
! ASA Configuration
aaa-server ISE_GROUP protocol radius
aaa-server ISE_GROUP (inside) host 192.168.10.100
 key C!sc012345
 dynamic-authorization port 3799
```

```text
# ISE Live Logs Error
CoA Error: Dynamic Authorization Failed - Timeout (No Response from Network Device)
```

**解答・解説**:
* **原因**: ASA 側で Dynamic Authorization (CoA) 受信ポートを `UDP 3799` に変更しているが、Cisco ISE の Network Device 設定（ASA の登録情報）における Default CoA Port が `UDP 1700` のままであるため、ISE が 1700 番ポートへ CoA パケットを送信し、ASA がそれを受け取れずにタイムアウトしている。
* **対策**: ISE 側の Network Device 設定で CoA Port を `3799` に変更するか、ASA 側のコンフィグを ISE デフォルトの `dynamic-authorization port 1700` に変更してポート番号を合致させる。

---

### 問題 4 (Design - モジュールデプロイとポリシー設計)
**質問**:
ある企業で、一般社員には 「VPN + Posture モジュール」、営業社員には 「VPN + Posture + Umbrella Roaming モジュール」 を自動配布したい。Cisco ISE の Client Provisioning Policy (CPP) においてどのような設計を行うべきか説明せよ。

**解答・解説**:
1. **AnyConnect リソースの準備**:
   * AnyConnect WebDeploy Package、Compliance Module、Posture Profile、および Umbrella の `OrgInfo.json` を ISE リソースに追加する。
2. **2つの AnyConnect Configuration の作成**:
   * `AC_Config_General`: AnyConnect Package + Compliance Module + Posture Profile
   * `AC_Config_Sales`: AnyConnect Package + Compliance Module + Posture Profile + Umbrella Module (`OrgInfo.json`)
3. **Client Provisioning Policy (CPP) ルールの階層定義**:
   * **Rule 1 (上位)**: Rule Name `Sales_Provisioning` | Group: `ActiveDirectory:Sales_Group` | Results: `AC_Config_Sales`
   * **Rule 2 (下位)**: Rule Name `General_Provisioning` | Group: `ActiveDirectory:Domain_Users` | Results: `AC_Config_General`
   * ※上位ルールで営業グループを先に評価・適用させることで、要求通りの条件分岐を実現する。

---

### 問題 5 (実効設定 - Dynamic ACL と SGT の選択)
**質問**:
ASA VPN セッションにおいて、Posture 完了後のアクセスポリシーとして SGT (Security Group Tag) を使用する場合と dACL (Dynamic ACL) を使用する場合の、ASA 側の設定要件および動作の違いを解説せよ。

**解答・解説**:
* **dACL (Dynamic ACL)**:
  * **動作**: ISE が RADIUS Access-Accept または CoA で ACL 定義本文を ASA に送信し、ASA がそれを動的 ACL (`#ACSACL#...`) として VPN セッションに適用する。
  * **ASA 要件**: 特別なインフラ設定は不要。`aaa-server` および RADIUS 連携のみで動作する。
* **SGT (Security Group Tag)**:
  * **動作**: ISE が RADIUS 認可で SGT 値 (例: `Cisco-AV-Pair = cts:security-group-tag=000f-05`) を返却し、ASA が VPN 接続端末からのトラフィックに SGT インラインタグを付与、または SXP 経由で IP-SGT マッピングを他スイッチへ伝播する。
  * **ASA 要件**: ASA で TrustSec (`cts` コマンド群) および SXP (Security Group Exchange Protocol) の有効化が必要。

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよびテクニカルノート一覧です。

* [Cisco ISE Posture Configuration Guide](https://www.cisco.com/c/en/us/td/docs/security/ise/3-1/admin_guide/b_ise_admin_3_1/b_ise_admin_3_1_posture.html)
* [Configure AnyConnect Secure Mobility Client with Posture Service on ISE](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/116041-config-anyconnect-posture-ise-00.html)
* [Cisco ASA Series VPN CLI Configuration Guide](https://www.cisco.com/c/en/us/td/docs/security/asa/asa918/configuration/vpn/asa-918-vpn-config.html)
* [Cisco ISE Client Provisioning Policy Configuration](https://www.cisco.com/c/en/us/td/docs/security/ise/3-1/admin_guide/b_ise_admin_3_1/b_ise_admin_3_1_client_provisioning.html)
* [Cisco Secure Client Administrator Guide](https://www.cisco.com/c/en/us/td/docs/security/vpn_client/anyconnect/anyconnect410/administration/guide/b_AnyConnect_Administrator_Guide_4-10.html)
* [Troubleshoot AnyConnect Posture with Cisco ISE](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/215444-troubleshoot-anyconnect-posture-with-ci.html)
* [Cisco Live BRKSEC-3032: Deep Dive on Cisco ISE Posture and Client Provisioning](https://www.ciscolive.com/global/on-demand-library.html)
* [Cisco Validated Design (CVD): Secure Remote Access with ASA and ISE](https://www.cisco.com/c/en/us/solutions/design-zone.html)

---

## 📝 補足（Notes）

### デザインメモと実装上の注意点

1. **Posture Modules の名称変更 (Cisco Secure Client 5.x 移行)**:
   AnyConnect バージョン 5.0 以降は 「Cisco Secure Client」 に名称変更され、Posture モジュールも 「ISE Secure Client Posture Module」 と呼ばれます。ISE 3.x 側のテンプレート選択時、バージョンに応じたプロファイル構成に注意してください。
2. **ASA トラフィックバイパス設定 (`sysopt connection permit-vpn`)**:
   デフォルトで ASA は `sysopt connection permit-vpn` が有効化されており、VPN トンネルからのパケットは ASA 外部/内部 ACL をバイパスします。リダイレクトや dACL によるアクセス制御を行う場合、VPN Filter または dynamic-authorization (RADIUS 返却属性) によるセッション単位 ACL を正しく適用してください。
3. **証明書チェーンのローカル登録**:
   AnyConnect が ISE Client Provisioning Portal に接続した際の SSL ハンドシェイクエラーを抑止するため、端末の Trusted Root Certificate Store に ISE の HTTPS サーバ証明書を発行した Root CA 証明書を事前デプロイ（GPO 等）しておくことが運用上必須となります。


