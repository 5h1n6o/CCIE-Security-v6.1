---
layout: default
title: 3.4.d-Port-security
nav_order: 4
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.4.d Port security

## 📘 概要

Port Security（ポートセキュリティ）は、Cisco Catalystスイッチのアクセスポートおよびトランクポートにおいて、接続を許可する端末（MACアドレス）を制限し、未許可デバイスによる不正アクセスやMACアドレススプーフィング、MACフラッディング攻撃（CAMテーブル溢れ攻撃）を防止するためのLayer 2セキュリティ機能である。

スイッチポート上で学習・固定化できるMACアドレスの最大数（Maximum MAC Addresses）を定義し、その上限を超えた未知のMACアドレスからのフレーム受信、あるいは許可リストに含まれないMACアドレスからのフレーム受信が発生した際に、トラフィックの破棄やポートの無効化（`err-disable`）などの違反処理（Violation Action）を即座に実行する。

キャンパスネットワークのエッジスイッチ（アクセスポート）、IP Phone接続ポート、会議室・ロビー等の物理的に保護されていない公開ポート、あるいはスイッチ間トランクポートにおける不正スイッチ増設防止など、多様なシナリオで基盤セキュリティの第一ラインとして広く適用される。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **特徴** | ポート単位で学習可能MAC数を制限。MACアドレスを静的登録（Static）、動的登録（Dynamic）、永続動的登録（Sticky）可能 |
| **用途** | 未許可端末のネットワーク接続防止、MACフラッディング（CAMテーブル枯渇攻撃）防止、不正ルーター/スイッチ増設制御 |
| **メリット** | ハードウェア（ASIC）レベルでの高速処理、設定がシンプルで効果的、802.1X未対応環境でも端末識別・保護が可能 |
| **デメリット** | MACスプーフィング（許可端末のMAC偽装）を単体で完全に阻止することは不可（DAIやIPSGとの併用が必要）、移動端末運用時にポートロックアウトのリスク |
| **対応機種** | Cisco Catalystシリーズ（2960, 3560, 3750, 3850, 9200, 9300, 9400, 9500等）、Cisco Nexusシリーズ（一部制限あり） |
| **制限事項** | SPAN Destinationポート、EtherChannelメンバーポート、Dynamic Accessポート（VTP/VMPS）、Private VLAN Promiscuousポートでは設定不可 |
| **設計上の注意点** | Voice VLAN構成時はIP Phone用とPC用でMaximum数を最低`2`以上に設定する必要あり。Sticky MAC使用時は `copy running-config startup-config` を忘れるとリボット後に消失 |

---

## 🏗 動作原理

Port Securityは、スイッチのCAM（Content Addressable Memory）テーブルとは独立した、Port Security専用の「Secure MAC Address Table」を保持する。ポートでPort Securityを有効化（`switchport port-security`）すると、デフォルトで最大許可MAC数（Maximum）は `1` に設定される。

### MACアドレスの学習タイプ

1. **Secure Dynamic（動的セキュア）**:
   - パケットを受信した際に動的にMACアドレスを学習し、RAM上のSecure MACテーブルに保持する。
   - スイッチ再起動やリンクダウンが発生すると、学習したMACアドレスは消去される。
2. **Secure Static（静的セキュア）**:
   - 管理者がCLIで手動でMACアドレス（`switchport port-security mac-address <MAC>`）を設定する。
   - `running-config` および `startup-config`（保存時）に永続化される。
3. **Sticky（スティッキーセキュア）**:
   - パケット受信時に動的にMACアドレスを学習し、自動的に `running-config` 内の静的コンフィグとして書き込む（`switchport port-security mac-address sticky <MAC>`）。
   - 再起動後も維持するには `copy running-config startup-config` の実行が必要。

### Violation（違反処理）モード

制限数を超えたMACアドレスからフレームを受信した場合、以下の3つのViolationモードに従って処理される。

```
[ Ingress Frame ]
        │
        ▼
[ MAC Address in Secure Table? ]
   ├── YES ──► [ Process / Forward Frame Normal ]
   │
   └── NO  ──► [ Exceeds Maximum MAC Count? ]
                   ├── NO  ──► [ Learn MAC & Forward Frame ]
                   │
                   └── YES ──► [ Trigger Violation Action ]
                                   │
                                   ├── Protect  : Drop Traffic (No Log / No SNMP Trap)
                                   ├── Restrict : Drop Traffic + Increment Counters + Syslog/SNMP Trap
                                   └── Shutdown : Drop Traffic + Set Port to err-disable + Syslog/SNMP Trap
```

---

## ⚙ 動作シーケンス

1. **Port Security有効化手順**:
   - インターフェイスを明示的に Access または Trunk に設定 (`switchport mode access` または `switchport mode trunk`)。
   - `switchport port-security` コマンドを発行してPort Security機能を起動。

