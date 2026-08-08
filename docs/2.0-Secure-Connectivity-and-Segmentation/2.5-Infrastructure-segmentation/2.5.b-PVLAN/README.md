---
layout: default
title: 2.5.b-PVLAN
nav_order: 2
parent: 2.5-Infrastructure-segmentation
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.5.b PVLAN (Private VLAN)

インフラストラクチャのセグメンテーションにおいて、**PVLAN（プライベートVLAN）**は、同一のレイヤ3サブネット（VLAN）内でありながら、レイヤ2レベルでのさらなる分離を実現する高度なセキュリティ機能です。これにより、デフォルトゲートウェイへの通信は許可しつつ、同じサブネット内の他端末との通信を制限することが可能になります。CCIE Security v6.1のブループリントでは、インフラセグメンテーション手法の主要技術として位置付けられています。

---

## 📘 概要

*   **機能概要**: 1つの「Primary VLAN」を複数の「Secondary VLAN」に分割し、ポート間の通信を制御します。
*   **利用目的**: レイヤ2での「ゼロトラスト」環境の構築。特定のホスト間通信のみを許可し、不要な横方向の通信（East-Westトラフィック）を遮断します。
*   **どのような場面で利用するか**: 
    *   **DMZ**: 公開サーバー同士の通信を禁止し、1台が侵害されても他への影響を最小限にする。
    *   **共有ホスティング/マンションISP**: 同じサブネットを利用する顧客端末同士を隔離する。
    *   **バックアップ・管理ネットワーク**: 管理端末からのみアクセスを許可し、サーバー間の通信は不要な場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **Primary VLAN** | Secondary VLANを包含するベースのVLAN。トラフィックを下流へ転送する。 |
| **Secondary VLAN** | Primary内部で動作。**Isolated** または **Community** のいずれか。 |
| **Promiscuous Port** | 全ポートと通信可能な「無差別」ポート。通常、ルータやFWを接続。 |
| **Isolated Port** | 同一VLAN内でも、Promiscuousポート以外とは一切通信できない。 |
| **Community Port** | 同じコミュニティ内のポートおよびPromiscuousポートと通信可能。 |
| **VTP モード** | **Transparent** または **Off** である必要がある。 |

---

## 🏗 動作原理

PVLANは、入力ポートと出力ポートのペアによってトラフィックを許可するかドロップするかを判断します。

```text
[ Promiscuous Port (GW/Router) ]
       ↕ (Communication OK with ALL)
------------------------------------------
[ Community A ]   [ Community B ]   [ Isolated ]
  - Host 1          - Host 3          - Host 5
  - Host 2          - Host 4          - Host 6
      ↕                 ↕                 ↕
 (1-2 OK)          (3-4 OK)          (All Blocked)
 (1-3 Blocked)     (3-1 Blocked)     (5-6 Blocked)
```

---

## ⚙ 動作シーケンス

1.  **フレーム受信**: スイッチポートがSecondary VLAN設定でフレームを受信。
2.  **タグ付け**: 内部的にSecondary VLAN IDが付与される。
3.  **転送判断**: 宛先MACを学習しているポートが同じCommunityか、Promiscuousポートであれば転送。
4.  **VLAN変換**: Promiscuousポートから送出される際、タグはPrimary VLAN IDにマッピングされる。
5.  **逆方向**: Promiscuousポートから入ったPrimary VLANのパケットは、マッピングされた全Secondary VLANへ転送可能。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **VTP Transparentの強制**: PVLANを設定する前に必ず `vtp mode transparent` を実行してください。これを忘れるとPVLANコマンドが拒否されます。
*   **Associationの確実な設定**: `private-vlan association` コマンドで、PrimaryとSecondaryを論理的に紐付ける手順が最重要です。
*   **SVIの作成箇所**: L3インターフェイス（Gateway）は **Primary VLAN** に対してのみ作成します。Secondary VLANにSVIを作成しても機能しません。
*   **PVLAN over Trunk**: トランクポートでPVLANを運ぶ際、PrimaryとSecondaryの両方のVLAN IDが許可されている必要があります。
*   **トラブルシュート問題**: 「Host AとBは通信できるが、Host AとCはできない。ただし全員GWとは通信できる」という要件を、CommunityとIsolatedの使い分けで実装させる問題が想定されます。

---

## 🛠 設定方法

### 1. VLANの定義と関連付け
```bash
vtp mode transparent

! Secondary VLAN (Isolated)
vlan 101
 private-vlan isolated

! Secondary VLAN (Community)
vlan 102
 private-vlan community

! Primary VLAN
vlan 10
 private-vlan primary
 private-vlan association 101,102
```

