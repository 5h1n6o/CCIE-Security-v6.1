---
layout: default
title: 3.4.e-DHCP-snooping
nav_order: 5
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.4.e DHCP snooping

## 📘 概要

* **機能概要**: DHCP Snooping（DHCPスノーピング）は、CatalystスイッチなどのLayer 2ネットワークデバイスで動作するファーストホップセキュリティ（First Hop Security: FHS）機能です。信頼できない（Untrusted）ポートと信頼できる（Trusted）ポートを論理的に分離し、DHCPメッセージをスヌーピング（盗み見・検査）することで、悪意のあるDHCPサーバーの乱入（Rogue DHCP Server）やDHCPスプーフィンク攻撃、DHCP飢餓攻撃（DHCP Starvation Attack）を防ぎます。また、正常に割り当てられたIP/MAC/VLAN/ポートのバインディング情報を動的に記録する「DHCP Snooping Binding Database（バインディングデータベース）」を構築・維持するコア基盤としても機能します。
* **利用目的**: 
  1. **不正DHCPサーバー（Rogue DHCP Server）の遮断**: 許可されていないDHCPサーバーがクライアントに対して誤ったIPアドレス、デフォルトゲートウェイ、DNSサーバー情報を配布し、Man-in-the-Middle（MitM）攻撃やサービス拒否（DoS）を引き起こすことを防止します。
  2. **セキュリティバインディングデータベースの自動生成**: Dynamic ARP Inspection（DAI）や IP Source Guard（IPSG）などの上位Security機能が機能するための信頼性の高いIP-MAC-Port-VLANマッピングテーブルを提供します。
  3. **DHCPメッセージの整合性検証**: Untrustedポートから受信したDHCPパケットのDHCPクライアントMACアドレス（chaddr）とイーサネット送信元MACアドレスの不一致チェックや、不当なDHCPサーバー応答（OFFER, ACK, NAK, LEASEQUERY）のドロップを行います。
  4. **DHCP Rate Limiting**: Untrustedポートからの過剰なDHCPリクエスト（Discover/Request）を制限し、スイッチCPUの高負荷やDHCPサーバーの飢餓（Starvation）を防止します。
* **どのような場面で利用するか**: 企業LAN環境のアクセス層（Access Layer）スイッチにおいて、すべてのエンドポイント接続ポートに適用します。ユーザーが持ち込んだWi-FiルーターのLANポート誤接続によるIP競合トラブル防止や、ゼロトラスト環境におけるインフラ防御の第一線として必須で配置されます。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **特徴** | Layer 2スイッチでDHCPメッセージをインターセプトし、Trusted/Untrustedポート制御とIP-MACバインディングDB構築を行うFHS機能 |
| **用途** | 不正DHCPサーバー遮断、DHCP Starvation防止、DAI/IPSG用バインディングデータ生成、Option 82挿入/削除 |
| **メリット** | トラフィック透過的に中間者攻撃やIPスプーフィングを根絶。DAI/IPSGとの連携で完全なLayer 2保護を実現 |
| **デメリット/オーバーヘッド** | オプション82処理不整合による通信断リスク。バインディングDBのフラッシュ（再起動時等）による一時的な通信拒否リスク |
| **対応機種** | Cisco Catalystシリーズ（2960, 3650, 3850, 9200, 9300, 9400, 9500等）、Cisco Nexusシリーズ、Cisco Industrial Ethernet |
| **制限事項** | 信頼できるDHCPサーバーへの経路上の全アップリンク/トランクポートを「Trusted」として明示設定する必要あり。Private VLAN環境での特殊な考慮 |
| **設計上の注意点** | Option 82のデフォルト挿入動作（`giaddr`が0.0.0.0でOption 82が付与されたパケットを上流のCisco IOS DHCPリレー/サーバーが拒否する問題）への対策 |

---

## 🏗 動作原理

DHCP Snoopingは、スイッチポートを以下の2つのカテゴリに分類して動作します。

1. **Trusted Port（信頼できるポート）**:
   - 正当なDHCPサーバーやDHCPリレーエージェントが接続されているポート（通常はアップリンクポートやトランクポート）。
   - DHCP OFFER, ACK, NAK などのすべてのDHCPメッセージの送受信が許可されます。
2. **Untrusted Port（信頼できないポート）**:
   - ユーザーPC、IP電話、プリンタなどのエンドポイントが接続されているポート（アクセスポート）。
   - クライアントからのメッセージ（DISCOVER, REQUEST, RELEASE, DECLINE）のみ受信が許可されます。
   - サーバーからの応答メッセージ（OFFER, ACK, NAK, LEASEQUERY）を受信した場合、スイッチは即座にパケットを破棄し、セキュリティ違反イベントを記録します。

### 通信フロー図

