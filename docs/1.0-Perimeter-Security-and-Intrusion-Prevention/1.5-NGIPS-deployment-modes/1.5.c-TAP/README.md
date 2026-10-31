---
layout: default
title: 1.5.c-TAP
nav_order: 3
parent: 1.5-NGIPS-deployment-modes
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.5.c TAP

Cisco Secure Firewall Threat Defense (FTD) および Firepower NGIPS アプライアンスにおける **TAP Mode（Inline Tap モード / インラインタップ配置）** は、物理的・論理的な配線はインラインモード（Inline Pair: 直列挿入）の構成を維持しつつ、内部処理としてはパケットのコピーを作成して Snort インスペクションエンジンへ回送する「インライン配線型・非ドロップ検出専用モード（Detection-Only Mode）」です。

CCIE Security v6.1 Blueprint **1.5.c TAP** では、Standard In-line Mode や Passive Mode との動作原理の違い、内部データプレーン（LINA / Snort Engine）におけるコピーパケット処理フロー、FMC (Firepower Management Center) イベントビューアにおける **`Would Have Dropped`** (インラインであれば遮断されていたはずのログ) の記録メカニズム、Inline Set 内での設定競合（Strict TCP Enforcement との共存不可等）、および検出専用運用からインライン防御運用への「無再配線切り替え（Zero-Recabling Transition）」の実装・検証能力が深く問われます。

---

## 📘 概要

* **機能概要**:
  * インラインペア（例: `GigabitEthernet0/0 <-> GigabitEthernet0/1`）として物理接続されたデータパスにおいて、FTD の内部 ASIC / ハードウェアまたはデータプレーン層で着信パケットを即座にバイパス転送（Egress 送出）すると同時に、パケットのコピー（Copy Packet）を生成して Snort 侵入検知エンジンへ渡すハイブリッド動作モード。
  * 侵入防御ポリシー (Intrusion Policy) やアクセス制御ポリシー (Access Control Policy) でルールのアクションが `Drop` や `Block` に設定されていても、本線のオリジナルパケットは一切破棄されずに通過し、FMC 上には「インラインモードであればドロップされていた」ことを示す `Would Have Dropped` アラートのみが記録される。
* **利用目的**:
  * 運用中の生産ネットワークや基幹通信経路に対して、物理配線を変更することなく安全に NGIPS シグネチャポリシーの誤検知（False Positive）リスクやパフォーマンス影響を評価する。
  * 将来的なインライン防御（In-line Prevention）への移行を前提とした、段階的セキュリティ評価フェーズの確立。
* **どのような場面で利用するか**:
  * 新規に FTD / NGIPS アプライアンスを社内境界やデータセンターに導入する際、最初の数週間〜数ヶ月間は通信切断トラブルを防ぎながらセキュリティ脅威を可視化したい場面。
  * 評価期間終了後、夜間メンテナンス等でケーブルの抜き差しを行うことなく、FMC GUI の設定変更（`Tap Mode` チェック解除）のみで即座にインラインリアルタイム遮断へ移行させたい環境。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **物理配線構造** | **インライン構成** (`Bump in the wire` / 1 つの Inline Pair に最低 2 ポート必要) |
| **内部処理メカニズム** | 本線パケットは即時転送。内部生成された**コピーパケットに対して Snort 検査を実施** |
| **トラフィック遮断能力** | **不可** (Block / Drop ルールにマッチしても本線通信はドロップされない) |
| **ログ表示 (Inline Result)** | **`Would Have Dropped`** (Reason: `Interface in Passive or Tap mode`) |
| **インライン移行性** | **極めて容易** (物理配線変更なしで FMC GUI の設定変更 ＆ Deploy のみで切替可能) |
| **設定場所** | `Devices > Device Management > Inline Sets > Edit > Advanced > Tap Mode` [✓] |
| **排他設定・制限事項** | **`Strict TCP Enforcement` との同時有効化は不可** |
| **パフォーマンス影響** | パケットのコピー生成処理および全コピーパケットの Snort 検査により、FTD CPU 負荷はインラインと同等以上に発生 |

