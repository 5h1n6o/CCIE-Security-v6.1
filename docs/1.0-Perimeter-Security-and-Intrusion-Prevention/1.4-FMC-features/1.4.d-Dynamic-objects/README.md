---
layout: default
title: 1.4.d-Dynamic-objects
nav_order: 4
parent: 1.4-FMC-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.4.d Dynamic objects

Cisco Secure Firewall Management Center (FMC) における **Dynamic Objects（動的オブジェクト）** は、クラウドアプライアンスや外部 Threat Intelligence、CI/CD パイプライン、ならびに各種オーケストレーションツール（Cisco Secure Dynamic Attributes Connector 等）と連携し、FMC から FTD (Secure Firewall Threat Defense) への**ポリシー再展開（Policy Deployment）を実行することなく、IP アドレス / CIDR マッピングをリアルタイムかつ動的に更新・適用**するための高度なオブジェクト機能です。

CCIE Security v6.1 Blueprint 1.4.d では、標準 Network Object との動作原理の違い、REST API (`/api/fmc_config/v1/domain/{domainUUID}/object/dynamicobjects`) を使用した動的マッピングの追加・更新・削除、Cisco Secure Dynamic Attributes Connector (旧 Cloud Dynamic Attributes Connector) との連携アーキテクチャ、FTD データプレーンへの即時反映メカニズム、制限事項、ならびに自動化スクリプトを用いたトラブルシューティング能力が厳格に評価されます。

---

## 📘 概要

* **機能概要**:
  * **デプロイフリーな IP マッピング更新**: オブジェクト定義自体を一度 Access Control Policy (ACP) や Prefilter Policy 等に組み込んで FTD へ Deploy しておけば、オブジェクト内に含まれる IP アドレス/CIDR のリスト変更（追加・削除）時に FMC の Deploy 操作（Policy Deployment）を必要とせず、FTD 側のメモリテーブルが即時（数秒以内）に更新される機能。
  * **外部システム・クラウドとの動的連携**: AWS、Azure、GCP、VMware vCenter、NSX-T などのマルチクラウド環境で動的にスケールイン/アウトする仮想マシン（VM）の IP アドレス変化や、SOC / SOAR からの脅威 IP 遮断指示を REST API 経由でリアルタイムに FTD に反映。
  * **Cisco Secure Dynamic Attributes Connector 連携**: AWS EC2 タグ、Azure リソースグループ、vCenter カスタム属性などのメタデータ（Tag）を監視し、条件に合致する IP アドレス群を FMC の Dynamic Object へ自動生成・同期。
* **利用目的**:
  * パブリッククラウドリソース（オートスケーリング群）の動的な IP 変化に伴うセキュリティポリシー適用の完全自動化。
  * 脅威検知システム（SIEM / SOAR / CTI）と連携した、悪意ある C2 IP アドレスの即時ブロック（1 秒を争う自動防御）。
  * 頻繁な IP 変更に伴う FMC の Deploy 処理負荷（Snort リロードや CPU パイプライン消費）の低減および運用コストの削減。
* **どのような場面で利用するか**:
  * クラウドネイティブ環境（Kubernetes、AWS EC2、Azure VM）とオンプレミス FTD 境界セキュリティの動的連携。
  * 高速なインシデントレスポンス（Rapid Automated Containment）における自動 IP ブラックリスト構成。
  * ポリシー展開ウィンドウ（メンテナンス時間）を待たずに即座にセキュリティアクセス制御を変更したい環境。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **役割** | Deploy 不要で IP アドレスマッピングをリアルタイム更新する動的オブジェクト |
| **設定メニュー位置** | <code>Objects > Object Management > External Attributes > Dynamic Objects</code> |
| **サポート対象形式** | **IPv4 / IPv6 アドレス、CIDR サブネット** (例: `192.168.10.50`, `10.20.0.0/16`) |
| **非サポート形式** | FQDN、IP アドレス範囲（Range: `10.1.1.1-10.1.1.10`）、ポート、プロトコル、MAC アドレス |
| **更新トリガー** | FMC REST API (POST/PUT/DELETE) または Cisco Secure Dynamic Attributes Connector |
| **Deploy の必要性** | **新規作成・ACP への割り当て時は Deploy 必須**。オブジェクト作成後の IP マッピング追加/更新は **Deploy 不要** |
| **対応ポリシー** | Access Control Policy (ACP Rules, Default Action), Prefilter Policy, NAT Policy 等 |
| **制限事項** | Dynamic Object 内部で Object Group や Object Overrides は使用不可 |

---

## 🏗 動作原理

