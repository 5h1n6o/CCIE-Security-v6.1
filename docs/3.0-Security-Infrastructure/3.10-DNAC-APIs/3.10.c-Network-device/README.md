---
layout: default
title: 3.10.c-Network-device
nav_order: 3
parent: 3.10-DNAC-APIs
grand_parent: 3.0-Security-Infrastructure
---

# 3.10.c Network device

## 📘 概要

Cisco DNA Center (Catalyst Center) Northbound Intent API における **Network Device API** は、DNAC がインベントリ管理および制御を行っている全ネットワーク機器（スイッチ、ルータ、アクセスポイント、WLC、ファイアウォールなど）の詳細情報（管理IPアドレス、MACアドレス、シリアル番号、IOS-XE/AireOSソフトウェアバージョン、リーチアビリティステータス、モジュール・スタック情報、物理/論理インターフェース詳細など）をプログラム経由で取得・操作するためのRESTfulインターフェース群です。

従来のネットワーク管理では、SNMP Polling や SSH CLI スクリプト（NetmikoやParamiko）を用いて個別の機器にログインし情報を収集していましたが、Cisco DNA Center の Network Device API を活用することで、DNA Center に集約された単一の真実のソース（Single Source of Truth）から数千台規模のネットワーク機器情報を統一された JSON 形式で瞬時に一括取得できます。

CCIE Security v6.1 ラボ試験においては、以下のようなセキュリティ運用・自動化シナリオで Network Device API の活用が直接問われます。
* **セキュリティ監査および脆弱性管理**: ネットワーク全機器のソフトウェアバージョンやシリアル番号を API 経由で取得し、未承認の旧バージョンや脆弱性が存在する機器を自動検出する。
* **動的セグメンテーションおよび隔離タスク**: リーチアビリティが低下した機器や、異常な挙動を示したスイッチポートの VLAN 変更・タグ付け（Tagging）を API 経由で実行し、ネットワークアクセス制限を自動化する。
* **インベントリとトポロジの自動同期**: Cisco ISE、FMC、SIEM、CMDB と DNA Center のインベントリ情報を同期し、IPアドレス/MACアドレス/ホスト名と物理スイッチポートの対応関係を自動マッピングする。


---

## 🔑 要点

Network Device API の主要な特徴、用途、制限事項、設計上の注意点を下表に整理します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | DNACに登録されている全ネットワーク機器の情報（インベントリ、ステータス、インターフェース、モジュール）を単一のREST APIエンドポイント群（`/dna/intent/api/v1/network-device`）で取得可能 |
| **用途** | ネットワークインベントリ自動収集、ソフトウェアバージョン/シリアル番号監査、機器リーチアビリティ監視、スイッチポートVLAN/ステータス自動制御、セキュリティタグ付与 |
| **メリット** | 個別機器へのSSH接続が不要、DNACのキャッシュデータベース参照により超高速応答、JSON形式での統一されたデータ処理、自動ページネーション対応 |
| **デメリット** | DNAC側で機器が「Unreachable」または「Unmanaged」状態の場合、一部の動的ステータス（ライブポート状態など）が最新でない場合がある |
| **対応機種** | Cisco Catalyst 9000 シリーズ（9200/9300/9400/9500/9600）、Catalyst 3850/3650、ISR/ASR ルータ、Catalyst 9800 WLC、AireOS WLC、Nexus スイッチ（一部）など |
| **制限事項** | 1リクエストあたりのデフォルト取得上限件数（limit=500など）が存在し、大量機器環境では `offset` パラメータによる繰り返しループ処理が必須 |
| **設計上の注意点** | API呼び出し時の認証ヘッダー `x-auth-token` の有効期限（1時間）の考慮、およびレスポンス内ステータス（`reachabilityStatus: Reachable`）の事前検証が不可欠 |


---

## 🏗 動作原理

DNAC Northbound Network Device API は、DNAC 内部のインベントリサービス（Inventory Database）と連携し、サウスバウンドプロトコル（SNMP/CLI/NETCONF/RESTCONF）を通じて収集・保持された機器情報をクライアント（Pythonスクリプト等）に提供します。

```
[ Python Script / REST Client ]
          │
          │ 1. HTTP GET /dna/intent/api/v1/network-device
          │    Header: x-auth-token: <token>
          ▼
[ Cisco DNA Center (Catalyst Center) ]
    ├── Northbound API Gateway (Kong / APIC-EM Gateway)
    ├── Inventory Microservice (PostgreSQL / MongoDB Cache)
    └── Southbound Collector (SNMP / CLI / NETCONF)
          │
          │ (バックグラウンドで継続収集・同期済み)
          ▼
[ Network Devices ] (Catalyst 9300, ISR 4400, Cat 9800 WLC, etc.)
```

Network Device API の主要エンドポイントと取得情報の関係は以下の通りです。

```
/dna/intent/api/v1/network-device
├── /count                               ── 登録機器の総件数を取得
├── /{id}                                ── デバイス UUID 指定で詳細取得
├── /ip-address/{ipAddress}              ── 管理 IP アドレス指定で詳細取得
├── /mac-address/{macAddress}            ── MAC アドレス指定で詳細取得
├── /serial-number/{serialNumber}        ── シリアル番号指定で詳細取得
└── /functional-capability               ── 機器のサポート機能（CLI/SNMP/TrustSec等）取得

/dna/intent/api/v1/interface
├── /network-device/{deviceId}           ── 特定機器の全物理/論理インターフェース取得
├── /network-device/{deviceId}/count     ── 特定機器のインターフェース総数取得
└── /by-ip                               ── IP アドレスから接続インターフェースを特定
```


---

## ⚙ 動作シーケンス

Python スクリプトが DNAC Network Device API を呼び出し、インベントリ情報をパースして特定の処理を行う際の一連のパケット/API処理シーケンスは以下の通りです。

