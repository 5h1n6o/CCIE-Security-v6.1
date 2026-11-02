---
layout: default
title: 1.12-Correlation-and-remediation
nav_order: 12
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.12 Correlation and remediation rules on Cisco FMC

Cisco ネットワークセキュリティアーキテクチャにおいて、**Firepower Management Center (FMC) Correlation Policy（相関ポリシー）** および **Remediation Rules（修復ルール）** は、NGFW/NGIPS（Cisco Secure Firewall Threat Defense: FTD）、Network Discovery（ネットワーク検出）、AMP（Advanced Malware Protection）、および外部 ID/アクセス制御基盤（Cisco ISE）から収集される多次元のセキュリティイベントをリアルタイムに集約・分析（相関分析）し、脅威に対して自動的かつ即時的な防御・隔離措置（Automated Incident Response / Rapid Threat Containment: RTC）を自動発動させる高次元なセキュリティ自動化エンジンです。

CCIE Security v6.1 Blueprint **1.12 Correlation and remediation rules on Cisco FMC** では、単なるログ監視を超えて、**イベントトリガー（Intrusion、Malware、Connection、Discovery、Host Input、User Activity、VPN Troubleshooting）の高度な定義**、**インパクトフラグ (Impact Level 1〜4) やホストプロファイル属性に基づく相関ルールの作成**、**Cisco ISE pxGrid (Adaptive Network Control: ANC / Endpoint Protection Services: EPS) による自動クアランティン (Quarantine/Shutdown)**、**Cisco IOS Null Route による自動ルーターブロック**、**Nmap スキャンによる自動詳細リコンサイナンス**、および **Remediation Status Events の検証・トラブルシューティング能力** が厳密に評価されます。

---

## 📘 概要

* **機能概要**:
  * **FMC Correlator (SFDataCorrelator)**: FMC 上のバックグラウンドプロセスであり、FMC がデータプレーン（FTD）やエンドポイント（AMP for Endpoints）、ID サーバー（ISE）、ネットワーク自動検出（Network Discovery）から取得した各種イベントを統合。事前に定義された **Correlation Rules（相関ルール）** や **Compliance Allow Lists（コンプライアンス許可リスト）** と照合します。
  * **Correlation Policy（相関ポリシー）**: 1つ以上の相関ルールやコンプライアンス許可リストを束ね、条件合致時に実行する **Responses（応答：アラート通知や修復アクション）** を割り当てて有効化 (Activate) するポリシーコンテナです。
  * **Remediation Module ＆ Instance（修復モジュール ＆ インスタンス）**: 脅威検知時に外部システムへアクティブな制御コマンドを発行するプラグイン機構。
    * **Cisco ISE pxGrid / ANC Mitigation Module**: pxGrid 経由で Cisco ISE へ命令を送り、対象端末の ANC ポートクアランティン（Quarantine）、ポートシャットダウン（Shutdown）、またはクアランティン解除（Unquarantine）を実行。
    * **Cisco IOS Null Route Module**: Telnet/SSH 経由で Cisco IOS / IOS-XE ルーターへ自動ログインし、対象 IP アドレスに対する <code>ip route &lt;IP&gt; 255.255.255.255 Null0</code> を動的設定してトラフィックをブラックホール化。
    * **Nmap Scanning Module**: 攻撃対象または攻撃元 IP に対して自動的に Nmap アクティブスキャンを発動し、OS やオープンポート情報を収集して Host Profile を自動更新。
    * **Set Attribute Value Module**: FMC 内の Host Attribute（例: `Host Criticality`, `Custom Status`）の値を自動変更。
* **利用目的**:
  * SOC（Security Operations Center）アナリストの手動介入を待つことなく、高深刻度（Impact Level 1）の侵入やマルウェア感染が検知された瞬間（ミリ秒〜秒単位）に、ネットワークアクセスを自動遮断（Rapid Threat Containment）して被害の横展開（Lateral Movement）を防止する。
  * 企業内のセキュリティポリシー違反（未承認の HTTP サーバー起動、許可されていない OS/プロトコルの検知、未暗号化通信の発生）に対し、自動でアラート発信や修復プログラムをトリガーする。
* **どのような場面で利用するか**:
  * ネットワーク境界でインバウンド攻撃が内部の脆弱なサーバーに到達（Impact Level 1: Red）した際の即時 ISE ポート隔離。
  * エンドポイント（AMP for Endpoints）でマルウェア隔離に失敗（Quarantine Failure）した場合のネットワークレベルでの即時クアランティン。
  * 社内ポリシーで禁止されている P2P / ファイル共有プロトコルや未承認 WEB サーバーが起動した際の自動検知・警告。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要コンポーネント** | **Correlation Policy**、**Correlation Rules**、**Compliance Allow Lists**、**Traffic Profiles**、**Remediation Modules / Instances** |
