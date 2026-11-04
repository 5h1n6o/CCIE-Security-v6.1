---
layout: default
title: 3.6-Monitoring
nav_order: 6
parent: 3.0-Security-Infrastructure
---

# 3.6 Monitoring protocols

## 📘 概要

ネットワーク監視プロトコル（Monitoring Protocols）は、ネットワークインフラストラクチャおよびセキュリティデバイスの稼働状態、リソース使用率、トラフィックフロー、セキュリティイベントをリアルタイムかつ可視化して把握するための包括的な技術群です。CCIE Security v6.1 において本分野は、マネジメントプレーンのセキュリティ保護（Management Plane Protection）、脅威検知・可視化（Cisco Secure Network Analytics / Stealthwatch）、ネットワークフォレンジック、およびセキュリティオペレーションセンター（SOC）へのログ統合を支える極めて重要な基盤技術と位置づけられています。

対象となる主要プロトコルおよび技術は以下の5領域です。

1. **SNMP (Simple Network Management Protocol - SNMPv2c / SNMPv3)**:
   * デバイスのステータス情報（CPU/メモリ使用率、インターフェイス状態、トラフィックカウンタ）のポーリング取得（GET/GETNEXT/GETBULK）、およびイベント発生時の自発的通知（TRAP/INFORM）機能を提供します。
   * CCIE Security では、クリアテキスト通信である SNMPv2c のリスクを排除し、暗号化と改ざん防止機能を備えた **SNMPv3（USM: User-based Security Model / VACM: View-based Access Control Model）** の厳格な構築が要求されます。

2. **Syslog (System Logging Protocol & Secure Syslog)**:
   * オペレーティングシステム、ファイヤーウォール（ASA / FTD）、ルーティングプロトコル、インサージョンプロテクション（IPS）から発生するシステムイベントログを中央 syslog サーバー（SIEM等）へリアルタイム転送します。
   * 標準 UDP 514 通信に加え、TLS による暗号化転送（Secure Syslog - TCP 6514）や、FTD / ASA における高信頼ログ転送（NSEL: NetFlow Secure Event Logging）の設計と実装が含まれます。

3. **NetFlow / IPFIX / Flexible NetFlow (FNF)**:
   * パケットヘッダー情報（IP 5-Tuple 等）からフローを生成し、ネットワーク内を通過するすべてのトラフィック統計および挙動データを可視化します。
   * Flexible NetFlow (FNF) は「Flow Record」「Flow Exporter」「Flow Monitor」「Sampler」のコンポーネントに分離されており、Cisco Stealthwatch (SMC / Flow Collector) や SIEM へのフローデータ供給源として必須の技術です。

4. **ポートミラーリング（SPAN / RSPAN / ERSPAN）**:
   * パケットレベルでの詳細解析や IDS/IPS（FTD/Snort）、WSA などの帯域外（Out-of-band）セキュリティセンサーへトラフィックのコピーを転送します。
   * ローカルスイッチ内の SPAN、VLANを跨ぐ RSPAN、ならびに IP/GRE ネットワークをカプセル化越えする **ERSPAN (Encapsulated Remote SPAN)** の設計・トラブルシューティングが重点対象です。

5. **Model-Driven Telemetry (MDT)**:
   * 従来の SNMP ポーリングによるスケール限界と遅延を克服するため、YANG モデルに基づいたストリーミングテレメトリ（Dial-in / Dial-out, gRPC, NETCONF）により、リアルタイムかつ低負荷で大量の運用データを外部コレクタに配信します。


---

## 🔑 要点

主要な監視プロトコルの基本特性、用途、メリット・デメリット、対応機種、制限事項および設計上の注意点を下表にまとめます。

| 項目 | SNMPv3 | Syslog / Secure Syslog | Flexible NetFlow (FNF) / IPFIX | ERSPAN | Model-Driven Telemetry |
| --- | --- | --- | --- | --- | --- |
| **主要用途** | 機器ステータス監視、MIB情報のポーリングおよびアラート通知 (Traps/Informs) | システムイベントログ、設定変更、セキュリティポリシー違反の集中記録 | 通信フロー統計の収集、異常トラフィック検知、Stealthwatch連携 | L3ネットワーク越しのトラフィックキャプチャ、IDS/IPS用パケット複製 | 高頻度・リアルタイムデータストリーミング、モダンな自動化運用 |
| **セキュリティ機構** | HMAC-SHA/MD5 認証、AES/DES 暗号化 (USM), 視点制御 (VACM) | Secure Syslog (TLS/TCP 6514) による相互認証と暗号化 | 転送プロトコルに UDP/IPFIX/SCTP 使用、IPsec トンネル等で保護可能 | 独自 GRE ヘッダー (Type 0x88be) でカプセル化、IPsec による保護が可能 | TLS 暗号化 (gRPC / NETCONF)、Mutual TLS 認証対応 |
| **メリット** | 標準プロトコル、細かな MIB オブジェクト制御、VACM によるアクセス制限 | テキストベースで解読が容易、標準的 SIEM 連携、低リソース負荷 | パケット全文を保存せずフロー単位で集約するため効率的、メタデータ可視化 | L3 境界を越えてリモートの解析サーバーへパケットコピーを転送可能 | プッシュ型配信により低遅延・高スケール、YANG 構造化データ出力 |
| **デメリット** | MIB 構造の複雑性、SNMPv2c は認証暗号化なし、ポーリング遅延 | UDP 送信時の消失リスク（Secure TLS 未使用時）、ログ量の膨大化 | ハードウェア（ASIC）メモリ消費、CPU 負荷（Sampler 未使用時） | カプセル化によるオーバーヘッド (GRE 50Byte)、MTU 調整が必須 | デバイスの CPU/メモリリソース消費、レガシー OS 非対応 |
| **対応機種** | IOS-XE, ASA, FTD, FMC, Switch, Router | IOS-XE, ASA, FTD, FMC, WSA, ESA, ISE | IOS-XE, ASA (NSEL), FTD (NSEL/NetFlow) | IOS-XE (Router/Switch), FTD (一部プラットフォーム) | IOS-XE (16.x以降), NX-OS |
| **制限事項** | SNMP Trap は UDP 送達保証なし (Inform は ACK あり) | UDP ログは攻撃者による改ざん・盗聴のリスク | サンプリングなしの全フロー記録は高負荷を招く | 途中のルータで GRE パケットがドロップされると疎通不可 | YANG モデルの定義に依存 |
| **設計上の注意点** | EngineID の重複回避、`authPriv` の標準化、CoPP での UDP 161/162 制限 | Log Buffer 溢れ防止、Syslog サーバーの冗長化、正確な NTP 同期 | Exporter の送信元 Loopback 指定、テンプレート送信間隔の最適化 | 源/宛先 IP のルーティング確保、PMTU 考慮、Reflector Port 設計 | Dial-out 送信先 IP の疎通確認、送信周期（Periodical/OnChange）設定 |