```
[ Rogue DHCP Server ] (受信用メッセージを即破棄)
       │
       ▼ (DHCP OFFER)
[ Untrusted Port (Gi0/2) ]
       │
       ├─► 【DHCP Snooping Engine】
       │      ├─ 応答メッセージチェック ──► [ DROP & LOG ]
       │      ├─ MAC Verification (chaddr vs Src-MAC)
       │      └─ Rate Limit Check
       │
[ Switch (Catalyst) ]
       │
[ Trusted Port (Gi0/1) ] ◄── (DHCP OFFER / ACK 許可) ── [ Legitimate DHCP Server ]
       │
       ▼ (DHCP DISCOVER / REQUEST 転送 & DB更新)
[ Client PC ]
```

### パケット検証ロジック

Untrustedポートで受信したDHCPパケットに対して、スイッチは以下の条件を順番に検証します。

1. **メッセージタイプのチェック**: 受信パケットが DHCP OFFER, ACK, NAK, LEASEQUERY などのサーバー応答メッセージである場合、即座に破棄。
2. **chaddr検証（MAC Address Verification）**: `ip dhcp snooping verify mac-address` が有効な場合、Ethernetヘッダーの送信元MACアドレスと、DHCPペイロード内の `chaddr`（Client Hardware Address）フィールドが一致するか検証。不一致の場合は破棄（DHCP Starvation攻撃防止）。
3. **Option 82 & giaddrのチェック**: Untrustedポートで受信したパケットにすでにOption 82が含まれているか、または `giaddr`（Relay Agent IP）が0.0.0.0以外に設定されている場合、不正なリレーパケットとみなして破棄（設定により許可可能）。
4. **Rate Limitチェック**: 設定された1秒あたりのDHCPパケット数（pps）を超過した場合、ポートを `err-disable` 状態に遷移させてブロック。

---

## ⚙ 動作シーケンス

DHCPクライアントが正常にIPアドレスを取得し、バインディングエントリーが作成されるまでのシーケンスは以下の通りです。

```
Client (Untrusted)            Switch (Snooping Engine)           DHCP Server (Trusted)
      │                                 │                                 │
      │── 1. DHCP DISCOVER ────────────►│                                 │
      │   (Src-MAC: A, chaddr: A)       │─ 2. Option 82 挿入 (Circuit/Remote)
      │                                 │   (※機能有効時)                 │
      │                                 │────────────────────────────────►│
      │                                 │                                 │
      │                                 │◄── 3. DHCP OFFER ───────────────│
      │                                 │    (Giaddr: 0.0.0.0, Your-IP: X)│
      │◄── 4. Option 82 除去 ───────────│                                 │
      │    (Clientへ転送)               │                                 │
      │                                 │                                 │
      │── 5. DHCP REQUEST ─────────────►│                                 │
      │                                 │────────────────────────────────►│
      │                                 │                                 │
      │                                 │◄── 6. DHCP ACK ─────────────────│
      │                                 │    (IP: X, MAC: A, Lease: T)    │
      │                                 │                                 │
      │                                 ├─ 7. Snooping DBエントリー追加   │
      │                                 │    [MAC: A, IP: X, VLAN: V,     │
      │                                 │     Port: Gi0/2, Lease: T]      │
      │◄── 8. ClientへACK転送 ──────────│                                 │
```

### バインディングデータベース構築詳細

DHCP ACKを受信した時点で、スイッチは以下の情報をバインディングDBに登録します。

- **MAC Address**: クライアントのハードウェアアドレス（chaddr）
- **IP Address**: 割り当てられたIPアドレス（Your IP Address）
- **Lease Time**: リース時間（秒）
- **Binding Type**: `dhcp-snooping`（動的取得）または `static`（静的設定）
- **VLAN ID**: パケットを受信したVLAN
- **Interface**: クライアントが接続されているUntrustedポート

---

## 🎯 試験対策（CCIE Securityラボ試験）

### 1. Blueprintで重要なポイント
- **Option 82のデフォルト動作とトラブルシューティング**: CiscoスイッチでDHCP Snoopingを有効にすると、デフォルトでOption 82（Relay Agent Information Option）が挿入されます。これにより、Untrustedアクセスポートから受信したパケットの `giaddr` が `0.0.0.0` のままOption 82が付与されて送信されます。非L3リレーのL2スイッチ環境において、上流のCisco IOS DHCPサーバーまたはリレーエージェントは、`giaddr = 0.0.0.0` かつ Option 82が存在するパケットをデフォルトで破棄します。この回避策として `ip dhcp relay information trust-all` または `ip dhcp bootp ignore information option` を上流ルーターに設定するか、L2スイッチ側で `no ip dhcp snooping information option` を設定する問題が多発します。
- **UntrustedポートでのOption 82パケット受信制限**: UntrustedポートでOption 82が既に付与されたDHCPリクエストを受信した場合、デフォルトでスイッチはパケットを破棄します（`DHCP_SNOOPING-2-DROP_IN_UNTRUSTED`）。アクセススイッチが多段構成（Cascaded Switches）になっている場合、下流スイッチからのOption 82付きパケットを受ける上流スイッチのUntrustedポートで `ip dhcp snooping information option allow-untrusted` を設定させる要件が出題されます。
- **Rate LimitとErrdisableアセット**: `ip dhcp snooping limit rate <value>` をUntrustedポートに設定し、過剰なDHCPパケットでポートが `err-disable` になった場合の `errdisable recovery cause dhcp-rate-limit` 設定。

