---
layout: default
title: 2.2-IOS-CA
nav_order: 1
parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.2 Cisco IOS CA for VPN authentication

Cisco IOS に組み込まれている **IOS Certificate Authority (CA) Server**（`crypto pki server` 機能）は、外部のサードパーティ PKI（Microsoft Active Directory Certificate Services や DigiCert 等）を必要とせず、Cisco IOS ルーター自体をスタンドアロンの **Root CA（ルート認証局）** または **Subordinate CA（下位認証局 / Sub-CS）** として動作させるオンボックス PKI 機能です。

CCIE Security v6.1 Blueprint **2.2 Cisco IOS CA for VPN authentication** では、IPsec S2S VPN (IKEv1 / IKEv2)、FlexVPN、DMVPN、GETVPN、および AnyConnect RAVPN (ASA / FTD / Router) において、事前共有鍵 (PSK) に代わるスケーラブルかつ高セキュリティなデジタル証明書認証 (RSA-SIG / ECDSA-SIG) を実装するための IOS CA の設計・構築・自動発行 (Auto-Grant)・証明書失効検証 (CRL / CDP / OCSP)・ロールオーバー (Auto-Rollover) およびトラブルシューティング能力が徹底的に評価されます。

---

## 📘 概要

* **機能概要**:
  * **Cisco IOS Certificate Authority (CA) Server**:
    * Cisco IOS / IOS-XE ルーター上で HTTP サービス (`ip http server`) および SCEP (Simple Certificate Enrollment Protocol) を利用し、VPN ピア（IOS ルーター、ASA、FTD、AnyConnect クライアント等）からの証明書署名要求 (CSR) を自動または手動で承認・発行する暗号化サービス。
    * **Root CA (CS Mode)** および **Subordinate CA (Sub-CS Mode)** の両アーキテクチャをサポート。
  * **PKI Trustpoint (トラストポイント)**:
    * デバイスがどの CA サーバーを信頼し、どのようなパラメータ（鍵ペア、ハッシュアルゴリズム、Subject Name、拡張機能 [EKU, SAN]、登録方法 [SCEP, Terminal, Self]、失効チェック [CRL, OCSP, None]）で証明書要求・検証を行うかを定義するコンテナ。
  * **SCEP (Simple Certificate Enrollment Protocol)**:
    * HTTP/HTTPS 上で PKCS#7 / PKCS#10 構造体を送受信し、ルーターやアプライアンスが CA から自動的に CA 証明書（GetCACert）および自デバイスのアイデンティティ証明書（Enrollment）を取得・更新する標準プロトコル。
* **利用目的**:
  * 大規模な IPsec VPN ネットワーク（DMVPN、GETVPN、FlexVPN）において、多数の Hub/Spoke ピア間の PSK 管理（平文鍵漏洩リスクや個別管理の煩雑さ）を排除し、証明書ベースの IKEv1/IKEv2 相互認証を構成する。
  * クローズドなラボ環境やエッジネットワークにおいて、外部 CA を設置することなくルーター自身で PKI トラストチェーンを自己完結させる。
* **どのような場面で利用するか**:
  * **DMVPN / FlexVPN Hub ノード**: Hub ルーターを IOS CA サーバーとして兼任させ、新規 Spoke ルーターが起動時に SCEP 経由で動的に証明書を取得して全自動で DMVPN 網へ参加するゼロタッチ・プロビジョニング。
  * **GETVPN Key Server ＆ Group Member 認証**: GETVPN の GKE/GM 間で高強度な RSA/ECDSA 署名認証を適用。
  * **Cross-Domain Subordinate CA**: 本社 Root CA (IOS CA) の下に、拠点ルーターを Subordinate CA (Sub-CS) として配置し、証明書発行権限と失効管理を階層分散させる設計。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **評価コンポーネント** | **IOS CA Server** (`crypto pki server`), **PKI Trustpoint** (`crypto pki trustpoint`), **Key Pair** (`crypto key generate rsa/ecdsa`), **SCEP / HTTP** (`ip http server`), **CRL / CDP** |
| **CA 動作モード** | **Root CA** (`mode cs` デフォルト) / **Subordinate CA** (`mode sub-cs`) |
| **承認モード (Granting)** | **`grant auto`** (即時自動発行・ラボ推奨) / **`grant auto rollover`** / **`grant prompt` (手動承認)** |
| **鍵タイプ (Key Pair)** | **RSA** (1024 / 2048 / 4096 bit) / **ECDSA** (P-256 / P-384 / P-521 curve) |
| **プロトコル / ポート** | **SCEP (HTTP TCP 80)**, **HTTPS (TCP 443)**, **CRL / CDP (HTTP 80 / LDAP 389)**, **OCSP (HTTP)** |
| **データベース保存先** | **NVRAM** (`nvram:` デフォルト), **Flash** (`flash:`), **TFTP/FTP/HTTP** |
| **時刻同期前提要件** | **NTP (Network Time Protocol) の完全同期が絶対必須** (有効期間・失効チェック・検証失敗の最大の原因) |
| **拡張機能 (Extensions)** | **Extended Key Usage (EKU)**: `ipsec-end-system`, `ipsec-tunnel`, `ipsec-user`, `server-auth`, `client-auth` |

---

## 🏗 動作原理

### 1. Cisco IOS CA と PKI 登録・VPN 認証の全自動シーケンス (SCEP ＋ IKEv2)

IOS CA サーバー (`R1_CA`) が起動しており、VPN クライアント (`R2_Spoke`) が SCEP 経由で証明書を取得し、IPsec VPN 相手 (`R3_Hub`) と証明書ベースの IKEv2 認証を確立するアーキテクチャ：

