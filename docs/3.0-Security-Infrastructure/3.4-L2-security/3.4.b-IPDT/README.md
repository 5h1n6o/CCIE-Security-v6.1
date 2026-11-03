---
layout: default
title: 3.4.b-IPDT
nav_order: 2
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.4.b IPDT

## 📘 概要

IP Device Tracking（IPDT、または Cisco IOS-XE における SISF: Switch Integrated Security Features ベースの Device Tracking）は、Layer 2 アクセススイッチが自身に接続されている全エンドポイントの MAC アドレス、IP アドレス、接続スイッチポート、所属 VLAN などのバインディング情報（IP-MAC Address Binding Table）を自動的に収集・追跡・維持するためのインフラストラクチャ機能です。

単なる接続機器の把握にとどまらず、Cisco Identity Services Engine（ISE）による Identity-Based Networking（802.1X, MAB, WebAuth）、Cisco TrustSec（SGT アサインメント）、Dynamic ARP Inspection（DAI）、IP Source Guard（IPSG）、ならびに IPv6 First Hop Security（FHS）の正確な動作を実現するための必須インフラ技術として利用されます。

*   **機能概要**: スイッチポートを通過する ARP、DHCP、IPv6 Neighbor Discovery (ND) などのコントロールプレーンパケットをパッシブにスヌーピングするとともに、必要に応じてアクティブな ARP Probe パケットを送出してエンドポイントの IP-MAC バインディングデータベースを動的に生成・保持します。
*   **利用目的**: エンドポイントの IP アドレス変更や移動をリアルタイムに検出し、認証サーバ（ISE）への属性通知、セキュリティポリシー（VACL/ACL/SGT）の動的適用、および IP スプーフィング攻撃の防御を行うための基盤データベースを提供します。
*   **どのような場面で利用するか**: 802.1X や MAB などの認証ソリューションを導入するネットワーク、Cisco ISE によるエンドポイントプロファイリング環境、およびデータセンターやエンタープライズのアクセス層セキュリティを強化する構成において必須となります。

---

## 🔑 要点

IP Device Tracking (IPDT) の主要な要点を下表に整理します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | L2 スイッチポートでパケット（DHCP, ARP, IPv6 ND）を常時スヌーピングし、MAC-IP-Port-VLAN バインディングテーブルを保持。定期的な ARP Probe による活性確認（Keepalive）を実施 |
| **用途** | Cisco ISE 連携（IP アドレスプロファイリング/CoA Trigger）、TrustSec SGT 動的バインディング、DAI / IP Source Guard のバインディングソース、802.1X/MAB セッション管理 |
| **メリット** | 静的 IP 端末および DHCP 端末の両方において、IP アドレスの割り当て状況と物理ポートの紐付けを透過的かつ高精度に追跡可能 |
| **デメリット** | スイッチのデフォルト設定のまま有効化すると、Windows 端末が自端末の IP アドレス重複（Duplicate IP Address 0.0.0.0）を誤検知して NIC を切断する重大なトラブルが発生するリスクがある |
| **対応機種** | Catalyst 2960/3560/3750/3850/4500/6500 シリーズ（Legacy IPDT CLI）、Catalyst 9000 シリーズおよび Cisco IOS-XE 16.x 以降（SISF Device Tracking CLI） |
| **制限事項** | スイッチ間 Trunk ポートやルータ接続 Uplink ポートで無制限に有効化すると、対向スイッチの IP や他セグメントの IP をテーブルに大量学習し、リソース枯渇やルーティング障害を引き起こす |
| **設計上の注意点** | Windows 端末の IP 重複検知問題を防止するため、`ip device tracking probe delay 10` および `ip device tracking probe auto-source`（または SISF における同等のポリシー）の適用が必須 |

---

## 🏗 動作原理

IP Device Tracking は、パッシブスヌーピング（DHCP/ARP/ND）とアクティブプロービング（ARP Probe）を組み合わせて動作します。

```
[ エンドポイント ]
       │
       │  1. DHCP Request / ARP Request 発行
       ▼
[ Access Port (IPDT Enabled) ]
       │  2. パケットをスヌーピングし、IP-MAC-Port-VLAN をテーブルに登録
       │  3. バインディングエントリの動作状態を ACTIVE に更新
       ▼
[ IPDT Database Table ]
       │
       │  4. 定期的に活性確認（ARP Probe: Src IP=0.0.0.0 or Auto-Source IP, Src MAC=Switch MAC, Dst IP=Target IP）
       ▼
[ エンドポイント ]  ── 5. ARP Reply 返答 ──▶ [ Active 状態を維持 ]
```

### Windows 重複 IP アドレス（Duplicate IP 0.0.0.0）検知問題と解決メカニズム

IPDT を有効化したスイッチにおいて、最も頻出する設計トラブルが Windows OS 端末における重複 IP アドレス警告（`Microsoft-Windows-TCPIP` イベント ID 4199）と NIC の自動無効化です。

