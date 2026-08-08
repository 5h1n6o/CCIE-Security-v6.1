---
layout: default
title: 2.4-VPN-HA
nav_order: 4
parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.4.b Dual-hub DMVPN deployments

Cisco の Dynamic Multipoint VPN（DMVPN）におけるデュアルハブ構成は、ハブ（NHS: Next Hop Server）を冗長化することで、ハブの単一障害点（SPOF）を排除し、ネットワークの可用性を高めるための重要な設計です。CCIE Security ラボ試験では、単に 2 台のハブを立てるだけでなく、適切なパス選定（Active/Standby または Active/Active）のためのルーティングプロトコルの微調整や、NHRP プロセスの動作理解が深く問われます。

---

## 📘 概要

*   **機能概要**: 1 つまたは複数の DMVPN クラウド内に 2 台以上のハブ（NHS）を配置し、スポーク（NHC）がそれぞれのハブに対して NHRP 登録を行う構成です。
*   **利用目的**: ハブデバイスの故障や回線断が発生した際、スポーク間の通信およびハブ・スポーク間の通信を維持する（高可用性）。
*   **どのような場面で利用するか**: 支店数が多い企業の WAN 環境において、本社のデータセンターが 2 拠点ある場合や、同一拠点でルータを冗長化する場合に採用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **トポロジ形式** | **Single Cloud**（同一サブネット）または **Dual Cloud**（別サブネット）。 |
| **冗長性の核** | スポークが複数の **NHS** IP を設定し、それぞれに登録を行う。 |
| **ルーティング制御** | EIGRP の **Delay** や OSPF の **Cost** を使用して優先パスを決定する。 |
| **Phase の影響** | Phase 3 では両ハブで `ip nhrp redirect` が必須となる。 |
| **メリット** | ハブの冗長化、トラフィックの負荷分散（Active/Active の場合）。 |
| **設計上の注意点** | 非対称ルーティングの防止と、ハブ間での経路情報の同期。 |

---

## 🏗 動作原理

デュアルハブ構成には、主に 2 つのデザインパターンがあります。

### 1. Single Cloud (Dual Hub)
すべてのハブとスポークが単一の Tunnel インターフェイスおよびサブネット（例: 172.16.1.0/24）に所属します。
*   スポークは 1 つの Tunnel IF 上で、ハブ 1 とハブ 2 の両方に NHRP 登録を行います。
*   最も一般的な構成であり、設定がシンプルですが、障害時の切り替わりはルーティングプロトコルの収束に依存します。

### 2. Dual Cloud (Dual Hub)
ハブごとに独立した DMVPN クラウド（サブネット）を作成します。
*   スポークには 2 つの Tunnel インターフェイス（Tunnel 0 と Tunnel 1）が必要です。
*   物理的に異なるキャリア回線を利用する場合などに適しており、障害分離が容易です。

---

## ⚙ 動作シーケンス

1.  **IKE/IPsec 確立**: スポークがハブ 1 および ハブ 2 の物理 IP に対して個別に VPN トンネルを確立します。
2.  **NHRP 登録**: スポークは `ip nhrp nhs <Hub1>` と `ip nhrp nhs <Hub2>` の設定に基づき、両方の NHS へ登録リクエストを送信します。
3.  **ルーティング情報の受信**: スポークは両方のハブから動的ルーティングプロトコル（EIGRP 等）を介してルートを学習します。
4.  **パス選定**: ルーティングメトリックに基づき、優先ハブが決定されます。
5.  **フェールオーバー**: 優先ハブ（R1）のトンネルがダウンすると、ルーティングプロトコルがネイバー断を検知し、メトリックの次点であるハブ 2（R2）経由のパスへ切り替わります。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Active/Standby の明示的な制御**: ラボ要件で「通常は R1 を使い、障害時のみ R2 を使うこと」と指示された場合、Tunnel インターフェイスで **EIGRP Delay** や **OSPF Cost** を変更するスキルが必須です。
*   **NHRP Redirect (Phase 3)**: デュアルハブ環境でもスポーク間直接通信を行う場合、**すべてのハブ**で `ip nhrp redirect` を有効にする必要があります。
*   **NHS 設定の順序**: スポーク側で `ip nhrp nhs` を記述する際、プライマリとセカンダリの順序や、静的なマッピング (`ip nhrp map`) の正確性が問われます。
*   **MTU と TCP MSS**: 冗長構成によりパケットのパスが変わっても、フラグメンテーションが発生しないよう、全ハブ・スポークで一貫した MTU 設定（通常 1400）が必要です。
*   **検証スキル**: `show ip nhrp` で複数のハブが `unique registered` となっているか、`show ip route` でネクストホップが意図したハブを指しているかを確認できることが合格の鍵です。

---

## 🛠 設定方法

