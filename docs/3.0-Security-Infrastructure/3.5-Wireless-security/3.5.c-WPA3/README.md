---
layout: default
title: 3.5.c-WPA3
nav_order: 3
parent: 3.5-Wireless-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.5.c WPA3

## 📘 概要

WPA3（Wi-Fi Protected Access 3）は、Wi-Fi Allianceによって2018年に策定された次世代の無線LANセキュリティ標準規格である。前世代のWPA2で指摘された脆弱性（特に4-Way Handshakeに対するKRACKs攻撃や、辞書攻撃・オフラインパケット解析によるPSK解読）を根本的に克服し、個人利用から政府機関レベルのエンタープライズ保護までをカバーする高度な暗号化および認証メカニズムを提供する。

### 利用目的
* **オフライン辞書攻撃および総当たり攻撃の防御**: 辞書攻撃に強いSAE（Simultaneous Authentication of Equals）ハンドシェイクを導入し、推測しやすいパスフレーズ（Pre-Shared Key）が使用されている場合でもパスワード推測を防止する。
* **前方秘匿性（Forward Secrecy）の確保**: 万一現在のパスフレーズが外部に漏洩した場合であっても、過去にキャプチャされた暗号化トラフィックを後から解読することを不可能な構造とする。
* **オープンネットワークの暗号化（OWE: Enhanced Open）**: 従来パスワードなしで平文通信されていたパブリックWi-Fi等において、ユーザ認証なしでエンドツーエンドのパケット暗号化を実現する。
* **強力な管理フレーム保護（PMF: Protected Management Frames）の必須化**: IEEE 802.11w規格に基づくPMFを必須化し、Deauthentication（切断）攻撃やDisassociation攻撃などのDoS攻撃を排除する。
* **政府・金融機関レベルの超高強度セキュリティ（WPA3-Enterprise 192-bit Mode）**: 米国NSAのCNSA（Commercial National Security Algorithm）スイートに準拠した256ビット暗号化および384ビット楕円曲線暗号をサポートし、極めて高い機密性を実現する。

### どのような場面で利用するか
* **企業・組織の社内無線LAN（WPA3-Enterprise / 192-bit Mode）**: Cisco ISE（Identity Services Engine）や外部RADIUS/PKIと連携し、EAP-TLSを用いた証明書ベース認証およびAES-256-GCM / GCMP-256による高強度データ保護を提供する場面。
* **ゲスト・パブリックアクセスネットワーク（Enhanced Open / OWE）**: 認証手続きを行わないオープンSSIDにおいて、盗聴被害を防止しつつ安全にゲストトラフィックを暗号化する場面。
* **拠点のIoT・スマートデバイスおよびPSKアクセス（WPA3-Personal / SAE）**: 共有パスフレーズ運用でありながら、盗聴や辞書攻撃のリスクを極小化したい環境。
* **WPA2端末とWPA3端末が混在する移行期環境（WPA3 Transition Mode）**: 既存のWPA2端末の接続性を維持しつつ、順次WPA3へのスムーズな移行を達成する場面。

---

## 🔑 要点

WPA3アーキテクチャの主要コンポーネント、用途、メリット、デメリット、対応機種、制限事項および設計上の注意点を以下の表に整理する。

| 項目 | 内容 |
| --- | --- |
| **主要特徴** | SAE（Dragonflyハンドシェイク）、OWE（DH鍵交換による無認証暗号化）、802.11w PMF必須（Mandatory）、WPA3-Enterprise 192-bit Mode（GCMP-256 / CNSA準拠） |
| **用途** | 企業内セキュア無線LAN（802.1X/EAP）、店舗・ホテル・ゲストWi-Fi（OWE）、IoT・拠点PSK接続（SAE）、政府・軍用超高機密ネットワーク |
| **メリット** | オフライン辞書攻撃への耐性、前方秘匿性（Forward Secrecy）による過去通信解読の阻止、Deauth DoS攻撃の防御、オープンWi-Fiでの暗号化保護 |
| **デメリット** | レガシー端末（WPA2以前のみ対応）との互換性欠如、Transition Mode利用時における一部クライアントの接続トラブル、WPA3-Enterprise 192-bit Modeでの暗号化計算負荷 |
| **対応機種** | Cisco Catalyst 9800 シリーズ WLC（IOS-XE 16.12.1以降推奨）、Cisco AireOS WLC（8.10以降）、Cisco Wave 2 & Wi-Fi 6/6E/7 Access Points（Catalyst 9100シリーズ等） |
| **制限事項** | TKIPおよびWEP暗号の完全撤廃（CCMP/GCMPのみ）、Transition ModeでのSAE-PK制限、一部レガシー端末におけるTransition Mode SSID発見不能・接続不能問題 |
| **設計上の注意点** | PMF（802.11w）のMandatory指定に伴う非対応端末の接続不可、Transition Mode使用時におけるWPA2/WPA3両用のAKM設定、OWE Transition Modeでの2SSID（Hidden OWE SSID + Open SSID）設計 |

---

## 🏗 動作原理

WPA3は「WPA3-Personal (SAE)」「WPA3-Enterprise」「Enhanced Open (OWE)」の3つの主要プロファイルで構成される。それぞれの通信フローと内部メカニズムを解説する。

### 1. WPA3-Personal (SAE: Simultaneous Authentication of Equals)

WPA3-Personalでは、従来のWPA2-PSKで用いられていた4-Way Handshake前のPre-Shared Key直接計算を排除し、Diffie-Hellman鍵交換に基づく**SAE（Dragonfly Key Exchange / RFC 7664）**をオープン認証（Authentication Phase）時に実行する。

#### 通信フロー（SAE + 4-Way Handshake）

