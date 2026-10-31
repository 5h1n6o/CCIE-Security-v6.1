---
layout: default
title: 1.5-NGIPS-deployment-modes
nav_order: 5
parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.5 Cisco NGIPS deployment modes

Cisco Secure Firewall Threat Defense (FTD) および Firepower NGIPS アプライアンスにおける **NGIPS Deployment Modes（NGIPS 導入モード）** は、ネットワークトラフィックの検査手法（インライン防御 vs パッシブ検知）および物理・論理インターフェイスの配置方法を定義するセキュリティアーキテクチャの根幹です。

CCIE Security v6.1 Blueprint **1.5 (1.5.a In-line, 1.5.b Passive, 1.5.c TAP)** では、各種モードの動作原理の違い、パケット処理パス、高可用性オプション（Propagate Link State、Snort Fail Open、Hardware Bypass）、非同期ルーティング（Asynchronous Routing）の構成、および障害発生時のトラブルシューティングが深く問われます。

---

## 📘 概要

* **機能概要**:
  * **In-line Mode (インラインモード / 1.5.a)**: 装置をネットワークの通信経路上に物理的・論理的に挿入（"Bump in the wire"）し、トラフィックをリアルタイムで直接検査・遮断（Drop / Reset）する動的防御モード。
  * **Passive Mode (パッシブモード / 1.5.b)**: ネットワークスイッチの SPAN/RSPAN/ERSPAN ポートや外部物理 TAP からトラフィックの「コピー」を受信し、通信に影響を与えずに帯域外（Out-of-band）で脅威を監視・検知（Detection-only）するモード。
  * **TAP / Inline Tap Mode (インラインタップモード / 1.5.c)**: 物理ケーブリングはインライン構成のまま、内部処理としてパケットのコピーに対してのみ Snort インスペクションを実行する検出専用モード。将来的なインラインインスペクション（遮断）への切り替えを配線変更なしで実現可能。
* **利用目的**:
  * リアルタイムでのマルウェア、ゼロデイ攻撃、悪意あるトラフィックの即時ブロック（In-line）。
  * 既存ネットワークの通信遅延や可用性に影響を与えずに、脅威の発生状況や可視性を評価（Passive / Inline Tap）。
  * 段階的な NGIPS 導入ポリシーの確立（Inline Tap で評価 ➔ In-line へ移行）。
* **どのような場面で利用するか**:
  * **In-line**: 社内 LAN とインターネットの境界、DMZ 境界、データセンター内の重要セグメント間など、確実に攻撃を阻止したい箇所。
  * **Passive**: 制御システム（ICS/SCADA）や医療機器ネットワーク、低遅延を極限まで要求される金融取引ネットワーク等、わずかなパケットドロップや遅延も許容されない環境。
  * **Inline Tap**: 運用中のサービスを一時的に止めずに IPS シグネチャの誤検知（False Positive）リスクやパフォーマンス影響を事前評価したい事前検証・移行フェーズ。

---

## 🔑 要点

| 項目 | In-line Mode | Passive Mode | TAP (Inline Tap) Mode |
| :--- | :--- | :--- | :--- |
| **配置方式** | 帯域内（In-band / Bump in the wire） | 帯域外（Out-of-band / SPAN / TAP） | 帯域内（物理配線）/ 内部コピー処理 |
| **トラフィック遮断能力** | **可能** (Drop, Block, Reset) | **不可** (検知のみ / オプションで TCP Reset) | **不可** (検知のみ / "Would Have Dropped") |
| **必要ポート数** | 最低 **2 ポート** (1 インラインペア) | 最低 **1 ポート** (Rx のみ) | 最低 **2 ポート** (1 インラインペア) |
| **IP アドレス設定** | **不要** (L2/L3 アドレス非依存) | **不要** (ERSPAN 送受信時は除外) | **Don't care** (不要) |
| **ネットワークへの影響** | 障害時に通信影響の可能性あり (バイパス設定で緩和) | **完全なし** (パケット遅延・切断ゼロ) | **完全なし** (オリジナルパケットは即送出) |
| **インライン切替の手間** | 最初からインライン | **ケーブル再配線が必要** | **FMC GUI の設定変更のみ（配線変更不要）** |
| **障害耐性機能** | Propagate Link State, Fail Open, Hardware Bypass | 装置ダウン時も本線通信に影響なし | 装置ダウン時も本線通信に影響なし |

