---
layout: default
title: 1.7.d-MITM
nav_order: 4
parent: 1.7-Attack-detection
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.7.d Man-in-the-middle

**Man-in-the-middle (MiM) 攻撃**は、攻撃者が通信している2者間に密かに割り込み、通信を傍受または改ざんする攻撃手法です。CCIE Security v6.1 においては、レイヤ2（L2）の脆弱性を突いた攻撃（ARPスプーフィング等）の緩和策と、暗号化通信に対するインスペクション（SSL復号）による検知が重要となります。

---

## 📘 概要

*   **機能概要**: 攻撃者が正規のホストやゲートウェイになりすますことで、パケットを自分自身を経由させる攻撃を阻止・検知します。
*   **利用目的**: 通信内容の盗聴、セッションハイジャック、データの改ざんを防止し、ネットワークの完全性と機密性を維持します。
*   **どのような場面で利用するか**: 同一VLAN内のクライアント間通信、またはクライアントとデフォルトゲートウェイ間の通信保護。特に、パブリックなアクセスエリアやセグメンテーションが不十分な社内ネットワークで必須となります。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な攻撃手法** | **ARP Poisoning (Spoofing)**, DHCP Spoofing, DNS Poisoning, ICMP Redirect. |
| **防御の三種の神器** | **DHCP Snooping**, **Dynamic ARP Inspection (DAI)**, **IP Source Guard (IPSG)**. |
| **IPv6対策** | **RA Guard**, IPv6 Snooping, IPv6 Source Guard. |
| **メリット** | L2レベルでのなりすましを物理的に遮断。高精度な信頼性確保。 |
| **デメリット** | 設定ミスにより正当な端末の通信が切断されるリスク。CPU負荷の増大。 |
| **対応機種** | Catalyst Switch, IOS-XE Router, Cisco Secure Firewall (FTD). |
| **設計上の注意点** | 信頼境界（Trusted vs Untrusted）の明確な定義. |

---

## 🏗 動作原理

MiM 攻撃（特に ARP スプーフィング）の緩和は、スイッチが「正しい IP と MAC の紐付け」を学習・保持し、それと矛盾するパケットを破棄することで実現します。

```text
[ Attacker ] (MAC: CCC) sends Fake ARP Reply: "I am Gateway (10.1.1.1)"
         ↓
[ Switch ] <--- (DAI/DHCP Snooping Check)
         │       Verification: "Does IP 10.1.1.1 belong to MAC CCC?"
         │       Result: NO (Expected MAC AAA)
         ↓
[ Drop Packet & Log ]
```

---

## ⚙ 動作シーケンス

1.  **DHCP Snooping による学習**: スイッチが DHCP 通信を監視し、どのポートにどの MAC と IP が割り当てられたかの **Snooping Binding Table** を作成します。
2.  **ARP パケットの着信**: クライアントから ARP パケット（Request/Reply）が届きます。
3.  **DAI による検証**: スイッチは ARP パケット内の情報を Binding Table と照合します。
4.  **フィルタリング**:
    *   **一致**: 通信を許可。
    *   **不一致**: パケットを破棄し、`err-disabled` にするかログを生成します。
5.  **IPv6 環境 (RA Guard)**: ルーター以外からの不正なルーター広告 (RA) を受信した際、即座に破棄してクライアントが攻撃者をゲートウェイとして認識するのを防ぎます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Blueprint の重要ポイント**: MiM 対策は、L2 セキュリティ機能（DAI, DHCP Snooping, Port Security）の組み合わせ問題として頻出です。
*   **ラボ試験で設定させられそうな内容**:
    *   特定の VLAN における **DHCP Snooping** の有効化。
    *   アップリンクポート（ルータ/サーバ接続側）の **Trust** 設定の適用。
    *   **DAI (Dynamic ARP Inspection)** による ARP スプーフィングの阻止。
    *   **RA Guard** による IPv6 MiM 攻撃の緩和。
*   **よくある設定ミス**:
    *   `ip dhcp snooping trust` をトランクポートやルータポートに入れ忘れ、クライアントが IP を取得できなくなる。
    *   DHCP Snooping を有効にしたが、`ip dhcp snooping vlan [ID]` を入れ忘れて機能していない。
*   **show コマンドからの判断**:
    *   `show ip dhcp snooping binding` で正しいエントリが存在するか確認。
    *   `show ip arp inspection statistics` でドロップ数が増えているか確認。

