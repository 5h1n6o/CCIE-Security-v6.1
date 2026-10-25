---
layout: default
title: 1.1.d-Multi-context
nav_order: 4
parent: 1.1-Deployment-modes-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.1.d Multi-context

Cisco ASA (Adaptive Security Appliance) における **Multi-Context Mode（マルチコンテキストモード / セキュリティコンテキスト）** は、1 台の物理/仮想 ASA を複数の独立した論理ファイアウォール（仮想ファイアウォールインスタンス）に分割・仮想化する技術です。各コンテキスト（Context）は独自のアクセス制御ポリシー、インターフェイスマッピング、ルーティングテーブル、および管理者アカウントを保持し、あたかも独立した複数の物理ファイアウォールが存在するかのように動作します。

なお、**Cisco FTD (Firepower Threat Defense)** では ASA 形式の Multi-Context モードはサポートされておらず、代わりに Firepower 4100/9300 シリーズの FXOS ハードウェア上で **Multi-Instance (Container Instance)** 機能を提供することでマルチテナント化を実現します。

---

## 📘 概要

* **機能概要**:
  * **System Execution Space（システム実行スペース）**: アプライアンス全体で 1 つ存在し、コンテキストの定義、インターフェイスの割り当て、リソース制限（Resource Class）、および Failover/Cluster の物理構成を管理する空間です。
  * **Admin Context（管理コンテキスト）**: システム管理者がネットワーク経由（SSH/ASDM）でアクセスし、システム実行スペースや他すべてのコンテキストへアクセス・制御するための特殊なコンテキストです。
  * **User Contexts（ユーザーコンテキスト）**: 顧客や部門（テナント）ごとに割り当てられる仮想ファイアウォールで、独自の `nameif`、IP アドレス、ACL、NAT、スタティックルートを持ちます。
  * **FTD Container Instance (FXOS)**: FTD のマルチテナント技術。Docker コンテナをベースとし、CPU コア・メモリ・ディスクをハード分離した独立した FTD インスタンスを同一シャシー上に複数実行します。
* **利用目的**:
  * データセンターやサービスプロバイダー（MSP）環境において、物理機器を共有しつつテナント間（複数顧客・部門間）のセキュリティポリシーおよび管理権限を完全分離する。
  * 設置スペース、電力、冷却コスト、およびアプライアンス調達コストを削減（シャーシ集約）する。
* **どのような場面で利用するか**:
  * サービスプロバイダーのマルチテナント型 FW サービス。
  * 企業内の部門間（営業・開発・人事等）の論理セグメンテーションおよび自律管理。
  * Active/Active Failover による複数コンテキストの負荷分散運用。

---

## 🔑 要点

| 項目 | Cisco ASA Multi-Context Mode | Cisco FTD Multi-Instance (Container) |
| :--- | :--- | :--- |
| **アーキテクチャ** | LINA OS 内の論理ソフトウェア分割 (プロセス共有) | FXOS 上の Docker コンテナによる独立実行環境 |
| **管理ドメイン** | System Space ＋ Admin Context ＋ User Contexts | FXOS Chassis Manager (基盤) ＋ FMC (各 FTD インスタンス管理) |
| **リソース分離** | 共有 CPU/メモリ（Resource Class で接続数等を制限） | **CPU コア・RAM・ディスクのハード分離** (40GB Disk/Inst) |
| **インターフェイス共有** | **Shared Interface** をサポート (Routed Mode のみ) | **Data-Sharing / FXOS VLAN Subinterface** をサポート |
| **パケット分類** | Ingress Interface、または **Unique MAC / IP** で識別 | FXOS ポート / VLAN カプセル化による論理識別 |
| **VPN 機能** | **Remote Access VPN 非サポート** (Site-to-Site は制限付き) | **Remote Access VPN & Site-to-Site VPN フルサポート** |
| **ルーティング** | **Static Route のみ** (古い ASA OS では Dynamic Routing 非サポート) | **OSPF, BGP, RIP 等の動的ルーティングをフルサポート** |
| **OS バージョン** | 全 Context で同一 ASA OS バージョン | **各 Container で異なる FTD バージョンを実行可能** |
| **HA 形態** | Active/Standby および **Active/Active Failover** | **Chassis 間 Active/Standby HA** (クラスタリングは制約あり) |

---

