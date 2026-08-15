---
layout: default
title: 3.10.d-Network-host
nav_order: 4
parent: 3.10-DNAC-APIs
grand_parent: 3.0-Security-Infrastructure
---

# 3.10.d Network host

Cisco DNA Center (DNAC) における **Network Host API** は、ネットワークに接続されている「エンドポイント（ホスト）」の情報を取得・管理するためのインターフェイスです。これには、有線・無線クライアントの IP アドレス、MAC アドレス、接続先のスイッチ、ポート、VLAN 情報などが含まれます,。CCIE Security v6.1 においては、Python スクリプトを使用して特定のホストを追跡したり、セキュリティ監査のためにクライアントの接続状況を可視化したりする能力が求められます。

---

## 📘 概要

*   **機能概要**: DNAC が収集したネットワーク内の全ホスト（クライアント）情報を REST API 経由で提供する機能です。
*   **利用目的**: 特定のユーザーやデバイスが「ネットワークのどこに接続されているか」をプログラムで特定し、可視化や資産管理、セキュリティイベントの調査に利用します。
*   **どのような場面で利用するか**: 
    *   **セキュリティ調査**: 不審な動きをした IP アドレスの物理的な接続場所（スイッチ名とポート）を即座に特定する。
    *   **資産管理**: 現在ネットワークにアクティブなクライアントのリストを取得し、社内の台帳と照合する。
    *   **トラブルシューティング**: クライアントの接続履歴や現在の VLAN 割り当て状況を確認する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主なエンドポイント** | `/dna/intent/api/v1/network-host` |
| **取得できる主な情報** | Host IP, MAC, VLAN, Connected Device (Switch/AP), Port,。 |
| **フィルタリング要素** | `hostIp`, `hostMac`, `connectedDeviceName`, `vlanId` 等で検索可能。 |
| **データソース** | スイッチからの SNMP, Syslog, NetFlow, および ISE からのデータ。 |
| **CCIE での重要度** | 「特定の IP を持つホストの接続先を特定せよ」といった自動化タスクで頻出。 |

---

## 🏗 動作原理

Network Host API は、DNAC がネットワークデバイスから収集したテレメトリデータを抽象化して提供します。

```text
[ Endpoint (PC/IoT) ]
      ↓
[ Access Switch / AP ] ── (Telemetry: SNMP/Syslog/NetFlow) ─→ [ Cisco DNA Center ]
                                                                    ↓ (DB Query)
[ Python Script ] ←───── (REST API: GET /network-host) ───── [ Host Database ]
```

1.  **データ収集**: スイッチや AP が、接続されたホストの情報を DNAC へ送信します（Southbound）。
2.  **正規化**: DNAC は複数のデバイスからの情報を統合し、一意のホストエントリとしてデータベースに保存します。
3.  **API 提供**: Python スクリプトからのリクエストに対し、構造化された JSON 形式でホストの詳細を返します。

---

## ⚙ 動作シーケンス

1.  **認証**: `/dna/system/api/v1/auth/token` でトークンを取得します。
2.  **ホスト検索**: `GET /dna/intent/api/v1/network-host?hostIp=10.1.1.5` のように、特定の IP や MAC を指定してリクエストを送信します。
3.  **接続先特定**: レスポンスに含まれる `connectedNetworkDeviceIpAddress` や `connectedInterfaceName` フィールドを確認します。
4.  **詳細解析**: 必要に応じて、ホストが接続されているスイッチの UUID を使用し、`Network Device API` でスイッチの詳細情報を取得して紐付けます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **特定ホストの追跡**: 「192.168.1.100 のホストがどのスイッチのどのポートにいるか出力せよ」という問題は、API 連携の典型例です。
*   **JSON のネスト構造**: ホスト情報は `response['response']` というリストの中に格納されます。`for host in response['response']:` でループを回す処理をマスターしてください。
*   **クエリパラメータの活用**: 全ホストを取得してから Python で filter するのではなく、URL に `?hostMac=xxxx` を付けて、DNAC 側で絞り込ませる手法が推奨されます。
*   **ISE との連携**: DNAC が ISE と連携している場合、ホスト情報に SGT (Scalable Group Tag) が含まれることがあります。
*   **空レスポンスの処理**: 検索条件に一致するホストがいない場合、空のリスト `[]` が返ります。エラーにならずに「Not Found」と出力するハンドリングが必要です。