2. **フレーム受信およびMACアドレス検証**:
   - スイッチのポートでフレームを受信。送信元MACアドレス（Source MAC）を抽出。
   - Secure MAC Address Table と照合。
   - 既存エントリーに一致する場合、通常のLayer 2スイッチング処理（CAMテーブル参照および転送）を実施。

3. **未知MACアドレスの検出と制限数チェック**:
   - Secure MAC Address Table に未登録のMACアドレスの場合、現在の学習済みSecure MAC件数と設定された `maximum` 値を比較。
   - **件数 < Maximum**: MACアドレスをSecure MACテーブルに追加（Stickyの場合はコンフィグに追加）し、正常転送。
   - **件数 >= Maximum**: 違反（Violation）と判定。

4. **Violation処理の実行**:
   - **Protectモード**: 未許可MACからのトラフィックのみをサイレントドロップ（送信元ポートで破棄）。違反カウンターは増分しない。
   - **Restrictモード**: 未許可MACからのトラフィックをドロップ。違反カウンター（`SecurityViolation`）をインクリメントし、Syslogメッセージ送出およびSNMP Trap（`cSecureMacAddrViolation`）を発生。
   - **Shutdownモード**: インターフェイスを即座に `err-disable` 状態へ遷移させ、ポートを物理的にダウン（Link-down）。Syslog/SNMP Trapを発行。手動で `shutdown` / `no shutdown` を実行するか、`errdisable recovery cause psecure-violation` による自動復旧が必要。

---


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、Port Securityは単体での設定だけでなく、Voice VLAN (IP Phone)、802.1X/MAB、DHCP Snooping、Dynamic ARP Inspection (DAI)、TrunkポートでのVLAN指定Maximum制限など、他技術と連携した複雑な条件で出題される。

### 試験で頻出の要点・設定ポイント

1. **Voice VLAN環境での Maximum 数設定**:
   - IP Phone（1台）とPC（1台）を同一ポートに接続する場合、IP PhoneのMACアドレス（Voice VLAN）とPCのMACアドレス（Data VLAN）の計2つのMACアドレスが提示される。
   - `switchport port-security maximum 2` の設定が必須。省略するとデフォルトの `maximum 1` となり、2台目のデバイス（通常PC）が接続された瞬間にポートが `err-disable` 化する。
   - トランクポートや Voice VLAN 明示指定（`switchport port-security maximum 2 vlan voice` 等）の細かな要件に注意。

2. **Sticky MAC Address の自動学習と保存要件**:
   - 「PCを接続してMACアドレスを動的に取得させ、リブート後も設定を固定化せよ」という要件。
   - `switchport port-security mac-address sticky` を設定した後、実際にICMP等でトラフィックを発生させてMACを学習させ、**必ず `copy running-config startup-config`（または `write memory`）を実行**しなければならない。これを怠ると、採点時の再起動処理でコンフィグが消失し失点となる。

3. **Violation モードの指定要件**:
   - 「未許可トラフィックのみを破棄し、Syslog通知を行え」 → **Restrict**
   - 「未許可トラフィックをサイレントに破棄し、通知を行うな」 → **Protect**
   - 「未許可デバイス検出時にポートを遮断（err-disable）せよ」 → **Shutdown**
   - 「特定のVLANのみをドロップせよ」 → **Shutdown VLAN** (`switchport port-security violation shutdown vlan`)

4. **Errdisable 状態の自動復旧設定**:
   - ポートセキュリティ違反による `err-disable` 状態を特定時間（例: 300秒）経過後に自動復旧させる要件。
   - `errdisable recovery cause psecure-violation`
   - `errdisable recovery interval 300`

5. **Secure MAC Address Aging (エージング) 設定**:
   - 一定時間通信がないMACアドレスをテーブルから消去する設定。
   - `switchport port-security aging time <minutes>`
   - `switchport port-security aging type {absolute | inactivity}`
   - `switchport port-security aging static` (静的登録MACに対してもエージングを有効化)

---

## 🛠 設定方法

### 1. 基本的な Port Security (Sticky MAC & Shutdown モード)

```bash
interface GigabitEthernet1/0/1
 description Access Port for Workstation
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation shutdown
 switchport port-security mac-address sticky
 switchport port-security aging time 10
 switchport port-security aging type inactivity
```

### 2. Voice VLAN (IP Phone + PC) 構成における Port Security

```bash
interface GigabitEthernet1/0/2
 description Connected to Cisco IP Phone & PC
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 switchport port-security
 switchport port-security maximum 3
 switchport port-security maximum 1 vlan access
 switchport port-security maximum 2 vlan voice
 switchport port-security violation restrict
 switchport port-security mac-address sticky
```

### 3. Trunk ポートにおける Per-VLAN Port Security