---

## 🏗 動作原理

Inline Tap モードでは、Ingress インターフェイスに着信したパケットは遅延なしで対向の Egress インターフェイスへ送出され、同時に複製されたコピーパケットが DAQ レイヤーを経由して Snort エンジンへ引き渡されます。

```text
                                [ Ingress Live Packet ]
                                           │
                                           ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Firepower Threat Defense (FTD) - Inline Set in TAP Mode                          │
│                                                                                  │
│  Ingress Interface (Gi0/0)                                                       │
│         │                                                                        │
│         ├─── (1) Fast Forward / Direct Hardware Bridge ────────────────────────┐  │
│         │                                                                     │  │
│         │ (2) Internal Copy Packet Generation                                 │  │
│         ▼                                                                     │  │
│  ┌──────────────┐                                                             │  │
│  │ LINA Engine  │                                                             │  │
│  └──────┬───────┘                                                             │  │
│         │                                                                     │  │
│         ▼                                                                     │  │
│  ┌──────────────┐ (Full Inspection on Copy Packet)                            │  │
│  │ Snort Engine ├──────────────────────────┐                                  │  │
│  └──────┬───────┘                          │                                  │  │
│         │                                  │ Match Drop Rule                  │  │
│         │ Clean / Allow                    ▼                                  │  │
│         ▼                          ┌──────────────┐                           │  │
│  [ Ignore Copy ]                   │ Generate Log │                           │  │
│                                    │ "Would Have  │                           │  │
│                                    │  Dropped"    │                           │  │
│                                    └──────────────┘                           │  │
│                                                                               │  │
│  Egress Interface (Gi0/1) ◄───────────────────────────────────────────────────┘  │
└─────────┬────────────────────────────────────────────────────────────────────────┘
          │
          ▼
   [ Original Live Packet Delivered Undisturbed ]
```

---

## ⚙ 動作シーケンス

1. **パケットの着信とハードウェア分岐**:
   * パケットが Inline Pair の Ingress ポート（例: `Gi0/0`）に着信。
   * インラインタップ有効時、データプレーンはパケットを即座に Egress ポート（`Gi0/1`）の送信バッファへ回送すると同時に、メモリバッファ上に同一のパケットコピーを生成します。
2. **本線パケットの即時送出**:
   * オリジナルの本線パケットは、Snort インスペクションの完了を待つことなく、即座に対向ポートから送出されます。これにより、FTD による通信遅延（Latency）はほぼゼロとなります。
3. **複製パケットの Snort エンジン回送**:
   * 複製されたコピーパケットが DAQ (Data Acquisition) 層を通じて Snort 2 / Snort 3 インスペクションエンジンへ入力されます。
4. **シグネチャ評価と判定**:
   * Snort はプロトコルデコード、再構築、および Access Control Policy / Intrusion Policy の各種ルールと照合します。
   * パケットが `Drop and Generate Events` や `Block` アクションのシグネチャに合致した場合、Snort 内部でドロップ判定が下されます。
5. **FMC への `Would Have Dropped` イベント送出**:
   * オリジナルパケットはすでに通過済みであるため、物理的なドロップは行われません。
   * FMC 側には **Inline Result = `Would Have Dropped`**、**Reason = `Interface in Passive or Tap mode`** として侵入イベントが送信・記録されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Inline Set における `Tap Mode` の構成手順**:
   * インラインタップモードを設定するには、インターフェイス設定でポートを個別設定するのではなく、**Inline Set の Advanced タブ内にある `Tap Mode` チェックボックスを有効化** します。
   * **標準手順**:
     1. インターフェイス `Gi0/0` と `Gi0/1` の Mode を `None` に設定して Enabled にする。
     2. `Devices > Device Management > Inline Sets` で Inline Set を作成し、ペアをアサイン。
     3. `Advanced` タブを開き、`Tap Mode` にチェックを入れて保存 ➔ Deploy。
