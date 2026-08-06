---
layout: default
title: 1.11.a-Network-connectivity-through-ASA
nav_order: 1
parent: 1.11-Network-connectivity
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.11.b Network connectivity through Cisco ASA

Cisco ASA（Adaptive Security Appliance）におけるネットワーク接続は、ファイアウォールの基本となる機能であり、パケットがデバイスを通過する際の制御、ルーティング、および管理を司ります。CCIE Security ラボ試験では、インターフェイスの基本設定から、パケット処理の内部シーケンスの理解、および高度なトラブルシューティングツール（Packet Tracer や Capture）の活用能力が厳しく問われます。

---

## 📘 概要

*   **機能概要**: 物理または仮想インターフェイスを論理的なセキュリティゾーンとして定義し、トラフィックの転送（Routed Mode）またはブリッジング（Transparent Mode）を行います。
*   **利用目的**: 異なるセキュリティレベルを持つネットワーク間の分離、ステートフルパケットインスペクションによる通信許可、および管理プレーンへのアクセス制御。
*   **どのような場面で利用するか**: 内部ネットワーク（Inside）から外部（Outside）へのアクセス提供、DMZ 公開サーバーへのトラフィック制御、およびデバイス自体の管理。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **動作モード** | **Routed Mode** (デフォルト、L3 ホップ) / **Transparent Mode** (L2 ステルスファイアウォール)。 |
| **セキュリティレベル** | 0（最低）から 100（最高）の数値。高いレベルから低いレベルへの通信はデフォルトで許可されます。 |
| **インターフェイス要素** | Nameif（論理名）、Security Level（数値）、IP Address。 |
| **デフォルト拒否** | 同一レベル間の通信、および低いレベルから高いレベルへの通信は ACL なしでは拒否されます。 |
| **管理アクセス** | `ssh` や `http` コマンドで明示的に許可されたホスト・インターフェイスのみ ASA を管理可能。 |
| **接続確認ツール** | **Packet Tracer** (内部処理のシミュレーション) および **Capture** (実パケットの追跡)。 |

---

## 🏗 動作原理

ASA の接続性は「フロー」に基づきます。最初のパケットが ASA に到着した際、複数の検証フェーズを経てコネクションテーブルに登録され、以降のパケットは高速に転送されます。

```text
[ Ingress Interface ]
        ↓
[ 1. ACL Check ] <--- Interface ACLs
        ↓
[ 2. Flow Lookup ] <--- Existing Session?
        ↓
[ 3. Route Lookup ] <--- Egress Interface determination
        ↓
[ 4. NAT/XLATE ] <--- Address translation
        ↓
[ 5. Inspection ] <--- MPF/Snort (e.g., ICMP inspect)
        ↓
[ Egress Interface ]
```

---

## ⚙ 動作シーケンス

1.  **インターフェイス受信**: パケットが入力インターフェイスに到着。
2.  **L2/L3検証**: MAC アドレス、IP ヘッダーの整合性を確認。
3.  **セキュリティレベルの適用**: 送信元と宛先の Nameif/Security Level を比較。
4.  **ACL/ポリシー評価**: `access-group` に基づく許可/拒否。
5.  **インスペクションエンジンの介入**: ステートレスなプロトコル（ICMP など）や複雑なプロトコル（FTP/SIP）に対する詳細検査。
6.  **転送**: コネクションが確立され、パケットが送信インターフェイスから送出される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **インターフェイス名の命名**: 試験要件で指定された `nameif` (例: `INSIDE`, `OUTSIDE`) を正確に設定すること。大文字小文字は区別されませんが、慣習に従います。
*   **Security Level の理解**: `inside` という名前を付けると自動的に 100 になりますが、それ以外はデフォルト 0 です。
*   **ICMP インスペクションの罠**: デフォルトでは ASA は ICMP をインスペクションしません。`ping` が通らない場合、ACL で戻りを許可するか、`inspect icmp` を MPF で有効にする必要があります。
*   **Identity インターフェイスへの通信**: ASA 自身の IP 宛の通信（管理アクセス等）は、インターフェイス ACL ではなく、`ssh` や `http` 設定が優先されます。
*   **トラブルシューティングツール**: `packet-tracer` の出力を読み解き、どのフェーズ（Phase 1: ACCESS-LIST, Phase 3: ROUTE-LOOKUP 等）でドロップしているか即座に判断できる必要があります。

