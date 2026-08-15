---
layout: default
title: 5.5-Web-filtering
nav_order: 5
parent: 5.0-Advanced-Threat-Protection
---

# 5.5 Web filtering, user identification, and Application Visibility and Control (AVC) on Cisco FTD and Cisco WSA

Cisco FTD (Firepower Threat Defense) と Cisco WSA (Web Security Appliance) は、組織の Web トラフィックを保護し、可視化するための 2 つの主要なコンポーネントです。FTD は次世代ファイアウォール (NGFW) として、インラインでのアプリケーション制御と URL フィルタリングを提供し、WSA は専用のセキュア Web ゲートウェイ (SWG) として、プロキシベースの詳細なコンテンツ検査とユーザ識別を提供します。

---

## 📘 概要

*   **機能概要**: ユーザの Web ブラウジングを「誰が (User ID)」「どのアプリで (AVC)」「どこに対して (URL Filtering)」行っているかを識別し、ポリシーに基づいて制御する機能です。
*   **利用目的**: 不適切なサイトへのアクセス制限、シャドー IT の抑制、マルウェア配布サイトのブロック、および帯域幅の管理。
*   **どのような場面で利用するか**:
    *   **FTD**: ネットワーク境界で、非標準ポートを使用するアプリや Web トラフィックを網羅的にフィルタリングする。
    *   **WSA**: HTTP/HTTPS トラフィックに対して、プロキシとしてキャッシュ、詳細なデータ漏洩防止 (DLP)、および複雑なユーザ認証を適用する。

---

## 🔑 要点

| 項目 | Cisco FTD (FMC 管理) | Cisco WSA |
| :--- | :--- | :--- |
| **主な役割** | NGFW (IP/Port + App + URL) | 透過/明示的プロキシ (HTTP/S 専念) |
| **ユーザ識別** | Identity Policy (Passive/Active) | Identification Profiles |
| **URL フィルタリング** | カテゴリおよびレピュテーション | カテゴリ、レピュテーション、カスタムリスト |
| **AVC** | 4,000 以上のアプリケーション識別 | Web アプリのきめ細やかな動作制御 |
| **展開方式** | ルーテッド、トランスペアレント | WCCP, PBR, 明示的 (Explicit) |
| **HTTPS 検査** | SSL Policy による復号 | HTTPS Proxy による復号 |

---

## 🏗 動作原理

### 通信フロー (WSA - WCCP 透過モードの例)
```text
Client
   ↓ (HTTP Request)
Switch/Router
   ↓ (WCCP Redirect)
Cisco WSA
   ↓ (1. Identification Profile: ユーザ特定)
   ↓ (2. Access Policy: URL/App カテゴリ確認)
   ↓ (3. Anti-Malware/DLP スキャン)
Internet
```

### 通信フロー (FTD - インライン)
```text
Client
   ↓
FTD (Ingress Interface)
   ↓ (Identity Policy: IP-User 紐付け)
   ↓ (Access Control Policy: AVC/URL ルール照合)
   ↓ (Intrusion/File Policy: 脅威スキャン)
FTD (Egress Interface)
```

---

## ⚙ 動作シーケンス

1.  **トラフィック捕捉**: デバイスが Web 通信を受信。WSA の場合はプロキシとして終端するか、WCCP 等で転送される。
2.  **ユーザ識別 (Identity)**:
    *   **Passive**: ISE (pxGrid) や AD Agent からログイン情報を取得。
    *   **Active**: キャプティブポータルを表示し、ユーザ名/パスワードを要求。
3.  **アプリケーション識別 (AVC)**: 第一パケットおよびペイロードのシグネチャを基に、実行されているアプリ（例：Facebook, YouTube）を特定。
4.  **URL カテゴリ照合**: 宛先 URL をクラウドデータベース (CSI: Cisco Systems Intelligence) と照合し、カテゴリ（例：SNS）とレピュテーション（信頼度）を確認。
5.  **ポリシー適用**: 合致するルールの「許可・ブロック・監視」を実行。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **WSA の WCCP 設定**: ルータ側での `ip wccp web-cache` と WSA 側でのサービス ID 設定の整合性は必須です。
*   **Identification Profiles**: 「特定のサブネットのみ認証を要求し、他はゲストとして扱う」といった条件分岐の設定。
*   **FMC Identity Policy**: ISE 経由の Passive 認証と、キャプティブポータルを使用した Active 認証の切り替え。
*   **Application Filter**: 個別のアプリ名ではなく、「カテゴリ：ファイル共有」や「リスク：高」といったフィルタを用いた動的ルールの作成。
*   **Custom URL Category**: 正規表現を用いた特定の URL パターンのブロック。
*   **トラブルシュート**:
    *   WSA: `testauth` コマンドによる LDAP/AD 連携の確認。
    *   FTD: `packet-tracer` による AC ルールのデバッグ。