```bash
interface GigabitEthernet1/0/24
 description Trunk to Downstream Switch
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 switchport port-security
 switchport port-security maximum 10
 switchport port-security maximum 3 vlan 10
 switchport port-security maximum 5 vlan 20
 switchport port-security violation shutdown vlan
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| --- | --- |
| **全体ステータス確認** | `show port-security` |
| **特定ポート詳細確認** | `show port-security interface GigabitEthernet1/0/1` |
| **学習済みSecure MAC一覧** | `show port-security address` |
| **特定VLAN/ポートのMAC一覧** | `show port-security interface GigabitEthernet1/0/1 address` |
| **Errdisable状態の確認** | `show errdisable recovery` / `show interfaces status err-disabled` |
| **Debugコマンド (注意して使用)** | `debug port-security` / `debug platform condition feature port-security` |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **端末接続直後にポートが err-disable になる** | Maximum値が小さすぎる（例: IP Phone + PC環境で `maximum 1` のまま） | `show port-security interface Gi1/0/1` | `switchport port-security maximum 2` 以上に調整 |
| **スイッチ再起動後にSticky MACが消える** | `copy run start` 未実行のため `startup-config` に反映されていない | `show running-config interface Gi1/0/1` | 学習後に `copy running-config startup-config` を実行 |
| **未許可MAC接続時にログが記録されない** | Violationモードが `Protect` に設定されている | `show port-security interface Gi1/0/1` | ログ出力が必要な場合は `Restrict` または `Shutdown` に変更 |
| **`switchport port-security` が入力できない** | ポートモードが `dynamic auto` / `dynamic desirable`（DTP）のまま | `show switchport interface Gi1/0/1` | `switchport mode access` または `switchport mode trunk` を先に設定 |
| **特定VLANトラフィックのみ通過しない** | `violation shutdown vlan` により該当VLANのみがブロックされた | `show port-security interface Gi1/0/1` | 違反原因（未許可MAC）を解除し、該当ポートで `clear port-security-vlan` を実行 |
| **802.1X/MAB環境でポートセキュリティが誤動作する** | 802.1X認証成功前後のMAC学習競合 | `show authentication sessions interface Gi1/0/1` | 802.1Xとポートセキュリティの統合設定基準に従い `maximum` を適切に増加 |

---


## ⚠ 制限事項

1. **互換性のないポートタイプ / 機能**:
   - **SPAN Destination ポート**: SPANの宛先ポートとして設定されているインターフェイスでは Port Security を有効化できない。
   - **EtherChannel メンバーポート**: EtherChannel（PAGP / LACP）に所属する個別の物理インターフェイスには設定不可。Port-Channel 論理インターフェイス側で設定する必要がある。
   - **Dynamic Access ポート**: VTP / VMPS による動的VLAN割り当てポート上ではサポートされない。
   - **Private VLAN Promiscuous / Isolated Trunk ポート**: 一部の古い Catalyst OS バージョンにおいて動作制限あり。

2. **MACスプーフィングに対する限界**:
   - Port Security は「受信したフレームの送信元MACアドレス」のみをチェックする。
   - 許可されている端末のMACアドレスを意図的に攻撃者が偽装（MAC Spoofing）した場合、Port Security 単体ではそれを見破ることができない。
   - IP/MAC のバインディング検証（Dynamic ARP Inspection, IP Source Guard）や 802.1X ポート認証との併用が必須。

3. **CPU処理およびテーブル上限**:
   - 大量のポートで非常に大きな `maximum` 値（例: 数百個）を設定した場合、TCAM/Secure MAC テーブルのリソースを大量に消費する。

---

## 🔄 他技術との関連

Port Securityは、他の Layer 2/3 セキュリティ機能やアクセス制御プロトコルと組み合わせて多層防御（Defense-in-Depth）を形成する。

### 1. 802.1X / MAB (MAC Authentication Bypass)
- **関係性**: 802.1X/MABはサプリカント（端末）をRADIUSサーバー（ISE等）で認証し、Port Securityはポート単位のMAC物理接続数を制御する。
- **注意点**: 802.1X環境でPort Securityを併用する場合、802.1XサプリカントのMAC、IP PhoneのMAC、および認証試行時のプロトコルMACなどが多重登録されるため、`switchport port-security maximum` を小さく設定しすぎると認証成立時にViolationが発生する。

### 2. DHCP Snooping & Dynamic ARP Inspection (DAI) & IP Source Guard (IPSG)
- **関係性**:
  - Port Security: 送信元MACの接続制限。
  - DHCP Snooping: DHCPパケットの監視と IP-MAC-Port-VLAN バインディングテーブル作成。
  - DAI: ARPパケットの偽装防止（DHCP Snoopingバインディング参照）。
  - IPSG: IPパケットの偽装防止（DHCP Snoopingバインディング参照）。
- **組み合わせ**: Port Securityで物理的な接続MAC数を縛り、IPSG/DAIでそのMACが使用するIPアドレスの偽装を防ぐことで、完全な Layer 2 - Layer 3 バインディング保護が実現する。

### 3. Voice VLAN (IP Phone)
- **関係性**: 1つの物理ポートに Cisco IP Phone（Voice VLAN）と PC（Access VLAN）がデイジーチェーン接続される。
- **必須設計**: `switchport port-security maximum 2` 以上（通常、安全域として `maximum 3`）。

---

## 🧩 比較表

### 1. Violation モードの動作比較

| 項目 | Protect | Restrict | Shutdown | Shutdown VLAN |
| --- | --- | --- | --- | --- |
| **違反トラフィック処理** | 破棄（Drop） | 破棄（Drop） | 破棄（Drop） | 破棄（Drop） |
| **許可MACトラフィック処理** | 正常転送 | 正常転送 | 全遮断（Port Down） | 該当VLANのみ遮断 |
| **ポート状態（Status）** | Up / Up | Up / Up | `err-disable` | Up / Up (VLAN Down) |
| **違反カウンター増加** | なし | あり（`SecurityViolation`） | あり（`SecurityViolation`） | あり |
| **Syslog / SNMP Trap** | なし | あり | あり | あり |
| **復旧方法** | 不要（自動正常化） | 不要（自動正常化） | `shut`/`no shut` または `errdisable recovery` | `clear port-security-vlan` |

---

### 2. MACアドレス学習タイプの比較

| 項目 | Dynamic | Static | Sticky |
| --- | --- | --- | --- |
| **設定方法** | 自動学習 | CLIで手動入力 | 自動学習 ➔ コンフィグ自動注入 |
| **コンフィグ保存 (`running-config`)** | 記録されない | 記録される | 記録される |
| **再起動後の保持** | 保持されない（消去） | 保持される（`startup-config`保存時） | 保持される（`startup-config`保存時） |
| **エイジング（Aging）処理** | 対応（デフォルト有効） | 対応（`aging static` 設定時） | 対応（`aging time` 設定時） |

---

## 💡 ベストプラクティス

1. **アクセスポートの標準化テンプレート**:
   - ユーザーアクセスポートには基本として `violation restrict` または `violation shutdown` を適用。
   - 無用なポートロックアウトを防ぎつつSyslog通知を得たい場合は `restrict` モードが推奨される。

2. **Sticky MAC 運用での保存自動化 / 確定運用**:
   - 開通作業時に端末を接続して Sticky MAC を学習させた後、直ちに `copy running-config startup-config` を実行する運用スクリプトまたは EEM などの自動化スクリプトを導入する。

3. **Errdisable 自動復旧の常時有効化**:
   - 誤接続による全社的サポートコストを低減するため、`errdisable recovery cause psecure-violation` および `errdisable recovery interval 300` をグローバル設定に含める。

4. **IP Phone 接続ポートでの Maximum の明確化**:
   - Voice VLAN ポートには `switchport port-security maximum 3` を設定し、`vlan access` と `vlan voice` の上限を明示的に分ける。

---


## 📝 ラボ学習・設定サンプル例

実際の CCIE Security ラボ試験レベルを意識した 10 個の実践的演習シナリオを掲載する。設定例は省略せず最後まで出力している。

---

### シナリオ 1: 静的 MAC アドレス固定登録と Shutdown Violation モード

#### 問題
アクセススイッチ SW1 の GigabitEthernet1/0/1 ポートに特定のサーバー（MACアドレス: `0011.2233.4455`）のみ接続を許可したい。未許可デバイスが接続された場合は即座にポートを遮断せよ。

#### 要件
1. ポートを Access モード（VLAN 10）に設定する。
2. 許可するMACアドレス `0011.2233.4455` を静的に登録する。
3. 違反検出時のモードは `shutdown` とする。
4. 設定を保存し、再起動後も静的設定が保持されるようにする。

#### 設定例
```bash
SW1# configure terminal
SW1(config)# vlan 10
SW1(config-vlan)# exit

SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# description Server-Port
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 1
SW1(config-if)# switchport port-security violation shutdown
SW1(config-if)# switchport port-security mac-address 0011.2233.4455
SW1(config-if)# end
SW1# copy running-config startup-config
```

#### 検証方法
```bash
SW1# show port-security interface GigabitEthernet1/0/1
SW1# show port-security address
```
未許可MACアドレスを持つ偽装PCを Gi1/0/1 に接続し、トラフィック（ICMP Ping）を送信する。SW1 上で Syslog (`%PM-4-ERR_DISABLE: psecure-violation error detected on Gi1/0/1`) が出力され、`show interfaces GigabitEthernet1/0/1` で `err-disabled` 状態へ遷移したことを確認する。

---

### シナリオ 2: Sticky MAC Address 自動学習と永続化コンフィグ

#### 問題
スイッチ SW1 の GigabitEthernet1/0/2 ポートにおいて、最初に接続された PC の MAC アドレスを自動学習し、コンフィグへ静的書き込み（Sticky）を行え。

#### 要件
1. VLAN 20 の Access ポートとして構成する。
2. 接続された最初の 1 台の MAC アドレスのみを Sticky 学習する。
3. 学習された MAC アドレスが `running-config` に反映されることを確認せよ。

#### 設定例
```bash
SW1# configure terminal
SW1(config)# vlan 20
SW1(config-vlan)# exit