| **主要ベースイベント型** | Intrusion Event、Malware Event、Connection Event、Discovery Event、Host Input Event、User Activity Event、VPN Troubleshoot Event、Traffic Profile Changes |
| **評価条件の構成要素** | イベントタイプ固有条件 ＋ **Host Profile Qualification**（OS/App/Port）＋ **User Qualification**（ユーザー/グループ）＋ **Connection Trackers** |
| **Impact Assessment** | **Impact Level 1 (Vulnerable - Red)**、Impact Level 2 (Potentially Vulnerable)、Impact Level 3 (Currently Not Vulnerable)、Impact Level 4 (Unknown Target)、Impact Level 0 (Unknown) |
| **主要修復モジュール** | **Cisco ISE pxGrid Mitigation (ANC/EPS)**、**Cisco IOS Null Route**、**Nmap Scan**、**Set Attribute Value** |
| **ISE 連携プロトコル** | **pxGrid (Platform Exchange Grid)** (XMPP / WebSocket ベースのコンテキスト共有 ＆ 制御) |
| **ライセンス要件** | **Threat License** (旧 Protection) または **AMP License** (関連イベント使用時)、FMC 管理機能 |
| **制限事項** | 相関ルールは相関ポリシーにアサインして **"Activate"（有効化）** しない限り評価・実行されない。また、最初の評価（初期ロード時）ではコンプライアンス違反イベントは一括生成されない。 |

---

## 🏗 動作原理

### 1. FMC Correlation ＆ Automated Remediation 処理パイプライン

FTD や AMP、Discovery がイベントを検出してから、FMC Correlator が相関ルールを評価し、Cisco ISE や IOS ルーターへ修復アクション（Remediation）を発行する一連のデータフロー：

```text
[ Managed FTD / AMP / Discovery ] ───(1) Event Data (Intrusion / Malware / Connection) ───► [ FMC Management Engine ]
                                                                                                  │
                                                                                                  ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ FMC SFDataCorrelator Process                                                                                   │
│                                                                                                                 │
│ 2. Impact Assessment & Host Profile Correlation:                                                               │
│    ├── Matches Intrusion Event against Targeted Host's Vulnerability Map                                       │
│    └── Assigns Impact Level (e.g., Impact 1: Vulnerable Red)                                                   │
│                                                                                                                 │
│ 3. Correlation Policy Evaluation:                                                                               │
│    ├── Check Active Correlation Policies & Rules                                                               │
│    ├── Evaluate Event Conditions (e.g., Event = Intrusion AND Impact = 1 AND Host OS = Windows)                │
│    └── Match Confirmed! ──► Trigger Correlation Event                                                          │
│                                                                                                                 │
│ 4. Response & Remediation Execution:                                                                           │
│    ├── Trigger Alerts (Syslog / SNMP / Email)                                                                  │
│    └── Launch Configured Remediation Instance                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                                │
                 ┌──────────────────────────────────────────────┴──────────────────────────────────────────────┐
                 │ (pxGrid XML/JSON API)                                                                      │ (Telnet/SSH)
                 ▼                                                                                            ▼
    [ Cisco ISE (ANC / EPS) ]                                                                      [ Cisco IOS Router ]
                 │                                                                                            │
   Sends CoA (Change of Authorization)                                                           Applies Null Route:
   to Access Switch Port                                                                         `ip route <IP> 255.255.255.255 Null0`
                 │                                                                                            │
                 ▼                                                                                            ▼
[ Endpoint Placed in Quarantine / Port Shutdown ]                                               [ Target IP Traffic Blackholed ]
```

---

### 2. Impact Assessment (インパクトレベル判定) メカニズム

FMC は単なるシグネチャマッチ（Intrusion Event）に留まらず、Network Discovery データベース（RNA）に記録された「攻撃対象ホストの OS、実行サービス、未修正脆弱性 (CVE)」と自動突き合わせを行い、以下の 5 段階の Impact Level を付与します。相関ルールではこの Impact Level を条件として使用できます。

```text
  Intrusion Event Triggered (e.g., Apache Struts Exploit SID:24599)
                            │
                            ▼
              Target Host Network Map Lookup
                            │
            ┌───────────────┴───────────────┐
            │ Target Host Details Found?    │
            └───────────────┬───────────────┘
                     YES    │    NO
             ┌──────────────┴───────────────┐
             │                              │
             ▼                              ▼
  Server Running Apache?             Impact Level 4
    │                                (Unknown Target / No Host Data)
    ├── YES ──► Vulnerable to CVE?
    │               ├── YES ──► Impact Level 1 (Vulnerable - RED) [最高警戒]
    │               └── NO  ──► Impact Level 3 (Currently Not Vulnerable - Yellow)
    │
    └── NO  ──► Target OS/App Mismatch ──► Impact Level 2 (Potentially Vulnerable / Mismatch - Orange)
```

---

## ⚙ 動作シーケンス

1. **イベント検知 ＆ FMC 転送フェーズ**:
   * FTD アプライアンスが Access Control / Intrusion Policy に基づき侵入イベント（Intrusion Event）を検知、または AMP for Endpoints がマルウェア検知失敗イベントを発生。
   * イベントデータが FMC の `SFDataCorrelator` プロセスへリアルタイム送信される。
2. **コンテキスト相関 ＆ インパクトレベル決定フェーズ**:
   * FMC はターゲット IP アドレスに基づいて Host Profile（Network Map）をロード。
   * ターゲットの OS、オープンポート、稼働サービス、および脆弱性（NVD/CVE マッピング）をチェックし、**Impact Level 1（赤: 脆弱性あり）** や **IOC Tag（Indication of Compromise）** を動的割り当て。
3. **Correlation Rule 評価フェーズ**:
   * `SFDataCorrelator` がアクティブ化されている **Correlation Policy** 内の **Correlation Rules** を上から順に評価。
   * 条件式（例: `If Intrusion Event occurs AND Impact Flag is 1 - Red AND Target Host OS is Windows`）を判定。
