---
layout: default
title: 1.7.b-Evasion
nav_order: 2
parent: 1.7-Attack-detection
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.7.b Evasion techniques

Cisco Secure Firewall Threat Defense (FTD)、Management Center (FMC)、Cisco ASA、および Cisco IOS/IOS-XE における **Evasion techniques（回避技術の検知と緩和 / Normalization）** は、攻撃者が IDS/IPS、次世代ファイアウォール（NGFW）、サンドボックス、プロトコルインスペクションエンジンによる不審トラフィックや悪意あるシグネチャの検知を意図的に迂回（Bypass / Evade）するために用いる多種多様な難読化・偽装・プロトコル分解技術に対し、インライン上でトラフィックの正規化（Normalization）やストリーム再構築（Reassembly）を実施して確定・遮断する包括的防御フレームワークです。

CCIE Security v6.1 Blueprint **1.7.b Evasion techniques** では、L3/L4（IP フラグメンテーション、TCP ストリーム分割・オーバーラップ）から L7 アプリケーション層（HTTP URL エンコーディング、Chunked 転送、DCE/RPC 断片化、MIME/Base64）に至る各種 Evasion 手法の動作メカニズム、Snort エンジンの **Network Analysis Policy (NAP)** や **Inline Normalization Preprocessor**、**Target-Based IP Defragmentation / TCP Stream Reassembly**、ASA/FTD の **TCP Map / IP Fragment / MPF 構造**、および SSL/TLS 復号とのバインド構成など、実効性の高い解析・設定・検証・トラブルシューティング能力が厳格に評価されます。

---

## 📘 概要

* **機能概要**:
  * **IP フラグメンテーション回避（IP Fragmentation Evasion）**: 攻撃パケットを細切れの IP フラグメントに分割し、オフセット値の重複（Overlapping）や順序入れ替え（Out-of-Order）を発生させて、ステートレスな IDS/IPS を通過した後にターゲット OS 上で悪意のあるペイロードを自動展開させる攻撃。これを **IP Defragmentation Preprocessor**（Target-Based Frag3）で先回りして正しく仮想再構築（Virtual Reassembly）する。
  * **TCP ストリーム分割・異常フラグ回避（TCP Stream / Segmentation Evasion）**: 攻撃シグネチャを複数の TCP セグメントへ細分化して送信したり、TCP ウインドウサイズ調整、乱立フラグ（SYN/RST/FIN 同時オン）、PAWS（Protection Against Wrapped Sequence numbers）の悪用を行う攻撃。これを **TCP Stream Preprocessor** (Stream5/Stream6) や **Inline Normalization Preprocessor** で正規化・再結合する。
  * **アプリケーション層エンコーディング・難読化（Protocol / Application Normalization）**:
    * **HTTP Evasion**: URL ％エンコーディング (`%20`, `%2f`)、Unicode/UTF-8 二重エンコーディング、Path Traversal (`/../..`)、Chunked Transfer Encoding によるペイロード分割。これらを **HTTP Inspect Preprocessor** でデコード・正規化して平文シグネチャと比較する。
    * **RPC / SMB / Mail Evasion**: DCE/RPC の断片化、SMTP/MIME の Base64/Quoted-Printable 難読化を **DCE/RPC / SMTP Preprocessor** でデコード。
  * **OS 固有の処理差分攻撃（Target-Based Evasion）**: 受信側 OS（Windows、Linux、FreeBSD、Solaris 等）ごとに TCP ACK 応答や IP フラグメント再構築の優先順位アルゴリズムが異なる点（Ptacek & Newsham 論文に起因する Insertion / Evasion 攻撃）を悪用する攻撃。FTD/FMC では **Target-Based Engine** により、宛先 IP の OS 属性（Host Profile）に応じた再構築ロジックを自動適用する。
* **利用目的**:
  * 攻撃者が IPS シグネチャのマッチングを回避するために仕込むヘッダー異常や難読化処理をデータプレーン上で事前に排除（Strip / Clear / Reassemble）し、セキュリティエンジンのシグネチャ検知率（Efficacy）を最大化する。
  * 攻撃者が IPS 検査をスルーさせたパケットをエンドポイント上で正常に復元・展開させるサイバー攻撃（Remote Code Execution, Buffer Overflow）を水際阻止する。
* **どのような場面で利用するか**:
  * 外部インターネットから DMZ や内部ネットワークの Web サーバー、ファイルサーバー、メールサーバー群宛ての着信トラフィックにおけるディープパケットインスペクション（DPI）。
  * 難読化されたマルウェア C2 通信やデータエクスフィルトレーション（情報漏洩）のインライン検知・遮断。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要回避カテゴリ** | **IP Fragmentation Evasion**、**TCP Stream/Segmentation Evasion**、**Application Encoding Evasion**、**Target-Based OS Diff Evasion** |
