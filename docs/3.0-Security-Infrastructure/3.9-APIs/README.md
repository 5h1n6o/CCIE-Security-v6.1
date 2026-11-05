---
layout: default
title: 3.9-APIs
nav_order: 9
parent: 3.0-Security-Infrastructure
---

# 3.9 Interaction with network devices through APIs using basic Python scripts

## 📘 概要

現代のエンタープライズ・ネットワークセキュリティインフラストラクチャにおいて、単一のデバイス管理からプログラマブルな集中管理および自動化オーケストレーションへの移行は不可欠な要件となっています。CCIE Security v6.1 Blueprint 3.9「Interaction with network devices through APIs using basic Python scripts」は、Cisco Security 製品群（Cisco Firepower Management Center: FMC, Firepower Device Manager: FDM, Identity Services Engine: ISE, Cisco Catalyst IOS-XE, Stealthwatch/Secure Network Analytics など）に対して、Python スクリプトおよび REST API / RESTCONF / NETCONF プロトコルを用いてプログラマブルに操作・統合・自動化する実践的スキルを評価する項目です。

従来の手動 GUI / CLI 操作では、数百台のファイアウォールへのポリシー適用、大量の IP / ネットワークオブジェクトの更新、あるいはセキュリティインシデント発生時の動的隔離（Quarantine）処理に多大な時間と人的ミスのリスクを伴っていました。基本 Python スクリプトと API の連携を活用することで、以下のユースケースを高速かつ正確に実現できます。

* **セキュリティオブジェクト・ポリシーの一括自動生成**: FMC や FDM に対して、数百個のネットワークオブジェクト、サービスオブジェクト、Access Control Policy (ACP) ルールを数秒で動的投入。
* **アイデンティティ & ゲストユーザー管理の自動化**: Cisco ISE の ERS (External RESTful Services) API や OpenAPI を経由し、BYOD / ゲストユーザーのアカウント発行、エンドポイント MAC アドレスのグループ登録を自動実行。
* **ネットワーク機器の宣言型パラメータ設定**: Catalyst IOS-XE や CSR1000v に対する RESTCONF (`/restconf/data/...`) または NETCONF (`ncclient`) を用いた、データモデル（YANG モデリング）ベースのインターフェース・ルーティング・ACL 設定。
* **セキュリティインシデントにおける動的対抗措置 (Dynamic Remediation)**: FMC や Stealthwatch から検知した脅威 IP アドレスを即座に抽出し、FMC の Security Intelligence (SI) リストや ISE の Adaptive Network Control (ANC) ポリシーを更新してネットワークから自動隔離。

本学習メモでは、CCIE Security ラボ試験環境（Python 3 環境、標準ライブラリおよび `requests` / `ncclient` モジュール）において、API 認証ヘッダーの取り扱い、トークンリフレッシュ、JSON データの相互変換、エラーハンドリング、および実効性の高いスクリプト実装例を徹底的に解説します。


---

## 🔑 要点

Cisco セキュリティデバイスと Python API 連携における主要な技術的特徴、利用用途、メリット、制限事項、および設計上の注意点を下表に整理します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | HTTP/HTTPS プロトコル上の REST (REpresentational State Transfer) 構造（JSON/XML 形式）や、YANG データモデルに基づく RESTCONF/NETCONF を使用し、Python スクリプトからプログラム経由でデバイスを設定・監視 |
| **用途** | FMC/FDM のオブジェクトおよびポリシー自動生成、ISE のエンドポイント・ゲストアカウント管理、IOS-XE の RESTCONF 設定更新、脅威検知時の動的ブロック処理 |
| **メリット** | ヒューマンエラーの排除、設定投入速度の飛躍的向上、他システム（SIEM, SOAR, Ticketing ツール）との双方向リアルタイム連携の実現 |
| **デメリット** | スクリプトのロジックエラーによる大規模な設定破壊リスク、認証トークンの有効期限管理（FMC: 30分/2時間、FDM: 24時間等）や API レート制限（Rate Limiting）への配慮が必要 |
| **対応機種** | Cisco FMC (v6.x/v7.x), Cisco FDM (v6.x/v7.x), Cisco ISE (v2.x/v3.x), Catalyst 9300/9500/ISR4000/CSR1000v (IOS-XE), Cisco WSA/ESA, Stealthwatch (SMC) |
| **制限事項** | ラボ試験環境ではインターネット接続が遮断されているため、外部 PyPI からのパッケージ追加不可 (`pip install` 不可)。標準ライブラリ (`urllib`, `json`, `ssl`, `csv`) および組み込み済みライブラリ (`requests`, `urllib3`, `ncclient`) のみで実装する制約 |
| **設計上の注意点** | 自己署名証明書環境での SSL/TLS 検証エラー回避 (`verify=False` と `urllib3.disable_warnings()`)、HTTP レスポンスコード (200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found) の厳格な判定 |


---

## 🏗 動作原理

Python スクリプトが Cisco セキュリティデバイスの REST API サーバーと通信する際の基本アーキテクチャおよびデータ構造は、標準的な Client-Server モデルに基づきます。

```
[ Python Script (Client) ]
          │
          │ 1. HTTP POST /api/fmc_platform/v1/auth/generatetoken
          │    (Header: Basic Auth / Authorization: Basic <Base64>)
          ▼
┌─────────────────────────────────────────────────────────────┐
│ Cisco FMC / ISE / FDM / IOS-XE (REST API / RESTCONF Server) │
└─────────────────────────────────────────────────────────────┘
          │
          │ 2. HTTP 200 OK
          │    (Header: X-auth-access-token & X-auth-refresh-token / DOMAIN_UUID)
          ▼
[ Python Script (Memory Storage) ]
          │
          │ 3. HTTP POST /api/fmc_config/v1/domain/{UUID}/object/networks
          │    (Header: X-auth-access-token, Content-Type: application/json)
          │    (Payload: {"name": "Host_10.1.1.1", "value": "10.1.1.1/32", "type": "Network"})
          ▼
┌─────────────────────────────────────────────────────────────┐
│ FMC Data Engine / Database Validation & Processing          │
└─────────────────────────────────────────────────────────────┘
          │
          │ 4. HTTP 201 Created / JSON Response with Object UUID
          ▼
[ Python Script (Parsing Output / Execution Log) ]
```

### 1. REST API のコア要素
* **HTTP メソッド (Verbs)**:
  * `GET`: リソース（オブジェクト、ポリシー、システム情報）の取得。
  * `POST`: 新規リソースの作成、あるいは認証トークンの生成リクエスト。
  * `PUT`: 既存リソースの完全更新（指定しなかったパラメータはデフォルト値にリセットされる危険性あり）。
  * `PATCH`: 既存リソースの一部更新（指定したフィールドのみ変更）。
  * `DELETE`: 指定された UUID / リソースの削除。
* **URI 構造**:
  * FMC: `https://<FMC_IP>/api/fmc_config/v1/domain/<DOMAIN_UUID>/object/networks`
  * FDM: `https://<FDM_IP>/api/fdm/v6/object/networks`
  * ISE ERS: `https://<ISE_IP>:9060/ers/config/endpoint`
  * IOS-XE RESTCONF: `https://<IOS_XE_IP>/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet1`