```
[ Python Client ]                            [ Cisco DNA Center ]
       │                                              │
       │ 1. POST /dna/system/api/v1/auth/token        │
       │ ───────────────────────────────────────────> │
       │ 2. 200 OK (Token: eyJhbGci...)               │
       │ <─────────────────────────────────────────── │
       │                                              │
       │ 3. GET /dna/intent/api/v1/network-device     │
       │    Header: x-auth-token: eyJhbGci...         │
       │ ───────────────────────────────────────────> │
       │                                              │
       │    (DNAC 内部データベース検索)                  │
       │                                              │
       │ 4. 200 OK (JSON Payload: response: [{...}])  │
       │ <─────────────────────────────────────────── │
       │                                              │
       │ 5. レスポンスからデバイス ID (UUID) を抽出    │
       │    deviceId = "a1b2c3d4-..."                 │
       │                                              │
       │ 6. GET /dna/intent/api/v1/interface/        │
       │    network-device/{deviceId}                 │
       │ ───────────────────────────────────────────> │
       │                                              │
       │ 7. 200 OK (JSON Payload: response: [{...}])  │
       │ <─────────────────────────────────────────── │
       │                                              │
       │ 8. インターフェース状態 (portName, status,   │
       │    vlanId) のフィルタリング・レポート出力     │
```


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、Cisco DNA Center の Northbound API（特に Network Device API）に関する問題が出題された場合、単なる情報取得だけでなく、**取得したデータに対する複合的なロジック処理や条件分岐スクリプトの作成**が求められます。

### Blueprintで重要なポイント
1. **特定条件によるデバイスフィルタリング**:
   * 管理 IP アドレス（`managementIpAddress`）、シリアル番号（`serialNumber`）、ホスト名（`hostname`）、デバイス役割（`role`: ACCESS, CORE, DISTRIBUTION）、製品ファミリー（`family`: Switches and Hubs, Routers, Wireless Controller）に基づくクエリ制御。
2. **機器ヘルスおよび状態の特定**:
   * リーチアビリティ（`reachabilityStatus`: `Reachable` vs `Unreachable`）および管理状態（`managementState`: `Managed` vs `Unmanaged`）の検証。
   * ラボ試験では、「DNACに登録されているが Reachable でないスイッチの一覧を抽出し、CSV/JSON ファイルに出力する」といった自動化課題が出題される可能性があります。
3. **デバイス ID (UUID) の動的ルックアップとインターフェース連携**:
   * DNAC API の多くの操作（インターフェース取得、トポロジ取得、テンプレート適用等）は、機器の IP アドレスではなく内部分割識別子である **UUID (`id`)** を要求します。
   * IP アドレスから UUID を取得し、その UUID を使用して二次 API（`/interface/network-device/{deviceId}`）を呼び出す連鎖処理（API Chaining）が必須となります。
4. **大量データのページネーション処理**:
   * リクエストクエリに `offset=1` と `limit=500` を指定し、全デバイスを取得する反復ループ処理の実装。

### よくある設定・実装ミス
* **API レスポンスキーの誤認**:
   * デバイス一覧のルートキーは `response` リストです（例: `data['response']`）。直下にリストが配置されていないため、キーの指定漏れで `KeyError` が発生します。
* **`id` と `instanceUuid` の混同**:
   * デバイスの固有識別子は `id` フィールドに格納されています。他のCisco API（FMCなど）とフィールド名が異なる点に注意してください。
* **SSL 証明書検証の未処理**:
   * ラボ環境の DNAC は自己署名証明書を使用していることが多いため、`requests.get(..., verify=False)` の設定および `urllib3.disable_warnings()` による警告抑止を忘れるとスクリプトが異常終了することがあります。


---

## 🛠 設定方法

DNAC Network Device API を Python から実行するための標準的なライブラリ構成、ヘッダー定義、および基本リクエスト構造は以下の通りです。

```python
import requests
import json
import urllib3

# SSL警告の抑止
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# DNAC 接続情報
DNAC_URL = "https://dnac.lab.local"
USERNAME = "admin"
PASSWORD = "Password123!"

def get_auth_token():
    url = f"{DNAC_URL}/dna/system/api/v1/auth/token"
    headers = {"Content-Type": "application/json"}
    response = requests.post(url, auth=(USERNAME, PASSWORD), headers=headers, verify=False)
    response.raise_for_status()
    return response.json()["Token"]

def get_network_devices(token, limit=500, offset=1):
    url = f"{DNAC_URL}/dna/intent/api/v1/network-device"
    headers = {
        "Content-Type": "application/json",
        "x-auth-token": token
    }
    params = {
        "limit": limit,
        "offset": offset
    }
    response = requests.get(url, headers=headers, params=params, verify=False)
    response.raise_for_status()
    return response.json()["response"]
```


## 🔍 検証コマンド

Cisco DNA Center の Network Device API および該当機器の動作確認用コマンド・APIレスポンス確認方法を下表に整理します。

| 目的 | コマンド / エンドポイント |
| --- | --- |
| **登録デバイス数確認** | `GET /dna/intent/api/v1/network-device/count` |
| **全デバイス一覧取得** | `GET /dna/intent/api/v1/network-device?limit=500&offset=1` |
| **IP指定デバイス詳細取得** | `GET /dna/intent/api/v1/network-device/ip-address/{ipAddress}` |
| **UUID指定デバイス詳細取得** | `GET /dna/intent/api/v1/network-device/{id}` |
| **デバイス別インターフェース取得** | `GET /dna/intent/api/v1/interface/network-device/{deviceId}` |
| **デバイス機能（Capability）確認** | `GET /dna/intent/api/v1/network-device/functional-capability` |
| **DNAC CLIでのインベントリサービス確認** | `sudo maglev service status inventory-manager` |
| **スイッチ側でのDNAC接続確認 (IOS-XE)** | `<code>show cdp neighbors</code>` / `<code>show ip http client status</code>` |


