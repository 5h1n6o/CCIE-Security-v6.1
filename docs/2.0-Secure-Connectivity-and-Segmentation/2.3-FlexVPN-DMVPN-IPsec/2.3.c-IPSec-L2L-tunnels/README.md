---
layout: default
title: 2.3.c IPsec L2L tunnels
nav_order: 3
parent: 2.3-FlexVPN-DMVPN-IPsec
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.3.c IPsec L2L tunnels

Cisco ASA、Firepower Threat Defense (FTD)、および IOS-XE ルータ間での **IPsec LAN-to-LAN (L2L) トンネル**の実装は、セキュアな拠点間接続の基盤です。CCIE Security ラボ試験では、従来の **Crypto Map** ベースの VPN に加え、ルーティングの柔軟性が高い **Virtual Tunnel Interface (VTI)**、および最新の **IKEv2** プロトコルを使用したプラットフォーム間（Cross-platform）の相互運用性が重要な評価ポイントとなります。

---

## 📘 概要

*   **機能概要**: 
    *   2 つのゲートウェイ間で暗号化された IPsec トンネルを構築し、プライベートネットワーク間の通信を保護します。
    *   **IKE (Internet Key Exchange)** を使用して暗号化キーを動的に生成・更新し、**ESP (Encapsulating Security Payload)** でデータをカプセル化します。
*   **利用目的**: 公衆回線（インターネット）を介したセキュアな拠点間接続。
*   **どのような場面で利用するか**: 
    *   固定 IP 同士のオフィス間接続。
    *   クラウド（AWS/Azure 等）とオンプレミス間のハイブリッド接続。
    *   GRE トンネルを IPsec で保護する場合（GRE over IPsec）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要プロトコル** | IKEv1, **IKEv2** (推奨), ESP, AH (稀)。 |
| **実装方式** | **Crypto Map** (静的/動的) または **VTI** (Interface ベース)。 |
| **認証方式** | Pre-shared Key (PSK), デジタル証明書 (PKI)。 |
| **カプセル化モード** | **Tunnel Mode** (標準) / **Transport Mode** (GRE等と併用)。 |
| **プラットフォーム** | Cisco ASA, FTD (FMC/FDM 管理), IOS-XE ルータ。 |
| **トラフィック選択** | Proxy ID (ACL) による定義、または VTI 経由のルーティング。 |

---

## 🏗 動作原理

IPsec L2L 通信は、コントロールプレーン（IKE）とデータプレーン（IPsec/ESP）の 2 段階で確立されます。

```text
[ Site A Gateway ]                                [ Site B Gateway ]
        |                                                 |
        |---- Phase 1: IKE SA (Management Tunnel) ------->|
        |     (Auth, DH Key Exchange, Encryption)         |
        |                                                 |
        |---- Phase 2: IPsec SA (Data Tunnel) ----------->|
        |     (Proxy IDs, Perfect Forward Secrecy)        |
        |                                                 |
        |<============= Encrypted Data ==================>|
```

---

## ⚙ 動作シーケンス

1.  **興味深いトラフィック (Interesting Traffic) の検知**: ACL またはルーティングにより、IPsec 保護が必要なパケットを識別。
2.  **IKE Phase 1 (ISAKMP/IKEv2 SA)**:
    *   **IKEv1**: Main Mode (6パケット) または Aggressive Mode (3パケット)。
    *   **IKEv2**: IKE_SA_INIT および IKE_AUTH 交換により、1ステップで認証まで完了。
3.  **IKE Phase 2 (Quick Mode / CHILD_SA)**:
    *   実際のデータ転送に使用する暗号化アルゴリズムとキーを決定。
    *   **Proxy ID (送信元/宛先サブネット)** の不一致は、この段階での失敗要因となります。
4.  **トンネル維持**: **DPD (Dead Peer Detection)** や **Keepalives** により、対向の生存を確認。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **ASA ICMP Inspection の罠**: ASA で L2L VPN を設定し、SA が `ACTIVE` になっても `ping` が通らないことがあります。これは ASA がデフォルトで ICMP をインスペクションしないため、戻りのパケットを既存フローとして認識できないことが原因です。`inspect icmp` を **MPF** で有効化する必要があります。
*   **NAT 免除 (No-NAT)**: VPN トラフィックがインターネット向けの通常の NAT ルールにマッチしないよう、ASA では `nat (inside,outside) source static`、FTD では **Identity NAT** ルールを最優先で設定することが必須です。
*   **Proxy ID の厳密な一致**: IKEv1 では特に、両端の ACL（送信元と宛先）が完全に鏡合わせ（Mirror image）である必要があります。
*   **IKEv2 VTI on ASA**: 従来の Crypto Map ではなく、`interface Tunnel` を使用した **VTI** 構成を指定される場合があります。ASA で VTI を使う際は、`tunnel mode ipsec ipv4` の設定が重要です。
*   **FMC による一元管理**: FTD を使用する場合、FMC の `Devices > VPN > Site To Site` ウィザードを使いこなし、デプロイ（Deploy）を忘れないようにします。

