---
layout: default
title: 1.2.b-Application-inspection
nav_order: 2
parent: 1.2-Firewall-features-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.2.b Application inspection

Cisco ASA (Adaptive Security Appliance) および Cisco FTD (Firepower Threat Defense) における **Application Inspection（アプリケーションインスペクション）** は、レイヤ 3 / レイヤ 4 の単純なヘッダー検証を超えて、パケットのデータペイロード（L7）を詳細に解析・制御する高度なステートフルインスペクション機能です。

プロトコル内の非標準ポートの動的オープン（マルチチャネルプロトコル）、IP アドレスの自動変換（NAT 補正）、不正なプロトコルフォーマットの遮断（Strict Inspection）、正規表現（Regex）や AppID による細粒度のアプリケーション制限（AVC: Application Visibility and Control）を実現します。

---

## 📘 概要

* **機能概要**:
  * **Cisco ASA (MPF: Modular Policy Framework)**: `class-map`、`policy-map`（`policy-map type inspect <protocol>`）、`service-policy` を組み合わせ、HTTP, FTP, ESMTP, DNS, SIP, H.323, SQLnet などの L7 ペイロードを検証・操作・遮断します。
  * **Cisco FTD (Snort 3 Engine / AVC / AppID)**: VDB (Vulnerability Database) および Snort 3 検出インスペクタを活用し、ポート非依存で 4,000 以上のアプリケーションおよびサブ機能（Facebook Chat, YouTube Video, BitTorrent 等）の自動分類・アクセス制御を実施します。
* **利用目的**:
  * 動的二次チャネル（エフェメラルポート）を開くマルチチャネルプロトコル（FTP PASSV/PORT、SIP/H.323 RTP メディアチャネル）に対して動的なピンホール（Pinhole）を開放し、通信を通す。
  * ペイロード内のプライベート IP アドレス（FTP PORT コマンドや SIP Contact ヘッダー等）をパブリック IP へ書き換え（NAT 補正）、通信整合性を保持する。
  * アプリケーション層のプロトコル規約違反（RFC 非準拠）、バッファオーバーフロー攻撃、不審なコマンド（FTP `DELE` や ESMTP `EXPN`/`VRFY`）を検出・ドロップする。
  * サーバーバナーの偽装・マスク（`mask-banner`）、特定 URL やドメイン（Regex / URL Filtering）へのアクセス制御。
* **どのような場面で利用するか**:
  * FTP サーバーへのアクティブ/パッシブモード接続の許可、および管理外コマンドの制限。
  * VoIP ネットワーク（SIP/SCCP/H.323）の境界FW通過と NAT 環境適用。
  * DNS トンネリング攻撃の防御や、DNS 応答レコードの動的書き換え（DNS Doctoring）。
  * 従業員の業務外 Web アプリケーション（SNS、ストレージ、P2P）の使用制限。

---

## 🔑 要点

| 項目 | Cisco ASA Application Inspection (MPF) | Cisco FTD Application Control (Snort 3 / AVC) |
| :--- | :--- | :--- |
| **制御対象層** | アプリケーション層 (L7 プロトコル制御・構文解析) | アプリケーション層 (L7 識別・コンテキスト制御・AVC) |
| **トラフィック識別** | ポートベース (`match port`) ＋ L7 正規表現 (`regex`) | **ポート非依存 DPI (Deep Packet Inspection)** ＋ VDB |
| **動作エンジン** | LINA OS 内のプロトコルインスペクタ | Snort 3 / AppID / Network Discovery エンジン |
| **デフォルトポリシー** | `global_policy` 配下の `inspection_default` クラス | Access Control Policy (ACP) ＋ Network Discovery Policy |
| **マルチチャネル対応** | ピンホール (Pinhole) を動的生成 | Dynamic Application Identification ＋ Snort Preprocessor |
| **NAT 補正機能** | ペイロード内 IP/Port を自動書き換え | LINA/Snort 連携による自動書き換え |
| **シグネチャ更新** | ASA OS バージョンアップで更新 | **VDB (Vulnerability Database) / SRU の自動・定期更新** |
| **拡張性・独自定義** | L7 Inspect Class/Policy-Map, Regex Object | Custom Application Detector (FMC) |

---

## 🏗 動作原理

### 1. マルチチャネルプロトコル (FTP / SIP 等) における Pinhole 制御

