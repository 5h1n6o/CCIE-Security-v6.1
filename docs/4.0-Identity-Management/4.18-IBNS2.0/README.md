---
layout: default
title: 4.18-IBNS2.0
nav_order: 18
parent: 4.0-Identity-Management
---

# 4.18 Cisco IBNS 2.0 (C3PL) for authentication, access control, and user policy enforcement

**Cisco IBNS 2.0 (Identity-Based Networking Services 2.0)** は、ネットワークアクセス制御を柔軟かつモジュール化された形式で実装するための次世代フレームワークです。従来の「レガシー」な認証コマンド（`authentication port-control auto` など）に代わり、**C3PL (Cisco Common Classification Policy Language)** を採用しています。これにより、QoS（MQC）のように「クラスマップ」と「ポリシーマップ」を用いて、特定のイベント（セッション開始、認証失敗など）に対して詳細なアクションを定義することが可能になります。

---

## 📘 概要

*   **機能概要**: 802.1X、MAB（MAC Authentication Bypass）、WebAuth などの認証方式を、イベント駆動型のポリシー（C3PL）で管理する仕組みです。
*   **利用目的**: 認証プロセスの一貫性を保ちつつ、デバイスやユーザーの状態に応じた高度に動的なアクセス制御（VLAN 割り当て、ACL 適用、SGT 付与など）を実現します。
*   **どのような場面で利用するか**: 
    *   単一のポートで複数の認証方式（802.1X と MAB）を順次実行する場合。
    *   認証の結果やデバイスの種類（プロファイリング結果）に基づいて、即座に特定のセキュリティポリシーを適用したい場合。
    *   Cisco DNA Center (DNAC) 環境における、自動化されたポート構成。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **基盤言語** | **C3PL (Cisco Common Classification Policy Language)**。 |
| **主要コンポーネント** | Control Class-map, Control Policy-map, Service Template。 |
| **認証方式** | 802.1X, MAB, WebAuth (L2/L3)。 |
| **適用レベル** | ポート（インターフェイス）単位で適用。 |
| **メリット** | 認証順序、フォールバック、再試行ロジックを柔軟にカスタマイズ可能。 |
| **互換性** | IBNS 1.0 (Legacy) と 2.0 は同一ポート内で混在不可。 |

---

## 🏗 動作原理

IBNS 2.0 は **Event-Condition-Action (ECA)** モデルで動作します。

```text
Event (イベント)
   ↓
Condition (条件/Class-map)
   ↓
Action (実行/Policy-map)
```

1.  **Event**: 「セッション開始」「認証失敗」「エージェント発見」などのトリガー。
2.  **Condition**: `class-map type control` で定義された、特定のプロトコルや状態（例：DOT1X が失敗した等）。
3.  **Action**: `policy-map type control` 内で実行される動作（例：MAB を開始する、VLAN 10 を付与する）。

---

## ⚙ 動作シーケンス

一般的な「802.1X が優先、失敗したら MAB」という IBNS 2.0 のフローは以下の通りです。

1.  **Session Started**: リンクアップを検知。
2.  **Step 1 (DOT1X)**: スイッチが 802.1X 認証を開始。
3.  **Authentication Event**: 
    *   **成功**: `Service-Template` を介して認可属性（VLAN, ACL）を適用。
    *   **失敗 (dot1x-no-response/fail)**: 次のクラス（MAB）へ移行。
4.  **Step 2 (MAB)**: 802.1X が応答しない場合、MAC アドレスをユーザー名として RADIUS サーバへ送信。
5.  **Finalize**: 認証・認可が完了すると、`access-session` として状態が管理される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **コンフィグの移行能力**: レガシーコマンドを IBNS 2.0 (C3PL) 形式に書き換える問題が予想されます。「`authentication order` を C3PL の `event session-started` セクションでどう表現するか」を理解しておく必要があります。
*   **Service Template の活用**: `service-template` で定義した ACL や VLAN 名が、RADIUS アトリビュートから返される値と一致しているかどうかが重要です。
*   **認証の優先順位とフォールバック**: 「dot1x タイムアウト後に即座に MAB を開始せよ」という要件に対し、`class-map` で `dot1x-timeout` を定義し、ポリシーマップで `activate method mab` を設定する手順は必須です。
*   **show コマンドの読解**: 
    *   `show access-session interface [int] details` で現在のポリシー適用状態を確認。
    *   `show authentication display config` で IBNS 1.0 か 2.0 かを判別。

