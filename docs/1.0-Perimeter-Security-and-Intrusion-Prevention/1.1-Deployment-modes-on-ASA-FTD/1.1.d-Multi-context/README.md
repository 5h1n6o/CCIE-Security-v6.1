---
layout: default
title: 1.1.d-Multi-context
nav_order: 4
parent: 1.1-Deployment-modes-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.1.d Multi-context

Cisco ASAおよびCisco Firepower Threat Defense (FTD) における**マルチコンテキストモード（Multiple Context Mode）**は、1台の物理的なデバイスを複数の論理的なファイアウォール（コンテキスト）に分割する技術です。各コンテキストは独自のセキュリティポリシー、インターフェイス、ルーティングテーブルを保持し、完全に独立したデバイスとして動作します。

---

## 📘 概要

*   **機能概要**: 物理リソース（CPU、メモリ、インターフェイス）を論理的に切り分け、仮想的な複数のファイアウォールインスタンスを作成します。
*   **利用目的**: サービスプロバイダーによるマルチテナント収容、企業内での部門間分離、テスト環境と本番環境の共存などに利用されます。
*   **ASA/FTDでの役割**:
    *   **Cisco ASA**: 1台の物理ユニットを最大数百のコンテキストに分割可能です（モデルによる）。
    *   **Cisco FTD**: FTD自体はASAのような「コンテキスト」機能を直接持ちませんが、Firepower 4100/9300シリーズなどのハイエンド機では**マルチインスタンス（Multi-instance）**機能により、Dockerコンテナベースの独立したFTDインスタンスを複数展開することで、実質的なマルチコンテキストを実現します。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | 管理プレーン、コントロールプレーン、データプレーンを論理的に分離する。 |
| **用途** | マルチテナント、部門別ポリシー管理、Active/Active冗長構成。 |
| **メリット** | コスト削減（ハードウェア集約）、管理権限の委譲、設定の簡素化。 |
| **デメリット** | **ASAではAnyConnect (RA-VPN) が非対応**。一部の動的ルーティングに制限がある。 |
| **分類方式** | MACアドレス、インターフェイス、または宛先IP（透過モード時）によるパケット分類。 |
| **管理エンティティ** | **System Execution Space**（全体管理）と **Admin Context**（管理用インスタンス）。 |
| **対応機種** | ASA 5500-X, ASAv, Firepower 4100/9300 (FTD Instance)。 |

---

## 🏗 動作原理

マルチコンテキスト環境は、3つの主要な実行スペースで構成されます。

1.  **System Execution Space**: 物理インターフェイスの割り当て、コンテキストの作成・削除、リソース制限の定義を行う領域です。ネットワークトラフィックの転送は行いません。
2.  **Admin Context**: 最初に作成される特別なコンテキストで、システムへの管理アクセスに使用されます。他のコンテキストの設定ファイルを読み書きする権限を持ちます。
3.  **Data Contexts**: 実際のトラフィック処理を行う論理ファイアウォールです。

```text
[ Physical ASA / FTD Hardware ]
   ↓
[ System Execution Space ] --- (Resource Management / Context Creation)
   ↓
   +--- [ Admin Context ]  --- (Management Access / Config Access)
   ↓
   +--- [ Context A ]      --- (Security Policy / NAT / Routing)
   ↓
   +--- [ Context B ]      --- (Security Policy / NAT / Routing)
```

---

## ⚙ 動作シーケンス

パケットが物理インターフェイスに着信した際、ASAはどのコンテキストに処理を渡すべきかを決定する**パケット分類（Classification）**を行います。

