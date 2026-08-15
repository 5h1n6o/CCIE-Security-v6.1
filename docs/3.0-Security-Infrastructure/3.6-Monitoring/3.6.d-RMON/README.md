---
layout: default
title: 3.6.d-RMON
nav_order: 4
parent: 3.6-Monitoring
grand_parent: 3.0-Security-Infrastructure
---

# 3.6.d RMON (Remote Monitoring)

**RMON (Remote Monitoring)** は、SNMP の拡張規格（MIB）であり、ネットワーク管理者や NMS (Network Management System) がリモートのネットワークセグメントを効率的に監視・分析するためのプロトコルです。CCIE Security においては、インフラの可用性維持と異常検知の文脈で、特定の MIB 変数が閾値を超えた際に自動的にアラートを生成する「Alarms and Events」グループの実装が重要となります。

---

## 📘 概要

*   **機能概要**: SNMP トラップがデバイスの状態変化を即時通知するのに対し、RMON は統計情報の蓄積や閾値の監視を行い、条件を満たしたときにイベントを発生させます。
*   **利用目的**: CPU 使用率の監視、インターフェイスのドロップ率、トラフィック量の閾値監視など、管理者が常にポーリング（監視）しなくても異常を検知できるようにします。
*   **どのような場面で利用するか**: ネットワークのパフォーマンス低下やセキュリティ攻撃（DoS 等）によるリソース枯渇をいち早く検知し、Syslog や SNMP トラップで通知したい場合に利用します。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | 管理端末からのポーリングなしで、デバイス自ら統計と閾値を管理する。 |
| **用途** | パフォーマンス監視、異常（しきい値超過）の自動通知。 |
| **メリット** | 管理ネットワークの帯域節約（NMS からの頻繁なポーリングが不要）。 |
| **デメリット** | 監視対象が多い場合、デバイスのメモリや CPU リソースを消費する。 |
| **対応機種** | ほぼ全ての Cisco Catalyst スイッチ、IOS-XE ルータ。 |
| **制限事項** | 監視できる変数（OID）は整数値に限られる。 |
| **設計上の注意点** | ポーリング間隔（Interval）が短すぎるとデバイス負荷が上がるため注意。 |

---

## 🏗 動作原理

RMON は「Alarm（アラーム）」と「Event（イベント）」の 2 つのコンポーネントを紐付けて動作します。

```text
[ Network Variable (OID) ]
       ↓ (Monitoring Interval)
[ RMON Alarm Group ] --- (Threshold Check) ---> [ Event Triggered ]
                                                       ↓
                                             [ RMON Event Group ]
                                                       ↓
                                             (Action: Log / Trap)
```

---

## ⚙ 動作シーケンス

1.  **変数監視**: デバイスが指定された OID（例：CPU 使用率、IF 統計）を一定間隔（Interval）でサンプリング。
2.  **閾値判定**: 
    *   **Rising Threshold**: 値が上昇し、上限値を超えた。
    *   **Falling Threshold**: 値が下降し、下限値を下回った。
3.  **イベント発火**: 閾値を超えた際、定義された `event` を呼び出す。
4.  **通知実行**: イベントに設定されたアクション（Syslog への記録、または NMS への Trap 送信）を実行。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Alarm と Event の紐付け**: ラボでは「CPU が 80% を超えたらイベント 1 を、50% を下回ったらイベント 2 を実行せよ」といった要件が出ます。アラーム設定の末尾にイベント ID を正しく指定できるかが鍵です。
*   **Absolute vs Delta**: 
    *   `absolute`: 測定値そのものを比較（例：CPU 使用率）。
    *   `delta`: 前回の測定値との差分を比較（例：エラーパケットの増加量）。
    *   要件に応じてどちらを選ぶべきか判断が必要です。
*   **OID の特定**: 多くの問題では OID 名（例：`ifInErrors.1`）が与えられますが、コマンド補完が効かない場合があるため、基本的な構造を理解しておく必要があります。
*   **ハードニングとの関係**: RMON はシステム管理の一部として「System Hardening and Availability」セクションに関連付けられています。

---

## 🛠 設定方法

### 1. イベントの定義
アラートが発生した際に何をするか（ログ記録またはトラップ送信）を定義します。
```bash
! ログを記録し、"Security_Alert"という名前でトラップを送信するイベント1
rmon event 1 log trap public description "High_Resource_Usage" owner ccie_admin
```

