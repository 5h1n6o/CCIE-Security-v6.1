---
layout: default
title: 3.10.d-Network-host
nav_order: 4
parent: 3.10-DNAC-APIs
grand_parent: 3.0-Security-Infrastructure
---

# 3.10.d Network host

## 📘 概要

Cisco DNA Center (Catalyst Center) Northbound Intent APIにおける **Network Host / Client API** は、ネットワークに接続されているすべてのエンドポイント（有線・無線クライアント、IP電話、IoTデバイス、サーバー等）の接続状態、位置情報、IP/MACアドレスマッピング、接続先ネットワークデバイス（スイッチ/WLC）およびポート情報、通信品質（Client Health Score）、VLAN割り当て状態などをプログラマブルに取得・解析するための集中APIインターフェースです。

CCIE Security v6.1 において本技術は、セキュリティインシデント発生時の「脅威端末の迅速な追跡」「感染端末の物理接続ポートおよびSSID/WLCの自動特定」「Cisco ISE (Identity Services Engine) やpxGridと連携した動的アクセス制御・検疫 (Quarantine) の前処理」、ならびに「ネットワーク可視化 (Visibility) およびアシュアランス (Assurance) 自動化」を実現するキーコンポーネントとして評価されます。

主な利用目的および運用シナリオは以下の通りです。

* **インシデントレスポンス自動化**: IDS/IPS (Firepower/Snort) やSIEM (Splunk/Stealthwatch) で検知された過失・攻撃端末のIP/MACアドレスをキーにして、DNAC API経由で物理スイッチ名、接続ポート、VLAN、アソシエーションWLC/SSID、および認証情報を自動特定。
* **クライアント健全性（Client Health）の監視と診断**: 無線・有線クライアントのHealth Score (1-10)、SNR、RSSI、DHCP/RADIUSハンドシェイク成功率、DNS応答速度等の時系列メトリクスを自動収集。
* **ダイナミックネットワークトポロジーの把握**: 端末が移動（ローミングやVLAN変更）した際のリアルタイム接続先トラッキングおよび履歴取得。
* **ゼロトラスト・セキュリティ（Zero Trust / Micro-segmentation）の可視化補助**: 接続端末のタイプやSGT (Security Group Tag) 割り当て状態をAPI経由で集計・監査。


---

## 🔑 要点

Network Host / Client APIの主要な特徴、用途、メリット・デメリット、対応機種、制限事項および設計上の注意点を下表に整理します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | エンドポイントのIP/MAC、接続先スイッチ/Port/WLC/SSID、VLAN、SGT、Health Scoreなどの総合的な可視化データを単一のREST APIエンドポイント経由で提供 |
| **用途** | エンドポイントの即時検索、接続トポロジー特定、クライアント障害診断、セキュリティインシデント発生時の物理ポート特定および検疫スクリプトの入力データ生成 |
| **メリット** | スイッチやWLCへの個別のCLI/SNMPアクセスが不要。全ネットワークに跨る端末の接続先と状態を単一のリクエスト（`GET /dna/intent/api/v1/host` や `/client-detail`）で即座に取得可能 |
| **デメリット** | 多数のクライアント（数千〜数万台）が存在する環境では、レスポンスサイズが肥大化しページネーション処理が必須。またDNACのホストトラッキングDB更新サイクル（数秒〜数分）による遅延が存在する場合がある |
| **対応機種** | Cisco DNA Center (Catalyst Center) アプライアンス (2.x / 3.x)。対象エンドポイントはCatalyst 9000シリーズ、AireOS/Catalyst WLC、Cisco IOS-XEスイッチ等に接続された全端末 |
| **制限事項** | クライアントのリアルタイム情報収集には、スイッチ/WLC側でNetFlow/Telemetry/Syslog/SNMP trap等のDNAC連携設定が正しく完了している必要がある |
| **設計上の注意点** | API実行時はIPアドレスだけでなくMACアドレス形式（コロン区切り `AA:BB:CC:DD:EE:FF` やハイフン区切り `aa-bb-cc-dd-ee-ff`）の表記ブレに配慮した正規化処理がスクリプト側に必須 |


---

## 🏗 動作原理

Cisco DNA Centerにおける Network Host / Client 情報の収集・提供メカニズムは、ネットワーク装置群からの各種テレメトリ収集と、Northbound APIを通じたサードパーティ/スクリプトへのデータ提供の2層構造で動作します。

```
[ エンドポイント (PC / スマホ / IP Phone / IoT) ]
         │ (DHCP / EAPoL / ARP / CDP / LLDP)
         ▼
[ ネットワークアクセスデバイス (Catalyst Switch / WLC) ]
         │
         ├─ Syslog / SNMP Traps / Telemetry / NetFlow (サウスボウンド通信)
         ▼
[ Cisco DNA Center (Catalyst Center) ]
         │
         ├─ Host Tracking Database & Assurance Engine (データインデックス化)
         ▼
[ Northbound Intent API (REST Service) ]
         │
         ├─ GET /dna/intent/api/v1/host
         ├─ GET /dna/intent/api/v1/client-detail
         └─ GET /dna/intent/api/v1/client-enrichment-details
         ▼
[ 自動化スクリプト / SecOpsツール / Python (CCIE Lab) ]
```

