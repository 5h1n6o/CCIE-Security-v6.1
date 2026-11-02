---
layout: default
title: 2.5.b-PVLAN
nav_order: 2
parent: 2.5-Infrastructure-segmentation
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.5.b PVLAN

Private VLAN（PVLAN）は、Layer 2（データリンク層）レベルにおいて同一サブネット内（同一VLAN内）のホスト間通信を分離・制限し、IPアドレス資源を節約しながら高度なマイクロセグメンテーション（Micro-segmentation）を実現するセキュリティ技術です。

CCIE Security v6.1 Blueprint **2.5.b PVLAN** では、Primary VLAN（プライマリVLAN）と Secondary VLAN（セカンダリVLAN: Isolated / Community）の包含関係、Promiscuous Port（無差別ポート）、Isolated Port（孤立ポート）、Community Port（コミュニティポート）、PVLAN Trunk Port（PVLAN トラフィックを別スイッチへ伝送するトランク）の動作原理、SVI（Switched Virtual Interface）へのマッピング、ならびに DHCP Snooping、Dynamic ARP Inspection (DAI)、VACL (VLAN ACL)、Cisco ASA / FTD への透過接続やファイアウォールとの連携設計に至るまで、実効性の高い設計・設定・トラブルシューティング能力が深く問われます。

---

## 📘 概要

* **機能概要**:
  * **Private VLAN (PVLAN)** は、1つの IP サブネット（Primary VLAN）を内部的に複数の物理的・論理的ドメイン（Secondary VLAN）に分割・構造化する L2 分割技術です。
  * **Primary VLAN**: 外部ネットワーク（ルーター、ファイアウォール、デフォルトゲートウェイ）や上位共有リソースと通信するためのメインVLAN。
  * **Secondary VLAN (Isolated VLAN)**: 同一 Isolated VLAN に所属するホスト同士の Layer 2 通信を **完全に遮断** する VLAN。通信は Promiscuous Port（デフォルトゲートウェイなど）宛てのみ許可されます。1つの Primary VLAN に対して定義できる Isolated Secondary VLAN は通常 1 つのみです。
  * **Secondary VLAN (Community VLAN)**: 同一 Community VLAN 内に所属するホスト間同士の Layer 2 通信を **許可** するが、別の Community VLAN や Isolated VLAN との通信は遮断する VLAN。
  * **Promiscuous Port**: すべての Secondary VLAN（Isolated / Community）からの通信を受信・送信できるポート。通常、ルーターやファイアウォール（ASA / FTD）、共通サーバーに割り当てます。
  * **Isolated Port**: Isolated VLAN に割り当てられたポート。Promiscuous Port とのみ通信可能であり、他の Isolated Port や Community Port とは Layer 2 直通通信ができません。
  * **Community Port**: 特定の Community VLAN に割り当てられたポート。同一 Community VLAN 内の他ポートおよび Promiscuous Port と通信可能です。
* **利用目的**:
  * **IP アドレス資源の節約**: サブネット（/24 など）を維持したまま、L2 の分離レベルのみを高められます（各ホストグループに個別の IP サブネットや VLAN SVI を割り当てる必要がありません）。
  * **マルチテナント・DMZ・データセンターのセグメンテーション**: Web サーバー群、テナント環境、パブリック Wi-Fi クライアント間の横移動攻撃（Lateral Movement / East-West 通信）を L2 レベルで防御します。
* **どのような場面で利用するか**:
  * データセンター内の DMZ セグメント（Web サーバー同士の直接通信を拒否し、常に ASA/FTD ファイアウォールを経由させる構成）。
  * 集合住宅（MDU: Multi-Dwelling Unit）やホテル・大学・ISP の顧客エッジスイッチ構成。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **評価対象技術** | **Private VLAN (PVLAN)**, Primary VLAN, Secondary VLAN (Isolated / Community), Port Types (Promiscuous, Isolated, Community, PVLAN Trunk) |
| **対応機種・プラットフォーム** | Cisco Catalyst 9000 シリーズ, Catalyst 3650/3850/4500/6500, Nexus (NX-OS) |
| **VTP との関係** | **VTP Transparent Mode 必須**（VTP v1/v2 利用時）。※ VTP v3 では Primary / Secondary PVLAN 属性の全社共有・伝播がサポートされます。 |
| **主要ポート種別** | **Promiscuous**: 全ポートと通信可能<br>**Isolated**: Promiscuous とのみ通信可能<br>**Community**: 同一 Community ポート ＋ Promiscuous と通信可能<br>**Promiscuous Trunk / Isolated Trunk**: PVLAN タグを維持してスイッチ間伝送 |
| **SVI マッピング** | Primary VLAN SVI 上に IP アドレスを定義し、`private-vlan mapping <Secondary_VLANs>` コマンドで全 Secondary VLAN を関連付けます。 |
| **L3 アドレスとの関係** | すべての Secondary VLAN に属するホストは **同一の IP サブネット** に配置され、デフォルトゲートウェイ（Promiscuous Port / SVI）を共有します。 |
| **設計上の注意点** | 外部ルーター（または SVI）上で **Local Proxy ARP** が有効な場合、Isolated ホスト同士が L3 ルーターを折り返して通信できてしまう「PVLAN 迂回（Bypass）」が発生するため、適切な配慮・制御が必要です。 |