## 🏗 動作原理

### 1. Cisco ASA Multi-Context アーキテクチャとパケット分類 (Packet Classifier)

Multi-Context ASA では、物理ポートにパケットが到着した際、データプレーンの **Packet Classifier（パケット分類器）** が受信パケットをどのセキュリティコンテキストへ配送するかを決定します。

```text
[ Incoming Frame on Shared Interface (e.g., GigabitEthernet0/0) ]
                            │
                            ▼
      ┌───────────────────────────────────────────┐
      │ ASA Ingress Packet Classifier             │
      └───────────────────────────────────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         │ (Unique MAC Match)│ (Unique Interface)│ (Management IP Match)
         ▼                  ▼                  ▼
┌──────────────────┐┌──────────────────┐┌──────────────────┐
│ Context: SALES   ││ Context: DMZ     ││ Admin Context    │
│  - Conn Table    ││  - Conn Table    ││  - System Admin  │
│  - ACL / NAT     ││  - ACL / NAT     ││  - Syslog / AAA  │
│  - Route Table   ││  - Route Table   ││  - Console Access│
└──────────────────┘└──────────────────┘└──────────────────┘
```

#### パケット分類基準の順序
1. **Unique Ingress Interface**: コンテキストに専用の物理ポートまたは VLAN サブインターフェイスが割り当てられている場合、受信ポートのみで直接特定。
2. **Unique Destination MAC Address (Shared Interface)**: 複数のコンテキストが同じ物理/サブインターフェイスを共有している場合、宛先 MAC アドレスを照合してコンテキストを特定。
   * **MAC 自動生成 (`mac-address auto`)**: 共有インターフェイス使用時、各コンテキストに固有の MAC アドレスを自動割り当てします。
3. **Destination IP Address**: アプライアンス自身の管理インターフェイス宛て通信（SSH/ASDM 等）の場合、IP アドレスで識別。

---

## ⚙ 動作シーケンス

1. **マルチコンテキストモードへの変更 (`mode multiple`)**:
   * システムが再起動し、既存の単一コンフィグが `old_running.cfg` として Flash に退避され、`system` スペースとデフォルトの `admin` コンテキスト（`admin.cfg`）が生成されます。
2. **System Space でのコンテキスト定義**:
   * 管理者が `system` コンテキストで `context <NAME>` を定義し、`config-url`（保存先 URL）と `allocate-interface`（割り当てポート）を指定します。
3. **コンテキスト間切替 (`changeto context <NAME>`)**:
   * 管理者は `changeto` コマンドで指定のコンテキストへ移動し、通常の ASA と同様に `nameif`, `ip address`, `access-list`, `nat` を設定します。
4. **リソース制限 (Resource Management) の適用**:
   * `system` スペースで `class` を定義し、`limit-resource`（接続数、XLATE 数、SSH 数等）を設定してコンテキストに `member` 割り当てを行うことで、特定コンテキストによるリソース枯渇（DoS）を防ぎます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **`mode multiple` 実行に伴う動作理解**:
   * コマンド投入時に自動再起動が発生します。
   * 単一モード時の設定はクリアされ、`disk0:/old_running.cfg` に保存されます。試験で「既存の設定を引き継いでマルチコンテキスト化せよ」と問われた場合、`admin.cfg` や新コンテキストへ旧設定を書き戻す作業が必要です。
2. **Shared Interface における `mac-address auto` の必須性**:
   * 複数のコンテキストで同じ物理ポートや Trunk ポートを共有（`allocate-interface`）する場合、`mac-address auto`（または手動 MAC 指定）を設定しないと、**対向スイッチからの ARP 解決に失敗し、パケット分類が正しく行われません**。
3. **Interface Alias / Visibility (`visible` / `invisible`)**:
   * `allocate-interface GigabitEthernet0/0 INSIDE visible` のように、コンテキスト内での名前エイリアス（マッピング名）を定義できます。
   * `invisible` オプションを付けると、コンテキスト管理者から物理ポート名（例: `GigabitEthernet0/0`）を隠蔽できます。
4. **Resource Management (Resource Class) の構築**:
   * ラボ問題で「Context A の最大同時接続数を 5000、Telnet セッション数を 2 に制限せよ」といった条件が出題されます。
   * `class` 内で `limit-resource conns 5000` や `limit-resource telnet 2` を設定し、`context <NAME>`配下で `member <CLASS_NAME>` を適用します。
