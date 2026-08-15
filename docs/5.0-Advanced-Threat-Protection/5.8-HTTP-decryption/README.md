---
layout: default
title: 5.8-HTTP-decryption
nav_order: 8
parent: 5.0-Advanced-Threat-Protection
---

# 5.8 HTTP decryption and inspection on Cisco FTD, Cisco WSA, and Cisco Umbrella

HTTP(S) 復号とインスペクションは、暗号化されたトラフィック（HTTPS）の中に隠れた脅威を可視化し、制御するための不可欠な技術です。現代の Web トラフィックの 90% 以上が暗号化されているため、復号なしでは次世代ファイアウォール（NGFW）やセキュア Web ゲートウェイ（SWG）の高度な機能（IPS、AMP、URL フィルタリングなど）が十分に機能しません。

---

## 📘 概要

*   **機能概要**: ネットワークデバイスが「中間者（Man-in-the-Middle）」として動作し、クライアントとサーバ間の SSL/TLS セッションを終端・復号して、通信の中身を検査する機能です。
*   **利用目的**: 暗号化通信内のマルウェア検知、データ漏洩防止（DLP）、詳細なアプリケーション識別（AVC）、およびポリシー違反の防止。
*   **どのような場面で利用するか**:
    *   **脅威防御**: HTTPS 経由でダウンロードされる実行ファイルからマルウェアを検出する場合。
    *   **コンプライアンス**: SNS や Web メール経由での機密情報送信を監視する場合。
    *   **バイパス制御**: 銀行や医療サイトなど、プライバシー保護が必要な通信を除外（Bypass）し、それ以外を検査する場合。

---

## 🔑 要点

| 項目 | Cisco FTD (FMC 管理) | Cisco WSA | Cisco Umbrella (SIG) |
| :--- | :--- | :--- | :--- |
| **主要ポリシー** | SSL Policy | HTTPS Proxy Policy | Web Policy (HTTPS Inspection) |
| **復号アクション** | Decrypt-Resign, Decrypt-Known Key, Block, Do not decrypt | Decrypt, Drop, Pass-through (Monitor) | Inspection On/Off, Selective Decryption |
| **証明書管理** | Internal CA (Resign 用), Internal Cert (Known Key 用) | Root CA / Trusted Root 局 | Umbrella Root CA |
| **主な用途** | 境界防御、IPS/AMP 連携 | プロキシ、詳細な Web 制御 | クラウド型 SIG、オフネット保護 |
| **パフォーマンス** | ハードウェアアクセラレーション依存 | 接続数とスループットに依存 | クラウド性能に依存 |

---

## 🏗 動作原理

HTTP 復号は、一般的に **Decrypt-Resign（再署名）** 方式で動作します。

```text
Client Browser (Trusts Internal CA)
   ↓ (1) HTTPS Request to google.com
Cisco Device (FTD/WSA/Umbrella)
   ├── (2) Terminate Client SSL Session
   ├── (3) Establish Server SSL Session to google.com
   ├── (4) Inspect Decrypted Payload (IPS, AMP, AVC)
   └── (5) Generate Spoofed Cert for google.com (Signed by Internal CA)
   ↓ (6) Send Spoofed Cert to Client
Client Browser (Validates via Internal CA)
```

---

## ⚙ 動作シーケンス

1.  **ハンドシェイク捕捉**: クライアントからの `Client Hello` をデバイスが受信します。
2.  **ポリシー照合**: `Client Hello` 内の SNI (Server Name Indication) や証明書情報を基に、復号ポリシー（SSL Policy 等）にマッチするか判断します。
3.  **セッション終端**: デバイスはサーバのふりをしてクライアントとセッションを確立し、同時にクライアントのふりをして実際のサーバとセッションを確立します。
4.  **クリアテキスト検査**: 復号されたデータに対し、Access Control Policy (FTD) や Access Policy (WSA) が適用され、IPS、ファイル検査、URL フィルタリングが実行されます。
5.  **再暗号化**: 検査をパスしたデータを再度暗号化し、宛先へ転送します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **CA 証明書のインポート**: デバイス（FTD/WSA）に証明書を発行するための Root CA または中間 CA を正しくインポート・設定できることが前提です。
*   **SSL Policy の配置**: FTD において、SSL Policy は Access Control Policy (ACP) に「関連付け（Associate）」させる必要があります。関連付けを忘れると復号されません。
*   **証明書の信頼（Client Side）**: ラボ内の Windows 端末やブラウザに、デバイスが署名に使用する CA 証明書をインストールし、警告が出ないように設定する手順を確認してください。
*   **Bypass 設定**: 「金融（Finance）」や「医療（Healthcare）」カテゴリを復号から除外する（Do Not Decrypt）要件が頻出です。
*   **トラブルシュート**:
    *   **Untrusted Authority**: クライアントがデバイスの CA を信頼していない。
    *   **Cipher Mismatch**: デバイスが対応していない暗号スイートをサーバが要求している。

