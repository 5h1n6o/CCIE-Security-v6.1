---
layout: default
title: 3.4.g-VACL
nav_order: 7
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.4.g VACL (VLAN Access Control List)

**VACL (VLAN Access Control List)** は、VLAN マップとも呼ばれ、特定の VLAN 内を流れるすべてのトラフィック（同一サブネット内のホスト間通信を含む）を制御するための強力なレイヤ 2 セキュリティ技術です。通常のルータ ACL (RACL) が L3 インターフェイス（SVI やルーテッドポート）を通過するトラフィックのみを対象とするのに対し、VACL は VLAN 内のスイッチングトラフィックに対してもフィルタリングを適用できる点が最大の特徴です。

---

## 📘 概要

*   **機能概要**: 物理ポートの種別（アクセス/トランク）に関わらず、特定の VLAN 全体に対して適用されるアクセス制御リストです。IP トラフィックだけでなく、非 IP トラフィック（MAC アドレスベース）の制御も可能です。
*   **利用目的**: 同一 VLAN 内における特定のホスト間通信の遮断、スニッフィング（盗聴）対策としてのトラフィックキャプチャ、インフラ保護のための不要プロトコル排除。
*   **どのような場面で利用するか**: 
    *   同一セグメント内に存在するサーバとクライアント間の通信を、ルータを経由させずに L2 レイヤで制限したい場合。
    *   L3 インターフェイス（SVI）が存在しないスイッチ上で VLAN トラフィックを制御したい場合。
    *   特定のトラフィックのみを分析用ポートへミラーリング（Capture）したい場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **処理場所** | スイッチのハードウェア（TCAM）で処理されるため、ワイヤレートでの動作が可能。 |
| **適用範囲** | VLAN 全体に適用。方向（Ingress/Egress）の概念はなく、VLAN 内の全パケットが対象。 |
| **識別単位** | IPv4 ACL、IPv6 ACL、および MAC ACL を使用してトラフィックを定義。 |
| **主なアクション** | **Forward**（転送）、**Drop**（破棄）、**Capture**（指定ポートへのミラーリング）。 |
| **メリット** | 同一 VLAN 内通信（Intra-VLAN）を制御できる唯一の標準的な ACL 手法。 |
| **デメリット** | 設定ミスにより VLAN 全体の通信が遮断されるリスクがある（暗黙の Deny に注意）。 |
| **設計上の注意点** | 最後に必ず `action forward` を定義しないと、合致しない全通信がドロップされる。 |

---

## 🏗 動作原理

VACL は「VLAN アクセスマップ」というシーケンシャルなリスト構造で動作します。パケットが VLAN 内でスイッチングまたはルーティングされる前に、このマップと照合されます。

```text
[ Packet enters VLAN 10 ]
          ↓
[ VACL Check (vlan access-map) ]
          ↓
   Match ACL 101?  ----(Yes)----> Action: Drop
          ↓ (No)
   Match ACL 102?  ----(Yes)----> Action: Forward
          ↓ (No)
[ Implicit Deny ]  ------------> Action: Drop
```

---

## ⚙ 動作シーケンス

1.  **トラフィック定義**: `ip access-list` や `mac access-list` で、制御対象のパケットを指定します。
2.  **アクセスマップの作成**: `vlan access-map` コマンドでシーケンス番号付きのマップを作成し、上記 ACL を `match` 条件に設定します。
3.  **アクションの割り当て**: マッチしたパケットに対して `action forward` または `action drop` を定義します。
4.  **フィルタの適用**: `vlan filter` コマンドを使用して、作成したアクセスマップを特定の VLAN ID（または VLAN リスト）に関連付けます。
5.  **パケット処理**: パケットが VLAN に入ると、ハードウェアはマップを上から順にスキャンし、最初にマッチしたエントリのアクションを実行します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **「SVI なしでの制御」要件**: ラボ試験で「L3 インターフェイスを設定せずに、VLAN 10 内のホスト A からホスト B への HTTP 通信を禁止せよ」という指示があれば、迷わず VACL を選択してください。
*   **暗黙の拒否 (Implicit Deny)**: VACL には `route-map` や通常の ACL と同様に、最後に「すべてをドロップする」という暗黙のルールが存在します。**すべてのパケットを通過させるエントリ（`action forward` のみを持つシーケンス）を最後に忘れないように設定**することが、ラボでの合格と失点の分かれ目になります。
*   **Capture アクション**: 特定の攻撃パケットのみを IDS/IPS やアナライザに送るための `action forward capture` の設定手順が問われることがあります。
*   **MAC ACL との組み合わせ**: IP 以外のプロトコル（ARP など）を制御する場合、`mac access-list` を使用した VACL の構築スキルが必要です。
*   **適用順序の理解**: VACL は入力 ACL (Port ACL) や RACL (Router ACL) と併用される場合、処理の優先順位を正しく把握しておく必要があります。