制御チャネル（Control Connection）とデータチャネル（Data Connection）を分離して通信するプロトコルでは、データ通信用ポートが制御パケットのペイロード内で動的にネゴシエーションされます。Application Inspection エンジンは制御チャネルを解析し、必要な一時的アクセス許可エントリ（Pinhole）をメモリ上に自動生成します。

```text
[ Client (Inside) ]                                   [ ASA / FTD Firewall ]                               [ FTP Server (Outside) ]
        │                                                        │                                                        │
        │ ── 1. TCP Syn -> Port 21 (Control Conn) ─────────────► │                                                        │
        │ ◄── 2. TCP Syn-Ack ─────────────────────────────────── │ ── 1. TCP Syn -> Port 21 ────────────────────────────► │
        │                                                        │ ◄── 2. TCP Syn-Ack ─────────────────────────────────── │
        │                                                        │                                                        │
        │ ── 3. PORT Command (IP: 10.1.1.5, Port: 50001) ──────► │                                                        │
        │                                                        │ ── [ Application Inspection Engine ]                  │
        │                                                        │     a. Inspects Payload                             │
        │                                                        │     b. Rewrites Payload IP (NAT Fixup)              │
        │                                                        │     c. Dynamically Creates Temporary Pinhole        │
        │                                                        │        (Outside:* -> Inside:10.1.1.5:50001)          │
        │                                                        │                                                        │
        │                                                        │ ── 4. Mapped PORT Cmd (203.0.113.5, Port: 50001) ───► │
        │                                                        │ ◄── 5. 200 PORT Command Successful ─────────────────── │
        │                                                        │                                                        │
        │ ◄── 6. TCP Syn (Data Conn from Port 20) ────────────── │ ◄── 6. TCP Syn (Server Port 20 -> Client:50001) ────── │
        │     (Allowed by Pinhole without Static ACL)            │                                                        │
```

---

## ⚙ 動作シーケンス

パケットが ASA / FTD に到着した際の Application Inspection 処理シーケンスは以下の通りです。

1. **L3/L4 接続テーブルおよび ACL/ACP 評価**:
   * 送信元/宛先 IP および宛先ポートでアクセスルールを評価し、制御チャネルを許可。
2. **インスペクションエンジンのバインド判定**:
   * **ASA**: MPF の `service-policy` に配置された `inspect <protocol>` ルールを検索（`match default-inspection-traffic` またはカスタムポート指定）。
   * **FTD**: Snort 3 / AppID エンジンがパケットをキャプチャし、DAQ (Data Acquisition) を介してインスペクションを開始。
3. **L7 ペイロードのデコードおよび構文検証 (Deep Inspection)**:
   * ヘッダー長、プロトコルフォーマット、不正なコマンド文字列、バッファオーバーフローを検出。
   * **Regex / Inspection Map**: 設定された拒否条件（例: 禁止された URL / 禁止されたコマンド）に一致した場合、パケット破棄（`reset-drop` または `drop`）を実行。
4. **NAT アドレス埋め込み補正 (NAT Fixup)**:
   * ペイロード内に送信元/宛先 IP が含まれる場合、事前定義された NAT ルールに従ってペイロード内の IP アドレスおよび TCP/UDP チェックサムを書き換え。
5. **動的ピンホール (Pinhole) の作成**:
   * ネゴシエートされたデータポートを特定し、一時的なアクセストランジットルール（Pinhole）をメモリに登録。
6. **データチャネル接続の透過および削除**:
   * データセッション確立後、通信を許可。データチャネル終了（またはタイマー到達）時に Pinhole を破棄。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **デフォルトの `inspection_default` クラスの動作理解**:
   * `class-map inspection_default` 内で使用される `match default-inspection-traffic` は、IANA で定義された標準ポート（HTTP:80, FTP:21, DNS:53, ESMTP:25 等）を網羅する特殊な CLI ショートカットです。
   * **注意**: 標準外ポート（例: HTTP on TCP 8080, FTP on TCP 2121）でインスペクションを行う場合、新しい L3/L4 `class-map` を作成して該当ポートを明示的に指定する必要があります。
2. **L7 Inspection Class-Map および Policy-Map の入れ子構成 (Nesting)**:
   * 例: 「FTP サーバーに対する `DELE`（削除）コマンドを遮断し、サーバーバナーを偽装せよ」という問題。
   * 手順:
     1. L7 Class-Map (`class-map type inspect ftp match-all CM_FTP_CMD`) で `match request-command dele` を定義。
     2. L7 Policy-Map (`policy-map type inspect ftp PM_FTP_INSPECT`) で `parameters` 内に `mask-banner` を設定し、`class CM_FTP_CMD` に `reset log` をバインド。
     3. 標準 L3/L4 Policy-Map (`global_policy`) 内で `inspect ftp PM_FTP_INSPECT` としてバインド。
