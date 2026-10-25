---
layout: default
title: 1.1-Deployment-modes-on-ASA-FTD
nav_order: 1
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.1 Deployment modes on Cisco ASA and Cisco FTD

Cisco ASA (Adaptive Security Appliance) および Cisco FTD (Firepower Threat Defense) は、ネットワークインフラにおける境界防御および侵入防止の中核を担うセキュリティデバイスです。組織のネットワーク構成、インスペクション要件、およびパフォーマンス要求に応じて、さまざまな「デプロイメントモード（展開モード）」および「インターフェイスモード」を選択できます。

---

## 📘 概要

* **機能概要**:
  * **ASA / FTD デバイスモード（Firewall Mode）**: デバイス全体がネットワーク上でどのように振る舞うかを定義します（L3 ールーティング動作を行う **Routed Mode** と、L2 スイッチ/ブリッジ動作を行う **Transparent Mode**）。
  * **ASA 仮想化モード（Context Mode）**: 1 台の物理 ASA を複数の論理ファイアウォールに分割する **Single Context Mode** と **Multi-Context Mode**。
  * **FTD マルチインスタンス（Multi-Instance）**: Firepower 4100/9300 ハードウェアシャシー上で、FXOS (Firepower eXtensible Operating System) を介して複数の独立した FTD コンテナインスタンスを実行する機能。
  * **FTD インターフェイスモード（Interface Modes）**: データプレーンでパケットをどのように受信・処理・転送するかをポート単位で制御します（**Routed**, **Switched (BVI)**, **Inline Pair**, **Inline Pair with Tap**, **Passive**, **Passive ERSPAN**）。
* **利用目的**:
  * **インライン制御**: トラフィックをインラインで通過させながら、L2-L7 ポリシー適用やマルウェア/IPS 遮断を行う。
  * **既存ネットワークへの透過導入**: IP サブネット設計を変更せずにセキュリティスキャンを挿入する。
  * **非侵入型（Out-of-band）監視**: SPAN や ERSPAN を利用して、既存トラフィックに一切影響を与えずに不正侵入を検知（IDS モード）する。
  * **マルチテナント化**: 複数の部門や顧客（テナント）に対して、独立したファイアウォール環境や設定管理を提供する。
* **どのような場面で利用するか**:
  * **Routed Mode**: 企業ネットーワーク境界、DMZ セグメンテーション、Remote Access / Site-to-Site VPN 終端。
  * **Transparent Mode**: データセンター内の同一 VLAN 内セキュリティ検査、IP アドレス変更が困難な既設ネットワークへの後付け配置。
  * **Inline Pair / Tap Mode**: 透過的な NGIPS / NGFW 導入。Tap モードは新ポリシーの誤検知・影響評価期間に活用。
  * **Passive / ERSPAN Mode**: ネットワーク監視、セキュリティオペレーションセンター (SOC) 向けのキャプチャ分析。
  * **Multi-Context (ASA) / Multi-Instance (FTD)**: サービスプロバイダー環境、大規模エンタープライズの部門別独立管理。

---

## 🔑 要点

| 項目 | Routed Mode | Transparent Mode | Multi-Context (ASA) | Multi-Instance (FTD) | Inline Pair / Tap (FTD) | Passive / ERSPAN (FTD) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **OS 適用** | ASA / FTD | ASA / FTD | ASA のみ | FTD (4100/9300) | FTD のみ | FTD のみ |
| **L2/L3 動作** | L3 (ルータホップ) | L2 (Bump in the wire) | L2 または L3 (全 Ctx 同一) | モード選択可 (各 Inst 独立) | L2 インライン透過 | プロミスキャス監視 |
| **IP アドレス割り当て** | 各インターフェイスに個別 IP | BVI に管理用 IP (同一サブネット) | 各 Context に IP 割り当て | 各 Instance に IP 割り当て | 不要 (Network Agnostic) | 不要 (ERSPAN は受信側 IP 要) |
| **主な用途** | 境界防御、ルーティング、VPN | L2 インスペクション、挿入 | テナント分離、A/A HA | リソースハード分離、独立 OS | 透過型 NGIPS / Block | IDS モード、トラフィック分析 |
| **ルーティングプロトコル** | OSPF, BGP, EIGRP, RIP | 透過 (ACL で許可時) | ステイックルーティングのみ | 各 Inst で完全サポート | 非サポート | 非サポート |
| **VPN サポート** | 完全サポート (IPsec/SSL) | 非サポート | 非サポート (Site-to-Site は限定) | 完全サポート | 非サポート | 非サポート |
| **NAT サポート** | 完全サポート | サポート (Bridge グループ間) | 完全サポート | 完全サポート | 非サポート | 非サポート |
| **パケット遮断能力** | 可能 | 可能 | 可能 | 可能 | Inline: 可能 / Tap: 不可 | 不可 |

