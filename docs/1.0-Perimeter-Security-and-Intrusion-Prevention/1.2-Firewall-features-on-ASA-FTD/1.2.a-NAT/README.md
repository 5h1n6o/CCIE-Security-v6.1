---
layout: default
title: 1.2.a-NAT
nav_order: 1
parent: 1.2-Firewall-features-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.2.a NAT

Network Address Translation (NAT) および Port Address Translation (PAT) は、Cisco ASA (Adaptive Security Appliance) および Cisco FTD (Firepower Threat Defense) における核心的なセキュリティおよびネットワーク制御機能です。IPv4 アドレス枯渇対策（RFC 1918 プライベート IP のカプセル化）にとどまらず、重複ネットワーク（Overlapping Subnets）の接続、外部公開サーバーの隠蔽、VPN トラフィックの非変換（Identity NAT / No-NAT）、および DNS レコードの動的書き換え（DNS Doctoring / Rewrite）など、多様な設計要件に応じた柔軟なポリシー展開を実現します。

ASA OS 8.3(1) 以降で導入された新 NAT アーキテクチャは、FTD (LINA Engine) にもそのまま継承されており、**Auto NAT (Object NAT)** と **Manual NAT (Twice NAT)** の 2 つの構成手法および **3 つのセクション（Section 1, Section 2, Section 3）** による厳格な評価順序（Order of Operation）に基づいて処理されます。

---

## 📘 概要

* **機能概要**:
  * パケットがファイアウォールを通過する際、L3 ヘッダー内の送信元/宛先 IP アドレス、および L4 ヘッダー内の送信元/宛先ポート番号を静的（Static）または動的（Dynamic）に書き換える機能です。
  * **Auto NAT (Object NAT)**: ネットワークオブジェクト（`object network`）の配下で定義され、単一のアドレス（送信元または宛先のいずれか一方）の変換を行います。
  * **Manual NAT (Twice NAT)**: グローバルコンフィグレーション（`nat` コマンド）で定義され、同一ルール内で送信元アドレスと宛先アドレス（および L4 ポート）を同時に指定・変換可能です。
* **利用目的**:
  * RFC 1918 プライベート IP アドレス空間からインターネットへの透過的なアクセス（Dynamic PAT）。
  * 内部 DMZ 上の Web / Mail サーバーをパブリック IP アドレスへ静的に公開（Static NAT / Static PAT）。
  * M&A や拠統合に伴う IP アドレス重複環境における相互通信（Twice NAT / Double NAT）。
  * IPsec VPN や AnyConnect Remote Access VPN 通信において、暗号化対象トラフィックを NAT 変換から除外（Identity NAT / NAT Exemption）。
  * 外部 DNS サーバーが返却するパブリック IP を内部用プライベート IP へ透過的に修正（DNS Rewrite / DNS Doctoring）。
* **どのような場面で利用するか**:
  * 企業インターネット境界（Internet Edge）、DMZ サーバー公開エリア、拠点間 IPsec VPN 接続ポイント、マルチテナント環境。

---

## 🔑 要点

| 項目 | Auto NAT (Object NAT) | Manual NAT (Twice NAT) |
| :--- | :--- | :--- |
| **定義場所** | ネットワークオブジェクト配下 (`object network <NAME>`) | グローバル設定 (`nat (...)` コマンド) |
| **評価セクション** | **Section 2 (Auto NAT)** に自動配置 | **Section 1 (Before Auto)** または **Section 3 (After Auto)** |
| **変換範囲** | 送信元アドレスまたは宛先アドレスの **いずれか一方** | **送信元および宛先（＋L4ポート）を同時に変換可能** |
| **ルール評価順序** | ASA/FTD 内部アルゴリズムにより自動ソート（Prefix長/IP順等） | **ライン番号順（Top-Down / 記述順）** に厳格評価 |
| **Policy NAT サポート** | 不可（宛先 IP や L4 サービスを条件とできない） | **可能**（宛先 IP・L4 サービスポート条件をインライン指定） |
| **主な用途** | 標準的な Dynamic PAT、公開サーバーの Static NAT/PAT | Identity NAT (VPN No-NAT)、Twice NAT、Policy NAT |
| **対応機種** | Cisco ASA (OS 8.3以降), Cisco FTD (FMC / FDM 全バージョン) | Cisco ASA (OS 8.3以降), Cisco FTD (FMC / FDM 全バージョン) |
| **制限事項** | オブジェクト内に含められる IP アドレス数は最大 131,072 | ICMP に対する Dynamic Policy PAT 制限等あり |
| **設計上の注意点** | セクション内の順序が自動決定されるため意図しない昇順ソートに注意 | Section 1 に広範なルールを入れると Section 2 が評価されない |

---

## 🏗 動作原理

ASA / FTD における NAT は、単なるパケットヘッダーの書き換え処理ではなく、**ルーティング決定 (Route Lookup)** および **アクセス制御ポリシー (ACL / ACP)** の評価と密接に相互作用します。