---

## 🛠 設定方法

### 1. Cisco WSA: URL ブロック設定 (GUI)
1.  **Web Security Manager > Custom and External URL Categories**: 特定のドメインを登録。
2.  **Web Security Manager > Access Policies**: `URL Categories` 列をクリック。
3.  作成したカスタムカテゴリまたは既存カテゴリに対し `Block` を選択。

### 2. Cisco FTD: AVC Policy (FMC GUI)
1.  **Policies > Access Control > Add Rule**。
2.  **Applications** タブで `Application Filters` を使用して「Risk: 5 (Very High)」を選択。
3.  **Action** を `Block` に設定して保存・デプロイ。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **URL カテゴリの確認** | WSA | <code>nslookup</code>, <code>url_category [URL]</code> |
| **ユーザ識別状態の確認** | WSA CLI | <code>authcache</code> |
| **Active セッションの確認** | FTD CLI | <code>show user-identity-policy</code> |
| **現在学習しているユーザ** | FTD CLI | <code>show user-identity state</code> |
| **WCCP 状態確認** | Router | <code>show ip wccp web-cache detail</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| WSA でブロックされない | 透過プロキシのバイパス | ブラウザの Proxy 設定または WCCP の ACL を確認。 |
| FTD でユーザ名が出ない | ISE/AD 通信断 | FMC の `Integration > Sources` で Status が緑か確認。 |
| YouTube がブロック不可 | HTTPS 復号が未設定 | SSL/HTTPS インスペクションを有効にする。 |
| 特定アプリが誤検知される | シグネチャが古い | FMC/WSA のインテリジェンス更新 (VDB) を実行。 |

---

## ⚠ 制限事項

*   **WSA の非標準ポート**: 明示的プロキシとして構成しない限り、ポート 80/443 以外の Web トラフィックはデフォルトで無視される場合があります。
*   **FTD の SSL パフォーマンス**: 大量のトラフィックを復号すると CPU 負荷が急増するため、重要なサイト（金融等）は復号除外リストに入れる設計が必要です。
*   **ライセンス**: URL フィルタリングには有効なサブスクリプション（URL Filtering / Content Security）が必須です。

---

## 🔄 他技術との関連

*   **4.14 Identity mapping**: ISE が提供する SGT (Security Group Tag) を FTD の認可条件として使用可能。
*   **1.1.a Routed Mode**: FTD が L3 デバイスとして動作する際に AVC を適用。
*   **3.6 eStreamer**: FTD が検知した Web イベントを詳細分析するために FMC から外部 SIEM へ転送。

---

## 🧩 比較表

### URL カテゴリ vs URL レピュテーション

| 特徴 | URL カテゴリ (Category) | URL レピュテーション (Reputation) |
| :--- | :--- | :--- |
| **内容** | サイトのジャンル（SNS, ギャンブル等） | サイトの信頼度（1: 危険 〜 5: 安全） |
| **用途** | 業務外サイトの制限 | マルウェア・フィッシング防御 |
| **精度** | 比較的固定 | 動的にスコアが変動 |

---

## 💡 ベストプラクティス

1.  **Reputation-Based Blocking**: カテゴリに関わらず、レピュテーションが「Suspicious」以下のサイトは一律ブロックすることを推奨します。
2.  **SSL Decryption**: 現代の Web の 90% 以上は HTTPS です。復号なしでは AVC も URL カテゴリ検査も精度が著しく低下します。
3.  **WCCP リダイレクトの制限**: WSA へ転送するトラフィックをルータ側の ACL で絞り、不要なトラフィック（内部向け等）による WSA の負荷を軽減します。

---

## 📝 ラボ学習・設定サンプル例

### 1. WSA: 透過プロキシ WCCP 設定
*   **要件**: ルータ Gi0/0 から入る HTTP を WSA (10.1.1.5) へ転送せよ。
*   **Router**: `ip wccp 0 group-list 10`; `access-list 10 permit 10.1.1.5`.
*   **WSA**: `Network > WCCP` でルータ IP とサービス ID 0 を登録。

