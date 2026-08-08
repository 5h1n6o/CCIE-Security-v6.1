---
layout: default
title: 3.1.c-iACLs
nav_order: 3
parent: 3.1-Device-hardening
grand_parent: 3.0-Security-Infrastructure
---

# 3.1.c iACLs (Infrastructure Access Control Lists)

**iACL (Infrastructure Access Control List)** は、ネットワークの境界（エッジ）において、ネットワークインフラストラクチャ自体（ルータやスイッチの物理/論理 IP アドレス）を宛先とするトラフィックを制限し、保護するための強力なセキュリティ手法です。通過するトラフィック（Transit Traffic）は許可しつつ、インフラデバイスへの直接的な攻撃を未然に防ぐ「第一防衛線」として機能します。

---

## 📘 概要

*   **機能概要**: ネットワークインフラを構成するデバイスの IP アドレス空間を特定し、その空間に対するアクセスをエッジインターフェイスで明示的に制御します。
*   **利用目的**: インフラデバイスへの DoS 攻撃の防止、不正な管理アクセスの遮断、およびスキャンによる偵察活動の抑制。
*   **どのような場面で利用するか**: エンタープライズネットワークのインターネット境界ルータや、異なるセキュリティセグメント間の境界ポイント。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **保護対象** | インフラデバイスの IP（Loopback, 物理 IF IP, HSRP VIP 等）。 |
| **適用場所** | ネットワーク境界（AS 境界）の外部インターフェイスの Ingress 方向。 |
| **トラフィック分類** | **Infrastructure Traffic**（自分宛）と **Transit Traffic**（通過）を区別。 |
| **実装方式** | 標準または拡張 IP アクセスリスト (ACL)。 |
| **メリット** | ハードウェア（ASIC）で処理されるため、CPU 負荷をかけずに広範囲を保護。 |
| **設計上の注意点** | 必要なプロトコル（BGP, OSPF 等）を漏れなく許可する緻密な設計が必要。 |

---

## 🏗 動作原理

iACL は、ルータを通過するトラフィックと、ルータそのものを宛先とするトラフィックを「宛先 IP」に基づいて振り分けます。

```text
[ Internet / Outside ]
       ↓
[ Edge Router (Ingress IF) ]
       ↓
[ iACL Check ]
       ├─ (A) 宛先 = インフラ IP 空間?
       │      ├─ 許可された送信元・プロトコル → [ CPU / Local Process ] へ
       │      └─ それ以外 → [ DROP / Log ]
       │
       └─ (B) 宛先 = 内部公開サーバ等 (Transit)?
              └─ 透過的に転送 (Permit any any at the end)
```

---

## ⚙ 動作シーケンス

1.  **インフラ IP 範囲の定義**: 自組織が管理するルータ/スイッチの全 IP アドレスをリストアップします。
2.  **プロトコル許可リストの作成**: 外部から正当に必要な通信（例：信頼できるピアとの BGP、外部からの ICMP 監視）を定義します。
3.  **拒否ポリシーの定義**: 許可リストにない、インフラ IP 宛の全トラフィックを `deny` します。
4.  **Transit の許可**: インフラ以外を宛先とする通常の通信を `permit ip any any` で許可します。
5.  **インターフェイス適用**: 外部（Untrusted）に面したインターフェイスに適用します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **インフラアドレスの識別**: 試験問題で指定されたサブネット（例：Loopback 網）を正確に ACL の宛先に含める必要があります。
*   **RFC 1918 との組み合わせ**: 送信元 IP がプライベートアドレス（10.0.0.0/8 等）である外部からのパケットを拒否する設定がしばしば求められます。
*   **確立済み通信の許可**: AnyConnect や Site-to-Site VPN など、インフラ（ASA/FTD）を終端とする通信を壊さないように、`established` キーワードや特定の UDP ポートを考慮した設計が問われます。
*   **フラグメントの処理**: iACL で `deny` する際、フラグメントパケットをどのように扱うか（L4 情報がない場合の挙動）が問われることがあります。
*   **ロギングの要件**: 「拒否されたパケットをログに記録せよ」という要件に対し、`log` または `log-input` を適切に使用します。

---

## 🛠 設定方法

