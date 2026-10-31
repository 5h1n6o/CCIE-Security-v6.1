---
layout: default
title: 1.6-NGFW-features
nav_order: 6
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.6 Cisco NGFW features

Cisco Secure Firewall Threat Defense (FTD) および Management Center (FMC) における **Cisco NGFW Features（次世代ファイアウォール機能群）** は、伝統的な L3/L4 ステートフルファイアウォール機能と、L7 アプリケーション識別・制御 (AVC)、URL Filtering、SSL/TLS 暗号化通信の可視化と復号 (SSL Inspection)、ユーザーアイデンティティ統合 (User Identity)、地域 IP データベースに基づく地理位置制御 (Geolocation)、Reputation ベースの早期遮断 (Security Intelligence)、ファイル制御および AMP (Advanced Malware Protection) 連携、ならびに Snort インストラクションエンジンによる侵入防御 (NGIPS) を単一の統合データプレーン上でシームレスに処理する多層防御アーキテクチャです。

CCIE Security v6.1 Blueprint **1.6 (1.6.a SSL inspection, 1.6.b User identity, 1.6.c Geolocation)** では、これらの各種機能がどのように階層化され、パケット処理パイプライン（LINA Engine ➔ DAQ ➔ Snort 2/3 Engine）の中でどの順番で評価・処理されるかを深く理解し、試験環境において迅速かつ正確に設定・検証・トラブルシューティングを行う能力が厳格に評価されます。

---

## 📘 概要

* **機能概要**:
  * **SSL/TLS Inspection (1.6.a)**: HTTPS や SMTPS、FTPS などの暗号化トラフィックを復号（Decrypt）して Snort IPS や File/AMP エンジンへ引き渡す、または証明書情報（SNI、Subject DN 等）に基づいて非復号（Do Not Decrypt / Passive）のまま制御する機能。
  * **User Identity (1.6.b)**: IP アドレスだけでなく、Active Directory (AD) や Cisco ISE (Identity Services Engine) と連携して「誰が（User/Group）」通信を行っているかを特定し、Access Control Policy でユーザーベースのアクセス制御を実施する機能。
  * **Geolocation (1.6.c)**: Cisco Talos が提供する IP アドレス ➔ 国・地域マッピングデータベースを利用し、特定国からの接続拒否やアクセス制限を動的に適用する機能。
  * **統合セキュリティポリシー処理**: Prefilter Policy、Security Intelligence、SSL Policy、Identity Policy、Access Control Policy (AVC/URL)、File/Malware Policy、Intrusion Policy (IPS) を組み合わせた包括的パケット評価。
* **利用目的**:
  * 暗号化通信内に隠蔽されたマルウェアや攻撃シグネチャの完全な可視化と防御。
  * ユーザーの職責やグループに応じた動的でグラニュラーなアクセス制御（ゼロトラストモデルの適用）。
  * 脅威レベルの高い特定国や地域との不要な通信の早期遮断によるリソース節約。
* **どのような場面で利用するか**:
  * 社内端末からインターネット宛ての Web 通信 (Outbound SSL Decryption & URL Filtering)。
  * 外部から公開 Web サーバー宛てのアクセスに対する暗号解読と保護 (Inbound SSL Decryption)。
  * ISE pxGrid 連携による 802.1X 認証ユーザー情報に基づいた社内セグメント間制御。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **評価パイプライン** | Prefilter ➔ L3/L4 ➔ SI (IP) ➔ SSL Policy ➔ Preprocessors/Identity ➔ SI (DNS/URL) ➔ L7 AVC/URL ➔ File/AMP ➔ IPS (Snort) ➔ QoS |
| **SSL Inspection (1.6.a)** | **Decrypt - Resign** (Outbound 復号), **Decrypt - Known Key** (Inbound 復号), **Do Not Decrypt**, **Block** |
| **User Identity (1.6.b)** | **Passive Identity**: ISE (pxGrid), ISE-PIC, Active Directory Agent (FMC-MDA)<br>**Active Identity**: Captive Portal (Web Authentication) |
| **Geolocation (1.6.c)** | Cisco Talos VDB (Vulnerability Database) パックに含まれる GeoDB を利用。Country / Continent 単位でルール定義 |
| **ライセンス要件** | **Base**: L3/L4 FW, Routing, NAT<br>**Threat**: IPS, SI, Directory/Identity<br>**URL Filtering**: Category/Reputation URL<br>**Malware**: File Control, AMP Cloud |
| **制限事項** | SSL 復号処理は大幅な CPU リソースを消費。特定の金融・医療サイト等はプライバシー保護のため Do Not Decrypt 設定が必要 |

---

## 🏗 動作原理

Cisco FTD の内部アーキテクチャでは、Classic Firewall Engine (LINA) と Next-Generation Security Engine (Snort) が密接に連携してパケットを評価します。

