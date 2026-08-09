---
layout: default
title: 3.3-Data-plane
nav_order: 3
parent: 3.0-Security-Infrastructure
---

# 3.3 Data plane protection techniques

Ciscoデバイスにおける**データプレーン保護**は、ルータやスイッチを通過するトラフィック（Transit Traffic）の整合性を確保し、ネットワーク全体を攻撃から守るための重要な防衛レイヤです。本セクションでは、送信元IP偽装を防止する **uRPF**、輻輳管理を行う **QoS**、および大規模なDDoS攻撃を緩和する **RTBH** に焦点を当てて解説します。

---

## 📘 概要

*   **機能概要**: ネットワークを通過するデータパケットに対して、送信元の妥当性確認、帯域制限、および不正なパケットの強制破棄を実施する技術群です。
*   **利用目的**: IPスプーフィング（なりすまし）の防止、Dos/DDoS攻撃の緩和、重要な通信の品質確保。
*   **利用場面**: インターネット境界での送信元検証（uRPF）、回線輻輳時のプロトコル保護（QoS）、攻撃対象トラフィックのエッジでの遮断（RTBH）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **uRPF (3.3.a)** | FIB（CEFテーブル）を参照し、送信元IPが正当な経路から来ているか検証する。 |
| **QoS (3.3.b)** | ポリシングやシェーピングにより、データプレーンの流量を制御し輻輳を管理する。 |
| **RTBH (3.3.c)** | BGPを利用して、特定の宛先（または送信元）への通信をネットワーク境界で破棄（Black Hole）する。 |
| **メリット** | スプーフィング攻撃の根絶、DoS耐性の向上、帯域の有効活用。 |
| **設計上の注意点** | uRPFは非対称ルーティング環境で通信断を招くリスクがあるため、モード選定が重要。 |

---

## 🏗 動作原理

### uRPF (Unicast Reverse Path Forwarding)
パケットを受信した際、その「送信元IPアドレス」をキーとしてCEFテーブルを検索します。
*   **Strictモード**: 送信元への戻りルートが、パケットを受信したインターフェイスと一致する場合のみ許可。
*   **Looseモード**: 送信元へのルートがFIBに存在すれば（インターフェイスは問わない）許可。

### RTBH (Remotely Triggered Black Hole)
1.  **Triggerルータ**で、攻撃対象のIPに対するスタティックルート（Next-hopを未使用のIPに設定）をBGPで広報。
2.  **エッジルータ**は、その未使用IPを `Null0` 宛として学習し、該当パケットを即座に破棄する。

---

## ⚙ 動作シーケンス

1.  **パケット受信**: 物理インターフェイスにデータパケットが到着。
2.  **uRPFチェック (3.3.a)**: CEFテーブルを参照し、送信元IPの妥当性を確認。不整合ならドロップ。
3.  **QoSポリシング (3.3.b)**: 設定されたレート制限に基づき、パケットを許可(Conform)か超過(Exceed)か判定。
4.  **ルーティング/RTBH (3.3.c)**: 宛先IPを検索。RTBH発動中であれば、Next-hopが `Null0` を指しているため破棄される。
5.  **転送**: すべてのチェックを通過したパケットのみが出力インターフェイスへ送出される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **uRPFモードの正確な選択**: 「非対称ルーティングが想定される境界でスプーフィング対策をせよ」という要件なら、Strictではなく **Looseモード** を選択する必要があります。
*   **uRPF allow-default**: デフォルトルートのみを持つ拠点ルータでは、`allow-default` オプションを付けないとuRPFでパケットが落ちる問題が頻出します。
*   **QoS Policing**: 「ICMPを100Kbpsに制限せよ」といった具体的な数値指定への対応。
*   **RTBHの設定フロー**: BGPのCommunity値を使用してエッジルータのNext-hopを `Null0` へ書き換えるルートマップ作成スキルが問われます。
*   **トラブルシュート**: `show ip interface` でuRPFによるドロップ数を確認できることが重要です。

---

## 🛠 設定方法

