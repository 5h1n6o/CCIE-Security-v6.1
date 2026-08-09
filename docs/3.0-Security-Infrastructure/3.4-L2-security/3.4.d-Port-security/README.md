---
layout: default
title: 3.4.d-Port-security
nav_order: 3
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.4.d Port security

**Port Security** は、Cisco スイッチのインターフェイスに接続できるデバイスを MAC アドレスに基づいて制限する、最も基本的かつ強力なレイヤ 2 セキュリティ機能の一つです。CCIE Security v6.1 の Blueprint では、「3.4 Layer 2 security techniques」の一部として、インフラストラクチャの要塞化（Hardening）において必須の知識とされています。

---

## 📘 概要

*   **機能概要**: スイッチポートに接続を許可する MAC アドレスを指定、または最大数を制限し、未許可のデバイスが接続された場合にポートの遮断（Shutdown）やトラフィックのドロップといったアクションを実行します。
*   **利用目的**: **MAC Flooding 攻撃**（CAM テーブルを偽装 MAC で溢れさせる攻撃）の防止、未許可デバイスの接続拒否、および特定のポートに対するネットワークアクセスの制御。
*   **どのような場面で利用するか**: 
    *   オフィス内の壁面情報コンセント（アクセスポート）。
    *   共有スペースのネットワークポート。
    *   特定のサーバーのみを接続させる専用ポート。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **対象ポート** | Access ポート、Trunk ポート（Voice VLAN 含む）。 |
| **識別単位** | 送信元 MAC アドレス。 |
| **学習方式** | Static（手動）, Dynamic（動的）, Sticky（動的学習後の永続保存）。 |
| **違反モード** | `Shutdown` (デフォルト), `Restrict`, `Protect`。 |
| **メリット** | MAC アドレスレベルでの物理的な接続制限を容易に実装可能。 |
| **デメリット** | MAC スプーフィング（偽装）には弱く、管理オーバーヘッドが高い。 |
| **設計上の注意点** | 無線 AP や IP 電話が接続されるポートでは、許可 MAC 数を慎重に設計する必要がある。 |

---

## 🏗 動作原理

スイッチは、受信したパケットの送信元 MAC アドレスを、設定された「セキュア MAC アドレス」と比較します。

```text
Host (MAC: A)
   ↓
[ Switch Port (Max MAC: 1, Secure MAC: B) ]
   ↓
[ Verification ] --- MAC A is NOT MAC B? --- YES (Violation!)
   ↓
[ Action Based on Policy ]
   ↓
- Shutdown: Port goes to Err-disabled state.
- Restrict: Drop traffic + Log + SNMP Trap + Increment counter.
- Protect: Drop traffic only.
```

---

## ⚙ 動作シーケンス

1.  **フレーム受信**: スイッチポートでイーサネットフレームを受信。
2.  **MAC アドレスチェック**: フレームの送信元 MAC アドレスが、そのポートのセキュア MAC アドレスリストに含まれているか、または新規学習可能（最大数未満）かを確認。
3.  **アドレス学習**: 新規 MAC の場合、設定に応じて動的、または `Sticky` としてデータベースに登録。
4.  **違反判定**: 最大数を超えた MAC や、他ポートで固定されている MAC を検知した場合、「セキュリティ違反」と判定。
5.  **アクション実行**: `Shutdown` ならインターフェイスを論理的に落とし、`Restrict/Protect` なら該当トラフィックのみを破棄。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Sticky MAC の要件**: 「最初に接続されたデバイスを再起動後も固定せよ」という要件では、`switchport port-security mac-address sticky` を使用します。これにより、学習した MAC が **Running-config** に書き込まれます。
*   **Voice VLAN との組み合わせ**: IP 電話と PC が接続されるポートでは、最大 MAC 数を最低でも `2`（電話＋PC）に設定する必要があります。試験ではこの数値ミスを狙われます。
*   **Err-disable Recovery**: ラボ試験では、セキュリティ違反で落ちたポートを自動復旧させる設定 (`errdisable recovery cause psecure-violation`) をセットで求められることが非常に多いです。
*   **違反モードの指定**: 「トラフィックは止めるがポートは落とすな、かつ管理者に通知せよ」という要件なら、`Restrict` モードを選択します。
*   **Trunk ポートでの設定**: 基本的にアクセスポートでの使用が推奨されますが、ラボではトランクポートに対して設定を求められる「ひっかけ」が出る場合があります。
*   **検証の重要性**: `show port-security interface` コマンドで、現在の MAC 学習数（CurrentAddr）と違反回数（SecurityViolation）を確認するスキルは必須です。

