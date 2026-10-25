---
layout: default
title: 1.1.e-Multi-instance
nav_order: 5
parent: 1.1-Deployment-modes-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.1.e Multi-instance

Cisco FTD (Firepower Threat Defense) における **Multi-instance（マルチインスタンス / コンテナインスタンス）** デプロイメントは、Firepower 4100 シリーズおよび 9300 シリーズの FXOS (Firepower eXtensible Operating System) 上において、Docker コンテナ技術を利用して 1 つの物理セキュリティモジュール / エンジン上に複数の独立した FTD インスタンス（Container Instances）を同時に稼働させる技術です。

従来の Cisco ASA における Multi-Context モードとは異なり、CPU コア、メモリ、および専用ディスク空間がハードウェアレベルで完全分離・割り当てされるため、1 つのインスタンスの障害や過負荷、リロード、OS アップグレードが他のインスタンスに影響を与えません。また、Remote Access VPN (Cisco AnyConnect) や動的ルーティング (OSPF, BGP) を含む、FTD の全機能が各コンテナインスタンスで利用可能です。

---

## 📘 概要

* **機能概要**:
  * **Native Instance vs Container Instance**: FXOS 上で全ハードウェアリソース（100%）を単一の FTD に割り当てる形態が **Native Instance**、リソースプロファイル（Resource Profile）に基づき CPU コア・メモリ・ディスクを分割して複数の FTD インスタンスを作成する形態が **Container Instance** です。
  * **ハードウェアリソース分離**: 各コンテナインスタンスには指定された CPU コア数、コア数に比例した RAM、および一律 **40 GB** の専用ディスク領域が割り当てられ、他インスタンスからのリソース干渉（DoS 状態）を防止します。
  * **完全独立運用**: 各コンテナインスタンスは独自の FTD OS バージョンを実行可能であり、個別リロード・個別ソフトウェアパッチ適用・個別 FMC 管理が可能です。
* **利用目的**:
  * データセンターや MSP（サービスプロバイダー）環境において、物理アプライアンスを統合しつつ、顧客・部門ごとの管理権限・障害ドメイン・リソースを安全に完全隔離する。
  * 開発・検証環境（Dev/Test）と本番環境（Prod）を同一ハードウェアシャーシ上に独立して構築・検証する。
* **どのような場面で利用するか**:
  * Firepower 4100（単一セキュリティエンジン）または Firepower 9300（最大 3 つのセキュリティモジュール）を保有し、マルチテナント化を行いたい場合。
  * ASA Multi-Context では不可能であった AnyConnect Remote Access VPN 終端や OSPF/BGP 動的ルーティングをマルチテナント構成で実現したい場合。

---

## 🔑 要点

| 項目 | Native Instance | Container Instance (Multi-instance) |
| :--- | :--- | :--- |
| **サポートプラットフォーム** | Firepower 1000/2100/3100/4100/9300 | **Firepower 4100 / 9300 シリーズのみ** |
| **基盤アーキテクチャ** | FXOS 上の直接実行 | FXOS 上の **Docker コンテナ実行環境** |
| **1 モジュール内の最大数** | 1 インスタンス（100% リソース占有） | **最大 3 〜 18 インスタンス**（モデル/モジュール依存） |
| **CPU コア割り当て** | 全 CPU コア固定 | **Resource Profile** により偶数コア指定（最小 6 コア） |
| **メモリ (RAM) 割り当て** | 全 RAM 固定 | CPU コア数に比例して動的自動計算 |
| **ディスク容量 (Disk)** | モジュール全体のディスク | **1 インスタンスあたり一律 40 GB** |
| **初回作成時の要件** | 不要 | **セキュリティモジュール/エンジンの Reinitialize（ディスク初期化）が必須** |
| **インターフェイス拡張** | 物理 / EtherChannel ポートのみ | Dedicated, Data-sharing, FXOS VLAN Subinterfaces |
| **MAC アドレス管理** | 焼付 MAC または手動 Virtual MAC | **FXOS MAC Pool (`mac-address auto`) による自動割り当て** |
| **FTD 機能制限** | なし（全機能サポート） | **ほぼ全機能サポート**（※Hardware Offload 等の一部機能のみ非対応） |
| **ライセンス消費** | 1 シャーシ/モジュールあたり 1 ライセンス | **1 シャーシ/モジュールあたり 1 ライセンス**（コンテナごとではない） |

---

## 🏗 動作原理

