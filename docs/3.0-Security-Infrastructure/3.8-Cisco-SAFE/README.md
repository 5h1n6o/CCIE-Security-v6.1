---
layout: default
title: 3.8-Cisco-SAFE
nav_order: 8
parent: 3.0-Security-Infrastructure
---

# 3.8 Cisco SAFE model to validate network security design and to identify threats to different PINs

## 📘 概要

Cisco SAFE (Security Architecture for Enterprise) は、エンタープライズネットワーク全体のセキュリティ設計を体系的に検証し、複雑化するサイバー脅威に対する包括的な防御策を構築するためのシスコ標準アーキテクチャフレームワークです。現代のネットワークは、従来の境界型防御（Perimeter Security）だけでは防ぎきれない高度標的型攻撃（APT）、ランサムウェア、内部不正、ならびにマルチクラウド環境への拡散に直面しています。Cisco SAFE モデルは、ネットワークを「Places in the Network (PINs)」と呼ばれる機能領域に分割し、それぞれの領域で発生し得る脅威モデルと必要なセキュリティ機能（Security Capabilities）を明確にマッピングすることで、設計の抜け漏れや脆弱性を防ぎます。

Cisco SAFE モデルの主な導入目的および利用シーンは以下の通りです。

* **ネットワークセキュリティ設計の標準化と検証**: 複雑なエンタープライズインフラを Campus、Branch、Data Center、Edge、WAN、Cloud などの PINs に分類し、各 PIN で実装すべき制御策（Access Control、Threat Defense、Visibility 等）を照合・検証。
* **脅威モデルの明確化とリスクアセスメント**: 偵察（Reconnaissance）、未認証アクセス、マルウェア拡散、データ窃取（Exfiltration）、DoS/DDoS 攻撃などの各段階における脅威を特定し、適切なセキュリティコンポーネントを配置。
* **Zero Trust アーキテクチャおよび Defense in Depth（多層防御）の実践**: ネットワークのあらゆる接続点（User、Device、Network、Application、Data）において「常に検証し、決して暗黙に信頼しない」原則を適用し、インフラ全体のセキュリティポスチャを強化。
* **CCIE Security ラボ試験における設計・実装指針**: 単一の製品設定にとどまらず、ISE、FTD/FMC、WSA/ESA、Stealthwatch、TrustSec、FlexVPN などの各種セキュリティソリューションを統合し、実効性の高いエンタープライズアーキテクチャを構成するための理論的・実践的基盤を提供。


---

## 🔑 要点

Cisco SAFE モデルの主要な技術的特徴、設計上の役割、対応コンポーネント、制限事項、および設計上の注意点を下表に示します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | ネットワークを機能領域（PINs）に分割し、脅威モデルとセキュリティ機能（Capabilities）を多層的にマッピングするシスコ標準の設計検証フレームワーク |
| **用途** | エンタープライズネットワーク全体のセキュリティアーキテクチャ設計、既設インフラのセキュリティポスチャ評価、脅威影響範囲の局所化（Micro-segmentation） |
| **メリット** | 複雑なセキュリティ設計の標準化・可視化、設計漏れの防止、ベンダーフリーなセキュリティ機能要求の定義、製品統合による脅威自動対応の実現 |
| **デメリット** | 完全なアーキテクチャ実装には多数の製品群（ISE、FTD、Stealthwatch、Umbrella等）の連携が必要となり、初期構築・運用設計の難易度が高まる |
| **対象コンポーネント** | Cisco Firepower (FTD/FMC), ISE, Catalyst 9000 (TrustSec/802.1X/ETA), WSA/ESA, Secure Network Analytics (Stealthwatch), Umbrella, AnyConnect |
| **制限事項** | SAFE モデル自体は概念的フレームワークであり、実際の通信制御は構成されるルータ、スイッチ、ファイアウォール、認証基盤のコンフィグおよびポリシープラグインに依存 |
| **設計上の注意点** | 特定の PIN（例: Internet Edge）だけに過度なセキュリティ投資を集中させず、Campus や Data Center 内の横展開（Lateral Movement）防止対策（SGT/VACL/PVLAN）を並行して実装すること |


---

## 🏗 動作原理

Cisco SAFE アーキテクチャは、「Places in the Network (PINs)」と「Security Capabilities（セキュリティ機能）」、およびそれらを支える「Architectural Foundations（建築的基盤ドメイン）」の相互作用によって成り立っています。

### 1. Architectural Foundations (5大基盤ドメイン)

SAFE モデルでは、すべての PINs において以下の 5 つの共通基盤ドメインが組み合わされて機能します。

1. **Management（管理ドメイン）**: デバイス構成管理、ポリシー一元配布、ロールベースアクセス制御（RBAC）、設定変更のログ監査（FMC, ISE Admin, DNA Center, TACACS+）。
2. **Visibility & Analytics（可視性と分析ドメイン）**: ネットワーク全体のトラフィック、セッション、アイデンティティ、暗号化通信の可視化（Flexible NetFlow, Encrypted Traffic Analytics (ETA), Syslog, eStreamer, Stealthwatch）。
3. **Control & Enforcement（制御と適用ドメイン）**: ユーザー/デバイスの認証・認可、ネットワークアクセス制御、マイクロセグメンテーション（802.1X, MAB, TrustSec SGT, DACL, VACL, PVLAN）。
4. **Threat Defense（脅威防御ドメイン）**: 境界および内部における未知・既知の脅威検知・遮断（NGFW/Access Control Policy, NGIPS/Snort3, Malware Defense/AMP, URL Filtering, Security Intelligence）。
5. **Identity & Access Management（アイデンティティ管理ドメイン）**: ユーザー、端末、ゲスト、IoT デバイスの識別と動的ポリシー紐付け（Cisco ISE, Active Directory, OAuth/SAML）。

### 2. Places in the Network (PINs) とトポロジフロー

エンタープライズインフラにおける代表的な PINs と、そこに配置される主要コンポーネントのトラフィック処理フローを以下に示します。

```
[ Remote / Mobile Worker PIN ]
  │ (AnyConnect / FlexVPN + Posture)
  ▼
[ Internet / Cloud Edge PIN ] ◄──► [ Cisco Umbrella / Cloud Security ]
  │ (FTD/FMC NGFW + WAF + Security Intelligence + WSA/ESA)
  ▼
[ WAN / SD-WAN PIN ] ◄──► [ Branch PIN ]
  │ (IPsec / MACsec / ZBFW)   │ (802.1X + ZBFW + Umbrella SIG)
  ▼                           ▼
[ Campus PIN ] ───────────────┤
  │ (Catalyst 9k + ISE + 802.1X + TrustSec SGT + ETA)
  ▼
[ Data Center PIN ]
  │ (Nexus / ACI / FTD Cluster + Micro-segmentation + Stealthwatch)
  ▼
[ Management Domain ] (ISE, FMC, Stealthwatch, Syslog/SNMP)
```

