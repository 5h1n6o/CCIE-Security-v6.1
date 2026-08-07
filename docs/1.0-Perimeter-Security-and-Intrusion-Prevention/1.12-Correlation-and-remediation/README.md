---
layout: default
title: 1.12-Correlation-and-remediation
nav_order: 12
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.12 Correlation and remediation rules on Cisco FMC

Cisco Secure Firewall Management Center (FMC) における**相関（Correlation）**と**修復（Remediation）**ルールは、ネットワーク内で発生する膨大なイベントの中から特定の脅威パターンを自動的に識別し、即座に応答アクションを実行するための強力なフレームワークです。CCIE Security ラボ試験では、単にイベントを検知するだけでなく、ISE との連携（pxGrid）や、FMC 自身による自動的なブロックといった「クローズドループ」な自動防御構成の実装能力が問われます。

---

## 📘 概要

*   **機能概要**: 
    *   **Correlation Rules**: 侵入イベント、接続イベント、ホスト情報の変更などの条件を組み合わせた「論理式」を定義します。
    *   **Remediation Rules**: 相関ルールにマッチした際、外部システム（Cisco ISE 等）への通知や、デバイスへの ACL 追加といった「自動修復アクション」を実行します。
*   **利用目的**: 管理者の介入なしに脅威を封じ込める（Quarantine）、または重要なセキュリティ変化（管理権限の奪取試行など）に対してリアルタイムで警告を発する。
*   **どのような場面で利用するか**: 
    *   特定のホストが短時間に複数の攻撃を受けた際に自動で隔離。
    *   許可されていない OS やアプリケーションの起動を検知して管理者へ通知。
    *   マルウェア検知時に、その送信元 IP を自動的にブラックリストへ登録。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要コンポーネント** | Correlation Rules, Correlation Policies, Responses (Alerts/Remediations)。 |
| **評価タイミング** | パケットが Snort エンジンを通過し、FMC がイベントを受信した直後。 |
| **修復の柔軟性** | 標準の修復モジュール（ISE, Nmap 等）に加え、SDK によるカスタムスクリプトも可能。 |
| **統合要件** | 外部連携（ISE 等）には、正しい証明書交換と pxGrid/REST API 設定が必要。 |
| **メリット** | インシデントレスポンスの高速化、手動操作ミスの削減。 |
| **設計上の注意点** | 過剰な自動化は正当な通信を遮断（False Positive）するリスクがあるため、ルールの精査が不可欠。 |

---

## 🏗 動作原理

FMC の相関エンジンは、管理下の全 FTD デバイスから集約されるイベントストリームを監視します。

```text
[ FTD Devices ] ----(Event Stream)----> [ FMC Correlation Engine ]
                                               ↓
                                        [ Rule Matching ] <--- (Logical AND/OR)
                                               ↓
                                        [ Policy Trigger ]
                                               ↓
[ Response: Alert ] <------------------- [ Action ] -------------------> [ Response: Remediation ]
(Syslog, Email, SNMP)                                                (ISE Quarantine, ACL Block)
```

---

## ⚙ 動作シーケンス

1.  **イベント受信**: FTD で Snort 判定（IPS/AMP）や接続制御が行われ、イベントが FMC に送出される。
2.  **ルール評価**: FMC は設定された `Correlation Rules` に基づき、イベントの属性（IP, アプリケーション, シグネチャ ID 等）をチェックする。
3.  **ポリシーマッチング**: ルールが `Correlation Policy` 内で有効化されており、かつ有効な時間帯（Time Range）であるかを確認。
4.  **レスポンスのトリガー**: 条件を満たした場合、紐付けられた `Responses`（警告または修復）をキューに入れる。
5.  **修復の実行**: 
    *   **ISE 連携の場合**: FMC が pxGrid を介して ISE に「ANC ポリシー（Quarantine）」の適用を依頼。
    *   **ローカル修復の場合**: FTD の設定を更新して特定のトラフィックを遮断。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **ISE Quarantine の実装**: 最も頻出するシナリオです。FMC と ISE を統合し、IPS イベント（重要度 High 等）をトリガーに、ISE 側でそのホストを自動的に隔離（Quarantine）させる一連の流れ。
*   **イベントの絞り込み**: 「すべての攻撃」ではなく、「特定のクラス（例: Web-active）の攻撃かつ、宛先が DMZ ゾーン」といった、複雑なルールの組み合わせ能力。
*   **ホワイトリスト/除外設定**: 自動修復の対象外とする管理端末などの設定。
*   **Remediation Module の設定**: FMC 上で `Policies > Responses > Remediation > Modules` から、インスタンスを正しく作成・保存する手順。
*   **Troubleshooting**: 修復が実行されない場合に、`Correlation Events` 画面でルールにマッチしているか、あるいは `Task Status` で外部連携エラーが出ていないかを確認する。

