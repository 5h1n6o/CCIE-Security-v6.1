---
layout: default
title: 2.5.a-VLAN
nav_order: 2
parent: 2.5-Infrastructure-segmentation
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.5.a VLAN (Virtual Local Area Network)

インフラストラクチャのセグメンテーションにおいて、**VLAN** は最も基本的かつ不可欠なレイヤ2分離技術です。物理的なネットワークインフラストラクチャを共有しながら、ブロードキャストドメインを論理的に分割することで、セキュリティの向上、トラフィックの管理、およびリソースの効率的な利用を実現します。CCIE Security ラボ試験では、単なる VLAN の作成だけでなく、VLAN ホッピング攻撃の防止、Firepower (FTD) や ASA とのトランク接続、および VACL を用いた詳細なフィルタリングの実装能力が問われます。

---

## 📘 概要

*   **機能概要**: 物理スイッチ上のポートを論理的なグループに分割し、それぞれを独立した LAN（ブロードキャストドメイン）として動作させる技術。
*   **利用目的**: セキュリティ境界の構築、ブロードキャストトラフィックの抑制、ネットワーク構成の柔軟な変更。
*   **どのような場面で利用するか**: 
    *   部門間（人事、財務、エンジニアリングなど）のトラフィックを分離する場合。
    *   DMZ、ゲストネットワーク、管理用ネットワークを物理的に同一のスイッチ上に共存させる場合。
    *   IP 電話（Voice VLAN）とデータ端末を同一ポートで分離する場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | レイヤ2（データリンク層）での論理的なネットワーク分離。 |
| **用途** | セグメンテーション、トラフィック制御、セキュリティ境界の定義。 |
| **メリット** | 物理構成に縛られない柔軟性、ブロードキャストドメインの縮小による性能向上。 |
| **デメリット** | 設定ミスによる VLAN 間通信の不備、VLAN ホッピングなどの L2 攻撃リスク。 |
| **対応機種** | Cisco Catalyst スイッチ、Cisco ルータ、Cisco ASA/FTD 等のセキュリティ製品。 |
| **制限事項** | VLAN ID の範囲（標準: 1-1005, 拡張: 1006-4094）。 |
| **設計上の注意点** | トランクポートでの Native VLAN の一致、不要な VLAN のプルーニング。 |

---

## 🏗 動作原理

VLAN は、イーサネットフレームに **IEEE 802.1Q** タグを挿入することで、パケットがどの論理ネットワークに属するかを識別します。

```text
Host A (VLAN 10)                               Host B (VLAN 20)
       ↓                                              ↑
[ Access Port (VLAN 10) ]                      [ Access Port (VLAN 20) ]
       ↓                                              ↑
[ Switch A ] ----------( Trunk Link / Tagged )-------- [ Switch B ]
                 (Frame + VLAN Tag 10/20)
```

---

## ⚙ 動作シーケンス

1.  **フレーム受信**: スイッチがアクセスポートからアンタグ（Untagged）フレームを受信。
2.  **VLAN 割り当て**: 受信ポートに設定された VLAN ID を内部的に付与。
3.  **転送判断**: 同じ VLAN ID を持つポート、またはトランクポートへ転送。
4.  **トランク転送**: トランクポートを通過する際、802.1Q タグが付加される。
5.  **フレーム配送**: 宛先スイッチがタグを読み取り、対応する VLAN のアクセスポートからタグを除去して送信。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **VLAN ホッピングの防御**: `switchport mode access` の明示的な設定、および未使用ポートの `shutdown` とダミー VLAN への割り当ては必須のセキュリティ対策です。
*   **ASA/FTD とのトランク接続**: ラボ試験では、ASA や FTD の物理インターフェイスをサブインターフェイスに分割し、複数の VLAN（Inside, Outside, DMZ 等）を収容する構成が頻繁に出題されます。
*   **Native VLAN のセキュリティ**: トランクリンクの両端で Native VLAN ID を一致させること、および Native VLAN をデフォルトの 1 以外に変更し、かつタグなしトラフィックを許可しない設計が求められます。
*   **VACL (VLAN ACL)**: ルータやファイアウォールを経由しない、同一 VLAN 内の端末間通信を L2 レベルでフィルタリングする設定（`vlan access-map`）は、インフラセキュリティの重要トピックです。
*   **VLAN Pruning**: VTP を使用している場合（または手動で）、不要な VLAN トラフィックがトランクリンクを流れないように制限する能力が問われます。

---

## 🛠 設定方法

