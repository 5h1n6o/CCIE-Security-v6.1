---
layout: default
title: 3.9.a-REST-API
nav_order: 1
parent: 3.9-APIs
grand_parent: 3.0-Security-Infrastructure
---

# 3.9.a REST API requests and responses

## 📘 概要

CCIE Security v6.1 Blueprint における「3.9.a REST API requests and responses」は、現代のシスコセキュリティインフラ（Cisco Firepower Management Center: FMC、Cisco Identity Services Engine: ISE、Cisco Firepower Device Manager: FDM、および Cisco IOS-XE デバイス）に対して、REST (Representational State Transfer) API を介してプログラムからアクセス・制御するための標準的な通信要求（Request）および応答（Response）の構造とメカニズムを定義したコア領域です。

本項目では、HTTP/HTTPS をトランスポート階層とし、以下の3つのサブトピックで構成される基本仕様とプログラミング実装の完全な習得が求められます。

* **3.9.a (i) HTTP action verbs, error codes, cookies, headers**: REST API の基本動作を定義する HTTP 動詞（GET, POST, PUT, DELETE, PATCH）、ステータスコード（200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 415 Unsupported Media Type, 500 Internal Server Error など）、Cookie ベースのセッション管理、ならびにリクエスト/レスポンスヘッダー（Content-Type, Accept, Authorization, X-auth-access-token, Cookie など）の正確な相互作用。
* **3.9.a (ii) JSON or XML payload**: リクエストボディおよびレスポンスボディで使用される構造化データフォーマットである JSON (JavaScript Object Notation) および XML (Extensible Markup Language) の構文仕様、パース処理、ならびにデータ構造の相互変換（Python `json` モジュール、`xml.etree.ElementTree` や `xmltodict` など）。
* **3.9.a (iii) Authentication**: ターゲットセキュリティ機器との安全な通信チャネルを確立・維持するための各種認証メカニズム。具体的には、HTTP Basic Authentication、Token-based Authentication (FMC の X-auth-access-token / X-auth-refresh-token、OAuth 2.0 / Bearer Token)、API Key 認証、Cookie/Session-based 認証、および相互 TLS (mTLS) 認証の動作シーケンスと実装。

ラボ試験においては、これら API の要求・応答モデルを理解し、標準的な Python ライブラリ（`requests`, `json`, `urllib3` など）を用いて、ネットワークセキュリティ機器の設定取得、オブジェクト作成、セキュリティポリシーの変更、デプロイメントのトリガー、およびシステム状態の監視をプログラマティックに実行できる能力が厳しく審査されます。


---

## 🔑 要点

REST API Requests and Responses に関する主要技術要素、用途、制限事項、設計上の注意点を下表に整理します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | HTTP/HTTPS をトランスポートとし、無状態（Stateless）なクライアント・サーバーアーキテクチャに基づいてリソース（URI）に対する操作を統一的な HTTP メソッドで実行するプログラマティックインターフェース |
| **用途** | Cisco FMC/ISE/FDM/IOS-XE の設定自動化、セキュリティオブジェクト一括登録、アクセスポリシーの動的変更、脅威インテリジェンス（IP/URL/Hash）の自動ブロック、監査ログ収集 |
| **メリット** | 標準的な HTTP/JSON 技術に基づくためプログラミング言語に依存せず、CLI/GUI 操作の自動化・高速化・ヒューマンエラー防止を実現可能 |
| **デメリット** | 大量リクエスト時のレート制限（Rate Limiting）やトークン有効期限（Token Expiration）に伴う再認証処理のオーバーヘッド、ネットワーク遅延の影響 |
| **対応機種** | Cisco FMC (v6.0+), Cisco ISE (v2.0+ ERS / v3.1+ OpenAPI), Cisco FDM (v6.2+), Cisco IOS-XE (RESTCONF) |
| **制限事項** | セッション数制限、API 要求毎のレート制限（例: FMC では毎分一定リクエスト数超で 429 Too Many Requests）、バルク処理の上限数 |
| **設計上の注意点** | セッション/トークンのライフサイクル管理（自動リフレッシュ）、自己署名証明書使用時の SSL 検証の適切なハンドリング（`verify=False` 時の警告無効化）、例外処理（HTTP ステータスコードの判定と再試行ロジック）の実装 |


---

## 🏗 動作原理

REST API は、Web の標準的アーキテクチャスタイルに準拠し、すべての設定項目や状態を「リソース」として URI (Uniform Resource Identifier) で表現します。クライアント（Python スクリプト等）は HTTP/HTTPS プロトコルを介してサーバー（FMC/ISE 等）にリクエストを送信し、サーバーは処理結果をステータスコードおよびレスポンスボディ（JSON/XML）として返却します。

### REST API 通信構造フロー

```
[ Python Client / Script ]
        │
        │  1. HTTP/HTTPS Request
        │     - HTTP Verb: GET / POST / PUT / DELETE / PATCH
        │     - URI: https://<Device_IP>/api/fmc_config/v1/domain/<domain_UUID>/object/networks
        │     - Headers: Content-Type: application/json, Accept: application/json, X-auth-access-token: <Token>
        │     - Body: { "name": "Host-10.1.1.1", "value": "10.1.1.1", "type": "NetworkAnalysisPolicy" ... }
        │
        ▼
[ Security Device: FMC / ISE / FDM / IOS-XE ]
        │
        │  2. Authentication & Authorization Check (Token / Session Validation)
        │  3. Payload Parsing & Schema Validation (JSON/XML Check)
        │  4. Backend Configuration Database Update / Execution
        │
        │  5. HTTP/HTTPS Response
        │     - Status Code: 200 OK / 201 Created / 204 No Content / 400 Bad Request / 401 Unauthorized
        │     - Headers: Content-Type: application/json, Cache-Control: no-cache
        │     - Body: { "id": "005056B8-11E1-0ed3-0000-017179869282", "name": "Host-10.1.1.1", ... }
        ▼
[ Python Client / Script ]
```

