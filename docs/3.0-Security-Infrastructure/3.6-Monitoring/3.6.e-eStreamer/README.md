---
layout: default
title: 3.6.e-eStreamer
nav_order: 5
parent: 3.6-Monitoring
grand_parent: 3.0-Security-Infrastructure
---

# 3.6.e eStreamer

## 📘 概要

eStreamer (Event Streamer) は、Cisco Firepower System (Firepower Management Center: FMC) において、リアルタイムに生成される各種セキュリティイベント、ネットワークディスカバリ情報、およびコンテキストメタデータを、外部のクライアントアプリケーション（Splunk、IBM QRadar、独自のSIEM、セキュリティ解析ツールなど）へ効率的かつ安全にストリーミング伝送するためのプロプライエタリなイベントストリーミングプロトコルおよびAPIアーキテクチャです。

従来のSyslog機能がシンプルなテキストメッセージ形式の片方向通知（主にUDP/TCP）であるのに対し、eStreamerは暗号化された相互TLS (Transport Layer Security) セッション (デフォルト TCP 8302 ポート) を確立し、構造化されたバイナリ形式または拡張コンテキストデータ形式でイベントをストリーミング配信します。

eStreamerの主な導入目的と利用場面は以下の通りです。

* **セキュリティイベントの集約とリアルタイムSIEM連携**: 侵入防御 (IPS) イベント、アンチマルウェア (AMP) イベント、接続ログ (Connection Events)、セキュリティインテリジェンスイベントを組織のSOC (Security Operations Center) やSIEMに統合。
* **フォレンジックと詳細パケット解析**: イベント発生時のトリガーパケットキャプチャ (Packet Logs) や、プロキシヘッダー経由の送信元IP等の追加属性 (Extra Data) を外部へエクスポート。
* **ネットワークアセット・ホストのプロファイリング**: FMCのネットワークディスカバリエンジン (RNA/RUA) が自動検出したホストOS、サービス、アプリケーション、ユーザーのログイン/ログアウト動態データを外部システムへリアルタイム共有。
* **エンタープライズ統合の標準化**: Splunk App for Firepower や IBM QRadar DSM などの事前ビルドされた統合モジュールを活用し、高度な検索、ダッシュボード作成、相関分析を自動化。

CCIE Security v6.1 ラボ試験および実務において、eStreamerは「FMCと外部解析システム間でのセキュリティデータの受け渡し」「Syslogとの機能的・構造的差異の理解」「クライアント証明書を用いた相互TLS認証とアクセス制御」「ドメイン構造（マルチテナンシー）におけるイベント配信の制限」といった観点から非常に重要度の高い技術です。

---

## 🔑 要点

eStreamerの主要な技術的特徴、制限事項、および設計上の注意点を下表に整理します。

| 項目 | 内容 |
| --- | --- |
| **機能特徴** | FMCと外部クライアント間で相互TLS (TCP 8302) を確立し、セキュリティイベントおよびディスカバリデータをリアルタイムストリーミングするバイナリ API |
| **主な用途** | SIEM (Splunk, IBM QRadar 等) や SOC プラットフォームへのイベントリアルタイム転送、フォレンジック解析用パケットデータのエクスポート |
| **メリット** | Syslogでは送信不可能なパケットデータ(Packet Logs)、拡張データ(Extra Data)、ディスカバリデータ(Host/OS/User)を完全抽出可能。TCP/TLSにより確実な到達性と暗号化を保証 |
| **デメリット** | FMCのCPU・メモリ・ディスクI/Oに一定の負荷がかかる。メッセージ構造がプロプライエタリであり、クライアント側でのパース処理やSDK/Appの更新・維持管理が必要 |
| **対応機種・プラットフォーム** | Cisco Firepower Management Center (FMC 1000/1600/2500/2600/4500/4600, FMCv), 旧Sourcefire Defense Center |
| **使用ポート** | TCP 8302 (FMCがTLSサーバー、外部SIEM/eStreamer ClientがTLSクライアントとして接続) |
| **制限事項** | FTDデバイスから直接eStreamer接続を確立することは不可(必ずFMCを経由)。15,000〜20,000 eps 以上の超高トラフィック環境ではイベントの絞り込み(Filtering)が必須 |
| **設計上の注意点** | FMCのマルチドメイン構成では、親ドメインで有効化したイベントタイプを子ドメインで個別に無効化することは不可。クライアント証明書は特定ドメインおよびその子孫ドメインに紐付く |

---

## 🏗 動作原理

eStreamerは、FMC内部のイベント管理デーモン (`sfestreamer`) と、外部ホスト上で動作する eStreamer クライアントアプリケーションとの間で、相互TLSハンドシェイクによる安全な通信チャネルを形成して動作します。

以下に通信フローと内部アーキテクチャの概要を示します。

```
[ FTD Managed Device ] 
       │
       │ (sftunnel: TCP 8305) ── IPS/Connection/File/Malware Events & Packet Captures
       ▼
[ Firepower Management Center (FMC) ]
       │
       ├─► [ SFDataCorrelator Process ] ── イベントの相関分析・データベース保管
       │
       └─► [ sfestreamer Daemon ] (Listening on TCP 8302)
                 ▲
                 │ (Mutual TLS Authentication via Client Certificate / Port 8302)
                 │
                 │ 1. TLS Connect & Certificate Handshake
                 │ 2. Client Request (Event Type Mask / Flags)
                 │ 3. Binary Data Stream (Events, Extra Data, Packets)
                 │
[ eStreamer Client (Splunk / QRadar / Custom App) ]
```

