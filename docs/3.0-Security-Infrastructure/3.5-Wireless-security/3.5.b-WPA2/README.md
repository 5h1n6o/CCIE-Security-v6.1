---
layout: default
title: 3.5.b-WPA2
nav_order: 2
parent: 3.5-Wireless-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.5.b WPA2

CCIE Security v6.1 Blueprint の 3.5.b「WPA2 (Wi-Fi Protected Access 2 / IEEE 802.11i)」に関する、試験直前復習用の高度な学習メモです。

---

## 📘 概要

* **機能概要**: WPA2 (IEEE 802.11i) は、従来の WEP や WPA (TKIP) のセキュリティ的脆弱性を根本的に解決するために標準化された無線 LAN セキュリティ規格です。堅牢な対称鍵暗号アルゴリズムである **AES (Advanced Encryption Standard)** と、パケットの改ざん防止および暗号化を同時に提供する **CCMP (Counter Mode with Cipher Block Chaining Message Authentication Code Protocol)** を必須要件として導入しています。
* **利用目的**: 無線区間（STA 〜 AP 間）の通信データの盗聴、改ざん、なりすまし、リプレイ攻撃を防御し、大規模エンタープライズから小規模拠点までのセキュアな無線アクセスを実現します。
* **どのような場面で利用するか**: 
  * **WPA2-Enterprise**: Cisco ISE や RADIUS サーバーと連携し、IEEE 802.1X/EAP (EAP-TLS, PEAP-MSCHAPv2, EAP-FAST) 認証を組み合わせる企業・団体向けネットワーク。ユーザー/端末ごとの動的 VLAN 割り当てや Dynamic ACL、SGT 割り当てを適用する環境。
  * **WPA2-Personal (PSK)**: 事前共有鍵 (Pre-Shared Key) を用いるネットワーク。IoT 機器、プリンタ、スマートデバイス、または簡易ゲストアクセス環境。

---

## 🔑 要点

WPA2 の主要な技術要素および実装上の注意点を以下にまとめます。

| 項目 | 内容 |
| :--- | :--- |
| **標準規格** | IEEE 802.11i（Wi-Fi Alliance による WPA2 認定プログラム） |
| **必須暗号プロトコル** | AES-CCMP（AES 128-bit ブロック暗号 + CBC-MAC メッセージ認証コード） |
| **オプション暗号** | TKIP（WPA 互換モードとして過渡期にサポートされたが、WPA2 単体では非推奨/廃止） |
| **認証方式** | WPA2-Personal (PSK) / WPA2-Enterprise (IEEE 802.1X / EAP + RADIUS) |
| **キー階層構造** | MSK / PMK (Pairwise Master Key) → PTK (Pairwise Transient Key) & GTK (Group Transient Key) |
| **ハンドシェイク** | 4-Way Handshake（PTK の相互導出と確認） / Group Key Handshake (2-Way) |
| **高速ローミング技術** | IEEE 802.11r (Fast BSS Transition - FT), OKC (Opportunistic Key Caching), PMK Caching |
| **対応機種** | Cisco Catalyst 9800 シリーズ WLC, AireOS WLC (8540, 5520, 3504), Cisco Meraki AP, Catalyst AP (CW9100, 9100) |
| **制限事項** | PSK モードにおける辞書攻撃リスク（4-Way Handshake のキャプチャによるオフライン解析）、管理フレーム (Management Frames) の暗号化非対応（802.11w PMF が無効な場合） |
| **設計上の注意点** | 802.11r FT の有効化時におけるレガシー端末の接続互換性、WPA2/WPA3 移行期における Transition Mode の WLC パラメータ設計 |

---

## 🏗 動作原理

WPA2-Enterprise における通信フローおよび認証・暗号化キーの生成プロセスは以下の通りです。

```text
[Supplicant (STA)]            [Authenticator (WLC/AP)]          [Authentication Server (Cisco ISE)]
        |                                 |                                    |
        |<=== 1. 802.11 Assoc / Open ====>|                                    |
        |                                 |                                    |
        |<================ 2. IEEE 802.1X / EAP Authentication ================>|
        |   (EAP-TLS / PEAP / EAP-FAST 等による相互認証とマスター鍵 MSK の生成)  |
        |                                 |                                    |
        |                                 |<======== RADIUS Access-Accept =====|
        |                                 |    (EAP-Success + MSK / AAA Attributes)
        |                                 |                                    |
  [PMK 導出]                               [PMK 受信/導出]
(MSK より生成)                          (RADIUS 属性より取得)
        |                                 |
        |<======= 3. 4-Way Handshake =====>|
        |  (ANonce / SNonce 交換による     |
        |   PTK (KCK, KEK, TK) 相互導出)    |
        |                                 |
        |<=== 4. Group Key Handshake ====>|
        |   (Encrypted GTK の配布)         |
        |                                 |
  [暗号化通信開始]                        [暗号化通信開始]
(AES-CCMP によるデータ保護)              (AES-CCMP によるデータ保護)
```