| **FTD 制御ポリシ** | **Network Analysis Policy (NAP)**: Preprocessors (Inspectors) のパラメーターおよび設定を集中定義 |
| **コア プレプロセッサ** | **Inline Normalization**、**IP Defragmentation (Frag3)**、**TCP Stream (Stream5/6)**、**HTTP Inspect**、**DCE/RPC**、**SMTP** |
| **ASA 防御機能** | **ASA FragGuard (`fragment chain`)**、**TCP Map (`sysopt npu / mpf tcp-map`)**、**MPF Protocol Inspection** |
| **Target-Based 連携** | FMC の **Host Profile (Network Discovery)** データベースと連動し、宛先 OS（例: Windows / Linux）の再構築挙動を動的模倣 |
| **復号依存性** | **HTTPS (TLS) 内の Evasion 攻撃** を検知・正規化するためには、**SSL Inspection Policy (Decrypt)** による平文化が必須 |
| **動作プレーン** | Snort エンジンの最前列（Data Acquisition: DAQ 直後、Rule Engine 前段）でパケットをデコード・正規化 |
| **制限事項** | 過度な TCP/IP 正規化処理（パケット書き換え）は、非標準的な TCP/IP スタックを持つレガシー制御機器（OT/IoT）の通信を破断させるリスクがある |

---

## 🏗 動作原理

### 1. Evasion (回避技術) と Normalization (正規化) のアーキテクチャ

攻撃者は IPS の検知エンジンと最終目的地である Target Host (OS) の「パケット解釈の差分」を悪用します。Normalization エンジンは、IPS 評価の直前にトラフィックを「Target Host が解釈するのと全く同一の標準形式」に強制変換（再構築・標準化）します。

```text
[ Attacker ] ─── Encoded / Fragmented / Split Evasion Packets ───► [ FTD (Data Plane) ]
                                                                          │
┌─────────────────────────────────────────────────────────────────────────┴────────┐
│ Firepower Threat Defense (FTD) Packet Processing Architecture                     │
│                                                                                  │
│  [ LINA Firewall Engine ]                                                        │
│   └── IP Fragment Checks (`fragment chain`) & TCP Map Verification               │
│            │                                                                     │
│            ▼                                                                     │
│  [ Snort Next-Gen Engine: Network Analysis Policy (NAP) ]                        │
│   ├── 1. Packet Decoder & Checksum Verification                                  │
│   │        └── Strip Invalid Headers, Verify L3/L4 Checksums                     │
│   │                                                                              │
│   ├── 2. Inline Normalization Preprocessor                                       │
│   │        ├── Strip Reserved Bits, Fix TCP Flags (Clear URG/FIN/RST anomalies)  │
│   │        └── Clear Don't Fragment (DF) & Option Padding Bytes                  │
│   │                                                                              │
│   ├── 3. IP Defragmentation Preprocessor (Target-Based Frag3)                    │
│   │        └── Reassemble Overlapping / Out-of-Order IP Fragments                │
│   │                                                                              │
│   ├── 4. TCP Stream Preprocessor (Stream5 / Stream6)                             │
│   │        └── Reassemble TCP Segments & Session State Alignment                 │
│   │                                                                              │
│   ├── 5. Application Preprocessors (HTTP Inspect, DCE/RPC, SMTP)                │
│   │        └── Decode %-Encoding, Unicode, Path Traversal, Chunked Transfers     │
│   │                                                                              │
│   └── 6. Snort Rule Engine Evaluation                                            │
│            └── Match Reassembled & Normalized Clean Payload against IPS Rules    │
│                     ├── Match Exploit Signature ──► Block / Drop Packet          │
│                     └── Generate Intrusion Event (Preproc GID:SID Alert) ──► FMC   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙ 動作シーケンス

1. **パケット着信と LINA L3/L4 初期検証 (ASA / FTD LINA)**:
   * トラフィックが着信すると、LINA ファイアウォールエンジンが IP フラグメントチェーンの上限 (`fragment chain 24`) や TCP オプションのサニティチェックを実行。
   * IP フラグメントが過多の場合やタイマーアウトしたパケットは、LINA レベルで早期ドロップ（ASP Drop）。
2. **Snort DAQ ＆ Packet Decoder (パケットデコード)**:
   * LINA から Snort エンジンへ渡されたパケットは、Packet Decoder により L2〜L4 ヘッダーが分解解析され、チェックサムエラーや異常なヘッダー拡張が検証されます。
3. **Inline Normalization Preprocessor (インライン正規化)**:
   * **TCP フラグ・ヘッダー調整**: 不正な TCP フラグ組み合わせ（例: SYN+RST、SYN+FIN）の補正、緊急ポインター (Urgent Pointer) の適正化、TCP アライメントパディングの初期化 (Zero-out)。
   * **IP ヘッダー調整**: Reserved ビットの削除、Don't Fragment (DF) ビットの消去設定処理。
4. **IP Defragmentation Preprocessor (Frag3 再構築)**:
   * 断片化された IP パケット（IP Fragments）が到着すると、Frag3 プレプロセッサは宛先 IP の OS 属性（Target-Based Policy: Windows、Linux、BSD 等）を参照。
   * 各 OS 特有のフラグメント再構築アルゴリズム（例: First-Match, Last-Match, BSD, Linux）に完全に合致させる形で**仮想再構築 (Virtual Reassembly)** を実行し、オーバーラップによる攻撃シグネチャの不可視化を無効化します。
5. **TCP Stream Preprocessor (Stream5 / Stream6 再構築)**:
   * 分割送信された TCP セグメントをセッションテーブル（TCP State Machine）に従って正しいシークエンス順序に整列・結合します。
   * ウィンドウサイズの不正変更や再送信パケットでのデータ改ざん（Data Tampering）を検出した場合、不正セグメントをドロップします。
6. **Application Layer Preprocessors (HTTP Inspect / L7 デコード)**:
   * **HTTP Inspect**: URI 内の `%2f` や `%20`、ダブル Hex エンコーディング、Unicode、Path Traversal (`/../../`)、`//`（重複スライティング）を標準の HTTP URI 文字列へ完全復元（Decode）。Chunked 転送のデータ部を結合。
   * **DCE/RPC ＆ Mail Inspect**: 断片化された RPC リクエストや Base64 エンコードされた添付ファイルを平文テキストにデコード。