1.  **問題発生のメカニズム**:
    *   Windows 端末はリンクアップ直後、RFC 5227 に基づき、自端末に割り当てられた IP アドレスがネットワーク内で重複していないか確認するために「RFC 5227 ARP Announce / Probe」を送出します。
    *   同時期に、アクセススイッチの IPDT 機能も新規端末を検出して「ARP Probe（送信元 IP: `0.0.0.0`、送信元 MAC: スイッチの MAC、宛先 IP: 検出した端末の IP）」を送出します。
    *   Windows 端末は、自分が使用しようとしている IP アドレスを宛先とし、送信元 IP が `0.0.0.0` である ARP パケットを外部（スイッチ）から受信すると、**「ネットワーク上に既に自端末の IP アドレスを使用またはテストしている他装置が存在する」**と誤認し、IP アドレスの利用を中断して NIC を無効化します。

2.  **回避メカニズム（Probe Delay & Auto-Source）**:
    *   **Probe Delay (`ip device tracking probe delay 10`)**: リンクアップ直後の 10 秒間、スイッチからの ARP Probe 送出を遅延させます。これにより、Windows 端末が自身の DAD (Duplicate Address Detection) 処理を安全に完了できるようにします。
    *   **Auto-Source (`ip device tracking probe auto-source`)**: スイッチが送出する ARP Probe の送信元 IP アドレスを `0.0.0.0` ではなく、非ゼロの擬似 IP（例: `192.0.2.1` など無効な未利用 IP、あるいは SVI の IP アドレス）に置き換えます。送信元 IP が `0.0.0.0` でなくなるため、Windows 端末の RFC 5227 誤検知トリガーを完全に回避できます。

---

## ⚙ 動作シーケンス

IPDT および SISF Device Tracking におけるパケット処理およびエントリ管理の順序は以下の通りです。

1.  **リンクアップとパケット受信**:
    エンドポイントがポートに接続され、DHCP Discover/Request、ARP Request、または IPv6 RS/NS パケットを送信します。
2.  **スヌーピングとデータベース登録**:
    スイッチの ASIC / Control Plane は受信したパケットの L2 MAC アドレス、L3 IP アドレス、受信 VLAN、および物理ポート番号を照合し、IPDT データベース（または SISF Binding Table）に `TENTATIVE` または `ACTIVE` 状態としてエントリを作成します。
3.  **ISE / 上位機能への通知**:
    新規 IP-MAC バインディングが確定すると、スイッチは Cisco ISE に対し RADIUS Accounting や Cisco RADIUS Attributes（`cisco-av-pair: framed-ip-address`）経由で端末の IP アドレス確定を通知します。
4.  **アクティブ保持とプロービング (Keepalive)**:
    バインディングエントリのタイマーが経過すると、スイッチはエンドポイント宛てに unicast / broadcast ARP Probe を送出し、応答の有無を確認します。応答を受信した場合はエントリを `ACTIVE` に維持します。
5.  **エントリの熟成と削除 (Aging & Deletion)**:
    端末が切断されたり、ARP Probe に対する応答が一定回数失敗した場合、エントリは `INACTIVE` 状態を経てデータベースから削除されます。スイッチは必要に応じて ISE に通知を行い、認証セッションのクリアや CoA（Change of Authorization）を誘導します。




## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、IPDT / Device Tracking は単独で問われるだけでなく、ISE 連携や DAI/IPSG の前提条件として非常に重要な位置を占めます。

*   **Blueprint で重要なポイント**:
    *   Cisco ISE によるプロファイリングおよび CoA の正常動作に必要な `framed-ip-address` 取得のための IPDT 有効化。
    *   Windows 端末の IP アドレス重複障害を防ぐための Probe Delay および Probe Auto-Source の正確な実装。
    *   Legacy IPDT CLI（IOS）と SISF Device Tracking CLI（IOS-XE 16.x/17.x）のコマンド移行と正しい設定箇所の区別。
    *   スイッチ間 Trunk ポートおよび Upstream ルータ接続ポートでの追跡除外（Exclusion Policy）設定。
*   **ラボ試験で設定させられそうな内容**:
    *   「アクセスポートポート群で端末の IP アドレスを自動追跡し、ISE に通知できるように構成せよ。ただし、Windows クライアントで IP 重複エラーが発生しないように考慮すること」という要件。
    *   「Catalyst 9300 (IOS-XE) において SISF デバイス・トラッキングポリシーを作成し、VLAN 10-20 のアクセスポートにのみ適用せよ。Trunk ポートでは学習を行わないこと」という構成要件。
*   **よくある設定ミス**:
    *   `ip device tracking probe delay` または `ip device tracking probe auto-source` の設定漏れによるテスト端末の通信不能。
    *   SISF 環境で `device-tracking binding` が無効化されている、または `device-tracking policy` が VLAN やポートにアタッチされていない。
    *   Trunk ポートで IPDT を全バインドしてしまい、対向スイッチやルータの IP が多重学習されてバインディングテーブルが枯渇する。
