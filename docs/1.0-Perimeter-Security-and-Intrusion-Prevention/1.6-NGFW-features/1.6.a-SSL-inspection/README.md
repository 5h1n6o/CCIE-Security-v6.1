---
layout: default
title: 1.6.a-SSL-inspection
nav_order: 1
parent: 1.6-NGFW-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.6.a SSL inspection

Cisco Secure Firewall Threat Defense (FTD) および Management Center (FMC) における **SSL/TLS Inspection（SSL/TLS 暗号化通信の復号・可視化）** は、現代の Web トラフィックの大半を占める暗号化通信（HTTPS, SMTPS, POP3S, FTPS 等）の中身を中間復号（Man-in-the-Middle / Proxy 動作）し、平文となったデータストリームに対して Snort IPS、AVC (Application Visibility and Control)、URL Filtering、File Policy、AMP (Advanced Malware Protection) などのディープパケットインスペクション（DPI）を適用可能にする核心的なセキュリティ機能です。

CCIE Security v6.1 ラボ試験において、SSL Inspection は単に SSL Policy を適用するだけでなく、**PKI オブジェクト（Internal CAs / Internal Certificates / Trusted CAs）の適切な定義、Outbound (Decrypt - Resign) と Inbound (Decrypt - Known Key) の使い分け、Undecryptable Actions（復号不能通信）の制御、Certificate Pinning や mTLS（クライアント証明書認証）への対処、および高負荷時の CPU / ハードウェアアクセラレーション最適化** まで含めた高度な設計・構築・トラブルシューティング能力が厳格に問われます。

---

## 📘 概要

* **機能概要**:
  * 暗号化された TLS/SSL トラフィックを動的に復号し、平文化されたペイロードを Snort インスペクションエンジンや File/AMP エンジンへ回送して、暗号化通信内に潜む攻撃シグネチャ、未知のマルウェア、非許可アプリケーション、データ漏洩（DLP）を検知・遮断する機能。
  * 検査通過後、FTD はパケットを再度暗号化して宛先へ送信する。
* **利用目的**:
  * 暗号化トンネル内に隠蔽された C2 (Command & Control) 通信、悪意のある実行ファイル (.exe) のダウンロード、フィッシング攻撃の可視化と阻止。
  * 企業ポリシーに従い、特定の Web カテゴリ（金融、医療、プライバシー配慮サイト等）は非復号（Do Not Decrypt）として保護しつつ、その他の一般 Web 通信を復号・検査するコンプライアンス維持。
* **どのような場面で利用するか**:
  * 社内端末からインターネット宛ての Web 通信のセキュリティ監視（Outbound SSL Inspection）。
  * 外部から社内 DMZ の公開 Web サーバー群宛ての暗号化アクセスに対する攻撃防御（Inbound SSL Inspection）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要復号方式** | **Decrypt - Resign**（Outbound 向け / 代理 CA 署名）<br>**Decrypt - Known Key**（Inbound 向け / 内部 Web サーバー秘密鍵所有） |
| **その他のアクション** | **Do Not Decrypt**（非復号通過）、**Block**（即時拒否）、**Block with Reset**（TCP RST 送信拒否）、**Monitor**（ログ記録のみ） |
| **証明書オブジェクト** | **Internal CAs**: Resign 用の CA 証明書 ＆ 秘密鍵<br>**Internal Certs**: Known Key 用のサーバー証明書 ＆ 秘密鍵<br>**Trusted CAs**: 外部証明書検証用の CA チェーン / CRL |
| **不可視/復号不能処理** | **Undecryptable Actions**: SSLv2, Handshake Error, Unknown/Unsupported Cipher, Session Compressed 等を個別にブロック/パス制御 |
| **インターフェイス要件** | **In-line Pair** または **Routed / Transparent インターフェイス** でのみサポート（**Passive / TAP モードでは利用不可**） |
| **パフォーマンス影響** | 大幅な CPU リソース消費（TLS 処理）。**Crypto Acceleration Card** や **EVE (Encrypted Visibility Engine)** による負荷軽減が重要 |
| **HA 制限** | Active/Standby Failover 時、復号セッション状態は同期されず切断される（Do Not Decrypt セッションは維持） |

---

## 🏗 動作原理

SSL/TLS Inspection では、トラフィックの方向（Outbound / Inbound）に応じて FTD が異なる役割を果たします。

### 1. Outbound SSL Inspection (Decrypt - Resign)

社内クライアントが外部 Web サーバーへアクセスする際、FTD は「プロキシ（Man-in-the-Middle）」として振る舞います。

