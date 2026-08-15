---
layout: default
title: 4.13.b-MAR
nav_order: 2
parent: 4.13-Auth-methods
grand_parent: 4.0-Identity-Management
---

# 4.13.b MAR (Machine Access Restriction)

**MAR (Machine Access Restriction)** は、Cisco ISE における認証コンポーネントの一つで、ユーザーがネットワークにアクセスを試みた際、その**デバイス自体が事前に正常な「マシン認証（コンピュータ認証）」を完了しているかどうか**をチェックし、アクセスの可否を決定する機能です,。これは主に、個人所有のデバイス（BYOD）による不正なドメイン参加ユーザーのログインを防止し、会社支給の管理端末のみを許可するために使用されます。

---

## 📘 概要

*   **機能概要**: ISE のランタイムキャッシュを使用して、特定の MAC アドレス（エンドポイント）が Active Directory (AD) に対するマシン認証に成功した状態にあるかを追跡し、その後のユーザー認証と紐付ける機能です。
*   **利用目的**: 「正しいユーザー ID」を持っていても、「許可されていないデバイス」からの接続を拒否することで、セキュリティを強化します。
*   **どのような場面で利用するか**:
    *   **PEAP (MS-CHAPv2) 環境**: Windows ログオン前にマシン認証を行い、ログオン後にユーザー認証を行うフローにおいて、両方の成功を条件とする場合。
    *   **BYOD 制限**: 社員が自宅の PC を持ち込み、自分の AD アカウントでログインしようとするのを防ぐ。
    *   **コンプライアンス維持**: マシン認証が成功している（＝ドメインに参加している）端末のみに社内リソースへのフルアクセスを許可する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **動作基盤** | Cisco ISE の **Runtime Cache**。マシン認証の成功フラグを一定期間保持する。 |
| **主要属性** | `Network Access:WasMachineAuthenticated`。 |
| **デフォルトのキャッシュ期間** | 24時間（設定により変更可能）。 |
| **認証プロトコル** | 主に PEAP-MS-CHAPv2。TEAP (EAP Chaining) の普及により補完的な位置付けへ。 |
| **依存関係** | Active Directory 統合が必須。マシン認証は AD に対して行われる必要がある。 |
| **制限事項** | キャッシュベースであるため、ポータビリティやセッション断による不整合が起きる可能性がある。 |

---

## 🏗 動作原理

MAR は「ステートフル」な動作を ISE 内部で行います。

```text
[ Endpoint ]          [ Authenticator ]          [ Cisco ISE ]          [ Active Directory ]
      |                      |                         |                         |
      |-- (1) Machine Auth ->|                         |                         |
      |   (Host/PC1)         |--- (2) RADIUS Req ----->|                         |
      |                      |                         |--- (3) Auth Check ----->|
      |                      |                         |<-- (4) Success ---------|
      |                      |<-- (5) Access-Accept ---|                         |
      |                      |                         | [ ISE Cache: PC1 = OK ] |
      |                      |                         |                         |
      |-- (6) User Login --->|                         |                         |
      |   (User: Alice)      |--- (7) RADIUS Req ----->|                         |
      |                      |                         |--- (8) Check Cache ---->|
      |                      |                         |     (Is PC1 Auth'd?)    |
      |                      |                         |--- (9) Auth Check ----->|
      |                      |                         |<-- (10) Success --------|
      |                      |<-- (11) Access-Accept --|                         |
      |                      |   (Full Access)         |                         |
```

---

## ⚙ 動作シーケンス

1.  **マシン認証**: Windows 端末の起動直後、サプリカントは `host/ComputerName.domain` の形式で RADIUS リクエストを送信します。
2.  **キャッシュ登録**: ISE は AD 連携を通じてマシンを認証し、成功した場合はその MAC アドレスを「Machine Authenticated = True」として内部キャッシュに登録します。
3.  **ユーザー認証**: ユーザーがログオン画面で資格情報を入力すると、ユーザー認証のリクエストが飛びます。
4.  **MAR チェック**: ISE は認可ポリシー（Authorization Policy）を評価する際、`WasMachineAuthenticated` 属性を確認します。
5.  **認可の決定**:
    *   キャッシュに「True」があれば、社内リソースへのアクセスを許可。
    *   キャッシュになければ（または期限切れ）、制限されたアクセスまたは拒否。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **属性の使い分け**: 認可ポリシーの条件として `Network Access:WasMachineAuthenticated EQUALS True` を正しく選択できることが重要です。