FTD Multi-instance では、物理シャーシの管理および L1/L2 物理ポート設定を司る **FXOS (Supervisor)** と、データプレーン処理・L3〜L7 セキュリティスキャンを実行する **FTD Container Instances (Docker)** が役割分担を行います。

```text
┌────────────────────────────────────────────────────────────────────────┐
│ Firepower 4100 / 9300 Hardware Chassis                                 │
│                                                                        │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ FXOS Supervisor (Chassis Manager / FXOS CLI)                       │ │
│ │  - Physical Interfaces & EtherChannel Management                   │ │
│ │  - FXOS VLAN Subinterfaces / Data-sharing Interfaces               │ │
│ │  - Hardware Packet Classifier & MAC Address Pool (User Prefix)     │ │
│ │  - Resource Profiles Allocation (CPU Core / RAM / 40GB Disk)       │ │
│ └────────────────────────────────────────────────────────────────────┘ │
│                                   │                                    │
│        ┌──────────────────────────┼──────────────────────────┐         │
│        ▼                          ▼                          ▼         │
│ ┌──────────────┐          ┌──────────────┐          ┌──────────────┐   │
│ │ FTD Cont. A  │          │ FTD Cont. B  │          │ FTD Cont. C  │   │
│ │ (6 Cores)    │          │ (8 Cores)    │          │ (10 Cores)   │   │
│ │ ├─ LINA      │          │ ├─ LINA      │          │ ├─ LINA      │   │
│ │ └─ Snort 3   │          │ └─ Snort 3   │          │ └─ Snort 3   │   │
│ │ (v7.0.5)     │          │ (v7.2.4)     │          │ (v7.3.0)     │   │
│ └──────────────┘          └──────────────┘          └──────────────┘   │
│        │                          │                          │         │
└────────┼──────────────────────────┼──────────────────────────┼─────────┘
         │                          │                          │
         ▼                          ▼                          ▼
 [ FMC Manager 1 ]          [ FMC Manager 1 ]          [ FMC Manager 1 ]
 (※1 つのモジュール内のすべてのコンテナは同一の FMC によって管理される必要あり)
```

### パケット分類（Hardware Packet Classifier）の仕組み
パケットが FXOS 物理ポートに入力された際、FXOS スイッチングハードウェア（Classifier）は以下の条件で受信コンテナインスタンスを特定・振り分けます。

1. **Unique Interfaces**: 物理ポートまたは FXOS VLAN サブインターフェイスが特定の 1 つのコンテナインスタンスにのみ割り当てられている場合、ポート識別のみでコンテナへパス。
2. **Unique MAC Addresses (Shared / Data-sharing)**: 複数のコンテナインスタンスが同一の `Data-sharing` ポートや FXOS インターフェイスを共有している場合、FXOS MAC Pool により自動生成された一意の **宛先 MAC アドレス** を照合してコンテナを特定。

---

## ⚙ 動作シーケンス

1. **セキュリティモジュールの Reinitialize（初期構成）**:
   * FXOS Chassis Manager または CLI から、対象のセキュリティモジュール/エンジンを **Reinitialize** します。これによりディスクパーティションが Docker コンテナ用に再フォーマットされます。
2. **Resource Profile の作成とバインド**:
   * FXOS 上で CPU コア数（例: 6 コア, 8 コア, 12 コア）を指定した **Resource Profile** を定義します。
   * コンテナインスタンス作成時にこのプロファイルをバインドすると、コア数に応じた RAM が自動計算され、40 GB のコンテナディスクが割り当てられます。
3. **FXOS インターフェイスおよび MAC Pool の割り当て**:
   * FXOS 上でポートタイプを `Data`（専用）または `Data-Sharing`（共有）に指定するか、`FXOS VLAN Subinterface` を作成します。
   * FXOS MAC Pool プレフィックス（例: Prefix `77` → `A24D.00xx.xxxx`）に基づき、インスタンス内の各ポートに一意の MAC アドレスが割り当てられます。
4. **ブートストラップ情報（Bootstrap）のインジェクション**:
   * 管理 IP アドレス、サブネットマスク、デフォルトゲートウェイ、FMC IP アドレス、および Registration Key / NAT ID を設定すると、FXOS スーパーバイザがコンテナ起動時にこれらをインジェクションします。
