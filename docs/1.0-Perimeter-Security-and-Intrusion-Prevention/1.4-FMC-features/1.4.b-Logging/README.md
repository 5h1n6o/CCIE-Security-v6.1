---
layout: default
title: 1.4.b-Logging
nav_order: 2
parent: 1.4-FMC-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.4.b Logging

Cisco Secure Firewall Management Center (FMC) における **Logging（ロギング機能）** は、管理下の Secure Firewall Threat Defense (FTD) および NGIPS アプライアンスが検出・処理した膨大なトラフィックデータ、脅威イベント（接続、侵入、ファイル/マルウェア、Security Intelligence）、ディスカバリデータ、システム監査ログ（Audit Log）、およびヘルス状態を包括的に収集・記録・分析・外部出力するための核となるセキュリティ可視化基盤です。

CCIE Security v6.1 ラボ試験では、単にログの有効化や表示方法を知るだけでなく、**ログの生成ポイント（FTD データプレーン vs FMC 管理プレーン）の明確な分離、開始時（Beginning）と終了時（End）ロギングの性能・データ粒度のトレーディングオフ、データベース上限（Database Event Limits）とイベントレート制御（Flow Rate Limit）、FTD Direct Syslog（Platform Settings）と FMC Alert Responses の設計的使い分け、ならびに eStreamer や Security Analytics and Logging (SAL) へのストリーミング統合** など、実践的かつ高度な構成・トラブルシューティング能力が多角的に問われます。

---

## 📘 概要

* **機能概要**:
  * **接続イベント（Connection Events）**: Access Control Policy (ACP)、Prefilter Policy、SSL Policy、DNS/URL/IP Blacklist (Security Intelligence) に合致したネットワークセッションの追跡・記録。
  * **セキュリティ脅威イベント（Security Events）**: Snort IPS インスペクションによる侵入イベント（Intrusion Events）、ファイル検出および AMP クラウド照会による File/Malware Events の生成。
  * **システムおよび運用監査ログ（Audit & Health Logs）**: FMC 上での管理操作履歴（Audit Log）および FTD/FMC のハードウェア・プロセス状態（Health Events）の収集。
  * **外部ストリーミング（External Logging）**: Syslog (UDP/TCP/EMBLEM)、SNMP Traps、eStreamer API、および Cisco Security Analytics and Logging (SAL) へのリアルタイムログ送信。
* **利用目的**:
  * セキュリティ侵害（Data Breach）発生時のフォレンジック調査、影響範囲特定、およびレトロスペクティブ分析（Retrospective Analysis）。
  * コンプライアンス遵守（PCI-DSS、HIPAA、GDPR 等）のための長期セッションログ・監査ログの外部アーカイブ保存。
  * ネットワークトラフィックのベースライン策定（Traffic Profiling）および SIEM / XDR プラットフォームでの自動相関・脅威検知。
* **どのような場面で利用するか**:
  * 全社境界セキュリティにおけるインターネット通過セッションの監視およびブロック理由の追跡。
  * 内部セグメント間通信における未承認プロトコルや P2P/マルウェア通信の検出・アラート出力。
  * 高トラフィック環境での FMC データベース処理負荷削減およびディスク枯渇を回避するための FTD 直接 Syslog 出力（Platform Settings）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **ログ分類** | Connection, Security Intelligence, Intrusion, File/Malware, Discovery, Health, Audit Logs |
| **ロギングタイミング** | **Log at Beginning of Connection** (接続開始時) vs **Log at End of Connection** (接続終了時) |
| **外部出力アーキテクチャ** | **FTD Direct Syslog** (Platform Settings 由来) vs **FMC Alert Responses** vs **eStreamer** |
| **Syslog フォーマット** | Standard RFC 5424 / RFC 3164 (PRI Header 含む) および Cisco EMBLEM フォーマット (UDP 限定) |
| **FMC データベース上限** | `System > Configuration > Database Event Limits` で種類ごとに保存イベント上限数を制限 |
| **性能制御 (Flow Rate)** | 優先度の低い Connection Events のローカル保存を `0` に設定して Flow Rate Limit の対象外化 |
| **対応プロトコル** | Syslog (UDP 514, TCP 514/SSL)、SNMP (v1/v2c/v3)、eStreamer (TCP 8302)、HTTP/HTTPS POST (Audit) |
| **設計上の注意点** | 単一ルールで Beginning と End の両方を選択（Double Logging）するとイベント量が倍増し性能を圧迫する |

---

## 🏗 動作原理

FMC における Logging アーキテクチャは、データプレーン（FTD/LINA/Snort）で生成されるトラフィックイベントと、管理プレーン（FMC/SFDataCorrelator）で相関・保存されるイベントデータの流れて構成されます。

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    Firepower Threat Defense (FTD) Data Plane                     │
│                                                                                  │
│  [ Ingress Packet ]                                                              │
│         │                                                                        │
│         ▼                                                                        │
│  ┌──────────────┐   Match Rule (Logging: Beginning/End)                          │
│  │ LINA / Snort ├─────────────────────────────────────────────┐                  │
│  └──────┬───────┘                                             │                  │
│         │                                                     │                  │
│         │ Generate Binary Event Data                          │ Direct Syslog    │
│         ▼                                                     ▼ (UDP/TCP 514)    │
│  ┌──────────────┐  sftunnel (TCP 8305: Control/Event)  ┌──────────────┐          │
│  │ EventHandler ├─────────────────────────────────────►│ External     │          │
│  └──────────────┘                                      │ Syslog / SIEM│          │
└────────────────────────────────────────────────────────┼──────────────┴──────────┘
                                                         │                         
                                                         │ Encrypted Event Stream  
                                                         ▼                         