* **ヘッダー (HTTP Headers)**:
  * `Content-Type`: ペイロードの形式を指定。通常 `application/json`（RESTCONF では `application/yang-data+json`）。
  * `Accept`: サーバーから受け取るレスポンス形式を指定。通常 `application/json`（ISE ERS では `application/json` または `application/xml`）。
  * `Authorization` / `X-auth-access-token`: 認証トークンまたは Basic Auth 情報。

### 2. データフォーマット (JSON & YANG)
Python では `json` モジュールまたは `requests` ライブラリの `.json()` メソッドを使用して、JSON 構造（キー・値のペア、リスト、ネストされた辞書）と Python の Native データ型（`dict`, `list`, `str`, `int`, `bool`）を相互変換（De-serialization / Serialization）します。

```json
{
  "name": "Branch_LAN_Network",
  "value": "192.168.10.0/24",
  "type": "Network",
  "description": "Created via Python REST API script"
}
```


---

## ⚙ 動作シーケンス

Python スクリプトから セキュリティデバイス（例: FMC）へ API リクエストを送信し、設定を反映させる全処理シーケンスをステップ順に説明します。

```
[ Python Script ]                                       [ FMC / ISE / FDM Server ]
       │                                                           │
       │ ── 1. SSL 検証無効化 & セッション初期化 ─────────────────►│ (準備)
       │                                                           │
       │ ── 2. POST /api/fmc_platform/v1/auth/generatetoken ──────►│ 認証要求 (Basic Auth)
       │    (Authorization: Basic YWRtaW46Q2lzY28xMjM=)            │
       │                                                           │
       │◄── 3. 200 OK + Header (X-auth-access-token, UUID) ────────┤ トークン発行
       │    (X-auth-access-token: eyJhbGciOi...)                  │ (有効期限: 30分)
       │                                                           │
       │ ── 4. GET /api/fmc_config/v1/domain/{UUID}/object/networks►│ オブジェクト存在確認
       │    (Header: X-auth-access-token)                          │
       │                                                           │
       │◄── 5. 200 OK + JSON Response (既存オブジェクト一覧) ───────┤ レスポンス返却
       │                                                           │
       │ ── 6. POST /api/fmc_config/v1/domain/{UUID}/object/networks►│ 新規オブジェクト作成
       │    (Header: X-auth-access-token, Content-Type: json)      │
       │    (Payload: JSON string)                                 │
       │                                                           │
       │◄── 7. 201 Created + JSON (生成された Object UUID) ─────────┤ DB 保存完了
       │                                                           │
       │ ── 8. POST /api/fmc_config/v1/domain/{UUID}/deployment/... ►│ 変更のデプロイ要求
       │                                                           │
       │◄── 9. 202 Accepted + Task ID ─────────────────────────────┤ 非同期処理開始
       │                                                           │
```

### シーケンス詳細解説
1. **初期化**: 自己署名証明書警告を抑止するため、`urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)` を呼び出し、`requests.Session()` インスタンスを生成。
2. **認証とトークン取得**: `POST` リクエストにより管理者クレデンシャル（User/Password）を送信。正常レスポンス（`200 OK`）の **レスポンスヘッダー** から `X-auth-access-token`（アクセストークン）、`X-auth-refresh-token`（リフレッシュトークン）、および `DOMAIN_UUID`（ドメイン識別子）を抽出。
3. **ヘッダー設定**: 以降の全 API 呼出ヘッダーに `X-auth-access-token` を設定。
4. **CRUD アクションの実行**: 対象リソースの URI に対し、`GET`（参照）、`POST`（追加）、`PUT/PATCH`（変更）、`DELETE`（削除）を実行。
5. **デプロイ（FMC / FDM）**: オブジェクトやポリシー作成後、FMC の場合は `/deployment/deployabledevices` および `/deployment/deploymentrequests` API を叩いて実機（FTD）へ設定を適用。
6. **トークンリフレッシュ**: トークン有効期限が切れそうになった場合、`POST /api/fmc_platform/v1/auth/refreshtoken` に `X-auth-refresh-token` を渡して新しいアクセストークンを取得。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、Blueprint 3.9 は「Python スクリプトを使用してデバイス API にアクセスし、指定された自動化要件を満たすスクリプトを完成させる」形で出題されます。試験端末（Linux / Windows ジャンプホスト）には Python 3 環境と テキストエディタ（VS Code, PyCharm, vim）が用意されています。

### 1. ラボ試験で頻出のタスク要件
* **FMC オブジェクト & アクセスポリシー自動投入**:
  * CSV ファイルや定数リストから大量の Host / Network / Group オブジェクトをパースし、FMC REST API で一括登録。
  * 作成したオブジェクトを既存の Access Control Policy (ACP) の最上部（Mandatory 領域）へ Deny ルールまたは Allow ルールとして自動組み込み。
* **ISE エンドポイント & ゲスト作成**:
  * ISE ERS API (Port 9060) を使用し、指定された MAC アドレスを特定の Endpoint Group (例: `Blacklist` や `Quarantine_Endpoints`) へ追加。
  * ゲストポータル用の臨時ゲストユーザー（Username, Password, Expiration）を ERS API 経由で作成。
* **IOS-XE RESTCONF インターフェース・ACL 設定**:
  * RESTCONF (`https://<router_ip>/restconf/...`) を使用して、インターフェースの Description、IP アドレス変更、または Access-List (ACL) の動的適用。

### 2. よくある設定ミス・ハマりポイント
* **FMC のトークン取得場所の勘違い**:
  * FMC REST API の認証トークン（`X-auth-access-token`）は、レスポンスの **JSON ペイロード内ではなく、レスポンスヘッダー (`response.headers['X-auth-access-token']`)** に含まれます。ペイロードから取ろうとすると `KeyError` になります。
* **FMC ドメイン UUID (DOMAIN_UUID) の未取得**:
  * FMC API のリソース URI には `.../domain/<DOMAIN_UUID>/...` が必要です。認証時のレスポンスヘッダー `response.headers['DOMAIN_UUID']` から取得するか、`/api/fmc_platform/v1/info/domain` で参照する必要があります。
* **ISE ERS API の ポート番号と HTTP ヘッダー**:
  * ISE ERS API はデフォルトポートが **`9060`** です（ポート 443 へのアクセスは失敗します）。
  * ヘッダーに `Accept: application/json` と `Content-Type: application/json` の両方を明示しないと、サーバーが XML やエラーを返します。
  * ISE ERS API は初期状態で無効化されている場合があり、ISE 管理画面 (`Administration > System > Settings > ERS Settings`) で `Enable ERS for Read/Write` をオンにする必要があります。
* **SSL 証明書検証による例外発生**:
  * `requests.get('https://...', verify=False)` の `verify=False` を忘れると、自己署名証明書環境で `SSLError` が発生してスクリプトが即座に停止します。
