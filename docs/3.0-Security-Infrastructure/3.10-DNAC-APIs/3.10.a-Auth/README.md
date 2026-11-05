---
layout: default
title: 3.10.a-Auth
nav_order: 1
parent: 3.10-DNAC-APIs
grand_parent: 3.0-Security-Infrastructure
---

# 3.10.a Authentication and authorization

## 📘 概要

Cisco DNA Center（現 Cisco Catalyst Center）の Northbound REST API（Intent API）を利用してネットワークインフラの自動監視・制御を行う場合、すべての API 操作の起点となるのが「認証（Authentication）」と「認可（Authorization）」です。Cisco DNA Center は、セキュリティ担保および操作の非 repudiation（否認防止）のために厳格な Token ベースの認証システムを採用しています。

本ドキュメントでは、CCIE Security v6.1 ラボ試験において確実に得点するために必要な、Cisco DNA Center の API 認証メカニズム（HTTP Basic Authentication による `x-auth-token` 取得フロー）、トークンの有効期限管理（1時間ルールと無効化ポリシー）、ロジカルな RBAC（Role-Based Access Control）によるアクセス制御、外部 AAA（ISE / RADIUS / TACACS+ / LDAP）との連携、および Python `requests` ライブラリを用いた堅牢な API スクリプトの実装手法について深く解説します。


---

## 🔑 要点

Cisco DNA Center Northbound API の認証および認可における核心的な要点を下表にまとめます。

| 項目 | 内容 |
| --- | --- |
| **認証方式** | HTTP Basic Authentication (`Authorization: Basic <base64_encode(user:pass)>`) を用いた初期トークン発行 |
| **認可ヘッダー** | トークン発行後のすべての API リクエストで使用するカスタムヘッダー `x-auth-token: <Token_Value>` |
| **トークン有効期限** | **1時間（60分）**。有効期限が切れると `HTTP 401 Unauthorized` エラーが発生し、トークンの再発行が必要 |
| **認証エンドポイント** | `POST /dna/system/api/v1/auth/token` |
| **ユーザーロール (RBAC)** | SUPER-ADMIN-ROLE, NETWORK-ADMIN-ROLE, OBSERVER-ROLE など。ロールに応じた API リソース制限 |
| **外部 AAA 連携** | Cisco ISE / RADIUS / TACACS+ / LDAP と統合可能。外部ユーザーグループを DNAC 内部ロールへマッピング |
| **SSL/TLS 要件** | HTTPS（TLS 1.2 / 1.3、デフォルト Port 443）。自己署名証明書使用時は Python で `verify=False` 設定が必要 |
| **エラー処理要件** | `401 Unauthorized`（トークン切れ/認証失敗）、`403 Forbidden`（権限不足）、`409 Conflict` の適切な捕捉 |


---

## 🏗 動作原理

Cisco DNA Center の REST API 認証・認可は、セッションレスなステートレス認証ではなく、**時限式アクセストークン（Ephemeral Access Token）**を用いたセキュアなモデルです。

### 認証・認可アーキテクチャフロー

```
[ Python Client / API Client ]             [ Cisco DNA Center (AAA Engine / API Gateway) ]             [ External AAA (Cisco ISE / RADIUS) ]
              │                                                │                                                         │
              │  1. POST /dna/system/api/v1/auth/token         │                                                         │
              │     Header: Authorization: Basic <base64>      │                                                         │
              ├───────────────────────────────────────────────►│                                                         │
              │                                                │  2. Authenticate User Credential                        │
              │                                                ├────────────────────────────────────────────────────────►│
              │                                                │  3. Access-Accept + User Role / Group                   │
              │                                                │◄────────────────────────────────────────────────────────┤
              │                                                │                                                         │
              │                                                │  4. Generate Token (Valid for 60 mins) & Assign RBAC    │
              │  5. HTTP 200 OK                                │                                                         │
              │     Body: {"Token": "eyJhbGciOi..."}           │                                                         │
              │◄───────────────────────────────────────────────┤                                                         │
              │                                                │                                                         │
              │  6. GET /dna/intent/api/v1/network-device      │                                                         │
              │     Header: x-auth-token: eyJhbGciOi...        │                                                         │
              ├───────────────────────────────────────────────►│                                                         │
              │                                                │  7. Validate Token & Check RBAC Privileges              │
              │  8. HTTP 200 OK + Device Inventory Data        │                                                         │
              │◄───────────────────────────────────────────────┤                                                         │
```

1. **初期認証（Authentication）**: クライアントは `username:password` を Base64 エンコードし、`Authorization: Basic <credentials>` ヘッダーを添えて `POST /dna/system/api/v1/auth/token` へリクエストを送ります。
2. **トークン生成**: 認証が成功すると、DNAC は一意の英数字トークン（通常 JWT 形式またはランダム文字列）を発行し、内部のセッションキャッシュに 60 分間の寿命を設定します。
3. **API リクエスト実行（Authorization）**: クライアントは取得したトークンを `x-auth-token` ヘッダーに設定して目的の Intent API（例: ネットワークデバイス一覧取得、CLI コマンド実行、トポロジー取得）を呼び出します。
4. **アクセス制御（RBAC 検証）**: API ゲートウェイはヘッダー内の `x-auth-token` を検証し、トークンに紐づくユーザーロールが指定された API エンドポイントの実行権限（例: Read/Write）を持っているかを確認します。


---

## ⚙ 動作シーケンス

パケットおよびアプリケーションレベルにおける詳細な処理シーケンスは以下の通りです。