┌──────────────────────────────────────────────────────────────────────────────────┐
│                      Firepower Management Center (FMC)                           │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │ [ FMC Processing & Analytics Engine ]                                      │  │
│  │   ├── SFDataCorrelator (Parses Binary Events & Correlates Discovery/Impact)  │  │
│  │   ├── Database (MySQL/Sybase: Event Storage according to Database Limits) │  │
│  │   ├── Unified Event Viewer / Connection Events (Analysis Menu)             │  │
│  │   └── External Alerting Engine (Alert Responses / eStreamer API)           │  │
│  └────────────────────────────────────┬───────────────────────────────────────┘  │
│                                       │                                          │
│                                       ├── eStreamer (TCP 8302) ──► Custom Client │
│                                       └── Syslog / SNMP / Email ─► External NMS  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

パケットが FTD を通過し、ログが最終的に FMC データベースまたは外部 Syslog サーバーへ到達するまでの処理順序は以下の通りです。

1. **トラフィックの受信とルールの評価**:
   * FTD の LINA および Snort 3 / Snort 2 インスペクションエンジンが着信パケットを処理し、Access Control Policy (ACP)、Prefilter、または SSL Policy のルールに照合します。
2. **ログ判定ポイントの評価 (Beginning vs End)**:
   * **`Log at Beginning of Connection`**: セッション開始時（SYN/SYN-ACK 確立時点または Block 実行時点）に、送信元/宛先 IP、ポート、Ingress/Egress ゾーンなどの初期情報のみでログエントリーを作成します。
   * **`Log at End of Connection`**: セッション終了時（FIN/RST 交換またはタイムアウト時）に、接続時間（`ConnectionDuration`）、双方の送信バイト数/パケット数、識別された L7 アプリケーション、URL カテゴリなどの完全なメタデータを含めてログエントリーを作成します。
3. **FTD 直接外部出力 (Direct Syslog: Platform Settings 経由)**:
   * ACP または Rule の Logging タブで `Use Threat Defense Platform Settings` が有効な場合、FTD の LINA システムログ機能が指定された外部 Syslog サーバー宛てに直接 UDP/TCP パケットを生成して送出します。
4. **FMC へのイベント転送 (`sftunnel`) および相関**:
   * イベント送信先に `Firepower Management Center` が指定されている場合、FTD 上の `EventHandler` プロセスがバイナリ形式でログをカプセル化し、管理通信トンネル `sftunnel` (TCP 8305) 経由で FMC へストリーミングします。
   * FMC 側の `SFDataCorrelator` プロセスがバイナリデータをデコードし、Host Discovery Map や Intrusion Impact Flag と自動照合・相関分析を行います。
5. **データベース格納と制限チェック (Database Event Limits)**:
   * FMC は指定された `Database Event Limits`（例: Connection Events 50M 件上限）を超過しないよう、古いイベントをローテーション消去（Silo Drain）しながらデータベース（MySQL/Sybase）へ格納します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Beginning vs End of Connection Logging の使い分け**:
   * **許可通信（Allow/Trust Action）**: 原則として **`Log at End of Connection` のみを有効化** します。開始時と終了時の両方を有効にするとログ量が倍増し（Double Logging）、FTD と FMC 間の帯域および FMC CPU を著しく圧迫して `Drain of unprocessed events` ヘルスアラートを引き起こします。
   * **拒否通信（Block/Block with reset Action）**: 拒否されたパケットはセッションが確立しないため、**`Log at Beginning of Connection` のみが選択可能** となります（通常、自動的に開始時ログとなります）。
   * **Interactive Block / Monitor Action**: ユーザーが警告を無視して通信を継続した場合や、Monitor ルールに合致した場合は、セッション全体の情報を追跡するため **`Log at End of Connection` が必須** となります。
2. **FTD Direct Syslog (Platform Settings) の設定手順と依存関係**:
   * FTD から直接外部 Syslog サーバーへセキュリティイベント（Connection, IPS, File/AMP）を出力させる場合、以下の **3 つの階層設定がすべて揃っている必要があります**:
     1. **Platform Settings**: `Devices > Platform Settings > Threat Defense > Syslog > Syslog Servers` で Syslog サーバー IP、プロトコル (UDP/TCP)、ポートを指定。
     2. **ACP 全体設定**: ACP の `More > Logging` タブで **`Use the syslog settings configured in the Threat Defense Platform Settings`** を選択し、必要に応じて `Send Syslog messages for File and Malware events` にチェック。
     3. **個別 Rule 設定**: ACP 内の各 Access Control Rule の `Logging` タブで、`Send Connection Events to: Syslog Server` を選択し、`Using default syslog configuration in Access Control Logging` になっていることを確認。
3. **EMBLEM フォーマットと UDP の関係**:
   * Platform Settings の `Logging Setup` タブで **`Send syslogs in EMBLEM format`** を有効化した場合、プロトコルは **UDP のみサポート** されます（TCP 指定時は EMBLEM フォーマット不可）。
   * EMBLEM フォーマットを有効にすると、ヘッダーに RFC 5424 互換の PRI (Priority) 値が含まれますが、`Syslog Device ID` は出力に含まれなくなります。
