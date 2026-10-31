---
layout: default
title: 1.6.c-Geolocation
nav_order: 3
parent: 1.6-NGFW-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.6.c Geolocation

Cisco Secure Firewall Threat Defense (FTD) および Management Center (FMC) における **Geolocation（地理位置情報機能 / GeoDB）** は、Cisco Talos が提供する **Geolocation Database (GeoDB)** を利用し、パブリック IP アドレスを地理的国・地域（ISO 国コード、国名、大陸）に対応付け、通信の送信元・宛先の地理的位置に基づいてアクセス制御や脅威フィルタリングを動的に実施する機能です。

CCIE Security v6.1 Blueprint **1.6.c Geolocation** では、GeoDB の構造と更新メカニズム（自動・手動・エアギャップ環境）、Access Control Policy (ACP) の **`Location` 条件** におけるルールマッチング、Security Intelligence (SI) や Event Analytics との連携、RFC 1918 プライベート IP との境界関係、および GeoDB 更新時の **「Policy Deploy 不要」** という動作特性の正しい理解とトラブルシューティング能力が深く問われます。

---

## 📘 概要

* **機能概要**:
  * パブリック IP アドレスと国 (Country)・大陸 (Continent) のマッピングテーブルである Cisco GeoDB を基盤とし、アクセス制御ルールや Security Intelligence の評価条件として「送信元/宛先の国・地域」を指定可能にする機能。
  * 通信イベント（Intrusion Events、Connection Events、File Events）において、通信に関与したホストの国旗アイコンや国名、大陸情報をグラフィカルに自動表示・分析・フィルタリング可能にする。
* **利用目的**:
  * 個別のパブリック IP アドレスやサブネットブロックを管理者が手動で更新・登録することなく、特定の国や地域（例: サイバー攻撃の発生率が高い特定国、アノニマスプロキシ等）との通信を一括して動的にブロック・制御する。
  * 特定地域のみをサービス対象とする公開 Web サーバーや DMZ ネットワークに対して、対象外地域からの不要な偵察・ポートスキャン・ブルートフォース攻撃をネットワーク境界で早期遮断し、ファイアウォールおよび内部サーバーの処理負荷を削減する。
* **どのような場面で利用するか**:
  * 海外拠点を持たない国内専用ビジネスの公開サーバーに対する、特定国発インバウンドアクセスのブロック（Inbound Geolocation Blocking）。
  * 社内端末から、特定国に存在する不正 C2 サーバーやデータ漏洩先となり得る不審な地域宛てアウトバウンド通信の遮断（Outbound Geolocation Filtering）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **データベース構造** | **Cisco Geolocation Database (GeoDB)**: IP アドレス ↔ ISO Country Code / Country Name / Continent マッピング |
| **評価コンポーネント** | **Access Control Policy (ACP)**: Access Control Rules の `Location` (または Networks/Geolocation) タブ<br>**Security Intelligence (SI)**: Dynamic Feed / List 連携<br>**Events / Analytics**: Unified Events, Context Explorer, Geolocation Lookup |
| **更新メカニズム** | FMC による自動定期更新（Weekly Scheduled Task）、または手動更新（Internet Direct Download / Air-gapped File Upload） |
| **Policy Deploy 要件** | **Deploy 不要**（GeoDB 更新後、FMC から FTD へメモリ上更新データが自動同期され即座に適用される） |
| **オブジェクト管理** | `Objects > Object Management > Geolocation` にて国・大陸を組み合わせた再利用可能なオブジェクトを定義 |
| **対象 IP アドレス** | **ルーティング可能なパブリック IP アドレス専用**（RFC 1918 プライベート IP や NAT 前の内部 IP には適用不可） |
| **ライセンス要件** | **Base License**（GeoDB による基本アクセス制御は追加ライセンス不要。ただし SI 連携時は Threat License が必要） |
| **最新パッケージ構造** | バージョン 7.2+ 以降、制御用 Country Code パッケージと詳細文脈用 IP パッケージに分離提供 |

---

