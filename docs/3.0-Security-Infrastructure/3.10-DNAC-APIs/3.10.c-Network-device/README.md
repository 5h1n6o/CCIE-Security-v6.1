---
layout: default
title: 3.10.c-Network-device
nav_order: 3
parent: 3.10-DNAC-APIs
grand_parent: 3.0-Security-Infrastructure
---

# 3.10.c Network device

Cisco DNA Center (DNAC) の **Network Device API** は、インベントリ内のネットワークデバイス（スイッチ、ルータ、ワイヤレスコントローラ、APなど）に関する情報の取得、管理、および診断を行うための主要な Northbound API カテゴリです。CCIE Security v6.1 ラボ試験では、Python スクリプトを使用してデバイス一覧の抽出、特定のデバイス詳細（UUIDやシリアル番号）の特定、さらには API 経由での診断コマンド実行（Command Runner）などが問われます,。

---

## 📘 概要

*   **機能概要**: DNAC が管理する全ネットワーク資産のデータベース（インベントリ）に対して、REST API を用いてプログラムからアクセスする機能です。
*   **利用目的**: 資産管理の自動化、デバイスの状態監視、大量のデバイスに対する一括情報収集、およびトラブルシューティングの迅速化。
*   **どのような場面で利用するか**: 
    *   **ダイナミックインベントリ**: 接続されている全ルータの IP アドレスとソフトウェアバージョンを抽出し、レポートを作成する。
    *   **コンプライアンスチェック**: セキュリティアドバイザリの対象となっているデバイスを特定する。
    *   **自動診断**: 特定の UUID を指定して `show` コマンドを API 経由で実行し、結果を外部ツールへ出力する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要エンドポイント** | `/dna/intent/api/v1/network-device` |
| **主な HTTP メソッド** | GET (情報取得), PUT (更新), DELETE (削除) |
| **識別子** | デバイス固有の **ID (UUID)** が操作の鍵となる。 |
| **データ形式** | リクエスト、レスポンス共に **JSON** を使用。 |
| **フィルタリング** | `hostname`, `managementIpAddress`, `macAddress` 等で検索可能。 |
| **連携機能** | Command Runner API と組み合わせて CLI コマンドを実行。 |

---

## 🏗 動作原理

Network Device API は、DNAC の内部データベースに格納された「ネットワークの状態（Intent）」に対するインターフェイスとして機能します。

```text
[ Python Script ]
      ↓ (API Call: GET /network-device)
[ Cisco DNA Center (Northbound) ]
      ↓ (Database Query)
[ Intent Database / Inventory ]
      ↑ (Southbound Sync: SNMP/SSH/NETCONF)
[ Physical Network Devices (Routers/Switches) ]
```

DNAC は常にデバイスと同期（再同期）を行っており、API を通じて提供される情報は物理デバイスの現在の状態を反映しています。

---

## ⚙ 動作シーケンス

1.  **UUID の取得**: 操作対象を特定するため、まず `managementIpAddress` などをクエリパラメータに指定して GET リクエストを送信し、デバイスの `id` (UUID) を取得します。
2.  **詳細情報の取得**: 取得した UUID をパスに含め（例: `/network-device/{id}`）、インターフェイス状態やライセンス情報などの詳細を取得します。
3.  **アクションの実行**: 設定変更や診断が必要な場合、UUID をペイロードに含めて Command Runner 等の別の API を叩きます。
4.  **結果のパース**: DNAC から返された JSON を Python の辞書型に変換し、必要なフィールド（例: `softwareVersion`, `reachabilityStatus`）を抽出します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **UUID の重要性**: ラボ試験では、デバイス名や IP アドレスではなく、API 内部で管理される **UUID** を使って操作を指示されることが多いため、名前から UUID を逆引きするロジックを必ず書けるようにしてください。
*   **ステータスの判定**: `reachabilityStatus` フィールドを確認し、デバイスが `Reachable` か `Unreachable` かをプログラムで判断するシナリオが想定されます。
*   **ページネーションの処理**: インベントリが膨大な場合、`offset` と `limit` パラメータを使用してデータを分割取得する知識が問われる可能性があります。
*   **Command Runner の併用**: 「全スイッチで API を使って `show version` を実行し、特定の脆弱性があるか確認せよ」といった複合問題に備えてください。
*   **JSON 構造の把握**: `response.json()['response']` の中にリスト形式でデバイス情報が入っている構造を正しくパースできる必要があります。

