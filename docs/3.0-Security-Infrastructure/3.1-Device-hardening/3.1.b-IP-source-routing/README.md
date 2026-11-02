---
layout: default
title: 3.1.b-IP-source-routing
nav_order: 2
parent: 3.1-Device-hardening
grand_parent: 3.0-Security-Infrastructure
---

# 3.1.b IP source routing

Cisco ネットワークインフラストラクチャにおける **IP Source Routing（IPソースルーティング）** は、送信元ホストが IP ヘッダー内のオプションフィールドを使用して、パケットがネットワーク内を通過する特定のルーターパス（ホップ順序）を明示的に指定するメカニズムです。

CCIE Security v6.1 Blueprint **3.1.b IP source routing** では、**Loose Source and Record Route (LSRR)** および **Strict Source and Record Route (SSRR)** の動作原理、IPv6 における **Routing Header Type 0 (RH0)** の脅威と無効化（RFC 5095）、**Cisco IOS/IOS-XE** における `no ip source-route` や `ip options drop` によるコントロール/データプレーン保護、および **Cisco ASA / Cisco FTD** における IP Options インスペクションとデフォルトでのドロップ挙動について、深く実践的な理解が要求されます。

---

## 📘 概要

* **機能概要**:
  * **IP Source Routing (IPv4)**: RFC 791 で定義された IPv4 オプション機能。パケットの送信元が最終目的地までの通過ルーターの IP アドレスリストを IP ヘッダー内に挿入し、中間ルーターの通常のルーティングテーブル（FIB）判定を上書きして指定パスへ転送させる。
    * **Strict Source Routing (SSRR, Type 137 / Option 9)**: パケットはヘッダーに指定された IP アドレスの順序通りの隣接ルーターのみを厳密に通過しなければならず、リストにない中間ホップの経由を一切認めない。
    * **Loose Source Routing (LSRR, Type 131 / Option 3)**: パケットはヘッダーに指定された主要ルーターを順番に通過するが、指定ルーター間にリスト外の中間ルーター（マルチホップ）が存在することを許可する。
  * **IPv6 Routing Header Type 0 (RH0)**: RFC 2460 で定義された IPv6 版のソースルーティング機能。中間ノードリストをヘッダーに含めるが、深刻な Amplification DoS（2つのルーター間でピンポン状態を発生させ帯域を枯渇させる攻撃）の脆弱性が発見されたため、RFC 5095 により正式に廃止（Deprecated）指定された。
  * **Cisco デバイスのセキュリティ保護**: IP オプション付きパケットは、Cisco Express Forwarding (CEF) の高速パス（ASIC/Hardware Switching）で処理できず、ルーターの CPU（Process Switching）へ送られるため、ルーターに対する Denial of Service (DoS) 攻撃やアクセス制御リスト（ACL/Firewall）のバイパスに悪用される。このため、モダンなセキュリティ設計ではグローバルまたはインターフェイス単位で完全に無効化/ドロップする。
* **利用目的**:
  * 元々はネットワークのデバッグ、トラフィックエンジニアリング、特定の実験的経路の強制用として設計された。
  * 現代のセキュアネットワークでは、**悪用リスクがメリットを完全に上回るため、セキュリティ境界および内部ルーターで無効化/ドロップすることが必須要件**となっている。
* **どのような場面で利用するか**:
  * セキュリティ要件として、エッジルーター（iACL / Ingress Filter）、データセンター境界、ファイアウォール（ASA / FTD）、およびコアスイッチ/ルーターのデバイス・ハーデンニング（Device Hardening）の一環として明示的に禁止・ドロップ設定を適用する場面。

---

## 🔑 要点

| 項目 | Loose Source Routing (LSRR) | Strict Source Routing (SSRR) | IPv6 Routing Header Type 0 (RH0) | Cisco ASA / FTD デフォルト挙動 |
| :--- | :--- | :--- | :--- | :--- |
| **IP Header Option / Type** | Option 3 (Type 131) | Option 9 (Type 137) | IPv6 Extension Header Type 0 | N/A (IP Options Inspection) |
| **パス指定の厳密性** | 指定 IP 間に他ルーターの経由を許可 | 指定 IP 間の直接接続のみ許可（他ホップ不可） | 明示リスト指定（ピンポン攻撃可能） | デフォルトで拒否 / ドロップ |
| **データプレーン処理** | Process Switching (CPU 負荷高) | Process Switching (CPU 負荷高) | Process Switching / Hardware Drop | LINA Engine / Snort で破棄 |
| **セキュリティ上の脅威** | ファイアウォール回避, Spoofing, DoS | ファイアウォール回避, Spoofing, DoS | Amplification DoS (ピンポン攻撃) | N/A (防御側) |
| **Cisco IOS 無効化コマンド** | `no ip source-route` | `no ip source-route` | `no ipv6 source-route` | デフォルト有効 (または `inspect ip-options`) |
| **パケット破棄コマンド** | `ip options drop` | `ip options drop` | N/A (IOS 12.4+ / 15.x で標準ドロップ) | `policy-map` / `inspect ip-options` |
| **関連標準 (RFC)** | RFC 791 / RFC 1812 | RFC 791 / RFC 1812 | RFC 2460 (廃止: RFC 5095) | Cisco Security Baseline |