### 2. ラボ試験で設定させられそうな内容
- 特定のVLAN（例: VLAN 10, 20）のみでDHCP Snoopingを有効化。
- アップリンクポート（例: Gi1/0/48）を `ip dhcp snooping trust` に設定。
- Snooping Binding DatabaseをフラッシュメモリまたはTFTP/FTPサーバーに永続化保存（`ip dhcp snooping database flash:dhcp_snoop.db`）。
- 意図的なOption 82 Circuit-ID / Remote-ID のカスタムフォーマット設定（`ip dhcp snooping information option format-type`）。

### 3. よくある設定ミス・ハマりポイント
- **VLANに対するSnooping有効化漏れ**: `ip dhcp snooping`（グローバルコマンド）のみを設定し、`ip dhcp snooping vlan 10`（VLAN指定コマンド）を設定し忘れる。
- **Trustedポートの設定漏れ**: スイッチ間のトランクポートやDHCPサーバー接続ポートで `ip dhcp snooping trust` を設定し忘れ、DHCP OFFER/ACK が途中で破棄されてクライアントがIPアドレスを取得できなくなる。
- **Option 82の不整合によるサイレントドロップ**: リレーエージェントでないL2スイッチでOption 82が挿入され、DHCPサーバー（Cisco IOS/IOS-XE）側で `ip dhcp relay information trust-all` が無いために破棄される。

### 4. コンフィグ読解・showコマンド解説
- `show ip dhcp snooping`: グローバルおよびVLANごとの有効化状態、Option 82の設定、Trustedポート一覧、データベース状態を確認。
- `show ip dhcp snooping binding`: 動的に構築されたIP-MAC-Port-VLANバインディングテーブルの表示。
- `show ip dhcp snooping statistics`: 破棄されたパケットのカウンターと原因（Rogue Server, MAC Mismatch, Invalid Option 82等）を詳細解析。

```text
# show ip dhcp snooping
Switch DHCP snooping is enabled
DHCP snooping is configured on following VLANs:
10,20,30
DHCP snooping is operational on following VLANs:
10,20,30
DHCP snooping is configured to insert option 82: YES
Option 82 option configuration:
  circuit-id format: vlan-mod-port
  remote-id format: MAC
Option 82 on untrusted ports is not allowed
Verification of hwaddr field is enabled
DHCP snooping trust/rate is configured the following way:

Interface                  Trusted    Allow option    Rate limit (pps)
-----------------------    -------    ------------    ----------------
GigabitEthernet1/0/1       No         No              15
GigabitEthernet1/0/48      Yes        Yes             unlimited
```


## 🛠 設定方法

### 1. 基本的なDHCP Snoopingの構成手順（Cisco IOS-XE / Catalyst）

```bash
# 1. グローバルでDHCP Snoopingを有効化
Switch(config)# ip dhcp snooping

# 2. 対象のVLANでDHCP Snoopingを有効化
Switch(config)# ip dhcp snooping vlan 10,20,30

# 3. MACアドレス検証（chaddr vs Src-MAC）の有効化（デフォルトで有効）
Switch(config)# ip dhcp snooping verify mac-address

# 4. DHCPサーバー / リレーが接続されているポートをTrustedに設定
Switch(config)# interface GigabitEthernet1/0/48
Switch(config-if)# description === Trunk to Core / DHCP Server ===
Switch(config-if)# ip dhcp snooping trust

# 5. Untrustedアクセスポートに対するRate Limit設定（DoS防止）
Switch(config)# interface range GigabitEthernet1/0/1 - 24
Switch(config-if)# description === Access Ports ===
Switch(config-if)# ip dhcp snooping limit rate 15

# 6. Option 82を挿入しない場合（L2スイッチ環境でトラブル回避時）
Switch(config)# no ip dhcp snooping information option

# 7. (オプション) 多段スイッチ構成でUntrustedポートでのOption 82パケット受信を許可する場合
Switch(config)# ip dhcp snooping information option allow-untrusted

# 8. バインディングデータベースの外部保存（再起動時のデータ維持）
Switch(config)# ip dhcp snooping database flash:dhcp_snooping_db.bin
Switch(config)# ip dhcp snooping database write-delay 60
```

### 2. 上流Cisco IOS DHCPサーバー側でのOption 82対策設定例

