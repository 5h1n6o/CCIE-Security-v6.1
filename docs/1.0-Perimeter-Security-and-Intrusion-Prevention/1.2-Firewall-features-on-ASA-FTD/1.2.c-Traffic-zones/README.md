---
layout: default
title: 1.2.c-Traffic-zones
nav_order: 1
parent: 1.2-Firewall-features-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.2.c Traffic zones

Cisco ASAおよびCisco Firepower Threat Defense (FTD) における**トラフィックゾーン（Traffic Zones）**は、複数のインターフェイスを論理的に1つのグループとして扱う機能です。この機能は、特に動的ルーティング環境における**非対称ルーティング（Asymmetric Routing）**の許容や、複数パスを利用した**負荷分散（ECMP: Equal-Cost Multi-Path）**をファイアウォールで実現するために極めて重要です,。

---

# 📘 概要

*   **機能概要**: 複数の物理または論理インターフェイスを「ゾーン」という1つのグループにまとめます。ファイアウォールは、同一ゾーンに属するインターフェイス間での通信の「出入り」を柔軟に処理できるようになります。
*   **利用目的**:
    *   **非対称ルーティングのサポート**: 行きのパケットと戻りのパケットが異なるインターフェイスを通過する場合でも、同一ゾーン内であればステートフルインスペクションによるドロップを防ぎます。
    *   **ECMPの実現**: 同一のコストを持つ複数の経路に対してトラフィックを分散させ、スループットを向上させます。
    *   **管理の簡素化**: FTDにおいては、複数のインターフェイスに対して1つのアクセスコントロールルールを一括適用するために利用されます。
*   **場面**: 冗長化されたインターネット接続（マルチホーミング）や、複雑な内部メッシュネットワークにおいて、通信経路が固定されない環境で展開されます。

---

# 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | 複数インターフェイス（最大8つ）を1つの論理単位に統合する。 |
| **主な用途** | 非対称ルーティングの解決、ECMPによる負荷分散、ポリシー適用の一元化。 |
| **メリット** | 接続の持続性を維持しつつ、複数のネットワークパスを有効活用できる。 |
| **デメリット** | 設定ミスによりセキュリティ境界が曖昧になるリスクがある（ゾーン内通信の制御）。 |
| **対応機種** | ASA 5500-X シリーズ（9.0以降）、FTD（全モデル）。 |
| **制限事項** | 同一ゾーン内のインターフェイスは、原則として同じセキュリティレベル（ASA）である必要がある。 |
| **設計上の注意点** | ゾーンを跨ぐ通信とゾーン内の通信で、セキュリティポリシーがどのように適用されるかを理解すること。 |

---

# 🏗 動作原理

トラフィックゾーンがない場合、ファイアウォールはパケットが入ってきたインターフェイスと戻ってくるインターフェイスが一致することを期待します（Symmetric Return）。ゾーンを構成すると、この制約が緩和されます。

```text
[ 外部ネットワーク ]
      /      \
(ISP-A)      (ISP-B)
   |            |
[ IF: G0/0 ]  [ IF: G0/1 ]
   \            /
    [ Traffic Zone: OUTSIDE ]
            |
    [ ASA / FTD Security Engine ]
            |
    [ Traffic Zone: INSIDE ]
   /            \
[ IF: G0/2 ]  [ IF: G0/3 ]
```

**非対称フローの例:**
1. クライアントが `ISP-A` 経由でリクエストを送信。
2. サーバーからのレスポンスがルーティングの都合で `ISP-B` 経由で到着。
3. ASA/FTDは `ISP-A` と `ISP-B` が同一ゾーンであれば、これを正当な戻りパケットとして受理します。

---

# ⚙ 動作シーケンス

1.  **パケット着信**: 物理インターフェイスでパケットを受信。
2.  **ゾーン識別**: 受信インターフェイスがどの `Traffic Zone` に属しているかを確認。
3.  **セッションルックアップ**: 既存のコネクションテーブルを検索。
    *   **ゾーン一致**: 既存セッションの「行き」のインターフェイスと異なっていても、同一ゾーン内であればヒットとみなされる。
