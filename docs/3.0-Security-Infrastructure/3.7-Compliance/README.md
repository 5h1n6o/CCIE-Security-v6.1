---
layout: default
title: 3.7-Compliance
nav_order: 7
parent: 3.0-Security-Infrastructure
---

# 3.7 Security features to comply with organizational security policies, procedures, and standards BCP 38

**BCP 38 (Best Current Practice 38)** は、インターネットにおける **IPソースアドレスのなりすまし（IP Spoofing）** を防止するための標準的な手法を定義したガイドライン（**RFC 2827**）です。CCIE Security v6.1 においては、この BCP 38 を実装するための具体的な技術（**uRPF**, **ACL**, **IP Source Guard** など）と、それが **ISO 27001** や **PCI-DSS** といった組織的なセキュリティコンプライアンスにどのように寄与するかを理解することが求められます。

---

## 📘 概要

*   **機能概要**: ネットワークの境界（エッジ）において、パケットの送信元 IP アドレスがそのインターフェイスから到達可能な正しい範囲に属しているかを検証し、不正なパケットを破棄します。
*   **利用目的**: DDoS 攻撃（特に増幅反射攻撃）の踏み台になることを防ぎ、ネットワークの整合性を維持します。
*   **利用場面**:
    *   **エンタープライズのエッジ**: 内部ネットワークから外部へ「なりすまし」パケットが流出するのを防ぐ。
    *   **サービスプロバイダーのエッジ**: 顧客からのトラフィックが許可された IP 範囲内であることを保証する。
    *   **コンプライアンス対応**: PCI-DSS や ISO 27001 で要求される「境界保護」と「アクセス制御」のベストプラクティスとして実装する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な技術** | **uRPF (Unicast Reverse Path Forwarding)**, **Ingress ACL**, **IP Source Guard**。 |
| **RFC 2827 / BCP 38** | インターネットサービスプロバイダーが顧客からの不正なソース IP をフィルタリングすることを提唱。 |
| **ISO 27001 関連** | 情報セキュリティマネジメントシステム（ISMS）におけるネットワーク管理の管理策に該当。 |
| **PCI-DSS 関連** | ファイアウォール設定、IP スプーフィング防止の要件に直結。 |
| **メリット** | DDoS 攻撃の抑制、ネットワークの透明性向上。 |
| **デメリット** | 非対称ルーティング環境での誤検知（uRPF Strict モード時）。 |

---

## 🏗 動作原理

BCP 38 の中核技術である **uRPF** は、ルーターがパケットを受信した際、その「送信元 IP アドレス」をルーティングテーブル（FIB）と照合して検証します。

```text
[ Packet Arrival ]
      ↓
[ Check Source IP in FIB ]
      ↓
[ Is the incoming interface the same as the egress interface for this source? ]
      ├─ YES ─→ [ Permit Packet ] (Strict Mode)
      └─ NO  ─→ [ Check if Source exists in FIB ]
                  ├─ YES ─→ [ Permit Packet ] (Loose Mode)
                  └─ NO  ─→ [ Drop Packet ]
```

---

## ⚙ 動作シーケンス

1.  **パケット受信**: ルーターがインターフェイスでパケットを受信します。
2.  **逆パス参照**: ルーターは受信パケットの **送信元 IP** をキーにして FIB（Forwarding Information Base）を検索します。
3.  **検証（Mode 依存）**:
    *   **Strict Mode**: 送信元 IP への最適な戻りパスが、パケットを受信したインターフェイスと一致するかを確認します。
    *   **Loose Mode**: 送信元 IP がルーティングテーブル内に存在するか（到達可能か）のみを確認します。
4.  **処理**: 検証に失敗した場合、パケットをドロップします。成功した場合は通常のデータプレーン処理へ進みます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **uRPF モードの選択**: 「非対称ルーティングが発生しないエッジでは **Strict**」、「マルチホーム環境などパスが変動する場合は **Loose**」という使い分けを判断させられます。
*   **RFC 1918 フィルタリング**: ラボ要件で「外部からプライベートアドレスの流入を防げ」とあれば、ACL を作成し `log` オプションを付けて適用するシナリオが頻出です。
*   **デフォルトルートの考慮**: `allow-default` オプションの有無による挙動の違いを確認してください。これが無いと、デフォルトルートしかないソースは Strict モードで落ちます。
*   **Infrastructure ACL (iACL)**: ネットワーク機器自体の保護（Management/Control Plane）のために、エッジで特定の管理トラフィック以外を遮断する設定が求められます。
*   **検証**: `show ip interface` で uRPF のドロップ数を確認できる必要があります。

