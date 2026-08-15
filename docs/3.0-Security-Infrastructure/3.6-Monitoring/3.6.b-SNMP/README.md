---
layout: default
title: 3.6.b-SNMP
nav_order: 2
parent: 3.6-Monitoring
grand_parent: 3.0-Security-Infrastructure
---

# 3.0 Security Infrastructure
# 3.6.b SNMP

**SNMP (Simple Network Management Protocol)** は、ネットワーク上のデバイスを監視・管理するための標準プロトコルです,。CCIE Security v6.1 においては、インフラの要塞化（System Hardening）および監視（Monitoring）の文脈で、特にセキュアな **SNMPv3** の実装と、コントロールプレーン保護（CoPP）との組み合わせが重要視されます,。

---

## 📘 概要

*   **機能概要**: ネットワーク管理システム（NMS）がデバイス（エージェント）の管理情報ベース（MIB）にアクセスし、情報の取得（GET）や設定（SET）を行い、またデバイス側から異常を通知（TRAP/INFORM）する仕組みです,。
*   **利用目的**: デバイスのヘルスチェック（CPU/メモリ）、インターフェイス状態の監視、セキュリティイベントのリアルタイム通知,。
*   **どのような場面で利用するか**: 
    *   **Cisco DNA Center** や Prime Infrastructure による集中管理。
    *   SIEM や監視サーバへのアラート送信。
    *   資産管理やキャパシティプランニングのための統計収集。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | マネージャ（NMS）とエージェント（デバイス）間での UDP 通信。 |
| **用途** | 監視、パフォーマンス測定、異常検知,。 |
| **メリット** | 業界標準であり、マルチベンダー環境での可視化が可能。 |
| **デメリット** | v1/v2c は平文パスワード（Community String）であり脆弱。 |
| **対応機種** | ほぼ全ての Cisco IOS/IOS-XE, ASA, FTD デバイス,。 |
| **制限事項** | 通信が UDP ベースのため、パケットロスの可能性がある（INFORM で緩和可能）。 |
| **設計上の注意点** | セキュリティ要件が高い環境では必ず **SNMPv3 (authPriv)** を使用する。 |

---

## 🏗 動作原理

SNMP は、NMS からのポーリングと、エージェントからのプッシュ通知の 2 つのモデルで動作します。

```text
[ NMS (Manager) ] <---------- (UDP 161: GET/SET) ----------> [ Device (Agent) ]
       |                                                         |
       | <--- (UDP 162: Trap/Inform) ----------------------------|
       |                                                         |
       └--- (Telemetry/SNMP Traps for DNA Center)
```

---

## ⚙ 動作シーケンス

1.  **情報要求**: NMS が特定の OID（Object Identifier）を指定して GET リクエストを送信。
2.  **応答**: エージェントが MIB からデータを取得し、NMS へ応答。
3.  **イベント発生**: デバイスの状態変化（IF Down など）を検知。
4.  **通知送信**: デバイスが `snmp-server host` で設定された宛先へ Trap または Inform を送信。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **SNMPv3 の 3 つのセキュリティレベル**: 
    *   `noAuthNoPriv`: 認証なし、暗号化なし（ユーザ名のみ一致）。
    *   `authNoPriv`: 認証（MD5/SHA）あり、暗号化なし。
    *   `authPriv`: **認証（SHA 等）あり、暗号化（AES 等）あり**。ラボではこの設定が標準です。
*   **EngineID の概念**: SNMPv3 では EngineID が鍵生成のベースとなります。VNF（CSR1000v 等）のクローン時に ID が重複して通信できないトラブルへの理解が求められます。
*   **View によるアクセス制限**: `snmp-server view` を使用して、NMS が閲覧できる MIB の範囲を特定のツリー（例：iso のみ）に限定する要件が頻出です。
*   **トラップ送信設定**: `snmp-server host [IP] [Community]` コマンドの正確な入力が必要です。
*   **CoPP との連携**: SNMP トラフィック（UDP 161）がコントロールプレーンを圧迫しないよう、CoPP でレート制限をかけるシナリオに注意してください。

