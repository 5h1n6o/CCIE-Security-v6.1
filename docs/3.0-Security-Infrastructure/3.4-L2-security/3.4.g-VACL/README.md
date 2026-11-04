---
layout: default
title: 3.4.g-VACL
nav_order: 7
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.4.g VACL

CCIE Security v6.1 Blueprint の **3.4.g VACL (VLAN Access Control List / VLAN Access Map)** に関する実戦的かつ高度な技術学習メモです。Catalyst スイッチおよび Cisco IOS/IOS-XE 環境における VLAN アクセス españal/制御マップ（VLAN Access Map）の動作メカニズム、TCAM ルックアップ順序、トラフィックリダイレクト（Redirect）、キャプチャ（Capture）、ならびに RACL/PACL/PVLAN との相互作用を包括的に解説します。

---

## 📘 概要

VLAN Access Control List（VACL、別名 VLAN Access Map）は、レイヤ 2 スイッチング環境において特定 VLAN 内を流れるすべてのパケット（同一 VLAN 内のブリッジトラフィックおよび VLAN 間を通過するルーテッドトラフィックの両方）に対してアクセス制御を実施するセキュリティ機能です。

### 1. 機能概要
VACL は、標準的なインターフェイス ACL（RACL: Routed ACL）がレイヤ 3 インターフェイスのインバウンド/アウトバウンドでルーティングされるパケットのみを制御するのとは異なり、VLAN 全体を対象とします。アクセスマップ形式（`vlan access-map`）でポリシーを定義し、パケットが ACL の `permit` 句に一致した際に実行するアクションとして **`forward`（転送）**、**`drop`（破棄）**、または **`redirect`（特定ポートへの転送）** を指定できます。

### 2. 利用目的
* **同一 VLAN 内（Layer 2 Bridged）トラフィックのフィルタリング**: 同一サブネット・同一 VLAN に所属するホスト同士の直接通信を L3 境界を介さずに L2 スイッチング段階で制御します。
* **特定の不正プロトコルや不要通信の遮断**: VLAN 内での不要なブロードキャスト、マルチキャスト、特定ポート（NetBIOS, SMB, MS-RPC 等）のレイヤ 2 レベルでのブロック。
* **トラフィックのインライン迂回・キャプチャ（Redirect / Capture）**: SPAN を使用せずに、特定トラフィックのみを IDS/IPS やセキュリティ解析装置が接続された物理ポートへリダイレクト（`redirect`）またはキャプチャ（`capture`）します。

### 3. どのような場面で利用するか
* **マイクロセグメンテーションの補完**: 同一 VLAN 内の端末間通信を制限したいが、Private VLAN (PVLAN) を構成できない既存ネットワーク環境への導入。
* **高セキュリティ VLAN のインバンド保護**: データベースサーバ VLAN や PCI-DSS 対象 VLAN において、同一 VLAN 内の他のサーバからの側方移動（Lateral Movement）やマルウェア感染拡大を防ぐ。
* **セキュリティモニタリング**: VLAN 内の特定の異常通信（特定 IP からの特定ポート宛通信）のみをセキュリティセンサに抽出・リダイレクトするケース。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | VLAN 全体（Bridge および Routed トラフィック）に適用される Access Map 形式の L2/L3/L4 パケットフィルタリング |
| **用途** | 同一 VLAN 内端末間通信制御、L2 ポートリダイレクト（IDS/IPS 誘導）、不要プロトコルの同一 VLAN 内遮断 |
| **メリット** | TCAM によるハードウェア処理（ワイヤスピード）、Bridge トラフィック制御可能、`redirect` や `capture` アクションが使用可能 |
| **デメリット** | 設定に Access Map 構造が必要で複雑、シーケンス管理や ACL の permit/deny の理解ミスによる誤遮断リスクが高い |
| **対応機種** | Catalyst 3650/3850/9000 シリーズ、Catalyst 4500/6500 シリーズ、Cisco IOS / IOS-XE スイッチプラットフォーム |
| **制限事項** | SVI や EtherChannel への直接リダイレクト不可（一部プラットフォーム）、暗黙の DROP による全遮断事故に注意 |
| **設計上の注意点** | ACL 内の `permit` が Access Map のアクション対象となる。ACL の `deny` は「不一致（次の条件へスキップ）」を意味する |

---

## 🏗 動作原理

VACL は Catalyst スイッチの TCAM（Ternary Content Addressable Memory）にプログラミングされ、パケットのフレーム受取時にワイヤスピードでルックアップ処理が行われます。

```
[ Ingress Port (VLAN X) ]
         │
         ▼
[ PACL Check (Port ACL) ] ── (Drop) ──► パケット破棄
         │ (Permit/Pass)
         ▼
[ VACL Check (VLAN Access Map) ]
         │
         ├─── (Action: Drop) ─────────► パケット破棄
         ├─── (Action: Redirect) ─────► 指定物理ポートへ出力 (スイッチングスキップ)
         └─── (Action: Forward)
                   │
                   ▼
       ┌───────────┴───────────┐
       ▼                       ▼
[ L2 Bridging ]         [ L3 Routing ]
(同一VLAN内宛)           (他VLAN宛/SVI経由)
       │                       │
       │                       ▼
       │               [ RACL Ingress Check ]
       │                       │
       │                       ▼
       │               [ Routing / FIB Lookup ]
       │                       │
       │                       ▼
       │               [ RACL Egress Check ]
       │                       │
       ▼                       ▼
[ Egress VACL Check ] ◄────────┘
       │
       ▼
[ Egress Port (VLAN Y) ]
```

### パケット一致とアクションのロジック
VACL の評価ロジックは標準的な ACL や Route-map とは異なり、**「参照する ACL（IP ACL / MAC ACL）の permit/deny」** と **「vlan access-map の action (forward/drop/redirect)」** の組み合わせで決定されます。

1. **ACL の `permit` 句に一致した場合**:
   - `vlan access-map` で定義された `action`（`forward` / `drop` / `redirect`）を実行します。
   - 処理が決定したため、評価はそこで終了します（Matches & Terminates）。
