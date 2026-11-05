---
layout: default
title: 3.6.b-SNMP
nav_order: 2
parent: 3.6-Monitoring
grand_parent: 3.0-Security-Infrastructure
---

# 3.6.b SNMP

## 📘 概要

Simple Network Management Protocol (SNMP) は、IP ネットワーク上のネットワーク機器（ルータ、スイッチ、ファイアウォール、サーバー等）の動作状態監視、パフォーマンス計測、および構成管理を行うための標準アプリケーション層プロトコル（UDP 161/162）です。

CCIE Security v6.1 においては、単なるネットワーク監視プロトコルとしてではなく、ネットワーク基盤のセキュリティ（Management Plane Protection）、制御プレーンの保護（CoPP/CPPr）、および暗号化・認証技術と組み合わせた安全な機器管理手法として非常に重視されます。

SNMP は歴史的に v1、v2c、v3 の 3 つの主要バージョンが存在します。
* **SNMPv1 / v2c**: コミュニティ文字列（Community String）と呼ばれる平文テキストによる簡易認証を使用するため、盗聴や改ざん、不正アクセスに極めて脆弱です。
* **SNMPv3**: USM (User-based Security Model) による強力な認証（MD5 / SHA-1 / SHA-256 / SHA-512）および暗号化（DES / 3DES / AES-128 / AES-192 / AES-256）、ならびに VACM (View-based Access Control Model) による細粒度な MIB オブジェクトアクセス制御を提供します。

インフラ保護の観点からは、SNMPv3 の authPriv モードの適用、コミュニティ文字列の廃止、アクセス制御リスト (ACL) または VRF によるアクセス限定、ならびに SNMP トラップ/インフォームの送信元インターフェイス指定が必須要件となります。


---

## 🔑 要点

SNMP（特に SNMPv3）の主要な機能特性、用途、メリット・デメリット、対応機種、制限事項および設計上の注意点を下表に示します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | USM によるユーザーベース認証・パケット暗号化、VACM による MIB View ベースのアクセス制御、Trap/Inform によるイベント通知メカニズム |
| **用途** | 機器ステータス監視（CPU/メモリ/温度/インターフェイス状態）、セキュリティイベント通知、MIB 変数の取得（GET/GETNEXT/BULK）および設定変更（SET） |
| **メリット** | 標準化されたプロトコルであり広範な NMS でサポート、SNMPv3 ではデータ機密性・完全性・送信元認証を完全に確保可能 |
| **デメリット** | SNMPv1/v2c は平文コミュニティのため極めて危険、SNMPv3 は鍵管理やエンジン ID（EngineID）同期のオーバーヘッドが発生 |
| **対応機種** | Cisco IOS / IOS-XE, Cisco ASA, Cisco FTD / FMC, Catalyst スイッチ, Nexus スイッチ等全 Cisco 機器 |
| **制限事項** | 大規模ネットワークにおける大量の SNMP ポーリングは CPU 負荷を引き起こす可能性あり。ASA や FTD では一部の機能制限（SET 操作非対応等）が存在 |
| **設計上の注意点** | authPriv (SHA/AES) の強制、管理専用 VRF/サブネットへの限定、CoPP による SNMP トラフィック制限、EngineID の統一管理 |


---

## 🏗 動作原理

SNMP は、管理者側の NMS (Network Management System / SNMP Manager) と、被監視機器側の SNMP エージェント (SNMP Agent) 間で通信を行います。

### 通信アーキテクチャ

```
[ NMS (SNMP Manager) ]
   │
   ├── (1) GET / GETNEXT / GETBULK / SET Request ──> [ UDP 161 ]
   │                                                     │
   │<── (2) Response (Data / PDU) ───────────────────────┤
   │                                                     │
   │<── (3) SNMP Trap (Unacknowledged) ──────────── [ UDP 162 ]
   │                                                     │
   │<── (4) SNMP Inform Request (Acknowledged) ──── [ UDP 162 ]
   │─── (5) Inform Response (ACK) ──────────────────────>│
                                                         ▼
                                             [ Cisco Device (SNMP Agent) ]
                                                         │
                                                  [ MIB Database ]
```

### 1. SNMPv3 USM (User-based Security Model)
USM は、SNMP メッセージのセキュリティを担当し、以下の 3 つのセキュリティレベルを提供します。
* **noAuthNoPriv**: 認証なし・暗号化なし（ユーザー名の一致のみで確認）。
* **authNoPriv**: 認証あり（HMAC-MD5, HMAC-SHA-1, SHA-224, SHA-256, SHA-384, SHA-512）・暗号化なし。
* **authPriv**: 認証あり ＋ 暗号化あり（CBC-DES, 3DES, AES-128, AES-192, AES-256）。

### 2. SNMPv3 VACM (View-based Access Control Model)
VACM は、特定のユーザーグループに対してアクセスを許可または制限する MIB ツリーの範囲（MIB View）を定義します。
* **MIB View**: MIB OID ツリー（例: `1.3.6.1.2.1`）の一部を含める（`included`）または除外する（`excluded`）定義。
* **Group Access**: 特定のグループに対して、Read View（読み取り）、Write View（書き込み）、Notify View（トラップ通知）を割り当てます。