7. **Snort Rule Engine マッチングと執行**:
   * 正規化およびストリーム再構築が完全に完了した平文データストリームに対して、Snort の侵入防御ルール（IPS Rules）が走査されます。
   * 隠蔽されていた脆弱性攻撃パターン（例: Buffer Overflow, RCE）が明確に露出し、IPS ルールにヒットして通信がインライン破棄（Block）されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **Network Analysis Policy (NAP) の選択とカスタム設定**:
   * **デフォルト NAP**: FMC には `Balanced Security and Connectivity`（標準）、`Connectivity Over Security`、`Security Over Connectivity`、`Maximum Detection` が用意されています。デフォルトでは `Balanced...` が全通信にバインドされます。
   * **試験要件**: 「Configure FTD to normalize HTTP URI encodings and reassemble TCP streams according to Linux OS target policy」などの要件が出題された場合、カスタム NAP を作成し、Access Control Policy の **`Advanced > Network Analysis and Intrusion Policies`** でアサインする知識が必要です。
2. **HTTP Inspect Preprocessor のデコードオプション設定**:
   * **URI Normalization**: `utf-8_bare_byte`、`double_decode`、`non_rfc_char`、`iis_unicode`、`directory_traversal`、`multiple_slash` などの検知・補正トグル。
   * **Chunked Encoding**: `chunk_length` の異常値チェックおよびレスポンス解凍。
3. **ASA / FTD MPF `tcp-map` による L4 Normalization (TCP 正常化)**:
   * ASA/FTD では MPF (`policy-map`) を介して **`tcp-map`** オブジェクトを適用し、TCP Evasion を無効化できます：
     * `check-retransmission`: 再送信パケットのデータ改ざんを検証・破棄。
     * `exceed-mss drop`: MSS 超過パケットをドロップ。
     * `invalid-ack drop`: 無効な ACK 番号パケットをドロップ。
     * `seq-past-window drop`: ウィンドウ範囲外のシークエンス番号パケットをドロップ。
     * `reserved-bits allow|drop`: TCP ヘッダーの予約ビット付きパケットの破棄。
4. **ASA Fragment Reassembly コマンド**:
   * **`fragment chain <count> <interface>`**: フラグメントチェーンの上限設定（デフォルト 24。`1` にすると実質フラグメント拒否）。
   * **`fragment timeout <seconds> <interface>`**: フラグメント再構築の保持タイムアウト（デフォルト 5 秒）。
   * **`fragment size <count> <interface>`**: フラグメントキューイング上限。
5. **HTTPS (TLS) 通信での Evasion 検知と SSL Inspection の必須関係**:
   * アプリケーション層（HTTP など）の Evasion 対策（HTTP Inspect Normalization）は、暗号化された HTTPS 通信内ではそのまま動作しません。
   * **SSL Inspection Policy (Decrypt - Resign)** をバインドして平文に戻したトラフィックに対してのみ、HTTP Inspect Preprocessor による URI 正規化や Chunked 解凍が機能します。

---

## 🛠 設定方法

### 1. FMC GUI: カスタム Network Analysis Policy (NAP) の作成と Normalization 設定

```text
[ Step 1: カスタム Network Analysis Policy の作成 ]
1. FMC UI から Policies > Access Control > Advanced に移動し "Network Analysis Policies" をクリック。
2. "Create Policy" を選択:
   - Name: NAP_CUSTOM_NORMALIZATION
   - Base Policy: Balanced Security and Connectivity
3. "Settings" タブを開く:
   - Inline Normalization Preprocessor ➔ Edit:
     - [✓] Normalize IPv4
     - [✓] Normalize Don't Fragment Bit
     - [✓] Normalize TCP Payload
     - [✓] Normalize/Clear Reserved Bits
   - IP Defragmentation ➔ Edit:
     - Target-Based Defragmentation Policies: Default Target Policy = Linux
   - TCP Stream Preprocessing ➔ Edit:
     - [✓] Perform Stream Reassembly
     - Target-Based Policy = Linux
4. "Save" をクリック。

[ Step 2: Access Control Policy への NAP のバインド ]
1. Policies > Access Control で対象 ACP を編集。
2. "Advanced" タブ ➔ "Network Analysis and Intrusion Policies" セクションを開く:
   - Default Network Analysis Policy: "NAP_CUSTOM_NORMALIZATION" を選択。
3. "Save" して対象 FTD へ Deploy。
```

