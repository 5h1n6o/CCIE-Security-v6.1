---
layout: default
title: 3.10.a-Auth
nav_order: 1
parent: 3.10-DNAC-APIs
grand_parent: 3.0-Security-Infrastructure
---

# 3.10.a Authentication and authorization

Cisco DNA Center (DNAC) の **Northbound API** は、外部アプリケーションやスクリプトがコントローラと対話するための RESTful インターフェイスです。その中でも **Authentication（認証）と Authorization（認可）** は、すべての API 操作の起点となる最重要ステップであり、セキュアな自動化を実現するための基盤となります。

---

## 📘 概要

*   **機能概要**: DNAC API へのアクセスを許可するために、ユーザー資格情報を検証し、一時的なアクセストークンを発行する機能です。
*   **利用目的**: スクリプトや外部システム（ServiceNow, Ansible 等）が DNAC のリソースにアクセスする際のセキュリティを確保します。
*   **どのような場面で利用するか**: 
    *   自動化スクリプトの実行開始時（トークン取得）。
    *   外部 AAA サーバ（ISE 等）を使用した管理者アクセスの統合。
    *   API 実行ユーザーに対するロールベースの権限管理（RBAC）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **プロトコル** | HTTPS (TCP 443) |
| **認証方式** | HTTP Basic Authentication (トークン取得時) |
| **認可方式** | トークンベース (以降の API コールで `X-Auth-Token` ヘッダーを使用) |
| **データ形式** | JSON |
| **トークン有効期限** | デフォルト 1 時間 |
| **認証エンドポイント** | `/dna/system/api/v1/auth/token` |
| **認可の仕組み** | DNAC 内部の RBAC（Role-Based Access Control）に依存 |

---

## 🏗 動作原理

DNAC Northbound API は、ステートレスな REST アーキテクチャを採用していますが、セキュリティのために **Token-based Authentication** を使用します。

1.  **クライアント**（Python スクリプト等）がユーザー名とパスワードをエンコードして DNAC に送信します。
2.  **DNAC** は資格情報を確認し、有効な場合に一意の **アクセストークン** を返します。
3.  **クライアント** は、取得したトークンを HTTP ヘッダーに含めることで、以降のリクエストに対する認可を得ます。

---

## ⚙ 動作シーケンス

1.  **POST Request**: クライアントが `/dna/system/api/v1/auth/token` に対して、`Authorization: Basic <Base64_Encoded_Credentials>` ヘッダーを付けて POST リクエストを送信します。
2.  **Response**: 成功すると HTTP 200 OK とともに、JSON ボディ内で `Token` 文字列が返されます。
3.  **Subsequent API Calls**: 次の API コール（例：デバイス一覧取得）を行う際、HTTP ヘッダーに `X-Auth-Token: <Token>` を付加します。
4.  **RBAC Check**: DNAC は受け取ったトークンに紐付くユーザーのロール（SUPER-ADMIN, NETWORK-ADMIN 等）を確認し、リクエストされた操作を許可するか判断します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Requests ライブラリの習熟**: Python の `requests.post()` を使用して、認証ヘッダーを正しく構成できる必要があります。
*   **SSL 証明書の検証回避**: ラボ環境では自己署名証明書が使われることが多いため、`verify=False` オプションの使用と、警告（urllib3）の抑制方法を覚えておいてください。
*   **JSON のパース**: レスポンスからトークン文字列のみを抽出する辞書操作（`response.json()['Token']`）は必須スキルです。
*   **ステータスコードの理解**: 
    *   `401 Unauthorized`: ユーザー名/パスワードの間違い。
    *   `403 Forbidden`: トークンは有効だが、操作権限がない（認可エラー）。
*   **AAA 統合**: DNAC に Cisco ISE を AAA サーバとして追加する GUI 操作が問われる可能性があります。

---

## 🛠 設定方法

### 1. Python による認証トークンの取得
```python
import requests
import json
from requests.auth import HTTPBasicAuth

# 警告を非表示にする（ラボ環境用）
requests.packages.urllib3.disable_warnings()

dnac_ip = "10.1.1.10"
auth_url = f"https://{dnac_ip}/dna/system/api/v1/auth/token"
user = "admin"
password = "Cisco123!"

# HTTP Basic認証を使用してトークンを取得
response = requests.post(
    auth_url, 
    auth=HTTPBasicAuth(user, password), 
    verify=False
)

if response.status_code == 200:
    token = response.json()['Token']
    print(f"Login Successful. Token: {token}")
else:
    print(f"Login Failed. Status Code: {response.status_code}")
```

### 2. GUI での AAA サーバ（ISE）の追加
1.  DNAC GUI の **System > Settings > Authentication Server** に移動します。
2.  **Add** をクリックし、ISE の IP アドレス、共有シークレットを入力します。
3.  RADIUS/TACACS+ プロトコルを選択し、API ユーザーの認証を外部委託するように構成します。

---