---

## ⚙ 動作シーケンス

SNMPv3 メッセージにおける処理およびパケット検証のフローは以下の通りです。

```
[ NMS ]                                            [ Cisco SNMP Agent ]
   │                                                       │
   │ 1. SNMPv3 メッセージ生成                              │
   │    - EngineID, UserName, MsgID 設定                   │
   │    - PDU を AES-128 等で暗号化 (authPriv 時)          │
   │    - HMAC-SHA 等で認証ダイジェスト付与                │
   │                                                       │
   │ ─────────────── 2. UDP 161 (GET/SET) ───────────────> │
   │                                                       │
   │                                                       │ 3. USM パケット検証
   │                                                       │    - EngineID & タイムスタンプ検証
   │                                                       │    - HMAC ダイジェスト一致確認
   │                                                       │    - 秘密鍵で PDU を復号
   │                                                       │
   │                                                       │ 4. VACM アクセス権検証
   │                                                       │    - ユーザー所属グループの取得
   │                                                       │    - 対象 OID が Read/Write View 内か照合
   │                                                       │    - ACL による送信元 IP チェック
   │                                                       │
   │                                                       │ 5. MIB データベースアクセス
   │                                                       │    - 要求 OID の値を取得 / 設定
   │                                                       │
   │ <────────────── 6. Response PDU 送信 ─────────────── │
   │   (USM 暗号化 & 認証ダイジェスト適用)                │
```


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security ラボ試験における SNMP の頻出テーマおよび実装上の注意点は以下の通りです。

### ラボ試験で出題されやすい要件
1. **SNMPv3 authPriv の厳格な構成**:
   * 指定された HMAC 認証アルゴリズム（SHA-256 等）および暗号化アルゴリズム（AES-128 / AES-256 等）を用いた USM ユーザーの設定。
2. **VACM による細粒度な MIB アクセス制御**:
   * システム情報（`sysUpTime`, `sysName`）やインターフェイス統計（`ifTable`）のみを許可し、IP ケーブリング情報やルーティングテーブル OID（`ipRouteTable` 等）を除外する MIB View 設定。
3. **トラップとインフォーム (Informs) の区別**:
   * 単なる UDP 通知（Trap）ではなく、NMS からの ACK 確認応答を要求する Inform メッセージの設定（`snmp-server host ... informs`）。
4. **送信元 IP / インターフェイスの固定**:
   * ルーティングの変動やマルチホーム環境に依存しないよう、Loopback インターフェイスを SNMP ソース（`snmp-server source-interface`）として指定。
5. **プラットフォームごとの構文の違い**:
   * Cisco IOS-XE CLI と Cisco ASA CLI、ならびに Cisco FTD (FMC FlexConfig / Platform Settings) の各設定方法の網羅。


---

## 🛠 設定方法

### 1. Cisco IOS / IOS-XE CLI 設定例 (SNMPv3 authPriv & VACM)

```bash
! 1. MIB View の作成 (特定の OID を許可/除外)
snmp-server view READ-ONLY-VIEW iso included
snmp-server view READ-ONLY-VIEW 1.3.6.1.2.1.4.21 excluded   ! ipRouteTable を除外

! 2. アクセス制御用 ACL の作成
ip access-list standard ACL-NMS-ACCESS
 permit 192.168.100.50
 permit 192.168.100.51

! 3. SNMPv3 グループの作成 (authPriv & VACM View 適用)
snmp-server group SEC-ADMIN-GRP v3 priv read READ-ONLY-VIEW access ACL-NMS-ACCESS

! 4. SNMPv3 ユーザーの作成 (SHA-256 認証 & AES-128 暗号化)
snmp-server user SEC-USER SEC-ADMIN-GRP v3 auth sha-256 AuthPass123! priv aes 128 PrivPass123!

! 5. トラップ/インフォーム送信設定
snmp-server enable traps config
snmp-server enable traps cpu threshold
snmp-server host 192.168.100.50 version 3 priv SEC-USER informs
snmp-server source-interface trap Loopback0
```

### 2. Cisco ASA CLI 設定例 (SNMPv3)

```bash
! 1. MIB View の作成
snmp-server view ASA-VIEW iso included

! 2. SNMPv3 グループの作成
snmp-server group ASA-SEC-GRP v3 priv user-defined ASA-VIEW

! 3. SNMPv3 ユーザーの作成
snmp-server user ASA-SEC-USER ASA-SEC-GRP v3 auth sha AuthPass123! priv aes 128 PrivPass123!

! 4. 管理インターフェイスでの SNMP 受信許可
snmp-server host inside 192.168.100.50 version 3 ASA-SEC-USER
snmp-server location DataCenter-Rack1
snmp-server contact admin@example.com
snmp-server enable traps all
```