### コアコンポーネントと役割

1. **DNAC Host Tracking Database**:
   * スイッチからのARP/MACテーブル情報、DHCP Snooping、802.1X/MAB認証イベント、CDP/LLDP隣接情報、WLCからのアソシエーション情報を集約し、端末ごとのユニークプロファイルをリアルタイムに維持。
2. **Intent API Host/Client Endpoints**:
   * `GET /dna/intent/api/v1/host`: ネットワーク上の接続ホスト一覧または特定IP/MAC/HostTypeの端末情報を取得。
   * `GET /dna/intent/api/v1/client-detail`: 特定クライアント（MACアドレスキー）の接続履歴、有線/無線詳細、アソシエーションAP、RSSI/SNR、L2/L3トポロジーパス、認証情報を取得。
   * `GET /dna/intent/api/v1/client-enrichment-details`: 複数エンドポイントの健全性およびトラブルシューティング関連情報を一括補強取得。


---

## ⚙ 動作シーケンス

Pythonスクリプトが DNAC API を使用して特定ホストの物理位置と接続状態を特定し、セキュリティインシデントに対応する際のパケットおよび制御シーケンスは以下の通りです。

```
[ Python Script ]          [ DNAC Northbound API ]       [ Network Devices (Switch/WLC) ]
       │                              │                                │
       │ 1. POST /auth/token          │                                │
       ├─────────────────────────────>│                                │
       │ 2. 200 OK (x-auth-token)     │                                │
       │<─────────────────────────────┤                                │
       │                              │                                │
       │ 3. GET /v1/host?hostIp=x.x.x.x│                                │
       ├─────────────────────────────>│ ネットワーク内DBをクエリ       │
       │                              │<───────────────────────────────┤ (SNMP/Syslog/Telemetry)
       │ 4. 200 OK (JSON Data)        │                                │
       │<─────────────────────────────┤                                │
       │  [解析: MAC, ConnectedDevice,│                                │
       │   Port, VLAN, Subnet, Health]│                                │
       │                              │                                │
       │ 5. GET /v1/client-detail     │                                │
       │    ?timestamp=...&macAddress=│                                │
       ├─────────────────────────────>│ 詳細アシュアランスメトリクス取得│
       │ 6. 200 OK (Client Detail)    │                                │
       │<─────────────────────────────┤                                │
       │                              │                                │
       │ 7. (必要に応じて) 物理ポートを特定し Quarantine または Port Shutdown スクリプトを実行
```

### シーケンスの詳細ステップ

1. **認証（Authentication）**:
   * スクリプトは `POST /dna/system/api/v1/auth/token` へ Basic 認証リクエストを送信し、ヘッダー付与用の `x-auth-token` を取得。
2. **ホスト基本情報の照会（Host Lookup）**:
   * 調査対象の IP アドレス（例: `192.168.10.50`）を指定し、`GET /dna/intent/api/v1/host?hostIp=192.168.10.50` を発行。
   * レスポンス配列（`response` 階層）から、`hostMac`、`connectedNetworkDeviceIpAddress`、`connectedNetworkDeviceName`、`connectedInterfaceName`、`vlanId` を抽出。
3. **クライアント詳細アシュアランス照会（Client Detail Lookup）**:
   * 抽出した MAC アドレス（例: `00:11:22:33:44:55`）を用いて `GET /dna/intent/api/v1/client-detail?macAddress=00:11:22:33:44:55` を実行。
   * 無線接続の場合はアソシエーションSSID、BSSID、WLC IP、RSSI (`-65 dBm` 等)、SNR (`30 dB` 等)、有線接続の場合はデュプレックス、スピード、Dot1X認証状態を取得。
4. **自動対向処理の実行（Action Handoff）**:
   * 取得した接続先スイッチIPおよびインターフェース名をもとに、スイッチへのCLIアクセスまたは外部セキュリティAPIへ連携し隔離アクションを実施。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験における **3.10.d Network host API** の出題ポイント、典型的な自動化要件、よくあるハマりポイント、およびデバッグ・状態確認方法をまとめます。

### 1. Blueprintにおける重要ポイント
* **特定条件によるホスト検索スクリプトの作成**: 特定のVLAN IDに属する全ホストのIPと接続先ポートをリストアップ、指定IPアドレスの接続先アクセススイッチとポート名を特定する自動化タスク。
* **MACアドレスフォーマットの統一**: DNAC APIではMACアドレスが `00:11:22:33:44:55` や `0011.2233.4455` などフォーマットが異なる場合があるためPython側で正規化が必要。
* **レスポンス構造の階層把握**: `GET /dna/intent/api/v1/host` のレスポンスは `{'response': [{...}, {...}]}` の構造。対象が存在しない場合は空リスト `response: []` となる。
* **主要キーの理解**: `hostIp`, `hostMac`, `hostType`, `connectedNetworkDeviceIpAddress`, `connectedNetworkDeviceName`, `connectedInterfaceName`, `vlanId`