---

## 🏗 動作原理

### 1. Routed Mode 通信フロー
デバイスは L3 ルータホップとして動作します。送信元から宛先へのルーティングテーブル検索およびステートフルインスペクション（LINA + Snort）を行います。

```text
[ Client: 10.1.1.100 ]
         ↓
(Ingress: GigabitEthernet0/0 - Inside: 10.1.1.1)
         ↓
  [ LINA Engine: Stateful Session Lookup / NAT / Access Control ]
         ↓
  [ Snort 3 Engine: IPS / AVC / URL / File Inspection ]
         ↓
  [ L3 Routing Table Lookup (Egress Selection) ]
         ↓
(Egress: GigabitEthernet0/1 - Outside: 10.1.2.1)
         ↓
[ Router / Server: 10.1.2.100 ]
```

### 2. Transparent Mode (Bridge Virtual Interface: BVI) 通信フロー
デバイスは L2 ブリッジとして動作します。同一サブネットの 2 つ以上の物理インターフェイスを Bridge Group にまとめ、BVI (Bridge Virtual Interface) を作成します。

```text
[ Client: 10.1.1.100 ]
         ↓
(Ingress Port: GigabitEthernet0/0 - Bridge Group 1)
         ↓
  [ L2 MAC Address Table Lookup ]
         ↓
  [ LINA Engine: ACL Check / Stateful Inspection ]
         ↓
  [ Snort Engine: IPS / Content Filtering ]
         ↓
(Egress Port: GigabitEthernet0/1 - Bridge Group 1)
         ↓
[ Server: 10.1.1.200 (Same Subnet) ]
  * BVI1 (10.1.1.1) はデバイス自身の管理・SYSLOG/AAA 通信用の L3 アドレスとして機能
```

### 3. FTD Inline Pair / Inline Tap モード通信フロー
インラインペアは 2 つの物理ポートを直結し、L2/L3 の処理を最小限に抑えてパケットを通します。Tap モードではパケットのコピーを Snort に送るため、実際の通信はドロップされません。

```text
--- [ Inline Pair Mode ] ---
Ingress Port (eth1) ---> [ Snort Engine: Inspect & Block/Drop ] ---> Egress Port (eth2)
                                                                 (Live Traffic Blocked)

--- [ Inline Pair with Tap Mode ] ---
Ingress Port (eth1) ----------------------------------------------> Egress Port (eth2)
        │                                                         (Traffic Always Passed)
        └─── (Copy Packet) ---> [ Snort Engine: Inspect & Alert ]
```

### 4. FTD Passive / ERSPAN Mode 通信フロー
スイッチの SPAN ポートまたは GRE カプセル化された ERSPAN トラフィックを受信し、パケットを検査します。

```text
[ Core Switch ] --(SPAN / ERSPAN GRE Port)--> (Ingress: Passive Port) ---> [ Snort Engine ]
                                                                             (Alert Only)
```

---

## ⚙ 動作シーケンス

FTD における内部パケット処理シーケンスは、ASA のステートフルアクセス制御エンジン（**LINA**）と、Sourcefire のディープパケットインスペクションエンジン（**Snort**）の連携によって実行されます。

1. **Ingress パケット受信**: パケットが物理/論理インターフェイスに入力されます。
2. **LINA Pre-filter Policy / DAQ (Data Acquisition Layer)**:
   * プリフィルタールールに合致した場合、Fastpath（Snort バイパス）または Drop、あるいは Analyze（Snort 転送）が決定されます。
3. **LINA Stateful Check & Connection Lookup**:
   * 既存セクションの確認。新規セクションの場合、接続表（Conn Table）の作成。
   * Security Level / ACL / NAT 処理の適用。
4. **Snort インスペクション（Analyze の場合）**:
   * パケットが共有メモリを介して Snort 3 (または Snort 2) プロセスへ渡されます。
   * **Security Intelligence (SI)**: IP / Domain / URL のブラックリスト照会。
   * **SSL/TLS Decryption**: SSL Policy による復号処理。
   * **Access Control Policy (ACP)**: アプリケーション識別 (AVC), URL フィルタリング。
   * **Intrusion Policy (IPS)**: シグネチャ照合。
   * **File Policy / AMP**: ファイル抽出、SHA-256 ハッシュ照会、サンドボックス送信。
5. **Egress 処理と転送**:
   * ポリシーでブロックされなかった場合、パケットは LINA に戻され、Egress インターフェイスから送出されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### 重要なポイントと設定の注意点