---

## 🛠 設定方法

### 1. 基本設定 (Access ポート)
```bash
interface GigabitEthernet1/0/1
 switchport mode access
 switchport port-security
 ! 最大接続数を1に制限（デフォルト）
 switchport port-security maximum 1
 ! 違反時はポートをシャットダウン（デフォルト）
 switchport port-security violation shutdown
```

### 2. Sticky MAC の有効化
```bash
interface GigabitEthernet1/0/2
 switchport mode access
 switchport port-security
 switchport port-security mac-address sticky
 ! 最初に通信したデバイスのMACを保存
```

### 3. エラーリカバリ設定
```bash
errdisable recovery cause psecure-violation
errdisable recovery interval 300
! 違反から5分後に自動的にno shutを試みる
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **全体の状態表示** | <code>show port-security</code> |
| **特定インターフェイスの詳細確認** | <code>show port-security interface [ID]</code> |
| **学習済みセキュア MAC の一覧** | <code>show port-security address</code> |
| **Err-disable 状態のポート確認** | <code>show interfaces status err-disabled</code> |
| **カウンタの確認** | <code>show port-security interface [ID] \| include Violation</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| ポートが `err-disabled` になる | 許可数を超えるデバイスが接続された | <code>show port-security interface</code> | 不要なデバイスを外し、<code>shut/no shut</code>。 |
| デバイスを繋ぎ変えても通信できない | Sticky MAC がコンフィグに残っている | <code>show run interface</code> | <code>no switchport port-security mac-address sticky [MAC]</code>。 |
| IP電話を繋ぐとポートが落ちる | <code>maximum</code> 数が1になっている | <code>show port-security interface</code> | <code>maximum 2</code> (またはそれ以上) に増やす。 |
| 違反しているのにログが出ない | <code>Protect</code> モードが設定されている | <code>show port-security interface</code> | <code>Restrict</code> または <code>Shutdown</code> に変更。 |

---

## ⚠ 制限事項

*   **動的学習の消失**: `Sticky` を使わない動的学習の場合、スイッチを再起動すると学習した MAC 情報は消えます。
*   **EtherChannel**: EtherChannel のメンバーポートには Port Security を設定できません。
*   **SPAN 宛先**: SPAN (Mirroring) の宛先ポートではサポートされません。
*   **MAC スプーフィング**: 攻撃者が許可されている MAC アドレスを偽装した場合、Port Security だけでは防げません（802.1X との併用を推奨）。

---

## 🔄 他技術との関連

*   **3.4.e DHCP Snooping**: DHCP Snooping と併用することで、IP と MAC の整合性を高めることができます。
*   **3.4.a DAI**: Port Security で L2 物理接続を制限し、DAI で ARP プロトコルを保護する多層防御が一般的です。
*   **2.1 AAA / 802.1X**: Port Security よりも柔軟で強力な「ID ベース」のアクセス制御を提供します。併用時の挙動（Multi-auth モードなど）に注意が必要です。

---

## 🧩 比較表

### Port Security 違反アクションの比較

| 特徴 | Shutdown | Restrict | Protect |
| :--- | :--- | :--- | :--- |
| **パケット破棄** | Yes | Yes | Yes |
| **ポートの停止** | **Yes (err-disable)** | No | No |
| **Syslog / SNMP** | Yes | **Yes** | **No** |
| **違反カウンタ** | Yes | Yes | No |
| **推奨用途** | 最も厳格な制限 | 運用継続しつつ監視 | 簡易的な制限 |

---

## 💡 ベストプラクティス

1.  **Sticky MAC の活用**: 運用環境では、再起動による通信断を防ぐために `Sticky` 設定と `write memory` の自動化を検討します。
2.  **Voice VLAN 対応**: デスクトップポートでは必ず `maximum 3` 程度に余裕を持たせます（電話、PC、および仮想環境用）。
3.  **Err-disable Recovery の有効化**: 物理的な現地対応を減らすため、自動復旧タイマーを設定します。
4.  **未使用ポートの Shutdown**: Port Security を設定するだけでなく、使っていないポートは `shutdown` するのが基本です。

---

## 📝 ラボ学習・設定サンプル例

### 1. 1台限定の厳格な制限
*   **問題**: Gi1/0/1 に接続できる MAC アドレスを 1 台に制限し、未許可デバイス接続時は即座にポートを閉じよ。
*   **設定**: `switchport port-security`（デフォルトで 1/Shutdown）。

### 2. Sticky MAC による永続化
*   **要件**: ポート Gi1/0/2 で最初に検知した MAC アドレスを保存し、設定を保存できるようにせよ。
*   **設定**: `switchport port-security mac-address sticky`。

### 3. IP 電話と PC の収容
*   **要件**: Voice VLAN 100 と Data VLAN 10 が設定されたポートで、MAC アドレスを最大 3 つまで許可せよ。
*   **設定**: `switchport port-security maximum 3`。

### 4. ポートを落とさない監視モード
*   **要件**: 違反検知時にパケットは捨てるが、ポートは維持し、管理者に SNMP トラップを送信せよ。
*   **設定**: `switchport port-security violation restrict`。

### 5. 高速復旧の自動化
*   **要件**: セキュリティ違反で落ちたポートを 30 秒後に自動復旧させよ。
*   **設定**: `errdisable recovery cause psecure-violation`, `errdisable recovery interval 30`。

### 6. MAC エイジングの設定
*   **要件**: 動的に学習したセキュア MAC を、無通信状態が 10 分続いたら削除せよ。
*   **設定**: `switchport port-security aging time 10`, `switchport port-security aging type inactivity`。

### 7. 静的 MAC の事前登録
*   **要件**: サーバーの MAC `0011.2233.4455` のみを Gi1/0/10 で許可せよ。
*   **設定**: `switchport port-security mac-address 0011.2233.4455`。

### 8. 最大 MAC 数の上限変更
*   **要件**: 共通ポート Gi1/0/20 で最大 10 台まで同時接続を許可せよ。

### 9. トランクポートでの Port Security
*   **要件**: Trunk ポートで VLAN ごとに MAC 制限をかけよ。

### 10. Sticky MAC の削除
*   **課題**: デバイス交換のため、Sticky で学習された古い MAC を設定から削除せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `switchport port-security violation protect` が設定されているポートで、最大数を超える MAC を検知した。何が起きるか？
    *   **回答**: 未許可のトラフィックはドロップされるが、ログは生成されず、違反カウンタも増えない。
2.  **Design**: IP 電話を使用する環境で Port Security を導入する際、注意すべき最大 MAC 数（Maximum）の設定値は？
    *   **回答**: 最低 2 以上。電話自体の MAC と、電話の PC ポートに繋がる端末の MAC を含める必要があるため。
3.  **トラブルシュート**: ポートが `err-disabled` になり、手動で `no shut` してもすぐに再度落ちてしまう。考えられる原因は？
    *   **回答**: 未許可デバイスが接続されたままである、または複数の MAC を送出するハブ等が接続されている。
4.  **実装**: 学習した MAC アドレスを再起動後も保持するための、`Dynamic` 以外の学習方式は？
    *   **回答**: **Sticky MAC**。
5.  **コンフィグ読解**: `switchport port-security aging type absolute` とはどのような動作か？
    *   **回答**: 通信の有無に関わらず、指定された時間が経過したら一律にセキュア MAC エントリを削除する。

---

## 🔗 参考リソース

*   **Cisco Configuration Guide**
    *   [Configuring Port Security](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3750/software/release/12-2_55_se/configuration/guide/scg3750/swportsec.html)
*   **Technical Notes**
    *   [Cisco Guide to Harden Cisco IOS Devices: Port Security](https://www.cisco.com/c/en/us/support/docs/ip/access-lists/13608-21.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [Layer 2 Security Deep Dive](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「Port Security は物理インターフェイスに鍵をかける技術」とイメージしてください。
*   **図解**: 
    1. MAC リスト確認
    2. 上限数確認
    3. アクション決定 (Shutdown/Drop)
*   **注意点**: ラボ試験では、 Port Security を有効にした後、正しく `maximum` が設定されていないために、自分自身の管理端末（テスト用 PC）が遮断されてしまうミスが多いです。設定変更は慎重に行ってください。
