---
layout: default
title: 1.7.c-Spoofing
nav_order: 3
parent: 1.7-Attack-detection
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.7.c Spoofing

**スプーフィング（Spoofing：なりすまし）**は、攻撃者が正規の送信元（IPアドレス、MACアドレス、またはARP情報）を偽装してパケットを送信する攻撃手法です。これにより、アクセス制御の回避、中間者攻撃（MiM）、またはDos攻撃の隠蔽が行われます。CCIE Security v6.1 では、L2/L3およびファイアウォール（FTD/ASA）における多層的な緩和策の実装が求められます,。

---

## 📘 概要

*   **機能概要**: パケットのヘッダー情報（IP/MAC）が、ネットワークの実際のトポロジや既知の割り当て情報と一致するかを検証し、矛盾するものを破棄します,。
*   **利用目的**: 送信元偽装による不正アクセス防止、セッションハイジャックの阻止、およびネットワークの整合性維持。
*   **利用場面**:
    *   **IP Spoofing**: インターネット境界での外部からの内部IP偽装防止。
    *   **ARP Spoofing**: 同一VLAN内でのゲートウェイなりすまし（MiM）防止。
    *   **MAC Spoofing**: 許可された端末以外の接続防止。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な対象** | IPアドレス、MACアドレス、ARPメッセージ。 |
| **用途** | なりすまし防止、中間者攻撃（MiM）の緩和。 |
| **メリット** | ネットワークの「信頼の境界」を強固にし、攻撃経路を遮断できる。 |
| **デメリット** | 設定ミスにより、正当な非対称ルーティング環境で通信断が発生する。 |
| **対応機種** | Catalyst Switch, IOS-XE Router, ASA, FTD。 |
| **主要技術** | uRPF, DAI, DHCP Snooping, Port Security, IPDT,。 |
| **設計上の注意点** | DHCP Snooping バインディングデータベースの維持（再起動対策）。 |

---

## 🏗 動作原理

スプーフィング対策は、「パケットの属性」を「信頼できるデータベース」と比較することで動作します。

```text
[ Attacker ] (Source IP: 10.1.1.100, MAC: AAA) -- (Spoofed Packet) --> [ Switch/Router ]
                                                                          │
    [ 1. IP Validation (uRPF) ] <--- Check: Is Source IP reachable via this IF?
    [ 2. ARP Validation (DAI) ] <--- Check: Is MAC-IP pair in Snooping DB?
    [ 3. MAC Validation (Port Sec) ] <--- Check: Is this MAC allowed on this Port?
                                                                          │
[ Target ] <----------------------- (Filtered Traffic) -------------------┘
```

---

## ⚙ 動作シーケンス

1.  **トラフィック着信**: インターフェイスがフレーム/パケットを受信。
2.  **L2検証 (Port Security)**: 送信元MACアドレスがポートに許可されたものか確認。
3.  **L2/L3バインディング検証 (DAI)**: ARPパケット内のMACとIPの組み合わせを、DHCP Snoopingデータベースと照合。
4.  **L3検証 (uRPF)**: IPパケットの送信元IPをルーティングテーブル（FIB）と照合し、パケットが到着したインターフェイスからそのIPへの「戻りパス」があるか確認。
5.  **破棄/ロギング**: 検証に失敗したトラフィックをドロップし、必要に応じてSNMPトラップやログを生成。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **uRPF (Unicast Reverse Path Forwarding)**:
    *   **Strictモード**: 到着インターフェイスがベストパスである必要がある。
    *   **Looseモード**: 経路が存在すればよい。デフォルトルートの考慮 (`allow-default`) が重要。
*   **DAI (Dynamic ARP Inspection)**: `trust` インターフェイス（アップリンク側）の設定を忘れないこと。設定ミスはネットワーク全体の停止を招きます。
*   **DHCP Snooping**: DAIの前提条件。`no ip dhcp snooping information option` (Option 82) の設定が必要な場合がある。
*   **IPv6 FHS (First Hop Security)**: IPv6環境でのなりすまし防止として **RA Guard** や **IPv6 Snooping** の設定が問われます,。
*   **iACL (Infrastructure ACL)**: エッジでRFC 1918などのプライベートIPや、自組織のIPを外部から受信しないように設定。

---

## 🛠 設定方法