1. **Firewall Mode 変更時の全設定破棄**:
   * CLI で `configure firewall transparent` または `configure firewall routed` を実行すると、**既存のすべてのインターフェイス設定・ポリシーが破棄**されます。
   * FTD の場合、Mode を変更するには一度 **FMC から登録解除（Deregister）** し、CLI で変更後に再登録する必要があります。

2. **FTD における Multi-Context 非サポートと Multi-Instance**:
   * FTD は ASA のような **Multi-Context モードをサポートしていません**。
   * ラボ試験で FTD のマルチテナント化が要求された場合、Firepower 4100/9300 ハードウェア上の **FXOS Multi-Instance (Container Instance)** を使用します。

3. **Inline Pair における制限事項**:
   * Inline Pair / Inline Set では、通常ダイナミックルーティングプロトコル、NAT、VPN 終端は機能しません（パケットは Bump in the wire として処理されるため）。
   * 非同期ルーティング（Asynchronous Routing）環境では、往路と復路のインラインペアを同一の **Inline Set** に含める必要があります。同一 Inline Set に含めないと Snort が片方向の通信しか見えず、セッションが不整合を起こします。

4. **ERSPAN インターフェイスの要件**:
   * FTD の Passive ERSPAN インターフェイスは、**Routed Firewall Mode でのみサポート**されます（Transparent Mode では設定不可）。

5. **FMC Access Data Interface の齟齬とデプロイブロック**:
   * FTD CLI で `configure network management-data-interface` を使ってデータポート管理を有効化した場合、FMC 側の設定と不一致が生じてデプロイがブロックされます。FMC 画面で **Acknowledge** を押して不一致を解消する必要があります。

---

## 🛠 設定方法

### 1. Cisco ASA: Routed Mode から Transparent Mode への変更 (CLI)

```bash
! 現在のモード確認
ciscoasa# show firewall
Firewall mode: Routed

! Transparent Mode への変更 (※設定が全消去されます)
ciscoasa# config t
ciscoasa(config)# firewall transparent
WARNING: Changing firewall mode clears all interface configurations.
Are you sure you want to proceed? [confirm] <Enter>

! Bridge Group および BVI の設定
ciscoasa(config)# interface GigabitEthernet0/0
ciscoasa(config-if)# nameif inside
ciscoasa(config-if)# bridge-group 1
ciscoasa(config-if)# no shutdown
!
ciscoasa(config)# interface GigabitEthernet0/1
ciscoasa(config-if)# nameif outside
ciscoasa(config-if)# bridge-group 1
ciscoasa(config-if)# no shutdown
!
ciscoasa(config)# interface interface BVI1
ciscoasa(config-if)# ip address 10.1.1.254 255.255.255.0
ciscoasa(config-if)# nameif management
```

### 2. Cisco ASA: Multi-Context モードの構成 (CLI)

```bash
! Multi-Context モードの有効化 (要再起動)
ciscoasa# mode multiple
WARNING: This command will change the mode of the appliance...
Reboot now? [confirm] <Enter>

! 再起動後、System Context にて Admin および Data Context を定義
ciscoasa# show mode
Security context mode: multiple

ciscoasa(config)# context admin
ciscoasa(config-ctx)# allocate-interface GigabitEthernet0/0
ciscoasa(config-ctx)# config-url disk0:/admin.cfg
!
ciscoasa(config)# context CTX-A
ciscoasa(config-ctx)# allocate-interface GigabitEthernet0/1
ciscoasa(config-ctx)# allocate-interface GigabitEthernet0/2
ciscoasa(config-ctx)# config-url disk0:/CTX-A.cfg
!
! Context への切り替えと設定
ciscoasa# changeto context CTX-A
ciscoasa/CTX-A# config t
ciscoasa/CTX-A(config)# interface GigabitEthernet0/1
ciscoasa/CTX-A(config-if)# nameif inside
ciscoasa/CTX-A(config-if)# security-level 100
ciscoasa/CTX-A(config-if)# ip address 10.10.1.1 255.255.255.0
```

### 3. Cisco FTD: CLI による Firewall Mode 変更手続き

```bash
! 1. FTD CLI で FMC 登録解除 (事前に FMC GUI 上で Device 削除を推奨)
> show managers
Managed by : 10.1.1.50
> configure manager delete

! 2. Firewall Mode の変更 (既存設定が消去されます)
> configure firewall transparent
This will destroy the current interface configurations, are you sure that you want to proceed? [y/N] y
The firewall mode was changed successfully.

! 3. FMC への再登録
> configure manager add 10.1.1.50 SecretKey123
```

### 4. Cisco FTD (FMC GUI): Inline Set および Inline Tap の構成

1. **Inline Set の作成**:
   * **Devices > Device Management** に移動し、対象 FTD の編集アイコンをクリック。
   * **Inline Sets** タブを選択し、**Add Inline Set** をクリック。
   * **Name**: `INLINE-SET-1` を入力。
   * **Advanced** タブで **Tap Mode** をチェック（※Tap モードにする場合）。
   * **Propagate Link State**: チェック（障害発生時にペアポートを連動ダウン）。
