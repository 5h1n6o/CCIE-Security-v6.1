---
layout: default
title: 4.2-Network-access-AAA
nav_order: 4
parent: 4.0-Identity-Management
---

# 4.2 Cisco switches and Cisco Wireless LAN Controllers for network access AAA with Cisco ISE

Cisco ISE (Identity Services Engine) を中心としたネットワークアクセス制御において、スイッチ (Switch) とワイヤレス LAN コントローラ (WLC) は **NAD (Network Access Device)** として機能し、エンドポイントの認証・認可を強制する重要な役割を担います。CCIE Security v6.1 では、単なる RADIUS 設定だけでなく、IBNS 2.0 (Cisco Common Classification Policy) を用いた高度なポリシー適用や、有線・無線が混在する環境での一貫したセキュリティ強制能力が問われます。

---

## 📘 概要

*   **機能概要**: スイッチや WLC が RADIUS クライアントとして ISE と対話し、802.1X、MAB (MAC Authentication Bypass)、WebAuth を使用してユーザやデバイスを識別・制御する機能です。
*   **利用目的**: ネットワークへの不正アクセスの防止、ユーザ属性に基づいた動的な VLAN 割り当て、dACL (Downloadable ACL) の適用、および TrustSec (SGT) によるセグメンテーションの実現。
*   **どのような場面で利用するか**: 
    *   **有線アクセス**: オフィス内のデスクポートにおける 802.1X 認証。
    *   **無線アクセス**: 社内 Wi-Fi における WPA2/WPA3 Enterprise 認証。
    *   **ゲストアクセス**: CWA (Central Web Authentication) を用いた一時的なインターネット接続。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **認証方式** | 802.1X (EAP), MAB, WebAuth。 |
| **RADIUS 属性** | VLAN ID, dACL, SGT (Security Group Tag), Session-Timeout。 |
| **CoA (Change of Authorization)** | ポスチャ変更や属性更新時に ISE から NAD へ通知し、セッション状態を即座に変更する。 |
| **IBNS 2.0** | `service-template` と `access-session` を用いた有線 NAD の次世代構成モデル。 |
| **WLC セキュリティ** | WPA2/WPA3 Enterprise、RADIUS NAC、AAA Override。 |
| **プロファイリング** | Device Sensor (LLDP/CDP/HTTP) を使用して NAD が属性を収集し ISE へ送信。 |

---

## 🏗 動作原理

ネットワークアクセスの AAA フローは、NAD がサプリカント (エンドポイント) と認証サーバ (ISE) の仲介役となることで成立します。

```text
Supplicant (PC/IoT)
      ↓ (1) EAPoL / MAC Packet
NAD (Switch / WLC)
      ↓ (2) RADIUS Access-Request
Cisco ISE
      ↓ (3) Verify Identity / Policy
      ↑ (4) RADIUS Access-Accept (Attributes: VLAN, dACL, SGT)
NAD (Switch / WLC)
      ↓ (5) Enforce Policy on Port/WLAN
      ↓ (6) RADIUS Accounting-Request (Start)
Cisco ISE
```

1.  **トリガー**: デバイスがリンクアップするか、パケットを送信した際に NAD が検知します。
2.  **認証フェーズ**: NAD は RADIUS を介して ISE へ問い合わせます。
3.  **認可フェーズ**: ISE は認証に成功すると、動的なアクセス権限（VLAN や ACL）を RADIUS 応答に含めます。
4.  **強制フェーズ**: NAD は指定された属性をポートや無線クライアントに適用します。

---

## ⚙ 動作シーケンス (IBNS 2.0 有線認証)

1.  **セッション開始**: ポートでトラフィックを受信し、`access-session` が生成されます。
2.  **ポリシー照合**: 構成された `class-map type control` に基づき、802.1X または MAB が開始されます。
3.  **ISE 問い合わせ**: スイッチが `Access-Request` を送信します。
4.  **属性適用**: ISE からの `Access-Accept` を受け取り、`service-template` に定義された属性（または直接の RADIUS 属性）が適用されます。
5.  **CoA 操作**: 必要に応じて ISE から `CoA-Request` が送られ、ポートの再認証や切断が行われます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **IBNS 2.0 の完全理解**: ラボ試験では従来の `authentication ...` コマンドではなく、`policy-map type control` を使用した IBNS 2.0 の設定が求められる可能性が高いです。
*   **WLC の RADIUS 設定**: 認証用とアカウンティング用のサーバを正しく登録し、**RFC 5176 (CoA)** を有効にするチェックボックスを忘れないことが重要です。
*   **AAA Override の有効化**: WLC の WLAN 設定において、ISE からの属性（VLAN や ACL）を優先させるための `AAA Override` を有効に設定する必要があります。
*   **RADIUS キーの不一致**: スイッチと ISE 間で共有シークレットが一致していないために認証が失敗するトラブルシュート問題に備えてください。
*   **dACL の構文**: ISE 側で定義された dACL がスイッチで正しく適用されているか、`show ip access-lists` で確認する手順を習得してください。
*   **FlexConnect の考慮**: WLC がローカルスイッチングモードの場合、ACL や VLAN のマッピングがスイッチ側でも必要になる点に注意してください。

