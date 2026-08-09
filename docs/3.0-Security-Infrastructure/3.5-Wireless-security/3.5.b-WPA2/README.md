---
layout: default
title: 3.5.b-WPA2
nav_order: 3
parent: 3.5-Wireless-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.5.b WPA2

**WPA2 (Wi-Fi Protected Access 2)** は、IEEE 802.11i 標準に基づいた無線 LAN セキュリティ規格であり、長年にわたりエンタープライズ無線ネットワークのデファクトスタンダードとして利用されています。CCIE Security v6.1 においては、単なる暗号化の設定だけでなく、**Cisco ISE (Identity Services Engine)** と連携した 802.1X 認証（Enterprise モード）、RADIUS 属性による動的なポリシー適用（VLAN 配布、ACL 適用）、およびトラブルシューティングが極めて重要な試験範囲となります。

---

## 📘 概要

*   **機能概要**: 脆弱な WEP や初期 WPA の後継として開発され、強力な暗号化アルゴリズム **AES (Advanced Encryption Standard)** と、パケットの整合性を担保する **CCMP (Counter Mode with Cipher Block Chaining Message Authentication Code Protocol)** を採用しています。
*   **利用目的**: 無線区間の通信傍受防止（秘匿性）、なりすまし接続の防止（認証）、パケット改ざんの検知（整合性）。
*   **利用場面**: 
    *   **WPA2-Personal (PSK)**: 事前共有鍵を使用。小規模拠点や IoT デバイス向け。
    *   **WPA2-Enterprise (802.1X)**: ユーザーごとに証明書や ID/PW で認証。企業のメインネットワーク向け。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **暗号化アルゴリズム** | **AES (FIPS 197 準拠)**。TKIP も選択可能だが非推奨。 |
| **整合性チェック** | **CBC-MAC (CCMP)**。 |
| **認証方式** | PSK（事前共有鍵）または 802.1X (EAP ベース)。 |
| **鍵管理** | **4-Way Handshake** による一時鍵 (PTK/GTK) の生成。 |
| **CCIE での焦点** | **WLC (Wireless LAN Controller)** と **ISE** の連携、AAA 属性による制御。 |
| **対応機種** | Catalyst 9800 シリーズ、AireOS WLC (3504, 5520 等)、各種 AP。 |
| **主な弱点** | KRACK 攻撃（4-Way Handshake の脆弱性）や、PSK におけるオフライン辞書攻撃。 |

---

## 🏗 動作原理

WPA2-Enterprise (802.1X) の通信フローは、Supplicant (端末)、Authenticator (WLC/AP)、Authentication Server (ISE) の三者間で行われます。

```
[ Client ]          [ WLC / AP ]          [ Cisco ISE ]
    |                    |                     |
    |<-- 802.11 Assoc -->|                     |
    |                    |                     |
    |<============= EAP Authentication =======>| (RADIUS over UDP)
    |                    |                     |
    |      (Success) <-------------------------| (RADIUS Accept + Attributes)
    |                    |                     |
    |<-- 4-Way Handshake -->| (Key Generation)  |
    |                    |                     |
    |<-- Encrypted Data -->|                     |
```

*   **4-Way Handshake**: 認証成功後、マスターキー (PMK) からセッションごとの一時鍵 (PTK: Pairwise Transient Key) を生成し、無線区間を暗号化します。

---

## ⚙ 動作シーケンス

1.  **L1/L2 接続**: クライアントが SSID をスキャンし、WPA2 ポリシーを確認してアソシエーションを完了。
2.  **802.1X 認証**: クライアントが EAP (PEAP, EAP-TLS 等) を開始。WLC はこれを RADIUS にカプセル化して ISE へ転送。
3.  **ポリシー決定**: ISE がユーザーを識別し、許可された場合は **RADIUS Accept** を返します。この際、VLAN ID や ACL 名などの属性を WLC に伝えます。
4.  **鍵交換 (4-Way Handshake)**: 
    *   Message 1: WLC から ANonce を送信。
    *   Message 2: Client が SNonce と MIC を返信。
    *   Message 3: WLC が GTK (Group Temporal Key) を含めて送信。
    *   Message 4: Client が完了を通知。
5.  **データ転送**: AES-CCMP による暗号化通信が開始されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **AAA Override の有効化**: ISE から送られる VLAN や ACL を適用するためには、WLC の WLAN 設定で `Allow AAA Override` を必ず有効にする必要があります。
*   **WLC ACL の適用**: 認証成功したクライアントに対し、WLC 側で定義した ACL（例: `WIFI_ACL`）を適用する要件が頻出です。
*   **FlexConnect の考慮**: 拠点の AP がローカルスイッチングを行う場合、ACL や VLAN の解決を AP 側（Local）で行うか WLC 側（Central）で行うかの設計が問われます。
*   **Fast Transition (802.11r)**: 移動時の再認証を高速化する設定と、その際の Over-the-Air / Over-the-DS の違い。
*   **IPv6 対応**: 無線クライアントに対する IPv6 ACL や DHCPv6 Guard の設定要否を確認してください。
*   **show コマンドの読み取り**: `show client detail` の出力から、`Policy Type: WPA2`、`AKM: 802.1x`、および適用されている ACL 名を即座に特定できる能力が求められます。