### 2. ポートの割り当て
```bash
! ホスト接続（Isolated）
interface GigabitEthernet0/1
 switchport mode private-vlan host
 switchport private-vlan host-association 10 101

! ホスト接続（Community）
interface GigabitEthernet0/2
 switchport mode private-vlan host
 switchport private-vlan host-association 10 102

! Promiscuous（ルータ/FW接続）
interface GigabitEthernet0/24
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 10 101,102
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **PVLAN構成の全体確認** | <code>show vlan private-vlan</code> |
| **ポートのマッピング確認** | <code>show interface private-vlan mapping</code> |
| **VLANタイプとステータス** | <code>show vlan private-vlan type</code> |
| **インターフェイス設定詳細** | <code>show running-config interface [ID]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| コマンドが入力できない | VTPモードがServer/Client | <code>vtp mode transparent</code>に変更。 |
| ホストからGWに届かない | Promiscuousマッピング漏れ | <code>mapping</code>コマンドに当該Secondary IDが含まれているか確認。 |
| Community間で通信不可 | VLAN IDが異なる | 同じCommunity VLAN IDに属しているか確認。 |
| ARPは届くが通信不可 | Primary SVIの未作成 | <code>interface vlan [Primary]</code>を作成しIPを付与。 |

---

## ⚠ 制限事項

*   **EtherChannel**: PVLANポートはEtherChannelのメンバーになれない場合があります（ハードウェア依存）。
*   **SPAN**: PVLAN IsolatedポートをDestinationとするSPANは制限されます。
*   **DHCP Snooping**: PVLAN環境下では設定に注意が必要（Primaryでの有効化が基本）。

---

## 🔄 他技術との関連

*   **VACL (3.4.g)**: PVLANと併用して、さらにIP/ポートベースのフィルタを適用可能。
*   **TrustSec (2.6)**: PVLANよりも柔軟なSGTベースのマイクロセグメンテーションへの発展。
*   **Promiscuous VTI**: VPN終端デバイスをPVLANのPromiscuousポートとして動作させる設計。

---

## 🧩 比較表

### Isolated vs Community

| 特徴 | Isolated VLAN | Community VLAN |
| :--- | :--- | :--- |
| **同一VLAN内通信** | **不可** | **可能** |
| **GW(Promiscuous)通信** | 可能 | 可能 |
| **別Secondary通信** | 不可 | 不可 |
| **主な用途** | バックアップ、ゲスト用 | 部門内、サーバーファーム |

---

## 💡 ベストプラクティス

1.  **VLAN番号の計画**: PrimaryとSecondaryの連番を避け、予期せぬ管理ミスを防ぎます。
2.  **デフォルトゲートウェイ**: 冗長化（HSRP/VRRP）を行う場合、HSRPのHelloもPVLANを通過できるようにPromiscuousマッピングを正しく行います。
3.  **ドキュメント化**: どのVLANがPrimaryでどれがSecondaryかを明示的にスイッチの `description` 等に記述することを推奨します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な Isolated 設定
*   **要件**: VLAN 500(P) 内でポート Gi0/1 と Gi0/2 を完全に隔離せよ。
*   **設定**: `vlan 501 (isolated)`, `vlan 500 (primary) association 501`.

### 2. Community を使用した部門分離
*   **要件**: VLAN 10 内で「開発」と「営業」を分け、各グループ内のみ通信させよ。

### 3. ASA を Promiscuous に接続
*   **要件**: ASA Outside を PVLAN の出口として構成せよ。

### 4. Catalyst 9k での SVI マッピング
*   **要件**: Primary VLAN 100 に IP 192.168.1.1 を付与し GW とせよ。

### 5. PVLAN over Trunk (Multiple Switches)
*   **要件**: スイッチ 1 とスイッチ 2 の間で PVLAN の設定を維持したまま転送せよ。

### 6. PVLAN と Port-Security の併用
*   **要件**: Isolated ポートで MAC 学習数を 1 に制限せよ。

### 7. PVLAN 環境での HSRP
*   **課題**: 2台のコアスイッチで HSRP を組み、Promiscuous マッピングを同期せよ。

### 8. VACL による追加フィルタ
*   **要件**: Isolated ポートからの DNS (UDP 53) のみを許可せよ。

### 9. Secondary VLAN ID の変更手順
*   **課題**: 既存の PVLAN 構成を維持したまま Secondary ID を変更する。

### 10. PVLAN Edge (Protected Port) との比較検証
*   **要件**: 単一スイッチ内での `switchport protected` との動作差異を確認せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `switchport private-vlan mapping 10 101` が Promiscuous ポートに設定されている。VLAN 102 のホストはこのポートと通信できるか？
    *   **回答**: できない。VLAN 102 もマッピングに含める必要がある。
2.  **トラブルシュート**: `vlan 101` に `private-vlan isolated` を設定しようとしたがエラーが出た。確認すべき VTP の設定は？
    *   **回答**: `show vtp status` でモードを確認し、`transparent` でない場合は変更する。
3.  **Design**: 同一サブネット内の特定の 5 台のサーバー間のみ通信を許可し、他とは隔離したい。どの PVLAN タイプを使うべきか？
    *   **回答**: Community VLAN。
4.  **実装**: 2台のスイッチをまたいで Isolated ポートを配置する場合、トランクリンクで必要な設定は？
    *   **回答**: 通常の 802.1Q トランク。Primary および Secondary 両方の VLAN ID をパスさせる。
5.  **コンフィグ読解**: `switchport mode private-vlan host` ポートに `switchport access vlan 10` を設定した場合の挙動は？
    *   **回答**: PVLAN 設定が優先され、`access vlan` 設定は無視される（あるいは矛盾として無視される）。

---

## 🔗 参考リソース

*   **Cisco Catalyst 9000 シリーズ 構成ガイド**
    *   [Configuring Private VLANs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-3/configuration_guide/vlan/b_173_vlan_9300_cg.html)
*   **Cisco Live 資料**
    *   [BRKSEC-2202: Layer 2 Security In-Depth](https://www.ciscolive.com/)
*   **テクニカルノート**
    *   [Private VLANs (PVLANs) - Common Issue Troubleshooting](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-6500-series-switches/10592-20.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「Primary = 親、Secondary = 子」の関係を常に意識してください。
*   **図解**: データの流れを「VLAN ID の書き換え」として捉えると、Promiscuous マッピングの必要性が理解しやすくなります。
*   **注意点**: ラボ試験の Catalyst スイッチでは、PVLAN は標準機能ですが、OS バージョンによって CLI の挙動が微妙に異なる場合があります。常に `show vlan private-vlan` で結果を確認しましょう。