## 🏗 動作原理

Geolocation 機能では、FTD のデータプレーン（Snort エンジン）のメモリ上に常駐する GeoDB インデックスを参照してパケットの送信元・宛先 IP アドレスを即時に地理識別します。

```text
[ Ingress Public Packet ] (e.g., Src IP: 203.0.113.50, Dst IP: 192.168.100.10)
       │
       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Firepower Threat Defense (FTD) Packet Processing Architecture                     │
│                                                                                  │
│  [ LINA Firewall Engine ]                                                        │
│   ├── Sanity Checks & Existing Connection Lookup                                 │
│   └── Ingress Interface Inbound Processing                                       │
│            │                                                                     │
│            ▼                                                                     │
│  [ Snort Next-Gen Engine ]                                                       │
│   ├── 1. In-Memory GeoDB Lookup                                                  │
│   │        ├── Extract Src IP (203.0.113.50) ──► Match GeoDB ──► Country: China  │
│   │        └── Extract Dst IP (192.168.100.10) ──► Private IP ──► Country: Unknown│
│   │                                                                              │
│   ├── 2. Security Intelligence Evaluation                                        │
│   │        └── Check Geo-based SI Blacklist (If configured)                      │
│   │                                                                              │
│   ├── 3. Access Control Policy (ACP) Evaluation                                  │
│   │        └── Match Rule "BLOCK_GEO_CN":                                        │
│   │                 ├── Location Condition: Source = China                       │
│   │                 └── Action: Block with Reset                                 │
│   │                                                                              │
│   └── 4. Action Enforcement & Logging                                            │
│            ├── Send TCP RST / Drop Packet                                        │
│            └── Generate Connection Event (Flag: 🇨🇳 / Src: China) ──► FMC         │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

1. **GeoDB のロードとメモリ常駐**:
   * FTD の起動時および GeoDB アップデート時、Cisco GeoDB のバイナリマッピングテーブルが Snort エンジンのメモリ空間（RAM）へ直ちにロードされます。
2. **パケットの着信と IP アドレス抽出**:
   * インターフェイスに着信したパケットから、L3 送信元 IP アドレス (Src IP) および宛先 IP アドレス (Dst IP) が抽出されます。
3. **インメモリ GeoDB ルックアップ (In-Memory Lookup)**:
   * 抽出された IP アドレスが GeoDB 構造体と照合され、ISO Country Code (例: `CN`, `RU`, `US`, `JP`) および Continent Code (例: `AS`, `EU`, `NA`) のメタデータが属性として付与されます。
   * **注**: RFC 1918 プライベート IP アドレス（10.0.0.0/8、172.16.0.0/12、192.168.0.0/16）は GeoDB に登録されていないため、`Unknown` または属性なしとして処理されます。
4. **Security Intelligence (SI) 評価**:
   * SI で Geolocation リスト/フィードが指定されている場合、DPI (Deep Packet Inspection) 前段階で早期ドロップが実行されます。
5. **Access Control Policy (ACP) ルール評価**:
   * ルールリストを上から順に評価（First Match）：
     * ルールの **`Location` タブ** に設定された国・大陸条件（例: `Source Locations: China`）とパケットの国属性が照合されます。
6. **アクション執行とイベント生成**:
   * ルールのアクション（`Allow`, `Block`, `Block with reset`）が即座に執行されます。
   * FMC へ送出されるイベントログには、該当 IP に対応する国名、ISO コード、および国旗アイコンが紐付けられて記録されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **GeoDB 更新後の「Policy Deploy 不要」の仕様**:
   * **試験の罠**: FMC で GeoDB パッケージをアップデート（自動または手動）した際、画面上で通常行う「Deploy」操作を実行しなくても、**更新データはバックグラウンドで管理下の全 FTD へ自動配信・適用** されます。
   * ラボ試験で GeoDB を更新した際、不要な Deploy 操作を実行して時間を浪費しないよう注意が必要です。
2. **RFC 1918 プライベート IP に対する Geolocation ルール不一致**:
   * **よくあるミス**: 内部 LAN（例: `10.1.1.0/24`）を送信元として外部宛てのルールを作成する際、`Source Location: Japan` のように指定してしまう構成エラー。
   * プライベート IP は GeoDB に存在しないためルールにヒットしません。アウトバウンド通信で制御する場合は **`Destination Location`** に目的の国を指定する必要があります。
3. **大陸 (Continent) チェック時の包括動作**:
   * Geolocation オブジェクトやルール内で Continent（例: `Asia`）にチェックを入れると、その大陸に属する**すべての国が包含** されます。
   * 大陸配下の特定国を 1 つでもチェック解除（Uncheck）すると、Continent 自体のチェックは外れますが、それ以外のチェックが入った国々は個別アサイン状態として維持されます。
4. **エアギャップ（オフライン環境）での GeoDB 手動更新**:
   * 試験で FMC がインターネット未接続（Air-gapped）の場合、Cisco サポートサイトから事前に取得した signed パッケージ（`.sh.REL.tar` または `.tar`）を `System > Updates > Geolocation Updates` から手動アップロード・インストールする手順が問われます。
5. **CLI による GeoDB 適用確認とデバッグ**:
   * FTD CLI で **`show geodb info`** を実行し、現在アクティブな GeoDB バージョンおよび登録エントリ数を確認します。
   * **`system support firewall-engine-debug`** を使用して、パケット処理時に Snort が正しい国コード（例: `Country: China`）を識別しているかをリアルタイム追跡します。

---

## 🛠 設定方法

### 1. FMC GUI: Geolocation オブジェクトの定義手順

```text
[ Step 1: 再利用可能な Geolocation オブジェクトの作成 ]
1. FMC UI から Objects > Object Management に移動。
2. 左側メニューから "Geolocation" を選択し "Add Geolocation" をクリック:
   - Name: OBJ_GEO_HIGH_RISK_COUNTRIES