### 3. Cisco FTD (FMC Managed) 設定手順
FMC 管理下の FTD においては、**Platform Settings** で SNMP を設定します。
1. FMC GUI にログインし、`Devices` > `Platform Settings` に移動。
2. 適用対象の FTD ポリシーを編集し、`SNMP` セクションを選択。
3. `Enable SNMP Server` にチェックを入れる。
4. `Users` タブで SNMPv3 ユーザーを追加（Username, Auth Protocol: SHA-256, Auth Password, Privacy Protocol: AES-128, Privacy Password）。
5. `Hosts` タブで NMS の IP アドレス、インターフェイス（例: `mgmt` または `inside`）、SNMP バージョン（v3）を指定。


---

## 🔍 検証コマンド

SNMP の動作状態、設定内容、セッションおよびカウント値を確認するための主要コマンド一覧です。

| 目的 | コマンド |
| --- | --- |
| **SNMP 概要ステータス確認** | <code>show snmp</code> |
| **SNMPv3 ユーザー一覧確認** | <code>show snmp user</code> |
| **SNMPv3 グループ一覧確認** | <code>show snmp group</code> |
| **VACM MIB View 設定確認** | <code>show snmp view</code> |
| **SNMP 送信統計・エラー確認** | <code>show snmp stats</code> |
| **EngineID の確認** | <code>show snmp engineID</code> |
| **ASA SNMP ホスト確認** | <code>show running-config snmp-server</code> |
| **SNMP トラフィックのデバッグ** | <code>debug snmp packets</code> |
| **SNMPv3 USM 処理デバッグ** | <code>debug snmp header</code> |


---

## 🚨 トラブルシュート

SNMP 構築および運用時に発生する代表的な障害症状、原因、確認コマンド、および対処方法を下表に整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **NMS からの GET リクエストに応答しない** | アクセス制限 ACL により NMS の IP が拒否されている | <code>show access-lists</code>, <code>show snmp</code> | ACL に NMS の IP アドレス（`permit`）を追加 |
| **SNMPv3 認証エラー (Authentication Failure)** | NMS 側と Agent 側で Auth パスワードまたはアルゴリズム (SHA/MD5) が不一致 | <code>debug snmp header</code>, <code>show snmp user</code> | 認証パスワードおよびアルゴリズム（SHA-1, SHA-256 等）の指定を両者で統一 |
| **SNMPv3 復号エラー (Decryption Error)** | Priv パスワードまたは暗号化方式 (AES-128/DES/AES-256) が不一致 | <code>debug snmp header</code> | 暗号キーおよび暗号化タイプ（AES 128 / 192 / 256）を統一 |
| **特定 OID (IfTable 等) が取得できない** | VACM MIB View 設定で該当 OID パスが `excluded` になっている | <code>show snmp view</code> | MIB View のエントリーを見直し、必要な OID パス（`1.3.6.1...`）を `included` に変更 |
| **EngineID 不一致による Inform / Trap 破棄** | NMS または Agent の再起動・変更後に EngineID の同期が乱れた | <code>show snmp engineID</code> | Agent の EngineID をクリア/静的定義し、NMS 側の被監視機器登録情報を更新 |
| **Inform メッセージが相手に届かない / 再送過多** | NMS までの到達性がない、または UDP 162 が途中の FW でドロップ | <code>show snmp stats</code> | ルーティング、インフラ ACL、および中間 FW の UDP 162 許可設定を確認 |


## ⚠ 制限事項

SNMP（特に SNMPv3）の実装および運用の際には、以下の制限事項および注意事項を考慮する必要があります。

1. **SHA-256 / SHA-512 および AES-192 / AES-256 のバージョン依存性**:
   * 古い Cisco IOS バージョンでは HMAC-SHA-1 および AES-128 / DES のみがサポートされており、HMAC-SHA-2 (SHA-256/512) や AES-256 (3DES/AES-192/256) を利用するには Cisco IOS-XE 16.x 以降等の近代的な暗号化イメージが必要です。
2. **ASA における SET リクエスト非サポート**:
   * Cisco ASA では、SNMP を使用した MIB 値の「読み取り (GET)」および「トラップ通知 (Trap)」のみがサポートされており、SNMP を経由した「設定変更 (SET)」操作はネイティブでサポートされていません。
3. **EngineID の変更とコンフィグ消去**:
   * 機器の `snmp-server engineID local` を変更すると、既存の `snmp-server user` コンフィグ内に保持されていた暗号化ハッシュ値が無効化され、ユーザー設定が消去または再設定を余儀なくされます。
4. **CoPP による過度なドロップ注意**:
   * CoPP (Control Plane Policing) で UDP 161 (SNMP) を極端に帯域制限した場合、大量のデータ要求（GETBULK）送信時に SNMP ポーリングのドロップが発生し、NMS 側で機器ダウンと誤判定されるリスクがあります。


---

## 🔄 他技術との関連

SNMP は他のセキュリティ・ネットワーク基盤技術と以下のように密接に関連しています。