---

## 🛠 設定方法

### 1. 相関ルールの作成 (FMC GUI)
1.  **Policies > Correlation > Rule Management**。
2.  **Create Rule** をクリック。
3.  **If**: `an intrusion event occurs` を選択。
4.  **Condition**: `Priority is High` かつ `Signature ID is [特定ID]` 等を指定。

### 2. 修復アクションの定義 (ISE 連携例)
1.  **Policies > Responses > Remediation > Modules**。
2.  `ISE Remediation` モジュールを設定し、ISE の IP や認証情報を入力。
3.  **Remediation > Configured Remediations** で、具体的なアクション（例: Adaptive Network Control - Quarantine）を作成。

### 3. 相関ポリシーへの適用
1.  **Policies > Correlation > Policy Management**。
2.  新しいポリシーを作成し、作成した **Rule** を追加。
3.  そのルールに対し、作成した **Response** (Remediation) を紐付ける。
4.  ポリシーの **Status** を `On` にして保存。

---

## 🔍 検証コマンド

| 目的 | コマンド / 操作 |
| :--- | :--- |
| **ルールのマッチング履歴確認** | FMC GUI: <code>Analysis > Correlation > Correlation Events</code> |
| **修復タスクの実行状態確認** | FMC GUI: <code>Message Center > Task Status</code> |
| **ISE 側の状態確認** | ISE GUI: <code>Operations > Adaptive Network Control > Endpoint Assignments</code> |
| **FTD 側のブロック確認** | FTD CLI: <code>system support diagnostic-cli</code> -> <code>show access-list</code> |
| **pxGrid 疎通確認** | <code>system support diagnostic-cli</code> (FMC側での接続性テスト) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| イベントは発生するがルールにマッチしない | ルールの論理条件（AND/OR）ミス | <code>Correlation Events</code> を確認し、どの条件が不一致か精査。 |
| 修復タスクが "Failed" になる | 外部システム（ISE）との認証失敗 | FMC の <code>pxGrid</code> 接続状態と、証明書の有効期限を確認。 |
| 隔離されたホストが戻らない | 解除ルールの未設定 | ISE 側で <code>Unquarantine</code> を手動または自動で実行する設定を確認。 |
| デプロイしてもポリシーが動かない | ポリシー自体が "Off" 状態 | <code>Policy Management</code> でトグルが <code>On</code> か確認。 |

---

## ⚠ 制限事項

*   **遅延**: FTD から FMC へのイベント転送と、FMC での相関処理には数秒〜数十秒の遅延が発生します。リアルタイム遮断（インラインドロップ）とは動作が異なります。
*   **ライセンス**: ISE 連携には ISE 側の **Advantage/Premier** ライセンス、FMC 側の **Threat** ライセンスが必要です。
*   **ホスト制限**: FMC が管理できるホストデータベースの容量（モデル依存）を超えると、ホスト情報に基づく相関ルールが機能しなくなる場合があります。

---

## 🔄 他技術との関連

*   **Cisco ISE (pxGrid)**: 修復ルールの主要な宛先であり、SGT 変更や Quarantine アクションを実行します。
*   **Intrusion Policy (1.9)**: ルールのトリガーとなる「イベント」を生成する根源です。
*   **Host Profiling**: FMC が収集する OS やアプリケーション情報に基づき、「脆弱な OS への攻撃」をトリガーにする高度な相関が可能。
*   **Access Control Policy**: 修復結果として動的に追加される ACL などの適用先です。

---

## 🧩 比較表

### Alert vs Remediation

| 特徴 | Alert (警告) | Remediation (修復) |
| :--- | :--- | :--- |
| **主な目的** | 管理者への通知 | 脅威の自動排除・封じ込め |
| **プロトコル** | Syslog, SNMP, Email | pxGrid, HTTP REST, SSH, Custom Script |
| **システムへの影響** | 低い（読み取り専用） | 高い（ネットワーク構成を変更） |
| **試験での重要度** | 中 | **非常に高い** |

---

## 💡 ベストプラクティス

1.  **段階的な導入**: 最初は `Response` を `Alert` のみに設定し、誤検知がないことを確認してから `Remediation` に切り替えます。
2.  **Specific Rules**: `an intrusion event occurs` だけのような広すぎるルールは避け、特定の脆弱性や重要サーバーを対象に絞ります。
3.  **証明書の管理**: 外部連携は証明書ベースの認証が多いため、NTP による時刻同期を完璧に保ちます。
4.  **テストイベントの活用**: 実際に攻撃パケットを生成（Scapy や Nmap 等）し、修復タスクが完結するか定期的に検証します。

---

## 📝 ラボ学習・設定サンプル例