2. **Interface Pair のバインド**:
   * **Add Pairs** をクリックし、`GigabitEthernet0/2` と `GigabitEthernet0/3` をペアとして指定。
3. **Security Zone の割り当て**:
   * インラインペアに対して `Inline-Zone` などの Security Zone を指定。
4. **Access Control Policy のバインドと Deploy**:
   * ACP の Source Zone / Destination Zone に `Inline-Zone` を指定し、FTD へ Deploy。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **ファイアウォールモード確認** | ASA / FTD CLI | <code>show firewall</code> |
| **Context モード確認** | ASA CLI | <code>show mode</code> |
| **Context 一覧・状態確認** | ASA CLI (System) | <code>show context</code> |
| **FMC 登録状態の確認** | FTD CLI | <code>show managers</code> |
| **BVI インターフェイス状態** | ASA / FTD CLI | <code>show interface bvi 1</code> |
| **MAC アドレス表確認 (Transparent)** | ASA / FTD CLI | <code>show mac-address</code> |
| **Inline Set 構成・状態確認** | FTD CLI | <code>show inline-set</code> |
| **インターフェイス詳細・リンク状態** | FTD CLI | <code>show interface detail</code> |
| **パケットフロー追跡・シミュレーション** | FTD CLI | <code>packet-tracer input inside tcp 10.1.1.10 12345 10.1.2.10 80</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **FMC デプロイがブロックされる** | FTD CLI での管理 IP 直変更による不一致 | FTD: <code>show network</code><br>FMC: GUI 警告バナー | FMC の `FMC Access Details` で **Acknowledge** を実行し、FMC 設定を修復後に Deploy。 |
| **Transparent モードで通信不能** | BVI 未構成、または同一サブネット違反 | ASA/FTD: <code>show interface bvi</code> | BVI に同一サブネット内の有効な IP を設定し、メンバーポートを正しい Bridge Group に紐付ける。 |
| **Inline Pair で片方向しか検知しない** | 非同期通信でインラインペアが別々 | FTD: <code>show inline-set</code> | 往路と復路のインラインペアを**同一の Inline Set** に登録する。 |
| **Tap モードで通信がブロックされない** | Tap モードが有効（コピーをスキャン） | FTD: <code>show inline-set</code> | FMC の Inline Set 設定で **Tap Mode** のチェックを外し、インラインドロップを有効化。 |
| **ERSPAN パケットが受信されない** | モード不整合（Transparent モード） | FTD: <code>show firewall</code> | ERSPAN は **Routed Firewall Mode でのみサポート**されるため、モードを確認・変更する。 |

---

## ⚠ 制限事項

* **FTD の Multi-Context 未サポート**: FTD ソフトウェア自体は ASA 形式の Multi-Context モードをサポートしていません。
* **Transparent Mode の機能制限**:
  * Dynamic Routing Protocols (OSPF, BGP 等) は透過しますが、デバイス自体で動的ルーティング処理は行えません。
  * Remote Access VPN (AnyConnect) および IPsec Site-to-Site VPN 終端はサポートされません。
* **Inline Pair の機能制限**:
  * NAT、ルーティング、VPN などの主要な ASA 伝統機能は Inline Pair 通信には適用されません。
  * Redundant Interface や VLAN Subinterface は Inline Pair のメンバーに指定できません。
* **ERSPAN の制限**:
  * ERSPAN インターフェイスは Routed Firewall Mode でのみ作成可能です。

---

## 🔄 他技術との関連

* **1.2 Access Control & Pre-filter Policies**: FTD のデプロイメントモードおよび Security Zone に基づいて ACP ルールが適用されます。
* **1.3 High Availability & Clustering**:
  * ASA Multi-Context モードでは **Active/Active Failover** が可能。
  * FTD Multi-Instance ではシャシー間 HA やクラスタリングの要件が異なります。
* **3.1 NAT**: Routed モードおよび Transparent モード（Bridge Group 間）で動作。Inline Pair では動作しません。
* **Snort 3 Engine**: インターフェイスモード（Inline, Tap, Passive）に応じて、Snort がパケットをドロップするかアラートのみを出すかが変化します。

---

## 🧩 比較表

### 1. Routed vs Transparent Mode

