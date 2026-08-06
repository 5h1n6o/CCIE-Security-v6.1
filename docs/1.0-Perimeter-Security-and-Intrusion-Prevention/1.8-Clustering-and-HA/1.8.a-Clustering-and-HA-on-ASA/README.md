---
layout: default
title: 1.8.a Clustering and HA features on ASA
nav_order: 1
parent: 1.8-Clustering-and-HA
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.8.a Clustering and high availability features on Cisco ASA

Cisco ASA における**ハイアベイラビリティ (HA)** と**クラスタリング (Clustering)** は、ネットワークの継続性を確保し、スケーラビリティを向上させるための重要な基盤技術です。HA（主にフェイルオーバー）は 2 台のデバイス間での冗長性を提供し、クラスタリングは複数台（最大 16 台）を論理的な 1 台として動作させることで、スループットの拡張と冗長性を同時に実現します。

---

## 📘 概要

*   **機能概要**: 
    *   **Failover (HA)**: 同一モデルの ASA 2 台を接続し、プライマリユニットの故障時にセカンダリユニットが即座に通信を引き継ぐ機能です [1.8]。
    *   **Clustering**: 複数台（最大 16 台）の ASA を Cluster Control Link (CCL) で統合し、1 つの IP アドレス（論理的な 1 台）として管理・運用します [1.8]。
*   **利用目的**: 
    *   デバイス故障時のダウンタイム最小化。
    *   トラフィック量に合わせた動的な FW 性能の拡張（クラスタリング）。
*   **どのような場面で利用するか**: 
    *   データセンターやインターネット境界など、パケットロスが許されないミッションクリティカルな環境。
    *   fw 1 台の性能限界を超えるギガビット/テラビット級のトラフィック処理が必要な場合。

---

## 🔑 要点

| 項目 | Failover (Active/Standby) | Clustering |
| :--- | :--- | :--- |
| **最大ユニット数** | 2 台 | 最大 16 台 (プラットフォームによる) |
| **動作モード** | プライマリ/セカンダリ | Master / Slave (Slave はデータ処理を並列実行) |
| **状態同期** | フェイルオーバーリンク + ステートリンク | Cluster Control Link (CCL) |
| **ライセンス** | 同一ライセンスが必要 | クラスタリングライセンスが必要 (ASA) |
| **メリット** | 設定がシンプル、実績が多い | 高いスケーラビリティ (N+1 冗長) |
| **インターフェイス** | 共通 IP/MAC (仮想) | Spanned (共有 IP) または Individual (各台固有 IP) |

---

## 🏗 動作原理

### Failover
専用のフェイルオーバーリンクを介してハローメッセージ（Keepalive）を交換し、相方の生存を確認します [1.8]。また、ステートリンクにより TCP/UDP セッション情報や VPN 状態をリアルタイムに同期します。

```text
[ Primary (Active) ] <--- Failover/State Link ---> [ Secondary (Standby) ]
        |                                                  |
   (Active IP/MAC)                                  (Standby IP/MAC)
        |                                                  |
   [ Interface Down ] -----------------------------> [ Takeover IP/MAC ]
```

### Clustering
Cluster Control Link (CCL) を通じて Master ユニットが Slave ユニットを制御します。トラフィックは各ユニットに分散されますが、コネクションの制御（Owner）は 1 台が担当し、他はバックアップ（Director）として情報を持ちます [1.8]。

---

## ⚙ 動作シーケンス

