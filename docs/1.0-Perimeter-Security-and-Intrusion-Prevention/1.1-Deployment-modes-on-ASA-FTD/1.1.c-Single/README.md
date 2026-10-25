---
layout: default
title: 1.1.c-Single
nav_order: 3
parent: 1.1-Deployment-modes-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.1.c Single

Cisco ASA (Adaptive Security Appliance) および Cisco FTD (Firepower Threat Defense) における **Single（シングルモード / 単一インスタンス）** デプロイメントは、物理または仮想アプライアンス全体を単一の管理・コントロールプレーンおよびデータプレーンとして使用する標準的な動作形態です。Cisco ASA における **Single Context Mode（単一コンテキストモード）** や、Cisco FTD における **Single Instance (Native Instance)** / **FDM (Firepower Device Manager) による単一デバイス管理** がこれに該当します。

---

## 📘 概要

* **機能概要**:
  * **Cisco ASA (Single Context Mode)**: アプライアンス上に単一の `running-config` および単一のルーティングテーブル（FIB）を保持し、すべての物理・論理インターフェイスを単一のセキュリティポリシーで一括管理するデフォルトの動作モードです。
  * **Cisco FTD (Single / Native Instance)**: Firepower 4100/9300 等の FXOS ハードウェアシャシー上で、セキュリティモジュール/エンジンのすべてのハードウェアリソース（CPU, メモリ, ディスク）を占有する単一の FTD アプリケーションインスタンス（Native Instance）としてデプロイする形態です。また、単体アプライアンスをローカルWeb UI（FDM）でスタンドアロン運用する構成も含みます。
* **利用目的**:
  * 単一組織や単一セキュリティドメインに対して、アプライアンスが持つすべての機能（フル機能セット）を提供し、パフォーマンスを最大化する。
  * 複数テナントの論理分離が不要な環境において、運用構成の複雑さを排除し、管理ポインタを一本化する。
* **どのような場面で利用するか**:
  * 企業ネットワークのインターネット境界（Internet Edge）やデータセンター境界。
  * Remote Access VPN (Cisco AnyConnect / Secure Client) や Site-to-Site IPsec VPN の集約ゲートウェイ。
  * 動的ルーティングプロトコル (OSPF, BGP, EIGRP) やマルチキャストルーティング、QoS などの高度な L3 機能をフル活用する拠点。

---

## 🔑 要点

| 項目 | Cisco ASA (Single Context Mode) | Cisco FTD (Single / Native Instance) |
| :--- | :--- | :--- |
| **デフォルト状態** | デフォルトモード (`mode single`) | デフォルトのデプロイ形式 (Native Instance) |
| **全機能のサポート** | **全機能サポート** (AnyConnect, 動的ルーティング, QoS, Threat Detection 等) | **全機能サポート** (FMC 管理時、VPN/ルーティング/NGIPS/AMP フル機能) |
| **管理プレーン** | 単一のグローバル設定 (`startup-config` / `running-config`) | 単一の FMC ポリシー割り当て、または FDM による単体管理 |
| **ルーティング機能** | OSPFv2/v3, BGP, EIGRP, RIP, Static, PBR, Multicast フルサポート | OSPFv2/v3, BGP, EIGRP, RIP, Static, PBR フルサポート |
| **VPN 機能** | Remote Access VPN (AnyConnect) & Site-to-Site IPsec 双方をフルサポート | Remote Access VPN & Site-to-Site IPsec 双方をフルサポート (FMC/FDM) |
| **リソース割り当て** | 全 CPU / メモリを単一インスタンスが動的に使用 | シャシー内の全 CPU / メモリを 100% 占有 (Native) |
| **モード切替動作** | `mode multiple` 実行で再起動発生（旧設定は `old_running.cfg` へ待避） | FXOS で Native / Container インスタンスを定義。FDM ↔ FMC 切替時は設定初期化 |

---

## 🏗 動作原理

Single モードでは、アプライアンスのすべてのハードウェアリソース（CPU コア、NIC、暗号化アクセラレータ、メモリ）が単一の OS インスタンス（LINA / Snort 3）によって統合管理されます。

