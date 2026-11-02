---
layout: default
title: 2.5.a-VLAN
nav_order: 1
parent: 2.5-Infrastructure-segmentation
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.5.a VLAN

Cisco ネットワークインフラストラクチャおよびセキュリティアーキテクチャにおける **VLAN（Virtual Local Area Network：仮想ローカルエリアネットワーク）** 技術は、物理的なスイッチングインフラストラクチャ上に論理的なブロードキャストドメイン（L2 セグメント）を形成し、トラフィックの孤立化、アクセス制御、帯域幅の最適化、およびアイデンティティに基づくセキュリティ境界を提供する基礎セグメンテーション技術です。

CCIE Security v6.1 Blueprint **2.5.a VLAN** では、単なる L2 スイッチングの概念にとどまらず、**IEEE 802.1Q フレームタギングメカニズム**、**VLAN Hopping（Double Tagging / DTP 悪用）攻撃および Native VLAN セキュリティ脅威の緩和**、**VTP v1/v2/v3 のセキュリティ強化（VTP v3 Domain Authentication & Primary Server）**、**Cisco ASA および FTD における 802.1Q サブインターフェイス統合（Inter-VLAN Routing & Security Levels）**、**Cisco ISE（Identity Services Engine）による動的 VLAN 割り当て（Dynamic VLAN Assignment via 802.1X / MAB）**、および **VACL（VLAN Access Control List）による Intra-VLAN トラフィック制御** に至るまで、インフラストラクチャのセキュリティ保護と透過的セグメンテーションを高度に実装・トラブルシューティングする能力が評価されます。

---

## 📘 概要

* **機能概要**:
  * **VLAN（IEEE 802.1Q）**: 物理スイッチポートを特定の論理ブロードキャストドメインにアサインし、レイヤー 2 レベルでのトラフィック分離を実現する。トランクリンク上では Ethernet フレームヘッダー内に 4 バイトの 802.1Q タグ（TPID: `0x8100`、VLAN ID: 1～4094）を挿入して複数 VLAN を単一の物理リンク上で多重化・伝送する。
  * **VLAN セキュリティ Hardening**: デフォルト VLAN（VLAN 1）の無効化・非推奨化、Native VLAN の明示的タグ付け（`vlan dot1q tag native`）および未割り当て専用 VLAN へのアサイン、DTP（Dynamic Trunking Protocol）の無効化（`switchport nonegotiate`）による VLAN Hopping 攻撃の完全排除。
  * **Cisco ASA / FTD 統合**: 単一の物理インターフェイス上に複数の 802.1Q サブインターフェイス（例: `GigabitEthernet0/0.10`）を構成し、各サブインターフェイスに独立した Security Level や Security Zone、IP サブネットをアサインして、ファイアウォール境界におけるレイヤー 3 インター VLAN ルーティングおよびステートフルアクセス制御を実行する。
  * **Cisco ISE 動的 VLAN 制御**: 端末が 802.1X または MAB（MAC Authentication Bypass）で認証された際、RADIUS 属性（IETF Attributes 64 `Tunnel-Type=VLAN`, 65 `Tunnel-Medium-Type=802`, 81 `Tunnel-Private-Group-ID=VLAN_ID/Name`）を用いて、スイッチポートを端末の属性やセキュリティポスチャ状態に応じた VLAN へ動的に変更する。
* **利用目的**:
  * 組織内の部門、サーバー群、ゲスト端末、IoT 機器、音声端末（Voice VLAN）間の L2 ブロードキャストドメインを分離し、マルウェアの横移動（Lateral Movement）やブロードキャストストームの被害範囲を限定する。
  * ファイアウォール（ASA / FTD）やマルチレイヤースイッチ（SVI）と連携し、異種セキュリティレベル間のトラフィックを明示的にアクセス制御（ACL / ACP）する。
* **どのような場面で利用するか**:
  * エンタープライズ Campus LAN、DMZ ネットワーク、Data Center マイクロセグメンテーション。
  * 802.1X / MAB によるゼロトラストアクセス制御環境での動的クランティン（検疫）VLAN やゲスト VLAN の動的割り当て。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **評価技術要素** | IEEE 802.1Q Tagging、Native VLAN Security、VLAN Hopping 軽減、DTP 制御、VTP v3、VACL、Dynamic VLAN (ISE)、ASA/FTD Subinterfaces |
| **802.1Q ヘッダー構造** | 4 バイト（TPID: 2 bytes `0x8100`、PCP/Priority: 3 bits、DEI/CFI: 1 bit、VLAN ID: 12 bits [1～4094]） |
| **標準/拡張 VLAN 範囲** | **Standard VLAN**: 1～1005（VTP v1/v2 にて VLAN データベースへ保存、1/1002-1005 は予約）<br>**Extended VLAN**: 1006～4094（`running-config` へ保存、VTP v3 で透過・伝播対応） |
| **セキュリティ脅威と対策** | **VLAN Hopping (DTP 悪用)**: ポートを `switchport mode access` / `nonegotiate` に固定<br>**Double Tagging**: Native VLAN を未使用の専用 VLAN に割り当て、かつ `vlan dot1q tag native` を有効化 |
| **動的制御 (Cisco ISE)** | RADIUS Attributes: `Tunnel-Type=13(VLAN)`, `Tunnel-Medium-Type=6(802)`, `Tunnel-Private-Group-ID=VLAN_ID/Name` |
| **ASA / FTD サブインターフェイス** | 802.1Q サブインターフェイスを作成し、`vlan <1-4094>` を指定。ASA では各サブインターフェイスに `security-level` を定義。 |
| **VACL (VLAN ACL)** | 同一 VLAN 内（Intra-VLAN）の通信を L2/L3/L4 条件（IP ACL / MAC ACL）でフィルタリングする機能 |

---

## 🏗 動作原理

### 1. 802.1Q タギングと Native VLAN 伝送メカニズム

