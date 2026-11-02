---
layout: default
title: 2.5.d-VRF-Lite
nav_order: 4
parent: 2.5-Infrastructure-segmentation
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.5.d VRF-Lite

Cisco ネットワークアーキテクチャにおける **VRF-Lite (Virtual Routing and Forwarding Lite)** は、単一の物理ルーター、マルチレイヤースイッチ、またはセキュリティアプライアンス（Cisco ASA / FTD）上で、MPLS (Multiprotocol Label Switching) のコントロールプレーン機能（LDP や RSVP-TE など）を使用せずに、複数の独立したルーティングテーブル（FIB: Forwarding Information Base）とインターフェイス割り当てを構成・維持するインフラストラクチャセグメンテーション技術です。

CCIE Security v6.1 Blueprint **2.5.d VRF-Lite** では、Cisco IOS-XE、Cisco ASA（Multiple Context Mode または VRF サブインターフェイス）、および Cisco FTD（Virtual Routers）における VRF の定義、802.1Q サブインターフェイスや Trunk ポートとの統合、VRF 固有のルーティングプロトコル（OSPFv2/v3, EIGRP, MP-BGP, Static Routes）の設定、VRF 間ルートリーク（Route Leaking）、Front-Door VRF (FVRF) / Inside VRF (IVRF) を用いた VPN トンネル分離、VRF-Aware NAT、およびトラブルシューティング能力が深く問われます。

---

## 📘 概要

* **機能概要**:
  * **VRF-Lite (Virtual Routing and Forwarding Lite)**: 物理デバイス内でルーティングテーブル（RIB）、転送テーブル（FIB）、およびインターフェイス割り当てを論理的に分離し、完全に隔離された仮想ルータードメインを構築する技術。MPLS バックボーンを必要とせず、点対点（Hop-by-Hop）の 802.1Q VLAN タグや GRE トンネルを介してマルチテナント拡張を実現する。
  * **マルチインスタンスルーティングプロトコル**: OSPFv2/v3、EIGRP、BGP、RIP などの動的ルーティングプロトコルを VRF ごとに独立したプロセスまたは Address-Family として起動可能。
  * **VRF 間ルートリーク (Inter-VRF Route Leaking)**: 共通サービス（DNS, NTP, ISE, Internet 出口など）へのアクセスのため、Static Route + Route-Map や MP-BGP (Route-Distinguisher / Route-Target) を使用して特定のルーティング情報を安全に相互注入する。
  * **Front-Door VRF (FVRF) / Inside VRF (IVRF)**: DMVPN または FlexVPN において、アンダーレイ（パブリック・トランスポート網）の IP アドレス空間と、オーバーレイ（暗号化されたプライベート網）の IP アドレス空間を同一ルーター上で完全に分離し、ルーティングループや重複 IP アドレスの問題を根本的に解決する手法。
* **利用目的**:
  * 同一の物理ルーター/スイッチ基盤上で、企業の複数部門（営業、経理、開発）、ゲストネットワーク、サードパーティベンダーを物理的・論理的に相互分離し、ゼロトラスト境界を確立する。
  * 競合する子会社の統合や M&A 時における重複 IP アドレス（Overlapping IP Addresses）の相互接続・ルーター共有処理。
  * VPN アンダーレイ網とオーバーレイ網の明確な分離によるトラフィックリークの防止とセキュリティ向上。
* **どのような場面で利用するか**:
  * エンタープライズキャンパス / データセンターの L3 セグメンテーション。
  * クラウド接続エッジにおけるマルチテナントハンドオフ（AWS Direct Connect / Azure ExpressRoute の VRF 分離）。
  * Cisco ISE と連携した Dynamic Access Control / TrustSec SGT / Dynamic VRF アサイン。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **評価対象技術** | **Cisco IOS-XE VRF-Lite**、**VRF-Aware Dynamic Routing**、**Inter-VRF Route Leaking**、**Front-Door VRF (FVRF)**、**FTD Virtual Routers** |
| **カプセル化方式** | ホップバイホップの **IEEE 802.1Q VLAN Tagging**、**GRE / mGRE**、**802.1ad (QinQ)** |
| **MPLS 非依存性** | **MPLS Label (LDP/RSVP) は一切不要**。Control Plane と Data Plane の分離はローカルの VRF テーブルおよび L2 タギングのみで完了。 |
| **ルーティング分離** | ルックアップテーブル（RIB/FIB）、ARP テーブル、Cisco Express Forwarding (CEF) アジャセンシーが VRF ごとに完全に独立。 |
| **重複 IP のサポート** | 異なる VRF 間であれば同一の IP サブネット（例: `10.1.1.0/24`）を重複割り当て・個別運用可能。 |
| **VRF ルートリーク方法** | 1. **Static Route + Route-Map** (ローカル VRF 間で次跳 IP/Interface を指定)<br>2. **MP-BGP (RD / RT)** (ローカル BGP プロセス内で `route-target export/import` を使用) |
| **セキュリティ設計上の注意点** | VRF ルートリークを構成する際、適切なアクセスリスト（ACL）やファイアウォール（ASA/FTD）を間に挟まないと、せっかくのセグメンテーションが完全に破綻するリスクがある。 |

---

## 🏗 動作原理

### 1. VRF-Lite におけるコントロールプレーン ＆ データプレーンの独立性

VRF-Lite では、物理ルーター内部で各 VRF 専用の RIB (Routing Information Base)、FIB (Forwarding Information Base)、および ARP テーブルが保持されます。