---

## 🛠 設定方法

### 1. IOS-XE: IKEv2 VTI (推奨)
```bash
! IKEv2 Proposal & Policy
crypto ikev2 proposal PROP
 encryption aes-gcm-256
 group 19
crypto ikev2 policy POL
 proposal PROP
!
! IKEv2 Profile
crypto ikev2 profile L2L_PROF
 match identity remote address 203.0.113.2 255.255.255.255
 authentication local pre-share key cisco123
 authentication remote pre-share key cisco123
!
! Interface VTI
interface Tunnel0
 ip address 10.255.1.1 255.255.255.252
 tunnel source GigabitEthernet1
 tunnel destination 203.0.113.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC_PROF
```

### 2. ASA: Crypto Map (IKEv1)
```bash
! ACL for Interesting Traffic
access-list VPN_ACL extended permit ip 10.1.1.0 255.255.255.0 10.2.2.0 255.255.255.0
!
! Phase 1 Policy
crypto ikev1 policy 10
 encryption aes-256
 hash sha
 authentication pre-share
 group 2
!
! Tunnel Group (PSK)
tunnel-group 203.0.113.2 type ipsec-l2l
tunnel-group 203.0.113.2 ipsec-attributes
 ikev1 pre-shared-key cisco123
!
! Crypto Map
crypto map MY_MAP 10 match address VPN_ACL
crypto map MY_MAP 10 set peer 203.0.113.2
crypto map MY_MAP 10 set ikev1 transform-set TSET
crypto map MY_MAP interface outside
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **IKEv1 状態確認** | <code>show crypto isakmp sa</code> |
| **IKEv2 状態確認** | <code>show crypto ikev2 sa</code> |
| **IPsec SA (Phase 2) 詳細** | <code>show crypto ipsec sa</code> |
| **VPN セッション概要** | <code>show crypto session [detail]</code> |
| **ASA パケットパス追跡** | <code>packet-tracer input inside icmp [src] 8 0 [dst] detailed</code> |
| **リアルタイムデバッグ** | <code>debug crypto ikev2</code> / <code>debug crypto condition</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| SA が <code>MM_NO_STATE</code> | 物理接続不可、または IKE ポート(500)遮断 | <code>ping</code> でピアへの到達性を確認。 |
| SA が <code>QM_IDLE</code> にならない | Phase 1 ポリシーの不一致 | <code>debug crypto isakmp</code> で Proposal 拒否を確認。 |
| SA は UP するが Ping 不通 | NAT 処理の誤り、または ACL | **NAT Exemption** 設定を確認。ASA では ICMP inspect を確認。 |
| Phase 2 確立失敗 | **Proxy ID (ACL)** の不一致 | 両端の送信元/宛先ネットワーク定義が一致しているか確認。 |
| トンネルがフラッピングする | PFS 設定の片方のみ有効化 | 両端で <code>set pfs</code> の有無を揃える。 |

---

## ⚠ 制限事項

*   **ASA VTI 制限**: ASA の古いバージョンでは VTI がサポートされておらず、Crypto Map が必須となります。
*   **マルチキャスト**: Crypto Map ベースの VPN ではマルチキャスト（OSPF 等）を直接流せないため、GRE トンネルとの併用が必要です。
*   **ハブアンドスポーク**: L2L トンネル単体では、スポーク数が増えると設定が指数関数的に増えるため、DMVPN や FlexVPN が推奨されます。

---

## 🔄 他技術との関連

*   **NAT**: VPN 通信を NAT 対象外にする **Identity NAT (No-NAT)** は実装の必須条件です。
*   **Routing**: VTI を使用する場合、トンネルインターフェイスを Next-hop とするスタティック/動的ルーティングの設定が必要です。
*   **PKI**: PSK の代わりにデジタル証明書を使用する場合、IOS CA 等との連携が必要になります。
*   **Access Control**: トンネルを通過した後のパケットは、受信側のインターフェイス ACL や ACP で許可されている必要があります。

---

## 🧩 比較表

### Crypto Map vs VTI (Virtual Tunnel Interface)

| 特徴 | Crypto Map | VTI (Virtual Tunnel Interface) |
| :--- | :--- | :--- |
| **構成単位** | 物理インターフェイスに適用 | 独立した論理インターフェイス |
| **ルーティング** | ACL によるポリシーベース | ルーティングテーブルに基づく |
| **動的ルーティング** | GRE が別途必要 | ネイティブで OSPF/EIGRP 等に対応 |
| **推奨ユースケース** | レガシー構成、単純な 1:1 接続 | 最新の設計、複雑なルーティングが必要な場合 |

---

## 💡 ベストプラクティス

1.  **IKEv2 への移行**: セキュリティと効率性の観点から、新規構築は IKEv2 を原則とします。
2.  **AES-GCM の採用**: 暗号化と整合性確認を同時に行う AES-GCM は、CPU 効率が良くパフォーマンスに優れます。
3.  **MSS 調整**: IPsec のオーバーヘッドによるフラグメンテーションを防ぐため、`tcp adjust-mss 1360` 程度の設定を推奨します。
4.  **DPD の有効化**: 接続断を迅速に検知し、ハングした SA をクリーンアップするために必須です。

---

## 📝 ラボ学習・設定サンプル例

### 1. ASA と IOS ルータ間の IKEv1 PSK 接続
*   **要件**: 10.1.1.0/24 と 172.16.1.0/24 を Crypto Map で結べ。
*   **ポイント**: ASA の `tunnel-group` 名は対向の **Public IP** にする必要があります。

### 2. IOS 間の IKEv2 VTI 接続
*   **要件**: ルーティングプロトコルを使用せずに、VTI でスタティックルートを回せ。

### 3. FTD (FMC管理) と ASA 間の S2S VPN
*   **要件**: FMC ウィザードを使用してトンネルを構築せよ。
*   **注意**: デプロイメントのステータスを確認すること。

### 4. 証明書 (PKI) を使用した L2L 認証
*   **要件**: IOS CA で発行した証明書を使用せよ。

### 5. GRE over IPsec (Transport Mode)
*   **要件**: GRE トンネルを IPsec で保護し、MTU 負荷を最小限にせよ。

### 6. NAT-Traversal (NAT-T) の検証
*   **要件**: 片方が NAT の背後にある環境で UDP 4500 を使用して接続せよ。

### 7. Proxy ID の集約設定
*   **課題**: 複数のサブネットを 1 つの SA で保護する（ACL の集約）。

### 8. ASA での VPN トラフィック・インスペクション
*   **要件**: トンネル経由の HTTP 通信に正規表現フィルタを適用せよ。

### 9. Perfect Forward Secrecy (PFS) の実装
*   **要件**: Phase 2 の鍵交換で DH Group 14 を強制せよ。

### 10. debug ログによる不一致箇所の特定
*   **課題**: `debug crypto isakmp` を読み解き、ハッシュアルゴリズムのミスマッチを特定せよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: ASA とルータ間の L2L トンネルで、ルータ側から `ping` は通るが、ASA 側からの `ping` が失敗する。ASA の `show crypto ipsec sa` ではエンカプ数が増えている。考えられる原因は？
    *   **回答**: ルータ側で **NAT 免除 (No-NAT)** が設定されておらず、戻りのパケットがインターネット向けに変換されている、または ASA の **ICMP Inspection** が無効。
2.  **Design**: IPsec トンネル上で OSPF を動作させたい場合、ASA で採用すべき実装方式は？
    *   **回答**: VTI (Virtual Tunnel Interface)、または Crypto Map と GRE over IPsec の併用。
3.  **コンフィグ読解**: `crypto map` に `set pfs group14` が設定されている場合、対向デバイスで必要な設定は？
    *   **回答**: 対向の IPsec ポリシー（Phase 2）でも同様に PFS Group 14 を有効化する必要がある。
4.  **実装**: FMC で複数の FTD 間に S2S VPN を設定する際、トポロジタイプとして選択可能なものは？
    *   **回答**: Point-to-Point, Hub and Spoke, Full Mesh。
5.  **Design**: IKEv1 Main Mode と Aggressive Mode のセキュリティ上の主な違いは？
    *   **回答**: Aggressive Mode はパケット数が少ないが、デバイスの ID (Hostname 等) が平文で送信されるため、メインモードより脆弱。

---

## 🔗 参考リソース

*   **Cisco ASA Series VPN Configuration Guide**
    *   [Site-to-Site VPNs](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/vpn/s2s/asa-94-s2s-vpn-config.html)
*   **Cisco Secure Firewall Management Center Administration Guide**
    *   [Firepower Threat Defense Site-to-Site VPNs](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/site_to_site_vpns_for_firepower_threat_defense.html)
*   **Cisco Live (BRKSEC-3052)**
    *   [IPsec VPN Site-to-Site Architectures](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「SA は上がっているのに通信できない」は CCIE の定番です。**NAT** と **ASA Inspection** を常に疑ってください。
*   **図解**: 常にパケットがカプセル化される前後のヘッダー（Inner IP / Outer IP）を意識して、ACL を設計しましょう。
*   **注意点**: ラボ試験の FTD では、FMC で設定を「保存」しただけでは不十分です。必ず **Deploy** を実行し、完了を待つ必要があります。
