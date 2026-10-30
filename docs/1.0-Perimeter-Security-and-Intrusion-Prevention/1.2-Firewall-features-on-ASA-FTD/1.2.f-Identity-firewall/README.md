---
layout: default
title: 1.2.f-Identity-firewall
nav_order: 6
parent: 1.2-Firewall-features-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.2.f Identity firewall

Cisco ASA (Adaptive Security Appliance) および Cisco FTD (Firepower Threat Defense) における **Identity Firewall (IDFW: アイデンティティファイアウォール)** は、従来の IP アドレスや MAC アドレスベースの静的なアクセス制御から脱却し、**「誰が (Who)」「どの端末で (Which Device)」「どのグループ権限で (Which Group)」** ネットワークにアクセスしているかに基づいて動的なセキュリティポリシーを適用する技術です。

Active Directory (AD) のログオン情報や Cisco ISE (Identity Services Engine) のコンテキストデータ（pxGrid）と連携することで、DHCP 環境や VDI（仮想デスクトップ）、モビリティ環境において IP アドレスが動的に変動しても、一貫したアクセス制御と高度な監査・トラフィック可視性を実現します。

---

## 📘 概要

* **機能概要**:
  * **パッシブアイデンティティ (Passive Identity)**: ユーザーに手動認証を強制することなく、Windows Active Directory のドメインログオンイベント (Event ID 4624 等) を AD Agent, CDA (Context Directory Agent), または ISE (pxGrid/WMI/Syslog) 経由でバックグラウンド収集し、IP アドレスとユーザー/グループ名のマッピングテーブル（IP-to-User/Group Binding）をファイアウォール上に自動生成・同期します。
  * **アクティブアイデンティティ (Active Identity / Captive Portal / Cut-Through Proxy)**: AD ログオン情報が存在しない未認証のユーザーやドメイン外端末に対して、HTTP/HTTPS トラフィックをキャプティブポータル（WebAuth）や Cut-Through Proxy 画面へ自動リダイレクトし、手動でユーザー名・パスワードを入力させて認証を行います。
  * **アイデンティティベース ACL / ACP**: アクセス制御ルール（ACL / Access Control Policy）の送信元条件に、IP サブネットではなく `domain\username` や `domain\user-group` オブジェクトを指定してフィルタリングを実行します。
* **利用目的**:
  * IP アドレスと物理位置に依存しない、ロールベース（職責・部署ベース）アクセス制御の実現。
  * セキュリティログや Syslog 内の IP アドレス表示を、ユーザー名 (`domain\user1`) に透過的に置換し、セキュリティインシデント発生時のトラッキングと監査効率を飛躍的に向上させる。
  * ゼロトラストネットワークアクセス (ZTNA) や BYOD / ゲストアクセスの境界セキュリティ強化。
* **どのような場面で利用するか**:
  * ユーザーが社内 LAN、Wi-Fi、VDI、AnyConnect RAVPN などの異なるアクセスチャネル間を移動し、IP アドレスが頻繁に変更される企業ネットワーク。
  * 同一 VLAN / 同一 IP サブネット内で、人事部・開発部・一時派遣社員などの異なるアクセス権限を混在させて制御する場合。

---

## 🔑 要点

| 項目 | Cisco ASA Identity Firewall (IDFW) | Cisco FTD Identity Policy (FMC Managed) |
| :--- | :--- | :--- |
| **アイデンティティソース** | AD Agent (RADIUS) / CDA / LDAP (グループ検索) | **Cisco ISE (pxGrid)** / PassiveID Agent / Realm (LDAP) |
| **ポリシー適用単位** | `object-group user` ＋ `access-list` | **Identity Policy** ＋ Access Control Policy (User Condition) |
| **パッシブマッピング取得** | AD Agent から RADIUS 経由で受信 | ISE から pxGrid 経由、または FMC PassiveID 経由で受信 |
| **グループメタデータ取得** | LDAP / LDAPS による AD 直接クエリ | Realm (LDAP/LDAPS) による AD 直接クエリ |
| **手動認証 (Active Auth)** | Cut-Through Proxy (AAA Authentication Match) | Captive Portal (Internal/External Web Portal) |
| **フォールバック動作** | `action domain-controller-down` / `ad-agent-down` | Identity Policy 内の Auth Failure / Timeout アクション指定 |
| **SGT (TrustSec) 連携** | SXP (Security Group Tag Exchange Protocol) 連携 | pxGrid 経由での SGT 取得・ACP マッチ |
| **ドメイン名構文要件** | **NetBIOS ドメイン名 (大文字小文字区別)** | Realm 内で設定した Domain Name / NetBIOS Name |

---

## 🏗 動作原理

Identity Firewall は、「アイデンティティ情報の収集 (Learn)」「グループ属性の検索 (Lookup)」「アクセスポリシーの評価 (Enforce)」の 3 つの主要コンポーネントで動作します。