```text
Client (Station)                                          AP (Catalyst 9100 / WLC)
   │                                                               │
   │ ─── 1. 802.11 Probe Request / Response ─────────────────────► │
   │                                                               │
   │ ─── 2. SAE Commit Request (Scalar & Element) ───────────────► │
   │ ◄── 3. SAE Commit Response (Scalar & Element) ─────────────── │
   │                                                               │  [の両者が共通の暗号鍵
   │ ─── 4. SAE Confirm Request (Send-Confirm Counter) ──────────► │   PMK (Pairwise Master Key)
   │ ◄── 5. SAE Confirm Response (Send-Confirm Counter) ────────── │   を相互に生成・検証]
   │                                                               │
   │ ─── 6. 802.11 Association Request (WPA3 RSN IE / PMF) ──────► │
   │ ◄── 7. 802.11 Association Response ────────────────────────── │
   │                                                               │
   │ ◄── 8. EAPOL-Key Frame 1 (ANonce) ─────────────────────────── │  [PMKからPTK/GTKを派生]
   │ ─── 9. EAPOL-Key Frame 2 (SNonce + MIC) ────────────────────► │
   │ ◄── 10. EAPOL-Key Frame 3 (Install PTK + Encrypted GTK) ──── │
   │ ─── 11. EAPOL-Key Frame 4 (ACK) ────────────────────────────► │
   │                                                               │
   │ ◄══ 12. Secure Encrypted Data Traffic (CCMP/GCMP) ══════════► │
```

* **Commitフェーズ**: 共有パスフレーズ（Password）と両者のMACアドレスから生成された楕円曲線上の点（PWE: Password Element）に基づき、スカラー値（Scalar）と有限体/楕円曲線要素（Element）を計算して送信・交換する。
* **Confirmフェーズ**: 交換したCommit値から互いに共通の暗号鍵（PMK: Pairwise Master Key）が正しく計算されたかを、MAC値（Send-Confirm）の照合によって検証する。この時点でAPとクライアントは通信経路上の盗聴者に秘密鍵を露呈することなく同一のPMKを共有する。

### 2. Enhanced Open (OWE: Opportunistic Wireless Encryption / RFC 8110)

OWEは、事前共有鍵やパスワード認証、ユーザーログインを必要としないパブリックWi-Fiにおいて、DH鍵交換（ECDH: Elliptic Curve Diffie-Hellman）を802.11 Association手続き中に透過的に埋め込むことで、盗聴不可能な暗号化セッションを確率する。

#### OWE Transition Mode の構造
既存のオープンアクセスSSIDとの互換性を保つため、APは**2つのSSID**を発行する。
1. **Open SSID (Broadcast)**: 従来のレガシー端末が見えるブロードキャストSSID。OWE Vendor Specific IE（OWE BSSIDおよびOWE SSIDへのポインタ）を保持。
2. **OWE SSID (Hidden)**: OWE対応端末専用の非公開SSID。OWE AKM（00-0F-AC:18）を広報。

```text
Client (OWE Capable)                                     AP (Catalyst 9800 WLC)
   │                                                               │
   │ ─── 1. Probe Req (Open SSID) ───────────────────────────────► │
   │ ◄── 2. Probe Resp (Open SSID with OWE Transition IE) ──────── │  [OWE Hidden SSIDの情報を取得]
   │                                                               │
   │ ─── 3. Probe Req / Assoc Req (OWE Hidden SSID + DH Element)─► │  [DH Group 19 (ECDH) キー送出]
   │ ◄── 4. Assoc Resp (OWE Hidden SSID + DH Element) ──────────── │  [APのDHキー返答 ➔ PMK共有]
   │                                                               │
   │ ◄── 5. 4-Way Handshake (EAPOL-Key Frame 1-4) ────────────────► │  [PTK/GTK生成]
   │                                                               │
   │ ◄══ 6. Encrypted Data Pass (CCMP-128 / GCMP-256) ═══════════► │
```

---

## ⚙ 動作シーケンス

WPA3における接続確立からデータ通信保護までの詳細な処理順序を以下に示す。

1. **ビーコン/プローブ応答の受信とAKM（Authentication and Key Management）のネゴシエーション**:
   * APは802.11 Beacon / Probe ResponseのRSN IE（Robust Security Network Information Element）内に以下のパラメータを広報する。
     * **WPA3-Personal**: AKM Suite 00-0F-AC:8 (SAE) または 00-0F-AC:24 (SAE-PK)。
     * **WPA3-Personal Transition Mode**: AKM Suite 00-0F-AC:2 (PSK) および 00-0F-AC:8 (SAE) の両方。
     * **WPA3-Enterprise 192-bit**: AKM Suite 00-0F-AC:12 (802.1X with SHA-384) または 00-0F-AC:13 (Suite B PKI)。Cipher Suite 00-0F-AC:9 (GCMP-256)。
     * **PMF Capabilities**: RSN Capabilitiesフィールドの `MFPR` (Management Frame Protection Required) = 1、`MFPC` (Capable) = 1。
2. **SAE 認証フェーズ (WPA3-Personal)**:
   * クライアントとAPは 802.11 Authentication Frame (Algorithm 3: SAE) を使用してCommitおよびConfirmフレームを交換する。
   * エフェメラルDiffie-Hellman鍵交換により、第三者にパスフレーズを推測されることなく相互に認証を行い、32バイト（またはそれ以上）の **PMK (Pairwise Master Key)** を導出する。
3. **802.11 アソシエーションフェーズ**:
   * クライアントは選択したAKMおよび暗号スイート（CCMP-128, GCMP-128/256）をAssoc Request内のRSN IEに格納してアソシエーションをリクエストする。
   * PMFが有効（Mandatory）であるため、アソシエーション完了後、すべてのユニキャスト管理フレーム（Action Frame等）およびマルチキャスト管理フレームはBIP（Broadcast Integrity Protocol: BIP-CMAC-128/256 または BIP-GMAC-128/256）によって改ざん防止・暗号化保護される。
4. **802.1X/EAP 認証フェーズ (WPA3-Enterprise)**:
   * クライアントと認証サーバ（Cisco ISE）間で EAP-TLS 認証が実行される。
   * WPA3-Enterprise 192-bit Modeの場合、TLS 1.2/1.3上でTLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 等の強度高い暗号スイートが必須となり、認証完了後にISEからWLCへPMK（Master Session Keyから導出）が送られる。
5. **4-Way Handshake**:
   * SAEまたはEAP認証で合意したPMKを基に、EAPOL-Keyフレーム1〜4を交換する。
   * ANonceおよびSNonceを交換して **PTK (Pairwise Transient Key)** を生成し、APから暗号化された **GTK (Group Temporal Key)** および **IGTK (Integrity Group Temporal Key)** をクライアントへ安全に配信する。
