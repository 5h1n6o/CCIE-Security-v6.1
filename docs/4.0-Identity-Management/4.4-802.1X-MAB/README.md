---
layout: default
title: 4.4-802.1X-MAB
nav_order: 4
parent: 4.0-Identity-Management
---

# 4.4 AAA for network access with 802.1X and MAB using Cisco ISE

Cisco ISE (Identity Services Engine) を使用した **802.1X** および **MAB (MAC Authentication Bypass)** は、企業ネットワークにおけるエンドポイント接続制御の核心です。802.1X は証明書や資格情報に基づく強力な認証を提供し、MAB はプリンタやカメラなどのサプリカントを持たないデバイスに対するフォールバック認証を提供します。

---

## 📘 概要

*   **機能概要**: ネットワークアクセスデバイス（NAD：スイッチや WLC）が、接続されたデバイスの ID を確認し、ISE 上のポリシーに基づいてアクセス許可、VLAN 割り当て、ACL 適用などを動的に行う機能です。
*   **利用目的**: 「誰が」「何を」「どこで」「いつ」「どのように」接続しているかに基づく、コンテキストを意識したセキュリティ制御（Identity-Based Networking）の実現。
*   **利用場面**:
    *   **802.1X**: PC、スマートフォン、タブレット等の証明書またはユーザー認証。
    *   **MAB**: IP 電話、ネットワークプリンタ、IoT デバイス等の MAC アドレスベースの識別。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要プロトコル** | IEEE 802.1X (EAP), RADIUS (UDP 1812/1813), HTTP/HTTPS。 |
| **コンポーネント** | サプリカント (端末), オーセンティケータ (NAD), 認証サーバ (ISE)。 |
| **802.1X EAP メソッド** | EAP-TLS (証明書), EAP-PEAP (MSCHAPv2), EAP-FAST。 |
| **MAB の仕組み** | オーセンティケータが端末の送信元 MAC を RADIUS ユーザー名として ISE に送る。 |
| **認可属性 (Result)** | VLAN ID, dACL, SGT (Security Group Tag), リダイレクト URL。 |
| **ホストモード** | Single-host, Multi-host, Multi-domain (IP Phone + PC), Multi-auth。 |

---

## 🏗 動作原理

ネットワークアクセスの AAA は、以下の 3 つのロール間でパケットが交換されることで機能します。

```text
  [ Supplicant ]           [ Authenticator ]           [ Auth Server ]
     (Client)                (Switch/WLC)                  (ISE)
        |                         |                          |
        |--- (1) EAPoL Start ---->|                          |
        |                         |--- (2) RADIUS Access --->|
        |<-- (3) EAP Request -----|        Request           |
        |                         |                          |
        |--- (4) EAP Response --->|                          |
        |                         |--- (5) RADIUS Access --->|
        |                         |        Challenge         |
        |          ......(EAP Negotiation)......             |
        |                         |<-- (6) RADIUS Access ----|
        |<-- (7) EAP Success -----|        Accept (+dACL/VLAN)|
        |                         |                          |
        |                         |--- (8) RADIUS Accounting-|
        |                         |        Start             |
```

---

## ⚙ 動作シーケンス

1.  **初期化**: リンクアップによりオーセンティケータが認証を開始、またはサプリカントが EAPoL-Start を送信。
2.  **認証**: EAP メソッド（TLS, PEAP 等）が選択され、資格情報（証明書やパスワード）が RADIUS パケットにラップされて ISE へ送られる。
3.  **認可**: ISE がアイデンティティを確認後、認可ポリシー（Authorization Policy）に合致する結果（dACL, VLAN, SGT 等）を `Access-Accept` に含めて返信。
4.  **強制**: スイッチがポートにポリシーを適用し、通信を許可する。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **IBNS 2.0 (Cisco Common Classification Policy)**: 従来の `authentication` コマンドではなく、`policy-map type control` を使用した最新の構成方法が問われます。
*   **認証の優先順位とフェイルオーバー**: 802.1X を優先し、失敗またはタイムアウト後に MAB を実行する順序の構成。
*   **dACL (Downloadable ACL) の構文**: ISE で ACL を作成し、スイッチ側で `ip device-tracking` (または `device-tracking upgrade-cli`) が正しく有効化されていることを確認する能力。
*   **CoA (Change of Authorization)**: ポスチャ（検疫）や属性変更後に、ISE からスイッチへセッション再認証を送る設定 (`aaa server radius dynamic-author`)。
*   **ホストモードの選択**: IP Phone と PC を同一ポートで認証する `multi-domain` モードの理解。
*   **デバッグログの読み取り**: `debug dot1x`, `debug authentication` の出力から、RADIUS 共有シークレットの不一致や EAP タイムアウトを特定する能力。