1. **Campus PIN**: エンドユーザーや IoT 端末が接続する領域。802.1X/MAB による認証と TrustSec (SGT) によるユーザーグループ単位のセグメンテーションを行い、内部での横展開攻撃を抑制。
2. **Internet Edge PIN**: 外部インターネットとの境界。NGFW (FTD) によるディープパケットインスペクション、NGIPS によるシグネチャマッチング、Security Intelligence (Talos IP/URL リピュテーション)、および WSA/ESA による Web/Email 保護を実施。
3. **Data Center PIN**: 基幹アプリケーションと企業データが保持される領域。ハイパフォーマンスファイアウォール（FTD クラスター）による南北トラフィック制御と、PVLAN/VACL/TrustSec による東西トラフィック（East-West）の微細セグメンテーション。
4. **Branch PIN**: 拠点領域。SD-WAN 統合セキュリティ（ZBFW, Snort, Umbrella SIG）によるダイレクトインターネットアクセス（DIA）の保護と HQ/DC への安全な IPsec トンネリング。


---

## ⚙ 動作シーケンス

Cisco SAFE モデルにおけるサイバー攻撃のライフサイクル（Cyber Kill Chain）に対する自動検知・防御・抑止の動作シーケンスを以下に示します。

```
[ 攻撃者 (Attacker) ]     [ Access Switch / FTD ]       [ Cisco ISE ]         [ Stealthwatch / FMC ]
         │                          │                       │                           │
 1. 偵察 (Reconnaissance) ────────► │                       │                           │
         │ (Port Scan / Ping Sweep) │                       │                           │
         │                          │ ─── (NetFlow / ETA) ────────────────────────────► │ 異常検知・アラート発報
         │                          │                       │                           │
 2. 端末接続 & 認証 ──────────────► │                       │                           │
         │ (802.1X / MAB Request)   │ ── (RADIUS Req) ────► │                           │
         │                          │ ◄─ (Authz: SGT=Employee, DACL) ────────────────── │ 認証成功 & SGT 付与
         │                          │                       │                           │
 3. マルウェア感染 & 横展開 ──────► │                       │                           │
         │ (Lateral Movement)       │                       │                           │
         │                          │ ── (Cisco TrustSec / SGT Check)                   │
         │                          │    [ Blocked: SGT Employee -> SGT Finance ]       │ 内部通信ブロック
         │                          │                       │                           │
 4. C2 通信 / 異常振る舞い ───────► │                       │                           │
         │                          │ ── (Flow Telemetry) ────────────────────────────► │ AI/Behavioral Analytics
         │                          │                       │                           │ 脅威検知 (High Risk)
         │                          │                       │ ◄── (pxGrid ANC Quarantine)
 5. 自動隔離 (Quarantine) ───────── │ ◄─ (CoA: Re-Auth / Shutdown) ──────────────────── │
         │                          │                       │                           │
         ▼                          ▼                       ▼                           ▼
  [ 攻撃通信の完全遮断 ]      [ ポート隔離 / Dynamic VLAN 移行 ]
```

### 処理ステップの説明

1. **偵察フェーズ（Reconnaissance）**: 攻撃者がネットワーク内部または境界からスキャンを実行すると、Catalyst スイッチおよび FTD が Flexible NetFlow (FNF) および Encrypted Traffic Analytics (ETA) データを Stealthwatch (Secure Network Analytics) に送信。機械学習エンジンがポートスキャンや異常なトラフィックパターンを即座に検知。
2. **アクセス・認証フェーズ**: 接続端末は 802.1X または MAB 経由で Cisco ISE により認証・認可され、セキュリティグループタグ（SGT: Security Group Tag）と DACL が動的に適用される。
3. **内部横展開の防止（Lateral Movement Mitigation）**: 感染端末が他のネットワークセグメント（例: 財務サーバー等）へ不正アクセスを試みた際、ネットワークスイッチおよび FTD 上の TrustSec SGACL (Security Group Access Control List) により通信が即座にドロップされる。
4. **脅威検知とオーケストレーション**: 端末が C2 (Command & Control) サーバーとの通信を試みたり、大量のデータ暗号化通信（ランサムウェアの挙動）を起こした場合、FMC または Stealthwatch が異常を特定し、Cisco pxGrid (Platform Exchange Grid) 経由で ISE に Adaptive Network Control (ANC) ポリシー発動を要求。
5. **自動封じ込め（Rapid Containment）**: ISE が Dynamic CoA (Change of Authorization) をスイッチに発行し、対象端末のポートを隔離 VLAN や Quarantine SGT へ即座に切り替えるか、ポート自体を Shutdown して感染拡大を防止する。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、Cisco SAFE モデルは単一コマンドの暗記ではなく、「エンタープライズアーキテクチャの要件を満たす多層防御コンフィグレーションおよびシステム連携」として直接問われます。各 PIN (Campus, Internet Edge, Data Center, Branch, Management) ごとに必要なセキュリティコンポーネントを正確に組み合わせ、ポリシーの漏れや不整合を排除することが合格の鍵となります。

### ラボ試験で重点的に問われるポイント

1. **PINs ごとのセキュリティ機能マッピングと設定要件**:
   * **Campus PIN**: 802.1X / MAB + IBNS 2.0 (Cisco Common Classification Policy Language: C3PL) による動的認可、TrustSec SGT インラインタギング、SXP (SGT Exchange Protocol) による IP-SGT マッピング配信、Encrypted Traffic Analytics (ETA) による暗号化パケットの平文復号なし脅威検知。
   * **Internet Edge PIN**: FTD / FMC での Access Control Policy (ACP) 設計、Prefilter Policy (Fastpath / Tunnel Zone)、Security Intelligence (IP/URL/DNS リピュテーションドロップ)、NGIPS Snort3 規則適用、WSA (Web Security Appliance) / ESA (Email Security Appliance) 連携。
   * **Data Center PIN**: FTD クラスター / ASA Routed/Transparent モードでの南北・東西トラフィック制御、VACL / PVLAN による同一 VLAN 内 L2 アイソレーション、uRPF (Strict/Loose) による送信元 IP 偽装防止。
   * **Branch / WAN PIN**: Cisco IOS-XE Zone-Based Firewall (ZBFW) の Zone/Zone-pair 定義、FlexVPN / DMVPN + IPsec (IKEv2 + AES-GCM) による拠点間暗号化、MACsec (802.1AE) による L2 暗号化。
   * **Management PIN**: CoPP (Control Plane Policing)、MPP (Management Plane Protection)、TACACS+ / RADIUS (Cisco ISE) による Device Admin (AAA)、Secure Syslog over TLS (TCP 6514)、SNMPv3 (authPriv)。

2. **Cisco 相互連携プロトコル（pxGrid / SXP / Grid / NetFlow）の実装**:
   * ISE ── FMC 間の pxGrid 連携による SGT / Identity 情報の同期および ANC (Adaptive Network Control) による検疫クアランティン自動化。
   * スイッチ ── ISE / ASA 間の SXP Peer 構築（Listener / Speaker）と TCP 6499 接続トラブルシューティング。

3. **試験で狙われやすい設定ミスと注意点**:
   * **SXP ピア接続不可**: SXP ソース IP アドレスの指定漏れ、MD5 パスワードの不一致、または中間 FW で TCP 6499 がブロックされている。
   * **ETA (Encrypted Traffic Analytics) 未検出**: スイッチ側で `ip flow-export destination` や `et-analytics` のインターフェイス有効化 (`et-analytics enable`) を忘れること。
   * **FTD Prefilter と ACP の順序誤り**: Prefilter で Fastpath された通信は Snort (NGIPS/AMP) 検査をバイパスするため、インスペクションが必要な通信を誤って Fastpath に入れないこと。
   * **CoPP / ZBFW での通信途絶**: CoPP や ZBFW ポリシーでルーティングプロトコル（OSPF/EIGRP/BGP）や IKEv2/IPsec 通信を Permit し忘れ、コントロールプレーンや VPN トンネルがダウンするミス。

