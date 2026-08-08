---
layout: default
title: 2.3-FlexVPN-DMVPN-IPsec
nav_order: 3
parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.3 FlexVPN, DMVPN, and IPsec L2L tunnels

Ciscoネットワークにおけるサイト間（Site-to-Site）接続は、レガシーなIPsec L2L、スケーラブルなDMVPN、そして最新の統合フレームワークであるFlexVPNへと進化してきました。CCIE Securityラボ試験では、IOS-XEルータ、ASA、FTDの各プラットフォーム間での相互運用性と、複雑なハブアンドスポーク構成における暗号化、ルーティング、および次世代機能の統合が問われます。

---

## 📘 概要

*   **機能概要**: 
    *   **IPsec L2L**: 2拠点間を固定的に結ぶ暗号化トンネル。
    *   **DMVPN**: mGREとNHRPを使用し、動的なスポーク間通信を可能にするハブアンドスポークVPN。
    *   **FlexVPN**: IKEv2を核とし、リモートアクセス、S2S、DMVPNの機能を単一のコマンド体系で統合した次世代VPN。
*   **利用目的**: 公衆回線経由でのセキュアな拠点間接続、大規模なメッシュ/スター型ネットワークの構築。
*   **場面**: 企業の支店接続、クラウド移行におけるハイブリッド接続、マルチベンダー環境での標準プロトコル（IKEv2）利用。

---

## 🔑 要点

| 項目 | IPsec L2L (Crypto Map) | DMVPN (Phase 1-3) | FlexVPN |
| :--- | :--- | :--- | :--- |
| **トランスポート** | GRE or Raw IPsec | mGRE (Multipoint GRE) | GRE or VTI (Static/Dynamic) |
| **主要プロトコル** | IKEv1 / IKEv2 | NHRP + IKEv1/v2 | **IKEv2 (必須)** |
| **スケーラビリティ** | 低い（1対1の設定が必要） | 高い（動的な追加が可能） | 非常に高い（統合ポリシー管理） |
| **スポーク間通信** | ハブ経由のみ | 直接通信可能 (Phase 2/3) | 直接通信可能 (Shortcut) |
| **ルーティング** | スタティック/動的 (GRE必須) | 動的 (EIGRP/OSPF/BGP) | 動的 + IKEv2による経路注入 |
| **ASA/FTDサポート** | フルサポート | サポートなし | 限定的（IKEv2 VTIベース） |

---

## 🏗 動作原理

### DMVPNの核：NHRP (Next Hop Resolution Protocol)
スポークがハブ（NHS）に対して自分の物理IPとトンネルIPのマッピングを登録し、ハブがデータベースを維持することで、宛先スポークの物理IPを動的に解決します。

### FlexVPNの核：IKEv2 Smart Defaults
IKEv2プロファイルを使用して認証（証明書/PSK）と認可（AAA）を処理し、仮想インターフェイス（Virtual-Template）を使用してセッションごとにトンネルを生成します。

```text
[ Spoke/Initiator ] --- (IKEv2 Auth / NHRP Registration) ---> [ Hub/Responder ]
        ↓                                                        ↓
[ TSET / Profile ] <------- (Negotiation & Policy Push) -------> [ IKEv2 Profile ]
        ↓                                                        ↓
[ VTI Establishment ] <---- (Encrypted Data Traffic) ----> [ Dynamic Virtual-Access ]
```

---

## ⚙ 動作シーケンス

1.  **IKE SA 交渉 (Phase 1)**: 暗号アルゴリズムの決定と、証明書またはPSKによるデバイス認証。
2.  **認可/属性プッシュ**: (FlexVPNのみ) AAAサーバーからIPアドレスやルーティング情報を取得。
3.  **IPsec SA 確立 (Phase 2)**: 実際のデータ転送用暗号キーの生成。
4.  **トンネルインターフェイス起動**: VTIまたはmGREインターフェイスがUP状態になる。
5.  **NHRP 登録/解決**: (DMVPN/FlexVPN) スポークの物理IPをハブへ通知、または他スポークへのショートカットを要求。
6.  **ルーティング同期**: 暗号トンネルを介して動的ルーティングプロトコル（OSPF/EIGRP等）を走行させる。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **DMVPN Phase 3 の実装**: `ip nhrp redirect`（ハブ）と `ip nhrp shortcut`（スポーク）の設定、および戻りの経路におけるCEFの重要性。
*   **OSPF Network Type**: DMVPN環境での `ip ospf network point-to-multipoint`（DR選出回避）の使い分け。
*   **IKEv2 VTI on ASA/FTD**: ASA/FTDにおいて、従来のCrypto Mapではなく、VTIインターフェイスを使用したS2S VPNの設定要件。
*   **FlexVPN Any-to-Any**: IKEv2プロファイルでの `match identity` の正確な記述と、`virtual-template` のクローン設定。
*   **NAT免除 (No-NAT)**: VPNトラフィックがNAT処理されないよう、ASA/FTDでのアイデンティティNAT設定。
*   **Troubleshooting**: IKEv2の状態確認（`show crypto ikev2 sa`）と、NHRP解決プロセスの追跡（`debug nhrp condition`）。

