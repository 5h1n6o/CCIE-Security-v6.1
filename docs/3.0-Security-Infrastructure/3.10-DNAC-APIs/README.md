---
layout: default
title: 3.10-DNAC-APIs
nav_order: 10
parent: 3.0-Security-Infrastructure
---

# 3.10 Cisco DNAC Northbound APIs use cases

## 📘 概要

Cisco DNA Center (Catalyst Center) の Northbound API (Intent API) は、ネットワーク管理者がコントロールプレーンおよびマネジメントプレーンと対話し、エンタープライズネットワーク全体におけるインフラストラクチャの自動化、アシュアランス (Assurance) 情報の取得、ネットワークデザイン、インベントリ管理、およびポリシー制御プログラミングを実行するための主要なインターフェースです。

RESTful アーキテクチャに基づいて設計されており、HTTP/HTTPS 上で標準的な JSON ペイロードをやり取りします。DNA Center Northbound API を活用することで、手動の CLI 操作や個別の Web GUI 操作から脱却し、エンタープライズ規模のセキュリティポリシー定義、SDA (Software-Defined Access) ファブリック制御、IP ギャップ検出、脅威検知・自動封じ込め (pxGrid / ISE 連携)、ネットワーク機器のゼロタッチプロビジョニング (PnP) などを Python 等のスクリプトや外部オーケストレーションツール (ServiceNow, Ansible など) からプログラムによって統合的に実行可能となります。

CCIE Security v6.1 ラボ試験および実務において、DNA Center Northbound API は「認証トークンの取得」「ネットワークインベントリ情報の収集」「サイトおよびデバイス設定の取得・適用」「アシュアランス・健康度 (Health Score) の監視」「SDA ファブリック内のセキュリティグループ (SGT) およびアクセス制御ポリシーの自動設定」というユースケースで重点的に問われます。

---

## 🔑 要点

Cisco DNA Center Northbound API (Intent API) の主な特徴、用途、利点、制限事項、および設計上の考慮事項を下表に示します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | RESTful HTTP API (GET, POST, PUT, DELETE)。HTTPS (443) 上で動作し、JSON ペイロードを採用。Token ベース認証 (HTTP Basic 認証でエンコードした `username:password` を送信し `x-auth-token` を取得) |
| **用途** | ネットワークインベントリ自動収集、SDA セグメンテーション/SGT/VN ポリシー自動構成、アシュアランス/クライアント・デバイス Health Score 監視、PnP (Plug and Play) プロビジョニング、Command Runner (CLI 実行) |
| **メリット** | 単一のコントローラ API を通じてネットワーク全体の自動化が可能。ISE 連携による ID/ポリシー同期。標準 JSON 構造により外部 SIEM/ITSM ツールとの連携が容易 |
| **デメリット** | API レート制限 (Rate Limit / Throttling) が存在。DNAC バージョン間の API エンドポイント URI やスキーマの変更。非同期タスク (Task API) のポーリング処理実装が必要 |
| **対応機種** | Cisco DNA Center (Catalyst Center) アプライアンス (2.2.x, 2.3.x 等)。管理対象: Catalyst 9000, ISR/ASR/Catalyst 8000, AireOS/9800 WLC 等 |
| **制限事項** | 同時アクティブ API セッション数の上限。非同期処理実行時のレスポンスとして直接結果ではなく `taskId` が返されるため、Task API (`/dna/intent/api/v1/task/{taskId}`) によるステータス確認が必須 |
| **設計上の注意点** | トークンの有効期限 (デフォルト1時間) に合わせた自動リフレッシュ設計。エラーハンドリング (401, 403, 429, 500) とバックオフロジックの実装。自己署名証明書使用時の SSL 検証バイパス |

---

## 🏗 動作原理

Cisco DNA Center Northbound API は、管理端末 (Python スクリプト、SIEM/ITSM ツール等) と DNA Center コントローラ間の RESTful 通信モデルによって動作します。

```
[ Management Application / Python Script ]
               │
               │ 1. POST /dna/system/api/v1/auth/token (Basic Auth Header)
               ▼
[ Cisco DNA Center (Northbound API Gateway) ]
               │
               │ 2. Validate Credentials & Generate Auth Token
               ▼
[ DNA Center Auth Subsystem ]
               │
               │ 3. Return {"Token": "eyJhbGciOiJI..."}
               ▼
[ Management Application / Python Script ]
               │
               │ 4. GET /dna/intent/api/v1/network-device (x-auth-token Header)
               ▼
[ Cisco DNA Center (Intent API Subsystem) ]
               │
               │ 5. Execute Query / Retrieve Data from Database
               ▼
[ DNA Center Database / Core Controller Engine ]
               │
               │ 6. Southbound Protocols (SSH / NETCONF / RESTCONF / SNMP / LISP)
               ▼
[ Network Devices (Switches / Routers / WLCs) ]
```

### コアメカニズム

1. **Token ベース認証とセッション管理**:
   * API リクエストの第1歩として、`/dna/system/api/v1/auth/token` エンドポイントに対し、ユーザー名とパスワードをエンコードした `Authorization: Basic` ヘッダーを付与して POST リクエストを送信します。
   * DNAC はレスポンスとして 1 時間有効な認証トークン文字列 (`x-auth-token`) を返します。以降のすべての Intent API リクエストでは、HTTP ヘッダーに `x-auth-token: <Token>` を設定してアクセスします。

