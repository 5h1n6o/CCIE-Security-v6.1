---
layout: default
title: 4.9-Posture
nav_order: 9
parent: 4.0-Identity-Management
---

# 4.9 Posture assessment with Cisco ISE

Cisco ISE (Identity Services Engine) における **Posture Assessment（ポスチャ評価）** は、ネットワークに接続しようとするエンドポイントが、組織のセキュリティポリシー（最新のウイルス対策ソフト、OS パッチ、特定のファイルやレジストリ設定など）に準拠しているかどうかを確認するプロセスです。準拠していない端末は「検疫（Quarantine）」状態に置かれ、必要な修正（Remediation）を行った後で初めてフルアクセスが許可されます。

---

## 📘 概要

*   **機能概要**: 接続端末の状態を詳細にスキャンし、あらかじめ定義された「健全性」の基準を満たしているか判定する機能です。
*   **利用目的**: ウイルス感染の防止、脆弱性のある端末の排除、およびコンプライアンス（法令順守）の維持。
*   **どのような場面で利用するか**:
    *   **リモートアクセス VPN**: 自宅 PC が最新のパッチを適用しているか確認。
    *   **キャンパス有線/無線**: 持ち込み PC（BYOD）が企業の指定するウイルス対策ソフトを有効にしているか強制。
    *   **コンプライアンス監査**: 特定のセキュリティ設定が全端末で有効であることを保証。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **ポスチャエージェント** | AnyConnect Posture Module, Stealthy Agent, Temporal Agent（一時的）。 |
| **ポスチャ状態** | **Unknown** (未評価), **Compliant** (準拠), **Non-Compliant** (非準拠)。 |
| **評価要素 (Check)** | AV, AS, OS Patch, File, Registry, Service, Application, Dictionary。 |
| **ライセンス要件** | **ISE Premier (旧 Apex)** ライセンスが必要。 |
| **Persona** | Policy Service Node (PSN) が評価リクエストを処理する。 |
| **強制 (Enforcement)** | NAD (Switch/WLC/ASA) での ACL 変更や VLAN 変更。 |

---

## 🏗 動作原理

ポスチャ評価は、端末内のエージェントと ISE 間の対話によって成立します。

```text
Endpoint (Supplicant)       NAD (Switch/ASA)          Cisco ISE (PSN)
      |                         |                          |
      |--- (1) RADIUS Auth ---->|                          |
      |                         |--- (2) Access-Request -->|
      |                         |                          |
      |                         |<-- (3) Access-Accept ----|
      |                         |    (Redirect URL/ACL)    |
      |<-- (4) Quarantined -----|                          |
      |        Access           |                          |
      |                         |                          |
      |--- (5) Agent Discovery --------------------------->|
      |         (HTTP/HTTPS)    |                          |
      |                         |                          |
      |--- (6) Posture Report ---------------------------->|
      |                         |                          |
      |<-- (7) Policy Result ------------------------------|
      |        (Compliant!)     |                          |
      |                         |<-- (8) RADIUS CoA -------|
      |                         |    (Change of Auth)      |
      |                         |                          |
      |                         |--- (9) Apply Full Access-|
      |<-- (10) Full Network ---|                          |
               Access
```

---

## ⚙ 動作シーケンス

1.  **初期認証**: 802.1X または MAB で認証。ISE はポスチャ状態が `Unknown` であるため、リダイレクト URL と限定的な ACL を含む認可プロファイルを返します。
2.  **エージェントの展開**: 必要に応じて AnyConnect Posture Module がインストールされます（Client Provisioning）。
3.  **Discovery**: エージェントが ISE (PSN) を探します。リダイレクト ACL を利用した HTTP インターセプトや、静的な Discovery Host 設定が使用されます。
4.  **評価 (Assessment)**: エージェントが端末をスキャンし、ISE のポリシーに基づくレポートを送信します。
5.  **修正 (Remediation)**: 非準拠の場合、ユーザーにパッチ適用やソフト起動を促すメッセージを表示します。
6.  **CoA (Change of Authorization)**: 準拠 (`Compliant`) になると、ISE から NAD へ CoA パケットが送られ、セッションが再認可されます。
7.  **アクセス許可**: 端末にフルアクセス ACL または VLAN が適用されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **リダイレクト ACL の作成**: ASA またはスイッチで、DNS, DHCP, ISE 通信を `deny`（リダイレクト除外）し、HTTP/HTTPS を `permit`（リダイレクト対象）にする ACL の作成は必須です。
*   **Posture Condition の詳細設定**: 「特定のレジストリ値が存在すること」や「特定のサービスが実行中であること」など、細かい条件設定が問われます。
*   **Client Provisioning Policy**: どの OS に対してどの AnyConnect パッケージとプロファイルを配布するか、正確な順序で設定する必要があります。
*   **CoA のトラブルシュート**: `aaa server radius dynamic-author` が NAD 側で抜けていると、ポスチャが「成功」しても通信が制限されたままになる点に注意してください。
*   **Discovery 障害**: 端末が ISE に到達できない場合、エージェントログを確認して Discovery ホスト設定や証明書の信頼性を確認する能力が求められます。

