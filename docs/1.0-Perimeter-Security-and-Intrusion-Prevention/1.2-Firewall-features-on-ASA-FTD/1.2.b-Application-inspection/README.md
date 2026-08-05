---
layout: default
title: 1.2.b-Application-inspection
nav_order: 2
parent: 1.2-Firewall-features-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.2.b Application inspection

Cisco ASAおよびFirepower Threat Defense (FTD) における**アプリケーションインスペクション（Application Inspection）**は、レイヤ4（トランスポート層）を超えてレイヤ7（アプリケーション層）までパケットを深く検査する機能です。CCIE Securityラボ試験では、ステートフルなパケットフィルタリングだけでは対応できない複雑なプロトコル（FTP、SIP、DNS等）の正常な通信を、NATやACL環境下で維持・制御する能力が問われます。

---

## 📘 概要

*   **機能概要**: パケットのペイロードをスキャンし、プロトコル固有の規則に準拠しているかを確認します。また、埋め込まれたIPアドレスの書き換えや、必要に応じてダイナミックなデータチャネルを自動的に解放します。
*   **利用目的**:
    *   **ダイナミックポートの制御**: FTPやVoIP（SIP/H.323）のように、制御通信の中で動的にデータポートを決定するプロトコルの通信を許可します。
    *   **プロトコル準拠の強制**: DNSのメッセージ長制限や、HTTPメソッドの制限など、攻撃に悪用されやすいプロトコルの振る舞いを制限します。
    *   **NATの補助**: アプリケーションのペイロード内に記述されたプライベートIPを、NAT後のパブリックIPに整合させる（Fix-up）ために使用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | ステートフルインスペクションをL7まで拡張。MPF（Modular Policy Framework）により実装。 |
| **用途** | ICMPの双方向通信維持、FTP/SQLNet等のNAT透過、不正なHTTPトラフィックの遮断。 |
| **メリット** | セキュリティの向上、複雑なプロトコルの自動透過、NAT環境での不整合解消。 |
| **デメリット** | 深い検査によるCPU負荷の増大。暗号化された通信には別途復号（SSL Decryption）が必要。 |
| **対応機種** | 全てのASAおよびFTD。FTDではLINAエンジン（ASAベース）とSnortの両方で役割を分担。 |
| **設計上の注意** | デフォルトで多くのインスペクションが有効だが、ICMPなどはASAで手動有効化が必要。 |

---

## 🏗 動作原理

アプリケーションインスペクションは、ASAの**Modular Policy Framework (MPF)** アーキテクチャの一部として動作します。

```text
Client
   ↓
Ingress Interface (受信)
   ↓
ACL/NAT Checks
   ↓
Modular Policy Framework (MPF)
   1. Class Map (どのトラフィックを検査するか)
   2. Policy Map (どのインスペクションエンジンを適用するか)
   3. Service Policy (どのIF、またはGlobalに適用するか)
   ↓
[ Inspection Engine ] --- (プロトコル解析、ダイナミックポートの開放、ペイロード書き換え)
   ↓
Egress Interface (送出)
```

---

## ⚙ 動作シーケンス

1.  **クラス分類**: クラスマップ（Class Map）が対象のトラフィック（例：TCP 21番ポート）を識別します。
2.  **インスペクション呼び出し**: ポリシーマップ（Policy Map）内で `inspect` コマンドが実行され、対応するアプリケーション用エンジンが呼び出されます。
3.  **コネクション監視**: エンジンは制御チャネルを監視します（例：FTPの `PORT` または `PASV` コマンド）。
4.  **動的ピンホール（Pin-hole）作成**: 制御チャネル内でネゴシエーションされたデータ転送用のポート番号を読み取り、一時的にそのポートのACLをバイパスして通信を許可します。
5.  **セッション終了**: 通信完了後、動的に開けられたポート（ピンホール）は自動的に閉じられます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **ICMP Inspectionの欠如**: ASAはデフォルトでICMPインスペクションが無効です。これにより、InsideからのPingに対する戻りのパケット（Echo Reply）がOutside ACLで拒否されるトラブルシュート問題が頻出します。
*   **デフォルト設定の変更**: ラボ要件で「DNSのメッセージサイズ制限を緩和せよ」や「特定のHTTPメソッドをブロックせよ」といった、デフォルトインスペクションポリシーのカスタマイズが指示されます。

### ラボ試験で設定させられそうな内容
*   **FTPインスペクション**: NAT環境下でActive/Passive FTPを正常に動作させる設定。
*   **VoIP（SIP/SCCP）の修正**: NATによって壊れたペイロード内のIPアドレスを修正する。
*   **カスタムインスペクションクラス**: デフォルトの `inspection_default` クラス以外の特定のトラフィックに対して個別のインスペクションを適用する。

### よくある設定ミス
*   **Service Policyの重複**: 同一インターフェイスに複数のポリシーを適用しようとして失敗する（GlobalとInterface各1つまで）。
*   **インスペクション順序**: ACLで先にドロップされている場合、インスペクションは実行されません。

