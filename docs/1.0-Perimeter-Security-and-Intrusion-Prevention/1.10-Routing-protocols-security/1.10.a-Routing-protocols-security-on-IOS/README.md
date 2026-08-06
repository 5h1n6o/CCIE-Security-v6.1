---
layout: default
title: 1.10.a Routing protocols security on IOS
nav_order: 1
parent: 1.10-Routing-protocols-security
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.10.a Routing protocols security on Cisco IOS

Cisco IOS におけるルーティングプロトコルのセキュリティは、ネットワークの**制御プレーン（Control Plane）**を保護し、インフラ全体の安定性と信頼性を維持するために不可欠です。不正なルート情報の注入、ネイバー関係のハイジャック、または大量の経路情報によるリソース枯渇（DoS）を防ぐため、認証、フィルタリング、およびプロトコル固有の制限を実装します。

---

## 📘 概要

*   **機能概要**: ルーティングプロトコル（OSPF, EIGRP, BGP, RIP）の通信における送信元の正当性確認（認証）と、送受信される経路情報の制御（フィルタリング）。
*   **利用目的**: 不正なルータによる経路の乗っ取り防止、ルートループの回避、特定のセグメントへの到達性制御。
*   **利用場面**: インターネット境界での BGP ピア保護、組織内の部門間でのルート配布制限、VPN（DMVPN 等）環境でのネイバー関係の維持。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な認証方式** | MD5 (標準), HMAC-SHA-256 (モダン/EIGRP Named mode)。 |
| **フィルタリング** | Prefix-lists, Access-lists, Route-maps, Distribute-lists。 |
| **ピア保護** | BGP TTL Security (GTSM), Neighbor Maximum-prefix。 |
| **対応機種** | IOS-XE (ISR/ASR/Catalyst), CSR1000v。 |
| **主要要素** | Key-chains, Authentication Modes, Passive Interfaces。 |
| **設計上の注意** | 認証キーの不一致はネイバー断を招くため、移行時は Key-chain の使用を推奨。 |

---

## 🏗 動作原理

ルーティングプロトコルのセキュリティは、「誰と話すか（Authentication）」と「何を話すか（Filtering）」の二段階で動作します。

```text
[ Routing Packet ]
       ↓
[ 1. Receive Check (ACL/CoPP) ] --- CPU保護レイヤ
       ↓
[ 2. Authentication (MD5/SHA) ] --- 送信元検証
       ↓
[ 3. Hello/Adjacency Process ] --- パラメータ一致確認
       ↓
[ 4. Route Filtering (Inbound) ] --- 経路の選別
       ↓
[ 5. Routing Table (RIB) ] --- 正当な経路のみを格納
```

---

## ⚙ 動作シーケンス

1.  **認証の実行**: Hello パケットの交換時にハッシュ値（MD5/SHA）を比較し、一致しなければパケットを破棄。
2.  **ネイバー確立**: 認証成功後、OSPF の FULL 状態や BGP の Established 状態へ移行。
3.  **フィルタリングの適用**: アップデート情報を受信した際、設定された `distribute-list` 等に基づき、許可されたプレフィックスのみを取り込む。
4.  **再配布の制御**: `redistribute` 実行時に `route-map` を通じて、タグ付けやメトリック調整を行い、不正な再配布を阻止。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **EIGRP SHA-256 (Named Mode)**: クラシックモード（MD5）ではなく、`router eigrp [NAME]` 形式の Named Mode で HMAC-SHA-256 を設定させる問題が頻出です。
*   **Key-chain の活用**: Key-ID ごとに有効期限（Accept/Send lifetime）を設定し、キーのローテーションを無停止で行う構成。
*   **BGP TTL Security**: ネイバーからのホップ数を制限することで、遠隔からの攻撃を防ぐ設定。
*   **DMVPN とルーティング**: `ip nhrp redirect/shortcut` (Phase 3) 環境下での OSPF ネットワークタイプ（Point-to-Multipoint）設定と認証の組み合わせ。
*   **CoPP との連携**: ルーティングパケットを `control-plane` 下でレートリミットし、DoS 攻撃から保護する設定。

---

## 🛠 設定方法

### 1. OSPF インターフェイス認証 (MD5)
```bash
interface GigabitEthernet1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CISCO123
```

