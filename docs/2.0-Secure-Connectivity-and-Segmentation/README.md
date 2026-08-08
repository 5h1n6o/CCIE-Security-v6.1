---
layout: default
parent: CCIE Security v6.1
title: 2.0-Secure-Connectivity-and-Segmentation
nav_order: 2
---

# 2.6 Microsegmentation with Cisco TrustSec using SGT and SXP

Cisco TrustSec (CTS) は、IPアドレスやトポロジーに依存せず、**SGT（Security Group Tag）** という論理的なタグを使用してネットワーク全体のセキュリティポリシーを定義・適用するフレームワークです。CCIE Security v6.1 においては、ISE (Identity Services Engine) と連携したマイクロセグメンテーションの実装、および SGT を転送するための **SXP (SGT Exchange Protocol)** の設定とトラブルシューティングが最重要項目となります。

---

## 📘 概要

*   **機能概要**: ユーザーやデバイスに 16 ビットの SGT を付与し、ネットワーク機器がそのタグを基に **SGACL (Security Group ACL)** を用いてアクセス制御を行います。
*   **利用目的**: ネットワークトポロジー（VLAN やサブネット）を変更することなく、柔軟なマイクロセグメンテーションを実現します。
*   **利用場面**:
    *   **キャンパスネットワーク**: 部門や役割（従業員、ゲスト、IoT デバイス）に基づいた分離。
    *   **データセンター**: アプリケーションの層（Web, App, DB）ごとの隔離。
    *   **クラウド移行**: 異なる拠点間での一貫したポリシー適用。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **分類 (Classification)** | IP、VLAN、または 802.1X 認証の結果に基づいて SGT を割り当てるプロセス。 |
| **伝搬 (Propagation)** | SGT 情報を対向デバイスに伝える手法（Inline Tagging または SXP）。 |
| **適用 (Enforcement)** | 宛先（Egress）デバイスで SGT を基に通信を許可/拒否するプロセス (SGACL)。 |
| **SXP** | SGT 対応デバイスと非対応デバイスの間で IP-to-SGT マッピングを転送するプロトコル (TCP 649)。 |
| **SGT 範囲** | 1 ～ 65535 (0 はタグなし/Unknown 用に予約)。 |
| **設計上の注意点** | エンフォースメントは常に「出力（Egress）」インターフェイスで行われる。 |

---

## 🏗 動作原理

TrustSec は「分類」「伝搬」「適用」の 3 フェーズで動作します。

```text
[ Ingress Node ] -------- [ Intermediate Nodes ] -------- [ Egress Node ]
      ↓                         ↓                             ↓
1. 分類 (Mapping)        2. 伝搬 (Transport)           3. 適用 (Enforcement)
(ISE/Static/802.1X)     (SXP or Inline CMD)           (SGACL check)
      ↓                         ↓                             ↓
[ SGT assigned ] ----> [ Tag carried in Header ] ----> [ Permitted/Denied ]
```

---

## ⚙ 動作シーケンス

1.  **SGT 割り当て**: ユーザーが認証される際、ISE は属性（AD グループ等）に基づき SGT を決定し、スイッチに通知します。
2.  **タグの挿入 (Inline)**: パケットがレイヤ 2 フレームの 802.1Q タグと EtherType の間に挿入される **Cisco Meta Data (CMD)** フィールドに SGT が書き込まれます。
3.  **マッピング転送 (SXP)**: Inline Tagging が不可能なデバイス間では、SXP ピア（Speaker/Listener）が TCP セッション越しに IP-SGT マッピングデータベースを同期します。
4.  **ポリシーダウンロード**: 適用ポイント（Egress スイッチ）は、ISE から SGACL ポリシーをダウンロードします。
5.  **パケットフィルタリング**: 送信元 SGT と宛先 SGT の組み合わせをマトリックス（Matrix）に照らし合わせ、SGACL でアクションを決定します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **SXP の Speaker/Listener 設定**: どちらが情報を送り (Speaker)、どちらが受けるか (Listener) を正確に把握してください。双方向通信が必要な場合は「Both」を設定します。
*   **SXP MD5 認証**: ピア間でのパスワード一致は必須です。
*   **Static IP-SGT Mapping**: 認証を通らないサーバー等に手動でタグを付与する設定 (`cts role-based entry`)。
*   **Enforcement の有効化**: グローバルおよびインターフェイスレベルで `cts role-based enforcement` を有効にしないと、SGACL は機能しません。
*   **ISE との同期**: ISE 側での Matrix 設定と、デバイス側での `show cts environment-data` による同期確認。
*   **ASA/FTD の SGT 対応**: ASA では SGT をオブジェクトとして扱い、ACL に組み込む独特の設定フローがあります。