Dynamic Objects は、FMC の静的データベース更新モデルとは異なり、REST API または Connector 経由で受信した IP マッピング情報を、`sftunnel` 管理チャネルを通じて FTD アプライアンスの動的オブジェクトキャッシュテーブルに直接書き込みます。

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│                      Firepower Management Center (FMC)                          │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │ [ Objects > Object Management > External Attributes > Dynamic Objects ]    │  │
│  │  - Object Name: Dynamic_Web_Servers                                        │  │
│  │  - Initial Definition Deployed to FTD via ACP Rule                         │  │
│  └────────────────────────────────────▲───────────────────────────────────────┘  │
│                                       │                                          │
│                                       │ REST API Request (POST/PUT/DELETE)       │
│                                       │ /api/fmc_config/v1/domain/{uuid}/       │
│                                       │  object/dynamicobjects/{id}/mappings     │
│                                       │                                          │
│                    ┌──────────────────┴──────────────────┐                       │
│                    │ External Orchestrator / API Client │                       │
│                    │ (Python / Postman / Secure Dynamic │                       │
│                    │  Attributes Connector)             │                       │
│                    └─────────────────────────────────────┘                       │
│                                       │                                          │
│                                       │ Real-time Mapping Push (No Deploy Required)│
│                                       ▼                                          │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │ FMC SFDataManager / Dynamic Object Process                                 │  │
│  └────────────────────────────────────┬───────────────────────────────────────┘  │
│                                       │                                          │
│                                       │ sftunnel Dynamic Update Message          │
│                                       ▼                                          │
└───────────────────────────────────────┼──────────────────────────────────────────┘
                                        │
                                        ▼ (TCP 8305: sftunnel)
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    Firepower Threat Defense (FTD) Data Plane                     │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │ [ Dynamic Object Cache / Memory Table ]                                    │  │
│  │   Dynamic_Web_Servers = { 10.1.1.50, 10.1.1.51, 10.2.0.0/24 }               │  │
│  └────────────────────────────────────┬───────────────────────────────────────┘  │
│                                       │ Immediate Inspection Table Lookup        │
│                                       ▼                                          │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │ LINA ACL & Snort 3 / Snort 2 Enforcement Engine                            │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

Dynamic Objects の作成から、リアルタイム IP マッピング追加・評価までの詳細シーケンスは以下の通りです。

1. **初期定義と初期 Deploy (初回のみ)**:
   * 管理者は FMC UI (`Objects > Object Management > External Attributes > Dynamic Objects`) または REST API にて空の Dynamic Object (例: `Obj_Block_Malware`) を作成。
   * ACP (Access Control Policy) のルール条件（Source/Destination Network）に `Obj_Block_Malware` を指定し、FTD アプライアンスへ **1 回目の Policy Deploy** を実行。
   * FTD 側に Dynamic Object 用の ACL 構造が構築される。
2. **REST API による IP マッピングの要求**:
   * 外部システム（Python スクリプト、SOAR、クラウドコネクタ）が FMC の REST API ポート (TCP 443) 宛てに認証トークンを取得後、マッピング追加 POST リクエストを送出。
   * エンドポイント: `/api/fmc_config/v1/domain/{domainUUID}/object/dynamicobjects/{objectId}/mappings`
   * ペイロード: `{"add": ["203.0.113.50", "198.51.100.0/24"]}`
3. **FMC から FTD への即時ブロードキャスト**:
   * FMC は受信した IP リストを自身の DB に保持すると同時に、登録されている全 FTD アプライアンス（または該当ポリシーがアサインされた FTD）へ `sftunnel` (TCP 8305) を通じて同期メッセージを即時伝送。
4. **FTD メモリテーブルの動的更新**:
   * FTD は受信した IP アドレス/CIDR をデータプレーンの Dynamic Object メモリテーブルへ直接挿入。**Snort プロセスのリロードや LINA の再コンパイルは発生しない**。
5. **パケットインスペクションへの反映**:
   * 次の通信パケットが FTD を通過する際、更新された Dynamic Object メモリテーブルと照合され、設定されたアクション (Allow/Block) が即座に適用される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **初回の Deploy 忘れ問題**:
   * Dynamic Object は「IP マッピングの追加/削除」には Deploy が不要ですが、**Dynamic Object 自身の新規作成、名前変更、または ACP ルールへのアサイン時には、必ず 1 回だけ FTD への Policy Deploy が必要** です。
   * 試験で「Create a dynamic object via API and apply it to ACP, then populate IPs without deployment」と問われた場合、オブジェクト作成および ACP アサイン直後の初回 Deploy を怠ると、API で IP を追加しても FTD 側で評価されません。
2. **REST API エンドポイントの正確な把握**:
   * ラボ試験で Python スクリプトの作成や Postman / cURL による操作が要求されるため、正確な URL パスと HTTP メソッドを記憶または API Explorer で迅速に特定できる必要があります。
   * Token 取得: `POST https://<FMC_IP>/api/fmc_platform/v1/auth/generatetoken`
   * Dynamic Object 作成: `POST https://<FMC_IP>/api/fmc_config/v1/domain/{domainUUID}/object/dynamicobjects`
   * Dynamic Mapping 更新: `POST https://<FMC_IP>/api/fmc_config/v1/domain/{domainUUID}/object/dynamicobjects/{objID}/mappings`
