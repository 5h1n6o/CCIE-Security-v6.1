---
layout: default
title: 1.7.a-DoS-DDoS
nav_order: 1
parent: 1.7-Attack-detection
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.7 Detect and mitigate common types of attacks
# 1.7.a DoS/DDoS

DoS（Denial of Service：サービス拒否攻撃）および DDoS（Distributed Denial of Service：分散型サービス拒否攻撃）は、ネットワークリソース、CPU、またはメモリを枯渇させることで、正規のユーザーへのサービス提供を妨害する攻撃です。CCIE Security v6.1 ラボ試験では、インフラストラクチャデバイス（IOS-XE、ASA、FTD）の機能を駆使して、これらの攻撃を「検知（Detect）」し「緩和（Mitigate）」する能力が問われます,。

---

## 📘 概要

*   **機能概要**: 閾値に基づいたレートリミット、パケットの正当性検証（uRPF）、コントロールプレーンの保護（CoPP）、および攻撃トラフィックのブラックホール化（RTBH）などを用いて、システム全体の可用性を維持します。
*   **利用目的**: ネットワーク機器のハングアップ防止、正規トラフィックの帯域確保、サービスの継続性維持。
*   **利用場面**: インターネット境界ルータでのなりすましパケット遮断、データセンターFWでの大量セッション接続制限,。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | 単一（DoS）または複数（DDoS）の攻撃元からのリソース枯渇。 |
| **用途** | 可用性 (Availability) の確保。 |
| **メリット** | 攻撃下でも管理アクセスや基幹サービスの通信を維持。 |
| **デメリット** | 設定が厳しすぎると正規通信もドロップする（誤検知）。 |
| **対応機種** | Cisco Catalyst, ISR/ASR (IOS-XE), ASA, Firepower (FTD)。 |
| **制限事項** | 帯域幅全体を埋め尽くすような超大規模DDoSはISP側での対応が必要。 |
| **設計上の注意点** | セキュリティとパフォーマンスのトレードオフ（検査負荷）。 |

---

## 🏗 動作原理

DoS/DDoS 緩和は、パケット処理の各階層で行われます。

```text
[ Attacker / Botnet ] ─── (Flood: SYN, ICMP, etc.) ───→ [ Router/Firewall ]
                                                              │
    [ 1. Data Plane Protection ] ← uRPF / RTBH / ACL         │
                                                              │ (Passed)
    [ 2. Security Engine ] ← Connection Limits / Inspection  │
                                                              │ (Passed)
    [ 3. Control Plane Protection ] ← CoPP / CPPr / Management Protection
                                                              │
[ CPU / Critical Services ] ←── (Clean Traffic Only) ─────────┘
```

---

## ⚙ 動作シーケンス

1.  **トラフィック着信**: インターフェイスでパケットを受信。
2.  **L3検証 (uRPF)**: 送信元IPがルーティングテーブル（FIB）と一致するか確認し、なりすましを防止。
3.  **統計監視 (Threat Detection)**: ASA/FTDが接続試行率やスキャン行動を分析し、異常を検知。
4.  **ポリシー適用 (CoPP)**: CPU宛の通信（BGP, SSH, ICMP等）を定義された帯域に制限。
5.  **不完全セッションの破棄**: TCP SYN Floodに対し、タイムアウトを早めるか不完全な接続数を制限。
6.  **破棄とロギング**: 閾値を超えたパケットを破棄し、管理者に通知を送信。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **CoPP (Control-Plane Policing)**: 特定のプロトコル（ICMP, SNMP等）を特定帯域に制限する設定。ソースIPによる例外処理が重要です。
*   **uRPF (Unicast Reverse Path Forwarding)**: Strictモード（経路とIFが一致）と Looseモード（経路が存在すればOK）の使い分け。
*   **RTBH (Remotely Triggered Black Hole)**: BGPを利用して攻撃対象（Destination）または攻撃元（Source）を Null0 へリダイレクトする手法,。
*   **ASA Threat Detection**: `basic-threat`（基本検知）と `scanning-threat`（スキャン攻撃検知）の有効化、および shun による自動遮断。
*   **Connection Limits**: 1ホストあたりの最大接続数や、ハーフオープン（Embryonic）接続数の制限。

---

## 🛠 設定方法