2. **ACL の `deny` 句に一致した場合**:
   - その ACL 条件には「不一致」とみなされ、同一 `vlan access-map` 内の次の `match` 条件、または次の Sequence Number（シーケンス）の評価へ進みます（Skips to next match/sequence）。
3. **すべてのシーケンスに不一致の場合**:
   - VLAN Access Map の末尾には暗黙の `drop` 条件（Implicit Drop）が存在するため、パケットは破棄されます。

---

## ⚙ 動作シーケンス

スイッチ上でパケットが VACL に遭遇した際の内部処理ステップは以下の通りです。

```
1. パケットが Ingress ポートに入力 (VLAN ID 割り当て)
   ↓
2. TCAM による VACL ルックアップ (配列シーケンス順: Seq 10, Seq 20, ...)
   ↓
3. Match 条件判定 (IP ACL または MAC ACL 参照)
   ├── Match (ACL permit) ──► Access-map Action 判定
   │                              ├── Action: Forward  ──► 正常処理継続 (L2/L3 Forwarding)
   │                              ├── Action: Drop     ──► パケット即時破棄 (TCAM Drop)
   │                              └── Action: Redirect ──► 指定 Egress ポートへ直接転送
   │
   └── No Match (ACL deny または 未一致)
          ↓
4. 次の Sequence Number へ移行
   ↓ (最終 Sequence まで不一致の場合)
5. 暗黙の DROP (Implicit Drop) によりパケット破棄
```

### ステップ別詳細解説
* **Step 1 (Ingress)**: フレームが物理ポートで受信され、VLAN ID が確定します。
* **Step 2 (TCAM Lookup)**: 該当 VLAN に `vlan filter` で VACL がバインドされている場合、ハードウェア TCAM 内でパケットヘッダー（L2 MAC, L3 IP, L4 Port）が検索されます。
* **Step 3 (Match & Action)**:
  - `match ip address IP_ACL` または `match mac address MAC_ACL` が評価されます。
  - 該当 ACL の `permit` 行に合致した場合、Access Map の `action` が適用されます。
  - `action redirect GigabitEthernet1/0/24` が指定されている場合、通常のおよび L3 ルーティング処理をバイパスし、フレームは直接指定されたポート G1/0/24 から送信されます。
* **Step 4 (Implicit Drop Prevention)**: VACL の全トラフィック遮断事故を防ぐため、特定の通信のみを `drop` したい場合は、そのトラフィックを ACL で `permit` して `action drop` を設定し、残りの全通信用に `permit any` の ACL を用意して `action forward` を設定するシーケンスを最後に配置する必要があります。

---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security ラボ試験において VACL は、シンプルでありながら非常に設定ミスが発生しやすい技術の一つです。

### 1. Blueprintで重要なポイント
* **ACL の Permit/Deny と Access Map の Action の混同防止**:
  - 間違いの典型: 「HTTP 通信をドロップしたいから、IP ACL で `deny tcp any any eq 80` と書いて `action drop` にする」 ── これを行うと、HTTP 通信は ACL で `deny`（不一致）となり、次の条件または暗黙の DROP に落ちて通信全体が破壊されます。
  - 正解: IP ACL で `permit tcp any any eq 80` と記述してマッチさせ、`vlan access-map` で `action drop` を指定します。
* **暗黙の DROP（Implicit Drop）対策**:
  - 明示的に `action forward` を行うシーケンスを忘れると、VLAN 内の全通信が不一致となり停止します。
* **Redirect アクションと出向ポート制限**:
  - `action redirect` で指定するポートは物理ポート（SPAN セッション等と競合しないこと）である必要があり、SVI や Port-channel に直接指定できるかどうかのプラットフォーム制限を意識する。

### 2. ラボ試験で設定させられそうな要件パターン
1. **「同一 VLAN 内の端末 Host-A (192.168.10.10) と Host-B (192.168.10.20) 間の通信のみをブロックし、それ以外の通信（デフォルトゲートウェイへの通信等）は許可せよ」**
2. **「VLAN 20 内の全端末からの UDP ポート 69 (TFTP) 通信を L2 レベルで破棄し、ログを記録せよ」**
3. **「VLAN 30 を通過する特定のパケット（Web通信）を、セキュリティ監視装置が接続された Interface GigabitEthernet1/0/12 へリダイレクトせよ」**
4. **「IP 以外の非 IP トラフィック（特定 MAC アドレス）を VACL を用いて遮断せよ」**

### 3. よくある設定ミスとトラブル
* **全通信断の発生**: 最後の `action forward` シーケンスを忘れた、またはマッチ用 ACL の記述を間違えた。
* **MAC ACL と IP ACL の重複適用エラー**: 同一 Sequence 内で IP ACL と MAC ACL の両方を match させようとして意図しない動作になる（プラットフォームごとの二重ルックアップ挙動）。
* **Logging（ログ取得）のパフォーマンス影響**: `action drop log` や `action forward log` を多用すると CPU に負荷がかかるため、試験要件で指定された場合のみ使用する。

---

## 🛠 設定方法

VACL の設定手順は大きく分けて 3 つのステップから構成されます。
1. マッチング用 ACL (IP ACL / MAC ACL) の作成
2. VLAN Access Map (`vlan access-map`) の作成とアクション定義
3. VLAN へのフィルタ適用 (`vlan filter`)

### 設定例 1: 基本的な VACL（特定ホスト間通信遮断＋他通信許可）
VLAN 10 内で 192.168.10.10 と 192.168.10.20 間の通信のみをドロップし、その他の通信をすべて許可する設定です。

```bash
! Step 1: マッチング用 ACL の作成 (permit で対象トラフィックを指定)
ip access-list extended ACL_BLOCK_HOSTS
 permit ip host 192.168.10.10 host 192.168.10.20
 permit ip host 192.168.10.20 host 192.168.10.10

ip access-list extended ACL_ALLOW_ALL
 permit ip any any

! Step 2: VLAN Access Map の作成
vlan access-map VACL_VLAN10 10
 match ip address ACL_BLOCK_HOSTS
 action drop

vlan access-map VACL_VLAN10 20
 match ip address ACL_ALLOW_ALL
 action forward

! Step 3: VLAN への適用
vlan filter VACL_VLAN10 vlan-list 10
```