```text
┌────────────────────────────────────────────────────────────────────────┐
│ Cisco ASA / FTD Appliance (Single Context / Single Native Instance)    │
│                                                                        │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ Single Control Plane & Management Space                            │ │
│ │  - Single Global Running Configuration                             │ │
│ │  - Single Unified Routing Table (RIB / FIB)                        │ │
│ │  - Global Management Access (SSH / HTTPS / FMC / ASDM / FDM)       │ │
│ └────────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ Single Unified Data Plane (LINA + Snort 3 Inspection Engine)       │ │
│ │  - Ingress Interface (Gi0/0 - Inside)                               │ │
│ │  - Global Connection Table (Conn Table - Unified Stateful Tracking)│ │
│ │  - Global Access Control Policy / NAT Policy Evaluation             │ │
│ │  - Egress Interface (Gi0/1 - Outside)                              │ │
│ └────────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ Full Security Feature Capabilities                                  │ │
│ │  - Dynamic Routing (OSPF / BGP / EIGRP)                            │ │
│ │  - Remote Access VPN (Cisco AnyConnect / Secure Client)            │ │
│ │  - Site-to-Site IPsec VPN / Threat Detection / QoS                 │ │
│ └────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

1. **システム起動と設定ロード**:
   * システム起動時、単一の不揮発性ストレージ（Flash / SSD）から `startup-config`（ASA）または FMC からデプロイされたポリシーデータベース（FTD）を単一のメモリ空間に読み込みます。
2. **物理ポート・論理インターフェイスの統合マッピング**:
   * アプライアンスに接続されたすべての物理ポート、Subinterface、EtherChannel、Redundant Interface が単一のシステムインスタンスに紐付けられます。
3. **パケットインスペクションとテーブル検索**:
   * パケットを受信すると、単一のグローバル接続テーブル（Conn Table）を参照します。
   * 新規セッションの場合、グローバル Access Control List (ACL) / Access Control Policy (ACP)、NAT ポリシー、ルーティングテーブル（FIB）を評価して最適な Egress ポートへ転送します。
4. **暗号化・VPN 処理**:
   * VPN パケットを受信した場合、アプライアンス全体で共有される暗号化ハードウェア（SSL/IPsec コプロセッサ）を使用して終端・復号し、内部インターフェイスへルーティングします。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

* **Multi-Context から Single Context モードへのロールバック手順**:
  * ラボ問題で「ASA の Multi-Context モードを解除して Single Context モードに戻せ」と要求される場合があります。
  * システム実行スペース（System Execution Space）で `copy disk0:old_running.cfg startup-config` を実行してから `mode single` を入力し、再起動を承認する必要があります。
  * **注意**: 単に `mode single` を実行するだけでは以前のコンフィグは復元されず、デフォルトの初期設定に初期化されてしまいます。
* **AnyConnect VPN や Dynamic Routing の要件**:
  * 試験で 「AnyConnect Remote Access VPN を終端せよ」「OSPF / BGP による動的ルーティングを構成せよ」 という要件がある場合、**Cisco ASA では Multi-Context モードが使用できません（非サポート）**。必ず Single Context モードで構成する必要があります。
* **FTD における FDM から FMC への管理者変更 (Manager Switching)**:
  * 単体 FTD（FDM 管理）を FMC 中央管理へ切り替える際、FTD CLI で `configure manager add <FMC_IP> <Registration_Key>` を実行します。
  * **重要**: FDM から FMC へ管理権限を移行すると、**FTD 上の既存データプレーン設定（インターフェイスや ACL、NAT 等）は原則として初期化・消去** され、FMC 側のポリシーで完全に上書きされます。
* **FMC Access on Data Interface の競合と解消 (Acknowledge)**:
  * 単一 FTD のデータポート経由で FMC 管理を行う際、FTD CLI で `configure network management-data-interface` を実行すると、FMC 側のデータベースと設定差分（Discrepancy）が発生します。
  * FMC の `FMC Access Details` ポップアップ画面で **Acknowledge** ボタンを押して不一致を認識・解除しないと、FMC からのポリシーデプロイが永久にブロックされます。

---

## 🛠 設定方法

### 1. Cisco ASA: Multi-Context モードから Single Context モードへの復元・変更 (CLI)

```bash
! 1. System Execution Space であることを確認
ciscoasa# show mode
Security context mode: multiple