5. **Active/Active Failover の構成**:
   * Multi-Context モードでのみ可能な Active/Active Failover では、`failover group 1`（Context A が Active）と `failover group 2`（Context B が Active）を作成し、各コンテキストに `join-failover-group <1|2>` を設定します。
6. **Multi-Context モードの機能制限（試験で狙われやすいポイント）**:
   * **Remote Access VPN (AnyConnect) 非サポート**。
   * **Dynamic Routing Protocols (OSPF/BGP) 非サポート** (古い ASA バージョン/試験仕様による)。
   * **Multicast Routing 非サポート**。
   * **Transparent Mode での Shared Interface 非サポート**。

---

## 🛠 設定方法

### 1. Cisco ASA: Multi-Context モードの有効化と Admin / User Context 構成 (CLI)

```bash
! 1. System Space でマルチコンテキストモードに変換 (再起動プロンプトで承認)
ciscoasa# config t
ciscoasa(config)# mode multiple
WARNING: This command will change the behavior of the device
WARNING: This command will initiate a Reboot
Proceed with change mode? [confirm] <Enter>

! 2. 再起動後、System Space で共有 MAC アドレスの自動生成を有効化
ciscoasa# show mode
Security context mode: multiple

ciscoasa(config)# mac-address auto prefix 19

! 3. Resource Class の作成 (リソース制限)
ciscoasa(config)# class CLASS_GOLD
ciscoasa(config-class)# limit-resource conns 10000
ciscoasa(config-class)# limit-resource rate Conns 1000
ciscoasa(config-class)# limit-resource ssh 5
ciscoasa(config-class)# limit-resource xlate 2000

! 4. Admin Context および User Context (CTX_SALES) の作成
ciscoasa(config)# admin-context admin
ciscoasa(config)# context admin
ciscoasa(config-ctx)# allocate-interface Management0/0
ciscoasa(config-ctx)# config-url disk0:/admin.cfg
!
ciscoasa(config)# context CTX_SALES
ciscoasa(config-ctx)# allocate-interface GigabitEthernet0/0.100 INSIDE
ciscoasa(config-ctx)# allocate-interface GigabitEthernet0/1.200 OUTSIDE
ciscoasa(config-ctx)# config-url disk0:/CTX_SALES.cfg
ciscoasa(config-ctx)# member CLASS_GOLD

! 5. コンテキストへ切り替えて内部ネットワーク設定
ciscoasa(config)# changeto context CTX_SALES
ciscoasa/CTX_SALES# config t
ciscoasa/CTX_SALES(config)# interface INSIDE
ciscoasa/CTX_SALES(config-if)# ip address 10.1.100.254 255.255.255.0
ciscoasa/CTX_SALES(config-if)# security-level 100
!
ciscoasa/CTX_SALES(config)# interface OUTSIDE
ciscoasa/CTX_SALES(config-if)# ip address 203.0.113.10 255.255.255.248
ciscoasa/CTX_SALES(config-if)# security-level 0
```

### 2. Cisco FTD (FXOS Chassis Manager): Multi-Instance (Container Instance) 構成

1. **FXOS にログインし、Resource Profile の作成**:
   * **Platform Settings > Resource Profiles > Add Resource Profile**。
   * **Profile Name**: `PROFILE_MEDIUM`, **Cores**: `6`。
2. **Container Instance の作成**:
   * **Logical Devices > Add Device** を選択。
   * **Device Name**: `FTD_INST_1`, **Template**: `Cisco Secure Firewall Threat Defense`。
   * **Instance Type**: `Container`。
   * **Resource Profile**: `PROFILE_MEDIUM` を割り当て。
3. **管理 IP / データポートのバインド**:
   * Management Interface の IP (`10.1.1.51/24`)、Gateway、および FMC IP (`10.1.1.50`) を指定。
   * データポート（例: `Ethernet1/1`, `Ethernet1/2`）を選択してインスタンスに割り当て。
4. **FMC でのインスタンス追加**:
   * FMC GUI より **Devices > Device Management > Add Device** を実行し、`FTD_INST_1` の管理 IP で追加。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **Context モード確認** | ASA CLI | <code>show mode</code> |