2. **同期処理 (Synchronous) と非同期処理 (Asynchronous Task API)**:
   * **同期 API**: 情報取得系 (GET リクエスト) の多くは同期的に即時レスポンスを返します (例: ネットワークデバイス一覧取得)。
   * **非同期 API**: 状態変更系 (POST, PUT, DELETE) や時間のかかる処理 (Command Runner 実行、テンプレート適用、デバイス追加等) では、HTTP ステータス 202 Accepted とともに `taskId` を含む JSON が返されます。スクリプトは `/dna/intent/api/v1/task/{taskId}` を繰り返しポーリングし、`progress` や `isError` フィールドを確認して完了を検知します。

3. **Domain & API Categories**:
   * **Authentication API**: トークン取得・管理。
   * **Know Your Network (Devices & Topology)**: インベントリ、トポロジー、インターフェース情報取得。
   * **Site Management**: サイト階層 (Area, Building, Floor) の作成・管理。
   * **Connectivity / Fabric (SDA)**: 仮想ネットワーク (VN)、セキュリティグループ (SGT)、ファブリックノード、アクセス制御ポリシーの設定。
   * **Assurance**: デバイス Health、クライアント Health、イシュー (Issue)、ネットワークテレメトリの取得。
   * **Command Runner**: ネットワーク機器への CLI コマンド (`show` コマンド等) のリモート一括実行。

---

## ⚙ 動作シーケンス

DNA Center Northbound API を使用してネットワーク操作およびデータ収集を行う際の、完全なパケットおよびリクエスト処理シーケンスは以下の通りです。

```
[ Application / Script ]             [ DNAC API Gateway ]            [ DNAC Services / DB ]           [ Network Device ]
          │                                   │                                │                         │
          │ ── 1. POST /auth/token ─────────> │                                │                         │
          │    (Basic Auth)                   │ ── 2. Authenticate User ────>  │                         │
          │ <── 3. 200 OK (x-auth-token) ──── │                                │                         │
          │                                   │                                │                         │
          │ ── 4. GET /network-device ──────> │                                │                         │
          │    (x-auth-token)                 │ ── 5. Query Inventory ──────>  │                         │
          │ <── 6. 200 OK (Device JSON) ───── │                                │                         │
          │                                   │                                │                         │
          │ ── 7. POST /command-runner ─────> │                                │                         │
          │    (x-auth-token, CLI payload)    │ ── 8. Register Task ─────────> │                         │
          │ <── 9. 202 Accepted (taskId) ──── │                                │                         │
          │                                   │                                │ ── 10. Execute CLI ───> │
          │                                   │                                │ <── 11. Return CLI Output│
          │ ── 12. GET /task/{taskId} ──────> │                                │                         │
          │ <── 13. 200 OK (In Progress) ──── │                                │                         │
          │ ── 14. GET /task/{taskId} ──────> │                                │                         │
          │ <── 15. 200 OK (SUCCESS + FileId) │                                │                         │
          │                                   │                                │                         │
          │ ── 16. GET /file/{fileId} ──────> │                                │                         │
          │ <── 17. 200 OK (CLI Output JSON)─ │                                │                         │
```

### シーケンス詳細手順

1. **認証トークン要求**: クライアントは POST `/dna/system/api/v1/auth/token` にて認証を要求。
2. **トークン発行**: DNAC はクレデンシャルを検証し、`Token` フィールドに文字列を格納した JSON を返却。
3. **同期情報取得**: クライアントは `x-auth-token` ヘッダーを伴って GET `/dna/intent/api/v1/network-device` を実行。DNAC は全デバイスの IP、MAC、モデル、IOS バージョン、シリアル番号、Reachability 状態等のリストを即座に返却。
4. **非同期タスク要求 (Command Runner 例)**: クライアントは POST `/dna/intent/api/v1/network-device-poller/cli/read-request` にて実行したい CLI コマンド (`["show ip interface brief", "show version"]`) と対象デバイス ID のリストを送信。
5. **Task ID 受領**: DNAC はリクエストを受理し、HTTP 202 とともに `taskId` を返却。
6. **Task ポーリング**: クライアントは GET `/dna/intent/api/v1/task/{taskId}` を定期実行。`endTime` が設定され `isError: false` となるまでループ監視。
7. **結果取得**: Command Runner の場合、Task のレスポンスに含まれる `fileId` を使用して GET `/dna/intent/api/v1/file/{fileId}` を呼び出し、各デバイスでの CLI 実行結果文字列を取得・表示。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、Cisco DNA Center (Catalyst Center) Northbound API に関する問は、自動化・プログラミングセクション (Automation and Programmability) またはセキュリティ可視化・アシュアランスセクションで出題される可能性があります。

### Blueprintで重要なポイント

* **Token 認証処理の正確な実装**: HTTP Basic 認証によるトークン要求において、ヘッダーに `Content-Type: application/json` を含め、レスポンスの JSON から `Token` キー（または `x-auth-token`）を正しく抽出できること。
* **ヘッダー名の厳密性**: Intent API の呼び出し時に `x-auth-token` (小文字・ハイフン区切り) を使用すること。API によっては Authorization ヘッダーと混同しやすいため注意が必要。
* **Task API による非同期ハンドリング**: 単に POST リクエストを送信して終わるのではなく、返却された `taskId` を使用してループ処理 (While loop / sleep) を行い、完了結果を取得・確認する処理コードを記述できること。
* **フィルタリングおよびクエリパラメータの使用**: `/network-device` エンドポイント等で、`managementIpAddress` や `serialNumber` などのクエリパラメータを指定して目的のデバイス情報のみを絞り込む技術。
* **JSON パースと必要なフィールドの抽出**: デバイスの `id`, `hostname`, `managementIpAddress`, `reachabilityStatus`, `softwareVersion` などを抽出して特定の形式で整形出力する能力。

### ラボ試験で設定させられそうな内容

