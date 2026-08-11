---
layout: default
title: 3.7.b-RFC2827
nav_order: 3
parent: 3.7-Compliance
grand_parent: 3.0-Security-Infrastructure
---

# 3.7.b RFC 2827

**RFC 2827**（現在は **BCP 38** として知られる）は、インターネットにおける **IP 送信元アドレスのなりすまし（IP Spoofing）** を利用したサービス拒否（DoS）攻撃を防止するためのネットワーク入力フィルタリング（Network Ingress Filtering）に関する標準的な手法を定義した文書です。CCIE Security v6.1 においては、このガイドラインを Cisco IOS/IOS-XE デバイス上で **uRPF (Unicast Reverse Path Forwarding)** や **ACL** を用いてどのように実装し、組織のセキュリティポリシーに適合させるかが問われます。

---

## 📘 概要

*   **機能概要**: ネットワークの境界（エッジ）において、外部から流入するパケットの送信元 IP アドレスが、そのインターフェイスの背後に正当に存在するものであるかを検証し、なりすましパケットを破棄します。
*   **利用目的**: 送信元 IP を偽装した DDoS 攻撃（特に増幅攻撃）の踏み台になることを防ぎ、攻撃者の追跡可能性（Traceability）を向上させます。
*   **どのような場面で利用するか**: 
    *   組織のインターネット境界ルータの Ingress インターフェイス。
    *   サービスプロバイダー（ISP）が顧客から受信するトラフィックのフィルタリング。
    *   内部ネットワーク間のセグメント境界における不正トラフィックの遮断。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な実装技術** | **uRPF (Unicast Reverse Path Forwarding)**、**Ingress ACL**。 |
| **中核概念** | パケットの「送信元」をルーティングテーブル（FIB）で逆引き検証する。 |
| **主な保護対象** | データプレーン、および偽装パケットによるコントロールプレーンへの負荷。 |
| **メリット** | DDoS 攻撃の抑制、RFC 1918 等の不正アドレスの流入防止。 |
| **デメリット** | 非対称ルーティング環境で Strict モードを使用すると正常通信がドロップされる。 |
| **対応機種** | Catalyst スイッチ、IOS-XE ルータ、ASA (ip verify)。 |
| **設計上の注意点** | 境界において「外から内の自組織 IP」や「内から外の他組織 IP」を拒否する。 |

---

## 🏗 動作原理

RFC 2827 の推奨事項を Cisco 機器で自動化したものが **uRPF** です。パケットがインターフェイスに到達した際、ルータはその送信元 IP アドレスがルーティングテーブル（FIB）において「正当なパス」を持っているかを確認します。

```text
[ Packet Arrival ]
      ↓
[ Extract Source IP Address ]
      ↓
[ Look up Source IP in FIB (Forwarding Information Base) ]
      ↓
[ Check: Is the source reachable via the interface it arrived on? ]
      ├─ YES ─→ [ Permit Packet ] (Strict Mode)
      └─ NO  ─→ [ Check: Is the source reachable via ANY interface? ]
                  ├─ YES ─→ [ Permit Packet ] (Loose Mode)
                  └─ NO  ─→ [ Drop Packet ]
```

---

## ⚙ 動作シーケンス

1.  **パケット受信**: Ingress インターフェイスで IP パケットを受信します。
2.  **FIB 参照**: CEF (Cisco Express Forwarding) テーブルを参照し、送信元 IP に対する戻りパスを確認します。
3.  **モード別検証**:
    *   **Strict Mode**: 送信元 IP への最短パスが、パケットを受信したインターフェイスと一致する場合のみ許可します。
    *   **Loose Mode**: 送信元 IP が FIB 内に存在（到達可能）であれば許可します。
4.  **ACL フィルタリング (補完)**: uRPF でカバーできない範囲や、特定の予約済みアドレス（RFC 1918 等）を静的 ACL でドロップします。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **uRPF モードの使い分け**: ラボ要件で「非対称パスがない環境」なら **Strict**、「マルチホームや冗長構成で戻りパスが変わる可能性」があるなら **Loose** を選択します。
*   **Default Route の考慮**: `allow-default` オプションの有無が重要です。これがないと、デフォルトルート経由でしか到達できない送信元アドレスが Strict モードで破棄されます。
*   **RFC 1918 フィルタリング**: 境界インターフェイスにおいて、送信元がプライベートアドレスであるパケットを明示的に拒否し、`log` オプションで記録を残す構成が頻出です。
*   **ログの読み取り**: `%SEC-6-IPACCESSLOGDP` メッセージから、どの送信元が RFC 2827 違反（スプーフィングの疑い）としてドロップされたかを特定できる必要があります。
*   **CEF の有効化**: uRPF は CEF が有効でないと動作しません。試験ではまず `ip cef` を確認してください。

