---
layout: default
title: 3.9-APIs
nav_order: 9
parent: 3.0-Security-Infrastructure
---

# 3.9 Interaction with network devices through APIs using basic Python scripts

ネットワークプログラマビリティと自動化は、現代のセキュリティインフラストラクチャ管理において不可欠なスキルです。CCIE Security v6.1 では、**REST API** を使用して Cisco FMC、Cisco ISE、Cisco DNA Center などのデバイスやコントローラと対話し、Python スクリプトを用いて設定の取得、変更、監視を自動化する能力が問われます。

---

## 📘 概要

*   **機能概要**: HTTP ベースの REST (Representational State Transfer) アーキテクチャを使用して、ネットワークデバイスやセキュリティ管理プラットフォームのノースバウンド API にアクセスします。
*   **利用目的**: 大規模なポリシー変更の一括実行、定期的なステータスチェックの自動化、および CI/CD パイプラインとの統合。
*   **どのような場面で利用するか**: 
    *   **FMC (Firepower Management Center)**: 数百のオブジェクトやルールを Python 経由で作成・更新する。
    *   **ISE (Identity Services Engine)**: 外部 RESTful サービス (ERS) を使用したエンドポイントの管理。
    *   **DNA Center**: ネットワークデバイスのインベントリ情報の抽出やプロビジョニングの自動化。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **通信プロトコル** | HTTP/HTTPS (主に TCP 443)。 |
| **REST の特徴** | ステートレス（Stateless）、キャッシュ可能、統一インターフェイス。 |
| **認証方式** | Basic Auth, Token-based Auth (Bearer Token), OAuth2。 |
| **データ形式** | JSON, XML, YAML。 |
| **Python ライブラリ** | `requests`（API 呼び出し）、`json`（解析）、`yaml`（解析）。 |
| **HTTP 動詞** | GET, POST, PUT, PATCH, DELETE。 |

---

## 🏗 動作原理

Python スクリプトは「API クライアント」として動作し、ネットワークデバイス（API サーバー）に対して HTTP リクエストを送信します。

```text
[ Python Script (Client) ]
      |
      |-- (1) HTTP Request (Verb + Header + Payload) -->
      |                                                |
      |                                      [ Network Device / Controller (Server) ]
      |                                                |
      |<-- (2) HTTP Response (Status Code + Payload) --|
```

1.  **Request**: 宛先 URL (Endpoint)、アクション (Verb)、認証情報 (Auth)、必要に応じてデータ (Payload) を送信します。
2.  **Response**: 処理結果を示すステータスコードと、要求されたデータ（通常は JSON 形式）が返されます。

---

## ⚙ 動作シーケンス

1.  **認証リクエスト**: ユーザー名/パスワードを POST し、一時的な **認証トークン** を取得します。
2.  **ヘッダーの準備**: 以降のリクエストに `X-Auth-Token` や `Authorization: Bearer <token>` を含めます。
3.  **API エンドポイントの呼び出し**: 特定のリソース（例: `/api/fmc_config/v1/domain/default/object/networks`）を指定します。
4.  **データ処理**: Python で受信した JSON を辞書型に変換し、必要な情報を抽出または加工します。
5.  **クローズ/ログアウト**: 必要に応じてセッションを終了します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Requests ライブラリの基本**: `requests.get()`, `requests.post()` の引数（url, headers, auth, data, verify=False）を正確に記述できる必要があります。
*   **認証フローの理解**: ラボでは「FMC にログインして認証トークンを取得するスクリプトを作成せよ」といった手順が最初のステップになることが多いです。
*   **JSON データのパース**: 受信した辞書データから、特定の `id` や `name` を抽出するループ処理が頻出です。
*   **HTTP ステータスコードの識別**: 
    *   `200 OK / 201 Created`: 成功。
    *   `401 Unauthorized`: 認証エラー。
    *   `403 Forbidden`: 権限不足。
    *   `404 Not Found`: エンドポイントの間違い。
*   **FMC API 固有の構造**: FMC API では `Domain UUID` を URL に含める必要があるため、これを最初に取得する手順をマスターしてください。