### 2. ラボ試験で想定される実装課題
* 特定サブネット内の不正IPアドレスが接続されている物理スイッチとポートをAPIで特定するスクリプト
* 有線・無線クライアント一覧を取得し、Client Health Scoreが3以下の端末のMACと接続先を抽出するスクリプト
* 指定MACアドレス端末のVLANと接続ポートを取得し、外部システムへ連携するスクリプト

### 3. よくある設定・実装ミスとハマりポイント
* **認証トークン期限切れ**: トークン期限（1時間）が切れた際の401エラー対策
* **クエリパラメタのタイポ**: `hostIp` (hは小文字、Iは大文字) など正確なキャメルケース指定が必要
* **IPとMACの検索パラメタ間違い**: `hostIp` と `hostMac` の使い分け
* **SSL検証エラー**: ラボ環境では `verify=False` が必要

---

## 🛠 設定方法

Network Host API は Cisco DNA Center の Northbound インターフェースとして標準提供されています。

### 1. ホスト基本情報検索スクリプト (`GET /dna/intent/api/v1/host`)

```python
import requests
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_HOST = '192.168.1.100'
DNAC_USER = 'admin_user'
DNAC_SECRET = 'dnac_secret'

def get_auth_token():
    url = f'https://{DNAC_HOST}/dna/system/api/v1/auth/token'
    res = requests.post(url, auth=(DNAC_USER, DNAC_SECRET), verify=False)
    res.raise_for_status()
    return res.json()['Token']

def find_host_by_ip(token, target_ip):
    url = f'https://{DNAC_HOST}/dna/intent/api/v1/host'
    headers = {'x-auth-token': token, 'Content-Type': 'application/json'}
    params = {'hostIp': target_ip}
    response = requests.get(url, headers=headers, params=params, verify=False)
    response.raise_for_status()
    hosts = response.json().get('response', [])
    if not hosts:
        print(f'[-] Host {target_ip} not found.')
        return None
    host_info = hosts[0]
    print(f'[+] Host Found: {target_ip}')
    print(f'    MAC: {host_info.get("hostMac")}')
    print(f'    Type: {host_info.get("hostType")}')
    print(f'    Switch IP: {host_info.get("connectedNetworkDeviceIpAddress")}')
    print(f'    Device Name: {host_info.get("connectedNetworkDeviceName")}')
    print(f'    Port: {host_info.get("connectedInterfaceName")}')
    print(f'    VLAN: {host_info.get("vlanId")}')
    return host_info

if __name__ == '__main__':
    tok = get_auth_token()
    find_host_by_ip(tok, '10.1.100.50')
```

---

## 🔍 検証コマンド

| 目的 | 確認手段 / コマンド |
| --- | --- |
| **APIトークン検証** | `curl -s -k -X POST -u "USER:SECRET" "https://<DNAC>/dna/system/api/v1/auth/token"` |
| **特定IPのホスト取得 (cURL)** | `curl -s -k -H "x-auth-token: <TOKEN>" "https://<DNAC>/dna/intent/api/v1/host?hostIp=10.1.100.50"` |
| **特定MACのクライアント詳細取得** | `curl -s -k -H "x-auth-token: <TOKEN>" "https://<DNAC>/dna/intent/api/v1/client-detail?macAddress=00:11:22:33:44:55"` |
| **スイッチ側ホスト追跡確認 (CLI)** | <code>show device-tracking database</code> |
| **スイッチ側MACテーブル確認 (CLI)** | <code>show mac address-table address <MAC></code> |
| **WLC側無線クライアント確認 (CLI)** | <code>show wireless client summary</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認方法 | 対処方法 |
| --- | --- | --- | --- |
| **`hostIp` 検索で `response: []` が返る** | 該当ホストが非アクティブ、またはDNAC DBに未登録 | <code>show device-tracking database</code> | Ping等でARP更新、サウスボウンドTelemetry設定再確認 |
| **HTTP 401 Unauthorized** | トークン不整合または期限切れ | ヘッダー名 `x-auth-token` 確認 | トークン再取得処理を実行 |
| **`connectedInterfaceName` が None** | 端末が無線接続 (WLC配下) である場合 | `hostType` 確認 (`WIRELESS`) | `GET /client-detail` でAP/SSID情報を取得 |
| **HTTP 400 Bad Request** | クエリパラメタの指定ミス | エラーレスポンスの `code` 確認 | パラメタ名を `hostIp` 等の正しい記法に修正 |
| **SSLエラー発生** | self-signed証明書の検証失敗 | 例外ログ確認 | `verify=False` を指定 |


## ⚠ 制限事項

