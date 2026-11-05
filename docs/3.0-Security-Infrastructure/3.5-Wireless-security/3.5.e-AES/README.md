---
layout: default
title: 3.5.e-AES
nav_order: 5
parent: 3.5-Wireless-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.5.e AES

## 📘 概要

Advanced Encryption Standard (AES) は、米国国家標準技術研究所 (NIST) によって制定された共通鍵暗号規格 (FIPS PUB 197) であり、現代の無線LAN (IEEE 802.11) セキュリティにおける標準かつ最上位のデータ暗号化アルゴリズムです。

従来の WEP や TKIP が抱えていた RC4 ストリーム暗号に基づく構造的な脆弱性を根本的に解決するため、IEEE 802.11i 規格において AES をベースとした暗号化プロトコルが導入されました。無線LAN環境において AES 暗号を用いる代表的なプロトコルには以下の2つが存在します。

* **AES-CCMP (Counter Mode with Cipher Block Chaining Message Authentication Code Protocol)**: IEEE 802.11i / WPA2 / WPA3 の標準的な暗号化方式。128ビットのブロック暗号である AES を使用し、データの「機密性（暗号化）」に CTR (Counter) モード、「整合性・認証（改ざん検知）」に CBC-MAC モードを採用。
* **AES-GCMP (Galois/Counter Mode Protocol)**: IEEE 802.11ad / 802.11ac / 802.11ax / 802.11be および WPA3 192-bit Mode (CNSA Suite B) などで採用される超高速・高セキュリティ暗号化方式。ガロア体演算による認証付き暗号 (AEAD) により、ハードウェアでの並列処理と10Gbps超の高速スループットを実現。

無線LANにおける AES の主な利用目的と適用場面は以下の通りです。

1. **データフレームの超高速・高セキュリティ暗号化**: 企業内無線LAN、Wi-Fi 6/6E/7 環境における標準データ保護。
2. **管理フレームの保護 (PMF / IEEE 802.11w)**: Deauthentication (切断) や Disassociation 攻撃などの DoS 攻撃を防ぐため、AES-BIP (BIP-CMAC-128 / BIP-GMAC-256) を用いてマネジメントフレームの改ざん防止・認証を実現。
3. **WPA3-Enterprise 192-bit Mode (CNSA適合)**: 政府機関、金融、医療などの極めて高い機密性が求められる環境向けに、256ビット鍵長をもつ AES-GCMP-256 を採用。


---

## 🔑 要点

無線LANにおける AES (CCMP / GCMP) の技術的特徴、制限事項、および設計上の注意点を下表に示します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | 128ビットブロック暗号 AES をベースとし、パケット番号 (PN) によるリプレイ攻撃防止と、改ざん検知 (MIC) を単一アルゴリズムで実現する高度な認証付き暗号 |
| **用途** | WPA2/WPA3 端末接続保護、PMF (Protected Management Frames)、WPA3-Enterprise 192-bit Mode (CNSA Suite)、Mesh/FlexConnect トンネル保護 |
| **メリット** | ハードウェア処理に最適化された高いスループット、幾何学的な鍵暗号強度、リプレイ攻撃およびパケット改ざんに対する完全な防御 |
| **デメリット** | 802.11i 対応のハードウェア暗号化エンジンが必須 (古いレガシー機器では動作不可) |
| **対応機種** | Cisco Catalyst 9800 シリーズ WLC、AireOS WLC (8.x以降)、Cisco Catalyst / Meraki アクセスポイント、IOS-XE Wireless |
| **制限事項** | WPA3 / 6GHz 帯 (Wi-Fi 6E/7) では TKIP/WEP の使用が不可となり、AES (CCMP/GCMP) および PMF (Protected Management Frames) が必須 (Mandatory) 化 |
| **設計上の注意点** | AES-GCMP-256 (192-bit Mode) 導入時は、クライアントのNICおよびドライバーが CNSA Suite (GCMP-256 / EAP-TLS 192-bit) に対応しているか事前検証が不可欠 |


---

## 🏗 動作原理

無線LANにおいて AES を適用する場合、暗号化処理モードとして **CCMP (Counter Mode with CBC-MAC)** または **GCMP (Galois/Counter Mode)** が使用されます。どちらも「暗号化」と「メッセージ認証 (MIC)」を同時に達成する AEAD (Authenticated Encryption with Associated Data) 構造を持っています。

### 1. AES-CCMP の動作構造

AES-CCMP は、AES ブロック暗号 (128-bit ブロックサイズ) を2つの異なる動作モードで組み合わせて機能します。

* **CTR Mode (Counter Mode)**: ペイロードデータの暗号化に使用。48ビットの Packet Number (PN) から生成されたカウンタブロックを AES で暗号化し、平文データと XOR 演算することで暗号文を生成します。
* **CBC-MAC (Cipher Block Chaining Message Authentication Code)**: メッセージ整合性コード (MIC) の計算に使用。802.11 ヘッダー（AAD: Additional Authentication Data）と平文ペイロードを暗号ブロックチェーン方式で順次 AES 暗号化し、最終ブロックの先頭8バイト (64-bit) を MIC として抽出します。

```
[ 平文データ (Payload) ]                   [ 802.11 Header (AAD) ]
          │                                           │
          ├───────────────────────────┐               │
          ▼                           ▼               ▼
   [ CTR Mode 暗号化 ]         [ CBC-MAC (MIC 計算) ] ◄──┘
   (Counter + 128-bit TK)      (128-bit TK + AAD + Payload)
          │                               │
          ▼                               ▼
    [ 暗号化データ ]             [ 64-bit MIC (CBC-MAC) ]
          │                               │
          └───────────────┬───────────────┘
                          ▼
             [ 送信 MPDU フレーム構築 ]
             (802.11 Header + CCMP Header + Encrypted Payload + MIC)
```

