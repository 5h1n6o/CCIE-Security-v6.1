---
layout: default
title: 3.4.a-DAI
nav_order: 1
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.4.a Dynamic ARP Inspection (DAI)

## 📘 概要

Dynamic ARP Inspection（DAI）は、スイッチドネットワークにおける Address Resolution Protocol（ARP）の脆弱性を悪用した攻撃（ARPポイズニング、ARPスプーフィング、Man-in-the-Middle 攻撃）を防御するための Layer 2 セキュリティ機能です。ARP プロトコルは元来、認証機構やメッセージの真正性を検証する仕組みを持っておらず、ネットワーク上の任意のホストが偽造された ARP 応答（Gratuitous ARP や Unsolicited ARP Reply）を送信して、他のホストやデフォルトゲートウェイの ARP キャッシュテーブルを汚染することが可能です。

DAI は、スイッチを通過するすべての ARP パケット（Request および Reply）をインターセプトし、そのパケット内に含まれる送信元 IP アドレスと送信元 MAC アドレスのバインディング情報が、信頼できるデータベース（DHCP Snooping Binding Table）または手動で設定された ARP ACL（Access Control List）と一致するかどうかを検証します。データベースに存在しない、あるいは不一致である不正な ARP パケットは即座にドロップされ、ログ記録や SNMP トラップが発生します。

キャンパスネットワーク、データセンター、マルチテナント環境において、同一 VLAN / サブネット内の盗聴、トラフィックハイジャック、および Denial of Service（DoS）攻撃を防止するために必須となるテクノロジーです。

---

## 🔑 要点

DAI の主要な特徴、用途、利点、制限事項、および設計上の考慮事項を以下の表に示します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | ARP パケットを Layer 2 インターフェイスでインターセプトし、IP-to-MAC バインディング情報を検証して不正 ARP をドロップ |
| **用途** | Man-in-the-Middle（MitM）攻撃防止、ARP ポイズニング/スプーフィング対策、IP アドレス詐称防止 |
| **メリット** | 不正な ARP レスポンスによる中間者攻撃を網で阻止し、ネットワーク全体のトラフィック完全性を維持 |
| **デメリット** | スイッチの CPU で ARP パケットを検査するため、大量の ARP 攻撃を受けると CPU 負荷が上昇するリスク（Rate Limit で緩和） |
| **対応機種** | Catalyst 2960/3560/3750/3850/9000 シリーズ、Cisco Nexus シリーズ、IOS / IOS-XE スイッチ |
| **制限事項** | 信頼できるバインディング データベース（DHCP Snooping データベースまたは ARP ACL）が必須 |
| **設計上の注意点** | アップリンク/ルータ接続ポートは `trusted` 設定が必須。アクセスポートはデフォルトで `untrusted` |

---

## 🏗 動作原理

DAI は、スイッチのインターフェイスを **Trusted（信頼）** と **Untrusted（非信頼）** の 2 つのセキュリティ状態に分類して動作します。

```
[ Attack Host / Attacker ]
         │ (Untrusted Port: Sending Invalid ARP Reply: IP 10.1.1.254 = Attacker MAC)
         ▼
+─────────────────────────────────────────────────────────────────+
| Switch (DAI Enabled on VLAN 10)                                 |
|                                                                 |
|   1. Intercept ARP Packet on Untrusted Port                     |
|   2. Check DHCP Snooping Binding Table or ARP ACL               |
|      - Expected: IP 10.1.1.254 ──> Gateway MAC (0011.2233.4455)   |
|      - Received: IP 10.1.1.254 ──> Attacker MAC (00AA.BBCC.DD00)  |
|   3. Result: MISMATCH!                                          |
|   4. Action: DROP Packet & Increment Violation Counter / Log    |
+─────────────────────────────────────────────────────────────────+
         │ (Blocked)
         x
[ Normal Host A / Gateway ]
```

1. **Untrusted Port（非信頼ポート）**:
   - デフォルトですべてのスイッチポートは Untrusted に設定されています。
   - このポートで受信したすべての ARP Request および ARP Reply パケットは、CPU に送られてバインディング検証が行われます。
   - 検証（DHCP Snooping Binding 表または ARP ACL）を通過したパケットのみが他のポートへ転送されます。

2. **Trusted Port（信頼ポート）**:
   - スイッチ間のトランクポート、ルータ、DHCPServer、または信頼できる管理機器が接続されたポートに対して手動で `ip arp inspection trust` を設定します。
   - Trusted ポートで受信した ARP パケットは、バインディング検証をスキップして即座にフォワーディングされます。

---

## ⚙ 動作シーケンス

スイッチが Untrusted ポートで ARP パケットを受信した際の内部処理シーケンスは以下の通りです。