eStreamerの動作原理を支える主要なコンポーネントと概念は以下の通りです。

1. **sfestreamer デーモン**:
   FMC 上で動作するバックグラウンドプロセスであり、TCP 8302 ポートでクライアントからの TLS 接続要求を待機します。クライアント認証用 PKCS#12 または PEM 証明書を照合し、認可されたピアとのみセッションを生成します。

2. **クライアント証明書認証とピア認証 (Peer Certificate Authentication)**:
   FMC Web UI にてクライアント登録 (Hostname/IP) を行うと、FMC 内部 CA により固有のクライアント証明書 (`.pkcs12` または `.pem` 形式) が生成されます。クライアントはこの証明書を提示して FMC と相互 TLS ハンドシェイクを実施します。パスワード保護も選択可能です。

3. **リクエスト型データストリーミング (Request-Driven Streaming)**:
   TLS セッション確立後、eStreamer クライアントは FMC に対し、「希望するイベントタイプ（例: IPSイベント、パケットログ、接続イベントなど）」を定義したリクエストメッセージを送信します。FMC はクライアントが要求したイベントのみをフィルタリングしてバイナリエンコード（またはJSON/構造体）形式で送出します。

4. **イベントデータの包含範囲**:
   Syslog では不可能な「トリガーパケットの生バイナリデータ (Packet Data)」「プロキシ送信元やX-Forwarded-Forを含む拡張情報 (Extra Data)」「ホストディスカバリ (RNA) / ユーザーマッピング (RUA) 情報」を含めて一元ストリーミングします。

---

## ⚙ 動作シーケンス

eStreamer におけるクライアント登録からデータ受信、接続切断までのシーケンスは以下のステップで実行されます。

```
[ eStreamer Client ]                                    [ FMC (sfestreamer) ]
        │                                                         │
        │ ── (事前にFMC GUIでClient登録 & pkcs12証明書をダウンロード) ── │
        │                                                         │
        │ 1. TCP 3-Way Handshake (Destination TCP Port 8302) ────►│
        │◄────────────────────────────────────────────────────────┤
        │                                                         │
        │ 2. TLS Handshake (Mutual Authentication) ───────────────►│
        │    - Client presents pkcs12/PEM cert issued by FMC      │
        │    - FMC verifies cert validity & allowed domain        │
        │◄────────────────────────────────────────────────────────┤
        │                                                         │
        │ 3. eStreamer Request Message ──────────────────────────►│
        │    - Requests specific Event Types (e.g., Flags: 0x01)  │
        │    - Specifies Version & Client Capability              │
        │                                                         │
        │ 4. FMC Validation & Event Stream Initiation ────────────►│
        │    - FMC checks eStreamer Global Allowed Event Types    │
        │                                                         │
        │ 5. Real-time Binary Event Data Streaming ──────────────►│
        │◄────────────────────────────────────────────────────────┤
        │    - Record Type 1: Intrusion Event                     │
        │    - Record Type 2: Packet Data (Cap)                   │
        │    - Record Type 71: Connection Event                   │
        │    - Record Type 9: Host Discovery Data                 │
        │                                                         │
        │ 6. TLS Session Keepalive / Heartbeat ──────────────────►│
        │◄────────────────────────────────────────────────────────┤
        │                                                         │
        │ 7. Client Disconnect / Session Close ───────────────────►│
        │    - TLS Close Notify / TCP FIN                         │
```

1. **初期登録と証明書の配備 (Registration & Certificate Distribution)**:
   * 管理者は FMC UI (`System > Integration > eStreamer`) でクライアントホスト（IPアドレスまたはFQDN）を登録し、証明書を生成・ダウンロードします。
   * 証明書ファイルをクライアント側の指定ディレクトリ（Splunk eStreamer Client ディレクトリ等）へ保存します。

2. **TLS 相互認証の確立 (Mutual TLS Establishment)**:
   * クライアントは FMC の TCP 8302 ポートへ TLS 接続を起動します。
   * FMC はクライアント証明書を検証し、登録された IP/Hostname およびアクセス権限（ドメイン範囲）を承認します。

3. **イベントフラグ・要求メッセージの送信 (Event Request Negotiation)**:
   * クライアントは受信したいイベントビットマスク（例: IPS、Packet、Connection、User、Host 等）を含んだヘッダーメッセージを送信します。
   * FMC 側の Global eStreamer Event Configuration で許可されていないイベントタイプをクライアントが要求した場合、該当イベントは配信されません。

4. **バイナリイベントのリアルタイム配信 (Event Data Streaming)**:
   * FMC 内部の `SFDataCorrelator` が生成したイベントログが `sfestreamer` を経由してクライアントへリアルタイムでストリーミング送出されます。
   * 各レコードにはヘッダー（レコードタイプ、バージョン、長さ）とペイロードが含まれます。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、eStreamer に関する問題が出題された場合、単なる概要知識ではなく「FMC上での正確なクライアント登録手順」「相互TLS用証明書の生成とパスワード設定」「Splunk等の外部SIEMとの連携要件」「SyslogとeStreamerの使い分けに関するシナリオ判断」が問われます。

### Blueprintにおける重要ポイント

* **FMCでの Client 登録手順**:
  `System > Integration > eStreamer` 画面において、正確な Client Hostname/IP を登録し、生成された証明書 (`.pkcs12`) を適切な暗号化パスワードで保護・ダウンロードできること。
