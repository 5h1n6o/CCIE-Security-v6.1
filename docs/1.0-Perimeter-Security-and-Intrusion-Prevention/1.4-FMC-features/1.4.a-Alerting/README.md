---
layout: default
title: 1.4.a-Alerting
nav_order: 1
parent: 1.4-FMC-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.4 Cisco FMC features
# 1.4.a Alerting

Cisco Secure Firewall Management Center (FMC) における**アラート機能（Alerting）**は、システムが検知した脅威イベント、ヘルスステータスの変化、または特定のネットワーク動向を、外部システムや管理者に即座に通知する重要な機能です。CCIE Securityラボ試験では、適切な外部レスポンスオブジェクトの作成と、ポリシーへの正確な紐付け能力が問われます,。

---

## 📘 概要

*   **機能概要**: 侵入イベント（Intrusion）、マルウェア検知（AMP）、システムヘルス、または相関ルール（Correlation Rules）に基づき、Syslog、SNMPトラップ、Email、HTTPレスポンスなどの手段で外部へ通知を送信します。
*   **利用目的**: セキュリティ侵害の即時把握、SIEM（Security Information and Event Management）へのデータ集約、運用監視の自動化。
*   **利用場面**: 重大な脆弱性攻撃（Impact Level 1/2）の発生時に管理者にメール通知する、あるいは全トラフィックの接続ログを中央ログサーバ（Syslog）に転送する場合などに利用されます,。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **通知手段** | Syslog, SNMP Trap, Email, HTTP Response (Webhook) |
| **主要コンポーネント** | Alert Responses, Correlation Policies, Health Monitoring |
| **メリット** | リアルタイムなインシデントレスポンスの実現、FMC GUIへの常駐不要。 |
| **デメリット** | 設定ミスによる大量のアラート発生（アラートストーム）の懸念。 |
| **対応機種** | 全てのFMC（ハードウェア、仮想、クラウド）, |
| **設計上の注意点** | イベントの種類（ACログ、Intrusion、System）によって設定箇所が異なる点。 |

---

## 🏗 動作原理

FMCは管理下のデバイス（FTD）からイベントを受信し、定義された条件に基づきアラートを生成します。

```text
[ Managed Device (FTD) ]
   ↓ (Event Data via SFTunnel)
[ Cisco FMC ]
   ↓ (Evaluate Correlation / Health / Policy)
[ Alerting Engine ]
   ↓
   ├─ [ Email Server ] → Admin Inbox
   ├─ [ Syslog Server ] → SIEM/Log Collector
   ├─ [ SNMP NMS ] → Operations Dashboard
   └─ [ HTTP Endpoint ] → Webhook / Chat Ops
```

---

## ⚙ 動作シーケンス

1.  **イベント受信**: FTDがトラフィックを検知し、FMCにセキュリティイベントを送信します。
2.  **ポリシーマッチング**: 
    *   **Access Control Policy**: ルールごとのログ設定に紐付いたアラートオブジェクトがトリガーされます。
    *   **Intrusion Policy**: 特定のシグネチャ検知時に外部レスポンスを生成します。
3.  **相関評価 (Optional)**: Correlation Policyが有効な場合、複数のイベントの組み合わせを確認します。
4.  **通知実行**: 定義された「External Response」オブジェクト（Email/Syslog等）を使用して、外部へ情報を送出します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Alert Responsesの作成**: **Policies > Actions > Alerts** から各通知サーバ（Syslog, SNMP, Email）をオブジェクトとして定義する手順が基本です。
*   **侵入イベントの外部通知**: Intrusion Policy内で **Rules > External Responses** を設定する箇所と、FMCのグローバルな **Alerts** 設定の違いを理解しておく必要があります。
*   **システムヘルスアラート**: デバイスのCPU負荷やディスク容量不足をSNMP/Emailで通知する構成は、System管理セクションで頻出です,。
*   **eStreamerの理解**: 大量データの転送には標準アラートではなく、eStreamer APIを使用したSIEM連携が推奨される点を知識として持っておく必要があります。

---

## 🛠 設定方法

### 1. Syslog アラートオブジェクトの作成 (FMC GUI)
1.  **Policies > Actions > Alerts** に移動。
2.  **Create Alert** ドロップダウンから **Create Syslog Alert** を選択。
3.  名前、IPアドレス、ポート（514等）、Facility、Severityを入力して保存。