```text
[ Ingress Packet ]
       │
       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Firepower Threat Defense (FTD) Packet Processing Architecture                     │
│                                                                                  │
│  [ LINA Firewall Engine ]                                                        │
│   ├── 1. Ingress Interface & Sanity Checks                                       │
│   ├── 2. Existing Connection Lookup (Fastpath Check)                             │
│   ├── 3. VPN Decryption / UN-NAT                                                 │
│   └── 4. Prefilter Policy Evaluation                                             │
│            ├── Fastpath ─────────────────────────────────────────┐                │
│            ├── Block ──► [ Drop Packet ]                         │                │
│            └── Analyze ──► (Forward to DAQ / Snort)             │                │
│                                 │                                │                │
│  [ Snort Next-Gen Engine ]      ▼                                │                │
│   ├── 5. L2-L4 Decode & Security Intelligence (IP)               │                │
│   ├── 6. SSL / TLS Inspection Policy                             │                │
│   │        ├── Decrypt ──► (Decrypted Stream)                    │                │
│   │        └── Do Not Decrypt ──► (Encrypted Stream)             │                │
│   ├── 7. Network Analysis Policy (NAP) & Preprocessors           │                │
│   ├── 8. Identity Policy (User Mapping Check)                    │                │
│   ├── 9. Security Intelligence (DNS & URL)                       │                │
│   ├── 10. L7 Access Control Rules (AVC, Geolocation, URL)        │                │
│   ├── 11. File Policy & AMP Malware Analysis                     │                │
│   └── 12. Intrusion Prevention Policy (Snort IPS Rules)          │                │
│                                 │                                │                │
│  [ LINA Firewall Engine ]       ▼                                │                │
│   ├── 13. QoS Policy Enforcement ◄───────────────────────────────┘                │
│   ├── 14. Egress Route Lookup & NAT Encapsulation                                │
│   └── 15. VPN Encryption                                                         │
└─────────┬────────────────────────────────────────────────────────────────────────┘
          │
          ▼
[ Egress Packet Transmitted ]
```

---

## ⚙ 動作シーケンス

1. **パケット着信と Prefilter 評価**:
   * LINA エンジンがパケットを受信し、Prefilter Policy を評価。`Fastpath` アクションにマッチしたトラフィックは Snort 検査を完全にバイパスして即座に Egress 処理へ移行。
2. **L3/L4 Firewall ＆ Security Intelligence (IP)**:
   * トラフィックは DAQ を経由して Snort へ転送。IP ベースの Security Intelligence Blacklist と照合され、悪意ある IP 宛て通信は早期ドロップ。
3. **SSL/TLS Inspection**:
   * SSL Policy と照合。Outbound 通信に対しては FTD 内部 CA 証明書で再署名（Decrypt - Resign）、Inbound 通信に対してはサーバー秘密鍵（Decrypt - Known Key）で復号され、平文ストリームが後続エンジンへ引き渡される。
4. **Identity Policy ＆ User Identification**:
   * Passive Identity (ISE pxGrid / AD Agent) または Active Identity (Captive Portal) により、送信元 IP アドレスに対応するユーザー名・セキュリティグループ (SGT) が特定される。
5. **L7 Access Control (AVC, Geolocation, URL Filtering)**:
   * アプリケーション識別 (NBAR2/AVC)、Cisco Talos GeoDB に基づく国識別、および URL Category/Reputation 照合が実行される。
6. **File Policy ＆ AMP Analysis**:
   * 通信内からファイル（Executables, PDFs, Archives 等）が抽出され、Local Malware Analysis や SHA-256 Hash 照合による AMP Cloud ルックアップ（Malware Disposition）が実施される。
7. **Intrusion Prevention System (IPS)**:
   * 復号された平文ペイロードおよびネットワークトラフィックに対して Snort シグネチャが適用され、脆弱性攻撃や過剰パケットが検査される。
8. **Egress 転送および QoS**:
   * 全インスペクションを通過したパケットは LINA エンジンへ戻り、QoS レート制限、NAT 変換、暗号化処理を経て Egress インターフェイスから送出される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **SSL Inspection (1.6.a) の証明書チェーン構築**:
   * **Outbound (Decrypt - Resign)**: FTD に内部 CA（または中間 CA）証明書と秘密鍵をインポートし、クライアント端末がその CA を信頼（Trust Store に登録）している必要があります。
   * **Inbound (Decrypt - Known Key)**: 内部 Web サーバーのサーバー証明書と**暗号化されていない秘密鍵（RSA Private Key）**を PKCS12 / PEM 形式で FMC にインポートする必要があります。
   * **試験の落とし穴**: SSL 復号ルール配下の Access Control Rule では、URL カテゴリや Application 条件が「復号後の平文 HTTP」として正常に認識・マッチすることを確認する。
