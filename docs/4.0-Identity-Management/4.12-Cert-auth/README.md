---
layout: default
title: 4.12-Cert-auth
nav_order: 12
parent: 4.0-Identity-Management
---

# 4.12 Certification-based authentication using Cisco ISE

Cisco ISE (Identity Services Engine) における証明書ベースの認証（Certificate-based authentication）は、主に **EAP-TLS** プロトコルを使用して行われます。これは、従来のユーザー名/パスワードによる認証に代わり、デジタル証明書（X.509）を用いてデバイスやユーザーの ID を検証する、最もセキュアな認証手法です。CCIE Security v6.1 において、この項目は PKI 階層の理解、ISE への信頼構築、および属性マッピングの柔軟な構成能力を求めています。

---

## 📘 概要

*   **機能概要**: 接続を試みるサプリカント（端末）が提示するクライアント証明書を ISE が検証し、認証を行う機能です。
*   **利用目的**: パスワード漏洩やフィッシングのリスクを排除し、組織が発行した正当な証明書を持つデバイスのみを許可すること。
*   **どのような場面で利用するか**:
    *   **企業管理 PC**: AD のグループポリシー経由で配布された証明書による自動認証。
    *   **モバイルデバイス（BYOD/MDM）**: オンボーディング時に発行された証明書を用いた接続。
    *   **マシン認証**: ユーザーがログインする前の、デバイス単体でのネットワーク接続。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要プロトコル** | **EAP-TLS**, **TEAP** (Tunnel Extensible Authentication Protocol)。 |
| **信頼基盤** | **PKI (Public Key Infrastructure)**。ISE がルート/中間 CA を信頼している必要がある。 |
| **検証要素** | 有効期限、署名の妥当性、および失効確認（**CRL** / **OCSP**）。 |
| **ID マッピング** | 証明書の特定のフィールド（Subject, SAN 等）をユーザー名として抽出する。 |
| **二重検証** | **Binary Comparison**。AD 内のユーザーオブジェクトと証明書を 1 対 1 で照合。 |
| **ホストモード** | 証明書はマシン（Computer）とユーザー（User）の両方で発行・利用が可能。 |

---

## 🏗 動作原理

証明書認証は、ISE とクライアント間での相互信頼に基づくハンドシェイクで成立します。

```text
  [ Supplicant ]           [ Authenticator ]           [ Cisco ISE ]
        |                         |                          |
        |--- (1) EAP-TLS Start -->|                          |
        |                         |--- (2) RADIUS Request -->|
        |                         |        (EAP-TLS)         |
        |<-- (3) Server Hello ----|                          |
        |    (ISE Certificate)    |                          |
        |                         |                          |
        |--- (4) Client Hello --->|                          |
        |    (Client Certificate) |                          |
        |                         |--- (5) Cert Validation ->| (Verify Trust & Revocation)
        |                         |                          |
        |<-- (6) EAP-Success -----|                          |
        |                         |<-- (7) Access-Accept ----|
```

---

## ⚙ 動作シーケンス

1.  **サーバー証明書の提示**: ISE は自身の証明書をクライアントに送り、クライアントは ISE が信頼できるサーバーか確認します。
2.  **クライアント証明書の提示**: クライアントは自身の証明書を ISE に送り、ISE はその証明書が「信頼された CA」によって署名されているか検証します。
3.  **証明書失効チェック**: ISE は CRL (Certificate Revocation List) または OCSP を用いて、証明書が失効していないか確認します。
4.  **アイデンティティ抽出**: ISE は **Certificate Authentication Profile (CAP)** に基づき、CN や SAN から ID を抽出します。
5.  **認可ポリシー適用**: 抽出された ID に基づき、AD グループや SGT などの認可結果が決定されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Trusted Certificates のインポート**: 外部 CA (Microsoft CA 等) のルート証明書を ISE の **Trusted Certificates** にインポートし、「Trust for client authentication」を有効にする手順は必須です。
*   **Certificate Authentication Profile (CAP)**: 証明書のどのフィールドをユーザー名として扱うかの設定が問われます。例：`Subject Alternative Name - OTHER NAME`。
*   **Binary Comparison**: 高度なセキュリティ要件として、証明書が正当であっても、AD 上で該当ユーザーが「無効」になっていれば拒否する設定（Binary Comparison）の実装が求められます。
*   **Allowed Protocols**: ポリシーセットで使用する Allowed Protocols で **EAP-TLS** がチェックされているか確認が必要です。
*   **OCSP/CRL の到達性**: ラボ環境内の ISE が CA サーバーの失効リスト（HTTP 等）に到達できるか、ACL 設定を含めて確認する能力が必要です。