6. **データ通信およびPMF保護された管理通信**:
   * データフレームはPTK（CCMP/GCMP）によりカプセル化される。
   * 接続確立後のDeauth / DisassociationフレームはSAE/PTK由来のキーで署名され、外部の攻撃者による偽造Deauthパケットはスイッチ/APによって無効として即座に破棄される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験における WPA3 の出題ポイント、設定上の必須要件、よくある設定ミス、およびトラブルシューティング技法を整理する。

### Blueprintで重要なポイント
1. **WPA3-Personal (SAE) の実装とPMF要件**:
   * WPA3単体指定時は `pmf required`（または Catalyst 9800 での `pmf-state mandatory`）が必須。これを `optional` や `disabled` に設定するとWPA3 RSN IEの整合性が崩れ、アソシエーションエラーとなる。
2. **WPA3-Personal Transition Mode の設計**:
   * 既存WPA2端末とWPA3端末を同一SSIDで混在させる設定。
   * WPA2用（`psk` / `aes`）とWPA3用（`sae` / `aes`）の両方のAKMを登録し、`pmf-state` を `optional` に指定する。
3. **WPA3-Enterprise 192-bit Mode (CNSA Mode) の完全構成**:
   * EAP-TLS認証が前提。802.1X AKMに `802.1x-sha384` または `suiteb-192` を指定。
   * 暗号スイートに `gcmp256`（データ面）および `bip-gmac256`（管理面）を指定。
   * Catalyst 9800 WLCのWLAN Profile、Policy Profile、およびCisco ISEのAllowed Protocols（TLS 1.2以上、ECDHE-ECDSA、AES-256-GCM）の整合性を保つ。
4. **Enhanced Open (OWE) Transition Mode の 2-SSID 構築**:
   * OWEオープンSSID（可視）とOWE非表示SSID（Hidden）をマッピングし、WLAN IDレベルで相互にOWE Transitionパラメータを設定する。

### ラボ試験で設定させられそうな内容
* **Catalyst 9800 CLIでのWPA3-Personal Transition Modeの設定**:
  ```text
  wlan wlan-wpa3-trans 13 WPA3-TRANS
   security wpa psk set-key ascii 0 Cisco12345678!
   security wpa akm psk enable
   security wpa akm sae enable
   security wpa wpa2 ciphers aes enable
   security wpa wpa3 ciphers aes enable
   security pmf optional
   no shutdown
  ```
* **Catalyst 9800 CLIでのWPA3-Enterprise 192-bit Modeの設定**:
  ```text
  wlan wlan-wpa3-ent192 14 WPA3-ENT192
   security wpa akm 802.1x bit-192 enable
   security wpa wpa3 ciphers gcmp256 enable
   security pmf mandatory
   security identity-protection
   no shutdown
  ```

### よくある設定ミス
* **WPA3-Personal単体（Pure WPA3）で `security pmf optional` と設定してしまう**:
  * Pure WPA3（SAEのみ）ではPMFは `mandatory` である必要がある。`optional` にするとクライアント側でアソシエーションが拒否される。
* **OWE Transition Modeで2つのWLAN IDに対する相互ポインタ設定漏れ**:
  * OWE SSID（Hidden）側に `owe transition-mode ssid <Open_SSID> wlan <Open_WLAN_ID>` を設定し、Open SSID側に `owe transition-mode ssid <OWE_SSID> wlan <OWE_WLAN_ID>` を設定する相互関連付けを忘れると、OWE対応端末がOWEに昇格できない。
* **WPA3-Enterprise 192-bit ModeでEAP暗号化強度（ISE側）とのミスマッチ**:
  * WLC側で `bit-192` (SHA-384) と `gcmp256` を設定していても、Cisco ISEのAllowed Protocols設定で「EAP-TLS: Allow SHA-384 / RSA-4096 or ECDSA P-384」が許可されていない場合、EAP認証がPhase 2で拒否される。

### showコマンドおよびdebugログによるトラブルシューティング
* **`show wlan name <SSID>`**: WLAN設定（AKM、Ciphers、PMFステータス）の確認。
* **`show wireless client mac-address <MAC> detail`**: クライアントの接続状態、交渉されたAKM（SAE / 802.1X-SHA384 / OWE）、暗号スイート、およびPMFステータス（Required/Enabled）の確認。
* **`debug wireless mac <MAC>` (Catalyst 9800 Trace-Parser)**:
  * クライアントアソシエーション時のRSN IE解析ログやSAE Commit/Confirmフレーム処理失敗ログ（例: `SAE commit processing failed`, `PMF Mismatch`）を確認。


## 🛠 設定方法

Catalyst 9800 WLC（IOS-XE）および Cisco ISE を用いた WPA3 各モード（WPA3-Personal SAE / Transition Mode、WPA3-Enterprise 192-bit Mode、Enhanced Open OWE）の設定手順を示す。

### 1. Catalyst 9800 WLC CLI 設定例

#### (A) WPA3-Personal (Pure SAE) 設定
```text
! WLAN Profile 作成
wlan wlan-wpa3-sae 15 WPA3-SAE-SSID
 security wpa psk set-key ascii 0 SecretPassword123!
 security wpa akm sae enable
 no security wpa akm psk
 security wpa wpa3 ciphers aes enable
 no security wpa wpa2 ciphers aes
 security pmf mandatory
 no shutdown

! Policy Profile へのバインドと有効化
wireless profile policy default-policy-profile
 service-policy input QOS-TRUST
 service-policy output QOS-TRUST
 vlan VLAN_WIRELESS_10
 no shutdown

wireless tag policy default-policy-tag
 wlan wlan-wpa3-sae policy default-policy-profile
```

#### (B) WPA3-Personal Transition Mode 設定
```text
wlan wlan-wpa3-trans 16 WPA3-TRANSITION
 security wpa psk set-key ascii 0 SharedSecretPass123!
 security wpa akm psk enable
 security wpa akm sae enable
 security wpa wpa2 ciphers aes enable
 security wpa wpa3 ciphers aes enable
 security pmf optional
 no shutdown
```

#### (C) WPA3-Enterprise 192-bit Mode (CNSA / Suite B) 設定
```text
! AAA 認証・認可サーバーグループ設定
aaa group server radius RADIUS-ISE-GROUP
 server name ISE-PRIMARY
 server name ISE-SECONDARY

aaa authentication dot1x ISE-8021X-AUTH group RADIUS-ISE-GROUP

! WPA3-Enterprise 192-bit WLAN Profile
wlan wlan-wpa3-ent192 17 WPA3-ENT-192BIT
 security wpa akm 802.1x bit-192 enable
 no security wpa akm dot1x
 security wpa wpa3 ciphers gcmp256 enable
 no security wpa wpa2 ciphers aes
 security pmf mandatory
 security wpa wpa3
 authentication-list ISE-8021X-AUTH
 no shutdown
```