3. **IP アドレス形式の制限判定**:
   * Dynamic Object に投入できるのは **IP アドレス (IPv4/IPv6) または CIDR サブネットのみ** です。
   * **NG 例**: `10.1.1.1-10.1.1.20`（Range 形式はエラー）、`www.cisco.com`（FQDN は非サポート）。これらを API ペイロードに含めると `400 Bad Request` が返却されます。
4. **Cisco Secure Dynamic Attributes Connector との連携**:
   * FMC と AWS/Azure/vCenter の中間に位置する Connector アプライアンスの役割と、FMC 側で定義された Dynamic Object 名との一致確認。
5. **Object Groups や Overrides の非サポート**:
   * Dynamic Object は Object Group のメンバーにすることができず、Device Overrides や Domain Overrides もサポートされません。

---

## 🛠 設定方法

### 1. FMC GUI: Dynamic Object の作成手順

1. **`Objects > Object Management`** に移動。
2. 左パネルから **`External Attributes > Dynamic Objects`** を選択。
3. **`Add Dynamic Object`** をクリック。
   * **Name**: `Dyn_C2_Blacklist`
   * **Description**: `Dynamic IP Blacklist via REST API`
   * **Type**: `IP`
4. **`Save`** をクリック。
5. **`Policies > Access Control`** を開き、目的のルール（例: `BLOCK_MALICIOUS_IP`）の Destination Network に `Dyn_C2_Blacklist` をアサインして保存。
6. 右上の **`Deploy`** をクリックし、対象 FTD へ初回デプロイを実行。

---

### 2. REST API: Python スクリプトによる Dynamic Object マッピングの動的追加

外部から REST API を呼び出して IP アドレスを追加・削除する実装コード例です。

```python
import requests
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

FMC_IP = "192.168.10.50"
USERNAME = "admin"
PASSWORD = "Cisco12345Password"

# 1. 認証トークンの取得
auth_url = f"https://{FMC_IP}/api/fmc_platform/v1/auth/generatetoken"
headers = {'Content-Type': 'application/json'}

response = requests.post(auth_url, auth=(USERNAME, PASSWORD), headers=headers, verify=False)
if response.status_code != 204:
    print(f"Auth Failed: {response.status_code}")
    exit()

auth_token = response.headers.get('X-auth-access-token')
domain_uuid = response.headers.get('DOMAIN_UUID')

# 2. Dynamic Object の UUID 取得 (名前から検索)
get_obj_url = f"https://{FMC_IP}/api/fmc_config/v1/domain/{domain_uuid}/object/dynamicobjects?filter=name:Dyn_C2_Blacklist"
req_headers = {
    'X-auth-access-token': auth_token,
    'Content-Type': 'application/json'
}

res_obj = requests.get(get_obj_url, headers=req_headers, verify=False)
obj_data = res_obj.json()
object_uuid = obj_data['items']['id']

# 3. Dynamic Object マッピングの追加 (Deploy 不要で FTD に即時適用)
mapping_url = f"https://{FMC_IP}/api/fmc_config/v1/domain/{domain_uuid}/object/dynamicobjects/{object_uuid}/mappings"

mapping_payload = {
    "add": [
        "203.0.113.100",
        "198.51.100.0/24"
    ],
    "remove": [
        "192.0.2.50"
    ]
}

res_map = requests.post(mapping_url, headers=req_headers, data=json.dumps(mapping_payload), verify=False)

if res_map.status_code in:
    print("Dynamic Object Mappings Updated Successfully!")
    print(json.dumps(res_map.json(), indent=2))
else:
    print(f"Failed to update mappings: {res_map.status_code} - {res_map.text}")
```

---

### 3. cURL コマンドによる Dynamic Mapping の手動更新

