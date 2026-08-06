---
layout: default
title: 1.8-Clustering-and-HA
nav_order: 8
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.8 Clustering and high availability features on Cisco ASA and Cisco FTD

Cisco ASA および Cisco FTD における**クラスタリング（Clustering）**と**ハイアベイラビリティ（High Availability: HA）**は、ネットワークの可用性とスケーラビリティを確保するための不可欠な機能です。HA は主に 2 台のデバイスによる冗長性（Active/Standby または Active/Active）を提供し、クラスタリングは複数台（最大 16 台等、モデルによる）を 1 台の論理的なデバイスとして動作させることで、スループットの向上と冗長性の両立を実現します。

---

## 📘 概要

*   **機能概要**: 
    *   **High Availability (Failover)**: 2 台の同一ユニット間で状態情報（コネクション、VPN セッション等）を同期し、障害時に瞬時に切り替える機能です。
    *   **Clustering**: 複数ユニットを統合して単一のエンティティとして動作させ、トラフィックを分散処理します。
*   **利用目的**: 
    *   単一障害点（SPOF）の排除。
    *   メンテナンス中の通信維持。
    *   トラフィック急増に伴う動的なスループット拡張（クラスタリング）。
*   **どのような場面で利用するか**: 
    *   インターネット境界やデータセンターのゲートウェイなど、ダウンタイムが許されない重要拠点。
    *   1 台の FW 性能では不足する大規模なトラフィック処理が必要な環境。

---

## 🔑 要点

| 項目 | Failover (HA) | Clustering |
| :--- | :--- | :--- |
| **最大ユニット数** | 2 台 | 最大 16 台 (モデルに依存) |
| **主な動作モード** | Active/Standby, Active/Active | All-Active (Spanned/Individual) |
| **インターフェイス構成** | 共通 IP (Standby IP を設定可能) | Spanned (L2) または Individual (L3) |
| **ライセンス要件** | 同一ライセンスが必要 | クラスタリングライセンス (ASA の場合) |
| **ハードウェア要件** | 同一モデル、同一メモリ、同一 IF 構成 | 同一モデル、同一 IF 構成 |
| **ソフトウェア要件** | 同一バージョン、同一 VDB (FTD) | 同一バージョン、同一 VDB (FTD) |

---

## 🏗 動作原理

### High Availability (Failover)
2 台のデバイスを専用の **Failover Link** と **State Link** で接続します。Failover Link はハローパケットによる生存確認、State Link はコネクション情報の同期に使用されます。

```text
[ Unit 1 (Active) ] <--- Failover/State Link ---> [ Unit 2 (Standby) ]
        |                                                 |
   (Processes Traffic)                            (Monitors Health)
        |                                                 |
   [ Interface Fail ] ----------------------------> [ Take Over ]
```

### Clustering
**Cluster Control Link (CCL)** を介して、コントロールプレーンの制御情報とデータプレーンのパケット転送（オーナーへの転送等）を行います。

```text
       [ External Switch (VPC/VSS) ]
             /      |      \
[ Unit 1 ] ---- [ Unit 2 ] ---- [ Unit 3 ]  <-- CCL (Cluster Control Link)
             \      |      /
       [ Internal Switch (VPC/VSS) ]
```

---

## ⚙ 動作シーケンス

1.  **初期化**: 各ユニットが自身のハードウェアおよびソフトウェアが要件を満たしているか確認します。
2.  **ネゴシエーション**: Failover/CCL リンクを介してハローパケットを交換し、ロール（Master/Slave または Active/Standby）を決定します。
3.  **構成同期**: 設定（Config）および動的な状態（State）情報を Master から Slave へ配布します。
4.  **障害検知**: インターフェイスのダウン、電源喪失、またはキープアライブの喪失を検知します。
5.  **切り替え (Failover)**:
    *   Standby ユニットが Active に昇格。
    *   GARP (Gratuitous ARP) を送信し、MAC アドレスと IP の紐付けをネットワークに通知します。