5. **FMC での登録とポリシー適用**:
   * 各コンテナインスタンスが起動し、管理ポート経由で FMC (`sftunnel` - TCP 8305) に接続します。
   * FMC 上で個別の FTD デバイスとして認識され、専用の Access Control Policy、NAT、VPN、ルーティングがデプロイされます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Reinitialize の失念によるコンテナ作成不可**:
   * 初めて Native インスタンスから Container インスタンスへ移行する際、FXOS 上でセキュリティモジュールを **Reinitialize** しないと、ディスクフォーマット不整合によりコンテナインスタンスを追加できません。
2. **同一モジュール内の全コンテナにおける FMC 一元管理要件**:
   * 1 つの物理セキュリティモジュール / エンジン上に存在するすべてのコンテナインスタンスは、**必ず同一の FMC アプライアンス（または HA ペア）によって管理** されなければなりません（インスタンスごとに異なる FMC に追加することはライセンスおよびアーキテクチャ上不可）。
3. **FXOS インターフェイスタイプと適用用途の違い**:
   * **`Data`**: 特定の 1 つのコンテナ専用ポート。他コンテナと共有不可。
   * **`Data-Sharing`**: 複数のコンテナインスタンスで共有可能。バックプレーン経由でコンテナ間通信が可能（※ただし Inline Set や Passive、Bridge Group ポートには使用不可）。
   * **`FXOS VLAN Subinterface`**: FXOS 側で VLAN タグ（802.1Q）を剥離・挿入して各コンテナに親ポートの特定 VLAN のみを渡す設定。最も推奨されるポート分割方式。
4. **Resource Profile 変更時のダウンタイム**:
   * 稼働中のコンテナインスタンスに対して Resource Profile（CPU コア数）を変更すると、**対象インスタンスが即座に再起動（約 5 分間のダウンタイム）** します。HA ペアの場合は、一度 HA を解除してから両方のプロファイルを変更し、再バインドする必要があります。
5. **TLS Crypto Acceleration (Hardware Crypto) の制限**:
   * FXOS 2.7.1+ / FTD 6.5+ 以降、1 つのセキュリティモジュールにつき **最大 16 個のコンテナインスタンス** まで SSL/TLS ハードウェアアクセラレーション (`Hardware Crypto`) を有効化できます。

---

## 🛠 設定方法

### 1. FXOS CLI: セキュリティモジュールの初期化 (Reinitialize) と Resource Profile 作成

```bash
! 1. FXOS CLI にログインし、セキュリティモジュール 1 を Reinitialize (ディスク再フォーマット)
Firepower-6k# scope module 1
Firepower-6k /module# reinitialize
WARNING: Reinitializing the security module will erase all user data and logical devices on this module.
Do you want to proceed? [y/N] y

! 2. Resource Profile の作成 (8 CPU コア割り当て)
Firepower-6k# scope platform
Firepower-6k /platform# create resource-profile Profile-8Core
Firepower-6k /platform/resource-profile*# set cpu-core-count 8
Firepower-6k /platform/resource-profile*# commit-buffer

! 3. MAC Address Pool プレフィックスの設定 (プレフィックス 77 -> MAC A24D.00xx.xxxx)
Firepower-6k /platform# scope auto-macpool
Firepower-6k /platform/auto-macpool# set prefix 77
Firepower-6k /platform/auto-macpool# commit-buffer
```

### 2. FXOS CLI: インターフェイスタイプ構成 (Data-Sharing & VLAN Subinterface)

```bash
! 1. 物理ポート Ethernet1/3 を Data-Sharing ポートに設定
Firepower-6k# scope eth-uplink
Firepower-6k /eth-uplink# scope interface Ethernet1/3
Firepower-6k /eth-uplink/interface# set port-type data-sharing
Firepower-6k /eth-uplink/interface# commit-buffer

! 2. Port-Channel10 上に FXOS VLAN サブインターフェイス (VLAN 100) を作成
Firepower-6k# scope eth-uplink
Firepower-6k /eth-uplink# create subinterface Port-Channel10 100
Firepower-6k /eth-uplink/subinterface*# set vlan 100
Firepower-6k /eth-uplink/subinterface*# set port-type data
Firepower-6k /eth-uplink/subinterface*# commit-buffer
```

### 3. FXOS Chassis Manager (GUI): Standalone FTD Container Instance の追加

1. **Logical Devices 画面へ移動**:
   * **Logical Devices > Add Device** を選択。
   * **Device Name**: `FTD-Container-01`, **Template**: `Cisco Secure Firewall Threat Defense`。
   * **Instance Type**: `Container`。
