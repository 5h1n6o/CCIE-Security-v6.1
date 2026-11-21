---
layout: default
title: 4.10-Profiling
nav_order: 10
parent: 4.0-Identity-Management
---

# 4.10 Endpoint profiling using Cisco ISE and Cisco network infrastructure including device sensor

## 📘 概要

Endpoint Profiling（エンドポイント プロファイリング）は、ネットワークに接続するデバイスの種別（PC、IP Phone、プリンタ、カメラ、医療機器、IoTデバイスなど）を自動的に識別・分類し、適切なアクセス制御ポリシー（802.1X、MAB、Dynamic VLAN、dACL、Security Group Tag: SGT）を適用するための基盤技術です。

CCIE Security v6.1 試験において、ISE Endpoint Profiling および Network Infrastructure（特に Catalyst Switch および WLC における Device Sensor）の統合設計と実装は、Identity & Access Control 領域の中で極めて重要なテーマです。エージェントレス環境（AnyConnect Posture エージェントを導入できないプリンタや IoT デバイス）における可視化とセキュリティポリシー自動化の中核を担います。

### 主な利用目的と適用シーン

1. **エージェントレスデバイスの自動識別と動的アクセス制御**
   - 802.1X サプリカントを搭載していない IP Phone、ネットワークプリンタ、防犯カメラ、医療機器などを MAB（MAC Authentication Bypass）で認証する際、MAC アドレスの偽装（MAC Spoofing）による不正アクセスを防止します。
   - Profiling 結果（例: `Cisco-IP-Phone` や `Printer`）に基づいて、ISE の Authorization Policy で自動的に専用 VLAN または専用 SGT を割り当てます。

2. **Device Sensor によるネットワークインフラ主導の属性収集**
   - アクセススイッチ（Catalyst）や無線 LAN コントローラ（WLC）自身が、接続されたエンドポイントから送出される L2/L3 パケット（CDP, LLDP, DHCP Request, HTTP User-Agent など）をリアルタイムにスヌーピング・解析します。
   - 収集した TLV（Type-Length-Value）情報を RADIUS Accounting パケット（または Cisco RADIUS Vendor-Specific Attributes: VSA）に格納して ISE に送信することで、ネットワーク帯域や ISE 処理負荷を最小限に抑えつつ高精度なプロファイリングを実現します。

3. **プロファイル変更に伴う動的ポリシー再適用（Profiler CoA）**
   - 初期接続時（MAC アドレスのみが判明している段階）には制限されたアクセス（Quarantine や Limited VLAN）を適用し、プロファイリング処理によってデバイス種別が確定した瞬間に RADIUS CoA（Change of Authorization: Re-Auth または Port-Bounce）を発行して、正規アクセス権限へ即座に動的昇格させます。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **機能名称** | Cisco ISE Endpoint Profiling Service & Device Sensor Integration |
| **主要コンポーネント** | Cisco ISE（Profiler Service）、Catalyst Switch（Device Sensor Feature）、Cisco WLC（Device Sensor）、DHCP Relay/IP Helper |
| **収集プローブ（Probes）** | DHCP Probe, RADIUS Probe, HTTP Probe, CDP Probe, LLDP Probe, NMAP Scan Probe, Active Directory Probe, SNMP Query/Trap Probe, NetFlow Probe, DNS Probe |
| **Device Sensor 役割** | スイッチ/WLC エッジで CDP/LLDP/DHCP/HTTP の TLV 属性をスヌーピングし、RADIUS Accounting パケットに載せて ISE へ効率的に伝送 |
| **プロファイル判定アルゴリズム** | Rule-based Evaluation (Attribute-based matching) + Certainty Factor (確信度ポイントの加算・減算) |
| **メリット** | エージェント不要、MAC 偽装攻撃の防止、ネットワークエッジでの分散属性収集による低負荷化、即時ポリシー再適用（CoA 連携） |
| **デメリット/考慮事項** | DHCP/HTTP パケットが ISE に到達しない構成では高精度な識別が不可、NMAP パック/SNMP パッシブ等のアクティブスキャンはネットワーク負荷増大の懸念 |
| **対応機種** | Cisco Catalyst 3650/3850/9200/9300/9400/9500 (IOS-XE)、Cisco AireOS WLC、Catalyst 9800 WLC、Cisco ISE 2.x/3.x |
| **制限事項** | 共有バッファの枯渇を防ぐため Device Sensor の出力 TLV フィルタリングが必要、Static Profile 割り当て時は Dynamic Profiling が無効化される点に注意 |
| **設計上の注意点** | DHCP Relay（ip helper-address）による ISE へのパケット転送、RADIUS Accounting の有効化（`aaa accounting network default start-stop group RADIUS-GROUP`）、CoA の疎通（UDP 1700/3799）確保 |

---

## 🏗 動作原理

### 全体アーキテクチャと属性伝送フロー

Cisco ISE Profiling アーキテクチャは、「属性収集（Attribute Collection）」「プロファイル評価（Profile Evaluation）」「ポリシー適用・CoA発行（Policy Enforcement & CoA）」の3つのステップで構成されます。

```
[Endpoint (IP Phone / PC / Printer)]
        │
        │ 1. L2/L3 パケット送出 (CDP, LLDP, DHCP Request, HTTP GET)
        ▼
[Catalyst Switch / WLC (Device Sensor)]
        │
        │ 2. パケットを解析し TLV 属性を抽出
        │ 3. RADIUS Accounting Request (拡張 VSA 属性) に集約して送出
        ▼
[Cisco ISE (PSN - Profiler Service)]
        │
        │ 4. 属性データベースと Profiling Policies (Certainty Factor) を照合
        │ 5. Endpoint Profile 確定 (例: Cisco-Device:Cisco-IP-Phone-7965)
        │ 6. Authorization Policy を再評価
        │ 7. RADIUS CoA (Re-Auth / Port-Bounce) 送出
        ▼
[Catalyst Switch / WLC]
        │
        │ 8. Port-Bounce または Re-Auth により新アクセス権限 (VLAN/dACL/SGT) 適用
        ▼
[Endpoint Profile & Access Updated]
```

### Profiling Probes の種類と収集方法