### 設定例 2: トラフィックリダイレクト（Redirect）設定
VLAN 20 内の HTTP 通信 (TCP 80) を、セキュリティ分析装置が接続された物理ポート GigabitEthernet1/0/20 へリダイレクトする設定です。

```bash
ip access-list extended ACL_HTTP_TRAFFIC
 permit tcp any any eq 80

ip access-list extended ACL_PERMIT_ALL
 permit ip any any

vlan access-map VACL_REDIRECT_MAP 10
 match ip address ACL_HTTP_TRAFFIC
 action redirect GigabitEthernet1/0/20

vlan access-map VACL_REDIRECT_MAP 20
 match ip address ACL_PERMIT_ALL
 action forward

vlan filter VACL_REDIRECT_MAP vlan-list 20
```

---

## 🔍 検証コマンド

VACL の設定状態および動作状態を確認するための主要コマンド一覧です。

| 目的 | コマンド | 解説 |
| :--- | :--- | :--- |
| **VACL 全体確認** | <code>show vlan access-map</code> | 定義されているすべての VLAN Access Map とシーケンス、Match 句、Action を確認 |
| **VLAN 適用確認** | <code>show vlan filter</code> | どの VLAN にどの VLAN Access Map が適用されているかをバインディング一覧表示 |
| **特定 Access Map 詳細** | <code>show vlan access-map VACL_NAME</code> | 指定した VLAN Access Map の設定内容のみを表示 |
| **TCAM プログラミング確認** | <code>show platform hardware fed switch active fwd-asic drop</code> | TCAM ハードウェア上で VACL によりドロップされたパケットカウンター（Cat9k） |
| **VLAN 統計確認** | <code>show mac address-table vlan 10</code> | VACL 適用 VLAN 内の MAC アドレス学習状態の確認 |
| **デバッグ** | <code>debug vlan access-map</code> | VACL 適用時の制御プレーンイベントや構成変更ログの追跡 |

### 検証コマンドの出力例

```text
Switch# show vlan filter
VLAN map VACL_VLAN10 is configured on vlan 10

Switch# show vlan access-map
Vlan access-map "VACL_VLAN10"  10
  Match clauses:
    ip address: ACL_BLOCK_HOSTS
  Action:
    drop
Vlan access-map "VACL_VLAN10"  20
  Match clauses:
    ip address: ACL_ALLOW_ALL
  Action:
    forward
```

---

## 🚨 トラブルシュート

VACL 導入時によく発生する障害症状、原因、確認コマンド、および対処方法を表形式で整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **VLAN 内の全通信が遮断される** | Access Map 末尾に `action forward` を含むシーケンスが存在しない（Implicit Drop） | <code>show vlan access-map</code> | 全許可用 ACL (`permit ip any any`) を指定した `action forward` シーケンスを Access Map の最後に追加 |
| **ブロックしたい通信が通り抜ける** | マッチ用 ACL で `deny` を書いてしまっているため Access Map でスルーされている | <code>show ip access-lists</code> | マッチ用 ACL の対象条件を `deny` から `permit` に変更し、Access Map 側で `action drop` を設定 |
| **`vlan filter` が適用されない** | 指定した VLAN が VLAN データベースに存在しない、またはアクティブでない | <code>show vlan brief</code> | 該当 VLAN を作成（`vlan 10`）し、ステータスを active に変更 |
| **リダイレクトが機能しない** | リダイレクト先ポートが Down している、または EtherChannel/SVI を指定している | <code>show interface status</code> | リダイレクト先インターフェイスの UP/UP 状態を確認。単一物理ポートを指定する |
| **DHCP 取得に失敗する** | VACL で DHCP 通信 (UDP 67/68) が考慮されておらず DROP されている | <code>show ip access-lists</code> | VACL の許可用 ACL に DHCP 通信や ARP 通信が確実に含まれるよう調整 |

---

## ⚠ 制限事項

VACL の実装および設計におけるプラットフォーム・ライセンス上の制限事項です。

1. **暗黙の DROP (Implicit Drop)**:
   - Route-map や ACL と同様に、VLAN Access Map の末尾には暗黙の drop 条件が存在します。明示的な `action forward` が存在しない場合、該当 VLAN の全パケットがドロップされます。
2. **リダイレクトターゲットの制限**:
   - `action redirect` で指定できるポートは、プラットフォームにより限定されます。一部のスイッチでは EtherChannel や SVI、SPAN 宛先ポートへのリダイレクトがサポートされません。
3. **パケットキャプチャ (Capture) の制限**:
   - `action forward capture` は、Hardware/ASIC レベルでパケットキャプチャ用 ASICs や CPU へ転送する機能ですが、TCAM リソースを余分に消費します。
4. **IPv6 と MAC ACL の相互作用**:
   - プラットフォームによっては、同一 VLAN Access Map 内で IPv6 ACL と MAC ACL を混在させた場合のハードウェアルックアップ順序に制限があります。
5. **VLAN 数の制限**:
   - `vlan filter` は複数の VLAN に同一 Access Map をバインド可能ですが、TCAM エントリ上限を超過するとソフトウェア転送に落ちるか設定が拒否されます。

---

## 🔄 他技術との関連

VACL は他のセキュリティ技術やネットワーク機能と密接に関連しており、それらと組み合わせて多層防御（Defense in Depth）を構築します。

* **RACL (Routed ACL)**:
  - RACL は SVI やルーテッドポートに適用され、L3 境界を跨ぐパケット（VLAN 間通信）のみを制御します。
  - パケットの流れ: インバウンドパケットは **Inbound PACL ──► Inbound VACL ──► Inbound RACL ──► Routing ──► Outbound RACL ──► Outbound VACL** の順で評価されます。同一 VLAN 内の L2 通信では RACL は通過せず VACL のみが適用されます。
* **PACL (Port ACL)**:
  - スイッチポート（L2 物理ポート）に直接 `ip access-group` で適用される ACL。
  - PACL は VACL よりも前に評価されます。PACL で drop されたパケットは VACL まで到達しません。
