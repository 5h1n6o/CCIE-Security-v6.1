---
layout: default
title: 3.4.e-DHCP-snooping
nav_order: 5
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.0 Security Infrastructure
# 3.4.e DHCP snooping

**DHCP Snooping**は、レイヤ2のセキュリティ機能であり、ネットワーク内の不正なDHCPサーバーによる攻撃を防ぎ、IPアドレスの割り当て情報を管理する基盤技術です。CCIE Security v6.1の試験範囲において、この機能は単体での動作だけでなく、**DAI (Dynamic ARP Inspection)**や**IP Source Guard (IPSG)**が動作するための「バインディング・データベース」を提供する極めて重要な役割を担っています。

---

## 📘 概要

*   **機能概要**: スイッチを通過するDHCPメッセージを監視し、信頼できないポートからの不正なDHCPレスポンス（DHCPOFFER, DHCPACKなど）を遮断します。
*   **利用目的**: 不正なDHCPサーバーがクライアントに誤ったデフォルトゲートウェイやDNS情報を配布する「DHCPスプーフィング攻撃」の防止。
*   **どのような場面で利用するか**: エンタープライズのアクセス層において、ユーザー端末が接続されるエッジポートからの悪意ある、あるいは設定ミスによるDHCPサーバー機能の無効化に必須です。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | インターフェイスをTrust（信頼）とUntrust（非信頼）に分類して管理する。 |
| **用途** | 不正DHCPサーバー排除、DHCP飢餓攻撃（Starvation）の緩和。 |
| **メリット** | DAIやIPSGの前提条件となるIP/MAC/ポートの紐付け表を自動生成する。 |
| **デメリット** | CPU負荷の増大（パケットのインターセプトが発生するため）。 |
| **対応機種** | Catalystスイッチ、Nexusスイッチ、および一部のFTDデバイス。 |
| **制限事項** | ハードウェアリソースによりバインディングエントリ数に上限がある。 |
| **設計上の注意点** | Uplink（DHCPサーバーへのパス）は必ずTrustに設定する必要がある。 |

---

## 🏗 動作原理

DHCP Snoopingは、スイッチのポートを以下の2種類に分類して動作します。

1.  **Trusted Port (信頼できるポート)**:
    *   DHCPサーバーやリレーエージェントが接続されているポート。
    *   すべてのDHCPメッセージ（サーバーからのレスポンスを含む）の通過を許可します。
2.  **Untrusted Port (信頼できないポート)**:
    *   エンドユーザーや一般的な端末が接続されているポート。
    *   **DHCPサーバー側のメッセージ（OFFER, ACK, NAK）を受信した瞬間にドロップ**します。
    *   クライアント側のメッセージ（DISCOVER, REQUEST）のみを許可します。

```text
[ DHCP Server ] 
      ↓
[ Core Switch ] (Trusted Port)
      ↓
[ Access Switch ] (Untrusted Port) → [ Client (Normal) ]
      ↓
(Untrusted Port) ← [ Rogue DHCP Server (Blocked!) ]
```

---

## ⚙ 動作シーケンス

1.  **バインディングの生成**: クライアントがUntrustedポート経由で正規のDHCPサーバーからIPを取得すると、スイッチはその「MACアドレス、IPアドレス、リース時間、VLAN、ポート番号」を**DHCP Snooping Binding Database**に記録します。
2.  **不正サーバーの遮断**: Untrustedポートから`DHCPOFFER`などのサーバー用メッセージが入ってきた場合、スイッチはそれを破棄し、セキュリティ違反としてログを記録します。
3.  **整合性チェック**: クライアントから送信される`DHCPRELEASE`や`DHCPDECLINE`に含まれるMACアドレスが、バインディングテーブル内のエントリと一致しない場合もドロップします。
4.  **Option 82の挿入**: 必要に応じて、パケットがどのポートから入ってきたかを示す「回路ID（Circuit ID）」などをDHCPヘッダーに追加してサーバーに転送します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **「VLAN有効化」の忘れ**: `ip dhcp snooping` をグローバルで有効にするだけでは不十分です。必ず `ip dhcp snooping vlan [ID]` で対象VLANを指定する必要があります。ラボでの失点ポイントの定番です。
*   **UplinkのTrust設定**: コアスイッチやDHCPサーバーに繋がるトランクポートに `ip dhcp snooping trust` を設定し忘れると、配下のクライアントが一切IPアドレスを取得できなくなります。
*   **Option 82とリレーエージェントの競合**: CiscoスイッチはデフォルトでOption 82を挿入しますが、上位のルータがそれを「非リレーエージェントからの不正なパケット」とみなして破棄することがあります。この場合、`no ip dhcp snooping information option` で無効化するか、ルータ側で信頼設定が必要です。
*   **データベースの永続化**: スイッチを再起動するとバインディング情報が消えてしまいます。ラボで「再起動後もバインディングを維持せよ」とあれば、`ip dhcp snooping database flash:snooping.db` のように外部メモリやFlashへの保存設定が求められます。
*   **Rate Limiting**: 大量のDHCPパケットによるDoSを防ぐため、`ip dhcp snooping limit rate` の設定が要求されることがあります。