```text
【Passive Identity Firewall (AD Agent / ISE pxGrid 連携) の通信フロー】

[ Client Workstation ]           [ Active Directory ]          [ Identity Agent / ISE ]         [ ASA / FTD Firewall ]
         │                               │                                │                               │
         │ ── 1. Windows Domain Login ──►│                                │                               │
         │    (Kerberos / NTLM)          │                                │                               │
         │                               │ ── 2. Audit Log Generated ────►│                               │
         │                               │    (Event ID 4624: IP + User)  │                               │
         │                               │                                │ ── 3. IP-to-User Mapping ────►│
         │                               │                                │    (RADIUS / pxGrid Push)     │
         │                               │                                │                               │
         │                               │◄────── 4. Query Group Membership (LDAP) ───────────────────────│
         │                               │──────── 5. Return Groups (e.g., Domain\Finance) ──────────────►│
         │                               │                                │                               │
         │ ── 6. Data Packet (Src: Client IP) ───────────────────────────────────────────────────────────►│
         │                               │                                │                               │ 7. Lookup IP in Mapping Cache
         │                               │                                │                               │ 8. Match ACL/ACP: User/Group
         │ ◄── 9. Forwarded / Permitted ──────────────────────────────────────────────────────────────────│
```

---

## ⚙ 動作シーケンス

1. **ユーザーログオンとイベントログ発生**:
   * クライアント端末が Windows ドメインコントローラ (DC) にログインし、Kerberos / NTLM 認証が完了すると、DC 上にログインセキュリティイベント (Event ID 4624) が記録されます。
2. **IP-to-User マッピングの収集 (Passive ID)**:
   * **ASA (AD Agent / CDA)**: AD Agent が WMI (Windows Management Instrumentation) や Event Log API を使用して DC のログを監視し、`IP: 10.1.1.50 <-> User: INELAB\USER1` のバインディング情報を生成して ASA へ通知します。
   * **FTD (ISE pxGrid)**: ISE の PassiveID ワークセンターが AD や Agent からバインディングを取得し、pxGrid パブリッシュ/サブスクライブバスを介して FMC / FTD へリアルタイムプッシュします。
3. **AD グループ属性の同期 (LDAP Lookup)**:
   * ASA / FTD は、設定された LDAP サーバー構成（ASA: `aaa-server` / FTD: `Realm`）を使用して AD コントローラへ自らクエリを送信し、対象ユーザーが所属するセキュリティグループ（例: `CN=Finance_Grp,OU=Groups,DC=inelab,DC=local`）を取得・キャッシュします。
4. **接続発生とパケットの評価**:
   * 端末からデータパケットが着信すると、ファイアウォールは送信元 IP アドレスをキーにして内部のアイデンティティマッピングキャッシュ（Identity Cache / uauth）を検索します。
5. **アクセス制御ルール (ACL / ACP) の適用**:
   * マッチしたユーザー名および所属グループ名を、適用されているアクセス制御ルール（ASA `access-list` / FTD `Access Control Policy`）の `user` / `user-group` 条件と照合します。
6. **ログとセッションの更新・削除**:
   * 通信ログ (Syslog / Connection Events) に IP アドレスだけでなくユーザー名が出力されます。ユーザーのログオフや非アクティブタイマーの到達、DC の応答停止時には動的マッピングが自動消去されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **NetBIOS ドメイン名の指定要件 (大文字小文字の区別)**:
   * ASA の `object-group user` や CLI 設定でドメイン名を定義する場合、**FQDN (`inelab.local`) ではなく、必ず大文字の NetBIOS ドメイン名 (`INELAB`)** を使用しなければなりません。
   * 例: `user INELAB\USER1` や `user-group INELAB\\CCIELAB`（バックスラッシュの重複エスケープに注意）。小文字で設定すると AD Agent や LDAP 検索とバインドせず、ルールがマッチしません。
2. **`user-identity monitor` (オンデマンド監視) の役割**:
   * デフォルトでは、ASA は**アクティブな ACL に含まれているユーザー/グループの情報のみ**を AD Agent からダウンロードします。
   * 問題文で「ACL で未定義のグループ情報を事前に AD Agent から取得・監視せよ」または「Cloud Web Security (CWS) 連携用に特定グループを強制モニターせよ」と指定された場合、**`user-identity monitor user-group <DOMAIN\Group>`** を明示的に設定する必要があります。
3. **ドメインコントローラ / AD Agent 障害時のフォールバック設定**:
   * 試験要件で「AD Agent または Domain Controller がダウンした際、セキュリティを最優先してアクセスを拒否せよ」または「業務継続のためルールをバイパスせよ」と問われます。
   * `user-identity action domain-controller-down <DOMAIN> disable-user-identity-rule` (ルール無効化 = デフォルト拒否へフォールバック) や `remove-user-identity-rule` の使い分けをマスターしておく必要があります。
4. **FTD (FMC) における Realm と Identity Policy の紐付け**:
   * FMC で Identity 制御を行う場合、単に **System > Integration > Realms** で AD を登録・有効化するだけでは機能しません。
   * 必ず **Policies > Identity** で Identity Policy を作成して Realm をバインドし、それを **Access Control Policy (ACP)** の `Identity Policy` フィールドへ適用した上で、ACP ルールの `Users` タブにグループを指定する必要があります。
5. **Cut-Through Proxy (Active Auth) と HTTP/HTTPS 仮想 IP の構成**:
   * ASA で手動認証を行う Cut-Through Proxy を構成する場合、`virtual http <IP>` や `virtual telnet <IP>` を設定し、`aaa authentication match` コマンドでトラフィックをキャプチャして認証ポータルへ誘引します。

---

## 🛠 設定方法

### 1. Cisco ASA: AD Agent ＋ LDAP による Passive Identity Firewall 設定 (CLI)

