---
layout: default
title: 3.5-Wireless-security
nav_order: 5
parent: 3.0-Security-Infrastructure
---

# 3.5 Wireless security technologies

無線 LAN セキュリティ技術は、物理的な境界がない「空気」を媒体とする通信を保護するために不可欠な要素です。CCIE Security v6.1 においては、**WPA/WPA2/WPA3** といった認証・暗号化フレームワークと、その基盤となる **TKIP/AES** 暗号プロトコルの理解、および **ISE (Identity Services Engine)** と連携したエンタープライズ認証の実装が重視されます,。

---

## 📘 概要

*   **機能概要**: 無線通信におけるパケットの秘匿性、整合性、およびデバイス・ユーザーの正当性を担保する技術群です。
*   **利用目的**: 盗聴、不正アクセス（乗り込み）、中間者攻撃、およびデータの改ざんからワイヤレスインフラを保護します。
*   **どのような場面で利用するか**: 
    *   **ゲストアクセス**: WPA3-SAE や OWE (Enhanced Open) による簡易かつセキュアな接続。
    *   **従業員アクセス**: WPA2/WPA3-Enterprise (802.1X) と ISE を組み合わせた厳格な ID ベースの制御。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **WPA (3.5.a)** | 初期のセキュリティ規格。TKIP を使用。現在は脆弱性が指摘され、非推奨。 |
| **WPA2 (3.5.b)** | AES-CCMP を採用。長らく標準として利用されている。 |
| **WPA3 (3.5.c)** | 最新規格。SAE (Simultaneous Authentication of Equals) や 192-bit 暗号を採用。 |
| **TKIP (3.5.d)** | WEP の弱点を補うために開発された一時的な暗号プロトコル。現在は使用不可または非推奨。 |
| **AES (3.5.e)** | 高度暗号化標準。WPA2/WPA3 の中核をなす強力なアルゴリズム。 |
| **802.1X/EAP** | WPA Enterprise における認証の仕組み。RADIUS/ISE が必須。 |

---

## 🏗 動作原理

無線セキュリティは、主に「認証（誰か）」と「暗号化（どう守るか）」の二段階で動作します。

```text
[ Wireless Client ]             [ Access Point / WLC ]             [ RADIUS / ISE ]
        |                               |                                 |
        |<------ 802.11 Association --->|                                 |
        |                               |                                 |
        |<========== 802.1X / EAP Authentication (Enterprise Mode) =======>|
        |                               |                                 |
        |<------ 4-Way Handshake ------>|                                 |
        |   (Generate PTK / GTK)        |                                 |
        |                               |                                 |
        |<------ Encrypted Data ------->|                                 |
```

---

## ⚙ 動作シーケンス

1.  **Discovery/Association**: クライアントが SSID を見つけ、AP にアソシエーションを要求。
2.  **Authentication**:
    *   **Personal (PSK/SAE)**: 事前共有鍵またはパスワードベースの認証。
    *   **Enterprise (802.1X)**: 証明書や ID/PW を用い、ISE 等の RADIUS サーバで検証。
3.  **Key Management**: **4-Way Handshake** を実行。
    *   一時鍵 (PTK: Pairwise Transient Key) とグループ鍵 (GTK: Group Temporal Key) を生成。
4.  **Encryption**: 決定されたプロトコル (AES-CCMP, GCMP 等) でデータペイロードを暗号化して転送開始。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **WPA3 の設定**: GUI (WLC) および CLI での WPA3-SAE および Enterprise 192-bit モードの設定手順を確認しておいてください。
*   **Transition Mode**: WPA2 と WPA3 が混在する環境（Transition Mode）の要件と、その際の制限（PMF 必須など）を理解する必要があります。
*   **ISE との連携**: ラボ試験では、WLC で RADIUS サーバ（ISE）を登録し、WLAN に 802.1X 認証を適用する構成が頻出です。
*   **MFP (Management Frame Protection)**: 802.11w の設定要否。WPA3 では **PMF (Protected Management Frames)** が必須となります。
*   **ACL の適用**: 特定の無線クライアントに対し、WLC レベルでトラフィック制限をかける設定（Source の `IPv4 ACL Name: WIFI_ACL` 参照）が問われます。
*   **FlexConnect**: ローカルスイッチング環境での認証と ACL の挙動の違い。