! 2. 過去の Single-Mode 実行コンフィグ (old_running.cfg) を startup-config へコピー
ciscoasa# copy disk0:old_running.cfg startup-config
Source filename [old_running.cfg]? <Enter>
Destination filename [startup-config]? <Enter>

! 3. モードを Single に変更 (再起動が実行されます)
ciscoasa# config t
ciscoasa(config)# mode single
WARNING: This command will change the behavior of the device
WARNING: This command will initiate a Reboot
Proceed with change mode? [confirm] <Enter>

! 4. 再起動後の確認
ciscoasa# show mode
Security context mode: single
```

### 2. Cisco FTD: CLI による FDM (Local) から FMC (Centralized) への管理者切り替え

```bash
! 1. FTD コンソール CLI にログイン (現在 FDM 管理)
> show managers
Managed by : Local (FDM)

! 2. FMC 管理への切り替え登録コマンド実行 (※データプレーン設定が初期化されます)
> configure manager add 10.1.1.50 SecretRegistrationKey123

! 3. 接続ステータスの確認
> show managers
Host                     : 10.1.1.50
Registration             : Completed
Status                   : Connected
```

### 3. Cisco FTD (FMC GUI): Single Instance FTD における AnyConnect Remote Access VPN 構成

1. **AnyConnect パッケージのアップロード**:
   * **Objects > Object Management > VPN > AnyConnect File** で `.pkg` ファイルを追加。
2. **Remote Access VPN ウィザードの実行**:
   * **Devices > VPN > Remote Access** へ移動し **Add** をクリック。
   * **Name**: `RA-VPN-Single`, **Device**: 対象の Single FTD インスタンスを選択。
   * **Connection Profile**: グループポリシー、AAA サーバー（ISE / LDAP）、IP アドレスプールを指定。
   * **Interfaces**: 外部インターフェイス（例: `outside`）を指定し、AnyConnect パッケージをバインド。
3. **Deploy の実行**:
   * **Deploy** をクリックして Single FTD アプライアンスへ適用。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **コンテキストモード確認** | ASA CLI | <code>show mode</code> |
| **システムバージョンおよびライセンス表示** | ASA / FTD CLI | <code>show version</code> |
| **インターフェイス IP / ステータス確認** | ASA / FTD CLI | <code>show interface ip brief</code> |
| **マネージャー（FMC/FDM）接続ステータス確認** | FTD CLI | <code>show managers</code> |
| **sftunnel 接続詳細確認** | FTD CLI | <code>sftunnel-status-brief</code> |
| **FXOS アプリケーションインスタンス確認** | FXOS CLI | <code>show app-instance</code> |
| **ルーティングテーブル表示** | ASA / FTD CLI | <code>show route</code> |
| **アクティブ接続テーブル表示** | ASA / FTD CLI | <code>show conn</code> / <code>show conn detail</code> |
| **Remote Access VPN セッション確認** | ASA / FTD CLI | <code>show vpn-sessiondb remote</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **`mode single` 実行後に以前の設定が消えた** | `copy disk0:old_running.cfg` を行わずにモード変更した | ASA: <code>dir disk0:</code> | Flash 内に保存されている `old_running.cfg` またはバックアップファイルから設定をテキスト復元する。 |
| **FTD を FMC に追加できない** | FMC ↔ FTD 間の sftunnel (TCP 8305) 遮断 | FTD: <code>show managers</code> | FTD CLI で `configure manager add` の IP/キーを再確認し、途中 FW で TCP 8305 が許可されているか検証。 |
| **FMC デプロイがブロックされる** | FTD ローカル CLI でデータポート管理アドレス等を変更した | FMC GUI: `FMC Access Details` | FMC の `FMC Access Details` 画面を開き、差分を確認した上で **Acknowledge** を押してブロック解除。 |
| **AnyConnect 接続時に証明書エラーが出る** | SSL 証明書が Trustpoint にバインドされていない | ASA/FTD: <code>show crypto ca certificates</code> | トラストポイントに正しく CA / サーバー証明書をバインドし、RA VPN 設定で選択する。 |
| **動的ルーティングのネイバーが張れない** | インターフェイスの Security Zone 設定不備、または ACL 遮断 | ASA/FTD: <code>show ospf neighbor</code> | OSPF/BGP 用マルチキャスト/ユニキャスト通信が Access Control Policy で許可されているか確認。 |

---

## ⚠ 制限事項

* **ハードウェアマルチテナント機能の非活用**:
  * Single Context / Single Native Instance では、複数顧客や複数部門間で完全な設定・データプレーン分離を行うことができません。
* **障害影響範囲の広大さ**:
  * プロセスクラッシュや設定ミス（ルーティングループ等）が発生した場合、アプライアンス全体の通信に影響が及びます。
* **FDM (Local Management) の機能制限**:
  * FTD を FDM で単体管理する場合、FMC で利用可能な一部の高度機能（高度な相関分析、マルチインスタンス管理、一部の複雑な PBR/クラスタリング等）が制限されます。

---

## 🔄 他技術との関連

* **1.1.a Routed / 1.1.b Transparent**: Single Context / Single Instance 環境下で、レイヤ 3 ルーテッドモードまたはレイヤ 2 トランスペアレントモードのいずれかを選択します。
* **4.0 Secure Connectivity (VPN)**: AnyConnect Remote Access VPN や FlexVPN は Single Context / Single Instance モードでのみ完全稼働します。
* **1.3 High Availability & Clustering**: Single Context ASA や Single Instance FTD では **Active/Standby Failover** または **Clustering** 構成が標準となります。

---

## 🧩 比較表

### 1. ASA Single-Context vs ASA Multi-Context

| 項目 | ASA Single-Context Mode | ASA Multi-Context Mode |
| :--- | :--- | :--- |
| **設定ファイル** | 単一の `startup-config` / `running-config` | System Context ＋ Admin Context ＋ 各 Data Context |
| **Dynamic Routing** | **サポート** (OSPF, BGP, EIGRP, RIP) | **非サポート** (Static ルートのみ) |
| **Remote Access VPN** | **サポート** (Cisco AnyConnect / Secure Client) | **非サポート** |
| **Multicast Routing** | **サポート** (PIM, IGMP) | **非サポート** |
| **QoS / Threat Detection** | **サポート** | **一部制限あり** |
| **Failover 形態** | Active/Standby Failover のみ | Active/Standby および **Active/Active Failover** |

### 2. FTD Native Instance (Single) vs FTD Multi-Instance (Container)

| 項目 | FTD Native Instance (Single) | FTD Multi-Instance (Container) |
| :--- | :--- | :--- |
| **リソース配分** | セキュリティモジュールの CPU/RAM を 100% 占有 | リソースプロファイル（CPU コア数/RAM）ごとに指定分割 |
| **インスタンス数** | 1 モジュールにつき 1 インスタンスのみ | 1 モジュールにつき複数のコンテナインスタンス |
| **管理モデル** | FMC または FDM による管理 | **FMC による一元管理のみ** (FDM 不可) |
| **OS バージョン** | 全体が同一 OS | インスタンスごとに異なる FTD バージョン実行可能 |
| **クラスタリング** | サポート | 非サポート |

### 3. FTD FDM Managed (Single Device) vs FTD FMC Managed

| 項目 | FDM Managed (Single Device) | FMC Managed (Centralized) |
| :--- | :--- | :--- |
| **管理インターフェース** | オンボックス Web UI (HTTPS) | 中央集中管理アプライアンス (FMC) |
| **推奨運用規模** | 小規模拠点・単体アプライアンス | 中〜大規模・多拠点・統合セキュリティ分析 |
| **マネージャ変更** | CLI から `configure manager add` で FMC に移行可能 | FDM へ戻す場合は設定初期化が必要 |

---

## 💡 ベストプラクティス

1. **機能要件に応じたモード選定**:
   * AnyConnect Remote Access VPN や OSPF/BGP 動的ルーティングが要件に含まれる場合、ASA では迷わず **Single Context Mode** を選択します。
2. **FXOS 上での Native Instance 活用**:
   * Firepower 4100/9300 において、最大の脅威処理スルー・プットと全機能を求められる境界防御アプライアンスでは、マルチインスタンスではなく **Native Instance** を使用します。
3. **モード変更前のバックアップ保存**:
   * ASA や FTD でデプロイモードやマネージャ（FDM ↔ FMC）を変更する際は、必ず事前に設定ファイル（`show running-config` や FMC バックアップアーカイブ）を外部へ待避させます。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA: Multi-Context モードから Single Context モードへのロールバック構成

#### 問題
ASA1 は現在 Multi-Context モードで動作している。Flash 内に保存されている `disk0:old_running.cfg` を使用して、設定を Single Context モードへロールバックさせよ。

#### 要件
* モード変更後に Single Context モードとして正常起動させること。

#### 設定例

```bash
! 1. System Context であることを確認
changeto system