```text
[ R2_Spoke (PKI Client) ] ───(1) SCEP GetCACert (HTTP GET) ───► [ R1_CA (IOS CA Server) ]
                                                                       │
[ R2_Spoke ] ◄───(2) Returns CA Certificate & Fingerprint ─────────────┘
      │
      ├── (2b) Administrator verifies CA Fingerprint (`crypto pki authenticate`)
      │
      ├── (3) Generates RSA/ECDSA Key Pair (`crypto key generate rsa label R2_KEY`)
      │
[ R2_Spoke ] ───(4) SCEP PKCS#10 CSR (Enrollment Request) ────────► [ R1_CA ]
                                                                       │
                                                                       ├── (4b) Grants Cert (`grant auto` / `grant 1`)
                                                                       └── Signs Cert with CA Private Key
                                                                       │
[ R2_Spoke ] ◄───(5) Returns PKCS#7 Signed ID Certificate ─────────────┘
      │ (Stores ID Cert & CA Cert in NVRAM/Flash)
      │
      └=================== [ IKEv2 / IPsec VPN Establishment ] ===================
      │
[ R2_Spoke ] ───(6) IKE_SA_INIT (Proposals, Nonce, DH) ─────────► [ R3_Hub (Peer) ]
[ R2_Spoke ] ◄───(7) IKE_SA_INIT Reply ────────────────────────── [ R3_Hub ]
      │
[ R2_Spoke ] ───(8) IKE_AUTH Request ────────────────────────────► [ R3_Hub ]
      │  ├── IDi (DN / FQDN)
      │  ├── CERT (R2's ID Certificate)
      │  └── AUTH (RSA/ECDSA Signature calculated with R2 Private Key)
      │                                                                │
      │                                                                ├── Validate R3:
      │                                                                │   1. Signature check using R2's Public Key from CERT
      │                                                                │   2. Chain check against Trusted CA Cert (R1_CA)
      │                                                                │   3. Validity Date check (Not Before / Not After)
      │                                                                │   4. Revocation check (CRL / OCSP)
      │                                                                │
[ R2_Spoke ] ◄───(9) IKE_AUTH Response (R3's CERT & AUTH) ─────────────┘
      │ (Session Established, IPsec SA Created)
```

---

### 2. Subordinate CA (Sub-CS Mode) の階層信頼モデル

大規模ネットワークにおいて、Root CA (`Root_CA`) の下に Subordinate CA (`Sub_CA`) を配置し、鍵の完全分離とドメイン管理を行う階層構造：

```text
[ Root CA (IOS CA - CS Mode) ]
   │ Issuer: CN=ROOT-CA.CCIE.LAB
   │ Private Key: ROOT_KEY
   │
   ├── (Signs Subordinate CA Certificate)
   │
   ▼
[ Subordinate CA (IOS CA - Sub-CS Mode) ]
   │ Issuer: CN=ROOT-CA.CCIE.LAB, Subject: CN=SUB-CA.CCIE.LAB
   │ Private Key: SUB_KEY
   │
   ├── (Issues End-Entity ID Certificates via SCEP)
   │
   ├──► [ VPN Spoke 1 Certificate (CN=Spoke1.ccie.lab) ]
   └──► [ VPN Spoke 2 Certificate (CN=Spoke2.ccie.lab) ]
```

---

## ⚙ 動作シーケンス

1. **時刻同期 ＆ 準備フェーズ (NTP Synchronization)**:
   * CA サーバーおよび全 VPN ピア間で NTP を完全同期。
   * **重要**: NTP が同期されていない場合、生成された証明書の `Validity Period`（有効期間の開始日時 `Not Before`）が現在時刻より未来になったり、既に有効期限切れ `Not After` と判定され、PKI 登録や VPN 認証が 100% 失敗します。
2. **IOS CA サーバー有効化フェーズ (CA Server Activation)**:
   * CA サーバー上で HTTP サーバー (`ip http server`) を起動。
   * RSA 鍵ペア (`crypto key generate rsa label CA_KEY modulus 2048`) を作成。
   * `crypto pki server <CA_NAME>` を定義し、`issuer-name`、`grant auto`、`lifetime` を構成の上、**`no shutdown`** を実行。
   * CA ルーターは自己署名 Root CA 証明書を自動生成し、`crypto pki trustpoint <CA_NAME>` を自動展開。
3. **クライアント トラストポイント認証フェーズ (`crypto pki authenticate`)**:
   * VPN ピア（クライアント）側で `crypto pki trustpoint <TP_NAME>` を定義し、`enrollment url http://<CA_IP>:80` を指定。
   * `crypto pki authenticate <TP_NAME>` を実行。
   * クライアントは HTTP SCEP で `GetCACert` リクエストを送信し、CA 証明書を受信。
   * 管理者は表示された CA Fingerprint（MD5 / SHA1 ハッシュ）を検証し、`yes` で承認。
4. **クライアント 証明書登録フェーズ (`crypto pki enroll`)**:
   * クライアント側で `crypto pki enroll <TP_NAME>` を実行。
   * ローカルの RSA/ECDSA 秘密鍵ペアを生成し、CSR (Certificate Signing Request) を構成して SCEP ポスト送信。
   * CA サーバー側で `grant auto` が有効な場合、即座に証明書が発行・返却され、クライアントの NVRAM / Flash へ格納される。