*   **キャッシュエイジング**: 試験問題で「12時間以内にマシン認証された端末のみ」といった要件が出た場合、ISE の **Settings > Endpoint Policy** 等でキャッシュの有効期間を調整する設定箇所を把握しておく必要があります。
*   **Identity Source Sequence (ISS)**: マシン認証とユーザー認証で参照する AD が異なる場合や、複数の AD フォレストがある環境での ISS 構成が問われます。
*   **MAR vs TEAP (EAP Chaining)**: 最新の v6.1 試験では、MAR の欠点（キャッシュ依存）を解決する **TEAP** [4.13.a] が優先される傾向にありますが、既存環境のトラブルシュートや移行シナリオとして MAR が出題される可能性があります。
*   **トラブルシュート問題**: 「ユーザー認証は通っているが、VLAN 割当が制限されている（Guest VLAN に落ちている）」場合、ライブログの詳細で `WasMachineAuthenticated` が `False` になっていないか確認する能力が求められます。

---

## 🛠 設定方法

### 1. Active Directory 連携の確認
ISE が AD ドメインに正常に参加していることを確認します（Administration > External Identity Sources > Active Directory）。

### 2. 認可ポリシー（Authorization Policy）の構成
1.  **Policy > Policy Sets** で該当のセットを選択。
2.  **Authorization Policy** セクションで新規ルールを追加。
3.  **Condition**: `Network Access:WasMachineAuthenticated EQUALS True` を追加。
4.  **Permissions**: `PermitAccess`（フルアクセス）を指定。
5.  **（重要）** その下位に、マシン認証されていない場合のフォールバックルール（例：制限 ACL の適用）を作成します。

### 3. マシンアクセスの制限設定 (Global Settings)
*   **Administration > System > Settings > Protocols > RADIUS**
*   `Machine Access Restriction (MAR)` セクションで `Enable MAR` にチェックを入れ、キャッシュ時間を定義します。

---

## 🔍 検証コマンド

| 目的 | コマンド / 手法 |
| :--- | :--- |
| **ISE 認証ログの確認** | **Operations > RADIUS > Live Logs** |
| **MAR 属性の確認** | Live Logs の **Details** を開き、`Authorization Attributes` セクションで `WasMachineAuthenticated` を確認。 |
| **エンドポイントの認証状態** | **Context Visibility > Endpoints** で対象 MAC の属性を確認。 |
| **AD 連携状態の確認** | <code>show application status ise</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| **User Authenticated だが MAR が False** | マシン認証がスキップされた | 端末の GPO またはサプリカント設定で「マシン認証を有効にする」がチェックされているか確認。 |
| **再起動後にアクセス拒否** | キャッシュの期限切れ | ISE 側の MAR キャッシュ有効期間（TTL）を長く設定する（デフォルト 24時間）。 |
| **MAC アドレスの偽装** | MAB 端末によるなりすまし | MAR は AD に対する証明書やパスワード認証を前提とするため、MAB 端末は `False` になるのが正常。 |
| **ライブログに属性が出ない** | MAR 設定が有効化されていない | **System > Settings** で MAR 機能自体がオンになっているか確認。 |

---

## ⚠ 制限事項

*   **キャッシュの不確実性**: ISE ノードを再起動したり、セッションがクリアされるとマシン認証の状態が失われます。
*   **マルチ PSN 環境**: マシン認証を受けた PSN と、ユーザー認証を受ける PSN が異なる場合（ロードバランス等）、キャッシュが同期されていないと MAR チェックに失敗することがあります。
*   **スマホ / タブレット**: iOS や Android はネイティブな AD マシン認証を行わないため、MAR を適用すると常に拒否されます。これらには MDM 連携 [4.11] が推奨されます。

---

## 🔄 他技術との関連

*   **4.13.a EAP Chaining (TEAP)**: MAR の「キャッシュ頼み」を排除し、単一パケットでマシンとユーザーを同時検証する後継技術です。
*   **4.7 Active Directory Integration**: MAR の判定基準となる ID ストアです。
*   **2.6 Microsegmentation**: `WasMachineAuthenticated = True` の端末にのみ特定の SGT を付与する設計が一般的です。

---

## 🧩 比較表

### MAR (PEAP) vs EAP Chaining (TEAP)

| 特徴 | MAR (Machine Access Restriction) | EAP Chaining (TEAP) |
| :--- | :--- | :--- |
| **判定ロジック** | **キャッシュベース** (状態の紐付け) | **プロトコルベース** (同時認証) |
| **信頼性** | 中 (キャッシュ切れや不一致が起きる) | **高** (常に最新の状態を検証) |
| **複雑さ** | 低 (既存の PEAP 設定＋ISE 条件) | 高 (TEAP 対応サプリカントが必要) |
| **推奨シーン** | レガシー Windows クライアント | **モダンな Windows 10/11 環境** |

---

## 💡 ベストプラクティス

