---
layout: default
title: 1.1.c-Single
nav_order: 3
parent: 1.1-Deployment-modes-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.1.c Single

Cisco ASAおよびCisco Firepower Threat Defense (FTD) における**シングルモード（Single Mode）**、またはシングルコンテキストモードは、デバイスを単一の論理的なエンティティとして運用する最も標準的な展開方式です。このメモでは、CCIE Security v6.1ラボ試験において重要となる、マルチコンテキストモードとの比較、機能的な利点、および実装上の注意点について詳述します。

---

## 📘 概要

*   **機能概要**: デバイス全体で1つの設定ファイル（running-config）のみを保持し、全てのインターフェイス、ポリシー、ルーティング、管理プレーンが単一のインスタンス内で動作します。
*   **利用目的**: 複数の独立した組織（テナント）を物理的に分離する必要がない環境で利用されます。
*   **ASA/FTDでの役割**:
    *   **ASA**: デフォルトの動作モードです。**リモートアクセスVPN (AnyConnect)** などの高度な機能をフルサポートする唯一のモードです。
    *   **FTD**: 現行のFTDソフトウェアアーキテクチャでは、デバイスは基本的にシングルエンティティとして管理されます（ハイエンドモデルでの「マルチインスタンス」は物理的なコンテナ分離に近い概念です）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | 単一の設定ファイル、共有されるシステムリソース（CPU/メモリ）。 |
| **用途** | 一般的な企業ネットワーク、インターネット境界、RA-VPN終端。 |
| **メリット** | **RA-VPN (AnyConnect) のフルサポート**。OSPFv3、マルチキャスト等の高度なルーティング。 |
| **デメリット** | テナント間での管理権限の完全な分離が困難。1つの設定ミスがデバイス全体に影響。 |
| **対応機種** | 全てのCisco ASAおよびFTDモデル。 |
| **制限事項** | 大規模な仮想化（マルチテナント）が必要なサービスプロバイダー環境には不向き。 |
| **設計上の注意点** | ASAでマルチコンテキストからシングルに戻す場合、**設定が初期化される**ため注意。 |

---

## 🏗 動作原理

シングルモードでは、管理アクセスからデータプレーンの処理までが1つの階層で完結します。

```text
Management (SSH/ASDM/FMC)
   ↓
[ Single Management Plane ]
   ↓
[ Single Control Plane (RIB/ARP/VPN) ]
   ↓
[ Single Data Plane (NAT/ACL/Inspection) ]
   ↓
Physical/Logical Interfaces
```

デバイスは全てのトラフィックを一元的に監視し、1つのコネクションテーブル（Conn Table）ですべてのセッションを管理します。

---

## ⚙ 動作シーケンス

1.  **パケット受信**: 物理/VLANインターフェイスでパケットを受信。
2.  **グローバル・ルックアップ**: 共通のコネクションテーブルで既存セッションを確認。
3.  **セキュリティポリシー適用**: `running-config` に記述された唯一のACLセットとNATルールを順次適用。
4.  **インスペクション**: MPF (Modular Policy Framework) または Snort エンジンによる深いパケット検査。
5.  **ルーティング**: 単一のルーティングテーブル（RIB）に基づき出力先を決定。
6.  **パケット送出**: Egressインターフェイスから送出。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **AnyConnectの実装**: ASAにおいて、マルチコンテキストモードではRA-VPNがサポートされないため、**AnyConnectの要件が出た時点でシングルモードである必要があります**。
*   **モード切替の挙動**: `mode multiple` と `mode single` の切り替えコマンドと、それに伴う再起動、設定消失のリスクを把握しておくこと。

### ラボ試験で設定させられそうな内容
*   **ASAの初期化とシングルモード確認**: `show mode` で現在の動作状態を確認し、必要に応じてリセットする手順。
*   **FTDの管理初期化**: CLIからの管理パラメータ設定（IP, Gateway, Manager登録）。
*   **高度なルーティングプロトコルの設定**: OSPF, BGP, EIGRP (FTDはFlexConfig経由) のシングルインスタンス構成。

### よくある設定ミス
*   **マルチコンテキストでのRA-VPN設定試行**: 試験中に誤ってマルチモードに設定した状態でAnyConnectを構成しようとすると、コマンド自体が拒絶されます。
*   **FMC登録時のモード不一致**: FTDをFMCに追加する際、デバイス側の管理IP設定とFMC側の疎通確認が取れない（シングルモードでの基本的なL3到達性不足）。

### showコマンドから状態を判断する問題
*   `show running-config`: シングルモードでは全てのポリシーが1つの出力で表示されます。
*   `show resource usage`: デバイス全体のCPU/メモリ負荷を確認し、リソースが枯渇していないか判断。

---

## 🛠 設定方法

### ASA (CLI) - モードの確認と切り替え
ASAは工場出荷状態でシングルモードですが、ラボ環境で確認が必要です。
```bash
# モードの確認
ASA# show mode
Security context mode: single

# (必要に応じて) シングルモードへの変更
# ※注意: 再起動と設定クリアが伴います
ASA(config)# mode single
```