```text
[ Ingress Packet (e.g., Inside: 10.1.1.10:12345 -> Outside: 203.0.113.100:80) ]
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ LINA Fastpath / Packet Processing Engine                                      │
│                                                                               │
│  1. NAT Table Evaluation (Section 1 -> Section 2 -> Section 3)                │
│     ├── Match Found?                                                          │
│     │     ├── YES ──► Determine UN-NAT (Real Destination IP: 192.168.10.10)   │
│     │     └── NO  ──► Retain Original Addresses                               │
│                                                                               │
│  2. Egress Interface & Route Lookup (based on UN-NAT Destination IP)           │
│     └── Determine Output Interface (e.g., DMZ)                                │
│                                                                               │
│  3. Access Control Check (ACL / ACP)                                          │
│     └── Evaluated against REAL (UN-NAT) Destination IP (192.168.10.10:80)     │
│                                                                               │
│  4. Session Creation (Connection Table - xlate Creation)                      │
│     └── Build 5-tuple xlate entry in Connection Table                         │
│                                                                               │
│  5. Header Rewrite (NAT Execution)                                            │
│     ├── Source IP Rewritten (if configured)                                   │
│     └── Destination IP Rewritten (203.0.113.100 -> 192.168.10.10)             │
└───────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
[ Egress Packet (e.g., DMZ: 10.1.1.10:12345 -> DMZ: 192.168.10.10:80) ]
```

---

## ⚙ 動作シーケンス

パケットが ASA / FTD に到着した際、NAT エンジンは以下のステップで処理を行います。

1. **Ingress パケットの受信と UN-NAT (Untranslate / Reverse NAT) 判定**:
   * 受信した宛先 IP アドレスが Static NAT / Static PAT のマッピング（Mapped Address）に一致するか検証します。
   * マッチした場合、パケットの「真の宛先 IP（Real Address）」を特定します（これが UN-NAT 処理です）。
2. **Egress インターフェイス決定 (Route Lookup)**:
   * **UN-NAT 後の真の宛先 IP** に基づき、グローバルルーティングテーブル（FIB）を参照して出力インターフェイス（Egress Interface）を決定します（※ `route-lookup` オプション有効時、または標準動作）。
3. **アクセス制御リスト (ACL / ACP) 評価**:
   * インバウンド ACL / ACP ルールを評価します。
   * **最重要**: アクセス制御ルールは、**変換後のパブリック IP ではなく「真の内部 IP（Real IP）」** を宛先として評価されます（ASA OS 8.3+ / FTD の標準仕様）。
4. **NAT テーブル検索 (NAT Order of Operation)**:
   * パケットの送信元・宛先・ポート条件がどの NAT セクションのどのルールにマッチするかを検索します。
     * **Section 1 (Manual NAT / Before Auto)**: 設定された行順（Line Number）に評価。
     * **Section 2 (Auto NAT)**: 静的（Static）ルール ➔ 動的（Dynamic）ルールの順に評価。同種内ではプレフィックス長が長い順（/32 ➔ /24）、IP アドレスの昇順、オブジェクト名のアルファベット順にソート。
     * **Section 3 (Manual NAT / After Auto)**: `after-auto` オプションで配置された手動ルールを行順に評価。
5. **xlate エントリの作成とヘッダー書き換え**:
   * 最初にマッチしたルールに基づき、`xlate` (Translation) テーブルにセッションを登録し、IP/Port ヘッダーを書き換えて送出します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

* **NAT オーダリング（順序）の制御要求**:
  * 問題文で「この NAT ルールは他のすべてのオブジェクト NAT より先に評価させよ」と指定された場合、**Section 1 (Manual NAT)** として作成します。
  * 逆に「すべてのオブジェクト NAT で変換されなかったトラフィックの最終フォールバックとして PAT 変換せよ」と指示された場合は、`after-auto` キーワードを使用して **Section 3 (Manual NAT After Auto)** に配置します。
* **VPN トラフィックに対する Identity NAT (No-NAT / NAT Exemption) の配置**:
  * 内部 LAN（`10.1.1.0/24`）が外部へ Dynamic PAT されている環境で、対向拠点（`192.168.1.0/24`）宛の VPN トラフィックを NAT 変換から除外する場合、**Section 1 の最優先位置に Identity NAT を配置** しなければなりません。
  * 構成ミス例: Dynamic PAT (Auto NAT) が先に評価され、VPN パケットの送信元 IP がパブリック IP に変換されて IPsec ピアでドロップされる。
* **Proxy ARP の副作用と `no-proxy-arp`**:
  * ASA / FTD は、Static NAT や Identity NAT で指定された Mapped IP に対し、受信インターフェイス上でデフォルトで **Proxy ARP** を応答します。
  * Wide-range な Identity NAT（例: `source static any any`）で Proxy ARP が有効なままだと、ファイアウォールがネットワーク上のあらゆる ARP 要求に自分の MAC を返してしまい、大量の通信障害（ARP 盗聴・ループ）を引き起こします。必ず **`no-proxy-arp`** オプションを付与する問題が頻出します。
* **`route-lookup` オプションの必須指定**:
  * Manual Twice NAT ルールにおいて、`nat (inside,outside)` のようにインターフェイスを明示的に指定した場合、デフォルトでは宛先 IP がルーティングテーブル上で別のポートに向いていても、指定された Egress インターフェイス (`outside`) へ強制転送されます。
  * 正しいインターフェイスへルーティングさせるため、**`route-lookup`** キーワードを付与させる要求が出題されます。