* **データ反映のタイムラグ**: エンドポイントの切断やIP変更がDNACの Host Database に反映されるまでに数秒〜数分のタイムラグ（Polling interval / Telemetry delay）が存在する場合がある。
* **無効なMACフォーマットでの拒否**: `GET /client-detail` 等のエンドポイントではコロン区切り (`AA:BB:CC:DD:EE:FF`) またはハイフン区切りの正確なフォーマットが必要。フォーマットが不正な場合 HTTP 400 エラーを返す。
* **大量データ取得時のレートリミット**: `GET /dna/intent/api/v1/host` でクエリパラメタなしで全ホストを取得する場合、デフォルトページサイズ制限（最大500件程度）が適用されるため `offset` および `limit` パラメタによるループ取得が必要。
* **マルチホーム・デュアルスタック端末の扱い**: IPv4/IPv6のデュアルスタック端末や複数NICを保持するサーバーの場合、単一のMACアドレスに対して複数のIPアドレスエントリが生成される場合がある。


---

## 🔄 他技術との関連

Network Host APIは単体でのモニタリングだけでなく、セキュリティプラットフォームや認証技術と強力に連携します。

* **Cisco ISE (Identity Services Engine) & pxGrid / ANC**: DNAC APIで検出した不正端末のMACアドレスを即座にISE Adaptive Network Control (ANC) API経由で Quarantine / Shut Down ポリシーへ適用。
* **Firepower Management Center (FMC)**: Snort3 / IPSが検知した攻撃元IPアドレスをDNAC Host APIに入力し、該当端末の物理アクセススイッチおよびポートを自動特定して隔離。
* **Stealthwatch (Secure Network Analytics)**: 異常トラフィック（C&C通信や横展開攻撃）を検知した際、該当IPアドレスからDNAC Host APIをコールし、端末ユーザーおよび物理ポートを特定。
* **IPDT (IP Device Tracking) & SISF**: アクセススイッチ側でIPDT / SISF (Software Impacted Device Tracking) が正しく動作していない場合、DNACが端末のIP/MACマッピングを学習できないため、Switch設定との強い依存関係が存在する。


---

## 🧩 比較表

Cisco DNA Center における主要な端末可視化・追跡APIおよび従来の個別CLI/SNMP追跡手法を下表で比較します。

| 比較項目 | GET /dna/intent/api/v1/host | GET /dna/intent/api/v1/client-detail | スイッチ個別CLI (`show device-tracking`) | Cisco ISE pxGrid Session API |
| --- | --- | --- | --- | --- |
| **目的** | 端末のIP/MAC/接続デバイス/Portの基本特定 | 有線・無線の詳細アシュアランス・健康度・RSSI等調査 | 特定スイッチ配下の直接接続端末の確認 | RADIUS認証セッションおよびSGT・プロファイル情報確認 |
| **データ粒度** | 基本接続メタデータ | 詳細時系列メトリクス・通信品質スコア | 物理L2/L3バインディングテーブル | 認証ユーザー・802.1X/MAB・SGT・グループ |
| **対応端末** | 有線・無線全般 | 有線・無線全般 | 有線直結端末のみ（スイッチ単位） | RADIUS認証を経由した端末のみ |
| **リアルタイム性** | 高（DNAC DBインデックス） | 極めて高（リアルタイムアシュアランス） | 即時（ローカルスイッチ制御プレーン） | 即時（認証時およびセッション更新時） |
| **CCIEラボでの用途** | IP/MACからの接続スイッチ・Port自動特定 | クライアント健康スコア調査・無線RSSI確認 | デバイス単体でのローカルバインディング確認 | 認証連携およびダイナミックポリシー適用確認 |


---

## 💡 ベストプラクティス

* **MACアドレス正規化関数の共通化**: 入力されるMACアドレスの形式（ドット区切り `0011.2233.4455`、ハイフン区切り `00-11-22-33-44-55`、コロン区切り `00:11:22:33:44:55`）を統一してコロン区切り小文字/大文字へ変換するヘパー関数を必ず作成する。
* **レスポンス存在チェックの徹底**: API呼び出し後、`if response.json().get('response'):` による配列存在チェックを行い、空リストの場合にIndexErrorでスクリプトが異常終了するのを防ぐ。
* **ページネーション処理の実装**: `limit=50` および `offset=1` から開始し、取得件数が `limit` と同数の間ループさせて全全ホストを取得する汎用ロジックをテンプレート化する。
* **例外処理 (`try-except`) の組み込み**: `requests.exceptions.HTTPError` および `requests.exceptions.ConnectionError` をキャッチし、ログ出力および安全な終了（sys.exit(1)）を行う。


## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルを想定した 10 個の実践的な Python 自動化スクリプト演習例です。すべてのスクリプトはエラーハンドリングおよびレスポンスパースを含み、実用に耐えうる完全なコンフィグ/コードブロックとして記載しています。

