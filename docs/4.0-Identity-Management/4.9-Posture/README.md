---
layout: default
title: 4.9-Posture
nav_order: 9
parent: 4.0-Identity-Management
---

# 4.9 Posture assessment with Cisco ISE

## 📘 概要

Posture assessment（ポスチャ評価）は、ネットワークに接続しようとする端末（エンドポイント）が組織のセキュリティポリシー（OSのパッチ適用状態、アンチウイルスソフトウェアの定義ファイル更新状況、必須プロセスの起動、パーソナルファイアウォールの有効化、ディスク暗号化状態など）に適合しているかを検証し、その適合状況（Compliant / Non-Compliant / Unknown）に応じてネットワークアクセスレベルを動的に制御するメカニズムである。

Cisco Identity Services Engine (ISE) におけるポスチャ評価は、端末にインストールされる永続型エージェント（Cisco AnyConnect / Secure Client Posture Module）、一時実行型エージェント（Temporal Agent）、あるいはエージェントレスポスチャ（Agentless Posture）を用いて実施される。RADIUS認証/認可フロー、AnyConnect Discovery (Discovery Host / Discovery DNS)、Web Client Redirection (ポスチャリダイレクト ACL)、ポスチャポリシー評価、および RADIUS Change of Authorization (CoA) を組み合わせることで、未評価端末の隔離（Quarantine）、評価処理、修復（Remediation）、および準拠状態確立後の完全アクセス許可（Compliant Access）をシームレスに実現する。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **主要機能** | エンドポイントのセキュリティ状態検証（OSパッチ、AV定義、プロセス、レジストリ、ファイル存在チェック等）と動的アクセス制御 |
| **主な構成要素** | AnyConnect ISE Posture Module、Temporal Agent、Agentless Posture、ISE Policy Service Node (PSN)、RADIUS NAS (Switch / WLC / ASA / FTD) |
| **主要認証・通信プロトコル** | RADIUS (UDP 1812/1813), RADIUS CoA (UDP 1700/3799), HTTPS (TCP 443), Discovery (UDP/TCP 8905) |
| **メリット** | 不正端末・パッチ未適用端末のネットワーク侵入防止、クライアント側自動修復（Auto-Remediation）による運用の自動化、ゼロトラストアクセスの推進 |
| **デメリット** | エージェント配備・更新管理のオーバーヘッド、ポスチャ評価完了までの接続遅延、クライアントOS依存の制限事項 |
| **対応プラットフォーム** | Windows、macOS、Linux (AnyConnect/Secure Client サポート範囲に依存) |
| **設計上の注意点** | リダイレクト ACL（Permit/Deny の逆転論理）の正確な記述、Discovery Host の名前解決・到達性確保、修復サーバー（WSUS、AV Server、ISE Portal）へのアクセス許可 |
| **ライセンス要件** | Cisco ISE Premier License (旧 Apex License) |

---

## 🏗 動作原理

ポスチャ評価は、未評価状態（Unknown）から評価中（In Progress）、不適合状態（Non-Compliant）によるアクセス制限および修復、適合状態（Compliant）への変化に伴うアクセスの自動昇格というライフサイクルに従って動作する。

### ポスチャ評価の全体通信フロー

1. **初期認証**: エンドポイントが 802.1X/MAB で接続。ISE は Posture Status: Unknown と判定。
2. **認可応答**: ISE は リダイレクト ACL と ポスチャポータル URL を送信。
3. **リダイレクト・ディスカバリ**: 端末の HTTP/8905 通信が ISE に到達。AnyConnect が ISE PSN を特定。
4. **ポリシー評価・修復**: ISE からポスチャルールが降下し、AV・パッチ状態をスキャン。不適合時は自動/手動修復。
5. **CoA 昇格**: 全項目クリアで ISE に Compliant 報告。ISE が RADIUS CoA (Reauth) を NAD に送信し、全アクセス許可を付与。

---

## ⚙ 動作シーケンス

1. **802.1X/MAB 受信と RADIUS 認証**: スイッチ/WLC は端末のアクセスを受け付け、ISE へ EAP-Access-Request を送信。
2. **初期認可ポリシーの合致**: ISE 認証ポリシー成功後、認可ポリシーにおいて `Session:PostureStatus EQUALS Unknown` にマッチ。
3. **リダイレクト属性の適用**: ISE は `cisco-av-pair = url-redirect` および `url-redirect-acl` を返却。
4. **AnyConnect エージェントのプローブ送信**: 端末上の AnyConnect Posture Module が起動し、8905 ポートで Discovery Probe を送出。
5. **ポスチャポリシーの適用と判定**: Client のシステム情報をスキャン。AV Policy や OS Patch を評価。
6. **CoA 送出とセッション更新**: 評価成功時、ISE は RADIUS CoA を NAD に送信。NAD は再認証を発行。
7. **Compliant 認可の適用**: 再認証時、ISE 内の `Session:PostureStatus EQUALS Compliant` ルールにマッチし完全アクセス許可を返却。


---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験における Posture Assessment の出題ポイント、実装要件、および落とし穴を以下に整理する。

### Blueprintにおける超重要ポイント
1. **リダイレクト ACL (Redirect ACL) の逆転論理**:
   - NAD (Catalyst Switch / WLC / ASA) に定義するリダイレクト ACL は、通常のアクセスリストと異なり「**Deny された通信がリダイレクト対象外（バイパスして通過させる通信）**」、「**Permit された通信が ISE ポータルに HTTP リダイレクトされる通信**」を意味する。
   - **必修バイパス対象 (Deny指定)**:
     - DNS (UDP/TCP 53)
     - DHCP (UDP 67/68)
     - ISE PSN への通信 (TCP 8905, TCP 8443, TCP 443, UDP 1812/1813/1700)
     - 修復サーバー通信 (WSUS, AV Update Server 等)
   - **リダイレクト対象 (Permit指定)**:
     - 一般の HTTP (TCP 80) 通信。