*   **show コマンドからの状態判断**:
    *   `show ip device tracking all`: エントリの IP, MAC, Port, State (IP-MAC-ACTIVE/INACTIVE) を確認。
    *   `show device-tracking database`: IOS-XE SISF におけるバインディング DB（State, Type, Interface）を確認。
*   **debug ログの読み取り**:
    *   `debug ip device tracking`: ARP Probe の送信タイミング、レスポンス受信、エントリ生成・破棄のログを出力。

---

## 🛠 設定方法

Legacy Cisco IOS CLI と Cisco IOS-XE SISF Device Tracking CLI の両方の設定例を示します。

### 1. Legacy Cisco IOS 設定例 (Catalyst 2960 / 3750 / 3850 旧バージョン)

```bash
! グローバルで IP Device Tracking を有効化
ip device tracking

! Windows Duplicate IP 0.0.0.0 回避設定（必須）
ip device tracking probe delay 10
ip device tracking probe auto-source fallback 192.0.2.1 255.255.255.0 override

! アクセスインターフェイスでの設定（デフォルトで有効化されるが明示設定も可能）
interface GigabitEthernet1/0/1
 description Access-Port-EndUser
 switchport mode access
 switchport access vlan 10
 ip device tracking maximum 10
!

! Trunk ポートでの IPDT 無効化（他スイッチの IP 誤学習防止）
interface GigabitEthernet1/0/48
 description Trunk-to-Core
 switchport mode trunk
 ip device tracking maximum 0
!
```

### 2. Cisco IOS-XE SISF Device Tracking 設定例 (Catalyst 9000 シリーズ / IOS-XE 16.x 以降)

```bash
! SISF デバイストラッキングポリシーの定義
device-tracking policy DT_ACCESS_POLICY
 tracking-target-switchport
 device-role node
 security-level inspect
 limit address-count 10
 no protocol ndp
!

! SISF 除外ポリシーの定義 (Trunk ポート用)
device-tracking policy DT_TRUNK_EXCLUDE_POLICY
 device-role switch-interface
 tracking-target-switchport
 no protocol arp
 no protocol dhcp4
 no protocol dhcp6
 no protocol ndp
!

! グローバルでのバインディング有効化
device-tracking binding reachable-lifetime 300

! インターフェイスへのポリシー適用
interface GigabitEthernet1/0/1
 description Access-Port-Client
 switchport mode access
 switchport access vlan 10
 device-tracking attach-policy DT_ACCESS_POLICY
!

interface GigabitEthernet1/0/48
 description Trunk-to-Upstream
 switchport mode trunk
 device-tracking attach-policy DT_TRUNK_EXCLUDE_POLICY
!
```

---

## 🔍 検証コマンド

IP Device Tracking および SISF の動作状態を検証するための主要コマンドを下表にまとめます。

| 目的 | コマンド |
| --- | --- |
| Legacy IPDT 全テーブル確認 | <code>show ip device tracking all</code> |
| 特定ポートの IPDT 状態確認 | <code>show ip device tracking interface GigabitEthernet1/0/1</code> |
| IPDT 設定パラメータ確認 | <code>show ip device tracking configuration</code> |
| SISF バインディング DB 確認 | <code>show device-tracking database</code> |
| SISF 詳細バインディング確認 | <code>show device-tracking database interface GigabitEthernet1/0/1</code> |
| SISF 適用ポリシー確認 | <code>show device-tracking policies</code> |
| IPDT デバッグログ実行 | <code>debug ip device tracking {arp \| engine \| event}</code> |
| SISF デバッグログ実行 | <code>debug device-tracking {arp \| dhcp \| event}</code> |

---

## 🚨 トラブルシュート

IPDT / SISF 運用時に発生しやすいトラブル、原因、確認コマンド、および対処方法を下表に整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| Windows クライアント接続直後に IP アドレス重複（0.0.0.0）エラーが発生し NIC が無効化される | スイッチが送信元 IP `0.0.0.0` の ARP Probe を即座に送信し、Windows の DAD 処理と衝突した | <code>show ip device tracking configuration</code> | `ip device tracking probe delay 10` および `ip device tracking probe auto-source` を設定する |
| ISE の Context Directory に端末の IP アドレスが反映されず、CoA が失敗する | スイッチ側で IPDT が無効になっている、または 802.1X 認証後に IPDT テーブルが更新されていない | <code>show ip device tracking all</code><br><code>show authentication sessions interface Gi1/0/1 detail</code> | グローバルおよび対象ポートで IPDT を有効化し、`probe delay` を設定する |
| スイッチの CPU 高負荷またはバインディングテーブルのオーバーフローが発生する | Trunk ポートやルータ接続ポートで IPDT が無効化されておらず、全セグメントの IP を学習している | <code>show ip device tracking all \| count</code><br><code>show device-tracking database</code> | Trunk/Uplink ポートで `ip device tracking maximum 0` または除外 SISF ポリシーを適用する |
| 静的 IP（Static IP）端末が IPDT データベースに学習されない | DHCP を使用しない端末で、初期化時に ARP パケットを発信していない | <code>show ip device tracking interface Gi1/0/1</code> | 端末からゲートウェイ宛てに Ping を送信させて ARP を誘発させるか、Probe 周期を調整する |
| IOS-XE にアップグレード後、`ip device tracking` コマンドが拒否される | IOS-XE 16.x 以降で SISF (Switch Integrated Security Features) 構文への統一が行われた | <code>show device-tracking database</code> | `device-tracking` 新構文（`device-tracking policy`）を用いて設定を書き換える |