---

## 🏗 動作原理

各導入モードにおけるトラフィックフローと Snort インスペクションエンジンの位置付けは以下の通りです。

```text
1. In-line Mode (1.5.a)
   [ Client ] ──► ( Ingress Port ) ──► [ LINA / Snort Engine ] ──► ( Egress Port ) ──► [ Server ]
                                              │
                                       ( Block / Drop )

2. Passive Mode (1.5.b)
   [ Client ] ────────────────────────────────────────────────────────────────────► [ Server ]
                     │ (Traffic Mirrored via SPAN / RSPAN / ERSPAN)
                     ▼
             ( Passive Port ) ──► [ Snort Engine ] (Alerts Only)

3. TAP / Inline Tap Mode (1.5.c)
   [ Client ] ──► ( Ingress Port ) ────────┬──────────────────────► ( Egress Port ) ──► [ Server ]
                                           │ (Copy Packet Internal)
                                           ▼
                                   [ Snort Engine ] (Alerts Only: "Would Have Dropped")
```

---

## ⚙ 動作シーケンス

1. **パケット受信と初期振り分け**:
   * **In-line**: パケットは Ingress インターフェイスに入り、LINA データプレーンの最小限のチェックを受けた後、即座に Snort 2/3 インスペクションエンジンに渡されます。
   * **Passive**: スイッチの SPAN ポートや TAP 装置から複製されたパケットが Passive インターフェイス（受信用）に着信。LINA はレイヤ 2/3 の状態保持を行わず、直接 Snort エンジンへ送出します（エグレス送信は行いません）。
   * **Inline Tap**: パケットが Ingress インターフェイスに着信した瞬間、ハードウェア/ASIC レベルでパケットが 2 つに分岐されます。本線パケットは Egress ポートへ即時転送され、複製パケットが Snort エンジンへ渡されます。
2. **Snort ポリシー評価**:
   * ネットワーク解析ポリシー（NAP）による事前処理（デフラグ、TCP ストリーム再構築、正規化）を実施。
   * アクセス制御ポリシー (ACP) および侵入防御ポリシー (Intrusion Policy) のルール（シグネチャ）と照合。
3. **アクションの実行**:
   * **In-line**: シグネチャが `Drop and Generate Events` にマッチした場合、Snort はパケットを破棄（Drop）し、必要に応じて TCP RST を両端に送信。
   * **Passive**: シグネチャが `Drop` に設定されていても、パケット破棄は行われず、`Intrusion Event` のみが生成されます。
   * **Inline Tap**: シグネチャが `Drop` にマッチした場合、本線パケットはすでに通過済みのため破棄できません。FMC のイベントビューアには **`Would Have Dropped`** (インラインであれば破棄されていた) という特殊な理由付きでログが記録されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Inline Set (インラインセット) の構築要件**:
   * インラインモードを使用する場合、単に 2 つの物理ポートを有効化するだけでは動作しません。
   * **手順**:
     1. インターフェイスの Mode を `None` に設定して Enabled にする。
     2. `Devices > Device Management > Inline Sets` タブで **Inline Set** を新規作成。
     3. 利用可能なペアから **Interface Pair**（例: `GigabitEthernet0/0 <-> GigabitEthernet0/1`）を結合して Inline Set にアサインする。
2. **Propagate Link State (リンク状態伝搬) の設定理由**:
   * インラインペアの一方のリンク（例: WAN 側 switch ポート）が Down した際、対向のポート（例: LAN 側 router ポート）のリンクを自動的に Down させる機能。
   * **目的**: 上流・下流のルーターやスイッチにリンクダウンを即座に検知させ、動的ルーティングプロトコル（OSPF、EIGRP）や HSRP/VRRP の高速な迂回切り替え（Failover）を誘発させるため。
3. **Snort Fail Open (Failsafe) vs Hardware Bypass (Fail-to-Wire)**:
   * **Snort Fail Open**: ソフトウェア機能。Snort プロセスが過負荷 (Busy) または停止 (Down) した際、Traffic を未検査のままバイパス通過させる設定。
   * **Hardware Bypass**: 物理ハードウェア機能（Firepower 2100/4100/9300 の特定 NetMod）。電源断やハードウェア障害時に物理リレーが閉じ、機械的に銅線/光ファイバーをバイパス接続（Fail-to-Wire）させる機能。
