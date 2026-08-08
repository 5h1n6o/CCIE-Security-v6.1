---
layout: default
title: 3.1-Device-hardening
nav_order: 1
parent: 3.0-Security-Infrastructure
---

# 3.1 Device hardening techniques and control plane protection methods

Cisco デバイスのハードニングとコントロールプレーン保護は、ネットワークインフラストラクチャ全体の安定性とセキュリティを確保するための第一防衛線です。デバイス自体を攻撃から守り、ルーティングや管理プロセスを処理する CPU リソースを枯渇（DoS 攻撃）から保護する手法を整理します。

---

## 📘 概要

*   **機能概要**: 
    *   **Device Hardening**: 不要なサービスの停止、管理アクセスの暗号化、アクセス制限などを通じてデバイスの攻撃面を最小化します。
    *   **Control Plane Protection**: デバイスの CPU へ向かうトラフィックをフィルタリングまたはレート制限し、正当な制御トラフィック（ルーティングプロトコル等）の優先度を確保します。
*   **利用目的**: CPU リソースの枯渇防止、不正アクセスや設定改ざんの防御、およびネットワーク全体の可用性維持。
*   **どのような場面で利用するか**: 全てのインフラストラクチャルータおよびスイッチの初期設定および境界ルータでの防御強化。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な技術** | CoPP (3.1.a), IP Source Routing (3.1.b), iACLs (3.1.c)。 |
| **ハードニング対象** | 管理アクセス (SSH/HTTPS)、デバイスサービス (SNMP/NTP/Syslog)。 |
| **保護レイヤ** | コントロールプレーン（CPU 宛）およびマネジメントプレーン。 |
| **メリット** | DoS 攻撃耐性の向上、管理トラフィックの機密性確保。 |
| **デメリット** | 設定ミスによる管理ロックアウトや正常なルーティング断のリスク。 |
| **対応機種** | Cisco IOS, IOS-XE, ASA, FTD 等の全インフラ製品。 |
| **設計上の注意点** | フィルタリングの順序（最優先の制御プロトコルを許可）と、段階的なレート制限。 |

---

## 🏗 動作原理

デバイスに到着したパケットは、宛先 IP アドレスに基づいて 3 つのパスに分類されます。

```text
Incoming Packet
   ↓
[ Packet Filter / iACL ] (3.1.c) --- マッチすればドロップ
   ↓
[ Destination Check ]
   ↓
   ├─→ Data Plane (Transit) : 他の宛先へ転送
   │
   └─→ Control Plane (Receive) : デバイス自身の CPU 宛
          ↓
       [ CoPP / CPPr ] (3.1.a) --- レート制限・フィルタリング
          ↓
       [ Management/Routing Process ] (SSH, OSPF, BGP, etc.)
```

---

## ⚙ 動作シーケンス

1.  **境界防御 (iACL)**: インフラストラクチャの入口で、内部の物理 IP 宛の不要なパケット（偵察スキャン等）を遮断します。
2.  **プロトコル制限**: IP Source Routing 等、攻撃に悪用される可能性のある古い機能を無効化します。
3.  **コントロールプレーン分類**: CPU へ送られるパケットを MQC (Modular QoS CLI) を使用してクラス分けします。
4.  **ポリシー適用 (CoPP)**: クラスごとに許可（Transmit）、破棄（Drop）、またはレート制限（Police）を適用します。
5.  **管理アクセス保護**: VTY 行に対する ACL やプロトコル制限（SSH のみ等）により、マネジメントプレーンを最終保護します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **CoPP の設定**: 「特定のプロトコル（ICMP 等）を 100Kbps に制限し、指定の Loopback 範囲以外を Police せよ」といった具体的な数値指定への対応が求められます。
*   **インターフェイス設定の禁止**: 要件に「Do not use any interface-level configuration（インターフェイスレベルの設定を使用しないこと）」とある場合、グローバルな CoPP を使用する必要があります。
*   **iACL の構築**: 全インフラデバイス（コアスイッチやルータ）を保護するために、境界（Edge）ルータの物理 IF に適用する ACL を正しく設計する必要があります。
*   **不要サービスの停止**: `no ip source-route`、`no ip http server`、`no service finger` 等のコマンドを反射的に実行できるようにします。
*   **管理プロトコルの要件**: 「SSH version 2 のみ許可」「タイムアウト 5 分」「ログイン試行 3 回でロック」といった詳細な hardening 設定が問われます。