## 🔍 検証コマンド

| 目的 | 手法 / コマンド |
| :--- | :--- |
| **API 接続性の確認 (curl)** | <code>curl -k -X POST -u "user:pass" https://[DNAC_IP]/dna/system/api/v1/auth/token</code> |
| **トークンの有効性確認** | 取得したトークンを使い GET <code>/dna/intent/api/v1/network-device</code> を叩き、200 が返るか確認。 |
| **ユーザーロールの確認** | <code>GET /dna/system/api/v1/user</code> (ログインユーザーの詳細情報を取得) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| `401 Unauthorized` | 資格情報の誤り、または Base64 エンコードミス | ユーザー名とパスワードを再確認。API では正しいエンドポイントを使用しているかチェック。 |
| `SSL: CERTIFICATE_VERIFY_FAILED` | 自己署名証明書の検証失敗 | <code>verify=False</code> を追加。 |
| トークンが急に無効になる | 有効期限切れ（デフォルト 1 時間） | トークン取得処理を再実行し、ヘッダーを更新する。 |
| `403 Forbidden` | ユーザーの RBAC ロールが不足 | DNAC GUI でユーザーに正しい Role (NETWORK-ADMIN 等) が付与されているか確認。 |

---

## ⚠ 制限事項

*   **同時セッション数**: 1 ユーザーあたりの同時アクティブトークン数には上限がある場合があります。
*   **外部 AAA 依存**: ISE 等の外部 AAA がダウンしている場合、ローカルユーザー以外は API 認証に失敗します。
*   **バージョン互換性**: API エンドポイント（v1, v2 等）は DNAC のソフトウェアバージョンにより変更される可能性があります。

---

## 🔄 他技術との関連

*   **3.9 API Interaction**: Python スクリプトによる基本的な API 操作能力。
*   **Cisco ISE**: 外部認証ソースとしての統合。
*   **Role-Based Access Control (RBAC)**: 取得したトークンで何ができるかを決定する認可メカニズム。

---

## 🧩 比較表

### Basic Auth vs Token-based Auth (DNAC API)

| 比較項目 | Basic Authentication | Token-based Authentication |
| :--- | :--- | :--- |
| **使用タイミング** | 初回のトークン取得時のみ | 以降のすべての API リクエスト |
| **ヘッダー名** | `Authorization` | `X-Auth-Token` |
| **データ内容** | ユーザー名:パスワード (Base64) | ランダムな英数字文字列 |
| **推奨理由** | シンプルだが、パスワードを毎回送るのは危険 | セッション管理が可能。パスワード露出を最小化。 |

---

## 💡 ベストプラクティス

1.  **トークンの再利用**: リクエストのたびにトークンを取得するのではなく、1 時間の有効期限内は保持して使い回すようにスクリプトを設計します。
2.  **機密情報の保護**: スクリプト内にパスワードをハードコードせず、環境変数や `getpass` モジュールを使用します。
3.  **エラーハンドリング**: 401 や 403 エラーをキャッチして、ユーザーに分かりやすいメッセージを出すようにします。
4.  **最小特権**: API 用のユーザーには、必要最小限のロール（例：読み取り専用なら OBSERVER）を割り当てます。

---

## 📝 ラボ学習・設定サンプル例

Cisco DNA Center (DNAC) の Northbound API における **Authentication（認証）** と **Authorization（認可）** に焦点を当てた、ラボ試験形式の実装シナリオです。

### 1. 基本的な認証トークンの取得
*   **問題**: Python スクリプトを使用して、DNAC (10.1.1.10) から `X-Auth-Token` を取得し、画面に出力せよ。
*   **要件**:
    1.  `requests` ライブラリと `HTTPBasicAuth` を使用すること。
    2.  SSL 証明書の検証をスキップし、警告を無効化すること。
*   **設定例**:
    ```python
    import requests
    from requests.auth import HTTPBasicAuth

    # SSL警告の無効化
    requests.packages.urllib3.disable_warnings()

    dnac_ip = "10.1.1.10"
    url = f"https://{dnac_ip}/dna/system/api/v1/auth/token"
    auth = HTTPBasicAuth("admin", "Cisco123!")

    # POSTリクエストによるトークン発行
    response = requests.post(url, auth=auth, verify=False)

    if response.status_code == 200:
        token = response.json().get('Token')
        print(f"Token acquired: {token}")
    else:
        print(f"Auth Failed: {response.status_code}")
    ```
*   **検証方法**: スクリプトを実行し、`Token acquired: ...` と英数字の長い文字列が表示されることを確認する。

### 2. トークンを使用した認可（Authorization）の確認
*   **問題**: 取得したトークンを使い、ネットワークデバイス一覧を取得する GET リクエストを送信せよ。
*   **要件**: 適切な HTTP ヘッダー (`X-Auth-Token`) を付与し、認可されたセッションであることを証明すること。
*   **設定例**:
    ```python
    # token は Lab 1 で取得済みと仮定
    headers = {
        "X-Auth-Token": token,
        "Content-Type": "application/json"
    }
    device_url = f"https://{dnac_ip}/dna/intent/api/v1/network-device"

    response = requests.get(device_url, headers=headers, verify=False)
    
    if response.status_code == 200:
        print("Authorization Success: Device list retrieved.")
    ```