### 2. EIGRP Named Mode HMAC-SHA-256 認証
```bash
router eigrp CCIE_LAB
 address-family ipv4 autonomous-system 100
  af-interface default
   authentication mode hmac-sha-256 PASS123
  exit-af-interface
```

### 3. Key-chain によるローテーション
```bash
key chain ROUTE-KEY
 key 1
  key-string KEY-A
  accept-lifetime 00:00:00 Jan 1 2024 infinite
  send-lifetime 00:00:00 Jan 1 2024 00:00:00 Jul 1 2024
 key 2
  key-string KEY-B
  send-lifetime 23:55:00 Jun 30 2024 infinite
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **ネイバーの状態確認** | <code>show ip [ospf\|eigrp\|bgp] neighbors</code> |
| **認証設定の確認** | <code>show ip [ospf\|eigrp] interface</code> |
| **BGPセッション詳細** | <code>show ip bgp neighbors</code> |
| **認証エラーのデバッグ** | <code>debug ip ospf adj</code> / <code>debug eigrp packets</code> |
| **CoPPでのドロップ確認** | <code>show policy-map control-plane</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| ネイバーが形成されない | Key-ID またはパスワードの不一致 | <code>show run interface</code> で両端のキー番号を確認。 |
| OSPF が 2-WAY で止まる | 認証タイプのミスマッチ (明文 vs MD5) | エリア全体または IF の認証設定を統一。 |
| 経路が学習されない | <code>distribute-list</code> で拒否されている | 適用されている ACL/Prefix-list の内容を確認。 |
| CPU負荷が高い | 大量のルーティングトラフィック | CoPP でルーティングプロトコルをレートリミット。 |

---

## ⚠ 制限事項

*   **MD5 の限界**: セキュリティ強度の観点から、可能であれば SHA 認証をサポートするプロトコル/モード（EIGRP Named mode 等）を使用してください。
*   **DMVPN Phase 2/3**: OSPF のマルチキャストハローは NHRP マッピングに依存するため、認証を有効にする前に疎通確認が必要です。

---

## 🔄 他技術との関連

*   **Control Plane Policing (CoPP)**: ルーティングパケット自体を CPU 宛の優先トラフィックとして分類・保護します。
*   **DMVPN**: `ip nhrp map multicast` によりルーティングプロトコルのハローパケットを配送します。
*   **VRF-Lite**: 各 VRF インスタンスごとに独立したルーティング認証とフィルタリングを適用します。

---

## 🧩 比較表

### IOS ルーティング認証の比較

| 特徴 | OSPFv2 | EIGRP (Classic) | EIGRP (Named) | BGP |
| :--- | :--- | :--- | :--- | :--- |
| **デフォルト方式** | なし | なし | なし | なし |
| **MD5 サポート** | あり | サポート | サポート | 標準サポート |
| **HMAC-SHA サポート** | RFC 5709 (限定的) | なし | **HMAC-SHA-256** | サポート |
| **Key-chain 連携** | 一部対応 | 標準 | 標準 | なし |

---

## 💡 ベストプラクティス

1.  **Passive Interface**: ユーザが接続するポートでは `passive-interface default` を使用し、意図しないネイバー形成を防ぎます。
2.  **Prefix-list の優先使用**: ACL よりも処理が高速で、範囲（`ge/le`）指定が可能な Prefix-list をフィルタリングに使用します。
3.  **BGP Max-prefix**: ピアから受け取る経路数に上限を設け、メモリ枯渇を防止します。
4.  **BGP GTSM**: 直接接続されたピアに対して TTL=255 を期待し、インターネット越しのなりすましを防ぎます。

---

## 📝 ラボ学習・設定サンプル例

### 1. EIGRP HMAC-SHA-256 の実装
*   **要件**: Named Mode "CCIE" を作成し、AS 100 で SHA 認証を設定せよ。
*   **設定**: `router eigrp CCIE` -> `address-family ipv4 as 100` -> `af-interface default` -> `authentication mode hmac-sha-256 CISCO`.

### 2. OSPF エリア認証
*   **要件**: エリア 0 全体で MD5 認証を有効にせよ。
*   **設定**: `router ospf 1` -> `area 0 authentication message-digest`. インターフェイス側でキーを設定。

### 3. BGP Neighbor Password
*   **要件**: ネイバー 10.1.1.2 との間に MD5 パスワードを設定せよ。
*   **設定**: `router bgp 65000` -> `neighbor 10.1.1.2 password CISCO123`.

### 4. Distribute-list によるルート制限
*   **要件**: `10.10.10.0/24` 以外の経路受信を拒否せよ。
*   **設定**: `ip prefix-list ONLY10 permit 10.10.10.0/24`, `router ospf 1` -> `distribute-list prefix ONLY10 in`.

### 5. OSPF Passive Interface の設定
*   **要件**: ルータ R1 の G0/1 ポートで Hello 送出を停止せよ。
*   **設定**: `router ospf 1`, `passive-interface GigabitEthernet0/1`.

### 6. BGP TTL Security (GTSM)
*   **要件**: EBGP ネイバーが直接接続されていることを保証せよ。
*   **設定**: `neighbor 1.1.1.1 ttl-security hops 1`.

### 7. Key-chain Send/Accept Lifetime
*   **要件**: 移行期間に 1 時間のオーバーラップを持つキーを設定せよ。
*   **設定**: 各キーに `lifetime` を適切に設定。

### 8. Redistribution フィルタリング
*   **要件**: RIP 経路を OSPF に入れる際、タグ 100 のみ許可せよ。
*   **設定**: `route-map FILT permit 10` -> `match tag 100`, `router ospf 1` -> `redistribute rip route-map FILT`.

### 9. BGP Maximum-Prefix
*   **要件**: ネイバーから 1000 以上の経路を受け取ったら警告せよ。
*   **設定**: `neighbor 1.1.1.1 maximum-prefix 1000 80`.

### 10. CoPP によるルーティング保護
*   **要件**: BGP (179) へのパケットを 256Kbps に制限せよ。
*   **設定**: `access-list 101 permit tcp any any eq 179` -> `class-map` -> `policy-map` -> `control-plane` 適用。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `ip ospf message-digest-key 1 md5 AAA` と `ip ospf message-digest-key 2 md5 BBB` が同一 IF にある場合の動作を説明せよ。
    *   **回答**: キーの移行（Rollover）状態。送信パケットには両方のハッシュが（あるいは最新が）含まれ、受信は両方を許可する。
2.  **トラブルシュート**: EIGRP Named mode で認証を設定したが、ネイバーが UP しない。確認すべき EIGRP 固有の項目は？
    *   **回答**: `address-family` 下の `autonomous-system` 番号が一致しているか、および `af-interface` で認証モードが正しく適用されているか。
3.  **Design**: DMVPN Phase 3 環境で、ハブからスポークへ効率的にサマリルートを送るための OSPF の推奨設定は？
    *   **回答**: `ip ospf network point-to-multipoint` を使用し、DR/BDR 選出を回避しつつ、Next-hop をハブに維持する。
4.  **実装**: 特定の BGP ネイバーから不正なプライベート IP アドレス（RFC1918）が広報されないようにする Prefix-list を作成せよ。
5.  **Design**: ルータの CPU をルーティングアップデートの Dos 攻撃から守るために、データプレーンではなくコントロールプレーンで適用すべき機能は？
    *   **回答**: Control Plane Policing (CoPP)。

---

## 🔗 参考リソース

*   [Cisco IOS-XE Routing Configuration Guide: Securing OSPF](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/configuration/xe-16/iro-xe-16-book/iro-sec-ospf.html)
*   [Cisco White Paper: EIGRP HMAC-SHA-256 Authentication](https://www.cisco.com/c/en/us/support/docs/ip/enhanced-interior-gateway-routing-protocol-eigrp/116246-technote-eigrp-00.html)
*   [Cisco Live: BRKSEC-2003 - Control Plane Protection Best Practices](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: ルーティングプロトコルのセキュリティは、単一のコマンドではなく「Key-chain -> Address-family -> Interface」といった多階層の依存関係を意識することが重要です。
*   **注意点**: ラボ試験では、認証を有効にした瞬間に既存のネイバーが落ちるため、タスクの順序（両端を迅速に設定する等）に注意してください。