---

## 🛠 設定方法

### 1. ポスチャポリシーの構成フロー (ISE GUI)
1.  **Conditions**: `Policy > Policy Elements > Conditions > Posture` で「AV 有効」などの条件を作成。
2.  **Remediation**: 失敗時のアクション（メッセージ表示、リンク誘導）を定義。
3.  **Requirements**: `Conditions` + `Remediation` を組み合わせて要件を作成。
4.  **Posture Policy**: `Policy > Posture` で、特定のユーザーグループ/OS に対し作成した `Requirements` を割り当てる。

### 2. スイッチ：リダイレクト ACL の設定例
```bash
ip access-list extended REDIRECT-POSTURE
 deny udp any any eq domain
 deny udp any any eq bootps
 deny ip any host 10.1.1.100  ! ISE PSN IP
 permit tcp any any eq 80     ! HTTP Redirect
 permit tcp any any eq 443    ! HTTPS Redirect
```

---

## 🔍 検証コマンド

| 目的 | コマンド / 手法 |
| :--- | :--- |
| **ポスチャ詳細の確認** | **Operations > Posture > Posture Assessment** (ISE GUI) |
| **認証ライブログ** | **Operations > RADIUS > Live Logs** でポスチャ状態を確認。 |
| **セッション状態 (スイッチ)** | <code>show access-session interface [int] details</code> |
| **適用された ACL の確認** | <code>show ip access-lists interface [int]</code> |
| **ポスチャログのデバッグ** | AnyConnect UI の **Diagnostics > Message History** |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| エージェントが ISE を見つけられない | リダイレクト ACL の誤り、DNS 解決不可 | `ping [ISE_FQDN]` を確認し、リダイレクト ACL で ISE 通信を除外しているか確認。 |
| 準拠しているのにアクセスが制限されたまま | CoA パケットの遮断、NAD の設定不足 | スイッチの <code>aaa server radius dynamic-author</code> を確認。 |
| ポスチャ評価が開始されない | Client Provisioning の失敗 | **Policy > Client Provisioning** で OS とパッケージが正しく紐付いているか確認。 |
| 証明書警告で停止する | ISE ポータル証明書が未信頼 | ISE のルート CA 証明書を端末の信頼されたストアへインポートする。 |

---

## ⚠ 制限事項

*   **OS サポート**: Windows と macOS はフルサポートされますが、Linux はエージェントの機能が限定される場合があります。
*   **モバイルデバイス**: iOS/Android は AnyConnect ポスチャを直接実行できず、MDM 連携が必要になるケースが一般的です。
*   **スキャンの限界**: カーネルレベルの深いスキャンを行うには、管理者権限が必要になります。

---

## 🔄 他技術との関連

*   **4.8 AnyConnect Provisioning**: ポスチャエージェントを端末に送り込むための必須機能。
*   **4.2 Network Access AAA**: ポスチャ評価の前後で RADIUS 属性（ACL/SGT）を変更する。
*   **2.6 Microsegmentation**: ポスチャ非準拠の端末に `Quarantined` タグ (SGT) を付与して論理分離。

---

## 🧩 比較表

### Full AnyConnect Posture vs Temporal Agent

| 特徴 | AnyConnect Posture Module | Temporal Agent |
| :--- | :--- | :--- |
| **永続性** | 常駐プログラムとしてインストール | 評価時のみ実行、終了後に削除 |
| **詳細度** | 高い（複雑な条件、修復が可能） | 中程度（ファイル、プロセスの存在確認） |
| **ユーザー体験** | 初回のプロビジョニングが必要 | インストール不要で実行が早い |
| **推奨用途** | **企業管理 PC (Managed)** | ゲスト、コントラクター (BYOD) |

---

## 💡 ベストプラクティス

