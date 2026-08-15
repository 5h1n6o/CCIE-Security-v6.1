---
layout: default
title: 5.7-Email-security
nav_order: 7
parent: 5.0-Advanced-Threat-Protection
---

# 5.7 Email security features

Cisco Secure Email (旧 Cisco Email Security Appliance, ESA) は、電子メールを介した脅威（スパム、フィッシング、マルウェア、データ漏洩）から組織を保護するための専用アプライアンスです。CCIE Security v6.1 では、ESA の高度な脅威防御機能、コンテンツフィルタリング、および認証技術（SPF/DKIM/DMARC）の理解と実装が求められます。

---

## 📘 概要

*   **機能概要**: メールの送受信プロセスにおいて、送信元の信頼性評価、添付ファイルのウイルス/マルウェアスキャン、本文のスパム解析、および機密情報の流出防止（DLP）を多層的に実行する機能です。
*   **利用目的**: 標的型攻撃（BEC: ビジネスメール詐欺）の阻止、スパムによる生産性低下の防止、およびコンプライアンス遵守。
*   **どのような場面で利用するか**:
    *   **外向き（Outgoing）**: 社内から外部へ送信されるメールに機密情報が含まれていないか検査し、必要に応じて暗号化や遮断を行う。
    *   **内向き（Incoming）**: 外部から届くメールをスキャンし、悪意のあるリンクやファイルを排除する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **HAT (Host Access Table)** | 送信元 IP アドレスに基づき、接続を許可（Whitelist）、拒否（Blacklist）、またはスロットリングする。 |
| **RAT (Recipient Access Table)** | 自社が受信を許可するドメインを指定し、オープンリレーを防ぐ。 |
| **Mail Policies** | ユーザやグループごとに異なるセキュリティレベル（スパム設定等）を適用する。 |
| **Anti-Spam / Anti-Virus** | CASE エンジンによるスパム判定、および Sophos/McAfee によるウイルス検知。 |
| **AMP for Email** | 添付ファイルのハッシュ判定、サンドボックス解析、レトロスペクティブ検知。 |
| **Content Filters** | 条件（Condition）とアクション（Action）を組み合わせ、メールヘッダーや本文を柔軟に制御。 |
| **DLP (Data Loss Prevention)** | クレジットカード番号やマイナンバーなどの機密情報を自動検知。 |

---

## 🏗 動作原理

ESA は「Eメール・パイプライン」と呼ばれる一連の処理プロセスを通じてパケットを処理します。

```text
[ Sender MTA ]
      ↓
(1) Connection Level: HAT (SBRS/IP Reputation)
      ↓
(2) SMTP Level: RAT (Recipient Verification)
      ↓
(3) Email Pipeline:
    - Advanced Malware Protection (AMP)
    - Anti-Spam (CASE)
    - Anti-Virus (Sophos/McAfee)
    - Graymail Detection
    - Content Filters (Header/Body)
    - DLP (Data Loss Prevention)
      ↓
(4) Delivery: Queueing and Sending to Destination MTA
```

---

## ⚙ 動作シーケンス

1.  **接続確立**: 送信元サーバが ESA に TCP 25 番ポートで接続。ESA は **HAT** を参照し、送信元 IP の評価（SenderBase Reputation Score）に基づき接続を継続するか判断します。
2.  **エンベロープ確認**: `RCPT TO` コマンドを受け取ると、ESA は **RAT** を参照し、その宛先ドメインが自身で受け入れるべきものか確認します。
3.  **セキュリティスキャン**:
    *   **Anti-Spam**: メールの「評判」を分析。
    *   **Anti-Virus**: 既知のウイルスシグネチャと照合。
    *   **AMP**: 添付ファイルの SHA-256 を計算しクラウドへ照会。
4.  **フィルタ処理**: 管理者が定義した **Content Filters** や **DLP Policies** に基づき、特定キーワードの検知やヘッダーの書き換えを実行します。
5.  **配送**: すべての検査をパスしたメール、または「隔離（Quarantine）」から解放されたメールが宛先のメールサーバへ配送されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **HAT Sender Group の構成**: `TRUSTED`, `BLOCKED`, `SUSPECT` などの送信元グループに特定の IP やドメインを正しく割り当てる。
*   **Incoming/Outgoing の切り分け**: メールの方向性によって適用されるリスナー（Public/Private）とポリシーが異なることを理解する。
*   **SPF/DKIM/DMARC の検証**: 送信元のなりすましを防ぐための DNS ベースの認証設定。
*   **Dictionary の作成**: Content Filter で使用するキーワードリスト（正規表現含む）の作成。
*   **Forged Email Detection (FED)**: 役員の名前を騙ったなりすましメールを検知するための構成。
*   **トラブルシュート**: `mail_logs` の読解。特に `ICID` (Incoming Connection ID), `MID` (Message ID), `DCID` (Delivery Connection ID) を追跡してどこでメールが止まったか特定する。