```bash
# ===== Step 1: Auth Token の生成 =====
curl -k -X POST [suspicious link removed] \
  -u "admin:Cisco12345Password" \
  -H "Content-Type: application/json" -i

# レスポンスヘッダーから X-auth-access-token と DOMAIN_UUID を取得
# Token: eyJhbGciOiJSUzI1Ni...
# Domain UUID: e276ab2c-103f-11e7-8b09-f81d11394f4c

# ===== Step 2: Dynamic Object への IP マッピング追加 (POST) =====
curl -k -X POST "[suspicious link removed]" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json" \
  -d '{
        "add": [
          "10.250.10.15",
          "172.16.100.0/24"
        ]
      }'
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI / FMC API / Bash) | 表示確認ポイント |
| :--- | :--- | :--- |
| **FTD データプレーン上の Dynamic Object 保持 IP 一覧確認** | `show dynamic-objects` / `show dynamic-object name <Name>` | FTD メモリ内にリアルタイム同期されている IP/CIDR リスト |
| **FTD 側の LINA アクセスリスト上の Dynamic Object 参照確認** | `show access-list | include dynamic` | LINA ACL エントリにおける Dynamic Object 名のバインド状態 |
| **FMC API 経由での Dynamic Object 一覧取得** | `GET /api/fmc_config/v1/domain/{uuid}/object/dynamicobjects` | 作成済み Dynamic Object の Name, UUID, Type 一覧 |
| **FMC API 経由での特定の Dynamic Object マッピング状態取得** | `GET /api/fmc_config/v1/domain/{uuid}/object/dynamicobjects/{id}/mappings` | FMC DB に登録されている最新の IP アドレス/CIDR 配列 |
| **sftunnel 経由での Dynamic Object 同期メッセージ追跡 (FMC Bash)** | `sudo tail -f /var/log/messages | grep -i dynamic` | FMC から FTD への Dynamic Object 更新通知ログ |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **API で IP を追加したが FTD 側で通信が遮断/許可されない** | 初回の Dynamic Object 作成後、ACP を FTD へ Deploy していない | FTD CLI: `show dynamic-objects`<br>FMC UI: `Deploy Status` | オブジェクトを ACP にアサインした状態で、初回のみ FTD へ Policy Deploy を実行する。 |
| **API からマッピング追加時に `400 Bad Request` エラーが発生する** | IP アドレス以外の不適切な形式（Range `10.1.1.1-5` や FQDN `example.com`）を送信している | API レスポンスボディの `error` メッセージ確認 | リクエストの JSON ペイロードを修正し、IPv4/IPv6 または CIDR 表記のみにする。 |
| **FMC 上ではマッピングが更新されているが FTD 側で反映されない** | FMC ↔ FTD 間の `sftunnel` 通信障害、または FTD 側の同期デーモン停止 | FTD CLI: `show manager`<br>FTD CLI: `sftunnel-status-brief` | TCP 8305 ポートの疎通を確認し、`sftunnel` 接続状態を復旧させる。 |
| **API 呼び出し時に `401 Unauthorized` または `403 Forbidden` が返る** | 取得した `X-auth-access-token` が有効期限切れ（デフォルト 30 分 / 最大 8 時間）、またはトークン未指定 | REST API ヘッダー確認 | `/api/fmc_platform/v1/auth/generatetoken` を再呼び出ししてトークンを再取得・リフレッシュする。 |
| **API の呼び出しが `429 Too Many Requests` で失敗する** | 短時間に大量の API リクエストを送信し、FMC の API Rate Limit に到達した | FMC `System > Configuration > REST API Preferences` | リクエスト送信間隔にウェイト（sleep）を入れるか、複数 IP を 1 回の API リクエストにまとめて送信する。 |

---

## ⚠ 制限事項

* **設定値の形式制限**:
  * 入力可能なデータタイプは **IPv4 アドレス、IPv6 アドレス、および CIDR ネットワーク** のみです。
  * FQDN、IP Range (ハイフン区切り)、ポート番号、VLAN タグは指定できません。
* **ネストおよびグループ化の不可**:
  * Dynamic Object を Network Object Group のメンバーとして追加（ネスト）することはできません。
* **1 リクエストあたりの処理件数上限**:
  * REST API の 1 回の POST リクエストで追加/削除できる IP アドレスの上限数は FMC のバージョンおよびモデルに依存します（通常 1 回のリクエストにつき最大 1,000 件程度を推奨）。
* **ライセンスおよび接続要件**:
  * Cisco Secure Dynamic Attributes Connector を利用する場合は、FMC REST API が有効化（`System > Configuration > REST API Preferences` で Enabled）されている必要があります。

---

## 🔄 他技術との関連

* **REST API (`/api/fmc_config/v1/...`)**: Dynamic Object のマッピングを動的にコントロールするための主要な管理インターフェイス。
* **Cisco Secure Dynamic Attributes Connector**: パブリッククラウド（AWS/Azure/GCP）や vCenter からインスタンスのタグ/属性情報を自動収集し、FMC の Dynamic Object へマッピングを自動同期する標準ミドルウェア。
* **Access Control Policy (ACP)**: Dynamic Object をルール条件（Source/Destination Networks）に配置して実際のトラフィック制御を実施。
* **Security Intelligence (SI) Feeds との比較**: SI Feed は外部 HTTP/HTTPS サーバーから URI 経由でファイルをダウンロードして IP/URL ブラックリストを更新するのに対し、Dynamic Object は REST API や Connector からの直接プッシュ（Push）型更新である点が異なります。

---

## 🧩 比較表

### 1. Dynamic Objects vs Standard Network Objects vs Security Intelligence (SI) Feeds

| 比較項目 | Dynamic Objects | Standard Network Objects | Security Intelligence Feeds |
| :--- | :--- | :--- | :--- |
| **更新時の Deploy** | **完全不要** (即時反映) | **必須** (Policy Deploy が必要) | **不要** (指定間隔で自動フェッチ) |
| **更新メカニズム** | REST API / Connector から Push | FMC GUI / API で作成後 Deploy | 外部 Web サーバーから Dynamic Fetch (Pull) |
| **データ形式** | IP / CIDR のみ | Host, Subnet, Range, FQDN | IP / Domain / URL リストファイル |
| **反映速度** | **数秒以内 (リアルタイム)** | 分単位 (Deploy 完了まで) | 分〜時間単位 (更新タイマー依存) |
| **適用場所** | ACP Rules, Prefilter, NAT | 全ポリシー・全ルール | ACP SI タブ, DNS Policy |
| **主な用途** | クラウドオートスケーリング, SOAR 自動遮断 | 静的インフラ、固定サーバー群 | 外部脅威インテリジェンスブラックリスト |

---

## 💡 ベストプラクティス

1. **バッチ処理による API リクエストの集約**:
   * 大量の IP アドレスを追加・削除する際、1 IP ごとに API リクエストを送信すると FMC の API レート制限や処理オーバーヘッドが発生します。JSON ペイロードの `add` / `remove` 配列に複数 IP（数千件単位）をまとめて 1 回の POST で送信してください。
2. **初回 Deploy の標準運用手順化**:
   * Dynamic Object を新規作成して ACP にアサインした際は、外部システムからの API 連携を開始する前に、必ず手動で **Policy Deploy を完了させておく** 運用フローを徹底します。
3. **API トークンのリフレッシュ管理**:
   * 長時間稼働する自動化スクリプトでは、`X-auth-access-token` の有効期限（デフォルト 30 分）を考慮し、定期的に `/api/fmc_platform/v1/auth/refreshtoken` を呼び出してトークンを更新します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: FMC UI での Dynamic Object の作成および ACP ルールへの割り当て

#### 問題
FMC UI 上で動的ブラックリスト用 Dynamic Object `Dyn_Blocked_IPs` を作成し、Access Control Policy `DMZ_POLICY` の最上位ルール `BLOCK_DYNAMIC_MALICIOUS` の Source Network に指定して初回 Deploy を完了させよ。

#### 要件
* オブジェクト名: `Dyn_Blocked_IPs`
* アクション: `Block with reset`

#### 設定・操作手順

```text
1. Objects > Object Management > External Attributes > Dynamic Objects に移動。
2. "Add Dynamic Object" をクリック。
   - Name: Dyn_Blocked_IPs
   - Type: IP
   - Save をクリック。
