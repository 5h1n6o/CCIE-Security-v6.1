---
layout: default
title: 3.6.c-SYSLOG
nav_order: 3
parent: 3.6-Monitoring
grand_parent: 3.0-Security-Infrastructure
---

# 3.6.c SYSLOG

## 📘 概要

Syslog（System Logging Protocol）は、ネットワーク機器、セキュリティアプライアンス、およびサーバーがシステムイベント、アラート、動作状態ログ、およびセキュリティ監査情報を中央の Syslog サーバー（Log Collector / SIEM: Security Information and Event Management）へリアルタイムに送信・集約するための標準化されたプロトコルです。

CCIE Security v6.1 において、Syslog は単なる監視手段にとどまらず、インシデントハンドリング、コンプライアンス監査、セキュリティポリシーの動作確認、およびコントロール/データプレーン障害のトラブルシューティングを支える中核インフラ技術として位置づけられています。

Syslog の主な利用目的および利用場面は以下の通りです。

* **集中ログ管理とセキュリティ監査**: 複数台の Cisco IOS-XE ルータ、ASA ファイアウォール、FTD (Firepower Threat Defense) アプライアンス、および WLC からログを一元集約し、不正アクセスや設定変更を証跡として保存。
* **リアルタイム・インシデント検知**: SIEM (Splunk, Cisco Secure Analytics 等) と連携し、インターフェイスの Down/Up、IPsec VPN の断線、802.1X 認証失敗、および ACL ドロップイベントを即座に通知・検知。
* **トラブルシューティングとデバッグ**: 稼働中の機器におけるパケット処理異常や制御プロトコルのフラッピング状態を、メモリバッファ (`logging buffered`) や外部サーバー経由で解析。
* **暗号化・高信頼ログ転送 (Secure Syslog)**: RFC 5425 (TLS over TCP 6514) に基づく TLS 暗号化 Syslog 送信により、ネットワーク上でのログ盗聴や改ざんを防止。

従来の RFC 3164 (BSD Syslog / UDP 514) に加え、モダンなネットワークデザインでは構造化データ（Structured Data）と高精度タイムスタンプ（ミリ秒/マイクロ秒）をサポートする RFC 5424 / RFC 5425 規格への移行が進んでいます。


---

## 🔑 要点

Syslog の主要な技術的特徴、重要パラメータ、および設計上の考慮点を下表に示します。

| 項目 | 内容 |
| --- | --- |
| **重要度レベル (Severity)** | Level 0 (Emergency) から Level 7 (Debugging) までの 8 段階で定義 (数値が小さいほど重要度が高い) |
| **転送プロトコル** | UDP 514 (非同期・軽量)、TCP 514 (信頼性重視・接続指向)、TLS/TCP 6514 (暗号化・相互認証) |
| **ログ出力先 (Destinations)** | Console (`logging console`), Terminal (`logging monitor`), Internal Buffer (`logging buffered`), Remote Server (`logging host`), Flash File (`logging file`) |
| **標準規格** | RFC 3164 (BSD Format: PRI + HEADER + MSG), RFC 5424 (IETF Extended Format: PRI + VERSION + TIMESTAMP + HOSTNAME + APP-NAME + PROCID + MSGID + STRUCTURED-DATA + MSG) |
| **ファシリティ (Facility)** | メッセージ送信元プロセスを識別するカテゴリ (標準は `local0` ～ `local7`、デフォルトは `local7` または `syslog`) |
| **主要対応機種** | Cisco IOS/IOS-XE Routers/Switches, Cisco ASA, Cisco FTD / FMC, Cisco Catalyst 9800 WLC, Cisco ISE |
| **制限事項 & 設計注意点** | TCP/TLS 送信時の接続切断時の動作 (ブロック vs ドロップ)、ログ大量発生時の CPU/メモリ枯渇リスク、時刻同期 (NTP) の必須性 |


---

## 🏗 動作原理

Syslog の動作原理は、イベントの発生からメッセージのフォーマット生成、フィルタリング/レート制限、および各種出力先へのディスパッチという一連の処理パイプラインに基づいています。

```
[ システムイベント発生 ]
    │ (インターフェイスダウン、認証失敗、ACLドロップ、CPU高負荷など)
    ▼
[ Syslog メッセージの生成 ]
    │ (Facility + Severity から PRI 値を計算、タイムスタンプを付与)
    ▼
[ フィルタリング & レート制限 (Discriminator / Rate-Limit) ]
    │ (不要メッセージの除去、同一メッセージの送信頻度制限)
    ▼
┌───┴───────────────────────────────────────────┐
│              出力先へのディスパッチ            │
├───────────────┬───────────────┬───────────────┤
│ Console / TTY │ Memory Buffer │ Remote Host   │
│ (console/mon) │ (buffered)    │ (UDP/TCP/TLS) │
└───────────────┴───────────────┴───────────────┘
```

### 1. Syslog メッセージ構造 (RFC 3164 vs RFC 5424)

Syslog メッセージは、優先度を示す PRI パート、発生時刻と送信元を示す HEADER パート、および実際の本文である MSG パートで構成されます。

#### RFC 3164 フォーマット例
```text
<34>Oct 24 10:15:30.123 R1 %LINK-3-UPDOWN: Interface GigabitEthernet0/0/0, changed state to up
```
* **PRI**: `<34>` ── Facility (`local4` = 20) × 8 + Severity (`Errors` = 3) = 160 + 3 = 163 ... 実際には `auth` (4) × 8 + `Critical` (2) = 34。
* **HEADER**: `Oct 24 10:15:30.123 R1` (タイムスタンプ + ホスト名)。
* **MSG**: `%LINK-3-UPDOWN: Interface...` (Cisco 固有のファシリティ/セベリティ/メッセージコードと詳細本文)。