### 2. AES-GCMP の動作構造

AES-GCMP は、ガロア体 $GF(2^{128})$ 上での乗算処理（GHASH）を利用した Galois/Counter Mode を採用しています。

* **CTR Mode**: カウンタモードによる超高速暗号化。
* **GHASH (Galois Hash)**: パケット全体の改ざん検知タグ (128-bit Auth Tag / MIC) を非対称演算なしに高速生成。CBC-MAC のようにブロックを逐次処理する必要がなく、パイプライン並列処理が可能なため、10Gbps〜100Gbps 超のハードウェア処理に非常に適しています。

```
[ Packet Number (PN) ] ───> [ J0 (Initial Counter Block) ]
                                    │
                                    ▼
[ 平文 Payload ] ───(AES-CTR)───> [ 暗号化 Payload ]
                                    │
802.11 Header (AAD) ──┐             │
                      ▼             ▼
               [ GHASH (Galois Multiplier) ]
                            │
                            ▼
              [ 128-bit Auth Tag (MIC) ]
```

### 3. Packet Number (PN) と AAD の役割

* **48-bit Packet Number (PN)**:
  * パケットが送信されるたびに 1 ずつ自動インクリメントされる 48 ビット（6バイト）のカウンタ。
  * 受信側は、送信元 MAC ごとに保持している前回受信 PN 値と比較し、`受信 PN <= 前回 PN` の場合はリプレイ攻撃と判定して即座にフレームを破棄します。
  * PN は 248 回（約 280 兆回）まで重複しないため、鍵のライフタイム中に PN が周回 (Rollover) することはありません。
* **AAD (Additional Authentication Data)**:
  * 802.11 ヘッダーの特定フィールド（送信元/宛先 MAC アドレス、フレーム制御情報、QoS 優先度など）が含まれます。
  * AAD は暗号化されませんが、MIC の計算対象に含まれるため、攻撃者が MAC アドレスや QoS 優先度を途中で改ざんした場合でも受信側で検知可能です。


---

## ⚙ 動作シーケンス

AES-CCMP / GCMP におけるフレームの送信側暗号化シーケンスおよび受信側復号・検証シーケンスを以下に説明します。

### 送信側暗号化シーケンス

```
[送信側: AP / クライアント]

1. Temporal Key (TK) の取得:
   4-Way Handshake 完了により、128ビット (CCMP-128 / GCMP-128) または 256ビット (GCMP-256) の TK を共有。

2. Packet Number (PN) のインクリメント:
   該当 TID (Traffic Identifier) の 48-bit PN を 1 インクリメント。

3. AAD (Additional Authentication Data) の構築:
   802.11 ヘッダー情報（Frame Control, Address 1, Address 2, Address 3, Sequence Control, Address 4, QoS Control）から AAD を抽出・形成。

4. Nonce (13 bytes) の生成:
   Priority (1 byte) + Transmit Address (TA: 6 bytes) + PN (6 bytes) を結合して固有の Nonce を作成。

5. MIC (改ざん検知コード) の計算:
   - CCMP: AES-CBC-MAC を用い、TK + Nonce + AAD + 平文 Payload から 8バイト (64-bit) の MIC を生成。
   - GCMP: GHASH を用い、TK + Nonce + AAD + 平文 Payload から 16バイト (128-bit) の Auth Tag (MIC) を生成。

6. ペイロードの暗号化:
   AES-CTR モードを使用し、Nonce に基づくカウンタブロックと平文 Payload を XOR 演算して暗号化。

7. CCMP / GCMP ヘッダー付与とフレーム送出:
   802.11 ヘッダーと暗号化ペイロードの間に 8バイトの CCMP/GCMP ヘッダー (PN0-PN5, Key ID, Extended IV) を挿入して送出。
```

### 受信側復号・検証シーケンス

```
[受信側: AP / クライアント]

1. CCMP / GCMP ヘッダーの解析:
   受信フレームから 6バイトの PN (Packet Number) および Key ID を抽出。

2. リプレイ攻撃チェック (Replay Protection):
   抽出した PN を、該当クライアントの既知最大 PN 値と比較。
   [判定] 受信 PN <= 既知最大 PN ──> リプレイ攻撃とみなしフレームをドロップ (Replay Error カウンタ加算)。
   [判定] 受信 PN > 既知最大 PN ──> 検証継続。

3. Nonce および AAD の再構築:
   受信した 802.11 ヘッダーおよび抽出した PN / TA から Nonce と AAD を再作成。

4. ペイロードの復号:
   共有 Temporal Key (TK) と Nonce を用い、AES-CTR モードで暗号化ペイロードを平文に復号。

5. MIC (改ざん検知コード) の検証:
   - 復号された平文 Payload + AAD + Nonce + TK より MIC を再計算。
   - フレーム末尾に添付された MIC と一致するか照合。
   - [判定] MIC 不一致 ──> 改ざんと判断しフレームをドロップ (CCMP/GCMP Decrypt/MIC Error 加算)。
   - [判定] MIC 一致 ──> 正常フレームとして受信処理を行い、既知最大 PN を受信 PN で更新。
```


---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、AES (CCMP/GCMP) に関連する必須知識とハマりやすい注意点は以下の通りです。

### 1. 試験で問われる重要ポイント