---

## 🛠 設定方法

### 1. IOS-XE: DMVPN Phase 3 (Hub)
```bash
interface Tunnel0
 ip address 172.16.1.1 255.255.255.0
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 ip nhrp network-id 1
 ip nhrp map multicast dynamic
 ip nhrp redirect ! Phase 3 必須
 tunnel protection ipsec profile DMVPN-PROF
```

### 2. IOS-XE: FlexVPN Hub (IKEv2)
```bash
crypto ikev2 profile HUB_PROF
 match identity remote any
 authentication local rsa-sig
 authentication remote rsa-sig
 virtual-template 1
!
interface Virtual-Template1 type tunnel
 ip unnumbered Loopback0
 tunnel protection ipsec profile IPSEC_PROF
```

### 3. ASA: IKEv2 L2L (VTI)
```bash
interface Tunnel1
 nameif vpn-tunnel
 ip address 10.255.1.1 255.255.255.252
 tunnel source outside
 tunnel destination 203.0.113.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC_PROF
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **IKEv2 SAの状態確認** | <code>show crypto ikev2 sa</code> |
| **IPsec SAの統計と詳細** | <code>show crypto ipsec sa</code> |
| **NHRPデータベースの確認** | <code>show ip nhrp [dynamic\|static]</code> |
| **VPNセッションのサマリー** | <code>show crypto session detail</code> |
| **パケットドロップの診断** | <code>packet-tracer input ...</code> (ASA/FTD) |
| **IKEv2 リアルタイムデバッグ** | <code>debug crypto ikev2 error</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| SAが確立しない | IKEプロポーザル/プロファイル不一致 | <code>show cry ikev2 sa</code> が見えない場合、ポリシー設定を再確認。 |
| DMVPNスポーク間通信不可 | NHRP 解決失敗 | ハブで <code>redirect</code>, スポークで <code>shortcut</code> が設定されているか確認。 |
| 経路は学習するが疎通なし | CEF不一致、またはACL | <code>show ip cef [dest]</code> で Tunnel インターフェイスを指しているか確認。 |
| トンネルが頻繁に切れる | DPD (Dead Peer Detection) ミス | 両端のキープアライブ設定時間を揃える。 |

---

## ⚠ 制限事項

*   **ASA/FTD の DMVPN 不可**: ASA および FTD は DMVPN のハブやスポークとして動作できません（IOSルータ限定）。
*   **FlexVPN IKEv1 混合制限**: FlexVPN フレームワーク内で IKEv1 を混在させることは推奨されず、機能が制限されます。
*   **MTU/MSS**: トンネルのカプセル化オーバーヘッドにより、1400バイト程度へのMTU調整と `tcp-mss-adjust` が不可欠です。

---

## 🔄 他技術との関連

*   **Routing**: OSPF、EIGRP、BGP のネイバーをトンネル経由で確立します。DMVPN では Split-horizon の無効化が必要です。
*   **PKI**: IKEv2 におけるスケーラブルな認証（証明書）の基盤となります。
*   **QoS**: トンネルインターフェイス（VTI）または物理インターフェイスでのシェーピング。
*   **ZBF**: トンネルインターフェイスを適切なセキュリティゾーンに所属させる必要があります。

---

## 🧩 比較表

### Crypto Map vs VTI (Virtual Tunnel Interface)

| 特徴 | Crypto Map (Legacy) | VTI (Recommended) |
| :--- | :--- | :--- |
| **インターフェイス** | 物理ポートに適用 | 独立した論理ポート |
| **マルチキャスト** | GRE が必須 | ネイティブサポート |
| **ルーティング** | ACL でトラフィックを指定 | ルーティングテーブルに従う |
| **設定の柔軟性** | 低い（複雑なACL） | 高い（Zone適用が容易） |

---

## 💡 ベストプラクティス

1.  **IKEv2 への移行**: セキュリティと柔軟性の観点から、新規構築はすべて IKEv2 を使用します。
2.  **ハブの冗長化 (Dual Hub)**: DMVPN 構成では 2 つ以上のハブを異なるクラウド（Dual Cloud）で構成し、可用性を高めます。
3.  **証明書認証**: PSK 管理を避け、IOS CA や外部 CA を利用した証明書認証を採用します。
4.  **NHRP 登録通知**: スポーク側で `ip nhrp registration no-unique` を使用し、スポーク再起動時の迅速な再登録を可能にします。

---

## 📝 ラボ学習・設定サンプル例

### 1. IOS-IOS IKEv2 L2L (Static VTI)
*   **要件**: R1 と R2 間を VTI を使用して AES-256 で接続せよ。
*   **設定**: `crypto ikev2 profile` -> `interface Tunnel` -> `tunnel protection ipsec profile`。

### 2. ASA-ASA IKEv2 VTI
*   **要件**: ASA1 と ASA2 間をトンネルインターフェイス経由で OSPF を回せ。

### 3. DMVPN Phase 1 (Static)
*   **要件**: ハブ R1 にスポーク R2 が固定登録される基本構成を組め。

### 4. DMVPN Phase 3 (EIGRP)
*   **要件**: スポーク R4, R5 間でハブを介さず直接通信させよ。
*   **ポイント**: ハブに `ip nhrp redirect`、スポークに `ip nhrp shortcut`。

### 5. FlexVPN Hub-and-Spoke (IKEv2)
*   **要件**: ハブ側を `Virtual-Template` で構成し、動的にスポークを収容せよ。

### 6. OSPF over DMVPN (Point-to-Multipoint)
*   **要件**: ハブでの DR/BDR 選出を無効にし、スポークの Next-hop を維持せよ。

### 7. IPsec over IPv6 輸送
*   **要件**: アンダーレイが IPv6 の環境で IPv4 トンネルを構築せよ。

### 8. FTD サイト間 VPN (FMC管理)
*   **要件**: FMC を使用して 2 台の FTD 間に S2S トンネルをデプロイせよ。

### 9. IKEv2 PSK ローテーション
*   **要件**: `ikev2 keyring` で特定のサブネットごとに異なる PSK を適用せよ。

### 10. VPN 障害時のデバッグ
*   **課題**: `packet-tracer` と `debug` を併用し、Phase 2 の ACL 不一致を特定せよ。

---

## ❓ 想定試験問題

1.  **Design**: DMVPN Phase 2 と Phase 3 の最大の違いは、スポーク間のルーティング処理においてどこにあるか？
    *   **回答**: Phase 2 はハブが Next-hop を書き換えない（スポークのIPを維持する）ことに依存するが、Phase 3 は `NHRP Redirect` により物理層で直接通信をトリガーし、CEF テーブルを書き換える。
2.  **トラブルシュート**: IPsec SA は UP しているが、GRE トンネルを介した OSPF ネイバーが Established にならない。考えられる原因は？
    *   **回答**: MTU サイズの超過によるパケットドロップ、またはインターフェイス ACL で OSPF (89) や GRE (47) が許可されていない。
3.  **コンフィグ読解**: IKEv2 プロファイル内の `match identity remote any` コマンドが持つ役割は？
    *   **回答**: Responder（ハブ）側で、接続を試みるすべてのピアを受け入れ、後の認証プロセス（証明書等）に委ねるための設定。
4.  **Design**: 大規模なハブアンドスポーク構成において、FlexVPN が DMVPN よりも優れている点は？
    *   **回答**: IKEv2 標準に基づいているためマルチベンダー対応が容易であり、AAA による動的なポリシー（SGT、ACL）のプッシュがネイティブで統合されている。
5.  **トラブルシュート**: DMVPN で `show ip nhrp` を実行したが、スポークの物理 IP が表示されない。どのフェーズを確認すべきか？
    *   **回答**: NHRP Registration が行われる前の IPsec SA 確立状態（Phase 1/2）をまず確認する。

---

## 🔗 参考リソース

*   **Cisco Configuration Guides**
    *   [Dynamic Multipoint VPN (DMVPN) Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_conn_dmvpn/configuration/xe-16/sec-conn-dmvpn-xe-16-book.html)
    *   [FlexVPN Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_conn_ikeev2/configuration/xe-16/sec-flex-vpn-xe-16-book.html)
*   **Cisco Live (BRKSEC-3052)**
    *   [DMVPN/FlexVPN Advanced Architectures](https://www.ciscolive.com/)
*   **Technical Notes**
    *   [Troubleshooting DMVPN Phase 3](https://www.cisco.com/c/en/us/support/docs/security-vpn/dynamic-multipoint-vpn-dmvpn/119022-technote-dmvpn-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「DMVPN は NHRP、FlexVPN は IKEv2」とシンプルに整理しましょう。
*   **注意点**: ラボ試験では **CEF (Cisco Express Forwarding)** が無効になっていると DMVPN Phase 3 のショートカットが機能しません。`ip cef` が有効であることを常に確認してください。