トランクリンク上における 802.1Q フレーム構造と、Native VLAN（未タグ付け）パケットの通過フロー：

```text
[ Access Port (VLAN 10) ]
       │ (Untagged Ethernet Frame)
       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Switch A (Ingress Processing)                                                    │
│ 1. Assigns Internal VLAN ID = 10                                                 │
│ 2. Egress Interface: Trunk Port (Native VLAN = 99)                               │
│ 3. Inserts 4-Byte 802.1Q Tag into Frame Header (TPID=0x8100, VID=10)             │
└──────────────────────────────────────────────────────────────────────────────────┘
       │
       ├─── Tagged Frame (VID = 10) ──────────────► [ Switch B Trunk Port ]
       │                                                     │
       │                                                     ├── Strips 802.1Q Tag
       │                                                     └── Forwards to Access Port (VLAN 10)
       │
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Switch A (Native VLAN Packet Processing)                                         │
│ 1. Packet belongs to Native VLAN (VID = 99)                                      │
│ 2. Transmits Frame WITHOUT 802.1Q Tag over Trunk Link                             │
└──────────────────────────────────────────────────────────────────────────────────┘
       │
       └─── Untagged Frame (Native VLAN 99) ────────► [ Switch B Trunk Port ]
                                                             │
                                                             └── Assumes Native VLAN (VID 99)
```

---

### 2. VLAN Hopping (Double Tagging) 攻撃と遮断メカニズム

攻撃者が Native VLAN と同じタグを第 1 タグ、目的のターゲット VLAN（例: VLAN 20）を第 2 タグとして二重カプセル化したパケットを送信し、中間スイッチを騙して別 VLAN へパケットを到達させる攻撃構造と防ぐ仕組み：

```text
[ Attacker PC (Access Port in Native VLAN 10) ]
       │
       ├── Sends Double-Tagged Frame: [ Outer Tag: VID 10 | Inner Tag: VID 20 | IP Data ]
       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Switch 1 (Trunk Link with Native VLAN = 10)                                      │
│ 1. Receives frame on Native VLAN 10.                                             │
│ 2. Strips Outer Tag (VID 10) because it matches Trunk Native VLAN.               │
│ 3. Forwards frame over Trunk Link WITHOUT Outer Tag.                             │
└──────────────────────────────────────────────────────────────────────────────────┘
       │
       ├── Frame with Inner Tag Only: [ Tag: VID 20 | IP Data ]
       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Switch 2                                                                         │
│ 1. Sees Tag VID 20.                                                              │
│ 2. Forwards frame directly into VLAN 20! (VLAN Hopping Success if Unprotected)    │
└──────────────────────────────────────────────────────────────────────────────────┘

【 緩和策 (Mitigation) 】
1. 全トランクポートで Native VLAN を全アクセスポートで使用されていない専用 VLAN（例: VLAN 999）に変更する。
2. グローバルで `vlan dot1q tag native` を設定し、Native VLAN パケットにも 802.1Q タグを強制付与してアウタータグ剥ぎ取りを無効化する。
3. 全アクセスポートで `switchport mode access` および `switchport nonegotiate` を投入し、DTP トランキング自動交渉を完全遮断する。
```

---

## ⚙ 動作シーケンス

1. **インプレス・パケット受信と VLAN 識別**:
   * **Access Port 着信**: 送信されたノーマル Ethernet フレームを受信。スイッチはポートに設定された `switchport access vlan <ID>` の PVID（Port VLAN ID）をローカルタグとしてパケットに内部アサイン。
   * **Trunk Port 着信**:
     * 802.1Q タグ付きフレームの場合: ヘッダー内の VLAN ID（VID）を読取り、対応する内部 VLAN バスのインスペクションに投入。
     * タグ無し（Untagged）フレームの場合: トランクポートに定義された `switchport trunk native vlan <ID>` の VLAN として受領。
2. **ASA / FTD ファイアウォールでの サブインターフェイス 評価**:
   * トランクリンク経由で ASA / FTD の物理ポート（例: `Gi0/0`）にタギングパケット（VID = 10）が着信。
   * LINA / Data Plane 階層で、`GigabitEthernet0/0.10` サブインターフェイスの `vlan 10` 定義とマッチング。
   * 該当サブインターフェイスに割り当てられた Security Level（ASA）または Security Zone（FTD）のアクセス制御ポリシー（ACP）および NAT テーブルが順次評価される。
3. **802.1X / Dynamic VLAN 割当シーケンス (Cisco ISE)**:
   * 端末がスイッチポートへ接続し、EAP-TLS や MAB 認証を開始。
   * スイッチ（RADIUS クライアント）は ISE（RADIUS サーバー）へ `Access-Request` を送信。
   * ISE は認証・認可ポリシーを評価し、`Access-Accept` と共に以下の VLAN 属性を返却：
     * IETF Attribute 64 (`Tunnel-Type = VLAN [13]`)
     * IETF Attribute 65 (`Tunnel-Medium-Type = 802 [6]`)
     * IETF Attribute 81 (`Tunnel-Private-Group-ID = "VLAN_NAME_OR_ID"`)
   * スイッチは受領した VLAN ID/Name をポートの動的 VLAN として適用し、MAC アドレスを該当 VLAN の MAC アドレステーブルへ追加。

---



## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **VLAN Hopping ＆ Native VLAN 脆弱性の完全緩和策**:
   * **`vlan dot1q tag native`**: グローバル設定で Native VLAN パケットにも 802.1Q タグを強制付与する（Double Tagging の完全阻止）。
   * **未使用ポーションの閉塞**: アクセスポートを明示的に `switchport mode access` および `switchport nonegotiate`（DTP 無効化）に設定し、トランク化（Dynamic Desirable / Auto）を狙う攻撃を阻止する。
   * **Parking Lot VLAN**: 未使用ポートをすべてアクティブなデータ VLAN や Native VLAN から独立した専用の黒星 VLAN（例: VLAN 999 `PARKING`）へアサインし、`shutdown` する。
