---
layout: default
title: 5.6-WCCP
nav_order: 6
parent: 5.0-Advanced-Threat-Protection
---

# 5.6 WCCP redirection on Cisco devices

**WCCP (Web Cache Communication Protocol)** は、トラフィック（主に HTTP/HTTPS）を Cisco スイッチ、ルータ、または ASA から、Cisco WSA (Web Security Appliance) などのコンテンツエンジンへ動的にリダイレクトするためのプロトコルです。これにより、クライアント側にプロキシ設定をすることなく（透過モード）、Web セキュリティ機能を適用することが可能になります。

---

## 📘 概要

*   **機能概要**: ネットワークデバイスを通過する特定のトラフィックを遮断し、分析やキャッシュのために外部の WSA へ転送する仕組みです。
*   **利用目的**: 透過プロキシの実現、Web トラフィックの集約、セキュリティポリシーの一元適用、キャッシュによる帯域節約。
*   **どのような場面で利用するか**:
    *   **透過型導入**: クライアントのブラウザ設定を変更せずに Web フィルタリングを導入したい場合。
    *   **負荷分散**: 複数の WSA をグループ化し、トラフィックを分散処理させたい場合。
    *   **冗長化**: WSA がダウンした際に自動的にリダイレクトを停止し、直接通信（Fail-Open）を許可したい場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **プロトコルバージョン** | WCCPv2 が一般的。UDP ポート 2048 を使用。 |
| **サービス ID** | **Standard (0)**: HTTP 用。**Dynamic (60-99)**: HTTPS や他プロトコル用。 |
| **転送方式** | **GRE**: カプセル化（L3 越え可）。**L2**: MAC 書き換え（直接接続）。 |
| **割り当て方式** | **Hash**: デフォルト。**Mask**: 高度な ASIC 処理・分散用。 |
| **リダイレクト方向** | `in` (入力時) または `out` (出力時)。 |
| **対応機種** | ISR/ASR ルータ, Catalyst スイッチ, Cisco ASA/FTD。 |

---

## 🏗 動作原理

WCCP は、**Router/Switch (Service Connector)** と **WSA (Content Engine)** の間の通信で成り立ちます。

```text
Client (Web Request)
   ↓
Router (Service Connector)
   ↓ (1) WCCP Redirection (GRE or L2)
WSA (Content Engine)
   ↓ (2) Inspection & Policy Match
   ↓ (3) Request to Internet (on behalf of client)
Internet (Web Server)
```

---

## ⚙ 動作シーケンス

1.  **Here I am**: WSA が UDP 2048 を使い、ルータに自身の存在を通知します。
2.  **I see you**: ルータが WSA を認識し、リダイレクトを開始するための「サービスグループ」を構築します。
3.  **トラフィック捕捉**: ルータのインターフェイスに届いたパケットが `ip wccp redirect` に合致するか判定されます。
4.  **リダイレクト**:
    *   **GRE**: ルータがパケットを GRE で包み WSA へ送信。
    *   **L2**: ルータが宛先 MAC を WSA のものに書き換えて送信（同一サブネット限定）。
5.  **リターン通信**: WSA は処理したパケットをルータへ戻すか、直接インターネットへ送出します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **サービス ID の不一致**: ルータ側で `ip wccp 61` と設定し、WSA 側でサービス ID `62` を設定すると通信が成立しません。必ず一致させる必要があります。
*   **Redirect-List の適用**: 全トラフィックを送ると負荷が高すぎるため、特定の内部サブネットのみを対象にする ACL（`redirect-list`）の作成が求められます。
*   **ASA のリダイレクト制限**: ASA ではリダイレクト方向がルータと異なる場合がある（`wccp redirect in` が基本）点に注意してください。
*   **HTTPS の透過リダイレクト**: ポート 443 を処理する場合、サービス ID 70 や独自の動的 ID を使い、WSA 側で HTTPS プロキシを有効にする必要があります。
*   **Forwarding/Return Method**: GRE か L2 か、両端で設定が一致していないと、パケットが WSA に届いても破棄されます。

---

## 🛠 設定方法