### 1. Web サーバーへの High 攻撃に対する ISE 隔離
*   **要件**: 宛先が WebServer_Object、かつ重要度 High の侵入イベントが発生したら、ISE で隔離せよ。
*   **設定**: Rule (Dst IP = WebServer AND Priority = High) -> Policy (Add Rule, Response = ISE Quarantine)。

### 2. 特定アプリケーション（BitTorrent）の使用検知
*   **要件**: BitTorrent の通信が検知されたら、管理者に Syslog を飛ばせ。
*   **設定**: Rule (Application = BitTorrent) -> Policy (Response = Syslog Alert)。

### 3. ホストの OS 変更に対する通知
*   **要件**: サーバーホストの OS が「Linux」から「Windows」に変更された（偽装の可能性）場合に警告せよ。
*   **設定**: Rule (Host Profile change: OS) -> Policy (Notification)。

### 4. 複数回失敗ログインの相関
*   **要件**: 同一 IP から 1 分以内に 5 回以上の接続拒否が発生したらブロックせよ。
*   **設定**: Rule (Connection Event, Count > 5, Time < 60s)。

### 5. マルウェア検知後のフルスキャン起動
*   **要件**: ファイルイベントでマルウェアが確定したら、対象ホストに Nmap スキャンを実行せよ。
*   **設定**: Remediation Module: Nmap を使用。

### 6. 許可されないポートのオープン
*   **要件**: ホストプロファイルで新たに TCP/445 が開かれた場合に通知せよ。

### 7. pxGrid による SGT 変更
*   **要件**: 不審な動きを検知したら、そのホストの SGT を `TrustMe` から `Untrusted` へ変更せよ。

### 8. FTD 自身へのブルートフォース対策
*   **要件**: 管理インターフェイスへの SSH 失敗をトリガーに送信元を遮断せよ。

### 9. ホワイトリストによる自動修復除外
*   **要件**: 管理者端末 (Admin_PC) からのイベントでは修復をスキップせよ。
*   **設定**: Rule Condition に `Source IP is NOT Admin_PC` を追加。

### 10. カスタム修復スクリプトの登録
*   **要件**: インシデント発生時に FMC から外部の REST API を叩く Python スクリプトを実行せよ。

---

## ❓ 想定試験問題

1.  **実装**: FMC と ISE 間の接続を確立するための必須設定項目を 3 つ挙げよ。
    *   **回答**: ISE の pxGrid サービス有効化、FMC/ISE 間の証明書信頼、FMC 上の pxGrid ユーザ登録。
2.  **トラブルシュート**: 相関ポリシーを有効にしたが、Connection Events には表示されるのに Correlation Events に何も記録されない。考えられる原因は？
    *   **回答**: ルール内で指定した条件（例: ユーザ名）が、実際のイベントに含まれていない（Identity Policy が未設定など）。
3.  **Design**: 自動修復による「ネットワークの分断」を最小限にするためのルール設計の工夫を述べよ。
    *   **回答**: 攻撃の「重要度（Priority）」だけでなく、「インパクト（Impact）」や「ホストの脆弱性（Vulnerability）」を条件に加える。
4.  **コンフィグ読解**: FMC の修復モジュール設定で `Instance Name` が複数ある場合、ポリシーではどのように使い分けるか？
    *   **回答**: ポリシー内の個別のルールに対して、異なる修復インスタンスをレスポンスとして紐付けることができる。
5.  **Design**: 大規模環境で相関ルールを多用した場合の FMC への負荷への影響を説明せよ。
    *   **回答**: 全イベントをメモリ上で論理評価するため、イベント数とルール数が極端に多い場合、FMC の応答性能が低下する可能性がある。

---

## 🔗 参考リソース

*   **Cisco Firepower Management Center Administration Guide, 7.0**
    *   [Correlation Policies](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/correlation_policies.html)
    *   [Remediation Policies](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/remediation_policies.html)
*   **Cisco Live BRKSEC-3020**
    *   [Advanced Firepower Troubleshooting](https://www.ciscolive.com/)
*   **Technical Notes**
    *   [FMC and ISE Integration using pxGrid](https://www.cisco.com/c/en/us/support/docs/security/firepower-ngfw/214954-configure-fmc-and-ise-integration-using.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「イベントを見ているだけでは IPS、何かをさせるのが相関」と整理しましょう。特に CCIE ラボでは **ISE との連携** が目玉ですので、ISE の ANC (Adaptive Network Control) 設定とセットで復習してください。
*   **図解**: パケットが FTD を通り (1)、イベントが FMC に上がり (2)、FMC が ISE に命令を出し (3)、ISE がスイッチに CoA を送る (4) という 4 ステップのフローを常に意識してください。
*   **注意点**: ラボ試験では `Save` を忘れないでください。相関ルールのエディタは保存ボタンが階層の深い位置にあることがあり、設定が消えてしまうミスが多発します。
