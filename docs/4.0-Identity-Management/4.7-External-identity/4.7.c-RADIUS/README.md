---
layout: default
title: 4.7.c-RADIUS
nav_order: 3
parent: 4.7-External-identity
grand_parent: 4.0-Identity-Management
---

# 4.7.c External RADIUS

Cisco ISEにおける **External RADIUS** 統合は、ISE自体が認証を行うのではなく、受信したRADIUSリクエストを別のRADIUSサーバ（外部のISE、FreeRADIUS、またはISPのサーバなど）に転送する **RADIUSプロキシ** 機能として主に利用されます。これは、組織間の提携（Eduroamなど）や、企業の合併・買収に伴う認証基盤の統合フェーズにおいて非常に重要な役割を果たします。

---

## 📘 概要

*   **機能概要**: ISEをRADIUSプロキシとして構成し、特定の条件（User-Nameのドメイン部分など）に基づいて、外部のRADIUSサーバへリクエストをリレーする機能です。
*   **利用目的**: 自組織で管理していない資格情報（外部ユーザー）を持つユーザーのネットワークアクセスを許可するため。
*   **利用場面**:
    *   **Eduroam**: 教育機関同士でWi-Fiを相互利用する場合。
    *   **企業合併**: 認証サーバがまだ統合されていない別部門のユーザーを認証する場合。
    *   **MSP/ISP**: サービスプロバイダーが顧客のRADIUSサーバへ認証を委託する場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要役割** | RADIUSプロキシ (Proxy Server)。 |
| **転送条件** | 通常、RADIUS属性（User-Nameのレルム/ドメイン）やNAS-IPで判定。 |
| **通信プロトコル** | UDP 1812 (Authentication), 1813 (Accounting)。 |
| **共有シークレット** | ISEと外部RADIUSサーバ間で個別の共有キーが必要。 |
| **属性の保持** | プロキシ時に特定の属性を追加、削除、または置換可能。 |
| **ライセンス** | ISE Base/Essentialライセンスで基本機能は利用可能。 |

---

## 🏗 動作原理

ISEは、クライアント（NAD）からは「認証サーバ」として見え、外部RADIUSサーバからは「RADIUSクライアント（NAS）」として振る舞います。

```text
Supplicant (user@external.com)
      ↓
NAD (Switch/WLC)
      ↓ (RADIUS Access-Request)
Cisco ISE (Proxy)
      ↓ (Check Policy: if domain == external.com)
      ↓ (Forward Access-Request)
External RADIUS Server
      ↓ (Verify Identity)
      ↑ (Access-Accept)
Cisco ISE (Proxy)
      ↑ (Relay Access-Accept)
NAD (Switch/WLC)
```

---

## ⚙ 動作シーケンス

1.  **リクエスト受信**: NADからRADIUS Access-Requestを受信。
2.  **ポリシー照合**: `Policy Set` または `Proxy Service` 設定により、外部転送が必要か判断。
3.  **プロキシ実行**: ISEは送信元IPを自身のIPに書き換え、外部サーバへリクエストを転送。
4.  **レスポンス待機**: 外部サーバからの応答を待機。タイムアウト時は再試行またはフォールバック。
5.  **結果のリレー**: 外部サーバからの Access-Accept/Reject をNADへ転送。この際、ISEで認可属性を追加することも可能。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **共有シークレットの不一致**: 最も多いトラブルポイントです。ISE側で定義した「外部RADIUSサーバ用キー」と、相手サーバ側の「ISE用クライアントキー」が一致していることを確認してください。
*   **ポート番号の設定**: 外部サーバが非標準ポート（例：1645/1646）を使用している場合、ISE側でも正確に合わせる必要があります。
*   **レルム（Realm）のパース**: `user@example.com` の `example.com` 部分を抽出して転送先を決める設定を習得してください。
*   **タイムアウトとリトライ**: 外部サーバへのWAN回線が不安定な場合を想定し、タイムアウト値をデフォルトより長く設定するなどの最適化が問われる可能性があります。
*   **VSA（ベンダー固有属性）の扱い**: 外部サーバから返されたVSAを、ISEがNADへ透過的に渡すか、あるいはISE側で上書きするかの挙動を理解してください。