1.  **MACアドレスによる分類**: 共有インターフェイスを使用している場合、インターフェイスに割り当てられた一意の仮想MACアドレスに基づいてコンテキストを特定します（最も推奨される方法）。
2.  **インターフェイスによる分類**: 物理インターフェイスまたはサブインターフェイスが特定のコンテキストに排他的に割り当てられている場合、そのインターフェイスに基づいて特定します。
3.  **宛先IPによる分類**: 透過（Transparent）モードで共有インターフェイスを使用し、かつ一意のMACアドレスがない場合に限り、宛先IPアドレスを参照します。
4.  **処理の移行**: 分類されたパケットは、該当コンテキストのコネクションテーブルとポリシーに従って処理（ACL/NAT等）されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **Active/Active Failoverとの組み合わせ**: マルチコンテキストモードは、ASAで Active/Active フェイルオーバーを実現するための必須条件です。各コンテキストを別々のFailover Groupに配置する手法が問われます。
*   **リソース制限（Resource Limits）**: `class` を使用して、特定のコンテキストが接続数（conn）やスループットを独占しないよう制限する設定が重要です。

### ラボ試験で設定させられそうな内容
*   **モードの変更**: `mode multiple` への切り替えと、それに伴う再起動への対応。
*   **共有インターフェイス（Shared Interface）**: 同一の物理/サブインターフェイスを複数のコンテキストで共有し、それぞれに異なる仮想MACを割り当てる構成。
*   **管理コンテキストの変更**: デフォルトの `admin` コンテキストを別のコンテキストに変更する手順。

### よくある設定ミス
*   **AnyConnectの設定**: 「マルチコンテキストのASAでAnyConnectを構成せよ」という問題は、**ASAの制限により不可能（Unsupported）**であることを知っているかが試されます。この場合、シングルモードに戻すか、FTDインスタンスを使用する必要があります。
*   **Config-URLの指定ミス**: `config-url` で指定するパス（disk0:/contextA.cfg等）が誤っていると、コンテキストが正常に起動しません。

### showコマンドから状態を判断
*   `show mode`: マルチモードかシングルモードかを確認。
*   `show context`: 作成済みのコンテキスト一覧と、割り当てられたインターフェイスを確認。
*   `show resource usage context [name]`: 特定のコンテキストのリソース消費量を確認。

---

## 🛠 設定方法

### ASA (CLI) - 基本構成手順

1.  **マルチコンテキストモードへの変更**（要再起動）:
    ```bash
    ASA(config)# mode multiple
    # 再起動後、running-configはシステムコンフィグになります
    ```
2.  **リソースクラスの定義**:
    ```bash
    ASA(config)# class GOLD-CUSTOMER
    ASA(config-class)# limit-resource all 20
    ASA(config-class)# limit-resource conn 1000
    ```
3.  **コンテキストの作成と割り当て**:
    ```bash
    ASA(config)# context CTX-SALES
    ASA(config-ctx)# allocate-interface GigabitEthernet0/1 sales-inside
    ASA(config-ctx)# allocate-interface GigabitEthernet0/0 sales-outside
    ASA(config-ctx)# config-url disk0:/ctx-sales.cfg
    ASA(config-ctx)# class GOLD-CUSTOMER
    ```
4.  **コンテキストへの切り替え**:
    ```bash
    ASA(config)# changeto context CTX-SALES
    CTX-SALES# # ここからは通常のファイアウォール設定が可能
    ```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **コンテキスト一覧表示** | <code>show context</code> |
| **リソース使用状況の確認** | <code>show resource usage all</code> |
| **現在の実行スペース確認** | <code>show curcontext</code> |
| **インターフェイス割り当て確認** | <code>show interface alias</code> |
| **コンテキスト設定のダンプ** | <code>show running-config context</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| パケットが正しいコンテキストに届かない | パケット分類（Classification）の失敗 | 共有インターフェイスで <code>mac-address auto</code> が有効か確認。 |
| コンテキスト作成時にエラーが出る | メモリまたはライセンスの不足 | <code>show cpu usage</code> / <code>show memory</code> を確認。 |
| 特定コンテキストでVPNが動作しない | RA-VPN (AnyConnect) 設定の試行 | ASAではRA-VPN非対応。Site-to-Site VPNのみ許可されているか確認。 |
| インターフェイスが表示されない | 他のコンテキストが独占している | <code>show context detail</code> でインターフェイスの重複割り当てがないか確認。 |
| 管理アクセス（SSH）が拒否される | Adminコンテキスト以外への直接接続 | <code>management-access [ifname]</code> の設定を確認。 |

---

