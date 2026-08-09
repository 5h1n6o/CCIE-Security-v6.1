---
layout: default
title: 3.2.a-CPU
nav_order: 1
parent: 3.2-Management-plane
grand_parent: 3.0-Security-Infrastructure
---

# 3.2.a CPU

ネットワークデバイスにおける**CPU**の保護は、マネジメントプレーンおよびコントロールプレーンの安定性を維持するための最優先事項です。CCIE Security v6.1において、この項目は単なるハードニングを超え、管理トラフィックや制御パケットがCPUリソースを枯渇させてデバイスの動作を不安定にしたり、管理不能な状態（DoS攻撃など）に陥るのを防ぐための高度な保護手法を指します。

---

## 📘 概要

*   **機能概要**: デバイスのCPU宛てトラフィックを制御・制限するための技術群。主に **Management Plane Protection (MPP)**、**Control Plane Protection (CPPr)**、および **Selective Packet Discard (SPD)** などが含まれます。
*   **利用目的**: 不正な管理アクセスや過剰なルーティングプロトコルのアップデートからCPUを保護し、高い可用性を確保すること。
*   **どのような場面で利用するか**: 境界ルータでのインターネット側からの管理プロトコル遮断、大規模ネットワークにおけるBGP/OSPFパケットの優先処理、およびDDoS攻撃下での管理セッション維持のために利用します。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | トラフィックのタイプ（管理・転送・例外）に基づいてCPUへのパスを細分化して制御する。 |
| **用途** | 管理プロトコルのインターフェイス制限、CPU過負荷時の重要パケット優先保護。 |
| **メリット** | デバイス全体の安定性向上、攻撃下でも管理コンソールへのアクセスを維持可能。 |
| **デメリット** | 設定ミスにより正当な管理者が締め出される（Lock-out）リスクがある。 |
| **対応機種** | Cisco IOS, IOS-XE 全般。 |
| **制限事項** | MPPは物理インターフェイスに依存し、論理IF（Loopback等）には直接適用できない。 |
| **設計上の注意点** | セーフティネットとして常にコンソール接続を確保した状態で設定を行うこと。 |

---

## 🏗 動作原理

CPU保護メカニズムは、パケットがハードウェア転送を外れてCPUに「パンティング」される際の入り口で機能します。

```text
Incoming Packet
   ↓
[ Layer 2/3 Check ]
   ↓
[ Punted to CPU? ] --- No ---> [ Hardware Switching (ASIC) ]
   ↓ Yes
[ Management Plane Protection (MPP) ] --- 許可された物理IFか？
   ↓
[ Control Plane Protection (CPPr) ] --- Host/Transit/CEF-exceptionのどこ宛か？
   ↓
[ Selective Packet Discard (SPD) ] --- 輻輳時に優先すべきパケットか？
   ↓
[ CPU Processing (BGP, SSH, SNMP, etc.) ]
```

---

## ⚙ 動作シーケンス

1.  **インターフェイス検証 (MPP)**: パケットが到着した物理ポートが、設定された管理プロトコル（SSH/HTTPS等）を許可しているか照合します。
2.  **サブインターフェイス分類 (CPPr)**: パケットを3つのカテゴリ（自分宛、トンネル等の通過、ARP等の例外）に分類し、個別のポリシーを適用します。
3.  **しきい値判定 (SPD)**: CPUへの入力キューが混雑している場合、優先度の低いパケットを破棄し、ルーティング維持に必要なパケットを優先します。
4.  **プロトコル処理**: フィルタリングを通過した正当なパケットのみが、対応するCPUプロセスに渡されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **MPPの確実な実装**: 「管理トラフィックをGi1経由のSSHのみに制限せよ」という要件に対し、`control-plane management`下での設定が求められます。
*   **IPv6 SPDの微調整**: ラボ試験では、IPv6の安定性を高めるために、SPDのしきい値（`min-threshold`, `max-threshold`）や `extended-headroom` の具体的な数値指定が出題される傾向にあります。
*   **IP Options Drop**: CPU負荷を避けるための `ip options drop` 設定は、デバイスハードニングの基本として必須です。
*   **CoPPとの併用**: コントロールプレーン全体のレート制限（3.1.a CoPP）と、インターフェイスを縛るMPP（3.2.a）の違いを理解し、要件に合わせて正しく使い分ける必要があります。
*   **検証の重要性**: 設定後、`show management-interface` や `show ipv6 spd` で、期待通りの状態になっているか必ず確認してください。

