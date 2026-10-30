---
layout: default
title: 1.3.c-NAT
nav_order: 3
parent: 1.3-Security-features-on-IOS
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.3.c NAT

Cisco IOS および Cisco IOS XE ソフトウェアにおける **Network Address Translation (NAT)** は、IP パケットのヘッダー内の送信元または宛先 IP アドレス（および L4 ポート番号）をルータ上で動的・静的に変換する不可欠な IP サービス機能です。

RFC 1918 プライベート IP アドレス空間の隠蔽、インターネット接続用のパブリック IP アドレス節約（PAT / NAT Overload）、企業統合（M&A）等に伴う重畳（Overlapping）IP アドレス空間の相互接続、さらには VRF-Aware NAT や HSRP / HA 構成における冗長化など、CCIE Security v6.1 ラボ試験では極めて広範かつ正確な実装能力が評価されます。

---

## 📘 概要

* **機能概要**:
  * **Static NAT / Static PAT**: 内部ローカル IP アドレス（および特定の L4 ポート）と外部グローバル IP アドレス（および特定の L4 ポート）を 1 対 1 で恒久的にマッピングし、双方向からの接続開始を許可します。
  * **Dynamic NAT**: 送信元 IP アドレスを、あらかじめ定義されたパブリック IP アドレスプールから先着順（First-Come, First-Served）で動的に割り当てます（単方向通信）。
  * **Dynamic PAT (NAT Overload)**: 多数の内部 IP アドレスを単一のパブリック IP アドレス（またはインターフェイス IP）にマッピングし、一意の送信元 L4 ポート番号を用いて多重化（Multiplexing）します。
  * **Policy NAT**: Route-Map や拡張 ACL（Extended ACL）を用いて、送信元 IP だけでなく宛先 IP アドレス、使用プロトコル、L4 ポート番号などの条件に合致したトラフィックのみを条件付きで変換します。
  * **NAT Virtual Interface (NVI)**: 従来の `ip nat inside` / `ip nat outside` による方向依存を排除し、`ip nat enable` をインターフェイスに適用することでルーティング決定後に統一されたロジックでアドレス変換を実行します（IOS 専用）。
  * **VRF-Aware NAT**: MPLS VPN や VRF-Lite 環境において、異なる VRF（IVRF / FVRF）間でのトラフィック変換をサポートします。
* **利用目的**:
  * IPv4 パブリック アドレス枯渇問題への対応および内部ネットワーク構成の隠蔽（セキュリティ確保）。
  * 異なる組織間（M&A やパートナー接続）の重複 IP アドレス空間同士の相互通信確立。
  * IPsec VPN パス上におけるトラフィックの NAT 除外（NAT Bypass / NAT Exemption）の適用。
* **どのような場面で利用するか**:
  * 企業 WAN ルータにおける Direct Internet Access (DIA) ポイントでの動的 PAT。
  * DMZ 上の Web / Mail サーバーを外部インターネットへ公開するための Static PAT (Port Forwarding)。
  * 重複するネットワークサブネット（例: 両社とも `10.1.1.0/24`）を統合する拠点間ルーティング接続。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **変換タイプ** | Static NAT (1:1), Dynamic NAT (M:N), Dynamic PAT (M:1), Policy NAT, Identity NAT |
| **アーキテクチャ** | 従来型 Inside/Outside モード (`ip nat inside/outside`)、NVI モード (`ip nat enable`) |
| **適用プレーン** | Data Plane (CEF 転送パイプラインと直接統合) |
| **デフォルト動作** | NAT テーブル（xlate）に該当しないパケットは未変換のままルーティング・送出される |
| **対応機種** | Cisco ISR 1000/4000, ASR 1000, CSR 1000v, Catalyst 8000, Cisco IOS / IOS XE 全般 |
| **制限事項** | IOS XE では NVI (NAT Virtual Interface) が非サポート。Zone-Based Firewall (ZBFW) との併用制限あり |
| **設計上の注意点** | Ingress パイプラインと Egress パイプラインでのルーティング決定と NAT 評価順序（Order of Operation）の完全理解が必須 |

---

## 🏗 動作原理

Cisco IOS / IOS XE の従来型 NAT アーキテクチャは、インターフェイスを `inside`（内部ネットワーク）と `outside`（外部ネットワーク）に区別し、パケットの進行方向に応じてアドレス変換とルーティングの順番を切り替えます。