5. **VPN 接続 ＆ 証明書検証フェーズ (IKEv1 / IKEv2 Negotiation)**:
   * IKE 認証時、送信側は自らの ID 証明書（`CERT`）と、自らの秘密鍵で暗号化した署名（`AUTH`）を送信。
   * 受信側は受信した `CERT` 内の公開鍵で `AUTH` を復号検証し、さらに `CERT` が自らの Trustpoint に登録された CA によって署名されているか（Trust Chain）、有効期限内か、失効（CRL/OCSP）していないかを連続評価する。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **IOS CA サーバー構築の必須 5 ステップ**:
   * **`ip http server`**: SCEP は HTTP を使用するため、これがないと `enrollment url` への接続が Refused になる。
   * **`crypto key generate rsa label <NAME> modulus 2048`**: CA 用の独立した高強度鍵ペアを作成。
   * **`crypto pki server <CA_NAME>`**: CA サーバーモードへの進入。
   * **`issuer-name CN=<NAME>, O=<ORG>`** ＆ **`grant auto`**: 発行元 DN および自動承認の設定。
   * **`no shutdown`**: 最後に必ず `no shutdown` を実行してパスフレーズを入力し、CA サーバーをアクティベートする。
2. **Subordinate CA (Sub-CS Mode) の構築罠**:
   * 下位 CA ルーターでは `mode sub-cs` を指定する。
   * 自身も上位 Root CA への Trustpoint (`enrollment url http://<ROOT_CA_IP>`) を作成し、`crypto pki authenticate` ＆ `crypto pki enroll` を行って **上位 CA から自らが Subordinate CA 証明書を取得してから `no shutdown`** しなければならない。
3. **失効チェック (Revocation Check) による IKE 接続拒否の回避**:
   * ラボ環境等で CRL サーバーや CDP が到達不能な場合、`revocation-check crl` のままだと IKEv2 認証が失敗します。
   * 要件に「Do not perform revocation checking」または「Ignore CRL failures」とある場合、Trustpoint 内で **`revocation-check none`** または **`revocation-check crl none`** を確実に設定します。
4. **IKEv2 Profile での Trustpoint 明示バインド**:
   * IKEv2 では、複数 CA が存在する場合、IKEv2 Profile 内で **`pki trustpoint <TP_NAME> [sign | verify]`** を明示的にバインドしないと、相手からの `CERTREQ` 送信や証明書検証でデフォルト Trustpoint 検索による意図しない不一致・失敗が発生します。
5. **ASA / FTD での IOS CA 登録手順の違い**:
   * **ASA CLI**: `crypto ca trustpoint <TP_NAME>` ➔ `enrollment url http://<IOS_CA_IP>:80` ➔ `crypto ca authenticate <TP_NAME>` ➔ `crypto ca enroll <TP_NAME>`。
   * **FTD FMC GUI**: `Objects > PKI > Cert Enrollment` ➔ Type: `SCEP`, URL: `http://<IOS_CA_IP>:80` ➔ Device の `Certificates` タブで追加（自動的に SCEP 登録がインボークされる）。

---

## 🛠 設定方法

### 1. Cisco IOS Root CA サーバー (CS Mode) の完全設定

```bash
! ===== Step 1: 時刻同期 (NTP) ＆ HTTP サーバー有効化 =====
clock timezone EST -5
ntp master 2
ip http server

! ===== Step 2: CA サーバー用 RSA 鍵ペア生成 =====
crypto key generate rsa label CA_ROOT_KEY modulus 2048

! ===== Step 3: IOS CA サーバーパラメータ設定 ＆ 有効化 =====
crypto pki server IOS_ROOT_CA
 issuer-name CN=IOS-Root-CA.lab.domain, OU=Security, O=Cisco
 hash sha256
 grant auto                               ! ラボ用自動発行
 lifetime certificate 1095                ! 発行証明書有効期間 (3年)
 lifetime ca-certificate 3650             ! CA証明書有効期間 (10年)
 cdp-url http://<CA_SERVER_IP>/ios_root_ca.crl
 eku server-auth ipsec-end-system ipsec-tunnel
 database level complete
 no shutdown                              ! CA 有効化 (パスフレーズ入力)
```

---

### 2. Cisco IOS Subordinate CA サーバー (Sub-CS Mode) の完全設定

```bash
! ===== Step 1: 上位 Root CA への Trustpoint 定義と登録 =====
crypto key generate rsa label SUB_CA_KEY modulus 2048

crypto pki trustpoint TP_ROOT_CA
 enrollment url http://<ROOT_CA_IP>:80
 subject-name CN=IOS-Sub-CA.lab.domain, OU=Security, O=Cisco
 rsakeypair SUB_CA_KEY
 revocation-check none

! Root CA 証明書の取得と自身の証明書登録
crypto pki authenticate TP_ROOT_CA
crypto pki enroll TP_ROOT_CA

! ===== Step 2: Subordinate CA モードの有効化 =====
crypto pki server IOS_SUB_CA
 mode sub-cs
 issuer-name CN=IOS-Sub-CA.lab.domain, OU=Security, O=Cisco
 hash sha256
 grant auto
 lifetime certificate 365
 lifetime ca-certificate 1095
 no shutdown
```

---

### 3. Cisco IOS VPN Client (Spoke/Peer) での SCEP 登録 ＆ IKEv2 設定

```bash
! ===== Step 1: RSA 鍵ペア ＆ Trustpoint 定義 =====
crypto key generate rsa label SPOKE1_KEY modulus 2048

crypto pki trustpoint TP_IOS_CA
 enrollment url http://<CA_SERVER_IP>:80
 subject-name CN=Spoke1.lab.domain, O=Cisco
 rsakeypair SPOKE1_KEY
 revocation-check none
 fqdn Spoke1.lab.domain

! ===== Step 2: SCEP 認証 ＆ 登録実行 =====
crypto pki authenticate TP_IOS_CA
crypto pki enroll TP_IOS_CA

! ===== Step 3: 証明書を使用した IKEv2 Profile 設定 =====
crypto ikev2 profile IKE2_PROF_CERT
 match identity remote fqdn Hub.lab.domain
 identity local fqdn Spoke1.lab.domain
 authentication local rsa-sig
 authentication remote rsa-sig
 pki trustpoint TP_IOS_CA
```

---

### 4. Cisco ASA での SCEP 証明書取得 ＆ IPsec VPN 設定 (CLI)