1. **マスターキー (MSK / PMK) の導出**:
   * **WPA2-Enterprise**: EAP 認証成功後、STA と ISE 間で生成された MSK (Master Session Key) から上位256ビットを抽出し、**PMK (Pairwise Master Key)** が導出されます。ISE は RADIUS Access-Accept (EAP-Key-Name 属性 / MS-MPPE-Recv-Key / MS-MPPE-Send-Key) にて WLC へ PMK を転送します。
   * **WPA2-Personal (PSK)**: `PMK = PBKDF2(Passphrase, SSID, 4096, 256)` アルゴリズムにより、パスフレーズと SSID 名から直接 PMK が計算されます。
2. **PTK (Pairwise Transient Key) の生成 (4-Way Handshake)**:
   * PMK, Authenticator Nonce (ANonce), Supplicant Nonce (SNonce), Authenticator MAC Address (AA), Supplicant MAC Address (SPA) を擬似乱数関数 (PRF-384/512) に入力し、**PTK** を生成します。
   * **PTK の内訳**:
     * **KCK (Key Confirmation Key)**: 4-Way Handshake フレームの MIC (Message Integrity Check) 計算。
     * **KEK (Key Encryption Key)**: GTK や IGTK などの鍵情報を暗号化して転送。
     * **TK (Temporal Key)**: 128-bit。実際に無線区間のユニキャストデータパケット（AES-CCMP）を暗号化。
3. **GTK (Group Transient Key) の配布**:
   * ブロードキャストおよびマルチキャストトラフィックを保護するための共通鍵。WLC/AP で生成され、KEK によって暗号化された上で 4-Way Handshake の Message 3 または独立した Group Key Handshake (2-Way) で STA に安全に配布されます。

---

## ⚙ 動作シーケンス

WPA2 の 4-Way Handshake パケット処理シーケンスの詳細は以下の通りです。

1. **Message 1 (WLC/AP → STA)**:
   * Authenticator は乱数 **ANonce** を生成し、STA に送信します。
   * パケットには Key Replay Counter が含まれます。MIC はまだ付与されません（STA 側で PTK が未計算のため）。
2. **Message 2 (STA → WLC/AP)**:
   * STA は自身の乱数 **SNonce** を生成し、受信した ANonce、自アドレス (SPA)、AP アドレス (AA)、PMK から **PTK** を計算します。
   * STA は SNonce と自身の RSNE (Robust Security Network Element) を含め、PTK 内の **KCK** を使って計算した **MIC** を付与して AP に送信します。
   * WLC/AP は受信した SNonce から同様に PTK を計算し、付与された MIC を検証します。MIC が一致すれば STA が正しい PMK を保持していることが証明されます。
3. **Message 3 (WLC/AP → STA)**:
   * WLC/AP は STA に対して、暗号化（**KEK** 使用）された **GTK**、AP の RSNE、Increment された Key Replay Counter、および **MIC** を付与して送信します。
   * これにより STA に対して、AP が正しく PTK を生成できたことを証明します。
4. **Message 4 (STA → WLC/AP)**:
   * STA は Message 3 の MIC および Replay Counter を検証後、ACK として MIC 付きの Message 4 を返答します。
   * この完了通知をもって、AP および STA は 802.1X ポートを AUTHORIZED 状態に移行させ、AES-CCMP によるユニキャスト/マルチキャストデータ通信が有効化されます。


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において WPA2 に関して問われる重要ポイント、設定上の必須要件、よくある不具合パターンを網羅します。

### Blueprint での重要ポイント
* **Catalyst 9800 (IOS-XE) / AireOS での WLAN プロファイル構築**:
  * **WPA2-Enterprise**: WLAN 設定において `AKM` (Authentication and Key Management) に `802.1X` を指定し、`CCMP128` (AES) 暗号化を有効化。
  * **WPA2-Personal**: AKM に `PSK` を指定し、`psk-format ascii` または `hex` で事前共有鍵を定義。