---

## 🏗 動作原理

### Private VLAN のドメイン構造とポートタイプ分類

PVLAN ドメインは、1 つの **Primary VLAN** と、それにバインドされた 1 つ以上の **Secondary VLANs**（Isolated / Community）によって構成されます。

```text
[ External Gateway / Firewall (ASA/FTD) ]
                   │
                   ▼ (Promiscuous Port: VLAN 100)
┌─────────────────────────────────────────────────────────────────────────┐
│ Primary VLAN 100 (Subnet: 10.1.1.0/24, GW: 10.1.1.254)                  │
│                                                                         │
│  ┌───────────────────────────────┐   ┌───────────────────────────────┐ │
│  │ Secondary VLAN 101 (Isolated) │   │ Secondary VLAN 102 (Community)│ │
│  │                               │   │                               │ │
│  │  [ Host A ]     [ Host B ]    │   │  [ Host C ]     [ Host D ]    │ │
│  │  (Isol Port)    (Isol Port)   │   │  (Comm Port)    (Comm Port)   │ │
│  └───────┬─────────────┬─────────┘   └───────┬─────────────┬─────────┘ │
└──────────┼─────────────┼─────────────────────┼─────────────┼───────────┘
           │             │                     │             │
           ✕ (Blocked)   │                     │ (Allowed)   │
           └─────────────┘                     └─────────────┘
      Isolated ホスト間は遮断             同一 Community ホスト間は許可
```

### 通信マトリクス (Inter-Port Communication Matrix)

| 通信送信元 ＼ 宛先 | Promiscuous Port | Isolated Port | Community Port (Group A) | Community Port (Group B) |
| :--- | :--- | :--- | :--- | :--- |
| **Promiscuous Port** | **許可** (Allowed) | **許可** (Allowed) | **許可** (Allowed) | **許可** (Allowed) |
| **Isolated Port** | **許可** (Allowed) | **拒否** (Blocked) | **拒否** (Blocked) | **拒否** (Blocked) |
| **Community Port (Group A)** | **許可** (Allowed) | **拒否** (Blocked) | **許可** (Allowed) | **拒否** (Blocked) |
| **Community Port (Group B)** | **許可** (Allowed) | **拒否** (Blocked) | **拒否** (Blocked) | **許可** (Allowed) |

---

## ⚙ 動作シーケンス

1. **送信フレームの内部 VLAN タグ付与**:
   * Isolated Port（例: Gi1/0/1、VLAN 101 アサイン）にホスト A からフレームが着信。
   * スイッチ内部 ASIC は、入出力を制御するため内部タグとして **Secondary VLAN (VLAN 101)** を付与。
2. **スイッチ内部 ASIC による出力ポート判定**:
   * 送信先 MAC アドレスを MAC テーブルで検索。
   * 宛先ポートが Promiscuous Port（例: Gi1/0/24、Primary VLAN 100 / Secondary 101 マッピング）である場合、出力を**許可**。
   * 宛先ポートが別の Isolated Port（例: Gi1/0/2、VLAN 101 アサイン）である場合、Isolated ドメインルールにより出力を**破棄 (Drop)**。
3. **スイッチ間トランク送出時 (Promiscuous Trunk / Isolated Trunk)**:
   * スイッチ対向へ PVLAN 情報を維持して転送する場合、Standard 802.1Q トランクではなく **PVLAN Trunk Port** を使用。
   * **Isolated Trunk**: 内部的に Secondary VLAN ID（例: VLAN 101）の 802.1Q ヘッダーを保持して送信。対向スイッチ側で Isolated 性質を維持。
   * **Promiscuous Trunk**: Primary VLAN ID（例: VLAN 100）の 802.1Q ヘッダーに変換して送出。対向スイッチからは Promiscuous 送信フレームとして処理。
4. **L3 SVI 折り返し通信シーケンス（重要リスク）**:
   * Host A (Isolated, `10.1.1.1`) が Host B (Isolated, `10.1.1.2`) 宛てに ARP リクエストをブロードキャスト送信。
   * Promiscuous Port 経由で L3 ルーター（または SVI）へ届く。
   * ルーター側で **Local Proxy ARP** が有効になっている場合、ルーターが自身の MAC アドレスで ARP Reply を返答。
   * Host A は Host B 宛てのパケットを L3 ルーター（GW）へ送信し、ルーターが L3 ルーティング（折り返し）して Host B へ転送してしまう（PVLAN が L3 レベルで迂回される）。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **VTP モードの確認（必須前提条件）**:
   * スイッチ上で VTP v1 または v2 が動作している場合、PVLAN を設定しようとすると `Command rejected: Private VLAN configuration is not allowed when VTP is in server or client mode.` エラーが発生します。
   * 必ず事前に **`vtp mode transparent`** を設定するか、VTP v3 を使用して `vtp primary vlan-features` を適用します。
2. **SVI へのマッピング設定 (`private-vlan mapping`)**:
   * L3 ゲートウェイとして Primary VLAN の SVI（例: `interface Vlan 100`）を使用する場合、IP アドレスを定義するだけでなく、**`private-vlan mapping <Secondary_VLAN_List>`**（例: `private-vlan mapping 101,102`）を追加投入しなければ、Secondary VLAN から SVI への L3 通信ができません。
