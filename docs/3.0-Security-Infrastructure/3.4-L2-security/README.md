---
layout: default
title: 3.4-L2-security
nav_order: 4
parent: 3.0-Security-Infrastructure
---

# 3.4 Layer 2 security techniques

L2 セキュリティ技術は、OSI 参照モデルの第 2 層（データリンク層）における攻撃を防御するための重要な境界線です。**DHCP Snooping**、**DAI (Dynamic ARP Inspection)**、**Port Security** などの技術を適切に組み合わせることで、MAC フローディング、ARP スプーフィング、DHCP サーバーのなりすまし、および STP の不安定化といった脅威からインフラを保護します。

---

## 📘 概要

*   **機能概要**: スイッチポートレベルでパケットの正当性を検証し、許可されていないデバイスや不正なプロトコルメッセージ（BPDU, DHCP Reply, RA 等）を遮断する技術群です。
*   **利用目的**: 中間者攻撃 (MITM)、IP/MAC スプーフィング、VLAN ホッピング、STP ルートハイジャックの防止。
*   **どのような場面で利用するか**: エンタープライズネットワークのアクセス層（Access Layer）において、ユーザー端末が接続されるポートの「ハードニング（要塞化）」として必須となります。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **DHCP Snooping (3.4.e)** | 不正 DHCP サーバーを遮断し、DAI や IPDT の基盤となるバインディング DB を構築する。 |
| **DAI (3.4.a)** | ARP パケットを検証し、送信元 IP/MAC の不整合（ARP スプーフィング）を防止する。 |
| **STP Security (3.4.c)** | BPDU Guard や Root Guard を使用して、STP トポロジの予期せぬ変更を防ぐ。 |
| **Port Security (3.4.d)** | ポートに接続可能な MAC アドレス数を制限し、MAC Flooding 攻撃を防止する。 |
| **RA Guard (3.4.f)** | IPv6 環境において、ホストからの不正なルータ広告 (RA) を遮断する。 |
| **VACL (3.4.g)** | VLAN 内の通信を L2 レイヤでフィルタリングし、同一サブネット内のアクセスを制御する。 |

---

## 🏗 動作原理

L2 セキュリティは、**「Trust (信頼)」と「Untrust (非信頼)」**の概念に基づいています。通常、ユーザーが接続されるポートを `Untrusted` とし、上位スイッチや正当なサーバーが接続されるポートを `Trusted` と定義します。

```text
[ Trusted Port (Uplink) ] <--- DHCP Reply, ARP, BPDU を許可
       ↑
[ Access Switch ]
       ↓
[ Untrusted Port (User) ] <--- 不正な DHCP Server 機能を遮断, ARP 検証
```

---

## ⚙ 動作シーケンス (DHCP Snooping + DAI)

1.  **DHCP Snooping 有効化**: スイッチがポートを流れる DHCP トラフィックの監視を開始します。
2.  **バインディング DB 構築**: `Untrusted` ポートの端末が IP を取得した際、その IP/MAC/VLAN/Port をデータベースに記録します。
3.  **ARP 受信**: `Untrusted` ポートから ARP パケットが到着します。
4.  **DAI 検証**: スイッチが ARP パケット内の情報を DHCP Snooping バインディング DB と照合します。
5.  **アクション**: 一致すれば転送、一致しなければパケットをドロップし、必要に応じてログを出力します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **依存関係の理解**: 「DAI を設定せよ」という要件には、暗黙的に **DHCP Snooping** の有効化が必要である（さもなければ全パケットがドロップされる）ことを忘れないでください。
*   **Err-disable 復旧**: Port Security や BPDU Guard でポートが `Err-disabled` になった際の自動復旧設定 (`errdisable recovery cause...`) が問われることがあります。
*   **VACL の適用順序**: VACL は RACL（L3 ACL）よりも前に処理されるため、フィルタリングの優先順位に注意が必要です。
*   **IPv6 RA Guard**: IPv6 セキュリティの基本として、ポリシー作成とインターフェイスへの適用手順 (`ipv6 nd raguard attach-policy`) を正確に把握してください。
*   **Static Binding**: 静的 IP を持つ端末（プリンタ等）がある場合、DHCP Snooping DB に手動でエントリを追加する (`ip source binding...`) か、DAI で例外 ACL を作成する必要があります。