SW1(config)# interface GigabitEthernet1/0/2
SW1(config-if)# description User-PC-Port
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 1
SW1(config-if)# switchport port-security violation restrict
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# end
```

#### 検証方法
PC から SW1 へ Ping を実行してフレームを送出させる。
SW1 上で以下を実行し、自動的に `switchport port-security mac-address sticky <PC-MAC>` が追加されたことを確認する。
```bash
SW1# show running-config interface GigabitEthernet1/0/2
SW1# show port-security address
SW1# copy running-config startup-config
```

---

### シナリオ 3: Voice VLAN (IP Phone + PC) 環境における Multi-VLAN Port Security

#### 問題
Gi1/0/3 に Cisco IP Phone とその背後に PC を接続する。Voice VLAN と Access VLAN の両方で適切な Port Security を構成せよ。

#### 要件
1. Access VLAN: 10、Voice VLAN: 100 とする。
2. ポート全体で最大 3 個の MAC アドレス（IP Phone 用 2 個、PC 用 1 個）を許可する。
3. Access VLAN では最大 1 個、Voice VLAN では最大 2 個の MAC アドレスを許可する。
4. 違反時は `restrict` モードとし、Syslog ログを出力してトラフィックをドロップせよ。

#### 設定例
```bash
SW1# configure terminal
SW1(config)# vlan 10,100
SW1(config-vlan)# exit

SW1(config)# interface GigabitEthernet1/0/3
SW1(config-if)# description IP-Phone-and-PC
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport voice vlan 100
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 3
SW1(config-if)# switchport port-security maximum 1 vlan access
SW1(config-if)# switchport port-security maximum 2 vlan voice
SW1(config-if)# switchport port-security violation restrict
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# end
```

#### 検証方法
```bash
SW1# show port-security interface GigabitEthernet1/0/3
```
IP Phone および PC からそれぞれトラフィックを送出し、`vlan access` および `vlan voice` の学習数カウンターが正しくインクリメントされることを確認。追加で 2 台目の PC をハブ経由で接続し、違反カウンター（SecurityViolation）が増加しログが出力されることを確認。

---

### シナリオ 4: Protect Violation モードと Inactivity Aging の制御

#### 問題
会議室用ポート Gi1/0/4 において、未許可端末接続時にポートをダウンさせずサイレントに通信を無視（Protect）し、さらに 15 分間不活性な MAC アドレスを自動消去（Inactivity Aging）せよ。

#### 要件
1. Access VLAN 30。
2. 最大学習 MAC 数: 2。
3. 違反処理: `protect`。
4. エージング機能: 不活性（inactivity）タイマー 15 分を設定。

#### 設定例
```bash
SW1# configure terminal
SW1(config)# vlan 30
SW1(config-vlan)# exit

SW1(config)# interface GigabitEthernet1/0/4
SW1(config-if)# description Meeting-Room-Port
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 30
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 2
SW1(config-if)# switchport port-security violation protect
SW1(config-if)# switchport port-security aging time 15
SW1(config-if)# switchport port-security aging type inactivity
SW1(config-if)# end
```

#### 検証方法
```bash
SW1# show port-security interface GigabitEthernet1/0/4
```
端末 A, B を接続して MAC を学習させた後、端末 C を接続する。端末 C の通信のみが破棄され、ポート状態は Up/Up のままであることを確認。また端末 A の通信を 15 分間停止させた後、Secure MAC テーブルから消去されることを確認。

---

### シナリオ 5: Restrict Violation モードと Syslog / SNMP Trap 通知の検証

#### 問題
公開ロビーポート Gi1/0/5 で違反が発生した際、ポートを落とさずに違反パケットのみ破棄し、管理システムへ Syslog および SNMP Trap を即座に発信せよ。

#### 要件
1. Access VLAN 40。
2. 最大 MAC 数: 1。
3. 違反処理: `restrict`。
4. SNMP Trap 送信元およびコミュニティ名を定義。

#### 設定例
```bash
SW1# configure terminal
SW1(config)# snmp-server enable traps port-security
SW1(config)# snmp-server host 192.168.100.50 public port-security
SW1(config)# logging host 192.168.100.50

