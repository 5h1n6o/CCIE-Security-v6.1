---
layout: default
title: 4.15-pxGrid
nav_order: 15
parent: 4.0-Identity-Management
---

# 4.15 pxGrid integration between security devices Cisco WSA, Cisco ISE, and Cisco FMC

**Cisco pxGrid (Platform Exchange Grid)** は、Cisco ISE をハブとして、異なるセキュリティ製品間でコンテキスト情報（ユーザー ID、デバイス タイプ、SGT、ポスチャ状態など）をリアルタイムに共有するためのオープンな統合フレームワークです。これにより、FMC や WSA は自身で認証を行わなくても、ISE から得た情報を基に詳細なアクセスポリシーを適用できるようになります。

---

## 📘 概要

*   **機能概要**: 多対多の通信プロトコル（主に XMPP ベース）を使用して、セキュリティ製品同士がベンダーを問わず情報を交換する仕組みです。
*   **利用目的**: ネットワークの可視化を向上させ、アイデンティティ（誰が）とコンテキスト（どのデバイスで、どのような状態か）に基づいた一貫性のある防御を実現します。
*   **どのような場面で利用するか**:
    *   **ISE + FMC**: ISE が学習したユーザー ID と IP アドレスのマッピングを FMC に共有し、Firepower 上でユーザー名ベースのルールを適用する。
    *   **ISE + WSA**: ISE から SGT (Security Group Tag) 情報を取得し、Web プロキシ側でタグに基づいた URL フィルタリングを行う。
    *   **Rapid Threat Containment (RTC)**: FMC や他デバイスが脅威を検知した際、pxGrid 経由で ISE に通知し、ISE が即座に対象ポートを隔離（ANC 等）する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要コンポーネント** | **ISE** (コントローラ/ブローカー), **pxGrid クライアント** (FMC, WSA, 等)。 |
| **通信プロトコル** | **XMPP** (Extensible Messaging and Presence Protocol) を基盤とする。 |
| **信頼モデル** | **証明書ベースの認証**。ISE と各クライアント間での相互信頼が必須。 |
| **役割 (Roles)** | Publisher（情報提供者）、Subscriber（購読者）、Controller（仲介者）。 |
| **共有される情報** | Session Directory, TrustSec Metadata (SGT), Adaptive Network Control (ANC)。 |
| **ライセンス** | ISE **Advantage** 以上のライセンスで pxGrid サービスが利用可能。 |

---

## 🏗 動作原理

pxGrid は「パブリッシュ/サブスクライブ」モデルを採用しており、ISE が情報のハブ（ブローカー）として機能します。

```text
[ Identity Source ] (AD/802.1X)
        ↓
    [ Cisco ISE ] <------- (Broker / Controller)
    (Publisher)
        ↓ 
        ↓ (pxGrid / XMPP)
        ↓ 
    __________________________
    ↓                        ↓
[ Cisco FMC ]            [ Cisco WSA ]
(Subscriber)             (Subscriber)
```

1.  **ISE** が 802.1X 認証などでエンドポイント情報を学習し、Session Directory に書き込む。
2.  **FMC/WSA** は ISE に対して特定の情報を「購読（Subscribe）」することを登録する。
3.  ISE 上で情報が更新されると、即座に **pxGrid** を通じて購読しているクライアントへ通知される。

---

## ⚙ 動作シーケンス

1.  **サービスの有効化**: ISE の Administration ノードで pxGrid サービスをオンにする。
2.  **証明書の構築**: 全デバイスが ISE pxGrid サーバー証明書を信頼し、自身の証明書を ISE に提示して承認を受ける。
3.  **クライアント登録**: FMC/WSA が ISE の pxGrid サービスに接続。ISE 管理者が GUI で接続を「承認 (Approve)」する。
4.  **情報の購読**: FMC は `Identity Store` として ISE を設定し、セッション情報の同期を開始する。
5.  **ポリシー適用**: パケットが FMC を通過する際、送信元 IP を pxGrid から得たマッピングテーブルと照合し、ルールを適用する。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **証明書の重要性**: ラボ試験で pxGrid 連携がうまくいかない最大の原因は証明書です。ISE の Root CA を FMC/WSA にインポートし、信頼関係を完全に確立してください。
*   **pxGrid 設定の順序**: ISE でサービス有効化 → FMC/WSA で ISE 登録 → **ISE 側でクライアントを「Approve」** するというステップを忘れないようにしてください。
*   **時刻同期 (NTP)**: 証明書認証を伴うため、ISE、FMC、WSA の時刻が同期していないとハンドシェイクに失敗します。
*   **ANC (Adaptive Network Control)**: 脅威検知後の「Quarantine」アクションを構成する際、ISE で ANC サービスが有効である必要があります。
*   **FQDN 解決**: pxGrid 通信は FQDN で行われることが多いため、DNS の設定が正しいことを確認します。