---

## 🏗 動作原理

### 1. 通常ルーティング vs IP Source Routing の処理比較

通常、ルーターはパケットの**宛先 IP アドレス（Destination IP）**のみを参照し、自身のルーティングテーブル（FIB）に基づいて最適な送信出力を決定します。これに対し、IP Source Routing パケットが着信した場合、ルーターは IP ヘッダー内のオプションフィールドを解析し、ヘッダーに記述された「次の経由地 IP」を実際の宛先として一時的に置き換えて転送処理を行います。

```text
[ 通常の IP ルーティング (CEF Switching) ]

  [ Client ] ──► [ Router A (FIB Lookup) ] ──► [ Router B ] ──► [ Router C ] ──► [ Server ]
                     │
                     └── 宛先 IP (Server) のみを見て最適経路を選択 (ASIC/Hardware)


[ Loose / Strict Source Routing (Process Switching) ]

  [ Attacker ] ──► [ Router A ] ────── (バイパス) ──────► [ Rogue Gateway / Firewall Bypass ]
                         │
                         ├── 1. IP Header 内の Option 3/9 (Source Route) を検出
                         ├── 2. CEF ハードウェアパスから離脱し CPU (Process) へハンドオフ
                         ├── 3. ヘッダー内の Pointer が指す IP アドレスに宛先を変更
                         └── 4. 攻撃者が意図した不審な中間ノードへ強制リダイレクト
```

---

### 2. IPv4 IP Header Option (Source Route) の内部フォーマット

IPv4 ヘッダーの基本長は 20 バイトですが、Option が付加されると最大 60 バイトまで拡張されます。

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Type (131/137)| Length (可変) | Pointer (1-based octet offset)|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        First Hop IP Address                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Second Hop IP Address                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               ...                             |
```

* **Type**:
  * `131` (Hex `0x83`): Loose Source Route (LSRR)
  * `137` (Hex `0x89`): Strict Source Route (SSRR)
* **Length**: オプション全体のバイト長。
* **Pointer**: 現在処理すべき経由地 IP アドレスを示すオフセット位置（初期値は 4）。ルーターがパケットを処理するたびに Pointer が 4 バイト（1 IP 分）進み、自身の IP と置き換えられます。

---

## ⚙ 動作シーケンス

ルーターが IP Source Routing オプションを含むパケットを受信した際の内部処理シーケンス：

1. **パケット着信 ＆ IP ヘッダー検査**:
   * イーサネットフレームから IP パケットがデカプセル化され、IP ヘッダーの IHL (Internet Header Length) フィールドがチェックされる。
   * IHL > 5 (20 バイト超) の場合、IP オプションが存在すると判定。
2. **CEF ポンピング ＆ プロセス・スイッチングへの脱落**:
   * CEF (Cisco Express Forwarding) は IP オプション付きパケットをハードウェア ASIC で高速転送できません。
   * パケットは Fast Path から安全に切り離され、ルーターの CPU (Process Switching) 宛てにリダイレクトされる。
3. **ソースルーティング有効・無効の判定**:
   * **`no ip source-route` (デフォルト推奨設定) の場合**:
     * ルーターはパケットを即座に破棄（Drop）。
     * 送信元に対して `ICMP Destination Unreachable (Code 5: Source Route Failed)` を返答（`no ip unreachables` で抑制可能）。
   * **`ip source-route` (有効化されている場合) の場合**:
     * ヘッダー内の Pointer フィールドを参照。
     * Pointer が指す IP アドレスが自ルーターのインターフェイス IP と一致するか確認。
     * SSRR の場合、直前のホップが直接接続されていたかを厳密に検証。条件を満たさない場合はドロップ。
     * 次のホップ IP アドレスを書き換え、Pointer インクリメント後、アウトバウンドインターフェイスからフォワード。
4. **Cisco ASA / FTD での処理**:
   * ASA / FTD (LINA Engine) では、デフォルトで IP オプション付きパケット（RH0 や Source Route）を拒否するルール (`domain=inspect-ip-options, deny=true`) が発動し、ディープインスペクション前にアーリー・ドロップされる。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Cisco IOS / IOS-XE での無効化コマンドの徹底**:
   * **IPv4**: Global モードで `no ip source-route` を確認・投入。
   * **IPv6**: Global モードで `no ipv6 source-route` を確認・投入（RFC 5095 準拠）。
   * ※注意: 現代の IOS-XE バージョンではデフォルトで `no ip source-route` が有効ですが、試験要件で「Explicitly disable IP source routing on all routers」と指示された場合は、明示的に設定を入力・確認する必要があります。
2. **IP Options Selective Drop (`ip options drop`) の利用**:
   * `no ip source-route` は「ソースルーティング機能を使った転送」を拒否しますが、パケット自体は CPU に上がって処理される可能性があります。
   * パケットが CPU に届く前にインターフェイス/グローバルで破棄し、Process Switching による CPU 高負荷 (DoS) を防ぐため、**`ip options drop`** コマンドを使用することが推奨されます（※特定オプションのみ許可し他をドロップする ACL 設定が出題されるケースあり）。
3. **Extended Access List による IP オプション制御**:
   * Extended ACL では `option` キーワードを用いて IP オプション付きパケットを直接マッチ・ドロップできます。
   * 例: `access-list 100 deny ip any any option any-options` または `option traceroute` / `option record-route` の選択的許可。
4. **CoPP (Control Plane Policing) との組み合わせ**:
   * CPU 宛てに届く IP オプションパケット（TTL 0/1 や IP Options）を CoPP の Class-Map で捕獲し、Rate-Limit または Drop させる構成。
5. **Cisco ASA / FTD での IP Options パイプライン確認**:
   * ASA / FTD の `packet-tracer` コマンドを実行した際、`Phase: IP-OPTIONS` で `Result: ALLOW` または `DROP` になる動作ログを解読できること。
   * ASA で `inspect ip-options` をカスタマイズし、特定の IP オプション（End of Options List, No Operation, Basic Security 等）を通過/ドロップ制御する設定。

---

## 🛠 設定方法

### 1. Cisco IOS / IOS-XE: IP Source Routing の完全無効化 ＆ selective ドロップ

```bash
! ===== Step 1: Global での Source Routing 無効化 =====
no ip source-route
no ipv6 source-route