* **DNS Rewrite (DNS Doctoring)**:
  * 外部にある DNS サーバーが、内部 Web サーバーのパブリック IP を返却する際、内部クライアントがプライベート IP で直接通信できるように A レコードを書き換える機能です。
  * **制限要件**: Static NAT / Static Auto NAT でのみサポートされ、**PAT や Twice NAT（送信元・宛先両方指定ルール）では使用不可** です。また、DNS Inspection が有効である必要があります。

---

## 🛠 設定方法

### 1. Cisco ASA: CLI による NAT 構成

```bash
! ===== 1. Static Auto NAT (サーバー公開) =====
object network OBJ_WEB_SERVER_REAL
 host 10.1.1.100
 nat (inside,outside) static 203.0.113.100 dns

! ===== 2. Dynamic Auto PAT (インターフェイス PAT ＆ PAT プール) =====
object network OBJ_INSIDE_SUBNET
 subnet 10.1.0.0 255.255.0.0
 nat (inside,outside) dynamic interface

! PAT Pool オブジェクトを使用した動的 PAT
object network OBJ_PAT_POOL
 range 203.0.113.200 203.0.113.210

object network OBJ_GUEST_SUBNET
 subnet 192.168.100.0 255.255.255.0
 nat (inside,outside) dynamic pat-pool OBJ_PAT_POOL flat round-robin

! ===== 3. Section 1: Manual Twice NAT (Identity NAT / VPN No-NAT) =====
object network OBJ_LOCAL_LAN
 subnet 10.1.0.0 255.255.0.0

object network OBJ_REMOTE_VPN_LAN
 subnet 172.16.0.0 255.255.0.0

nat (inside,outside) section 1 source static OBJ_LOCAL_LAN OBJ_LOCAL_LAN destination static OBJ_REMOTE_VPN_LAN OBJ_REMOTE_VPN_LAN no-proxy-arp route-lookup

! ===== 4. Section 1: Manual Twice NAT (Policy Dynamic PAT) =====
object network OBJ_DMZ_HOST
 host 192.168.10.50

object service OBJ_SVC_TELNET
 service tcp destination eq 23

object service OBJ_SVC_MAPPED_2323
 service tcp destination eq 2323

nat (inside,dmz) section 1 source dynamic OBJ_INSIDE_SUBNET interface destination static OBJ_DMZ_HOST OBJ_DMZ_HOST service OBJ_SVC_MAPPED_2323 OBJ_SVC_TELNET
```

### 2. Cisco FTD (FMC GUI): NAT ポリシー構成手順

1. **Threat Defense NAT Policy の作成**:
   * **Devices > NAT** へ移動し、**New Policy > Threat Defense NAT** を選択。
   * **Name**: `FTD_NAT_POLICY` を入力し、対象 FTD デバイスを割り当て。
2. **Section 1 (Before Auto) Manual NAT ルールの追加**:
   * **Add Rule** をクリック。
   * **NAT Rule**: `Manual NAT Rule`, **Type**: `Static`。
   * **Interface Objects**: Source: `inside-zone`, Destination: `outside-zone`。
   * **Translation**:
     * Original Source: `OBJ_LOCAL_LAN`, Translated Source: `OBJ_LOCAL_LAN`
     * Original Destination: `OBJ_REMOTE_VPN`, Translated Destination: `OBJ_REMOTE_VPN`
   * **Advanced**: `Do not proxy ARP on Destination Interface` および `Perform Route Lookup for Destination Interface` にチェックを入れて保存。
3. **Section 2 (Auto NAT) ルールの追加**:
   * **Objects > Object Management > Network** で内部ネットワークオブジェクトを開く。
   * オブジェクトの編集画面（または NAT ポリシー編集画面）で Auto NAT を追加:
     * **Type**: `Dynamic`, **Original Source**: `OBJ_INSIDE_NET`, **Translated Source**: `Destination Interface IP`。
4. **Deploy の実行**:
   * 設定保存後、対象 FTD へ Deploy を適用。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **NAT ルール一覧・セクション別順序・ヒット数表示** | ASA / FTD CLI | <code>show nat detail</code> |