---

## 🛠 設定方法

### 1. Python Requests による GET リクエスト (DNAC デバイス一覧取得例)
```python
import requests
import json

# 認証トークンの取得（簡易化）
token_url = "https://dnac/dna/system/api/v1/auth/token"
auth = ("username", "password")
response = requests.post(token_url, auth=auth, verify=False)
token = response.json()['Token']

# デバイス一覧の取得
url = "https://dnac/dna/intent/api/v1/network-device"
headers = {
    "X-Auth-Token": token,
    "Content-Type": "application/json"
}

resp = requests.get(url, headers=headers, verify=False)
devices = resp.json()['response']

for device in devices:
    print(f"Hostname: {device['hostname']}, IP: {device['managementIpAddress']}")
```

---

## 🔍 検証コマンド

| 目的 | 手法 |
| :--- | :--- |
| **API 接続性の確認** | <code>curl -k -u user:pass https://[IP]/api/...</code> |
| **Python 内でのデバッグ** | <code>print(response.status_code)</code> / <code>print(response.text)</code> |
| **FMC API エクスプローラ** | FMC GUI の <code>https://[FMC_IP]/api/api-explorer</code> を利用。 |
| **データの型確認** | <code>print(type(data))</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 認証エラー (401) | トークンの期限切れ、または認証情報の誤り | 新しいトークンを再取得する。 |
| 接続拒否 (ConnectionRefused) | デバイス側で API サービスが未有効 | FMC/ISE で REST API 機能を Enable にする。 |
| JSONDecodeError | 応答が JSON 形式ではない（HTML エラーページ等） | <code>response.status_code</code> を確認し、エラー原因を特定。 |
| SSL 証明書エラー | 自己署名証明書による検証失敗 | リクエストに <code>verify=False</code> を追加。 |

---

## ⚠ 制限事項

*   **レート制限 (Rate Limiting)**: 短時間に大量の API リクエストを送信すると、デバイス側で一時的にブロックされることがあります。
*   **データ型の不一致**: JSON は文字列や数値を厳格に区別するため、Python 側での型変換に注意が必要です。
*   **バージョン依存**: FMC や ISE のバージョンアップにより、API のエンドポイントパスやペイロードの構造が変更される場合があります。

---

## 🔄 他技術との関連

*   **3.10 Cisco DNAC Northbound APIs**: インフラ全体のオーケストレーションに使用。
*   **Access Control Policy**: FMC API を使用して、セキュリティルールをプログラムで動的に挿入。
*   **ISE ERS/pxGrid**: 認証情報の共有やエンドポイントの隔離を自動化。

---

## 🧩 比較表

### データエンコーディング形式の比較

| 特徴 | JSON | XML | YAML |
| :--- | :--- | :--- | :--- |
| **可読性** | 高い | 中程度（タグが多い） | **最高** |
| **データ構造** | Key-Value, Array | Tree構造 | Key-Value, List |
| **API での普及度** | **標準的 (REST)** | レガシー / SOAP | 設定ファイル / Ansible |
| **構文例** | `{"key": "value"}` | `<key>value</key>` | `key: value` |

---

## 💡 ベストプラクティス

1.  **環境変数の活用**: パスワードやトークンをスクリプト内にハードコードせず、環境変数から読み込むようにします。
2.  **例外処理の記述**: `try-except` ブロックを使用して、ネットワーク切断や API エラーを適切に処理します。
3.  **モジュール化**: 認証処理や共通の API 呼び出しを関数化し、コードの再利用性を高めます。
4.  **検証環境の使用**: 本番の FMC 構成を破壊しないよう、必ず API Explorer や Sandbox 環境でテストします。

---

## 📝 ラボ学習・設定サンプル例

### 1. FMC 認証トークンの取得
*   **要件**: FMC (10.1.1.10) に対し、Basic 認証を使用してアクセストークンを取得せよ。
*   **設定**: 
    ```python
    url = "https://10.1.1.10/api/fmc_platform/v1/auth/generatetoken"
    r = requests.post(url, auth=('admin', 'Cisco123'), verify=False)
    token = r.headers.get('X-auth-access-token')
    ```