2. **`Strict TCP Enforcement` との競合排除**:
   * Inline Set の Advanced 設定において、**`Tap Mode` と `Strict TCP Enforcement`（厳格な TCP 状態チェック）を同時に有効化することは FMC 上で禁止** されています。
   * 両方をチェックして保存しようとするとバリデーションエラーとなるため、試験で Inline Tap が指定された場合は Strict TCP Enforcement を必ず無効化（オフ）にします。
3. **`Would Have Dropped` イベントの確認と解読**:
   * 試験問題で「Verify that the attack packet triggers an intrusion event but is not dropped by FTD」と指示された場合、FMC の `Analysis > Intrusions > Events` 画面で **Inline Result** カラムを確認します。
   * 表示が **`Dropped`** ではなく **`Would Have Dropped`** となっていることを検証します。
4. **Inline Tap から In-line Prevention への迅速な切り替え試験**:
   * 試験の指示で「Phase 1: Monitor traffic in tap mode without dropping. Phase 2: Switch to prevention mode to block traffic in real-time without recabling」が出題されます。
   * FMC GUI で対象 Inline Set の `Tap Mode` のチェックを外し、変更を Deploy するだけで完了することを理解しておく必要があります。
5. **Passive モードとの使い分け・選択問題 (Design / Implementation)**:
   * **永久的な検出専用運用 (Permanent Detection-Only)** ➔ **`Passive Mode`**（装置障害時にも本線に一切影響を与えないため推奨）。
   * **一時的な評価・将来のインライン遮断への移行前提 (Temporary Evaluation for Future Prevention)** ➔ **`Inline Tap Mode`**（物理配線の変更が不要であるため推奨）。

---

## 🛠 設定方法

### 1. FMC GUI: Inline Set での TAP Mode 有効化手順

```text
[ Step 1: 物理ポートの事前準備 ]
1. FMC UI で Devices > Device Management に移動し対象 FTD を選択。
2. Interfaces タブで GigabitEthernet0/0 および GigabitEthernet0/1 を編集:
   - Mode: None
   - Enabled: [✓] チェック
3. "Save" をクリック。

[ Step 2: Inline Set の作成と TAP Mode の設定 ]
1. "Inline Sets" タブを選択し "Add Inline Set" をクリック。
2. General タブ:
   - Name: TAP_INLINE_SET
   - Selected Interface Pair: "GigabitEthernet0/0 <-> GigabitEthernet0/1" を追加。
3. Advanced タブ:
   - [✓] Tap Mode (チェックをオンにする)
   - [ ] Strict TCP Enforcement (非チェックであることを確認)
   - [✓] Propagate Link State (必要に応じて有効化)
4. "OK" ➔ "Save" をクリック。
5. 警告ダイアログ（インターフェイスモード変更の警告）で "Yes" を選択し、対象 FTD へ Deploy。
```

---

### 2. FTD CLI: TAP Mode の動作検証コマンド

FTD CLI では `show inline-set` コマンドを実行し、`Tap mode is on` と表示されることを確認します。