4.  **ポリシー適用**: ゾーンベースのACL（ASA）またはSecurity ZoneベースのACP（FTD）を評価。
5.  **Egress決定**: ルーティングテーブル（RIB）に基づき出力インターフェイスを決定。ECMPが有効な場合、ゾーン内のいずれかのインターフェイスが選ばれる。
6.  **ステート更新**: 通信の状態を更新し、戻りのパケットを待機。

---

# 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **非対称ルーティングのトラブルシュート**: 試験で「経路が冗長化されているが、時々通信が切れる」というシナリオが出た場合、トラフィックゾーンによる解決が正解である可能性が高いです。
*   **ECMPの構成**: `route` コマンドで同一宛先に複数のネクストホップを設定し、それらのインターフェイスを同一ゾーンに入れる構成が問われます。

### ラボ試験で設定させられそうな内容
*   **ASAでのZone作成**: `zone` コマンドを作成し、`nameif` 設定下で `zone` に割り当てる手順。
*   **FTD Security Zoneの活用**: 複数のサブインターフェイスを1つのゾーン（例: `INSIDE_ZONE`）にまとめ、アクセスコントロールポリシー（ACP）で送信元として指定する実装。
*   **ゾーン内トラフィックの制御**: 同一ゾーンに属するインターフェイス間での通信を許可するか拒否するか（`same-security-traffic` 関連）の設定。

### 試験で狙われやすい制限事項
*   **インターフェイスの重複不可**: 1つのインターフェイスは1つのゾーンにしか所属できません。
*   **VPNとの組み合わせ**: トラフィックゾーンに属するインターフェイスでのVPN終端には一部制約があるため、ドキュメントの確認が必要です。

### showコマンドによる状態判断
*   `show zone`: ゾーンの定義と所属インターフェイス、統計情報を確認。
*   `show nameif`: 各インターフェイスの名前、セキュリティレベル、および所属ゾーンを一覧表示。

---

# 🛠 設定方法

### ASA (CLI) - トラフィックゾーンの設定
ASAでは、まずゾーンを定義し、次にインターフェイス設定モードでそのゾーンに割り当てます。
```bash
# 1. ゾーンの定義
zone OUTSIDE_ZONE

# 2. インターフェイスへの割り当て
interface GigabitEthernet0/0
 nameif ISP-A
 security-level 0
 zone OUTSIDE_ZONE
 ip address 203.0.113.1 255.255.255.0
!
interface GigabitEthernet0/1
 nameif ISP-B
 security-level 0
 zone OUTSIDE_ZONE
 ip address 203.0.113.2 255.255.255.0

# 3. ECMPスタティックルートの設定
route ISP-A 0.0.0.0 0.0.0.0 203.0.113.254
route ISP-B 0.0.0.0 0.0.0.0 203.0.113.254
```

### FTD (FMC管理) - Security Zoneの設定
FTDでは、インターフェイスをゾーンに割り当てることで、ポリシーの抽象化を行います。
1.  **Objects > Object Management > Address Pools / Security Zones**:
    *   **Add Security Zone** をクリック。
    *   ゾーン名（例: `Inside_Zone`）を入力し、**Interface Type**（Routed/Transparent）を選択。
2.  **Devices > Device Management > Interfaces**:
    *   物理インターフェイス（またはサブインターフェイス）を編集。
    *   **Security Zone** ドロップダウンから作成したゾーンを選択。
3.  **Policies > Access Control**:
    *   ルール作成時、**Source Zones / Destination Zones** に作成したゾーンを指定。

---

# 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **ゾーン設定の確認** | <code>show zone</code> |
| **インターフェイスとゾーンの紐付け** | <code>show nameif</code> |
| **ゾーンごとの接続数確認** | <code>show conn zone OUTSIDE_ZONE</code> |
| **パケットフローのシミュレーション** | <code>packet-tracer input ISP-A tcp 10.1.1.1 1234 8.8.8.8 443</code> |
| **ECMPルートの確認** | <code>show route</code> (複数のネクストホップが表示されるか) |

---