1. **DNAC 認証スクリプトの作成**: 指定された IP アドレス、ユーザー名、パスワードを用いて Token を取得し、標準出力または変数に保持する処理。
2. **特定のネットワーク機器情報の抽出**: 管理 IP やデバイス名に基づいてデバイス ID を特定し、そのインターフェース一覧や Neighbor 情報を取得するスクリプトの記述。
3. **Command Runner を使用した一括セキュリティ検証**: 複数台のスイッチに対して `show port-security` や `show ip dhcp snooping` を一括実行し、結果を集約表示する処理。
4. **クライアントアシュアランス・健康度の集計**: 全クライアントの Health Score を取得し、閾値 (例: Health Score < 50) を下回るクライアントの MAC アドレスと接続 AP 名をリスティングする処理。
5. **エラー処理とレスポンス検証**: レスポンスコードが 200/202 以外の場合に例外処理 (`requests.exceptions.HTTPError`) を行い、ログにエラーメッセージを出力する堅牢な実装。

### よくある設定ミス・落とし穴

* **証明書検証エラー (SSL Verification Failure)**: ラボ環境の DNAC は自己署名証明書を使用していることが多く、`requests.get(..., verify=False)` を指定しないと `SSLError` が発生してスクリプトが停止する。
* **非標準ポート・パス誤り**: DNAC の Northbound API エンドポイント URI スラッシュ位置やバージョン指定 (`v1`) の入力ミス。
* **トークン切れの放置**: 長時間実行されるスクリプトにおいて、1 時間のトークン有効期限切れによる HTTP 401 Unauthorized エラーへの対策不足。
* **Command Runner 取得時の Task API スキップ**: Command Runner の POST レスポンスには実行結果そのものは含まれず、`taskId` のみが含まれている。Task が `SUCCESS` になってから `fileId` を使って `/file/{fileId}` を叩かないと結果文字列が取得できない。

---

## 🛠 設定方法

DNA Center Northbound API を Python スクリプトから利用する際の実践的な実装コード例を示します。

### 1. DNAC 認証トークン取得スクリプト (`get_dnac_token.py`)

```python
import requests
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_IP = "192.168.10.100"
DNAC_USER = "admin"
DNAC_PASS = "SecretPass123"

def get_auth_token(dnac_ip, username, password):
    url = f"https://{dnac_ip}/dna/system/api/v1/auth/token"
    headers = {
        "Content-Type": "application/json"
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
        print(f"[+] Successfully generated Auth Token")
        return token
    except requests.exceptions.RequestException as e:
        print(f"[-] Failed to obtain token: {e}")
        return None

if __name__ == "__main__":
    token = get_auth_token(DNAC_IP, DNAC_USER, DNAC_PASS)
```

### 2. インベントリ情報取得と抽出スクリプト (`get_inventory.py`)

```python
import requests
import urllib3
import json

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_IP = "192.168.10.100"
TOKEN = "YOUR_AUTH_TOKEN_HERE"

def get_network_devices(dnac_ip, token):
    url = f"https://{dnac_ip}/dna/intent/api/v1/network-device"
    headers = {
        "Content-Type": "application/json",
        "x-auth-token": token
    }
    
    response = requests.get(url, headers=headers, verify=False)
    if response.status_code == 200:
        devices = response.json().get("response", [])
        print(f"\n{'Hostname':<25} {'IP Address':<15} {'Model':<20} {'Status':<12}")
        print("-" * 75)
        for dev in devices:
            hostname = dev.get("hostname", "N/A")
            mgmt_ip = dev.get("managementIpAddress", "N/A")
            model = dev.get("platformId", "N/A")
            status = dev.get("reachabilityStatus", "N/A")
            print(f"{hostname:<25} {mgmt_ip:<15} {model:<20} {status:<12}")
    else:
        print(f"[-] Error: {response.status_code} - {response.text}")

if __name__ == "__main__":
    get_network_devices(DNAC_IP, TOKEN)
```

### 3. 非同期 Task ポーリングおよび Command Runner 実行スクリプト (`run_command.py`)

```python
import requests
import time
import urllib3
import json

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_IP = "192.168.10.100"
TOKEN = "YOUR_AUTH_TOKEN_HERE"
DEVICE_ID = "3f8a42c1-8e9b-4c12-a890-123456789abc"

def wait_for_task(dnac_ip, token, task_id, timeout=60):
    url = f"https://{dnac_ip}/dna/intent/api/v1/task/{task_id}"
    headers = {"x-auth-token": token}
    
    start_time = time.time()
    while time.time() - start_time < timeout:
        res = requests.get(url, headers=headers, verify=False)
        if res.status_code == 200:
            task_info = res.json().get("response", {})
            if task_info.get("endTime"):
                if task_info.get("isError"):
                    print(f"[-] Task failed: {task_info.get('progress')}")
                    return None
                print(f"[+] Task completed successfully.")
                return task_info
        time.sleep(3)
    print("[-] Task timed out.")
    return None

def execute_cli_command(dnac_ip, token, device_id, commands):
    url = f"https://{dnac_ip}/dna/intent/api/v1/network-device-poller/cli/read-request"
    headers = {
        "Content-Type": "application/json",
        "x-auth-token": token
    }
    payload = {
        "commands": commands,
        "deviceUuids": [device_id]
    }
    
    res = requests.post(url, json=payload, headers=headers, verify=False)
    if res.status_code == 202:
        task_id = res.json().get("response", {}).get("taskId")
        print(f"[+] Command task submitted. Task ID: {task_id}")
        
        task_result = wait_for_task(dnac_ip, token, task_id)
        if task_result:
            progress = task_result.get("progress")
            print(f"[+] Task Progress/Output Info: {progress}")
            file_id = json.loads(progress).get("fileId") if "fileId" in progress else None
            if file_id:
                file_url = f"https://{dnac_ip}/dna/intent/api/v1/file/{file_id}"
                file_res = requests.get(file_url, headers=headers, verify=False)
                print("\n=== Command Output ===")
                print(file_res.json())
    else:
        print(f"[-] Failed to submit command: {res.status_code} - {res.text}")

if __name__ == "__main__":
    execute_cli_command(DNAC_IP, TOKEN, DEVICE_ID, ["show ip interface brief"])
```