ISE は多様なプローブを使用してエンドポイントの属性を収集します。

1. **RADIUS Probe & Device Sensor**
   - **メカニズム**: スイッチからの RADIUS Access-Request および Accounting-Request に含まれる MAC アドレス、NAS-Port、Calling-Station-ID、および Device Sensor が付加した VSA（CDP/LLDP/DHCP/HTTP の TLV）を取得します。
   - **利点**: ネットワーク帯域を消費せず、最も効率的かつリアルタイムに属性を収集できます。

2. **DHCP Probe**
   - **メカニズム**: エンドポイントが送出する DHCP Request に含まれる `Option 55 (Parameter Request List)`、`Option 60 (Vendor Class Identifier)`、`Option 12 (Host Name)` などを解析します。
   - **収集方式**: 
     a) DHCP Relay (`ip helper-address <ISE-PSN-IP>`) で ISE に UDP 67 パケットを転送。
     b) SPAN / SPAN ポートミラーリングで ISE の NIC に注入。
     c) Device Sensor 経由で RADIUS Accounting に載せて伝送。

3. **HTTP Probe**
   - **メカニズム**: エンドポイントが Web ブラウジングやポータルリダイレクト時に送信する HTTP ヘッダー内の `User-Agent` 属性（例: `Mozilla/5.0 (iPhone; CPU iPhone OS 15_0 like Mac OS X)`）を解析します。
   - **収集方式**: Central Web Authentication (CWA) や Guest Portal リダイレクト時、または Device Sensor の HTTP スヌーピング機能を使用。

4. **CDP / LLDP Probe**
   - **メカニズム**: シスコ製機器（CDP）や汎用ネットワーク機器・IP Phone（LLDP）が隣接スイッチへ通知する `Device ID`、`Capabilities`、`Software Version` などの TLV を解析します。

5. **NMAP Scan Probe (Active Probe)**
   - **メカニズム**: ISE がエンドポイントに対して能動的に TCP/UDP ポートスキャンや OS 指紋認証（OS Fingerprinting）を発行します。
   - **注意点**: ネットワークファイアウォールでブロックされる可能性があり、エンドポイントのセキュリティ警報を誘発するリスクがあるため、特定プロファイルの補正目的でのみ補完的に使用します。

---

## ⚙ 動作シーケンス

エンドポイント接続からプロファイリング完了、および CoA による動的アクセス変更までの詳細なシーケンスは以下の通りです。

```
Endpoint             Switch / WLC                 ISE (PSN)
   │                      │                           │
   │─── 1. Link Up ──────>│                           │
   │                      │── 2. RADIUS Access-Req ──>│ (MAC Auth / MAB)
   │                      │   (Calling-Station-ID)    │
   │                      │<─ 3. RADIUS Access-Accept─│ (Initial Authorization:
   │                      │   (Limit-VLAN / Default)  │  Dynamic Profile: Unknown)
   │<── 4. Port Authorized│                           │
   │    (DHCP開始)        │                           │
   │                      │                           │
   │─── 5. CDP/LLDP ─────>│ (Device Sensor Snooping)  │
   │─── 6. DHCP Request ─>│ (Option 55/60 Snooping)   │
   │                      │                           │
   │                      │── 7. RADIUS Accounting ──>│ (Device Sensor TLVs:
   │                      │   Request (Start/Update)  │  CDP Device-ID,
   │                      │                           │  DHCP Option 60 etc.)
   │                      │                           │
   │                      │                           │─── 8. Profiler Engine Evaluation
   │                      │                           │    - Match Rule: Option 60 = "Cisco IP Phone"
   │                      │                           │    - Add Certainty Factor (+60)
   │                      │                           │    - Threshold Reached (100/100)
   │                      │                           │    - Profile: Cisco-IP-Phone
   │                      │                           │
   │                      │<─ 9. RADIUS CoA (Re-Auth)─│ (Change of Authorization)
   │                      │── 10. RADIUS CoA-ACK ────>│
   │                      │                           │
   │                      │── 11. Re-Authentication ─>│ (New Session)
   │                      │<─ 12. RADIUS Access-Accept│ (Matched Rule: Cisco-IP-Phone
   │                      │   (Voice-VLAN / Voice-SGT)│  Assign Voice Access)
   │<── 13. Policy Applied│                           │
```

---

## 🎯 試験対策（CCIE Securityラボ試験）

### 1. Blueprint における重要ポイント
- **Device Sensor の正確な CLI 設定**: `device-sensor filter-list` の定義、`device-sensor filter-spec` のバインド、および `device-sensor notify all-changes` の投入。
- **DHCP Option 60 / 55 のプロファイリングルール構築**: ISE 側で特定の文字列（例: `PolycomVVX`, `Cisco AP`）にマッチするカスタム Profiling Policy を作成し、Certainty Factor の加算ロジックを設定できること。
- **Profiler CoA の適切な選択**:
  - `Re-Auth`: EAP や MAB セッションを維持したまま、バックグラウンドで認可ポリシー（VLAN / dACL / SGT）を更新する場合に使用。
  - `Port-Bounce`: エンドポイントが DHCP IP アドレスを再取得する必要がある場合（VLAN が変更された場合など）、スイッチポートを物理的にダウン/アップさせて IP 再取得を強制します。

### 2. よくある設定ミス・落とし穴
- **RADIUS Accounting が未設定**: `aaa accounting network default start-stop group RADIUS-GROUP` を設定し忘れると、Device Sensor が収集した TLV が ISE に送信されず、プロファイルが `Unknown` のままになります。
- **Device Sensor の TLV フィルタ未バインド**: `device-sensor filter-list` を作成しても、`device-sensor filter-spec` にバインドしないと TLV が RADIUS 属性に含まれません。
- **DHCP Relay (ip helper-address) の考慮不足**: Device Sensor を使わない構成で DHCP Probe を利用する場合、SVI に `ip helper-address <ISE-PSN-IP>` を設定していないと ISE が DHCP パケットを受信できません。
- **CoA 疎通および共有鍵の不一致**: スイッチ側で `aaa server radius dynamic-author` が未設定、あるいは client 指定の `key` が ISE の Client 設定と一致していないと CoA がタイムアウト（CoA-NAK / No Response）します。

