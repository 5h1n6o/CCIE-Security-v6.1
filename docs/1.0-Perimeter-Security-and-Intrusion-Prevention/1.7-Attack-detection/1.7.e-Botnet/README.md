---
layout: default
title: 1.7.e-Botnet
nav_order: 5
parent: 1.7-Attack-detection
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.7.e Botnet

**ボットネット（Botnet）**は、マルウェアによって感染し、攻撃者（ボットマスター）の指令サーバー（C&C: Command and Control）によって遠隔操作される多数のコンピュータ群です。CCIE Security v6.1 においては、Cisco Secure Firewall (FTD) の **Security Intelligence (SI)** や **DNS ポリシー**、および ASA の **Botnet Traffic Filter** を使用して、C&C サーバーとの通信を検知・遮断する能力が求められます。

---

## 📘 概要

*   **機能概要**: 既知の悪意ある IP アドレス、ドメイン、URL のデータベース（Cisco Talos 等）に基づき、ボットに感染した内部ホストが外部の C&C サーバーへ接続しようとする動きをリアルタイムで阻止します。
*   **利用目的**: データの外部流出（Exfiltration）の阻止、二次攻撃（DDoS への加担等）の防止、および感染ホストの早期特定。
*   **どのような場面で利用するか**: 
    *   インターネット境界でのアウトバウンド通信の監視。
    *   未知のマルウェアが内部に侵入した後の「事後対策（Post-Infection）」として。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な特徴** | シグネチャ検査の前段階（L3/L4/DNS）で、評判（Reputation）に基づき遮断。 |
| **防御の要** | **Security Intelligence (SI)**、**DNS Sinkhole**、**Botnet Traffic Filter**。 |
| **情報の供給源** | **Cisco Talos** (フィード)、独自作成の Blocklist。 |
| **メリット** | Snort エンジンの負荷を軽減（早い段階でドロップ）。ゼロデイ攻撃の C&C 通信にも対応。 |
| **デメリット** | データベースの更新が止まると検知率が低下する。 |
| **対応機種** | Firepower (FTD), ASA (Botnet Traffic Filter ライセンスが必要)。 |
| **設計上の注意点** | 通信を止める「Block」と、調査用の「Monitor」の使い分け。 |

---

## 🏗 動作原理

ボットネット対策は、感染したクライアントが「外」へ助け（指示）を求める通信を遮断することに特化しています。

```text
[ Infected Host ] ─── (C&C Callback: "I'm alive") ───→ [ Cisco FTD / ASA ]
                                                                │
    [ 1. DNS Check ] <--- Is the domain a known C&C?            │
    [ 2. IP Reputation ] <--- Is the Target IP in Blocklist?    │
    [ 3. DNS Sinkhole ] <--- Redirect request to a safe IP.     │
                                                                ↓
[ C&C Server ] <──────────── (Connection Blocked) ──────────── [X]
```

---

## ⚙ 動作シーケンス

1.  **DNS クエリの発生**: 感染ホストが C&C サーバーのドメイン（例: `bad-botnet.com`）を解決しようとします。
2.  **DNS インスペクション**: FTD の DNS ポリシーがクエリを傍受し、Talos のボットネットデータベースと照合します。
3.  **Sinkhole 処理**: 悪意あるドメインの場合、FTD は偽の応答（Sinkhole IP）を返し、クライアントが本物の C&C に繋がらないようにします。
4.  **IP フィルタリング**: 直接続を試みる場合、Security Intelligence が L3 ヘッダーを確認し、アクセスコントロールルールの評価前にパケットを破棄します。
5.  **イベント通知**: FMC に「Botnet C&C 通信」として詳細なログ（ソースホスト、宛先 C&C 情報）が送信されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Security Intelligence (SI) の有効化**: FMC の Access Control Policy 内で **Security Intelligence** タブを設定する手順が最重要です。
*   **DNS ポリシーと Sinkhole**: 
    *   単にブロックするだけでなく、「Sinkhole」を使用して特定の IP アドレス（例: `1.1.1.1`）にリダイレクトさせる設定。
    *   これにより、どの内部ホストがボット通信を試みたかを、その Sinkhole IP 宛の通信ログから特定できます。
*   **ASA Botnet Traffic Filter (BTF)**:
    *   `dynamic-filter-database` の有効化。
    *   `inspect dns dynamic-filter-lookup` の設定。
    *   `ambiguous`（グレーな通信）に対するアクション設定。
*   **カスタム Blocklist の作成**: 
    *   Talos フィードにない特定の IP アドレスをテキストファイルで作成し、FMC にインポートして SI ルールでブロックする手順。