! ===== Step 2: IP Options 付きパケットの Selective Drop (CPU 保護) =====
! 全ての IP オプション付きパケットをコントロールプレーン手前で即座にドロップ
ip options drop

! ===== Step 3: ACL を用いた IP オプションパケットの明示的フィルタリング =====
! Traceroute や Timestamp 以外の危険な IP オプションパケットをドロップする ACL
ip access-list extended ACL_FILTER_IP_OPTIONS
 permit ip any any option traceroute
 permit ip any any option record-route
 permit ip any any option timestamp
 deny   ip any any option any-options
 permit ip any any

interface GigabitEthernet0/0/0
 ip access-group ACL_FILTER_IP_OPTIONS in
 no ip redirects
 no ip unreachables
```

---

### 2. Cisco IOS / IOS-XE: CoPP による IP オプションパケットの CPU 制限

```bash
! ===== Step 1: IP オプションパケットを識別する ACL =====
ip access-list extended ACL_COPP_IP_OPTIONS
 permit ip any any option any-options

! ===== Step 2: Class Map ＆ Policy Map の作成 =====
class-map match-all CM_COPP_IP_OPTIONS
 match access-group name ACL_COPP_IP_OPTIONS

policy-map PM_COPP_DEVICE_HARDENING
 class CM_COPP_IP_OPTIONS
  drop                                 ! または police 8000 conform-action transmit exceed-action drop

! ===== Step 3: Control Plane への適用 =====
control-plane
 service-policy input PM_COPP_DEVICE_HARDENING
```

---

### 3. Cisco ASA: MPF による IP Options インスペクション・制御

```bash
! ===== Step 1: IP Options Map の作成 =====
ip-options-map MAP_DROP_SOURCE_ROUTE
 drop lsrr                            ! Loose Source Route パケットをドロップ
 drop ssrr                            ! Strict Source Route パケットをドロップ
 drop rrt                             ! Record Route パケットをドロップ

! ===== Step 2: MPF Policy Map への組み込み =====
class-map CM_IP_OPTIONS
 match any

policy-map PM_GLOBAL_INSPECT
 class CM_IP_OPTIONS
  inspect ip-options MAP_DROP_SOURCE_ROUTE

service-policy PM_GLOBAL_INSPECT global
```

---

### 4. Cisco FTD (FMC Managed): FlexConfig / Platform Settings による制御

```text
[ FMC GUI 設定手順 ]

1. Platform Settings での無効化確認:
   - Devices > Platform Settings > 対象 FTD Policy 編集。
   - Security Configuration Parameters ➔ Anti-Spoofing / IP Options 設定画面を開く。
   - FTD ではデフォルトで Unicast RPF および IP Options Drop が有効化されている。

