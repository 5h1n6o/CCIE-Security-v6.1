---
layout: default
title: 3.10.b-Network-discovery
nav_order: 2
parent: 3.10-DNAC-APIs
grand_parent: 3.0-Security-Infrastructure
---

# 3.10.b Network discovery

## 📘 概要

Cisco DNA Center（Catalyst Center）Northbound Intent APIにおける **Network Discovery API** は、ネットワーク内のスイッチ、ルータ、アクセスポイント、WLC（Wireless LAN Controller）などのアクティブなネットワークインフラ機器を自動的に検出・識別し、DNACのインベントリデータベースに一括登録するためのプログラムインターフェース群です。

ネットワークの可視化と自動化の第一歩は、管理対象となる全デバイスの正確な発見とインベントリ化です。従来の手動によるIPアドレス指定や個別ログイン設定、あるいはGUIベースの操作に代わり、REST APIを活用することで大規模なグリーンフィールド構築や既存（ブラウンフィールド）環境のインベントリ移行・維持管理を完全に自動化できます。

Network Discovery APIの主な利用目的および利用場面は以下の通りです。

* **自動インベントリオンボーディング**: 新規拠点展開（Greenfield）や既存ネットワーク統合時、数百〜数千台のネットワーク機器をIP範囲（IP Range）やLLDP/CDPトポロジー追跡によって自動検出。
* **認証情報の一括適用と検証**: 暗号化された管理資格情報（CLI, SNMPv2c/v3, HTTP/HTTPS）を用いて各デバイスへのアクセシビリティおよび管理権限を検証。
* **定期的な構成変更・未管理機器の検出**: スケジュール化されたDiscoveryジョブを実行し、ネットワーク内に追加された未承認機器やIPアドレス変更を追跡。
* **他システム（ISE/IPAM/ITSM）との連携統合**: 発見されたデバイス情報を外部の認証基盤（Cisco ISE）や資産管理データベース（ServiceNow等）へ自動同期するためのデータソースとして利用。


---

## 🔑 要点

Network Discovery APIに関する技術的特徴、主な用途、メリット・デメリット、制限事項および設計上の注意点を下表にまとめます。

| 項目 | 内容 |
| --- | --- |
| **特徴** | RESTful APIベースでのDiscoveryジョブ作成、非同期処理（Async Job API）、CDP/LLDP/IP Range/Multi-Rangeディスカバリ対応 |
| **用途** | ネットワークインフラの全自動検出、暗号化資格情報（Credential）のバインドと検証、インベントリへの自動追加 |
| **メリット** | 手作業不要で大規模ネットワークの発見を高速化、スクリプトによる定期実行・他システム（ISE/ServiceNow）連携が容易 |
| **デメリット** | 非同期ジョブ管理（Polling）が必要、ネットワーク機器側のSNMP/SSH/CDP設定不備による検出失敗リスク |
| **対応機種** | Cisco Catalyst 9000シリーズ、ISR/ASRルータ、Catalyst 9800/AireOS WLC、Nexus (LAN Automation対象) 等 |
| **制限事項** | 1ジョブあたりのIPアドレス範囲制限、最大同時実行Discoveryジョブ数制限、APIレートリミット（429 Too Many Requests） |
| **設計上の注意点** | サブネット全域のDiscovery実行時におけるネットワークトラフィック負荷、暗号化資格情報ID（Credential ID）の事前定義が必須 |


---

## 🏗 動作原理

Cisco DNA Centerにおけるネットワークディスカバリは、DNA Centerコントローラが指定されたIPアドレス範囲やシードデバイスから各種ネットワーク管理プロトコルを用いて能動的に通信を試みるプロブ（Probing）メカニズムに基づいています。

```
+------------------+
|  Python Script   |
+--------+---------+
         | (1) POST /dna/intent/api/v1/discovery (JSON Payload)
         v
+------------------+
| Cisco DNA Center |
+--------+---------+
         | (2) Job ID 発行 (202 Accepted)
         |
         |---> (3) Protocol Probing (Ping / ICMP)
         |
         |---> (4) SNMP Community / SNMPv3 Auth Check
         |
         |---> (5) CLI Login Check (SSH / Telnet)
         |
         |---> (6) CDP / LLDP Neighbor Crawling (Scan Mode依存)
         |
         v
+------------------+     SNMP / SSH / ICMP     +---------------------+
| Network Devices  | <=======================> | Target Subnet / BSS |
| (Switches/WLCs)  |                           +---------------------+
+--------+---------+
         |
         | (7) Success: Ingest into DNAC Inventory Database
         v
+------------------+
| DNAC Inventory   |
+------------------+
```

DNA Centerのディスカバリエンジンは、以下のコア技術要素を組み合わせて動作します。

1. **Discovery Profile & Protocol Probing**:
   * **Ping (ICMP)**: 対象IPアドレスの稼働状態を確認。
   * **SNMP (v2c/v3)**: デバイスのsysObjectID、ホスト名、シリアル番号、IOS/IOS-XEバージョン、モデル名を高速取得。
   * **CLI (SSH/Telnet)**: 詳細な構成情報、インターフェイス状態、スタック構成、ライセンス状態の収集および高度な自動管理権限の確認。
   * **HTTP/HTTPS (RESTCONF/NETCONF)**: Catalyst 9000等のモダンデバイスにおける高度なTelemetry/API連携確認。