* **Private VLANs (PVLAN)**:
  - PVLAN も同一 VLAN 内の L2 隔離技術ですが、PVLAN は Secondary VLAN（Isolated/Community）のポート分類による単純制御を行います。
  - PVLAN が構成された VLAN に VACL を適用する場合、Primary VLAN に適用した VACL は全プロムスキャスポートおよびセカンダリ通信に影響し、Secondary VLAN にバインドした VACL はそのセカンダリ VLAN 内の通信に作用します。
* **DAI (Dynamic ARP Inspection) / DHCP Snooping**:
  - DAI や DHCP Snooping は ARP パケットや DHCP パケットを検査しますが、VACL はそれらのパケットにも作用します。
  - VACL で ARP 通信（MAC ACL または IP ACL で非 IP としてキャッチ）をドロップしてしまうと DAI に到達する前に破棄されるため、VACL 設計時は ARP トラフィックを阻害しないよう配慮が必要です。
* **SPAN / RSPAN / ERSPAN**:
  - ポートミラーリング機能である SPAN と VACL の `action redirect` または `action forward capture` は、トラフィック監視の用途で競合または相補的に利用されます。

---

## 🧩 比較表

### L2/L3 アクセス制御機能の比較（VACL vs RACL vs PACL）

| 比較項目 | VACL (VLAN ACL) | RACL (Routed ACL) | PACL (Port ACL) |
| :--- | :--- | :--- | :--- |
| **適用場所** | VLAN 全体 (`vlan filter`) | L3 インターフェイス (SVI, Routed Port) | L2 物理ポート / EtherChannel |
| **対象トラフィック** | 同一 VLAN 内 L2 通信 + VLAN 間 L3 通信 | VLAN 間 L3 ルーテッド通信のみ | 該当ポートに入力される全 L2/L3 通信 |
| **方向 (Direction)** | インバウンド/アウトバウンドの区別なし | Ingress / Egress 指定可能 | Ingress のみ |
| **アクション** | Forward, Drop, Redirect, Capture | Permit, Deny | Permit, Deny |
| **設定構造** | VLAN Access Map (`vlan access-map`) | 標準/拡張 Access List (`ip access-group`) | 標準/拡張/MAC Access List (`ip access-group`) |
| **評価タイミング** | PACL の後、RACL の前 | VACL の後（Ingress）、VACL の前（Egress） | 最も最初 (Ingress 入力直後) |

---

### レイヤ 2 分離技術の比較（VACL vs Private VLAN）

| 比較項目 | VACL (VLAN Access Map) | Private VLAN (PVLAN) |
| :--- | :--- | :--- |
| **基本コンセプト** | アクセスリストと Access Map による柔軟なパケット記述・条件制御 | ポート種別 (Promiscuous / Isolated / Community) による構造的孤立 |
| **制御粒度** | IP, MAC, Port 番号による精密なフィルタリング | ポート単位・VLAN 構造単位の全遮断（Isolated） |
| **設定の複雑さ** | 高い（ACL + Access Map シーケンス管理が必要） | 中程度（VLAN 構造のマッピングとポート割り当て） |
| **リダイレクト機能** | 有り (`action redirect`) | 無し |
| **主な用途** | 特定端末間通信遮断、特定プロトコルブロック、IDS 誘導 | 多数のクライアント/サーバの簡易 L2 相互孤立化 |

---

## 💡 ベストプラクティス

1. **末尾の `action forward` シーケンスを必ず明記する**:
   - VACL 設定時は、最初に全通信許可用 ACL (`permit ip any any` および非 IP 通信用 `permit mac any any`) を指定したシーケンスを最後に配置する設計を標準化し、全全遮断事故を防ぎます。
2. **IP トラフィックと非 IP トラフィック（ARP等）の分離配慮**:
   - `match ip address` の設定のみを行った場合、IP 以外のプロトコル（ARP、RARP、CDP、LLDP）がどのように扱われるかを理解する。IP ACL の不一致で次のシーケンスに落ち、最終的に `drop` されないよう、必要に応じて MAC ACL や明示的な許可ルールを設計します。
3. **シーケンス番号の刻み幅を 10 単位にする**:
   - `vlan access-map VACL_NAME 10`, `20`, `30` のようにシーケンス番号の間隔を空けて設定し、将来のルールの割り込み挿入を容易にします。
4. **`action redirect` 使用時のバックアップポート配慮**:
   - リダイレクト先のセキュリティ機器（IDS/IPS）がダウンした場合、通信が停止するため、必要に応じて冗長構成や物理リンクモニタリングを設計します。
5. **事前検証の実施**:
   - 本番 VLAN に `vlan filter` を適用する前に、テスト VLAN またはキャプチャ環境で TCAM ヒットカウントおよび通信テストを実施します。

---

## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験を想定した 10 個の実践的な設定サンプル例です。

---

### シナリオ 1: 同一 VLAN 内特定ホスト間通信遮断 (Host-to-Host Isolation)

#### 問題
VLAN 10 に所属する 2 台のホスト Host-A (10.1.10.11) と Host-B (10.1.10.12) は同一 L2 セグメントに存在します。セキュリティ要件に基づき、この 2 台のホスト間の直接通信を VACL を用いて遮断してください。ただし、両ホストからデフォルトゲートウェイ (10.1.10.254) および他の端末への通信は許可しなければなりません。

#### 要件
* VLAN 10 上で Host-A (10.1.10.11) と Host-B (10.1.10.12) 間の双方向 IP 通信をドロップすること。
* その他の全 IP トラフィックおよび L2 通信（ARP 等）は正常に通過させること。
* VACL を使用して実装すること。

#### 設定例