2. **VTP v3（VLAN Trunking Protocol Version 3）の必須構成**:
   * CCIE ラボ試験では、従来型の VTP v1/v2 ではなく **VTP v3** の運用が指示されることが一般的です。
   * **VTP v3 の特徴と必須コマンド**:
     * 拡張 VLAN（VLAN 1006～4094）の伝播に対応。
     * 勝手にデータベースが上書きされるリスクを防ぐため、**`vtp primary vlan`** コマンドで明示的に Primary Server を選定（`secret <password>` が設定されている場合は入力が必要）。
     * コマンド: `vtp domain <NAME>`, `vtp version 3`, `vtp mode server vlan`, `vtp password <PASS> hidden`, `vtp primary vlan`（EXEC モード）。
3. **Cisco ISE 動的 VLAN アサイン (RADIUS VSA / IETF Attributes)**:
   * 802.1X / MAB 認証成功後、ISE からスイッチポートへ動的 VLAN をプッシュする要件。
   * **必須 RADIUS 属性 3 点セット**:
     * Attribute 64 (`Tunnel-Type = VLAN`)
     * Attribute 65 (`Tunnel-Medium-Type = 802`)
     * Attribute 81 (`Tunnel-Private-Group-ID = <VLAN_ID_OR_NAME>`)
   * スイッチ側の必須設定: `dot1x pae authenticator`, `authentication port-control auto`, `mab`, `aaa authorization network default group radius`。
4. **Cisco ASA / FTD 802.1Q サブインターフェイス ＆ ルーティング制御**:
   * **ASA サブインターフェイス**:
     ```text
     interface GigabitEthernet0/0.10
      vlan 10
      nameif INSIDE_VLAN10
      security-level 100
      ip address 10.1.10.1 255.255.255.0
     ```
   * **`same-security-traffic`**: 同一セキュリティレベルを持つ 2 つの VLAN サブインターフェイス間（例: `INSIDE10` = 100, `INSIDE20` = 100）での通信を許可する場合、**`same-security-traffic permit inter-interface`** の投入が必要。
   * **FTD サブインターフェイス (FMC)**: Devices > Device Management で物理ポートに Subinterface (`.10`) を作成 ➔ 802.1Q VLAN ID を指定 ➔ Security Zone (例: `Zone_VLAN10`) にアサイン。
5. **VLAN Access Control List (VACL) による同一 VLAN 内 L2 フィルタリング**:
   * 同一 VLAN サブネット内で、特定ホスト間（例: Host A `10.1.1.10` と Host B `10.1.1.20`）の通信を L2 レベルで遮断・キャプチャする設定。
   * `vlan access-map <MAP_NAME> <SEQ>` ➔ `match ip address <ACL_NAME>` ➔ `action drop` / `action forward` ➔ `vlan filter <MAP_NAME> vlan-list <VLAN_IDs>`。

---

## 🛠 設定方法

### 1. Cisco Switch: セキュリティ Hardening ＆ VTP v3 / Native VLAN Tagging 完全構成

```bash
! ===== Step 1: グローバル 802.1Q Native VLAN タギング ＆ VTP v3 構成 =====
vlan dot1q tag native                   ! Native VLAN に対しても 802.1Q タグを強制付与

vtp domain CCIE_SECURITY
vtp version 3
vtp mode server vlan
vtp password Cisco123Secret hidden

! 物理スイッチ上で VTP Primary Server の権限をアクティベート (EXEC Mode)
! # vtp primary vlan

! ===== Step 2: 未使用 VLAN / Native VLAN の定義 =====
vlan 10
 name CORPORATE_DATA
vlan 20
 name VOICE_NET
vlan 99
 name NATIVE_TRUNK
vlan 999
 name PARKING_LOT

! ===== Step 3: アクセスポートの Hardening 設定 =====
interface GigabitEthernet1/0/10
 description --- Access Port for Workstation ---
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 switchport nonegotiate                 ! DTP パケットの送信・交渉を完全に停止
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! 未使用ポートの Parking Lot 設定
interface range GigabitEthernet1/0/20 - 24
 description --- Unused Ports ---
 switchport mode access
 switchport access vlan 999
 switchport nonegotiate
 shutdown

! ===== Step 4: トランクポートの Hardening 設定 =====
interface GigabitEthernet1/0/1
 description --- Trunk Link to Distribution Switch ---
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99         ! デフォルトの VLAN 1 から未割り当て VLAN 99 へ変更
 switchport trunk allowed vlan 10,20,99  ! 必要な VLAN のみを明示許可 (Pruning)
 switchport nonegotiate                 ! DTP 交渉を無効化
 no shutdown
```

---

### 2. Cisco ASA: 802.1Q サブインターフェイス ＆ インター VLAN ルーティング

```bash
! ===== ASA 物理ポート ＆ サブインターフェイス構成 =====
interface GigabitEthernet0/0
 description --- Trunk Link to Switch ---
 no shutdown

interface GigabitEthernet0/0.10
 description --- Corporate Data VLAN 10 ---
 vlan 10
 nameif INSIDE_DATA
 security-level 100
 ip address 10.1.10.1 255.255.255.0

interface GigabitEthernet0/0.20
 description --- Guest VLAN 20 ---
 vlan 20
 nameif GUEST_NET
 security-level 10
 ip address 10.1.20.1 255.255.255.0

interface GigabitEthernet0/0.30
 description --- DMZ VLAN 30 ---
 vlan 30
 nameif DMZ_ZONE
 security-level 50
 ip address 10.1.30.1 255.255.255.0

! 同一 Security Level (例: Security Level 50 同士) 間通信の許可設定
same-security-traffic permit inter-interface
```

---

### 3. Cisco ISE: Dynamic VLAN Assignment 認証・認可プロファイル設定 (Switch & RADIUS)