4. **非同期ルーティング (Asynchronous Routing) の対処**:
   * 往路と復路が異なる FTD ポートペアを通過する環境では、**往路ペアと復路ペアを同一の Inline Set に含める** 必要があります。これにより Snort は両方のペアを単一のセッションフローとして認識・状態管理できます。
5. **ERSPAN Passive インターフェイスの制限事項**:
   * ERSPAN (Encapsulated Remote SPAN) を使用した Passive モードは、FTD が **`Routed Firewall Mode`** で動作している場合のみサポートされます（`Transparent Mode` では ERSPAN は利用不可）。

---

## 🛠 設定方法

### 1. FMC GUI: In-line Mode (Inline Pair ＆ Inline Set) の構成

```text
[ Step 1: 物理インターフェイスの有効化 ]
1. Devices > Device Management を開き、対象 FTD の Interfaces タブを選択。
2. GigabitEthernet0/0 を編集:
   - Name: INLINE_IN
   - Mode: None
   - Enabled: [✓] チェック
3. GigabitEthernet0/1 を編集:
   - Name: INLINE_OUT
   - Mode: None
   - Enabled: [✓] チェック
4. "Save" をクリック。

[ Step 2: Inline Set の作成と Inline Pair のアサイン ]
1. "Inline Sets" タブを選択し "Add Inline Set" をクリック。
2. General タブ:
   - Name: INSIDE_OUTSIDE_INLINE_SET
   - Available Interfaces Pairs から "INLINE_IN <-> INLINE_OUT" を選択し "Add" をクリック。
3. Advanced タブ:
   - [✓] Propagate Link State
   - [✓] Snort Fail Open (Busy [✓], Down [✓])
4. "OK" ➔ "Save" をクリックし、対象 FTD へ Deploy。
```

---

### 2. FMC GUI: TAP (Inline Tap) Mode への切り替え手順

```text
1. Devices > Device Management > Inline Sets タブを開く。
2. 既存の "INSIDE_OUTSIDE_INLINE_SET" の編集 (鉛筆アイコン) をクリック。
3. "Advanced" タブを選択。
4. [✓] Tap Mode にチェックを入れる。
5. "OK" ➔ "Save" をクリックし、対象 FTD へ Deploy。
   ※ 警告メッセージ表示: 配線変更なしで検出専用 (Tap) モードへ移行します。
```

---

### 3. FMC GUI: Passive Interface (SPAN 用) の構成

```text
1. Devices > Device Management > Interfaces タブを開く。
2. GigabitEthernet0/2 を編集:
   - Name: PASSIVE_SPAN_IF
   - Mode: "Passive" を選択
   - Enabled: [✓] チェック
   - Security Zone: PASSIVE_ZONE (新規作成または既存アサイン)
3. "OK" ➔ "Save" をクリックし、対象 FTD へ Deploy。
```

---

### 4. FMC GUI: Passive ERSPAN Interface の構成

