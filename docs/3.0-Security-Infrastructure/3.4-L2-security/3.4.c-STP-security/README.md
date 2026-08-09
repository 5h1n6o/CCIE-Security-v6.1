---
layout: default
title: 3.4.c-STP-security
nav_order: 3
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.4.c STP security

Ciscoスイッチにおける**STP（Spanning Tree Protocol）セキュリティ**は、レイヤ2トポロジの安定性を維持し、攻撃者によるルートブリッジの乗っ取りや、意図しないデバイス接続による通信ループを防止するための必須技術です。CCIE Security v6.1において、この項目はインフラストラクチャの堅牢化（Hardening）の重要な要素として位置付けられています。

---

## 📘 概要

*   **機能概要**: BPDU Guard、Root Guard、Loop Guard、BPDU Filterなどの機能を使用して、STPのメッセージ（BPDU）交換を制御し、不正なトポロジ変更を阻止します。
*   **利用目的**: 管理外のスイッチ接続によるルートブリッジの奪取（中間者攻撃の足掛かり）の防止、および物理的なループ発生時の迅速な遮断。
*   **利用場面**: 
    *   エンドユーザが接続されるアクセスポート（BPDU Guard）。
    *   自組織の管理下にあるコアスイッチから、他部署や他社のスイッチが接続される境界ポート（Root Guard）。
    *   単方向リンク障害によるループが懸念されるファイバ接続（Loop Guard / UDLD連携）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要機能** | BPDU Guard, Root Guard, Loop Guard, BPDU Filter, PortFast。 |
| **保護対象** | L2トポロジの整合性、ルートブリッジの役割。 |
| **メリット** | 不正接続によるネットワークダウンの防止、STP収束の高速化。 |
| **デメリット** | 設定ミスにより、正当な冗長経路まで遮断（Err-disable）されるリスク。 |
| **対応機種** | Catalystスイッチ全般。 |
| **設計上の注意点** | エッジポートにはBPDU Guard、上位へのリンクにはRoot Guardを基本とする。 |

---

## 🏗 動作原理

STPセキュリティは、ポートが受信するBPDUの内容や、受信そのものの可否に基づいてアクションを決定します。

```text
[ Root Bridge ]
       ↓ (Superior BPDU)
[ Distribution Switch ]
       ↓ (Root Guard: "I won't let you be Root")
[ External Switch / Attacker ]
```

*   **BPDU Guard**: PortFastが有効なポートでBPDUを受信すると、即座にポートを `err-disable` にします。
*   **Root Guard**: そのポートが「Root Port」になることを禁止します。対向から優れた（Superior）BPDUを受信すると、ポートを `root-inconsistent` 状態にしてトラフィックを遮断します。
*   **Loop Guard**: ブロッキングポートがBPDUを受信しなくなった際、勝手に転送状態（Forwarding）へ移行してループを作るのを防ぎ、`loop-inconsistent` 状態にします。

---

## ⚙ 動作シーケンス

1.  **ポートの役割定義**: ポートをエッジ（Access）または非エッジ（Trunk/Uplink）として構成。
2.  **BPDU監視**: インターフェイスがBPDUを受信。
3.  **ポリシー照合**:
    *   **BPDU Guard有効時**: 受信した瞬間に `shutdown` (err-disable)。
    *   **Root Guard有効時**: 送信元ブリッジIDを比較。自身より優位な場合、遮断。
4.  **リカバリ**: 設定された `errdisable recovery` または手動の `shutdown/no shutdown` により復旧。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **BPDU Guardのグローバル設定 vs インターフェイス設定**: 
    *   グローバル (`spanning-tree portfast bpduguard default`) は **PortFastが有効なポートにのみ** 効きます。
    *   インターフェイス (`spanning-tree bpduguard enable`) はPortFastの設定に関わらず効きます。この差を突く問題に注意。
*   **Root Guardの適用場所**: 通常、下位スイッチへ向かう指定ポート（Designated Port）に設定します。ルートブリッジ自体に設定することはありません。
*   **BPDU Filterの危険性**: BPDUの送受信を完全に止めるため、ループが発生しても検知できなくなります。ラボ要件で「BPDUを送るな、且つ受信しても無視せよ」という指示がない限り、使用は控えるべきです。
*   **Err-disable Recovery**: STPセキュリティ違反でポートが落ちた際、自動復旧させる設定 (`errdisable recovery cause bpduguard`) はセットで問われることが多いです。
*   **PortFastとの依存関係**: BPDU Guardの動作はPortFast（エッジポート設定）に依存することが多いため、基礎設定の確認を怠らないでください。

