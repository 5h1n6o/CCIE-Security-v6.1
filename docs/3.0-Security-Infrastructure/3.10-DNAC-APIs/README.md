---
layout: default
title: 3.10-DNAC-APIs
nav_order: 10
parent: 3.0-Security-Infrastructure
---

# 3.10 Cisco DNAC Northbound APIs use cases

Cisco DNA Center (DNAC) は、シスコのインテントベース ネットワーク（IBN）の中核をなすコントローラです。**Northbound API** は、外部のアプリケーション、スクリプト（Python等）、またはオーケストレーション ツール（ServiceNow等）が DNAC と対話するための RESTful インターフェイスを提供します。CCIE Security v6.1 では、単なる知識だけでなく、Python を使用して認証、ネットワーク検出、デバイス管理、ホスト情報の取得を自動化する実層能力が問われます。

---

## 📘 概要

*   **機能概要**: HTTPS 経由で JSON 形式のデータをやり取りする REST API インターフェイス。DNAC のインベントリ情報、トポロジ、ポリシー、アシュアランスデータへのアクセスを可能にします。
*   **利用目的**: ネットワーク運用の自動化、大量のデバイス設定の一括変更、カスタムレポートの作成、および外部システムとのデータ同期。
*   **どのような場面で利用するか**:
    *   **インベントリ管理**: 新しく接続されたデバイスやホストを自動的に検出し、資産管理データベース（CMDB）へ登録する。
    *   **トラブルシューティング**: API を介して「コマンドランナー」を実行し、複数のデバイスから同時に診断情報を収集する。
    *   **セキュリティコンプライアンス**: デバイスのソフトウェアバージョンやパッチ適用状況を定期的にチェックし、レポートを生成する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **通信プロトコル** | HTTPS (TCP 443)。 |
| **データ形式** | JSON (JavaScript Object Notation)。 |
| **認証方式** | トークンベース認証（ユーザー名/パスワードでトークンを取得し、以降のヘッダーに使用）。 |
| **主要カテゴリ** | 認証(Authentication)、検出(Discovery)、デバイス(Device)、ホスト(Host)。 |
| **非同期処理** | 多くの POST/PUT リクエストは「Task ID」を返し、処理完了を別途確認する必要がある。 |
| **設計上の注意点** | API のレート制限（スロットリング）と、トークンの有効期限（デフォルト 1時間）を考慮する。 |

---

## 🏗 動作原理

DNAC Northbound API は、アプリケーション層とネットワーク制御層を橋渡しします。

```text
[ External App / Python Script ]
      ↓ (REST API Request: GET/POST/PUT/DELETE)
[ Cisco DNA Center (Northbound Interface) ]
      ↓ (Intent / Analysis)
[ Network Infrastructure (Southbound: SSH/SNMP/NETCONF) ]
```

1.  **意図（Intent）の送信**: スクリプトが「全デバイスのリストを取得せよ」という意図を REST リクエストとして送信します。
2.  **抽象化**: DNAC は背後の複雑なネットワーク構成を抽象化し、統一された JSON 形式で結果を返します。

---

## ⚙ 動作シーケンス

1.  **Authentication (認証)**:
    *   `/dna/system/api/v1/auth/token` に対し、Basic 認証（ユーザー名/パスワード）を用いて POST リクエストを送信し、`serviceTicket`（トークン）を取得します。
2.  **API Call (リクエスト)**:
    *   取得したトークンを HTTP ヘッダー `X-Auth-Token` に含め、目的のエンドポイント（例：`/dna/intent/api/v1/network-device`）を呼び出します。
3.  **Task Handling (タスク処理)**:
    *   デバイス検出や設定変更など時間がかかる処理の場合、DNAC は `taskId` を返します。スクリプトは `/taskId` エンドポイントをポーリングして結果を確認します。
4.  **Response Parsing (解析)**:
    *   返された JSON ボディを Python の辞書オブジェクトとしてパースし、必要な情報を抽出します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **認証トークンの取得**: すべての API 操作の起点です。Python の `requests.post()` を使用し、正しい URL と `auth=(user, pass)` を指定できる必要があります。
*   **エンドポイントの特定**:
    *   デバイス一覧: `/network-device`
    *   ホスト一覧: `/network-host`
    *   ネットワーク検出: `/discovery`
