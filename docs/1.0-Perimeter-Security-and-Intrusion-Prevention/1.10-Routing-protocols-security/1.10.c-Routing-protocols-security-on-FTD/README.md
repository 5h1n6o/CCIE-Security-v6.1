---
layout: default
title: 1.10.c Routing protocols security on FTD
nav_order: 3
parent: 1.10-Routing-protocols-security
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.10 Routing protocols security on Cisco FTD

Cisco Firepower Threat Defense (FTD) における**ルーティングプロトコルのセキュリティ**は、ネットワークのコントロールプレーンを保護し、不正なルート情報の注入や隣接関係のハイジャックを防止するために不可欠です。FTD は内部的に LINA エンジン（ASA ベースのコード）を使用してルーティングを処理しており、OSPF、BGP、RIP は FMC GUI からネイティブに設定可能ですが、EIGRP などの一部の機能は **FlexConfig** を介した実装が必要となります。

---

## 📘 概要

*   **機能概要**: ルーティングプロトコル（OSPF, BGP, RIP, EIGRP）のメッセージ交換において、認証（Authentication）とルートフィルタリング（Filtering）を適用し、信頼されたピア間でのみ正しい経路情報を共有します。
*   **利用目的**: 不正なルーターによるトラフィックの引き込み（ブラックホール化や中間者攻撃）の阻止、およびルーティングアップデートによるリソース枯渇の防止。
*   **どのような場面で利用するか**: 境界 FW として ISP と BGP を組む際や、内部コアスイッチと OSPF で動的ルーティングを行うエンタープライズ環境。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **サポートプロトコル** | OSPF (v2/v3), BGP (IPv4/IPv6), RIP (IPv4), EIGRP (FlexConfig経由)。 |
| **主な認証方式** | MD5 (標準), SHA (プロトコルによる), BGP パスワード。 |
| **設定管理** | **Firepower Management Center (FMC)** による一元管理。 |
| **フィルタリング** | Prefix-lists, Route-maps を使用した柔軟な制御。 |
| **ルーティングの性質** | ASA と同様の動作（AD, メトリック, 管理/データの RIB 分離）。 |
| **設計上の注意点** | 制御パケットが FTD 自身（Identity インターフェイス）宛であることを ACL で考慮。 |

---

## 🏗 動作原理

FTD はパケットを受信した際、それが自身宛（To-the-box）のルーティングプロトコルパケットであるかを判別します。

```text
Neighbor Router
   ↓ (Hello / Update Packet)
FTD Physical Interface
   ↓
[ Platform/Identity ACL ] <--- プロトコル番号 (OSPF:89, BGP:179) の許可が必要
   ↓
[ Authentication Engine ] <--- Key/Password の照合
   ↓
[ LINA Routing Process ]  <--- 経路情報の評価 (Route-map/Prefix-list)
   ↓
[ Routing Table (RIB) ]   <--- 最適経路の格納
```

---

## ⚙ 動作シーケンス

1.  **ネイバー検出**: インターフェイスで Hello パケットを受信。
2.  **認証ネゴシエーション**: 設定された認証キー（MD5 等）を照合。一致しない場合はパケットを破棄しログを生成。
3.  **隣接関係 (Adjacency) 確立**: 認証成功後、Full または Established 状態へ移行。
4.  **ルート情報の受信とフィルタリング**: アップデート情報に対し、インバウンドの Prefix-list 等を適用してフィルタリング。
5.  **情報の配布 (Redistribution)**: 必要に応じて他プロトコルへ経路を渡す際、Route-map でメトリックやタグを制御。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **FMC での OSPF 認証**: `Devices > Device Management > Routing > OSPF` のインターフェイス設定で、MD5 認証を確実に有効化する手順が重要です。
*   **BGP Neighbor Password**: ISP 接続シナリオにおいて、ネイバーごとの MD5 パスワード設定が頻出です。
*   **EIGRP via FlexConfig**: FTD 7.x 以降でも EIGRP は GUI サポートが限定的（またはなし）であるため、**FlexConfig オブジェクト**を使用して ASA ライクな CLI を流し込む能力が試されます。
*   **Passive Interface の適用**: ユーザーセグメントなど、ネイバーを形成すべきでないインターフェイスでの Hello 送出停止要件。
*   **トラブルシュート**: ネイバーが Established にならない原因を、FTD CLI (`system support diagnostic-cli`) から `show ospf neighbor` 等を実行して特定するスキル。

