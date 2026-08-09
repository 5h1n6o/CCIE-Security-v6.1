---
layout: default
title: 3.4.a-DAI
nav_order: 3
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.4.a DAI (Dynamic ARP Inspection)

**DAI (Dynamic ARP Inspection)** は、ネットワーク内の ARP トラフィックを検証し、不正な ARP パケットを破棄することで、**ARP スプーフィング（なりすまし）**や中間者攻撃（Man-in-the-Middle）を防止するレイヤ 2 セキュリティ機能です,。DAI は主にアクセススイッチで実装され、DHCP Snooping によって構築されたバインディングデータベースを利用してパケットの正当性を判断します。

---

## 📘 概要

*   **機能概要**: 受信した ARP パケットの送信元 MAC アドレスと送信元 IP アドレスの組み合わせをチェックし、正当なバインディングと一致しないパケットをドロップします。
*   **利用目的**: ARP キャッシュポイズニング攻撃を未然に防ぎ、通信の盗聴や改ざんを防止します。
*   **どのような場面で利用するか**: 不特定多数の端末が接続されるエンタープライズのアクセス層ポートに適用します。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | インターフェイスを Trust（信頼）と Untrust（非信頼）に分類する。 |
| **用途** | ARP スプーフィング攻撃の防御。 |
| **メリット** | 同一 VLAN 内の通信を L2 レイヤで保護できる。 |
| **前提条件** | **DHCP Snooping** の有効化、または手動の ARP ACL 定義が必要。 |
| **対応機種** | Catalyst スイッチ全般、および FTD（一部構成）。 |
| **制限事項** | ハードウェアリソース（CPU）を消費するため、ARP パケットのレート制限が必要。 |
| **設計上の注意点** | Uplink ポート（ルータや上位スイッチ接続ポート）は必ず **Trust** に設定する。 |

---

## 🏗 動作原理

DAI は、スイッチのインターフェイスに入力される ARP パケットをインターセプトし、内部のデータベースと照合します。

```text
[ Client A ] --- (ARP Reply: IP=B, MAC=A) ---> [ Access Switch (DAI) ]
                                                   ↓
                                         [ Binding Database Check ]
                                         (IP=B is actually MAC=B?)
                                                   ↓
                                         [ Result: Discard / Drop ]
```

*   **Trusted Port**: 照合を行わずに ARP パケットを通過させます。通常、他のスイッチやルータに接続するポートに設定します。
*   **Untrusted Port**: すべての ARP パケットを検証対象とします。デフォルトはすべてのポートが Untrusted です。

---

## ⚙ 動作シーケンス

1.  **パケット受信**: Untrusted インターフェイスで ARP パケットを受信。
2.  **データベース照合**: ARP ヘッダー内の送信元 IP/MAC を、DHCP Snooping バインディングテーブル、またはスタティックに設定された ARP ACL と照合。
3.  **妥当性判定**: 
    *   一致する場合：転送。
    *   一致しない場合：破棄し、必要に応じて Syslog を生成。
4.  **レート制限チェック**: 設定された ARP パケットの到着レートを超えていないか確認。超えた場合はポートを `err-disable` 状態にする。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **依存関係**: 「DAI を設定せよ」という要件に対し、前提となる **DHCP Snooping** が設定されているか、または静的な `ip source binding` が必要かを見極める必要があります。
*   **Trust 設定の漏れ**: Uplink ポートを Trust にし忘れると、上位デバイスからの正当な ARP もドロップされ、ネットワーク全体がダウンします。
*   **ARP ACL**: 固定 IP アドレスを持つサーバーやプリンタなどのデバイスがある場合、それらを許可するための ARP ACL を作成し、DAI に関連付ける必要があります。
*   **検証項目**: `show ip arp inspection` コマンドでドロップ数や設定 VLAN を即座に確認できるようにしてください。
*   **ログ要件**: 特定の攻撃を検知した際にログを出す、またはレート制限による `err-disable` からの自動復旧設定（`errdisable recovery`）が問われることがあります。