```bash
> show inline-set
Inline-set TAP_INLINE_SET
Mtu is 1500 bytes
Fail-open for snort down is off
Fail-open for snort busy is off
Tap mode is on
Propagate-link-state option is on
hardware-bypass mode is disabled
Interface-Pair:
Interface: GigabitEthernet0/0 "INSIDE_IF"
Current-Status: UP
Interface: GigabitEthernet0/1 "OUTSIDE_IF"
Current-Status: UP
Bridge Group ID: 507
>
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI / FMC) | 表示確認ポイント |
| :--- | :--- | :--- |
| **Inline Set の TAP モード設定確認** | <code>show inline-set</code> | `Tap mode is on` と表示されること |
| **インラインペアポートの動作確認** | <code>show interface GigabitEthernet 0/0</code> | ポートの Up/Down 状態、ドロップカウントなしの推移 |
| **Snort によるリアルタイム評価の追跡** | <code>system support firewall-engine-debug</code> | 攻撃パケット検出時に `Intrusion Block` / `Drop` 判定が内部ログに出力されること |
| **FMC 上でのイベント判定結果の確認** | `Analysis > Intrusions > Events` | Inline Result カラムに **`Would Have Dropped`**、Reason に **`Interface in Passive or Tap mode`** と記録されること |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **Inline Set 設定時に「Tap Mode cannot be enabled with Strict TCP Enforcement」エラーが出る** | `Strict TCP Enforcement` と `Tap Mode` が同時にチェックされている | FMC Inline Set Advanced 設定画面 | Advanced タブで `Strict TCP Enforcement` のチェックを外してから `Tap Mode` を有効化する。 |
| **TAP モードで運用中、攻撃パケットを送信しても通信が遮断されずに相手に届いてしまう** | 仕様通りの動作。TAP モードはコピーパケットを検査するため、本線通信はドロップされない | <code>show inline-set</code><br>FMC Intrusion Events | 正常動作。実際にリアルタイム遮断したい場合は Inline Set の `Tap Mode` をオフにして Deploy する。 |
| **TAP モード設定後、FMC のイベント画面に `Dropped` と表示される** | ポートが TAP モードではなく標準の In-line モードとして動作している | <code>show inline-set</code> | `show inline-set` で `Tap mode is on` であるか確認し、オフの場合は FMC から再設定して Deploy。 |
| **TAP モードに設定したところ FTD の CPU 使用率が高高止まりする** | TAP モードは全パケットのコピーを生成して Snort 検査するため、CPU 負荷はインラインと同等以上になる | <code>show cpu</code> | 検査対象のルール（Intrusion Policy）を絞り込むか、不要なプロトコルプリプロセッサを無効化。 |

---

## ⚠ 制限事項

* **排他機能 (Strict TCP Enforcement)**:
  * `Tap Mode` が有効な Inline Set では、**`Strict TCP Enforcement` を同時に有効化することはできません**。
* **通信制御機能の非対応**:
  * TAP モードでは、本線パケットに対するドロップ（Drop）、修正（Normalize）、接続リセット（TCP Reset 送信）、レートリミット（QoS / Rate Shaping）は実行できません。
* **パフォーマンス上の影響**:
  * TAP モードはトラフィックのコピーを作成して処理するため、ネットワークのトラフィックパターンによってはインライン防御モードと同等以上の CPU / メモリリソースを消費します。

---

## 🔄 他技術との関連

* **Intrusion Policy ("Drop when Inline" / Inspection Mode)**:
  * 侵入防御ポリシー側の設定で `Inspection Mode` を `Detection`（旧バージョン表記: `Drop when Inline` のチェックオフ）に設定することでも、インラインポートのまま非ドロップ評価が可能です。ただし、`Tap Mode` は**インターフェイスレベルで一括して非ドロップコピー処理**を行う点が異なります。
* **Link State Propagation**:
  * Inline Set に `Tap Mode` を設定している場合でも `Propagate Link State` は併用可能です。

---

## 🧩 比較表

### 1. In-line Mode vs Passive Mode vs TAP (Inline Tap) Mode

| 比較項目 | In-line Mode | Passive Mode | TAP (Inline Tap) Mode |
| :--- | :--- | :--- | :--- |
| **物理配線** | インライン (直列本線) | 帯域外 (SPAN / TAP 由来) | **インライン (直列本線)** |
| **内部パケット処理** | 本線パケットを直接インスペクション | 受信したコピーをインスペクション | **本線は即時通過、コピーをインスペクション** |
| **リアルタイム遮断** | **可能** (Drop / Reset) | **不可** (アラートのみ) | **不可** (`Would Have Dropped`) |
| **本線遅延 (Latency)** | 微小なインスペクション遅延発生 | **遅延ゼロ** | **遅延ゼロ** |
| **インライン遮断への移行** | 最初から遮断動作 | **物理配線の再変更が必要** | **配線変更なし (GUI 設定変更のみ)** |
| **スイッチ側 SPAN 設定** | 不要 | **必須** | **不要** |

---

## 💡 ベストプラクティス

1. **段階的セキュリティ導入（POC / チューニングフェーズ）での活用**:
   * 本番環境に NGIPS を導入する際、初期配線をインライン（Inline Pair）で構成した上で **`Tap Mode` を有効化** して 2〜4 週間運用します。FMC 上で発生する `Would Have Dropped` イベントを分析して誤検知（False Positive）ルールを無効化/チューニングした後、メンテナンスウィンドウで `Tap Mode` を無効化して完全なインライン防御へ移行させます。
2. **恒久的な検出専用運用では Passive Mode を推奨**:
   * 将来的にもインライン遮断を行う予定がなく、永久に検出専用（IDS）として運用する場合は、装置故障時のリスクを完全にゼロ化できる **`Passive Mode`**（SPAN 接続）を選択することが Cisco の推奨設計です。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Inline Set における TAP Mode の基本構成

#### 問題
FTD の物理ポート `GigabitEthernet0/0` と `GigabitEthernet0/1` を使用して Inline Set `SET_TAP_EVAL` を作成し、検出専用モード (TAP Mode) として構成せよ。

#### 要件
* インターフェイス名: `Gi0/0` ➔ `INSIDE`、`Gi0/1` ➔ `OUTSIDE`
* FMC GUI を使用して設定すること。

#### 設定・操作手順

```text
1. Devices > Device Management で対象 FTD を編集。
2. Interfaces タブで Gi0/0 (INSIDE) および Gi0/1 (OUTSIDE) の Mode を None にして Enabled にチェック。
3. Inline Sets タブで "Add Inline Set" を作成:
   - Name: SET_TAP_EVAL
   - Selected Interface Pair: INSIDE <-> OUTSIDE を追加。