```text
[ Client ] ──(1) TLS ClientHello ──► [ FTD (SSL Policy) ] ──(2) TLS ClientHello ──► [ External Server ]
                                             │                                              │
                                             │                                      (3) Server Certificate
                                             │                                              │
                                             │◄─────────────────────────────────────────────┘
                                             │
                                  (4) Validate Server Cert against Trusted CAs
                                  (5) Generate Dynamic Cert using Internal CA (Resign)
                                             │
[ Client ] ◄──(6) Signed Cert (Resigned) ───┤
    │                                        │
    ├─── (7) Establish Session 1 ───────────►│─── (8) Establish Session 2 ─────────► [ External Server ]
    │    (Encrypted with Resigned Key)       │    (Encrypted with Server Key)
    │                                        │
    ▼                                        ▼
[ Send HTTP Data ] ───────────────────► [ Decrypted Stream ] ──► [ Snort / IPS / AMP ] ──► [ Re-encrypt & Forward ]
```

### 2. Inbound SSL Inspection (Decrypt - Known Key)

外部クライアントが社内 DMZ の Web サーバーへアクセスする際、FTD は事前にインポートされた「サーバーの秘密鍵」を用いて通信を直接復号します。

```text
[ External Client ] ──(1) TLS Handshake ──► [ FTD (Known Key) ] ──(2) TLS Handshake ──► [ DMZ Web Server ]
                                                  │ (Holds Private Key)
                                                  ▼
                                     [ Decrypt Payload with Key ]
                                                  │
                                                  ▼
                                    [ Snort / IPS / AVC Inspection ]
                                                  │
                                                  ▼
                                    [ Forward Clean Traffic to Server ]
```

---

## ⚙ 動作シーケンス

1. **パケット着信と L3/L4 評価**:
   * パケットが FTD に着信し、Prefilter Policy および L3/L4 セキュリティインテリジェンス（IP Blacklist）を通過。
2. **TLS Handshake 検出と SSL Policy 照合**:
   * 80/443 等のポートまたはプロトコル識別により TLS ClientHello が認識されると、トラフィックは **SSL Policy** と照合される。
3. **証明書・SNI・Cipher Suite 情報の抽出**:
   * ClientHello / ServerHello から SNI (Server Name Indication)、提案された Cipher Suite、TLS バージョン、サーバー証明書の Subject DN などのメタデータを抽出。
4. **SSL Rule マッチングとアクション決定**:
   * ルールを上から順に評価（First Match Algorithm）：
     * **Do Not Decrypt**: 復号を行わず、アクセス制御ポリシー (ACP) へ暗号状態のまま引き渡す。
     * **Block / Block with Reset**: SSL セッションを即座に破棄（または TCP RST）。
     * **Decrypt - Resign**: サーバー証明書を検証の上、FTD の `Internal CA` で動的に再署名してクライアントへ応答。
     * **Decrypt - Known Key**: 事前登録された `Internal Cert` の秘密鍵を使用してセッションキーを計算・復号。
5. **復号ストリームのディープインスペクション (DPI)**:
   * 復号された平文データストリームが DAQ を経由して Snort エンジンへ渡され、ACP ルール（L7 AVC、URL Category、File Policy/AMP、Intrusion Policy）が適用される。
6. **再暗号化とエグレス送信**:
   * 検査をクリアしたデータは、FTD によって再度暗号化され、宛先へと送信される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Internal CA と Internal Certificate オブジェクトの事前作成**:
   * **Decrypt - Resign (Outbound)** ➔ **`Objects > Object Management > PKI > Internal CAs`** で CA 証明書と秘密鍵を生成/インポート。
   * **Decrypt - Known Key (Inbound)** ➔ **`Objects > Object Management > PKI > Internal Certs`** でサーバー証明書と暗号化されていないペアの秘密鍵（RSA Private Key）をインポート。
   * **共通ミス**: 秘密鍵にパスフレーズがかかったままインポートしようとしてエラーになる（`openssl rsa -in key.pem -out key_nopass.pem` で解除しておく必要あり）。
2. **証明書暗号アルゴリズムのマッチング (Algorithm Matching)**:
   * 外部 サーバーが **Elliptic Curve (ECDSA)** 証明書を使用している場合、FTD 側にも **ECDSA 対応の Internal CA** が用意されている必要があります。RSA 用 CA しかない場合、復号に失敗するか、`Replace Key Only` オプションの調整が必要になります。
3. **Undecryptable Actions（復号不能通信）の個別定義**:
   * `Policies > Access Control > SSL > Undecryptable Actions` タブで以下の設定が問われます：
     * **SSLv2 Session**: 通常 `Block` に設定。
     * **Unknown Cipher Suite / Unsupported Cipher Suite**: 安全のため `Block with reset` または `Do Not Decrypt` に設定。
     * **Handshake Errors / Decryption Errors**: 業務影響回避のため `Do Not Decrypt` か、厳格セキュリティのため `Block` かを要件に従って設定。
