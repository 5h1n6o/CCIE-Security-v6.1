---
layout: default
title: 1.10.b Routing protocols security on ASA
nav_order: 2
parent: 1.10-Routing-protocols-security
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.10 Routing protocols security on Cisco ASA

Cisco ASA における**ルーティングプロトコルのセキュリティ**は、ファイアウォールの制御プレーン（Control Plane）を保護し、不正なルート情報の注入やネイバー関係のハイジャックを防止するための不可欠な要素です。ASA は OSPF (v2/v3)、EIGRP、BGP、RIP をサポートしており、それぞれのプロトコルにおいて認証やフィルタリングを実装することで、ネットワーク全体の整合性と可用性を維持します。

---

## 📘 概要

*   **機能概要**: ルーティングアップデートの送受信を制御し、隣接関係を結ぶデバイスの正当性を検証します。
*   **利用目的**: 不正なルータによるトラフィックの引き込み（ブラックホール化や中間者攻撃）の防止、およびルーティングプロセスの過負荷保護。
*   **どのような場面で利用するか**: 境界ルータや内部コアスイッチと ASA 間で動的ルーティングを行う際、信頼されたデバイス間でのみ経路交換を行うために使用します。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な認証方式** | MD5 (OSPF, EIGRP, BGP)、明文認証（推奨されない）。 |
| **ルート制御手法** | Distribute-lists, Prefix-lists, Route-maps。 |
| **インターフェイス制御** | Passive-interface による不要な Hello 送出の停止。 |
| **対応プロトコル** | OSPFv2, OSPFv3, EIGRP, BGP, RIP。 |
| **設計上の注意点** | ASA 自身が宛先となるルーティングパケットは「Identity インターフェイス」への通信として処理される。 |

---

## 🏗 動作原理

ASA を通過するトラフィック（Through-the-box）とは異なり、ルーティングプロトコルのパケットは ASA のインターフェイス IP 自体を宛先として届きます。

```text
[ Neighbor Router ] ── (Routing Update/Hello) ──> [ ASA Interface ]
                                                         │
    [ 1. Input ACL Check ] <--- Must permit protocol (e.g., OSPF/89) to ASA IP
    [ 2. Authentication ]  <--- Check MD5 Hash / Key Match
    [ 3. Routing Engine ]  <--- Update RIB (Routing Information Base)
    [ 4. Filter Check ]    <--- Distribute-list applied?
```

---

## ⚙ 動作シーケンス

1.  **ネイバー検出**: インターフェイスで Hello パケットを受信します。
2.  **正当性検証**: 設定された認証キー（MD5 等）を使用して、パケットの送信元が信頼できるか確認します。
3.  **データベース同期**: 認証成功後、隣接関係（Adjacency）を確立し、LSA やアップデートを交換します。
4.  **ルートフィルタリング**: 受信したアップデートに対し、`distribute-list` に基づいてルーティングテーブルへの格納可否を判断します。
5.  **再配布 (Redistribution)**: 必要に応じて他のプロトコルへ経路を渡す際、`route-map` でタグ付けやメトリックの変更を行います。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **MD5 認証の設定**: OSPF や EIGRP におけるインターフェイスレベルの認証設定は必須スキルです。
*   **Identity ACL の罠**: ASA のインターフェイスに `access-group` を適用している場合、ネイバーからのプロトコル番号（OSPF なら 89, EIGRP なら 88）を ASA のインターフェイス IP 宛に許可する必要があります。
*   **Passive-interface**: セキュリティ要件として「不必要なインターフェイスからのルーティング情報を停止せよ」という指示に対し、`passive-interface default` と必要なポートのみの `no passive-interface` の組み合わせが問われます。
*   **BGP Neighbor Password**: BGP におけるピアごとの MD5 パスワード設定。
*   **Troubleshooting**: `show ospf neighbor` や `show eigrp neighbors` の出力から状態を判断し、`packet-tracer` でパケットのドロップ箇所を特定します。

---

## 🛠 設定方法

### 1. OSPF MD5 認証 (CLI)
```bash
interface GigabitEthernet0/0
 ospf authentication message-digest
 ospf message-digest-key 1 md5 CISCO123
!
router ospf 1
 network 10.1.1.0 255.255.255.0 area 0
```