## 🔍 検証コマンド

DNA Center Northbound API の通信・ステータス検証、DNAC アプライアンス CLI での API ログ確認コマンドの一覧です。

| 目的 | コマンド / エンドポイント |
| -- | --------------------- |
| API 認証テスト (cURL) | <code>curl -k -X POST https://<DNAC_IP>/dna/system/api/v1/auth/token -u "admin:password"</code> |
| デバイスインベントリ取得 | <code>curl -k -X GET https://<DNAC_IP>/dna/intent/api/v1/network-device -H "x-auth-token: <TOKEN>"</code> |
| タスク状態確認 | <code>curl -k -X GET https://<DNAC_IP>/dna/intent/api/v1/task/<TASK_ID> -H "x-auth-token: <TOKEN>"</code> |
| API サービス状態確認 (DNAC CLI) | <code>magctl service status | grep intent-api</code> |
| API ログのリアルタイム確認 (DNAC CLI) | <code>magctl service logs -f intent-api-gateway</code> |
| Nginx 逆プロキシログ確認 (DNAC CLI) | <code>tail -f /var/log/nginx/access.log</code> |

---

## 🚨 トラブルシュート

DNA Center Northbound API 利用時によく遭遇する問題、原因、確認方法、および対処方法を下表にまとめます。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| -- | -- | ------ | ---- |
| **HTTP 401 Unauthorized** | トークン未設定、無効、または有効期限切れ (1時間経過) | API レスポンス `{"error": "Unauthorized"}` | POST `/dna/system/api/v1/auth/token` を再実行し、最新の `x-auth-token` をヘッダーに設定 |
| **HTTP 403 Forbidden** | ユーザー権限不足 (OBSERVER 権限で POST 呼び出し等) | DNAC GUI ユーザー管理画面確認 | API を呼び出すユーザーに `NETWORK-ADMIN` または `SUPER-ADMIN` 権限を付与 |
| **HTTP 404 Not Found** | API エンドポイント URI の記述誤り (バージョン `v1` 抜け等) | API ドキュメント参照 | リクエスト URI パスを正確に記述 (`/dna/intent/api/v1/...`) |
| **HTTP 429 Too Many Requests** | API レート制限 (Throttling) の超過 | レスポンスヘッダー `Retry-After` 確認 | スクリプト内に Exponential Backoff (指数バックオフ) と `time.sleep()` を導入して送信頻度を抑制 |
| **HTTP 500 Internal Server Error** | DNAC 内部サービス障害または無効な POST ペイロード構造 | `magctl service logs -f intent-api` | ペイロードの JSON 構造・データ型を公式 API スキーマと照合。DNAC サービスを再起動 |
| **Task API が <code>isError: true</code>** | 機器への接続不可、コマンド構文誤り、タイムアウト | Task の `progress` フィールド確認 | 管理対象デバイスの Reachability (SSH/SNMP) 状態を確認。実行コマンドの構文を検証 |

---

## ⚠ 制限事項

* **トークン寿命の制限**: 発行された `x-auth-token` は 60 分（1 時間）で無効化されます。長期間動作するデーモンや大量バッチ処理スクリプトでは、トークンの有効期限を管理するか定期的な自動再取得ロジックが必須です。
* **API レート制限 (Throttling)**: 短時間に大量の API リクエストを連続送信すると、DNAC の API Gateway が HTTP 429 エラーを返却して接続を強制制限します。
* **非同期タスクのタイムアウト**: タスクの実行完了までに最大数分を要する場合があります。スクリプト側で適切なタイムアウト値 (例: 120 秒) とループ間隔を設定しないと、無限ループまたは早期中断が発生します。
* **バージョン依存性**: DNA Center 1.3.x, 2.1.x, 2.2.x, 2.3.x で API エンドポイントのパスや JSON スキーマが変更されている場合があります (例: `/dna/intent/api/v1` への統一)。

---

## 🔄 他技術との関連

Cisco DNA Center Northbound API は、ネットワークセキュリティおよび運用自動化を構成する以下の各種技術と密接に関連します。

* **Cisco ISE (Identity Services Engine)**: DNAC は pxGrid および REST API を介して ISE と北向き・南向きに連携し、SGT (Security Group Tag)、ポリシーマトリクス、エンドポイント識別情報を相互同期します。 Northbound API を使用して DNAC 側から SGT 割り当てを制御可能。
* **SDA (Software-Defined Access)**: Northbound API を通じて、仮想ネットワーク (VN: Virtual Network) や Scalable Group ACL (SGACL) のプログラミングを実施。
* **ServiceNow / ITSM 統合**: ネットワークイシューやアシュアランスの警告が発生した際、DNAC Northbound Webhook / API が ServiceNow に自動インシデント起票を実施。
* **SIEM / Telemetry 統合 (Splunk, ELK)**: アシュアランスデータやクライアント Health データを API 経由で定期取得し、外部 SIEM へ転送してログ集約。

---

## 🧩 比較表

Cisco ネットワークコントローラの主要 API インターフェースの比較を下表に示します。