2. **AnyConnect Discovery ホストと FQDN/DNS**:
   - AnyConnect Posture Module は、`Discovery Host` 設定値（IPアドレスまたはFQDN）に対して 8905 ポートでプローブを試みる。
   - FQDN を使用する場合、未評価状態（Unknown）の時点でエンドポイントが DNS 解決を行えるよう、リダイレクト ACL で DNS (UDP 53) が Deny（通過許可）されている必要がある。

3. **RADIUS Change of Authorization (CoA) 設定**:
   - NAD 上で `aaa server radius dynamic-author` を設定し、ISE からの CoA Packet (UDP 1700 または 3799) を受信・処理できるように設定しなければならない。
   - `client <ISE_PSN_IP> server-key <KEY>` の不一致は、ポスチャ評価完了後のアクセス昇格失敗の最大原因となる。

4. **ポスチャ認可ポリシーの条件判定**:
   - ISE Policy Sets において、認可ルールに `Session:PostureStatus EQUALS Unknown`（リダイレクト認可）と `Session:PostureStatus EQUALS Compliant`（完全アクセス認可）を正しく上下関係で並べる必要がある。

---

## 🛠 設定方法

### 1. NAD (Cisco Catalyst Switch - IOS-XE) の設定

スイッチ側で 802.1X/MAB、RADIUS 認証・認可、Dynamic Authorization (CoA)、およびポスチャリダイレクト ACL を設定する。

```text
! RADIUS Server & CoA 設定
radius server ISE_PSN_1
 address ipv4 192.168.10.10 auth-port 1812 acct-port 1813
 key Cisco123!

aaa group server radius ISE_GROUP
 server name ISE_PSN_1

aaa authentication dot1x default group ISE_GROUP
aaa authorization network default group ISE_GROUP
aaa accounting identity default start-stop group ISE_GROUP

aaa server radius dynamic-author
 client 192.168.10.10 server-key Cisco123!

! IP Device Tracking / IP Source Guard の有効化 (ポスチャ通信維持に必須)
device-tracking binding recovery interval 30
ip device tracking probe delay 10

! ポスチャ用リダイレクト ACL (Deny=通過許可, Permit=リダイレクト)
ip access-list extended REDIRECT_POSTURE
 deny udp any any eq domain
 deny tcp any any eq domain
 deny udp any any eq bootps
 deny udp any any eq bootpc
 deny ip any host 192.168.10.10
 deny ip any host 192.168.10.20  ! WSUS / AV Patch Server
 permit tcp any any eq www
 permit tcp any any eq 443

! ポート設定 (IBNS 2.0 / C3PL モード)
interface GigabitEthernet1/0/11
 switchport mode access
 switchport access vlan 100
 authentication host-mode multi-auth
 authentication order dot1x mab
 authentication priority dot1x mab
 authentication port-control auto
 mab
 dot1x pae authenticator
 access-session host-mode multi-auth
 access-session closed
 access-session port-control auto
```

### 2. Cisco ISE の設定手順 (GUI)

1. **Client Provisioning Package のアップロード**:
   - `Work Centers` > `Posture` > `Client Provisioning` > `Resources` に進む。
   - AnyConnect/Secure Client ソフトウェアパッケージ (`anyconnect-win-*.pkg` または `webdeploy`) および Posture Profile XML を登録。

2. **AnyConnect Posture Profile の作成**:
   - `Agent Behavior (Posture) Profile` を作成。
   - **Server name rules**: `*` または `192.168.10.10`
   - **Discovery host**: `192.168.10.10` (または `ise.example.com`)
   - **Posture Agent Discovery Port**: `8905`

3. **Posture Policy Rules (条件とアセスメント) の定義**:
   - `Work Centers` > `Posture` > `Posture Policy` に進む。
   - **OS**: Windows All / macOS All
   - **Conditions**: 
     - `AV_Installed`: アンチウイルスソフトが常駐していること。
     - `AV_Defs_3Days`: AV定義ファイルが3日以内であること。
     - `Windows_Patch_KB`: 特定の修正プログラムが適用されていること。
   - **Remediation Actions**: 
     - `AV_Update_Message`: 定義ファイル更新サイトへのリンクリダイレクトまたは自動更新。

4. **Authorization Policy (Policy Sets) の作成**:
   - **Rule 1 (Posture Unknown)**:
     - 条件: `Wire_Authenticated` AND `Session:PostureStatus EQUALS Unknown`
     - 認可結果 (Authorization Result): `Posture_Redirect_Profile` (Web Redirection: `Posture Discovery`, ACL: `REDIRECT_POSTURE`)
   - **Rule 2 (Posture Non-Compliant)**:
     - 条件: `Wire_Authenticated` AND `Session:PostureStatus EQUALS Non_Compliant`
     - 認可結果: `Quarantine_Profile` (制限VACL / dACL)
   - **Rule 3 (Posture Compliant)**:
     - 条件: `Wire_Authenticated` AND `Session:PostureStatus EQUALS Compliant`
     - 認可結果: `PermitAccess` (dACL: `PERMIT_ALL`, SGT: `Employee_16`)


---

## 🔍 検証コマンド

ポスチャ状態、RADIUSセッション、リダイレクト状態を検証するためのコマンド一覧。

### NAD (Catalyst Switch) での検証コマンド

