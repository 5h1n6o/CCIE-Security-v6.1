---
layout: default
title: 3.4.f-RA-Guard
nav_order: 6
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.4.f RA Guard

**RA Guard (IPv6 Router Advertisement Guard)** は、IPv6 環境における「First Hop Security」の主要なコンポーネントであり、ネットワーク内の不正なルータ広告（Router Advertisement: RA）をブロックするためのレイヤ 2 セキュリティ機能です。IPv4 における DHCP Snooping が「誰が IP を配るか」を制御するのと同様に、RA Guard は「誰がネットワークプレフィックスとデフォルトゲートウェイ情報を配るか」を制御し、中間者攻撃（MitM）や DoS 攻撃を防止します。

---

## 📘 概要

*   **機能概要**: スイッチポートに届く IPv6 ルータ広告（RA）およびルータ再構成メッセージをインターセプトし、そのポートが「信頼されている（Trusted）」か、またはメッセージの内容が事前に定義されたポリシーに合致するかを検証します。
*   **利用目的**: 不正なルータが広告を送信してホストの通信を自身の方向に誘導（ルータのなりすまし）したり、不正なプレフィックスを配布してネットワークを混乱させたりするのを防ぎます。
*   **どのような場面で利用するか**: エンタープライズネットワークのアクセス層（Access Layer）において、ユーザー端末が接続されるポートに対して適用します。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | ステートレス（単純なフィルタリング）またはステートフルな検証が可能。 |
| **用途** | 不正な IPv6 ルータの排除、プレフィックスハイジャックの防止。 |
| **メリット** | スイッチのハードウェアで RA をドロップできるため、パフォーマンスへの影響が少ない。 |
| **デメリット** | 設定を誤ると、正当なルータからの RA も遮断され、IPv6 通信ができなくなる。 |
| **対応機種** | Catalyst スイッチ (IOS-XE)、Nexus シリーズ。 |
| **制限事項** | 断片化された RA パケットの処理において、一部のプラットフォームで制限がある。 |
| **設計上の注意点** | Uplink（正当なルータが繋がるポート）は必ず **Trust** に設定する。 |

---

## 🏗 動作原理

RA Guard は、スイッチのインターフェイスを「ルータ役割（Trust）」と「ホスト役割（Untrust）」に分離する論理に基づいています。

```text
[ Legitimate IPv6 Router ]
          ↓ (RA: Prefix 2001:DB8:1::/64)
[ Trusted Port (Uplink) ]
          ↓
[ Access Switch (RA Guard Enabled) ]
          ↓
[ Untrusted Port (User) ] ←── [ Rogue Router ] (RA: "I am your Gateway!")
          ↓                         ┃
          ↓                         ┗━━ [ BLOCKED / DROP ]
[ IPv6 Client ]
```

---

## ⚙ 動作シーケンス

1.  **パケット受信**: スイッチがインターフェイスで IPv6 パケットを受信。
2.  **パケット識別**: 受信パケットが ICMPv6 Type 134 (RA) または Type 137 (Redirect) であるかを判別。
3.  **ポリシー照合**:
    *   **Trusted Port**: 検証なしでパケットを通過。
    *   **Untrusted Port**: 設定された `ipv6 nd raguard policy` と照合。
4.  **検証プロセス**: 送信元のリンクローカルアドレス、プレフィックス、M/O フラグ、優先度などがポリシーに合致するか確認。
5.  **アクション**: ポリシー違反（Untrusted ポートから RA が届くなど）を検知した場合、パケットをドロップし、ログを生成する。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **基本的な「Untrusted」設定**: ラボ試験で「不正な RA をブロックせよ」とだけ指示された場合、ポリシーを作成し、アクセスポートにアタッチするだけで機能します。デフォルトでは Untrusted 扱いになるためです。
*   **VLAN ベース vs ポートベース**: RA Guard は VLAN 単位 (`vlan configuration`) またはインターフェイス単位で設定可能です。要件をよく読み、適用範囲を間違えないようにしてください。
*   **ステートレス（Stateless）**: 最も一般的なラボ要件です。特定の送信元をチェックせず、単にそのポートで RA を許可しない設定です。
*   **ロギングの確認**: `debug ipv6 nd` や `debug ipv6 nd secured` を実行した際に出力される `! DROP: ND_ROUTER_ADVERT reason=2` といったログの意味を理解しておく必要があります。
*   **IPv6 Snooping との関連**: RA Guard は IPv6 Snooping フレームワークの一部です。`device-tracking` (3.4.b) と組み合わせて、バインディング情報の整合性を問われることがあります。