### 演習1: 特定IPアドレスのホスト物理接続位置（スイッチ・ポート・VLAN）の特定

#### 要件
* 指定されたターゲットIP（例: `10.1.100.50`）を引数として受領。
* DNAC Host API (`GET /dna/intent/api/v1/host?hostIp=...`) を呼び出し。
* 端末のMACアドレス、接続スイッチ名、スイッチIP、接続ポート名、VLAN IDを整形して標準出力へ表示。

#### 設定・実装例

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_IP = '192.168.1.100'
API_USER = 'admin_user'
API_PASS = 'dnac_secret'

def get_token():
    url = f'https://{DNAC_IP}/dna/system/api/v1/auth/token'
    res = requests.post(url, auth=(API_USER, API_PASS), verify=False)
    res.raise_for_status()
    return res.json()['Token']

def locate_host(token, target_ip):
    url = f'https://{DNAC_IP}/dna/intent/api/v1/host'
    headers = {'x-auth-token': token}
    params = {'hostIp': target_ip}
    res = requests.get(url, headers=headers, params=params, verify=False)
    res.raise_for_status()
    hosts = res.json().get('response', [])
    if not hosts:
        print(f'[-] Host {target_ip} not found in DNAC Host DB.')
        return
    h = hosts[0]
    print(f'=== Host Location Info for {target_ip} ===')
    print(f'MAC Address     : {h.get("hostMac")}')
    print(f'Switch Name     : {h.get("connectedNetworkDeviceName")}')
    print(f'Switch IP       : {h.get("connectedNetworkDeviceIpAddress")}')
    print(f'Port            : {h.get("connectedInterfaceName")}')
    print(f'VLAN ID         : {h.get("vlanId")}')
    print(f'Host Type       : {h.get("hostType")}')

if __name__ == '__main__':
    token = get_token()
    locate_host(token, '10.1.100.50')
```

#### 検証方法
* スクリプトを実行し、`10.1.100.50` に該当するMACアドレスおよび `GigabitEthernet1/0/12` などのポート情報が出力されることを確認。

---

### 演習2: 特定MACアドレスの無線/有線クライアント詳細アシュアランス情報取得

#### 要件
* 端末のMACアドレス（例: `00:11:22:33:44:55`）を指定して `GET /dna/intent/api/v1/client-detail` を実行。
* 無線通信の場合はSSID、WLC IP、アソシエーションAP名、RSSI、SNR、Health Scoreを出力。

#### 設定・実装例

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_HOST = '192.168.1.100'
USER_ID = 'admin_user'
USER_SEC = 'dnac_secret'

def get_token():
    url = f'https://{DNAC_HOST}/dna/system/api/v1/auth/token'
    res = requests.post(url, auth=(USER_ID, USER_SEC), verify=False)
    res.raise_for_status()
    return res.json()['Token']

def get_client_detail(token, mac_addr):
    url = f'https://{DNAC_HOST}/dna/intent/api/v1/client-detail'
    headers = {'x-auth-token': token}
    params = {'macAddress': mac_addr}
    res = requests.get(url, headers=headers, params=params, verify=False)
    res.raise_for_status()
    detail = res.json().get('detail', {})
    print(f'=== Client Detail for MAC: {mac_addr} ===')
    print(f'Connection Type : {detail.get("connectionStatus")}')
    print(f'Health Score    : {detail.get("healthScore")}')
    print(f'SSID            : {detail.get("ssid")}')
    print(f'Associated AP   : {detail.get("location")}')
    print(f'VLAN            : {detail.get("vlanId")}')
    print(f'IP Address      : {detail.get("hostIpV4")}')

if __name__ == '__main__':
    token = get_token()
    get_client_detail(token, '00:11:22:33:44:55')
```

#### 検証方法
* クライアント詳細アシュアランスデータが正常に取得され、Health Scoreや接続種別がプリントされることを確認。

---

### 演習3: 特定VLAN IDに所属する全アクティブホストのリストアップ

#### 要件
* 指定VLAN（例: `VLAN 10`）に属するホストを全件抽出。
* 各ホストのIP、MAC、接続スイッチ、接続ポートを一覧表示。

#### 設定・実装例

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_IP = '192.168.1.100'
API_USER = 'admin_user'
API_PASS = 'dnac_secret'

def get_token():
    url = f'https://{DNAC_IP}/dna/system/api/v1/auth/token'
    res = requests.post(url, auth=(API_USER, API_PASS), verify=False)
    return res.json()['Token']

def list_hosts_in_vlan(token, target_vlan):
    url = f'https://{DNAC_IP}/dna/intent/api/v1/host'
    headers = {'x-auth-token': token}
    params = {'vlanId': target_vlan}
    res = requests.get(url, headers=headers, params=params, verify=False)
    hosts = res.json().get('response', [])
    print(f'=== Hosts in VLAN {target_vlan} (Total: {len(hosts)}) ===')
    for h in hosts:
        ip = h.get('hostIp')
        mac = h.get('hostMac')
        sw = h.get('connectedNetworkDeviceName')
        port = h.get('connectedInterfaceName')
        print(f'IP: {ip:<15} | MAC: {mac:<17} | Switch: {sw:<15} | Port: {port}')

