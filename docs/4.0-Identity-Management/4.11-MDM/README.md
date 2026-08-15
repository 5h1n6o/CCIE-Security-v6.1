---
layout: default
title: 4.11-MDM
nav_order: 11
parent: 4.0-Identity-Management
---

# 4.11 Integration of MDM with Cisco ISE

Cisco ISE (Identity Services Engine) と **MDM (Mobile Device Management)** または **EMM (Enterprise Mobility Management)** の統合は、スマートフォンやタブレットなどのモバイルデバイス、および管理対象のラップトップに対して、デバイスの「健全性（コンプライアンス）」に基づいた動的なアクセス制御を実現するための重要な技術です。ISE は MDM サーバ（Meraki, Microsoft Intune, Jamf 等）の API と連携し、デバイスが登録されているか、パスコードが設定されているか、脱獄（Jailbreak）されていないかといった情報を取得して認可判定を行います。

---

## 📘 概要

*   **機能概要**: ISE をポリシー決定ポイント (PDP) とし、外部の MDM サーバからデバイス属性（登録状況、コンプライアンス状態等）を API 経由で取得して、ネットワークアクセスの可否や権限を決定する機能です。
*   **利用目的**: 私物端末（BYOD）や会社支給端末が、企業のセキュリティ基準を満たしている場合のみ社内リソースへのアクセスを許可すること。
*   **どのような場面で利用するか**: 
    *   **コンプライアンスチェック**: 「ディスク暗号化が無効な端末」や「古い OS バージョンの端末」のアクセスを制限する。
    *   **セルフオンボーディング**: 未登録端末を検知し、自動的に MDM の登録画面（Enrollment Portal）へリダイレクトする。
    *   **紛失時の対応**: MDM 側で紛失マークが付いた端末のネットワークアクセスを ISE で即座に遮断する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要プロトコル** | **HTTPS (API)** による ISE と MDM サーバ間の対話。 |
| **主要属性** | `MDM:DeviceRegistered`, `MDM:IsCompliant`, `MDM:OSVersion` 等。 |
| **ライセンス要件** | ISE **Advantage** (旧 Plus) または **Premier** ライセンスが必要。 |
| **連携方式** | ポーリング（ISE からの問い合わせ）および CoA（MDM からの通知をトリガーとする再認可）。 |
| **証明書** | ISE と MDM 間の信頼構築のため、MDM 側の CA 証明書のインポートが必要。 |

---

## 🏗 動作原理

ISE は MDM サーバを「アイデンティティソース（属性ソース）」の一つとして扱います。

```text
  [ Mobile Device ]          [ Cisco Switch/WLC ]          [ Cisco ISE ]          [ MDM Server ]
         |                         |                         |                         |
         |--- (1) RADIUS Auth ---->|                         |                         |
         |                         |--- (2) Access-Request ->|                         |
         |                         |                         |--- (3) API Query ------>|
         |                         |                         |    (MAC/UDID check)     |
         |                         |                         |<-- (4) API Response ----|
         |                         |                         |    (Compliant/Reg info) |
         |                         |<-- (5) Access-Accept ---|                         |
         |                         |    (dACL/VLAN/SGT)      |                         |
         |<-- (6) Network Access --|                         |                         |
```


---

## ⚙ 動作シーケンス

1.  **接続開始**: 端末が SSID や有線ポートに接続。NAD（スイッチ/WLC）が ISE へ RADIUS 要求を送信。
2.  **デバイス識別**: ISE は MAC アドレスや証明書情報を基に、対象がモバイルデバイスであることを識別。
3.  **MDM 問い合わせ**: ISE は構成済みの MDM サーバへ API クエリを送り、当該 MAC アドレスの登録状態とコンプライアンス状態を確認。
4.  **認可判定**: 
    *   **未登録の場合**: 認可プロファイルにより MDM Enrollment Portal URL への**リダイレクト**を NAD へ返信。
    *   **登録済み・非準拠の場合**: 制限されたアクセス（Quarantine VLAN/ACL）を割り当て。
    *   **登録済み・準拠の場合**: 社内リソースへのフルアクセスを許可。