```text
【Inside ➔ Outside 通信 (例: LAN から Internet へのアクセス)】

[ Client: 10.1.1.10 ]
         │
         │ (1) Ingress ACL Check
         ▼
[ Ingress Interface: ip nat inside ]
         │
         │ (2) Routing Decision (CEF Lookup for Destination)
         ▼
[ NAT Engine: Inside -> Outside Translation ]
         │   - Source IP: 10.1.1.10  ──► Translated to 203.0.113.1 (Global IP)
         │   - Create Entry in xlate Table
         ▼
[ Egress Interface: ip nat outside ]
         │
         │ (3) Egress ACL Check / IPsec Encryption
         ▼
[ Internet / Server: 8.8.8.8 ]

【Outside ➔ Inside 通信 (例: Internet から DMZ サーバーへのアクセス)】

[ External Client: 198.51.100.5 ]
         │
         │ (1) Ingress ACL Check
         ▼
[ Ingress Interface: ip nat outside ]
         │
         │ (2) NAT Engine: Outside -> Inside Translation
         │   - Destination IP: 203.0.113.10 ──► Translated to 10.1.1.100 (Real DMZ IP)
         │   - Look up xlate / Static Mapping
         ▼
[ Routing Decision (CEF Lookup for Translated Real IP 10.1.1.100) ]
         │
         ▼
[ Egress Interface: ip nat inside ]
         │
         │ (3) Egress ACL Check
         ▼
[ DMZ Web Server: 10.1.1.100 ]
```

---

## ⚙ 動作シーケンス

Cisco IOS / IOS XE におけるパケット処理の評価順序（Order of Operation）は、トラフィックの方向によって決定的に異なります。

### 1. Inside ➔ Outside パケット処理シーケンス

1. **Inbound ACL チェック**: 着信インターフェイス (`ip nat inside`) のインバウンド ACL を評価します。
2. **ルーティング決定 (Routing Lookup)**: 宛先 IP アドレスに基づいて CEF テーブルを参照し、送信出力を決定します。**（※ 転送先が決まらない場合は NAT 変換されずにドロップされます）**
3. **Policy-Based Routing (PBR)**: PBR が設定されている場合は優先評価されます。
4. **Inside ➔ Outside NAT 変換**: 送信元 IP アドレスを NAT ルールに従って変換し、`xlate` テーブルにセッションを登録/更新します。
5. **Outbound ACL チェック**: 送出インターフェイス (`ip nat outside`) のアウトバウンド ACL を評価します**（※ 変換後の IP アドレスを基準にチェックします）**.
6. **IPsec 暗号化 / パケット送出**: トンネル保護が定義されている場合は暗号化して送出します。

### 2. Outside ➔ Inside パケット処理シーケンス

1. **Inbound ACL チェック**: 着信インターフェイス (`ip nat outside`) のインバウンド ACL を評価します**（※ 変換前の Mapped Destination IP アドレスを基準にチェックします）**.
2. **IPsec 復号**: 暗号化パケットの場合、復号処理を行います。
3. **Outside ➔ Inside NAT 変換**: 宛先 IP アドレスを変換テーブルに従って内部リアル IP アドレスに置換します。
4. **ルーティング決定 (Routing Lookup)**: **変換後の Real IP アドレス** に基づいて CEF テーブルを参照し、内部宛先インターフェイスを決定します。
5. **Outbound ACL チェック**: 送出インターフェイス (`ip nat inside`) のアウトバウンド ACL を評価します。
6. **パケット送出**: 内部クライアント/サーバーへ送信します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **IPsec VPN 通信時の NAT バイパス (NAT Exemption / Policy NAT)**:
   * 拠点間 (Site-to-Site) IPsec VPN や DMVPN / FlexVPN を構築する際、VPN 対向へのトラフィックが通常のダイナミック PAT ルールに誤ってマッチし、暗号化される前にパブリック IP へ変換されてトンネルが不通になるトラブルが頻出します。
   * **解決策**: Route-Map を用いた Dynamic PAT (`ip nat inside source route-map <NAME> interface <IF> overload`) を構成し、Route-Map 内の Access-List で **VPN トラフィックを `deny`**、**インターネット宛てトラフィックを `permit`** とすることで、VPN トラフィックを安全に NAT 除外させます。
2. **`no-alias` キーワードの用途とトラブル回避**:
   * IOS / IOS XE で `ip nat inside source static <LOCAL_IP> <GLOBAL_IP>` を投入すると、ルータはデフォルトで `<GLOBAL_IP>` に対する **Proxy ARP エントリを自動生成** します。
   * グローバル IP がルータ自身の WAN インターフェイスと同一サブネット上に存在しない場合や、別ルータがその IP の ARP に応答すべき環境では、ルータが不正に Proxy ARP 応答を返してルーティング競合を引き起こします。
   * 問題文で「ルータが該当エイリアス IP に対して Proxy ARP に応答しないようにせよ」と指定された場合は、静的設定の末尾に **`no-alias`** を必ず付与します。
3. **Route-Map 内の ACL における `permit` と `deny` の特殊な意味**:
   * NAT で使用する Route-Map において、`match ip address <ACL>` に指定した ACL の **`deny` 行は「NAT 変換を行わない (Bypass)」** ことを意味し、**`permit` 行は「NAT 変換を実行する」** ことを意味します。パケット自体がドロップされるわけではありません。
4. **VRF-Aware NAT の方向とプロトコル指定**:
   * VRF-Lite 環境において、特定の VRF 内のホストをグローバル VRF 経由でインターネットへ抜けるようにする場合、`ip nat inside source list <ACL> interface <IF> overload vrf <VRF-NAME>` や `match-in-vrf` オプションを使用します。