* **eStreamer Event Configuration の整合性**:
  FMC 側でグローバルに許可するイベントタイプ（Intrusion Events, Packet Data, Extra Data, Connection Events, Discovery Events 等）のチェックボックスが正しく有効化されていないと、クライアント側で要求を出してもイベントが届かない点。
* **Syslog vs eStreamer の機能比較要件**:
  問題文で「Packet Logs（侵入時の生パケット）や Extra Data、Host/User ディスカバリデータを含めて SIEM に送出せよ」「パケットドロップが発生しない信頼性の高い暗号化ストリームを使用せよ」と指示された場合は Syslog ではなく eStreamer を選択して設定する必要があります。
* **マルチドメイン環境における制約**:
  マルチドメイン (Multi-Domain) 構成において、親ドメインで有効化されたイベントタイプは子ドメインで無効化不可である点、またクライアント証明書はその作成されたドメインおよび配下の子孫ドメインのイベントのみを受信できる点を把握しておくこと。

### よくある設定ミスとトラブルシューティングポイント

1. **ポート 8302/tcp のブロック**:
   FMC と eStreamer Client 間のインラインファイアウォールやアクセスリストで TCP 8302 ポートが遮断されている。
2. **証明書ミスマッチ・IP/Hostname 不一致**:
   FMC UI でクライアント登録時に指定した IP アドレス / ホスト名と、実際の eStreamer Client が接続を試みる送信元 IP アドレスが一致していない。
3. **FMC 側での Event Type 未許可**:
   FMC の `System > Integration > eStreamer` 画面で、該当するイベントタイプ（例: `Connection Events` や `Intrusion Event Packet Data`）のチェックボックスが外れている。
4. **証明書パスワードの誤り**:
   FMC で証明書生成時に設定したパスワードと、Splunk/QRadar/eStreamer SDK 側で証明書を復号・ロードする際に入力したパスワードが一致しない。
5. **時刻同期 (NTP) の不一致**:
   FMC と SIEM クライアント間で時刻が大きくズレていると、TLS 証明書の有効期限検証エラーやタイムスタンプ解析エラーが発生する。

---

## 🛠 設定方法

FMC における eStreamer の設定手順および Splunk / SIEM クライアント側の基本構成例を解説します。

### 1. FMC GUI での eStreamer イベント有効化とクライアント登録

#### ステップ 1: グローバルイベントタイプの有効化
1. FMC Web UI にログインし、`System (歯車アイコン) > Integration > eStreamer` に移動します。
2. **eStreamer Event Configuration** セクションで、外部クライアントへ配信を許可するイベントタイプを選択します。
   * `Intrusion Events` (侵入イベント)
   * `Intrusion Event Packet Data` (トリガーパケットデータ)
   * `Intrusion Event Extra Data` (プロキシIP等の拡張メタデータ)
   * `Discovery Events` (ホスト/OS/アプリ検出データ)
   * `Correlation and Allow List Events` (相関ルール/違反イベント)
   * `User Events` (ユーザーログイン/アイデンティティデータ)
   * `Malware Events` & `File Events` (ファイル・マルウェア解析データ)
   * `Connection Events` (ネットワーク接続ログ)
3. **Save** をクリックして設定を保存します。

#### ステップ 2: eStreamer クライアントの作成と証明書ダウンロード
1. 同じ `System > Integration > eStreamer` ページで、**Create Client** をクリックします。
2. **Hostname / IP Address**: eStreamer クライアント（例: Splunk サーバー）の IP アドレスまたは FQDN を入力します（例: `192.168.10.50`）。
3. **Password**: クライアント証明書 (`.pkcs12` ファイル) を保護するための暗号化パスワードを入力します（例: `Cisco12345!`）。
4. **Save** をクリックします。
5. クライアント一覧に追加された該当ホストの右側にある **Download** (矢印アイコン) をクリックし、証明書ファイル (`192.168.10.50.pkcs12`) をローカル PC / SIEM サーバーへ保存します。

### 2. Splunk (Cisco Secure Firewall App for Splunk) での eStreamer 連携構成例

Splunk 側で Cisco eStreamer Client Add-on (eStreamer eNcore) を使用する場合の一般的な設定の流れです。

1. **証明書ファイルの配備**:
   FMC からダウンロードした `.pkcs12` 証明書ファイルを Splunk の eStreamer アプリ指定ディレクトリ（例: `$SPLUNK_HOME/etc/apps/TA-eStreamer/bin/envs/client.pkcs12`）へ配置します。
2. **証明書の変換・自動ロード (eNcore CLI / Web UI)**:
   eNcore 設定スクリプトを実行し、PKCS#12 証明書を PEM 形式に復号・展開します。
   ```bash
   cd $SPLUNK_HOME/etc/apps/TA-eStreamer/bin/
   python3 estreamer_setup.py
   # パスワード入力プロンプトで FMC で設定した証明書パスワードを入力
   ```
3. **接続テストとイベントマスク設定**:
   FMC (TCP 8302) への接続テストを実行し、取り込むイベント（IPS、Connection、User、Host 等）を定義します。

### 3. FMC CLI での eStreamer プロセス動作確認

FMC の Shell (Expert Mode) において、eStreamer デーモン (`sfestreamer`) の動作状態やソケットバインド状況を確認するコマンド例です。