4. **Remediation アクション発動 ＆ 外部連携フェーズ**:
   * ルールが合致（Violation）した場合、設定された **Response**（Syslog/Email/SNMP アラートおよび Remediation Instance）を起動。
   * **Cisco ISE pxGrid 連携時**: FMC は pxGrid コントローラー（XMPP/WebSocket）を経由して ISE へ `ANC Quarantine` セッションを発行。ISE は RADIUS CoA (Change of Authorization) を対象アクセススイッチへ送信し、端末の VLAN / dACL をクアランティン用に動的変更。
   * **Cisco IOS Null Route 連携時**: FMC は指定されたルーターへ Telnet/SSH 接続し、CLI で `ip route <IP> 255.255.255.255 Null0` を投入。
5. **Remediation Status Event 記録フェーズ**:
   * FMC は修復プログラムの実行結果（成功・失敗、リターンコード、メッセージ）を `Analysis > Correlation > Remediation Status` ログテーブルに永続記録。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Correlation Policy の活性化 (Activation) の絶対要件**:
   * 相関ルール（Rule）や修復インスタンス（Remediation Instance）を完璧に作成しても、**Policies > Correlation > Policy Management** 画面でルールをポリシーに紐付け、右側のスイッチを **"Active"**（緑色）にし、**Save** を押さなければ 1% も動作しません。試験で最も多い失点原因です。
2. **Cisco ISE pxGrid Mitigation (ANC) の構成手順**:
   * **ISE 側前提条件**: ISE で `pxGrid` サービスおよび `Adaptive Network Control (ANC)` 機能を有効化。クアランティン用の ANC Policy（例: `Quarantine_Policy` ➔ ANC Action: `QUARANTINE`）を作成しておくこと。
   * **FMC 側接続要件**: FMC の `System > Integration > Identity Services Engine` で ISE PSN/MNT と pxGrid 接続（Client Certificate 承認完了）を維持。
   * **Remediation Instance の作成**: `Policies > Actions > Modules` で `pxGrid Mitigation` の magnifying glass アイコンをクリックし、Instance（例: `ISE_Instance`）を作成。さらに Instance 内で `Add Remediation` を選択し、Mitigation Action を `Quarantine` に設定。
3. **Cisco IOS Null Route Remediation の構文と制限**:
   * FMC は Telnet/SSH でルーターにログインするため、ルーター側で <code>username &lt;USER&gt; privilege 15 secret &lt;PASS&gt;</code>、<code>line vty</code> での認証設定、および <code>ip route</code> の権限が必要です。
   * **Source IP vs Destination IP の使い分け**:
     * 内部ホストが攻撃を受けた場合、攻撃元を破棄するには **Block Source IP**。
     * 内部ホストが外部 C2 に感染した場合、C2 へのアクセスを破棄するには **Block Destination IP**。
4. **Compliance Allow Lists（コンプライアンス許可リスト）の設計**:
   * ネットワークセグメント（Target Network）ごとに「許可する OS・アプリプロトコル・Web アプリ・クライアント」を定義。
   * リスト外の未知のサービス（例: 未承認の Telnet や HTTP サーバー）が起動した瞬間、`Allow List Event` を発動させて修復アクションをトリガーする。
5. **Impact Level 1 (Vulnerable) 条件の作り方**:
   * 相関ルールの条件で `an intrusion event occurs` を選択し、`Add Condition` ➔ **`Impact Flag` `is` `1 - red (Vulnerable)`** を指定します。これが「攻撃が成功する可能性が非常に高いイベント」のみを厳選して修復を発動させるベストプラクティス構成です。

---

## 🛠 設定方法

### 1. FMC GUI: Cisco ISE pxGrid (ANC) 自動クアランティン修復の設定完全手順

```text
[ Step 1: pxGrid Remediation Instance の作成 ]
1. FMC UI で Policies > Actions > Modules に移動。
2. "pxGrid Mitigation" モジュールの右側にある Magnifying Glass (Search/Edit) アイコンをクリック。
3. "Add Instance" をクリック:
   - Instance Name: ISE_PXGRID_INSTANCE
   - Description: FMC to ISE Rapid Threat Containment
4. 作成した Instance の設定を開き "Add Remediation" をクリック:
   - Remediation Name: REMEDIATE_QUARANTINE_SOURCE
   - Mitigation Action: Quarantine
   - Configured IP: Source IP (または Destination IP)
   - Save をクリック。

[ Step 2: Correlation Rule の作成 ]
1. Policies > Correlation > Rule Management に移動し "Create Rule" をクリック。
2. Rule Information:
   - Rule Name: RULE_HIGH_IMPACT_INTRUSION
3. Event Selection:
   - Select the type of event: "an intrusion event occurs"
4. Add Condition:
   - "Impact Flag" "is" "1 - red (Vulnerable)"
5. (オプション) Add Complex Condition / Host Qualification:
   - Target Host Operating System is Windows
6. Save をクリック。

[ Step 3: Correlation Policy の作成 ＆ Responses アサイン ＆ 有効化 ]
1. Policies > Correlation > Policy Management に移動し "Create Policy" をクリック。
2. Policy Name: POLICY_RTC_AUTOMATION
3. "Add Rules" をクリックし "RULE_HIGH_IMPACT_INTRUSION" をアサイン。
4. アサインされたルールの右側にある "Responses" (ノートアイコン) をクリック:
   - Unassigned Responses 一覧から "REMEDIATE_QUARANTINE_SOURCE" を選択し Assigned Responses へ移動。
   - Update ➔ Save をクリック。
5. Policy Management 一覧画面に戻り、"POLICY_RTC_AUTOMATION" の右側にあるトグルスイッチを "Active" (緑) に変更して Save。
```