### 1. Cisco IOS ルータ：基本設定
```bash
! WCCP サービス ID 0 (HTTP) の有効化
ip wccp web-cache redirect-list WCCP_CLIENTS
!
! リダイレクト対象の制限
ip access-list extended WCCP_CLIENTS
 permit tcp 10.1.1.0 0.0.0.255 any eq 80
!
! インターフェイスへの適用
interface GigabitEthernet0/1
 ip wccp web-cache redirect in
```

### 2. Cisco ASA：基本設定
```bash
! WSA の IP アドレスを指定
wccp interface inside 10.1.1.5 redirect-list WCCP_ACL
!
! リダイレクト対象の指定
access-list WCCP_ACL extended permit tcp 10.1.1.0 255.255.255.0 any eq 80
!
! インターフェイスでリダイレクトを有効化
wccp-redirect inside in
```

### 3. Cisco WSA：WCCP 構成 (GUI 概要)
1.  **Network > WCCP** に移動。
2.  **Add Service Group** をクリック。
3.  Service ID: `Standard (0)` または `61` 等。
4.  Router IP: ルータ/ASA の IP を入力。
5.  Forwarding/Return Method: `GRE` または `L2` を選択（ルータと一致させる）。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **WCCP 全体状態確認** | <code>show ip wccp [ID]</code> |
| **WSA との接続確認** | <code>show ip wccp [ID] detail</code> |
| **統計情報の表示** | <code>show ip wccp [ID] stats</code> |
| **ASA での状態確認** | <code>show wccp</code> |
| **パケットの流れを確認** | <code>debug ip wccp events</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| **Status: Not Visible** | UDP 2048 が遮断されている | ルータ/WSA 間の ACL を確認し、UDP 2048 を許可する。 |
| **リダイレクトされない** | インターフェイスの向きミス | `redirect in` (着信) か `out` (発信) かをフロー図で再確認。 |
| **WSA がループする** | 自身の通信をリダイレクト | `group-list` を使い、WSA 自身の IP をリダイレクトから除外する。 |
| **Web サイトが開かない** | Method の不一致 | <code>show ip wccp detail</code> で Forward/Return が一致しているか確認。 |

---

## ⚠ 制限事項

*   **MTU の問題**: GRE を使用する場合、24 バイトのオーバーヘッドによりパケットが断片化され、パフォーマンスが低下することがあります。
*   **ハードウェアサポート**: Catalyst スイッチなどの一部の機種では、WCCP はハードウェア（ASIC）ではなくソフトウェアで処理され、CPU 負荷が高まる場合があります。
*   **マルチキャスト**: WCCP 通信でマルチキャストを使用する場合、ルータ側での PIM 設定など複雑な構成が必要になるため、通常はユニキャストが推奨されます。

---

## 🔄 他技術との関連

*   **5.5 Web filtering**: リダイレクト後のトラフィックに対して、WSA が URL フィルタリングを実施します。
*   **3.3.b QoS**: リダイレクトされた Web トラフィックに優先順位を付けることができます。
*   **1.2.a NAT**: WSA がインターネットへ抜ける際にルータで NAT 変換を行う設計が一般的です。

---

## 🧩 比較表

### GRE 転送 vs L2 転送

| 特徴 | GRE (Generic Routing Encapsulation) | L2 (Layer 2) Forwarding |
| :--- | :--- | :--- |
| **構成** | L3 ヘッダーを付与してカプセル化。 | 宛先 MAC アドレスを WSA に書き換え。 |
| **トポロジー** | ルータと WSA が別サブネットでも可。 | **同一サブネット**（直接接続）が必須。 |
| **負荷** | カプセル化解除のため WSA の CPU 消費増。 | 低負荷（ASIC での高速処理が可能）。 |
| **推奨** | 複雑なネットワーク構成時。 | 高パフォーマンスが求められる時。 |

---

## 💡 ベストプラクティス

1.  **Group-List の使用**: ルータの `ip wccp group-list` で WSA の IP を明示的に許可し、不正なデバイスがサービスグループに参加するのを防ぎます。
2.  **Exclude ACL**: WSA 自身の IP トラフィックが再度自分にリダイレクトされないよう、ACL の先頭で WSA 宛/発の通信を `deny` (除外) します。
3.  **L2 Forwarding の優先**: パフォーマンスと MTU の問題を避けるため、可能な限り L2 転送を使用します。

---

## 📝 ラボ学習・設定サンプル例