---

## 🚨 トラブルシュート

DNAC Network Device API 実行時に発生しやすい問題、原因、確認 API / コマンド、および解決策を下表にまとめます。

| 症状 | 原因 | 確認コマンド / API | 対処方法 |
| --- | --- | --- | --- |
| **401 Unauthorized** | 認証トークンが無効または有効期限切れ（1時間経過） | `POST /dna/system/api/v1/auth/token` のレスポンス確認 | `x-auth-token` を再取得し、ヘッダーに正しくセットして再実行する |
| **404 Not Found** | リクエスト URL のタイポ、または指定した Device UUID / IP アドレスが存在しない | `GET /dna/intent/api/v1/network-device` | 指定した UUID または IP アドレスがインベントリ内に正しく存在するか確認する |
| **空のリスト `[]` が返る** | フィルタクエリ（`managementIpAddress` や `serialNumber`）のミス | クエリ引数の文字列確認 | 指定文字列（大文字小文字や空白の有無）を正確に一致させる |
| **一部機器が `Unreachable`** | スイッチ側の SNMP/CLI 応答不可、DNAC からの通信拒否 | スイッチ側: `<code>show run | inc snmp</code>` / `<code>show ip access-list</code>` | DNAC とスイッチ間の ICMP / SNMP (161) / SSH (22) 通信の ACL および認証情報を確認する |
| **レスポンス件数が不足** | デフォルト上限（500件）による取得打ち切り | レスポンス件数の `len(response['response'])` 確認 | `offset` と `limit` パラメータを使用した自動ループ取得を実装する |


---

## ⚠ 制限事項

1. **API レートリミット（Throttling）**:
   * DNAC の Northbound API にはリクエスト数のレートリミットが設定されており、短時間に大量のパラレルリクエストを送信すると `429 Too Many Requests` エラーが返されます。
2. **機器管理状態による取得データの制限**:
   * DNAC に追加されたばかりの機器や `Unmanaged` 状態の機器では、インターフェース詳細（`vlanId` や `portMode` 等）が空（Null）として返される場合があります。
3. **バージョン間のエンドポイント非互換性**:
   * Cisco DNA Center 2.x から 2.3.x へのバージョンアップに伴い、一部のレガシー API パスが非推奨（Deprecated）となり、Catalyst Center ブランドへの統合に伴うパス変更が順次進んでいる点に留意が必要です。


---

## 🔄 他技術との関連

* **Cisco ISE (Identity Services Engine)**:
  * DNAC は Network Device API で取得したインベントリ情報を pxGrid 経由で Cisco ISE と共有し、Network Access Control (802.1X / MAB) の認証対象スイッチとして自動登録・同期します。
* **Cisco FMC (Firepower Management Center)**:
  * DNAC のトポロジ・デバイス情報を FMC の Security Intelligence や Access Control Policy と連携させ、境界ファイアウォールでの動的防御ポリシーに反映させます。
* **Python 自動化スクリプト**:
  * Network Device API で取得した `id` (UUID) をトリガーとして、Command Runner API や Template Programmer API を自動実行し、大規模な一括コンフィグ変更を実現します。


---

## 🧩 比較表

ネットワーク機器情報の収集・管理における各種手法の比較を下表に示します。

| 比較項目 | DNAC Northbound API | 従来型 CLI / SSH (Netmiko) | SNMP Polling | RESTCONF / NETCONF |
| --- | --- | --- | --- | --- |
| **データ取得先** | DNAC 単一コントローラ | 各ネットワーク機器個別 | 各ネットワーク機器個別 | 各ネットワーク機器個別 |
| **データフォーマット** | JSON | 非構造化テキスト (CLI) | MIB OID / BER | JSON / XML |
| **処理速度・負荷** | 超高速（DNACキャッシュ参照） | 遅い（機器ごとにSSH確立） | 中程度（UDP通信） | 高速（機器処理能力に依存） |
| **セキュリティ** | HTTPS (TLS 1.2/1.3) + Token | SSH v2 (Password/Key) | SNMPv3 (USM) | HTTPS / SSH |
| **事前設定コスト** | 低（DNACに機器追加のみ） | 高（全機器個別のアクセス権設定） | 高（全機器のMIB/Community設定） | 高（全機器でRESTCONF有効化） |


---

## 💡 ベストプラクティス

1. **認証トークンの再利用と自動リフレッシュ**:
   * リクエストごとにトークン取得 API を呼び出すのではなく、1 時間の有効期限内は取得済みトークンを使い回し、`401 Unauthorized` を検知した際のみ自動再取得する設計とする。
2. **頑健なエラーハンドリングの実装**:
   * `response.raise_for_status()` を呼び出し、HTTP ステータスコードが 2xx 以外の場合に例外処理（`try-except`）で適切にエラー理由を出力する。
3. **安全な資格情報管理**:
   * スクリプト内に DNAC のユーザー名やパスワードをハードコードせず、環境変数（`os.environ`）や暗号化された設定ファイルから読み込む。
4. **構造化された JSON パーサーの活用**:
   * レスポンスの `response` リストを安全にルックアップし、キーが存在しない場合に備えて `.get('key', 'N/A')` メソッドを使用する。


## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルの実践的な Python スクリプト演習例 10 選です。各コード例は省略せず、認証・例外処理・レスポンス解析まで含めた完全なスクリプトとして記述しています。

---

### 演習例1: 全ネットワークデバイスの基本インベントリ一覧取得

#### 問題
Cisco DNA Center に登録されている全ネットワーク機器の一覧を取得し、ホスト名、管理 IP アドレス、プラットフォーム、ソフトウェアバージョン、リーチアビリティステータスをコンソールに出力する Python スクリプトを作成しなさい。

