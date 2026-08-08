---
layout: default
title: 2.5.d-VRF-Lite
nav_order: 4
parent: 2.5-Infrastructure-segmentation
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.5.d VRF-Lite

**VRF-Lite（Virtual Routing and Forwarding Lite）**は、MPLS（Multi-Protocol Label Switching）を使用せずに、単一のルータやスイッチ内に複数の独立したルーティングテーブル（ルックアップテーブル）を保持する技術です。インフラストラクチャのセグメンテーションにおいて、レイヤ3レベルでの完全な論理分離を実現するために使用されます。

---

## 📘 概要

*   **機能概要**: 物理的な1台のデバイスを、あたかも複数台の仮想的なルータとして動作させます。各VRF（インスタンス）は独自のルーティングテーブルと転送テーブルを持ち、IPアドレスの重複も許可されます。
*   **利用目的**: セキュリティ上の理由によるトラフィックの完全分離、マルチテナント環境の構築、または管理トラフィックとデータトラフィックの分離を目的とします。
*   **利用場面**: 
    *   ゲストネットワークと社内ネットワークのL3分離。
    *   同じIPアドレス体系を持つ複数の顧客を同一のコアスイッチに収容する場合。
    *   ファイアウォールを介したVRF間通信（サンドイッチ構成）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | MPLS Labelを必要としない「Lite」版のVRF実装。 |
| **用途** | エンタープライズやデータセンター内でのL3セグメンテーション。 |
| **メリット** | トラフィックの機密性向上、IPアドレス設計の柔軟性（オーバーラップ可）。 |
| **デメリット** | 設定の複雑化。VRFごとにインターフェイスやルーティングプロセスが必要。 |
| **対応機種** | IOS-XEルータ（ISR/ASR）、Catalystスイッチ、ASA/FTD。 |
| **制限事項** | インターフェイスは1つのVRFにのみ所属可能。 |
| **設計上の注意点** | VRF間の「ルートリーク（Route Leaking）」が必要な場合の静的ルートやBGP管理。 |

---

## 🏗 動作原理

各VRFインスタンスは、独自の**RIB（Routing Information Base）**と**FIB（Forwarding Information Base）**を保持します。

```text
[ VRF: RED ] <--- Interface Gi1 ---> [ Segment A ]
      ↓ (Isolated RIB/FIB)
[ Physical Router ]
      ↑ (Isolated RIB/FIB)
[ VRF: BLUE ] <--- Interface Gi2 ---> [ Segment B ]
```
※ インターフェイスGi1に入ったパケットは、VRF REDのテーブルのみを参照し、VRF BLUEのルートを見ることはできません。

---

## ⚙ 動作シーケンス

1.  **パケット受信**: ルータのインターフェイスでパケットを受信します。
2.  **VRFの特定**: そのインターフェイスが所属するように設定されているVRFを特定します。
3.  **テーブル照会**: 特定されたVRF専用のルーティングテーブル（RIB）を検索します。
4.  **転送判断**: マッチしたエントリに基づき、次ホップを決定します。
5.  **パケット送出**: 出力インターフェイスから送出します。この際、宛先が別VRFである場合、ルートリーク設定がない限りドロップされます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **インターフェイスの所属設定**: `vrf forwarding [NAME]` コマンドをインターフェイスに適用すると、**設定済みのIPアドレスが削除される**というIOSの挙動に注意してください [外部情報]。IPを振る前にVRFを定義・適用するのが鉄則です。
*   **VRF-aware Services**: pingやtracerouteを実行する際、`ping vrf RED [TARGET]` のようにVRFを指定するオプションを忘れないでください。また、SNMPやSyslogなどの管理サービスもVRF経由で動作させる設定が問われます。
*   **Firewallとの組み合わせ**: VRF REDから外部へ出る際、ASAやFTDを経由させる「アーム型」のトポロジが頻出です。
*   **ASAのContextとの違い**: ASAのマルチコンテキストモードに似ていますが、ルータではVRF-Liteを用いて同様の分離を実現します。
*   **ルートリークの実装**: 特定の共有サービス（DNS/NTP）のみに全VRFからアクセスさせるためのスタティックルートによるリーク設定。