* **WPA2 と WPA3 の暗号スイート要件の違い**:
  * **WPA2-Personal / Enterprise**: AES-CCMP-128 が標準。
  * **WPA3-Personal (SAE)**: AES-CCMP-128 が基本。PMF (802.11w) は **Mandatory (必須)**。
  * **WPA3-Enterprise 128-bit Mode**: AES-CCMP-128 または AES-GCMP-128。PMF は Mandatory。
  * **WPA3-Enterprise 192-bit Mode (CNSA Suite)**: 暗号スイートは **AES-GCMP-256**、HMAC-SHA384、EAP-TLS (ECDHE-P-384 + ECDSA-P-384 / RSA-3072)、PMF は **BIP-GMAC-256** が必須。
* **PMF (Protected Management Frames / IEEE 802.11w) と AES**:
  * PMF のユニキャスト管理フレーム保護には AES-CCMP / GCMP を使用。
  * PMF のマルチキャスト/ブロードキャスト管理フレーム保護（Action フレーム等）には **BIP-CMAC-128** または **BIP-GMAC-256** が使用されます。
  * WPA3 や 6GHz 帯 (Wi-Fi 6E) SSID を設定する際、`pmf mandatory` を設定し忘れると Commit エラーが発生するか、WPA3 規格違反によりクライアントがアソシエートできなくなります。

### 2. よくある設定ミス・トラップ

* **WPA2/WPA3 Transition Mode における PMF の設定ミス**:
  * Transition Mode (WPA2-PSK + WPA3-SAE 同一SSID) では、PMF を `Optional` に設定する必要があります。`Mandatory` にすると WPA2 専用の古い端末が接続できなくなります。
* **WPA3 192-bit Mode での暗号不適合**:
  * WLAN 構成で `wpa3-ent-192` を有効にした場合、WPA エレメントとして `GCMP256` を選択し、かつ EAP 認証プロファイルで SHA-384 以上の強固な暗号を強制する必要があります。

### 3. CLI / Show コマンドによる解析ポイント

WLC 上で AES 暗号化および PMF が正しく適用されているか確認する際の核心コマンド：

* `show wlan id <id>`: WLAN の Security 設定（WPA2/WPA3, Ciphers: AES (CCMP) / GCMP256, PMF 状態）を確認。
* `show wireless client mac <MAC> detail`: クライアントのアソシエーション状態、ネゴシエートされた Cipher (CCMP-128 / GCMP-256)、PMF Status (Enabled/Disabled) を確認。


---

## 🛠 設定方法

Cisco Catalyst 9800 WLC (IOS-XE) における AES 暗号化設定例（WPA2-Enterprise AES-CCMP および WPA3-Enterprise 192-bit Mode AES-GCMP-256）を示します。

### 1. Catalyst 9800 WLC: WPA2-Enterprise (AES-CCMP-128) の設定 (CLI)

```bash
# RADIUS サーバー定義
radius server ISE_PRIMARY
 address ipv4 192.168.10.50 auth-port 1812 acct-port 1813
 key Cisco123!

# AAA 認証グループ設定
aaa group server radius RADIUS_GROUP_ISE
 server name ISE_PRIMARY

aaa authentication dot1x ISE_DOT1X group RADIUS_GROUP_ISE

# WLAN (SSID) 設定: WPA2-Enterprise AES-CCMP
wlan WLAN_CORP_WPA2 10 CORP_NET
 security dot1x authentication-list ISE_DOT1X
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security pmf optional
 no shutdown
```

### 2. Catalyst 9800 WLC: WPA3-Enterprise 192-bit Mode (AES-GCMP-256) の設定 (CLI)

```bash
# WLAN 設定: WPA3 192-bit CNSA Mode (GCMP-256 + BIP-GMAC-256)
wlan WLAN_SECURE_192BIT 11 SECURE_192
 security dot1x authentication-list ISE_DOT1X
 security wpa wpa3
 security wpa akm 192-bit
 security wpa wpa3 ciphers gcmp256
 security pmf mandatory
 no shutdown
```


## 🔍 検証コマンド

Cisco Catalyst 9800 WLC (IOS-XE) および AireOS WLC における AES 暗号化、PMF、アソシエーション状態の確認・検証コマンドを下表に示します。

| 目的 | コマンド |
| --- | --- |
| WLANのAES/PMF設定確認 (9800) | <code>show wlan id <WLAN_ID></code> |
| クライアントのネゴシエーション暗号確認 (9800) | <code>show wireless client mac <MAC_ADDR> detail</code> |
| クライアント暗号・キー状態確認 (9800) | <code>show wireless client mac <MAC_ADDR> summary</code> |
| PMF (802.11w) アソシエーション状態確認 (9800) | <code>show wireless client mac <MAC_ADDR> key</code> |
| APにおけるRadio暗号暗号化統計確認 | <code>show ap name <AP_NAME> dot11 5ghz operational</code> |
| WPA3/AES デバッグ追跡 (9800 Radioactive Trace) | <code>debug wireless mac <MAC_ADDR></code><br><code>show logging profile wireless filter mac <MAC_ADDR></code> |
| 4-Way Handshake / EAP デバッグ (9800) | <code>debug client-mac <MAC_ADDR></code> |
| WLAN暗号設定確認 (AireOS) | <code>show wlan <WLAN_ID></code> |
| クライアントCipher/PMF確認 (AireOS) | <code>show client detail <MAC_ADDR></code> |


---

## 🚨 トラブルシュート

