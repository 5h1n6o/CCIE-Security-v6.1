---
layout: default
title: 3.9.b-Data-encoding
nav_order: 2
parent: 3.9-APIs
grand_parent: 3.0-Security-Infrastructure
---

# 3.9.b Data encoding formats (JSON, XML, YAML)

## 📘 概要

現代のネットワークプログラマビリティおよびネットワーク自動化において、ネットワーク機器とのデータ交換（API、RESTCONF、NETCONF、自動化フレームワーク）や設定ファイルの管理は、従来の CLI（Command Line Interface）のテキストベースから構造化データフォーマットへの移行が不可欠となっています。CCIE Security v6.1 Blueprint 3.9.b では、データ符号化フォーマットの3大主要規格である **JSON (JavaScript Object Notation)**、**XML (eXtensible Markup Language)**、および **YAML (YAML Ain't Markup Language)** の構造的特徴、構文規則、Pythonにおける相互変換・エンコード/デコード処理、ならびに各種ネットワークインターフェース（FMC REST API, ISE ERS API, IOS-XE RESTCONF/NETCONF）における適用方法が試験対象となっています。

* **JSON (JavaScript Object Notation)**:
  * **機能概要**: キーと値のペア、および配列構造でデータを表現する軽量なテキストベースのデータ交換フォーマット（RFC 8259）。
  * **利用目的**: REST API（FMC API、ISE API、FDM API、RESTCONF）における標準データ交換フォーマットとして使用。
  * **利用場面**: Webサービス連携、APIリクエスト・レスポンス処理、軽量な設定データの伝送。

* **XML (eXtensible Markup Language)**:
  * **機能概要**: タグ（Tag）と属性（Attribute）を用いて木構造（Tree Structure）状にデータ要素を定義する拡張可能なマークアップ言語（W3C Recommendation）。
  * **利用目的**: NETCONF（RFC 6241）における通信メッセージおよびデータモデルの表現、標準化されたスキーマ（XSD）検証の実行。
  * **利用場面**: Cisco IOS-XE/NX-OS/ASAのNETCONF管理、ISE ERS APIでのXMLペイロード送信、大規模なデータモデルの表現。

* **YAML (YAML Ain't Markup Language)**:
  * **機能概要**: インデント（オフサイドライン規則）を用いて視認性を極限まで高めた人間可読性の高いデータ構造化フォーマット。
  * **利用目的**: Ansible プレイブック、Docker Compose、Kubernetes マニフェスト、およびネットワークインフラの宣言的設定ファイルの記述。
  * **利用場面**: ネットワーク自動化ツールの設定ファイル作成、複雑な階層型パラメータ定義、プレイブックの記述。


---

## 🔑 要点

JSON、XML、YAML の技術的特徴、主な用途、制限事項および設計上の注意点を下表に示します。

| 項目 | JSON | XML | YAML |
| --- | --- | --- | --- |
| **特徴** | キー/値ペアと配列による軽量構造。仕様が非常にシンプル | タグと属性による木構造。命名空間(Namespace)とスキーマ検証をサポート | インデントベースの人間可読構造。JSONのスーパーセット |
| **用途** | REST API (FMC, ISE, FDM), RESTCONF, Webフック | NETCONF, ISE ERS API (XML選択時), Webサービス (SOAP) | Ansible プレイブック, pyATS / Genie 設定, CI/CD パイプライン |
| **メリット** | 解析(Parse)速度が高速、パケットサイズが小さく軽量、Python標準 `json` モジュールで即座に操作可能 | 厳密な構造定義(XSD)や命名空間の分離が可能。テキストデータに加え属性を保持可能 | 人間にとって極めて読みやすく書きやすい。コメント記法(`#`)を直接記述可能 |
| **デメリット** | コメント記法を非サポート。末尾の余分なカンマ(Trailing Comma)で構文エラー発生 | 冗長なタグ構造によりパケットサイズが肥大化。解析処理(DOM/SAX)のCPU負荷が高い | インデント(タブとスペースの混在)による構文エラーが頻発。セキュリティ脆弱性(PyYAML PyYAML unsafe load) |
| **対応機種/プロトコル** | Cisco FMC, ISE, ASA, FDM, Catalyst 9000 (RESTCONF) | Cisco IOS-XE, NX-OS, ASA, Firepower (NETCONF / ERS) | Cisco pyATS, Ansible Control Node, DevNet ツールチェイン |
| **制限事項** | バイナリデータの直接保持不可(Base64エンコード必須)、DateTime型の標準非サポート | 構文が複雑で手動記述時の入力ミスが発生しやすい | タブ文字(`\t`)の使用は禁止(スペースのみ)。Python標準ライブラリ非搭載(`pyyaml`のインストール必須) |
| **設計上の注意点** | 大容量データ伝送時はストリーミング解析(ijson等)を考慮 | Namespace(ns0:, xmlns)の処理を行わないと要素取得に失敗する | 心理的インデントのズレが論理的設定エラーを引き起こすため、Linter (yamllint) の併用が必須 |


---

## 🏗 動作原理

ネットワーク機器やAPIサーバーとクライアント（Pythonスクリプト）間でデータエンコーディングフォーマットが処理されるアーキテクチャおよびパースモデルを説明します。

```
 [ Python スクリプト (Client) ]
   │
   ├─ 1. 内部データ構造 (Python Dictionary / List / Tree)
   │     │
   │     ├── (Serialize / Dump)
   │     │     ├─ json.dumps()    ──> [ JSON String ] ──> HTTP Header: Content-Type: application/json
   │     │     ├─ ET.tostring()   ──> [ XML String ]  ──> HTTP Header: Content-Type: application/xml
   │     │     └─ yaml.safe_dump() ──> [ YAML String ] ──> Config File / Ansible / pyATS
   │     │
   │     ▼ (Network Transmission / API Request)
   │
   [ Network Device / API Gateway (FMC, ISE, IOS-XE) ]
   │
   │     ▲ (Network Transmission / API Response)
   │     │
   │     ├── (Deserialize / Load)
   │     │     ├─ json.loads()    <── [ JSON String ] <── Response Status: 200 OK
   │     │     ├─ ET.fromstring() <── [ XML String ]  <── NETCONF <rpc-reply>
   │     │     └─ yaml.safe_load() <── [ YAML File ]   <── Ansible Inventory / Vault
   │
   └─ 2. Python オブジェクト変換後のデータ抽出・操作
```

1. **シリアライゼーション (Serialization / Encoding)**:
   * Python のデータ構造（`dict`, `list`, `str`, `int`, `bool`）を、ネットワーク経由で送信可能、あるいはファイルに保存可能な特定のデータ記述フォーマット（文字列バイナリ）に変換する処理です。
   * JSON: `json.dumps()` または `json.dump()`
   * XML: `xml.etree.ElementTree.tostring()` または `xmltodict.unparse()`
   * YAML: `yaml.safe_dump()`

2. **デシリアライゼーション (Deserialization / Decoding / Parsing)**:
   * ネットワーク機器から受信したテキスト（JSON, XML）やファイルから読み込んだテキスト（YAML）を解析し、Pythonが直接アクセス可能なデータ構造（`dict` や `Element` オブジェクト）に変換する処理です。
   * JSON: `json.loads()` または `json.load()`
   * XML: `xml.etree.ElementTree.fromstring()` または `xmltodict.parse()`
   * YAML: `yaml.safe_load()`

3. **パーサーモデルの相違点**:
   * **JSON Parser**: 文字列をトークン分割し、`{}` を Dictionary、`[]` を List、キーと値を基本データ型にマッピング。
   * **XML DOM / ElementTree Parser**: タグの階層構造を階層ツリー（Node, Parent, Child）としてメモリ上に展開。要素属性（`attrib`）とテキスト値（`text`）を区別。
   * **YAML Parser**: 行指向でインデントのスペース数を計測し、ブロック構造（`-` によるリスト、`:` による辞書）を構築。


---

## ⚙ 動作シーケンス

PythonスクリプトがREST APIやNETCONFを介して各種フォーマットを処理するシーケンスを以下に示します。

```
[ Python Client Script ]               [ Cisco REST API / RESTCONF / NETCONF ]
         │                                               │
         │ ─── 1. API リクエストの組み立て ──────────────> │
         │      - Python Dict を作成                      │
         │      - JSON/XML にシリアライズ                │
         │      - Header: Content-Type 設定              │
         │                                               │
         │                                               │ ─── 2. パケット受信 & パーサー検証
         │                                               │      - 構文エラーチェック (Syntax Check)
         │                                               │      - スキーマ検証 (Schema Validation)
         │                                               │      - オブジェクト生成 & データベース登録
         │                                               │
         │ <─── 3. API レスポンス返送 ─────────────────── │
         │      - HTTP 200 OK / 201 Created              │
         │      - Header: Content-Type: application/json │
         │      - ペイロード (JSON/XML 文字列)            │
         │                                               │
         │ ─── 4. レスポンスのデシリアライズ ──────────── │
         │      - json.loads() / xmltodict.parse()       │
         │      - Python Dict から必要なデータ抽出         │
         │                                               │
```

1. **データ作成とシリアライズ**:
   * クライアントは Python の辞書オブジェクトとして設定データを定義します。
   * リクエストヘッダー（`Content-Type: application/json` や `application/yang-data+json`）を設定し、文字列化したペイロードを HTTP Body に乗せて送信します。
2. **サーバー側処理**:
   * APIサーバー（FMC, ISE, IOS-XE）は受信した文字列を構文解析します。
   * 構文違反（例: JSONのダブルクォート欠落、XMLの閉じタグ不足、YAMLの不正なインデント）がある場合、即座に HTTP 400 Bad Request や XML RPC Error を返答します。
3. **データ展開と活用**:
   * クライアントは受信したレスポンス文字列をデシリアライズし、キー指定（`response_json['items'][0]['id']` 等）で目的のオブジェクトIDや状態値を取得します。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Securityラボ試験において、PythonスクリプトによるAPI操作やデータフォーマット処理で最も頻出するポイント、ハマりやすい罠、および対策を整理します。

### Blueprintで重要なポイント

1. **Python標準モジュールと外部ライブラリの使い分け**:
   * 試験環境で使用可能なライブラリ（json, xml.etree.ElementTree）と、インストールが必要な外部ライブラリ（requests, pyyaml, xmltodict）を把握すること。
   * インターネット接続が制限されている試験環境では、標準モジュール（json, xml.etree.ElementTree）のみで全処理を完結できるスキルが極めて重要です。
2. **JSONにおける型マッピングと構文エラーの回避**:
   * Pythonの True/False/None は JSON で true/false/null に変換されます。手動の文字列連結でJSONを作成すると True が大文字のまま出力され API サーバーで 400 エラーになる事故が多発します。必ず json.dumps() を使用すること。
   * シングルクォートは JSON 規格上無効です。ダブルクォートが必須です。
3. **XMLにおける Namespace (xmlns) の取り扱い**:
   * RESTCONF や NETCONF の XML レスポンスには Namespace が含まれます。
   * ElementTree.find() や findall() を使用する際、命名空間プレフィックスを指定しないと要素がマッチせず None になる罠が存在します。
4. **YAMLのインデント規則と型変換の罠**:
   * タブ文字の混在による ScannerError。
   * yes / no や 1.0 が Boolean や Float に型変換される問題。
5. **JSON vs XML vs YAML の相互変換シナリオ**:
   * ラボ試験問題例: YAML形式で提供された設定ファイルを読み込み、内部でJSONに変換して Cisco FMC REST API へ POST リクエストを送信せよ。




---

## 🛠 設定方法 (Pythonスクリプトによる操作)

Python環境における JSON、XML、YAML のエンコード、デコード、相互変換、ファイル読み書きの実践的な実装コードパターンを記述します。

### 1. JSON の操作 (json モジュール)

```python
import json

python_data = {
    "name": "Access-Rule-01",
    "action": "ALLOW",
    "enabled": True,
    "sourceNetworks": ["10.1.1.0/24", "10.1.2.0/24"],
    "destinationPort": None
}

json_string = json.dumps(python_data, indent=2)
print("=== JSON String ===")
print(json_string)

parsed_dict = json.loads(json_string)
print(f"Rule Name: {parsed_dict['name']}, Enabled: {parsed_dict['enabled']}")
```

### 2. XML の操作 (xml.etree.ElementTree および xmltodict)

```python
import xml.etree.ElementTree as ET
import xmltodict

root = ET.Element("config", xmlns="urn_netconf_base")
native = ET.SubElement(root, "native", xmlns="urn_cisco_ios_xe_native")
hostname = ET.SubElement(native, "hostname")
hostname.text = "R1-CORE"

xml_raw_string = ET.tostring(root, encoding="utf-8").decode("utf-8")
print("=== XML via ElementTree ===")
print(xml_raw_string)
```

### 3. YAML の操作 (pyyaml ライブラリ)

```python
import yaml

network_config = {
    'router': {
        'hostname': 'Edge-Router-01',
        'interfaces': [
            {'name': 'GigabitEthernet1', 'ip': '10.0.0.1/30', 'description': 'Uplink to ISP'},
            {'name': 'GigabitEthernet2', 'ip': '172.16.1.1/24', 'description': 'LAN Gateway'}
        ]
    }
}

yaml_string = yaml.safe_dump(network_config, default_flow_style=False, sort_keys=False)
print("=== YAML Output ===")
print(yaml_string)

parsed_yaml = yaml.safe_load(yaml_string)
print(f"Router Hostname: {parsed_yaml['router']['hostname']}")
```




---

## 🔍 検証コマンド

PythonスクリプトおよびLinux環境でデータエンコーディングの整合性や構文を検証するコマンド一覧。

| 目的 | コマンド / コード snippet | 説明 |
| --- | --- | --- |
| **JSON 構文チェック (CLI)** | `python3 -m json.tool input.json` | Linux CLI上でJSONファイルの構文チェックおよび整形出力 |
| **JSON の整形式検証 (Python)** | `json.loads(data)` | JSON文字列のロード試行でエラーハンドリング |
| **XML の整形式検証 (Python)** | `ET.fromstring(xml_data)` | XML構文エラーの検知 |
| **YAML の構文検証 (Python)** | `yaml.safe_load(yaml_data)` | インデントや記法エラーの検出 |
| **YAML 構文チェック (CLI)** | `yamllint config.yaml` | YAML専用Linterによるインデント・構文の自動検査 |

---

## 🚨 トラブルシュート

データエンコーディング処理における主要な症状、原因、確認コード、および解決策を下表に整理します。

| 症状 | 原因 | 確認コード / エラー | 対処方法 |
| --- | --- | --- | --- |
| **JSONDecodeError** | JSON文字列内にシングルクォートが含まれている | `JSONDecodeError` | キーと文字列値をダブルクォートに置換。`json.dumps()`を利用 |
| **XML解析で AttributeError** | Namespace を無視して `find()` を実行した | `find('hostname')` が `None` を返却 | XPath表記で `ns` 辞書を指定するか `xmltodict` を使用 |
| **ScannerError** | インデントにタブ文字が混在している | `ScannerError` | タブをスペース2文字に変換。`:` の直後に半角スペースを入れる |
| **HTTP 400 Bad Request** | リクエストヘッダーの Content-Type 不一致 | HTTP Status 400 | `headers={'Content-Type': 'application/json'}` を明示 |
| **TypeError in XML** | ElementTree に int や bool を代入した | `TypeError: cannot serialize` | `.text` にセットする値は `str()` でキャストする |


## ⚠ 制限事項

データエンコーディング規格を使用する際の制限事項および注意点です。

* **JSON**:
  * コメントを記述できないため、設定の意図やドキュメンテーションをファイル内に保持不可。
  * 日時型（DateTime）やバイナリ型の標準仕様が存在せず、ISO 8601 文字列や Base64 エンコード文字列への変換が必要。
  * 巨大な JSON ファイル（数ギガバイト）の全件ロード（`json.load()`）はメモリ不足を引き起こすため、ストリーミング解析（`ijson` 等）の検討が必要。

* **XML**:
  * タグによるオーバーヘッドが大きく、ネットワーク帯域および CPU のパース負荷が最も高い。
  * 属性（Attribute）と要素（Element）のどちらにデータを格納するかという設計上の揺れが存在。
  * 外部実体参照（XXE: XML External Entity）攻撃によるローカルファイル読み取りの脆弱性に注意が必要。

* **YAML**:
  * タブ文字（`\\t`）によるインデントは厳格に禁止（`ScannerError`）。
  * PyYAML のデフォルト `load()` 関数は任意オブジェクトのインスタンス化を許すため、リモートコード実行 (RCE) 脆弱性を招く。必ず `safe_load()` を使用すること。
  * `yes` / `no` / `true` / `false` / `on` / `off` 等が非文字列として解釈され、ネットワーク設定の `description: no` が `False` に変換される罠に注意。


---

## 🔄 他技術との関連

本項目（データ符号化フォーマット）と関連する CCIE Security Blueprint 項目およびプロトコルとの連携関係を以下に示します。

* **FMC / ISE / FDM REST API (Blueprint 3.9.a)**:
  * クライアントは POST/PUT 時のペイロードとして JSON を使用。ヘッダーに `Content-Type: application/json` を指定。
* **RESTCONF / NETCONF (Blueprint 3.9.a / 1.11)**:
  * NETCONF は XML のみをサポート（`application/yang-data+xml`）。
  * RESTCONF は JSON と XML の両方をサポート（`application/yang-data+json` および `application/yang-data+xml`）。
* **YANG データモデル (Blueprint 3.9)**:
  * データモデルの定義は YANG（RFC 6020）で行われ、実際のインスタンス化データが JSON または XML にエンコードされて伝送されます。
* **Ansible / pyATS (ネットワーク自動化ツール)**:
  * Ansible プレイブックおよび pyATS Testbed ファイルは YAML 形式で記述されます。


---

## 🧩 比較表

3つのデータエンコーディングフォーマットを詳細に比較します。

| 比較項目 | JSON (JavaScript Object Notation) | XML (eXtensible Markup Language) | YAML (YAML Ain't Markup Language) |
| --- | --- | --- | --- |
| **主用途** | REST API, Webフック, 設定伝送 | NETCONF, レガシーSOAP, スキーマ厳密データ | Ansible, pyATS, CI/CD設定ファイル |
| **可読性** | 中程度 (インデント整形時) | 低い (冗長なタグ構造) | 極めて高い (直感的構造) |
| **パース速度** | 極めて高速 | 比較的低速 (DOM/SAX) | 中程度 |
| **データ型サポート** | String, Number, Boolean, Array, Object, Null | 文字列のみ (XSDにより型定義可能) | String, Int, Float, Bool, List, Map, Timestamp等 |
| **コメント** | 非対応 | 対応 (`<!-- comment -->`) | 対応 (`# comment`) |
| **スキーマ検証** | JSON Schema (RFCDraft) | XML Schema (XSD), DTD | YAML Schema, Kube-Linter |
| **Python標準対応** | 対応 (`import json`) | 対応 (`import xml.etree.ElementTree`) | サードパーティ (`import yaml`) |


---

## 💡 ベストプラクティス

1. **Python における安全なロード処理の徹底**:
   * YAML ファイルをロードする際は、決して `yaml.load()` を直接使わず、常に `yaml.safe_load()` を使用してセキュリティリスクを排除する。
2. **REST API / RESTCONF 操作での標準エンコーダ利用**:
   * 手動での文字列フォーマット（f-string）による JSON/XML の組み立てを回避し、必ず Dict を構築した上で `json.dumps()` や `ElementTree` を使用してエンコードする。
3. **XML 解析における `xmltodict` の積極利用**:
   * 試作や自動化スクリプト開発において、繁雑な ElementTree / Namespace 処理を回避するため `xmltodict.parse()` を活用し、Python 辞書として直感的に操作する。
4. **Ansible / YAML のインデント・文字列エスケープ管理**:
   * 文字列として扱いたい IP サブネットやブール値風の文字列（`yes`, `no`, `10.1.1.0/24`）は、クォート（`'yes'`）で明示的にエスケープする。


## 📝 ラボ学習・設定サンプル例

CCIE Securityラボ試験レベルの実践的な Python 自動化・データエンコーディング演習例を10個掲載します。

### 演習1: JSON データの相互変換とスキーマバリデーション

* **問題**: ネットワーク機器の設定パラメータを受け取り、JSON形式にエンコードして保存した後、JSON Schema を用いてデータの妥当性を検証せよ。
* **要件**:
  * `jsonschema` ライブラリ（または手動チェック）を用いて `hostname` (文字列), `asn` (1-65535の数値) の型および範囲を検証すること。
  * 整形された JSON 文字列として出力すること。

```python
import json

# ネットワークデータ定義
device_data = {
    "hostname": "Core-Rtr-01",
    "asn": 65000,
    "interfaces": [
        {"name": "Gi1", "ip": "10.0.0.1"},
        {"name": "Gi2", "ip": "10.0.0.5"}
    ]
}

# 1. シリアライズ
json_payload = json.dumps(device_data, indent=2)

# 2. 手動簡易スキーマ検証関数
def validate_device_json(json_str):
    data = json.loads(json_str)
    if not isinstance(data.get("hostname"), str):
        raise ValueError("hostname must be string")
    asn = data.get("asn")
    if not isinstance(asn, int) or not (1 <= asn <= 65535):
        raise ValueError("asn must be int between 1 and 65535")
    return True

# 検証
if validate_device_json(json_payload):
    print("Validation Passed!")
    print(json_payload)
```

* **検証方法**: スクリプトを実行し、`Validation Passed!` と整形された JSON が出力されることを確認。

---

### 演習2: Cisco FMC REST API 用 Access Rule JSON ペイロード構築

* **問題**: Cisco FMC に新しい Access Control Rule を追加するための JSON ペイロードを Python 辞書から動的に生成せよ。
* **要件**:
  * ルール名 `Block-Malicious-IP`, アクション `BLOCK`, 有効化フラグ `True`, 送信元IP `192.168.99.0/24` を指定すること。

```python
import json

def build_fmc_access_rule(name, action, source_ip):
    rule = {
        "name": name,
        "action": action,
        "enabled": True,
        "type": "AccessRule",
        "sourceNetworks": {
            "objects": [
                {
                    "type": "IPv4Network",
                    "value": source_ip
                }
            ]
        }
    }
    return json.dumps(rule, indent=4)

payload = build_fmc_access_rule("Block-Malicious-IP", "BLOCK", "192.168.99.0/24")
print(payload)
```

* **検証方法**: 出力された JSON 内で `enabled` が小文字の `true` に変換されていることを確認。

---

### 演習3: ElementTree による NETCONF XML `<config>` メッセージの生成

* **問題**: Cisco IOS-XE の NETCONF インターフェース向けに、ホスト名設定用の XML メッセージを手動構築せよ。
* **要件**:
  * `xml.etree.ElementTree` を使用すること。
  * Namespace として `urn_ietf_params_xml_ns_netconf_base` を設定すること。

```python
import xml.etree.ElementTree as ET

def create_netconf_hostname_xml(hostname_str):
    config_elem = ET.Element("config", xmlns="urn_ietf_params_xml_ns_netconf_base_1_0")
    native_elem = ET.SubElement(config_elem, "native", xmlns="urn_cisco_ios_xe_native")
    hostname_elem = ET.SubElement(native_elem, "hostname")
    hostname_elem.text = hostname_str
    
    return ET.tostring(config_elem, encoding="utf-8").decode("utf-8")

xml_output = create_netconf_hostname_xml("R1-HQ")
print(xml_output)
```

* **検証方法**: XML構造に `<config>` と `<native>` タグが正しく入れ子処理されていることを確認。

---

### 演習4: `xmltodict` を使用した XML レスポンスからのデータ抽出

* **問題**: NETCONF `<rpc-reply>` の XML 文字列から、全インターフェースのIPアドレスを辞書として抽出せよ。
* **要件**: `xmltodict` ライブラリを使用して可読性の高いコードを書くこと。

```python
import xmltodict

xml_reply = """<rpc-reply xmlns="urn_ietf_params_xml_ns_netconf_base_1_0">
  <data>
    <interfaces xmlns="urn_ietf_params_xml_ns_yang_ietf_interfaces">
      <interface>
        <name>GigabitEthernet1</name>
        <ip xmlns="urn_ietf_params_xml_ns_yang_ietf_ip">
          <address><ip>10.1.1.1</ip><netmask>255.255.255.0</netmask></address>
        </ip>
      </interface>
    </interfaces>
  </data>
</rpc-reply>"""

data_dict = xmltodict.parse(xml_reply)
iface = data_dict['rpc-reply']['data']['interfaces']['interface']
ip_addr = iface['ip']['address']['ip']
print(f"Interface: {iface['name']}, IP: {ip_addr}")
```

* **検証方法**: `Interface: GigabitEthernet1, IP: 10.1.1.1` が出力されることを確認。

---

### 演習5: YAML 設定ファイルのロードと JSON への相互変換

* **問題**: ネットワーク構成が定義された YAML 設定ファイルをロードし、JSON 形式に変換してファイルへ保存せよ。
* **要件**: `yaml.safe_load()` および `json.dump()` を使用すること。

```python
import yaml
import json

yaml_data = """
bgp_config:
  local_as: 64512
  router_id: 1.1.1.1
  neighbors:
    - ip: 10.0.0.2
      remote_as: 64513
      description: "PE-Router-02"
"""

# 1. YAML Load
dict_data = yaml.safe_load(yaml_data)

# 2. JSON Dump
json_output = json.dumps(dict_data, indent=2)
print(json_output)
```

* **検証方法**: YAMLのリスト構文が JSON の `neighbors: [...]` 配列に正しく変換されることを確認。

---

### 演習6: pyATS / Genie 用 YAML Testbed ファイルの動的生成

* **問題**: ルーターIPのリストから、pyATS で利用可能な YAML 形式の Testbed 設定ファイルを自動生成せよ。

```python
import yaml

devices = [
    {"name": "R1", "ip": "10.0.0.1", "os": "iosxe"},
    {"name": "R2", "ip": "10.0.0.2", "os": "iosxe"}
]

testbed = {"devices": {}}
for d in devices:
    testbed["devices"][d["name"]] = {
        "connections": {
            "cli": {
                "protocol": "ssh",
                "ip": d["ip"]
            }
        },
        "os": d["os"],
        "type": "router"
    }

yaml_testbed = yaml.safe_dump(testbed, default_flow_style=False)
print(yaml_testbed)
```

* **検証方法**: 出力結果が pyATS 仕様に適合する YAML 階層になっていることを確認。

---

### 演習7: RESTCONF 用 XML と JSON ペイロードの双方向変換

* **問題**: RESTCONF リクエスト用の JSON ペイロードを読み込み、同等の意味を持つ XML ペイロードへ変換せよ。

```python
import json
import xmltodict

json_str = '{"Cisco-IOS-XE-native:hostname": "Switch-Core-01"}'

# JSON -> Dict -> XML
data_dict = json.loads(json_str)
xml_payload = xmltodict.unparse({"config": data_dict}, pretty=True)
print(xml_payload)
```

* **検証方法**: `<config><Cisco-IOS-XE-native:hostname>Switch-Core-01...</config>` 形式が出力されることを確認。

---

### 演習8: 破損した JSON/XML/YAML データの堅牢なエラーハンドリング

* **問題**: ネットワークから受信した不完全なデータに対して、構文エラーを適切に検知・パース中断するスクリプトを作成せよ。

```python
import json
import xml.etree.ElementTree as ET
import yaml

bad_json = '{"hostname": "R1",}' # 末尾カンマ

try:
    json.loads(bad_json)
except json.JSONDecodeError as e:
    print(f"JSON Error Caught: {e}")

bad_xml = '<root><host>R1</root>' # タグ未閉鎖
try:
    ET.fromstring(bad_xml)
except ET.ParseError as e:
    print(f"XML Error Caught: {e}")

bad_yaml = "router:\n\thostname: R1" # タブ混在
try:
    yaml.safe_load(bad_yaml)
except yaml.YAMLError as e:
    print(f"YAML Error Caught: {e}")
```

* **検証方法**: 3つの各フォーマットで例外がキャッチされ、エラーメッセージが表示されることを確認。

---

### 演習9: Ansible Playbook 用 YAML 変数の検証と文字列エスケープ

* **問題**: YAML 内でブール値として誤解されやすい文字列（`yes`, `no`）を安全に処理するコードを書け。

```python
import yaml

# 暗黙の型変換が起きる例
yaml_raw = """
interface_status:
  Gi1: yes
  Gi2: "no"
"""

parsed = yaml.safe_load(yaml_raw)
print(f"Gi1 type: {type(parsed['interface_status']['Gi1'])}, value: {parsed['interface_status']['Gi1']}")
print(f"Gi2 type: {type(parsed['interface_status']['Gi2'])}, value: {parsed['interface_status']['Gi2']}")
```

* **検証方法**: クォートなしの `yes` が `bool` 型に、ダブルクォートありの `"no"` が `str` 型になることを確認。

---

### 演習10: 複雑なネスト構造を持つ大容量 JSON の目的データピンポイント抽出

* **問題**: Cisco FMC API から返却されたネストの深い JSON レスポンスから、特定の Access Rule 名と Action のみを抽出せよ。

```python
import json

fmc_response = {
    "items": [
        {"name": "Rule-101", "action": "ALLOW", "id": "uuid-1"},
        {"name": "Rule-102", "action": "BLOCK", "id": "uuid-2"}
    ],
    "paging": {"count": 2, "offset": 0}
}

rules_summary = [
    {"rule_name": item["name"], "rule_action": item["action"]}
    for item in fmc_response.get("items", [])
]

print(json.dumps(rules_summary, indent=2))
```

* **検証方法**: `[{"rule_name": "Rule-101", "rule_action": "ALLOW"}, ...]` が抽出されることを確認。


## ❓ 想定試験問題

CCIE Security ラボ試験を想定した選択式・記述式の実践問題と解説です。

### 質問1 (スクリプト読解)

**問題**: 以下の Python スクリプトを実行して Cisco FMC に Access Rule オブジェクトを作成しようとしたところ、API サーバーから HTTP Status 400 (Bad Request) が返却されました。原因として最も適切なものを選択してください。

```python
import requests
import json

rule_data = "{'name': 'Block_Rule', 'action': 'BLOCK', 'enabled': True}"
headers = {'Content-Type': 'application/json', 'X-auth-access-token': 'token123'}

response = requests.post("https_fmc_example_com_api_accessrules", data=rule_data, headers=headers, verify=False)
```

* A. `verify=False` が指定されているため、リクエストが送信されなかった。
* B. `rule_data` がシングルクォートで書かれた文字列であり、かつ `True` が Python の Boolean のまま文字列化されているため、無効な JSON 構文となっている。
* C. `headers` に `Accept: application/json` が指定されていないため。
* D. `requests.post()` の引数として `data` ではなく `json` を使用しなければならない。

**正解**: **B**

**解説**: JSON 規格（RFC 8259）では、文字列およびキーを囲む引用符はダブルクォート（`"`）でなければならず、シングルクォート（`'`）は構文エラーとなります。また、Python の `True` は JSON では小文字の `true` である必要があります。手動で文字列を構築するのではなく、Python 辞書を定義して `json.dumps(dict_data)` を利用すべきです。

---

### 質問2 (トラブルシューティング)

**問題**: Python の `xml.etree.ElementTree` を使用して、Cisco IOS-XE の NETCONF レスポンス XML から `<hostname>` 要素を取得しようとしましたが、`root.find('.//hostname')` が常に `None` を返します。XML レスポンスには `<hostname>R1</hostname>` が確実に存在します。対処方法として正しいものを選択してください。

* A. `ElementTree` は属性を持たない要素を検索できないため、`minidom` ライブラリに変更する。
* B. XML レスポンスに含まれる命名空間 (xmlns) を辞書として定義し、`root.find('.//ios:hostname', namespaces=ns)` のようにプレフィックス付きで検索する。
* C. `find()` の代わりに `getroot()` を再度呼び出す。
* D. XML レスポンスを一度ファイルに保存してから `parse()` する。

**正解**: **B**

**解説**: NETCONF や RESTCONF の XML レスポンスには通常デフォルト命名空間（`xmlns="..."`）が定義されています。`ElementTree` では命名空間配下の要素は `{namespace}tagname` という内部表現になるため、検索時に `namespaces` 辞書を指定してプレフィックス検索を行うか、`xmltodict` ライブラリを使用して辞書に変換してアクセスする必要があります。

---

### 質問3 (Design)

**問題**: 大規模なネットワーク自動化基盤を設計するにあたり、Ansible プレイブック、NETCONF、REST API のそれぞれで採用すべき最適なデータエンコーディングフォーマットの組み合わせとして最も適切な選択肢を選びなさい。

* A. Ansible: XML / NETCONF: JSON / REST API: YAML
* B. Ansible: YAML / NETCONF: XML / REST API: JSON
* C. Ansible: JSON / NETCONF: YAML / REST API: XML
* D. Ansible: YAML / NETCONF: YAML / REST API: XML

**正解**: **B**

**解説**: Ansible のプレイブックやインベントリファイルは人間が最も読み書きしやすい **YAML** 形式で記述されます。NETCONF（RFC 6241）のトランスポートは **XML** 構造のメッセージおよび YANG データモデル表現を基本とします。REST API（Cisco FMC, ISE, FDM）は軽量でパースが容易な **JSON** 形式をデファクト標準として採用しています。

---

### 質問4 (実装)

**問題**: 以下の要求仕様を満たす Python スクリプトの空欄 (1) および (2) に入る適切なコードを記述せよ。

* 要求仕様: YAML 形式の文字列 `yaml_str` を安全にデシリアライズして Python 辞書に変換し、それをインデント幅 2 の整形式 JSON 文字列に変換して出力する。

```python
import yaml
import json

yaml_str = """
device:
  name: Firewall-01
  management_ip: 10.10.10.1
"""

# (1) YAML の安全なロード
data = yaml.______(yaml_str)

# (2) JSON へのエンコード
json_output = json.______(data, indent=2)

print(json_output)
```

**解答**:
* (1): `safe_load`
* (2): `dumps`

**解説**: PyYAML でセキュリティ脆弱性を防ぎ安全にロードするには `yaml.safe_load()` を使用します。Python 辞書オブジェクトを JSON 形式の文字列にシリアライズするには `json.dumps()` を使用します。

---

### 質問5 (コンフィグ/プロトコル読解)

**問題**: Cisco IOS-XE RESTCONF API に対して、特定のインターフェース設定を JSON 形式で送信して更新しようとしています。HTTP リクエストヘッダーに設定すべき正しい `Content-Type` を選択してください。

* A. `Content-Type: text/plain`
* B. `Content-Type: application/yang-data+json`
* C. `Content-Type: application/xml+yang`
* D. `Content-Type: multipart/form-data`

**正解**: **B**

**解説**: RESTCONF（RFC 8040）において、YANG データモデルに基づく JSON ペイロードを送信する際の標準メディアタイプは `application/yang-data+json` です（XML の場合は `application/yang-data+xml`）。一般的な `application/json` も受け付けられる場合がありますが、RFC 仕様上の正確な指示としては `application/yang-data+json` が最も推奨されます。


---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび標準規格の参照リソース一覧です。

* Cisco DevNet: REST API Basics and Data Encoding
* RFC 8259: The JavaScript Object Notation (JSON) Data Interchange Format
* RFC 6241: Network Configuration Protocol (NETCONF)
* RFC 8040: RESTCONF Protocol
* PyYAML Official Documentation & Security Guidelines
* Python 3 Documentation: json — JSON encoder and decoder
* Python 3 Documentation: xml.etree.ElementTree — The ElementTree XML API


---

## 📝 **補足（Notes）**

### 学習メモまとめ

1. **JSON 構文の鉄則**:
   * キーおよび文字列値はダブルクォート（`"`）で囲む。
   * オブジェクトやリストの最後の要素の後にカンマを置かない（Trailing Comma の禁止）。
   * `json.dumps()` でシリアライズ、`json.loads()` でデシリアライズ。

2. **XML 処理の鉄則**:
   * XML にはデフォルト命名空間（`xmlns="..."`）が存在するため、要素検索時は `ns` 辞書を渡すか `xmltodict` を使用する。
   * 文字列以外の値（`int`, `bool`）を XML エレメントのテキストに設定する際は、必ず `str()` でキャストする。

3. **YAML 処理の鉄則**:
   * インデントには絶対スペースのみを使用する（タブ文字の排除）。
   * パース時は常に `yaml.safe_load()` を使用して任意コード実行脆弱性を回避する。
   * ブール値として自動変換される文字列（`yes`, `no`, `true`, `false`）を文字列として保持したい場合はシングル/ダブルクォートで囲む。