# 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 戻りパケットがドロップされる | 非対称パスのインターフェイスが別ゾーン | <code>show zone</code> で所属を確認し、同一ゾーンに追加する。 |
| ECMPが動作しない | メトリックまたはアドミニストレーティブディスタンスの不一致 | <code>show route</code> でコストが同一であることを確認。 |
| ゾーン作成コマンドが拒否される | ASAのバージョンが古い、またはライセンス制限 | <code>show version</code> で9.0以降であることを確認。 |
| 特定インターフェイスの統計が増えない | ルーティングによる偏り | <code>show zone</code> のパケットカウンタを確認し、RIBの等コスト設定を見直す。 |
| ポリシーが適用されない (FTD) | ゾーンのタイプ(Routed/Transparent)不整合 | FMCでゾーンのタイプとインターフェイスのモードが一致しているか確認。 |

---

# ⚠ 制限事項

*   **最大インターフェイス数**: 1つのゾーンに含めることができるインターフェイス数はプラットフォームにより制限がありますが、一般的に最大8つです。
*   **透過モードの制約**: 透過モード（Transparent）では、ブリッジグループ（BVI）との兼ね合いでゾーンの動作が異なる場合があります。
*   **マルチキャスト**: トラフィックゾーンを介したマルチキャストルーティングには特定の制約があるため、PIMの設定時には注意が必要です。
*   **同一セキュリティレベル**: ASAではゾーンに含めるインターフェイスのセキュリティレベルを一致させることが推奨されます（`same-security-traffic` の影響を避けるため）。

---

# 🔄 他技術との関連

*   **Routing (OSPF/BGP)**: 動的ルーティングで複数のパスが学習される環境で、トラフィックゾーンはステートフル性の維持に不可欠です。
*   **Access Control (ACP)**: FTDにおいて、ゾーンはアクセスコントロールルールの「コンテナ」として機能し、物理トポロジの変化からポリシーを保護します。
*   **NAT**: ゾーンを跨ぐNAT（Inter-zone NAT）と、同一ゾーン内でのNAT処理の違いを理解する必要があります。
*   **High Availability**: ゾーンの設定はフェイルオーバーユニット間で同期されます。

---

# 🧩 比較表

### ASA Traffic Zones vs FTD Security Zones

| 比較項目 | ASA Traffic Zone | FTD Security Zone |
| :--- | :--- | :--- |
| **主な目的** | 非対称ルーティング / ECMP | ポリシー適用の一元化 |
| **設定場所** | CLI / ASDM | FMC (Firepower Management Center) |
| **非対称フロー** | ゾーン内で自動許容 | 同様にサポート（LINAエンジン） |
| **最大数** | 最大8インターフェイス/ゾーン | 制限は緩やか（論理的な制約） |
| **階層構造** | なし | なし（フラットなグループ） |

---

# 💡 ベストプラクティス

*   **論理的な境界に合わせる**: 物理的な配線ではなく、「信頼済み（Inside）」「非信頼（Outside）」「DMZ」という論理的な役割に基づいてゾーンを構成します。
*   **一貫したセキュリティレベル**: ゾーン内のインターフェイスには同じセキュリティレベルを割り当て、意図しないドロップや許可を防ぎます。
*   **名前解決の活用**: ゾーン名には `ZONE_OUTSIDE_INTERNET` のように、その役割が明確にわかる名前を付けます。
*   **FMCでの事前定義**: FTD展開時は、物理設定の前にまず `Security Zone` オブジェクトを定義し、設計を明確化します。

---

# 📝 ラボ学習・設定サンプル例

### 1. ASAでの基本的なトラフィックゾーン構成
*   **問題**: 2つのISPインターフェイス（Gig0/0, Gig0/1）を1つのゾーン `ZONE-OUT` にまとめよ。
*   **設定例**:
```bash
zone ZONE-OUT
interface GigabitEthernet0/0
 zone ZONE-OUT
 nameif isp1
interface GigabitEthernet0/1
 zone ZONE-OUT
 nameif isp2
```

### 2. ASAでの非対称ルーティング解決
*   **要件**: `isp1` から出たパケットの戻りが `isp2` に来ても許可されるようにせよ。
*   **設定例**: 上記のZone構成により、ASAは同一ステートとして処理します。

### 3. FTDでの複数VLANを1ゾーンに統合 (FMC)
*   **要件**: サブインターフェイス Gig0/1.10, Gig0/1.20 を `Internal_Zone` に追加せよ。
*   **手順**: FMCで `Security Zone` オブジェクトを作成し、両インターフェイスを選択して保存。