4. **Certificate Pinning アプリおよび mTLS (Client Cert) 対策**:
   * クライアント証明書認証を要求するサイトや、アプリ内に証明書がハードコードされているサービス（Dropbox、OneDrive、Zoom、金融アプリ等）は、Decrypt を適用すると通信が破綻します。
   * これらは SSL Policy の上位ルールで **`Do Not Decrypt`** アサイン（Category: `Financial Services`, Application: `Dropbox` 等）を行うことが試験での必須テクニックです。
5. **TLS 1.3 への対応と Server Identity Discovery**:
   * TLS 1.3 では Server Certificate が暗号化されます。SSL 復号を行わずに URL カテゴリ判定等を行うには、ACP の Advanced タブで **`TLS Server Identity Discovery`** を有効化する必要があります。
6. **HA (High Availability) Failover 時の挙動理解**:
   * 復号された SSL セッションの状態は HA ペア間で同期されません（Un-replicated）。Standby 機への Failover 発生時、アクティブだった復号 SSL セッションはリセットされ、クライアントは再接続が必要となります。

---

## 🛠 設定方法

### 1. FMC GUI: Outbound SSL Inspection (Decrypt - Resign) の構築手順

```text
[ Step 1: Internal CA オブジェクトの作成 ]
1. FMC UI から Objects > Object Management > PKI > Internal CAs に移動。
2. "Generate CA" または "Import CA" をクリック:
   - Name: FTD_OUTBOUND_CA
   - Common Name: Secure Firewall Corporate CA
   - Key Type: RSA 2048 (または ECDSA)
3. Generate / Import をクリックして保存。

[ Step 2: SSL Policy の作成とルールの追加 ]
1. Policies > Access Control > SSL を開き "New Policy" をクリック:
   - Name: SSL_INSPECTION_POLICY
   - Default Action: Do Not Decrypt
2. "Add Rule" をクリック (Rule 1: Bypass Sensitive Sites):
   - Name: BYPASS_FINANCE_HEALTH
   - Action: Do Not Decrypt
   - Category タブ: "Financial Services", "Health and Medicine" を選択して追加。
3. "Add Rule" をクリック (Rule 2: Decrypt General Web Traffic):
   - Name: DECRYPT_OUTBOUND_WEB
   - Action: Decrypt -> Decrypt - Resign
   - With: FTD_OUTBOUND_CA を選択
   - Zones タブ: Source = INSIDE_ZONE, Destination = OUTSIDE_ZONE
4. "Undecryptable Actions" タブを開く:
   - SSLv2 Session: Block
   - Decryption Errors: Block
5. "Save" をクリック。

[ Step 3: Access Control Policy へのバインド ]
1. Policies > Access Control を開き、対象の ACP を編集。
2. 画面上部にある "SSL Policy: None" のリンクをクリック。
3. ポップアップ画面で "SSL_INSPECTION_POLICY" を選択し "OK" ➔ "Save"。
4. 変更を FTD アプライアンスへ Deploy。
```

---

### 2. FMC GUI: Inbound SSL Inspection (Decrypt - Known Key) の構築手順

