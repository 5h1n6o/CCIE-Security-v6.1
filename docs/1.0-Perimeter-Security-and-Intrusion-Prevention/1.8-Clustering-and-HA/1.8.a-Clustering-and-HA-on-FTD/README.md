---
layout: default
title: 1.8-Clustering-and-high-availability-features-on-Cisco-FTD
parent: 1.8-Clustering-and-HA
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.8 Clustering and high availability features on Cisco FTD

Cisco Secure Firewall (FTD) における **ハイアベイラビリティ (High Availability: HA)** と **クラスタリング (Clustering)** は、エンタープライズネットワークにおいてサービスの中断を最小限に抑え、トラフィックの増大に柔軟に対応するための重要な機能です。FTD は ASA と同様の基盤技術を継承しつつ、Firepower Management Center (FMC) による一元管理下で高度な冗長構成を実現します。

---

## 📘 概要

*   **機能概要**: 
    *   **High Availability (HA)**: 2 台の同一ユニット間でステート情報を同期し、障害時に通信を引き継ぐ **Active/Standby** 冗長構成です。
    *   **Clustering**: 複数台（モデルにより最大 16 台）のデバイスを論理的な 1 台のデバイスとして統合し、パフォーマンスのスケーラビリティと冗長性を同時に提供します。
*   **利用目的**: ハードウェア障害、インターフェイス障害、ソフトウェアフリーズが発生した際の通信維持、およびスループットの拡張。
*   **どのような場面で利用するか**: データセンター、インターネット境界、または大量のパケット検査（IPS/AMP）が必要な高負荷環境。

---

## 🔑 要点

| 項目 | FTD HA (Failover) | FTD Clustering |
| :--- | :--- | :--- |
| **動作モード** | **Active/Standby** のみ | **All-Active** (並列処理) |
| **最大ユニット数** | 2 台 | 最大 16 台（ハードウェアに依存） |
| **同期方式** | Failover リンク + State リンク | Cluster Control Link (CCL) |
| **管理主体** | FMC または FDM | 原則として **FMC による一元管理** |
| **インターフェイス構成** | 共通 IP/MAC (論理) | Spanned (L2) または Individual (L3) |
| **対応機種** | Firepower 1000/2100/4100/9300, vFTD | Firepower 4100/9300, vFTD (限定的) |

---

## 🏗 動作原理

### High Availability (Failover)
2 台のデバイスは、専用のフェイルオーバーリンクを介してハローメッセージを交換し、互いの生存を確認します。また、ステートリンクを通じて TCP/UDP セッション、VPN、NAT などの状態をリアルタイムで同期します。

```text
[ FTD-1 (Active) ] <--- Failover/State Link ---> [ FTD-2 (Standby) ]
        |                                                |
   (Processes Traffic)                            (Monitors Health)
        |                                                |
   [ Interface Fail ] ---------------------------> [ Switch to Active ]
```

### Clustering
CCL を介して、どのユニットがどのフローを「オーナー (Owner)」として処理するか、あるいは「バックアップ (Backup/Director)」として情報を保持するかを動的に決定します。

---

## ⚙ 動作シーケンス

1.  **初期ネゴシエーション**: FMC から HA/クラスタ設定がプッシュされると、デバイス間で役割（Primary/Secondary, Master/Slave）を決定します。
2.  **構成同期**: 設定情報が Primary (Master) から他方のユニットへ配布されます。
3.  **状態同期**: 通信が開始されると、コネクション情報がバックアップユニットへ即座に送信されます。
4.  **障害検知**: 指定されたインターフェイスのダウンや、一定回数のハローパケット損失を検知します。
5.  **切り替え (Failover/Re-election)**: Standby が Active に昇格し、Gratuitous ARP を送信してトラフィックを自分に引き寄せます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **FMC による HA 構築**: `Devices > Device Management > Add > High Availability` からの設定手順が必須です。
*   **インターフェイス監視設定**: インターフェイスがダウンした際にフェイルオーバーをトリガーさせるための `Monitor` チェックボックスの設定。
*   **Failover IP の割り当て**: Active 用 IP と Standby 用 IP の双方を正しく設定する必要があります。
*   **仮想 MAC アドレスの設定**: `failover mac address` を設定することで、切り替え時のスイッチ側の学習時間を短縮する要件が出題されます。
*   **Clustering の CCL 設定**: CCL インターフェイスの MTU サイズ（カプセル化のため通常 1600 以上）の設定。
*   **不一致の解消**: VDB バージョンやソフトウェアバージョンが 2 台で一致していないと HA が組めないため、`System > Updates` での確認が先決です。

---

## 🛠 設定方法

### 1. FTD HA の作成 (FMC)
1.  FMC に 2 台の FTD が登録されていることを確認。
2.  `Devices > Device Management > Add > High Availability` を選択。
3.  名前を入力し、Device Type で `Firepower Threat Defense` を選択。
4.  Primary と Secondary のデバイスを指定。
5.  **Failover Link** と **State Link** に使用するインターフェイス（物理またはポートチャネル）を指定し、IP アドレスを入力。