---

### 2. FMC GUI: Cisco IOS Null Route 自動ブロックの設定手順

```text
[ Step 1: Cisco IOS Null Route Remediation Instance の作成 ]
1. Policies > Actions > Modules ➔ "Cisco IOS Null Route" モジュールを選択。
2. "Add Instance" をクリック:
   - Instance Name: ROUTER_EDGE_01
   - Router IP: 10.1.1.1
   - Protocol: SSH (または Telnet)
   - User Name: admin
   - Password: CiscoPassword123!
   - Enable Password: CiscoEnable123!
3. 作成した Instance 内で "Add Remediation":
   - Name: REMEDIATE_NULL_ROUTE_ATTACKER
   - Action: Block Source (または Block Destination)
   - Subnet Mask: 255.255.255.255 (/32 ホストブロック)
   - Save をクリック。

[ Step 2: Correlation Policy への紐付け ＆ 有効化 ]
1. Policies > Correlation > Policy Management で対象ポリシーを編集。
2. 作成したルールに対して Responses として "REMEDIATE_NULL_ROUTE_ATTACKER" をアサイン。
3. ポリシーを "Active" にして Save。
```

---

### 3. FMC GUI: Compliance Allow List（許可リスト違反検知）の設定手順

```text
[ Step 1: Compliance Allow List の定義 ]
1. Policies > Correlation > Allow List タブに移動し "Add Allow List" をクリック。
2. Name: ALLOW_LIST_SERVER_FARM
3. Target Networks: 172.16.1.0/24 (DMZ Server Network)
4. Allowed Host Profiles 構成:
   - Operating System: Linux / Windows Server
   - Globally Allowed Application Protocols: HTTP, HTTPS, SSH
5. Save をクリック。

[ Step 2: Correlation Policy へのバインド ]
1. Policy Management でポリシーを新規作成/編集。
2. "Add Allow List" をクリックし "ALLOW_LIST_SERVER_FARM" をアサイン。
3. 違反発生時の Response (Syslog Alert や Remediation) をアサインして Policy を Active 化。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FMC CLI / Web UI / Cisco IOS) | 表示確認ポイント |
| :--- | :--- | :--- |
| **FMC 相関エンジンプロセス状態確認** | <code>expert</code> ➔ <code>sudo manage_procs.pl \| grep Correlator</code> | `SFDataCorrelator` プロセスが `running` 状態であること |
| **FMC 相関イベント発生履歴確認** | FMC UI: `Analysis > Correlation > Events` | 生成された Correlation Event の日時、違反ルール名、IP アドレス |
| **修復アクション (Remediation) 実行ログ確認** | FMC UI: `Analysis > Correlation > Remediation Status` | Remediation 名、実行時刻、ステータス (`Successful` / `Failed`) |
| **FMC 内部修復実行スクリプトのログ確認** | <code>expert</code> ➔ <code>cat /var/sf/remediations/remediation.log</code> | ISE/Router 接続時の対話ログおよびエラーメッセージ詳細 |
| **Cisco ISE 側 ANC クアランティン状態確認** | ISE UI: `Operations > ANC > Policy Assignments`<br>ISE CLI: <code>show logging dynamic-authorization</code> | 対象 IP/MAC に対し ANC Policy がアサインされ CoA が成功したこと |
| **Cisco IOS ルーターでの Null Route 確認** | <code>show ip route static \| inc Null0</code> | `S 198.51.100.50/32 [1/0] via Null0` が自動追加されていること |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **相関条件に合致する侵入イベントが発生しても、修復アクションが一切発動しない** | Correlation Policy 内でルールが正しく登録されていないか、ポリシー自体が **"Active" に有効化されていない** | FMC UI `Policies > Correlation > Policy Management` | 対象 Correlation Policy のステータスを **Active**（緑色）に変更して Save を押す。 |
| **ISE pxGrid Mitigation が `Failed: Connection Error` で失敗する** | FMC と ISE 間の pxGrid 通信が未確立、または ISE 側で pxGrid クライアント（FMC）の証明書が未承認 | FMC `System > Integration > Identity Services Engine`<br>ISE `Administration > pxGrid Services` | ISE の pxGrid 画面で FMC からの接続承認状態を確認し、Pending の場合は Approve を押す。 |
| **Cisco IOS Null Route 修復が `Failed: Login Timeout / Authorization Error` で失敗する** | ルーター側の Telnet/SSH 設定誤り、特権レベル (Privilege 15) 不足、または enable パスワード不一致 | `/var/sf/remediations/remediation.log`<br>ルーター側 `show log` | FMC の Instance 設定でユーザー名・パスワード・enable パスワードを再確認し、ルーター側で Priv 15 を与える。 |
| **Impact Level 1 条件の相関ルールがヒットしない** | Network Discovery (RNA) が有効になっておらず、対象ターゲットホストの Host Profile（OS/Vulnerability）が未収集ため Impact 4 (Unknown) になっている | FMC `Analysis > Hosts > Network Map`<br>`Analysis > Intrusions > Events` | Network Discovery Policy で対象サブネットの自動検出を有効化し、ホスト情報を収集・蓄積させる。 |
| **コンプライアンス許可リストを作成したが、既存の非準拠ホストに対する違反イベントが出力されない** | 仕様上、初回評価時やリスト作成直後には既存不整合端末のイベントは一括生成されない（過剰イベント防止） | FMC Correlation Guide | リスト有効化後に新しく発生した通信・サービス変更でのみイベントがトリガーされる仕様を理解しておく。 |

---

## ⚠ 制限事項

* **FMC-HA (High Availability) 環境での Correlator 動作制限**:
  * FMC HA 構成時、`SFDataCorrelator` プロセスは **Active FMC 上でのみ動作** します。Standby FMC 側では相関ルール評価や Remediation 発動は実行されません（Active 障害発生時のフェールオーバー後に自動引き継ぎ）。
* **ISE-PIC (Passive Identity Connector) の機能制限**:
  * FMC と ISE の連携において、ISE-PIC を使用している場合は **pxGrid ANC/EPS Remediation アクションは実行できません**（完全な Cisco ISE ソフトウェアが必要）。
* **Null Route 修復の方向性制限**:
  * Destination ベースの修復アクションを Discovery や Host Input イベントをトリガーとしてアサインすることはできません（これらは Source ホストに関連付けられるため）。

---

## 🔄 他技術との関連

* **Cisco ISE (Identity Services Engine)**: pxGrid 経由で FMC からの修復命令を受け取り、Rapid Threat Containment (RTC) / ANC Quarantine を実行。
* **AMP for Endpoints (Secure Endpoint)**: エンドポイント上でのマルウェア検出・クアランティン失敗イベントを FMC へ通知し、相関ルールのトリガーとして供給。
* **Network Discovery Policy (RNA)**: ネットワーク内の OS、サービス、Vulnerability 情報を収集し、Impact Assessment (Impact Level 1〜4) の精度を決定。

---

## 🧩 比較表

### 1. FMC Correlation Rule vs Compliance Allow List vs Traffic Profile

| 比較項目 | Correlation Rule (相関ルール) | Compliance Allow List (許可リスト) | Traffic Profile (トラフィックプロファイル) |
| :--- | :--- | :--- | :--- |
| **目的** | 複合セキュリティイベント発生時のリアルタイム検知 ＆ 自動修復 | ポリシー違反（未承認 OS/App/Port）の検出 | ネットワークトラフィック帯域/件数の偏差（アノマリ）検出 |
| **主な評価条件** | Event Type (Intrusion/Malware) ＋ Impact Level ＋ Host OS | Subnet 宛ての許可 OS, Services, Protocols の明示定義 | 一定期間（時間/曜日）の平均トラフィック量に対する増減率 |
| **発動アクション** | Alert (Syslog/SNMP/Email) ＋ **Remediation (ISE/IOS)** | Alert ＋ **Remediation** | Alert ＋ **Remediation** (※Nmap を除く) |

### 2. Cisco ISE pxGrid Remediation vs Cisco IOS Null Route Remediation

| 比較項目 | Cisco ISE pxGrid Mitigation (ANC) | Cisco IOS Null Route Remediation |
| :--- | :--- | :--- |
| **制御対象** | アクセス層スイッチ/WLC ポート (端末レベルの隔離) | L3 ルーター (ルーティングレベルでのブラックホール化) |
| **通信プロトコル** | **pxGrid (XMPP / WebSocket API)** | **Telnet / SSH (CLI コマンド自動スクリプト)** |
| **動作アクション** | Quarantine (VLAN/dACL 変更)、Port Shutdown | <code>ip route &lt;IP&gt; 255.255.255.255 Null0</code> 追加 |
| **復旧 (Un-quarantine)**| **対応** (Unquarantine アクションを定義可能) | ルーターからの手動 `no ip route` 削除が必要 |

---

## 💡 ベストプラクティス

1. **Impact Level 1 に限定した自動修復 (Rapid Threat Containment) の運用**:
   * 自動遮断（ISE Quarantine や Null Route）を軽率に全侵入イベントへ適用すると、誤検知 (False Positive) 時に正常な業務端末を隔離してしまうリスクがあります。相関ルール条件には必ず **`Impact Flag is 1 - red (Vulnerable)`** を指定し、実際に脆弱性が存在するターゲットへの攻撃時のみ自動遮断を発動させます。
2. **Snooze Period（スヌーズ期間）の適切な設定**:
   * 同一の攻撃通信が短時間に大量発生した場合、修復スクリプトや pxGrid 命令が連射されて FMC/ISE に過大な負荷がかかります。相関ルール内で **`Snooze: 1 hour`** 等を設定し、一度修復が発動した後は一定時間再発動を抑止します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: FMC での ISE pxGrid ANC クアランティン相関ルールの作成

#### 問題
FMC において、Windows サーバー群 (`172.16.10.0/24`) 宛てに Impact Level 1 (Vulnerable) の侵入イベントが発生した場合、即座に ISE pxGrid 経由で攻撃元 IP をクアランティン処理する相関ルール `RULE_RTC_QUARANTINE` を作成せよ。

#### 要件
* Impact Flag: `1 - red (Vulnerable)`
* Target Network: `172.16.10.0/24`

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Actions > Modules ➔ "pxGrid Mitigation" ➔ Add Instance "ISE_INST"
2. Edit "ISE_INST" ➔ Add Remediation "REM_QUARANTINE":
   - Mitigation Action: Quarantine
   - Configured IP: Source IP
3. Policies > Correlation > Rule Management ➔ "Create Rule"
   - Rule Name: RULE_RTC_QUARANTINE
   - Event: an intrusion event occurs
   - Condition: Impact Flag is 1 - red (Vulnerable)
   - Add Complex Condition: Destination IP is in 172.16.10.0/24
4. Policies > Correlation > Policy Management ➔ "Create Policy" (Name: POL_RTC)
   - Add Rule: RULE_RTC_QUARANTINE
   - Assign Response: REM_QUARANTINE
   - Status: Active ➔ Save.
```