```text
[ Physical Switch / Router (IOS-XE) ]
 ├── Global Routing Table (Default / Non-VRF)
 │    └── Interfaces: Gi0/0 (Global)
 │
 ├── VRF RED Routing Table (FIB RED)
 │    ├── ARP Table RED
 │    └── Interfaces: Gi0/1.10 (VLAN 10), Tunnel1 (RED)
 │
 └── VRF BLUE Routing Table (FIB BLUE)
      ├── ARP Table BLUE
      └── Interfaces: Gi0/1.20 (VLAN 20), Tunnel2 (BLUE)
```

1. **パケット着信と VRF 識別**:
   * パケットがサブインターフェイス `Gi0/1.10`（VRF RED に割り当て）に着信すると、IOS-XE はパケットの L3 宛先 IP を **VRF RED の専用 FIB** でのみルックアップします。
2. **ルーティングテーブルの隔離**:
   * VRF RED の FIB に宛先が存在しない場合、Global や VRF BLUE のテーブルにルートが存在していたとしても、パケットは **Unreachable として即座に破棄** されます。
3. **802.1Q タグによるトランク延伸**:
   * 対向の L3 スイッチやルーターとの間は 802.1Q トランクで接続され、VLAN 10 のフレームは対向の VRF RED に、VLAN 20 のフレームは対向の VRF BLUE にマッピングされ、L3 分離が維持されます。

---

### 2. Front-Door VRF (FVRF) と Inside VRF (IVRF) による VPN トンネル分離原理

DMVPN または FlexVPN において、アンダーレイのトランスポート網を `FVRF_INET` に隔離し、暗号化後のオーバーレイ通信を `IVRF_CORP` で処理するアーキテクチャ：

```text
[ Customer LAN (10.1.1.0/24) ]
       │
       ▼ (IVRF_CORP)
[ Inside Interface: Gi0/1 ] ──► (VRF: IVRF_CORP)
                                      │
                                Routing Lookup (IVRF_CORP)
                                      │
                                      ▼
                        [ Tunnel Interface: Tunnel0 ]
                          ├── ip vrf forwarding IVRF_CORP  (Overlay / Inner Packet)
                          └── tunnel vrf FVRF_INET         (Underlay / Outer Packet)
                                      │
                                Encapsulation (ESP + IP)
                                      │
                                      ▼
                                Routing Lookup (FVRF_INET)
                                      │
[ Outside Interface: Gi0/0 ] ◄── (VRF: FVRF_INET)
       │
       ▼
[ Internet / Public Transport (192.0.2.0/24) ]
```

---

## ⚙ 動作シーケンス

1. **VRF 定義とインターフェイスバインドのシーケンス**:
   * 管理者が `vrf definition <NAME>` (または従来の `ip vrf <NAME>`) を作成し、`address-family ipv4 / ipv6` を有効化。
   * インターフェイス（物理ポート、サブインターフェイス、SVIs、トンネル）に `vrf forwarding <NAME>` を適用。
   * **重要シーケンス**: インターフェイスに `vrf forwarding` を適用した瞬間、**既存の IP アドレスが削除される** ため、必ず IP アドレスを再設定（Re-apply）する。
2. **VRF 固有のルーティングプロトコル実行シーケンス**:
   * **OSPFv2**: `router ospf <PROCESS_ID> vrf <NAME>` で独立した OSPF プロセスを起動。LSA は該当 VRF 内のみで生成・維持される。
   * **BGP**: `router bgp <AS_NUMBER>` の下で `address-family ipv4 vrf <NAME>` を作成し、VRF 固有の Neighbor や ネットワークを定義。
3. **Inter-VRF Route Leaking の処理シーケンス (Static Route + Route-Map)**:
   * **VRF A から VRF B へのルート漏洩**:
     * `ip route vrf VRF_A 10.2.0.0 255.255.255.0 GigabitEthernet0/1.20 192.168.2.1`（出力インターフェイスと次跳 IP を明示指定）。
     * 送信元パケットが VRF A で受信された際、次跳インターフェイスが VRF B に属する `Gi0/1.20` であるため、ルーターはインターフェイスの所属に従い VRF B 側へパケットを透過転送する。
4. **MP-BGP による Route-Target (RT) ベースのルートリークシーケンス**:
   * VRF RED のルートが BGP へ再配布される際、`rd 65000:1`（Route Distinguisher）が付与されて VPNv4 ルート（`65000:1:10.1.1.0/24`）に変換される。
   * エクスポート時に `route-target export 65000:100` が付与される。
   * VRF BLUE 側で `route-target import 65000:100` が設定されている場合、BGP コントロールプレーンは VPNv4 テーブルから該当ルートを抽出し、VRF BLUE の IPv4 RIB に自動適用する。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **`vrf definition` (New Syntax) vs `ip vrf` (Legacy Syntax)**:
   * **New Syntax (推奨)**: `vrf definition RED` ➔ `address-family ipv4` / `address-family ipv6` ➔ インターフェイスで `vrf forwarding RED`。IPv4/IPv6 デュアルスタックを統一管理可能。
   * **Legacy Syntax**: `ip vrf RED` ➔ `rd 1:1` ➔ インターフェイスで `ip vrf forwarding RED`。IPv6 は別途 `ipv6 vrf RED` が必要となりラボでの混乱を招く。**必ず New Syntax をマスターすること**。
