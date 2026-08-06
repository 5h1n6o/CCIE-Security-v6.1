# 1.9 Policies and rules for traffic control on Cisco ASA

Cisco ASA における**トラフィック制御のポリシーとルール**は、単純な L3/L4 のアクセス制御リスト (ACL) に留まらず、オブジェクト指向の管理、Modular Policy Framework (MPF) による詳細なインスペクション、およびパケット処理の順序（Order of Operations）を包含する概念です。CCIE Security ラボ試験では、複雑な要件を最小限のルールで、かつパフォーマンスに配慮して実装する能力が問われます。

---

## 📘 概要

*   **機能概要**: パケットの通過可否を決定するアクセスルール、アプリケーション層まで踏み込んで検査するインスペクション、および帯域制限や接続数制限を行う QoS ポリシーを組み合わせてトラフィックを制御します。
*   **利用目的**: セキュリティ境界の確立、特定プロトコルの動作正常化、リソース枯渇の防止。
*   **どのような場面で利用するか**: 内部から外部へのインターネットアクセス許可、外部から DMZ サーバーへの公開、音声トラフィック (SIP/H.323) の透過、DoS 攻撃からの保護。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **ACL 方式** | Extended (推奨), Standard, EtherType (L2), WebType (Clientless VPN)。 |
| **ポリシー管理** | **Modular Policy Framework (MPF)**: Class-map, Policy-map, Service-policy。 |
| **メリット** | ステートフル検査による高いセキュリティ、オブジェクトグループによる管理の簡素化。 |
| **デメリット** | 設定ミス（順序や NAT との不整合）により正当な通信が遮断されやすい。 |
| **対応機種** | 全 ASA モデル、ASAv。 |
| **主要要素** | Security Level, Object Groups, Inspection Engine, Packet Filtering。 |
| **設計上の注意** | 常に **Most Specific** なルールを上に配置し、オブジェクトを多用すること。 |

---

## 🏗 動作原理

ASA のトラフィック制御は、**「最初のパケット」**が到着した際のフロー確立プロセスに基づきます。

```text
Incoming Packet
   ↓
[ 1. ACL Check ] --- (If denied -> Drop)
   ↓
[ 2. NAT/XLATE ] --- (Global/Static translations)
   ↓
[ 3. Conn Lookup ] --- (Existing session?)
   ↓
[ 4. MPF Inspection ] --- (Deep Packet Inspection: e.g., HTTP, ICMP)
   ↓
[ 5. Egress ACL ]
   ↓
Outgoing Packet
```

---

## ⚙ 動作シーケンス

1.  **パケット受信**: インターフェイスでパケットを受信し、既存のコネクションテーブルにマッチするか確認します。
2.  **アクセスルールの評価**: 新規通信の場合、インターフェイスに適用された `access-group` を上から順にチェックします。
3.  **NAT 処理**: アクセスルールにマッチした後、宛先または送信元 IP の変換（NAT）が行われます。
4.  **インスペクション (MPF)**: `inspect` ルールに基づき、プロトコルの整合性を確認します。例えば ICMP インスペクションが有効な場合、ASA は戻りの Echo Reply を許可するための動的エントリを作成します。
5.  **フロー作成**: すべてのチェックを通過すると、コネクションテーブルにフローが登録され、以降のパケットは高速パスで処理されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **順序の罠**: ACL は常に「最初の一致」が優先されます。広い範囲を許可するルールの前に、特定のホストを拒否するルールを置く必要があります。
*   **ICMP インスペクション**: デフォルトでは ASA は ICMP をインスペクションしません。`inspect icmp` が設定されていない場合、戻りのトラフィック用に明示的な戻り ACL が必要になります。
*   **Object Groups の活用**: 「10台のサーバーに特定の3つのポートを許可せよ」という課題に対し、30行の ACL ではなく、オブジェクトグループを使用して 1 行で実装する能力が評価されます。
*   **Identity Firewall (IDFW)**: ユーザー名や AD グループを条件にしたルールの作成。
*   **Troubleshooting**: 通信が止まった際、`packet-tracer` を使用して、どの「Phase」でドロップしているか（ACL なのか、NAT なのか、インスペクションなのか）を特定するスキルが必須です。

---

## 🛠 設定方法