---

## 🛠 設定方法

### 1. 外部RADIUSサーバの定義 (GUI)
1.  **Administration > Identity Management > External Identity Sources > External RADIUS Server** に移動。
2.  **Add** をクリックし、外部サーバの IP、共有キー、ポート番号（通常1812/1813）を入力。

### 2. RADIUSサーバシーケンスの作成
1.  **Administration > Identity Management > RADIUS Server Sequences** に移動。
2.  作成した外部サーバをリストに追加。
3.  `Local Accounting` を記録するかどうかを選択。

### 3. ポリシーセットでの適用
1.  **Policy > Policy Sets** で、特定の条件（例：`RADIUS:User-Name ENDS_WITH @external.com`）を設定。
2.  `Allowed Protocols` の横にある **Server Sequence** で、作成したシーケンスを選択。

---

## 🔍 検証コマンド

| 目的 | コマンド / 手法 |
| :--- | :--- |
| **ISE 認証ログの確認** | **Operations > RADIUS > Live Logs** で転送状態を確認。 |
| **外部サーバへの到達性** | ISE CLI から <code>ping [External_Server_IP]</code>。 |
| **転送プロセスのデバッグ** | <code>debug runtime-proxy</code> (ISE CLI) ※高負荷注意。 |
| **NAD側での統計確認** | <code>show radius statistics</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| **タイムアウト (No Response)** | IP疎通不可、またはUDP 1812の遮断 | 中間のFirewallでRADIUS通信が許可されているか確認。 |
| **Authentication Failed** | 共有キーの不一致 | 両方のサーバでキーを再投入する。 |
| **ISEでパケットが捨てられる** | 転送条件（ドメイン名等）の誤り | Live Log の `Steps` でプロキシ条件に合致しているか確認。 |
| **外部サーバでRejectされる** | ISEのIPが相手に登録されていない | 外部サーバ側にISEの管理IPがクライアントとして登録されているか確認。 |

---

## ⚠ 制限事項

*   **遅延の影響**: プロキシ構成はRTT（往復遅延）の影響を強く受けます。遅延が大きすぎるとNAD側でタイムアウトが発生します。
*   **EAPチェーンの断絶**: EAP-TLSなどの高度な認証では、フラグメンテーションの処理がプロキシ経由で複雑になる場合があります。
*   **属性の制限**: 外部サーバが返すVSAの種類によっては、ISEが解釈できず破棄される場合があります。

---

## 🔄 他技術との関連

*   **4.1 ISE Scalability**: 大規模環境では、特定のPSNのみをプロキシ専用として構成することがあります。
*   **3.10 Cisco DNAC**: DNACが管理するネットワークにおいて、外部AAAサーバとしてISEを登録し、さらにISEが外部へプロキシする構成。
*   **IPsec VPN**: リモートアクセスVPNの認証をISEが受け、外部RADIUSへプロキシする構成。

---

## 🧩 比較表

### ISE 内部認証 vs External RADIUS Proxy

| 特徴 | ISE 内部認証 (AD/LDAP) | External RADIUS Proxy |
| :--- | :--- | :--- |
| **資格情報の場所** | ISEが参照可能なDB (AD/Internal) | ISEが直接触れない外部サーバ |
| **認証プロトコル** | ISEがEAPを終端 | ISEはRADIUSを転送（リレー） |
| **制御の細かさ** | 非常に細かいポリシー適用が可能 | 外部サーバの応答に依存 |
| **主な用途** | 社内従業員、管理デバイス | ゲスト、提携組織ユーザー |

---

## 💡 ベストプラクティス

1.  **フォールバックの検討**: 外部サーバがダウンした場合に備え、特定の「ゲストVLAN」へ落とすなどのローカル認可を構成します。
2.  **ログの局所化**: アカウンティングログは外部に送るだけでなく、トラブルシュートのためにISE内部（MnT）にもコピーを残すよう設定します。
3.  **セキュリティ**: WAN経由でプロキシを行う場合は、RADIUS over DTLS または IPsec トンネルを使用してRADIUSパケット自体を保護します。

---

## 📝 ラボ学習・設定サンプル例