* **Fast Roaming (高速ローミング) の実装要件**:
  * **802.11r FT (Fast BSS Transition)**: `ft` (FT over-the-Air / FT over-the-DS) の設定。クライアントが 802.11r 非対応の場合の接続拒否トラブル防止要件（`ft adaptive` や PSK + FT 混合設定）。
  * **OKC (Opportunistic Key Caching)**: 同一 WLC 内の AP 間で PMK をキャッシュ・共有し、再認証時の 802.1X/EAP フルハンドシェイクをスキップする設定。
* **Protected Management Frames (PMF / 802.11w)**:
  * Deauthentication / Disassociation 攻撃を防ぐための管理フレーム保護。`pmf-mode` パラメータ（`disabled` / `optional` / `required`）。
  * WPA2 単体では `optional` が標準。WPA3 との Transition Mode では `optional` 必須。

### ラボ試験で出題されやすいシナリオ・設定要件
1. **ISE と連携した WPA2-Enterprise WLAN の作成**:
   * RADIUS Authentication / Accounting サーバーグループの定義と WLAN への紐付け。
   * `AAA Override` (Catalyst 9800 では Policy Profile 内の `allow-mic` / `aaa-override` 有効化) による動的 VLAN / SGT 属性の適用。
   * `CoA` (Change of Authorization - RFC 3576 / 5176) の有効化（Port 1700）。
2. **Identity PSK (iPSK) の構築**:
   * 同一 PSK 用 SSID で、MAC アドレスやユーザー属性ごとに異なる PSK (Unique PSK) を ISE 経由で動的割り当て（RADIUS 応答属性 `cisco-av-pair = psk-mode=ascii` / `psk=...`）。

### よくある設定ミスと制限事項
* **暗号方式のミスマッチ**: WPA2 では TKIP は完全排除または非推奨です。WLAN で `TKIP` を選択すると、802.11n / 802.11ac / 802.11ax (Wi-Fi 6) の高遅延・低速化（データレートが 54Mbps (802.11g 上限) に制限される）が発生します。必ず `AES (CCMP128)` のみを指定します。
* **Catalyst 9800 における WLAN と Policy Profile の分離ミス**:
  * 9800 では `WLAN Profile`（SSID / L2 セキュリティ設定）と `Policy Profile`（VLAN / L3 セキュリティ / AAA Override / Switching Mode）を分離し、`Policy Tag` で結合します。Policy Profile 側で `Status` を有効化（`no shutdown`）し忘れるトラブルが頻発します。
* **802.11r (FT) による非対応端末の接続拒否**:
  * `ft` を pure 状態（非 adaptive）で強制すると、802.11r を解釈できない旧型端末や一部の IoT 端末が Assoc 段階で拒否されます。ラボ要件で「Legacy 端末の接続を維持しつつ FT 対応端末に高速ローミングを提供せよ」と指定された場合は `ft adaptive` を設定します。

---

## 🛠 設定方法

### 1. Catalyst 9800 WLC (IOS-XE) での WPA2-Enterprise 設定 (CLI)

```bash
# 1. RADIUS サーバーおよびグループの定義
radius server ISE-PRI
 address ipv4 192.168.10.50 auth-port 1812 acct-port 1813
 key 7 Ciscokey123!

aaa group server radius RADIUS-ISE-GROUP
 server name ISE-PRI
 ip radius source-interface GigabitEthernet1

# 2. AAA 認証・認可リストの定義
aaa authentication login default local
aaa authentication dot1x ISE-8021X-AUTH group RADIUS-ISE-GROUP
aaa authorization network ISE-8021X-AUTH group RADIUS-ISE-GROUP
aaa accounting network ISE-8021X-ACCT start-stop group RADIUS-ISE-GROUP

# 3. WLAN プロファイルの作成 (WPA2-Enterprise + 802.11r FT)
wlan WLAN-CORP-ENTERPRISE 10 CORP-SSID
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security wpa akm dot1x
 security wpa akm ft dot1x
 pmf optional
 no shutdown

# 4. Policy Profile の作成 (AAA Override 有効化)
wireless profile policy POLICY-CORP-ENTERPRISE
 aaa-override
 accounting-list ISE-8021X-ACCT
 vlan VLAN-DATA-100
 no shutdown

# 5. Policy Tag による WLAN と Policy Profile のバインド
wireless tag policy TAG-CORP-POLICY
 wlan WLAN-CORP-ENTERPRISE policy POLICY-CORP-ENTERPRISE

# 6. AP または AP Group への Tag 割り当て
ap profile AP-PROF-SITE1
 policy-tag TAG-CORP-POLICY
```

### 2. Catalyst 9800 WLC (IOS-XE) での WPA2-Personal (PSK) 設定 (CLI)