2. **Scan Algorithm Modes**:
   * **IP Range Discovery**: 開始IPから終了IPまでの範囲をスキャン。サブネットマスク指定や複数サブネットの一括指定が可能。
   * **CDP / LLDP Multi-hop Discovery**: シードデバイス（起始ノード）のIPを指定し、CDP/LLDP隣接テーブルを再帰的にトラバースしてネットワーク全体を追跡（Hop Count指定）。
3. **Credential Binding Mechanism**:
   * APIリクエスト送信前に、DNAC上に事前登録された「Global Credentials (CLI Credentials, SNMPv2 Read/Write, SNMPv3 Credentials)」のUUID（Credential ID）をリクエストボディに含める必要があります。DNACはこれらのIDに対応する暗号化パスワード/コミュニティ文字列を用いて機器へアクセスします。


---

## ⚙ 動作シーケンス

API経由でNetwork Discoveryが実行され、デバイスがインベントリに追加されるまでのパケットおよびAPI処理シーケンスは以下の通りです。

```
[Client / Python]             [Cisco DNA Center]               [Network Device]
      |                                |                              |
      |--- (1) POST /discovery ------->|                              |
      |    (IP Range + Credential IDs) |                              |
      |                                |                              |
      |<-- (2) 202 Accepted -----------|                              |
      |    (Execution ID / Job ID)     |                              |
      |                                |                              |
      |                                |-- (3) ICMP Echo Request ---->|
      |                                |<-- ICMP Echo Reply ----------|
      |                                |                              |
      |                                |-- (4) SNMP Get-Request ----->|
      |                                |<-- SNMP Response (sysOID) ---|
      |                                |                              |
      |                                |-- (5) SSH SYN (Port 22) ---->|
      |                                |<-- SSH SYN-ACK --------------|
      |                                |-- SSH Handshake & Auth ----->|
      |                                |<-- Auth Success -------------|
      |                                |                              |
      |                                |-- (6) Exec 'show version' -->|
      |                                |-- Exec 'show cdp neighbors'->|
      |                                |                              |
      |--- (7) GET /discovery/{id} --->|                              |
      |<-- (8) Job Status: IN_PROGRESS-|                              |
      |                                |                              |
      |                                | [Discovery Engine Finished]  |
      |                                | [Save to DB as REACHABLE]    |
      |                                |                              |
      |--- (9) GET /discovery/{id} --->|                              |
      |<-- (10) Job Status: SUCCESS ---|                              |
      |                                |                              |
      |--- (11) GET /network-device -->|                              |
      |<-- (12) Discovered Device Data-|                              |
```

1. **API Initiating Phase**: クライアントは `POST /dna/intent/api/v1/discovery` に対し、`discoveryType` (例: Single Range, Multi Range, CDP)、`ipAddressList`、`cdpLevel`、および使用する `globalCredentialIdList` を指定してリクエストを送信します。
2. **Job Allocation Phase**: DNACは非同期レスポンス `202 Accepted` とともに `executionId` (ジョブ識別用UUID) を返却します。
3. **Probing & Access Phase**: DNAC内部のDiscoveryタスクが起動し、指定IP範囲に対してICMP Pingを実施。応答があったIPに対してSNMP GetおよびSSH接続を試行します。
4. **Information Mining Phase**: ログインに成功した場合、`show version`、`show inventory`、`show cdp neighbors detail` などを内部的に発行し、シリアル番号、MACアドレス、製品ID (PID)、ソフトウェアバージョン等の属性情報を抽出します。
5. **Database Ingestion & Validation Phase**: 収集したデータを元にDNACのTopology DBおよびInventory DBに登録。アクセシビリティ状態を `REACHABLE` または `UNREACHABLE`、管理状態を `MANAGED` に更新します。
6. **Job Completion & Retrieval Phase**: スクリプトは `GET /dna/intent/api/v1/discovery/{id}/job` を定期的にポーリングし、ステータスが `SUCCESS` または `COMPLETED` に変化したことを確認後、`GET /dna/intent/api/v1/discovery/{id}/network-device` を呼び出して発見されたデバイス一覧を取得します。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験における Cisco DNA Center (Catalyst Center) 関連のタスクでは、GUIによる直感的な操作だけでなく、**Pythonスクリプトを用いてDNAC APIとインタラクションを行い、自動化要件を満たすこと**が強く求められます。特に Network Discovery はインベントリ構築の基盤であるため、確実にAPIを実行・検証できるスキルが必須です。

### 1. ラボ試験で頻出の問われ方・要件
* **特定のサブネット範囲の自動ディスカバリ実行**:
  * 例: 「10.1.100.1–10.1.100.254 の範囲にある全機器を、事前に登録されているCLI資格情報ID（Credential ID）とSNMPv3資格情報IDを用いて自動検出しなさい。ジョブが完了するまでPythonスクリプト内で待機し、完了後に発見されたデバイスのIPとホスト名をコンソール出力しなさい。」
* **既存シードデバイスからのCDPネイバー再帰ディスカバリ**:
  * 例: 「コアスイッチ（192.168.1.1）をシードノードとして、CDPホップ数 2 までの全ネットワーク機器を検出するDiscoveryジョブを作成しなさい。」
* **ディスカバリ失敗機器のアラートと抽出**:
  * 例: 「Discovery実行結果のうち、SNMP認証エラーまたはPing不達（Unreachable）となったデバイスのIPアドレス一覧を抽出するPythonスクリプトを作成しなさい。」

### 2. よくある設定ミス・ハマりポイント
* **Content-Type および Header の指定漏れ**:
  * API呼び出し時、`Content-Type: application/json` と `x-auth-token: <Token>` を正しくヘッダーに設定しないと `400 Bad Request` または `401 Unauthorized` となります。
