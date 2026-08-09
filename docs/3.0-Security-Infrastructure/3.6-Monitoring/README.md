---
layout: default
title: 3.6-Monitoring
nav_order: 6
parent: 3.0-Security-Infrastructure
---

# 3.6 Monitoring protocols

ネットワーク監視プロトコルは、インフラストラクチャ内の可視性を確保し、異常検知や事後解析（フォレンジック）を行うための重要な基盤です。CCIE Security v6.1 では、単なる監視設定だけでなく、**Stealthwatch (Cisco Secure Network Analytics)** との連携や、**FMC (Firepower Management Center)** からのイベント抽出など、セキュリティ運用に直結する実装が問われます。

---

## 📘 概要

*   **機能概要**: ネットワークデバイスの稼働状態、トラフィックフロー、セキュリティイベントをリアルタイムまたは蓄積型で収集・転送する技術群です。
*   **利用目的**: 不正アクセスの検知、トラフィックパターンの分析、コンプライアンス維持、およびトラブルシューティングの迅速化を目的とします。
*   **どのような場面で利用するか**: 
    *   **NetFlow/IPFIX**: 誰が・いつ・どこに通信しているかの可視化と Stealthwatch での分析。
    *   **SNMP**: デバイスのヘルスチェック（CPU/Memory）や構成管理。
    *   **SYSLOG**: セキュリティポリシーの拒否ログや設定変更の監査。
    *   **eStreamer**: FMC に蓄積された大量の脅威イベントを外部の SIEM などへ高速転送。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **NetFlow/IPFIX (3.6.a)** | トラフィックの統計情報を収集。IPFIX は NetFlow v9 ベースの標準規格。 |
| **NSEL (3.6.a)** | ASA/FTD における NetFlow 実装（NetFlow Security Event Logging）。ステートフルなイベントを送信。 |
| **SNMP (3.6.b)** | デバイス情報の取得（Pull）とトラップ送信（Push）。v3 は認証と暗号化に対応。 |
| **SYSLOG (3.6.c)** | 標準的なロギング。8 段階の重要度（Severity）でイベントを管理。 |
| **RMON (3.6.d)** | リモート監視。閾値に基づいたアラームとイベント生成（Alarm/Event）が中心。 |
| **eStreamer (3.6.e)** | Cisco Secure Firewall (FMC) 独自のイベント転送プロトコル。 |

---

## 🏗 動作原理

監視プロトコルは、情報を収集する「エージェント（デバイス）」と、それを受け取る「コレクタ（管理サーバ）」の役割分担で動作します。

```text
[ Network Device ] --- (Push: NetFlow/Syslog/SNMP Trap) ---> [ Collector / SIEM ]
        ↑                                                     (Stealthwatch / FMC)
        └---------- (Pull: SNMP Get / eStreamer Request) ----- [ Management Station ]
```

*   **NetFlow**: パケットをフロー（送信元/宛先IP、ポート等）でまとめ、キャッシュがいっぱいになるかタイマーが切れるとコレクタへ転送します。
*   **eStreamer**: クライアント（SIEM等）が FMC に対して特定のイベントカテゴリをリクエストし、FMC がバイナリ形式でデータをプッシュします。

---

## ⚙ 動作シーケンス

1.  **イベント発生/トラフィック流入**: デバイス内でパケット処理や状態変化が発生。
2.  **識別と記録**:
    *   NetFlow ならフローキャッシュを作成。
    *   Syslog なら定義された Severity に基づきメッセージを生成。
3.  **転送判断**:
    *   **Push型**: 即時（Syslog）または一定条件（NetFlow）でコレクタへ送信。
    *   **Pull型**: ポーリング（SNMP Get）を受けて応答。