### 1. HTTP Action Verbs (動作動詞) の使い分け
* **GET**: リソースの取得。サーバー側の状態を変更しない冪等（Idempotent）かつ安全な操作。成功時 `200 OK`。
* **POST**: 新しいリソースの作成、または認証トークンの発行。非冪等（Non-Idempotent）な操作。成功時 `201 Created`（作成されたリソースの URI や JSON を返却）または `200 OK`。
* **PUT**: 既存リソースの全体置換（Replace）、または特定 ID を指定した作成。冪等な操作。成功時 `200 OK` または `204 No Content`。
* **DELETE**: 既存リソースの削除。冪等な操作。成功時 `200 OK` または `204 No Content`。
* **PATCH**: 既存リソースの一部更新（Partial Update）。変更点のみを送信。成功時 `200 OK` または `204 No Content`。

### 2. HTTP Status Error Codes (ステータスコード) の分類
* **2xx (Success)**:
  * `200 OK`: 処理成功（GET, PUT, POST で結果データがある場合）。
  * `201 Created`: POST によるリソース作成が成功。
  * `202 Accepted`: 非同期処理（例: FMC の Deploy）が受け付けられた。
  * `204 No Content`: 処理成功だがレスポンスボディが空（DELETE や一部の PUT）。
* **4xx (Client Errors)**:
  * `400 Bad Request`: リクエスト構文エラー、JSON フォーマット不正、必須フィールドの不足。
  * `401 Unauthorized`: 認証ヘッダーの欠落、無効なトークン、期限切れトークン。
  * `403 Forbidden`: 認証は成功したがアクセス権限不足（RBAC 違反）。
  * `404 Not Found`: 指定された URI やリソース ID が存在しない。
  * `409 Conflict`: リソースの重複（同名のオブジェクトが既に存在）や依存関係による削除不可。
  * `415 Unsupported Media Type`: `Content-Type` ヘッダーが不適切（例: JSON 必須の場所に text/plain）。
  * `429 Too Many Requests`: レート制限超過。
* **5xx (Server Errors)**:
  * `500 Internal Server Error`: サーバー内部エラー。
  * `503 Service Unavailable`: サーバー過負荷またはメンテナンス中。

### 3. Header & Cookie の制御
* **Content-Type**: クライアントが送信するリクエストボディのデータ形式を指定（例: `application/json`, `application/xml`）。
* **Accept**: クライアントが受け取りを希望するレスポンスデータ形式を指定（例: `application/json`）。
* **Authorization**: 認証情報を伝送。Basic 認証では `Basic <base64(user:pass)>`、Bearer トークン認証では `Bearer <token>`。
* **Custom Headers**: Cisco FMC での `X-auth-access-token` や `X-auth-refresh-token` など、特定のセキュリティ製品固有の認証トークン保持ヘッダー。
* **Cookie**: サーバーが発行したセッション ID（例: `APIC-cookie` や `JSESSIONID`）を保持し、以降のリクエストに付与してステートフルな認証状態を維持。


---

## ⚙ 動作シーケンス

Cisco セキュリティ機器に対する REST API 通信の一般的な動作シーケンスは、認証ハンドシェイク、トークン保持、リクエスト実行、エラー判定、およびトークン破棄のステップで構成されます。

### FMC REST API 認証およびデータ取得・更新の標準シーケンス

```
[ Python Client ]                            [ Cisco FMC Server ]
       │                                              │
       │ 1. POST /api/fmc_platform/v1/auth/generatetoken
       │    Headers: Authorization: Basic <Base64(user:pass)>
       │ ───────────────────────────────────────────► │
       │                                              │ (認証判定 & トークン生成)
       │ 2. 204 No Content (または 200 OK)             │
       │    Response Headers:                         │
       │      X-auth-access-token: <Access_Token_A>   │
       │      X-auth-refresh-token: <Refresh_Token_R> │
       │      DOMAIN_UUID: <Domain_UUID_D>            │
       │ ◄─────────────────────────────────────────── │
       │                                              │
       │ 3. GET /api/fmc_config/v1/domain/<UUID_D>/object/networks
       │    Headers: X-auth-access-token: <Access_Token_A>
       │             Accept: application/json
       │ ───────────────────────────────────────────► │
       │                                              │ (トークン検証 & データベース検索)
       │ 4. 200 OK                                    │
       │    Response Body: JSON (Network Objects List)│
       │ ◄─────────────────────────────────────────── │
       │                                              │
       │ 5. POST /api/fmc_config/v1/domain/<UUID_D>/object/networks
       │    Headers: X-auth-access-token: <Access_Token_A>
       │             Content-Type: application/json
       │    Body: { "name": "Web_Server", "value": "192.168.10.50", "type": "Network" }
       │ ───────────────────────────────────────────► │
       │                                              │ (オブジェクト登録)
       │ 6. 201 Created                               │
       │    Response Body: JSON (Created Object Detail)
       │ ◄─────────────────────────────────────────── │
       │                                              │
       │ 7. (トークン有効期限 30分経過時)            │
       │    POST /api/fmc_platform/v1/auth/refreshtoken
       │    Headers: X-auth-access-token: <Access_Token_A>
       │             X-auth-refresh-token: <Refresh_Token_R>
       │ ───────────────────────────────────────────► │
       │                                              │ (トークン更新)
       │ 8. 204 No Content                            │
       │    Response Headers:                         │
       │      X-auth-access-token: <New_Access_Token> │
       │ ◄─────────────────────────────────────────── │
```