2. **User Identity (1.6.b) の ISE pxGrid 連携**:
   * FMC と ISE 間で pxGrid (TCP 8900) ピアリングを構築する際、両者のシステム時間が NTP で同期していること、および pxGrid 証明書が相互に信頼されていることが必須要件。
   * **Realm の構成**: FMC の `Integration > Realms` で Active Directory を登録し、ユーザー/グループの定期ダウンロードを正常完了させておく。
3. **Geolocation (1.6.c) の GeoDB 更新およびルール適用**:
   * GeoDB (Geolocation Database) は FMC 上の `System > Updates > Geolocation Updates` でバージョン確認。
   * Access Control Rule の **`Location` タブ** で対象の国（例: `Anonymous Proxy`, `China`, `Russian Federation` 等）を指定する。
4. **Encrypted Visibility Engine (EVE) の活用**:
   * FTD 7.1+ で導入された EVE 機能。TLS 1.3 等で SSL 復号（Decrypt）を行わずに、TLS Client Hello / Server Hello のフィンガープリントおよび機械学習によりクライアントプロセスや脅威を非破壊的に識別する設定。
5. **Prefilter Fastpath と Access Control Trust の決定的な違い**:
   * **Prefilter Fastpath**: **LINA レベルで処理**。Snort へのパケット転送自体を回避するため、CPU 負荷が最も低い（QoS、SI、IPS すべてバイパス）。
   * **Access Control Trust**: **Snort エンジンへ送られた後**に Deep Packet Inspection (DPI) をスキップする。Security Intelligence や QoS 制限は適用される。

---

## 🛠 設定方法

### 1. FMC GUI: SSL Inspection Policy (Decrypt - Resign) の構築

```text
[ Step 1: 内部 CA 証明書のインポート ]
1. Objects > Object Management > PKI > Internal CAs に移動。
2. "Add Internal CA" をクリック:
   - Name: LAB_INTERNAL_CA
   - Certificate & Private Key (PEM 形式) を貼り付け、Save。

[ Step 2: SSL Policy の作成 ]
1. Policies > Access Control > SSL を開き "New Policy" をクリック:
   - Name: OUTBOUND_SSL_POLICY
   - Default Action: Do Not Decrypt
2. "Add Rule" を追加:
   - Name: DECRYPT_WEB_TRAFFIC
   - Action: Decrypt
   - Encryption Algorithm / Decrypt With: Decrypt - Resign
   - CA Certificate: LAB_INTERNAL_CA を選択
   - Network / Category: 必要に応じて特定カテゴリ (例: Finance/Health 以外) を指定
3. Save をクリック。

[ Step 3: Access Control Policy へのバインド ]
1. Policies > Access Control を開き、対象 ACP の編集画面を開く。
2. 画面上部の "SSL Policy: None" をクリックし "OUTBOUND_SSL_POLICY" をアサインして Save ➔ Deploy。
```

---

### 2. FMC GUI: ISE pxGrid 連携による Passive Identity の構成

```text
[ Step 1: FMC への Realm (Active Directory) の追加 ]
1. Integration > Realms を開き "Add Realm" をクリック:
   - Name: LAB_AD_REALM
   - Type: AD
   - AD Host / AD Join Username / Password を入力して接続テスト。
   - User / Group の Download を実行。

[ Step 2: Identity Source (Cisco ISE) の追加 ]
1. Integration > Identity Sources を開き "Identity Services Engine" タブを選択。
2. Primary ISE Host IP, Primary pxGrid Client Name, CA Certificate を設定。
3. "Test" をクリックして pxGrid 接続が SUCCESS となることを確認。

[ Step 3: Identity Policy の作成と ACP へのバインド ]
1. Policies > Access Control > Identity を開き "New Identity Policy" を作成:
   - Rule Name: PASSIVE_ID_RULE
   - Action: Passive Authentication
   - Realm: LAB_AD_REALM
2. ACP の "Identity Policy" フィールドに作成したポリシーをバインドして Deploy。
```

---

### 3. FMC GUI: Geolocation に基づく特定国からの通信ブロック