---

## 🛠 設定方法

### 1. VRFの定義（IOS-XE）
```bash
vrf definition SEGMENT_A
 address-family ipv4
 ! RD (Route Distinguisher) は必須。一意の値を指定。
 rd 100:1
 exit-address-family
```

### 2. インターフェイスへの適用
```bash
interface GigabitEthernet0/1
 vrf forwarding SEGMENT_A
 ip address 10.1.1.1 255.255.255.0
 no shutdown
```

### 3. VRF対応ルーティング（OSPFの例）
```bash
router ospf 1 vrf SEGMENT_A
 network 10.1.1.0 0.0.0.255 area 0
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **VRF定義とIFの確認** | <code>show vrf</code> |
| **VRFごとのルート確認** | <code>show ip route vrf [NAME]</code> |
| **VRFを指定した疎通確認** | <code>ping vrf [NAME] [IP]</code> |
| **FIBテーブルの確認** | <code>show ip cef vrf [NAME]</code> |
| **インターフェイス詳細** | <code>show ip interface brief</code> (所属VRFも表示) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| インターフェイスのIPが消えた | `vrf forwarding`をIP設定後に実行 | <code>show run int</code>を確認し、IPを再設定する。 |
| 隣接ノードとネイバーが張れない | ルーティングプロセスがVRF未指定 | <code>router ospf [ID] vrf [NAME]</code>を確認。 |
| VRFを跨いだPingが飛ばない | ルートリーク設定の欠如 | <code>show ip route vrf [NAME]</code>に宛先があるか確認。 |
| デフォルトGWが効かない | globalテーブルにのみルートが存在 | <code>ip route vrf [NAME] 0.0.0.0 ...</code>を追加。 |

---

## ⚠ 制限事項

*   **リソース消費**: インスタンスを増やすごとに、ルータのメモリ（RIB/FIB保持用）を消費します。
*   **非対応プロトコル**: 古いIOSや特定の機能（マルチキャストの一部等）はVRF-Lite環境で制限される場合があります。
*   **MPLS VPNとの違い**: ラベル交換（LDP）を行わないため、エンドツーエンドで全デバイスにVRF設定が必要です。

---

## 🔄 他技術との関連

*   **Infrastructure Segmentation (2.5)**: VLAN (2.5.a) や GRE (2.5.c) と組み合わせて、多層的なセグメンテーションを構築します。
*   **VPN (2.1/2.3)**: VTIインターフェイスをVRFに割り当てる「VRF-aware IPsec」により、拠点ごとに独立したVPNを収容可能です。
*   **TrustSec (2.6)**: L3分離のVRFに対し、L2/L3横断的なSGTベースのマイクロセグメンテーションを重ねて適用します。

---

## 🧩 比較表

### VRF-Lite vs MPLS VPN (L3VPN)

| 特徴 | VRF-Lite (Lite) | MPLS L3VPN (Service Provider) |
| :--- | :--- | :--- |
| **プロトコル** | IPのみ | MPLS + MP-BGP |
| **スケーラビリティ** | 小〜中規模向け | 大規模・キャリア向け |
| **複雑さ** | 低い | 高い |
| **必要コンポーネント** | 物理的なVRF設定のみ | PE, P, CEルータとラベル交換 |

---

## 💡 ベストプラクティス

1.  **命名規則の統一**: デバイス間でVRF名（例：GUEST, PROD）を一致させ、運用ミスを防ぎます。
2.  **管理VRFの分離**: デバイス自体の管理（SSH/SNMP）用として、データ用とは別の専用VRF（例：Mgmt-vrf）を使用します。
3.  **ルートリークの制限**: セキュリティを担保するため、リークは最小限のホストルート（/32）に限定します。
4.  **MTUの考慮**: VRF単体では影響しませんが、GREやVPNと併用する場合はオーバーヘッドに注意します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 2つの部門のL3完全分離
*   **要件**: ルータ1台で、Sales用(10.1.1.0)とEng用(10.2.2.0)のルーティングを完全に分けよ。
*   **設定**: `vrf definition Sales`, `vrf definition Eng` を作成し、各IFを割り当て。

### 2. VRF-aware Pingの検証
*   **課題**: ルータから宛先 192.168.1.1 へ ping を飛ばせ。ただしそのターゲットは VRF "BLUE" に属している。
*   **実行**: `ping vrf BLUE 192.168.1.1`。

### 3. スタティックルートによるVRFリーク
*   **要件**: VRF "RED" から、GlobalテーブルにあるDNSサーバー 8.8.8.8 へ疎通させよ。
*   **設定**: `ip route vrf RED 8.8.8.8 255.255.255.255 Gi0/0 1.1.1.1` (Next-hopはGlobalの出口)。

### 4. VRFを用いたOSPFマルチインスタンス
*   **要件**: 同一デバイス上で、2つの異なるOSPFプロセスをVRFごとに走らせよ。

### 5. 管理トラフィックの隔離 (Mgmt-vrf)
*   **要件**: 管理ポート Gi0/0 を `Mgmt-intf` というVRFに所属させ、社内データが流れないようにせよ。

### 6. VRF-Lite on ASA (Context-lite)
*   **要件**: ASAの1つのコンテキスト内で、名前ベースのVRFによるセグメンテーションを行え。

### 7. BGP Multi-protocol VRF
*   **要件**: VRFごとに異なるAS番号（または異なるNeighbor設定）を使用してBGPを回せ。

### 8. VRF-aware NAT
*   **要件**: 特定のVRFからのトラフィックのみを、特定のPublic IPへ変換せよ。

### 9. VRFを跨ぐパケットトレーサー (FTD)
*   **課題**: FMCを使用して、VRF間のアクセス制御ポリシーが正しく動作しているかパケットトレーサーで検証せよ。

### 10. VRF-aware ZBF (Zone-Based Firewall)
*   **要件**: VRF "GUEST" のインターフェイスに専用のZoneを適用し、内部へのアクセスを制限せよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: インターフェイスに `vrf forwarding` を設定した後、以前設定していた IP アドレスで通信ができない。理由を述べよ。
    *   **回答**: `vrf forwarding` コマンドを適用すると、インターフェイスに設定されていた IP アドレスが削除される仕様であるため。
2.  **Design**: IPアドレスの重複（オーバーラップ）が発生している2つのネットワークを1台のルータに収容する場合、どのような技術を使用すべきか？
    *   **回答**: **VRF-Lite** を使用して、各ネットワークを独立したルーティングインスタンスに分離する。
3.  **実装**: VRF "DMZ" に属するホストから Global テーブルにある NTP サーバーへアクセスさせたい。最もシンプルな方法は？
    *   **回答**: VRF スタティックルートを使用して、ターゲット IP へのネクストホップを Global インターフェイスへ向ける（ルートリーク）。
4.  **コンフィグ読解**: `show ip route` を実行したが、期待した特定のサブネットが表示されない。そのサブネットのインターフェイスが `vrf forwarding` に属している場合、どのコマンドを実行すべきか？
    *   **回答**: `show ip route vrf [NAME]`。
5.  **Design**: MPLS ラベルを使用せずに、各拠点間で複数の論理ネットワークを維持したまま転送するための手法は？
    *   **回答**: **VRF-Lite** と **GRE トンネル** を組み合わせる（VRFごとにトンネルを張る）。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [Configuring VRF-lite](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-3/configuration_guide/vlan/b_173_vlan_9300_cg/m_173_vrf_lite_9300.html)
*   **Integrated Security Technologies and Solutions, Volume II**
*   **Technical Notes**
    *   [VRF-lite Configuration Example](https://www.cisco.com/c/en/us/support/docs/ip/virtual-routing-and-forwarding-vrf/200504-VRF-lite-Configuration-Example.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「VRFはレイヤ3のVLANである」と考えると概念を理解しやすくなります。
*   **図解**: 1台の物理ルータの中に、仕切り板（VRF）で区切られた複数の仮想ルータがあるイメージを持ってください。
*   **注意点**: ラボ試験では、`rd` (Route Distinguisher) を設定し忘れると IPv4 `address-family` が有効にならない場合があります。常に `vrf definition` 形式を使用することを推奨します。