---

## 🛠 設定方法

### 1. Python による特定の IP を持つホストの接続先特定
```python
import requests
import json

# 認証トークン(token)取得済みの前提
dnac_ip = "10.1.1.10"
headers = {"X-Auth-Token": token, "Content-Type": "application/json"}

target_ip = "10.1.100.55"
url = f"https://{dnac_ip}/dna/intent/api/v1/network-host?hostIp={target_ip}"

response = requests.get(url, headers=headers, verify=False)
host_data = response.json().get('response', [])

if host_data:
    host = host_data
    print(f"Host IP: {host['hostIp']}")
    print(f"Connected to Device: {host['connectedNetworkDeviceName']}")
    print(f"Interface: {host['connectedInterfaceName']}")
    print(f"VLAN: {host['vlanId']}")
else:
    print("Host not found.")
```

---

## 🔍 検証コマンド

| 目的 | 手法 / エンドポイント |
| :--- | :--- |
| **全ホスト一覧の取得** | <code>GET /dna/intent/api/v1/network-host</code> |
| **MACアドレスでの検索** | <code>GET /network-host?hostMac=00:11:22:33:44:55</code> |
| **特定のスイッチに繋がる全ホスト** | <code>GET /network-host?connectedNetworkDeviceName=Access-SW1</code> |
| **ホスト総数の確認** | <code>GET /dna/intent/api/v1/network-host/count</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| ホスト情報が表示されない | デバイス側でのテレメトリ未設定 | DNAC GUI でデバイスの **Telemetry** が `Enabled` か確認。 |
| IP アドレスが `None` | ARP/DHCP 情報が未収集 | デバイスで DHCP Snooping や IP Device Tracking (IPDT) が有効か確認。 |
| 401 Unauthorized | トークン期限切れ | トークンを再取得し <code>X-Auth-Token</code> を更新。 |
| 接続先が古い/間違っている | DNAC とデバイスの同期遅延 | `POST /network-device/sync` でデバイスを強制同期させる。 |

---

## ⚠ 制限事項

*   **情報の鮮度**: API で返されるホスト情報は、DNAC データベースの最新の同期状態に基づきます。リアルタイムの物理状態と数分のタイムラグがある場合があります。
*   **非アクティブホスト**: 一定期間通信がないホストは、API の結果から削除される（DNAC のクリーンアップ設定に依存）場合があります。
*   **仮想化ホスト**: 同一ポートに複数のホスト（VM 等）がある場合、適切にリストされますが、物理トポロジの解釈に注意が必要です。

---

## 🔄 他技術との関連

*   **3.10.c Network Device**: ホストの接続先スイッチの詳細（型番や場所）を取得するために併用します。
*   **3.4.e DHCP Snooping**: ホストの IP アドレスを DNAC が学習するための主要なソースです。
*   **3.6.a NetFlow / Telemetry**: クライアントのトラフィック統計を取得するための基盤。
*   **ISE (ERS API)**: ホストの認証プロファイルやポスチャ状態をさらに詳しく知るために ISE API と組み合わせます。

---

## 🧩 比較表

### Network Host API vs CLI (`show mac address-table`)

| 特徴 | Network Host API | CLI (Switch Direct) |
| :--- | :--- | :--- |
| **範囲** | **ネットワーク全体を一括検索** | 1 台のスイッチ内のみ |
| **情報の深さ** | IP, MAC, VLAN, スイッチ名を紐付け | L2 情報（MAC/Port）のみが基本 |
| **履歴** | 過去の接続履歴を確認可能 | 現在の状態のみ |
| **自動化** | Python での大量処理に最適 | 手動または Expect スクリプトが必要 |

---

## 💡 ベストプラクティス

1.  **バルク取得の制限**: ホスト数が数千ある環境では、`limit` パラメータを使用してページネーションを実装し、DNAC の負荷を抑えます。
2.  **MAC アドレス形式の正規化**: API への入力前に、MAC アドレスを `aa:bb:cc...` または `aaaa.bbbb...` などの形式に Python 側で揃えます。
3.  **例外処理**: ホスト情報が取得できなかった場合（空の応答）の `IndexError` を防ぐため、リストの長さを必ずチェックします。