```text
1. Policies > Access Control を開き、ACP ルールを追加。
2. Rule 編集画面:
   - Name: BLOCK_HIGH_RISK_COUNTRIES
   - Action: Block with reset
   - Location タブを選択:
     - Source Locations リストから "Anonymous Proxy", "China", "Russian Federation" を選択して "Add to Source" をクリック。
3. Save をクリックして Deploy。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI / FMC) | 表示確認ポイント |
| :--- | :--- | :--- |
| **SSL 復号セッションのリアルタイム確認** | <code>show ssl-policy</code> / <code>show dynamic-objects</code> | SSL Policy のマッチ件数および Active 復号セッション数 |
| **ISE pxGrid から同期されたユーザーマッピング確認** | <code>show user-identity errors</code> / <code>show user-identity user</code> | IP アドレス ➔ Domain\User 名のマッピングテーブル表示 |
| **Geolocation データベースのバージョン・動作確認** | <code>show geodb info</code> | GeoDB の適用バージョンおよび国コードエントリー数 |
| **LINA データプレーンでの Access Control ACL 展開状態** | <code>show access-list</code> | Snort へ送るための LINA ACL line と Hit Count |
| **Snort 3 エンジンにおけるリアルタイム評価デバッグ** | <code>system support firewall-engine-debug</code> | アプリケーション名、URL Category、User 名、Geo Location 情報の合致 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **SSL Decryption 有効化後、クライアントブラウザで「Secured Connection Failed / Untrusted CA」エラーが出る** | FTD の Resigning CA 証明書がクライアントの Trust Store に登録されていない | クライアント証明書ストア | FMC で使用している Resigning CA 証明書 (.crt) を GPO や MDM で端末の「信頼されたルート証明機関」へ配布する。 |
| **Inbound SSL Decryption で Web サーバーへの通信が RST される** | FMC にインポートしたサーバー秘密鍵にパスフレーズがかかっている、または暗号方式不一致 | FMC SSL Log / FTD CLI: <code>debug ssl</code> | 秘密鍵からパスフレーズを解除 (`openssl rsa -in key.pem -out key_nopass.pem`) してから FMC に再インポートする。 |
| **Identity Policy を設定したが ACP ルールでユーザー名がマッチしない** | ① ISE pxGrid の承認が PENDING 状態<br>② Realm の User Download 未実施<br>③ NTP 時間不一致 | <code>show user-identity user</code><br>ISE: `Work Centers > pxGrid Services` | ISE 上で FMC からの pxGrid 接続申請を "Approve" し、FMC 上で Realm ダウンロードを完了させる。 |
| **Geolocation ルールを設定したがブロックされない** | GeoDB が未ダウンロード、または送信元 IP がプライベート IP (RFC 1918) であるため国判定不能 | <code>show geodb info</code> | GeoDB を更新。プライベート IP は NAT 後のパブリック IP が FTD の Ingress ポートに着信する場合のみ GeoDB マッチ可能。 |
| **SSL 復号を有効にすると FTD の CPU 使用率が 100% に張り付く** | 全トラフィックを過剰に復号対象としている (Over-subscription) | <code>show cpu</code><br><code>show ssl-policy</code> | 大容量ビデオストリーミングや信頼できる Trusted アプリケーションを SSL Policy で `Do Not Decrypt` に除外設定する。 |

---

## ⚠ 制限事項

* **SSL/TLS 1.3 復号の制限**:
  * TLS 1.3 では Encrypted SNI (ESNI) や Client Hello の一部暗号化により、復号を行わずに URL やドメイン名を高精度に識別することが困難になります（EVE 機能等で補完が必要）。
* **Passive Identity のスケール制限**:
  * ISE pxGrid や AD Agent 経由の IP-to-User マッピング保持数は FTD アプライアンスのメモリサイズ（ハードウェアモデル）に依存します。
* **Geolocation の適用範囲**:
  * GeoDB はパブリック IP アドレスに対してのみ機能します。内部 LAN のプライベート IP (10.0.0.0/8 等) に対して Location 条件を指定してもマッチしません。

---

## 🔄 他技術との関連

* **Cisco ISE ＆ pxGrid**: ユーザーアイデンティティ、SGT (Security Group Tag)、端末プロファイル情報を FMC へリアルタイム転送するコアインフラ。
* **Cisco Talos Intelligence Feeds**: Security Intelligence (IP/URL/DNS) および GeoDB の最新脅威データを自動提供。
* **Encrypted Visibility Engine (EVE)**: TLS 復号によるオーバーヘッドを発生させずに、暗号化トラフィック内のマルウェアやプロセス情報を推論・識別する最新機能。

---

## 🧩 比較表

### 1. SSL Decryption: Outbound (Resign) vs Inbound (Known Key) vs Passive

| 比較項目 | Outbound (Decrypt - Resign) | Inbound (Decrypt - Known Key) | Passive SSL Inspection |
| :--- | :--- | :--- | :--- |
| **保護対象** | 内部クライアント ➔ 外部 Web | 外部クライアント ➔ 内部 Web Server | インラインまたは SPAN トラフィック |
| **必要証明書/鍵** | **FTD 内部 CA 証明書 ＋ 秘密鍵** | **Web サーバーの証明書 ＋ 秘密鍵** | サーバー秘密鍵 (Passive 復号時) |
| **クライアント側の変更** | CA 証明書の信頼登録が必要 | **一切変更不要** | 一切変更不要 |
| **主な用途** | Web 閲覧時のマルウェア/IPS 検査 | 自社 Web サーバーへの攻撃防止 | オフロード型監視・可視化 |

### 2. User Identity: Passive Identity vs Active Identity (Captive Portal)

| 比較項目 | Passive Identity (ISE / AD Agent) | Active Identity (Captive Portal) |
| :--- | :--- | :--- |
| **ユーザー体感** | **完全透明 (サイレント)** | Web ブラウザでのログイン画面表示 |
| **認証方式** | ドメインログオン情報 (Kerberos/NTLM/802.1X) を横取り | HTTP/HTTPS リダイレクトによる Web 認証 |
| **適用端末** | ドメイン参加端末 (Windows / Mac) | 未知の端末、ゲスト端末、BYOD 端末 |
| **設定負荷** | ISE / AD / FMC 間の連携構成が必要 | FTD 上での Captive Portal 設定のみ |

---

## 💡 ベストプラクティス

1. **SSL Decryption のバイパスリスト設計 (Bypass List)**:
   * 個人情報・プライバシー関連カテゴリ（`Financial Services`, `Health and Medicine`）や高帯域ストリーミングサービスは、SSL Policy で明示的に **`Do Not Decrypt`** にアサインし、CPU 負荷軽減と法令順守（HIPAA/GDPR 等）を両立させます。
2. **Identity Policy のフォールバック設計**:
   * Passive Identity（ISE/AD Agent）でユーザー特定に失敗した通信に対して、最終ルールで **`Active Authentication (Captive Portal)`** へフォールバックさせる二段構え構成を採用します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: FMC での Outbound SSL Decryption (Decrypt - Resign) の構成

#### 問題
社内クライアントから外部 HTTPS サイト宛ての通信を復号インスペクションするため、内部 CA 証明書 `SUB_CA_OBJ` を使用した SSL Policy `SSL_OUTBOUND_POLICY` を構成せよ。

#### 要件
* カテゴリ `Financial Services` は復号から除外 (`Do Not Decrypt`) すること。
* それ以外の HTTPS トラフィックを `Decrypt - Resign` とすること。

#### 設定・操作手順

```text
1. Objects > Object Management > PKI > Internal CAs で "SUB_CA_OBJ" (CA Cert & Key) をインポート。
2. Policies > Access Control > SSL で "SSL_OUTBOUND_POLICY" を作成 (Default Action: Do Not Decrypt)。
3. Add Rule 1:
   - Name: BYPASS_FINANCE
   - Category: Financial Services
   - Action: Do Not Decrypt