---

## 🛠 設定方法

### 1. DHCP Snooping と DAI の連携 (IOS-XE)
```bash
! DHCP Snoopingの有効化
ip dhcp snooping
ip dhcp snooping vlan 10
! オプション82を無効化（必要な場合）
no ip dhcp snooping information option

! 信頼されるポート（ルータ/サーバ側）
interface GigabitEthernet0/1
 ip dhcp snooping trust
 ip arp inspection trust

! DAIの有効化
ip arp inspection vlan 10
```

### 2. IP Source Guard (IPSG) の設定
```bash
interface GigabitEthernet0/2
 ! IPとMACの両方を検証
 ip verify source mac-check
```

### 3. IPv6 RA Guard の設定
```bash
ipv6 nd ra-guard policy LIMIT-RA
 device-role host
!
interface GigabitEthernet0/2
 ipv6 nd ra-guard attach-policy LIMIT-RA
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **DHCPバインディングの確認** | `show ip dhcp snooping binding` |
| **DAIのステータスと統計** | `show ip arp inspection vlan [ID]` |
| **IPSGの適用状態確認** | `show ip verify source` |
| **RA Guardの適用確認** | `show ipv6 ra-guard` |
| **パケットドロップ履歴** | `show logging` |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| クライアントがIPを取得できない | アップリンクが `trust` ではない | `ip dhcp snooping trust` を追加。 |
| 静的IP端末が通信できない | バインディングDBにエントリがない | `arp access-list` で手動エントリを作成し適用。 |
| 通信が頻繁に途切れる | ARPレート制限を超過 | `ip arp inspection limit` を調整。 |
| IPv6クライアントがGWを見失う | RA Guardがルータ側で有効 | ルータポートの役割を `router` に設定。 |

---

## ⚠ 制限事項

*   **バインディングDBの永続性**: スイッチ再起動時にバインディング情報が消えるため、外部フラッシュや FTP サーバへの保存設定（`ip dhcp snooping database`）が推奨されます。
*   **CPU負荷**: DAI はパケットを CPU で検査するため、大量の ARP 攻撃を受けると CPU 負荷が上昇します。
*   **非IPトラフィック**: ARP/IP 以外の L2 通信（非 IP プロトコル）に対する MiM はこれらでは防げません。

---

## 🔄 他技術との関連

*   **Access Control**: FTD のアクセスコントロールポリシーで、なりすまし元 IP からの通信を遮断。
*   **SSL Inspection**: SSL 復号を行うことで、暗号化通信の中に隠れた MiM スクリプトや攻撃コードを検知。
*   **Cisco ISE**: 802.1X 認証により、デバイスがネットワークに参加する前にアイデンティティを確認し、未承認端末による MiM を根源から断ちます。

---

## 🧩 比較表

### DAI vs IP Source Guard

| 特徴 | Dynamic ARP Inspection (DAI) | IP Source Guard (IPSG) |
| :--- | :--- | :--- |
| **主な対象** | ARP パケット (L2) | IP パケット (L3) |
| **検証内容** | 送信元 MAC と IP の整合性 | 送信元 IP アドレスの正当性 |
| **目的** | ARP スプーフィングの阻止 | IP スプーフィングの阻止 |
| **前提条件** | DHCP Snooping | DHCP Snooping |

---

## 💡 ベストプラクティス

1.  **Trust の最小化**: ルータや DHCP サーバが接続されているポートのみを `trust` とし、エッジポートはすべて `untrusted` にします。
2.  **Port Security との併用**: MAC アドレスの学習数を制限することで、MiM の前段階となる MAC Flooding 攻撃を防止します。
3.  **iACL の適用**: 外部からのなりすまし IP パケットをエッジルータでドロップします。

---

## 📝 ラボ学習・設定サンプル例

### 1. ARP スプーフィングの完全遮断
*   **要件**: VLAN 100 で DAI を有効化し、未登録 ARP をすべてドロップせよ。
*   **設定**: `ip arp inspection vlan 100`。
*   **検証**: `show ip arp inspection statistics`。

### 2. 静的 IP サーバーの除外
*   **要件**: 静的 IP (10.1.1.10) を持つサーバーが DAI でブロックされないようにせよ。
*   **設定**: `arp access-list STATIC-SERVER` で `permit ip host 10.1.1.10 mac host XXXX...` を定義し、`ip arp inspection filter` で適用。

### 3. RA Guard によるルータなりすまし防止
*   **要件**: ユーザーポート G0/2 からの IPv6 ルーター広告を無効化せよ。
*   **設定**: `ipv6 nd ra-guard attach-policy`。

### 4. DHCP Snooping Database の保存
*   **要件**: スイッチ再起動後もバインディングテーブルを保持せよ。
*   **設定**: `ip dhcp snooping database flash:snoop.db`。

### 5. DAI レート制限の設定
*   **要件**: 1秒間に 20 以上の ARP パケットを送るポートをシャットダウンせよ。
*   **設定**: `ip arp inspection limit rate 20`。

### 6. SSL 復号による MiM 検知
*   **要件**: FTD で HTTPS 通信を復号し、不正な証明書を検知せよ。
*   **設定**: SSL Policy で `Decrypt - Resign` アクションを構成。

### 7. VACL によるトラフィック制御
*   **要件**: 同一 VLAN 内の特定のホスト間通信を VACL で禁止し、MiM の横展開を防げ。
*   **設定**: `vlan access-map` と `vlan filter`。

### 8. Port Security と DAI の組み合わせ
*   **要件**: MAC 学習数を1に制限し、かつ DAI を適用せよ。
*   **設定**: `switchport port-security` ＋ `ip arp inspection`。

### 9. 管理プレーンの保護 (MPP)
*   **要件**: 特定のインターフェイス以外からの SSH 管理アクセスを拒否せよ。
*   **設定**: `control-plane` 下で `management-interface` を制限。

### 10. IPv6 Snooping の実装
*   **要件**: IPv6 の Neighbor Discovery 通信を監視し、Binding Table を作成せよ。
*   **設定**: `ipv6 snooping vlan [ID]`。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `ip dhcp snooping trust` が物理インターフェイスに設定されているが、通信ができない。VLAN 設定を確認すべき理由は？
    *   **解答**: `ip dhcp snooping vlan [ID]` がグローバルで設定されていないと、バインディングテーブルが作成されず DAI で全ドロップされるため。
2.  **トラブルシュート**: クライアントがスタティック IP を設定した途端、ゲートウェイに ping が飛ばなくなった。DAI 稼働環境での原因は？
    *   **解答**: スタティック IP 端末は DHCP メッセージを投げないため Binding Table に載らず、DAI によって ARP が拒否される。
3.  **Design**: IPv6 環境で攻撃者が偽のルータとして動作するのを防ぐために、アクセスポートに適用すべき機能は？
    *   **解答**: IPv6 RA Guard。
4.  **実装**: 攻撃者が大量の偽 ARP を送り、スイッチの CPU が 100% になった。緩和するために設定すべきコマンドは？
    *   **解答**: `ip arp inspection limit rate [value]` によるレート制限。
5.  **コンフィグ読解**: `ip verify source mac-check` の設定がある場合、パケットのどのフィールドが検証されるか？
    *   **解答**: 送信元 IP アドレスと送信元 MAC アドレスの両方が DHCP バインディングまたは静的エントリと照合される。

---

## 🔗 参考リソース

*   **Configuration Guide**:
    *   [Cisco Catalyst 9000 Series - Configuring Dynamic ARP Inspection](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/16-6/configuration_guide/sec/b_166_sec_9300_cg/configuring_dynamic_arp_inspection.html)
    *   [IPv6 First Hop Security Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipv6/configuration/xe-3s/ipv6-xe-3s-book/ip6-fhs.html)
*   **Technical Notes**:
    *   [Securing the Data Plane - Mitigating Man-in-the-Middle Attacks](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-3750-series-switches/113685-asa-threat-detection-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: MiM 対策は「点」ではなく「面（VLAN全体）」で考える必要があります。DHCP Snooping が正しく動いていないと、その上の DAI や IPSG はすべて誤動作するため、トラブルシュート時は必ずレイヤの低い順（Snooping -> DAI -> IPSG）に確認してください。
*   **注意点**: ラボ試験では、トランクポートの `trust` 設定を忘れがちです。トランクポートは複数の VLAN トラフィックが流れるため、ここが `untrusted` だと被害が全 VLAN に及び、全通信が停止します。