#### 要件
* `GET /dna/intent/api/v1/network-device` を使用すること。
* SSL 証明書検証エラーを無視すること。
* 取得結果を整列されたテキスト形式で表示すること。

#### 設定例（Python スクリプト）
```python
import requests
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_URL = "https://dnac.lab.local"
USERNAME = "admin"
PASSWORD = "Password123!"

def get_auth_token():
    url = f"{DNAC_URL}/dna/system/api/v1/auth/token"
    headers = {"Content-Type": "application/json"}
    response = requests.post(url, auth=(USERNAME, PASSWORD), headers=headers, verify=False)
    response.raise_for_status()
    return response.json()["Token"]

def print_device_inventory():
    try:
        token = get_auth_token()
        url = f"{DNAC_URL}/dna/intent/api/v1/network-device"
        headers = {
            "Content-Type": "application/json",
            "x-auth-token": token
        }
        
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        
        devices = response.json().get("response", [])
        
        print(f"{'Hostname':<25} {'Management IP':<15} {'Platform':<15} {'Version':<15} {'Status':<10}")
        print("=" * 85)
        
        for dev in devices:
            hostname = dev.get("hostname", "N/A")
            ip = dev.get("managementIpAddress", "N/A")
            platform = dev.get("platformId", "N/A")
            version = dev.get("softwareVersion", "N/A")
            status = dev.get("reachabilityStatus", "N/A")
            
            print(f"{hostname:<25} {ip:<15} {platform:<15} {version:<15} {status:<10}")
            
    except requests.exceptions.RequestException as e:
        print(f"Error executing API request: {e}")

if __name__ == "__main__":
    print_device_inventory()
```

#### 検証方法
1. スクリプトを実行し、`Hostname` や `Management IP` が正しいフォーマットで整列出力されるか確認する。
2. 出力されたリーチアビリティステータスが `Reachable` であるか確認する。

---

### 演習例2: 管理 IP アドレス指定による特定のデバイス詳細および UUID 取得

#### 問題
指定した管理 IP アドレス（例: `10.1.1.1`）を持つネットワーク機器の詳細情報を検索し、そのデバイスの固有識別子 (UUID)、シリアル番号、および稼働時間 (Up Time) を表示する Python スクリプトを作成しなさい。

#### 要件
* エンドポイント `GET /dna/intent/api/v1/network-device/ip-address/{ipAddress}` を使用すること。
* 指定 IP が存在しない場合の例外処理を組み込むこと。

#### 設定例（Python スクリプト）
```python
import requests
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_URL = "https://dnac.lab.local"
USERNAME = "admin"
PASSWORD = "Password123!"

def get_auth_token():
    url = f"{DNAC_URL}/dna/system/api/v1/auth/token"
    response = requests.post(url, auth=(USERNAME, PASSWORD), headers={"Content-Type": "application/json"}, verify=False)
    response.raise_for_status()
    return response.json()["Token"]

def get_device_by_ip(target_ip):
    token = get_auth_token()
    url = f"{DNAC_URL}/dna/intent/api/v1/network-device/ip-address/{target_ip}"
    headers = {"x-auth-token": token, "Content-Type": "application/json"}
    
    try:
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        
        device_data = response.json().get("response", {})
        
        print(f"=== Device Detail for IP: {target_ip} ===")
        print(f"Device UUID    : {device_data.get('id')}")
        print(f"Hostname       : {device_data.get('hostname')}")
        print(f"Serial Number  : {device_data.get('serialNumber')}")
        print(f"MAC Address    : {device_data.get('macAddress')}")
        print(f"Role           : {device_data.get('role')}")
        print(f"Up Time        : {device_data.get('upTime')}")
        print(f"Family         : {device_data.get('family')}")
        
    except requests.exceptions.HTTPError as err:
        if err.response.status_code == 404:
            print(f"Device with IP {target_ip} not found in DNAC inventory.")
        else:
            print(f"HTTP Error occurred: {err}")

if __name__ == "__main__":
    get_device_by_ip("10.1.1.1")
```

#### 検証方法
1. スクリプトに存在する管理 IP アドレスを与えて実行し、UUID (`id`) が正常に表示されるか確認する。
2. 存在しない IP アドレスを入力し、404 エラーハンドリングが正しく機能するか検証する。

---

### 演習例3: 特定デバイスの全物理・論理インターフェース情報取得と Down ポート抽出

#### 問題
特定のデバイス IP（`10.1.1.1`）に接続されている全インターフェース情報を取得し、現在ステータスが `down` であるアクセスポート（VLAN指定あり）の一覧を出力する Python スクリプトを作成しなさい。

#### 要件
* 最初に IP から Device UUID を検索し、次に `GET /dna/intent/api/v1/interface/network-device/{deviceId}` を呼び出す連鎖リクエスト（API Chaining）を実装すること。