4. Add Rule 2:
   - Name: DECRYPT_ALL_OTHER
   - Action: Decrypt (Decrypt - Resign)
   - Resign with: SUB_CA_OBJ
5. ACP の SSL Policy に "SSL_OUTBOUND_POLICY" をバインドして Deploy。
```

#### 検証方法
* クライアントから `https://www.cisco.com` へアクセスし、ブラウザの証明書発行者が `SUB_CA_OBJ` に置き換わっていることを確認。

---

### 例2: FMC での Inbound SSL Decryption (Decrypt - Known Key) の構成

#### 問題
DMZ 内の外部公開 Web サーバー (`192.168.100.50`) 宛ての HTTPS トラフィックを FTD 上で復号し、Snort IPS 検査を実行せよ。

#### 要件
* サーバー証明書と秘密鍵オブジェクト: `WEB_SERVER_CERT_KEY`
* SSL Policy: `SSL_INBOUND_POLICY`

#### 設定・操作手順

```text
1. Objects > Object Management > PKI > Internal Certs で "WEB_SERVER_CERT_KEY" (Cert & Private Key) を追加。
2. Policies > Access Control > SSL で "SSL_INBOUND_POLICY" を作成。
3. Add Rule:
   - Name: DECRYPT_INBOUND_WEB
   - Destination Network: 192.168.100.50
   - Destination Port: 443 (HTTPS)
   - Action: Decrypt (Decrypt - Known Key)
   - Server Certificate: WEB_SERVER_CERT_KEY を選択。
4. ACP にバインドして Deploy。
```

#### 検証方法
* 外部クライアントから該当 Web サーバーへアクセスし、FTD CLI で `show ssl-policy` を実行して `Known Key` セッションが正常にヒット・復号されていることを確認。

---

### 例3: Cisco ISE pxGrid 連携による Passive Identity ルールの設定

#### 問題
Cisco ISE から pxGrid 経由で取得したユーザーグループ `CN=Domain Admins` に属するユーザーのみ、重要サーバーゾーン宛ての SSH 通信を許可せよ。

#### 要件
* ACP ルールの Conditions > Users タブでグループを指定すること。

#### 設定・操作手順

```text
1. Integration > Identity Sources > ISE で pxGrid 接続が UP であることを確認。
2. Integration > Realms で AD Realm を同期完了させておく。
3. ACP を編集し、新規ルールを追加:
   - Name: ALLOW_ADMIN_SSH
   - Destination Port: 22 (SSH)
   - Users タブ: Realm = "LAB_AD_REALM" ➔ Users/Groups から "Domain Admins" を指定。
   - Action: Allow
4. Save して Deploy。
```