```bash
wlan WLAN-GUEST-PSK 20 GUEST-SSID
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security wpa akm psk
 security wpa ikm psk set-key ascii 0 SecretPskPass123!
 pmf optional
 no shutdown

wireless profile policy POLICY-GUEST-PSK
 vlan VLAN-GUEST-200
 no shutdown

wireless tag policy TAG-GUEST-POLICY
 wlan WLAN-GUEST-PSK policy POLICY-GUEST-PSK
```

### 3. AireOS WLC での WPA2-Enterprise 設定 (CLI)

```bash
# RADIUS サーバーの設定
config radius auth add 1 192.168.10.50 1812 ascii Ciscokey123!
config radius acct add 1 192.168.10.50 1813 ascii Ciscokey123!

# WLAN の作成と Layer 2 セキュリティ設定
config wlan create 10 CORP-WLAN CORP-SSID
config wlan security wpa enable 10
config wlan security wpa wpa1 disable 10
config wlan security wpa wpa2 enable 10
config wlan security wpa wpa2 ciphers aes enable 10
config wlan security wpa wpa2 ciphers tkip disable 10
config wlan security wpa wpa2 akm dot1x enable 10

# 802.11r (FT) および PMF の設定
config wlan ft enable 10
config wlan ft reassociation-timeout 10
config wlan pmf optional 10

# AAA サーバーの紐付けと AAA Override の有効化
config wlan radius_server auth add 10 1
config wlan radius_server acct add 10 1
config wlan aaa-override enable 10
config wlan enable 10
```


## 🔍 検証コマンド

WPA2 接続、認証、高速ローミングの動作確認およびトラブルシューティングに使用する主要コマンド一覧です。

| 目的 | コマンド (Catalyst 9800 / IOS-XE) | コマンド (AireOS) |
| :--- | :--- | :--- |
| **WLAN 設定確認** | `show wlan name <wlan-name>` | `show wlan <wlan-id>` |
| **接続クライアント一覧** | `show wireless client summary` | `show client summary` |
| **クライアント詳細情報 (キー交換/PMF/FT)** | `show wireless client mac-address <MAC> detail` | `show client detail <MAC>` |
| **802.1X / EAP 統計確認** | `show aaa servers` / `show dot1x summary` | `show radius statistics` |
| **高速ローミング (FT/OKC) キャッシュ確認** | `show wireless client mac-address <MAC> roaming` | `show client detail <MAC>` (Roam History) |
| **リアルタイムトレースデバッグ (指定MAC)** | `debug wireless mac <MAC>`<br>`show logging profile wireless filter mac <MAC> internal` | `debug client <MAC>` |
| **Dot1x / EAP パケットデバッグ** | `debug dot1x all` / `debug eapol all` | `debug dot1x events enable` |

---

## 🚨 トラブルシュート

WPA2 環境でよく発生するトラブルシューティング項目、原因、確認方法、対策を表形式で整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **クライアントが 4-Way Handshake Message 2/4 でタイムアウトし接続不可** | 事前共有鍵 (PSK) の不一致、または EAP 認証成功後の PMK 保持不一致 | `show wireless client mac-address <MAC> detail`<br>`debug wireless mac <MAC>` | PSK パスフレーズを正しく再設定。RADIUS サーバー (ISE) の MSK/PMK 送信属性（`MS-MPPE-Recv-Key` / `Send-Key`）の設定確認。 |
| **802.11r FT 非対応端末が SSID に接続できない（Assoc Reject）** | WLAN で pure 802.11r FT が強制設定されており、Over-the-Air IE を正しく処理できない | `show wlan name <wlan-name>` | `ft adaptive` を設定するか、非 FT 用の別 SSID を用意。 |
| **AAA Override による Dynamic VLAN 割り当てが反映されない** | Policy Profile で `aaa-override` が未有効化、または ISE 送信 VLAN 属性名/ID が WLC に存在しない | `show wireless profile policy detailed <policy-name>` | Policy Profile 側で `aaa-override` を有効化。WLC 上に指示された VLAN ID/Name をあらかじめ作成。 |
| **無線クライアントの通信速度が 54Mbps 以上に上がらない** | L2 暗号化方式として TKIP または WEP が有効化されている | `show wlan name <wlan-name>` | 暗号化方式を `AES (CCMP128)` のみに変更し、TKIP を無効化。 |
| **4-Way Handshake 完了後に IP アドレスが取得できない (DHCP Failure)** | 802.11 接続完了後の VLAN 設定不備、または DHCP Snooping / ARP 制御の制限 | `show wireless client mac-address <MAC> detail` (L3 State) | WLC-Switch 間の Trunk ポートにおける VLAN 許可、Policy Profile の VLAN 割り当てを確認。 |

