---
layout: default
title: 3.10.b-Network-discovery
nav_order: 2
parent: 3.10-DNAC-APIs
grand_parent: 3.0-Security-Infrastructure
---

# 3.10.b Network discovery

Cisco DNA Center (DNAC) の **Network Discovery API** は、ネットワーク上のデバイスを自動的に検出し、インベントリへ登録するための RESTful インターフェイスです。管理者は Python スクリプトを使用して、特定の IP 範囲、プロトコル（SNMP, SSH 等）、および資格情報を指定してディスカバリ・ジョブを開始し、その進行状況や結果をプログラムで管理できます,。

---

## 📘 概要

*   **機能概要**: 指定したネットワークセグメントをスキャンし、到達可能なシスコ製およびサードパーティ製デバイスを特定して DNAC の管理下におく機能です。
*   **利用目的**: 新規拠点の立ち上げ時や、既存環境のデバイス情報を一括でインベントリに同期させる作業を自動化します。
*   **どのような場面で利用するか**:
    *   **大量導入 (ZTP/PnP)**: 新しいサブネット内のデバイスを一括検出する。
    *   **定期スキャン**: ネットワーク構成の変更を検知するために定期的にディスカバリを実行する。
    *   **インベントリ同期**: 外部の資産管理ツールと連携し、API 経由で検出を開始する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主なプロトコル** | SNMP (v2c/v3), CLI (SSH/Telnet), HTTP/HTTPS, CDP。 |
| **認証情報 (Credentials)** | CLI 認証、SNMP コミュニティ、SNMPv3 ユーザー情報が必要。 |
| **検出モード** | CDP ベース、IP 範囲（Range）ベース、LLDP ベース。 |
| **API 処理方式** | **非同期 (Asynchronous)**。POST リクエスト後に `taskId` を取得して完了を待つ。 |
| **主要エンドポイント** | `/dna/intent/api/v1/discovery`。 |
| **メリット** | 手動による誤入力を防ぎ、大規模環境のインベントリ登録時間を短縮する。 |

---

## 🏗 動作原理

DNAC は Northbound API からの「意図（Intent）」を受け取ると、Southbound プロトコルを使用して実際のデバイスと対話します。

```text
[ Python Script ]
      ↓ (1) POST /dna/intent/api/v1/discovery (JSON Payload)
[ Cisco DNA Center ]
      ↓ (2) Southbound Protocols (SNMP Get/Walk, SSH Login)
[ Network Devices ]
      ↓ (3) Device Facts (Hostname, Serial, Model, OS version)
[ Cisco DNA Center Inventory ]
      ↓ (4) Update Discovery Job Status
[ Python Script ]
      ↑ (5) GET /dna/intent/api/v1/task/{taskId}
```

---

## ⚙ 動作シーケンス

1.  **認証トークン取得**: `/dna/system/api/v1/auth/token` でセッションを開始する。
2.  **Discovery Job 作成**: 検出対象（IP 範囲）、スキャンプロトコル、優先順位などを JSON で定義し、`/discovery` に POST する。
3.  **Task ID の受理**: DNAC はジョブを受け付けると即座に `taskId` を返す。
4.  **ポーリング (Polling)**: スクリプトは `/task/{taskId}` を一定間隔で GET し、ステータスが `COMPLETED` になるのを待機する。
5.  **結果の取得**: ジョブ完了後、`/discovery/{id}` を使用して検出されたデバイスのリストや失敗原因を取得する。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **非同期処理の実装**: ラボ試験では、単に POST するだけでなく、**Task ID を使って完了を待つループ処理**を書かされる可能性が高いです。
*   **JSON ペイロードの構造**: IP 範囲 (`ipAddressList`) やプロトコル設定の正しいキー名を把握しておく必要があります。
*   **認証情報の指定**: スクリプト内で既存の `globalCredentialId` を指定するか、新規に定義するかの判断が求められます。
*   **エラーハンドリング**: 指定した IP 範囲にデバイスが不在だった場合や、SNMP 認証失敗時の API 応答をパースする能力が問われます。
*   **CDP 検出の活用**: シードデバイス（Seed IP）を指定して、CDP 隣接関係を辿ってネットワークを自動探索するシナリオに備えてください。

