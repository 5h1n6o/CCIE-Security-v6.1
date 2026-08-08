---
layout: default
title: 2.4.b-Dual-hub-DMVPN
nav_order: 2
parent: 2.4-VPN-HA
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.4.b Dual-hub DMVPN deployments

CiscoのDynamic Multipoint VPN（DMVPN）における**デュアルハブ構成**は、ハブ（Next Hop Server: NHS）を冗長化することで、ハブの単一障害点（SPOF）を排除し、ネットワークの可用性を高めるための重要な設計です。CCIE Securityラボ試験では、単に2台のハブを立てるだけでなく、適切なパス選定（Active/Standby または Active/Active）のためのルーティングプロトコルの微調整や、NHRPプロセスの動作理解が深く問われます。

---

## 📘 概要

*   **機能概要**: 1つまたは複数のDMVPNクラウド（Overlayサブネット）内に2台以上のハブを配置し、スポーク（Next Hop Client: NHC）がそれぞれのハブに対してNHRP登録を行う構成です。
*   **利用目的**: ハブデバイスの故障や回線断が発生した際、スポーク間の通信およびハブ・スポーク間の通信を維持する（高可用性）。
*   **どのような場面で利用するか**: 支店数が多い企業のWAN環境において、本社のデータセンターが2拠点ある場合や、同一拠点でルータを冗長化する場合に採用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **トポロジ形式** | **Single Cloud**（同一サブネット）または **Dual Cloud**（別サブネット）。 |
| **冗長性の核** | スポークが複数の **NHS** IPを設定し、それぞれに登録を行う。 |
| **ルーティング制御** | EIGRPの **Delay** や OSPFの **Cost** を使用して優先パスを決定する。 |
| **Phaseの影響** | Phase 3では、冗長パスでもスポーク間通信を最適化するために全ハブで <code>ip nhrp redirect</code> が必須。 |
| **メリット** | ハブの冗長化、トラフィックの負荷分散（Active/Activeの場合）。 |
| **設計上の注意点** | 非対称ルーティングの防止と、ハブ間での経路情報の同期。 |

---

## 🏗 動作原理

デュアルハブ構成には、主に2つのデザインパターンがあります。

### 1. Single Cloud (Dual Hub)
すべてのハブとスポークが単一のTunnelインターフェイスおよびサブネット（例: 172.16.1.0/24）に所属します。
*   スポークは1つのTunnelインターフェイス上で、ハブ1とハブ2の両方にNHRP登録を行います。
*   設定がシンプルですが、障害時の切り替わりはルーティングプロトコルの収束に依存します。

### 2. Dual Cloud (Dual Hub)
ハブごとに独立したDMVPNクラウド（サブネット）を作成します。
*   スポークには2つのTunnelインターフェイス（Tunnel0とTunnel1）が必要です。
*   物理的に異なるキャリア回線を利用する場合などに適しており、障害分離が容易です。

---

## ⚙ 動作シーケンス

1.  **VPN確立**: スポークがハブ1およびハブ2の物理IPに対して個別にISAKMP/IKEv2ネゴシエーションを実施し、IPsecトンネルを確立します。
2.  **NHRP登録**: スポークは `ip nhrp nhs <Hub1>` と `ip nhrp nhs <Hub2>` の設定に基づき、両方のNHSへ登録リクエストを送信します。
3.  **NHRPエントリの作成**: ハブ側では `show ip nhrp` を実行すると、スポークのトンネルIPと物理IP（NBMA）のマッピングが `dynamic` かつ `unique registered` 状態で登録されます。
4.  **ルーティングの伝搬**: トンネル経由でルーティングプロトコル（EIGRP等）を走行させます。ハブは受信したルートを他スポークへ再広報するため、`no ip split-horizon` の設定が必要です。
5.  **フェールオーバー**: 優先ハブのトンネルがダウンすると、ルーティングプロトコルがネイバー断を検知し、メトリックの次点であるセカンダリハブ経由のパスへ切り替わります。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Active/Standbyの明示的な制御**: ラボ要件で「通常はHub 1を使い、障害時のみHub 2を使うこと」と指示された場合、Tunnelインターフェイスで **EIGRP Delay** を変更するスキルが必須です。
*   **NHRP Redirect (Phase 3)**: デュアルハブ環境でもスポーク間直接通信（Shortcut）を行う場合、**すべてのハブ**で `ip nhrp redirect` を有効にする必要があります。
*   **Next-hopの維持**: EIGRPを使用する場合、ハブ側で `no ip next-hop-self eigrp <AS>` を設定し、スポークが学習するルートのNext-hopが学習元のスポークを指すようにします（Phase 2/3）。
*   **OSPF Network Type**: ハブ冗長環境でOSPFを使用する場合、ハブでのDR選出を回避し、Next-hopを適切に処理するために `ip ospf network point-to-multipoint` を使用するのが一般的です。
*   **MTU/MSSの調整**: 冗長パスを通過する際もフラグメンテーションを防ぐため、`ip mtu 1400` および `ip tcp adjust-mss 1360` の一貫した設定が重要です。
*   **検証のポイント**: `show ip nhrp` で複数のハブが登録されているか、`show ip route` で意図したハブがネクストホップになっているかを確認します。