```bash
! ===== 1. LDAP サーバーグループの構成 (AD グループ検索用) =====
aaa-server AD-LDAP protocol ldap
aaa-server AD-LDAP (inside) host 172.16.10.100
 server-port 389
 ldap-base-dn DC=inelab,DC=local
 ldap-scope subtree
 ldap-naming-attribute sAMAccountName
 ldap-login-from-dn CN=ldapuser,OU=Users,DC=inelab,DC=local
 ldap-login-password 1qaz@WSX
 ldap-over-ssl disable

! ===== 2. AD Agent サーバーグループの構成 (RADIUS 通信) =====
aaa-server AD-AGENT-GRP protocol radius
aaa-server AD-AGENT-GRP (inside) host 172.16.10.100
 key CISCO_SECRET
 authentication-port 1645
 accounting-port 1646

! ===== 3. User-Identity グローバルパラメータの設定 =====
user-identity enable
user-identity default-domain INELAB
user-identity domain INELAB aaa-server AD-LDAP
user-identity ad-agent aaa-server AD-AGENT-GRP

! タイマーおよび障害時アクションの設定
user-identity inactive-user-timer minutes 30
user-identity poll-import-user-group-timer hours 1
user-identity action domain-controller-down INELAB disable-user-identity-rule
user-identity action ad-agent-down disable-user-identity-rule

! ===== 4. ユーザーオブジェクトグループとアクセスルールの作成 =====
object-group user OG_IDENTITY_USERS
 user INELAB\USER1
 user-group INELAB\\CCIELAB

access-list INSIDE_INBOUND extended permit ip object-group-user OG_IDENTITY_USERS any any
access-group INSIDE_INBOUND in interface inside
```

### 2. Cisco ASA: Cut-Through Proxy (Active Authentication) 設定 (CLI)

```bash
! ===== 1. 仮想 IP アドレスと AAA サーバの定義 =====
virtual http 192.168.100.1
virtual telnet 192.168.100.2

aaa-server ISE-RADIUS protocol radius
aaa-server ISE-RADIUS (inside) host 172.16.3.100
 key Secret123

! ===== 2. Cut-Through Proxy 認証トラフィックのキャプチャ =====
access-list AUTH_PROX_ACL extended permit tcp 10.1.0.0 255.255.0.0 host 10.2.0.10 eq 80
access-list AUTH_PROX_ACL extended permit tcp 10.1.0.0 255.255.0.0 host 10.2.0.10 eq 23

aaa authentication match AUTH_PROX_ACL inside ISE-RADIUS

! ===== 3. WebAuth 認証タイムアウトと認証プロキシローカルユーザーの設定 =====
aaa authentication include http inside 0.0.0.0 0.0.0.0 0.0.0.0 0.0.0.0 ISE-RADIUS
timeout uauth 01:00:00 absolute
```

### 3. Cisco FTD (FMC GUI): ISE pxGrid 連携 ＆ Active/Passive Identity Policy 構成

1. **FMC と ISE の pxGrid 接続設定**:
   * **System > Integration > Identity Sources > Identity Services Engine** へ移動。
   * **Primary Host Name/IP Address**: ISE PSN の FQDN (`ise.inelab.local`) を入力。
   * **pxGrid Server CA / MNT Server CA**: ISE のルート CA 証明書オブジェクトを選択。
   * **FMC Server Certificate**: FMC 用の pxGrid クライアント証明書（ISE CA から発行）を選択。
   * **Test** をクリックし、ISE 側で `Approve` して接続ステータスを Green (Online) にする。
2. **Active Directory Realm の作成**:
   * **System > Integration > Realms > Add Realm**。
   * **Name**: `INELAB_Realm`, **Type**: `Active Directory`, **Domain**: `inelab.local`。
   * **Directory AD Primary Host**: AD の IP (`172.16.10.100`), Port: `389`, Encryption: `STARTTLS` または `NONE`。
   * **Directory Username / Password**: AD バインド用アカウント資格情報を入力。
   * **User Search Base**: `DC=inelab,DC=local`。
   * **Save** 後、**User Download** タブで `Download Users and Groups` を実行し、トグルを **Enabled** にする。
3. **Identity Policy の作成と Access Control Policy への紐付け**:
   * **Policies > Identity > Add Identity Policy**: Name: `ID_Policy_Corp`。
   * **Add Rule**: Action: `Passive Authentication`, Realm: `INELAB_Realm`。
   * **Policies > Access Control > ACP 編集**:
     * **Identity Policy** ドロップダウンで `ID_Policy_Corp` を選択。
     * **Add Rule**: Action: `Allow`, **Users** タブを選択 ➔ Realm: `INELAB_Realm` から対象ユーザー/グループ（例: `CCIELAB`）を追加して **Deploy** 実行。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **ASA AD Agent の接続・登録ステータス確認** | ASA CLI | <code>show user-identity ad-agent</code> |