AES (CCMP/GCMP) および WPA2/WPA3 暗号化接続における代表的な障害症状、根本原因、確認コマンド、対処方法を下表に示します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| WPA3-SAE クライアントが接続処理途中で切断される | PMF が Disabled になっている、またはドライバーが WPA3 SAE / PMF に未対応 | <code>show wlan id <ID></code><br><code>show wireless client mac <MAC> detail</code> | WPA3 では PMF (802.11w) が必須要件。WLANのPMF設定を Mandatory に変更し、端末ドライバーを更新 |
| WPA2 専用レガシー端末が Transition Mode SSID にアソシエートできない | WLAN の PMF 設定が Mandatory になっている | <code>show wlan id <ID></code> | WPA2/WPA3 混在 (Transition Mode) の場合、PMF 設定を <code>optional</code> に変更 |
| WPA3-Enterprise 192-bit Mode でクライアントの EAP 認証が失敗する | クライアントが GCMP-256 や EAP-TLS SHA-384 / Suite B 暗号スイートに対応していない | <code>show wireless client mac <MAC> detail</code> | 端末の NIC/ドライバーが CNSA (192-bit Mode) に適合しているか確認。不適合な場合は WPA3-Enterprise 128-bit Mode を使用 |
| データ通信中に頻繁にパケットドロップが発生し、リプレイエラーが記録される | 隣接AP間での PN (Packet Number) 同期不整合、または中間でのフレーム重複伝送 | <code>show wireless client mac <MAC> detail</code> | Radioactive Trace (<code>debug wireless mac</code>) で Decrypt/PN Error を解析。11r FT 設定または AP ファームウェアの不具合を確認 |
| 6GHz 帯 (Wi-Fi 6E) SSID にアソシエートできない | WLAN に WPA2 や TKIP 暗号が設定されている | <code>show wlan id <ID></code> | 6GHz 帯では WPA3 (SAE / OWE / Enterprise) かつ AES (CCMP/GCMP) + PMF Mandatory のみがサポートされるため WLAN 設定を修正 |


---

## ⚠ 制限事項

無線LAN環境における AES 暗号化および関連プロトコルの制限事項は以下の通りです。

1. **レガシー暗号との非互換性**:
   * 802.11n / 802.11ac / 802.11ax / 802.11be 規格において、TKIP または WEP を使用した場合、High Throughput (HT) データレートが完全に無効化され、最大通信速度が 54 Mbps に強制制限されます。
2. **Wi-Fi 6E / 7 (6GHz 帯) の厳格制限**:
   * 6GHz 帯周波数においては、WPA2、TKIP、WEP、非保護 Open ネットワークの設定が規格上禁止されています。AES-CCMP/GCMP を用いた WPA3 (SAE / Enterprise) または Enhanced Open (OWE) かつ PMF Mandatory が必須です。
3. **WPA3-Enterprise 192-bit Mode (CNSA) のハードウェア制約**:
   * GCMP-256 および BIP-GMAC-256 は暗号化アクセラレータハードウェアのサポートを要するため、古いクライアント端末（iPhone 8以前や旧型のWi-Fi 5クライアントカード）では動作しません。


---

## 🔄 他技術との関連

AES 暗号化技術は、無線LANセキュリティを構成する他の要素技術と密接に連携しています。

* **802.1X / EAP 認証**: 802.1X/EAP (EAP-TLS, PEAP 等) 認証成功後、Master Session Key (MSK) から Pairwise Master Key (PMK) が生成され、これが 4-Way Handshake を通じて AES-CCMP/GCMP 用の 128/256-bit Temporal Key (TK) に変換されます。
* **WPA3 SAE (Simultaneous Authentication of Equals)**: PSK (事前共有鍵) に代わる Dragonfly ハンドシェイクにより PMK を生成し、AES-CCMP-128 暗号化を確立します。
* **PMF (Protected Management Frames / IEEE 802.11w)**: ユニキャスト管理フレームを AES-CCMP/GCMP で暗号化し、マルチキャスト管理フレームを BIP-CMAC-128 または BIP-GMAC-256 で改ざん保護します。
* **Cisco FlexConnect / Mesh**: CAPWAP トンネル内でのデータフレーム処理において、Local Switching モードでは AP のハードウェア暗号化エンジンが直接 AES-CCMP 処理を実施します。


---

## 🧩 比較表

無線LANにおける主要な暗号化プロトコルの技術比較を下表に示します。

| 比較項目 | WEP | TKIP (WPA) | AES-CCMP-128 (WPA2/3) | AES-GCMP-256 (WPA3 192-bit) |
| --- | --- | --- | --- | --- |
| **暗号アルゴリズム** | RC4 (24-bit IV) | RC4 (48-bit IV + PPK) | AES (128-bit Block) | AES (256-bit Block) |
| **動作モード** | Stream Cipher | Stream Cipher + Mixing | CTR + CBC-MAC (CCMP) | Galois/Counter Mode (GCMP) |
| **鍵長 (TK)** | 40-bit / 104-bit | 128-bit | 128-bit | 256-bit |
| **改ざん検知 (MIC)** | CRC-32 (ICV) | Michael (64-bit MIC) | CBC-MAC (64-bit MIC) | GHASH Auth Tag (128-bit) |
| **リプレイ防御** | なし | 48-bit TSC | 48-bit PN | 48-bit PN |
| **処理方式** | ソフトウェア/HW | ソフトウェア/HW | ハードウェア推奨 | HW並列処理必須 |
| **最大サポート速度** | 54 Mbps | 54 Mbps | 10Gbps+ (Wi-Fi 6/6E) | 10Gbps+ (Wi-Fi 6E/7 CNSA) |
| **セキュリティレベル** | 非常に脆弱 (破綻) | 脆弱 (非推奨) | 強固 (業界標準) | 極めて強固 (政府・軍事級) |