---

## 🛠 設定方法

### 1. OSPF MD5 認証 (FMC GUI)
1.  **Devices > Device Management** で対象 FTD を編集。
2.  **Routing > OSPF > Interface** タブへ移動。
3.  対象インターフェイスを選択し、**Authentication** を `Message Digest` に設定。
4.  **Key ID** と **Key**（パスワード）を入力。

### 2. BGP ピアパスワード設定 (FMC GUI)
1.  **Routing > BGP > Neighbor** を選択。
2.  ネイバーの追加/編集画面で、**Authentication Key** フィールドにパスワードを入力。

### 3. EIGRP 認証 (FlexConfig 例)
```bash
! FlexConfig Object
router eigrp 100
 address-family ipv4 autonomous-system 100
  af-interface GigabitEthernet0/0
   authentication mode md5
   authentication key CISCO123
```

---

## 🔍 検証コマンド

FTD のルーティング状態を確認するには、診断用 CLI に入る必要があります。

| 目的 | コマンド (FTD 診断 CLI) |
| :--- | :--- |
| **OSPF ネイバー確認** | <code>show ospf neighbor</code> |
| **BGP ネイバー確認** | <code>show bgp summary</code> |
| **EIGRP 状態確認** | <code>show eigrp neighbors</code> |
| **ルーティングテーブル表示** | <code>show route</code> |
| **認証ミスのデバッグ** | <code>debug ospf adj</code> / <code>debug bgp updates</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| ネイバーが形成されない | インターフェイス ACL の遮断 | FTD のインターフェイス ACL で OSPF/BGP を許可しているか確認。 |
| 認証失敗 (Auth mismatch) | キーまたは ID の不一致 | 両端で <code>Key-ID</code> と <code>Password</code> が完全に一致しているか確認。 |
| 経路が学習されない | フィルタリングルールの誤り | 適用している <code>Prefix-list</code> や <code>Route-map</code> の Permit/Deny ロジックを確認。 |
| EIGRP が動かない | FlexConfig のデプロイミス | FMC のデプロイ履歴を確認し、CLI が正しく LINA へ反映されているか確認。 |

---

## ⚠ 制限事項

*   **EIGRP GUI サポート**: 現行バージョンでも EIGRP の詳細なセキュリティ設定は FlexConfig に依存する場合が多い。
*   **マルチインスタンス**: 上位機種（4100/9300）のマルチインスタンス環境では、インスタンス間のルーティングリソース共有に注意が必要。
*   **暗号化**: ルーティングパケット自体を SSL/IPsec で暗号化することは標準では行われず（認証のみ）、必要に応じて VTI VPN 等との併用が必要。

---

## 🔄 他技術との関連

*   **Access Control Policy (ACP)**: FTD を通過するトラフィックに影響しますが、FTD 自身へのルーティングパケットはプラットフォーム ACL 等で制御されます。
*   **Virtual Tunnel Interface (VTI)**: VPN トンネル上で動的ルーティングを走らせる際、認証設定が必須となります。
*   **High Availability (HA)**: Active/Standby 構成では、ルーティングの状態は同期されますが、切り替え時にピアリングの再確立が発生する場合があります。

---

## 🧩 比較表

### FTD におけるルーティングプロトコル管理

| 機能 | OSPF / BGP / RIP | EIGRP |
| :--- | :--- | :--- |
| **FMC GUI 設定** | **フルサポート** | **FlexConfig 推奨** |
| **認証設定の容易さ** | 高い (チェックボックス) | 低い (CLI 記述) |
| **CCIE 試験での重要度** | 非常に高い | 高い (FlexConfig 枠として) |

---

## 💡 ベストプラクティス

1.  **Passive Interface の徹底**: ピアが存在しない全インターフェイスで `passive-interface default` を適用します。
2.  **MD5 以上の認証使用**: 平文認証は避け、OSPF/BGP では必ず MD5 またはパスワード認証を適用します。
3.  **Prefix-list の活用**: ルートフィルタリングには ACL ではなく、処理効率の良い Prefix-list を使用します。
4.  **BGP Max-Prefix**: ネイバーから受け取る経路数に上限を設け、予期せぬ大量経路によるメモリ枯渇を防ぎます。

---