---

## 🛠 設定方法

### 1. スイッチ：IBNS 2.0 認証ポリシーの基本構成
```bash
! RADIUS サーバの定義
radius server ISE
 address ipv4 10.1.1.100 auth-port 1812 acct-port 1813
 key cisco123
!
! AAA 構成
aaa new-model
aaa authentication dot1x default group radius
aaa authorization network default group radius
aaa accounting dot1x default start-stop group radius
!
! IBNS 2.0 ポリシー
class-map type control match-all DOT1X-MATCH
 match method dot1x
!
policy-map type control PORT-POLICY
 class type control DOT1X-MATCH seq 10
  authenticate using dot1x priority 10
 class type control always seq 20
  authenticate using mab priority 20
!
! インターフェイスへの適用
interface GigabitEthernet1/0/1
 access-session port-control auto
 service-policy type control input PORT-POLICY
 mab
 dot1x pae authenticator
```

### 2. WLC：ISE 連携のための RADIUS 構成 (GUI 手順)
1.  **Security > AAA > RADIUS > Authentication** に移動。
2.  **New** をクリックし、ISE の IP (10.1.1.100) と共有シークレットを入力。
3.  **Support for RFC 5176 (CoA)** を有効にする。
4.  **WLANs > WLAN ID > Security > AAA Servers** タブで、作成した ISE サーバを選択。
5.  **Advanced** タブで **Allow AAA Override** をチェック。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **有線セッションのステータス確認** | <code>show access-session interface [int] details</code> |
| **WLC クライアントの認証状態確認** | <code>show client details [MAC_ADDR]</code> |
| **RADIUS サーバの接続性確認** | <code>show aaa servers</code> |
| **適用された ACL の確認** | <code>show ip access-lists</code> |
| **dACL の詳細確認** | <code>show epm session interface [int]</code> |
| **認証デバッグ (スイッチ)** | <code>debug dot1x all</code> または <code>debug authentication all</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| クライアントが IP を取得できない | VLAN が ISE で指定されているが NAD に存在しない | <code>show vlan</code> で存在を確認。 |
| dACL が適用されない | スイッチに IP デバイス追跡 (IPDT) が設定されていない | <code>device-tracking upgrade-cli</code> を確認。 |
| WLC で WebAuth リダイレクトが失敗 | ACL の `deny` 行で ISE への通信を許可していない | リダイレクト用 ACL の先頭で ISE 通信を <code>permit</code> する。 |
| CoA が効かない | NAD 側で CoA ポート (1700/3799) が開放されていない | <code>aaa server radius dynamic-author</code> 設定を確認。 |

---

## ⚠ 制限事項

*   **dACL と SGT の併用**: プラットフォームによって、dACL と SGT を同時に単一セッションに適用できない場合があります。
*   **WLC FlexConnect**: FlexConnect Local Switching を使用する場合、dACL はサポートされず、NAD スイッチ上のローカル ACL 名を指定する必要があります。
*   **認証タイムアウト**: 遅延の大きい WAN 経由の認証では、RADIUS タイムアウト値をデフォルト (5秒) から増やす必要があります。

---

## 🔄 他技術との関連

*   **2.6 Microsegmentation (TrustSec)**: ISE から SGT を NAD に配信し、SXP で伝播させる。
*   **3.4.a DAI / 3.4.e DHCP Snooping**: 認証成功後に IP 偽装を防ぐための L2 セキュリティと ISE セッションの連携。
*   **4.1 ISE Scalability**: 複数の PSN ノードに対して NAD が負荷分散を行う構成。
*   **3.10 Cisco DNAC**: SD-Access 環境において、ISE ポリシーを自動的に NAD にプロビジョニングする。

---

## 🧩 比較表

### Wired vs Wireless AAA Integration (with ISE)

| 特徴 | 有線 (Switch) | 無線 (WLC) |
| :--- | :--- | :--- |
| **主なプロトコル** | 802.1X over Ethernet (EAPoL) | 802.11i (WPA2/3 Enterprise) |
| **ACL 適用** | **dACL** (ISE からダウンロード) | **Named ACL** (WLC に事前定義) |
| **リダイレクト** | HTTP リダイレクト (VACL/IP Redirect) | WLC 内部でのリダイレクト処理 |
| **セッション識別** | ポート ID + MAC | クライアント MAC アドレス |
| **CoA サポート** | ポートリスタート、再認証 | クライアント切断、プロファイル更新 |

---

## 💡 ベストプラクティス

1.  **Critical Auth (Fail-Open)**: ISE がダウンした場合に備え、スイッチポートを特定の VLAN に自動配置する `aaa critical vlan` を構成します。
2.  **RADIUS ソース IP の固定**: `ip radius source-interface` を使用し、ISE 側での NAD 登録 IP を一貫させます。
3.  **Device Sensor の活用**: スイッチの CDP/LLDP 情報を RADIUS 属性に含めて送信することで、ISE での高精度なデバイス識別（例：Cisco IP Phone）を可能にします。
4.  **CoA サーバ設定**: スイッチ上で `aaa server radius dynamic-author` を必ず設定し、ISE からの変更通知を受け入れ可能にします。