---

## 💡 ベストプラクティス

1. **WPA3-SAE への段階的移行 (Transition Mode の活用)**:
   * 既存の WPA2-PSK 端末が多数残存している環境では、WPA2-PSK + WPA3-SAE の Transition Mode SSID を構築し、`pmf optional` に設定して段階的に WPA3 へ移行します。
2. **新規エンタープライズ構成では WPA3-Enterprise 128-bit を標準採用**:
   * Wi-Fi 6/6E 環境では WPA3-Enterprise (AES-CCMP-128 / GCMP-128 + PMF Mandatory) を基本設計とします。
3. **6GHz 帯 (Wi-Fi 6E) 用 SSID の分離設計**:
   * 6GHz 帯では WPA2/TKIP が完全に禁止されているため、2.4GHz/5GHz 帯の Transition SSID とは別に、6GHz 専用の WPA3-SAE / OWE / WPA3-Enterprise SSID を設計・配置します。


## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験に対応した実効性の高い設定演習例を10個掲載します。設定例は省略せず最後まで完全に出力しています。

### 演習1: Catalyst 9800 WLC — WPA2-Enterprise (AES-CCMP-128) と RADIUS 認証

#### 問題
企業内エンタープライズ無線環境として、802.1X/EAP 認証と AES-CCMP-128 暗号化を使用する SSID "CORP_WPA2" を構築しなさい。

#### 要件
* RADIUS サーバー (192.168.10.50, Key: Cisco123!) を使用。
* WPA2-Enterprise (dot1x) を使用し、暗号化アルゴリズムは AES (CCMP-128) のみを許可すること。
* 802.11w PMF は Optional とすること。

#### 設定例

```bash
# 1. RADIUS サーバーおよびグループの設定
radius server ISE_PRIMARY
 address ipv4 192.168.10.50 auth-port 1812 acct-port 1813
 key Cisco123!

aaa group server radius RADIUS_GROUP_ISE
 server name ISE_PRIMARY

aaa authentication dot1x ISE_DOT1X_POLICY group RADIUS_GROUP_ISE

# 2. WLAN (SSID) 設定
wlan WLAN_CORP_WPA2 10 CORP_WPA2
 security dot1x authentication-list ISE_DOT1X_POLICY
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security pmf optional
 no shutdown

# 3. Policy Profile への紐付けと有効化
wireless profile policy POLICY_CORP
 vlan 100
 no shutdown

wireless tag policy TAG_CORP
 wlan CORP_WPA2 policy POLICY_CORP
```

#### 検証方法

```bash
# WLAN のセキュリティ設定および AES-CCMP 有効化の確認
show wlan id 10

# 接続クライアントのネゴシエート暗号 (CCMP-128) の確認
show wireless client mac 0011.2233.4455 detail
```


---

### 演習2: Catalyst 9800 WLC — WPA3-Enterprise 192-bit CNSA Mode (AES-GCMP-256)

#### 問題
政府機関・軍事レベルのセキュリティ要件に基づき、WPA3-Enterprise 192-bit Mode (CNSA Suite) を用いた SSID "SECURE_CNSA" を構成しなさい。

#### 要件
* 暗号方式として AES-GCMP-256 を指定すること。
* AKM として 192-bit を指定し、PMF (Protected Management Frames) を Mandatory (必須) とすること。

#### 設定例

```bash
# 1. AAA 設定
aaa group server radius RADIUS_CNSA_GROUP
 server name ISE_PRIMARY

aaa authentication dot1x DOT1X_CNSA group RADIUS_CNSA_GROUP

# 2. WPA3 192-bit Mode WLAN 設定
wlan WLAN_CNSA 11 SECURE_CNSA
 security dot1x authentication-list DOT1X_CNSA
 security wpa wpa3
 security wpa akm 192-bit
 security wpa wpa3 ciphers gcmp256
 security pmf mandatory
 no shutdown

# 3. Policy Profile 設定
wireless profile policy POLICY_CNSA
 vlan 200
 no shutdown

wireless tag policy TAG_CNSA
 wlan SECURE_CNSA policy POLICY_CNSA
```

#### 検証方法

```bash
# WLAN 設定で GCMP256 および PMF Mandatory を確認
show wlan id 11

# クライアントが GCMP-256 および BIP-GMAC-256 でアソシエートしているか確認
show wireless client mac 0011.2233.4455 detail
```


---

### 演習3: Catalyst 9800 WLC — WPA3-Personal (SAE) と AES-CCMP-128

#### 問題
WPA3-Personal (SAE) を使用して、事前共有鍵の辞書攻撃を防ぐ SSID "GUEST_WPA3" を構築しなさい。

#### 要件
* SAE (Simultaneous Authentication of Equals) パスワードとして "SecretPass123!" を設定すること。
* 暗号化は AES-CCMP-128 を使用し、PMF は Mandatory とすること。

#### 設定例

```bash
# WLAN 設定: WPA3-SAE
wlan WLAN_GUEST_SAE 12 GUEST_WPA3
 security wpa wpa3
 security wpa akm sae
 security wpa wpa3 ciphers aes
 security wpa sae password SecretPass123!
 security pmf mandatory
 no shutdown

wireless profile policy POLICY_GUEST
 vlan 300
 no shutdown

wireless tag policy TAG_GUEST
 wlan GUEST_WPA3 policy POLICY_GUEST
```

#### 検証方法