```text
1. FTD が Routed Firewall Mode であることを確認。
2. Devices > Device Management > Interfaces を開く。
3. GigabitEthernet0/3 を編集:
   - Name: ERSPAN_IF
   - Mode: "Erspan" を選択
   - Enabled: [✓] チェック
   - Erspan ID: 100
   - IP Address: 172.16.10.50 /24 (ERSPAN カプセル化終端用 IP)
4. "OK" ➔ "Save" をクリックして Deploy。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI) | 表示確認ポイント |
| :--- | :--- | :--- |
| **Inline Set 全体のステータス・設定確認** | <code>show inline-set</code> | `Tap mode is on/off`、`Propagate-link-state option is on`、`Bridge Group ID` |
| **特定物理ポートのモードおよびドロップ統計確認** | <code>show interface GigabitEthernet 0/0</code> | `IPS Interface-Mode: Inline / Passive / Inline Tap` の確認 |
| **ハードウェアバイパス (Hardware Bypass) の状態確認** | <code>show module</code> / <code>show inline-set</code> | `hardware-bypass mode is enabled/disabled` |
| **LINA パケットトレーサーによる NGIPS 処理確認** | <code>packet-tracer input INLINE_IN icmp 10.1.1.1 8 8 10.1.2.1</code> | `Phase: NGIPS-MODE` ➔ `Result: ALLOW` の内部処理フロー |
| **Snort エンジンのリアルタイム統計・落とし確認** | <code>system support firewall-engine-debug</code> | 通信パケットが Snort に渡された際のアクション出力 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **インラインペアの一方のケーブルを抜いても対向ポートが Down にならない** | Inline Set の `Propagate Link State` が有効化されていない | <code>show inline-set</code> | FMC の Inline Set 設定画面（Advanced タブ）で `Propagate Link State` にチェックを入れて Deploy。 |
| **Inline Tap モードでルールを Drop に設定したのに通信が遮断されない** | Inline Tap モードはコピーパケットを検査する「検出専用」仕様であるため | <code>show inline-set</code> | 仕様通りの動作。通信を実際に遮断したい場合は Inline Set の `Tap Mode` チェックを外す。 |
| **ERSPAN インターフェイス設定時に対象モードが選択できない** | FTD が Transparent Firewall Mode で動作している | <code>show running-config firewall</code> | ERSPAN は Routed Mode 専用機能。FTD の Firewall Mode を Routed Mode へ変更する。 |
| **非同期通信（往路と復路でポートが異なる）時にセッションが意図せず切断される** | 往路ペアと復路ペアが異なる Inline Set に分割配置されている | <code>show inline-set</code> | 往復の両ポートペアを単一の Inline Set に含めることで、Snort が同一フローとして正しく認識。 |
| **In-line モード配下で BFD (Bidirectional Forwarding Detection) パケットがドロップする** | FTD が BFD Echo パケット（送信元/宛先 IP が同一）を LAND 攻撃と誤判定する | <code>show asp drop</code> | 該当 BFD ピア間で BFD Echo 機能を無効化（`no bfd echo`）するか、アクセスルールで不検査設定。 |

---

## ⚠ 制限事項

* **NAT および Routing 機能の非サポート (IPS-Only 形式時)**:
  * Inline Set および Passive インターフェイス（IPS-Only モード）では、LINA による NAT、動的ルーティングプロトコル終端、VPN 終端、DHCP サーバー機能はサポートされません。
* **Q-in-Q (Double VLAN Tagging) サポート制限**:
  * Inline Set および Passive インターフェイスは最大 2 つの 802.1Q ヘッダー (Q-in-Q) を透過処理できますが、Firepower 4100/9300 シャーシでは 1 タグのみサポートされます。
* **Virtual Appliance (FTDv / NGIPSv) におけるハイパーバイザ要件**:
  * ESXi 等の仮想環境で Inline Pair や Passive モードを正常動作させるには、vSwitch の Port Group 設定で **`Promiscuous Mode: Accept`**、**`MAC Address Changes: Accept`**、**`Forged Transmits: Accept`** の 3 項目を必ず有効化する必要があります。

---

## 🔄 他技術との関連

* **Routed vs Transparent Firewall Mode**: NGFW としてのデータプレーン動作モード。IPS-Only インターフェイス (Inline Set / Passive) はどちらの Firewall Mode 上でも共存・配置可能。
* **Network Analysis Policy (NAP)**: Inline / Passive で受信したパケットの初期正規化（Normalization）、ストリーム再構築、プロトコルプリプロセッサの処理を定義。
* **Snort 2 vs Snort 3 Engine**: Snort 3 ではマルチスレッド処理により Inline Tap および Passive モードでの処理パフォーマンスが大幅に向上。

---

## 🧩 比較表

### 1. In-line vs Passive vs TAP (Inline Tap)

| 比較項目 | In-line Mode | Passive Mode | TAP (Inline Tap) Mode |
| :--- | :--- | :--- | :--- |
| **通信パスの位置** | インライン (本線) | 帯域外 (コピー受信) | 物理インライン / 処理コピー |
| **脅威防御 (Block)** | **リアルタイム遮断可能** | 遮断不可 (検知のみ) | 遮断不可 (Would Have Dropped) |
| **遅延 (Latency)** | わずかなインスペクション遅延発生 | **遅延ゼロ** | **遅延ゼロ** |
| **スイッチ設定** | 不要 | **SPAN/RSPAN/TAP 設定が必須** | 不要 |
| **運用移行の容易性** | 最初から防御運用 | 物理配線変更が必要 | **GUI 操作のみで即時インライン化** |

### 2. Fault Tolerance (障害対策技術) 比較

| 技術名 | 処理層 | トリガー | 障害発生時の動作 |
| :--- | :--- | :--- | :--- |
| **Propagate Link State** | L1 / L2 物理リンク | 一方のポートの Link Down | 対向ポートの Link を物理的に強制 Down させる |
| **Snort Fail Open** | ソフトウェア (Snort) | Snort プロセスの Busy / Crash | LINA パスでパケットを未検査のまま通過（Pass）させる |
| **Hardware Bypass** | 物理ハードウェア (Relay) | 電源断 / システム致命的障害 | 物理リレーが短絡し、直結銅線として通信をバイパス |

---

## 💡 ベストプラクティス

1. **新規 NGIPS 導入時の段階的切り替え設計**:
   * 新規拠点への導入時は、物理配線をインラインで接続した上で、FMC 上で **`Inline Tap Mode`** として構成します。1〜2 週間運用して誤検知（False Positive）ルールをチューニングした後、メンテナンスウィンドウで Tap Mode チェックを外して **`In-line Prevention`** へ即座に切り替えます。
2. **Propagate Link State の常時有効化**:
   * すべての Inline Set において `Propagate Link State` を有効化し、上流/下流スイッチの STP や動的ルーティング（OSPF/BGP）がリンク障害を即座に検知して迂回できるように設計します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Inline Pair ＆ Inline Set (In-line Prevention) の基本構築

#### 問題
FTD の 2 つの物理ポート (`GigabitEthernet0/0` と `GigabitEthernet0/1`) を使用して、インライン防御モード (In-line Mode) を構成せよ。

#### 要件
* インラインセット名: `SET_INLINE_PROD`
* 両インターフェイスの物理モードをアサインすること。

#### 設定・操作手順

```text
1. Devices > Device Management > Interfaces タブを開く。
2. GigabitEthernet0/0 を編集: Name = INSIDE_IF, Mode = None, Enabled = [✓]
3. GigabitEthernet0/1 を編集: Name = OUTSIDE_IF, Mode = None, Enabled = [✓]
4. Inline Sets タブを開き "Add Inline Set" をクリック:
   - Name: SET_INLINE_PROD
   - Selected Interface Pair: INSIDE_IF <-> OUTSIDE_IF を追加。
