---
layout: default
title: 1.6.a-SSL-inspection
nav_order: 1
parent: 1.6-NGFW-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.6.a SSL inspection

Cisco Secure Firewall (FTD) における **SSL inspection**（SSLポリシー）は、暗号化されたトラフィック（HTTPS等）を復号し、内部のパケットを次世代ファイアウォール（NGFW）機能や次世代IPS（NGIPS）で検査可能にするための不可欠な機能です。SSLポリシーは、アクセスコントロールポリシー（ACP）とは独立した固有の機能として提供されます。

---

## 📘 概要

*   **機能概要**: TLS/SSLで暗号化された通信を、Firewallが「中間者（MITM）」として仲介し、ペイロードを平文として取り出して検査します。
*   **利用目的**: 暗号化通信の中に隠れたマルウェア、攻撃シグネチャ、または不適切なURLやファイルを検知・遮断するために利用されます。
*   **どのような場面で利用するか**: ユーザーのインターネット閲覧（アウトバウンド）における脅威防御、および自社Webサーバ（インバウンド）への攻撃保護の双方で利用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な特徴** | ポリシーは複数のルールで構成され、**上から順に評価（First-match）**される。 |
| **用途** | 暗号化トラフィックの可視化、コンプライアンス（金融・医療サイトの除外）。 |
| **メリット** | Snortエンジン（IPS）によるL7検査を暗号化通信に対しても実行可能。 |
| **デメリット** | 復号処理によるデバイスのCPU負荷増大とレイテンシの発生。 |
| **対応機種** | Firepower Threat Defense (FTD), ASA with Firepower Services。 |
| **復号方式** | **Decrypt - Resign**（アウトバウンド用）、**Decrypt - Known Key**（インバウンド用）。 |
| **設計上の注意** | 信頼された認証局（Internal CA）の証明書を事前にFMCにインポートし、クライアントに配布する必要がある。 |

---

## 🏗 動作原理

SSLポリシーは、パケットがアクセスコントロールポリシーで詳細に処理される前に適用されます。

```text
Encrypted Packet (TLS)
   ↓
SSL Policy Lookup (FMC)
   ↓
Match Rule? (Top-down)
   ↓
Action: Decrypt (Resign / Known Key)
   ↓
Plaintext Payload
   ↓
Snort Engine (IPS / Malware / URL filtering)
   ↓
Re-encrypt / Verdict (Allow / Block)
   ↓
Outside Interface
```

---

## ⚙ 動作シーケンス

1.  **SSLハンドシェイクの検知**: FTDがクライアントとサーバ間のClient Helloを傍受します。
2.  **ポリシー評価**: SSLルールを上から順に評価し、送信元/宛先IP、ポート、URLカテゴリ、証明書ステータス等に基づきマッチングします。
3.  **復号アクションの実行**:
    *   **Decrypt - Resign**: FTDがサーバになりすましてクライアントと接続し、内部CAで署名し直した証明書を提示します。
    *   **Decrypt - Known Key**: 事前に登録したサーバの秘密鍵を使用して復号します。
4.  **内部検査**: 復号された平文パケットがSnortエンジンに送られ、IPSポリシーやファイルポリシーの検査を受けます。
5.  **再暗号化と送出**: 検査に合格したパケットは再度暗号化され、宛先へ転送されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **CA証明書の管理**: FMCで **Objects > Object Management > PKI > Internal CAs** に署名用証明書を正しく登録する手順が頻出です。
*   **Do Not Decrypt の使い分け**: 銀行や医療サイトなど、プライバシー要件で「復号してはいけない」トラフィックをカテゴリベースで除外する設定が問われます。
*   **TLS 1.3 への対応**: TLS 1.3ではハンドシェイク自体が暗号化されるため、復号には特定の互換性設定やダウングレード（SNIの読み取り等）が必要になる点に注意してください。
*   **Undecryptable（復号不能）トラフィック**: 期限切れ証明書やサポート外の暗号スイート（Cipher Suite）を持つ通信に対し、Blockするか通過させるかのグローバル設定。
*   **debugコマンドの活用**: ラボでトラブルが発生した際、`system support firewall-engine-debug` でSSL判定プロセスを確認できる必要があります。

---

## 🛠 設定方法