#### RFC 5424 構造化フォーマット例
```text
<165>1 2026-10-24T10:15:30.123456Z R1.example.com IOS-XE 1234 ID47 [exampleSDID@32473 iut="3" eventSource="Application" eventID="1011"] %LINK-3-UPDOWN: Interface GigabitEthernet0/0/0, changed state to up
```
* **VERSION**: `1`
* **TIMESTAMP**: ISO 8601 形式 (`2026-10-24T10:15:30.123456Z` : マイクロ秒精度および UTC 表記)。
* **STRUCTURED-DATA**: 名前/値ペアで構成されたキー情報 (`[exampleSDID@32473 ... ]`) であり、プログラムによる自動パースが容易。

### 2. Syslog 重要度レベル (Severity Levels)

Cisco IOS/ASA/FTD では、0 から 7 までの 8 段階の重要度レベルが定義されています。

| 数値 (Level) | 名称 | キーワード (IOS/ASA) | 概要・発生事象例 |
| :-: | --- | --- | --- |
| **0** | **Emergency** | `emergencies` | システム使用不能状態 (ハードウェア致命的故障、カーネルパニック) |
| **1** | **Alert** | `alerts` | 即時対応が必要な状態 (温度閾値超過、電源ユニット障害) |
| **2** | **Critical** | `critical` | 致命的なクリティカル状態 (主系リンクダウン、HA ピア切断) |
| **3** | **Error** | `errors` | エラー状態 (インターフェイス Up/Down、IPsec SA 確立失敗) |
| **4** | **Warning** | `warnings` | 警告状態 (メモリ使用率高、クロック非同期、Duplex ミスマッチ) |
| **5** | **Notice** | `notifications` | 正常だが重要な状態 (設定変更、802.1X 認証成功、システムリブート) |
| **6** | **Informational**| `informational` | 情報メッセージ (ACL ログ、通常の接続確立/切断) |
| **7** | **Debugging** | `debugging` | デバッグ出力 (debug コマンドによるプロトコル詳細ログ) |

> **記憶用アナグラム (Mnemonic)**:  
> **E**very **A**lmost **C**ritical **E**rror **W**arns **N**ew **I**nternational **D**evelopers  
> (0: Emergency, 1: Alert, 2: Critical, 3: Error, 4: Warning, 5: Notice, 6: Informational, 7: Debugging)

### 3. トランスポートプロトコルの特性比較

* **UDP 514**: ノンブロッキングで非常に高速。ネットワーク不通やサーバーダウン時でも機器の通信処理や CPU に影響を与えないが、パケットドロップや順序逆転が発生する可能性がある。
* **TCP 514**: ハンドシェイクと確認応答 (ACK) により確実にログを届ける。ネットワーク遅延やサーバー応答停止時に機器側の送信バッファが溢れ、接続ブロックモードの場合はトラフィック処理に遅延が生じるリスクがある。
* **TLS over TCP 6514 (Secure Syslog)**: TCP の信頼性に加え、X.509 証明書による相互認証および AES 暗号化を提供。暗号化処理に伴う CPU 負荷と PKI/証明書管理の手間が発生する。


---

## ⚙ 動作シーケンス

Cisco アプライアンス内部における Syslog メッセージ生成から外部ネットワーク送信までの処理シーケンスは以下の通りです。

```
[ イベント発生プロセス ]
       │
       ▼
1. ログイベントの作成
   - メッセージコード、Severity、タイムスタンプ (NTP同期時刻) の取得
       │
       ▼
2. メッセージフォーマット化
   - 施設コード (Facility) と Severity から PRI 計算
   - Cisco 固有ヘッダー (%FACILITY-SEVERITY-MNEMONIC) の付与
       │
       ▼
3. Logging Discriminator / Logging List によるフィルタリング
   - 定義された正規表現 (Body, MsgID, Severity 等) に基づき、ログをドロップまたは通過判定
       │
       ▼
4. レート制限 (Rate-Limiting) 処理
   - 同一ログの過剰発生（ストーム）時に指定されたインターバル (例: 1秒間に最大10件) で制限
       │
       ▼
5. ディスパッチ & トランスポート送信
   ┌────────────────────────────────────────────────────────┐
   │ - Console/Monitor: TTY 画面へ直接レンダリング           │
   │ - Buffer: リングバッファメモリへの書き込み             │
   │ - Socket Transmit: 指定された出力用 VRF から送出        │
   │   - UDP 514: パケット生成後、即時送信 (ACKなし)         │
   │   - TCP/TLS: セッション確立確認後、暗号化/プレーン送信   │
   └────────────────────────────────────────────────────────┘
```


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security ラボ試験において、Syslog の設定は単なる「ログ出力設定」にとどまらず、複雑な制約要件を満たす高度なフィルタリングやトランスポートセキュリティ、VRF 統合が要求されます。

### 1. 試験で頻出する設定要件とポイント

* **Logging Discriminator (IOS-XE) による高度なフィルタリング**:
  特定のメッセージ（例: `%LINK-3-UPDOWN` や特定 IP アドレスを含むログ）のみを収集サーバーへ転送し、コンソールや内部バッファには別の基準でフィルタリングして出力させる要件。
* **ASA Logging List による条件分岐**:
  ASA において、Syslog ID（例: `%ASA-6-302013`）や特定セベリティのログを抽出し、管理用インターフェイスから TCP または Secure TLS 経由で送信する設定。