2. **`vrf forwarding` 設定時の IP アドレス消失トラップ**:
   * ラボ試験でインターフェイスに IP アドレスを設定した後に `vrf forwarding <NAME>` コマンドを打つと、IOS-XE は静かに **IP アドレスをクリア** します。
   * **対策**: 必ず `vrf forwarding <NAME>` を設定してから `ip address x.x.x.x y.y.y.y` を投入する順序を徹底する。
3. **OSPF Down-Bit および Domain-ID によるルートドロップ問題 (VRF-Lite BGP PE-CE 連携)**:
   * OSPF を VRF 内で動作させ、BGP と相互再配布を行う場合、ルーターは LSA タイプ 3 に **Down-Bit (DN Bit)** をセットします。
   * VRF-Lite 構成（MPLS なしでルーター同士が OSPF で直接接続されている場合）、受信側ルーターは Down-Bit が立った LSA タイプ 3 を「ループ防止」のため**無条件で破棄**します。
   * **解消コマンド**: `router ospf <ID> vrf <NAME>` の配下で **`capability vrf-lite`** を投入し、Down-Bit による LSA 破棄チェックを無効化する必要があります。
4. **Front-Door VRF (FVRF) における Tunnel 設定**:
   * アンダーレイ用の Outside インターフェイスを `vrf forwarding FVRF_INET` に配置。
   * トンネルインターフェイス側で:
     * `ip vrf forwarding IVRF_CORP` (内部/オーバーレイ用 VRF)
     * `tunnel vrf FVRF_INET` (アウター IP パケットの送信に使用するアンダーレイ VRF)
   * この 2 行の組み合わせを正確に設定できるかが問われます。
5. **Static Inter-VRF Route Leaking の双方向指定要件**:
   * 片方向の `ip route vrf` 設定だけでは戻りトラフィックが破棄されます。
   * **双方向のルートリーク**（VRF A ➔ VRF B、および VRF B ➔ VRF A）を設定するか、NAT（VRF-Aware NAT）を組み合わせて片方向からの通信を成立させる必要があります。

---

## 🛠 設定方法

### 1. IOS-XE: VRF-Lite (New Syntax) と OSPFv2 / BGP マルチインスタンス設定

```bash
! ===== Step 1: VRF Definition =====
vrf definition VRF_GUEST
 rd 65000:10
 address-family ipv4
 exit-address-family

vrf definition VRF_CORP
 rd 65000:20
 address-family ipv4
 exit-address-family

! ===== Step 2: Interface Assignment =====
interface GigabitEthernet0/1.10
 description GUEST VLAN
 encapsulation dot1Q 10
 vrf forwarding VRF_GUEST
 ip address 10.10.10.1 255.255.255.0

interface GigabitEthernet0/1.20
 description CORP VLAN
 encapsulation dot1Q 20
 vrf forwarding VRF_CORP
 ip address 10.20.20.1 255.255.255.0

! ===== Step 3: VRF-Aware OSPFv2 Configuration =====
router ospf 10 vrf VRF_GUEST
 capability vrf-lite                     ! Down-bit チェック無効化 (VRF-Lite 必須)
 router-id 10.10.10.1
 network 10.10.10.0 0.0.0.255 area 0

! ===== Step 4: VRF-Aware BGP Configuration =====
router bgp 65000
 bgp log-neighbor-changes
 
 address-family ipv4 vrf VRF_CORP
  neighbor 10.20.20.2 remote-as 65100
  neighbor 10.20.20.2 activate
  network 10.20.20.0 mask 255.255.255.0
 exit-address-family
```

---

### 2. IOS-XE: Static Inter-VRF Route Leaking 設定

```bash
! ===== VRF_A (10.1.1.0/24) から VRF_B (10.2.2.0/24) へのルートリーク =====
! 出力インターフェイスと Next-Hop IP を両方明示指定する
ip route vrf VRF_A 10.2.2.0 255.255.255.0 GigabitEthernet0/1.20 10.2.2.2
ip route vrf VRF_B 10.1.1.0 255.255.255.0 GigabitEthernet0/1.10 10.1.1.2
```

---

### 3. IOS-XE: MP-BGP (Route-Target) を用いたローカル Inter-VRF Route Leaking

```bash
! ===== MP-BGP による安全な VRF 相互漏洩 =====
vrf definition VRF_SHARED
 rd 65000:100
 address-family ipv4
  route-target export 65000:100
  route-target import 65000:200
 exit-address-family

vrf definition VRF_DEPT1
 rd 65000:200
 address-family ipv4
  route-target export 65000:200
  route-target import 65000:100
 exit-address-family

router bgp 65000
 address-family ipv4 vrf VRF_SHARED
  redistribute connected
 exit-address-family

 address-family ipv4 vrf VRF_DEPT1
  redistribute connected
 exit-address-family
```

---

### 4. FMC GUI / FTD: Virtual Routers (VRF-Lite) 設定手順

```text
[ FMC GUI Steps: FTD Virtual Routers 構成 ]
1. Devices > Device Management に移動し、対象 FTD の編集（鉛筆アイコン）をクリック。
2. "Routing" タブを選択 ➔ 左側メニューから "Virtual Routers" を選択。
3. "Add Virtual Router" をクリック:
   - Name: VRF_DMZ
   - Description: Virtual Router for DMZ Segmentation
4. "Interfaces" タブで、バインドするインターフェイス（例: Subinterface Gi0/0.100）を移動して追加。
5. "Routing Protocol" タブ（OSPF / BGP / Static Routes）を選択し、VRF_DMZ 固有のルーティングを設定。
6. Save して Deploy を実行。
```