### 1. スイッチでの基本設定
```bash
! VLANの作成
vlan 10
 name SALES
vlan 20
 name GUEST
!
! アクセスポートの設定
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
!
! トランクポートの設定
interface GigabitEthernet0/24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20
```

### 2. ASA サブインターフェイス設定 (Trunk 接続時)
ASA はサブインターフェイスを使用して VLAN を終端します。
```bash
interface GigabitEthernet0/1.10
 vlan 10
 nameif inside
 security-level 100
 ip address 10.1.10.1 255.255.255.0
!
interface GigabitEthernet0/1.20
 vlan 20
 nameif dmz
 security-level 50
 ip address 10.1.20.1 255.255.255.0
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **VLAN の作成状態とポート割り当て確認** | <code>show vlan brief</code> |
| **トランクポートの状態、Native VLAN 確認** | <code>show interfaces trunk</code> |
| **特定のポートのレイヤ2設定詳細確認** | <code>show interfaces [id] switchport</code> |
| **ASA の VLAN 収容状態確認** | <code>show interface ip brief</code> |
| **VACL の適用状態確認** | <code>show vlan filter</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| VLAN 間通信ができない | ルーティング（L3）の欠如 | <code>show ip route</code> | デフォルトゲートウェイ（SVI等）の設定を確認。 |
| トランク越しに通信不可 | VLAN ID の不一致または Allowed List | <code>show int trunk</code> | 対向と許可 VLAN リストを揃える。 |
| STP ブロッキングが発生 | Native VLAN ミスマッチ | <code>show logging</code> | <code>%CDP-4-NATIVE_VLAN_MISMATCH</code> を確認し修正。 |
| 特定 VLAN のみ疎通なし | VLAN が <code>shutdown</code> 状態 | <code>show vlan</code> | 当該 VLAN 設定内で <code>no shutdown</code>。 |

---

## ⚠ 制限事項

*   **スケーラビリティ**: 802.1Q で定義される VLAN 数は 4094 までであり、大規模なマルチテナント環境では不足する場合があります（VXLAN 等の検討が必要）。
*   **プラットフォーム依存**: 低価格帯のスイッチでは、拡張 VLAN（1006-4094）がサポートされない、または透明モード（VTP Transparent）のみのサポートとなる場合があります。
*   **ハードウェアリソース**: VACL を多用すると、スイッチの TCAM（三状態連想メモリ）を消費し、パフォーマンスに影響を与える可能性があります。

---

## 🔄 他技術との関連

*   **PVLAN (2.5.b)**: VLAN 内部をさらに隔離する（隔離、コミュニティ、プロミスキャス）高度なセグメンテーション。
*   **VRF-Lite (2.5.d)**: 同一 VLAN 設定を持つ複数の論理ネットワークを、L3 レベルで完全に分離して運用する場合に併用。
*   **Access Control (1.1)**: VLAN 間の通信を ASA/FTD で制御するためのポリシー定義。
*   **STP Security (3.4.c)**: VLAN ごとのスパニングツリー（PVST+）の安定性を確保するための機能。

---

## 🧩 比較表

### Access Port vs Trunk Port

| 特徴 | Access Port | Trunk Port |
| :--- | :--- | :--- |
| **役割** | エンドデバイス（PC等）の接続。 | スイッチ間、ルータ・FW 間の接続。 |
| **タグの有無** | 基本的にアンタグ。 | **802.1Q タグが付加される。** |
| **VLAN 収容数** | 1つの VLAN（+ Voice VLAN）。 | 複数の VLAN を同時に転送可能。 |
| **主な用途** | ユーザー端末収容。 | VLAN の集約と伝搬。 |

---

## 💡 ベストプラクティス

1.  **未使用ポートの保護**: すべての未使用ポートは `shutdown` し、トラフィックを転送しない隔離用 VLAN に割り当てます。
2.  **Native VLAN の変更**: デフォルトの VLAN 1 を Native VLAN として使用せず、トラフィックが流れない特定の ID（例: 999）に変更します。
3.  **DTP の無効化**: トランクの自動ネゴシエーション（DTP）を無効にし、`switchport mode trunk` または `access` を明示的に設定して VLAN ホッピングを防ぎます。
4.  **VTP の適切な管理**: ラボ試験では混乱を避けるため、VTP モードを `Transparent` または `Off` に設定することが推奨されます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な VLAN セグメンテーション
*   **要件**: スイッチ上で VLAN 10 (Users), 20 (Servers) を作成し、各ポートを割り当てよ。
*   **設定**: `vlan 10`, `vlan 20`, `interface range Gi0/1-10`, `switchport access vlan 10`...

### 2. ASA との 802.1Q トランク構成
*   **要件**: ASA の Gi0/1 とスイッチ間をトランクにし、VLAN 101, 102 を ASA で終端せよ。

### 3. Native VLAN のミスマッチ修正
*   **課題**: ログに表示される Native VLAN Mismatch を検出し、設定を揃えて STP を安定させよ。

### 4. VACL による intra-VLAN ブロック
*   **要件**: VLAN 10 内で 10.1.10.5 から 10.1.10.10 への通信のみを拒否せよ。

### 5. FTD での VLAN サブインターフェイス実装
*   **要件**: FMC を使用して、FTD の物理インターフェイスに複数の VLAN タグ付きサブインターフェイスを作成せよ。

### 6. VLAN ホッピング攻撃のシミュレーションと対策
*   **課題**: DTP を悪用したトランク奪取を、`switchport nonegotiate` で防御せよ。

### 7. Voice VLAN の実装
*   **要件**: 同一ポートで PC (VLAN 10) と IP Phone (VLAN 100) を分離せよ。

### 8. VLAN プルーニングによる帯域最適化
*   **要件**: 特定のトランクリンクで VLAN 30 のトラフィックのみを明示的に禁止せよ。

### 9. 管理用 VLAN の隔離
*   **要件**: スイッチの管理 IP (SVI) を専用の VLAN 99 に配置し、他 VLAN からのアクセスを制限せよ。

### 10. Bridge-Group と VLAN の組み合わせ (ASA)
*   **要件**: ASA で VLAN 10 と 20 をブリッジし、L2 ファイアウォールとして動作させよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: スイッチ間でトランクを構成したが、特定の VLAN だけ通信できない。`show interfaces trunk` で確認すべき項目は？
    *   **回答**: `Vlans allowed on trunk` リストにその VLAN ID が含まれているか、および `Vlans in spanning tree forwarding state` になっているかを確認する。
2.  **Design**: VLAN ホッピング攻撃を防ぐために、トランクポートの Native VLAN 設定において推奨される対策は？
    *   **回答**: Native VLAN を未使用の VLAN ID に変更し、さらにトランクポートを通過するすべてのフレーム（Native VLAN 含む）にタグを強制する（`vlan dot1q tag native`）。
3.  **コンフィグ読解**: ASA の `interface GigabitEthernet0/0.100` に `vlan 100` が設定されている場合、対向スイッチポートの必須設定は？
    *   **回答**: 802.1Q トランキングが有効であり、VLAN 100 が許可されている必要がある。
4.  **実装**: 同一 VLAN 内の通信を IP アドレスに基づいて制限したい。ルータの ACL 以外で実現する方法は？
    *   **回答**: **VACL (VLAN ACL)** をスイッチ上で構成し、`vlan filter` を適用する。
5.  **Design**: 複数のスイッチで構成される大規模ネットワークで、VLAN データベースの整合性を保つための推奨設定は？
    *   **回答**: 設定の衝突を避けるため、VTP Transparent モードを使用し、手動で VLAN を作成・管理する。

---

## 🔗 参考リソース

*   **Cisco Catalyst 9000 Series Switches Configuration Guide**
    *   [Configuring VLANs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-3/configuration_guide/vlan/b_173_vlan_9300_cg.html)
*   **Cisco ASA Series Firewall CLI Configuration Guide**
    *   [VLAN Interfaces](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/general/asa-94-general-config/interface-subinterfaces.html)
*   **Cisco Live (BRKSEC-2202)**
    *   [Layer 2 Security In-Depth](https://www.ciscolive.com/)
*   **Technical Notes**
    *   [Understanding and Configuring VLAN Access Control Lists (VACLs)](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-6500-series-switches/10592-20.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「セグメンテーションは VLAN に始まり VLAN に終わる」と言われるほど重要です。L2 の分離が不完全であれば、L3 以上のセキュリティポリシー（Firewall 等）も回避されるリスクがあることを意識してください。
*   **図解**: トランクリンクを流れるフレームを想像し、どこでタグが付けられ、どこで外されるかをパケットトレーサー等のツールで可視化して理解しましょう。
*   **注意点**: ラボ試験では、`vlan [ID]` コマンドの後に **`exit`** を入力しないと、データベースに反映されない古い IOS バージョンがあるため、確実に抜ける癖をつけましょう。
