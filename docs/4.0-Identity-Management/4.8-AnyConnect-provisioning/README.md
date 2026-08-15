---
layout: default
title: 4.8-AnyConnect-provisioning
nav_order: 8
parent: 4.0-Identity-Management
---

# 4.8 Provisioning Cisco AnyConnect with Cisco ISE and Cisco ASA

Cisco AnyConnect プロビジョニングは、Cisco ISE (Identity Services Engine) と Cisco ASA (Adaptive Security Appliance) を連携させ、エンドポイントに対して AnyConnect クライアント、特定のモジュール（Posture, NAM, Umbrella等）、およびプロファイルを自動的に配布・更新するプロセスです。CCIE Security v6.1 においては、リモートアクセス VPN 接続時に、クライアントが最新のソフトウェアとポリシーを維持し、検疫（Posture）を通過させるための一連のワークフローを構築する能力が問われます。

---

## 📘 概要

*   **機能概要**: ネットワーク接続時（VPN または 有線/無線 LAN）に、エンドポイントの状態を ISE が判定し、必要に応じて AnyConnect ソフトウェアや各種設定ファイルをプッシュする機能です。
*   **利用目的**: 管理者が手動でデバイスをセットアップすることなく、常に最新のセキュリティエージェントと設定を維持することを目的とします。
*   **どのような場面で利用するか**:
    *   **リモートワーク**: 社外から ASA 経由で VPN 接続する際、Posture モジュールを自動インストールしてパッチ適用状況をチェックする。
    *   **コンプライアンス維持**: 特定のウイルス対策ソフトが導入されていない端末に対し、AnyConnect 経由で修正を促す。
    *   **NAM の展開**: 802.1X 設定を管理するための Network Access Manager モジュールを一括配布する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要コンポーネント** | Cisco ISE (Policy Server), Cisco ASA (Headend), AnyConnect Client。 |
| **プロビジョニング方式** | Web-Deploy (ASA 経由) または Client Provisioning Portal (ISE 経由)。 |
| **ISE ポリシー要素** | Client Provisioning Policy, Resources (AnyConnect PKG, Profile), Results。 |
| **ASA の役割** | VPN セッションの終端、ISE への RADIUS 要求、リダイレクト処理の実行。 |
| **連携プロトコル** | RADIUS, HTTPS, Change of Authorization (CoA)。 |
| **チェック対象** | OS バージョン、ユーザーグループ、デバイスタイプ。 |

---

## 🏗 動作原理

AnyConnect のプロビジョニングは、ISE の「Client Provisioning Policy」を核として動作します。

```text
AnyConnect Client (Old/None)
      ↓ (1) VPN Connection Attempt
Cisco ASA (VPN Gateway)
      ↓ (2) RADIUS Access-Request
Cisco ISE (Policy Server)
      ↓ (3) Access-Accept with Redirect URL (Provisioning Portal)
Cisco ASA
      ↓ (4) HTTP Redirect to ISE Portal
AnyConnect Client
      ↓ (5) Posture/Provisioning Agent Download & Run
Cisco ISE
      ↓ (6) Push AnyConnect PKG & Profiles based on Policy
AnyConnect Client
```

1.  **初期認証**: ユーザーが ASA 経由でログインします。
2.  **ポリシー照合**: ISE は接続元の OS やユーザー属性を確認し、プロビジョニングが必要と判断します。
3.  **リダイレクト**: クライアントは ISE のプロビジョニングポータルへリダイレクトされます。
4.  **展開**: クライアント上で AnyConnect Web Helper が動作し、必要なモジュールをダウンロード・インストールします。

---

## ⚙ 動作シーケンス

1.  **AnyConnect 認証**: ASA がクライアントからの VPN 接続を受け、ISE に RADIUS 認証を要求します。
2.  **Client Provisioning 照合**: ISE は `Policy > Client Provisioning` 設定に基づき、適切な AnyConnect バージョンとプロファイルを選択します。
3.  **プロビジョニングリソースの提供**:
    *    AnyConnect パッケージ（`.pkg`）
    *    Posture プロファイル（`VPN_Posture.xml`）
    *    ISE 中間証明書（必要な場合）
4.  **インストールと再起動**: クライアント側でモジュールが有効化され、必要に応じて VPN セッションが更新されます。
5.  **CoA (Change of Authorization)**: プロビジョニング完了後、ISE は ASA に対して CoA を送信し、制限されていたアクセス権限を解除します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **リソースのアップロード**: `Policy > Policy Elements > Results > Client Provisioning > Resources` で、`.pkg` ファイルを ISE にアップロードする手順を正確に行う必要があります。
*   **プロファイルエディタ**: Posture プロファイルや AnyConnect VPN プロファイルを ISE 上の GUI エディタで作成させられるケースが多いです。
*   **ASA の `group-policy`**: ASA 側で `vbis` (Web-deploy) 設定や、AnyConnect パッケージの優先順位を `anyconnect image` コマンドで指定する設定が必要です。
*   **リダイレクト ACL**: ポスチャやプロビジョニングのために、ASA 上で特定のトラフィックのみを ISE にリダイレクトする ACL を作成し、DAP (Dynamic Access Policy) または RADIUS 属性で適用します。
*   **証明書の不一致**: ISE のポータル証明書が信頼されていないとプロビジョニングが失敗するため、信頼チェーンの構築を確認します。