3. カントリーツリーから対象の国を選択:
   - Asia ➔ [✓] China
   - Europe ➔ [✓] Russian Federation
   - (必要に応じて Anonymous Proxy などの特殊カテゴリを選択)
4. "Save" をクリック。
```

---

### 2. FMC GUI: Access Control Policy への Geolocation 条件ルールの適用

```text
[ Step 1: ACP ルールの作成と Location 条件のアサイン ]
1. Policies > Access Control を開き対象の ACP を編集。
2. "Add Rule" を押下:
   - Name: BLOCK_HIGH_RISK_INBOUND
   - Action: Block with reset
   - Zones タブ: Source = OUTSIDE_ZONE, Destination = DMZ_ZONE
   - Location タブを選択:
     - Available Locations リストから "OBJ_GEO_HIGH_RISK_COUNTRIES" (または国を直接指定) を選択。
     - "Add to Source" を押下して Source Locations 欄へ移動させる。
3. "Save" をクリック。
4. 作成したルールを ACP リストの上位（Top）へ配置し、"Deploy" を実行。
```

---

### 3. FMC GUI: GeoDB の自動更新タスクのスケジュール構成

```text
[ Step 1: GeoDB 定期自動アップデートの設定 ]
1. FMC UI の System > Updates に移動。
2. "Geolocation Updates" タブを選択。
3. Recurring Geolocation Updates セクション:
   - [✓] Enable Recurring Weekly Updates from the Support Site にチェック。
   - Update Start Time: 毎週日曜日 02:00 (トラフィックの少ない時間帯を指定)