```text
[ Step 1: Internal Cert (サーバー証明書＋秘密鍵) のインポート ]
1. Objects > Object Management > PKI > Internal Certs に移動し "Add Internal Cert" をクリック。
2. Name: DMZ_WEB_SERVER_CERT
3. Certificate Data: サーバー証明書の PEM テキストを貼り付け。
4. Private Key Data: パスフレーズ解除済みの秘密鍵 PEM テキストを貼り付け。
5. "Save" をクリック。

[ Step 2: SSL Policy への Known Key ルールの追加 ]
1. Policies > Access Control > SSL で "SSL_INSPECTION_POLICY" を編集。
2. "Add Rule" を追加:
   - Name: DECRYPT_INBOUND_DMZ_WEB
   - Action: Decrypt -> Decrypt - Known Key
   - With: DMZ_WEB_SERVER_CERT を選択
   - Zones タブ: Source = OUTSIDE_ZONE, Destination = DMZ_ZONE
   - Destination Network: 192.168.100.50 (Web Server IP)
   - Destination Port: 443 (HTTPS)
3. Save をクリックし、ACP 経由で FTD へ Deploy。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI) | 表示確認ポイント |
| :--- | :--- | :--- |
| **SSL Policy の適用状態確認** | <code>show ssl-policy</code> | ポリシー名、ルールのヒットカウント、Default Action |
| **復号アクセラレーション (Crypto) 統計確認** | <code>show dynamic-objects</code> / <code>show counters</code> | SSL 復号エンジンでの処理セッション数 |
| **Snort 3 でのリアルタイム SSL 復号デバッグ** | <code>system support firewall-engine-debug</code> | `SSL Policy Match`, `Decrypt (Resign)`, `Decrypt (Known Key)` の出力 |
| **LINA パケットトレーサーによる SSL 処理フェーズ確認** | <code>packet-tracer input INSIDE tcp 10.1.1.10 50000 203.0.113.10 443</code> | `Phase: SSL-POLICY` ➔ `Action: DECRYPT` の評価ステップ |
| **PKI 証明書・Trustpoint の直接確認** | <code>show crypto ca certificates</code> | FTD 上に展開された Resign CA や Known Key 証明書の情報 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **Decrypt - Resign 有効化後、全クライアントで「Untrusted CA」エラーが発生する** | FTD の Internal CA ルート証明書がクライアント端末の OS / ブラウザの Trust Store に登録されていない | クライアント証明書ストア | FMC から Internal CA の公開鍵 (.crt) をエクスポートし、GPO や MDM を使用して端末の「信頼されたルート証明機関」へ配布する。 |
| **特定アプリ（Zoom、Dropbox、Bank アプリ）が SSL 復号有効化時に接続エラーになる** | アプリ側で **Certificate Pinning**（証明書のハードコード検証）や mTLS が使用されているため | FMC SSL Event Log / FTD CLI Debug | 該当アプリケーションまたは URL を SSL Policy の最上位ルールで **`Do Not Decrypt`** アサインしてバイパスさせる。 |
| **Inbound Decryption 設定時、外部からのアクセスが TCP RST で切断される** | FMC にインポートした秘密鍵にパスフレーズがかかっている、または証明書と秘密鍵の対が一致していない | FTD CLI: <code>debug ssl</code> | OpenSSL で秘密鍵のパスフレーズを削除 (`openssl rsa -in key.pem -out key_nopass.pem`) し、証明書の Modulus と一致するか確認後再インポート。 |
| **SSL 復号を有効にすると FTD の CPU 使用率が 100% に達し遅延が発生する** | 大量の高帯域トラフィック（YouTube、Netflix、大容量ファイル転送）を無差別復号している (Oversubscription) | <code>show cpu</code><br><code>show ssl-policy</code> | SSL Policy で高帯域カテゴリや信頼できるドメインを `Do Not Decrypt` に指定し、復号対象トラフィックを絞り込む。 |
| **FMC で SSL Policy を構成したが FTD 上で全く効果がない** | Access Control Policy (ACP) の画面で SSL Policy がバインドされていない（`SSL Policy: None` のまま） | FMC ACP Editor | ACP の編集画面上部で SSL Policy を選択・アサインしてから Deploy を実行する。 |

---

## ⚠ 制限事項

* **パッシブ / タップモード非対応**:
  * SSL/TLS Inspection（復号処理）は、インライン（Inline Pair）または Routed / Transparent モードのインターフェイスでのみ利用可能です。**Passive モードおよび Inline Tap モードでは動作しません**。
* **HA (High Availability) Failover 時のセッション断**:
  * 復号された SSL/TLS セッションの暗号化ステート（TLS Session Keys）は HA ペア間で同期されません。Failover 発生時、既存の復号済み SSL 接続はすべて断（Reset）となり、クライアントによる再ハンドシェイクが必要です。
* **暗号強度の制限 (Weak Crypto Restrictions)**:
  * FTD / FMC 7.0+ 以降では、セキュリティ基準の厳格化により **RSA 2048 bit 未満の鍵サイズ** および **SHA-1** 署名アルゴリズムの新規オブジェクト作成は拒否されます（既存アプライアンスのダウングレード対応時のみ `Enable Weak-Crypto` フラグで緩和可能）。

---

## 🔄 他技術との関連

* **Access Control Policy (ACP)**: SSL Policy は ACP にバインドされて評価される。SSL 復号によって得られた平文データに対して ACP ルール（L7 AVC、URL Filtering）が適用される。
* **File Policy ＆ AMP**: 暗号化された HTTPS 通信内からファイルを抽出し、マルウェア判定（Block Malware）を行うには SSL Decryption が必須。
* **Intrusion Prevention System (Snort IPS)**: 暗号化通信内のペイロード攻撃シグネチャ（SQL Injection、Buffer Overflow 等）を検知するには SSL Decryption が必須。
* **Encrypted Visibility Engine (EVE)**: FTD 7.1+ 機能。TLS 復号を行わずに、TLS フィンガープリントと機械学習により暗号化通信のままプロセスや脅威を特定する補完機能。

---

## 🧩 比較表

### 1. Decrypt - Resign vs Decrypt - Known Key

| 比較項目 | Decrypt - Resign | Decrypt - Known Key |
| :--- | :--- | :--- |
| **通信方向** | **Outbound** (内部 Client ➔ 外部 Server) | **Inbound** (外部 Client ➔ 内部 Server) |
| **使用する PKI オブジェクト** | **Internal CA** (CA 証明書 ＋ 秘密鍵) | **Internal Cert** (サーバー証明書 ＋ 秘密鍵) |
| **証明書の発行方式** | FTD が外部サーバー証明書を模して**動的に再署名** | 事前に所有する内部サーバーの**固定秘密鍵で復号** |
| **クライアント側の準備** | **Internal CA ルート証明書の信頼登録が必要** | **クライアント側の変更・設定は一切不要** |
| **対応可能な暗号鍵交換** | Diffie-Hellman (DHE/ECDHE), RSA | **RSA 鍵交換のみ** (DHE/ECDHE で PFS を使う場合は Resign 方式必要) |

### 2. FMC SSL Inspection vs WSA HTTPS Proxy

| 比較項目 | Cisco Secure Firewall (FTD / FMC) | Cisco Secure Web Appliance (WSA) |
| :--- | :--- | :--- |
| **製品カテゴリ** | NGFW / NGIPS 統合アプライアンス | 専業 Web Security Gateway (Proxy) |
| **復号アーキテクチャ** | LINA/Snort パイプライン組み込み型 | 専業 HTTP/HTTPS プロキシエンジン (AsyncOS) |
| **対応プロトコル** | HTTPS, SMTPS, POP3S, FTPS, IMAPS 等 | 主に HTTP/HTTPS / FTP over HTTP |
| **主な用途** | ネットワーク境界での多層脅威防御・IPS 連携 | 高度な Web 閲覧制御、DLP、Web レピュテーション |

---

## 💡 ベストプラクティス

1. **Do Not Decrypt ルールの最上位配置**:
   * SSL Policy のルールベースでは、CPU 負荷軽減およびプライバシー保護のため、**`Do Not Decrypt`** アサインのルール（金融・ヘルスケアカテゴリ、Certificate Pinning アプリ等）を **ルールリストの最も上（Top）に配置** します。
2. **証明書チェーン (Chain of Trust) の完全な登録**:
   * `Decrypt - Known Key` を使用する際、サーバー証明書単体だけでなく、中間 CA (Intermediate CA) 証明書を含む完全な証明書チェーンを FMC にインポートしておくことで、外部クライアントでの SSL ハンドシェイクエラーを防止します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Outbound HTTPS Decryption (Decrypt - Resign) の基本構築

#### 問題
社内セグメント (`10.1.1.0/24`) から外部への HTTPS トラフィックを FTD 上で復号インスペクションするため、Internal CA オブジェクト `LAB_CORP_CA` を使用した SSL Policy `SSL_OUTBOUND_POLICY` を構成せよ。

#### 要件
* SSL Policy Default Action: `Do Not Decrypt`
* ACP へバインドして Deploy すること。

#### 設定・操作手順

```text
1. Objects > Object Management > PKI > Internal CAs で "LAB_CORP_CA" を生成。
2. Policies > Access Control > SSL で "SSL_OUTBOUND_POLICY" を作成 (Default Action: Do Not Decrypt)。
3. Add Rule:
   - Name: DECRYPT_LAN_OUTBOUND
   - Action: Decrypt -> Decrypt - Resign
   - With: LAB_CORP_CA
   - Source Network: 10.1.1.0/24
