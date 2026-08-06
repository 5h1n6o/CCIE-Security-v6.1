---
layout: default
title: 1.6.c-Geolocation
nav_order: 3
parent: 1.6-NGFW-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.6.c Geolocation

Cisco Secure Firewall (FTD) における**ジオロケーション（Geolocation）**機能は、IP アドレスを地理的な位置（国や大陸）にマッピングし、その情報を基にトラフィックを制御・可視化する機能です。Firepower Management Center (FMC) が管理するジオロケーションデータベース（VDB と共に更新される GeoDB）を使用し、特定の地域からの攻撃遮断や、法規制（コンプライアンス）に基づいた通信制限を直感的に実装できます。

---

## 📘 概要

*   **機能概要**: 公有 IP アドレスのデータベースを参照し、パケットの送信元または宛先がどの国・地域に属するかを特定します。
*   **利用目的**: 特定の国からのサイバー攻撃（DDoS やスキャン）の一括遮断、コンテンツ配信の地域制限、特定地域へのデータ転送禁止（データ漏洩防止）。
*   **どのような場面で利用するか**: 
    *   海外からの不正アクセスが急増している際の緊急防御。
    *   国内ユーザーのみを対象とした Web サービスの公開。
    *   セキュリティインテリジェンス（SI）と組み合わせた、脅威ベースの動的フィルタリング。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | IP アドレスを国・都市・ドメインに紐付け。動的に更新されるデータベースを使用。 |
| **用途** | リージョン制限、攻撃元遮断、コンプライアンス（輸出規制等）。 |
| **メリット** | 膨大な IP リストを手動管理せず、国名指定で数千のサブネットを制御可能。 |
| **デメリット** | データベースの正確性に依存（VPN やプロキシ経由の回避に弱い）。 |
| **対応機種** | Firepower Threat Defense (FTD) 全モデル。 |
| **制限事項** | **データベースの定期更新（GeoDB Update）**が不可欠。 |
| **設計上の注意点** | 内部のプライベート IP は「Unknown」として処理されるため、ホワイトリスト設計に注意。 |

---

## 🏗 動作原理

ジオロケーションは、アクセスコントロールエンジンがパケットを処理する際に、メモリ上の GeoDB を参照することで動作します。

```text
[ Incoming Packet ] (Source IP: 203.0.113.5)
         ↓
[ Security Intelligence / Access Control Policy ]
         ↓
[ GeoDB Lookup ] --- (IP 203.0.113.5 -> "Country: Japan")
         ↓
[ Policy Match ] --- (Rule: Block "Japan" -> NO / Rule: Permit "Japan" -> YES)
         ↓
[ Verdict ] --- (Allow / Block)
```

---

## ⚙ 動作シーケンス

1.  **GeoDB のロード**: FMC が最新のジオロケーションデータベース（GeoDB）を FTD に配布・ロードします。
2.  **パケット受信**: FTD がインターフェイスで IP パケットを受信します。
3.  **データベース照合**: パケットの送信元/宛先 IP アドレスをインデックスとして、GeoDB 内の国コード（ISO 3166）を検索します。
4.  **ポリシー適用**: 
    *   **Security Intelligence (SI)**: アクセスコントロールルール評価前の早い段階で、国別リストに基づいて遮断します。
    *   **Access Control Policy (ACP)**: ルール内の「Geolocation」タブで定義された条件に基づき処理します。
5.  **ロギング**: イベントログに国名が表示され、Analysis ダッシュボードで視覚化されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **データベースの更新**: ラボ試験では、インターネット接続がない環境を想定し、FMC に手動で GeoDB ファイルをアップロードして更新させる手順が問われる可能性があります。
*   **Security Intelligence への組み込み**: ACP ルールよりも上位の SI レベルで特定の国をブロックし、パフォーマンスを最適化する構成が求められます。
*   **「Unknown」の扱い**: GeoDB に登録されていない、またはプライベート IP アドレスの場合の挙動（Unknown）をどう処理するかが設計上のポイントです。
*   **例外処理（Whitelist）**: 特定の国をブロックしつつ、その国の中にある特定のビジネスパートナーの IP だけを許可する「例外ルール」の作成スキルが必要です。
*   **VPN アクセス制限**: AnyConnect（Remote Access VPN）の接続元を特定の国に制限する設定と、ACP でのジオロケーション制御の組み合わせ。

---

## 🛠 設定方法

### 1. ジオロケーションデータベースの更新 (FMC GUI)
1.  **System > Updates > Geolocation Updates** に移動します。
2.  **Upload Update** をクリックして最新のバイナリを選択するか、スケジュール更新を設定します。