#### (D) Enhanced Open (OWE) Transition Mode 設定
```text
! 1. OWE 非表示SSID (Hidden OWE SSID) の作成
wlan wlan-owe-hidden 18 OWE-HIDDEN
 owe transition-mode ssid OWE-GUEST-OPEN wlan 19
 security owe
 security pmf mandatory
 security wpa wpa3 ciphers aes enable
 hide-ssid
 no shutdown

! 2. OWE オープンSSID (Visible Open SSID) の作成
wlan wlan-owe-open 19 OWE-GUEST-OPEN
 owe transition-mode ssid OWE-HIDDEN wlan 18
 no security wpa
 no shutdown
```

---

### 2. Catalyst 9800 WLC GUI 設定手順
1. **WPA3-Personal / Transition Mode (GUI)**:
   * **Configuration > Wireless > WLANs** に移動し **+ Add** をクリック。
   * **General Tab**: Profile Name と SSID 名を入力。
   * **Security Tab > Layer2**:
     * **Authentication**: PSK（または不要）。
     * **WPA3 / WPA2-AES**: Checkbox をオン。
     * **Auth Key Management (AKM)**: WPA3-Personal の場合は `SAE` をチェック。Transition Mode の場合は `PSK` と `SAE` の両方をチェック。
     * **Protected Management Frame (PMF)**: Pure WPA3 は `Mandatory`、Transition Mode は `Optional` を選択。
2. **WPA3-Enterprise 192-bit Mode (GUI)**:
   * **Security Tab > Layer2**:
     * **Security Mode**: WPA2 + WPA3（または WPA3 Enterprise）。
     * **AKM**: `192-bit 802.1X` をチェック。
     * **Encryption Cipher**: `GCMP256`（Data Engine）および `BIP-GMAC256`（Management）を有効化。
     * **PMF**: `Mandatory` に設定。

---

### 3. Cisco ISE (Identity Services Engine) 統合設定

WPA3-Enterprise 192-bit Mode 運用における Cisco ISE の推奨ポリシー設定:

1. **Allowed Protocols Policy**:
   * **Policy > Policy Elements > Results > Authentication > Allowed Protocols** に移動。
   * **Allow EAP-TLS**: チェックをオン。
   * **TLS Version**: `TLS 1.2` および `TLS 1.3` のみを許可（TLS 1.0/1.1 は無効化）。
   * **Cipher Suites**: CNSA 規格に準拠した強度の高い暗号スイートを優先指定。
     * `TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384`
     * `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`
     * `TLS_DHE_RSA_WITH_AES_256_GCM_SHA384`
2. **Authorization Policy**:
   * **Policy > Authorization Policies**:
     * **Condition**: `Radius:Called-Station-ID ENDS_WITH <WPA3_SSID_Name>` AND `Network Access:AuthenticationMethod EQUALS EAP-TLS` AND `CERTIFICATE:KeySize GE 384`.
     * **Permissions**: `PermitAccess`（必要に応じて DACL や SGT タグを割り当て）。

---

## 🔍 検証コマンド

WPA3 接続状態、AKM、暗号化スイート、PMF ネゴシエーションの確認用コマンドを以下の表にまとめる。

| 目的 | コマンド |
| --- | --- |
| **WLAN設定全体確認** | <code>show wlan name &lt;SSID_NAME&gt;</code> |
| **WPA3クライアント詳細ステータス** | <code>show wireless client mac-address &lt;MAC&gt; detail</code> |
| **WPA3/PMF統計情報** | <code>show wireless client summary</code> |
| **AP別PMF/RSN機能確認** | <code>show ap name &lt;AP_NAME&gt; config general</code> |
| **WPA3 AKMネゴシエーション動作追跡** | <code>show logging profile wireless filter mac &lt;MAC&gt;</code> |
| **クライアントアソシエーションリアルタイムデバッグ** | <code>debug wireless mac &lt;MAC&gt;</code><br><code>set platform software trace wireless chassis active R0 client mac &lt;MAC&gt; debug</code> |
| **EAPOL/SAEハンドシェイク追跡** | <code>show trace connection wireless mac &lt;MAC&gt;</code> |

---

## 🚨 トラブルシュート

WPA3 導入時に発生しやすいトラブルの原因、確認コマンド、および対処方法を以下の表にまとめる。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **WPA3端末がアソシエーション拒否される（Auth/Assoc Fail）** | PMF（802.11w）の設定不整合。Pure WPA3に `pmf optional` が設定されているか、端末がPMF非対応。 | <code>show wlan name &lt;SSID&gt;</code><br><code>show wireless client mac &lt;MAC&gt; detail</code> | Pure WPA3では `pmf mandatory` を設定。PMF非対応端末が存在する場合は Transition Mode に変更し `pmf optional` とする。 |
| **Transition Mode で WPA2 端末が SSID を検出できない / 接続失敗** | 一部レガシー端末のドライバが RSN IE 内の複数 AKM（PSK+SAE）解析に対応していない。 | <code>show logging profile wireless</code><br>（クライアント側ドライバログ） | クライアントのWi-Fiドライバを最新に更新。更新不能な端末群は別専用のWPA2-PSK SSID（VLAN分離）へ隔離収容する。 |
| **WPA3-Enterprise 192-bit Mode で EAP 認証失敗** | Cisco ISE側の Allowed Protocols で SHA-384 / ECDSA P-384 または GCMP-256 / TLS 1.2 が許可されていない。 | ISE **Operations > RADIUS > Live Logs**<br><code>show aaa servers</code> | Cisco ISE の Allowed Protocols で EAP-TLS の強固な Cipher Suite（SHA-384）および TLS 1.2/1.3 を有効化する。 |
| **OWE Transition Mode で端末が OWE に昇格せず Open 接続になる** | Hidden OWE SSID と Open SSID 間の相互ポインタ設定（`owe transition-mode`）のミスマッチ。 | <code>show wlan name &lt;OPEN_SSID&gt;</code><br><code>show wlan name &lt;OWE_SSID&gt;</code> | 両 WLAN Profile にて、相手方の SSID 名および WLAN ID が正しく相互参照されているか確認・再設定する。 |
| **SAE Commit/Confirm 応答遅延による接続タイムアウト** | 無線環境の電波干渉によるパケットロス、または AP の CPU 高負荷による Dragonfly 楕円曲線計算の遅延。 | <code>show controllers dot11Radio &lt;INT&gt;</code><br><code>show process cpu</code> | APのチャネル設計・送信電力を最適化（RRM調整）。APファームウェアをCisco推奨バージョン（Cisco TAC Recommended IOS-XE）へアップデート。 |

