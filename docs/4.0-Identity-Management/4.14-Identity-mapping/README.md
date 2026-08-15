---
layout: default
title: 4.14-Identity-mapping
nav_order: 14
parent: 4.0-Identity-Management
---

# 4.0 Identity Management, Information Exchange, and Access Control
# 4.14 Identity mapping on Cisco ASA, Cisco ISE, Cisco WSA, and Cisco FTD

**Identity Mapping（アイデンティティ・マッピング）**とは、ネットワーク上の「IPアドレス」を特定の「ユーザー名」や「所属グループ」、「セキュリティグループタグ (SGT)」などの識別子に紐付けるプロセスです,。これにより、IPベースの静的なルールではなく、ユーザーの属性に基づいた動的で文脈に応じた（Context-Aware）アクセス制御が可能になります。Ciscoのセキュリティエコシステムでは、**Cisco ISE**がアイデンティティ情報のハブとなり、pxGridやSXPを通じてASA、FTD、WSAへ情報を配信します,。

---

## 📘 概要

*   **機能概要**: ユーザーのログインイベント（ADログ、802.1X認証、ポータル認証など）を監視し、そのユーザーがどのIPアドレスを使用しているかを特定・共有する機能です,。
*   **利用目的**: ユーザー移動やDHCPによるIP変更に左右されない一貫したセキュリティポリシーの適用、および監査ログにおける「誰が」の可視化。
*   **どのような場面で利用するか**:
    *   **ASA/FTD**: ファイアウォールのアクセスリスト（ACL）で「営業グループのみ許可」といった記述を行う場合。
    *   **WSA**: Webアクセスプロキシにおいて、ユーザーごとに異なるURLフィルタリングカテゴリを適用する場合。
    *   **ISE**: 全デバイスの認証状態を管理し、他のセキュリティコンポーネントにその情報を伝搬させる役割。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要コンポーネント** | Cisco ISE (情報の提供元), FMC (FTD管理), ASA, WSA,。 |
| **情報の伝搬プロトコル** | **Cisco pxGrid** (推奨), SXP (TrustSec用), RADIUS CoA,。 |
| **Mappingの種類** | **Passive ID** (ADログ監視等) と **Active ID** (Captive Portal等)。 |
| **ASAの役割** | Identity Firewall (IDFW) として、ユーザー/グループベースのACLを実行。 |
| **FTDの役割** | FMCで定義したIdentity Policyに基づき、User-based Access Controlを処理。 |
| **WSAの役割** | Identification Profileを使用して、ユーザー名に基づくWebポリシーを適用。 |
| **ISEの役割** | アイデンティティの「信頼できる唯一の情報源（Source of Truth）」。 |

---

## 🏗 動作原理

Ciscoのアイデンティティ環境は、ISEを中心にpxGridネットワークを形成します。

```text
 [ Active Directory ] <--- (Security Log / WMI) --- [ Cisco ISE (PassiveID) ]
                                                            |
                                                            | (Identity Sharing via pxGrid)
          __________________________________________________|_________________________________
         ↓                                                  ↓                                 ↓
 [ Cisco FTD / FMC ]                                [ Cisco ASA ]                      [ Cisco WSA ]
         ↓                                                  ↓                                 ↓
  Identity Policy                                    Identity Firewall                  Identification Profiles
 (Assign IP to User)                                (Assign IP to User)                (Assign IP to User)
         ↓                                                  ↓                                 ↓
 Access Control Policy                              Network Object Group                Web Access Policy
 (Permit User Group A)                              (user AD_Domain\GroupA)             (Allow Social-Media)
```

---

## ⚙ 動作シーケンス

1.  **アイデンティティの学習 (ISE)**: ユーザーがWindowsにログイン。ISEはADからWMI経由でログを取得、または802.1X認証により「User A = 10.1.1.5」であることを学習します。
2.  **コンテキストの配信 (pxGrid)**: ISEはpxGridを通じて、購読者（ASA, FTD, WSA）にアイデンティティ情報の更新をリアルタイムで通知します。
3.  **マッピングテーブルの更新**: 各デバイス（ASA/FTD/WSA）は自身のローカルメモリ上に「IP to User Mapping」テーブルを保持・更新します。
4.  **ポリシー照合**:
    *   **FTD**: パケットが到着すると、送信元IPをマッピングテーブルで引き、合致するユーザー/グループのAccess Control Ruleを適用します。
    *   **ASA**: ACL内で指定された`user`キーワードに基づき、ユーザー属性を確認してトラフィックを処理。
    *   **WSA**: HTTPリクエストの送信元IPからユーザーを特定し、 Identification Profileに合致させます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **pxGridの構成**: ISEとFMC（またはASA/WSA）の間で証明書による信頼関係（Trusted Certificates）を構築し、pxGridノードとして承認（Approve）する手順は必須です,。
