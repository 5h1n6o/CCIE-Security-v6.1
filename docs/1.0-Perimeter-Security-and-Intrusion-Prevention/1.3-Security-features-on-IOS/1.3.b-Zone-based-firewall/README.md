---
layout: default
title: 1.3.b-Zone-based-firewall
nav_order: 1
parent: 1.3-Security-features-on-IOS
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.3.b Zone-based firewall

Cisco IOS/IOS XEにおける**Zone-based Policy Firewall (ZBFW)**は、インターフェイスベースの旧来のファイアウォール（CBAC）に代わり、インターフェイスを「ゾーン」としてグループ化し、ゾーン間のトラフィックに対して一貫したセキュリティポリシーを適用する次世代のステートフルファイアウォール機能です。

---

## 📘 概要

*   **機能概要**: インターフェイスを論理的な「ゾーン（Security Zone）」に割り当て、ゾーンのペア（Zone-Pair）に対してCisco Common Classification Policy Language (C3PL)を用いたポリシーを適用します。
*   **利用目的**: ネットワークトポロジの変更に左右されない、柔軟でスケーラブルなセキュリティ管理を実現します。
*   **どのような場面で利用するか**: 内部ネットワーク（Inside）、外部ネットワーク（Outside）、公開サーバ領域（DMZ）といった論理的境界を定義し、境界を跨ぐ通信を厳格に制御する場合に利用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **基本コンポーネント** | Zone, Zone-pair, Class-map, Policy-map, Parameter-map |
| **デフォルトの動作** | ゾーン間の通信は**デフォルトで全てドロップ**。同一ゾーン内インターフェイス間は許可（設定変更可）。 |
| **アクション** | **Inspect**（ステートフル検査）、**Pass**（一方向許可）、**Drop**（破棄） |
| **Self Zone** | ルータ自身（コントロールプレーン）を発信地・宛先とする特別なゾーン |
| **対応OS** | Cisco IOS, IOS XE（ISR, ASR, CSR1000v/Catalyst 8000Vなど） |
| **利点** | ルーティングプロトコルや管理通信を「Self Zone」で個別に制御可能。NBAR2との統合。 |
| **設計上の注意点** | インターフェイスは1つのゾーンにしか所属できない。VPN仮想インターフェイスの扱いに注意。 |

---

## 🏗 動作原理

ZBFWはインターフェイスがどのゾーンに属しているかを基準にポリシーを決定します。

```text
[Inside Zone]           [Outside Zone]
  (Gi1, Gi2)              (Gi0, Tun0)
      |                       |
      +------ Zone-Pair ------+
      |    (Inside-to-Out)    |
      |          |            |
      |      Policy-Map       |
      |     (C3PL Logic)      |
      ↓                       ↓
[Inspect/Pass/Drop]     [Inspect/Pass/Drop]
```

*   **同一ゾーン内トラフィック**: デフォルトで通過。
*   **ゾーン間トラフィック**: 対応するZone-pairとPolicyが定義されていない限り破棄。
*   **非ゾーンインターフェイス**: ゾーン所属インターフェイスとの通信は不可。

---

## ⚙ 動作シーケンス

1.  **インターフェイス着信**: パケットがルータのインターフェイスに到着。
2.  **ゾーン識別**: イングレスおよびエグレスインターフェイスが属するゾーンを特定。
3.  **ゾーンペア検索**: 送信元ゾーンと宛先ゾーンの組み合わせ（Zone-pair）が存在するか確認。
4.  **ポリシー適用**: Zone-pairに紐付けられた `policy-map type inspect` を参照。
5.  **クラスマッチング**: `class-map type inspect` の条件（プロトコル、ACL、NBAR等）に合致するか評価。
6.  **アクション実行**:
    *   **Inspect**: ステートテーブルにエントリを作成し、戻りのトラフィックを自動許可。
    *   **Pass**: パケットを通過させるが、戻りのパケットは別途ポリシーが必要（ステートレス）。
    *   **Drop**: パケットを破棄し、必要に応じてログを出力。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **C3PL（Common Classification Policy Language）の正確な記述**: `type inspect` キーワードを忘れると、通常のQoS用クラスマップになり、ZBFWとして動作しません。
*   **Self Zoneの制御**: ルータ宛のBGP/OSPF/SSHなどを許可し忘れると、隣接関係が切れるため、試験の初期段階での構成が重要です。

### ラボ試験で設定させられそうな内容
*   **NBAR2との連携**: 特定のL7プロトコル（例：BitTorrent）をブロックする。
*   **パラメータマップのカスタマイズ**: セッションタイムアウトや最大セッション数の制限。
*   **VPNインターフェイス（Virtual-Template/Tunnel）のゾーン割り当て**: FlexVPNやDMVPN環境での実装。