3. Policies > Access Control > "DMZ_POLICY" を編集。
4. "Add Rule" を追加:
   - Name: BLOCK_DYNAMIC_MALICIOUS
   - Action: Block with reset
   - Source Networks: External Attributes > Dynamic Objects 内の "Dyn_Blocked_IPs" を選択して追加。
   - Save をクリック。
5. 右上の Deploy をクリックし、対象 FTD アプライアンスにポリシーを展開。
```

#### 検証方法
* FTD CLI で `show dynamic-objects` を実行し、`Dyn_Blocked_IPs` オブジェクトが存在し、内部 IP アドレスが空の状態で登録されていることを確認。

---

### 例2: REST API を使用した Dynamic Object への単一 IP アドレスの追加

#### 問題
REST API を使用して、既存の Dynamic Object `Dyn_Blocked_IPs` に対し、悪意ある IP アドレス `203.0.113.199` を即時追加せよ。

#### 要件
* FTD への Policy Deploy を実行しないこと。

#### 設定・操作手順

```bash
# ===== 1. 認証トークンおよび Object UUID の取得 =====
# Auth Token 取得後、UUID 検索
curl -k -X GET "[suspicious link removed]" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..."

# ===== 2. IP アドレスの追加 POST =====
curl -k -X POST "[suspicious link removed]" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json" \
  -d '{
        "add": [
          "203.0.113.199"
        ]
      }'
```

#### 検証方法
* FTD CLI で `show dynamic-objects` を実行し、`203.0.113.199` が Deploy なしで即時に一覧へ表示されることを確認。

---

### 例3: REST API を使用した Dynamic Object からの IP アドレスの削除

#### 問題
Dynamic Object `Dyn_Blocked_IPs` から、誤検知と判定された IP アドレス `203.0.113.199` を REST API で削除せよ。

#### 要件
* JSON ペイロードで `"remove"` を使用すること。

#### 設定・操作手順

```bash
curl -k -X POST "[suspicious link removed]" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json" \
  -d '{
        "remove": [
          "203.0.113.199"
        ]
      }'