4. Policies > Access Control で ACP を編集し、SSL Policy に "SSL_OUTBOUND_POLICY" をアサイン。
5. Deploy を実行。
```

#### 検証方法
* クライアントから `https://www.cisco.com` へアクセスし、証明書の発行元が `LAB_CORP_CA` に変更されていることを確認。FTD CLI で `show ssl-policy` を実行し、ヒッツカウントが増加していることを確認。

---

### 例2: DMZ Web サーバー保護のための Inbound SSL Decryption (Decrypt - Known Key)

#### 問題
外部から DMZ 内の Web サーバー (`192.168.100.80:443`) への着信通信を復号し、Snort IPS で検査するため、サーバー証明書オブジェクト `DMZ_WEB_CERT` を使用した Known Key ルールを作成せよ。

#### 要件
* アクション: `Decrypt - Known Key`

#### 設定・操作手順

```text
1. Objects > Object Management > PKI > Internal Certs で "DMZ_WEB_CERT" (Cert & Private Key) をインポート。
2. SSL Policy にルールを追加:
   - Name: DECRYPT_INBOUND_DMZ
   - Action: Decrypt -> Decrypt - Known Key
   - With: DMZ_WEB_CERT
   - Destination Network: 192.168.100.80
   - Destination Port: 443
3. Save して Deploy。
```