### よくある設定ミス
*   **クラスマップの不一致**: `match-any` と `match-all` の選択ミス。
*   **戻り通信の考慮漏れ**: `Pass` アクションを使用した場合、対向のZone-pairでも許可設定が必要。
*   **インターフェイス割り当て忘れ**: `zone-member security` コマンドの投入漏れ。

---

## 🛠 設定方法

### 1. ゾーンの作成とインターフェイスの割り当て
```bash
zone security INSIDE
zone security OUTSIDE
!
interface GigabitEthernet1
 zone-member security INSIDE
interface GigabitEthernet0
 zone-member security OUTSIDE
```

### 2. クラスマップの定義（トラフィックの分類）
```bash
# HTTPとHTTPSをステートフル検査対象にする
class-map type inspect match-any WEB_TRAFFIC
 match protocol http
 match protocol https
 match protocol icmp
```

### 3. ポリシーマップの定義（アクションの定義）
```bash
policy-map type inspect IN_TO_OUT_POLICY
 class type inspect WEB_TRAFFIC
  inspect
 class class-default
  drop log
```

### 4. ゾーンペアの作成とポリシー適用
```bash
zone-pair security IN_TO_OUT source INSIDE destination OUTSIDE
 service-policy type inspect IN_TO_OUT_POLICY
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **構成済みゾーンの確認** | <code>show zone security</code> |
| **ゾーンペアと適用ポリシーの表示** | <code>show zone-pair security</code> |
| **ランタイム統計とマッチング確認** | <code>show policy-map type inspect zone-pair [name]</code> |
| **現在確立されているセッションの確認** | <code>show policy-map type inspect sessions</code> |
| **クラスマップの定義確認** | <code>show class-map type inspect</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 通信が一方通行、または戻りが届かない | アクションが <code>Pass</code> になっている | <code>show policy-map type inspect zone-pair</code> | <code>Inspect</code> アクションに変更するか戻りポリシーを作成。 |
| ZBFW設定後、OSPF/BGPがDownした | Self-zone 宛のトラフィックが遮断された | <code>show logging</code> | Self-zone を含む Zone-pair で制御プロトコルを許可。 |
| クラスマップにヒットしない | <code>type inspect</code> が付いていない | <code>show run class-map</code> | <code>class-map type inspect</code> として再作成。 |
| 特定のL7アプリが遮断できない | NBAR2エンジンの未稼働、またはシグネチャ不一致 | <code>show ip nbar protocol-discovery</code> | <code>ip nbar protocol-discovery</code> の有効化を確認。 |

---

## ⚠ 制限事項

*   **CBACとの共存不可**: 同一インターフェイス上で旧式のCBAC（ip inspect）とZBFWは同時に使用できません。
*   **パフォーマンス**: 大量のステートフルインスペクションはソフトウェアベースのルータにおいてスループット低下を招く可能性があります。
*   **マルチキャスト**: ZBFW経由のマルチキャストトラフィック処理には追加の設定や制限がある場合があります。

---

## 🔄 他技術との関連

*   **NAT**: 一般的にZBFWのチェックはNAT処理の「後（アウトバウンド時）」または「前（インバウンド時）」に行われるため、ACLやクラスマップで指定するIPアドレスがリアルIPかマップIPかを正確に把握する必要があります。
*   **VPN**: DMVPNの `Tunnel` インターフェイスやFlexVPNの `Virtual-Template` にゾーンを適用することで、暗号化トラフィックの終端後にフィルタリングをかけることが可能です。
*   **NBAR2**: アプリケーションアウェアなフィルタリングを実現するための必須コンポーネントです。

---

## 🧩 比較表

### ZBFW vs CBAC (Context-Based Access Control)

| 機能 | ZBFW (推奨) | CBAC (旧式) |
| :--- | :--- | :--- |
| **基本単位** | ゾーン（論理グループ） | インターフェイス |
| **ポリシー記述** | C3PL (Class/Policy-map) | インターフェイス直下の ACL/Inspect |
| **管理の容易性** | 高い（多数のIFを一括管理） | 低い（IFごとに設定が必要） |
| **Self Zone制御** | 容易に可能 | 困難 |
| **アクション** | Inspect, Pass, Drop | Inspectのみ（ACLと併用） |

---

## 💡 ベストプラクティス

1.  **明示的なドロップとログ**: `class class-default` には必ず `drop log` を設定し、拒否されたトラフィックを可視化します。
2.  **Self Zoneの保護**: 管理アクセス（SSH, SNMP）とルーティング（OSPF, EIGRP, BGP）のみに絞った Self Zone ポリシーを最初に作成します。
3.  **Parameter-mapの活用**: DoS攻撃対策として、ハーフオープンセッション数や接続時間の最大値を定義した `parameter-map` を作成し、`inspect` アクションに適用します。
4.  **ネーミングコンベンション**: `ZP_INSIDE_TO_OUTSIDE` のように、役割がひと目でわかるゾーンペア名を付けます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的なインターネットアクセス許可
*   **要件**: 内部（INSIDE）から外部（OUTSIDE）へのHTTP, HTTPS, ICMPを許可し、他はログを出して遮断。
*   **設定**:
```bash
class-map type inspect match-any CM-WEB
 match protocol http
 match protocol https
 match protocol icmp