```

#### 検証方法
* FTD CLI で `show dynamic-objects` を実行し、`203.0.113.199` が消去されていることを確認。

---

### 例4: CIDR サブネットのバッチ追加設定

#### 問題
クラウド上の新規サブネット群 (`10.50.0.0/16` および `10.60.0.0/16`) を、1 回の REST API リクエストで Dynamic Object `Dyn_Cloud_Networks` に一括追加せよ。

#### 要件
* JSON ペイロードで配列として複数 CIDR を指定すること。

#### 設定・操作手順

```bash
curl -k -X POST "[suspicious link removed]" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json" \
  -d '{
        "add": [
          "10.50.0.0/16",
          "10.60.0.0/16"
        ]
      }'
```

#### 検証方法
* FTD CLI で `show dynamic-object name Dyn_Cloud_Networks` を実行し、両サブネットが登録されていることを確認。

---

### 例5: Dynamic Object 内の全マッピングの置換 (PUT リクエスト)

#### 問題
Dynamic Object `Dyn_Allowed_DevOps` 内に現在登録されているすべての IP アドレスをクリアし、指定した新しい IP アドレス群 (`172.16.50.10`, `172.16.50.11`) のみで全置換（Overwrite）せよ。

#### 要件
* HTTP メソッド `PUT` または `mappings` の全置換 API を使用すること。

#### 設定・操作手順

```bash
curl -k -X PUT "[suspicious link removed]" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json" \
  -d '{
        "mappings": [
          "172.16.50.10",
          "172.16.50.11"
        ]
      }'
```

#### 検証方法
* FTD CLI で `show dynamic-objects` を実行し、旧 IP アドレス群が消去され、`172.16.50.10` と `172.16.50.11` のみが保持されていることを確認。

---

### 例6: Python による API トークン自動リフレッシュ付き Dynamic Object 更新スクリプト

#### 問題
FMC の認証トークンの有効期限切れを考慮し、トークン自動取得・リフレッシュ機能を含んだ Python スクリプトを作成して Dynamic Object `Dyn_Automation_IPs` に IP `192.0.2.100` を追加せよ。

#### 要件
* Python `requests` ライブラリを使用すること。

#### 設定・操作手順

```python
import requests
import json

FMC_IP = "192.168.10.50"
URL_BASE = f"https://{FMC_IP}/api"

def get_auth_token():
    url = f"{URL_BASE}/fmc_platform/v1/auth/generatetoken"
    res = requests.post(url, auth=('admin', 'Cisco12345Password'), verify=False)
    return res.headers['X-auth-access-token'], res.headers['DOMAIN_UUID']

token, domain_uuid = get_auth_token()

headers = {
    'X-auth-access-token': token,
    'Content-Type': 'application/json'
}

# Dynamic Object の取得
obj_url = f"{URL_BASE}/fmc_config/v1/domain/{domain_uuid}/object/dynamicobjects?filter=name:Dyn_Automation_IPs"
obj_res = requests.get(obj_url, headers=headers, verify=False).json()
obj_id = obj_res['items']['id']

# マッピングの追加
map_url = f"{URL_BASE}/fmc_config/v1/domain/{domain_uuid}/object/dynamicobjects/{obj_id}/mappings"
payload = {"add": ["192.0.2.100"]}

response = requests.post(map_url, headers=headers, data=json.dumps(payload), verify=False)
print("Status Code:", response.status_code)
print("Response:", response.json())
```

#### 検証方法
* スクリプトを実行し、`Status Code: 200` または `201` が返されることを確認。

---

### 例7: Cisco Secure Dynamic Attributes Connector 用の FMC REST API 有効化

#### 問題
外部の Cisco Secure Dynamic Attributes Connector アプライアンスが FMC と通信できるよう、FMC 上で REST API 機能を有効化せよ。

#### 要件
* `System > Configuration > REST API Preferences` を使用すること。

#### 設定・操作手順

```text
1. FMC UI 画面右上の Cog アイコン (システム設定) をクリック。
2. "REST API Preferences" を選択。
3. "Enable REST API" チェックボックスをオンにする。
4. "Save" をクリック。
5. Web ブラウザで [suspicious link removed] にアクセスし、API Explorer が開くことを確認。
```

#### 検証方法
* ブラウザまたは cURL から `/api/fmc_platform/v1/auth/generatetoken` への POST リクエストに対して HTTP 204 が返ることを確認。

---

### 例8: Prefilter Policy 内での Dynamic Object による Fastpath 処理

#### 問題
大容量データ転送を行う特定のクラウドストレージ IP アドレス群をリアルタイムに集約する Dynamic Object `Dyn_Fastpath_Cloud` を作成し、Prefilter Policy で最前線で Fastpath（Snort バイパス）処理せよ。

#### 要件
* `Policies > Access Control > Prefilter` 内で設定すること。

#### 設定・操作手順

```text
1. Objects > Object Management > External Attributes > Dynamic Objects で "Dyn_Fastpath_Cloud" (Type: IP) を作成。
2. Policies > Access Control > Prefilter を開き、対象 Prefilter Policy を編集。
3. "Add Prefilter Rule" を追加:
   - Name: FASTPATH_DYNAMIC_CLOUD
   - Action: Fastpath
   - Destination IP: Dynamic Objects > "Dyn_Fastpath_Cloud" を指定。