*   **ホワイトリスト (Exemption)**: 正規の通信（例: セキュリティスキャナ）がボット判定されないよう、Exempt 設定を行うスキル。

---

## 🛠 設定方法

### 1. FTD: Security Intelligence の設定 (FMC)
1.  **Policies > Access Control** で対象ポリシーを編集。
2.  **Security Intelligence** タブを選択。
3.  **Networks** セクションで `Cisco-Botnet-IP-List` を **Blocklist** へ移動。
4.  **DNS/Domains** タブで `Cisco-Botnet-Domain-List` を **Blocklist** へ移動。
5.  (オプション) DNS ポリシーを作成し、Sinkhole IP を定義して適用。

### 2. ASA: Botnet Traffic Filter の CLI 設定
```bash
! データベースの自動更新を有効化
dynamic-filter-database
! DNS インスペクションとの連携
policy-map global_policy
 class inspection_default
  inspect dns dynamic-filter-lookup
! 境界インターフェイスでの有効化
dynamic-filter compliance outside
! 自動遮断の有効化 (Threat Level に基づく)
dynamic-filter drop blacklist interface outside threat-level very-high
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **FTD SI 統計確認** | <code>show security-intelligence statistics</code> |
| **ASA ボットネットDB状態** | <code>show dynamic-filter data</code> |
| **ASA 現在の遮断リスト** | <code>show dynamic-filter reports client-stats</code> |
| **Snort 判定デバッグ** | <code>system support firewall-engine-debug</code> |
| **FMC イベント確認** | Analysis > **Connections** > **Security Intelligence Events** |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| ボット通信が素通りする | ライセンス不足 | <code>Threat</code> ライセンスが有効か確認。 |
| DB が更新されない | 疎通または時刻同期の問題 | FMC がインターネットに接続可能か、NTP が同期しているか確認。 |
| 正常な通信がブロックされる | 過検知（False Positive） | <code>Security Intelligence</code> イベントで原因を確認し <code>Whitelist</code> へ追加。 |
| Sinkhole が機能しない | DNS ポリシーの適用漏れ | ACP の <code>Advanced</code> タブで DNS ポリシーが選択されているか確認。 |

---

## ⚠ 制限事項

*   **ASA BTF ライセンス**: FTD の SI とは異なり、ASA でボットネットフィルタリングを行うには専用のサブスクリプションが必要です。
*   **暗号化 C&C 通信**: HTTPS 等の暗号化された C&C 通信は、IP/ドメインレベルでは遮断可能ですが、ペイロードの詳細検査には SSL 復号が必要です。
*   **プライベート DNS**: 内部 DNS サーバーのみを使用している環境では、FTD/ASA が DNS クエリを直接見られない場合があるため、パッシブな IP フィルタリングに頼ることになります。

---

## 🔄 他技術との関連

*   **Talos Intelligence**: 常に最新のボットネット情報をデバイスへフィードします。
*   **AMP (Advanced Malware Protection)**: ボットネットを生み出す原因となる「ファイル（マルウェア）」そのものをエンドポイントやゲートウェイで排除します。
*   **Stealthwatch (Secure Network Analytics)**: シグネチャがなくても、異常な大量通信（DDoS 加担）や未定義の C&C への定期的通信（ビーコニング）をフロー解析で検知します。

---

## 🧩 比較表

### Security Intelligence (FTD) vs Botnet Traffic Filter (ASA)

| 機能 | FTD Security Intelligence | ASA Botnet Traffic Filter |
| :--- | :--- | :--- |
| **管理単位** | オブジェクトベース（FMC） | インターフェイス/グローバル |
| **DNS Sinkhole** | **標準サポート** | 限定的（リダイレクトのみ） |
| **情報の詳細度** | Talos カテゴリ別に詳細分類 | 5段階の Threat Level |
| **推奨用途** | モダンな NGFW 構成 | レガシー ASA 構成の延命 |

---

## 💡 ベストプラクティス

1.  **Monitor から Block へ**: 導入初期は SI アクションを `Monitor` に設定し、数日間ログを観察して誤検知がないことを確認してから `Block` に切り替えます。
2.  **Sinkhole の活用**: ボットネットの検知には `Block` よりも `Sinkhole` が推奨されます。これにより、感染端末を特定しやすくなるためです。
3.  **複数階層での防御**: DNS レイヤ（Umbrella）＋ ゲートウェイ（FTD SI）＋ エンドポイント（AMP）の 3 段階で C&C 通信をブロックします。

---

## 📝 ラボ学習・設定サンプル例

### 1. Talos フィードによる IP ブロック
*   **要件**: ボットネットに関連する既知の IP アドレスを SI で自動遮断せよ。
*   **設定**: FMC ACP > Security Intelligence > Networks > `Cisco-Botnet-IP-List` を Blocklist へ。

### 2. カスタムドメイン Blocklist の作成
*   **要件**: `bad-bot.org` への接続を SI で禁止せよ。
*   **設定**: `bad-bot.org` と記述した .txt をアップロードし、Domain オブジェクトとして SI に適用。

### 3. DNS Sinkhole の実装
*   **要件**: ボットネットドメインへの問い合わせに対し、`1.1.1.1` を返して追跡せよ。
*   **設定**: DNS Policy > Rule (Category: Botnet) > Action: `Sinkhole` (IP: 1.1.1.1)。

### 4. 信頼された内部スキャナの除外
*   **要件**: 管理サーバー `10.1.1.50` がボットネットチェックに引っかからないようにせよ。
*   **設定**: SI タブの **Whitelist** に対象 IP を追加。

### 5. ASA でのボットネットDB手動ダウンロード
*   **コマンド**: `dynamic-filter database fetch`。

### 6. ボットネットイベントのレポーティング
*   **課題**: 過去 1 週間でボット感染が疑われるホストのトップ 10 を FMC で表示せよ。
*   **操作**: Analysis > Dashboards > Security Intelligence。

### 7. 特定カテゴリ（Spam）のボット遮断
*   **要件**: ボットネットのうち、Spam 送信に関連するものだけをモニターせよ。
*   **設定**: SI タブで `Cisco-Spam-IP-List` を Monitor アクションで追加。

### 8. ASA BTF の自動ドロップ設定 (Threat Level)
*   **要件**: ASA において Threat Level が `high` 以上の通信のみ遮断せよ。
*   **設定**: `dynamic-filter drop blacklist interface outside threat-level high`。

### 9. DNS ポリシーと ACP の紐付け
*   **課題**: DNS ポリシーを作成したが、機能していない。原因を特定せよ。
*   **解決**: ACP ルール内ではなく、ACP の **Advanced タブ** にある DNS Policy 設定を確認する。

### 10. IPv6 ボットネット通信の遮断
*   **要件**: IPv6 環境においても SI 機能が動作することを確認せよ。

---

## ❓ 想定試験問題

1.  **実装**: FMC を使用して、組織内のホストが `Talos-Botnet-List` に含まれるドメインを解決しようとした際に、通信を遮断せずに `192.168.99.99` へリダイレクトする構成を完了しなさい。
2.  **トラブルシュート**: ASA で Botnet Traffic Filter を設定したが、`show dynamic-filter data` の件数が 0 のままである。トラブルシュートの手順を述べよ。
    *   **回答**: ライセンスの有無、DNS 設定（`inspect dns`）、および Talos サーバー（`update-server.ironport.com`）への 443 疎通を確認する。
3.  **Design**: ボットネット対策において、通常の Block アクションよりも Sinkhole アクションがセキュリティ運用面で優れている理由を述べよ。
    *   **回答**: 内部ホストに特定の Sinkhole IP を返却させることで、その IP 宛のパケットを監視するだけで「どの端末が感染しているか」を容易に特定・追跡できるため。
4.  **コンフィグ読解**: `dynamic-filter compliance outside` コマンドが ASA に設定されている。この設定の目的は何か？
    *   **回答**: `outside` インターフェイスを通過するトラフィックをボットネットデータベースと照合し、違反をレポート（またはドロップ）するため。
5.  **実装**: Talos フィード以外の、自社独自の「要注意 IP リスト」を FMC の SI に取り込む方法を述べよ。
    *   **回答**: IP リストのテキストファイルを作成し、`Objects > Object Management > Security Intelligence > Network Lists and Feeds` からインポートする。

---

## 🔗 参考リソース

*   **Cisco Firepower Management Center Administration Guide, 7.1**
    *   [Security Intelligence Policies](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/security_intelligence.html)
*   **Cisco ASA Series Configuration Guide**
    *   [Configuring the Botnet Traffic Filter](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/firewall/asa-94-firewall-config/inspect-botnet.html)
*   **Cisco Live (BRKSEC-2021)**
    *   Firepower Threat Defense - Packet Flow and Troubleshooting

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「ボットネット」という要件が出たら、迷わず **Security Intelligence (SI)** を設定箇所として探してください。通常の ACP ルール（L3/L4）で 1 つずつ IP を入れるのは非効率であり、正解ではありません。
*   **図解**: 「評判の悪い場所（Blacklist）」へ行こうとするパケットを、玄関（SI）で追い返すイメージです。
*   **注意点**: ラボ試験では、デバイスが Talos と通信できない隔離環境の場合があります。その際は「カスタムフィード」や「静的リスト」の作成能力が問われるため、手動インポートの手順を確実に覚えておきましょう。