4. "Save" を押下。
```

---

### 4. FMC REST API: Geolocation オブジェクト情報および GeoDB ステータスの取得

```bash
# ===== FMC REST API を使用した GeoDB バージョン情報の GET =====
curl -k -X GET "https://fmc.corp.local/api/fmc_config/v1/domain/e276abec-e0f2-11e3-8169-6d9ed13b206f/object/geolocations" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json"
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI / FMC CLI) | 表示確認ポイント |
| :--- | :--- | :--- |
| **GeoDB の適用バージョン・データベースステータス確認** | <code>show geodb info</code> | アクティブな GeoDB バージョン、ビルド番号、登録国数 |
| **Snort 3 エンジンでの Geolocation マッチのリアルタイム追跡** | <code>system support firewall-engine-debug</code> | パケットの Src/Dst IP に対する `Geo Location: China` 等の識別ログ |
| **LINA データプレーンでの Location ルールマッチングの検証** | <code>packet-tracer input OUTSIDE tcp 203.0.113.50 12345 192.168.100.10 80</code> | `Phase: ACCESS-LIST` ➔ `Action: DROP (BLOCK_HIGH_RISK_INBOUND)` の表示 |
| **FMC 上での GeoDB バージョンの確認 (Web GUI 代替 CLI)** | <code>show version</code> (FMC CLI) | `Geolocation Database Version: ...` の出力行 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **Geolocation ルールを設定したが通信が一切ブロックされない** | ルールで送信元に RFC 1918 プライベート IP (例: `10.1.1.50`) が指定されており GeoDB に存在しない | <code>system support firewall-engine-debug</code> | 送信元がパブリック IP となる通信（WAN から DMZ 宛て等）でアサインするか、アウトバウンド制御時は Destination Location を指定する。 |
| **GeoDB の自動更新タスクがエラーで失敗する** | FMC がインターネットへアクセスできない、DNS 解決失敗、またはプロキシ認証エラー | FMC Message Center / <code>ping cisco.com</code> | `System > Configuration > Management Interfaces` で DNS 設定を確認。エアギャップ環境では手動ファイルアップロードを実施。 |
| **特定国をブロックしたところ、意図していない周辺国の通信までブロックされた** | Geolocation オブジェクトで Continent（大陸）全体にチェックを入れてしまっている | FMC Objects > Geolocation | 大陸のチェックを外し、ブロック対象とする特定の国のみを個別指定（Check）する。 |
| **GeoDB 手動アップロード時に「Invalid Package」エラーが発生する** | ソフトウェアバージョンに適合しないパッケージ、または伸張（un-tar）したファイルをアップロードしている | FMC Message Center | ダウンロードした `.sh.REL.tar` ファイルを解凍（解凍不可）せずにそのまま FMC へアップロードする。 |

---

## ⚠ 制限事項

* **プライベート IP アドレス非対応**:
  * RFC 1918（10.0.0.0/8、172.16.0.0/12、192.168.0.0/16）などのプライベート IP アドレスは、GeoDB 上で特定国・大陸へ割り当てることができません。
* **IP アドレス移転時の反映タイムラグ**:
  * ISP 間の IP アドレスの売買・移転が発生した場合、Cisco Talos が GeoDB パッケージを更新し、FMC へ適用されるまでに若干のタイムラグ（数日〜数週間）が生じることがあります。
* **プロキシ・VPN 通信の限界**:
  * クライアントが TOR、アノニマスプロキシ、海外の VPN サーバーを経由して通信を行っている場合、Geolocation 機能は「プロキシの出口 IP アドレスの国」として判定します（アノニマスプロキシ自体の遮断には Security Intelligence フィードを併用します）。

---

## 🔄 他技術との関連

* **Security Intelligence (SI)**: IP / Domain / URL リストと Geolocation を組み合わせ、DPI 前段階で早期ドロップを実施。
* **URL Filtering**: 海外にホストされた悪質 Web サイトに対し、URL カテゴリ制御と Geolocation 判定の二重フィルターを適用。
* **Unified Events ＆ Context Explorer**: 接続ログや侵入イベントに送信元・宛先国の国旗アイコンおよび ISO コードを動的表示。

---

## 🧩 比較表

### 1. Geolocation (GeoDB) vs Security Intelligence (SI) IP Blacklist vs Access Control Networks Condition