---

## 🛠 設定方法

### 1. Cisco ISE：pxGrid の有効化 (GUI)
1.  **Administration > System > Deployment** に移動。
2.  pxGrid サービスを担当するノードを選択し、`pxGrid` にチェックを入れて **Save**。

### 2. Cisco FMC：ISE アイデンティティソースの追加
1.  **Integration > Other Integrations > Identity Sources** に移動。
2.  **Add** をクリックし、ISE の IP/FQDN を入力。
3.  ISE からエクスポートした **pxGrid Server CA Certificate** をアップロード。
4.  自身の **FMC Client Certificate** を選択または生成。
5.  **Test** をクリックし、ISE 側で承認待ち状態にさせる。

### 3. Cisco ISE：クライアントの承認
1.  **Administration > pxGrid Services > Clients** に移動。
2.  `Pending` 状態の FMC/WSA を選択し、**Approve** をクリック。

---

## 🔍 検証コマンド

| 目的 | コマンド / 手法 |
| :--- | :--- |
| **ISE でのサービス状態確認** | <code>show application status ise</code> (pxGrid が running か確認) |
| **ISE pxGrid クライアント確認** | **Administration > pxGrid Services** の画面で `Connected` を確認。 |
| **FMC でのマッピング学習確認** | FMC CLI: <code>show identity-user all</code> |
| **WSA での SGT 学習確認** | WSA CLI: <code>external-data-source status</code> |
| **認証イベントの追跡** | **Operations > RADIUS > Live Logs** |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| pxGrid Status: **Disconnected** | 証明書の信頼関係なし | 各デバイスの Root CA が Trusted ストアにあるか確認。 |
| クライアントが表示されない | ISE サービス未起動 | <code>show application status ise</code> で確認。 |
| **Status: Pending** が続く | ISE での承認漏れ | ISE GUI の pxGrid Clients 画面で手動 Approve する。 |
| ユーザー ID が FMC に反映されない | Session Directory 購読失敗 | ISE で `Session Directory` トピックがパブリッシュされているか確認。 |
| 時刻不一致エラー | NTP 未同期 | <code>show ntp status</code> を各デバイスで実行。 |

---

## ⚠ 制限事項

*   **スケール制限**: 1 つの ISE pxGrid ノードが処理できる最大クライアント数には上限があります。
*   **証明書有効期限**: pxGrid 用の証明書が切れると、全連携が即座に停止します。
*   **暗号化要件**: TLS 1.2 以降が推奨されます。

---

## 🔄 他技術との関連

*   **2.6 Microsegmentation**: ISE から SGT 情報を pxGrid で伝搬し、FMC で SGT ベースのフィルタリングを実現。
*   **4.14 Identity mapping**: パッシブ認証（AD ログ監視）で得た情報を pxGrid で共有。
*   **3.10 Cisco DNAC**: DNAC が ISE と連携し、SD-Access 環境のコンテキストを他デバイスへ共有。

---

## 🧩 比較表

### pxGrid vs RADIUS CoA (Cooperation)

| 特徴 | pxGrid | RADIUS CoA |
| :--- | :--- | :--- |
| **通信形態** | 双方向 / 多対多 (Pub-Sub) | 1対1 (Push型) |
| **速度** | **リアルタイム**（イベント駆動） | 即時的だがセッション単位 |
| **情報量** | **豊富** (SGT, Posture, Device info等) | 限定的 (VLAN変更等) |
| **用途** | 製品間連携、エコシステム統合 | 認証プロセスの動的変更 |

---

## 💡 ベストプラクティス

1.  **専用 PSN の配置**: 大規模環境では、pxGrid サービス専用の PSN (Policy Service Node) を用意し、認証負荷と分離します。
2.  **ワイルドカード証明書の回避**: セキュリティのため、各デバイスに固有の SAN (Subject Alternative Name) を含む証明書を使用します。
3.  **自動 Approve の検討**: ラボ環境以外ではセキュリティ上推奨されませんが、試験等で迅速な構築が必要な場合は `Auto-Approve` 設定の有無を確認します。

