---
layout: default
title: 3.6.a-NetFlow-IPFIX-NSEL
nav_order: 3
parent: 3.6-Monitoring
grand_parent: 3.0-Security-Infrastructure
---

# 3.6.a NetFlow/IPFIX/NSEL

**NetFlow/IPFIX/NSEL** は、ネットワークを通過するトラフィックを「フロー」という単位で識別し、その統計情報を収集・転送する技術です。CCIE Security v6.1 においては、インフラの可視化、異常検知（Stealthwatch 連携）、および監査トレースの主要ツールとして位置づけられています。

---

## 📘 概要

*   **機能概要**: パケットのヘッダー情報を基に通信の統計（送信元/宛先IP、ポート、プロトコル、パケット数、バイト数など）をまとめ、外部のコレクタへエクスポートします。
*   **利用目的**: トラフィック分析、キャパシティプランニング、および**セキュリティフォレンジック（誰がいつ誰と通信したかの追跡）**。
*   **どのような場面で利用するか**: 
    *   **Cisco Secure Network Analytics (Stealthwatch)** へのデータ供給源として。
    *   ファイアウォール（ASA/FTD）でのセッション開始・終了ログの効率的な取得（NSEL）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **NetFlow v5/v9** | Cisco 独自のプロトコル。v9 はテンプレートベースで拡張性が高い。 |
| **IPFIX** | NetFlow v9 をベースとした IETF 標準規格（事実上の NetFlow v10）。 |
| **NSEL** | ASA/Firewall 用。全パケットではなく、フローの作成・削除・拒否イベントを送信する。 |
| **Flexible NetFlow** | IOS-XE で採用。ユーザが「何をマッチさせ、何を収集するか」を定義可能。 |
| **メリット** | ミラーリング（SPAN）に比べ帯域を消費せず、全通信のメタデータを記録可能。 |
| **デメリット** | 大規模環境ではデバイスの CPU やメモリを消費する可能性がある。 |

---

## 🏗 動作原理

NetFlow は、受信したパケットが既存の「フローキャッシュ」に存在するかを確認します。新しい通信であればエントリを作成し、終了またはタイムアウトした際にコレクタへ送信します。

```text
[ Packet Arrival ]
      ↓
[ Flow Cache Lookup ] --- (Match?) --- [ Update Statistics ]
      ↓ (No Match)
[ Create New Flow Entry ] (7-tuple: Src/Dst IP, Port, Proto, ToS, Input Intf)
      ↓
[ Flow Aging/Timeout ]
      ↓
[ NetFlow Exporter ] --- (UDP/IPFIX) ---> [ NetFlow Collector (Stealthwatch) ]
```

---

## ⚙ 動作シーケンス

1.  **フロー識別**: 送信元/宛先 IP、L4ポート、プロトコルなどの「7-tuple」でパケットを分類。
2.  **統計更新**: 同一フローのパケット数、バイト数をキャッシュ内でカウント。
3.  **エクスポート判定**: 
    *   **Active Timeout**: 長時間のセッション（デフォルト 30分）を分割送信。
    *   **Inactive Timeout**: 通信が途絶（デフォルト 15秒）したエントリを送信。
4.  **転送**: UDP プロトコルを用いてコレクタへ統計データを送信。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Flexible NetFlow (FNF) の 3 ステップ**: 
    1. `flow record`: 何をマッチさせ、何を収集（collect）するか。
    2. `flow exporter`: どこに（コレクタ IP）、どのポート（UDP 2055等）で送るか。
    3. `flow monitor`: record と exporter を紐付け、インターフェイスに適用。
*   **ASA の NSEL 設定**: ASA では通常の FNF ではなく、**MPF (Modular Policy Framework)** を使用して設定します。`flow-export destination` コマンドと `policy-map` での適用が必要です。
*   **Stealthwatch との連携**: 
    *   コレクタ側でフローを受信できているか確認する。
    *   ラボ内では Stealthwatch Management Console (SMC) でフローが表示されることが「正解」の指標となります。
*   **インターフェイスの方向**: `ip flow monitor [NAME] input` を全インターフェイスの `input` に適用するのが一般的（重複カウントを防ぐため）。

---

## 🛠 設定方法

### 1. IOS-XE Flexible NetFlow (FNF)
```bash
! 1. レコードの定義
flow record MY-RECORD
 match ipv4 source address
 match ipv4 destination address
 match ipv4 protocol
 match transport source-port
 match transport destination-port
 collect counter bytes long
 collect counter packets long

! 2. エクスポーター（転送設定）
flow exporter MY-EXPORTER
 destination 192.168.10.100
 source GigabitEthernet1
 transport udp 2055

! 3. モニター（紐付け）
flow monitor MY-MONITOR
 exporter MY-EXPORTER
 record MY-RECORD

! 4. インターフェイス適用
interface GigabitEthernet2
 ip flow monitor MY-MONITOR input
```