5. **IOS XE (CSR1000v / ASR1000 / Cat8000) における NVI 非サポート**:
   * 試験問題で IOS XE アプライアンスが指定されている場合、`ip nat enable` (NVI) コマンドはサポートされておらず、投入すると構文エラーとなります。必ず従来型の `ip nat inside` / `ip nat outside` を使用しなければなりません。

---

## 🛠 設定方法

### 1. Cisco IOS / IOS XE: 基本的な Dynamic PAT (Interface Overload) 設定

```bash
! ===== 1. インサイド / アウトサイド インターフェイスの指定 =====
interface GigabitEthernet0/0/1
 description LAN_INSIDE
 ip address 10.1.1.254 255.255.255.0
 ip nat inside

interface GigabitEthernet0/0/2
 description WAN_OUTSIDE
 ip address 203.0.113.1 255.255.255.252
 ip nat outside

! ===== 2. NAT 対象トラフィックの ACL 定義 =====
ip access-list standard ACL_NAT_SOURCE
 permit 10.1.1.0 0.0.0.255

! ===== 3. Dynamic PAT のバインド =====
ip nat inside source list ACL_NAT_SOURCE interface GigabitEthernet0/0/2 overload
```

### 2. Cisco IOS / IOS XE: Route-Map を使用した IPsec VPN バイパス付き Policy NAT 設定

```bash
! ===== 1. ACL で VPN 通信 (10.1.1.0/24 -> 10.2.2.0/24) と Internet 通信を分離 =====
ip access-list extended ACL_NAT_POLICY
 ! VPN トラフィックは NAT 除外 (deny)
 deny ip 10.1.1.0 0.0.0.255 10.2.2.0 0.0.0.255
 ! インターネット宛てトラフィックは NAT 実行 (permit)
 permit ip 10.1.1.0 0.0.0.255 any

! ===== 2. Route-Map の構成 =====
route-map RM_NAT_POLICY permit 10
 match ip address ACL_NAT_POLICY

! ===== 3. Route-Map を用いた Dynamic PAT の適用 =====
ip nat inside source route-map RM_NAT_POLICY interface GigabitEthernet0/0/2 overload
```

### 3. Cisco IOS / IOS XE: Static PAT (Port Forwarding) ＆ Proxy ARP 無効化 (`no-alias`)

```bash
! DMZ サーバー (10.1.1.100:80) を WAN アドレス (203.0.113.10:8080) に公開
! Proxy ARP 応答を無効化 (no-alias)
ip nat inside source static tcp 10.1.1.100 80 203.0.113.10 8080 extendable no-alias
```

### 4. Cisco IOS (Legacy): NVI (NAT Virtual Interface) 設定 (CLI)

```bash
! NVI では inside / outside を指定せず ip nat enable を適用
interface GigabitEthernet0/0/1
 description LAN
 ip address 10.1.1.254 255.255.255.0
 ip nat enable

interface GigabitEthernet0/0/2
 description WAN
 ip address 203.0.113.1 255.255.255.252
 ip nat enable

! NAT ステートメントから inside/outside キーワードを除外
ip access-list standard ACL_NVI_SRC
 permit 10.1.1.0 0.0.0.255

ip nat source list ACL_NVI_SRC interface GigabitEthernet0/0/2 overload
```

---

## 🔍 検証コマンド

| 目的 | コマンド | 表示確認ポイント |
| :--- | :--- | :--- |
| **アクティブな NAT 変換テーブル（xlate）の表示** | <code>show ip nat translations</code> | Inside/Outside Local/Global IP、L4 ポート番号、プロトコル |
| **NAT 変換の詳細情報（タイムアウト値、フラグ）表示** | <code>show ip nat translations verbose</code> | セッションのアイドル時間、作成からの経過時間、xlate ID |
| **NAT 統計情報（変換数、ミス数、プール使用率）確認** | <code>show ip nat statistics</code> | インサイド/アウトサイド定義インターフェイス、ヒット数、エラー |
| **NVI アクティブ変換テーブルの表示** | <code>show ip nat nvi translations</code> | NVI インターフェイス経由でのアドレス変換エントリ |
| **NAT 変換テーブルのクリア（特定 IP または全消去）** | <code>clear ip nat translation *</code><br><code>clear ip nat translation inside <LOCAL_IP> <GLOBAL_IP></code> | ダイナミック NAT/PAT テーブルの即時消去 |
| **NAT パケット処理のリアルタイムデバッグ** | <code>debug ip nat</code> / <code>debug ip nat detailed</code> | `s=10.1.1.10->203.0.113.1, d=8.8.8.8` 等のアドレス変換ログ |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **NAT 設定後にクライアントから外部へ疎通できない** | 宛先へのルーティングが存在しないため、Inside ➔ Outside 評価プロセスで NAT 処理前にドロップされている | <code>show ip route</code><br><code>show ip nat statistics</code> | デフォルトルート等の宛先経路が正しく存在するか確認する。 |
| **IPsec トンネルが確立せず、暗号化パケットが送出されない** | VPN トラフィックが Dynamic PAT にキャッチされてパブリック IP に変換されている | <code>show ip nat translations</code> | Route-Map または ACL で VPN トラフィックを `deny`（NAT 除外）に設定する。 |
| **`ip nat inside source list` を設定したのに変換されない** | コマンド末尾の `overload` キーワードが欠落している（パケットプール枯渇） | <code>show ip nat statistics</code> | ポート多重化を行う場合はコマンド末尾に `overload` を付与する。 |
| **ルータが他装置の IP アドレスに対して不正に ARP 応答を返してしまう** | Static NAT 設定によりルータが自動的に Proxy ARP エントリ（Alias）を生成している | <code>show ip alias</code> | 静的 NAT コマンドの末尾に `no-alias` キーワードを追加する。 |
| **IOS XE ルータで `ip nat enable` を投入すると構文エラーになる** | IOS XE ソフトウェアでは NVI (NAT Virtual Interface) が未サポート | <code>show version</code> | `ip nat inside` および `ip nat outside` の従来型設定に移行する。 |
| **Outside ➔ Inside の静的 PAT でインバウンド通信が届かない** | `ip nat outside` インターフェイスのインバウンド ACL で **変換前の Destination IP** を許可していない | <code>show access-list</code> | インバウンド ACL で Mapped Global IP アドレス宛ての通信を許可する。 |