```bash
! ===== Switch 側 802.1X / Dynamic VLAN アサイン有効化設定 =====
aaa new-model
aaa authentication dot1x default group radius
aaa authorization network default group radius

radius server ISE_NODE
 address ipv4 192.168.1.100 auth-port 1812 acct-port 1813
 key Cisco123Key

interface GigabitEthernet1/0/5
 description --- Dynamic 802.1X Port ---
 switchport mode access
 switchport access vlan 999             ! 未認証時のダミー VLAN
 authentication port-control auto
 dot1x pae authenticator
 mab
 no shutdown
```

```text
[ Cisco ISE Authorization Profile 設定 (GUI) ]
1. Policy > Policy Elements > Results > Authorization > Authorization Profiles ➔ "Add"
2. Profile Name: AUTH_PROF_FINANCE_VLAN
3. Common Tasks:
   - [✓] VLAN:
     - ID/Name: 10 (または "CORPORATE_DATA")
4. Advanced Attributes Settings (自動生成される RADIUS 属性):
   - Tunnel-Type = 13 (VLAN)
   - Tunnel-Medium-Type = 6 (802)
   - Tunnel-Private-Group-ID = 10
5. Policy > Authorization Policies で条件 (例: User_Group EQUALS Finance) に上記 Profile をアサイン。
```

---

### 4. Switch CLI: VLAN Access Control List (VACL) 同一 VLAN 内通信制限

```bash
! ===== 同一 VLAN 10 内の Host A (10.1.10.50) から Host B (10.1.10.60) への通信をドロップ =====
ip access-list extended ACL_VACL_BLOCK_HOSTS
 permit ip host 10.1.10.50 host 10.1.10.60
 permit ip host 10.1.10.60 host 10.1.10.50

vlan access-map MAP_VACL_SEC 10
 match ip address ACL_VACL_BLOCK_HOSTS
 action drop

vlan access-map MAP_VACL_SEC 20
 action forward                         ! その他の全通信は許可

vlan filter MAP_VACL_SEC vlan-list 10   ! VLAN 10 へ VACL を適用
```

---

## 🔍 検証コマンド

| 目的 | コマンド (Switch / ASA / FTD) | 表示確認ポイント |
| :--- | :--- | :--- |
| **VLAN データベース ＆ ポート割当確認** | <code>show vlan brief</code> | 各 VLAN の Active 状態、所属アクセスポート一覧 |
| **トランクポート ＆ Native VLAN 確認** | <code>show interfaces trunk</code> | Native VLAN ID、Status (`trunking`)、Allowed VLANs 一覧 |
| **Native VLAN タギング状態確認** | <code>show vlan dot1q tag native</code> | `dot1q native vlan tagging is enabled` 表示の有無 |
| **VTP v3 ドメイン ＆ Primary 状態確認** | <code>show vtp status</code> | Version (`3`)、Mode (`Server`)、Primary ID、Domain Name |
| **Switchport の DTP 動作状態確認** | <code>show interfaces Gi1/0/1 switchport</code> | `Administrative Mode: static access`, `Negotiation of Trunking: Off` |
| **VACL (VLAN Filter) 適用状態確認** | <code>show vlan filter</code> | VLAN Access Map 名とアサインされている VLAN List |
| **スイッチポートでの動的 VLAN アサイン確認** | <code>show authentication sessions interface Gi1/0/5</code> | `Status: Auth Success`, `VLAN Domain: DATA`, `Assigned VLAN: 10` |
| **ASA サブインターフェイス ＆ VLAN 確認** | <code>show interface ip brief</code><br><code>show running-config interface</code> | 各サブインターフェイス (例: `Gi0/0.10`) の VLAN ID と Security Level |
| **トランクリンクのパケットデバッグ** | <code>debug sw-vlan packets</code> | VLAN タグの制御パケット処理状態 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **トランクリンクを介した通信が不通、または CDP/STP エラーログが大量発生する** | 対向スイッチ間で **Native VLAN ID の不一致 (Mismatch)** | <code>show interfaces trunk</code><br><code>show logging</code> | 両端のトランクポートで `switchport trunk native vlan <ID>` の値を一致させる。 |
| **DTP 攻撃を阻止するために `switchport mode access` を入れたがトランク化してしまう** | 対向側が `dynamic desirable` で、自ポートで `switchport nonegotiate` が未投入 | <code>show interfaces switchport</code> | `switchport mode access` に加え、明示的に **`switchport nonegotiate`** を設定して DTP ネゴシエーションを完全に停止する。 |
| **VTP v3 環境で新しく作成した VLAN が他スイッチへ伝播しない** | 自スイッチが **Primary Server** として昇格（EXEC モードで `vtp primary vlan` 実行）していない | <code>show vtp status</code> | EXEC モードで **`vtp primary vlan`** を実行し、データベース編集権限を確保の上でコミットする。 |
| **ISE から動的 VLAN（例: VLAN 10）が降ってきたが、スイッチで適用に失敗（`Auth Fail`）する** | スイッチ上に該当する VLAN（VLAN 10）が**あらかじめ作成（`vlan 10`）されていない** | <code>show vlan brief</code><br><code>show authentication sessions</code> | スイッチ上で `vlan 10` を事前に定義しておく（存在しない VLAN は動的アサイン不可）。 |
| **ASA の同一 Security Level を持つ 2 つの VLAN サブインターフェイス間で通信がブロックされる** | 同一セキュリティレベル間通信を不許可とするデフォルト動作 | <code>show running-config same-security-traffic</code> | **`same-security-traffic permit inter-interface`** コマンドを投入する。 |

---



## ⚠ 制限事項

* **予約済み VLAN ID の使用不可**:
  * **VLAN 1**: デフォルト VLAN。全ポートが初期所属。CDP/DTP/PVST+ などの制御トラフィックが通過するため、ユーザーデータ VLAN としての使用は非推奨。
  * **VLAN 1002～1005**: Token Ring / FDDI 用のシスコ予約 VLAN（変更・削除不可）。
  * **VLAN 4094 (モデル依存)**: 内部システム処理用予約 VLAN。