### 1. インフラ保護 ACL の作成例
```bash
! インフラ用のアドレス範囲をオブジェクトグループ等で管理（推奨）
ip access-list extended ACL-INFRA-PROTECTION
 ! 1. 信頼できるピアからのルーティングプロトコルを許可
 permit tcp host 203.0.113.10 host 203.0.113.1 eq 179
 ! 2. 管理セグメントからの SSH を許可
 permit tcp 192.168.100.0 0.0.0.255 203.0.113.0 0.0.0.255 eq 22
 ! 3. 外部からの監視 ICMP を許可 (必要最小限)
 permit icmp any 203.0.113.0 0.0.0.255 echo
 permit icmp any 203.0.113.0 0.0.0.255 unreachable
 ! 4. その他のインフラ IP 宛トラフィックを拒否してログ
 deny ip any 203.0.113.0 0.0.0.255 log-input
 ! 5. 残りの Transit トラフィック（通過パケット）を許可
 permit ip any any
```

### 2. インターフェイスへの適用
```bash
interface GigabitEthernet0/0
 description OUTSIDE-INTERFACE
 ip access-group ACL-INFRA-PROTECTION in
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **ACL のヒットカウント確認** | <code>show access-lists ACL-INFRA-PROTECTION</code> |
| **インターフェイスへの適用状態** | <code>show ip interface GigabitEthernet0/0 \| include access</code> |
| **拒否ログのリアルタイム確認** | <code>show logging</code> (ログレベル 6 以上が必要) |
| **インフラ IP への到達性確認** | <code>ping [Infrastructure_IP]</code> (外部ホストから) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 正当な BGP ネイバーが切断される | iACL で TCP 179 を許可し忘れた | <code>show access-list</code> でヒットを確認し、許可行を追加。 |
| 内部からのインターネット通信が不可 | ACL 末尾の <code>permit ip any any</code> 欠落 | 設定を確認し、Transit 許可エントリを追加。 |
| 外部からの VPN 接続が失敗する | IPsec (ESP/UDP 500) が拒否対象に含まれている | 外部 IP 宛の <code>permit esp</code> 等を確認。 |
| 拒否ログが出力されない | ACL 行に <code>log</code> オプションがない | <code>deny</code> 行の末尾に <code>log</code> を追記する。 |

---

## ⚠ 制限事項

*   **メンテナンスの負担**: インフラ IP が追加されるたびに ACL を更新する必要があり、漏れがあると脆弱性になります。
*   **フラグメント処理**: 拡張 ACL の L4 フィルタはフラグメントされた 2 パケット目以降を正しく識別できない場合があります。
*   **CPU への影響 (Logging)**: `log` オプションを多用すると、大量の攻撃を受けた際に CPU 負荷が高まります。`logging rate-limit` の併用を検討してください。

---

## 🔄 他技術との関連

*   **CoPP (3.1.a)**: iACL が「ネットワーク境界でのフィルタ」であるのに対し、CoPP は「デバイス自身の CPU 直前での Police/Filter」です。多層防御として併用します。
*   **uRPF (3.3.a)**: 送信元 IP スプーフィングを防止する技術。iACL と併せて「偽装パケットによるインフラ攻撃」を防ぎます。
*   **Infrastructure アドレス空間**: BCP 38 (RFC 2827) に基づくフィルタリングの一部として実装されます。

---

## 🧩 比較表

### iACL vs CoPP

| 特徴 | iACL (Infrastructure ACL) | CoPP (Control Plane Policing) |
| :--- | :--- | :--- |
| **フィルタリング位置** | インターフェイス（入口） | コントロールプレーン（CPU 直前） |
| **処理エンジン** | 主にハードウェア (ASIC) | ソフトウェア/ハードウェア混在 |
| **主な機能** | Permit / Deny | Rate Limit (Policing) / Drop |
| **保護対象** | インフラ全体の IP 資産 | デバイス個別のリソース |

---

## 💡 ベストプラクティス

1.  **インフラ用サブネットの集約**: ACL 管理を容易にするため、ルータの IP アドレスを特定のサブネットに集約して設計します。
2.  **established の活用**: 内部から外部への通信の戻りパケットを効率的に許可するために使用します。
3.  **ICMP Unreachable の許可**: パス MTU 探索などを壊さないよう、特定の ICMP タイプを許可します。
4.  **RFC 1918 の遮断**: 外部インターフェイスでは、送信元がプライベート IP であるパケットをインフラ宛・Transit 宛に関わらず拒否します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的なインフラ保護
*   **要件**: 外部からのトラフィックのうち、172.16.0.0/24（インフラ網）宛を全て拒否し、それ以外は許可せよ。
*   **設定**: `permit icmp any 172.16.0.0 0.0.0.255`, `deny ip any 172.16.0.0 0.0.0.255`, `permit ip any any`。

### 2. 特定の BGP ピアのみ許可
*   **要件**: ネイバー 203.0.113.5 からの BGP (TCP 179) のみを許可せよ。

### 3. SSH ホワイトリストの実装
*   **要件**: 管理ホスト 192.168.1.10 からの SSH のみインフラ IP 宛に許可せよ。

### 4. 拒否ログの記録
*   **課題**: 拒否されたパケットの送信元と受信 IF をログで確認せよ。
*   **設定**: `deny ip any 172.16.0.0 0.0.0.255 log-input`。

### 5. Fragment パケットの制御
*   **要件**: インフラ宛のフラグメントパケットを明示的に拒否せよ。
*   **設定**: `deny ip any 172.16.0.0 0.0.0.255 fragments`。

### 6. IPv6 iACL の実装
*   **要件**: IPv6 インフラアドレス宛の不正トラフィックを遮断せよ。

### 7. ASA における iACL
*   **要件**: Outside インターフェイスに `access-group` を適用し、ASA 自身の IP 宛通信を制限せよ。

### 8. HSRP VIP の保護
*   **要件**: デフォルトゲートウェイの VIP 宛の不要なパケットをエッジでブロックせよ。

### 9. オブジェクトグループによる簡素化
*   **課題**: 多数のルータ IP を `object-group network` でまとめ、ACL を記述せよ。

### 10. Transit トラフィックへの影響確認
*   **課題**: インフラ保護 ACL 適用後も、内部サーバへの HTTP 通信が維持されていることを確認せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: ACL の最後に `permit ip any any` がない場合、iACL は Transit トラフィックにどう影響するか？
    *   **回答**: 暗黙の `deny` により、インフラ宛以外のすべての通過トラフィックも遮断される。
2.  **Design**: iACL を適用するのに最適な場所は？
    *   **回答**: ネットワークの外部境界（Untrusted 側）のインターフェイスの Ingress 方向。
3.  **トラブルシュート**: iACL 適用後、外部監視ツールからの Ping が応答しなくなった。原因は？
    *   **回答**: iACL で監視ツール IP からの `icmp echo` を明示的に許可していないため。
4.  **実装**: 外部からのスプーフィングパケットを iACL で防ぐための設定行は？
    *   **回答**: 自組織の内部 IP 範囲を送信元とするパケットを外部インターフェイスで `deny` する設定。
5.  **コンフィグ読解**: `deny tcp any range 1 65535 any range 1 65535` という行が iACL にある場合、これは何を意味するか？
    *   **回答**: ポート番号を持つすべての TCP トラフィックを拒否する設定。インフラ保護としては強すぎる可能性があるため注意が必要。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [Configuring Infrastructure Access Control Lists](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_acl/configuration/xe-16/sec-data-acl-xe-16-book/sec-cfg-infra-acl.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [Securing the Control Plane of Cisco Routers](https://www.ciscolive.com/)
*   **Technical Notes**
    *   [Core Router Security: Infrastructure Protection ACLs](https://www.cisco.com/c/en/us/support/docs/ip/access-lists/43501-copp.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「iACL はインフラの鎧」です。中を通る人（Transit）を止めるのではなく、鎧（インフラ IP）に槍を投げようとする人を入口で止めるイメージです。
*   **図解**: ネットワークの全体図を書き、どこが「インフラ空間」でどこが「Transit 空間」かを色分けすると、ACL の Permit/Deny ロジックが整理しやすくなります。
*   **注意点**: ラボ試験では、物理インターフェイスの IP を宛先にした `deny` だけでなく、ルータが持つ全ての IP（Loopback 含む）を保護対象として認識しているかが採点ポイントになります。