```
受信用 Untrusted Port
        │
        ▼
[ ARP Packet Intercepted ]
        │
        ▼
[ ARP Validation Check Options Enabled? ] ──(Yes)──> [ Validate dst-mac / src-mac / ip ] ──(Fail)──> [ DROP & Log ]
        │ (Pass / Disabled)                                                                            ▲
        ▼                                                                                              │
[ ARP ACL Configured for VLAN? ] ──(Yes)──> [ Match ARP ACL Entry? ] ──(Permit)──> [ FORWARD ]            │
        │ (No)                                   │ (Deny)                                              │
        │                                        └─────────────────────────────────────────────────────┘
        ▼
[ Check DHCP Snooping Binding Table ]
        │
        ├─ Match (IP + MAC + VLAN) ──> [ FORWARD ]
        │
        └─ Mismatch / Not Found ─────> [ DROP & Log ]
```

1. **パケットのインターセプト**: スイッチは Untrusted ポートに入った Layer 2 ARP パケットをキャプチャし、CPU 処理パスへ迂回させます。
2. **ARP ヘッダー検証（Validation Checks）**:
   - 追加の検証オプションが有効な場合、以下の整合性を確認します。
     - `dst-mac`: ARP Reply における目的 MAC アドレスが、Ethernet ヘッダーの宛先 MAC アドレスと一致するか。
     - `src-mac`: ARP Request/Reply における送信元 MAC アドレスが、Ethernet ヘッダーの送信元 MAC アドレスと一致するか。
     - `ip`: ARP パケット内の IP アドレスが無効（0.0.0.0 や 255.255.255.255、マルチキャスト IP など）でないか。
3. **ARP ACL の照合（優先度 1）**:
   - 対象 VLAN に ARP ACL が適用されている場合、DHCP Snooping データベースよりも優先して照合されます。
   - 静的 IP アドレスを使用するホストが接続されている環境で、非 DHCP 端末を許可するために使用します。
4. **DHCP Snooping Binding Table の照合（優先度 2）**:
   - ARP ACL で許可されなかった（または ARP ACL が未設定の）場合、DHCP Snooping バインディング データベースを参照します。
   - パケットの「送信元 IP アドレス」「送信元 MAC アドレス」「受信用 VLAN ID」の 3 つのエントリが完全一致すれば正常なパケットとして転送します。
5. **ドロップおよびアクション処理**:
   - 一致するバインディングが存在しない場合、パケットは破棄され、ログメッセージの生成およびカウンタのインクリメントが行われます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security ラボ試験において DAI は、Layer 2 セキュリティの必須項目として頻繁に出題されます。単独設定だけでなく、他の Layer 2/3 テクノロジーとの組み合わせ問題が特徴です。

### 1. Blueprintで重要なポイント
- VLAN ごとの DAI 有効化 (`ip arp inspection vlan <vlan-id>`)
- アップリンク / デフォルトゲートウェイポートの Trusted 化 (`ip arp inspection trust`)
- 静的 IP 端末を保護するための **ARP ACL** 作成と VLAN へのバインド (`ip arp inspection filter <acl-name> vlan <vlan-id> [static]`)
- **ARP Validation** オプションの設定 (`ip arp inspection validate src-mac dst-mac ip`)
- **Rate Limiting**（レート制限）の設定と `err-disable` 自動復旧設定 (`ip arp inspection limit rate <pps>`)

### 2. ラボ試験で設定させられそうな内容
- **静的 IP ホスト混在環境での DAI**:
  - DHCP Snooping で動的取得するクライアントと、静的 IP が割り振られた Server/Router が混在する VLAN に対し DAI を適用する要件。
  - `static` キーワードの有無による挙動の違い（`static` を付けると、ARP ACL にマッチしないものは DHCP Binding 表を評価せずに即ドロップする）。
- **Rate Limit と Err-disable 回避**:
  - アクセスポートでの ARP バースト（例: 15 pps 超過）によるポートの `err-disable` 化の防止、および `errdisable recovery cause arp-inspection` の設定。

### 3. よくある設定ミス
- **アップリンクポートの Trust 設定忘れ**:
  - スイッチ間トランクやルータポートに `ip arp inspection trust` を設定し忘れると、対向機器からの ARP 応答がすべてドロップされ、ネットワーク全体の通信が遮断される。
- **`static` オプションの誤用**:
  - `ip arp inspection filter MY_ARP_ACL vlan 10 static` と設定した場合、ACL に登録されていない DHCP クライアントの ARP がすべてドロップされる。DHCP 端末と静的 IP 端末が混在する場合は `static` を外す必要がある。
- **DHCP Snooping の有効化忘れ / Binding テーブル未生成**:
  - DAI は DHCP Snooping バインディング表に依存するため、DHCP Snooping が正しく動作していないと全 ARP が拒否される。