5. 保存後、FTD へ Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`INSIDE_IF` と `OUTSIDE_IF` がアサインされ `Tap mode is off` であることを確認。

---

### 例2: Inline Tap Mode (In-line Detection-only) への動的切替設定

#### 問題
既存のインラインセット `SET_INLINE_PROD` を、配線変更を行わずにトラフィックに影響を与えない Detection-only モード（Inline Tap）へ変更せよ。

#### 要件
* `Tap Mode` オプションを有効化すること。

#### 設定・操作手順

```text
1. Devices > Device Management > Inline Sets タブを開く。
2. "SET_INLINE_PROD" の編集をクリック。
3. "Advanced" タブを選択。
4. [✓] Tap Mode にチェックを入れる。
5. OK ➔ Save をクリックし、FTD へ Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`Tap mode is on` と表示されることを確認。

---

### 例3: SPAN トラフィック用 Passive Interface (Passive IDS) の構成

#### 問題
スイッチの SPAN ポートから転送されるトラフィックを受信するため、FTD の `GigabitEthernet0/2` ポートを Passive モードとして構成せよ。

#### 要件
* インターフェイス名: `SPAN_MONITOR_IF`
* セキュリティゾーン: `ZONE_PASSIVE`

#### 設定・操作手順

```text
1. Devices > Device Management > Interfaces タブを開く。
2. GigabitEthernet0/2 を編集:
   - Name: SPAN_MONITOR_IF
   - Mode: Passive
   - Enabled: [✓]
   - Security Zone: ZONE_PASSIVE