2. FlexConfig による ASA 互換 CLI 投入 (必要な場合):
   - Objects > FlexConfig > Text Location ➔ "Add Text Location" (Name: Flex_IP_Options)
   - Code 本文:
     ip-options-map MAP_FTD_DROP_SR
      drop lsrr
      drop ssrr
     policy-map global_policy
      class inspection_default
       inspect ip-options MAP_FTD_DROP_SR
   - Devices > FlexConfig Policy 編集 ➔ 対象 FTD へアサインして Deploy。
```

---

## 🔍 検証コマンド

| 目的 | コマンド (Cisco IOS / ASA / FTD) | 表示・判定の確認ポイント |
| :--- | :--- | :--- |
| **IP Source Routing 設定状態確認** | `show ip traffic` | `0 strict source route`, `0 loose source route` カウントの増加確認 |
| **IOS オプション設定有効化確認** | `show running-config | include source-route` | `no ip source-route`, `no ipv6 source-route` の存在確認 |
| **IP Options Drop 統計確認** | `show ip traffic | section Opts` | `Opts: 0 end, 0 nop, ... 0 loose source route, 0 strict source route` |
| **ACL による Option マッチ件数確認**| `show ip access-lists ACL_FILTER_IP_OPTIONS` | `deny ip any any option any-options (matches)` 件数の増加 |
| **CoPP ドロップ統計確認** | `show policy-map control-plane` | `CM_COPP_IP_OPTIONS` における Drop パケット数およびバイト数 |
| **ASA MPF インスペクション状態確認**| `show service-policy inspect ip-options` | `Inspect: ip-options`, パケット処理数、ドロップカウンター |
| **ASA/FTD パケットフローシミュレーション** | `packet-tracer input INSIDE ip <Src> <Dst> detailed` | `Phase: IP-OPTIONS` で `Result: ALLOW` または `DROP` 判定 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **`no ip source-route` を投入したにもかかわらずルーターの CPU 使用率が高騰する** | `no ip source-route` は転送を拒否するが、パケット自体は CPU に上がって処理されるため DoS を防げない | `show processes cpu sorted`<br>`show ip traffic` | **`ip options drop`** コマンドを有効化し、ハードウェア/データプレーンでパケット自体を即座に破棄させる。 |
| **Traceroute や Path MTU Discovery が一部のネットワークで失敗する** | `ip options drop` または ACL で `option any-options` を全ドロップしたため、必要な IP オプション（Timestamp や Record Route）が遮断された | `show ip access-lists`<br>`debug ip packet` | ACL で `permit ip any any option traceroute` や `option record-route` を明示的に拒否ルールより上位に配置する。 |
| **IPv4 の Source Route はブロックされたが、IPv6 パケットでルーターがループ・高負荷になる** | IPv6 の `no ipv6 source-route` (RH0 対策) が未設定で放置されていた | `show running-config | inc ipv6` | **`no ipv6 source-route`** をグローバル設定に追加し、RH0 パケットをブロックする。 |
| **ASA/FTD を通過する特定の診断ツール通信が Drop される** | ASA のデフォルト MPF または `inspect ip-options` で IP オプションが拒否されている | `packet-tracer`<br>`show service-policy inspect ip-options` | 特定の IP オプションが必要な場合、`ip-options-map` で該当オプション（例: `allow rrt`）を明示的に許可にカスタマイズする。 |

---

## ⚠ 制限事項

* **Cisco IOS/IOS-XE ハードウェアスイッチングの制限**:
  * IP オプションを含むすべてのパケットは、CEF の ASIC / ハードウェアフォワーディングパスをバイパスして CPU (Process Switching) へリダイレクトされます。大量の IP オプションパケットが流入すると、スイッチング方式に関わらず CPU 枯渇を引き起こします。
* **IPv6 Routing Header Type 0 (RH0) の非推奨化 (RFC 5095)**:
  * Cisco IOS Release 12.4T / 15.x 以降では、IPv6 RH0 はデフォルトで拒否処理されますが、古い IOS バージョンでは明示的に `no ipv6 source-route` の投入が必要です。
* **ASA / FTD でのマルチコンテキスト制限**:
  * ASA マルチコンテキストモードでは、一部のグローバル IP オプション設定や MPF インスペクションが Admin コンテキストまたはシステム実行空間で一括管理される場合があります。

---

## 🔄 他技術との関連

* **Control Plane Policing (CoPP)**: CPU へ向かう IP オプションパケット（LSRR/SSRR）に対し、帯域制限（Rate Limiting）やドロップポリシーを適用してコントロールプレーンを保護する。
* **Unicast Reverse Path Forwarding (uRPF)**: 送信元 IP アドレスのスプーフィングを防止する技術。IP Source Routing と組み合わせることで、L3 ルーティング層での完全な非正当トラフィック排除を実現する。
* **Infrastructure ACL (iACL)**: ネットワーク境界（Edge Router）でインフラ設備（ルーターのループバックや物理 IP）宛ての不正パケット（IP オプション含む）をブロックする。

---

## 🧩 比較表

| 比較項目 | Loose Source Routing (LSRR) | Strict Source Routing (SSRR) | IPv6 RH0 | IP Options Drop (`ip options drop`) |
| :--- | :--- | :--- | :--- | :--- |
| **プロトコル** | IPv4 (Option 3) | IPv4 (Option 9) | IPv6 Extension Header | IPv4 / IPv6 全般 |
| **経路指定の条件** | 指定 IP 間に任意ルーター経由可 | 指定 IP 間の直接接続のみ | リスト順（反復指定可能） | 全 IP オプション付きパケット |
| **目的** | デバッグ・手動経路制御 | 厳密な特定経路の強制 | IPv6 手動経路指定 | **CPU 保護・DoS 緩和 (防衛)** |
| **セキュリティ評価** | **高リスク (FW バイパス)** | **高リスク (FW バイパス)** | **極めて危険 (RFC 5095 廃止)** | **Cisco 推奨ベストプラクティス** |

---

## 💡 ベストプラクティス

1. **すべての Cisco IOS / IOS-XE ルーターで `no ip source-route` および `no ipv6 source-route` を標準無効化**:
   * ゴールデンコンフィグ（標準構成テンプレート）に組み込み、全ルーターで常時無効化を徹底します。
2. **`ip options drop` の適用による CPU 保護**:
   * 単に機能として拒否するだけでなく、`ip options drop` を有効化してパケットをコントロールプレーンに上げずに初期段階で破棄します。
3. **エッジルーターでの Ingress ACL / iACL による遮断**:
   * 外部インターネット境界の Ingress ACL において、`deny ip any any option any-options` を配置し、組織ネットワーク内への流入を物理エッジで遮断します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Cisco IOS-XE での IP Source Routing (IPv4/IPv6) 完全無効化

#### 問題
ルーター R1 において、悪意ある送信元によるアクセス制御バイパスや IPv6 拡張ヘッダー攻撃を防ぐため、IPv4 Loose/Strict Source Routing および IPv6 Routing Header Type 0 (RH0) 処理を完全に無効化せよ。

#### 要件
* グローバル設定で IPv4 および IPv6 のソースルーティングを明示的に無効化すること。
* インターフェイス `GigabitEthernet0/0/0` で ICMP リダイレクトも停止すること。

#### 設定例

```bash
! ===== R1 Configuration =====
configure terminal