3. **Local Proxy ARP による PVLAN 迂回問題の防止**:
   * 要件に「Ensure hosts in isolated VLAN cannot communicate even through Layer 3 forwarding」とある場合、Primary VLAN SVI または対向ルーターのインターフェイスで **`no ip proxy-arp`** および **`no ip local-proxy-arp`** の適用が必要です。
4. **VACL (VLAN Access Map) と PVLAN の関係**:
   * VACL を適用する場合、**Primary VLAN にのみ VACL をバインド**します（Primary VLAN 上の VACL は、すべての Associated Secondary VLAN トラフィックに対して均一に適用されます）。
   * Secondary VLAN に個別 VACL をバインドしようとすると無視されるかエラーとなります。
5. **DHCP Snooping / DAI と PVLAN のバインディング**:
   * DHCP Snooping や Dynamic ARP Inspection (DAI) を有効化する場合、**Primary VLAN および全 Secondary VLAN の両方** で機能を有効化する必要があります (`ip dhcp snooping vlan 100,101,102`, `ip arp inspection vlan 100,101,102`)。
6. **ASA / FTD と PVLAN スイッチの統合**:
   * ASA / FTD の物理ポートまたは 802.1Q サブインターフェイスは PVLAN タグを直接解釈しません。
   * スイッチ側で ASA 接続ポートを **Promiscuous Port** に割り当てるか、802.1Q トランク上で Secondary VLAN を個別のサブインターフェイス（例: `subif .101`, `subif .102`）として直接受信させ、ASA/FTD の Access Control Policy (ACP) で L3/L4/L7 インスペクションを行う設計が問われます。

---

## 🛠 設定方法

### 1. Cisco Catalyst IOS-XE: PVLAN（Primary, Isolated, Community）標準構成

```bash
! ===== Step 1: VTP Mode を Transparent に変更 =====
vtp mode transparent

! ===== Step 2: Secondary VLANs の定義 =====
vlan 101
 private-vlan isolated                  ! Isolated Secondary VLAN

vlan 102
 private-vlan community                 ! Community Secondary VLAN

! ===== Step 3: Primary VLAN の定義と アソシエーション =====
vlan 100
 private-vlan primary
 private-vlan association 101,102        ! Secondary VLAN 101, 102 を紐付け

! ===== Step 4: Access Port のアサイン =====
! Promiscuous Port (ルーター / GW 接続用)
interface GigabitEthernet1/0/24
 description To_Core_Router_GW
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 100 add 101,102

! Isolated Port (ホスト A / B 接続用)
interface GigabitEthernet1/0/1
 description Host_A_Isolated
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101

interface GigabitEthernet1/0/2
 description Host_B_Isolated
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101

! Community Port (サーバー C / D 接続用)
interface GigabitEthernet1/0/3
 description Server_C_Community
 switchport mode private-vlan host
 switchport private-vlan host-association 100 102

interface GigabitEthernet1/0/4
 description Server_D_Community
 switchport mode private-vlan host
 switchport private-vlan host-association 100 102

! ===== Step 5: L3 SVI へのマッピング設定 (SVI を ゲートウェイにする場合) =====
interface Vlan100
 ip address 10.1.1.254 255.255.255.0
 private-vlan mapping 101,102
 no ip proxy-arp
 no ip local-proxy-arp
```

---

### 2. スイッチ間 PVLAN Trunk 構成 (Promiscuous Trunk ＆ Isolated Trunk)

```bash
! ===== Switch-A (PVLAN Trunk 送信側) =====
interface GigabitEthernet1/0/48
 description To_Switch_B_PVLAN_Trunk
 switchport mode private-vlan trunk promiscuous
 switchport private-vlan mapping trunk 100 add 101,102
 switchport trunk allowed vlan 100,101,102

! ===== Switch-B (対向スイッチ側) =====
vtp mode transparent

vlan 101
 private-vlan isolated
vlan 102
 private-vlan community
vlan 100
 private-vlan primary
 private-vlan association 101,102

interface GigabitEthernet1/0/48
 description To_Switch_A_PVLAN_Trunk
 switchport mode private-vlan trunk secondary
 switchport private-vlan association trunk 100 add 101,102
 switchport trunk allowed vlan 100,101,102
```

---

## 🔍 検証コマンド

