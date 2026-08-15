---
layout: default
title: 3.9.a-REST-API
nav_order: 1
parent: 3.9-APIs
grand_parent: 3.0-Security-Infrastructure
---

# 3.9.a REST API requests and responses

REST (Representational State Transfer) API は、現代のセキュリティ運用（SecOps）において、Cisco FMC、ISE、DNA Center などのデバイスやコントローラを自動管理するための標準的なインターフェイスです。CCIE Security v6.1 では、単なる知識だけでなく、Python の `requests` ライブラリ等を用いて実際に設定を取得・変更する実装力が求められます。

---

## 📘 概要

*   **機能概要**: HTTP/HTTPS プロトコルを利用し、リソース（設定情報やステータス）に対して一貫した操作を行うアーキテクチャです。
*   **利用目的**: 大量のリソース（ACL、オブジェクト、エンドポイント）の一括作成、外部システムとの連携、構成のバックアップ、およびポリシーの動的変更。
*   **利用場面**:
    *   **FMC API**: 数百のファイアウォールルールをプログラムでデプロイする。
    *   **ISE API**: 特定の属性を持つエンドポイントを隔離（Quarantine）する。
    *   **DNAC API**: ネットワーク全体のインベントリ情報を抽出し、レポートを作成する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **通信プロトコル** | 主に HTTPS (TCP 443)。ステートレスな通信。 |
| **主要コンポーネント** | エンドポイント (URL)、HTTP 動詞、ヘッダー、ボディ (Payload)。 |
| **データ形式** | **JSON** (標準的) または **XML**。 |
| **認証方式** | Basic 認証、トークンベース (X-Auth-Token)、OAuth2。 |
| **メリット** | 言語に依存せず、人間にも可読性が高い (特に JSON)。 |
| **制限事項** | レート制限 (API Throttling) や、セッションタイムアウトの考慮が必要。 |

---

## 🏗 動作原理

Python スクリプト（クライアント）が API サーバ（FMC/ISE 等）に対してリクエストを送信し、サーバがステータスコードとデータを返します。

```text
[ Python Client ]
      |
      |-- (1) Request: URL + Verb + Auth Header + JSON Payload -->
      |                                                           |
      |                                               [ Security Controller (Server) ]
      |                                                           |
      |<-- (2) Response: Status Code (e.g., 200 OK) + JSON Data --|
```

---

## ⚙ 動作シーケンス

1.  **Authentication**: 認証情報を送信し、以降の操作に使用する一時的な **Token** を取得します。
2.  **Resource Identification**: 操作対象を URL（エンドポイント）で指定します。
3.  **Action Selection**: HTTP 動詞（GET, POST 等）を選択して実行します。
4.  **Payload Processing**: 必要に応じて JSON 形式でデータを送受信します。
5.  **Validation**: 返ってきたステータスコード（200, 201, 401 等）を確認し、処理の成否を判断します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Python `requests` の習熟**: `requests.post()`, `requests.get()` の引数に `headers`, `auth`, `json`, `verify=False` を正しく指定できる必要があります。
*   **FMC ドメイン UUID の取得**: FMC API では操作にドメイン ID が必須となるため、認証直後にドメイン一覧を取得するステップが基本となります。
*   **JSON パース能力**: Python の辞書型 (`response.json()`) から、目的の `id` や `name` をループで抽出する処理が頻出です。
*   **認証トークンの再利用**: 一度のログインでトークンを保持し、後続のリクエストヘッダーに `X-Auth-Token` としてセットする流れをマスターしてください。
*   **ステータスコードの理解**: `201 Created`（作成成功）と `200 OK`（取得・更新成功）の違い、および `401/403` のエラー対処。

---

## 🛠 設定方法

### Python スクリプトによる基本リクエスト例 (FMC 認証)
```python
import requests
import json

# 自己署名証明書の警告を無視
requests.packages.urllib3.disable_warnings()

url = "https://fmc.example.com/api/fmc_platform/v1/auth/generatetoken"
auth = ('admin', 'Cisco123')

# (iii) Authentication: Basic Auth を用いて POST
response = requests.post(url, auth=auth, verify=False)

if response.status_code == 204:
    # (i) Headers: レスポンスヘッダーからトークンを抽出
    access_token = response.headers.get('X-auth-access-token')
    domain_uuid = response.headers.get('domain_uuid')
    print(f"Token: {access_token}")
else:
    print(f"Error: {response.status_code}")
```

---

## 🔍 検証コマンド

