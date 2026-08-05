---
layout: default
title: 1.1.b-Transparent
nav_order: 2
parent: 1.1-Deployment-modes-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.1.b Transparent Mode

Cisco ASAおよびCisco Firepower Threat Defense (FTD) における**透過モード（Transparent Mode）**は、デバイスを「ネットワーク内のステルス・ホップ」として配置するレイヤ2（L2）展開方式です。このモードでは、既存のIPアドレッシングを変更することなく、セキュリティ機能を追加できます。

---

# 📘 概要

*   **機能概要**: デバイスがレイヤ2ブリッジとして動作します。パケットのIPヘッダーを書き換えることなく転送し、レイヤ2の送信元/宛先MACアドレスに基づいてトラフィックを検査・制御します。
*   **利用目的**: 既に運用されているネットワークトポロジにおいて、IPサブネットの変更やルータの再設定を行うことなく、ファイアウォールによるセキュリティ（ACL、インスペクション）を挿入する場合に使用されます。
*   **場面**: データセンターの既存セグメント間、または境界ルータと内部スイッチの間に「Bump-in-the-wire（配線上の隆起）」として挿入されます。

---

# 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | デバイス自体がL3ホップにならず、L2スイッチのようにパケットをブリッジする。 |
| **用途** | 既存ネットワークへの迅速な挿入、IPアドレスの節約、L2トラフィック（非IP）の制御。 |
| **メリット** | ネットワーク構成の変更が不要。MACアドレスに基づくフィルタリング（EtherType ACL）が可能。 |
| **デメリット** | ダイナミックルーティングへの参加が限定的。RA-VPN（Remote Access）の終端に非対応。 |
| **対応機種** | 全てのCisco ASAおよびFTDモデル。ASAv/FTDvを含む。 |
| **制限事項** | ブリッジグループ（BVI）が必要。マルチキャストやルーティングプロトコル通過には設定が必要。 |
| **設計上の注意点** | 管理通信用にBridge Virtual Interface (BVI) にIPを割り当てる必要がある。 |

---

# 🏗 動作原理

透過モードでは、パケットは入力インターフェイスから入り、L2ルックアップとセキュリティチェックを経て、同一ブリッジグループ内の出力インターフェイスへ転送されます。

```text
Host A (10.1.1.10)
   ↓
[ Ingress Interface (Bridge-group 1) ]
   ↓
[ Security Engine ]
   1. MAC Look-up (既存のMACテーブル参照)
   2. ACL Check (IP ACL または EtherType ACL)
   3. Inspection (L4-L7検査 / Snort)
   4. Bridge decision (宛先MACに基づく出力ポート決定)
   ↓
[ Egress Interface (Bridge-group 1) ]
   ↓
Router (10.1.1.1)
```

---

# ⚙ 動作シーケンス

透过モードでのパケット処理フローは以下の通りです。

1.  **L2フレーム受信**: インターフェイスでフレームを受信します。
2.  **接続テーブル参照**: 既存のセッション（L3-L4）があるか確認し、あれば高速処理パスへ回します。
3.  **ACLチェック**:
    *   **IP ACL**: 送信元/宛先IPアドレスに基づき許可/拒否を判断。
    *   **EtherType ACL**: 非IPプロトコル（BPDU、MPLSなど）を制御。
4.  **MAC学習/参照**: 送信元MACを学習し、宛先MACに基づき転送先を決定します。
5.  **インスペクション**: サービスポリシー（MPF）またはSnortエンジンによる高度な検査（HTTP, DNS等）を行います。
6.  **L2ヘッダー維持**: 送信元/宛先IPアドレスやMACアドレスを維持したまま、フレームを送出します（TTLは通常減算されません）。

---

# 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **BVI (Bridge Virtual Interface)**: 透過モードでは、管理アクセスや自身が発生させる通信のために、ブリッジグループに対してBVIを作成し、IPアドレスを設定する必要があります。
*   **IRB (Integrated Routing and Bridging)**: ルーテッドインターフェイスとブリッジグループを同一デバイス内で混在させる高度な構成が問われます。