if __name__ == '__main__':
    tok = get_token()
    list_hosts_in_vlan(tok, '10')
```

#### 検証方法
* 指定VLANの端末一覧がフォーマットされて出力されることを確認。

---

### 演習4: 通信品質スコア (Health Score <= 3) 劣悪端末の自動検出

#### 要件
* 全クライアント情報を走査し、健康度スコア (Health Score) が 3 以下の劣悪通信端末を抽出。
* 端末MAC、IP、接続デバイス名、接続ポート、スコアを表示。

#### 設定・実装例

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_IP = '192.168.1.100'
API_USER = 'admin_user'
API_PASS = 'dnac_secret'

def get_token():
    url = f'https://{DNAC_IP}/dna/system/api/v1/auth/token'
    res = requests.post(url, auth=(API_USER, API_PASS), verify=False)
    return res.json()['Token']

def audit_unhealthy_hosts(token, threshold=3):
    url = f'https://{DNAC_IP}/dna/intent/api/v1/host'
    headers = {'x-auth-token': token}
    res = requests.get(url, headers=headers, verify=False)
    hosts = res.json().get('response', [])
    print(f'=== Unhealthy Hosts (Health Score <= {threshold}) ===')
    count = 0
    for h in hosts:
        score = h.get('overallHealthScore')
        if score is not None and score <= threshold:
            count += 1
            print(f'[-] Poor Host: IP={h.get("hostIp")}, MAC={h.get("hostMac")}, Score={score}, Switch={h.get("connectedNetworkDeviceName")}, Port={h.get("connectedInterfaceName")}')
    if count == 0:
        print('[+] All hosts have healthy scores.')

if __name__ == '__main__':
    tok = get_token()
    audit_unhealthy_hosts(tok, threshold=3)
```

#### 検証方法
* 閾値以下のホストが検出され、警告ログが出力されることを確認。

---

### 演習5: 有線・無線クライアントの種別（WIRED vs WIRELESS）集計レポート出力

#### 要件
* ネットワーク全体のホストをスキャンし、有線 (`WIRED`) と無線 (`WIRELESS`) の接続台数をカウント。
* それぞれの割合（パーセンテージ）と集計結果をレポート表示。

#### 設定・実装例

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_IP = '192.168.1.100'
API_USER = 'admin_user'
API_PASS = 'dnac_secret'

def get_token():
    url = f'https://{DNAC_IP}/dna/system/api/v1/auth/token'
    res = requests.post(url, auth=(API_USER, API_PASS), verify=False)
    return res.json()['Token']

def summarize_host_types(token):
    url = f'https://{DNAC_IP}/dna/intent/api/v1/host'
    headers = {'x-auth-token': token}
    res = requests.get(url, headers=headers, verify=False)
    hosts = res.json().get('response', [])
    
    wired_count = 0
    wireless_count = 0
    other_count = 0
    
    for h in hosts:
        htype = h.get('hostType')
        if htype == 'WIRED':
            wired_count += 1
        elif htype == 'WIRELESS':
            wireless_count += 1
        else:
            other_count += 1
            
    total = len(hosts)
    print('=== Host Connection Summary ===')
    print(f'Total Connected Hosts : {total}')
    if total > 0:
        print(f'Wired Hosts           : {wired_count} ({wired_count/total*100:.1f}%)')
        print(f'Wireless Hosts        : {wireless_count} ({wireless_count/total*100:.1f}%)')
        print(f'Other/Unknown         : {other_count} ({other_count/total*100:.1f}%)')

if __name__ == '__main__':
    tok = get_token()
    summarize_host_types(tok)
```

#### 検証方法
* 有線・無線の割合が正しく分類・計算されて出力されることを確認。


--- 

### 演習6: ページネーション（`offset` & `limit`）による全ホスト取得とCSVエクスポート

#### 要件
* 大規模環境での制限を回避するため `limit=50` でページネーションループを実施。
* 取得したすべてのホスト情報を `hosts_inventory.csv` ファイルへ出力保存。

#### 設定・実装例

```python
import requests
import csv
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_IP = '192.168.1.100'
API_USER = 'admin_user'
API_PASS = 'dnac_secret'

def get_token():
    url = f'https://{DNAC_IP}/dna/system/api/v1/auth/token'
    res = requests.post(url, auth=(API_USER, API_PASS), verify=False)
    return res.json()['Token']