### 4. show / debug コマンドによる判断
- `show ip arp inspection vlan <vlan-id>`: DAI の有効化状態、Forwarded/Dropped カウンタを確認。
- `show ip arp inspection interfaces`: インターフェイスごとの Trust 状態と Rate Limit 設定を確認。
- `show ip arp inspection log`: 過去にドロップされた不整合 ARP パケットのログログバッファを確認。
- `debug ip arp inspection`: リアルタイムの ARP パケット評価ログの確認。

---



## 🛠 設定方法

### Cisco IOS / IOS-XE 基本コンフィグレーション例

以下は、VLAN 10 および 20 において DAI を有効化し、アップリンクポートを Trusted に設定、静的サーバー用の ARP ACL を適用し、バースト対策の Rate Limit を調整する基本設定例です。

```bash
! 1. DHCP Snooping の有効化（DAIの前提条件）
ip dhcp snooping
ip dhcp snooping vlan 10,20

! 2. 静的 IP サーバー用の ARP ACL の作成
arp access-list STATIC_SERVERS_ARP
 permit ip host 10.1.1.10 mac host 0050.56a1.2233
 permit ip host 10.1.1.254 mac host 0000.0c9f.f00a

! 3. VLAN に対する DAI の有効化および ARP ACL の適用
! 注: static キーワードを付けないことで、ACL未マッチ時は DHCP Snooping 表を参照
ip arp inspection vlan 10,20
ip arp inspection filter STATIC_SERVERS_ARP vlan 10

! 4. ARP 追加検証オプション（MAC/IPのチェック）の有効化
ip arp inspection validate src-mac dst-mac ip

! 5. インターフェイスレベルの設定
interface GigabitEthernet1/0/1
 description "Access Port for Users"
 switchport mode access
 switchport access vlan 10
 ! アクセスポートの Rate Limit 設定（デフォルト15ppsから要件に応じて調整）
 ip arp inspection limit rate 30

interface GigabitEthernet1/0/24
 description "Uplink to Core Switch / Router"
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 ! アップリンクポートを Trust 設定（DAIの検証をバイパス）
 ip arp inspection trust
 ip dhcp snooping trust

! 6. Err-disable の自動復旧設定
errdisable recovery cause arp-inspection
errdisable recovery interval 300
```

---

## 🔍 検証コマンド

DAI の動作状態および統計情報を確認するためのコマンド一覧です。

| 目的 | コマンド |
| --- | --- |
| DAI の全体ステータス・VLAN毎のドロップ統計確認 | `show ip arp inspection vlan <vlan-id>` |
| 各ポートの Trust 状態および Rate Limit 確認 | `show ip arp inspection interfaces` |
| DAI のログバッファおよび破棄された ARP 詳細確認 | `show ip arp inspection log` |
| DHCP Snooping バインディング データベース確認 | `show ip dhcp snooping binding` |
| 設定された ARP ACL の内容確認 | `show arp access-list` |
| Err-disable 状態のインターフェイス確認 | `show interfaces status err-disabled` |
| DAI 処理に関するリアルタイムデバッグ | `debug ip arp inspection` |

---

## 🚨 トラブルシュート

DAI 運用時に発生しやすい障害現象、原因、確認コマンド、および対処方法をまとめたマトリクスです。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| 正当な静的 IP 端末の通信が不可 | DHCP Snooping 表にエントリがなく、ARP ACL も未設定 | `show ip arp inspection log` | ARP ACL を作成し、対象 VLAN に `ip arp inspection filter` で適用 |
| アップリンク配下の全端末で通信不可 | アップリンクポートの Trust 設定が漏れている | `show ip arp inspection interfaces` | 対向スイッチ/ルータ接続ポートで <code>ip arp inspection trust</code> を設定 |
| ポートが頻繁に `err-disable` 化する | ARP パケットの受信レートが設定閾値（例: 15 pps）を超過 | `show interfaces status err-disabled` | 該当ポートの `ip arp inspection limit rate` 閾値を引き上げるか、`none` に変更 |
| DHCP 端末が通信不能になる | ARP ACL に `static` キーワードが付与されており DHCP 表が無視された | `show running-config \| inc ip arp inspection filter` | `static` キーワードを削除し、フォールスルー評価を許可する |
| スイッチの CPU 使用率が高騰する | 攻撃者による大量の無効 ARP パケットの流入 | `show process cpu detailed` | アクセスポートの Rate Limit を調整し、CoPP や Storm Control と併用 |

---

## ⚠ 制限事項

- **IPv4 専用**: DAI は IPv4 の ARP パケットのみを保護します。IPv6 環境における Neighbor Discovery 攻撃対策には **IPv6 First-Hop Security（IPv6 ND Inspection / RA Guard）** を別途実装する必要があります。
- **DHCP Snooping / ARP ACL 依存**: 有効なバインディング情報が存在しない場合、正当な通信であっても ARP パケットがドロップされます。
- **CPU 処理のオーバーヘッド**: Untrusted ポートの ARP パケットはすべて ASIC ではなく CPU（Control Plane）で検査されるため、適切な Rate Limit 設定が必要です。