## ⚠ 制限事項

*   **サポートされない構成・ポートタイプ**:
    *   Trunk ポートや Dynamic Desirable ポートで無制限に有効化することは非推奨（不必要な L2 トラフィックからの自動学習によるテーブル枯渇）。
    *   Routed Port（L3 インターフェイス）上での直接的な Legacy IPDT の動作制限。
*   **バージョン依存**:
    *   Cisco IOS 15.2(1)E 未満のバージョンでは `ip device tracking probe auto-source` の `override` オプションがサポートされない場合があります。
    *   Cisco IOS-XE 16.3.1 以降、従来型の `ip device tracking` CLI は内部的に SISF `device-tracking` ポリシーへ自動変換されます。
*   **ライセンス依存**:
    *   IPDT / SISF 自体は Network Essentials / IP Base ライセンスで動作可能ですが、ISE 連携や TrustSec (SGT) との統合には Network Advantage / IP Services ライセンスが必要となります。

---

## 🔄 他技術との関連

IP Device Tracking は、L2 アクセス層セキュリティおよびアイデンティティアーキテクチャの中核コンポーネントです。

*   **Cisco ISE (Identity Services Engine)**:
    ISE が 802.1X / MAB 認証された端末の IP アドレスを識別し、Profiling Services や Dynamic CoA（Change of Authorization: 例: 隔離 VLAN 変更、DACR 適用）を発行するために IPDT が提供する `framed-ip-address` が不可欠です。
*   **Dynamic ARP Inspection (DAI) & IP Source Guard (IPSG)**:
    DAI および IPSG はパケットフィルタリングの判定に DHCP Snooping Binding Database を使用しますが、DHCP を使用しない静的 IP 端末環境においては IPDT エントリを補助バインディングソースとして統合利用します。
*   **Cisco TrustSec (CTS) / SGT**:
    IPDT テーブルの IP-MAC-Port バインディング情報に基づき、ISE からダウンロードされた SGT（Security Group Tag）を対象 IP アドレスに紐付ける IP-SGT Mapping 処理を実行します。
*   **IPv6 First Hop Security (FHS)**:
    SISF (Switch Integrated Security Features) 環境において、IPv6 ND Snooping と統合され、IPv6 端末のリンクローカルアドレスおよびグローバルユニキャストアドレスのトラッキングを実施します。

---

## 🧩 比較表

### 1. Legacy IPDT vs IOS-XE SISF Device Tracking

| 比較項目 | Legacy IPDT (IOS 15.x / Early XE) | SISF Device Tracking (IOS-XE 16.x+) |
| --- | --- | --- |
| **アーキテクチャ** | IPDT モジュール独立実装 | SISF (Switch Integrated Security Features) 統合モデル |
| **設定 CLI** | `ip device tracking ...` | `device-tracking policy ...` / `device-tracking binding` |
| **適用レベル** | グローバル指定＋ポート制限 | ポリシー定義によるポート・VLAN へのフレキシブルアタッチ |
| **IPv4 / IPv6 統合** | IPv4 主体（IPv6 は別途 ND Snooping） | IPv4 (ARP/DHCP) と IPv6 (NDP/DHCP6) を単一ポリシーで統合管理 |
| **Trunk ポート除外** | `ip device tracking maximum 0` | `device-tracking policy` 内で `no protocol` アサイン |

### 2. ARP Probe vs DHCP Snooping Binding Table

| 比較項目 | ARP Probe (IPDT) | DHCP Snooping Binding Table |
| --- | --- | --- |
| **学習方式** | パッシブスヌーピング＋アクティブ L2 ARP プロービング | DHCP Ack パケットのパッシブスヌーピングのみ |
| **静的 IP 端末対応** | 対応可能（ARP トラフィック検知＋Probe 活性確認） | 不可（DHCP トランザクションが存在しないため） |
| **主な目的** | リアルタイム IP アドレス追跡・Keepalive | DHCP トラフィック偽装・IP/MAC スプーフィング防止 |

---

## 💡 ベストプラクティス

1.  **Windows 重複 IP 防止用コンフィグの標準化**:
    Legacy IOS スイッチにおいては、全アクセススイッチで以下の 2 行を標準テンプレートとして組み込むことが推奨されます。
    ```bash
    ip device tracking probe delay 10
    ip device tracking probe auto-source fallback 192.0.2.1 255.255.255.0 override
    ```
