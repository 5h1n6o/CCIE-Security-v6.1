---
layout: default
title: 4.13-Auth-methods
nav_order: 13
parent: 4.0-Identity-Management
---

# 4.13 Authentication methods

認証（Authentication）は、ネットワークリソースにアクセスしようとするエンティティ（ユーザーまたはデバイス）の「身元」を検証するプロセスです。CCIE Security v6.1 においては、有線/無線 LAN での 802.1X/MAB、デバイス管理用の TACACS+/RADIUS、VPN 接続、さらには API を介したシステム間連携まで、多岐にわたる認証方式の深い理解と実装能力が求められます。

---

## 📘 概要

*   **機能概要**: ID/パスワード、デジタル証明書、トークン、またはデバイスの物理アドレス（MAC）を使用して、アクセスの正当性を確認する機能です。
*   **利用目的**: 不正アクセスの防止、監査ログ（誰がいつ入ったか）の記録、および認証結果に基づく動的な権限割当の実現。
*   **どのような場面で利用するか**:
    *   **キャンパスアクセス**: 社員 PC の有線 802.1X 認証やプリンタの MAB 認証。
    *   **リモートアクセス**: AnyConnect を使用した証明書＋ID/パスワードによる二要素認証。
    *   **デバイス管理**: 管理者が Cisco 機器の CLI にログインする際の TACACS+ 認証。
    *   **自動化/API**: スクリプトから DNA Center や FMC を操作するためのトークンベース認証。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要方式** | 802.1X (EAP-TLS, PEAP, FAST), MAB, Web Auth, SAML, API Auth。 |
| **認証プロトコル** | RADIUS (UDP 1812), TACACS+ (TCP 49), HTTPS (API)。 |
| **認証要素** | 知識情報 (Password)、所有情報 (Certificate/Token)、固有情報 (Biometrics)。 |
| **Identity Store** | Cisco ISE 内部 DB, Active Directory, LDAP, SAML IdP。 |
| **Identity Source Sequence** | 複数のアイデンティティソースを優先順位に従って検索する論理。 |
| **セッション管理** | 認証後のセッション維持、再認証タイマー、CoA による動的更新。 |

---

## 🏗 動作原理

ネットワークアクセスにおける一般的な RADIUS/EAP 認証フロー（802.1X）を以下に示します。

```text
Supplicant (PC)          Authenticator (Switch/WLC)       Authentication Server (ISE)
      |                          |                                 |
      |--- (1) EAPOL-Start ----->|                                 |
      |                          |--- (2) RADIUS Access-Request -->|
      |                          |        (EAP-Message)            |
      |                          |<-- (3) RADIUS Access-Challenge -|
      |<-- (4) EAP-Request/ID ---|                                 |
      |--- (5) EAP-Response/ID ->|                                 |
      |                          |--- (6) RADIUS Access-Request -->|
      |                          |                                 |
      |   [ (7) EAP Method Negotiation & Identity Verification ]   |
      |                          |                                 |
      |                          |<-- (8) RADIUS Access-Accept ----|
      |<-- (9) EAP-Success ------|                                 |
      |                          |                                 |
      |--- (10) Data Traffic --->|                                 |
```

---

## ⚙ 動作シーケンス

1.  **初期コンタクト**: サプリカントが L2 で接続を検知。認証が必要なポートの場合、スイッチが EAP-Request/Identity を送付。
2.  **リクエスト転送**: スイッチ（認証者）は受信した EAP パケットを RADIUS パケットにカプセル化し、ISE（認証サーバ）へ転送。
3.  **プロトコル交渉**: ISE とサプリカント間で、使用する EAP メソッド（TLS, PEAP 等）を決定。
4.  **検証**: 
    *   **EAP-TLS**: デジタル証明書の相互検証（PKI 基盤が必要）。
    *   **PEAP (MS-CHAPv2)**: サーバー証明書の検証後、暗号化トンネル内で ID/パスワードを検証。
5.  **認可の紐付け**: 認証成功後、ISE は認証された ID に基づき、VLAN や SGT などの認可プロファイルを Access-Accept に含めて返信。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **EAP メソッドの選択**: 要件が「証明書のみ」なら EAP-TLS、「パスワード＋AD」なら PEAP、「両方の組み合わせ」なら TEAP を選択する能力。
*   **MAB (MAC Authentication Bypass)**: サプリカントを持たないデバイスに対し、`mab` コマンドで MAC アドレスをユーザー名として認証させる設定。
*   **Identity Source Sequence (ISS)**: 複数の外部 DB（AD と LDAP 等）がある場合に、どの順番で検索するかを ISS で構成する問題は頻出。
*   **AnyConnect 二要素認証**: ユーザー証明書を必須としつつ、AD パスワードも要求する AnyConnect プロファイルと ISE ポリシーの構成。
*   **API Authentication**: FMC や DNA Center への REST API 通信において、Basic Auth だけでなくトークン（`X-Auth-Token`）を取得して使用するフロー。
*   **TACACS+ vs RADIUS**: 管理用アクセスには TACACS+（コマンド承認が可能）、ネットワークアクセスには RADIUS という使い分けの厳守。