| 目的 | 手法 |
| :--- | :--- |
| **API 接続性の確認 (CLI)** | <code>curl -k -u user:pass https://[IP]/...</code> |
| **ステータスコード表示** | <code>print(response.status_code)</code> |
| **JSON ボディの確認** | <code>print(json.dumps(response.json(), indent=4))</code> |
| **FMC API エクスプローラ** | ブラウザで <code>https://[FMC_IP]/api/api-explorer/</code> を開く。 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| `401 Unauthorized` | 認証情報の誤り、またはトークン期限切れ。 | ユーザー名/パスワードおよびトークンの取得し直し。 |
| `403 Forbidden` | API ユーザーの権限不足（RBAC）。 | FMC/ISE 側で API 操作権限を持つロールを付与。 |
| `404 Not Found` | URL (Endpoint) のスペルミス。 | API Explorer で正しいパスを確認。 |
| `SSL: CERTIFICATE_VERIFY_FAILED` | 自己署名証明書。 | `requests` の引数に <code>verify=False</code> を追加。 |

---

## ⚠ 制限事項

*   **同時セッション数**: FMC 等では API の同時接続数に制限がある場合があります。
*   **Content-Type**: ボディを送る際は `Content-Type: application/json` ヘッダーが必須です。
*   **データの不整合**: PUT 操作時に一部の必須フィールドが欠けていると、エラーになるか意図しない上書きが発生します。

---

## 🔄 他技術との関連

*   **3.10 Cisco DNAC API**: インベントリ取得やサイト管理に使用。
*   **Access Control Policy**: FMC API を用いた動的なルール追加。
*   **ISE pxGrid**: API を通じて取得した脅威情報を ISE へフィードバックし、端末を隔離。

---

## 🧩 比較表

### HTTP 操作アクション (Verbs)

| Verb | 操作 | 説明 |
| :--- | :--- | :--- |
| **GET** | Read | リソースの取得。サーバの状態は変化しない。 |
| **POST** | Create | 新しいリソースの作成。 |
| **PUT** | Update/Replace | 既存リソースの完全な置き換え（更新）。 |
| **PATCH** | Partial Update | 既存リソースの一部のみ変更。 |
| **DELETE** | Remove | リソースの削除。 |

---

## 💡 ベストプラクティス

1.  **エラーハンドリング**: `response.raise_for_status()` を使用して、エラー時にスクリプトを停止させます。
2.  **API Explorer の活用**: スクリプトを書く前に、ブラウザ上のツールでペイロードの構造を確認します。
3.  **秘匿情報の管理**: パスワードはスクリプトにハードコードせず、環境変数や `getpass` モジュールを使用します。

---

## 📝 ラボ学習・設定サンプル例（3.9 API / Python 実装）

### 1. FMC: 認証トークンの取得とドメインUUIDの抽出
*   **問題**: FMC API にアクセスするための認証トークンを取得し、後続の操作に必要なドメイン UUID を表示するスクリプトを作成せよ。
*   **要件**:
    1.  `requests` ライブラリを使用する。
    2.  Basic 認証を用いて `POST` リクエストを送信。
    3.  レスポンスヘッダーから `X-auth-access-token` を取得。
*   **設定例**:
    ```python
    import requests
    import json

    # 警告の無効化
    requests.packages.urllib3.disable_warnings()

    fmc_ip = "10.1.1.10"
    auth_url = f"https://{fmc_ip}/api/fmc_platform/v1/auth/generatetoken"
    auth = ("admin", "Cisco123")

    # POSTリクエストの実行
    response = requests.post(auth_url, auth=auth, verify=False)

    if response.status_code == 204:
        token = response.headers.get("X-auth-access-token")
        domain_uuid = response.headers.get("domain_uuid")
        print(f"Token: {token}")
        print(f"Domain UUID: {domain_uuid}")
    else:
        print(f"Auth Failed: {response.status_code}")
    ```
*   **検証方法**: スクリプトを実行し、ターミナルに英数字のトークンと UUID が出力されることを確認する。

### 2. FMC: ネットワークオブジェクトの全件取得
*   **問題**: FMC に登録されている全ての「ネットワークオブジェクト」の名前と ID を取得してリスト化せよ。
*   **要件**: 取得した JSON データのリスト構造をループで回し、特定のフィールド（name, id）のみを表示する。
*   **設定例**:
    ```python
    # 認証トークン(token)とdomain_uuidが取得済みである前提
    headers = {"X-Auth-Token": token, "Content-Type": "application/json"}
    obj_url = f"https://{fmc_ip}/api/fmc_config/v1/domain/{domain_uuid}/object/networks"

    resp = requests.get(obj_url, headers=headers, verify=False)
    objects = resp.json().get("items", [])

    for obj in objects:
        print(f"Name: {obj['name']}, ID: {obj['id']}")
    ```