| 目的 | コマンド (IOS / IOS-XE) | 表示確認ポイント |
| :--- | :--- | :--- |
| **PVLAN 設定・アソシエーション確認** | <code>show vlan private-vlan</code> | Primary VLAN と Associated Secondary VLANs (Isolated/Community) の関係 |
| **インターフェイスの PVLAN 動作状態確認** | <code>show interface <IF> switchport</code> | `Private VLAN Operational`, `Operational Mode` (Promiscuous / Host / Trunk) |
| **PVLAN インターフェイス一覧確認** | <code>show vlan private-vlan type</code> | 各ポートのバインドタイプ (`promiscuous`, `host`, `isolated`, `community`) |
| **SVI マッピング状態確認** | <code>show interface vlan <ID> private-vlan mapping</code> | Primary SVI に紐づいている Secondary VLAN ID リスト |
| **MAC アドレス・PVLAN マッピング確認** | <code>show mac address-table vlan <Secondary_ID></code> | Secondary VLAN 上で学習されている MAC アドレスと学習ポート |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **VLAN 設定時に `Command rejected` となり `private-vlan` コマンドが拒否される** | スイッチの VTP モードが Client または Server モードになっている（VTP v1/v2） | <code>show vtp status</code> | **`vtp mode transparent`** に変更してから PVLAN コマンドを投入する。 |
| **Isolated ホストから Promiscuous ポート（デフォルトゲートウェイ）宛ての Ping が届かない** | Primary VLAN と Secondary VLAN のアソシエーション、または Promiscuous ポートの `mapping` 設定漏れ | <code>show vlan private-vlan</code><br><code>show interface <IF> switchport</code> | `vlan 100` 配下で `private-vlan association 101` を投入し、Promiscuous ポートで `switchport private-vlan mapping 100 add 101` を設定する。 |
| **Isolated ホスト同士（例: Host A ➔ Host B）で Ping が疎通してしまう** | デフォルトゲートウェイ（ルーター/SVI）上で **Local Proxy ARP** が有効化されており、L3 折り返し通信が発生している | <code>show ip interface <IF></code> | SVI またはルーターインターフェイスで **`no ip proxy-arp`** および **`no ip local-proxy-arp`** を設定する。 |
| **SVI に IP アドレスを設定したが、Secondary VLAN 内のホストから SVI への Ping が全滅する** | SVI インターフェイス上で `private-vlan mapping` コマンドのバインド設定が欠落している | <code>show interface vlan 100 private-vlan mapping</code> | SVI（`interface Vlan100`）上で **`private-vlan mapping 101,102`** を設定する。 |
| **別スイッチに接続された Isolated ホスト同士の間で通信が漏れてしまう** | スイッチ間接続ポートが通常の 802.1Q トランクとなっており、対向スイッチ側で PVLAN タグが適切に処理されていない | <code>show interface <IF> switchport</code> | スイッチ間トランクポートを **`switchport mode private-vlan trunk`** に変更し、Native / Secondary タグを正常伝達させる。 |

---

## ⚠ 制限事項

* **VLAN ID 範囲制限**:
  * PVLAN として定義できる VLAN ID は Standard VLAN (1–1001) および Extended VLAN (1006–4094) の両方でサポートされますが、予約済み VLAN ID (1002–1005) は使用できません。
* **VTP モード制限**:
  * VTP v1/v2 環境では VTP Transparent モードが必須です。VTP Server/Client モードでは PVLAN 情報はデータベースに保存・伝播されません（VTP v3 でのみ伝播可能）。
* **EtherChannel へのアサイン制限**:
  * EtherChannel メンバーポート個別で Private VLAN ポートタイプを変更することはできません。Private VLAN 設定は EtherChannel ポートチャネルインターフェイス（`interface Port-channelX`）上に対して一括設定する必要があります。

---

## 🔄 他技術との関連

* **VACL (VLAN Access Map)**: Primary VLAN に対して VACL を適用することで、所属するすべての Secondary VLAN トラフィックに対して一括セキュリティポリシングを適用可能です。
* **Dynamic ARP Inspection (DAI) / DHCP Snooping**: DAI / DHCP Snooping バインディングデータベースは Primary および Secondary VLAN の両方で有効化しなければ動作不整合を起こします。
* **Cisco ASA / FTD (Firewall)**: PVLAN スイッチと ASA/FTD を接続する場合、ASA 接続ポートを Promiscuous Port とする構成、または 802.1Q サブインターフェイス経由で各 Secondary VLAN を個別のセキュリティゾーンとしてファイアウォールへ収容する構成が一般的です。

---

## 🧩 比較表

### 1. PVLAN ポート種別比較 (Promiscuous vs Isolated vs Community)

| 比較項目 | Promiscuous Port | Isolated Port | Community Port |
| :--- | :--- | :--- | :--- |
| **所属 VLAN** | Primary VLAN | Secondary VLAN (Isolated) | Secondary VLAN (Community) |
| **Promiscuous ポートとの通信** | **許可** | **許可** | **許可** |
| **同一種別ポートとの通信** | **許可** | **拒否** (完全に通信不可) | **許可** (同一 Community 内のみ) |
| **異種 Community ポートとの通信**| **許可** | **拒否** | **拒否** |
| **主な割り当て対象** | デフォルトGW, ASA/FTD, 共通サーバ | 一般クライアント, Webサーバ群 | 同一グループのサーバ/プリンタ群 |

### 2. PVLAN vs Protected Ports (Switchport Port-Security)

| 比較項目 | Private VLAN (PVLAN) | Protected Ports (Switchport Protected) |
| :--- | :--- | :--- |
| **制御スコープ** | スイッチ全体 ＆ スイッチ間（Trunk 経由伝播対応） | **単一スイッチのローカルポート間のみ** |
| **VLAN 構造** | Primary / Secondary (Isolated/Community) 構造 | 通常の単一 VLAN |
| **設定複雑度** | 高い（VLAN アソシエーション・ポートマッピングが必要） | 非常に低い（`switchport protected` 1行のみ） |
| **コミュニティ分離**| 対応（Community VLAN によりグループ単位の制御が可能）| 非対応（Protected ポートか非 Protected ポートかの2択） |