* **Secure Syslog (TLS) と PKI 連携**:
  証明書（Trustpoint）を使用して Syslog サーバーとの TLS セッション（TCP 6514）を確立し、ログ通信を暗号化。認証局 (CA) から発行された証明書の適切な適用が必要。
* **OOB (Out-of-Band) 管理 VRF 経由のログ送出**:
  IOS-XE においてデータプレーンではなく管理専用 VRF (`Mgmt-vrf`) のソースインターフェイス（`GigabitEthernet0` 等）から Syslog パケットを出力する設定 (`logging host ... vrf Mgmt-vrf`)。
* **TCP 送信時のブロッキング動作回避**:
  TCP Syslog 使用時、サーバー応答停止に伴う機器のコマンド入力遅延や通信ブロックを防ぐための適切なキューイングおよびフェールオーバー設定。

### 2. よくある設定ミスとトラブルシューティングポイント

1. **ソースインターフェイスおよび VRF のミスマッチ**:
   `logging source-interface` で指定したインターフェイスの IP アドレスが `logging host` で指定した VRF 内でルーティングできない場合、Syslog パケットが送出されない。
2. **ASA における `logging enable` の設定漏れ**:
   ASA では `logging host` を設定しても、グローバルで `logging enable` を実行しない限りリモート Syslog が有効化されない。
3. **NTP 時刻非同期によるログ監査無効化**:
   Syslog メッセージのタイムスタンプが不正確な場合、セキュリティインシデントの相関分析が不可能となる。`service timestamps log datetime msec localtime show-timezone` の設定が必須。
4. **Secure Syslog (TLS) における IP / FQDN の不一致**:
   Syslog サーバーの証明書内の Subject Alternative Name (SAN) または Common Name (CN) と、機器側で指定した `logging host` の IP/FQDN が一致せず TLS ハンドシェイクが失敗する。

---

## 🛠 設定方法

Cisco IOS-XE、Cisco ASA、および Cisco FMC/FTD における詳細な設定例を示します。

### 1. Cisco IOS-XE における設定例

#### 要件:
* OOB 管理 VRF (`Mgmt-vrf`) 経由で Syslog サーバー (`192.168.100.50`) へ送信。
* トランスポートプロトコルは TLS (Port 6514) を使用し、RFC 5424 フォーマットを適用。
* 不要なデバッグログを除外する Logging Discriminator を設定。

```text
! 1. NTP 時刻同期およびタイムスタンプ詳細化の設定
ntp server vrf Mgmt-vrf 192.168.100.1
service timestamps log datetime msec localtime show-timezone year

! 2. Logging Discriminator の定義 (特定メッセージのドロップ・通過制御)
! メッセージボディに "CRYPTO-6-IKEMP" を含むログを除外し、Severity 5 以上のみを通過
logging discriminator DISC_SIEM msg-body drop CRYPTO-6-IKEMP severity drop 6 7

! 3. バッファおよびコンソールログの最適化
logging buffered 64000 informational
logging console critical

! 4. ソースインターフェイスおよび VRF 経由の Secure Syslog (TLS) 設定
logging source-interface GigabitEthernet0 vrf Mgmt-vrf
logging host 192.168.100.50 vrf Mgmt-vrf transport tls port 6514 discriminator DISC_SIEM sequence-num
logging format rfc5424
logging trap informational
logging facility local4
```

### 2. Cisco ASA における設定例

#### 要件:
* ASA の `management` インターフェイス経由で Syslog サーバー (`10.1.100.50`) へ TCP 514 で送信。
* 接続切断時のデータドロップ（非ブロック）を設定し、ビルドインセッションログ (`302013`, `302014`) を特定リストに抽出。

```text
! 1. ログ機能のグローバル有効化
logging enable
logging timestamp

! 2. ログバッファおよびコンソール制限
logging buffer-size 1048576
logging buffered informational
logging console errors

! 3. カスタム Logging List の作成 (特定セベリティおよび特定 Message ID の抽出)
logging list LIST_SIEM level warning
logging list LIST_SIEM message 302013-302016
logging list LIST_SIEM message 106023

! 4. リモート Syslog サーバーとトランスポート設定
logging host management 10.1.100.50 tcp/514
logging list LIST_SIEM host 10.1.100.50
logging facility 20

! 5. TCP 送信障害時のブロッキング防止設定 (キューサイズ調整とバッファドロップ)
logging queue 512
```

### 3. Cisco FTD / FMC (Platform Settings / FlexConfig) における設定例

FMC 管理下の FTD においては、Platform Settings より GUI で統合設定を行います。

#### FMC GUI 設定手順:
1. FMC GUI にログインし、**Devices** ──> **Platform Settings** へ移動。
2. 対象の FTD ポリシーを編集し、左メニューから **Syslog** を選択。
3. **Syslog Servers** タブで以下を設定:
   * **Allow Syslog on Management Interface**: 有効化
   * **IP Address**: `192.168.100.50`
   * **Protocol / Port**: `TCP / 514` または `TLS / 6514`
   * **Log Exchange Format**: `Default` または `Emblematics`
4. **Logging Setup** タブで `Enable Logging` にチェックを入れる。
5. **FlexConfig** による詳細パラメータ調整 (必要な場合):
   * FlexConfig Text Object および Policy を使用し、`logging rate-limit` や `logging class` を追加注入。

```text
! FlexConfig 経由で挿入する追加 CLI 例
logging rate-limit 100 1 level 6
logging class default console errors
```