### 1. スポークの設定 (Single Cloud / Phase 3 / EIGRP)
```bash
interface Tunnel0
 ip address 172.16.1.10 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication cisco123
 ip nhrp network-id 1
 ! ハブ1(R1)へのマッピングと登録
 ip nhrp map 172.16.1.1 203.0.113.1
 ip nhrp map multicast 203.0.113.1
 ip nhrp nhs 172.16.1.1
 ! ハブ2(R2)へのマッピングと登録
 ip nhrp map 172.16.1.2 203.0.113.2
 ip nhrp map multicast 203.0.113.2
 ip nhrp nhs 172.16.1.2
 ip nhrp shortcut
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
```

### 2. メトリックによる優先制御 (スポーク側)
ハブ 2 へのパスをバックアップにするため、メトリックを悪くします。
```bash
! EIGRP の場合 (Delay を大きくする)
interface Tunnel0
 ! 実際の経路制御はハブ側からの広報メトリックや、スポーク側での受信時調整で行う
 ! 以下の例は、特定のハブ向けのトンネルインターフェイスが分かれている Dual Cloud の場合
 interface Tunnel1
  delay 2000  ! Hub2 側を優先度低にする
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **ハブ登録状態の確認** | <code>show ip nhrp [brief\|dynamic]</code> |
| **VPN セッションの確認** | <code>show crypto session [detail]</code> |
| **ルーティングパスの確認** | <code>show ip route [network]</code> |
| **EIGRP トポロジの優先度** | <code>show ip eigrp topology</code> |
| **パケットパスのトレース** | <code>traceroute [destination]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 片方のハブにしか登録されない | `nhs` 設定の欠落 | <code>show run interface tunnel</code> で両方の NHS IP があるか確認。 |
| ハブ間でルートがループする | Split-horizon 設定ミス | ハブのトンネル IF で <code>no ip split-horizon eigrp</code> を確認。 |
| 障害時に切り替わらない | NHRP ホールドタイムが長い | <code>ip nhrp holdtime</code> を短縮し、ルーティングの Hello/Hold タイムも調整。 |
| スポーク間でハブを経由し続ける | `redirect` の欠落 | ハブ 1 とハブ 2 の両方で <code>ip nhrp redirect</code> があるか確認。 |
| 非対称ルーティングが発生 | 戻りルートのメトリック不一致 | 両ハブからのルート広報メトリックが対称（あるいは意図した比率）か確認。 |

---

## ⚠ 制限事項

*   **ハブ間通信**: ハブ同士が直接通信できるルート（または専用の DMVPN トンネル）がないと、ハブをまたいだスポーク間のショートカットが失敗する場合があります。
*   **OSPF の制約**: `point-to-multipoint` ネットワークタイプを使用しないと、ネクストホップの解決で問題が発生しやすい。
*   **パフォーマンス**: デュアルハブ構成では、スポークが維持する SA (Security Association) 数が倍増するため、デバイスのリソース消費に注意が必要です。

---

## 🔄 他技術との関連

*   **EIGRP/OSPF**: デュアルハブにおけるパス選定の「脳」となります。
*   **IKEv2**: CCIE v6.1 では IKEv1 ではなく IKEv2 による Dual-hub 実装が標準です。
*   **IPsec Profile**: 冗長化されたトンネルを保護するために、同一の `crypto ipsec profile` を全デバイスで一貫して使用します。
*   **BGP**: 超大規模なデュアルハブ DMVPN では、EIGRP の代わりに BGP を使用してスケーラビリティを確保します。

---

## 🧩 比較表

### Single Cloud vs Dual Cloud

| 特徴 | Single Cloud (同一サブネット) | Dual Cloud (別サブネット) |
| :--- | :--- | :--- |
| **スポーク IF 数** | 1 (Tunnel0) | 2 (Tunnel0, Tunnel1) |
| **設計の複雑さ** | 低 | 高 |
| **冗長性のレベル** | ハブデバイスの冗長 | 回線・クラウドレベルの完全冗長 |
| **ルーティング** | メトリック調整が重要 | インターフェイスレベルで制御可能 |

---

## 💡 ベストプラクティス

1.  **Phase 3 の採用**: スケーラビリティと直接通信の最適化のため、常に Phase 3 を使用します。
2.  **一貫したポリシー**: ハブ 1 と ハブ 2 で、NHRP 認証キー、ネットワーク ID、IPsec プロファイルを完全に一致させます。
3.  **BFD の併用**: デュアルハブ環境で高速なフェールオーバーを実現するため、トンネル上での BFD (Bidirectional Forwarding Detection) の使用を検討します。
4.  **ハブ間の直接接続**: 2 台のハブ間は LAN または専用リンクで接続し、DMVPN トンネルを介さずに経路交換を行うことで安定性を高めます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 2台のハブへの同時登録 (Single Cloud)
*   **要件**: スポーク R3 から、R1(NHS1) と R2(NHS2) の両方に NHRP 登録を行え。
*   **設定**: `ip nhrp nhs 172.16.1.1`, `ip nhrp nhs 172.16.1.2`。

### 2. EIGRP メトリックによる Active/Standby 制御
*   **要件**: R1 経由を優先し、R2 経由のルートメトリックを 100 加算せよ。
*   **手法**: R2 側の広報メトリックを `offset-list` または `delay` で調整。

### 3. OSPF Point-to-Multipoint による冗長構成
*   **要件**: ハブ冗長環境で OSPF を使用し、スポークで DR 選出を回避せよ。
*   **設定**: `ip ospf network point-to-multipoint`。

### 4. ハブ故障時の自動切り替えテスト
*   **課題**: R1 のトンネルを `shutdown` し、R2 経由に 10 秒以内に切り替わることを確認せよ。

### 5. Dual Cloud 環境での VTI 構成
*   **要件**: スポークに Tunnel0 (Hub1 宛) と Tunnel1 (Hub2 宛) を作成せよ。

### 6. IKEv2 Wildcard PSK によるハブ冗長
*   **要件**: 2台のハブで共通の PSK を使用し、スポークを収容せよ。

### 7. Phase 3 NHRP Redirect のデバッグ
*   **要件**: R2 がプライマリとして動作している際、Redirect パケットが送信されるか確認せよ。
*   **コマンド**: `debug nhrp packet`。

### 8. スポーク間直接通信の優先パス確認
*   **要件**: R1 経由と R2 経由で、どちらのショートカットパスが優先されるか確認せよ。

### 9. ハブ間での NHRP エントリ同期
*   **課題**: ハブ同士を BGP で接続し、NHS データベースの整合性を保て。

### 10. MTU 不一致による冗長パスの通信不可トラブルシュート
*   **課題**: Hub2 経由のパスのみ Ping が失敗する原因（MTU）を特定せよ。

---

## ❓ 想定試験問題

1.  **Design**: Single Cloud のデュアルハブ構成において、スポークが両方のハブから同じネットワークのルートを受信した場合、どのように優先順位を決定すべきか？
    *   **回答**: ルーティングプロトコルのメトリック（EIGRP Delay、OSPF Cost 等）を調整して、優先したいハブ経由のパスを最適（最小メトリック）にする。
2.  **トラブルシュート**: スポークが R1(Hub1) には登録できているが、R2(Hub2) に登録できない。`show ip nhrp` では R2 が `incomplete` となっている。原因は？
    *   **回答**: スポーク側で R2 の物理 IP に対する `ip nhrp map` が欠落している、あるいは R2 との間の IPsec SA が確立できていない。
3.  **コンフィグ読解**: スポークの Tunnel 設定に `ip nhrp nhs 172.16.1.1` が 2 行ある場合（異なる IP）の挙動を述べよ。
    *   **回答**: スポークは両方の NHS に対して登録を試みる。これはデュアルハブにおける標準的な冗長化設定である。
4.  **実装**: Phase 3 のデュアルハブ環境で、R1 故障時に R2 経由でスポーク間直接通信を維持するために必要なコマンドは？
    *   **回答**: ハブ R2 の Tunnel インターフェイスにおける `ip nhrp redirect`。
5.  **Design**: Dual Cloud デザインが Single Cloud よりも優れている点は？
    *   **回答**: コントロールプレーンが物理回線レベルで完全に分離されるため、一方のクラウドの NHRP 不具合が他方に波及せず、高い可用性を確保できる。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [Dynamic Multipoint VPN (DMVPN) Phase 3 Configuration](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_conn_dmvpn/configuration/xe-16/sec-conn-dmvpn-xe-16-book.html)
*   **Cisco Live (BRKSEC-3052)**
    *   [DMVPN - Phases, Implementation and Troubleshooting](https://www.ciscolive.com/)
*   **Technical Notes**
    *   [DMVPN and Dual-Hub Designs](https://www.cisco.com/c/en/us/support/docs/security-vpn/dynamic-multipoint-vpn-dmvpn/119022-technote-dmvpn-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: デュアルハブは「設定の重複」ではなく「論理的な冗長」です。スポークの Tunnel 設定を書き出すとき、2つ目のハブ向けの設定が漏れていないか指差し確認しましょう。
*   **図解**: 常にスポークから見て「2本の暗号化トンネルがハブに向かって立っている」イメージを持ってください。
*   **注意点**: ラボ試験では、ハブをシャットダウンした後の「切り替わり時間」も採点基準になることがあります。`ip nhrp holdtime` やルーティングタイマーの短縮指示を見落とさないでください。