```bash
# Cisco IOSルーターがDHCPサーバー兼リレーの場合、Option 82（giaddr=0.0.0.0）を許可
Router(config)# ip dhcp relay information trust-all

# または、BOOTP/DHCPメッセージ内のOption 82情報を無視してアドレス割り当てを実行
Router(config)# ip dhcp bootp ignore information option
```

---

## 🔍 検証コマンド

| 目的 | コマンド | 判定基準・確認ポイント |
| --- | --- | --- |
| 動作状態確認 | `show ip dhcp snooping` | グローバルおよびVLANごとのSnooping状態、Option 82設定、Trustedポート一覧の確認 |
| バインディングDB確認 | `show ip dhcp snooping binding` | IP-MAC-Port-VLANマッピング、リース時間、タイプ（dhcp-snooping）の正常登録の確認 |
| 統計情報確認 | `show ip dhcp snooping statistics` | 破棄されたパケット数、Rogue ServerからのOFFER拒否数、MAC不一致破棄数の解析 |
| データベースエージェント確認 | `show ip dhcp snooping database` | 永続化ストレージへの書き込み成功ログ、エージェントURL、最終同期時間の確認 |
| リアルタイムデバッグ | `debug ip dhcp snooping packet` | DHCPパケットの受信、Option 82処理、パケット破棄のリアルタイムログ出力 |
| リアルタイムイベントデバッグ | `debug ip dhcp snooping event` | バインディングDB追加/削除、ポート制限違反イベントの監視 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| クライアントがIPアドレスを取得できない（DORAのOFFERが届かない） | アップリンクポートで `ip dhcp snooping trust` が未設定 | `show ip dhcp snooping` | DHCPサーバー側のトランク/アクセスポートで <code>ip dhcp snooping trust</code> を設定 |
| DHCPパケットが上流ルーターで破棄される | L2スイッチがOption 82を挿入し、上流Ciscoルーターの `giaddr` チェックで拒否 | `show ip dhcp snooping statistics` | スイッチ側で <code>no ip dhcp snooping information option</code> を設定するか、ルーター側で <code>ip dhcp relay information trust-all</code> を設定 |
| 多段スイッチ環境で下流アクセススイッチ配下の端末が通信不能 | 上流スイッチのUntrustedトランクポートがOption 82付きパケットを受信して破棄 | `debug ip dhcp snooping packet` | 上流スイッチの該当ポートで <code>ip dhcp snooping information option allow-untrusted</code> を設定するか、トランクポートをTrustedに変更 |
| ポートが `err-disable` になる | クライアントまたは悪意ある端末が設定されたRate Limit（例: 15pps）を超過してパケットを送信 | `show interface status err-disabled` | <code>ip dhcp snooping limit rate</code> の閾値調整、または <code>errdisable recovery cause dhcp-rate-limit</code> を設定 |
| スイッチ再起動後にDAI/IPSGで全通信が拒否される | DHCP Snooping Binding Databaseが永続化ストレージに保存されておらず消滅 | `show ip dhcp snooping binding` | <code>ip dhcp snooping database flash:snoop.db</code> を設定しDBを保存 |

---

## ⚠ 制限事項

1. **CPU負荷とハードウェアリソース**:
   - DHCP Snoopingはコントロールプレーン（CPU）でパケットをスヌーピングおよび検証します。極めて高いパケットレート（大量のDISCOVERフラッド等）が発生すると、スイッチのCPU高負荷を引き起こすため、Untrustedポートでの Rate Limiting 設定が不可欠です。
2. **In-Band Management / SVIとの関係**:
   - スイッチ自身がDHCPクライアントとして管理IPアドレス（SVI）を取得する場合、該当VLANのインバンドポートがTrustedに指定されていないと、自身のDHCP OFFERも破棄されるリスクがあります。
3. **Private VLAN（PVLAN）での制限**:
   - PVLAN構成では、Promiscuousポート、Isolatedポート、CommunityポートでのSnooping設定において、Primary VLANに対してDHCP Snoopingを有効化する必要があります。Secondary VLAN側のみで有効化しても機能しません。

---

## 🔄 他技術との関連

* **Dynamic ARP Inspection (DAI)**:
  - DAIはARPパケットのIP-MACバインディングを検証する際、DHCP Snooping Binding Databaseを参照します。DHCP Snoopingが正しく動作していないと、DAIはすべての正規ARPパケットをドロップしてしまいます。
* **IP Source Guard (IPSG)**:
  - IPSGはデータポートでのIP/MACスプーフィングを防ぐため、DHCP Snooping Binding Databaseを元にポートアクセスACL（PACL）を動的生成します。
* **802.1X / MAC Authentication Bypass (MAB)**:
  - 認証成功後のIPアドレス確定およびCisco ISEへのDevice Tracking情報伝達（IPDT/SISF）において、Snoopingデータベースが連携します。