---

## 🔍 検証コマンド

Syslog の動作確認、パケット送出状態、バッファ確認、およびデバッグで使用する主要コマンドを下表に示します。

| 目的 | Cisco IOS-XE コマンド | Cisco ASA / FTD CLI コマンド |
| --- | --- | --- |
| **全体ステータス・設定確認** | <code>show logging</code> | <code>show logging</code> |
| **Syslog サーバー接続・統計確認** | <code>show logging \| include host</code> | <code>show logging info</code> |
| **内部ログバッファの閲覧** | <code>show logging buffer</code> | <code>show logging buffer</code> |
| **Logging Discriminator の確認** | <code>show logging discriminator</code> | N/A (Logging List を確認) |
| **TLS/Secure Syslog 接続状態確認** | <code>show logging \| section TLS</code> | <code>show crypto ipsec sa</code> / <code>show ssl</code> |
| **Syslog パケット送出リアルタイム確認** | <code>debug logging console</code> | <code>logging debug-trace</code> |
| **インフラカウンターの確認** | <code>show platform software logging statistics</code> | <code>show logging queue</code> |


## 🚨 トラブルシュート

Syslog 転送障害および設定トラブルにおける主要な症状、原因、確認コマンド、および対処方法を下表に整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **Syslog サーバーにログが一切届かない** | 1. 物理/L3 疎通不可<br>2. ルーティング/VRF 設定不一致<br>3. ASA で `logging enable` の設定漏れ | <code>ping vrf <vrf> <server_ip></code><br><code>show logging</code><br><code>show ip route vrf <vrf></code> | 1. ネットワーク疎通とルートを確認<br>2. `logging source-interface` と VRF 名を合致させる<br>3. ASA で `logging enable` を実行 |
| **TCP Syslog 使用時に機器のコマンド応答が極端に遅くなる** | Syslog サーバーがダウンし、TCP 送信バッファが溢れブロック状態が発生 | <code>show logging queue</code><br><code>show processes cpu</code> | 1. `logging queue` サイズを拡大<br>2. TCP 送信タイムアウトを調整し、非ブロックモードに変更<br>3. UDP または非同期書き込みに変更 |
| **Secure Syslog (TLS) の接続が確立しない** | 1. PKI 証明書（CA / Trustpoint）の検証失敗<br>2. ポート 6514 のブロッキング<br>3. 時刻非同期による証明書有効期限エラー | <code>show crypto pki trustpoints</code><br><code>show clock</code><br><code>debug ssl openssl errors</code> | 1. NTP を設定し正しい時刻に同期<br>2. Syslog サーバーの CA 証明書を Trustpoint に正しくインポート<br>3. FQDN/IP の SAN 一致を確認 |
| **不要なログが大量発生しログバッファやサーバーが溢れる** | 1. Debug コマンドが有効化されたまま放置<br>2. ACL ログ (`log` オプション) の過剰出力<br>3. レート制限の未設定 | <code>show debug</code><br><code>show access-lists</code><br><code>show logging | include rate</code> | 1. 不要な `debug all` を `undebug all` で停止<br>2. `logging rate-limit` を設定してストームを防止<br>3. Logging Discriminator で特定の不要メッセージを除外 |
| **ログのタイムスタンプが実際の発行時刻とズレている** | 1. NTP サーバーとの同期外れ<br>2. タイムゾーン/夏時間 (DST) の未設定<br>3. タイムスタンプ精度が秒単位になっている | <code>show ntp status</code><br><code>show clock detail</code> | 1. NTP サーバーを設定し同期状態 (`assoc`) を確認<br>2. `service timestamps log datetime msec localtime show-timezone` を適用 |


---

## ⚠ 制限事項

1. **プラットフォーム・バッファサイズの制限**:
   * Cisco IOS/IOS-XE の DRAM 内 `logging buffered` の最大サイズは機種依存（ Catalyst 9000 シリーズでは通常数 MB ～ 数十 MB 程度）。過剰なバッファ割り当てはコントロールプレーンのメモリを圧迫します。
2. **ASA / FTD におけるデータプレーン・パフォーマンスへの影響**:
   * ASA/FTD においてコネクションごとに Syslog (`302013`/`302014`) を高頻度で発行する場合、CPU 負荷が急増します。大規模環境では Syslog の代わりに **NSEL (NetFlow Secure Event Logging)** を使用することが強く推奨されます。
3. **TCP Syslog 接続障害時のバッファ動作**:
   * TCP トランスポートを使用している際、Syslog サーバーとの接続が途絶すると、ローカルのキュー制限 (`logging queue`) に達した段階で以降のログがドロップされるか、設定によってはコントロールプレーン処理が同期待ちとなりブロックされます。
4. **Console Logging のボトルネック**:
   * `logging console debugging` や高低速シリアルポート（9600 bps 等）への多量ログ出力は、スイッチ/ルータの CPU 処理を極度に遅延させ、最悪の場合デバイスのハングアップを引き起こします。


---

## 🔄 他技術との関連

* **NTP (Network Time Protocol)**:  
  Syslog の最も重要な依存技術。正確な時刻同期がなされていない場合、異なる機器間でのイベント相関分析や SIEM でのフォレンジック解析が完全に不可能となります。
* **NSEL (NetFlow Secure Event Logging) / IPFIX**:  
  Cisco ASA/FTD における高速トラフィックログ収集プロトコル。接続の作成・破棄・ドロップイベントを UDP バイナリ形式で送出し、Syslog のテキスト処理負荷を劇的に軽減します。