### 2. FMC ネットワークオブジェクトの作成
*   **要件**: 名前 `Internal_Net`、値 `192.168.1.0/24` のオブジェクトを作成せよ。
*   **設定例 (Payload)**: 
    ```json
    {
      "name": "Internal_Net",
      "type": "Network",
      "value": "192.168.1.0/24"
    }
    ```

### 3. GET リクエストの結果からのフィルタリング
*   **問題**: ISE のエンドポイント一覧を取得し、`MACAddress` が `00:11:22` で始まるものだけをリストアップせよ。

### 4. HTTP ステータスコードの検証スクリプト
*   **要件**: API 実行後、コードが `201` でない場合にエラーメッセージを表示する処理を書け。

### 5. JSON データの特定フィールド更新 (PUT/PATCH)
*   **要件**: 既存のオブジェクトの `description` を更新せlyo.

### 6. DNAC インベントリのエクスポート (CSV 保存)
*   **要件**: DNAC から取得したデバイス名を Python で CSV ファイルに書き出せ。

### 7. YAML からのデータ読み込みと API 適用
*   **要件**: `vars.yaml` に定義された複数の IP アドレスをループで FMC に登録せよ。

### 8. エラー処理 (Connection Error)
*   **要件**: デバイスがダウンしている場合に `ConnectionTimeout` をキャッチするスクリプトを作成せよ。

### 9. FMC ドメイン ID の取得
*   **操作**: `generatetoken` のレスポンスに含まれる `DOMAIN_UUID` を抽出し変数に格納せよ。

### 10. クエリパラメータの使用
*   **要件**: `limit=10` パラメータを使用して、最初の 10 件のみリソースを取得せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `headers = {"Content-Type": "application/json"}` という記述がある。POST リクエストにおいてこのヘッダーが必須な理由は？
    *   **回答**: サーバーに対して、リクエストボディ（Payload）のデータ形式が JSON であることを伝えるため。
2.  **実装**: FMC API でリソースを作成 (Create) する際に使用すべき HTTP 動詞は？
    *   **回答**: **POST**。
3.  **トラブルシュート**: スクリプトを実行すると `403 Forbidden` が返ってくる。認証トークンは正しく取得できている場合、何が原因か？
    *   **回答**: 使用している API ユーザーに、その操作（例：ポリシー変更）を実行するための **RBAC 権限** が付与されていない。
4.  **Design**: XML と JSON の両方をサポートする API において、Python で処理する際により適した（ネイティブに近い）形式はどちらか？
    *   **回答**: **JSON**。Python の辞書型 (Dictionary) と構造が非常に近く、`json` モジュールで簡単に変換できるため。
5.  **コンフィグ読解**: `requests.get(url, verify=False)` の `verify=False` は何を意味するか？
    *   **回答**: HTTPS 接続時にサーバーの SSL 証明書の検証をスキップすることを意味する。ラボ環境の自己署名証明書でエラーを防ぐために多用される。

---

## 🔗 参考リソース

*   **Cisco DevNet**: [FMC REST API Quick Start](https://developer.cisco.com/site/firepower/)
*   **Requests Documentation**: [Official Requests: HTTP for Humans](https://requests.readthedocs.io/)
*   **Cisco Learning Network**: [Programmability Topic Overview](https://learningnetwork.cisco.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: API の学習は、実際に「叩いてみる」ことが近道です。ラボ内の FMC ターミナルから Python を起動し、`dir(requests)` や `help(requests.post)` を活用して引数を確認する練習をしましょう。
*   **図解**: JSON は `[]` がリスト（PythonのList）、`{}` がオブジェクト（PythonのDict）に対応します。入れ子構造（Nested JSON）を正しく掘り下げる（例: `data['items']['id']`）感覚を養ってください。
*   **注意点**: ラボ試験では、スクリプトの正確な動作だけでなく、指定されたファイル名で保存することや、正確な出力を得ることも評価対象となります。