```bash
! Step 1: 遮断対象の双方向 IP トラフィックを定義する ACL
ip access-list extended ACL_BLOCK_A_B
 permit ip host 10.1.10.11 host 10.1.10.12
 permit ip host 10.1.10.12 host 10.1.10.11

! 全通信許可用の ACL
ip access-list extended ACL_PASS_ALL
 permit ip any any

! Step 2: VLAN Access Map の構成
vlan access-map MAP_HOST_ISOLATION 10
 match ip address ACL_BLOCK_A_B
 action drop

vlan access-map MAP_HOST_ISOLATION 20
 match ip address ACL_PASS_ALL
 action forward

! Step 3: VLAN 10 への適用
vlan filter MAP_HOST_ISOLATION vlan-list 10
```

#### 検証方法
1. Host-A (10.1.10.11) から Host-B (10.1.10.12) へ ping を実行し、タイムアウト（失敗）することを確認します。
2. Host-A (10.1.10.11) から Gateway (10.1.10.254) へ ping を実行し、成功することを確認します。
3. スイッチで `show vlan access-map` および `show vlan filter` を実行し、バインド状態を確認します。

---

### シナリオ 2: MAC ACL を使用した非 IP トラフィック（特定 MAC アドレス）のドロップ

#### 問題
VLAN 20 内において、特定のアタッチメントデバイス（MAC アドレス: `0011.2233.4455`）から送出されるすべての非 IP トラフィック（L2 フレーム）を破棄し、それ以外のすべての通信を許可してください。

#### 要件
* MAC アドレス `0011.2233.4455` を送信元とする非 IP トラフィックを VACL でドロップすること。
* その他の IP および非 IP トラフィックはすべて許可すること。

#### 設定例

```bash
! Step 1: 対象 MAC アドレスを識別する MAC ACL
mac access-list extended MAC_BLOCK_DEV
 permit host 0011.2233.4455 any

ip access-list extended ACL_IP_ALL
 permit ip any any

mac access-list extended MAC_PASS_ALL
 permit any any

! Step 2: Access Map 構成
vlan access-map MAP_MAC_FILTER 10
 match mac address MAC_BLOCK_DEV
 action drop

vlan access-map MAP_MAC_FILTER 20
 match ip address ACL_IP_ALL
 action forward

vlan access-map MAP_MAC_FILTER 30
 match mac address MAC_PASS_ALL
 action forward

! Step 3: VLAN 20 にバインド
vlan filter MAP_MAC_FILTER vlan-list 20
```

#### 検証方法
1. 該当 MAC アドレスを持つ端末から非 IP フレーム（例: CDP、LLDP、または非 IP イーサネットフレーム）を送信し、ドロップされることを確認します。
2. `show vlan access-map` を実行して、MAC マッチ条件と Action drop が正しく定義されているか確認します。

---

### シナリオ 3: 特定 Web トラフィックのセキュリティ機器ポートへのリダイレクト (Redirect)

#### 問題
VLAN 30 を通過するクライアントからの HTTP (TCP 80) および HTTPS (TCP 443) トラフィックを、インライン型セキュリティセンサーが接続されているインターフェイス GigabitEthernet1/0/15 へ直接リダイレクトしてください。その他の通信は通常の L2/L3 スイッチングを行ってください。

#### 要件
* 宛先 TCP 80 および 443 への通信をリダイレクト対象とすること。
* リダイレクト先ポートは `GigabitEthernet1/0/15` とすること。
* Web 通信以外のトラフィックは影響を受けずに転送されること。

#### 設定例

```bash
! Step 1: Web トラフィックを指定する ACL
ip access-list extended ACL_WEB_TRAFFIC
 permit tcp any any eq 80
 permit tcp any any eq 443

ip access-list extended ACL_ALLOW_REST
 permit ip any any

! Step 2: Access Map 構成
vlan access-map MAP_WEB_REDIRECT 10
 match ip address ACL_WEB_TRAFFIC
 action redirect GigabitEthernet1/0/15

vlan access-map MAP_WEB_REDIRECT 20
 match ip address ACL_ALLOW_REST
 action forward

! Step 3: VLAN 30 にバインド
vlan filter MAP_WEB_REDIRECT vlan-list 30
```

#### 検証方法
1. VLAN 30 内のクライアントから 宛先 TCP 80 へ通信を発生させます。
2. スイッチの GigabitEthernet1/0/15 パケットキャプチャまたはインターフェイスパケットカウンター（`show interface GigabitEthernet1/0/15`）を確認し、出力パケットが増加していることを確認します。

---

### シナリオ 4: VLAN 内 TFTP 通信のドロップと Syslog ログ出力

#### 問題
VLAN 40 内で発生する TFTP 通信（UDP ポート 69）を L2 レベルで破棄し、ドロップされた事象をログに記録してください。

#### 要件
* 宛先 UDP ポート 69 の通信をブロックすること。
* ドロップ時にログ出力（`log` オプション）を行うこと。
* その他の通信はすべて許可すること。

#### 設定例

```bash
! Step 1: TFTP トラフィックを指定する ACL
ip access-list extended ACL_TFTP
 permit udp any any eq 69

ip access-list extended ACL_ALLOW_ALL
 permit ip any any

! Step 2: Access Map 構成（log オプション指定）
vlan access-map MAP_TFTP_BLOCK 10
 match ip address ACL_TFTP
 action drop log

vlan access-map MAP_TFTP_BLOCK 20
 match ip address ACL_ALLOW_ALL
 action forward

! Step 3: VLAN 40 にバインド
vlan filter MAP_TFTP_BLOCK vlan-list 40
```

#### 検証方法
1. VLAN 40 内の端末から TFTP クライアントを用いてファイル転送を試みます。
2. 通信がタイムアウトして失敗することを確認します。
3. スイッチのコンソールまたは Syslog サーバーに VACL ドロップログ（`%SEC-6-IPACCESSLOGP` 等）が出力されていることを確認します。

---

### シナリオ 5: 同一 VLAN 内マルチキャスト通信の限定フィルタリング

#### 問題
VLAN 50 内で特定のマルチキャストアプリケーション（239.1.1.10: UDP 5001）が帯域を圧迫しています。この特定マルチキャストグループ宛の通信のみを VLAN 内でドロップし、その他のマルチキャストおよびユニキャスト通信は許可してください。

#### 要件
* 宛先 IP `239.1.1.10` かつ UDP ポート `5001` のトラフィックを遮断すること。
* その他の通信はすべて転送すること。