! 2. バックアップコンフィグを startup-config へコピー
copy disk0:old_running.cfg startup-config

! 3. モードを Single に変更して再起動
config t
mode single
! (Reboot 確認プロンプトで Enter)
```

#### 検証方法
* 再起動完了後、`show mode` を実行し `Security context mode: single` であることを確認。
* `show interface ip brief` で以前の Single Mode インターフェイス設定が復元されていることを確認。

---

### 例2: ASA: Single Context モードにおける OSPFv2 と Dynamic NAT/PAT の統合構成

#### 問題
Single Context モードの ASA1 において、`inside` (IP: `10.1.1.254/24`) および `outside` (IP: `209.165.200.226/29`) を構成せよ。内部サブネット `10.1.1.0/24` から外部への通信を `outside` IP で動的 PAT 変換し、`inside` インターフェイス上で OSPF Process 1 (Area 0) を有効化せよ。

#### 要件
* OSPF Router ID: `10.1.1.254`

#### 設定例

```bash
! インターフェイス構成
interface GigabitEthernet0/0
 nameif inside
 security-level 100
 ip address 10.1.1.254 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/1
 nameif outside
 security-level 0
 ip address 209.165.200.226 255.255.255.248
 no shutdown

! Dynamic PAT (Auto NAT) 設定
object network OBJ_INSIDE_NET
 subnet 10.1.1.0 255.255.255.0
 nat (inside,outside) dynamic interface