---

## 🛠 設定方法

### 1. BPDU Guard (アクセスポート保護)
```bash
! グローバル設定（推奨）
spanning-tree portfast edge default
spanning-tree portfast bpduguard default

! 特定インターフェイスでの個別設定
interface GigabitEthernet1/0/1
 spanning-tree bpduguard enable
```

### 2. Root Guard (ルートブリッジ保護)
```bash
interface GigabitEthernet1/0/24
 description UPLINK-TO-ACCESS-SWITCH
 spanning-tree guard root
```

### 3. Loop Guard (単方向リンク障害対策)
```bash
! グローバル設定
spanning-tree loopguard default

! 特定インターフェイス
interface GigabitEthernet1/0/10
 spanning-tree guard loop
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **全体の状態確認** | <code>show spanning-tree summary</code> |
| **特定ポートの状態表示** | <code>show spanning-tree interface [ID] detail</code> |
| **遮断されているポートの確認** | <code>show spanning-tree inconsistentports</code> |
| **Err-disable状態の確認** | <code>show interfaces status err-disabled</code> |
| **リカバリタイマーの確認** | <code>show errdisable recovery</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| ポートが `err-disable` になる | ハブの接続や他スイッチからのBPDU受信 | <code>show interfaces status</code> | BPDU Guardの対象を確認、不要なデバイスを除去。 |
| 通信が断続的に切れる | Root GuardがSuperior BPDUを検知 | <code>show spanning-tree inconsistent</code> | 対向スイッチのブリッジプライオリティを修正。 |
| STPループが発生している | BPDU Filterの誤用 | <code>show run \| include filter</code> | <code>no spanning-tree bpdufilter</code> を適用。 |
| ポートが自動で復旧しない | リカバリ設定の欠落 | <code>show errdisable recovery</code> | <code>errdisable recovery cause bpduguard</code> を追加。 |

---

## ⚠ 制限事項

*   **EtherChannel**: Loop GuardをEtherChannelの物理メンバポートに設定すると、チャネル全体のアグリゲーションに影響を与える場合があります。
*   **MSTとの共存**: Loop GuardはMSTインスタンス単位ではなく、ポート単位またはVLAN単位で動作します。
*   **BPDU Filterの優先度**: インターフェイスでBPDU Filterを有効にすると、BPDU Guardの設定よりも優先され、ガードが機能しなくなります。

---

## 🔄 他技術との関連

*   **Port Security (3.4.d)**: MACアドレス制限。BPDU Guardと併用してポートを完全に要塞化します。
*   **VACL (3.4.g)**: 特定のBPDU（マルチキャストMAC 0180.c200.0000）をフィルタリングすることも可能ですが、STPガード機能の使用が推奨されます。
*   **UDLD**: Loop Guardと同様に単方向リンクを検知しますが、Loop GuardはSTPの論理レベル、UDLDはL1/L2の物理・プロトコルレベルで動作します。

---

## 🧩 比較表

### BPDU Guard vs Root Guard

| 特徴 | BPDU Guard | Root Guard |
| :--- | :--- | :--- |
| **主な目的** | アクセスポートへのスイッチ接続禁止 | ルートブリッジの役割固定 |
| **アクション** | ポートを `Shutdown` (err-disable) | ポートを `Discarding` (inconsistent) |
| **対象BPDU** | あらゆるBPDU | Superior（自身より優れた）BPDUのみ |
| **推奨適用場所** | アクセス層のエッジポート | 配布/コア層の下位スイッチ向けポート |

---

## 💡 ベストプラクティス

1.  **デフォルトのPortFast/BPDU Guard**: アクセススイッチではグローバルでこれらを有効にし、トランクポートでのみ個別に無効化します。
2.  **Root Guardの戦略的配置**: ネットワークの「北側（コア）」へ向かうポートを除き、すべての指定ポート（Designated Port）にRoot Guardを設定します。
3.  **UDLDとの併用**: 光ファイバ環境では、Loop GuardだけでなくUDLDを `aggressive` モードで併用することを推奨します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 全アクセスポートの自動保護
*   **要件**: PortFastが有効な全ポートで、BPDUを受信したら自動的に遮断せよ。
*   **設定**: `spanning-tree portfast edge default` + `spanning-tree portfast bpduguard default`。

### 2. 特定ポートのBPDU拒否（PortFastなし）
*   **要件**: PortFast設定に関わらず、Gi1/0/1でBPDUを検知したらポートを閉じよ。
*   **設定**: `interface Gi1/0/1` > `spanning-tree bpduguard enable`。

### 3. ルートブリッジの保護（Root Guard）
*   **要件**: Gi1/0/24の先に接続されたデバイスがルートブリッジにならないようにせよ。
*   **設定**: `interface Gi1/0/24` > `spanning-tree guard root`。

### 4. 単方向リンクによるループ防止
*   **要件**: Gi1/0/10でBPDUが途絶えた際、フォワーディングへの移行を阻止せよ。

### 5. 自動復旧タイマーの設定
*   **要件**: BPDU Guard違反で落ちたポートを60秒後に自動で復旧させよ。
*   **設定**: `errdisable recovery cause bpduguard`, `errdisable recovery interval 60`。

### 6. BPDU Filterの特定の挙動
*   **要件**: Gi1/0/5からBPDUを送信せず、受信しても無視せよ（注意が必要な設定）。

### 7. PVST+環境でのRoot Guard検証
*   **課題**: VLANごとにRoot Guardが動作していることを `show spanning-tree inconsistentports` で確認せよ。

### 8. MSTトポロジのガード
*   **要件**: MSTインスタンス0においてルートブリッジが移動しないよう設定せよ。

### 9. ログ出力の強化
*   **要件**: ガード機能が働いた際にSyslogを生成せよ（デフォルトで生成されるが、ロギングレベルを確認）。

### 10. インターフェイス詳細の確認
*   **課題**: `show spanning-tree interface [ID] detail` を実行し、"BPDU guard is enabled" の表示を探せ。

---

## ❓ 想定試験問題

1.  **実装**: ルータが接続されているポートでPortFastを有効にしつつ、誤ってスイッチが繋がれた場合にネットワークを守るコマンドは？
    *   **回答**: `spanning-tree bpduguard enable`。
2.  **トラブルシュート**: `show spanning-tree inconsistentports` にインターフェイスが表示されている。このポートの状態と原因は？
    *   **回答**: `root-inconsistent` 状態。原因は Root Guard が設定されたポートで対向から Superior BPDU を受信したため。
3.  **Design**: STPセキュリティにおいて、BPDU Guard と BPDU Filter の決定的な違いは？
    *   **回答**: BPDU Guard は受信時にポートを遮断するが、BPDU Filter は受信しても無視してポートを維持するため、ループのリスクが残る。
4.  **コンフィグ読解**: `spanning-tree portfast bpduguard default` が設定されているが、トランクポートにハブを繋いでも `err-disable` にならない。なぜか？
    *   **回答**: グローバル設定は PortFast (Edge) ポートにのみ適用されるため。トランクポートには個別の `spanning-tree bpduguard enable` が必要。
5.  **Design**: ネットワーク内の特定のスイッチをルートブリッジとして完全に固定するための最適な組み合わせは？
    *   **回答**: ブリッジプライオリティを最小（0）にし、下位へ向かう全ポートに **Root Guard** を適用する。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [Configuring STP Extensions](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3750/software/release/12-2_55_se/configuration/guide/scg3750/swstpopt.html)
*   **Cisco Live (BRKSEC-2202)**
    *   [Securing the Layer 2 Infrastructure](https://www.ciscolive.com/)
*   **Technical Notes**
    *   [Spanning Tree PortFast BPDU Guard Enhancement](https://www.cisco.com/c/en/us/support/docs/lan-switching/spanning-tree-protocol/10586-107.html)

---

## 📝 **補足（Notes）**
*   **学習メモ**: 「BPDU Guard = 門前払い」「Root Guard = 権力維持」と覚えると役割を混同しません。
*   **注意点**: ラボ試験で `spanning-tree portfast` (古い構文) と `spanning-tree portfast edge` (新しい構文) が混在する場合がありますが、機能は同じです。
*   **図解**: 自身のスイッチより小さいブリッジIDを持つパケットが Root Guard ポートに入った瞬間、色がオレンジ（遮断）に変わる様子をイメージしてください。