---

## 🏗 動作原理

各監視プロトコルのアーキテクチャおよびパケットフローの概要を解説します。

### 1. SNMPv3 (USM & VACM) 動作モデル

SNMPv3 は、従来のコミュニティ文字列に頼る不安全な方式を廃止し、**USM (User-based Security Model)** による認証・暗号化と、**VACM (View-based Access Control Model)** による MIB ビュー制御を統合しています。

```
[ NMS (SNMP Manager) ]
       │
       │ ① EngineID Discovery / Time Synchronization
       │ ② SNMPv3 Get-Request (Authenticated & Encrypted: SHA/AES)
       ▼
[ Security Device (SNMP Agent) ]
       │
       ├─► [ USM Check ]: EngineID + User + HMAC-SHA Hash + AES Decrypt
       │      │ (Pass)
       │      ▼
       ├─► [ VACM Check ]: Group ──> Access List ──> View (Permit/Deny MIB OID)
       │      │ (Pass)
       │      ▼
       └─► [ Execute MIB Read/Write ] ──► Response (Auth + Priv) ──► NMS
```

### 2. Flexible NetFlow (FNF) アーキテクチャ

Flexible NetFlow は、モジュール構造を採用することで、任意のフィールドを抽出・集約し、指定された宛先へ送信します。

```
[ Ingress Packets ]
       │
       ▼
[ Flow Monitor ] ◄──── Applies ──── [ Flow Record ]
       │                              (Match: Src/Dst IP, Port, Proto)
       │                              (Collect: Counters, Timestamps)
       ▼
[ Flow Cache (ASIC / Memory) ]
       │
       ▼ (Active / Inactive Timeout)
[ Flow Exporter ]
       │ (Encapsulation: NetFlow v9 / IPFIX)
       ▼
[ NetFlow Collector / Cisco Stealthwatch (SMC / Flow Collector) ]
```

### 3. ERSPAN (Encapsulated Remote SPAN) パケットカプセル化フロー

ERSPAN は、送信元（Source Session）でコピーしたパケットに ERSPAN GRE ヘッダーおよび外側 IP ヘッダーを付与し、ルーテッドネットワーク（L3）を経由して受信側（Destination Session）またはコレクタに送出します。

```
[ Source Port (Traffic) ] ──► [ Copy Packet ]
                                   │
                                   ▼
                   [ Add ERSPAN Header (Type 0x88be) ]
                   [ Add Outer IP Header (Src: AP, Dst: Collector) ]
                                   │
                                   ▼ (Routed L3 IP Network)
                                   │
                                   ▼
                   [ Destination / Wireshark / IDS Sensor ]
                   [ Decapsulate GRE & Analyze Raw Frame ]
```


---

## ⚙ 動作シーケンス

各プロトコルの詳細なパケット処理ステップおよび制御シーケンスを解説します。

### 1. SNMPv3 認証・暗号化および VACM 処理シーケンス

```
1. Discovery パケット交換 (EngineID と Timers の同期):
   - NMS は未知の EngineID を取得するため、空の User/EngineID で Get-Request を送信。
   - Agent は自身の Local EngineID および EngineBoots/EngineTime を含めた Report パケットを返答。

2. USM (User-based Security Model) 処理:
   - NMS は認証鍵 (Auth Key - SHA/MD5) と暗号鍵 (Priv Key - AES/DES) を Local EngineID を用いて派生。
   - パケットヘッダーに MsgUserName, MsgAuthenticationParameters (HMACハッシュ), MsgPrivacyParameters (IV) を付与して送信。
   - Agent 側で HMAC ハッシュを照合し（改ざん・なりすましチェック）、AES でペイロードを復号。

3. VACM (View-based Access Control Model) 処理:
   - User が所属する Group を特定。
   - Group に割り当てられた Security Level (noAuthNoPriv / authNoPriv / authPriv) を確認。
   - Group にバインドされた Read/Write View を照合。
   - 要求された MIB オブジェクト（OID）が指定 View の permit 範囲に含まれる場合のみ処理を実行し、結果を応答。
```

### 2. Flexible NetFlow (FNF) エージングおよびパケット送出シーケンス

```
1. フローの検出とキャッシュ登録:
   - パケット受信用に設定された Flow Monitor がパケットヘッダーの Match キー（5-Tuple 等）を照合。
   - 既存フローテーブルに一致すればパケット数/バイト数カウンタを更新。新規フローであればキャッシュにエントリーを作成。

2. キャッシュのエージング（Timeout）判定:
   - Active Timeout (デフォルト 1800 秒 / 30分): 長時間継続する TCP フローの統計を定期的に吐き出すためにタイムアウト。
   - Inactive Timeout (デフォルト 15 秒): 一定時間パケットが途切れたフローを削除対象として判定。

3. エクスポート処理 (Flow Exporter):
   - タイムアウトしたフローをキャシュから抽出し、NetFlow v9 または IPFIX フォーマットのパケットに整形。
   - テンプレートフローセット (Template FlowSet) を定期的に送信し、コレクタ側にレコード構造（データ型定義）を通知。
   - 設定された Exporter の送信元（Loopback 等）から UDP/IPFIX パケットを出力。
```


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security ラボ試験において、監視プロトコルは単独での設定問題だけでなく、**「マネジメントプレーンの厳格なハードニング」「Stealthwatch 連携用の正確なフロー送出」「ネットワーク異常トラブルシューティング」**の文脈で高頻度に出題されます。