### FTD (FMC管理)
FTDはFMCに登録された時点で、単一の論理デバイスとして扱われます。
1.  **FTD CLI**: 管理インターフェイスの初期設定。
    ```bash
    > configure network ipv4 manual 192.168.1.10 255.255.255.0 192.168.1.1
    > configure manager add 192.168.1.100 <registration_key>
    ```
2.  **FMC GUI**: **Devices > Device Management** からデバイスを追加。この時、特別な「コンテキスト」の指定はなく、シングルデバイスとして登録されます。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **動作モードの確認** | <code>show mode</code> |
| **全コネクションの確認** | <code>show conn</code> |
| **インターフェイス状態** | <code>show interface ip brief</code> |
| **VPNセッションの確認** | <code>show vpn-sessiondb [anyconnect&#124;l2l]</code> |
| **FTD管理疎通確認** | <code>show managers</code> |
| **パケットトレース** | <code>packet-tracer input inside tcp <src_ip> <sp> <dst_ip> <dp></code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| AnyConnectコマンドが効かない | マルチコンテキストで動作している | <code>show mode</code> | <code>mode single</code>に変更（要再起動）。 |
| 特定の動的ルートが学習されない | ルーティングプロセスの不整合 | <code>show route</code> | ネットワーク設定とネイバー確立を確認。 |
| FTDがFMCに登録できない | 管理通信のルーティング欠如 | <code>show network</code> | <code>configure network static-routes</code>で追加。 |
| インターフェイス間の通信断 | セキュリティレベルによる拒否 | <code>show nameif</code> | <code>same-security-traffic permit</code>を検討。 |

---

## ⚠ 制限事項

*   **論理分離の欠如**: 物理的な1台のデバイスを、あたかも複数台の独立したファイアウォール（コンテキスト）であるかのように分割して、異なる管理者に委譲することはできません。
*   **リソース共有**: 特定のトラフィックフローが過剰にCPUを消費した場合、シングルモード内では他の全てのフローに影響が及びます（マルチモードのようなリソースクォータの設定不可）。
*   **構成の肥大化**: 大規模環境では、1つの `running-config` が数千行に及び、管理の複雑性が増します。

---

## 🔄 他技術との関連

*   **Access Control**: シングルモードでは、単一の「アクセスポリシー」または「アクセスコントロールリスト（ACL）」が全トラフィックに適用されます。
*   **VPN**: **リモートアクセスVPN** はシングルモード運用の主要な理由となります。
*   **High Availability (Failover)**: 2台のユニット間で、単一のステート情報を同期します。
*   **Routing**: 全てのインターフェイスが共有ルーティングテーブル（RIB）に参加します。

---

## 🧩 比較表

### Single Context vs Multiple Context (ASA)

| 機能 | Single Mode | Multiple Context Mode |
| :--- | :--- | :--- |
| **AnyConnect (RA-VPN)** | **サポート** | **非サポート** |
| **Dynamic Routing** | 全てサポート | OSPF, BGP等はバージョン/環境に依存 |
| **管理の分離** | なし (共有) | コンテキストごとに管理者/設定を分離 |
| **インターフェイス共有** | なし (直割当) | 複数コンテキストで共有可能 |
| **リソース管理** | 全体で共有 | 各コンテキストに上限値を割当 |

---

## 💡 ベストプラクティス

*   **AnyConnect要件の事前確認**: ラボ試験の初期段階で AnyConnect 設定が必要か確認し、必要であればシングルモードであることを即座に確認します。
*   **インターフェイス名の統一**: `inside`, `outside`, `dmz` などの標準的なネーミングコンベンションを使用し、設定ミスを防ぎます。
*   **Managementインターフェイスの活用**: データトラフィックとは別に、専用の管理インターフェイス（Management 0/0など）を使用して管理プレーンの安定性を確保します。

---

## 📝 ラボ学習・設定サンプル例

### 1. シングルモードの確認と初期設定 (ASA)
*   **問題**: ASA1がシングルモードで動作していることを確認し、Gig0/1をInside、Gig0/0をOutsideに設定せよ。
*   **設定例**:
```bash
show mode
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 203.0.113.1 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 192.168.1.1 255.255.255.0
 no shutdown
```

### 2. FTD 管理IPの初期設定 (CLI)
*   **要件**: 新しいFTDデバイスを管理IP 10.1.1.10、Gateway 10.1.1.1 として初期化せよ。
*   **設定例**:
```bash
> configure network ipv4 manual 10.1.1.10 255.255.255.0 10.1.1.1
```

### 3. ASAでの AnyConnect 終端設定
*   **問題**: シングルモードのASAで AnyConnect クライアント用のIPプール（172.16.10.1-100）を作成せよ。
*   **設定例**:
```bash
ip local pool VPN_POOL 172.16.10.1-172.16.10.100 mask 255.255.255.0
```

### 4. OSPF 動的ルーティングの構成 (ASA)
*   **問題**: シングルモードASAで OSPF 1 を構成し、Insideネットワークを Area 0 で広告せよ。
*   **設定例**:
```bash
router ospf 1
 network 192.168.1.0 255.255.255.0 area 0
 log-adj-changes
```

### 5. FTD Access Control Policy の基本設定 (FMC)
*   **要件**: InsideゾーンからOutsideゾーンへのWeb通信（HTTP/HTTPS）を許可するルールを作成せよ。
*   **手順**: FMC GUI > Policies > Access Control > Edit > Add Rule。

### 6. Identity NAT (NAT Exemption) の実装 (ASA)
*   **問題**: Inside (192.168.1.0/24) から特定の拠点 (10.10.10.0/24) への通信はNATしないようにせよ。
*   **設定例**:
```bash
object network obj-inside
 subnet 192.168.1.0 255.255.255.0
object network obj-remote
 subnet 10.10.10.0 255.255.255.0
nat (inside,outside) source static obj-inside obj-inside destination static obj-remote obj-remote
```

### 7. 管理アクセスの制限 (ASA)
*   **問題**: 192.168.1.100のホストからのみ、Inside経由でのSSHアクセスを許可せよ。
*   **設定例**:
```bash
ssh 192.168.1.100 255.255.255.255 inside
```

### 8. FTD FMC への登録プロセス (CLI)
*   **問題**: IP 192.168.1.100 の FMC に対し、キー "cisco123" を使用して FTD を登録せよ。
*   **設定例**:
```bash
> configure manager add 192.168.1.100 cisco123
```

### 9. Packet Tracer によるパケットフロー検証 (ASA)
*   **問題**: Inside (192.168.1.50) から 8.8.8.8 への DNS 通信 (UDP 53) が許可されているか検証せよ。
*   **検証コマンド**:
```bash
packet-tracer input inside udp 192.168.1.50 12345 8.8.8.8 53
```

### 10. ASA フェイルオーバーの基本構成 (Single Mode)
*   **問題**: Primaryユニットとして、Gig0/2 をフェイルオーバー用インターフェイスに設定せよ。
*   **設定例**:
```bash
failover lan unit primary
failover lan interface fov_int GigabitEthernet0/2
failover interface ip fov_int 172.16.1.1 255.255.255.252 standby 172.16.1.2
```

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: ASAに `mode multiple` が設定されていない状態で、`context admin` コマンドを入力した場合の挙動は？
    *   **正解**: シングルモードではコンテキストの概念がないため、エラーが返される。
2.  **トラブルシュート**: ASAでRA-VPNを設定したがクライアントが接続できない。`show mode` を実行したところ `multiple` と表示された。解決策は？
    *   **正解**: ASAのマルチコンテキストモードはRA-VPNをサポートしていない。設定をバックアップし、`mode single` に切り替えて再構築する必要がある。
3.  **Design**: 1台の物理ASAでAnyConnectをサポートしつつ、2つの部門で個別のルーティングテーブルを持ちたい。最適な構成は？
    *   **正解**: シングルモードで動作させ、VRF（Virtual Router）機能を使用して論理的に経路を分離する。
4.  **実装**: FTDデバイスを新規にセットアップする際、FMCとの通信を確立するためにCLIで最初に行うべき設定は？
    *   **正解**: 管理インターフェイスのIPアドレス設定（`configure network`）とマネージャの追加（`configure manager add`）。
5.  **制限事項**: シングルモードASAにおいて、CPU負荷の高いインスペクション（例: HTTP）が特定の通信に適用されている。これが他のVPNトラフィックに与える影響は？
    *   **正解**: シングルモードでは全てのリソースが共有されるため、特定の高負荷処理がデバイス全体のパフォーマンス低下を招き、VPNセッションの遅延や切断を引き起こす可能性がある。

---

## 🔗 参考リソース

*   **Cisco Live動画**:
    *   [BRKSEC-3020 - Troubleshooting Firewall Threat Defense (FTD)](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-3020)
*   **Cisco Liveスライド**:
    *   BRKSEC-2021: Firepower Threat Defense - Packet Flow and Troubleshooting
*   **Configuration Guide**:
    *   [Cisco ASA Series General Operations CLI Configuration Guide, 9.4 - Security Contexts](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config/general/asa-94-general-config/ha-contexts.html)
    *   [Cisco Firepower Threat Defense Administration Guide - FMC Management](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70.html)
*   **Command Reference**:
    *   Cisco ASA Command Reference - `mode single`
    *   Cisco FTD Command Reference - `configure network`
*   **Technical Notes**:
    *   [ASA Multiple Context Mode Overview and RA-VPN Support Limitations](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/116450-config-asa-00.html)
*   **Design Guide**:
    *   [Cisco SAFE Design Guide - Firewall Deployment Modes](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Aug2014/CVD-FirewallDeploymentGuide-AUG14.html)
*   **CVD (Cisco Validated Design)**
    *   [Firewall Deployment Guide - Single vs Multi Context](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Aug2014/CVD-FirewallDeploymentGuide-AUG14.html)

---

## 📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