---

## 🛠 設定方法

### 1. Management Plane Protection (MPP)
GigabitEthernet0/1 でのみ SSH と SNMP を許可する例。
```bash
control-plane management
 management-interface GigabitEthernet0/1 allow ssh snmp
 ! 他のインターフェイスからの管理アクセスは自動的にドロップされる
```

### 2. IPv6 Selective Packet Discard (SPD)
CPU負荷が高い状況下で IPv6 パケットを保護する設定。
```bash
ipv6 spd mode aggressive
ipv6 spd queue min-threshold 80
ipv6 spd queue max-threshold 100
spd extended-headroom 20
```

### 3. 追加のCPU保護（IP Options / Logging）
```bash
! CPU処理を要するIPオプションパケットを破棄
ip options drop
! ログ生成によるCPU負荷を抑える
logging rate-limit 100
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **MPP設定・有効IFの確認** | <code>show management-interface</code> |
| **IPv6 SPDの状態確認** | <code>show ipv6 spd</code> |
| **CPUプロセスと負荷の確認** | <code>show processes cpu sorted</code> |
| **IPオプションドロップ統計** | <code>show ip traffic</code> |
| **コントロールプレーン全体の統計** | <code>show control-plane all</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 許可IPなのにSSHできない | MPPで当該IFが許可されていない | <code>show management-interface</code> | <code>control-plane management</code>にIFを追加。 |
| 高負荷時に管理応答が極端に遅い | SPDの <code>headroom</code> 不足 | <code>show ipv6 spd</code> | <code>spd extended-headroom</code> の値を増やす。 |
| ルーティングが切れる | CPPr/CoPPで制御通信をドロップ | <code>show control-plane host</code> | ルーティングプロトコルを <code>transmit</code> するクラスを修正。 |
| デバイス全体が重い | IPオプションによるCPU高騰 | <code>show ip traffic</code> | <code>ip options drop</code> を有効化。 |

---

## ⚠ 制限事項

*   **物理IFの制約**: MPPは、パケットが入ってくる物理的なインターフェイスで判断します。Loopback宛であっても、そこに至る物理ポートが許可されていなければなりません。
*   **プラットフォーム依存**: CPPrの3つのサブインターフェイス（Host, Transit, CEF-exception）は、すべてのIOSバージョンで完全にサポートされているわけではありません。
*   **ロックアウト**: 管理IFを1つに限定した場合、そのIFや対向スイッチに障害が起きると、リモートから二度とログインできなくなります。

---

## 🔄 他技術との関連

*   **3.1.a CoPP**: CPUに向かうトラフィックを「量（bps）」で制限するのに対し、MPPは「場所（IF）」で制限します。
*   **3.2.c Securing device access**: MPPで物理IFを絞り、さらに `line vty` の `access-class` でIPアドレスを絞る多層防御がベストプラクティスです。
*   **3.6.c SYSLOG**: `logging rate-limit` により、大量のログ出力に伴うCPU高負荷を回避します。

---

## 🧩 比較表

### MPP vs CoPP vs CPPr

| 技術 | 制御の軸 | 主な目的 |
| :--- | :--- | :--- |
| **MPP** | 受信物理インターフェイス | 管理アクセスの入り口を限定する。 |
| **CoPP** | トラフィックのレート (MQC) | CPU宛てパケットを全体的に流量制限する。 |
| **CPPr** | CPU内部のサブインターフェイス | CPU宛てトラフィックを性質別に細かく制御する。 |

---

## 💡 ベストプラクティス

1.  **管理専用ポートの使用**: 可能な限り、サービスが流れない独立した管理用IF（Mgmtポート）でのみ管理アクセスを許可します。
2.  **IPv6 SPDの有効化**: モダンなネットワークでは、輻輳時のプロトコル保護のために `ipv6 spd mode aggressive` を推奨します。
3.  **IP Options のドロップ**: 特殊なデバッグを行わない限り、常に `ip options drop` を設定して攻撃面を減らします。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な MPP による SSH 制限
*   **要件**: GigabitEthernet1 でのみ SSH を許可せよ。
*   **設定**: `control-plane management` > `management-interface Gi1 allow ssh`。

### 2. IPv6 高可用性のための SPD 設定
*   **要件**: CPU 混雑時でも IPv6 近隣探索を保護せよ。
*   **設定**: `ipv6 spd mode aggressive` + `spd extended-headroom 20`。

### 3. IP Options 攻撃の緩和
*   **要件**: ソースルーティングなどの IP オプションを含むパケットを破棄せよ。
*   **設定**: `ip options drop`。

### 4. CPPr による自分宛トラフィックの保護
*   **要件**: 自分宛（Host）の ICMP トラフィックを 100Kbps に警察（Police）せよ。

### 5. SNMP マネジメントインターフェイスの指定
*   **要件**: Gi2 でのみ SNMP ポーリングを受け付けよ。

### 6. Logging 過負荷の防止
*   **要件**: 毎秒 10 メッセージ以上のログ生成を制限せよ。
*   **設定**: `logging rate-limit 10`。

### 7. MPP 適用後の疎通確認
*   **課題**: 許可されていない IF から SSH を試み、接続が拒否されることを確認せよ。

### 8. SPD キューしきい値の変更
*   **要件**: キューが 90% でドロップを開始するようにせよ。

### 9. CEF-exception トラフィックの監視
*   **課題**: ARP や TTL Exceeded パケットが CPU に与える影響を確認せよ。

### 10. 全コントロールプレーン統計のリセット
*   **課題**: 攻撃テストの前に `clear control-plane *` で統計をリセットせよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `management-interface Gi1 allow ssh` があるルータに、Gi2 から SSH 接続できるか？
    *   **回答**: できない。MPP によって Gi1 以外からの SSH 着信はドロップされる。
2.  **トラブルシュート**: `ipv6 spd mode aggressive` を設定したが、`show ipv6 spd` で `Current mode: normal` と表示される。なぜか？
    *   **回答**: 実際の輻輳（キューのしきい値到達）が発生していないため。動作自体は正常。
3.  **Design**: CPU を DDoS 攻撃から守るために、特定の管理ネットワーク以外からの全トラフィックを入口で遮断したい。MPP 以外に併用すべき技術は？
    *   **回答**: iACL (Infrastructure ACL) を外部物理 IF に適用する。
4.  **実装**: IP Source Routing パケットによる CPU 負荷を防ぐコマンドは？
    *   **回答**: `ip options drop`。
5.  **コンフィグ読解**: CPPr の `Host` サブインターフェイスポリシーに適用した設定は、誰宛のトラフィックに影響するか？
    *   **回答**: ルータ自身の IP アドレスを宛先とするトラフィック。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [Configuring Management Plane Protection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_mpp/configuration/xe-16/sec-data-mpp-xe-16-book.html)
    *   [Configuring Control Plane Protection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_policing/configuration/xe-16/qos-policing-xe-16-book/qos-policing-control-plane.html)
*   **Technical Notes**
    *   [Cisco Guide to Harden Cisco IOS Devices](https://www.cisco.com/c/en/us/support/docs/ip/access-lists/13608-21.html)
*   **CCIE Security v6.1 Learning Matrix**

---

## 📝 **補足（Notes）**
- **学習メモ**: MPP は「物理ポートの鍵」、CPPr/CoPP は「流量の蛇口」とイメージすると区別しやすいです。
- **図解**: パケットが CPU に届くまでに、何重ものフィルターを通る様子を書き出してみてください。
- **注意点**: ラボ試験で `ip options drop` を設定すると、特定のデバッグ用 ping（オプション付き）が通らなくなる場合があります。要件でデバッグが必要とされていないか確認してください。