| 項目 | Routed Mode | Transparent Mode |
| :--- | :--- | :--- |
| **ネットワーク配置** | L3 ルータホップ | L2 ブリッジ (Bump in the wire) |
| **サブネット要件** | 各ポートが異なる L3 サブネット | 全ポートおよび BVI が同一 L3 サブネット |
| **ルーティング機能** | OSPF, BGP, EIGRP, Static | ルーティング不可 (BVI による管理のみ) |
| **VPN 終端** | IPsec / Remote Access (AnyConnect) | 非サポート |
| **橋渡しトラフィック** | L3 IP パケットのみ | L2 報知 (ARP, BPDU, IPX 等 ACL で制御可) |

### 2. ASA Multi-Context vs FTD Multi-Instance

| 項目 | ASA Multi-Context | FTD Multi-Instance |
| :--- | :--- | :--- |
| **基盤アーキテクチャ** | LINA ソフトウェアの論理分割 | FXOS 上の Docker/Container インスタンス |
| **リソース分離** | 共有 CPU / メモリ (Quota による制限) | **ハードウェア CPU / メモリの完全固定割り当て** |
| **OS バージョン** | 全 Context で同一 OS バージョン | **各 Instance で異なる OS バージョンが動作可能** |
| **障害・リロード** | 一部の障害が全体に影響する可能性あり | **個別の Instance が完全独立してリロード可能** |
| **対応プラットフォーム** | ASA 5500-X シリーズ等 | Firepower 4100 / 9300 シリーズのみ |

---

## 💡 ベストプラクティス

1. **新規導入時のモード選定**:
   * ネットワーク構成変更が許容できる場合は、完全な L3-L7 制御および VPN 対応が可能な **Routed Mode** を第一選択とします。
2. **Inline Tap モードの段階的運用**:
   * 新規に IPS を導入する際、まずは **Inline Tap Mode** で数週間運用し、誤検知（False Positive）のシグネチャを特定・チューニングしてから、**Inline Mode (Drop)** へ切り替えます。
3. **Link State Propagation の有効化**:
   * Inline Set を使用する際は、必ず **Propagate Link State** を有効にします。これにより、一方のリンク障害時に他方もダウンさせ、アップストリームのルータ/スイッチが高速に迂回路（ECMP / STP / HSRP）へ切替を行えるようにします。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA Single-Mode Routed から Transparent Mode への変更と BVI 設定

#### 問題
ASA1 を Routed Mode から Transparent Mode に変更し、Inside (Gi0/0) と Outside (Gi0/1) を Bridge Group 1 にバインドせよ。管理用 IP アドレスとして BVI1 に `192.168.1.254/24` を割り当てよ。

#### 要件
* モード変更時の設定消去に注意すること。
* インターフェイス名を `inside`, `outside` と定義すること。

#### 設定例

```bash
! モード変更
firewall transparent

! インターフェイスおよび Bridge Group 構成
interface GigabitEthernet0/0
 nameif inside
 bridge-group 1
 no shutdown
!
interface GigabitEthernet0/1
 nameif outside
 bridge-group 1
 no shutdown
!
interface BVI1
 nameif mgmt
 ip address 192.168.1.254 255.255.255.0
 no shutdown
```

#### 検証方法
* `show firewall` を実行し `Transparent` であることを確認。
* `show interface bvi 1` で IP アドレスとステータスを確認。
* 同一サブネット内の端末から `192.168.1.254` への ping 通信を確認。

---

### 例2: ASA Multi-Context モードの有効化と Admin/Data Context の構成

#### 問題
ASA2 で Multi-Context モードを有効化し、`admin` コンテキスト（Gi0/0 割当）と `CTX-FINANCE` コンテキスト（Gi0/1, Gi0/2 割当）を作成せよ。

#### 要件
* 設定ファイルは `disk0:/` に保存すること。

#### 設定例

```bash
! Multi-Context モードへの変更
mode multiple

! システム設定 (自動リブート後)
context admin
 allocate-interface GigabitEthernet0/0
 config-url disk0:/admin.cfg
!
context CTX-FINANCE
 allocate-interface GigabitEthernet0/1
 allocate-interface GigabitEthernet0/2
 config-url disk0:/CTX-FINANCE.cfg
!
! Data Context 内の設定
changeto context CTX-FINANCE
config t
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 10.10.10.1 255.255.255.0
!
interface GigabitEthernet0/2
 nameif outside
 security-level 0
 ip address 172.16.10.1 255.255.255.0
```

#### 検証方法
* `show mode` で `multiple` を確認。
* `show context` で各 Context の状態と割り当てポートを確認。

---

### 例3: ASA Multi-Context モードでの Active/Active Failover 構成

#### 問題
2 台の ASA (ASA1, ASA2) 上で `CTX1` (グループ 1: ASA1 が Active) と `CTX2` (グループ 2: ASA2 が Active) を構成し、Active/Active Failover を実現せよ。

#### 要件
* LAN ベース Failover インターフェイスとして GigabitEthernet0/3 を使用すること。

#### 設定例