#### 検証方法
* 外部クライアントから `[suspicious link removed]` へアクセスし、通信が正常に接続され、FTD CLI の `system support firewall-engine-debug` で `Decrypt (Known Key)` がログ出力されることを確認。

---

### 例3: 金融・医療カテゴリサイトの復号除外 (Do Not Decrypt Bypass)

#### 問題
コンプライアンス要件に基づき、URL カテゴリ `Financial Services` および `Health and Medicine` 宛ての HTTPS 通信を復号処理から除外 (`Do Not Decrypt`) せよ。

#### 要件
* 復号ルールより上位に配置すること。

#### 設定・操作手順

```text
1. SSL Policy で "Add Rule" を選択:
   - Name: BYPASS_SENSITIVE_CATEGORIES
   - Action: Do Not Decrypt
   - Category タブ: "Financial Services", "Health and Medicine" を追加。
2. 作成したルールを Decrypt ルールより上 (Order 1) に配置。
3. Deploy を実行。
```

#### 検証方法
* クライアントから銀行サイト (例: `[suspicious link removed]`) にアクセスし、証明書が再署名されずオリジナルのまま通過することを確認。

---

### 例4: 復号不能通信 (Undecryptable Actions) のセキュリティ強化設定

#### 問題
暗号化強度が低く脆弱な `SSLv2` 通信およびハンドシェイクエラー (`Handshake Errors`) が発生した通信を FTD 上で自動ブロックせよ。

#### 要件
* Undecryptable Actions タブで設定すること。

#### 設定・操作手順

```text
1. Policies > Access Control > SSL で対象 SSL Policy を編集。
2. "Undecryptable Actions" タブを選択:
   - SSLv2 Session: Block
   - Handshake Errors: Block with reset
3. Save して Deploy。
```

#### 検証方法
* FTD CLI で `show ssl-policy` を確認し、Undecryptable Actions に Block 設定が反映されていることを検証。

---

### 例5: Certificate Pinning アプリ (Dropbox) の復号バイパスルール設定

#### 問題
SSL 復号導入により接続不能となった Dropbox アプリケーション通信を正常化するため、`Dropbox` アプリを識別して復号から除外せよ。

#### 要件
* Application 条件を使用すること。

#### 設定・操作手順

```text
1. SSL Policy にルールを追加:
   - Name: BYPASS_DROPBOX
   - Action: Do Not Decrypt
   - Applications タブ: "Dropbox" を選択して追加。
2. ルール位置を Decrypt ルールの上位に移動させて Deploy。
```

#### 検証方法
* クライアント上で Dropbox クライアントアプリがエラーなく同期を開始することを確認。

---

### 例6: SSL Decryption と File Policy の連携による暗号化通信内 EXE ファイル遮断

#### 問題
HTTPS 経由での実行可能ファイル (`Executables / .exe`) のダウンロードを検知・遮断せよ。

#### 要件
* SSL Policy で復号後、ACP ルールで File Policy をバインドすること。

#### 設定・操作手順

```text
1. SSL Policy で Outbound 通信の Decrypt - Resign ルールが有効であることを確認。
2. Malware & File Policy で `.exe` を Block する File Policy `BLOCK_EXE_POLICY` を作成。
3. Access Control Policy の Web 許可ルールの Inspection タブで `BLOCK_EXE_POLICY` をバインドして Deploy。
```

#### 検証方法
* クライアントから `[suspicious link removed]` 等から `.exe` ファイルをダウンロード試行し、通信がブロックされ、FMC の `Analysis > Files > File Events` に Decrypted 状態でイベントが記録されることを確認。

---

### 例7: ECDSA (Elliptic Curve) 署名サイトに対応する SSL Resign 設定

#### 問題
ECDSA 暗号アルゴリズムを使用する外部 Web サイト宛ての通信を正常に Decrypt - Resign できるよう、FTD 上に Elliptic Curve 対応の Internal CA を追加設定せよ。

#### 要件
* Key Type: `ECDSA` (P-256)

#### 設定・操作手順

```text
1. Objects > Object Management > PKI > Internal CAs で Generate CA:
   - Name: LAB_EC_CA
   - Key Type: ECDSA / Curve: P-256
2. SSL Policy の Decrypt - Resign ルールにおいて、`Replace Key Only` の挙動を確認し、EC CA をバックアップとしてアサイン。
3. Deploy を実行。
```

#### 検証方法
* ECDSA 証明書を使用している HTTPS サイトへアクセスし、エラーなく復号されて通過することを確認。

---

### 例8: クライアント証明書認証 (mTLS) サイトの自動バイパス設定

