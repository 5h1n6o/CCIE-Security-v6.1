# 1.11 Network connectivity through Cisco FTD

Cisco Secure Firewall (FTD) におけるネットワーク接続は、従来の ASA の堅牢な L3/L4 転送能力（LINA エンジン）と、Snort による高度な脅威防御を統合したアーキテクチャに基づいています。CCIE Security ラボ試験では、Firepower Management Center (FMC) を介したインターフェイス、ゾーン、およびルーティングの正確な実装に加え、Snort エンジンがパケット転送にどのように介入するかを深く理解していることが求められます。

---

## 📘 概要

*   **機能概要**: 物理インターフェイスを論理的な **Security Zones** にグループ化し、トラフィックの転送（Routed Mode）または透過（Transparent Mode）を行います。
*   **利用目的**: Snort インスペクションを介したセキュアな通信経路の確立、およびインフラ全体の可視化。
*   **どのような場面で利用するか**: エンタープライズ境界のエッジ、データセンター内のセグメンテーション、または既存ネットワークを変更せずに導入するステルス構成（Transparent）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **転送モード** | **Routed Mode** (L3 ホップ) / **Transparent Mode** (L2 ブリッジ)。 |
| **論理構成** | **Security Zones** を使用。ASA のような 0-100 の数値による自動許可はない。 |
| **アーキテクチャ** | **LINA (ASA)** エンジンと **Snort** エンジンのハイブリッド。 |
| **インターフェイス種類** | Physical, Sub-interface, EtherChannel, Redundant, Tunnel (VTI)。 |
| **特殊モード** | Inline, Passive, Tap, ERSPAN (分析用)。 |
| **管理** | **Firepower Management Center (FMC)** または FDM による一元管理。 |

---

## 🏗 動作原理

FTD の接続性は、パケットが LINA エンジンに入力され、特定の「判別」を経て Snort エンジンへオフロードされるプロセスに基づきます。

```text
[ Ingress Interface ]
        ↓
[ LINA: L2/L3 Check ] <--- ARP, Routing, NAT Lookup
        ↓
[ Prefilter Policy ] <--- Early Fastpath or Block
        ↓
[ Snort: Inspection ] <--- AVC, IPS, File, Malware (Snort 2 or 3)
        ↓
[ LINA: Egress ] <--- Final forwarding & NAT
        ↓
[ Egress Interface ]
```

---

## ⚙ 動作シーケンス

1.  **インターフェイス受信**: 物理/仮想インターフェイスが L2 フレームを受信。
2.  **LINA 処理 (Phase 1)**: 送信元/宛先 IP、ポートを確認。既存コネクションがあれば Snort へ送るか判断。
3.  **セキュリティゾーン評価**: 受信インターフェイスが属するゾーンを確認。
4.  **Snort インスペクション**: アクセスコントロールポリシー (ACP) に基づき、Snort がアプリケーション層まで検査。
5.  **LINA 処理 (Phase 2)**: 検査をパスしたパケットに対し、NAT を適用しルーティングテーブル (RIB) に基づき出口 IF を決定。
6.  **送出**: レイヤ 2 ヘッダーを書き換え、パケットを隣接デバイスへ転送。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Deployment (デプロイ)**: FMC で設定を変更しても、**Deploy** ボタンを押して FTD デバイスにプッシュしない限り反映されません。これはラボでの「設定したはずなのに動かない」の最大要因です。
*   **Security Zones**: インターフェイスをゾーンに割り当てるのを忘れると、ACP ルールでその IF を指定できません。
*   **FMC vs CLI**: 基本設定は FMC で行いますが、トラブルシュートには FTD CLI (`system support diagnostic-cli`) を使用して、従来の ASA コマンド (`show conn`, `show asp drop`) を実行するスキルが必須です。
*   **BVI (Bridge Virtual Interface)**: Transparent Mode 設定時、管理用 IP と転送トラフィックのブリッジングに BVI が使用されます。
*   **MTU の不一致**: 複雑な VPN やオーバーレイ（VXLAN 等）を構成する際、FMC のインターフェイス設定で MTU を調整する能力が問われます。

---

## 🛠 設定方法