---

### 2. ASA CLI: MPF `tcp-map` による L4 TCP Normalization (回避防御) の設定

```text
! ===== Step 1: 詳細な TCP 検証を行う tcp-map の定義 =====
tcp-map MAP_STRICT_NORMALIZATION
 check-retransmission                   ! 再送信パケットのデータ改ざんを検証・ドロップ
 exceed-mss drop                        ! MSS 超過パケットのドロップ
 invalid-ack drop                       ! 無効な ACK 番号のドロップ
 seq-past-window drop                   ! TCP ウィンドウ外シークエンス番号のドロップ
 tcp-options range 6 7 drop             ! 不正な TCP オプションの削除
 reserved-bits drop                     ! 予約ビット設定パケットのドロップ

! ===== Step 2: MPF で Class-Map ＆ Policy-Map へのバインド =====
access-list ACL_TCP_TRAFFIC extended permit tcp any any

class-map CM_TCP_TRAFFIC
 match access-list ACL_TCP_TRAFFIC

policy-map PM_TCP_NORMALIZATION
 class CM_TCP_TRAFFIC
  set connection advanced-options MAP_STRICT_NORMALIZATION

! ===== Step 3: インターフェイスへの適用 =====
service-policy PM_TCP_NORMALIZATION interface OUTSIDE
```

---

### 3. ASA / FTD CLI: IP Fragment (FragGuard) 制御パラメータの設定