| **ASA アイデンティティキャッシュ (IP-to-User) 一覧表示** | ASA CLI | <code>show user-identity user</code> / <code>show uauth</code> |
| **ASA 監視対象 AD グループ一覧および LDAP ドウンロード状態確認** | ASA CLI | <code>show user-identity group</code> / <code>show user-identity ad-groups</code> |
| **ASA メモリ消費量・セッション数確認** | ASA CLI | <code>show user-identity memory</code> |
| **ASA AD Agent 通信テスト** | ASA CLI | <code>test aaa-server ad-agent IDENTITY</code> |
| **FTD (FMC) pxGrid 接続ステータス確認 (ISE GUI)** | ISE GUI | **Administration > pxGrid Services > Clients** |
| **FTD CLI ユーザーアイデンティティキャッシュ検索** | FTD CLI | <code>system support diagnostic-cli</code> ➔ <code>show user-identity user</code> |
| **FTD Real-time Debug (Identity フロー追跡)** | FTD CLI | <code>system support firewall-engine-debug</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **AD Agent が `Status: down` と表示される** | ASA/Agent 間の RADIUS 共有鍵不一致、または UDP 1645/1646 ブロック | ASA: <code>show user-identity ad-agent</code> | 共有鍵を再確認し、ファイアウォールで UDP 1645/1646/3799 を許可する。 |
| **ユーザーログインしても ASA の ACL にマッチしない** | ドメイン名に FQDN を使用している、または大文字小文字のミス | ASA: <code>show running-config object-group</code> | `object-group user` 内で大文字の NetBIOS 名 (`INELAB\User1`) を使用する。 |
| **特定グループのユーザー情報が ASA にダウンロードされない** | 当該グループがアクティブな ACL または `user-identity monitor` に未登録 | ASA: <code>show user-identity group</code> | `user-identity monitor user-group <DOMAIN\Group>` を追加設定する。 |
| **FTD FMC で pxGrid ステータスが `Offline` / `Failed` になる** | FMC クライアント証明書が ISE で未承認 (Pending) | ISE GUI: <code>pxGrid Services > Clients</code> | ISE の Clients 画面で FMC の `ia-fmc` / `st-fmc` クライアントを **Approve** する。 |
| **Cut-Through Proxy で認証画面が表示されない** | `virtual http` IP 宛ての通信がインバウンド ACL でドロップされている | ASA: <code>show access-list</code> | インバウンド ACL で `virtual http` IP 宛ての TCP 80 通信を事前に許可する。 |
| **VPN 接続端末で Passive Identity マッピングが重複・誤マッチする** | VPN 割り当て IP と LAN 物理 IP の競合、またはログオフ検知失敗 | ASA: <code>show user-identity user</code> | `inactive-user-timer` や WMI ログオフ検出プローブを構成する。 |

---

## ⚠ 制限事項

* **Cisco ASA IDFW の制限**:
  * ASA 上で監視・ロードできる AD グループ数の上限は最大 **512 グループ**（`user-identity monitor` 含む）。
  * ドメイン指定時、マルチドメイン環境でも主ドメインとの一貫性が必要（NetBIOS 名の重複不可）。
* **Cisco FTD Identity Policy の制限**:
  * FTD 単体（FDM 管理）での Passive Identity はサポートが限定され、高度な pxGrid 統合には FMC ＋ ISE が必須。
  * 非 Windows 端末（Linux, ワークステーション, 認証ダイアログ非対応デバイス）は Captive Portal または MAB / MAC-to-IP マッピングへのフォールバックが必要。
* **ライセンスおよびバージョン依存**:
  * ASA OS 8.4(2) 以降で IDFW をサポート。
  * FTD で ISE pxGrid 連携を行う場合は FMC 6.0+ および ISE 2.0+（pxGrid v2 は ISE 2.4+）が必須。

---

## 🔄 他技術との関連

* **1.2.a NAT**: IP-to-User マッピングは **NAT 適用前（実 IP アドレス）** をキーとして記録されます。内部端末が PAT 変調される前段の Ingress インターフェイスで評価されます。
* **1.2 Firewall Access Control**: ACL / ACP ルールの Match 条件に User/Group オブジェクトを組み込みます。
* **Cisco ISE / TrustSec (SGT)**: ISE は pxGrid 経由で Passive Identity (IP-to-User) と SGT (IP-to-SGT) の両方のバインディングを FTD / WSA へ共有します。
* **Cisco WSA (Web Security Appliance)**: WSA は ISE pxGrid または CDA-RADIUS を介して Passive Identity を受信し、透過的 Web フィルタリングを実施します。

---

## 🧩 比較表

### 1. Passive Identity 収集アーキテクチャの比較

| 比較項目 | Cisco AD Agent (Legacy) | Cisco Context Directory Agent (CDA) | Cisco ISE (PassiveID / pxGrid) |
| :--- | :--- | :--- | :--- |
| **動作プラットフォーム** | 独立した Windows Server アプリ | 独立した Linux 仮想アプライアンス | **ISE (PSN / PassiveID Persona)** |
| **AD ログ収集方式** | WMI / Windows API | WMI (WinRM) | **WMI, Syslog, SPAN, Agent, REST API** |
| **FW 連携プロトコル** | RADIUS (UDP 1645/1646) | CDA-RADIUS (プロプライエタリ) | **pxGrid (REST-PubSub / WebSockets)** |
| **拡張性・推奨度** | レガシー (小規模向け) | 移行推奨 (サポート終了系) | **標準推奨 (次世代エンタープライズ)** |

### 2. Active Authentication vs Passive Authentication