policy-map type inspect PM-INSIDE-OUTSIDE
 class type inspect CM-WEB
  inspect
 class class-default
  drop log

zone security INSIDE
zone security OUTSIDE
zone-pair security ZP-IN-OUT source INSIDE destination OUTSIDE
 service-policy type inspect PM-INSIDE-OUTSIDE

interface GigabitEthernet1
 zone-member security INSIDE
interface GigabitEthernet0
 zone-member security OUTSIDE
```
*   **検証**: `show policy-map type inspect zone-pair ZP-IN-OUT` でヒット数を確認。

### 2. ルータへの管理アクセス（Self Zone）の保護
*   **要件**: INSIDEの特定の管理端末(10.1.1.100)からのみSSHを許可し、他は拒否。
*   **設定**:
```bash
access-list 10 permit 10.1.1.100

class-map type inspect match-all CM-ADMIN-SSH
 match access-group 10
 match protocol ssh

policy-map type inspect PM-IN-SELF
 class type inspect CM-ADMIN-SSH
  pass
 class class-default
  drop

zone-pair security ZP-IN-SELF source INSIDE destination self
 service-policy type inspect PM-IN-SELF
```
*   **検証**: 管理端末からSSHを実行し、`show policy-map type inspect sessions` を確認。

### 3. 管理アクセスのSelf Zone制御
*   **要件**: 内部10.1.1.0/24からルータへのSSHアクセスのみを許可。
*   **設定**: 
    ```bash
    access-list 100 permit tcp 10.1.1.0 0.0.0.255 any eq 22
    class-map type inspect match-all CM_ADMIN
     match access-group 100
    zone-pair security ZP_IN_SELF source INSIDE destination self
     service-policy type inspect PM_ADMIN
    ```
    
### 4. NBAR2によるP2Pアプリケーションの遮断
*   **要件**: BitTorrentトラフィックを識別して遮断せよ。
*   **設定**:
```bash
class-map type inspect match-any CM-P2P
 match protocol bittorrent

policy-map type inspect PM-INSIDE-OUTSIDE
 class type inspect CM-P2P
  drop log
 class type inspect CM-WEB
  inspect
```
*   **検証**: `show ip nbar protocol-discovery` で検知を確認後、ポリシーのドロップカウンタを確認。

### 5. 同一ゾーン内インターフェイス間の通信遮断
*   **要件**: 同一ゾーン「INSIDE」に属する Gi1 と Gi2 間の通信を明示的に遮断せよ。
*   **設定**:
```bash
# デフォルトでは同一ゾーン間は許可されるが、ペアを作ることで制御可能
policy-map type inspect PM-DENY-ALL
 class class-default
  drop log

zone-pair security ZP-INTRA-INSIDE source INSIDE destination INSIDE
 service-policy type inspect PM-DENY-ALL
```

### 6. セッションタイムアウトのカスタマイズ
*   **要件**: TCPセッションのアイドルタイムアウトを1200秒に延長せよ。
*   **設定**:
```bash
parameter-map type inspect PARAM-TIMEOUT
 tcp idle-time 1200

policy-map type inspect PM-INSIDE-OUTSIDE
 class type inspect CM-WEB
  inspect PARAM-TIMEOUT
```
*   **検証**: `show parameter-map type inspect PARAM-TIMEOUT` で値を確認。

### 7. IPv6 トラフィックのインスペクション
*   **要件**: IPv6環境においてもステートフルインスペクションを有効にせよ。
*   **設定**:
```bash
# IOS XEではIPv4/IPv6のクラスマップは共通で動作可能
class-map type inspect match-any CM-V6
 match protocol tcp
 match protocol udp

policy-map type inspect PM-V6
 class type inspect CM-V6
  inspect