! OSPFv2 動的ルーティング構成
router ospf 1
 router-id 10.1.1.254
 network 10.1.1.0 255.255.255.0 area 0
 log-adj-changes
```

#### 検証方法
* `show ospf neighbor` で OSPF ネイバー確立を確認。
* `show nat` および `show xlate` で PAT 変換テーブルを確認。

---

### 例3: ASA: Single Context モードでの AnyConnect Remote Access VPN 終端構成

#### 問題
Single Context モードの ASA1 において、`outside` インターフェイス上で AnyConnect Remote Access VPN を構成せよ。クライアントプールとして `192.168.100.100-192.168.100.200/24` を割り当てよ。

#### 要件
* 認証にはローカルユーザーデータベースを使用すること。

#### 設定例

```bash
! IP アドレスプールの作成
ip local pool ANYCONNECT_POOL 192.168.100.100-192.168.100.200 mask 255.255.255.0

! AnyConnect パッケージの指定
webvpn
 enable outside
 anyconnect image disk0:/anyconnect-win-4.10.pkg 1

! グループポリシーの作成
group-policy GP_EMPLOYEES internal
group-policy GP_EMPLOYEES attributes
 dns-server value 8.8.8.8
 vpn-tunnel-protocol ssl-client

! トンネルグループの作成
tunnel-group TG_EMPLOYEES type remote-access
tunnel-group TG_EMPLOYEES general-attributes
 address-pool ANYCONNECT_POOL
 default-group-policy GP_EMPLOYEES
tunnel-group TG_EMPLOYEES webvpn-attributes
 group-alias EMPLOYEES enable

! ローカルユーザーの作成
username vpnuser password Cisco123Secret privilege 15
```

#### 検証方法
* AnyConnect クライアントから接続し、`show vpn-sessiondb remote` でアクティブセッションを確認。

---

### 例4: ASA: Single Context モードにおける Active/Standby Failover 構成

#### 問題
2 台の ASA (ASA1: Primary, ASA2: Secondary) 間で Single Context モードの Active/Standby Failover を構成せよ。Failover リンクとして `GigabitEthernet0/2` を使用せよ。

#### 要件
* Failover 共有鍵: `CiscoFailoverKey123`

#### 設定例

```bash
! [ASA1 - Primary]
interface GigabitEthernet0/2
 description Failover Link
 no shutdown