---

## 🛠 設定方法

### 1. ポリシーの作成 (IOS-XE)
```bash
ipv6 nd raguard policy RAGUARD-CLIENT-PORT
  device-role monitor
  ! デフォルトの host ロール（RA拒否）を指定する場合
  device-role node
```

### 2. インターフェイスへの適用 (Access Port)
```bash
interface GigabitEthernet1/0/1
  description User-Access-Port
  ipv6 nd raguard attach-policy RAGUARD-CLIENT-PORT
```

### 3. VLAN への適用
```bash
vlan configuration 10
  ipv6 nd raguard attach-policy RAGUARD-CLIENT-PORT
```

### 4. 信頼できるポートの設定 (Uplink)
信頼できるポート（ルータ側）には、デバイスロールを `switch` に設定したポリシーを適用するか、あるいは設定自体を行わないことで（デフォルトで許可される場合があるが）明示的に許可します。
```bash
ipv6 nd raguard policy RAGUARD-TRUSTED
  device-role switch
!
interface GigabitEthernet1/0/24
  ipv6 nd raguard attach-policy RAGUARD-TRUSTED
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **全体の設定状態確認** | <code>show ipv6 nd raguard policy</code> |
| **インターフェイスごとの統計** | <code>show ipv6 nd raguard statistics</code> |
| **ドロップログのデバッグ** | <code>debug ipv6 nd secured</code> |
| **隣接関係の確認** | <code>show ipv6 neighbor</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 正当な RA がドロップされる | Uplink に <code>device-role switch</code> が設定されていない | <code>show ipv6 nd raguard statistics</code> でドロップを確認し、ロールを修正。 |
| RA Guard が効かない | ポリシーが物理ポートではなく VLAN に適用され、且つ不整合がある | <code>show run vlan</code> を確認。 |
| クライアントが IPv6 アドレスを取得できない | RA がすべて止まっている | <code>show ipv6 nd raguard statistics</code> を確認。 |
| ログに "reason=2" と出る | Untrusted ポートで RA を受信した | 送信元デバイスがルータでないか確認、またはポート設定の修正。 |

---

## ⚠ 制限事項

*   **ハードウェアの限界**: 非常に古いスイッチモデルでは、IPv6 RA Guard がソフトウェア処理（CPU）になる場合があり、DoS 攻撃に対して脆弱になる可能性があります。
*   **断片化パケット**: IPv6 ヘッダーの後に拡張ヘッダーが多用され、RA 情報がパケットの深い位置にある場合、ハードウェアが検知できないことがあります（`hardware inspection` オプション等で対処）。
*   **SeND との共存**: Secure Neighbor Discovery (SeND) と併用する場合、暗号化署名の検証プロセスとの順序に注意が必要です。

---

## 🔄 他技術との関連

*   **3.4.e DHCP Snooping**: IPv4 での類似機能。設計コンセプトはほぼ同一です。
*   **3.4.b IPDT / Device Tracking**: RA Guard で学習された情報を元に、スイッチが IPv6 バインディングテーブルを維持します。
*   **IPv6 Snooping**: RA Guard を含む IPv6 セキュリティ機能の総称。
*   **VACL (3.4.g)**: 特定の L2/L3 トラフィックを VLAN 内で止める際、RA (ICMPv6 Type 134) を手動で止めることも可能ですが、RA Guard の方が高機能です。

---

## 🧩 比較表

### RA Guard vs ACL フィルタリング

| 特徴 | RA Guard | IPv6 ACL (PACL/VACL) |
| :--- | :--- | :--- |
| **識別単位** | ICMPv6 Type 単位の自動認識 | IP/ポートの手動指定 |
| **詳細検証** | プレフィックスや優先度の検査が可能 | 不可（パケット単位の許可/拒否のみ） |
| **設定の容易さ** | 高（ポリシーベース） | 低（各ルールを細かく定義） |
| **推奨用途** | **不正ルータ対策の標準** | 汎用的な通信制御 |

---

## 💡 ベストプラクティス

1.  **エッジポートでの強制**: すべてのユーザー接続ポート（Access Port）に RA Guard ポリシーを適用することを標準のハードニングとします。
2.  **デバイスロールの明示**: デフォルトに頼らず、`device-role node`（ホスト）と `device-role switch`（ルータ）を明確に使い分けます。
3.  **プレフィックスリストの活用**: 信頼できるポートであっても、許可するプレフィックスをポリシーで制限 (`match ipv6 access-list` 等) することで、設定ミスによる誤った広報を防ぎます。
4.  **ロギングの有効化**: 攻撃を早期に検知するため、違反発生時に Syslog を生成する設定を推奨します。

---

## 📝 ラボ学習・設定サンプル例

### 1. シンプルなアクセスポート保護
*   **要件**: ポート Gi1/0/1 からの RA 送信を一切禁止せよ。
*   **設定**: 
    ```bash
    ipv6 nd raguard policy DROP-RA
     device-role node
    interface Gi1/0/1
     ipv6 nd raguard attach-policy DROP-RA
    ```

### 2. 特定プレフィックスのみ許可する Uplink
*   **要件**: ルータが繋がる Gi1/0/24 では `2001:DB8:A::/64` の RA のみ許可せよ。
*   **設定**: 
    ```bash
    ipv6 access-list ALLOW-PF
     permit 2001:db8:a::/64 any
    ipv6 nd raguard policy TRUST-RA
     device-role switch
     match ipv6 prefix-list ALLOW-PF
    ```

### 3. VLAN 全体への一括適用
*   **要件**: VLAN 20 に所属する全てのポートで RA をブロックせよ。
*   **設定**: `vlan configuration 20` 内で `attach-policy`。

### 4. 優先度（Preference）によるフィルタ
*   **要件**: 優先度が `High` に設定された RA のみを許可せよ。

### 5. 管理フラグ (O-flag) の検証
*   **要件**: Stateless アドレス設定を禁止するため、O フラグがセットされていない RA をドロップせよ。

### 6. 送信元リンクローカルアドレスの固定
*   **要件**: 特定のルータ MAC アドレスから来る RA のみ許可せよ。

### 7. RA Guard と Port Security の併用
*   **課題**: L2 接続と IPv6 ルーティングプレフィックスの両方を保護する。

### 8. RA Guard 統計のリセット
*   **操作**: `clear ipv6 nd raguard statistics`。

### 9. ステートフル RA Guard
*   **要件**: 以前学習した正当なルータ情報と一致しない RA を破棄せよ。

### 10. ドロップログのデバッグ検証
*   **課題**: 不正デバイスから RA を送り、`debug ipv6 nd secured` でドロップ理由を確認せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: インターフェイスに `ipv6 nd raguard attach-policy P1` が設定され、`P1` 内で `device-role monitor` が指定されている。このポートの挙動は？
    *   **回答**: RA を受信してもドロップせず、内容を監視してログを出力する（インラインでの遮断は行わない）。
2.  **トラブルシュート**: RA Guard 設定後、クライアントがデフォルトゲートウェイを見失った。まず確認すべき項目は？
    *   **回答**: 正当なルータが接続されているポートが `Trusted`（`device-role switch`）になっているか、`show ipv6 nd raguard statistics` でドロップが増えていないかを確認する。
3.  **Design**: DHCPv6 サーバーが存在する環境で、RA Guard ポリシーに追加すべき設定は？
    *   **回答**: M フラグ（Managed Address Configuration）または O フラグ（Other Configuration）の状態を検証する `managed-config-flag` 等のオプション。
4.  **実装**: 物理ポートごとにポリシーを設定するのが面倒な場合、推奨される設定箇所は？
    *   **回答**: **VLAN Configuration** モード。
5.  **コンフィグ読解**: `debug ipv6 nd secured` にて `! DROP: ND_ROUTER_ADVERT reason=2` と表示された。この "reason=2" が指す意味は？
    *   **回答**: ポリシーが `host` ロール（または `node`）に設定されているポートで RA を受信したことによる拒否。

---

## 🔗 参考リソース

*   **Cisco Configuration Guide**
    *   [IPv6 RA Guard](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipv6_fhsec/configuration/xe-16/ip6-fhs-xe-16-book/ip6-ra-guard.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [IPv6 Security Features on Cisco Switches](https://www.ciscolive.com/)
*   **Technical Notes**
    *   [Understanding IPv6 First Hop Security](https://www.cisco.com/c/en/us/support/docs/ip/ip-version-6-ipv6/113141-ipv6-first-hop-security-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「RA Guard は IPv6 の門番」です。誰でもルータになれる IPv6 の自由さを、エンタープライズの秩序で縛る機能だと覚えましょう。
*   **注意点**: ラボ試験では `ipv6 nd raguard` と `ipv6 nd inspection` を混同しやすいですが、前者は RA（ルータ）対策、後者は ND（近隣探索/ARP相当）対策です。
*   **図解**: 
    1. RA 到着 
    2. ポリシーチェック（Role 確認）
    3. 合致すれば Pass、不一致なら Drop + Log。