4. **FMC Database Event Limits による Flow Rate 制限の調整**:
   * 大規模トラフィック環境において、FMC の処理可能フロー数（Flow Rate Limit）を超過する場合、`System > Configuration > Database Event Limits` において **`Maximum Connection Events` の値を `0` に設定** します。
   * これにより、セキュリティ上重要でない低優先度の一般接続イベントの FMC ローカル DB 格納が停止され、ハードウェアの Flow Rate カウント制限対象外（Exempt）となります（Intrusion、File、Security Intelligence 関連の接続ログは継続して保持されます）。
5. **eStreamer と Syslog の機能的差異と使い分け**:
   * **Syslog**: 設定が極めて容易で標準的な SIEM (Splunk, QRadar 等) と即座に統合可能ですが、生パケットデータ（Packet Logs / Payload）、拡張メタデータ（LDAP 属性詳細、SSL 証明書 CN 詳細）、Discovery イベントは送信できません。
   * **eStreamer**: FMC から TCP ポート 8302 経由でバイナリストリーミングを行う独自プロトコル。パケットヘッダー/ペイロード、Extra Data、Discovery、ユーザーログイン、相関イベントを含めた完全なデータを外部にストリーミングできます。
6. **TCP Syslog 接続障害時のユーザーデータ通信制御**:
   * FTD から外部 Syslog サーバーへ TCP プロトコルで送信している際、Syslog サーバーがダウンすると、デフォルトで FTD はデータプレーンの新規トラフィックをブロック（遮断）する動作をとる場合があります。
   * これを防ぐため、Platform Settings の `Syslog > Syslog Settings` において **`Allow user traffic to pass when TCP syslog server is down`** チェックボックスを明示的に有効化しておく必要があります。

---

## 🛠 設定方法

### 1. FMC GUI: Access Control Policy における接続ロギング設定

```text
[ Step 1: ACP 全体ロギングの設定 ]
1. Policies > Access Control を開き、対象の Access Control Policy を編集。
2. 画面右上の "More" > "Logging" タブをクリック。
3. FTD 6.3 以降のベストプラクティス構成:
   - [✓] Use the syslog settings configured in the Threat Defense Platform Settings deployed on the device
   - [✓] Send Syslog messages for File and Malware events
   - Syslog Severity: Informational (6)
4. "Save" をクリック。

[ Step 2: 個別 Access Control Rule におけるロギング設定 ]
1. ACP 内の個別ルール（例: PERMIT_WEB）の編集画面を開く。
2. "Logging" タブを選択。
3. アクションが "Allow" または "Trust" の場合:
   - [ ] Log at Beginning of Connection (非チェック)
   - [✓] Log at End of Connection (チェック)
   - [✓] Log Files (ファイル/マルウェア検出ログの保持)
4. Send Connection Events to:
   - [✓] Firepower Management Center (FMC UI 表示用)
   - [✓] Syslog Server (Using default syslog configuration in Access Control Logging)
5. "Add" / "Save" をクリックし、変更を Deploy。
```

---

### 2. FMC GUI: FTD Direct Syslog (Platform Settings) 構成手順

```text
1. Devices > Platform Settings を開く。
2. 対象 FTD にバインドされている Threat Defense Settings ポリシーを編集。
3. 左ナビゲーションペインから "Syslog" を選択。

[ Sub-Menu 1: Syslog Servers ]
  - "Add" をクリック:
    - IP Address: 192.168.10.100 (Syslog サーバー IP)
    - Protocol: UDP (または TCP)
    - Port: 514
    - Log Messages in Cisco EMBLEM format: チェック (UDP 時のみ)
    - Select Interface: "Device Management Interface" または "Security Zones/Named Interfaces" (例: OUTSIDE_ZONE)

[ Sub-Menu 2: Syslog Settings ]
  - [✓] Enable Timestamp on Syslog Messages (Timestamp Format: Date and Time in MMM DD YYYY HH:MM:SS)
  - [✓] Enable Syslog Device ID (User Defined ID: FTD-DC01)
  - [✓] Allow user traffic to pass when TCP syslog server is down (障害時バッファリング/通過)

[ Sub-Menu 3: Logging Setup ]
  - [✓] Enable Logging
  - [✓] Enable Logging on the Failover Standby Unit
  - [✓] Send syslogs in EMBLEM format (全体適用時)
4. ポリシーを保存し、FTD へ Deploy。
```

---

### 3. FMC GUI: Database Event Limits によるローカル DB チューニング