---


## 🛠 設定方法

### 1. Cisco Catalyst スイッチにおける Device Sensor の設定 (IOS-XE)

Catalyst スイッチ側で CDP、LLDP、DHCP の TLV 情報をスヌーピングし、RADIUS Accounting パケットに追加して ISE 送信するコンフィグ例です。

```text
! --- 1. RADIUS Server および AAA Accounting の設定 ---
aaa new-model
aaa authentication dot1x default group ISE-GROUP
aaa authorization network default group ISE-GROUP
aaa accounting network default start-stop group ISE-GROUP

aaa server radius dynamic-author
 client 10.1.100.21 auth-type any secret Cisco123!
 client 10.1.100.22 auth-type any secret Cisco123!

radius-server attribute 6 1
radius-server attribute 8 1
radius-server attribute 25 re-evaluate
radius-server attribute 31 mac format ietf upper-case

! --- 2. Device Sensor フィルタの定義 (CDP, LLDP, DHCP) ---
device-sensor filter-list cdp list CDP-LIST
 tlv name device-name
 tlv name capabilities-address
 tlv name platform-string
 tlv name version-string

device-sensor filter-list lldp list LLDP-LIST
 tlv name system-name
 tlv name system-description
 tlv name system-capabilities
 tlv name management-address

device-sensor filter-list dhcp list DHCP-LIST
 option name host-name
 option name parameter-request-list
 option name class-identifier
 option name client-identifier

! --- 3. Device Sensor Spec へのフィルタバインドと通知有効化 ---
device-sensor filter-spec cdp include list CDP-LIST
device-sensor filter-spec lldp include list LLDP-LIST
device-sensor filter-spec dhcp include list DHCP-LIST

device-sensor notify all-changes
device-sensor accounting
```

### 2. Cisco WLC (AireOS / Catalyst 9800) における Device Sensor の設定

#### Catalyst 9800 (IOS-XE WLC) CLI 設定

```text
! --- WLC での Device Sensor 有効化 ---
device-sensor filter-list dhcp list WLC-DHCP-LIST
 option name class-identifier
 option name user-class-id
 option name parameter-request-list

device-sensor filter-spec dhcp include list WLC-DHCP-LIST
device-sensor accounting
device-sensor notify all-changes

ap profile default-ap-profile
 hyperlocation ble-beacon 0
! Wireless Policy Profile で RADIUS Profiling を有効化
wireless profile policy DEFAULT-POLICY
 device-sensor dhcp
 device-sensor http-tlv
 aaa-override
```

### 3. Cisco ISE 側での Profiling Probes および Profiling Policies 設定

#### GUI 設定手順 (ISE 3.x)

1. **Profiling Probes の有効化**
   - パス: `Work Centers` > `Profiler` > `Settings` > `Profiling Tracking` / `Probes`
   - 各 PSN ノードに対して以下のプローブを有効化:
     - **RADIUS Probe**: 有効 (Device Sensor TLVs の受信に必須)
     - **DHCP Probe**: 有効 (Port 67)
     - **CDP Probe**: 有効
     - **LLDP Probe**: 有効
     - **HTTP Probe**: 有効

2. **Custom Profiling Policy の作成例**
   - パス: `Work Centers` > `Profiler` > `Profiling Policies`
   - **Policy Name**: `Custom_Polycom_IP_Phone`
   - **Parent Policy**: `IP-Phone`
   - **Minimum Certainty Factor**: `60`
   - **Rules**:
     - Rule 1: `DHCP: dhcp-class-identifier CONTAINS PolycomVVX` (Certainty Factor: 60)
     - Rule 2: `LLDP: lldpSystemDescription CONTAINS Polycom` (Certainty Factor: 40)

3. **Profiler CoA の設定**
   - パス: `Work Centers` > `Profiler` > `Settings` > `Profiler CoA`
   - **CoA Type**: `Re-Auth` (デフォルト) または `Port-Bounce` (VLAN 変更を伴うポリシー適用時)

---

## 🔍 検証コマンド

| 目的 | コマンド | 確認・診断ポイント |
| --- | --- | --- |
| **Device Sensor 状態確認** | <code>show device-sensor cache all</code> | スイッチが抽出したエンドポイントごとの CDP/LLDP/DHCP TLV キャッシュ一覧を表示 |
| **特定の MAC の TLV 確認** | <code>show device-sensor cache mac &lt;MAC&gt;</code> | 該当デバイスのプロファイリング属性（DHCP Option 60 文字列など）が正常にスヌーピングされているか確認 |
| **Device Sensor 統計確認** | <code>show device-sensor debug status</code> | TLV 抽出処理の成功回数およびエラーカウンターを確認 |
| **RADIUS パケット送信確認** | <code>debug radius accounting</code> | RADIUS Accounting-Request に Device Sensor VSA (Attribute 26) が添付されて ISE へ送信されているか検証 |
| **Device Sensor デバッグ** | <code>debug device-sensor [error\|event\|detail]</code> | スヌーピング処理および RADIUS モジュールへの通知イベントをリアルタイムデバッグ |
| **ISE 側ログ確認** | `tail -f /opt/CSCOise/logs/profiler.log` | ISE CLI から Profiler Engine の評価・属性受信ログをリアルタイム追跡 |
| **ISE セッション状態確認** | `show sessions engine` (ISE CLI) | 該当 MAC の現在の割当エンドポイントプロファイル名および認証ステータスを表示 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **プロファイルが `Unknown` のまま変更されない** | スイッチ側で RADIUS Accounting が未設定、または Device Sensor 属性が更新されていない。 | <code>show device-sensor cache mac &lt;MAC&gt;</code><br><code>show aaa sessions</code> | スイッチコンフィグに `aaa accounting network default start-stop group ISE-GROUP` を追加し、`device-sensor accounting` を有効化する。 |
| **DHCP Option 60 が ISE に届かない** | スイッチで DHCP Snooping が無効、または `device-sensor filter-spec dhcp` に Option 60 (class-identifier) が含まれていない。 | <code>show device-sensor cache mac &lt;MAC&gt;</code> | `device-sensor filter-list dhcp` に `option name class-identifier` を明示的に登録し、`filter-spec` にバインドする。 |
| **プロファイル判定後もアクセス権限が昇格しない** | Profiler CoA が無効化されている、または CoA（UDP 1700）がファイアウォールで遮断されている。 | <code>debug radius</code> (Switch)<br>ISE `profiler.log` | ISE の Profiler CoA 設定を `Re-Auth` または `Port-Bounce` に設定し、スイッチ側で `aaa server radius dynamic-author` を正常に設定する。 |
| **スイッチの CPU 使用率が高騰する** | `device-sensor notify all-changes` により過度な RADIUS パケットが生成されている、または未知のパケットが過剰流入。 | <code>show process cpu sorted</code><br><code>show device-sensor debug status</code> | 必要な TLV のみに `filter-list` を絞り込み、不要なプロトコル（例: 未使用の HTTP スヌーピング）を削除する。 |
| **VLAN 変更後に IP アドレスが更新されない** | Profiler CoA のタイプが `Re-Auth` に設定されており、エンドポイントが DHCP Release/Renew を実行しない。 | <code>show authentication sessions interface &lt;Int&gt; details</code> | ISE の Profiler CoA 設定を `Port-Bounce` に変更し、ポートリンクダウンを発生させて DHCP 再取得を強制する。 |

