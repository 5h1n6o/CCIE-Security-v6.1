---
layout: default
title: 3.6.e-eStreamer
nav_order: 3
parent: 3.6-Monitoring
grand_parent: 3.0-Security-Infrastructure
---

# 3.6.e eStreamer

**eStreamer (Event Streamer)** は、Cisco Secure Firewall (旧 Firepower) システムにおいて、FMC (Firepower Management Center) が収集した大量のセキュリティイベントデータを、外部のコレクタ（SIEM、ログ解析サーバ、カスタムアプリケーションなど）へ効率的に転送するためのプロトコルおよびサービスです,,。

---

## 📘 概要

*   **機能概要**: FMC 上で動作するサーバープロセスが、バイナリ形式の構造化データとして脅威イベント、侵入イベント、マルウェアイベント、接続統計などを外部クライアントにストリーミング配信します,。
*   **利用目的**: 大規模環境におけるイベントの集約管理、SIEM (Splunk, QRadar等) での高度な相関分析、および長期的なコンプライアンス用ログの外部保存を目的とします,。
*   **どのような場面で利用するか**: 
    *   FMC の GUI だけでは不十分な、企業全体の統合ログ監視が必要な場合。
    *   FMC 側のデータベース負荷を軽減しつつ、リアルタイムに侵入検知イベントを外部通知したい場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **通信プロトコル** | TCP 8302 ポートを使用した双方向通信。 |
| **セキュリティ** | TLS による暗号化と、証明書ベース（PKCS#12）の認証が必須,。 |
| **データ形式** | バイナリデータ形式（構造化されたレコードタイプ）。 |
| **転送元 (Server)** | **FMC** (Firepower Management Center)。※Managed Device ではない。 |
| **転送先 (Client)** | SIEM (Splunk/Sentinel 等) や eStreamer SDK 搭載クライアント。 |
| **メリット** | Syslog よりも詳細なメタデータ（URL 属性、SGT、ファイルハッシュ等）を転送可能。 |
| **設計上の注意点** | FMC とクライアント間で TCP 8302 を許可する必要がある。 |

---

## 🏗 動作原理

eStreamer はクライアント・サーバーモデルで動作します。FMC がサーバーとなり、外部デバイスがクライアントとして接続を要求します。

```text
[ eStreamer Client ]             [ FMC (Server) ]
        |                               |
        |---- TCP 8302 (TLS Conn) ----->| 1. 証明書による認証
        |                               |
        |<--- Request for Data ---------| 2. クライアントが希望するデータ型を指定
        |                               |
        |==== Streaming Binary Data ===>| 3. FMC がイベント発生の都度プッシュ
        |                               |
```

---

## ⚙ 動作シーケンス

1.  **クライアント登録**: FMC の GUI でクライアントの IP アドレスを登録し、認証用のパスワードを設定します。
2.  **証明書の発行**: FMC がクライアント専用の証明書（.pkcs12）を生成し、管理者がこれをダウンロードしてクライアント側にインストールします。
3.  **接続確立**: クライアントが TCP 8302 ポートで FMC に接続を開始し、TLS ハンドシェイクで相互認証を行います。
4.  **リクエスト送信**: クライアントは受信したいイベントの種類（例：侵入イベントのみ、または接続ログ全てなど）をリクエストメッセージとして送信します。
5.  **データ配信**: FMC は内部データベース（Event Database）に書き込まれた新規データを即座にバイナリパケットとしてクライアントへ転送します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **ポート番号の把握**: ラボ要件で「外部 SIEM へイベントを転送せよ」とあれば、中間のファイアウォールで **TCP 8302** を許可する ACL 設定が必要になる場合があります。
*   **証明書管理**: FMC でクライアントを登録し、証明書ファイルをエクスポートする一連の手順は頻出です。パスワードの入力ミスに注意してください。
*   **転送元デバイスの特定**: eStreamer は FMC から提供される機能であり、**Managed Device (FTD/ASA) から直接 SIEM へ eStreamer で送ることはできません**。
*   **Syslog との使い分け**: テキスト形式の簡易ログなら Syslog (UDP 514/TCP 1468)、詳細なフォレンジックデータなら eStreamer という切り分けが Design 問題で問われます,。
*   **アクセスコントロール**: FMC 側で特定の IP アドレスからの接続のみを許可する設定が、セキュリティ要塞化（Hardening）の観点で重要です。