4. Advanced タブで [✓] Tap Mode にチェックを入れる。
5. OK ➔ Save をクリックして Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`Tap mode is on` であることを確認。

---

### 例2: TAP Mode 有効化時の Strict TCP Enforcement エラーの解消

#### 問題
Inline Set `SET_TAP_EVAL` にて `Tap Mode` を有効化しようとしたところ、FMC 上でエラーが発生して保存できない。原因を特定し修正せよ。

#### 要件
* `Strict TCP Enforcement` との競合を解消すること。

#### 設定・操作手順

```text
1. Devices > Device Management > Inline Sets を開く。
2. "SET_TAP_EVAL" の編集画面を開き "Advanced" タブを選択。
3. [ ] Strict TCP Enforcement のチェックを外す (オフにする)。
4. [✓] Tap Mode にチェックを入れる。
5. OK ➔ Save をクリックして Deploy。
```

#### 検証方法
* エラーなく設定が保存され、FTD へ Deploy が正常完了することを確認。

---

### 例3: FMC における `Would Have Dropped` イベントの検出検証

#### 問題
TAP モードで動作している FTD 配下のクライアントから、侵入ルール（SID: `2000001` / Action: `Drop and Generate Events`）に合致する攻撃パケットを送出し、FMC イベントビューアでの判定結果を検証せよ。

#### 要件
* 通信自体は遮断されずターゲットへ到達すること。
* FMC 上で Inline Result が `Would Have Dropped` となること。

#### 設定・操作手順

```text
1. 対象 FTD の Inline Set が Tap Mode であることを確認。
2. Access Control Policy 配下の Intrusion Policy で SID 2000001 を "Drop and Generate Events" に設定して Deploy。
3. クライアントから攻撃パケットを送出。
```

#### 検証方法
* クライアントの通信が正常完了することを確認。
* FMC の `Analysis > Intrusions > Events` 画面を開き、該当イベントの Inline Result カラムが **`Would Have Dropped`**、Reason が **`Interface in Passive or Tap mode`** であることを確認。

---

### 例4: TAP Mode から In-line Prevention モードへの切り替え