| 項目 | Active Authentication (Captive Portal / Cut-Through) | Passive Authentication (AD Agent / ISE PassiveID) |
| :--- | :--- | :--- |
| **ユーザー体験** | **手動介入が必要** (Web ポータルでログイン) | **完全透過 (Single Sign-On)** |
| **非ドメイン端末対応** | **対応可能** (ゲスト, BYOD, スマホ) | 非対応 (AD ドメイン参加必須) |
| **オーバーヘッド** | FW での HTTP リダイレクト処理負荷が高い | FW の処理負荷が極めて低い (キャッシュ照合のみ) |

---

## 💡 ベストプラクティス

1. **ISE pxGrid 自動承認 (Auto-Approve) の一時活用とセキュリティ確保**:
   * FTD と ISE の連携構築時、証明書承認エラーによる接続失敗を防ぐため、一時的に ISE の `Automatically approve new certificate-based accounts` を有効にしてテストし、接続完了後に手動承認モードに戻します。
2. **`inactive-user-timer` と `poll-import-user-group-timer` の最適化**:
   * モバイル端末の IP 再割り当てによるセキュリティホールを防ぐため、`inactive-user-timer` を 30 分程度に設定し、`poll-import-user-group-timer` を 1〜2 時間程度で同期させます。
3. **NetBIOS ドメイン名と LDAP DN の整合性検証**:
   * CLI / GUI で設定するすべてのユーザーオブジェクトにおいて、ドメイン名が AD コントローラの NetBIOS 名と完全に一致していることを `show user-identity user` で常時確認します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA: AD Agent + LDAP を使用した Passive IDFW の基本構成

#### 問題
ASA2 において、AD サーバー (`172.16.10.100`) および AD Agent (`172.16.10.100`) と連携して Identity Firewall を有効化せよ。ドメイン名は `INELAB`（NetBIOS 名）とし、ドメインコントローラが応答しない場合はルールを自動的に無効化（`disable-user-identity-rule`）させよ。

#### 要件
* LDAP Base DN: `DC=inelab,DC=local`
* AD Agent 共有キー: `CISCO`
* ASA CLI で設定すること。

#### 設定例

```bash
! 1. LDAP サーバー設定
aaa-server AD protocol ldap
aaa-server AD (VLAN26) host 172.16.10.100
 server-port 389
 ldap-base-dn DC=inelab,DC=local
 ldap-scope subtree
 ldap-naming-attribute sAMAccountName
 ldap-login-from-dn CN=ldapuser,OU=Users,DC=inelab,DC=local
 ldap-login-password 1qaz@WSX

! 2. AD Agent 設定
aaa-server IDENTITY protocol radius
aaa-server IDENTITY (VLAN26) host 172.16.10.100
 key CISCO
 authentication-port 1645
 accounting-port 1646

! 3. User Identity パラメータ設定
user-identity enable
user-identity default-domain INELAB
user-identity domain INELAB aaa-server AD
user-identity ad-agent aaa-server IDENTITY

user-identity inactive-user-timer minutes 30
user-identity poll-import-user-group-timer hours 1
user-identity action domain-controller-down INELAB disable-user-identity-rule
user-identity action ad-agent-down disable-user-identity-rule
```

#### 検証方法
* `show user-identity ad-agent` を実行し、`Status: up (registered)` であることを確認。
* `test aaa-server ad-agent IDENTITY` を実行し、`Ad-agent Successful` と出力されることを確認。

---

### 例2: ASA: `object-group user` と Identity ベース ACL の定義

#### 問題
ASA2 において、ユーザー `INELAB\USER1` および AD グループ `INELAB\CCIELAB` を含むオブジェクトグループ `FIREWALL` を作成し、`VLAN19` インターフェイスからの全通信を許可する Identity ACL を適用せよ。

#### 要件
* オブジェクトグループ名: `FIREWALL`
* バックスラッシュは適切にエスケープすること。

#### 設定例

```bash
object-group user FIREWALL
 user INELAB\USER1
 user-group INELAB\\CCIELAB

access-list VLAN19_INBOUND extended permit ip object-group-user FIREWALL any any
access-group VLAN19_INBOUND in interface VLAN19
```

#### 検証方法
* `show user-identity ad-groups INELAB filter CCIELAB` を実行し、AD グループが正常に認識されていることを確認。
* 対象ユーザーでログイン後、`show user-identity user` で IP アドレスとユーザー名のマッピングが保持されていることを確認。

---

### 例3: ASA: Cut-Through Proxy (Telnet 経由) と RADIUS 認証・動的 ACL 付与

#### 問題
ASA3 において、VLAN 17 背後のユーザーが外部ルータ (`136.1.37.3`) へ通信する際、事前に仮想 IP (`136.1.17.100`) へ Telnet して Cut-Through Proxy 認証を行わせよ。認証・認可サーバーとして RADIUS (ISE: `172.16.3.100`) を使用せよ。

#### 要件
* 仮想 IP: `136.1.17.100`
* 認証成功後、RADIUS から付与される動的 ACL で通過を制御せよ。

#### 設定例

```bash
virtual telnet 136.1.17.100

aaa-server ISE-RADIUS protocol radius
aaa-server ISE-RADIUS (VLAN17) host 172.16.3.100
 key CISCO

access-list CUT_THROUGH_ACL extended permit tcp any host 136.1.17.100 eq 23
access-list CUT_THROUGH_ACL extended permit ip any host 136.1.37.3

aaa authentication match CUT_THROUGH_ACL VLAN17 ISE-RADIUS
```