---

## 🛠 設定方法

### 1. スイッチ：RADIUS サーバーと CoA の基本設定
```bash
aaa new-model
!
radius server ISE
 address ipv4 10.1.1.100 auth-port 1812 acct-port 1813
 key cisco123
!
aaa group server radius ISE_GRP
 server name ISE
!
! CoA (Change of Authorization) 受信設定
aaa server radius dynamic-author
 client 10.1.1.100 server-key cisco123
!
! dACLを適用するために必須
device-tracking-policy TRACK-ALL
 tracking enable
!
interface GigabitEthernet1/0/1
 device-tracking attach-policy TRACK-ALL
```

### 2. スイッチ：IBNS 2.0 による 802.1X + MAB 構成
```bash
class-map type control match-all DOT1X-EVENT
 match method dot1x
!
policy-map type control PORT-POLICY
 class type control DOT1X-EVENT seq 10
  authenticate using dot1x priority 10
 class type control always seq 20
  authenticate using mab priority 20
!
interface GigabitEthernet1/0/1
 access-session port-control auto
 access-session host-mode multi-domain
 service-policy type control input PORT-POLICY
 dot1x pae authenticator
 mab
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **セッション詳細の確認** | <code>show access-session interface [int] details</code> |
| **適用された ACL (dACL) の確認** | <code>show ip access-lists interface [int]</code> |
| **RADIUS サーバの状態** | <code>show aaa servers</code> |
| **DOT1X 詳細デバッグ** | <code>debug dot1x all</code> |
| **認証マネージャデバッグ** | <code>debug authentication all</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 認証がタイムアウトする | 共有シークレットの不一致、または IP 到達不可 | <code>ping</code> で疎通確認。<code>aaa servers</code> で応答数を確認。 |
| dACL が適用されない | デバイストラッキング設定の不足 | <code>show device-tracking database</code> で IP 学習を確認。 |
| IP Phone 接続時に PC が通信不可 | ホストモードの設定ミス | <code>access-session host-mode multi-domain</code> を設定。 |
| MAB が開始されない | 802.1X のタイムアウト待ちが長い | <code>dot1x timeout tx-period</code> を短縮する。 |

---

## ⚠ 制限事項

*   **ライセンス依存**: ISE の Advantage または Premier ライセンスが一部の高度な認可機能に必要。
*   **dACL サイズ**: ハードウェア（スイッチ）により、動的にダウンロードできる ACL のエントリ数に上限がある。
*   **FlexConnect の制限**: 無線 FlexConnect ローカルスイッチング環境では dACL が使えない場合がある。

---

## 🔄 他技術との関連

*   **2.6 Cisco TrustSec (SGT)**: 認可の結果として SGT を割り当て、マイクロセグメンテーションを実現。
*   **3.4.e DHCP Snooping**: ISE がエンドポイントの IP 情報を取得するための重要なソース。
*   **4.1 ISE Personas**: PSN ノードが RADIUS 認証を直接処理する役割を担う。

---

## 🧩 比較表

### 802.1X vs MAB vs CWA

| 特徴 | 802.1X | MAB | CWA (Central Web Auth) |
| :--- | :--- | :--- | :--- |
| **認証の強さ** | **最高** (証明書/ユーザー) | **低** (MAC偽装可能) | 中 (資格情報) |
| **サプリカント** | 必須 | 不要 | 不要 (ブラウザ) |
| **主な用途** | 社内 PC, モバイル | プリンタ, IoT, 電話 | ゲスト, BYOD |
| **フロー** | EAPoL | MAC 送信 | L2 許可 -> HTTP リダイレクト |

---

## 💡 ベストプラクティス

1.  **Critical Auth (Fail-Open)**: RADIUS サーバ（ISE）が全滅した場合に備え、ポートを特定の「緊急 VLAN」に配置する設定を行う。
2.  **Low-Impact Mode**: 認証前から限定的な通信（DHCP/DNS/PXE等）を許可する ACL をポートに適用し、認証成功時に dACL で上書きする手法。
3.  **RADIUS ソース IP**: `ip radius source-interface` を使用し、ISE 側での管理を容易にする。

---

## 📝 ラボ学習・設定サンプル例

### 1. 802.1X + MAB フェイルオーバー (IBNS 2.0)
*   **問題**: ポート Gi1/0/1 で DOT1X を 1 回試行し、失敗したら即座に MAB を実行せよ。
*   **設定**: `policy-map` 内の `class type control always` で `authenticate using mab` を設定。

### 2. Multi-Domain Auth (IP Phone 連携)
*   **要件**: ポートに Cisco IP Phone を接続。音声は VLAN 110、PC データは ISE 認可に従え。
*   **設定**: `access-session host-mode multi-domain` と `switchport voice vlan 110`。

### 3. Dynamic VLAN Assignment
*   **要件**: ユーザーが `Sales` グループなら VLAN 10、`HR` なら VLAN 20 を ISE から割り当てよ。
*   **検証**: `show access-session details` で "Vlan: 10" 等が表示されるか確認。

### 4. dACL (Downloadable ACL) の実装
*   **問題**: ISE で `PERMIT_HTTP_ONLY` という ACL 名で `permit tcp any any eq 80` を作成し適用。
*   **設定**: スイッチで `device-tracking` と `aaa authorization network` を有効化。

### 5. Critical VLAN (Auth Server 不達時)
*   **要件**: ISE がダウンした場合、ポートを VLAN 999 (Fail-Open) に配置せよ。
*   **設定**: `service-template CRITICAL-TEMP` で `vlan 999` を定義し、ポリシーマップで `action vlan 999` を指定。

### 6. MAB 優先順位の調整
*   **問題**: セキュリティ上、特定の MAC 以外は 802.1X をスキップして MAB だけを実行せよ。

### 7. Change of Authorization (CoA) の検証
*   **要件**: ポスチャ準拠後、セッションを切断せずに ACL を更新せよ。
*   **設定**: `aaa server radius dynamic-author`。

### 8. Multi-Auth モードでの複数端末接続
*   **要件**: 1 ポートにハブを介して接続された全端末を個別に認証せよ。
*   **設定**: `access-session host-mode multi-auth`。

### 9. リダイレクト ACL (CWA 用)
*   **操作**: Web 認証用にトラフィックを ISE へ飛ばすためのローカル ACL を定義せよ。

### 10. SGT (Scalable Group Tag) の動的割当
*   **要件**: 認証成功時、SGT 10 を付与してタグベースの制御を行え。
*   **設定**: 認可プロファイルで SGT 属性を選択。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `access-session host-mode multi-host` が設定されたポートに 2 台目のデバイスが接続された場合の挙動は？
    *   **回答**: 最初のデバイスが認証済みであれば、2 台目のトラフィックも**認証なしで全て通過**する。
2.  **トラブルシュート**: ISE から `Access-Accept` が来ているのに VLAN が変わらない。スイッチの何を確認すべきか？
    *   **回答**: `show vlan` で指定された VLAN が**スイッチ内に存在（作成）**しているかを確認。
3.  **Design**: IBNS 2.0 を使用する際、特定の順序で認証メソッドを試行するための構成要素は？
    *   **回答**: `policy-map type control` と `class-map type control`。
4.  **実装**: dACL を正常に動作させるためにスイッチ上で有効化が必要な L3 トラッキング機能は？
    *   **回答**: **Device Tracking** (旧 IPDT)。
5.  **Design**: EAP-TLS と EAP-PEAP の最大の違いは？
    *   **回答**: EAP-TLS は**クライアント証明書**が必要（相互認証）、EAP-PEAP はユーザー名/パスワード（サーバー証明書のみ）で動作する。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [ポートベース認証の概要](https://www.cisco.com/c/en/us/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31.html)
*   **Cisco Validated Design (CVD)**: [ISE Wired Access Control Design Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/ise-deployment-guide.html)
*   **Cisco Live (BRKSEC-2022)**: [Identity-Based Networking Services 2.0](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「NAD = スイッチ」「Auth Server = ISE」「Supplicant = 端末」の 3 者関係を常に意識。
*   **図解**: EAPoL はオーディオケーブルのように L2 上で直接やり取りされるが、RADIUS は IP パケットとしてルーティングされる点を理解。
*   **注意点**: ラボ試験では `aaa server radius dynamic-author` を入れ忘れると、CWA やポスチャのフローが完結しないため致命的な失点になります。