---

## 🔄 他技術との関連

- **DHCP Snooping**: DAI のメインデータソース。DHCP アロケーション時に IP/MAC/VLAN/Port のバインディングテーブルを生成します。
- **IP Source Guard（IPSG）**: DAI が Layer 2 (ARP) レベルの詐称を防ぐのに対し、IPSG は Layer 3 (IP) データパケットの送信元 IP/MAC 詐称を防ぎます。両者は同じ DHCP Snooping データベースを共有します。
- **Port Security**: 1 ポートに接続可能な MAC アドレス数を制限し、DAI や DHCP Snooping バインディング表の枯渇攻撃を防ぎます。
- **Private VLANs（PVLAN）**: Secondary VLAN 上で DAI を有効化すると、Isolated ポートや Community ポート間の ARP スプーフィングも防止できます。

---

## 🧩 比較表

### DAI vs IP Source Guard (IPSG) vs Port Security

| 項目 | DAI (Dynamic ARP Inspection) | IP Source Guard (IPSG) | Port Security |
| --- | --- | --- | --- |
| **保護対象レイヤー** | Layer 2 (ARP パケット) | Layer 3 (IP データパケット) | Layer 2 (MAC アドレス) |
| **主な防御対象** | ARP ポイズニング / MitM 攻撃 | IP アドレススプーフィング | MAC フローディング / CAM テーブル枯渇 |
| **判定データソース** | DHCP Snooping 表 / ARP ACL | DHCP Snooping 表 / IP Source Binding | ポートごとの静的/動的 MAC テーブル |
| **処理ポイント** | 制御プレーン (CPU) 検査 | データプレーン (Hardware TCAM) フィルタ | データプレーン (MAC テーブル照合) |

---

## 💡 ベストプラクティス

1. **アクセスポートの Default Untrusted 設計**: 原則としてユーザー端末が接続されるアクセスポートはすべて Untrusted のまま保持し、スイッチ間接続・デフォルトゲートウェイポートのみを `trust` に設定します。
2. **ARP ACL の階層的運用**: ネットワークインフラ機器や静的 IP サーバーに対しては、事前に設計された ARP ACL をバインドし、DHCP Snooping データベース非依存の環境を構成します。
3. **適切な Rate Limiting と自動復旧**: アクセスポートには `ip arp inspection limit rate 15`（デフォルト）〜`30` を設定し、`errdisable recovery cause arp-inspection` を設定して運用オーバーヘッドを軽減します。
4. **追加検証オプションの有効化**: `ip arp inspection validate src-mac dst-mac ip` を有効化し、プロトコルレベルで異常なヘッダーを持つ ARP パケットを早期に遮断します。

---



## 📝 ラボ学習・設定サンプル例

実際の CCIE Security ラボ試験レベルを意識した 10 個の実践的な演習シナリオです。

### シナリオ 1: DAI の基本構成とトランクポートの Trust 設定

- **問題**: 企業ネットワークの SW1 において、VLAN 100 上で発生している ARP ポイズニング攻撃を防止するため、DAI を構築してください。
- **要件**:
  - SW1 上の VLAN 100 で DAI を有効化すること。
  - Gi1/0/1 〜 10 は端末用アクセスポートとし、DAI のデフォルト設定（Untrusted）を維持すること。
  - ルータ R1 へ接続する Gi1/0/24 ポート（トランクポート）を DAI の Trusted ポートに指定し、正当なルータの ARP がドロップされないようにすること。
- **設定例**:
```bash
! SW1 の設定
vlan 100
 name DATA_VLAN
!
ip dhcp snooping
ip dhcp snooping vlan 100
!
ip arp inspection vlan 100
!
interface range GigabitEthernet1/0/1 - 10
 switchport mode access
 switchport access vlan 100
 ip dhcp snooping limit rate 15
!
interface GigabitEthernet1/0/24
 switchport mode trunk
 switchport trunk allowed vlan 100
 ip arp inspection trust
 ip dhcp snooping trust
```
- **検証方法**:
  - `show ip arp inspection vlan 100` で DAI が VLAN 100 で有効であり、Forwarded/Dropped カウンタが正常に記録されているか確認。
  - `show ip arp inspection interfaces` で Gi1/0/24 が `Trusted`、Gi1/0/1-10 が `Untrusted` と表示されることを確認。

---

### シナリオ 2: 静的 IP 端末混在環境での ARP ACL 適用