3. **Regex (正規表現) オブジェクトの活用**:
   * 特定の文字列（例: `.exe` ファイルダウンロードの防止、不審な Host ヘッダーのドロップ）を条件にする場合、`regex` コマンドで定義した上で、`class-map type regex` または `class-map type inspect` でグループ化して適用します。
4. **SIP / H.323 インスペクションと NAT の相互作用**:
   * IP 電話（VoIP）の通信で片通話（One-way Audio）が発生する原因の多くは、SIP インスペクションの未設定（または不適切な NAT 設定）による Contact/Via ヘッダーの書き換え失敗です。
5. **FTD における AVC (Application Visibility and Control) と VDB 更新**:
   * FMC 上でアプリケーション制御ルールを設定する際、最新の VDB (Vulnerability Database) が登録されていることが前提となります。試験でカスタムアプリ検出が求められた場合は `Custom Application Detector` を利用します。

---

## 🛠 設定方法

### 1. Cisco ASA: MPF による L7 ESMTP / FTP / HTTP 詳細インスペクション (CLI)

```bash
! ===== 1. Regex (正規表現) の作成 =====
regex REGEX_FORBIDDEN_EXT ".*\. (exe|bat|msi)$"
regex REGEX_BLOCKED_CMD "^(DELE|RMD)$"

! ===== 2. L7 Inspect Class-Map の作成 =====
class-map type inspect http match-any CM_HTTP_FILE_BLOCK
 match request header uri regex REGEX_FORBIDDEN_EXT

class-map type inspect ftp match-any CM_FTP_BLOCK_CMD
 match request-command regex REGEX_BLOCKED_CMD

! ===== 3. L7 Inspect Policy-Map の作成 (パラメータ & アクション設定) =====
policy-map type inspect http PM_HTTP_L7_INSPECT
 parameters
  spoof-server "Apache/2.4.41 (Secure)"
 class CM_HTTP_FILE_BLOCK
  reset log

policy-map type inspect ftp PM_FTP_L7_INSPECT
 parameters
  mask-banner
  mask-syst-reply
 class CM_FTP_BLOCK_CMD
  reset log

! ===== 4. L3/L4 Class-Map / Policy-Map への結合と適用 =====
class-map CM_HTTP_TRAFFIC
 match port tcp eq 8080

policy-map global_policy
 class inspection_default
  inspect ftp PM_FTP_L7_INSPECT
  inspect dns preset_dns_map
  inspect esmtp
 class CM_HTTP_TRAFFIC
  inspect http PM_HTTP_L7_INSPECT

service-policy global_policy global
```

### 2. Cisco FTD (FMC GUI): AVC (Application Visibility and Control) によるアプリ制限

1. **Access Control Rule の追加**:
   * **Policies > Access Control > Access Control Policy** へ移動し編集。
   * **Add Rule** をクリック。
   * **Name**: `Block_Social_Media_Games`
   * **Action**: `Block` または `Block with reset`
2. **Applications タブでの条件設定**:
   * **Available Applications** の検索窓で `Facebook` または `Games` を検索。
   * 例: `Facebook Games`, `Facebook Like`, `BitTorrent` を選択し **Add to Rule** をクリック。
3. **Rule の配置と Deploy**:
   * 作成したルールを Access Control Policy の上位（Mandatory 領域）に配置して保存。
   * **Deploy** を選択して対象 FTD へ適用。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **ASA MPF サービスポリシー統計・インスペクションヒット表示** | ASA CLI | <code>show service-policy inspect \<protocol\></code> |