---

## ⚠ 制限事項

1. **暗号方式および認証方式の制限**:
   * WPA3環境では **TKIP（Temporal Key Integrity Protocol）および WEP 暗号の利用が完全に禁止** されている。利用可能な暗号化方式は CCMP-128、GCMP-128、GCMP-256 のみ。
   * Shared Key（事前共有キー認証）による 802.11 認証アルゴリズムは不可。必ず Open System + SAE / 802.1X となる。
2. **PMF (Protected Management Frames) の強制要件**:
   * Pure WPA3（SAE / 192-bit Mode / OWE）では **PMF Mandatory が規格上必須** となる。802.11w 非対応の古い端末（IoTデバイスや旧型プリンタ等）は Pure WPA3 SSID に接続できない。
3. **Transition Mode における互換性リスク**:
   * 一部クライアントの無線NICドライバの実装問題により、RSN IE内に `SAE` と `PSK` の両方が広報されているとSSIDにアソシエーションできない現象が報告されている（Cisco Bug Search System 参照）。

---

## 🔄 他技術との関連

WPA3 とその他の主要なネットワーク・セキュリティ技術との関連性を整理する。

* **Cisco ISE (Identity Services Engine)**:
  * WPA3-Enterprise 認証における RADIUS/EAP-TLS サーバーとして動作。WPA3-192bit モードでの SHA-384 / CNSA 暗号スイートのネゴシエーションおよび RADIUS 属性（dACL, SGT, VLAN）の動的割り当てを実施。
* **Cisco TrustSec (SGT / SXP)**:
  * WPA3-Enterprise 802.1X 認証成功時、ISE から WLC / AP に対して Security Group Tag (SGT) が動的付与され、データ面でのマイクロセグメンテーションを実現。
* **PMF (IEEE 802.11w)**:
  * WPA3 のセキュリティの根幹をなす管理フレーム保護機構。Deauth / Disassociation フレームの偽造を防止し、無線ルーター・AP 偽装による DoS 攻撃を防止。
* **Fast Roaming (802.11r FT / OKC)**:
  * WPA3-Enterprise および WPA3-Personal (SAE) 環境において、FT-SAE (AKM 00-0F-AC:9) や FT-802.1X (AKM 00-0F-AC:3) を使用することで、AP 間の高速ハンドオーバー（キー再計算の高速化）を実現。


## 🧩 比較表

WPA2、WPA3-Personal、WPA3-Enterprise (128-bit/192-bit)、および Enhanced Open (OWE) の機能差を比較表に整理する。

| 比較項目 | WPA2-Personal (PSK) | WPA3-Personal (SAE) | WPA3-Enterprise (128-bit) | WPA3-Enterprise (192-bit) | Enhanced Open (OWE) |
| --- | --- | --- | --- | --- | --- |
| **主な用途** | 小規模拠点・家庭 | 個人・拠点・IoT | 企業一般アクセス | 政府・軍・金融機関 | パブリックWi-Fi・ゲスト |
| **認証方式** | Pre-Shared Key (PSK) | SAE (Dragonfly) | 802.1X / EAP (EAP-TLS等) | 802.1X / EAP (SHA-384) | 認証なし (Unauthenticated) |
| **鍵交換プロトコル** | 4-Way Handshake (PBKDF2) | ECC Diffie-Hellman (ECDH) | EAP / 4-Way Handshake | EAP-TLS / 4-Way Handshake | ECDH (Group 19) |
| **データ暗号化** | AES-CCMP-128 | AES-CCMP-128 / GCMP-256 | AES-CCMP-128 / GCMP-256 | AES-GCMP-256 (Mandatory) | AES-CCMP-128 / GCMP-256 |
| **管理フレーム保護 (PMF)** | 任意 (Optional) | 必須 (Mandatory) | 必須 (Mandatory) | 必須 (Mandatory) | 必須 (Mandatory) |
| **オフライン辞書攻撃耐性** | なし (キャプチャ後解読可能) | あり (SAEにより防御) | 該当なし (802.1X) | 該当なし (802.1X) | あり (盗聴不可能) |
| **前方秘匿性 (Forward Secrecy)** | なし | あり | あり | あり | あり |
| **移行モード (Transition Mode)** | 該当なし | サポート (WPA2/WPA3) | サポート (WPA2/WPA3) | 非対応 (Pure 192-bit) | サポート (OWE / Open 2-SSID) |

---

## 💡 ベストプラクティス

1. **移行戦略（Transition Strategy）の段階的適用**:
   * 新規構築環境では **Pure WPA3-Personal (SAE)** または **Pure WPA3-Enterprise** を第一選択とする。
   * レガシー端末が残存する既存環境では **WPA3 Transition Mode** を一時的に導入し、クライアント側のドライバ更新と平行して徐々に Pure WPA3 へ統合する。
2. **PMF (Protected Management Frames) の設計注意**:
   * Pure WPA3 SSID では PMF を `Mandatory`（必須）に指定。Transition Mode SSID では `Optional` に指定する。
3. **ゲストネットワークにおける OWE 導入**:
   * キャプティブポータルやオープンWi-Fiでは、平文の Open SSID を廃止し、OWE Transition Mode（2-SSID設計）を適用してユーザーの盗聴リスクを排除する。
4. **WPA3-Enterprise 192-bit Mode の証明書暗号強度統一**:
   * WLC側で `bit-192` および `gcmp256` を設定する場合、Cisco ISE / PKI側でも CA証明書およびサーバー証明書（ECDSA P-384 / RSA-4096）と暗号アルゴリズム（SHA-384）を統一設計する。

---

## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験レベルの実践的な設定演習例（問1〜5）を以下に示す。

### 【演習 1】Catalyst 9800 WLC における Pure WPA3-Personal (SAE) の構築