- **問題**: VLAN 200 には DHCP クライアントの他に、静的 IP アドレスを持つファイルサーバー（IP: 192.168.200.10, MAC: 0050.56bb.1122）が接続されています。DAI を有効化しつつ、ファイルサーバーの通信がドロップされないよう構成してください。
- **要件**:
  - ARP ACL `ARP_ACL_SERVER` を作成し、ファイルサーバーの IP/MAC バインディングを許可すること。
  - DHCP クライアントの ARP パケットも正しく処理されるよう、`static` キーワードを使用せずに ARP ACL を VLAN 200 に適用すること。
- **設定例**:
```bash
! 1. ARP ACL の作成
arp access-list ARP_ACL_SERVER
 permit ip host 192.168.200.10 mac host 0050.56bb.1122
!
! 2. DAI 有効化および ARP ACL の紐付け（フォールスルー許可）
ip arp inspection vlan 200
ip arp inspection filter ARP_ACL_SERVER vlan 200
```
- **検証方法**:
  - `show arp access-list` で ACL エントリを確認。
  - サーバー（192.168.200.10）からの ARP Request/Reply がドロップされず正常に通り、`show ip arp inspection vlan 200` でドロップが発生していないことを確認。

---

### シナリオ 3: 厳格な ARP ヘッダー検証（ARP Validation）の有効化

- **問題**: ネットワーク内で不正な MAC アドレスや不整合な ARP パケットを検知・遮断するため、追加の ARP パケット検証オプションを有効化してください。
- **要件**:
  - 送信元 MAC アドレス不一致（Ethernet src MAC と ARP src MAC）、宛先 MAC アドレス不一致（Ethernet dst MAC と ARP dst MAC）、および無効 IP アドレス（0.0.0.0 やマルチキャスト IP）の 3 項目すべてを同時に検証・検査するよう構成すること。
- **設定例**:
```bash
! ARP ヘッダーの完全検証オプションの設定
ip arp inspection validate src-mac dst-mac ip
```
- **検証方法**:
  - `show running-config | include ip arp inspection validate` を実行し、`src-mac dst-mac ip` が有効になっていることを確認。
  - 不整合 MAC アドレスを持つテスト ARP を送信し、`show ip arp inspection log` に `Invalid MAC` や `Invalid IP` の理由でドロップログが記録されることを確認。

---

### シナリオ 4: DAI Rate Limit と Err-disable 自動復旧の構成

- **問題**: アクセスポート配下の端末やハブからの ARP バースト攻撃によりスイッチ CPU が高負荷になるのを防ぎ、かつ誤ってポートが遮断された際に自動復旧する仕組みを構築してください。
- **要件**:
  - ユーザーアクセスポート Gi1/0/5 の DAI レート制限を 20 pps に変更すること。
  - ARP レート制限違反によりポートが `err-disable` 状態となった場合、120 秒後に自動的に復旧するようグローバル設定を行うこと。
- **設定例**:
```bash
! 1. インターフェイスレベルの Rate Limit 調整
interface GigabitEthernet1/0/5
 ip arp inspection limit rate 20
!
! 2. Err-disable 自動復旧の設定
errdisable recovery cause arp-inspection
errdisable recovery interval 120
```
- **検証方法**:
  - `show ip arp inspection interfaces GigabitEthernet1/0/5` で `Rate (pps)` が 20 と表示されることを確認。
  - 疑似的にバースト ARP を送信し、`show interfaces status err-disabled` で原因が `arp-inspection` となっていることを確認後、120 秒後に自動復旧することを確認。

---

### シナリオ 5: DHCP Snooping と DAI の完全連携構築

- **問題**: 新規拠点スイッチ SW2 において、ゼロから DHCP Snooping と DAI を連携させた完全な Layer 2 セキュリティ基盤を構築してください。
- **要件**:
  - VLAN 10, 20 で DHCP Snooping および DAI を有効化すること。
  - DHCP サーバー接続ポート（Gi1/0/24）およびルータ接続ポート（Gi1/0/23）は DHCP Snooping と DAI の両方で Trust 設定すること。
- **設定例**:
```bash
! 全体有効化
ip dhcp snooping
ip dhcp snooping vlan 10,20
ip arp inspection vlan 10,20
!
! アップリンク / サーバーポートの設定
interface range GigabitEthernet1/0/23 - 24
 switchport mode trunk
 ip dhcp snooping trust
 ip arp inspection trust
!
! アクセスポートの設定
interface range GigabitEthernet1/0/1 - 10
 switchport mode access
 switchport access vlan 10
 ip dhcp snooping limit rate 10
 ip arp inspection limit rate 15
```
- **検証方法**:
  - クライアントが DHCP で IP を取得後、`show ip dhcp snooping binding` にバインディングが登録され、そのバインディングに基づき `show ip arp inspection vlan 10` で正常に ARP パケットが Forward されていることを確認。

---