---

## 🛠 設定方法

### 1. 基本的なパケット破棄（特定ホスト間通信の遮断）
```bash
! 1. 拒否したいトラフィックをACLで定義（permitで指定したものがVACLの対象）
ip access-list extended ACL-BLOCK-HOST
 permit ip host 10.1.1.10 host 10.1.1.20
!
! 2. VLANアクセスマップの作成
vlan access-map VACL-POLICY 10
 match ip address ACL-BLOCK-HOST
 action drop
!
! 3. 他の全てのトラフィックを許可するエントリ（重要）
vlan access-map VACL-POLICY 20
 action forward
!
! 4. VLANへフィルターを適用
vlan filter VACL-POLICY vlan-list 10
```

### 2. 特定トラフィックのキャプチャ（分析用）
```bash
vlan access-map VACL-CAPTURE 10
 match ip address ACL-MALICIOUS
 action forward capture
!
vlan access-map VACL-CAPTURE 20
 action forward
!
! 適用
vlan filter VACL-CAPTURE vlan-list 100
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **VACL の適用状態確認** | <code>show vlan filter</code> |
| **VLAN マップの定義確認** | <code>show vlan access-map [Map_Name]</code> |
| **ACL の内容確認** | <code>show ip access-lists</code> |
| **インターフェイスの統計確認** | <code>show interfaces [ID] counters</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| VLAN 内の全通信が止まった | 最後に <code>action forward</code> エントリがない | <code>show vlan access-map</code> | 全許可エントリを末尾に追加。 |
| 特定の ACL がマッチしない | ACL 内で <code>deny</code> を使用している | <code>show run \| sec access-list</code> | VACL で処理したいパケットは ACL 内で <code>permit</code> にする。 |
| VACL が反映されない | <code>vlan filter</code> が適用されていない | <code>show vlan filter</code> | <code>vlan filter [Name] vlan-list [ID]</code> を確認。 |
| CPU 負荷が異常に高い | ロギング (log オプション) の多用 | <code>show processes cpu sorted</code> | ソフトウェア処理されるオプションを外す。 |

---

## ⚠ 制限事項

*   **方向指定不可**: VACL はパケットが VLAN に入る際も出る際も一律にチェックされるため、一方向のみの制限をかける場合は ACL の送信元/宛先定義で工夫する必要があります。
*   **ハードウェア制限**: 使用できるマップの数やエントリ数は、スイッチの TCAM リソースに依存します。
*   **Private VLAN との共存**: 一部のプラットフォームでは、PVLAN (3.4.h) と VACL を同一 VLAN に設定する際に制限がある場合があります。

---

## 🔄 他技術との関連

*   **3.4.a DAI**: ARP パケットの制御。DAI で防げない特殊な ARP 攻撃を VACL で補完することがあります。
*   **3.4.e DHCP Snooping**: VACL を使用して不正な DHCP パケットをドロップすることも可能ですが、基本的には DHCP Snooping の使用が推奨されます。
*   **3.1.c iACLs**: インフラ保護。VACL を使用して、VLAN 内の管理通信（SSH/SNMP）をエッジで保護する多層防御を構成できます。

---

## 🧩 比較表

### VACL vs RACL (Router ACL)

| 特徴 | VACL (VLAN ACL) | RACL (Router ACL) |
| :--- | :--- | :--- |
| **適用インターフェイス** | **VLAN 全体** | **SVI / L3 Port** |
| **制御対象** | **同一 VLAN 内通信 (L2)** | **VLAN 越え通信 (L3)** |
| **方向 (Direction)** | 指定なし（全トラフィック） | In / Out 指定が必要 |
| **フィルタ単位** | VLAN Map 内の Match | インターフェイス毎の Group |

---

## 💡 ベストプラクティス

1.  **「Forward All」の徹底**: VACL を構築する際は、必ず最後に何もマッチ条件を持たない `action forward` シーケンスを配置する癖をつけてください。
2.  **明確な命名規則**: `VACL_VLAN10_PROTECT` のように、対象 VLAN と目的がわかる名前を使用します。
3.  **ACL での `permit` 指定**: VACL の `match` 条件として使う ACL では、**ドロップしたいトラフィックであっても `permit` と書く**必要がある点に注意してください（「このパケットを VACL の処理対象にする」という意味になるため）。
4.  **IPv6 への配慮**: デュアルスタック環境では、IPv4 ACL だけでなく IPv6 ACL もマップに含める必要があります。

---

## 📝 ラボ学習・設定サンプル例

### 1. 同一 VLAN 内の ICMP 禁止
*   **要件**: VLAN 10 内の端末間で Ping を禁止せよ。
*   **設定**: `permit icmp any any` を含む ACL を作成し、VACL で `action drop`、最後に `action forward`。

### 2. 特定サーバへの HTTP アクセス制限
*   **要件**: VLAN 20 内のクライアントからサーバ 192.168.20.100 への HTTP(80) のみを許可し、他は拒否せよ。

### 3. VACL によるトラフィックキャプチャ
*   **要件**: VLAN 50 を流れる Telnet 通信のみを Gi1/0/10 に接続されたアナライザへ送れ。

### 4. 非 IP プロトコル（AppleTalk等）の遮断
*   **要件**: `mac access-list` を使用し、特定の L2 プロトコルをドロップせよ。

### 5. マルチ VLAN フィルターの適用
*   **要件**: 同一のアクセスマップを VLAN 10-20, 30 に一括適用せよ。

### 6. IPv6 Neighbor Discovery の保護
*   **要件**: VACL を使用して特定の IPv6 制御メッセージ以外を遮断せよ。

### 7. RACL との併用シミュレーション
*   **課題**: パケットが SVI を通る際、VACL と RACL どちらが先に処理されるか検証せよ（通常 VACL が先）。

### 8. VACL による ARP スプーフィングの簡易対策
*   **要件**: 特定の不正 MAC からの ARP パケットをドロップせよ。

### 9. シーケンス番号によるエントリ挿入
*   **操作**: 既存の `vlan access-map` の途中に、新しいフィルタ条件を挿入せよ。

### 10. 空の VACL 適用による全遮断の検証
*   **課題**: `action forward` を持たない VACL を適用し、通信が途絶することを確認してから復旧せよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: `vlan filter MAP1 vlan-list 10` を設定した直後、VLAN 10 のすべての通信が停止した。原因として最も考えられるのは？
    *   **回答**: VACL の最後に `action forward` シーケンスが設定されておらず、暗黙の拒否によってすべてのパケットがドロップされているため。
2.  **Design**: 同一 VLAN 内に所属する複数の部門用 PC が、互いに通信できないようにしたい。L3 デバイスを介さずに実現する方法は？
    *   **回答**: **VACL** (または PVLAN) を使用して、同一 VLAN 内の通信をフィルタリングする。
3.  **コンフィグ読解**: ACL 10 で `permit ip any any` と設定し、VACL で `match ip address 10` と `action drop` を設定した場合、どのような挙動になるか？
    *   **回答**: その VLAN 内のすべての IP トラフィックがドロップされる。
4.  **実装**: 特定のセキュリティ攻撃パケットをドロップしつつ、解析のために IDS へコピーを送りたい。使用すべき `action` は？
    *   **回答**: `action drop` と、コピー用の `action forward capture`（ただしプラットフォームのサポート状況に依存）。
5.  **Design**: VACL を設定する際、VLAN 内の ARP 通信を維持するために必要な考慮事項は？
    *   **回答**: IP ACL だけでは ARP (非 IP) は制御されないが、末尾の `action forward` があれば ARP も許可される。もし MAC ACL を使用している場合は、明示的に ARP を許可する必要がある。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [Configuring VLAN Maps (Cisco.com)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3750/software/release/12-2_55_se/configuration/guide/scg3750/swvlan.html#wp1037314)
*   **Cisco Live (BRKSEC-2003)**
    *   [Securing the Infrastructure with ACLs](https://www.ciscolive.com/)
*   **Technical Notes**
    *   [VLAN Access Control Lists (VACLs) Overview](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-6000-series-switches/22093-vacl-22093.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「VACL は VLAN の壁面に設置されたフィルター」です。L2 スイッチングの過程でパケットが通り抜ける際に強制的にチェックされます。
*   **注意点**: ラボ試験では、通常の ACL のように `permit` が「許可」を意味するのではなく、「VACL のマッチ対象とする」という意味であることを常に意識してください。ドロップしたいなら `action drop` を組み合わせる必要があります。
*   **図解**: パケットが Ingress Port に入る -> VACL チェック (TCAM) -> スイッチング先決定 -> Egress Port へ。この流れの中で、宛先 MAC 学習よりも前に VACL が処理される場合があります。