4.  **処理**: コレクタ側で解析・グラフ化・アラート通知を実行。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Stealthwatch との統合**: ルータ、スイッチ、ASA で NetFlow (NSEL) を設定し、コレクタ（SMC/Flow Collector）でフローが表示されるかを確認するシナリオが重要です。
*   **SNMP v3 のセキュリティ**: `auth` (MD5/SHA) と `priv` (AES/DES) の設定、および `engineID` の概念が問われます。
*   **Logging のフィルタリング**: 全てのログを送るのではなく、特定の ACL にマッチしたログのみ、あるいは特定の Severity 以上のみを送る設定が求められます。
*   **Flexible NetFlow (FNF)**: `flow record`、`flow exporter`、`flow monitor` の 3 要素を正しく紐付ける手順をマスターしてください。
*   **FMC eStreamer**: eStreamer クライアントを FMC に登録するための証明書のインポート手順などが問われる可能性があります。

---

## 🛠 設定方法

### 1. Flexible NetFlow (Router)
```bash
! フローレコードの定義
flow record REC1
 match ipv4 source address
 match ipv4 destination address
 collect counter bytes long
!
! 転送先（コレクタ）の定義
flow exporter EXP1
 destination 192.168.10.100
 transport udp 2055
!
! モニターの作成と適用
flow monitor MON1
 record REC1
 exporter EXP1
!
interface GigabitEthernet0/1
 ip flow monitor MON1 input
```

### 2. ASA NSEL (NetFlow)
```bash
flow-export destination inside 192.168.1.50 2055
!
policy-map global_policy
 class inspection_default
  flow-export event-type all destination 192.168.1.50
```