* **Credential ID (UUID) の不一致・取得漏れ**:
  * Discovery APIのペイロード `globalCredentialIdList` には、ユーザー名や暗号コミュニティ名そのものではなく、DNAC上で発行された **UUID文字列** (`GET /dna/intent/api/v1/global-credential` で取得) を配列で渡す必要があります。文字列名をそのまま渡すと即座にジョブがエラー終了します。
* **同期型 API と非同期型 (Async) API の誤解**:
  * Discovery POST API は即座にデバイス一覧を返すのではなく、**`202 Accepted` と `executionId`（Job ID）のみを返します**。ジョブ完了を待たずにデバイス取得API（`GET /discovery/{id}/network-device`）を叩くと、空のリストが返るかエラーとなります。必ず Job Status をポーリングして `COMPLETED` または `SUCCESS` になるまで待機するループ処理が必要です。
* **IP Range フォーマットの指定誤り**:
  * DNAC Discovery APIの `ipAddressList` は、文字列形式 `"10.1.10.1-10.1.10.254"` または CIDR 表記を正しく配列形式で渡す必要があります。


---

## 🛠 設定方法

Network Discovery API を実行するための標準的な Python スクリプト実装モデル手順を以下に示します。

```python
import requests
import json
import time
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# 1. DNAC 接続情報 & 認証
DNAC_URL = "https://dnac.example.com"
USERNAME = "cisco_user"
PASSWORD = "cisco_password"

session = requests.Session()
session.verify = False
session.headers.update({"Content-Type": "application/json"})

# トークン取得
auth_resp = session.post(f"{DNAC_URL}/dna/system/api/v1/auth/token", auth=(USERNAME, PASSWORD))
auth_resp.raise_for_status()
token = auth_resp.json()["Token"]
session.headers.update({"x-auth-token": token})

# 2. 事前登録済みの Credential ID (UUID) を取得
cred_resp = session.get(f"{DNAC_URL}/dna/intent/api/v1/global-credential")
cred_resp.raise_for_status()
credentials = cred_resp.json().get("response", [])

cli_cred_id = None
snmp_cred_id = None

for cred in credentials:
    # CLI 資格情報の抽出演習
    if cred.get("credentialType") == "GLOBAL_CLI":
        cli_cred_id = cred.get("id")
    elif cred.get("credentialType") == "GLOBAL_SNMPV2_READ":
        snmp_cred_id = cred.get("id")

print(f"[*] Found CLI Credential ID: {cli_cred_id}")
print(f"[*] Found SNMP Credential ID: {snmp_cred_id}")

# 3. Discovery ジョブのペイロード作成
discovery_payload = {
    "name": "Lab_Exam_Discovery_Subnet100",
    "discoveryType": "Range",
    "ipAddressList": "10.1.100.1-10.1.100.50",
    "protocolOrder": "ssh",
    "timeOut": 5,
    "retryCount": 2,
    "globalCredentialIdList": [cli_cred_id, snmp_cred_id]
}

# 4. Discovery ジョブの起動 (POST)
disc_resp = session.post(f"{DNAC_URL}/dna/intent/api/v1/discovery", json=discovery_payload)
disc_resp.raise_for_status()
job_info = disc_resp.json().get("response", {})
job_id = job_info.get("taskId") or job_info.get("executionId")
print(f"[+] Discovery Job Triggered. Job ID: {job_id}")

# 5. ジョブ完了の非同期ポーリング (Polling Loop)
while True:
    status_resp = session.get(f"{DNAC_URL}/dna/intent/api/v1/discovery/{job_id}/job")
    if status_resp.status_code == 200:
        res_data = status_resp.json().get("response", {})
        progress = res_data.get("progress")
        status = res_data.get("status")
        print(f"[*] Discovery Status: {status} ({progress})")
        if status in ["SUCCESS", "COMPLETED", "FAILED"]:
            break
    time.sleep(10)

# 6. 発見されたデバイス一覧の取得
devices_resp = session.get(f"{DNAC_URL}/dna/intent/api/v1/discovery/{job_id}/network-device")
devices = devices_resp.json().get("response", [])

print(f"\\n[=== Discovered Devices Summary ({len(devices)} items) ===]")
for dev in devices:
    print(f"Hostname: {dev.get('hostname')}, IP: {dev.get('managementIpAddress')}, Status: {dev.get('reachabilityStatus')}, Family: {dev.get('family')}")
```


---

## 🔍 検証コマンド

REST APIによるDiscovery実行状態およびターゲット機器側の設定・動作状態を確認するための検証手順を下表に示します。

### REST API 検証エンドポイント

| 目的 | HTTP Verb | エンドポイント | 説明 |
| --- | --- | --- | --- |
| グローバル資格情報一覧取得 | `GET` | `/dna/intent/api/v1/global-credential` | DNACに登録されているCLI/SNMP/HTTP等のUUID一覧を取得 |
| Discovery ジョブ作成 | `POST` | `/dna/intent/api/v1/discovery` | ディスカバリジョブを起動しJob IDを取得 |
| Discovery ジョブ状態確認 | `GET` | `/dna/intent/api/v1/discovery/{id}/job` | ジョブの進捗率・ステータス（IN_PROGRESS, SUCCESS等）を確認 |
| 発見デバイス一覧取得 | `GET` | `/dna/intent/api/v1/discovery/{id}/network-device` | 指定したDiscoveryジョブで検出された機器詳細情報を取得 |
| 全Discovery ジョブ一覧 | `GET` | `/dna/intent/api/v1/discovery/1/100` | 過去に実行された全ディスカバリジョブの履歴を取得 |
| インベントリ全体デバイス一覧 | `GET` | `/dna/intent/api/v1/network-device` | DNACインベントリ全体の登録状況およびReachabilityを確認 |