---

## ⚠ 制限事項

* **TKIP 暗号の非推奨・廃止**: WPA2 では AES-CCMP が標準です。TKIP を使用すると 802.11n / 802.11ac / 802.11ax の高速度拡張（HT/VHT/HE データレート）が規格上禁止され、通信速度が最大 54Mbps に制限されます。
* **管理フレームの非保護 (802.11w PMF 未使用時)**: 802.11i WPA2 の基本規格ではデータフレームのみが暗号化されます。802.11w (PMF) が配置されていない場合、攻撃者からの偽造 Deauth / Disassoc パケットによる DoS 攻撃を防止できません。
* **PSK モードの辞書攻撃リスク**: WPA2-PSK は 4-Way Handshake パケット（ANonce, SNonce, EAPOL-Key MIC）をキャプチャされると、オフラインで Hashcat や Aircrack-ng 等を用いた辞書攻撃 / 総当たり攻撃を受け、パスフレーズが解読される脆弱性があります。

---

## 🔄 他技術との関連

* **Cisco ISE (Identity Services Engine)**:
  * WPA2-Enterprise の EAP 認証（EAP-TLS, PEAP）を担当。認証完了後、RADIUS 送信属性（`Tunnel-Type=VLAN`, `Tunnel-Private-Group-ID=VLAN_ID`, `cisco-av-pair = option:sgt=SGT_TAG`）により、WLC 上のクライアント通信ポリシーを動的制御。
* **802.11r (Fast BSS Transition - FT)**:
  * AP 間のハンドオーバー時に、初回接続時に導出した PMK を元に事前鍵交換を行い、ローミング時の 4-Way Handshake 遅延を 50ms 未満に短縮。
* **Cisco TrustSec (SGT / SXP)**:
  * WPA2-Enterprise 認証成功時に ISE からクライアントへ Security Group Tag (SGT) を割り当て。WLC は CAPWAP トンネル経由でスイッチへ SGT を伝播させ、ネットワーク全体でグループベースのアクセス制御（SGACL）を適用。

---

## 🧩 比較表

### 1. WPA / WPA2 / WPA3 の機能比較

| 項目 | WPA (IEEE 802.11i draft) | WPA2 (IEEE 802.11i) | WPA3 (Wi-Fi Alliance) |
| :--- | :--- | :--- | :--- |
| **標準暗号化アルゴリズム** | RC4 (TKIP) | **AES (CCMP128)** | AES-GCM (128-bit / 256-bit) |
| **Personal 認証** | PSK (PBKDF2) | **PSK (PBKDF2)** | SAE (Dragonfly Handshake) |
| **辞書攻撃への耐性** | 低い（キャプチャ解析可能） | **低い（キャプチャ解析可能）** | 極めて高い（オフライン解析不可） |
| **管理フレーム保護 (PMF)** | 不可 | **オプション (802.11w)** | **必須要件** |
| **Enterprise 強度** | 802.1X / EAP | **802.1X / EAP (128-bit)** | 192-bit CNSA Suite / 128-bit |

### 2. 高速ローミング方式の比較 (WPA2)

| 方式 | 標準規格 | メカニズム | クライアント要件 | 特徴・注意点 |
| :--- | :--- | :--- | :--- | :--- |
| **PMK Caching** | IEEE 802.11i | 接続済み AP の PMKID を保持し再利用 | 個別 AP ごとに PMKID 保持 | 過去に接続した AP へのローミング時のみ高速化 |
| **OKC (Opportunistic Key Caching)** | Cisco 独自拡張 | WLC 内の全 AP 間で 1 つの PMK をキャッシュ共有 | OKC 対応クライアント | 単一 WLC 配下の全 AP 間で高速ローミング可能 |
| **802.11r FT** | IEEE 802.11r | BSS 間移動時に事前鍵交換を実施 | 802.11r (FT) 対応 | 最も高速（< 50ms）。非対応端末との互換性に注意 |


## 💡 ベストプラクティス

1. **暗号方式の厳格化**: TKIP 暗号および WEP を完全に無効化し、WPA2-AES (CCMP128) のみを許可することで、無線パケット処理の HW アクセラレーションを最大化し、802.11n/ac/ax の高速データレートを維持します。
2. **高速ローミング (802.11r FT) の互換性設計**: 多種多様なクライアント（スマートデバイス、レガシー端末、医療機器など）が混在するエンタープライズ環境では、`ft adaptive` を導入するか、レガシー用 SSID と FT 専用 SSID を分ける運用を推奨します。
3. **802.11w PMF の導入**: Deauth 攻撃による DoS を防ぐため、WPA2 環境でも PMF (`optional`) を設定し、対応端末に対して管理フレーム保護を有効化します。
4. **iPSK (Identity PSK) による IoT セグメンテーション**: 個別認証が困難な PSK 端末群に対して、ISE 経由で MAC ごとの個別 PSK および Dynamic VLAN/SGT を割り当て、端末間の横方向攻撃（Lateral Movement）を防止します。