## ⚠ 制限事項

*   **RA-VPN (AnyConnect) 非対応**: ASAマルチコンテキストモード最大の制限事項です。モバイルVPNが必要な場合はシングルモードを使用します。
*   **動的ルーティングの制限**: 古いOSバージョンではOSPFv3やBGPに制限がありましたが、最新（9.x以降）では多くが解消されています。ただし、マルチキャストルーティングは依然として一部制限があります。
*   **Shared Interfaceの脆弱性**: 共有インターフェイスでMACアドレスが同一だと分類不能になるため、手動または自動で一意のMACを割り当てる必要があります。

---

## 🔄 他技術との関連

*   **High Availability (Failover)**: **Active/Active Failover** をサポートする唯一のモードです。2つのフェイルオーバーグループを作成し、コンテキストを分散させることで負荷分散を物理ユニット間で行います。
*   **SNMP/Syslog**: システム全体で1つのSyslog/SNMPサーバを指定することも、各コンテキストで個別に設定することも可能です。
*   **FlexConfig (FTD)**: FTDのマルチインスタンス環境で、ASA固有のCLIベース機能を追加する場合に利用されます。

---

## 🧩 比較表

### Single vs Multi Context (ASA)

| 機能 | Single Mode | Multiple Mode |
| :--- | :--- | :--- |
| **AnyConnect (RA-VPN)** | サポート | **非サポート** |
| **Site-to-Site VPN** | サポート | サポート |
| **Failover方式** | Active/Standby | Active/Active または Active/Standby |
| **管理権限** | 単一（全体） | コンテキストごとの分離 |
| **用途** | 一般的なエッジFW | MSP、データセンター、テナント分離 |

---

## 💡 ベストプラクティス

*   **仮想MACの自動生成**: `mac-address auto` をグローバルで有効にし、各コンテキストのインターフェイスに一意のL2識別子を持たせます。
*   **リソース制限の明示**: `default` クラスのままにせず、各コンテキストに適切な `class` を割り当ててDoS耐性を高めます。
*   **Adminコンテキストの専用化**: Adminコンテキストにはデータトラフィックを通さず、管理通信（SSH/ASDM）専用に維持します。

---

## 📝 ラボ学習・設定サンプル例

### 1. マルチコンテキストモードへの切り替え
*   **要件**: ASAをマルチコンテキストモードに変更し、再起動後に確認せよ。
*   **設定**:
```bash
ASA(config)# mode multiple
# (Proceed with save and reboot)
ASA# show mode
Security context mode: multiple
```

### 2. リソース制限クラスの構成
*   **要件**: 「LIMITED」という名前のクラスを作成し、Telnet接続数を10に制限せよ。
*   **設定**:
```bash
class LIMITED
 limit-resource telnet 10
```

### 3. データコンテキストの新規作成
*   **要件**: コンテキスト「CUSTOMER-A」を作成し、設定ファイルをdisk0に保存せよ。
*   **設定**:
```bash
context CUSTOMER-A
 config-url disk0:/cust-a.cfg
```

### 4. インターフェイスの排他的割り当て
*   **要件**: 物理ポートGig0/2をコンテキスト「WEB-SERVER」に割り当て、コンテキスト内での名前を「outside」にせよ。
*   **設定**:
```bash
context WEB-SERVER
 allocate-interface GigabitEthernet0/2 outside
```

### 5. サブインターフェイスによるマルチコンテキスト
*   **要件**: Gig0/1.10をContext-A、Gig0/1.20をContext-Bに割り当てよ。
*   **設定**:
```bash
interface GigabitEthernet0/1.10
 vlan 10
interface GigabitEthernet0/1.20
 vlan 20
context Context-A
 allocate-interface GigabitEthernet0/1.10
context Context-B
 allocate-interface GigabitEthernet0/1.20
```

### 6. 仮想MACアドレスの自動生成
*   **要件**: 共有インターフェイス使用時のパケット分類を容易にするため、MACアドレスを自動生成せよ。
*   **設定**:
```bash
ASA(config)# mac-address auto
```