3. OK ➔ Save をクリックして Deploy。
```

#### 検証方法
* FTD CLI で `show interface GigabitEthernet0/2` を実行し、`IPS Interface-Mode: Passive` であることを確認。

---

### 例4: L3 離隔環境における Passive ERSPAN インターフェイスの構成

#### 問題
遠隔地の Catalyst スイッチから GRE トンネル経由で送信される ERSPAN トラフィック（ERSPAN ID: 50）を受信するため、FTD (`GigabitEthernet0/3`) 上に ERSPAN インターフェイスを構成せよ。

#### 要件
* FTD Firewall Mode: Routed Mode
* ERSPAN IP: `192.168.100.50/24`

#### 設定・操作手順

```text
1. Devices > Device Management > Interfaces を開く。
2. GigabitEthernet0/3 を編集:
   - Name: ERSPAN_MONITOR_IF
   - Mode: Erspan
   - Erspan ID: 50
   - IP Address: 192.168.100.50 /24
3. OK ➔ Save をクリックして Deploy。
```

#### 検証方法
* 対向スイッチから ERSPAN トラフィックを送信させ、FTD CLI で `show interface GigabitEthernet0/3` の Rx パケットカウントが増加していることを確認。

---

### 例5: 障害耐性を担保する Propagate Link State の設定

#### 問題
インラインセット `SET_INLINE_PROD` において、一方の物理リンク障害時に対向の物理リンクも自動的に遮断させるよう `Propagate Link State` を構成せよ。

#### 要件
* Inline Set の Advanced タブで設定すること。

#### 設定・操作手順

```text
1. Devices > Device Management > Inline Sets を開く。
2. "SET_INLINE_PROD" の編集画面を開く。
3. "Advanced" タブを選択。
4. [✓] Propagate Link State にチェックを入れる。
5. OK ➔ Save をクリックして Deploy。
```

#### 検証方法
* 片方のケーブルを引き抜き、対向ポートの LED が消灯し FTD CLI の `show inline-set` で両インターフェイスの Current-Status が Down に推移することを確認。

---

### 例6: Snort 障害時の通信維持を図る Snort Fail Open の設定

#### 問題
Snort インスペクションエンジンが高負荷 (Busy) に達した際、またはプロセス障害 (Down) 発生時に、パケットドロップを回避して通信を通過させる `Snort Fail Open` を設定せよ。

#### 要件
* `Busy` および `Down` の両条件で有効化すること。

#### 設定・操作手順

```text
1. Devices > Device Management > Inline Sets を開く。
2. "SET_INLINE_PROD" を編集 ➔ "Advanced" タブ。
3. Snort Fail Open セクションで以下をチェック:
   - [✓] Busy
   - [✓] Down
4. OK ➔ Save をクリックして Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`Fail-open for snort down is on` および `Fail-open for snort busy is on` を確認。

---

### 例7: 非同期ルーティング環境における Multiple Inline Pairs の統合

#### 問題
往路トラフィック（`Gi0/0 <-> Gi0/1`）と復路トラフィック（`Gi0/2 <-> Gi0/3`）が異なる物理ポートを通過する非同期ルーティング環境において、Snort が単一セッションとして正常追跡できるよう 2 つのインラインペアを同一の Inline Set へアサインせよ。

#### 要件
* インラインセット名: `ASYNC_INLINE_SET`

#### 設定・操作手順

```text
1. Gi0/0 (IN_1), Gi0/1 (OUT_1), Gi0/2 (IN_2), Gi0/3 (OUT_2) を有効化。
2. Inline Sets で "Add Inline Set" を作成:
   - Name: ASYNC_INLINE_SET
   - Selected Interface Pairs に "IN_1 <-> OUT_1" と "IN_2 <-> OUT_2" の両方を追加。
3. Save をクリックして Deploy。
```

#### 検証方法
* 非同期トラフィックを発生させ、FMC の Connection Events 画面で Ingress / Egress ポートが正しく記録され、セッションが切断されないことを確認。

---

### 例8: Firepower 4100/9300 ハードウェアバイパス (Hardware Bypass) の構成

#### 問題
Firepower 4100 アプライアンスの Fail-to-Wire 対応 NetMod ポートペアにおいて、全障害時に物理リレーをショートさせて通信をバイパスさせる `Hardware Bypass` を有効化せよ。