* **VRF (Virtual Routing and Forwarding)**:  
  マネジメントプレーンとデータプレーンを分離する構成（OOB 管理）において、Syslog パケットを指定した VRF (例: `Mgmt-vrf`) 経由で安全に送出。
* **PKI (Public Key Infrastructure) & TLS**:  
  Secure Syslog (RFC 5425) 確立時の相互認証およびセッション暗号化鍵の生成に使用。
* **CoPP (Control Plane Policing)**:  
  大量のイベント発生時に CPU へ殺到するログ生成プロセス処理を制御し、管理通信（SSH, SNMP, BGP 等）を保護。


---

## 🧩 比較表

### 1. Syslog トランスポートプロトコルの比較

| 比較項目 | UDP Syslog (Port 514) | TCP Syslog (Port 514) | Secure TLS Syslog (Port 6514) |
| --- | --- | --- | --- |
| **信頼性** | ベストエフォート (到達保証なし) | 接続指向 (ACK による確実な到達) | 接続指向 (ACK による確実な到達) |
| **暗号化** | なし (平文) | なし (平文) | あり (TLS 1.2/1.3 暗号化) |
| **認証機構** | なし | なし | あり (X.509 デジタル証明書) |
| **オーバーヘッド** | 最小 (CPU/帯域負荷極少) | 中程度 (TCP 状態管理が必要) | 高 (TLS ハンドシェイク・暗号化演算) |
| **障害時の影響** | サーバー不通時も機器へ影響なし | サーバー不通時にキュー溢れリスク | サーバー不通時にキュー溢れリスク |
| **推奨用途** | 大容量内部 LAN ログ収集 | 信頼性が求められる WAN 送信 | DMZ・パブリック網・厳格コンプライアンス環境 |

### 2. Syslog 標準規格の比較 (RFC 3164 vs RFC 5424)

| 比較項目 | RFC 3164 (BSD Syslog) | RFC 5424 (IETF Syslog) |
| --- | --- | --- |
| **構造定義** | PRI + HEADER (Time/Host) + MSG | PRI + VER + TIME + HOST + APP + PROCID + MSGID + SD + MSG |
| **タイムスタンプ精度** | 秒単位 (`Oct 24 10:15:30`) | ミリ秒/マイクロ秒 + UTC オフセット (`2026-10-24T10:15:30.123Z`) |
| **構造化データ (SD)** | 非対応 (すべて非定形テキスト) | 対応 (`[キー="値"]` 形式によるプログラム解析の容易化) |
| **文字エンコーディング** | ASCII 中心 (文字化けリスク) | UTF-8 標準サポート |
| **互換性** | ほぼすべてのレガシー機器で対応 | モダンな IOS-XE / SIEM で標準サポート |


---

## 💡 ベストプラクティス

1. **必ず NTP 時刻同期を適用し、ミリ秒単位のタイムスタンプを有効化する**:
   ```text
   service timestamps log datetime msec localtime show-timezone year
   ```
2. **コンソールログの停止または Severity の制限**:
   本番環境およびラボ試験では、大量ログによる CPU ハングアップを防ぐため、コンソール出力は `critical` 以下に制限するか完全に無効化します。
   ```text
   logging console critical
   ```
3. **管理専用 VRF (OOB) によるログ送出の分離**:
   業務トラフィックの輻輳や障害の影響を受けないよう、Syslog 送出は管理用 VRF 経由で行います。
   ```text
   logging source-interface GigabitEthernet0 vrf Mgmt-vrf
   logging host 192.168.100.50 vrf Mgmt-vrf
   ```
4. **Logging Discriminator や ASA Logging List によるフィルタリング**:
   不要な情報ログ（例: 頻繁なネイバー維持ログ等）を送信元機器で遮断し、ネットワーク帯域と SIEM サーバーのライセンス/ストレージ消費を節約します。
5. **バッファサイズの適切な確保**:
   通信障害時でも一時的に機器内部にログを保持できるよう、十分なバッファメモリ（例: 64 KB ～ 1 MB）を確保します。
   ```text
   logging buffered 1048576 informational
   ```


## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験を想定した、実戦的で高度な Syslog 設定演習サンプル 11 選です。各問題には「問題」「要件」「完全な設定例」「検証方法」が含まれています。

---

### サンプル 1: IOS-XE OOB 管理 VRF 経由の Syslog 基礎設定

#### 問題:
ルータ R1 のログを、業務トラフィックから分離された Out-of-Band 管理ネットワーク経由で中央コレクターに送出してください。

#### 要件:
* 送信元インターフェイスは `GigabitEthernet0`（VRF: `Mgmt-vrf`）。
* Syslog サーバー `192.168.100.50`（VRF: `Mgmt-vrf`）へ UDP 514 で送信。
* ファシリティは `local5`、重要度レベルは `informational` 以上を転送。
* タイムスタンプにミリ秒、ローカルタイムゾーン、および年を表示。

#### 設定例:
```text
! R1 (Cisco IOS-XE)
clock timezone JST 9
service timestamps log datetime msec localtime show-timezone year

logging facility local5
logging source-interface GigabitEthernet0 vrf Mgmt-vrf
logging host 192.168.100.50 vrf Mgmt-vrf
logging trap informational
```

#### 検証方法:
```text
R1# show logging
! 出力結果内に "Logging to 192.168.100.50 (vrf Mgmt-vrf)" が表示され、パケットカウントが増加していることを確認

R1# send log 5 "TEST SYSLOG MESSAGE FROM R1"
! サーバー側で受信ログを確認
```

---

### サンプル 2: IOS-XE Logging Discriminator によるメッセージ除外