2.  **Trunk / Uplink ポートの明示的除外**:
    コアスイッチ、ディストリビューションスイッチ、および他スイッチへの Trunk リンクでは必ず IPDT 学習を無効化し、メモリ枯渇と誤学習を防ぎます。
3.  **IOS-XE SISF への早期移行**:
    Catalyst 9000 シリーズ運用時は Legacy CLI ではなく SISF `device-tracking policy` を使用して明確にアクセスポリシーと Trunk ポリシーを分離・運用します。




## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験レベルを意識した実戦的な 10 個の演習シナリオを掲載します。

### シナリオ 1: Legacy IPDT の基本有効化と Windows 重複 IP (0.0.0.0) 発生防止設定

*   **問題**: Catalyst 3850 スイッチ（SW1）の全アクセスポートで IP Device Tracking を有効化し、接続端末の IP アドレスを自動追跡せよ。ただし、Windows 10 端末接続時に「IP アドレス重複 (0.0.0.0)」警告が発生して通信不能となる障害を防止すること。
*   **要件**:
    *   リンクアップ後、スイッチからの ARP Probe 送出を 10 秒間遅延させること。
    *   ARP Probe の送信元 IP アドレスには未利用のダミー IP `192.0.2.254` を使用し、常に上書きすること。
*   **設定例**:
    ```bash
    SW1(config)# ip device tracking
    SW1(config)# ip device tracking probe delay 10
    SW1(config)# ip device tracking probe auto-source fallback 192.0.2.254 255.255.255.0 override
    ```
*   **検証方法**:
    *   `show ip device tracking configuration` を実行し、Probe delay が 10 秒、Auto-source が 192.0.2.254 で override 設定されていることを確認します。

---

### シナリオ 2: Cisco IOS-XE SISF Device Tracking 基本ポリシー構築

*   **问题**: Catalyst 9300 スイッチ（SW2）において、SISF Device Tracking ポリシー `POLICY_USER_ACCESS` を定義し、VLAN 10 のアクセスポート `Gi1/0/1` に適用せよ。
*   **要件**:
    *   ARP および DHCPv4 のみを追跡ターゲットとすること。
    *   単一ポートあたりの追跡アドレス上限数を 5 エントリに制限すること。
*   **設定例**:
    ```bash
    SW2(config)# device-tracking policy POLICY_USER_ACCESS
    SW2(config-device-tracking)# tracking-target-switchport
    SW2(config-device-tracking)# device-role node
    SW2(config-device-tracking)# limit address-count 5
    SW2(config-device-tracking)# no protocol ndp
    SW2(config-device-tracking)# no protocol dhcp6
    SW2(config-device-tracking)# exit

    SW2(config)# interface GigabitEthernet1/0/1
    SW2(config-if)# switchport mode access
    SW2(config-if)# switchport access vlan 10
    SW2(config-if)# device-tracking attach-policy POLICY_USER_ACCESS
    ```
*   **検証方法**:
    *   `show device-tracking policies POLICY_USER_ACCESS` を実行し、プロトコル制限およびアドレス上限が正しく構成され、Gi1/0/1 にアタッチされていることを確認します。

---

### シナリオ 3: Trunk / Uplink ポートにおける IPDT 追跡除外構成

*   **問題**: SW1 の Uplink インターフェイス `Gi1/0/48`（Trunk ポート）において、対向コアスイッチや別セグメントの端末 IP が IPDT データベースに誤学習されるのを完全に遮断せよ。
*   **要件**:
    *   Legacy IOS および IOS-XE SISF の両手法での遮断要件を満たすこと。
*   **設定例**:
    *   **Legacy IOS 構成**:
        ```bash
        SW1(config)# interface GigabitEthernet1/0/48
        SW1(config-if)# switchport mode trunk
        SW1(config-if)# ip device tracking maximum 0
        ```
    *   **IOS-XE SISF 構成**:
        ```bash
        SW2(config)# device-tracking policy POLICY_TRUNK_EXCLUDE
        SW2(config-device-tracking)# device-role switch-interface
        SW2(config-device-tracking)# tracking-target-switchport
        SW2(config-device-tracking)# no protocol arp
        SW2(config-device-tracking)# no protocol dhcp4
        SW2(config-device-tracking)# no protocol dhcp6
        SW2(config-device-tracking)# no protocol ndp
        SW2(config-device-tracking)# exit

        SW2(config)# interface GigabitEthernet1/0/48
        SW2(config-if)# device-tracking attach-policy POLICY_TRUNK_EXCLUDE
        ```
*   **検証方法**:
    *   `show ip device tracking interface Gi1/0/48`（または `show device-tracking database interface Gi1/0/48`）を実行し、該当インターフェイスでバインディングが一切作成されないことを確認します。

---

### シナリオ 4: Cisco ISE エンドポイント追跡連携のための IPDT 調整