6.  **リカバリ**: 障害ユニットが復旧後、設定に基づき Standby として復帰、または Preempt 設定がある場合は Active に戻ります。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **インターフェイス監視**: `monitor-interface` の設定。どの IF がダウンしたら Failover させるかという条件（閾値）の設定が重要です。
*   **Failover Mac-address**: 切り替え時のパケットロスを最小限にするため、仮想的な MAC アドレスを手動で割り当てる設定が推奨されます。
*   **FTD HA の管理**: FMC 経由での設定手順。FMC がデバイスを正しく認識し、同期が取れていることを確認するログの読み取り。
*   **Clustering の CCL 設定**: CCL インターフェイスの MTU サイズ。カプセル化オーバーヘッドを考慮し、通常より大きく（1600 以上）設定する要件が出ることがあります。
*   **Stateful Failover の対象**: HTTP や TCP は同期されますが、一部の動的ルーティングプロトコルやマルチキャストの状態同期には制限がある点を把握してください。

---

## 🛠 設定方法

### 1. ASA Active/Standby Failover (CLI)
```bash
! Primaryユニットの設定
failover
failover lan unit primary
failover lan interface F-LINK GigabitEthernet0/3
failover interface ip F-LINK 172.16.1.1 255.255.255.252 standby 172.16.1.2
failover link S-LINK GigabitEthernet0/4
failover interface ip S-LINK 172.16.2.1 255.255.255.252 standby 172.16.2.2
```

### 2. FTD HA の構成 (FMC GUI)
1.  **Devices > Device Management** に移動。
2.  **Add > High Availability** を選択。
3.  Primary デバイスと Secondary デバイスを選択。
4.  Failover Link と State Link に使用する物理インターフェイスを指定。
5.  IP アドレス（Active/Standby 用）を入力し、**Form HA** をクリック。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **Failover 状態確認** | `show failover` |
| **Failover 履歴と原因** | `show failover history` |
| **同期されているコネクション数** | `show conn count` |
| **クラスタの状態確認** | `show cluster info` |
| **CCL の統計確認** | `show cluster history` |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| Config が同期されない | Failover Link の疎通不良 | `ping` (Failover IP) | リンクの物理状態と ACL を確認。 |
| 頻繁な切り替え (Flapping) | インターフェイスハローの欠落 | `show failover history` | インターフェイスの負荷やエラーを確認。 |
| クラスタから離脱する | バージョンの不一致 | `show version` | 全ユニットを同一 OS バージョンに揃える。 |
| Failover リンクが DOWN | MTU の不一致 | `show interface` | リンクの両端で MTU サイズを一致させる。 |

---

## ⚠ 制限事項

*   **非対称ルーティング**: クラスタリング環境では、戻りパケットが別のユニットに届く「非対称ルーティング」が発生しやすく、CCL を介したパケット転送（Forwarding）が発生しスループットが低下します。
*   **FTD HA の管理リンク**: 管理インターフェイス（Diagnostic）を Failover リンクとして共有することはサポートされません。
*   **AnyConnect セッション**: 特定の ASA バージョンを除き、クラスタリングにおける AnyConnect の完全なステートフル同期には制限があります。

---

## 🔄 他技術との関連

*   **Routing**: HA 構成では OSPF や BGP のネイバー関係も同期されます。クラスタリングでは、Master ユニットがルーティングプロセスを管理します。
*   **VPN**: Site-to-Site VPN はステートフル Failover の対象です。
*   **NAT**: 動的パット（PAT）のエントリ同期は Failover 時の通信維持に不可欠です。

---

## 🧩 比較表

### Spanned vs Individual Interface (Clustering)

| 特徴 | Spanned (L2) | Individual (L3) |
| :--- | :--- | :--- |
| **IPアドレス** | 全ユニットで共有の仮想 IP | 各ユニットが独自の IP を持つ |
| **ネットワーク構成** | スイッチ側で EtherChannel が必要 | スイッチ側は通常の L3 接続 |
| **可用性** | ユニット障害時に他が引き継ぐ | ルーティングによる切り替え |
| **推奨環境** | 透明 (Transparent) モード | 複雑な L3 ネットワーク |

---

## 💡 ベストプラクティス

1.  **専用リンクの確保**: Failover リンクと State リンクは、可能な限り物理的に分離された 10G 以上のインターフェイスを使用します。
2.  **MAC アドレスの手動設定**: `failover mac address` コマンドで仮想 MAC を設定し、スイッチの ARP テーブル更新をスムーズにします。
3.  **NTP の同期**: ログのタイムスタンプを一致させ、トラブルシューティングを容易にするため、全ユニットで NTP を同期します。
4.  **CCL の信頼性**: クラスタリングでは CCL の帯域がボトルネックになるため、冗長化された EtherChannel で構成します。

---

## 📝 ラボ学習・設定サンプル例