### ターゲット機器（IOS-XE / Catalyst 9000）側 CLI 確認コマンド

| 目的 | コマンド |
| --- | --- |
| SSHセッション・ログイン確認 | `show users` または <code>show ssh</code> |
| CDP隣接関係（Seed Discovery用） | <code>show cdp neighbors detail</code> |
| SNMPアクセスカウンタ確認 | <code>show snmp</code> |
| VTYラインアクセス制限確認 | <code>show line vty 0 4</code> |


## 🚨 トラブルシューティング

DNAC Network Discovery API 実行時に発生しやすいトラブル症状、根本原因、確認方法および解決策を下表に整理します。

| 症状 | 原因 | 確認コマンド / API | 対処方法 |
| --- | --- | --- | --- |
| Discovery ジョブが即座に `FAILED` になる | リクエストペイロードの `globalCredentialIdList` に無効なUUIDが含まれる、または必須パラメータ欠落 | `GET /dna/intent/api/v1/discovery/{id}/job` | `GET /global-credential` で正しいCredential UUIDを再取得しペイロードを修正 |
| ジョブが `IN_PROGRESS` のまま長時間進まない | 大規模IP範囲指定時のPing/SNMPタイムアウト多発、またはDNACディスカバリタスクのキュー詰まり | `GET /dna/intent/api/v1/discovery/{id}/job` | 対象IP範囲をスプリット（分割実行）し、`timeOut` および `retryCount` パラメータを最適化 |
| デバイスは検出されるが `Bad Credentials` と表示される | ターゲット機器側のSSH/SNMPパスワード・コミュニティ変更、VTY Access-class制限 | ターゲット機器で <code>show users</code>, <code>show log</code> | ターゲット機器のVTY ACLにDNAC IPを追加、またはDNAC側のCredential更新 |
| デバイスが `UNREACHABLE` ステータスで登録される | DNACからターゲット機器へのPing/SNMP不達、途中のファイアウォールによるUDP 161 / TCP 22 遮断 | DNAC CLIから <code>ping <target_ip></code>, 機器側 <code>show access-lists</code> | 管理ネットワークのルーティングおよびFW許可ルール（ICMP, SSH/Port 22, SNMP/Port 161, NETCONF/Port 830）を修復 |
| API呼び出し時に `429 Too Many Requests` | DNAC APIのレート制限超過（短時間での過剰なAPIリクエスト呼び出し） | HTTP レスポンスヘッダー (`Retry-After`) | スクリプトのポーリング間隔を広げる（例: 5秒→15秒）、指数バックオフ（Exponential Backoff）を実装 |
| Discoveryでスイッチの一部スタックメンバーが検出されない | スイッチスタック構成時のSNMP MIBインデックス未同期、または一時的なスタックリンク障害 | スイッチ側 <code>show switch</code>, <code>show inventory</code> | スタック状態が `Ready` であることを確認し、DNAC側で手動再ディスカバリ（Resync）を実行 |


---

## ⚠ 制限事項

Cisco DNA Center Network Discovery API における機能制限、バージョン・ライセンス依存関係は以下の通りです。

* **1ジョブあたりの最大IPアドレス数**:
  * 1回のDiscoveryジョブでスキャンできる最大IPアドレス数は標準で 2,560 IP（/22 サブネット相当）に制限されます。それ以上の規模をスキャンする場合は複数ジョブに分割する必要があります。
* **同時実行ディスカバリジョブ数**:
  * DNA Center コントローラクラスターの負荷保護のため、同時に実行可能なDiscoveryジョブ数は最大 5 ジョブ程度に制限されます。
* **ライセンス依存 (Cisco DNA Software License)**:
  * Discovery機能自体は基本機能に含まれますが、検出された機器を完全に管理し、Assurance や Automation 機能（SGT割り当て、SWIM等）を適用するには `Cisco DNA Essentials` または `Cisco DNA Advantage` ライセンスが必要です。
* **サポートプロトコル要件**:
  * 機器発見には SNMP (v2c/v3) または CLI (SSH/Telnet) のいずれか少なくとも1つの有効なアクセスが必須です。ICMP Ping のみでは機器種別が識別できずインベントリに登録されません。


---

## 🔄 他技術との関連

DNAC Network Discovery API は、他ネットワーク技術・セキュリティソリューションと密接に連携します。

* **Cisco ISE (Identity Services Engine)**:
  * DNACでディスカバリされたネットワーク機器（スイッチ・WLC）は、pxGrid または REST API を介して Cisco ISE の Network Devices (NAD) リストへ自動同期可能。802.1X/RADIUS Shared Secret も統合管理。
* **Cisco TrustSec (SGT / SXP)**:
  * ディスカバリ完了後、各デバイスが TrustSec 対応機器として識別されると、SXP (Security Group Exchange Protocol) ピアリング設定や SGT (Security Group Tag) インラインタギングの自動化が有効化。
* **SNMP / Syslog Monitoring Protocols**:
  * Discovery実行時、DNACはデバイスに対して自身のIPを SNMP Trap 送信先および Syslog サーバーとして自動的に追加コンフィグレーション（Telemetry構成）を投入。