! IPv4 / IPv6 Source Routing の完全無効化
no ip source-route
no ipv6 source-route

interface GigabitEthernet0/0/0
 no ip redirects
 no ipv6 redirects
exit
end
```

#### 検証方法
* `show running-config | include source-route` を実行し、両コマンドが記述されていることを確認。

---

### 例2: `ip options drop` によるデータプレーンでの IP オプションパケット即時破棄

#### 問題
ルーター R2 において、IP オプション付きパケットが CPU (Process Switching) へ到達してリソースを消費する DoS 攻撃を防止するため、データプレーンレベルでパケットを即座に破棄する設定を投入せよ。

#### 要件
* `ip options drop` コマンドを使用すること。

#### 設定例

```bash
! ===== R2 Configuration =====
configure terminal

! IP オプション付きパケットの即時ドロップ
ip options drop
end
```

#### 検証方法
* `show ip traffic` を実行し、`Opts:` カウントおよびドロップ統計を確認。

---

### 例3: 拡張 ACL を用いた IP オプション付きパケットの選択的フィルタリング

#### 問題
ルーター R3 の `GigabitEthernet0/0/1` インターフェイスにおいて、診断用の `traceroute` オプションおよび `record-route` オプション付きパケットのみを許可し、その他の IP オプション（LSRR/SSRR 等）を含む全パケットをドロップする Ingress ACL を設定せよ。

#### 要件
* ACL 名: `ACL_IP_OPTION_SELECTIVE`
* インターフェイス: `GigabitEthernet0/0/1` Inbound

#### 設定例

```bash
! ===== R3 Configuration =====
configure terminal

ip access-list extended ACL_IP_OPTION_SELECTIVE
 permit ip any any option traceroute
 permit ip any any option record-route
 deny   ip any any option any-options
 permit ip any any
exit

interface GigabitEthernet0/0/1
 ip access-group ACL_IP_OPTION_SELECTIVE in
exit
end
```

#### 検証方法
* `show ip access-lists ACL_IP_OPTION_SELECTIVE` を実行し、`deny ip any any option any-options` のマッチ件数（`matches`）が増加することを確認。

---

### 例4: CoPP (Control Plane Policing) による IP オプションパケットの CPU 保護

#### 問題
コアコア波ルーター R4 において、コントロールプレーン宛てに送られてくる IP オプション付きパケットをクラス分けし、CoPP で完全に破棄 (Drop) するポリシーを構成せよ。

#### 要件
* Class Map 名: `CM_COPP_IP_OPTIONS`
* Policy Map 名: `PM_COPP_HARDENING`

#### 設定例

```bash
! ===== R4 Configuration =====
configure terminal

