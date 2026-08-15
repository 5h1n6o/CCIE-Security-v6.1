---
layout: default
title: 3.9.b-Data-encoding
nav_order: 2
parent: 3.9-APIs
grand_parent: 3.0-Security-Infrastructure
---

# 3.9.b Data encoding formats (JSON, XML, YAML)

**データエンコーディング形式**は、ネットワークプログラマビリティと自動化（REST API, NETCONF, Ansible等）において、デバイス間やコントローラ間で情報をやり取りするための「共通言語」です。CCIE Security v6.1では、Cisco FMC、ISE、DNA Centerとの対話、およびAnsibleを用いた自動化において、これらの形式を正しく記述・パース・変換する能力が不可欠です,。

---

## 📘 概要

*   **機能概要**: 構造化されたデータをテキストベースで表現する手法。人間が読みやすく、かつコンピュータが効率的に処理できる構造を持ちます。
*   **利用目的**:
    *   **API ペイロード**: REST APIで設定変更情報を送信する際のデータ形式（主にJSON）。
    *   **設定ファイル**: 自動化ツール（Ansible）や構成管理での利用（主にYAML）。
    *   **データモデルの表現**: デバイスの状態や設定を階層的に表現（XML, JSON）。
*   **利用場面**:
    *   FMC APIを使用してアクセスポリシーを作成する際（JSON）。
    *   Ansible PlaybookでFTDの設定を定義する際（YAML）。
    *   NETCONFを使用してルータのインベントリを取得する際（XML）。

---

## 🔑 要点

| 項目 | JSON | XML | YAML |
| :--- | :--- | :--- | :--- |
| **正式名称** | JavaScript Object Notation | eXtensible Markup Language | YAML Ain't Markup Language |
| **可読性** | 高い | 中（タグが多い） | **最高** |
| **主な用途** | **REST API (FMC, ISE)** | NETCONF, SOAP | **Ansible, Docker, 構成ファイル** |
| **データ構造** | Key-Value, List/Array | Tag-based (Tree) | Indentation-based (階層) |
| **文法要素** | `{ }`, `[ ]`, `:` | `<tag> </tag>` | インデント, `-`, `:` |
| **ファイル拡張子** | `.json` | `.xml` | `.yml`, `.yaml` |

---

## 🏗 動作原理

データエンコーディングは、メモリ上の「オブジェクト（Pythonの辞書やリスト等）」と「転送用の文字列」を相互変換（シリアライズ/デシリアライズ）することで機能します。

```text
[ Python Script ]           [ Serialization ]           [ Network Device / API ]
{ "name": "Inside" }  ────→  '{"name": "Inside"}'  ────→  REST API Server
(Dict Object)               (Text Stream)                (Data Processing)
```

1.  **シリアライズ**: プログラム内のデータをJSON/XML/YAML文字列に変換。
2.  **トランスポート**: HTTP(S)やSSH等のプロトコルでテキストを送信。
3.  **デシリアライズ**: 受信側で文字列をパースし、データ構造を復元。

---

## ⚙ 動作シーケンス

1.  **データ定義**: 管理者がYAMLやJSONで設定（例：IPアドレス、オブジェクト名）を定義。
2.  **APIリクエスト**: Pythonの`requests`ライブラリ等が、JSON形式のペイロードをHTTP POST/PUTに含めて送信。
3.  **構文検証**: デバイス側が受信データのフォーマット（括弧の閉じ忘れやインデント等）を検証。
4.  **処理実行**: 検証成功後、データがデータベースや実行コンフィグに反映される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **構文エラーの特定**: ラボ試験では「動作しないスクリプトの修正」が求められることがあります。JSONのコンマ不足、YAMLのインデントミス（タブ文字混入など）を瞬時に見抜く必要があります。
*   **FMC APIとの連携**: FMCはJSONを標準ペイロードとして使用します。FMC API Explorerを利用して、正しいJSON構造をコピー＆ペーストし、Pythonスクリプト内で動的に値を書き換える能力が問われます。
*   **AnsibleによるFTD管理**: YAML形式で記述されたPlaybookを読み取り、変数がどのように渡されているかを理解する必要があります。
*   **データの抽出（Parsing）**: API応答（JSON）の中から、特定のオブジェクトIDやステータスをPythonのループ処理で抽出する手順は頻出です。
*   **XMLの理解**: NETCONFの出力結果（XML）を解析し、特定のインターフェイス状態を確認する問題に備えてください。