* **IPAM (IP Address Management) / ServiceNow Integration**:
  * Discovery API経由で取得したMACアドレス、IP、シリアル番号、IPアドレス使用状況を外部IPAMやServiceNow CMDBへ即座に転送し資産管理を同期。


---

## 🧩 比較表

### Discovery Modes 比較

| スキャン方式 | 対象定義手法 | 主な適用場面 | メリット | デメリット |
| --- | --- | --- | --- | --- |
| **IP Range Scan** | IP開始・終了アドレス指定 | 単一サブネットや特定IP領域 | 検出範囲が明確で予測しやすい | 未知のサブネット機器を検出できない |
| **Multi-Range Scan** | CIDRまたは複数範囲指定 | 拠点全体の全サブネット一括 | 1回のジョブで複数VLANを一括処理 | 対象数が大きく時間がかかる |
| **CDP Discovery** | Seed IP + Hop Count | トポロジー構造が既知の環境 | 物理接続構造に沿って自動追跡 | CDP無効機器や非Cisco機器で停止 |
| **LLDP Discovery** | Seed IP + Hop Count | マルチベンダー混在環境 | 標準プロトコルでサードパーティ対応 | 機器によってLLDP MIB実装にバラつき |

### インベントリ登録手法の比較

| 登録手法 | 実行手段 | 柔軟性 | 適合スケール | 主な用途 |
| --- | --- | --- | --- | --- |
| **Network Discovery API** | REST API (Python) | **極めて高い** (自動化・スクリプト連携) | 大規模 (自動分割・並列処理) | CI/CD, 自動化運用, 定期スキャン |
| **DNAC GUI Discovery** | Web UI 操作 | 中程度 (GUI手動入力) | 小〜中規模 | 単発の拠点追加, 試走確認 |
| **CSV Bulk Import** | ファイルアップロード | 低い (事前ファイル作成が必要) | 中規模 | 既知のIP/MACリストの一括登録 |


---

## 💡 ベストプラクティス

1. **IP Range の最適化**:
   * 未使用IPが大量に含まれる広大なサブネット（例: /16 全域）の一括指定は避け、VLAN/サブネット単位（/24 程度）に分割してディスカバリを実行してください。これによりタイムアウト待ちによるジョブ遅延を防止できます。
2. **優先アクセスプロトコルの設定 (Protocol Order)**:
   * 可能な限り `ssh` および `snmpv3` を優先プロトコルとして指定してください。レガシーな Telnet や SNMPv2c の使用はセキュリティポリシー上無効化し、APIペイロード内でも排除することを推奨します。
3. **エラーハンドリングとポーリング間隔の最適化**:
   * Pythonスクリプトで非同期ジョブを監視する際は、最低 10〜15 秒のポーリング間隔を設定し、DNACへの負荷を抑えてください。また `status == "FAILED"` 発生時にはログに `progress` メッセージを出力して原因追跡を容易にしてください。


## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験の実践的自動化タスクを想定した10個のPythonスクリプト演習例です。各コード例は省略せず、エラー処理およびレスポンスパースを含めて記載しています。

---

### シナリオ 1: IP Range Discovery Job の新規作成

**【問題】**
DNAC API を使用して、IP アドレス範囲 `10.1.50.1-10.1.50.30` に対するディスカバリジョブを作成し、発行された Job ID を取得しなさい。

**【要件】**
* ディスカバリ名: `Lab_Discovery_Subnet50`
* 優先プロトコル: `ssh`
* 既存の CLI Global Credential UUID を検索してアタッチすること。

```python
import requests
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_URL = "https://dnac.example.com"
USERNAME = "cisco_user"
PASSWORD = "cisco_password"

session = requests.Session()
session.verify = False
session.headers.update({"Content-Type": "application/json"})

# 1. Auth Token 取得
tok_resp = session.post(f"{DNAC_URL}/dna/system/api/v1/auth/token", auth=(USERNAME, PASSWORD))
token = tok_resp.json()["Token"]
session.headers.update({"x-auth-token": token})

# 2. CLI Credential ID 検索
cred_resp = session.get(f"{DNAC_URL}/dna/intent/api/v1/global-credential")
credentials = cred_resp.json().get("response", [])
cli_id = next((c["id"] for c in credentials if c.get("credentialType") == "GLOBAL_CLI"), None)

if not cli_id:
    raise ValueError("GLOBAL_CLI Credential not found!")

# 3. Discovery Payload
payload = {
    "name": "Lab_Discovery_Subnet50",
    "discoveryType": "Range",
    "ipAddressList": "10.1.50.1-10.1.50.30",
    "protocolOrder": "ssh",
    "globalCredentialIdList": [cli_id]
}

# 4. Trigger Discovery
res = session.post(f"{DNAC_URL}/dna/intent/api/v1/discovery", json=payload)
res.raise_for_status()
job_id = res.json()["response"]["taskId"]
print(f"SUCCESS: Triggered Discovery Job. Task/Job ID: {job_id}")
```

---

### シナリオ 2: Discovery Status 追跡とジョブ完了待機

**【問題】**
指定した Discovery Job ID の進行状態を 10 秒間隔でポーリングし、ジョブが `SUCCESS` または `COMPLETED` になるまで待機するループ関数を作成しなさい。

**【要件】**
* 進行状況 (`progress`) およびステータス (`status`) をコンソールにリアルタイム表示すること。
* `FAILED` になった場合は例外を発生させること。

