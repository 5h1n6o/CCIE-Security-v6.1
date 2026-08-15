---
layout: default
title: 5.1-AMP
nav_order: 1
parent: 5.0-Advanced-Threat-Protection
---

# 5.1 Cisco AMP for networks, Cisco AMP for endpoints, and Cisco AMP for content security (Cisco ESA, and Cisco WSA)

Cisco AMP（Advanced Malware Protection：現在は Cisco Secure Endpoint/Cisco Secure Firewall Malware Defense 等に改称）は、攻撃の「前・中・後」の全ての段階でマルウェアを検知、ブロック、および追跡するための包括的なセキュリティソリューションです。ネットワーク（FTD/ASA）、エンドポイント（PC/サーバ）、およびコンテンツ（WSA/ESA）の各レイヤで、クラウドベースの脅威インテリジェンスと連携して動作します。

---

## 📘 概要

*   **機能概要**: ファイルのハッシュ値に基づく「ファイルレピュテーション」、サンドボックスによる「動的ファイル解析」、および過去に遡って脅威を通知する「レトロスペクティブ（遡及検知）」を提供する機能です。
*   **利用目的**: 既知のマルウェアの即時ブロックだけでなく、未知の脅威を特定し、感染経路を可視化して被害を最小限に抑えること。
*   **どのような場面で利用するか**:
    *   **AMP for Networks**: Firepower (FTD) を通過するファイルを検査し、ネットワーク境界でマルウェアを阻止する。
    *   **AMP for Endpoints**: デバイス上のプロセスを監視し、OS レベルでの振る舞い検知や感染後の隔離を行う。
    *   **AMP for Content Security**: 電子メール（ESA）や Web プロキシ（WSA）を通じて流入する添付ファイルやダウンロードファイルをスキャンする。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **ファイルレピュテーション** | SHA-256 ハッシュをクラウド (Cisco Talos) に照会し、数ミリ秒で判定。 |
| **動的ファイル解析** | 未知のファイルをサンドボックス (Cisco Threat Grid) で実行し、挙動を分析。 |
| **遡及検知 (Retrospective)** | 一度「クリーン」と判定されたファイルが後に「マルウェア」と判明した際、管理者にアラートを通知。 |
| **ファイルトラジェクトリ** | ネットワーク内のどこでファイルが侵入し、どの端末へ伝播したかを時系列で表示。 |
| **デバイストラジェクトリ** | 特定のエンドポイント内でのプロセス実行やファイル操作の履歴を可視化。 |
| **対応プロトコル** | HTTP, SMTP, FTP, SMB, POP3, IMAP 等。 |

---

## 🏗 動作原理

AMP は、ローカルデバイスとクラウドエンジンのハイブリッドモデルで動作します。

```text
[ File Transfer ]
       ↓
[ Cisco Device (FTD/WSA/ESA/Endpoint) ]
       ↓ (1) Extract SHA-256 Hash
       ↓ (2) Query Cloud for Reputation
[ Cisco AMP Cloud ]
       ↓ (3) Response: Clean / Malicious / Unknown
       ↓
[ Result Treatment ]
       ↓- Malicious: Block / Alert
       ↓- Clean: Allow
       ↓- Unknown: (4) Submit to Sandbox (Threat Grid)
       ↓
[ Dynamic Analysis ]
       ↓ (5) Behavioral Score & Report
```

---

## ⚙ 動作シーケンス

1.  **ハッシュ抽出**: デバイス（FTD/WSA等）がトラフィックからファイルを抽出し、SHA-256 ハッシュを計算します。
2.  **レピュテーション照会**: デバイスは AMP クラウドにハッシュ値を送信し、レピュテーション（評判）を確認します。
3.  **アクション実行**: 判定が「Malicious（悪意あり）」なら遮断、「Clean」なら通過させます。
4.  **動的解析の申請**: 判定が「Unknown」かつ設定が有効な場合、ファイルを Threat Grid へアップロードしてサンドボックス解析を行います。
5.  **遡及的アラート**: 後にレピュテーションが更新された場合、クラウドからデバイスへ通知が送られ、過去の通信ログに遡ってイベントが生成されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **FMC での File Policy 作成**: `Policies > Access Control > Malware & File` でポリシーを作成し、Block Malware を選択する手順。
*   **WSA での AMP 有効化**: `Security Services > Anti-Malware and AMP` からレピュテーションと解析設定をオンにする手順。
*   **AMP Cloud 接続の確認**: デバイスからクラウド FQDN への到達性（TCP 443）を確認するトラブルシュート問題。
*   **解析対象拡張子の選択**: 実行ファイル（.exe, .dll）や PDF、MS Office ファイルなど、スキャン対象とするカテゴリの正確な指定。
*   **イベント確認**: FMC の `Analysis > Files` または `Analysis > Malware` ダッシュボードから、検知されたファイルの SHA-256 値を確認する問題。