2. **Resource Profile & インターフェイス割り当て**:
   * **Resource Profile Selection**: `Profile-8Core` を指定。
   * **Data Ports**: 割当ポート（`Ethernet1/1`, `Port-Channel10.100` 等）を選択。
   * **Management Interface**: 専用管理ポートを選択（例: `Ethernet1/8`）。
3. **Bootstrap Configuration の入力**:
   * **Management IP / Netmask / Gateway**: `10.1.1.101/24`, GW: `10.1.1.1`。
   * **Firepower Management Center IP**: `10.1.1.50`。
   * **Registration Key / NAT ID**: `CiscoKey123`。
   * **Hardware Crypto**: `Enabled`。
   * **OK** をクリックして展開を開始。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **コンテナインスタンス概要・ステータス確認** | FXOS CLI | <code>show app-instance</code> |
| **コンテナリソース使用状況 (cgroups) 確認** | FXOS CLI | <code>show cgroups container</code> |
| **特定コンテナの動作詳細・エラーログ確認** | FXOS CLI | <code>show app-instance detail name \<NAME\></code> |
| **登録済み Resource Profile 一覧表示** | FXOS CLI | <code>show resource-profile</code> |
| **自動割り当て MAC アドレス一覧表示** | FXOS CLI | <code>show mac-address</code> |
| **FXOS サブインターフェイスステータス確認** | FXOS CLI | <code>show subinterface</code> |
| **FXOS から FTD コンテナコンソールへの直接接続** | FXOS CLI | <code>connect ftd \<INSTANCE_NAME\></code> |
| **コンテナ内からの FTD CLI 終了・抜け出し** | FTD CLI | <code>exit</code> (Telnet 脱出: <code>~</code> や <code>Ctrl-], .</code>) |
| **TLS ハードウェアアクセラレーション分配確認** | FTD CLI | <code>show hw-crypto</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **「Reinitialize required」エラーでコンテナが作成できない** | ディスクが Native 用パーティションのまま | FXOS: <code>show module 1 detail</code> | <code>scope module 1</code> に移動し <code>reinitialize</code> を実行してディスクをフォーマットする。 |
| **コンテナインスタンスの Oper State が "Down" や "Init" のまま** | リソース不足（CPU/RAM不足）または Bootstrap 設定誤り | FXOS: <code>show app-instance detail</code> | 対象シャーシの利用可能 CPU コア数を確認し、Resource Profile の割り当てを調整する。 |
| **FMC にコンテナインスタンスが登録されない** | sftunnel (TCP 8305) の通信遮断、または Key 不一致 | FTD CLI: <code>show managers</code><br>FXOS: <code>connect ftd \<NAME\></code> | FTD コンテナにコンソール接続し、`configure manager add` コマンドで FMC IP と Registration Key を再設定する。 |
| **Data-Sharing ポートで特定コンテナに通信が届かない** | MAC アドレスプール重複、またはスイッチ側の MAC 学習問題 | FXOS: <code>show mac-address</code> | FXOS の <code>auto-macpool</code> プレフィックスがユニークであることを確認し、手動 Virtual MAC 設定と競合していないか検証する。 |
| **Resource Profile 変更後に FTD の設定が不整合を起こす** | FMC 側のインベントリ未更新 | FMC: `Devices > Device Management > Inventory` | Profile 変更後、FMC の Device 画面で **System > Inventory** を開いて再同期（Update Inventory）を実行する。 |

---

## ⚠ 制限事項

* **非サポートプラットフォーム**: Firepower 1000, 2100, 3100 シリーズ、ASAv、および ASA OS では Multi-instance は一切使用できません（Firepower 4100 / 9300 専用）。
* **非サポート機能 (FTD Container Instance 内)**:
  * Hardware Flow Offload (NPU パススルーの一部)。
  * Radware DefensePro Link Decorator モジュール。
  * FMC UCAPL / CC セキュリティ認証モード。
* **Intra-Module HA の非サポート**: 同一セキュリティモジュール内の 2 つのコンテナ間で HA (Active/Standby) ペアを組むことは不可（HA は必ず別々の物理シャーシ上のコンテナ間で行う必要あり）。

---

## 🔄 他技術との関連

* **1.3 High Availability & Clustering**:
  * **Inter-Chassis HA**: 2 台の別シャーシ上のコンテナインスタンス間で Active/Standby HA を構成可能。
  * **Multi-Instance Clustering**: 各コンテナをノードとし、別シャーシ/モジュール間に跨がる最大 6 ノードの FTD クラスタを形成可能（※ Port-Channel 48 の FXOS CCL サブインターフェイスを使用）。