---

## 🛠 設定方法

### 1. uRPF Strict モードの実装（エッジルーター）
```bash
interface GigabitEthernet0/1
 description Internet-Facing-Interface
 ! 送信元がこのIFから到達可能かつ最適なパスであるか確認
 ip verify unicast source reachable-via rx
```

### 2. uRPF Loose モードの実装（コア/マルチホーム）
```bash
interface GigabitEthernet0/2
 ! 送信元がFIBに存在するかのみ確認
 ip verify unicast source reachable-via any
```

### 3. Ingress フィルタリング ACL (RFC 2827 準拠)
外部から流入してはいけない送信元（自組織の IP やプライベート IP）を拒否します。
```bash
ip access-list extended BCP38-INGRESS
 deny ip 10.0.0.0 0.255.255.255 any log
 deny ip 172.16.0.0 0.15.255.255 any log
 deny ip 192.168.0.0 0.0.255.255 any log
 permit ip any any
!
interface GigabitEthernet0/1
 ip access-group BCP38-INGRESS in
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **uRPF の設定状態確認** | <code>show ip interface [Interface]</code> |
| **uRPF によるドロップ統計** | <code>show ip traffic</code> または <code>show cef drop</code> |
| **ACL のヒット数確認** | <code>show access-lists [Name]</code> |
| **IPv6 版 uRPF 確認** | <code>show ipv6 interface</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 正当なパケットがドロップされる | 非対称ルーティング | Strict モードを <code>Loose (any)</code> に変更検討。 |
| デフォルトルート経由の通信が不可 | オプション不足 | <code>allow-default</code> オプションを uRPF コマンドに追加。 |
| uRPF が全く動作しない | CEF が無効 | <code>ip cef</code> がグローバルで有効か確認。 |
| 特定ホストのみ繋がらない | ACL で例外未許可 | uRPF コマンド末尾に <code>[ACL]</code> を指定し例外を許可。 |

---

## ⚠ 制限事項

*   **CEF 必須**: uRPF は CEF (Cisco Express Forwarding) が有効でないと動作しません。
*   **パフォーマンス**: 大規模な FIB を持つルーターでの uRPF は CPU 負荷になる可能性があるため、ハードウェア転送がサポートされているか確認が必要です。
*   **トンネルインターフェイス**: GRE や IPsec トンネルでは、パケットの展開順序により uRPF の適用箇所に注意が必要です。

---

## 🔄 他技術との関連

*   **3.3.a uRPF**: データプレーン保護の直接的な技術。
*   **3.1.c iACLs**: 境界での不要なトラフィック遮断によるコントロールプレーン保護。
*   **3.4.e DHCP Snooping**: IP Source Guard と連携し、L2 レベルでのスプーフィングを防止。
*   **Stealthwatch (Monitoring)**: uRPF のドロップや ACL Log を NetFlow (NSEL) 経由で分析し、攻撃を可視化します。

---

## 🧩 比較表

### uRPF Strict vs Loose

| 特徴 | Strict Mode (rx) | Loose Mode (any) |
| :--- | :--- | :--- |
| **検証条件** | ソースへの戻りパスが受信 IF と一致 | ソースが FIB に存在すれば OK |
| **セキュリティ強度** | 最高 | 中 |
| **非対称ルーティング** | サポート不可（ドロップされる） | サポート可能 |
| **主な用途** | 顧客エッジ、シングルホーム | ISP コア、マルチホーム ISP |

---

## 💡 ベストプラクティス

1.  **エッジでの Strict 実装**: 可能な限りユーザーに近い場所で Strict モードを適用します。
2.  **RFC 1918 フィルタリングの徹底**: インターネット境界ではプライベートアドレスを明示的に拒否します。
3.  **Logging の活用**: ACL で拒否したパケットには `log` を付け、SIEM (eStreamer等) に送信します。
4.  **例外 ACL**: モバイル IP や特殊なルーティングが必要なホストには、uRPF の例外 ACL を活用します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な uRPF Strict の設定
*   **要件**: Gi0/1 で受信するパケットに対し、厳格なパス検証を行え。
*   **設定**: `ip verify unicast source reachable-via rx`。

### 2. 非対称パスを許容する Loose の設定
*   **要件**: マルチホーム環境の Gi0/2 で、ソース IP が到達可能なら許可せよ。
*   **設定**: `ip verify unicast source reachable-via any`。

### 3. デフォルトルートを許容する uRPF
*   **要件**: FIB 内のデフォルトルートを戻りパスとして認める Strict uRPF を設定せよ。
*   **設定**: `ip verify unicast source reachable-via rx allow-default`。

### 4. 特定の例外を許可するフィルタリング
*   **要件**: uRPF でドロップされるはずの特定の監視サーバー (1.1.1.1) を許可せよ。
*   **設定**: `access-list 100 permit ip host 1.1.1.1 any`, `ip verify unicast source reachable-via rx 100`。

### 5. RFC 1918 流入防止
*   **要件**: 外部 IF で送信元が 10.0.0.0/8 のパケットを拒否しログに記録せよ。

### 6. IPv6 での BCP 38 実装
*   **要件**: IPv6 トラフィックに対しても Strict uRPF を構成せよ。
*   **設定**: `ipv6 verify unicast source reachable-via rx`。

### 7. インフラ保護のための iACL
*   **要件**: エッジルーターで自装置の Loopback 宛の Telnet を 10.1.1.0/24 以外から拒否せよ。

### 8. IP Source Guard との併用
*   **要件**: L2 スイッチで、DHCP で割り当てられた IP 以外からの通信を遮断せよ。

### 9. VACL による L2 フィルタリング
*   **要件**: VLAN 10 内で特定の IP スプーフィングを防止せよ。

### 10. uRPF 統計の監視
*   **課題**: `show ip traffic` を定期的に実行し、攻撃によるドロップの急増を検知せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `ip verify unicast source reachable-via rx` が設定されたインターフェイスで、非対称ルーティングにより戻りパスが別のインターフェイスを指している場合、パケットはどうなるか？
    *   **回答**: ドロップされる。Strict モードでは受信インターフェイスと戻りパスの一致が必須であるため。
2.  **Design**: 大規模な ISP 網で、顧客からインターネットへの DDoS 攻撃を防ぐための最も効果的な「BCP 38」実装場所は？
    *   **回答**: 顧客トラフィックが最初に流入する **アクセス層（エッジ）ルーターの Ingress インターフェイス**。
3.  **トラブルシュート**: ルーターで uRPF を設定したが、統計（Drop count）が全く増えず、明らかに不正なパケットが通過している。何を確認すべきか？
    *   **回答**: **CEF** が有効になっているか（`show ip cef`）。CEF 無効環境では uRPF は機能しない。
4.  **コンフィグ読解**: `access-list 10 permit 192.168.1.0 0.0.0.255` を `ip verify ... any 10` のように適用した場合、この ACL はどのような役割を果たすか？
    *   **回答**: 例外（Whitelist）として機能し、ACL で permit されたソースは uRPF の検証に失敗しても許可される。
5.  **Design**: ISO 27001 のコンプライアンス監査において、IP スプーフィング対策がなされていることを証明するための資料は？
    *   **回答**: 境界ルーターにおける **uRPF 設定（running-config）** および **Ingress フィルタリング ACL のログ**。

---

## 🔗 参考リソース

*   **Cisco Live (BRKSEC-2003)**: [Securing the Management and Control Plane](https://www.ciscolive.com/)
*   **Cisco Configuration Guide**: [Configuring Unicast Reverse Path Forwarding](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_urpf/configuration/xe-16/sec-data-urpf-xe-16-book.html)
*   **Technical Notes**: [Understanding Unicast Reverse Path Forwarding](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/17445-bcp38.html)
*   **Design Guide**: [Cisco SAFE - Threat Identification and Mitigation](https://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/Security/SAFE_RG/SAFE_rg/chap3.html)

---

## 📝 **補足（Notes）**
*   **学習メモ**: 「BCP 38 は近所の不審者チェック」です。自分の家の門（インターフェイス）から入ってきた人が、本当にその道（パス）の先に住んでいる人かを確認する仕組みだと覚えましょう。
*   **注意点**: ラボ試験では `Strict` と `Loose` のキーワードの書き換え（`rx` vs `any`）でルーティングの要件（非対称か否か）をクリアする必要があります。
*   **図解**: 
    *   `reachable-via rx` = 送信元へ帰るならこの門（rx）から出る。
    *   `reachable-via any` = 送信元が誰かは知っている（どこかの門から出られる）。