このシーケンスにより、管理者パスワードの平文送信を最初のトークン取得時のみに限定し、以降の操作は一時的なアクセストークンを用いて安全かつ高効率に処理されます。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、REST API 領域で最も失点しやすいポイントと対策事項を体系化します。

### 1. Blueprintで重要なポイント
* **Python `requests` モジュールによる完全な通信実装**: サードパーティの外部ライブラリ（fmcapi など）は試験環境で利用できない可能性が高いため、標準の `requests` ライブラリおよび `json` モジュールのみで HTTP ヘッダー、レスポンス解析、認証管理を完結させるスキルが要求されます。
* **ヘッダーの正確な構築**: FMC や ISE ERS において、`Content-Type: application/json` や `Accept: application/json`（ISE の場合は `application/json` または `application/xml`）を正しく付与しないと、`415 Unsupported Media Type` や `406 Not Acceptable` エラーが発生します。
* **SSL 検証の無効化と警告抑制**: ラボ環境では自己署名証明書が使用されるため、`requests.get(..., verify=False)` の指定が必須となります。さらに、実行時の端末ログを汚さないために `urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)` を記述することが推奨されます。
* **JSON ペイロードの構造理解**: オブジェクト作成やポリシー変更時の必須フィールド（FMC の場合は `name`, `value`, `type` など）の欠落による `400 Bad Request` を即座に診断・修復できるデバッグ能力。

### 2. よくある設定・実装ミスと対策
* **HTTP ヘッダーへのトークン設定忘れ**: FMC では `X-auth-access-token` ヘッダーにアクセストークンをセットする必要があります。`Authorization: Bearer` と混同すると `401 Unauthorized` となります。
* **JSON のエンコード/デコード不備**: `requests.post(url, data=payload)` に辞書型オブジェクトを直接渡すとエラーになります。`json=payload` パラメータを使用するか、`data=json.dumps(payload)` を使用して明示的に JSON 文字列化します。
* **レスポンスオブジェクトの扱い誤り**: レスポンスボディを取得する際、`response.json()` は JSON を Python の辞書/リストに変換しますが、レスポンスが空（`204 No Content`）の場合に `response.json()` を呼び出すと `json.decoder.JSONDecodeError` 例外が発生します。ステータスコードを事前チェック（`if response.status_code == 200:` 等）するか、`response.text` の有無を確認する実装が必要です。


---

## 🛠 設定方法

Cisco FMC、ISE、および IOS-XE (RESTCONF) に対して REST API リクエストを送信し、レスポンスを処理する標準的な Python スクリプト実装例を掲載します。

### 1. Cisco FMC REST API 認証 & ネットワークオブジェクト作成スクリプト

```python
import json
import requests
import urllib3

# 自己署名証明書の警告を無効化
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# FMC 接続情報
FMC_IP = "192.168.10.50"
USERNAME = "admin"
PASSWORD = "AdminPassword123!"

BASE_URL = f"https://{FMC_IP}/api"

# Step 1: 認証トークンおよび Domain UUID の取得
auth_url = f"{BASE_URL}/fmc_platform/v1/auth/generatetoken"
headers = {"Content-Type": "application/json"}

try:
    response = requests.post(
        auth_url,
        auth=(USERNAME, PASSWORD),
        headers=headers,
        verify=False
    )
    
    if response.status_code == 204 or response.status_code == 200:
        access_token = response.headers.get("X-auth-access-token")
        refresh_token = response.headers.get("X-auth-refresh-token")
        domain_uuid = response.headers.get("DOMAIN_UUID")
        print(f"[+] Token Auth Successful!")
        print(f"    Domain UUID: {domain_uuid}")
    else:
        print(f"[-] Auth Failed. Status Code: {response.status_code}, Response: {response.text}")
        exit(1)

except Exception as e:
    print(f"[-] Connection Error: {e}")
    exit(1)

# Step 2: 共通ヘッダーの設定
api_headers = {
    "Content-Type": "application/json",
    "Accept": "application/json",
    "X-auth-access-token": access_token
}

# Step 3: Network Host オブジェクトの作成 (POST)
object_url = f"{BASE_URL}/fmc_config/v1/domain/{domain_uuid}/object/networks"

payload = {
    "name": "Host_DB_Server",
    "value": "10.200.1.100",
    "type": "Network",
    "description": "Created via Python REST API"
}

create_resp = requests.post(
    object_url,
    data=json.dumps(payload),
    headers=api_headers,
    verify=False
)

if create_resp.status_code == 201:
    created_obj = create_resp.json()
    print(f"[+] Network Object Created Successfully!")
    print(f"    Object Name: {created_obj.get('name')}")
    print(f"    Object ID:   {created_obj.get('id')}")
else:
    print(f"[-] Failed to Create Object. Status: {create_resp.status_code}")
    print(f"    Details: {create_resp.text}")
```

### 2. Cisco ISE ERS (External RESTful Services) API 認証 & Guest User 取得 (XML / JSON)

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

ISE_IP = "192.168.10.70"
ERS_USER = "ers_admin"
ERS_PASS = "ErsPassword123!"

# ISE ERS API はポート 9060 を使用
url = f"https://{ISE_IP}:9060/ers/config/internaluser"