```text
! ===== ASA / FTD LINA CLI での IP フラグメント設定 =====
fragment chain 1 OUTSIDE                ! OUTSIDE でのフラグメントチェーンを 1 に制限 (再構築無効化)
fragment chain 24 INSIDE                ! INSIDE での標準フラグメント許可
fragment timeout 3 OUTSIDE              ! フラグメント保持タイムアウトを 3 秒に縮小
fragment size 100 OUTSIDE               ! メモリバッファ上限を 100 パケットに制限
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI / ASA) | 表示確認ポイント |
| :--- | :--- | :--- |
| **Snort 3 エンジンでの Normalization カウンター確認** | <code>show statistics normalization</code> | TCP/IP/ICMP 正規化パケット数、ドロップ件数 |
| **IP Defragmentation / Frag3 統計確認** | <code>show statistics engine</code> / FMC Events | 再構築失敗、オーバーラップ検知カウンター |
| **ASA IP Fragment (FragGuard) 状態確認** | <code>show fragment</code> | インターフェイスごとの Chain、Size、Timeout、Reassembly 設定値 |
| **ASA MPF `tcp-map` 接続評価カウンター確認** | <code>show service-policy interface OUTSIDE</code> | `tcp-map` でドロップされた不正 TCP セグメント数 |
| **Snort リアルタイムデバッグログ追跡** | <code>system support firewall-engine-debug</code> | `the fragmentation preprocessed`, `the stream preprocessed` などの処理メッセージ |
| **FMC 上での Preprocessor イベント確認** | `Analysis > Intrusion > Events` (GID: 119/120/123/129) | GID 120 (HTTP Inspect)、GID 123 (IP Defrag)、GID 129 (Stream) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **HTTP Evasion ルールを設定したのに、%20 等を含む URI 攻撃が検知されず通過する** | HTTPS 通信に対して **SSL Inspection Policy (Decrypt) が設定されていない** ため、平文化されていない | FMC SSL Policy / <code>show service-policy</code> | SSL Inspection Policy で対象通信を `Decrypt - Resign` にアサインし、平文 HTTP を Snort に解読させる。 |
| **`tcp-map` で `check-retransmission` を適用後、一部のレガシーアプリ通信が突然ブロックされる** | 送信側スタックのバグにより、再送信時にパケットサイズやペイロードが微小に変化しており改ざんと判定された | <code>show service-policy interface</code> | 該当トラフィック用クラスで `check-retransmission` を無効化するか、対象通信を別クラスへ孤立化させる。 |
| **IP フラグメントパケットが FTD を一切通過できない** | ASA/FTD LINA レベルで `fragment chain 1` が設定されており、フラグメント再構築が全拒否されている | <code>show fragment</code> | 対象インターフェイスで `fragment chain 24`（デフォルト）に復元する。 |
| **異なるインターフェイスペアから届いた IP フラグメントが再構築されずにドロップする** | FTD の仕様上、同一パケットのフラグメント群は **同一の Interface Pair** に着信しなければ再構築できない | <code>show asp drop</code> | ネットワーク側で非対称ルーティング（Asymmetric Routing）を修正し、同一ファイアウォールインターフェイスに着信させる。 |
| **NAP で Inline Normalization を有効化したら FTD の CPU 使用率が高騰した** | 大容量ファイル転送トラフィックに対して過剰なペイロード正規化を全件実行している | <code>show cpu</code><br><code>show statistics normalization</code> | 信頼できるバックアップ通信や大容量レプリケーション通信を Prefilter Policy (`Fastpath`) で Snort 検査から除外する。 |

---

## ⚠ 制限事項

* **非対称ルーティング環境におけるストリーム再構築不可**:
  * IP フラグメントや TCP セグメントの再構築を行う際、パケットの一部が別のアプライアンスや別インターフェイスを経由する非対称ルーティング環境では、再構築（Reassembly）に失敗してパケットがドロップされます。
* **暗号化トラフィック (TLS/IPsec) への直接適用不可**:
  * 暗号化された IPsec トンネル内トラフィックや HTTPS 通信の内部ペイロードに対しては、ファイアウォール上で復号（Decryption）を実行しない限り、L7 アプリケーションプレプロセッサ（HTTP Inspect 等）による正規化処理を適用できません。
* **アクティブ応答 (Active Response) への影響**:
  * Normalization でヘッダーを書き換える際、インライン配置 (`Inline Mode`) で動作していなければパケットの補正・ドロップ（Drop/Reset）をリアルタイムに執行できません（受動的な Passive/SPAN 配置では警告アラート生成のみにとどまります）。

---

## 🔄 他技術との関連

* **SSL Inspection Policy**: HTTPS トラフィック内の暗号化ペイロードを解読し、HTTP Inspect Normalization エンジンへ平文引渡し。
* **Snort IPS Rules**: Normalization プレプロセッサによって解読・平文復元されたストリームに対して、シグネチャ（Exploit Rules）を正確に走査・適用。
* **Prefilter Policy (Fastpath)**: 信頼できる大容量通信（SAN バックアップ等）を DPI/Normalization の手前で Fastpath 転送し、CPU 負荷を削減。

---

## 🧩 比較表

### 1. IP Defragmentation (Frag3) vs TCP Stream Reassembly (Stream5/6) vs HTTP Normalization

| 比較項目 | IP Defragmentation (Frag3) | TCP Stream Reassembly (Stream5/6) | HTTP Normalization (HTTP Inspect) |
| :--- | :--- | :--- | :--- |
| **動作レイヤー** | **Layer 3 (IP)** | **Layer 4 (TCP)** | **Layer 7 (Application / HTTP)** |
| **解決する Evasion** | Overlapping IP Fragments, Teardrop | Split TCP Segments, Out-of-Order | %-Encoding, Unicode, Path Traversal |
| **動作概要** | 細分化された IP フラグメントをメモリ上で仮想的に 1 パケットへ結合 | 散発的な TCP セグメントをセッション順序に整列・結合 | URI 難読化や Chunked 転送の文字列を標準平文へ解読 |
| **OS 属性連動** | Target-Based (Windows, Linux, BSD 等) | Target-Based (Windows, Linux, BSD 等) | Web Server Specific (IIS, Apache 等) |

### 2. ASA Fragment Chain vs FTD Snort Frag3 Defragmentation

| 比較項目 | ASA Fragment Chain (LINA) | FTD Snort Frag3 (NAP) |
| :--- | :--- | :--- |
| **処理位置** | LINA データプレーン (L3/L4 接続前段) | Snort インスペクションエンジン (DPI 前段) |
| **設定方法** | CLI: `fragment chain <count> <interface>` | FMC UI: `Network Analysis Policy (NAP)` |
| **主な機能** | フラグメントバッファ上限・鎖数の制限 (DoS 防御) | ターゲット OS に応じた仮想再構築 ＋ 攻撃シグネチャ露出 |
| **デフォルト値** | `chain 24`, `size 200`, `timeout 5s` | Enabled (Balanced Security & Connectivity) |

---

## 💡 ベストプラクティス

1. **Target-Based Policies (Host Profile 連動) の活用**:
   * FMC の `Network Analysis Policy` において、TCP Stream および IP Defragmentation プレプロセッサのターゲットポリシーを **`Target-Based`**（Network Discovery で収集した OS 情報と動的連動）に設定し、宛先ホストの OS に完全一致した再構築ロジックを適用させます。
2. **SSL Inspection と HTTP Inspect Normalization のセット運用**:
   * Web サーバー保護および Web アクセス制御において、SSL Inspection Policy (`Decrypt - Resign` / `Decrypt - Known Key`) を必ず併用し、暗号化通信を平文に戻した状態で HTTP Normalization を実行させます。

---

## 📝 ラボ学習・設定サンプル例

### 例1: FMC でのカスタム Network Analysis Policy (NAP) 作成と Normalization 有効化

#### 問題
FMC 上でカスタム NAP `NAP_STRICT_NORM` を作成し、Inline Normalization および IP Defragmentation を有効化して Access Control Policy のデフォルト NAP としてバインドせよ。

#### 要件
* Base Policy: `Balanced Security and Connectivity` を使用すること。

#### 設定・操作手順

```text
1. FMC UI で Policies > Access Control > Advanced を開き "Network Analysis Policies" をクリック。
2. "Create Policy" を選択:
   - Name: NAP_STRICT_NORM
   - Base Policy: Balanced Security and Connectivity