---

## 🛠 設定方法

### 1. Cisco FTD (FMC 管理): SSL Policy 設定
1.  **Objects > Object Management > PKI > Internal CAs**: 再署名用 CA を登録。
2.  **Policies > SSL**: `Add SSL Policy` をクリック。
3.  **Add Rule**:
    *   `Action`: **Decrypt - Resign**.
    *   `Certificate`: 手順1で登録した CA を指定。
    *   `Category`: 復号対象（例: SNS）を選択。
4.  **Policies > Access Control**: 該当の ACP を開き、`Settings > SSL Policy` で作成したポリシーを選択して保存・デプロイ。

### 2. Cisco WSA: HTTPS Proxy 設定
1.  **Security Services > HTTPS Proxy**: `Enable` をクリック。
2.  **Edit Settings**: `Use Root CA` または `Upload Root CA` を設定。
3.  **Web Security Manager > HTTPS Proxy**: `Decryption Policies` で `Decrypt` アクションを設定。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **SSL 統計情報の表示** | FTD CLI | <code>show ssl-statistics</code> |
| **現在の復号セッション確認** | FTD CLI | <code>show conn detail</code> |
| **ポリシーマッチ確認** | FMC | `Analysis > Unified Events` (SSL Policy 情報を確認) |
| **HTTPS プロキシ状態** | WSA CLI | <code>proxystat</code> |
| **特定の URL 判定** | WSA CLI | <code>testauthconfig</code> |
| **パケット追跡** | FTD CLI | <code>packet-tracer input inside tcp [SrcIP] [Port] [DstIP] 443</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| **ブラウザで警告が出る** | Root CA が端末に未配布 | 証明書を信頼されたルート証明機関へインストールする。 |
| **一部のサイトが開かない** | 暗号スイートの不一致 | `show ssl-statistics` でドロップを確認し、SSL Policy で `Action: Block` になっていないか確認。 |
| **復号アクションが無視される** | SSL Policy の未適用 | ACP の Settings タブで SSL Policy が正しく選択されているか確認。 |
| **Umbrella で復号されない** | Web トラフィックが SIG を通っていない | PAC ファイルまたは AnyConnect SWG トンネルの状態を確認。 |
| **CPU 負荷が非常に高い** | 復号対象が多すぎる | 動画サイトや Windows Update などを除外リストに追加する。 |

---

## ⚠ 制限事項

*   **ハードウェアの限界**: 復号は CPU 負荷が非常に高く、スループットが大幅に低下する可能性があります（デバイスによっては 50% 以上低下）。
*   **HSTS (HTTP Strict Transport Security)**: クライアントが以前の直接通信を記憶している場合、中間者攻撃とみなされて接続が拒否されることがあります。
*   **Certificate Pinning**: アプリケーション（Dropbox、一部の Google App 等）が独自の証明書を埋め込んでいる場合、復号すると動作しなくなります。

---

## 🔄 他技術との関連

*   **3.6.a NetFlow/NSEL**: 復号された通信の詳細なフロー情報を外部にエクスポートできます。
*   **5.5 AVC**: 復号されることで、HTTPS 内の特定のマイクロアプリケーション（例: Facebook Post）が識別可能になります。
*   **5.7.e Encryption**: ESA における電子メール暗号化とは異なり、こちらは「インスペクション目的の復号」に特化しています。

---

## 🧩 比較表

### Decrypt-Resign vs Decrypt-Known Key

| 特徴 | Decrypt-Resign (再署名) | Decrypt-Known Key (既知の鍵) |
| :--- | :--- | :--- |
| **主な用途** | アウトバウンド（Web 閲覧） | インバウンド（自社サーバの保護） |
| **証明書要件** | デバイス内に CA 証明書が必要 | 宛先サーバの秘密鍵が必要 |
| **クライアント設定** | CA の配布が必要 | 設定不要 |
| **制御範囲** | 全ての外部サイトが対象可能 | 秘密鍵を所有する特定サーバのみ |

---

## 💡 ベストプラクティス

1.  **Selective Decryption**: 全トラフィックを復号せず、リスクの高いカテゴリ（Uncategorized, Newly Seen Domains）やビジネス目的のみを復号対象にします。
2.  **Privacy Exclusion**: 銀行（Finance）や医療（Health）カテゴリは法律やプライバシーの観点から復号除外に設定することを推奨します。
3.  **Log SSL Errors**: ハンドシェイクエラーや期限切れ証明書のログを有効にし、トラブルシュートを容易にします。

---

## 📝 ラボ学習・設定サンプル例

### 1. FTD: 基本的な再署名復号
*   **要件**: `inside` からの `outside` 宛 HTTPS トラフィックをすべて復号せよ。
*   **設定**: SSL Policy で `Source Zone: inside`, `Destination Zone: outside`, `Action: Decrypt - Resign`。