### 2. アラームの定義（閾値監視）
監視対象、間隔、閾値、および上記イベントとの紐付けを定義します。
```bash
! インターフェイス GigabitEthernet1 の入力エラー(OID: .1.3.6.1.2.1.2.2.1.14.1)を監視
! 60秒おきにサンプリング、差分(delta)で比較
! 上限(rising) 10 で イベント1、下限(falling) 5 で イベント2(未定義でも可)
rmon alarm 10 .1.3.6.1.2.1.2.2.1.14.1 60 delta rising-threshold 10 1 falling-threshold 5
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **設定済みアラームの確認** | <code>show rmon alarms</code> |
| **イベントの定義と実行履歴確認** | <code>show rmon events</code> |
| **統計情報の確認** | <code>show rmon statistics</code> |
| **履歴の確認** | <code>show rmon history</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| イベントが発生しない | 閾値の設定ミス | <code>show rmon alarms</code> | <code>rising</code> と <code>falling</code> の値を逆にしていないか確認。 |
| トラップが NMS に届かない | <code>snmp-server host</code> 不足 | <code>show run \| inc snmp</code> | トラップ送信先の IP 設定を確認。 |
| OID が無効と出る | OID の書式または存在しない | <code>show snmp mib</code> | 正しい OID (先頭のドットなど) を確認。 |
| 通信量が少ないのにアラームが出る | <code>absolute</code> 設定ミス | <code>show rmon alarms</code> | 累計値ではなく「増加分」を見たい場合は <code>delta</code> を使用。 |

---

## ⚠ 制限事項

*   **データ型**: 監視できる MIB 変数は ASN.1 整数型のみです。文字列などの監視には対応していません。
*   **リソース消費**: サンプリング間隔を極端に短く（例：1秒）すると、スイッチの CPU 負荷が上昇し、本来のパケット処理に影響を与える可能性があります。

---

## 🔄 他技術との関連

*   **3.6.b SNMP**: RMON は SNMP の一部であり、設定には `snmp-server` の基本設定が必要です。
*   **3.6.c SYSLOG**: RMON イベントのアクションとして `log` を指定すると、内部ログにメッセージが記録されます。
*   **3.1.a CoPP**: RMON が生成する SNMP トラップトラフィックも、コントロールプレーン保護の対象となります。

---

## 🧩 比較表

### RMON vs SNMP Polling

| 特徴 | RMON | SNMP Polling (標準) |
| :--- | :--- | :--- |
| **監視の主体** | **デバイス自身** | NMS (管理サーバ) |
| **ネットワーク負荷** | 低（異常時のみ通信） | 高（定期的に常に通信） |
| **即時性** | 極めて高い | ポーリング間隔に依存 |
| **設定負荷** | デバイスごとの個別設定が必要 | NMS 側で一括管理 |

---

## 💡 ベストプラクティス

1.  **Delta 監視の推奨**: エラーカウンタなどは単調増加するため、`absolute` ではなく `delta` を使用して「急増」を検知するようにします。
2.  **イベント名の明確化**: `description` を活用し、何の障害でアラートが出たのかログから判別しやすくします。
3.  **SNMP v3 の併用**: イベントで `trap` を送る際は、セキュリティ確保のため SNMP v3 経由での送信を構成します。

---

## 📝 ラボ学習・設定サンプル例

### 1. CPU使用率監視
*   **要件**: 5分間の CPU 平均が 90% を超えたら Syslog に記録せよ。
*   **設定**: `rmon alarm 1 .1.3.6.1.4.1.9.2.1.58.0 300 absolute rising 90 1`

### 2. インターフェイス過負荷検知
*   **要件**: Gi0/1 の入力トラフィック増加分が 10MB を超えたら通知せよ。

### 3. 送信エラー監視
*   **要件**: Output Errors が 1 分間に 5 回以上発生したらログを出力せよ。

### 4. 空きメモリ監視
*   **要件**: フリーメモリが 100MB を下回ったらアラームを発生させよ。

### 5. SNMP トラップとの連携
*   **要件**: アラーム発生時に NMS (10.1.1.1) へトラップを送れ。

### 6. 二重イベントの設定
*   **要件**: 上限超過時（Event 1）と復旧時（Event 2）で異なるアクションを定義せよ。

### 7. 履歴グループ（History）の設定
*   **要件**: 特定ポートの統計情報を 30 分ごとに 10 サンプル保存せよ。
*   **設定**: `interface Gi0/1`, `rmon collection history 1 buckets 10 interval 1800`

### 8. 統計グループ（Statistics）の設定
*   **要件**: レイヤ 2 統計（パケットサイズ分布等）を収集せよ。

### 9. デッドロック防止（Hysteresis）
*   **解説**: 上限と下限の差（遊び）を設けて、境界付近でのアラーム乱発を防ぐ。

### 10. 全アラームのクリア
*   **操作**: `clear rmon alarms`

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `rmon alarm 10 OID 60 delta rising 100 1`。この設定で、OID の値が 500 から 550 に変化した場合、イベント 1 は実行されるか？
    *   **回答**: 実行されない。`delta`（差分）が 50 であり、閾値の 100 を超えていないため。
2.  **Design**: 管理トラフィックを最小限に抑えつつ、深夜帯のリソース異常を検知したい。最適な手法は？
    *   **回答**: **RMON Alarms and Events** を使用し、デバイス側で閾値判定を行わせる。
3.  **トラブルシュート**: `show rmon alarms` で監視は動いているがログが出ない。何を確認すべきか？
    *   **回答**: `rmon event` 設定に `log` キーワードが含まれているか、および `logging on` が有効かを確認。
4.  **実装**: インターフェイス `GigabitEthernet 1/0/1` の入力ドロップを監視したい。OID の末尾のインデックス（インスタンス ID）はどう決まるか？
    *   **回答**: `ifIndex` に基づく。通常、ポート番号と一致するか `show snmp mib ifmib ifindex` で確認可能。
5.  **コンフィグ読解**: `falling-threshold` にイベント ID が指定されていない場合、どのような挙動になるか？
    *   **回答**: 値が下限を下回っても、どのアクション（イベント）も実行されない。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [Configuring RMON (Cisco.com)](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/snmp/configuration/xe-16/snmp-xe-16-book/nm-snmp-cfg-rmon-supp.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [Securing the Management Plane](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「RMON はデバイス内の自動監視員」です。管理者が寝ている間も、設定された閾値（金利のようなもの）をチェックし、ルール違反があれば笛を吹く（Event）とイメージしてください。
*   **図解**: `show rmon alarms` の出力にある `Sample type` が `absolute` か `delta` かを確認することが、トラブルシューティングの第一歩です。
*   **注意点**: ラボ試験では OID をドット形式（`.1.3.6...`）で入力させる場合があるため、最初の「ドット」を忘れないようにしてください。