| 比較項目 | Geolocation (GeoDB) | Security Intelligence (IP) | Access Control Networks Condition |
| :--- | :--- | :--- | :--- |
| **制御識別単位** | **国 (Country) / 大陸 (Continent)** | 個別悪意 IP / CIDR / Feed | 静的 Network Object / Subnet |
| **更新方式** | GeoDB パッケージによる動的更新 | Talos Feed による 30分毎動的更新 | 管理者による手動設定 |
| **評価タイミング** | ACP ルール評価時 (または SI 内) | **Snort インスペクション初期 (早期)** | ACP ルール評価時 (順次 Match) |
| **主な用途** | 地域的セキュリティバウンダリ制御 | 既知の悪意ある IP からの攻撃遮断 | 組織内セグメント間・サブネット間制御 |

---

## 💡 ベストプラクティス

1. **必要な国のみを個別指定 (Specific Country Selection)**:
   * Continent（大陸）一括指定は、将来追加される国や意図しない地域の通信まで遮断する可能性があるため、明確に脅威対象となっている特定の国（Top Threat Countries）を個別にアサインします。
2. **週次自動更新タスク (Weekly Auto-Update Task) の構成**:
   * Cisco Talos が提供する最新の IP ↔ 国対応データを維持するため、毎週ネットワーク使用率の低い時間帯（例: 日曜深夜 02:00）に GeoDB の自動更新タスクを有効化しておきます。
3. **Inbound Block Rule の最上位配置**:
   * ACP ルールリストにおいて、特定国からの不要なインバウンドパケットを早期に Drop / Reset させるため、Geolocation Block ルールをリストの上位（Top）に配置します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Geolocation オブジェクト `OBJ_GEO_BLOCK_COUNTRIES` の定義

#### 問題
FMC 上で、`China` および `Russian Federation` を含む再利用可能な Geolocation オブジェクト `OBJ_GEO_BLOCK_COUNTRIES` を作成せよ。

#### 要件
* オブジェクト管理画面から構成すること。

#### 設定・操作手順

```text
1. FMC UI で Objects > Object Management > Geolocation を選択。
2. "Add Geolocation" をクリック:
   - Name: OBJ_GEO_BLOCK_COUNTRIES
3. カントリーツリーから以下にチェックを入れる:
   - Asia ➔ [✓] China
   - Europe ➔ [✓] Russian Federation
4. "Save" をクリック。
```

#### 検証方法
* FMC オブジェクト一覧に `OBJ_GEO_BLOCK_COUNTRIES` が追加され、2 カ国が含まれていることを確認。

---

### 例2: GeoDB 定期自動更新タスクの設定

#### 問題
FMC において、毎週日曜日午前 3 時 (03:00) に Cisco Support Site から最新の GeoDB を自動ダウンロード・インストールする定期タスクを有効化せよ。

#### 要件
* Geolocation Updates 画面で設定すること。

#### 設定・操作手順

```text
1. System > Updates > Geolocation Updates タブを開く。
2. Recurring Geolocation Updates セクション:
   - [✓] Enable Recurring Weekly Updates... にチェック。
   - Update Start Time: Sunday 03:00
3. "Save" をクリック。
```

#### 検証方法
* 設定画面で `Enable Recurring Weekly Updates` がオンとして保存されていることを確認。

---

### 例3: 特定国からの DMZ 宛てインバウンド通信の遮断

#### 問題
Geolocation オブジェクト `OBJ_GEO_BLOCK_COUNTRIES` を使用し、対象国から内部 DMZ Web サーバー (`192.168.100.10`) 宛ての着信通信を FTD 上で拒否せよ。

#### 要件
* ACP ルールのアクション: `Block with reset`

#### 設定・操作手順

```text
1. Policies > Access Control で対象 ACP を編集。
2. "Add Rule" をクリック:
   - Name: BLOCK_GEO_INBOUND
   - Action: Block with reset
   - Source Zones: OUTSIDE_ZONE
   - Destination Networks: 192.168.100.10
   - Location タブ: Available Locations から "OBJ_GEO_BLOCK_COUNTRIES" を追加 (Add to Source)。
3. Save をクリックし、ルールを上位に配置して Deploy。
```