---

## 🛠 設定方法

### 1. SNMPv2c の基本設定（Read-Only）
```bash
snmp-server community CISCO-READ RO
snmp-server host 172.16.6.55 CISCO-READ
```

### 2. SNMPv3 のセキュア設定 (authPriv)
```bash
! 1. ビューの定義 (全てのMIBを許可)
snmp-server view ALL-ACCESS iso included

! 2. グループの作成 (認証と暗号化を必須にする)
snmp-server group SECURE-GROUP v3 priv read ALL-ACCESS

! 3. ユーザの作成 (SHA認証、AES暗号化)
snmp-server user ADMIN SECURE-GROUP v3 auth sha Cisco123 priv aes 128 Cisco456

! 4. ホスト（通知先）の設定
snmp-server host 172.16.6.55 version 3 priv ADMIN
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **SNMP設定の全体確認** | <code>show snmp</code> |
| **SNMPv3 ユーザの確認** | <code>show snmp user</code> |
| **SNMPv3 グループの確認** | <code>show snmp group</code> |
| **通知先ホストの確認** | <code>show snmp host</code> |
| **統計情報（パケット数など）** | <code>show snmp stats</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| NMS から通信できない | Community String またはユーザ名不一致 | <code>show snmp</code> | 文字列が正しいか再確認。 |
| SNMPv3 認証エラー | パスフレーズまたはハッシュ方式 (SHA/MD5) 不一致 | <code>debug snmp packet</code> | 設定を両端で合わせる。 |
| トラップが届かない | 宛先 IP 設定ミス、または ACL で遮断 | <code>show run \| inc snmp-server host</code> | IP 疎通と ACL 許可を確認。 |
| タイムアウトが発生 | EngineID の不整合（クローン時など） | <code>show snmp engineID</code> | <code>snmp-server engineID local</code> で再生成。 |

---

## ⚠ 制限事項

*   **CPU負荷**: 大量のポーリング（GET）やトラップ生成は、デバイスの CPU 負荷を増大させます。
*   **ポート番号**: デフォルトの UDP 161/162 以外を使用する場合、NMS 側の設定変更も必要です。
*   **暗号化ライセンス**: 強力な暗号化（AES-256 など）を使用するには、K9 イメージ（ペイロード暗号化）が必要です。

---

## 🔄 他技術との関連

*   **3.1.a CoPP**: 管理通信プレーンの保護対象として SNMP パケットを制御,。
*   **3.1.c iACLs**: インフラ ACL により、許可された NMS 以外からの SNMP アクセスをエッジで遮断。
*   **3.10 Cisco DNAC API**: DNA Center が SNMP トラップを受信し、インベントリやテレメトリを更新。

---

## 🧩 比較表

### SNMP バージョン比較

| 機能 | SNMP v1 | SNMP v2c | SNMP v3 |
| :--- | :--- | :--- | :--- |
| **セキュリティ** | Community (平文) | Community (平文) | **ユーザベース (USM)** |
| **認証/暗号化** | なし | なし | **HMAC / AES / DES** |
| **主な特徴** | 基本機能 | GetBulk 追加 | セキュリティ強化 |
| **推奨** | 非推奨 | 読み取り専用限定 | **推奨 (Enterprise)** |

---

## 💡 ベストプラクティス

1.  **SNMPv3 authPriv の採用**: 常に最高レベルのセキュリティを維持します。
2.  **ACL による制限**: `snmp-server community [String] [ACL]` またはグループ設定で、NMS のソース IP を制限します。
3.  **不要な Trap の無効化**: 必要なイベントのみに絞り、不要なトラフィックと負荷を削減します。
4.  **RO/RW の使い分け**: 通常は Read-Only (RO) を使い、設定変更 (RW) は必要な場合のみ一時的に有効にします。

---

## 📝 ラボ学習・設定サンプル例

### 1. NMS ホストの設定
*   **問題**: 管理サーバ 10.1.1.100 へ Community 名 "MONITOR" でトラップを送れ。
*   **設定**: `snmp-server host 10.1.1.100 MONITOR`。

### 2. SNMPv3 Group の定義
*   **要件**: 全ての MIB を閲覧可能な "ADMIN-GROUP" を作成せよ。
*   **設定**: `snmp-server view ALL iso included`, `snmp-server group ADMIN-GROUP v3 priv read ALL`。

### 3. SHA/AES ユーザの作成
*   **要件**: ユーザ "OPERATOR" を作成し、SHA 認証と AES-128 暗号を適用せよ。

### 4. MIB View による制限
*   **要件**: インターフェイス情報 (ifMIB) のみ閲覧可能なビューを作成せよ。

### 5. SNMP ACL による NMS 制限
*   **要件**: 10.1.1.0/24 以外のネットワークからの SNMP 通信を拒否せよ。

### 6. Inform 通知の設定
*   **要件**: 到達確認が必要な Inform 通知を構成せよ。
*   **設定**: `snmp-server host [IP] informs version 3 priv [User]`。

### 7. エンジン ID の設定
*   **課題**: デバイス固有の EngineID を手動で設定せよ。

### 8. ASA での SNMP 構成
*   **要件**: ASA で inside インターフェイスからの SNMP 通信を許可せよ。
*   **設定**: `snmp-server host inside 10.1.1.10`, `snmp-server enable`。

### 9. 特定トラップ（Config変更）の有効化
*   **要件**: コンフィグが変更された際にトラップを送出せよ。
*   **設定**: `snmp-server enable traps config`。

### 10. CoPP による SNMP 制限
*   **課題**: SNMP トラフィックを 100kbps に制限する CoPP ポリシーを作成せよ。

---

## ❓ 想定試験問題

1.  **Design**: SNMPv3 を実装する際、整合性と機密性を両立するために必要な設定要素は？
    *   **回答**: **authPriv** セキュリティレベルを選択し、認証ハッシュ (SHA 等) と暗号化アルゴリズム (AES 等) を定義する。
2.  **トラブルシュート**: `show snmp user` ではユーザが表示されるが、NMS からポーリングできない。何を確認すべきか？
    *   **回答**: ユーザが所属する **Group** のセキュリティレベル（Priv 指定の有無）と、適用されている **ACL/View** の範囲を確認する。
3.  **コンフィグ読解**: `snmp-server host 192.168.1.10 v3 priv USER1` という設定がある。通知が届かない場合、NMS 側で一致させるべき項目は？
    *   **回答**: USER1 の認証・暗号化パスワード、およびデバイスの **EngineID**。
4.  **実装**: インターフェイスの Up/Down ログのみを特定の NMS に送りたい。どのような設定が必要か？
    *   **回答**: `snmp-server enable traps snmp linkup linkdown` を実行し、host 設定で対象 NMS を指定する。
5.  **Design**: Trap と Inform の主な違いは何か？
    *   **回答**: Trap は到達確認を行わない（Unreliable）が、**Inform** は受信確認（Acknowledgement）を要求するため信頼性が高い。

---

## 🔗 参考リソース

*   **Cisco Configuration Guide**
    *   [Configuring SNMP (Cisco.com)](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/snmp/configuration/xe-16/snmp-xe-16-book.html)
*   **Technical Notes**
    *   [SNMPv3 Configuration on Cisco IOS](https://www.cisco.com/c/en/us/support/docs/ip/simple-network-management-protocol-snmp/13506-snmp-traps.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [Securing the Management Plane](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「SNMP はネットワークの問診票」です。NMS がデバイスの健康状態を聞き、デバイスが急病の時に自ら訴える (Trap) 仕組みだと覚えましょう。
*   **図解**: `snmp-server host` はラボにおける疎通の起点です。ここから NMS への道が ACL やルーティングで塞がれていないかを常にチェックしてください。
*   **注意点**: CCIE ラボ試験では、コマンドのスペルミス（`snmp-server` のハイフン有無など）や、ユーザとグループの紐付けミスで時間を浪費しやすいため、CLI 補完を活用して正確に設定してください。
