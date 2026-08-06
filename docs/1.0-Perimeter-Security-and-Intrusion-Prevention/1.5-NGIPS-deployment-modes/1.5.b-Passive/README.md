---
layout: default
title: 1.5.b-Passive
nav_order: 2
parent: 1.5-NGIPS-deployment-modes
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.5.b Passive

Cisco Next-Generation IPS (NGIPS) における**パッシブ（Passive）展開モード**は、ネットワークのトラフィックパスに直接介入せず、スイッチの SPAN (Switch Port Analyzer) やネットワーク TAP を介してパケットのコピーを受信し、検査を行うモードです。このモードは、既存のネットワーク構成を変更することなく、トラフィックの可視化と脅威検知を実現するために利用されます。

---

## 📘 概要

*   **機能概要**: トラフィックのコピーを受け取り、Snort インスペクションエンジンで分析します。パケットのドロップ（遮断）は物理的に不可能ですが、アラートの生成や、コネクションの中断（TCP Reset の送信）を試みることができます。
*   **利用目的**: ネットワークのパフォーマンス（遅延）に影響を与えたくない環境や、IPS 導入前のベースライン調査、およびトラブルシューティング時のトラフィック分析に利用されます。
*   **どのような場面で利用するか**: 
    *   コアスイッチの監視（SPAN 接続）。
    *   インライン導入が困難な、非常に高いスループットを要求されるネットワーク。
    *   検知のみが必要なセキュリティコンプライアンス要件。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な特徴** | トラフィックパスの外側に配置（Out-of-band）。遅延ゼロ。 |
| **用途** | 脅威の可視化、IDS（侵入検知）、監査。 |
| **メリット** | ネットワークの可用性に影響を与えない。デバイス故障時も通信が維持される。 |
| **デメリット** | パケットをリアルタイムで破棄できない。単一パケットの攻撃を防げない。 |
| **対応機種** | Firepower (FTD) センサー, ASA with Firepower Services。 |
| **応答機能** | アラート生成、TCP Reset (要：Response Interface構成)。 |
| **設計上の注意点** | SPANポートのオーバーサブスクリプションによるパケット取りこぼしに注意。 |

---

## 🏗 動作原理

パッシブモードでは、パケットはスイッチから複製（ミラーリング）されてセンサーに届きます。

```text
[ Client ] <---- Traffic Path ----> [ Server ]
                     |
               [ Switch (SPAN) ]
                     |
              (Copy of Packet)
                     ↓
[ Passive Interface (Eth1/1) on FTD ]
                     ↓
        [ Snort Inspection Engine ] ----> [ Generate Alert to FMC ]
                     ↓
        (Optional: Send TCP Reset via Response Interface)
```

---

## ⚙ 動作シーケンス

1.  **パケットコピーの受信**: スイッチが特定のポートや VLAN の通信を複製し、FTD のパッシブインターフェイスへ送信します。
2.  **前処理 (Pre-processing)**: 物理層で取り込まれたデータが、Snort エンジンによって正規化・デコードされます。
3.  **シグネチャ照合**: Intrusion Policy に定義されたルールセットに基づき、攻撃パターンとの比較が行われます。
4.  **イベント生成**: 脅威を検知すると、FMC へアラートログが送信されます。
5.  **リアクティブ制御（任意）**: ルールの設定が `Drop` になっていても、パッシブモードでは `Alert` として扱われますが、構成されていれば TCP Reset パケットを送信してセッション中断を試みます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Blueprintで重要なポイント**: 
    *   FTD インターフェイスを `Passive` モードに設定する正確な手順。
    *   パッシブインターフェイスを割り当てる `Passive Zone` の作成。
    *   パッシブ構成下での `Intrusion Event` の確認方法。
*   **ラボ試験で設定させられそうな内容**:
    *   **インターフェイス設定**: 物理ポートをパッシブモードにし、セキュリティゾーンを適用。
    *   **ACPルールの作成**: アクションを `Allow` または `Monitor` にし、Inspection タブで `Intrusion Policy` を適用。
    *   **SPAN構成**: スイッチ側で対象の FTD ポートを宛先（destination）として SPAN を組む（ルータ・スイッチの試験範囲との複合問題）。