1.  **ユニット選出**: 起動時にフェイルオーバーリンク/CCL を介して、ロール（Active/Standby または Master/Slave）をネゴシエーションします。
2.  **Config 同期**: Active (Master) ユニットの設定が自動的に Standby (Slave) へコピーされます。
3.  **状態同期 (Stateful)**: 確立されたセッション情報を随時同期し、切り替え後も再認証なしで通信を維持します。
4.  **障害検知**: IF ダウンや電源喪失を検知すると、数秒（最短数百ミリ秒）で切り替えが行われます。
5.  **トラフィックの再配布 (Cluster)**: クラスタ内の負荷状況に応じて、新しい接続を各ユニットへインテリジェントに割り振ります。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **インターフェイス監視**: `monitor-interface` の設定を忘れないようにします。どの IF が落ちたらフェイルオーバーさせるかの要件を読み取ります。
*   **Failover MAC-address**: ラボでは「切り替え時の ARP 学習時間を最小化せよ」といった要件に対し、仮想 MAC アドレスの手動設定が求められます。
*   **Active/Active Failover**: マルチコンテキストモード環境での冗長化設定が出題されます。各コンテキストを別の ASA で Active にする設定が重要です。
*   **CCL MTU**: クラスタリングでは、CCL インターフェイスの MTU をカプセル化オーバーヘッドを考慮して 1600 以上に設定する要件が頻出です。
*   **show コマンドの読み取り**: `show failover` や `show cluster info` の出力から、どのユニットが Active なのか、同期が取れているかを瞬時に判断する必要があります。

---

## 🛠 設定方法

### 1. Active/Standby Failover 基本設定 (ASA1)
```bash
failover
failover lan unit primary
failover lan interface F-LINK GigabitEthernet0/3
failover interface ip F-LINK 172.16.1.1 255.255.255.252 standby 172.16.1.2
failover link STATE-LINK GigabitEthernet0/4
failover interface ip STATE-LINK 172.16.2.1 255.255.255.252 standby 172.16.2.2
```