---

## 🛠 設定方法

### 1. HAT による特定の送信元拒否 (GUI)
1.  **Mail Policies > HAT Overview** に移動。
2.  `BLACKLIST` 送信元グループを選択。
3.  **Add Sender**: 拒否したい IP アドレス（例: `192.168.10.1`）を追加。
4.  ポリシー設定で Action が `REJECT` になっていることを確認。

### 2. コンテンツフィルタによる実行ファイルブロック
1.  **Mail Policies > Incoming Content Filters**。
2.  **Add Filter**: `Conditions` に `Attachment File Info` > `File Extension` > `matches .exe` を追加。
3.  `Actions` に `Strip Attachment` または `Quarantine` を追加。
4.  **Mail Policies > Incoming Mail Policies** で該当フィルタを有効化。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **リアルタイムログ監視** | <code>tail mail_logs</code> |
| **特定 MID の検索** | <code>grep "MID 123" mail_logs</code> |
| **リスナー状態確認** | <code>status detail</code> |
| **送信元レピュテーション確認** | <code>reputation [IP]</code> |
| **キューの状態確認** | <code>tophosts</code>, <code>workqueue</code> |
| **ネットワーク疎通確認** | <code>telnet [Destination_IP] 25</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| メールが届かない | HAT で拒否されている | `mail_logs` で `Rejected by HAT` の記録がないか確認。 |
| 外部へ送信できない | RAT 設定ミス | 外部ドメインが `RELAYLIST` に含まれているか確認。 |
| スパムが検知されない | CASE エンジン未更新 | `updatenow` コマンドでエンジンの最新化を確認。 |
| AMP 解析が動作しない | クラウド通信不可 | `ping amp.cisco.com` とライセンス状態を確認。 |
| メールの遅延 | Workqueue の詰まり | `status` でキューの量を確認。DNS 引っ掛かりや AV スキャンの過負荷。 |

---

## ⚠ 制限事項

*   **暗号化メール**: S/MIME や PGP で暗号化されたメールは、ESA で中身（ペイロード）をスキャンできません。
*   **パスワード保護ファイル**: 暗号化された ZIP ファイル内は AMP や AV でのスキャンが制限されます。
*   **ハードウェアリソース**: 大量のログ出力や詳細な正規表現フィルタは、CPU 負荷を増大させ、メール処理能力を低下させます。

---

## 🔄 他技術との関連

*   **5.1 Cisco AMP**: ESA に内蔵された AMP 機能がファイルを抽出し、ネットワーク全体の脅威情報と共有します。
*   **4.0 Identity Management**: LDAP (Active Directory) と連携し、宛先アドレスの有効性確認やユーザ属性に基づいたポリシー適用を行います。
*   **3.6 Monitoring**: `eStreamer` や `Syslog` を使用して、検知したセキュリティイベントを監視サーバへ転送します。

---

## 🧩 比較表

### HAT (Incoming) vs RAT (Outgoing/Relay)

| 特徴 | HAT (Host Access Table) | RAT (Recipient Access Table) |
| :--- | :--- | :--- |
| **判定基準** | 送信元サーバの **IP アドレス / ホスト名** | 送信先の **メールアドレス / ドメイン** |
| **主な目的** | 送信元の信頼性（接続許可）の制御 | 不正中継（リレー）の防止 |
| **処理タイミング** | SMTP セッションの開始直後 | `RCPT TO` コマンド受信時 |
| **主要アクション** | ACCEPT, REJECT, SUSPECT, RELAY | ACCEPT, REJECT |

---

## 💡 ベストプラクティス

1.  **SBRS の活用**: SenderBase レピュテーションが低い（例: -10 〜 -3.0）送信元は、詳細検査せずに HAT で即座にドロップまたはスロットリングすることでリソースを節約します。
2.  **多層防御**: アンチウイルスは必ず 2 つのエンジン（Sophos + McAfee 等）を有効にし、検知漏れを防ぎます。
3.  **DLP 通知**: 内部ユーザが機密情報を送ろうとした際は、単に遮断するだけでなく、本人にポリシー違反である旨を通知して教育効果を高めます。

---

## 📝 ラボ学習・設定サンプル例