* **4.0 Secure Connectivity (VPN)**:
  * ASA Multi-Context とは異なり、各 FTD コンテナインスタンス内で **Cisco AnyConnect Remote Access VPN** および **Site-to-Site IPsec VPN** が完全に動作します。

---

## 🧩 比較表

### 1. ASA Multi-Context vs FTD Multi-instance (Container)

| 比較項目 | Cisco ASA Multi-Context Mode | Cisco FTD Multi-instance (Container) |
| :--- | :--- | :--- |
| **隔離レイヤ** | ソフトウェア論理分割（同一 LINA プロセス共有） | **Docker コンテナ分離（ハードウェアリソース分離）** |
| **CPU / RAM 制限** | 共有（Quota によるソフト制限のみ） | **物理 CPU コア数および RAM の硬直割り当て** |
| **OS バージョン** | 全 Context で同一 ASA バージョン | **コンテナごとに異なる FTD バージョンを実行可能** |
| **障害波及性** | 1 プロセスのクラッシュが全 Context に影響し得る | **1 コンテナの障害・再起動は他コンテナに影響なし** |
| **Remote Access VPN** | **非サポート** | **フルサポート (Cisco AnyConnect / Secure Client)** |
| **動的ルーティング** | **非サポート** (Static ルートのみ) | **フルサポート (OSPFv2/v3, BGP, EIGRP, RIP)** |
| **管理モデル** | CLI / ASDM / FMC (一部) | **FMC (Firepower Management Center) 一元管理専用** |

### 2. Native Instance vs Container Instance (FTD on FXOS)

| 比較項目 | Native Instance | Container Instance (Multi-instance) |
| :--- | :--- | :--- |
| **1 モジュールあたりの制限** | 1 インスタンスのみ | 複数インスタンス (モジュールに応じ最大 3〜18) |
| **ディスク領域** | モジュール全領域 | インスタンスあたり固定 **40 GB** |
| **管理インターフェース** | FMC または FDM (Local Management) | **FMC 専用** (FDM 不可) |
| **FXOS サブインターフェイス** | 非サポート（FTD アプリ内で定義） | **サポート**（FXOS 側で定義・割当可能） |
| **Shared Interface** | 非サポート | **`Data-Sharing` ポートタイプとしてサポート** |

---

## 💡 ベストプラクティス

1. **FXOS VLAN Subinterface の優先利用**:
   * コンテナ間で物理ポートを共有する場合は、`Data-Sharing`（L2 MAC 識別）ではなく、**`FXOS VLAN Subinterface`** を使用して VLAN ID ごとにポートを論理分割して割り当てます。これにより、パケット分類処理のオーバーヘッドが軽減しスループットが向上します。
2. **モジュール初期化（Reinitialize）の計画的設定**:
   * 本番運用開始後に Native から Multi-instance へ変更する場合、モジュール内の全データが消去されるため、構築初期段階で Reinitialize を完了させます。
3. **`Permit Expert mode` の無効化**:
   * セキュリティおよびテナント非分離リスクを抑えるため、コンテナ作成時の Bootstrap 設定で `Permit Expert mode from FTD SSH sessions` を **`No`** に設定し、FXOS スーパバイザ経由でのみ Expert Shell へ入れるように制限します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: FXOS セキュリティモジュールの Reinitialize と Resource Profile 定義

#### 問題
Firepower 4100 のセキュリティエンジン 1 をコンテナインスタンス用に初期化 (Reinitialize) し、CPU コア 12 個を割り当てる Resource Profile `RP-12Core` を作成せよ。

#### 要件
* FXOS CLI から操作を行うこと。

#### 設定例

```bash
! 1. セキュリティエンジンの Reinitialize
scope security-engine 1
reinitialize
! Confirm で 'y' を入力

! 2. Resource Profile の作成
scope platform
create resource-profile RP-12Core
set cpu-core-count 12
commit-buffer
```

#### 検証方法
* `show resource-profile RP-12Core` を実行し、`CPU Cores: 12` であることを確認。

---

### 例2: FXOS における MAC Pool プレフィックス定義と自動生成アドレス検証

#### 問題
FXOS 上で、コンテナインスタンス用の自動 MAC 生成プレフィックスを `1009` に設定せよ。

#### 要件
* アドレスフォーマットが `A2F1.03xx.xxxx` になることを確認すること。

#### 設定例

```bash
scope platform
scope auto-macpool
set prefix 1009
commit-buffer
```