| **Context 一覧および割り当てインターフェイス表示** | ASA CLI (System) | <code>show context</code> |
| **Context 詳細情報表示** | ASA CLI (System) | <code>show context detail</code> |
| **割り当てられた MAC アドレス一覧表示** | ASA CLI | <code>show interface \| include (Interface)\|(MAC)</code> |
| **リソース割り当ておよび使用状況確認** | ASA CLI | <code>show resource allocation detail</code> / <code>show resource usage</code> |
| **Resource Class 定義表示** | ASA CLI | <code>show class</code> |
| **System Space ↔ Context 間の移動** | ASA CLI | <code>changeto context \<NAME\></code> / <code>changeto system</code> |
| **FXOS コンテナインスタンス状態確認** | FXOS CLI | <code>show app-instance</code> / <code>show cgroups container</code> |
| **FXOS から FTD コンテナコンソールへの接続** | FXOS CLI | <code>connect ftd \<INSTANCE_NAME\></code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **Shared Interface で通信が届かない / 片方向** | 各 Context で固有の MAC アドレスが生成されていない | ASA: <code>show interface</code> | System Space で <code>mac-address auto</code> を有効化するか、手動で一意な MAC を定義する。 |
| **`changeto` コマンドで Context に入れない** | Context 名のタイポ（大文字小文字区別あり） | ASA: <code>show context</code> | <code>show context</code> で正確な Context 名を確認して入力する（ケースセンシティブ）。 |
| **特定の Context で接続が拒否される** | Resource Limit（同時接続数等）の上限突破 | ASA: <code>show resource usage detail</code> | Resource Class の <code>limit-resource conns</code> 等の制限値を引き上げる。 |
| **AnyConnect VPN を設定しようとするとエラー** | Multi-Context モードでは Remote Access VPN 非サポート | ASA: <code>show mode</code> | AnyConnect が必要な場合は Single-Context モード（または FTD Multi-Instance）を使用する。 |
| **FXOS で Container Instance が起動しない (DISK ERROR)** | 初回コンテナ作成時のディスク初期化未実施 | FXOS: <code>show app-instance</code> | FXOS の Security Modules 画面で **Reinitialize** を実行しディスク領域をフォーマットする。 |

---

## ⚠ 制限事項

* **Cisco ASA Multi-Context モードの非サポート機能**:
  * **Remote Access VPN (AnyConnect / Secure Client)**。
  * **Dynamic Routing Protocols (OSPF, BGP, EIGRP)** (※古い ASA OS / 標準ラボ仕様)。
  * **Multicast Routing (PIM / IGMP)**。
  * **QoS (Quality of Service) および Threat Detection の一部機能**。
* **Cisco FTD の Multi-Context 非サポート**:
  * FTD ソフトウェア自体は ASA 形式の Multi-Context をサポートしません（FXOS Multi-Instance で代替）。

---

## 🔄 他技術との関連

* **1.3 High Availability & Clustering**:
  * ASA Multi-Context モードでは **Active/Active Failover** が可能（Context ごとに Active 機器を分散）。
* **3.1 NAT**:
  * 各 Context 内で独立した Auto NAT および Manual Twice NAT を構成します。

---

## 🧩 比較表

### 1. ASA Multi-Context vs FTD Multi-Instance (Container)

| 比較項目 | Cisco ASA Multi-Context Mode | Cisco FTD Multi-Instance (Container) |
| :--- | :--- | :--- |
| **基盤 OS** | ASA LINA OS (単一カーネル) | FXOS (Linux Container / Docker) |
| **分離レベル** | 論理コンフィグ分離 (メモリ/CPU 共有) | **完全ハードウェアリソース分離** (CPU/RAM/Disk) |
| **個別再起動** | 不可 (1 つの障害が全 Context に波及する可能性) | **可能** (個別コンテナを独立リロード/アップグレード) |
| **Remote Access VPN** | 非サポート | **フルサポート** |
| **動的ルーティング** | 非サポート | **フルサポート** (OSPF/BGP) |
| **管理コンソール** | CLI / ASDM | **FMC (Firepower Management Center) 一元管理** |

---

## 💡 ベストプラクティス

1. **`mac-address auto` の標準有効化**:
   * Multi-Context ASA を構築する際は、インターフェイス共有時の通信障害を防ぐため、System Space で即座に `mac-address auto` を投入します。