### ラボ試験で設定させられそうな内容
*   **非IPトラフィックの許可**: ラボで「OSPFハローを通過させよ」または「BPDUを許可せよ」という課題が出た場合、EtherType ACLの適切な設定が必要です。
*   **ARPインスペクション**: L2デバイスとして動作するため、偽装ARPを防ぐための設定が求められることがあります。
*   **透過モードへの切り替え**: `firewall transparent` コマンドによりモードを変更すると、**設定が全て消去される**ため、切り替え後にゼロからインターフェイスとBVIを構成する手順を習熟しておく必要があります。

### よくある設定ミス
*   **BVIのIP不足**: BVIにIPがないと、FMCとの通信やSyslog送信、SNMPなどの管理機能が動作しません。
*   **同一サブネットの制約**: 透過モードの各インターフェイスは異なるL2ドメインに属するべきですが、IP的には同一サブネットを「跨いで」配置されます。

### showコマンド/debugによる状態判断
*   `show bridge-group`: ブリッジグループのメンバーとBVIの状態を確認します。
*   `packet-tracer`: 透過モードでも、パケットがどのACLでドロップしているかを確認するのに不可欠です。

---

# 🛠 設定方法

### ASA (CLI) - 透過モード設定
ASAでは、グローバル設定でモードを変更してからインターフェイスをブリッジグループに所属させます。
```bash
# モードを透過モードに変更 (既存設定はクリアされる)
firewall transparent

# インターフェイス設定
interface GigabitEthernet0/0
 bridge-group 1
 nameif outside
 security-level 0
!
interface GigabitEthernet0/1
 bridge-group 1
 nameif inside
 security-level 100

# BVI (管理IP) の設定
interface BVI 1
 ip address 10.1.1.254 255.255.255.0
```

### FTD (FMC管理) - 透過モード設定
FTDの設定はFMCのGUIから行います。
1.  **Devices > Device Management**: 対象デバイスの編集。
2.  **Interfacesタブ**: 物理インターフェイスを編集し、**Mode**を「Transparent」に設定。
3.  **Add Interfaces > Bridge Group Interface**:
    *   **ID**: 1などを指定。
    *   **Bridge Group ID**: メンバーとなる物理ポートを選択。
    *   **IPv4タブ**: BVI用のIPアドレスを設定。
4.  **Security Zone**: ブリッジグループの各メンバーインターフェイスにゾーンを割り当てます。

---

# 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **動作モードの確認** | <code>show firewall</code> |
| **ブリッジグループの確認** | <code>show bridge-group</code> |
| **MACアドレステーブルの表示** | <code>show mac-address-table</code> |
| **BVIインターフェイスの状態** | <code>show interface bvi 1</code> |
| **パケットパスのシミュレーション** | <code>packet-tracer input inside icmp 10.1.1.10 8 0 10.1.1.1</code> |
| **ARPキャッシュの確認** | <code>show arp</code> |

---

# 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 特定の動的ルートが届かない | OSPF/EIGRP等の非IP通信がドロップされている | <code>packet-tracer</code>でEtherType ACLの不足を確認し、マルチキャストを許可。 |
| 通信が不安定（フラッピング） | L2ループが発生している | <code>show bridge-group</code>でポート状態を確認。Spanning-tree構成を見直す。 |
| デバイス自体へのSSHができない | BVIにIPがない、またはルートがない | <code>show ip address</code>でBVIの状態を確認。ゲートウェイ設定を追加。 |
| モード変更後に通信断 | モード変更による全設定クリア | <code>show running-config</code>でインターフェイス設定が残っているか確認。 |

---

# ⚠ 制限事項