---

## 🛠 設定方法

### 1. デバイス一覧を取得する Python 基本スクリプト
```python
import requests
import json

# 認証トークン(token)は取得済みと仮定
dnac_ip = "10.1.1.10"
headers = {
    "X-Auth-Token": token,
    "Content-Type": "application/json"
}

# GETリクエストで全デバイスを取得
url = f"https://{dnac_ip}/dna/intent/api/v1/network-device"
response = requests.get(url, headers=headers, verify=False)

if response.status_code == 200:
    devices = response.json()['response']
    for device in devices:
        print(f"Hostname: {device['hostname']}, IP: {device['managementIpAddress']}, UUID: {device['id']}")
```

---

## 🔍 検証コマンド

| 目的 | API エンドポイント / 手法 |
| :--- | :--- |
| **全デバイス情報の取得** | <code>GET /dna/intent/api/v1/network-device</code> |
| **特定の IP アドレスで検索** | <code>GET /dna/intent/api/v1/network-device?managementIpAddress=10.1.1.1</code> |
| **デバイス総数の確認** | <code>GET /dna/intent/api/v1/network-device/count</code> |
| **特定 UUID の詳細確認** | <code>GET /dna/intent/api/v1/network-device/{id}</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| リストが空で返ってくる | 権限不足、またはインベントリ未登録 | API ユーザーの Role を確認し、DNAC GUI でデバイスが Discover されているか確認。 |
| 404 Not Found | 指定した UUID が存在しない | <code>/network-device</code> で最新の UUID 一覧を再取得する。 |
| 通信タイムアウト | DNAC サービスの過負荷 | <code>limit</code> パラメータを使用して 1 回の取得件数を減らす。 |
| 情報が古い | デバイスとの同期失敗 | <code>POST /network-device/sync</code> API を使用して強制同期を行う。 |

---

## ⚠ 制限事項

*   **API スロットリング**: 短時間での過度な GET リクエストは、DNAC 側でレート制限がかかる場合があります。
*   **情報の遅延**: 物理デバイスで設定変更が行われた直後、API 上のインベントリ情報に反映されるまで数分のタイムラグ（同期処理）が発生することがあります。
*   **サードパーティ製**: シスコ製以外のデバイスは、取得できる詳細情報フィールドが制限される場合があります。

---

## 🔄 他技術との関連

*   **3.10.a Authentication**: デバイス情報を取得するための事前条件。
*   **3.10.b Network Discovery**: デバイスを API 経由でインベントリに登録する前段階のプロセス。
*   **3.10.d Network Host**: デバイスに接続されている「エンドポイント（クライアント）」の情報を取得する API と連携。

---

## 🧩 比較表

### Network Device API vs CLI (Show run/inventory)

| 特徴 | Network Device API | CLI (Device Direct) |
| :--- | :--- | :--- |
| **情報の集約** | DNAC で全台分を一括取得可能 | 1 台ずつログインが必要 |
| **形式** | 構造化データ (JSON) で解析容易 | テキスト形式でパースが必要 |
| **リアルタイム性** | DNAC データベースの同期状況に依存 | 常に最新 |
| **主な用途** | 資産管理、自動化、レポート | 詳細なデバッグ、緊急操作 |

---

## 💡 ベストプラクティス

1.  **UUID のキャッシュ**: 頻繁に操作するデバイスの UUID は変数に格納し、API コール回数を最小限にします。
2.  **フィルタリングの活用**: 全件取得してから Python で加工するのではなく、クエリパラメータ（例: `family=Switches`）を使用して DNAC 側で絞り込ませます。
3.  **エラーハンドリング**: `response.json().get('response', [])` のように `get()` メソッドを使い、キーが存在しない場合のエラーを回避します。
4.  **アシュアランス連携**: デバイス情報だけでなく、`Health API` と組み合わせて、セキュリティ上問題のある（Health スコアの低い）デバイスを自動特定します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 特定のホスト名を持つデバイスの UUID 特定
*   **要件**: "Core-SW" というホスト名のデバイス UUID を見つけよ。
*   **設定**: 
    ```python
    res = requests.get(f"{url}?hostname=Core-SW", headers=headers, verify=False)
    uuid = res.json()['response']['id']
    ```