---

## 💡 ベストプラクティス

1. **Proxy ARP の確実な無効化**:
   * PVLAN を導入する最大の目的は L2 横移動の防止です。デフォルトゲートウェイとなる SVI またはルーターポートで `no ip proxy-arp` および `no ip local-proxy-arp` を必ず設定し、L3 経由の迂回を遮断します。
2. **VTP v3 または VTP Transparent モードの事前徹底**:
   * PVLAN 設定前に必ず `vtp mode transparent` を適用し、既存設定の競合エラーを排除します。
3. **Primary VLAN への VACL 集中バインド**:
   * セキュリティポリシー（Access-list）を適用する場合は、Secondary VLAN ではなく Primary VLAN にのみ VACL を適用することで運用・トラブルシュートを単純化します。

---
## 📝 ラボ学習・設定サンプル例

### 例1: 基本的な Isolated PVLAN 構成 (Primary 100, Isolated 101)

#### 問題
スイッチ SW1 において、VLAN 100 を Primary VLAN、VLAN 101 を Isolated Secondary VLAN として定義せよ。`Gi1/0/24` をデフォルトゲートウェイ接続用の Promiscuous ポートとし、`Gi1/0/1` および `Gi1/0/2` を Isolated ポートとして割り当てよ。

#### 要件
* VTP モードを Transparent に変更すること。
* `Gi1/0/1` と `Gi1/0/2` の間での Layer 2 直接通信を遮断すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
vtp mode transparent

vlan 101
 private-vlan isolated

vlan 100
 private-vlan primary
 private-vlan association 101

interface GigabitEthernet1/0/24
 description To_Default_Gateway
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 100 add 101

interface GigabitEthernet1/0/1
 description Host_A_Isolated
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101

interface GigabitEthernet1/0/2
 description Host_B_Isolated
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101
```

#### 検証方法
* SW1 上で `show vlan private-vlan` を実行し、Primary 100 に Secondary 101 がバインドされていることを確認する。
* Host A (`Gi1/0/1`) から Host B (`Gi1/0/2`) 宛てに Ping を実行し、タイムアウト（遮断）することを確認する。
* Host A から デフォルトゲートウェイ宛てに Ping を実行し、正常に疎通することを確認する。

---

### 例2: Community PVLAN 構成 (Community 102)

#### 問題
SW1 において、Web サーバーアプライアンス群用の Community VLAN 102 を作成し、Primary VLAN 100 にバインドせよ。`Gi1/0/3` と `Gi1/0/4` を Community ポートとして設定せよ。

#### 要件
* `Gi1/0/3` と `Gi1/0/4` の間の通信を許可すること。
* `Gi1/0/3`（Community）と `Gi1/0/1`（Isolated）の間の Layer 2 通信を遮断すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
vlan 102
 private-vlan community

vlan 100
 private-vlan primary
 private-vlan association add 102

interface GigabitEthernet1/0/24
 switchport private-vlan mapping 100 add 102

interface GigabitEthernet1/0/3
 description Server_C_Community
 switchport mode private-vlan host
 switchport private-vlan host-association 100 102

interface GigabitEthernet1/0/4
 description Server_D_Community
 switchport mode private-vlan host
 switchport private-vlan host-association 100 102
```

#### 検証方法
* Server C (`Gi1/0/3`) から Server D (`Gi1/0/4`) 宛てに Ping を実行し、成功することを確認する。
* Server C (`Gi1/0/3`) から Host A (`Gi1/0/1`) 宛てに Ping を実行し、遮断されることを確認する。

---

### 例3: Primary SVI への PVLAN Mapping 設定

#### 問題
SW1 上の SVI `Vlan 100` をデフォルトゲートウェイとして機能させるため、Primary IP `10.1.1.254/24` をアサインし、Secondary VLAN 101 および 102 のトラフィックを SVI 上で L3 受信・ルーティングできるように設定せよ。

#### 要件
* `private-vlan mapping` コマンドを使用すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
interface Vlan100
 ip address 10.1.1.254 255.255.255.0
 private-vlan mapping 101,102
 no shutdown
```

#### 検証方法
* SW1 上で `show interface vlan 100 private-vlan mapping` を実行し、`101, 102` が正しくマッピングされていることを確認する。
* Isolated ポート (`Gi1/0/1`) および Community ポート (`Gi1/0/3`) のホストから `10.1.1.254` へ Ping を実行し、両方から応答が返ることを確認する。

---

### 例4: Local Proxy ARP による L3 迂回（Bypass）の防止設定

#### 問題
SW1 の SVI `Vlan 100` において、Isolated ホスト同士がデフォルトゲートウェイ経由で L3 ルーティング（折り返し通信）を行う動作を完全に無効化せよ。

#### 要件
* Proxy ARP および Local Proxy ARP を無効化すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
interface Vlan100
 no ip proxy-arp
 no ip local-proxy-arp
```

#### 検証方法
* SW1 上で `show ip interface vlan 100` を実行し、`Proxy ARP is disabled` および `Local Proxy ARP is disabled` と表示されることを確認する。
* Isolated ホスト A から Isolated ホスト B 宛てに ARP リクエストが送出された際、SVI が自身（GW）の MAC アドレスで代理応答しないことを確認する。

