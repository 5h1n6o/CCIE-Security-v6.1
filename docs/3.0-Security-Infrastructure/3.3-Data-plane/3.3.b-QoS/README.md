---
layout: default
title: 3.3.b-QoS
nav_order: 3
parent: 3.3-Data-plane
grand_parent: 3.0-Security-Infrastructure
---

# 3.3.b QoS (Quality of Service)

Cisco デバイスにおける **QoS (Quality of Service)** は、ネットワークトラフィックの優先順位付け、帯域幅の割り当て、および輻輳制御を行うための技術群です。CCIE Security v6.1 において、QoS は単なるネットワーク品質向上ツールとしてだけでなく、**データプレーン保護（Transit Traffic Control）**および**デバイスハードニング（Congestion Management）**の一環として、DoS 攻撃の緩和や重要な制御トラフィック（BGP/OSPF等）の保護という重要な役割を担います。

---

## 📘 概要

*   **機能概要**: Modular QoS CLI (MQC) を使用してトラフィックを分類（Classification）し、ポリシング（Policing）、シェーピング（Shaping）、または優先制御（Scheduling）を適用します。
*   **利用目的**: ネットワークの帯域枯渇防止、重要通信の低遅延化、および悪意ある大量トラフィック（DoS/DDoS）によるインフラへの影響最小化。
*   **どのような場面で利用するか**:
    *   インターネット境界での ICMP や SYN パケットのレート制限。
    *   WAN 回線における音声/ビデオ会議トラフィックの帯域確保。
    *   CPU 保護を目的としたコントロールプレーンへのパケット制限（CoPP）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要コンポーネント** | Class-map, Policy-map, Service-policy (MQCモデル)。 |
| **分類手法** | ACL, NBAR (Network Based Application Recognition), DSCP/CoS値。 |
| **保護対象** | データプレーン（通過トラフィック）およびコントロールプレーン。 |
| **主なアクション** | **Police**（レート制限・ドロップ）、**Shape**（遅延送出）、**Priority**（優先順位）。 |
| **メリット** | 攻撃下での通信安定性向上、重要プロトコルの維持。 |
| **設計上の注意点** | 複雑すぎる ACL は転送パフォーマンスに影響を及ぼす可能性がある。 |

---

## 🏗 動作原理

QoS は、パケットがデバイスのインターフェイスを通過する際の「仕分け」と「処理」のプロセスで動作します。

```text
Incoming Packet
   ↓
[ Classification ] (クラス分け: 誰が来たか？) -- ACL, NBAR, DSCP
   ↓
[ Marking ] (色付け: ランク付け) -- set dscp, set precedence
   ↓
[ Policing / Shaping ] (交通整理: 制限するか？) -- police, shape
   ↓
[ Queuing / Scheduling ] (並び替え: 誰を先に出すか？) -- Priority, Bandwidth
   ↓
Outgoing Packet
```

---

## ⚙ 動作シーケンス

1.  **トラフィック識別**: 受信したパケットが、`class-map` で定義された ACL や `match protocol` (NBAR) に合致するかを判定します。
2.  **ポリシー参照**: `policy-map` 内の該当クラスに定義されたアクションを確認します。
3.  **メータリング (Policingの場合)**: トークンバケットアルゴリズムに基づき、現在のトラフィックレートが許可範囲（CIR）内か、超過（Exceed）しているかを測定します。
4.  **アクション実行**: 
    *   **Conform**: 許可（Transmit または Marking）。
    *   **Exceed / Violate**: 破棄（Drop）または優先度を下げて転送。
5.  **出力制御**: 輻輳時には、キューイングアルゴリズムに従って重要パケットを優先的に送信バッファへ送ります。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **MQC の基本構文**: `class-map` -> `policy-map` -> `service-policy` の流れは、迷わずに設定できる必要があります。
*   **Transit Traffic Control**: 「外部から内部への ICMP を 100Kbps に制限せよ」といった要件が出題されます。これはデータプレーンインターフェイスへの適用を意味します。
*   **NBAR によるフィルタリング**: 単なる L4 ポートではなく、`match protocol http url "*attack*"` のようにアプリケーション層の識別を QoS と組み合わせて制限する要件に注意してください。
*   **単位の正確性**: `police 100000` (bps) と `police rate 100 pps` (packet per second) を正しく使い分ける必要があります。
*   **CoPP との関係**: コントロールプレーン保護（3.1.a）の実装技術は QoS そのものです。ラボでは「インターフェイスレベルの設定を使わずに CPU を守れ」という指示がある場合、`control-plane` 下で QoS を適用します。
*   **IPv6 への対応**: `ipv6 spd mode aggressive` など、IPv6 特有の輻輳管理設定が問われることがあります。