---

## 🛠 設定方法

### 1. WLC (AireOS) での WLAN 基本設定 (CLI)
```bash
! RADIUS サーバ（ISE）の登録
config radius auth add 1 172.16.100.10 1812 ascii Cisco123

! WLAN の作成と WPA2-Enterprise 設定
config wlan create 10 CORP-WIFI
config wlan security wpa akm 802.1x enable 10
config wlan security wpa encryption aes enable 10
config wlan radius auth add 10 1
! AAA Override の有効化（最重要）
config wlan aaa-override enable 10
config wlan enable 10
```

### 2. WLC ACL の作成と適用
```bash
! ACL の作成（Source にあるような WIFI_ACL の例）
config acl create WIFI_ACL
config acl rule add WIFI_ACL 1
config acl rule action WIFI_ACL 1 permit
config acl rule destination WIFI_ACL 1 0.0.0.0 0.0.0.0
! 認証成功時に RADIUS 属性 "Airespace-ACL-Name" でこの ACL 名を指定する
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **クライアントの詳細ステータス確認** | <code>show client detail [MAC_ADDR]</code> |
| **WLAN のセキュリティ設定確認** | <code>show wlan [WLAN_ID]</code> |
| **RADIUS サーバとの疎通確認** | <code>show radius summary</code> |
| **適用されているポリシーの確認** | <code>show client detail</code> 内の "Policy Type" |
| **詳細な認証プロセスデバッグ** | <code>debug client [MAC_ADDR]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 4-Way Handshake で止まる | PSK の不一致、または MIC エラー | <code>debug client</code> | 正しいパスワードを再入力。 |
| IP アドレスが取得できない | AAA Override 設定漏れで VLAN 不一致 | <code>show client detail</code> | <code>Allow AAA Override</code> を有効化。 |
| 認証に失敗する (RADIUS) | 共有鍵の不一致、ISE 側のポリシーミス | <code>show radius summary</code> | WLC と ISE の共有鍵を一致させる。 |
| ACL が効かない | WLC に ACL が定義されていない | <code>show acl summary</code> | 指定された ACL 名を WLC に作成。 |

---

## ⚠ 制限事項

*   **TKIP の使用制限**: 802.11n 以上の高速レート（HT/VHT）を利用する場合、TKIP 暗号化はサポートされず、AES が必須となります。
*   **レガシーデバイス**: 非常に古いデバイスは AES をサポートしていないことがあり、その場合は WPA(WPA1)-TKIP へのダウングレードが必要（セキュリティ低下）。
*   **証明書の期限**: 802.1X (EAP-TLS) を使用する場合、ISE やクライアントの証明書期限切れにより突然接続不能になるリスクがあります。

---

## 🔄 他技術との関連

*   **Cisco ISE (Identity Services Engine)**: 認証、ポスチャ、プロファイリングの統合管理。
*   **QoS**: 無線トラフィックに対する優先制御（WMM）。`show client detail` で QoS レベル（Silver 等）を確認可能。
*   **Layer 2 Security**: 無線ポートに対する DHCP Snooping や DAI の適用（WLC 上で制御）。
*   **WPA3 (3.5.c)**: WPA2 の後継。SAE や前方秘匿性 (PFS) を提供。WPA2 との Transition モード運用が可能。

---

## 🧩 比較表

### WPA2-Personal vs WPA2-Enterprise

| 特徴 | WPA2-Personal (PSK) | WPA2-Enterprise (802.1X) |
| :--- | :--- | :--- |
| **認証の仕組み** | 単一のパスワード | ユーザーごとの ID/PW または証明書 |
| **拡張性** | 低い（変更時に全端末再設定） | 高い（ISE で一括管理） |
| **セキュリティ** | 辞書攻撃に脆弱 | 非常に強固 |
| **主な用途** | ゲスト、家庭用、IoT | **エンタープライズ、CCIE 要件** |
| **追加コンポーネント** | 不要 | **RADIUS / ISE サーバ必須** |

---

## 💡 ベストプラクティス

1.  **AES-Only の構成**: 互換性の問題がない限り、暗号化は AES のみに限定し TKIP を無効化します。
2.  **AAA Override の活用**: ユーザーロールに基づき、動的に VLAN を分ける設計を推奨します。
3.  **Management Frame Protection (MFP)**: 802.11w (PMF) を有効にして、切断攻撃を防止します。
4.  **ロギングの集中管理**: WLC の Syslog を外部サーバに送り、不正な認証試行を監視します。

---

## 📝 ラボ学習・設定サンプル例

※以下の例は の show 出力を実現するための要件に基づいています。

### 1. 802.1X 認証 WLAN の構築
*   **要件**: SSID "CORP"、VLAN 100、WPA2-Enterprise (AES) を設定せよ。
*   **設定**: `config wlan security wpa akm 802.1x enable <id>`。

### 2. 動的 ACL の適用 (ISE 連携)
*   **要件**: ユーザー認証後、WLC 上の ACL `WIFI_ACL` を適用せよ。
*   **ISE 設定**: Authorization Profile で `Airespace-ACL-Name = WIFI_ACL` を送信。

### 3. 動的 VLAN 配布
*   **要件**: ISE のポリシーに基づき、VLAN 20 をクライアントに割り当てよ。
*   **WLC 設定**: `config wlan aaa-override enable <id>`。

### 4. QoS レベルの指定
*   **要件**: WLAN のトラフィッククラスを `Silver` (Best Effort) に設定せよ。
*   **設定**: `config wlan qos <id> silver`。

### 5. 再認証タイムアウトの設定
*   **要件**: クライアントの再認証間隔を 780 秒に設定せよ。
*   **設定**: `config wlan session-timeout <id> 780`。

### 6. IPv6 到達性の確保
*   **要件**: クライアントが IPv6 アドレスを取得し、外部と通信できることを確認せよ。

### 7. RADIUS Accounting の有効化
*   **要件**: ISE で接続時間を管理するため、Accounting 情報を送信せよ。

### 8. FlexConnect Local Switching
*   **要件**: 拠点 AP 配下のトラフィックを WLC を経由せずローカルで VLAN 10 にブリッジせよ。

### 9. Fast Transition (802.11r)
*   **要件**: AP 間の移動時に 4-Way Handshake を省略する FT を設定せよ。

### 10. SSID 隠蔽 (Broadcasting Off)
*   **要件**: セキュリティ強化のため、ビーコンに SSID を含めないようにせよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `show client detail` を見たところ、`IPv4 ACL Name: WIFI_ACL` と表示されている。この ACL はどこで定義され、どのように適用されたか？
    *   **回答**: WLC 上でローカルに定義されており、ISE から RADIUS 属性 `Airespace-ACL-Name` を通じて動的に適用された。
2.  **トラブルシュート**: 802.1X 認証は成功しているが、クライアントが正しい VLAN に所属しない。WLC 側で確認すべき最も可能性の高い設定項目は？
    *   **回答**: WLAN 設定の **`Allow AAA Override`** が `Disabled` になっている。
3.  **Design**: WPA2-Enterprise 環境で、ユーザーが複数の端末を持ち込んでいる。特定の MAC アドレスのみを許可しつつ 802.1X 認証を継続させる技術は？
    *   **回答**: **ISE プロファイリング** と **MAC Filtering (MAB)** の併用、あるいは **Identity PSK (iPSK)**。
4.  **実装**: 無線区間の暗号化において、AES ではなく TKIP を選択した場合の最大のパフォーマンス上のデメリットは？
    *   **回答**: 最大スループットが 54Mbps (802.11g 上限) に制限され、802.11n/ac/ax の高速化機能が利用できなくなる。
5.  **コンフィグ読解**: `config wlan security wpa akm psk enable 10` と `config wlan security wpa akm 802.1x enable 10` を同じ WLAN に設定できるか？
    *   **回答**: 原則として可能だが、同一の SSID で混合 AKM を使用すると一部の古いクライアントが接続できなくなる可能性があるため、設計上の注意が必要。

---

## 🔗 参考リソース

*   **Cisco WLC 8.10 Configuration Guide**
    *   [Configuring Security Solutions](https://www.cisco.com/c/en/us/td/docs/wireless/controller/8-10/config-guide/b_cg810/security.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [Wireless Security Deep Dive](https://www.ciscolive.com/)
*   **CVD (Cisco Validated Design)**
    *   [Campus Outreach Wireless Design Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/cisco-en-sw-design-guide.html)

---

## 📝 **補足（Notes）**
*   **学習メモ**: WPA2 は「ISE の手のひらの上」で動作します。ISE 側の Authorization Policy と WLC 側の AAA Override が噛み合っているかを確認するのが CCIE ラボ合格の鍵です。
*   **図解**: `show client detail` は、トラブルシューティングの際の「正解の状態」を示す地図です。この出力項目（Policy, AKM, ACL, VLAN）が全て要件通りになっているかを常に意識してください。