---

## 🛠 設定方法

### 1. DHCP Snooping & DAI
```bash
ip dhcp snooping
ip dhcp snooping vlan 10
!
interface GigabitEthernet0/1
 description UPLINK-TO-CORE
 ip dhcp snooping trust
 ip arp inspection trust
!
interface GigabitEthernet0/2
 description USER-PORT
 ip arp inspection vlan 10
```

### 2. STP Security
```bash
interface GigabitEthernet0/2
 spanning-tree bpduguard enable
 spanning-tree guard root
```

### 3. Port Security
```bash
interface GigabitEthernet0/2
 switchport mode access
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation shutdown
 switchport port-security mac-address sticky
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **DHCP バインディングの確認** | <code>show ip dhcp snooping binding</code> |
| **DAI の統計と状態確認** | <code>show ip arp inspection vlan [ID]</code> |
| **Port Security の違反確認** | <code>show port-security interface [ID]</code> |
| **STP ガードの状態確認** | <code>show spanning-tree summary</code> |
| **VACL の適用確認** | <code>show vlan filter</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 端末が通信できない (DAI) | バインディング DB にエントリがない | <code>show ip arp inspection statistics</code> | DHCP Snooping の有効化または静的バインド追加。 |
| ポートが Down する | Port Security 違反または BPDU 受信 | <code>show interface status err-disabled</code> | 原因除去後、<code>shut/no shut</code> または自動復旧設定。 |
| DHCP パケットが通らない | Trust ポートの設定漏れ | <code>show ip dhcp snooping</code> | 上位ポートに <code>ip dhcp snooping trust</code> を設定。 |
| IPv6 RA Guard が効かない | ポリシーの適用方向ミス | <code>show ipv6 nd raguard</code> | 適切なターゲットポートに attach する。 |

---

## ⚠ 制限事項

*   **ハードウェアリソース**: DHCP Snooping バインディング DB はメモリを消費するため、数千ホストを収容するスイッチではリソース枯渇に注意が必要です。
*   **CPU 負荷**: DAI や VACL で多量のロギングを有効にすると、スイッチの CPU 負荷が高まります。
*   **DHCP Option 82**: スイッチがデフォルトで挿入する Option 82 を、上位のルータが拒否して IP 取得に失敗するケースがあります (`no ip dhcp snooping information option` 等で対処)。

---

## 🔄 他技術との関連

*   **ISE (Device Profiling)**: IPDT (3.4.b) は、ISE がネットワーク上の端末を正確に識別するために使用されます。
*   **uRPF (3.3.a)**: L2 レベルでのスプーフィング対策 (DAI) と L3 レベルでの対策 (uRPF) を組み合わせることで、送信元偽装を多層防御します。
*   **802.1X**: Port Security と 802.1X は排他的な動作をすることが多いため、同時使用時の仕様に注意が必要です。

---

## 🧩 比較表

### Port Security vs 802.1X

| 特徴 | Port Security | 802.1X (DOT1X) |
| :--- | :--- | :--- |
| **認証単位** | MAC アドレスの「数」 | ユーザー/デバイスの「ID」 |
| **複雑さ** | 低（スイッチのみで完結） | 高（RADIUS/ISE が必要） |
| **セキュリティ強度** | 中（MAC は偽装可能） | 高（証明書やパスワードを使用） |
| **主な用途** | MAC Flooding 対策 | 厳格なアクセス制御 |

---

## 💡 ベストプラクティス

1.  **エッジポートの標準化**: すべてのアクセスポートに `spanning-tree portfast` と `bpduguard enable` を設定します。
2.  **Untrusted Port の制限**: アクセスポートには一律で `ip dhcp snooping limit rate` を設定し、DHCP パケットによる DoS を防ぎます。
3.  **データベースの永続化**: スイッチ再起動時にバインディング DB が消えないよう、Flash や外部 TFTP サーバーに保存する設定 (`ip dhcp snooping database...`) を推奨します。

---

## 📝 ラボ学習・設定サンプル例

### 1. ポートセキュリティの基本
*   **要件**: ポートにつき 1 台の MAC のみ許可し、違反時はログを出してポートを閉じよ。
*   **設定**: `switchport port-security`, `switchport port-security violation shutdown`。

### 2. DAI と Static Binding の併用
*   **要件**: DHCP を使わないサーバー（192.168.1.100）の ARP を許可せよ。
*   **設定**: `ip source binding [MAC] vlan [ID] 192.168.1.100 interface [ID]`。

### 3. VACL による特定プロトコルの遮断
*   **要件**: VLAN 10 内での Telnet 通信のみを拒否せよ。
*   **設定**: `vlan access-map` 内で Telnet を `drop` し、他を `forward`。

### 4. DHCP Snooping Rate Limit
*   **要件**: 不正な DHCP クライアントからの攻撃を防ぐため、毎秒 10 パケットに制限せよ。

### 5. IPv6 RA Guard の実装
*   **要件**: GigabitEthernet1/0/1 からの IPv6 ルータ広告をすべて破棄せよ。

### 6. BPDU Guard の自動復旧
*   **要件**: BPDU 受信で落ちたポートを 5 分後に自動で再開させよ。

### 7. DAI のロギング制限
*   **要件**: 大量の ARP 違反ログで CPU が飽和しないよう、ログの生成レートを制限せよ。

### 8. IPDT (SISF) の設定
*   **要件**: 端末の IP アドレスを監視し、ISE へ通知できるようにせよ。

### 9. Sticky MAC による永続管理
*   **要件**: 最初に接続された端末の MAC を保存し、再起動後も固定せよ。

### 10. Root Guard によるルート防衛
*   **要件**: 特定のポートから接続される外部スイッチが STP ルートブリッジにならないようにせよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: DAI 有効化後、一部の端末が通信できない。`show ip dhcp snooping binding` が空である。原因は？
    *   **回答**: DHCP Snooping が有効だが、当該ポートが `Untrusted` で、且つ端末がまだ DHCP プロセスを完了していないため。
2.  **Design**: STP トポロジの安定性を確保するために、アクセスポートで有効にすべき機能は？
    *   **回答**: **BPDU Guard**。
3.  **コンフィグ読解**: `switchport port-security violation restrict` の動作は？
    *   **回答**: ポートは Up のままで、不正 MAC からのパケットのみ破棄し、SNMP トラップとログを生成する。
4.  **実装**: IPv6 環境でホストが誤ってルータとして振る舞うのを防ぐ設定は？
    *   **回答**: **IPv6 RA Guard**。
5.  **Design**: VLAN 内の通信を IP アドレスだけでなく L2 レイヤの全てのトラフィック（Non-IP 含む）に対して制御したい。何を使用すべきか？
    *   **回答**: **VACL (VLAN ACL)**。

---

## 🔗 参考リソース

*   **Cisco Configuration Guide**
    *   [Configuring DHCP Snooping](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3750/software/release/12-2_55_se/configuration/guide/scg3750/swdhcp82.html)
    *   [Configuring Dynamic ARP Inspection](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3750/software/release/12-2_55_se/configuration/guide/scg3750/swdynarp.html)
*   **Cisco Live (BRKSEC-2202)**
    *   [Securing the Layer 2 Infrastructure](https://www.ciscolive.com/)
*   **Technical Notes**
    *   [Cisco Guide to Harden Cisco IOS Devices: Layer 2 Hardening](https://www.cisco.com/c/en/us/support/docs/ip/access-lists/13608-21.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「L2 セキュリティは、物理的な繋ぎ込みに対する信頼の定義」です。
*   **注意点**: ラボ試験では、各機能のコマンドが似ているため (`ip dhcp snooping` vs `ip arp inspection` 等)、混同しないように実機での反復練習が不可欠です。