headers = {
    "Accept": "application/json",
    "Content-Type": "application/json"
}

# ERS API は HTTP Basic 認証を使用
response = requests.get(
    url,
    auth=(ERS_USER, ERS_PASS),
    headers=headers,
    verify=False
)

if response.status_code == 200:
    data = response.json()
    print(f"[+] Internal Users Retrieved ({len(data['SearchResult']['resources'])} users found):")
    for user in data['SearchResult']['resources']:
        print(f"    - Name: {user['name']}, ID: {user['id']}")
else:
    print(f"[-] ERS Request Failed. Status: {response.status_code}, Text: {response.text}")
```


---

## 🔍 検証コマンド

Python スクリプト内でのデバッグ検証手法および curl を用いた対向機器 API レスポンスの直接確認コマンドを表形式で整理します。

| 目的 | 確認手法 / コマンド |
| --- | --- |
| **Python `requests` 詳細ログ有効化** | `import logging; logging.basicConfig(level=logging.DEBUG)` をスクリプト冒頭に追加し、送信パケット・受信用ヘッダーを完全出力 |
| **HTTP レスポンス状態確認** | `print(response.status_code)`, `print(response.headers)`, `print(response.text)` |
| **JSON レスポンスの整形表示** | `print(json.dumps(response.json(), indent=4))` |
| **curl による FMC 認証テスト** | `curl -k -v -X POST -u "admin:AdminPass123!" -H "Content-Type: application/json" https://<FMC_IP>/api/fmc_platform/v1/auth/generatetoken` |
| **curl による FMC リソース取得** | `curl -k -X GET -H "X-auth-access-token: <TOKEN>" -H "Accept: application/json" https://<FMC_IP>/api/fmc_config/v1/domain/<UUID>/object/networks` |
| **curl による ISE ERS GET テスト** | `curl -k -u "ers_admin:ErsPass123!" -H "Accept: application/json" https://<ISE_IP>:9060/ers/config/networkdevice` |


---

## 🚨 トラブルシュート

REST API 通信時の典型的な障害原因と確認手順および解決策を下表にまとめます。

| 症状 | 原因 | 確認手順 | 対処方法 |
| --- | --- | --- | --- |
| **401 Unauthorized** | 認証資格情報の誤り、トークンの期限切れ、または認証ヘッダー名のミス | `print(response.headers)` でトークン発行有無を確認。FMC は `X-auth-access-token` を使用しているか確認 | `generatetoken` を再実行して新しいトークンを取得し、ヘッダーに正しくセットする |
| **403 Forbidden** | 管理者アカウントの RBAC 権限不足、または API 機能が無効化されている | ユーザーロール（FMC: API Access enabled, ISE: ERS Admin group への所属）を確認 | ISE Admin UI で ERS API を有効化（`Administration > System > Settings > ERS Settings`）し、ユーザーに ERS Admin 権限を付与する |
| **400 Bad Request** | JSON ペイロードの構文エラー、不適切なデータ型、または必須プロパティの不備 | `response.text` を出力し、API スキーマ違反の詳細エラーメッセージを解析 | JSON ペイロード構造を見直し、文字列型・数値型・配列構造が公式 API スキーマと一致しているか確認する |
| **415 Unsupported Media Type** | リクエストヘッダー `Content-Type` の欠落または値の誤り | `request.headers` を確認（`Content-Type: application/json` が存在するか） | `headers={'Content-Type': 'application/json'}` をリクエストに必ず追加する |
| **SSLError / CertificateVerifyError** | 自己署名証明書による SSL 検証失敗 | スクリプトの例外メッセージを確認 | `verify=False` を `requests` 呼び出しに追加し、`urllib3.disable_warnings()` で警告を抑制する |
| **429 Too Many Requests** | API レート制限超過 | `response.headers.get('Retry-After')` を確認 | リクエストループ内に `time.sleep()` を挿入して送信間隔を調整する |


## ⚠ 制限事項

REST API の実装および運用における制限事項は以下の通りです。

* **FMC API セッション制限**: 同時ログイン可能な API セッション数には制限があり、不要になったトークンは明示的に Revoke（DELETE `/api/fmc_platform/v1/auth/revoke-token`）するか、タイムアウト（デフォルト30分）させる必要があります。
* **バルク（Bulk）処理の上限**: 単一の API リクエストで作成できるオブジェクト数やルールの数には上限（例: FMC では 1,000 件等）が存在します。大量のデータ投入時はスライス処理（バッチ処理）が必要です。
* **ISE ERS API の制限**: ISE ERS API は内部データベースへの直接的な CRUD 操作を提供しますが、ノードの再起動や認証ポリシーのグローバル変更など一部の低レイヤー設定はサポートされていません。
* **非同期処理の待機が必要な操作**: FMC のポリシー適用（Deployment）などは即時完了せず、タスク ID (Task ID) を受け取った後にタスクステータス API を定期的にポーリング（Polling）して完了を検知する必要があります。


---

## 🔄 他技術との関連

REST API はシスコのセキュリティエコシステム全体において、他の制御・監視技術と密接に連携します。