* **拡張 VLAN (Extended Range VLAN 1006～4094) の制限**:
  * VTP v1/v2 では拡張 VLAN を伝播できません（VTP Transparent モードでのみローカル作成可能）。VTP v3 を使用することで VTP 経由での自動伝播が可能となります。
* **ASA / FTD での Native VLAN 制限**:
  * ASA / FTD の 802.1Q サブインターフェイス構成では、物理インターフェイス自体（Untagged）とサブインターフェイス（Tagged）を混在させる場合、物理インターフェイスに割り当てられた IP が Native VLAN として動作します。

---

## 🔄 他技術との関連

* **Private VLANs (PVLANs)**: VLAN 内をさらに Primary, Isolated, Community ポートに再分割し、L2 分離を高度化する技術（Blueprint 2.5 該当）。
* **802.1X / MAB ＆ Cisco ISE**: 認証結果に基づき RADIUS VSA / IETF 属性を用いてポートに動的に VLAN をアサインするゼロトラストアクセス制御基盤。
* **Cisco ASA / FTD Access Control**: VLAN サブインターフェイスごとに Security Level や Security Zone を割り当て、インター VLAN トラフィックに対してステートフルインスペクションおよび IPS スキャンを実行。
* **VXLAN (Virtual Extensible LAN)**: 12 ビットの VLAN ID 制限（4094 個）を超えて、24 ビットの VNI（1600 万個）による L2 Over L3 オーバーレイセグメンテーションを提供するデータセンター技術。

---

## 🧩 比較表

### 1. Access Port vs Trunk Port vs Dynamic Port (DTP)

| 比較項目 | Access Port | Trunk Port | Dynamic Port (DTP) |
| :--- | :--- | :--- | :--- |
| **802.1Q タグ** | 付与しない (Untagged) | **付与する (Tagged)** ※Native VLAN 除く | ネゴシエーション結果に依存 |
| **所属可能 VLAN 数** | 1 個（＋ Voice VLAN 1 個） | **複数 (1～4094)** | ネゴシエーション結果に依存 |
| **DTP 交渉** | 送信しない (`nonegotiate`) | 送信しない (`nonegotiate`) | **`dynamic desirable` / `auto`** で送信 |
| **推奨用途** | ユーザー PC、プリンタ、IP Phone | スイッチ間、ASA/FTD/ESXi 接続 | **非推奨 (セキュリティ脅威のため排除)** |

### 2. VTP v1/v2 vs VTP v3

| 比較項目 | VTP v1 / v2 | VTP v3 |
| :--- | :--- | :--- |
| **拡張 VLAN (1006-4094)** | **非対応** (Transparent モードのみローカル可) | **対応** (全域へ自動伝播可能) |
| **データベース誤上書き防止** | リビジョン番号が大きいと無条件上書き (リスク高) | **Primary Server の概念を導入** (暗黙上書きを阻止) |
| **認証方式** | パスワードのみ | **パスワード ＋ Secret Key (Hidden 記録対応)** |
| **PVLAN 伝播** | 非対応 | **対応** (Private VLAN 構成情報を自動伝播) |

---

## 💡 ベストプラクティス

1. **VLAN Hopping ＆ Double Tagging 対策の徹底 (Hardening 3 原則)**:
   * ① 全トランクポートで `vlan dot1q tag native` を有効化する。
   * ② Native VLAN を未使用の専用 VLAN（例: VLAN 99）にアサインし、VLAN 1 をトランクから Pruning する。
   * ③ 全アクセスポートで `switchport mode access` および `switchport nonegotiate` を明示設定する。
2. **未使用ポートの Parking Lot 設定**:
   * 使用していないスイッチポートはすべて専用の孤立 VLAN（例: VLAN 999 `PARKING`）へアサインし、`shutdown` 状態で管理する。
3. **VTP v3 の標準化運用**:
   * 大規模環境では勝手な VLAN データベースの上書きを防ぎ、拡張 VLAN を安全に自動伝播させるため、VTP v3 を採用し、パスワードを `hidden` オプションで保持する。

---



## 📝 ラボ学習・設定サンプル例

### 例1: Catalyst Switch での DTP 無効化と VLAN Hopping 攻撃の完全緩和

#### 問題
スイッチ SW1 の `GigabitEthernet1/0/10` を端末用アクセスポート（VLAN 10）として設定せよ。攻撃者による DTP ネゴシエーションを利用したトランク化攻撃（VLAN Hopping）を完全に防ぐ設定を投入せよ。

#### 要件
* 明示的に `access` モードに指定すること。
* DTP トランキングネゴシエーションを完全に停止すること (`nonegotiate`)。
* Spanning-Tree PortFast および BPDU Guard を有効化すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
vlan 10
 name DATA_VLAN

interface GigabitEthernet1/0/10
 description --- Security Hardened Access Port ---
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
```

#### 検証方法
* `show interfaces GigabitEthernet1/0/10 switchport` を実行し、`Administrative Mode: static access` および `Negotiation of Trunking: Off` と表示されることを確認する。

---

### 例2: Global Native VLAN Tagging ＆ トランク Pruning 構成

#### 問題
SW1 と SW2 間のトランクリンク `GigabitEthernet1/0/1` において、Native VLAN を未使用の `VLAN 99` に変更し、VLAN 10, 20, 99 のみを通信可能（Pruning）とせよ。また、二重タギング（Double Tagging）攻撃を防ぐため、Global Native VLAN Tagging を有効化せよ。

#### 要件
* `vlan dot1q tag native` を設定すること。
* トランクリンクで DTP を停止すること。

#### 設定例

```bash
! ===== SW1 / SW2 Configuration =====
vlan dot1q tag native

vlan 10
 name DATA_A
vlan 20
 name DATA_B
vlan 99
 name TRUNK_NATIVE

interface GigabitEthernet1/0/1
 description --- Secure Trunk Link ---
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,99
 switchport nonegotiate
 no shutdown
