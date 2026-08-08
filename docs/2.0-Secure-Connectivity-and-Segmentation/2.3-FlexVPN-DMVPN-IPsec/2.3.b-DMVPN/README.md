# 2.3.b DMVPN (Dynamic Multipoint VPN)

DMVPNは、ハブアンドスポーク型またはフルメッシュ型のVPNトポロジを動的かつスケーラブルに構築するためのCisco独自のソリューションです。従来のGRE over IPsecでは、各拠点（スポーク）ごとに固定のトンネル設定が必要でしたが、DMVPNは **mGRE (Multipoint GRE)** と **NHRP (Next Hop Resolution Protocol)** を組み合わせることで、スポーク側の動的IPアドレス環境下でも、最小限の設定で数千の拠点を収容することを可能にします。

---

## 📘 概要

*   **機能概要**: ハブ（NHS: Next Hop Server）が全スポークの物理IPアドレス（NBMA）と仮想トンネルIPのアドレスマッピングを動的に管理し、スポーク間のオンデマンドな直接通信（ショートカット）を支援します。
*   **利用目的**: 大規模な拠点間接続における管理負荷の軽減、およびスポーク間通信時の遅延とハブの負荷削減。
*   **どのような場面で利用するか**: 支店数が多い企業のWAN構築、SD-WANの基盤技術、またはインターネットを介した動的なフルメッシュネットワークが必要な環境。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要コンポーネント** | mGRE, NHRP, IPsec (IKEv1/IKEv2), 動的ルーティング (EIGRP, OSPF, BGP)。 |
| **ハブの役割** | **NHS (Next Hop Server)** として動作。マッピングデータベースを保持する。 |
| **スポークの役割** | **NHC (Next Hop Client)** として動作。起動時にハブへ自身の情報を登録する。 |
| **セキュリティ** | `tunnel protection` によるIPsec暗号化。認証にはPSKまたはPKIを使用可能。 |
| **フェーズの種類** | Phase 1 (ハブ経由のみ), Phase 2 (直接通信可), Phase 3 (Redirect/Shortcutによる最適化)。 |
| **設計上の注意点** | MTU/MSSの調整（トンネルオーバーヘッド）、ルーティングプロトコルのSplit-horizon無効化。 |

---

## 🏗 動作原理

DMVPNは、パケット転送のための「データプレーン」と、アドレス解決のための「コントロールプレーン」を分離しています。

```text
[ Spoke A ] <------- (Internet/Underlay) -------> [ Hub (NHS) ]
     |                                               |
     |--- 1. NHRP Registration Request ------------> | (Aの物理IPとトンネルIPを記録)
     |                                               |
     |--- 2. Data to Spoke B ----------------------> | (最初はハブ経由)
     |                                               |
     | <--- 3. NHRP Redirect (Phase 3) ------------- | (ハブ: 「Bには直接行けるぞ」)
     |                                               |
     | --- 4. NHRP Resolution Request -------------> | (AからBの物理IPを問い合せ)
     | <--- 5. NHRP Resolution Reply --------------- | (ハブからBの物理IPを回答)
     |                                               |
[ Spoke A ] <--- 6. Direct Dynamic Tunnel ------> [ Spoke B ]
```

---

## ⚙ 動作シーケンス

1.  **物理接続確立**: スポークがISP経由でインターネットに接続。
2.  **IPsec SA確立**: ハブとスポーク間でISAKMP/IKEv2ネゴシエーションを実施。
3.  **NHRP登録**: スポークがハブ（NHS）に対し、自身のトンネルIPとNBMA（物理IP）のアドレスマッピングを登録する。
4.  **ルーティング確立**: トンネル経由でルーティングプロトコル（EIGRP等）を走行させ、拠点情報を交換。
5.  **スポーク間ショートカット (Phase 2/3)**:
    *   スポークAがスポークB宛のトラフィックをハブへ送信。
    *   ハブが「ショートカット可能」と判断し、Redirectメッセージを送信。
    *   スポークAがBのNBMAアドレスをハブに照会し、A-B間に動的トンネルを作成。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Phase 3 の設定**: ラボ試験では Phase 3 が標準です。ハブに `ip nhrp redirect`、スポークに `ip nhrp shortcut` が設定されているか、必ず確認してください。
*   **OSPF ネットワークタイプ**: OSPFを使用する場合、ハブでの `ip ospf network point-to-multipoint` 設定が重要です。これによりDR選出が回避され、Next-hopが正しく維持されます。
*   **Split-Horizon**: EIGRPを使用する場合、ハブのトンネルインターフェイスで `no ip split-horizon eigrp <AS>` を設定しないと、スポーク間でルートが交換されません。
*   **IPsec Profile**: 従来の Crypto Map ではなく `crypto ipsec profile` を作成し、`tunnel protection` コマンドでトンネルに適用する手順を習得してください。
*   **Next-hop の上書き**: `no ip next-hop-self eigrp` (ハブ側) が Phase 2 で必要になる場合があります。
*   **トラブルシュートコマンド**: `show ip nhrp` でマッピングが `dynamic` かつ `unique` に登録されているかを確認できることが合格への鍵です。