---

## 🛠 設定方法

### 1. Cisco FTD (FMC 管理)：File Policy の構成 (GUI)
1.  **Policies > Malware & File** に移動し **Add File Policy**。
2.  **Add Rule**:
    *   Application Protocol: `Any`
    *   Direction of Transfer: `Any`
    *   Action: `Block Malware`
    *   Check `Spero Analysis` (機械学習) および `Dynamic Analysis` (サンドボックス)。
3.  **Policies > Access Control**: 該当ルールを選択し `Inspection` タブの `File Policy` に作成したポリシーを紐付け。

### 2. Cisco WSA：AMP の有効化 (GUI)
1.  **Security Services > Anti-Malware and AMP** に移動。
2.  **Edit Settings** をクリックし、`Enable File Reputation` および `Enable File Analysis` にチェック。
3.  **Web Security Manager > Access Policies**: 解析対象とするトラフィックのグループに対して、Anti-Malware 設定で AMP を有効化。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **AMP ステータス確認** | FTD/ASA | <code>show amp status</code> |
| **接続確認 (Cloud)** | FTD CLI | <code>ping amp.cisco.com</code> (または該当リージョン) |
| **ファイルスキャン統計** | WSA CLI | <code>ampstatus</code> |
| **詳細デバッグ** | FTD CLI | <code>system support application-log</code> (AMP ログの tail) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 検知されない | Policy が適用されていない | AC ポリシーの Rule に File Policy が紐付いているか確認。 |
| 解析待ちのまま進まない | Threat Grid 通信不可 | `panacea` サービスの疎通と、ライセンスの有効性を確認。 |
| クラウド接続エラー | 証明書または NTP | <code>show ntp status</code> を確認。時刻ズレは TLS 接続に失敗する。 |
| ファイルがパスされる | ファイルサイズ超過 | `Maximum File Size` 設定がスキャン対象より小さいか確認。 |

---

## ⚠ 制限事項

*   **ファイルサイズの制限**: 非常に大きなファイル（例：100MB以上）は、パフォーマンス維持のため解析がスキップされる場合がある。
*   **パスワード保護**: 暗号化されたアーカイブ（.zip/.7z）内はパスワードなしでは AMP でスキャンできない。
*   **オフライン環境**: クラウド連携が基本のため、Air-gapped な環境では **AMP Private Cloud** 構成が必要。

---

## 🔄 他技術との関連

*   **2.0 Firewall**: FTD/ASA の Access Control ルールの一部として AMP が機能する。
*   **3.6 eStreamer**: 検知したマルウェアイベントを SIEM へ転送するために使用する。
*   **6.0 API**: Threat Grid API を使用して、スクリプトから自動的にファイルを提出・解析結果を取得できる。

---

## 🧩 比較表

### AMP for Networks vs AMP for Endpoints

| 特徴 | AMP for Networks (FTD/WSA) | AMP for Endpoints (Connector) |
| :--- | :--- | :--- |
| **検査場所** | 通信経路（インライン） | ホスト内（常駐） |
| **可視性** | ネットワーク伝播経路 | プロセス、レジストリ、ファイル操作 |
| **アクション** | 通信の遮断 | ファイル削除、端末隔離 |
| **強み** | エージェントレスでの保護 | オフライン時の検知、詳細調査 |

---

## 💡 ベストプラクティス

1.  **Block Malware アクション**: 検知した際にアラートを出すだけでなく、確実に `Block Malware` を設定して侵入を阻止する。
2.  **Dynamic Analysis の活用**: シグネチャにない 0-day 攻撃を防ぐため、信頼できない送信元からのファイルにはサンドボックス解析を強制する。
3.  **Talos 連携**: 常に最新の脅威情報を受け取れるよう、 Talos の IP レピュテーションフィルタリングと併用する。

---