4. **show コマンド・debug による状態判定能力**:
   * `show cts sxp connections`: SXP ピアのステータス（On / Pending / Inactive）と SGT マッピングテーブルの学習状況を確認。
   * `show authentication sessions interface <id> details`: 802.1X/MAB 認証ステータス、適用された SGT、DACL 名の確認。
   * `show ip cache flow` / `show flow monitor`: NetFlow および ETA エクスポート統計の確認。


---

## 🛠 設定方法

Cisco SAFE アーキテクチャを構成する代表的な PINs（Campus, Internet Edge, Data Center, Management）における具体的な Cisco IOS-XE および ASA/FTD コンフィグレーション例を以下に掲載します。

### 1. Campus PIN: IBNS 2.0 (802.1X/MAB) + TrustSec (SGT) + ETA 設定 (Cisco IOS-XE)

```text
! --- 1. TrustSec & SXP 設定 ---
cts logging verbose
cts sxp enable
cts sxp default source-interface Loopback0 password 7 Cisco123!
cts sxp connection peer 10.1.100.100 role listener password 7 Cisco123!

! --- 2. Encrypted Traffic Analytics (ETA) 設定 ---
et-analytics
 ip flow-record et-rec
 ip flow-exporter et-exp
  destination 10.1.200.50 2055
  source Loopback0
  transport udp 2055

interface GigabitEthernet1/0/10
 description Access Port to User Device
 et-analytics enable

! --- 3. IBNS 2.0 (C3PL) 認証・認可ポリシー設定 ---
class-map match-all C3PL_DOT1X
 match method dot1x
class-map match-all C3PL_MAB
 match method mab

policy-map type control subscriber C3PL_AUTHPOL
 event authentication-failure match-first
  10 action authorize
 event agent-found match-first
  10 action authenticate using dot1x
 event failure match-first
  10 action terminate mab
  20 action authorize

interface GigabitEthernet1/0/10
 switchport mode access
 switchport access vlan 10
 access-session port-control auto
 access-session control-direction in
 service-policy type control subscriber C3PL_AUTHPOL
 cts manual
  policy static sgt 15
```

### 2. Internet Edge PIN: FTD/FMC NGFW Access Control & Security Intelligence 設定

FMC (Firepower Management Center) GUI および FlexConfig / CLI による多層防御設定例：

```text
! --- FTD CLI / FlexConfig: Security Intelligence & Snort3 Engine ---
! FMC GUI: Policies > Access Control > Access Control Policy
! 1. Security Intelligence Tab:
!    - Networks: Block Blacklist_IPs (Talos Feed + Custom High-Risk Feeds)
!    - URLs: Block Malicious_URLs
! 2. Access Control Rules:
!    Rule 1: Name = Block_Malware_Exfiltration
!            Src Zone = Inside_Zone, Dst Zone = Outside_Zone
!            Inspection = IPS Policy (Balanced Security and Connectivity)
!            File Policy = Block Malware (AMP) + Malware Cloud Lookup
!            Action = Allow with Logging (Send to FMC & Syslog)

! --- FTD CLI でのドロップ統計および Snort ステータス確認 ---
> show system support engine-status
> show access-list
```

### 3. Management PIN: CoPP & MPP ハードニング設定 (Cisco IOS-XE)

```text
! --- Management Plane Protection (MPP) ---
control-plane host
 management-interface GigabitEthernet0/0 allow ssh snmp https

! --- Control Plane Policing (CoPP) ---
ip access-list extended ACL_COPP_ROUTING
 permit ospf any any
 permit udp any any eq rip
 permit tcp any any eq bgp
ip access-list extended ACL_COPP_MGMT
 permit tcp 10.1.100.0 0.0.0.255 any eq ssh
 permit udp 10.1.100.0 0.0.0.255 any eq snmp

class-map match-all CM_COPP_ROUTING
 match access-group name ACL_COPP_ROUTING
class-map match-all CM_COPP_MGMT
 match access-group name ACL_COPP_MGMT

policy-map PM_COPP
 class CM_COPP_ROUTING
  police 8000000 conform-action transmit exceed-action transmit
 class CM_COPP_MGMT
  police 1000000 conform-action transmit exceed-action drop
 class class-default
  police 80000 conform-action drop exceed-action drop

control-plane
 service-policy input PM_COPP
```


---

## 🔍 検証コマンド

Cisco SAFE アーキテクチャを構成する主要コンポーネントの状態や制御状況を検証するためのコマンド一覧を下表にまとめます。

| 目的 | コマンド | 判定基準・注目箇所 |
| --- | --- | --- |
| **802.1X/MAB 認証確認** | `show access-session interface <id> details` | Session Status = Authorize, Oper SGT, Applied DACL の印字 |
| **TrustSec SGT 確認** | `show cts environment-data` | SGT 割り当てテーブル、ISE から受信した CTS 環境データ |
| **SXP ピア状態確認** | `show cts sxp connections` | Connection State = On, SXP Version (1-5), Peer IP |
| **SXP IP-SGT マッピング** | `show cts sxp sgt-map` | IP アドレスと SGT の紐付けエントリー一覧 |
| **ETA 動作ステータス** | `show flow monitor et-mon` | ETA エクスポートパケット数、コレクター接続状態 |
| **CoPP ドロップ統計** | `show policy-map control-plane` | 各 Class-map の dropped packets カウンタの推移 |
| **uRPF 統計確認** | `show cef interface <id>` | `unicast RPF check is enabled`, Drop カウンタ |
| **FTD インスペクション** | `show system support engine-status` | Snort プロセス ID、CPU/メモリ使用率、処理パケット数 |
| **debug: RADIUS 認可** | `debug radius authentication` / `authorization` | ISE からの Access-Accept パケット、VLAN/SGT/DACL 属性 |
| **debug: SXP パケット** | `debug cts sxp messages` | SXP ピア間の Binding 共有パケット交換ログ |


---

## 🚨 トラブルシュート

Cisco SAFE モデル実装時に遭遇しやすい障害の症状、原因、確認コマンド、および対処方法を下表に整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **802.1X 成功も SGT が適用されない** | ISE 認可ポリシーで SGT 割り当てプロファイルが未設定、またはスイッチ側で `cts manual` / `cts dot1x` が未有効 | `show access-session interface <id> details` | ISE の Authorization Profile に SGT を追加し、スイッチ側で CTS 制御を有効化 |
| **SXP Connection が "Pending" から進まない** | ソース IP の誤り、中間 FW で TCP 6499 ブロック、または MD5 鍵の不一致 | `show cts sxp connections`, `debug cts sxp messages` | SXP ソース IP を Loopback に統一し、TCP 6499 を FW で許可し、双方の Password を再設定 |
| **Stealthwatch で ETA アラートが発報しない** | アクセススイッチ側で `et-analytics enable` 未設定、または Flexible NetFlow コレクター IP の誤り | `show flow monitor`, `show flow exporter` | 対象インターフェイスに `et-analytics enable` を適用し、エクスポート先 UDP 2055/2056 を確認 |
| **FTD 導入後に正規 BGP 通信が落とされる** | Access Control Policy または Prefilter Policy で BGP (TCP 179) がブロックされている | `show access-list`, `capture` | ACP に Control Traffic 許可ルールを追加し、プレフィルターで必要に応じ Fastpath 設定 |
| **CoPP 適用後に OSPF 隣接関係がフラップする** | CoPP の Class-map で IP Protocol 89 (OSPF) が Permit されず Drop されている | `show policy-map control-plane` | CoPP ACL に `permit ospf any any` を追加し、ポリスレート（police rate）を拡張 |