### 1. HAT Whitelist 設定
*   **要件**: パートナー企業 (10.1.50.10) からのメールは、レピュテーションに関わらず常に許可せよ。
*   **設定**: `Mail Policies > HAT Overview > TRUSTED` に 10.1.50.10 を追加。

### 2. RAT ドメイン許可設定
*   **要件**: 自社ドメイン `example.com` 宛のメールのみ受信を許可せよ。
*   **設定**: `Mail Policies > RAT > Add Recipient` で `example.com` を追加。

### 3. 送信元認証 (SPF) の有効化
*   **要件**: 送信元の SPF レコードを検証し、Fail 判定の場合はヘッダーに `X-SPF-Status: Fail` を挿入せよ。

### 4. Forged Email Detection (FED)
*   **要件**: 表示名に自社 CEO の「Alice Smith」を含む外部メールを隔離せよ。
*   **設定**: Content Filter で `Condition: Envelope Sender` と `Dictionary` を使用。

### 5. 添付ファイルサイズ制限
*   **要件**: 10MB を超える添付ファイルを持つメールを拒否せよ。
*   **設定**: Content Filter > `Condition: Attachment Size > 10M` > `Action: Bounce`.

### 6. 受信者フィルタリング (LDAP)
*   **要件**: AD に存在しない宛先へのメールを SMTP レベルで拒否せよ。
*   **設定**: `System Administration > LDAP` でクエリを構成し、リスナーで有効化。

### 7. スパム判定のカスタマイズ
*   **要件**: スパム判定スコアが 90 以上のメールは即時削除せよ。
*   **設定**: `Incoming Mail Policy > Anti-Spam > Positively Identified Spam Action`.

### 8. Graymail (マーケティングメール) の隔離
*   **要件**: ニュースレターを検知し、別の隔離フォルダへ移動せよ。

### 9. 特定拡張子のマクロ排除
*   **要件**: `.doc` ファイル内にマクロが含まれている場合、添付ファイルを削除せよ。

### 10. クラスタリング構成
*   **要件**: 2 台の ESA 間で設定を同期するように構成せよ。
*   **CLI**: <code>clusterconfig</code>。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: HAT で `RELAYLIST` に 192.168.1.0/24 が登録されている。このネットワークの端末から外部ドメイン宛にメールを送った際、RAT は参照されるか？
    *   **回答**: いいえ。`RELAYLIST` にマッチした場合、その接続は信頼されたリレーとして扱われ、RAT の受信者チェックはバイパスされます。
2.  **トラブルシュート**: 特定のドメインからのメールが全て「スパム」と誤判定される。最小限の影響でこれを回避する設定は？
    *   **回答**: HAT の `TRUSTED` グループにそのドメインを登録するか、Whitelist 用の `Mail Policy` を作成する。
3.  **Design**: CEO の名前を語る「なりすまし」メールが届いている。送信元 IP は頻繁に変わる。最も効果的な ESA の機能は？
    *   **回答**: **Forged Email Detection (FED)** および **DMARC 検証**。
4.  **実装**: メールの本文にある URL が、送信時にはクリーンだったが後にフィッシングサイトに変わった。これを検知できる機能は？
    *   **回答**: **Outbreak Filters** による動的 URL スキャン、または **AMP Retrospective**。
5.  **トラブルシュート**: `mail_logs` に `ICID 500 REJECT` と表示されている。これはどのテーブルで拒否されたことを示しているか？
    *   **回答**: **HAT (Host Access Table)**。IP アドレス接続レベルでの拒否。

---

## 🔗 参考リソース

*   **Cisco Secure Email (ESA) Configuration Guide**: [User Guide for AsyncOS 11.1](https://www.cisco.com/c/en/us/td/docs/security/esa/esa11-1/user_guide/b_ESA_Admin_Guide_11_1.html)
*   **Cisco Live (BRKSEC-2041)**: [Email Security Best Practices](https://www.ciscolive.com/)
*   **Cisco Validated Design (CVD)**: [Email Security with Content Filters](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/ise-deployment-guide.html)
*   **Technical Note**: [Understanding ESA mail_logs](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117970-technote-esa-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: ESA は「住所(IP)を見る HAT」と「宛名(ドメイン)を見る RAT」の二段階チェックが基本です。これをパスして初めて「中身(本文・添付ファイル)」を見るパイプラインに入ります。
*   **図解**: パイプラインは工場のベルトコンベアのようなものです。各工程（Spam, AV, AMP, DLP）で不良品が弾かれます。
*   **注意点**: ラボ試験では、**設定変更後に「Commit Changes」を忘れずに行う**ことが非常に重要です。設定しても Commit しなければ反映されません。