### Blueprintにおける超重要ポイント

1. **SNMPv3 厳格構築 (EngineID, USM, VACM, Traps)**:
   * 単に `snmp-server community` を設定すると減点対象となります。必ず **SNMPv3 `authPriv` (SHA + AES128)** で構築を指示されるケースが多いため、グループ名、ユーザー名、ビュー名、アクセスリスト（ACL）の紐付け順序を正確に把握する必要があります。
   * トラップ送信側で `snmp-server host <IP> version 3 priv <User>` を設定する際、事前に Remote EngineID または Local User の関連付けが必要です。

2. **Secure Syslog (TLS) & Logging Source Interface**:
   * Syslog の送信元 IP アドレスを強制指定する `logging source-interface Loopback0` は基本要件です。
   * ASA や FTD では、特定ファイヤーウォールイベント（Denied traffic 等）のみを高速で送出する NSEL (NetFlow Secure Event Logging) と、標準 Syslog の使い分けが問われます。

3. **Flexible NetFlow (FNF) コンポーネントの完全構築**:
   * 「特定VLANの入出力トラフィックの5-TupleとDSCP値を記録し、Stealthwatch (SMC) に 60 秒ごとにアクティブエクスポートせよ」といった要件が出題されます。
   * `record`（Match/Collect項目の定義）、`exporter`（宛先IP、送信元Port/Interface、Template Interval）、`monitor`（RecordとExporterのバインド）、および `interface`（`ip flow monitor <NAME> input/output`）の4層構造を順番通りに正しく適用する必要があります。

4. **ERSPAN Source/Destination & GRE トンネル透過**:
   * ERSPAN の設定では、`erspan-id`（1〜1023）、`origin ip`（送信元IP）、`destination ip`（受信側IP）の一致が必須です。
   * ルーティングテーブル上で Destination IP への到達性がない場合や、中間ファイヤーウォールで GRE (IP Protocol 47) がドロップされている場合のトラブルシューティングが出題されます。

---

### よくある設定ミス＆試験での落とし穴

* ❌ **SNMPv3 で `authPriv` を指定しているのに、グループ設定で `noauth` / `auth` になっている**:
  * `snmp-server group <GROUP> v3 priv` と設定していない場合、ユーザー作成時に `priv` を指定しても通信が拒否されます。
* ❌ **FNF で `ip flow monitor` をインターフェイスに適用し忘れる**:
  * Record, Exporter, Monitor を完璧に設定しても、インターフェイス（またはSampler）に適用しなければキャッシュは作成されません。
* ❌ **FNF テンプレート送出間隔（Template Option）の設定漏れ**:
  * Stealthwatch 等のコレクタは、テンプレート（`option default-templates` / `template data-template`）を受信するまでフローを正しくパースできません。
* ❌ **ERSPAN で Source インターフェイスに Reflector Port 設定を誤適用する**:
  * Destination Session 側で必要となる `destination interface` や `reflector-port` の設定位置を混同するケース。
* ❌ **ASA / FTD で Syslog レベルの過剰設定による CPU 高負荷**:
  * ASA で `logging level debugging` を有効化すると、大量の接続ログにより CPU が枯渇します。`logging message 106023 level ...` のように特定の Message ID のみをカスタマイズする技法が問われます。


---

## 🛠 設定方法

Cisco IOS-XE, Cisco ASA, および Cisco FTD (FMC / FlexConfig) における完全な設定例を示します。

### 1. Cisco IOS-XE (Router / Switch) 設定例

#### (A) SNMPv3 (USM / VACM / Traps)
```bash
! 1. MIB View の定義 (cisco, system, interfaces 関連のみ許可)
snmp-server view V1-SECURED cisco included
snmp-server view V1-SECURED system included
snmp-server view V1-SECURED interfaces included

! 2. アクセスリストによる NMS IP の制限
ip access-list standard ACL-NMS-ONLY
 permit 10.1.100.50
 permit 10.1.100.51

! 3. SNMPv3 グループの作成 (authPriv + VACM View + ACL)
snmp-server group G-SECURE-ADMIN v3 priv read V1-SECURED write V1-SECURED access ACL-NMS-ONLY

! 4. SNMPv3 ユーザーの作成 (SHA 認証 + AES128 暗号化)
snmp-server user U-ADMIN G-SECURE-ADMIN v3 auth sha AuthPass123! priv aes 128 PrivPass123!

! 5. トラップ送出設定
snmp-server host 10.1.100.50 version 3 priv U-ADMIN cpu memory config
snmp-server enable traps cpu threshold
snmp-server source-interface informs Loopback0
```

#### (B) Secure Syslog (TLS)
```bash
! 送信元インターフェイスとログフォーマットの設定
logging source-interface Loopback0
logging origin-id hostname
logging facility local7
logging trap informational

! Secure Syslog (TLS) 設定 (要 PKI Trustpoint)
logging host 10.1.100.200 transport tcp port 6514 tls trustpoint TP-SYS-CA
```

#### (C) Flexible NetFlow (FNF)
```bash
! 1. Flow Record の定義
flow record REC-STEALTHWATCH
 match ipv4 protocol
 match ipv4 source address
 match ipv4 destination address
 match transport source-port
 match transport destination-port
 match ip tos
 collect counter bytes long
 collect counter packets long
 collect timestamp sys-uptime first
 collect timestamp sys-uptime last

! 2. Flow Exporter の定義
flow exporter EXP-SMC-COLLECTOR
 destination 10.1.200.50
 source Loopback0
 transport udp 2055
 template data-template timeout 60

! 3. Flow Monitor の定義
flow monitor MON-STEALTHWATCH
 record REC-STEALTHWATCH
 exporter EXP-SMC-COLLECTOR
 cache timeout active 60
 cache timeout inactive 15

! 4. インターフェイスへの適用
interface GigabitEthernet0/0/1
 ip flow monitor MON-STEALTHWATCH input
 ip flow monitor MON-STEALTHWATCH output
```