---

## 🛠 設定方法

### 1. ハブの設定 (Phase 3)
```bash
interface Tunnel0
 ip address 172.16.1.1 255.255.255.0
 no ip redirects
 ip nhrp authentication cisco123
 ip nhrp map multicast dynamic
 ip nhrp network-id 1
 ip nhrp redirect         ! Phase 3 必須
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel key 12345
 tunnel protection ipsec profile DMVPN-PROF
```

### 2. スポークの設定 (Phase 3)
```bash
interface Tunnel0
 ip address 172.16.1.10 255.255.255.0
 ip nhrp authentication cisco123
 ip nhrp map 172.16.1.1 203.0.113.1   ! HubのトンネルIPと物理IP
 ip nhrp map multicast 203.0.113.1
 ip nhrp network-id 1
 ip nhrp nhs 172.16.1.1
 ip nhrp shortcut       ! Phase 3 必須
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel key 12345
 tunnel protection ipsec profile DMVPN-PROF
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **NHRPマッピングの確認** | <code>show ip nhrp [dynamic\|static]</code> |
| **スポーク登録状態の確認** | <code>show ip nhrp brief</code> |
| **IPsec SAの確立確認** | <code>show crypto ipsec sa</code> |
| **IKE SAの状態確認** | <code>show crypto isakmp sa</code> |
| **ルーティングテーブルの確認** | <code>show ip route</code> |
| **NHRPデバッグ** | <code>debug nhrp condition</code> / <code>debug nhrp packet</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| ネイバーが確立しない | `tunnel key` の不一致 | ハブとスポークの <code>tunnel key</code> 番号を揃える。 |
| 経路が学習されない | Split-horizon の有効化 | ハブで <code>no ip split-horizon</code> を設定。 |
| スポーク間直接通信不可 | CEFが無効 | <code>ip cef</code> が有効であることを確認。Phase 3 では <code>redirect/shortcut</code> を確認。 |
| 特定サイズのパケットがドロップ | MTU/MSS設定ミス | <code>ip mtu 1400</code> および <code>ip tcp adjust-mss 1360</code> を設定。 |
| IKE認証失敗 | PSK/証書エラー | <code>show clock</code> で時刻を確認し、PSKの文字列を再確認。 |

---

## ⚠ 制限事項

*   **プラットフォーム**: DMVPNは Cisco IOS/IOS-XE ルータの機能であり、ASA や FTD ではハブ/スポークとして動作できません。
*   **プロトコル制限**: NHRP は CEF (Cisco Express Forwarding) に依存するため、CEF を無効にすると Phase 2/3 の動的トンネルが作成されません。
*   **NAT**: スポークが NAT デバイスの背後にある場合、IPsec NAT-Traversal (UDP 4500) の許可が必要です。

---

## 🔄 他技術との関連

*   **Routing**: EIGRP, OSPF, BGP 等の動的ルーティングがトンネル上で必須。
*   **PKI**: IOS CA を使用して、PSK ではなくデジタル証明書による大規模認証を実装。
*   **QoS**: トンネルインターフェイスへの `service-policy` 適用による拠点ごとの帯域制御。
*   **FlexVPN**: DMVPN の次世代版。IKEv2 を使用し、より柔軟なポリシー適用が可能。

---

## 🧩 比較表

### DMVPN Phase の違い

| 特徴 | Phase 1 | Phase 2 | Phase 3 |
| :--- | :--- | :--- | :--- |
| **スポーク設定** | ユニキャスト GRE | **mGRE** | **mGRE** |
| **スポーク間通信** | ハブ経由 (常に) | **直接通信 (フルメッシュ)** | **直接通信 (最適化)** |
| **Next-hop** | ハブを指す | スポークを指す | 最初はハブ、後にスポーク |
| **主なメリット** | 設定が極めてシンプル | 帯域効率が良い | **スケーラビリティが最高** |

---

## 💡 ベストプラクティス

1.  **Phase 3 の採用**: 最もスケーラブルでトラブルが少ない Phase 3 を常に優先します。
2.  **IKEv2 への移行**: セキュリティと安定性の向上のため、IKEv1 (ISAKMP) ではなく IKEv2 を使用します。
3.  **デュアルハブ構成**: 可用性を高めるため、複数のハブを異なる拠点（または異なるISP）に配置し、Dual Cloud 構成にします。
4.  **MTU調整の徹底**: `ip mtu 1400` を基準とし、フラグメンテーションを防ぐための `tcp adjust-mss` を適用します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な Phase 1 (Hub-and-Spoke)
*   **要件**: スポーク側の NBMA アドレスが動的な環境で、ハブ経由のみの通信を確立せよ。
*   **ポイント**: スポークのトンネルを `tunnel mode gre ip`（ユニキャスト）にする。

### 2. DMVPN Phase 3 (EIGRP)
*   **要件**: スポーク間での直接通信を有効にし、ハブでの経路集約を可能にせよ。
*   **ポイント**: ハブに `ip nhrp redirect`、スポークに `ip nhrp shortcut` を設定。

### 3. OSPF over DMVPN (Broadcast)
*   **要件**: トンネルを OSPF ブロードキャストネットワークとして構成せよ。
*   **ポイント**: ハブの Priority を高く設定して常に DR にし、スポークは 0 に設定する。

### 4. OSPF over DMVPN (Point-to-Multipoint)
*   **要件**: スポーク間での DR 選出を回避し、自動的に Next-hop を解決せよ。
*   **ポイント**: `ip ospf network point-to-multipoint` を全ノードに適用。

### 5. デュアルハブ・シングルクラウド
*   **要件**: 1つのトンネルサブネット内で2台のハブを冗長化せよ。
*   **ポイント**: スポーク側で `ip nhrp nhs` を2つ列挙する。

### 6. IPsec PSK による保護
*   **要件**: ワイルドカード PSK を使用して、全スポークからの接続を許可せよ。
*   **設定**: `crypto isakmp key cisco123 address 0.0.0.0`。

### 7. デジタル証明書 (PKI) 認証
*   **要件**: IOS CA をハブで稼働させ、スポークを証明書で認証せよ。

### 8. DMVPN 経由の BGP 構成
*   **要件**: スケーラビリティ向上のため、iBGP を DMVPN 上で走行させよ。
*   **ポイント**: `neighbor <VPN_IP> next-hop-self` の挙動に注意。

### 9. NHRP 登録時間の調整
*   **要件**: スポークのNBMA変更を迅速に反映させるため、ホールドタイムを短縮せよ。
*   **設定**: `ip nhrp holdtime 300`。

### 10. IPv6 アンダーレイでの DMVPN
*   **要件**: インターネット接続が IPv6 の環境で、IPv4 トンネルを構築せよ。
*   **ポイント**: `tunnel source` に IPv6 アドレスまたは IF を指定。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: スポークからハブへの ping は通るが、スポーク A からスポーク B への ping が通らない。`show ip nhrp` を実行した際、B のエントリーが表示されない。考えられる原因は？
    *   **回答**: ハブで `ip nhrp redirect` が設定されていない、またはスポーク A で `ip nhrp shortcut` が欠落している。
2.  **Design**: DMVPN Phase 2 と Phase 3 における「ハブでの経路集約（Summarization）」の可否とその理由を述べよ。
    *   **回答**: Phase 2 では集約不可（Next-hopが消失するため）。Phase 3 では NHRP Redirect により Next-hop を動的に書き換えるため、ハブでの集約が可能。
3.  **コンフィグ読解**: ハブのトンネル設定に `ip nhrp map multicast dynamic` が含まれている理由を説明せよ。
    *   **回答**: ルーティングプロトコルのハロー（マルチキャスト）を、登録されている全スポーク（動的NBMA）へ複製して送信するため。
4.  **実装**: EIGRP を使用している環境で、スポーク側で他拠点のルートが学習されない。ハブの設定で確認すべきコマンドは？
    *   **回答**: `no ip split-horizon eigrp <AS>`。
5.  **トラブルシュート**: `show crypto isakmp sa` で状態が `QM_IDLE` であるのに、通信ができない。次に見るべき NHRP の状態は？
    *   **回答**: `show ip nhrp` でマッピングが正しく作成されているか、有効期限が切れていないかを確認。

---

## 🔗 参考リソース

*   **Cisco Configuration Guides**
    *   [Dynamic Multipoint VPN (DMVPN) Configuration Guide, Cisco IOS XE](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_conn_dmvpn/configuration/xe-16/sec-conn-dmvpn-xe-16-book.html)
*   **Cisco Live**
    *   [BRKSEC-3052: DMVPN - Phases, Implementation, and Troubleshooting](https://www.ciscolive.com/)
*   **Technical Notes**
    *   [DMVPN Phase 3: NHRP Redirect and Shortcut Explanation](https://www.cisco.com/c/en/us/support/docs/security-vpn/dynamic-multipoint-vpn-dmvpn/119022-technote-dmvpn-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「DMVPN はコントロールプレーン（NHRP）が命」であることを意識してください。IPsec SA が確立していても、NHRP 登録が失敗すればトラフィックは流れません。
*   **図解**: パケットの流れを追うときは、Underlay（ISP）と Overlay（Tunnel）のIPアドレスを明確に区別して書き出すのがミスを防ぐコツです。
*   **注意点**: ラボ試験では、`no ip redirects` (ハブの物理IF) など、細かいが不可欠なコマンドの入力を忘れないようにしましょう。