---

## 🛠 設定方法

### 1. uRPF Strict モードの設定 (IOS-XE)
```bash
interface GigabitEthernet0/1
 ip verify unicast source reachable-via rx
```

### 2. uRPF Loose モードの設定 (非対称パス対応)
```bash
interface GigabitEthernet0/2
 ip verify unicast source reachable-via any
```

### 3. RFC 1918/RFC 2827 準拠 Ingress ACL の例
組織の境界インターフェイス (Outside) に適用し、内部からのなりすましや不正アドレスを排除します。
```bash
ip access-list extended ANTI-SPOOF
 deny ip 10.0.0.0 0.255.255.255 any log
 deny ip 172.16.0.0 0.15.255.255 any log
 deny ip 192.168.0.0 0.0.255.255 any log
 permit ip any any
!
interface GigabitEthernet0/1
 ip access-group ANTI-SPOOF in
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **uRPF の設定とドロップ数確認** | <code>show ip interface [Interface]</code> |
| **ACL のヒット数確認** | <code>show access-lists [Name]</code> |
| **CEF テーブルの確認** | <code>show ip cef [IP]</code> |
| **ログバッファの確認** | <code>show logging \| include SEC-6</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 正常な通信がドロップされる | 非対称ルーティングでの Strict 検証失敗 | <code>show ip interface</code> | <code>Loose (any)</code> モードへ変更。 |
| デフォルトルート先からのパケットが落ちる | <code>allow-default</code> 設定漏れ | <code>show ip interface</code> | コマンド末尾に <code>allow-default</code> を追加。 |
| uRPF が動作しない | CEF が無効 | <code>show ip cef</code> | グローバルで <code>ip cef</code> を有効化。 |
| 偽装パケットが通過する | インターフェイス適用間違い | <code>show run interface</code> | 適切な Ingress ポイントへの適用を確認。 |

---

## ⚠ 制限事項

*   **IPv6 依存**: IPv4 と IPv6 でコマンドが異なります（`ipv6 verify ...`）。
*   **パフォーマンス**: 大規模な FIB を持つ環境では、uRPF チェックによりフォワーディング性能に影響が出る場合があります。
*   **トンネル構成**: GRE や IPsec トンネルの終端点では、デカプセル化後のパケットに対する検証順序に注意が必要です。

---

## 🔄 他技術との関連

*   **3.3.a uRPF**: RFC 2827 を実現するための直接的な機能。
*   **3.1.c iACLs**: インフラ ACL。デバイス自体の保護を目的とした境界フィルタリング。
*   **3.4.e DHCP Snooping**: L2 レベルでの送信元 IP 検証（IP Source Guard）の基盤。
*   **3.7.a ISO 27001**: 組織のセキュリティ管理策として、ネットワークの整合性維持に RFC 2827 準拠が求められます。

---

## 🧩 比較表

### uRPF Strict vs Loose

| 特徴 | Strict Mode (rx) | Loose Mode (any) |
| :--- | :--- | :--- |
| **検証基準** | 到達インターフェイスと戻りパスの一致 | FIB 内にルートが存在することのみ |
| **セキュリティ強度** | 高（なりすましを厳格に排除） | 中（FIB にないアドレスのみ排除） |
| **非対称ルーティング** | サポート不可 | サポート可能 |
| **主な用途** | シングルホームのエッジ | コアネットワーク、マルチホーム ISP |

---

## 💡 ベストプラクティス

1.  **エッジでの実装**: 可能な限りトラフィックの発生源（エッジ）に近い場所でフィルタリングを行います。
2.  **Logging の併用**: `log` キーワードを ACL に付け、不正アクセス試行を SIEM 等で監視します。
3.  **段階的導入**: 最初は ACL で permit しつつ log を取り、正当なパケットが落ちないことを確認してから uRPF を有効化します。
4.  **例外の定義**: 特定の監視サーバやルーティングの例外がある場合は、uRPF の検証対象から外すための ACL を活用します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な Strict uRPF
*   **要件**: Gi0/1 で受信する全パケットに対し、厳格な送信元検証を適用せよ。
*   **設定**: `ip verify unicast source reachable-via rx`

### 2. RFC 1918 遮断とログ
*   **要件**: インターネット境界において、10.0.0.0/8 からの流入を拒否し、詳細を記録せよ。
*   **設定**: `deny ip 10.0.0.0 0.255.255.255 any log`

### 3. デフォルトルートを許容する uRPF
*   **要件**: デフォルトルート経由の送信元も許可する Strict uRPF を構成せよ。
*   **設定**: `ip verify unicast source reachable-via rx allow-default`

### 4. 複数パス環境での Loose uRPF
*   **要件**: パスが非対称になる可能性がある境界で、送信元 IP が実在することのみ確認せよ。
*   **設定**: `ip verify unicast source reachable-via any`

### 5. ACL による特定の送信元ホワイトリスト
*   **要件**: uRPF で落ちるはずの監視サーバ (1.1.1.1) だけは許可せよ。
*   **設定**: `ip verify unicast source reachable-via rx 100` (ACL 100 で 1.1.1.1 を許可)

### 6. IPv6 での Ingress Filtering
*   **要件**: IPv6 ネットワークにおいても同様の送信元検証を実施せよ。
*   **設定**: `ipv6 verify unicast source reachable-via rx`

### 7. VACL を用いた L2 スプーフィング防止
*   **要件**: VLAN 10 内で、特定 IP 以外からの通信を L2 レベルで遮断せよ。

### 8. ASA での IP 検証
*   **要件**: ASA の Outside インターフェイスで anti-spoofing を有効にせよ。
*   **設定**: `ip verify reverse-path interface outside`

### 9. Control-Plane 保護との連携
*   **要件**: 偽装された ICMP トラフィックを CoPP で制限せよ。

### 10. トラブルシュート：非対称パスの修正
*   **課題**: `show ip interface` で uRPF ドロップを確認し、Loose モードへの切り替えで解決せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `ip verify unicast source reachable-via rx` 設定下で、非対称ルーティングが発生した場合、パケットはどう処理されるか？
    *   **回答**: 戻りパスが異なるインターフェイスを指すため、パケットは破棄される。
2.  **Design**: 大規模なマルチホーム ISP ネットワークで RFC 2827 を実装する場合、推奨される uRPF モードは？
    *   **回答**: **Loose Mode (any)**。非対称パスによる正常パケットのドロップを避けるため。
3.  **トラブルシュート**: ルータで uRPF を設定したが、統計上の `drop count` が全く増えない。何を確認すべきか？
    *   **回答**: CEF が有効か（`show ip cef`）。CEF 無効時は uRPF は機能しない。
4.  **実装**: 「自組織の IP アドレス範囲が外部インターフェイスの送信元として現れた場合、それを破棄せよ」という要件を ACL で実装せよ。
    *   **回答**: 外部インターフェイスの `in` 方向に `deny ip [自組織Network] [Mask] any log` を含む ACL を適用する。
5.  **コンフィグ読解**: uRPF 設定コマンドの末尾にある ACL 番号はどのような役割を果たすか？
    *   **回答**: 例外リスト（Whitelist）。ACL で permit されたパケットは、uRPF の検証に失敗しても許可される。

---

## 🔗 参考リソース

*   **Cisco Live (BRKSEC-2003)**: [Securing the Infrastructure with Ingress Filtering](https://www.ciscolive.com/)
*   **Cisco Configuration Guide**: [Configuring Unicast Reverse Path Forwarding](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_urpf/configuration/xe-16/sec-data-urpf-xe-16-book.html)
*   **Technical Notes**: [Understanding Unicast Reverse Path Forwarding](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/17445-bcp38.html)
*   **Design Guide**: [Cisco SAFE - Network Edge Security](https://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/Security/SAFE_RG/SAFE_rg/chap3.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「RFC 2827 = BCP 38 = uRPF」と脳内でリンクさせてください。
*   **図解**: 
    *   `reachable-via rx` は「来た道をそのまま帰れるか（対称）」をチェック。
    *   `reachable-via any` は「その人の家（ルート）を知っているか」をチェック。
*   **注意点**: ラボ試験では、設定自体は簡単ですが、**非対称ルーティング**というトラップに気づかずに `rx` を設定して通信を全断させてしまうミスが最も危険です。必ずトポロジのパスを確認してください。