ip access-list extended ACL_COPP_OPTIONS
 permit ip any any option any-options

class-map match-all CM_COPP_IP_OPTIONS
 match access-group name ACL_COPP_OPTIONS

policy-map PM_COPP_HARDENING
 class CM_COPP_IP_OPTIONS
  drop

control-plane
 service-policy input PM_COPP_HARDENING
exit
end
```

#### 検証方法
* `show policy-map control-plane` を実行し、`CM_COPP_IP_OPTIONS` の Drop カウンターが更新されていることを確認。

---

### 例5: Cisco ASA MPF による Loose/Strict Source Route パケットのドロップ

#### 問題
ASA1 において、Modular Policy Framework (MPF) を構成し、外線および内線から着信する Loose Source Route (LSRR) および Strict Source Route (SSRR) パケットを即座に破棄せよ。

#### 要件
* `ip-options-map` 名: `MAP_BLOCK_SR`
* グローバル サービスポリシーに組み込むこと。

#### 設定例

```bash
! ===== ASA1 Configuration =====
configure terminal

ip-options-map MAP_BLOCK_SR
 drop lsrr
 drop ssrr

policy-map global_policy
 class inspection_default
  inspect ip-options MAP_BLOCK_SR

service-policy global_policy global
end
```

#### 検証方法
* `show service-policy inspect ip-options` を実行し、`lsrr` および `ssrr` のドロップカウントが増加することを確認。

---

### 例6: Cisco ASA における Record Route オプションの通過許可カスタマイズ

#### 問題
ASA2 において、社内ネットワーク診断で運用されている Record Route (`rrt`) オプション付きパケットのみを検査で通過許可とし、その他のソースルーティングオプション（LSRR/SSRR）を拒否する設定を行え。

#### 要件
* `ip-options-map` 名: `MAP_ALLOW_RRT`

#### 設定例

```bash
! ===== ASA2 Configuration =====
configure terminal

ip-options-map MAP_ALLOW_RRT
 allow rrt
 drop lsrr
 drop ssrr

policy-map global_policy
 class inspection_default
  inspect ip-options MAP_ALLOW_RRT

service-policy global_policy global
end
```

#### 検証方法
* `packet-tracer input INSIDE ip 10.1.1.10 172.16.1.100 detailed` を実行し、`Phase: IP-OPTIONS` の判定結果を検証。

---

### 例7: Cisco FTD (FlexConfig) による IP Options カスタムドロップポリシーの投入

#### 問題
FMC 管理下の FTD1 デバイスに対し、FlexConfig を使用して ASA 互換の `ip-options-map` を定義し、ソースルーティングパケットをドロップさせよ。

#### 要件
* FlexConfig Text Object 名: `Flex_IP_Options_Drop`

#### 設定例

```text
[ FMC FlexConfig 設定例 ]

1. Objects > Object Management > FlexConfig > Text Location ➔ "Add Text Location":
   - Name: Flex_IP_Options_Drop
   - Deployment: Every Time
   - CLI Code:
     ip-options-map MAP_FTD_SR_DROP
      drop lsrr
      drop ssrr
     policy-map global_policy
      class inspection_default
       inspect ip-options MAP_FTD_SR_DROP

2. Devices > FlexConfig > 対象 FTD1 の Policy 編集 ➔ Flex_IP_Options_Drop をアサインして Deploy。
```

#### 検証方法
* FTD Diagnostic CLI で `show service-policy inspect ip-options` を実行し、ドロップカウントを確認。

---

### 例8: Ingress iACL (Infrastructure ACL) における IP Source Route 遮断

#### 問題
エッジルーター R5 のインターネット対向インターフェイス `GigabitEthernet0/0/0` において、自社 IP プレフィックス (`203.0.113.0/24`) 宛ておよび自ルーター宛てのトラフィックのうち、IP オプションが付加されたパケットを最前段でブロックせよ。

#### 要件
* ACL 名: `iACL_INGRESS_PROTECT`

#### 設定例

```bash
! ===== R5 Configuration =====
configure terminal

ip access-list extended iACL_INGRESS_PROTECT
 ! IP オプション付きパケットのドロップ
 deny ip any any option any-options
 ! 自社ネットワーク宛て通常の通信の許可
 permit ip any 203.0.113.0 0.0.0.255
 permit ip any host 198.51.100.1
exit

interface GigabitEthernet0/0/0
 ip access-group iACL_INGRESS_PROTECT in