* **FMC Access Control Policy (ACP)**: REST API を介して動的オブジェクト（Network/Host/URL Object）を書き換えることで、防火壁のセキュリティポリシーを外部スクリプトからリアルタイムに変更可能。
* **Cisco ISE ERS / OpenAPI & Dynamic Quarantine**: 外部 SIEM や SOC 端末から ISE ERS API を呼び出し、不審な端末の MAC アドレスを特定グループに追加して ANC (Adaptive Network Control) ポリシーを即座に適用（Quarantine）。
* **Cisco SecureX / Threat Response**: 脅威インテリジェンスプラットフォームから REST API を経由して FMC/ISE/ESA/WSA へ指示を出し、IoC (Indicators of Compromise) である悪意のある IP/URL/Hash を一括ブロック。
* **Python Automation / CI/CD**: Git や Jenkins などの DevOps ツールチェーンと組み合わせることで、セキュリティ設定のコード化 (Security as Code) を実現。


---

## 🧩 比較表

### 1. HTTP Verbs (動作動詞) の機能比較表

| HTTP Action Verb | 操作内容 | 冪等性 (Idempotent) | 安全性 (Safe) | 主な成功ステータスコード | FMC/ISE での典型的な用途 |
| --- | --- | --- | --- | --- | --- |
| **GET** | リソース取得 | Yes | Yes | 200 OK | オブジェクト一覧・アクセスルール・システム情報の取得 |
| **POST** | リソース作成 / 認証 | No | No | 201 Created / 200 OK / 204 No Content | 新規オブジェクト追加、トークン発行、検索リクエスト |
| **PUT** | リソース置換 / 更新 | Yes | No | 200 OK / 204 No Content | 既存オブジェクトの全フィールド上書き更新 |
| **PATCH** | リソース部分更新 | No | No | 200 OK / 204 No Content | 既存ルールの特定の1項目（説明文等）のみを変更 |
| **DELETE** | リソース削除 | Yes | No | 200 OK / 204 No Content | 不要なオブジェクトやユーザーの削除、トークン破棄 |

### 2. データフォーマット比較: JSON vs XML

| 比較項目 | JSON (JavaScript Object Notation) | XML (Extensible Markup Language) |
| --- | --- | --- |
| **データ記述方式** | キー/値ペア、配列、ネスト構造（軽量） | タグベースの階層構造（冗長） |
| **人間の可読性** | 非常に高い（シンプルで読みやすい） | やや低い（閉じタグによる冗長性） |
| **パース処理速度** | 高速（軽量なためメモリ消費が少ない） | 遅い（DOM 解析などのオーバーヘッド） |
| **主要セキュリティ機器での採用** | Cisco FMC (完全採用), ISE ERS/OpenAPI, FDM, RESTCONF | Cisco ISE ERS (JSON/XML 両対応), 旧式 API (ASA REST API 等) |
| **Python での扱い** | `json.loads()`, `json.dumps()`, `requests.json()` で直接辞書型に変換可能 | `xml.etree.ElementTree` や `xmltodict` ライブラリでの変換が必要 |

### 3. 主要認証方式の比較表

| 認証方式 | 仕組み | トークン管理の必要性 | セキュリティレベル | Cisco 製品での採用例 |
| --- | --- | --- | --- | --- |
| **HTTP Basic Auth** | リクエスト毎に `Authorization: Basic Base64(user:pass)` を送信 | 不要 | 低（リクエスト毎にクレデンシャル送信） | FMC 認証トークン取得時、ISE ERS API |
| **Token-based (Custom Header)** | 初期認証で Base64 クレデンシャルを送信し、レスポンスヘッダーから Access Token を取得 | 必要（有効期限・リフレッシュ管理） | 高 | Cisco FMC (`X-auth-access-token`), Cisco FDM |
| **Bearer Token (OAuth 2.0)** | 認可サーバーから発行された JWT トークンを `Authorization: Bearer <token>` で送信 | 必要 | 非常に高 | Cisco ISE OpenAPI (v3.1+), SecureX |
| **Cookie/Session-based** | ログイン時にセットされた `Cookie: JSESSIONID=...` を維持して通信 | 必要 | 中 | Cisco APIC / ACI API |


---

## 💡 ベストプラクティス

1. **トークンの有効期限・自動リフレッシュ管理**: FMC のトークン有効期限は 30 分、リフレッシュ可能時間は 12 時間です。長時間を要する自動化スクリプトでは、タイマー判定または `401 Unauthorized` 検知時に自動で `refreshtoken` エラーハンドリングを実行する設計を組み込んでください。
2. **REST API 用専用管理アカウントの作成**: スクリプト用の認証アカウントには最低限の権限（RBAC: Role-Based Access Control）を付与し、個人管理者のアカウントと分離して監査追跡を容易にしてください。
3. **明示的なセッション終了 (Token Revocation)**: スクリプトの処理完了時（`finally` ブロック等）で明示的に DELETE リクエストを発行し、サーバー側のトークンセッションを解放してください。
4. **構造化エラーログの出力**: ステータスコードが `2xx` 以外の場合、単純に例外で停止させるのではなく、`response.json()` からエラーメッセージ・コード・影響を受けたフィールドを出力させてトラブルシューティングを迅速化してください。


import os