### シナリオ 6: Private VLANs（PVLAN）環境での DAI 適用

- **問題**: マルチテナントデータセンターにおいて、Private VLAN（Primary VLAN 500, Isolated VLAN 501, Community VLAN 502）が構築されています。テナント間の ARP スプーフィングを防ぐため DAI を構成してください。
- **要件**:
  - Primary VLAN および Secondary VLAN のすべてで DAI が正しく機能するように構成すること。
  - 適切な VLAN 指定により DAI を有効化すること。
- **設定例**:
```bash
! PVLAN の構成（前提）
vlan 501
 private-vlan isolated
vlan 502
 private-vlan community
vlan 500
 private-vlan primary
 private-vlan association 501-502
!
! DAI の有効化（Primary および Secondary VLAN すべてを明示指定）
ip arp inspection vlan 500-502
```
- **検証方法**:
  - `show ip arp inspection vlan 500` で Primary および関連付けられた Secondary VLAN にて DAI がアクティブであり、Isolated ポート間の不審な ARP が正しくドロップされることを確認。

---

### シナリオ 7: トランクポートにおける DAI 不整合トラブルシューティング

- **問題**: SW1 と SW2 間のトランクポート経由で通信する VLAN 30 の端末同士が、互いに ARP 解決できず通信不能となっています。原因を調査し、修復してください。
- **要件**:
  - SW1/SW2 双方のインターフェイス設定を確認し、正しく DAI Trust が設定されていない箇所を特定・修正すること。
- **設定例（修正コンフィグ）**:
```bash
! SW1 および SW2 の対向トランクポート（Gi1/0/48）での修正
interface GigabitEthernet1/0/48
 switchport mode trunk
 ip arp inspection trust
```
- **検証方法**:
  - 修正前: `show ip arp inspection log` で対向スイッチから届く ARP が Untrusted ポート受信としてドロップされているログを確認。
  - 修正後: `show ip arp inspection interfaces GigabitEthernet1/0/48` で `Trust State` が `Yes` に変わり、端末間で Ping 通信が可能になることを確認。

---

### シナリオ 8: 静的 IP 専用 VLAN における `static` キーワードを使用した要塞化

- **問題**: 管理用 VLAN 999 は全端末が静的 IP で運用されており、DHCP サーバーは存在しません。未許可の外部端末による ARP 詐称を完全にシャットアウトするよう DAI を構成してください。
- **要件**:
  - ARP ACL `MGMT_STATIC_ARP` を定義し、許可された管理端末および Gateway のみを登録すること。
  - ACL にマッチしない ARP パケットは DHCP Snooping 参照を行わずに即座に拒否（`static` オプションを使用）すること。
- **設定例**:
```bash
! 1. 管理端末の ARP ACL 定義
arp access-list MGMT_STATIC_ARP
 permit ip host 10.99.99.10 mac host 0011.2233.4455
 permit ip host 10.99.99.254 mac host 0000.0c9f.f099
!
! 2. static キーワードを付与した DAI フィルター適用
ip arp inspection vlan 999
ip arp inspection filter MGMT_STATIC_ARP vlan 999 static
```
- **検証方法**:
  - 登録された 10.99.99.10 以外の任意の IP を手動設定した端末を接続し、ARP を送信した際に即座にドロップされ、`show ip arp inspection vlan 999` の `Explicit Filter Drops` カウンタがインクリメントされることを確認。

---

### シナリオ 9: 802.1X ポートセキュリティ環境と DAI の統合

- **問題**: 802.1X 認証が導入されたアクセスポートにおいて、認証成功後の動的 IP 割り当てに伴う ARP 詐称を防ぐため、DAI を併用構成してください。
- **要件**:
  - Access ポート Gi1/0/2 に 802.1X 認証と DAI を共に有効化すること。
  - 認証前および認証後の ARP トラフィックが DAI によって安全に検査されること。
- **設定例**:
```bash
interface GigabitEthernet1/0/2
 switchport mode access
 switchport access vlan 10
 authentication port-control auto
 dot1x pae authenticator
 ip dhcp snooping limit rate 15
 ip arp inspection limit rate 15
```
- **検証方法**:
  - サプリカント端末が 802.1X 認証に成功し、DHCP で IP アドレスを取得した後、`show ip dhcp snooping binding` に正しく登録され、かつ DAI を通過して双方向通信ができることを確認。

---

### シナリオ 10: DAI ログ制御および Logging Buffer のチューニング

- **問題**: 大規模ネットワークにおいて、不正 ARP パケットの発生時に大量の Syslog が生成され、コンソールおよびログサーバーが圧迫されるのを防ぐため、DAI のログレートを調整してください。
- **要件**:
  - DAI のログバッファサイズを 1024 エントリに変更すること。
  - ログ出力レートを 1 秒あたり最大 5 メッセージ、測定間隔を 5 秒に制限すること。