```bash
! ===== Step 1: ASA 上での Trustpoint 定義 =====
crypto key generate rsa label ASA_KEY modulus 2048 noconfirm

crypto ca trustpoint TP_ASA_IOSCA
 enrollment url http://<CA_SERVER_IP>:80
 subject-name CN=ASA.lab.domain, O=Cisco
 keypair ASA_KEY
 revocation-check none

! ===== Step 2: Authenticate ＆ Enroll =====
crypto ca authenticate TP_ASA_IOSCA
crypto ca enroll TP_ASA_IOSCA

! ===== Step 3: IKEv2 / IPsec への証明書バインド =====
crypto ikev2 policy 10
 encryption aes-256
 integrity sha256
 group 14

tunnel-group <HUB_IP> type ipsec-l2l
tunnel-group <HUB_IP> ipsec-attributes
 ikev2 local-authentication certificate TP_ASA_IOSCA
 ikev2 remote-authentication certificate
```

---

### 5. Cisco FTD (FMC Managed) での SCEP 証明書オブジェクト割り当て

```text
[ Step 1: FMC GUI での SCEP Cert Enrollment オブジェクト作成 ]
1. Objects > Object Management > PKI > Cert Enrollment に移動。
2. "Add Cert Enrollment" をクリック:
   - Name: SCEP_IOS_CA
   - Enrollment Type: SCEP
   - Enrollment URL: http://<CA_SERVER_IP>:80
   - CA Information: (Get CA Certificate ボタンで取得)
   - Revocation タブ: Enable Revocation Check のチェックを外す (または None)。
3. Save をクリック。

[ Step 2: FTD アプライアンスへの証明書追加 ＆ RAVPN / S2S バインド ]
1. Devices > Device Management ➔ 対象 FTD を編集 ➔ "Certificates" タブ。
2. "Add" ➔ Cert Enrollment: "SCEP_IOS_CA" を選択し "Add" をクリック。
   (※ FMC が自動的に SCEP を介して FTD へ証明書を発行・インストールする)
3. S2S VPN または RAVPN 設定の Authentication で "Certificate" を選択し、インストールした Trustpoint をバインド ➔ Deploy。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (Cisco IOS / ASA) | 表示確認ポイント |
| :--- | :--- | :--- |
| **IOS CA サーバー設定・動作状態確認** | <code>show crypto pki server</code> | `Status: enabled`, `Granting mode`, `CA cert fingerprint`, 発行シリアル番号 |
| **CA データベース内の発行済み証明書一覧** | <code>show crypto pki server <CA_NAME> cert-db</code> | 発行された証明書のシリアル番号、Subject DN、有効期限 |
| **インストール済み CA / ID 証明書の一覧表示** | <code>show crypto pki certificates</code> | `Status: Available`, `Certificate Usage`, `Validity Date`, `Issuer/Subject` |
| **Trustpoint 設定 ＆ 鍵アサイン確認** | <code>show crypto pki trustpoints</code> | `Enrollment URL`, `Key Pair name`, `Revocation check` 設定値 |
| **CRL (証明書失効リスト) キャッシュ確認** | <code>show crypto pki crls</code> | ダウンロードされた CRL の発行元、次回更新時刻 (`NextUpdate`) |
| **RSA 鍵ペア一覧・公開鍵確認** | <code>show crypto key mypubkey rsa</code> | 鍵のラベル名、ビット長 (2048)、Usage (General Purpose) |
| **SCEP / PKI トランザクション・デバッグ** | <code>debug crypto pki transactions</code><br><code>debug crypto pki messages</code> | SCEP HTTP リクエスト (`GetCACert`, `PKIOperation`) と応答コード |
| **証明書検証プロセスの詳細デバッグ** | <code>debug crypto pki validation</code> | チェーン検証、署名チェック、有効期限照合、CRL / OCSP の試行ログ |
| **IKEv2 側での証明書使用デバッグ** | <code>debug crypto ikev2 internal</code> | `CERT` / `CERTREQ` ペイロードの送受信、Trustpoint マッチング判定 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **`crypto pki authenticate` 実行時、`connection refused` や `timeout` で失敗する** | CA サーバー上で **`ip http server`** が起動していない、または中間 FW で TCP 80 がブロックされている | <code>show ip http server status</code><br><code>telnet <CA_IP> 80</code> | CA サーバーで **`ip http server`** を設定し、クライアントからの TCP 80 (HTTP) 通信を許可する。 |
| **`crypto pki server` で `no shutdown` を実行しても `Status: disabled` のまま** | CA サーバー用キーペアが未作成、または管理者パスフレーズの入力が完了していない | <code>show crypto pki server</code> | 先に `crypto key generate rsa label <NAME>` で鍵を生成し、`no shutdown` 実行時にプロンプトに従ってパスフレーズを入力する。 |
| **`crypto pki enroll` 実行後、証明書が受け取れず `Pending` 状態が続く** | CA サーバー側が **`grant prompt` (手動承認)** モードになっており、管理者の手動承認待ち | <code>show crypto pki server</code><br><code>show crypto pki server <CA> requests</code> | CA サーバーで `crypto pki server <CA> grant <REQ_ID>` を実行して手動承認するか、`grant auto` に変更する。 |
| **IKE/IPsec 接続時、`Certificate Validation Failed` や `Certificate Expired` エラーが発生する** | ピア間で **時刻 (NTP) がずれており**、証明書の `Not Before` (未来時刻) または `Not After` (期限切れ) と判定された | <code>show clock</code><br><code>show ntp status</code> | 全デバイスで NTP を設定し、`show clock` で時刻・タイムゾーンが完全に一致していることを確認する。 |
| **`debug crypto ikev2` で `CRL check failed` と出力され IKE 認証がドロップされる** | Trustpoint 内で `revocation-check crl` が設定されているが、CDP/CRL サーバーにアクセスできない | <code>debug crypto pki validation</code> | 要件に従い、Trustpoint 内で **`revocation-check none`** または `revocation-check crl none` を設定する。 |

---

## ⚠ 制限事項

* **IOS CA サーバーの冗長性制限**:
  * Cisco IOS CA サーバーはクラスタリングやリアルタイムの Active/Standby データベース同期をネイティブサポートしていません。高可用性（HA）が必要な場合は、外部の Microsoft AD CS や Dual Subordinate CA 構成を採用する必要があります。
* **暗号化アルゴリズム制限**:
  * 古い IOS バージョンでは SHA-1 / RSA-1024 がデフォルトでしたが、最新の IOS-XE ではセキュリティ上の理由から SHA-256 / RSA-2048 以上（または Suite B ECDSA P-256/P-384）が必須となっています。
* **HTTP サーバー必須要件**:
  * SCEP Protocol は HTTP 上で動作するため、IOS CA を有効化するルーターでは `ip http server` (または `ip http secure-server`) の起動が絶対必須です。

---

## 🔄 他技術との関連

* **IKEv1 / IKEv2 / FlexVPN**: 事前共有鍵 (PSK) の代わりに `authentication local rsa-sig` / `authentication remote rsa-sig` および `pki trustpoint` を使用して相互認証を実施。
* **DMVPN (Dynamic Multipoint VPN)**: 大規模 Spoke 群が SCEP で動的に証明書を取得・自動更新することで、PSK 共有なしで安全なメッシュ型トンネルを確立。
* **GETVPN (Group Encrypted Transport VPN)**: Key Server (KS) と Group Member (GM) 間のグループ鍵配布プロトコル (GKE) において、RSA 署名証明書認証を使用。
* **Cisco ISE (Identity Services Engine)**: ISE 内蔵 CA (Internal Endpoint CA) や SCEP RA Profile を介して AnyConnect / BYOD エンドポイントへ証明書を自動プロビジョニング。

---

## 🧩 比較表

### 1. Root CA (CS Mode) vs Subordinate CA (Sub-CS Mode)

| 比較項目 | Root CA (CS Mode) | Subordinate CA (Sub-CS Mode) |
| :--- | :--- | :--- |
| **信頼の起点** | **自らが最上位の Root Trust Anchor** (自己署名) | **上位 Root CA によって署名** される |
| **必要設定** | `crypto pki server <CA_NAME>` (デフォルトモード) | `crypto pki server <CA_NAME>` ＋ **`mode sub-cs`** |
| **信頼関係** | 自身の CA 証明書を配布 | Root CA 証明書 ＋ 自身の Sub CA 証明書の両方を保持 |
| **用途** | 単一ドメイン・ラボ・完全独立型 PKI | 拠点ごとの管理分散、大規模企業の階層型 PKI |

### 2. 証明書失効検証: CRL (Certificate Revocation List) vs OCSP (Online Certificate Status Protocol)

| 比較項目 | CRL (Certificate Revocation List) | OCSP (Online Certificate Status Protocol) |
| :--- | :--- | :--- |
| **検証方式** | 失効シリアル番号の**リスト一括ダウンロード** | **個別の証明書ステータスをリアルタイム問い合わせ** |
| **プロトコル** | HTTP (`http://`), LDAP (`ldap://`) | HTTP (`http://`) |
| **ルーター負荷** | メモリに CRL ファイル全体をキャッシュ | リアルタイムリクエストメッセージ処理のみ |
| **更新頻度** | スケジュール指定 (例: 24時間ごと) | 即時 (リアルタイム) |