| 目的 | コマンド |
| --- | --- |
| 認証・認可セッション詳細表示 | `<code>show access-session interface GigabitEthernet1/0/11 details</code>` |
| リダイレクト ACL 適用状況確認 | `<code>show ip access-lists REDIRECT_POSTURE</code>` |
| RADIUS CoA / 属性送信確認 | `<code>show aaa servers</code>` |
| IP デバイス トラッキング確認 | `<code>show device-tracking database interface GigabitEthernet1/0/11</code>` |
| リアルタイム認証/認可デバッグ | `<code>debug devicetracking execution</code>` <br> `<code>debug access-session host</code>` <br> `<code>debug radius verbose</code>` |

### Cisco ISE での検証・デバッグログ

| 目的 | 確認手順 / ログファイル |
| --- | --- |
| リアルタイムセッションモニタ | `Operations` > `RADIUS` > `Live Logs` (Posture Status 列を確認) |
| ポスチャ詳細レポート | `Operations` > `Reports` > `Reports` > `Endpoint Policy` > `Posture Assessment by Endpoint` |
| ISE PSN ポスチャデバッグログ | `Operations` > `Troubleshoot` > `Debug Wizard` > `Debug Log Configuration` > PSN 選択 > `posture` component を `DEBUG` に変更 (`/opt/CSCOise/logs/posture.log`) |

---

## 🚨 トラブルシューティング

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| エージェントが ISE を発見できず 「ISE Not Found」 になる | リダイレクト ACL で DNS / ISE IP が Permit (リダイレクト対象) になっているか、8905 ポートがブロックされている | `<code>show access-session interface ... details</code>` <br> `<code>show ip access-lists REDIRECT_POSTURE</code>` | リダイレクト ACL 内で DNS (UDP 53) および ISE PSN IP への通信を `deny` (通過許可) に設定する。 |
| ポスチャ評価完了後もリダイレクト制限が解除されない | RADIUS CoA が NAD で受信できていないか、Shared Key の不一致、または UDP 1700 がドロップしている | `<code>show aaa servers</code>` <br> ISE Live Logs (CoA Failure) | NAD 上で `aaa server radius dynamic-author` と `client <ISE-IP> server-key <KEY>` を正確に設定する。 |
| エージェントが Posture Profile をダウンロードできない | Client Provisioning Policy のマッチ失敗、または PKG ファイルのポータル権限不足 | ISE Live Logs <br> `posture.log` | Client Provisioning Policy で OS タイプおよびグループ属性が一致しているか確認し、PKG パックを再アタッチする。 |
| IP アドレスの取得遅延により AnyConnect が Discovery を諦める | DHCP シーケンスと 802.1X/AnyConnect 起動のタイマ不整合 | AnyConnect Logs (system event log) | NAD で PortFast / Edge Port 設定および IP Device Tracking のタイマ最適化を実施する。 |
| ポスチャが常に 「Pending」 または 「Unknown」 で止まる | ポスチャポリシーの特定条件 (特定の修正プログラムチェック) がパッチバージョンミスマッチを起こしている | ISE Posture Assessment Report | Posture Condition 内のパッチ定義規則を最新の Cisco Feed またはワイルドカード評価に修正する。 |

---

## ⚠ 制限事項

1. **OS およびエージェント依存関係**:
   - AnyConnect / Secure Client Posture Module は、iOS / Android などのモバイル OS には対応していない（モバイルは MDM / EMM 連携による Compliance チェックを使用する）。
2. **証明書と HTTPS リダイレクトの警告**:
   - HTTP Redirection 時、ブラウザが HSTS (HTTP Strict Transport Security) 対応サイト（Google, Yahoo 等）に接続しようとすると、証明書エラーが表示されリダイレクト画面が開かない場合がある。端末は HTTP サイトへのアクセスまたは AnyConnect 独自の Probe でリダイレクトを検出する必要がある。
3. **CoA タイムアウトとネットワーク遅延**:
   - WAN 経由で PSN と NAD が離れている場合、CoA 応答のタイムアウト（デフォルト5秒）によりセッション更新が失敗することがある。

---

## 🔄 他技術との関連

* **802.1X / MAB**: ポスチャ評価の第1フェーズとして機能。端末の基本アイデンティティを確立。
* **RADIUS CoA**: ポスチャ状態変更時（Unknown → Compliant）にセッションを動的に再評価・再認証させるキー技術。
* **Cisco TrustSec (SGT)**: ポスチャ判定結果に基づき、アクセスレイヤで Compliant 用 SGT (例: SGT 16) や Quarantine 用 SGT (例: SGT 15) を割り当てる。
* **dACL / VACL**: 評価前（Unknown）および不適合（Non-Compliant）時の通信宛先を制限。
* **MDM / Integration (pxGrid)**: モバイル端末や非AnyConnect端末のセキュリティ状態情報を ISE へ共有。

---

## 🧩 比較表

| 比較項目 | AnyConnect Permanent Agent | AnyConnect Temporal Agent | Agentless Posture |
| --- | --- | --- | --- |
| **インストール要否** | 永続インストールが必要 | 一時ダウンロード・起動（終了時消去） | 不要（Active Directory/WMI/WinRM経由） |
| **評価機能の深さ** | 非常に深い（ファイル、AV、パッチ、プロセス、暗号化状態） | 中程度（AV、OSバージョン等） | 限定的（レジストリ、パッチ、ドメイン参加状態） |
| **自動修復 (Remediation)** | 完全サポート（ファイル修正、AV自動更新） | 制限あり（メッセージ表示・リンク提供等） | サポートなし |
| **適用場面** | 企業管理端末 (Corporate Owned) | ゲスト・BYOD・一時接続端末 | 管理管理者権限のないドメイン端末・ライトウェイト検証 |
| **管理者権限** | インストール時のみ要権限 | 実行時にローカル管理者権限要 | AD 管理者資格情報が ISE 側に必要 |