#### 問題:
ルータ R1 において、Syslog サーバーへ送信するログからノイズとなる特定メッセージを除外してください。

#### 要件:
* メッセージボディに `BGP-5-ADJCHANGE` または `SEC_LOGIN-5-LOGIN_SUCCESS` が含まれるログを無条件で除外。
* それ以外の `notice` (Severity 5) 以上のログを Syslog サーバー `10.1.1.100` へ送信。
* Discriminator 名は `FILTER_NOISE` とする。

#### 設定例:
```text
! R1 (Cisco IOS-XE)
logging discriminator FILTER_NOISE msg-body drop BGP-5-ADJCHANGE msg-body drop SEC_LOGIN-5-LOGIN_SUCCESS severity drop 6 7

logging host 10.1.1.100 discriminator FILTER_NOISE
logging trap notice
```

#### 検証方法:
```text
R1# show logging discriminator FILTER_NOISE
! 定義されたドロップルールを確認

R1# clear logging
R1# interface GigabitEthernet0/1
R1(config-if)# shutdown
! リンクダウンログは送信され、BGPフラップ等の対象ログがドロップされていることをコレクター側で確認
```

---

### サンプル 3: IOS-XE RFC 5424 構造化ログフォーマットの適用

#### 問題:
SIEM サーバーでの高精度パースを実現するため、R1 のログフォーマットを RFC 5424 に変更してください。

#### 要件:
* 全 Syslog メッセージを IETF RFC 5424 構造化フォーマットに変換。
* シーケンス番号をログに付与。
* サーバー `172.16.10.50` へ送出。

#### 設定例:
```text
! R1 (Cisco IOS-XE)
service sequence-numbers
service timestamps log datetime msec unit-msec localtime show-timezone

logging format rfc5424
logging host 172.16.10.50
```

#### 検証方法:
```text
R1# show logging | include 172.16.10.50
! 送信フォーマットが RFC 5424 形式（<PRI>1 TIMESTAMP HOSTNAME...）になっていることを確認
```

---

### サンプル 4: IOS-XE Secure Syslog (TLS over TCP 6514)

#### 問題:
R1 と Syslog サーバー間の通信を TLS 暗号化し、盗聴および改ざんを防止してください。

#### 要件:
* サーバー `192.168.1.200` のポート `6514` へ TLS 経由でログ送信。
* 既存の Trustpoint `MY_CA` に含まれる CA 証明書を用いてサーバーを検証。
* 送信元は `Loopback0`。

#### 設定例:
```text
! R1 (Cisco IOS-XE)
crypto pki trustpoint MY_CA
 enrollment profile CA_PROFILE
 revocation-check crl none

logging source-interface Loopback0
logging host 192.168.1.200 transport tls port 6514
```

#### 検証方法:
```text
R1# show logging
! "Transport TLS Port 6514 Status: Active" などの表示を確認

R1# show crypto ipsec sa / show ssl status
! TLS ハンドシェイク成功ログを確認
```

---

### サンプル 5: IOS-XE ログ発生レート制限とバッファ保護

#### 問題:
大量のセキュリティイベント発生時に CPU 負荷を抑え、ログの損失を防御してください。

#### 要件:
* 内部バッファサイズを `1048576` バイト (1MB) に拡張。
* ログの発生レートを 1 秒あたり最大 20 通に制限。
* コンソールログは `errors` (Severity 3) 以下に制限。

#### 設定例:
```text
! R1 (Cisco IOS-XE)
logging buffered 1048576 informational
logging console errors
logging rate-limit 20 except 3
```

#### 検証方法:
```text
R1# show logging
! Buffer size: 1048576 bytes, Rate limit: 20 msg/sec を確認
```

---

### サンプル 6: ASA 基礎 Syslog および Management インターフェイス出力

#### 問題:
Cisco ASA5516 において Syslog 転送を有効化し、管理ネットワーク経由で出力してください。

#### 要件:
* ログ機能を有効化 (`logging enable`)。
* `management` インターフェイス経由で Syslog サーバー `10.254.1.50` (UDP 514) へ送信。
* タイムスタンプを有効化。

#### 設定例:
```text
! ASA1 (Cisco ASA)
logging enable
logging timestamp
logging host management 10.254.1.50
logging facility 20
logging trap warnings
```

#### 検証方法:
```text
ASA1# show logging
! Syslog logging: enabled
! Simple logging host: management 10.254.1.50

ASA1# show logging info
```

---

### サンプル 7: ASA Logging List による特定イベント抽出

#### 問題:
ASA において、特定の重要セキュリティイベント（ACL ドロップおよび VPN 接続ログ）のみを抽出して Syslog サーバーへ送出してください。

#### 要件:
* Logging List 名 `LIST_SECURITY_EVENTS` を作成。
* Severity `alerts` 以上の全ログ、および ACL ドロップログ (`106023`)、AnyConnect 接続ログ (`716001`) を抽出。
* サーバー `10.254.1.100` へ送出。

#### 設定例:
```text
! ASA1 (Cisco ASA)
logging enable

logging list LIST_SECURITY_EVENTS level alerts
logging list LIST_SECURITY_EVENTS message 106023
logging list LIST_SECURITY_EVENTS message 716001

logging host inside 10.254.1.100
logging list LIST_SECURITY_EVENTS host 10.254.1.100
```

#### 検証方法:
```text
ASA1# show logging list LIST_SECURITY_EVENTS
! 定義された Message ID およびレベルを確認
```

---

### サンプル 8: ASA Secure Syslog (TLS/TCP 6514) の構築