### 1. 内部CA証明書の登録
FMC GUIの `Objects > Object Management > PKI > Internal CAs` で、署名に使用するCA証明書と秘密鍵をアップロードします。

### 2. SSLポリシーの作成
`Policies > SSL` に移動し、**Add SSL Policy** をクリックします。

### 3. ルールの追加 (例: Decrypt - Resign)
1.  **Add Rule** をクリック。
2.  **Action** を `Decrypt - Resign` に設定。
3.  **Certificate** タブで、手順1で登録した Internal CA を選択。
4.  **Network** や **Category** タブで対象を指定し、保存します。

### 4. ACPへの紐付け
`Policies > Access Control` の対象ポリシーを編集し、**Advanced** タブの **SSL Policy Settings** で作成したSSLポリシーを選択して保存・デプロイします。

---

## 🔍 検証コマンド

| 目的 | コマンド（FTD CLI） |
| :--- | :--- |
| **適用されているポリシーの確認** | <code>show service-policy inspect snort</code> |
| **SSL復号のリアルタイムデバッグ** | <code>system support firewall-engine-debug</code> |
| **SSL統計情報（復号数等）の表示** | <code>show ssl-stats</code> (LINA) |
| **現在のアクティブなSSLセッション確認** | <code>show asp table socket</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| ブラウザで「安全ではない接続」と表示される | Internal CAがクライアントに配布されていない | FMCからInternal CAをエクスポートし、PCの「信頼されたルート証明書機関」にインポート。 |
| 特定のサイトだけ接続できない | 暗号スイートの不一致、またはTLS 1.3の制限 | SSLポリシーの **Undecryptable Actions** を確認。 |
| パフォーマンスが極端に低下する | 大量のセッションによるCPU過負荷 | <code>show cpu</code> でSnortの負荷を確認。不要な通信を <code>Do Not Decrypt</code> に。 |
| IPSイベントが発生しない | SSLポリシーがACPに紐付いていない | ACPのAdvancedタブでSSLポリシーの選択を再確認。 |

---

## ⚠ 制限事項

*   **証明書ピンニング (Certificate Pinning)**: Dropboxや一部のモバイルアプリ等、証明書を固定しているアプリは復号を行うと通信が切断されます。これらは `Do Not Decrypt` に設定する必要があります。
*   **ハードウェア制限**: SSL復号の最大セッション数は、デバイスのモデル（RAM容量）によって制限されます。
*   **パスワード保護されたファイル**: 暗号化されたアーカイブ内のファイルは、SSL復号を行っても中身のマルウェア検査ができない場合があります。

---

## 🔄 他技術との関連

*   **Access Control Policy**: SSLポリシーはACPの補助機能として動作し、暗号化解除後のトラフィックをACPの各ルール（IPS, Malware等）に渡します。
*   **URL Filtering**: SSL復号を行わない場合、SNI情報からドメイン名までは特定できますが、詳細なパス（URLの後半部分）の制御にはSSL復号が必須です。
*   **Snort Engine**: SSLポリシーによって平文化されたパケットの実際のインスペクションを担当します。

---

## 🧩 比較表

### Decrypt - Resign vs Decrypt - Known Key

| 特徴 | Decrypt - Resign (再署名) | Decrypt - Known Key (既知の鍵) |
| :--- | :--- | :--- |
| **主な用途** | 内部から外部サイトへの通信保護 | 外部から自社サーバへの通信保護 |
| **必要となる鍵** | **内部CA証明書と秘密鍵** | **対象サーバ自体の秘密鍵** |
| **クライアントへの影響** | 内部CAの信頼設定が必要 | 影響なし（透過的） |
| **MITMの有無** | あり（セッションが分割される） | 原則なし（パッシブまたはプロキシ） |

---

## 💡 ベストプラクティス

1.  **段階的な導入**: 最初は全ルールを `Monitor` または `Do Not Decrypt` でログのみ取得し、影響範囲を特定してから復号を有効にします。
2.  **カテゴリによる除外**: 金融（Finance）、ヘルスケア（Health and Medicine）などの機密カテゴリは、最初から `Do Not Decrypt` に設定し、ログを無効にすることを検討します（プライバシー保護）。
3.  **証明書ステータスの活用**: 有効期限切れや、自己署名証明書を持つサーバへの通信をSSLポリシーレベルで一括ブロックし、セキュリティを高めます。

---

## 📝 ラボ学習・設定サンプル例