#### 検証方法
* テスト攻撃を送信し、FMC の `Analysis > Correlation > Remediation Status` で `Successful` と記録され、ISE の ANC 画面で該当 IP が Quarantine されていることを確認。

---

### 例2: AMP for Endpoints マルウェア隔離失敗時の自動 ISE ネットワーク隔離

#### 問題
AMP for Endpoints エンドポイント上でマルウェアの自動隔離に失敗（`Malware Execution / Quarantine Failure`）したイベントを受信した際、該当ホスト IP を ISE で自動クアランティンする相関ルールを作成せよ。

#### 要件
* Event Base: `a malware event occurs`

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Correlation > Rule Management ➔ "Create Rule"
   - Rule Name: RULE_AMP_QUARANTINE_FAIL
   - Select Event: a malware event occurs
   - Condition: Disposition is Malware AND Action is not Quarantined
2. POL_RTC ポリシーへアサインし、Remediation Response として `REM_QUARANTINE` をアサインして Save ＆ Active。
```

#### 検証方法
* AMP テストファイル（EICAR）を端末上で検知させ、FMC Correlation Events に発生記録が残ることを確認。

---

### 例3: Cisco IOS Null Route による C2 サーバー自動ブラックホール化

#### 問題
内部ホストから外部 Botnet C2 サーバーへの通信（Connection Event で Security Intelligence が C2-IP にマッチ）が発生した際、エッジルーター R1 (`10.1.1.1`) 上で該当 C2 アドレスを Null Route 化する修復インスタンスを作成せよ。

#### 要件
* Module: `Cisco IOS Null Route`
* Action: `Block Destination`

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Actions > Modules ➔ "Cisco IOS Null Route" ➔ Add Instance "R1_NULLROUTE":
   - Router IP: 10.1.1.1, Credentials: admin / Cisco123
2. Add Remediation "REM_NULL_C2":
   - Action: Block Destination, Subnet Mask: 255.255.255.255
3. Correlation Rule "RULE_C2_DETECTED" (Connection Event with SI Category = Botnet) を作成し、Response として "REM_NULL_C2" をアサインして Policy を Active 化。
```