## 🔍 検証コマンド

| 目的 | コマンド (IOS-XE / FTD CLI) | 表示確認ポイント |
| :--- | :--- | :--- |
| **VRF 一覧 ＆ バインドインターフェイス確認** | <code>show vrf</code><br><code>show ip vrf</code> | VRF 名、RD (Route Distinguisher)、バインドされている物理/サブインターフェイス一覧 |
| **VRF 固有のルーティングテーブル (RIB) 確認** | <code>show ip route vrf <NAME></code> | 該当 VRF 内にのみ存在する特定サブネット、Next-hop、Protocol フラグ |
| **VRF 固有の CEF 転送テーブル (FIB) 確認** | <code>show ip cef vrf <NAME></code> | 該当 VRF での CEF 転送エントリー、出力インターフェイス、Adjacency 状態 |
| **VRF 固有の ARP テーブル確認** | <code>show ip arp vrf <NAME></code> | VRF ごとに分離・保持されている IP-to-MAC アドレス変換エントリー |
| **VRF 指定 Ping / Traceroute テスト** | <code>ping vrf <NAME> <IP></code><br><code>traceroute vrf <NAME> <IP></code> | 特定 VRF のルーティングテーブルを経由した対向宛先への L3 疎通確認 |
| **VRF 固有の OSPF ネイバー ＆ データベース確認**| <code>show ip ospf vrf <NAME> neighbor</code><br><code>show ip ospf vrf <NAME> database</code> | VRF プロセス配下での OSPF アジャセンシー確立状態と LSA タイプ確認 |
| **VRF 固有の BGP サマリー ＆ ルートテーブル確認** | <code>show bgp vpnv4 vrf <NAME> summary</code><br><code>show ip bgp vrf <NAME></code> | VRF アドレスファミリーでの BGP ピア状態（Established）および受信ルート |
| **VPNv4 ルーティングテーブル（MP-BGP）確認** | <code>show ip bgp vpnv4 all</code> | RD 付き VPNv4 ルート（`RD:IP`）および付与された Route-Target 属性 |
| **ルーティング更新のデバッグ追跡** | <code>debug ip routing vrf <NAME></code><br><code>debug ip ospf events</code> | VRF 内でのルート挿入・削除ログ、Down-Bit による LSA ドロップ通知 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **インターフェイスに `vrf forwarding` を適用したら IP アドレスが消えて通信が全断した** | IOS-XE の仕様上、`vrf forwarding` コマンド投入時に既存の IP アドレスが自動削除される | <code>show ip interface brief</code> | `vrf forwarding <NAME>` を設定した後に、**必ず IP アドレスを再設定** する。 |
| **VRF 固有の OSPF で対向とネイバーがアップしているが、Routes が RIB に注入されない** | OSPF LSA タイプ 3 に Down-Bit (DN Bit) が立っており、VRF-Lite ルーターがループと判定して破棄している | <code>show ip ospf vrf <NAME> database summary</code><br><code>debug ip ospf rib</code> | OSPF プロセス配下で **`capability vrf-lite`** コマンドを投入し、DN Bit チェックを無効化する。 |
| **Static Route で Inter-VRF Route Leaking を設定したが、対向へ Ping が通らない** | `ip route vrf` 設定で出力インターフェイスまたは Next-Hop IP の指定が漏れている、または戻りルートが無い | <code>show ip route vrf <NAME></code> | ルートリーク設定で **「出力インターフェイス ＋ Next-Hop IP」を明示** し、送信元/送信先双方の VRF で相互リークを設定する。 |
| **DMVPN / FlexVPN で Tunnel が Down し、`IPSEC-3-EGP_DUP_SPI` や Routing Loop アラートが出る** | アンダーレイ (FVRF) とオーバーレイ (IVRF) のルーティングが同一 VRF に存在し、ルーティングループが発生している | <code>show ip route vrf FVRF</code><br><code>show ip route vrf IVRF</code> | Outside インターフェイスを `FVRF` に配置し、Tunnel インターフェイスで `ip vrf forwarding IVRF` ＋ **`tunnel vrf FVRF`** を正しく分離指定する。 |
| **BGP による Inter-VRF Route Leaking でルートが別 VRF へ注入されない** | `route-target export` と `route-target import` の RT 値が一致していない、または BGP にルートが再配布されていない | <code>show ip bgp vpnv4 all</code><br><code>show vrf detail</code> | 双方の VRF で `route-target export/import` の RT 値（例: `65000:100`）を完全一致させ、BGP 配下で `redistribute connected/static` を適用する。 |

---

## ⚠ 制限事項

* **Global ルートと VRF ルート間の制約**:
  * VRF にバインドされたインターフェイスは、Global (Default) ルーティングテーブルに属するインターフェイスと直接 L3 ルックアップを共有できません。相互通信には明示的な Route Leaking または物理/VLAN バック・トゥ・バック接続が必要です。
* **Cisco FTD Virtual Routers の制限**:
  * FTD 7.0 以降で Virtual Routers (VRF-Lite) がサポートされましたが、ASA の Multiple Context Mode とは異なり、完全に独立した管理プレーンを持つわけではありません（単一の Snort エンジンと FMC ポリシーで統合管理されます）。
* **マルチキャスト (PIM) 制限**:
  * VRF-Aware Multicast (MVPN) を動作させる場合、ルーターのプラットフォームおよびライセンスレベル（Advanced IP Services 以上）に依存します。

---

## 🔄 他技術との関連