exit
end
```

#### 検証方法
* `show ip access-lists iACL_INGRESS_PROTECT` を実行し、1行目の `deny` ルールに対するマッチを確認。

---

### 例9: uRPF Strict Mode ＋ `no ip source-route` によるスプーフィング/経路バイパスの多層防御

#### 問題
境界ルーター R6 の内部 WAN インターフェイス `GigabitEthernet0/0/2` において、uRPF Strict Mode と IP Source Routing 無効化を組み合わせ、IP スプーフィングおよび手動経路変更攻撃を多層防御せよ。

#### 要件
* `ip verify unicast source reachable-via rx` を設定すること。
* グローバルで `no ip source-route` を適用すること。

#### 設定例

```bash
! ===== R6 Configuration =====
configure terminal

no ip source-route

interface GigabitEthernet0/0/2
 ip verify unicast source reachable-via rx
 no ip redirects
exit
end
```

#### 検証方法
* `show ip interface GigabitEthernet0/0/2 | section verify` を実行し、IP verify が Strict (rx) モードで動作していることを確認。

---

### 例10: IPv6 ネットワークにおける RH0 対策 ＆ IPv6 ACL によるオプション制御

#### 問題
Dual-Stack ルーター R7 において、IPv6 パケットの Routing Header Type 0 (RH0) 攻撃を無効化し、さらに IPv6 拡張 ACL で不要なヘッダー付きパケットを拒否せよ。

#### 要件
* グローバル設定: `no ipv6 source-route`
* IPv6 ACL 名: `ACL6_HARDENING`

#### 設定例

```bash
! ===== R7 Configuration =====
configure terminal

! IPv6 RH0 処理の完全無効化
no ipv6 source-route

ipv6 access-list ACL6_HARDENING
 deny ipv6 any any routing
 permit ipv6 any any
exit

interface GigabitEthernet0/0/0
 ipv6 traffic-filter ACL6_HARDENING in
exit
end
```

#### 検証方法
* `show ipv6 access-list ACL6_HARDENING` を実行し、routing ヘッダー拒否ルールのヒットカウントを検証。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: Cisco IOS-XE ルーターにおいて、以下の設定が投入されている。外部からの Loose Source Route (LSRR) オプションを含む IP パケットを受信した場合のルーターの内部挙動として最も適切な説明はどれか。

```bash
no ip source-route
```

* A) ルーターは CEF ハードウェアテーブルでパケットを照合し、高速にフォワードする。
* B) ルーターはパケットの転送処理（Source Route 処理）を拒否して破棄するが、パケット自体は CPU (Process Switching) へリダイレクトされて評価されるため、大量パケット着信時には CPU 高負荷を引き起こす可能性がある。
* C) ルーターはパケットをインターフェイス ASIC レベルで無条件に即座破棄するため、CPU には一切負荷がかからない。
* D) ルーターは IP ヘッダーから Source Route オプションのみを自動削除して、通常の宛先 IP ルーティングを行う。

**正解**: **B**  
**解説**: `no ip source-route` コマンドはソースルーティング機能を用いたパケット転送を無効化しますが、IP オプション検出自体が CEF から CPU (Process Switching) へのハンドオフを引き起こすため、パケット評価は CPU 上で行われます。パケットそのものを CPU に上げずに初期段階で破棄するには `ip options drop` コマンドが必要です。

---

### 2. トラブルシューティング問題

**問題**: ネットワーク管理者から「コアライセンスルーターの CPU 使用率が上昇し、`show processes cpu sorted` で IP Input プロセスの占有率が高い」と報告があった。原因調査コマンドとして `show ip traffic` を実行したところ、以下の結果が得られた。直ちにとるべき緩和処置コマンドはどれか。

```text
Opts:  0 end, 0 nop, 0 basic security, 1425012 loose source route
       0 timestamp, 0 extended security, 0 record route
       0 stream ID, 1205941 strict source route, 0 alert, 0 cipso
```

* A) `ip verify unicast source reachable-via rx`
* B) `no ip redirects`
* C) `ip options drop`
* D) `service-policy input PM_QOS`

**正解**: **C**  
**解説**: ログから大量の Loose Source Route および Strict Source Route パケットが着信し、Process Switching を引き起こして CPU を圧迫していることが確認できます。`ip options drop` をグローバル設定で有効化することで、IP オプション付きパケットをコントロールプレーンに上げずに即座にドロップさせ、CPU 高負荷を即座に収束させることができます。

---

### 3. Design 問題

**問題**: 新規 IPv6 ネットワークを構築する際、Cisco セキュリティガイドラインに従ってデバイス保護を実施している。IPv6 環境におけるソースルーティング（Routing Header Type 0 [RH0]）の保護に関する記述として、最も適切なものはどれか。

* A) IPv6 にはソースルーティング機能が存在しないため、特別な設定は不要である。
* B) RFC 5095 により IPv6 RH0 は非推奨化および廃止されており、Cisco IOS-XE では `no ipv6 source-route` コマンドの投入によってブロックすべきである。
* C) IPv6 RH0 パケットは常に CEF ハードウェアで高速転送されるため、リスクはない。
* D) IPv6 ソースルーティングは uRPF Strict Mode を設定することで自動的に完全無効化される。

**正解**: **B**  
**解説**: IPv6 Routing Header Type 0 (RH0) は 2 つのルーター間でパケットを反復ループさせて帯域を消費させる問題があるため、RFC 5095 で廃止されました。`no ipv6 source-route` を明示的に投入することがデザイン上のベストプラクティスです。

---

### 4. 実装問題 (ASA MPF)

**問題**: Cisco ASA において、`ip-options-map` を使用して Strict Source Route (`ssrr`) および Loose Source Route (`lsrr`) パケットをドロップし、かつ全体にインスペクションを適用するための正しい設定手順はどれか。

* A) 
```bash
ip-options-map MAP_DROP
 drop lsrr
 drop ssrr