#### 設定例

```bash
! Step 1: 特定マルチキャストトラフィック用 ACL
ip access-list extended ACL_MCAST_DROP
 permit udp any host 239.1.1.10 eq 5001

ip access-list extended ACL_PERMIT_ALL
 permit ip any any

! Step 2: Access Map 構成
vlan access-map MAP_MCAST_FILTER 10
 match ip address ACL_MCAST_DROP
 action drop

vlan access-map MAP_MCAST_FILTER 20
 match ip address ACL_PERMIT_ALL
 action forward

! Step 3: VLAN 50 にバインド
vlan filter MAP_MCAST_FILTER vlan-list 50
```

#### 検証方法
1. 送信元から 239.1.1.10:5001 宛に UDP パケットを送出し、同一 VLAN 内の受信側端末で受信できないことを確認します。
2. 別のマルチキャストアドレス（例: 239.1.1.20）への通信は正常に受信できることを確認します。

---

### シナリオ 6: 特定 IP サブネット間の同一 VLAN 内直接通信遮断

#### 問題
VLAN 60 は `/24` の大きなサブネット（10.1.60.0/24）で構成されていますが、開発グループ（10.1.60.0/25）と試験グループ（10.1.60.128/25）が共存しています。この 2 つのサブネット間の直接通信を VACL で遮断し、各グループからゲートウェイ（10.1.60.254）への通信は許可してください。

#### 要件
* 10.1.60.0/25 と 10.1.60.128/25 間の双方向通信をブロックすること。
* 各端末から 10.1.60.254 への通信は許可すること。

#### 設定例

```bash
! Step 1: サブネット間通信を指定する ACL
ip access-list extended ACL_SUBNET_ISOLATE
 permit ip 10.1.60.0 0.0.0.127 10.1.60.128 0.0.0.127
 permit ip 10.1.60.128 0.0.0.127 10.1.60.0 0.0.0.127

ip access-list extended ACL_ALLOW_OTHER
 permit ip any any

! Step 2: Access Map 構成
vlan access-map MAP_SUBNET_FILTER 10
 match ip address ACL_SUBNET_ISOLATE
 action drop

vlan access-map MAP_SUBNET_FILTER 20
 match ip address ACL_ALLOW_OTHER
 action forward

! Step 3: VLAN 60 にバインド
vlan filter MAP_SUBNET_FILTER vlan-list 60
```

#### 検証方法
1. 10.1.60.10 から 10.1.60.130 へ ping を実行し、失敗することを確認します。
2. 10.1.60.10 および 10.1.60.130 から 10.1.60.254 へ ping を実行し、共に成功することを確認します。

---

### シナリオ 7: 特定端末からの ICMP (Ping) のみブロックし他サービスは許可

#### 問題
VLAN 70 内の運用管理用端末 (10.1.70.50) からの ICMP トラフィック（Ping 等）のみを同一 VLAN 内の全端末宛てにブロックし、SSH (TCP 22) や HTTPS (TCP 443) などの管理プロトコル通信は正常に許可してください。

#### 要件
* 送信元 10.1.70.50 からの ICMP 通信をドロップすること。
* 10.1.70.50 からのその他の IP 通信（TCP/UDP）は正常に許可すること。

#### 設定例

```bash
! Step 1: ICMP 通信指定 ACL
ip access-list extended ACL_BLOCK_ICMP
 permit icmp host 10.1.70.50 any

ip access-list extended ACL_ALLOW_ALL
 permit ip any any

! Step 2: Access Map 構成
vlan access-map MAP_ICMP_RESTRICT 10
 match ip address ACL_BLOCK_ICMP
 action drop

vlan access-map MAP_ICMP_RESTRICT 20
 match ip address ACL_ALLOW_ALL
 action forward

! Step 3: VLAN 70 にバインド
vlan filter MAP_ICMP_RESTRICT vlan-list 70
```

#### 検証方法
1. 10.1.70.50 から同一 VLAN 内の別端末 10.1.70.100 へ `ping` を実行し、失敗することを確認します。
2. 10.1.70.50 から 10.1.70.100 へ SSH (TCP 22) 接続を試み、正常に接続できることを確認します。

---

### シナリオ 8: VACL と SVI 上の RACL の結合多層防御構成

#### 問題
VLAN 80 において、VLAN 内の端末相互間での NetBIOS 通信（TCP 139/445, UDP 137/138）を VACL で遮断し、さらに外部 VLAN から SVI 80 経由で流入する通信に対しては SSH のみを許可する RACL を設定してください。

#### 要件
* VACL を用いて VLAN 80 内の NetBIOS 関連ポートをすべてドロップすること。
* SVI インターフェイス（Interface Vlan80）に RACL を適用し、外部からのルーテッド通信は SSH (TCP 22) のみを許可すること。

#### 設定例

```bash
! === VACL 設定 (L2 セグメント保護) ===
ip access-list extended ACL_NETBIOS
 permit tcp any any eq 139
 permit tcp any any eq 445
 permit udp any any eq 137
 permit udp any any eq 138

ip access-list extended ACL_VLAN_PASS
 permit ip any any

vlan access-map MAP_VACL_80 10
 match ip address ACL_NETBIOS
 action drop

vlan access-map MAP_VACL_80 20
 match ip address ACL_VLAN_PASS
 action forward

vlan filter MAP_VACL_80 vlan-list 80

! === RACL 設定 (L3 境界保護) ===
ip access-list extended RACL_SVI80_IN
 permit tcp any 10.1.80.0 0.0.0.255 eq 22
 deny ip any any

interface Vlan80
 ip address 10.1.80.254 255.255.255.0
 ip access-group RACL_SVI80_IN in
```

#### 検証方法
1. VLAN 80 内の端末同士で SMB/NetBIOS 接続を試み、VACL によりブロックされることを確認します。
2. 外部 VLAN の端末から VLAN 80 内の端末宛に SSH 接続を試み、RACL 経由で許可されることを確認します。

---

### シナリオ 9: 複数 VLAN への同一 VACL の一括バインド (Multi-VLAN Filtering)