```text
1. System > Configuration > Database Event Limits を開く。
2. イベント種類ごとの保持上限数をチューニング:
   - Maximum Connection Events: 0 (ローカル保存を無効化して外部 Syslog/SAL のみに委ねる場合)
   - Maximum Intrusion Events: 10,000,000
   - Maximum File Events: 10,000,000
   - Maximum Health Events: 100,000
3. "Save" をクリック。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI / FMC Bash) | 表示確認ポイント |
| :--- | :--- | :--- |
| **FTD 側の Syslog 配信設定および送信統計確認** | `show logging` | 送信先 IP、プロトコル、送信済みメッセージ数、ドロップ数、リンク状態 |
| **FMC 側の Data Correlator リアルタイム処理統計確認** | `sudo perfstats -Cq < /var/sf/rna/correlator-stats/now` | FMC が受信・パースしている 1 秒あたりのイベント数およびメモリ使用量 |
| **FTD 側のイベントドロップ・バッファ溢れ確認** | `show disk-manager` / `cat /ngfw/var/log/diskmanager.log` | イベントサイロ（Silo）の LWM / HWM 排水（Drain）ログと遅延情報 |
| **FTD 上での Snort パケットログ出力リアルタイムデバッグ** | `system support firewall-engine-debug` | 指定した IP/Port のパケットが生成した SOF (Start of Flow) / EOF (End of Flow) イベント |
| **`sftunnel` 管理・イベントチャネル接続状態の確認** | `sftunnel-status-brief` / `show manager` | Channel-B (EVENT) が `connected` 状態であること |
| **FMC 上でのシステムログリアルタイム追跡** | `sudo tail -f /var/log/messages` | FMC 内部プロセス (SFDataCorrelator, eStreamer 等) の動作ログ |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **FMC 上で `Drain of unprocessed events` ヘルスアラートが頻発する** | ACP ルールで開始時と終了時の両方をログ記録（Double Logging）している、またはイベント発生数が FMC 処理限界を超過 | `sudo perfstats -Cq`<br>`show disk-manager` | ACP ルールを修正して `Log at End of Connection` のみに統一し、必要に応じてローカル DB 保存上限数を削減。 |
| **FTD から外部 Syslog サーバーへセキュリティイベントが届かない** | Platform Settings の設定漏れ、または ACP ルールで `Use Threat Defense Platform Settings` を選択していない | `show logging` | Platform Settings で Syslog Server を定義し、ACP の Logging タブで `Use Threat Defense Platform Settings` を選択して Deploy。 |
| **Syslog サーバーの障害時に FTD 経由の通信が全停止する** | TCP プロトコルで Syslog を送信しており、`Allow user traffic to pass when TCP syslog server is down` が無効 | `show logging`<br>`show asp drops` | Platform Settings の Syslog Settings で TCP 障害時のパストラフィック許可にチェックを入れるか、UDP 送信に変更。 |
| **eStreamer クライアントへ接続イベントが送出されない** | `System > Integration > eStreamer` で `Connection Events` チェックボックスが無効化されている | FMC `System > Integration > eStreamer` | eStreamer Event Configuration で対象イベントタイプを有効化し、クライアント認証証明書を再作成。 |
| **HTTP アクセス時の URL 情報が接続ログに表示されない** | ACP の Advanced タブで `Limit URL Logging` が `0` (Disabled) に設定されている | FMC `Policies > Access Control > Advanced` | `Max URL Characters to Store` の設定値をデフォルト（2048 等）に引き上げる。 |

---

## ⚠ 制限事項

* **未確立 TCP セッションの非記録制約**:
  * TCP 3-way ハンドシェイクが正常完了しなかった接続（例: SYN Flood 攻撃、一方的 RST）は、DoS 攻撃による FMC データベース枯渇を防止するため、接続イベント（Connection Events）として記録されません。トラブルシューティングには `show asp drop` や Packet Capture を使用します。
* **同一ルール内での両方ロギング制約**:
  * 1 つの Access Control Rule で `Log at Beginning of Connection` と `Log at End of Connection` を同時に有効化すると、1 セッションあたり 2 つのログが生成され、システムパフォーマンスが大幅に低下します。
* **マルチインスタンス（FTD Container）における Syslog 送信インターフェイス**:
  * Firepower 4100 / 9300 上の FTD コンテナインスタンスにおいて、データプレーン Syslog をシャーシ管理ポートから直接出力させることはできません。必ず FTD のデータインターフェイスまたは Diagnostic インターフェイスを指定する必要があります。

---

## 🔄 他技術との関連

* **1.4.a Alerting**: FMC Alert Responses（Syslog、SNMP、Email）と FTD Direct Syslog の関係。相関ポリシー違反やヘルス異常は FMC Alerting 経由で送出され、データプレーン接続ログは FTD Direct Syslog で送信されます。
* **Cisco Security Analytics and Logging (SAL) / Secure Network Analytics (Stealthwatch)**: 大容量の FTD 接続ログを FMC ローカル DB ではなく外部の SAL クラウドまたはオンプレミス Secure Network Analytics アプライアンスへストリーミング保存し、FMC からクロスランチ（Cross-Launch）参照する構成。
* **Cisco ISE (Identity Services Engine)**: 接続イベント内にユーザー名、端末プロファイル（Endpoint Profile）、SGT (Security Group Tag)、認証ロケーション情報を紐付けて記録するための連携。

---

## 🧩 比較表

### 1. Beginning vs End of Connection Logging

| 比較項目 | Log at Beginning of Connection | Log at End of Connection |
| :--- | :--- | :--- |
| **記録タイミング** | セッション開始時（最初のパケット検出時） | セッション終了時（FIN/RST 交換または Timeout） |
| **取得可能データ** | 送信元/宛先 IP・Port、Zone、初期マッチルール | 左記全データ ＋ **セッション時間、送受信バイト/パケット数、URL、App、SGT** |
| **適用対象ルール** | **Block / Block with reset** アクションの通信 | **Allow / Trust / Interactive Block / Monitor** アクション |
| **システム負荷** | 比較的高頻度（即時生成） | 1 セッションあたり 1 ログで**効率的・標準推奨** |
| **制限事項** | セッション中の転送量や正確な L7 アプリ識別不可 | 正常終了しない切断セッションはタイムアウトまで保留 |

### 2. FTD Direct Syslog (Platform Settings) vs FMC Alert Responses vs eStreamer

| 比較項目 | FTD Direct Syslog | FMC Alert Responses | eStreamer API |
| :--- | :--- | :--- | :--- |
| **送信元** | **FTD (Data Plane / LINA)** | **FMC (Management Plane)** | **FMC (Management Plane)** |
| **送信プロトコル** | Syslog (UDP/TCP 514, EMBLEM) | Syslog / SNMP / Email (SMTP) | TLS Encrypted Stream (TCP 8302) |
| **対象イベント** | Connection, IPS, File/AMP, SI | Audit, Health, Correlation, Impact 1 | **全イベント (Raw Packet/Payload 含む)** |
| **FMC 負荷** | **極めて低い (FMC を完全バイパス)** | 中程度 (FMC でパース・生成) | 高い (FMC 外部ストリーミング) |
| **用途** | **SIEM への大容量接続ログ直接送信** | **管理者向け即時アラート通知** | 高度な XDR/SIEM へのフルコンテキスト連携 |

---

## 💡 ベストプラクティス

1. **接続ロギングの単一タイミング設定原則**:
   * 同一の Access Control Rule で `Log at Beginning of Connection` と `Log at End of Connection` を同時に有効化しないでください。原則として許可ルールには **`Log at End of Connection` のみを適用** します。
2. **大容量環境での FTD Direct Syslog（Platform Settings）採用**:
   * 秒間数千〜数万フローが発生する環境では、FMC ローカル DB への Connection Events 格納を回避（Database Event Limits で Max Connection Events を 0 に設定）し、**FTD Platform Settings 経由で外部 SIEM へ直接 Syslog 送信** します。
3. **TCP Syslog 障害時アクションの事前設定**:
   * TCP プロトコルで外部 Syslog サーバーへ送信する場合は、Syslog サーバー障害時の通信遮断トラブルを防ぐため、必ず `Allow user traffic to pass when TCP syslog server is down` を有効化します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Access Control Rule における End-of-Connection ロギングと FMC 送信構成

#### 問題
FMC において、内部 LAN (`10.1.1.0/24`) から外部 Web サーバー宛ての HTTP/HTTPS 許可ルール `ALLOW_WEB` を作成せよ。通信終了時にログを記録（`Log at End of Connection`）し、ファイル/マルウェア検出ログを含めて FMC イベントビューアへ送信するよう構成せよ。

#### 要件
* 接続開始時ロギングは無効化すること。
* `Log Files` を有効化すること。

#### 設定・操作手順

```text
1. Policies > Access Control を開き、該当ポリシーを編集。
2. Rule "ALLOW_WEB" の編集画面を開く。
   - Action: Allow
   - Source Networks: 10.1.1.0/24
   - Applications: HTTP, HTTPS