#### 検証方法
* 端末から `telnet 136.1.17.100` を実行して認証を成功させる。
* ASA3 で `show uauth` を実行し、`user 'user3' at 136.1.17.9, authenticated` および動的ダウンロード ACL (`AAA-user-user3-xxx`) が適用されていることを確認。

---

### 例4: ASA: Cut-Through Proxy (HTTP 経由) と Web インターフェース認証ポータル設定

#### 問題
VLAN 17 からの全 HTTP 通信に対し、ASA3 のローカルデータベースを使用して WebAuth 認証を強制せよ。認証用の仮想 HTTP IP アドレスを `136.1.17.18` とせよ。

#### 要件
* ローカルユーザー `authproxy` / パスワード `authproxy` を使用すること。

#### 設定例

```bash
virtual http 136.1.17.18

username authproxy password authproxy

access-list WEBAUTH_ACL extended permit tcp any any eq 80

aaa authentication match WEBAUTH_ACL VLAN17 LOCAL
http server enable
```

#### 検証方法
* ブラウザから任意の Web サイトを開き、`[suspicious link removed]` へ自動リダイレクトされて認証プロンプトが表示されることを確認。
* 認証完了後、`show uauth` でアクティブユーザー数を確認。

---

### 例5: FTD (FMC): Active Directory Realm の構成と LDAP 疎通検証

#### 問題
FMC 上で Active Directory Realm `CORP_AD` を構成せよ。ドメイン名 `inelab.local`、DC IP `172.16.10.100` (Port 389) を指定し、ユーザーダウンロードを有効化せよ。

#### 要件
* FMC GUI の Realm 設定より実施すること。

#### 設定例

```text
1. FMC: System > Integration > Realms > Add Realm:
   - Name: CORP_AD
   - Type: Active Directory
   - Domain: inelab.local
   - Group Attribute: sAMAccountName
2. Directory AD Primary Host:
   - IP: 172.16.10.100, Port: 389
   - Username: CN=ldapuser,OU=Users,DC=inelab,DC=local
   - Password: 1qaz@WSX
   - Base DN: DC=inelab,DC=local
3. Test 実行後、OK をクリックして Save。
4. User Download タブ:
   - Download Users and Groups にチェックを入れ、Interval を 1 Hours に設定。
   - State を Enabled (トグルスイッチ ON) に変更。
```

#### 検証方法
* FMC の Realm 画面で、ダウンロードされたユーザー数およびグループ数が正常に更新されていることを確認。

---

### 例6: FTD (FMC): ISE pxGrid を接続した Passive Identity エンドツーエンド構成

#### 問題
FMC において、ISE (`ise.securitydemo.net`) と pxGrid 経由で接続し、ISE が保持する Passive Identity 情報を FMC へ自動パブリッシュさせよ。

#### 要件
* FMC GUI から Identity Services Engine 設定を行うこと。

#### 設定例

```text
1. FMC: System > Integration > Identity Sources > Identity Services Engine を選択。
2. 設定パラメータ入力:
   - Primary Host Name/IP: ise.securitydemo.net
   - pxGrid Server CA: ISE_Root_CA
   - MNT Server CA: ISE_Root_CA
   - FMC Server Certificate: FMC_pxGrid_Cert
3. Save 実行後、Test をクリック。
4. ISE GUI: Administration > pxGrid Services > Clients 画面を開き、FMC のクライアント要求を Approve する。
5. FMC に戻り、Test ステータスが Successful になることを確認。
```

#### 検証方法
* ISE の pxGrid Clients 画面で FMC のステータスが `Online` になっていることを確認。
* FMC の `Analysis > Users > Active Sessions` で ISE から同期された IP-to-User セッションが表示されることを確認。

---

### 例7: FTD (FMC): Identity Policy と Access Control Policy (User Condition) の紐付け

#### 問題
FTD1 において、Identity Policy `ID_POLICY_PASSIVE` を作成し、Realm `CORP_AD` のパッシブ認証ルールを定義せよ。ACP 内で `INELAB\CCIELAB` グループに属するユーザーのみ外部への Web アクセスを許可せよ。

#### 要件
* FMC GUI より設定すること。

#### 設定例

```text
1. Policies > Identity > Add Identity Policy:
   - Name: ID_POLICY_PASSIVE
   - Add Rule: Name: Rule1, Action: Passive Authentication, Realm: CORP_AD
2. Policies > Access Control > ACP 編集:
   - Identity Policy タブ: `ID_POLICY_PASSIVE` を選択。
3. ACP ルールの追加:
   - Rule Name: Allow_CCIELAB_Users, Action: Allow
   - Users タブ: Realm: CORP_AD ➔ Selected Users/Groups に `CCIELAB` グループを追加。
4. Save & Deploy 実行。
```

#### 検証方法
* ドメインユーザー `USER1` (CCIELAB グループ所属) でクライアント端末にログインして Web 閲覧を実行。
* FMC の `Analysis > Connection Events` で、該当通信の `User` カラムに `USER1` と表示され、ルールが許可マッチすることを確認。

---

### 例8: ASA: `user-identity monitor` を使用した未登録 AD グループの明示的ダウンロード

#### 問題
ASA2 において、Cloud Web Security (CWS) / ScanCenter ポリシーで使用するために、アクセスルールでまだ直接使用されていない AD グループ `INELAB\Engineering` のユーザー情報を AD Agent から強制ダウンロード・監視させよ。