#### 検証方法
* FTD CLI で `packet-tracer input OUTSIDE tcp 203.0.113.50 50000 192.168.100.10 80` を実行し、`BLOCK_GEO_INBOUND` ルールで Block されることを確認。

---

### 例4: 社内 LAN から特定地域宛てのアウトバウンド通信遮断

#### 問題
社内 LAN (`10.1.1.0/24`) から `Anonymous Proxy` カテゴリ宛てのアウトバウンド通信を検知・遮断せよ。

#### 要件
* Location 条件で Destination に指定すること。

#### 設定・操作手順

```text
1. ACP ルールを作成:
   - Name: BLOCK_ANONYMOUS_PROXIES
   - Action: Block
   - Source Zones: INSIDE_ZONE
   - Location タブ: Available Locations から "Anonymous Proxy" を選択し "Add to Destination" を押下。
2. Save して Deploy。
```

#### 検証方法
* FMC の Connection Events 画面で、該当ルールによって遮断された接続の Destination Country / Proxy 種別を確認。

---

### 例5: Geolocation と AVC (アプリケーション制御) の組み合わせルール

#### 問題
`China` に属する IP アドレスからの通信に対して、すべてのアプリケーションをブロックするのではなく、`SSH` および `Telnet` 通信のみをピンポイントでブロックせよ。

#### 要件
* Location 条件と Applications タブ条件を併用すること。

#### 設定・操作手順

```text
1. ACP ルールを作成:
   - Name: BLOCK_CHINA_MANAGEMENT_PROTOS
   - Action: Block with reset
   - Location タブ: Source Locations = China
   - Applications タブ: "SSH", "TELNET" を追加。
2. Save して Deploy。
```

#### 検証方法
* FTD CLI の `system support firewall-engine-debug` を有効化し、該当国からの SSH 通信が発生した際にルールに正常にマッチして Reset されることを検証。

---

### 例6: オフライン環境 (Air-gapped) での GeoDB 手動アップデート

#### 問題
インターネット接続を持たない FMC に対して、手動ダウンロードした GeoDB パッケージ (`Cisco_GEODB_Update-2026-08-01-001.sh.REL.tar`) を適用せよ。

#### 要件
* FMC GUI からアップロード・インストールすること。

#### 設定・操作手順

```text
1. System > Updates > Geolocation Updates タブを開く。
2. One-Time Geolocation Update セクション:
   - "Upload and install geolocation update" ラジオボタンを選択。
   - "Choose File" を押し、ローカルの GeoDB パッケージを選択。
3. "Import" をクリック。
```

#### 検証方法
* インストール完了後、`Help > About` 画面で Geolocation Database Version がアップロードしたビルド番号に更新されたことを確認（Deploy は実行不要）。

---

### 例7: FTD CLI を用いた GeoDB バージョンおよびビルドの確認

#### 問題
FTD CLI にログインし、現在アプライアンスにロードされている GeoDB のバージョン情報をコマンドラインから確認せよ。

#### 要件
* `show geodb info` コマンドを使用すること。

#### 設定・操作手順

```bash
> show geodb info
```

#### 検証方法
* CLI 出力において、`GeoDB Version` および `Total IP Ranges Loaded` の情報が表示されることを確認。

---

### 例8: Security Intelligence での Geolocation オブジェクト適用

#### 問題
Access Control Policy の Security Intelligence タブにおいて、Geolocation オブジェクト `OBJ_GEO_HIGH_RISK` を Blacklist にアサインし、DPI 前段階で早期ドロップさせよ。

#### 要件
* Security Intelligence 設定を使用すること。

#### 設定・操作手順

```text
1. Policies > Access Control で ACP を編集。
2. "Security Intelligence" タブを選択。
3. Network List / Feeds セクションにおいて、"OBJ_GEO_HIGH_RISK" を Blacklist 欄へドラッグ＆ドロップ。
4. Save して Deploy。
```

#### 検証方法
* 対象国からの通信が Snort の詳細ルールの評価を待たずに SI フェーズで早期ドロップされることを確認。