* **RESTCONF の `Content-Type`**:
  * RESTCONF では、通常の `application/json` ではなく **`application/yang-data+json`** を要求されるケースが多く、不一致の場合は `415 Unsupported Media Type` エラーとなります。

### 3. デバッグとエラー判定テクニック
スクリプト実行時は必ずステータスコードをチェックし、エラー発生時に詳細を出力させます。

```python
import requests
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

response = requests.post(url, headers=headers, json=payload, verify=False)
if response.status_code not in [200, 201, 202, 204]:
    print(f"[-] API Call Failed! Status Code: {response.status_code}")
    print(f"[-] Response Text: {response.text}")
else:
    print("[+] Success!")
```


---

## 🛠 設定方法

Python スクリプトから API を利用する前に、各端末・サーバー側で API サービスを有効化しておく必要があります。

### 1. Cisco FMC (Firepower Management Center)
FMC ではデフォルトで REST API が有効化されていますが、有効化状態の確認は以下の手順で行います。
* **FMC GUI**: `System > Configuration > REST API Preferences`
* **設定確認**: `Enable REST API` がチェックされていることを確認（デフォルト: 有効）。
* **API Explorer**: `https://<FMC_IP>/api/api-explorer/` へブラウザでアクセスすると、利用可能な全 API ターゲットと対話型ドキュメントが参照可能。

### 2. Cisco ISE (Identity Services Engine)
ISE ERS API (External RESTful Services) の有効化手順:
* **ISE GUI**: `Administration > System > Settings > ERS Settings`
* **設定変更**: `ERS (Read/Write)` を **`Enable`** に設定。
* **ERS SDK / API Explorer**: `https://<ISE_IP>:9060/ers/sdk` へアクセス。

### 3. Cisco FDM (Firepower Device Manager)
FDM もデフォルトで REST API が有効です。
* **FDM API Explorer**: `https://<FDM_IP>/api-explorer/`

### 4. Cisco IOS-XE (Catalyst / CSR1000v)
IOS-XE ルータ/スイッチで RESTCONF および NETCONF を有効化する CLI 設定:

```text
! RESTCONF 有効化 (HTTPS ポート 443)
ip http server
ip http secure-server
restconf

! NETCONF 有効化 (SSH ポート 830)
netconf-yang

! AAA およびローカルユーザーの設定 (API 認証用)
aaa new-model
aaa authentication login default local
aaa authorization exec default local
username admin privilege 15 secret Cisco12345!
```


---

## 🔍 検証コマンド

API 呼び出しおよび Python スクリプト動作確認のための検証コマンド・API リクエスト一覧を下表にまとめます。

| 目的 | 確認コマンド / API リクエスト | 期待されるレスポンス / 動作 |
| --- | --- | --- |
| **FMC 認証トークン取得** | `POST https://<FMC>/api/fmc_platform/v1/auth/generatetoken` (Basic Auth) | `200 OK`, ヘッダーに `X-auth-access-token` および `DOMAIN_UUID` |
| **FMC ネットワークオブジェクト参照** | `GET https://<FMC>/api/fmc_config/v1/domain/{UUID}/object/networks` | `200 OK`, オブジェクト一覧の JSON レスポンス |
| **FMC ACP ポリシー取得** | `GET https://<FMC>/api/fmc_config/v1/domain/{UUID}/policy/accesspolicies` | `200 OK`, アクセスポリシー名と Policy UUID |
| **FMC デプロイ状態参照** | `GET https://<FMC>/api/fmc_config/v1/domain/{UUID}/deployment/deployabledevices` | `200 OK`, 未デプロイ変更を持つデバイス一覧 |
| **ISE エンドポイント一覧参照** | `GET https://<ISE>:9060/ers/config/endpoint` (Header: `Accept: application/json`) | `200 OK`, エンドポイント MAC アドレス一覧 |
| **IOS-XE RESTCONF インターフェース確認** | `GET https://<IOS_XE>/restconf/data/ietf-interfaces:interfaces` | `200 OK`, インターフェース設定の JSON データ |
| **IOS-XE NETCONF セッション確認 (CLI)** | `show netconf-yang sessions` | アクティブな NETCONF-YANG SSH セッション |
| **IOS-XE RESTCONF 稼働状態確認 (CLI)** | `show processes cpu | include http` / `show ip http server status` | HTTP/HTTPS サーバーが `enabled` であること |


## 🚨 トラブルシュート

API 連携および Python スクリプト実行時に発生しやすいトラブル症状、原因、および確認・対処方法を下表に整理します。

| 症状 | 原因 | 確認・判定方法 | 対処方法 |
| --- | --- | --- | --- |
| **`requests.exceptions.SSLError` でスクリプトが停止** | セバイスが自己署名 TLS 証明書を使用しており、Python が証明書検証に失敗 | トレースバックに `[SSL: CERTIFICATE_VERIFY_FAILED]` が表示 | `requests.get(..., verify=False)` を指定し、`urllib3.disable_warnings()` で警告を抑止 |
| **HTTP 401 Unauthorized エラー** | ① クレデンシャル（ユーザー名/パスワード）誤り<br>② アクセストークン切れ（FMC 30分経過） | レスポンスボディまたはヘッダーの確認 | ① クレデンシャル再確認<br>② `POST /auth/refreshtoken` でトークン再取得、または再度 `generatetoken` を実行 |
| **HTTP 403 Forbidden エラー** | API を実行するユーザーに必要な管理権限（Admin / API 権限）がない | ISE ERS または FMC ユーザーロールの確認 | FMC GUI または ISE GUI で API ユーザーに `API Access` または `System Admin` 権限を付与 |
| **HTTP 415 Unsupported Media Type** | `Content-Type` ヘッダーの設定漏れまたは誤り | スクリプトの `headers` 辞書を確認 | FMC/ISE: `Content-Type: application/json`<br>RESTCONF: `Content-Type: application/yang-data+json` を正しく指定 |
| **HTTP 400 Bad Request エラー** | 送信した JSON ペイロードの構文エラー、必須フィールドの不足、型不一致 | `response.text` でエラー詳細メッセージをインスペクト | エラーメッセージの指摘に従い、JSON ペイロードのキー名・データ型（文字列/数値/リスト）を修正 |
| **HTTP 404 Not Found エラー** | URI 内の UUID 誤り、またはドメイン UUID (`DOMAIN_UUID`) の未設定 | 呼び出し先 URL 文字列を出力 (`print(url)`) | FMC ドメイン UUID やオブジェクト UUID が正しく URI に埋め込まれているか検証 |
| **ISE ERS API で `Connection Refused` エラー** | ERS API が ISE 上で無効化されている、またはポート `9060` 未指定 | `telnet <ISE_IP> 9060` でポート接続確認 | ISE 管理画面 (`Administration > System > Settings > ERS Settings`) で `Enable ERS` を有効化 |


---

## ⚠ 制限事項

API 連携における技術的制約および運用上の制限事項は以下の通りです。

* **FMC API トークン有効期限**:
  * FMC アクセストークン (`X-auth-access-token`) の寿命は **30 分** です。
  * リフレッシュトークン (`X-auth-refresh-token`) を使用して最大 3 回までリフレッシュ可能（通算最大 **2 時間**）。2 時間経過後は再度 `generatetoken` による完全再認証が必要となります。