### 7. コンテキスト間移動と基本設定
*   **要件**: システム実行スペースから「CTX1」に移動し、内部インターフェイスにIPを設定せよ。
*   **設定**:
```bash
ASA# changeto context CTX1
CTX1# config t
CTX1(config)# interface inside
CTX1(config-if)# ip address 10.1.1.1 255.255.255.0
```

### 8. Adminコンテキストの変更
*   **要件**: 既存の「MGMT-CTX」を新しいAdminコンテキストとして指定せよ。
*   **設定**:
```bash
ASA(config)# admin-context MGMT-CTX
```

### 9. 共有インターフェイス上のSite-to-Site VPN
*   **要件**: 共通のOutsideインターフェイスを共有する2つのコンテキストで、それぞれ異なるピアとVPNを構築せよ。
*   **注意**: 各コンテキストは一意のパブリックIPまたはポートを持つ必要がある。

### 10. Active/Active Failoverへの割り当て
*   **要件**: Context-AをGroup 1（Primary上でActive）、Context-BをGroup 2（Secondary上でActive）に設定せよ。
*   **設定**:
```bash
failover group 1
 primary
failover group 2
 secondary
context Context-A
 join-failover-group 1
context Context-B
 join-failover-group 2
```

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `allocate-interface GigabitEthernet0/1 sales` と設定されたコンテキスト内において、`show interface ip brief` を実行した際に表示されるインターフェイス名は？
    *   **正解**: `sales`（システムで割り当てられたエイリアス名が表示される）。
2.  **トラブルシュート**: マルチコンテキストのASAで、Outsideから各コンテキストへのPingが不安定。`mac-address auto` を設定していない場合、何が起きているか？
    *   **正解**: 物理MACが全コンテキストで共有されており、ASAがパケットをどのコンテキストへ渡すべきか正しく分類（Classification）できていない。
3.  **Design**: 100の独立した顧客セグメントを1台の物理FWで収容しつつ、各顧客に自身のWebフィルタリング設定を管理させたい。ASAで最適な構成は？
    *   **正解**: Multiple Context Mode（各顧客にコンテキスト管理権限を委譲）。
4.  **実装**: システム実行スペース（System）で設定可能な項目はどれか？
    *   A. OSPFネイバーの定義
    *   B. 物理インターフェイスのコンテキストへの割り当て
    *   C. ユーザごとのNATルール
    *   **正解**: B。AとCは各コンテキスト内でのみ設定可能。
5.  **動作シーケンス**: 共有インターフェイス環境において、パケット着信時にASAが最初に参照するL2/L3情報は何か？
    *   **正解**: 宛先MACアドレス。これによってどの論理ファイアウォール（コンテキスト）にパケットをディスパッチするかを決定する。

---

## 🔗 参考リソース

*   **Cisco Live 動画/スライド**:
    *   [BRKSEC-3020: Troubleshooting Firewall Threat Defense (FTD)](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-3020)
    *   [BRKSEC-2021: Firepower Threat Defense - Packet Flow and Troubleshooting](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-2021)
*   **Configuration Guides**:
    *   [Cisco ASA Series General Operations CLI Configuration Guide, 9.4 - Security Contexts](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config/general/asa-94-general-config/ha-contexts.html)
    *   [Cisco Firepower Threat Defense Configuration Guide for FMC, 7.1 - Multi-Instance Capability](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/device_management_multi_instance.html)
*   **Command Reference**:
    *   [Cisco ASA Series Command Reference - mode multiple / changeto context](https://www.cisco.com/c/en/us/td/docs/security/asa/command-reference/m-p/cmdref2/m4.html)
*   **Technical Notes**:
    *   [ASA Multiple Context Mode Overview and Sample Configuration](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/116450-config-asa-00.html)
*   **Design Guide**:
    *   [Cisco SAFE Design Guide - Firewall Deployment Modes](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Aug2014/CVD-FirewallDeploymentGuide-AUG14.html)
*   **CVD (Cisco Validated Design)**:
    *   [Secure Campus Design Guide - Implementation of Virtualized Firewalls](https://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/Campus/campover.html)

---

## 📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