!
failover lan unit primary
failover lan interface FO_LINK GigabitEthernet0/2
failover interface ip FO_LINK 172.16.255.1 255.255.255.252 standby 172.16.255.2
failover key CiscoFailoverKey123
failover link FO_LINK
failover

! [ASA2 - Secondary]
interface GigabitEthernet0/2
 description Failover Link
 no shutdown
!
failover lan unit secondary
failover lan interface FO_LINK GigabitEthernet0/2
failover interface ip FO_LINK 172.16.255.1 255.255.255.252 standby 172.16.255.2
failover key CiscoFailoverKey123
failover link FO_LINK
failover
```

#### 検証方法
* `show failover` を実行し、ASA1 が `Active`、ASA2 が `Standby Ready` であることを確認。

---

### 例5: FTD (FXOS): Firepower 4100/9300 における Native Instance (Single) のデプロイ構成

#### 問題
Firepower 4100 シャシーの FXOS Chassis Manager を使用して、モジュール全体のすべての CPU/RAM リソースを使用する Native Instance `FTD-NATIVE-1` を作成せよ。

#### 要件
* 管理者タイプ: `FMC`

#### 設定例

```text
1. FXOS Chassis Manager: Logical Devices > Add Device.
2. Device Configuration:
   - Name: FTD-NATIVE-1
   - Template: Firepower Threat Defense
   - Deployment Type: Native Instance
   - Management Type: FMC
   - Management IP: 10.1.1.101/24, Gateway: 10.1.1.1
3. Data Interfaces を論理デバイスにバインドして保存・デプロイ。
```

#### 検証方法
* FXOS CLI にて `show app-instance` を実行し、`FTD-NATIVE-1` の `Operational Status` が `ONLINE` であることを確認。

---

### 例6: FTD: FDM (Single Device Local Management) の初期セットアップとインターフェイス構成

#### 問題
単体 FTD アプライアンスにおいて、FDM (Firepower Device Manager) によるローカル管理を有効化し、`inside` (Gi0/0: `10.0.1.254/24`) および `outside` (Gi0/1: `203.0.113.2/24`) を構成せよ。

#### 要件
* FDM Web UI より設定を行うこと。

#### 設定例

```text
1. ブラウザで https://192.168.45.45 (FTD 管理 IP) にアクセス。
2. 初期セットアップウィザード:
   - Management Type: Standalone (FDM)
   - Enable Smart Licensing Evaluation.
3. Device > Interfaces:
   - GigabitEthernet0/0: Name: inside, Mode: Routed, IPv4: 10.0.1.254/24, Enabled: On
   - GigabitEthernet0/1: Name: outside, Mode: Routed, IPv4: 203.0.113.2/24, Enabled: On
4. Save & Deploy Changes (画面右上の緑アイコン).
```

#### 検証方法
* FTD CLI にて `show managers` を実行し、`Managed by : Local (FDM)` と表示されることを確認。

---

### 例7: FTD: Local FDM 管理から FMC (Centralized Management) への管理者切り替え手続き

#### 問題
現在 FDM で管理されている単一 FTD を、中央管理アプライアンス FMC (`10.0.0.50`) 管理へ変更せよ。

#### 要件
* Registration Key: `CiscoKey123`

#### 設定例

```bash
! 1. FTD コンソール CLI に接続
> show managers
Managed by : Local (FDM)

! 2. FMC 管理者追加コマンド実行
> configure manager add 10.0.0.50 CiscoKey123

! 3. FMC GUI (10.0.0.50): Devices > Device Management > Add Device
!   - IP: FTD 管理 IP
!   - Display Name: FTD-Single-1
!   - Registration Key: CiscoKey123
!   - Access Control Policy を選択して登録実施
```

#### 検証方法
* FTD CLI にて `show managers` を実行し、`Host : 10.0.0.50` および `Status : Connected` を確認。

---

### 例8: FTD: FMC 管理下の Single FTD における データポート経由管理 (FMC Access on Data Interface) 構成

#### 問題
Single FTD1 において、専用管理ポート障害時に備え、データポート `outside` (`203.0.113.2/24`) 経由で FMC (`10.0.0.50`) からの管理アクセスを有効化せよ。

#### 要件
* FTD CLI 設定後の FMC デプロイブロックを解消すること。

#### 設定例

```bash
! 1. FTD コンソール CLI でデータポート経由の管理アクセスを有効化
> configure network ipv4 manual 192.168.1.100 255.255.255.0 data-interfaces