* **FMC レート制限 (Rate Limiting)**:
  * FMC API は 1 分間に受容できるリクエスト数に上限があります（環境により 120 リクエスト/分程度）。短時間に大量の POST/PUT をループ実行すると `429 Too Many Requests` エラーが返されるため、スクリプト内で `time.sleep(0.5)` などの適切なウェイトを挿入する必要があります。
* **ISE ERS レート制限**:
  * ISE ERS API も過大なリクエストに対しスロットリングを発生させます。
* **RESTCONF の部分更新における制限**:
  * `PUT` メソッドは対象パスのリソース全体を置き換えるため、パラメータの一部分だけを変更する目的で `PUT` を使用すると、他の未指定パラメータが初期化される恐れがあります。特定パラメータのみの修正には `PATCH` メソッドを使用するか、事前に `GET` で全設定を取得してから更新する必要があります。


---

## 🔄 他技術との関連

Python API スクリプトは、CCIE Security Blueprint における各種セキュリティコンポーネントと密接に連携します。

* **Cisco FMC / FTD (3.0 Infrastructure)**:
  * ネットワーク/サービスオブジェクト、Access Control Policy (ACP)、Security Intelligence (SI) ブラックリスト、およびデプロイ処理の全自動化。
* **Cisco ISE (Identity & Access Control)**:
  * 外部 ID データベース連携、エンドポイント MAC グループ登録、Adaptive Network Control (ANC) ポリシー適用による感染端末の Quarantine (隔離) 処理。
* **Cisco Catalyst IOS-XE (Control/Data Plane Protection)**:
  * RESTCONF / NETCONF を使用した CoPP (Control Plane Policing)、iACL (Infrastructure ACL)、VACL、ルーティングポリシーの集中的投入。
* **SIEM / 脅威情報連携 (Threat Intelligence)**:
  * 外部 Threat Feed (STIX/TAXII 等) から脅威 IP を取得し、Python スクリプト経由で FMC や ISE のブロックリストへ自動反映するエコシステム構築。


---

## 🧩 比較表

プログラマブル管理インタフェース（REST API, RESTCONF, NETCONF）および主要デバイスの API 特性の比較を下表に示します。

### 1. 管理インタフェース比較 (REST API vs RESTCONF vs NETCONF)

| 比較項目 | REST API (FMC/ISE/FDM) | RESTCONF (IOS-XE) | NETCONF (IOS-XE) |
| --- | --- | --- | --- |
| **標準規格** | HTTP/REST 設計思想（ベンダー独自 URI） | RFC 8040 (標準化 REST) | RFC 6241 (標準化 XML over SSH) |
| **トランスポート** | HTTP / HTTPS (Port 443) | HTTP / HTTPS (Port 443) | SSH (Port 830) |
| **データフォーマット** | JSON / XML | JSON (`application/yang-data+json`) / XML | XML |
| **データモデル** | ベンダー独自オブジェクトモデル | YANG Data Model (RFC 6020 / Native) | YANG Data Model (RFC 6020 / Native) |
| **Python ライブラリ** | `requests` | `requests` | `ncclient` |
| **主な用途** | FMC/ISE/FDM 等の上位管理サーバー自動化 | ネットワーク機器の Web 指向軽量自動化 | 大規模ルータ/スイッチのトランザクション管理 |

### 2. 主要デバイス API 特性比較 (FMC vs FDM vs ISE)

| 比較項目 | FMC REST API | FDM REST API | ISE ERS API |
| --- | --- | --- | --- |
| **認証方式** | `POST /auth/generatetoken` (Basic Auth) ──► `X-auth-access-token` | `POST /fdm/token` (Grant Type) ──► Bearer Token | Basic Auth (`Authorization: Basic ...`) または OAuth |
| **通信ポート** | HTTPS (Port 443) | HTTPS (Port 443) | HTTPS (Port **9060**) |
| **トークン寿命** | 30分 (Refresh で最大 2時間) | 24時間 | 要求ごとに Basic Auth ヘッダー送信可能 |
| **デプロイ要否** | **必要** (`/deployment/deploymentrequests`) | **必要** (`/operational/deploy`) | **不要** (設定即時反映) |


---

## 💡 ベストプラクティス

Python API スクリプト開発およびラボ試験におけるベストプラクティスを以下に提示します。

1. **`requests.Session()` の活用**:
   * 同一サーバーへの連続リクエストでは `s = requests.Session()` を使用することで、TCP/TLS ハンドシェイクを再利用し、処理速度を大幅に向上させます。
2. **モジュール化と関数設計**:
   * 認証（トークン取得）、オブジェクト作成、ポリシー更新、デプロイなどの機能を独立した関数（例: `get_fmc_token()`, `create_network_object()`）に分割し、コードの再利用性と可読性を高めます。
3. **堅牢なエラーハンドリング**:
   * 全ての API 呼び出しに対して `response.raise_for_status()` または `status_code` チェックを実装し、エラー発生時に HTTP ステータスコードとエラーメッセージ本文を出力させることで、デバッグ時間を最小化します。
4. **大量データ処理におけるバッチ化 & スロットリング**:
   * 大量のオブジェクトを作成する際は、数10個ごとに `time.sleep(0.5)` を挟んで FMC/ISE サーバーの API レート制限回避を図ります。


## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験で即座に応用できる、実践的かつ完全な Python スクリプトのサンプルコードを 10 個掲載します。コードは省略せず、そのまま実行可能な形で記載しています。

---

### シナリオ 1: FMC - 認証トークン取得 & ドメイン UUID 自動抽出スクリプト

#### 【問題】
FMC の REST API を使用して認証を行い、後続の API 呼び出しで必要となるアクセストークン (`X-auth-access-token`) およびドメイン UUID (`DOMAIN_UUID`) を自動取得する Python 関数を作成しなさい。

#### 【要件】
* FMC IP: `192.168.10.5`
* ユーザー名: `admin`, パスワード: `LabPassword123`
* TLS 証明書検証エラーを抑止すること。
* 正常取得時にトークンと Domain UUID を画面に表示し、他の関数から利用できるように戻り値として返却すること。

#### 【設定例 (Python スクリプト)】

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

FMC_IP = "192.168.10.5"
USERNAME = "admin"
PASSWORD = "LabPassword123"

def get_fmc_auth_token(fmc_ip, username, password):
    url = f"https://{fmc_ip}/api/fmc_platform/v1/auth/generatetoken"
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
        
        if response.status_code == 200:
            token = response.headers.get("X-auth-access-token")
            domain_uuid = response.headers.get("DOMAIN_UUID")
            print("[+] FMC Authentication Successful!")
            print(f"    - Access Token : {token[:20]}... (Truncated)")
            print(f"    - Domain UUID  : {domain_uuid}")
            return token, domain_uuid
        else:
            print(f"[-] Authentication Failed. Status Code: {response.status_code}")
            print(f"    - Response: {response.text}")
            return None, None
            
    except Exception as e:
        print(f"[-] Exception during authentication: {str(e)}")
        return None, None