```

#### 検証方法
* `show vlan dot1q tag native` で `dot1q native vlan tagging is enabled` を確認。
* `show interfaces GigabitEthernet1/0/1 trunk` で `Native vlan` が `99`、`Vlans allowed on trunk` が `10,20,99` であることを確認。

---

### 例3: VTP v3 ドメイン認証 ＆ Primary Server 昇格構成

#### 問題
スイッチ SW1 を VTP v3 Primary Server、SW2 を VTP v3 Client/Server として構築せよ。ドメイン名を `CCIE_LAB`、隠しパスワードを `SecretPass123` とし、VLAN 100～105 を新規作成・伝播させよ。

#### 要件
* VTP Version 3 を使用すること。
* 暗号化パスワード（`hidden`）を設定すること。

#### 設定例

```bash
! ===== SW1 / SW2 Global Configuration =====
vtp domain CCIE_LAB
vtp version 3
vtp mode server vlan
vtp password SecretPass123 hidden

! ===== SW1 EXEC Mode (Primary Server への昇格) =====
SW1# vtp primary vlan
! (パスワードプロンプトが表示された場合は SecretPass123 を入力)

! ===== SW1 上での VLAN 作成 =====
SW1(config)# vlan 100,101,102,103,104,105
```

#### 検証方法
* SW1 および SW2 で `show vtp status` を実行し、`VTP Operating Mode: Server`、`VTP Domain Name: CCIE_LAB`、および SW2 側で VLAN 100～105 が同期されていることを確認。

---

### 例4: Cisco ASA 802.1Q サブインターフェイス ＆ インター VLAN ルーティング

#### 問題
ASA1 の物理ポート `GigabitEthernet0/0` 上にサブインターフェイス `.10` (VLAN 10, Security Level 100) および `.20` (VLAN 20, Security Level 100) を作成せよ。同一 Security Level 同士のサブインターフェイス間での通信を許可せよ。

#### 要件
* `same-security-traffic permit inter-interface` を使用すること。

#### 設定例

```bash
! ===== ASA1 Configuration =====
interface GigabitEthernet0/0
 no shutdown

interface GigabitEthernet0/0.10
 vlan 10
 nameif INSIDE_CORP
 security-level 100
 ip address 10.1.10.1 255.255.255.0

interface GigabitEthernet0/0.20
 vlan 20
 nameif INSIDE_ENG
 security-level 100
 ip address 10.1.20.1 255.255.255.0

same-security-traffic permit inter-interface
```

#### 検証方法
* `show interface ip brief` で両サブインターフェイスが `UP/UP` であることを確認し、VLAN 10 の端末から 10.1.20.1 宛てに Ping が通り、ASA の接続テーブルが生成されることを確認。

---

### 例5: Cisco FTD サブインターフェイス ＆ FMC Security Zone マッピング

#### 問題
FMC 管理下の FTD1 アプライアンスにおいて、物理ポート `GigabitEthernet0/0` 上に 802.1Q サブインターフェイス `Gi0/0.30` (VLAN 30) を作成し、Security Zone `Zone_DMZ` に割り当てよ。

#### 要件
* FMC GUI からアサインすること。

#### 設定例

```text
[ FMC GUI Configuration Steps ]
1. Devices > Device Management ➔ 対象 FTD1 の Edit (鉛筆アイコン) をクリック。
2. Interfaces タブ ➔ GigabitEthernet0/0 を選択し "Add Subinterface" をクリック:
   - SubInterface ID: 30
   - VLAN ID: 30
   - Security Zone: "Zone_DMZ" を新規作成または選択
   - Mode: Routed
   - IPv4 Address: 10.1.30.1 / 24
3. Save をクリックし、対象 FTD1 へ Deploy を実行。
```

#### 検証方法
* FTD1 CLI で `show interface GigabitEthernet0/0.30` を実行し、`VLAN Identifier 30` および IP アドレス `10.1.30.1` がアサインされていることを確認。

---

### 例6: Cisco ISE 動的 VLAN アサイン (802.1X / MAB RADIUS 属性 64/65/81)

#### 問題
スイッチ SW1 の `GigabitEthernet1/0/5` で MAB 認証を有効化せよ。認証成功時、Cisco ISE から RADIUS IETF 属性（64, 65, 81）を受領してポートの通信 VLAN を動的に `VLAN 50`（`GUEST_VLAN`）に変更させよ。

#### 要件
* あらかじめスイッチ上に `vlan 50` が定義されていること。

#### 設定例

```bash
! ===== SW1 Configuration =====
vlan 50
 name GUEST_VLAN

aaa new-model
aaa authentication dot1x default group radius
aaa authorization network default group radius

radius server ISE_NODE
 address ipv4 192.168.1.100 auth-port 1812 acct-port 1813
 key SecretKey123

interface GigabitEthernet1/0/5
 description --- Dynamic MAB Port ---
 switchport mode access
 switchport access vlan 999             ! 初期拒否 VLAN
 authentication port-control auto
 mab
 dot1x pae authenticator
 no shutdown
```

#### 検証方法
* MAB 認証成功後、`show authentication sessions interface GigabitEthernet1/0/5` を実行し、`Assigned VLAN: 50` または `Assigned VLAN: GUEST_VLAN` と表示されることを確認。

---

### 例7: Voice VLAN ＆ Auxiliary VLAN セキュリティ構成

#### 問題
スイッチ SW1 の `GigabitEthernet1/0/12` に Cisco IP Phone と PC を共有接続する。データ用に VLAN 10、音声用に VLAN 20 (Voice VLAN) を割り当て、DTP ネゴシエーションを完全に無効化せよ。

#### 要件
* `switchport voice vlan` コマンドを使用すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
vlan 10
 name DATA_USERS
vlan 20
 name VOICE_PHONES

interface GigabitEthernet1/0/12
 description --- IP Phone + PC Shared Port ---
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
```