---

## ⚠ 制限事項

* **ハードウェア依存性**: Cisco TrustSec インラインタギングや Encrypted Traffic Analytics (ETA) などの高度なセキュリティ機能は、Catalyst 9000 シリーズなどの UADP ASIC 搭載スイッチでのみハードウェア処理可能であり、旧型スイッチではサポートされないか CPU 処理によるパフォーマンス低下を引き起こします。
* **SXP (SGT Exchange Protocol) のスケーラビリティ制限**: SXP ピアリング接続数および IP-SGT マッピングの保持上限はデバイスのプラットフォーム（スイッチ / ルータ / FW）のメモリ量に依存します。大規模構成では Cisco ISE を中心としたハブ＆スポーク階層トポロジを設計する必要があります。
* **FTD クラスタリングと非対称ルーティング制限**: FTD クラスター構成において、往路と復路が異なる FTD ユニットを通過する非対称ルーティングが発生した場合、TCP 接続のセッションテーブル不一致によりパケットがドロップされる可能性があります（Cluster Control Link (CCL) 経由でのセッション同期の適切な設計が必須）。


---

## 🔄 他技術との関連

Cisco SAFE モデルは、Blueprint 内の他の多数のセキュリティ技術を統合・包含する上位設計フレームワークです。

* **Cisco TrustSec / SGT (Blueprint 2.6)**: SAFE Campus および Data Center PIN におけるアクセス制御・マイクロセグメンテーションの中核技術。
* **802.1X / MAB / ISE (Blueprint 1.x / 2.x)**: SAFE Identity & Access Management ドメインを実現する認証・認可基盤。
* **Unicast RPF (uRPF) & BCP 38 (Blueprint 3.3.a / 3.7)**: SAFE Edge および WAN PIN における IP スプーフィング・反射攻撃防御の必須要件。
* **Flexible NetFlow / ETA / Stealthwatch (Blueprint 3.6.a)**: SAFE Visibility & Analytics ドメインを提供するトラフィック収集・異常検知基盤。
* **CoPP & MPP (Blueprint 3.1.a / 3.2)**: SAFE Management ドメインおよびコントロールプレーンのハードニング技術。
* **FlexVPN / DMVPN / MACsec (Blueprint 2.3 / 2.4 / 2.5)**: SAFE WAN 及び Branch PIN における暗号化通信インフラ。


---

## 🧩 比較表

Cisco SAFE モデルにおける主要 PINs のセキュリティ要件および他の代表的なセキュリティアーキテクチャフレームワークとの比較を以下に示します。

### 1. SAFE PINs (Places in the Network) のセキュリティ要件比較

| PIN (Place in Network) | 主な脅威 | 主なセキュリティ機能 (Capabilities) | 推奨実装コンポーネント |
| --- | --- | --- | --- |
| **Campus PIN** | 未認証デバイス接続、内部マルウェア拡散、成りすまし | 802.1X/MAB、TrustSec SGT、ETA、DHCP Snooping、DAI | Catalyst 9000、Cisco ISE、Stealthwatch |
| **Internet Edge PIN** | 外界からの侵入、DDoS、Web/Email マルウェア | NGFW、NGIPS、Security Intelligence、WAF、WSA/ESA | Cisco FTD (Firepower)、Umbrella、ESA/WSA |
| **Data Center PIN** | 東西トラフィック暗号化漏れ、特権奪取によるデータ窃取 | クラスター FW、VACL、PVLAN、マイクロセグメンテーション | Cisco FTD Cluster、Nexus、ACI、ISE |
| **Branch PIN** | ダイレクトインターネットアクセス (DIA) 脅威、拠点侵入 | SD-WAN ZBFW、Umbrella SIG、IPsec VPN | Cisco ISR/ASR (IOS-XE)、Cisco SD-WAN |

### 2. Cisco SAFE vs Zero Trust Architecture (ZTA) vs Purdue Model (ICS/OT)

| 比較項目 | Cisco SAFE Model | Zero Trust Architecture (NIST SP 800-207) | Purdue Model (ICS/OT) |
| --- | --- | --- | --- |
| **主対象** | エンタープライズ全般（Campus, DC, Edge, Cloud） | ユーザー・アイデンティティとデータリソース制御 | 産業制御システム (ICS/SCADA) および OT 網 |
| **基本コンセプト** | PINs ごとの多層防御 (Defense in Depth) | 「明示的検証」「最小権限アクセス」「侵害の想定」 | 階層化セキュリティレベル (Level 0〜5) |
| **制御軸** | ネットワーク機能領域・脅威分類・デバイス配置 | ユーザーポスチャ・コンテキスト・セッション単位認可 | 物理プロセスからエンタープライズ IT への厳格分離 |


---

## 💡 ベストプラクティス

Cisco SAFE アーキテクチャを成功させるための設計・運用上のベストプラクティスは以下の通りです。

1. **防御の深層（Defense in Depth）の実践**:
   * 単一の境界ファイアウォールに依存せず、Campus 内部（802.1X + TrustSec SGT）、Data Center 内部（PVLAN + FTD Micro-segmentation）、および Endpoint（Cisco Secure Endpoint / AMP）の各レイヤーで二重・三重の検証構造を維持すること。
2. **可視性の最大化（Visibility First）**:
   * セキュリティポリシーを閉塞（Block）モードで適用する前に、Flexible NetFlow、ETA、および Stealthwatch を全スイッチ・ルータで有効化し、正常時の通信ベースラインを最低 2〜4 週間収集・分析すること。
3. **pxGrid による自動化オーケストレーションの導入**:
   * Cisco ISE、FMC、Stealthwatch 間で pxGrid を構成し、未知の脅威や C2 通信が検知された際に、人手を介さず自動的に ANC (Adaptive Network Control) 隔離プロファイルを適用するクローズドループ応答を構築すること。


## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルの実践的な設定サンプル例を 10 個掲載します。各シナリオには問題、要件、完全な設定例（コンフィグ省略なし）、および検証方法が含まれています。

---

### サンプル例 1: Campus PIN - IBNS 2.0 (802.1X/MAB) と Dynamic SGT 割り当て及び SXP ピアリング (Cisco IOS-XE)

#### 問題
Campus PIN 内のアクセススイッチ SW1 において、IBNS 2.0 (C3PL) を使用して 802.1X および MAB 認証を構成し、認証成功時に Cisco ISE から付与された SGT (Security Group Tag) を動的に割り当ててください。また、SW1 と Core Switch SW2 間で SXP (SGT Exchange Protocol) ピアリングを構築し、IP-SGT マッピングを伝達してください。

#### 要件
1. SW1 の Gi1/0/1 で 802.1X を優先認証メソッドとし、未対応端末は MAB にフォールバックすること。
2. RADIUS サーバーとして Cisco ISE (`10.1.100.100`) を登録し、キー `CiscoSecret123!` を使用すること。
3. SW1 (SXP Connection Speaker: `10.1.10.1`) と SW2 (SXP Connection Listener: `10.1.10.2`) 間で SXP v4 ピアリングを構成し、MD5 パスワード `SxpPass123!` を設定すること。