* **Cisco TrustSec (SGT)**:
  - IP-SGTバインディングを動的に生成・学習する際、DHCP Snoopingから取得したIPアドレス情報が利用されます。

---

## 🧩 比較表

### DHCP Snooping ポートモード比較

| 項目 | Trusted Port（信頼できるポート） | Untrusted Port（信頼できないポート） |
| --- | --- | --- |
| **主な接続先** | 正当なDHCPサーバー、DHCPリレー、上流スイッチ | ユーザーPC、IP電話、アクセスポート |
| **DHCP Server応答 (OFFER/ACK)** | **許可**（通過させる） | **拒否**（パケット破棄＆ログ記録） |
| **DHCP Client要求 (DISCOVER/REQ)** | **許可** | **許可**（検閲・検証後に通過） |
| **Option 82の自動挿入** | 行わない | 行う（デフォルト有効時） |
| **Option 82既存パケットの受信** | **許可** | **原則拒否**（allow-untrusted無しの場合はドロップ） |
| **Rate Limitの必要性** | 不要（通常unlimited） | **推奨**（デフォルトunlimited、通常15〜30ppsを設定） |
| **Binding DBへの登録** | 登録しない | **登録する**（ACK応答検知時に動的追加） |

---

## 💡 ベストプラクティス

1. **アクセス層スイッチでの標準配備**:
   - アクセスポートはすべて Untrusted、上流（Core/Distribution）へのトランクポートはすべて Trusted に明示設定します。
2. **Option 82の明確な方針決定**:
   - L2アクセススイッチ構成でリレー機能を使用しない場合は、`no ip dhcp snooping information option` を設定し、パケット処理のオーバーヘッドと上流での不必要なドロップトラブルを未然に回避します。
3. **Snooping Databaseの永続化**:
   - ネットワーク障害やスイッチ再起動時の通信障害を防ぐため、必ず `ip dhcp snooping database flash:dhcp_db.bin` を設定し、`write-delay 60` 程度でストレージに保存します。
4. **Rate Limitの適切な閾値設定**:
   - 通常の端末ポートには `15` 〜 `30` pps 程度を設定し、意図しないフラッディングからの保護と `err-disable recovery cause dhcp-rate-limit 30` による自動復旧を組み込みます。


## 📝 ラボ学習・設定サンプル例

### シナリオ1: 基本的なDHCP Snoopingの導入とTrustedポートの指定

* **問題**: VLAN 10およびVLAN 20のユーザー環境において、不正DHCPサーバーの接続を遮断し、IPアドレス割り当て情報を保護する設定を行ってください。
* **要件**:
  - VLAN 10, 20でDHCP Snoopingを有効化すること。
  - 上流のDHCPサーバーに接続するポート `GigabitEthernet1/0/48` を信頼できるポートとして指定すること。
  - アクセスポート `GigabitEthernet1/0/1 - 20` は信頼できないポートとすること。
* **設定例**:
  ```bash
  Switch(config)# ip dhcp snooping
  Switch(config)# ip dhcp snooping vlan 10,20
  Switch(config)# interface GigabitEthernet1/0/48
  Switch(config-if)# ip dhcp snooping trust
  ```
* **検証方法**:
  `show ip dhcp snooping` を実行し、VLAN 10, 20でSnoopingがOperationalになっており、Gi1/0/48がTrustedリストに含まれていることを確認します。

---

### シナリオ2: L2スイッチにおけるOption 82挿入機能の無効化（ドロップ問題の回避）

* **問題**: DHCP Snoopingを有効化したところ、クライアントがIPアドレスを取得できなくなりました。上流のCiscoルーターがOption 82付きパケット（giaddr=0.0.0.0）を破棄していることが原因です。
* **要件**:
  - L2スイッチ側でOption 82の挿入を無効化し、クライアントが正常にIPアドレスを取得できるように設定してください。
* **設定例**:
  ```bash
  Switch(config)# no ip dhcp snooping information option
  ```
* **検証方法**:
  クライアント側で `ipconfig /renew` を実行し、IPアドレスが取得できることを確認します。`show ip dhcp snooping` で `DHCP snooping is configured to insert option 82: NO` と表示されることを確認します。

---

### シナリオ3: 多段スイッチ（Cascaded Switch）環境における allow-untrusted の設定

* **問題**: SW1（アクセススイッチ）とSW2（ディストリビューションスイッチ）が接続されています。SW1でOption 82を有効化した状態で、SW2のUntrustedポート経由でパケットが受信され破棄されています。
* **要件**:
  - SW2のSW1接続ポート（`GigabitEthernet1/0/24`）はUntrustedのままとしつつ、SW1から送信されたOption 82付きパケットを受信許可するように設定してください。
* **設定例**:
  ```bash
  SW2(config)# interface GigabitEthernet1/0/24
  SW2(config-if)# ip dhcp snooping information option allow-untrusted
  ```