if __name__ == "__main__":
    token, domain_uuid = get_fmc_auth_token(FMC_IP, USERNAME, PASSWORD)
```

#### 【検証方法】
端末上でスクリプトを実行し、`[+] FMC Authentication Successful!` とドメイン UUID が表示されることを確認します。

---

### シナリオ 2: FMC - リストからの大量ネットワークオブジェクト一括登録スクリプト

#### 【問題】
Python の辞書リストに定義された複数の IP / ネットワークオブジェクトを FMC へ REST API を経由して一括で追加しなさい。

#### 【要件】
* 対象オブジェクト群:
  * `Host_DB_Server`: `10.100.10.50/32`
  * `Host_App_Server`: `10.100.10.51/32`
  * `Net_Branch_Office`: `192.168.50.0/24`
* 登録成功時にオブジェクト名と FMC から返却された Object UUID を表示すること。
* エラー発生時は該当オブジェクト名を明示して処理を継続すること。

#### 【設定例 (Python スクリプト)】

```python
import requests
import urllib3
import time

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

FMC_IP = "192.168.10.5"
USERNAME = "admin"
PASSWORD = "LabPassword123"

OBJECTS_TO_CREATE = [
    {"name": "Host_DB_Server", "value": "10.100.10.50/32", "type": "Network"},
    {"name": "Host_App_Server", "value": "10.100.10.51/32", "type": "Network"},
    {"name": "Net_Branch_Office", "value": "192.168.50.0/24", "type": "Network"}
]

def create_network_objects():
    auth_url = f"https://{FMC_IP}/api/fmc_platform/v1/auth/generatetoken"
    res = requests.post(auth_url, auth=(USERNAME, PASSWORD), verify=False)
    if res.status_code != 200:
        print("[-] Auth failed.")
        return
        
    token = res.headers.get("X-auth-access-token")
    domain_uuid = res.headers.get("DOMAIN_UUID")
    
    obj_url = f"https://{FMC_IP}/api/fmc_config/v1/domain/{domain_uuid}/object/networks"
    headers = {
        "Content-Type": "application/json",
        "X-auth-access-token": token
    }
    
    for obj in OBJECTS_TO_CREATE:
        payload = {
            "name": obj["name"],
            "value": obj["value"],
            "type": obj["type"],
            "description": "Created automatically via Python script"
        }
        
        response = requests.post(obj_url, headers=headers, json=payload, verify=False)
        
        if response.status_code == 201:
            res_data = response.json()
            print(f"[+] Successfully Created: {obj['name']} (UUID: {res_data.get('id')})")
        else:
            print(f"[-] Failed to Create {obj['name']}: {response.status_code} - {response.text}")
            
        time.sleep(0.2)

if __name__ == "__main__":
    create_network_objects()
```

#### 【検証方法】
FMC GUI の `Objects > Object Management > Network` に移動し、3つのオブジェクトが正しく登録されていることを確認します。

---

### シナリオ 3: FMC - Access Control Policy (ACP) への新規 Deny ルール自動割り込みスクリプト

#### 【問題】
FMC 上の既存 Access Control Policy (ACP) 名 `Lab_ACP` の最上部（Mandatory 領域）に、特定の悪意のあるサブネット `198.51.100.0/24` からの全通信をドロップする `Block_Malicious_Subnet` ルールを追加しなさい。

#### 【要件】
* 事前に `198.51.100.0/24` を持つ Network オブジェクト `Net_Malicious_198.51.100.0` を作成（または存在確認）。
* ACP `Lab_ACP` の UUID を自動取得し、そのポリシー内の Rule 1（最上部）に `action: BLOCK` ルールを追加すること。

#### 【設定例 (Python スクリプト)】

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

FMC_IP = "192.168.10.5"
USERNAME = "admin"
PASSWORD = "LabPassword123"
ACP_NAME = "Lab_ACP"
MALICIOUS_NET_NAME = "Net_Malicious_198.51.100.0"
MALICIOUS_NET_VAL = "198.51.100.0/24"

def add_deny_rule_to_acp():
    auth_res = requests.post(f"https://{FMC_IP}/api/fmc_platform/v1/auth/generatetoken", 
                             auth=(USERNAME, PASSWORD), verify=False)
    token = auth_res.headers.get("X-auth-access-token")
    domain_uuid = auth_res.headers.get("DOMAIN_UUID")
    headers = {"Content-Type": "application/json", "X-auth-access-token": token}
    
    net_url = f"https://{FMC_IP}/api/fmc_config/v1/domain/{domain_uuid}/object/networks"
    net_payload = {"name": MALICIOUS_NET_NAME, "value": MALICIOUS_NET_VAL, "type": "Network"}
    net_res = requests.post(net_url, headers=headers, json=net_payload, verify=False)
    
    if net_res.status_code == 201:
        net_obj_data = net_res.json()
    else:
        get_net = requests.get(f"{net_url}?filter=name:{MALICIOUS_NET_NAME}", headers=headers, verify=False)
        net_obj_data = get_net.json()["items"][0]
        
    net_uuid = net_obj_data["id"]
    
    acp_url = f"https://{FMC_IP}/api/fmc_config/v1/domain/{domain_uuid}/policy/accesspolicies"
    acp_res = requests.get(acp_url, headers=headers, verify=False)
    acp_uuid = None
    for policy in acp_res.json().get("items", []):
        if policy["name"] == ACP_NAME:
            acp_uuid = policy["id"]
            break
            
    if not acp_uuid:
        print(f"[-] Policy {ACP_NAME} not found!")
        return
        
    rule_url = f"https://{FMC_IP}/api/fmc_config/v1/domain/{domain_uuid}/policy/accesspolicies/{acp_uuid}/accessrules?section=Mandatory&insertBefore=1"
    rule_payload = {
        "name": "Block_Malicious_Subnet",
        "action": "BLOCK",
        "enabled": True,
        "type": "AccessRule",
        "sourceNetworks": {
            "objects": [
                {
                    "id": net_uuid,
                    "type": "Network",
                    "name": MALICIOUS_NET_NAME
                }
            ]
        }
    }
    
    rule_res = requests.post(rule_url, headers=headers, json=rule_payload, verify=False)
    if rule_res.status_code == 201:
        print(f"[+] Successfully inserted Block_Malicious_Subnet rule to {ACP_NAME} at Rule #1!")
    else:
        print(f"[-] Failed to insert rule: {rule_res.status_code} - {rule_res.text}")

if __name__ == "__main__":
    add_deny_rule_to_acp()
```

#### 【検証方法】
FMC GUI で `Policies > Access Control > Access Control` を開き、`Lab_ACP` の Mandatory ルール 1 番目に `Block_Malicious_Subnet` (Action: Block) が挿入されていることを確認します。

---

### シナリオ 4: FMC - 変更の自動デプロイ (Deployment Request) スクリプト

#### 【問題】
FMC で保留中（Pending）となっている設定変更を持つ全デバイスを取得し、それらに対するデプロイ処理（Deployment Request）を API 経由でキックしなさい。