#### 検証方法
* `Domain Admins` に属するユーザーでログインした端末から SSH 接続を発生させ、FMC の Connection Events に Username が正しく記録され、ルールにヒットして許可されることを確認。

---

### 例4: Active Identity (Captive Portal) の構成

#### 問題
ドメイン未参加のゲスト端末に対して、Web 閲覧時に Captive Portal リダイレクト画面を表示させ、AD ユーザー認証を強制せよ。

#### 要件
* Captive Portal 応答ポート: 8890
* Identity Policy Action: `Active Authentication`

#### 設定・操作手順

```text
1. Devices > Platform Settings で FTD に HTTP/Captive Portal ポート (8890) を設定。
2. Policies > Access Control > Identity で "IDENTITY_POLICY" を作成:
   - Add Rule: Action = Active Authentication, Realm = LAB_AD_REALM.
3. ACP の Identity Policy にアサイン。
4. ACP ルールで認証済みユーザーのみインターネットアクセスを許可するよう構成して Deploy。
```

#### 検証方法
* ゲスト端末から HTTP アクセスを発生させ、Captive Portal ログイン画面へリダイレクトされ、AD ユーザー名/パスワード入力後に通信が許可されることを確認。

---

### 例5: Geolocation に基づく特定地域からのトラフィックブロック

#### 問題
Cisco Talos GeoDB を利用し、`China` および `Russian Federation` から内部 DMZ ネットワーク宛ての全着信通信を拒否せよ。

#### 要件
* ACP ルールのアクション: `Block with reset`

#### 設定・操作手順

```text
1. Policies > Access Control で対象 ACP を編集。
2. Add Rule:
   - Name: BLOCK_GEO_THREATS
   - Action: Block with reset
   - Location タブ: Source Locations に "China" および "Russian Federation" を追加。
   - Destination Networks: DMZ_SUBNET
3. Save して Deploy。
```

#### 検証方法
* 該当国の IP アドレスを模したテストパケット（または Packet Tracer）を送出し、`BLOCK_GEO_THREATS` ルールにヒットして Reset されることを検証。

---

### 例6: Custom Security Intelligence IP Feed の追加と早期遮断

#### 問題
外部 Threat Intelligence サーバー (`[suspicious link removed]`) から 30 分周期で動的ダウンロードされる IP Feed を定義し、Security Intelligence で早期遮断せよ。

#### 要件
* Feed 名: `CUSTOM_MALWARE_IP_FEED`
* 更新間隔: 30 分

#### 設定・操作手順

```text
1. Objects > Object Management > Security Intelligence > Network Lists and Feeds を開く。
2. "Add Network Lists and Feeds" を選択:
   - Name: CUSTOM_MALWARE_IP_FEED
   - Type: Feed
   - Feed URL: [suspicious link removed]
   - Update Frequency: 30 Minutes
3. ACP の "Security Intelligence" タブを開き、"CUSTOM_MALWARE_IP_FEED" を Blacklist 欄へ移動。
4. Deploy を実行。
```

#### 検証方法
* FTD CLI で `show dynamic-objects` または SI リストテーブルを確認し、リスト内の IP 宛てパケットが Snort 検査前に早期ドロップされることを確認。

---

### 例7: Prefilter Policy による Encrypted GRE Tunnel の Fastpath 処理

#### 問題
拠点間 GRE トンネルトラフィック (`GRE Protocol 47`) に対し、Snort インスペクションをバイパスさせて FTD の CPU 負荷を最小化する Prefilter Rule を構成せよ。

#### 要件
* Prefilter Action: `Fastpath`

#### 設定・操作手順

```text
1. Policies > Access Control > Prefilter を開き "Add Prefilter Policy" を作成。
2. Add Tunnel Rule:
   - Name: FASTPATH_GRE
   - Action: Fastpath
   - Protocol: GRE (47)
3. ACP の Advanced タブで作成した Prefilter Policy をアサインして Deploy。
```

#### 検証方法
* FTD CLI で `show access-list` を確認し、Prefilter 用の LINA ACL (`CSM_PF_ACL_`) に Fastpath エントリが生成され、Hit Count が増加することを確認。

---

### 例8: AVC (Application Visibility and Control) による Facebook ゲームの個別ブロック

#### 問題
社員の `Facebook` サイトの閲覧は許可しつつ、Facebook 内のゲームアプリケーション (`Facebook Games`) のみをピンポイントでブロックせよ。

#### 要件
* ACP ルールの Applications タブを使用すること。

#### 設定・操作手順

