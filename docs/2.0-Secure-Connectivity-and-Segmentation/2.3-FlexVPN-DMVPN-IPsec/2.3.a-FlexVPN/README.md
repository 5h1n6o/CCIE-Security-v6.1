---
layout: default
title: 2.3.a FlexVPN
nav_order: 1
parent: 2.3-FlexVPN-DMVPN-IPsec
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.3.a FlexVPN

FlexVPN は、IKEv2 プロトコルに基づいた Cisco 独自の統合 VPN ソリューションです。リモートアクセス (RA)、サイト間 (S2S)、ハブアンドスポーク、およびマネージド VPN サービスを単一のコマンド体系で統合し、高度なスケーラビリティと柔軟性を提供します。

---

## 📘 概要

*   **機能概要**: IKEv2 (Internet Key Exchange version 2) を核とした次世代 VPN フレームワークです。スマートデフォルト、共通の CLI、および柔軟な認証・認可モデルを使用します。
*   **利用目的**: 複雑な VPN 構成の簡素化、マルチベンダー間の相互運用性の向上、および AAA サーバーを利用した動的なポリシー適用を目的とします。
*   **利用場面**: 大規模な拠点間接続、モバイルユーザーの収容、クラウド接続、および従来の DMVPN からのアップグレードパスとして利用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要プロトコル** | **IKEv2** (UDP 500/4500), IPsec。 |
| **インターフェイス** | **Virtual-Template** (動的) または Tunnel Interface (静的)。 |
| **認証方式** | PSK, デジタル証明書 (PKI), EAP, AAA。 |
| **ルーティング** | 動的ルーティング (BGP, OSPF, EIGRP) または IKEv2 経由の経路注入。 |
| **スケーラビリティ** | 非常に高い。AAA による属性プッシュで個別設定を排除。 |
| **特徴** | サイト間とリモートアクセスを同一プロファイルで共存可能。 |

---

## 🏗 動作原理

FlexVPN は「誰が（Identity）」接続し、「何ができるか（Authorization）」を分離して管理します。

```text
[ Initiator (Spoke/Client) ] <--- IKEv2 AUTH ---> [ Responder (Hub) ]
                                                        ↓
                                              [ AAA Authentication ]
                                                        ↓
                                              [ AAA Authorization ]
                                              (IP Pool, ACL, Routing)
                                                        ↓
                                              [ Virtual-Access ]
                                              (Virtual-Template からクローン)
```

---

## ⚙ 動作シーケンス

1.  **IKE_SA_INIT**: 暗号アルゴリズム、Diffie-Hellman 鍵交換、および Nonce の交換。
2.  **IKE_AUTH**: 
    *   デバイス認証（証明書または PSK）を実施。
    *   ハブ側で **IKEv2 Profile** をマッチさせ、AAA サーバーへユーザー情報を照会。
3.  **CREATE_CHILD_SA**: データ転送用の IPsec SA を構築。
4.  **Interface Cloning**: `Virtual-Template` を元に、セッション固有の `Virtual-Access` インターフェイスを生成。
5.  **Policy Push**: AAA から取得した ACL やルート情報をインターフェイスに適用。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **IKEv2 Smart Defaults**: Cisco IOS には標準のプロポーザルとポリシーが含まれています。要件で「デフォルトを無効にせよ」または「特定のアルゴリズムを指定せよ」とあれば、明示的な `crypto ikev2 proposal` の作成が必要です。
*   **Virtual-Template の正確な設定**: `type tunnel` を使用し、`ip unnumbered` や `tunnel protection` を漏れなく設定できるかが鍵です。
*   **Any-to-Any 通信**: スポーク間での直接通信（ショートカット）には、ハブとスポークの両方で NHRP の設定が必要になる場合があります。
*   **証明書認証 (PKI)**: IOS-XE を CA とし、SCEP 経由で証明書を取得して FlexVPN を確立する一連のフローは頻出です。
*   **トラブルシュート**: `show crypto ikev2 sa` で状態を確認し、`debug crypto ikev2` で Proposal の不一致や ID ミスマッチを特定するスキルが求められます。