#### 検証方法
* `show mac-address` を実行し、生成される MAC アドレスの先頭ブロックが `A2F1.03` になっていることを検証。

---

### 例3: FXOS VLAN サブインターフェイスの作成とコンテナ割り当て準備

#### 問題
FXOS 上で `Port-Channel1` インターフェイス配下に VLAN 200 用のサブインターフェイス `Port-Channel1.200` を追加作成せよ。

#### 要件
* ポートタイプは `Data` とすること。

#### 設定例

```bash
scope eth-uplink
create subinterface Port-Channel1 200
set vlan 200
set port-type data
commit-buffer
```

#### 検証方法
* `show subinterface` を実行し、`Port-Channel1.200` が `VLAN 200` で作成されていることを確認。

---

### 例4: FTD Container Instance の基本デプロイ (Bootstrap 設定)

#### 問題
FXOS Chassis Manager を使用して、コンテナインスタンス `FTD-TENANT-A` を作成せよ。

#### 要件
* Profile: `RP-12Core`
* Mgmt IP: `10.1.1.105/24`, GW: `10.1.1.1`
* FMC IP: `10.1.1.50`, RegKey: `Cisco123Key`

#### 設定例

```text
1. FXOS GUI: Logical Devices > Add Device.
2. Device Name: FTD-TENANT-A, Template: Cisco Secure Firewall Threat Defense, Instance Type: Container.
3. Resource Profile: RP-12Core.
4. Management Interface: Ethernet1/8 を指定.
5. Bootstrap Settings:
   - IPv4 Address: 10.1.1.105, Mask: 255.255.255.0, Gateway: 10.1.1.1
   - Management Center IP: 10.1.1.50, Registration Key: Cisco123Key
6. OK をクリックして Deploy.
```

#### 検証方法
* FXOS CLI で `show app-instance` を実行し、`FTD-TENANT-A` の `Oper State` が `Online` になることを確認。

---

### 例5: FTD Container Instance に対する TLS Crypto Acceleration の有効化

#### 問題
`FTD-TENANT-A` において、ハードウェア TLS 暗号化アクセラレーション (`Hardware Crypto`) を有効化せよ。

#### 要件
* FXOS CLI から設定すること。

#### 設定例

```bash
scope app-instance ftd FTD-TENANT-A
enter hw-crypto
set admin-state enabled
commit-buffer
```

#### 検証方法
* コンテナコンソールに接続 (`connect ftd FTD-TENANT-A`) し、`show hw-crypto` を実行して有効状態を確認。

---

### 例6: FMC 側での FTD Container Instance 追加手続き

#### 問題
FMC (`10.1.1.50`) GUI から、新規稼働したコンテナインスタンス `FTD-TENANT-A` (IP: `10.1.1.105`) をデバイス登録せよ。

#### 要件
* RegKey: `Cisco123Key`, Access Control Policy: `ACP-TENANT-A`

#### 設定例

```text
1. FMC: Devices > Device Management > Add > Add Device.
2. Host: 10.1.1.105, Display Name: FTD-TENANT-A
3. Registration Key: Cisco123Key
4. Access Control Policy: ACP-TENANT-A を選択
5. Register をクリックして同期完了を待つ.
```

#### 検証方法
* FMC の Device Management 画面で `FTD-TENANT-A` がグリーンアイコン（Normal）で追加されたことを確認。

---

### 例7: FTD Container Instance 上での AnyConnect Remote Access VPN 設定

#### 問題
FMC を使用して、コンテナインスタンス `FTD-TENANT-A` の `outside` インターフェイス上で AnyConnect Remote Access VPN を構成せよ。

#### 要件
* クライアントプール: `172.16.100.1-172.16.100.50`

#### 設定例

```text
1. FMC: Objects > Object Management > Address Pools > Add Address Pool:
   - Name: Pool-RA-VPN, Range: 172.16.100.1-172.16.100.50
2. FMC: Devices > VPN > Remote Access > Add:
   - Name: RA-VPN-ContA, Device: FTD-TENANT-A
   - Connection Profile: AAA: Local User, Address Pool: Pool-RA-VPN
   - Interfaces: outside
3. Save & Deploy を実行.
```

#### 検証方法
* AnyConnect クライアントから接続し、FTD コンテナ CLI で `show vpn-sessiondb remote` を確認。

---

### 例8: FTD Container Instance 間での Inter-Chassis Active/Standby HA 構成

#### 問題
Chassis 1 上の `FTD-HA-1` と Chassis 2 上の `FTD-HA-2`（同一 Resource Profile）を FMC 上で Active/Standby HA ペアとして構成せよ。