---

## ⚠ 制限事項

1. **Static Profile による Dynamic Profiling の上書き禁止**
   - ISE 上で管理者手動により `Static Assignment`（固定プロファイル割り当て）を設定した場合、ネットワークからいくら新たな TLV 属性を受信しても Profiler Engine はプロファイルを動的更新しません。

2. **MAC Address Spoofing（MAC 偽装）の限界**
   - MAC アドレスのみに依存する MAB + Profiling では、攻撃者が正常な IP Phone の MAC アドレスと DHCP Option 60 文字列を偽装した場合、プロファイラを騙せる可能性があります。完全な防御には EAP-TLS 認証または Strict Certainty Factor ルールを組み合わせる必要があります。

3. **Device Sensor バッファおよびパケットサイズ制限**
   - 大量の TLV 情報を送信すると、RADIUS パケットサイズが MTU を超えて断片化（Fragmentation）し、RADIUS サーバ側でドロップされるリスクがあります。フィルタリストで不要な TLV を除外することが推奨されます。

---

## 🔄 他技術との関連

- **MAB (MAC Authentication Bypass)**: 初期認証において MAC アドレスのみで認証を行い、直後にプロファイリング処理を行ってデバイス種別を確定させる基本組み合わせ技術。
- **Cisco TrustSec (SGT)**: Profiling 判定結果に基づいて、アクセス層で即座に Voice SGT (`SGT 4`) や Camera SGT (`SGT 12`) を動的に割り当てるセキュリティグループ制御。
- **Dynamic VLAN & dACL**: デバイスプロファイルに応じて、プリンタ用 VLAN や監視カメラ用 restricted dACL を即座に適用する動的ネットワークアクセスポリシー。
- **802.1X Auth Fail / Guest VLAN**: プロファイリング失敗時または不正デバイス判定時に、隔離 VLAN（Quarantine VLAN）へ誘導するセーフティネット構造。

---

## 🧩 比較表

| 比較項目 | Device Sensor (RADIUS Accounting 経由) | Direct DHCP Relay / SPAN Probe | NMAP Scan (Active Probe) |
| --- | --- | --- | --- |
| **処理位置** | ネットワークエッジ (Switch / WLC) | インフラ核心部 / ネットワーク集中ポイント | ISE PSN ノード |
| **帯域/負荷** | 極めて低い（既存 Accounting パケットに統合） | 中～高（DHCP パケットの全コピーを ISE へ転送） | 高（能動的に TCP スキャンを実行） |
| **識別精度** | 高（L2 直結情報 CDP/LLDP + DHCP/HTTP） | 中（L3 パケット DHCP のみ） | 最高（ポート応答・TCP ACK ウインドウ解析） |
| **ネットワーク影響** | なし | わずか | あり（IDS/IPS 警報誘発リスク） |
| **適用推奨シーン** | Cisco スイッチ/WLC 統合環境の標準構成 | 他社製スイッチ混在環境 | 特殊医療機器・産業用 IoT 等の最終検証用 |

---

## 💡 ベストプラクティス

1. **Device Sensor を最優先プローブとして設計**
   - 大規模ネットワークにおける ISE の CPU / ネットワーク帯域の負荷を軽減するため、直接 DHCP コピーや SPAN ではなく、スイッチ/WLC の Device Sensor による TLV 収集を第一選択とします。
2. **必要最小限の TLV フィルタリング**
   - `device-sensor filter-list` にはプロファイリングで評価する必須 TLV（DHCP Option 12/55/60、CDP/LLDP の System Name/Capabilities）のみを登録し、不要な TLV の転送を制限します。
3. **Certainty Factor の段階的設計**
   - 1つの属性（例: MAC OUI のみ）で確信度 100 に達する設計は避け、`MAC OUI (+30)` + `DHCP Option 60 (+40)` + `CDP Device ID (+30)` のように複数の独立した属性を組み合わせるルールを設計します。
4. **VLAN 変更時は Port-Bounce CoA を使用**
   - プロファイリング結果によってエンドポイントの割り当て VLAN が変更される設計では、必ず `Port-Bounce` CoA を使用し、リンクダウンによってクライアントの DHCP リトライを誘発させます。

---


## 📝 ラボ学習・設定サンプル例

### 演習例 1: Catalyst スイッチにおける Device Sensor (CDP/LLDP/DHCP) の基本構成

#### 問題
Catalyst 9300 スイッチに接続するエンドポイントから CDP、LLDP、および DHCP の TLV 情報を収集し、Cisco ISE へ RADIUS Accounting 経由で転送するための Device Sensor を構成しなさい。

#### 要件
- RADIUS サーバ IP: `10.1.100.21`（Shared Secret: `Cisco123!`）
- RADIUS Accounting を有効化し、`start-stop` 通知を送信すること。
- CDP 属性: `device-name`, `capabilities-address`, `platform-string`
- LLDP 属性: `system-name`, `system-description`, `system-capabilities`
- DHCP 属性: `host-name`, `parameter-request-list`, `class-identifier`
- 属性値に変更が生じた場合は即座に RADIUS 経由で ISE へ通知すること。