#### 問題
評価フェーズが終了した Inline Set `SET_TAP_EVAL` を、配線変更を行わずにリアルタイム遮断が可能なインライン防御モードへ移行させよ。

#### 要件
* FMC GUI から `Tap Mode` を無効化すること。

#### 設定・操作手順

```text
1. Devices > Device Management > Inline Sets タブを開く。
2. "SET_TAP_EVAL" の編集をクリックし "Advanced" タブを選択。
3. [ ] Tap Mode のチェックを外す (オフにする)。
4. OK ➔ Save をクリックし、FTD へ Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`Tap mode is off` に変更されていることを確認。再度攻撃パケットを送出し、通信がリアルタイムにドロップされ FMC に `Dropped` と表示されることを検証。

---

### 例5: Propagate Link State と TAP Mode の併用構成

#### 問題
TAP モードで動作する Inline Set `SET_TAP_EVAL` において、一方の物理リンク障害時に対向ポートも自動的に Down させる `Propagate Link State` を併用構成せよ。

#### 要件
* Advanced タブで設定すること。

#### 設定・操作手順

```text
1. Devices > Device Management > Inline Sets を開く。
2. "SET_TAP_EVAL" を編集 ➔ "Advanced" タブ。
3. [✓] Tap Mode および [✓] Propagate Link State の双方にチェックを入れる。
4. OK ➔ Save をクリックして Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`Tap mode is on` および `Propagate-link-state option is on` の両方が表示されることを確認。

---

### 例6: Snort リアルタイムデバッグによる TAP コピーパケットの評価追跡

#### 問題
FTD CLI のデバッグツールを使用して、TAP モード配下を通過する ICMP トラフィックのコピーが Snort エンジンで正常に評価されているか追跡せよ。

#### 要件
* `system support firewall-engine-debug` を使用すること。

#### 設定・操作手順

```bash
> system support firewall-engine-debug

Please specify an IP protocol: icmp
Please specify a client IP address: 10.1.1.50
Please specify a server IP address:
```

#### 検証方法
* クライアントから ICMP パケットを発生させた際、FTD CLI 上に Snort の評価ログが表示され、パケットのコピーに対してルール評価が行われていることを確認。

---

### 例7: FTD CLI を用いた TAP モード インターフェイスの個別の統計確認

#### 問題
FTD CLI コマンドを使用して、TAP モードで動作している `GigabitEthernet0/0` の詳細ステータスおよびパケットドロップ統計を確認せよ。

#### 要件
* `show interface` コマンドを使用すること。

#### 設定・操作手順

```bash
> show interface GigabitEthernet0/0
Interface GigabitEthernet0/0 "INSIDE", is up, line protocol is up
  Hardware is i82540EM Gigabit Ethernet, address is 0050.568a.0001
  MTU 1500 bytes, BW 1000000 Kbit/sec full duplex
  IPS Interface-Mode: Inline Pair (Tap Mode Enabled)
  0 packets input, 0 bytes
  0 input drops, 0 buffer drops
```

#### 検証方法
* 出力結果において `IPS Interface-Mode` または Inline Pair ステータスが正常に表示されることを確認。

---

### 例8: EtherChannel での TAP Mode インラインペア構成

#### 問題
FTD の `Port-Channel1` と `Port-Channel2` で構築された大容量インラインペアに対して TAP モードを有効化せよ。

#### 要件
* Inline Set 名: `BUNDLE_TAP_SET`

#### 設定・操作手順