- **設定例**:
```bash
! DAI ログのチューニング設定
ip arp inspection log-buffer entries 1024
ip arp inspection log-buffer logs 5 interval 5
```
- **検証方法**:
  - `show ip arp inspection log` を実行し、`Syslog rate: 5 entries per 5 seconds` および `Buffer Size: 1024` と表示されることを確認。

---



## ❓ 想定試験問題

CCIE Security ラボ試験・記述問題レベルの想定試験問題セットです。

### 問題 1（コンフィグ読解）
以下のコンフィグが適用されたスイッチ SW1 において、VLAN 10 に接続された新規 DHCP クライアント（端末 A）が、デフォルトゲートウェイへの Ping 通信を行えません。原因として最も適切なものを選択し、その理由と修正コマンドを述べてください。

```bash
arp access-list VIP_HOSTS
 permit ip host 10.1.1.50 mac host 0050.56bf.1111
!
ip arp inspection vlan 10
ip arp inspection filter VIP_HOSTS vlan 10 static
```

- **選択肢**:
  - A) `ip dhcp snooping` が設定されていないため。
  - B) `static` キーワードにより、ARP ACL にマッチしない全 DHCP クライアントの ARP が即座にドロップされているため。
  - C) `ip arp inspection trust` がアクセスポートに設定されていないため。
  - D) ARP ACL のステートメント末尾に `permit ip any any` が記載されていないため。

- **正解**: **B**
- **解説**: `ip arp inspection filter <acl-name> vlan <vlan-id> static` コマンドで `static` キーワードを指定すると、ARP ACL で明示的に許可されていないパケットは、DHCP Snooping バインディング データベースの参照を行わずに即座にドロップ（Implicit Deny）されます。端末 A は DHCP クライアントであり ARP ACL に未登録であるため、すべての ARP パケットが拒否されます。
- **修正コマンド**:
  ```bash
  ip arp inspection filter VIP_HOSTS vlan 10
  ```
  （`static` キーワードを削除することで、ACL 未マッチ時に DHCP Snooping 表をフォールスルー評価するようになります。）

---

### 問題 2（トラブルシュート）
SW1 と SW2 をトランク接続（Gi1/0/48）した環境において、SW1 側の VLAN 20 端末から SW2 側の VLAN 20 端末への Ping が突然不通となりました。SW1 の `show ip arp inspection log` を確認すると、以下のメッセージが頻発しています。

```text
%SW_DAI-4-DHCP_SNOOPING_DENY: 1 Invalid ARPs (R) on Gi1/0/48 , vlan 20.([0010.1122.3344/192.168.20.1/0000.0000.0000/0.0.0.0/01:23:45 UTC Mon Nov 2 2026])
```

この障害の根本原因と最も適切な対処方法を記述してください。

- **解答・解説**:
  - **原因**: SW1 のトランクポート Gi1/0/48 において `ip arp inspection trust` が設定されていない（デフォルトの Untrusted 状態である）ため、対向の SW2 配下の端末から届いた正当な ARP Reply/Request パケットが、SW1 上の DHCP Snooping 表に存在しないという理由でドロップされています。
  - **対処方法**: SW1 および SW2 の両スイッチ間トランクポートにおいて、`ip arp inspection trust` を設定します。
    ```bash
    interface GigabitEthernet1/0/48
     ip arp inspection trust
    ```

---

### 問題 3（Design）
大規模エンタープライズのアクセス層において、DHCP Snooping、DAI、IP Source Guard (IPSG)、および Port Security を併用する場合の設計上の優先度および処理順序について正しく説明している記述を選びなさい。

- **選択肢**:
  - A) DAI はハードウェア TCAM で処理されるため、Port Security よりも前に実行される。
  - B) パケット受信時、Layer 2 の Port Security (MAC 制限) が最初に適用され、次に DAI による ARP 検査、データパケットに対して IPSG (IP/MAC フィルタ) が適用される。
  - C) IPSG と DAI は独立しており、DHCP Snooping データベースを共有することはできない。
  - D) DAI を有効にした場合、Port Security は自動的に無効化される。

- **正解**: **B**
- **解説**: 受信フレーム処理において、Layer 2 MAC アドレスの検証（Port Security）が最初に実行され、許可されたフレームのうち ARP パケットは DAI（CPU パス）で検査され、IPv4 データパケットは IPSG（TCAM）でフィルタリングされます。DAI と IPSG はともに DHCP Snooping バインディング表を共通の信頼できる情報源として参照します。

---

### 問題 4（実装・エラー解析）
スイッチのアクセスポート Gi1/0/10 に接続された端末が、ネットワークスキャンツールを起動した直後に通信不能となりました。`show interfaces status` を実行したところ、該当ポートが `err-disabled` 状態となっていました。この現象が発生した理由と、自動復旧させるための設定を記述してください。