```bash
# SAE 認証および PMF Mandatory 設定の確認
show wlan id 12

# クライアントの AKM が SAE であることを確認
show wireless client mac 0011.2233.4455 detail
```


---

### 演習4: Catalyst 9800 WLC — Enhanced Open (OWE) と AES-GCMP-128

#### 問題
パスワード入力なしで暗号化通信を提供する Opportunistic Wireless Encryption (OWE / Enhanced Open) SSID "OPEN_SECURE" を設定しなさい。

#### 要件
* 認証なし (Open) でありながら、Diffie-Hellman による自動鍵交換と AES-GCMP-128 または AES-CCMP-128 による個別暗号化を行うこと。
* PMF は Mandatory とすること。

#### 設定例

```bash
wlan WLAN_OWE 13 OPEN_SECURE
 security wpa wpa3
 security wpa akm owe
 security wpa wpa3 ciphers gcmp128 aes
 security pmf mandatory
 no shutdown

wireless profile policy POLICY_OWE
 vlan 400
 no shutdown
```

#### 検証方法

```bash
# OWE AKM および PMF Mandatory 設定の確認
show wlan id 13

# クライアントが OWE でアソシエートし、個別暗号鍵が生成されていることを確認
show wireless client mac 0011.2233.4455 detail
```


---

### 演習5: AireOS WLC — WPA2-PSK (AES-CCMP) と PMF Optional の設定

#### 問題
AireOS WLC において、WPA2-PSK (AES-CCMP) および PMF Optional を適用した SSID "AIRE_WPA2" を CLI から構成しなさい。

#### 要件
* PSK パスワード: "AireOSPassword123"
* 暗号化: AES-CCMP (AES Enabled, TKIP Disabled)
* PMF: Optional (PMF Client 許可、非対応クライアントも接続可)

#### 設定例

```bash
# WLAN の作成と SSID 設定
config wlan create 5 AIRE_WPA2 AIRE_WPA2
config wlan ssid 5 AIRE_WPA2

# セキュリティ設定: WPA2 PSK + AES
config wlan security wpa disable 5
config wlan security wpa wpa2 enable 5
config wlan security wpa wpa2 ciphers aes enable 5
config wlan security wpa wpa2 ciphers tkip disable 5
config wlan security wpa wpa2 psk set-key ascii AireOSPassword123 5

# PMF (802.11w) 設定: Optional
config wlan security pmf optional 5

# WLAN 有効化
config wlan enable 5
```

#### 検証方法

```bash
# WLAN 設定の確認
show wlan 5

# クライアントのアソシエーション状態確認
show client detail 00:11:22:33:44:55
```


---

### 演習6: Catalyst 9800 WLC — FlexConnect Local Switching での AES 暗号化

#### 問題
拠点（Branch）アクセスポイント向けに、ローカルでデータフレームをスイッチングしつつ、AES-CCMP 暗号化を維持する FlexConnect SSID "BRANCH_WPA2" を構築しなさい。

#### 要件
* 認証は本社の RADIUS サーバーで集中実行（Central Authentication）。
* データフレームの復号・スイッチングは AP 上でローカル実行（Local Switching）。

#### 設定例

```bash
# 1. WLAN 設定
wlan WLAN_FLEX 14 BRANCH_WPA2
 security dot1x authentication-list ISE_DOT1X_POLICY
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security pmf optional
 no shutdown

# 2. FlexConnect 対応 Policy Profile 設定
wireless profile policy POLICY_FLEX
 flex-local-switching
 flex-central-auth
 flex-central-dhcp
 vlan 500
 no shutdown

wireless tag policy TAG_FLEX
 wlan BRANCH_WPA2 policy POLICY_FLEX
```

#### 検証方法

```bash
# Policy Profile で flex-local-switching が有効であることを確認
show wireless profile policy detailed POLICY_FLEX

# クライアントの Data Switching が "Local" であり、Cipher が "CCMP" であることを確認
show wireless client mac 0011.2233.4455 detail
```


---

### 演習7: Catalyst 9800 WLC — Identity PSK (iPSK) と AES-CCMP-128

#### 問題
単一の SSID "IOT_IPSK" で、デバイスごとに異なる MAC/PSK を動的割り当てし、AES-CCMP-128 で保護する iPSK 環境を構成しなさい。

#### 要件
* MAC Authentication (MAC filtering) を有効にし、RADIUS (ISE) から Dynamic PSK および Dynamic VLAN を返却させること。
* 暗号化は AES-CCMP-128 を適用すること。

#### 設定例

```bash
# MAC 認証用 AAA 設定
aaa authorization network ISE_MAC_AUTH group RADIUS_GROUP_ISE

# WLAN 設定 (iPSK)
wlan WLAN_IPSK 15 IOT_IPSK
 mac-filtering ISE_MAC_AUTH
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security wpa akm psk
 security pmf optional
 no shutdown

wireless profile policy POLICY_IPSK
 aaa-override
 vlan 600
 no shutdown
```

#### 検証方法

```bash
# WLAN で mac-filtering が設定されていることを確認
show wlan id 15

# クライアントのダイナミックPSK適用状態および AES 暗号を確認
show wireless client mac 0011.2233.4455 detail
```


---

### 演習8: AireOS WLC — WPA3-Enterprise 192-bit CNSA Mode (GCMP-256)

#### 問題
AireOS WLC において、WPA3-Enterprise 192-bit CNSA Mode (AES-GCMP-256) SSID "AIRE_CNSA" を作成しなさい。

#### 要件
* WPA3 192-bit Mode 有効化。
* 暗号化: GCMP-256、PMF: Required (Mandatory)。

#### 設定例