### 2. クラスタリングの有効化 (CCL 設定例)
```bash
cluster group ASA-CLUSTER
  key cisco123
  ccl interface CCL-PORT GigabitEthernet0/5
  enable
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **冗長状態の概要確認** | <code>show failover</code> |
| **フェイルオーバーの履歴表示** | <code>show failover history</code> |
| **クラスタの状態とメンバー確認** | <code>show cluster info</code> |
| **各ユニットのスループット確認** | <code>show cluster history</code> |
| **同期されているコネクション数** | <code>show conn count</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| Config が同期されない | ソフトウェアバージョンの不一致 | <code>show version</code> で両台を同一に揃える [1.8]。 |
| "Negotiation" から進まない | リンクの物理的な切断またはキーの不一致 | リンク状態の確認と <code>failover key</code> を再設定。 |
| 頻繁な切り替え (Flapping) | インターフェイスの閾値が敏感すぎる | <code>failover poll</code> タイムを調整する。 |
| メンバーがクラスタから外れる | ライセンス不足 | <code>show license</code> で各台に Clustering ライセンスがあるか確認。 |

---

## ⚠ 制限事項

*   **同一要件**: 両ユニットは、ハードウェアモデル、インターフェイス構成、RAM 容量が完全に同一である必要があります [1.8]。
*   **非ステートフルな通信**: ICMP 等の一部のプロトコルは、デフォルトでステートフルフェイルオーバーの対象外となるため、`inspect icmp` 等の設定が必要です。
*   **AnyConnect クラスタリング**: ASA バージョンや構成により、VPN セッションの完全な同期には特定の制限事項が存在します。

---

## 🔄 他技術との関連

*   **Routing**: HA 構成では、OSPF や BGP のネイバー関係も同期されます。
*   **Security Contexts**: Active/Active フェイルオーバーを実装するための前提条件です。
*   **NAT**: 動的 PAT などのエントリは、HA 切り替え後も維持されるようステートリンクで同期されます。

---

## 🧩 比較表

### Spanned vs Individual Interface (Clustering)

| 機能 | Spanned モード | Individual モード |
| :--- | :--- | :--- |
| **ネットワーク構成** | 全ユニットが L2 スイッチで 1 つのセグメントに繋がる | 各ユニットが独自の L3 接続を持つ |
| **IP 管理** | 単一の共有 IP アドレスを使用 | ユニットごとに異なる IP を割り当てる |
| **要件** | EtherChannel (LACP) をサポートした対向スイッチ | ダイナミックルーティング (OSPF/BGP) |

---

## 💡 ベストプラクティス

1.  **ステートフル同期の推奨**: パケットロスを防ぐため、必ず `failover link` を設定し、TCP 状態情報を同期させます。
2.  **物理的分離**: フェイルオーバーリンクとデータトラフィック用のリンクは、可能な限り物理的に分離（または EtherChannel 化）します。
3.  **仮想 MAC の活用**: `failover mac address` を設定して、切り替え時のスイッチ側の ARP テーブル更新を高速化します。
4.  **MTU の拡張**: クラスタリング CCL ではオーバーヘッドを吸収するため MTU 1600 以上を標準とします。

---

## 📝 ラボ学習・設定サンプル例

### 1. ユニットロールの固定
*   **要件**: ASA1 を常に Primary、ASA2 を常に Secondary にせよ。
*   **設定**: `failover lan unit primary` (ASA1) / `secondary` (ASA2)。

### 2. HTTP 接続のステートフル同期
*   **要件**: HTTP セッションをフェイルオーバー対象に含めよ。
*   **設定**: `failover replication http`。

### 3. インターフェイス障害の閾値設定
*   **要件**: 2 つ以上のインターフェイスがダウンした場合にのみフェイルオーバーさせよ。
*   **設定**: `failover interface-threshold 2`。

### 4. フェイルオーバータイマーの高速化
*   **要件**: インターフェイスハローを 200ms、ホールドを 1s に設定せよ。
*   **設定**: `failover poll interface 200 msec holdtime 1`。

### 5. クラスタリング個別（Individual）インターフェイス設定
*   **要件**: メンバーごとに異なる IP を持つ Individual モードで構成せよ。
*   **設定**: `interface <IF_NAME>`, `ip address <IP> <mask> cluster-pool <POOL_NAME>`。

### 6. SSL VPN セッションの冗長化
*   **要件**: クライアント VPN 接続を維持したまま切り替えを行え。

### 7. フェイルオーバー用キーの適用
*   **要件**: 制御パケットに認証キー `cisco123` を設定せよ。
*   **設定**: `failover key cisco123`。

### 8. クラスタ内でのパケットトレース
*   **課題**: 特定のパケットがどの Slave で処理されるか追跡せよ。
*   **コマンド**: `cluster exec packet-tracer input inside ...`。

### 9. デバッグによる冗長動作の確認
*   **操作**: `debug fover` を有効にし、リンクダウン時の挙動をログで確認せよ。

### 10. 管理プレーンの保護 (MPP) との併用
*   **要件**: 特定の HA/Cluster 管理インターフェイスからのアクセスのみを許可せよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: `show failover` でセカンダリ側が "App-Sync" 状態で止まっている。原因として考えられる最も可能性の高いものは？
    *   **回答**: ステートリンクの帯域不足、または MTU 設定のミスマッチ。
2.  **Design**: クラスタリングで 4 台構成にする際、対向スイッチ側で必須となる機能は？
    *   **回答**: LACP (802.3ad) によるマルチシャーシ EtherChannel (vPC/VSS)。
3.  **実装**: ASA クラスタにおいて、CCL ポートの MTU を 1500 のままにした場合に発生し得る事象を述べよ。
    *   **回答**: 制御情報の断片化が発生し、クラスタ全体の不安定化やパケットドロップを招く。
4.  **コンフィグ読解**: `monitor-interface inside` 設定がない場合、inside IF が物理ダウンしてもフェイルオーバーするか？
    *   **回答**: いいえ。管理用のフェイルオーバーリンクが無事な限り、データ IF のダウンだけでは切り替わりません。
5.  **Design**: Active/Active フェイルオーバーを構成するために必須となる ASA の動作モードは？
    *   **回答**: マルチコンテキストモード (Multiple Context Mode)。

---

## 🔗 参考リソース

*   [Cisco ASA Series CLI Configuration Guide, 9.4 - Failover](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/firewall/asa-94-firewall-config/ha-failover.html)
*   [Cisco ASA Cluster Configuration Guide](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/116035-asa-91-cluster-config-00.html)
*   [Technical Note: Troubleshooting ASA Failover](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/113685-asa-threat-detection-00.html)

---

## 📝 **補足（Notes）**  

*   **学習メモ**: HA は「2 台の完全同期」が鍵です。設定だけでなく、OS バージョンや物理的なインターフェイス番号（例えば Gi0/1 と Gi0/1 を繋ぐ）が一致していることが成功の近道です。
*   **図解**: 常にパケットの流れ（CCL を通るのか、直接抜けるのか）を意識して、非対称ルーティングの発生を考慮した設計を心がけましょう。