2. **Admin Context の専用化**:
   * `admin` コンテキストにはデータ通信を流さず、管理通信（Syslog, AAA, SNMP, 外部コンフィグダウンロード）専用として保持します。
3. **Resource Class によるガード**:
   * すべての User Context を `default` クラスのまま運用せず、用途に応じた Resource Class を割り当てて接続数・XLATE 数を制御します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA Multi-Context モードへの変更と Admin/User Context の初期構成

#### 問題
ASA1 を Multi-Context モードに変更し、`admin` コンテキスト（管理用）および `CTX_CUST_A` コンテキストを作成せよ。

#### 要件
* モード変更時の自動再起動を承認すること。
* コンフィグ URL は `disk0:/admin.cfg` および `disk0:/CTX_CUST_A.cfg` とすること。

#### 設定例

```bash
! モード変更 (System Space)
config t
mode multiple

! 再起動完了後、System Space にて基本構成
mac-address auto

admin-context admin

context admin
 allocate-interface Management0/0
 config-url disk0:/admin.cfg
!
context CTX_CUST_A
 allocate-interface GigabitEthernet0/0.10 INSIDE
 allocate-interface GigabitEthernet0/1.20 OUTSIDE
 config-url disk0:/CTX_CUST_A.cfg
```

#### 検証方法
* `show context` を実行し、`admin` および `CTX_CUST_A` が `Active` であることを確認。

---

### 例2: ASA Shared Interface における MAC アドレス自動生成の適用

#### 問題
`GigabitEthernet0/2` ポートを `CTX_1` と `CTX_2` で共有（Shared Interface）せよ。上流ルータでの ARP 競合を防止するため、プレフィックス `19` を使用した自動 MAC 生成を有効化せよ。

#### 要件
* System Space にて設定すること。

#### 設定例

```bash
! System Space
mac-address auto prefix 19

context CTX_1
 allocate-interface GigabitEthernet0/2 OUTSIDE
 config-url disk0:/CTX_1.cfg
!
context CTX_2
 allocate-interface GigabitEthernet0/2 OUTSIDE
 config-url disk0:/CTX_2.cfg
```

#### 検証方法
* `changeto context CTX_1` 後、`show interface OUTSIDE` で MAC アドレスを確認。
* `changeto context CTX_2` 後、`show interface OUTSIDE` で異なる MAC アドレスが割り当てられていることを確認。

---

### 例3: Resource Class による同時接続数および SSH セッション数の制限

#### 問題
`CTX_CUST_A` に対し、最大同時 TCP/UDP 接続数を `5000`、最大 SSH セッション数を `3` に制限する Resource Class `CLASS_CUST_A` を作成・適用せよ。

#### 要件
* System Space にて設定すること。

#### 設定例

```bash
! System Space
class CLASS_CUST_A
 limit-resource conns 5000
 limit-resource rate Conns 1000
 limit-resource ssh 3

context CTX_CUST_A
 member CLASS_CUST_A
```

#### 検証方法
* `show class CLASS_CUST_A` で定義を確認。
* `show resource allocation detail` で `CTX_CUST_A` に制限値が適用されていることを確認。

---

### 例4: ASA Multi-Context モードにおける Active/Active Failover 構成

#### 問題
2 台の ASA (ASA1, ASA2) 間で Active/Active Failover を構成せよ。`CTX_1` は ASA1 で Active (Group 1)、`CTX_2` は ASA2 で Active (Group 2) として動作させよ。

#### 要件
* LAN ベース Failover ポートとして `GigabitEthernet0/3` を使用すること。

#### 設定例

```bash
! [ASA1 - System Space]
failover
failover lan unit primary
failover lan interface FO_LINK GigabitEthernet0/3
failover interface ip FO_LINK 172.16.255.1 255.255.255.252 standby 172.16.255.2
failover link FO_LINK

failover group 1
 primary
failover group 2
 secondary

context CTX_1
 allocate-interface GigabitEthernet0/0.10
 config-url disk0:/CTX_1.cfg
 join-failover-group 1

context CTX_2
 allocate-interface GigabitEthernet0/0.20
 config-url disk0:/CTX_2.cfg
 join-failover-group 2
```

#### 検証方法
* `show failover` を実行し、Group 1 が ASA1 で Active、Group 2 が ASA2 で Active であることを確認。