```
[クライアント (Python Script)]                                   [Cisco DNA Center API Gateway]

    │                                                                        │
    │ ─── (Step 1) HTTP POST /dna/system/api/v1/auth/token ────────────────► │
    │     Headers:                                                           │
    │       Content-Type: application/json                                   │
    │       Authorization: Basic YWRtaW46TWFnazEkMjM0                        │
    │                                                                        │
    │ ◄── (Step 2) HTTP 200 OK ───────────────────────────────────────────── │
    │     Headers:                                                           │
    │       Set-Cookie: ...                                                  │
    │     Body:                                                              │
    │       { "Token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." }            │
    │                                                                        │
    │ ─── (Step 3) HTTP GET /dna/intent/api/v1/network-device ──────────────► │
    │     Headers:                                                           │
    │       x-auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...            │
    │       Content-Type: application/json                                   │
    │                                                                        │
    │                                                                        │ ─── (Step 4) トークン妥当性 & 有効期限検証
    │                                                                        │     - トークン存在チェック
    │                                                                        │     - 経過時間 < 3600秒
    │                                                                        │     - ロール権限チェック (NETWORK-ADMIN)
    │                                                                        │
    │ ◄── (Step 5) HTTP 200 OK ───────────────────────────────────────────── │
    │     Body:                                                              │
    │       { "response": [ { "hostname": "Core-SW1", ... } ] }              │
    │                                                                        │
    │   ... 60分経過後 ...                                                   │
    │                                                                        │
    │ ─── (Step 6) HTTP GET /dna/intent/api/v1/network-device ──────────────► │
    │     Headers:                                                           │
    │       x-auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9... (Expired)  │
    │                                                                        │
    │ ◄── (Step 7) HTTP 401 Unauthorized ─────────────────────────────────── │
    │     Body:                                                              │
    │       { "error": "Token expired or invalid" }                          │
    │                                                                        │
    │ ─── (Step 8) トークン再取得処理 (Step 1 へ戻る) ────────────────────────► │
```


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、Cisco DNA Center の Northbound API 認証・認可に関する問題が出題された場合、単に「トークンを取得して終わり」という単純なタスクではなく、**「例外処理を含む実用的な自動化スクリプトの記述」**や**「特定ロールでのアクセス制限の検証」**が問われます。

### Blueprintで重要なポイント

1. **正確な認証ヘッダーの選択**:
   * 初期トークン取得時: `Authorization: Basic <base64_credentials>` または Python `requests` の `auth=('user', 'pass')` を使用。
   * API リクエスト時: `x-auth-token: <Token>` ヘッダーを使用。`Authorization: Bearer <Token>` ではなく **`x-auth-token`** である点に注意（他製品の REST API との混同を狙うトラップ）。
2. **トークン期限切れ（HTTP 401）への堅牢な対応**:
   * 長時間実行スクリプトやループ処理内でトークンが失効した場合に、自動的にトークンを再発行してリクエストをリトライするロジックの実装。
3. **権限エラー（HTTP 403）の認識**:
   * 読み取り専用アカウント（OBSERVER-ROLE など）で設定変更 API（POST/PUT/DELETE）を呼び出した場合に発生する 403 エラーのハンドリング。
4. **SSL/TLS 検証エラーの回避**:
   * ラボ環境では自己署名証明書（Self-Signed Certificate）が使用されているため、Python の `requests.get(..., verify=False)` 設定および `urllib3.disable_warnings()` による警告抑制が必須。

---

### ラボ試験で想定される設定・実装タスク

* **タスク A**: Cisco DNA Center からアクセストークンを取得し、有効なトークンをヘッダーにセットして全デバイスの IP アドレスおよびホスト名の一覧をテキストファイルに保存するスクリプトを作成せよ。
* **タスク B**: 取得したトークンを用いて特定デバイスへ Command Runner API (`POST /dna/intent/api/v1/network-device-poller/cli/read-request`) を送信し、非同期タスク ID (`taskId`) を追跡して結果を出力するスクリプトを作成せよ。
* **タスク C**: トークンの有効期限切れ（HTTP 401）を検出した際、自動的に再認証を行って要求を再試行する Python クラスを実装せよ。

---

### よくある設定・実装ミス

* **ヘッダー名の打ち間違い**: `X-Auth-Token` や `Authorization` などの大文字・小文字、ハイフンの表記ミス（Python の `requests` はヘッダー名をケースインセンシティブに扱いますが、明確に `x-auth-token` と記載するのが安全）。
* **JSON パースミス**: 認証レスポンスの辞書キーは大文字始まりの **`Token`** です (`response.json()['Token']`)。小文字の `token` と指定すると `KeyError` が発生します。
* **SSL 証明書エラーによるスクリプト中断**: `verify=False` を指定し忘れて `SSLError` / `CERTIFICATE_VERIFY_FAILED` でスクリプトが異常終了する。
* **Content-Type ヘッダーの欠落**: `POST` / `PUT` リクエスト時に `Content-Type: application/json` を付与し忘れ、DNAC 側で `415 Unsupported Media Type` または `400 Bad Request` となる。


---

## 🛠 設定方法

### 1. Python による基本認証・トークン取得テンプレート

```python
import requests
import json
import urllib3
import os

# SSL警告の無効化（ラボ環境の自己署名証明書対策）
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# DNAC接続情報
DNAC_HOST = "dnac.example.com"
API_USER = "api_user"
API_PASS = os.getenv("DNAC_PASSWORD", "secret_value")

def get_auth_token(host, username, password):
    url = f"https://{host}/dna/system/api/v1/auth/token"
    headers = {
        "Content-Type": "application/json",
        "Accept": "application/json"
    }
    
    try:
        response = requests.post(
            url,
            auth=(username, password),
            headers=headers,
            verify=False,
            timeout=10
        )
        response.raise_for_status()
        
        token = response.json().get("Token")
        print(f"[SUCCESS] Token successfully obtained: {token[:15]}...")
        return token
        
    except requests.exceptions.HTTPError as err:
        print(f"[ERROR] HTTP Error: {err} - Status Code: {response.status_code}")
        print(f"[DETAILS] {response.text}")
        return None
    except Exception as e:
        print(f"[ERROR] Connection failed: {e}")
        return None

if __name__ == "__main__":
    token = get_auth_token(DNAC_HOST, API_USER, API_PASS)
```