#### 要件
* `Hardware Bypass` オプションを有効化すること。

#### 設定・操作手順

```text
1. Devices > Device Management > Inline Sets を開く。
2. ハードウェアバイパス対応ポートペアを含む Inline Set を編集。
3. "Advanced" タブを選択。
4. [✓] Hardware Bypass にチェックを入れる。
5. Save をクリックして Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`hardware-bypass mode is enabled` を確認。

---

### 例9: Intrusion Policy での "Would Have Dropped" 評価の検証

#### 問題
Inline Tap モードで動作している FTD において、侵入ルール（SID: `2000001`）のアクションを `Drop and Generate Events` に設定した際、生成されるイベントログの挙動を検証せよ。

#### 要件
* ログ上で "Would Have Dropped" と出力されることを確認すること。

#### 設定・操作手順

```text
1. Inline Set が Tap Mode であることを確認。
2. Intrusion Policy で Rule 2000001 を "Drop and Generate Events" に設定して Deploy。
3. 該当の攻撃パケットを送信。
```

#### 検証方法
* FMC の `Analysis > Intrusions > Events` を開き、該当イベントの Inline Result カラムに **`Would Have Dropped`** と表示され、通信自体は遮断されていないことを確認。

---

### 例10: Inline VLAN Pair (802.1Q Subinterface / Inline-on-a-Stick) の構成

#### 問題
単一の物理ポート `GigabitEthernet0/0` 上の 2 つの VLAN (`VLAN 10` と `VLAN 20`) 間で VLAN タグの掛け替えを行いながらインラインインスペクションを実行する `Inline VLAN Pair` を構成せよ。

#### 要件
* Subinterface: `Gi0/0.10` (VLAN 10) および `Gi0/0.20` (VLAN 20)

#### 設定・操作手順

```text
1. Devices > Device Management > Interfaces を開く。
2. GigabitEthernet0/0 に Subinterface を追加:
   - Gi0/0.10 (VLAN ID: 10, Mode: None)
   - Gi0/0.20 (VLAN ID: 20, Mode: None)
3. Inline Sets タブで "Add Inline Set" を作成:
   - Selected Interface Pair: "Gi0/0.10 <-> Gi0/0.20" をアサイン。