---

## 💡 ベストプラクティス

1. **時刻同期 (NTP) の最優先設定**:
   * PKI および証明書を扱う場合、**最初のステップとして必ず NTP を設定・同期** させます。NTP がずれていると、すべての証明書検証が失敗します。
2. **鍵ペアの非可出化 ＆ 十分なビット長指定**:
   * RSA 鍵生成時は最小 2048 bit を指定し、必要に応じて `exportable` オプションの有無を慎重に判断します（一般的には鍵の盗聴を防ぐため `exportable` は付与しない）。
3. **`revocation-check none` の適切な扱い**:
   * 試験やラボ環境で失効サーバーが準備されていない場合は `revocation-check none` を設定して不要なトラブルを回避します。本番環境では `revocation-check crl none` (CRL が引けない場合は接続を許可するフォールバック) または OCSP を組み合わせます。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Cisco IOS Root CA サーバーの構築と `grant auto` 設定

#### 問題
R1 ルーターにおいて IOS CA サーバー `ROOT_CA` を構築せよ。鍵ペア名 `ROOT_KEY` (2048 bit) を作成し、証明書の自律発行 (`grant auto`)、有効期間 3年（CA証明書は 10年）を設定し、HTTP 経由で SCEP 要求を受信できるようにせよ。

#### 要件
* Issuer Name: `CN=ROOT-CA.lab.domain, O=Cisco`
* Hash: `sha256`
* CA サーバーを有効化 (`no shutdown`) すること。

#### 設定例

```bash
! ===== R1 Configuration =====
clock timezone EST -5
ntp master 2
ip http server

crypto key generate rsa label ROOT_KEY modulus 2048

crypto pki server ROOT_CA
 issuer-name CN=ROOT-CA.lab.domain, O=Cisco
 hash sha256
 grant auto
 lifetime certificate 1095
 lifetime ca-certificate 3650
 database level complete
 no shutdown
```

#### 検証方法
* `show crypto pki server` を実行し、`Status: enabled` および `Granting mode is: auto` であることを確認。

---

### 例2: SCEP を使用した IOS Client (R2) の証明書自動取得