#### 設定例

```text
! --- SW1 (Access Switch) ---
aaa new-model
aaa authentication dot1x default group radius
aaa authorization network default group radius
aaa accounting update periodic 5
aaa accounting network default start-stop group radius

radius server ISE_SERVER
 address ipv4 10.1.100.100 auth-port 1812 acct-port 1813
 key CiscoSecret123!

aaa group server radius RADIUS_SERVERS
 server name ISE_SERVER

cts logging verbose
cts sxp enable
cts sxp default source-interface Loopback0 password 7 SxpPass123!
cts sxp connection peer 10.1.10.2 role speaker version 4 password 7 SxpPass123!

ip radius source-interface Loopback0

class-map match-all CM_DOT1X
 match method dot1x
class-map match-all CM_MAB
 match method mab

policy-map type control subscriber PM_AUTH
 event authentication-failure match-first
  10 action authorize
 event agent-found match-first
  10 action authenticate using dot1x
 event failure match-first
  10 action terminate mab
  20 action authorize

interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
 access-session port-control auto
 access-session control-direction in
 service-policy type control subscriber PM_AUTH
 cts manual
  policy static sgt 10
```

#### 検証方法
* `show access-session interface GigabitEthernet1/0/1 details`: 端末の認証ステータスが `Authorized` であり、ISE から付与された SGT が正しく表示されることを確認。
* `show cts sxp connections`: SW1 と SW2 間の SXP ステータスが `Connection State = On` であることを確認。
* `show cts sxp sgt-map`: SXP 経由で学習された IP-SGT マッピングテーブルを確認。

---

### サンプル例 2: Internet Edge PIN - FTD / FMC Access Control Policy & Security Intelligence & Snort3 IPS 設定

#### 問題
Internet Edge PIN に配置された FTD (Firepower Threat Defense) において、FMC を使用して外部からの不正トラフィックを多層防御する Access Control Policy (ACP) を構築してください。

#### 要件
1. Security Intelligence (SI) により、Talos の悪質 IP ブラックリストおよびカスタム悪質 URL リストからの通信を境界で自動ドロップすること。
2. Inside から Outside への HTTP/HTTPS 通信に対して Snort3 NGIPS ポリシー (`Balanced Security and Connectivity`) および AMP (Malware Defense) を適用し、ファイルを検査・自動ブロックすること。
3. すべての拒否トラフィックおよび拒否イベントを FMC ログへ送信すること。

#### 設定例 (FMC CLI / Policy Definition Summary)

```text
! --- FMC GUI 設定構造の CLI 表現および FTD 実装コンフィグ ---
! 1. Security Intelligence Configuration (FMC Policy)
! - Feed Updates: Enable Automatic Update (Every 2 Hours)
! - Blacklist Action: Drop Network Traffic immediately on Ingress

! 2. Access Control Rule Definition:
! Rule Name: Outbound_Web_Inspection
! Source Zone: Inside_Zone
! Destination Zone: Outside_Zone
! Application: HTTP, HTTPS
! Action: Allow
! Inspection Tab:
!   - IPS Policy: Balanced Security and Connectivity (Snort3)
!   - File Policy: Block Malware All (AMP Cloud Lookup + Block Malware)
! Logging Tab:
!   - Log at End of Connection = Enabled
!   - Send Logs To = FMC Host

! --- FTD CLI での確定コンフィグ・検査確認 ---
> show access-list
> show system support engine-status
```

#### 検証方法
* `show system support engine-status`: Snort3 インスペクションエンジンが正常稼働していることを確認。
* FMC 画面 `Analysis > Vulnerabilities / Intrusion Events`: 試験用攻撃トラフィック（例: EICAR テストファイルダウンロード等）が正しく Drop され、イベントが検知されることを確認。

---

### サンプル例 3: Data Center PIN - ASA/FTD クラスター + Private VLAN (PVLAN) マイクロセグメンテーション

#### 問題
Data Center PIN 内の Web サーバー群が属する VLAN 100 において、サーバー間の横展開（East-West）攻撃を防止するため Private VLAN (PVLAN) を構成し、さらに外部からの南北トラフィックに対して ASA/FTD でセキュリティポリシーを適用してください。

#### 要件
1. VLAN 100 を Primary VLAN とし、VLAN 101 を Isolated VLAN、VLAN 102 を Community VLAN としてマッピングすること。
2. スイッチ SW1 の Gi1/0/1（ゲートウェイ FW 接続口）を Promiscuous ポートとし、Gi1/0/2〜3（Web サーバー口）を Isolated ポートとすること。
3. FW 側インターフェイスで Ingress/Egress トラフィックに対してステートフルインスペクションを有効化すること。

#### 設定例

```text
! --- SW1 (Data Center Switch) ---
vlan 101
 private-vlan type isolated
vlan 102
 private-vlan type community
vlan 100
 private-vlan type primary
 private-vlan association 101,102

interface GigabitEthernet1/0/1
 description Connected to DC Firewall Gateway
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 100 add 101,102

interface GigabitEthernet1/0/2
 description Web Server 1
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101

interface GigabitEthernet1/0/3
 description Web Server 2
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101

! --- ASA / FTD Gateway Configuration ---
interface GigabitEthernet0/0
 nameif DC_Inside
 security-level 100
 ip address 10.100.1.254 255.255.255.0

access-list DC_IN_ACL extended permit tcp any 10.100.1.0 255.255.255.0 eq www
access-list DC_IN_ACL extended permit tcp any 10.100.1.0 255.255.255.0 eq 443
access-group DC_IN_ACL in interface DC_Inside
```

#### 検証方法
* `show vlan private-vlan`: Primary VLAN 100 と Secondary VLAN 101/102 のバインド状態を確認。
* Web Server 1 (Gi1/0/2) から Web Server 2 (Gi1/0/3) への Ping 通信を実行し、同じ VLAN 内であっても Isolated ポート間で Ping がドロップすることを確認。

---

### サンプル例 4: Branch PIN - Cisco IOS-XE Zone-Based Firewall (ZBFW) & FlexVPN 双方向暗号化

#### 問題
Branch PIN のルータ BR1 において、ダイレクトインターネットアクセス（DIA）トラフィックを防御する Zone-Based Firewall (ZBFW) を構築し、HQ への接続用に FlexVPN (IKEv2/IPsec) トンネルを設定してください。

#### 要件
1. Security Zone `LAN_ZONE` (Inside) と `WAN_ZONE` (Outside) を作成すること。
2. `LAN_ZONE` から `WAN_ZONE` への HTTP, HTTPS, DNS, ICMP 通信のみを Inspect（ステートフル監視）し、反対方向の未認証通信を自動ブロックすること。
3. HQ 向けの FlexVPN トンネル通信（IKEv2: UDP 500/4500, IPsec: ESP）を ZBFW の例外として通過許可すること。

#### 設定例