| 比較項目 | Cisco DNA Center Intent API | Cisco ISE ERS / OpenAPI | Cisco FMC REST API |
| --- | --- | --- | --- |
| **主な用途** | キャンパス網自動化、アシュアランス、SDA セグメンテーション | ID 認証、TACACS+/RADIUS ポリシー、PXGrid 連携 | ファイアウォールポリシー、NGIPS、アクセス制御 |
| **認証方式** | HTTP Basic 認証 ──> `x-auth-token` (1時間) | HTTP Basic 認証 / Bearer Token | HTTP Basic 認証 ──> `X-auth-access-token` |
| **主要ペイロード** | JSON | JSON / XML | JSON |
| **非同期タスク** | Task API (`/task/{taskId}`) を多用 | 一部バッチ処理のみ | TaskStatus API (`/taskstatus/{id}`) を使用 |
| **イベント通知** | Webhook / Event REST API | Syslog / pxGrid | eStreamer / Syslog / Webhook |

---

## 💡 ベストプラクティス

1. **クラス構造による API セッションの共通化**: トークンの保持・自動更新、ヘッダー付与、エラーハンドリングをカプセル化した Python クラス (`DNACClient`) を作成し、コードの再利用性を高める。
2. **バックオフとリトライの実装**: HTTP 429 や一時的な 503 エラーに対して、`urllib3.util.retry.Retry` や `tenacity` ライブラリを使用して自動リトライを行う。
3. **環境変数による資格情報の保護**: ユーザー名やパスワードを Python コード内にハードコードせず、`os.environ.get("DNAC_PASSWORD")` や `.env` ファイルから読み込む。
4. **Task API のヘルパー化**: 非同期レスポンス処理を行う汎用関数を1つ作成し、すべての POST/PUT リクエストで使い回す。


## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験を想定した、Cisco DNA Center Northbound API を用いた実用的な Python スクリプトおよび設定演習サンプル 10 選です。コードは省略せず完全に記述しています。

---

### 演習サンプル 1: 認証トークン自動リフレッシュ付き API クラスの構築

#### 問題
DNAC との通信を行う際、トークンの期限切れ (1時間) を自動的に検知して再取得を行う再利用可能な Python クラスを実装しなさい。

#### 要件
* クラス名: `DNACClient`
* 初期化時に DNAC IP, Username, Password を受け取る。
* API 呼び出し毎にトークンの有無を確認し、未取得または失敗時に自動再認証を行う。
* `get()` および `post()` メソッドを提供する。

#### 設定例 (Python)

```python
import requests
import time
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

class DNACClient:
    def __init__(self, dnac_ip, username, password):
        self.dnac_ip = dnac_ip
        self.username = username
        self.password = password
        self.base_url = f"https://{dnac_ip}/dna"
        self.token = None
        self.token_time = 0
        self.headers = {"Content-Type": "application/json"}

    def authenticate(self):
        url = f"{self.base_url}/system/api/v1/auth/token"
        try:
            res = requests.post(
                url,
                auth=(self.username, self.password),
                headers=self.headers,
                verify=False,
                timeout=10
            )
            res.raise_for_status()
            self.token = res.json().get("Token")
            self.token_time = time.time()
            self.headers["x-auth-token"] = self.token
            print(f"[+] Token obtained successfully")
            return True
        except Exception as e:
            print(f"[-] Authentication failed: {e}")
            return False

    def ensure_token(self):
        if not self.token or (time.time() - self.token_time > 3000):
            return self.authenticate()
        return True

    def get(self, endpoint, params=None):
        if not self.ensure_token():
            return None
        url = f"{self.base_url}/intent/api/v1/{endpoint.lstrip('/')}"
        res = requests.get(url, headers=self.headers, params=params, verify=False)
        return res

    def post(self, endpoint, payload=None):
        if not self.ensure_token():
            return None
        url = f"{self.base_url}/intent/api/v1/{endpoint.lstrip('/')}"
        res = requests.post(url, headers=self.headers, json=payload, verify=False)
        return res

if __name__ == "__main__":
    client = DNACClient("192.168.10.100", "admin", "SecretPass123")
    res = client.get("network-device")
    if res and res.status_code == 200:
        print(f"[+] Device Count: {len(res.json().get('response', []))}")
```

#### 検証方法
スクリプトを実行し、正常に Token が生成され、ネットワークデバイス数が取得できることを確認する。

---

### 演習サンプル 2: 管理 IP からのデバイス UUID 動的検索

#### 問題
指定された管理 IP アドレス (例: `10.1.10.1`) を入力とし、該当するネットワーク機器の `id` (UUID) および `platformId` を取得するスクリプトを作成しなさい。

#### 要件
* クエリパラメータ `managementIpAddress` を使用してフィルタリングする。
* 該当デバイスが存在しない場合は適切なエラーメッセージを表示する。

#### 設定例 (Python)

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def get_device_id_by_ip(dnac_ip, token, target_ip):
    url = f"https://{dnac_ip}/dna/intent/api/v1/network-device"
    headers = {
        "Content-Type": "application/json",
        "x-auth-token": token
    }
    params = {
        "managementIpAddress": target_ip
    }
    
    res = requests.get(url, headers=headers, params=params, verify=False)
    if res.status_code == 200:
        devices = res.json().get("response", [])
        if devices:
            dev = devices[0]
            dev_id = dev.get("id")
            platform = dev.get("platformId")
            hostname = dev.get("hostname")
            print(f"[+] Found Device: Hostname={hostname}, Model={platform}, UUID={dev_id}")
            return dev_id
        else:
            print(f"[-] No device found with IP {target_ip}")
            return None
    else:
        print(f"[-] Error querying device: {res.status_code} - {res.text}")
        return None