| **ASA グローバルサービスポリシー全体の表示** | ASA CLI | <code>show service-policy global</code> |
| **ASA アクティブ接続とプロトコルフラグの確認** | ASA CLI | <code>show conn detail</code> (Flags: `M`=SMTP, `h`=H.323 等) |
| **ASA 動的生成された Pinhole 一覧の確認** | ASA CLI | <code>show local-host</code> / <code>show connection</code> |
| **ASA SIP 登録・セッション詳細表示** | ASA CLI | <code>show sip</code> / <code>show h225</code> / <code>show skinny</code> |
| **FTD Snort 3 デバッグおよびアプリケーション検出ログ** | FTD CLI | <code>system support firewall-engine-debug</code> |
| **FTD 検出アプリ統計ダッシュボードデータ確認** | FMC GUI | **Overview > Dashboards > Application Statistics** |
| **FTD VDB バージョンおよび検出アプライアンス確認** | FMC GUI | **Help > About** または **Objects > Application Detectors** |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **FTP パッシブモード (PASV) でファイル転送がタイムアウトする** | ASA/FTD で FTP インスペクションが未無効化/未適用 | ASA: <code>show service-policy inspect ftp</code> | `policy-map` 内で `inspect ftp` を追加し、動的パッシブポート（Pinhole）の追跡を有効化する。 |
| **標準外ポート (8080) の HTTP 通信でインスペクションが機能しない** | `match default-inspection-traffic` にポート 8080 が含まれない | ASA: <code>show class-map</code> | 新しい L3/L4 `class-map` を作成して `match port tcp eq 8080` を定義し、`inspect http` を適用する。 |
| **VoIP 通信時、片通話（音声が一方しか届かない）が発生する** | SIP/SCCP インスペクションによる SDP ペイロード内の IP 変換失敗 | ASA: <code>show sip</code><br>ASA: <code>show conn</code> | SIP インスペクションを有効化するか、NAT ポリシーで `no-proxy-arp` や適切な Static NAT を配置する。 |
| **ESMTP 経由のメール送信で、TLS 暗号化（STARTTLS）が失敗する** | ASA ESMTP インスペクションが `STARTTLS` コマンドを制限 | ASA: <code>show service-policy inspect esmtp</code> | ESMTP Inspection Policy Map 内で `allow-tls` を無効化/修正するか、`inspect esmtp` を調整する。 |
| **FTD で特定アプリ（例: Facebook Video）のみのブロックが効かない** | HTTPS トラフィックの SNI/証明書スキャン不備、または SSL 復号未設定 | FTD CLI: <code>system support firewall-engine-debug</code> | FMC の **SSL Policy** で該当通信を復号スキャン（Decrypt-Resign）するか、TLS Server Identity Discovery を有効化する。 |

---

## ⚠ 制限事項

* **Cisco ASA MPF の制限**:
  * ポート番号ベースの判定に強く依存するため、完全にポート非依存でマルチポート通信を行う現代の Complex App (Skype, P2P) の高精度識別には限界があります（FTD AVC の使用を推奨）。
* **SSL/TLS 暗号化トラフィックの制限**:
  * HTTP Inspection や FTP Inspection、Regex マッチングは、**トラフィックが暗号化（HTTPS/FTPS/SMTPS）されている場合はそのままではペイロードの解読が不可能** です。FTD の SSL Decryption Policy（または ASA SSL Appliance）で事前復号する必要があります。
* **クラスタリング環境における制限**:
  * CTIQBE, H.323, MGCP, SCCP (Skinny) などの一部のインスペクションエンジンは、ASA クラスタリング構成では非サポートです。

---

## 🔄 他技術との関連

* **1.2.a NAT**: FTP や SIP などのマルチチャネルプロトコルインスペクションは、NAT 処理と自動連動して L7 ペイロード内のプライベート IP をパブリック IP へ書き換えます。
* **2.0 Intrusion Prevention (IPS / Snort 3)**: FTD では、Access Control Policy 内で AVC（アプリ制御）を通過したパケットに対して Intrusion Policy (IPS Rule) を適用して脅威スキャンを行います。
* **WSA (Web Security Appliance) / CWS**: ASA 上で高度な L7 WEB 制御を行うため、WCCP や ScanSafe (CWS) モジュールへトラフィックをリダイレクトする構成と併用されます。

---

## 🧩 比較表

### 1. ASA MPF Inspection vs FTD Snort 3 / AVC

| 比較項目 | Cisco ASA MPF Application Inspection | Cisco FTD Snort 3 / AVC (NGFW) |
| :--- | :--- | :--- |
| **アーキテクチャ** | LINA C 言語ベースモジュール (Fixup 継承) | Snort 3 インスペクタ ＋ AppID 検出エンジン |
| **識別方式** | L3/L4 ポート ＋ L7 正規表現 (`regex`) | **ポート非依存 DPI (Deep Packet Inspection)** |
| **対応アプリケーション数** | 定型インターネットプロトコル (約 20〜30 種類) | **4,000 以上 (VDB アップデートで最新化)** |
| **Web 2.0 サブ機能制御** | 不可（ドメイン/URL 文字列単位でのみ制限） | **可能（例: Facebook 閲覧許可 / Facebook Game 遮断）** |
| **設定インターフェース** | ASA CLI (`class-map`, `policy-map`) | FMC / FDM GUI (Access Control Rules) |