---

### 例5: スイッチ間 PVLAN Promiscuous Trunk 構成

#### 問題
SW1（コアスイッチ）と SW2（ディストリビューションスイッチ）間を接続する `Gi1/0/48` ポート上で、PVLAN タグ情報を通信可能とする Promiscuous Trunk を構成せよ。

#### 要件
* SW1 側を `promiscuous trunk` モードとし、Primary 100 配下の Secondary 101, 102 を送信可能とすること。

#### 設定例

```bash
! ===== SW1 (Core Switch) =====
interface GigabitEthernet1/0/48
 description To_SW2_PVLAN_Trunk
 switchport mode private-vlan trunk promiscuous
 switchport private-vlan mapping trunk 100 add 101,102
 switchport trunk allowed vlan 100,101,102

! ===== SW2 (Access Switch) =====
vtp mode transparent

vlan 101
 private-vlan isolated
vlan 102
 private-vlan community
vlan 100
 private-vlan primary
 private-vlan association 101,102

interface GigabitEthernet1/0/48
 description To_SW1_PVLAN_Trunk
 switchport mode private-vlan trunk secondary
 switchport private-vlan association trunk 100 add 101,102
 switchport trunk allowed vlan 100,101,102
```

#### 検証方法
* SW1 上で `show interface GigabitEthernet1/0/48 switchport` を実行し、`Operational Mode: private-vlan trunk promiscuous` となっていることを確認する。
* SW2 に接続された Isolated ホストから SW1 直結の Promiscuous ポート宛ての Ping が導通することを確認する。

---

### 例6: VACL による Primary VLAN 上での特定プロトコル制限

#### 問題
PVLAN ドメイン（Primary 100, Secondary 101, 102）において、全 Secondary VLAN から外部への ICMP トラフィックを拒否し、それ以外の IP トラフィックを許可する VACL `VACL_BLOCK_ICMP` を作成し、Primary VLAN 100 に適用せよ。

#### 要件
* VACL は Primary VLAN にのみ適用すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
ip access-list extended ACL_ICMP_DENY
 permit icmp any any

vlan access-map VACL_BLOCK_ICMP 10
 match ip address ACL_ICMP_DENY
 action drop

vlan access-map VACL_BLOCK_ICMP 20
 action forward

vlan filter VACL_BLOCK_ICMP vlan-list 100
```

#### 検証方法
* SW1 上で `show vlan filter` を実行し、`VLAN map VACL_BLOCK_ICMP is configured on vlan 100` と表示されることを確認する。
* Isolated ホスト A から外部ルーター宛てに Ping を実行し、VACL によってドロップされることを確認する。

---

### 例7: DHCP Snooping ＆ DAI (Dynamic ARP Inspection) と PVLAN の統合設定

#### 問題
PVLAN が構成された SW1 において、IP スプーフィングおよび不正 DHCP サーバーの設置を防ぐため、Primary VLAN 100 および Secondary VLAN 101, 102 のすべてに対して DHCP Snooping と DAI を有効化せよ。`Gi1/0/24`（Promiscuous Port）を Trusted ポートに指定せよ。

#### 要件
* `ip dhcp snooping vlan` および `ip arp inspection vlan` を Primary ＋ Secondary 全 VLAN に設定すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
ip dhcp snooping
ip dhcp snooping vlan 100,101,102
no ip dhcp snooping information option

ip arp inspection vlan 100,101,102

interface GigabitEthernet1/0/24
 description To_DHCP_Server_GW
 ip dhcp snooping trust
 ip arp inspection trust
```

#### 検証方法
* SW1 上で `show ip dhcp snooping` および `show ip arp inspection vlan 100` を実行し、VLAN 100, 101, 102 でアクティブになっていることを確認する。
* Isolated ポートから不正 ARP パケットを送信し、DAI によってドロップされることを確認する。

---

### 例8: Cisco ASA と PVLAN スイッチの Promiscuous Port 接続

#### 問題
Cisco ASA の Outside インターフェイス (`Gi0/0`) を SW1 の `Gi1/0/24` (Promiscuous Port) に接続し、ASA を PVLAN ドメイン（Primary 100, Secondary 101, 102）のデフォルトゲートウェイとして機能させよ。

#### 要件
* ASA 上で VLAN 100 宛てのアクセスリストと NAT を適切に定義すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
interface GigabitEthernet1/0/24
 description To_ASA_Outside
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 100 add 101,102

! ===== ASA Configuration =====
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 10.1.1.254 255.255.255.0
 no shutdown

object network OBJ_INSIDE_NET
 subnet 10.1.1.0 255.255.255.0
 nat (outside,outside) dynamic interface