```text
! --- BR1 (Branch Router) ---
crypto ikev2 proposal IKEV2_PROP
 encryption aes-gcm-256
 prf sha384
 group 19

crypto ikev2 policy IKEV2_POL
 proposal IKEV2_PROP

crypto ikev2 keyring KEYRING
 peer HQ_HUB
  address 199.1.1.1
  pre-shared-key CiscoBranchKey123!

crypto ikev2 profile IKEV2_PROF
 match identity remote address 199.1.1.1 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYRING

crypto ipsec profile IPSEC_PROF
 set ikev2-profile IKEV2_PROF

interface Tunnel1
 ip address 172.16.1.2 255.255.255.0
 tunnel source GigabitEthernet0/0/0
 tunnel destination 199.1.1.1
 tunnel protection ipsec profile IPSEC_PROF

! --- ZBFW Configuration ---
zone security LAN_ZONE
zone security WAN_ZONE

ip access-list extended ACL_OUTBOUND_PASS
 permit tcp any any eq www
 permit tcp any any eq 443
 permit udp any any eq domain
 permit icmp any any

class-map type inspect match-any CM_OUTBOUND
 match access-group name ACL_OUTBOUND_PASS

policy-map type inspect PM_LAN_TO_WAN
 class CM_OUTBOUND
  inspect
 class class-default
  drop

zone-pair security ZP_LAN_WAN source LAN_ZONE destination WAN_ZONE
 service-policy type inspect PM_LAN_TO_WAN

interface GigabitEthernet0/0/1
 description Branch LAN
 zone-member security LAN_ZONE

interface GigabitEthernet0/0/0
 description Internet WAN
 zone-member security WAN_ZONE
```

#### 検証方法
* `show zone-pair security`: Zone-pair の定義とサービスポリシーの適用状態を確認。
* `show policy-map type inspect zone-pair`: インスペクションテーブル内のアクティブセッション数とパケットカウントを確認。

---

### サンプル例 5: Management Domain PIN - CoPP (Control Plane Policing) & MPP ハードニング

#### 問題
コアコアスイッチ/ルータ R1 において、コントロールプレーンに対する DoS 攻撃を防ぐ CoPP (Control Plane Policing) と、特定インターフェイスからの管理通信のみを許可する MPP (Management Plane Protection) を構成してください。

#### 要件
1. MPP により、Gi0/0 (Out-of-band Management) からのみ SSH (TCP 22) および SNMP (UDP 161) を許可すること。
2. CoPP により、BGP (TCP 179) 通信を最高優先度（8 Mbps）、SSH/SNMP を中優先度（1 Mbps）、その他の制御通信を低優先度（80 kbps）でポリシングし、過大なトラフィックをドロップすること。

#### 設定例

```text
! --- R1 (Management Hardening) ---
! 1. MPP 設定
control-plane host
 management-interface GigabitEthernet0/0 allow ssh snmp

! 2. CoPP ACL & Class-map 定義
ip access-list extended ACL_ROUTING_PROTO
 permit tcp any any eq bgp
 permit tcp any eq bgp any
 permit ospf any any

ip access-list extended ACL_MGMT_PROTO
 permit tcp 10.1.100.0 0.0.0.255 any eq ssh
 permit udp 10.1.100.0 0.0.0.255 any eq snmp

class-map match-all CM_ROUTING
 match access-group name ACL_ROUTING_PROTO

class-map match-all CM_MGMT
 match access-group name ACL_MGMT_PROTO

! 3. Policy-map 定義 & コントロールプレーン適用
policy-map PM_COPP_POLICY
 class CM_ROUTING
  police 8000000 conform-action transmit exceed-action transmit
 class CM_MGMT
  police 1000000 conform-action transmit exceed-action drop
 class class-default
  police 80000 conform-action drop exceed-action drop

control-plane
 service-policy input PM_COPP_POLICY
```

#### 検証方法
* `show control-plane host`: MPP で許可された管理インターフェイスとプロトコル一覧を確認。
* `show policy-map control-plane`: コントロールプレーンポリシーの各クラスにおける一致パケット数および Policing ドロップ数をリアルタイムで確認。

---

### サンプル例 6: Visibility PIN - Encrypted Traffic Analytics (ETA) & Flexible NetFlow (FNF) 設定

#### 問題
Campus PIN のコアスイッチ SW1 において、暗号化通信の復号を行わずにマルウェアの挙動を検知するため、Encrypted Traffic Analytics (ETA) と Flexible NetFlow (FNF) を構成し、Stealthwatch Collector へ送信してください。

#### 要件
1. 送信元 IP、宛先 IP、L4 プロトコル、TCP フラグ、および ETA 固有の暗号化パケット情報（Initial Data Packet: IDP、Sequence of Packet Lengths and Times: SPLT）を収集すること。
2. コレクター IP `10.1.200.50` (UDP ポート 2055) へ NetFlow v9 / IPFIX 形式でエクスポートすること。

#### 設定例

```text
! --- SW1 (Flow & ETA Configuration) ---
flow record REC_ETA_FNF
 match ipv4 source address
 match ipv4 destination address
 match ip protocol
 match ip tos
 match transport source-port
 match transport destination-port
 collect flow creation-time
 collect flow extension et-analytics

flow exporter EXP_STEALTHWATCH
 destination 10.1.200.50
 source Loopback0
 transport udp 2055
 template data timeout 60

flow monitor MON_ETA
 record REC_ETA_FNF
 exporter EXP_STEALTHWATCH

et-analytics
 ip flow-record REC_ETA_FNF
 ip flow-exporter EXP_STEALTHWATCH

interface GigabitEthernet1/0/1
 description Access Layer Link
 ip flow monitor MON_ETA input
 et-analytics enable
```

#### 検証方法
* `show flow monitor MON_ETA`: キャッシュ内のアクティブフロー数を確認。
* `show et-analytics global`: ETA 機能のグローバル動作ステータスとエクスポート統計を確認。

---

### サンプル例 7: WAN PIN - 点対点 802.1AE MACsec (MKA) レイヤー 2 暗号化設定

#### 問題
WAN PIN において、拠点間 L2 リンク（SW1 ── SW2 間）の盗聴・改ざんを防ぐため、802.1X MKA (MACsec Key Agreement) プロトコルを用いた 802.1AE MACsec 暗号化を設定してください。

#### 要件
1. プレシェアード Pre-Shared Key (CAK/CKN) 方式を使用して MKA 鍵交換を実行すること。
2. CKN (32 文字 Hex) および CAK (32 文字 Hex) を設定し、256 ビット AES-GCM (GCM-AES-256) 暗号スィートを強制すること。

#### 設定例

```text
! --- SW1 & SW2 共通設定 ---
mka keychain MK_KEYCHAIN macsec
 key 0123456789ABCDEF0123456789ABCDEF
  cryptographic-algorithm aes-256
  mka-string 0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF

macsec profile MACSEC_PROFILE
 cipher-suite gcm-aes-256
 replay-protection window-size 64

interface GigabitEthernet1/0/48
 description Point-to-Point WAN Link
 macsec network-link
 macsec profile MACSEC_PROFILE
 mka keychain MK_KEYCHAIN
```

#### 検証方法
* `show macsec summary`: インターフェイス上で MACsec が `Secured` 状態でアクティブか確認。
* `show mka sessions`: MKA セッション状態が `Secured` であり、暗号化アルゴリズムが `GCM-AES-256` であることを確認。

---

### サンプル例 8: Cloud Edge PIN - WSA Transparent Proxy WCCPv2 統合設定