---

## 🛠 設定方法

### 1. IOS スイッチでの 802.1X/MAB 基本設定 (CLI)
```bash
! RADIUSサーバの定義
aaa new-model
radius server ISE_NODE
 address ipv4 10.1.1.100 auth-port 1812 acct-port 1813
 key cisco123

! 認証・認可リストの作成
aaa authentication dot1x default group radius
aaa authorization network default group radius
aaa accounting dot1x default start-stop group radius

! インターフェイスへの適用
interface GigabitEthernet1/0/1
 switchport mode access
 authentication port-control auto
 dot1x pae supplicant
 mab
```

### 2. Cisco ISE での認証ポリシー設定 (GUI)
1.  **Policy > Policy Sets** で、特定のネットワーク条件（Wired/Wireless）に合致するセットを選択。
2.  **Authentication Policy** セクションを開く。
3.  `Rule 1`: 条件を `Wired_802.1X` とし、`Use` に `Active Directory` を指定。
4.  `Options`: `If user not found` を `Reject` ではなく `Continue` にすることで、次の ID ソース（MAB 用の内部 DB 等）へ進めることが可能。

---

## 🔍 検証コマンド

| 目的 | コマンド / 手法 |
| :--- | :--- |
| **認証セッションの確認** | <code>show access-session interface [int] details</code> |
| **RADIUS 統計の確認** | <code>show radius statistics</code> |
| **ISE 認証ライブログ** | **Operations > RADIUS > Live Logs** |
| **802.1X 状態のデバッグ** | <code>debug dot1x all</code> / <code>debug radius</code> |
| **API 認証テスト** | <code>curl -u "user:pass" -X POST https://[IP]/api/v1/ticket</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 認証がタイムアウトする | 共有キーの不一致、UDP 1812 遮断 | <code>show aaa servers</code> で応答があるか確認。 |
| 証明書認証 (EAP-TLS) 失敗 | 時刻同期ズレ、信頼チェーン欠如 | <code>show ntp status</code> を確認。ISE に Root CA を登録。 |
| MAB で拒否される | MAC アドレスが ISE 内部 DB に未登録 | ISE の **Endpoints** リストに対象 MAC があるか確認。 |
| AD ユーザーがログイン不可 | AD 連携（Join）の切断 | <code>show application status ise</code> で AD Connector 状態を確認。 |

---

## ⚠ 制限事項

*   **MAB のセキュリティ**: MAC アドレスは偽装が容易なため、プロファイリング（Device Sensor）と組み合わせてデバイスの種類を検証することが推奨される。
*   **EAP-TLS フラグメンテーション**: 証明書サイズが大きい場合、RADIUS パケットが分割され、古いネットワーク機器で破棄されることがある。MTU 調整が必要。
*   **SAML の RADIUS 非対応**: SAML は Web ベースの認証（ポータル）には使えるが、通常の 802.1X サプリカント認証には直接使用できない。

---

## 🔄 他技術との関連

*   **2.6 Microsegmentation (TrustSec)**: 認証に成功したエンティティに対し、SGT（Security Group Tag）を動的に割り当てる。
*   **4.6 BYOD Flow**: 初回はパスワード認証、プロビジョニング後に証明書認証（EAP-TLS）へ遷移するワークフロー。
*   **4.11 MDM Integration**: 認証プロセス中に外部 MDM サーバへ問い合わせ、デバイスのコンプライアンス状態を確認する。

---

## 🧩 比較表

### EAP-TLS vs PEAP vs EAP-FAST

| 特徴 | EAP-TLS | PEAP (MS-CHAPv2) | EAP-FAST |
| :--- | :--- | :--- | :--- |
| **認証要素** | デジタル証明書 | パスワード (AD 連携) | PAC (Protected Access Credential) |
| **セキュリティ** | 最高（PKI 必須） | 高（TLS トンネル内） | 中〜高（Cisco 独自拡張） |
| **クライアント要件** | 証明書のインストールが必要 | 設定のみで容易 | PAC の配布が必要 |
| **推奨用途** | 企業管理デバイス | 一般ユーザー、BYOD | PKI 導入が困難な環境 |

---

## 💡 ベストプラクティス

1.  **段階的な導入 (Monitor Mode)**: 認証を強制する前に、`authentication open` コマンドで全通信を許可し、ログのみを ISE で収集して影響範囲を調査する。
2.  **マルチ認証の構成**: 同一ポートで `dot1x` と `mab` の両方を有効にし、サプリカントの有無にかかわらず適切な認証が行われるようにする。
3.  **フェイルオープン設計**: RADIUS サーバが全ダウンした場合に備え、`authentication event server dead` 時の VLAN 指定（Critical VLAN）を構成する。
4.  **Certificate Chain の最適化**: ISE のポータル証明書や認証証明書には、中間 CA を含めたフルチェーンをバインドし、クライアント側の検証エラーを防止する。