3. Settings タブで "Inline Normalization Preprocessor" 編集画面を開き、全 Normalization 項目にチェックを入れて Save。
4. 対象 ACP の Advanced タブ ➔ "Network Analysis and Intrusion Policies" で Default Network Analysis Policy に "NAP_STRICT_NORM" をアサインして Deploy。
```

#### 検証方法
* FTD CLI で `show statistics normalization` を実行し、正規化カウンターが増加していることを確認。

---

### 例2: HTTP Inspect Preprocessor での URI Encoding 正常化設定

#### 問題
`NAP_STRICT_NORM` 内の HTTP Inspect Preprocessor において、`Multiple Slash` (`//`) および `Directory Traversal` (`/../..`) 難読化攻撃を検知・正規化するよう構成せよ。

#### 要件
* FMC NAP 設定画面から構成すること。

#### 設定・操作手順

```text
1. NAP_STRICT_NORM の編集画面を開き "HTTP Inspect" プレプロセッサを選択。
2. HTTP Server Configuration 編集画面を開く:
   - [✓] Normalize Multiple Slashes
   - [✓] Normalize Directory Traversal
3. Save を押し、ACP 経由で Deploy。
```

#### 検証方法
* 擬似攻撃端末から `curl [suspicious link removed]` を送信し、FTD 上で正規化されて IPS アラート (GID 120) が生成されることを確認。

---

### 例3: ASA MPF `tcp-map` による TCP 再送信データ改ざん (Data Tampering) 阻止

#### 問題
ASA において、TCP 再送信パケットでのデータ書き換え攻撃を防ぐため、`tcp-map` で `check-retransmission` を有効化し、`outside` インターフェイスの全 TCP トラフィックへ適用せよ。

#### 要件
* MPF を使用して構成すること。

#### 設定・操作手順

```text
ASA1(config)# tcp-map MAP_CHECK_RETRANS
ASA1(config-tcp-map)# check-retransmission
ASA1(config-tcp-map)# invalid-ack drop

ASA1(config)# class-map CM_ALL_TCP
ASA1(config-cmap)# match port tcp any

ASA1(config)# policy-map PM_TCP_SECURITY
ASA1(config-pmap)# class CM_ALL_TCP
ASA1(config-pmap-c)# set connection advanced-options MAP_CHECK_RETRANS

ASA1(config)# service-policy PM_TCP_SECURITY interface OUTSIDE
```

#### 検証方法
* `show service-policy interface OUTSIDE` を実行し、`MAP_CHECK_RETRANS` が適用されていることを確認。

---

### 例4: ASA/FTD 外部インターフェイスでの IP Fragment チェーン制限 (FragGuard)

#### 問題
外部 WAN インターフェイス `OUTSIDE` において、IP フラグメントによる DoS 攻撃を抑制するため、フラグメントチェーン上限を `1` に設定（実質フラグメント破棄）せよ。

#### 要件
* `fragment` コマンドを使用すること。

#### 設定・操作手順

```text
ASA1(config)# fragment chain 1 OUTSIDE
ASA1(config)# fragment timeout 2 OUTSIDE
```

#### 検証方法
* `show fragment` を実行し、`Interface: OUTSIDE Size: 200, Chain: 1, Timeout: 2` であることを確認。

---

### 例5: Snort 3 Target-Based TCP Stream Reassembly の構成

#### 問題
FTD (Snort 3) において、宛先 Linux サーバー宛ての TCP ストリーム再構築アルゴリズムを `Linux` ターゲットポリシーに設定せよ。

#### 要件
* NAP 内の TCP Stream Preprocessor 設定から構成すること。

#### 設定・操作手順

```text
1. FMC NAP 編集画面で "TCP Stream Preprocessing" を選択。
2. Target-Based Policy セクションで "Linux" を選択。
3. [✓] Perform Stream Reassembly にチェックを入れ Save。
4. Deploy を実行。
```

#### 検証方法
* FMC の Connection Events / Intrusion Events で、TCP ストリームが Linux 方式で正常再構築されて評価されたことを確認。

---

### 例6: HTTPS 通信内の HTTP Evasion 攻撃検知のための SSL Decryption 連携

#### 問題
HTTPS アクセス内に隠蔽された Path Traversal Evasion 攻撃を検知させるため、SSL Inspection Policy で復号処理を行ったうえで ACP にバインドせよ。

#### 要件
* SSL Action: `Decrypt - Resign`

#### 設定・操作手順

```text
1. Policies > Access Control > SSL で Outbound HTTPS に対する Decrypt - Resign ルールを作成。
2. 対象 ACP に作成した SSL Policy をバインド。
3. NAP で HTTP Inspect の Normalization が有効になっていることを確認し Deploy。
```