#### 【要件】
* `/deployment/deployabledevices` からデプロイ対象デバイスの UUID 一覧を取得。
* デプロイ対象が存在する場合、`/deployment/deploymentrequests` へ POST してデプロイを呼び出し、タスク ID を出力すること。

#### 【設定例 (Python スクリプト)】

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

FMC_IP = "192.168.10.5"
USERNAME = "admin"
PASSWORD = "LabPassword123"

def trigger_fmc_deployment():
    auth_res = requests.post(f"https://{FMC_IP}/api/fmc_platform/v1/auth/generatetoken", 
                             auth=(USERNAME, PASSWORD), verify=False)
    token = auth_res.headers.get("X-auth-access-token")
    domain_uuid = auth_res.headers.get("DOMAIN_UUID")
    headers = {"Content-Type": "application/json", "X-auth-access-token": token}
    
    pending_url = f"https://{FMC_IP}/api/fmc_config/v1/domain/{domain_uuid}/deployment/deployabledevices"
    res = requests.get(pending_url, headers=headers, verify=False)
    
    if res.status_code != 200 or "items" not in res.json():
        print("[*] No pending deployments found.")
        return
        
    device_uuids = []
    for item in res.json()["items"]:
        device_uuids.append(item["device"]["id"])
        print(f"[*] Found pending changes for Device: {item['device']['name']} (UUID: {item['device']['id']})")
        
    if not device_uuids:
        print("[*] No devices require deployment.")
        return
        
    deploy_url = f"https://{FMC_IP}/api/fmc_config/v1/domain/{domain_uuid}/deployment/deploymentrequests"
    deploy_payload = {
        "type": "DeploymentRequest",
        "version": str(int(requests.get(pending_url, headers=headers, verify=False).json()["items"][0]["version"])),
        "deviceList": device_uuids,
        "forceDeploy": True
    }
    
    deploy_res = requests.post(deploy_url, headers=headers, json=deploy_payload, verify=False)
    if deploy_res.status_code in [200, 201, 202]:
        task_id = deploy_res.json().get("metadata", {}).get("task", {}).get("id")
        print(f"[+] Deployment Initiated Successfully! Task ID: {task_id}")
    else:
        print(f"[-] Deployment Request Failed: {deploy_res.status_code} - {deploy_res.text}")

if __name__ == "__main__":
    trigger_fmc_deployment()
```

#### 【検証方法】
FMC GUI 右上の `Deploy` タスク通知アイコンを開き、デプロイタスクが実行中状態になっていることを確認します。

---

### シナリオ 5: ISE ERS API - 特定のエンドポイント MAC を Quarantine グループへ追加するスクリプト

#### 【問題】
Cisco ISE の ERS API (Port 9060) を呼び出し、不審な端末の MAC アドレス `00:11:22:33:44:55` を `Blacklist` エンドポイントグループへ自動登録しなさい。

#### 【要件】
* ISE IP: `192.168.10.20`, Port: `9060`
* ユーザー名: `ersadmin`, パスワード: `LabPassword123`
* `Accept: application/json` および `Content-Type: application/json` ヘッダーを指定すること。

#### 【設定例 (Python スクリプト)】

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

ISE_IP = "192.168.10.20"
USERNAME = "ersadmin"
PASSWORD = "LabPassword123"
TARGET_MAC = "00:11:22:33:44:55"
GROUP_NAME = "Blacklist"

def add_endpoint_to_ise():
    base_url = f"https://{ISE_IP}:9060/ers/config"
    headers = {
        "Accept": "application/json",
        "Content-Type": "application/json"
    }
    
    group_res = requests.get(
        f"{base_url}/endpointgroup?filter=name.EQ.{GROUP_NAME}",
        auth=(USERNAME, PASSWORD),
        headers=headers,
        verify=False
    )
    
    if group_res.status_code != 200 or not group_res.json()["SearchResult"]["resources"]:
        print(f"[-] Identity Group {GROUP_NAME} not found!")
        return
        
    group_uuid = group_res.json()["SearchResult"]["resources"][0]["id"]
    
    endpoint_payload = {
        "ERSEndpoint": {
            "name": TARGET_MAC,
            "mac": TARGET_MAC,
            "groupId": group_uuid,
            "description": "Quarantined via Python ERS API Script",
            "staticGroupAssignment": True
        }
    }
    
    create_res = requests.post(
        f"{base_url}/endpoint",
        auth=(USERNAME, PASSWORD),
        headers=headers,
        json=endpoint_payload,
        verify=False
    )
    
    if create_res.status_code == 201:
        print(f"[+] Successfully Quarantined MAC {TARGET_MAC} into Group '{GROUP_NAME}'!")
    else:
        print(f"[-] Failed to add endpoint: {create_res.status_code} - {create_res.text}")

if __name__ == "__main__":
    add_endpoint_to_ise()
```

#### 【検証方法】
ISE 管理画面の `Context Visibility > Endpoints` で MAC アドレス `00:11:22:33:44:55` を検索し、Endpoint Group が `Blacklist` になっていることを確認します。


### シナリオ 6: ISE ERS API - ゲストユーザーの自動動的発行スクリプト

#### 【問題】
ISE ERS API を利用し、名前 `Guest_User_01` の臨時ゲストアカウントを作成し、有効期間を 1 日に設定する Python スクリプトを作成しなさい。

#### 【要件】
* ISE IP: `192.168.10.20:9060`
* ゲストユーザー名: `Guest_User_01`
* パスワード: `TempPass12345`
* 登録完了後に生成された Guest User の ID を画面に出力すること。

#### 【設定例 (Python スクリプト)】

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

ISE_IP = "192.168.10.20"
USERNAME = "ersadmin"
PASSWORD = "LabPassword123"

def create_guest_user():
    url = f"https://{ISE_IP}:9060/ers/config/guestuser"
    headers = {
        "Accept": "application/json",
        "Content-Type": "application/json"
    }
    
    payload = {
        "GuestUser": {
            "name": "Guest_User_01",
            "guestAccessInfo": {
                "validDays": 1,
                "groupName": "Contractor (default)"
            },
            "guestType": "Contractor (default)",
            "personToVisit": "admin@example.com",
            "reasonForVisit": "Temporary Audit Vendor",
            "password": "TempPass12345",
            "portalId": "d32788e0-8c01-11e6-996c-005056bf0111"
        }
    }
    
    response = requests.post(url, auth=(USERNAME, PASSWORD), headers=headers, json=payload, verify=False)
    
    if response.status_code == 201:
        print("[+] Guest User 'Guest_User_01' Created Successfully!")
        print(f"    - Resource Location: {response.headers.get('Location')}")
    else:
        print(f"[-] Failed to Create Guest User: {response.status_code} - {response.text}")

if __name__ == "__main__":
    create_guest_user()