---

## 💡 ベストプラクティス

1. **標準外ポートへの明確なクラス割り当て**:
   * デフォルトの `inspection_default` クラスに依存せず、社内で使用している標準外ポート（例: 8080, 8443, 2121）がある場合は専用の `class-map` を定義して明示的にインスペクションを適用します。
2. **FTD における VDB (Vulnerability Database) の自動更新設定**:
   * 新規アプリケーションの識別やシグネチャの誤検知を防ぐため、FMC 上で **System > Updates > Dynamic Objects / VDB** の定期更新タスク（週次/月次）をスケジュールします。
3. **暗号化通信に対する SSL Policy の導入**:
   * L7 アプリケーション制御や AVC を極限まで機能させるため、必要に応じて FMC 上で **SSL Policy (Decrypt - Resign)** を構成し、暗号化通信の可視化を行います。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA: FTP ディレクトリ操作・ファイル削除コマンド (`DELE` / `RMD`) のドロップ

#### 問題
ASA1 において、内部ユーザーが外部 FTP サーバー上でファイルを削除するコマンド（`DELE`, `RMD`）を実行した際、接続をリセット（`reset log`）して遮断する L7 FTP インスペクションを構成せよ。

#### 要件
* L7 Inspect Class-Map および Policy-Map を使用すること。

#### 設定例

```bash
! 1. 拒否コマンド用 Regex と L7 Class-Map
regex REGEX_FTP_DELETE "^(DELE|RMD)$"

class-map type inspect ftp match-any CM_FTP_DELETE_CMD
 match request-command regex REGEX_FTP_DELETE

! 2. L7 Inspect Policy-Map の定義
policy-map type inspect ftp PM_FTP_DELETE_BLOCK
 class CM_FTP_DELETE_CMD
  reset log

! 3. グローバルポリシーへの適用
policy-map global_policy
 class inspection_default
  inspect ftp PM_FTP_DELETE_BLOCK

service-policy global_policy global
```

#### 検証方法
* FTP クライアントから外部サーバーに接続し `delete test.txt` を実行。`Connection closed by remote host` となり、`show service-policy inspect ftp` でドロップ数が増加することを確認。

---

### 例2: ASA: HTTP サーバーバナーの偽装と特定拡張子 (`.exe`, `.bat`) ダウンロード禁止

#### 問題
ASA1 を通過する HTTP 通信において、外部 Web サーバーが返却する Server ヘッダーを `Server: AnonymousWeb` に書き換え（`spoof-server`）、URI 末尾が `.exe` または `.bat` であるレスポンスを遮断せよ。

#### 要件
* L7 HTTP Inspect Policy-Map を使用すること。

#### 設定例

```bash
! 1. 拡張子検索用 Regex と L7 Class-Map
regex REGEX_BLOCKED_EXT ".*\. (exe|bat)$"

class-map type inspect http match-any CM_HTTP_EXEC_FILES
 match request header uri regex REGEX_BLOCKED_EXT

! 2. L7 HTTP Policy-Map 定義
policy-map type inspect http PM_HTTP_SECURITY
 parameters
  spoof-server "AnonymousWeb"
 class CM_HTTP_EXEC_FILES
  reset log

! 3. グローバルポリシー適用
policy-map global_policy
 class inspection_default
  inspect http PM_HTTP_SECURITY

service-policy global_policy global
```

#### 検証方法
* `curl -I http://example.com/test.exe` を実行し、レスポンスヘッダーの `Server:` が書き換わっていること、および `.exe` ファイルがダウンロード不可であることを検証。

---

### 例3: ASA: ESMTP インスペクションによる送信元/宛先アドレス制限

#### 問題
ASA1 において、ESMTP 通信をインスペクションせよ。送信元メールアドレスが `spammer@bad.com` または宛先が `victim@internal.local` であるメールのやり取りを遮断せよ。

#### 要件
* `parameter-map type regex` および L7 ESMTP Map を使用すること。

#### 設定例

```bash
! 1. Regex の作成
parameter-map type regex REGEX_BAD_SENDER
 pattern "spammer@bad\.com"

parameter-map type regex REGEX_VICTIM_RECP
 pattern "victim@internal\.local"

! 2. L7 ESMTP Class-Map の作成
class-map type inspect esmtp match-all CM_ESMTP_SPAM
 match sender address regex REGEX_BAD_SENDER
 match recipient address regex REGEX_VICTIM_RECP

! 3. L7 ESMTP Policy-Map 定義
policy-map type inspect esmtp PM_ESMTP_SECURITY
 class CM_ESMTP_SPAM
  reset log

! 4. 適用
policy-map global_policy
 class inspection_default
  inspect esmtp PM_ESMTP_SECURITY
```