*   **問題**: SW1 上で 802.1X / MAB 認証を運用中、端末が IP アドレスを取得した後に ISE に `framed-ip-address` が遅延なく通知されるよう IPDT を最適化せよ。
*   **要件**:
    *   端末接続直後の ARP スヌーピングを確実にキャプチャし、ARP Probe 応答を受信した段階で迅速に ACTIVE 状態へ遷移させること。
*   **設定例**:
    ```bash
    SW1(config)# ip device tracking
    SW1(config)# ip device tracking probe delay 10
    SW1(config)# ip device tracking probe count 3
    SW1(config)# ip device tracking probe interval 30
    SW1(config)# ip device tracking probe auto-source fallback 192.0.2.1 255.255.255.0 override
    ```
*   **検証方法**:
    *   端末認証後、`show authentication sessions interface Gi1/0/1 detail` を実行し、「IPv4 Address: 10.1.10.50」のように IP アドレスが正常に追跡・バインドされていることを確認します。

---

### シナリオ 5: 静的 IP（Static IP）端末環境における ARP Keepalive 設定

*   **問題**: 工場フロアのプリンタや監視カメラ（VLAN 20）など、DHCP を使用せず静的 IP が割り当てられている端末群ポートにおいて、IPDT エントリの維持とダウン検知を自動化せよ。
*   **要件**:
    *   静的 IP 端末の活性状態を維持するため、定期的に ARP Probe を送出すること。
*   **設定例**:
    ```bash
    SW1(config)# ip device tracking probe interval 60
    SW1(config)# interface range GigabitEthernet1/0/10 - 20
    SW1(config-if-range)# switchport mode access
    SW1(config-if-range)# switchport access vlan 20
    SW1(config-if-range)# ip device tracking maximum 5
    ```
*   **検証方法**:
    *   `show ip device tracking interface Gi1/0/10` を確認し、静的 IP 端末のエントリが `IP-MAC-ACTIVE` 状態を維持していることを検証します。

---

### シナリオ 6: 802.1X / MAB 認証環境における IPDT 統合

*   **問題**: アクセスポート `Gi1/0/2` において 802.1X 認証および MAB を有効化し、認証成功後に IPDT によって端末の IP アドレスを即座に特定・固定せよ。
*   **要件**:
    *   MAB 端末が DHCP オファーを受領した直後に IP アドレスを確定させ、セッションテーブルにバインドすること。
*   **設定例**:
    ```bash
    SW1(config)# ip device tracking
    SW1(config)# ip device tracking probe delay 10
    SW1(config)# ip device tracking probe auto-source fallback 192.0.2.1 255.255.255.0 override

    SW1(config)# interface GigabitEthernet1/0/2
    SW1(config-if)# switchport mode access
    SW1(config-if)# authentication port-control auto
    SW1(config-if)# mab
    SW1(config-if)# dot1x pae authenticator
    ```
*   **検証方法**:
    *   端末接続後、`show ip device tracking interface Gi1/0/2` と `show authentication sessions interface Gi1/0/2` を照合し、MAC アドレスと取得 IP が完全一致していることを確認します。

---

### シナリオ 7: Dynamic ARP Inspection (DAI) と IPDT バインディングの連携

*   **問題**: DHCP を使用しない静的 IP サーバが接続されたポート `Gi1/0/5` において DAI を無効化することなく、IPDT エントリを用いて ARP スプーフィング攻撃を防御せよ。
*   **要件**:
    *   静的 IP アドレスと MAC アドレスのバインディングを ARP ACL で定義し、DAI の検証ソースとして適用すること。
*   **設定例**:
    ```bash
    ! 静的 ARP ACL の定義
    SW1(config)# arp access-list STATIC_SERVER_ARP
    SW1(config-arp-acl)# permit ip host 10.1.20.100 mac host 0011.2233.4455
    SW1(config-arp-acl)# exit

    ! DAI への ARP ACL 適用
    SW1(config)# ip arp inspection filter STATIC_SERVER_ARP vlan 20

    ! IPDT 活性保持の有効化
    SW1(config)# ip device tracking probe delay 10
    ```
*   **検証方法**:
    *   `show ip arp inspection vlan 20` を実行し、静的 ARP ACL が DAI フィルタとして登録されていることを確認します。

---

### シナリオ 8: IPv6 端末のトラッキング（SISF NDP Snooping）構築

*   **問題**: Catalyst 9300（SW2）において、VLAN 30 に接続された IPv6 エンドポイントの IPv6 リンクローカルおよびグローバルアドレスを SISF Device Tracking で自動追跡せよ。
*   **要件**:
    *   IPv6 Neighbor Discovery (ND) および DHCPv6 を追跡対象に含めること。