### 2. ソフトウェアバージョンの一括出力
*   **要件**: 全ルータのソフトウェアバージョンを表示せよ。
*   **設定**: `for dev in devices: if dev['family'] == 'Routers': print(dev['softwareVersion'])`

### 3. 未同期（Unreachable）デバイスのリストアップ
*   **要件**: 状態が `Unreachable` なデバイスの IP を出力せよ。

### 4. シリアル番号によるデバイス特定
*   **要件**: シリアル番号 "SN12345" のデバイスの管理 IP を取得せよ。

### 5. デバイス総数のカウント
*   **要件**: API を使用して登録済みデバイスの合計数を表示せよ。
*   **エンドポイント**: `/network-device/count`

### 6. 特定サイトに所属するデバイスの抽出
*   **操作**: `siteId` パラメータを使用して、"Tokyo" サイトのデバイスのみ取得せよ。

### 7. インターフェイス情報の取得
*   **要件**: 特定の UUID に対し、そのデバイスが持つ全インターフェイス名をリストせよ。

### 8. デバイスの削除リクエスト
*   **要件**: UUID を指定してインベントリからデバイスを削除せよ。

### 9. 最終同期時刻の確認
*   **操作**: JSON 内の `lastUpdateTime` フィールドをパースして表示せよ。

### 10. Command Runner との連携
*   **問題**: API で取得した UUID を使い、そのデバイスに対して `show ip int brief` を実行せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `GET /network-device?macAddress=00:11:22:33:44:55` を実行して返ってくる情報の種類は？
    *   **回答**: その MAC アドレスを持つネットワークデバイス（スイッチ等）のインベントリ詳細情報。
2.  **トラブルシュート**: デバイス UUID を使って PUT リクエストを送ったが `401 Unauthorized` となった。何が間違っているか？
    *   **回答**: 認証トークンがヘッダーに含まれていないか、有効期限が切れている。
3.  **Design**: プログラムで「特定のソフトウェアバージョン (17.3.1)」を持つデバイスのみにセキュリティパッチを当てる際、最初に叩くべき API は？
    *   **回答**: `GET /dna/intent/api/v1/network-device`。
4.  **実装**: ラボ試験で「デバイス一覧の JSON 応答からホスト名のみを抜き出してリスト化せよ」と言われた際の Python 構文は？
    *   **回答**: `hostnames = [d['hostname'] for d in response.json()['response']]`。
5.  **Design**: 大量のデバイス情報を取得する際、ネットワーク帯域を節約し DNAC の負荷を下げるための API パラメータは？
    *   **回答**: `limit` パラメータによる取得件数の制限。

---

## 🔗 参考リソース

*   **Cisco DNA Center API Reference**: [Network Device APIs](https://developer.cisco.com/docs/dna-center/api/2-2-2/)
*   **Configuration Guide**: [Cisco DNA Center User Guide - Inventory Management](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/dna-center/2-2-2/user-guide/b_cisco_dna_center_ug_2_2_2.html)
*   **Cisco Live (DEVNET-2012)**: [Network Automation with DNAC APIs](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「Network Device API = デバイスの名簿」と覚えましょう。名簿には名前（Hostname）だけでなく、背番号（UUID）や住所（IP）、健康状態（Health）が書かれています。
*   **図解**: 
    - **GET /network-device**: 名簿全体を見る。
    - **GET /network-device/{id}**: 特定の個人の詳細プロフィールを見る。
*   **注意点**: ラボ環境の DNAC は応答が重い場合があるため、スクリプト内でリクエストを送る際は `timeout` 設定を入れるか、十分な待機時間を考慮してください。