---

## 🛠 設定方法

### 1. ハブの設定 (R1: Primary Hub / Phase 3 / EIGRP)
```bash
interface Tunnel0
 ip address 172.16.1.1 255.255.255.0
 ip nhrp network-id 1
 ip nhrp redirect
 ip nhrp map multicast dynamic
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
!
router eigrp 100
 no ip split-horizon eigrp 100
```

### 2. スポークの設定 (Dual Hub / Single Cloud)
```bash
interface Tunnel0
 ip address 172.16.1.10 255.255.255.0
 ip nhrp network-id 1
 ip nhrp shortcut
 ! Hub 1への静的マッピングと登録
 ip nhrp map 172.16.1.1 203.0.113.1
 ip nhrp map multicast 203.0.113.1
 ip nhrp nhs 172.16.1.1
 ! Hub 2への静的マッピングと登録
 ip nhrp map 172.16.1.2 203.0.113.2
 ip nhrp map multicast 203.0.113.2
 ip nhrp nhs 172.16.1.2
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **ハブ登録状態の確認** | <code>show ip nhrp [brief\|dynamic]</code> |
| **VPNセッションの確認** | <code>show crypto session [detail]</code> |
| **ルーティングパスの確認** | <code>show ip route [network]</code> |
| **EIGRPトポロジの優先度** | <code>show ip eigrp topology</code> |
| **NHRPの詳細ログ** | <code>debug nhrp condition</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 片方のハブにしか登録されない | <code>nhs</code> 設定の重複・欠落 | <code>show run int tunnel</code> で両方の NHS 設定があるか確認。 |
| スポーク間でハブを経由し続ける | <code>redirect</code> または <code>shortcut</code> 欠落 | ハブで <code>redirect</code>、スポークで <code>shortcut</code> を確認。 |
| 障害時に切り替わらない | NHRPホールドタイムが長い | <code>ip nhrp holdtime</code> を短縮（例: 300）し、ルーティングタイマーも調整。 |
| OSPFネイバーが <code>FULL</code> にならない | Network Type の不一致 | <code>ip ospf network point-to-multipoint</code> に揃える。 |
| 特定サイズの通信が失敗 | MTU/MSSの不整合 | <code>ip mtu 1400</code> / <code>ip tcp adjust-mss 1360</code> を再確認。 |

---

## ⚠ 制限事項

*   **ハブ間通信**: 2台のハブが互いのNBMAアドレスを知る（あるいは直接接続される）必要があり、そうでないとハブをまたいだスポーク間直接通信が失敗することがあります。
*   **リソース消費**: デュアルハブ構成では、スポークが維持するIPsec SA数が倍増するため、低スペックルータではCPU/メモリ負荷に注意が必要です。
*   **非対称ルーティング**: ルーティングメトリックが適切に設計されていないと、行きと帰りで別のハブを通る「非対称パス」が発生し、ステートフルFW（ASA等）が介在する場合に通信が遮断されます。

---

## 🔄 他技術との関連

*   **EIGRP/OSPF**: デュアルハブにおけるパス選定を制御します。
*   **IKEv2**: ラボ試験ではレガシーなIKEv1ではなく、IKEv2ベースのDMVPNが問われることが多くなっています。
*   **GET VPN**: デュアルハブに似た冗長化手法として、キーサーバー（KS）の冗長化（COOP）がありますが、DMVPNとは動作が異なります。
*   **Infrastructure Segmentation**: 異なるVRFごとにデュアルハブ構成を組む「VRF-Aware DMVPN」への発展。

---

## 🧩 比較表

### Single Cloud vs Dual Cloud (Dual Hub)

| 特徴 | Single Cloud | Dual Cloud |
| :--- | :--- | :--- |
| **スポークIF数** | 1 (Tunnel0) | 2 (Tunnel0, Tunnel1) |
| **サブネット** | 全拠点で同一のOverlayサブネット | ハブごとに異なるOverlayサブネット |
| **設計の複雑さ** | 低（インターフェイスが少ない） | 高（メトリック調整がより複雑） |
| **冗長性のレベル** | ハブデバイスの冗長化 | インフラ・ISPレベルの完全冗長化 |

---

## 💡 ベストプラクティス

1.  **Phase 3 の採用**: スケーラビリティと直接通信の最適化のため、常にPhase 3（Redirect/Shortcut）を使用します。
2.  **メトリックの一貫性**: 優先パスを明確にするため、EIGRP Delayなどを全スポークで一貫して設定します。
3.  **ハブ間の直接接続**: 2台のハブ間をLANまたは独立したリンクで接続し、DMVPNトンネルを介さずに経路情報を同期させることで安定性を高めます。
4.  **BFDの併用**: トンネル上でのBFD (Bidirectional Forwarding Detection) を使用することで、数秒以内での高速な障害検知と切り替えを可能にします [Cisco公式]。

---

## 📝 ラボ学習・設定サンプル例

### 1. 2台のハブへの同時登録 (Single Cloud)
*   **要件**: スポークR3から、R1(NHS1)とR2(NHS2)の両方にNHRP登録を行え。
*   **設定**: `ip nhrp nhs 172.16.1.1`, `ip nhrp nhs 172.16.1.2`。

### 2. EIGRP メトリックによる優先制御
*   **要件**: R1経由を優先し、R2経由のルートメトリックを悪くせよ。
*   **設定**: R2側のTunnelインターフェイスで `delay 2000` を設定。

### 3. Phase 3 NHRP Redirect の実装
*   **要件**: ハブ冗長環境で、スポーク間が常に最短パスを通るようにせよ。
*   **設定**: 全ハブに `ip nhrp redirect`、全スポークに `ip nhrp shortcut`。

### 4. OSPF Point-to-Multipoint による冗長
*   **要件**: 2台のハブ環境でOSPFネイバーを安定させ、DR選出を回避せよ。

### 5. ハブ故障時のフェールオーバーテスト
*   **課題**: R1のTunnelを `shutdown` し、R2経由に切り替わる時間を測定せよ。

### 6. NHRP 登録パスワードの共有
*   **要件**: 2台のハブで共通のNHRP認証キーを設定せよ。
*   **設定**: `ip nhrp authentication lab123`。

### 7. Dual Cloud での VTI 冗長
*   **要件**: スポークにTunnel0 (Hub1宛) と Tunnel1 (Hub2宛) を構成せよ。

### 8. IKEv2 Wildcard PSK による保護
*   **要件**: 全ハブで共通の IKEv2 Keyring を使用し、スポークを収容せよ。

### 9. NHRP Holdtime の調整
*   **要件**: 冗長パスの収束を早めるため、ホールドタイムを300秒に短縮せよ。

### 10. ハブ間での BGP ルート交換
*   **課題**: ハブ同士を iBGP で接続し、全スポークのルート情報を同期せよ。

---

## ❓ 想定試験問題

1.  **Design**: Single Cloud のデュアルハブ構成において、スポークが両方のハブから同じルートを受信した場合、特定のハブを優先させるための推奨設定は？
    *   **回答**: EIGRP を使用している場合は、バックアップハブ側のインターフェイスで `delay` を大きく設定し、OSPF の場合は `cost` を上げる。
2.  **トラブルシュート**: スポークが Hub 1 には登録できているが、Hub 2 に登録できない。Hub 2 への `ping` は通る。確認すべき NHRP 設定は？
    *   **回答**: スポーク側で Hub 2 の物理 IP に対する `ip nhrp map` が設定されているか、および `ip nhrp nhs` の設定に誤りがないか確認する。
3.  **コンフィグ読解**: `ip nhrp redirect` コマンドが設定されていない Phase 3 ハブ環境での通信の挙動は？
    *   **回答**: スポーク間通信は常にハブを経由し続け、ショートカットトンネルが作成されない。
4.  **実装**: EIGRP を使用する DMVPN で、ハブがスポークから学習したルートを他のスポークへ再広報するために必須のインターフェイスコマンドは？
    *   **回答**: `no ip split-horizon eigrp <AS>`.
5.  **Design**: Dual Cloud デザインを採用する最大のメリットは？
    *   **回答**: コントロールプレーン（NHRP）が物理キャリアごとに分離されるため、一方の障害が他方に影響せず、高い可用性が得られる。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [Dynamic Multipoint VPN (DMVPN) Phase 3 Configuration](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_conn_dmvpn/configuration/xe-16/sec-conn-dmvpn-xe-16-book.html)
*   **Cisco Live (BRKSEC-3052)**
    *   [DMVPN - Phases, Implementation, and Troubleshooting](https://www.ciscolive.com/global.html)
*   **Technical Notes**
    *   [Troubleshooting DMVPN Phase 3 Redirect and Shortcut](https://www.cisco.com/c/en/us/support/docs/security-vpn/dynamic-multipoint-vpn-dmvpn/119022-technote-dmvpn-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: デュアルハブは「設定の重複」ではなく「論理的な冗長」です。スポークのTunnel設定を書く際、2つ目のハブ向けの設定が漏れていないか指差し確認しましょう。
*   **注意点**: ラボ試験では、ハブをシャットダウンした後の「切り替わり時間」も採点基準になることがあります。`ip nhrp holdtime` やルーティングタイマーの短縮指示を見落とさないでください。