* **Management Plane Protection (MPP)**:
  * インターフェイス単位で受信可能な管理プロトコル（SNMP, SSH, HTTPS）を制限。MPP が有効な場合、指定されていない外部インターフェイスからの SNMP UDP 161 パケットはインターフェイスレベルで破棄されます。
* **Control Plane Policing (CoPP)**:
  * ルータ/スイッチの CPU に届く SNMP (UDP 161) パケットをポリサーで帯域制御し、DoS 攻撃（SNMP Flooding）から制御プレーンを保護します。
* **VRF-Aware SNMP (OOB 監視)**:
  * 外部監視ネットワークをデータプレーンから分離するため、管理用 VRF（例: `Mgmt-intf`）に SNMP 処理をマッピング（`snmp-server host ... vrf Mgmt-intf`）し、セキュアな Out-of-Band 管理を実現します。
* **Cisco Stealthwatch (Secure Network Analytics)**:
  * NetFlow とともに SNMP を併用し、ネットワーク機器のインターフェイス名や IP-to-Host マッピング情報などのコンテキストデータを補足します。


---

## 🧩 比較表

### 1. SNMP バージョン比較 (v1 vs v2c vs v3)

| 比較項目 | SNMPv1 | SNMPv2c | SNMPv3 |
| --- | --- | --- | --- |
| **セキュリティモデル** | Community String (平文) | Community String (平文) | USM (User-based Security Model) |
| **暗号化 (Priv)** | 非対応 | 非対応 | DES, 3DES, AES-128, AES-192, AES-256 |
| **認証 (Auth)** | 非対応 (文字列一致のみ) | 非対応 (文字列一致のみ) | HMAC-MD5, HMAC-SHA-1, SHA-256, SHA-512 |
| **アクセス制御** | なし (コミュニティ一致) | なし (コミュニティ一致) | VACM (View-based Access Control Model) |
| **データ取得方式** | GET, GETNEXT | GET, GETNEXT, GETBULK | GET, GETNEXT, GETBULK, SET |
| **バルク転送** | 非対応 | 対応 (GETBULK) | 対応 (GETBULK) |
| **セキュリティレベル** | なし | なし | `noAuthNoPriv`, `authNoPriv`, `authPriv` |

### 2. イベント通知方式の比較 (Trap vs Inform)

| 項目 | SNMP Trap | SNMP Inform |
| --- | --- | --- |
| **プロトコル応答 (ACK)** | なし (Unacknowledged) | あり (Acknowledged: Inform-Response) |
| **信頼性** | ベストエフォート (パケット損失時再送なし) | 高信頼 (タイムアウト内に ACK がない場合再送) |
| **CPU / ネットワーク負荷** | 低い | やや高い (再送キュー管理および ACK 処理が必要) |
| **ユースケース** | 軽微なアラート、大量イベント発生時 | 重要なセキュリティアラート、システム障害通知 |
| **UDP ポート** | 162 | 162 |


---

## 💡 ベストプラクティス

1. **SNMPv1 / v2c の完全無効化**:
   * 本番環境では `snmp-server community` 設定を完全に削除し、暗号化が保証される `v3 authPriv` のみを許可します。
2. **強力な暗号・認証アルゴリズムの選定**:
   * 認証には `sha-256` 以上、暗号化には `aes 128` または `aes 256` を選択します（`des` や `md5` は非推奨）。
3. **VACM による最小権限原則の適用**:
   * NMS グループに付与する MIB View は、業務上必要な OID ツリーのみを `included` に設定し、敏感なルーティングテーブルや設定情報をマスクします。
4. **管理用 VRF と Loopback インターフェイスの利用**:
   * SNMP トラップの送信元アドレスを `Loopback0` に固定し、OOB 管理 VRF を使用してデータプレーンと分離します。
5. **CoPP による制御プレーン保護の導入**:
   * 未承認の IP アドレスからの UDP 161/162 通信を CoPP でドロップし、ルータ CPU の過負荷を防ぎます。


## 📝 ラボ学習・設定サンプル例

### 演習1: Cisco IOS-XE における SNMPv3 authPriv (SHA-256 / AES-128) と VACM MIB View 制限

#### 問題
R1 (Cisco IOS-XE) において、最高レベルのセキュリティ要件を満たす SNMPv3 監視環境を構築してください。

#### 要件
1. MIB View `RESTRICT-VIEW` を作成し、全 OID (`iso`) を基本許可した上で、IP ルーティングテーブル OID (`1.3.6.1.2.1.4.21`) を明示的に除外してください。
2. 標準 ACL `10` を作成し、NMS サーバ `10.1.1.100/32` からの通信のみを許可してください。
3. グループ `GRP-ADMIN` を SNMPv3 `priv` モードで作成し、上記 MIB View と ACL を割り当ててください。
4. ユーザー `NMS-ADMIN` を作成し、認証アルゴリズムに `sha-256`（パスワード: `AuthPass123!`）、暗号化アルゴリズムに `aes 128`（パスワード: `PrivPass123!`）を設定してください。