SW1(config)# interface GigabitEthernet1/0/5
SW1(config-if)# description Lobby-Port
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 40
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 1
SW1(config-if)# switchport port-security violation restrict
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# end
```

#### 検証方法
1 台目の PC で通信発生後、2 台目の PC を接続して通信を試みる。SW1 上で `%PORT_SECURITY-2-PSECURE_VIOLATION` の Syslog が出力され、`show port-security interface Gi1/0/5` の `SecurityViolation` カウンターが増加することを確認。

---

### シナリオ 6: Errdisable Automatic Recovery（自動復旧）の実装

#### 問題
Port Security 違反により `err-disable` 状態となったポートを、手動介入なしで 120 秒後に自動的に復旧させるようシステム全体で構成せよ。

#### 要件
1. `psecure-violation` による `err-disable` を自動リカバリの対象とする。
2. リカバリタイマー（Interval）を 120 秒に設定する。

#### 設定例
```bash
SW1# configure terminal
SW1(config)# errdisable recovery cause psecure-violation
SW1(config)# errdisable recovery interval 120
SW1(config)# end
```

#### 検証方法
```bash
SW1# show errdisable recovery
```
Port Security 違反を発生させてポートを `err-disable` に落とした後、`show interfaces status err-disabled` を実行。Timer status が `120 sec` からカウントダウンし、120 秒後に自動的に `shutdown` -> `no shutdown` が内部実行されて通信が再開（未許可端末解除時）することを確認。

---

### シナリオ 7: Trunk ポートにおける VLAN 毎の Maximum MAC 受信上限制御

#### 問題
スイッチ間トランクポート Gi1/0/24 において、VLAN 10, 20, 30 を通過させ、各 VLAN 毎に受信できる MAC アドレス上限数を個別に制限せよ。

#### 要件
1. トランクポートとし、許可 VLAN は 10, 20, 30。
2. 全体 Maximum 数: 15。
3. VLAN 10 Maximum: 3。
4. VLAN 20 Maximum: 5。
5. VLAN 30 Maximum: 5。
6. 違反モード: `shutdown`。

#### 設定例
```bash
SW1# configure terminal
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# description Trunk-to-SW2
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20,30
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 15
SW1(config-if)# switchport port-security maximum 3 vlan 10
SW1(config-if)# switchport port-security maximum 5 vlan 20
SW1(config-if)# switchport port-security maximum 5 vlan 30
SW1(config-if)# switchport port-security violation shutdown
SW1(config-if)# end
```

#### 検証方法
```bash
SW1# show port-security interface GigabitEthernet1/0/24
SW1# show port-security interface GigabitEthernet1/0/24 vlan
```
各 VLAN 内で制限数を超えるMACアドレスからのトラフィックを送出し、特定 VLAN の上限到達時にポート全体が `err-disable` に遷移することを確認。

---

### シナリオ 8: Shutdown VLAN モードによるマルチ VLAN トランクの一部遮断

#### 問題
トランクポート Gi1/0/23 において、違反が発生した VLAN のトラフィックのみを破棄（VLAN 単位遮断）し、他の正常な VLAN の通信は維持せよ。

#### 要件
1. トランクポート（VLAN 10, 20）。
2. VLAN 10 上限: 2、VLAN 20 上限: 2。
3. 違反モード: `shutdown vlan`。

#### 設定例
```bash
SW1# configure terminal
SW1(config)# interface GigabitEthernet1/0/23
SW1(config-if)# description Trunk-Port-ShutdownVlan
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 4
SW1(config-if)# switchport port-security maximum 2 vlan 10
SW1(config-if)# switchport port-security maximum 2 vlan 20
SW1(config-if)# switchport port-security violation shutdown vlan
SW1(config-if)# end
```

#### 検証方法
VLAN 10 上で 3 台目の MAC からトラフィックを注入。VLAN 10 のトラフィックのみが遮断され、VLAN 20 の通信が正常に継続することを確認。`show port-security interface Gi1/0/23` で VLAN 10 が Shut 状態になっていることを確認し、復旧は `clear port-security-vlan 10 interface Gi1/0/23` を実行。

---

### シナリオ 9: Port Security と 802.1X / MAB 統合認証環境の構築

#### 問題
ポート Gi1/0/6 において、802.1X 認証を有効化し、認証成功デバイスの MAC アドレスを Secure MAC としても保持させよ。

#### 要件
1. 802.1X 認証および MAB を設定。
2. Port Security を有効化し、最大 2 台（802.1X PC + IP Phone）を制御。
3. 違反モード: `restrict`。

#### 設定例
```bash
SW1# configure terminal
SW1(config)# aaa new-model
SW1(config)# aaa authentication dot1x default group radius
SW1(config)# radius server ISE
SW1(config-radius-server)# address ipv4 192.168.100.100 auth-port 1812 acct-port 1813
SW1(config-radius-server)# key Cisco123
SW1(config-radius-server)# exit

SW1(config)# dot1x system-auth-control