---

## 📝 ラボ学習・設定サンプル例

### 1. EAP-TLS 認証の構成
*   **要件**: 証明書を持つ Windows マシンのみを許可せよ。
*   **設定**: ISE で `Certificate Authentication Profile` を作成し、ID を `Subject - Common Name` から抽出するよう設定。

### 2. 802.1X + MAB の順序制御
*   **要件**: 最初に 802.1X を試行し、失敗（またはタイムアウト）したら MAB を実行せよ。
*   **設定**: `authentication order dot1x mab` / `authentication priority dot1x mab`。

### 3. TACACS+ による CLI 認証
*   **要件**: スイッチへのログインを ISE (TACACS+) で管理し、`enable` 権限を制御せよ。
*   **設定**: `aaa authentication login default group tacacs+ local`。

### 4. 認証失敗時の Guest VLAN 遷移
*   **要件**: 認証に失敗したユーザーを VLAN 999 (Guest) に隔離せよ。
*   **設定**: `authentication event fail action authorize vlan 999`。

### 5. Identity Source Sequence の複合検索
*   **要件**: `Internal Users` にいなければ `Active Directory` を検索する ISS を作成せよ。

### 6. TEAP (Chained Authentication)
*   **要件**: マシン証明書とユーザー証明書の両方が有効な場合のみ「準拠」とみなせ。

### 7. AnyConnect 証明書認証
*   **要件**: VPN 接続時に、PC にインストールされた特定 CA 発行の証明書を自動提示させよ。

### 8. DNA Center Northbound API Auth
*   **操作**: Python `requests` ライブラリを使用して `/dna/system/api/v1/auth/token` からトークンを取得し、以降の GET リクエストのヘッダーに含めよ。

### 9. Web Authentication (CWA)
*   **要件**: 認証機能のないゲスト端末が HTTP アクセスした際、ISE ポータルへリダイレクトせよ。

### 10. 証明書失効チェック (OCSP)
*   **要件**: 認証プロセスの一環として、外部 OCSP レスポンダへの問い合わせを構成せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: スイッチに `authentication host-mode multi-auth` が設定されている。この挙動は？
    *   **回答**: 各 MAC アドレスが個別に認証される。音声デバイス（IP Phone）と PC がぶら下がっている環境で、それぞれ別の権限を与えるのに適している。
2.  **トラブルシュート**: ISE ライブログで `24408 User not found` と出るが、ユーザーは AD に存在する。ISS のどこを確認すべきか？
    *   **回答**: Identity Source Sequence のリストに、対象の AD ドメイン（Join Point）が含まれているか、または検索の順序が正しいかを確認。
3.  **Design**: パスワードベースの認証において、辞書攻撃や総当たり攻撃に対する耐性を高めるための ISE 側の機能は？
    *   **回答**: **Lockout Policy**（連続失敗時のアカウントロック）または **Multi-Factor Authentication (MFA)** との連携。
4.  **実装**: API スクリプトで `X-Auth-Token` が期限切れ（401 Unauthorized）になった際のフローを設計せよ。
    *   **回答**: `try-except` 構文でエラーをキャッチし、再度ログイン API を叩いて新しいトークンを取得・キャッシュする処理を入れる。
5.  **Design**: 拠点間の WAN 遅延が非常に大きい環境で、802.1X 認証の安定性を高めるための構成は？
    *   **回答**: 拠点に **PSN (Policy Service Node)** を分散配置するか、またはスイッチ側で `radius-server timeout` 値を調整する。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [認証ポリシーの設定](https://www.cisco.com/c/ja_jp/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31.html)。
*   **CVD**: [Campus Wired LAN 802.1X Design Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/ise-deployment-guide.html)。
*   **Cisco Live (BRKSEC-2022)**: [Demystifying 802.1X and MAB](https://www.ciscolive.com/)。
*   **API Reference**: [DNA Center Authentication API](https://developer.cisco.com/docs/dna-center/)。

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「認証（誰か？）」と「認可（何ができるか？）」の境界を常に意識してください。本項目は「誰か？」を確定させるための多様なゲートウェイ設定を扱います。
*   **図解**: 
    - **PEAP**: 外部は TLS トンネル、内部はパスワード（MS-CHAPv2）。
    - **EAP-TLS**: 外部・内部ともに証明書ベース。
*   **注意点**: ラボ試験では、**RADIUS 共有キーのスペース（余計な空白）**や、**VLAN 名のタイポ**だけで認証フロー全体が失敗するため、投入後の `show` コマンドによる確認が不可欠です。
