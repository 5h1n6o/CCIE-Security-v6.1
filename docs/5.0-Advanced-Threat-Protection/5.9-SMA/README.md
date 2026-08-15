---
layout: default
title: 5.9-SMA
nav_order: 9
parent: 5.0-Advanced-Threat-Protection
---

# 5.9 Cisco SMA for centralized content security management

Cisco SMA（Security Management Appliance / 旧 M-Series）は、複数の **Cisco Secure Email (ESA)** および **Cisco Secure Web (WSA)** アプライアンスの管理、レポーティング、およびトラッキング機能を一元化するための専用アプライアンスです。大規模な環境において、分散した複数のセキュリティデバイスから生成される膨大なログを収集・分析し、管理者の運用負荷を大幅に軽減します。

---

## 📘 概要

*   **機能概要**: 複数の ESA/WSA からデータを集約し、一箇所で「メッセージ追跡（Message Tracking）」「レポーティング」「スパム隔離（Centralized Quarantine）」を管理する機能を提供します。
*   **利用目的**: 管理コンソールの統合、長期間のデータ保持、複数のゲートウェイを跨いだ脅威情報の相関分析。
*   **どのような場面で利用するか**:
    *   **集中型レポーティング**: 組織全体の Web/Email トラフィック統計を単一のダッシュボードで確認したい場合。
    *   **集中型メッセージ追跡**: 特定のメールがどの ESA を通過したかに関わらず、組織全体で検索したい場合。
    *   **集中型スパム隔離**: ユーザーが 1 つのポータルにログインするだけで、自分宛のすべての隔離メールを確認できるようにしたい場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な役割** | レポーティング、メッセージトラッキング、集中隔離、ポリシールールの同期。 |
| **対応デバイス** | Cisco ESA (Email Security), Cisco WSA (Web Security)。 |
| **接続プロトコル** | SSH (TCP 22), Reporting/Tracking 用 (TCP 2222), HTTPS (TCP 443)。 |
| **データ転送** | ESA/WSA から SMA へデータを「プッシュ」または SMA が「プル」する。 |
| **隔離機能** | スパム隔離 (ISQ) および ポリシー隔離 (PVO) を SMA 上でホスト可能。 |
| **ライセンス** | SMA 本体ライセンスに加え、集中管理するノード数に応じたライセンスが必要。 |

---

## 🏗 動作原理

SMA は、各セキュリティアプライアンス（ESA/WSA）とセキュアな通信（SSH）を確立し、管理データとログデータを同期します。

```text
[ ESA 1 ] --- (Reporting/Tracking Data) ---> [ Cisco SMA ] <--- [ Administrator ]
[ ESA 2 ] --- (Centralized Quarantine)  ---> [ (Storage) ]      (HTTPS Console)
                                              [ (Analysis) ]
[ WSA 1 ] --- (Web Reporting Data)      ---> [ (Indexing) ]
[ WSA 2 ] --- (Web Reporting Data)      ---> [           ]
```

---

## ⚙ 動作シーケンス

1.  **接続確立**: ESA/WSA と SMA 間で SSH 公開鍵の交換を行い、信頼関係を構築します。
2.  **サービス有効化**: 各アプライアンス側で「Centralized Reporting/Tracking」を有効にします。
3.  **データプッシュ**: ESA/WSA は、生成されたメッセージトラッキングログやレポートログを SMA の TCP 2222 ポートへ転送します。
4.  **インデックス作成**: SMA は受信したログをリアルタイムでデータベースに登録し、高速な検索インデックスを作成します。
5.  **隔離転送**: ESA がスパムやポリシー違反メールを検知すると、メール本体を SMA の隔離領域へ転送（オフロード）します。
6.  **ユーザーアクセス**: エンドユーザーは SMA の Web ポータル（EUQ）にアクセスし、自身の隔離メールを操作します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **接続設定の不備**: 最も多いミスは、ESA/WSA 側での「Management Appliance」設定において、SSH ホストキーの承認を忘れることです。
*   **ポート番号の理解**: 標準の SSH ポート (22) だけでなく、レポート用サービスポート (2222) がファイアウォールで許可されている必要があります。
*   **集中隔離 (Centralized Quarantine)**: ラボ試験では、ESA 上のローカル隔離を無効化し、SMA への転送を構成するシナリオが頻出です。
*   **メッセージ追跡の検証**: 特定の MID (Message ID) を SMA 上で検索し、配送ステータスが正しく表示されるかを確認するスキルが求められます。
*   **認証設定**: 集中隔離ポータルへのログインに LDAP (Active Directory) を使用する場合、SMA 側で LDAP プロファイルとクエリを構成する必要があります。