#### 設定例

```bash
! 1. MIB View の設定
snmp-server view RESTRICT-VIEW iso included
snmp-server view RESTRICT-VIEW 1.3.6.1.2.1.4.21 excluded

! 2. アクセス制御 ACL の設定
access-list 10 permit 10.1.1.100

! 3. SNMPv3 グループの設定
snmp-server group GRP-ADMIN v3 priv read RESTRICT-VIEW access 10

! 4. SNMPv3 ユーザーの設定
snmp-server user NMS-ADMIN GRP-ADMIN v3 auth sha-256 AuthPass123! priv aes 128 PrivPass123!
```

#### 検証方法

```bash
! ユーザー情報と暗号・認証アルゴリズムの確認
R1# show snmp user NMS-ADMIN

! MIB View の判定ルールの確認
R1# show snmp view RESTRICT-VIEW

! NMS (10.1.1.100) から snmpget を実行し、System 情報が取得でき、ipRouteTable が Access Denied になることを確認
! $ snmpget -v3 -l authPriv -u NMS-ADMIN -a SHA-256 -A AuthPass123! -x AES -X PrivPass123! 10.1.1.1 1.3.6.1.2.1.1.1.0
```


---

### 演習2: Cisco ASA における SNMPv3 (SHA / AES) の CLI 設定

#### 問題
ASA1 において、インサイドネットワーク上の NMS 管理端末（`172.16.10.50`）からの SNMPv3 監視を許可してください。

#### 要件
1. MIB View `ASA-LIMITED-VIEW` を定義し、`iso` 全体を許可してください。
2. グループ `ASA-MON-GRP` を SNMPv3 `priv` モードで定義してください。
3. ユーザー `ASA-ADMIN` を作成し、認証 `sha`（`CiscoAuth!23`）、暗号化 `aes 128`（`CiscoPriv!23`）を設定してください。
4. インサイドインターフェイス（`inside`）経由で `172.16.10.50` からの SNMPv3 通信を許可してください。

#### 設定例

```bash
! 1. View と Group の設定
snmp-server view ASA-LIMITED-VIEW iso included
snmp-server group ASA-MON-GRP v3 priv user-defined ASA-LIMITED-VIEW

! 2. User の設定
snmp-server user ASA-ADMIN ASA-MON-GRP v3 auth sha CiscoAuth!23 priv aes 128 CiscoPriv!23

! 3. ホスト許可とトラップ有効化
snmp-server host inside 172.16.10.50 version 3 ASA-ADMIN
snmp-server enable traps all
```

#### 検証方法

```bash
ASA1# show running-config snmp-server
ASA1# show snmp-server statistics

! NMS (172.16.10.50) からの SNMPv3 認証要求が成功することを確認
```


---

### 演習3: Cisco IOS-XE における SNMPv3 Inform メッセージの構成と再送タイマー調整

#### 問題
R1 から NMS (`10.2.2.50`) に対して、不確定な UDP トラップではなく、確実な配送が保証される SNMPv3 Inform メッセージを送信するように設定してください。

#### 要件
1. グループ `GRP-NOTIFY` および ユーザー `INF-USER` (v3 authPriv, sha-256: `PassAuth1!`, aes-128: `PassPriv1!`) を作成してください。
2. NMS ホスト `10.2.2.50` に対して SNMPv3 Inform メッセージを送信するように構成してください。
3. Inform の再送タイムアウトを `5秒`、最大再送回数を `3回` に設定してください。
4. トラップおよびインフォームの送信元インターフェイスを `Loopback0` に固定してください。

#### 設定例

```bash
snmp-server view VIEW-ALL iso included
snmp-server group GRP-NOTIFY v3 priv notify VIEW-ALL
snmp-server user INF-USER GRP-NOTIFY v3 auth sha-256 PassAuth1! priv aes 128 PassPriv1!

! Inform ホスト設定とタイマー調整
snmp-server host 10.2.2.50 version 3 priv INF-USER informs
snmp-server timeout 5
snmp-server packetsize 1400
snmp-server source-interface trap Loopback0
snmp-server enable traps config
```

#### 検証方法

```bash
! Inform 送信統計および ACK 確認ステータスの検証
R1# show snmp
R1# show snmp stats

! トラップ発生テスト (例: 構成変更)
R1(config)# hostname R1-TEST
R1(config)# hostname R1
! debug snmp packets を有効にし、Inform 要求と NMS からの Response(ACK) 受信ログを確認
```


---

### 演習4: Cisco FTD (FMC Managed) における Platform Settings を利用した SNMPv3 トラップおよび NMS 構成

#### 問題
FMC で管理されている FTD デバイスに対して、Platform Settings ポリシーを用いて SNMPv3 監視を有効化してください。

#### 要件
1. SNMPv3 ユーザー `FTD-SEC-USER` を作成（Auth: SHA-256, Priv: AES-128）。
2. NMS アドレス `192.168.200.10` からの SNMP 監視を `mgmt` インターフェイス経由で許可。
3. システム・インターフェイス関連のトラップを有効化。