#### 問題
Catalyst 9800 WLC 上に、WPA3-Personal（SAE）専用の無線ネットワークを作成せよ。

#### 要件
1. Profile Name: `WPA3-SAE-PROF` / SSID: `CORP-WPA3-SAE`
2. 認証方式: WPA3-Personal (SAE) のみを許可。WPA2-PSK は禁止すること。
3. 暗号化: AES-CCMP-128 を使用。
4. PMF (Protected Management Frames): 必須（Mandatory）とすること。
5. パスフレーズ: `CiscoSecureSAE2026!`

#### 設定例
```text
wlan WPA3-SAE-PROF 20 CORP-WPA3-SAE
 security wpa psk set-key ascii 0 CiscoSecureSAE2026!
 security wpa akm sae enable
 no security wpa akm psk
 security wpa wpa3 ciphers aes enable
 no security wpa wpa2 ciphers aes
 security pmf mandatory
 no shutdown
```

#### 検証方法
```text
# show wlan name CORP-WPA3-SAE
! AKM Suites に SAE のみが含まれ、PMF が Required (Mandatory) になっていることを確認。
```

---

### 【演習 2】WPA3-Personal Transition Mode の設定

#### 問題
WPA2 端末と WPA3 端末が共存できる WPA3-Personal Transition Mode SSID を構築せよ。

#### 要件
1. Profile Name: `WPA3-TRANS-PROF` / SSID: `CORP-MIXED-WIFI`
2. AKM: WPA2-PSK および WPA3-SAE の両方を有効化すること。
3. 暗号化: AES (CCMP) を使用。
4. PMF: 任意（Optional）に設定すること。
5. 共有キー: `MixedPass321!`

#### 設定例
```text
wlan WPA3-TRANS-PROF 21 CORP-MIXED-WIFI
 security wpa psk set-key ascii 0 MixedPass321!
 security wpa akm psk enable
 security wpa akm sae enable
 security wpa wpa2 ciphers aes enable
 security wpa wpa3 ciphers aes enable
 security pmf optional
 no shutdown
```

#### 検証方法
```text
# show wlan name CORP-MIXED-WIFI
! AKM Suites に PSK と SAE の双方が表示され、PMF が Optional であることを確認。
```

---

### 【演習 3】WPA3-Enterprise 192-bit Mode (CNSA) の構築

#### 問題
政府機関レベルの暗号強度を満たす WPA3-Enterprise 192-bit Mode SSID を構築せよ。

#### 要件
1. Profile Name: `WPA3-192BIT-PROF` / SSID: `SECURE-GOV-192`
2. AKM: 192-bit 802.1X（`bit-192` / SHA-384）のみを許可すること。
3. データ暗号化: GCMP-256（`gcmp256`）を使用すること。
4. 管理フレーム保護: PMF Mandatory。
5. 認証サーバー: RADIUS サーバーグループ `ISE-RADIUS-GRP` を使用。

#### 設定例
```text
wlan WPA3-192BIT-PROF 22 SECURE-GOV-192
 security wpa akm 802.1x bit-192 enable
 no security wpa akm dot1x
 security wpa wpa3 ciphers gcmp256 enable
 no security wpa wpa2 ciphers aes
 security pmf mandatory
 security wpa wpa3
 authentication-list ISE-AUTH-LIST
 no shutdown
```

#### 検証方法
```text
# show wlan name SECURE-GOV-192
! AKM に 802.1X (SHA-384)、Cipher に GCMP256、PMF に Mandatory が指定されていることを確認。
```

---

### 【演習 4】Enhanced Open (OWE) Transition Mode の 2-SSID 構築

#### 問題
盗聴を防止するための Enhanced Open (OWE) ゲストWi-Fi を OWE Transition Mode（2-SSID構成）で構築せよ。

#### 要件
1. **Open SSID**:
   * Profile Name: `GUEST-OPEN-PROF` / SSID: `GUEST-WIFI`
   * 可視（Broadcast SSID）。OWE Hidden SSID へのトランジションポインタを設定すること。
2. **OWE SSID**:
   * Profile Name: `GUEST-OWE-PROF` / SSID: `GUEST-WIFI-OWE`
   * 非表示（Hidden SSID）。OWE AKM 有効化、PMF Mandatory。
3. 両 SSID 間で相互参照設定を行うこと。

#### 設定例
```text
! 1. OWE Hidden SSID の設定
wlan GUEST-OWE-PROF 23 GUEST-WIFI-OWE
 owe transition-mode ssid GUEST-WIFI wlan 24
 security owe
 security pmf mandatory
 security wpa wpa3 ciphers aes enable
 hide-ssid
 no shutdown

! 2. Open Visible SSID の設定
wlan GUEST-OPEN-PROF 24 GUEST-WIFI
 owe transition-mode ssid GUEST-WIFI-OWE wlan 23
 no security wpa
 no shutdown
```

#### 検証方法
```text
# show wlan name GUEST-WIFI
# show wlan name GUEST-WIFI-OWE
! それぞれ相手の SSID および WLAN ID が OWE Transition Mode パラメータとして表示されることを確認。
```

---

### 【演習 5】WPA3-Enterprise (128-bit Mode) と Cisco ISE 統合設定

#### 問題
標準的な WPA3-Enterprise（128-bit Mode）SSID を構築し、Cisco ISE による 802.1X 認証を設定せよ。

#### 要件
1. Profile Name: `WPA3-ENT128-PROF` / SSID: `CORP-ENTERPRISE-WPA3`
2. AKM: 標準 802.1X (Dot1x) および WPA3 802.1X を許可。
3. 暗号化: AES-CCMP-128。
4. PMF: Mandatory。
5. RADIUS リクエストには Called-Station-ID 属性を含めること。

#### 設定例
```text
wlan WPA3-ENT128-PROF 25 CORP-ENTERPRISE-WPA3
 security wpa akm dot1x enable
 security wpa wpa3 ciphers aes enable
 security pmf mandatory
 security wpa wpa3
 authentication-list ISE-AUTH-LIST
 no shutdown
```

#### 検証方法
```text
# show wireless client mac-address <MAC> detail
! Security Policy に WPA3 Enterprise、AKM に 802.1X、PMF に Required が表示されることを確認。
```


### 【演習 6】Fast BSS Transition (802.11r) と WPA3-Personal (FT-SAE) の設定