#### 検証方法
* `show interfaces GigabitEthernet1/0/12 switchport` で `Access Mode VLAN: 10` および `Voice VLAN: 20` と表示されることを確認。

---

### 例8: VLAN Access Control List (VACL) による同一 VLAN 内 L2 フィルタリング

#### 問題
VLAN 10 内に存在するサーバー `10.1.10.100` に対し、同 VLAN 内の端末 `10.1.10.200` からの SSH (TCP 22) アクセスのみを L2 レベルでドロップし、その他の通信は同一 VLAN 内で許可する VACL を構成せよ。

#### 要件
* `vlan access-map` および `vlan filter` を使用すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
ip access-list extended ACL_VACL_BLOCK_SSH
 permit tcp host 10.1.10.200 host 10.1.10.100 eq 22

vlan access-map VACL_NO_SSH 10
 match ip address ACL_VACL_BLOCK_SSH
 action drop

vlan access-map VACL_NO_SSH 20
 action forward

vlan filter VACL_NO_SSH vlan-list 10
```

#### 検証方法
* `10.1.10.200` から `10.1.10.100` 宛てに SSH を実行し、接続がタイムアウト（Drop）することを確認。ICMP Ping など他の通信は正常に通ることを確認。

---

### 例9: Switchport Security ＆ VLAN 指定 MAC アドレスバインド

#### 問題
スイッチ SW1 の VLAN 10 ポート `GigabitEthernet1/0/15` において、MAC アドレス固定登録（Sticky MAC）を有効化せよ。最大許可 MAC アドレス数を 1 とし、違反発生時はポートを即座に `err-disable` 状態（Violation Mode: Shutdown）にせよ。

#### 要件
* `switchport port-security` を使用すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
interface GigabitEthernet1/0/15
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation shutdown
 switchport port-security mac-address sticky
 no shutdown
```

#### 検証方法
* 別端末を接続した際、ポートが即座に `err-disabled` 状態へ遷移することを確認 (`show interfaces GigabitEthernet1/0/15 status`)。

---

### 例10: Catalyst L3 Switch SVI インター VLAN ルーティング ＆ アクセス制限 ACL

#### 問題
スイッチ SW1 上で VLAN 10 (SVI: `10.1.10.1/24`) および VLAN 20 (SVI: `10.1.20.1/24`) の SVI を作成し、L3 ルーティングを有効化せよ。VLAN 20 から VLAN 10 の管理 IP `10.1.10.1` 宛ての Telnet (TCP 23) のみをブロックする ACL をアサインせよ。

#### 要件
* `ip routing` を有効化すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
ip routing

vlan 10
vlan 20

interface Vlan10
 description --- SVI Corporate Data ---
 ip address 10.1.10.1 255.255.255.0
 no shutdown

ip access-list extended ACL_BLOCK_TELNET_MGMT
 deny tcp 10.1.20.0 0.0.255.255 host 10.1.10.1 eq 23
 permit ip any any

interface Vlan20
 description --- SVI Guest Data ---
 ip address 10.1.20.1 255.255.255.0
 ip access-group ACL_BLOCK_TELNET_MGMT in
 no shutdown
```

#### 検証方法
* VLAN 20 端末から `10.1.10.1` 宛てに Telnet を試み、拒否（`Connection refused` / `Destination unreachable`）されることを確認。

---



## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: スイッチ SW1 で以下のコンフィグが投入されている。攻撃者が VLAN 10 に所属するアクセスポートから Double-Tagged パケット（アウタータグ: VID 10, インナータグ: VID 20）を送信した場合、スイッチおよびトランクリンクの処理に関する記述として正しいものはどれか。

```bash
vlan dot1q tag native

interface GigabitEthernet1/0/1
 switchport mode trunk
 switchport trunk native vlan 10
 switchport trunk allowed vlan 10,20