*   **VPNの制約**: リモートアクセスVPN（AnyConnect等）は透過モードではサポートされません。サイト間VPN（L2L）は可能ですが、管理通信を通すなどの限定的な用途になります。
*   **L3機能の欠如**: 動的ルーティングプロトコル（OSPF/BGP）の「ネイバー」として自身が参加することはできません（パケットを「透過」させることは可能です）。
*   **DHCPパススルー**: デフォルトではDHCP要求をドロップする場合があるため、明示的に許可設定が必要です。
*   **IRBの制約**: クラスタリング構成など、特定のハイエンド機能とIRBの同時使用には制限がある場合があります。

---

# 🔄 他技術との関連

*   **EtherType ACL**: 透過モードにおいて、ARP、BPDU、IS-IS、MPLSなどの非IPプロトコルを制御するために使用されます。
*   **NAT**: 透過モードでもNATは可能ですが、ルーテッドモードほど柔軟ではありません。
*   **High Availability (Failover)**: 透過モードでも冗長化は可能であり、BVIのIPとMACアドレスがフェイルオーバーの対象となります。
*   **Integrated Routing and Bridging (IRB)**: 透過モード（Bridge）とルーティング（Route）を一つのデバイス内で統合する技術です。

---

# 🧩 比較表

### Routed vs Transparent (ASA/FTD)

| 機能 | Routed Mode (L3) | Transparent Mode (L2) |
| :--- | :--- | :--- |
| **OSIレイヤ** | レイヤ3 (Router) | レイヤ2 (Bridge) |
| **IPアドレス** | 各インターフェイスに必要 | BVIに1つ必要 |
| **ネットワーク挿入** | 再アドレッシングが必要 | 既存構成を維持可能 |
| **VPN終端** | 全VPNをフルサポート | 管理用などの限定的サポート |
| **ステルス性** | 低い (ホップとして見える) | 高い (ホップとして見えない) |

---

# 💡 ベストプラクティス

*   **EtherType ACLの事前定義**: ネットワーク内でルーティングプロトコルが動いている場合、導入前にそれらを許可するEtherType ACLを準備します。
*   **BVIの冗長化**: フェイルオーバー構成時は、Active/Standbyの両ユニットでBVIのIPアドレスが整合していることを確認します。
*   **ARP制御**: `arp-inspection` 機能を有効にして、L2環境でのMITM攻撃（中間者攻撃）を防止します。

---

# 📝 ラボ学習・設定サンプル例

### 1. 基本的なブリッジグループとBVIの構成 (ASA)
*   **課題**: Gig0/0とGig0/1をブリッジし、管理用IP 192.168.1.254 を設定せよ。
*   **設定**:
```bash
firewall transparent
interface GigabitEthernet0/0
 nameif outside
 bridge-group 1
!
interface GigabitEthernet0/1
 nameif inside
 bridge-group 1
!
interface BVI 1
 ip address 192.168.1.254 255.255.255.0
```

### 2. EtherType ACLによるBPDUの許可
*   **課題**: 透過モードのASAを通過するSTP BPDUを許可せよ。
*   **設定**:
```bash
ethertype access-list ALLOW_BPDU permit bpdu
access-group ALLOW_BPDU in interface inside
access-group ALLOW_BPDU in interface outside
```

### 3. FTDにおけるブリッジグループ(BVI)の設定 (FMC)
*   **課題**: FMCを使用して、FTDのGig0/0と0/1でBridge Group 1を作成せよ。
*   **手順**: FMCのインターフェイス設定で **Add Interfaces > Bridge Group Interface** を選択。物理ポートを追加し、BVI IPを指定してデプロイ。

### 4. DHCPパケットのパススルー設定
*   **課題**: クライアントがASA越しにDHCPサーバからIPを取得できるようにせよ。
*   **設定**:
```bash
# グローバルポリシー等でインスペクション設定
parameter-map type inspect global
 l2-transparent dhcp-passthrough enable
```