### 1. インターフェイスの基本設定 (FMC)
1.  **Devices > Device Management** で対象 FTD を編集。
2.  **Interfaces** タブで物理インターフェイス（例: eth0/0）を編集。
3.  **Mode**: `Routed` または `Transparent` を選択。
4.  **IPv4**: `Static` を選び IP/マスクを入力。
5.  **Security Zones**: `New` でゾーン名（例: Inside_Zone）を作成し割り当て。

### 2. ルーティングの設定
1.  **Devices > Device Management > Routing** へ移動。
2.  **Static Route** を選択し、ゲートウェイを指定。
3.  OSPF や BGP を使用する場合は、各プロトコルのタブでプロセスを定義。

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI / Diagnostic) |
| :--- | :--- |
| **インターフェイス IP 状態** | <code>show interface ip brief</code> |
| **ゾーンの割り当て確認** | <code>show zone</code> |
| **コネクションテーブル確認** | <code>system support diagnostic-cli</code> -> <code>show conn</code> |
| **パケットドロップの統計** | <code>show asp drop</code> |
| **リアルタイムデバッグ** | <code>system support firewall-engine-debug</code> |
| **シミュレーション** | <code>packet-tracer input [IF] tcp [src] [port] [dst] [port]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 設定が反映されない | デプロイ未完了 | FMC の **Task Status** でデプロイが成功したか確認。 |
| 隣接デバイスと Ping 不通 | ICMP インスペクション | ACP ルールで ICMP が許可されているか、または <code>inspect icmp</code> 設定を確認。 |
| ゾーン間で通信不可 | ゾーン指定のミス | ACP ルールで正しい **Source/Destination Zone** が選択されているか確認。 |
| EIGRP が動かない | FlexConfig 未適用 | FTD では EIGRP は **FlexConfig** でのみ設定可能です。 |

---

## ⚠ 制限事項

*   **ASA からの移行**: 一部の ASA 機能（マルチキャストルーティングの一部、特定の VPN 構成）は FTD での実装方法が異なるか、サポートが限定的です。
*   **ハードウェアバイパス**: 特定の 4100/9300 ネットワークモジュールを除き、ハードウェアレベルのバイパスはサポートされません。
*   **ライセンス**: Base ライセンスで接続性は確保できますが、Snort 検査を行うには **Threat** サブスクリプションが必要です。

---

## 🔄 他技術との関連

*   **Routing (1.10)**: FTD は LINA 上で OSPF, BGP, RIP, EIGRP を実行します。
*   **High Availability (1.8)**: HA 構成では、インターフェイス設定と接続状態が 2 台間で同期されます。
*   **Access Control (1.9)**: 接続性は ACP ルールによって最終的に許可/拒否されます。
*   **NAT**: 接続フロー中に LINA エンジンがアドレス変換を実施します。

---

## 🧩 比較表

### FTD Security Zone vs ASA Security Level

| 特徴 | FTD Security Zone | ASA Security Level |
| :--- | :--- | :--- |
| **概念** | 名前ベースのグループ化 | 0-100 の数値ベース |
| **デフォルト許可** | なし（すべて明示的ルールが必要） | 高レベルから低レベルは自動許可 |
| **柔軟性** | 複数の IF を 1 つのゾーンに纏められる | 原則 1 IF = 1 Level (共有も可) |
| **試験の勘所** | ゾーンを ACP の条件に使用 | Level による暗黙の許可を考慮 |

---

## 💡 ベストプラクティス

1.  **一貫したゾーン命名**: `Inside_Zone`, `Outside_Zone` など、役割が明確な命名を行います。
2.  **デプロイ前の検証**: 常に **Packet Tracer** を使用して、論理的なパケットフローが正しいか確認してからデプロイします。
3.  **診断 CLI の活用**: FMC の GUI だけでは見えない L3/L4 の詳細な状態は、常に Diagnostic CLI で確認する癖をつけます。
4.  **MTU の最適化**: カプセル化（IPsec 等）を使用する場合、MTU を 1400 程度に下げ、TCP MSS を調整することを検討します。

---

## 📝 ラボ学習・設定サンプル例

### 1. FMC による基本的な Routed IF 実装
*   **要件**: eth0/0 を外部、eth0/1 を内部として、それぞれ 203.0.113.1、10.1.1.1 を設定せよ。
*   **設定**: FMC > Device Management > Interfaces > 設定。