#### 問題
Internet Edge / Cloud PIN に配置されたルータ R1 と WSA (Web Security Appliance) 間で WCCPv2 (Web Cache Communication Protocol) を構成し、HTTP (TCP 80) トラフィックを WSA へ透明（Transparent）にリダイレクトして Web 検査を行ってください。

#### 要件
1. WCCP Service Group `90` を使用し、Password `WccpPass123!` による認証を構成すること。
2. R1 の Gi0/1 (Inside LAN) にて Ingress リダイレクトを適用すること。

#### 設定例

```text
! --- R1 (WCCP Router) ---
ip wccp 90 password S3cur3Wccp!

access-list 100 permit ip 10.1.10.0 0.0.255.255 any

ip wccp 90 redirect-list 100

interface GigabitEthernet0/1
 description Inside LAN
 ip wccp 90 redirect in

! --- WSA Side Configuration (WSA GUI / CLI) ---
! System Administration > WCCP Services
! Service Group ID: 90
! Router IP: 10.1.1.1
! Authentication Password: S3cur3Wccp!
! Forwarding Method: GRE / L2
```

#### 検証方法
* `show ip wccp 90 detail`: WCCP サービスグループ 90 のアクティブな WSA ルータ/キャッシュエンジン情報およびリダイレクトパケットカウントを確認。

---

### サンプル例 9: Remote Worker PIN - AnyConnect SSL-VPN & ISE Posture Check 設定

#### 問題
Remote Worker PIN の ASA/FTD において、AnyConnect SSL-VPN 接続時に Cisco ISE と連携して端末ポスチャチェック（アンチウイルスソフト適用状況確認）を実行し、非準拠端末を隔離する設定を行ってください。

#### 要件
1. ASA/FTD 上に AnyConnect SSL-VPN ゲートウェイを構成すること。
2. ISE 認可ポリシーにおいて、Posture Status = `Compliant` の場合は Full Access、`Non-Compliant` の場合は Remediate DACL (Web Redirect to ISE Portal) を動的付与すること。

#### 設定例 (ASA Configuration)

```text
! --- ASA SSL-VPN Gateway ---
ip local pool VPN_POOL 192.168.100.1-192.168.100.254 mask 255.255.255.0

group-policy GP_REMOTE_WORKER internal
group-policy GP_REMOTE_WORKER attributes
 dns-server value 10.1.100.10
 vpn-tunnel-protocol ssl-client

tunnel-group TG_REMOTE_WORKER type remote-access
tunnel-group TG_REMOTE_WORKER general-attributes
 default-group-policy GP_REMOTE_WORKER
 address-pool VPN_POOL
 authentication-server-group ISE_RADIUS_GROUP
 aaa-authorization-required

aaa-server-host (inside) host 10.1.100.100
 key CiscoSecret123!
```

#### 検証方法
* `show vpn-sessiondb detail anyconnect`: アクティブな AnyConnect セッション、割り当てられた IP アドレス、および ISE から適用された Dynamic ACL / Dynamic Group 属性を確認。

---

### サンプル例 10: Automated Threat Defense PIN - FMC Correlation Policy & pxGrid ANC 隔離設定

#### 問題
FMC (Firepower Management Center) において、特定の C2 (Command & Control) サーバー通信イベントが発生した際、Cisco pxGrid を介して Cisco ISE へシグナルを送り、感染端末をネットワークから自動隔離（ANC Quarantine）する相関ポリシー（Correlation Policy）を構成してください。

#### 要件
1. FMC の Correlation Rule にて「Intrusion Event: Priority = High」または「Malware Event: Action = Block」を検知条件とすること。
2. リメディエーションアクション（Remediation Action）として pxGrid 経由で ISE ANC Policy `Quarantine` を適用すること。

#### 設定例 (FMC & ISE Policy Logic Summary)

```text
! --- FMC Correlation Policy Definition ---
! 1. Create Rule: C2_Infection_Detected
!    - Event Type: Intrusion Event
!    - Impact Flag: Red / High
!
! 2. Create Policy: Auto_Quarantine_Policy
!    - Add Rule: C2_Infection_Detected
!    - Assign Response: pxGrid ANC Remediation -> Action: Quarantine
!
! --- ISE ANC Profile Definition (ISE GUI) ---
! Operations > Adaptive Network Control > Policies
! Policy Name: Quarantine
! Action: Shutdown Port / Apply Dynamic DACL (Block All Except Remediation Server)
```

#### 検証方法
* 攻撃端末からテスト侵入パケットを送出し、FMC でハイインパクト侵入イベントを発生させる。
* ISE 画面 `Operations > ANC > Session Status` にて対象端末の MAC アドレスが自動的に `Quarantine` 状態へ変更され、アクセススイッチ側で Change of Authorization (CoA) パケットを受信してポートが隔離されることを確認。


## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験および筆記試験（SCOR 350-701）を想定した解説付き演習問題を 6 問掲載します。

---

### 問題 1 (Design & Architecture)

**質問**:
大規模エンタープライズにおいて、Cisco SAFE モデルに基づいて Data Center PIN と Campus PIN 間のトラフィック制御およびアイデンティティ継承を設計しています。データセンター内の仮想化サーバー（East-West トラフィック）間での不審通信の拡散を最小化し、かつキャンパスのユーザーロール（SGT）を継承してアクセス制御を行うための最適な構成手法の組み合わせはどれですか？

A. Data Center 境界に ASA 透過モードを配置し、すべての内部通信を BGP 経由で強制ルーティングする。  
B. Campus スイッチと DC スイッチ間で SXP (SGT Exchange Protocol) を構成して IP-SGT マッピングを同期し、DC 内で Private VLAN (PVLAN) および TrustSec SGACL を併用する。  
C. Campus スイッチで 802.1Q トランクを構成し、Data Center 内の全 VLAN を単一の L2 ルーティングドメインに統合する。  
D. Data Center 境界の FTD で全ポートを Fastpath に設定し、パフォーマンスを最優先化する。

**正解**: B

**解説**:
Cisco SAFE アーキテクチャにおいて、Data Center PIN のマイクロセグメンテーション（East-West 制御）には Private VLAN (PVLAN) や TrustSec (SGT) の使用が推奨されます。また、Campus PIN で認証されたユーザーの SGT 情報を Data Center 側のインフラ（Nexus/ACI/FTD 等）へ伝達するために SXP (SGT Exchange Protocol) ピアリングを構築することで、IP-SGT マッピングが伝達され、ロールベースの動的アクセス制御（SGACL）が可能になります。

---

### 問題 2 (トラブルシューティング - SXP ピアリング)

**質問**:
エンジニアが SW1 (SXP Speaker) と SW2 (SXP Listener) 間で SXP 接続を設定しましたが、`show cts sxp connections` コマンドを実行したところ、ステータスが `Pending-On` のまま `On` に移行しません。原因として最も考えられるものはどれですか？

A. SXP バージョンが SW1 で Version 2、SW2 で Version 4 に設定されており、上位互換性がない。  
B. SW1 と SW2 の間で TCP ポート 6499 通信が中間ファイアウォールによってドロップされているか、SXP パスワードが一致していない。  
C. SW1 のインターフェイスで `cts manual` が設定されていない。  
D. Cisco ISE が SXP ピアとして登録されていない。

**正解**: B

**解説**:
SXP は TCP ポート 6499 を使用してピア間の接続を確立し、認証キー（Password）を使用してハンドシェイクを行います。ステータスが `Pending-On` で停止している場合、TCP ポート 6499 通信が中間デバイス（FW等）で拒否されているか、設定された SXP Password が両者間で不一致である可能性が最も高い原因です。