---

## 🛠 設定方法

### 1. 信頼された CA の登録 (GUI)
1.  **Administration > System > Certificates > Trusted Certificates** に移動。
2.  **Import** をクリックし、CA の証明書ファイルを選択。
3.  **Trust for client authentication and Syslog** にチェックを入れる。

### 2. 証明書認証プロファイル (CAP) の作成
1.  **Administration > Identity Management > External Identity Sources > Certificate Authentication Profile** へ移動。
2.  **Add** をクリックし、名前を定義（例：`CERT_PROFILE`）。
3.  **Identity Store**: ID 情報を保持する場所（通常は AD または Internal）を選択。
4.  **Principal Identity Store**: `Subject - Common Name` など、ID を抽出する場所を指定。

### 3. ポリシーセットでの適用
1.  **Policy > Policy Sets** で、認証ポリシー（Authentication Policy）の `Use` 列に作成した `CERT_PROFILE` を指定。

---

## 🔍 検証コマンド

| 目的 | コマンド / 手法 |
| :--- | :--- |
| **証明書詳細の確認 (NAD)** | <code>show crypto ca certificates</code> |
| **ISE 認証ログの確認** | **Operations > RADIUS > Live Logs** |
| **証明書フィールド抽出の確認** | Live Logs の詳細画面で `IdentitySelection` ステップを確認。 |
| **AD 連携状態の確認** | <code>show application status ise</code> (AD コネクタの状態) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 認証失敗 (12514 EAP-TLS failed) | 信頼チェーンの欠如 | ルート CA が ISE の **Trusted Certificates** にあるか確認。 |
| 証明書失効チェックエラー | CRL/OCSP サーバーへ到達不可 | DNS 解決と Firewall ポート (通常 TCP 80) を確認。 |
| ID 抽出エラー | CAP のフィールド指定ミス | 証明書の Subject/SAN フィールドと CAP 設定が一致しているか確認。 |
| ユーザーが見つからない | Binary Comparison の不一致 | AD 上のユーザーオブジェクトに、提示された証明書が紐付いているか確認。 |

---

## ⚠ 制限事項

*   **時刻同期**: クライアントと ISE の時刻が大きくずれていると、有効期限内であっても証明書検証に失敗します。
*   **証明書チェーンの深さ**: 中間 CA が多すぎる場合、RADIUS パケットのフラグメンテーションが発生し、一部のネットワーク機器で認証が失敗することがあります。
*   **iOS の挙動**: iOS デバイスなどでは、信頼されていない証明書（自己署名等）を使用すると、プロファイルがインストールできない場合があります。

---

## 🔄 他技術との関連

*   **4.6 BYOD Flow**: オンボーディング後に、ISE CA から発行された証明書を用いて EAP-TLS 接続を行う。
*   **4.11 MDM Integration**: MDM が配布した証明書を ISE が検証し、コンプライアンス状態と紐付ける。
*   **AnyConnect VPN**: リモートアクセス時にユーザー証明書を必須とする構成。

---

## 🧩 比較表

### 証明書認証 (EAP-TLS) vs パスワード認証 (PEAP-MSCHAPv2)

| 特徴 | EAP-TLS (証明書) | PEAP (パスワード) |
| :--- | :--- | :--- |
| **セキュリティ強度** | **極めて高い** | 中程度（総当たりに弱い） |
| **ユーザー体験** | **透過的**（ログイン操作不要） | ユーザー名/パスワード入力が必要 |
| **複雑さ** | 高（PKI の管理が必要） | 低（既存の AD で容易） |
| **フィッシング耐性** | **あり** | なし |

---

## 💡 ベストプラクティス