---

### 例5: Multimode Transparent Firewall (MMTF) の Context 構成

#### 問題
ASA1 上で `CTX_TRANS_1` を作成し、Transparent Firewall Mode で動作させよ。`GigabitEthernet0/0.30` と `GigabitEthernet0/1.30` を割り当て、BVI1 に IP `10.30.1.254/24` を指定せよ。

#### 要件
* コンテキスト内で `firewall transparent` を有効化すること。

#### 設定例

```bash
! System Space
context CTX_TRANS_1
 allocate-interface GigabitEthernet0/0.30
 allocate-interface GigabitEthernet0/1.30
 config-url disk0:/CTX_TRANS_1.cfg

! Context 内設定
changeto context CTX_TRANS_1
config t
firewall transparent

interface BVI1
 ip address 10.30.1.254 255.255.255.0
 no shutdown

interface GigabitEthernet0/0.30
 nameif inside
 bridge-group 1
 security-level 100

interface GigabitEthernet0/1.30
 nameif outside
 bridge-group 1
 security-level 0
```

#### 検証方法
* `CTX_TRANS_1` 内で `show firewall` を実行し `Transparent` を確認。

---

### 例6: Interface Alias と Visibility (`invisible`) による物理ポートの隠蔽

#### 問題
管理者がコンテキスト `CTX_SECURE` 内から物理ポート名（`GigabitEthernet0/0.50`）を識別できないよう、エイリアス名 `INTERNAL_NET` として割り当て、`invisible` オプションを適用せよ。

#### 要件
* System Space で設定すること。

#### 設定例

```bash
! System Space
context CTX_SECURE
 allocate-interface GigabitEthernet0/0.50 INTERNAL_NET invisible
 config-url disk0:/CTX_SECURE.cfg
```

#### 検証方法
* `changeto context CTX_SECURE` 後、`show interface` を実行し、物理ポート名が表示されず `INTERNAL_NET` のみが表示されることを確認。

---

### 例7: FXOS (Firepower 4100/9300) における FTD Multi-Instance (Container) のデプロイ

#### 問題
FXOS Chassis Manager を使用して、セキュリティモジュール 1 上に CPU コア 8 個を割り当てた FTD コンテナインスタンス `FTD_CONTAINER_A` を作成せよ。

#### 要件
* FMC 管理者 IP: `10.0.0.50`

#### 設定例

```text
1. FXOS Chassis Manager: Platform Settings > Resource Profiles > Add:
   - Name: PROFILE_8CORE, Cores: 8
2. Logical Devices > Add Device:
   - Name: FTD_CONTAINER_A
   - Template: Cisco Secure Firewall Threat Defense
   - Instance Type: Container
   - Resource Profile: PROFILE_8CORE
   - Management IP: 10.0.0.101/24, Gateway: 10.0.0.1, FMC IP: 10.0.0.50
3. Data Ports 画面で Ethernet1/1 および Ethernet1/2 を選択して保存・デプロイ。
```

#### 検証方法
* FXOS CLI で `show app-instance` を実行し、`FTD_CONTAINER_A` が `ONLINE` であることを確認。

---

### 例8: FXOS VLAN Subinterface を使用した複数 FTD コンテナ間での物理ポート共有

#### 問題
FXOS 上で物理ポート `Ethernet1/5` 上に VLAN 100 サブインターフェイス (`Ethernet1/5.100`) を作成し、`FTD_INST_1` に割り当てよ。

#### 要件
* FXOS 側で L2 VLAN サブインターフェイスを定義すること。

#### 設定例

```text
1. FXOS Chassis Manager: Interfaces > Add New > Subinterface.
   - Port: Ethernet1/5, Subinterface ID: 100, VLAN ID: 100
2. Logical Devices > FTD_INST_1 の編集:
   - Data Ports で Ethernet1/5.100 にチェックを入れて保存。
```

#### 検証方法
* FXOS CLI で `show subinterface` を実行し、`Ethernet1/5.100` が `FTD_INST_1` にマウントされていることを確認。

---

### 例9: ASA Multi-Context モードでの Cascade Context 接続 (VFW 間ルーティング)

#### 問題
ASA1 上の `CTX_ADMIN` と `CTX_USER` を、物理ケーブルや外部スイッチを経由させずに、内部の仮想ペアインターフェイス（Cascade Connection）で直接 L3 接続せよ。