```

### 8. マルチVRF環境でのZBFW
*   **要件**: VRF「GUEST」に属するゾーン間のポリシーを構成せよ。
*   **設定**:
```bash
zone security GUEST-INSIDE
zone security GUEST-OUTSIDE

interface Gi1.10
 vrf forwarding GUEST
 zone-member security GUEST-INSIDE
interface Gi0.10
 vrf forwarding GUEST
 zone-member security GUEST-OUTSIDE

zone-pair security ZP-GUEST source GUEST-INSIDE destination GUEST-OUTSIDE
 service-policy type inspect PM-GUEST
```

### 9. ICMP Errorパケットの透過（Traceroute対応）
*   **要件**: 外部からのICMP Time-Exceededなどを許可し、Tracerouteを正常化せよ。
*   **設定**:
```bash
# parameter-map で icmp error インスペクションを有効化
parameter-map type inspect PARAM-ICMP
 icmp error

policy-map type inspect PM-INSIDE-OUTSIDE
 class type inspect CM-WEB
  inspect PARAM-ICMP
```

### 10. 監査ログ（Audit Trail）の有効化
*   **要件**: 許可された全セッションの開始・終了をSyslogに記録せよ。
*   **設定**:
```bash
parameter-map type inspect PARAM-AUDIT
 audit-trail on

policy-map type inspect PM-INSIDE-OUTSIDE
 class type inspect CM-WEB
  inspect PARAM-AUDIT
```

### 11. 公開サーバ（DMZ）向けのポリシー
*   **要件**: OUTSIDEからDMZのWebサーバ(192.168.20.10)へのHTTPアクセスのみを許可。
*   **設定**:
```bash
access-list 100 permit ip any host 192.168.20.10

class-map type inspect match-all CM-DMZ-HTTP
 match access-group 100
 match protocol http

policy-map type inspect PM-OUT-DMZ
 class type inspect CM-DMZ-HTTP
  inspect
 class class-default
  drop

zone-pair security ZP-OUT-DMZ source OUTSIDE destination DMZ
 service-policy type inspect PM-OUT-DMZ4
```

---

# ❓ 想定試験問題

1.  **実装**: 3つのゾーン（Inside, Outside, DMZ）があり、InsideからDMZへは全通信許可、InsideからOutsideへはInspect、DMZからOutsideへはDNS/HTTPのみ許可する設定を完了せよ。
2.  **トラブルシュート**: ZBFW設定後、ルータがSyslogをサーバに送信しなくなった。原因として考えられるZone-pairは何か？
    *   **回答**: `self` から `OUTSIDE` (またはサーバのあるゾーン) への Zone-pair における許可設定漏れ。
3.  **コンフィグ読解**: `pass` アクションが設定されたポリシーマップが提示されている。この時、外部からの戻りパケットがドロップされる理由を説明せよ。
    *   **回答**: `pass` はステートレスな許可であり、戻りのトラフィックに対するステート情報が作成されないため。
4.  **Design**: ルータ自身から発生するルーティングプロトコルのアップデートを保護しつつ、外部からの不要なプローブを遮断するための最適なゾーン設計を述べよ。
5.  **動作シーケンス**: ZBFWにおいて、パケットが `Drop` されるデフォルトのタイミング（どのルールにもマッチしない場合）を特定せよ。

---

# 🔗 参考リソース

*   **Configuration Guide**
    *   [Zone-Based Policy Firewall Configuration Guide (Cisco.com)](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_zbf/configuration/xe-16/sec-data-zbf-xe-16-book.html)
*   **Cisco Live**
    *   BRKSEC-2007: Design and Deployment of IOS Zone-Based Firewall
*   **Technical Notes**
    *   [Zone-Based Policy Firewall Design and Application Guide](https://www.cisco.com/c/en/us/support/docs/security/ios-firewall/98628-zone-design-guide.html)
*   **CVD**
    *   Cisco Secure Campus Design Guide

---

## 📝 **補足（Notes）**

*   **学習メモ**: ZBFWの設定は非常に定型的ですが、`type inspect` のキーワードを1つ忘れるだけで機能しなくなるため、ラボ試験ではタイピングの正確性が求められます。
*   **図解**: 常に `source` と `destination` の方向性を意識して Zone-pair を描く習慣をつけましょう。
*   **注意点**: `Self zone` はデフォルトでは「全て許可（Pass）」の状態から開始されますが、一度 `self` を宛先または送信元とする Zone-pair を作成すると、その方向のトラフィックは「デフォルトドロップ」に変わるというクリティカルな挙動があります。慎重に導入してください。 