#### 問題
WPA3-Personal (SAE) 環境において、802.11r 高速ローミング（FT-SAE）を有効化せよ。

#### 要件
1. Profile Name: `WPA3-FT-SAE-PROF` / SSID: `VOICE-WPA3-SAE`
2. AKM: `sae` および `ft-sae` の両方を有効化すること。
3. 暗号化: AES-CCMP-128。
4. PMF: Mandatory。
5. パスフレーズ: `VoiceSecSAE2026!`

#### 設定例
```text
wlan WPA3-FT-SAE-PROF 26 VOICE-WPA3-SAE
 security wpa psk set-key ascii 0 VoiceSecSAE2026!
 security wpa akm sae enable
 security wpa akm ft-sae enable
 security wpa wpa3 ciphers aes enable
 security pmf mandatory
 no shutdown
```

#### 検証方法
```text
# show wlan name VOICE-WPA3-SAE
! RSN AKM Suites に SAE (00-0F-AC:8) と FT-SAE (00-0F-AC:9) の両方が含まれることを確認。
```

---

### 【演習 7】AireOS WLC CLI における WPA3-Personal (SAE) の設定

#### 問題
AireOS WLC 環境において、WPA3-Personal（SAE）WLAN を作成せよ。

#### 要件
1. WLAN ID: `10` / Profile Name & SSID: `AIREOS-WPA3-SAE`
2. AKM: SAE 有効化。
3. 暗号化: AES (CCMP128)。
4. PMF: Required。
5. PSK: `AireOSPass12345!`

#### 設定例
```text
config wlan create 10 AIREOS-WPA3-SAE AIREOS-WPA3-SAE
config wlan security wpa enable 10
config wlan security wpa wpa1 disable 10
config wlan security wpa wpa2 enable 10
config wlan security wpa wpa2 ciphers aes enable 10
config wlan security wpa akm psk disable 10
config wlan security wpa akm sae enable 10
config wlan security wpa akm sae set-key ascii 10 AireOSPass12345!
config wlan pmf required 10
config wlan enable 10
```

#### 検証方法
```text
(AireOS-WLC) > show wlan 10
! WPA2 Policy: Enabled, Ciphers: AES, AKM: SAE, Management Frame Protection: Required を確認。
```

---

### 【演習 8】PMF (802.11w) の保護設定と Deauth 攻撃遮断の検証

#### 問題
WPA3-Personal SSID において PMF を強制適用し、攻撃者からの偽造 Deauth フレームを拒否する設定を行え。

#### 要件
1. Profile Name: `PMF-PROTECT-PROF` / SSID: `PMF-TEST-SSID`
2. WPA3 SAE 有効。
3. PMF SA Query Retry Timeout: `200ms` / Comeback Timer: `10`。
4. PMF Mandatory。

#### 設定例
```text
wlan PMF-PROTECT-PROF 27 PMF-TEST-SSID
 security wpa psk set-key ascii 0 PmfTestPass2026!
 security wpa akm sae enable
 security pmf mandatory
 security pmf saquery-time 200
 security pmf comeback-time 10
 no shutdown
```

#### 検証方法
```text
# show wlan name PMF-TEST-SSID
! PMF Protection: Required, SA Query Timer 設定値を確認。
```

---

### 【演習 9】WPA3-Enterprise 192-bit モードでの EAP-TLS 認証と Cisco ISE 設定

#### 問題
Cisco ISE 認証サーバーで WPA3-Enterprise 192-bit モード接続を要求する Allowed Protocols ポリシーを構成せよ。

#### 要件
1. Policy Name: `WPA3-192BIT-ALLOWED-PROTOCOLS`
2. 許可プロトコル: EAP-TLS のみ。
3. TLS バージョン: TLS 1.2 以上を強制（TLS 1.0/1.1 拒否）。
4. 暗号スイート: ECDHE-ECDSA-AES-256-GCM-SHA384 または DHE-RSA-AES-256-GCM-SHA384 を必須指定。

#### 設定例（ISE 管理手順）
1. **Cisco ISE GUI**: **Policy > Policy Elements > Results > Authentication > Allowed Protocols** に進み **Add** をクリック。
2. Name に `WPA3-192BIT-ALLOWED-PROTOCOLS` を入力。
3. **Allow EAP-TLS** にチェックを入れる。他（PEAP, EAP-FAST等）はチェックを外す。
4. **Key Exchange**: ECDHE / DHE を指定。
5. **Encryption**: AES-256-GCM。
6. **Digest Algorithm**: SHA-384。
7. **TLS Min Version**: `TLS 1.2` に設定して保存。

#### 検証方法
ISE **Operations > RADIUS > Live Logs** にて、認証成功詳細の Authentication Protocol が EAP-TLS (TLS 1.2/1.3) かつ Cipher Suite が `TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384` であることを確認。

---

### 【演習 10】FlexConnect Local Switching 環境における WPA3-Personal (SAE) の適用

#### 問題
FlexConnect 拠点において、ローカルスイッチング（Local Switching）構成の WPA3-Personal (SAE) SSID を構築せよ。

#### 要件
1. Profile Name: `FLEX-WPA3-PROF` / SSID: `FLEX-SAE-SSID`
2. FlexConnect Local Switching 有効化。
3. AKM: WPA3 SAE。
4. PMF: Mandatory。
5. パスフレーズ: `FlexSaePass2026!`

#### 設定例
```text
wlan FLEX-WPA3-PROF 28 FLEX-SAE-SSID
 security wpa psk set-key ascii 0 FlexSaePass2026!
 security wpa akm sae enable
 security pmf mandatory
 no shutdown

wireless profile policy FLEX-SMART-POLICY
 flex-profile FLEX-SITE-PROFILE
 vlan VLAN_GUEST_20
 central switching disable
 central dhcp disable
 no shutdown
```

#### 検証方法
```text
# show wireless client mac-address <MAC> detail
! Client State: RUN, FlexConnect Local Switching: Yes, AKM: SAE であることを確認。
```


## ❓ 想定試験問題

CCIE Security ラボ試験・筆記試験を意識した実践問題（全5問）。

### 【問題 1: コンフィグ読解・トラブルシューティング】
以下の Catalyst 9800 WLC のコンフィグレーションにおいて、WPA3 対応端末がアソシエーションに失敗する原因を指摘し、修正コンフィグを示せ。