* **検証方法**:
  SW1配下の端末からDHCPリクエストを送信し、SW2で `show ip dhcp snooping statistics` を確認して `Option 82 on untrusted port` によるドロップカウンターが増加しないことを確認します。

---

### シナリオ4: DHCP Rate Limit と Errdisable 自動復旧の設定

* **問題**: 悪意のあるユーザーによるDHCP Starvation攻撃や大量のDHCPリクエストの乱発からスイッチCPUを保護してください。
* **要件**:
  - アクセスポート `GigabitEthernet1/0/1 - 10` のDHCPパケット受信レートを 15 pps に制限すること。
  - 制限を超過してポートが `err-disable` になった場合、30秒後に自動復旧させること。
* **設定例**:
  ```bash
  Switch(config)# interface range GigabitEthernet1/0/1 - 10
  Switch(config-if)# ip dhcp snooping limit rate 15
  Switch(config)# errdisable recovery cause dhcp-rate-limit
  Switch(config)# errdisable recovery interval 30
  ```
* **検証方法**:
  擬似攻撃ツール等で大量のDHCP DISCOVERを送信し、`show interface status err-disabled` で該当ポートが `err-disabled` に移行した後、30秒後に自動復旧することを確認します。

---

### シナリオ5: バインディングデータベースのFlashストレージ永続化設定

* **問題**: スイッチが再起動した際、DHCP Snooping Binding Databaseが消滅し、後続のDAI/IPSGによって全通信がブロックされる事態を防止してください。
* **要件**:
  - バインディングDBを `flash:dhcp_snoop_db.bin` ファイルとして保存すること。
  - データベースの書き込み遅延タイマーを 60 秒に設定すること。
* **設定例**:
  ```bash
  Switch(config)# ip dhcp snooping database flash:dhcp_snoop_db.bin
  Switch(config)# ip dhcp snooping database write-delay 60
  ```
* **検証方法**:
  `show ip dhcp snooping database` を実行し、URLが `flash:dhcp_snoop_db.bin` となっており、`Read status: Successful` および `Write status: Successful` であることを確認します。

---

### シナリオ6: 静的バインディングエントリ（Static Binding）の手動登録

* **問題**: 固定IPアドレスが手動設定されているサーバー（IP: `10.1.1.100`, MAC: `0011.2233.4455`）が `GigabitEthernet1/0/5`（VLAN 10）に接続されています。
* **要件**:
  - DHCP Snooping Binding Table にこの手動設定サーバーの静的エントリーを追加してください。
* **設定例**:
  ```bash
  Switch(config)# ip dhcp snooping binding 0011.2233.4455 vlan 10 10.1.1.100 interface GigabitEthernet1/0/5 expiry 4294967295
  ```
* **検証方法**:
  `show ip dhcp snooping binding` を実行し、Typeが `static` として追加されていることを確認します。

---

### シナリオ7: MAC Address Verification（chaddr一致検証）の有効化

* **問題**: クライアントが自らの送信元MACアドレスとは異なる偽装された `chaddr` を使用して大量のDHCPリース要求を送信する攻撃を防ぎます。
* **要件**:
  - EthernetヘッダーのSrc-MACとDHCPペイロード内の `chaddr` が一致しないパケットを自動破棄するよう設定してください。
* **設定例**:
  ```bash
  Switch(config)# ip dhcp snooping verify mac-address
  ```
* **検証方法**:
  `show ip dhcp snooping` で `Verification of hwaddr field is enabled` を確認します。MACアドレスを偽装したパケットを送信し、`show ip dhcp snooping statistics` の `MAC verification failure` カウンターが増加することを確認します。

---

### シナリオ8: 上流Cisco IOSルーター（DHCP Server）側でのOption 82信頼設定

* **問題**: L2スイッチでOption 82が挿入されて送られてくるDHCPリクエストに対して、Cisco IOS DHCPサーバーがIPアドレスを正常に割り当てるようルーター側で設定します。
* **要件**:
  - ルーターの全インターフェイスでOption 82付きリクエスト（`giaddr = 0.0.0.0`）を信頼して受け入れること。
* **設定例**:
  ```bash
  Router(config)# ip dhcp relay information trust-all
  ```
* **検証方法**:
  ルーター側で `debug ip dhcp server packet` を有効化し、`DHCPDISCOVER received with Option 82... Relay info trust-all enabled` のログと共に `DHCPOFFER` が送信されることを確認します。

---

### シナリオ9: Private VLAN（PVLAN）環境でのDHCP Snooping設定

* **問題**: PVLAN（Primary VLAN: 100, Secondary Isolated: 101, Secondary Community: 102）が設定されたスイッチでDHCP Snoopingを構成します。
* **要件**:
  - Private VLAN環境で正常にDHCP Snoopingが動作するようにVLAN指定を行ってください。