### 3. SNMP v3 (Secure)
```bash
snmp-server group G1 v3 priv
snmp-server user U1 G1 v3 auth sha Cisco123 priv aes 128 Cisco456
snmp-server host 192.168.1.100 version 3 priv U1
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **NetFlow キャッシュの確認** | <code>show flow monitor MON1 cache</code> |
| **NetFlow 送信統計の確認** | <code>show flow exporter statistics</code> |
| **Syslog バッファの確認** | <code>show logging</code> |
| **SNMP グループ/ユーザー確認** | <code>show snmp group / user</code> |
| **RMON アラート確認** | <code>show rmon alarms</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| NetFlow がコレクタに届かない | インターフェイスへの適用漏れ | <code>show flow monitor</code> で適用箇所を確認。 |
| SNMP v3 認証エラー | パスフレーズや EngineID の不一致 | <code>debug snmp packets</code> で不一致を確認。 |
| Syslog が表示されない | ロギングが無効、またはレベルが低い | <code>logging on</code> と <code>logging trap [level]</code> を確認。 |
| FMC イベントが抽出できない | eStreamer 証明書の不整合 | FMC の <code>eStreamer</code> 設定画面で証明書を再生成。 |

---

## ⚠ 制限事項

*   **パフォーマンス**: NetFlow (特に Full Flow) は CPU とメモリを消費するため、サンプリング NetFlow の検討が必要になる場合があります。
*   **NSEL の制限**: ASA の NSEL は、通常の NetFlow v9 と異なり、コネクションの開始・終了・拒否イベントのみを送信し、中間統計は送りません。
*   **eStreamer 接続数**: FMC がサポートする同時 eStreamer クライアント数には上限があります。

---

## 🔄 他技術との関連

*   **3.1.a CoPP**: SNMP や Syslog のトラフィックをコントロールプレーン保護の対象として制御します。
*   **3.4.g VACL**: VACL の `capture` アクションと監視を組み合わせ、特定トラフィックを分析します。
*   **DNAC Telemetry**: DNAC が SNMP/Syslog/NetFlow を使用してネットワークのヘルス情報を収集します。

---

## 🧩 比較表

### NetFlow v9 vs IPFIX

| 特徴 | NetFlow v9 | IPFIX (v10) |
| :--- | :--- | :--- |
| **標準化** | Cisco 独自（事実上の標準） | IETF 標準 (RFC 7011) |
| **拡張性** | テンプレートベース | ベンダー固有フィールド (Enterprise IE) をサポート |
| **主な用途** | Cisco デバイスの可視化 | マルチベンダー環境の標準監視 |

---

## 💡 ベストプラクティス

1.  **SNMP v3 の使用**: v1/v2c は平文パスワードのため、必ず v3 (authPriv) を使用します。
2.  **NTP 同期**: Syslog や NetFlow のタイムスタンプを正確に保つため、監視対象とコレクタ間の NTP 同期は必須です。
3.  **リモートロギング**: デバイス内のバッファ (`logging buffered`) だけでなく、外部 Syslog サーバへの転送を構成します。
4.  **Stealthwatch の活用**: 単なる監視ではなく、フローデータを Stealthwatch に送り、機械学習による脅威検知を行います。

---

## 📝 ラボ学習・設定サンプル例

1.  **Flexible NetFlow 設定**: ルータの全インターフェイスで入力トラフィックを監視し、コレクタ 10.1.1.100 へ送れ。
2.  **ASA NSEL 設定**: ASA で拒否されたフローも含めて NetFlow イベントを生成せよ。
3.  **SNMP v3 管理**: ユーザー `ADMIN` が AES-128 で通信できるよう構成せよ。
4.  **Syslog 転送**: `Warnings` 以上の重要度のログをサーバ 172.16.1.5 へ送れ。
5.  **RMON 閾値設定**: CPU 使用率が 90% を超えたら Syslog を生成するアラームを設定せよ。
6.  **NetFlow フィルタリング**: 送信元 `10.0.0.0/8` のトラフィックのみを NetFlow 収集対象とせよ。
7.  **FMC eStreamer 登録**: FMC に外部クライアントを登録し、証明書をダウンロードせよ。
8.  **Logging ラベル**: Syslog メッセージにデバイスのホスト名または IP を付与せよ。
9.  **NSEL レート制限**: NetFlow 送信による負荷を防ぐため、ASA で NSEL の生成レートを制限せよ。
10. **DNAC 監視統合**: DNAC を通じてスイッチの SNMP トラップ先を自動設定せよ。

---

## ❓ 想定試験問題

1.  **Design**: 大規模なマルチベンダー環境で、フロー情報の標準的なエクスポート形式として採用すべきプロトコルは？
    *   **回答**: **IPFIX**。
2.  **トラブルシュート**: ASA で NSEL を設定したが、コレクタ側で「トラフィック量」が 0 と表示される。なぜか？
    *   **回答**: NSEL はコネクションイベント（開始/終了等）を送信するものであり、従来の NetFlow のように定期的なバイト数カウンタを更新しない仕様であるため。
3.  **コンフィグ読解**: `snmp-server host 1.1.1.1 version 3 priv USER1` という設定がある。この通信で保証されるセキュリティ要素は？
    *   **回答**: 認証（Authentication）と暗号化（Privacy/Confidentiality）。
4.  **実装**: ルータで NetFlow データを送信する際、コレクタとの通信断を防ぐために `flow exporter` で設定すべき推奨項目は？
    *   **回答**: `source [Interface]` の指定による送信元 IP の固定。
5.  **Design**: FMC の脅威イベントを、ネットワーク帯域を極力消費せずに外部 SIEM に送るための最適な手法は？
    *   **回答**: **eStreamer** プロトコルの使用。

---

## 🔗 参考リソース

*   **Cisco Live (BRKSEC-2202)**: [Network Telemetry and Monitoring](https://www.ciscolive.com/)
*   **Cisco Configuration Guide**: [Configuring Flexible NetFlow](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/fnetflow/configuration/xe-16/fnf-xe-16-book.html)
*   **Cisco Configuration Guide**: [SNMP Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/snmp/configuration/xe-16/snmp-xe-16-book.html)
*   **Stealthwatch Integration Guide**: [Configuring NetFlow on Cisco Devices for Stealthwatch](https://www.cisco.com/c/en/us/td/docs/security/stealthwatch/netflow/Cisco_NetFlow_Configuration.html)

---

## 📝 **補足（Notes）**
*   **学習メモ**: 監視プロトコルは「証拠」を残すためのものです。トラブルシュートやラボ検証でも `show logging` や `show flow monitor` を多用し、自分の設定が正しく動いているかを確認する癖をつけてください。
*   **注意点**: ラボ試験では `vlan filter` (VACL) や `access-list` の `log` オプション が監視プロトコルと組み合わされることが多いため、多層的な視点が必要です。