#### 設定例

```text
! --- AAA および RADIUS 基本設定 ---
aaa new-model
aaa group server radius ISE-GROUP
 server-private 10.1.100.21 auth-port 1812 acct-port 1813 key Cisco123!

aaa authentication dot1x default group ISE-GROUP
aaa authorization network default group ISE-GROUP
aaa accounting network default start-stop group ISE-GROUP

aaa server radius dynamic-author
 client 10.1.100.21 secret Cisco123!

radius-server attribute 6 1
radius-server attribute 8 1
radius-server attribute 25 re-evaluate
radius-server attribute 31 mac format ietf upper-case

! --- Device Sensor フィルタリスト定義 ---
device-sensor filter-list cdp list CDP-FILTER
 tlv name device-name
 tlv name capabilities-address
 tlv name platform-string

device-sensor filter-list lldp list LLDP-FILTER
 tlv name system-name
 tlv name system-description
 tlv name system-capabilities

device-sensor filter-list dhcp list DHCP-FILTER
 option name host-name
 option name parameter-request-list
 option name class-identifier

! --- フィルタスペシフィケーションへのバインドと有効化 ---
device-sensor filter-spec cdp include list CDP-FILTER
device-sensor filter-spec lldp include list LLDP-FILTER
device-sensor filter-spec dhcp include list DHCP-FILTER

device-sensor notify all-changes
device-sensor accounting
```

#### 検証方法

1. スイッチでキャッシュされた属性の確認:
   `show device-sensor cache all`
2. 特定の MAC アドレスに対する抽出品目の詳細確認:
   `show device-sensor cache mac 0004.f211.2233`
3. RADIUS Accounting パケットに VSA が添付されていることの確認:
   `debug radius accounting` を実行し、ポートに端末を接続した際に送信される Accounting-Request 内に `cisco-av-pair = dhcp-class-identifier=...` や `cdp-device-name=...` が含まれているか検証します。

---

### 演習例 2: ISE カスタム Profiling Policy の作成（Polycom IP Phone 識別）

#### 問題
Cisco ISE において、ネットワーク上の Polycom 製 IP Phone を高精度に識別するためのカスタム Profiling Policy `Polycom_VVX_Phone` を作成しなさい。

#### 要件
- 親プロファイル: `IP-Phone`
- 判定に必要な最小確信度（Minimum Certainty Factor）: `60`
- ルール 1: DHCP クラス識別子（Option 60）に `PolycomVVX` が含まれる場合、確信度 `50` を加算。
- ルール 2: LLDP システム記述（lldpSystemDescription）に `Polycom` が含まれる場合、確信度 `30` を加算。
- 条件合致時の判定動作: プロファイルが `Polycom_VVX_Phone` に更新された場合、即座に `Re-Auth` CoA を発行すること。

#### 設定例（ISE 3.x 設定手順の同等記述）

1. **ISE 管理コンソールでプロファイル作成**:
   - `Work Centers` > `Profiler` > `Profiling Policies` に移動し、`Add` をクリック。
   - **Name**: `Polycom_VVX_Phone`
   - **Parent Policy**: `IP-Phone`
   - **Minimum Certainty Factor**: `60`

2. **Check Rules の追加**:
   - **Rule 1**:
     - Expression: `DHCP : dhcp-class-identifier CONTAINS PolycomVVX`
     - Action: `Increase Certainty Factor by 50`
   - **Rule 2**:
     - Expression: `LLDP : lldpSystemDescription CONTAINS Polycom`
     - Action: `Increase Certainty Factor by 30`

3. **Profiler CoA アクションの構成**:
   - `Work Centers` > `Profiler` > `Settings` > `Profiler CoA` に移動。
   - **Profiler CoA Type**: `Re-Auth` に設定して保存。

#### 検証方法

1. Polycom 電話機をスイッチポートに接続し、MAB 認証をトリガーします。
2. ISE Live Logs (`Operations` > `RADIUS` > `Live Logs`) で、初期接続時の Profiling 属性受領ログを確認します。
3. `Context Visibility` > `Endpoints` にて対象端末の MAC アドレスを検索し、`Endpoint Profile` が `Polycom_VVX_Phone` に遷移していること、および確信度が 80 に達していることを確認します。

---

### 演習例 3: DHCP Option 60 と MAC OUI を組み合わせた複合プロファイル設計

#### 問題
Cisco 監視カメラ（Cisco Video Surveillance IP Camera）を誤判定なく識別するため、MAC アドレスの OUI 情報と DHCP Option 60 を組み合わせた Profiling Policy を構成しなさい。

#### 要件
- ポリシー名: `Cisco_IP_Camera_Custom`
- 最小確信度: `70`
- ルール 1: MAC OUI が シスコベンダー（例: `00:1B:D4` または `00:24:C4`）に一致する場合、確信度 `40` を加算。
- ルール 2: DHCP Option 60（`dhcp-class-identifier`）が `Cisco IP Camera` で始まる場合、確信度 `50` を加算。

#### 設定例（ISE 構成パラメーター定義）

1. **Policy Name**: `Cisco_IP_Camera_Custom` (Parent: `Cisco-Device`)
2. **Minimum Certainty Factor**: `70`
3. **Rule 1 (MAC OUI Match)**:
   - `MAC : MAC Address STARTS_WITH 00:1B:D4` OR `MAC : MAC Address STARTS_WITH 00:24:C4`
   - `Certainty Factor`: `+40`
4. **Rule 2 (DHCP Option 60 Match)**:
   - `DHCP : dhcp-class-identifier STARTS_WITH Cisco IP Camera`
   - `Certainty Factor`: `+50`

#### 検証方法

1. `show device-sensor cache mac <CAMERA_MAC>` で、スイッチがカメラの DHCP Class Identifier を正常にキャプチャしているか確認します。
2. ISE の `Context Visibility` > `Endpoints` でカメラの MAC アドレスを開き、`Attributes` タブで `MACAddress` と `dhcp-class-identifier` が正しく抽出し評価されていることを確認します。