#### 検証方法
* Telnet 経由で SMTP サーバーのポート 25 に接続し `MAIL FROM: spammer@bad.com` を送信してセッションが即座に切断されることを確認。

---

### 例4: ASA: 標準外ポート (TCP 8080) に対する HTTP インスペクションの適用

#### 問題
内部クライアントがプロキシサーバー (`10.1.1.250:8080`) を経由して行う HTTP 通信に対し、ポート 8080 上で HTTP インスペクションを有効化せよ。

#### 要件
* 新しい L3/L4 Class-Map を作成すること。

#### 設定例

```bash
! 1. ポート 8080 用 L3/L4 Class-Map
class-map CM_HTTP_8080
 match port tcp eq 8080

! 2. Policy-Map へのクラス追加とインスペクション有効化
policy-map global_policy
 class CM_HTTP_8080
  inspect http

service-policy global_policy global
```

#### 検証方法
* `show service-policy inspect http` を実行し、`Class-map: CM_HTTP_8080`配下でパケット数がカウントされることを確認。

---

### 例5: ASA: DNS インスペクションと最大パケット長 (EDNS0 / 1280 bytes) の変更

#### 問題
DNS インスペクションにおいて、DNS UDP パケットの最大許可長をデフォルトの 512 バイトから **1280 バイト**（DNSSEC / EDNS0 対応）へ変更せよ。

#### 要件
* `preset_dns_map` またはカスタム DNS Policy-Map を使用すること。

#### 設定例

```bash
! 1. DNS Inspect Policy-Map 定義
policy-map type inspect dns PM_DNS_LARGE
 parameters
  message-length maximum 1280

! 2. グローバルポリシーへのバインド
policy-map global_policy
 class inspection_default
  inspect dns PM_DNS_LARGE

service-policy global_policy global
```

#### 検証方法
* `show service-policy inspect dns` を実行し、`message-length maximum 1280` が反映されていることを確認。

---

### 例6: ASA: ICMP ステートful インスペクションの有効化

#### 問題
デフォルト状態の ASA1 では、内部から外部への `ping` パケット送信時、戻りの ICMP Echo-Reply が ACL で遮断される。戻り通信を ACL なしで許可するため、ICMP ステートフルインスペクションを有効化せよ。

#### 要件
* `global_policy` 内に設定すること。

#### 設定例

```bash
policy-map global_policy
 class inspection_default
  inspect icmp

service-policy global_policy global
```

#### 検証方法
* 内部端末から外部 IP へ `ping` を送信し、インバウンド ACL を追加することなく 100% 成功することを確認。
* `show service-policy inspect icmp` で ICMP パケットカウントが増加することを確認。

---

### 例7: FTD (FMC): Access Control Policy による BitTorrent P2P 通信の完全ドロップ

#### 問題
FMC を使用して FTD1 にルールを作成せよ。内部ネットワーク (`10.0.0.0/8`) から外部への **BitTorrent** アプリケーション通信をポート番号に関わらず全ドロップせよ。

#### 要件
* FMC Access Control Policy の Applications 条件を使用すること。

#### 設定例

```text
1. FMC: Policies > Access Control > ACP 編集 > Add Rule.
2. Rule Name: Block_P2P_BitTorrent, Action: Block.
3. Source Networks: 10.0.0.0/8.
4. Applications タブ:
   - Filter 欄に `BitTorrent` を入力。
   - 表示された `BitTorrent` アプリケーションカテゴリを選択して Add to Rule。
5. Save & Deploy 実行。
```

#### 検証方法
* 内部端末で BitTorrent クライアントを起動し、通信がドロップされることを FMC の `Analysis > Connection Events` で確認。

---

### 例8: FTD (FMC): Custom Application Detector の作成と特定ヘッダーの検出

#### 問題
社内独自 Web アプリケーション (HTTP 上で動作し、ヘッダーに `X-Custom-App: InternalService` が含まれる) を識別するため、FMC 上で **Custom Application Detector** を作成せよ。

#### 要件
* Objects > Application Detectors 画面を使用すること。

#### 設定例

