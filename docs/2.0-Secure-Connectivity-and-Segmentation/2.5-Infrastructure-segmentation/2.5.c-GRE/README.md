---
layout: default
title: 2.5.c-GRE
nav_order: 3
parent: 2.5-Infrastructure-segmentation
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.5.c GRE (Generic Routing Encapsulation)

GRE（Generic Routing Encapsulation）は、ネットワーク層プロトコルを別のプロトコル内にカプセル化するための汎用的なトンネリングプロトコルです。CCIE Security v6.1のブループリントにおいて、GREはインフラセグメンテーションの重要な手法として位置付けられており、特にIPsec VPN上でルーティングプロトコル（マルチキャスト）を通過させるための基盤技術として不可欠です。

---

## 📘 概要

*   **機能概要**: 任意のレイヤ3パケットをGREヘッダーで包み、新しいIPヘッダーを付与して転送します。これにより、物理的に離れたネットワーク間に仮想的なポイントツーポイント（P2P）リンクを構築します。
*   **利用目的**: 非IPトラフィックの転送、プライベートIPのオーバーレイ、およびマルチキャスト（OSPF/EIGRP等のハローパケット）の透過的な伝送。
*   **利用場面**:
    *   **IPsecとの併用**: IPsecはマルチキャストを直接サポートしないため、GREトンネルをIPsecで保護する（GRE over IPsec）ことで動的ルーティングを実現します。
    *   **DMVPNの基盤**: **mGRE (Multipoint GRE)** を使用して、ハブ・スポーク間の動的接続を実現します。
    *   **セグメンテーション**: 異なるVRF間や組織間のトラフィックを分離して輸送する場合に利用します。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **プロトコル番号** | **IP 47** |
| **オーバーヘッド** | 基本 24 バイト（GRE 4B + 外側IP 20B）。オプション追加で増大。 |
| **主なコンポーネント** | Tunnel Source, Tunnel Destination, Tunnel Interface。 |
| **マルチキャスト** | **ネイティブでサポート**。OSPF/EIGRP 等の走行が可能。 |
| **セキュリティ** | **暗号化機能なし**。IPsec（Tunnel Protection）による保護が必須。 |
| **インターフェイス** | 論理的な `interface Tunnel` を使用。 |
| **動作モード** | Static P2P (GRE/IP) または mGRE。 |

---

## 🏗 動作原理

GREは、元のIPパケット（ペイロード）に対してGREヘッダーを付加し、さらに外側のIPヘッダー（輸送ヘッダー）でカプセル化します。

```text
[ 元のパケット ]
(Payload: IP / TCP / Data)
       ↓
[ カプセル化プロセス ]
(New IP Header) + (GRE Header) + (元のパケット)
       ↓
[ 転送 ]
ルータ間を通常のIPパケットとして通過
       ↓
[ カプセル解除 ]
対向ルータが外側ヘッダーとGREヘッダーを剥離し、内部パケットを転送
```

---

## ⚙ 動作シーケンス

1.  **トラフィックの受信**: ルータがトンネル宛のルート（スタティックまたは動的）にマッチするパケットを受信。
2.  **GREエンカプセル**: パケットにGREヘッダー（4バイト）を付与し、さらに `tunnel destination` を宛先とした外側IPヘッダーを付与。
3.  **ルーティング**: カプセル化されたパケットを、物理インターフェイスから送り出す。
4.  **パケット受信**: 対向ルータがIPプロトコル47を受信し、トンネル設定を確認。
5.  **デカプセル**: 外側ヘッダーを剥離し、元のパケットをルーティングテーブルに従って転送。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **MTUとフラグメンテーション**: GREの24バイトのオーバーヘッドにより、物理IFのMTUを超えやすくなります。ラボ試験では、`ip mtu 1400` および `ip tcp adjust-mss 1360` の設定が正しく行われているかが厳しくチェックされます。
*   **再帰ルーティング (Recursive Routing)**: トンネルの宛先IPへの経路が、そのトンネルインターフェイス自体をネクストホップとして学習されると、トンネルがフラッピングします。これは頻出のトラブルシュート項目です。
*   **Tunnel Protection**: `crypto ipsec profile` を作成し、トンネルインターフェイスに `tunnel protection ipsec profile` を適用してGREトラフィックを暗号化する手順を習得してください。
*   **Keepalive**: GREはデフォルトでは、`tunnel source` が生きていれば対向の状態に関わらず `up/up` になります。対向のダウンを検知するために `keepalive` コマンドの使用が推奨されます。
*   **GRE over IPsec モード**: IPsecを適用する場合、GREヘッダーが既に存在するため、IPsecのトランスポートモード（Transport Mode）を使用して無駄なIPヘッダーを削減することがベストプラクティスです。