---

## 📝 ラボ学習・設定サンプル例

### 1. ISE pxGrid サービスの起動
*   **要件**: ISE-1 ノードで pxGrid プロセスを開始せよ。
*   **手順**: Administration > Deployment > ISE-1 > [x] pxGrid.

### 2. FMC への ISE 証明書の信頼
*   **要件**: ISE の Root CA 証明書を FMC の Trusted Certificate ストアにインポートせよ。

### 3. FMC での ISE Identity Source 設定
*   **問題**: FMC を ISE pxGrid の購読者として構成し、ユーザーセッションを取得せよ。

### 4. ISE での FMC 承認
*   **操作**: ISE GUI で `Pending` になっている FMC クライアントを承認せよ。

### 5. WSA と ISE の pxGrid 連携
*   **要件**: WSA で ISE を外部データソースとして登録し、SGT 情報をインポートせよ。

### 6. FMC でのユーザーベースルールの作成
*   **要件**: AD グループ `HR` に属するユーザーのインターネットアクセスを許可せよ。
*   **前提**: マッピングは pxGrid から取得すること。

### 7. Rapid Threat Containment (RTC) のテスト
*   **要件**: FMC でマルウェアを検知した際、ISE の ANC 属性 `Quarantine` をトリガーせよ。

### 8. pxGrid 証明書の更新
*   **要件**: 期限切れ間近の証明書を CSR 生成から再署名まで行い更新せよ。

### 9. 複数 PSN による pxGrid 冗長化
*   **要件**: 2 台の ISE PSN を pxGrid サーバーとして FMC に登録せよ。

### 10. FMC CLI での同期確認
*   **検証**: `system support diagnostic-cli` から `show identity-user all` を実行し、期待するユーザーが表示されるか確認せよ。

---

## ❓ 想定試験問題

1.  **Design**: pxGrid 連携において、ISE 側でクライアントを承認する前に FMC 側で「Test Connection」が失敗する場合、最も疑うべき点は？
    *   **回答**: **証明書の信頼関係**（FMC が ISE の CA を信頼しているか）および **NTP の同期状態**。
2.  **トラブルシュート**: FMC でアイデンティティソースを構成したが、ユーザー名が IP アドレスにマッピングされない。ISE の pxGrid 設定画面で確認すべき項目は？
    *   **回答**: 該当のクライアントが **Approve** されているか、および **Session Directory** サービスが有効か。
3.  **コンフィグ読解**: FMC の Identity ルールで `Realm` ではなく `ISE` が選択されている理由を述べよ。
    *   **回答**: ユーザー情報を AD から直接引くのではなく、**ISE から pxGrid を介して**動的なコンテキスト情報を取得するため。
4.  **Design**: SGT 情報を WSA で利用したい。どの pxGrid トピックを購読すべきか？
    *   **回答**: **TrustSec Metadata** (SGT) トピック。
5.  **実装**: pxGrid 通信において、サーバー証明書の SAN (Subject Alternative Name) に含めるべき必須情報は？
    *   **回答**: ISE ノードの **FQDN** および必要に応じて **IP アドレス**。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [pxGrid サービスの設定](https://www.cisco.com/c/ja_jp/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31.html)
*   **FMC 7.1 設定ガイド**: [Cisco ISE との統合](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/external_identity_sources.html#task_40859F588B8C4B598E9B1F93427E8C8C)
*   **Cisco Live (BRKSEC-2041)**: [Context Sharing with Cisco pxGrid Deep Dive](https://www.ciscolive.com/)
*   **Technical Note**: [Troubleshoot pxGrid Connectivity Issues](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/214533-troubleshoot-pxgrid-issues.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「ISE = セキュリティ情報の取引所、pxGrid = 通貨（プロトコル）」と考えると分かりやすいです。
*   **図解**: `Client --(Cert)--> ISE (Hub) <--(Cert)-- FMC`。三角形の信頼関係を意識してください。
*   **注意点**: ラボ試験では **"ISE pxGrid node is not approved"** というエラーメッセージを FMC 側で見落とさないように。ISE 側の GUI 操作が必ずセットで必要です。