```python
import time

def wait_for_discovery_completion(session, dnac_url, job_id):
    print(f"[*] Monitoring Job ID: {job_id}")
    while True:
        url = f"{dnac_url}/dna/intent/api/v1/discovery/{job_id}/job"
        resp = session.get(url)
        if resp.status_code == 200:
            data = resp.json().get("response", {})
            status = data.get("status")
            progress = data.get("progress")
            print(f" -> Status: {status} | Progress: {progress}")
            
            if status in ["SUCCESS", "COMPLETED"]:
                print("[+] Discovery Job Completed Successfully!")
                return True
            elif status == "FAILED":
                raise RuntimeError(f"[-] Discovery Job Failed. Detail: {progress}")
        else:
            print(f"[!] Warning: HTTP {resp.status_code} received when fetching job status.")
            
        time.sleep(10)
```

---

### シナリオ 3: 発見されたデバイス一覧の取得と Reachability フィルタリング

**【問題】**
完了した Discovery Job によって検出されたデバイス一覧を取得し、`REACHABLE` な機器と `UNREACHABLE` な機器を分類して出力しなさい。

```python
def analyze_discovered_devices(session, dnac_url, job_id):
    url = f"{dnac_url}/dna/intent/api/v1/discovery/{job_id}/network-device"
    resp = session.get(url)
    resp.raise_for_status()
    
    devices = resp.json().get("response", [])
    reachable = []
    unreachable = []
    
    for dev in devices:
        info = {
            "hostname": dev.get("hostname", "Unknown"),
            "ip": dev.get("managementIpAddress"),
            "platform": dev.get("platformId"),
            "status": dev.get("reachabilityStatus")
        }
        if dev.get("reachabilityStatus") == "REACHABLE":
            reachable.append(info)
        else:
            unreachable.append(info)
            
    print(f"=== Reachable Devices ({len(reachable)}) ===")
    for r in reachable:
        print(f"  [+] Host: {r['hostname']} | IP: {r['ip']} | Platform: {r['platform']}")
        
    print(f"\\n=== Unreachable Devices ({len(unreachable)}) ===")
    for u in unreachable:
        print(f"  [-] Host: {u['hostname']} | IP: {u['ip']} | Status: {u['status']}")
```

---

### シナリオ 4: CDP ネイバー情報に基づく再帰的ディスカバリ実行

**【問題】**
コアスイッチ `192.168.1.1` をシードノードとして、ホップ数 `2` までの CDP ディスカバリジョブを発行するコードを作成しなさい。

```python
def trigger_cdp_discovery(session, dnac_url, seed_ip, cdp_level, cli_cred_id):
    payload = {
        "name": f"CDP_Discovery_Seed_{seed_ip}",
        "discoveryType": "CDP",
        "ipAddressList": seed_ip,
        "cdpLevel": cdp_level,
        "protocolOrder": "ssh",
        "globalCredentialIdList": [cli_cred_id]
    }
    
    resp = session.post(f"{dnac_url}/dna/intent/api/v1/discovery", json=payload)
    resp.raise_for_status()
    job_id = resp.json()["response"]["taskId"]
    print(f"[+] CDP Discovery Triggered for Seed {seed_ip} (Hop Count: {cdp_level}). Job ID: {job_id}")
    return job_id
```

---

### シナリオ 5: グローバル資格情報の動的抽出と Discovery へのバインド

**【問題】**
DNAC 上の全 Credentials を取得し、CLI (GLOBAL_CLI) と SNMPv3 (GLOBAL_SNMPV3) の UUID を自動抽出して Discovery ペイロードにバインドしなさい。

```python
def get_required_credential_ids(session, dnac_url):
    url = f"{dnac_url}/dna/intent/api/v1/global-credential"
    resp = session.get(url)
    resp.raise_for_status()
    
    creds = resp.json().get("response", [])
    selected_ids = []
    
    for c in creds:
        ctype = c.get("credentialType")
        cid = c.get("id")
        if ctype in ["GLOBAL_CLI", "GLOBAL_SNMPV3", "GLOBAL_SNMPV2_READ"]:
            print(f"[*] Binding Credential Type: {ctype} (ID: {cid})")
            selected_ids.append(cid)
            
    if not selected_ids:
        raise RuntimeError("No suitable global credentials found on DNAC.")
        
    return selected_ids
```

---

### シナリオ 6: 複数サブネット (Multi-Subnet) 並列ディスカバリジョブの起動

**【問題】**
複数のサブネット（`10.1.10.0/24`, `10.1.20.0/24`）に対してそれぞれ個別の Discovery ジョブを並列的に起動し、管理するスクリプトを作成しなさい。

```python
subnets = ["10.1.10.1-10.1.10.254", "10.1.20.1-10.1.20.254"]
active_jobs = {}

for idx, sub in enumerate(subnets):
    payload = {
        "name": f"Multi_Discovery_Subnet_{idx+1}",
        "discoveryType": "Range",
        "ipAddressList": sub,
        "protocolOrder": "ssh",
        "globalCredentialIdList": [cli_cred_id]
    }
    res = session.post(f"{DNAC_URL}/dna/intent/api/v1/discovery", json=payload)
    if res.status_code == 202 or res.status_code == 200:
        jid = res.json()["response"]["taskId"]
        active_jobs[sub] = jid
        print(f"[+] Started Job for {sub} -> Job ID: {jid}")

print(f"\\nAll jobs triggered. Active Job Tracking: {active_jobs}")
```

---

### シナリオ 7: Discovery 障害機器のアラート抽出スクリプト

**【問題】**
過去の Discovery 結果から、PING 応答があるものの SSH/SNMP ログイン失敗（Authentication Error）となったデバイスを特定してログを出力しなさい。