```text
1. FMC: Objects > Application Detectors > Create Custom Detector:
   - Name: Custom_Internal_App
   - Application Protocol: HTTP
2. Pattern Conditions 編集:
   - Pattern: InternalService
   - Position: HTTP Header (X-Custom-App)
3. OK をクリックして保存。
4. Access Control Policy で `Custom_Internal_App` を条件としたルールを作成して Deploy。
```

#### 検証方法
* `curl -H "X-Custom-App: InternalService" http://10.1.1.100` を送信し、FMC の Connection Events で `Custom_Internal_App` として識別されることを確認。

---

### 例9: FTD (FMC): OpenVPN アプリケーション通信の遮断ルール構成

#### 問題
従業員が個人用 OpenVPN ソフトウェア（TCP/UDP ポート非依存）を使用して外部プロキシへ接続する行為を拒否するルールを FMC で作成せよ。

#### 要件
* Application 条件で `OpenVPN` を指定すること。

#### 設定例

```text
1. FMC: Access Control Policy 編集 > Add Rule.
2. Name: Block_OpenVPN, Action: Block with reset.
3. Applications タブ:
   - Category: Encrypted Tunnel / VPN ➔ `OpenVPN` を選択して追加。
4. Logging タブ: `Log at Beginning of Connection` にチェック。
5. Save & Deploy 実行。
```

#### 検証方法
* クライアント上で OpenVPN 通信を開始し、接続が即座に Reset ドロップされることを検証。

---

### 例10: ASA: SIP インスペクションと VoIP シグナリング (UDP 5060) の有効化

#### 問題
内部 VoIP サーバー (`10.1.1.200`) と外部 SIP プロバイダ (`203.0.113.50`) 間の SIP シグナリング通信 (UDP 5060) に対し、SIP インスペクションを有効化し、RTP メディアチャネルが動的に許可されるようにせよ。

#### 要件
* `inspect sip` を使用すること。

#### 設定例

```bash
class-map CM_SIP_TRAFFIC
 match port udp eq 5060

policy-map global_policy
 class CM_SIP_TRAFFIC
  inspect sip

service-policy global_policy global
```

#### 検証方法
* `show sip` を実行し、アクティブな SIP 通信セッションおよび割り当てられた RTP Pinhole を確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ASA1 上で以下のコンフィグが適用されている。内部ユーザーが `http://10.1.1.100:8080/download/tool.exe` へアクセスしようとした際のアクションとして正しいものはどれか。

```bash
regex REGEX_EXE ".*\.exe$"

class-map type inspect http match-all CM_EXE
 match request header uri regex REGEX_EXE

policy-map type inspect http PM_HTTP
 class CM_EXE
  reset log

class-map CM_HTTP_PORT
 match port tcp eq 80

policy-map global_policy
 class CM_HTTP_PORT
  inspect http PM_HTTP

service-policy global_policy global
```

* A) パケットは即座にリセット（`reset`）され遮断される。
* B) ポートが 8080 であり、`CM_HTTP_PORT` (port 80) にマッチしないため、HTTP インスペクションは適用されず通信は通過する（ACLで許可されている場合）。
* C) `global_policy` の `inspection_default` が適用されドロップされる。
* D) `spoof-server` によりフェイクレスポンスが返却される。

**正解**: **B**
**解説**: L3/L4 `class-map CM_HTTP_PORT` は `match port tcp eq 80` のみを条件としているため、ポート 8080 宛ての通信はこのクラスにマッチせず、バインドされている `inspect http PM_HTTP` は評価されません。

---

### 2. トラブルシュート問題

**問題**: 管理者が ASA1 上で FTP サーバーに対する削除コマンド制限（`DELE` ブロック）を適用するために以下の設定を行ったが、削除コマンドがブロックされずに通過してしまう。原因として正しいものはどれか。

```bash
class-map type inspect ftp match-all CM_FTP_DEL
 match request-command dele

policy-map type inspect ftp PM_FTP
 class CM_FTP_DEL
  reset

policy-map global_policy
 class inspection_default
  inspect ftp
```

* A) `class-map` 内で `match-all` ではなく `match-any` を使用しなければならないため。
* B) `global_policy` 内の `inspect ftp` コマンドで L7 Policy-Map (`PM_FTP`) が指定・リンクされていないため。
* C) `regex` オブジェクトを使用していないため。
* D) `service-policy` コマンドが欠落しているため。

**正解**: **B**
**解説**: L7 Inspect Policy-Map (`PM_FTP`) を作成しても、L3/L4 Policy-Map 内の `inspect ftp` の後ろにマップ名として指定（`inspect ftp PM_FTP`）しなければ、作成した L7 ポリシーは反映されず、デフォルトのインスペクション動作のままとなります。

