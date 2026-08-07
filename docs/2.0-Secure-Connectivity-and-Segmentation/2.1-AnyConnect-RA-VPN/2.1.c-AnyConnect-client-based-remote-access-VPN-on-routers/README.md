---
layout: default
title: 2.1.c Cisco AnyConnect client-based, remote-access VPN technologies on Cisco routers
nav_order: 3
parent: 2.1-AnyConnect-RA-VPN
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.1 Cisco AnyConnect client-based, remote-access VPN technologies on Cisco routers

Cisco IOS-XE ルータにおける **AnyConnect リモートアクセス VPN** は、主に **FlexVPN** フレームワークを利用して実装されます。ASA が SSL VPN を得意とするのに対し、ルータ（ISR/ASR）は **IKEv2** プロトコルを使用した AnyConnect 接続の終端プラットフォームとして機能します。これは、単一のフレームワークでサイト間（S2S）、リモートアクセス（RA）、ハブアンドスポークの全構成をカバーできる高い柔軟性を持っています。

---

## 📘 概要

*   **機能概要**: IKEv2 プロトコルに基づき、AnyConnect クライアントと Cisco ルータ間でセキュアな暗号化トンネルを確立します。
*   **利用目的**: 高度なスケーラビリティが求められる環境や、既存の IOS インフラを統合的な VPN ハブとして活用する場合に利用します。
*   **利用場面**: 大規模なテレワーカー収容、証明書ベースの強固な認証が必要な環境、または ASA ではなくルータでの VPN 集約が必要な設計において選択されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要プロトコル** | **IKEv2** (必須)。AnyConnect 接続には IKEv2 のサポートが必要。 |
| **アーキテクチャ** | **FlexVPN**。Virtual-Template インターフェイスを使用して動的にセッションを生成。 |
| **認証方式** | EAP-AnyConnect, 証明書 (Certificates), AAA (RADIUS/ISE)。 |
| **メリット** | 単一設定で S2S と RA を混在可能。動的ルーティングプロトコルのネイティブサポート。 |
| **デメリット** | **SSL VPN (AnyConnect SSL) は現代の IOS-XE ではサポートされない**（IKEv2 のみ）。 |
| **対応機種** | ISR 4000 シリーズ, ASR 1000 シリーズ, CSR 1000v / C8000v。 |
| **設計上の注意点** | Virtual-Template から生成される Virtual-Access のクローン属性の管理。 |

---

## 🏗 動作原理

AnyConnect クライアントはルータの `crypto ikev2 profile` に定義されたポリシーに従って接続を試みます。

```text
AnyConnect Client
   ↓ (IKEv2 Auth)
Router Outside Interface
   ↓
[ IKEv2 Profile Check ] --- (Certificate/AAA Verification)
   ↓
[ AAA Authorization ] --- (Address assignment & Policy push)
   ↓
[ Virtual-Template ] --- (Cloning to Virtual-Access IF)
   ↓
[ Internal Network ]
```

---

## ⚙ 動作シーケンス

1.  **IKE_SA_INIT**: 暗号アルゴリズムのネゴシエーションと Diffie-Hellman 鍵交換。
2.  **IKE_AUTH**: 
    *   クライアントが証明書または EAP で認証。
    *   ルータは **AAA サーバー (ISE)** へ問い合わせ、認可属性（IP プール、ACL 等）を取得。
3.  **Configuration Payload (MODE_CFG)**: ルータからクライアントへ仮想 IP アドレス、DNS、スプリットトンネル情報をプッシュ。
4.  **Interface Creation**: ルータが `Virtual-Template` を元に、そのユーザー専用の `Virtual-Access` インターフェイスを生成。
5.  **Traffic Flow**: 暗号化された IPsec トンネルを通じて内部リソースへの通信を開始。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **EAP-AnyConnect**: ルータ側で `authentication remote anyconnect-eap` を設定し、クライアントにユーザー名/パスワードを要求させる構成は頻出です。
*   **Virtual-Template**: `type tunnel` を使用し、`ip unnumbered` でループバックを参照する正確な設定手順が問われます。
*   **IKEv2 Name Configuration**: AnyConnect の XML プロファイルで指定された「Gateway アドレス」とルータの `match identity remote` 設定の整合性。
*   **ISE 連携**: RADIUS VSA (Vendor Specific Attributes) を使用して、ISE からスプリットトンネル ACL を動的にプッシュする構成。
*   **Troubleshooting**: `debug crypto ikev2` を読み解き、Proposal ミスマッチや AAA 拒否のフェーズを特定できる必要があります。