---

## 🛠 設定方法

### 1. Control Class-map の定義
特定の状態（認証成功、失敗等）を定義します。

```bash
class-map type control subscriber match-all DOT1X_FAILED
 match method dot1x
 match result-type method-dot1x authentication-failed
!
class-map type control subscriber match-all DOT1X_NO_RESP
 match method dot1x
 match result-type method-dot1x agent-not-found
```

### 2. Service Template の定義（オプション）
ローカルで権限を定義する場合に使用します。

```bash
service-template GUEST_TEMPLATE
 vlan 999
 access-group GUEST_ACL
```

### 3. Control Policy-map の構成
ロジックを組み立てます。

```bash
policy-map type control subscriber IBNS_POLICY
 event session-started
  10 authenticate using dot1x priority 10
 event authentication-failure
  10 class DOT1X_NO_RESP
   10 authenticate using mab priority 20
  20 class DOT1X_FAILED
   10 terminate dot1x
   20 authenticate using mab priority 20
 event authentication-success
  10 permit
```

### 4. インターフェイスへの適用
```bash
interface GigabitEthernet1/0/1
 service-policy type control subscriber IBNS_POLICY
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **セッション詳細確認** | <code>show access-session interface [int] details</code> |
| **ポリシー適用状態の確認** | <code>show policy-map type control subscriber</code> |
| **全セッションのサマリー** | <code>show access-session</code> |
| **C3PL のデバッグ** | <code>debug access-session control-plane</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 認証が全く始まらない | `service-policy` がインターフェイスに未適用 | `show run int` で適用を確認。 |
| クラスマップに合致しない | `match` 条件の誤り（成功/失敗の定義ミス） | `show policy-map control` の `hits` カウンタを確認。 |
| MAB へのフォールバックが遅い | タイムアウト値がデフォルトで長い | `dot1x timeout tx-period` を短縮設定。 |
| 認可属性（VLAN等）が反映されない | Service Template と RADIUS 応答の不一致 | RADIUS サーバ側の VSA 設定と `service-template` 名を確認。 |

---

## ⚠ 制限事項

*   **同時実行の禁止**: `authentication` (Legacy) コマンドと `service-policy` (C3PL) は同一インターフェイスに共存できません。
*   **プラットフォーム依存**: 古いスイッチモデル（Catalyst 2960 等）では IBNS 2.0 がフルサポートされていない場合があります。
*   **複雑性**: 従来の 3 行の設定が 20 行の C3PL になるため、管理オーバーヘッドが増大します。

---

## 🔄 他技術との関連

*   **4.14 Identity mapping**: C3PL 認証後のセッション情報を pxGrid 経由で他デバイスへ共有。
*   **4.2 Network Access AAA**: C3PL はバックエンドで RADIUS/ISE と通信します。
*   **2.6 TrustSec**: C3PL のアクションとして SGT を動的に付与可能。

---

## 🧩 比較表

### IBNS 1.0 (Legacy) vs IBNS 2.0 (C3PL)

| 特徴 | IBNS 1.0 (Legacy) | IBNS 2.0 (C3PL) |
| :--- | :--- | :--- |
| **設定形式** | インターフェイス直下の個別コマンド | クラス/ポリシーマップ（MQCライク） |
| **柔軟性** | 低（固定されたフロー） | **極めて高い**（ECAロジック） |
| **可読性** | 高（シンプル） | 低（階層構造が複雑） |
| **推奨環境** | 小規模・シンプル | **SD-Access, 大規模エンタープライズ** |

---

## 💡 ベストプラクティス

1.  **テンプレート化**: `service-template` を積極的に利用し、インターフェイス設定を簡素化します。
2.  **明確な命名規則**: `DOT1X_SUCCESS`, `MAB_FALLBACK` など、役割がひと目でわかるクラス名を使用します。
3.  **Default Action**: 想定外のイベント（ポリシーに一致しないケース）に対する `permit` または `terminate` アクションを末尾に定義し、セッションが「ハング」するのを防ぎます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 802.1X + MAB Fallback (基本)
*   **要件**: 802.1X を優先し、サプリカントがない場合は 30 秒後に MAB を開始せよ。

### 2. 認証失敗時の隔離 VLAN 適用
*   **要件**: 認証に失敗した端末を VLAN 666 に隔離せよ。
*   **C3PL**: `event authentication-failure` -> `class ALL_FAILED` -> `activate service-template QUARANTINE`.

### 3. 特権ユーザー用 SGT 付与
*   **要件**: 成功時に SGT 10 を付与せよ。

### 4. 複数セッションの許可 (Multi-Auth)
*   **要件**: 1 ポートに IP Phone と PC がある環境で IBNS 2.0 を構成せよ。

### 5. WebAuth へのリダイレクト
*   **要件**: 認証前は HTTP 通信を ISE ポータルへリダイレクトせよ。

### 6. オープンアクセス（可視化モード）
*   **要件**: 認証成否にかかわらず通信を許可しつつ、ログを ISE へ送れ。

### 7. 再認証タイマーの構成
*   **要件**: 60 分ごとに再認証を強制せよ。

### 8. リンクダウン時のクリーンアップ
*   **要件**: リンクダウン検知後、即座に access-session を削除せよ。

### 9. 特定ベンダー OUI によるクラス分け
*   **要件**: Cisco 製デバイスからのアクセス時のみ特別なロジックを適用せよ。

### 10. Service Template の RADIUS 駆動適用
*   **要件**: RADIUS サーバから送られる `VSA: Service-Template` 名に合致するローカル設定を呼び出せ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: 下記の C3PL ポリシーにおいて、`dot1x` がタイムアウトした場合に何が起こるか説明せよ。
    ```bash
    event authentication-failure
     class DOT1X_NO_RESP
      authenticate using mab
    ```
    *   **回答**: 802.1X の応答がない（Agent-not-found）場合、認証失敗イベントがトリガーされ、`mab` 認証が開始される。
2.  **トラブルシュート**: `show access-session` で状態が `No Method` となっている。C3PL のどこを確認すべきか？
    *   **回答**: `event session-started` セクションで、初期認証メソッド（`dot1x` または `mab`）が正しく `authenticate using` されているかを確認。
3.  **Design**: IBNS 1.0 から 2.0 へ移行する際の最大のメリットは？
    *   **回答**: 認証イベントに対する柔軟なアクション定義（例：特定の失敗理由に応じた異なる VLAN への隔離など）が可能になる点。
4.  **実装**: 既存の `authentication priority dot1x mab` コマンドを IBNS 2.0 で表現せよ。
    *   **回答**: `authenticate using dot1x priority 10` と `authenticate using mab priority 20` のように、プライオリティ値を指定してポリシーに組み込む。
5.  **Design**: 大規模なキャンパスネットワークで DNA Center を使用する場合、なぜ IBNS 2.0 が選ばれるのか？
    *   **回答**: 抽象化されたポリシー定義が可能であり、ネットワークファブリック全体で一貫した ID ベースの制御を自動化できるため。

---

## 🔗 参考リソース

*   **Configuration Guide**: [Identity-Based Networking Services 2.0 Deployment Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-3/configuration_guide/lyr2/b_173_lyr2_9300_cg/identity_based_networking_services.html)
*   **Cisco Live (BRKSEC-2022)**: [IBNS 2.0 (C3PL) Deep Dive](https://www.ciscolive.com/)
*   **Technical Note**: [Legacy to C3PL Configuration Mapping](https://community.cisco.com/t5/security-knowledge-base/ibns-2-0-c3pl-for-identity-mapping/ta-p/3161044)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「C3PL はネットワークアクセスのためのプログラミング」と考えてください。
*   **図解**: 
    - Session Start -> [if no dot1x response] -> Try MAB -> [if success] -> Apply Template.
*   **注意点**: ラボ試験では、**クラスマップ名の大文字小文字の不一致**でポリシーが効かないことが多いため、注意深く確認してください。