#### 問題
R2 ルーターから R1 (IOS CA: `http://<CA_SERVER_IP>:80`) へ SCEP 経由で証明書を自動申請・取得せよ。

#### 要件
* Trustpoint 名: `TP_ROOT_CA`
* Subject Name: `CN=R2.lab.domain, O=Cisco`
* Revocation Check は無効化 (`none`) すること。

#### 設定例

```bash
! ===== R2 Configuration =====
crypto key generate rsa label R2_KEY modulus 2048

crypto pki trustpoint TP_ROOT_CA
 enrollment url http://<CA_SERVER_IP>:80
 subject-name CN=R2.lab.domain, O=Cisco
 rsakeypair R2_KEY
 revocation-check none

! 認証と登録の実行
crypto pki authenticate TP_ROOT_CA
! (指紋を確認し yes を入力)

crypto pki enroll TP_ROOT_CA
! (プロンプトで yes を入力して自動取得)
```

#### 検証方法
* `show crypto pki certificates` を実行し、`R2.lab.domain` の General Purpose 証明書が `Status: Available` で格納されていることを確認。

---

### 例3: Cisco IOS Subordinate CA (R3) の構築と階層証明書発行

#### 問題
R3 ルーターを R1 (Root CA) に従属する Subordinate CA (`SUB_CA`) として構築せよ。R1 から Subordinate CA 証明書を取得した上で CA サービスを開始せよ。

#### 要件
* Mode: `sub-cs`
* Issuer Name: `CN=SUB-CA.lab.domain, O=Cisco`

#### 設定例

```bash
! ===== R3 Configuration =====
ip http server
crypto key generate rsa label SUB_KEY modulus 2048

! Root CA へのアソシエーション
crypto pki trustpoint TP_UPPER_ROOT
 enrollment url http://<ROOT_CA_IP>:80
 subject-name CN=SUB-CA.lab.domain, O=Cisco
 rsakeypair SUB_KEY
 revocation-check none

crypto pki authenticate TP_UPPER_ROOT
crypto pki enroll TP_UPPER_ROOT

! Subordinate CA サービス起動
crypto pki server SUB_CA
 mode sub-cs
 issuer-name CN=SUB-CA.lab.domain, O=Cisco
 hash sha256
 grant auto
 no shutdown
```

#### 検証方法
* R3 上で `show crypto pki server` を実行し、`Server configured in subordinate server mode` および `Status: enabled` と表示されることを確認。

---

### 例4: ASA から IOS CA への SCEP 登録と IKEv2 バインド

#### 問題
ASA1 において IOS CA (`http://<CA_SERVER_IP>:80`) から SCEP 経由で証明書を取得し、IKEv2 S2S VPN のローカル認証として使用せよ。

#### 要件
* Trustpoint 名: `TP_ASA_CA`
* Revocation Check: `none`

#### 設定例

```bash
! ===== ASA1 Configuration =====
crypto key generate rsa label ASA_KEY modulus 2048 noconfirm

crypto ca trustpoint TP_ASA_CA
 enrollment url http://<CA_SERVER_IP>:80
 subject-name CN=ASA1.lab.domain, O=Cisco
 keypair ASA_KEY
 revocation-check none

crypto ca authenticate TP_ASA_CA
crypto ca enroll TP_ASA_CA

tunnel-group <PEER_IP> type ipsec-l2l
tunnel-group <PEER_IP> ipsec-attributes
 ikev2 local-authentication certificate TP_ASA_CA
 ikev2 remote-authentication certificate
```

#### 検証方法
* `show crypto ca certificates` を実行し、ASA に有効な ID 証明書がアサインされていることを確認。

---

### 例5: IKEv2 S2S VPN での RSA-SIG 証明書相互認証の実装

#### 問題
R2 (Spoke) と R4 (Hub) 間で IKEv2 S2S VPN を構築せよ。認証には IOS CA から発行されたデジタル証明書 (`rsa-sig`) を使用し、FQN マッチングを行え。

#### 要件
* R2 ID: `Spoke1.lab.domain`, R4 ID: `Hub.lab.domain`

#### 設定例

```bash
! ===== R2 (Spoke) Configuration =====
crypto ikev2 profile IKE2_PROFILE_CERT
 match identity remote fqdn Hub.lab.domain
 identity local fqdn Spoke1.lab.domain
 authentication local rsa-sig
 authentication remote rsa-sig
 pki trustpoint TP_IOS_CA

crypto ipsec profile IPSEC_PROF_CERT
 set ikev2-profile IKE2_PROFILE_CERT

interface Tunnel0
 ip address 172.16.100.2 255.255.255.0
 tunnel source GigabitEthernet0/1
 tunnel destination <HUB_IP>
 tunnel protection ipsec profile IPSEC_PROF_CERT
```

#### 検証方法
* `show crypto ikev2 sa` および `show crypto session detail` を実行し、`Authentication method: RSA-SIG` でトンネルが UP していることを確認。

---

### 例6: FlexVPN Remote Access での IOS CA 証明書 ＋ EAP-MSCHAPv2 認証

#### 問題
FlexVPN Server (R4) において、サーバー認証に IOS CA 証明書を使用し、クライアント認証に EAP-MSCHAPv2 (RADIUS/Local) を使用する RAVPN プロファイルを構成せよ。

#### 要件
* IKEv2 Profile 名: `FLEX_RA_PROFILE`
* Trustpoint: `TP_IOS_CA`

#### 設定例

```bash
! ===== R4 (FlexVPN Server) Configuration =====
crypto ikev2 profile FLEX_RA_PROFILE
 match identity remote any
 identity local fqdn FlexServer.lab.domain
 authentication local rsa-sig
 authentication remote eap query-identity
 pki trustpoint TP_IOS_CA
 aaa authentication eap VPNAUTH_LIST
 virtual-template 10
```

