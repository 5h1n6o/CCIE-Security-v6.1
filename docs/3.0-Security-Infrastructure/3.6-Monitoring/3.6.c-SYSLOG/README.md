---
layout: default
title: 3.6.c-SYSLOG
nav_order: 3
parent: 3.6-Monitoring
grand_parent: 3.0-Security-Infrastructure
---

# 3.6.c SYSLOG

**SYSLOG**は、ネットワークデバイスが生成するイベントメッセージを収集、記録、転送するための標準プロトコルです。CCIE Security v6.1において、Syslogは単なるログ記録機能ではなく、**脅威の特定（Threat Identification）**、**インフラの要塞化（System Hardening）**、および**事後解析（Forensics）**に不可欠な監視コンポーネントとして位置づけられています。

---

## 📘 概要

*   **機能概要**: デバイス内で発生したソフトウェア、ハードウェアの状態変化や、セキュリティポリシー（ACLなど）へのマッチングをメッセージとして生成し、ローカルバッファや外部サーバへ送信します。
*   **利用目的**: セキュリティインシデントの検知、監査ログの保持、トラブルシューティング時のトラフィック挙動確認。
*   **利用場面**:
    *   **ACLによる拒否パケットの監視**: `log`オプションを使用して不正アクセス試行を特定する。
    *   **ファイアウォールのデバッグ**: ASAなどのデバイスでパケットがドロップされた原因を特定する。
    *   **集中管理**: 複数のデバイスのログをCisco DNA CenterやSIEMに集約し、テレメトリとして活用する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | 重要度（Severity）に基づいたメッセージ管理（0-7のレベル）。 |
| **用途** | 監視、診断、コンプライアンス（PCI-DSS/ISO 27001）への適合。 |
| **メリット** | トラフィックのメタデータを永続的に記録でき、異常検知のトリガーとなる。 |
| **デメリット** | 大量のログ生成によりデバイスのCPU負荷が増大するリスクがある。 |
| **対応機種** | ルータ、スイッチ、ASA、FTD、WSA、ISEなどほぼ全てのCisco製品。 |
| **制限事項** | 標準のSyslog（UDP 514）は平文であり、到達保証がない。 |
| **設計上の注意点** | ログには正確な時刻（NTP同期）が必須であり、レート制限を設定すべき。 |

---

## 🏗 動作原理

Syslogは、メッセージを生成する「エージェント」と、それを受け取る「コレクタ」のモデルで動作します。

```
[ ネットワークデバイス (Agent) ]
      ↓ 1. イベント発生（ACL Deny, Interface Down等）
      ↓ 2. メッセージ生成（SeverityとIDの付与）
      ├──────────→ [ Console ] (リアルタイム表示)
      ├──────────→ [ Buffered ] (メモリ内保存)
      └──────────→ [ Syslog Server (Collector) ] (UDP/TCP転送)
```

---

## ⚙ 動作シーケンス

1.  **イベント検知**: デバイスが特定の条件（例：ACLでのパケット拒否）を検知します。
2.  **フォーマット**: メッセージが生成されます。標準形式は `%FACILITY-SEVERITY-MNEMONIC: Description` です。
3.  **フィルタリング**: 設定された `logging trap` レベルに基づき、送信するか破棄するかを決定します。
4.  **エクスポート**: 指定された宛先（外部サーバ等）へプロトコル（デフォルトはUDP 514）を使用して転送します。
5.  **ローカル記録**: メモリ内のバッファ（Buffered Logging）に記録し、後で `show logging` で確認可能にします。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **重要度レベル (Severity Levels)**: 0（Emergency）から 7（Debugging）までの各レベルの意味を暗記しておく必要があります。ラボでは「レベル 6（Informational）以上のログをサーバへ送れ」といった要件が出ます。
*   **ACL Logging**: `access-list ... deny ... log` または `log-input` オプションの設定。送信元MACアドレスや入力インターフェイス情報を含むメッセージの生成方法が問われます。
*   **ASAの特定メッセージ管理**: ASAでは `logging message [ID] level [Level]` を使用して、特定のメッセージID（例：302013）の重要度を変更する高度な設定が求められることがあります。
*   **CPU保護**: `logging interval` を設定し、短時間に大量の同一ログが生成されてCPUを圧迫するのを防ぐ「System Hardening」の観点での設定が必要です。
*   **時刻の正確性**: `service timestamps log datetime msec` などのコマンドで、ログにミリ秒単位の正確な時刻を付与することが、証拠能力（Forensics）の観点から重要視されます。