---

## 🛠 設定方法

### 1. IKEv2 Proposal & Policy (IOS-XE)
```bash
crypto ikev2 proposal FLEX-PROP 
 encryption aes-gcm-256
 group 19
!
crypto ikev2 policy FLEX-POL 
 proposal FLEX-PROP
```

### 2. IKEv2 Profile & Virtual-Template (Hub)
```bash
crypto ikev2 profile HUB-PROFILE
 match identity remote any
 authentication local rsa-sig
 authentication remote rsa-sig
 pki trustpoint MY-CA
 virtual-template 1
!
interface Virtual-Template1 type tunnel
 ip unnumbered Loopback0
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-PROF
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **IKEv2 SA の状態確認** | <code>show crypto ikev2 sa [detail]</code> |
| **IPsec SA の統計確認** | <code>show crypto ipsec sa</code> |
| **動的生成インターフェイス確認** | <code>show derived-config interface Virtual-AccessX</code> |
| **セッション概要確認** | <code>show crypto session [detail]</code> |
| **リアルタイムデバッグ** | <code>debug crypto ikev2 [error\|packet]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| IKEv2 SA が確立しない | Proposal の不一致 | <code>debug cry ikev2</code> でアルゴリズムを確認。 |
| 認証に失敗する (AUTH_FAILED) | PSK 不一致または証明書無効 | NTP の同期状態と証明書の有効期限を確認。 |
| 経路が学習されない | Authorization ポリシーのミス | AAA からの <code>Cisco-AV-Pair</code> 属性を確認。 |
| スポーク間直接通信不可 | NHRP または CEF 無効 | ハブに <code>ip nhrp redirect</code>、スポークに <code>shortcut</code> 設定を確認。 |

---

## ⚠ 制限事項

*   **プラットフォーム依存**: 高度な AAA 統合や Virtual-Template 機能は IOS-XE に特化しており、ASA/FTD では限定的な実装となります。
*   **IKEv1 非互換**: FlexVPN は IKEv2 専用です。IKEv1 デバイスとの接続には従来の Crypto Map 等を併用する必要があります。
*   **MTU/MSS**: 暗号化オーバーヘッドを考慮した <code>ip tcp adjust-mss</code> の設定がネットワークの安定稼働に不可欠です。

---

## 🔄 他技術との関連

*   **Cisco ISE (2.2)**: FlexVPN ユーザーの認証・認可および SGT (Security Group Tag) 割り当てに使用されます。
*   **DMVPN (2.3)**: FlexVPN は mGRE をサポートし、DMVPN のような柔軟なトポロジを IKEv2 で構築可能です。
*   **ZBF (1.9)**: 動的に生成される `Virtual-Access` を適切なセキュリティゾーンに所属させる必要があります。

---

## 🧩 比較表

### FlexVPN vs 従来の DMVPN

| 特徴 | FlexVPN | DMVPN |
| :--- | :--- | :--- |
| **プロトコル** | IKEv2 (標準) | IKEv1/IKEv2 |
| **構成要素** | IKEv2 Profile 重視 | mGRE + NHRP 重視 |
| **RA統合** | ネイティブでサポート | 非対応 (Site-to-Site 専用) |
| **ポリシー管理** | AAA による一元管理 | 個別デバイス設定が基本 |

---

## 💡 ベストプラクティス

1.  **IKEv2 Name Configuration**: 複数の VPN サービスをホストする場合、IKEv2 プロファイル内で `match identity` を使用して明確に分離します。
2.  **証明書認証の推奨**: 管理負荷の低減とセキュリティ向上のため、PSK よりもデジタル証明書を使用します。
3.  **BGP によるスケーリング**: 経路交換には、スケーラビリティに優れた BGP を FlexVPN 上で走行させることを検討します。
4.  **AES-GCM の使用**: パフォーマンスとセキュリティを両立させるため、可能であれば AES-GCM 暗号化をプロポーザルに含めます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な FlexVPN Hub (PSK)
*   **要件**: 全スポークからの接続を PSK "cisco123" で受け入れよ。
*   **設定**: `ikev2 keyring` -> `ikev2 profile` -> `virtual-template`。

### 2. FlexVPN スポークの構成
*   **要件**: ハブ 1.1.1.1 に対して静的 Tunnel インターフェイスで接続せよ。
*   **設定**: `interface Tunnel0` 内で `tunnel destination 1.1.1.1`。

### 3. デジタル証明書 (PKI) による認証
*   **要件**: ハブ R1 を CA とし、スポーク R2 を証明書で認証せよ。
*   **設定**: `crypto pki server` 設定および `authentication local/remote rsa-sig`。

### 4. AAA Authorization による IP 割り当て
*   **要件**: 接続ユーザーにローカルプール "VPN-POOL" から IP を配れ。
*   **設定**: `crypto ikev2 authorization policy` 内で `address-pool` を定義。

### 5. FlexVPN スプリットトンネル (RA)
*   **要件**: リモートアクセスユーザーに対し、10.0.0.0/8 への通信のみを許可せよ。
*   **設定**: AAA 属性で `access-list` をプッシュ。

### 6. NHRP ショートカットの有効化
*   **要件**: スポーク間で直接通信が発生するように構成せよ。
*   **設定**: `ip nhrp redirect` (Hub) および `ip nhrp shortcut` (Spoke)。

### 7. Dual Hub FlexVPN
*   **要件**: 2台のハブを冗長構成で運用せよ。

### 8. FlexVPN over IPv6 アンダーレイ
*   **要件**: インターネット接続が IPv6 の環境で VPN を構築せよ。

### 9. Per-Peer QoS の適用
*   **要件**: AAA を使用して特定のスポークに帯域制限を適用せよ。
*   **設定**: `Cisco-AV-Pair` で `lcp:interface-config=service-policy input LIMIT`。

### 10. Local AAA Fallback
*   **要件**: RADIUS サーバーがダウンした際にローカル DB で認証せよ。

---

## ❓ 想定試験問題

1.  **Design**: FlexVPN において、サイト間 VPN とリモートアクセス VPN を同一のハブで共存させるために必要な構成要素は？
    *   **回答**: 共通の IKEv2 プロファイル、または Identity に基づいた複数のプロファイルの使い分け、および `Virtual-Template`。
2.  **トラブルシュート**: スポークからハブへのトンネルは UP しているが、スポーク間で直接通信ができない。どのコマンドで診断を開始すべきか？
    *   **回答**: スポーク側で `show ip nhrp shortcut` および、ハブ側で `ip nhrp redirect` が設定されているか確認。
3.  **コンフィグ読解**: `crypto ikev2 profile` 内に `match identity remote fqdn domain.com` がある場合の動作は？
    *   **回答**: ピアが自身の ID として `domain.com` という FQDN を提示した場合のみ、このプロファイルにマッチする。
4.  **実装**: FlexVPN 経由で動的ルーティングを確立する際、ハブ側で必要なインターフェイス設定は？
    *   **回答**: `Virtual-Template` での `ip address` または `ip unnumbered` 設定。
5.  **Design**: 数千のスポークを収容する際、ハブのコンフィグが煩雑になるのを防ぐ FlexVPN の機能は？
    *   **回答**: AAA Authorization による属性（IP, Route, ACL）の動的プッシュ。

---

## 🔗 参考リソース

*   [Cisco IOS-XE IKEv2 FlexVPN Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_conn_ikeev2/configuration/xe-16/sec-flex-vpn-xe-16-book.html)
*   [Cisco Live: BRKSEC-2121 - FlexVPN Advanced Architectures](https://www.ciscolive.com/)
*   [Technical Note: Troubleshooting FlexVPN Common Issues](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-key-management/116221-technote-flexvpn-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「FlexVPN = IKEv2 + Virtual-Template + AAA」と覚えると構造が整理されます。
*   **図解**: `Virtual-Template` が親、`Virtual-Access` が子。セッションごとに子がクローンされるイメージを持つことが重要です。
*   **注意点**: ラボ試験では `crypto logging ikev2` を有効にすると、デバッグが非常に楽になります。