---

## 📝 ラボ学習・設定サンプル例

実際の CCIE Security ラボ試験レベルの要件を満たす 10 個の実践的な設定例と検証手順です。

### サンプル 1: Catalyst 9800 - 基本 WPA2-Enterprise (802.1X) WLAN の構築

* **問題**: Catalyst 9800 WLC において、SSID `CORP-WPA2-EAP` を作成し、WPA2-Enterprise (AES-CCMP) 認証を実装しなさい。
* **要件**:
  1. RADIUS サーバー `192.168.10.50` (Key: `CiscoKey123!`) を使用すること。
  2. WLAN は VAP ID `10` とすること。
  3. 認証・認可グループ名は `ISE-GROUP` とする。
* **設定例**:
```bash
radius server ISE-SERVER
 address ipv4 192.168.10.50 auth-port 1812 acct-port 1813
 key 7 CiscoKey123!

aaa group server radius ISE-GROUP
 server name ISE-SERVER

aaa authentication dot1x ISE-DOT1X-AUTH group ISE-GROUP

wlan CORP-WPA2-EAP 10 CORP-WPA2-EAP
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security wpa akm dot1x
 no shutdown

wireless profile policy POLICY-CORP
 vlan VLAN-DATA-100
 no shutdown

wireless tag policy TAG-CORP
 wlan CORP-WPA2-EAP policy POLICY-CORP
```
* **検証方法**:
  * `show wlan name CORP-WPA2-EAP` で WPA2/AES/Dot1x が有効であることを確認。

---

### サンプル 2: Catalyst 9800 - 802.11r Fast BSS Transition (FT) の実装

* **問題**: WLAN `CORP-WPA2-EAP` に対し、802.11r FT 機能を有効化し、高速ローミングを提供しなさい。
* **要件**: FT 非対応端末も接続できるように Adaptive FT モードを設定すること。
* **設定例**:
```bash
wlan CORP-WPA2-EAP 10 CORP-WPA2-EAP
 shutdown
 security wpa akm ft dot1x
 ft adaptive
 no shutdown
```
* **検証方法**:
  * `show wlan name CORP-WPA2-EAP` で `FT Support: Adaptive` および `FT Dot1x AKM: Enabled` を確認。

---

### サンプル 3: Catalyst 9800 - WPA2-PSK (Personal) の構築

* **問題**: IoT 端末用 SSID `IOT-PSK` を設定しなさい。
* **要件**:
  1. 鍵方式は WPA2-PSK (AES) とする。
  2. 事前共有鍵は `SecureIoTKey2026!` とする。
  3. VLAN `200` にマッピングすること。
* **設定例**:
```bash
wlan IOT-PSK 20 IOT-PSK
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security wpa akm psk
 security wpa ikm psk set-key ascii 0 SecureIoTKey2026!
 no shutdown

wireless profile policy POLICY-IOT
 vlan 200
 no shutdown

wireless tag policy TAG-IOT
 wlan IOT-PSK policy POLICY-IOT
```
* **検証方法**:
  * `show wlan name IOT-PSK` で AKM PSK および ciphers aes を確認。

---

### サンプル 4: Catalyst 9800 - WPA2 における PMF (802.11w) の有効化

* **問題**: 無線管理フレームに対する Deauthentication 攻撃を防御するための保護を設定しなさい。
* **要件**: WLAN `CORP-WPA2-EAP` に対し、PMF を Optional モードで有効化すること。
* **設定例**:
```bash
wlan CORP-WPA2-EAP 10 CORP-WPA2-EAP
 shutdown
 pmf optional
 no shutdown
```
* **検証方法**:
  * `show wlan name CORP-WPA2-EAP` の出力で `PMF Option: Optional` を確認。

---

### サンプル 5: Catalyst 9800 - AAA Override による動的 VLAN 割り当て

* **問題**: Cisco ISE からの RADIUS 応答属性に基づき、ユーザーごとに Dynamic VLAN を適用しなさい。
* **要件**: Policy Profile `POLICY-CORP` において AAA Override 機能を有効化すること。
* **設定例**:
```bash
wireless profile policy POLICY-CORP
 shutdown
 aaa-override
 no shutdown
```
* **検証方法**:
  * `show wireless profile policy detailed POLICY-CORP` で `AAA Override: Enabled` を確認。