5.  **状態変更 (CoA)**: 端末が MDM への登録を完了したり、準拠状態が変化したりすると、ISE は NAD に対して **CoA (Change of Authorization)** を発行し、動的にアクセス権限を更新。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **MDM サーバの登録**: `Administration > Network Resources > External MDM` で Hostname、Port (443)、API Credentials を正しく入力する。
*   **信頼チェーンの構成**: MDM サーバの HTTPS 証明書を署名したルート CA を、ISE の **Trusted Certificates** に追加し、「Trust for authentication within ISE」にチェックを入れる。
*   **リダイレクト ACL の設計**: NAD 側で、MDM サーバへの通信は `permit`（＝リダイレクト除外）し、それ以外の HTTP 通信を `deny`（＝リダイレクト対象）にする ACL の作成。
*   **Authorization Policy の条件設定**: `MDM:IsCompliant EQUALS True` や `MDM:DeviceRegistered EQUALS True` を And 条件で組み合わせる。
*   **トラブルシュート（Live Logs）**: 認可失敗時、Live Log の `Steps` を確認し、MDM クエリがタイムアウトしていないか、または属性が正しく取得できているかを特定する。

---

## 🛠 設定方法

### 1. Cisco ISE：MDM サーバの追加 (GUI)
1.  **Administration > Network Resources > External MDM** に移動。
2.  **Add** をクリック。
3.  **Server Name**, **Server IP/Hostname** を入力。
4.  **Authentication Type** (通常は Basic または OAuth) を選択し、MDM 側の API アカウント情報を入力。
5.  **Test Connection** を実行し、疎通を確認。

### 2. Cisco ISE：認可ポリシーの構成
*   **Rule 1 (Compliant)**: 
    *   Condition: `MDM:IsCompliant Equals True`
    *   Result: `PermitAccess`
*   **Rule 2 (Not Registered)**:
    *   Condition: `MDM:DeviceRegistered Equals False`
    *   Result: `MDM_Redirect_Profile` (Enrollment URL を含む)。

---

## 🔍 検証コマンド

| 目的 | 手法 / コマンド |
| :--- | :--- |
| **RADIUS 認証ログの確認** | **Operations > RADIUS > Live Logs** で適用された認可ポリシーを確認。 |
| **MDM 属性取得の確認** | Live Logs の **Details** を開き、`MDM` セクションに属性値があるか確認。 |
| **MDM 連携ステータス** | **Administration > Network Resources > External MDM** のステータスランプ。 |
| **NAD での適用 ACL 確認** | <code>show access-session interface [int] details</code>。 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| **MDM 属性が Unknown になる** | ISE と MDM 間の API 疎通不可 | ISE CLI から MDM サーバへの <code>ping</code> および 443 ポートの疎通を確認。 |
| **証明書エラーで連携不可** | MDM サーバの証明書が未信頼 | MDM のルート CA を ISE の **Trusted Certificates** にインポートする。 |
| **リダイレクトがループする** | 認可条件の不整合 | `DeviceRegistered = True` になった後のポリシーがリダイレクトルールより上位にあるか確認。 |
| **CoA が動作しない** | NAD 側の構成不足 | <code>aaa server radius dynamic-author</code> の設定と共有キーを確認。 |

---

## ⚠ 制限事項

*   **ポーリング間隔**: ISE が MDM へ問い合わせる間隔（デフォルト 24 時間等）により、コンプライアンス情報の反映にタイムラグが生じる場合がある。
*   **API の制限**: 一部の MDM ベンダーは API コールの回数制限を設けており、大規模環境では注意が必要。
*   **MAC アドレスの不一致**: モバイル OS の「プライベート Wi-Fi アドレス」機能により、MDM 登録時の MAC と接続時の MAC が異なると連携に失敗する。

---

## 🔄 他技術との関連

*   **4.6 BYOD Flow**: オンボーディングプロセスの一環として MDM 登録を組み込む。
*   **4.9 Posture**: エージェント（AnyConnect）による詳細なポスチャと、MDM による簡易的なポスチャを組み合わせて使用することが可能。
*   **2.6 TrustSec (SGT)**: 準拠デバイスに特定の SGT を割り当て、マイクロセグメンテーションを行う。

---

## 🧩 比較表

### ISE Posture vs MDM Integration

| 特徴 | ISE Posture (AnyConnect) | MDM Integration |
| :--- | :--- | :--- |
| **エージェント** | AnyConnect Posture モジュール必須 | MDM エージェント (OS 標準等) |
| **詳細度** | **非常に高い** (ファイル、レジストリ) | 中程度 (パスコード、脱獄、OS 版) |
| **制御対象** | Windows, macOS, Linux | **iOS, Android**, macOS, Windows |
| **推奨用途** | 企業管理 PC の詳細検疫 | **BYOD モバイル端末の管理** |