---

### 演習例 4: Profiling に基づく Voice VLAN / Data VLAN 動的変更と Port-Bounce CoA 構成

#### 問題
未識別状態のエンドポイントが接続された場合はデータ VLAN（VLAN 10）に配置し、プロファイリング処理によって `Cisco-IP-Phone` と判明した時点で音声 VLAN（VLAN 20）へ変更するために Port-Bounce CoA を発生させる設定を構成しなさい。

#### 要件
- エンドポイントは物理的に同一ポートに接続。
- VLAN 変更時に端末が確実に IP アドレスを再取得できるよう、ISE は `Port-Bounce` CoA を発行すること。
- スイッチ側で RADIUS CoA（Port-Bounce）を正常に受信処理できること。

#### 設定例

**スイッチ側 CLI 設定 (IOS-XE):**
```text
aaa server radius dynamic-author
 client 10.1.100.21 secret Cisco123!

interface GigabitEthernet1/0/10
 switchport mode access
 switchport access vlan 10
 authentication host-mode multi-domain
 authentication order dot1x mab
 authentication priority dot1x mab
 authentication port-control auto
 mab
 dot1x pae authenticator
```

**ISE 側 GUI 設定:**
1. `Work Centers` > `Profiler` > `Settings` > `Profiler CoA`
   - **CoA Type**: `Port-Bounce` を選択。
2. `Policy` > `Authorization` > `Authorization Policy`:
   - **Rule 1 (IP Phone Rule)**:
     - Condition: `IdentityGroup:Name EQUALS Endpoint Identity Groups:Cisco-IP-Phone`
     - Result: `AccessAccept` + `cisco-av-pair:device-traffic-class=voice` + `Tunnel-Private-Group-ID=20`
   - **Rule 2 (Default Rule)**:
     - Result: `AccessAccept` + `Tunnel-Private-Group-ID=10`

#### 検証方法

1. 電話機をポート `Gi1/0/10` に接続します。
2. 初期 MAB 完了時、スイッチ上で `show authentication sessions interface Gi1/0/10` を実行し、VLAN 10 が適用されていることを確認します。
3. ISE がプロファイリングを完了すると、Port-Bounce CoA が送信され、ポートが一時的に Link Down / Up します。
4. ポート復旧後、再度 `show authentication sessions interface Gi1/0/10` を実行し、VLAN 20 (Voice) が適用されていることを検証します。

---

### 演習例 5: WLC (Catalyst 9800) における Device Sensor & HTTP User-Agent プロファイリング

#### 問題
Catalyst 9800 WLC 上で無線の端末属性（DHCP および HTTP User-Agent）を Device Sensor 経由で収集し、Apple iOS デバイスを識別するプロファイリングを有効化しなさい。

#### 要件
- Catalyst 9800 上で HTTP および DHCP のスヌーピングを無効化せず有効化すること。
- WLAN Policy Profile で RADIUS Profiling 属性の通知を有効化すること。

#### 設定例

```text
! --- Catalyst 9800 CLI 設定 ---
device-sensor filter-list dhcp list WLC-DHCP-LIST
 option name class-identifier
 option name user-class-id
 option name parameter-request-list

device-sensor filter-list http list WLC-HTTP-LIST
 header name user-agent

device-sensor filter-spec dhcp include list WLC-DHCP-LIST
device-sensor filter-spec http include list WLC-HTTP-LIST

device-sensor accounting
device-sensor notify all-changes

wireless profile policy CORP-WLAN-POLICY
 aaa-override
 device-sensor dhcp
 device-sensor http-tlv
 service-policy input QOS-IN
 service-policy output QOS-OUT
 no shutdown
```

#### 検証方法

1. iPhone または iPad を無線の WLAN に接続します。
2. 9800 WLC CLI にて `show wireless client mac <MAC> detail` を実行し、`Device Sensor Data` に `User-Agent` および `DHCP Option 60` が表示されることを確認します。
3. ISE の Live Logs で `HTTP User-Agent` 属性が受信され、プロファイルが `Apple-iDevice` に自動分類されることを検証します。

---

### 演習例 6: DHCP Helper (UDP 67) による ISE へのパケット直接転送構成

#### 問題
スイッチの Device Sensor を使用せず、Catalyst レイヤ 3 スイッチの SVI（VLAN 10）から ISE PSN の DHCP プローブノードへ直接 DHCP パケットをリレーする構成を設定しなさい。

#### 要件
- クライアント VLAN: VLAN 10（サブネット: `192.168.10.0/24`）
- 本大DHCP サーバ IP: `10.1.10.254`
- ISE PSN ノード IP: `10.1.100.21`

#### 設定例

```text
! --- Catalyst L3 スイッチ CLI 設定 ---
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 10.1.10.254   ! 正規の DHCP サーバ
 ip helper-address 10.1.100.21  ! ISE PSN (DHCP Probe 兼受信用)
```

#### 検証方法

1. クライアントが DHCP Discover / Request を送信した際、ISE の CLI にて `tcpdump -i eth0 port 67` を実行し、SVI の IP (`192.168.10.1`) を送信元とする リレーパケットを受信できているかキャプチャ検証します。

---

### 演習例 7: SNMP Query Probe による他社製スイッチ接続デバイスのプロファイリング

#### 問題
Device Sensor に非対応の他社製アクセススイッチに接続されたプリンタをプロファイリングするため、ISE から定期的に SNMP Query (SNMPv2c) を発行して MAC アドレス / MIB 情報を収集する設定を構成しなさい。

#### 要件
- スイッチ IP: `10.2.20.50`
- SNMP コミュニティ名: `ReadSecCommunity`
- ISE は SNMP Query Probe を使用してスイッチの Bridge MIB (`1.3.6.1.2.1.17.4.3`) をクエリすること。

#### 設定例

1. **他社製スイッチ側設定 (SNMP 有効化)**:
   ```text
   snmp-server community ReadSecCommunity ro
   ```