#### 設定例 (FMC GUI 設定手順および生成コンフィグ要約)

**FMC GUI 手順**:
1. `Devices` > `Platform Settings` > FTD ポリシー選択。
2. `SNMP` > `Enable SNMP Server` にチェック。
3. `Users` タブ: Name `FTD-SEC-USER`, Authentication: `SHA-256`, Password: `AuthPassword123!`, Privacy: `AES-128`, Password: `PrivPassword123!`。
4. `Hosts` タブ: IP `192.168.200.10`, Interface `mgmt`, Version `3`, User `FTD-SEC-USER`。
5. デプロイの実行。

#### 検証方法

```bash
! FTD CLI に SSH 接続してバックエンド設定を確認
ftd# show running-config snmp-server
ftd# show snmp-server statistics
```


---

### 演習5: VACM 細粒度 MIB View 設定 (System / Interfaces のみ許可、Routing Table 遮断)

#### 問題
ルータ R2 において、サードパーティ製監視システムからのアクセスに対し、システム基本情報 (`1.3.6.1.2.1.1`) および インターフェイス情報 (`1.3.6.1.2.1.2`) のみにアクセスを限定し、それ以外の全 MIB へのアクセスを禁止してください。

#### 要件
1. MIB View `SAFE-VIEW` を作成し、デフォルトで全て除外 (`excluded`) とした上で、`sysUpTime`/`sysName` 等が含まれる `system` サブツリー (`1.3.6.1.2.1.1`) と `interfaces` サブツリー (`1.3.6.1.2.1.2`) のみを `included` に設定してください。
2. グループ `LIMITED-GRP` に本 View を割り当ててください。

#### 設定例

```bash
! 1. デフォルト全除外で特定の OID サブツリーのみ許可
snmp-server view SAFE-VIEW iso excluded
snmp-server view SAFE-VIEW 1.3.6.1.2.1.1 included
snmp-server view SAFE-VIEW 1.3.6.1.2.1.2 included

! 2. グループおよびユーザーの定義
snmp-server group LIMITED-GRP v3 priv read SAFE-VIEW
snmp-server user MONITOR-USER LIMITED-GRP v3 auth sha-256 Auth123456! priv aes 128 Priv123456!
```

#### 検証方法

```bash
R2# show snmp view SAFE-VIEW
! NMS から ifTable (1.3.6.1.2.1.2.2) が正常取得でき、ipAddrTable (1.3.6.1.2.1.4.20) が Access Denied となることを検証
```


### 演習6: SNMP EngineID の静的定義と NMS 同期設定

#### 問題
ルータ R1 の初期化や再起動時に SNMPv3 ユーザー情報および NMS との EngineID 同期が乱れるのを防ぐため、静的 EngineID を定義してください。

#### 要件
1. ローカル EngineID を静的に `1234567890ABCDEF12345678` に設定してください。
2. その後、グループ `STATIC-GRP` および ユーザー `ENGINE-USER` (v3 authPriv, sha-256/aes-128) を再構成してください。

#### 設定例

```bash
! 1. ローカル EngineID の静的設定 (事前に既存ユーザーがクリアされる点に注意)
snmp-server engineID local 1234567890ABCDEF12345678

! 2. ユーザーおよびグループの再設定
snmp-server view ALL-VIEW iso included
snmp-server group STATIC-GRP v3 priv read ALL-VIEW
snmp-server user ENGINE-USER STATIC-GRP v3 auth sha-256 AuthKey123! priv aes 128 PrivKey123!
```

#### 検証方法

```bash
R1# show snmp engineID
! 設定した EngineID (1234567890ABCDEF12345678) が表示されることを確認
```


---

### 演習7: CoPP による SNMP トラフィックの帯域制限と未許可 IP からの SNMP 遮断

#### 問題
ルータ R1 への DoS 攻撃対策として、Control Plane Policing (CoPP) を用い、許可された NMS (`10.1.1.100`) からの SNMP トラフィックを 256 kbps にポリシングし、それ以外の不審な IP からの SNMP 通信を完全にドロップしてください。

#### 要件
1. ACL `ACL-SNMP-ALLOWED` で `10.1.1.100` からの UDP 161 通信をマッチさせてください。
2. Class-map `CM-SNMP` を作成し、上記 ACL を指定してください。
3. Policy-map `PM-COPP` 内で `CM-SNMP` に対し `police 256000 conform-action transmit exceed-action drop` を適用してください。
4. Control-plane に本ポリシーを適用してください。

#### 設定例

```bash
! 1. 許可リスト ACL の作成
ip access-list extended ACL-SNMP-ALLOWED
 permit udp host 10.1.1.100 any eq snmp

! 2. Class-map および Policy-map の定義
class-map match-all CM-SNMP
 match access-group name ACL-SNMP-ALLOWED

policy-map PM-COPP
 class CM-SNMP
  police 256000 conform-action transmit exceed-action drop
 class class-default
  permit

! 3. Control-plane への適用
control-plane
 service-policy input PM-COPP
```