SW1(config)# interface GigabitEthernet1/0/6
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# authentication port-control auto
SW1(config-if)# mab
SW1(config-if)# dot1x pae authenticator
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 2
SW1(config-if)# switchport port-security violation restrict
SW1(config-if)# end
```

#### 検証方法
```bash
SW1# show authentication sessions interface GigabitEthernet1/0/6
SW1# show port-security interface GigabitEthernet1/0/6
```
802.1X 認証成功後、サプリカントの MAC が認証テーブルおよび Secure MAC テーブルの双方に正しく登録されることを確認。

---

### シナリオ 10: Layer 2 総合セキュリティ統合（Port Security + DHCP Snooping + DAI）

#### 問題
アクセスポート Gi1/0/7 に対し、未許可 MAC の接続防止（Port Security）、偽造 DHCP サーバー遮断およびバインディング生成（DHCP Snooping）、ならびに ARP ポイズニング防止（DAI）を包括的に適用せよ。

#### 要件
1. VLAN 10 上で DHCP Snooping および DAI を有効化。
2. Gi1/0/7 に Port Security (Maximum 2, Sticky, Restrict) を設定。
3. Uplink ポート (Gi1/0/24) は DHCP Snooping / DAI の Trusted とする。

#### 設定例
```bash
SW1# configure terminal
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10
SW1(config)# ip arp inspection vlan 10

SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# description Uplink-Trunk
SW1(config-if)# switchport mode trunk
SW1(config-if)# ip dhcp snooping trust
SW1(config-if)# ip arp inspection trust
SW1(config-if)# exit

SW1(config)# interface GigabitEthernet1/0/7
SW1(config-if)# description Edge-User-Port
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 2
SW1(config-if)# switchport port-security violation restrict
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# ip dhcp snooping limit rate 15
SW1(config-if)# end
```

#### 検証方法
```bash
SW1# show ip dhcp snooping
SW1# show ip arp inspection vlan 10
SW1# show port-security interface GigabitEthernet1/0/7
```
DHCP で IP を取得させた後、不正な静的 IP や他人の MAC アドレスを自称する ARP パケットを送出。DAI および Port Security の双方が不正トラフィックを検知して遮断することを確認。

---


## ❓ 想定試験問題

CCIE Security ラボ試験・記述試験を想定した実践問題と詳細解説。

---

### 質問 1 (トラブルシューティング)

**問題**:
あるアクセススイッチの Gi1/0/1 ポートに Cisco IP Phone（Voice VLAN 100）と配下 PC（Access VLAN 10）を接続したところ、IP Phone は正常に起動して通話可能だが、背後の PC がネットワークに接続できず、スイッチ上で以下のログが出力されてポートが遮断された。

```text
%PM-4-ERR_DISABLE: psecure-violation error detected on Gi1/0/1, putting Gi1/0/1 in err-disable state
%PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred, caused by MAC address 0050.56aa.bbcc on port GigabitEthernet1/0/1.
```

現在のポート設定は以下の通りである。

```text
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 100
 switchport port-security
 switchport port-security mac-address sticky
```

この障害の原因を説明し、最低限の変更で PC と IP Phone の双方が通信可能となるコンフィグ修正案を提示せよ。

**回答・解説**:
- **原因**: `switchport port-security` をパラメーターなしで有効化した場合、`maximum` のデフォルト値は `1` となる。IP Phone が起動してその MAC アドレスが先着で Secure MAC として登録された後、PC からのフレーム（MAC: `0050.56aa.bbcc`）を受信した際に `maximum 1` を超越したため、デフォルトの `violation shutdown` モードが発動してポートが `err-disable` 状態となった。
- **修正案**: `maximum` 値を 2 以上（安全域として 3 や VLAN 明示指定）に変更し、`err-disable` を解除する。

```bash
interface GigabitEthernet1/0/1
 switchport port-security maximum 3
 shutdown
 no shutdown
```

---

### 質問 2 (コンフィグ読解・設計)

**問題**:
以下のコンフィグが設定されたスイッチポート Gi1/0/2 の挙動について、正しく説明しているものを全て選べ。

```text
interface GigabitEthernet1/0/2
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 switchport port-security
 switchport port-security maximum 5
 switchport port-security maximum 2 vlan 10
 switchport port-security maximum 2 vlan 20
 switchport port-security violation shutdown vlan