```

* A) Native VLAN パケットに 802.1Q タグが付与されないため、アウタータグが剥ぎ取られて VLAN 20 へパケットが侵入する（VLAN Hopping 成功）。
* B) `vlan dot1q tag native` が有効なため、Native VLAN 10 のフレームにも 802.1Q タグが強制保持・付与され、攻撃パケットは VLAN 20 へ浸透せずドロップ/分離される。
* C) トランクポートで `switchport nonegotiate` が無効なため、パケットは即座に拒否される。
* D) DTP ネゴシエーションにより、トランクポートが自動的にアクセスポートへダウングレードされる。

**正解**: **B**  
**解説**: 通常の 802.1Q トランク動作では、Native VLAN（ここでは VLAN 10）宛てのフレームはタグなしで送信されるため、Outer Tag VID 10 が剥がされて Inner Tag VID 20 が露呈し VLAN Hopping が発生します。しかし **`vlan dot1q tag native`** がグローバルで設定されている場合、Native VLAN パケットにも 802.1Q タグが強制付与されるため、Outer Tag は剥がされず、VLAN Hopping 攻撃は完全に無効化されます。

---

### 2. トラブルシュート問題

**問題**: Cisco ISE からスイッチポートへ動的 VLAN アサイン（VLAN ID: `50`）をプッシュする Authorization Profile を適用した。認証ログでは `Access-Accept` が返却され、`Tunnel-Private-Group-ID = 50` が受領されているが、スイッチポートが `Auth-Fail` 状態となり通信が不通となる。スイッチ側で確認すべき最重要原因はどれか。

* A) スイッチポートで `switchport mode trunk` が設定されている。
* B) スイッチ上に `vlan 50` があらかじめ定義（作成）されていない。
* C) ISE 側で AAA 共有キーが間違っている。
* D) ポートで CDP が無効になっている。

**正解**: **B**  
**解説**: スイッチ（IOS / IOS-XE）で RADIUS から動的に降ってきた VLAN ID / Name をポートへ適用するためには、**その VLAN がスイッチの VLAN データベース上にローカル定義（`vlan 50`）されている必要があります**。存在しない VLAN を受領した場合、スイッチは動的 VLAN の適用に失敗し、ポートを非アクティブ/拒否状態にします。

---

### 3. Design 問題

**問題**: エンタープライズ Campus ネットワークにおいて、同一 VLAN 100（`10.1.100.0/24`）内に存在する医療端末同士の直接の L2 通信（横移動）を遮断しつつ、デフォルトゲートウェイ（ルーター/SVI）宛ての通信のみを許可したい。最も追加機器コストが少なく、かつ L2 レベルで通信を制限できる制御技術はどれか。

* A) ASA ファイアウォールを透過モードで増設する。
* B) スイッチ上で **VACL（VLAN Access Control List）** または **Private VLANs (Isolated Port)** を適用する。
* C) SVI インターフェイスに Inbound Router ACL を適用する。
* D) ポートごとに独立した VLAN ID を再割り当てし、全端末を別サブネットへ移動する。

**正解**: **B**  
**解説**: 同一 VLAN / 同一 L2 サブネット内でのホスト間通信は、レイヤー 3 ルーターや通常の SVI ACL を経由しないため、標準 Router ACL ではブロックできません。**VACL（VLAN Filter）** または **Private VLAN（Isolated Port）** を使用することで、追加のファイアウォールを導入することなくスイッチ内部で L2 フィルタリングが可能です。

---

### 4. 実装問題

**問題**: Cisco ASA において、サブインターフェイス `GigabitEthernet0/0.100` (VLAN 100) と `GigabitEthernet0/0.200` (VLAN 200) を構成した。両サブインターフェイスの Security Level を共に `100` に設定した場合、サブインターフェイス間での双方向通信を可能にするために必要な設定コマンドはどれか。

* A) `same-security-traffic permit intra-interface`
* B) `same-security-traffic permit inter-interface`
* C) `no sysopt connection permit-vpn`
* D) `access-group ACL_ALLOW in interface GigabitEthernet0/0`

**正解**: **B**  
**解説**: ASA では、**異なるインターフェイス間だが同一の Security Level**（例: 共に 100）を持つインターフェイス間での通信はデフォルトでブロックされます。これを許可するには、グローバル設定で **`same-security-traffic permit inter-interface`** を投入する必要があります（※`intra-interface` は同一ポート内での折返しヘアピン通信用）。

---

### 5. コンフィグ読解問題

**問題**: VTP v3 の設定において、以下のコマンドを投入したのにもかかわらず、SW1 上で新規 VLAN の追加（`vlan 300`）を実行しようとすると `VTP VLAN configuration not allowed when device is not the primary server` というエラーが発生した。原因として正しいものはどれか。

```bash
SW1(config)# vtp domain CORP_LAB
SW1(config)# vtp version 3
SW1(config)# vtp mode server vlan
```

* A) VTP パスワードが未設定であるため。
* B) VTP v3 では `vtp mode server` 設定後に、EXEC モードで **`vtp primary vlan`** を実行して自機を Primary Server に昇格させる必要があるため。
* C) 拡張 VLAN 300 は VTP v3 でサポートされていないため。
* D) SW1 のリビジョン番号が 0 のままであるため。

**正解**: **B**  
**解説**: VTP v3 では、単に `vtp mode server` に設定しただけではデータベースの作成・編集権限が与えられません。勝手なデータベース上書き事故を防ぐため、特権 EXEC モードで **`vtp primary vlan`** コマンドを実行し、自スイッチを明示的に Primary Server に昇格させる手順が必要です。

---

## 🔗 参考リソース

* [Cisco Catalyst 9300 Series Switches Configuration Guide: VLAN Configuration](https://www.cisco.com/)
* [Cisco Catalyst 9300 Series Switches Configuration Guide: VTP Configuration](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Device Configuration Guide: Subinterfaces](https://www.cisco.com/)
* [Cisco ASA Series General Operations Configuration Guide: Interface Objects and Subinterfaces](https://www.cisco.com/)
* [Cisco Technical Notes: Mitigate VLAN Hopping Attacks on Cisco Catalyst Switches](https://www.cisco.com/)
* [Cisco Live: Enterprise Campus LAN Security Best Practices (BRKSEC-2001)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で VLAN や L2 セグメンテーションが出題された場合の迅速チェックフロー：
    * **「Secure all access ports from VLAN Hopping」** ➔ `switchport mode access` ＋ `switchport nonegotiate` ＋ Unused ports to Parking VLAN 999 (`shutdown`)。
    * **「Mitigate Double Tagging attack」** ➔ Global `vlan dot1q tag native` ＋ Change Trunk Native VLAN from 1 to 99。
    * **「Configure VTP v3 Server」** ➔ `vtp domain X` ➔ `vtp version 3` ➔ `vtp mode server vlan` ➔ EXEC mode: `vtp primary vlan`。
    * **「Inter-VLAN Routing on ASA with Same Security Level」** ➔ `vlan <ID>` on Subinterfaces ➔ `same-security-traffic permit inter-interface`。
    * **「Dynamic VLAN from ISE」** ➔ RADIUS Attributes 64 (`VLAN`), 65 (`802`), 81 (`VLAN_ID/Name`) ➔ Pre-create VLAN on Switch (`vlan <ID>`)。
* **図解（VLAN パケット処理パイプライン概要）**:
  ```text
  [ Ingress Access Port ] ──► Assign PVID (e.g. VLAN 10)
                                     │
  [ Egress Trunk Port ]   ──► VID == Native VLAN?
                                     ├── YES ──► Send Untagged (or Tagged if 'vlan dot1q tag native' is enabled)
                                     └── NO  ──► Insert 4-Byte 802.1Q Header (VID 10) ──► Transmit Frame
  ```
* **注意点**:
  * ラボ試験で VLAN やサブインターフェイスを設定した後は、必ず `show vlan brief`、`show interfaces trunk`、`show authentication sessions`（ISE アサイン時）、および `show interface ip brief`（ASA/FTD 時）を実行し、パケットが意図した VLAN でタギング・フォワーディングされていることを確認してください。