#### 設定例（Python スクリプト）
```python
import requests
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_URL = "https://dnac.lab.local"
USERNAME = "admin"
PASSWORD = "Password123!"

def get_auth_token():
    url = f"{DNAC_URL}/dna/system/api/v1/auth/token"
    resp = requests.post(url, auth=(USERNAME, PASSWORD), headers={"Content-Type": "application/json"}, verify=False)
    resp.raise_for_status()
    return resp.json()["Token"]

def check_down_interfaces(device_ip):
    token = get_auth_token()
    headers = {"x-auth-token": token, "Content-Type": "application/json"}
    
    url_dev = f"{DNAC_URL}/dna/intent/api/v1/network-device/ip-address/{device_ip}"
    resp_dev = requests.get(url_dev, headers=headers, verify=False)
    resp_dev.raise_for_status()
    device_id = resp_dev.json()["response"]["id"]
    
    url_intf = f"{DNAC_URL}/dna/intent/api/v1/interface/network-device/{device_id}"
    resp_intf = requests.get(url_intf, headers=headers, verify=False)
    resp_intf.raise_for_status()
    interfaces = resp_intf.json().get("response", [])
    
    print(f"=== Down Interfaces on Device {device_ip} (UUID: {device_id}) ===")
    print(f"{'Interface Name':<20} {'Admin Status':<12} {'VLAN ID':<10} {'Port Mode':<12}")
    print("-" * 60)
    
    for intf in interfaces:
        status = intf.get("status", "").lower()
        if status == "down":
            name = intf.get("portName", "N/A")
            admin_status = intf.get("adminStatus", "N/A")
            vlan = intf.get("vlanId", "N/A")
            mode = intf.get("portMode", "N/A")
            print(f"{name:<20} {admin_status:<12} {vlan:<10} {mode:<12}")

if __name__ == "__main__":
    check_down_interfaces("10.1.1.1")
```

#### 検証方法
1. 対象スイッチのインターフェース一覧から `down` 状態のポートが正しく抽出されて表示されることを確認する。

---

### 演習例4: リーチアビリティ不調機器 (Unreachable) の自動検出とアラート生成

#### 問題
DNAC インベントリ内でリーチアビリティステータスが `Unreachable` または管理ステータスが `Unmanaged` になっている不調機器を自動検知し、障害通知メッセージを作成する Python スクリプトを作成しなさい。

#### 要件
* `GET /dna/intent/api/v1/network-device` を使用。
* 対象機器のホスト名、管理 IP、最終更新日時を出力に含めること。

#### 設定例（Python スクリプト）
```python
import requests
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_URL = "https://dnac.lab.local"
USERNAME = "admin"
PASSWORD = "Password123!"

def get_auth_token():
    url = f"{DNAC_URL}/dna/system/api/v1/auth/token"
    res = requests.post(url, auth=(USERNAME, PASSWORD), headers={"Content-Type": "application/json"}, verify=False)
    res.raise_for_status()
    return res.json()["Token"]

def detect_unreachable_devices():
    token = get_auth_token()
    url = f"{DNAC_URL}/dna/intent/api/v1/network-device"
    headers = {"x-auth-token": token, "Content-Type": "application/json"}
    
    res = requests.get(url, headers=headers, verify=False)
    res.raise_for_status()
    devices = res.json().get("response", [])
    
    unreachable_list = []
    
    for dev in devices:
        reachability = dev.get("reachabilityStatus", "")
        mgmt_state = dev.get("managementState", "")
        
        if reachability == "Unreachable" or mgmt_state == "Unmanaged":
            unreachable_list.append({
                "hostname": dev.get("hostname"),
                "ip": dev.get("managementIpAddress"),
                "reachability": reachability,
                "mgmtState": mgmt_state,
                "type": dev.get("type")
            })
            
    print(f"Unreachable Devices Alert Count: {len(unreachable_list)}")
    print("=" * 70)
    for alert_dev in unreachable_list:
        print(f"ALERT: {alert_dev['hostname']} ({alert_dev['ip']}) - Reachability: {alert_dev['reachability']}, MgmtState: {alert_dev['mgmtState']}")

if __name__ == "__main__":
    detect_unreachable_devices()
```

#### 検証方法
1. スクリプトを実行し、ネットワーク通信が遮断されている機器が `ALERT` ログとして正しく出力されるか確認する。

---

### 演習例5: 監査用全機器シリアル番号・IOS-XE バージョンレポートの CSV 出力

#### 問題
DNAC に登録されているすべてのスイッチおよびルータの「ホスト名」「管理IP」「プラットフォーム」「シリアル番号」「OSバージョン」を取得し、`inventory_audit.csv` ファイルに保存する Python スクリプトを作成しなさい。

#### 要件
* Python 標準の `csv` モジュールを使用すること。
* データヘッダーを含む正しい CSV 形式で保存すること。

#### 設定例（Python スクリプト）
```python
import requests
import json
import csv
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_URL = "https://dnac.lab.local"
USERNAME = "admin"
PASSWORD = "Password123!"

def get_auth_token():
    url = f"{DNAC_URL}/dna/system/api/v1/auth/token"
    res = requests.post(url, auth=(USERNAME, PASSWORD), headers={"Content-Type": "application/json"}, verify=False)
    res.raise_for_status()
    return res.json()["Token"]

def export_inventory_csv(filename="inventory_audit.csv"):
    token = get_auth_token()
    url = f"{DNAC_URL}/dna/intent/api/v1/network-device"
    headers = {"x-auth-token": token, "Content-Type": "application/json"}
    
    res = requests.get(url, headers=headers, verify=False)
    res.raise_for_status()
    devices = res.json().get("response", [])
    
    with open(filename, mode="w", newline="", encoding="utf-8") as file:
        writer = csv.writer(file)
        writer.writerow(["Hostname", "Management IP", "Platform", "Serial Number", "Software Version", "Role"])
        
        for dev in devices:
            writer.writerow([
                dev.get("hostname", ""),
                dev.get("managementIpAddress", ""),
                dev.get("platformId", ""),
                dev.get("serialNumber", ""),
                dev.get("softwareVersion", ""),
                dev.get("role", "")
            ])
            
    print(f"Successfully exported {len(devices)} device records to '{filename}'.")

if __name__ == "__main__":
    export_inventory_csv()
```

#### 検証方法
1. スクリプト実行後、生成された `inventory_audit.csv` の内容を確認し、全機器のインベントリデータが正確にCSV化されていることを検証する。


### 演習例6: ページネーション (offset/limit) による全大量デバイス取得ループの実装

#### 問題
DNAC に登録されている機器数が 500 台を超える大規模環境を想定し、`limit` と `offset` パラメータを使用した繰り返しループによって、全ネットワーク機器のインベントリ漏れなく取得する Python スクリプトを作成しなさい。