---

## 🛠 設定方法

### 1. SMA 側：管理ノードの登録 (GUI)
1.  **Management Appliance > Centralized Services > Security Appliances**。
2.  **Add Appliance** をクリック。
3.  ESA/WSA の IP アドレスと、接続に使用する管理者パスワードを入力。
4.  **Test Connection** を実行し、SSH 公開鍵を承認する。

### 2. ESA 側：集中レポーティングの有効化
1.  **Management Appliance > Centralized Services**。
2.  **Reporting / Message Tracking** を `Edit Settings` で有効化。
3.  登録した SMA を送信先として選択。

### 3. スパム隔離の集中化設定 (SMA)
1.  **Email > Spam Quarantine > Enable**。
2.  **External Spam Quarantine** を有効にし、ESA からの接続を許可。
3.  ESA 側の **Management Appliance > Centralized Services > Spam Quarantine** で SMA を指定。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **SMA との接続状態確認** | ESA CLI: <code>smastatus</code> |
| **トラッキングログの送信確認** | ESA CLI: <code>tail reporting_logs</code> / <code>tail tracking_logs</code> |
| **SMA のシステム状態確認** | SMA CLI: <code>status detail</code> |
| **疎通テスト** | <code>telnet [SMA_IP] 2222</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| SMA でレポーティングが出ない | 通信ポート TCP 2222 の遮断 | ESA と SMA 間の FW ルールを確認。 |
| "Connection Failed" エラー | SSH ホストキーの不一致 | `sshconfig` で古いホストキーを削除し、再登録。 |
| メッセージ検索が遅い | インデックスの破損 | <code>logconfig > tracking > reindex</code> を実行（CLI）。 |
| ユーザーが EUQ にログイン不可 | LDAP 連携の不整合 | SMA 側の `ldaptest` コマンドで AD との疎通を確認。 |
| データが同期されない | 時刻のズレ (NTP) | 両デバイスの NTP 設定を確認。許容範囲を超えると認証に失敗する。 |

---

## ⚠ 制限事項

*   **バージョン互換性**: SMA の AsyncOS バージョンは、管理対象の ESA/WSA と同等か、それより新しい必要があります。
*   **スループット**: 1 台の SMA が管理できるノード数には制限があり、それを超えるとレポート生成に遅延が生じます。
*   **データ保護**: SMA はログを集約するため、SMA のバックアップ（Config およびデータ）は ESA 単体よりも重要度が高くなります。

---

## 🔄 他技術との関連

*   **5.7.c Quarantine**: ESA で発生する隔離処理の実体（ストレージ）を SMA に移管します。
*   **5.5 Web filtering**: 複数の WSA での Web フィルタリングログを SMA で集計し、組織全体の Web 利用動向を分析します。
*   **4.7 Active Directory**: SMA 上の隔離ポータルへのログイン認証に使用されます。

---

## 🧩 比較表

### SMA (Centralized) vs Local Management

| 特徴 | ローカル管理 (ESA/WSA 単体) | 集中管理 (SMA) |
| :--- | :--- | :--- |
| **レポート範囲** | デバイス 1 台分 | **全デバイス合算** |
| **トラッキング保持期間** | 短い (ディスク容量に依存) | **長い (専用大容量 HDD)** |
| **スパム隔離ポータル** | デバイスごとに個別の URL | **1 つの共通 URL** |
| **運用効率** | デバイス数に比例して負荷増 | デバイスが増えても一定 |

---

## 💡 ベストプラクティス