```bash
# FMC の Expert Shell へ移行
admin@fMC:~$ expert
admin@fMC:~$ sudo -i

# sfestreamer デーモンのプロセス確認
root@fMC:~# ps aux | grep sfestreamer
root     14205  0.2  0.8 152344 32412 ?        Ss   08:15   0:05 /usr/local/sf/bin/sfestreamer

# TCP 8302 ポートのリスニング状態確認
root@fMC:~# netstat -tuln | grep 8302
tcp        0      0 0.0.0.0:8302            0.0.0.0:*               LISTEN

# アクティブな eStreamer クライアント接続の確認
root@fMC:~# netstat -an | grep 8302
tcp        0      0 192.168.1.100:8302      192.168.10.50:48922     ESTABLISHED
```


## 🔍 検証コマンド

FMC Web GUI および FMC SSH / Expert Shell で eStreamer の動作状態やログを検証・確認するためのコマンドを一覧表で示します。

| 目的 | コマンド / 確認画面 |
| --- | --- |
| **GUIでの接続状態・イベント確認** | FMC Web UI: `System > Health > Monitor > FMC Event Stream Status` |
| **クライアント登録・証明書一覧確認** | FMC Web UI: `System > Integration > eStreamer` |
| **eStreamer デーモンのプロセス状態確認** | `ps aux \| grep sfestreamer` (FMC Expert Shell) |
| **8302/tcp ポートのバインド確認** | `netstat -tuln \| grep 8302` または `ss -tuln \| grep 8302` (FMC Shell) |
| **クライアントとの暗号化セッション確立確認** | `netstat -an \| grep 8302` (ESTABLISHED 状態の確認) |
| **eStreamer システムログの確認** | `tail -f /var/log/messages \| grep sfestreamer` (FMC Shell) |
| **eStreamer デバッグログの監視** | `tail -f /var/sf/log/estreamer.log` または `/var/log/sf/sfestreamer.log` |
| **FMC ネットワークファブリック通信確認** | `tcpdump -i eth0 port 8302 -nn` (パケットキャプチャ) |

---

## 🚨 トラブルシュート