#### (D) ERSPAN (Source Session)
```bash
! 送信元スイッチ (ERSPAN Source)
monitor session 1 type erspan-source
 source interface GigabitEthernet0/0/2 both
 destination
  erspan-id 100
  ip precision 6
  origin ip 10.1.1.1
  ip address 10.1.200.99
```

---

### 2. Cisco ASA 設定例

#### Syslog & NSEL (NetFlow) 設定
```bash
! 1. Syslog 基本設定
logging enable
logging timestamp
logging buffer-size 1048576
logging buffered debugging
logging trap informational
logging host inside 10.1.100.200 format syslog

! 2. 特定の不要 Syslog メッセージの無効化（CPU保護）
no logging message 106015
logging message 106023 level critical

! 3. NSEL (NetFlow) 設定 (MPF 経由)
flow-export destination inside 10.1.200.50 2055
flow-export template timeout-rate 1

policy-map global_policy
 class class-default
  flow-export event-type all destination 10.1.200.50
```

---

### 3. Cisco FTD / FMC 設定例

FMC GUI を用いたモニタリングプロトコルの統一管理手順：

1. **Syslog 設定**:
   * FMC GUI: **Devices** -> **Platform Settings** -> **Syslog** に移動。
   * **Logging Setup**: `Enable Logging` をチェック。
   * **Syslog Servers**: 送信元インターフェイス（Management または Data Interface）、IP アドレス (`10.1.100.200`)、Protocol (`TLS` または `UDP`)、Port (`6514` または `514`) を指定。

2. **NetFlow / NSEL 設定**:
   * FMC GUI: **Devices** -> **Platform Settings** -> **NetFlow**。
   * **Add NetFlow Collector**: Collector IP (`10.1.200.50`) と Port (`2055`) を設定。
   * **FlexConfig** を用いて FTD の LINA エンジン上に NSEL コマンドをプッシュ可能。


## 🔍 検証コマンド

各プロトコルの稼働状態、統計情報、デバッグ情報を確認するためのコマンド一覧を下表にまとめます。

| 目的 | コマンド |
| --- | --- |
| **SNMP エージェント状態の確認** | <code>show snmp</code> |
| **SNMPv3 グループ設定の確認** | <code>show snmp group</code> |
| **SNMPv3 ユーザー情報と暗号化レベル確認** | <code>show snmp user</code> |
| **SNMPv3 MIB View 割当確認** | <code>show snmp view</code> |
| **SNMP セッション・トラップ送信統計** | <code>show snmp stats</code> |
| **Syslog 設定および送信統計の確認** | <code>show logging</code> |
| **Flexible NetFlow モニタ状態の確認** | <code>show flow monitor [NAME]</code> |
| **FNF キャッシュ内のアクティブフロー確認** | <code>show flow monitor [NAME] cache format table</code> |
| **FNF Exporter 送信統計およびドロップ確認** | <code>show flow exporter [NAME] statistics</code> |
| **SPAN / RSPAN / ERSPAN セッション確認** | <code>show monitor session [all | N]</code> |
| **ERSPAN カプセル化詳細の確認** | <code>show monitor session N detail</code> |
| **Telemetry 配信状態の確認 (MDT)** | <code>show telemetry connection all</code> / <code>show telemetry ietf subscription all</code> |
| **ASA Syslog バッファおよび送信状態確認** | <code>show logging</code> |
| **ASA NetFlow (NSEL) 送信統計** | <code>show flow-export counters</code> |
| **SNMP パケットレベルのデバッグ** | <code>debug snmp packets</code> |
| **SNMPv3 USM 認証・暗号化処理のデバッグ** | <code>debug snmp header</code> |
| **Flexible NetFlow 制御処理のデバッグ** | <code>debug flow monitor</code> / <code>debug flow exporter</code> |
| **Syslog 配信処理のデバッグ** | <code>debug logging</code> |


---

## 🚨 トラブルシュート

トラブル発生時の症状、根本原因、確認コマンド、および対処方法を下表に整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **SNMPv3 NMS からの GET 応答がタイムアウトする** | 1. SNMPv3 ユーザーのパスフレーズ不一致<br>2. ACL による NMS IP のブロック<br>3. VACM View に対象 MIB が含まれていない<br>4. Group と User の Security Level 不一致 | 1. <code>show snmp user</code><br>2. <code>show ip access-lists</code><br>3. <code>show snmp view</code><br>4. <code>debug snmp header</code> | 1. `snmp-server user` のパスフレーズ再設定<br>2. ACL で NMS IP を `permit`<br>3. View に `included` で OID を追加<br>4. Group の `priv`/`auth` レベルを合わせる |
| **Secure Syslog (TLS) がサーバーに届かない** | 1. PKI 証明書の検証エラー（CA不一致/有効期限切れ）<br>2. ポート 6514 への到達不可<br>3. TCP ハンドシェイク失敗 | 1. <code>show crypto pki trustpoints</code><br>2. <code>show logging</code><br>3. <code>debug logging</code> | 1. CA 証明書を正しく再インポート<br>2. 中間 FW で TCP 6514 許可<br>3. 送信元 IP (Loopback) のルーティング確認 |
| **NetFlow コレクタ（Stealthwatch等）でグラフが表示されない** | 1. Template が送出されていない<br>2. Exporter の宛先 IP/Port 誤り<br>3. インターフェイスに `ip flow monitor` 未適用 | 1. <code>show flow exporter statistics</code><br>2. <code>show flow monitor cache</code> | 1. `template data-template timeout 60` を設定<br>2. Collector IP/Port(2055) 確認<br>3. 対象 IF で `ip flow monitor <NAME> input/output` 適用 |
| **ERSPAN 宛先でキャプチャパケットが受信できない** | 1. Origin IP と Dst IP 間で L3 ルーティング不能<br>2. 中間ルータで GRE (Protocol 47) が遮断されている<br>3. MTU 超過による断片化ドロップ | 1. <code>ping <DST_IP> source Loopback0</code><br>2. <code>show monitor session detail</code> | 1. L3 ルーティングの確保<br>2. IPsec / ACL で GRE (Proto 47) を許可<br>3. 送信元で IP MTU / PMTU を適切に設定 |
| **ASA / FTD の CPU 使用率が 100% に高騰する** | Logging Buffer または Console への高頻度ログ（Level 7 Debugging）出力 | <code>show processes cpu-usage non-zero</code><br><code>show logging</code> | `no logging console` / `no logging monitor` を実行。特定メッセージのみ `logging message ID level` で制限 |