```

A. ポート全体で最大 5 個の MAC アドレスを学習できる。
B. VLAN 10 で 3 個目の未知 MAC アドレスを受信すると、物理ポート全体が `err-disable` になる。
C. VLAN 10 で 3 個目の未知 MAC アドレスを受信した場合、VLAN 10 のトラフィックのみが遮断され、VLAN 20 の通信は継続する。
D. VLAN 10 で 2 個、VLAN 20 で 2 個の MAC を学習している状態で、VLAN 30 の未許可トラフィックを受信すると VLAN 30 が遮断される。

**回答・解説**:
- **正解**: **A, C**
- **解説**:
  - `violation shutdown vlan` が設定されているため、制限を超過した特定 VLAN のみが遮断（Shut）され、他の VLAN の通信は影響を受けない（Cが正解、Bは誤り）。
  - ポート全体の上限は `maximum 5`（Aが正解）。
  - `allowed vlan 10,20` であり、VLAN 30 は通過しないため D は無関係。

---

### 質問 3 (実装・要件定義)

**問題**:
セキュリティポリシーにより、「公開端末ポート Gi1/0/10 において、未許可デバイスのトラフィックはドロップし、監視サーバーへ Syslog 通知を出力しなければならない。また、未許可トラフィックによって正常な既存端末の通信を停止させてはならない」という要件が出された。どの Violation モードを選択すべきか？

A. Protect
B. Restrict
C. Shutdown
D. Shutdown VLAN

**回答・解説**:
- **正解**: **B (Restrict)**
- **解説**:
  - `Protect`: 違反トラフィックをドロップするが、Syslog や SNMP Trap の通知は出力されない。
  - `Restrict`: 違反トラフィックをドロップし、かつ Syslog / SNMP Trap 通知を発行し、カウンターを増分させる。既存の正常な通信は停止しない。
  - `Shutdown`: ポート全体を `err-disable`（Link-down）にするため、既存端末の通信も停止してしまう。
  - したがって要件を満たすのは `Restrict` である。

---

### 質問 4 (トラブルシューティング)

**問題**:
ネットワーク管理者が SW1 の全アクセスポートに Sticky MAC 設定を追加し、動作検証を行って正常に MAC アドレスが学習されたことを確認した。しかし、翌週スイッチの保守再起動を行ったところ、全ての端末で再度 Violation が発生し通信不能となった。考えられる原因は何か？

**回答・解説**:
- **原因**: Sticky MAC アドレスは動的に学習された後、`running-config` 内に静的コマンドとして自動挿入されるが、NVRAM 上の `startup-config` へは自動保存されない。管理者が学習完了後に `copy running-config startup-config`（または `write memory`）を実行しなかったため、再起動により `running-config` がリセットされ、Secure MAC 情報が消失したことが原因である。

---

### 質問 5 (技術比較・設計)

**問題**:
MAC アドレスの偽装（MAC Address Spoofing）攻撃に対し、Port Security 単体で防御できる限界と、それを補完するために併用すべき 2 つの Layer 2 セキュリティ機能を挙げよ。

**回答・解説**:
- **限界**: Port Security はポートで受信した「送信元 MAC アドレス」が許可リストに含まれているか否かのみを判定する。すでに許可されている端末の MAC アドレスを攻撃者が自端末に設定して通信した場合、Port Security はそれを正常通信とみなして許可してしまう（MAC スプーフィングの通過）。
- **併用すべき機能**:
  1. **Dynamic ARP Inspection (DAI)**: ARP パケット内の Sender MAC/IP と DHCP Snooping バインディングを比較し、ARP レベルでの MAC/IP 偽装を阻止。
  2. **IP Source Guard (IPSG)**: IP パケットの送信元 IP/MAC アドレスと DHCP Snooping バインディングを照合し、IP/MAC 双方の偽装を ASIC レベルでブロック。

---

## 🔗 参考リソース

Cisco 公式ドキュメント（Configuration Guide, Technical Notes）:

* Cisco Catalyst 9300 Series Switches Security Configuration Guide - Configuring Port Security
* Cisco Catalyst 3850 Series Switch Software Configuration Guide - Port Security
* Cisco Technical Support & Documentation - Secure MAC Addresses and Port Security Options
* Cisco Technical Support - Understand Port Security Violation Modes and Troubleshooting
* Cisco Live Presentation - Layer 2 Security Attacks and Mitigation Controls (BRKSEC-2001)

---

## 📝 補足（Notes）

### 補足 1: Secure MAC Address Aging のフラグと動作
`switchport port-security aging` には `absolute` と `inactivity` の 2 種類が存在する。
- **Absolute**: 設定された時間（分）が経過すると、通信の有無に関わらず Secure MAC がテーブルから消去される。
- **Inactivity**: 設定された時間内に通信が発生しなかった場合のみ、MAC が消去される。通信が継続している間はタイマーがリセットされるため保持され続ける。
- **`aging static`**: 明示的に `switchport port-security mac-address` で手動登録した静的 MAC に対してもエージングを適用させる場合に指定する。

### 補足 2: Secure MAC と CAM テーブルの関係
Port Security が有効化されると、ポートの通常の CAM テーブル学習が制御され、Secure MAC Address Table が優先される。Secure MAC に登録された MAC アドレスは、他のスイッチポート上で受信された場合、不正な移動（MAC Move）としてブロックされる。

---