### 2. 仮想 MAC アドレスの設定
`Devices > Device Management > Edit (HA Pair) > Interfaces` タブにて、各インターフェイスの `Advanced` 設定から仮想 MAC アドレスを指定します。

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI) |
| :--- | :--- |
| **冗長状態の全体確認** | <code>show failover</code> |
| **フェイルオーバー履歴の確認** | <code>show failover history</code> |
| **ステート同期の統計確認** | <code>show failover state</code> |
| **クラスタ情報の確認** | <code>show cluster info</code> |
| **クラスタ内の特定ユニットで実行** | <code>cluster exec [command]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| HA の構成に失敗する | ソフトウェア/VDB バージョンの不一致 | FMC から両デバイスを最新の同一バージョンへ更新。 |
| "Negotiation" から進まない | Failover リンクの物理ダウン、共有 Key 不一致 | リンクの疎通確認と設定パスワードの再入力。 |
| 意図しないフェイルオーバー | インターフェイス監視が敏感すぎる | フェイルオーバーの閾値（Threshold）を調整。 |
| ステート同期が遅れる | ステートリンクの帯域不足 | ステートリンクを EtherChannel 化し、10G 以上のリンクを推奨。 |

---

## ⚠ 制限事項

*   **ハードウェア要件**: 同一モデル、同一インターフェイス構成である必要があります。
*   **動作モード**: HA 構成時はマルチコンテキストモードが使用できません。
*   **ステート同期対象**: ステートフルインスペクション情報は同期されますが、一部のルーティングプロトコルや、特定の動的セッションには制限があります。

---

## 🔄 他技術との関連

*   **Routing**: HA 構成ではルーティングテーブルも同期されますが、ルーティングプロトコルのネイバー関係（OSPF/BGP）は切り替え時に再確立が必要な場合があります。
*   **NAT**: PAT（ポートアドレス変換）のエントリはステートフルに同期されるため、切り替え後も通信が維持されます。
*   **VPN**: Site-to-Site VPN および Remote Access VPN (AnyConnect) のセッション情報を同期可能です。

---

## 🧩 比較表

### FTD HA vs FTD Clustering

| 機能 | High Availability (HA) | Clustering |
| :--- | :--- | :--- |
| **主な目的** | **可用性 (Redundancy)** | **拡張性 (Scalability)** |
| **スループット** | 1 台分のみ | ノード数に応じて増加（N 台分） |
| **構成難易度** | 低い | 高い（対向スイッチの設定が必要） |
| **推奨環境** | 一般的な拠点、小中規模 DC | 大規模 DC、クラウドサービス |

---

## 💡 ベストプラクティス

1.  **物理的分離**: フェイルオーバー/ステートリンクは、データトラフィック用のリンクと物理的に分離したインターフェイスを使用します。
2.  **MAC アドレスの固定**: 仮想 MAC アドレス（Active/Standby 用）を手動設定することで、切り替え時のネットワークの混乱（CAM テーブルのフラッピング）を防止します。
3.  **時刻同期**: トラブルシューティングのログ解析を容易にするため、必ず NTP で全ユニットの時刻を同期させます。
4.  **MTU の調整**: クラスタリング CCL ではカプセル化が行われるため、MTU を 1600 以上に設定することを検討します。

---

## 📝 ラボ学習・設定サンプル例

1.  **FTD HA 基本構成**: FMC 経由で 2 台の vFTD を HA に設定。
2.  **インターフェイス監視の設定**: Inside/Outside インターフェイスを監視対象に含める。
3.  **フェイルオーバーキーの適用**: 制御通信の暗号化のためにパスワードを設定。
4.  **手動切り替えの実行**: CLI から `failover active` を実行し、ロールが入れ替わることを確認。
5.  **仮想 MAC アドレスの構成**: 特定の VLAN インターフェイスに仮想 MAC を割り当て。
6.  **ステートフルフェイルオーバーの確認**: アクティブな TCP セッションが切り替え後も切断されないことを確認。
7.  **インターフェイスダウン時の挙動**: 片方のリンクを切断し、`show failover history` で原因を確認。
8.  **VDB バージョン不一致のトラブルシュート**: 一方のデバイスのみ VDB を更新し、HA が組めない状況を再現。
9.  **Clustering の個別 IP 設定**: Individual モードで各ノードに固有の管理 IP を設定。
10. **クラスタ統計の取得**: `show cluster stats` でトラフィックが分散されているか確認。

---

## ❓ 想定試験問題

1.  **Design**: FTD の HA 構成において、ステートリンクに求められる帯域幅の設計指針を述べよ。
2.  **実装**: FMC を使用して FTD HA ペアを構築する際、2 台のデバイス間で一致していなければならない主要な 3 つの項目を挙げよ。
3.  **トラブルシュート**: フェイルオーバー後に外部への通信が数分間停止した。スイッチ側の CAM テーブル学習を早めるために FTD で設定すべき項目は何か？
4.  **コンフィグ読解**: `show failover` の出力で `Syncing - Config` 状態で止まっている場合、どのような不具合が考えられるか？
5.  **Design**: クラスタリング環境における Spanned インターフェイスと Individual インターフェイスの使い分けについて説明せよ。

---

## 🔗 参考リソース

*   [Cisco Secure Firewall Management Center Administration Guide, 7.1 - Device High Availability](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/device_high_availability.html)
*   [Cisco Firepower Threat Defense Clustering Configuration Guide](https://www.cisco.com/c/en/us/td/docs/security/firepower/640/configuration/guide/fpmc-config-guide-v64/ftd_clustering.html)
*   [Cisco Live BRKSEC-3020: Troubleshooting FTD High Availability](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: FTD HA は ASA のフェイルオーバー機能をベースにしていますが、FMC からの管理が前提となるため、GUI 操作と CLI による確認の双方に慣れる必要があります。
*   **注意点**: ラボ試験では、デバイスを HA に追加する前に、それぞれが独立して FMC に正常に登録（Health Check がグリーン）されていることを必ず確認してください。