## 📝 ラボ学習・設定サンプル例

### 1. OSPFv2 エリア認証の強制
*   **要件**: Area 0 の全ネイバーに対して MD5 認証を強制せよ。
*   **設定 (FMC)**: Routing > OSPF > Area タブで Area 0 を選択、`Authentication: Message Digest`。

### 2. BGP 外部ピアの保護
*   **要件**: AS 65001 のネイバー (1.1.1.1) に対しパスワード "SECURE" を設定せよ。
*   **設定 (FMC)**: Routing > BGP > Neighbor > Authentication Key に入力。

### 3. EIGRP 認証の FlexConfig 実装
*   **要件**: FlexConfig を使用して EIGRP AS 10 の認証を設定せよ。
*   **設定**: FlexConfig Object 内で `authentication mode eigrp 10 md5` ... を記述。

### 4. 経路配布の制限 (Route-map)
*   **要件**: タグ 100 が付いた経路のみを再配布せよ。
*   **設定**: FMC オブジェクトで Route-map を作成し、Redistribution 設定で適用。

### 5. IPv6 BGP 認証
*   **要件**: IPv6 BGP ネイバーとの間で認証を設定せよ。

### 6. OSPF Passive Interface 設定
*   **要件**: Inside VLAN 以外への Hello 送出を停止せよ。
*   **設定**: FMC > Routing > OSPF > `Passive Interface: default` ＋ Inside のみを例外指定。

### 7. BGP TTL Security (GTSM)
*   **要件**: 直結ネイバー以外からの BGP 偽装を防げ。

### 8. ルーティング用プラットフォーム ACL
*   **要件**: Outside インターフェイスで OSPF プロトコルを明示的に許可せよ。

### 9. OSPF MTU Ignore
*   **課題**: MTU 不一致でネイバーが確立しない。
*   **設定**: FMC > Interface 設定で `MTU Ignore` をチェック。

### 10. CLI による状態デバッグ
*   **要件**: CLI から OSPF 認証エラーをリアルタイムで追跡せよ。
*   **実行**: `system support diagnostic-cli` -> `debug ospf adj`。

---

## ❓ 想定試験問題

1.  **実装**: FTD において EIGRP を有効にするための FMC 上のコンポーネントを答えよ。
    *   **回答**: FlexConfig。
2.  **トラブルシュート**: FMC で OSPF 認証を設定したが、隣接関係が `EXSTART` で止まっている。考えられる原因は？
    *   **回答**: MTU サイズのミスマッチ。または認証設定が片方のみに適用されている。
3.  **コンフィグ読解**: `show route` で `O*E2` と表示されている経路の意味を説明せよ。
    *   **回答**: OSPF 外部タイプ 2 経路。
4.  **Design**: BGP ネイバーから 5000 以上のプレフィックスを受信した場合にセッションを切断する設定はどこで行うか？
    *   **回答**: FMC の BGP ネイバー設定内、`Maximum Prefix` フィールド。
5.  **トラブルシュート**: FTD の診断 CLI で `show ospf neighbor` が何も出力されない。最初に確認すべきインターフェイスの状態は？
    *   **回答**: 物理的な `up/up` 状態および、`ip address` が正しく割り当てられているか。

---

## 🔗 参考リソース

*   **Cisco Configuration Guides**
    *   [Cisco Secure Firewall Management Center Administration Guide, 7.2 - Routing](https://www.cisco.com/c/en/us/td/docs/security/firepower/720/configuration/guide/fpmc-config-guide-v72/routing_overview.html)
*   **Technical Notes**
    *   [Configure EIGRP on Firepower Threat Defense via FlexConfig](https://www.cisco.com/c/en/us/support/docs/security/firepower-ngfw/212512-configure-eigrp-on-ftd-via-flexconfig.html)
*   **Cisco Live (BRKSEC-3020)**
    *   [Troubleshooting Firepower Policies and Routing](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: FTD のルーティングは「FMC での GUI 操作」が基本ですが、トラブルシュートは「FTD CLI での診断」が基本です。両方のインターフェイスをスムーズに使いこなせるようにしましょう。
*   **注意点**: CCIE ラボ試験では、**FlexConfig** の使用が必要な箇所と、GUI で完結できる箇所を瞬時に判断することが時間短縮の鍵となります。EIGRP は FlexConfig であることを覚えておきましょう。