1.  **TEAP への移行**: 新規導入時は MAR ではなく **TEAP** の使用を強く推奨します。
2.  **猶予期間の設定**: マシン認証のキャッシュ期間は、週末の PC シャットダウンを考慮して、社員の勤務サイクル（例：36時間〜48時間）に合わせるのが実用的です。
3.  **可視化モード**: 最初から MAR で拒否（Reject）せず、属性をログに記録して `WasMachineAuthenticated = False` の端末がどれだけいるか調査してから強制適用に移行します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な MAR 条件の作成
*   **要件**: `Domain Users` グループのユーザーであっても、マシン認証が完了していない場合は VLAN 99 へ隔離せよ。
*   **ISE 設定**:
    - Authz Rule 1: `AD-Group = Domain Users` AND `WasMachineAuthenticated = True` -> `VLAN 10`.
    - Authz Rule 2: `AD-Group = Domain Users` -> `VLAN 99`.

### 2. MAR キャッシュ時間の変更
*   **操作**: ISE 設定画面から MAR 有効期限を 12 時間に変更せよ。

### 3. マシン認証の優先実行
*   **要件**: Windows 端末の有線接続時に、ログオン前に必ずマシン認証が走るよう GPO 設定を模した確認を行え。

### 4. 特定 identity ソースでの MAR 無効化
*   **要件**: ゲストユーザー（Web Auth）に対しては MAR チェックをスキップするように構成せよ。

### 5. MAR と Profiling の併用
*   **要件**: `Workstation` とプロファイルされたデバイスのみ MAR をチェックせよ。

### 6. マルチフォレスト環境での MAR
*   **要件**: Forest A のマシンと Forest B のユーザーの組み合わせで認証を成功させよ。

### 7. CoA による MAR 状態の更新
*   **シナリオ**: マシン認証が後から成功した場合、CoA を通じてユーザーの権限を即座に昇格させよ。

### 8. ライブログ詳細の読解
*   **課題**: ユーザー認証がパスしているにもかかわらず、属性 `Machine-Access-Restriction = Lookup failed` となる原因を特定せよ。

### 9. マシン認証のみの認可ルール
*   **要件**: ログオン前（マシン認証のみ成功）の状態に対し、AD 通信のみを許可する dACL を付与せよ。

### 10. MAR 障害時のフォールバック
*   **要件**: ISE が AD との通信を一時的に失った際、キャッシュにある MAR 情報をどのように扱うかポリシーを定義せよ。

---

## ❓ 想定試験問題

1.  **Design**: MAR が ISE のランタイムキャッシュに依存していることによる最大のリスクは何か？
    *   **回答**: キャッシュがクリアされた場合（ノード再起動等）、ユーザーが再ログオンしてマシン認証を再試行するまで正しい権限が付与されないこと。
2.  **トラブルシュート**: Windows 10 端末で、マシン認証は成功しているが、ユーザーがログオンすると `WasMachineAuthenticated` が `False` になる。原因として考えられるサプリカントの設定は？
    *   **回答**: ユーザー認証時に「既存のマシン認証セッションを維持する」設定が無効化されている、または別のネットワークプロファイルが適用されている。
3.  **コンフィグ読解**: `Network Access:WasMachineAuthenticated` という属性は、ISE のどのポリシー階層で使用されるべきか？
    *   **回答**: **Authorization Policy（認可ポリシー）** の Conditions（条件）部分。
4.  **Design**: MAR の代わりに TEAP を使用する構成上のメリットを 1 つ挙げよ。
    *   **回答**: キャッシュの状態に依存せず、1 つの EAP トンネル内でマシンとユーザーの整合性をプロトコルレベルで保証できる点。
5.  **実装**: 共有 PC 環境で、前のユーザーのマシン認証キャッシュが残っているために、次のユーザーが非ドメイン端末からログインできてしまう問題を解決するには？
    *   **回答**: ユーザーログアウト時にポートをバウンス（Port Bounce）させる CoA 設定を行うか、キャッシュ有効期間を最短に設定する。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [Machine Access Restriction の設定](https://www.cisco.com/c/ja_jp/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31.html)
*   **Technical Note**: [Understanding and Configuring MAR in ISE](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/116347-device-sensor-ise-00.html)
*   **Cisco Live (BRKSEC-2041)**: [Active Directory Integration with ISE Deep Dive](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「MAR は ISE が付ける付箋」と考えてください。マシン認証が通ると MAC アドレスに「合格」の付箋を貼り、後でユーザーが来た時にその付箋が付いているかを確認します。
*   **図解**: 
    - 端末起動 -> ISE: `MAC 00:11:22 = Machine OK` (Cache)
    - ユーザーログイン -> ISE: `User Alice on MAC 00:11:22... check cache... Found OK!`
*   **注意点**: ラボ試験で MAR が要求された場合、**ISE の設定画面で MAR が明示的に有効（Enable）になっているか**を最初に見落とさないようにしてください。デフォルトでオフの場合があります。