---

## 🛠 設定方法

### 1. 基本設定（IOS-XE）
```bash
! グローバルで有効化
ip dhcp snooping
! 対象VLANを指定（必須）
ip dhcp snooping vlan 10,20
! Option 82の挿入を無効化（必要に応じて）
no ip dhcp snooping information option

! DHCPサーバー側のインターフェイスを信頼
interface GigabitEthernet1/0/24
 description TO-DHCP-SERVER
 ip dhcp snooping trust

! クライアント側のポートにレート制限を適用
interface GigabitEthernet1/0/1
 ip dhcp snooping limit rate 15
```

### 2. データベースの保存設定
```bash
! Flashに保存し、300秒ごとに更新
ip dhcp snooping database flash:dhcp_bind.db
ip dhcp snooping database write-delay 300
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **全体の設定状態の確認** | <code>show ip dhcp snooping</code> |
| **バインディングテーブルの表示** | <code>show ip dhcp snooping binding</code> |
| **統計情報の確認（ドロップ数など）** | <code>show ip dhcp snooping statistics</code> |
| **データベースエージェントの状態確認** | <code>show ip dhcp snooping database</code> |
| **リアルタイムのデバッグ** | <code>debug ip dhcp snooping [events\|packet]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 端末がIPを取得できない | UplinkがTrustされていない | <code>show ip dhcp snooping</code> でTrustポートを確認。 |
| `Statistics` でドロップが増える | レート制限の超過 | <code>limit rate</code> の値を増やすか、攻撃を特定。 |
| スイッチ再起動後にDAIが失敗する | DBが保存されていない | <code>ip dhcp snooping database</code> を設定。 |
| 特定のVLANだけIPが取れない | VLAN有効化の漏れ | <code>ip dhcp snooping vlan [ID]</code> があるか確認。 |
| ルータ越しのDHCPが通らない | Option 82の不整合 | <code>no ip dhcp snooping information option</code> を試す。 |

---

## ⚠ 制限事項

*   **CPU処理**: DHCP Snoopingを有効にすると、DHCPパケットはハードウェアではなくスイッチのCPUで処理されます。過度なパケット流入はスイッチ全体のパフォーマンスを低下させます。
*   **メモリ制限**: バインディングデータベースはRAMを消費するため、数千台のクライアントを収容する環境ではリソースを考慮する必要があります。
*   **EtherChannel**: EtherChannelポートに設定する場合、物理メンバポートではなく論理ポート（Port-channel）に設定する必要があります。

---

## 🔄 他技術との関連

*   **3.4.a DAI (Dynamic ARP Inspection)**: DHCP Snoopingが作成したデータベースを使用して、ARPパケットの正当性を検証します。
*   **IP Source Guard (IPSG)**: データベースを参照し、バインディングにない送信元IPアドレスを持つIPトラフィック（データパケット）をL2ポートで遮断します。
*   **3.1.a CoPP (Control Plane Policing)**: CPUへ向かうDHCPトラフィックを制限し、スイッチ自身の可用性を守ります。

---

## 🧩 比較表

### DHCP Snooping vs DAI vs IPSG

| 機能 | 監視対象 | 目的 | 依存関係 |
| :--- | :--- | :--- | :--- |
| **DHCP Snooping** | DHCPパケット | 不正サーバー排除・DB構築 | なし |
| **DAI** | ARPパケット | ARPスプーフィング防止 | **DHCP Snoopingが必要** |
| **IPSG** | IPパケット | IPスプーフィング防止 | **DHCP Snoopingが必要** |

---

## 💡 ベストプラクティス

1.  **最小権限の原則**: ユーザーポートはすべてUntrustedにし、DHCPサーバーが存在する既知のパスのみをTrustにします。
2.  **レート制限の設定**: すべてのUntrustedポートに `limit rate 15-20` 程度を設定し、バースト的な攻撃を抑制します。
3.  **永続DBの使用**: ラボや実環境に関わらず、再起動後の通信断を防ぐために `database` 保存設定は必須です。
4.  **DAIとの同時導入**: 単体での運用よりも、DAIやIPSGと組み合わせてL2全体の信頼性を担保することを推奨します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な不正DHCPサーバー対策
*   **要件**: VLAN 10でDHCP Snoopingを有効にし、Gi0/1（DHCPサーバー側）を信頼せよ。
*   **設定**: `ip dhcp snooping`, `ip dhcp snooping vlan 10`, `int gi0/1` > `ip dhcp snooping trust`。

### 2. DHCP飢餓攻撃の緩和
*   **要件**: アクセスポート Gi0/2 でDHCPパケットを秒間10パケットに制限せよ。
*   **設定**: `int gi0/2` > `ip dhcp snooping limit rate 10`。

### 3. Option 82の無効化（リレー問題対策）
*   **要件**: 上位ルータがOption 82付きパケットを破棄するため、情報オプションを挿入しないようにせよ。
*   **設定**: `no ip dhcp snooping information option`。

### 4. データベースのFlash保存
*   **要件**: バインディング情報を `flash:snoop.db` に保存せよ。
*   **設定**: `ip dhcp snooping database flash:snoop.db`。

### 5. 静的なバインディング追加
*   **要件**: DHCPを使用しないサーバー（MAC: 00AA.BBCC.DDEE, IP: 10.1.1.50）をデータベースに手動登録せよ（DAI/IPSG用）。
*   **設定**: `ip source binding 00aa.bbcc.ddee vlan 10 10.1.1.50 interface gi0/10`。

### 6. MACアドレス検証の強制
*   **要件**: DHCP要求に含まれる送信元MACと、L2ヘッダーのMACが一致することを確認せよ。
*   **設定**: `ip dhcp snooping verify mac-address`。

### 7. トランクポートのTrust設定
*   **要件**: 複数のスイッチを経由してDHCPサーバーに届く場合、スイッチ間のトランクを信頼せよ。

### 8. DHCP Snooping 統計のクリア
*   **課題**: トラブルシューティングのため現在のドロップカウンタをリセットせよ。
*   **コマンド**: `clear ip dhcp snooping statistics`。

### 9. 特定のMACからのDHCP拒否
*   **要件**: ACLとDHCP Snoopingを併用し、特定の不正端末のDHCP取得を阻止せよ。

### 10. DAIとの統合ラボ
*   **要件**: DHCP Snoopingを有効にした後、そのDBを使用してVLAN 20のARP偽装を防止せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `ip dhcp snooping vlan 10` が設定されているが、VLAN 20のクライアントに不正DHCPサーバーがIPを配布してしまった。なぜか？
    *   **回答**: DHCP SnoopingはVLANごとに有効化する必要があり、VLAN 20が設定に含まれていないため。
2.  **トラブルシュート**: `show ip dhcp snooping binding` にエントリが一つも表示されない。クライアントは正常に通信できている。原因は？
    *   **回答**: クライアントが固定IPを使用している、あるいはDHCP Snoopingが有効になる前にIPを取得済みである。
3.  **Design**: DAIを導入したいが、ネットワーク内にDHCPを使用しない静的IPのデバイスがある。設計上の配慮は？
    *   **回答**: 静的デバイスのために `ip source binding` を手動で作成するか、DAIの例外ACL（ARP ACL）を定義する。
4.  **実装**: ラボ試験で「Option 82のせいでIPが取れない」という状況を解決するための最も手っ取り早いコマンドは？
    *   **回答**: `no ip dhcp snooping information option`。
5.  **トラブルシュート**: UntrustedポートでDHCPOFFERを受信した。スイッチのデフォルトの動作と、どこでその確認ができるか？
    *   **回答**: スイッチはパケットをドロップする。`show ip dhcp snooping statistics` の "Illegal server messages" で確認できる。

---

## 🔗 参考リソース

*   **Cisco Live (BRKSEC-2202)**: [Securing the Layer 2 Infrastructure](https://www.ciscolive.com/)
*   **Cisco Configuration Guide**: [Configuring DHCP Snooping](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3750/software/release/12-2_55_se/configuration/guide/scg3750/swdhcp82.html)
*   **Integrated Security Technologies and Solutions, Volume I**: Chapter 5 (Layer 2 Security)

---

## 📝 **補足（Notes）**
*   **学習メモ**: 「DHCP Snoopingは名簿係」と覚えましょう。誰がどこでどのIPを貰ったかを常にメモし、その名簿（Binding DB）をDAIやIPSGという「警備員」に渡して不正をチェックさせます。
*   **注意点**: ラボ試験では、コマンドの結果が反映されるまで数秒かかる場合があります。設定後すぐに `show` コマンドで確認してもバインディングが出ないときは、クライアント側で `ipconfig /renew` を叩くなどのアクションが必要です。