#### 要件
* サブインターフェイスおよび一意の VLAN タグ（例: VLAN 999）を利用して内部接続すること。

#### 設定例

```bash
! System Space
interface GigabitEthernet0/3.999
 vlan 999

context CTX_ADMIN
 allocate-interface GigabitEthernet0/3.999 TO_USER
 config-url disk0:/admin.cfg

context CTX_USER
 allocate-interface GigabitEthernet0/3.999 TO_ADMIN
 config-url disk0:/user.cfg

! CTX_ADMIN 内設定
changeto context CTX_ADMIN
interface TO_USER
 ip address 192.168.255.1 255.255.255.252
 security-level 100

! CTX_USER 内設定
changeto context CTX_USER
interface TO_ADMIN
 ip address 192.168.255.2 255.255.255.252
 security-level 0
```

#### 検証方法
* `CTX_ADMIN` から `192.168.255.2` 宛てに ICMP Ping を送信し疎通を確認。

---

### 例10: ASA Multi-Context モードから Single-Context モードへの復元手順

#### 問題
Multi-Context モードで動作している ASA1 を、退避されている `disk0:/old_running.cfg` の設定を復元した上で Single-Context モードへ戻せ。

#### 要件
* ロールバック時に旧設定を全消去させないこと。

#### 設定例

```bash
! 1. System Space であることを確認
changeto system

! 2. 旧 Single-Mode コンフィグを startup-config へ上書きコピー
copy disk0:/old_running.cfg startup-config

! 3. モードを Single に変更 (自動再起動承認)
config t
mode single
! (Reboot 確認で Enter)
```

#### 検証方法
* 再起動完了後、`show mode` で `Security context mode: single` を確認。
* `show interface ip brief` で以前の IP アドレスが復元されていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ASA1 の System Space にて以下の設定が投入されている。`CTX_SALES` と `CTX_MKTG` が物理ポート `GigabitEthernet0/1` を共有（Shared Interface）している場合、外部ルータから両コンテキスト宛ての ARP 解決が失敗する原因はどれか。

```bash
context CTX_SALES
 allocate-interface GigabitEthernet0/1 OUTSIDE
 config-url disk0:/sales.cfg
!
context CTX_MKTG
 allocate-interface GigabitEthernet0/1 OUTSIDE
 config-url disk0:/mktg.cfg
```

* A) `allocate-interface` に `visible` キーワードが付与されていないため。
* B) System Space で `mac-address auto` が有効化されておらず、両コンテキストの `OUTSIDE` インターフェイスが同じ物理 MAC アドレスを使用しているため。
* C) `admin-context` が定義されていないため。
* D) Shared Interface は Transparent Firewall Mode でしかサポートされないため。

**正解**: **B**
**解説**: 複数のコンテキストで同じ物理/論理ポートを共有する場合、各コンテキストのインターフェイスに一意な MAC アドレスが割り当てられていないと（`mac-address auto` または手動指定）、L2 分類および上流ルータでの ARP 解決に失敗します。

---

### 2. トラブルシュート問題

**問題**: 管理者が ASA1 のコンテキスト `CTX_USER1` 内にログインし、Cisco AnyConnect Remote Access VPN を構成しようとしたところ、コマンド入力時にエラーが発生した。原因として正しいものはどれか。

* A) ASA の Flash メモリ内に AnyConnect PKG ファイルが存在しないため。
* B) Cisco ASA の Multi-Context モードでは Remote Access VPN 機能がサポートされていないため。
* C) `CTX_USER1` に Resource Class が割り当てられていないため。
* D) `admin` コンテキスト以外のユーザーコンテキストでは VPN 機能がすべて遮断されるため。

**正解**: **B**
**解説**: Cisco ASA の Multi-Context モードにおける明確な制限事項として、**Remote Access VPN (AnyConnect / Secure Client) は非サポート** です。AnyConnect を終端するには Single-Context モード（または FTD Multi-Instance）を使用する必要があります。

---

### 3. Design 問題

**問題**: 大規模データセンターにおいて、セキュリティアプライアンスのマルチテナント化を行いたい。テナント間で「独立した FTD OS バージョンの実行」「完全な CPU/RAM ハードウェアリソース分離」「動的ルーティング (OSPF/BGP) の利用」「Remote Access VPN の終端」をすべて満たす設計として最も適切なものはどれか。