* **Cisco TrustSec / SGT (Security Group Tagging)**: SGT 属性と VRF をマッピングし、L3 セグメンテーション（VRF）と L2/L3 マイクロセグメンテーション（SGT）を統合。
* **Cisco ISE (Identity Services Engine)**: 802.1X / MAB 認証成功時、RADIUS 属性（`Tunnel-Private-Group-ID` や Dynamic VRF）を用いて端末を動的に特定の VRF へ自動バインド。
* **EVPN / VXLAN**: データセンター Fabric において、L3 VNI (Virtual Network Identifier) と VRF-Lite をエッジルーター（Border Leaf）で相互マッピングして外部ハンドオフを実行。

---

## 🧩 比較表

### 1. VRF-Lite vs Full MPLS-VPN vs ASA Multiple Contexts

| 比較項目 | VRF-Lite | Full MPLS-VPN | ASA Multiple Contexts |
| :--- | :--- | :--- | :--- |
| **動作レイヤー** | **Layer 3 (Routed)** | Layer 3 (Routed) | Layer 2 / Layer 3 |
| **MPLS/LDP 依存** | **一切不要** (L2 VLAN / GRE で拡張) | **必須** (LDP / RSVP-TE / MP-BGP) | **一切不要** (Context 仮想化) |
| **オーバーヘッド** | 802.1Q (4 Bytes) / GRE (24 Bytes) | MPLS Labels (4–8 Bytes) | なし |
| **設定複雑度** | 低〜中 (Hop-by-Hop 設定) | 高 (Core / PE / P ルーター設計) | 中 (System ＋ Context 設定) |
| **管理プレーン** | 共通 (単一 IOS-XE CLI) | 共通 (各 PE ルーター) | **分散** (Context ごとに固有 CLI) |
| **推奨適用領域** | キャンパス / DMZ / エッジハンドオフ | キャリア SP / 大規模 WAN | ファイアウォール専用マルチテナント |

---

## 💡 ベストプラクティス

1. **New Syntax (`vrf definition`) の標準採用**:
   * 今後の設計および CCIE ラボ試験では、非推奨となりつつある `ip vrf` ではなく、IPv4/IPv6 デュアルスタックに自然対応する **`vrf definition <NAME>`** 構文を全ルーターで標準統一します。
2. **`capability vrf-lite` の事前投入順守**:
   * OSPF を VRF 配下で動的ルーティングとして使用する場合、BGP 再配布やマルチホップ構成での Down-Bit トラップを回避するため、`router ospf <ID> vrf <NAME>` 定義時に直ちに **`capability vrf-lite`** を書き足す習慣をつけます。
3. **FVRF / IVRF の名前付け規則明確化**:
   * トンネル構成時、アンダーレイ VRF には `FVRF_`（例: `FVRF_INET`）、オーバーレイ VRF には `IVRF_`（例: `IVRF_CORP`）というプレフィックスを付与し、`tunnel vrf` と `vrf forwarding` の指定ミスを視覚的に防止します。

## 📝 ラボ学習・設定サンプル例

### 例1: IOS-XE: 基本的な VRF-Lite 定義とインターフェイス割り当て (`vrf definition`)

#### 問題
ルーター R1 において、`vrf definition` 構文を用いて VRF `RED` (RD `65000:10`) および VRF `BLUE` (RD `65000:20`) を作成せよ。`Gi0/1.10` を VRF `RED` (`10.10.10.1/24`) に、`Gi0/1.20` を VRF `BLUE` (`10.20.20.1/24`) に割り当てよ。

#### 要件
* IPv4 アドレスファミリーを有効化すること。
* インターフェイス割り当て時に IP アドレスが消去されないよう正しい順番で設定すること。

#### 設定例

```bash
! ===== Step 1: VRF Definitions =====
vrf definition RED
 rd 65000:10
 address-family ipv4
 exit-address-family

vrf definition BLUE
 rd 65000:20
 address-family ipv4
 exit-address-family

! ===== Step 2: Subinterface Configuration =====
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 vrf forwarding RED                      ! 必ず IP アドレス設定前に適用
 ip address 10.10.10.1 255.255.255.0
 no shutdown

interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 vrf forwarding BLUE                     ! 必ず IP アドレス設定前に適用
 ip address 10.20.20.1 255.255.255.0
 no shutdown
```

#### 検証方法
* `show vrf` を実行し、`RED` と `BLUE` にそれぞれのサブインターフェイスが正しく割り当てられていることを確認。
* `show ip route vrf RED` および `show ip route vrf BLUE` で直結ルートが表示されることを確認。

---

### 例2: IOS-XE: VRF-Aware OSPFv2 マルチインスタンス構成 (`capability vrf-lite`)

#### 問題
R1 において、VRF `RED` 内で OSPF プロセス `100` を起動し、エリア 0 で `Gi0/1.10` (`10.10.10.0/24`) を広報せよ。対向ルーターからの Type-3 LSA が Down-Bit によって破棄されないよう設定を追加せよ。

#### 要件
* `capability vrf-lite` を設定すること。

#### 設定例

```bash
router ospf 100 vrf RED
 router-id 1.1.1.1
 capability vrf-lite                     ! Down-Bit ドロップの回避
 network 10.10.10.0 0.0.0.255 area 0
```

#### 検証方法
* `show ip ospf vrf RED neighbor` を実行し、対向ルーターと `FULL` 状態になっていることを確認。
* `show ip route vrf RED` で OSPF 経由のルートが正しく RIB に注入されていることを確認。