1.  **OCSP の優先**: CRL はファイルサイズが大きくなり更新に時間がかかるため、リアルタイム性の高い OCSP の利用を推奨します。
2.  **専用の CAP**: ユーザー用とマシン（コンピュータ）用で異なる証明書フィールドを使用する場合、個別の CAP を作成して使い分けます。
3.  **信頼チェーンの最小化**: パフォーマンスと安定性のため、CA 階層は 2〜3 段階に留めるのが一般的です。
4.  **SAN の利用**: Subject フィールド（CN）は重複する可能性があるため、メールアドレスなどの一意な値を含む **Subject Alternative Name (SAN)** を ID ソースに選ぶのが安全です。

---

## 📝 ラボ学習・設定サンプル例

### 1. ルート CA 証明書のインポート
*   **要件**: 外部 Root CA を ISE に信頼させ、クライアント認証に使用せよ。
*   **手順**: Administration > Trusted Certificates > Import.

### 2. マシン認証用 CAP の構成
*   **要件**: 証明書の `Common Name (CN)` フィールドからマシン名を取得せよ。
*   **設定**: CAP Identity Source = `Subject - Common Name`.

### 3. EAP-TLS ポリシーセットの構築
*   **要件**: `Domain Users` かつ証明書認証の場合のみアクセスを許可せよ。

### 4. Binary Comparison の有効化
*   **問題**: 証明書の内容を AD の属性と照合し、AD に存在しない場合は拒否せよ。

### 5. SAN:UPN からの ID 抽出
*   **要件**: 証明書の `SAN` に含まれる `User Principal Name (UPN)` を使用して認証せよ。

### 6. 失効確認 (OCSP) の設定
*   **操作**: ISE に OCSP サーバーの URL を登録し、証明書の有効性をリアルタイム確認せよ。

### 7. Allowed Protocols のカスタマイズ
*   **要件**: セキュリティ強化のため、認証プロトコルを EAP-TLS のみに制限せよ。

### 8. 証明書ベースの VPN プロビジョニング
*   **要件**: AnyConnect 接続にユーザー証明書を要求するように ASA を構成せよ。

### 9. 信頼チェーンのデバッグ
*   **課題**: 中間 CA 証明書が欠落している環境での認証失敗を Live Log で特定せよ。

### 10. 有効期限通知の構成
*   **要件**: ISE の管理証明書が期限切れになる 30 日前にアラートを出すように設定せよ。

---

## ❓ 想定試験問題

1.  **Design**: EAP-TLS 認証において、ISE がサプリカントの証明書を信頼するために最低限必要な設定は？
    *   **回答**: 発行元 CA の証明書を ISE の **Trusted Certificates** ストアにインポートし、**Client Authentication** 用に信頼する設定を有効にする。
2.  **トラブルシュート**: Live Log に `22045 Root CA not found` と表示されている。原因は？
    *   **回答**: クライアント証明書の署名に使われた **Root CA または中間 CA** が ISE に登録されていない。
3.  **コンフィグ読解**: CAP 設定で `Identify Store = Active Directory` となっている場合、ISE は認証後に何を行うか？
    *   **回答**: 抽出された ID を用いて **AD へのクエリを行い**、グループ所属情報などの属性を取得する。
4.  **実装**: ユーザーが Windows ログイン前にネットワークに接続できるようにするための証明書認証の種類は？
    *   **回答**: **Machine Authentication** (Computer Certificate)。
5.  **Design**: 証明書の失効確認において、帯域消費を抑えつつ最新の状態を確認する方法は？
    *   **回答**: **OCSP (Online Certificate Status Protocol)** の利用。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [証明書ベース認証の構成](https://www.cisco.com/c/ja_jp/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31.html)
*   **Configuration Example**: [AnyConnect EAP-TLS Authentication with ISE](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/116143-configure-ise-00.html)
*   **Cisco Live (BRKSEC-2041)**: [Deep Dive into ISE Certificates and PKI](https://www.ciscolive.com/)
*   **Technical Note**: [Understanding EAP-TLS and ISE Certificate Validation](https://community.cisco.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「証明書がある＝許可」ではなく、「信頼された CA が出した有効な証明書があり、かつ ID ストアでその ID が有効であること」が証明書認証の完成形です。
*   **注意点**: ラボ試験では、**ISE のシステム時刻（NTP）** が狂っていると、証明書が「未来のもの」や「期限切れ」と判断されるため、必ず `show ntp` で同期を確認してください。