---

### サンプル 6: AireOS WLC - WPA2-Enterprise + AES の CLI 設定

* **問題**: AireOS WLC において、WLAN ID `5` に SSID `AIRE-CORP` を作成し、WPA2-Enterprise (AES) を適用しなさい。
* **要件**: TKIP を無効化し、AES のみを許可すること。
* **設定例**:
```bash
config wlan create 5 AIRE-CORP AIRE-CORP
config wlan security wpa enable 5
config wlan security wpa wpa1 disable 5
config wlan security wpa wpa2 enable 5
config wlan security wpa wpa2 ciphers aes enable 5
config wlan security wpa wpa2 ciphers tkip disable 5
config wlan security wpa wpa2 akm dot1x enable 5
config wlan enable 5
```
* **検証方法**:
  * `show wlan 5` で `WPA2 Ciphers: AES` および `WPA2 AKM: Dot1x` を確認。

---

### サンプル 7: AireOS WLC - OKC (Opportunistic Key Caching) の有効化

* **問題**: AireOS WLAN 5 において、WLC 内 AP 間の再認証を高速化する OKC を有効化しなさい。
* **要件**: OKC 機能を有効化し、再認証不要のローミングを実現すること。
* **設定例**:
```bash
config wlan security wpa wpa2 okc enable 5
```
* **検証方法**:
  * `show wlan 5` で `Opportunistic Key Caching: Enabled` を確認。

---

### サンプル 8: Catalyst 9800 - Identity PSK (iPSK) の動的認証設定

* **問題**: 同一 PSK 用 SSID `IPSK-SSID` で、ISE 認証により個別 PSK を割り当てる iPSK を構築しなさい。
* **要件**:
  1. AKM に PSK を指定すること。
  2. MAC 認証 (MAB) を RADIUS グループ `ISE-GROUP` 経由で有効化すること。
* **設定例**:
```bash
wlan IPSK-SSID 30 IPSK-SSID
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security wpa akm psk
 mac-filtering ISE-GROUP
 no shutdown

wireless profile policy POLICY-IPSK
 mac-filtering ISE-GROUP
 aaa-override
 no shutdown
```
* **検証方法**:
  * クライアント接続時、ISE ログで `cisco-av-pair = psk-mode=ascii` / `psk=...` が正しく送出されているか確認。

---

### サンプル 9: Catalyst 9800 - WPA2/WPA3 Transition Mode の構築

* **問題**: 既存の WPA2 クライアントと新規 WPA3 クライアントが共存できる SSID `TRANSITION-SSID` を作成しなさい。
* **要件**:
  1. WPA2-PSK (AES) と WPA3-SAE を同一 WLAN で同時にサポートすること。
  2. PMF を Optional に設定すること。
* **設定例**:
```bash
wlan TRANSITION-SSID 40 TRANSITION-SSID
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security wpa akm psk
 security wpa akm sae
 security wpa ikm psk set-key ascii 0 SecretPass123!
 pmf optional
 no shutdown
```
* **検証方法**:
  * `show wlan name TRANSITION-SSID` で AKM に `PSK` と `SAE` の両方が表示されることを確認。

---

### サンプル 10: 接続トラブルにおけるクライアントキャプチャとデバッグの実行

* **問題**: 特定のクライアント (MAC: `a4c3.f012.3456`) が WPA2 接続中に 4-Way Handshake で失敗する原因を特定しなさい。
* **要件**: Catalyst 9800 の Trace-on-Failure または MAC 条件付きデバッグログを生成・出力すること。
* **設定例**:
```bash
# MAC アドレス制限付きデバッグの有効化
debug wireless mac a4c3.f012.3456

# ログバッファのクリアと再現後の確認
clear logging
show logging profile wireless filter mac a4c3.f012.3456 internal
```
* **検証方法**:
  * ログ内で `State: M2 -> M3` 送信後の `EAPOL key M2 MIC mismatch` や `Timeout waiting for M2/M4` エラーを確認。


## ❓ 想定試験問題

CCIE Security ラボ試験を想定した実践的なオリジナル問題集です。

### 問題 1 (コンフィグ読解)
以下の Catalyst 9800 WLC コンフィグを読み取り、この WLAN のセキュリティ仕様および接続可能なクライアントの条件を説明しなさい。

```bash
wlan CORP-WIFI 15 CORP-WIFI
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security wpa akm ft dot1x
 pmf required
 no shutdown
```