```python
def audit_discovery_auth_failures(session, dnac_url, job_id):
    url = f"{dnac_url}/dna/intent/api/v1/discovery/{job_id}/network-device"
    resp = session.get(url)
    devices = resp.json().get("response", [])
    
    auth_failures = []
    for d in devices:
        ping_status = d.get("pingStatus")
        auth_status = d.get("authStatus")
        if ping_status == "SUCCESS" and auth_status != "SUCCESS":
            auth_failures.append({
                "ip": d.get("managementIpAddress"),
                "error": d.get("failureReason", "Authentication Failed")
            })
            
    print(f"=== Authentication Failure Report ({len(auth_failures)} items) ===")
    for af in auth_failures:
        print(f"  [ALERT] Target IP: {af['ip']} | Failure Cause: {af['error']}")
```

---

### シナリオ 8: 発見されたデバイスの自動 Site 割り当て (Assign Device to Site)

**【問題】**
Discovery ジョブで検出された機器を特定した Site（例: `Global/USA/SAN-JOSE/Building-1`）へ自動的に割り当てる API を発行しなさい。

```python
def assign_device_to_site(session, dnac_url, site_id, device_ip):
    # まず IP から Device ID を取得
    dev_res = session.get(f"{dnac_url}/dna/intent/api/v1/network-device/ip-address/{device_ip}")
    dev_res.raise_for_status()
    device_id = dev_res.json()["response"]["id"]
    
    # Assign Payload
    payload = {
        "device": [{"ip": device_ip}]
    }
    assign_res = session.post(f"{dnac_url}/dna/intent/api/v1/site/{site_id}/device", json=payload)
    assign_res.raise_for_status()
    print(f"[+] Device {device_ip} (ID: {device_id}) successfully assigned to Site ID: {site_id}")
```

---

### シナリオ 9: 発見されたスイッチの全 Interface データ取得

**【問題】**
Discovery によってインベントリに追加されたスイッチの Device ID を使用して、実装されている全物理インターフェイスおよびリンク状態を取得しなさい。

```python
def get_device_interfaces(session, dnac_url, device_ip):
    # Device ID 取得
    dev_res = session.get(f"{dnac_url}/dna/intent/api/v1/network-device/ip-address/{device_ip}")
    device_id = dev_res.json()["response"]["id"]
    
    # Interface 一覧取得
    int_res = session.get(f"{dnac_url}/dna/intent/api/v1/interface/network-device/{device_id}")
    interfaces = int_res.json().get("response", [])
    
    print(f"=== Interface List for {device_ip} ({len(interfaces)} ports) ===")
    for i in interfaces:
        port_name = i.get("portName")
        status = i.get("status")
        speed = i.get("speed")
        print(f"  Port: {port_name:<15} | Status: {status:<10} | Speed: {speed}")
```

---

### シナリオ 10: DNAC Discovery 情報と Cisco ISE Network Device 同期スクリプト

**【問題】**
DNAC Discovery で検出された Reachable なスイッチの IP アドレスとホスト名を置き換え、Cisco ISE REST API (ERS API) を用いて Network Access Device (NAD) として自動登録する連携スクリプトを作成しなさい。

```python
import requests

def sync_dnac_device_to_ise(ise_url, ise_user, ise_pass, device_ip, hostname, shared_secret):
    ise_headers = {
        "Content-Type": "application/json",
        "Accept": "application/json"
    }
    
    ise_nad_payload = {
        "NetworkDevice": {
            "name": hostname,
            "description": "Auto-synced from DNAC Discovery",
            "authenticationSettings": {
                "networkProtocol": "RADIUS",
                "radiusSharedSecret": shared_secret
            },
            "IPList": [{
                "ipaddress": device_ip,
                "mask": 32
            }]
        }
    }
    
    resp = requests.post(
        f"{ise_url}/ers/config/networkdevice",
        auth=(ise_user, ise_pass),
        headers=ise_headers,
        json=ise_nad_payload,
        verify=False
    )
    
    if resp.status_code == 201:
        print(f"[+] Successfully registered {hostname} ({device_ip}) to Cisco ISE NAD list.")
    else:
        print(f"[-] Failed to sync {hostname} to ISE. Status: {resp.status_code}, Body: {resp.text}")
```


# Part 5a: Exam Questions

## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験およびテクノロジー評価を想定した実践問題セットです。

---

### 問題 1 (API Implementation)
**【問題】**
Python スクリプトを使用して Cisco DNA Center の Discovery API を実行する際、HTTP ステータスコード `202 Accepted` が返却されました。スクリプトが即座に `GET /dna/intent/api/v1/network-device` を実行したところ、新しく検出されるはずのスイッチがリストに含まれていませんでした。この現象の根本的な原因と、スクリプトの正しい修正手順を説明しなさい。

**【解答・解説】**
* **原因**: `POST /dna/intent/api/v1/discovery` は非同期 (Asynchronous) API であり、呼び出し成功時にはタスクを受領したことを示す `202 Accepted` と `executionId` (Job ID) のみが返されます。バックグラウンドでのICMP/SNMP/SSHスキャンおよびインベントリDBへの追加処理はまだ完了していません。
* **修正手順**: 返却された `executionId` を使用し、`GET /dna/intent/api/v1/discovery/{id}/job` を定期的にポーリングするループ処理を実装します。ステータスが `SUCCESS` または `COMPLETED` に遷移したことを確認してから、発見デバイス取得 API またはインベントリ API を呼び出す必要があります。

---