#### 要件
* 1リクエストの取得上限 `limit=500` とし、取得件数が 0 になるまで `offset` を更新して再帰ループすること。

#### 設定例（Python スクリプト）
```python
import requests
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_URL = "https://dnac.lab.local"
USERNAME = "admin"
PASSWORD = "Password123!"

def get_auth_token():
    url = f"{DNAC_URL}/dna/system/api/v1/auth/token"
    res = requests.post(url, auth=(USERNAME, PASSWORD), headers={"Content-Type": "application/json"}, verify=False)
    res.raise_for_status()
    return res.json()["Token"]

def get_all_devices_paginated():
    token = get_auth_token()
    headers = {"x-auth-token": token, "Content-Type": "application/json"}
    
    all_devices = []
    limit = 500
    offset = 1
    
    while True:
        url = f"{DNAC_URL}/dna/intent/api/v1/network-device?limit={limit}&offset={offset}"
        res = requests.get(url, headers=headers, verify=False)
        res.raise_for_status()
        
        batch = res.json().get("response", [])
        if not batch:
            break
            
        all_devices.extend(batch)
        print(f"Fetched batch of {len(batch)} devices (Offset: {offset})")
        offset += limit
        
    print(f"Total Network Devices fetched: {len(all_devices)}")
    return all_devices

if __name__ == "__main__":
    get_all_devices_paginated()
```

#### 検証方法
1. スクリプトを実行し、`Fetched batch...` ログが出力され、ページの終わりまで漏れなく取得完了することを確認する。

---

### 演習例7: デバイスの役割 (Role) 別フィルタリングと機器台数カウント

#### 問題
DNAC 内のデバイスを役割（`role`: `ACCESS`, `CORE`, `DISTRIBUTION`, `BORDER ROUTER`）ごとに分類し、役割ごとの機器台数および各役割に属するホスト名一覧を表示する Python スクリプトを作成しなさい。

#### 要件
* `GET /dna/intent/api/v1/network-device` を使用。
* Python の辞書（`dict`）型を用いてロール別に自動マッピングすること。

#### 設定例（Python スクリプト）
```python
import requests
import json
import urllib3
from collections import defaultdict

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_URL = "https://dnac.lab.local"
USERNAME = "admin"
PASSWORD = "Password123!"

def get_auth_token():
    url = f"{DNAC_URL}/dna/system/api/v1/auth/token"
    res = requests.post(url, auth=(USERNAME, PASSWORD), headers={"Content-Type": "application/json"}, verify=False)
    res.raise_for_status()
    return res.json()["Token"]

def summarize_devices_by_role():
    token = get_auth_token()
    url = f"{DNAC_URL}/dna/intent/api/v1/network-device"
    headers = {"x-auth-token": token, "Content-Type": "application/json"}
    
    res = requests.get(url, headers=headers, verify=False)
    res.raise_for_status()
    devices = res.json().get("response", [])
    
    role_map = defaultdict(list)
    
    for dev in devices:
        role = dev.get("role", "UNKNOWN")
        hostname = dev.get("hostname", "N/A")
        ip = dev.get("managementIpAddress", "N/A")
        role_map[role].append(f"{hostname} ({ip})")
        
    print("=== Network Device Distribution by Role ===")
    for role, dev_list in role_map.items():
        print(f"\nRole: {role} (Count: {len(dev_list)})")
        for dev_info in dev_list:
            print(f"  - {dev_info}")

if __name__ == "__main__":
    summarize_devices_by_role()
```

#### 検証方法
1. 各役割（`ACCESS`, `CORE` 等）別に正しく分類され、該当機器のホスト名と IP が表示されるか確認する。

---

### 演習例8: 特定デバイスの機能 (Functional Capability) 取得スクリプト

#### 問題
指定したデバイスのサポート機能（例: TrustSec サポート、MACsec 機能、CLI サポート等）を取得し、表示する Python スクリプトを作成しなさい。

#### 要件
* `GET /dna/intent/api/v1/network-device/functional-capability` エンドポイントを利用すること。

#### 設定例（Python スクリプト）
```python
import requests
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_URL = "https://dnac.lab.local"
USERNAME = "admin"
PASSWORD = "Password123!"

def get_auth_token():
    url = f"{DNAC_URL}/dna/system/api/v1/auth/token"
    res = requests.post(url, auth=(USERNAME, PASSWORD), headers={"Content-Type": "application/json"}, verify=False)
    res.raise_for_status()
    return res.json()["Token"]

def get_functional_capabilities(device_ip):
    token = get_auth_token()
    headers = {"x-auth-token": token, "Content-Type": "application/json"}
    
    url_dev = f"{DNAC_URL}/dna/intent/api/v1/network-device/ip-address/{device_ip}"
    resp_dev = requests.get(url_dev, headers=headers, verify=False)
    resp_dev.raise_for_status()
    device_id = resp_dev.json()["response"]["id"]
    
    url_cap = f"{DNAC_URL}/dna/intent/api/v1/network-device/functional-capability?deviceId={device_id}"
    resp_cap = requests.get(url_cap, headers=headers, verify=False)
    resp_cap.raise_for_status()
    capabilities = resp_cap.json().get("response", [])
    
    print(f"=== Functional Capabilities for Device {device_ip} ===")
    for cap in capabilities:
        cap_name = cap.get("functionName", "N/A")
        cap_details = cap.get("functionDetails", {})
        print(f"Capability: {cap_name}")
        print(f"  Details: {json.dumps(cap_details, indent=2)}")

if __name__ == "__main__":
    get_functional_capabilities("10.1.1.1")
```

#### 検証方法
1. スクリプトを実行し、機器の機能属性が取得・表示されることを確認する。