#### 要件
* `user-identity monitor` コマンドを使用すること。

#### 設定例

```bash
user-identity monitor user-group INELAB\Engineering
```

#### 検証方法
* `show user-identity group` を実行し、`INELAB\Engineering` が Monitoring 一覧に含まれ、ユーザー数がカウントされていることを確認。

---

### 例9: FTD (FMC): Captive Portal (Active Authentication) によるフォールバック認証構成

#### 問題
FTD1 において、Passive Authentication で識別できなかった未認証ユーザーに対し、HTTP アクセス発生時に FMC の Captive Portal ポータル画面へリダイレクトして Active Authentication を実行させよ。

#### 要件
* Identity Policy 内で Active Authentication ルールを追加すること。

#### 設定例

```text
1. FMC: Objects > Object Management > PKI > Internal Certs に Captive Portal 用証明書を準備。
2. Devices > Device Management > FTD1 編集 > Device Settings > Captive Portal 設定:
   - Enable Captive Portal にチェック。
   - Port: 8843, Certificate: 準備した証明書を選択。
3. Policies > Identity Policy 編集:
   - Rule 1: Action: Passive Authentication, Realm: CORP_AD
   - Rule 2 (Catch-all): Action: Active Authentication, Realm: CORP_AD, Authentication Type: HTTP Basic / Response Page
4. Save & Deploy 実行。
```

#### 検証方法
* ドメイン外の端末から Web サイトを開き、FTD の Captive Portal 画面（Port 8843）へリダイレクトされてログインできることを検証。

---

### 例10: ASA: Active Directory / Agent 障害時の自動バイパス動作構成

#### 問題
ASA2 において、万が一 Active Directory サーバーまたは AD Agent との通信が途絶した場合でも、業務通信を全停止させないよう、Identity ルールの制限を解除してすべてのトラフィックを通過（Bypass）させる障害時動作を構成せよ。

#### 要件
* `remove-user-identity-rule` アクションを使用すること。

#### 設定例

```bash
user-identity action domain-controller-down INELAB remove-user-identity-rule
user-identity action ad-agent-down remove-user-identity-rule
```

#### 検証方法
* `show running-config user-identity` を実行し、障害時アクションが `remove-user-identity-rule` に変更されていることを確認。

---

### 例11: ASA: `show user-identity` コマンド群による IP-to-User キャッシュの手動検証とクリア

#### 問題
ASA2 上のアイデンティティマッピングキャッシュを手動で検証し、特定 IP (`10.1.1.50`) のマッピングキャッシュおよび全アイデンティティ統計情報をクリアせよ。

#### 要件
* ASA CLI コマンドを使用すること。

#### 設定例

```bash
! アイデンティティキャッシュの確認
show user-identity user ip 10.1.1.50
show user-identity memory

! 特定 IP または全アイデンティティキャッシュのクリア
clear user-identity user ip 10.1.1.50
clear uauth
```

#### 検証方法
* `show user-identity user` を実行し、`10.1.1.50` のエントリが消去されていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ASA2 上で以下の Identity Firewall コンフィグが適用されている。ドメインユーザー `INELAB\USER1`（`10.1.1.50`）が `10.2.2.100` 宛てに通信を発生させたが、パケットが拒否ドロップされた。原因として正しいものはどれか。

```bash
user-identity enable
user-identity default-domain inelab.local

object-group user OG_USERS
 user inelab.local\USER1

access-list INSIDE_ACL extended permit ip object-group-user OG_USERS any
access-group INSIDE_ACL in interface inside
```

* A) `user-identity enable` コマンドの文法が誤っている。
* B) `object-group user` 内で FQDN (`inelab.local`) が指定されており、大文字の NetBIOS ドメイン名 (`INELAB`) が使用されていないため。
* C) `access-group` で `global` キーワードが指定されていないため。
* D) AD Agent の IP アドレスが `127.0.0.1` に固定されているため。

**正解**: **B**
**解説**: Cisco ASA の Identity Firewall では、ユーザーやグループのドメイン名指定において FQDN (`inelab.local`) ではなく、必ず**大文字の NetBIOS ドメイン名 (`INELAB`)** を使用しなければなりません。FQDN で指定すると AD Agent や LDAP から受信したログオン情報とマッチせず、ルールが適用されません。

---

### 2. トラブルシュート問題

**問題**: 管理者が FTD1 と ISE 間の pxGrid 統合を構成し、FMC 上で Test ボタンをクリックしたが、接続ステータスが `Failed` になった。ISE の pxGrid Clients 画面を確認したところ、FMC のクライアント名 (`ia-fmc-xxx`) のステータスが `Pending` になっていた。このトラブルを解決するための正しい手順はどれか。

* A) FMC を再起動する。
* B) ISE GUI の **Administration > pxGrid Services > Clients** で対象 FMC クライアントを選択し **Approve** ボタンをクリックする。
* C) ISE の Monitoring サービスを停止する。
* D) FMC 上で LDAP サーバーのポートを 389 から 636 に変更する。

**正解**: **B**
**解説**: ISE で pxGrid クライアントの自動承認 (`Automatically approve new certificate-based accounts`) が有効化されていない場合、新規に接続を試みた FMC は `Pending` 状態（承認待ち）として保持されます。ISE GUI から対象クライアントを選択して `Approve` を実行することで接続が正常に確立されます。