*   **PassiveIDの設定**: ISE側でADドメインコントローラを登録し、AgentなしのWMI方式、またはISEエージェント方式でログを取得する設定が問われます。
*   **Identity Policy (FTD)**: FMCにおいて、どのインターフェイスからのトラフィックに対してIdentity Mappingを適用するかを決める「Identity Policy」を正しく作成・適用する必要があります,。
*   **SXP (Scalable Group Tag Exchange Protocol)**: SGT情報をASAなどに伝える際、ピアの設定やパスワードの整合性に注意してください,。
*   **トラブルシュート（マッピング確認）**: 
    *   `show user-identity` (ASA)
    *   `show identity-user` (FTD CLI)
    *   FMCの `Analysis > Users > Sessions` でマッピングが反映されているか確認。

---

## 🛠 設定方法

### 1. Cisco ISE：pxGrid の有効化
1.  **Administration > System > Deployment**: pxGrid サービスを有効化。
2.  **Administration > pxGrid Services**: クライアント（FMC等）からの接続要求を `Approve`。

### 2. Cisco FTD (FMC管理)：Identity統合
1.  **Devices > Certificates**: ISEのルートCAをインポート。
2.  **Integration > Other Integrations > Identity Sources**: ISEをpxGridサーバとして追加。
3.  **Policies > Identity**: 新規ポリシー作成。`Passive ID` をルールとして追加。
4.  **Policies > Access Control**: `Identity Policy` を紐付け、ルール内の `Users` タブでADグループを選択。

### 3. Cisco WSA：ISE-pxGrid 連携
1.  **Network > ISE pxGrid**: ISEのホスト名、証明書をアップロードして接続。
2.  **Web Security Manager > Identification Profiles**: `Transparently Identify Users with ISE` を選択。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド / 手法 |
| :--- | :--- | :--- |
| **pxGrid接続状態の確認** | ISE | **Administration > pxGrid Services > Clients** |
| **学習したユーザー情報の確認** | FMC | **Analysis > Users > Active Sessions** |
| **マッピングテーブルの確認** | FTD CLI | <code>show identity-user all</code> |
| **Identity Firewall状態確認** | ASA | <code>show user-identity ip 10.1.1.5</code> |
| **認証プロファイルの確認** | WSA | <code>testauthconfig -アカウント名-</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| マッピングが表示されない | ISE/AD間の連携失敗 | ISEの **PassiveID Dashboard** でADログの取得数を確認。 |
| pxGridが "Disconnected" | 証明書の不整合 | 各デバイスの時刻(NTP)と、ISE CA証明書の有効性を確認。 |
| ユーザーACLが効かない | Identity Policyの未適用 | Access Control Policyの `Advanced` タブで Identity Policy が選択されているか確認。 |
| 特定ユーザーのみIP不明 | ログインイベントの欠落 | ユーザーがPCにログインした際のADログがISEに届いているかパケットキャプチャで確認。 |

---

## ⚠ 制限事項

*   **同時ログイン**: 1つのIPアドレスに複数のユーザーがログインしている場合（VDIやターミナルサーバ）、ISEの特定のバージョンや設定（TS-Agent）がないと正確に識別できません。
*   **プロトコルのオーバーヘッド**: 非常に大規模な環境（数万ユーザー）では、pxGridの更新トラフィックが管理プレーンに負荷をかける場合があります。
*   **ASAの制限**: ASAのIdentity Firewallは、FMC/FTDに比べると属性ベースの柔軟性が限定的です。

---

## 🔄 他技術との関連

*   **2.6 Microsegmentation**: Identity Mappingで得た「ユーザー」を「SGT」にマッピングし、TrustSecポリシーを実行します,。
*   **4.7 external identity sources**: ADやLDAPがマッピングの元データとなります。
*   **3.6 Monitoring protocols**: NetFlow (NSEL) ログにユーザー名が含まれるようになり、可視性が向上します。

---

## 🧩 比較表

### Passive ID vs Active ID

| 特徴 | Passive ID (受動的) | Active ID (能動的) |
| :--- | :--- | :--- |
| **ユーザー体験** | **透過的**（再ログイン不要） | ポータル画面でのログインが必要 |
| **仕組み** | ADログ監視、ISE pxGrid | Captive Portal (Web Auth) |
| **確実性** | 中（ログが流れるまで不明） | **高**（認証しないと通さない） |
| **主な用途** | 企業管理のWindows端末 | ゲスト、非ドメイン端末 |

---

## 💡 ベストプラクティス

1.  **NTP同期の徹底**: 証明書ベースのpxGrid通信において、ISEと各セキュリティデバイス（ASA/WSA/FTD）の時刻同期は必須です。
2.  **複数のADログソース**: ドメインコントローラが複数ある場合、ISEですべてのDCを監視対象とし、マッピングの欠落を防ぎます。
3.  **証明書の SAN フィールド**: pxGrid通信用の証明書を作成する際、Subject Alternative Name (SAN) にFQDNとIPアドレスの両方を含めることで、解決エラーを回避します。
4.  **フォールバックルールの設計**: アイデンティティが不明（Unknown User）なトラフィックに対して、デフォルトで拒否するのか、最小限のアクセスを許可するのかを事前に定義します。