---

## 🛠 設定方法

### 1. JSON (FMC ネットワークオブジェクト作成用)
```json
{
  "name": "Inside_LAN",
  "type": "Network",
  "value": "192.168.10.0/24",
  "overridable": false,
  "description": "Created via API"
}
```

### 2. YAML (Ansible Playbook 定義例)
```yaml
- name: Create Object in FMC
  cisco.fmc.fmc_configuration:
    operation: "createNetworkObject"
    data:
      name: "Web_Server"
      value: "10.1.1.10/32"
```

### 3. XML (NETCONF リクエスト例)
```xml
<get-config>
  <source>
    <running/>
  </source>
  <filter>
    <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"/>
  </filter>
</get-config>
```

---

## 🔍 検証コマンド

| 目的 | コマンド / 手法 |
| :--- | :--- |
| **PythonでのJSONパース** | <code>import json; data = json.loads(text)</code> |
| **JSON整形出力** | <code>print(json.dumps(data, indent=4))</code> |
| **YAMLの読み込み** | <code>import yaml; config = yaml.safe_load(file)</code> |
| **XMLの要素検索** | <code>import xml.etree.ElementTree as ET; root = ET.fromstring(text)</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認ポイント |
| :--- | :--- | :--- |
| **JSONDecodeError** | カンマ不足、またはクォーテーションの誤り | 末尾のカンマや、文字列が `"` で囲まれているか。 |
| **YAML Syntax Error** | インデントにタブを使用している、または階層ミス | スペースのみを使用し、`-` と `:` の後のスペースを確認。 |
| **XML Parsing Error** | タグの閉じ忘れ、不正な属性名 | 開始タグと終了タグが一致しているか。 |
| **400 Bad Request** | ペイロードのキー名がAPI仕様と不一致 | API Explorerでフィールド名（`name` vs `objectName`等）を確認。 |

---

## ⚠ 制限事項

*   **YAMLのタブ禁止**: YAMLではインデントにタブを使用できません（スペースのみ）。
*   **JSONのデータ型**: JSONでは数値、文字列、ブール値を厳密に区別します。
*   **XMLの冗長性**: XMLはタグが多くデータ量が大きくなるため、リアルタイムの大量データ転送にはJSONの方が好まれます。

---

## 🔄 他技術との関連

*   **3.9.a REST API**: 通信プロトコルとしてのRESTと、データ形式としてのJSON/XMLの関係。
*   **3.10 DNAC Northbound APIs**: ネットワークインベントリ情報のJSON取得,。
*   **Cisco FMC API Explorer**: インターラクティブにJSON構造をテストするツール。
*   **Ansible / Terraform**: YAML/HCL形式を用いたインフラストラクチャ・アズ・コード。

---

## 🧩 比較表

### 構造の表現方法

| 形式 | リスト（配列）の表現 | オブジェクト（辞書）の表現 |
| :--- | :--- | :--- |
| **JSON** | `["A", "B", "C"]` | `{"key": "value"}` |
| **XML** | `<item>A</item><item>B</item>` | `<key>value</key>` |
| **YAML** | `- A \n - B \n - C` | `key: value` |

---

## 💡 ベストプラクティス

1.  **JSON Lintの活用**: スクリプトに組み込む前に、オンラインツールやIDEの拡張機能で構文をチェックします。
2.  **YAMLでのコメント活用**: YAMLは `#` でコメントが書けるため、Ansible Playbook等では意図を明文化します（JSONは公式にコメント非サポート）。
3.  **変数の外部化**: JSON/YAMLファイルをスクリプト本体から分離し、環境依存の設定を外部ファイルから読み込むように設計します。

---

## 📝 ラボ学習・設定サンプル例