---

## ⚠ 制限事項

* **Cisco IOS XE における機能制限**:
  * **NVI (NAT Virtual Interface) 非サポート**: CSR1000v, ASR1000, ISR4000 等の IOS XE プラットフォームでは NVI が完全に削除されています。
  * **ACL における `permit ip any any` の非推奨/制限**: NAT ACL 内で `permit ip any any` を使用すると想定外のローカル制御通信が変換されるため禁止されています。
* **Zone-Based Policy Firewall (ZBFW) との非互換性・共存制限**:
  * 同一インターフェイス上で ZBFW と NVI を併用することはできません。
* **セッション数とメモリ制限**:
  * Dynamic PAT は単一 IP アドレスあたり最大約 64,512 個の同時 TCP/UDP セッション（ポート数）に制限されます。

---

## 🔄 他技術との関連

* **1.3.b Zone-Based Policy Firewall (ZBFW)**: ZBFW と NAT を同時適用する場合、パケットフロー順序（Ingress NAT ➔ ZBFW ➔ Egress NAT）を把握し、ZBFW ACL には **Real (変換前) IP アドレス** を記述します。
* **IPsec VPN / FlexVPN / DMVPN**: トンネル保護を行う通信は NAT 変換より前に識別されるか、Route-Map で明確に NAT バイパス（NAT Exemption）する必要があります。
* **VRF (Virtual Routing and Forwarding)**: `ip nat inside source ... vrf <NAME> match-in-vrf` コマンドを使用して、マルチテナント環境の VRF 間アドレス変換を実現します。
* **HSRP / VRRP (Hot Standby Router Protocol)**: HSRP 仮想 IP アドレスを NAT 変換アドレスとして利用する場合や、アクティブ/スタンバイのルータ間で NAT 状態テーブルを同期させる Stateful NAT (SNAT) を構成します。

---

## 🧩 比較表

### 1. NAT 変換タイプの比較

| 項目 | Static NAT | Dynamic NAT | Dynamic PAT (Overload) |
| :--- | :--- | :--- | :--- |
| **アドレス変換比率** | **1 対 1** (固定) | **M 対 N** (動的プール) | **M 対 1** (L4 ポート多重化) |
| **通信開始方向** | **双方向可能** (Inbound / Outbound) | **単方向のみ** (Outbound 発信) | **単方向のみ** (Outbound 発信) |
| **パブリック IP 消費量** | 高い (ホスト毎に 1 IP 必要) | 中程度 (同時に通信するホスト数分) | **極めて低い** (1 IP で数万セッション) |
| **主な用途** | DMZ Web/Mail サーバー公開 | 特別な一対一動的割り当て環境 | **社内 クライアント端末の Web アクセス** |

### 2. Legacy NAT (Inside/Outside) vs NVI (NAT Virtual Interface)

| 比較項目 | Legacy NAT (`ip nat inside/outside`) | NVI (`ip nat enable`) |
| :--- | :--- | :--- |
| **インターフェイス設定** | `ip nat inside` または `ip nat outside` | **`ip nat enable`** (方向指定なし) |
| **ルーティング依存度** | 通信方向によりルーティングと NAT の順番が変わる | **常にルーティング決定後に NAT が実行される** |
| **静的ルート要件** | インサイド/アウトサイドの両方向静的ルートが必要な場合あり | 単一のルーティングテーブルで一貫処理 |
| **IOS XE サポート** | **完全サポート** | **非サポート** (IOS のみ対応) |

---

## 💡 ベストプラクティス

1. **Route-Map と Named ACL を用いた明確な Policy NAT 設計**:
   * 今後の標準化と可読性のため、番号付き ACL ではなく **拡張名前付き ACL (Extended Named ACL)** と **Route-Map** を組み合わせた Policy NAT 構成を採用します。