*   **検証方法**: ステータスコード `200 OK` が返り、デバイス情報（JSON）が取得できているか確認する。

### 3. トークン失効（401 Unauthorized）のハンドリング
*   **問題**: トークンの有効期限切れ（通常 1 時間）により `401` エラーが発生した際、自動的に再ログインを試みるロジックを実装せよ。
*   **設定例**:
    ```python
    def get_devices(token):
        response = requests.get(device_url, headers={"X-Auth-Token": token}, verify=False)
        if response.status_code == 401:
            print("Token expired. Re-authenticating...")
            # 再取得ロジックを呼び出す
            new_token = login_to_dnac() 
            return get_devices(new_token)
        return response.json()
    ```
*   **検証方法**: 意図的に古いトークンを使用して関数を呼び出し、再認証プロセスがトリガーされるか確認する。

### 4. RBAC (ロールベースアクセス制御) による操作制限の検証
*   **問題**: 読み取り専用ロール（OBSERVER等）を持つユーザーが、デバイス削除 API を実行した際に `403 Forbidden` を受けることを確認するスクリプトを作成せよ。
*   **設定例**:
    ```python
    # 低権限ユーザーのトークンで実行
    del_url = f"{device_url}/[device-uuid]"
    res = requests.delete(del_url, headers=headers, verify=False)

    if res.status_code == 403:
        print("Correctly restricted: Role does not have delete permission.")
    ```
*   **検証方法**: 指定したユーザーの権限レベルに基づき、期待通りのエラーコードが返るか確認する。

### 5. 外部 AAA (ISE) 連携による API 認証設定
*   **問題**: DNAC の API 認証に外部 ISE サーバーを使用するように構成せよ。
*   **要件**: DNAC GUI で ISE を認証サーバーとして登録すること。
*   **設定例（GUI手順概要）**:
    1.  DNAC **System > Settings > Authentication Server** に移動する。
    2.  ISE の IP アドレスと Shared Secret を入力して **Add** する。
    3.  Python スクリプト側で ISE に登録された外部ユーザー (`ise_admin`) を使ってトークン取得を試行する。
*   **検証方法**: スクリプトの `auth = HTTPBasicAuth("ise_admin", "password")` でトークンが正常に発行されるか確認する。
---

## ❓ 想定試験問題

1.  **実装**: DNAC API で認証トークンを取得するために使用すべき HTTP メソッドとエンドポイントを答えよ。
    *   **回答**: メソッド: **POST**、エンドポイント: **/dna/system/api/v1/auth/token**。
2.  **トラブルシュート**: 正しいユーザー名/パスワードを使用しているにもかかわらず API が 401 を返す。考えられる原因は？
    *   **回答**: ユーザーが DNAC にローカル登録されておらず、かつ外部 AAA サーバとの連携設定が未完了である。
3.  **コンフィグ読解**: Python で `headers = {'X-Auth-Token': token}` を定義した。これを `requests.get()` のどこに渡すべきか？
    *   **回答**: `headers` 引数。例: `requests.get(url, headers=headers)`。
4.  **Design**: API 経由でネットワークデバイスの設定変更を行う際、セキュリティ上最も望ましい RBAC ロールは何か？
    *   **回答**: **NETWORK-ADMIN**（SUPER-ADMIN よりも権限を絞り、ネットワーク操作に限定するため）。
5.  **Design**: トークンベースの認証において、パスワード認証を毎回行わなくて済むメリットは？
    *   **回答**: パスワードがネットワーク上を流れる回数を減らし、盗聴リスクを低減できること、およびサーバー側の処理負荷を軽減できること。

---

## 🔗 参考リソース

*   **Cisco DNA Center API Reference**: [DevNet - DNAC Intent API](https://developer.cisco.com/docs/dna-center/api/2-2-2/)
*   **Configuration Guide**: [Cisco DNA Center User Guide - Managing Users and Roles](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/dna-center/2-2-2/user-guide/b_cisco_dna_center_ug_2_2_2.html)
*   **Cisco Live (BRKPRO-2012)**: [Introduction to DNA Center REST APIs](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「API の玄関口は `/auth/token`」と暗記してください。ここを通らなければ何も始まりません。
*   **注意点**: ラボ試験では、スクリプトの変数名やファイル名が指定されることがあるため、問題文の要件を細部まで読み飛ばさないように注意してください。
*   **図解**: 
    *   **Input**: User/Pass (Basic Auth)
    *   **Output**: Token (X-Auth-Token)
    *   **Flow**: Client -> Auth API -> Get Token -> Call Intent API with Token.