### 5. 透過モードにおけるIP ACLの設定
*   **課題**: Insideの10.1.1.10からOutsideの8.8.8.8へのHTTPS通信のみを許可せよ。
*   **設定**:
```bash
access-list IN_TO_OUT extended permit tcp host 10.1.1.10 host 8.8.8.8 eq 443
access-group IN_TO_OUT in interface inside
```

### 6. OSPFトラフィック（マルチキャスト）の透過許可
*   **課題**: 対向ルータ間のOSPFネイバー確立を妨げないようにせよ。
*   **設定**:
```bash
access-list OSPF_PERMIT extended permit ospf any any
access-group OSPF_PERMIT in interface inside
access-group OSPF_PERMIT in interface outside
```

### 7. IRB (Integrated Routing and Bridging) の構成 (FTD)
*   **課題**: Gig0/0, 0/1はブリッジ（BVI 1）、Gig0/2はルーテッドとして構成せよ。
*   **手順**: FMCにてインターフェイスModeを混在させ、Bridge GroupとRoutedインターフェイスをそれぞれ定義する。

### 8. MACアドレススティッキーの設定
*   **課題**: セキュリティ向上のため、学習したMACアドレスを固定せよ。
*   **設定**: `mac-address-table static ...` コマンドで対応。

### 9. 透過モードでのARPインスペクション
*   **課題**: 不正なARP応答をブロックせよ。
*   **設定**:
```bash
arp-inspection inside enable flood
```

### 10. 透過モードASAへの管理アクセスの制限
*   **課題**: BVIのIP(192.168.1.254)へのSSHを特定の端末のみに制限せよ。
*   **設定**:
```bash
ssh 192.168.1.100 255.255.255.255 inside
```

---

# ❓ 想定試験問題

1.  **問題**: 透過モードのASAにおいて、デフォルトでドロップされるトラフィックはどれか？
    *   A. 同一サブネット内のIPユニキャストパケット
    *   B. 非IPのL2ブロードキャスト（BPDUなど）
    *   C. セキュリティレベルの低い方から高い方へのパケット
    *   **解答**: B, C (非IPは明示的なEtherType ACLが必要、低いレベルからはACLが必要)。
2.  **問題**: 透過モードのFTDにおいて、管理アクセス（SSH）用のIPアドレスはどこに設定すべきか？
    *   **解答**: Bridge Virtual Interface (BVI)。
3.  **問題**: 透過モードで動作しているデバイスにおいて、`packet-tracer` を実行する際の注意点は？
    *   **解答**: 送信元インターフェイスとプロトコル、送信元/宛先IPを指定するが、透過モード特有のブリッジング判定プロセスもシミュレーションされることを理解しておく必要がある。
4.  **問題**: ルーテッドモードから透過モードへの変更をCLIで行った場合、既存のポリシーはどうなるか？
    *   **解答**: 全て消去される。デバイスは工場出荷時の透過モード設定にリセットされる。
5.  **問題**: 透過モードのインターフェイスで、TTL（Time to Live）は減算されるか？
    *   **解答**: 通常、透過モード（L2）ではTTLは減算されないが、IRB構成などでL3転送が発生する場合は減算される。

---

# 🔗 参考リソース

*   **Configuration Guides**:
    *   [Cisco ASA Series General Operations CLI Configuration Guide, 9.4 - Transparent Firewall Mode](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config/general/asa-94-general-config/intro-fw.html#ID-2101-0000003b)
    *   [Cisco Firepower Threat Defense Configuration Guide for FMC, 7.1 - Transparent Mode Interfaces](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/interfaces_firewall_mode.html)
*   **Cisco Live (Video/Slides)**:
    *   [BRKSEC-3020: Troubleshooting Firewall Threat Defense (FTD)](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-3020)
*   **Technical Notes**:
    *   [Transparent Firewall Mode Overview (Cisco Support)](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/118804-config-asa-00.html)
*   **Design Guide**:
    *   [Cisco SAFE Design Guide - Firewall Deployment Modes](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Aug2014/CVD-FirewallDeploymentGuide-AUG14.html)

---

📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

---