```

#### 【検証方法】
ISE GUI `Work Centers > Guest Access > Managing Guests > Guest Users` を開き、アカウント `Guest_User_01` が作成されていることを確認します。


### シナリオ 7: FDM REST API - スタティックルート自動設定スクリプト

#### 【問題】
FDM (Firepower Device Manager) 管理下の FTD に対し、REST API を用いてデフォルトゲートウェイ `10.1.1.254`（インターフェース `outside` 経由）のスタティックルートを自動登録しなさい。

#### 【要件】
* FDM IP: `192.168.10.10`
* 認証 API: `POST /api/fdm/v6/fdm/token` (Grant Type: `password`)
* Bearer Token を取得して以降のリクエストヘッダーへ設定すること。

#### 【設定例 (Python スクリプト)】

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

FDM_IP = "192.168.10.10"
USERNAME = "admin"
PASSWORD = "LabPassword123"

def add_fdm_static_route():
    token_url = f"https://{FDM_IP}/api/fdm/v6/fdm/token"
    token_data = {
        "grant_type": "password",
        "username": USERNAME,
        "password": PASSWORD
    }
    
    t_res = requests.post(token_url, data=token_data, verify=False)
    if t_res.status_code != 200:
        print("[-] FDM Authentication Failed")
        return
        
    access_token = t_res.json()["access_token"]
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json"
    }
    
    intf_url = f"https://{FDM_IP}/api/fdm/v6/devices/default/operational/interfaces"
    intf_res = requests.get(intf_url, headers=headers, verify=False)
    outside_id = None
    for item in intf_res.json()["items"]:
        if item.get("name") == "outside":
            outside_id = item["id"]
            break
            
    route_url = f"https://{FDM_IP}/api/fdm/v6/devices/default/routing/staticroutes"
    route_payload = {
        "name": "Default_Route_Outside",
        "networks": ["0.0.0.0/0"],
        "gateway": "10.1.1.254",
        "interface": {"id": outside_id, "type": "PhysicalInterface"},
        "metric": 1,
        "type": "staticroute"
    }
    
    r_res = requests.post(route_url, headers=headers, json=route_payload, verify=False)
    if r_res.status_code in [200, 201]:
        print("[+] FDM Static Route Added Successfully!")
    else:
        print(f"[-] Route Addition Failed: {r_res.status_code} - {r_res.text}")

if __name__ == "__main__":
    add_fdm_static_route()
```

#### 【検証方法】
FDM GUI の `Routing > Static Routes` でデフォルトルートが追加されていることを確認します。


### シナリオ 8: IOS-XE RESTCONF - インターフェース IP アドレス取得 & 設定更新スクリプト

#### 【問題】
IOS-XE ルータの RESTCONF API (`https://<IP>/restconf/data/...`) を使用して、`GigabitEthernet1` インターフェースの現在の IP アドレスとステータスを取得し、IP アドレスを `10.20.30.1/24` へ PATCH メソッドで変更しなさい。

#### 【要件】
* ルータ IP: `192.168.10.1`
* BASIC Authentication: `admin` / `LabPassword123`
* `Content-Type: application/yang-data+json` および `Accept: application/yang-data+json` を設定すること。

#### 【設定例 (Python スクリプト)】

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

ROUTER_IP = "192.168.10.1"
USERNAME = "admin"
PASSWORD = "LabPassword123"

def update_iosxe_interface_ip():
    base_url = f"https://{ROUTER_IP}/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet1"
    headers = {
        "Accept": "application/yang-data+json",
        "Content-Type": "application/yang-data+json"
    }
    
    get_res = requests.get(base_url, auth=(USERNAME, PASSWORD), headers=headers, verify=False)
    if get_res.status_code == 200:
        print("[+] Current Interface Config:")
        print(get_res.text)
    
    patch_payload = {
        "ietf-interfaces:interface": {
            "name": "GigabitEthernet1",
            "description": "Configured via Python RESTCONF Script",
            "type": "iana-if-type:ethernetCsmacd",
            "enabled": True,
            "ietf-ip:ipv4": {
                "address": [
                    {
                        "ip": "10.20.30.1",
                        "netmask": "255.255.255.0"
                    }
                ]
            }
        }
    }
    
    patch_res = requests.patch(base_url, auth=(USERNAME, PASSWORD), headers=headers, json=patch_payload, verify=False)
    
    if patch_res.status_code in [200, 204]:
        print("[+] IOS-XE GigabitEthernet1 IP Updated Successfully to 10.20.30.1/24!")
    else:
        print(f"[-] RESTCONF Update Failed: {patch_res.status_code} - {patch_res.text}")

if __name__ == "__main__":
    update_iosxe_interface_ip()
```

#### 【検証方法】
ルータ CLI で `show ip interface brief GigabitEthernet1` を実行し、IP アドレスが `10.20.30.1` に変更されていることを確認します。


### シナリオ 9: IOS-XE RESTCONF - アクセスリスト (ACL) 動的生成・適用スクリプト

#### 【問題】
IOS-XE ルータに対し、RESTCONF API を用いて IPv4 拡張 ACL `RESTCONF_BLOCK_ACL` を定義し、特定の宛先 IP `10.1.1.100` への HTTP (TCP 80) 通信を deny し、その他を permit するルールを設定しなさい。

#### 【要件】
* RESTCONF ターゲット: `/restconf/data/Cisco-IOS-XE-native:native/ip/access-list`
* ACL 名: `RESTCONF_BLOCK_ACL`

#### 【設定例 (Python スクリプト)】

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

ROUTER_IP = "192.168.10.1"
USERNAME = "admin"
PASSWORD = "LabPassword123"

def create_restconf_acl():
    url = f"https://{ROUTER_IP}/restconf/data/Cisco-IOS-XE-native:native/ip/access-list/extended=RESTCONF_BLOCK_ACL"
    headers = {
        "Accept": "application/yang-data+json",
        "Content-Type": "application/yang-data+json"
    }
    
    payload = {
        "Cisco-IOS-XE-native:extended": [
            {
                "name": "RESTCONF_BLOCK_ACL",
                "access-list-seq-rule": [
                    {
                        "sequence": "10",
                        "ace-rule": {
                            "action": "deny",
                            "protocol": "tcp",
                            "ipv4-address": "any",
                            "dst-ipv4-address": "10.1.1.100",
                            "dst-mask": "0.0.0.0",
                            "dst-eq": "80"
                        }
                    },
                    {
                        "sequence": "20",
                        "ace-rule": {
                            "action": "permit",
                            "protocol": "ip",
                            "ipv4-address": "any",
                            "dst-ipv4-address": "any"
                        }
                    }
                ]
            }
        ]
    }
    
    res = requests.put(url, auth=(USERNAME, PASSWORD), headers=headers, json=payload, verify=False)
    
    if res.status_code in [200, 201, 204]:
        print("[+] Extended ACL 'RESTCONF_BLOCK_ACL' Created Successfully via RESTCONF!")
    else:
        print(f"[-] ACL Creation Failed: {res.status_code} - {res.text}")

if __name__ == "__main__":
    create_restconf_acl()
```

#### 【検証方法】
ルータ CLI で `show ip access-lists RESTCONF_BLOCK_ACL` を実行し、定義されたシーケンス 10 および 20 を確認します。


### シナリオ 10: FMC - Security Intelligence (SI) カスタムリストへの脅威 IP 自動追加スクリプト