```text
1. Policies > Access Control を開き、ACP ルールを追加。
2. Add Rule 1 (Block):
   - Name: BLOCK_FB_GAMES
   - Action: Block
   - Applications タブ: "Facebook Games" を検索して追加。
3. Add Rule 2 (Allow):
   - Name: ALLOW_FB_BASE
   - Action: Allow
   - Applications タブ: "Facebook" を追加。
4. Rule 1 が Rule 2 より上位（Top）に配置されていることを確認し、Deploy。
```

#### 検証方法
* クライアントから Facebook ホームページにアクセス（許可）、その後 Facebook ゲームを起動（遮断されること）を確認。

---

### 例9: File Policy による Executable (.exe) ファイルの AMP クラウド自動ブロック

#### 問題
HTTP / HTTPS 経由でダウンロードされるすべての実行可能ファイル (`Executables`) に対して、AMP Cloud Disposition ルックアップを実施し、Malware と判定された場合に自動ブロックせよ。

#### 要件
* Action: `Block Malware`

#### 設定・操作手順

```text
1. Policies > Access Control > Malware & File を開き "New File Policy" を作成.
2. Add Rule:
   - File Types: Category = Executables
   - Application Protocol: Any, Direction: Any
   - Action: Block Malware (Spero Analysis & Dynamic Analysis を必要に応じて有効化)
3. ACP の Web 許可ルールの "Inspection" タブで作成した File Policy をセットして Deploy。
```

#### 検証方法
* テスト用 EICAR ファイルまたは疑似マルウェア .exe をダウンロード試行し、FTD が通信を遮断して FMC の `Analysis > Files > File Events` に Malware イベントが記録されることを確認。

---

### 例10: Encrypted Visibility Engine (EVE) の有効化と TLS 1.3 通信の識別

#### 問題
FTD 7.1+ 環境において、SSL 復号を行わずに TLS 1.3 暗号化通信内のクライアントプロセスおよび脅威を識別する Encrypted Visibility Engine (EVE) を有効化せよ。

#### 要件
* ACP の Advanced タブで構成すること。

#### 設定・操作手順

```text
1. Policies > Access Control を開き対象 ACP を編集。
2. "Advanced" タブを選択。
3. Encrypted Visibility Engine セクション:
   - [✓] Enable Encrypted Visibility Engine にチェック。
4. Save をクリックして Deploy。
```

#### 検証方法
* FMC の Connection Events 画面を開き、暗号化通信ログの `Client Process` および `EVE Process Confidence` フィールドにプロセス情報が可視化されていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で Outbound SSL Decryption Policy を構築し、Resigning CA 証明書をアサインして Deploy 完了した。しかし、クライアント端末から外部 HTTPS サイトへアクセスすると、ブラウザ上に「証明書エラー（Untrusted Authority）」が継続して発生する。この問題を根本解決するための正しい対処手順はどれか。

* A) FTD CLI で `configure ssl-policy disable-cert-check` を実行する。
* B) FMC で使用している Resigning CA のルート証明書（Public Key）を、全クライアント端末の OS / ブラウザの「信頼されたルート証明機関 (Trusted Root CAs)」ストアへ配布・登録する。
* C) Access Control Policy の Default Action を Trust に変更する。
* D) FTD の Firewall Mode を Transparent Mode から Routed Mode へ変更する。

**正解**: **B**  
**解説**: Outbound SSL Decryption (Decrypt - Resign) では、FTD が外部 Web サーバーの証明書を透過的に差し替え、自身の Resigning CA で動的に再署名してクライアントへ渡します。クライアント端末がこの Resigning CA を信頼していない場合、Untrusted CA エラーが発生します。CA ルート証明書を端末の Trust Store へ配布することが必須の対処です。

---

### 2. トラブルシュート問題

**問題**: FMC と Cisco ISE 間で pxGrid による Identity 連携を構成した。ISE 側で 802.1X 認証が成功しユーザーがログインしているにもかかわらず、FMC の Access Control Policy でユーザー名条件を指定したルールに通信がヒットせず、デフォルトルールまで通過してしまう。FTD CLI で `show user-identity user` を実行したところ、該当 IP アドレスのマッピング情報が表示されなかった。原因として最も可能性が高いものはどれか。

* A) FMC 上で Active Directory Realm のユーザー/グループの初期ダウンロード（User Download）が未実施である。
* B) FTD 上で BFD エコーが有効化されているため。
* C) FTD が Active/Standby HA 構成で動作しているため。
* D) Prefilter Policy で Analyze アクションが指定されているため。

**正解**: **A**  
**解説**: ISE から pxGrid 経由で IP-to-User マッピング通知を受信しても、FMC 上で **Active Directory Realm** が設定され、ユーザーおよびグループ構造が FMC データベースへダウンロードされていない場合、FMC/FTD は受信したユーザー情報を正しい Identity オブジェクトとして認識・ルールバインドできません。

---

### 3. Design 問題