---

### 2. トークン自動管理機能付き API クライアントクラス

トークンの有効期限切れ（401 Unauthorized）を自律検知し、再認証を実行してリクエストを自動リトライする高度な実装例です。

```python
import requests
import urllib3
import time
import os

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

class DNACClient:
    def __init__(self, host, username, password):
        self.base_url = f"https://{host}"
        self.username = username
        self.password = password
        self.token = None
        self.token_obtained_at = 0
        self.session = requests.Session()
        self.session.verify = False
        self.authenticate()

    def authenticate(self):
        auth_url = f"{self.base_url}/dna/system/api/v1/auth/token"
        headers = {"Content-Type": "application/json"}
        
        response = self.session.post(
            auth_url,
            auth=(self.username, self.password),
            headers=headers
        )
        if response.status_code == 200:
            self.token = response.json()["Token"]
            self.token_obtained_at = time.time()
            self.session.headers.update({"x-auth-token": self.token})
            print("[INFO] Authentication successful. Token updated.")
        else:
            raise Exception(f"Authentication failed: {response.status_code}")

    def request(self, method, endpoint, **kwargs):
        url = f"{self.base_url}{endpoint}"
        
        if time.time() - self.token_obtained_at > 3300:
            print("[INFO] Token is older than 55 minutes. Re-authenticating...")
            self.authenticate()

        response = self.session.request(method, url, **kwargs)

        if response.status_code == 401:
            print("[WARNING] Received 401 Unauthorized. Attempting to re-authenticate...")
            self.authenticate()
            response = self.session.request(method, url, **kwargs)

        return response

if __name__ == "__main__":
    client = DNACClient("dnac.example.com", "api_user", os.getenv("DNAC_PASSWORD", "secret_value"))
    res = client.request("GET", "/dna/intent/api/v1/network-device")
    if res.status_code == 200:
        devices = res.json()["response"]
        print(f"Total Devices: {len(devices)}")
```

---

### 3. cURL コマンドによる動作確認手法

```bash
# 1. トークンの取得
curl -s -k -X POST "https://dnac.example.com/dna/system/api/v1/auth/token" \
     -H "Content-Type: application/json" \
     -u "api_user:secret_value"

# 出力例: {"Token":"token_string"}

# 2. 変数へのトークン保存
TOKEN=$(curl -s -k -X POST "https://dnac.example.com/dna/system/api/v1/auth/token" \
     -H "Content-Type: application/json" \
     -u "api_user:secret_value" | jq -r '.Token')

# 3. トークンを使用した API 呼び出し
curl -s -k -X GET "https://dnac.example.com/dna/intent/api/v1/network-device" \
     -H "Content-Type: application/json" \
     -H "x-auth-token: $TOKEN" | jq '.'
```


## 🔍 検証コマンド

Cisco DNA Center の API 認証・認可ステータス、ログ確認、および CLI/GUI からの検証コマンドを下表にまとめます。

| 目的 | コマンド / 確認箇所 |
| --- | --- |
| **API トークン手動検証** | <code>curl -k -v -X GET https://&lt;DNAC_IP&gt;/dna/intent/api/v1/network-device -H "x-auth-token: &lt;TOKEN&gt;"</code> |
| **ユーザーロール・権限確認** | DNAC GUI: <code>System &gt; Settings &gt; Authentication and Policy Server</code> / <code>Users &amp; Roles</code> |
| **API 監査ログの確認** | DNAC GUI: <code>System &gt; Audit Logs</code> (Filter by Category: API / User Access) |
| **DNAC 内部 API サービスログ** | DNAC CLI: <code>magctl service logs -f apimgmt-service</code> |
| **外部 AAA 連携ログ (ISE)** | Cisco ISE GUI: <code>Operations &gt; RADIUS / TACACS Livelogs</code> |
| **アクティブセッション確認** | DNAC GUI: <code>System &gt; Settings &gt; Users &amp; Roles &gt; Active Sessions</code> |
| **Python 通信デバッグ** | <code>import logging; logging.basicConfig(level=logging.DEBUG)</code> |


---

## 🚨 トラブルシュート

API 認証・認可に関するトラブル、発生原因、確認コマンド、および対処方法を下表に整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **HTTP 401 Unauthorized** | 1. ユーザー名・パスワード不一致<br>2. トークンの有効期限切れ（60分経過）<br>3. `x-auth-token` ヘッダー名のタイポ | <code>curl -u "user:pass" ... /auth/token</code><br>Python ログ確認 | 1. 資格情報を再確認<br>2. 自動トークン再取得ロジックを実装<br>3. ヘッダー名が小文字の `x-auth-token` であるか確認 |
| **HTTP 403 Forbidden** | ユーザーに割り当てられた RBAC ロール（例: OBSERVER-ROLE）が該当操作（書き込み/変更）に対応していない | DNAC GUI: <code>Users &amp; Roles</code><br>Audit Logs | 対象アカウントに NETWORK-ADMIN-ROLE または SUPER-ADMIN-ROLE を付与 |
| **HTTP 415 Unsupported Media Type** | `Content-Type: application/json` ヘッダーが欠落している | cURL / Python リクエストヘッダー確認 | リクエストヘッダーに `Content-Type: application/json` を明示的に追加 |
| **SSLError / Connection Refused** | 1. 証明書検証エラー（自己署名証明書）<br>2. ポート 443 のブロック（FW等） | <code>curl -k https://&lt;DNAC_IP&gt;</code><br><code>nc -zv &lt;DNAC_IP&gt; 443</code> | 1. Python で `verify=False` を設定<br>2. FW ルールで TCP 443 の到達性を許可 |
| **KeyError: 'Token'** | 認証成功レスポンスの辞書キー参照で `response.json()['token']` (小文字) と指定している | Python レスポンス出力確認 (`print(res.json())`) | キー名を大文字始まりの `'Token'` に変更 |
| **外部 AAA 認証失敗** | DNAC と Cisco ISE / RADIUS 間の Shared Secret 不一致、または TACACS+/RADIUS 設定不備 | Cisco ISE Livelogs<br>DNAC <code>Authentication Server Logs</code> | ISE 側の Client IP / Shared Secret、DNAC 側の AAA 設定を再確認 |