---

### 3. Design 問題

**問題**: 企業ネットワークにおいて、数千台の非ドメイン参加端末（BYOD モバイル、Linux サーバ等）とドメイン参加 Windows PC が混在している。ネットワークアクセスポリシーにおいて、ドメイン端末には透過的な Single Sign-On (SSO) を提供し、非ドメイン端末に対してはブラウザ経由で手動ログインさせてアクセス権を与える設計として最も適切な構成はどれか。

* A) 全トラフィックに対して Cut-Through Proxy のみを適用する。
* B) FTD 上で Identity Policy を構成し、第 1 ルールに `Passive Authentication` (Realm 連携)、第 2 ルール（Fallback）に `Active Authentication` (Captive Portal) を定義する。
* C) ASA 上で `user-identity action domain-controller-down remove-user-identity-rule` のみを設定する。
* D) 全端末に AnyConnect Nam モジュールを強制インストールする。

**正解**: **B**
**解説**: ドメイン端末には AD ログオンをトリガーとする Passive Authentication で透過 SSO を提供し、マッピングが存在しない非ドメイン端末に対しては Active Authentication (Captive Portal) へフォールバックさせて手動認証を要求する構成が、ユーザー利便性とセキュリティを両立させるベストプラクティス設計です。

---

### 4. 実装問題

**問題**: ASA2 において、新しい AD グループ `INELAB\Accounting` を追加したが、アクセスルールにまだ記述していない。このグループのユーザーマッピング情報を AD Agent から事前に取得し、メモリ上で監視対象とするために必要な ASA CLI コマンドはどれか。

* A) `user-identity monitor user-group INELAB\Accounting`
* B) `aaa-server AD-AGENT host 172.16.10.100 poll`
* C) `ldap-base-dn OU=Accounting,DC=inelab,DC=local`
* D) `object-group user Accounting`

**正解**: **A**
**解説**: ASA はデフォルトでアクティブな ACL に含まれるグループの情報のみを AD Agent から取得します。ACL にまだ含まれていないグループの情報を事前にダウンロード・監視するには `user-identity monitor user-group <DOMAIN\group>` コマンドを使用します。

---

### 5. コンフィグ読解問題

**問題**: ASA3 において Cut-Through Proxy を設定し、以下のコマンドを投入した。ユーザーが最初に Web サーバー (`10.2.0.10`) へアクセスして認証が成功した後、2 回目のアクセス時の動作として正しいものはどれか。

```bash
virtual http 192.168.100.1
aaa authentication match AUTH_ACL inside ISE-RADIUS
timeout uauth 01:00:00 absolute
```

* A) 毎回 Web 認証画面が表示され、パスワード再入力が必要になる。
* B) 最初の認証成功後、ASA の `uauth` キャッシュにセッションが保持され、タイマー（1時間）以内であれば再認証なしで通信がカットスルー（直通）通過する。
* C) パケットは ISE に毎回 RADIUS リダイレクトされる。
* D) 通信は即座にドロップされる。

**正解**: **B**
**解説**: Cut-Through Proxy は、初回アクセス時にのみ認証プロキシとして動作してユーザーを認証し、認証完了後は送信元 IP アドレスとユーザーのバインディングを `uauth` キャッシュに記録します。キャッシュが有効な間（`timeout uauth` 到達前）は、以降の通信が認証なしで高速にカットスルー処理されます。

---

## 🔗 参考リソース

* [Cisco ASA Series Firewall Configuration Guide - Identity Firewall]([suspicious link removed])
* [Cisco Secure Firewall Threat Defense Configuration Guide (FMC) - User Identity Policies]([suspicious link removed])
* [Cisco Identity Services Engine (ISE) PassiveID Work Center User Guide]([suspicious link removed])
* [Cisco Technical Notes: ASA Identity Firewall Configuration Example with AD Agent and LDAP]([suspicious link removed])
* [Cisco Live: Deep Dive into Firepower & ISE Identity Integrations (BRKSEC-3022)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * ASA IDFW の試験対策ポイントは **「NetBIOS 名の大文字指定」「`user-identity monitor`」「AD Agent と LDAP の二重設定」** です。
  * FTD Identity Policy では **「Realm (LDAP) ➔ pxGrid (ISE) ➔ Identity Policy ➔ Access Control Policy Users Condition」** というデータの流れを確実に理解して設定できるようにしてください。
* **図解（ASA Cut-Through Proxy uauth キャッシュの仕組み）**:
  ```text
  [ Client 10.1.1.50 ] ─── 1. HTTP Req ───► [ ASA Firewall ] ─── 2. Intercept & Redirect ───► [ WebAuth Portal ]
                                                 │                                                │
                                                 │◄── 3. Authenticated (User: Alice) ─────────────┘
                                                 │
                                     [ Register to uauth Cache ]
                                     (10.1.1.50 <-> User: Alice)
                                                 │
  [ Client 10.1.1.50 ] ─── 4. Next Req ───► [ ASA Firewall ] ─── 5. Matched uauth Cache ──► [ Direct Cut-Through ]
  ```
* **注意点**:
  * トラブルシューティング時、マッピングキャッシュが更新されない場合は、まず `show user-identity ad-agent` で RADIUS セッション状態を確認し、次に `show user-identity user` で特定の IP アドレスが ASA 上に学習されているかを段階的に追跡してください。