2. **`no-alias` キーワードの標準適用**:
   * 自装置の WAN インターフェイス IP 以外の IP を Static NAT / Static PAT で使用する場合、不要な Proxy ARP 応答によるネットワークトラブルを防ぐため、常に `no-alias` を付与します。
3. **`show ip nat statistics` による定期的なミスカウント監視**:
   * パケットプール枯渇エラー（`allocation failures`）が発生していないかを `show ip nat statistics` で監視し、必要に応じて PAT プール IP を増設します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Static 1-to-1 NAT と Proxy ARP 無効化 (`no-alias`)

#### 問題
ルータ R1 において、内部サーバー `10.1.1.100` を外部グローバル IP `203.0.113.100` へ 1 対 1 で静的マッピングせよ。ルータ R1 自装置が `203.0.113.100` に対する Proxy ARP 応答を返さないよう設定せよ。

#### 要件
* インサイド: `GigabitEthernet0/0/1`, アウトサイド: `GigabitEthernet0/0/2`
* `no-alias` キーワードを使用すること。

#### 設定例

```bash
interface GigabitEthernet0/0/1
 ip nat inside

interface GigabitEthernet0/0/2
 ip nat outside

ip nat inside source static 10.1.1.100 203.0.113.100 no-alias
```

#### 検証方法
* `show ip nat translations` を実行し、`10.1.1.100` と `203.0.113.100` の Static マッピングを確認。
* `show ip alias` を実行し、`203.0.113.100` がエイリアス登録されていない（Proxy ARP を返さない）ことを確認。

---

### 例2: Dynamic PAT (Interface Overload と拡張 ACL)

#### 問題
内部サブネット `10.1.0.0/16` からの通信を、ルータ R1 の WAN インターフェイス `GigabitEthernet0/0/2` の IP アドレスへ Dynamic PAT 変換せよ。ただし、社内管理ネットワーク `10.1.254.0/24` からの通信は NAT 変換対象から除外せよ。

#### 要件
* 拡張 ACL を使用すること。

#### 設定例

```bash
interface GigabitEthernet0/0/1
 ip nat inside

interface GigabitEthernet0/0/2
 ip nat outside

ip access-list extended ACL_PAT_FILTER
 deny ip 10.1.254.0 0.0.0.255 any
 permit ip 10.1.0.0 0.0.255.255 any

ip nat inside source list ACL_PAT_FILTER interface GigabitEthernet0/0/2 overload
```

#### 検証方法
* `10.1.1.10` から外部へ Ping を打ち、`show ip nat translations` で WAN IP に PAT 変換されていることを確認。
* `10.1.254.10` からの通信が xlate テーブルに生成されないことを確認。

---

### 例3: Dynamic NAT Pool と PAT フォールバック (Pool Overload)

#### 問題
内部ユーザー `10.2.0.0/24` に対し、パブリック IP プール `198.51.100.10` 〜 `198.51.100.20` から先着順で Dynamic NAT 変換せよ。IP プールが満杯になった場合は、動的 PAT へ自動フォールバック（Overload）させよ。

#### 要件
* `ip nat pool` に `overload` キーワードを組み合わせること。

#### 設定例

```bash
interface GigabitEthernet0/0/1
 ip nat inside

interface GigabitEthernet0/0/2
 ip nat outside

ip access-list standard ACL_POOL_NAT
 permit 10.2.0.0 0.0.0.255

ip nat pool NAT_POOL_198 198.51.100.10 198.51.100.20 netmask 255.255.255.0

ip nat inside source list ACL_POOL_NAT pool NAT_POOL_198 overload
```

#### 検証方法
* `show ip nat statistics` を実行し、`Pool NAT_POOL_198` の割り当て状態および Overload 動作を確認。

---

### 例4: Static PAT (Port Forwarding / 非標準ポートマッピング)

#### 問題
外部ユーザーがルータ R1 の WAN アドレス `203.0.113.1` の TCP ポート `2222` へアクセスした際、内部 DMZ 上の Linux サーバー `10.1.1.50` の SSH ポート (TCP `22`) へポートフォワーディングせよ。

#### 要件
* `extendable` キーワードを使用すること。

#### 設定例

```bash
interface GigabitEthernet0/0/1
 ip nat inside

interface GigabitEthernet0/0/2
 ip nat outside

ip nat inside source static tcp 10.1.1.50 22 interface GigabitEthernet0/0/2 2222 extendable
```

#### 検証方法
* 外部から `ssh -p 2222 203.0.113.1` を実行。
* `show ip nat translations` で `tcp 203.0.113.1:2222 (10.1.1.50:22)` の静的 PAT エントリがアクティブになることを確認。

---

### 例5: Policy NAT (Route-Map による IPsec VPN バイパス ＆ インターネット PAT)

#### 問題
ルータ R1 において、対向拠点ルータ R2 (`10.2.0.0/24`) 宛ての IPsec VPN トラフィックを NAT 変換から除外（Bypass）し、それ以外の全インターネット宛てトラフィックを WAN ポート `Gig0/0/2` へ Dynamic PAT 変換せよ。

#### 要件
* Route-Map を使用すること。

#### 設定例