---

## 🛠 設定方法

### 1. ネットワーク検出の開始 (Python requests)
```python
import requests
import json
import time

# 認証トークン取得済み(token)の前提
headers = {
    "X-Auth-Token": token,
    "Content-Type": "application/json"
}

discovery_url = "https://{dnac_ip}/dna/intent/api/v1/discovery"

# 検出パラメータの定義
payload = {
    "name": "Lab_Discovery_Scan",
    "discoveryType": "Range",
    "ipAddressList": "10.1.1.1-10.1.1.254",
    "protocolOrder": "snmp,ssh",
    "preferredMgmtIPMethod": "None",
    "snmpV2ReadCommunity": "public"
}

# (2) POSTリクエスト送信
response = requests.post(discovery_url, headers=headers, json=payload, verify=False)
task_id = response.json()['response']['taskId']
print(f"Discovery Started. Task ID: {task_id}")
```

### 2. Task ステータスのポーリング
```python
task_url = f"https://{dnac_ip}/dna/intent/api/v1/task/{task_id}"

while True:
    task_res = requests.get(task_url, headers=headers, verify=False).json()
    status = task_res['response']['progress']
    print(f"Current Status: {status}")
    
    if "COMPLETED" in status or task_res['response']['endTime']:
        print("Discovery Task Finished.")
        break
    time.sleep(10)
```

---

## 🔍 検証コマンド

| 目的 | 手法 / エンドポイント |
| :--- | :--- |
| **実行中のジョブ確認** | <code>GET /dna/intent/api/v1/discovery</code> |
| **特定ジョブの詳細取得** | <code>GET /dna/intent/api/v1/discovery/{id}</code> |
| **検出されたデバイス数の確認** | <code>GET /dna/intent/api/v1/discovery/{id}/device-count</code> |
| **GUIでの確認** | **DNAC GUI > Tools > Discovery** |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| デバイスが検出されない | ネットワーク疎通(ICMP/SNMP)不可 | <code>Discovery API</code> の結果から ping 失敗を確認。 |
| 検出されるがインベントリに載らない | 資格情報の不一致 | <code>GET /discovery/{id}/summary</code> で Auth Error を確認。 |
| Task が途中で止まる | タイムアウト設定が短い | ペイロードの <code>timeout</code> 値を増やす。 |
| 403 Forbidden | API ユーザーの権限不足 | Role が "Network-Admin" 以上であることを確認。 |

---

## ⚠ 制限事項

*   **同時実行数**: 同時に実行できる Discovery ジョブの数には上限があります。
*   **IP アドレスの重複**: 複数のジョブで同じ IP 範囲を重複して指定することは避けるべきです。
*   **サポート対象**: シスコ製以外のデバイスは、SNMP による基本的な検出のみに制限される場合があります。

---

## 🔄 他技術との関連

*   **3.10.c Network Device**: 検出後にデバイスの詳細情報を取得する API。
*   **3.11 SD-Access**: Fabric 構成要素としてデバイスをアンダーレイ検出する際に使用。
*   **SNMP/SSH Credentials**: Discovery API を使用する前に、これらが DNAC に登録されている必要があります。

---

## 🧩 比較表

### Discovery モード比較

| モード | メカニズム | 適したシナリオ |
| :--- | :--- | :--- |
| **Range Scan** | 指定した IP 範囲を全スキャン | セグメント全体の把握、新規導入。 |
| **CDP Discovery** | シードから CDP 隣接を辿る | 物理トポロジが明確な環境での迅速な検出。 |
| **LLDP Discovery** | LLDP 隣接を辿る | サードパーティ製スイッチ混在環境。 |

---

## 💡 ベストプラクティス

1.  **プロトコル順序の最適化**: ネットワークで標準的なプロトコル（例：SNMPv3）を `protocolOrder` の先頭に配置します。
2.  **タグ付け**: 検出時に `preferredMgmtIPMethod` 等を利用し、管理 IP を固定化します。
3.  **ジョブのクリーンアップ**: 完了した古いディスカバリ・ジョブは API 経由で定期的に削除し、管理画面を整理します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 特定サブネットの検出開始
*   **問題**: `192.168.10.0/24` の範囲を SNMPv2c でスキャンせよ。
*   **設定**: `ipAddressList: "192.168.10.0-192.168.10.255"`, `snmpV2ReadCommunity: "cisco"`.