---

### 例3: IOS-XE: VRF 毎の BGP Address-Family 構成と eBGP ペアリング

#### 問題
R1 (AS 65000) において、VRF `BLUE` 配下で対向ルーター R2 (`10.20.20.2`, AS 65100) と eBGP ピアを確立し、自組織のネットワーク `172.16.20.0/24` を広報せよ。

#### 要件
* `address-family ipv4 vrf BLUE` 構文を使用すること。

#### 設定例

```bash
router bgp 65000
 bgp log-neighbor-changes
 
 address-family ipv4 vrf BLUE
  neighbor 10.20.20.2 remote-as 65100
  neighbor 10.20.20.2 activate
  network 172.16.20.0 mask 255.255.255.0
 exit-address-family
```

#### 検証方法
* `show bgp vpnv4 vrf BLUE summary` を実行し、対向 `10.20.20.2` との BGP セッションが `Established` であることを確認。

---

### 例4: IOS-XE: Static Route + Route-Map による Intra-Router Inter-VRF Route Leaking

#### 問題
R1 上の VRF `CLIENT_A` (`10.1.1.0/24`) から VRF `SHARED_SVC` 上の DNS サーバー (`192.168.100.53/32`) へのアクセスを許可するため、スタティックルートによる相互ルートリークを構成せよ。

#### 要件
* 出力インターフェイスと Next-Hop IP を両方指定すること。

#### 設定例

```bash
! VRF CLIENT_A から SHARED_SVC 宛ての静的ルート
ip route vrf CLIENT_A 192.168.100.53 255.255.255.255 GigabitEthernet0/2.100 192.168.100.1

! SHARED_SVC から CLIENT_A への戻りルート
ip route vrf SHARED_SVC 10.1.1.0 255.255.255.0 GigabitEthernet0/1.10 10.1.1.1
```

#### 検証方法
* `ping vrf CLIENT_A 192.168.100.53` を実行し、正常に応答が返ることを確認。

---

### 例5: IOS-XE: MP-BGP (Route-Target) を用いたローカル VRF Route Leaking

#### 問題
R1 上で MP-BGP を利用し、VRF `GUEST` (RD `65000:10`) と VRF `INTERNET` (RD `65000:99`) の間で、Route-Target (`65000:999`) を介した動的ルートリークを構成せよ。

#### 要件
* `route-target export/import` を使用すること。

#### 設定例

```bash
vrf definition GUEST
 rd 65000:10
 address-family ipv4
  route-target export 65000:999
  route-target import 65000:999
 exit-address-family

vrf definition INTERNET
 rd 65000:99
 address-family ipv4
  route-target export 65000:999
  route-target import 65000:999
 exit-address-family

router bgp 65000
 address-family ipv4 vrf GUEST
  redistribute connected
 exit-address-family

 address-family ipv4 vrf INTERNET
  redistribute connected
 exit-address-family
```

#### 検証方法
* `show ip route vrf GUEST` を実行し、VRF `INTERNET` の接続ルートが BGP (`b`) として学習されていることを確認。

---

### 例6: IOS-XE: DMVPN における Front-Door VRF (FVRF) と Inside VRF (IVRF) 統合構成

#### 問題
R1 において、パブリックインターネット接続ポート `Gi0/0` を `FVRF_INET` に配置し、暗号化オーバーレイ `Tunnel0` を `IVRF_CORP` に配置した DMVPN スポークを構成せよ。

#### 要件
* `tunnel vrf FVRF_INET` および `ip vrf forwarding IVRF_CORP` を併用すること。

#### 設定例

```bash
! ===== Step 1: VRFs =====
vrf definition FVRF_INET
 address-family ipv4
 exit-address-family

vrf definition IVRF_CORP
 address-family ipv4
 exit-address-family

! ===== Step 2: Transport Interface =====
interface GigabitEthernet0/0
 description Outside Public
 vrf forwarding FVRF_INET
 ip address 192.0.2.2 255.255.255.0
 no shutdown

ip route vrf FVRF_INET 0.0.0.0 0.0.0.0 192.0.2.1

! ===== Step 3: Overlay Tunnel =====
interface Tunnel0
 vrf forwarding IVRF_CORP               ! 内側パケットの VRF
 ip address 10.255.255.2 255.255.255.0
 tunnel source GigabitEthernet0/0
 tunnel destination 192.0.2.100
 tunnel vrf FVRF_INET                   ! 外側パケットのトランスポート VRF
 tunnel protection ipsec profile IPSEC_PROF
```

#### 検証方法
* `show ip nhrp brief` および `show crypto ipsec sa` で、トランスポート通信が `FVRF_INET` 経由で正常に確立されていることを確認。

---

### 例7: Cisco ASA: Multiple Context Mode によるテナント分離と Shared Interface 設定

#### 問題
Cisco ASA において Multiple Context Mode を有効化し、`CTX_GUEST` と `CTX_CORP` の 2 つのコンテキストを作成せよ。双方のコンテキストで物理ポート `Gi0/0` (Outside) を共有インターフェイス (Shared Interface) として割り当てよ。

#### 要件
* MAC アドレスによる分類 (`mac-address auto`) を設定すること。

#### 設定例

```bash
! ===== Admin / System Context =====
mode multiple
mac-address auto

context CTX_GUEST
 allocate-interface GigabitEthernet0/0
 allocate-interface GigabitEthernet0/1.10
 config-url disk0:/CTX_GUEST.cfg

context CTX_CORP
 allocate-interface GigabitEthernet0/0
 allocate-interface GigabitEthernet0/1.20
 config-url disk0:/CTX_CORP.cfg
```