* **解答・解説**:
  * **セキュリティ仕様**: WPA2-Enterprise (AES-CCMP) 認証。AKM として 802.11r FT-Dot1x のみが有効化されています。また PMF (802.11w) が required に設定されています。
  * **接続条件**: 接続クライアントは IEEE 802.11r FT および IEEE 802.11w PMF の両方に対応している必要があります。標準の 802.1X 非 FT 端末や PMF 非対応端末は Association 段階で拒否されます。

---

### 問題 2 (トラブルシューティング)
WPA2-Enterprise を設定した WLAN において、認証完了後に無線クライアントが適切な Dynamic VLAN (VLAN 105) に割り当てられず、デフォルトの VLAN 10 に接続されてしまいます。WLC の Policy Profile 設定および ISE の RADIUS 応答属性の観点から考えるべき原因を 2 つ挙げ、対処法を説明しなさい。

* **解答・解説**:
  * **原因 1**: Catalyst 9800 WLC の Wireless Profile Policy 側で aaa-override 設定が漏れている。
    * **対処**: `wireless profile policy <profile-name>` モードで `aaa-override` を有効化する。
  * **原因 2**: ISE から送出されている RADIUS 属性の VLAN 名/ID が WLC 上に存在しないか、またはフォーマットエラー。
    * **対処**: WLC 上で `show vlan brief` を実行し、ISE から送られる VLAN 105 (または VLAN 名) が事前定義されているか確認し、一致させる。

---

### 問題 3 (Design)
病院施設において、高精度の音声通話 (VoIP) を行う医療用無線 IP 電話端末（802.11r 対応）と、旧型の患者用端末（802.11r 非対応）を単一の SSID で同時にサポートする必要があります。ローミング遅延を最小化しつつ旧型端末の接続性を維持するための最適な設計を説明しなさい。

* **解答・解説**:
  * 単一の WPA2 SSID 上で `ft adaptive` を設定します。
  * `ft adaptive` を設定すると、802.11r FT 対応端末に対しては RSN IE / Fast BSS Transition IE を広告して 50ms 未満の FT 高速ローミングを提供し、FT 非対応端末に対しては通常の 802.1X 認証および 4-Way Handshake での接続を許可します。

---

### 問題 4 (実装・認証フロー)
WPA2-Enterprise 接続において、Supplicant と Authenticator 間の 4-Way Handshake で使用される 4 つの鍵（KCK, KEK, TK, GTK）のうち、データの暗号化に直接使用される鍵と、メッセージ整合性チェック (MIC) の計算に使用される鍵をそれぞれ名称で答えなさい。

* **解答・解説**:
  * **データ暗号化に使用される鍵**: **TK (Temporal Key)**（128-bit。AES-CCMP によるユニキャストデータフレームの暗号化に使用）。
  * **MIC 計算に使用される鍵**: **KCK (Key Confirmation Key)**（4-Way Handshake フレームの改ざん検知に使用）。

---

### 問題 5 (トラブルシューティング)
WPA2-Personal (PSK) 環境において、特定のクライアントがパスフレーズ入力後に接続を開始したものの、WLC 側で「EAPOL-Key M2 Timeout」ログが出力されて接続が切断されました。考えられる技術的原因を説明しなさい。

* **解答・解説**:
  * **原因**: クライアントに入力された事前共有鍵 (PSK) のパスフレーズが不一致である場合、クライアントは正しく PTK を生成できず、Message 1 の ANonce から正常な MIC を含む Message 2 を返答できません。結果として Authenticator (WLC/AP) は Message 2 を受信できず、タイムアウトが発生します。

---

## 🔗 参考リソース

Cisco 公式の技術情報およびドキュメント一覧です。

* Cisco Catalyst 9800 Series Wireless Controller Software Configuration Guide
* Cisco Wave 2 APs and Catalyst 9100 Series APs 802.11r Deployment Guide
* Wi-Fi Protected Access (WPA/WPA2/WPA3) Security Deployment Guide
* Cisco Technical Note: Troubleshoot 802.1X Client Authentication on Catalyst 9800 WLC
* Cisco Live Presentation: Advanced Wireless Security Architecture (BRKEWK-3001)

---

## 📝 補足（Notes）

* **WPA2 の強固な基盤**: WPA2 は登場から長年経過しているものの、AES-CCMP による安定した暗号化機能と IEEE 802.1X 統合機能により、現代のエンタープライズ無線ネットワークの基幹技術であり続けています。
* **WPA3 への移行**: WPA3 が導入された現在でも、Transition Mode や 802.11r FT、802.11w PMF などの相互運用技術の深い理解が CCIE Security ラボ試験で非常に強く求められます。