*   **検証方法**: FMC GUI 上の Objects ページと、スクリプトの出力結果が一致することを確認する。

### 3. FMC: 新規ネットワークオブジェクトの作成 (POST)
*   **問題**: 名前 `Internal_LAN`、アドレス `192.168.10.0/24` のネットワークオブジェクトをプログラム経由で作成せよ。
*   **要件**: JSON 形式のペイロードを `POST` リクエストの `json` 引数に含める。
*   **設定例**:
    ```python
    payload = {
        "name": "Internal_LAN",
        "type": "Network",
        "value": "192.168.10.0/24"
    }

    resp = requests.post(obj_url, headers=headers, json=payload, verify=False)

    if resp.status_code == 201:
        print("Successfully Created Object")
    else:
        print(f"Failed: {resp.text}")
    ```
*   **検証方法**: ステータスコード `201` が返ることを確認し、FMC GUI でオブジェクトが存在することを確認する。

### 4. DNAC: インベントリ情報の取得 (REST API)
*   **問題**: DNA Center (10.1.1.20) から、全てのネットワークデバイスのホスト名とシリアル番号を抽出せよ。
*   **要件**: `X-Auth-Token` を使用してデバイスリストを取得する。
*   **設定例**:
    ```python
    dnac_ip = "10.1.1.20"
    # トークン取得はFMCと異なるエンドポイントだがここではヘッダー適用のみ記述
    headers = {"X-Auth-Token": dnac_token, "Content-Type": "application/json"}
    device_url = f"https://{dnac_ip}/dna/intent/api/v1/network-device"

    response = requests.get(device_url, headers=headers, verify=False)
    devices = response.json()['response']

    for dev in devices:
        print(f"Hostname: {dev['hostname']}, Serial: {dev['serialNumber']}")
    ```
*   **検証方法**: `show network-device` API の結果が正しくパースされ、デバイス一覧が表示されるか確認する。

### 5. ISE: 外部RESTfulサービス(ERS)によるエンドポイント確認
*   **問題**: ISE の ERS API を使用して、MACアドレス `00:50:56:AB:CD:EF` を持つエンドポイントが存在するか確認せよ。
*   **要件**: `Accept: application/json` ヘッダーを必須とする。
*   **設定例**:
    ```python
    ise_ip = "10.1.1.30"
    mac = "00:50:56:AB:CD:EF"
    ers_url = f"https://{ise_ip}:9060/ers/config/endpoint/name/{mac}"
    headers = {"Accept": "application/json"}
    auth = ("api_user", "Cisco123")

    resp = requests.get(ers_url, headers=headers, auth=auth, verify=False)

    if resp.status_code == 200:
        print(f"Endpoint Found: {resp.json()['ERSEndpoint']['name']}")
    else:
        print("Endpoint Not Found")
    ```
*   **検証方法**: 存在する MAC アドレスと存在しない MAC アドレスで実行し、分岐が正しく動作するか確認する。

### 6. 複数オブジェクトの一括登録 (Bulk Import)
*   **問題**: 辞書形式のリストに含まれる複数の IP アドレスを一括で FMC に登録せよ。
*   **要件**: `for` ループを使用して `POST` を繰り返す。
*   **設定例**:
    ```python
    ip_list = [
        {"name": "Server1", "value": "10.1.1.1/32"},
        {"name": "Server2", "value": "10.1.1.2/32"}
    ]

    for item in ip_list:
        payload = {"name": item["name"], "type": "Network", "value": item["value"]}
        r = requests.post(obj_url, headers=headers, json=payload, verify=False)
        print(f"Creating {item['name']}... Status: {r.status_code}")
    ```
*   **検証方法**: 実行後に FMC GUI で `Server1`, `Server2` が作成されていることを確認する。

### 7. ステータスコードによる条件付きエラー処理
*   **問題**: API リクエストが失敗した場合、ステータスコードに応じた日本語のエラーメッセージを出力する処理を実装せよ。
*   **設定例**:
    ```python
    resp = requests.get(url, headers=headers, verify=False)
    
    if resp.status_code == 401:
        print("エラー: 認証に失敗しました。トークンを確認してください。")
    elif resp.status_code == 403:
        print("エラー: 権限がありません。管理者ロールを確認してください。")
    elif resp.status_code == 404:
        print("エラー: リソースが見つかりません。URLが正しいか確認してください。")
    ```
*   **検証方法**: 意図的に間違ったパスワードや URL を入力し、対応するメッセージが出るか確認する。