*   **よくある設定ミス**:
    *   インターフェイスが `Shutdown` 状態のまま。
    *   セキュリティゾーンの種類が `Passive` ではなく `Routed` 等になっている。
    *   ACP ルールで `Intrusion Policy` が紐付けられていない。
*   **showコマンドから状態を判断**:
    *   `show interface` でパケットの受信数（rx）を確認し、SPAN 通信が実際に届いているか確認。
    *   `show snort statistics` でインスペクションエンジンの処理状況を確認。

---

## 🛠 設定方法

### 1. FTD インターフェイスのパッシブ化 (FMC GUI)
1.  **Devices > Device Management** で対象デバイスを編集。
2.  インターフェイス（例: Eth1/1）の **Mode** を `Passive` に変更します。
3.  `Enabled` にチェックを入れ、**Security Zone** を新規作成して割り当てます（Zone Type も `Passive` になります）。

### 2. アクセスコントロールポリシー (ACP) の適用
1.  **Policies > Access Control** で対象ポリシーを編集。
2.  **Add Rule** をクリック。
    *   **Source Zone**: 先ほど作成した Passive Zone。
    *   **Action**: `Allow`。
    *   **Inspection**: 適切な `Intrusion Policy` を選択。
3.  保存して **Deploy** を実行します。

---

## 🔍 検証コマンド

| 目的 | コマンド（FTD CLI） |
| :--- | :--- |
| **IFの状態と受信確認** | <code>show interface</code> |
| **インスペクション統計の表示** | <code>show snort statistics</code> |
| **パケットパスのデバッグ** | <code>system support firewall-engine-debug</code> |
| **適用されているポリシーの確認** | <code>show service-policy inspect snort</code> |

---

## 🚨 **トラブルシュート**

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| アラートが一切発生しない | SPAN設定がスイッチ側にない | <code>show interface</code> | IFの入力カウンタが増えているか確認。 |
| イベントにパケットが表示されない | Logging設定の不備 | FMC ACP Rule | ACPのLoggingタブで <code>Log at Beginning</code> 等を有効化。 |
| デプロイが失敗する | インターフェイスモード競合 | <code>show interface mode</code> | 既存のIP設定やVLAN設定を削除してからPassiveに変更。 |
| 大量のパケットドロップが発生 | CPU/リソース枯渇 | <code>show cpu</code> | 検査対象のプロトコルを絞り込む。 |

---

## ⚠ 制限事項

*   **インラインドロップ不可**: ルールアクションを `Drop` にしても、物理的にパス上にいないため、アラートのみが出力されます。
*   **ハーフクローズ接続**: TCP Reset を送る場合、攻撃パケットがサーバーに到達した後に Reset が届く「レースコンディション」が発生する可能性があります。
*   **暗号化**: SPAN された暗号化トラフィックは、SSL Policy で復号（Decrypt）しない限り、IPS シグネチャとのマッチングができません。

---

## 🔄 他技術との関連

*   **Access Control**: パッシブトラフィックを Snort に渡すためのトリガーとなります。
*   **Intrusion Policy**: 検知のロジックそのものです。
*   **Response Interface**: パッシブモードでありながら通信の中断を試みる際に、TCP Reset を射出する専用ポートとして構成されます。

---

## 🧩 比較表

### In-line vs Passive

| 比較項目 | In-line | Passive |
| :--- | :--- | :--- |
| **役割** | 侵入防止 (IPS) | 侵入検知 (IDS) |
| **パケット転送** | デバイスが転送を担当 | スイッチが転送を担当 |
| **可用性への影響** | 故障時に通信断の可能性あり | 影響なし |
| **リアルタイム遮断** | **可能** | **不可** |

---

## 💡 ベストプラクティス

1.  **専用セグメントの利用**: SPAN トラフィックが管理通信（SFTunnel）を圧迫しないよう、データインターフェイスを分離します。
2.  **Monitorアクションの使用**: 通信の通過を目的とする Routed 等と異なり、パッシブでは `Monitor` アクションを使用して、統計のみを抽出する設計も有効です。
3.  **MTUの調整**: SPAN 通信でタグが付与される場合、MTU サイズを増やしてフラグメンテーションを防ぎます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的なパッシブIF構成
*   **要件**: Eth1/5 をパッシブモードで有効化し、ゾーン `INSIDE-IDS` に所属させよ。
*   **設定**: FMC > Device > Interfaces > Eth1/5 (Mode: Passive) > Add Zone.