#### 検証方法

```bash
! ポリシーの合致およびドロップカウントの確認
R1# show policy-map control-plane
```


---

### 演習8: IPsec / DMVPN 経由での SNMP 監視とソースインターフェイス (Loopback0) 指定

#### 問題
DMVPN ハブ・スポーク環境において、スポークルータ R2 が IPsec トンネル経由で HQ の NMS (`10.254.1.100`) へ SNMP トラップを送信する際、ルーティングの変動に関わらず送信元 IP アドレスが常に `Loopback0` (`10.2.2.2`) になるよう設定してください。

#### 要件
1. トラップ/インフォーム送信元インターフェイスを `Loopback0` に固定してください。
2. SNMPv3 authPriv ユーザー `VPN-SNMP-USER` を設定してください。

#### 設定例

```bash
snmp-server view VPN-VIEW iso included
snmp-server group VPN-GRP v3 priv read VPN-VIEW
snmp-server user VPN-SNMP-USER VPN-GRP v3 auth sha-256 VpnAuthKey1! priv aes 128 VpnPrivKey1!

! ソースインターフェイスの固定
snmp-server source-interface trap Loopback0
snmp-server host 10.254.1.100 version 3 priv VPN-SNMP-USER
```

#### 検証方法

```bash
R2# show snmp
! "Logging source-interface: Loopback0" の表示を確認
```


---

### 演習9: SNMPv2c から SNMPv3 への完全移行 (レガシーコミュニティの安全な削除と監査)

#### 問題
ルータ R3 に残存している非推奨な SNMPv1/v2c コミュニティ文字列（`public`, `private` 等）を完全に削除し、安全な SNMPv3 authPriv 構成へ完全移行してください。

#### 要件
1. 既存のコミュニティ設定（`no snmp-server community ...`）を全て除去してください。
2. 新規に SNMPv3 グループ `SECURE-GRP` と ユーザー `MIGRATE-USER` を作成してください。

#### 設定例

```bash
! 1. 既存の平文コミュニティの削除
no snmp-server community public
no snmp-server community private
no snmp-server

! 2. 厳格な SNMPv3 の再設定
snmp-server view SEC-VIEW iso included
snmp-server group SECURE-GRP v3 priv read SEC-VIEW
snmp-server user MIGRATE-USER SECURE-GRP v3 auth sha-256 NewAuthPass1! priv aes 256 NewPrivPass1!
```

#### 検証方法

```bash
R3# show running-config | include snmp-server
! コミュニティ文字列が存在しないことを確認
```


---

### 演習10: VRF-Aware SNMPv3 構成 (Out-of-Band 管理 VRF 経由のポーリングとトラップ送信)

#### 問題
ルータ R1 において、データプレーンから隔離された Out-of-Band (OOB) 管理 VRF (`MGMT-VRF`) 経由でのみ SNMPv3 監視を許可し、かつトラップも同 VRF 経由で送信してください。

#### 要件
1. SNMP サーバーコンテキストに VRF `MGMT-VRF` をマッピングしてください。
2. NMS `192.168.99.50`（VRF `MGMT-VRF` 内）に対して SNMPv3 トラップを送信するように構成してください。

#### 設定例

```bash
! 1. 管理 VRF の作成とインターフェイス割当 (事前状態)
vrf definition MGMT-VRF
 address-family ipv4
exit-address-family

! 2. VRF 対応 SNMPv3 の設定
snmp-server view OOB-VIEW iso included
snmp-server group OOB-GRP v3 priv read OOB-VIEW
snmp-server user OOB-USER OOB-GRP v3 auth sha-256 OobAuth123! priv aes 128 OobPriv123!

! 3. VRF 明記による Host 設定およびソースインターフェイス指定
snmp-server host 192.168.99.50 vrf MGMT-VRF version 3 priv OOB-USER
snmp-server source-interface trap GigabitEthernet0/0
```

#### 検証方法

```bash
R1# show snmp
! VRF MGMT-VRF 経由で NMS へのパケットがカウントされることを確認
```


## ❓ 想定試験問題

### 問題 1 (コンフィグ読解・トラブルシューティング)
以下の R1 のコンフィグレーションにおいて、NMS (`10.1.1.100`) から SNMPv3 (`USER-A`) を使用して `sysName` や `ifTable` を取得しようとしたところ、「Access Denied (NoSuchInstance)」エラーが返されました。原因と修正コンフィグを答えてください。

```bash
snmp-server view VIEW-LIMITED 1.3.6.1.2.1.1 included
snmp-server group GRP-A v3 priv read VIEW-LIMITED
snmp-server user USER-A GRP-A v3 auth sha MyAuthPass123 priv aes 128 MyPrivPass123
```

* **解答・解説**:
  * **原因**: MIB View `VIEW-LIMITED` には `1.3.6.1.2.1.1` (`system` グループ) のみが `included` に設定されており、`ifTable` が含まれる `1.3.6.1.2.1.2` (`interfaces` グループ) が許可されていません。
  * **修正コマンド**:
    ```bash
    snmp-server view VIEW-LIMITED 1.3.6.1.2.1.2 included
    ```