#### 検証方法
* R1 ルーター上で `show ip route static` を実行し、`S <C2_IP>/32 is directly connected, Null0` が存在することを確認。

---

### 例4: 未承認 WEB サーバー起動を検出する Compliance Allow List の構成

#### 問題
DMZ セグメント (`172.16.50.0/24`) において、事前に登録された WEB サーバー以外のホストで TCP 80/443 サーバーポートが新たにオープンした場合に Syslog アラートを発出する Compliance Allow List を構築せよ。

#### 要件
* `Compliance Allow List` を使用すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Correlation > Allow List ➔ "Add Allow List" (Name: ALLOW_WEB_ONLY)
   - Target Networks: 172.16.50.0/24
   - Allow Host Profiles:
     - Globally Allowed Application Protocols: HTTP, HTTPS
2. Policy Management で POL_ALLOW_CHECK を作成し、ALLOW_WEB_ONLY をアサイン。
3. Response として Syslog Alert をアサインして Active 化。
```

#### 検証方法
* 対象セグメント内の非許可端末で `python3 -m http.server 80` を起動し、FMC の `Analysis > Correlation > Allow List Events` に違反が記録されることを確認。

---

### 例5: Nmap 自動スキャン修復による不審ホストのリコンサイナンス

#### 問題
新しく未認識のホスト（New Host Discovery Event）が LAN 内 (`10.20.0.0/16`) に参加した際、自動的に FMC から Nmap スキャンを発動して OS/ポート情報を収集する設定を行え。

#### 要件
* Module: `Nmap Scanning`

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Actions > Modules ➔ "Nmap Scanning" ➔ Add Instance "NMAP_INST":
   - Scan Options: -sS -O (TCP SYN scan + OS Detection)
2. Add Remediation "REM_NMAP_NEW_HOST":
   - Scan Target: Host involved in event
3. Rule "RULE_NEW_HOST" (a discovery event occurs: a new host is detected) を作成し、Response として "REM_NMAP_NEW_HOST" をバインドして Policy を Active 化。
```

#### 検証方法
* 新規 IP で通信を発生させ、FMC の `Analysis > Correlation > Remediation Status` で Nmap スキャンの実行完了を確認。

---

### 例6: Host Attribute (Host Criticality) の自動更新修復

#### 問題
特定ホストで高深刻度の脆弱性が検知された際、FMC 内の Host Criticality 属性を自動的に `High` に更新する修復アクションを作成せよ。

#### 要件
* Module: `Set Attribute Value`

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Actions > Modules ➔ "Set Attribute Value" ➔ Add Instance "SET_CRITICALITY":
2. Add Remediation "REM_SET_HIGH_CRITICALITY":
   - Attribute: Host Criticality, Value: High