### 2. EIGRP 認証の設定
```bash
router eigrp 100
 authentication mode md5 100
 authentication key md5 100 CISCO_KEY
```

### 3. BGP 認証の設定
```bash
router bgp 65001
 neighbor 192.168.1.1 remote-as 65002
 neighbor 192.168.1.1 password BGP_PASS
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **OSPFネイバー確認** | <code>show ospf neighbor</code> |
| **EIGRPネイバー確認** | <code>show eigrp neighbors</code> |
| **ルーティングプロトコル詳細** | <code>show router [ospf\|eigrp\|bgp]</code> |
| **認証ミスのデバッグ** | <code>debug ospf adj</code> / <code>debug eigrp packets</code> |
| **パケットドロップの追跡** | <code>packet-tracer input inside rawip 10.1.1.1 89 10.1.1.10 detailed</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| ネイバーが形成されない | インターフェイス ACL で拒否 | <code>show access-list</code> | ASA IP 宛のプロトコルを許可。 |
| 認証失敗 (Auth Failure) | キーまたは ID の不一致 | <code>debug ospf adj</code> | 両端の <code>key-id</code> と文字列を揃える。 |
| 経路が学習されない | Distribute-list の誤設定 | <code>show run router</code> | ACL/Prefix-list の Permit 条件を確認。 |
| BGP が "Active" 状態 | 179/tcp が遮断されている | <code>show conn</code> | ACL で TCP 179 を許可。 |

---

## ⚠ 制限事項

*   **マルチコンテキストモード**: 一部のルーティングプロトコル（OSPFv3 等）の動作や、共有インターフェイス使用時のマルチキャスト処理に制限がある場合があります。
*   **ステートフルフェイルオーバー**: ルーティングプロトコルのネイバー関係は、デフォルトではフェイルオーバー時に同期されず、切り替え後に再確立が必要です。

---

## 🔄 他技術との関連

*   **Access Control**: インターフェイス ACL がルーティングパケットを妨げないように調整が必要です。
*   **High Availability**: フェイルオーバー構成では、両方の ASA ユニットで同一のルーティング設定が同期されます。
*   **VPN**: VTI (Virtual Tunnel Interface) を使用した VPN 構成において、動的ルーティングを安全に走らせるために認証が多用されます。

---

## 🧩 比較表

### ルーティング認証方式の比較

| プロトコル | 認証タイプ | ASA 設定箇所 | 特徴 |
| :--- | :--- | :--- | :--- |
| **OSPF** | MD5 / 明文 | Interface レベル | Area 単位の有効化も可能。 |
| **EIGRP** | MD5 | Router レベル | キー番号の指定が必要。 |
| **BGP** | MD5 | Neighbor レベル | TCP MD5 シグネチャを使用。 |

---

## 💡 ベストプラクティス

1.  **MD5 の使用**: 明文（Plaintext）認証は使用せず、常に MD5 またはそれ以上の強度を使用します。
2.  **Passive-interface の活用**: ルータが存在しないサブネット（ユーザセグメント）には `passive-interface default` を適用し、脆弱な箇所への Hello 送出を止めます。
3.  **ルート集約 (Summarization)**: セキュリティデバイスとしての負荷を減らすため、可能な限りルートを集約して RIB サイズを抑えます。
4.  **ログ監視**: `log-adjacency-changes` を有効にし、ネイバーの切断を Syslog で即座に検知できるようにします。

---

## 📝 ラボ学習・設定サンプル例

### 1. インターフェイス ACL を考慮した OSPF 設定
*   **要件**: ASA の outside に ACL が適用されている環境で、R1 (1.1.1.1) と OSPF ネイバーを組め。
*   **設定**: 
    `access-list OUT extended permit ospf host 1.1.1.1 host 1.1.1.10`
    `access-group OUT in interface outside`

### 2. 特定のルートのみを受信するフィルタリング
*   **要件**: ネイバーから 192.168.0.0/16 以外のルートを拒否せよ。
*   **設定**: 
    `access-list FILTER standard permit 192.168.0.0 255.255.0.0`
    `router ospf 1` -> `distribute-list FILTER in`

### 3. EIGRP 受動インターフェイスの構成
*   **要件**: Inside インターフェイス以外での EIGRP アップデートを停止せよ。
*   **設定**: 
    `router eigrp 10` -> `passive-interface default` -> `no passive-interface inside`

### 4. BGP ピアの MD5 認証
*   **要件**: ピア 2.2.2.2 との間でパスワード "SECURE" を設定せよ。
*   **設定**: `router bgp 65000` -> `neighbor 2.2.2.2 password SECURE`

### 5. 再配布時のセキュリティ (Route-map)
*   **要件**: 外部からの BGP 経路を OSPF に入れる際、特定のタグが付いたもののみ許可せよ。
*   **設定**: `route-map REDIST permit 10` -> `match tag 100` -> `router ospf 1` -> `redistribute bgp 65000 route-map REDIST`

### 6. OSPFv3 (IPv6) の基本的な保護
*   **要件**: IPv6 環境で OSPFv3 を有効化し、インターフェイス Gi0/0 で動作させよ。
*   **設定**: `ipv6 router ospf 1` -> `interface Gi0/0` -> `ipv6 ospf 1 area 0`

### 7. デフォルトルートの配布制限
*   **要件**: ASA から外部へ 0.0.0.0 を広報する際、特定の条件（Route-map）を満たす場合のみ許可せよ。
*   **設定**: `default-information originate route-map CHECK`

### 8. 管理インターフェイス経由のスタティックルート保護
*   **要件**: 管理通信のみを Management インターフェイス経由で行うよう、ルーティングを分離せよ。

### 9. BGP Maximum-prefix による DoS 対策
*   **要件**: ピアから 1000 以上の経路を受信した場合、警告を出してセッションを切断せよ。
*   **設定**: `neighbor 1.1.1.1 maximum-prefix 1000 80`

### 10. packet-tracer による OSPF 疎通確認
*   **コマンド**: `packet-tracer input outside rawip [Peer_IP] 89 [ASA_IP] detailed`

---

## ❓ 想定試験問題

1.  **実装**: ASA の outside インターフェイスに ACL が適用されており、対向ルータとの OSPF ネイバーが Full にならない。ACL に追加すべきエントリを記述せよ。
    *   **回答**: `access-list [NAME] extended permit ospf host [Neighbor_IP] host [ASA_Outside_IP]`
2.  **トラブルシュート**: `show eigrp neighbors` を実行しても何も表示されない。確認すべき 2 つの主要な項目は何か？
    *   **回答**: 1. インターフェイスの `authentication mode` とキーの一致。 2. 該当インターフェイスが `passive-interface` に設定されていないか。
3.  **Design**: セキュリティ要件により、ASA はルーティング情報の「送信」のみを行い、「受信」は一切行わないようにしたい。どの機能を使用すべきか？
    *   **回答**: `distribute-list` を `in` 方向に適用し、すべてを `deny` する ACL を紐付ける。
4.  **コンフィグ読解**: `router ospf 1` の下に `network 0.0.0.0 0.0.0.0 area 0` が設定されている場合のセキュリティリスクを述べよ。
    *   **回答**: 全インターフェイスで OSPF が有効になるため、意図しないセグメント（DMZ 等）でネイバーが形成され、内部情報が漏洩するリスクがある。
5.  **実装**: 内部ネットワークの特定のルート (10.0.0.0/8) が外部 (BGP ピア) へ漏れないように ASA で設定せよ。

---

## 🔗 参考リソース

*   **Cisco Configuration Guides**
    *   [Cisco ASA Series CLI Configuration Guide - Configuring OSPF](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/firewall/asa-94-firewall-config/route-ospf.html)
    *   [Cisco ASA Series CLI Configuration Guide - Configuring EIGRP](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/firewall/asa-94-firewall-config/route-eigrp.html)
*   **Technical Notes**
    *   [Troubleshooting ASA Routing Issues with Packet-Tracer](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/113685-asa-threat-detection-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: ASA におけるルーティングは「自身への通信」であることを忘れないでください。ACL 設定時に、宛先を `any` にするのではなく、ASA の `interface IP` に絞るのがセキュアな設計です。
*   **図解**: 常に `show route` と `show [protocol] neighbor` を交互に確認し、コントロールプレーンが正しく構築されているかを確認する癖をつけましょう。