p6 = """---

### 演習6: Cisco ISE ERS API による特定 Endpoint の MAC アドレス検索とグループ割り当て (PUT)

#### 【問題】
特定 MAC アドレスのエンドポイントを ISE ERS API で検索し、そのエンドポイントの所属グループを変更（Quarantine）するスクリプトを作成してください。

#### 【要件】
* MAC アドレスから Endpoint ID を取得 (`GET /ers/config/endpoint?filter=mac.eq.<MAC>`)。
* 取得した ID を用いて `PUT /ers/config/endpoint/{id}` で `groupId` を更新すること。

#### 【設定例（Python スクリプト）】
```python
import json
import os
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

ISE_HOST = "10.0.0.1"
API_USER = os.getenv("API_USER", "user")
API_PASS = os.getenv("API_PASS", "pass")

BASE_URL = f"https://{ISE_HOST}:9060/ers/config"
headers = {"Content-Type": "application/json", "Accept": "application/json"}
auth = (API_USER, API_PASS)

grp_res = requests.get(f"{BASE_URL}/endpointgroup?filter=name.eq.Blacklist", auth=auth, headers=headers, verify=False)
blacklist_group_id = grp_res.json()['SearchResult']['resources'][0]['id']

target_mac = "11:22:33:44:55:66"
ep_res = requests.get(f"{BASE_URL}/endpoint?filter=mac.eq.{target_mac}", auth=auth, headers=headers, verify=False)

items = ep_res.json().get('SearchResult', {}).get('resources', [])
if items:
    endpoint_id = items[0]['id']
    ep_detail = requests.get(f"{BASE_URL}/endpoint/{endpoint_id}", auth=auth, headers=headers, verify=False).json()
    ep_detail['ERSEndpoint']['groupId'] = blacklist_group_id

    put_res = requests.put(f"{BASE_URL}/endpoint/{endpoint_id}", auth=auth, data=json.dumps(ep_detail), headers=headers, verify=False)
    if put_res.status_code == 200:
        print(f"[+] Endpoint {target_mac} Updated Successfully!")
```

#### 【検証方法】
スクリプトを実行し、`200 OK` が返され、対象エンドポイントの `groupId` が変更されることを確認します。
"""

with open("/workspace/scratch/test_p6.md", "w", encoding="utf-8") as f:
    f.write(p6)

print("Ex 6 written.")


import os

p7 = """---

### 演習7: トークン有効期限切れに伴う自動リフレッシュハンドラーの実装

#### 【問題】
FMC REST API 通信において、送信したリクエストが `401 Unauthorized` エラーで失敗した場合に、自動的に `refreshtoken` エンドポイントを呼び出してトークンを更新し、元のリクエストを再試行するラッパー関数スクリプトを作成してください。

#### 【要件】
* `FMCClient` クラスとして `request(method, endpoint, payload)` を実装。
* `401` 検知時に `POST /api/fmc_platform/v1/auth/refreshtoken` を実行。
* トークン再取得後、更新された `X-auth-access-token` で元のリクエストを再実行すること。

#### 【設定例（Python スクリプト）】
```python
import json
import os
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

class FMCClient:
    def __init__(self, fmc_host):
        self.fmc_host = fmc_host
        self.user = os.getenv("FMC_USER", "user")
        self.password = os.getenv("FMC_PASS", "pass")
        self.base_url = f"https://{fmc_host}/api"
        self.access_token = None
        self.refresh_token = None
        self.domain_uuid = None
        self.authenticate()

    def authenticate(self):
        url = f"{self.base_url}/fmc_platform/v1/auth/generatetoken"
        res = requests.post(url, auth=(self.user, self.password), verify=False)
        if res.status_code in [200, 204]:
            self.access_token = res.headers.get("X-auth-access-token")
            self.refresh_token = res.headers.get("X-auth-refresh-token")
            self.domain_uuid = res.headers.get("DOMAIN_UUID")

    def refresh_access_token(self):
        url = f"{self.base_url}/fmc_platform/v1/auth/refreshtoken"
        headers = {
            "X-auth-access-token": self.access_token,
            "X-auth-refresh-token": self.refresh_token
        }
        res = requests.post(url, headers=headers, verify=False)
        if res.status_code in [200, 204]:
            self.access_token = res.headers.get("X-auth-access-token")
            self.refresh_token = res.headers.get("X-auth-refresh-token")
            return True
        return False

    def request(self, method, endpoint, payload=None):
        url = f"{self.base_url}/fmc_config/v1/domain/{self.domain_uuid}/{endpoint}"
        headers = {
            "Content-Type": "application/json",
            "Accept": "application/json",
            "X-auth-access-token": self.access_token
        }
        
        data = json.dumps(payload) if payload else None
        res = requests.request(method, url, headers=headers, data=data, verify=False)

        if res.status_code == 401:
            if self.refresh_access_token():
                headers["X-auth-access-token"] = self.access_token
                res = requests.request(method, url, headers=headers, data=data, verify=False)

        return res

client = FMCClient("10.0.0.2")
resp = client.request("GET", "object/hosts")
print(f"[+] Result Status: {resp.status_code}")
```

#### 【検証方法】
意図的に古いトークンをセットして呼び出した際、`401` を検知して自動的に `refresh_access_token()` が実行されることを確認します。
"""

with open("/workspace/scratch/test_p7.md", "w", encoding="utf-8") as f:
    f.write(p7)

print("Ex 7 written.")


import os