3. "Logging" タブを選択:
   - [ ] Log at Beginning of Connection (非チェック)
   - [✓] Log at End of Connection
   - [✓] Log Files
   - Send Connection Events to: [✓] Firepower Management Center
4. Rule を保存し、FTD へ Deploy。
```

#### 検証方法
* クライアント `10.1.1.10` から外部 Web サイトへアクセス。
* FMC 上で `Analysis > Connections > Events` を開き、該当通信の `Last Packet` タイムスタンプ、送信バイト数、および検出ファイル情報が表示されることを確認。

---

### 例2: FTD Direct Syslog (Platform Settings) による UDP 514 出力設定

#### 問題
FTD アプライアンスからのデータプレーンセキュリティイベントおよび接続ログを、FMC を経由させず FTD の OUTSIDE インターフェイスから直接 Syslog サーバー `192.168.10.100` (UDP 514) へ送信するよう Platform Settings を構成せよ。

#### 要件
* Platform Settings 名: `FTD_Platform_Policy`
* EMBLEM フォーマットを有効化すること。

#### 設定・操作手順

```text
1. Devices > Platform Settings を開き "FTD_Platform_Policy" を編集。
2. 左メニューから "Syslog" > "Syslog Servers" を選択し "Add" をクリック:
   - IP Address: 192.168.10.100
   - Protocol: UDP / Port: 514
   - Log Messages in Cisco EMBLEM format: [✓] チェック
   - Select Interface: OUTSIDE_ZONE (または Named Interface)
3. "Syslog Settings" を選択:
   - [✓] Enable Timestamp on Syslog Messages
   - [✓] Enable Syslog Device ID (User Defined ID: FTD-01)
4. "Logging Setup" を選択:
   - [✓] Enable Logging
   - [✓] Send syslogs in EMBLEM format
5. ポリシーを保存し、対象 FTD へ Deploy。
6. ACP の "More > Logging" タブを開き、"Use the syslog settings configured in the Threat Defense Platform Settings" を有効化して Deploy。
```

#### 検証方法
* FTD CLI にログインし、`show logging` を実行。`Logging to 192.168.10.100` および `EMBLEM` フォーマットが有効になっていることを確認。

---

### 例3: Block ルールにおける Beginning-of-Connection ロギング構成

#### 問題
内部ホストから既知の悪意ある C2 サーバー宛てトラフィックを遮断する ACP ルール `BLOCK_C2` を作成せよ。遮断された時点で即座にログを記録（`Log at Beginning of Connection`）し、FMC イベントビューアおよび Platform Settings 定義の Syslog サーバーへ送信せよ。

#### 要件
* アクション: `Block with reset`
* 接続開始時ロギングを有効化すること。

#### 設定・操作手順

```text
1. ACP 内で Rule "BLOCK_C2" を追加。
   - Action: Block with reset
   - Destination Networks: C2_Blacklist_IPs