TOKEN = "YOUR_TOKEN"
get_device_id_by_ip("192.168.10.100", TOKEN, "10.1.10.1")
```

#### 検証方法
指定管理 IP を持つスイッチの UUID が取得できることを確認する。

---

### 演習サンプル 3: デバイスの物理/論理インターフェース一覧の取得

#### 問題
特定のネットワークデバイス UUID を指定し、そのデバイスの全インターフェースの「ポート名」「ステータス (UP/DOWN)」「VLAN」を取得し一覧表示しなさい。

#### 要件
* エンドポイント `/interface/network-device/{deviceId}` を使用する。
* ポート名、Operational Status、VLAN 名を綺麗に整列表示する。

#### 設定例 (Python)

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def get_device_interfaces(dnac_ip, token, device_id):
    url = f"https://{dnac_ip}/dna/intent/api/v1/interface/network-device/{device_id}"
    headers = {"x-auth-token": token}
    
    res = requests.get(url, headers=headers, verify=False)
    if res.status_code == 200:
        interfaces = res.json().get("response", [])
        print(f"\n{'Port Name':<20} {'Status':<12} {'Admin Status':<12} {'VLAN/Speed':<15}")
        print("-" * 60)
        for port in interfaces:
            name = port.get("portName", "N/A")
            status = port.get("status", "N/A")
            admin_status = port.get("adminStatus", "N/A")
            vlan = port.get("vlanId", "N/A")
            print(f"{name:<20} {status:<12} {admin_status:<12} VLAN:{vlan:<10}")
    else:
        print(f"[-] Failed to fetch interfaces: {res.status_code}")

TOKEN = "YOUR_TOKEN"
DEVICE_ID = "YOUR_DEVICE_UUID"
get_device_interfaces("192.168.10.100", TOKEN, DEVICE_ID)
```

#### 検証方法
指定デバイスのポート一覧およびステータスが表示されることを確認する。

---

### 演習サンプル 4: サイト階層 (Sites) の取得とフィルタリング

#### 問題
DNAC に登録されているすべてのサイト (Global / Area / Building / Floor) の一覧を取得し、Building の名前と属性を表示しなさい。

#### 要件
* エンドポイント `/site` を使用する。
* サイトのタイプが `building` であるものを抽出する。

#### 設定例 (Python)

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def get_buildings(dnac_ip, token):
    url = f"https://{dnac_ip}/dna/intent/api/v1/site"
    headers = {"x-auth-token": token}
    
    res = requests.get(url, headers=headers, verify=False)
    if res.status_code == 200:
        sites = res.json().get("response", [])
        print(f"\n{'Site Name':<35} {'Site Type':<15} {'Site ID':<30}")
        print("-" * 80)
        for site in sites:
            site_type = site.get("type", "")
            if site_type == "building" or "building" in site.get("additionalInfo", [{}])[0].get("namespace", ""):
                name = site.get("siteNameHierarchy", site.get("name"))
                site_id = site.get("id")
                print(f"{name:<35} {'Building':<15} {site_id:<30}")
    else:
        print(f"[-] Failed to get sites: {res.status_code}")

TOKEN = "YOUR_TOKEN"
get_buildings("192.168.10.100", TOKEN)
```

#### 検証方法
DNAC 上で定義された Building サイトの一覧が出力されることを確認。

---

### 演習サンプル 5: Command Runner による複数スイッチへの CLI 一括実行

#### 問題
指定された 2 台のスイッチに対し、Command Runner API を使用して `show ip dhcp snooping` を実行し、その出力を表示しなさい。

#### 要件
* エンドポイント `/network-device-poller/cli/read-request` を使用。
* 非同期 Task API による結果待機処理を実装する。

#### 設定例 (Python)

```python
import requests
import time
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def run_cli_commands(dnac_ip, token, device_ids, commands):
    url = f"https://{dnac_ip}/dna/intent/api/v1/network-device-poller/cli/read-request"
    headers = {
        "Content-Type": "application/json",
        "x-auth-token": token
    }
    payload = {
        "commands": commands,
        "deviceUuids": device_ids
    }
    
    res = requests.post(url, json=payload, headers=headers, verify=False)
    if res.status_code != 202:
        print(f"[-] Command submission failed: {res.status_code}")
        return
        
    task_id = res.json().get("response", {}).get("taskId")
    print(f"[+] Task created: {task_id}. Polling status...")
    
    task_url = f"https://{dnac_ip}/dna/intent/api/v1/task/{task_id}"
    file_id = None
    for _ in range(20):
        t_res = requests.get(task_url, headers=headers, verify=False)
        if t_res.status_code == 200:
            t_data = t_res.json().get("response", {})
            if t_data.get("endTime"):
                progress = t_data.get("progress", "")
                print(f"[+] Task Finished. Progress info: {progress}")
                try:
                    prog_json = json.loads(progress)
                    file_id = prog_json.get("fileId")
                except:
                    file_id = progress
                break
        time.sleep(3)
        
    if file_id:
        file_url = f"https://{dnac_ip}/dna/intent/api/v1/file/{file_id}"
        f_res = requests.get(file_url, headers=headers, verify=False)
        print("\n=== CLI Execution Output ===")
        print(json.dumps(f_res.json(), indent=2))