### showコマンド/debugログの読み取り
*   `packet-tracer`: インスペクションフェーズでパケットがどのように処理されるかを確認する最重要ツールです。
*   `show service-policy inspect`: 各インスペクションエンジンでどれだけのパケットがヒットし、エラーでドロップされたかを確認できます。

---

## 🛠 設定方法

### ASA (CLI) - ICMPインスペクションの有効化
```bash
# クラスマップの定義（デフォルトで存在することが多い）
class-map inspection_default
 match default-inspection-traffic
!
# ポリシーマップでの設定
policy-map global_policy
 class inspection_default
  inspect icmp
!
# インターフェイス（またはグローバル）への適用
service-policy global_policy global
```

### FTD (FMC管理) - インスペクションの設定
1.  **Policies > Access Control**: 対象のポリシーを編集。
2.  **Advancedタブ**: 「Transport/Network Layer Preprocessor Settings」を選択。
3.  **Inline Normalization**: 不正なパケットフラグメント等のインスペクションを制御。
4.  **Service Policy**: ASAのMPFに相当する設定は、FTD 7.x以降のFMC GUI上で「Service Policy」として個別に定義・適用可能。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **インスペクション統計の表示** | <code>show service-policy inspect icmp</code> |
| **パケットパスのトレース** | <code>packet-tracer input inside icmp 10.1.1.1 8 0 8.8.8.8 detailed</code> |
| **現在開いているピンホールの確認** | <code>show conn</code> (インスペクションによって開いた接続には `inspect` フラグが付く) |
| **デフォルトポリシーの確認** | <code>show run policy-map</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| InsideからOutsideへPingが通らない | ICMPインスペクションが無効 | <code>packet-tracer</code> | <code>inspect icmp</code>をポリシーに追加。 |
| FTPのデータ転送（ls/get）が失敗 | NAT環境でFTPインスペクション未設定 | <code>show service-policy inspect ftp</code> | <code>inspect ftp</code>を有効にする。 |
| DNS応答が大きすぎてドロップされる | デフォルトのDNS最大サイズ制限 | <code>show service-policy inspect dns</code> | DNSマップで <code>max-app-payload</code>を拡張。 |
| ESMTP（TLS）使用時に通信断 | インスペクションが暗号化を解読不能 | <code>debug esmtp</code> | 明示的にESMTPインスペクションを無効化するか調整。 |

---

## ⚠ 制限事項

*   **暗号化パケット**: SSL/TLSで暗号化されたペイロードの中身をインスペクションすることはできません。これを行うには、SSL Decryption（FTDのSSL Policy等）が必要です。
*   **パフォーマンス**: 深いプロトコル検査はスループットに影響を与えます。大量のトラフィックを処理する場合、不要なインスペクション（ESMTP等）をオフにする設計が必要です。
*   **非標準ポート**: プロトコルが非標準ポート（例：TCP 8080のHTTP）で動いている場合、デフォルトの `match default-inspection-traffic` では認識されず、個別のクラスマップ設定が必要です。

---

## 🔄 他技術との関連

*   **Access Control**: ACLはレイヤ4までのポートを静的に制御しますが、インスペクションはそれを動的に補完します。
*   **NAT**: ペイロードにIPアドレスを埋め込むプロトコル（FTP, SIP, SQLNet）において、NAT変換の一貫性を保つために必須です。
*   **High Availability**: インスペクションによって作成された動的ステート情報は、フェイルオーバーユニット間で同期されます。
*   **Snort (FTD)**: FTDでは、ASAのインスペクション機能の一部がSnortエンジンにオフロードされ、より高度な脅威検知と統合されています。

---

## 🧩 比較表

### ASA MPF Inspection vs FTD Snort Inspection

| 機能 | ASA MPF (LINA) | FTD Snort Engine |
| :--- | :--- | :--- |
| **主な処理レイヤ** | L4 - L7 (基本的なプロトコル) | L7 (深いアプリケーション識別) |
| **設定場所** | CLI (Class/Policy Map) | FMC GUI (ACP / Intrusion Policy) |
| **VoIP対応** | 非常に得意 (SIP, H.323) | IPS機能の一部として対応 |
| **主な役割** | 固定ポートの透過・NAT修正 | 脆弱性保護、アプリ識別(AVC) |

---

## 💡 ベストプラクティス

*   **ICMPの原則有効化**: ネットワークの診断（Traceroute含む）を容易にするため、ラボ試験では最初に `inspect icmp` を有効にすることが推奨されます。
*   **不要なインスペクションの無効化**: デフォルトの `inspection_default` クラスから、環境で使用しないプロトコル（例：H323, Skinny）を削除することでリソースを節約します。
*   **パケットトレーサーの活用**: 疎通問題が発生した際、ACLだけでなく「Inspect」フェーズでドロップしていないかを常に確認します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的なICMPインスペクション
*   **要件**: InsideからのPingに対する戻り通信を、ACLを追加せずに許可せよ。
*   **設定例**: `policy-map global_policy; class inspection_default; inspect icmp`。