#### 【問題】
FMC REST API を使用して、カスタム Security Intelligence (SI) ネットワークリスト `SI_Block_List` に新しい脅威 IP アドレス `203.0.113.50` を動的追加しなさい。

#### 【要件】
* FMC IP: `192.168.10.5`
* FMC 既存の SI ネットワークリスト `SI_Block_List` の UUID を自動取得し、そのリクエストに対して IP アドレスを追加更新すること。

#### 【設定例 (Python スクリプト)】

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

FMC_IP = "192.168.10.5"
USERNAME = "admin"
PASSWORD = "LabPassword123"
SI_LIST_NAME = "SI_Block_List"
NEW_THREAT_IP = "203.0.113.50/32"

def update_si_custom_list():
    # 1. 認証
    auth_res = requests.post(f"https://{FMC_IP}/api/fmc_platform/v1/auth/generatetoken", 
                             auth=(USERNAME, PASSWORD), verify=False)
    token = auth_res.headers.get("X-auth-access-token")
    domain_uuid = auth_res.headers.get("DOMAIN_UUID")
    headers = {"Content-Type": "application/json", "X-auth-access-token": token}
    
    # 2. SI カスタムリストの検索
    si_url = f"https://{FMC_IP}/api/fmc_config/v1/domain/{domain_uuid}/object/securityintelligencenetworklists"
    res = requests.get(si_url, headers=headers, verify=False)
    
    si_uuid = None
    if res.status_code == 200 and "items" in res.json():
        for item in res.json()["items"]:
            if item["name"] == SI_LIST_NAME:
                si_uuid = item["id"]
                break
                
    if not si_uuid:
        print(f"[-] Security Intelligence List {SI_LIST_NAME} not found!")
        return
        
    # 3. SI リストの更新 (PUT)
    get_item_url = f"{si_url}/{si_uuid}"
    current_data = requests.get(get_item_url, headers=headers, verify=False).json()
    
    # 新規 IP の追加
    if "listContent" not in current_data:
        current_data["listContent"] = []
    current_data["listContent"].append({"value": NEW_THREAT_IP})
    
    put_res = requests.put(get_item_url, headers=headers, json=current_data, verify=False)
    if put_res.status_code in [200, 204]:
        print(f"[+] Successfully added Threat IP {NEW_THREAT_IP} to SI List '{SI_LIST_NAME}'!")
    else:
        print(f"[-] SI List Update Failed: {put_res.status_code} - {put_res.text}")

if __name__ == "__main__":
    update_si_custom_list()
```

#### 【検証方法】
FMC GUI の `Objects > Object Management > Security Intelligence > Network List and Feeds` で `SI_Block_List` 内に `203.0.113.50/32` が追加されていることを確認します。


## ❓ 想定試験問題

CCIE Security ラボ試験を想定したコンフィグ読解、トラブルシューティング、デザイン、および実装に関する問題を 6 問掲載します。

---

### 問 1 (実装・FMC REST API)
* **問題**: FMC から認証トークン (`X-auth-access-token`) およびドメイン UUID (`DOMAIN_UUID`) を取得する際、HTTP レスポンスのどこから取得すべきか答えなさい。
* **解答**: レスポンスボディ (JSON) ではなく、**HTTP レスポンスヘッダー (`response.headers`)** 内の `X-auth-access-token` および `DOMAIN_UUID` フィールドから取得する。

---

### 問 2 (トラブルシューティング・ISE ERS API)
* **問題**: ISE ERS API (Port 9060) にアクセスした際、Connection Refused エラーが発生しました。考えられる原因と対処法を 2 点説明しなさい。
* **解答**:
  1. ISE 上で ERS API 機能が無効化されているため、`Administration > System > Settings > ERS Settings` で `ERS (Read/Write)` を有効化する。
  2. ERS API サービスのポート番号 (9060) を指定していない、または経路上のファイアウォールでブロックされているため、Port 9060 への通信を許可する。

---

### 問 3 (実装・IOS-XE RESTCONF)
* **問題**: IOS-XE RESTCONF API でインターフェース更新時に `415 Unsupported Media Type` エラーが返されました。必要な HTTP ヘッダーを答えなさい。
* **解答**: `Content-Type: application/yang-data+json` および `Accept: application/yang-data+json` を設定する。

---

### 問 4 (コンフィグ読解・FMC デプロイ)
* **問題**: FMC に対して `/deployment/deploymentrequests` API を呼び出した際、返却される HTTP ステータスコードと意味を答えなさい。
* **解答**: `202 Accepted` が返される。デプロイ処理は非同期で実行されるため、タスク ID (`metadata.task.id`) を受け取り、別 API で進捗状況をポリング確認する。

---

### 問 5 (Design・エラーハンドリング)
* **問題**: 大量のオブジェクトをループで登録するスクリプトを作成する際の設計上の注意点を 3 点挙げなさい。
* **解答**:
  1. `try-except` 例外処理を組み込み、個別の登録失敗でスクリプトが停止しないようにする。
  2. FMC/ISE の API レート制限 (`429 Too Many Requests`) を避けるため、リクエスト間に `time.sleep()` を挟む。
  3. 可能であれば Bulk API（一括登録 API）を使用して HTTP オーバヘッドを削減する。

---

### 問 6 (実装・FDM REST API)
* **問題**: FDM REST API の認証トークン取得時の Endpoint URI とトークンの利用方法を答えなさい。
* **解答**: URI は `/api/fdm/v6/fdm/token` (POST)。取得した `access_token` を以降のリクエストで `Authorization: Bearer <access_token>` ヘッダーとして送信する。


## 🔗 参考リソース

CCIE Security 復習に役立つ Cisco 公式ドキュメントおよび API リファレンス一覧です。

* Cisco DevNet FMC REST API Quick Start Guide
* Cisco Firepower Management Center REST API Reference Guide
* Cisco Identity Services Engine (ISE) ERS API Reference Guide
* Cisco Firepower Device Manager (FDM) REST API Guide
* Cisco IOS-XE RESTCONF API Configuration Guide
* RFC 8040 - RESTCONF Protocol


---

## 📝 補足（Notes）

### 記憶定着のためのチートシート

| プラットフォーム | API 通信ポート | 認証方式 / トークン位置 | 特有の要件 / 注意点 |
| --- | --- | --- | --- |
| **Cisco FMC** | Port 443 (HTTPS) | 認証エンドポイント ──► レスポンスヘッダーの Access Token & Domain UUID | 設定変更後はデプロイ API の呼出が必要 |
| **Cisco ISE (ERS)** | Port 9060 (HTTPS) | Basic Authentication (要求ごとに送信) | ISE GUI の ERS Settings で有効化が必須。`Accept: application/json` ヘッダーが必須 |
| **Cisco FDM** | Port 443 (HTTPS) | 認証エンドポイント ──► レスポンスボディの Access Token (Bearer Token) | 設定変更後はデプロイ API の呼出が必要 |
| **IOS-XE (RESTCONF)** | Port 443 (HTTPS) | Basic Authentication | CLI で `restconf` 有効化が必要。ヘッダーに `application/yang-data+json` 指定が必要 |