2. "Logging" タブを選択:
   - [✓] Log at Beginning of Connection (Block のため自動適用)
   - Send Connection Events to:
     - [✓] Firepower Management Center
     - [✓] Syslog Server (Using default syslog configuration)
3. Rule を保存し、FTD へ Deploy。
```

#### 検証方法
* C2 対象 IP 宛てに接続を試み、即座に TCP RST が返されて通信が遮断されることを確認。
* FMC の `Analysis > Connections > Events` で Action が `Block with reset` の接続イベントが即座に記録されていることを確認。

---

### 例4: TCP Syslog サーバー冗長化およびダウン時パストラフィック許可構成

#### 問題
FTD から 2 台の外部 Syslog サーバー (`192.168.10.101`, `192.168.10.102`: TCP 514) へログを出力するよう構成せよ。両 Syslog サーバーが完全にダウンした場合でも、FTD のユーザーデータ通信がドロップせず通過（Bypass）するように設定せよ。

#### 要件
* プロトコル: TCP 514
* TCP サーバーダウン時の通過許可オプションを有効にすること。

#### 設定・操作手順

```text
1. Platform Settings > Syslog > Syslog Servers を開く。
2. 2 台の Syslog サーバーを追加:
   - Server 1: 192.168.10.101 / TCP / 514
   - Server 2: 192.168.10.102 / TCP / 514
3. Syslog Settings タブを開く:
   - [✓] Allow user traffic to pass when TCP syslog server is down
4. 設定を保存し、FTD へ Deploy。
```

#### 検証方法
* FTD CLI で `show logging` を実行し、両 TCP ポートへの接続が確立していることを確認。
* テスト環境で Syslog サーバーを停止させた際、データプレーンの通信が維持され、`show asp drops` で Syslog 起因のドロップが発生しないことを確認。

---

### 例5: Database Event Limits によるローカル Connection Events 保存の停止 (Exempt Flow Rate)

#### 問題
FMC の CPU 処理負荷およびディスク使用量を削減するため、一般接続イベント（低優先度フロー）の FMC ローカルデータベースへの保存を停止（上限 `0`）せよ。ただし、IPS、ファイル、マルウェアに関連するセキュリティ接続イベントは自動的に保持されることを確認せよ。

#### 要件
* `System > Configuration > Database Event Limits` を使用すること。

#### 設定・操作手順

```text
1. FMC UI で System > Configuration > Database Event Limits に移動。
2. Connection Events リストの "Maximum Connection Events" の設定値を "0" に変更。
3. "Save" をクリック。
```

#### 検証方法
* 通常の Web アクセスログが FMC の `Analysis > Connections > Events` に新規保存されなくなることを確認。
* テスト攻撃（Intrusion イベント発生通信）を実行した際、該当セッションに関連する接続ログ（High Priority Connection Event）のみが自動的に保存されることを確認。

---

### 例6: URL フィルタリングログにおける文字数制限 (Max URL Characters) のカスタマイズ

#### 問題
Web アクセスログに記録される URL の最大文字数をデフォルトから `512` 文字に制限し、データベース容量の節約と検索パフォーマンスの向上を図れ。

#### 要件
* ACP の Advanced タブで設定すること。

#### 設定・操作手順

```text
1. Policies > Access Control を開き、対象 ACP を編集。
2. "Advanced" タブを選択。
3. "General Settings" セクションの "Max URL Characters to Store" を表示。
4. 値を "512" に変更して保存。
5. FTD へ Deploy。
```

#### 検証方法
* 512 文字を超える長い URL パスを持つ Web サイトへアクセス。
* FMC の Connection Events 画面で、URL フィールドが 512 文字目で切り捨て（Truncate）されて記録されていることを確認。

---

### 例7: FMC Audit Log の外部 Syslog ストリーミング構成

#### 問題
FMC 上で管理者が行った設定変更、ログイン/ログアウト、ポリシー展開などの管理操作履歴（Audit Log）を、外部の監査用 Syslog サーバー (`192.168.10.200:514`, Facility: `AUDIT`) へリアルタイムストリーミング出力せよ。

#### 要件
* `System > Configuration > Audit Log` で設定すること。

#### 設定・操作手順

```text
1. System > Configuration > Audit Log を開く。
2. "Stream Audit Logs to Syslog" オプションを有効化:
   - Host: 192.168.10.200
   - Port: 514
   - Facility: AUDIT (または AUTHPRIV)
3. "Save" をクリック。
```

#### 検証方法
* FMC 上でオブジェクトやポリシーの変更操作を実行し、保存する。
* Syslog サーバー `192.168.10.200` 上で、操作ユーザー名、送信元 IP、実行アクションを含む Audit メッセージ（`Sender: admin@192.168.10.50 ...`）を受信していることを確認。

---

### 例8: eStreamer API による外部 SIEM へのフルイベントストリーミング有効化

#### 問題
FMC 上で eStreamer サービスを有効化し、外部 SIEM クライアント宛てに Intrusion Events、Intrusion Event Packet Data (生パケット)、Connection Events、および Discovery Events のストリーミング許可を設定せよ。

#### 要件
* `System > Integration > eStreamer` を使用すること。

#### 設定・操作手順

```text
1. System > Integration > eStreamer タブを開く。
2. eStreamer Event Configuration セクションで以下をチェック:
   - [✓] Intrusion Events
   - [✓] Intrusion Event Packet Data
   - [✓] Connection Events
   - [✓] Discovery Events