### 問題 2 (Troubleshooting)
**【問題】**
DNA Center から特定サブネットに対して Discovery ジョブを発行したところ、ジョブステータスが直ちに `FAILED` となり、進行ログに `Invalid Global Credential ID` と表示されました。リクエストの JSON ペイロードには `globalCredentialIdList: ["cli_user", "snmp_comm"]` と記述されていました。コンフィグ / ペイロード上の誤りを指摘し、修正方法を述べなさい。

**【解答・解説】**
* **誤り**: `globalCredentialIdList` にユーザー識別名 (`"cli_user"`) やコミュニティ名 (`"snmp_comm"`) の文字列が直接指定されている点。
* **修正方法**: `globalCredentialIdList` には、DNAC 内で該当資格情報に割り当てられた **UUID 文字列** を渡す必要があります。事前に `GET /dna/intent/api/v1/global-credential` を実行して各 Credential オブジェクトの `"id"` フィールドを取得し、その UUID 配列をペイロードにセットします。

---

### 問題 3 (Design)
**【問題】**
ある企業ネットワークにおいて、拠点内の未管理サブネット 10.50.0.0/16 全域に対して DNAC ディスカバリを実行しようとしています。ネットワーク帯域制限および DNAC コントローラの処理負荷を考慮した場合、推奨されるディスカバリ設計および API 呼び出し戦略を述べなさい。

**【解答・解説】**
* **設計戦略**:
  1. **/16 などの広大なサブネットの一括スキャン回避**: 未使用 IP への Ping/SNMP タイムアウトが大量発生し、ジョブが長時間停滞するため、アクティブな VLAN / サブネット（例: /24 単位）へ分割します。
  2. **Multi-Range または分割 POST リクエスト**: サブネットリストを小さなブロックに分け、複数の Discovery ジョブとして順次または並列（最大 5 ジョブ以内）で実行します。
  3. **優先プロトコルの絞り込み**: 不要なプロトコルトライアルを避けるため `protocolOrder: "ssh"` および有効な SNMPv3 Credential のみに限定します。


### 問題 4 (Code Analysis)
**【問題】**
以下の Python コードにおけるセキュリティ上の不備および実行上の欠陥を 2 つ指摘しなさい。

```python
import requests

resp = requests.post(
    "https://dnac.example.com/dna/intent/api/v1/discovery",
    headers={"x-auth-token": "MY_API_TOKEN"},
    json={
        "name": "Test",
        "ipAddressList": "10.1.1.1-10.1.1.254"
    }
)
print(resp.json()["response"])
```

**【解答・解説】**
1. **欠陥 1 (Credential 未指定)**: Discovery ペイロードに `globalCredentialIdList` が含まれておらず、ログイン認証が試行できないため、機器の識別・詳細収集に失敗します。
2. **欠陥 2 (レスポンスキーの誤解)**: `POST /discovery` のレスポンスは Job ID 情報を含むオブジェクトであり、直接デバイスアレイは返りません。非同期ジョブ監視を行った後に別エンドポイントを叩く必要があります。

---

### 問題 5 (Implementation)
**【問題】**
シードデバイス `172.16.1.1` を起点とし、CDP ネイバーをたどって 2 ホップ以内のデバイスを発見する Discovery ジョブを Python スクリプトで定義する際の必須パラメータを挙げなさい。

**【解答・解説】**
* `discoveryType`: `"CDP"`
* `ipAddressList`: `"172.16.1.1"` (シードIP)
* `cdpLevel`: `2` (ホップ数)
* `globalCredentialIdList`: 配列で指定するアクセス資格情報ID
* `protocolOrder`: `"ssh"` (推奨アクセスプロトコル)

---

### 問題 6 (Integration)
**【問題】**
DNAC ディスカバリで検出されたスイッチを Cisco ISE に NAD として同期登録する自動化パイプラインにおいて、スイッチ側で事前に設定しておくべき必須セキュリティ設定を 3 つ挙げなさい。

**【解答・解説】**
1. **SSH / VTY Access-list (MPP)**: DNAC および ISE からの SSH/HTTPS アクセスを許可するインバウンド ACL 設定。
2. **SNMPv2c/v3 設定**: DNAC からのリード/ライトアクセスを許可するコミュニティまたはユーザー設定。
3. **AAA RADIUS 設定**: ISE との RADIUS 通信を許可する <code>radius server</code> 設定。


## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術資料です。

* Cisco Catalyst Center REST API Central (DevNet Developer Portal)
* Cisco Catalyst Center Intent API Reference Guide
* Cisco Catalyst Center User Guide - Manage Network Discovery
* Cisco Live Presentation: Automated Network Onboarding with Catalyst Center APIs (BRKDEV-2015)
* Cisco Validated Design: Enterprise Campus Network Design Guide


---

## 📝 **補足（Notes）**

### 💡 API 実行上のワンポイントチェック
* **トークン有効期限 (Token Lifetime)**:
  * DNAC の API アクセストークンの有効期限は **1 時間 (60分)** です。長時間に及ぶ大量サブネットのディスカバリスキャンやポーリングを行うスクリプトでは、トークン切れエラー (`401 Unauthorized`) を防ぐため、定期的なトークン再取得ロジックを組み込んでください。
* **ジョブ結果のデバッグ確認**:
  * ジョブステータスが `SUCCESS` であっても、個別の機器が Ping 不達や認証エラーでスキップされている場合があります。必ず Discovery デバイス取得 API のレスポンスに含まれる `reachabilityStatus` および `failureReason` を確認してください。