---

### 3. Design 問題

**問題**: 大規模企業のセキュリティ設計において、全従業員のソーシャルメディア（Facebook, X, LinkedIn）における「投稿」「ゲーム」「チャット」などのサブ機能単位でアクセス制限を行いたい。最も適した実装アーキテクチャはどれか。

* A) Cisco ASA 上で Regex オブジェクトと MPF HTTP Inspection を組み合わせる。
* B) Cisco FTD 上で Access Control Policy を使用し、AVC (Application Visibility and Control) ルールを構成する。
* C) ASA 上で `match default-inspection-traffic` を使用する。
* D) ASA 上で WCCP を使用して DNS サーバーへリダイレクトする。

**正解**: **B**
**解説**: Web 2.0 アプリケーションのサブ機能（投稿、ゲーム、チャット等）を細粒度でコントロールするには、VDB シグネチャに基づく DPI エンジンを備えた **Cisco FTD AVC (Application Visibility and Control)** が最も適しています。ASA MPF の Regex では動的 URL や暗号化通信の追従が極めて困難です。

---

### 4. 実装問題

**問題**: ASA1 において、ESMTP サーバーが外部へ応答する接続バナー（220 Banner）内の OS / バージョン情報を隠蔽（`mask-banner`）したい。適切な設定手順はどれか。

* A) `policy-map type inspect esmtp PM_ESMTP` を作成し、`parameters` 配下で `mask-banner` を指定した上で、`global_policy` 内の `inspect esmtp` に割り当てる。
* B) `access-list` で TCP 25 ポートのパケットをドロップする。
* C) `class-map inspection_default` 配下に `mask-banner` コマンドを直接投入する。
* D) `system support firewall-engine-debug` を実行する。

**正解**: **A**
**解説**: ESMTP のバナー隠蔽は、L7 Inspect Policy-Map (type inspect esmtp) の `parameters` モードで `mask-banner` を有効化し、それを L3/L4 Policy-Map へ紐付けることで実現します。

---

### 5. コンフィグ読解問題

**問題**: FTD を管理する FMC の Access Control Policy において、以下のルールが適用されている。ユーザーが Port 8080 を使用する未知の P2P アプリケーションでファイル共有を行った場合の動作として正しい説明はどれか。

* Rule 1: Action: `Allow`, Applications: `HTTP`, `HTTPS`
* Rule 2: Action: `Block`, Applications: `P2P Applications (Category)`
* Default Action: `Block all traffic`

* A) Rule 1 にマッチして通信は許可される。
* B) FTD の Snort/AppID エンジンがパケットを解析し、P2P アプリケーションとして識別された時点で Rule 2 にマッチしてドロップされる。
* C) Port 8080 であるため、ルール評価前に Prefilter で Fastpath される。
* D) 接続は許可されるが、IPS ログのみが記録される。

**正解**: **B**
**解説**: FTD の AVC / AppID 検出はポート番号ではなく **パケットのペイロード構造 (DPI)** に基づいて行われます。Port 8080 であっても P2P アプリケーションのシグネチャが検出されれば、Rule 2 (`Block`) が適用されます。

---

## 🔗 参考リソース

* [Cisco ASA Series Firewall Configuration Guide - Application Layer Protocol Inspection](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/firewall/asa-94-firewall-config/inspect-overview.html)
* [Cisco Secure Firewall Threat Defense Configuration Guide (FMC) - Application Detection](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/application_detection.html)
* [Cisco Technical Notes: ASA Application Inspection and MPF Configuration Examples](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/113642-asa-mpf-config.html)
* [Cisco Live: Deep Dive into Firepower AVC and Application Detection (BRKSEC-3020)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * ASA の MPF Application Inspection は **「L3/L4 Class ➔ L3/L4 Policy ➔ L7 Policy (オプション) ➔ Service Policy」** という階層構造をしっかり頭に整理しておくことが重要です。
  * FTD の AVC は **ポート非依存** で動作するため、試験では「ポート番号に惑わされずにアプリケーション識別子でルールを作成できるか」が試されます。
* **注意点**:
  * ASA で `inspect http` や `inspect ftp` などの L7 マップをバインドし忘れると、デフォルト動作のままになり要件を満たせなくなります。検証時には必ず `show service-policy inspect <protocol>` で自分が作成した L7 マップ名が表示されているか確認してください。