def export_all_hosts_csv(token):
    url = f'https://{DNAC_IP}/dna/intent/api/v1/host'
    headers = {'x-auth-token': token}
    offset = 1
    limit = 50
    all_hosts = []
    
    while True:
        params = {'offset': offset, 'limit': limit}
        res = requests.get(url, headers=headers, params=params, verify=False)
        hosts = res.json().get('response', [])
        if not hosts:
            break
        all_hosts.extend(hosts)
        if len(hosts) < limit:
            break
        offset += limit
        
    print(f'[+] Total Hosts Retrieved: {len(all_hosts)}')
    
    with open('hosts_inventory.csv', 'w', newline='', encoding='utf-8') as f:
        writer = csv.writer(f)
        writer.writerow(['IP Address', 'MAC Address', 'Type', 'Switch Name', 'Switch IP', 'Port', 'VLAN'])
        for h in all_hosts:
            writer.writerow([
                h.get('hostIp'),
                h.get('hostMac'),
                h.get('hostType'),
                h.get('connectedNetworkDeviceName'),
                h.get('connectedNetworkDeviceIpAddress'),
                h.get('connectedInterfaceName'),
                h.get('vlanId')
            ])
    print('[+] Exported to hosts_inventory.csv successfully.')

if __name__ == '__main__':
    tok = get_token()
    export_all_hosts_csv(tok)
```

#### 検証方法
* スクリプト実行後、`hosts_inventory.csv` が作成され全ホストが記載されていることを確認。

---

### 演習7: 複数IPリストを入力とする物理スイッチ・ポート一括解決スクリプト

#### 要件
* 対象IPリスト（例: `['10.1.10.10', '10.1.10.20', '10.1.20.30']`）を受領。
* 各IPアドレスに対してホストAPIを実行し、接続スイッチとポートのマッピングテーブルを表示。

#### 設定・実装例

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_IP = '192.168.1.100'
API_USER = 'admin_user'
API_PASS = 'dnac_secret'

def get_token():
    url = f'https://{DNAC_IP}/dna/system/api/v1/auth/token'
    res = requests.post(url, auth=(API_USER, API_PASS), verify=False)
    return res.json()['Token']

def bulk_resolve_hosts(token, ip_list):
    url = f'https://{DNAC_IP}/dna/intent/api/v1/host'
    headers = {'x-auth-token': token}
    print('=== Bulk Host Location Resolution ===')
    print(f'{"Target IP":<15} | {"Switch Name":<15} | {"Port":<15} | {"VLAN":<5}')
    print('-' * 60)
    for ip in ip_list:
        params = {'hostIp': ip}
        res = requests.get(url, headers=headers, params=params, verify=False)
        hosts = res.json().get('response', [])
        if hosts:
            h = hosts[0]
            sw = h.get('connectedNetworkDeviceName', 'Unknown')
            port = h.get('connectedInterfaceName', 'N/A')
            vlan = h.get('vlanId', 'N/A')
            print(f'{ip:<15} | {sw:<15} | {port:<15} | {vlan:<5}')
        else:
            print(f'{ip:<15} | {"NOT FOUND":<15} | {"N/A":<15} | {"N/A":<5}')

if __name__ == '__main__':
    tok = get_token()
    targets = ['10.1.10.10', '10.1.10.20', '10.1.20.30']
    bulk_resolve_hosts(tok, targets)
```

#### 検証方法
* 指定した複数のIPに対する接続ポート一覧が表形式で出力されることを確認。

---

### 演習8: MACアドレスフォーマット自動正規化関数付きクライアント詳細検索

#### 要件
* 入力フォーマットが不定（例: `0011.2233.4455` または `00-11-22-33-44-55`）なMACアドレスを自動でコロン区切り (`00:11:22:33:44:55`) に正規化。
* 正規化後に `/client-detail` API を安全に実行。

#### 設定・実装例

```python
import requests
import re
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_IP = '192.168.1.100'
API_USER = 'admin_user'
API_PASS = 'dnac_secret'

def normalize_mac(mac_str):
    clean = re.sub(r'[^a-fA-F0-9]', '', mac_str)
    if len(clean) != 12:
        raise ValueError(f'Invalid MAC string: {mac_str}')
    return ':'.join([clean[i:i+2] for i in range(0, 12, 2)]).lower()

def get_token():
    url = f'https://{DNAC_IP}/dna/system/api/v1/auth/token'
    res = requests.post(url, auth=(API_USER, API_PASS), verify=False)
    return res.json()['Token']

def query_client_normalized(token, raw_mac):
    norm_mac = normalize_mac(raw_mac)
    print(f'[+] Normalized MAC: {raw_mac} -> {norm_mac}')
    url = f'https://{DNAC_IP}/dna/intent/api/v1/client-detail'
    headers = {'x-auth-token': token}
    params = {'macAddress': norm_mac}
    res = requests.get(url, headers=headers, params=params, verify=False)
    print(f'[+] API Response Status: {res.status_code}')

if __name__ == '__main__':
    tok = get_token()
    query_client_normalized(tok, '0011.2233.4455')
```

#### 検証方法
* `0011.2233.4455` が `00:11:22:33:44:55` に変換されAPIリクエストが正常完了することを確認。

---

### 演習9: 特定アクセススイッチ (Switch IP) 配下に接続された全端末の自動抽出