---

### 例9: FMC REST API による Geolocation オブジェクトの読み出し

#### 問題
FMC REST API を使用して、定義されている全 Geolocation オブジェクトの UUID および構成国リストを Curl コマンドで取得せよ。

#### 要件
* `/api/fmc_config/v1/domain/{domainUUID}/object/geolocations` エンドポイントを使用すること。

#### 設定・操作手順

```bash
# ===== Auth Token 取得後、Geolocation オブジェクト情報を GET =====
curl -k -X GET "https://fmc.corp.local/api/fmc_config/v1/domain/e276abec-e0f2-11e3-8169-6d9ed13b206f/object/geolocations" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json"
```

#### 検証方法
* 返却された JSON レスポンス内に、設定した Geolocation オブジェクトの名称および包含されている国コードが含まれることを確認。

---

### 例10: FMC Unified Events での Geolocation ログフィルタリング

#### 問題
FMC の Unified Events 画面において、`Source Country: Russian Federation` に合致する過去 24 時間のイベントログのみを抽出・分析せよ。

#### 要件
* Analysis > Unified Events 画面を使用すること。

#### 設定・操作手順

```text
1. FMC UI で Analysis > Unified Events を開く。
2. 検索フィルターバーに `SrcCountry: "Russian Federation"` を入力して Enter。
```

#### 検証方法
* イベントテーブル上にロシア連邦の国旗アイコン 🇷🇺 と対象ログのみがフィルタ表示されることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で Geolocation オブジェクトを作成し、`Europe` 大陸を選択して保存した。その後、特定の国（例: `France`）のみをリストから単独でチェック解除（Uncheck）した。この操作を行った際のオブジェクトの挙動として正しいものはどれか。

* A) オブジェクト自体が自動削除される。
* B) 大陸（Europe）全体のチェックは自動的に外れるが、France 以外のヨーロッパ各国に対する個別チェック状態は自動的に保持される。
* C) Europe 全体のチェックが入ったままとなり、France の除外は無視される。
* D) FTD アプライアンスが強制再起動する。

**正解**: **B**  
**解説**: Geolocation オブジェクトにおいて大陸 (Continent) を選択すると傘下の全国家が選択状態となりますが、その中の 1 つの国をチェック解除すると、大陸自体のチェックボックスは外れ、代わりにそれ以外の国々に個別チェックが入った状態に自動更新されます。

---

### 2. トラブルシュート問題

**問題**: 社内クライアント (`10.1.1.50`) から外部インターネットへの Web 通信を制御するため、ACP ルールで `Source Location: Japan` を指定して `Allow` ルールを作成した。しかし、クライアントからのアクセスは本ルールに一切マッチせず、後続のデフォルトブロックルールにヒットして切断されてしまった。原因として適切なものはどれか。

* A) FTD の GeoDB が最新バージョンに更新されていないため。
* B) クライアントの IP アドレス (`10.1.1.50`) が RFC 1918 プライベート IP であり、GeoDB に存在しないため。
* C) FMC 上で GeoDB 更新後の Deploy を実行していないため。
* D) FTD の Firewall Mode が Transparent Mode に設定されているため。

**正解**: **B**  
**解説**: **RFC 1918 プライベート IP アドレス**（10.0.0.0/8 等）は GeoDB に登録されておらず、どの国にもマッピングされません。社内クライアントからのアウトバウンド通信を Geolocation で制御する場合は、`Destination Location` に目的地のパブリック IP の国を指定する必要があります。

---

### 3. Design 問題

**問題**: 新規に導入する FTD において、Cisco Talos が提供する最新の地理位置情報を常に維持しつつ、更新作業に伴う通信の遮断や管理者の保守運用負荷を極力排除したい。適切な構成手法はどれか。

* A) 毎週末に FMC 上で GeoDB を手動アップデートし、全 FTD へ Deploy を実行する。
* B) FMC 上で GeoDB の週次自動更新タスク (`Recurring Weekly Updates`) を構成し、更新後の Deploy 操作は行わない（自動同期されるため）。
* C) FTD CLI 上で `cron` タスクを作成して GeoDB を更新する。
* D) Prefilter Policy で全ポートを Fastpath に設定する。