p8 = """---

### 演習8: FMC ポリシーデプロイ (Deployment) のトリガーとタスク監視 (Polling)

#### 【問題】
FMC の保留中の変更が存在するデバイスに対してポリシーデプロイリクエスト (POST) を送信し、Task ID を定期的にポーリングして完了 (`SUCCEEDED`) まで待機するスクリプトを作成してください。

#### 【要件】
* `POST /api/fmc_config/v1/domain/{domain}/deployment/deployabledevices` でデプロイ対象デバイスを取得。
* `POST .../deployment/deploymentrequests` でデプロイ実行。
* `GET .../job/taskstatuses/{task_id}` を 10 秒間隔でループ実行し、`status` が `SUCCEEDED` になるまで監視すること。

#### 【設定例（Python スクリプト）】
```python
import os
import time
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

FMC_HOST = "10.0.0.2"
USER = os.getenv("FMC_USER", "user")
PASS = os.getenv("FMC_PASS", "pass")

res = requests.post(f"https://{FMC_HOST}/api/fmc_platform/v1/auth/generatetoken", auth=(USER, PASS), verify=False)
token = res.headers.get("X-auth-access-token")
domain = res.headers.get("DOMAIN_UUID")

headers = {"Content-Type": "application/json", "Accept": "application/json", "X-auth-access-token": token}

deployable_url = f"https://{FMC_HOST}/api/fmc_config/v1/domain/{domain}/deployment/deployabledevices"
dep_res = requests.get(deployable_url, headers=headers, verify=False).json()

device_list = dep_res.get('items', [])
if device_list:
    device_id = device_list[0]['device']['id']

    deploy_req_url = f"https://{FMC_HOST}/api/fmc_config/v1/domain/{domain}/deployment/deploymentrequests"
    deploy_payload = {
        "type": "DeploymentRequest",
        "version": str(int(time.time())),
        "deviceList": [device_id]
    }

    deploy_res = requests.post(deploy_req_url, json=deploy_payload, headers=headers, verify=False)
    task_id = deploy_res.json()['metadata']['task']['id']

    task_url = f"https://{FMC_HOST}/api/fmc_config/v1/domain/{domain}/job/taskstatuses/{task_id}"

    while True:
        task_res = requests.get(task_url, headers=headers, verify=False).json()
        status = task_res.get('status')
        if status == "SUCCEEDED":
            print("[+] Deployment Completed Successfully!")
            break
        elif status in ["FAILED", "CANCELLED"]:
            print(f"[-] Deployment Failed: {status}")
            break
        time.sleep(10)
```

#### 【検証方法】
10 秒ごとに進捗が表示され、最終的に `SUCCEEDED` で終了することを確認します。
"""

with open("/workspace/scratch/test_p8.md", "w", encoding="utf-8") as f:
    f.write(p8)

print("Ex 8 written.")


import os

p9 = """---

### 演習9: Cisco IOS-XE RESTCONF API による IP アドレス設定 (PATCH)

#### 【問題】
Cisco IOS-XE デバイスに対して RESTCONF API を使用し、GigabitEthernet2 インターフェイスの IP アドレスを更新し、有効化 (`no shutdown`) するスクリプトを作成してください。

#### 【要件】
* URL: `https://<IOS_XE_IP>/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet2`
* ヘッダー: `Content-Type: application/yang-data+json`
* 認証: HTTP Basic

#### 【設定例（Python スクリプト）】
```python
import json
import os
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

ROUTER_HOST = "10.0.0.3"
USER = os.getenv("ROUTER_USER", "user")
PASS = os.getenv("ROUTER_PASS", "pass")

url = f"https://{ROUTER_HOST}/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet2"

headers = {
    "Content-Type": "application/yang-data+json",
    "Accept": "application/yang-data+json"
}

payload = {
    "ietf-interfaces:interface": {
        "name": "GigabitEthernet2",
        "description": "Configured via RESTCONF",
        "type": "iana-if-type:ethernetCsmacd",
        "enabled": True,
        "ietf-ip:ipv4": {
            "address": [
                {
                    "ip": "192.168.20.1",
                    "netmask": "255.255.255.0"
                }
            ]
        }
    }
}

response = requests.patch(
    url,
    auth=(USER, PASS),
    data=json.dumps(payload),
    headers=headers,
    verify=False
)

if response.status_code in [200, 204]:
    print("[+] RESTCONF Interface Update Successful!")
```

#### 【検証方法】
ルーター CLI にて `show ip interface brief GigabitEthernet2` を実行し、`192.168.20.1` が設定され `up/up` 状態になっていることを確認します。
"""

with open("/workspace/scratch/test_p9.md", "w", encoding="utf-8") as f:
    f.write(p9)

print("Ex 9 written.")


import os

p10 = """---

### 演習10: XML ペイロードを使用した API リクエスト処理と Python による XML パース

#### 【問題】
Cisco ISE ERS API に対して `Accept: application/xml` ヘッダーを指定して XML 形式でネットワークデバイス一覧を取得し、Python の `xml.etree.ElementTree` モジュールを用いて全デバイスの `name` 属性を抽出して出力するスクリプトを作成してください。

#### 【要件】
* XML 形式でレスポンスを受信すること。
* `xml.etree.ElementTree.fromstring(response.text)` を使用して XML ノードをパースすること。

#### 【設定例（Python スクリプト）】
```python
import os
import requests
import xml.etree.ElementTree as ET
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

ISE_HOST = "10.0.0.1"
USER = os.getenv("ISE_USER", "user")
PASS = os.getenv("ISE_PASS", "pass")

url = f"https://{ISE_HOST}:9060/ers/config/networkdevice"

headers = {
    "Accept": "application/xml"
}

response = requests.get(
    url,
    auth=(USER, PASS),
    headers=headers,
    verify=False
)

if response.status_code == 200:
    root = ET.fromstring(response.text)
    for elem in root.iter():
        if 'name' in elem.attrib:
            print(f"    - Resource: {elem.attrib['name']}")
```

#### 【検証方法】
スクリプトを実行し、XML ツリーからリソース名が抽出されてリスト出力されることを確認します。
"""

with open("/workspace/scratch/test_p10.md", "w", encoding="utf-8") as f:
    f.write(p10)

print("Ex 10 written.")


import os