---

## 📝 ラボ学習・設定サンプル例

### 1. 全ホストの IP/MAC 対応表の作成
*   **問題**: インベントリ内の全ホストをリストアップし、IP と MAC を CSV 形式で出力せよ。
*   **設定例**: `for h in resp['response']: print(f"{h['hostIp']},{h['hostMac']}")`

### 2. 特定 VLAN に所属するホストの抽出
*   **要件**: VLAN 10 に接続されている全クライアントを表示せよ。
*   **設定**: `GET /network-host?vlanId=10`

### 3. 不審な MAC アドレスの物理位置特定
*   **問題**: MAC `0000.1111.2222` のホストが現在どのスイッチポートにいるか答えよ。

### 4. 無線クライアントのみのフィルタリング
*   **要件**: 接続タイプが `Wireless` であるホストのみをカウントせよ。

### 5. ホスト情報の JSON 整形出力 (Pretty Print)
*   **操作**: `json.dumps(data, indent=4)` を使用して、デバッグ用にホスト詳細を表示せよ。

### 6. スイッチ名から接続ホストを逆引き
*   **問題**: "Edge-SW-01" に接続されている全ホストの IP を表示せよ。

### 7. 未知の IP アドレス（Static IP）ホストの特定
*   **要件**: DNAC に登録されているが、IP アドレス情報が欠落しているホストをリストアップせよ。

### 8. 複数 IP アドレスを持つホストの処理
*   **操作**: 1 つのホストエントリに複数の IP が紐付いている場合のパース方法を確認せよ。

### 9. 接続ポートのステータス確認との連携
*   **要件**: Network Host API で特定したポートに対し、Command Runner で `show interface` を実行せよ。

### 10. サイト（拠点）ごとのホスト数集計
*   **問題**: 特定の `siteId` をパラメータに含め、拠点ごとのクライアント数を算出せよ。

---

## ❓ 想定試験問題

1.  **実装**: Python スクリプトで特定の MAC アドレス `00:50:56:88:99:AA` を持つホストの詳細を取得するための API パラメータを記述せよ。
    *   **回答**: `?hostMac=00:50:56:88:99:AA`。
2.  **Design**: 大規模ネットワークで、特定の IP アドレスが「どのスイッチのどのポートに接続されているか」を最も効率的に特定する Northbound API カテゴリは？
    *   **回答**: **Network Host API** (`/network-host`)。
3.  **トラブルシュート**: API を叩いてもホストの IP アドレスが `null` で返ってくる。ネットワークデバイス側で不足している可能性のある設定は何か？
    *   **回答**: **DHCP Snooping** または **IP Device Tracking (IPDT)**。
4.  **コンフィグ読解**: `host['connectedNetworkDeviceIpAddress']` フィールドが示す意味を説明せよ。
    *   **回答**: ホストが物理的に（または無線で）接続されているアクセス層デバイス（スイッチや WLC）の管理 IP アドレス。
5.  **Design**: ネットワーク内の有線クライアントと無線クライアントの合計数を API 経由で取得したい。最適なエンドポイントは？
    *   **回答**: `GET /dna/intent/api/v1/network-host/count`。

---

## 🔗 参考リソース

*   **Cisco DNA Center API Reference**: [Network Host APIs](https://developer.cisco.com/docs/dna-center/api/2-2-2/)
*   **Cisco Configuration Guide**: [Cisco DNA Center User Guide - Host Inventory](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/dna-center/2-2-2/user-guide/b_cisco_dna_center_ug_2_2_2.html)
*   **Cisco Live**: [Automating Network Operations with DNAC Northbound APIs](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「Network Device API = インフラ機器の名簿」「Network Host API = 利用者の名簿」と区別してください。
*   **図解**: ホスト情報は、デバイス（スイッチ）の情報と密接にリンクしています。JSON の中で `connectedNetworkDeviceName` を見つけたら、それが「親」であるスイッチだと認識しましょう。
*   **注意点**: ラボ試験では、MAC アドレスの形式（ハイフン、コロン、ピリオド）が、DNAC のデータベースに登録されている形式と一致しないと検索に失敗する場合があるため、ワイルドカードや柔軟なパースを心がけてください。