#### 検証方法
* `show context` を実行し、`CTX_GUEST` および `CTX_CORP` がともに正常起動しており、`Gi0/0` が共有されていることを確認。

---

### 例8: Cisco FTD (FMC Managed): Virtual Routers (VRF-Lite) の設定と 802.1Q サブインターフェイス割当

#### 問題
FMC 管理下の FTD において、Virtual Router `VR_SECURE` を作成し、サブインターフェイス `Gi0/1.300` を割り当て、静的デフォルトルート (`0.0.0.0/0` -> `10.30.30.254`) を構成せよ。

#### 要件
* FMC GUI の Device Management > Routing から構成すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Devices > Device Management > FTD 編集 > Routing > Virtual Routers
2. "Add Virtual Router" ➔ Name: VR_SECURE
3. Interfaces タブで "Gi0/1.300" を追加。
4. Static Routing タブで "Add Route":
   - Virtual Router: VR_SECURE
   - Interface: Gi0/1.300
   - Network: any-ipv4 (0.0.0.0/0)
   - Gateway: 10.30.30.254
5. Save して Deploy を実行。
```

#### 検証方法
* FTD Diagnostic CLI で `show ip route vrf VR_SECURE` を実行し、`S* 0.0.0.0/0 [1/0] via 10.30.30.254` が存在することを確認。

---

### 例9: IOS-XE + Cisco ISE: Dynamic VRF アサインと VRF-Aware 802.1X / MAB 制御

#### 問題
R1 のアクセスポート `Gi0/2` において 802.1X 認証を有効化し、Cisco ISE から返却される RADIUS 属性 (`Tunnel-Private-Group-ID` / VRF 名) に基づき、端末の接続先 VRF を動的にアサインせよ。

#### 要件
* `ip device tracking` および AAA RADIUS 属性受信を有効化すること。

#### 設定例

```bash
aaa new-model
aaa authentication dot1x default group radius
aaa authorization network default group radius

interface GigabitEthernet0/2
 switchport mode access
 authentication port-control auto
 dot1x pae authenticator
 ip device tracking
```

#### 検証方法
* 端末認証成功後、`show authentication sessions interface Gi0/2 details` を実行し、`VRF Name: VRF_APPROVED` と表示されることを確認。

---

### 例10: IOS-XE: VRF-Aware NAT (Inside VRF から Outside VRF への Dynamic PAT 変換)

#### 問題
R1 上で、`VRF_INSIDE` (`10.100.1.0/24`) から `VRF_OUTSIDE` へ抜けるトラフィックに対し、インターフェイス `Gi0/0` (`192.0.2.2`) の IP アドレスを用いて VRF-Aware Dynamic PAT を構成せよ。

#### 要件
* `ip nat inside source list ... interface ... vrf ... overload` コマンドを使用すること。

#### 設定例

```bash
ip access-list extended ACL_NAT_INSIDE
 permit ip 10.100.1.0 0.0.0.255 any

interface GigabitEthernet0/1.100
 vrf forwarding VRF_INSIDE
 ip address 10.100.1.1 255.255.255.0
 ip nat inside

interface GigabitEthernet0/0
 vrf forwarding VRF_OUTSIDE
 ip address 192.0.2.2 255.255.255.0
 ip nat outside

ip nat inside source list ACL_NAT_INSIDE interface GigabitEthernet0/0 vrf VRF_INSIDE overload
```

#### 検証方法
* `VRF_INSIDE` 側の端末から外部 IP へ Ping を打ち、`show ip nat translations vrf VRF_INSIDE` で NAT 変換エントリーが表示されることを確認。

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 以下のルーターコンフィグが投入されている。`Gi0/1` に着信したパケットに関する記述として最も適切なものはどれか。

```bash
vrf definition RED
 address-family ipv4
 exit-address-family

interface GigabitEthernet0/1
 ip address 10.1.1.1 255.255.255.0
 vrf forwarding RED