```bash
interface GigabitEthernet0/0/1
 ip nat inside

interface GigabitEthernet0/0/2
 ip nat outside

! 1. VPN 通信を deny、インターネット通信を permit に設定
ip access-list extended ACL_POLICY_NAT
 deny ip 10.1.1.0 0.0.0.255 10.2.0.0 0.0.0.255
 permit ip 10.1.1.0 0.0.0.255 any

! 2. Route-Map の定義
route-map RM_NAT_EXEC permit 10
 match ip address ACL_POLICY_NAT

! 3. Policy NAT の適用
ip nat inside source route-map RM_NAT_EXEC interface GigabitEthernet0/0/2 overload
```

#### 検証方法
* `10.1.1.10` から `10.2.0.10` 宛てに Ping を送信し、IPsec トンネル経由で到達すること、および `show ip nat translations` にエントリが作成されないことを確認。
* `8.8.8.8` 宛てに Ping を送信し、xlate テーブルに PAT 変換が生成されることを確認。

---

### 例6: VRF-Aware Dynamic PAT (IVRF から FVRF へのアドレス変換)

#### 問題
ルータ R1 において、内部 VRF `CUSTOMER_A` に属するインターフェイス `GigabitEthernet0/0/1` からの通信を、グローバルルーティングテーブル (FVRF) に属する WAN ポート `GigabitEthernet0/0/2` 経由で動的 PAT 変換せよ。

#### 要件
* `vrf` 指定および `match-in-vrf` オプションを使用すること。

#### 設定例

```bash
vrf definition CUSTOMER_A
 address-family ipv4
 exit-address-family

interface GigabitEthernet0/0/1
 vrf forwarding CUSTOMER_A
 ip address 10.100.1.254 255.255.255.0
 ip nat inside

interface GigabitEthernet0/0/2
 ip address 203.0.113.1 255.255.255.252
 ip nat outside

ip access-list standard ACL_VRF_SRC
 permit 10.100.1.0 0.0.0.255

ip nat inside source list ACL_VRF_SRC interface GigabitEthernet0/0/2 vrf CUSTOMER_A overload
```

#### 検証方法
* `VRF CUSTOMER_A` 内の端末から Ping を実行し、`show ip nat translations vrf CUSTOMER_A` で VRF 紐付きのアドレス変換が表示されることを確認。

---

### 例7: NVI (NAT Virtual Interface) 静的 ＆ 動的統合設定 (IOS)

#### 問題
Cisco IOS ルータ R1 において、従来の Inside/Outside 方向指定を使用せず、NVI (`ip nat enable`) を使用して内部 PC `10.1.1.10` を `203.0.113.10` へ静的 1:1 NAT 変換し、その他の LAN 端末を `Gig0/0/2` へ Dynamic PAT 変換せよ。

#### 要件
* `ip nat enable` を使用すること。

#### 設定例

```bash
interface GigabitEthernet0/0/1
 ip address 10.1.1.254 255.255.255.0
 ip nat enable

interface GigabitEthernet0/0/2
 ip address 203.0.113.1 255.255.255.252
 ip nat enable

ip access-list standard ACL_NVI_PAT
 permit 10.1.1.0 0.0.0.255

! NVI 静的 NAT
ip nat source static 10.1.1.10 203.0.113.10

! NVI 動的 PAT
ip nat source list ACL_NVI_PAT interface GigabitEthernet0/0/2 overload
```

#### 検証方法
* `show ip nat nvi translations` を実行し、NVI 変換テーブルが生成されていることを確認。

---

### 例8: Dual WAN / Multi-ISP 環境における Route-Map 連携 Policy NAT

#### 問題
ルータ R1 が ISP1 (`Gig0/0/2`) および ISP2 (`Gig0/0/3`) のマルチホーム構成となっている。内部端末 `10.1.1.0/24` からの通信が出力インターフェイスに応じて適切な WAN アドレスへ Dynamic PAT 変換されるよう構成せよ。

#### 要件
* 2 つの Route-Map を定義すること。

#### 設定例

```bash
interface GigabitEthernet0/0/1
 ip nat inside

interface GigabitEthernet0/0/2
 description ISP1
 ip nat outside

interface GigabitEthernet0/0/3
 description ISP2
 ip nat outside

ip access-list standard ACL_LAN
 permit 10.1.1.0 0.0.0.255

! Route-Map for ISP1
route-map RM_NAT_ISP1 permit 10
 match ip address ACL_LAN
 match interface GigabitEthernet0/0/2

! Route-Map for ISP2
route-map RM_NAT_ISP2 permit 10
 match ip address ACL_LAN
 match interface GigabitEthernet0/0/3

! NAT ポリシー適用
ip nat inside source route-map RM_NAT_ISP1 interface GigabitEthernet0/0/2 overload
ip nat inside source route-map RM_NAT_ISP2 interface GigabitEthernet0/0/3 overload
```

#### 検証方法
* 経路に応じて ISP1 または ISP2 へ送出されるパケットが、それぞれの出口 IP に正しく変換されることを確認。

---

### 例9: 重畳 IP ネットワーク相互接続 (Overlapping Network Twice-like NAT)