### 1. ドメインベースのプロキシ構成
*   **要件**: `@lab.local` で終わるユーザーを 10.2.2.2 の RADIUS サーバへ転送せよ。
*   **設定**: Proxy Condition `RADIUS:User-Name MATCHES .*@lab\.local$`.

### 2. 共有シークレットの設定
*   **操作**: 外部サーバ `EXT_AAA` に対しキー `Cisco123` を設定。

### 3. UDPポートのカスタマイズ
*   **要件**: 外部サーバが 1645 ポートを使用している場合の設定。

### 4. アカウンティングのコピー
*   **要件**: 認証は外部で行い、アカウンティングはISEと外部の両方に記録せよ。

### 5. タイムアウト値の調整
*   **要件**: 外部サーバの応答が遅いため、タイムアウトを 10 秒、リトライを 2 回に設定せよ。

### 6. 属性の置換 (Attribute Manipulation)
*   **要件**: 外部へ送る前に `Called-Station-ID` を削除せよ。

### 7. RADIUSサーバシーケンスの冗長化
*   **要件**: Server A がダウンしたら Server B へ転送せよ。

### 8. ローカル認証とのISS併用
*   **要件**: シーケンス内で「Internal Users」を確認後、見つからなければ外部プロキシを実行せよ。

### 9. 特定のNAS-IPに基づく転送
*   **要件**: 特定のスイッチ（10.1.1.5）からの要求のみ外部へプロキシせよ。

### 10. 外部VSAの透過転送確認
*   **検証**: 外部サーバから返された `cisco-av-pair` がそのままNADに届いているかパケットキャプチャで確認。

---

## ❓ 想定試験問題

1.  **Design**: ISEをRADIUSプロキシとして使用する場合、ISE自体で認証ポリシー（Authentication Policy）を詳細に構成する必要があるか？
    *   **回答**: いいえ。プロキシの場合は `RADIUS Server Sequence` をポリシーセット全体に適用するため、個別の認証ルールよりも上位で転送が決定されます。
2.  **トラブルシュート**: プロキシ設定後、Live Log に `RADIUS-Proxy: No reply from remote RADIUS server` と表示される。考えられる原因は？
    *   **回答**: ネットワーク疎通の問題、または外部サーバ側でISEのIPが **Authorized Client** として登録されていない。
3.  **コンフィグ読解**: `RADIUS Server Sequence` 設定内の `Continue to next server on No Response` が有効な場合の挙動は？
    *   **回答**: 最初の外部サーバから応答がない場合のみ、リスト内の次のサーバへリクエストを転送する。
4.  **実装**: ユーザー名にドメインが含まれていないリクエストを、デフォルトで外部RADIUSへ送るようにするには？
    *   **回答**: ポリシーセットの条件を `Any` にし、Server Sequence を外部RADIUSに設定する。
5.  **Design**: RADIUSプロキシ環境で、ユーザーの接続時間をISE側で制限したい。どの機能を使うべきか？
    *   **回答**: 外部から返ってきた Access-Accept に対し、ISEの **Authorization Profile** で `Session-Timeout` 属性を付加してNADへ送る。

---

## 🔗 参考リソース

*   **Configuration Guide**: [Cisco ISE 3.1 - Configuring External RADIUS Servers](https://www.cisco.com/c/en/us/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31/b_ise_admin_guide_31_chapter_010.html#ID552)
*   **Technical Note**: [ISE RADIUS Proxy Configuration Example](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/116140-configure-ise-00.html)
*   **Cisco Live**: [BRKSEC-3432 - ISE Tips and Tricks](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「ISEは郵便局の転送窓口」とイメージしてください。宛先（ドメイン）を見て、自分の担当外なら正しい配送先（外部サーバ）へ送り直します。
*   **図解**: プロキシ時は、RADIUSパケット内の `Identifier` 番号をISEが管理し、戻ってきたパケットを正しいNADへ紐付け直すステートフルな動作をしています。
*   **注意点**: ラボ試験では **RADIUS共有キーのタイポ** が致命的です。必ず `show aaa servers` 等で、送信数と受信数が一致しているか（応答があるか）を最初に見る癖をつけてください。