policy-map global_policy
 class inspection_default
  inspect ip-options MAP_DROP
```
* B)
```bash
ip-options-map MAP_DROP
 permit lsrr
 permit ssrr
access-group MAP_DROP in interface outside
```
* C)
```bash
access-list ACL_DROP extended deny ip any any option source-route
access-group ACL_DROP in interface outside
```
* D)
```bash
sysopt connection permit-ip-options MAP_DROP
```

**正解**: **A**  
**解説**: ASA での IP オプションの高度な制御は、`ip-options-map` でドロップ対象条件（`drop lsrr`, `drop ssrr`）を定義し、MPF の `policy-map` 内で `inspect ip-options <map-name>` を適用することで実現します。

---

### 5. コンフィグ読解問題

**問題**: 以下の Access List が Cisco IOS インターフェイスの Inbound に適用されている。外部から着信した `option traceroute` 付加パケットおよび `option loose-source-route` 付加パケットに対する ACL の処理結果の組み合わせとして正しいものはどれか。

```bash
ip access-list extended ACL_CHECK
 permit ip any any option traceroute
 deny   ip any any option any-options
 permit ip any any
```

* A) `traceroute`: Permit / `loose-source-route`: Permit
* B) `traceroute`: Permit / `loose-source-route`: Deny
* C) `traceroute`: Deny / `loose-source-route`: Deny
* D) `traceroute`: Deny / `loose-source-route`: Permit

**正解**: **B**  
**解説**: ACL は Top-Down で評価されます。1 行目の `permit ip any any option traceroute` により `traceroute` オプション付きパケットは許可されます。次に 2 行目の `deny ip any any option any-options` により、`loose-source-route` を含むその他の全 IP オプション付きパケットがドロップされます。

## 🔗 参考リソース

* Cisco IOS IP Addressing Services Configuration Guide: Disabling IP Source Routing
* Cisco Technical Notes: IP Options Selective Drop Feature
* Cisco ASA 9.x CLI Configuration Guide: Configuring IP Options Inspection
* Cisco Secure Firewall Management Center Device Configuration Guide: Security Configuration Parameters
* RFC 791: Internet Protocol
* RFC 1812: Requirements for IP Version 4 Routers
* RFC 5095: Deprecation of Type 0 Routing Headers in IPv6

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で Device Hardening / Control Plane Protection に関する問題が出題された場合、`no ip source-route` と `ip options drop` の 2 コマンドの違いを正確に理解しておくことが決定的なポイントとなります。
  * `no ip source-route`: ルーター機能としてのソースルーティング転送処理を無効化（パケットは CPU へ届く）。
  * `ip options drop`: IP オプション付きパケット自体をデータプレーン初期段階で無条件ドロップ（CPU 保護）。
* **図解（IP オプションパケットの評価分岐）**:
  ```text
  [ Ingress IP Packet ]
            │
            ▼
  [ IP Header IHL > 5 (Options Exist?) ]
            │
            ├── NO ──► [ Standard CEF Forwarding (ASIC Fast Path) ]
            │
            └── YES ──► [ Is `ip options drop` Enabled? ]
                              │
                              ├── YES ──► [ DROP Immediately (CPU Saved) ]
                              │
                              └── NO ──► [ Hand off to CPU (Process Switching) ]
                                                │
                                                ├── Is `no ip source-route` set?
                                                │     ├── YES ──► [ DROP & Send ICMP Unreachable ]
                                                │     └── NO  ──► [ Process & Forward ]
  ```
* **注意点**:
  * 試験で「Do not affect standard network diagnostics like traceroute」という条件がついている場合、`ip options drop` を単純にグローバル設定すると `traceroute` オプション等が遮断されるリスクがあるため、ACL による選択的許可 (`permit ip any any option traceroute`) ＋ `deny ip any any option any-options` の構成を検討してください。