### 1. JSON: FMCホストオブジェクトの定義
*   **要件**: 名前 `DB_Server`、IP `10.1.2.3` のJSONデータを作成せよ。
*   **設定**: `{"name": "DB_Server", "type": "Host", "value": "10.1.2.3"}`

### 2. YAML: Ansible インベントリ作成
*   **要件**: `firewalls` グループに `FTD1` と `FTD2` を定義せよ。
*   **設定**:
    ```yaml
    firewalls:
      hosts:
        FTD1:
        FTD2:
    ```

### 3. JSON: リスト構造の記述
*   **要件**: 複数のDNSサーバ（8.8.8.8, 8.8.4.4）をJSONリストで表現せよ。
*   **設定**: `["8.8.8.8", "8.8.4.4"]`

### 4. XML: インターフェイス設定の表現
*   **要件**: Gi0/1 を `up` 状態にするXMLタグを作成せよ。
*   **設定**: `<interface><name>GigabitEthernet0/1</name><enabled>true</enabled></interface>`

### 5. Python: JSONから特定キーの抽出
*   **要件**: `response.json()` から `id` を取得する1行を書け。
*   **コード**: `obj_id = response.json().get('id')`

### 6. YAML: キー・バリューのネスト
*   **要件**: `interface` の下に `name` と `ip` をネストせよ。

### 7. JSON: ブール値とNullの記述
*   **要件**: `active` を真、`tag` を空として記述せよ。
*   **設定**: `{"active": true, "tag": null}`

### 8. Python: YAMLファイルの読み込み
*   **要件**: `config.yml` を読み込み辞書型に変換せよ。

### 9. JSON: 特殊文字のエスケープ
*   **要件**: Descriptionに `"` を含む文字列を記述せよ。
*   **設定**: `"description": "This is a \"Test\""`

### 10. XML: 名前空間（Namespace）の定義
*   **要件**: Cisco固有の名前空間を指定するXMLタグを作成せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: JSONペイロード `{"ip": 10.1.1.1}` がエラーになる理由は？
    *   **回答**: IPアドレスは文字列として扱う必要があるため、`"10.1.1.1"` とダブルクォーテーションが必要。
2.  **トラブルシュート**: Ansibleを実行すると「expected <block end>, but found '-'」とエラーが出る。YAMLのどこを直すべきか？
    *   **回答**: リストを示す `-` の前のインデントが、親要素と正しく揃っているか、またはスペースが不足していないかを確認する。
3.  **Design**: 設定変更の履歴を管理する際、人間にとって最も読みやすく、コメントが記述可能な形式はどれか？
    *   **回答**: **YAML**。
4.  **実装**: NETCONFを使用してルータから情報を取得する場合、返ってくるデータ形式は通常どれか？
    *   **回答**: **XML**。
5.  **コンフィグ読解**: JSONにおいて `[ ]` と `{ }` の使い分けを述べよ。
    *   **回答**: `[ ]` は順序のあるリスト（Array）、`{ }` は順序のないキー・バリューのペア（Object）を表す。

---

## 🔗 参考リソース

*   **Cisco DevNet**: [Coding Fundamentals - JSON, XML, YAML](https://developer.cisco.com/learning/modules/coding-fundamentals)
*   **Cisco Live (DEVNET-1111)**: [Intro to Data Formats: JSON, XML, YAML](https://www.ciscolive.com/)
*   **W3Schools**: [JSON Syntax Guide](https://www.w3schools.com/js/js_json_syntax.asp)
*   **Ansible Documentation**: [YAML Syntax](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: CCIEラボでは「1文字のミス」が致命的になります。JSONの最後のエントリに不要なコンマ `,` を付けていないか、YAMLでタブキーを押していないか（VSCode等のエディタ設定でスペースに変換されるか）を常に意識してください。
*   **図解**: JSONは「辞書」、YAMLは「箇条書き」、XMLは「本（目次）」とイメージすると、それぞれの構造の利点が理解しやすくなります。
*   **注意点**: Cisco FMC APIなどでは、データ型（Integer vs String）に非常に厳格なため、ドキュメントの型定義を必ず参照してください。