! 2. FMC GUI: Devices > Device Management > Interfaces > outside ポート編集
!    「Enable FMC Access」にチェックを入れて保存

! 3. FMC GUI: Devices > Device Management > Device > Management > FMC Access Details
!    表示される差分を確認し「Acknowledge」をクリック

! 4. Deploy を実行
```

#### 検証方法
* FTD CLI で `sftunnel-status-brief` を確認し、ステータスが `CONNECTED` であることを検証。

---

### 例9: FTD: Single Instance 環境での Routed Mode ＋ AnyConnect Remote Access VPN 構成

#### 問題
FMC で管理される Single FTD1 の `outside` インターフェイスにおいて AnyConnect Remote Access VPN を構成せよ。認証には ISE (RADIUS: `10.0.1.100`) を使用せよ。

#### 要件
* IP アドレスプール: `172.16.10.100 - 172.16.10.200`

#### 設定例

```text
1. FMC: Objects > Object Management > RADIUS Server Group > Add RADIUS Server Group:
   - Name: ISE_Group, Server IP: 10.0.1.100, Secret: CiscoRadiusKey
2. FMC: Objects > Object Management > Address Pools > Add Address Pool:
   - Name: VPN_Pool, Range: 172.16.10.100-172.16.10.200
3. FMC: Devices > VPN > Remote Access > Add:
   - Name: RA_VPN_FTD, Device: FTD1
   - Connection Profile: Authentication: RADIUS (ISE_Group), Pool: VPN_Pool
   - Interfaces: outside
4. Deploy を実行。
```

#### 検証方法
* クライアント端末から AnyConnect で接続し、FTD CLI で `show vpn-sessiondb remote` を確認。

---

### 例10: FTD: Single Instance 環境における BGP 動的ルーティングおよび Policy-Based Routing (PBR) の適用

#### 問題
Single FTD1 において、`outside` インターフェイスで BGP AS `65001` を動作させ、対向 ISP ルータ (`203.0.113.1`, AS `65000`) とピアリングせよ。また、内部からの特定 HTTP 通信をプロキシ (`10.0.1.200`) へ転送する PBR を `inside` インターフェイスに適用せよ。

#### 要件
* FMC GUI から設定を行うこと。

#### 設定例

```text
1. FMC: Devices > Device Management > FTD1 > Routing > BGP:
   - AS Number: 65001
   - IPv4 > Neighbor > Add: IP: 203.0.113.1, Remote AS: 65000
2. FMC: Objects > Extended Access List > Add:
   - Name: ACL_PBR_PROXY, Allow tcp any any eq 80
3. FMC: Devices > Device Management > FTD1 > Routing > Policy Based Routing > Add:
   - Ingress Interface: inside
   - Match ACL: ACL_PBR_PROXY
   - Send To: 10.0.1.200