---

## ⚠ 制限事項

1. **SNMP の構造的制限**:
   * SNMP Traps は UDP 通信であるため送達確認がありません（信頼性が必要な場合は Inform を使用）。
   * 大量の MIB オブジェクトを短時間で Get-Bulk 要求すると、CPU 使用率が一律スパイクするリスクがあります。
2. **NetFlow / FNF のリソース制限**:
   * 高トラフィック（10Gbps+）環境で Sampler（`sampler`）を使用しない全パケットインスペクションを行うと、フローキャッシュメモリが圧迫され、新しいフローがドロップされます。
3. **ERSPAN の帯域とカプセル化制限**:
   * ERSPAN は元のパケットに GRE/IP ヘッダー（約50バイト）を付与するため、1500バイトのフルサイズパケットは断片化（Fragmentation）されるかドロップされます。パケットサイズの増幅（MTU 1550バイト以上）に対応したネットワーク設計が必要です。
4. **FTD / FMC の設定反映**:
   * FTD のモニタリング設定変更は、FMC での「Deploy」操作を実行しない限り有効化されません。


---

## 🔄 他技術との関連

* **CoPP (Control Plane Policing)**:
  * SNMP (UDP 161/162) や Syslog (UDP 514 / TCP 6514) トラフィックが Control Plane を圧迫しないよう、CoPP にてレートリミットを適用します。
* **iACL / Management Plane Protection (MPP)**:
  * 指定されたマネジメントインターフェイス（例: GigabitEthernet0/0）以外からの SNMP/Syslog アクセスを拒否します。
* **Cisco Stealthwatch (Secure Network Analytics)**:
  * Flexible NetFlow (FNF) および ASA/FTD NSEL データを収集し、機械学習（Behavioral Modeling）によってアノマリー（異常な横展開やデータ漏洩）を検知します。
* **NTP (Network Time Protocol)**:
  * Syslog ログスタンプ、NetFlow タイムスタンプ、および SNMPv3 Time Sync（EngineTime）の整合性を保つため、ミリ秒単位で同期された NTP サーバーの存在が絶対前提となります。


---

## 🧩 比較表

### 1. SNMPv2c vs SNMPv3

| 比較項目 | SNMPv2c | SNMPv3 |
| --- | --- | --- |
| **セキュリティモデル** | コミュニティ文字列（Clear Text） | USM (User-based Security Model) |
| **認証機構** | なし（文字列一致のみ） | HMAC-SHA / HMAC-MD5 / HMAC-SHA-256 |
| **暗号化** | なし | AES-128 / AES-192 / AES-256 / DES |
| **アクセス制御** | 標準/拡張 ACL | VACM (View-based Access Control Model) |
| **信頼性** | 不安全（盗聴・改ざん・成りすましが容易） | 安全（改ざん防止・リプレイ防止・暗号化） |

### 2. SPAN vs RSPAN vs ERSPAN

| 比較項目 | SPAN (Local) | RSPAN (Remote) | ERSPAN (Encapsulated Remote) |
| --- | --- | --- | --- |
| **スコープ** | 単一スイッチ内 | 同一 L2 ドメイン（VLAN 経由） | 任意の L3 ルーテッド IP ネットワーク |
| **カプセル化** | なし（直接ポートへコピー） | 特殊な RSPAN VLAN タグを付与 | GRE カプセル化 (IP Protocol 47 / Port 0x88be) |
| **ルーティング** | 不可 | 不可 (L2 アダプティブ) | 可能（IP ルーティング可能な任意の宛先へ送出） |
| **主な用途** | ローカル Wireshark / 局所 IDS | 同一拠点の別スイッチ上の解析器 | データセンター中央の IDS / パケットキャプチャ農場 |


---

## 💡 ベストプラクティス

1. **SNMPv3 の完全標準化**:
   * すべての機器で `noAuthNoPriv` / `authNoPriv` を禁止し、`authPriv` (SHA + AES128 以上) を適用すること。
   * MIB View を VACM で制限し、不要なシステム内部 MIB や機密情報 MIB（例: Cisco CLI コマンド履歴 MIB）へのアクセスをブロックすること。

2. **Syslog の安全な転送と時刻同期**:
   * 通信用プロトコルには UDP 514 ではなく Secure Syslog (TLS/TCP 6514) を採用し、中間経路での改ざん・盗聴を防ぐこと。
   * 全デバイスで `service timestamps log datetime msec show-timezone` を有効化し、NTP サーバーと同期すること。

3. **Flexible NetFlow (FNF) のサンプリング設計**:
   * 高トラフィックバックボーンでは 1:1000 などの Sampled NetFlow (`sampler`) を適用し、スイッチの CPU / ASIC メモリ枯渇を回避すること。
   * テンプレート送信間隔（`template data-template timeout`）は 60 秒以下に設定し、コレクタの再起動後速やかにフロー解析が再開できるようにすること。


## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験レベルの実践的演習シナリオを10個作成しました。

---

### シナリオ1: IOS-XE における SNMPv3 (authPriv) と VACM の構築