#### 問題:
ASA から Syslog サーバーへの通信を PKI 証明書を用いて TLS 暗号化してください。

#### 要件:
* サーバー `172.16.200.50` へ TCP Port 6514 (TLS) で接続。
* 信頼された Trustpoint `ASA_CA` を使用して相互検証。
* `outside` インターフェイスから送出。

#### 設定例:
```text
! ASA1 (Cisco ASA)
logging enable
logging host outside 172.16.200.50 tcp/6514 secure
```

#### 検証方法:
```text
ASA1# show logging info
! Secure TLS status を確認
```

---

### サンプル 9: ASA TCP Syslog ブロッキング防止とキュー最適化

#### 問題:
ASA において TCP Syslog 転送を使用する際、サーバー停止に伴う ASA の通信処理フリーズを防止してください。

#### 要件:
* Syslog 送信キューの最大サイズを `1024` パケットに増加。
* サーバー応答不可時にセッション確立要求を待たずにバッファドロップする設定。

#### 設定例:
```text
! ASA1 (Cisco ASA)
logging enable
logging queue 1024
logging host inside 10.0.0.50 tcp/514
! TCP 接続障害時の過剰ブロックを防ぐためキュー調整を実施
```

#### 検証方法:
```text
ASA1# show logging queue
! キューの使用状況およびドロップカウントを確認
```

---

### サンプル 10: FTD / FMC FlexConfig による Syslog レート制限注入

#### 問題:
FMC で管理される FTD アプライアンスに対し、FlexConfig を使用して Syslog レート制限コマンドを挿入してください。

#### 要件:
* FTD の CLI に `logging rate-limit 50 1 level 6` を注入。
* デフォルトログクラスの出力レベルを制御。

#### 設定例 (FlexConfig Text / CLI):
```text
! FlexConfig Object: FX_Syslog_RateLimit
logging rate-limit 50 1 level 6
logging class default console errors
```

#### 検証方法:
```text
! FTD CLI にアクセス
FTD> show running-config logging
! 注入された logging rate-limit 設定を確認
```

---

### サンプル 11: IOS-XE CoPP による過剰ログ生成プロセスの保護

#### 問題:
大量の ACL ドロップログが発生した際、CPU コントロールプレーンが過負荷に陥るのを CoPP で防止してください。

#### 要件:
* Syslog 送出トラフィック (UDP/TCP/TLS) に対する CoPP ポリシーの適用。
* コントロールプレーン宛ての過剰なパケットをポリシング。

#### 設定例:
```text
! R1 (Cisco IOS-XE)
class-map match-all CM_SYSLOG
 match access-group name ACL_SYSLOG

ip access-list extended ACL_SYSLOG
 permit udp any any eq syslog
 permit tcp any any eq 6514

policy-map PM_COPP
 class CM_SYSLOG
  police 1000000 bc 31250
   conform-action transmit
   exceed-action drop

control-plane
 service-policy input PM_COPP
```

#### 検証方法:
```text
R1# show policy-map control-plane class CM_SYSLOG
! パケットカウントおよびポリシングドロップ数を確認
```


## ❓ 想定試験問題

CCIE Security ラボ試験・記述試験レベルの想定問題 6 問および詳細な解説です。

---

### 問題 1 (コンフィグ読解・フィルタリング)

以下の Cisco IOS-XE ルータのコンフィグを分析し、Syslog サーバー `192.168.10.50` へ送信されるログメッセージの条件として正しいものを選択してください。

```text
logging discriminator DISC1 msg-body drop SSH-5-EXIT severity drop 6 7
logging host 192.168.10.50 discriminator DISC1
logging trap notice
```

* **A**: 重要度レベル 6 (Informational) 以上のすべてのログが送信される。
* **B**: メッセージボディに `SSH-5-EXIT` が含まれず、かつ重要度レベル 5 (Notice) 以上のログのみが送信される。
* **C**: 重要度レベル 0 から 7 までのすべてのログが送信されるが、`SSH-5-EXIT` のみ除外される。
* **D**: メッセージボディに `SSH-5-EXIT` が含まれるログのみが送信される。

#### 正解: **B**

#### 解説:
1. `logging discriminator DISC1 msg-body drop SSH-5-EXIT` により、メッセージボディに `SSH-5-EXIT` が含まれるログが除外されます。
2. `severity drop 6 7` により、重要度レベル 6 (Informational) および 7 (Debugging) のログがドロップ（除外）されます。
3. グローバルの `logging trap notice`（Severity 5）と組み合わせることで、レベル 5 (Notice) 以上のログのうち、`SSH-5-EXIT` を除いたものが送信されます。

---

### 問題 2 (トラブルシューティング・TCP ブロッキング)

エンジニアが ASA ファイアウォールで TCP 514 を使用した Syslog 送信を設定したところ、Syslog サーバーの障害発生時に ASA 経由の通信設定変更コマンド入力が極端に遅延する現象が発生しました。この問題を根本的に回避・軽減するための最も適切な設定変更はどれですか。

* **A**: `logging console debugging` を設定する。
* **B**: `logging queue` の値を増やすとともに、送信プロトコルを UDP に変更するか非同期設定を調整する。
* **C**: `logging rate-limit 1 1` を設定してログ発生を完全に停止する。
* **D**: `no logging enable` を実行する。

#### 正解: **B**