---

## 📝 ラボ学習・設定サンプル例

### 1. 有線：802.1X + MAB のフェイルオーバー
*   **要件**: DOT1X を最初に試行し、失敗した場合は MAB に切り替えよ。
*   **設定**: IBNS 2.0 `PORT-POLICY` で priority を調整（前述の設定方法参照）。

### 2. 有線：動的 VLAN 割り当て (Dynamic VLAN Assignment)
*   **要件**: ISE の認可結果に基づき、VLAN 10 または 20 を動的に割り当てよ。
*   **設定**: `radius-server attribute 6 on-for-login-auth` (VLAN 属性の有効化)。

### 3. 有線：ISE への Device Sensor 送信
*   **要件**: CDP および LLDP の属性を ISE に報告せよ。
*   **設定**: 
    ```bash
    device-sensor filter-list cdp list CDP-LIST
     tlv name device-name
    device-sensor notify all-attributes
    ```

### 4. 無線：WPA2 Enterprise (PEAP) の構成
*   **要件**: WLC で SSID "Corporate" を作成し、RADIUS 認証を強制せよ。

### 5. 無線：AAA Override による SGT 適用
*   **要件**: ISE から SGT 5 を無線クライアントに適用せよ。
*   **設定**: WLAN の `AAA Override` を有効にし、ISE 側の認可プロファイルで SGT を指定。

### 6. スイッチ：CoA (Change of Authorization) 受信許可
*   **要件**: ISE (10.1.1.100) からの切断要求を受け入れよ。
*   **設定**: 
    ```bash
    aaa server radius dynamic-author
     client 10.1.1.100 server-key cisco123
    ```

### 7. スイッチ：IBNS 2.0 での WebAuth
*   **要件**: 802.1X/MAB 両方に失敗したエンドポイントに対し、WebAuth を開始せよ。

### 8. 有線：マルチドメイン認証 (IP Phone + PC)
*   **要件**: 同一ポートで音声 VLAN とデータ VLAN の両方を個別に認証せよ。
*   **設定例**: `access-session host-mode multi-domain`。

### 9. 有線：dACL による特定ポートの遮断
*   **要件**: ISE で定義した `permit tcp any any eq 80` の ACL をダウンロードし適用せよ。

### 10. 有線：CWA (Central Web Authentication) のリダイレクト
*   **要件**: 認証前のブラウザ通信を `https://ise-server:8443...` へ転送せよ。
*   **設定**: ポートに `ip http server` 設定と、ISE からのリダイレクト URL 属性の受け入れを確認。

---

## ❓ 想定試験問題

1.  **Design**: スイッチで IBNS 2.0 を使用する際、従来の `authentication host-mode multi-auth` に相当する設定はどこで行うか？
    *   **回答**: インターフェイスの下で `access-session host-mode multi-auth` を直接指定する。
2.  **トラブルシュート**: WLC で RADIUS 認証は成功しているが、クライアントが ISE で指定した VLAN に入らない。WLC のどこを修正すべきか？
    *   **回答**: 該当 WLAN の設定で **`Allow AAA Override`** が無効になっている可能性があるため、これを有効にする。
3.  **実装**: ISE が一時的に到達不能な場合でも、有線ポートで最低限の通信を許可するために必要な設定は？
    *   **回答**: `aaa critical vlan` または `service-template` 内での `vlan` 指定による **Critical Auth** の構成。
4.  **コンフィグ読解**: `radius-server vsa send authentication` というコマンドの役割は？
    *   **回答**: ベンダー固有属性 (VSA) を RADIUS リクエストに含めて ISE に送信することを許可する。
5.  **Design**: 多数のスイッチがある環境で、dACL の一貫性を保つためのメリットは？
    *   **回答**: ACL を各スイッチにローカル定義する必要がなく、ISE 上で一元管理し、認証時に動的に配信できるため。

---

## 🔗 参考リソース

*   **Configuration Guide**: [Cisco IOS-XE Identity-Based Networking Services (IBNS) 2.0](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ibns/configuration/xe-16/ibns-xe-16-book.html)
*   **CVD**: [Cisco ISE Wired and Wireless Deployment Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/ise-deployment-guide.html)
*   **Cisco Live (BRKSEC-2022)**: [Advanced Wired Access Control](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 有線・無線で ISE との「共通言語」は RADIUS ですが、属性の適用方法（dACL vs Named ACL）が微妙に異なる点を整理しておきましょう。
*   **図解**: 
    - **Control-Plane**: RADIUS メッセージのやり取り。
    - **Data-Plane**: 受信ポートにおける ACL や VLAN の強制。
*   **注意点**: ラボ試験では、**CoA ポートの設定ミス**により「ポスチャ後に権限が上がらない」という状況が頻出します。`aaa server radius dynamic-author` は必須項目です。