### 1. uRPF (Strictモード) の設定
```bash
interface GigabitEthernet0/1
 ip verify unicast source reachable-via rx
 ! 送信元へのルートが受信IFと一致することを要求
```

### 2. QoS Policing (ICMP制限)
```bash
ip access-list extended ACL-ICMP
 permit icmp any any
!
class-map CLASS-ICMP
 match access-group name ACL-ICMP
!
policy-map PROTECT-DATA-PLANE
 class CLASS-ICMP
  police 100000 conform-action transmit exceed-action drop
!
interface GigabitEthernet0/1
 service-policy input PROTECT-DATA-PLANE
```

### 3. RTBH (Destination-based)
Triggerルータでの設定：
```bash
ip route 192.168.1.1 255.255.255.255 Null0 tag 666
!
router bgp 65001
 redistribute static route-map RTBH-MAP
!
route-map RTBH-MAP permit 10
 match tag 666
 set ip next-hop 192.0.2.1  ! Blackhole用の仮想IP
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **uRPFの状態とドロップ数** | <code>show ip interface [type] \| include verify</code> |
| **CEFエントリの確認** | <code>show ip cef [prefix]</code> |
| **QoS統計の確認** | <code>show policy-map interface [type]</code> |
| **BGPによるRTBHルートの確認** | <code>show ip bgp [prefix]</code> |
| **スプーフィング攻撃のデバッグ** | <code>debug ip packet [ACL]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 正常な通信がuRPFでドロップされる | 非対称ルーティング | <code>reachable-via any</code> (Loose) に変更。 |
| uRPFが効かない | CEFが無効 | <code>ip cef</code> がグローバルで有効か確認。 |
| RTBHで全通信が止まった | BGP広報の間違い | Triggerルータの <code>ip route</code> 宛先マスクを確認。 |
| QoSで期待通り制限されない | 単位の誤り (bps vs pps) | <code>police</code> コマンドの構文を再確認。 |

---

## ⚠ 制限事項

*   **uRPFの依存関係**: CEF (Cisco Express Forwarding) が有効である必要があります。
*   **マルチキャスト**: uRPFはユニキャストトラフィックのみを対象とします。
*   **RTBHの収束**: BGPの伝搬時間に依存するため、即時反映にはBGPタイマーの調整が必要です。

---

## 🔄 他技術との関連

*   **BGP (2.4.c)**: RTBHのコントロールプレーンとして利用。
*   **CEF**: uRPFのフォワーディングベース。CEFがないとuRPFは動作しません。
*   **ACL (3.1.c)**: iACLによるインフラ保護と組み合わせて多層防御を実現。
*   **NetFlow (3.6.a)**: 攻撃トラフィックを特定し、RTBHを発動させるためのトリガーとして使用。

---

## 🧩 比較表

### uRPF Strict vs Loose

| 特徴 | Strict Mode | Loose Mode |
| :--- | :--- | :--- |
| **検証条件** | ルート存在 ＋ **インターフェイス一致** | ルート存在のみ（IF不問） |
| **用途** | キャンパス、単一ISP接続エッジ | 非対称パス、マルチホーム接続 |
| **セキュリティ強度** | 高 | 中 |
| **allow-default** | 利用可能（推奨） | 利用可能 |

---

## 💡 ベストプラクティス

1.  **BCP 38 (RFC 2827) の遵守**: ネットワーク境界で自組織のIP以外を送信元とするパケットをフィルタリングします。
2.  **uRPF Logging**: ドロップが発生した際にログを記録する設定（`logging` オプション）を追加します。
3.  **S/RTBHの活用**: 送信元ベースのRTBH (S/RTBH) を使用して、攻撃者IPからの通信を全ルートで遮断します。
4.  **QoSによる重要パケット保護**: 攻撃下でもルーティングプロトコルを維持するため、制御通信を優先キューに入れます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 送信元IPスプーフィングの完全遮断
*   **要件**: 外部IF Gi0/0 でStrictモードのuRPFを設定し、且つデフォルトルートでの到達も許可せよ。
*   **設定**: `ip verify unicast source reachable-via rx allow-default`。

### 2. RFC 1918 フィルタリング
*   **要件**: 外部からの送信元がプライベートIPであるパケットをドロップし、ログを記録せよ。
*   **設定**: `deny ip 10.0.0.0 0.255.255.255 any log` など。

### 3. RTBH Triggerの設定
*   **要件**: 攻撃対象 203.0.113.1 を Blackhole 化せよ。
*   **設定**: ルートマップで Next-hop を Null0 へ誘導する設定。

### 4. ICMP 帯域制限
*   **要件**: インターフェイスを通過する ICMP を 25pps に制限せよ。

### 5. IPv6 uRPFの実装
*   **要件**: IPv6 トラフィックに対しても送信元検証を有効化せよ。
*   **設定**: `ipv6 verify unicast source reachable-via rx`。

### 6. uRPF ドロップの統計確認
*   **課題**: `show ip interface` を実行し、"verification drops" のカウンタが増えることを確認せよ。

### 7. QoS による TCP SYN フラッドの抑制
*   **要件**: SYN パケットのレートを制限し、サービスダウンを防止せよ。

### 8. BGP Community を用いた RTBH
*   **要件**: 特定の Community 値が付与されたルートを自動で Null0 宛にせよ。

### 9. 非対称環境での uRPF
*   **課題**: Loose モードを設定し、異なる IF からの戻りルートを持つパケットが許可されることを確認せよ。

### 10. QoS NBAR によるアプリケーション識別
*   **要件**: NBAR を使用して特定の攻撃的なプロトコルをデータプレーンで制限せよ。

---

## ❓ 想定試験問題

1.  **Design**: シングルホームのエンタープライズ拠点で、最もセキュリティ強度が高い uRPF 設定モードは？
    *   **回答**: **Strict モード** (`reachable-via rx`)。パスが単一のため不整合が起きにくく、偽装パケットを確実に排除できる。
2.  **トラブルシュート**: uRPF を設定したが、インターネットからのパケットがすべてドロップされる。ルータはデフォルトルートのみを保持している。原因は？
    *   **回答**: `allow-default` オプションが設定されていないため、FIB 上の具体的なルートと一致せずドロップされている。
3.  **コンフィグ読解**: `police 100000 18750 37500` の設定値において、各数値が表す意味は？
    *   **回答**: CIR (100Kbps)、Normal Burst (18750 bytes)、Extended Burst (37500 bytes)。
4.  **実装**: RTBH を実装する際、エッジルータが攻撃トラフィックを Null0 に捨てるように仕向けるための BGP 属性の変更内容は？
    *   **回答**: 学習した攻撃ルートの **Next-hop** を、エッジルータ上で Null0 にスタティック設定された IP アドレスへ書き換える。
5.  **Design**: uRPF が CEF を必要とする理由は？
    *   **回答**: uRPF はパケットごとの送信元逆引きを高速に行う必要があり、その情報を保持する **FIB (Forwarding Information Base)** が CEF によって生成されるため。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Security Configuration Guide**
    *   [Configuring Unicast Reverse Path Forwarding](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_urpf/configuration/xe-16/sec-data-urpf-xe-16-book.html)
*   **White Paper**
    *   [Remotely Triggered Black Hole Filtering](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/213541-remotely-triggered-black-hole-filtering.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [Securing the Data Plane](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: uRPF は「パケットが本来来るべき扉から入ってきたか」をチェックする防犯カメラのような存在です。
*   **図解**: パケットがルータに入り、L2 解除後に L3 チェック（uRPF）を受け、QoS ポリサーを通過し、最後に RIB/FIB に基づいて転送または Null0 へ破棄される流れを意識してください。
*   **注意点**: ラボ試験では、uRPF と iACL (3.1.c) の設定が重複するように見えることがありますが、uRPF は「動的なルートベースのチェック」、iACL は「静的なリストベースのチェック」であることを明確に区別して実装してください。