#### 要件
* Failover リンクポート: `Port-Channel20.500`

#### 設定例

```text
1. FMC: Devices > Device Management > Add > Add High Availability.
2. Name: FTD-HA-PAIR, Device Type: Threat Defense
3. Primary Device: FTD-HA-1, Secondary Device: FTD-HA-2
4. Failover & State Interfaces: Port-Channel20.500 を指定
5. Continue & Save -> Deploy 実行.
```

#### 検証方法
* FMC および FTD CLI で `show failover` を実行し、Primary が `Active`、Secondary が `Standby Ready` であることを確認。

---

### 例9: Multi-Instance Inter-Chassis Clustering の Cluster Control Link サブインターフェイス構成

#### 問題
FXOS 上で、マルチインスタンスクラスタ制御リンク（CCL）用として `Port-Channel48` 配下に VLAN 900 サブインターフェイスを作成し、ポートタイプを `cluster` に設定せよ。

#### 要件
* FXOS CLI で設定すること。

#### 設定例

```bash
scope eth-uplink
create subinterface Port-Channel48 900
set vlan 900
set port-type cluster
commit-buffer
```

#### 検証方法
* `show subinterface` を実行し、`Port-Channel48.900` の Type が `Cluster` となっていることを確認。

---

### 例10: Resource Profile の動的変更と FMC インベントリ更新手順

#### 問題
稼働中の `FTD-TENANT-A` の CPU コア数を 8 コアから 12 コア（`RP-12Core`）に変更し、FMC のインベントリ情報を再同期せよ。

#### 要件
* インスタンスの再起動が伴う点に留意すること。

#### 設定例

```bash
! 1. FXOS CLI で Profile を更新 (対象コンテナが再起動します)
scope app-instance ftd FTD-TENANT-A
set resource-profile-name RP-12Core
commit-buffer

! 2. コンテナ起動完了後、FMC GUI にて再同期
! FMC: Devices > Device Management > FTD-TENANT-A の編集
! System > Inventory > Update Inventory をクリック
```

#### 検証方法
* FMC の Device 詳細画面で、割り当て CPU コア数が 12 に更新されていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: FXOS CLI にて `show app-instance` を実行したところ、以下の出力が得られた。この状態に関する正しい説明はどれか。

```text
Firepower-4k# show app-instance
App Name     Slot ID   Admin State   Oper State   Running Version   Deployment Type
----------   -------   -----------   ----------   ---------------   ---------------
ftd1         1         Enabled       Online       7.2.4.15          Container
ftd2         1         Enabled       Init         7.3.0.1676        Container
```

* A) `ftd1` と `ftd2` は同一モジュール上で動作しており、`ftd2` は初期起動（またはディスク再フォーマット中）状態である。
* B) `ftd1` は Native インスタンスであり、モジュール全体を占有している。
* C) `ftd2` は FDM で管理されているため `Init` と表示されている。
* D) `ftd1` と `ftd2` は自動的に Active/Standby HA ペアを形成している。

**正解**: **A**
**解説**: `Slot ID 1` の同一モジュール上で 2 つの `Container` インスタンスが動作しており、`ftd1` は正常稼働（`Online`）、`ftd2` は初期化・プロビジョニング中（`Init`）です。

---

### 2. トラブルシュート問題

**問題**: Firepower 4115 上の FXOS で初めてコンテナインスタンスを作成しようと `Logical Devices > Add Device` を実行したが、`Instance Type: Container` を選択するプロンプトでエラーが表示されて追加できない。原因として最も適切なものはどれか。

* A) FXOS のバージョンが 6.0 未満であるため。
* B) セキュリティエンジンに対して事前に `Reinitialize`（ディスクフォーマット）が実行されていないため。
* C) FMC に Threat ライセンスが登録されていないため。
* D) データポートがすべて `Data-Sharing` に設定されているため。

**正解**: **B**
**解説**: 物理モジュールを Native モードから Container モードへ移行・初回導入する際は、必ず事前に FXOS 上でセキュリティモジュールを **Reinitialize** し、ディスク構造をコンテナ用に初期化する必要があります。

---

### 3. Design 問題

**問題**: 大規模マルチテナント環境において、Firepower 9300 の 1 つのセキュリティモジュール上に 4 つの FTD コンテナインスタンスを作成した。ライセンス設計に関する説明として正しいものはどれか。