---

## 🛠 設定方法

### 1. IOS-XE: SXP ピアの設定 (Speaker)
```bash
cts sxp enable
cts sxp default source-ip 10.1.1.1
cts sxp default password cisco123
! Listener (10.1.1.2) に対して情報を送る
cts sxp peer 10.1.1.2 password default mode local speaker
```

### 2. IOS-XE: 静的 IP-SGT マッピング
```bash
cts role-based entry 10.1.10.100 sgt 10
cts role-based entry 10.1.20.200 sgt 20
```

### 3. IOS-XE: SGACL の適用
```bash
! 10から20へのICMPを拒否
ip access-list role-based DENY-ICMP
 deny icmp
! ポリシーの定義
cts role-based policy from 10 to 20
 access-list DENY-ICMP
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **SXP ピアの状態確認** | <code>show cts sxp connections brief</code> |
| **IP-SGT マッピングの確認** | <code>show cts role-based sgt-map all</code> |
| **SGACL ポリシーの確認** | <code>show cts role-based policy</code> |
| **環境データの同期確認** | <code>show cts environment-data</code> |
| **パケットドロップの統計** | <code>show access-lists role-based</code> |
| **SXP の詳細デバッグ** | <code>debug cts sxp messages</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| SXP ピアが <code>Off</code> | パスワード不一致、IP 到達性なし | ピア間の <code>ping</code> と <code>password</code> を再確認。 |
| SGT がパケットに乗らない | 経路上のデバイスが Inline 非対応 | SXP を使用してマッピングを飛ばす設計に変更。 |
| SGACL が効かない | Enforcement が無効 | <code>cts role-based enforcement</code> が設定されているか確認。 |
| ISE からデータが落ちてこない | AAA/RADIUS の不備 | <code>cts credentials</code> と ISE の信頼関係を確認。 |

---

## ⚠ 制限事項

*   **ハードウェア依存**: Inline Tagging は特定の ASIC（Cisco スイッチ）でのみサポートされます。ルータのサブインターフェイス等では制限がある場合があります。
*   **MTU**: CMD ヘッダーの挿入によりフレームサイズが 40 バイト増加するため、MTU 調整が必要になるケースがあります。
*   **SXP ホップ数**: SXP 経由での転送は CPU 処理になるため、大規模環境では負荷に注意が必要です。

---

## 🔄 他技術との関連

*   **ISE (2.1/2.2)**: TrustSec ポリシーの「脳」であり、マトリックス管理を行います。
*   **802.1X (2.1.a)**: ダイナミックな SGT 割り当ての主要なトリガーです。
*   **ASA/FTD (1.1)**: ファイアウォールとして SGT 属性に基づいた L3/L4 フィルタリングを実施します。
*   **SD-Access**: TrustSec は DNA Center/SD-Access におけるセグメンテーションの基盤技術です。

---

## 🧩 比較表

### Inline Tagging vs SXP

| 特徴 | Inline Tagging (CMD) | SXP (SGT Exchange Protocol) |
| :--- | :--- | :--- |
| **データプレーン** | パケットヘッダーにタグを埋め込む | TCP セッションによるコントロールプレーン |
| **スループット** | ハードウェア処理（高速） | CPU 処理が介在する場合あり |
| **ハードウェア要件** | 全ホップが対応している必要あり | 非対応デバイス（他社製等）をスキップ可能 |
| **推奨環境** | 最新の Catalyst スイッチ間 | レガシー混在環境、ASA との連携 |

---

## 💡 ベストプラクティス

1.  **ISE による一元管理**: 手動設定を避け、ISE で SGT-to-Group マッピングを一括管理します。
2.  **Default Action の検討**: マトリックスに定義がない通信を許可するか拒否するか (Default Permit/Deny) を慎重に設計します。
3.  **SXP 冗長化**: 主要な SXP ノード間ではピアを冗長化し、マッピング情報の喪失を防ぎます。
4.  **Unknown SGT の監視**: 予期せぬデバイスが SGT 0 で通信していないか定期的に <code>show cts role-based sgt-map</code> で監視します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な SXP Speaker/Listener の構築
*   **要件**: SW1(Speaker) から SW2(Listener) へタグ情報を転送せよ。
*   **設定**: `cts sxp enable` -> `cts sxp peer ... mode local speaker/listener`。

### 2. 静的マッピングによるサーバー分離
*   **要件**: 10.1.1.100 に SGT 5、10.1.1.200 に SGT 10 を付与せよ。

### 3. SGACL による特定通信の拒否
*   **要件**: SGT 2 から SGT 3 への Telnet (TCP 23) をブロックせよ。

### 4. 802.1X 連携による動的 SGT 付与
*   **要件**: ISE で認証された PC に SGT 15 を割り当てよ。

### 5. SXP 認証の設定
*   **要件**: SXP ピア間で MD5 認証を強制せよ。
*   **設定**: `cts sxp default password KEY`。

### 6. ASA における SGT enforcement の実装
*   **要件**: ASA を SXP Listener とし、SGT ベースの ACL を適用せよ。

### 7. マルチホップ SXP の検証
*   **課題**: 中間スイッチが SXP をリレーする構成を組め。

### 8. VTP のように SGT 情報を伝搬させる
*   **要件**: SXP を使用せずに、トランクリンク経由でタグを保持せよ。
*   **設定**: `switchport trunk native vlan tag` とインターフェイスの CTS 有効化。

### 9. SGACL 統計のトラブルシュート
*   **課題**: ヒットカウントが上がらない原因（Enforcement 漏れ）を特定せよ。

### 10. SGT 伝搬における VRF の考慮
*   **要件**: VRF 内部で SXP を動作させよ。
*   **設定**: `cts sxp peer ... vrf [NAME]`。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `cts role-based enforcement` がグローバル設定にあるが、特定のインターフェイスのみ SGACL が効かない。なぜか？
    *   **回答**: そのインターフェイスで `cts role-based enforcement` コマンドが個別に入力されていない、または Egress 方向のパケットに SGT が正しく付与されていない。
2.  **トラブルシュート**: SXP ピアの状態が `Pending` から進まない。確認すべきポートは？
    *   **回答**: TCP ポート **649**。中間デバイスの ACL でこのポートが許可されているか確認する。
3.  **Design**: TrustSec 環境で、SGT 非対応のサードパーティ製スイッチをパケットが通過する場合、SGT を維持する方法は？
    *   **回答**: SGT 非対応区間を **SXP** を使用してバイパス（論理的にマッピングを転送）し、パケット自体はタグなしで転送する。
4.  **実装**: ISE から新しい SGT マトリックスを即座にスイッチへ反映させるコマンドは？
    *   **回答**: `cts refresh environment-data` および `cts refresh policy`。
5.  **コンフィグ読解**: `cts role-based entry 0.0.0.0 sgt 65535` という設定の意図は？
    *   **回答**: 明示的にタグが付与されていないすべてのトラフィック（Unknown）に対し、デフォルトで SGT 65535 を割り当てるための設定。

---

## 🔗 参考リソース

*   **Cisco TrustSec Configuration Guide**
    *   [Cisco TrustSec Configuration Guide, Cisco IOS XE](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-3/configuration_guide/sec/b_173_sec_9300_cg/cisco_trustsec.html)
*   **Cisco SXP Guide**
    *   [SGT Exchange Protocol (SXP) Deployment Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/trustsec/configuration/guide/sxp_config-guide.html)
*   **Cisco Live (BRKSEC-2022)**
    *   [Segmentation with TrustSec](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「SGT はパケットの身分証」です。物理的な場所が変わっても身分証が変わらなければ、ポリシーは一貫して適用されます。
*   **図解**: パケットの L2 ヘッダーを思い浮かべ、CMD フィールドがどこに挟まるかを意識してください。
*   **注意点**: ラボ試験では、SXP の IP アドレス（ピア）に Loopback を使う場合、その Loopback へのルーティングが確立していることをまず確認してください。