3. "Save" をクリック。
4. "Create Client" をクリックし、SIEM サーバー IP (192.168.10.250) および Password を入力して PKCS12 証明書を生成・ダウンロード。
```

#### 検証方法
* FMC CLI / Bash で `sudo netstat -natp | grep 8302` を実行し、eStreamer デーモン (`sfestreamer`) が TCP ポート 8302 で Listen していることを確認。

---

### 例9: SSL Policy における暗号化通信の End-of-Connection ロギング構成

#### 問題
SSL Policy `CUSTOM_SSL_POLICY` 内のルール `DECRYPT_HTTPS` において、復号化対象（Decrypt - Resign）となる SSL/TLS セッションの終了時ログ（`Log at End of Connection`）を収集するよう設定せよ。

#### 要件
* `Policies > SSL` 内で構成すること。

#### 設定・操作手順

```text
1. Policies > SSL を開き "CUSTOM_SSL_POLICY" を編集。
2. Rule "DECRYPT_HTTPS" の編集アイコンをクリック。
3. "Logging" タブを選択。
   - [✓] Log at End of Connection
   - Send Connection Events to: [✓] Firepower Management Center
4. Rule を保存し、ACP の SSL Policy オプションにバインドして Deploy。
```

#### 検証方法
* SSL 復号対象の HTTPS アクセスを実行。
* FMC の Connection Events 画面で `SSL Status` が `Decrypt (Resign)` となり、`SSL Cipher Suite` や `SSL Version` がログに記録されていることを確認。

---

### 例10: Prefilter Policy Fastpath ルールのロギング構成

#### 問題
FTD に適用する Prefilter Policy において、大容量バックアップ通信（10.1.1.200 ➔ 198.51.100.200）を Snort DPI の手前で Fastpath 処理するルールを作成し、当該 Fastpath セッションの終了時ログを記録せよ。

#### 要件
* Prefilter Policy 内でロギングを設定すること。

#### 設定・操作手順

```text
1. Policies > Access Control > Prefilter を開き、対象ポリシーを編集。
2. Prefilter Rule "FASTPATH_BACKUP" を追加:
   - Action: Fastpath
   - Source IP: 10.1.1.200
   - Destination IP: 198.51.100.200
3. "Logging" タブを選択:
   - [✓] Log at End of Connection
   - Send Connection Events to: [✓] Firepower Management Center
4. Rule を保存し、ACP の Advanced タブでバインドして Deploy。
```

#### 検証方法
* バックアップ通信を発生させ、FMC の Connection Events で `Action` が `Fastpath` と表示され、Snort DPI をバイパスしたログが正常に記録されることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で Access Control Policy 内の主要な Allow ルールに対し、`Log at Beginning of Connection` と `Log at End of Connection` の両方を有効化して Deploy 実行した。その後、FMC 上で `Drain of unprocessed events` の Critical ヘルスアラートが頻発し始めた。原因と適切な修正手順として正しいものはどれか。

* A) FTD と FMC 間の sftunnel ポート (TCP 8305) が非ブロック状態になっているため。sftunnel を再起動する。
* B) 単一セッションに対して重複ログ（Double Logging）が発生し、FMC の処理限界を超過したため。ルール設定を変更し `Log at End of Connection` のみを有効化する。
* C) FTD 上で `ip nbar protocol-discovery` が無効化されているため。
* D) eStreamer サービスが停止しているため。

**正解**: **B**
**解説**: 許可ルールに対して開始時と終了時の両方でログを有効化すると、1 セッションあたり 2 つのイベントが生成されてイベント処理パイプライン（EventHandler ➔ sftunnel ➔ SFDataCorrelator）が過小になり、`Drain of unprocessed events` アラートが発生します。原則として `Log at End of Connection` のみを有効化します。

---

### 2. トラブルシュート問題

**問題**: FTD アプライアンスから外部 Syslog サーバー (`192.168.10.100`: TCP 514) へセキュリティイベントを直接出力するよう Platform Settings を構成した。しかし、保守作業で Syslog サーバーがダウンした際、FTD を通過するすべてのユーザーデータ通信が途絶した。原因と直ちにとるべき対策コマンド/設定はどれか。

* A) FTD の LINA エンジンがフリーズしたため、FTD を再起動する。
* B) TCP Syslog サーバー不達時にトラフィックをドロップする安全仕様が働いている。Platform Settings の Syslog Settings で `Allow user traffic to pass when TCP syslog server is down` を有効化する。
* C) UDP 514 に切り替えるため、FTD CLI で `no logging on` を投入する。
* D) FTD の管理ポート IP アドレスを変更する。

**正解**: **B**
**解説**: FTD から TCP プロトコルで Syslog 送信する設定において、`Allow user traffic to pass when TCP syslog server is down` がオフの場合、Syslog サーバー不達時にバッファが満杯になるとデータプレーン通信をブロックします。該当チェックボックスを有効化することで通信障害を回避できます。

---

### 3. Design 問題

**問題**: 毎秒 100,000 フローを超える超高トラフィック環境において、FMC (FMCv) を導入して FTD を集中管理する。FMC アプライアンスのハードウェア Flow Rate Limit 超過およびディスク満杯を防ぎつつ、セキュリティ侵入イベント (IPS) および外部 SIEM への全セッションログ保存要件を両立させるための最適な設計方針はどれか。

* A) すべての ACP ルールでロギングを完全無効化する。
* B) FMC の `Database Event Limits` で `Maximum Connection Events` を `0` に設定して一般接続ログの FMC ローカル保存を停止し、FTD Platform Settings 経由で全セキュリティイベントを外部 SIEM へ直接 Syslog 送信（または SAL へストリーミング）する。
* C) 全通信に対して eStreamer のみを使用して生パケットを FMC ローカルに保存する。
* D) `Log at Beginning of Connection` のみ全ルールで有効化する。

**正解**: **B**
**解説**: FMC の `Maximum Connection Events` を `0` に設定すると、一般の低優先度接続ログが FMC ローカル DB に保存されなくなり、Flow Rate カウント制限の対象外となります（IPS/File 関連の重要セッションは保持される）。同時に FTD Platform Settings 経由で直接外部 SIEM へ Syslog 送信することで、FMC に負荷をかけずに全セッションログをアーカイブできます。

---

### 4. 実装問題

**問題**: Managed Device (FTD) から直接外部 Syslog サーバーへ UDP プロトコルで送信する際、Cisco 独自の EMBLEM フォーマットを有効化した。生成される Syslog メッセージに関する記述として正しいものはどれか。

* A) メッセージヘッダーに RFC 5424 互換の PRI (Priority) 値が含まれ、Syslog Device ID は出力されなくなる。
* B) 送信プロトコルとして TCP 514 が強制使用される。
* C) 生のパケットペイロード（Packet Payload）が暗号化されて添付される。
* D) FMC の管理 IP アドレスが送信元 IP として書き換えられる。

**正解**: **A**
**解説**: Cisco EMBLEM フォーマットは UDP 送信時のみ利用可能で、ヘッダーに Facility と Severity を表す PRI 値 (`<PRI>`) が付与されますが、`Syslog Device ID` オプションを有効にしていても Device ID は出力に含まれなくなります。

---

### 5. コンフィグ読解問題

**問題**: 以下の図は FMC の Access Control Policy における、ある個別の Access Control Rule の `Logging` タブ設定画面である。この設定に関する動作説明として正しいものはどれか。

```text
[ Logging Tab Settings ]
[ ] Log at Beginning of Connection
[✓] Log at End of Connection
[✓] Log Files