* A) FMC 上で 4 個分の FTD Base ライセンスおよび 4 個分の Threat ライセンスが消費される。
* B) 1 つの物理セキュリティモジュール上の全コンテナインスタンスに対して、消費されるライセンスは **1 個分**（1 モジュール単位）である。
* C) コンテナインスタンスでは Smart Licensing が非サポートとなるため、Evaluation モードのみで動作する。
* D) 各コンテナごとに異なる Smart Account からライセンスを割り当てる必要がある。

**正解**: **B**
**解説**: FTD Multi-instance の大きなメリットとして、ライセンスは物理セキュリティモジュール / エンジン単位で消費されます。1 モジュール内に複数のコンテナを作成しても、消費される Base / Feature ライセンス数は **1** です。

---

### 4. 実装問題

**問題**: FTD コンテナインスタンス間で、物理ポート `Ethernet1/5` を論理的に分割して共有させたい。パケット処理パフォーマンスが最も高く、Cisco が推奨する FXOS 設定構成はどれか。

* A) `Ethernet1/5` のポートタイプを `Data-Sharing` に設定し、各 FTD コンテナのアプリケーション内で VLAN サブインターフェイスを作成する。
* B) FXOS 側で `Ethernet1/5` 配下に `FXOS VLAN Subinterface` を作成し、ポートタイプを `Data` に設定して各コンテナに個別割り当てする。
* C) `Ethernet1/5` のポートタイプを `Cluster` に変更する。
* D) FMC 上で Pre-filter Policy を作成してポート共有をシミュレートする。

**正解**: **B**
**解説**: コンテナ間でのポート共有において、`Data-Sharing`（L2 MAC 照合）よりも **`FXOS VLAN Subinterface`**（ハードウェア VLAN タグ分離）を使用する構成（Scenario 1）が、最もパケット分類パフォーマンスが高く推奨される構成です。

---

### 5. コンフィグ読解問題

**問題**: Cisco ASA Multi-Context モードと比較した際の、FTD Multi-instance (Container) の機能差に関する記述として**誤っているもの**はどれか。

* A) FTD Multi-instance では、各コンテナインスタンスで AnyConnect Remote Access VPN がサポートされる。
* B) ASA Multi-Context では、各 Context で OSPF や BGP などの動的ルーティングがサポートされる。
* C) FTD Multi-instance では、各コンテナにハードウェアレベルで CPU コアと RAM が固定割り当てされる。
* D) FTD Multi-instance では、コンテナごとに異なる FTD OS バージョンを実行できる。

**正解**: **B**
**解説**: ASA Multi-Context モードでは動的ルーティング（OSPF/BGP）や AnyConnect Remote Access VPN は**非サポート**です（FTD Multi-instance では双方とも完全サポートされます）。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Threat Defense Configuration Guide (FMC) - Multi-Instance Capability](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/multi_instance_capability_for_firepower_threat_defense.html)
* [Cisco Firepower 4100/9300 FXOS Firewall Chassis Manager Configuration Guide](https://www.cisco.com/c/en/us/td/docs/security/firepower/fxos/fxos281/config-guide/b_FXOS_CLI_ConfigGuide_281.html)
* [Cisco Live: Deep Dive into Firepower Multi-Instance Architecture (BRKSEC-3020)](https://www.ciscolive.com/)
* [CVD: Secure Firewall Deployment in Data Center Networks](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/cisco-secure-firewall-design-guide.html)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * FTD Multi-instance のキモは **「FXOS（L1/L2・コンテナ基盤）と FMC（L3-L7・FTD ポリシー）の役割分離」** です。
  * 「Reinitialize による初回初期化」「Resource Profile（偶数 CPU コア指定）」「1 モジュール内の全コンテナは同一 FMC 管理」「1 モジュール 1 ライセンス消費」という 4 大原則を確実に押さえておきましょう。
* **図解（リソース割り当て構造）**:
  ```text
  [ SM-44 Module: 86 Cores Available ]
   ├── Container 1 (Profile: 12 Cores) -> RAM Allocated -> 40GB Disk
   ├── Container 2 (Profile: 24 Cores) -> RAM Allocated -> 40GB Disk
   ├── Container 3 (Profile: 16 Cores) -> RAM Allocated -> 40GB Disk
   └── Remaining Cores: 34 Cores Available for new containers
  ```
* **注意点**:
  * 実機ラボ試験中に Resource Profile の変更を行うとコンテナの再起動（約 5 分間）が発生するため、構築の初期段階で正しいプロファイルを適用してください。