| **アクティブな xlate (NAT変換) テーブル表示** | ASA / FTD CLI | <code>show xlate detail</code> |
| **特定 IP / ポートの xlate セッション検索** | ASA / FTD CLI | <code>show xlate debug</code> / <code>show xlate local 10.1.1.100</code> |
| **パケットフロー追跡 (NAT 評価フェーズ確認)** | ASA / FTD CLI | <code>packet-tracer input inside tcp 10.1.1.10 12345 203.0.113.100 80 detail</code> |
| **NAT ルールの running-config 出力表示** | ASA CLI | <code>show running-config nat</code> / <code>show running-config object</code> |
| **FTD (FMC) デプロイ済み LINA NAT 構成確認** | FTD CLI | <code>system support diagnostic-cli</code> ➔ <code>show nat detail</code> |
| **リアルタイム NAT パケットデバッグ** | ASA CLI | <code>debug nat</code> / <code>debug ip packet</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **VPN 通信が暗号化されず平文で送出またはドロップされる** | Section 2 の Auto PAT が先にマッチし、送信元 IP が変換されている | ASA/FTD: <code>show nat detail</code> | VPN 用の Identity NAT (No-NAT) を Section 1 (Before Auto) に配置する。 |
| **外部から公開サーバーへアクセスしても応答がない** | インバウンド ACL / ACP で Mapped IP を宛先にして拒否されている | ASA/FTD: <code>packet-tracer</code> | ACL / ACP の destination 条件を Mapped IP ではなく **Real IP（内部実IP）** に修正する。 |
| **同一サブネット内の通信で ARP テーブルが混乱し不達になる** | Identity NAT や Wide-range NAT による Proxy ARP の副作用 | ASA/FTD: <code>show arp</code> | NAT ルールに **`no-proxy-arp`** オプションを付与して Proxy ARP を無効化する。 |
| **Twice NAT 設定後、パケットが間違ったポートから出力される** | 明示的インターフェイス指定による Egress Route バイパス | ASA/FTD: <code>packet-tracer</code> | NAT ルールに **`route-lookup`** キーワードを追加し、ルーティングテーブルに委ねる。 |
| **外部 DNS を参照すると内部サーバーのパクリック IP が返る** | Static NAT ルールで DNS Rewrite (Doctoring) 未設定 | ASA/FTD: <code>show service-policy inspect dns</code> | Static NAT ルールに **`dns`** オプションを追加し、DNS Inspection を有効化する。 |
| **大量のクライアント接続時に PAT ポート枯渇エラーが発生** | 単一 IP アドレスによる 64,512 ポート上限到達 | ASA/FTD: <code>show xlate count</code> | `pat-pool` (複数 IP パットプール) や `flat round-robin` / `block-allocation` 構成に変更する。 |

---

## ⚠ 制限事項

* **非サポート構成・プロトコル**:
  * **Multicast Traversal**: PIM レジスタパケットの内部ペイロードに対する NAT 変換は非サポート。
  * **Transparent Mode での Interface PAT**: 透過モードの ASA / FTD ではインターフェイス IP を使用した Dynamic PAT は構成不可（BVI または専用 IP プールが必要）。
  * **Twice NAT と DNS Rewrite の併用**: 送信元と宛先を同時に指定する Twice NAT ルールでは `dns` オプションは非サポート。
* **オブジェクト制限**:
  * 単一の NAT ネットワークオブジェクトに含められる IP アドレス数は最大 **131,072 個**。

---

## 🔄 他技術との関連

* **Access Control (ACL / ACP)**: ASA 8.3+ / FTD では、インバウンド ACL の宛先条件に **UN-NAT 後の Real IP（実IP）** を記述します。
* **Routing**: NAT 評価（UN-NAT）の直後に Egress インターフェイスを判定する Route Lookup が実行されます。
* **VPN (Site-to-Site / AnyConnect)**: VPN トンネルを通過するトラフィックは、暗号化 ACL にマッチさせるために Identity NAT (Section 1) で原形保持させる必要があります。
* **High Availability & Clustering**: アクティブ機器で生成された `xlate` 変換テーブルは、Stateful Failover リンクを介してスタンバイ機器へリアルタイム同期されます。

---

## 🧩 比較表

### 1. NAT セクション（Section 1 vs Section 2 vs Section 3）比較

| 項目 | Section 1 (Manual NAT / Before Auto) | Section 2 (Auto NAT / Object NAT) | Section 3 (Manual NAT / After Auto) |
| :--- | :--- | :--- | :--- |
| **定義コマンド** | `nat (...)` (Before Auto) | `object network` 配下の `nat` | `nat (...) after-auto` |
| **評価優先度** | **第 1 優先 (最高)** | **第 2 優先 (中位)** | **第 3 優先 (最低)** |
| **ルールソート** | **ユーザー記述順 (Line Number)** | **システム自動ソート (Static/Prefix長順)** | **ユーザー記述順 (Line Number)** |
| **Twice NAT** | サポート (送信元・宛先同時指定) | 非サポート (単一アドレスのみ) | サポート (送信元・宛先同時指定) |
| **用途** | Identity NAT, Policy NAT, Overlapping NAT | 通常の Static NAT/PAT, Dynamic PAT | 最終フォールバック用 PAT ルール |

### 2. Static NAT vs Dynamic NAT vs Dynamic PAT

| 項目 | Static NAT (1:1) | Dynamic NAT (1:1 Dynamic) | Dynamic PAT (N:1 / Overload) |
| :--- | :--- | :--- | :--- |
| **マッピング比率** | 1 対 1 (固定) | 1 対 1 (動的プールから一時割当) | 多 対 1 (L4 ポート番号で識別) |
| **双方向通信 (Bi-directional)** | **双方向可能** (外部からの開始可) | 送信元からの通信で動的エントリ生成 | 原則インサイド開始通信のみ |
| **xlate エントリ** | 永続的 (Static xlate) | アイドルタイムアウトで消滅 | アイドルタイムアウトで消滅 |
| **主な用途** | Web / Mail サーバーのパブリック公開 | パブリック IP プールがある拠点アクセス | 一般クライアント端末の Internet 接続 |

---

## 💡 ベストプラクティス