*   **JSON のパース**: 戻り値は通常 `response.json()['response']` のような階層構造になっています。目的の値（管理 IP アドレスや MAC アドレス）を抽出するループ処理が頻出です。
*   **エラーコードの識別**: `401` (Unauthorized)、`403` (Forbidden)、`404` (Not Found) 等を適切にハンドリングすることが求められます。
*   **非同期タスクの待ち合わせ**: ラボで「検出が完了するまで待機せよ」という要件があれば、Task API をループで回す実装が必要です。

---

## 🛠 設定方法

### 1. 認証トークンの取得 (Python)
```python
import requests
import json

# 自己署名証明書の警告を無視
requests.packages.urllib3.disable_warnings()

dnac_ip = "10.1.1.10"
auth_url = f"https://{dnac_ip}/dna/system/api/v1/auth/token"
# Basic Auth
auth = ("admin", "Cisco123")

response = requests.post(auth_url, auth=auth, verify=False)
token = response.json()['Token']
print(f"Token: {token}")
```

### 2. デバイス一覧の取得
```python
headers = {
    "X-Auth-Token": token,
    "Content-Type": "application/json"
}
device_url = f"https://{dnac_ip}/dna/intent/api/v1/network-device"

# GETリクエスト
devices = requests.get(device_url, headers=headers, verify=False).json()['response']

for device in devices:
    print(f"Hostname: {device['hostname']}, IP: {device['managementIpAddress']}")
```

---

## 🔍 検証コマンド

| 目的 | 手法 / エンドポイント |
| :--- | :--- |
| **API 到達性確認** | <code>curl -k -u user:pass https://[IP]/dna/system/api/v1/auth/token</code> |
| **インベントリ確認** | <code>GET /dna/intent/api/v1/network-device</code> |
| **クライアント/ホスト確認** | <code>GET /dna/intent/api/v1/network-host</code> |
| **タスク状況確認** | <code>GET /dna/intent/api/v1/task/[taskId]</code> |
| **GUIでの確認** | **DNAC GUI > Platform > Developer Toolkit > APIs** |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| `401 Unauthorized` | ユーザー名/パスワードの間違い、またはトークン期限切れ | トークンを再生成し、`X-Auth-Token` ヘッダーを更新する。 |
| `403 Forbidden` | API ユーザーの RBAC 権限不足 | DNAC GUI でユーザーに正しいロールが割り当てられているか確認。 |
| 接続タイムアウト | ネットワーク到達性または DNAC サービスのダウン | ポート 443 の疎通確認。DNAC の `maglev` サービスの状態を確認。 |
| JSONDecodeError | 応答が空、または HTML エラーページが返った | `response.status_code` を確認。URL のスペルミスをチェック。 |

---

## ⚠ 制限事項

*   **同時接続数**: 大量のリクエストを並列で投げると、DNAC 側で 503 エラー（Service Unavailable）が発生する場合があります。
*   **バージョン依存**: API バージョン（v1, v2 等）により、エンドポイントや JSON 構造が変更されることがあります。
*   **読み取り専用**: デフォルトの API ユーザー設定によっては、GET は可能だが POST/PUT（設定変更）が制限されている場合があります。

---

## 🔄 他技術との関連

*   **3.9 Python Interaction**: REST API を操作するための言語基盤。
*   **3.11 SD-Access**: Fabric 構成をプログラムから制御するために DNAC API を使用。
*   **3.6.a NetFlow / Application Telemetry**: API を介してテレメトリ設定のプロビジョニングを自動化する。

---

## 🧩 比較表

### Northbound API vs Southbound Interface

| 特徴 | Northbound (REST) | Southbound (CLI/SNMP/NETCONF) |
| :--- | :--- | :--- |
| **対話相手** | アプリケーション、スクリプト | ネットワーク機器（スイッチ等） |
| **抽象度** | 高い（意図に基づく） | 低い（個別のコマンド） |
| **標準** | HTTP/JSON | 各プロトコルに依存 |
| **用途** | 自動化、オーケストレーション | デバイスへの直接設定適用 |

---

## 💡 ベストプラクティス

1.  **環境変数の使用**: スクリプト内に認証情報をハードコードせず、外部ファイルや環境変数から読み込む。
2.  **`verify=False` の慎重な使用**: ラボ環境では多用しますが、本番環境では証明書検証を有効にすべきです。
3.  **Task ID の検証**: POST 操作後は必ず戻り値の `taskId` を確認し、成功メッセージを受け取るまで処理を終了させない。
4.  **API エクスプローラの活用**: コードを書く前に DNAC GUI の「API エクスプローラ」でパラメータとレスポンス構造をテストする。