*   **設定例**:
    ```bash
    SW2(config)# device-tracking policy POLICY_IPV6_ACCESS
    SW2(config-device-tracking)# tracking-target-switchport
    SW2(config-device-tracking)# device-role node
    SW2(config-device-tracking)# security-level inspect
    SW2(config-device-tracking)# protocol ndp
    SW2(config-device-tracking)# protocol dhcp6
    SW2(config-device-tracking)# exit

    SW2(config)# interface GigabitEthernet1/0/3
    SW2(config-if)# switchport mode access
    SW2(config-if)# switchport access vlan 30
    SW2(config-if)# device-tracking attach-policy POLICY_IPV6_ACCESS
    ```
*   **検証方法**:
    *   `show device-tracking database interface Gi1/0/3` を実行し、端末の IPv6 アドレス（FE80:: および 2001:DB8:: 体系）がバインディングテーブルに表示されることを確認します。

---

### シナリオ 9: 単一ポートにおける IPDT アドレス学習上限数の制御

*   **問題**: ハブや小型スイッチの無断接続（MAC/IP スプーフィング）を防止するため、アクセスポート `Gi1/0/4` において IPDT が追跡する最大 IP アドレス数を 2 に制限せよ。
*   **要件**:
    *   3 つ目以降の異なった IP アドレスが検出された場合、バインディングの追加を拒否すること。
*   **設定例**:
    *   **Legacy IOS 構成**:
        ```bash
        SW1(config)# interface GigabitEthernet1/0/4
        SW1(config-if)# ip device tracking maximum 2
        ```
    *   **IOS-XE SISF 構成**:
        ```bash
        SW2(config)# device-tracking policy POLICY_LIMIT_2
        SW2(config-device-tracking)# tracking-target-switchport
        SW2(config-device-tracking)# limit address-count 2
        SW2(config-device-tracking)# exit

        SW2(config)# interface GigabitEthernet1/0/4
        SW2(config-if)# device-tracking attach-policy POLICY_LIMIT_2
        ```
*   **検証方法**:
    *   `Gi1/0/4` に 3 台目の IP アドレスを持つ端末を接続し、`show ip device tracking interface Gi1/0/4`（または `show device-tracking database interface Gi1/0/4`）でエントリ数が 2 で頭打ちになっていることを確認します。

---

### シナリオ 10: IPDT バインディングテーブルの手動クリアと動的再学習検証

*   **問題**: 保守作業に伴い、SW1 のポート `Gi1/0/1` に残存する古い IPDT バインディングエントリを安全に全消去し、新規端末の即時再学習を実施せよ。
*   **要件**:
    *   スイッチを再起動することなく対象ポートのエントリのみをクリアすること。
*   **設定例**:
    *   **Legacy IOS 実行コマンド**:
        ```bash
        SW1# clear ip device tracking interface GigabitEthernet1/0/1
        ```
    *   **IOS-XE SISF 実行コマンド**:
        ```bash
        SW2# clear device-tracking database interface GigabitEthernet1/0/1
        ```
*   **検証方法**:
    *   コマンド実行直後に `show ip device tracking interface Gi1/0/1`（または `show device-tracking database interface Gi1/0/1`）を実行してテーブルが一旦空になり、端末から通信（Ping 等）が発生すると即座に再学習されることを検証します。




## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験および筆記試験（SCOR 350-701）を想定した実戦的クイズです。

### 問題 1 (コンフィグ読解)
**質問**: 以下のコンフィグが適用された Catalyst 3850 スイッチ（IOS 15.2(2)E）において、`ip device tracking probe delay 10` が設定されている主な技術的理由は何か？
```bash
ip device tracking
ip device tracking probe delay 10
ip device tracking probe auto-source fallback 192.0.2.1 255.255.255.0 override
```
*   A) スイッチの CPU 負荷を軽減するため
*   B) Windows クライアントがリンクアップ直後に行う Duplicate Address Detection (DAD) 処理とスイッチの ARP Probe が衝突し、0.0.0.0 重複 IP エラーが発生するのを防止するため
*   C) DHCP サーバからの IP アドレス割り当て応答を 10 秒間待機するため
*   D) DAI (Dynamic ARP Inspection) のデータベース同期を待つため

**正解**: **B**
**解説**: Windows OS 端末はリンクアップ直後に RFC 5227 に基づく DAD (ARP Announce/Probe) を送出します。スイッチが即座に送信元 `0.0.0.0` の ARP Probe を送信すると、Windows 側で IP アドレス衝突と誤検知して NIC が切断されるため、10 秒間の遅延 (`probe delay 10`) を設けて衝突を防ぎます。

---

### 問題 2 (トラブルシューティング)
**質問**: 新しく導入した Catalyst 9300 (IOS-XE 16.12) スイッチにおいて、Cisco ISE によるプロファイリングが動作せず、エンドポイントの IP アドレスが ISE 側で取得できない。スイッチ側で `show device-tracking database` を実行したところ、テーブルが完全に空であった。原因として最も可能性が高いものはどれか？
*   A) `ip device tracking` コマンドがグローバルで入力されていない
*   B) `device-tracking policy` が作成されているが、対象のアクセスインターフェイスに `device-tracking attach-policy` で適用されていない、または `device-tracking binding` が無効化されている
*   C) 802.1X 認証が失敗している
*   D) DHCP Snooping が無効化されているため IPDT は動作しない