2. **ISE 側 GUI 設定**:
   - `Administration` > `Network Resources` > `Network Devices`
   - `Switch-Floor2` (IP: `10.2.20.50`) を選択。
   - `SNMP Settings` タブを開く:
     - **SNMP Version**: `v2c`
     - **Read Community**: `ReadSecCommunity`
   - `Work Centers` > `Profiler` > `Settings` > `Probes`
     - **SNMP Query Probe**: 有効化

#### 検証方法

1. ISE から対話型テストを実行し、SNMP 接続テストが `Successful` になることを確認します。
2. エンドポイント接続時、ISE が SNMP でスイッチの MAC アドレス テーブルおよび CDP/LLDP MIB を引き抜き、エンドポイント属性に追加されることを確認します。

---

### 演習例 8: Profiling Policy の例外処理（Static Profile 割り当てと解除）

#### 問題
特定の重要な医療機器（IP: `10.5.1.50`, MAC: `00:50:56:A1:B2:C3`）において、Dynamic Profiling の誤判定を防止するため、プロファイル `Medical-Device-Static` を手動固定設定し、その後動的判定に戻す手順を実施しなさい。

#### 要件
- 対象 MAC `00:50:56:A1:B2:C3` に対し、Static Assignment を無効化／有効化する正しい手順を理解すること。

#### 設定例 (ISE GUI 操作手順)

1. **固定割当（Static Assignment）**:
   - `Context Visibility` > `Endpoints` にて MAC `00:50:56:A1:B2:C3` をクリック。
   - `Edit` を選択し、`Static Group Assignment` チェックボックスをオンにする。
   - `Endpoint Profile`: `Medical-Device-Static` を選択して保存。

2. **動的判定への復帰（Static 解除）**:
   - 該当エンドポイントの編集画面にて `Static Group Assignment` のチェックを外す。
   - `Save` をクリック。ネットワークから新たなパケット（DHCP Request 等）が届いた段階で Profiler Engine が再評価を開始します。

#### 検証方法

1. `Static Assignment` がオンの間は、異なった DHCP Option 60（例: Windows PC 用）を送信してもプロファイルが `Medical-Device-Static` のまま維持されることを確認します。
2. チェック解除後、属性受信に伴い動的にプロファイルが更新されることを検証します。

---

### 演習例 9: NMAP Active Scan Probe による特定ポート開放端末の補正識別

#### 問題
ISE の NMAP Probe を構成し、HTTP Probe や DHCP Probe のみでは識別が困難な特殊な Web サーバ組み込み端末（ポート 8080 が開いている端末）に対してアクティブスキャンを実行し、確信度を補正しなさい。

#### 要件
- サブネット: `172.16.50.0/24` のエンドポイントのみを対象とする。
- NMAP スキャン条件: TCP ポート `8080` が Open 状態の場合、確信度 `40` を加算。

#### 設定例

1. **ISE NMAP Probe の有効化**:
   - `Work Centers` > `Profiler` > `Settings` > `Probes` で `NMAP Scan Probe` を有効化。
2. **NMAP Action の作成**:
   - `Work Centers` > `Profiler` > `NMAP Actions` > `Add`
   - **Name**: `Scan_Web_Port_8080`
   - **Subnet**: `172.16.50.0/24`
   - **Ports**: `T:8080`
3. **Profiling Policy のルール作成**:
   - Policy `Custom_Web_Appliance` の Rule 内で:
   - `NMAP : Port : 8080 Status EQUALS Open` -> Certainty Factor: `+40`

#### 検証方法

1. 対象エンドポイントが接続された際、ISE から対象 IP へ SYN パケットが送信されることをキャプチャで確認します。
2. スキャン完了後、`Context Visibility` の NMAP 属性結果に `Port 8080: Open` が記録され、プロファイルが確定することを検証します。

---

### 演習例 10: Profiler CoA トラブルシューティング（Re-Auth / Port-Bounce 疎通検証）

#### 問題
スイッチと ISE 間で Profiler CoA が動作せず、プロファイル確定後もアクセス権限が初期状態のままになる問題を修正しなさい。

#### 要件
- スイッチ側で RADIUS Dynamic Authorization (CoA) を受信できるよう正しく設定すること。
- Port-Bounce CoA および Re-Auth CoA の双方に対応すること。

#### 設定例

```text
! --- スイッチ側 修正コンフィグ ---
aaa server radius dynamic-author
 client 10.1.100.21 secret Cisco123!
 client 10.1.100.22 secret Cisco123!
 port 1700  ! デフォルト RFC 5176 ポート (必要に応じて指定)

interface GigabitEthernet1/0/1
 description Access Port with CoA Support
 switchport mode access
 authentication port-control auto
 mab
 dot1x pae authenticator
```

#### 検証方法

1. スイッチで `debug radius dynamic-author` を有効化します。
2. ISE の `Operations` > `RADIUS` > `Live Logs` から該当セッションの `CoA` ボタンを押し、手動で `Re-Auth` または `Port-Bounce` を実行します。
3. スイッチのデバッグログで `CoA-Request received from 10.1.100.21` および `CoA-ACK sent` が表示されることを確認します。

---


## ❓ 想定試験問題

### 質問 1 (コンフィグ読解・Device Sensor)

以下の Catalyst スイッチのコンフィグレーションにおいて、Device Sensor によるプロファイリング属性が Cisco ISE に正常に送信されない原因として、最も適切な説明はどれですか？

```text
aaa new-model
aaa authentication dot1x default group ISE-GROUP
aaa authorization network default group ISE-GROUP

device-sensor filter-list dhcp list DHCP-FL
 option name class-identifier
 option name parameter-request-list

device-sensor filter-spec dhcp include list DHCP-FL
device-sensor notify all-changes
```

A. `aaa server radius dynamic-author` が投入されていないため。  
B. `aaa accounting network default start-stop group ISE-GROUP` および `device-sensor accounting` が投入されていないため。  
C. `device-sensor filter-list cdp` が定義されていないため。  
D. `radius-server attribute 31 mac format ietf upper-case` が設定されていないため。  

**正解: B**  
**解説:**  
Device Sensor は、スヌーピングして抽出した TLV 情報を RADIUS Accounting-Request パケットの VSA 属性に挿入して ISE へ送ります。スイッチに `aaa accounting network ...` および `device-sensor accounting` コマンドが設定されていない場合、属性が送信されないため ISE 側でプロファイリングできません。