#### 問題
クライアント証明書による相互認証 (mTLS) を要求するサイト宛ての通信において、ハンドシェイク失敗を防ぐため、`Client Certificate Requested` 状態を検知して自動バイパスさせよ。

#### 要件
* Certificate Status 条件を使用すること。

#### 設定・操作手順

```text
1. SSL Policy でルールを追加:
   - Name: BYPASS_MUTUAL_TLS
   - Action: Do Not Decrypt
   - Cert Status タブ: [✓] Client Certificate Requested にチェック。
2. ルールを最上位に配置して Deploy。
```

#### 検証方法
* mTLS を要求する Web サーバーへの接続試行時、FTD が復号を回避して直接ハンドシェイクをパススルーさせることを確認。

---

### 例9: TLS 1.3 環境における TLS Server Identity Discovery の有効化

#### 問題
TLS 1.3 通信において、パケットを全復号することなくサーバー証明書情報を取得し、URL カテゴリ制御の精度を向上させよ。

#### 要件
* ACP の Advanced タブで設定すること。

#### 設定・操作手順

```text
1. Policies > Access Control を開き対象 ACP を編集。
2. "Advanced" タブを選択。
3. TLS Server Identity Discovery セクション:
   - [✓] Enable Early application detection and URL categorization にチェック。
4. Save して Deploy。
```

#### 検証方法
* TLS 1.3 対応サイトへのアクセス時、復号なしで FMC の Connection Events に正確な Server Cert SNI / URL カテゴリが記録されることを確認。

---

### 例10: FMC REST API を使用した SSL Policy の一覧取得および検証

#### 問題
FMC REST API を使用して、登録されている SSL Policy のオブジェクト ID および設定詳細を Curl コマンドで取得せよ。

#### 要件
* `/api/fmc_config/v1/domain/{domainUUID}/policy/sslpolicies` エンドポイントを使用すること。

#### 設定・操作手順

```bash
# ===== REST API 経由での SSL Policy 情報取得 =====
curl -k -X GET "[suspicious link removed]" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json"
```

#### 検証方法
* JSON レスポンス内に定義された SSL Policy の名前および UUID が正しく返却されることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で Outbound SSL Decryption Policy (`Decrypt - Resign`) を設定し、FTD へ Deploy 完了した。しかし、社内クライアントから外部 HTTPS サイトへアクセスした際、ブラウザ上に「SEC_ERROR_UNKNOWN_ISSUER」または「Untrusted CA」という警告が表示される。この問題を解消するための根本的な対処法はどれか。

* A) FTD CLI で `configure ssl-policy disable-cert-check` を投入する。
* B) FMC で使用している Internal CA の証明書（パブリックキー）をエクスポートし、グループポリシー (GPO) 等でクライアント端末の「信頼されたルート証明機関 (Trusted Root CAs)」ストアへ配布・登録する。
* C) Access Control Policy の Default Action を Trust に変更する。
* D) FTD を Transparent Firewall Mode から Routed Firewall Mode へ変更する。

**正解**: **B**  
**解説**: Outbound SSL Decryption (Decrypt - Resign) では、FTD が外部 Web サーバーの代わりに自身の Internal CA で署名した偽装証明書をクライアントへ提示します。クライアント OS/ブラウザがこの Internal CA をルート CA として信頼していない場合、Untrusted CA エラーが発生します。CA 証明書を端末の Trust Store へ配布することが必須の対処手順です。

---

### 2. トラブルシュート問題

**問題**: DMZ 内の外部公開 Web サーバー宛ての着信 HTTPS 通信を復号するため、FMC 上で `Decrypt - Known Key` ルールを作成し Deploy した。しかし、外部クライアントからのアクセス時に FTD がパケットを TCP RST で即座に切断してしまう。FTD CLI の `debug ssl` を確認したところ「Private key mismatch or decrypt failure」が出力されていた。原因として最も可能性が高いものはどれか。

* A) FMC にインポートした秘密鍵にパスフレーズが設定されたままである、または証明書と対になる正しい秘密鍵ではない。
* B) 対向の外部クライアントが Active Directory ドメインに参加していないため。
* C) FTD 上で BFD エコー機能が有効化されているため。
* D) FTD が Single Instance モードで動作しているため。

**正解**: **A**  
**解説**: `Decrypt - Known Key` アクションを使用する場合、FMC にインポートする秘密鍵は**パスフレーズが解除された無暗号の PEM 形式**であり、かつ対象のサーバー証明書とペア（Modulus が一致）である必要があります。不一致やパスフレーズ保護が残っている場合、復号処理に失敗して TCP RST が送信されます。

---

### 3. Design 問題