---

## 📝 ラボ学習・設定サンプル例

### 1. ISE PassiveID の基本構成
*   **要件**: AD 10.1.1.10 のログを取得し、WMI経由でユーザー情報を学習せよ。

### 2. FMC と ISE の pxGrid 接続
*   **要件**: FMC を ISE の pxGrid 購読者として登録し、Active Sessions が見えるようにせよ。

### 3. FTD でのユーザーベースACR
*   **要件**: 「Domain Users」グループに属するユーザーのみ、DMZサーバーへのアクセスを許可せよ。
*   **設定**: ACR の `Users` タブで該当グループを `Add`。

### 4. WSA Identification Profile の作成
*   **要件**: ISE pxGrid から取得した情報を使い、認証なしでユーザーを特定せよ。

### 5. ASA ユーザーオブジェクトグループの設定
```bash
! ASA CLI での定義例
user-identity ad-domain LOCAL_DOMAIN
 aaa-server ISE_GROUP protocol radius
!
object-group user SALES_USERS
 user-group LOCAL_DOMAIN\\Sales
!
access-list INSIDE_IN extended permit ip object-group user SALES_USERS any
```

### 6. SXP 経由での SGT 伝搬
*   **要件**: ISE を SXP Speaker、FTD を SXP Listener として構成せよ。

### 7. FTD Captive Portal の構成
*   **要件**: PassiveID で特定できないユーザーに対し、HTTPリダイレクトによるActive認証を実行せよ。

### 8. ISE 証明書認証プロファイル (CAP) の高度なマッピング
*   **要件**: クライアント証明書の `Subject-OU` 属性をアイデンティティとしてマッピングせよ。

### 9. ターミナルサーバ (TS) エージェントの利用
*   **要件**: 同一IPの複数ユーザーを識別するため、ISE TS-Agent を構成せよ。

### 10. FMC でのユーザーセッション統計の確認
*   **操作**: `Analysis > Users > Sessions` を開き、特定のユーザーが現在どのIPでマッピングされているか確認せよ。

---

## ❓ 想定試験問題

1.  **Design**: FTDでユーザーベースのポリシーを適用したいが、ネットワーク上にサプリカントがないレガシー端末が多い。最適なマッピング手法は？
    *   **回答**: **Passive ID**。ISEがドメインコントローラのログを監視することで、サプリカントなしで透過的にマッピングを行う。
2.  **トラブルシュート**: FMCでIdentity Sourceが "Connected" になっているが、Live Logsにユーザー名が表示されない。何を確認すべきか？
    *   **回答**: ISE側で **pxGridノードの承認 (Approve)** が行われているか、および対象インターフェイスに **Identity Policy** が適用されているか。
3.  **コンフィグ読解**: ASAで `user-identity default-domain` が設定されている場合の効果は？
    *   **回答**: ユーザー名にドメイン指定がない場合、自動的にそのデフォルトドメインを補完して検索を行う。
4.  **Design**: WSAで複数のADフォレストがある環境において、ISE pxGridを使用するメリットは？
    *   **回答**: WSAが各ADと個別に通信する必要がなく、**ISEがすべてのアイデンティティを集約・抽象化**してWSAに提供するため、構成がシンプルになる。
5.  **実装**: pxGrid 連携において証明書エラー (Handshake failure) が発生している。最初に見るべきCLIコマンドは？
    *   **回答**: <code>show ntp status</code> (時刻同期) および <code>show crypto ca certificates</code> (信頼チェーンの確認),。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [Identity Managementの設定](https://www.cisco.com/c/ja_jp/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31.html)
*   **FMC 7.1 構成ガイド**: [Identity Policies の作成](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/identity_policies.html)
*   **Cisco Live (BRKSEC-2041)**: [Context Sharing with Cisco pxGrid](https://www.ciscolive.com/)
*   **Design Guide**: [Identity-Based Firewall Deployment Guide](https://www.cisco.com/c/en/us/td/docs/security/firepower/firewalls/idfw/identity-firewall-using-ise-and-fmc.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: Identity Mapping は「セキュリティにおける翻訳機」です。L3/L4 の言語（IP）を、人間が理解できる言語（ユーザー名）に翻訳してデバイスに伝えます。
*   **図解**: `ISE --(pxGrid)--> FMC --(Deployment)--> FTD` という情報の流れを意識してください。
*   **注意点**: ラボ試験では、**ISE の FQDN 解決**ができないために FMC との pxGrid 連携が失敗するケースがあります。DNS サーバーの設定または各デバイスの `/etc/hosts` (CLI) 的な設定を確認してください。