### 2. ASA NSEL (NetFlow Security Event Logging)
```bash
! 転送先の設定
flow-export destination inside 192.168.10.100 2055

! MPF による適用
access-list NETFLOW-ACL extended permit ip any any
class-map NETFLOW-CLASS
 match access-list NETFLOW-ACL

policy-map global_policy
 class NETFLOW-CLASS
  flow-export event-type all destination 192.168.10.100
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **キャッシュ内のフロー確認** | <code>show flow monitor [NAME] cache</code> |
| **転送統計（エラーなど）の確認** | <code>show flow exporter statistics</code> |
| **適用インターフェイスの確認** | <code>show flow monitor [NAME]</code> |
| **ASA の転送状態確認** | <code>show flow-export counters</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| コレクタにフローが表示されない | Exporter の送信元（source）が未定義 | `source [interface]` を明示する。 |
| フローが断続的に途切れる | Active Timeout が長すぎる | `cache timeout active 60` 等で短縮する。 |
| ASA で特定の通信が記録されない | ACL または Class-map の不一致 | `show access-list` でヒット数を確認。 |
| 統計値が 0 のまま | `collect` コマンドの不足 | `flow record` 内で `collect counter` を追加。 |

---

## ⚠ 制限事項

*   **NSEL の特殊性**: NSEL はコネクションの状態変化（作成、削除、拒否）のみをエクスポートするため、ルータの NetFlow のように一定間隔でバイト数を更新し続ける動作はしません。
*   **サンプリング**: CPU 負荷が高い場合、`sampler` を使用して全パケットではなく N 分の 1 のみを抽出する設定が必要になる場合があります。

---

## 🔄 他技術との関連

*   **Cisco Secure Network Analytics (Stealthwatch)**: NetFlow は Stealthwatch が分析を行うための「原材料」です。
*   **DNAC (DNA Center)**: アプリケーションテレメトリとして NetFlow 設定を自動プロビジョニング可能です。
*   **NBAR (Network Based Application Recognition)**: FNF の record 内で `match application name` を使用することで、アプリ単位の可視化が可能です。

---

## 🧩 比較表

| 機能 | Flexible NetFlow | NSEL (ASA/FTD) |
| :--- | :--- | :--- |
| **主な対象** | ルータ・スイッチ | ファイアウォール |
| **エクスポート単位** | 定期的な統計情報 | イベント（Create/Delete/Deny） |
| **カスタマイズ性** | 非常に高い | 低い（固定イベント） |
| **設定手法** | Record/Exporter/Monitor | MPF (Modular Policy Framework) |

---

## 💡 ベストプラクティス

1.  **Ingress Only の原則**: 特殊な要件がない限り、各インターフェイスの `input` 方向のみに適用し、トラフィックの重複カウントと負荷を避けます。
2.  **UDP 2055 の使用**: 慣習的に NetFlow には UDP 2055 ポートを使用します。
3.  **タイムアウト値の調整**: Stealthwatch 連携時は、Active 60秒 / Inactive 15秒 への変更が推奨されます。

---

## 📝 ラボ学習・設定サンプル例

### 1. Stealthwatch 向けの基本構成
*   **要件**: R1 の全インターフェイスで IPv4 トラフィックを監視し、10.1.1.50 のコレクタへ送れ。
*   **設定**: Record に Src/Dst IP, Port, Proto を含め、全 IF の `input` に適用。

### 2. ASA での拒否トラフィック可視化
*   **要件**: ASA で拒否された通信（Deny）のみを NetFlow イベントとして送れ。
*   **設定**: `flow-export event-type flow-denied` を活用（または all）。

---

## ❓ 想定試験問題

1.  **Design**: Stealthwatch を導入する際、ルータ側でフロー情報の精度を上げるために調整すべきタイマー設定は？
    *   **回答**: `cache timeout active` を短く（例: 60秒）設定し、長時間セッションがコレクタ側でリアルタイムに反映されるようにする。
2.  **コンフィグ読解**: `ip flow monitor MON1 input` が Gi0/1 と Gi0/2 の両方に設定されている。Gi0/1 から Gi0/2 へ抜けるパケットは、コレクタ側でどう見えるか？
    *   **回答**: 各 IF の入力時にのみカウントされるため、1 つのフローとして正しく記録される。
3.  **トラブルシュート**: ASA で `show flow-export counters` を実行したが、"Records sent" が増えない。考えられる MPF 上の設定ミスは？
    *   **回答**: `service-policy` が `global` またはインターフェイスに正しく適用されていない、もしくは `class-map` でマッチするトラフィックが定義されていない。

---

## 🔗 参考リソース

*   **Cisco Live**: BRKSEC-2003 "Securing the Infrastructure with NetFlow"
*   **Configuration Guide**: [Cisco IOS Flexible NetFlow Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/fnetflow/configuration/xe-16/fnf-xe-16-book.html)
*   **ASA Guide**: [Configuring NetFlow Security Event Logging (NSEL)](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/asdm74/general/asdm-74-general-config/monitor-nsel.html)

---

## 📝 **補足（Notes）**
*   **学習メモ**: 「NetFlow はネットワークの防犯カメラの記録（ログ）」と考えると分かりやすいです。
*   **注意点**: ラボ試験では、設定後に実際に通信（Ping など）を発生させないと `show flow monitor cache` に何も表示されないため、必ずトラフィックを発生させてから検証してください。