* **設定例**:
  ```bash
  # Private VLAN環境では「Primary VLAN」に対してDHCP Snoopingを有効化します
  Switch(config)# ip dhcp snooping vlan 100
  Switch(config)# interface GigabitEthernet1/0/48
  Switch(config-if)# description === Promiscuous Port to DHCP Server ===
  Switch(config-if)# ip dhcp snooping trust
  ```
* **検証方法**:
  Isolatedポート（VLAN 101所属）配下の端末がDHCPリクエストを送信し、Primary VLAN 100のPromiscuousポート経由でDHCPサーバーからIPアドレスを取得できることを確認します。

---

### シナリオ10: Dynamic ARP Inspection（DAI）および IP Source Guard（IPSG）との完全統合設定

* **問題**: VLAN 10において、不正DHCPサーバー対策、ARPポイズニング対策、およびIP/MACスプーフィング対策を組み合わせた統合L2セキュリティを構築します。
* **要件**:
  - VLAN 10でDHCP Snooping、DAI、IPSGを有効化すること。
  - アクセスポート `GigabitEthernet1/0/1` でIPおよびMACアドレスの検証を行うこと。
  - アップリンクポート `GigabitEthernet1/0/48` を全機能のTrustedポートとすること。
* **設定例**:
  ```bash
  # 1. DHCP Snooping 設定
  Switch(config)# ip dhcp snooping
  Switch(config)# ip dhcp snooping vlan 10
  Switch(config)# interface GigabitEthernet1/0/48
  Switch(config-if)# ip dhcp snooping trust

  # 2. Dynamic ARP Inspection (DAI) 設定
  Switch(config)# ip arp inspection vlan 10
  Switch(config-if)# ip arp inspection trust

  # 3. IP Source Guard (IPSG) 設定 (アクセスポート)
  Switch(config)# interface GigabitEthernet1/0/1
  Switch(config-if)# ip verify source binding ip-mac
  ```
* **検証方法**:
  `show ip dhcp snooping binding` でIPアドレスが動的取得された後、`show ip verify source` でポート `Gi1/0/1` にPACLフィルターが適用されていること、および `show ip arp inspection interfaces` で `Gi1/0/48` がTrustされていることを確認します。


## ❓ 想定試験問題

### 質問1 (コンフィグ読解・トラブルシュート)
**問題**: 以下のスイッチ（SW1）およびルーター（R1）のコンフィグが設定されています。SW1配下のアクセスポート（Gi1/0/1）に接続されたクライアントPCがDHCP DISCOVERを送信しても、IPアドレスを取得できません。原因として最も適切なものを1つ選び、修正コマンドを答えてください。

**SW1 Configuration**:
```text
ip dhcp snooping
ip dhcp snooping vlan 10
interface GigabitEthernet1/0/48
 switchport mode trunk
 ip dhcp snooping trust
```

**R1 Configuration**:
```text
interface GigabitEthernet0/0/0
 ip address 10.1.1.1 255.255.255.0
 ip dhcp pool LAN_POOL
  network 10.1.1.0 255.255.255.0
  default-router 10.1.1.1
```

* **選択肢**:
  A. SW1のGi1/0/48で `ip dhcp snooping trust` が設定されているため。
  B. SW1がデフォルトでOption 82を挿入し、R1（`giaddr=0.0.0.0`）がこれを破棄しているため。
  C. SW1のGi1/0/1で `ip dhcp snooping limit rate` が未設定のため。
  D. R1で `ip dhcp snooping` が有効化されていないため。

* **正解**: **B**
* **解説**: SW1はデフォルトで `ip dhcp snooping information option`（Option 82挿入）が有効化されています。L2スイッチから送られたDHCP DISCOVERには `giaddr = 0.0.0.0` かつ Option 82が含まれます。Cisco IOS DHCPサーバー（R1）はデフォルトでこのパケットを破棄します。
* **修正方法**: SW1で `no ip dhcp snooping information option` を設定するか、R1で `ip dhcp relay information trust-all` を設定します。

---

### 質問2 (トラブルシュート)
**問題**: スイッチ再起動後、VLAN 10配下の全端末で通信が不通となりました。スイッチにはDHCP SnoopingおよびDynamic ARP Inspection（DAI）が有効化されています。ログを確認すると大量の `DHCP_SNOOPING` および `SW_DAI` ドロップが発生していました。この問題の根本原因と適切な永久対策を述べてください。

* **正解・解説**:
  - **根本原因**: DHCP Snooping Binding Databaseが不揮発性ストレージ（FlashやTFTPサーバー）に保存されておらず、スイッチ再起動に伴いメモリ上のバインディングテーブルが完全に消失した。その結果、DAIが正当なクライアントからのARPパケットを「DB未登録」としてすべてドロップした。
  - **永久対策**: 以下のコマンドを設定してバインディングDBをFlashメモリに定期保存・復元するように構成する。
    ```bash
    Switch(config)# ip dhcp snooping database flash:dhcp_snoop_db.bin
    Switch(config)# ip dhcp snooping database write-delay 60
    ```