---

## 💡 ベストプラクティス

1. **ポスチャリダイレクト ACL のテンプレート化**:
   - すべての NAD スイッチで一貫したリダイレクト ACL 名・ルールを適用し、DNS / DHCP / ISE IP / 修復サーバー を確実に `deny`（通過）に含める。
2. **AnyConnect Posture Profile の冗長化**:
   - Discovery Host に ISE PSN の VIP (Virtual IP) または 複数 PSN IP アドレス（カンマ区切り）を指定し、単一 PSN 障害時もディスカバリが成功するように設計する。
3. **Grace Period (猶予期間) の活用**:
   - 新規パッチ配布時、不適合端末を即座に遮断するのではなく、「Posture Grace Period (例: 3日間)」を設定し、修復を促しつつ限定的な業務アクセスを許可する。


---

## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルを想定した実践的な演習問題10選。設定例は省略せず網羅的に記載する。

### 【演習 1】Catalyst Switch における Posture 用 Redirect ACL と Dynamic Authorization (CoA) の構築

#### 問題
スイッチ SW1 において、802.1X 認証成功後に未評価端末（Posture Status: Unknown）を ISE ポスチャポータルに誘導するためのリダイレクト ACL を設定し、ISE からの RADIUS CoA を受信できるように設定しなさい。

#### 要件
1. RADIUS サーバー IP: 10.1.100.10、Shared Secret: CiscoLab123!
2. RADIUS CoA (Dynamic Authorization) ポート 1700 で ISE からの CoA を受信用に設定。
3. リダイレクト ACL 名: ACL_POSTURE_REDIRECT
4. 通信要件:
   - DNS (UDP 53) 通信はリダイレクトさせず正常に通過させること。
   - DHCP (UDP 67/68) 通信は正常に通過させること。
   - ISE PSN (10.1.100.10) への直通信はリダイレクトさせず通過させること。
   - 一般の HTTP (TCP 80) および HTTPS (TCP 443) 通信は ISE へリダイレクトさせること。
5. ギガビットポート Gi1/0/1 で IBNS 2.0 (C3PL) ポート設定を有効化。

#### 設定例

```text
! RADIUS サーバー定義
radius server ISE_PSN_PRIMARY
 address ipv4 10.1.100.10 auth-port 1812 acct-port 1813
 key CiscoLab123!

aaa group server radius ISE_GROUP
 server name ISE_PSN_PRIMARY

! RADIUS CoA 設定
aaa server radius dynamic-author
 client 10.1.100.10 server-key CiscoLab123!
 port 1700

! AAA 認証・認可・会計設定
aaa authentication dot1x default group ISE_GROUP
aaa authorization network default group ISE_GROUP
aaa accounting identity default start-stop group ISE_GROUP

! IP Device Tracking 設定
device-tracking binding recovery interval 30
ip device tracking probe delay 10

! ポスチャリダイレクト ACL (Deny=通過, Permit=リダイレクト)
ip access-list extended ACL_POSTURE_REDIRECT
 deny udp any any eq domain
 deny tcp any any eq domain
 deny udp any any eq bootps
 deny udp any any eq bootpc
 deny ip any host 10.1.100.10
 permit tcp any any eq www
 permit tcp any any eq 443

! インターフェース設定
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
 authentication host-mode multi-auth
 authentication order dot1x mab
 authentication priority dot1x mab
 authentication port-control auto
 mab
 dot1x pae authenticator
 access-session host-mode multi-auth
 access-session closed
 access-session port-control auto
```

#### 検証方法
- 端末接続後、`show access-session interface Gi1/0/1 details` を実行。
- `URL Redirect ACL: ACL_POSTURE_REDIRECT` および URL Redirect パラメータがバインドされていることを確認。
- `show ip access-lists ACL_POSTURE_REDIRECT` でパケットのヒットカウントが増加していることを確認。


---

### 【演習 2】ISE Policy Set における Posture Unknown / Compliant / Non-Compliant の認可ルール設計

#### 問題
Cisco ISE の Policy Sets において、有線 LAN 接続（802.1X/MAB）端末に対するポスチャ状態に応じた動的認可ルール（Authorization Policy）を設定しなさい。

#### 要件
1. Rule 1 (Posture Unknown Rule):
   - 条件: Wired_Network_Access AND Session:PostureStatus EQUALS Unknown
   - 認可結果: Profile_Posture_Redirect (Web Redirection: Posture Discovery, ACL: ACL_POSTURE_REDIRECT)
2. Rule 2 (Posture Non-Compliant Rule):
   - 条件: Wired_Network_Access AND Session:PostureStatus EQUALS Non_Compliant
   - 認可結果: Profile_Quarantine (dACL: DACL_QUARANTINE_DENY_ALL_INTERNAL)
3. Rule 3 (Posture Compliant Rule):
   - 条件: Wired_Network_Access AND Session:PostureStatus EQUALS Compliant
   - 認可結果: PermitAccess (SGT: Employee_16)

#### 設定例 (ISE GUI 設定パラメタ表現)

```text
Policy Set: Wired_Corporate_Policy
  Policy Conditions: Radius:NAS-Port-Type EQUALS Ethernet

  [Authorization Policy Rules]
  
  Rule 1: Posture_Unknown_Redirect
    - Status: Enabled
    - Conditions: 
        Session:PostureStatus EQUALS Unknown
    - Permissions (Authorization Profile):
        Profile_Posture_Redirect

  Rule 2: Posture_NonCompliant_Quarantine
    - Status: Enabled
    - Conditions: 
        Session:PostureStatus EQUALS Non_Compliant
    - Permissions (Authorization Profile):
        Profile_Quarantine

  Rule 3: Posture_Compliant_FullAccess
    - Status: Enabled
    - Conditions: 
        Session:PostureStatus EQUALS Compliant
    - Permissions (Authorization Profile):
        PermitAccess
```