---

## 💡 ベストプラクティス

1.  **証明書ベースの認証**: MDM 経由でデバイス証明書を配布し、EAP-TLS と MDM 属性チェックを組み合わせるのが最もセキュア。
2.  **Grace Period の設定**: 非準拠になった直後に遮断せず、修正のための猶予期間（Grace Period）を MDM 側で設ける。
3.  **複数 MDM の利用**: 部署や OS ごとに異なる MDM サーバを使用する場合、ISE の **Authorization Policy** で適切に振り分ける。

---

## 📝 ラボ学習・設定サンプル例

### 1. MDM サーバの登録 (Cisco Meraki 例)
*   **要件**: Meraki MDM を API キーを使用して ISE に登録せよ。
*   **設定**: `External MDM > Add > Meraki`、API Key を入力。

### 2. 未登録デバイスのリダイレクト
*   **要件**: MDM 未登録デバイスを Meraki の Enrollment URL へリダイレクトせよ。
*   **プロファイル**: Redirect URL に `https://m.meraki.com` を指定。

### 3. 脱獄（Jailbreak）端末の拒否
*   **要件**: `MDM:IsJailbroken EQUALS True` の端末は即座に拒否せよ。

### 4. OS バージョンによる制限
*   **要件**: iOS 14 未満の端末はインターネットアクセスのみ（Guest ACL）に制限せよ。

### 5. パスコード未設定端末の検疫
*   **要件**: MDM で `Passcode Compliant = False` の場合、隔離 VLAN に配置せよ。

### 6. CoA による状態更新テスト
*   **操作**: 端末を MDM 登録後、自動的にフルアクセスへ遷移することを確認。

### 7. リダイレクト ACL の作成 (WLC)
*   **設定**: `config access-list add-rule REDIRECT 1 0.0.0.0 0.0.0.0 any [MDM_IP] 255.255.255.255 any permit`.

### 8. 証明書認証との組み合わせ
*   **要件**: EAP-TLS 認証成功、かつ MDM 準拠の場合のみフルアクセスを許可せよ。

### 9. 紛失（Lost）モード端末の遮断
*   **要件**: MDM で `Device Lost = True` とマークされた端末を RADIUS Reject せよ。

### 10. MDM 属性の取得間隔調整
*   **設定**: 特定のポリシーセットで `MDM Query` の頻度を 1 時間に短縮せよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: MDM 連携を設定したが、Live Log に MDM 属性が全く表示されない。ISE 設定画面での確認ポイントは？
    *   **回答**: **Test Connection** の実行、および MDM サーバの **CA 証明書** が ISE の Trusted Certificates にインポートされているか確認。
2.  **Design**: モバイルデバイスの MAC アドレスがランダム化される環境で、ISE と MDM を確実に紐付けるために推奨される識別子は？
    *   **回答**: **UDID** (Unique Device Identifier) または証明書内の **GUID**。
3.  **コンフィグ読解**: `MDM:IsCompliant EQUALS Unknown` という条件が意味することは？
    *   **回答**: デバイスは MDM に登録されているが、MDM サーバ側でまだ**コンプライアンス評価が完了していない**状態。
4.  **Design**: MDM 連携において、ISE が MDM サーバに問い合わせるタイミングは？
    *   **回答**: 通常は **RADIUS 認証の認可フェーズ中**、および設定された定期的なポーリング時。
5.  **実装**: NAD 側で MDM リダイレクトを正常に機能させるために必須の AAA 構成は？
    *   **回答**: **CoA (RFC 5176)** の受信設定 (`aaa server radius dynamic-author`)。

---

## 🔗 参考リソース

*   **Configuration Guide**: [Cisco ISE 3.1 - Configure MDM Integration](https://www.cisco.com/c/ja_jp/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31.html)
*   **Design Guide**: [ISE BYOD and MDM Deployment Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/ise-deployment-guide.html)
*   **Technical Note**: [Troubleshoot ISE and MDM Server Integration Issues](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/119143-configure-ise-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「ISE はゲートキーパー、MDM は健康診断結果」とイメージしてください。
*   **注意点**: ラボ試験では、**リダイレクト ACL 名の不一致**（ISE 側と NAD 側）が原因でリダイレクトが失敗するトラップに注意してください。
*   **図解**: 
    - 未登録 -> Redirect to MDM
    - 登録済 -> Check Compliance API
    - 準拠 -> Permit Access