### 2. アクセスコントロールルールでの利用
1.  **Policies > Access Control** でルールを追加/編集。
2.  **Geolocation** タブを選択。
3.  左側のリストから「国（Countries）」または「大陸（Continents）」を選択し、**Source/Destination** に追加して保存・デプロイします。

### 3. Security Intelligence での利用
1.  **Policies > Access Control** の対象ポリシーの **Security Intelligence** タブを開きます。
2.  **Networks** セクションで **Geolocation** を選択。
3.  ブロックしたい国を選択して **Blocklist** に移動します。

---

## 🔍 検証コマンド

| 目的 | コマンド（FTD CLI） |
| :--- | :--- |
| **GeoDB のバージョン確認** | <code>show geolocation version</code> |
| **特定の IP の所属国を確認** | <code>show geolocation ip [IP_ADDRESS]</code> |
| **Snort での判定デバッグ** | <code>system support firewall-engine-debug</code> (国名マッチングが表示される) |
| **統計情報の表示** | <code>show statistics security-intelligence</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 特定の国をブロックしても通信が通る | GeoDB が古い | <code>show geolocation version</code> で日付を確認。FMC から手動更新を実行。 |
| 内部通信がドロップされる | Unknown のブロック | プライベート IP が Geolocation ルールにマッチしていないか確認。 |
| イベントログに国名が出ない | ロギング設定の不備 | ACP ルールの Logging タブで <code>Log at Beginning</code> が有効か確認。 |
| SI でのブロックが効かない | オブジェクトの未デプロイ | FMC でデプロイステータスを確認。SI オブジェクトが正しく適用されているか再確認。 |

---

## ⚠ 制限事項

*   **正確性の限界**: IP アドレスの所有権移転やモバイルネットワークの特性により、100% 正確な位置特定は不可能です。
*   **パフォーマンス**: 大量の国を SI でブロックすると、メモリ消費が増加します。
*   **プライベート IP**: RFC 1918 アドレスは GeoDB には含まれず、通常は特定不能（None/Unknown）となります。

---

## 🔄 他技術との関連

*   **Security Intelligence (SI)**: ジオロケーションを「フィード」の一つとして利用し、パケット処理の極めて早い段階でドロップします。
*   **Reporting**: ダッシュボードやレポートで「攻撃がどの国から来ているか」をグラフ化するために、ジオロケーションデータが利用されます。
*   **Remote Access VPN**: 接続試行元の IP をチェックし、許可されていない国からの VPN 接続を拒否します。

---

## 🧩 比較表

### Security Intelligence vs Access Control Rule (Geolocation)

| 特徴 | Security Intelligence (SI) | Access Control Rule (ACP) |
| :--- | :--- | :--- |
| **処理順序** | **極めて早い（プリフィルタ直後）** | SI および SSL 復号の後 |
| **CPU 負荷** | 低い（メモリ検索のみ） | 中（複雑なルール評価が必要） |
| **アクション** | Block (Drop) / Monitor | Allow / Block / Trust |
| **主な用途** | 悪意のある地域の一括排除 | きめ細やかなビジネス要件の制御 |

---

## 💡 ベストプラクティス

1.  **SI で大陸単位のブロック**: 明らかに通信の必要がない大陸（例: 業務外の地域）は、SI レベルで一括遮断し、後続の ACP 処理負荷を軽減します。
2.  **重要なサーバーには「国内限定」**: 公開サーバーへのアクセスを日本国内の IP のみに制限することで、海外からの無差別スキャンを 90% 以上削減できます。
3.  **GeoDB 更新の自動化**: 最低でも週に一度は GeoDB を自動更新するようスケジュールを設定します。
4.  **ホワイトリストの併用**: ジオロケーションブロックを行う際は、必ず重要なパートナーの IP アドレスを SI ホワイトリストに登録し、誤検知を回避します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 特定国（例: 北朝鮮）からの全通信遮断
*   **要件**: 北朝鮮（North Korea）からの全てのインバウンド通信を SI で遮断せよ。
*   **設定**: ACP > Security Intelligence > Networks > Geolocation > North Korea を Blocklist へ。

### 2. 特定の宛先国へのデータ転送制限
*   **要件**: 内部ネットワークから特定の国（例: 競合国）への FTP 通信を遮断せよ。
*   **設定**: ACP ルールを作成。Source: Any, Destination Geolocation: [Country], Service: FTP, Action: Block.

### 3. 国内限定 Web 公開
*   **要件**: DMZ の Web サーバー（10.1.1.100）への HTTPS アクセスを、日本（Japan）からの通信のみ許可し、他を拒否せよ。
*   **設定**: ルール 1: Destination IP: 10.1.1.100, Source Geo: Japan, Action: Allow。ルール 2: Destination IP: 10.1.1.100, Action: Block。