```text
1. Port-Channel1 と Port-Channel2 を作成。
2. Inline Sets タブで "Add Inline Set" を作成:
   - Name: BUNDLE_TAP_SET
   - Selected Interface Pair: "Port-Channel1 <-> Port-Channel2"
3. Advanced タブで [✓] Tap Mode にチェックを入れる。
4. OK ➔ Save をクリックして Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、`Interface-Pair` が Port-Channel で構成され、`Tap mode is on` であることを確認。

---

### 例9: Multiple Inline Pairs を含む Inline Set での TAP Mode 適用

#### 問題
非同期ルーティングに対応するために構成された 2 つのインラインペア (`Gi0/0-Gi0/1` および `Gi0/2-Gi0/3`) を含む Inline Set `SET_ASYNC_TAP` 全体に対して TAP モードを一括適用せよ。

#### 要件
* 1 つの Inline Set 設定で両ペアを TAP モードにすること。

#### 設定・操作手順

```text
1. Inline Set "SET_ASYNC_TAP" を編集。
2. General タブで両方の Interface Pair が追加されていることを確認。
3. Advanced タブで [✓] Tap Mode を有効化。
4. Save して Deploy。
```

#### 検証方法
* FTD CLI で `show inline-set` を実行し、両方の Interface-Pair および Interface-Pair の配下で `Tap mode is on` が適用されていることを確認。

---

### 例10: cURL (REST API) を使用した Inline Set の TAP Mode 状態の読み出し

#### 問題
FMC REST API を使用して、Managed FTD 上の Inline Set の設定情報を取得し、TAP モードが有効になっているか JSON レスポンスで確認せよ。

#### 要件
* `GET /api/fmc_config/v1/domain/{domainUUID}/devices/devicerecords/{containerUUID}/inlinesets` エンドポイントを使用すること。

#### 設定・操作手順

```bash
# ===== Auth Token 取得後、Inline Set 情報を GET =====
curl -k -X GET "[suspicious link removed]" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json"
```

#### 検証方法
* JSON レスポンス内の `"tapMode": true` となっていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: 管理者が FMC 上で Inline Set の Advanced 設定を開き、`Tap Mode` を有効化しようとしたが、`OK` ボタンを押した際にエラーダイアログが表示されて設定を保存できなかった。画面上では別の高度なオプションが有効化されていた。原因として正しいものはどれか。

* A) `Propagate Link State` が有効化されていたため。
* B) `Strict TCP Enforcement` が有効化されていたため。
* C) `Snort Fail Open (Busy)` が有効化されていたため。
* D) FTD が Routed Firewall Mode で動作していたため。

**正解**: **B**  
**解説**: FMC の Inline Set Advanced 設定において、**`Tap Mode` と `Strict TCP Enforcement`（厳格な TCP 状態チェック）は相互に排他的な設定** であり、同時に有効化することはできません。エラーを解消するには `Strict TCP Enforcement` のチェックを外す必要があります。

---

### 2. トラブルシュート問題

**問題**: FTD のインラインペアポートに対して TAP モードを有効化し、ポリシーを Deploy した。その後、侵入ポリシーで `Drop and Generate Events` と定義された悪意あるトラフィックを送信したところ、ターゲット端末でパケットが正常に受信されてしまった。FMC の侵入イベントログを確認したところ、Inline Result に `Would Have Dropped` と表示されていた。この状況に対する説明として適切なものはどれか。

* A) FTD アプライアンスの故障であり、アプライアンスの再起動が必要である。
* B) TAP モードは物理インライン配線のままパケットのコピーを検査する検出専用モードであり、本線パケット自体はドロップされない正常な動作である。
* C) FMC ↔ FTD 間の sftunnel が切断されているためドロップ指示が伝わらなかった。
* D) FTD 上で NAT 規則が正しく設定されていないため。

**正解**: **B**  
**解説**: **TAP モード** は、オリジナルパケットをそのまま通過させ、複製されたコピーパケットに対して Snort 検査を行う「検出専用モード」です。そのため、ルールアクションが Drop であっても物理的な遮断は発生せず、FMC 上に **`Would Have Dropped`** と記録されるのが仕様通りの正しい動作です。

---

### 3. Design 問題

**問題**: セキュリティ運用チームから、「現在導入予定の FTD について、初期の 1 ヶ月間は既存通信に一切影響を与えずに脅威を評価したい。ただし評価完了後は、ネットワークのケーブリング作業や物理メンテナンスを行うことなく、即座にインラインリアルタイム遮断運用へ移行させたい」という設計要望を受けた。推奨される構築手法はどれか。

* A) 物理ポートを Passive モード（SPAN 接続）で構築する。
* B) 物理ポートを Inline Pair として配線し、Inline Set で `Tap Mode` を有効化して構築する。
* C) FTD を Dedicated Routed Mode で構築し、BVI インターフェイスをバインドする。
* D) FXOS 上で Hardware Bypass を Bypass-Force モードに設定する。

**正解**: **B**  
**解説**: 最初からインライン配線（Inline Pair）を行った上で **`Tap Mode`** を有効にしておくことで、評価期間中は非ドロップで可視化を行い、移行時は FMC GUI 上で `Tap Mode` をオフにして Deploy するだけで、**ケーブルを一切触ることなくリアルタイムインライン遮断へ即時移行** できます。

---

### 4. 実装問題

**問題**: FTD CLI にログインし、現在動作している Inline Set において TAP モードが正常にオンになっているかを 1 行のコマンドで検証するための正確な CLI コマンドはどれか。

* A) <code>show interface ip brief</code>
* B) <code>show inline-set</code>
* C) <code>show running-config interface</code>
* D) <code>show service-policy</code>

**正解**: **B**  
**解説**: FTD CLI で **`show inline-set`** を実行すると、設定されている Inline Set の詳細オプションが表示され、**`Tap mode is on`**（または `off`）であることを直接確認できます。

---

### 5. コンフィグ読解問題

**問題**: FMC の Intrusion Events 画面において、特定のイベントの表示が以下のようになっていた。このイベントが発生した背景として正しいものはどれか。

* **Inline Result**: `Would Have Dropped`
* **Reason**: `Interface in Passive or Tap mode`

* A) FTD がインラインモードで動作しており、パケットが正常に破棄された。
* B) FTD のインターフェイスが TAP モード（または Passive モード）で構成されているため、ルールは Drop アクションに合致したもののパケット自体は通過を許可された。
* C) FMC のライセンス期限が切れたためドロップが保留された。
* D) FTD 上で Snort エンジンがクラッシュした。

**正解**: **B**  
**解説**: FMC は、インターフェイスが **TAP モード** または **Passive モード** の際、Drop ルールに合致したイベントに対して **`Would Have Dropped`** (Reason: `Interface in Passive or Tap mode`) と表示します。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Threat Defense Configuration Guide - Inline Sets and Passive Interfaces](https://www.cisco.com/)
* [CCNP Security Cisco Secure Firewall and Intrusion Prevention System Official Cert Guide (Chapter 7)](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot Inline Sets and Tap Mode on FTD](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Cisco Secure Firewall Architecture and Deployment Modes (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * ラボ試験で「Deploy FTD inline to evaluate policies without dropping traffic, with a requirement for fast transition to prevention without recabling」と提示された場合の解は **`Inline Set` + `Tap Mode [✓]`** です。
* **図解（In-line Prevention への切り替えワークフロー）**:
  ```text
  [ Phase 1: Evaluation (Detection-Only) ]
    Physical Cabling: Inline Pair (In-band)
    FMC Setting: Inline Set > Advanced > Tap Mode = [✓] Enabled
    Result: Live Traffic Forwarded + Copy Inspected + Logs "Would Have Dropped"

                       │
                       ▼ (No Physical Cable Touch Required)
  [ Phase 2: Production (In-line Prevention) ]
    Physical Cabling: Inline Pair (Unchanged)
    FMC Setting: Inline Set > Advanced > Tap Mode = [ ] Disabled
    Action: Click "Deploy" in FMC
    Result: Live Traffic Inspected + Threats Dropped Real-Time + Logs "Dropped"
  ```
* **注意点**:
  * TAP モード変更の Deploy 実行時は、FTD 内部のデータパス処理モードが切り替わるため、非常に短い時間ですがパケットの瞬断や処理遅延が発生する可能性があります。ラボ試験および実務では、メンテナンスウィンドウ内で切り替え設定を行うことがベストプラクティスです。