```bash
! [ASA1 - System Context]
failover
failover lan unit primary
failover lan interface FO_LINK GigabitEthernet0/3
failover interface ip FO_LINK 192.168.254.1 255.255.255.0 standby 192.168.254.2
failover link FO_LINK
!
failover group 1
 primary
failover group 2
 secondary
!
context CTX1
 allocate-interface GigabitEthernet0/0
 config-url disk0:/ctx1.cfg
 join-failover-group 1
!
context CTX2
 allocate-interface GigabitEthernet0/1
 config-url disk0:/ctx2.cfg
 join-failover-group 2

! [ASA2 - System Context]
failover
failover lan unit secondary
failover lan interface FO_LINK GigabitEthernet0/3
failover interface ip FO_LINK 192.168.254.1 255.255.255.0 standby 192.168.254.2
failover link FO_LINK
```

#### 検証方法
* `show failover` を実行し、グループ 1 とグループ 2 の Primary/Secondary アクティブ状態を確認。

---

### 例4: FTD CLI による Firewall Mode の Routed から Transparent への変更手続き

#### 問題
FTD1 を FMC から一時的に切り離し、CLI を用いて Routed Mode から Transparent Mode に変更後、再度 FMC に登録せよ。

#### 要件
* FMC の IP: `10.1.1.50`, Registration Key: `Cisco123`

#### 設定例

```bash
! 1. FTD CLI でのマネージャ削除
> configure manager delete

! 2. ファイアウォールモードの透明化変更
> configure firewall transparent
This will destroy the current interface configurations, are you sure that you want to proceed? [y/N] y

! 3. 再登録プロセスの実行
> configure manager add 10.1.1.50 Cisco123
```

#### 検証方法
* `show firewall` で `Transparent` を確認。
* `show managers` で FMC への接続ステータスを確認。

---

### 例5: FTD (FMC) Transparent Mode での Bridge Group (BVI) と Access Control Policy 適用

#### 問題
FMC を使用して、Transparent Mode の FTD 上で `GigabitEthernet0/0` と `Gi0/1` を Bridge Group 10 に設定し、BVI10 に `10.0.0.254/24` を割り当てよ。

#### 要件
* FMC GUI から操作を行うこと。

#### 設定例

```text
1. FMC: Devices > Device Management > Interfaces に移動。
2. Add Bridge Group を選択。
   - Bridge Group ID: 10
   - Name: BVI10
   - IP Address: 10.0.0.254 /24
   - Member Interfaces: GigabitEthernet0/0 (inside), GigabitEthernet0/1 (outside)
3. Access Control Policy を作成・編集:
   - Rule 1: Source Zone (inside-zone) -> Destination Zone (outside-zone) ALLOW
4. FTD へ Deploy を実行。
```

#### 検証方法
* FTD CLI にて `show bridge-group` および `show interface bvi 10` を実行。

---

### 例6: FTD (FMC) Inline Set (Inline Pair) の作成と Access Control Policy のバインド

#### 問題
FTD 上の物理ポート `Gi0/2` と `Gi0/3` をペアとした Inline Set `IPS-INLINE` を構成せよ。

#### 要件
* インラインドロップを有効にし、直結されたトラフィックを IPS 検査せよ。

#### 設定例

```text
1. FMC: Devices > Device Management > Inline Sets タブ。
2. Add Inline Set をクリック:
   - Name: IPS-INLINE
   - Snort Fail Open: Disabled
   - Propagate Link State: Enabled
   - Tap Mode: Disabled (Unchecked)
3. Add Pair:
   - First Interface: GigabitEthernet0/2
   - Second Interface: GigabitEthernet0/3
4. Interfaces タブで両インターフェイスの Security Zone に "Inline-Security-Zone" を設定。
5. ACP ルールで Inline-Security-Zone 間の通信に IPS Policy をバインドして Deploy。
```

#### 検証方法
* FTD CLI にて `show inline-set` を実行し、ステータスが `UP` かつ `Tap mode is off` であることを確認。

---

### 例7: FTD (FMC) Inline Tap Mode の構成と検証

#### 問題
既存の Inline Set `IPS-INLINE` を一元的な設定変更により `Inline Tap Mode` に変更し、トラフィックを遮断せずにスキャンのみ行うように設定せよ。

#### 要件
* 配線を変更せずに設定のみで変更すること。

#### 設定例

```text
1. FMC: Devices > Device Management > Inline Sets タブ。
2. "IPS-INLINE" を編集。
3. Advanced タブの "Tap Mode" にチェックを入れる。
4. 設定を保存し、FTD へ Deploy。
```

#### 検証方法
* FTD CLI にて `show inline-set` を実行し、`Tap mode is on` と表示されることを確認。
* 攻撃トラフィックを発生させ、イベントログに Dropped 表示ではなく Alert のみが生成されることを確認。