---

## ⚠ 制限事項

* **トークン有効期限の非可変性**: Cisco DNA Center の API トークン有効期限はデフォルトで **1時間（60分）** に固定されており、GUI/CLI から個別延長することはできません。
* **同時アクティブセッション数上限**: 同一ユーザーアカウントにおける同時アクティブ API セッション数には制限があります。大量の分散スクリプトを実行する場合は専有のサービスアカウントを作成することを推奨します。
* **API Rate Limiting**: DNA Center リソース保護のため、API リクエスト数（TPS: Transactions Per Second）に上限が設定されています。過剰なポーリングを行うと `HTTP 429 Too Many Requests` が返される場合があります。
* **トークン明示的破棄機能の制限**: 一部のバージョンにおいて、発行済みトークンを即座に無効化（Revoke）する API がサポートされていない場合があるため、タイムアウトを待つかセッション消去が必要です。


---

## 🔄 他技術との関連

Cisco DNA Center API 認証・認可は、以下のセキュリティインフラ構成要素と密接に連携します。

* **Cisco ISE (Identity Services Engine)**: RADIUS / TACACS+ / ERS API 経由で DNAC の外部アイデンティティプロバイダとして機能。ユーザーグループ属性を DNAC 内の RBAC ロールへ同期。
* **Identity Management / LDAP / Active Directory**: エンタープライズディレクトリサービスと連携し、単一の識別子（SSO / Centralized Auth）で DNAC API へのアクセスを集中管理。
* **Firewall / Access Control Lists (ACL)**: 開発端末・自動化サーバーから DNAC 管理インターフェース（HTTPS / TCP 443）への安全な到達性を確保。
* **Secure Syslog / SIEM Integration**: DNAC の API 認証試行ログ（成功・失敗・権限昇格）を syslog 経由で SIEM (Splunk / QRadar) に転送し、リアルタイム監査を実行。


---

## 🧩 比較表

Cisco DNA Center API 認証と他主要 Cisco セキュリティ製品の API 認証方式の比較を下表に示します。

| 機能 / 仕様 | Cisco DNA Center Intent API | Cisco FMC REST API | Cisco ISE ERS / OpenAPI | Cisco IOS-XE RESTCONF |
| --- | --- | --- | --- | --- |
| **初期認証方式** | HTTP Basic Auth (`POST /auth/token`) | HTTP Basic Auth (`POST /api/fmc_platform/v1/auth/generatetoken`) | HTTP Basic Auth | HTTP Basic Auth / OAuth2 |
| **API 認可ヘッダー** | `x-auth-token: <Token>` | `X-Auth-Access-Token: <Token>` | `Authorization: Basic <base64>` | `Authorization: Basic <base64>` |
| **トークンレスポンスキー** | `{"Token": "..."}` | `X-Auth-Access-Token` (Header) | なし (Basic Auth毎時) | なし (Basic Auth毎時) |
| **トークン有効期限** | 60 分 | 30 分 (最大 3 回更新可能) | セッション依存 | セッション依存 |
| **証明書検証要件** | HTTPS / TLS 1.2/1.3 | HTTPS / TLS 1.2/1.3 | HTTPS / TLS 1.2 | HTTPS / TLS 1.2 |


---

## 💡 ベストプラクティス

1. **環境変数・シークレットマネージャによる資格情報保護**:
   * スクリプト内に `username` や `password` をハードコードせず、`os.getenv('DNAC_USER')` や HashiCorp Vault / AWS Secrets Manager から動的に読み込む。
2. **Session オブジェクトの活用**:
   * リクエスト毎に `requests.get()` や `requests.post()` を個別呼び出しするのではなく、`requests.Session()` を生成してヘッダーや TLS 設定を共有し、TCP ハンドシェイクのオーバーヘッドを削減する。
3. **HTTP 401/429 のエラーハンドリングと指数バックオフ（Exponential Backoff）**:
   * リクエスト失敗時に即座に再試行するのではなく、`time.sleep(2 ** retry_count)` などのバックオフを導入して API ゲートウェイの過負荷を防ぐ。
4. **Dedicated API Service Account の利用**:
   * 管理個人のインタラクティブ用アカウントではなく、API 専用のサービスアカウント（例: `svc_dnac_automation`）を作成し、必要最小限の RBAC ロールを割り当てる。


## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験の実践的な対策として、Cisco DNA Center の API 認証・認可に関連する 10 個の完全な Python スクリプトおよび設定シナリオを掲載します。コードはすべて省略せず、単体で動作する形式で記述しています。

---

### 【演習 1】DNA Center 認証トークン取得と例外処理スクリプト

* **問題**: Cisco DNA Center へ認証リクエストを送信し、トークンを取得して標準出力へ表示せよ。接続エラーや認証失敗時の例外処理を実装すること。
* **要件**:
  1. `requests.post()` を使用し、`POST /dna/system/api/v1/auth/token` を呼び出すこと。
  2. SSL 警告を抑制し、`verify=False` で接続すること。
  3. 取得したトークン（レスポンスの `"Token"`）を表示すること。