4. Deploy を実行。
```

#### 検証方法
* FTD CLI にて `show bgp summary` で BGP ピアリング状態を確認。
* FTD CLI にて `show route-map` で PBR のヒットカウンタを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ASA1 のコンソールで以下のコマンドを実行した。この操作に関する説明として正しいものはどれか。

```bash
ciscoasa# copy disk0:old_running.cfg startup-config
ciscoasa# config t
ciscoasa(config)# mode single
```

* A) ASA1 は再起動せずに即座に Single Context モードへ切り替わる。
* B) ASA1 は再起動し、`old_running.cfg` の設定を読み込んで Single Context モードで起動する。
* C) `old_running.cfg` は自動的に削除され、工場出荷時の初期設定で起動する。
* D) モード変更が拒否され、`ERROR: Cannot change mode` と表示される。

**正解**: **B**
**解説**: `copy disk0:old_running.cfg startup-config` をあらかじめ実行してから `mode single` を適用することで、再起動後に過去の Single-Mode コンフィグがロードされた状態で復元・起動します。

---

### 2. トラブルシュート問題

**問題**: 単一の FTD アプライアンスを FDM で管理していたが、事業拡大に伴い FMC 中央管理へ変更することになった。管理者が FTD CLI で `configure manager add 10.0.0.50 Key123` を実行し、FMC への登録が成功した。しかし登録完了後、FDM で設定していた ACL や NAT ルールがすべて消失した。この問題の原因はどれか。

* A) FTD ハードウェアの Flash メモリが破損した。
* B) FDM から FMC へ管理者を切り替えると、既存のポリシー設定は消去され、FMC 側の設定で上書きされる仕様であるため。
* C) `configure manager add` コマンドの構文エラー。
* D) FMC が Evaluation ライセンスモードで動作しているため。

**正解**: **B**
**解説**: FTD の管理者を FDM から FMC（またはその逆）に変更すると、既存のポリシー設定は消去され初期化されるのが正常な仕様です。

---

### 3. Design 問題

**問題**: 企業のセキュリティ設計において、Cisco ASA 上で Remote Access AnyConnect VPN 終端および OSPF/BGP 動的ルーティングの稼働が必須要件となっている。最も適切な ASA デプロイメントモードはどれか。

* A) ASA Multi-Context Mode
* B) ASA Single Context Mode
* C) ASA Multimode Transparent Mode
* D) ASA Active/Active Failover Mode

**正解**: **B**
**解説**: Cisco ASA において、Remote Access VPN (AnyConnect) および動的ルーティングプロトコル（OSPF, BGP, EIGRP）は **Single Context Mode でのみサポート** されます。Multi-Context モードではこれらはサポートされません。

---

### 4. 実装問題

**問題**: FTD CLI にて `configure network ipv4 manual 192.168.1.100 255.255.255.0 data-interfaces` を実行し、データポート経由の FMC 管理アクセスを構成した。その後 FMC からポリシーをデプロイしようとしたところ、デプロイがブロックされた。このブロックを解消するための正しい手順はどれか。

* A) FTD を再起動する。
* B) FTD CLI で `configure manager delete` を実行して再登録する。
* C) FMC GUI の `FMC Access Details` ポップアップ画面で不一致（Discrepancy）を確認し、`Acknowledge` ボタンをクリックする。
* D) FTD の Firewall Mode を Transparent に変更する。

**正解**: **C**
**解説**: データポート管理設定を CLI で直変更すると FMC の DB と差分が生じデプロイが自動ブロックされます。FMC GUI の `FMC Access Details` で差分を認識（Acknowledge）することでブロックが解除されます。

---

### 5. コンフィグ読解問題

**問題**: FXOS Chassis Manager にて FTD アプリケーションをデプロイする際、`Native Instance` を選択した。この設定に関する正しい説明はどれか。

* A) セキュリティモジュール内に複数の FTD コンテナインスタンスを作成できる。
* B) セキュリティモジュールのすべての CPU、RAM、ディスクリソースを単一の FTD インスタンスが占有する。
* C) FDM 管理のみが許可され、FMC 管理は選択できない。
* D) クラスタリング構成が不可となる。

**正解**: **B**
**解説**: FXOS の Native Instance は、対象のセキュリティモジュール/エンジンのハードウェアリソースを 100% 占有する単一インスタンスデプロイ形態です。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Threat Defense Configuration Guide (FMC) - Device Management](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/device_management_overview.html)
* [Cisco ASA Series General Operations Configuration Guide - Security Contexts](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/general/asa-94-general-config/ha-contexts.html)
* [Cisco Technical Notes: Convert ASA Security Context Modes](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/118334-technote-asa-00.html)
* [Cisco Live: Deep Dive into Cisco Secure Firewall Architecture (BRKSEC-3020)](https://www.ciscolive.com/)
* [Cisco Validated Design (CVD): Secure Firewall Deployment Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/cisco-secure-firewall-design-guide.html)

---

## 📝 補足（Notes）

* **学習メモ**:
  * ASA の Single Context モードは「何でもできるフル機能モード」です。Multi-Context モードでの機能制限（AnyConnect や Dynamic Routing の不可）との対比を常に意識してください。
  * FTD では ASA のような「Multi-Context」概念はなく、FXOS 上の **Multi-Instance (Container Instance)** または単一の **Native Instance** に置き換わっている点を整理しておきましょう。
* **注意点**:
  * ラボ試験で FDM 運用から FMC 運用に切り替える指示があった場合、FDM 側の設定が消えるため、切替後の FMC 側でのインターフェイス・ルーティング再設定を迅速に行えるように訓練しておきましょう。

## 📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