```bash
config wlan create 6 AIRE_CNSA AIRE_CNSA
config wlan security wpa disable 6
config wlan security wpa wpa3 enable 6
config wlan security wpa wpa3 ciphers gcmp256 enable 6
config wlan security wpa wpa3 akm 192-bit enable 6
config wlan security pmf required 6
config wlan enable 6
```

#### 検証方法

```bash
show wlan 6
show client detail 00:11:22:33:44:55
```


---

### 演習9: Catalyst 9800 WLC — 802.11w PMF (Protected Management Frames) による Deauth 攻撃防御

#### 問題
SSID "CORP_PMF_STRICT" において、Deauth 攻撃による強制切断を防ぐため、802.11w PMF を強制 (Mandatory) 適用しなさい。

#### 要件
* WPA2-Enterprise (AES-CCMP-128) と組み合わせて PMF Mandatory を設定すること。
* SA Query Timeout を 200ms に調整すること。

#### 設定例

```bash
wlan WLAN_PMF_STRICT 16 CORP_PMF_STRICT
 security dot1x authentication-list ISE_DOT1X_POLICY
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security pmf mandatory
 security pmf saquery-retry-time 200
 no shutdown

wireless profile policy POLICY_PMF
 vlan 100
 no shutdown
```

#### 検証方法

```bash
# WLAN の PMF 状態が "Mandatory" であることを確認
show wlan id 16

# クライアントの 802.11w PMF アソシエーション状態が "Enabled" であることを確認
show wireless client mac 0011.2233.4455 detail
```


---

### 演習10: Catalyst 9800 WLC — WPA2/WPA3 Transition Mode (WPA2 AES-CCMP / WPA3 SAE)

#### 問題
既存端末と新規格端末が同一 SSID "TRANSITION_NET" に接続できるよう、WPA2-PSK (AES-CCMP) と WPA3-SAE の Transition Mode を構成しなさい。

#### 要件
* AKM として psk と sae の両方を許可すること。
* PMF は Transition Mode 必須条件である `optional` に設定すること。

#### 設定例

```bash
wlan WLAN_TRANSITION 17 TRANSITION_NET
 security wpa wpa2 wpa3
 security wpa akm psk sae
 security wpa wpa2 ciphers aes
 security wpa wpa3 ciphers aes
 security wpa sae password TransitionPass123!
 security pmf optional
 no shutdown

wireless profile policy POLICY_TRANSITION
 vlan 100
 no shutdown

wireless tag policy TAG_TRANSITION
 wlan TRANSITION_NET policy POLICY_TRANSITION
```

#### 検証方法

```bash
# WLAN で AKM (psk, sae) および PMF Optional を確認
show wlan id 17

# WPA2 端末と WPA3 端末がそれぞれ正しい AKM で接続されているか確認
show wireless client summary
show wireless client mac 0011.2233.4455 detail
```


## ❓ 想定試験問題

CCIE Security ラボ試験を想定した実践的な問題を5問掲載します。

### 設問 1: 6GHz (Wi-Fi 6E) 用 SSID 設定の不具合トラブルシューティング

**[問題]**
エンジニアが Catalyst 9800 WLC 上に 6GHz 帯をサポートする SSID "WLAN_6GHZ" を構成した。しかし、Wi-Fi 6E 対応端末が 6GHz 帯で SSID を認識できず、接続もできない。WLAN の設定の一部は以下の通りである。

```bash
wlan WLAN_6GHZ 20 WLAN_6GHZ
 security wpa wpa2 wpa3
 security wpa akm psk
 security wpa wpa2 ciphers aes
 security pmf optional
 no shutdown
```

この設定の何が問題か。Wi-Fi 6E (6GHz 帯) 規格の制限を挙げて正しいコンフィグに修正しなさい。

**[解答・解説]**
Wi-Fi 6E (6GHz 帯) 規格では、以下の2点が厳格に規定されています。
1. **WPA2 および PSK の使用禁止**: 6GHz 帯では WPA3 (SAE / OWE / Enterprise) のみがサポートされ、WPA2 および従来の PSK は使用できません。
2. **PMF (802.11w) の Mandatory 必須化**: PMF を `optional` や `disabled` にすることは禁止されています。

修正設定例:

```bash
wlan WLAN_6GHZ 20 WLAN_6GHZ
 security wpa wpa3
 security wpa akm sae
 security wpa wpa3 ciphers aes
 security wpa sae password Secret6GHzPass!
 security pmf mandatory
 no shutdown
```


---

### 設問 2: Transition Mode SSID におけるレガシー端末接続不可のトラブルシューティング

**[問題]**
既存の WPA2-PSK 端末と新しい WPA3-SAE 端末の両方を単一の SSID "CORP_HYBRID" で収容するため、Transition Mode を設定した。しかし、旧型の WPA2 専用産業用端末が一切アソシエートできなくなった。WLAN の設定は以下の通りである。

```bash
wlan CORP_HYBRID 21 CORP_HYBRID
 security wpa wpa2 wpa3
 security wpa akm psk sae
 security wpa wpa2 ciphers aes
 security pmf mandatory
 no shutdown
```

障害の根本原因を特定し、両端末が問題なく接続できるように設定を修正しなさい。

**[解答・解説]**
障害の原因は `security pmf mandatory` にあります。
旧型の WPA2 専用端末の多くは 802.11w PMF (Protected Management Frames) に対応していません。PMF が `mandatory` に設定されていると、PMF 非対応端末からのアソシエーション要求が WLC によって拒否されます。
Transition Mode で WPA2 端末を併存させる場合、PMF 設定は必ず `optional` にする必要があります。