1. **基本ルールは Auto NAT、特殊条件のみ Manual NAT を使用**:
   * 単純な 1:1 Static NAT や一般的な Dynamic PAT には、設定が簡潔でトラブルの少ない **Auto NAT (Section 2)** を使用します。
   * VPN No-NAT、Policy NAT、Twice NAT、Port Block Allocation などの高度な要件にのみ **Manual NAT (Section 1)** を使用します。
2. **`no-proxy-arp` と `route-lookup` の明示的付与**:
   * Manual NAT ルールを作成する際は、不用意な ARP 応答による障害を防ぐために **`no-proxy-arp`** を、正しく Egress ルーティングさせるために **`route-lookup`** を標準で付与します。
3. **PAT Pool の枯渇対策**:
   * 高密度な VDI や LAN 環境では、単一 IP PAT ではなく `pat-pool` (IP 範囲) を指定し、`round-robin` または `flat` オプションを適用してポート枯渇を防ぎます。

---

## 📝 ラボ学習・設定サンプル例

### 例1: Static Auto NAT による内部 DMZ Web サーバーのパブリック公開と DNS Rewrite

#### 問題
ASA1 の DMZ 上にある Web サーバー (`192.168.2.100`) を、パブリック IP `203.0.113.100` として外部（`outside`）へ静的に公開せよ。外部 DNS サーバーが返却する A レコードを内部クライアント通信時に内部 IP へ透過変換する DNS Rewrite を有効化せよ。

#### 要件
* Auto NAT (Section 2) として構成すること。
* 外部からの HTTP/HTTPS アクセスをインバウンド ACL で許可すること。

#### 設定例

```bash
! ネットワークオブジェクトの定義と Static Auto NAT + DNS Rewrite
object network OBJ_DMZ_WEB_SERVER
 host 192.168.2.100
 nat (dmz,outside) static 203.0.113.100 dns

! インバウンド ACL 構成 (※宛先は Real IP を指定)
access-list OUTSIDE_IN extended permit tcp any host 192.168.2.100 eq 80
access-list OUTSIDE_IN extended permit tcp any host 192.168.2.100 eq 443

access-group OUTSIDE_IN in interface outside
```

#### 検証方法
* `show nat detail` を実行し、Section 2 に Auto NAT ルールが登録されていることを確認。
* 外部端末から `203.0.113.100` 宛てに HTTP 接続し、`show xlate detail` で `flags s D` (Static, DNS) が付与された変換テーブルを確認。

---

### 例2: Static Auto PAT による特定ポート (Port Forwarding / 2323 -> 23) 転送

#### 問題
外部クライアントが ASA1 の `outside` インターフェイス IP (`203.0.113.1`) の TCP ポート `2323` へアクセスした際、DMZ 上の Telnet サーバー (`192.168.2.50`) の TCP ポート `23` へポート変換して転送せよ。

#### 要件
* Auto NAT (Section 2) を使用すること。

#### 設定例

```bash
object network OBJ_TELNET_SERVER
 host 192.168.2.50
 nat (dmz,outside) static interface service tcp 23 2323

access-list OUTSIDE_IN extended permit tcp any host 192.168.2.50 eq 23
access-group OUTSIDE_IN in interface outside
```

#### 検証方法
* `show nat` を実行し、`service tcp telnet 2323` の変換ルールを確認。
* 外部から `telnet 203.0.113.1 2323` を実行し、`show conn` で DMZ サーバーのポート 23 へセッションが到達していることを確認。

---

### 例3: Dynamic Auto PAT (インターフェイス PAT & バックアップ PAT プール)

#### 問題
内部 `inside` ネットワーク (`10.1.0.0/16`) から外部 `outside` への通信を、パブリック IP プール (`203.0.113.200 - 203.0.113.205`) を使用して動的 PAT 変換せよ。IP プールが枯渇した場合は、`outside` インターフェイス IP をフォールバック（Fallback）として使用せよ。

#### 要件
* FMC / FTD または ASA の Auto NAT で構成すること。

#### 設定例

```bash
! PAT プールオブジェクトの作成
object network OBJ_PAT_POOL
 range 203.0.113.200 203.0.113.205

! 内部ネットワークオブジェクトおよび Dynamic PAT + Interface Fallback 構成
object network OBJ_INSIDE_LAN
 subnet 10.1.0.0 255.255.0.0
 nat (inside,outside) dynamic pat-pool OBJ_PAT_POOL interface
```

#### 検証方法
* `show running-config nat` で `dynamic pat-pool OBJ_PAT_POOL interface` の記述を確認。
* 内部端末から大量通信を発生させ、`show xlate` でプールアドレスおよびインターフェイス IP が使用されていることを確認。

---

### 例4: Dynamic Auto NAT から Interface PAT へのフォールバック (Pool Exhaustion)

#### 問題
内部サブネット `10.2.0.0/24` からの通信に対し、1:1 Dynamic NAT プール (`203.0.113.210 - 203.0.113.212`) からアドレスを動的割り当てよ。プール内の全 IP アドレス（3個）が使い切られた（Exhausted）後は、`outside` インターフェイス IP による Dynamic PAT へ自動移行させよ。

#### 要件
* 1:1 Dynamic NAT と Interface PAT を同一ルール内で組み合わせること。

#### 設定例