---

### 演習例9: デバイスのモジュール (Modules / Line Cards / Power Supply) 情報取得

#### 問題
スイッチに挿入されているファン、電源ユニット、ラインカード、SFPトランシーバー等のモジュール情報を取得する Python スクリプトを作成しなさい。

#### 要件
* エンドポイント `GET /dna/intent/api/v1/network-device/module` を使用すること。

#### 設定例（Python スクリプト）
```python
import requests
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_URL = "https://dnac.lab.local"
USERNAME = "admin"
PASSWORD = "Password123!"

def get_auth_token():
    url = f"{DNAC_URL}/dna/system/api/v1/auth/token"
    res = requests.post(url, auth=(USERNAME, PASSWORD), headers={"Content-Type": "application/json"}, verify=False)
    res.raise_for_status()
    return res.json()["Token"]

def get_hardware_modules(device_ip):
    token = get_auth_token()
    headers = {"x-auth-token": token, "Content-Type": "application/json"}
    
    url_dev = f"{DNAC_URL}/dna/intent/api/v1/network-device/ip-address/{device_ip}"
    resp_dev = requests.get(url_dev, headers=headers, verify=False)
    resp_dev.raise_for_status()
    device_id = resp_dev.json()["response"]["id"]
    
    url_mod = f"{DNAC_URL}/dna/intent/api/v1/network-device/module?deviceId={device_id}"
    resp_mod = requests.get(url_mod, headers=headers, verify=False)
    resp_mod.raise_for_status()
    modules = resp_mod.json().get("response", [])
    
    print(f"=== Installed Modules for Device {device_ip} ===")
    print(f"{'Module Name':<30} {'Part Number':<20} {'Serial Number':<20}")
    print("-" * 70)
    
    for mod in modules:
        name = mod.get("name", "N/A")
        part_no = mod.get("partNumber", "N/A")
        serial = mod.get("serialNumber", "N/A")
        print(f"{name:<30} {part_no:<20} {serial:<20}")

if __name__ == "__main__":
    get_hardware_modules("10.1.1.1")
```

#### 検証方法
1. スクリプトを実行し、ラインカードや電源などのモジュール情報が表形式で表示されることを確認する。

---

### 演習例10: デバイス IP からインターフェース詳細および接続パラメータを特定する統合 API チェーンスクリプト

#### 問題
管理 IP アドレスを与えると、その機器の基本情報（ホスト名・バージョン）、および全インターフェースの「MACアドレス」「速度（Speed）」「デュプレックス（Duplex）」「VLAN」を一覧表示する完全な統合 Python スクリプトを作成しなさい。

#### 要件
* トークン自動取得、IPからのUUID解決、インターフェース取得の3段階チェーン処理を行うこと。
* HTTPエラーが発生した場合に適切なメッセージを出力して正常終了すること。

#### 設定例（Python スクリプト）
```python
import requests
import json
import urllib3
import sys

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_URL = "https://dnac.lab.local"
USERNAME = "admin"
PASSWORD = "Password123!"

class DNACClient:
    def __init__(self, base_url, username, password):
        self.base_url = base_url
        self.username = username
        self.password = password
        self.token = None
        self.authenticate()

    def authenticate(self):
        url = f"{self.base_url}/dna/system/api/v1/auth/token"
        headers = {"Content-Type": "application/json"}
        try:
            res = requests.post(url, auth=(self.username, self.password), headers=headers, verify=False)
            res.raise_for_status()
            self.token = res.json()["Token"]
            print("Successfully authenticated with Cisco DNA Center.")
        except Exception as e:
            print(f"Authentication failed: {e}")
            sys.exit(1)

    def get_device_info_and_interfaces(self, ip_address):
        headers = {"x-auth-token": self.token, "Content-Type": "application/json"}
        
        dev_url = f"{self.base_url}/dna/intent/api/v1/network-device/ip-address/{ip_address}"
        try:
            res_dev = requests.get(dev_url, headers=headers, verify=False)
            res_dev.raise_for_status()
            dev_data = res_dev.json().get("response", {})
        except requests.exceptions.HTTPError as err:
            print(f"Error fetching device {ip_address}: {err}")
            return
            
        device_id = dev_data.get("id")
        print(f"\nDevice: {dev_data.get('hostname')} | Platform: {dev_data.get('platformId')} | Version: {dev_data.get('softwareVersion')}")
        print(f"Device UUID: {device_id}")
        
        intf_url = f"{self.base_url}/dna/intent/api/v1/interface/network-device/{device_id}"
        try:
            res_intf = requests.get(intf_url, headers=headers, verify=False)
            res_intf.raise_for_status()
            interfaces = res_intf.json().get("response", [])
        except requests.exceptions.HTTPError as err:
            print(f"Error fetching interfaces for device UUID {device_id}: {err}")
            return
            
        print(f"\n{'Port Name':<20} {'Status':<8} {'Speed':<10} {'Duplex':<10} {'VLAN':<8} {'MAC Address':<18}")
        print("=" * 80)
        
        for intf in interfaces:
            port = intf.get("portName", "N/A")
            status = intf.get("status", "N/A")
            speed = intf.get("speed", "N/A")
            duplex = intf.get("duplex", "N/A")
            vlan = intf.get("vlanId", "N/A")
            mac = intf.get("macAddress", "N/A")
            print(f"{port:<20} {status:<8} {speed:<10} {duplex:<10} {vlan:<8} {mac:<18}")

if __name__ == "__main__":
    client = DNACClient(DNAC_URL, USERNAME, PASSWORD)
    client.get_device_info_and_interfaces("10.1.1.1")
```

#### 検証方法
1. スクリプトを実行し、`DNACClient` クラス経由で認証・デバイス検索・インターフェース一覧表示が一連の流れでエラーなく完了することを確認する。


## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験および記述試験を想定した実践問題と詳細な解答・解説です。

---

### 問題 1 (Python スクリプト読解 & エラー修正)

#### 質問
以下の Python スクリプトは、DNAC 内の特定のスイッチ (`10.1.1.5`) の全物理インターフェース情報を取得しようとしていますが、実行時に `KeyError` が発生して失敗します。コード内の不具合を指摘し、修正したコードを示しなさい。

```python
import requests

url = "https://dnac.lab.local/dna/intent/api/v1/network-device/ip-address/10.1.1.5"
headers = {"x-auth-token": "MY_AUTH_TOKEN"}

response = requests.get(url, headers=headers, verify=False)
device_id = response.json()["id"]

intf_url = f"https://dnac.lab.local/dna/intent/api/v1/interface/network-device/{device_id}"
intf_response = requests.get(intf_url, headers=headers, verify=False)
print(intf_response.json())
```

#### 解答・解説
**原因**:
DNAC Network Device API のレスポンス JSON は、ルートオブジェクトの直下に `response` というキーを配し、その中に該当デバイスのオブジェクト（またはリスト）を格納します。
`response.json()["id"]` と直接参照すると、ルートディクショナリに `id` というキーが存在しないため `KeyError` が発生します。

**修正コード**:
```python
# 正しいアクセス方法
device_data = response.json().get("response", {})
device_id = device_data.get("id")
```

---

### 問題 2 (アクセス制御 API の設計)

#### 質問
セキュリティ自動化スクリプトにおいて、DNAC ネットワークデバイス API を使用して全アクセススイッチ（`role == "ACCESS"`）の `Unreachable` 状態の機器のみを抽出し、そのスイッチの管理 IP アドレスをリスト化するロジックを作成したい。Python のリスト内包表記を用いてこれを一律で抽出する記述を示しなさい。

#### 解答・解説
**正解**:
```python
devices = response.json().get("response", [])

unreachable_access_switches = [
    dev.get("managementIpAddress")
    for dev in devices
    if dev.get("role") == "ACCESS" and dev.get("reachabilityStatus") == "Unreachable"
]

print("Unreachable Access Switches:", unreachable_access_switches)
```

---

### 問題 3 (トラブルシューティング & 認証エラー)

#### 質問
DNAC Network Device API を呼び出した際、ステータスコード `401 Unauthorized` が返されました。原因として考えられる可能性を 2 つ挙げ、それらをスクリプト内で自動的にハンドリングする設計について説明しなさい。

#### 解答・解説
**原因**:
1. リクエストヘッダー `x-auth-token` にセットしたトークンが失効（取得から 1 時間以上経過）している。
2. ヘッダー名やトークン文字列のタイポ。

**自動ハンドリング設計**:
HTTPレスポンスのステータスコードが 401 であるかを判定し、401 を検知した場合に `get_auth_token()` を自動的に再実行して新しいトークンを取得し、リクエストを 1 回のみ再試行するロジックを組み込みます。

---

### 問題 4 (大量データ取得におけるページネーション)

#### 質問
DNAC 環境に 1,200 台のネットワーク機器が登録されている場合、`GET /dna/intent/api/v1/network-device` をパラメータなしで単純呼出ししたときに発生する問題と、その解決策を述べなさい。

#### 解答・解説
**発生する問題**:
DNAC Network Device API の 1 リクエストあたりデフォルト取得上限（`limit`）は 500 件に制限されています。パラメータを指定しない場合、最初の 500 台分しか返されず、残りの 700 台のデータが欠落します。

**解決策**:
クエリ引数 `limit=500` および `offset=1` を指定し、レスポンス内のリスト件数が 0 になるまで `offset += 500` の加算を行いながら `while` ループで繰り返し API を呼び出します。

---

### 問題 5 (実務・実装問題: デバイスシリアル番号監査)

#### 質問
特定のシリアル番号（例: `FOC2134XXXX`）を持つスイッチを特定するため、DNAC Network Device API のシリアル番号指定エンドポイントを用いた Python 処理例を示しなさい。

#### 解答・解説
**エンドポイント**:
`GET /dna/intent/api/v1/network-device/serial-number/{serialNumber}`

**Python 実装例**:
```python
import requests

def search_by_serial(serial_num, token):
    url = f"https://dnac.lab.local/dna/intent/api/v1/network-device/serial-number/{serial_num}"
    headers = {"x-auth-token": token, "Content-Type": "application/json"}
    
    resp = requests.get(url, headers=headers, verify=False)
    if resp.status_code == 200:
        dev = resp.json().get("response", {})
        print(f"Found Device: {dev.get('hostname')} ({dev.get('managementIpAddress')})")
    else:
        print(f"Device with Serial {serial_num} not found.")
```


## 🔗 参考リソース

* Cisco Catalyst Center (DNA Center) Northbound API Reference Guide
* Cisco DNA Center API Documentation - Network Device API
* Cisco DevNet Learning Labs - Getting Started with Cisco DNA Center APIs
* Cisco Technical Support and Documentation - Cisco DNA Center Device Management
* Cisco Live Session: Automation with Cisco DNA Center Platform APIs


## 📝 補足（Notes）

* **x-auth-token ヘッダーの注意点**:
  * Cisco DNA Center の Intent API では、`Authorization: Bearer <token>` ではなく、カスタムヘッダー `x-auth-token: <token>` を使用します。
* **SSL/TLS 検証の扱い**:
  * ラボ環境では自己署名証明書が使用されるため、Python の `requests` ライブラリで `verify=False` を指定しないとエラーになります。
* **レスポンスキーの階層構造**:
  * レスポンス JSON は常に `{"response": ...}` の構造を持ちます。一覧取得時は `response` にリストが入り、単一検索（UUID/IP指定）の際は `response` にディクショナリが入ります。