#### 検証方法
- エンドポイント接続初期に ISE Live Logs にて Posture_Unknown_Redirect ルールにマッチし、Posture Status が Unknown となっていることを確認。
- エージェント評価成功後、再認証ログが発生し Posture_Compliant_FullAccess ルールにマッチし Posture Status が Compliant に昇格していることを確認。


---

### 【演習 3】AnyConnect Posture Profile XML の設計とディスカバリ設定

#### 問題
AnyConnect ISE Posture Module が ISE PSN を正確に特定・接続するための AnyConnect Posture Profile XML を作成し、全自動ディスカバリを実現しなさい。

#### 要件
1. Discovery Host に ISE PSN の FQDN `ise-psn.example.com` を指定。
2. ポスチャディスカバリポートを `8905` に指定。
3. サーバー名検証ルールに `*.example.com` を許可設定。
4. 接続失敗時の再試行インターバルを `10秒` に設定。

#### 設定例 (ISEPostureProfile.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<PostureConfiguration version="1.0">
    <ClientAgent>AnyConnect</ClientAgent>
    <PostureState>Enabled</PostureState>
    <DiscoveryHost>ise-psn.example.com</DiscoveryHost>
    <ServerNameRules>*.example.com</ServerNameRules>
    <PostureAgentDiscoveryPort>8905</PostureAgentDiscoveryPort>
    <DiscoveryTimeout>10</DiscoveryTimeout>
    <EnableAgentRecheck>true</EnableAgentRecheck>
    <AgentRecheckInterval>14400</AgentRecheckInterval>
    <RemediationTimer>1800</RemediationTimer>
</PostureConfiguration>
```

#### 検証方法
- クライアント端末のローカル設定フォルダ内に `ISEPostureProfile.xml` が正常に配信されているか確認。
- AnyConnect UI の Posture ログ画面において、Discovery ログおよび ISE Host Identified の出力を確認。


---

### 【演習 4】アンチウイルス定義ファイル更新チェックおよび自動修復ポスチャルールの構築

#### 問題
Cisco ISE ポスチャポリシーにおいて、Windows 端末上のアンチウイルスソフトウェア（例: Windows Defender）が有効であり、かつ定義ファイルが過去7日以内に更新されていることを検証するポリシーを作成しなさい。

#### 要件
1. 対象OS: Windows All
2. 条件 1 (AV Installed): Antivirus_Installed_Condition (すべてのメジャーAV対応)
3. 条件 2 (AV Definition Status): AV_Def_Age_7Days (定義ファイルの最新経過日数 <= 7日)
4. 修復アクション: 定義ファイルが古い場合、自動更新プロセスの開始要求メッセージ画面を提示し、修復サイトへのアクセスを許可すること。

#### 設定例 (ISE GUI ポスチャ要件定義)

```text
1. Posture Condition の作成:
   - Condition Name: Check_AV_Def_Age
   - Type: Antivirus Condition
   - Check Type: Definition Date
   - Operator: Less Than or Equal To
   - Value: 7 Days

2. Posture Requirement の作成:
   - Requirement Name: Req_AV_UpToDate
   - Operating System: Windows All
   - Conditions: Check_AV_Def_Age
   - Remediation Actions: AV_Auto_Update_Action (Message: "Your AV definitions are out of date. Updating...")

3. Posture Policy の作成:
   - Rule Name: Windows_AV_Policy
   - Target OS: Windows All
   - Groups: Any
   - Requirements: Req_AV_UpToDate
```

#### 検証方法
- 7日以上定義ファイルが更新されていないテスト端末で接続を試行。
- AnyConnect 画面上に修復要請ポップアップが表示され、ポスチャ状態が Non-Compliant となることを確認。
- 定義ファイル更新後、「Re-scan」を実行し Compliant へ遷移することを確認。

---

### 【演習 5】WLC (Catalyst 9800) における Central Web Authentication (CWA) と Posture リダイレクトの構築

#### 問題
無線 LAN コントローラ Cisco Catalyst 9800 (IOS-XE) において、ISE ポスチャ評価のための CWA / ポスチャリダイレクト WLAN および RADIUS 設定を構成しなさい。

#### 要件
1. WLAN SSID: Corporate-Wi-Fi、Security: 802.1X (WPA2-Enterprise)
2. ISE RADIUS Server: 10.1.100.10 (Key: CiscoLab123!)
3. RADIUS CoA (UDP 1700) 有効化。
4. AAA Advanced Attributes で AAA Override および Change of Authorization (CoA) を有効化。
5. ポスチャリダイレクト ACL `ACL_WLAN_POSTURE_REDIRECT` を作成。

#### 設定例 (Catalyst 9800 CLI)

```text
! RADIUS サーバー設定
radius server ISE_PSN
 address ipv4 10.1.100.10 auth-port 1812 acct-port 1813
 key CiscoLab123!

aaa group server radius ISE_WLAN_GROUP
 server name ISE_PSN

aaa server radius dynamic-author
 client 10.1.100.10 server-key CiscoLab123!
 port 1700

! WLAN ポスチャリダイレクト ACL
ip access-list extended ACL_WLAN_POSTURE_REDIRECT
 deny udp any any eq domain
 deny tcp any any eq domain
 deny udp any any eq bootps
 deny udp any any eq bootpc
 deny ip any host 10.1.100.10
 permit tcp any any eq www
 permit tcp any any eq 443