### 1. HTTP 基本リダイレクト
*   **要件**: 内部セグメント 10.1.1.0/24 からのポート 80 通信を WSA (10.1.1.100) へ送れ。

### 2. HTTPS サービスグループ (ID 70)
*   **要件**: サービス ID 70 を使用して、SSL トラフィックをリダイレクトせよ。
*   **設定**: `ip wccp 70 redirect-list HTTPS_ACL`.

### 3. リダイレクト方向 `out` の活用
*   **要件**: インターネットへ出ていく（Outside インターフェイス）トラフィックを捕まえよ。

### 4. Redirect-List による特定ドメイン除外
*   **要件**: 社内アップデートサーバへの通信は WSA に送らず直接通信させよ。

### 5. ASA 透過リダイレクト
*   **要件**: ASA inside インターフェイスで WCCP を有効にせよ。

### 6. 複数 WSA によるハッシュ分散
*   **要件**: 2 台の WSA を追加し、デフォルトのハッシュ方式で負荷分散されるか確認せよ。

### 7. L2 Forwarding への変更
*   **要件**: GRE から L2 転送に切り替え、`show` コマンドで確認せよ。

### 8. Fail-Open (WCCP 切断時の動作)
*   **要件**: WSA をシャットダウンした際、クライアントが直接インターネットへ抜けられることを確認せよ。

### 9. 認証付き WCCP (MD5)
*   **要件**: WSA とルータ間の WCCP メッセージにパスワードを設定せよ。
*   **設定**: `ip wccp password [PASSWORD]`.

### 10. WCCP Mask-Assignment 構成
*   **要件**: Catalyst 6500 等で一般的に使われる Mask 方式をシミュレートせよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: ルータで <code>show ip wccp 0</code> を実行したところ、`WCCP Client: Not Visible` となっている。最も可能性の高い原因は？
    *   **回答**: ルータと WSA 間の通信路で **UDP ポート 2048** が拒否されている、または WSA 側の Router IP 設定が誤っている。
2.  **Design**: ルータと WSA が異なる VLAN に配置されている。どの転送方式を選択すべきか？
    *   **回答**: **GRE 転送**。L2 転送は隣接する（同一セグメントの）デバイス間でのみ動作するため。
3.  **コンフィグ読解**: 下記の設定で、10.1.1.50 からの通信はリダイレクトされるか？
    `access-list 100 permit tcp 10.1.1.0 0.0.0.255 any eq 80`
    `ip wccp web-cache redirect-list 100`
    *   **回答**: はい、10.1.1.50 は 10.1.1.0/24 の範囲内であり、ポート 80 通信であればリダイレクトされる。
4.  **実装**: WCCP で HTTPS トラフィックを扱う場合、標準の `web-cache` (ID 0) サービスを使用できるか？
    *   **回答**: **不可**。`web-cache` はポート 80 専用。HTTPS には動的サービス ID (例: 70) を使用する。
5.  **トラブルシュート**: 特定の Web サイトだけが表示されない。WSA のログには記録がない。ルータ側で何を調査すべきか？
    *   **回答**: <code>show ip access-lists</code> でリダイレクト用 ACL のカウンタが上がっているか、および <code>show ip wccp stats</code> でパケットがドロップされていないかを確認。

---

## 🔗 参考リソース

*   **Cisco Configuration Guide**: [Configuring WCCP on IOS](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipapp_wccp/configuration/xe-16/iap-wccp-xe-16-book.html)
*   **Cisco ASA 9.4 Guide**: [WCCP Redirection on ASA](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/firewall/asa-94-firewall-config/access-wccp.html)
*   **Cisco Live (BRKSEC-3020)**: [Troubleshooting Content Security with WCCP](https://www.ciscolive.com/)
*   **WSA 9.2 User Guide**: [Integrating WSA with WCCP Devices](https://www.cisco.com/c/en/us/td/docs/security/wsa/wsa9-2/user_guide/b_WSA_UserGuide_9_2_0.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: WCCP は「交渉」です。ルータが勝手に投げるのではなく、WSA が「投げてくれ」と頼みに来るプロセスを理解してください。
*   **図解**: `WSA --(I am here)--> Router --(Got it, here is traffic)--> WSA`.
*   **注意点**: ラボ試験では、**WSA 自身のトラフィックを除外する ACL** を忘れないことが、ループ（CPU 100%）を防ぐ鉄則です。