```bash
object network OBJ_DYN_NAT_POOL
 range 203.0.113.210 203.0.113.212

object network OBJ_INSIDE_SUBNET_2
 subnet 10.2.0.0 255.255.255.0
 nat (inside,outside) dynamic OBJ_DYN_NAT_POOL interface
```

#### 検証方法
* 4 台以上の端末から同時に外部へ通信を開始し、最初の 3 台には 1:1 変換（flags `i`）、4 台目以降には Interface PAT（flags `ri`）が割り当てられることを `show xlate` で確認。

---

### 例5: Manual Twice NAT (Section 1: 重複ネットワーク間通信の双方向変換)

#### 問題
企業統合に伴い、自社 `10.1.1.0/24`（`inside`）と統合先他社 `10.1.1.0/24`（`outside`）のアドレス空間が完全に重複している。自社端末 `10.1.1.50` が他社サーバー `10.1.1.200` へアクセスする際、自社送信元を `172.16.1.50` に、他社宛先を `172.16.2.200` に相互変換（Twice NAT）して通信を成立させよ。

#### 要件
* Section 1 (Manual Twice NAT) で構成すること。

#### 設定例

```bash
! オブジェクトの定義
object network OBJ_LOCAL_REAL_HOST
 host 10.1.1.50

object network OBJ_LOCAL_MAPPED_HOST
 host 172.16.1.50

object network OBJ_REMOTE_MAPPED_HOST
 host 172.16.2.200

object network OBJ_REMOTE_REAL_HOST
 host 10.1.1.200

! Section 1 Manual Twice NAT の構成
nat (inside,outside) section 1 source static OBJ_LOCAL_REAL_HOST OBJ_LOCAL_MAPPED_HOST destination static OBJ_REMOTE_MAPPED_HOST OBJ_REMOTE_REAL_HOST no-proxy-arp route-lookup
```

#### 検証方法
* 自社端末 `10.1.1.50` から `ping 172.16.2.200` を実行。
* `show xlate detail` で `flags static, twice, net-to-net` が付与された双方向変換エントリを確認。

---

### 例6: Manual Identity NAT (Section 1: AnyConnect / IPsec VPN トラフィックの NAT 免除)

#### 問題
内部ネットワーク `10.0.0.0/8` からインターネット宛ての通信は `outside` インターフェイス IP で Dynamic PAT されている。AnyConnect ユーザープール (`192.168.254.0/24`) および IPsec VPN 拠点 (`172.16.0.0/12`) 宛てのトラフィックが Dynamic PAT に引っかからないよう、Identity NAT (No-NAT) を構成せよ。

#### 要件
* Section 1 (Before Auto) に配置し、Proxy ARP を無効化すること。

#### 設定例

```bash
object-group network OG_INTERNAL_NETS
 network-object 10.0.0.0 255.0.0.0

object-group network OG_VPN_DESTINATIONS
 network-object 192.168.254.0 255.255.255.0
 network-object 172.16.0.0 255.240.0.0

! Section 1 Identity NAT (No-NAT) 構成
nat (inside,outside) section 1 source static OG_INTERNAL_NETS OG_INTERNAL_NETS destination static OG_VPN_DESTINATIONS OG_VPN_DESTINATIONS no-proxy-arp route-lookup
```

#### 検証方法
* `show nat detail` を実行し、Section 1 の行番号 1 に Identity NAT ルールが存在し、ヒットカウンタ (`translate_hits`) が増加することを確認。

---

### 例7: Dynamic Policy NAT (特定の宛先/ポート条件に基づく動的 PAT)

#### 問題
内部 LAN `10.1.0.0/16` から外部 Web サーバー群 (`203.0.113.0/24`) の TCP ポート `80/443` 宛ての通信のみを専用パブリック IP `203.0.113.50` で動的 PAT 変換せよ。それ以外の宛先宛て通信は通常の標準 PAT を適用させよ。

#### 要件
* Section 1 で Service オブジェクトを組み合わせた Dynamic Policy PAT を作成すること。

#### 設定例

```bash
object network OBJ_INSIDE_LAN
 subnet 10.1.0.0 255.255.0.0

object network OBJ_WEB_DESTINATIONS
 subnet 203.0.113.0 255.255.255.0

object network OBJ_PAT_SPECIFIC_IP
 host 203.0.113.50

object service OBJ_SVC_HTTP
 service tcp destination eq 80

object service OBJ_SVC_HTTPS
 service tcp destination eq 443

! Dynamic Policy PAT ルールの追加
nat (inside,outside) section 1 source dynamic OBJ_INSIDE_LAN OBJ_PAT_SPECIFIC_IP destination static OBJ_WEB_DESTINATIONS OBJ_WEB_DESTINATIONS service OBJ_SVC_HTTP OBJ_SVC_HTTP
nat (inside,outside) section 1 source dynamic OBJ_INSIDE_LAN OBJ_PAT_SPECIFIC_IP destination static OBJ_WEB_DESTINATIONS OBJ_WEB_DESTINATIONS service OBJ_SVC_HTTPS OBJ_SVC_HTTPS
```

#### 検証方法
* クライアントから HTTP/HTTPS 通信を発生させ、`show xlate` で送信元が `203.0.113.50` に変換されていることを確認。

---