1.  **段階的な導入**: 最初は `Audit Only` モード（非準拠でも拒否しない）で導入し、端末の実態を把握してから強制（Block）へ移行します。
2.  **Discovery Host の利用**: リダイレクトだけに頼らず、AnyConnect プロファイルに複数の ISE PSN を `Discovery Host` として登録し、冗長性を高めます。
3.  **メッセージの明確化**: 修正アクションが発生した際、ユーザーが何をすべきか（「このボタンを押して更新してください」など）を明確に記述します。

---

## 📝 ラボ学習・設定サンプル例

### 1. ウイルス対策ソフトの有効化チェック
*   **要件**: Windows 10 端末で Windows Defender が有効であることを確認せよ。
*   **設定**: Anti-Malware Condition で `Windows Defender` を選択し、`Monitor` ではなく `Enforce` に設定。

### 2. 特定ファイルの存在確認
*   **要件**: `C:\CCIE\lab_policy.txt` というファイルが存在しない端末はアクセスを拒否せよ。
*   **設定**: File Condition でパスを指定し、`File Existence` をチェック。

### 3. レジストリキーの検証
*   **問題**: 特定のレジストリ値が `1` であることを確認せよ。

### 4. リダイレクト ACL の実装 (VPN)
*   **要件**: ASA 上でポスチャ前のトラフィックを ISE ポータルへリダイレクトせよ。

### 5. Client Provisioning の優先順位付け
*   **要件**: IT 部門（AD グループ）には最新版、営業部門には安定版のパッケージを配布せよ。

### 6. カスタムスクリプトによる修復
*   **要件**: 非準拠の場合、特定の PowerShell スクリプトを実行して設定を修正せよ。

### 7. Grace Period（猶予期間）の設定
*   **問題**: OS パッチ未適用の端末に、3 日間の猶予期間を与えてフルアクセスを維持させよ。

### 8. スイッチでの CoA 受信設定
*   **設定**: `aaa server radius dynamic-author` に ISE の IP と共有シークレットを設定。

### 9. ポスチャ評価前の限定通信 ACL
*   **要件**: 評価中も会社の WSUS サーバーへの通信のみ許可せよ。

### 10. Agentless Posture の構成
*   **要件**: エージェントをインストールせず、ブラウザ経由で一時的なチェックを実行せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: ISE の認可プロファイルで `posture:status = Compliant` という条件がある。このセッションが最初に確立された直後のポスチャ状態は？
    *   **回答**: **`Unknown`**。評価が完了するまでこの認可ポリシーには合致しません。
2.  **トラブルシュート**: 端末に AnyConnect は入っているが、評価が始まらない。`debug radius` でリダイレクト URL が送信されていることは確認済み。次に確認すべきは？
    *   **回答**: 端末ブラウザでの **HTTP リダイレクトの動作**、およびポスチャエージェントが使用する **Discovery ポート (TCP 80/443/8443)** の到達性。
3.  **Design**: ポスチャ環境における PSN の分散配置のメリットは？
    *   **回答**: リダイレクトトラフィックの局所化と、エージェントからの評価レポート処理の負荷分散。
4.  **Design**: ポスチャ評価の結果、`Non-Compliant` になった場合の NAD での一般的な動作は？
    *   **回答**: 認可プロファイルにより、**制限された ACL (Quarantine ACL)** が適用され、インターネットやパッチサーバ以外のアクセスを遮断する。
5.  **実装**: 特定のレジストリチェックの結果によって異なる VLAN に割り当てたい。どのように構成すべきか？
    *   **回答**: ポスチャ結果を **Identity Group** の変更に紐付けるか、認可ポリシーの条件としてポスチャレポート内の特定の属性を使用する。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [ポスチャ サービスの設定](https://www.cisco.com/c/ja_jp/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31.html)
*   **Cisco Live (BRKSEC-2024)**: [ISE Posture Deep Dive](https://www.ciscolive.com/)
*   **CVD**: [AnyConnect Secure Mobility Client with ISE Posture Design Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/ise-deployment-guide.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: ポスチャは「後出しジャンケン」のようなものです。最初は仮のアクセス（Redirect）を与え、端末が正しい情報（Report）を出してきたら、本当のアクセス（Full）を返します。
*   **図解**: 
    - **Policy Set**: 全体の枠組み。
    - **Posture Policy**: 端末の中身に対する詳細なチェックリスト。
*   **注意点**: ラボ試験では、**AnyConnect ポスチャプロファイル内の ISE FQDN** が間違っているだけで Discovery が失敗し、すべての評価フローが止まるため、プロファイル設定は極めて重要です。