### 2. 検知アラートの有効化
*   **要件**: パッシブゾーンからの HTTP 攻撃を `Balanced Security and Connectivity` ポリシーで検知せよ。
*   **設定**: ACP Rule (Source: INSIDE-IDS) > Inspection > Intrusion Policy 指定.

### 3. TCP Reset の送信 (Response Interface)
*   **要件**: 攻撃を検知した際、パッシブポートから TCP Reset を送信せよ。
*   **設定**: Interface 設定で `Response Interface` を有効化し、IPS ルールで `Reset` を選択。

### 4. 複数パッシブポートの集約
*   **要件**: 2つの SPAN ソースから届くトラフィックを1つのポリシーで監視せよ。
*   **設定**: 2つの IF を同じ Passive Security Zone に追加。

### 5. SPANパケットの受信確認
*   **コマンド**: FTD CLI にて `show interface gigabitEthernet 1/1` を実行し `packets input` が増えることを確認。

### 6. Snort 3 での動作
*   **要件**: 次世代エンジン Snort 3 を使用してパッシブ検知を行え。
*   **設定**: Device 設定で Snort 3 を有効化し、Passive IF を構成。

### 7. 特徴的なシグネチャのテスト
*   **課題**: ICMP 攻撃をパッシブで検知し、FMC の Intrusion Events 画面で確認せよ。

### 8. VACL (VLAN ACL) との連携
*   **構成**: スイッチ側で VACL を使用して特定のトラフィックのみを FTD のパッシブポートにリダイレクト。

### 9. 非IPトラフィックの除外
*   **要件**: ARP 等の非 IP パケットを検査対象から除外して CPU 負荷を下げよ。
*   **設定**: Prefilter Policy にて Fast-path ルールを作成。

### 10. パッシブモードでのパケットキャプチャ
*   **コマンド**: `capture [NAME] interface [Passive_IF] match ip any any` で実際に SPAN された中身を確認。

---

## ❓ 想定試験問題

1.  **実装**: ネットワークの遅延を一切許容できない財務部門の通信を NGIPS で監視したい。どのモードを選択すべきか？
    *   **回答**: Passive Mode.
2.  **トラブルシュート**: パッシブモードを設定したが FMC にイベントが来ない。`show interface` でパケットの着信は確認できている。ACP で確認すべき設定は？
    *   **回答**: ACP ルールのソースゾーンが正しくパッシブインターフェイスに紐付いているか、および Intrusion Policy が適用されているか.
3.  **コンフィグ読解**: FMC のインターフェイス一覧で `Passive` と表示されているが、ACP のアクションが `Block` になっている。この時の実際の挙動を述べよ。
    *   **回答**: 通信は遮断されず、アラート（Alert）のみが生成される.
4.  **Design**: パッシブモードで TCP 通信を積極的に停止させたい場合に、追加で構成すべき機能は？
    *   **回答**: TCP Reset (Response Interface).
5.  **Design**: SPAN ポートから届くトラフィックがインターフェイス帯域を超えている。どのような悪影響があるか？
    *   **回答**: パケットロスが発生し、IPS の検知精度が低下（偽陰性の増加）する。

---

## 🔗 参考リソース

*   **Configuration Guides**:
    *   [Cisco Secure Firewall Management Center Administration Guide, 7.1 - Passive Interfaces](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/interfaces-settings-passive.html)
    *   [Cisco Secure Firewall Threat Defense Configuration Guide for FMC, 7.0 - Deployment Modes](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/deployment_modes.html)
*   **Technical Notes**:
    *   [Understanding Firepower Deployment Modes (Cisco Support)](https://www.cisco.com/c/en/us/support/docs/security/firepower-management-center/212356-understand-firepower-deployment-modes.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: パッシブモードは「見るだけ」の機能ですが、CCIE ラボではスイッチ側の `monitor session` 設定とセットで問われることが多いです。FTD 側の受信カウンタが増えていない場合は、まずスイッチ側の設定を疑いましょう。
*   **図解**: パケットがセンサーを「通り抜けない」という物理的なトポロジを頭に叩き込んでください。
*   **注意点**: パッシブインターフェイスには IP アドレスを設定することはできません。L3 設定をしようとしてエラーが出る場合は、インターフェイスモードが正しく変更されているか再確認してください。