3. 相関ルール違反時の Response として本インスタンスを割り当てて適用。
```

#### 検証方法
* FMC の `Analysis > Hosts > Network Map` で該当ホストの Criticality が `High` に変化したことを確認。

---

### 例7: 特定ユーザー（Guest）による大量データ送信（Traffic Profile Anomaly）の検知

#### 問題
過去 1 週間のベースラインと比較し、ゲストネットワークからの送信トラフィック量が 300% 以上急増した際に管理者宛てに Email アラートを送信する Traffic Profile と相関ルールを作成せよ。

#### 要件
* `Traffic Profiles` 機能を使用すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Correlation > Traffic Profiles ➔ "Create Traffic Profile"
   - Name: TP_GUEST_TRAFFIC
   - Tracked Traffic: Source Network 192.168.200.0/24
   - Sample Interval: 1 Hour
2. Rule Management ➔ Create Rule:
   - Select Event: a traffic profile changes (TP_GUEST_TRAFFIC increases by 300%)
3. Correlation Policy にバインドして Email Alert をアサイン。
```

#### 検証方法
* 該当セグメントから疑似大容量トラフィックを発生させ、Traffic Profile 変化イベントがトリガーされることを確認。

---

### 例8: Correlation Rule 内での Snooze Period（通知抑制）の設定

#### 問題
相関ルール `RULE_RTC_QUARANTINE` が一度発動した後、同じホストに対する連続的なイベント発生によって修復アクションが多発するのを防ぐため、1 時間の Snooze 期間を設定せよ。

#### 要件
* `Snooze` パラメータを 1 時間に設定すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. Policies > Correlation > Rule Management ➔ `RULE_RTC_QUARANTINE` を編集。
2. 画面下部 "Rule Options" ➔ "Snooze":
   - Check "If this rule generates an event, snooze for 1 hours"