**問題**: FTD において SSL 復号（SSL Inspection）を大規模導入するにあたり、アプライアンスの CPU リソース枯渇による通信遅延を防止し、同時に法的なプライバシー要件（GDPR/HIPAA）を遵守するための最適な SSL Policy 設計ルールはどれか。

* A) 全トラフィックに対して一括で `Decrypt - Resign` を適用し、ACP の Intrusion Policy で除外を行う。
* B) ルールベースの最上位に `Do Not Decrypt` アクションを設定し、`Financial Services` や `Health and Medicine` などの機微カテゴリおよび大容量動画配信サイトを除外指定する。
* C) SSL Policy の Default Action を `Block` に変更する。
* D) インターフェイスを Passive モードに変更して SSL 復号を実行する。

**正解**: **B**  
**解説**: SSL 復号は高い CPU リソースを消費します。パフォーマンスの維持およびプライバシー保護のため、SSL Policy の上位ルールで金融・医療カテゴリや信頼できる高帯域サービスを **`Do Not Decrypt`** にアサインしてバイパスさせることが Cisco 推奨のベストプラクティスです。

---

### 4. 実装問題

**問題**: FTD CLI にログインし、現在動作している SSL Policy の名前、ルールの合致状況（Hit Counts）、および復号セッションの動作ステータスを検証するための正確なコマンドはどれか。

* A) <code>show access-list</code>
* B) <code>show ssl-policy</code>
* C) <code>show running-config firewall</code>
* D) <code>show service-policy</code>

**正解**: **B**  
**解説**: FTD CLI で **`show ssl-policy`** を実行すると、FTD に展開されている SSL Policy の詳細設定、アクティブなルール、およびヒッツカウントを一括で確認できます。

---

### 5. コンフィグ読解問題

**問題**: FMC の SSL Policy 設定画面において、`Undecryptable Actions` タブ内の `SSLv2 Session` オプションを `Block` に設定した。この設定の目的として正しいものはどれか。

* A) SSLv2 を使用する通信を解読して平文でログ出力する。
* B) 脆弱で非推奨な旧式暗号プロトコルである SSLv2 によるセッション接続試行を、復号処理を行う前に即座に拒否・遮断する。
* C) SSLv2 通信に対して新しい証明書を再発行する。
* D) FTD CLI の SSH アクセスを遮断する。

**正解**: **B**  
**解説**: **`Undecryptable Actions`** 設定は、ハンドシェイクエラーやサポート外の暗号方式（SSLv2 等）など、構造的に安全に復号できないトラフィックに対する拒否/通過のハンドリングを定義します。SSLv2 などの脆弱な通信を `Block` に設定することで早期に遮断できます。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Device Configuration Guide - SSL Policies](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot SSL Policies on FTD](https://www.cisco.com/)
* [Cisco Technical Notes: Troubleshoot SSL Inspection Issues on Secure Firewall](https://www.cisco.com/)
* [Cisco Live: Deep Dive into SSL/TLS Inspection with Cisco Secure Firewall (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で「Decrypt outgoing web traffic except for Banking sites and block malware inside HTTPS」と指定された場合、構築手順は以下の通りです：
    1. Internal CA を作成（Objects > PKI > Internal CAs）。
    2. SSL Policy を作成：Rule 1 = `Do Not Decrypt` (Category: Financial Services), Rule 2 = `Decrypt - Resign` (With: Internal CA)。
    3. File Policy を作成：Block Malware (.exe 等)。
    4. ACP で SSL Policy をバインドし、Web 許可ルールの Inspection タブで File Policy をアサインして Deploy。
* **図解（SSL Inspection トラフィック制御の処理フロー）**:
  ```text
  [ Ingress HTTPS Packet ]
             │
             ▼
  [ SSL Policy Evaluation ]
             │
             ├── Match "Do Not Decrypt" Rule ──► [ Pass Encrypted to ACP ]
             ├── Match "Block" Rule ──────────► [ Drop Session / Send TCP RST ]
             └── Match "Decrypt" Rule
                      │
                      ├── Outbound ──► Resign with Internal CA
                      └── Inbound  ──► Decrypt with Server Private Key
                                            │
                                            ▼
                              [ Decrypted Plaintext Stream ]
                                            │
                                            ▼
                              [ Snort IPS / AVC / File Policy / AMP ]
                                            │
                                            ▼
                              [ Re-encrypt & Egress Transmission ]
  ```
* **注意点**:
  * ラボ試験で `Decrypt - Resign` の設定を行った後は、クライアント PC（テスト用 Windows / Linux 端末等）のブラウザで実際に HTTPS サイトを開き、証明書チェーンを表示させて FTD の Internal CA 名が発行元になっていることを必ずテスト・検証してください。