1.  **時刻同期の徹底**: ログの相関分析を正確に行うため、ESA/WSA/SMA すべてで同一の NTP サーバを参照させます。
2.  **専用ネットワークインターフェイス**: 管理トラフィック（M-Port）とデータトラフィックを物理的または論理的に分離します。
3.  **定期的な再インデックス**: 検索パフォーマンスを維持するため、定期的なメンテナンスを推奨します。

---

## 📝 ラボ学習・設定サンプル例

### 1. ESA を SMA に登録
*   **要件**: SMA (10.1.1.50) から ESA (10.1.1.10) をノードとして追加せよ。

### 2. 集中メッセージトラッキングの有効化
*   **要件**: ESA のメッセージトラッキングを SMA へオフロードせよ。
*   **設定**: ESA > Management Appliance > Centralized Services > Tracking Enable.

### 3. Web セキュリティレポートの集約
*   **要件**: WSA1 と WSA2 の Web レポートを SMA で一括表示せよ。

### 4. 集中スパム隔離の設定
*   **要件**: ユーザーが `https://sma.example.com/euq` でスパムを確認できるようにせよ。

### 5. PVO 隔離の集中化
*   **要件**: コンテンツフィルタで隔離されたメールを SMA の `Policy_Q` へ転送せよ。

### 6. SMA 上での LDAP プロファイル作成
*   **操作**: AD (10.1.1.100) を参照する LDAP プロファイルを SMA に作成し、認証をテストせよ。

### 7. SMA への構成バックアップ転送
*   **要件**: 毎日深夜に ESA の設定ファイルを SMA へ自動保存せよ。

### 8. 役割ベースの管理 (RBAC)
*   **要件**: "Helpdesk" グループのユーザーには SMA でのメッセージ追跡のみを許可せよ。

### 9. トラッキングログのアーカイブ
*   **要件**: SMA のディスクを節約するため、30日以上前のログを外部サーバへ退避せよ。

### 10. SMA のリソース監視
*   **操作**: `status detail` を使用して、SMA のデータベース書き込み不可（Wait）が発生していないか確認せよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: ESA で <code>smastatus</code> を実行した際、Tracking サービスが `Disconnected` となっている。確認すべきポートは？
    *   **回答**: SMA 側の **TCP ポート 2222**。
2.  **Design**: 複数の ESA がある環境で、ユーザーがどの ESA から送信されたスパムも 1 つの画面で見られるようにしたい。どの機能を実装すべきか？
    *   **回答**: **Centralized Spam Quarantine**。
3.  **コンフィグ読解**: ESA の `Management Appliance` 設定画面で、SMA の SSH ホストキーを承認しないまま設定を完了した場合、どうなるか？
    *   **回答**: 通信が確立されず、データのアップロードや集中管理が失敗する。
4.  **実装**: SMA で WSA のレポーティングを有効にする際、WSA 側で必要な設定は？
    *   **回答**: **Centralized Reporting** の有効化と SMA への接続プロファイルの構成。
5.  **Design**: SMA 自体が故障した場合、ESA のメール配送に影響が出るか？
    *   **回答**: **通常は出ない**。レポーティングやトラッキングはオフライン処理であり、配送（SMTP）自体は ESA が独立して継続する。

---

## 🔗 参考リソース

*   **Cisco SMA User Guide**: [Security Management Appliance Guide v12.0](https://www.cisco.com/c/en/us/td/docs/security/sma/sma12-0/user_guide/b_SMA_Admin_Guide_12_0.html)
*   **Cisco Live (BRKSEC-2041)**: [Centralized Management with Cisco SMA](https://www.ciscolive.com/)
*   **Technical Note**: [Troubleshooting Connection issues between ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118334-technote-sma-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: SMA は「分析官」です。現場（ESA/WSA）の兵士たちが送ってくる報告書（ログ）を整理し、司令官（管理者）に読みやすく提示します。
*   **図解**: 
    - ESA: ログ生成
    - SMA: ログ保存・検索
    - SSH (2222): 報告書を送る専用のバイパス道路
*   **注意点**: ラボ試験では、**ESA 側の設定と SMA 側の設定を両方行う必要がある**（片方だけでは通信が始まらない）点に十分注意してください。