修正設定例:

```bash
wlan CORP_HYBRID 21 CORP_HYBRID
 security pmf optional
```


---

### 設問 3: WPA3-Enterprise 192-bit CNSA Mode コンフィグ読解と設計問題

**[問題]**
以下の Catalyst 9800 WLC のコンフィグレーションを読み取り、この WLAN に接続するためにクライアント端末が満たすべき暗号化および認証の要件を 3 点答えなさい。

```bash
wlan WLAN_CNSA_GOV 30 CNSA_GOV
 security dot1x authentication-list ISE_DOT1X
 security wpa wpa3
 security wpa akm 192-bit
 security wpa wpa3 ciphers gcmp256
 security pmf mandatory
 no shutdown
```

**[解答・解説]**
クライアント端末が満たすべき要件：
1. **暗号化スイート**: AES-GCMP-256 (256ビット Galois/Counter Mode) および BIP-GMAC-256 (管理フレーム保護) に対応していること。
2. **AKM / EAP 認証**: AKM 192-bit モードおよび CNSA Suite に適合する強固な EAP 認証（SHA-384 以上のハッシュ、ECDHE-P-384 以上の楕円曲線ディフィー・ヘルマン鍵交換、および ECDSA-P-384 または RSA-3072 以上の証明書）に対応していること。
3. **802.11w PMF (Protected Management Frames)**: PMF 機能が有効であり、アソシエーション時に PMF ネゴシエーションが成功すること。


### 設問 4: Radioactive Trace ログ解析問題（Decrypt/PN Error）

**[問題]**
Catalyst 9800 WLC の Radioactive Trace ログで、特定の無線クライアント (MAC: `a43e.5112.9900`) から以下のログが記録され、データ通信が頻繁に途切れていた。

```text
2026/11/04 10:15:22.123 {wireless} [client-orphan]: a43e.5112.9900 CCMP Decrypt Error / PN Mismatch detected.
2026/11/04 10:15:22.124 {wireless} [dot11-security]: Received Packet Number (PN) 0x000000045F12 <= Current Expected PN 0x000000045F20. Frame dropped.
```

このログが示す現象と、想定される根本原因を説明しなさい。

**[解答・解説]**
* **現象**: クライアントから受信したフレームの Packet Number (PN: 0x45F12) が、WLC/AP が記憶している期待値 (0x45F20) 以下であるため、リプレイ攻撃防止機能 (Replay Protection) によってフレームが改ざん/リプレイパケットとみなされドロップされています。
* **原因**:
  1. クライアントの無線ドライバーの不具合により、PN の自動インクリメント順序が逆転・重複している。
  2. 802.11r FT (Fast BSS Transition) などの高速ローミング時、移動先 AP への Key / PN 同期が正常に行われなかった。
  3. 中間ネットワークやアタッカーによるリプレイパケットの挿入。


---

### 設問 5: PMF (Protected Management Frames) による Deauth 攻撃遮断メカニズム

**[問題]**
攻撃者が偽装した Deauthentication フレーム（送信元: 正常 AP の MAC アドレス）を送信してクライアントを切断させようとした。PMF (802.11w) が設定されている場合、クライアントおよび AP はこの攻撃をどのように防ぐか、動作シーケンスを説明しなさい。

**[解答・解説]**
1. **ユニキャスト Deauth フレームの受信**: クライアントは受信した Deauth フレームに有効な AES-CCMP/GCMP 暗号化（または MIC）が付与されているか検証します。
2. **非暗号化 Deauth の拒否**: 攻撃者が送信した偽装 Deauth フレームには正当な Temporal Key (TK) による暗号化が施されていないため、クライアントはこれを不正フレームとして即座に破棄します。
3. **SA Query メカニズムの起動**: 万一アソシエーション状態に不整合が生じた場合、クライアントは AP に対して暗号化された SA Query Request を送信し、AP からの保護された SA Query Response を受けることで、正当なセッションが維持されているか確認します。これにより切断攻撃は完全に無効化されます。


## 🔗 参考リソース

Cisco 公式ドキュメントおよび Cisco Live 資料一覧です。

* Cisco Catalyst 9800 Series Wireless Controller Software Configuration Guide, Cisco IOS XE 17.x
* Wi-Fi 6E Infrastructure Deployment Guide - Cisco Systems
* WPA3 Deployment Guide - Cisco Systems
* Cisco Technical Note: Understand Protected Management Frames (PMF) 802.11w
* Cisco Live BRKCRT-2101: CCIE Security - Wireless Security Integration Essentials
* Cisco Live BRKWVN-2020: WPA3 & Enhanced Open Security Technical Deep Dive
* Cisco Wireless Command Reference, Cisco IOS XE 17.x


---

## 📝 **補足（Notes）**

* **AES-CCMP の名称由来**: Counter mode with Cipher Block Chaining Message Authentication Code Protocol の略。128ビットブロック暗号 AES を用い、機密性保護に CTR モード、完全性保護に CBC-MAC を使用。
* **AES-GCMP の名称由来**: Galois/Counter Mode Protocol の略。ガロア体乗算演算を用いた Galois/Counter Mode を用い、10Gbps 超のハードウェア並列高速処理を実現。
* **BIP (Broadcast Integrity Protocol)**: マルチキャスト/ブロードキャスト管理フレーム（Beacon 以外の Action フレーム等）の偽装を防ぐため、AES-128-CMAC (BIP-CMAC-128) または AES-256-GMAC (BIP-GMAC-256) を用いて 802.11w PMF で保護します。