### 4. ジオロケーションデータベースの手動アップデート
*   **要件**: インターネット接続のない FMC で、提供された `Cisco_Geolocation_Update-XXX.pkg` を使用して DB を更新せよ。
*   **操作**: System > Updates > Geolocation Updates > Browse & Upload.

### 5. Geo ログのカスタムアラート
*   **要件**: ロシア（Russia）からの通信を検知した際に、即座に SNMP トラップを送信せよ。
*   **設定**: Geolocation を条件にした Monitor ルールを作成し、Logging タブで SNMP Alert を紐付け。

### 6. VPN 接続のジオフィルタリング
*   **要件**: RA VPN ユーザーが海外旅行中に接続できないよう、日本国内からの接続のみを許可せよ。
*   **設定**: Outside インターフェイスに適用される ACP SI で日本以外を Block。

### 7. GeoDB による特定 IP の調査
*   **課題**: IP `8.8.8.8` が GeoDB 上でどの国に判定されるか、FTD CLI で確認せよ。
*   **実行**: `show geolocation ip 8.8.8.8`

### 8. ジオロケーションと URL フィルタリングの組み合わせ
*   **要件**: 特定の国からのアクセスかつ「SNS カテゴリ」への通信をブロックせよ。
*   **設定**: ACP ルールで Geolocation と Category タブの両方を指定。

### 9. 未知の IP（Unknown）の統計取得
*   **要件**: GeoDB で判定できない「Unknown」の通信量をモニターせよ。
*   **設定**: Geolocation タブで「Unknown」を選択した Monitor ルールを作成。

### 10. クラスタ環境での Geo 同期確認
*   **要件**: FTD クラスタの全ユニットで GeoDB バージョンが一致しているか確認せよ。
*   **コマンド**: `show geolocation version` を各ユニットで実行。

---

## ❓ 想定試験問題

1.  **実装**: FMC において、日本以外の全ての国からの SSH アクセスを、アクセスコントロールルールの評価前に効率的にブロックする構成を完了しなさい。
2.  **トラブルシュート**: 特定の国をブロックするルールを作成したが、その国に属するはずの新しい IP アドレスからの通信が通過してしまう。確認すべき FMC の設定箇所は？
    *   **回答**: System > Updates > Geolocation Updates で、GeoDB が最新の状態に更新されているか確認する。
3.  **Design**: ジオロケーションフィルタリングにおいて、社内ネットワーク（10.0.0.0/8）が「Unknown」と判定され、ブロックルールに巻き込まれるのを防ぐための最適な方法は？
    *   **回答**: 内部ネットワークオブジェクトを Security Intelligence のホワイトリスト（Whitelist）に登録する。
4.  **コンフィグ読解**: `show geolocation ip 1.1.1.1` の出力が `Country: Australia` と表示された。この情報を ACP ルールで利用するために、FMC 上でどのタブを設定すべきか？
    *   **回答**: Access Control Rule の「Geolocation」タブ。
5.  **実装**: インバウンド攻撃を国別で可視化するために、FMC ダッシュボードで「攻撃元国別トップ 10」を表示するウィジェットを追加する手順を述べよ。

---

## 🔗 参考リソース

*   **Configuration Guide**:
    *   [Cisco Secure Firewall Management Center Administration Guide, 7.1 - Geolocation](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/objects.html#id_92425)
*   **Technical Notes**:
    *   [Verify and Troubleshoot Geolocation Database Updates on Firepower](https://www.cisco.com/c/en/us/support/docs/security/firepower-management-center/215354-configure-syslog-on-firepower-firewall-m.html)
*   **Cisco Live (Slides)**:
    *   [BRKSEC-2021: Firepower Threat Defense - Packet Flow and Troubleshooting](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-2021) (SI と Geo の順序について)

---

## 📝 **補足（Notes）**  

*   **学習メモ**: ジオロケーションは設定自体は非常に簡単（GUI で国を選ぶだけ）ですが、**「SI（Security Intelligence）」と「ACP ルール」のどちらでやるべきか**という設計上の判断が CCIE ラボでは問われます。
*   **図解**: 常に「パケットが入ってきて、まず SI（Geo）で弾かれ、生き残ったものが ACP ルール（Geo）に行く」という、より早い段階での遮断を意識してください。
*   **注意点**: ラボ試験中に GeoDB 更新を実行すると、配布に数分かかることがあります。その間、他のタスクを進めるなどの時間管理が重要です。