---

## 🛠 設定方法

### 1. IOS-XE 基本設定
```bash
! タイムスタンプにミリ秒と日付を付与（重要）
service timestamps log datetime msec
! 外部Syslogサーバの設定
logging host 192.168.10.100
! 送出する重要度の指定（Informational以上）
logging trap 6
! ローカルバッファの設定
logging buffered 16384
```

### 2. ACLと連携したログ生成例
```bash
ip access-list extended SECURE-ACL
 deny ip 10.10.10.0 0.0.0.255 any log
 permit ip any any
```

### 3. ASAでのデバッグ向けログ設定
```bash
! バッファに最高レベル(7)で記録し、後でトラブルシュートに利用
logging enable
logging buffered 7
logging on
! 特定のID(106023)のメッセージを有効化
no logging message 106023
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **ログバッファの内容確認** | <code>show logging</code> |
| **特定のメッセージを抽出** | <code>show logging \| include [文字列]</code> |
| **ログ設定の統計確認** | <code>show logging count</code> |
| **ASAでのログ状態確認** | <code>show logging</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 外部サーバにログが届かない | ルーティングまたはACL遮断 | サーバIPへの疎通を確認し、UDP 514が許可されているか確認。 |
| ログの時刻がずれている | NTP同期不足 | <code>show ntp status</code> を確認。 |
| バッファがすぐに上書きされる | バッファサイズ不足 | <code>logging buffered [size]</code> を増やす。 |
| 必要なログが表示されない | <code>logging on</code> の欠落 | <code>logging on</code> が設定にあるか確認。 |
| CPU負荷が高い | ログ生成頻度が高すぎる | <code>logging rate-limit</code> を設定する。 |

---

## ⚠ 制限事項

*   **バッファの揮発性**: デバイスを再起動すると、`logging buffered` に保存されたログは失われます。
*   **UDPの非信頼性**: 標準のSyslog送信はUDPのため、ネットワーク混雑時にログパケットが失われる可能性があります。
*   **パフォーマンス**: デバッグレベル（7）を常時有効にすると、ハードウェアのリソースを大幅に消費します。

---

## 🔄 他技術との関連

*   **3.1.a CoPP**: ログメッセージの生成プロセスの優先順位を制御し、管理プレーンを保護します。
*   **3.4.g VACL**: `action forward capture` と組み合わせて、特定のL2トラフィックイベントを監視します。
*   **3.10 Cisco DNAC API**: DNA CenterがSyslogをテレメトリデータとして収集し、ネットワーク全体の可視化に利用します。
*   **NTP**: 複数のデバイス間でログのタイムラインを一致させるために必須のサービスです。

---

## 🧩 比較表

### IOS-XE Syslog vs ASA Syslog

| 特徴 | IOS-XE | ASA |
| :--- | :--- | :--- |
| **デフォルトプロトコル** | UDP 514 | UDP 514 (TCPも可) |
| **メッセージ識別** | 文字列（Mnemonic） | 数字ID（Message ID） |
| **レート制限** | 全体制限 | メッセージIDごとの制限 |
| **出力先** | コンソール、バッファ、ホスト | バッファ、ホスト、ASDM、Mail |

---

## 💡 ベストプラクティス

1.  **正確な時刻同期**: すべてのログにNTP同期されたタイムスタンプを付与します。
2.  **ログサーバの冗長化**: 単一障害点を防ぐため、複数のSyslogサーバへ転送します。
3.  **階層的なログレベル**: コンソールには `Critical`、バッファには `Debugging`、外部サーバには `Informational` といった使い分けを行います。
4.  **レートリミットの適用**: `logging rate-limit` を使用して、インフラの可用性を維持します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的なバッファロギング
*   **要件**: ログをメモリ内に1MB分保存し、レベルは `Debugging` とせよ。
*   **設定**: `logging buffered 1000000 7`。

### 2. 外部サーバへのセキュア転送
*   **要件**: 10.1.1.1 のサーバへ `Warning` 以上のログを送信せよ。
*   **設定**: `logging host 10.1.1.1`, `logging trap 4`。

### 3. ACL拒否ログの有効化
*   **要件**: 不正な1918アドレスからのICMPを拒否し、その履歴を記録せよ。
*   **設定**: `deny icmp 10.0.0.0 0.255.255.255 any log`。

### 4. タイムスタンプのカスタマイズ
*   **要件**: ログにローカルタイムゾーンとミリ秒を含めよ。
*   **設定**: `service timestamps log datetime msec localtime show-timezone`。

### 5. 送信元インターフェイスの固定
*   **要件**: Syslogパケットの送信元を Loopback 0 に固定せよ。
*   **設定**: `logging source-interface Loopback0`。

### 6. ASAでのパケットドロップ追跡
*   **要件**: ACLでドロップされたパケットをログで確認できるよう設定せよ。
*   **設定**: `logging enable`, `logging list DROP_LIST message 106023`。

### 7. ログのレート制限
*   **要件**: コンソールへのログ出力を秒間10個に制限せよ。
*   **設定**: `logging rate-limit 10 console`。

### 8. メッセージ本文の加工
*   **要件**: ログにデバイスのホスト名を含めよ。
*   **設定**: `logging origin-id hostname`。

### 9. 特定のメッセージのみフィルタリング
*   **要件**: 設定変更（Config）のログのみをレベル 5 で送れ。

### 10. DNAC連携のためのSyslog設定
*   **課題**: テレメトリ収集のため、DNA Centerのアドレスをログ送信先に設定せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `logging trap 3` が設定されている環境で、`%SEC-6-IPACCESSLOGDP` は外部サーバに送信されるか？
    *   **回答**: 送信されない。Trapレベルが 3 (Errors) のため、レベル 6 (Informational) のメッセージは除外される。
2.  **トラブルシュート**: `show logging` を実行しても何も表示されない。トラブルシューティングのために確認すべき最初のコマンドは？
    *   **回答**: `logging on` がグローバル設定に含まれているか確認する。
3.  **Design**: 大規模ネットワークでSyslogメッセージによるCPU負荷が懸念される。実装すべき「Hardening」設定は？
    *   **回答**: `logging rate-limit` による流量制限と、不要なデバッグログ（レベル7）の無効化。
4.  **実装**: ラボ試験で「ACLにマッチしたパケットの送信元MACアドレスをログに含めよ」と指示された場合に使用するキーワードは？
    *   **回答**: ACLの末尾に `log-input` を追加する。
5.  **コンフィグ読解**: ASAにおいて `sh logging` でドロップログを確認している。パケットが「Implicit Rule」で落ちている場合、どのキーワードを探すべきか？
    *   **回答**: `Deny inbound ... by configured rule` やメッセージ ID `106023`。

---

## 🔗 参考リソース

*   **Cisco Live (BRKSEC-2003)**: [Securing the Infrastructure with Logging](https://www.ciscolive.com/)
*   **Cisco Configuration Guide**: [Configuring System Message Logging](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/bsm/configuration/xe-16/bsm-xe-16-book/bsm-logging.html)
*   **Technical Notes**: [Syslog Messages on Cisco ASA](https://www.cisco.com/c/en/us/support/docs/security/pix-500-series-security-appliances/63884-config-syslog-asa.html)
*   **Design Guide**: [Cisco SAFE Design Guide - Monitoring](https://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/Security/SAFE_RG/SAFE_rg/chap9.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「Syslogはネットワークの目撃証言」です。証言には正確な時刻（NTP）と、目撃者の身元（Origin-ID）が必要であると覚えましょう。
*   **図解**: `logging buffered` はトラブルシューティングにおける「最強のレコーダー」です。パケットトレーサーやキャプチャが使えない環境でも、ログは真実を語ります。
*   **注意点**: ラボ試験では、`logging trap` と `logging buffered` のレベル設定を混同しないようにしてください。前者は「外へ送る基準」、後者は「中で貯める基準」です。