#### 問題
Router R1 において、NMS サーバー (10.1.100.50) からの MIB 取得用に SNMPv3 を構築してください。

#### 要件
1. グループ名: `GRP-MONITOR`
2. 認証方式: SHA (`AuthKey123!`)
3. 暗号化方式: AES 128 (`PrivKey123!`)
4. ユーザー名: `USR-ADMIN`
5. MIB View: `VIEW-LIMITED` を作成し、`system` (1.3.6.1.2.1.1) および `interfaces` (1.3.6.1.2.1.2) MIB のみを許可すること。
6. NMS からのアクセスは ACL `ACL-NMS` (10.1.100.50) で制限すること。

#### 設定例
```bash
! ACL の定義
ip access-list standard ACL-NMS
 permit 10.1.100.50

! MIB View の定義
snmp-server view VIEW-LIMITED system included
snmp-server view VIEW-LIMITED interfaces included

! グループの定義 (authPriv + VACM View + ACL)
snmp-server group GRP-MONITOR v3 priv read VIEW-LIMITED write VIEW-LIMITED access ACL-NMS

! ユーザーの定義
snmp-server user USR-ADMIN GRP-MONITOR v3 auth sha AuthKey123! priv aes 128 PrivKey123!
```

#### 検証方法
```bash
R1# show snmp group
R1# show snmp user
! NMS (10.1.100.50) から snmpwalk 実行:
! snmpwalk -v3 -u USR-ADMIN -l authPriv -a SHA -A AuthKey123! -x AES -X PrivKey123! 10.1.1.1 system
```

---

### シナリオ2: SNMPv3 トラップおよび通知の送信設定

#### 問題
R1 上で CPU 使用率超過およびインターフェイスの状態変化アラートを SNMPv3 トラップとして NMS (10.1.100.50) へ送信してください。

#### 要件
1. トラップ送信用ユーザー: `USR-ADMIN` (SHA + AES128)
2. 送信元 IP アドレスは `Loopback0` の IP アドレスを使用すること。
3. トラップ対象: CPU 超過、インターフェイス linkDown/linkUp。

#### 設定例
```bash
snmp-server enable traps cpu threshold
snmp-server enable traps snmp linkdown linkup
snmp-server host 10.1.100.50 version 3 priv USR-ADMIN
snmp-server source-interface informs Loopback0
```

#### 検証方法
```bash
R1# show snmp host
R1# show snmp stats
! インターフェイスの shutdown / no shutdown を実行し NMS 側で Trap 受信ログを確認
```

---

### シナリオ3: Secure Syslog (TLS 6514) の構築

#### 問題
R1 から Central Syslog Server (`10.1.100.200`) へ、TLS による暗号化セキュア Syslog を送信してください。

#### 要件
1. ログレベル: `informational` (Level 6) 以上。
2. ポート: TCP `6514`。
3. 送信元インターフェイス: `Loopback0`。
4. PKI Trustpoint: `TP-SYS-CA` を使用すること。

#### 設定例
```bash
logging source-interface Loopback0
logging origin-id hostname
logging facility local7
logging trap informational
logging host 10.1.100.200 transport tcp port 6514 tls trustpoint TP-SYS-CA
```

#### 検証方法
```bash
R1# show logging
! "Logging to 10.1.100.200 port 6514 via TLS" のステータスが ACTIVE であることを確認
```

---

### シナリオ4: Flexible NetFlow (FNF) による 5-Tuple 統計収集と Stealthwatch 送出

#### 問題
R1 の GigabitEthernet0/0/1 を通過する IPv4 トラフィックに対して Flexible NetFlow を設定し、Stealthwatch Collector (`10.1.200.50:2055`) へ送出してください。

#### 要件
1. Flow Record 名: `REC-FNF-5TUPLE`（Match: Src/Dst IP, Src/Dst Port, Protocol / Collect: Packets, Bytes, Timestamps）
2. Flow Exporter 名: `EXP-STEALTHWATCH`（Destination: `10.1.200.50`, UDP `2055`, Source: `Loopback0`, Template interval: `60` 秒）
3. Flow Monitor 名: `MON-STEALTHWATCH`（Active Timeout: `60` 秒, Inactive Timeout: `15` 秒）
4. インターフェイス Gi0/0/1 の Ingress / Egress の両方に適用すること。

#### 設定例
```bash
! 1. Flow Record
flow record REC-FNF-5TUPLE
 match ipv4 protocol
 match ipv4 source address
 match ipv4 destination address
 match transport source-port
 match transport destination-port
 collect counter bytes long
 collect counter packets long
 collect timestamp sys-uptime first
 collect timestamp sys-uptime last

! 2. Flow Exporter
flow exporter EXP-STEALTHWATCH
 destination 10.1.200.50
 source Loopback0
 transport udp 2055
 template data-template timeout 60

! 3. Flow Monitor
flow monitor MON-STEALTHWATCH
 record REC-FNF-5TUPLE
 exporter EXP-STEALTHWATCH
 cache timeout active 60
 cache timeout inactive 15

! 4. インターフェイス適用
interface GigabitEthernet0/0/1
 ip flow monitor MON-STEALTHWATCH input
 ip flow monitor MON-STEALTHWATCH output
```

#### 検証方法
```bash
R1# show flow monitor MON-STEALTHWATCH cache format table
R1# show flow exporter EXP-STEALTHWATCH statistics
```

---

### シナリオ5: ERSPAN (Encapsulated Remote SPAN) による L3 ネットワーク越えのトラフィック監視

#### 問題
Switch SW1 の GigabitEthernet0/1 を通過する双方向トラフィックをコピーし、L3 ネットワーク経由でデータセンターの Wireshark キャプチャサーバー (`10.1.200.99`) へ送出してください。

#### 要件
1. ERSPAN Session ID: `200`
2. Origin IP: `10.1.1.1` (SW1 Loopback0)
3. Destination IP: `10.1.200.99`
4. IP DSCP: `46` (EF)