---

## 📝 ラボ学習・設定サンプル例

### 1. トークン生成スクリプト
*   **問題**: DNAC に対し、`X-Auth-Token` を取得して画面に表示するスクリプトを作成せよ。
*   **要件**: `requests` モジュールを使用すること。

### 2. デバイスインベントリの CSV 出力
*   **問題**: 全てのデバイスの `hostname` と `managementIpAddress` を取得し、`inventory.csv` に保存せよ。

### 3. 特定デバイスの ID 取得
*   **要件**: ホスト名が "Edge-SW" であるデバイスの `uuid` を抽出せよ。

### 4. ネットワーク検出の開始 (POST)
*   **問題**: IP 範囲 `192.168.1.0/24` に対して SNMPv2 を使用した検出を開始せよ。
*   **エンドポイント**: `/dna/intent/api/v1/discovery`。

### 5. ホスト（クライアント）の検索
*   **要件**: MAC アドレス `00:0C:29:XX:XX:XX` を持つホストが接続されているスイッチポートを特定せよ。

### 6. 設定テンプレートの適用 (Source 参照)
*   **要件**: 事前に作成されたテンプレート `Security-Baseline` をデバイスに適用せよ。

### 7. コマンドランナーの実行
*   **要件**: 特定のデバイスに対して `show running-config` を実行し、結果を取得せよ。

### 8. テレメトリ設定の確認
*   **問題**: NetFlow コレクタの設定が特定のデバイスにプロビジョニングされているか確認せよ。

### 9. IP プール情報の取得
*   **要件**: `Campus-Pool` という名前の IP アドレスプールの残数を確認せよ。

### 10. エラーハンドリングの実装
*   **課題**: HTTP 401 エラーが発生した際に "Authentication Failed" と表示して終了する try-except 構造を作成せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: Python スクリプトで `headers = {'X-Auth-Token': token}` が設定されている。このヘッダーを付与せずに GET リクエストを送信した場合、返ってくる HTTP ステータスコードは？
    *   **回答**: `401 Unauthorized`。
2.  **Design**: 大規模環境で API を使用して全デバイスのステータスを取得する際、スクリプトが突然 503 エラーで停止した。最も可能性の高い原因は？
    *   **回答**: DNAC の API レート制限（Throttling）に達した。
3.  **トラブルシュート**: `/network-device` API でデバイス情報は取得できるが、`/discovery` API で POST すると失敗する。何を確認すべきか？
    *   **回答**: API ユーザーに割り当てられた Role（権限）が "Network-Admin" などの Write 権限を持っているか。
4.  **実装**: DNAC API における非同期処理（Async）の一般的な流れを説明せよ。
    *   **回答**: POST/PUT 送信後、レスポンスに含まれる `taskId` を取得し、その ID を用いて Task API をポーリングして結果を確認する。
5.  **Design**: ネットワーク内のすべての無線クライアントの接続状況をリアルタイムに外部ダッシュボードに表示したい。使用すべき Northbound API カテゴリは？
    *   **回答**: **Network Host API** (`/network-host`)。

---

## 🔗 参考リソース

*   **Cisco DNA Center API Reference**: [Cisco DevNet - DNAC Intent API](https://developer.cisco.com/docs/dna-center/)
*   **Configuration Guide**: [Cisco DNA Center User Guide, Release 2.2.2 - APIs](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/dna-center/2-2-2/user-guide/b_cisco_dna_center_ug_2_2_2.html)
*   **Cisco Live (DEVNET-2012)**: [Introduction to Cisco DNA Center REST APIs](https://www.ciscolive.com/)
*   **CVD**: [Software-Defined Access Design Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/cisco-sda-design-guide.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: API の学習には、実際に Postman や Python を使って叩いてみることが不可欠です。DNAC の GUI にある「Developer Toolkit」を活用して JSON 構造に慣れておきましょう。
*   **図解**: 
    - `/auth/token` → カギを借りる。
    - `/network-device` → カギを使って中を見る（インベントリ）。
    - `/task` → 頼んだ仕事が終わったか確認する。
*   **注意点**: ラボ試験では、API を叩く際に対象の `uuid` が必要になることが多いため、「名前から UUID を引く」ステップをコード内で忘れないようにしてください。