### 例8: FTD (FMC) における Dynamic Manual PAT と Port Block Allocation (PBA)

#### 問題
FTD1 において、高密度な内部クライアントネットワーク (`10.50.0.0/16`) に対し、キャリアグレード NAT (CGNAT) 方式の **Port Block Allocation (PBA)** を使用した動的 PAT を構成せよ。各クライアント IP に対してブロックサイズ 512 ポートを静的に一括割り当てよ。

#### 要件
* FMC Threat Defense NAT Policy で PAT Pool の Block Allocation オプションを設定すること。

#### 設定例

```text
1. FMC: Objects > Object Management > Network:
   - OBJ_INSIDE_10_50 = 10.50.0.0/16
   - OBJ_CGNAT_POOL = 203.0.113.100 - 203.0.113.105 (Range)
2. Devices > NAT > FTD NAT Policy 編集 > Add Rule:
   - NAT Rule: Manual NAT Rule, Type: Dynamic
   - Interface Objects: Source: inside-zone, Destination: outside-zone
   - Translation:
     - Original Source: OBJ_INSIDE_10_50
     - Translated Source: PAT Pool -> OBJ_CGNAT_POOL
   - PAT Pool タブ:
     - Select: Port Block Allocation
     - Block Size: 512
3. Save & Deploy.
```

#### 検証方法
* FTD CLI にて `show nat detail` を実行し、`PAT Pool Port Block Allocation` のパラメータを確認。

---

### 例9: Section 3 (Manual NAT After Auto) を使用した最終フォールバック PAT

#### 問題
内部の各種専用セグメント（DMZ, Guest, Internal）に対し個別の Auto NAT ルールが定義されている。これらどの Auto NAT ルールにもマッチしなかったトラフィックに対する最終救済（Fallback）として、`outside` インターフェイス IP で動的 PAT 変換するグローバルルールを Section 3 に作成せよ。

#### 要件
* `after-auto` キーワードを使用すること。

#### 設定例

```bash
object network OBJ_ANY_INTERNAL
 subnet 0.0.0.0 0.0.0.0

! Section 3 (After Auto) Manual NAT ルールの作成
nat (any,outside) after-auto source dynamic OBJ_ANY_INTERNAL interface
```

#### 検証方法
* `show nat detail` を実行し、`Manual NAT Policies (Section 3)` にルールが配置されていることを確認。

---

### 例10: FTD (FDM) における Bi-directional Static Auto NAT の構成

#### 問題
FDM (Firepower Device Manager) で管理されている FTD 上において、内部データベースサーバー (`192.168.10.25`) をパブリック IP `203.0.113.25` へ双方向 Static Auto NAT 変換せよ。

#### 要件
* FDM Web UI より設定を行うこと。

#### 設定例

```text
1. FDM Web UI: Objects > Networks に移動し、オブジェクトを作成:
   - db_server_real = 192.168.10.25
   - db_server_mapped = 203.0.113.25
2. Policies > NAT > Add NAT Rule (+ アイコン):
   - Title: DB_Server_Static_NAT
   - Create Rule For: Auto NAT
   - Type: Static
   - Source Interface: inside
   - Destination Interface: outside
   - Original Address: db_server_real
   - Translated Address: db_server_mapped
3. Advanced Options タブ:
   - "Translate DNS replies that match this rule" にチェックを入れる (必要に応じて)
4. Save & Deploy Changes.
```

#### 検証方法
* FTD CLI (Diagnostic CLI) で `show nat` および `show xlate` を実行し、`Auto NAT Policies (Section 2)` に Static 変換が反映されていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ASA1 上で以下の NAT コンフィグが適用されている。内部ホスト `10.1.1.50` から外部サーバー `203.0.113.10` 宛てに通信を発生させた際、送信元 IP はどのように変換されるか。

```bash
object network OBJ_HOST_A
 host 10.1.1.50
 nat (inside,outside) dynamic 198.51.100.10

nat (inside,outside) section 1 source static OBJ_HOST_A OBJ_HOST_A destination static any any
```

* A) `198.51.100.10` に変換される。
* B) `10.1.1.50` のまま変換されない（Identity NAT）。
* C) `outside` インターフェイス IP に PAT 変換される。
* D) パケットは NAT テーブル不一致でドロップされる。

**正解**: **B**
**解説**: Section 1 (Manual NAT) の Identity NAT ルールが最高優先度で評価されるため、Section 2 (Auto NAT) の動的変換ルールより先にマッチし、`10.1.1.50` のまま変換されず通過します。

---

### 2. トラブルシュート問題

**問題**: FTD1 において、内部 DMZ サーバー (`172.16.1.100`) をパブリック IP (`203.0.113.100`) へ Static Auto NAT で公開した。外部クライアントから `203.0.113.100` 宛てに HTTP リクエストを送信したが接続がタイムアウトする。`packet-tracer` を実行したところ、以下のフェーズでドロップが発生していた。原因として適切なものはどれか。

```text
Phase: 3
Type: ACCESS-LIST
Subtype: log
Result: DROP
Config:
access-list CSM_FW_ACL_ line 5 advanced permit tcp any host 203.0.113.100 eq 80
```