eStreamer 導入時・運用時に発生しやすいトラブル症状、原因、確認コマンド、および対処方法を下表にまとめます。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **クライアントが FMC (8302) に TLS 接続できない (Connection Refused / Timeout)** | 1. FMC 側で `sfestreamer` デーモンが停止している<br>2. 途中の FW で TCP 8302 がブロックされている | `netstat -tuln \| grep 8302`<br>`tcpdump -i eth0 port 8302` | `sfestreamer` サービスの再起動。<br>ネットワーク機器で TCP 8302 を許可 |
| **TLS Handshake Alert / Certificate Verification Failed Error** | 1. クライアントに配置した証明書が古い/無効<br>2. FMC クライアント登録の IP/Host と送信元 IP の不一致 | `/var/log/sf/sfestreamer.log`<br>`System > Integration > eStreamer` | FMC UI でクライアントを再登録し、新しい `.pkcs12` 証明書を再ダウンロードして配置 |
| **証明書復号エラー (Invalid PKCS#12 Password)** | クライアント側（Splunk/QRadar）で設定したパスワードが FMC で登録時に指定したパスワードと不一致 | `/var/log/sf/sfestreamer.log`<br>SIEM 側の eStreamer ログ | 正しいパスワードを SIEM 設定に入力、または FMC でクライアントを再作成 |
| **TLS セッションは確立するがイベントが一切届かない** | FMC の `eStreamer Event Configuration` で該当イベントタイプが未選択（チェックオフ） | FMC GUI: `System > Integration > eStreamer` | 必要なイベントタイプ (Intrusion, Connection, Packet 等) にチェックを入れて Save |
| **パケットデータ (Packet Logs) が届かない** | FMC の Device 管理で `Transfer Packets` オプションが無効化されている | FMC GUI: `Devices > Device Management > Edit Device` | `Transfer Packets` オプションを有効化してデプロイ |
| **イベント転送に伴い FMC の CPU/メモリ負荷が高騰** | 大量の Connection Events 等を広範囲にストリーミングしており、FMC 処理能力を超過 | `top` / `htop`<br>`perfstats` (FMC Shell) | クライアント側で要求するイベントフラグを絞り込む。不必要な Connection Events の除外 |

---

## ⚠ 制限事項

eStreamer の導入および運用における主な制限事項・制約条件を整理します。

* **FTD デバイスからの直接送信不可**:
  eStreamer サーバー機能は Firepower Management Center (FMC) 上でのみ動作します。Firepower Threat Defense (FTD) スイッチ/ルーターから直接 SIEM に eStreamer 通信を行うことはできません（FTD から直接外部送信する場合は Syslog を使用）。
* **パフォーマンスと最大スループット制限**:
  大量の Connection Events や Packet Capture データをリアルタイムストリーミングする場合、FMC のハードウェアスペック（FMCv / FMC1000〜4600）に応じた処理限界があります。高トラフィック環境では、ACP ルールレベルでのログ絞り込み（`Log at End of Connection` のみ適用）や eStreamer イベントタイプの最適化が不可欠です。
* **マルチドメイン環境における制限**:
  親ドメイン (Ancestor Domain) で許可された eStreamer イベントタイプは、子ドメイン (Descendant Domain) で個別に無効化することはできません。また、子ドメインで作成されたクライアント証明書は、親ドメインや他の兄弟ドメインのイベントを取得することは不可能です。
* **Secure Certifications Compliance Mode (CC / UCAPL Mode)**:
  FMC を Common Criteria (CC) モードまたは UCAPL モードに設定して運用する場合、eStreamer による外部イベント出力機能は安全基準の制約によりサポート対象外となります。

---

## 🔄 他技術との関連

eStreamer は FMC 内外のさまざまな技術要素と連携して機能します。

* **Firepower Threat Defense (FTD) & sftunnel**:
  FTD はセンサリングしたイベントやトリガーパケットを `sftunnel` (TCP 8305 ポート) 経由で FMC へ集約します。eStreamer は FMC 内の `SFDataCorrelator` が生成したイベントデータベースを抽出し、外部へストリーミングします。
* **Syslog (Security Event Syslog)**:
  FMC/FTD は Syslog によるイベント出力もサポートしますが、Syslog ではパケットログやディスカバリデータの出力が不可能です。補完関係または排他的構成として設計されます。
* **Cisco Secure Analytics and Logging (SAL) / Secure Network Analytics (Stealthwatch)**:
  FMC は SAL や Secure Network Analytics と連携し、フローデータやセキュリティイベントを外部の Secure Analytics Manager へ大量バックアップ・保存する構成を取ることができます。
* **Cisco ISE (Identity Services Engine) & pxGrid**:
  FMC は pxGrid 経由で ISE から Identity (ユーザー名、IP、SGT、デバイスタイプ) を取得します。eStreamer はこれらの属性を含んだ User Events / Connection Events を外部 SIEM へストリーミングします。
* **Snort IPS Engine**:
  Snort エンジンが検知した侵入イベントやドロップ理由、ルール ID (GID/SID/Rev)、およびルールトリガー時の生パケット (Packet Logs) が eStreamer を介して外部へ提供されます。


## 🧩 比較表

eStreamer と他のモニタリング・イベント転送プロトコルの比較、および eStreamer Client と pxGrid Participant の比較を下表にまとめます。

### 1. eStreamer vs Syslog vs SNMP Traps vs NetFlow (NSEL)

| 比較項目 | eStreamer | Syslog | SNMP Traps | NetFlow / NSEL |
| --- | --- | --- | --- | --- |
| **送信元コンポーネント** | Firepower Management Center (FMC) | FMC または FTD デバイス直接 | FMC または FTD デバイス直接 | FTD / ASA デバイス直接 |
| **トランスポート / ポート** | TCP 8302 (TLS 暗号化) | UDP 514 / TCP 514 / TLS 6514 | UDP 162 | UDP (任意ポート) |
| **データフォーマット** | プロプライエタリ・バイナリ (または JSON/拡張構造) | RFC 3164 / RFC 5424 テキスト | SNMP PDU (BER エンコード) | NetFlow v9 / IPFIX テンプレート |
| **侵入イベント (IPS)** | 完全サポート (詳細属性含む) | サポート (基本テキスト) | サポート (Trap メッセージ) | 不可 |
| **生パケットデータ (Packet Logs)** | **サポート** (生バイナリキャプチャ) | 不可 | 不可 | 不可 |
| **拡張データ (Extra Data)** | **サポート** (プロキシIP/HTTP属性等) | 不可 | 不可 | 不可 |
| **ネットワークディスカバリデータ** | **サポート** (Host/OS/App/User) | 不可 | 不可 | 不可 |
| **データ損失保護** | **あり** (TCP/TLS による再送制御) | UDP は不可 (TCP/TLS 時あり) | なし (UDP) | なし (UDP) |

### 2. eStreamer Client vs Cisco ISE pxGrid Participant

| 比較項目 | eStreamer Client | pxGrid Participant |
| --- | --- | --- |
| **主要目的** | FMC から外部 SIEM へのセキュリティイベント・キャプチャデータの単方向ストリーミング | ISE とセキュリティ製品間での動的コンテキスト共有 (Identity, SGT, Rapid Threat Containment) |
| **中心ノード** | Firepower Management Center (FMC) | Cisco Identity Services Engine (ISE) |
| **通信方向** | 主に FMC ──► Client へのストリーミング | 双方向 (Publish / Subscribe 制御、ANC Remediation アクション実行) |
| **セキュリティ** | 相互 TLS (FMC 内部 CA 発行の PKCS#12 証明書) | 相互 TLS (ISE CA または外部 Enterprise PKI 証明書) |

---

## 💡 ベストプラクティス

1. **必要最低限のイベントタイプのみ選択**:
   FMC の `System > Integration > eStreamer` 画面で、不必要なイベント（例: 大量の Connection Events や全 Discovery Events）を無暗に有効化しないこと。侵入イベント (Intrusion) とファイル/マルウェア (File/Malware) に絞り込むことで FMC の負荷を大幅に軽減できます。
2. **接続イベントのログ記録ルール最適化 (End-of-Connection Only)**:
   Access Control Policy (ACP) のルールにおいて、`Log at Beginning of Connection` と `Log at End of Connection` の両方を有効にするとログ量が倍増します。特別な理由がない限り `Log at End of Connection` のみを有効化してください。
3. **安全なクライアント証明書管理**:
   FMC で eStreamer クライアントを作成する際は、必ず強固な暗号化パスワードを設定して `.pkcs12` ファイルを保護し、クライアントホスト（Splunk 等）上のアクセス権限を厳格に制御してください。
4. **時刻同期 (NTP) の厳格な運用**:
   FMC および eStreamer Client (SIEM) は、同一の信頼できる NTP サーバー（例:Stratum 1/2）に同期させてください。時刻の不一致は TLS 証明書検証失敗やイベントのタイムライン解析ミスを招きます。
5. **ネットワーク帯域とファイアウォールルールの事前確保**:
   FMC と eStreamer Client 間の通信経路（TCP 8302）における十分なネットワーク帯域幅を確保し、ステートフルファイアウォールでタイムアウトによるセッション断が発生しないようキープアライブ設定を適切に維持してください。

---

## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験レベルの要件を想定した 10 個の演習シナリオと設定例、検証手順を示します。

### サンプル 1: eStreamer グローバルイベントタイプの設定

* **問題**: FMC において、外部 SIEM へ配信可能な eStreamer イベントとして「侵入イベント」「トリガーパケットデータ」「拡張メタデータ」「ファイルおよびマルウェアログ」を有効化せよ。
* **要件**:
  * FMC GUI 上で設定すること。
  * 接続イベント (Connection Events) は含めないこと。
* **設定例**:
  1. FMC UI: `System > Integration > eStreamer` へ移動。
  2. `eStreamer Event Configuration` で以下の項目のみチェックを有効化:
     * `[x] Intrusion Events`
     * `[x] Intrusion Event Packet Data`
     * `[x] Intrusion Event Extra Data`
     * `[x] Malware Events`
     * `[x] File Events`
     * `[ ] Connection Events` (チェックオフ)
  3. `Save` をクリック。
* **検証方法**:
  * FMC UI 上で設定が保存されていることを確認。

---

### サンプル 2: Splunk 用 eStreamer クライアントの登録と証明書作成

* **問題**: IP アドレス `192.168.100.50` を持つ Splunk サーバーを eStreamer クライアントとして FMC に登録し、パスワード `CiscoLab123!` で保護された PKCS#12 証明書を生成・ダウンロードせよ。
* **要件**:
  * クライアントホスト名/IP: `192.168.100.50`
  * 証明書暗号化パスワード: `CiscoLab123!`
* **設定例**:
  1. FMC UI: `System > Integration > eStreamer` へ移動。
  2. `Create Client` をクリック。
  3. `Hostname / IP Address`: `192.168.100.50`
  4. `Password`: `CiscoLab123!`
  5. `Save` をクリック。
  6. 生成されたクライアント一覧より `192.168.100.50` の `Download` アイコンをクリックし、`192.168.100.50.pkcs12` を保存。
* **検証方法**:
  * FMC Shell で `/var/sf/etc/estreamer/` ディレクトリ配下にクライアントエントリーが作成されているか確認。

---

### サンプル 3: FTD デバイスでのトリガーパケット転送 (Transfer Packets) 有効化

* **問題**: 侵入イベント発生時に、トリガーとなった生パケットが FMC を経由して eStreamer へ転送されるよう、FTD デバイス `FTD-HQ` の設定を確認・修正せよ。
* **要件**:
  * `Transfer Packets` オプションを有効化すること。
* **設定例**:
  1. FMC UI: `Devices > Device Management` へ移動。
  2. `FTD-HQ` の `Edit (鉛筆アイコン)` をクリック。
  3. `Device` タブの `General` セクションで `Transfer Packets` にチェックが入っていることを確認（入っていない場合はチェックを入れて保存）。
  4. `Deploy` を実行。
* **検証方法**:
  * IPS イベント発生時に eStreamer Client 側で Packet Record (Record Type 2) が受信できるか検証。

---

### サンプル 4: ACP ルールにおけるログ設定と eStreamer 連携最適化

* **問題**: 内部 LAN から DMZ Web サーバーへの HTTP/HTTPS 通信を許可する Access Control Rule において、FMC および eStreamer へのログ転送を最小限の負荷で設定せよ。
* **要件**:
  * 接続終了時 (`Log at End of Connection`) のみログを記録すること。
  * 接続開始時 (`Log at Beginning of Connection`) は記録しないこと。
* **設定例**:
  1. FMC UI: `Policies > Access Control > Access Control` で対象ポリシーを編集。
  2. 該当ルール `Allow_LAN_to_DMZ` を編集。
  3. `Logging` タブへ移動。
  4. `[x] Log at End of Connection` を選択。
  5. `[ ] Log at Beginning of Connection` を解除。
  6. `Send Events to: [x] FMC Event Viewer` を選択。
  7. ポリシーを保存して FTD へデプロイ。
* **検証方法**:
  * 通信終了時にのみ Connection Event が 1 通発出され、eStreamer 経由で転送されることを確認。

---

### サンプル 5: FMC Shell での eStreamer サービス状態およびソケット検証

* **問題**: eStreamer サービスが正常に 8302/tcp ポートでリスニングしているか、FMC の CLI/Shell を使用して確認せよ。
* **要件**:
  * Expert Shell コマンドを使用すること。
* **設定例・検証コマンド**:
  ```bash
  admin@fMC:~$ expert
  admin@fMC:~$ sudo netstat -tuln | grep 8302
  # 期待される出力:
  # tcp        0      0 0.0.0.0:8302            0.0.0.0:*               LISTEN
  
  admin@fMC:~$ sudo ps aux | grep sfestreamer
  # 期待される出力:
  # root     14205  ... /usr/local/sf/bin/sfestreamer
  ```

---

### サンプル 6: アクティブな eStreamer セッションの接続トラブルシューティング

* **問題**: SIEM サーバー (`10.1.100.200`) から FMC (`10.1.10.10`) への eStreamer TLS 接続が確立しているか、TCP ステートを確認せよ。
* **要件**:
  * FMC Shell コマンドを使用。
* **設定例・検証コマンド**:
  ```bash
  admin@fMC:~$ expert
  admin@fMC:~$ sudo ss -an | grep 8302
  # 期待される出力:
  # ESTAB   0      0   10.1.10.10:8302   10.1.100.200:54210
  ```

---

### サンプル 7: eStreamer クライアントの失効・削除処理

* **問題**: 破棄された旧 SIEM サーバー (`192.168.200.99`) の eStreamer アクセス権限を FMC 上で即座に取り消せ。
* **要件**:
  * FMC サービスを再起動せずに即時適用すること。
* **設定例**:
  1. FMC UI: `System > Integration > eStreamer` へ移動。
  2. クライアント一覧より `192.168.200.99` の箇所の `Delete (ゴミ箱アイコン)` をクリック。
  3. 削除確認ダイアログで `OK` をクリック。
* **検証方法**:
  * FMC 側で該当クライアントからの TLS 8302 接続要求が即座に拒否されることをログ (`/var/sf/log/sfestreamer.log`) で確認。

---

### サンプル 8: eStreamer デバッグログのリアルタイム追跡

* **問題**: クライアントとの TLS ハンドシェイク失敗の原因を調査するため、eStreamer の内部ログファイルをリアルタイム監視せよ。
* **要件**:
  * FMC Shell コマンドを使用。
* **設定例・検証コマンド**:
  ```bash
  admin@fMC:~$ expert
  admin@fMC:~$ sudo tail -f /var/log/sf/sfestreamer.log
  # 接続エラー例:
  # [ERROR] Peer 192.168.100.50 connection failed: Certificate verification failed (Unknown CA / Host Mismatch)
  ```

---

### サンプル 9: eStreamer と Syslog の共存設定

* **問題**: 高優先度の侵入イベント (IPS) は Syslog サーバー (`10.0.0.100`) へ直接送出しつつ、パケットキャプチャ付きの詳細分析用データは eStreamer 経由で FMC から SIEM へストリーミング配信せよ。
* **要件**:
  * FTD から直接 Syslog 送信、FMC から eStreamer 送信の二重構成。
* **設定例**:
  1. **FMC/FTD Syslog 設定**: `Devices > Platform Settings` で Syslog サーバー `10.0.0.100` (UDP 514) を設定。
  2. **Intrusion Policy Syslog 設定**: Intrusion Policy の Advanced Settings で Syslog 送信を有効化。
  3. **eStreamer 設定**: `System > Integration > eStreamer` で `Intrusion Events` および `Intrusion Event Packet Data` を有効化し、SIEM クライアントを登録。
* **検証方法**:
  * Syslog サーバーでテキストアラートを受信し、SIEM サーバーでパケットキャプチャデータを含む eStreamer バイナリレコードを受信できることを確認。

---

### サンプル 10: eStreamer 通信用の TCP パケットキャプチャ実行

* **問題**: FMC インターフェース上で SIEM サーバー (`192.168.100.50`) との 8302/tcp 通信パケットをキャプチャし、TLS ハンドシェイクの到達性を検証せよ。
* **要件**:
  * FMC Expert Shell の `tcpdump` を使用。
* **設定例・検証コマンド**:
  ```bash
  admin@fMC:~$ expert
  admin@fMC:~$ sudo tcpdump -i eth0 host 192.168.100.50 and port 8302 -nn -v
  # 期待される出力:
  # 192.168.100.50.51234 > 10.1.10.10.8302: Flags [S], seq ...
  # 10.1.10.10.8302 > 192.168.100.50.51234: Flags [S.], seq ..., ack ...
  # Client Hello / Server Hello / Certificate Exchange のキャプチャ確認
  ```


## ❓ 想定試験問題

CCIE Security ラボ試験および記述試験を想定した理解度確認問題と詳細解説です。

---

### 質問 1 (コンフィグ読解・診断)

SIEM アナリストから「FMC から侵入イベントのアラートテキストは届いているが、攻撃パケットのヘッダーおよびペイロード（Packet Data）が eStreamer に含まれていない」との報告を受けました。FMC GUI の System > Integration > eStreamer では Intrusion Events および Intrusion Event Packet Data の両方にチェックが入っています。この問題の根本原因として最も可能性が高いものはどれですか？

* A. FMC の sfestreamer デーモンが停止している。
* B. FTD デバイスの管理設定で Transfer Packets オプションが無効化されている。
* C. eStreamer クライアント証明書の有効期限が切れている。
* D. TCP 8302 ポートでパケットのフラグメンテーションが発生している。

**正解**: **B**

**解説**:
FMC がインスペクション結果としてトリガーパケットデータ（Packet Logs）をログ記録し、eStreamer へ転送するためには、イベントを検知した FTD デバイス側の設定で Transfer Packets オプションが有効化されている必要があります。このオプションがオフの場合、FTD から FMC へパケット自体が転送されないため、FMC の eStreamer 設定でパケットデータが有効になっていても eStreamer クライアントへパケットは配信されません。

---

### 質問 2 (プロトコル比較・実装)

ある企業では、新しい SIEM プラットフォームを導入し、Firepower システムから「侵入イベント」「接続イベント」に加えて、「ホストのOS/アプリケーション識別情報 (Discovery Data)」および「ユーザーのログイン動態 (User Events)」をリアルタイムで収集することを求めています。ネットワーク管理者は FTD デバイスから直接 Syslog (UDP 514) を送出する構成を提案しました。この提案に対する評価として正しいものはどれですか？

* A. Syslog はテキスト形式で軽量なため、ホストディスカバリデータやユーザーイベントを含めて最も効率的に全データを収集できる。
* B. FTD から直接送出する Syslog ではホストディスカバリデータやユーザーイベント、トリガーパケットデータを含めることができないため、FMC の eStreamer (TCP 8302) を利用する必要がある。
* C. Syslog (TLS 6514) を使用すれば、eStreamer と全く同じデータ（パケットログ、ネットワークマップ、ユーザー履歴）を送信可能である。
* D. FTD デバイスから直接 eStreamer クライアントセッションを確立してストリーミング配信すべきである。

**正解**: **B**

**解説**:
Syslog 通信では、基本的な接続ログや侵入イベントアラートは送信できますが、FMC が一元管理する「ネットワークディスカバリデータ (RNA)」「ユーザーアクティビティデータ (RUA)」「生パケットログ (Packet Data)」「拡張コンテキスト (Extra Data)」を直接送信することはできません。これらを取得するには FMC 上で eStreamer (TCP 8302) を設定し SIEM と連係させる必要があります。



---

### 質問 3 (トラブルシューティング・クライアント認証)

FMC に新しい eStreamer クライアント (10.1.20.50) を登録し、生成された証明書ファイルをダウンロードして SIEM サーバーに配置しました。しかし、SIEM 側で sfestreamer との TLS ハンドシェイクが失敗し、FMC の /var/log/sf/sfestreamer.log に Certificate verification failed: IP Address Mismatch と記録されています。この原因として考えられるものはどれですか？

* A. FMC の Web UI でクライアント登録時に入力した IP アドレス/ホスト名と、実際に eStreamer 接続を起動した SIEM サーバーの IP アドレスが一致していない。
* B. SIEM サーバーの NTP 同期が正常に行われており、時刻が FMC と完全に一致している。
* C. FMC 側で TCP 8302 ポートが Listen 状態になっていない。
* D. クライアント設定時に認証パラメータを指定しなかった。

**正解**: **A**

**解説**:
FMC の eStreamer ピア認証では、クライアント登録時に指定した IP アドレス/ホスト名と、実際に TLS 8302 ポートにアクセスしてきたクライアントの送信元 IP アドレスが厳格に照合されます。送信元 IP アドレスが登録値と異なる場合、IP Address Mismatch エラーが発生して TLS ハンドシェイクが拒否されます。

---

### 質問 4 (マルチドメイン・設計)

FMC がマルチドメイン構造 (Global Domain > Domain_A) で運用されています。Global Domain の管理者が eStreamer 設定で Intrusion Events および Connection Events を有効化しました。Domain_A の管理者が、自ドメイン配下の eStreamer クライアントにおいて Connection Events の配信を無効化（除外）しようとしています。この操作に関する記述として正しいものはどれですか？

* A. Domain_A の管理者は、自ドメインの eStreamer 設定で自由に Connection Events を無効化できる。
* B. 親ドメイン (Global Domain) で有効化された eStreamer イベントタイプは、子ドメイン (Domain_A) で個別に無効化することはできない。
* C. 子ドメインで作成されたクライアント証明書は、Global Domain 全体のイベントを取得する特権を持つ。
* D. マルチドメイン環境では eStreamer 機能自体が使用不可となる。

**正解**: **B**

**解説**:
親ドメイン（上位ドメイン）でグローバルに有効化された eStreamer イベントタイプは継承され、子孫ドメインで個別にオフ（無効化）にすることはできません。また、子ドメインで作成された証明書はそのドメインおよび配下の子ドメインのイベントのみにアクセスが制限されます。

---

### 質問 5 (アーキテクチャ・運用)

eStreamer 連携において、FMC のパフォーマンス悪化やメモリ枯渇を防ぐためのベストプラクティスとして不適切なものはどれですか？

* A. アクセスコントロールルールにおいて、特別な理由がない限り Log at End of Connection のみを有効化し、Log at Beginning of Connection は無効化する。
* B. SIEM 側で必要としないイベントタイプ（例: 大量の Connection Events）は FMC の eStreamer グローバル設定でチェックを外しておく。
* C. すべての通信ルールで即座に生パケットログ (Packet Data) と全接続ログを収集するため、すべての Access Control Rule で Beginning と End の両方のログを常時オンにする。
* D. FMC と eStreamer クライアント間で信頼できる NTP サーバーによる時刻同期を維持する。

**正解**: **C**

**解説**:
すべての Access Control Rule で Beginning と End の両方のログを記録し、さらに全パケットデータを無条件にストリーミングすると、FMC の SFDataCorrelator および sfestreamer デーモンに過大な負荷がかかり、処理遅延やイベントドロップを引き起こします。

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび各種統合ガイド:

* Cisco Firepower Management Center Configuration Guide - eStreamer Streaming
* Firepower System Event Streamer Integration Guide
* Cisco Secure Firewall App for Splunk User Guide
* IBM QRadar Integration Guide for Cisco Firepower
* Cisco Security Analytics and Logging (SAL) Overview Guide

---

## 📝 補足（Notes）

* **eStreamer Port 8302 / sftunnel Port 8305 の区別**:
  * **TCP 8305**: FTD Managed Device ──► FMC 間の管理およびイベント転送トンネル (sftunnel)。
  * **TCP 8302**: FMC ──► eStreamer Client (SIEM) 間のイベントストリーミング用ポート (sfestreamer)。
* **Syslog との使い分け要約**:
  * 単純なテキストアラート、軽量な SIEM 送信、または FTD からの直接出力 ──► **Syslog**。
  * トリガーパケット (Packet Captures)、拡張メタデータ (Extra Data)、ネットワークディスカバリ (RNA/Host/App/OS)、ユーザー識別 ──► **eStreamer**。
* **証明書の形式要件**:
  * FMC からダウンロードされる証明書は PKCS#12 (.pkcs12) 形式。Splunk 等で利用する際は PEM 形式 (client.crt / client.key) へ変換・展開して使用するケースが多い点に留意。