#### 検証方法
* `https://.../../etc/passwd` 宛てにリクエストを送信し、FTD が HTTPS を解読したうえで HTTP Inspect プレプロセッサ（GID 120）で警告・遮断することを確認。

---

### 例7: FTD CLI を用いた Normalization カウンターのリアルタイム確認

#### 問題
FTD CLI にログインし、現在アプライアンス上で執行されている TCP/IP 正規化処理の統計カウンターを表示させよ。

#### 要件
* `show statistics normalization` コマンドを使用すること。

#### 設定・操作手順

```bash
> show statistics normalization
```

#### 検証方法
* CLI 出力において、`IPv4 DF Flag Normalizations` や `TCP Segment Reassembly Normalizations` の数値を確認。

---

### 例8: DCE/RPC 断片化 Evasion 攻撃の検知設定

#### 問題
社内 SMB / Active Directory サーバー宛ての DCE/RPC トラフィックにおいて、パケット断片化による IPS 回避を検知・正規化するよう NAP を構成せよ。

#### 要件
* DCE/RPC Preprocessor を有効化すること。

#### 設定・操作手順

```text
1. FMC NAP 編集画面で "DCE/RPC Preprocessor" を選択。
2. [✓] Enabled にチェックを入れ、[✓] Reassemble Fragmented RPC Requests をオンにする。
3. Save して Deploy。
```

#### 検証方法
* 断片化された DCE/RPC 攻撃パケットがプレプロセッサによって結合され、IPS シグネチャにマッチしてブロックされることを確認。

---

### 例9: ASA MPF による TCP Window 外パケットの自動ドロップ

#### 問題
TCP セグメントのウインドウ外送信（`seq-past-window`）を利用したファイアウォール回避攻撃を止めるため、`tcp-map` でドロップ設定を追加せよ。

#### 要件
* `seq-past-window drop` コマンドを使用すること。

#### 設定・操作手順

```text
ASA1(config)# tcp-map MAP_DROP_OUT_OF_WINDOW
ASA1(config-tcp-map)# seq-past-window drop
ASA1(config)# policy-map PM_GLOBAL
ASA1(config-pmap)# class class-default
ASA1(config-pmap-c)# set connection advanced-options MAP_DROP_OUT_OF_WINDOW
```

#### 検証方法
* `show service-policy` を実行し、ウインドウ外パケットのドロップカウンターが増加することを確認。

---

### 例10: FMC REST API による Network Analysis Policy 一覧の取得

#### 問題
FMC REST API を使用して、登録されている全 Network Analysis Policy (NAP) の UUID 一覧を Curl コマンドで取得せよ。

#### 要件
* `/api/fmc_config/v1/domain/{domainUUID}/policy/networkanalysispolicies` エンドポイントを使用すること。

#### 設定・操作手順

```bash
# ===== Auth Token 取得後、NAP 一覧を GET =====
curl -k -X GET "https://fmc.corp.local/api/fmc_config/v1/domain/e276abec-e0f2-11e3-8169-6d9ed13b206f/policy/networkanalysispolicies" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json"
```

#### 検証方法
* 返却された JSON レスポンス内に、設定した NAP の名称および UUID が含まれていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**问题**: 攻撃者が Web サーバーに対して `[suspicious link removed]` という URL（%エンコーディングおよび Path Traversal の組み合わせ）を送信した。FTD の Snort エンジンにおいて、この難読化を解除して平文文字列にデコード・正規化するコンポーネントとして正しいものはどれか。

* A) Prefilter Fastpath Policy
* B) Network Analysis Policy (NAP) 内の HTTP Inspect Preprocessor
* C) Security Intelligence IP Feed
* D) CoPP (Control Plane Policing)

**正解**: **B**  
**解説**: **HTTP Inspect Preprocessor**（Network Analysis Policy 内）は、URL 内の %2f（`/`）や %2e（`.`）などの Hex エンコーディング、Unicode、Path Traversal (`/../`) を元通りの標準的な HTTP リクエスト文字列へデコード・正規化し、IPS ルールエンジンに渡す役割を担います。

---

### 2. トラブルシュート問題

**問題**: 管理者が ASA 上で詳細な TCP 検証を行うため `tcp-map` を定義し、`check-retransmission` を有効化して全トラフィックへ適用した。直後から、一部のレガシーなクライアントからのファイルアップロード通信が途中で切断される障害が発生した。原因として適切なものはどれか。

* A) レガシー端末の TCP スタックがパケット再送信時に不変であるべきデータペイロードを微小に変更して送信しており、ASA が「データ改ざん (Data Tampering)」と判断してドロップしたため。
* B) ASA の IPsec トンネルが切断されたため。
* C) FTD の License が期限切れになったため。
* D) BGP RTBH が発動したため。

**正解**: **A**  
**解説**: **`tcp-map` の `check-retransmission`** は、TCP パケット再送信時に初回送信データと再送信データが完全一致するかを検証します。非標準な TCP スタック（レガシー OS 等）が再送信時にペイロード内容を微小変化させると、Evasion 攻撃（データ改ざん）とみなされて ASA にドロップされます。

---

### 3. Design 問題