#### 問題
自社 LAN (`10.1.1.0/24`) とパートナー社 LAN (`10.1.1.0/24`) のサブネットが重複している。自社端末 `10.1.1.10` がパートナー社端末 `10.1.1.20` へアクセスできるよう、自社側を `172.16.1.10`、パートナー側を `172.16.2.20` へ相互変換（Static Inside/Outside NAT）せよ。

#### 要件
* インサイドおよびアウトサイドの両方で静的変換を適用すること。

#### 設定例

```bash
interface GigabitEthernet0/0/1
 description LOCAL_LAN
 ip nat inside

interface GigabitEthernet0/0/2
 description PARTNER_WAN
 ip nat outside

! 内部ローカル IP 10.1.1.10 を内部グローバル IP 172.16.1.10 へ変換
ip nat inside source static 10.1.1.10 172.16.1.10

! 外部グローバル IP 172.16.2.20 を外部ローカル IP 10.1.1.20 へ変換
ip nat outside source static 172.16.2.20 10.1.1.20
```

#### 検証方法
* 自社端末 `10.1.1.10` から `10.1.1.20` 宛てに Ping を送信し、両方向のアドレスが透明に変換されて疎通することを確認。

---

### 例10: HSRP 連携 Stateful NAT (SNAT) と冗長化構成

#### 問題
ルータ R1 (Active) と R2 (Standby) 間で HSRP グループ 1 を構成し、アクティブ側の NAT セッションテーブルをスタンバイ側ルータへ同期させる Stateful NAT (SNAT) を設定せよ。

#### 要件
* `ip nat stateful` を使用すること。

#### 設定例

```bash
! ===== ルータ R1 (Active) 構成 =====
interface GigabitEthernet0/0/1
 ip address 10.1.1.253 255.255.255.0
 ip nat inside
 standby 1 ip 10.1.1.254
 standby 1 priority 110
 standby 1 name HSRP_LAN

interface GigabitEthernet0/0/2
 ip address 203.0.113.2 255.255.255.0
 ip nat outside

! SNAT グループ構成
ip nat stateful id 1
 redundancy HSRP_LAN
  mapping-id 10

ip access-list standard ACL_SNAT
 permit 10.1.1.0 0.0.0.255

ip nat inside source list ACL_SNAT interface GigabitEthernet0/0/2 overload
```

#### 検証方法
* `show ip nat statistics` を実行し、Stateful NAT のピアリング状態が Sync されていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ルータ R1 で以下の NAT コンフィグが適用されている。内部ホスト `10.1.1.50` が対向拠点の `10.2.2.100` 宛てに通信を送信した。処理動作として正しいものはどれか。

```bash
ip access-list extended ACL_NAT
 deny ip 10.1.1.0 0.0.0.255 10.2.2.0 0.0.0.255
 permit ip 10.1.1.0 0.0.0.255 any

route-map RM_NAT permit 10
 match ip address ACL_NAT

ip nat inside source route-map RM_NAT interface GigabitEthernet0/0/2 overload
```

* A) パケットは ACL の `deny` 行にマッチするため、ルータ R1 によってドロップされる。
* B) パケットは NAT 変換されずに未変換のまま（送信元 IP `10.1.1.50`）ルーティング出力される。
* C) パケットは `GigabitEthernet0/0/2` の IP に PAT 変換されて送出される。
* D) Route-Map のパーサーエラーが発生し、ルータがフリーズする。

**正解**: **B**
**解説**: Route-Map に紐付けられた ACL の `deny` エントリは、パケットの破棄ではなく「**NAT 変換条件から除外する (Bypass)**」ことを意味します。パケットはドロップされず、元の送信元 IP (`10.1.1.50`) のまま通常のルーティングに従って送出されます（IPsec VPN の興味深いトラフィック等で利用されます）。

---

### 2. トラブルシュート問題

**問題**: 管理者が CSR1000v (IOS XE) 上で新しい NAT 設定を投入しようとして `interface GigabitEthernet1` 配下で `ip nat enable` を入力したところ、`% Invalid input detected at '^'` エラーが発生した。原因と正しい解決策はどれか。

* A) インターフェイスの IP アドレスが割り当てられていないため。
* B) CSR1000v (IOS XE) ソフトウェアでは NVI (`ip nat enable`) がサポートされていないため。`ip nat inside` / `ip nat outside` を使用する。
* C) `ip nat enable` の前に `license feature nat` を投入していないため。
* D) ルータのメモリが不足しているため。

**正解**: **B**
**解説**: Cisco IOS XE ソフトウェア（CSR1000v, ASR1000, ISR4000 等）では、NVI (NAT Virtual Interface) 機能が非サポート（削除）となっています。そのため `ip nat enable` コマンドは存在せず構文エラーとなります。従来型の `ip nat inside` / `ip nat outside` を使用する必要があります。

---

### 3. Design 問題

**問題**: 自社ネットワーク (`10.1.1.0/24`) と M&A 先企業ネットワーク (`10.1.1.0/24`) で完全な IP アドレスの重複が発生している。両社のシステムを一切変更することなく、自社の Web サーバー `10.1.1.100` を M&A 先企業からアクセス可能にするための設計として最も適切なものはどれか。