### 2. アクセスコントロールルールへの適用
1.  **Policies > Access Control** で対象ルールを編集。
2.  **Logging** タブを選択。
3.  **Log at Beginning/End of Connection** を有効にし、**Alerts** セクションで作成したSyslogオブジェクトを選択して保存・デプロイ。

---

## 🔍 検証コマンド

FMCはGUIベースの管理が主ですが、バックエンドやFTD側での確認が有効です。

| 目的 | コマンド（FMC/FTD CLI） |
| :--- | :--- |
| **通知プロセスの動作確認 (FMC)** | <code>tail -f /var/log/messages</code> (外部への送出ログを確認) |
| **FTDからFMCへのイベント送信確認** | <code>show statistics system flow-export</code> |
| **FMC構成ファイルの確認** | <code>cat /etc/sf/responses.conf</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| アラートが外部に届かない | ネットワーク疎通の問題 | FMC CLIから通知先サーバへ <code>ping</code> または <code>telnet [port]</code> で疎通確認。 |
| Email通知が来ない | SMTPリレーの拒否 | FMCの **System > Configuration > Email Notification** 設定とメールサーバのログを確認。 |
| Syslogが重複する | 冗長な設定 | ACルールとIntrusion Policyの両方で同じSyslogサーバを指定していないか確認。 |
| ヘルスアラートが飛ばない | モニタリング間隔の不備 | **System > Health > Monitor** で対象項目のチェックが有効か確認。 |

---

## ⚠ 制限事項

*   **ライセンス**: FMCの全機能を利用するには、対象デバイスに適切なライセンス（Threat, Malware等）が割り当てられている必要があります。
*   **スループット**: 非常に大量のアラートをEmailで送信すると、FMCのキューが溜まり、パフォーマンスに影響を与える可能性があります。
*   **プロトコル制限**: FMC 7.x以降、セキュアなアラート送出のためにTLSを使用したSyslog等の要件が厳格化されています。

---

## 🔄 他技術との関連

*   **eStreamer**: FMCから外部クライアントへバイナリストリームでイベントをプル/プッシュする高度なモニタリングプロトコル。
*   **Correlation Policies**: 「侵入検知 + 脆弱性あり」のような条件（ホワイトボード/ホストプロファイル）を組み合わせて高度なアラートを生成します。
*   **Health Monitoring**: デバイスのハードウェア状態を監視し、異常時にアラートをトリガーします。

---

## 🧩 比較表

### FMC Standard Alerting vs eStreamer

| 機能 | Standard Alerting (Syslog/SNMP) | eStreamer |
| :--- | :--- | :--- |
| **方式** | プッシュ型 (リアルタイム) | プル/プッシュ型 (APIベース) |
| **データ量** | 中〜低 | 非常に高い (フルイベントデータ) |
| **主な用途** | 即時通知、簡易ログ保管 | 高度な分析、SIEM統合 |
| **設定の難易度** | 低 (GUIのみ) | 高 (証明書/クライアント実装が必要) |

---

## 💡 ベストプラクティス

*   **アラートのフィルタリング**: 全ての接続ログをEmailで送るのではなく、重大な脅威（Critical）のみをEmailにし、その他はSyslogに集約します。
*   **相関ルールの活用**: ノイズを減らすため、特定のホストに対する連続した攻撃など、意味のあるイベントの組み合わせに対してのみアラートを設定します。
*   **テスト通知の実行**: オブジェクト作成直後に「Test」ボタンを使用して、設定内容の有効性を必ず確認します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 重大な侵入イベントのEmail通知
*   **要件**: Impact Level 1の侵入イベントが発生した際、`sec-admin@example.com` にメールを送信せよ。
*   **設定**: `Policies > Actions > Alerts` でEmailサーバ設定後、`Intrusion Policy > Rules > External Responses` に適用。

### 2. ACルールによる特定の通信監視
*   **要件**: DMZサーバへの全SSHアクセスをSyslogサーバ `10.1.1.50` に通知せよ。
*   **設定**: ACルールのLoggingタブでSyslogアラートを選択。

### 3. FTDディスク容量不足のアラート
*   **要件**: デバイスのディスク使用率が90%を超えたらSNMPトラップを送信せよ。
*   **設定**: `System > Health > Monitor` 内のディスク容量閾値を調整し、Health PolicyでSNMP通知を紐付け。