---

## 🛠 設定方法

### 1. インターフェイスの基本設定 (Routed Mode)
```bash
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 192.168.1.10 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 10.1.1.10 255.255.255.0
 no shutdown
```

### 2. 同一セキュリティレベル間の通信許可
```bash
! インターフェイスをまたぐ通信の許可
same-security-traffic permit inter-interface
! 同一インターフェイス（U-turn）通信の許可
same-security-traffic permit intra-interface
```

### 3. ICMP インスペクションの有効化 (推奨)
```bash
policy-map global_policy
 class inspection_default
  inspect icmp
!
service-policy global_policy global
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **インターフェイス状態の概要** | <code>show interface ip brief</code> |
| **論理名とセキュリティレベルの確認** | <code>show nameif</code> |
| **コネクションテーブルの確認** | <code>show conn</code> |
| **パケット処理パスのシミュレーション** | <code>packet-tracer input inside icmp 10.1.1.1 8 0 192.168.1.1 detailed</code> |
| **パケットキャプチャの実行** | <code>capture MYCAP interface outside match ip any any</code> |
| **ドロップパケットの統計** | <code>show asp drop</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 直接接続のルータと通信できない | ARP または Security Level | <code>show arp</code> で MAC 学習を確認。対向ルータの IP が正しいか確認。 |
| Ping は飛ぶが戻ってこない | ICMP インスペクション未設定 | <code>packet-tracer</code> で戻りのパケットが拒否されていないか確認。 |
| 特定の宛先へのルートがない | ルーティングテーブルの欠如 | <code>show route</code> で宛先への経路が存在するか確認。 |
| 管理アクセス（SSH/ASDM）が拒否される | 許可ネットワークの未設定 | <code>show run ssh</code> で許可範囲を確認。<code>ssh [network] [mask] [nameif]</code> を追加。 |

---

## ⚠ 制限事項

*   **マルチコンテキストモード**: 一部の機能（動的ルーティングの一部、VPN、脅威検知の一部）に制限があります。
*   **Transparent Mode**: ルーティング機能がなく、L3 の接続性は限定的です（管理用 IP のみ）。
*   **ハードウェア依存**: 物理インターフェイスの MTU 設定や、ハードウェアバイパス機能はモデルに依存します。

---

## 🔄 他技術との関連

*   **Routing**: ASA はスタティック、OSPF、EIGRP、BGP をサポートし、接続性を確立します。
*   **Access Control**: 接続性は ACL によって許可/拒否が決定されます。
*   **NAT**: 接続の際、送信元または宛先 IP が変換されることが多く、接続性のトラブルの一因となります。
*   **VPN**: トンネルインターフェイス（VTI）を介した接続性の構築。

---

## 🧩 比較表

### Routed Mode vs Transparent Mode

| 特徴 | Routed Mode | Transparent Mode |
| :--- | :--- | :--- |
| **L3 ホップ** | あり (デフォルトゲートウェイになる) | なし (L2 ブリッジとして動作) |
| **管理 IP** | 各インターフェイスに必要 | BVI インターフェイスに 1 つ |
| **主な用途** | 一般的なエッジ/境界保護 | 既存 NW 構成を変更しない導入 |
| **ルーティングサポート** | フルサポート | サポートなし |

---

## 💡 ベストプラクティス

1.  **Packet-Tracer の常用**: 設定を変更する前に、`packet-tracer` コマンドで意図したルールにマッチするか確認します。
2.  **Capture による可視化**: 通信不可の原因が ASA 内部なのか対向デバイスなのか不明な場合、`capture` を両インターフェイスで実行します。
3.  **明確な Security Level**: 100(Inside) と 0(Outside) 以外の数値を使用する際は、管理上の意味を明確にします。
4.  **Identity インターフェイスの ACL**: 自身宛のパケットは Identity ACL（`control-plane` ACL）で保護することを検討します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な 2 インターフェイス接続
*   **要件**: 内部 10.1.1.0/24 と外部 172.16.1.0/24 の接続性を確保せよ。
*   **設定**: `nameif inside/outside`, `security-level 100/0`, `ip address` 定義。

### 2. サブインターフェイス (VLAN Trunk) の構成
*   **要件**: Gi0/0 上で VLAN 10 と 20 を使用せよ。
*   **設定**: `interface GigabitEthernet0/0.10`, `vlan 10`, `nameif dmz1`.

### 3. 同一セキュリティレベルの DMZ 分離
*   **要件**: Security Level 50 の DMZ1 と DMZ2 間の通信を許可せよ。
*   **設定**: `same-security-traffic permit inter-interface`.

### 4. Transparent モードへの切り替え
*   **設定**: `firewall transparent` (グローバルコンフィグ)。
*   **注意**: 既存の設定が消去されるため、作業前に保存必須。

### 5. 管理アクセスの制限
*   **要件**: 内部ネットワーク 10.1.1.100 ホストからのみ SSH 管理を許可せよ。
*   **設定**: `ssh 10.1.1.100 255.255.255.255 inside`.

### 6. ICMP デバッグの設定
*   **課題**: 通信障害時、ASA が ICMP をどのように処理しているか追跡せよ。
*   **実行**: `debug icmp trace`。

### 7. ASDM 管理インターフェイスの有効化
*   **設定**: `http server enable`, `http 10.1.1.0 255.255.255.0 inside`.

### 8. Packet-Tracer による ACL マッチ確認
*   **コマンド**: `packet-tracer input inside tcp 10.1.1.5 1025 8.8.8.8 80`。

### 9. 物理インターフェイスの冗長化 (Redundant Interface)
*   **設定**: `interface Redundant1`, `member-interface Gi0/0`, `member-interface Gi0/1`.

### 10. IPv6 接続性の確立
*   **要件**: インターフェイスに IPv6 アドレスを設定し、ND を確認せよ。
*   **設定**: `ipv6 address 2001:db8:1::10/64`, `ipv6 nd prefix ...`.

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `packet-tracer` の出力で `Action: DROP`, `Phase: 3 (ROUTE-LOOKUP)` と表示された。解決策は？
    *   **解答**: 宛先 IP アドレスへのルートが ASA 上に存在しないため、`route` コマンドでスタティックルートを追加するか、動的ルーティングを設定する。
2.  **トラブルシュート**: Inside から Outside への `ping` が通らない。ASA でキャプチャすると `Echo Request` は外へ出ているが、`Echo Reply` が ASA の外部インターフェイスで止まっている。考えられる原因は？
    *   **解答**: ICMP インスペクションが有効でないため、ASA が戻りのパケットを既存フローとして認識できず、暗黙の拒否 (Implicit Deny) でドロップしている。
3.  **Design**: 3 つのインターフェイスを持つ ASA で、全インターフェイスのセキュリティレベルが 0 の場合、デフォルトで通信は可能か？
    *   **解答**: 不可能。同一レベル間の通信を許可する `same-security-traffic permit inter-interface` 設定が必要。
4.  **実装**: 管理者が ASDM に接続できない。ASDM サーバーが有効で、HTTP 許可リストも正しい。ASL（ASA）自体のファイルシステムで確認すべきことは？
    *   **解答**: `show asdm image` で、フラッシュ内に正しい ASDM バイナリが指定されているか確認する。
5.  **コンフィグ読解**: `packet-tracer` の出力で `Phase: 1 (ACCESS-LIST)` にて `Result: DROP` となった。ACL の内容を確認したところ、許可エントリは存在する。他に何を確認すべきか？
    *   **解答**: ACL の適用方向 (`in` か `out` か) と、適用されているインターフェイスが正しいかを確認する。

---

## 🔗 参考リソース

*   **Cisco ASA Series CLI Configuration Guide, 9.4**
    *   [Configuring Interfaces](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/firewall/asa-94-firewall-config/interface-config.html)
    *   [Configuring Management Access](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/firewall/asa-94-firewall-config/access-management.html)
*   **Technical Notes**
    *   [ASA Packet Capturing and Tracer Feature](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/113685-asa-threat-detection-00.html)
*   **CVD / Design Guides**
    *   [Safe Architecture Guide for Firewall Deployment](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Aug2014/CVD-SafeArchitectureGuide-Aug2014.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: ASA は「箱」としての基本が接続性です。インターフェイスが `up` になり、`nameif` が設定されない限り、セキュリティポリシー（ACL や NAT）を適用することはできません。
*   **図解**: 常に `packet-tracer` の各フェーズ（ Phase 1, 2, ...）を頭に描きながら設定を行うのがコツです。
*   **注意点**: ラボ試験の初期状態では `no shutdown` が忘れられていることがあります。まず物理層の確認を怠らないでください。