### 8. JSON データの特定フィールド更新 (PUT)
*   **問題**: 既存のネットワークオブジェクト `Internal_LAN` の説明文（description）を `Updated by Python` に変更せよ。
*   **要件**: オブジェクトの ID を指定して `PUT` リクエストを送信する。
*   **設定例**:
    ```python
    obj_id = "OBJECT-UUID-1234-5678"  # 事前に取得したID
    update_url = f"{obj_url}/{obj_id}"
    payload = {
        "name": "Internal_LAN",
        "type": "Network",
        "value": "192.168.10.0/24",
        "description": "Updated by Python"
    }

    resp = requests.put(update_url, headers=headers, json=payload, verify=False)
    print(f"Status: {resp.status_code}")
    ```
*   **検証方法**: FMC GUI でオブジェクトを開き、Description フィールドが更新されていることを確認する。

### 9. クエリパラメータを使用したフィルタリング
*   **問題**: FMC API の `limit` パラメータを使用して、最初の 5 件のオブジェクトのみを取得せよ。
*   **設定例**:
    ```python
    params = {"limit": 5}
    resp = requests.get(obj_url, headers=headers, params=params, verify=False)
    
    items = resp.json().get("items", [])
    print(f"Total items received: {len(items)}")
    ```
*   **検証方法**: 出力されるアイテム数が 5 以下であることを確認する。

### 10. FMC API 接続タイムアウトの実装
*   **問題**: ネットワーク遅延やデバイスダウンを考慮し、5 秒以内に応答がない場合に例外を発生させるスクリプトを作成せよ。
*   **設定例**:
    ```python
    try:
        resp = requests.get(obj_url, headers=headers, timeout=5, verify=False)
        resp.raise_for_status()
    except requests.exceptions.Timeout:
        print("Timeout Error: FMC is not responding.")
    except requests.exceptions.HTTPError as err:
        print(f"HTTP Error: {err}")
    ```
*   **検証方法**: 無効な IP アドレスを指定して実行し、"Timeout Error" が表示されるか確認する。

---


## ❓ 想定試験問題

1.  **コンフィグ読解**: `requests.put(url, data=payload, headers=headers)` でエラーが出る。ペイロードが辞書型の場合、修正箇所は？
    *   **回答**: `data=payload` を `json=payload` に変更するか、`json.dumps(payload)` を使用する。
2.  **トラブルシュート**: FMC API でリソース取得はできるが、作成（POST）が `403` で失敗する。何を確認すべきか？
    *   **回答**: API ユーザーに割り当てられた **アクセス権限（Admin Role）** が、Write 操作を許可しているか確認する。
3.  **Design**: XML と JSON の両方が選択可能な場合、Python で扱う際により適した形式とその理由は？
    *   **回答**: **JSON**。Python の辞書型と構造が酷似しており、標準の `json` ライブラリで容易に相互変換できるため。
4.  **コンフィグ読解**: `headers = {"X-Auth-Token": token}` を GET リクエストに含める目的は？
    *   **回答**: サーバーに対し、ログイン済みの有効なセッションであることを証明するため。
5.  **実装**: ラボ試験で「SSL 検証エラーを回避して API を叩け」と指示された場合のオプションは？
    *   **回答**: `verify=False` を `requests` のメソッドに追加する。

---

## 🔗 参考リソース

*   **Cisco DevNet**: [FMC REST API Quick Start](https://developer.cisco.com/site/firepower/fmc-api/)
*   **Cisco Configuration Guide**: [REST API - Cisco DNA Center](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/dna-center/2-2-2/user-guide/b_cisco_dna_center_ug_2_2_2.html)
*   **Official Cert Guide**: CCNP and CCIE Security Core SCOR 350-701
*   **Cisco DNA Center API**: [DNAC Intent API Reference](https://developer.cisco.com/docs/dna-center/api/2-2-2/)
*   **ISE API Guide**: [ISE External RESTful Services (ERS)](https://www.cisco.com/c/en/us/td/docs/security/ise/3-1/api-guide/b_ise_api_guide_3_1.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: API ラボの問題は「まず認証」「次にリソース特定」「最後にアクション」の 3 ステップで考えます。
*   **注意点**: ラボ試験で使用する `requests.post()` や `requests.get()` の引数に **`verify=False`** を入れ忘れると、自己署名証明書エラーでスクリプトが停止してしまいます。
*   **図解**: 
    - `response.json()` → JSON を Python の**辞書/リスト**に変換
    - `json.dumps(data)` → Python オブジェクトを **JSON 文字列**に変換（表示用）