* A) Cisco ASA 5585-X 上で ASA Multi-Context モードを導入する。
* B) Firepower 4100 シャシー上で FXOS を使用し、FTD Container Instances (Multi-Instance) をデプロイする。
* C) ASA 上で Multimode Transparent Firewall (MMTF) を構成する。
* D) FTD を FDM でローカル管理し、Multi-Context モードを有効化する。

**正解**: **B**
**解説**: FTD Multi-Instance (Container Instance on FXOS) は、CPU/RAM/Disk のハード分離、独立した OS バージョン実行、動的ルーティング、および Remote Access VPN のすべてをサポートする次世代マルチテナントアーキテクチャです。ASA Multi-Context モードでは動的ルーティングや AnyConnect VPN がサポートされません。

---

### 4. 実装問題

**問題**: ASA1 を Single-Context モードから Multi-Context モードに変更するために `mode multiple` コマンドを実行した。この操作によって直後に発生するイベントとして正しいものはどれか。

* A) 既存の単一コンフィグが即座に `admin` コンテキストへ自動変換され、無停止で切り替わる。
* B) デバイスが即座に自動再起動し、既存の Single-Mode コンフィグが `disk0:/old_running.cfg` に退避された状態で起動する。
* C) フラッシュメモリ内のすべてのファイルが初期化消去される。
* D) 全インターフェイスが自動的に `shutdown` 状態になり、通信が二重化される。

**正解**: **B**
**解説**: `mode multiple` を実行すると再起動が必要となり、既存のコンフィグは `old_running.cfg` として保存され、`system` スペースおよび初期の `admin` コンテキスト（`admin.cfg`）が生成されます。

---

### 5. コンフィグ読解問題

**問題**: ASA1 の System Space で以下のコマンドを実行した。この設定の目的として正しい説明はどれか。

```bash
class LIMIT_SSH
 limit-resource ssh 2
!
context CTX_GUEST
 member LIMIT_SSH
```

* A) `CTX_GUEST` 内のユーザーが外部へ接続できる SSH セッション数を最大 200 に拡大する。
* B) `CTX_GUEST` コンテキストに対する同時 SSH 管理ログインセッション数を最大 2 セッションに制限し、管理リソースの枯渇を防止する。
* C) SSH ポート (TCP 22) のパケットスループットを 2 Mbps に制限する。
* D) 管理コンテキスト `admin` への SSH アクセスを遮断する。

**正解**: **B**
**解説**: Resource Class (`class`) で `limit-resource ssh 2` を定義し、コンテキストに `member` 適用することで、該当コンテキストに対する同時 SSH セッション数を 2 に制限します。

---

## 🔗 参考リソース

* [Cisco ASA Series General Operations Configuration Guide - Security Contexts](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/general/asa-94-general-config/ha-contexts.html)
* [Cisco Secure Firewall Threat Defense Configuration Guide (FMC) - Multi-Instance Capability](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/multi_instance_capability_for_firepower_threat_defense.html)
* [Cisco Live: Advanced ASA and FTD Virtualization Architectures (BRKSEC-3020)](https://www.ciscolive.com/)
* [CVD: Secure Firewall Multi-Instance Deployment Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/cisco-secure-firewall-design-guide.html)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * ASA の **Multi-Context Mode** は「単一 OS 内での論理分割 (コンフィグ分離)」です。
  * FTD の **Multi-Instance (Container)** は「FXOS 上での Docker コンテナによる独立実行 (ハード分離)」です。
  * 試験で ASA の Multi-Context が出題された場合、**`mac-address auto` の忘れ** と **Resource Management (`class` / `limit-resource`) の定義** が配点ポイントになります。
* **図解**:
  * `ASA Physical Appliance -> System Space (Global) -> [ Context Admin | Context A | Context B ]`
  * `FP4100/9300 Chassis -> FXOS -> [ FTD Container 1 (6 Cores) | FTD Container 2 (8 Cores) ]`
* **注意点**:
  * ラボ試験で Multi-Context モードの ASA において AnyConnect や Dynamic Routing を設定させる問題が出た場合はトラップ（仕様上不可能）ですので、要件を再確認するか Single-Context モードへの変更が必要ないかを吟味してください。