---

### 問題 2 (トラブルシューティング)
NMS から Cisco ASA に対し SNMPv3 `authPriv` で問い合わせを行った際、`debug snmp header` にて認証失敗 (Authentication Failure) が記録されデータが取得できません。考えられる原因を 2 つ挙げてください。

* **解答・解説**:
  1. NMS 側と ASA 側で認証アルゴリズム（例: SHA-1 と MD5、または SHA-256 等）の指定が不一致である。
  2. NMS 側で設定した認証パスワードと ASA 側の `snmp-server user` のパスワードテキストが不一致である。


---

### 問題 3 (Design)
ネットワーク監視において、重要なアラート通知に SNMP Trap ではなく SNMP Inform を採用すべき理由と、その際の設計上の注意点を述べてください。

* **解答・解説**:
  * **理由**: SNMP Trap は UDP ベストエフォート送信でありパケットドロップ時に再送されませんが、SNMP Inform は NMS からのプロトコルレベルの ACK 応答 (`Inform-Response`) を要求し、応答がない場合に自動再送を行うため、通知の信頼性が高いため。
  * **注意点**: Inform は再送キューを保持するため、NMS ダウン時やネットワーク断時にルータのメモリおよび CPU 資源を消費します。そのため、再送タイムアウト (`snmp-server timeout`) や最大再送回数 (`snmp-server retransmit`) を適切に調整する必要があります。


---

### 問題 4 (実装・制限事項)
Cisco FTD (FMC 管理) 環境において、特定の内部インターフェイス経由でのみ SNMPv3 監視を許可し、かつコミュニティ文字列による SNMPv2c アクセスを完全に遮断するための設定手順を説明してください。

* **解答・解説**:
  1. FMC の `Platform Settings` にて対象の FTD ポリシーを開きます。
  2. `SNMP` タブで `Enable SNMP Server` を有効化します。
  3. `Community Maps` / `Community Strings` に何も登録されていないこと（または空）を確認し、SNMPv1/v2c を無効化します。
  4. `Users` タブで SNMPv3 ユーザー (`authPriv` 指定) を作成します。
  5. `Hosts` タブにて、指定の内部インターフェイス（例: `inside`）と NMS IP アドレスを指定し、バージョン `v3` を選択して保存・デプロイします。


---

### 問題 5 (セキュリティ実装)
Control Plane Policing (CoPP) を運用中のルータにおいて、SNMP 監視パケットが意図せずドロップされる問題が発生しています。原因を特定し、セキュリティを維持したまま解決するためのコンフィグのポイントを説明してください。

* **解答・解説**:
  * **原因**: CoPP の `class-default` または汎用管理クラスにおいて UDP ポート 161/162 のポリサー帯域幅制限（Police Rate）が低すぎるか、あるいは NMS の IP アドレスが許可 ACL に含まれておらずドロップされている。
  * **解決策**: 正確な NMS の IP アドレスおよび UDP ポート 161 を特定する分類 ACL (`ip access-list extended ACL-SNMP`) を作成し、専用の Class-map で帯域（例: 256 kbps〜1 Mbps）を個別に確保（`police` 適用）した上で、許可された IP からの SNMP リクエストを保護するように CoPP ポリシーを設計します。


---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび各種技術資料一覧です。

* Cisco IOS-XE Network Management Configuration Guide - Configuring SNMP
* Cisco ASA Series General Operations CLI Configuration Guide - Monitoring SNMP
* Cisco Management Center Device Configuration Guide - Platform Settings SNMP
* Cisco Technical Notes - SNMPv3 Configuration Examples and Troubleshooting
* Cisco Live Presentation - Securing the Management Plane on Cisco IOS / IOS-XE
* Cisco Live Presentation - Overview of SNMPv3 and Network Security Models
* Cisco Validated Design (CVD) - Secure Management and Monitoring


---

## 📝 補足（Notes）

### SNMPv3 組み合わせチェックリスト

| 認証アルゴリズム (`auth`) | 暗号化アルゴリズム (`priv`) | セキュリティ評価 | 推奨度 |
| --- | --- | --- | --- |
| **MD5** | **DES** | **低い** | ❌ 非推奨 |
| **SHA-1** | **3DES** | **低い** | ❌ 非推奨 |
| **SHA-1** | **AES-128** | **標準** | ⚠️ 移行用 |
| **SHA-256 / SHA-512** | **AES-128 / AES-256** | **高い** | ✅ **推奨** |

### コマンド設定時の注意点まとめ
* Cisco IOS-XE では `snmp-server user` コマンドで設定したパスワードテキストは、ランニングコンフィグ上では直接表示されず、ハッシュ値として保持されます。パスワードを変更する場合はユーザーの再作成が必要です。
* `snmp-server host` コマンドで指定するユーザーは、事前に `snmp-server user` で定義されている必要があります。