* A) NAT ルールが Section 3 に配置されているため。
* B) Access Control Policy (ACP) の許可ルールの宛先 IP に Mapped IP (`203.0.113.100`) が指定されており、Real IP (`172.16.1.100`) が指定されていないため。
* C) Proxy ARP が有効化されているため。
* D) `route-lookup` オプションが指定されていないため。

**正解**: **B**
**解説**: ASA OS 8.3+ および FTD では、アクセス制御ルール（ACL / ACP）の宛先条件は変換後のパブリック IP ではなく **UN-NAT 後の Real IP（実IP: 172.16.1.100）** を指定しなければドロップされます。

---

### 3. Design 問題

**問題**: 2 つの拠点（拠点 A: `10.1.0.0/16`、拠点 B: `10.1.0.0/16`）を Route-Based IPsec VPN で接続する際、両拠点の IP アドレス空間が重複している。両拠点の端末同士がスムーズに相互通信できるようにするための NAT 設計手法として最も適切なものはどれか。

* A) 各拠点の ASA / FTD で Section 2 に Auto PAT を構成する。
* B) 各拠点の ASA / FTD で Section 1 に 仮想サブネット（例: `172.16.1.0/16`）を用いた Manual Twice NAT を構成する。
* C) インターフェイス PAT に Fallback 構成を追加する。
* D) DNS Rewrite オプションを全 Auto NAT に有効化する。

**正解**: **B**
**解説**: 重複ネットワーク（Overlapping Networks）間での VPN 通信では、送信元と宛先のアドレス空間を同時に相互変換できる **Manual Twice NAT (Section 1)** を使用するのが標準設計です。

---

### 4. 実装問題

**問題**: ASA1 において、内部 LAN `10.1.1.0/24` から外部への通信を Dynamic PAT 変換しつつ、内部 Web サーバー (`10.1.1.10`) を外部パブリック IP `203.0.113.10` として公開したい。外部 DNS サーバーを参照する内部クライアントが FQDN で Web サーバーにアクセスした際、パブリック IP ではなく直接内部 IP (`10.1.1.10`) で通信できるようにするための正しい設定はどれか。

* A) `nat (inside,outside) dynamic interface` に `dns` オプションを追加する。
* B) `object network OBJ_WEB_SERVER` 配下の `nat (inside,outside) static 203.0.113.10` に **`dns`** オプションを追加する。
* C) Twice NAT ルールを作成し `dns-rewrite` を付与する。
* D) DNS サーバー上で Static Route を作成する。

**正解**: **B**
**解説**: DNS Doctoring / Rewrite 機能は、Static NAT / Static Auto NAT ルールの配下に **`dns`** キーワードを付与することで有効化され、外部 DNS 応答内の A レコードが Real IP へ自動書き換えられます。

---

### 5. コンフィグ読解問題

**問題**: ASA1 上で以下の 2 つの Auto NAT オブジェクトが定義されている。ASA 内部アルゴリズムにより、Section 2 (Auto NAT) 内で先に評価されるルールはどちらか。

```bash
object network OBJ_NET_A
 subnet 10.1.0.0 255.255.0.0
 nat (inside,outside) static 203.0.113.0

object network OBJ_HOST_B
 host 10.1.1.100
 nat (inside,outside) static 203.0.113.100
```

* A) `OBJ_NET_A` (コンフィグで先に記述されているため)
* B) `OBJ_HOST_B` (ホストアドレス /32 の方が サブネット /16 より プレフィックス長が長いため)
* C) ランダムに決定される。
* D) アルファベット順により `OBJ_HOST_B` が優先される。

**正解**: **B**
**解説**: Auto NAT (Section 2) セクション内では、記述順ではなく内部アルゴリズムによってルールが並び替えられます。Static 同士の比較では、**プレフィックス長がより長いオブジェクト（/32 ホスト）** が最優先で評価されます。

---

## 🔗 参考リソース

* [Cisco ASA Series Firewall Configuration Guide - Network Address Translation (NAT)](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/firewall/asa-94-firewall-config/nat-overview.html)
* [Cisco Secure Firewall Threat Defense Configuration Guide (FMC) - NAT Rules](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/nat_rules_for_threat_defense.html)
* [Cisco Command Reference - nat command on ASA](https://www.cisco.com/c/en/us/td/docs/security/asa/command/reference/fps/command-reference/n.html#pgfId-1845187)
* [Cisco Technical Notes: ASA 8.3+ NAT Rule Order and Operation](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/116388-config-nat-rules-00.html)
* [Cisco Live: Deep Dive into Cisco ASA / FTD NAT Architectures (BRKSEC-3020)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * NAT の評価順序 **Section 1 (Manual) ➔ Section 2 (Auto) ➔ Section 3 (Manual After-Auto)** は CCIE ラボ試験の最頻出項目です。
  * Auto NAT 内のソート順（Static > Dynamic、Prefix長が長い順）と、Manual NAT 内のソート順（完全な記述ライン順）の違いを体で覚えておきましょう。
* **注意点**:
  * ラボ試験で「VPN トラフィックが導通しない」トラブルシューティング問題の約 8 割は、**Identity NAT (No-NAT) が設定されていないか、Auto PAT より下に配置されていること** が原因です。`show nat detail` コマンドで位置関係を真っ先に確認する癖をつけてください。

## 📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