**問題**: 外部からの DDoS 攻撃や不正スキャンから内部ネットワークを防護するため、特定の大手アノニマスプロキシグループや高リスク国からの通信を最低限の CPU 負荷でブロックしたい。FMC 上で採用すべき最も効果的な設計手法はどれか。

* A) Access Control Rule で Destination IP 条件に全プライベート IP を指定して Block する。
* B) Access Control Rule の Location タブで対象の国（Geolocation）および Anonymous Proxy を指定し、早期のルール位置に Block with Reset アクションで配置する。
* C) Prefilter Policy で全ポートを Fastpath に設定する。
* D) SSL Policy で Decrypt - Known Key を設定する。

**正解**: **B**  
**解説**: **Geolocation** 機能を Access Control Policy の上位ルールに配置し、`Location` 条件で対象国や `Anonymous Proxy` を指定して Block/Reset に設定することで、不要な海外トラフィックやプロキシトラフィックを効果的に遮断できます。

---

### 4. 実装問題

**問題**: FTD において、特定の安全な内部サーバー間通信（10.1.1.0/24 ↔ 10.2.2.0/24）について、Snort インスペクションエンジンへのパケット転送自体を完全に回避し、最大の通信スループットと最小遅延を実現するための設定箇所はどれか。

* A) Access Control Rule の Action を `Allow` に設定する。
* B) Access Control Rule の Action を `Trust` に設定する。
* C) Prefilter Policy で該当の送信元・宛先 IP 条件を指定し、Action を `Fastpath` に設定する。
* D) Identity Policy で `No Authentication` を設定する。

**正解**: **C**  
**解説**: **Prefilter Policy** の **`Fastpath`** アクションは、LINA ファイアウォールエンジンレベルで通信をバイパス処理し、Snort エンジンへのパケット転送自体を行わないため、最も高いスループットと最小の遅延を実現します（ACP Trust は Snort に渡された後に DPI をスキップする処理です）。

---

### 5. コンフィグ読解問題

**問題**: FMC の Access Control Rule において、以下の条件が設定されている。このルールの動作に関する記述として正しいものはどれか。

* **Action**: `Block Malware`
* **File Types**: `Category: Executables`
* **Inspection**: `File Policy: AMP_BLOCK_POLICY`

* A) 該当通信を通過させず、L3 レベルで即座に RST パケットを送信する。
* B) HTTP/HTTPS 等の通信内から抽出された Executable ファイルの SHA-256 ハッシュ値を AMP Cloud と照合し、Malware ディスポジションの場合にファイルダウンロードを遮断する。
* C) 全ての ZIP ファイルを強制的に解凍して暗号解除する。
* D) 端末上にインストールされている Cisco AMP for Endpoints エージェントをアンインストールする。

**正解**: **B**  
**解説**: File Policy における **`Block Malware`** アクションは、ネットワーク通信を通過する対象ファイルタイプ（Executables）をリアルタイムに抽出し、ハッシュ値を Cisco AMP Cloud と照合して、悪意あるファイル（Malware）と判定された場合に該当ファイルの転送をブロックします。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Device Configuration Guide](https://www.cisco.com/)
* [Cisco Secure Firewall Threat Defense Configuration Guide](https://www.cisco.com/)
* [CCNP Security Cisco Secure Firewall and Intrusion Prevention System Official Cert Guide](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot SSL Policies on FTD](https://www.cisco.com/)
* [Cisco Live: Advanced Threat Defense with Cisco Secure Firewall (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で「Block malware without decrypting TLS 1.3 traffic」と指示された場合は、`SSL Decryption Policy` ではなく **`Encrypted Visibility Engine (EVE)`** を ACP の Advanced 設定で有効化します。
* **図解（NGFW ポリシー評価フローの簡略決定ツリー）**:
  ```text
  [ Ingress Traffic ]
          │
          ├── Prefilter Rule Match (Fastpath)? ──► YES ──► [ Pass without Snort DPI ]
          └── NO (Analyze)
                │
                ├── Security Intelligence IP Match (Blacklist)? ──► YES ──► [ Early Drop ]
                └── NO
                      │
                      ├── SSL Policy Match? ──► [ Decrypt / Do Not Decrypt ]
                      │
                      ├── Identity Policy ──► [ Identify User / SGT ]
                      │
                      ├── ACP Rule Match (AVC / URL / Geolocation)?
                      │       ├── Block ──► [ Drop / Reset ]
                      │       └── Allow ──► [ Pass to File/AMP & Snort IPS Inspection ]
                      │
                      └── Egress Routing / QoS / NAT ──► [ Egress Traffic ]
  ```
* **注意点**:
  * ラボ試験で SSL 復号や File Policy、IPS ルールを作成した後は、必ず対象 FTD への **Policy Deploy** を完了させてください。Deploy 漏れによる設定不反映は試験での典型的な減点対象となります。