```python
import requests
import urllib3
import sys
import os

# SSL 警告の抑制
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_HOST = "dnac.example.com"
API_USER = "api_user"
API_PASS = os.getenv("DNAC_PASSWORD", "secret_value")

def get_token(host, username, password):
    url = f"https://{host}/dna/system/api/v1/auth/token"
    headers = {"Content-Type": "application/json"}
    
    try:
        response = requests.post(url, auth=(username, password), headers=headers, verify=False, timeout=10)
        response.raise_for_status()
        token_data = response.json()
        
        if "Token" in token_data:
            print(f"[SUCCESS] Token Retrieved Successfully!")
            print(f"Token: {token_data['Token']}")
            return token_data["Token"]
        else:
            print("[ERROR] 'Token' key not found in response JSON.")
            return None
            
    except requests.exceptions.HTTPError as http_err:
        print(f"[HTTP ERROR] Status Code: {response.status_code}, Message: {http_err}")
        print(f"Response Body: {response.text}")
    except requests.exceptions.ConnectionError as conn_err:
        print(f"[CONNECTION ERROR] Failed to connect to {host}: {conn_err}")
    except Exception as e:
        print(f"[ERROR] Unexpected Exception: {e}")
    return None

if __name__ == "__main__":
    token = get_token(DNAC_HOST, API_USER, API_PASS)
    if not token:
        sys.exit(1)
```

---

### 【演習 2】取得トークンを使用したデバイスインベントリ取得スクリプト

* **問題**: 演習 1 で取得したトークンを `x-auth-token` ヘッダーにセットし、DNAC 管理下の全デバイスのホスト名、管理 IP、管理プラットフォーム、およびシリアル番号を抽出し出力せよ。

```python
import requests
import urllib3
import os

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_HOST = "dnac.example.com"
API_USER = "api_user"
API_PASS = os.getenv("DNAC_PASSWORD", "secret_value")

def fetch_devices(host, token):
    url = f"https://{host}/dna/intent/api/v1/network-device"
    headers = {
        "Content-Type": "application/json",
        "x-auth-token": token
    }
    
    response = requests.get(url, headers=headers, verify=False)
    if response.status_code == 200:
        devices = response.json().get("response", [])
        print(f"=== Total Devices Discovered: {len(devices)} ===")
        for dev in devices:
            hostname = dev.get("hostname", "N/A")
            mgmt_ip = dev.get("managementIpAddress", "N/A")
            platform = dev.get("platformId", "N/A")
            serial = dev.get("serialNumber", "N/A")
            role = dev.get("role", "N/A")
            print(f"Host: {hostname:20s} | IP: {mgmt_ip:15s} | Platform: {platform:15s} | Serial: {serial:12s} | Role: {role}")
    else:
        print(f"Failed to fetch devices. Status Code: {response.status_code}")

if __name__ == "__main__":
    auth_url = f"https://{DNAC_HOST}/dna/system/api/v1/auth/token"
    res = requests.post(auth_url, auth=(API_USER, API_PASS), verify=False)
    if res.status_code == 200:
        token = res.json()["Token"]
        fetch_devices(DNAC_HOST, token)
```

---

### 【演習 3】トークン自動リフレッシュ機能付き API クラスの実装

* **問題**: 認証状態を保持し、HTTP 401（Unauthorized）が発生した場合に自動でトークンを再取得してリクエストを継続するクラス `DNACSessionManager` を実装せよ。

```python
import requests
import urllib3
import os

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

class DNACSessionManager:
    def __init__(self, host, username, password):
        self.base_url = f"https://{host}"
        self.username = username
        self.password = password
        self.session = requests.Session()
        self.session.verify = False
        self.token = None
        self.login()

    def login(self):
        url = f"{self.base_url}/dna/system/api/v1/auth/token"
        headers = {"Content-Type": "application/json"}
        resp = self.session.post(url, auth=(self.username, self.password), headers=headers)
        if resp.status_code == 200:
            self.token = resp.json()["Token"]
            self.session.headers.update({"x-auth-token": self.token})
            print("[SESSION] Authentication successful. Token updated in session headers.")
        else:
            raise PermissionError(f"Login failed: {resp.status_code} - {resp.text}")

    def execute_get(self, endpoint, params=None):
        url = f"{self.base_url}{endpoint}"
        resp = self.session.get(url, params=params)
        
        if resp.status_code == 401:
            print("[SESSION] Token expired (401). Re-authenticating...")
            self.login()
            resp = self.session.get(url, params=params)
            
        return resp

if __name__ == "__main__":
    mgr = DNACSessionManager("dnac.example.com", "api_user", os.getenv("DNAC_PASSWORD", "secret_value"))
    resp = mgr.execute_get("/dna/intent/api/v1/network-device/count")
    if resp.status_code == 200:
        print(f"Device Count Response: {resp.json()}")
```

---

### 【演習 4】Command Runner API による CLI 実行と非同期 Task 追跡

* **問題**: DNA Center API を経由して特定スイッチに `show version` を実行し、非同期タスク ID (`taskId`) をポーリングしてコマンド実行結果を取得するスクリプトを作成せよ。