4. 設定を保存し、ACP の Advanced タブでバインドして対象 FTD へ初回 Deploy。
```

#### 検証方法
* API 経由で `Dyn_Fastpath_Cloud` に Cloud IP `198.51.100.50` を追加。
* FTD CLI で `show access-list | include PREFILTER` を実行し、該当 IP 宛ての Fastpath 処理が有効になっていることを確認。

---

### 例9: IPv6 アドレスを含む Dynamic Object の構成

#### 問題
IPv6 トラフィック制限用として、Dynamic Object `Dyn_IPv6_Blocked` を作成し、REST API から IPv6 CIDR `2001:db8:ffff::/48` を追加せよ。

#### 要件
* IPv6 アドレスを正しく解釈させること。

#### 設定・操作手順

```bash
curl -k -X POST "[suspicious link removed]" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json" \
  -d '{
        "add": [
          "2001:db8:ffff::/48"
        ]
      }'
```

#### 検証方法
* FTD CLI で `show dynamic-objects` を実行し、`2001:db8:ffff::/48` が動的オブジェクトテーブルに含まれていることを確認。

---

### 例10: FTD CLI を用いた Dynamic Object トラブルシューティングおよびメモリテーブルの直接検証

#### 問題
FTD CLI にログインし、FMC から `sftunnel` 経由で同期された全 Dynamic Object の一覧および特定の Dynamic Object (`Dyn_C2_Blacklist`) 内に保持されている現在の IP アドレス数をコマンドで確認せよ。

#### 要件
* FTD CLI コマンドを使用すること。

#### 設定・操作手順

```bash
# ===== FTD CLI にログイン =====

> show dynamic-objects
Object Name: Dyn_C2_Blacklist
  ID: 0050568A-22B2-0ed3-0034-858993459221
  Type: IP
  Mapping Count: 3
  Mappings:
    203.0.113.100
    198.51.100.0/24
    10.250.10.15

Object Name: Dyn_Allowed_DevOps
  ID: 0050568A-22B2-0ed3-0034-858993459222
  Type: IP
  Mapping Count: 2
  Mappings:
    172.16.50.10
    172.16.50.11