---

## 🛠 設定方法

### 1. IKEv2 ポリシーとプロポーザル
```bash
crypto ikev2 proposal PROP 
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 policy POL 
 proposal PROP
```

### 2. AAA と IP アドレスプール
```bash
aaa new-model
aaa authentication login vpn_auth group radius
aaa authorization network vpn_author group radius
!
ip local pool VPN_POOL 10.50.50.1 10.50.50.100
```

### 3. IKEv2 Profile (AnyConnect 用)
```bash
crypto ikev2 profile ANYCONNECT_PROFILE
 match identity remote any
 authentication local rsa-sig
 authentication remote anyconnect-eap aggregate
 pki trustpoint MY_CA
 aaa authentication login vpn_auth
 aaa authorization network vpn_author
 virtual-template 1
```

### 4. Virtual-Template インターフェイス
```bash
interface Virtual-Template1 type tunnel
 ip unnumbered Loopback0
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC_PROF
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **IKEv2 SA の確認** | <code>show crypto ikev2 sa</code> |
| **セッション詳細（割り当てIP等）** | <code>show crypto session detail</code> |
| **Virtual-Access の状態** | <code>show interfaces virtual-access</code> |
| **IKEv2 リアルタイムデバッグ** | <code>debug crypto ikev2 [error\|packet]</code> |
| **AAA 通信のデバッグ** | <code>debug aaa authentication</code> / <code>debug radius</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| 認証に失敗する | `anyconnect-eap` の未設定 | `authentication remote` に `anyconnect-eap` を追加。 |
| IPが割り当てられない | プールの枯渇または AAA 設定ミス | `show ip local pool` で残数を確認、AAA の `authorization` 設定を見直す。 |
| インターフェイスが立ち上がらない | `tunnel protection` の欠如 | Virtual-Template に `tunnel protection ipsec profile` を適用。 |
| 通信が片方向 | 戻りルートの欠如 | 内部ネットワーク側で VPN プール宛のルートをルータへ向ける。 |

---

## ⚠ 制限事項

*   **プロトコル制限**: AnyConnect SSL (TCP 443 終端) はサポートされず、IKEv2 (UDP 500/4500) のみとなります。
*   **ライセンス**: AnyConnect クライアントをルータに接続させる場合でも、AnyConnect Apex/Plus ライセンスが必要です。
*   **最大数**: 生成可能な Virtual-Access 数は CPU/メモリ資源に依存します。

---

## 🔄 他技術との関連

*   **PKI (3.2.c)**: デバイス証明書のインポートと信頼関係の構築が前提となります。
*   **Cisco ISE (2.2)**: ユーザー認証および動的ポリシー（SGT 割り当て等）のプッシュに使用されます。
*   **ZBF (Zone-Based Firewall)**: Virtual-Template をセキュリティゾーンに含める必要があります。

---

## 🧩 比較表

### IOS AnyConnect (IKEv2) vs ASA AnyConnect (SSL)

| 特徴 | IOS FlexVPN AnyConnect | ASA AnyConnect VPN |
| :--- | :--- | :--- |
| **主要プロトコル** | IKEv2 (UDP) | SSL/TLS (TCP) & IKEv2 |
| **設定体系** | FlexVPN (ポリシーベース) | Tunnel-Group / Group-Policy |
| **動的ルーティング** | フルサポート (OSPF/EIGRP/BGP) | 限定的 |
| **透過性** | ファイアウォールで UDP ブロックの懸念 | 高い（HTTPS 443 のため） |

---

## 💡 ベストプラクティス

1.  **AnyConnect プロファイルの使用**: サーバーリストやバックアップ GW を含む XML プロファイルを適切に配布します。
2.  **証明書認証の活用**: ユーザー名/パスワードへの依存を減らすため、マシン証明書認証を推奨します。
3.  **Dead Peer Detection (DPD)**: クライアントの切断を迅速に検知するため、キープアライブ設定を最適化します。
4.  **Loopback 参照**: `Virtual-Template` の IP アドレスには常に `Loopback` を使用し、物理インターフェイスの状態に依存しないようにします。

---

## 📝 ラボ学習・設定サンプル例

### 1. EAP 認証による AnyConnect 収容
*   **要件**: AnyConnect ユーザーを Local DB で認証し、10.1.1.0/24 の IP を配れ。
*   **設定**: `crypto ikev2 profile` 内で `aaa authentication login default` を使用。

### 2. 証明書認証（パスワードレス）
*   **要件**: 証明書のみで接続を許可せよ。
*   **設定**: `authentication remote rsa-sig`, `authentication local rsa-sig`。

### 3. スプリットトンネルの設定
*   **要件**: 内部 192.168.1.0/24 への通信のみ VPN を通せ。
*   **設定**: `crypto ikev2 authorization policy` 内で `route set access-list` を適用。

### 4. Virtual-Template の番号管理
*   **課題**: 複数の RA プロファイルがある場合、個別の Virtual-Template を割り当て分ける。

### 5. ISE 経由の SGT 付与
*   **要件**: 接続ユーザーに SGT タグ 10 を付与せよ。
*   **設定**: RADIUS VSA で `cts:security-group-tag=000A` を送信。

### 6. IPv6 AnyConnect 接続
*   **要件**: IPv6 外側インターフェイスで AnyConnect を待ち受けよ。

### 7. ローカル ACL によるアクセス制限
*   **要件**: VPN ユーザーからサーバー 10.1.1.10 への SSH を禁止せよ。
*   **設定**: Virtual-Template に `ip access-group` を適用。

### 8. 最大セッション数制限
*   **要件**: 同一ユーザーの同時接続を 1 に制限せよ。

### 9. DNS情報のプッシュ
*   **要件**: クライアントに DNS 8.8.8.8 を通知せよ。
*   **設定**: `crypto ikev2 authorization policy` 内で `nameserver 8.8.8.8`。

### 10. トンネル内動的ルーティング
*   **要件**: AnyConnect ユーザーとの間で OSPF を動作させよ。
*   **設定**: Virtual-Template を `ip ospf 1 area 0` に含める。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: `show crypto ikev2 sa` でセッションは見えるが、AnyConnect クライアント側で「Secure Gateway への接続に失敗」と出る。原因は？
    *   **回答**: ルータ側で `authentication remote anyconnect-eap` が抜けており、クライアントが必要とする認証フローを完結できていない。
2.  **Design**: ルータで AnyConnect を提供する際、SSL ではなく IKEv2 を使用する最大のメリットは何か？
    *   **回答**: パフォーマンス（IPsec のハードウェアアクセラレーション）と、トンネル上での動的ルーティングプロトコルのネイティブサポート。
3.  **実装**: 特定の AD グループに属するユーザーのみに特定の VPN プール IP を割り当てるには？
    *   **回答**: ISE の認証ポリシーに基づき、RADIUS 属性 `Framed-IP-Address` または `Cisco-AV-Pair` でプール名をルータへ返す。
4.  **コンフィグ読解**: `match identity remote any` が `crypto ikev2 profile` にある場合のリスクを述べよ。
    *   **回答**: すべてのリモート ID を受け入れるため、適切な認証（証明書や EAP）が設定されていない場合、なりすまし接続を許容する可能性がある。
5.  **Design**: ルータのリモートアクセス VPN でスプリットトンネルを実装する際、設定箇所はどこか？
    *   **回答**: `crypto ikev2 authorization policy` オブジェクト内。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [FlexVPN Remote Access](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_conn_ikeev2/configuration/xe-16/sec-flex-vpn-xe-16-book.html)
*   **Cisco Live (BRKSEC-3033)**
    *   [Advanced AnyConnect Implementation and Troubleshooting](https://www.ciscolive.com/)
*   **Technical Notes**
    *   [AnyConnect IKEv2 Client to IOS Router Configuration Example](https://www.cisco.com/c/en/us/support/docs/security/anyconnect-secure-mobility-client/115947-anyconnect-ikev2-router-config.html)

---

## 📝 **補足（Notes）**  

*   **学習メモ**: ルータでの AnyConnect は「ASA とは全く別の設定体系（FlexVPN）」であることを認識してください。
*   **図解**: `Virtual-Template` が親、`Virtual-Access` が子。セッションごとに子がクローンされるイメージです。
*   **注意点**: ラボ試験では、ルータの自己署名証明書（Self-signed）の有効期限や、クライアント PC との時刻同期（NTP）の不一致による接続エラーに十分注意してください。