## 📝 ラボ学習・設定サンプル例

*(以下の内容はソースに基づき構成していますが、詳細な CLI/GUI 操作は一般的な CCIE 技術知識を含みます。)*

### 1. FTD での実行ファイルブロック
*   **要件**: `.exe` ファイルのダウンロードを検知・ブロックせよ。
*   **設定**: File Policy で `File Type: Executable` に対して `Block Malware` を設定。

### 2. WSA でのサンドボックス解析設定
*   **要件**: 未知の PDF ファイルを自動的に Threat Grid へ送信せよ。
*   **操作**: WSA AMP 設定で `Dynamic Analysis: PDF` を Enable にする。

### 3. FMC でのファイルトラジェクトリ確認
*   **要件**: 特定のマルウェアがネットワーク内のどのホストに存在するか特定せよ。
*   **操作**: `Analysis > Files > File Trajectory` を確認。

### 4. AMP Cloud リージョンの変更
*   **要件**: データプライバシーのため、AMP Cloud の接続先を Europe リージョンに変更せよ。

### 5. バイパス設定の構成
*   **要件**: 信頼された社内アップデートサーバからの通信は AMP 検査を除外せよ。

### 6. アーカイブファイル（ZIP）の再帰的スキャン
*   **要件**: ZIP ファイル内のファイルをスキャンするように FMC を構成せよ。

### 7. 大容量ファイル解析のデバッグ
*   **問題**: 50MB のファイルが AMP でスキャンされない。
*   **対処**: FMC の `Advanced Settings` で解析可能な最大サイズを調整。

### 8. Threat Grid との連携検証
*   **要件**: サンドボックスで生成された解析レポートを FMC から閲覧せよ。

### 9. レトロスペクティブイベントの生成
*   **シナリオ**: 解析後に判定が Malicious に変わった際のアラートを確認せよ。

### 10. FMC API による AMP イベント抽出
*   **要件**: Python スクリプトを使用して過去 24 時間の Malware Event を取得せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: FMC の AC Policy で `File Policy` が紐付いているが、`Action` が `Allow` のルールである場合、AMP は動作するか？
    *   **回答**: はい、AMP Policy 自体で Block 設定になっていれば、ハッシュ判定に基づいてブロック可能です。
2.  **トラブルシュート**: WSA を経由した通信でマルウェアが検知されるが、FMC にイベントが表示されない。なぜか？
    *   **回答**: WSA と FMC はそれぞれ独立した AMP 実装であり、WSA のログは WSA 上（または SMA/CTR）で確認する必要があります。
3.  **Design**: マルウェアのネットワーク内での拡散経路（ Lateral Movement）を視覚化するために必要な AMP の機能は？
    *   **回答**: **File Trajectory**（ファイルトラジェクトリ）。
4.  **実装**: Threat Grid へのファイル提出を最小限に抑えつつ、未知のマルウェア防御を最大化するための設定は？
    *   **回答**: **SHA-256 レピュテーションチェック**を優先し、レピュテーションが `Unknown` の場合にのみ Dynamic Analysis を実行する構成。
5.  **トラブルシュート**: FTD CLI で <code>show amp status</code> を実行した際、`Connection: Down` となっている場合に確認すべきスイッチの ACL は？
    *   **回答**: 管理インターフェイスからクラウドへの **TCP 443 (HTTPS)** 許可設定。

---

## 🔗 参考リソース

*   **Cisco Secure Firewall (FTD) Configuration Guide**: [Malware Defense (AMP)](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/file_and_malware_detection.html)
*   **Cisco WSA User Guide**: [Configuring AMP on WSA](https://www.cisco.com/c/en/us/td/docs/security/wsa/wsa12-0/user_guide/b_WSA_UserGuide_12_0.html)
*   **Cisco Live (BRKSEC-2041)**: [Deep Dive into Advanced Malware Protection (AMP)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「AMP は SHA-256 がパスポート」と覚えてください。パスポートの評判（Reputation）が悪いと入国拒否、不明なら別室（Sandbox）送りになります。
*   **図解**: 
    - **Cloud**: 巨大なデータベース（判定役）
    - **Connector/Device**: ハッシュを計算して送る（現場役）
*   **注意点**: ラボ試験では、**AMP 用のライセンスが有効になっていない**と設定メニュー自体が表示されないことがあるため、最初に `System > Licenses` を確認しましょう。