```

#### 検証方法
* API で IP を変更した直後に上記コマンドを再度実行し、`Mapping Count` および `Mappings` のリストが Deploy 操作なしで即座に更新されることを検証。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で Dynamic Object `Dyn_Block_IP` を新規作成し、Access Control Policy 内のブロックルールにアサインした。その後、Deploy 操作を一切行わずに、Python スクリプトから REST API を使用して `Dyn_Block_IP` に IP アドレス `203.0.113.50` を追加する POST リクエストを送信した。API リクエストは HTTP 200 OK で正常完了したが、該当 IP からのトラフィックが FTD でブロックされない。原因として正しいものはどれか。

* A) Dynamic Object は REST API からの IP 追加をサポートしていないため。
* B) Dynamic Object の新規作成および ACP ルールへのアサイン後、初回のみ FTD への Policy Deploy を実行する必要があるため。
* C) API リクエストのボディで `"add"` ではなく `"insert"` キーワードを使用する必要があるため。
* D) FTD 上で `ip nbar protocol-discovery` を無効化する必要があるため。

**正解**: **B**
**解説**: Dynamic Object は「IP マッピングの追加・削除」には Deploy が不要ですが、**オブジェクトの新規作成、名前変更、または ACP ルールへのアサイン時には、必ず 1 回だけ FTD へ Policy Deploy を実行する** 必要があります。初回の Deploy が行われていない場合、FTD 側に該当ルール構造が存在しないため評価されません。

---

### 2. トラブルシュート問題

**問題**: 自動化スクリプトを用いて FMC の Dynamic Object に IP アドレス範囲を追加しようとして、以下の REST API リクエストを送信したところ、HTTP 400 Bad Request エラーが返された。エラーの根本原因はどれか。

```json
POST /api/fmc_config/v1/domain/{uuid}/object/dynamicobjects/{id}/mappings
{
  "add": [
    "10.10.10.1-10.10.10.50"
  ]
}
```

* A) REST API 経由でのマッピング追加時は `"add"` キーワードを使用できないため。
* B) Dynamic Object は IP アドレス範囲 (Range: `10.10.10.1-10.10.10.50`) をサポートしておらず、単一 IP または CIDR サブネット表記（例: `10.10.10.0/24`）のみサポートしているため。
* C) JSON ヘッダーに `X-auth-access-token` が含まれていないため。
* D) ポート番号が指定されていないため。

**正解**: **B**
**解説**: Dynamic Object に登録できるデータ形式は **IPv4/IPv6 単一 IP アドレス、または CIDR 形式（例: `10.10.10.0/24`）のみ** です。IP アドレス範囲 (Range: `10.10.10.1-10.10.10.50`) や FQDN はサポートされておらず、指定すると 400 Bad Request エラーが発生します。

---

### 3. Design 問題

**問題**: マルチクラウド環境（AWS / Azure）において、数百台の仮想マシンがオートスケーリングにより頻繁に起動・削除される。これらの VM の IP アドレス変化に追従し、FTD 境界ファイアウォールでリアルタイムにアクセス制御を行いたい。FMC の CPU 処理負荷（Deploy 処理負荷）および Snort プロセスへの影響を最小限に抑えるための最適な設計手法はどれか。

* A) 外部スクリプトで Standard Network Object を毎回編集し、API 経由で毎分 Deploy コマンドを実行する。
* B) Cisco Secure Dynamic Attributes Connector を導入してクラウド上の VM タグを監視させ、FMC の Dynamic Object に IP を自動同期させて Deploy なしで FTD へ即時反映させる。
* C) FTD 上で FlexConfig を使用して ASA CLI の Access-list を毎分書き換える。
* D) 各 VM に個別の VPN ピアを設定する。

**正解**: **B**
**解説**: **Cisco Secure Dynamic Attributes Connector** と **Dynamic Objects** を組み合わせることで、クラウド VM の IP 変化を Deploy 不要で FTD データプレーンへリアルタイム同期できます。これにより FMC の Deploy 処理（Snort リロード等）を一切発生させずに完全なポリシー追従が実現します。

---

### 4. 実装問題

**問題**: FTD CLI にログインし、FMC から `sftunnel` 経由で動的に同期されている最新の Dynamic Object の保持 IP アドレスリストを検証するための正しい CLI コマンドはどれか。

* A) <code>show access-list</code>
* B) <code>show dynamic-objects</code>
* C) <code>show object-group</code>
* D) <code>show running-config object</code>

**正解**: **B**
**解説**: FTD CLI で **`show dynamic-objects`**（または `show dynamic-object name <Name>`）を実行することで、FMC から Deploy なしで同期された Dynamic Object の名前、ID、および現在メモリに保持されている IP アドレス/CIDR リストを直接確認・検証できます。

---

### 5. コンフィグ読解問題

**問題**: 以下の cURL コマンドの目的として適切なものはどれか。

```bash
curl -k -X POST "[suspicious link removed]" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json" \
  -d '{"remove": ["192.0.2.100"]}'
```

* A) 指定した UUID の Dynamic Object 自身を FMC データベースから完全削除する。
* B) 指定した Dynamic Object から IP アドレス `192.0.2.100` のマッピングのみを Deploy なしで即時削除する。
* C) FMC から FTD への全ポリシー展開をアボート（キャンセレーション）する。
* D) 192.0.2.100 を静的ネットワークオブジェクトに変換する。

**正解**: **B**
**解説**: エンドポイント `/object/dynamicobjects/{id}/mappings` に対して POST リクエストで `{"remove": ["192.0.2.100"]}` を送信することで、指定した Dynamic Object 内の特定の IP アドレスマッピングのみを **Deploy 不要で即時に除去** できます。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Configuration Guide - Dynamic Objects]([suspicious link removed])
* [Cisco Secure Firewall Management Center REST API Quick Start Guide]([suspicious link removed])
* [Cisco Secure Dynamic Attributes Connector Configuration Guide]([suspicious link removed])
* [Cisco DevNet: Secure Firewall Management Center REST API Reference]([suspicious link removed])
* [Cisco Live: Network Automation and Programmability with Cisco Secure Firewall (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で「Automate IP blocking via REST API without pushing deployment」といった課題が出題された場合、使用すべき技術は **Dynamic Objects** 一択です。通常の Network Object や Object Group を API で編集すると Deploy が必要になるため、問題文の「without deployment」制約を回避できません。
* **図解（Dynamic Objects リアルタイム同期フロー）**:
  ```text
  [ External Event: SOAR / Cloud Tag Change ]
                      │
                      ▼ (HTTP POST / PUT)
  [ FMC REST API Endpoint ] ────► Updates FMC DB
                      │
                      ▼ (sftunnel: TCP 8305 Instant Push)
  [ FTD Data Plane Cache ] ────► Updates Dynamic Object Memory Table
                      │
                      ▼ (Immediate Policy Enforcement)
  [ Traffic Permitted or Dropped Instantly ]
  ```
* **注意点**:
  * API スクリプトのテスト時は、IP アドレス指定ミスのほか、FMC の Domain UUID や Object UUID の指定間違いが非常に多く発生します。必ず `GET /api/fmc_config/v1/domain/{uuid}/object/dynamicobjects` で目的のオブジェクト UUID を動的検索・取得してから更新リクエストを発行するようコードを実装してください。