### 2. FTD: 信頼できない証明書のブロック
*   **要件**: 自己署名証明書や期限切れ証明書を使用しているサーバへのアクセスを遮断せよ。
*   **設定**: SSL Policy の `Undecryptable Actions` で `Expired Certificate` などを **Block** に設定。

### 3. WSA: 特定グループの復号バイパス
*   **要件**: `Executive-Group` 所属ユーザの通信は復号するな。
*   **設定**: `Decryption Policies` で `Identification Profile` を指定し、アクションを **Pass-through** に設定。

### 4. Umbrella: 新規ドメインの強制復号
*   **要件**: `Newly Seen Domains` カテゴリに属する Web サイトを復号してインスペクションせよ。

### 5. FTD: 特定カテゴリの復号除外 (Bypass)
*   **要件**: `Financial Services` と `Health and Recreation` カテゴリは復号するな。
*   **設定**: SSL Policy で該当カテゴリに対し `Action: Do Not Decrypt` を上位ルールに配置。

### 6. FMC: 復号用内部 CA のエクスポート
*   **操作**: `Internal CAs` オブジェクトから Root CA 証明書をダウンロードし、AD 端末の GPO で配布する手順をシミュレートせよ。

### 7. WSA: WCCP 経由の透過 HTTPS 復号
*   **要件**: ルータから WCCP で転送された 443 通信を WSA で復号せよ。

### 8. FTD: 復号後の IPS 適用
*   **要件**: HTTPS 通信を復号し、その中の `SQL Injection` 攻撃を IPS で検知せよ。
*   **設定**: SSL Policy で復号後、ACP の該当ルールに `Intrusion Policy` を適用。

### 9. Umbrella: ルート証明書の検証
*   **操作**: クライアントで `https://dashboard.umbrella.com` にアクセスし、証明書チェーンが Umbrella Root CA になっているか確認せよ。

### 10. FTD: Cipher Suite 制限
*   **要件**: セキュリティの低い `TLS 1.0` 通信を復号せずにブロックせよ。
*   **設定**: SSL Policy で `TLS Version: TLS 1.0` に対して `Action: Block`。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: FTD で SSL Policy を構成したが、`Analysis > Events` に SSL イベントが表示されない。考えられる原因は？
    *   **回答**: Access Control Policy の `Settings` タブで **SSL Policy が関連付けられていない**。
2.  **Design**: HTTPS 復号を導入する際、Web 会議アプリ（Zoom, Teams）の通信が不安定になった。推奨される回避策は？
    *   **回答**: それらのアプリケーションが使用するドメインやカテゴリ（Unified Communications）を SSL Policy で **Do Not Decrypt (Pass-through)** に設定する。
3.  **コンフィグ読解**: `Decrypt - Known Key` アクションを使用する場合、FMC にアップロードする必要があるものは何か？
    *   **回答**: 内部にある実際のサーバの **証明書と秘密鍵（PrivateKey）**。
4.  **実装**: WSA で HTTPS Proxy を有効にしたが、透過モード（WCCP）で 443 ポートがリダイレクトされない。ルータ側で必要な設定は？
    *   **回答**: WCCP のサービス ID (通常 70) を定義し、443 ポートをリダイレクト対象に含める ACL を適用する。
5.  **Design**: 拠点ごとに異なる再署名用 CA を使用したい。FMC でどのように管理すべきか？
    *   **回答**: 各拠点（Device）ごとに異なる SSL Policy を作成するか、単一の SSL Policy 内で `Certificate` をオーバーライドして適用する。

---

## 🔗 参考リソース

*   **Cisco Secure Firewall (FTD) Configuration Guide**: [SSL Inspection Policy](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/ssl_inspection_policies.html)
*   **Cisco WSA User Guide**: [Configuring HTTPS Proxy](https://www.cisco.com/c/en/us/td/docs/security/wsa/wsa12-0/user_guide/b_WSA_UserGuide_12_0.html)
*   **Cisco Live (BRKSEC-3020)**: [Deep Dive into SSL/TLS Inspection on Secure Firewall](https://www.ciscolive.com/)
*   **CVD**: [Secure Web Gateway Deployment Guide (WSA)](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/wsa-deployment-guide.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「SSL 復号は、暗号という封筒を開けて手紙を読む作業」です。封筒を開けない限り、中身が感謝の手紙か脅迫状（ウイルス）か分かりません。
*   **図解**: `Client <--(HTTPS A)--> FTD (Decrypted Data) <--(HTTPS B)--> Server`.
*   **注意点**: ラボ試験では、**日付と時刻 (NTP)** がずれていると証明書の有効期限切れとみなされ、復号がすべて失敗するため、最初に NTP を確認してください。