#### 問題
セキュリティポリシーにより、社内の複数のユーザー VLAN（VLAN 100, 101, 102）において、P2P ファイル共有プロトコル（BitTorrent: TCP/UDP 6881-6889）の通信を全拒否する必要があります。単一の VLAN Access Map を作成し、複数 VLAN へ一括適用してください。

#### 要件
* TCP/UDP ポート 6881-6889 の通信をドロップすること。
* 対象 VLAN は 100, 101, 102 とすること。
* 単一の `vlan filter` コマンドで一括バインドすること。

#### 設定例

```bash
! Step 1: P2P ポート指定 ACL
ip access-list extended ACL_P2P_PORTS
 permit tcp any any range 6881 6889
 permit udp any any range 6881 6889

ip access-list extended ACL_ALLOW_ALL
 permit ip any any

! Step 2: Access Map 構成
vlan access-map MAP_P2P_BLOCK 10
 match ip address ACL_P2P_PORTS
 action drop

vlan access-map MAP_P2P_BLOCK 20
 match ip address ACL_ALLOW_ALL
 action forward

! Step 3: 複数 VLAN への一括バインド
vlan filter MAP_P2P_BLOCK vlan-list 100,101,102
```

#### 検証方法
1. `show vlan filter` を実行し、`VLAN map MAP_P2P_BLOCK is configured on vlan 100,101,102` と表示されることを確認します。
2. VLAN 100、101、102 の各セグメントで対象ポートのパケットを送信し、正しく破棄されることを確認します。

---

### シナリオ 10: ARP 通信を阻害しない高度な VACL フィルタリング設計

#### 問題
VLAN 200 において、IP アドレス `10.1.200.66` を持つの非許可端末からの全 IP 通信を VACL で破棄してください。ただし、スイッチの DAI（Dynamic ARP Inspection）や ARP 正常動作に影響を与えないよう、ARP トラフィックは明示的に許可する設計を行ってください。

#### 要件
* 10.1.200.66 からの全 IP トラフィックをドロップすること。
* ARP 通信（非 IP）およびその他の端末の全通信は正常に通過させること。

#### 設定例

```bash
! Step 1: 遮断対象 IP ACL
ip access-list extended ACL_BLOCK_BAD_IP
 permit ip host 10.1.200.66 any
 permit ip any host 10.1.200.66

ip access-list extended ACL_ALLOW_IP
 permit ip any any

! ARP トラフィック許可用 MAC ACL
mac access-list extended MAC_ALLOW_ARP
 permit any any EtherType 0x0806 0x0000

! Step 2: Access Map 構成
vlan access-map MAP_SAFE_FILTER 10
 match ip address ACL_BLOCK_BAD_IP
 action drop

vlan access-map MAP_SAFE_FILTER 20
 match mac address MAC_ALLOW_ARP
 action forward

vlan access-map MAP_SAFE_FILTER 30
 match ip address ACL_ALLOW_IP
 action forward

! Step 3: VLAN 200 にバインド
vlan filter MAP_SAFE_FILTER vlan-list 200
```

#### 検証方法
1. 10.1.200.66 から他端末への IP ping は失敗することを確認します。
2. 10.1.200.66 が送信する ARP リクエスト/レスポンスは正常に通過し、MAC テーブルや ARP テーブルが更新されることを確認します。

---

## ❓ 想定試験問題

CCIE Security ラボ試験対策を意識した実践的な想定試験問題です。

---

### 問題 1（コンフィグ読解）

以下のコンフィグが適用されたスイッチにおいて、端末 10.1.10.50 から 10.1.10.100 宛の HTTP (TCP 80) 通信および SSH (TCP 22) 通信はどのように処理されますか？

```bash
ip access-list extended ACL_TEST_1
 deny tcp host 10.1.10.50 host 10.1.10.100 eq 80
 permit ip host 10.1.10.50 host 10.1.10.100

ip access-list extended ACL_TEST_2
 permit ip any any

vlan access-map MAP_TEST 10
 match ip address ACL_TEST_1
 action drop

vlan access-map MAP_TEST 20
 match ip address ACL_TEST_2
 action forward

vlan filter MAP_TEST vlan-list 10
```

#### 選択肢 / 質問
1. TCP 80 通信および TCP 22 通信の処理結果をそれぞれ答えよ。
2. なぜその処理結果になるのか理由を説明せよ。

#### 解答と解説
1. **処理結果**:
   - HTTP (TCP 80) 通信: **許可（Forward）** されます。
   - SSH (TCP 22) 通信: **破棄（Drop）** されます。
2. **理由解説**:
   - HTTP (TCP 80) 通信が発生した場合、まず Sequence 10 の `ACL_TEST_1` が評価されます。ACL 内の最初の行 `deny tcp ... eq 80` に一致するため、この ACL においては **「不一致 (No Match)」** と判定され、Sequence 10 の `action drop` は適用されず、次の Sequence 20 へ評価が移行します。Sequence 20 の `ACL_TEST_2` (`permit ip any any`) に一致するため、`action forward` が適用されて許可されます。
   - SSH (TCP 22) 通信が発生した場合、Sequence 10 の `ACL_TEST_1` の 2 行目 `permit ip host 10.1.10.50 host 10.1.10.100` に一致します。ACL で `permit` になったため、Sequence 10 の `action drop` が適用され、即座に破棄されます。

---

### 問題 2（トラブルシューティング）

管理者から「VLAN 30 に VACL を追加した直後、VLAN 30 内の全端末が外部インターネットとも同一 VLAN 内の他端末とも一切通信できなくなった」と報告がありました。設定されたコンフィグは以下の通りです。原因と正しい修正コンフィグを提示してください。

```bash
ip access-list extended ACL_BLOCK_SMB
 permit tcp any any eq 445

vlan access-map VACL_NO_SMB 10
 match ip address ACL_BLOCK_SMB
 action drop

vlan filter VACL_NO_SMB vlan-list 30
```