#### 解説:
TCP Syslog 使用時、Syslog サーバーとの接続が途絶すると、ASA の送信キューが満杯になり、TCP 送信制御処理がコマンド実行プロセスをブロックすることがあります。送信キューサイズ (`logging queue`) を拡大し、信頼性が許容される場合は UDP 転送に変更するか、ブロッキングを回避するキュー制御を導入するのがベストプラクティスです。

---

### 問題 3 (実装・VRF-aware Syslog)

Cisco IOS-XE において、管理専用 VRF `Mgmt-vrf` に所属する物理ポート `GigabitEthernet0` から Syslog パケットを送信する場合、正しい設定の組み合わせを選択してください。

```text
(1) logging source-interface GigabitEthernet0 vrf Mgmt-vrf
(2) logging host 10.1.1.50 vrf Mgmt-vrf
(3) logging source-interface GigabitEthernet0
(4) logging host 10.1.1.50
```

* **A**: (1) と (2)
* **B**: (3) と (4)
* **C**: (1) と (4)
* **D**: (2) と (3)

#### 正解: **A**

#### 解説:
IOS-XE において VRF 内のインターフェイスから特定 VRF 内の Syslog サーバーへログを送信する場合、`logging source-interface <if> vrf <vrf_name>` および `logging host <ip> vrf <vrf_name>` の両方に該当 VRF 名を指定する必要があります。片方でも VRF 名が欠落するとデフォルトルーティングテーブルを参照し、ログ送出に失敗します。

---

### 問題 4 (デザイン・Secure Syslog)

RFC 5425 (Secure Syslog over TLS) を導入する際の必要要件として、不適切な記述はどれですか。

* **A**: トランスポートプロトコルとして標準ポート TCP 6514 が一般的に使用される。
* **B**: 送信側機器と受信側サーバーの両方で正確な時刻同期 (NTP) が必要である。
* **C**: 暗号化通信のために UDP プロトコル上で直接 TLS ハンドシェイクを実行する。
* **D**: X.509 デジタル証明書と CA Trustpoint の設定が必要である。

#### 正解: **C**

#### 解説:
RFC 5425 Secure Syslog は **TCP** (標準ポート 6514) の上で TLS 接続を確立します。UDP 上で暗号化を行う場合は DTLS (Datagram TLS) となり、RFC 5425 の規格（TLS over TCP）とは異なります。

---

### 問題 5 (コンフィグ読解・ASA Logging List)

以下の ASA 設定において、Syslog サーバー `10.0.0.10` に送信されるログメッセージに関する正しい記述を選択してください。

```text
logging enable
logging list MY_LIST level errors
logging list MY_LIST message 106023
logging host inside 10.0.0.10
logging list MY_LIST host 10.0.0.10
```

* **A**: Severity `errors` (Level 3) 以下のログと Message ID `106023` のログが送出される。
* **B**: Message ID `106023` のログのみが送出される。
* **C**: 全レベルのログが送出される。
* **D**: Severity `errors` (Level 3) 以上のログ（Emergency, Alert, Critical, Error）および Message ID `106023` (Deny IP by ACL) が送出される。

#### 正解: **D**

#### 解説:
ASA の `logging list` では、指定した `level` (ここでは `errors` 以上 = Level 0～3) のログと、明示的に指定した `message` ID (`106023`) の論理和 (OR) が抽出され、該当ホストへ転送されます。

---

### 問題 6 (トラブルシューティング・タイムスタンプ)

Syslog サーバー上で受信したログの時刻が実際の発生時刻より 9 時間遅れて表示されており、ミリ秒情報も欠落していました。IOS-XE ルータ上で修正すべき最も適切なコマンドはどれですか。

* **A**: `service timestamps log datetime msec localtime show-timezone` および `clock timezone JST 9`
* **B**: `no logging timestamp`
* **C**: `service sequence-numbers`
* **D**: `ntp disable`

#### 正解: **A**

#### 解説:
タイムスタンプのズレはタイムゾーン (`clock timezone`) の未設定および `service timestamps log` における `localtime` オプションの欠落が原因です。また、ミリ秒精度の付与には `msec` オプションが必要となります。


## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術ガイド一覧です。

* Cisco IOS-XE 17.x Network Management Configuration Guide - System Logging
* Cisco ASA Series General Operations CLI Configuration Guide - Monitoring Syslog Messages
* Cisco Secure Firewall Management Center Administration Guide - Logging Configurations
* RFC 3164 - The BSD Syslog Protocol
* RFC 5424 - The Syslog Protocol
* RFC 5425 - Transport Layer Security (TLS) Transport Mapping for Syslog
* Cisco Technical Notes - ASA Syslog Configuration and Troubleshooting
* Cisco Live - Advanced Network Monitoring and Security Logging Architecture


---

## 📝 補足（Notes）

### 1. Severity レベル暗記用まとめ

* **0: Emergency (emergencies)** - システム使用不能
* **1: Alert (alerts)** - 即時対応が必要
* **2: Critical (critical)** - 致命的条件
* **3: Error (errors)** - エラー条件
* **4: Warning (warnings)** - 警告条件
* **5: Notice (notifications)** - 正常だが重要な条件
* **6: Informational (informational)** - 案内情報
* **7: Debugging (debugging)** - デバッグ出力

### 2. トラブルシューティング時のチェックリスト
1. `show clock` で機器の現在時刻と NTP 同期状態を確認。
2. `show ip route [vrf]` で Syslog サーバーへの L3 経路が存在することを確認。
3. `show logging` で送信カウンター (`logging to ...`) が加算されているか確認。
4. ASA の場合は `logging enable` が投入されているか確認。
5. TLS Syslog の場合は Trustpoint 証明書の期限および SAN/CN を確認。