**正解**: **B**  
**解説**: **GeoDB の更新は FMC から FTD へメモリ上更新データとして自動同期されるため、Policy Deploy は不要** です。FMC 上で weekly の自動更新タスクを有効化しておくだけで、無停止で最新の GeoDB を維持できます。

---

### 4. 実装問題

**問題**: FTD CLI にログインし、現在アプライアンスの Snort エンジンにロードされている GeoDB のバージョン番号およびデータベースの正常性を確認するための正確なコマンドはどれか。

* A) <code>show version</code>
* B) <code>show geodb info</code>
* C) <code>show access-list</code>
* D) <code>show service-policy</code>

**正解**: **B**  
**解説**: FTD CLI で **`show geodb info`** を実行すると、アプライアンス上で現在アクティブな GeoDB のバージョン、ビルド番号、ロードされている IP レンジ数などの詳細情報を一括確認できます。

---

### 5. コンフィグ読解問題

**問題**: 管理者が FMC 上で GeoDB の手動アップデート（Cisco Support Site からの直接ダウンロード）を実行し、正常完了のメッセージが表示された。管理者は続けて managed FTD に対する Deploy ボタンを押そうとしたが、Deploy キューには変更通知が表示されていなかった。この挙動の理由として正しいものはどれか。

* A) FMC ↔ FTD 間の sftunnel が切断されているため。
* B) GeoDB の更新は FMC から FTD への自動データ同期メカニズムによって処理されるため、Access Control Policy の Deploy キューを生成しない正常な動作である。
* C) GeoDB パッケージの暗号化署名検証に失敗したため。
* D) FTD 上で Threat License が無効化されているため。

**正解**: **B**  
**解説**: **GeoDB のアップデートはポリシーコンフィグの変更として扱われない** ため、Deploy キュー（Deployment Page）に保留中の変更として表示されません。GeoDB ファイルは FTD へ直接同期・ロードされます。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Device Configuration Guide - Geolocation Objects and Rules](https://www.cisco.com/)
* [Cisco Technical Notes: Update and Troubleshoot Geolocation Database (GeoDB) on FMC](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Cisco Secure Firewall Policy Architecture (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で「Block incoming traffic from high-risk countries without requiring manual deployment after database updates」と要求された場合は、以下の通り構成します：
    1. `Objects > Object Management > Geolocation` で対象国をまとめたオブジェクトを作成。
    2. ACP にて `Source Location` に作成したオブジェクトを指定した `Block` ルールを追加して Deploy。
    3. `System > Updates > Geolocation Updates` で定期更新を有効化（GeoDB 更新自体は Deploy 不要であることを理解しておく）。
* **図解（GeoDB 判定と評価パイプライン）**:
  ```text
  [ Ingress Packet ] (Src IP: 203.0.113.10)
            │
            ▼
  [ Extract Src IP ] ──► [ Lookup in GeoDB Memory ] ──► Identified: "China (CN)"
            │
            ▼
  [ Security Intelligence ]
            ├── Match Geo Blacklist? ──► YES ──► [ Early Drop ]
            └── NO
                  │
                  ▼
  [ Access Control Policy Rules ]
            ├── Match Rule "BLOCK_GEO" (Location = China)? ──► YES ──► [ Block / Reset ]
            └── Match Rule "ALLOW_OTHER" ──────────────────► [ Allow & Inspect ]
  ```
* **注意点**:
  * ラボ試験で Geolocation ルールを作成した後は、`packet-tracer` や CLI デバッグコマンド (`system support firewall-engine-debug`) を活用して、疑似パケットのパブリック IP に対する国識別およびルールマッチが意図通り行われていることを必ず確認してください。

---

ご不明な点や追加の解説が必要な場合は、お気軽にお知らせください。復習や実機演習の際にお役立ていただければ幸いです！