* A) 両社のルータに HSRP を設定して IP アドレスを共通化する。
* B) 自社 Web サーバー用に仮想 IP `172.16.1.100` を割り当て、自社境界ルータで `ip nat inside source static 10.1.1.100 172.16.1.100` の Static NAT を設定する。
* C) 全クライアント端末のサブネットマスクを `/16` に変更する。
* D) `ip nbar custom` を使用して IP アドレスを隠蔽する。

**正解**: **B**
**解説**: 重畳（Overlapping）IP ネットワークの接続設計では、競合しない別体系のアドレス（`172.16.1.100` 等）を Mapped アドレスとして定義し、Static 1:1 NAT または Twice NAT を用いて相互変換を行うのが標準設計です。

---

### 4. 実装問題

**問題**: ルータ R1 の WAN インターフェイス `203.0.113.50` に割り当てられていない外部 IP `203.0.113.99` を使用して、内部 DMZ サーバー `10.1.1.99` への Static NAT を構成した。しかし、ルータ R1 が `203.0.113.99` に対する不要な Proxy ARP 応答を周囲のネットワークに返してしまう。これを防止する正しい CLI 設定はどれか。

* A) `no ip proxy-arp` をインサイドポートに設定する。
* B) `ip nat inside source static 10.1.1.99 203.0.113.99 no-alias`
* C) `clear ip arp 203.0.113.99`
* D) `ip nat service arp-disable`

**正解**: **B**
**解説**: Cisco IOS / IOS XE の Static NAT でエイリアス IP に対する自動 Proxy ARP 応答を停止するには、静的 NAT コマンドの末尾に **`no-alias`** キーワードを追加します。

---

### 5. コンフィグ読解問題

**問題**: 以下のコンフィグを適用したルータ R1 において、外部ユーザーから `203.0.113.1` の TCP ポート `80` 宛てにアクセスが届いた。外部 WAN インターフェイス `GigabitEthernet0/0/2` に適用すべきインバウンド ACL (`ACL_IN`) の正解はどれか。

```bash
ip nat inside source static tcp 10.1.1.10 80 interface GigabitEthernet0/0/2 80
```

* A) `permit tcp any host 10.1.1.10 eq 80`
* B) `permit tcp any host 203.0.113.1 eq 80`
* C) `permit tcp host 10.1.1.10 any eq 80`
* D) `permit ip any any` のみ許可可能

**正解**: **B**
**解説**: Outside ➔ Inside のトラフィック処理順序（Order of Operation）では、**Inbound ACL のチェックが NAT 変換よりも先に実行されます**。そのため、インバウンド ACL (`ACL_IN`) には変換前の Mapped Global IP アドレス（`203.0.113.1`）宛ての通信を許可するルールを記述する必要があります。

---

## 🔗 参考リソース

* [Cisco IOS XE 17.x IP Addressing Configuration Guide - NAT Services](https://www.cisco.com/)
* [Cisco Technical Notes: NAT Order of Operation Overview](https://www.cisco.com/)
* [Cisco Technical Notes: How NAT Works with Route-Maps and Policy NAT](https://www.cisco.com/)
* [Cisco Command Reference - ip nat inside source](https://www.cisco.com/)
* [Cisco Live: Advanced NAT Architectures and Troubleshooting (BRKSEC-3004)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * NAT のトラブルシュートでは、**「Inside ➔ Outside は Routing が先、Outside ➔ Inside は NAT が先」** という基本原則を暗記しておくことが成功の鍵です。
  * `show ip nat translations` でエントリ（xlate）が生成されているにもかかわらず通信が届かない場合は、NAT ではなく Egress インターフェイスの ACL やルーティングテーブルの欠落を疑ってください。
* **図解（IOS NAT Order of Operation マトリクス）**:
  ```text
  ┌─────────────────────────────────────────────────────────────┐
  │ Inside ➔ Outside (LAN to WAN)                               │
  │   1. Ingress ACL (Check Inside Local IP)                    │
  │   2. Routing Decision (CEF Lookup for Destination)          │
  │   3. NAT Translation (Inside Local ➔ Inside Global)         │
  │   4. Egress ACL (Check Inside Global IP)                    │
  ├─────────────────────────────────────────────────────────────┤
  │ Outside ➔ Inside (WAN to LAN)                               │
  │   1. Ingress ACL (Check Outside Global / Mapped Dest IP)    │
  │   2. NAT Translation (Outside Global ➔ Inside Local)        │
  │   3. Routing Decision (CEF Lookup for Translated Real IP)   │
  │   4. Egress ACL (Check Inside Local IP)                     │
  └─────────────────────────────────────────────────────────────┘
  ```
* **注意点**:
  * トラブルシューティング中に NAT ポリシーを修正した場合、旧ポリシーに基づく既存の動的セッションがキャッシュに残るため、必ず **`clear ip nat translation *`** を実行してセッションテーブルをクリアしてから再検証を行ってください。