---

## 🛠 設定方法

### 1. 基本的なポイントツーポイントGRE設定
```bash
interface Tunnel0
 ip address 10.255.1.1 255.255.255.252
 tunnel source GigabitEthernet1
 tunnel destination 203.0.113.2
 ! MTU調整は必須
 ip mtu 1400
 ip tcp adjust-mss 1360
```

### 2. GRE over IPsec (VTI的な保護)
```bash
crypto ipsec profile GRE-PROTECT
 set transform-set TSET
!
interface Tunnel0
 ...
 tunnel protection ipsec profile GRE-PROTECT
```

### 3. Multipoint GRE (mGRE) - DMVPN基盤
```bash
interface Tunnel0
 ip address 172.16.1.1 255.255.255.0
 tunnel mode gre multipoint
 ip nhrp network-id 1
 ...
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **トンネルインターフェイスの状態確認** | <code>show interfaces tunnel [ID]</code> |
| **カプセル化/解除の統計確認** | <code>show ip interface brief</code> |
| **IPsec保護の状態確認** | <code>show crypto session detail</code> |
| **ルーティングネクストホップの確認** | <code>show ip route [destination]</code> |
| **MTU/MSSの確認** | <code>show run interface tunnel [ID]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| トンネルが <code>up/down</code> | 物理的な到達性欠如 | <code>tunnel destination</code> への ping を確認。 |
| トンネルが <code>up/up</code> だが ping 不通 | ACLによるプロトコル47の遮断 | 物理インターフェイスの ACL で <code>permit gre</code> を確認。 |
| トンネルが頻繁にフラッピング | 再帰ルーティング | 宛先 IP へのルートがトンネルを通っていないか <code>show ip route</code> で確認。 |
| 特定サイズのデータのみ通らない | MTU/MSS の不整合 | <code>ip mtu</code> および <code>ip tcp adjust-mss</code> の再設定。 |
| 経路学習ができない | スプリットホライズン | EIGRP 等では <code>no ip split-horizon</code> が必要。 |

---

## ⚠ 制限事項

*   **暗号化の欠如**: GRE単体では平文で送信されるため、インターネット経由では必ず IPsec と併用します。
*   **NATの制約**: GRE (IP 47) はポート番号を持たないため、PAT (Port Address Translation) 環境下での通過には NAT-Traversal 等の配慮が必要です。
*   **ステートレス**: デフォルトの GRE はセッション状態を持たないため、片方向の障害を検知できません。

---

## 🔄 他技術との関連

*   **IPsec (2.3.c)**: GREの保護に使用。Transportモードとの組み合わせが最適。
*   **DMVPN (2.3.b)**: mGRE (Multipoint GRE) を使用して、1つの IF で複数のピアを収容します。
*   **Routing Protocols**: OSPF/EIGRP 等の動的ルーティングを確立するための仮想リンクを提供します。
*   **VRF-Lite (2.5.d)**: VRFごとに個別の GRE トンネルを構築し、L3セグメンテーションを維持したまま転送します。

---

## 🧩 比較表

### GRE vs IPsec VTI (Virtual Tunnel Interface)

| 特徴 | GRE over IPsec | IPsec VTI |
| :--- | :--- | :--- |
| **柔軟性** | 非IP/マルチキャストに非常に強い | IP/マルチキャストに対応 |
| **オーバーヘッド** | 高い（24バイト追加） | 若干低い |
| **設定** | 2段階（Tunnel + Protection） | 1段階 |
| **推奨場面** | レガシー非IP転送や複雑なDMVPN | 純粋なIPベースのS2S VPN |

---

## 💡 ベストプラクティス

1.  **常に MTU を意識**: トンネルパケットのフラグメンテーションは CPU 負荷を増大させます。1400バイト以下への調整を標準とします。
2.  **Transport Mode の使用**: GRE over IPsec では、不要な外側 IP ヘッダーを 1 つ分節約できます。
3.  **Tunnel Key の活用**: 同一の Source/Destination ペアで複数のトンネルを張る場合、`tunnel key [ID]` で識別します。
4.  **Loopback の利用**: `tunnel source` には物理 IP ではなく安定した Loopback IP を使用し、冗長経路を確保します。

---

## 📝 ラボ学習・設定サンプル例

### 1. サイト間 GRE トンネルの基本構築
*   **要件**: R1 と R2 の Gi1 インターフェイスを使用して、仮想 P2P リンクを張れ。
*   **設定**: `interface Tunnel0`, `tunnel source Gi1`, `tunnel destination <R2_IP>`。

### 2. GRE 経由の OSPF ネイバー確立
*   **要件**: トンネルインターフェイスを OSPF Area 0 に含めよ。
*   **設定**: `router ospf 1`, `network 10.255.1.0 0.0.0.3 area 0`。

### 3. IPsec トランスポートモードによる GRE 保護
*   **要件**: GRE トラフィックを AES で暗号化し、ヘッダーサイズを最小化せよ。
*   **設定**: `crypto ipsec transform-set TSET esp-aes esp-sha-hmac`, `mode transport`。

### 4. GRE トンネルにおける MTU 最適化
*   **要件**: パケットサイズが原因の疎通不良を解消せよ。
*   **設定**: `ip mtu 1400`, `ip tcp adjust-mss 1360`。

### 5. GRE Tunnel Keepalive の実装
*   **要件**: 対向の Tunnel IF がシャットダウンされた際、自側もダウン状態にせよ。
*   **設定**: `keepalive 10 3`。

### 6. mGRE (Multipoint GRE) の基本設定
*   **要件**: 複数のスポークを 1 つのハブインターフェイスで待ち受けよ。
*   **設定**: `tunnel mode gre multipoint`。

### 7. GRE Tunnel Key によるトンネル分離
*   **要件**: 同一ペア間で 2 つの独立したトンネルを構築せよ。
*   **設定**: `tunnel key 100`, `tunnel key 200`。

### 8. 再帰ルーティングのトラブルシュート
*   **課題**: フラッピングするトンネルの経路を修正せよ。
*   **手法**: トンネル宛の IP を 32bit スタティックルートで物理 IF 側に固定する。

### 9. VRF を跨ぐ GRE トンネル (VRF-Lite)
*   **要件**: VRF "RED" のトラフィックを GRE 経由で転送せよ。
*   **設定**: `interface Tunnel0`, `vrf forwarding RED`。

### 10. GRE over IPsec (Crypto Map 方式)
*   **要件**: 従来の Crypto Map を使用して GRE (プロトコル 47) を保護せよ。
*   **設定**: `access-list 100 permit gre host <src> host <dst>`。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: GRE トンネルインターフェイスは `up/up` だが、OSPF ネイバーが `INIT` から進まない。物理インターフェイスの ACL で何を許可すべきか？
    *   **回答**: 外側の IP ヘッダーに基づく **GRE (IP Protocol 47)**。
2.  **Design**: IPsec トンネルで GRE パケットをカプセル化する際、カプセル化オーバーヘッドを最小にするために推奨される IPsec モードは？
    *   **回答**: **トランスポートモード (Transport Mode)**。GRE が既にトンネルヘッダーを提供しているため。
3.  **コンフィグ読解**: `tunnel mode gre multipoint` コマンドが設定されている場合、このルータは何の役割を果たすことが多いか？
    *   **回答**: DMVPN におけるハブ (NHS) または FlexVPN のレスポンダー。
4.  **実装**: GRE トンネル上で EIGRP を動作させているが、スポーク間でルートが学習されない。ハブ側で必要な設定は？
    *   **回答**: トンネルインターフェイスでの `no ip split-horizon eigrp <AS>`。
5.  **トラブルシュート**: GRE トンネルを介した Web 通信 (HTTP) が一部のサイトで失敗する。ICMP は通る。どのコマンドで修正を試みるべきか？
    *   **回答**: トンネルインターフェイスでの `ip tcp adjust-mss 1360`。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [Generic Routing Encapsulation (GRE) Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/interface/configuration/xe-16/ir-xe-16-book/ir-cfg-gre-xe.html)
*   **Technical Notes**
    *   [Troubleshooting GRE Tunnel Recursive Routing Issues](https://www.cisco.com/c/en/us/support/docs/ip/generic-routing-encapsulation-gre/118370-technote-gre-00.html)
    *   [IPsec Transport Mode with GRE Explanation](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-key-management/116221-technote-flexvpn-00.html)
*   **Integrated Security Technologies and Solutions, Volume II**

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「GRE はただの筒（筒自体に鍵はない）」とイメージしてください。鍵（暗号化）をかけるのが IPsec の役割です。
*   **図解**: パケットがルータを通過するたびに、ヘッダーが「着替え」をしていく様子（Encapsulation/Decapsulation）を紙に書いて整理しましょう。
*   **注意点**: ラボ試験では `tunnel source` に指定した IP アドレスに到達性がない場合、インターフェイスは `up/down` になります。まず下位レイヤの導通を優先してください。
