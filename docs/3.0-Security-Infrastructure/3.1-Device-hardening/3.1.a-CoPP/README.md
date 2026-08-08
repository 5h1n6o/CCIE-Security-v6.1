---
layout: default
title: 3.1.a-CoPP
nav_order: 1
parent: 3.1-Device-hardening
grand_parent: 3.0-Security-Infrastructure
---

# 3.1.a CoPP (Control Plane Policing)

Control Plane Policing (CoPP) は、ルータやスイッチのコントロールプレーン（CPU）を保護するための強力な QoS ベースのメカニズムです。ルータ自体を宛先とする不要なトラフィックや不正なトラフィック（DoS 攻撃、偵察スキャンなど）を制限・フィルタリングすることで、デバイスの安定稼働を維持します。

---

## 📘 概要

*   **機能概要**: Modular QoS CLI (MQC) を使用して、コントロールプレーンへ向かうトラフィックのレート制限（Policing）やドロップを行います。
*   **利用目的**: CPU リソースの枯渇防止、ルーティングプロトコルの安定性確保、管理アクセスの可用性維持。
*   **利用場面**:
    *   インターネット境界ルータでの ICMP 到達制限。
    *   特定の管理端末以外からの SSH/SNMP トラフィックの遮断。
    *   大規模ネットワークにおけるルーティングアップデート（BGP/OSPF 等）の優先保護。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **保護対象** | デバイスの CPU (Control Plane)。 |
| **実装方式** | **MQC (Modular QoS CLI)**: Class-map, Policy-map を使用。 |
| **適用場所** | グローバル設定モードの `control-plane` 下。 |
| **主なアクション** | `transmit` (許可), `drop` (破棄), `police` (レート制限)。 |
| **トラフィック分類** | ACL やプロトコル番号に基づいて識別。 |
| **特徴** | インターフェイス単位ではなく、デバイス全体で一括適用される。 |

---

## 🏗 動作原理

ルータに到着したパケットは、宛先 IP アドレスに基づいて「転送されるもの（Transit）」か「自分宛のもの（Control Plane）」かに分類されます。CoPP は、このコントロールプレーンに向かう論理的なパスに「ゲート」を設置するイメージです。

```text
[ Incoming Packet ]
       ↓
[ Classification ] (Is it for me?)
       ↓ Yes
[ CoPP Policy Check ] (Class-map / Policy-map)
       ↓
[ Action ] --------------------┐
       ↓ Match?                ↓ No Match
[ Police / Drop / Transmit ]  [ Default Action (Permit) ]
       ↓
[ CPU Processing ]
```

---

## ⚙ 動作シーケンス

1.  **パケットの識別**: 設定された ACL に基づき、パケットがどのクラス（例: ICMP, ルーティング, 管理）に属するかを判定。
2.  **ポリシーの参照**: `policy-map` で定義されたアクションを確認。
3.  **レートの測定**: `police` 設定がある場合、現在のトラフィック量がしきい値（bps/pps）を超えていないか計算。
4.  **アクションの実行**:
    *   しきい値内: CPU へ転送。
    *   しきい値超: `drop` または低い優先度で転送。
5.  **統計の更新**: ヒットしたカウンタをインクリメントし、管理者が `show` コマンドで確認可能にする。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **インターフェイス設定の禁止**: 要件で「インターフェイスレベルの設定を使用せずに（Do not use any interface-level configuration）」と指定されることが一般的です。
*   **ホワイトリストの作成**: 特定の管理ネットワーク（例: Loopback0 範囲）からの通信は制限をかけず、それ以外を制限するといった「例外」の設定が問われます。
*   **ルーティングプロトコルの保護**: 誤って BGP や OSPF をドロップするとネイバーが切断されます。必須プロトコルを明示的に `transmit` するクラスを最上位に配置するのが鉄則です。
*   **暗黙の動作**: Policy-map の最後にマッチしないトラフィックは、デフォルトで許可されます（通常の QoS と同様の挙動）。
*   **IPv6 の考慮**: ラボでは IPv4 と IPv6 両方の CoPP を求められる場合があります。
*   **トラブルシュート**: `show policy-map control-plane` で、意図したクラスでドロップ（Droppedパケット）が発生しているか確認できることが合格への鍵です。