```python
import requests
import urllib3
import time
import os

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_HOST = "dnac.example.com"
API_USER = "api_user"
API_PASS = os.getenv("DNAC_PASSWORD", "secret_value")

def run_command(host, token, device_id, command):
    url = f"https://{host}/dna/intent/api/v1/network-device-poller/cli/read-request"
    headers = {"Content-Type": "application/json", "x-auth-token": token}
    payload = {
        "commands": [command],
        "deviceUuids": [device_id]
    }
    
    res = requests.post(url, json=payload, headers=headers, verify=False)
    if res.status_code != 202:
        print(f"Failed to initiate command. Status: {res.status_code}, Body: {res.text}")
        return
        
    task_id = res.json()["response"]["taskId"]
    print(f"[STEP 1] Command Initiated. Task ID: {task_id}")
    
    task_url = f"https://{host}/dna/intent/api/v1/task/{task_id}"
    for _ in range(10):
        time.sleep(2)
        task_res = requests.get(task_url, headers=headers, verify=False)
        task_data = task_res.json()["response"]
        
        if task_data.get("endTime"):
            file_id = task_data.get("progress")
            print(f"[STEP 2] Task Completed. File ID: {file_id}")
            break
    else:
        print("Task timed out.")
        return

    file_url = f"https://{host}/dna/intent/api/v1/file/{file_id}"
    file_res = requests.get(file_url, headers=headers, verify=False)
    print("=== CLI Output ===")
    print(file_res.json())

if __name__ == "__main__":
    token = requests.post(f"https://{DNAC_HOST}/dna/system/api/v1/auth/token", auth=(API_USER, API_PASS), verify=False).json()["Token"]
    dev_res = requests.get(f"https://{DNAC_HOST}/dna/intent/api/v1/network-device", headers={"x-auth-token": token}, verify=False)
    devices = dev_res.json()["response"]
    if devices:
        target_uuid = devices[0]["id"]
        print(f"Targeting Device: {devices[0]['hostname']} ({target_uuid})")
        run_command(DNAC_HOST, token, target_uuid, "show version")
```

---

### 【演習 5】ネットワーク健全性（Overall Network Health）スコアの取得

* **問題**: Intent API `/dna/intent/api/v1/network-health` を呼び出し、ネットワーク全体のヘルススコアおよびカテゴリ別スコア（Router, Switch, Wireless）を出力せよ。

```python
import requests
import urllib3
import os

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_HOST = "dnac.example.com"
API_USER = "api_user"
API_PASS = os.getenv("DNAC_PASSWORD", "secret_value")

def get_health_score():
    token = requests.post(f"https://{DNAC_HOST}/dna/system/api/v1/auth/token", auth=(API_USER, API_PASS), verify=False).json()["Token"]
    headers = {"x-auth-token": token, "Content-Type": "application/json"}
    
    url = f"https://{DNAC_HOST}/dna/intent/api/v1/network-health"
    resp = requests.get(url, headers=headers, verify=False)
    
    if resp.status_code == 200:
        health_data = resp.json().get("response", [])
        print("=== Network Health Summary ===")
        for item in health_data:
            category = item.get("category")
            health_score = item.get("healthScore")
            total_count = item.get("totalCount")
            good_count = item.get("goodCount")
            print(f"Category: {category:15s} | Health Score: {health_score}% | Good/Total: {good_count}/{total_count}")
    else:
        print(f"Error fetching health score: {resp.status_code}")

if __name__ == "__main__":
    get_health_score()
```


### 【演習 6】ネットワークトポロジー情報（Topology API）のダウンロードと解析

* **問題**: DNAC Physical Topology API (`GET /dna/intent/api/v1/topology/physical-topology`) を実行し、発見されたリンク一覧（送信元デバイス・ポート ── 宛先デバイス・ポート）を解析して表示せよ。

```python
import requests
import urllib3
import os

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_HOST = "dnac.example.com"
API_USER = "api_user"
API_PASS = os.getenv("DNAC_PASSWORD", "secret_value")

def get_physical_topology():
    token = requests.post(f"https://{DNAC_HOST}/dna/system/api/v1/auth/token", auth=(API_USER, API_PASS), verify=False).json()["Token"]
    headers = {"x-auth-token": token}
    
    url = f"https://{DNAC_HOST}/dna/intent/api/v1/topology/physical-topology"
    res = requests.get(url, headers=headers, verify=False)
    
    if res.status_code == 200:
        links = res.json()["response"]["links"]
        print(f"=== Discovered Physical Links ({len(links)}) ===")
        for idx, link in enumerate(links, 1):
            src = link.get("source", "N/A")
            dst = link.get("target", "N/A")
            src_port = link.get("startPortName", "N/A")
            dst_port = link.get("endPortName", "N/A")
            print(f"Link #{idx:02d}: Device {src} [{src_port}] <---> Device {dst} [{dst_port}]")

if __name__ == "__main__":
    get_physical_topology()
```

---

### 【演習 7】OBSERVER ロールによるアクセス制御（403 Forbidden 補獲）の検証

* **問題**: 読み取り専用ユーザー（`observer_user`）の資格情報でトークンを取得し、デバイス削除 API (`DELETE /dna/intent/api/v1/network-device/<id>`) を呼び出して `HTTP 403 Forbidden` が返されることを検証せよ。

```python
import requests
import urllib3
import os

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_HOST = "dnac.example.com"
OBSERVER_USER = "observer_user"
OBSERVER_PASS = os.getenv("OBSERVER_PASSWORD", "secret_value")

def test_rbac_restriction():
    tok_res = requests.post(f"https://{DNAC_HOST}/dna/system/api/v1/auth/token", auth=(OBSERVER_USER, OBSERVER_PASS), verify=False)
    if tok_res.status_code != 200:
        print(f"Failed to authenticate observer user: {tok_res.status_code}")
        return
        
    token = tok_res.json()["Token"]
    headers = {"x-auth-token": token, "Content-Type": "application/json"}
    
    dummy_uuid = "00000000-0000-0000-0000-000000000000"
    delete_url = f"https://{DNAC_HOST}/dna/intent/api/v1/network-device/{dummy_uuid}"
    
    res = requests.delete(delete_url, headers=headers, verify=False)
    print(f"Delete API Response Status Code: {res.status_code}")
    
    if res.status_code == 403:
        print("[TEST PASSED] RBAC is working correctly. Observer role is denied from performing DELETE actions.")
    else:
        print(f"[TEST FAILED] Unexpected status code: {res.status_code}")

if __name__ == "__main__":
    test_rbac_restriction()
```