3. Save して変更を更新。
```

#### 検証方法
* 短時間に連続して攻撃イベントを発生させ、最初のアラートから 1 時間以内は新たな相関イベントが抑止されることを確認。

---

### 例9: FMC CLI を使用した SFDataCorrelator プロセスログの直接調査

#### 問題
FMC CLI にログインし、`SFDataCorrelator` プロセスの動作ログおよび修復スクリプト実行ログをリアルタイムで追跡せよ。

#### 要件
* `/var/sf/remediations/` ディレクトリ内のログを確認すること。

#### 設定例

```bash
# ===== FMC CLI Expert Mode =====
> expert
admin@FMC:~$ cd /var/sf/remediations/
admin@FMC:/var/sf/remediations$ tail -f remediation.log
```

#### 検証方法
* トラブルシューティング時に `remediation.log` 内にエラーコード（例: SSH 認証失敗や ISE ポート応答なし）が正確に出力されていることを確認。

---

### 例10: FMC REST API を使用した Correlation Policy 一覧の取得

#### 問題
FMC REST API を使用して、定義されている全 Correlation Policy のオブジェクト情報および有効化ステータスを JSON 形式で取得せよ。

#### 要件
* エンドポイント `/api/fmc_config/v1/domain/{domainUUID}/policy/correlationpolicies` を使用すること。

#### 設定例

```bash
curl -k -X GET "https://fmc.corp.local/api/fmc_config/v1/domain/e276abec-e0f2-11e3-8169-6d9ed13b206f/policy/correlationpolicies" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json"
```

#### 検証方法
* レスポンスの JSON 内で `enabled: true` と表示されているアクティブポリシーを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者は FMC 上で Impact Level 1 の侵入イベントが発生した際に Cisco ISE 経由で攻撃元 IP を自動隔離（Quarantine）する相関ルールと修復インスタンスを作成した。しかし、実際のラボテストで Impact Level 1 の侵入イベントが発生したにもかかわらず、ISE 上で端末が隔離されず、FMC 上にも Correlation Event が一切記録されなかった。FMC 設定上の原因として最も可能性が高いものはどれか。

* A) FTD の Threat License が期限切れであるため。
* B) 作成した Correlation Rule をアサインした Correlation Policy が **"Active"（有効）状態に設定されていない**。
* C) ISE 側で AMP for Endpoints ライセンスが未登録であるため。
* D) FTD の Snort エンジンがリスタート中であるため。

**正解**: **B**  
**解説**: FMC の相関エンジンでは、ルールや修復アクションを作成しただけでは不十分であり、必ず **Correlation Policy 内にルールを追加し、そのポリシーを "Active"（トグルを緑色）に設定して保存すること** が動作の絶対条件です。

---

### 2. トラブルシュート問題

**問題**: FMC から Cisco ISE への pxGrid Mitigation (ANC Quarantine) 修復アクションを実行したところ、FMC の Remediation Status ログに `Failed: pxGrid Client Not Approved` と記録された。この問題を解決するための最適な手順はどれか。

* A) FMC で FTD を再登録する。
* B) Cisco ISE の GUI 画面 (`Administration > pxGrid Services`) にログインし、Pending 状態になっている FMC からの **pxGrid Client 接続リクエストを "Approve" (承認)** する。
* C) ルーター上で `ip route` コマンドを手動投入する。
* D) FMC の SSL Inspection Policy を無効化する。

**正解**: **B**  
**解説**: FMC と ISE 間で pxGrid 連携を初期構成する際、ISE 側の pxGrid 画面において FMC からのクライアント接続証明書リクエストが Pending 状態になります。これを ISE 管理者が手動で **Approve** しない限り、修復命令 (CoA) は拒否されます。

---

### 3. Design 問題

**問題**: 企業の SOC において、侵入検知システム (NGIPS) が発生させる膨大なアラートの中から、実際に攻撃対象サーバーの OS や脆弱性に合致して成功するリスクが極めて高い攻撃（Impact Level 1 - Vulnerable）のみを自動抽出し、Cisco ISE 経由で即時ネットワーク隔離（Rapid Threat Containment）を行いたい。相関ルールの条件設定として最も適切な構成はどれか。

* A) Base Event: `a connection event occurs` ＋ Condition: `Security Intelligence Category is Block`
* B) Base Event: `an intrusion event occurs` ＋ Condition: **`Impact Flag is 1 - red (Vulnerable)`**
* C) Base Event: `a discovery event occurs` ＋ Condition: `Host OS is Any`
* D) Base Event: `user activity is detected` ＋ Condition: `User Priority is High`

**正解**: **B**  
**解説**: FMC の **Impact Assessment** 機能は、ターゲットホストの脆弱性情報と攻撃シグネチャを自動照合します。**`Impact Flag is 1 - red (Vulnerable)`** を指定することで、実際に脆弱性が存在するターゲットに対する攻撃イベントのみを厳選して自動隔離を発動させることができます。

---

### 4. 実装問題

**問題**: FMC の Cisco IOS Null Route 修復モジュールを使用して、攻撃元 IP をエッジルーター上でブラックホール化させる際、FMC がルーターへ接続してコマンドを発行するために必要なルーター側の必須設定はどれか。

* A) ルーター側で RADIUS CoA を有効化する。
* B) ルーター側で Telnet または SSH を有効化し、**Privilege 15（特権レベル 15）のログインアカウント** を構成する。
* C) ルーター側で SNMPv3 Trap を設定する。
* D) ルーター側で NetFlow v9 を有効化する。

**正解**: **B**  
**解説**: Cisco IOS Null Route モジュールは、FMC がルーター宛てに Telnet/SSH 接続し、CLI で <code>ip route &lt;IP&gt; 255.255.255.255 Null0</code> を流し込む仕組みです。そのためルーター側に Telnet/SSH と特権レベル 15 (Privilege 15) のアカウントが必要です。

---

### 5. コンフィグ読解問題

**問題**: FMC の Correlation Policy において、`Snooze` オプションを `1 Hour` に設定した場合の動作に関する説明として正しいものはどれか。

* A) イベント発生後、1 時間経過してから修復アクションが実行される。
* B) 相関ルールが一度トリガーされた後、**続く 1 時間（スヌーズ期間）はそのルールによる新たな相関イベントの生成・修復発動が抑止される**。
* C) 1 時間ごとに FTD の Snort エンジンが自動リロードされる。
* D) 相関ルールが 1 時間後に自動削除される。

**正解**: **B**  
**解説**: **Snooze（スヌーズ）** 機能は、短い時間内に同一ルールが連射されて修復アクションやアラートが過剰実行されるのを防ぐための抑制タイマーです。一度発動すると指定時間内は新たなイベントトリガーを抑止します。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center System Configuration Guide: Correlation Policies](https://www.cisco.com/)
* [Cisco Secure Firewall Management Center System Configuration Guide: Remediations](https://www.cisco.com/)
* [Cisco Technical Notes: Configure Rapid Threat Containment (RTC) with FMC and Cisco ISE via pxGrid](https://www.cisco.com/)
* [Cisco Technical Notes: Troubleshoot FMC Correlation Engine and Remediation Modules](https://www.cisco.com/)
* [Cisco Live: Automated Incident Response with Firepower Management Center and Cisco ISE (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で Correlation ＆ Remediation が出題された場合の構成判断フローチャート：
    * **「Automatically quarantine host on ISE when Impact 1 intrusion occurs」** ➔ FMC Policies > Actions > Modules ➔ pxGrid Instance ➔ Remediation: Quarantine ➔ Rule (Intrusion + Impact 1) ➔ Policy にアサイン ➔ **Activate**。
    * **「Null route attacker IP on Cisco IOS router when C2 connection detected」** ➔ FMC Modules ➔ IOS Null Route Instance (Router IP + Credentials) ➔ Remediation: Block Source ➔ Rule (Connection + C2) ➔ Policy にアサイン ➔ **Activate**。
    * **「Detect unauthorized web server in DMZ」** ➔ FMC Correlation > Allow List ➔ Target Net + Allowed Ports (80/443) ➔ Policy にアサイン ➔ **Activate**。
* **図解（Rapid Threat Containment 全体フロー）**:
  ```text
  [ Attacker ] ──(Exploit)──► [ FTD (IPS Detect) ] ──(Event)──► [ FMC Correlator ]
                                                                      │
                                                           Matches Impact 1 Rule
                                                                      │
                                                      (pxGrid ANC Quarantine)
                                                                      ▼
                                                                [ Cisco ISE ]
                                                                      │
                                                          (RADIUS CoA Push)
                                                                      ▼
                                                           [ Access Switch ]
                                                                      │
                                                      [ Attacker/Target Port Blocked ]
  ```
* **注意点**:
  * ラボ試験で Correlation 設定を完了した後は、必ず `Policies > Correlation > Policy Management` でトグルスイッチが **Active** (緑色) になっていることを目視確認し、テストパケット発生後に `Analysis > Correlation > Remediation Status` でステータスが **Successful** になっていることをログ確認してください。