### 1. IOS-XE: CoPP による ICMP 制限
```bash
! 例: ループバック以外からの ICMP を 100Kbps に制限
access-list 100 deny ip 150.1.0.0 0.0.255.255 any
access-list 100 permit icmp any any
!
class-map CM-ICMP-POLICE
 match access-group 100
!
policy-map PM-COPP
 class CM-ICMP-POLICE
  police 100000 conform-action transmit exceed-action drop
!
control-plane
 service-policy input PM-COPP
```

### 2. ASA: Threat Detection
```bash
! 基本的な脅威検知の有効化
threat-detection basic-threat
! 特定のレート設定（スキャン攻撃等）
threat-detection rate scanning-threat threshold 500
! 統計の表示
threat-detection statistics access-list
```

### 3. ZBFW: TCP SYN Flood 対策
```bash
! 不完全なコネクション数を制限
parameter-map type inspect default
 max-incomplete low 80
 max-incomplete high 100
 tcp synwait-time 30
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **CoPPの統計確認** | <code>show policy-map control-plane</code> |
| **ASA脅威検知の統計** | <code>show threat-detection statistics</code> |
| **uRPFドロップ数確認** | <code>show ip interface [INT] \| include verification</code> |
| **不完全接続の確認** | <code>show parameter-map type inspect default</code> |
| **IPv6 CPU保護確認** | <code>show ipv6 spd</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| 正当なBGPネイバーが切れる | CoPPの設定が厳しすぎる | 制御用ACLにBGPピアのIPを許可（Exempt）に含める。 |
| uRPFでパケットが落ちる | 非対称ルーティングが発生 | モードを <code>Strict</code> から <code>Loose</code> に変更。 |
| ASAで特定の通信が止まる | Threat Detectionの誤検知 | <code>show threat-detection scanning-threat</code> で遮断IPを確認し解除。 |
| CPU負荷が下がらない | パケットがプロセススイッチされている | <code>ip cef</code> が有効であることを確認し、ハードウェア処理を優先させる。 |

---

## ⚠ 制限事項

*   **ハードウェア制限**: CoPPやuRPFのパケット処理性能は、プラットフォーム（ASIC/FPGA）の性能に依存します。
*   **非対称ルーティング**: uRPF Strictモードは戻り経路が異なる環境（マルチホーム等）では使用できません。
*   **IPv6**: IPv6環境では `ipv6 spd`（Selective Packet Discard）など、IPv4とは異なるコマンド体系が必要な場合があります。

---

## 🔄 他技術との関連

*   **BGP**: RTBH（ブラックホール）の伝搬に利用されます。
*   **QoS**: 通信全体のレートリミット（MPP: Management Plane Protection等）として機能します,。
*   **Snort (IPS)**: FTDにおいて、シグネチャベースのプロトコルアノマリ検知（HTTP Flood等）を担当します,。
*   **Stealthwatch (Secure Network Analytics)**: フローベースで大規模なDDoSを検知し、ルータに自動的に緩和指示を出します,。

---

## 🧩 比較表

### uRPF Strict vs Loose vs VRF-Mode

| モード | チェック内容 | 推奨用途 |
| :--- | :--- | :--- |
| **Strict** | 到着IFがベストパスであること | エッジ（シングルホーム） |
| **Loose** | FIBに経路が存在すること | 非対称ルーティング環境、RTBH連携 |
| **VRF-mode** | 特定のVRF内に経路が存在すること | MPLS VPN, マルチテナント環境 |

---

## 💡 ベストプラクティス

1.  **階層型防御**: ルータ（uRPF/CoPP）→ FW（Conn Limit）→ IPS（Anomaly Detection）の順でフィルタリングを行い、CPU負荷を分散します。
2.  **ベースラインの把握**: 平常時の `show threat-detection statistics` を記録しておき、閾値を適切に設定します。
3.  **ブラックホール（Null0）の活用**: 明らかな攻撃は BGP-RTBH で ISP 境界に近い箇所で破棄します。
4.  **IPv6保護**: `ipv6 spd mode aggressive` を使用して、高負荷時に不正なIPv6パケットを積極的に破棄します。

---

## 📝 ラボ学習・設定サンプル例

### 1. ICMP Flood の緩和 (IOS-XE)
*   **問題**: インターフェイス G1 からの ICMP を 200kbps に制限し、超過分を破棄せよ。
*   **設定**: `policy-map` で `police` を適用し、インターフェイスの `service-policy input` に適用。

### 2. Unicast RPF Strict の実装
*   **要件**: なりすましを防止するため、全インターフェイスで経路一致確認を行え。
*   **設定**: `interface G1` -> `ip verify unicast source reachable-via rx allow-default`。

### 3. ASA Embryonic Connection 制限
*   **要件**: 特定のサーバー `10.1.1.50` へのハーフオープン接続を最大 100 に制限せよ。
*   **設定**: `policy-map` -> `class` -> `set connection embryonic-conn-max 100`。

### 4. Destination-Based RTBH (BGP)
*   **要件**: 攻撃対象 `203.0.113.1` への通信をネットワーク全体で遮断せよ。
*   **設定**: Triggerルータで `ip route 203.0.113.1 255.255.255.255 Null0 tag 666` を設定し、BGPでタグ付きルートを広報。

### 5. IPv6 SPD による CPU 保護
*   **要件**: 不正な IPv6 パケットをアグレッシブに破棄するモードを有効にせよ。
*   **設定**: `ipv6 spd mode aggressive`。

### 6. Control Plane Protection (CPPr)
*   **要件**: CPU の `host` ポート宛の通信のみを特定のクラスで保護せよ。
*   **設定**: `control-plane host` 下での `service-policy` 適用。

### 7. ASA Scanning Threat の有効化
*   **課題**: ネットワークスキャンを行っているホストを自動検知せよ。
*   **設定**: `threat-detection rate scanning-threat threshold 1000`。

### 8. NetFlow による DoS 可視化
*   **要件**: 大量のフローを Stealthwatch へ送信して分析せよ。
*   **設定**: `flow monitor` の構成とインターフェイス適用,。

### 9. Management Plane Protection (MPP)
*   **要件**: 特定のインターフェイス（G0/0）からのみ SSH 管理を許可せよ。
*   **設定**: `control-plane` -> `management-interface GigabitEthernet0/0 allow ssh`。

### 10. ASA shun による緊急遮断
*   **要件**: 攻撃元 IP `1.1.1.1` をコマンドで即座に遮断せよ。
*   **コマンド**: `shun 1.1.1.1`。

---

## ❓ 想定試験問題

1.  **実装**: ルータの管理用IPへの ICMP 通信が CPU を圧迫している。特定の管理者ネットワーク `10.0.0.0/24` からの通信は無制限とし、それ以外は 128kbps に制限する CoPP を構成しなさい。
2.  **トラブルシュート**: `ip verify unicast source reachable-via rx` を設定後、正規の BGP ピアリングがダウンした。原因を特定し、解決しなさい。
    *   **回答**: ネイバーがデフォルトルート経由でしか到達できない場合、`allow-default` オプションが必要。
3.  **Design**: サービスプロバイダーとの境界で、送信元偽装パケット（Spoofing）を最小の CPU 負荷で防ぐために最適な機能は何か？
    *   **回答**: Unicast RPF (uRPF)。
4.  **実装**: ASAにおいて、外部からの大量の接続試行によりメモリが不足している。接続が完了しないセッション（Embryonic）の最大数を 500 に制限する設定を完了しなさい。
5.  **コンフィグ読解**: 提示された `show policy-map control-plane` の出力で、`exceed-action drop` のカウンタが増えている。これが示す状態を説明しなさい。
    *   **回答**: 設定された帯域制限（CoPP）を超過した DoS 攻撃または過剰トラフィックが実際に破棄されている状態。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [Control Plane Policing (CoPP)](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_plcly/configuration/xe-16/qos-plcly-xe-16-book/qos-plcly-ctrl-pln-plcing.html)
*   **ASA Configuration Guide**
    *   [Threat Detection on the Cisco ASA](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/113685-asa-threat-detection-00.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [Control Plane Protection Best Practices](https://www.ciscolive.com/on-demand/on-demand-library.html)
*   **Cisco White Paper**
    *   [Remotely Triggered Black Hole Filtering](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/212356-understand-firepower-deployment-modes.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: DoS 対策は「ルータ単体」で完結させようとせず、ASA/FTD の機能や、ときには BGP を使ったネットワーク全体の連携（RTBH）を意識して学習してください。
*   **図解**: パケットが CPU に届く前に「ふるい」にかけるイメージが CoPP です。
*   **注意点**: ラボ試験では `service-policy` を `control-plane` に適用するのを忘れがちですので、必ず最後に `show control-plane` で確認してください。
---