---

## 🛠 設定方法

### 1. トラフィックの分類 (Class-map)
```bash
! ACLによる分類
ip access-list extended ACL-POLICE-ICMP
 permit icmp any any
! NBARによるアプリケーション分類
class-map match-all CLASS-HTTP-PROTECT
 match protocol http
class-map match-all CLASS-ICMP-LIMIT
 match access-group name ACL-POLICE-ICMP
```

### 2. ポリシーの定義 (Policy-map)
```bash
policy-map PM-TRANSIT-PROTECTION
 class CLASS-ICMP-LIMIT
  police 100000 conform-action transmit exceed-action drop
 class CLASS-HTTP-PROTECT
  bandwidth remaining percent 20
```

### 3. インターフェイスへの適用 (Service-policy)
```bash
interface GigabitEthernet0/1
 service-policy input PM-TRANSIT-PROTECTION
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **QoS ポリシーの統計確認** | <code>show policy-map interface [type]</code> |
| **CoPP 統計の確認** | <code>show policy-map control-plane</code> |
| **NBAR の有効化状態** | <code>show ip nbar protocol-discovery</code> |
| **パケットドロップのデバッグ** | <code>debug qos vlan-policer</code> (プラットフォーム依存) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 正当なパケットがドロップされる | <code>police</code> 値が低すぎる | <code>show policy-map interface</code> | バーストサイズや CIR を緩和する。 |
| QoS ポリシーが機能しない | インターフェイスへの適用漏れ | <code>show run interface</code> | <code>service-policy</code> を追加。 |
| CPU 負荷が高騰している | ソフトウェアベースの QoS 処理 | <code>show processes cpu sorted</code> | ハードウェア処理可能な設定に見直す。 |
| NBAR がトラフィックを検知しない | プロトコル発見が無効 | <code>show ip nbar protocol-discovery</code> | <code>ip nbar protocol-discovery</code> を有効化。 |

---

## ⚠ 制限事項

*   **プラットフォームの制約**: スイッチ（Catalyst 等）では、Ingress と Egress で使用できるコマンドやキューの数にハードウェア上の制限があります。
*   **フラグメントパケット**: 断片化されたパケットは、L4 情報を正確に識別できないため、QoS 分類に失敗する場合があります。
*   **トンネルトラフィック**: GRE や IPsec 内のトラフィックに対して QoS を適用するには、`qos pre-classify` の設定が必要になる場合があります。

---

## 🔄 他技術との関連

*   **3.1.a CoPP**: QoS エンジンを使用してデバイス自身の CPU を保護します。
*   **3.3.a uRPF**: 送信元検証に失敗したパケットを QoS 以前にドロップし、QoS リソースを節約します。
*   **Firepower (FTD) QoS**: FMC/FDM を通じて、より高度なアプリケーション層（AVC）ベースの帯域制限を実施します。
*   **Monitoring (NetFlow)**: QoS ポリシーを設計するためのトラフィック分析に使用されます。

---

## 🧩 比較表

### Policing vs Shaping

| 特徴 | Policing (ポリシング) | Shaping (シェーピング) |
| :--- | :--- | :--- |
| **しきい値超過時の動作** | 即座にドロップまたはマーキング | バッファに溜めて遅延送信 |
| **適用方向** | Ingress / Egress | 主に Egress のみ |
| **適したトラフィック** | ICMP, UDP (DoS対策) | TCP (パケット再送の抑制) |
| **遅延への影響** | 低い（ドロップするため） | 高い（溜めるため） |

---

## 💡 ベストプラクティス

1.  **ルーティングプロトコルの優先**: BGP, OSPF トラフィックは常に `priority` クラスまたは `bandwidth` 指定で保護し、攻撃下でもネイバーが切れないようにします。
2.  **階層型 QoS (HQoS)**: インターフェイス全体の帯域をシェーピングし、その中で個別のポリシングを行う構成を検討します。
3.  **DSCP の活用**: 信頼できる境界においてパケットに `set dscp` でマークを付け、内部ネットワークで効率的に仕分けを行えるようにします。
4.  **Logging**: `police` コマンドでドロップが発生した際にログを出す設定は、攻撃の予兆検知に有効ですが、CPU 負荷に注意が必要です。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な ICMP レート制限 (100Kbps)
*   **問題**: インターフェイス Gi0/1 を通過する ICMP トラフィックを 100Kbps に制限し、超過分を破棄せよ。
*   **設定**: `policy-map PM_ICMP` > `class CLASS-ICMP` > `police 100000 conform-action transmit exceed-action drop`

### 2. ルーティングプロトコルの帯域確保
*   **要件**: BGP (TCP 179) 通信に対し、最低 50Kbps の帯域を保証せよ。

### 3. NBAR を使用した HTTP 制限
*   **要件**: HTTP トラフィックのみを 1Mbps にポリシングせよ。
*   **設定**: `class-map` で `match protocol http` を使用。

### 4. IPv6 優先制御
*   **要件**: IPv6 パケットに対して DSCP EF を設定し、優先キューに入れよ。

### 5. SYN フラッド攻撃の緩和
*   **要件**: TCP SYN パケットのレートを 50pps に制限せよ。

### 6. 特定管理ネットワークの例外
*   **要件**: 10.1.1.0/24 以外からの全トラフィックに対し、QoS ポリシーを適用せよ。

### 7. トンネルトラフィックの QoS 継承
*   **要件**: GRE トンネルを通過する際、内部パケットの DSCP 値を外部ヘッダーにコピーせよ。

### 8. ARP/Broadcast のレート制限
*   **要件**: ブロードキャストパケットによる帯域浪費を防ぐため、物理 IF で制限せよ。

### 9. 拒否ログを伴うポリシング
*   **要件**: ポリシー制限を超えたパケットが発生した際、メッセージを出力せよ。

### 10. CoPP による CPU 保護
*   **要件**: ルータ自身への Telnet 通信を 8Kbps に制限せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `police 8000 conform-action transmit exceed-action drop` が設定されている。1秒間に 16,000 bits 到着した場合、何 bits が転送されるか？
    *   **回答**: 8,000 bits。残りの 8,000 bits はドロップされる。
2.  **Design**: リアルタイム通信（音声）を保護するために QoS ポリシーで使用すべきキーワードは？
    *   **回答**: `priority`（LLQ: Low Latency Queuing を実現するため）。
3.  **トラブルシュート**: QoS ポリシーを適用したが `show policy-map interface` でヒットカウントが増えない。考えられる理由は？
    *   **回答**: `class-map` で指定している ACL の定義ミス、またはトラフィックの進行方向（input/output）の指定誤り。
4.  **実装**: インターフェイスを通過するトラフィック（Transit）ではなく、ルータ自身へ向かうトラフィックを制限する設定箇所は？
    *   **回答**: `control-plane` セクション。
5.  **Design**: 未知のプロトコルを動的に識別して制限するために併用すべき機能は？
    *   **回答**: **NBAR** (Network Based Application Recognition)。

---

## 🔗 参考リソース

*   **Cisco IOS-XE QoS Configuration Guide**
    *   [Configuring Quality of Service](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos/configuration/xe-16/qos-xe-16-book.html)
*   **Technical Notes**
    *   [Modular QoS CLI (MQC) Overview](https://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-policing/10108-25.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [Securing the Data Plane using QoS mechanisms](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: QoS は「ネットワークの蛇口」です。Security 担当者としては、どの蛇口を全開にし（BGP/OSPF）、どの蛇口を絞るべきか（ICMP/Unknown）を常に意識する必要があります。
*   **注意点**: ラボ試験の `police` コマンドにおけるバーストサイズ（Bc/Be）の計算ミスは、通信の極端な劣化を招くため、指定がない場合はデフォルト値の挙動を理解しておくことが重要です。