---

### 質問 2 (トラブルシューティング・CoA)

Cisco ISE がプロファイリング処理によってデバイス種別を `Apple-iPad` と確定させましたが、エンドポイントの配属 VLAN が旧 VLAN から新 VLAN へ変更されず、エンドポイントも IP アドレスの再取得を開始しませんでした。ISE の Profiler CoA タイプは `Re-Auth` に設定されていました。この問題を解決するための最も効果的な対策はどれですか？

A. 認証順序（authentication order）を MAB から 802.1X に変更する。  
B. ISE の Profiler CoA タイプを `Port-Bounce` に変更する。  
C. スイッチポートに `ip dhcp snooping` を設定する。  
D. スイッチ側で RADIUS 属性 25（`radius-server attribute 25 re-evaluate`）を無効化する。  

**正解: B**  
**解説:**  
`Re-Auth` CoA は既存の EAP/MAB セッションの再認証を行いますが、リンク状態は維持されるため、多くのエンドポイント（特にモバイル端末や PC）は VLAN が変わったことを検知できず DHCP Release/Renew を発行しません。VLAN が変更されるプロファイリング構成では、`Port-Bounce` CoA を使用して物理リンクを一時的にダウンさせ、端末に DHCP 再取得を強制する必要があります。

---

### 質問 3 (Design / Architecture)

大規模ネットワーク環境において、10,000 台以上のエンドポイントのプロファイリングを実施する予定です。ネットワーク帯域および ISE の CPU 処理負荷を最小限に抑えるための最適設計として、最も推奨されるアプローチはどれですか？

A. 全アクセススイッチの全 VLAN から ISE PSN へ DHCP Relay (`ip helper-address`) を設定する。  
B. コアスイッチで全トラフィックの SPAN ポートミラーリングを設定し、ISE の Dedicated NIC に接続する。  
C. Catalyst スイッチで Device Sensor を有効化し、必要な TLV 情報のみを RADIUS Accounting パケットで ISE へ転送する。  
D. 全エンドポイントに対して 5 分間隔で NMAP Active Probe スキャンを実行する。  

**正解: C**  
**解説:**  
Device Sensor は、アクセススイッチのエッジで CDP/LLDP/DHCP パケットの TLV をスヌーピングし、差分発生時やセッション開始時のみ RADIUS Accounting パケットに載せて ISE へ転送します。無駄な DHCP パケットのコピー転送や全ポートミラーリング（SPAN）、アクティブスキャン（NMAP）に比べて、ネットワーク帯域および ISE の処理負荷を劇的に削減できるベストプラクティスです。

---

### 質問 4 (実装・Certainty Factor)

Cisco ISE の Profiling Policy における Certainty Factor（確信度）の動作説明として、正しいものはどれですか？

A. 確信度が 100 に達したプロファイルは自動的に `Static Assignment` に昇格し、以降の変更は禁止される。  
B. エンドポイントの受信属性が複数ルールにマッチした場合、各ルールで定義された Certainty Factor が加算され、合計値が判定閾値（Minimum Certainty Factor）を超えた場合に該当プロファイルが適用される。  
C. 属性の受信順序によって Certainty Factor の加算値が 2 倍になる。  
D. 一度判定されたプロファイルの Certainty Factor は減算されることはない。  

**正解: B**  
**解説:**  
ISE Profiler Engine は、マッチした各ルールの Certainty Factor ポイントを加算（条件によっては減算）し、累計ポイントが該当プロファイルの Minimum Certainty Factor（例: 60）に達した際にそのプロファイルを適用します。

---

### 質問 5 (制限事項)

Cisco ISE において、プロファイリング結果が手動で `Static Assignment`（固定割当）に設定されているエンドポイントに対する動作として、正しいものはどれですか？

A. 新たな DHCP Option 60 が受信された場合、Static Assignment は自動的に解除され動的判定に戻る。  
B. スイッチが Port-Bounce CoA を送信すると Static Assignment はリセットされる。  
C. ネットワークから新たなプロファイリング属性を受信しても、Profiler Engine はプロファイルを動的更新しない。  
D. Static Assignment が設定されたデバイスは 802.1X 認証が拒否される。  

**正解: C**  
**解説:**  
管理者が手動で `Static Assignment` を設定したエンドポイントは固定化され、その後ネットワークからどのようなプロファイリング属性（CDP, LLDP, DHCP, HTTP 等）を受信しても、動的プロファイリングエンジンによる再自動分類は行われません。


## 🔗 参考リソース

Cisco 公式ドキュメントおよび Cisco Live 資料一覧です。

- Cisco ISE Profiling Design Guide (Cisco Official Guide)
- Cisco Catalyst Device Sensor Configuration Guide (IOS-XE 17.x)
- Cisco Technical Support Note: Profiling Services via Device Sensor Troubleshooting
- Cisco Live BRKSEC-2048: Deployment and Troubleshooting of ISE Profiling Services
- Cisco Live BRKSEC-3004: Advanced Identity Services Engine (ISE) Deep Dive
- Cisco Validated Design (CVD): Secure Unified Access with Cisco ISE

---

## 📝 補足（Notes）

### 1. MAC アドレスフォーマットの標準化
スイッチ側で `radius-server attribute 31 mac format ietf upper-case` コマンドを設定しておくことで、RADIUS パケット内の Calling-Station-ID（MAC アドレス）が標準化され、ISE 上での文字列検索や Profiling Policy 評価での不一致を防止できます。

### 2. Device Sensor と CPU 保護
大規模な環境では、Device Sensor によるスヌーピングパケットの処理がスイッチ CPU に負荷を与える可能性があります。`device-sensor filter-list` に不要な Option や TLV を含めない設計を行ってください。

### 3. Prof-CoA 実行時のセッションキャッシュ動作
ISE が Port-Bounce CoA を発行すると、スイッチポートがリンクダウンするためクライアントの EAP/MAB セッションは一端消去されます。ポートリンクアップ後に新規セッションとして認証が再開され、最新の Profiling 結果に基づく認可ルールが完全に適用されます。