---

## 🛠 設定方法

### 1. トラフィックの定義 (ACL)
```bash
ip access-list extended ACL-ICMP-PROTECT
 permit icmp any any
! 特定範囲(150.1.0.0/16)を許可し、それ以外を警察(Police)対象とする例
ip access-list extended ACL-MGMT-ALLOW
 permit ip 150.1.0.0 0.0.255.255 any
```

### 2. クラスマップの作成
```bash
class-map match-all CLASS-ICMP
 match access-group name ACL-ICMP-PROTECT
class-map match-all CLASS-MGMT
 match access-group name ACL-MGMT-ALLOW
```

### 3. ポリシーマップの作成とアクション定義
```bash
policy-map POLICY-COPP
 class CLASS-MGMT
  transmit
 class CLASS-ICMP
  police 100000 conform-action transmit exceed-action drop
```

### 4. コントロールプレーンへの適用
```bash
control-plane
 service-policy input POLICY-COPP
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **CoPP 統計と状態の確認** | <code>show policy-map control-plane [input]</code> |
| **定義済みクラスの確認** | <code>show class-map</code> |
| **適用されている ACL の確認** | <code>show access-lists</code> |
| **CPU 負荷の確認** | <code>show processes cpu sorted</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| SSH が切断される | 管理通信を誤って Policing 対象にした | <code>show policy-map control-plane</code> | 管理用 ACL に許可エントリを追加。 |
| ルートが学習されない | ルーティングプロトコルのドロップ | <code>show ip ospf neighbor</code> | OSPF/EIGRP を許可するクラスを作成。 |
| Ping が通らない | <code>police</code> 値が低すぎる | <code>show policy-map control-plane</code> | <code>police</code> 値を緩和する。 |
| CoPP が効かない | `service-policy` が未適用 | <code>show control-plane</code> | <code>control-plane</code> 下で適用。 |

---

## ⚠ 制限事項

*   **ハードウェアサポート**: 物理スイッチ（Catalyst 等）では、CoPP は ASIC レベルで処理されるため、複雑な ACL や特定のプロトコルマッチングに制限がある場合があります。
*   **出力方向の制限**: CoPP は基本的に `input` 方向（ルータに入ってくる自分宛パケット）にのみ適用可能です。
*   **フラグメントパケット**: 一部のプラットフォームでは、フラグメントされたパケットの処理が特殊なため、正確に Policing できない場合があります。

---

## 🔄 他技術との関連

*   **iACL (Infrastructure ACL)**: ネットワーク全体の境界で自分宛トラフィックを制限するが、CoPP はデバイス自身のエッジで保護を行う。
*   **MPP (Management Plane Protection)**: 管理トラフィック（SSH, HTTP 等）が入ってくる物理インターフェイス自体を制限する機能。
*   **CPPr (Control Plane Protection)**: CoPP の進化版。コントロールプレーンをさらに `host`, `transit`, `cef-exception` の 3 つのサブインターフェイスに分けて細かく制御する。

---

## 🧩 比較表

### CoPP vs iACL

| 特徴 | CoPP | iACL (Infrastructure ACL) |
| :--- | :--- | :--- |
| **適用場所** | 各デバイスの CPU 直前。 | ネットワークの入口（エッジ）。 |
| **主な機能** | レート制限（Policing）。 | 通過フィルタリング（Permit/Deny）。 |
| **保護の深さ** | デバイス個別のリソース保護。 | ネットワークインフラ全体の保護。 |

---

## 💡 ベストプラクティス

1.  **最優先クラスの作成**: ルーティングプロトコル（BGP, OSPF, EIGRP）や重要な管理プロトコル（SSH, SNMP）を一番上のクラスで明示的に許可します。
2.  **デフォルトドロップを避ける**: 最初は `police` でレート制限を行い、完全に遮断する前に統計を確認します。
3.  **IPv6 の忘れず設定**: デュアルスタック環境では IPv6 用の ACL とクラスも必ず定義します。
4.  **Logging**: `police` の `exceed-action` で `drop` する際にログを記録する設定は、攻撃の検知に役立ちます。

---

## 📝 ラボ学習・設定サンプル例

### 1. ICMP レート制限 (100Kbps)
*   **要件**: 自分宛の ICMP を 100Kbps に制限せよ。

### 2. 特定ネットワークのホワイトリスト
*   **要件**: 150.1.0.0/16 からの通信は制限なし、それ以外は Policing せよ。

### 3. SSH 接続試行の制限
*   **要件**: SSH 通信を 50Kbps に制限し、ブルートフォースを防げ。

### 4. OSPF ネイバー保護
*   **要件**: OSPF プロトコル (IP 89) を無制限に許可せよ。

### 5. SNMP 問い合わせの制限
*   **要件**: 特定の NMS サーバー以外からの SNMP (UDP 161) をドロップせよ。

### 6. IPv6 CoPP の実装
*   **要件**: IPv6 の Neighbor Discovery パケットを保護せよ。

### 7. 未使用サービスの完全遮断
*   **要件**: Telnet (TCP 23) をコントロールプレーンで即座にドロップせよ。

### 8. Logging の設定
*   **要件**: 制限値を超えたパケットが発生した際に Syslog を出力せよ。

### 9. BGP の優先転送
*   **要件**: BGP (TCP 179) トラフィックに高い優先度を与えよ。

### 10. 全トラフィックの統計収集
*   **要件**: 最後にマッチしなかったトラフィックの量をカウントせよ（`class-default` の活用）。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `policy-map` 内に `police 8000 conform-action transmit exceed-action drop` がある。パケットが 10Kbps で継続的に送られてきた場合の動作は？
    *   **回答**: 8Kbps までは CPU に転送され、超えた 2Kbps 分はドロップされる。
2.  **トラブルシュート**: CoPP を適用した直後、ルータへの SSH はできるが外部ネットワークへの Telnet ができない。考えられる原因は？
    *   **回答**: CoPP は自分宛のトラフィックのみを制限するため、通常は Transit 通信（通過トラフィック）には影響しない。原因は CoPP ではなく、物理インターフェイスの ACL 等にある可能性が高い。
3.  **Design**: ルーティングプロトコルのパケットを保護するために、Policy-map 内でのクラスの順序はどうあるべきか？
    *   **回答**: 最も優先的に処理されるよう、Policy-map の先頭（上部）に配置すべきである。
4.  **実装**: 既存の `service-policy` を削除せずに、新しいプロトコルを保護対象に追加する手順は？
    *   **回答**: 対応する `access-list` に permit エントリを追加し、既存の `class-map` がその ACL を参照していれば、動的に反映される。
5.  **Design**: CoPP と MPP (Management Plane Protection) を併用する際の注意点は？
    *   **回答**: MPP が特定のインターフェイスからのアクセスを許可していても、CoPP のレート制限が厳しいとパケットがドロップされる可能性があるため、整合性を取る必要がある。

---

## 🔗 参考リソース

*   [Cisco IOS-XE Control Plane Policing Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_policing/configuration/xe-16/qos-policing-xe-16-book/qos-policing-control-plane.html)
*   [Cisco Live: BRKSEC-2003 - Securing the Control Plane](https://www.ciscolive.com/)
*   [White Paper: Control Plane Policing Implementation Best Practices](https://www.cisco.com/c/en/us/support/docs/ip/access-lists/43501-copp.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「CoPP はルータのボディガード」です。すべてのパケットを調べるのではなく、重要なパケットを「通し」、怪しいパケットを「制限」します。
*   **注意点**: ラボ試験で `police` コマンドを入れる際、単位が `bps` か `pps` かを慎重に確認してください。指定と異なると大幅にパケットがドロップされます。