```

#### 検証方法
* Isolated ポート接続ホストから ASA の IP `10.1.1.254` 宛てに Ping を実行し、応答を得られることを確認する。

---

### 例9: FTD サブインターフェイスによる Secondary VLAN の個別 L3/L4/L7 インスペクション

#### 問題
Cisco FTD の物理ポート `Gi0/0` を SW1 の標準 802.1Q トランクポートに接続し、Isolated VLAN 101 および Community VLAN 102 を個別の FTD サブインターフェイス (`Gi0/0.101`, `Gi0/0.102`) として収容せよ。FTD の Access Control Policy (ACP) で両セグメント間の通信を検査・制御せよ。

#### 要件
* FTD サブインターフェイス上で 802.1Q タグ 101 および 102 を個別に終端すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
interface GigabitEthernet1/0/23
 description To_FTD_Trunk
 switchport mode trunk
 switchport trunk allowed vlan 101,102

! ===== FTD / FMC GUI Steps =====
! 1. Devices > Device Management > Interfaces ➔ Add Subinterface
!    - Physical Interface: GigabitEthernet0/0
!    - Subinterface ID: 101, VLAN ID: 101, Name: Isolated_Zone_101
!    - IP Address: 10.1.101.254/24
! 2. Add Subinterface:
!    - Subinterface ID: 102, VLAN ID: 102, Name: Community_Zone_102
!    - IP Address: 10.1.102.254/24
! 3. Policies > Access Control ➔ Rule: Allow Isolated_Zone_101 to Community_Zone_102 with IPS Inspection.
```

#### 検証方法
* FTD CLI で `show interface` を実行し、`GigabitEthernet0/0.101` および `.102` が Up 状態であることを確認する。
* FMC の Connection Events ログで、VLAN 101 から VLAN 102 への通信が FTD 上で ACP ルールおよび IPS スキャンを通過していることを確認する。

---

### 例10: PVLAN 上での MAC アドレス学習制限と Port Security の統合

#### 問題
SW1 の Isolated ポート `Gi1/0/1` において、MAC アドレスの偽装（Spoofing）や MAC Flooding 攻撃を防ぐため、Port Security を有効化し、最大学習 MAC アドレス数を 1 に制限せよ。

#### 要件
* `switchport port-security` を Isolated ポートに設定すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
interface GigabitEthernet1/0/1
 description Host_A_Isolated
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 switchport port-security mac-address sticky
```

#### 検証方法
* SW1 上で `show port-security interface GigabitEthernet1/0/1` を実行し、`Port Security: Enabled` および `MaxAddresses: 1` と表示されることを確認する。
* ポートに別 MAC アドレスを持つ端末を接続した際、Violation カウンタが増加し、通信が遮断されることを確認する。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: スイッチ SW1 上で以下のコンフィグを投入したところ、`Command rejected: Private VLAN configuration is not allowed` というエラーが表示された。原因として最も適切なものはどれか。

```bash
SW1(config)# vlan 101
SW1(config-vlan)# private-vlan isolated
Command rejected: Private VLAN configuration is not allowed when VTP is in server or client mode.
```

* A) VLAN 101 が予約済み VLAN ID（1002–1005）と重複しているため。
* B) SW1 の VTP モードが Client または Server モードになっているため。
* C) `vlan 100`（Primary VLAN）を事前に定義していないため。
* D) 物理インターフェイス上で `switchport mode private-vlan` を設定していないため。

**正解**: **B**  
**解説**: VTP v1 および v2 環境では、スイッチが Server または Client モードの場合に PVLAN を設定できません。事前に **`vtp mode transparent`** を適用して Transparent モードへ変更する必要があります。

---

### 2. トラブルシュート問題

**問題**: SW1 上で Primary VLAN 100（IP: `10.1.1.254`）、Isolated Secondary VLAN 101 を設定し、`Gi1/0/1` に Isolated ホスト A (`10.1.1.1`) を接続した。ホスト A から `10.1.1.254` への Ping が応答しない。`show vlan private-vlan` の確認結果は問題なかった。最も可能性の高い原因と確認コマンドはどれか。

* A) `interface Vlan100` 上で `private-vlan mapping 101` コマンドが設定されていない。確認コマンド: `show interface vlan 100 private-vlan mapping`
* B) ホスト A のデフォルトゲートウェイが誤っている。確認コマンド: `show ip route`
* C) `Gi1/0/1` ポートで `switchport mode access` が設定されている。確認コマンド: `show interface GigabitEthernet1/0/1 status`
* D) VTP モードが Off になっている。確認コマンド: `show vtp status`

**正解**: **A**  
**解説**: Primary VLAN の SVI（`interface Vlan100`）経由で Secondary VLAN 内のホストと L3 通信を行うには、SVI インターフェイス上で **`private-vlan mapping <Secondary_VLAN_ID>`** を明示的に設定しなければなりません。これがない場合、SVI は Secondary VLAN からのフレームを受け入れません。

---

### 3. Design 問題

**問題**: データセンターの Web サーバーアプライアンス層において、同一次元（同一 IP サブネット `172.16.10.0/24`）に属する 50 台の Web サーバー間の横移動（East-West 通信）を遮断し、上位の Cisco ASA ファイアウォールでのみすべてのトラフィックを集中検査させたい。また、IP アドレスの変更は許容されない。最も適切な設計はどれか。

* A) 各 Web サーバーを別々の VLAN および /30 サブネットに分割する。
* B) Web サーバー接続ポートを **Private VLAN Isolated Port** に割り当て、ASA 接続ポートを **Promiscuous Port** に割り当てる。また、ASA 上で (Outside, Outside) ヘアピン Inspection を構成する。
* C) スイッチ全ポートで `switchport mode trunk` を設定する。
* D) Web サーバー接続ポートで 802.1X 認証を必須化する。

**正解**: **B**  
**解説**: 同一サブネット（IP アドレス変更なし）でホスト間の L2 直通通信を遮断し、必ず上位のファイアウォール（ASA）を経由させるには、**PVLAN の Isolated Port と Promiscuous Port の組み合わせ** がベストプラクティスです。

---

### 4. 実装問題

**問題**: PVLAN ドメインにおいて、Isolated ポートに接続されたホスト A (`10.1.1.1`) とホスト B (`10.1.1.2`) が、同一 L2 スイッチ上に存在しているにもかかわらず、互いに Ping が通ってしまう現象が発生した。ネットワーク管理者が確認したところ、デフォルトゲートウェイ（ルーター）上で特定の機能が有効になっていた。この原因となるルーターの機能および修正コマンドの組み合わせとして正しいものはどれか。

* A) 機能: Dynamic ARP Inspection、修正: `no ip arp inspection`
* B) 機能: Local Proxy ARP、修正: **`no ip proxy-arp`** および **`no ip local-proxy-arp`**
* C) 機能: ICMP Redirects、修正: `no ip redirects`
* D) 機能: IP Unreachables、修正: `no ip unreachables`

**正解**: **B**  
**解説**: デフォルトゲートウェイ（ルーターや SVI）上で **Proxy ARP** または **Local Proxy ARP** が有効になっていると、Isolated ホストが送信した同一サブネット宛ての ARP リクエストに対してルーターが自身の MAC アドレスを代理応答してしまいます。これによりパケットがルーターで L3 折り返し転送（Bypass）され、Isolated ホスト間の通信が成功してしまいます。これを防ぐには `no ip proxy-arp` および `no ip local-proxy-arp` の設定が必要です。

---

### 5. コンフィグ読解問題

**問題**: 以下のコンフィグが投入されたスイッチ SW1 のポート `Gi1/0/3` に関する記述として正しいものはどれか。

```bash
vlan 102
 private-vlan community