---

### 【演習 8】パジネーション（Pagination: limit / offset）とトークン期限管理

* **問題**: 大量のネットワークインターフェースを無効化・エラー発生させることなく取得するため、`limit=50` および `offset` パラメータを使用したループパジネーションスクリプトを作成せよ。

```python
import requests
import urllib3
import os

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_HOST = "dnac.example.com"
API_USER = "api_user"
API_PASS = os.getenv("DNAC_PASSWORD", "secret_value")

def get_all_interfaces_paginated():
    token = requests.post(f"https://{DNAC_HOST}/dna/system/api/v1/auth/token", auth=(API_USER, API_PASS), verify=False).json()["Token"]
    headers = {"x-auth-token": token}
    
    offset = 1
    limit = 50
    total_interfaces = []
    
    while True:
        url = f"https://{DNAC_HOST}/dna/intent/api/v1/interface?offset={offset}&limit={limit}"
        res = requests.get(url, headers=headers, verify=False)
        
        if res.status_code != 200:
            print(f"Error at offset {offset}: {res.status_code}")
            break
            
        items = res.json().get("response", [])
        if not items:
            break
            
        total_interfaces.extend(items)
        print(f"Fetched {len(items)} interfaces (Offset: {offset})")
        offset += limit
        
    print(f"=== Total Interfaces Retrieved Across All Pages: {len(total_interfaces)} ===")

if __name__ == "__main__":
    get_all_interfaces_paginated()
```

---

### 【演習 9】無効なトークン送信時の 401 エラー補獲とログ出力スクリプト

* **問題**: 意図的に不正なトークン文字列を `x-auth-token` ヘッダーに設定して API を送信し、DNAC からの `401 Unauthorized` エラーメッセージ構造を検証せよ。

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_HOST = "dnac.example.com"
INVALID_TOKEN = "DUMMY_INVALID_TOKEN_FOR_TESTING"

def test_invalid_token():
    url = f"https://{DNAC_HOST}/dna/intent/api/v1/network-device"
    headers = {
        "Content-Type": "application/json",
        "x-auth-token": INVALID_TOKEN
    }
    
    res = requests.get(url, headers=headers, verify=False)
    print(f"HTTP Status Code: {res.status_code}")
    print("=== Response Body ===")
    print(res.text)
    
    if res.status_code == 401:
        print("[VERIFIED] Invalid token correctly rejected with 401 Unauthorized.")

if __name__ == "__main__":
    test_invalid_token()
```

---

### 【演習 10】マルチスレッド環境におけるセッション共有と安全な API 呼び出し

* **問題**: `concurrent.futures.ThreadPoolExecutor` を使用し、同一の認証トークンを安全に共有しながら複数デバイスのコンフィグ取得 API を並行実行するスクリプトを作成せよ。

```python
import requests
import urllib3
import os
from concurrent.futures import ThreadPoolExecutor

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_HOST = "dnac.example.com"
API_USER = "api_user"
API_PASS = os.getenv("DNAC_PASSWORD", "secret_value")

def fetch_device_detail(device, token):
    dev_id = device["id"]
    hostname = device.get("hostname", "Unknown")
    url = f"https://{DNAC_HOST}/dna/intent/api/v1/network-device/{dev_id}"
    headers = {"x-auth-token": token}
    
    res = requests.get(url, headers=headers, verify=False)
    if res.status_code == 200:
        return f"[THREAD SUCCESS] Host: {hostname} | MAC: {res.json()['response'].get('macAddress')}"
    else:
        return f"[THREAD ERROR] Host: {hostname} Status: {res.status_code}"

def main():
    token = requests.post(f"https://{DNAC_HOST}/dna/system/api/v1/auth/token", auth=(API_USER, API_PASS), verify=False).json()["Token"]
    dev_res = requests.get(f"https://{DNAC_HOST}/dna/intent/api/v1/network-device", headers={"x-auth-token": token}, verify=False)
    devices = dev_res.json().get("response", [])
    
    print(f"Starting Multi-threaded Audit for {len(devices)} Devices...")
    
    with ThreadPoolExecutor(max_workers=5) as executor:
        futures = [executor.submit(fetch_device_detail, dev, token) for dev in devices]
        for future in futures:
            print(future.result())

if __name__ == "__main__":
    main()
```


## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験および筆記試験（SCOR）を想定した出題形式の問題と詳細な解説です。

---

### 問題 1（コンフィグ・スクリプト読解）

以下の Python スクリプトを実行したところ、`KeyError: 'token'` が発生してスクリプトが異常終了しました。原因として最も適切なものを 1 つ選べ。

```python
import requests
import urllib3
import os
urllib3.disable_warnings()