#### 要件
* アクセススイッチIP（例: `192.168.10.1`）を指定。
* DNAC Host API から `connectedNetworkDeviceIpAddress` が合致する全端末をフィルタ抽出。

#### 設定・実装例

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_IP = '192.168.1.100'
API_USER = 'admin_user'
API_PASS = 'dnac_secret'

def get_token():
    url = f'https://{DNAC_IP}/dna/system/api/v1/auth/token'
    res = requests.post(url, auth=(API_USER, API_PASS), verify=False)
    return res.json()['Token']

def get_hosts_by_switch(token, switch_ip):
    url = f'https://{DNAC_IP}/dna/intent/api/v1/host'
    headers = {'x-auth-token': token}
    res = requests.get(url, headers=headers, verify=False)
    all_hosts = res.json().get('response', [])
    
    switch_hosts = [h for h in all_hosts if h.get('connectedNetworkDeviceIpAddress') == switch_ip]
    print(f'=== Hosts Connected to Switch ({switch_ip}) : Total {len(switch_hosts)} ===')
    for h in switch_hosts:
        print(f'Port: {h.get("connectedInterfaceName"):<15} | IP: {h.get("hostIp"):<15} | MAC: {h.get("hostMac")}')

if __name__ == '__main__':
    tok = get_token()
    get_hosts_by_switch(tok, '192.168.10.1')
```

#### 検証方法
* 対象スイッチIPに直結されている端末のポート番号一覧が出力されることを確認。

---

### 演習10: 不正端末の追跡と隔離用スイッチ・ポート情報の安全ログ出力（SecOps連携）

#### 要件
* 侵害された不正IPアドレス（例: `10.1.50.99`）を受領。
* 接続スイッチIPおよびインターフェース名を取得し、隔離自動化スクリプト用に `quarantine_targets.json` へ出力。

#### 設定・実装例

```python
import requests
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DNAC_IP = '192.168.1.100'
API_USER = 'admin_user'
API_PASS = 'dnac_secret'

def get_token():
    url = f'https://{DNAC_IP}/dna/system/api/v1/auth/token'
    res = requests.post(url, auth=(API_USER, API_PASS), verify=False)
    return res.json()['Token']

def prepare_quarantine_target(token, rogue_ip):
    url = f'https://{DNAC_IP}/dna/intent/api/v1/host'
    headers = {'x-auth-token': token}
    params = {'hostIp': rogue_ip}
    res = requests.get(url, headers=headers, params=params, verify=False)
    hosts = res.json().get('response', [])
    if not hosts:
        print(f'[-] Rogue IP {rogue_ip} not found.')
        return
    h = hosts[0]
    target_payload = {
        'rogue_ip': rogue_ip,
        'rogue_mac': h.get('hostMac'),
        'switch_ip': h.get('connectedNetworkDeviceIpAddress'),
        'switch_name': h.get('connectedNetworkDeviceName'),
        'port_name': h.get('connectedInterfaceName'),
        'vlan_id': h.get('vlanId'),
        'action_required': 'QUARANTINE_PORT'
    }
    with open('quarantine_targets.json', 'w', encoding='utf-8') as f:
        json.dump(target_payload, f, indent=4)
    print(f'[+] Security Action Handoff File Created for {rogue_ip}:')
    print(json.dumps(target_payload, indent=2))

if __name__ == '__main__':
    tok = get_token()
    prepare_quarantine_target(tok, '10.1.50.99')
```

#### 検証方法
* `quarantine_targets.json` が生成され、スイッチIPとポート名が正確にパースされていることを確認。


## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験および記述問題を想定したクイズと解説です。

### 質問1 (コンフィグ・コード読解)
**問題**: 次の Python スクリプトを実行したところ、IndexError が発生してスクリプトが停止しました。原因と修正方法を説明しなさい。

```python
def get_switch_port(token, target_ip):
    url = f'https://{DNAC_IP}/dna/intent/api/v1/host?hostIp={target_ip}'
    headers = {'x-auth-token': token}
    res = requests.get(url, headers=headers).json()
    return res['response'][0]['connectedInterfaceName']
```

**解答と解説**:
* **原因**: 指定した target_ip が Host DB 内に存在しない場合、`response` フィールドは空リスト `[]` となります。
* **修正方法**: `res.get('response')` の戻り値が存在するかチェックするガードブロックを追加します。

---

## 🔗 参考リソース

* Cisco DNA Center Intent API Reference Guide
* Cisco Catalyst Center REST API Platform Guide
* Cisco DevNet Learning Labs: DNA Center Intent APIs
* Cisco Live BRKDEV-2011
* Cisco Live BRKSEC-3032


---

## 📝 補足（Notes）

* **トークン管理の鉄則**: DNAC API の `x-auth-token` は全リクエストのヘッダーにセットする必要があります。
* **MACアドレス表記**: `GET /client-detail` 実行時はコロン区切り表記が安全です。