---

### 例8: FTD (FMC) Passive Mode (SPAN/Mirror) インターフェイスの構成

#### 問題
FTD の `GigabitEthernet0/4` を Passive Mode インターフェイスとして構成し、スイッチの SPAN ポートからのトラフィックを監視せよ。

#### 要件
* パケットの送信・応答を行わないプロミスキャススキャンとすること。

#### 設定例

```text
1. FMC: Devices > Device Management > Interfaces.
2. GigabitEthernet0/4 を編集:
   - Mode: Passive
   - Security Zone: Passive-Zone
   - Enabled: Checked
3. Access Control Policy を作成:
   - Source Zone: Passive-Zone のトラフィックに対して IPS スキャンを適用。
4. Deploy を実行。
```

#### 検証方法
* FTD CLI で `show interface GigabitEthernet0/4` を実行し、入力パケットカウンタのみが増加し、出力パケットが 0 であることを確認。

---

### 例9: FTD (FMC) Routed Mode での ERSPAN Passive インターフェイス構成

#### 問題
Routed Mode の FTD において、`GigabitEthernet0/5` を ERSPAN ソースからの GRE トラフィックを受信する Passive ERSPAN ポートとして設定せよ。

#### 要件
* ERSPAN ID: `100`

#### 設定例

```text
1. FMC: Devices > Device Management > Interfaces.
2. GigabitEthernet0/5 を編集:
   - Mode: ERSPAN
   - ERSPAN ID: 100
   - Security Zone: ERSPAN-Zone
3. Deploy を実行。
```

#### 検証方法
* FTD CLI で `show interface detail` を確認し、ERSPAN デカプセル化が機能していることを検証。

---

### 例10: FTD (FXOS) Firepower 4100/9300 における Multi-Instance (Container Instance) 構成

#### 問題
Firepower 4100 シャシー上で、FXOS Chassis Manager を使用して 2 つの独立した FTD コンテナインスタンス (`FTD-INSTANCE-A`, `FTD-INSTANCE-B`) を作成せよ。

#### 要件
* それぞれに CPU コアとメモリを固定割り当てすること。

#### 設定例

```text
1. FXOS Chassis Manager: Logical Devices > Add Device.
2. Instance 1:
   - Name: FTD-INSTANCE-A
   - Template: Firepower Threat Defense
   - Type: Container
   - Resource Profile: CPU 10 Cores, Memory 32GB
   - Management IP: 10.1.1.101/24
3. Instance 2:
   - Name: FTD-INSTANCE-B
   - Template: Firepower Threat Defense
   - Type: Container
   - Resource Profile: CPU 10 Cores, Memory 32GB
   - Management IP: 10.1.1.102/24
4. 各インスタンスに対し独立して FMC 登録手続きを実施。
```

#### 検証方法
* FXOS CLI にて `show app-instance` を実行し、両インスタンスが `ONLINE` であることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題
**問題**: ASA の設定で以下のコマンドが出力されている。この ASA の展開モードおよびトラフィック転送挙動として正しい説明はどれか。

```text
firewall transparent
!
interface GigabitEthernet0/0
 nameif inside
 bridge-group 5
!
interface GigabitEthernet0/1
 nameif outside
 bridge-group 5
!
interface BVI5
 ip address 10.20.30.254 255.255.255.0
```

* A) Routed Mode で動作しており、BVI5 が Inside と Outside 間のデフォルトゲートウェイとしてルーティングを行う。
* B) Transparent Mode で動作しており、Gi0/0 と Gi0/1 間は L2 ブリッジングされ、BVI5 の IP は管理通信等に使用される。
* C) Inline Pair モードで動作しており、LINA エンジンを完全にバイパスして Snort のみがスキャンを行う。
* D) Multi-Context モードの Admin Context としてのみ動作する。

**正解**: **B**
**解説**: `firewall transparent` により Transparent Mode となっており、同一 `bridge-group 5` に属するポート間は L2 ブリッジ処理されます。BVI5 の IP はデバイス自身の管理・SYSLOG・AAA 通信用です。

---

### 2. トラブルシュート問題
**問題**: 管理者が FTD CLI 上で `configure network management-data-interface` コマンドを実行し、データポート経由での FMC 管理アクセスを有効化した。しかし、その後 FMC からポリシーをデプロイしようとすると「Deployment Blocked」エラーが発生した。この問題を解決するための正しい手順はどれか。

* A) FTD CLI で `configure manager delete` を行い、再登録する。
* B) FTD を再起動して CLI の変更をリセットする。
* C) FMC GUI の `FMC Access Details` 画面を開き、設定の不一致を確認した上で `Acknowledge` をクリックし、FMC 側のインターフェイス設定を再調整してデプロイする。
* D) FTD CLI で `configure firewall routed` を実行してモードを初期化する。