! WLAN プロファイル設定
wlan Corporate-Wi-Fi 1 Corporate-Wi-Fi
 security ft
 security wpa wpa2 ciphers aes
 security wpa akm dot1x
 no shutdown

! Policy Profile 設定
wireless profile policy Policy_Corporate
 aaa-override
 nac
 redirect-acls
 service-policy input QOS_TRUST
 vlan 100
 no shutdown

! Policy Tag マッピング
wireless tag policy PolicyTag_Corporate
 wlan Corporate-Wi-Fi policy Policy_Corporate
```

#### 検証方法
- クライアントを SSID `Corporate-Wi-Fi` に接続。
- C9800 で `show wireless client mac-address <MAC> detail` を実行。
- `URL Redirect ACL: ACL_WLAN_POSTURE_REDIRECT` および URL Redirect パラメータがバインドされていることを確認。


---

### 【演習 6】AnyConnect Temporal Agent を用いた一時型ポスチャ評価の構築

#### 問題
永続エージェントがインストールされていない一括接続端末や未管理端末に対し、Web 経由で一時的エージェント (Temporal Agent) をダウンロード・起動させてポスチャ評価を行う設定を構成しなさい。

#### 要件
1. Client Provisioning Policy において、AnyConnect Permanent Agent 未検出端末に対し `AnyConnect Temporal Agent` パッケージを割り当てる。
2. Temporal Agent の動作モード: ポスチャチェック完了後にメモリから自己消去する設定とすること。
3. チェック項目: Windows パーソナルファイアウォールが有効化されていること。

#### 設定例 (ISE Client Provisioning & Posture Policy)

```text
1. Client Provisioning Resource の登録:
   - Resource Type: AnyConnect Temporal Agent Package
   - File: AnyConnect-TemporalAgent-win.exe

2. Client Provisioning Policy の定義:
   - Rule Name: TempAgent_Provisioning
   - Operating System: Windows All
   - User Groups: Guest_Users, Contractor_Group
   - Results: AnyConnect Temporal Agent Profile

3. Posture Policy:
   - Rule Name: TempAgent_Firewall_Check
   - Condition: Windows_Firewall_Enabled
   - Requirement: Req_Firewall_On
```

#### 検証方法
- 未管理 Windows 端末で Web ブラウザを開き、ポスチャリダイレクトポータルにアクセス。
- ブラウザ上で Temporal Agent の自動実行が開始され、ポスチャチェック完了後にプロセスが正常終了して接続が昇格することを確認。

---

### 【演習 7】Agentless Posture (WMI/WinRM) によるドメイン参加チェック

#### 問題
クライアントエージェントを一切配備できない環境において、ISE から Active Directory ドメイン参加端末へ WMI/WinRM 接続を行い、Agentless ポスチャ評価を実施しなさい。

#### 要件
1. エージェントレス認証プロトコル: WMI (Windows Management Instrumentation) / WinRM
2. 接続認証資格情報: AD ドメインサービスアカウント `domain\ise_agentless_svc`
3. 検証条件: 端末がドメイン `example.com` に正常に参加していること、および特定のシステムサービス `RemoteRegistry` が起動していること。

#### 設定例 (ISE Agentless Posture 設定)

```text
1. Active Directory Domain Service Account 設定:
   - Admin Center > System > Secure Access Control > Credentials
   - User: ise_agentless_svc
   - Role: Domain Admin / Local Admin on Endpoints

2. Agentless Posture Policy 設定:
   - Protocol: WinRM (HTTPS 5986)
   - Condition 1: Domain_Joined EQUALS example.com
   - Condition 2: Service_Running EQUALS RemoteRegistry
   - Posture Result: Compliant / Non-Compliant
```

#### 検証方法
- エンドポイント接続時、ISE ログにて `Agentless Posture Triggered` が発生し、WMI/WinRM 経由でステータスを取得して Compliant に切り替わることを確認。

---

### 【演習 8】Cisco ASA VPN 接続における AnyConnect Posture と Dynamic Authorization

#### 問題
Cisco ASA 9.x において、AnyConnect Remote Access VPN ユーザーのポスチャ評価を実施し、未評価時は限定アクセス、適合時は完全アクセスを割り当てる設定を構成しなさい。

#### 要件
1. ASA Group Policy: `GP_VPN_POSTURE`
2. ISE RADIUS 認証・認可および CoA (UDP 1700) を有効化。
3. リダイレクト ACL: `ACL_ASA_POSTURE_REDIRECT`
4. Posture Unknown 時の ASA 内部 Filter: `DACL_VPN_POSTURE_TEMP` (DNS, ISE への通信のみ許可)。

#### 設定例 (Cisco ASA CLI)

```text
! RADIUS サーバーグループ定義
aaa-server ISE_GROUP protocol radius
aaa-server ISE_GROUP (inside) host 10.1.100.10
 key CiscoLab123!
 authentication-port 1812
 accounting-port 1813
 dynamic-authorization port 1700

! VPN リダイレクト ACL (Deny=通過, Permit=リダイレクト)
access-list ACL_ASA_POSTURE_REDIRECT extended deny udp any any eq domain
access-list ACL_ASA_POSTURE_REDIRECT extended deny ip any host 10.1.100.10
access-list ACL_ASA_POSTURE_REDIRECT extended permit tcp any any eq www
access-list ACL_ASA_POSTURE_REDIRECT extended permit tcp any any eq 443