### 1. ASA Active/Standby 基本設定
*   **要件**: ASA1 を Primary、ASA2 を Secondary とし、Gi0/3 を Failover リンクに使用せよ。
*   **設定**: `failover lan unit primary`, `failover lan interface F-LINK Gi0/3`, `failover interface ip F-LINK 10.0.0.1 255.255.255.252 standby 10.0.0.2`。
*   **検証**: `show failover` で `This host: Primary - Active` を確認。

### 2. インターフェイス監視の追加
*   **要件**: Inside と Outside インターフェイスを監視対象に加え、一方が落ちたら Failover させよ。
*   **設定**: `monitor-interface inside`, `monitor-interface outside`。

### 3. Failover タイマーの調整
*   **要件**: 障害検知を高速化するため、インターフェイスハローを 500ms に設定せよ。
*   **設定**: `failover poll interface 500 msec`。

### 4. FTD HA の作成（FMC 操作）
*   **要件**: FMC 7.x を使用して 2 台の FTD 仮想ユニットで HA を組め。
*   **手順**: FMC GUI > Devices > Add High Availability。

### 5. クラスタリング CCL 設定
*   **要件**: CCL インターフェイスの MTU を 1600 に設定せよ。
*   **設定**: `interface <CCL_IF>`, `mtu 1600`。

### 6. 仮想 MAC アドレスの割り当て
*   **要件**: Failover 時のパケットロス防止のため、MAC アドレスを固定せよ。
*   **設定**: `failover mac address outside 00aa.bbcc.0001 00aa.bbcc.0002`。

### 7. ステートフル Failover の有効化
*   **要件**: コネクション情報の同期を有効化せよ。
*   **設定**: `failover link <IF_NAME> <IF_ID>`。

### 8. Failover Group の設定 (ASA Active/Active)
*   **要件**: マルチコンテキストモードで Failover Group 1 を ASA1、Group 2 を ASA2 で Active にせよ。
*   **設定**: `failover group 1`, `primary`, `failover group 2`, `secondary`。

### 9. ユニットの手動切り替え
*   **要件**: Active ユニットを強制的に Standby に変更せよ。
*   **コマンド**: `failover active` (Standby側で実行) または `no failover active` (Active側)。

### 10. クラスタ内のパケットトレース
*   **要件**: 特定のトラフィックがどのユニットで処理されているか確認せよ。
*   **コマンド**: `cluster exec packet-tracer input inside tcp 10.1.1.1 1234 10.2.2.2 80`。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: `show failover` を実行すると `Negotiation` 状態のまま進まない。想定される原因を 3 つ挙げよ。
    *   **回答**: 物理リンクのダウン、共有パスワードの不一致、ソフトウェアバージョンの不一致。
2.  **Design**: クラスタリングにおいて Spanned モードを使用する場合、対向スイッチ側で必要な機能は何か？
    *   **回答**: LACP (802.3ad) をサポートしたマルチシャーシ EtherChannel (vPC, VSS 等)。
3.  **実装**: FTD HA において、State Link に設定した IP アドレスを変更する手順を述べよ。
    *   **回答**: FMC 上で一度 HA を解消（Break）し、再構成する際に新しい IP を指定する。
4.  **コンフィグ読解**: `failover replication http` 設定の意味を述べよ。
    *   **回答**: HTTP 接続の状態情報を Standby ユニットに同期し、切り替え後も HTTP セッションを維持する。
5.  **Design**: クラスタリングのノード数を増やす際、CCL インターフェイスの帯域設計で考慮すべき点は何か？
    *   **回答**: 各ユニットが受け取ったパケットをオーナーに転送する「Forwarding トラフィック」の量。

---

## 🔗 参考リソース

*   [Cisco Secure Firewall Management Center Administration Guide, 7.0 - High Availability](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/device_high_availability.html)
*   [Cisco ASA Series CLI Configuration Guide, 9.4 - Failover](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/firewall/asa-94-firewall-config/ha-failover.html)
*   [Cisco Live: BRKSEC-3020 - Troubleshooting Firepower High Availability](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: HA とクラスタリングは概念は似ていますが、設定方法とトラブルシューティングのポイントが大きく異なります。ASA の CLI 設定と FMC の GUI 操作の両方に慣れておく必要があります。
*   **注意点**: ラボ試験では、デバイスの VDB バージョンが僅かでも異なると FTD HA の構成に失敗することがあります。設定前に必ず System > Updates でバージョンを確認してください。