---

### 問題 3 (コンフィグ読解 - CoPP & ZBFW)

**質問**:
以下の Cisco IOS-XE ルータのコンフィグを分析してください。この設定に関する記述のうち正しいものはどれですか？

```text
ip access-list extended ACL_COPP
 permit tcp any any eq bgp
 permit ospf any any

class-map match-all CM_COPP
 match access-group name ACL_COPP

policy-map PM_COPP
 class CM_COPP
  police 8000000 conform-action transmit exceed-action drop
 class class-default
  police 8000 conform-action drop exceed-action drop

control-plane
 service-policy input PM_COPP
```

A. ルータ宛てのすべての SSH 接続および SNMP 監視通信が `class-default` にマッチして 80 kbps でドロップされる。  
B. BGP および OSPF パケットは制限なしに無制限で許可される。  
C. このルータを通過する（Transit）データ通信がすべてドロップされる。  
D. `control-plane` にポリサーを適用したため、ルータの全物理インターフェイスが Shutdown になる。

**正解**: A

**解説**:
`ACL_COPP` には BGP (TCP 179) と OSPF (IP Protocol 89) のみが定義されているため、ルータ自身宛て（コントロールプレーン宛て）の SSH (TCP 22) や SNMP (UDP 161) 等の管理通信はすべて `class-default` にマッチします。`class-default` では `police 8000 conform-action drop` （または 80 kbps 制限かつ超過ドロップ）と記述されているため、管理通信パケットは即座にドロップまたは極度に制限され、SSH/SNMP アクセスが不能になります（管理アクセス用の Class-map を別途明示的に作成して許可する必要があります）。

---

### 問題 4 (Implementation - Encrypted Traffic Analytics)

**質問**:
Cisco SAFE モデルの Visibility & Analytics ドメインにおいて、Encrypted Traffic Analytics (ETA) を適用して暗号化パケット内の不審要素を検知するための必須コンポーネントと前提条件の組み合わせはどれですか？

A. 暗号化通信を一時的に平文復号するための SSL/TLS Proxy 証明書と WSA (Web Security Appliance)。  
B. Flexible NetFlow (FNF) をサポートするスイッチ（Catalyst 9000 等）、ETA 対応 IOS-XE ソフトウェア、および Cisco Secure Network Analytics (Stealthwatch)。  
C. FTD 上での Fastpath プレフィルタールールと IPsec トンネルの解除。  
D. Cisco ISE における TACACS+ コマンドセットと 802.1X ポスチャエージェント。

**正解**: B

**解説**:
ETA (Encrypted Traffic Analytics) は、暗号化通信を平文復号することなく、パケットの初期データ（Initial Data Packet: IDP）およびパケットサイズ・送信タイミグのシーケンス（Sequence of Packet Lengths and Times: SPLT）をメタデータとして FNF 経由で収集し、Cisco Secure Network Analytics (Stealthwatch) の暗号解析分析エンジンへ送信して異常を検知する技術です。したがって、ETA 対応スイッチと FNF、および Stealthwatch が必須コンポーネントとなります。

---

### 問題 5 (Threat Defense - FTD & Security Intelligence)

**質問**:
Internet Edge PIN において Cisco FTD を運用しています。Talos から提供される最新の IP/URL リピュテーションフィードに基づいて、Snort インスペクションエンジンによる処理を開始する前の最も初期の段階で通信を自動ドロップさせるために設定すべき機能はどれですか？

A. Access Control Policy (ACP) の File Policy  
B. Security Intelligence (SI)  
C. Intrusion Policy (IPS Rules)  
D. Identity Policy  

**正解**: B

**解説**:
Security Intelligence (SI) は、FTD の Access Control Policy において、パケットが重い Snort IPS インスペクションやファイル分析へ送出される前の段階（Ingress 早期段階）で、既知のハイリスクな IP アドレス、ドメイン名、URL を高速にドロップする機能です。これにより、ファイアウォール CPU の無駄な消費を抑え、効率的な境界防御を実現します。

---

### 問題 6 (Automation & Orchestration)

**質問**:
Cisco SAFE モデルにおける「自動応答（Automated Mitigation）」を実現するため、FMC で検知された重大なセキュリティイベントをトリガーとして、Cisco ISE が対象端末のアクセスポートを隔離（Quarantine）する際に使用されるプロトコルおよびメカニズムの組み合わせはどれですか？

A. NetFlow v9 と SNMP Trap  
B. Cisco pxGrid (Platform Exchange Grid) と RADIUS Dynamic CoA (Change of Authorization)  
C. SXP (SGT Exchange Protocol) と TACACS+ Authorize  
D. WCCPv2 と BGP Flowspec  

**正解**: B

**解説**:
FMC や Stealthwatch などの検知ソリューションは、Cisco pxGrid を介して Cisco ISE とリアルタイムにコンテキスト情報を共有します。セキュリティイベントが検知されると、FMC/Stealthwatch は pxGrid を通じて ISE へ ANC (Adaptive Network Control) ポリシー適用を要求し、ISE はスイッチに対して RADIUS Dynamic CoA (Change of Authorization) を発行してポート隔離や Dynamic VLAN 移行を実行します。


---

## 🔗 参考リソース

Cisco SAFE モデルおよび関連アーキテクチャ設計に関する Cisco 公式リソースの一覧です。

* Cisco SAFE Architectural Guide (Official Overview)
* Cisco SAFE Design Guide & Places in the Network (PINs)
* Cisco Validated Designs (CVD) for Security
* Cisco TrustSec Design and Implementation Guide
* Encrypted Traffic Analytics (ETA) Deployment Guide
* Cisco Secure Firewall (FTD/FMC) 7.0 Configuration Guide
* Cisco Identity Services Engine (ISE) 3.1 Administrator Guide
* Control Plane Policing (CoPP) Configuration Guide - Cisco IOS-XE
* Cisco Live Presentation: Enterprise Security Architecture with Cisco SAFE


---

## 📝 **補足（Notes）**

### Cisco SAFE モデルをラボ試験・実務設計で成功させるための 3 大ルール

1. **"Never Rely on a Single PIN Barrier"（単一境界への過信排除）**:
   * インターネット境界（Internet Edge PIN）をどれほど強固にしても、持ち込み端末（BYOD）や VPN 経由の接続、USB メモリ等による内部侵入は防止できません。Campus 及び Data Center PIN における TrustSec SGT / PVLAN / 802.1X の徹底的な適用（Zero Trust）を怠らないこと。
2. **"Correlate Telemetry with Action"（テレメトリと自動対応の連動）**:
   * 単に Syslog や NetFlow を集めるだけでは「受動的な監視」にとどまります。SAFE の真価は、Stealthwatch / FMC の検知と ISE pxGrid (ANC) の連携による「アクティブな自動封じ込め」にあります。
3. **"Protect the Control and Management Planes First"（制御・管理プレーンの先行的保護）**:
   * データプレーンのセキュリティ機能（FW/IPS/SGT）がどれほど優れていても、CoPP や MPP、TACACS+ によるルータ・スイッチ自体の保護が欠落していれば、コントロールプレーンへの過剰負荷によってシステム全体が停止します。管理・制御プレーンのハードニングを最優先で確立してください。