! Group Policy 設定
group-policy GP_VPN_POSTURE internal
group-policy GP_VPN_POSTURE attributes
 dns-server value 10.1.100.1
 vpn-tunnel-protocol ssl-client
 default-domain value example.com

! Tunnel Group 設定
tunnel-group TG_VPN_POSTURE type remote-access
tunnel-group TG_VPN_POSTURE general-attributes
 authentication-server-group ISE_GROUP
 authorization-server-group ISE_GROUP
 accounting-server-group ISE_GROUP
 default-group-policy GP_VPN_POSTURE
```

#### 検証方法
- AnyConnect VPN 接続後、ASA 上で `show vpn-sessiondb detail anyconnect` を実行。
- RADIUS 属性 `url-redirect-acl` および `url-redirect` が正常に適用され、ポスチャ完了後に CoA によりアクセスポリシーが更新されることを確認。

---

### 【演習 9】Posture Grace Period (猶予期間) と動的アクセスポリシーの統合

#### 問題
企業組織で新しいセキュリティポリシー（例: 特定の修正プログラム KB9876543）を導入する際、未適用端末を即座に隔離せず、3日間の猶予期間 (Grace Period) を与えて業務アクセスを維持させつつ修復を促すポリシーを設定しなさい。

#### 要件
1. 猶予期間: 3 Days
2. Grace Period 中のアクセスレベル: `Profile_Grace_Access` (限定社内リソースのみ許可)
3. 猶予期間超過後のアクセスレベル: `Profile_Quarantine` (社内アクセス完全遮断)

#### 設定例 (ISE GUI 設定)

```text
1. Posture Policy 設定:
   - Policy Rule: Win_Patch_Rule
   - Condition: KB9876543_Applied
   - Grace Period: Enabled (Duration: 3 Days)

2. Authorization Policy Rules:
   - Rule Grace: 
       Condition: Session:PostureStatus EQUALS InGracePeriod
       Result: Profile_Grace_Access (dACL: DACL_LIMITED_INTERNAL)
   - Rule NonCompliant:
       Condition: Session:PostureStatus EQUALS Non_Compliant
       Result: Profile_Quarantine (dACL: DACL_DENY_ALL_INTERNAL)
```

#### 検証方法
- パッチ未適用端末で接続し、ISE 上のステータスが `InGracePeriod` となり限定社内アクセスが許可されることを確認。
- 3日超過後またはテストで猶予期限を切断後、`Non_Compliant` へ遷移し完全拒否されることを確認。

---

### 【演習 10】Security Group Tag (SGT) と ポスチャ状態の統合セグメンテーション

#### 問題
スイッチおよび ISE において、ポスチャ状態に基づいて SGT を動的に割り当て、Cisco TrustSec インフラストラクチャ上でマイクロセグメンテーションを実現しなさい。

#### 要件
1. Posture Status `Compliant` 時の SGT: `16` (`Compliant_Host`)
2. Posture Status `Non-Compliant` / `Unknown` 時の SGT: `15` (`Quarantine_Host`)
3. SGT インラインタギングを Catalyst Switch ポート `Gi1/0/1` で有効化。

#### 設定例 (Switch CLI & ISE Profile)

```text
! Switch CLI (CTS 有効化)
cts status
cts logging verbose

interface GigabitEthernet1/0/1
 cts manual
  policy static sgt 15 --default--

! ISE Authorization Profile 設定
Authorization Profile 1: Profile_Compliant_SGT
  - Security Group Tag: Compliant_Host (16)
  - Access-Type: ACCESS_ACCEPT

Authorization Profile 2: Profile_Unknown_SGT
  - Security Group Tag: Quarantine_Host (15)
  - Access-Type: ACCESS_ACCEPT
  - Advanced Attributes: cisco-av-pair = url-redirect-acl=ACL_POSTURE_REDIRECT
```

#### 検証方法
- 端末接続時、`show cts interface Gi1/0/1` を実行し、初期状態として `SGT 15` が付与されていることを確認。
- ポスチャ完了後、CoA により `SGT 16` に動的更新されることを確認。


---

## ❓ 想定試験問題

CCIE Security ラボ試験・記述試験を想定した実践問題5選。

### 問題 1 (トラブルシューティング)
**質問**:
スイッチ SW1 に接続した Windows 端末において、AnyConnect Posture Module が起動しているにもかかわらず、「ISE Not Found」 というエラーが表示されポスチャ評価が進行しない。`show access-session interface Gi1/0/1 details` を確認したところ、`URL Redirect ACL: ACL_POSTURE` が正常に適用されている。このトラブルの根本原因として最も可能性が高いものを2つ挙げ、その対処方法を説明しなさい。

**解答・解説**:
1. **原因 1: リダイレクト ACL 内で DNS (UDP/TCP 53) 通信が Permit（リダイレクト対象）になっていること**
   - **理由**: AnyConnect は Discovery Host 名の FQDN 解決を行うため DNS クエリを発行する。DNS がリダイレクトされると名前解決に失敗し、ISE PSN の特定が不能となる。
   - **対処法**: リダイレクト ACL の最上部に `deny udp any any eq domain` および `deny tcp any any eq domain` を追加し、DNS 通信を無条件通過させる。
2. **原因 2: ISE PSN IP アドレス（8905ポート）への通信がリダイレクト ACL で拒否（リダイレクト対象）されていること**
   - **理由**: AnyConnect ディスカバリプローブ (UDP/TCP 8905) が ISE PSN に到達できずリダイレクトループに陥る。
   - **対処法**: リダイレクト ACL に `deny ip any host <ISE_PSN_IP>` を追加し、ISE PSN 宛の直接通信を許可する。

---

### 問題 2 (コンフィグ読解)
**質問**:
以下の Cisco Catalyst スイッチの設定において、ポスチャ評価完了後に ISE から送出された RADIUS CoA パケットがスイッチで拒否された。コンフィグ内の不備を指摘し、正しい修正コマンドを記述しなさい。

```text
aaa new-model
aaa group server radius ISE_GROUP
 server name ISE_1