TOKEN = "YOUR_TOKEN"
DEVICES = ["uuid-switch-1", "uuid-switch-2"]
run_cli_commands("192.168.10.100", TOKEN, DEVICES, ["show ip dhcp snooping"])
```

#### 検証方法
実行後、対象スイッチの DHCP Snooping 設定状態が JSON として返却・表示されることを確認する。

---

### 演習サンプル 6: アシュアランス・クライアント Health Score 監視

#### 問題
無線・有線クライアントの全体健康度 (Client Health) を取得し、Overall Health スコアを表示するスクリプトを作成しなさい。

#### 要件
* エンドポイント `/client-health` を使用する。
* クライアントカテゴリごとの Health スコアおよび総クライアント数を取得・計算する。

#### 設定例 (Python)

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def get_client_health_summary(dnac_ip, token):
    url = f"https://{dnac_ip}/dna/intent/api/v1/client-health"
    headers = {"x-auth-token": token}
    
    res = requests.get(url, headers=headers, verify=False)
    if res.status_code == 200:
        health_data = res.json().get("response", [])
        print(f"\n{'Category':<15} {'Overall Health':<15} {'Total Clients':<15} {'Poor Health Count':<18}")
        print("-" * 65)
        for item in health_data:
            cat = item.get("siteId", "Global/Category")
            score = item.get("scoreDetail", [{}])[0].get("scoreCategory", {}).get("scoreValue", "N/A")
            total = item.get("clientCount", {}).get("total", 0)
            poor = item.get("clientCount", {}).get("poor", 0)
            print(f"{cat:<15} {str(score):<15} {total:<15} {poor:<18}")
    else:
        print(f"[-] Failed to get Client Health: {res.status_code}")

TOKEN = "YOUR_TOKEN"
get_client_health_summary("192.168.10.100", TOKEN)
```

#### 検証方法
クライアントの健康度サマリー情報が出力されることを確認する。

---

### 演習サンプル 7: 接続不可 (UNREACHABLE) デバイスの自動検出スクリプト

#### 問題
DNAC に登録されているすべてのネットワーク機器のうち、`reachabilityStatus` が `UNREACHABLE` になっている機器を特定し、警告表示するスクリプトを作成しなさい。

#### 要件
* 全デバイス一覧を取得し、ループ内でステータスを評価する。
* 障害機器の ホスト名、IP、シリアル番号、最終疎通確認時間を表示する。

#### 設定例 (Python)

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def check_unreachable_devices(dnac_ip, token):
    url = f"https://{dnac_ip}/dna/intent/api/v1/network-device"
    headers = {"x-auth-token": token}
    
    res = requests.get(url, headers=headers, verify=False)
    if res.status_code == 200:
        devices = res.json().get("response", [])
        unreachable = [d for d in devices if d.get("reachabilityStatus") == "UNREACHABLE"]
        
        print(f"\n[!] Audit Complete: Found {len(unreachable)} Unreachable Device(s).")
        if unreachable:
            print(f"{'Hostname':<25} {'IP Address':<15} {'Serial Number':<18} {'Up Time':<15}")
            print("-" * 75)
            for d in unreachable:
                print(f"{d.get('hostname'):<25} {d.get('managementIpAddress'):<15} {d.get('serialNumber'):<18} {d.get('upTime', 'N/A'):<15}")
    else:
        print(f"[-] API Error: {res.status_code}")

TOKEN = "YOUR_TOKEN"
check_unreachable_devices("192.168.10.100", TOKEN)
```

#### 検証方法
疎通不可な機器が検出され、適切なアラート情報が出力されることを確認する。

---

### 演習サンプル 8: 新しい Building サイトの自動作成

#### 問題
DNAC の Northbound API を使用して、指定された Parent Site 配下に新しい Building サイト (`Tokyo-HQ`) を作成するスクリプトを記述しなさい。

#### 要件
* エンドポイント `/site` に対する POST リクエスト。
* `type: "building"` および緯度・経度属性を含める。

#### 設定例 (Python)

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def create_building_site(dnac_ip, token, parent_name, building_name, address):
    url = f"https://{dnac_ip}/dna/intent/api/v1/site"
    headers = {
        "Content-Type": "application/json",
        "x-auth-token": token
    }
    payload = {
        "type": "building",
        "site": {
            "building": {
                "name": building_name,
                "address": address,
                "parentName": parent_name,
                "latitude": 35.6812,
                "longitude": 139.7671
            }
        }
    }
    
    res = requests.post(url, json=payload, headers=headers, verify=False)
    if res.status_code in [200, 202]:
        task_id = res.json().get("executionId") or res.json().get("taskId")
        print(f"[+] Site creation initiated. Task/Execution ID: {task_id}")
    else:
        print(f"[-] Site creation failed: {res.status_code} - {res.text}")

TOKEN = "YOUR_TOKEN"
create_building_site("192.168.10.100", TOKEN, "Global", "Tokyo-HQ", "Tokyo, Japan")
```

#### 検証方法
リクエスト完了後、DNAC GUI の Hierarchy 画面または GET `/site` にて新サイトが作成されていることを確認。

---

### 演習サンプル 9: アシュアランス Open Issues の取得と重要度フィルタリング

#### 問題
ネットワーク全体で発生している現在アクティブなイシュー (Network Issues) を取得し、Priority が P1 (CRITICAL) のイシューのみを抽出して表示しなさい。

#### 要件
* エンドポイント `/issues` を使用。
* `priority: "P1"` をフィルタリング条件とする。

#### 設定例 (Python)

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def get_critical_issues(dnac_ip, token):
    url = f"https://{dnac_ip}/dna/intent/api/v1/issue"
    headers = {"x-auth-token": token}
    params = {"priority": "P1"}
    
    res = requests.get(url, headers=headers, params=params, verify=False)
    if res.status_code == 200:
        issues = res.json().get("response", [])
        print(f"\n[!] Critical Issues Count: {len(issues)}")
        print(f"{'Issue ID':<20} {'Entity/Device':<25} {'Description':<40}")
        print("-" * 85)
        for issue in issues:
            issue_id = issue.get("issueId", "N/A")[:18]
            entity = issue.get("name", "N/A")[:23]
            desc = issue.get("description", "N/A")[:38]
            print(f"{issue_id:<20} {entity:<25} {desc:<40}")
    else:
        print(f"[-] Failed to fetch issues: {res.status_code}")