---

## 🛠 設定方法

### 1. FMC GUI でのクライアント追加
1.  **System > Configuration > eStreamer** に移動します。
2.  **Create Client** をクリックします。
3.  **Hostname** にクライアントの IP アドレス（SIEM 等）を入力します。
4.  **Password** を入力（証明書ファイルの保護用）。
5.  **Save** 後、作成されたクライアントの横にある **Download** アイコンをクリックして証明書を保存します。

### 2. 転送イベントの選択
*   同画面の **Event Discovery** セクションで、転送したい項目（Intrusion, Connection, File, Malware等）にチェックが入っていることを確認します。

### 3. CLI による状態確認（FMC 内部）
```bash
# eStreamer プロセスの稼働状態確認 (Expert mode)
sudo pmtool status | grep stream
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **ポートのリスニング確認** | <code>netstat -an \| grep 8302</code> |
| **プロセス稼働確認** | <code>show processes \| include estreamer</code> (FMC Shell) |
| **接続ログの確認** | <code>tail -f /var/log/messages</code> |
| **パケットキャプチャ** | <code>tcpdump -ni any port 8302</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| クライアントが接続できない | 証明書の期限切れ、またはパスワード不一致 | FMC でクライアントを一度削除し、証明書を再発行する。 |
| 通信がタイムアウトする | ネットワーク上の ACL/Firewall による遮断 | クライアントから FMC への 8302 通信が許可されているか確認。 |
| イベントが転送されない | FMC のデータベース設定ミス | 収集対象のイベントが FMC 側で記録されているか確認する。 |
| 証明書インポートエラー | PKCS#12 形式の非互換 | クライアント側のライブラリ（OpenSSL 等）のバージョンを確認。 |

---

## ⚠ 制限事項

*   **同時接続数**: 1 台の FMC がサポートできる eStreamer クライアント数には上限があります。過度な接続は管理プレーンの負荷となります。
*   **後方互換性**: FMC のメジャーアップデート時、eStreamer のバイナリ構造（SDK バージョン）が変更されることがあり、クライアント側の更新が必要になる場合があります。
*   **データ量**: Connection Event 全てを送る設定にすると、大量のトラフィックが発生し、FMC のパフォーマンスに影響を与える可能性があります。

---

## 🔄 他技術との関連

*   **3.6.c SYSLOG**: eStreamer は詳細な構造化データを送りますが、Syslog はシンプルなテキスト通知用として併用されます。
*   **3.1.a CoPP**: FMC への 8302 通信は、FMC 自身のコントロールプレーンリソースを消費します。
*   **Access Control Policy**: ログを eStreamer で送るためには、該当する ACP ルールで "Log at End of Connection" などのロギングが有効である必要があります。

---

## 🧩 比較表

### eStreamer vs Syslog (in Secure Firewall)

| 特徴 | eStreamer | Syslog |
| :--- | :--- | :--- |
| **データ構造** | バイナリ (複雑・詳細) | テキスト (単純) |
| **転送単位** | ストリーミング (プッシュ) | メッセージ単位 |
| **情報の豊富さ** | 非常に高い (SGT, Application ID等) | 低～中 |
| **実装負荷** | 高い (証明書・SDKが必要) | 低い (標準プロトコル) |
| **主な用途** | SIEM での詳細解析 | リアルタイム監視・アラート |

---

## 💡 ベストプラクティス

1.  **専用ネットワークの利用**: ログ転送による帯域圧迫を避けるため、可能な限り管理用（Management）ネットワークを介して転送します。
2.  **イベントフィルタリング**: クライアント側で全てのイベントをリクエストするのではなく、必要なイベントタイプのみを選択してリクエストします。
3.  **証明書の定期更新**: セキュリティ維持のため、証明書の有効期限を把握し、期限が切れる前に更新（再発行）手順を実施します。
4.  **FMC 冗長化時の考慮**: HA 構成の FMC では、プライマリ FMC とセカンダリ FMC の両方でクライアント登録が必要になる場合があります。

---

## 📝 ラボ学習・設定サンプル例

### 1. クライアント登録
*   **要件**: SIEM サーバ (10.1.1.50) を eStreamer クライアントとして登録せよ。パスワードは `Cisco123` とせよ。

### 2. 証明書のダウンロードと検証
*   **要件**: 生成された証明書ファイルをダウンロードし、中身が PKCS#12 形式であることを確認せよ。

### 3. ポート 8302 の開放
*   **要件**: 外継ぎルータで FMC (172.16.1.10) 宛の TCP 8302 トラフィックを許可する ACL を設定せよ。

### 4. 侵入イベントの転送設定
*   **要件**: Intrusion イベントのみが eStreamer で送出されるよう FMC を構成せよ。

### 5. 接続イベントのロギング有効化
*   **要件**: ACP の `Allow-Web` ルールで発生した通信を eStreamer で飛ばすための事前設定を行え。

### 6. eStreamer プロセスの再起動
*   **課題**: FMC CLI から eStreamer サービスを強制停止し、再度開始せよ。

### 7. デバッグログの追跡
*   **操作**: `/var/log/messages` を監視し、クライアントからの接続試行ログを特定せよ。

### 8. 複数クライアントの登録
*   **要件**: 異なる 2 つの SIEM サーバに同時にイベントを配信するよう構成せよ。

### 9. 証明書パスワードの変更
*   **課題**: 既存クライアントのパスワードを変更し、古い証明書が無効になることを確認せよ。

### 10. パケットキャプチャによる解析
*   **要件**: `tcpdump` を使用して、接続確立時の TLS ハンドシェイクを確認せよ。

---

## ❓ 想定試験問題

1.  **Design**: eStreamer クライアントを FMC に接続するために必要な最小限のネットワーク要件は？
    *   **回答**: クライアント IP から FMC IP への **TCP ポート 8302** の到達性。
2.  **トラブルシュート**: クライアント接続時に "Authentication Failed" がログに記録されている。確認すべき点は？
    *   **回答**: FMC に登録されたクライアント IP が正しいか、およびクライアントが使用している証明書がその FMC で発行された最新のものか。
3.  **コンフィグ読解**: FMC の eStreamer 設定画面で "Connection Events" のチェックが外れている場合、SIEM で接続ログは受信できるか？
    *   **回答**: 受信できない。FMC 側で配信対象として選択されている必要がある。
4.  **実装**: FTD デバイスから直接 eStreamer で SIEM へ送るように指示された。この要件は実現可能か？
    *   **回答**: **不可能**。eStreamer サーバー機能は FMC が提供するため、FTD は FMC にイベントを送り、FMC がそれをクライアントに中継する。
5.  **Design**: ネットワーク帯域が非常に限定的な環境で、eStreamer の負荷を抑えるための手法は？
    *   **回答**: クライアントのリクエストメッセージで、Connection イベントを排除し、Intrusion または Malware イベントのみに絞り込む。

---

## 🔗 参考リソース

*   **Cisco Configuration Guide**:
    *   [Event Streamer (eStreamer) - Cisco Secure Firewall Management Center Administration Guide](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/event_streamer_estreamer.html)
*   **Technical Notes**:
    *   [Troubleshooting eStreamer Connectivity on Firepower Systems](https://www.cisco.com/c/en/us/support/docs/security/firepower-ngfw/200505-Troubleshoot-eStreamer-Connectivity-on-Fi.html)
*   **Cisco Live (BRKSEC-2022)**:
    *   [Firepower Management Center: Integration with SIEM and eStreamer](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「eStreamer は FMC のイベント専門の特急便」です。Syslog という普通郵便に比べて、より重く詳細な荷物（バイナリデータ）を、証明書という身分証（TLS）を確認した相手にだけ届ける仕組みと覚えましょう。
*   **注意点**: ラボ試験では、GUI 設定だけでなく、接続がうまくいかない時の原因として「FMC 側のクライアント IP 登録ミス」がよく問われます。DHCP 環境ではなく固定 IP での運用が基本です。
*   **図解**: 接続が確立されると、FMC のメッセージログには `estreamer: Client connected` という明快な記録が残ります。トラブル時はまずここを見ましょう。