radius server ISE_1
 address ipv4 10.1.100.10 auth-port 1812 acct-port 1813
 key CiscoSecret123

aaa server radius dynamic-author
 client 10.1.100.10
```

**解答・解説**:
- **不備**: `aaa server radius dynamic-author` 設定ブロック内において、ISE PSN (10.1.100.10) に対する共有鍵 (`server-key`) が設定されていない。スイッチは CoA パケットの認証（Authenticator 計算）に失敗し、CoA パケットをドロップする。
- **修正コマンド**:
```text
aaa server radius dynamic-author
 client 10.1.100.10 server-key CiscoSecret123
 port 1700
```

---

### 問題 3 (Design / アーキテクチャ)
**質問**:
企業ネットワークにおいて、管理対象の Windows 端末（永続エージェント配備済み）と、ゲスト/サードパーティベンダーの未管理端末が同一の 802.1X/MAB 有線ポートに接続する。両方の端末群に対してポスチャ評価を実施したい場合、Cisco ISE の Client Provisioning Policy をどのように設計すべきか説明しなさい。

**解答・解説**:
1. **Client Provisioning Policy の階層化設計**:
   - **ルール 1 (Managed Corporate Devices)**:
     - 条件: AD ドメイン参加済み端末、または AnyConnect 検出済みグループ。
     - 結果: AnyConnect ISE Posture Profile (既存の AnyConnect Agent を利用してポスチャ実施)。
   - **ルール 2 (Unmanaged / Guest Devices)**:
     - 条件: AnyConnect 未検出、非ドメイン参加端末。
     - 結果: AnyConnect Temporal Agent パッケージを割り当て。
2. **動作メカニズム**:
   - ゲスト端末が Web ブラウザを起動すると、ポスチャリダイレクトポータルに移動。Web 画面から一時実行型エージェント (Temporal Agent) がダウンロード・実行され、メモリモジュール上で AV/Firewall 等のポスチャチェックが行われた後、自動消去される。

---

### 問題 4 (トラブルシューティング)
**質問**:
AnyConnect エージェントによるポスチャ評価が成功し、AnyConnect 画面上では 「System is Compliant」 と表示されたが、端末が社内サーバーにアクセスできず、スイッチ上のセッションも未評価制限状態のままである。トラブルシューティング手順と確認すべきポイントを述べよ。

**解答・解説**:
1. **原因の追究**:
   - エージェントと ISE 間の評価は完了しているが、ISE から スイッチ (NAD) への RADIUS CoA 処理が失敗しているか、再認証後の ISE 認可ルール選択に誤りがある。
2. **確認手順**:
   - **ステップ 1 (ISE Live Logs)**: Operations > RADIUS > Live Logs で CoA Status を確認。CoA Failed や CoA Timeout が発生していないか確認。
   - **ステップ 2 (NAD 側の CoA 受信確認)**: スイッチ側で `show aaa servers` を確認し、CoA リクエスト受信数 (CoA Executed / CoA Requests) を確認。
   - **ステップ 3 (認可ルールの評価順序)**: 昇格後の RADIUS Access-Request に対して、ISE の認可ポリシーで `Session:PostureStatus EQUALS Compliant` ルールが最上位に位置し、正しくバインドされているか確認。

---

### 問題 5 (実装・ポリシー構築)
**質問**:
Cisco ISE ポスチャポリシーにおいて、端末が「Non-Compliant」と判定された際、エンドポイントの通信を特定修復サーバー（10.1.100.20）および DNS/DHCP のみに制限する Quarantine dACL を作成しなさい。

**解答・解説**:
**dACL 名**: `DACL_QUARANTINE_RESTRICTED`
**設定内容**:
```text
permit udp any any eq domain
permit udp any any eq bootps
permit udp any any eq bootpc
permit ip any host 10.1.100.20
deny ip any any
```

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよびテクニカルノート：

* Cisco ISE Configuration Guide - Posture Assessment Services
* Cisco AnyConnect / Secure Client Posture Module Configuration Guide
* Cisco Technical Note: Configure ISE 3.x Posture with AnyConnect and Catalyst Switch
* Cisco Live Presentation: BRKSEC-3432 - Advanced ISE Posture Architecture and Deployment
* Cisco Validated Design (CVD): Campus Security Posture Deployment Guide

---

## 📝 **補足（Notes）**

### 学習メモ・設計上の注意点
- **リダイレクト ACL の設計原理**:
  - スイッチ / WLC のリダイレクト ACL は「**Permit = リダイレクト対象**」、「**Deny = バイパス（通過許可）**」という逆転論理であることを常に叩き込むこと。試験本番で逆に設定すると、接続が遮断されるか無限リダイレクトループが発生する。
- **DHCP と IP Device Tracking (SISF)**:
  - 802.1X 認証直後に AnyConnect が IP アドレスを保持していない場合、ディスカバリが失敗する。スイッチポートには必ず device-tracking 設定を入れて IP アドレスと MAC アドレスのバインディングテーブルを早期確立させること。
- **証明書の信頼関係**:
  - AnyConnect ディスカバリおよび Web リダイレクト時、ISE PSN の Admin / Portal 証明書がクライアント端末で信頼されている（ルート CA 証明書が配置されている）必要がある。信頼されていない場合、AnyConnect ポスチャモジュールが SSL/TLS エラーを発行して接続を中断することがある。