vlan 100
 private-vlan primary
 private-vlan association 102

interface GigabitEthernet1/0/3
 switchport mode private-vlan host
 switchport private-vlan host-association 100 102
```

* A) `Gi1/0/3` は Isolated ポートであり、他のすべてのポートと通信できない。
* B) `Gi1/0/3` は Community ポートであり、同一 Community VLAN 102 に属する他の Community ポート、および Primary VLAN 100 の Promiscuous ポートと通信できる。
* C) `Gi1/0/3` は Promiscuous ポートであり、全 VLAN のトラフィックを受信できる。
* D) `Gi1/0/3` は VLAN 100 と VLAN 102 の標準 802.1Q トランクポートとして動作する。

**正解**: **B**  
**解説**: `vlan 102` は `private-vlan community` として定義されており、`Gi1/0/3` は `host-association 100 102` により **Community Port** として動作します。同じ Community VLAN 102 内のポートおよび Promiscuous ポートとのみ通信可能です。

---

## 🔗 参考リソース

* [Cisco Catalyst 9300 Series Switch CLI Configuration Guide: Configuring Private VLANs](https://www.cisco.com/)
* [Cisco Technical Notes: Secure Private VLAN Configurations and Troubleshooting](https://www.cisco.com/)
* [Cisco Validated Design (CVD): Data Center Micro-Segmentation with Private VLANs](https://www.cisco.com/)
* [Cisco Live: Enterprise L2 Security and Private VLAN Architecture (BRKSEC-2001)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で PVLAN が出題された場合の迅速な構成判断チェックリスト：
    * 1. `vtp mode transparent` を真っ先に確認・投入。
    * 2. Secondary VLAN（`isolated` / `community`）を定義 ➔ Primary VLAN を定義し `private-vlan association <Secondary_IDs>` をバインド。
    * 3. GW ポート（Promiscuous）に `switchport mode private-vlan promiscuous` ＋ `switchport private-vlan mapping <Primary> add <Secondary_List>` をアサイン。
    * 4. クライアントポート（Host）に `switchport mode private-vlan host` ＋ `switchport private-vlan host-association <Primary> <Secondary>` をアサイン。
    * 5. SVI を使用する場合は `interface Vlan<Primary>` 上で `private-vlan mapping <Secondary_List>` ＋ `no ip proxy-arp` ＋ `no ip local-proxy-arp` を追加。
* **図解（PVLAN トラフィック分離ロジック）**:
  ```text
  [ Isolated Host A ] ───► (VLAN 101) ───► [ Switch ] ───► (Blocked) ───► [ Isolated Host B ]
                                              │
                                              ├───► (Allowed) ───► [ Promiscuous GW / ASA ]
                                              │
  [ Community Host C ] ──► (VLAN 102) ───► [ Switch ] ───► (Allowed) ───► [ Community Host D ]
                                              │
                                              └───► (Blocked) ───► [ Isolated Host A ]
  ```
* **注意点**:
  * ラボ試験で PVLAN を設定した後は、必ず `show vlan private-vlan` および `show interface <IF> switchport` を実行し、Operational Mode が正しく反映されていること、および端末間 Ping 検証で Isolated ポート同士が遮断され GW 宛てが疎通することを確認してください。