### 2. FTD: 特定アプリ内の特定動作ブロック
*   **要件**: Facebook の閲覧は許可するが、Facebook Post は禁止せよ。
*   **FMC**: Applications タブで `Facebook` と `Facebook Post` を個別に検索し、Post のみ Block ルールを作成。

### 3. WSA: ユーザ認証の強制
*   **要件**: 内部 LDAP ユーザのみ Web アクセスを許可せよ。
*   **設定**: `Identification Profiles` で `Authenticate Users` をオンにし LDAP を選択。

### 4. FTD: ユーザ名に基づいた認可
*   **要件**: AD の "Marketing" グループのみ動画サイトへのアクセスを許可せよ。
*   **設定**: AC ルールの `Users` タブで該当グループを選択。

### 5. WSA: クォータベースの制限
*   **要件**: ユーザごとに 1 日 100MB 以上のダウンロードを制限せよ。
*   **設定**: `Access Policies > Quotas`.

### 6. FTD: SSL Decryption の例外
*   **要件**: 「Financial Services」カテゴリの通信は復号するな。
*   **設定**: SSL Policy で `Category: Financial` に対して `Do Not Decrypt` アクションを構成。

### 7. WSA: カスタムブロックページの作成
*   **要件**: 遮断時に会社のロゴと連絡先を表示せよ。

### 8. FTD: YouTube 教育チャンネルのみ許可
*   **要件**: 全 YouTube はブロックし、特定の教育用 URL のみ許可せよ。

### 9. WSA: ブラウザの種類による識別
*   **要件**: Mozilla Firefox 以外のブラウザからのアクセスを拒否せよ。
*   **設定**: `Identification Profiles` の User Agent 文字列でフィルタリング。

### 10. FMC: レポートによる AVC 統計の表示
*   **操作**: `Analysis > Dashboard` を開き、最も帯域を消費しているアプリケーションを特定せよ。

---

## ❓ 想定試験問題

1.  **Design**: FTD と WSA の両方がある環境で、社内全体の Web キャッシュを最適化したい。どちらに実装すべきか？
    *   **回答**: **WSA**。FTD はキャッシュ機能を持たないが、WSA は専用のプロキシとして高度なキャッシュが可能。
2.  **トラブルシュート**: FTD で URL フィルタリングを設定したが、すべてのサイトが「Uncategorized」になる。原因は？
    *   **回答**: FTD から **Cisco クラウドへの通信 (TCP 443)** が遮断されている、またはライセンスが未適用。
3.  **実装**: WSA で HTTPS サイトをブロックした際、ユーザにブロックページを表示させるために必要な設定は？
    *   **回答**: **HTTPS Proxy** の有効化と、WSA のルート証明書をクライアントに配布すること。
4.  **コンフィグ読解**: WSA のアクセスポリシーで `Identification Profile: All` となっている場合、認可は何に基づき行われるか？
    *   **回答**: すべてのユーザ（ゲスト含む）に対して同一のポリシーが適用される。
5.  **Design**: 透過プロキシ環境で、WSA がダウンした際に通信を継続（バイパス）させる設定は？
    *   **回答**: ルータ側で WCCP の **service-list** に `fail-open` 相当のロジック（ACL での制御）を持たせる。

---

## 🔗 参考リソース

*   **Cisco WSA User Guide**: [Configuring Web Filtering](https://www.cisco.com/c/en/us/td/docs/security/wsa/wsa12-0/user_guide/b_WSA_UserGuide_12_0.html)
*   **Cisco Secure Firewall (FTD) Config Guide**: [Access Control and AVC](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/access_control_policies.html)
*   **Cisco Live (BRKSEC-3020)**: [Troubleshooting Firepower AVC/URL](https://www.ciscolive.com/)
*   **CVD**: [Secure Web Gateway Deployment Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/wsa-deployment-guide.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「FTD は門番、WSA は執事」です。FTD は通り過ぎるパケットを監視し、WSA はユーザの代わりに Web サイトへ行き、内容を精査して持ち帰ります。
*   **注意点**: ラボ試験では、**WCCP のルータ側インターフェイスでの `ip wccp redirect in/out`** の向きを間違えると、トラフィックが WSA に届かず詰まってしまうため、フローを紙に書いて確認してください。