---

## 🛠 設定方法

### 1. WPA2-Enterprise (802.1X) - WLC CLI 例
```bash
! RADIUS サーバの登録
config radius auth add 1 172.16.100.10 1812 ascii-0 Cisco123
! WLAN の作成と 802.1X 有効化
config wlan create 10 CORP-WIFI
config wlan security wpa akm 802.1x enable 10
config wlan security wpa encryption aes enable 10
config wlan radius auth add 10 1
config wlan enable 10
```

### 2. WPA3-SAE (Personal) の設定
```bash
! WPA3 の有効化と SAE パスワード設定
config wlan security wpa akm sae enable 10
config wlan security wpa wpa3 sae psk set 10 ascii SecretPass123
! PMF (Protected Management Frame) を必須に設定
config wlan security pmf required 10
```

### 3. WPA3-Enterprise (192-bit)
```bash
! 192-bit 向けに GCMP-256 暗号化を指定
config wlan security wpa akm suiteb-gcmp256 enable 10
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **クライアントの接続状態・認証詳細** | <code>show client detail [MAC_ADDR]</code> |
| **WLAN のセキュリティ設定確認** | <code>show wlan [WLAN_ID]</code> |
| **RADIUS サーバのステータス** | <code>show radius summary</code> |
| **AP の稼働状態** | <code>show ap summary</code> |
| **詳細なデバッグ** | <code>debug client [MAC_ADDR]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| クライアントが接続できない | RADIUS 共有鍵の不一致 | <code>debug client</code> で RADIUS 拒否を確認。 |
| WPA3 クライアントが切断される | PMF が有効になっていない | WPA3 では <code>pmf required</code> が必須。 |
| 通信速度が極端に遅い | TKIP が有効になっている | TKIP は 802.11n 以降の高速化を阻害するため AES に統一。 |
| 認証後の通信ができない | WLAN ACL の定義ミス | <code>show client detail</code> で適用 ACL を確認。 |

---

## ⚠ 制限事項

*   **ハードウェア互換性**: 古い AP（Wave 1 以前）では WPA3 をサポートしていない場合があります。
*   **TKIP の使用**: TKIP を有効にすると、無線スループットが最大 54Mbps (802.11g レベル) に制限されます。
*   **Transition Mode の脆弱性**: WPA2/WPA3 移行モードでは、ダウングレード攻撃のリスクが一部残ります。

---

## 🔄 他技術との関連

*   **Cisco ISE**: 802.1X 認証、プロファイリング、ポスチャ（検疫）の心臓部。
*   **Layer 2 Security (3.4)**: 無線クライアントに対する DHCP Snooping や DAI の適用（WLC 上で設定）。
*   **FlexConnect**: 拠点の AP が WAN 越しに WLC と通信する際のデータプレーン保護。
*   **Identity PSK (iPSK)**: 単一の SSID でデバイスごとに異なる PSK を割り当てる ISE 連携技術。

---

## 🧩 比較表

### WPA2 vs WPA3

| 特徴 | WPA2 (AES-CCMP) | WPA3 (SAE/Enterprise) |
| :--- | :--- | :--- |
| **PSK 認証方式** | 4-Way Handshake | **SAE (Dragonfly)** |
| **オフライン攻撃** | 辞書攻撃に脆弱 | 非常に強い（前方秘匿性あり） |
| **暗号強度** | 128-bit | **128-bit / 192-bit** |
| **管理フレーム保護 (PMF)** | 任意 (Optional) | **必須 (Required)** |
| **オープンネットワーク** | 保護なし | **OWE (Enhanced Open)** により暗号化 |

---

## 💡 ベストプラクティス

1.  **AES-Only の強制**: 互換性の問題がない限り、暗号化は AES (CCMP/GCMP) のみに限定します。
2.  **Enterprise 認証の採用**: 組織内では PSK を避け、ISE と連携した 802.1X 認証（証明書ベース）を推奨します,。
3.  **PMF の有効化**: 既存の WPA2 環境でも管理フレーム保護を有効化し、切断攻撃（Deauthentication attack）を防ぎます。
4.  **SSID の分離**: 管理用、従業員用、ゲスト用で SSID を分け、異なるセキュリティポリシー（VACL/WLAN ACL）を適用します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な 802.1X 無線 LAN の構築
*   **要件**: WLAN ID 10 "Staff" を作成し、AES 暗号化と ISE による 802.1X 認証を設定せよ。
*   **設定**: `config wlan create 10 Staff`, `config wlan security wpa akm 802.1x enable 10`。

### 2. WPA3-SAE の実装
*   **要件**: ゲスト向けにパスワード "CiscoLab123" を使用した WPA3-SAE 接続を提供せよ。

### 3. OWE (Enhanced Open) の設定
*   **要件**: 認証なし（Open）だが通信は暗号化される SSID を作成せよ。
*   **設定**: AKM に `owe` を指定。

### 4. 192-bit セキュリティモードの強制
*   **要件**: 最高機密情報を扱う WLAN で WPA3 Suite-B モードを設定せよ。

### 5. WLAN ACL による通信制限
*   **要件**: 無線クライアントから管理サブネット (10.1.1.0/24) への通信を遮断せよ。

### 6. PMF の設定検証
*   **課題**: `pmf required` 設定時、PMF 非対応端末がアソシエーションに失敗することを確認せよ。

### 7. FlexConnect ローカル認証
*   **要件**: WLC との通信断時も拠点で認証を継続できるよう設定せよ。

### 8. WebAuth (LWA) の実装
*   **要件**: ゲスト向けに ISE のポータル画面を表示させる認証を構成せよ。

### 9. RADIUS フォールバック
*   **要件**: メインの ISE がダウンした際、セカンダリ ISE へ切り替える設定をせよ。

### 10. クライアント詳細の読み取り
*   **課題**: `show client detail` の出力から、現在の AKM と暗号化プロトコルを特定せよ。

---

## ❓ 想定試験問題

1.  **Design**: WPA3 を導入する際、WPA2 しかサポートしない旧型デバイスも同時にサポートするための設定モードは？
    *   **回答**: **WPA3 Transition Mode**。
2.  **トラブルシュート**: `show client detail` を確認したところ、認証は成功しているがトラフィックが全く流れない。出力に `IPv4 ACL Name: DENY-ALL` とある。何が起きているか？
    *   **回答**: クライアントに対し、WLC 上のポリシーまたは RADIUS からの属性付与により、全拒否 ACL が適用されている。
3.  **コンフィグ読解**: WPA3-SAE を設定した WLAN で、一部の端末が SSID に接続できなくなった。設定には `config wlan security pmf disabled 10` とある。原因は？
    *   **回答**: WPA3 規格では PMF (Protected Management Frames) が必須であり、無効化されていると正常に動作しない。
4.  **Design**: 公衆無線 LAN のような「パスワードなし」の環境で、盗聴を防ぐために導入すべき最新技術は？
    *   **回答**: **OWE (Opportunistic Wireless Encryption / Enhanced Open)**。
5.  **実装**: 802.1X 認証時に ISE から特定の VLAN を割り当てる（VLAN Override）ために、WLC の WLAN 設定で有効にすべき項目は？
    *   **回答**: **Allow AAA Override**。

---

## 🔗 参考リソース

*   [Cisco WLC 8.10 Configuration Guide: Configuring Security Solutions](https://www.cisco.com/c/en/us/td/docs/wireless/controller/8-10/config-guide/b_cg810/security.html)
*   [WPA3 Deployment Guide - Cisco](https://www.cisco.com/c/en/us/td/docs/wireless/controller/technotes/8-8/b_wpa3_deployment_guide.html)
*   [Cisco Live: BRKWRL-2020 - Wireless Security Deep Dive](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**
- **学習メモ**: 無線セキュリティは「4-Way Handshake」が全ての鍵生成の基本です。WPA3 ではこれが SAE というより強固な方式に置き換わっている点に注目してください。
- **図解**: Source の出力にある `Policy Type: WPA2` や `AKM: 802.1x` は、実際のラボ試験で設定が意図通り反映されているかを確認する最重要ポイントです。
- **注意点**: ラボ試験のトポロジによっては、物理 WLC ではなく vWLC や Catalyst 9800-CL (IOS-XE) が使用されるため、構文の違い（特に 9800 の Tag ベース設定）に慣れておく必要があります。