### 2. Task ID を用いた完了待ちロジック
*   **要件**: Python で `while` 文と `time.sleep` を使い、ステータスを監視せよ。

### 3. 検出されたデバイスのサマリ取得
*   **操作**: ジョブ完了後に `/discovery/{id}/summary` を GET し、検出成功数を出力せよ。

### 4. CDP を使用したシード検出
*   **要件**: シード IP `10.1.1.1` を指定し、ホップ数 3 で CDP 検出を実行せよ。

### 5. 資格情報の ID 指定 (Source 参照)
*   **要件**: DNAC に登録済みの SSH 資格情報 ID を使用してディスカバリを実行せよ。

### 6. ジョブの進行状況 (Percentage) の表示
*   **課題**: Task API の `progress` フィールドからパーセンテージを抽出し表示せよ。

### 7. スキャン失敗原因の特定
*   **操作**: `discovery/{id}/nodes` エンドポイントを確認し、`errorStatus` をパースせよ。

### 8. Discovery ジョブの削除
*   **要件**: `DELETE /discovery/{id}` を送信し、特定のジョブ履歴を消去せよ。

### 9. タイムアウト値の動的設定
*   **要件**: ネットワーク遅延を考慮し、デフォルトより長いスキャンタイムアウトを指定せよ。

### 10. 特定デバイス名によるフィルタリング
*   **課題**: 検出された全デバイスの中から、名前に "Core" を含むものだけを表示せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: Discovery POST リクエストの JSON に `"discoveryType": "Range"` とある。この場合、探索範囲はどのように指定すべきか？
    *   **回答**: `ipAddressList` フィールドに IP 範囲（例: `"10.0.0.1-10.0.0.50"`）を記述する。
2.  **Design**: 大規模なネットワークで全デバイスを検出する際、API 応答の `202 Accepted` を受け取った直後にデバイス一覧 API を叩いたがデータが空だった。なぜか？
    *   **回答**: Discovery API は非同期処理であり、ジョブ完了まで時間がかかるため。Task ID をポーリングして完了を確認してからインベントリを確認する必要がある。
3.  **トラブルシュート**: `/discovery` API で POST したが `400 Bad Request` が返ってきた。ペイロードの何を確認すべきか？
    *   **回答**: 必須フィールド（`name`, `ipAddressList` 等）の欠落、または JSON の構文エラー。
4.  **実装**: CDP 隣接関係を利用してデバイスを自動検出したい。使用すべき API パラメータは？
    *   **回答**: `discoveryType` を `"CDP"` に設定し、`ipAddressList` にシードデバイスの IP を指定する。
5.  **Design**: 検出されたデバイスを特定の「サイト」に自動割り当てしたい。Discovery API 単体で可能か？
    *   **回答**: いいえ。Discovery API でデバイスを検出・登録した後、別の `Site API` (Assign Device to Site) を使用して紐付ける必要がある。

---

## 🔗 参考リソース

*   **Cisco DNA Center API Reference**: [Network Discovery APIs](https://developer.cisco.com/docs/dna-center/api/2-2-2/)
*   **Cisco Configuration Guide**: [Cisco DNA Center User Guide - Discovery](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/dna-center/2-2-2/user-guide/b_cisco_dna_center_ug_2_2_2/b_cisco_dna_center_ug_2_1_2_chapter_010.html)
*   **Cisco Live (DEVNET-2012)**: [Network Automation with DNA Center APIs](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「Discovery は DNAC の目」です。API で「目」を動かすには、IP という「見る場所」と SNMP/CLI という「見るための眼鏡（資格情報）」をセットで渡す必要があると覚えましょう。
*   **注意点**: ラボ試験では、Python の `time.sleep()` が長すぎると試験時間が足りなくなるため、ポーリング間隔は 5〜10 秒程度に設定するのが実用的です。
*   **図解**: 
    - **Trigger**: POST /discovery -> Task ID
    - **Wait**: GET /task/{id} -> "COMPLETED"
    - **Check**: GET /discovery/{id}/summary -> Success Count.