### 2. VLAN サブインターフェイスの構成
*   **要件**: eth0/1 上で VLAN 10 と 20 のトラフィックを個別のゾーンとして扱え。
*   **設定**: eth0/1 に `Add Sub-interface` し、ID 10/20 を指定。それぞれ別ゾーンへ。

### 3. EtherChannel (LACP) の実装
*   **要件**: eth0/2 と eth0/3 を冗長化し、スループットを向上させよ。
*   **設定**: `Add Port Channel` > インターフェイス選択 > Mode: `Active (LACP)`。

### 4. Transparent Mode での BVI 設定
*   **要件**: FTD を L2 モードで構成し、管理用に 10.1.1.10 を割り当てよ。
*   **設定**: Device 設定でモードを `Transparent` へ変更 > `Add Bridge Group` > BVI IP 設定。

### 5. スタティックデフォルトルート
*   **要件**: すべての不明なトラフィックを 203.0.113.254 へ送れ。
*   **設定**: Routing > Static Route > `0.0.0.0/0` via `Outside_Zone` gateway `203.0.113.254`.

### 6. OSPFv2 の基本構成
*   **要件**: 内部ゾーンで Area 0 を動作させよ。
*   **設定**: Routing > OSPF > Process 1 > Area 0 > インターフェイス選択。

### 7. IPv6 アドレスの割り当て
*   **要件**: Inside インターフェイスに `2001:db8:1::1/64` を設定せよ。

### 8. Diagnostic CLI による ARP 確認
*   **課題**: 対向ルータの MAC が学習できない。
*   **コマンド**: `system support diagnostic-cli` -> `show arp`.

### 9. Management ネットワークの分離
*   **要件**: 管理トラフィックを専用の `Diagnostic` インターフェイス経由に限定せよ。

### 10. FlexConfig による EIGRP 設定
*   **要件**: EIGRP AS 100 を設定せよ（GUI 非サポート）。
*   **設定**: FMC > Objects > FlexConfig オブジェクト作成 > `router eigrp 100` ... > デプロイ。

---

## ❓ 想定試験問題

1.  **Design**: FTD において、3 つの物理インターフェイスを同じ「Trusted」ゾーンに所属させた場合のトラフィック制御の挙動を述べよ。
    *   **解答**: 同一ゾーン内のインターフェイス間通信であっても、デフォルトでは ACP による明示的な許可ルールが必要です。
2.  **トラブルシュート**: FMC からスタティックルートを設定し、デプロイも成功したが、Diagnostic CLI で `show route` を見るとルートが表示されない。何を確認すべきか？
    *   **解答**: ゲートウェイ IP への到達性が物理/L2 レベルであるか、およびインターフェイスが `No Shutdown` かを確認。
3.  **コンフィグ読解**: `packet-tracer` の出力で `Snort: Block` と表示された。どのポリシーを確認すべきか？
    *   **解答**: Access Control Policy (ACP) または関連する Intrusion Policy (IPS)。
4.  **実装**: Transparent Mode の FTD で、特定の VLAN トラフィックのみをブリッジさせるために必要なインターフェイス構成要素は？
    *   **解答**: Bridge Group Member インターフェイスと、対応する Bridge Virtual Interface (BVI)。
5.  **トラブルシュート**: 1500 バイトを超える ping パケットが FTD を通過できない。MTU 以外で確認すべき FTD 固有の設定は？
    *   **解答**: Prefilter Policy におけるフラグメンテーション処理設定、または `tcp-mss` 調整。

---

## 🔗 参考リソース

*   **Cisco Secure Firewall Management Center Administration Guide, 7.1**
    *   [Managing Interfaces](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/device_management_interfaces.html)
*   **Cisco Live (BRKSEC-2021)**
    *   [Firepower Threat Defense - Packet Flow and Troubleshooting](https://www.ciscolive.com/)
*   **CVD / Design Guides**
    *   [Cisco Firepower NGIPS Deployment Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Aug2014/CVD-FirepowerNGIPSDeploymentGuide-DEC14.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: FTD は「ASA + Snort」という 2 つのエンジンが共存しているため、トラブル時には「どちらのエンジンで問題が起きているか」を切り分けるのが合格への近道です。
*   **注意点**: ラボ試験での FTD 設定は、GUI のレスポンス待ち時間が発生します。1 つの設定ごとにデプロイするのではなく、関連する項目（IF、Zone、Route）をすべて入力してから一気に **Deploy** することで、時間を大幅に節約できます。