```text
wlan WPA3-CORP 29 WPA3-CORP-SSID
 security wpa psk set-key ascii 0 CorporatePass123!
 security wpa akm sae enable
 no security wpa akm psk
 security wpa wpa3 ciphers aes enable
 no security wpa wpa2 ciphers aes
 security pmf optional
 no shutdown
```

* **解答・解説**:
  * **原因**: WPA3-Personal (Pure SAE) モードでは、IEEE 802.11 規格上 **PMF（Protected Management Frames）が Mandatory（必須）** である必要がある。コンフィグでは `security pmf optional` と設定されているため、RSN IE の整合性が崩れてアソシエーションが拒否される。
  * **修正コンフィグ**:
    ```text
    wlan WPA3-CORP 29 WPA3-CORP-SSID
     security pmf mandatory
    ```

---

### 【問題 2: トラブルシューティング】
OWE (Enhanced Open) Transition Mode を設定したところ、OWE 対応クライアントが通常のアンプラグド（Open）接続となり、OWE 暗号化に昇格しない。以下のコンフィグから不具合原因を特定せよ。

```text
wlan OWE-HIDDEN 30 OWE-HIDDEN-SSID
 owe transition-mode ssid GUEST-OPEN wlan 31
 security owe
 security pmf mandatory
 hide-ssid
 no shutdown

wlan GUEST-OPEN 31 GUEST-OPEN
 no security wpa
 no shutdown
```

* **解答・解説**:
  * **原因**: 可視側の Open SSID (`GUEST-OPEN`) に、非表示側 OWE SSID (`OWE-HIDDEN-SSID`) への `owe transition-mode` ポインタ設定が欠落している。両 SSID 間で相互に相手の SSID 名と WLAN ID を指定しなければ、Open SSID の 802.11 Beacon に OWE Vendor Specific IE が挿入されず、端末が OWE 存在を認知できない。
  * **修正コンフィグ**:
    ```text
    wlan GUEST-OPEN 31 GUEST-OPEN
     owe transition-mode ssid OWE-HIDDEN-SSID wlan 30
    ```

---

### 【問題 3: 設計】
WPA3-Enterprise 192-bit Mode（CNSA 準拠）を導入する際、WLC・Cisco ISE・クライアント証明書の間で確保すべきセキュリティパラメータ（ハッシュ関数、暗号アルゴリズム、鍵長）の要件を述べよ。

* **解答・解説**:
  1. **公開鍵・証明書**: ECDSA P-384 楕円曲線証明書（または RSA-4096 以上）。
  2. **認証ハッシュ・HMAC算法**: SHA-384（AKM 802.1X-SHA384）。
  3. **データ暗号化スイート**: 256-bit Galois/Counter Mode（AES-GCMP-256）。
  4. **管理フレーム保護 (PMF)**: BIP-GMAC-256 (Broadcast Integrity Protocol)。
  5. **TLS バージョン**: TLS 1.2 または TLS 1.3。

---

### 【問題 4: 実装・障害対応】
WPA3 Transition Mode SSID に接続を試みた一部の古めの WPA2-PSK 端末が、SSID 自体を検出できないか、アソシエーション直後に拒否される。この現象の技術的背景と現場での回避策を説明せよ。

* **解答・解説**:
  * **背景**: 古い無線のドライバー/ファームウェアは、Beacon / Probe Response 内の RSN IE に含まれる複数の AKM（PSK と SAE の並列広報）を正常に解釈できず、異常なフレームとして処理を中断してしまう。
  * **回避策**:
    1. クライアント側の Wi-Fi ドライバを最新にアップデートする。
    2. ドライバ更新不可能なレガシー端末群に対しては、Transition Mode SSID とは別に「WPA2-PSK 専用の隔離 SSID」を作成し、VLAN 分離のうえ収容する。

---

### 【問題 5: 動作原理】
WPA3-Personal (SAE) が、従来の WPA2-PSK に対する「オフライン辞書攻撃（Offline Dictionary Attack）」を防御できる理由を、SAE（Dragonfly ハンドシェイク）のメカニズムに基づいて説明せよ。

* **解答・解説**:
  * WPA2-PSK では、4-Way Handshake 内の EAPOL フレーム（ANonce/SNonce/MIC）を攻撃者が盗聴すると、共有パスフレーズから手元の高速計算環境（GPU等）でオフライン総当たり計算（PBKDF2+HMAC-SHA1）を行い、MIC が一致するパスワードを推測可能であった。
  * WPA3 SAE では、アクセスポイントとクライアントが認証フレーム交換時（Commit/Confirm フェーズ）に**エフェメラル Diffie-Hellman 鍵交換（ECDH）**を実行する。この鍵交換値はパケットを盗聴されてもパスワードを露呈させず、攻撃者はセッションごとに動的に変わるキーを解読できない。オフラインでの一括計算が不可能なため、辞書攻撃が無効化される。

---

## 🔗 参考リソース

* Cisco Catalyst 9800 Series Wireless Controller Software Configuration Guide - WPA3 Configuration
* Cisco Technical Notes - Configure WPA3 on Catalyst 9800 Wireless Controller
* Wi-Fi Alliance - WPA3 Specification
* Cisco Live Deployment Guide - BRKCRT-3001: Mastering WPA3 and Wi-Fi 6 Security Architecture
* Cisco Validated Design (CVD) - Secure Wireless Integration Guide with Cisco ISE

---

## 📝 補足（Notes）

* **WPA3-Personal SAE の PWE (Password Element) 生成アルゴリズム**:
  * H2E (Hash-to-Element) 方式と Hunting-and-Pecking 方式が存在する。Wi-Fi Alliance の最新 WPA3 規定では、サイドチャネル攻撃（タイミング攻撃）への耐性を高めるため H2E 方式の利用が推奨・必須化されている。
* **WPA3 と Wi-Fi 6E / 6GHz 帯（IEEE 802.11ax 6GHz）**:
  * 6GHz 帯（Wi-Fi 6E / Wi-Fi 7）では、**WPA2 やオープン（平文）接続の利用が規格上全面的に禁止** されている。6GHz 帯 SSID では **Pure WPA3 (SAE / WPA3-Enterprise) または Enhanced Open (OWE)** の設定が必須である（Transition Mode は 6GHz 帯では使用不可）。