### 4. マルウェア検知のWebhook通知
*   **要件**: AMPでマルウェアを検知した際、外部HTTPエンドポイントに情報を送信せよ。
*   **設定**: HTTP Responseオブジェクトを作成し、マルウェアポリシーに紐付け。

### 5. 相関ルール：特定サブネットへのスキャン検知
*   **要件**: 内部ネットワークからのポートスキャンを検知した際にアラートを生成せよ。
*   **設定**: Correlation Ruleで「Port Scan」イベントを条件に定義。

### 6. eStreamerクライアントの追加
*   **要件**: IP `10.1.1.100` のサーバがeStreamer経由でイベントを取得できるようにせよ。
*   **設定**: `System > Integration > eStreamer` でクライアントを追加し、証明書を発行。

### 7. SNMPv3 トラップの構成
*   **要件**: 認証と暗号化（SHA/AES）を使用したSNMPv3トラップをNMSへ送信せよ。
*   **設定**: Alert ResponseでSNMPv3のUser/EngineID等を定義。

### 8. アラートスロットリング（Thresholding）
*   **要件**: 同一イベントによる通知を10分間に1回に制限せよ。
*   **設定**: Intrusion PolicyのThresholding設定で抑制。

### 9. ヘルスステータス：VPNトンネルダウンの通知
*   **要件**: 拠点間VPNが切断された際に即座にSyslogを送信せよ。
*   **設定**: Health MonitoringのVPNステータス項目をアラート対象に含める。

### 10. カスタムSyslogフォーマット
*   **要件**: 送信されるSyslogに「CUSTOMER_ABC」というタグを付与せよ。
*   **設定**: Syslog Alertオブジェクト内のメッセージタグフィールドを編集。

---

## ❓ 想定試験問題

1.  **実装**: FTDで検知された「Critical」レベルのマルウェアイベントのみを、特定の管理者にEmailで通知する相関ポリシーを構成しなさい。
2.  **トラブルシュート**: FTDの接続ログをSyslogサーバに転送するようACルールを設定したが、サーバ側でログが受信できない。FMC側で確認すべきプロセスとログファイルは何か？
    *   **回答**: `/var/log/messages` を確認し、`sfmgr` または `alert` 関連のプロセスが外部疎通エラーを出していないか特定する。
3.  **Design**: 毎秒10,000イベントが発生する環境で、全ての詳細情報を外部SIEMに送る必要がある。SyslogアラートではなくeStreamerを使用すべき理由は何か？
    *   **回答**: Syslogはパケットオーバーヘッドが大きくFMCのCPUを消費するが、eStreamerはバイナリ形式で効率的に大量データを転送するために設計されているため。
4.  **実装**: FMCのヘルスモニターで、特定のFTDユニットのみアラート対象から除外する方法を述べなさい。
5.  **コンフィグ読解**: `responses.conf` の中身が提示された際、どの外部サーバに対してどのタイプのイベントが紐付けられているかを読み解く。

---

## 🔗 参考リソース

*   **Configuration Guide**:
    *   [Cisco Secure Firewall Management Center Administration Guide, 7.1 - External Alerting](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/external_alerting_for_intrusion_events.html)
*   **Cisco Live (Slides)**:
    *   [BRKSEC-3020: Troubleshooting Firewall Threat Defense (FTD)](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-3020)
*   **Technical Notes**:
    *   [Configuring Syslog for Firepower Management Center](https://www.cisco.com/c/en/us/support/docs/security/firepower-management-center/215354-configure-syslog-on-firepower-firewall-m.html)
*   **eStreamer SDK**:
    *   [Cisco Firepower eStreamer Integration Guide](https://www.cisco.com/c/en/us/td/docs/security/firepower/60/api/estreamer/EventStreamerGuide.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: FMCのアラートは「どこで設定するか」が迷いやすいポイントです。ACルールならACポリシーのLogging、システム全体ならActions > Alerts、ヘルスならHealth Monitorと、目的別に場所を整理して覚えましょう。
*   **注意点**: ラボ試験では、通知先サーバのIPアドレスやポート番号の指定ミスが致命的な減点に繋がります。設定後のデプロイと「Test Alert」による実動作確認を怠らないでください。