```

* A) `Gi0/1` は IP アドレス `10.1.1.1/24` を保持し、VRF RED で正常動作する。
* B) `vrf forwarding RED` コマンドが `ip address` の後に実行されたため、`Gi0/1` の IP アドレスは消去（Unassigned）されている。
* C) `vrf definition RED` に RD (Route Distinguisher) が設定されていないため、ルーターがクラッシュする。
* D) `Gi0/1` は自動的に Global ルーティングテーブルにフォールバックする。

**正解**: **B**  
**解説**: Cisco IOS-XE では、インターフェイスに `vrf forwarding` コマンドを適用すると、既存の IP アドレス設定が即座にクリアされます。そのため、正しく動作させるには `vrf forwarding` を設定した後に `ip address` コマンドを投入する必要があります。

---

### 2. トラブルシュート問題

**問題**: VRF-Lite 環境において、ルーター R1 と R2 の間で VRF `CORP` 配下の OSPFv2 アジャセンシーが `FULL` になっているにもかかわらず、R2 上で R1 から学習したはずの OSPF 外部ルート/要約ルートが RIB (Routing Table) に一切注入されない。原因および解決策として最も適切なものはどれか。

* A) OSPF プロセスで `capability vrf-lite` が設定されておらず、Down-Bit (DN Bit) が立った LSA がループ防止機構によりドロップされている。
* B) OSPF の Area ID が一致していない。
* C) R1 と R2 の間で MPLS LDP が有効化されていない。
* D) BGP との相互再配布が設定されていない。

**正解**: **A**  
**解説**: VRF 固有の OSPF プロセスでは、ルートのループを回避するために LSA タイプ 3/5/7 に Down-Bit (DN Bit) が付与されます。PE-CE 構成ではなく単なる VRF-Lite 構成（MPLS なし）で OSPF を使用している場合、受信ルーターは DN Bit を検知して LSA を破棄します。これを無効化するには OSPF プロセス配下で **`capability vrf-lite`** の設定が必須です。

---

### 3. Design 問題

**問題**: エンタープライズ企業の WAN エッジルーターにおいて、パブリックインターネット網（アンダーレイ）のルーティングテーブルと、暗号化された社内オーバーレイ網（DMVPN）のルーティングテーブルを同一物理ルーター上で完全に分離し、ルーティングループやトラフィックリークを遮断したい。最適な設計オプションはどれか。

* A) ASA Multiple Context Mode
* B) **Front-Door VRF (FVRF)** と **Inside VRF (IVRF)** の分離設計
* C) 全ポートを Access Port に変更した VLAN 制御
* D) Q-in-Q タギングのみによるカプセル化

**正解**: **B**  
**解説**: DMVPN や FlexVPN において、アンダーレイ接続を `FVRF`（Front-Door VRF）にバインドし、IPsec トンネル内部の暗号化パケットを `IVRF`（Inside VRF）に配置する **FVRF / IVRF アーキテクチャ** は、パブリック網とプライベート網のルーティングテーブルを完全に分離し、ルーティングループを根本的に防ぐ Cisco 推奨のベストプラクティス設計です。

---

### 4. 実装問題

**問題**: IOS-XE において、VRF `DEPT_A` から VRF `DEPT_B` への静的 Inter-VRF Route Leaking コマンド `ip route vrf DEPT_A 10.2.2.0 255.255.255.0 GigabitEthernet0/1.20 10.2.2.2` を投入した。通信を双方向で成立させるために追加で必要な対応はどれか。

* A) VRF `DEPT_B` から VRF `DEPT_A` への逆方向の静的ルートリーク（`ip route vrf DEPT_B ...`）を投入する。
* B) `mpls ip` を全インターフェイスに適用する。
* C) `no ip routing` を実行する。
* D) BGP プロセスを起動して `neighbor fall-over` を有効化する。

**正解**: **A**  
**解説**: Inter-VRF Route Leaking は片方向のみの静的経路設定では、対向からの戻りパケット（SYN-ACK や Ping Reply など）が送信元 VRF のテーブルを見つけられず破棄されます。双方向通信を確立するためには、必ず逆方向の VRF からも戻りルートをリークさせる必要があります。

---

### 5. コンフィグ読解問題

**問題**: MP-BGP を用いた Inter-VRF Route Leaking の設定において、VRF `SHARED` で `route-target export 65000:100` が設定され、VRF `USER_A` で `route-target import 65000:100` が設定されている。この設定の結果として正しい説明はどれか。

* A) VRF `SHARED` のルートが MP-BGP 経由で VRF `USER_A` のルーティングテーブルへ自動注入される。
* B) VRF `USER_A` のルートが VRF `SHARED` へ注入される。
* C) 両方の VRF のルートが Global テーブルへエクスポートされる。
* D) BGP セッションが切断される。

**正解**: **A**  
**解説**: `route-target export` で指定された RT 値（`65000:100`）は VPNv4 ルートの属性として付与され、受取側の VRF 内で `route-target import 65000:100` が定義されている場合、その RT 値と一致する VPNv4 ルートが受取側 VRF (`USER_A`) の RIB へ自動挿入されます。

---

## 🔗 参考リソース

* [Cisco IOS-XE Network Virtualization Configuration Guide: VRF-Lite](https://www.cisco.com/)
* [Cisco Technical Notes: Troubleshoot OSPF VRF-Lite Capability and Down-Bit Issues](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Device Configuration Guide: Virtual Routers](https://www.cisco.com/)
* [Cisco Live: Enterprise Network Segmentation with VRF-Lite and TrustSec (BRKSEC-2014)](https://www.ciscolive.com/)
* [Cisco Validated Design (CVD): Campus Network Segmentation Design Guide](https://www.cisco.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で VRF-Lite を構築する際の重要チェックリスト：
    1. `vrf definition <NAME>` ➔ `address-family ipv4` の順で作成する。
    2. インターフェイスに `vrf forwarding <NAME>` を設定してから `ip address` を投入する。
    3. VRF 配下の OSPF では `capability vrf-lite` を即座に投入する。
    4. Inter-VRF Route Leaking では「双方向の経路定義」が存在するか確認する。
    5. DMVPN / FlexVPN では `tunnel vrf FVRF` と `ip vrf forwarding IVRF` を明確に分ける。
* **図解（Front-Door VRF vs Inside VRF トラフィックフロー）**:
  ```text
  [ User Traffic ] ──► (Inbound: Gi0/1) ──► [ Lookup in IVRF_CORP ]
                                                    │
                                         Encapsulated into IPsec
                                                    │
                                         [ Lookup in FVRF_INET ] ──► (Outbound: Gi0/0) ──► [ Public Internet ]
  ```
* **注意点**:
  * ラボ試験中、`show ip route vrf <NAME>` で意図したルートが見えない場合は、まず `show ip interface brief` でインターフェイスの IP アドレスが消去されていないかを疑ってください。