**正解**: **B**
**解説**: IOS-XE 16.x 以降の SISF アーキテクチャでは、単にポリシーを定義するだけでなく、`device-tracking attach-policy <NAME>` を使用して明示的にインターフェイス（または VLAN）にアタッチする必要があります。ポリシーが適用されていない場合、バインディング DB は作成されません。

---

### 問題 3 (Design)
**質問**: 企業ネットワークのコアスイッチとアクセススイッチを結ぶ Trunk ポートにおいて、IP Device Tracking / SISF の設計として適切なアプローチはどれか？
*   A) 全ての端末の IP を中央で一括管理するため、Trunk ポートでも追跡ポリシーを全プロトコル有効で適用する
*   B) 大量の異機種 IP バインディングの誤学習によるバインディングテーブル枯渇および CPU 高負荷を防ぐため、Trunk ポートでは IPDT / SISF 追跡を明示的に無効化（除外）する
*   C) Trunk ポートでは `ip device tracking probe delay 0` を設定する
*   D) Trunk ポートの Native VLAN でのみ IPDT を有効化する

**正解**: **B**
**解説**: Trunk ポートや Uplink ポートで IPDT を有効のままにすると、対向スイッチや他の全アクセスポートに接続された端末の IP/MAC を多重学習してしまい、データベースのオーバーフローや CPU 高負荷、さらには誤った IP-MAC バインディングを引き起こします。そのため、Trunk ポートでは明示的に除外設定を行います。

---

### 問題 4 (実装)
**質問**: Legacy Cisco IOS において、スイッチが送出する ARP Probe の送信元 IP アドレスを `0.0.0.0` からダミーの非ゼロ IP アドレスへ強制的に変更するための正しいコマンドはどれか？
*   A) `ip device tracking probe source 192.0.2.1`
*   B) `ip device tracking probe auto-source fallback 192.0.2.1 255.255.255.0 override`
*   C) `ip device tracking source-ip 192.0.2.1`
*   D) `ip device tracking probe static 192.0.2.1`

**正解**: **B**
**解説**: `ip device tracking probe auto-source fallback <IP> <NETMASK> override` コマンドを使用することで、SVI の有無にかかわらず、送出される ARP Probe の送信元 IP を指定した IP アドレスで上書きできます。

---

### 問題 5 (トラブルシューティング)
**質問**: ネットワーク管理者がアクセスポートで `ip device tracking maximum 1` を設定したところ、PC の NIC を交換した後に新しい PC が一切通信できなくなった。原因と解決策として正しい組合せはどれか？
*   A) 原因: 古い MAC-IP バインディングが IPDT テーブルに残存しているため。解決策: 古いエントリがタイムアウトするのを待つか、`clear ip device tracking interface` を実行する
*   B) 原因: スイッチのポートが err-disable になったため。解決策: `shutdown` / `no shutdown` を実行する
*   C) 原因: PC の IP アドレスが変更されたため。解決策: PC で `ipconfig /renew` を実行する
*   D) 原因: 802.1X 認証がブロックされたため。解決策: RADIUS サーバを再起動する

**正解**: **A**
**解説**: `maximum 1` の制限により、古いエントリがテーブルに保持されている間は新しい端末のバインディング追加が拒否されます。手動で `clear ip device tracking interface` を実行するか、エイジングタイマーを調整することで解決できます。

---

## 🔗 参考リソース

Cisco 公式の最新ドキュメントおよび技術資料一覧です（Workbook / Cert Guide を除く）。

*   Cisco Technical Note: Troubleshooting IP Device Tracking (IPDT)
*   Cisco Technical Note: Duplicate IP Address 0.0.0.0 Error Message Solutions
*   Cisco IOS-XE 17.x Configuration Guide: Switch Integrated Security Features (SISF) / Device Tracking
*   Cisco Live Technical Presentation: Enterprise Campus System Hardening and Device Security
*   Cisco Validated Design (CVD): Software-Defined Access (SD-Access) Deployment Guide
*   Cisco Technical Support Document: Cisco ISE and IP Device Tracking Integration Guide

---

## 📝 補足（Notes）

*   **学習メモ**:
    *   CCIE ラボ試験の初期コンフィグで `ip device tracking` が既に投入されている場合でも、`probe delay 10` や `probe auto-source` が欠落しているケースが多々あります。試験問題文に「クライアント接続時の IP アドレス重複エラーを防ぐこと」等の記述がある場合は、必ず追加設定を投入してください。
*   **注意点**:
    *   IOS-XE 16.x 以降の環境で Legacy コマンド `ip device tracking` を入力すると、内部的に SISF ポリシーへ自動変換されてコンフィグに反映されます。`show running-config` で表示が変わっても焦らず、`show device-tracking database` で正しく学習されているか確認してください。