#### 検証方法
* AnyConnect クライアントから接続を試み、サーバー証明書が正常検証されて EAP ユーザー認証へ遷移することを確認。

---

### 例7: GETVPN Key Server ＆ Group Member 間での証明書認証構成

#### 問題
GETVPN Key Server (KS) と Group Member (GM) 間で事前共有鍵ではなく IOS CA 発行の証明書を用いた GKE 認証を構成せよ。

#### 要件
* Trustpoint: `TP_IOS_CA`

#### 設定例

```bash
! ===== Key Server (KS) Configuration =====
crypto gdoi group GETVPN_GROUP
 identity number 100
 server local
  rekey authentication my-cert TP_IOS_CA
  sa ipsec 1
   profile GETVPN_IPSEC_PROFILE

! ===== Group Member (GM) Configuration =====
crypto gdoi group GETVPN_GROUP
 identity number 100
 server address ipv4 <KS_IP>
  credentials my-cert TP_IOS_CA
```

#### 検証方法
* `show crypto gdoi` を実行し、KS/GM 間の GKE SA 認証が証明書経由で成功していることを確認。

---

### 例8: SCEP 登録時における Extended Key Usage (EKU) の動的挿入

#### 問題
IOS CA サーバーにおいて、発行するすべての証明書に `server-auth` および `ipsec-tunnel` の Extended Key Usage (EKU) を自動付与するよう設定せよ。

#### 要件
* `eku` コマンドを使用すること。

#### 設定例

```bash
! ===== IOS CA Server Configuration =====
crypto pki server IOS_ROOT_CA
 eku server-auth ipsec-tunnel ipsec-end-system
```

#### 検証方法
* クライアント側で `show crypto pki certificates verbose` を実行し、`X509v3 Extended Key Usage` フィールドに指定した EKU が含まれていることを確認。

---

### 例9: 証明書失効チェック (CRL / CDP) と Fallback コマンドの設定

#### 問題
VPN クライアントにおいて、証明書検証時に CRL を取得し、万が一 CRL サーバーがダウンしている場合は検証エラーを無視して接続を許可 (`none`) とする失効ポリシーを構成せよ。

#### 要件
* `revocation-check crl none` を使用すること。

#### 設定例

```bash
! ===== Client Router Configuration =====
crypto pki trustpoint TP_IOS_CA
 enrollment url http://<CA_SERVER_IP>:80
 revocation-check crl none
```

#### 検証方法
* `show crypto pki trustpoints` を実行し、`Revocation check: crl, fallback: none` と表示されることを確認。

---

### 例10: SCEP 自動ロールオーバー (Auto-Rollover) 設定

#### 問題
IOS CA サーバーにおいて、CA 証明書が失効する 60 日前に自動的に新しい CA 鍵ペアおよびロールオーバー用 CA 証明書を予備作成 (`auto-rollover`) するよう構成せよ。

#### 要件
* `auto-rollover` コマンドを使用すること。

#### 設定例

```bash
! ===== IOS CA Server Configuration =====
crypto pki server IOS_ROOT_CA
 auto-rollover 60
```

#### 検証方法
* `show crypto pki server` を実行し、`Auto-Rollover configured` が有効になっていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 以下の設定を持つ Cisco IOS CA サーバーに対し、クライアントルーターから SCEP による `crypto pki enroll` を実行した。

```bash
crypto pki server MY_LOCAL_CA
 issuer-name CN=MY_LOCAL_CA.lab.domain
 hash sha256
 grant prompt
 no shutdown
```

クライアント側で `crypto pki enroll` を完了させた直後、`show crypto pki certificates` を実行したところ、アイデンティティ証明書が表示されなかった。原因として最も適切なものはどれか。

* A) SCEP は HTTP を使用するため、クライアント側で `ip http server` が起動していない。
* B) CA サーバー側で **`grant prompt`** に設定されているため、管理者が CA サーバー上で **`crypto pki server MY_LOCAL_CA grant <REQ_ID>`** を実行して手動承認するまで証明書が発行されない。
* C) `issuer-name` に FQDN が含まれていないため CA サーバーが要求を破棄した。
* D) CA サーバーのデータベースレベルが未設定のためメモリエラーが発生した。

**正解**: **B**  
**解説**: **`grant prompt`** が指定されている場合、CA サーバーは SCEP 受信時に証明書を即座に自動発行せず、`Pending` 状態として保持します。管理者が CA サーバー上で `show crypto pki server <CA> requests` でリクエスト ID を確認し、`crypto pki server <CA> grant <REQ_ID>` を実行して承認するまで発行されません。自動発行させるには **`grant auto`** を設定します。

---

### 2. トラブルシュート問題

**問題**: 2台のルーター間 (R1, R2) で証明書ベースの IKEv2 IPsec VPN を構成した。両ルーターは IOS CA サーバーから正常に証明書を取得済みである。しかし、IKEv2 ネゴシエーション時に以下のデバッグメッセージが出力され、トンネルの確立に失敗した。

```text
%CRYPTO-4-PKI_CERT_NOT_YET_VALID: Certificate is not valid yet - current time is prior to Not Before date
```

この問題の原因および対処方法として最も適切なものはどれか。

* A) ピア間で暗号化アルゴリズム (AES-256 vs 3DES) が不一致である。
* B) ルーター間で **NTP による時刻同期が取れておらず**、証明書の有効開始日時 (`Not Before`) よりもルーターの現在時刻が過去になっている。両ルーターで NTP を同期させる。
* C) Trustpoint で `revocation-check none` が指定されていない。
* D) 証明書の RSA 鍵長が 1024 bit 未満である。

**正解**: **B**  
**解説**: エラーメッセージ `NOT_YET_VALID` は、ルーターのシステムクロックが証明書の有効期間開始日時 (`Not Before`) より前の時刻を指していることを示します。PKI 環境では NTP による正確な時刻同期が最優先前提条件であり、時刻がずれると証明書検証は 100% 失敗します。