url = "https://dnac.example.com/dna/system/api/v1/auth/token"
response = requests.post(url, auth=('api_user', os.getenv('API_PASS')), verify=False)
token = response.json()['token']
```

* (A) 認証リクエストの HTTP メソッドは `GET` である必要がある。
* (B) `requests.post()` の認証引数は `headers={'Authorization': 'Bearer ...'}` で指定する必要がある。
* (C) Cisco DNA Center の認証レスポンス JSON のトークンキー名は、小文字の `'token'` ではなく大文字始まりの `'Token'` である。
* (D) `verify=False` が指定されているため JSON レスポンスが空になる。

**正解**: (C)

**解説**: Cisco DNA Center の Intent API 認証エンドポイント（`/dna/system/api/v1/auth/token`）が返すレスポンス JSON は `{"Token": "token_string"}` という形式です。キー名が大文字始まりの `'Token'` であるため、小文字の `'token'` で辞書参照を行うと Python で `KeyError` が発生します。

---

### 問題 2（認証・認可トラブルシューティング）

Cisco DNA Center の Northbound API を利用する自動化スクリプトを運用している。スクリプト開始直後は正常にデバイス一覧を取得できていたが、開始から約 65 分後に実行された設定変更 API リクエストが `HTTP 401 Unauthorized` で拒否された。この問題の根本原因と最も適切な対応策の組み合わせを選べ。

* (A) DNAC の IP アドレスが変更されたため、DNS キャッシュをクリアする。
* (B) API トークンの有効期限（60分）が切れたため、401 エラーを補獲してトークンを再取得・更新するロジックをスクリプトに追加する。
* (C) ユーザーの RBAC ロールが変更されたため、DNAC 管理者に権限昇格を申請する。
* (D) `x-auth-token` ヘッダーを `Authorization: Bearer` へ変更する。

**正解**: (B)

**解説**: Cisco DNA Center が発行する `x-auth-token` の有効期限は固定で 60 分（1時間）です。1時間を超えて連続実行されるスクリプトでは、トークン失効に伴い HTTP 401 が返されるため、スクリプト側で 401 を検知して自動的に再認証（`/auth/token` の再実行）およびリクエストの再試行を行う実装が必要です。

---

### 問題 3（RBAC・アクセス制御 Design）

セキュリティ運用チームのジュニアアナリストに対して、Cisco DNA Center の Northbound API を経由した「ネットワーク状態およびデバイスインベントリの閲覧」のみを許可し、構成変更やデバイス削除などの書き込み操作を禁止したい。割り当てるべき最も適切な DNAC ロールを選べ。

* (A) SUPER-ADMIN-ROLE
* (B) NETWORK-ADMIN-ROLE
* (C) OBSERVER-ROLE
* (D) SYSTEM-ADMIN-ROLE

**正解**: (C)

**解説**: Cisco DNA Center の組み込みロールにおいて、`OBSERVER-ROLE` は読み取り専用（Read-Only）のアクセス権限を提供します。このロールが割り当てられたアカウントで `POST`, `PUT`, `DELETE` などの変更 API を呼び出すと `HTTP 403 Forbidden` エラーで拒否されます。

---

### 問題 4（API ヘッダー仕様）

取得した認証トークンを意図した API エンドポイント（例: `/dna/intent/api/v1/network-device`）へ送信する際、リクエストヘッダーとして正しく設定すべきキーと値のペアを選べ。

* (A) `Authorization: Bearer <Token>`
* (B) `x-auth-token: <Token>`
* (C) `Authentication: Token <Token>`
* (D) `X-Cisco-DNAC-API-Key: <Token>`

**正解**: (B)

**解説**: Cisco DNA Center の Intent API において、認証完了後の各 API リクエストで提示する認可ヘッダーは **`x-auth-token: <Token>`** です。`Authorization: Bearer` や `Authentication` ヘッダーではないため、ヘッダー名の指定には十分注意が必要です。

---

### 問題 5（外部 AAA 連携の実装）

Cisco DNA Center の API ユーザー認証を Cisco ISE（外部 RADIUS / TACACS+ サーバー）と統合する場合の認証フローおよびコンポーネントの役割として正しいステートメントを 2 つ選べ。

* (A) DNAC は API クライアントから受信した HTTP Basic Auth 資格情報を外部 AAA サーバーへ転送して検証する。
* (B) 外部 AAA 認証が成功した場合でも、API リクエスト時に使用するトークンは外部 AAA サーバーから直接発行される。
* (C) 外部 AAA サーバーから返されたグループ属性（VSAP / Group Attribute）に基づき、DNAC 内部の RBAC ロール（ネットワーク管理者、オブザーバー等）がマッピングされる。
* (D) 外部 AAA 連携を有効にすると、DNAC のローカルアカウントは即座に無効化され API 認証に使用できなくなる。

**正解**: (A), (C)

**解説**: 外部 AAA (ISE) 統合環境では、DNAC API ゲートウェイが受け取った資格情報を RADIUS/TACACS+ を通じて ISE へ転送し、認証を行います (A)。認証成功時、ISE から返却されたグループ情報に基づいて DNAC 内の特定 RBAC ロールへアクセス権がマッピングされます (C)。トークン自体は常に DNAC 自体が発行・管理します。

---

## 🔗 参考リソース

CCIE Security v6.1 学習および実務に役立つ Cisco 公式ドキュメントおよび開発者リソースの一覧です。

* Cisco DNA Center Intent API Resources (Cisco DevNet)
* Cisco Catalyst Center Authentication API Reference Guide
* Cisco Catalyst Center Administrator Guide - Manage Users and Roles
* Cisco DNA Center Northbound API Quick Start Guide
* Cisco DevNet Learning Labs: Getting Started with Cisco DNA Center REST APIs


---

## 📝 補足（Notes）

* **トークン表記の大文字・小文字**:
  * トークン取得時の JSON キー: `"Token"` (先頭大文字)
  * API 送信時の HTTP ヘッダー: `"x-auth-token"` (すべて小文字、ハイフン区切り)
* **Python `requests` のセッション再利用**:
  * `session = requests.Session()` を利用して `session.headers.update({"x-auth-token": token})` を設定しておくと、以降のすべての `session.get()` や `session.post()` で自動的にヘッダーが引き継がれ、コード記述を簡略化できます。
* **ラボ試験での事前動作確認手順**:
  1. `curl` で `/dna/system/api/v1/auth/token` へアクセスし、200 OK と Token が返るかテスト。
  2. トークンをコピーして `/dna/intent/api/v1/network-device` へ `x-auth-token` を添えて GET テスト。
  3. Python スクリプトを作成・実行。問題発生時は `print(response.status_code)` および `print(response.text)` を入れて原因特定。