---

## 🛠 設定方法

### 1. Cisco ISE 側の構成 (Client Provisioning)
1.  **リソース追加**: `Policy > Policy Elements > Results > Client Provisioning > Resources` で AnyConnect パッケージをインポート。
2.  **プロファイル作成**: 同画面で `AnyConnect Posture Profile` を新規作成し、ISE のサーバー名を指定。
3.  **設定の統合**: `AnyConnect Configuration` を作成し、PKG ファイル、Posture プロファイルを紐付ける。
4.  **ポリシー作成**: `Policy > Client Provisioning` で、`Windows All` などの条件に対し、上記設定を割り当てる。

### 2. Cisco ASA 側の構成 (AnyConnect VPN)
```bash
! AnyConnectパッケージの指定
anyconnect image disk0:/anyconnect-win-4.10-webdeploy-k9.pkg 1
anyconnect enable

! RADIUSサーバ（ISE）の定義
aaa-server ISE_GROUP protocol radius
aaa-server ISE_GROUP (inside) host 10.1.1.100
 key cisco123

! グループポリシーでの AnyConnect 設定
group-policy VPN_POLICY internal
group-policy VPN_POLICY attributes
 vpn-tunnel-protocol ssl-client
 address-pool local VPN_POOL
! ISEからのプロビジョニングを許可するためのリダイレクト属性（通常ISEからプッシュ）
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **ASA での VPN ユーザー状態確認** | <code>show vpn-sessiondb detail anyconnect</code> |
| **ISE でのプロビジョニングログ** | **Operations > Reports > Endpoints and Users > Client Provisioning** |
| **RADIUS 属性の送受信確認** | <code>debug radius all</code> (ASA) |
| **AnyConnect 側のログ** | AnyConnect UI の **Statistics > Message History** |
| **Posture 状態の確認** | <code>show access-session interface [int] details</code> (有線の場合) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| リダイレクトが発生しない | 認可ポリシーにリダイレクト ACL が含まれていない | ISE の **Authorization Profile** で `Web Redirection` がチェックされているか確認。 |
| ポータルで「PKG not found」 | ISE へのリソースアップロード不足 | `Resources` に正しい OS 用の PKG が登録されているか確認。 |
| AnyConnect が更新されない | ASA の `anyconnect image` 設定ミス | ASA 側のパッケージパスとバージョンを確認。 |
| CoA 失敗 | UDP 1700 が Firewall で遮断されている | ASA と ISE 間の通信経路上で CoA ポートを開放する。 |

---

## ⚠ 制限事項

*   **OS バージョン**: Linux などの一部の OS では、ISE 経由の自動プロビジョニング（Web-Launch）が限定的である場合があります。
*   **Java/ActiveX**: 以前はブラウザの Java/ActiveX を使用していましたが、現在は **AnyConnect Web Helper** (HTTPS) への移行が推奨されています。
*   **ライセンス**: AnyConnect Apex ライセンス（現在は Advantage）が Posture 機能の利用に必要です。

---

## 🔄 他技術との関連

*   **4.4 802.1X**: 有線 LAN 環境での AnyConnect NAM モジュールのプロビジョニング。
*   **2.1.a FTD Routed Mode**: Firepower への VPN 移行。設定ロジックは ASA と類似しているが FMC で管理する。
*   **3.10 Cisco DNAC**: SD-Access 環境でのエンドポイント制御との統合。

---

## 🧩 比較表

### ASA Web-Deploy vs ISE Client Provisioning

| 特徴 | ASA Web-Deploy | ISE Client Provisioning |
| :--- | :--- | :--- |
| **主な配布物** | AnyConnect クライアント本体 | **モジュール (Posture/NAM) + プロファイル** |
| **ポリシーの細かさ** | グループポリシー単位 | **ユーザー・OS・グループ・サイト単位** |
| **主な用途** | VPN 接続の確立 | **検疫 (Posture) の実行と設定更新** |
| **複雑さ** | 低 | 高 |

---

## 💡 ベストプラクティス

1.  **段階的な展開**: AnyConnect の新バージョンを展開する際は、Client Provisioning Policy で特定のテストグループのみに適用し、動作を確認します。
2.  **外部配布の検討**: 大規模な PKG ファイルを ISE から多数の端末にプッシュすると、ISE (PSN) の負荷と帯域を圧迫するため、可能であれば CDN やローカル Web サーバーを活用します。
3.  **共通プロファイルの作成**: 複数の PSN がある場合、各 PSN 共通で利用できる Posture プロファイルを構成します。

---

## 📝 ラボ学習・設定サンプル例

### 1. AnyConnect PKG の ISE への登録
*   **要件**: Windows 用 AnyConnect バージョン 4.10 を ISE リソースに追加せよ。
*   **手順**: Policy > Resources > Add > Cisco Provided Packages。

### 2. Posture プロファイルのエディタ作成
*   **要件**: ISE サーバー名 `ise-psn.lab.local` を参照する Posture プロファイルを作成せよ。

### 3. Client Provisioning Policy の定義
*   **要件**: `Domain Users` グループかつ `Windows` OS の場合にプロビジョニングを実行せよ。

### 4. ASA リダイレクト ACL の作成
*   **設定例**: `access-list REDIRECT_ACL extended deny udp any any eq 53` (DNS除外), `access-list REDIRECT_ACL extended permit tcp any any eq 80`。

### 5. 認可プロファイル (Authorization Profile) の紐付け
*   **要件**: 認可結果として `AnyConnect-Redirect` を返し、URL と ACL を ASA へプッシュせよ。

### 6. SSL VPN 上での AnyConnect 更新テスト
*   **課題**: 古いバージョンの AnyConnect で接続し、自動アップグレードが開始されることを確認せよ。

### 7. Posture モジュールのサイレントインストール
*   **要件**: ユーザーの操作なしに Posture モジュールをバックグラウンドでインストールせよ。

### 8. NAM (Network Access Manager) プロファイル配布
*   **要件**: 有線ポート接続用の XML 設定ファイルを AnyConnect Configuration に含めて配布せよ。

### 9. 証明書配布 (Certificate Provisioning)
*   **要件**: AnyConnect 接続中に、SAML/SCEP 経由でユーザー証明書をプッシュせよ。

### 10. プロビジョニング後の CoA 確認
*   **検証**: プロビジョニング成功後、ASA 上のセッションがリセットされ、フルアクセスが付与されることを `show vpn-sessiondb` で確認せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: ISE の Client Provisioning 設定で `AnyConnect Configuration` が指定されていない場合、エンドポイントに何が起こるか？
    *   **回答**: リダイレクトは発生するが、ポータル上で配布するリソースが見つからず、プロビジョニングに失敗する。
2.  **トラブルシュート**: AnyConnect クライアントが Posture チェックをスキップしてしまう。ISE の Client Provisioning Policy で確認すべき点は？
    *   **回答**: 該当する OS とユーザーグループに対して、Posture モジュールを含む `AnyConnect Configuration` が正しく割り当てられているか確認。
3.  **Design**: プロビジョニング中にインターネット上の Google などへアクセスさせたい場合、ASA のリダイレクト ACL はどう構成すべきか？
    *   **回答**: ACL の先頭に `deny ip any [Google_IP_Range]` を記述し、リダイレクトから除外する。
4.  **実装**: ASA 側で特定の VPN グループにのみプロビジョニングを強制するために使用する ASA のコマンドは？
    *   **回答**: `group-policy [name] attributes` 配下での `vbis` もしくは ISE からプッシュされる `url-redirect` 属性の受け入れ。
5.  **Design**: 拠点ごとに異なる AnyConnect プロファイルを配布したい。ISE のどのポリシー階層で制御すべきか？
    *   **回答**: **Client Provisioning Policy** の条件（Condition）に `Location` 属性を追加して分離する。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [クライアントプロビジョニングの設定](https://www.cisco.com/c/ja_jp/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31/b_ise_admin_guide_31_chapter_010111.html)。
*   **ASA Configuration Guide**: [AnyConnect VPN Client Management](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/vpn/asa-94-vpn-config.html)。
*   **Cisco Live (BRKSEC-2024)**: [AnyConnect and ISE Deployment Best Practices](https://www.ciscolive.com/)。
*   **CVD**: [AnyConnect Secure Mobility Client with ISE Posture](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/ise-deployment-guide.html)。

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「ASA はトンネルを作り、ISE はトンネルを通る中身（AnyConnect）を整える」という役割分担を理解してください。
*   **図解**: プロビジョニングは一回限りのプロセスではなく、バージョンアップのたびにトリガーされます。
*   **注意点**: ラボ試験では、**Windows の「隠しファイル」や「プロファイル保存先」**のトラブル（クライアント側の問題）は対象外ですが、ISE 上のプロファイルエディタで **XML 構文エラー** を起こさないよう、常に「Save」後にエディタを閉じる癖をつけましょう。