---

## 🛠 設定方法

### 1. Control Plane Policing (CoPP) の基本構成
```bash
! トラフィックの定義
ip access-list extended ACL-COPP-ICMP
 permit icmp any any
! クラスマップの作成
class-map match-all CLASS-ICMP
 match access-group name ACL-COPP-ICMP
! ポリシーマップの作成
policy-map POLICY-COPP
 class CLASS-ICMP
  police 100000 conform-action transmit exceed-action drop
! コントロールプレーンへの適用
control-plane
 service-policy input POLICY-COPP
```

### 2. IP Source Routing の無効化
※攻撃者がパケットの経路を意図的に指定するのを防ぎます。
```bash
no ip source-route
```

### 3. 管理プレーンのハードニング (SSH)
```bash
hostname R1
ip domain-name ccie.com
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
line vty 0 4
 transport input ssh
 access-class MGMT-HOSTS in
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **CoPP 統計の確認** | <code>show policy-map control-plane</code> |
| **SSH のステータス確認** | <code>show ip ssh</code> |
| **不要なサービスの一覧表示** | <code>show ip sockets</code> / <code>show udp</code> |
| **CPU 負荷の確認** | <code>show processes cpu sorted</code> |
| **ACL ヒットカウントの確認** | <code>show access-lists</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| SSH が接続できない | ACL による拒否、または `transport input` ミス | <code>show run line vty</code> を確認。 |
| ルーティングが不安定 | CoPP で OSPF/BGP を誤ってドロップ | <code>show policy-map control-plane</code> で Drop 数を確認。 |
| デバイスの CPU が高騰 | CoPP が未設定、またはレートが高すぎる | パケットキャプチャを実行し、攻撃ソースを特定。 |
| 設定が反映されない | `control-plane` 下での適用漏れ | <code>service-policy input</code> があるか確認。 |

---

## ⚠ 制限事項

*   **CoPP の方向**: CoPP は `input` 方向（デバイスに入ってくるパケット）にのみ適用され、自分から出すパケットには影響しません。
*   **プラットフォーム依存**: CoPP は ASIC レベルで処理されるスイッチ（Catalyst 等）と、ソフトウェアで処理されるルータで、サポートされる `match` 条件が異なります。
*   **再帰的アクセスの危険**: 管理 ACL で自身の IP を許可し忘れると、二度とリモートアクセスできなくなるため、`reload in 10` 等のタイマーを併用して設定することを推奨します。

---

## 🔄 他技術との関連

*   **Management Plane Protection (MPP)**: 管理トラフィックが許可される物理インターフェイス自体を制限する機能。CoPP と併用されることが多い。
*   **Authentication (2.1)**: デバイスへのログインに ISE (RADIUS/TACACS+) を使用する、AAA 設定との連携。
*   **Monitoring (3.6)**: SNMP や Syslog を保護し、且つ監視を行うための設定。

---

## 🧩 比較表

### CoPP vs iACL (Infrastructure ACL)

| 特徴 | CoPP | iACL |
| :--- | :--- | :--- |
| **適用場所** | デバイスの CPU 直前 (Control-plane) | ネットワーク境界の物理インターフェイス |
| **主な目的** | CPU 負荷の管理・DoS 防止 | 内部 IP 資産への不要トラフィック遮断 |
| **実装方式** | MQC (Class/Policy Map) | Extended ACL |
| **制御単位** | レート制限 (Police) が可能 | Permit / Deny のみ |

---

## 💡 ベストプラクティス

1.  **デフォルト Transmit**: Policy-map の最後に `class-default` で `transmit` を設定し、意図しないドロップを防ぎます。
2.  **重要なプロトコルのホワイトリスト**: BGP, OSPF, SSH は Policing クラスの「前」に、無制限に Transmit するクラスとして定義します。
3.  **SSH タイムアウト**: セッションの放置を防ぐため、`exec-timeout` を短め（例: 5 分）に設定します。
4.  **バナーの活用**: 侵入者に対する法的警告として `banner motd` を設定します。

---

## 📝 ラボ学習・設定サンプル例

### 1. ICMP Policing (100Kbps)
*   **要件**: 自分宛の ICMP を 100Kbps に制限し、それを超える場合は破棄せよ。
*   **設定**: `policy-map` 内で `police 100000` を適用。

### 2. 特定ホストからの SSH 許可
*   **要件**: 10.1.1.0/24 のホストのみが SSH アクセス可能にせよ。
*   **設定**: `line vty` で `access-class` を適用。

### 3. 不要な IP サービスの無効化
*   **要件**: finger, pad, bootp, source-route を全て無効にせよ。
*   **設定**: `no ip source-route`, `no service finger`, `no ip bootp server`, `no ip http server` 等。

### 4. SNMP アクセスリストの適用
*   **要件**: SNMP RO コミュニティ "PUBLIC" を設定し、ホスト 10.1.1.50 からのみ許可せよ。
*   **設定**: `snmp-server community PUBLIC RO 10`。

### 5. iACL によるインフラ保護
*   **要件**: 外部（Outside）から内部ルータの全 IP 宛トラフィックを、確立済みの BGP を除きドロップせよ。

### 6. IPv6 CoPP の実装
*   **要件**: IPv6 ICMP (NDパケット) を保護する CoPP を構成せよ。

### 7. VTY ログインバナー
*   **要件**: "Unauthorized access prohibited" というバナーを表示せよ。

### 8. ログイン試行制限
*   **要件**: 120 秒間に 3 回ログインに失敗した IP を 60 秒間ロックせよ。
*   **設定**: `login block-for 60 attempts 3 within 120`。

### 9. Logging サーバーの保護
*   **要件**: Syslog を 10.1.1.100 へ送信し、ソースを Loopback0 に固定せよ。
*   **設定**: `logging host 10.1.1.100`, `logging source-interface Loopback0`。

### 10. CPPr (Control Plane Protection)
*   **要件**: コントロールプレーンを `host`, `transit`, `cef-exception` サブインターフェイスに分けて管理せよ。

---

## ❓ 想定試験問題

1.  **実装**: 既存のルーティングプロトコル（EIGRP）に影響を与えずに、新規の CoPP ポリシーを導入する際、最も注意すべき設定順序は？
    *   **回答**: EIGRP トラフィックをマッチさせて `transmit`（許可）するクラスを、ICMP などの Policing クラスよりも上位に配置すること。
2.  **トラブルシュート**: CoPP を適用した後、NMS からの SNMP 監視が途切れるようになった。どの `show` コマンドで原因を特定するか？
    *   **回答**: `show policy-map control-plane` で SNMP クラスの `exceed-action` によるドロップが発生していないか確認する。
3.  **Design**: IP Source Routing を無効化すべきセキュリティ上の理由は？
    *   **回答**: 攻撃者がパケットのソースルートオプションを悪用し、正規のセキュリティチェック（Firewall 等）を迂回するパスを強制するのを防ぐため。
4.  **コンフィグ読解**: `line vty 0 4` に `transport input ssh` と設定されている場合、Telnet で接続しようとした時の挙動は？
    *   **回答**: デバイスによって接続が拒否（Connection refused）される。
5.  **実装**: 境界ルータで、内部ネットワークの IP アドレスが送信元になっている外部からのパケット（Spoofing）を遮断するための技術は？
    *   **回答**: **iACL** または **uRPF**（3.3.a）。3.1.c の文脈では iACL でインフラ IP を保護する。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Hardening Guide**
    *   [Cisco IOS-XE Security Configuration Guide: Securing the Control Plane](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_policing/configuration/xe-16/qos-policing-xe-16-book/qos-policing-control-plane.html)
*   **Technical Notes**
    *   [Control Plane Policing Implementation Best Practices](https://www.cisco.com/c/en/us/support/docs/ip/access-lists/43501-copp.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [Securing the Control Plane of Cisco Routers](https://www.ciscolive.com/global.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: CCIE Security において、ハードニングは「加点要素」というより「減点回避」の基本です。Telnet を残したままにするなどの凡ミスは避けましょう。
*   **注意点**: ラボ試験での CoPP 設定時は、必ず `permit` だけでなく、設定した Policing が実際に `show` コマンドでカウントアップされるかを、ルータ自身に `ping` を打つなどして確認してください。