- **解答・解説**:
  - **発生理由**: 端末がスキャンツールによって短時間に大量の ARP パケット（バースト）を送信したため、アクセスポートのデフォルト制限値（15 pps）を超過し、DAI の Rate Limit 違反によってポートが `err-disable` 状態に遷移しました。
  - **対処設定**:
    1. 該当ポートのレート制限値を運用要件に合わせて引き上げる。
    2. 自動復旧メカニズムを設定する。
    ```bash
    interface GigabitEthernet1/0/10
     ip arp inspection limit rate 30
    !
    errdisable recovery cause arp-inspection
    errdisable recovery interval 180
    ```

---

### 問題 5（トラブルシュート）
ロードバランサ（HA 構成）を接続している VLAN 50 において、`ip arp inspection validate src-mac dst-mac ip` を有効化後、ロードバランサ間の仮想 IP（VIP）の引き継ぎ（GARP）に伴う通信切り替えが正常に行われなくなりました。理由と回避策を説明してください。

- **解答・解説**:
  - **理由**: 一部のロードバランサやクラスタリングソフトが送信する Gratuitous ARP（GARP）や共有 MAC アドレスパケットでは、Ethernet ヘッダーの宛先 MAC アドレス（例: ブロードキャスト ff-ff-ff-ff-ff-ff）と ARP パケット内部の Target MAC アドレス（例: 00-00-00-00-00-00）が一致しない構造を持つ場合があります。`dst-mac` 検証を有効にすると、これが「不整合な MAC」と判定されてドロップされます。
  - **回避策**: `dst-mac` 検証のみを外し、`src-mac` および `ip` のみの検証に変更します。
    ```bash
    ip arp inspection validate src-mac ip
    ```

---

### 問題 6（コンフィグ読解）
以下の設定において、スイッチが実行するパケット検証の順序および結果として正しいものを選択してください。

```bash
arp access-list FILTER_STATIC
 permit ip host 10.2.2.100 mac host 0011.2233.4455
!
ip arp inspection vlan 20
ip arp inspection filter FILTER_STATIC vlan 20
ip arp inspection validate ip
```

- **選択肢**:
  - A) `validate ip` の検査が最初に実行され、無効 IP が検出された場合は ARP ACL や DHCP 表の検証を行わずに即ドロップされる。
  - B) ARP ACL の検証が最後に行われる。
  - C) DHCP Snooping 表が ARP ACL よりも優先して参照される。
  - D) `validate ip` は ARP ACL で許可されたパケットに対しては実行されない。

- **正解**: **A**
- **解説**: DAI の処理シーケンスでは、まずヘッダーレベルの検証（`validate` オプション: `src-mac`, `dst-mac`, `ip`）が最初に実行され、これに違反した不整合パケットは即座にドロップされます。その後に ARP ACL の評価、最後に DHCP Snooping バインディング表の照合が行われます。

---



## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術資料一覧です。

- Cisco Catalyst Switch Configuration Guide - Configuring Dynamic ARP Inspection
- Cisco Catalyst Command Reference - ip arp inspection Commands
- Cisco Technical Support & Documentation - Dynamic ARP Inspection (DAI) Troubleshooting and Best Practices
- Cisco Live On-Demand Library - BRKSEC-2001 Enterprise Layer 2 Security Fundamentals
- Cisco Validated Design (CVD) - Campus LAN and Wireless LAN Security Design Guide

---

## 📝 **補足（Notes）**

### DAI 処理フローとパケット判定マトリクスまとめ

DAI 評価時のパケット処理判定フローを以下のマトリクスに整理します。直前復習時に活用してください。

| 受信ポート種別 | 条件 | 処理結果 | 備考 |
| --- | --- | --- | --- |
| **Trusted Port** | 任意の ARP パケット | **無条件許可（Forward）** | CPU 検査をバイパス |
| **Untrusted Port** | `validate` オプション不適合 | **即時ドロップ（Drop）** | 理由: Invalid MAC / IP |
| **Untrusted Port** | ARP ACL にマッチ (`permit`) | **許可（Forward）** | DHCP Snooping 表は参照しない |
| **Untrusted Port** | ARP ACL にマッチ (`deny`) | **即時ドロップ（Drop）** |  |
| **Untrusted Port** | ARP ACL 未マッチ + `static` なし | **DHCP 表を照合** | IP + MAC + VLAN の完全一致で Forward |
| **Untrusted Port** | ARP ACL 未マッチ + `static` あり | **即時ドロップ（Drop）** | DHCP 表の照合を行わない |
| **Untrusted Port** | Rate Limit 閾値（pps）超過 | **ポート遮断 (`err-disable`)** | Syslog 通知発生 |

---