4. Save をクリックして Deploy。
```

#### 検証方法
* スイッチ側 Trunk ポート経由で VLAN 10 からパケットを送出し、FTD が Snort 検査後に VLAN 20 のタグに書き換えて同一ポートから転送（Hair-pinning）することを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で FTD の物理ポート `Gi0/0` と `Gi0/1` を使用して Inline Set を構成した。対向スイッチのリンク障害テストを行うため `Gi0/0` の LAN ケーブルを抜いたが、`Gi0/1` 側の対向ルーターポートのリンクが Up のままであり、ルーターのルーティング切り替えが発生しなかった。FMC 上で不足している設定はどれか。

* A) Snort Fail Open が無効化されているため。
* B) Inline Set の Advanced 設定で `Propagate Link State` がチェックされていないため。
* C) FTD の Firewall Mode が Transparent Mode に設定されているため。
* D) `Strict TCP Enforcement` が無効化されているため。

**正解**: **B**
**解説**: **Propagate Link State（リンク状態伝搬）** が有効化されていない場合、インラインペアの一方のリンクが Down しても他方のリンク状態は維持されます。これを有効にすることで、一方の障害時に対向ポートのリンクも自動的に Down させ、周りのルーターやスイッチにルーティング/Failover 迂回処理を発生させることができます。

---

### 2. トラブルシュート問題

**問題**: 新しく導入した FTD アプライアンスにおいて、トラブルシューティング目的で Inline Set の `Tap Mode` を有効化した。その後、侵入ポリシーで明確に `Drop and Generate Events` と定義したシグネチャに合致する攻撃パケットを送信したが、ターゲットホストへの通信が遮断されずに到達してしまった。この原因として適切なものはどれか。

* A) FTD のライセンスが Threat から Base にダウングレードしたため。
* B) Inline Tap モードはパケットのコピーを検査する検出専用モードであり、本線パケットを破棄（Drop）する能力を持たない仕様であるため。
* C) FTD 側で TCP Normalization が無効化されているため。
* D) FMC と FTD 間の sftunnel が切断されているため。

**正解**: **B**
**解説**: **Inline Tap モード** は、物理ポートはインライン配置されているものの、内部的には複製されたパケットのコピーに対してのみ Snort インスペクションを行う「検出専用モード」です。そのため、ルールアクションが Drop であっても本線パケットは破棄されず通過します（イベントログには `Would Have Dropped` と表示されます）。

---

### 3. Design 問題

**問題**: 遠隔拠点にある複数の Catalyst スイッチから、L3 ネットワークを越えて GRE カプセル化されたミラーリングトラフィックを本社の FTD アプライアンスへ集約し、検出専用で監視したい。FTD 上で構成すべき適切なインターフェイスモードおよび前提条件はどれか。

* A) Inline Tap モード (Transparent Firewall Mode)
* B) Passive ERSPAN モード (Routed Firewall Mode)
* C) Passive モード (Transparent Firewall Mode)
* D) Inline Set モード (Multi-Instance Mode)

**正解**: **B**
**解説**: 遠隔スイッチから GRE カプセル化経由でミラーリングパケットを受信する方式は **Passive ERSPAN** モードです。ERSPAN インターフェイスは FTD が **`Routed Firewall Mode`** で動作している場合のみサポートされます。

---

### 4. 実装問題

**問題**: FTD CLI にログインし、現在構成されている Inline Set の動作モード（In-line か Inline Tap か）およびリンク状態伝搬の設定の有無を一括で検証するための正しいコマンドはどれか。

* A) <code>show interface ip brief</code>
* B) <code>show inline-set</code>
* C) <code>show running-config interface</code>
* D) <code>show service-policy</code>

**正解**: **B**
**解説**: FTD CLI で **`show inline-set`** を実行すると、定義されている Inline Set 名、構成メンバーインターフェイスペア、`Tap mode is on/off`、`Propagate-link-state option is on/off`、および `Fail-open` 状態がまとめて表示されます。

---

### 5. コンフィグ読解問題

**問題**: FTD の Connection Events ビューアにおいて、特定の接続ログの Action が `Allow` で記録されているにもかかわらず、Associated Intrusion Event の判定結果として `Would Have Dropped` と表示されていた。この通信が発生した FTD のインターフェイス設定状態として正しいものはどれか。

* A) 物理ポートがシャットダウンしている。
* B) インターフェイスが Inline Set で構成され、`Tap Mode` が有効化されている。
* C) インターフェイスが Routed モードで配置され、NAT 規則が適用されている。
* D) FTD が Clustering 構成で動作している。

**正解**: **B**
**解説**: **`Would Have Dropped`** は、Inline Tap モード（または Passive モード）において、侵入ルールが Drop アクションに合致したものの、検出専用モードのためパケット自体は破棄されずに通過を許可されたことを示す典型的なイベント表示です。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Threat Defense Configuration Guide - Inline Sets and Passive Interfaces](https://www.cisco.com/)
* [CCNP Security Cisco Secure Firewall Official Cert Guide (Chapters 6 & 7)](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot Inline Sets and Tap Mode on FTD](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Cisco Secure Firewall Deployment Modes (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * ラボ試験で「Deploy FTD in detection-only mode with the capability to quickly switch to prevention without recabling」と指示された場合は、`Passive` ではなく **`Inline Set with Tap Mode`** を選択するのが正解です。
* **図解（In-line vs Inline Tap vs Passive の決定ツリー）**:
  ```text
  [ Traffic Inspection Requirement ]
                 │
                 ├── Need Real-time Threat Blocking?
                 │         ├── YES ──► In-line Mode (Inline Set)
                 │         └── NO
                 │              │
                 │              ├── Will transition to Prevention soon?
                 │              │         ├── YES ──► Inline Tap Mode (Physical Inline + Tap Enabled)
                 │              │         └── NO  ──► Passive Mode (SPAN / RSPAN / ERSPAN)
  ```
* **注意点**:
  * 仮想環境（FTDv / NGIPSv）で Inline Set や Passive インターフェイスを構築する際は、ESXi の Port Group レベルで **Promiscuous Mode を Accept に変更** することを絶対に忘れないでください。これを忘れると FTD がパケットを受信・転送できず、疎通障害となります。