**問題**: 宛先サーバーの OS（Windows Server や Linux）によって、IP フラグメントや TCP セグメントの再構築時の重複優先順位（First-Match / Last-Match）が異なる点を利用した Insertion / Evasion 攻撃を完全に防御したい。FTD/FMC において推奨される最も効率的な設計手法はどれか。

* A) 全通信を無条件で Fastpath 設定にする。
* B) FMC の Network Discovery 機能で収集した Host Profile データベースと連携し、Network Analysis Policy の Preprocessor 再構築ポリシーを **`Target-Based`** に設定する。
* C) ASA で uRPF Strict Mode を有効化する。
* D) 全ポートで BFD を有効化する。

**正解**: **B**  
**解説**: **Target-Based Policies** を利用することで、FMC は宛先 IP アドレスの実際の OS 属性（Host Profile）を自動参照し、その Target OS が行うパケット再構築アルゴリズムと「全く同一の再構築処理」を Snort 上で実行するため、OS 間の処理差分を突いた Evasion 攻撃を完璧に封じ込めることができます。

---

### 4. 実装問題

**問題**: FTD CLI にログインし、アプライアンス上で実行された TCP/IP パケットの正規化（Don't Fragment ビット消去、TCP 予約ビットクリア、ペイロード調整等）の処理件数統計を表示させるための正確なコマンドはどれか。

* A) <code>show access-list</code>
* B) <code>show statistics normalization</code>
* C) <code>show service-policy</code>
* D) <code>show version</code>

**正解**: **B**  
**解説**: FTD CLI で **`show statistics normalization`** を実行すると、Snort エンジンが実行した各種 Header / Payload の正規化処理（Normalization）の統計カウンターを一括確認できます。

---

### 5. コンフィグ読解問題

**問題**: 暗号化された Web アクセス (`https://10.1.1.10/`) 内に仕込まれた Chunked Transfer Encoding や URI Unicode 難読化攻撃を FTD で検知・遮断したい。この構成において必至となる前処理として正しいものはどれか。

* A) Prefilter で Fastpath ルールを作成する。
* B) SSL Inspection Policy で該当 HTTPS トラフィックを復号 (`Decrypt - Resign`) して平文 HTTP ストリームに戻す。
* C) ASA 上で `fragment chain 1` を設定する。
* D) 内部 DNS サーバーを Cisco Umbrella へ変更する。

**正解**: **B**  
**解説**: アプリケーション層（HTTP Inspect 等）の Evasion 対策は、ペイロードが平文の状態でしか機能しません。HTTPS などの TLS 暗号化通信に対しては、**SSL Inspection Policy による復号 (Decrypt)** が不可欠です。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Configuration Guide - Preprocessors and Normalization](https://www.cisco.com/)
* [Cisco ASA Series Configuration Guide - Threat Detection and MPF TCP Options](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot Network Analysis Policies (NAP) on FMC](https://www.cisco.com/)
* [Cisco Live: Deep Dive into Snort Architecture and Evasion Prevention (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で Evasion 対策・Normalization が問われた場合の設計・構成チェックリスト：
    1. **FMC NAP**: `Policies > Access Control > Advanced > Network Analysis Policies` でカスタム NAP を作成 ➔ Inline Normalization、Frag3 Defrag、TCP Stream Reassembly、HTTP Inspect を有効化。
    2. **Target-Based 連携**: NAP の Defrag/Stream 再構築設定で `Target-Based` を選択。
    3. **ASA/FTD LINA MPF**: `tcp-map` で `check-retransmission` ＆ `seq-past-window drop` を構成。
    4. **HTTPS 復号**: SSL Policy (`Decrypt - Resign`) をアサインし、L7 Normalization への平文引き渡しを徹底。
* **図解（Evasion 攻撃 ➔ Normalization ➔ IPS 判定パイプライン）**:
  ```text
  [ Evasion Packets ] (%2f, Overlapping IP Frags, Split TCP Segments)
            │
            ▼
  [ Snort DAQ / Packet Decoder ] ──► Checksum & Header Verification
            │
            ▼
  [ Inline Normalization Preprocessor ] ──► Strip Reserved Bits & Clear TCP Flags
            │
            ▼
  [ IP Defragmentation (Frag3) ] ──► Reassemble Overlapping IP Fragments (Target-Based)
            │
            ▼
  [ TCP Stream Preprocessor ] ──► Reassemble Split TCP Segments into Session Flow
            │
            ▼
  [ HTTP Inspect Preprocessor ] ──► Decode %2f -> "/", Path Traversal -> Clean URI
            │
            ▼
  [ Snort Rule Engine ] ──► Match Clean Plaintext Payload against IPS Signatures ──► BLOCK!
  ```
* **注意点**:
  * ラボ試験で Normalization や NAP の設定を行った後は、FTD CLI で `show statistics normalization` を実行するか、FMC の `Analysis > Intrusion > Events` を確認し、GID 119/120 (HTTP)、123 (IP Defrag)、129 (Stream) の Preprocessor アラートが正常に生成されていることを確認してください。

---