TOKEN = "YOUR_TOKEN"
get_critical_issues("192.168.10.100", TOKEN)
```

#### 検証方法
P1 イシューが正しくフィルタリングされて表示されることを確認。

---

### 演習サンプル 10: Task API 非同期進捗監視用ラッパー関数の実装

#### 問題
任意の非同期 API 呼び出しで返された `taskId` を引数に取り、Task が完了するまで標準出力にプログレス表示を行いながら待機する汎用関数を実装しなさい。

#### 要件
* タイムアウト制限 (デフォルト 90 秒) を設ける。
* エラー発生時 (`isError: True`) は例外 `RuntimeError` を発生させる。

#### 設定例 (Python)

```python
import requests
import time
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def poll_dnac_task(dnac_ip, token, task_id, timeout=90, interval=3):
    url = f"https://{dnac_ip}/dna/intent/api/v1/task/{task_id}"
    headers = {"x-auth-token": token}
    
    start_time = time.time()
    print(f"[*] Waiting for Task ID {task_id} to complete...")
    
    while time.time() - start_time < timeout:
        res = requests.get(url, headers=headers, verify=False)
        if res.status_code == 200:
            task_data = res.json().get("response", {})
            end_time = task_data.get("endTime")
            is_error = task_data.get("isError", False)
            progress = task_data.get("progress", "In progress...")
            
            if end_time:
                if is_error:
                    failure_reason = task_data.get("failureReason", progress)
                    raise RuntimeError(f"Task Failed: {failure_reason}")
                print(f"[+] Task Completed Successfully! Progress: {progress}")
                return task_data
                
            print(f"    - Status: {progress}")
        else:
            print(f"    - Warning: Failed to query task status ({res.status_code})")
            
        time.sleep(interval)
        
    raise TimeoutError(f"Task {task_id} did not complete within {timeout} seconds.")
```

#### 検証方法
任意の非同期タスクに対して関数を呼び出し、プログレスログが表示されて正常終了/例外発生することを検証する。


## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験および記述・実践形式を意識した想定問題です。

---

### 問題 1 (コード読解)

DNAC Northbound API にアクセスする Python コードで、認証ヘッダーの指定誤りにより 401 エラーが発生しました。原因と修正方法を説明しなさい。

#### 正解・解説
DNAC Intent API では、トークン認証ヘッダーとして `x-auth-token` を使用する必要があります。一般的な `Authorization: Bearer` では認証に失敗します。

---

### 問題 2 (トラブルシューティング)

Command Runner API の呼び出し直後に `/file/{fileId}` エンドポイントにアクセスしたところ 404 エラーが発生しました。メカニズムの観点から原因を述べよ。

#### 正解・解説
Command Runner は非同期タスク処理を行うため、POST 直後には出力ファイルが生成されていません。`taskId` を用いてタスク完了状態を確認した後に `fileId` にアクセスする必要があります。

---

### 問題 3 (Design)

大量の機器情報を API 経由で並列取得する際、レート制限 (HTTP 429) 回避のための設計考慮点を述べよ。

#### 正解・解説
指数バックオフ (Exponential Backoff) の導入、同時接続セッション数のスロットリング、および一括取得 (Bulk) エンドポイントの活用が推奨されます。


### 問題 4 (実装)

DNAC Northbound API からネットワーク機器のソフトウェアバージョンを取得し、不一致デバイスを抽出するロジックを説明しなさい。

#### 正解・解説
`GET /dna/intent/api/v1/network-device` から得られるリストをループ処理し、各要素の `softwareVersion` キーの値をターゲットバージョンと比較して抽出します。

---

### 問題 5 (コンフィグ/コード読解)

`/dna/intent/api/v1/client-health` エンドポイントで取得したデータから不調なクライアントを特定する条件を述べよ。

#### 正解・解説
`scoreCategory` 内の `scoreValue` が 50 未満 (Poor Health) に設定されている項目をフィルタリングすることで特定します。

---

### 問題 6 (トラブルシューティング)

API トークン取得後に POST リクエストが HTTP 403 で拒否される原因と対策を述べよ。

#### 正解・解説
アカウントの RBAC ロール権限不足が原因です。DNAC GUI にて当該ユーザーに NETWORK-ADMIN などの適切な書き込み権限を付与します。

---

## 🔗 参考リソース

Cisco DNA Center (Catalyst Center) Northbound API に関する Cisco 公式ドキュメントおよび技術資料の一覧です。

* Cisco Catalyst Center Platform Intent API Reference
* Cisco DNA Center Authentication API Guide
* Cisco Catalyst Center Automation and Assurance API Developer Guide
* Cisco Live BRKDEV-2011: Hands-on Cisco DNA Center Intent API Automation
* Cisco DevNet Learning Labs: Cisco DNA Center REST API with Python

---

## 📝 補足（Notes）

* **x-auth-token の必須化**: DNAC Intent API では、トークン指定ヘッダーとして小文字の `x-auth-token` が標準採用されています。
* **非同期処理の待機**: Lab 試験で Python スクリプト作成が出題された場合、POST 操作後の `taskId` のポーリング処理が正しく実装されているかが自動採点チェックポイントとなります。
* **urllib3 警告無効化**: `verify=False` を使用する際は、`urllib3.disable_warnings()` を記述して出力を整えてください。


