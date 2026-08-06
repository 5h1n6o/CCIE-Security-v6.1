---
layout: default
title: 1.8-Clustering-and-HA
nav_order: 8
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.8 Clustering and high availability features on Cisco ASA and Cisco FTD

Cisco ASAとCisco FTDにおけるクラスタリングおよびハイアベイラビリティ（HA）機能の比較表を作成しました。

### **Cisco ASA vs Cisco FTD：HA・クラスタリング比較表**

| 項目 | Cisco ASA | Cisco FTD (Firepower Threat Defense) |
| :--- | :--- | :--- |
| **HA (フェイルオーバー) モード** | **Active/Standby** および **Active/Active** | **Active/Standby のみ** |
| **Active/Active HA の要件** | **マルチコンテキストモード**の有効化が必要 | サポート対象外（FTDはコンテキストモードを持たない） |
| **状態同期 (Stateful)** | TCP/UDP接続状態、VPNセッションなどを同期 | ASAと同様に接続状態やVPN状態を同期 |
| **クラスタリング最大ユニット数** | 最大 **16 ユニット**（ASA 5585-Xなど） | 最大 **16 ユニット**（Firepower 4100/9300シリーズ） |
| **クラスタリング対応機種** | 5500-X (2台まで), 5585-X (16台), 9300/4100 | Firepower 4100/9300 シャーシ |
| **クラスタ接続モード** | Spanned EtherChannel, Individual Interface | Spanned EtherChannel, Individual Interface |
| **クラスタ内の役割** | Owner, Director, Forwarder | Owner, Director, Forwarder |
| **管理ツール** | CLI, ASDM, FMC | **FMC (推奨)**, FDM (小規模向け) |
| **ハードウェア要件** | 同一モデル、同一メモリ、同一インターフェイス構成 | 同一モデル、同一インターフェイス、ソフトウェア、ライセンス |
| **仮想化の類似機能** | セキュリティコンテキスト | **マルチインスタンス** (4100/9300のみ、コンテキストとは実装が異なる) |

### **主な相違点と特徴**

1.  **HAモードの柔軟性**:
    *   **ASA**はActive/Active構成をサポートしており、複数のコンテキスト（仮想FW）を利用して、各物理デバイスで異なるコンテキストをActiveにすることで負荷を分散できます。
    *   **FTD**はActive/Standbyのみをサポートしており、待機系デバイスは通常トラフィックを処理しません。

2.  **クラスタリングの目的**:
    *   両者ともクラスタリングにより、複数のデバイスを1つの論理ユニットとして統合し、スループットの向上（N+1の冗長性）を実現します。
    *   クラスタ内の各パケットは「Owner」となるユニットが処理し、非対称トラフィックの場合は「Cluster Control Link (CCL)」を介して転送されます。

3.  **管理と構成**:
    *   **FTD**のHAやクラスタ設定は、基本的にGUIベースの **FMC (Firepower Management Center)** から行います。4100/9300などの上位機種では、インターフェイス設定に **Chassis Manager (FXOS)** も使用します。
    *   **ASA**は、長年培われた **CLI** による直接設定や **ASDM** での管理が主流です。

4.  **仮想化とスケーラビリティ**:
    *   FTDにはASAの「コンテキストモード」はありませんが、Firepower 4100/9300シリーズでは **マルチインスタンス (Multi-instance)** 機能により、1つのハードウェア上で複数の独立したFTDインスタンスを実行し、それぞれをHAやクラスタ（インスタンス単位）に組み込むことが可能です。