---

### 質問3 (実装・設計)
**問題**: 以下のネットワーク構成において、アクセススイッチSW1配下の端末がディストリビューションスイッチSW2のDHCPサーバーからIPを取得できるようにするための最小限の設定要件を述べてください。

```text
[ Client PC ] ── (Gi1/0/1) [ SW1 ] ── (Trunk: Gi1/0/24) ── (Trunk: Gi1/0/24) [ SW2 ] ── [ DHCP Server ]
```

* **条件**:
  - SW1、SW2ともにDHCP Snoopingを有効化。
  - SW1はOption 82を挿入する設定とする。
  - SW2のGi1/0/24（SW1接続ポート）は Untrusted ポートのままとすること。

* **正解・解説**:
  - SW1側: SW2接続ポート（Gi1/0/24）を `ip dhcp snooping trust` に設定。
  - SW2側: SW1接続ポート（Gi1/0/24）で、UntrustedポートでありながらOption 82付きパケットの受信を許可するよう `ip dhcp snooping information option allow-untrusted` を設定。さらにDHCPサーバー接続ポートを `ip dhcp snooping trust` に設定。

---

### 質問4 (コンフィグ読解・分析)
**問題**: 以下の `show ip dhcp snooping statistics` 出力を確認し、発生している事象とそれを防ぐために有効な設定コマンドを答えてください。

```text
# show ip dhcp snooping statistics
Packets Forwarded = 1200
Packets Dropped   = 450
  MAC verification failure = 420
  Option 82 on untrusted port = 30
```

* **正解・解説**:
  - **発生している事象**: `MAC verification failure` が420件発生しているため、Ethernetヘッダーの送信元MACアドレスとDHCPパケット内の `chaddr` が異なる不正なメッセージが発生しています。
  - **有効な防護設定**: スイッチ上で `ip dhcp snooping verify mac-address`（デフォルト有効）が正しくパケットを検知・遮断しています。追加対策としてアクセスポートに `ip dhcp snooping limit rate 15` を適用し、不正パケットを送るポートを `err-disable` に落とす設定が推奨されます。

---

### 質問5 (Design & Architecture)
**問題**: セキュアネットワーク環境において、DHCP Snooping、Dynamic ARP Inspection (DAI)、IP Source Guard (IPSG) を導入する際の設計上の依存関係および適切な設定順序を説明してください。

* **正解・解説**:
  - **依存関係**: IPSGおよびDAIは、DHCP Snoopingが作成する「DHCP Snooping Binding Database」に完全に依存します。
  - **適切な設定・有効化順序**:
    1. **第一段階**: DHCP Snooping（グローバル・VLAN・Trustポート・DB保存設定）を有効化し、ネットワーク内の全端末のバインディングテーブルが正常に構築されるまで待機。
    2. **第二段階**: Dynamic ARP Inspection（DAI）を有効化し、ARP検証を開始。
    3. **第三段階**: IP Source Guard（IPSG）をアクセスポートに適用し、IP/MACスプーフィング防止フィルター（PACL）を発動。
  - 順序を誤ってSnooping DBが無い状態でDAI/IPSGを有効化すると、既存通信がすべて遮断される重篤な障害が発生します。

---

## 🔗 参考リソース

Cisco公式の技術情報およびドキュメント一覧：

* Cisco Catalyst Switch Configuration Guide: DHCP Snooping
* Cisco Technical Support Note: Troubleshooting DHCP Snooping
* Cisco Command Reference: ip dhcp snooping commands
* Cisco Design Guide: Secure Campus LAN Deployment Guide

---

## 📝 補足（Notes）

### 1. DHCP Snooping と Option 82 挿入のまとめ図

```text
[ Access Switch (L2) ] ──── Option 82 挿入 ────► [ Distribution / Core Router ]
   (giaddr: 0.0.0.0)                               (giaddr: 0.0.0.0 + Opt82)
          │                                                    │
          │                                                    ▼
   no ip dhcp snooping                                 ip dhcp relay information trust-all
   information option                                  (または ignore information option)
   【スイッチ側で制御】                                【ルーター側で制御】
```

### 2. 試験直前チェックリスト
- [ ] `ip dhcp snooping` と `ip dhcp snooping vlan <X>` の両方を設定したか？
- [ ] DHCPサーバーへの経路上のすべてのトランクポートで `ip dhcp snooping trust` を設定したか？
- [ ] Option 82のドロップ問題に対する対処（`no ip dhcp snooping information option` または `trust-all`）を理解しているか？
- [ ] Snooping Databaseの不揮発性ストレージ保存（`flash:`）を設定したか？
- [ ] DAI / IPSG を導入する前に Snooping DB が正常に生成されているか確認したか？