---

## 🛠 設定方法

※以下の設定は、一般的な Cisco Catalyst IOS-XE の例です（外部資料に基づく標準的な実装）。

### 1. DHCP Snooping の有効化（前提）
```bash
ip dhcp snooping
ip dhcp snooping vlan 10
```

### 2. DAI の有効化
```bash
ip arp inspection vlan 10
```

### 3. インターフェイスの Trust 設定
```bash
interface GigabitEthernet0/1
 description Uplink-to-Core
 ip arp inspection trust
```

### 4. 静的デバイス用の ARP ACL
```bash
ip access-list arp STATIC-DEVICES
 permit ip host 192.168.1.100 mac host 0011.2233.4455
!
ip arp inspection filter STATIC-DEVICES vlan 10
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **全体の状態確認** | <code>show ip arp inspection</code> |
| **VLAN ごとの統計** | <code>show ip arp inspection vlan 10</code> |
| **インターフェイスの状態** | <code>show ip arp inspection interfaces</code> |
| **DHCP バインディング確認** | <code>show ip dhcp snooping binding</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 正当な端末が通信できない | DHCP Snooping DB にエントリがない | <code>show ip dhcp snooping binding</code> | 端末の IP 再取得、またはスタティック登録。 |
| ポートが Down した | ARP パケットの流量超過 | <code>show interface status err-disabled</code> | <code>ip arp inspection limit</code> の調整。 |
| 特定 VLAN でのみ DAI が効かない | VLAN 指定の漏れ | <code>show ip arp inspection</code> | <code>ip arp inspection vlan [ID]</code> を追加。 |
| 全通信が停止した | Uplink ポートの Trust 設定漏れ | <code>show ip arp inspection interfaces</code> | Uplink IF で <code>trust</code> を設定。 |

---

## ⚠ 制限事項

*   **CPU 負荷**: ARP パケットは CPU（コントロールプレーン）で処理されるため、大量の ARP を受けるとデバイス全体のパフォーマンスに影響します。
*   **DHCP 依存**: 基本的に DHCP 環境を前提としています。固定 IP 環境では管理オーバーヘッドが増大します。
*   **非 IP パケット**: DAI は ARP (IPv4) を対象とします。IPv6 の場合は ND Inspection などの機能を使用します。

---

## 🔄 他技術との関連

*   **DHCP Snooping**: DAI の判断基準となるデータベースを提供します。
*   **Port Security**: MAC アドレスレベルの制限。DAI と併用してポートの信頼性を高めます。
*   **IP Source Guard (IPSG)**: データパケットの IP/MAC を検証。DAI が ARP パケットを保護するのに対し、IPSG は通常の IP トラフィックを保護します。
*   **Err-disable Recovery**: DAI のレート制限による自動シャットダウンからの復旧に使用します。

---

## 🧩 比較表

### DAI vs IP Source Guard

| 特徴 | DAI | IP Source Guard |
| :--- | :--- | :--- |
| **検証対象** | **ARP パケット** (L2) | **IP トラフィック** (L3) |
| **目的** | ARP スプーフィングの防止 | IP スプーフィングの防止 |
| **データベース** | DHCP Snooping / ARP ACL | DHCP Snooping / IP Source Binding |

---

## 💡 ベストプラクティス

1.  **段階的導入**: 最初は `ip arp inspection vlan [ID] logging` のみで正当なドロップがないか確認します。
2.  **レート制限の最適化**: バースト的な通信を考慮し、デフォルトの 15 pps から適切な値に調整します。
3.  **Uplink の徹底した Trust**: スイッチ間のトランクポートやルータ接続ポートは必ず Trust にします。
4.  **バインディングの保存**: スイッチ再起動時にバインディングが消えないよう、Flash や外部 TFTP サーバーに DB を保存する設定を検討します。

---

## 📝 ラボ学習・設定サンプル例

1.  **基本設定**: VLAN 10 で DAI を有効にし、Gi0/1 を Trust にせよ。
2.  **レート制限の変更**: アクセスポート Gi0/2 の ARP レートを 20 pps に制限せよ。
3.  **静的 ARP 許可**: 10.1.1.50 (MAC: aaaa.bbbb.cccc) の固定 IP デバイスを例外許可せよ。
4.  **ロギングのカスタマイズ**: 無効なパケットを 5 秒に 1 回のみログ出力するようにせよ。
5.  **バインディングチェックの追加**: `dst-mac` の検証も追加してセキュリティを強化せよ（`ip arp inspection validate dst-mac`）。
6.  **Err-disable 復旧**: DAI 違反による停止を 300 秒後に自動復旧させよ。
7.  **SVI インターフェイス**: 管理用 SVI IP 宛の ARP が保護されているか確認せよ。
8.  **マルチ VLAN 構成**: 複数 VLAN (10, 20) で一括有効化せよ。
9.  **DHCP Snooping 連携**: DB にエントリがあるか `show` コマンドで確認せよ。
10. **スプーフィング攻撃のシミュレーション**: 攻撃パケットを生成し、`show ip arp inspection statistics` でドロップを確認せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: あるスイッチの Uplink ポートで `ip arp inspection trust` が設定されていない。どのような不具合が発生するか？
    *   **回答**: 上位デバイスからの ARP 応答がすべて破棄され、スイッチ配下の端末がデフォルトゲートウェイ等の MAC アドレスを学習できなくなる。
2.  **トラブルシュート**: DAI 有効化後、固定 IP のプリンタが通信不能になった。原因と対策は？
    *   **回答**: DHCP Snooping DB に情報がないため。対策は ARP ACL を作成し、`ip arp inspection filter` で適用する。
3.  **Design**: DAI の CPU 負荷を最小限に抑えつつ、ARP スプーフィングを防止する設計上の工夫は？
    *   **回答**: 各ポートに適切な `limit rate` を設定し、信頼できる Uplink のみを `trust` に限定する。
4.  **実装**: `ip arp inspection validate src-mac` コマンドの効果は何か？
    *   **回答**: ARP パケット内の送信元 MAC アドレスと、L2 ヘッダー内の送信元 MAC アドレスが一致しているかを追加で検証する。
5.  **コンフィグ読解**: `ip arp inspection vlan 10,20` が設定されている環境で VLAN 30 の端末が ARP 偽装を行った。DAI はこれを阻止できるか？
    *   **回答**: 阻止できない。DAI は VLAN 単位で有効化する必要があるため、VLAN 30 には適用されていない。

---

## 🔗 参考リソース

*   **Cisco Configuration Guide**
    *   [Configuring Dynamic ARP Inspection (Cisco.com)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3750/software/release/12-2_55_se/configuration/guide/scg3750/swdynarp.html)
*   **Cisco Learning Matrix**
    *   [CCIE Security v6.1 Learning Matrix](https://learningnetwork.cisco.com/s/article/ccie-security-v6-1-learning-matrix)
*   **Technical Notes**
    *   [Understanding and Configuring Dynamic ARP Inspection (DAI)](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-3750-series-switches/72804-cat3750-da-inspection.html)

---

## 📝 **補足（Notes）**
- **学習メモ**: 「DAI は DHCP Snooping の弟」のような関係です。兄（DHCP Snooping）が名簿（DB）を作り、弟（DAI）がその名簿を玄関（ポート）でチェックするイメージで覚えると分かりやすいです。
- **注意点**: ラボ試験では、複数の L2 セキュリティ技術（DAI, DHCP Snooping, Port Security, IPSG）が同時に要求されることが多いため、それぞれの設定順序と依存関係を整理しておくことが合格への近道です。