p5 = """## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験およびテクノロジー記述問題を意識した想定試験問題を 5 問掲載します。

---

### 問題1 (コンフィグ・スクリプト読解)
以下の Python スクリプトを実行した際、FMC 上で発生する処理として最も適切なものを1つ選べ。

```python
import json, requests, urllib3
urllib3.disable_warnings()

headers = {"Content-Type": "application/json", "Accept": "application/json", "X-auth-access-token": token}
url = f"https://{fmc_host}/api/fmc_config/v1/domain/{domain}/object/networkgroups/{group_id}"

payload = {
    "id": group_id,
    "name": "Grp_Internal",
    "type": "NetworkGroup",
    "objects": [{"id": "00000000-0000-0000-0000-000000000000", "type": "Host"}]
}

r = requests.put(url, data=json.dumps(payload), headers=headers, verify=False)
```

A) `group_id` で指定された Network Group に新しい Host オブジェクトが差分追加される。  
B) `group_id` で指定された Network Group の全メンバーが破棄され、ペイロードで指定された 1 つの Host オブジェクトのみに上書き更新される。  
C) `PUT` メソッドは新規作成専用であるため、エラーが返却される。  
D) `Content-Type` ヘッダーが不適切であるためエラーが返却される。

**【正解】** B  
**【解説】** HTTP `PUT` メソッドは既存リソースの「完全置換（Replace）」を行います。差分追加ではなく、ペイロードに含まれる `objects` リストでグループ全体が上書きされる点に注意してください。

---

### 問題2 (トラブルシューティング)
Cisco ISE ERS API を用いて新しい Endpoint を登録するため `POST` リクエストを送信したところ、ステータスコード `415 Unsupported Media Type` が返却された。この障害の最も可能性の高い原因はどれか。

A) 認証資格情報の誤り。  
B) リクエストヘッダーに `Content-Type: application/json` が含まれていない。  
C) 指定した MAC アドレスが既に存在している。  
D) ERS API が無効化されている。

**【正解】** B  
**【解説】** ステータスコード `415 Unsupported Media Type` は、クライアントが送信したリクエストボディのデータ形式（Content-Type）をサーバーが理解またはサポートできない場合に発生します。

---

### 問題3 (Design & セキュリティ)
REST API を使用したセキュリティ運用自動化スクリプトの設計において、セキュリティおよび信頼性の観点から最も推奨されるプラクティスはどれか。

A) トークンをコード内にハードコードし永続利用する。  
B) SSL/TLS 証明書の検証（`verify=True`）を常に有効化し、適切な CA 証明書バンドルを指定して Man-in-the-Middle 攻撃を防止する。  
C) トークンの無効化処理は自動的に行われるため、スクリプト終了時の明示的な DELETE リクエストは省略する。  
D) HTTP メソッドにはすべての処理において `POST` のみを使用する。

**【正解】** B  
**【解説】** 本番環境および高セキュリティ設計においては、`verify=True` を指定してサーバーの証明書を検証することがインフラへの攻撃を防ぐベストプラクティスです。

---

### 問題4 (実装)
Cisco FMC REST API において、設定の変更（オブジェクト作成など）を行った後、その変更を実際のプラットフォームデバイス（FTD）へ適用するために呼び出すべき正確な API エンドポイントのカテゴリーはどれか。

A) `/object/hosts`  
B) `/deployment/deploymentrequests`  
C) `/auth/refreshtoken`  
D) `/policy/accesspolicies`

**【正解】** B  
**【解説】** FMC でオブジェクトやルールを変更しただけでは FTD デバイスには適用されません。`/deployment/deploymentrequests` エンドポイントへ POST リクエストを送信してデプロイメントジョブを発行する必要があります。

---

### 問題5 (レスポンスパース)
Python の `requests` ライブラリを使用して `response = requests.delete(url, headers=headers, verify=False)` を実行し、レスポンスステータスコードが `204 No Content` であった場合、`data = response.json()` を呼び出すとどのような挙動になるか。

A) 空の辞書オブジェクト `{}` が返却される。  
B) `None` が返却される。  
C) レスポンスボディが存在しないため JSON パース例外が発生する。  
D) 自動的に再認証が行われる。

**【正解】** C  
**【解説】** `204 No Content` は成功応答ですが、レスポンスボディが完全に空であることを意味します。空の文字列に対して `response.json()` を実行すると JSON パースエラーが発生します。


---

## 🔗 参考リソース

Cisco 公式の API レファレンスおよびガイド一覧です。

* Cisco DevNet: Firepower Management Center REST API Quick Start Guide
* Cisco ISE ERS API Reference Guide
* Cisco IOS-XE RESTCONF API Configuration Guide
* RFC 7231: Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content
* RFC 8259: The JavaScript Object Notation (JSON) Data Interchange Format
* RFC 8040: RESTCONF Protocol


---

## 📝 補足（Notes）

### API トラブルシューティングチェックリスト
1. **URI の正確性**: `domain_uuid` や `object_id` が URI パスに含まれているか。
2. **ヘッダーの網羅性**: `Content-Type: application/json` および `Accept: application/json` が両方設定されているか。
3. **認証トークン**: トークン取得時の `X-auth-access-token` が毎リクエストに正しく付与されているか。
4. **ステータスコード判定**: `200` (OK), `201` (Created), `204` (No Content) の違いを正しく認識してコード内で条件分岐しているか。
5. **例外処理**: ネットワーク不達やタイムアウト、JSON パースエラーに対する `try-except` ブロックが適切か。
"""

with open("/workspace/scratch/test_questions.md", "w", encoding="utf-8") as f:
    f.write(p5)

print("Part 5 written.")