#### 解答と解説
* **原因**:
  - `vlan access-map` の末尾には暗黙の `drop` 条件（Implicit Drop）が存在します。上記設定では Sequence 10 で TCP 445 のみを一致させてドロップしていますが、それ以外のすべてのパケット（その他の IP 通信、ARP 通信等）に対して許可するシーケンスが存在しないため、末尾の暗黙の drop により全通信が破棄されています。
* **修正コンフィグ**:

```bash
ip access-list extended ACL_ALLOW_ALL
 permit ip any any

vlan access-map VACL_NO_SMB 20
 match ip address ACL_ALLOW_ALL
 action forward
```

---

### 問題 3（Design / 実装）

VLAN 50 に存在する特定サーバー (192.168.50.10) に対し、同一 VLAN 内のクライアント端末群からのアクセスを制限します。以下の要件を満たす VACL を設定してください。

* クライアントから 192.168.50.10 への HTTPS (TCP 443) 通信のみを許可すること。
* 192.168.50.10 へのその他の全通信（HTTP, ICMP 等）は同一 VLAN 内でドロップすること。
* 他の端末同士の通信、およびデフォルトゲートウェイとの通信は一切妨げないこと。

#### 解答と解説
* **設定コンフィグ**:

```bash
ip access-list extended ACL_PERMIT_HTTPS
 permit tcp any host 192.168.50.10 eq 443
 permit tcp host 192.168.50.10 eq 443 any

ip access-list extended ACL_DROP_SERVER
 permit ip any host 192.168.50.10
 permit ip host 192.168.50.10 any

ip access-list extended ACL_PERMIT_REST
 permit ip any any

vlan access-map MAP_SERVER_SEC 10
 match ip address ACL_PERMIT_HTTPS
 action forward

vlan access-map MAP_SERVER_SEC 20
 match ip address ACL_DROP_SERVER
 action drop

vlan access-map MAP_SERVER_SEC 30
 match ip address ACL_PERMIT_REST
 action forward

vlan filter MAP_SERVER_SEC vlan-list 50
```

---

### 問題 4（トラブルシューティング）

特定 VLAN (VLAN 100) を通過する HTTP トラフィックを別ポート (GigabitEthernet1/0/24) にリダイレクトするため以下の VACL を設定しましたが、リダイレクト先ポートでパケットが受信されず、通信自体が切断されます。原因として考えられる理由と確認コマンドを挙げてください。

```bash
ip access-list extended ACL_HTTP
 permit tcp any any eq 80

vlan access-map MAP_REDIRECT 10
 match ip address ACL_HTTP
 action redirect GigabitEthernet1/0/24
```

#### 解答と解説
* **原因**:
  1. インターフェイス `GigabitEthernet1/0/24` が Down 状態である場合、VACL リダイレクト先が無効となりパケットが破棄されます。
  2. 対象ポートが EtherChannel の一部として構成されている場合、リダイレクト機能が制限される可能性があります。
  3. リダイレクト対象の Access Map に全通信許可用の `action forward` シーケンスがなく、暗黙の drop が適用されている可能性。
* **確認コマンド**:
  - `show interface GigabitEthernet1/0/24 status`
  - `show vlan access-map MAP_REDIRECT`

---

### 問題 5（コンフィグ読解 - パケット処理順序）

Ingress ポートから受信したパケットが、同一 VLAN 内の別ポートへスイッチング（Bridged）される場合と、SVI 経由で別 VLAN へルーティング（Routed）される場合において、PACL, VACL, RACL の評価順序をそれぞれ正しく並べ替えてください。

#### 解答と解説
1. **同一 VLAN 内スイッチング（Bridged Traffic）**:
   - **Ingress PACL ──► Ingress VACL ──► Egress VACL**
2. **別 VLAN へルーティング（Routed Traffic）**:
   - **Ingress PACL ──► Ingress VACL ──► Ingress RACL ──► Routing Lookup ──► Egress RACL ──► Egress VACL**

---

### 問題 6（実装）

VLAN 150 において、特定の端末の MAC アドレス (`0011.2233.4455`) からのパケットを全破棄し、それ以外の正常端末の IP 通信および ARP 通信を維持するコンフィグを作成してください。

#### 解答と解説
* **設定コンフィグ**:

```bash
mac access-list extended MAC_BLOCK_SPECIFIC
 permit host 0011.2233.4455 any
 permit any host 0011.2233.4455

ip access-list extended ACL_PASS_IP
 permit ip any any

mac access-list extended MAC_PASS_ALL
 permit any any

vlan access-map MAP_BLOCK_MAC 10
 match mac address MAC_BLOCK_SPECIFIC
 action drop

vlan access-map MAP_BLOCK_MAC 20
 match ip address ACL_PASS_IP
 action forward

vlan access-map MAP_BLOCK_MAC 30
 match mac address MAC_PASS_ALL
 action forward

vlan filter MAP_BLOCK_MAC vlan-list 150
```

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび技術資料の一覧です。

* Cisco Live: BRKSEC-2003 - Layer 2 Security Features and Best Practices
* Cisco Live: BRKARC-2007 - Catalyst 9000 Switching Architecture
* Catalyst 9300 Series Switches Security Configuration Guide: Configuring VLAN ACLs (VACLs)
* Cisco Command Reference: vlan access-map Command
* Cisco Command Reference: vlan filter Command
* Technical Note: Configuring VLAN ACLs (VACLs) on Catalyst Switches
* Cisco Validated Design (CVD): Campus LAN and Wireless LAN Design Guide

---

## 📝 補足（Notes）

### VACL 設定時のトラブル予防チェックリスト
1. **末尾に `action forward` シーケンスを追加したか？** （暗黙の drop による VLAN 全体障害を防ぐ）
2. **ACL の条件行は `permit` になっているか？** （`deny` と書くと Access Map 上で「不一致」となりスルーされる）
3. **ARP パケットを通す考慮はされているか？** （非 IP MAC ACL や明示的な IP パスで ARP 阻害を防ぐ）
4. **リダイレクト先インターフェイスは UP 状態か？** （Down ポートを指定するとトラフィックがドロップする）
5. **本番適用前に `show vlan access-map` でシーケンス番号とアクションを確認したか？**

---