### 2. 非標準ポートでのHTTPインスペクション
*   **要件**: ポート8080で動作するHTTP通信に対してもインスペクションを適用せよ。
*   **設定例**:
    ```bash
    class-map HTTP_8080
     match port tcp eq 8080
    policy-map global_policy
     class HTTP_8080
      inspect http
    ```

### 3. DNSメッセージサイズの拡張
*   **要件**: 512バイトを超える大きなDNS応答を許可せよ。
*   **設定例**:
    ```bash
    policy-map type inspect dns DNS_MAP
     parameters
      max-app-payload 1200
    policy-map global_policy
     class inspection_default
      inspect dns DNS_MAP
    ```

### 4. FTPインスペクション（Active/Passive対応）
*   **要件**: NAT後のクライアントがFTPサーバからデータを取得できるようにせよ。
*   **設定例**: `policy-map global_policy; class inspection_default; inspect ftp`.

### 5. HTTPメソッドの制限（GET/POSTのみ許可）
*   **要件**: HTTPの脆弱性対策として、CONNECTメソッド等をブロックせよ。
*   **設定例**: `policy-map type inspect http HTTP_MAP; methods range get post; class inspection_default; inspect http HTTP_MAP`.

### 6. SIPインスペクションとNAT修正
*   **要件**: VoIP端末がNAT越えで通話できるように設定せよ。
*   **設定例**: `policy-map global_policy; class inspection_default; inspect sip`.

### 7. ICMP Errorインスペクション
*   **要件**: Tracerouteの結果がNAT後のIPではなく、正しい送信元を表示するようにせよ。
*   **設定例**: `policy-map global_policy; class inspection_default; inspect icmp error`.

### 8. SQLNetインスペクション
*   **要件**: Oracleデータベースへのダイナミックポート通信を許可せよ。
*   **設定例**: `inspect sqlnet`.

### 9. 特定のインターフェイスのみインスペクションを無効化
*   **要件**: DMZインターフェイスに限り、ESMTPインスペクションを無効にせよ。
*   **設定例**: `policy-map DMZ_POLICY; class inspection_default; no inspect esmtp; service-policy DMZ_POLICY interface dmz`.

### 10. インスペクションによるパケットドロップの確認
*   **要件**: インスペクションによって拒否されたパケットを特定せよ。
*   **検証**: `show service-policy inspect` の `drop` カウントを確認。

---

## ❓ 想定試験問題

1.  **問題**: ASAで内部ホストから外部へのPingは通るが、Tracerouteの応答が返ってこない。原因と対策は？
    *   **解答**: デフォルトでは `inspect icmp` が無効であり、かつ `inspect icmp error` も無効であるため。対策として `inspect icmp error` をポリシーに追加する。
2.  **問題**: 1つのインターフェイスに対して、個別のService PolicyとGlobal Service Policyを同時に適用した場合、どちらが優先されるか？
    *   **解答**: インターフェイス固有のポリシー（Interface Service Policy）が優先され、Globalポリシーは無視される。
3.  **問題**: インスペクション設定を行ったが、`show service-policy` でヒットカウントが増えない。何を確認すべきか？
    *   **解答**: トラフィックがクラスマップの条件に一致しているか、および上位のACLで通信が許可されているかを確認する。
4.  **問題**: FTDにおいて、ASAのインスペクション機能（LINA）とSnortエンジンの優先順位はどうなっているか？
    *   **解答**: 一般的に、LINAエンジンが先にパケットを処理し、その後Snortエンジンへ転送される（Flowによっては同時並行）。
5.  **問題**: 暗号化されたTLSトラフィックに対してESMTPインスペクションがエラーを出す。回避策は？
    *   **解答**: `inspect esmtp` を無効化するか、SSL復号設定を行う。

---

## 🔗 参考リソース

*   **Configuration Guide**:
    *   [Cisco ASA Series Firewall CLI Configuration Guide, 9.4 - Getting Started with MPF](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/firewall/asa-94-firewall-config/inspect-mpf.html)
    *   [Cisco Firepower Threat Defense Configuration Guide for FMC - Service Policies](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/service_policies.html)
*   **Cisco Live (Videos & Slides)**:
    *   BRKSEC-3020: Troubleshooting Firewall Threat Defense (FTD)
    *   BRKSEC-2021: Firepower Threat Defense - Packet Flow and Troubleshooting
*   **Command Reference**:
    *   [Cisco ASA Series Command Reference - inspect](https://www.cisco.com/c/en/us/td/docs/security/asa/command-reference/i-l/cmdref1/i1.html)
*   **Technical Notes**:
    *   [ASA: Application Inspection Configuration and Troubleshooting (Cisco Support)](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/113264-asa-icmp-inspection-00.html)
    *   FTP Inspection Through a Firewall - TechNotes

---

## 📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  