#### 設定例
```bash
monitor session 200 type erspan-source
 source interface GigabitEthernet0/1 both
 destination
  erspan-id 200
  ip precision 6
  origin ip 10.1.1.1
  ip address 10.1.200.99
  ip dscp 46
```

#### 検証方法
```bash
SW1# show monitor session 200
SW1# show monitor session 200 detail
! 宛先 10.1.200.99 上で GRE (Protocol 47, Port 0x88be) のパケットが届いているか確認
```

---

### シナリオ6: Cisco ASA における NSEL (NetFlow Secure Event Logging) の設定

#### 問題
ASA1 において、すべてのコネクション生成・終了イベントを NSEL コレクタ (`10.1.200.50:2055`) へ送出してください。

#### 要件
1. 送信元インターフェイス: `inside`
2. MPF (Modular Policy Framework) を使用して全トラフィックのフローイベントを出力すること。

#### 設定例
```bash
flow-export destination inside 10.1.200.50 2055
flow-export template timeout-rate 1

policy-map global_policy
 class class-default
  flow-export event-type all destination 10.1.200.50
```

#### 検証方法
```bash
ASA1# show flow-export counters
ASA1# show service-policy flow-export
```

---

### シナリオ7: Cisco ASA での特定の非重要 Syslog の抑制と重要ログのカスタム設定

#### 問題
ASA1 において、高頻度に発生するログ ID `106015` (Deny Inbound UDP) を無効化し、ログ ID `106023` (Deny IP by ACL) のログレベルを `Critical` (Level 2) に変更してください。

#### 要件
1. 不要ログによる Syslog サーバーおよび CPU の高負荷を防御すること。

#### 設定例
```bash
no logging message 106015
logging message 106023 level critical
```

#### 検証方法
```bash
ASA1# show logging | include 106023
```

---

### シナリオ8: FTD (FMC) における Syslog サーバー冗長化の設定

#### 問題
FMC で管理されている FTD デバイスから、2台の Syslog サーバー (`10.1.100.200` および `10.1.100.201`) へ重み付けログ出力を設定してください。

#### 要件
1. FMC の Platform Settings を使用すること。
2. Facility: `LOCAL7`, Level: `Informational`.

#### 設定例（FMC CLI / FlexConfig 概念）
FMC GUI 操作手順:
1. **Devices** -> **Platform Settings** -> Policy 編集。
2. **Syslog** -> **Syslog Servers** に移動。
3. Server 1: `10.1.100.200`, UDP 514, Interface: `management` を追加。
4. Server 2: `10.1.100.201`, UDP 514, Interface: `management` を追加。
5. **Logging Setup**: `Enable Logging` をチェック。
6. Policy を保存し FTD へ **Deploy**。

#### 検証方法
```bash
FTD-CLI# show running-config logging
FTD-CLI# show logging
```

---

### シナリオ9: Sampled NetFlow (FNF Sampler) による高負荷スイッチでのフロー制限

#### 問題
高トラフィックが流入する Switch SW1 の 10Gbps インターフェイス (Te1/0/1) に対して、100 パケットに 1 パケットのサンプリング率で NetFlow を収集してください。

#### 要件
1. Sampler 名: `SAMP-1IN100` (Random 1 out of 100)
2. Flow Monitor `MON-STEALTHWATCH` に適用すること。

#### 設定例
```bash
! 1. Sampler の作成
sampler SAMP-1IN100
 mode random 1 out-of 100

! 2. インターフェイスへの適用
interface TenGigabitEthernet1/0/1
 ip flow monitor MON-STEALTHWATCH sampler SAMP-1IN100 input
```

#### 検証方法
```bash
SW1# show sampler SAMP-1IN100
SW1# show flow monitor MON-STEALTHWATCH statistics
```

---

### シナリオ10: Model-Driven Telemetry (Dial-out) の構築

#### 問題
Cisco IOS-XE Router R1 において、YANG モデルに基づいたシステムCPU情報を外部コレクタ (`10.1.100.80:57500`) へ 10 秒周期でストリーミング配信（Dial-out）してください。

#### 要件
1. Subscription ID: `101`
2. Transport: `grpc-tcp`
3. Encoding: `encode-kvgpb` (Key-Value Google Protocol Buffers)
4. XPath: `Cisco-IOS-XE-process-cpu-oper:cpu-usage/cpu-utilization`

#### 設定例
```bash
telemetry ietf subscription 101
 encoding encode-kvgpb
 filter xpath /Cisco-IOS-XE-process-cpu-oper:cpu-usage/cpu-utilization
 stream yawg
 update-policy periodic 1000
 receiver ip address 10.1.100.80 57500 protocol grpc-tcp
```

#### 検証方法
```bash
R1# show telemetry ietf subscription 101 summary
R1# show telemetry connection all
```


## ❓ 想定試験問題

CCIE Security ラボ試験・記述問題レベルの想定問題と詳細解説です。

---

### 質問 1 (コンフィグ読解 - SNMPv3 VACM)

**問題**: 以下の R1 のコンフィグを分析し、NMS ユーザー `U-MON` が MIB オブジェクト `1.3.6.1.2.1.2` (interfaces) を書き込み（SET 要求）できるか判断し、その理由を説明してください。

```bash
snmp-server view V-READ system included
snmp-server view V-WRITE interfaces included
snmp-server group G-SEC v3 priv read V-READ write V-WRITE
snmp-server user U-MON G-SEC v3 auth sha AuthPassKey priv aes 128 PrivPassKey
```

**解答と解説**:
* **判定**: できません（SET 要求は拒否されます）。
* **理由**: `snmp-server group` コマンドでグループ `G-SEC` に対して `v3 priv` レベルが指定されていますが、ユーザー `U-MON` は NMS から GET / SET 要求を送信する際に `authPriv` セッションを構築する必要があります。しかし、VACM の Write View において `V-WRITE` がバインドされているものの、インターフェイス MIB の多くのオブジェクト（例: `ifInOctets` 等）は Read-Only であり構造上 SET が不可能です。さらに ACL が未定義の場合、デフォルト動作やセキュリティレベルのチェックにより即座に破棄されます。