---

### 3. Design 問題

**問題**: 大規模な DMVPN 網を構築するにあたり、Hub ルーターを IOS CA サーバーとして機能させ、数百台の Spoke ルーターが手動操作なしで自動的に証明書を取得してメッシュ接続へ参加するアーキテクチャを設計したい。以下の要件を満たす IOS CA 設定の組み合わせはどれか。

* A) `mode sub-cs` ＋ `grant prompt` ＋ `enrollment terminal`
* B) `ip http server` ＋ **`grant auto`** ＋ SCEP (`enrollment url http://<HUB_IP>:80`)
* C) `no ip http server` ＋ `grant auto` ＋ `enrollment self`
* D) `crypto ca import default` ＋ `revocation-check ocsp`

**正解**: **B**  
**解説**: ゼロタッチで Spoke に証明書を全自動取得・登録させるには、Hub (CA) 上で **`ip http server`** を有効化し、CA 承認を **`grant auto`** に設定した上で、Spoke 側の Trustpoint で SCEP (`enrollment url http://...`) をアサインして自動登録させる構成が必須です。

---

### 4. 実装問題

**問題**: 外部の CRL サーバー（CDP）にアクセスできない隔離された試験環境において、証明書検証時の CRL チェック失敗による IKE 接続阻害を防止するために、Trustpoint 内で記述すべき正しいコマンドはどれか。

* A) `revocation-check crl required`
* B) **`revocation-check none`**
* C) `no crypto pki trustpoint`
* D) `crl query ldap://disable`

**正解**: **B**  
**解説**: 失効確認サーバーが存在しない環境や隔離ラボでは、Trustpoint 設定内で **`revocation-check none`** を指定することで、失効確認手順を完全にスキップし、有効な証明書チェーンのみで認証を成功させることができます。

---

### 5. コンフィグ読解問題

**問題**: 以下の IOS ルーターのコンフィグにおいて、証明書発行手続きを完了させるために管理者が CLI で直接実行すべきコマンドの組み合わせとして正しいものはどれか。

```bash
crypto key generate rsa label MY_KEY modulus 2048

crypto pki trustpoint TP_IOSCA
 enrollment url http://<CA_SERVER_IP>:80
 subject-name CN=RouterA.lab.domain
 rsakeypair MY_KEY
 revocation-check none
```

* A) `crypto pki server TP_IOSCA start`
* B) **`crypto pki authenticate TP_IOSCA`** を実行して CA 証明書を取得・承認した後、**`crypto pki enroll TP_IOSCA`** を実行して自ルーターの証明書を申請・インストールする。
* C) `crypto ikev2 enable` のみ実行する。
* D) `crypto ca import TP_IOSCA pkcs12` を実行する。

**正解**: **B**  
**解説**: SCEP 経由での証明書取得手順は、最初に **`crypto pki authenticate`** で CA ルート証明書を取得・検証し、次に **`crypto pki enroll`** で自デバイスの証明書発行申請 (CSR) を送信して返却された ID 証明書をインストールする 2 段階ステップとなります。

---

## 🔗 参考リソース

* [Cisco IOS Security Configuration Guide: Public Key Infrastructure Configuration Guide](https://www.cisco.com/)
* [Cisco Technical Notes: Configure IOS Certificate Authority (CA) Server for VPN Authentication](https://www.cisco.com/)
* [Cisco IOS Command Reference: crypto pki server](https://www.cisco.com/)
* [Cisco ASA Series Configuration Guide: Configuring Digital Certificates](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Configuration Guide: PKI and Certificate Enrollment Objects](https://www.cisco.com/)
* [Cisco Live: Advanced PKI Design and Deployment in Enterprise VPNs (BRKSEC-3012)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で IOS CA が出題された場合の迅速な構成判断フローチャート：
    * **「Configure Router as CA Server and auto-grant certs」** ➔ `ip http server` ➔ `crypto key generate rsa label <CA_KEY>` ➔ `crypto pki server <NAME>` ➔ `issuer-name` ➔ `grant auto` ➔ `no shutdown`。
    * **「Enroll Client/Spoke Router using SCEP」** ➔ `crypto key generate rsa label <KEY>` ➔ `crypto pki trustpoint <TP>` ➔ `enrollment url http://<CA_IP>:80` ➔ `revocation-check none` ➔ `crypto pki authenticate <TP>` ➔ `crypto pki enroll <TP>`。
    * **「Configure Subordinate CA」** ➔ `crypto pki server <SUB_CA>` ➔ `mode sub-cs` ➔ 上位 CA へ Trustpoint 認証/登録完了後に `no shutdown`。
* **図解（IOS CA 証明書自動承認・発行パイプライン）**:
  ```text
  [ Client Router ]                                       [ IOS CA Server ]
        │                                                        │
        ├── (1) GetCACert Request (HTTP) ───────────────────────►│
        │◄── (2) CA Root Certificate & Fingerprint ──────────────┤
        │                                                        │
  [ User Authenticates CA Fingerprint (`crypto pki authenticate`)]
        │                                                        │
        ├── (3) PKCS#10 CSR (HTTP POST) ────────────────────────►│
        │                                                        ├── Check Granting Mode:
        │                                                        │   ├── `grant auto`: Sign Cert immediately
        │                                                        │   └── `grant prompt`: Wait for Admin CLI `grant <ID>`
        │                                                        │
        │◄── (4) PKCS#7 Signed Identity Certificate ─────────────┘
  ```
* **注意点**:
  * ラボ試験で CA サーバーやクライアントを設定した後は、必ず `show crypto pki server` および `show crypto pki certificates` を実行し、`Status: Available` / `Status: enabled` となっていることを確認してください。