### 1. インターネット通信の基本復号 (Resign)
*   **要件**: 内部ネットワークからのHTTPS通信を復号せよ。
*   **設定**: Action: `Decrypt - Resign`。Internal CAを指定。

### 2. 特定カテゴリ（銀行・医療）の除外
*   **要件**: 金融関連サイトの復号をスキップせよ。
*   **設定**: Category: `Finance`。Action: `Do Not Decrypt`。

### 3. 内部Webサーバの保護 (Known Key)
*   **要件**: IP `192.168.10.100` のHTTPSサーバ宛の通信を復号せよ。
*   **設定**: Objects > Certificates にサーバ証明書/鍵を登録。Action: `Decrypt - Known Key`。

### 4. 弱いTLSバージョンの遮断
*   **要件**: TLS 1.1以下の古いプロトコルを拒否せよ。
*   **設定**: SSLルール内の **Cipher Suite** またはプロトコルオプションで、TLS 1.0/1.1 を選択し Action: `Block`。

### 5. 期限切れ証明書のブロック
*   **要件**: 有効期限が切れた証明書を持つサイトへの接続を阻止せよ。
*   **設定**: SSL Policyの **Undecryptable Actions** セクションで、`Expired Certificate` を `Block` に設定。

### 6. SNIによるフィルタリング検証
*   **課題**: 復号なし（Do Not Decrypt）の状態でURLフィルタリングがドメイン単位で動作することを確認せよ。

### 7. 復号後のIPSマッチング
*   **要件**: HTTPS通信内の特定のEICARテストファイルをAMPで検知せよ。
*   **設定**: SSL復号設定 ＋ ACPルールにFile Policyを適用。

### 8. 内部CA証明書のエクスポート
*   **課題**: FMCから署名用CA証明書を取り出し、テスト端末へ手動インポートせよ。

### 9. Cipher Suite ベースの除外
*   **要件**: 復号不可能な特定の強力な暗号スイート通信をバイパスさせよ。

### 10. SSLセッション制限の監視
*   **コマンド**: FTD CLIで復号によるリソース消費を `show memory` と `show cpu` で比較確認せよ。

---

## ❓ 想定試験問題

1.  **実装**: 内部の特定の管理端末グループのみ、インターネットアクセス時に全てのHTTPS通信を復号してIPS検査を行う設定を完了せよ。
2.  **トラブルシュート**: SSL復号を有効にした後、特定の社内アプリケーションが「証明書エラー」で動作しなくなった。どのオブジェクト設定を確認すべきか？
    *   **回答**: Internal CA証明書が対象アプリケーションによって信頼されているか、またはアプリケーションが証明書ピンニングを使用していないか。
3.  **Design**: 1,000名のユーザーがいる拠点でSSL復号を全有効にする場合、考慮すべきハードウェアリソースは何か？
    *   **回答**: デバイスのスループット（SSL性能）と、同時に処理可能なSSLフロー数。
4.  **コンフィグ読解**: SSLポリシーに `Decrypt - Resign` アクションのルールがある。この時、Snortエンジンが処理するパケットは暗号化されているか？
    *   **回答**: いいえ、Snortは復号された後の平文パケットを処理します。
5.  **実装**: TLS 1.3トラフィックにおいてSNI情報が見えない場合、どのように対応すべきか？

---

## 🔗 参考リソース

*   **Configuration Guide**
    *   [Cisco Secure Firewall Management Center Administration Guide, 7.1 - SSL Policies](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/ssl_policies.html)
*   **Cisco Live (Slides)**
    *   BRKSEC-3020: Troubleshooting Firewall Threat Defense (FTD)
*   **Technical Notes**
    *   [Firepower SSL Inspection - Features and Performance](https://www.cisco.com/c/en/us/support/docs/security/firepower-management-center/212354-configure-syslog-on-firepower-firewall-m.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: SSLポリシーは「どこで設定するか」だけでなく「どうやってSnortに渡されるか」を意識してください。
*   **図解**: 常に「暗号化された箱」を一度開けて、中身を検査し、また「別の箱（内部CA署名）」に詰め直すイメージを持つと理解が早まります。
*   **注意点**: ラボ試験ではInternal CAの「秘密鍵」をセットでインポートし忘れるミスが多いため、設定時は必ずペアであることを確認してください。