---

### 質問 2 (トラブルシューティング - NetFlow テンプレート不完全)

**問題**: 管理者は R1 に Flexible NetFlow を設定し、Cisco Stealthwatch Collector にフローを送信しました。しかし、Stealthwatch 画面上に「Unparsed Flow Records Received」というエラーが表示され、通信データが可視化されません。原因と対処方法を説明してください。

**解答と解説**:
* **原因**: NetFlow v9 および IPFIX は動的フォーマットであり、コレクタ側がデータレコードを解読するためには、データ構造を定義した「テンプレートデータ（Template Record）」を定期的に受信する必要があります。FNF Exporter に `template data-template timeout` が設定されていないか、タイムアウト値が長すぎるため、コレクタがテンプレートを受信できていないことが原因です。
* **対処方法**: Flow Exporter 設定内に `template data-template timeout 60` を追加し、60 秒ごとにテンプレートフレームを自動再送するように変更します。

---

### 質問 3 (Design - 監視通信の保護と可視化)

**問題**: エンタープライズ環境において、Syslog および NetFlow 通信がネットワーク中間経路で閲覧・変更されるリスクを低減するための 3 つの設計施策を述べてください。

**解答と解説**:
1. **Syslog の TLS 暗号化 (Secure Syslog - TCP 6514)**: UDP 514 通信を廃止し、PKI 証明書による相互認証を備えた Secure Syslog (TLS) へ変更し、ログの非改ざん性と機密性を確保します。
2. **NetFlow の IPsec トンネル化または Out-of-Band マネジメント分離**: FNF / IPFIX 通信を専用の VRF (Management VRF) または Out-of-Band インターフェイス経由で転送するか、IPsec VPN トンネル内でカプセル化してコレクタへ送出します。
3. **Control Plane Policing (CoPP) と MPP 適用**: ネットワークデバイス側で CoPP を適用し、許可された NMS / Syslog サーバー以外の IP アドレスからの SNMP (UDP 161/162) / Syslog (UDP 514/TCP 6514) トラフィックのレート制限および制御を実施します。

---

### 質問 4 (実装 - ERSPAN カプセル化と MTU 問題)

**問題**: スイッチ SW1 で ERSPAN を設定し、1500 バイトのデータパケットをコピーして L3 経由のコレクタへ送信しました。しかし、コレクタ側でパケットが途中で破損・欠損していることが判明しました。技術的原因とコンフィグでの解決策を述べてください。

**解答と解説**:
* **原因**: ERSPAN はコピーした元のフレームに対して、20バイトの IP ヘッダー、8バイトの GRE ヘッダー、および 12/20バイトの ERSPAN ヘッダー（計約 50 バイト）をカプセル化付加します。そのため、元パケットが 1500 バイトの場合、送信される ERSPAN パケットは約 1550 バイトとなり、途中の L3 ネットワークの標準 MTU (1500バイト) を超過して断片化が発生するか、DF ビットがセットされている場合ルータでドロップされます。
* **解決策**: 
  1. 途中の L3 スイッチおよびルータの物理インターフェイスで ジャンボフレーム（MTU 9000 等）を有効化する。
  2. ERSPAN 送信元セッションでパケットのトランケーション（`ip truncation`）を設定し、解析に必要なヘッダー領域（例: 128バイト）のみをコピーして送信する。

---

### 質問 5 (トラブルシューティング - ASA Syslog 高負荷)

**問題**: セキュリティ運用チームが ASA ファイヤーウォールのログレベルを `debugging` (Level 7) に変更した直後、ASA の CPU 使用率が 98% に急昇し、パケットドロップが発生し始めました。緊急対処コマンドと再発防止策を述べてください。

**解答と解説**:
* **緊急対処コマンド**:
  ```bash
  ASA# no logging console
  ASA# no logging monitor
  ASA# logging trap informational
  ```
* **解説と防止策**: Console や SSH セッションへのリアルタイムデバッグログ出力 (`logging console` / `logging monitor`) は高い CPU 割込を発生させます。防止策として Console ログ出力を無効化し、Syslog は内部バッファ（`logging buffer`）または専用 Syslog サーバー（`logging host`）へ `informational` (Level 6) 以下で送出します。特定の不要メッセージ（例: `106015` 等）は `no logging message 106015` で個別に無効化します。

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術ガイド一覧です。

* Cisco IOS-XE 17.x Network Management Configuration Guide
* Cisco Technical Note: SNMPv3 Configuration and Verification Guide
* Cisco Flexible NetFlow Configuration Guide, Cisco IOS XE
* Cisco Secure Firewall Management Center (FMC) Syslog & Logging Guide
* Cisco ASA Series General Operations CLI Configuration Guide - Logging
* Cisco Live Presentation: BRKSEC-3015 - Advanced Network Telemetry & Stealthwatch Integration
* Cisco Validated Design (CVD): Secure Network Analytics Deployment Guide
* Cisco Technical Support: ERSPAN Configuration & Troubleshooting Examples


---

## 📝 補足（Notes）

* **SNMPv3 EngineID の注意点**:
  * `snmp-server engineID local` を手動変更した場合、既存の全 SNMPv3 ユーザーの暗号化ハッシュが無効化されるため、ユーザーの再作成が必須となります。
* **NetFlow (NSEL) と ASA / FTD 性能**:
  * ASA / FTD での NSEL は、TCP/UDP コネクション生成時（Flow Created）、終了時（Flow Denied/Deleted）のみログを送出するため、全パケットを追跡するルータの NetFlow に比べて帯域・CPU 消費が格段に抑えられています。
* **Model-Driven Telemetry (MDT) の優位性**:
  * SNMP が約 1〜5 分間隔のポーリングであるのに対し、MDT はサブ秒〜数秒単位でのリアルタイムストリーミングが可能なため、高度な運用自動化に適しています。