### 4. ゾーンベースのアクセスコントロール (FTD)
*   **要件**: `Inside_Zone` から `Outside_Zone` へのWebトラフィックを許可せよ。
*   **設定**: ACPのSourceに `Inside_Zone`、Destinationに `Outside_Zone` を指定したルールを作成。

### 5. ASA ECMPスタティックルートの設定
*   **要件**: 同一ゾーン内の2つのゲートウェイに対し、デフォルトルートを分散させよ。
*   **設定例**:
```bash
route isp1 0.0.0.0 0.0.0.0 1.1.1.254 1
route isp2 0.0.0.0 0.0.0.0 2.2.2.254 1
```

### 6. 同一ゾーン内インターフェイス間通信の許可 (ASA)
*   **要件**: 同じゾーンに属するインターフェイス間での直接通信を許可せよ。
*   **設定例**: `same-security-traffic permit inter-interface`

### 7. トラフィックゾーンの統計確認
*   **課題**: ゾーンを通過している現在のコネクション数を確認せよ。
*   **コマンド**: `show zone ZONE-OUT`

### 8. FTD ゾーンベースNATの設定
*   **要件**: `Inside_Zone` からの全通信を外部IPでPATせよ。
*   **設定**: NATポリシーで送信元インターフェイスオブジェクトとして `Inside_Zone` を指定。

### 9. 透過モードFTDでのブリッジグループゾーン
*   **課題**: ブリッジグループのメンバーをゾーンに追加せよ。
*   **手順**: 物理インターフェイスをゾーンに追加し、BVIは管理用として維持。

### 10. ゾーン削除時の挙動確認
*   **課題**: ゾーンを削除した際、インターフェイス設定がどうなるか確認せよ。
*   **結果**: インターフェイスの名前（nameif）は残りますが、ゾーンの紐付けのみが解除されます。

---

# ❓ 想定試験問題

1.  **問題**: ASAにおいて、トラフィックゾーンを使用する主な技術的な利点は何か？
    *   **解答**: 非対称ルーティング環境におけるステートフル検査の維持と、ECMP (Equal-Cost Multi-Path) ルーティングのサポート。
2.  **問題**: FTDのアクセスコントロールポリシーで、物理インターフェイスを直接指定せずゾーンを使用することのメリットは？
    *   **解答**: ポリシーの抽象化。将来的にインターフェイスが追加・変更されても、ゾーンに紐付けるだけで既存のルールをそのまま適用できるため。
3.  **問題**: ASAの1つのトラフィックゾーンに含めることができるインターフェイスの最大数は？
    *   **解答**: 8つ。
4.  **問題**: `packet-tracer` を使用して非対称パスを検証する場合、何に注意すべきか？
    *   **解答**: 入力インターフェイスを指定する必要があるが、ゾーンが構成されている場合、結果出力で `Inspect` フェーズがゾーンを考慮して `Allow` になることを確認する。
5.  **問題**: トラフィックゾーンに属するインターフェイス間でセキュリティレベルが異なる場合、ASAはどう動作するか？
    *   **解答**: デフォルトでは高いレベルから低いレベルへの通信のみ許可される。ゾーン内であってもセキュリティレベルのルールは依然として適用される。

---

# 🔗 参考リソース

*   **Cisco Configuration Guides**
    *   [Cisco ASA Series Firewall CLI Configuration Guide, 9.4 - Traffic Zones](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/firewall/asa-94-firewall-config/zones-overview.html)
    *   [Cisco Firepower Threat Defense Configuration Guide for FMC, 7.0 - Security Zones](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/object_management.html#ID-2243-000003b5)
*   **Cisco Live (Videos & Slides)**
    *   [BRKSEC-3020: Troubleshooting Firewall Threat Defense (FTD)](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-3020)
*   **Command Reference**
    *   [Cisco ASA Series Command Reference - zone](https://www.cisco.com/c/en/us/td/docs/security/asa/command-reference/v-z/cmdref3/z1.html)
*   **Technical Notes**
    *   [ASA 9.x: Traffic Zones for Asymmetric Routing Support](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/115014-config-asa-9x-zones-00.html)

---

📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

---