### 1. IP Spoofing 対策 (uRPF on IOS-XE)
```bash
interface GigabitEthernet1
 ip verify unicast source reachable-via rx allow-default
 ! rx = Strict mode, any = Loose mode
```

### 2. ARP Spoofing 対策 (DHCP Snooping + DAI)
```bash
ip dhcp snooping
ip dhcp snooping vlan 10
!
interface GigabitEthernet1/1
 ip dhcp snooping trust
!
ip arp inspection vlan 10
interface GigabitEthernet1/1
 ip arp inspection trust
```

### 3. MAC Spoofing 対策 (Port Security)
```bash
interface GigabitEthernet1/2
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **uRPF ドロップ統計確認** | <code>show ip interface [INT] \| include verification</code> |
| **DAI 統計確認** | <code>show ip arp inspection statistics</code> |
| **DHCP Snooping DB 確認** | <code>show ip dhcp snooping binding</code> |
| **Port Security 状態確認** | <code>show port-security interface [INT]</code> |
| **IPv6 RA Guard 確認** | <code>show ipv6 ra-guard</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 正当な通信が uRPF で落ちる | 非対称ルーティング環境 | <code>show ip route</code> | モードを <code>Loose</code> に変更。 |
| クライアントが ARP 解決できない | DAI が全パケットを拒否 | <code>show ip dhcp snoop bind</code> | DBが空の場合、Snooping設定を先に見直す。 |
| DHCP サーバから IP が取れない | アップリンクが <code>trust</code> ではない | <code>show ip dhcp snoop</code> | サーバ側IFを <code>trust</code> に設定。 |
| ポートが err-disabled になる | MAC 偽装または端末移動 | <code>show interface status</code> | Port Security 違反。原因除いて <code>shut/no shut</code>。 |

---

## ⚠ 制限事項

*   **uRPF Strict の制限**: マルチホーム接続や非対称ルーティング環境では、正常なパケットも破棄してしまいます。
*   **DAI のリソース消費**: CPU で ARP パケットを検査するため、大量の ARP 発生時に CPU 負荷が高まる可能性があります。
*   **静的 IP 端末**: DHCP を使わない端末は、DAI によって遮断されます。`arp access-list` による手動登録が必要です。

---

## 🔄 他技術との関連

*   **DHCP**: Snooping 機能を通じてバインディングデータベースを提供します。
*   **Routing**: uRPF は FIB (CEF) テーブルを参照して判断を行います。
*   **IPv6**: IPv4 の DAI/Snooping に相当する機能として、RA Guard や ND Inspection があります。
*   **TrustSec**: SGT を用いて、IP/MAC ではなくタグベースで「なりすまし不能」なセグメンテーションを提供します。

---

## 🧩 比較表

### uRPF Strict vs Loose

| 機能 | Strict Mode | Loose Mode |
| :--- | :--- | :--- |
| **チェック条件** | FIBに経路があり、且つ受信IFがベストパス | FIBに経路があればOK（どのIFでも可） |
| **推奨環境** | エッジ（シングルホーム） | コア、マルチホーム環境 |
| **非対称ルーティング** | サポート不可（ドロップされる） | サポート可能 |

---

## 💡 ベストプラクティス

1.  **DHCP Snooping の常時有効化**: DAI や IP Source Guard の基盤となるため、まずは Snooping を正しく組むのが鉄則です。
2.  **外部境界での uRPF**: ISP 境界ルータには少なくとも Loose モード、可能な箇所には Strict モードを適用します。
3.  **iACL の適用**: 自組織のネットワークアドレスを外部インターフェイスで「送信元」として受け取らないよう、ACL でフィルタリングします。
4.  **DB の永続化**: スイッチ再起動時にバインディング情報が消えないよう、外部 TFTP/Flash に DB を保存する設定を入れます。

---

## 📝 ラボ学習・設定サンプル例

### 1. uRPF による IP なりすまし緩和
*   **要件**: 外部からの偽装 IP パケットを防ぐため、R1 の G1 で Strict モードの uRPF を有効にせよ。
*   **設定**: `int g1`, `ip verify unicast source reachable-via rx`。

### 2. DAI による MiM 攻撃の防止
*   **要件**: VLAN 20 で ARP なりすましを防げ。
*   **設定**: `ip arp inspection vlan 20`, トランクポートで `ip arp inspection trust`。

### 3. IP Source Guard の実装
*   **要件**: ポート G1/0/1 で MAC+IP の組み合わせを固定せよ。
*   **設定**: `int g1/0/1`, `ip verify source mac-check`。

### 4. 静的 IP 端末の DAI 許可
*   **要件**: 静的 IP `10.1.1.5` を持つサーバの通信を DAI 環境で許可せよ。
*   **設定**: `arp access-list`, `permit ip host 10.1.1.5 mac host XXXX...`, `ip arp inspection filter`。

### 5. IPv6 RA Guard の設定
*   **要件**: ユーザポートからの不正なルータ広告をブロックせよ。
*   **設定**: `ipv6 nd ra-guard attach-policy [NAME]`。

### 6. RFC 1918 フィルタリング (iACL)
*   **要件**: 外部境界でプライベート IP を送信元とするパケットを破棄せよ。
*   **設定**: `access-list 100 deny ip 10.0.0.0 0.255.255.255 any` ... `permit ip any any`。

### 7. DHCP Snooping Rate Limit
*   **要件**: DHCP サーバへの Dos 攻撃を防ぐため、受信レートを制限せよ。
*   **設定**: `int g1/0/1`, `ip dhcp snooping limit rate 10`。

### 8. Port Security の MAC 固定 (Sticky)
*   **要件**: 最初に接続された端末の MAC を永続的に学習せよ。
*   **設定**: `switchport port-security mac-address sticky`。

### 9. uRPF とデフォルトルートの併用
*   **課題**: デフォルトルート経由の戻りパスしかない環境で uRPF を有効にせよ。
*   **設定**: `ip verify unicast source reachable-via any allow-default`。

### 10. FTD でのアンチスプーフィング
*   **要件**: FTD において IP 偽装を防御せよ。
*   **設定**: デバイス設定の **Anti-Spoofing** オプションで、インターフェイスごとのネットワークを定義。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `ip verify unicast source reachable-via rx` が設定されたルータで、非対称ルーティングにより戻りパケットが別の IF から来た場合、どうなるか？
    *   **回答**: ルータは「ベストパスではない IF から来た」と判断し、パケットをドロップする。
2.  **トラブルシュート**: DAI を有効にした後、全ての通信が止まった。`show ip dhcp snooping binding` が空である場合に疑うべき箇所は？
    *   **回答**: DHCP Snooping が有効になっていないか、あるいは DHCP サーバ宛のポートが `trust` 設定されていない。
3.  **Design**: L2 ネットワークで中間者（MiM）攻撃を防ぐために組み合わせるべき 3 つの主要技術は？
    *   **回答**: DHCP Snooping, Dynamic ARP Inspection (DAI), IP Source Guard。
4.  **実装**: Port Security で違反があった際、ポートを閉じずにログだけ出す設定を述べよ。
    *   **回答**: `switchport port-security violation restrict`。
5.  **コンフィグ読解**: iACL で `deny ip 172.16.0.0 0.15.255.255 any log` が設定されている目的を説明せよ。
    *   **回答**: インターネットなどの外部境界において、送信元がプライベート IP である偽装パケット（IP Spoofing）を検知・遮断し、ログに残すため。

---

## 🔗 参考リソース

*   **Cisco Configuration Guides**
    *   [Cisco Catalyst 9300 Series - Configuring Dynamic ARP Inspection](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/16-6/configuration_guide/sec/b_166_sec_9300_cg/configuring_dynamic_arp_inspection.html)
    *   [Cisco IOS-XE - Unicast Reverse Path Forwarding](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_urpf/configuration/xe-16/sec-data-urpf-xe-16-book.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [L2 Security Best Practices](https://www.ciscolive.com/on-demand/on-demand-library.html)
*   **Technical Notes**
    *   [Securing the Data Plane on Cisco ASA and FTD](https://www.cisco.com/c/en/us/support/docs/security/firepower-management-center/212356-understand-firepower-deployment-modes.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: スプーフィング対策は「L2（MAC/ARP）」と「L3（IP）」に分けて整理しましょう。ラボでは、特定の攻撃シナリオ（例：MiM を防げ）に対して、複数の機能を組み合わせて実装する能力が試されます。
*   **注意点**: DAI や DHCP Snooping を設定する際は、必ず **トランクポートやルータ接続ポートを `trust` にする** ことを忘れないでください。これを忘れると、正規の通信まで全てスプーフィングとみなされて遮断されます。