Send Connection Events to:
  [✓] Firepower Management Center
  [✓] Syslog Server (Using default syslog configuration in Access Control Logging)
```

* A) 接続開始時と終了時の両方でログが FMC と Syslog サーバーへ二重送信される。
* B) 接続終了時にセッション全体メタデータ（バイト数、時間、アプリケーション、検出ファイル）が FMC データベースおよび ACP で指定されたデフォルト Syslog サーバーへ送信される。
* C) 接続ログは外部 Syslog サーバーのみに送信され、FMC のイベントビューアには一切表示されない。
* D) 接続ログは生成されず、ファイルイベントのみが記録される。

**正解**: **B**
**解説**: `Log at End of Connection` のみがチェックされているため、セッション終了時に完全なメタデータを持つログが生成され、`Firepower Management Center` (FMC UI 表示用) および `Syslog Server` (ACP 設定継承) の両方へ正常に送出されます。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Administration Guide 7.1 - Connection Logging](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center Administration Guide 7.1 - About Sending Syslog Messages for Security Events](https://www.cisco.com/)
* [Cisco Secure Firewall Threat Defense Device Configuration Guide with FMC 7.1 - Configure Syslog Logging for FTD Devices](https://www.cisco.com/)
* [Cisco Technical Notes: Troubleshoot Drain of FMC Unprocessed Events and High Disk Usage](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Cisco Secure Firewall Eventing and Logging Architectures (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * ラボ試験で「Configure FTD to log events directly to Syslog」と要求された場合は **Platform Settings**、「Configure FMC to alert via Syslog」と要求された場合は **Alert Responses**、「Configure Access Control logging」と要求された場合は **ACP Rules / Policy Logging** を選択する、という明確な役割分担を常に意識してください。
* **図解（Logging パイプラインとイベントフロー概観）**:
  ```text
  [ Traffic Traversing FTD ]
             │
             ├── Match Access Control Rule (Action: Allow)
             │      │
             │      ├── [ Log at End of Connection Enabled ]
             │      │          │
             │      │          ├── Output option: Platform Settings Syslog ──► Direct to SIEM (UDP/TCP 514)
             │      │          │
             │      │          └── Output option: FMC Event Viewer ─────────► sftunnel (TCP 8305) ──► FMC DB
             │      │
             │      └── [ Log Files Enabled ] ──────────────────────────────► AMP Cloud Check ──► FMC DB
             │
             └── Match Security Intelligence / IPS / File Block Rule
                    │
                    └── [ Log at Beginning of Connection ] ─────────────────► Instant Drop Log ──► FMC DB / Syslog
  ```
* **注意点**:
  * ロギング設定を更新した後は、必ず対象 FTD アプライアンスへ **`Deploy` を実行** しないと、FTD 上の LINA コンフィグや Snort ログインスタンスに変更が反映されません。