**正解**: **C**
**解説**: CLI でデータポート管理設定を直接変更すると FMC 側の DB と不一致が生じ、誤設定上書きを防ぐためにデプロイがブロックされます。FMC GUI で不一致を認識（Acknowledge）し、FMC 側の設定を同期させる必要があります。

---

### 3. Design 問題
**問題**: ある企業で既存のデータセンターネットワークに NGIPS 機能を後付けで導入することになった。ネットワークの IP サブネット設計やルータの隣接関係変更は一切許されず、また万が一セキュリティアプライアンスに障害が発生した場合でも、物理リンクダウンを対向スイッチに伝播させて高速にネットワーク迂回を発生させたい。最も適切な構成はどれか。

* A) Routed Mode の FTD をアクティブ/スタンバイ HA で配置する。
* B) Inline Set を構成した FTD を使用し、`Propagate Link State` オプションを有効化する。
* C) Passive Mode (SPAN) の FTD を配置し、スイッチ側で HSRP を調整する。
* D) Multi-Context モードの ASA を Transparent モードで配置する。

**正解**: **B**
**解説**: IP サブネットを変更しないインライン挿入には Inline Set (Inline Pair) が適しています。さらに `Propagate Link State` を有効にすることで、一方のポートダウン時に反対側のポートも落とし、対向機器のアップストリーム迂回をトリガーできます。

---

### 4. 実装問題
**問題**: FTD において ERSPAN トラフィックを受信・分析するための Passive ERSPAN インターフェイスを構成したいが、インターフェイス追加画面で ERSPAN モードが選択肢に表示されない。考えられる原因はどれか。

* A) FTD が Single-Mode ではなく Multi-Context モードで動作しているため。
* B) FTD の Firewall Mode が Transparent Mode に設定されているため。
* C) FTD に Threat ライセンスが適用されていないため。
* D) インターフェイスが EtherChannel に組み込まれているため。

**正解**: **B**
**解説**: FTD の Passive ERSPAN インターフェイス機能は、**Routed Firewall Mode でのみサポート**されています。Transparent Mode では選択できません。

---

### 5. コンフィグ読解問題
**問題**: FTD CLI にて `show inline-set` を実行したところ、以下の出力が得られた。現在のこのデバイスの動作状態として正しいものはどれか。

```text
> show inline-set
Inline-set INSIDE_OUTSIDE_PAIR
  Mtu is 1500 bytes
  Fail-open for snort down is off
  Tap mode is on
  Propagate-link-state option is on
  Interface-Pair:
    Interface: GigabitEthernet0/0 "INSIDE"
    Interface: GigabitEthernet0/1 "OUTSIDE"
```

* A) パケットはインラインで通過し、悪意のあるパケットはドロップされる。
* B) トラフィックのコピーが Snort に送られて検知・アラート生成が行われるが、実際の通信パケットはドロップされず通過する。
* C) インラインペアの片方のリンクがダウンしても、もう片方のリンクは UP 状態を維持する。
* D) Snort プロセスが停止した場合、パケットは無検査でそのまま通過（Fail-Open）する。

**正解**: **B**
**解説**: `Tap mode is on` であるため、インラインタップモードで動作しています。トラフィックのコピーを検査してイベントを生成しますが、実トラフィックをドロップすることはありません。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Threat Defense Configuration Guide (FMC)](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70.html)
* [Cisco ASA Series Configuration Guide - Transparent / Routed Mode](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/general/asa-94-general-config.html)
* [Cisco Live: Deep Dive into Cisco Secure Firewall Architecture (BRKSEC-3020)](https://www.ciscolive.com/)
* [CVD: Cisco Secure Firewall Deployment Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/cisco-secure-firewall-design-guide.html)

---

## 📝 補足（Notes）

* **学習メモ**:
  * ASA / FTD のデプロイメントモード選定は、セキュリティ設計の最初の分岐点です。「IP 設計を変更できるか？」「VPN 終端が必要か？」「インライン遮断か監視のみか？」の 3 つの質問でモードを特定できます。
  * **"Mode" の用語の整理**:
    * **Firewall Mode**: Routed vs Transparent (デバイス全体)
    * **Context Mode (ASA)**: Single vs Multi-Context
    * **Multi-Instance (FTD)**: Native vs Container (FXOS)
    * **Interface Mode (FTD)**: Routed, Switched (BVI), Inline Pair, Inline Tap, Passive, ERSPAN
* **注意点**:
  * ラボ試験のCLI操作で `configure firewall transparent` や `mode multiple` を実行する際は、**設定が全消去されたり再起動が発生したりする**ため、試験の初期段階で計画的に実行してください。