### 1. オブジェクトグループと ACL の作成
```bash
object-group network SERVERS
 network-object host 10.1.1.10
 network-object host 10.1.1.11
!
object-group service WEB_PORTS tcp
 port-object eq 80
 port-object eq 443
!
access-list OUTSIDE_IN extended permit tcp any object-group SERVERS object-group WEB_PORTS
access-group OUTSIDE_IN in interface outside
```

### 2. MPF による ICMP インスペクションの有効化
```bash
policy-map global_policy
 class inspection_default
  inspect icmp
!
service-policy global_policy global
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **ACL マッチ数の確認** | <code>show access-list [NAME]</code> |
| **パケット処理パスの追跡** | <code>packet-tracer input inside tcp 1.1.1.1 1234 2.2.2.2 80 detailed</code> |
| **パケットキャプチャの実行** | <code>capture CAP_IN interface inside match ip any any</code> |
| **サービスポリシーの状態確認** | <code>show service-policy</code> |
| **コネクションテーブルの表示** | <code>show conn</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| Ping が通らない | ICMP インスペクション未設定 | <code>show service-policy</code> | <code>inspect icmp</code> を追加。 |
| 期待した ACL にマッチしない | 順序の間違い、または NAT 前後の IP 指定ミス | <code>packet-tracer</code> | NAT 前の「実 IP」で ACL を記述しているか確認。 |
| 特定アプリ (FTP/SIP) が動かない | Inspection がヘッダーを不正と判断 | <code>show service-policy inspect</code> | インスペクション設定の微調整または無効化。 |
| 管理アクセスが拒否される | <code>http</code>/<code>ssh</code> コマンドの不足 | <code>show run http</code> | <code>http [network] [mask] [interface]</code> を追加。 |

---

## ⚠ 制限事項

*   **暗号化トラフィック**: SSL/TLS で暗号化されたパケットの中身をインスペクションするには、ASA 側で SSL 復号の設定が必要です。
*   **Security Level**: 同一セキュリティレベルのインターフェイス間通信は、デフォルトで拒否されます（`same-security-traffic permit inter-interface` が必要）。
*   **IPv6 ACL**: IPv4 とは別の ACL オブジェクトとして管理する必要があります。

---

## 🔄 他技術との関連

*   **NAT**: 8.3 以降の ASA では、ACL は通常、変換後の宛先 IP ではなく、**「リアル IP」**を宛先として記述します。
*   **Routing**: トラフィックが ACL を通過しても、出口インターフェイスへの経路がなければドロップされます。
*   **AnyConnect VPN**: VPN 経由のアクセスには `vpn-filter` や `webtype` ACL が使用されます。

---

## 🧩 比較表

### Access Control vs Service Policy (MPF)

| 特徴 | Access Control (ACL) | Service Policy (MPF) |
| :--- | :--- | :--- |
| **レイヤー** | L3/L4 | L4 - L7 |
| **処理内容** | 許可 (Permit) / 拒否 (Deny) | インスペクション、QoS、接続制限 |
| **適用単位** | インターフェイス単位 | インターフェイスまたはグローバル |
| **主な用途** | IP アドレスとポートのフィルタリング | プロトコルの詳細検査 (Deep Packet Inspection) |

---

## 💡 ベストプラクティス

1.  **Any の排除**: 可能な限り `any` は使用せず、ネットワークオブジェクトを使用します。
2.  **グローバルポリシーの活用**: ICMP などの共通インスペクションは `global_policy` で一括適用します。
3.  **ロギングの最適化**: 頻繁に発生する通信（DNS 等）には `interval` を設定し、Syslog サーバーの負荷を抑えます。
4.  **Packet-Tracer での事前検証**: ルールをデプロイする前に、必ず `packet-tracer` でシミュレーションを行います。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な Web サーバー公開
*   **要件**: 外部から DMZ サーバー (192.168.1.100) への HTTPS アクセスを許可せよ。
*   **設定**: `access-list EXT extended permit tcp any host 192.168.1.100 eq 443` -> `access-group EXT in interface outside`

### 2. オブジェクトグループによる集約
*   **要件**: Inside 全体から特定の 3 つの DNS サーバーへの UDP/53 を 1 行で許可せよ。
*   **設定**: `object-group network DNS_SRVS`, `access-list ... object-group DNS_SRVS eq 53`

### 3. ICMP インスペクションのトラブルシュート
*   **課題**: Inside から Outside への Ping が通らない。ACL 以外で解決せよ。
*   **解決**: `policy-map global_policy` 内で `inspect icmp` を有効化。

### 4. 特定ホストへの接続数制限 (MPF)
*   **要件**: 外部からの攻撃を防ぐため、宛先 10.1.1.1 への同時接続数を 100 に制限せよ。
*   **設定**: `class-map` でホストを定義 -> `policy-map` で `set connection conn-max 100`。

### 5. 時間帯指定 ACL
*   **要件**: 就業時間内 (9:00-17:00) のみ SNS サイトへの通信を許可せよ。
*   **設定**: `time-range WORK_TIME`, `periodic daily 9:00 to 17:00`, `access-list ... time-range WORK_TIME`

### 6. 管理プレーンの保護
*   **要件**: 特定の管理セグメント (10.99.1.0/24) からのみ ASA への SSH を許可せよ。
*   **設定**: `ssh 10.99.1.0 255.255.255.0 inside`

### 7. IPv6 トラフィックのフィルタリング
*   **要件**: 不正な IPv6 パケットをアグレッシブに破棄せよ。
*   **設定**: `ipv6 spd mode aggressive`

### 8. FQDN ベースの ACL
*   **要件**: `www.cisco.com` への通信のみを許可せさよ（IP が変動するサイト）。
*   **設定**: `object network CISCO_URL`, `fqdn www.cisco.com`, `access-list ... object CISCO_URL`

### 9. 透過型 (Transparent) モードでの L2 制御
*   **要件**: ARP 以外の全非 IP トラフィックを遮断せよ。
*   **設定**: `access-list L2 ethertype deny any`

### 10. パケットトレースによるデバッグ
*   **課題**: 通信が止まっている。パケットが ACL、NAT、インスペクションのどこで落ちているか特定せよ。
*   **コマンド**: `packet-tracer input ... detailed`

---

## ❓ 想定試験問題

1.  **トラブルシュート**: Inside から Outside への通信を ACL で `permit ip any any` にしたが Ping が通らない。原因を 2 つ挙げよ。
    *   **回答**: 1. ICMP インスペクションが未設定。 2. 戻りのルートが ASA 上に存在しない。
2.  **Design**: 数百のホストがある環境で、管理負荷を最小にしつつアクセス制御を更新しやすくする ASA の機能は？
    *   **回答**: Object Groups (Network/Service/Security Group Tags)。
3.  **実装**: 特定のプロトコル（例: SQL）が非標準ポートで動作している場合に、ASA にそのトラフィックを正規の SQL 通信として検査させるための設定手順は？
    *   **回答**: `class-map` で非標準ポートをマッチさせ、`policy-map` 内で該当する `inspect` エンジンを紐付ける。
4.  **コンフィグ読解**: `packet-tracer` の出力で `Action: DROP`, `Drop-reason: (acl-drop)` と出た。次に行うべき確認は？
    *   **回答**: 対象インターフェイスに適用されている `access-list` の順序と、Implicit Deny にマッチしていないかを確認する。
5.  **Design**: ルーターと ASA がある環境で、DoS 攻撃（SYN Flood）を最も効率的に緩和するために ASA で設定すべき MPF の項目は？
    *   **回答**: `set connection embryonic-conn-max` (Half-open 接続の制限)。

---

## 🔗 参考リソース

*   **Cisco Live (BRKSEC-3020)**: [Troubleshooting ASA/FTD Policy Matching](https://www.ciscolive.com/)
*   **Configuration Guide**: [Cisco ASA Series CLI Configuration Guide - Configuring Access Rules](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/firewall/asa-94-firewall-config/access-rules.html)
*   **Technical Note**: [ASA Packet Capturing and Tracer Feature](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/113685-asa-threat-detection-00.html)

---

## 📝 **補足（Notes）**  

*   **学習メモ**: トラフィック制御の「主役」は ACL ですが、「監督」は MPF です。ACL で通しても MPF がプロトコルアノマリと判断すればパケットは落ちます。
*   **図解**: 常に `packet-tracer` の出力を脳内でイメージしてください。Phase 1 から 10 まで、どのゲートを通過しているかを意識するのが CCIE 合格の鍵です。
*   **注意点**: ラボ試験では、設定後に **`show access-list` でヒットカウンタが増えているか**を必ず確認してください。カウンタが 0 なら、パケットはそのルールまで届いていません。
