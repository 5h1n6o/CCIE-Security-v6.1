---
layout: default
title: 1.3.a-Application-awareness
nav_order: 1
parent: 1.3-Security-features-on-IOS
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.3.a Application awareness

Cisco IOS/IOS XEにおける**アプリケーション・アウェアネス（Application Awareness）**は、主に **NBAR2 (Next Generation Network-Based Application Recognition)** 技術を使用して、パケットのペイロードを深くスキャン（DPI: Deep Packet Inspection）し、レイヤ7（アプリケーション層）のプロトコルを識別する機能です。CCIE Securityラボ試験では、これを **ZBFW (Zone-Based Firewall)** と組み合わせて、特定のアプリケーション（例：SNS、P2P）のみを許可または遮断する実装が求められます。

---

## 📘 概要

*   **機能概要**: ポート番号（L4）だけでなく、パケットの内容（L7）に基づいてトラフィックを識別します。暗号化されたトラフィックでも、証明書のSNI（Server Name Indication）などから識別を試みます。
*   **利用目的**:
    *   **トラフィックの可視化**: ネットワーク上を流れるアプリケーションの統計取得。
    *   **ポリシー制御**: 不正なアプリケーションのブロック、または重要なビジネスアプリの優先（QoS連携）。
*   **利用場面**: ZBFWでのアプリケーションフィルタリング、AVC (Application Visibility and Control) によるNetFlow統計の強化など。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要技術** | **NBAR2** (Network-Based Application Recognition 2) |
| **識別方法** | PDLM (Protocol Description Language Modules) によるシグネチャ照合 |
| **用途** | ZBFWでのL7フィルタリング、QoSのマーキング、AVCによる可視化 |
| **メリット** | 非標準ポートを使用するアプリやWebベースのアプリを正確に識別可能 |
| **デメリット** | 深い検査によるCPU負荷の増大（ハードウェアアクセラレーションがない場合） |
| **対応OS** | Cisco IOS, IOS XE (ISR/ASRシリーズ等) |
| **設計上の注意点** | 暗号化（TLS 1.3等）により可視性が制限されるため、ETA等の併用を検討 |

---

## 🏗 動作原理

NBAR2は、パケットの最初の数数バイトをスキャンし、事前に定義されたシグネチャ（PDLM）と照合します。

```text
Client (HTTPS/P2P)
   ↓
Ingress Interface
   ↓
[ NBAR2 Engine ] --- (PDLMによるシグネチャ照合)
   ↓
[ Classification ] --- (例: "facebook" or "bittorrent" としてマーク)
   ↓
[ Policy Application ]
   ├─ ZBFW (Drop/Inspect)
   └─ QoS (Mark/Police)
   ↓
Egress Interface
```

---

## ⚙ 動作シーケンス

1.  **プロトコル・ディスカバリ**: インターフェイスで有効にすると、通過するトラフィックをリアルタイムで分析・分類します。
2.  **マッチング**: `class-map` 内で `match protocol` コマンドが実行されると、NBARエンジンがそのプロトコルに一致するパケットを識別します。
3.  **アクション適用**: マッチしたトラフィックに対し、`policy-map` で定義された `drop`（遮断）、`inspect`（ステートフル検査）、または `pass` などのアクションを実行します。
4.  **統計送信 (NetFlow)**: AVC機能と連携している場合、識別されたアプリケーション情報をFlowコレクタへ送信します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **ZBFWとの統合**: 単なる `match port` ではなく、`match protocol http url "*facebook*"` や `match protocol bittorrent` といった高度なマッチングが問われます。
*   **プロトコル・ディスカバリの有効化**: インターフェイスで `ip nbar protocol-discovery` を有効にして統計を確認する手順。

### ラボ試験で設定させられそうな内容
*   **P2Pアプリケーションのブロック**: ZBFWを使用して、社内から外部へのBitTorrentトラフィックを完全に遮断する。
*   **カスタムシグネチャの作成**: 特定のURLキーワード（例：ログインページ）に基づいて通信を制御する。
*   **AVC統計の確認**: 特定のインターフェイスでどのアプリが帯域を消費しているか `show` コマンドで判断する。

### よくある設定ミス
*   **NBARの有効化忘れ**: `class-map` で `match protocol` を使っているのに、肝心の `ip nbar protocol-discovery` がインターフェイスに入っていない（IOS XEのバージョンにより自動有効化されるが、手動設定が確実）。
*   **正規表現の誤り**: HTTP URLマッチングでワイルドカード `*` の使い方が誤っており、意図したサイトにマッチしない。

### showコマンドから状態を判断
*   `show ip nbar protocol-discovery`: インターフェイスごとのアプリ統計を確認。
*   `show policy-map type inspect ...`: ZBFWでのマッチングとドロップ数を確認。

---

## 🛠 設定方法

### 1. アプリケーション識別（NBAR2）の有効化
```bash
interface GigabitEthernet1
 ip nbar protocol-discovery
```

### 2. ZBFWでのアプリケーション制御（BitTorrent遮断例）
```bash
# クラスマップの定義
class-map type inspect match-any L7_BLOCK_CLASS
 match protocol bittorrent
 match protocol gnutella

# ポリシーマップでのドロップ設定
policy-map type inspect INSIDE_TO_OUTSIDE_POLICY
 class type inspect L7_BLOCK_CLASS
  drop log
 class class-default
  inspect
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **アプリ統計の表示** | <code>show ip nbar protocol-discovery</code> |
| **ZBFW統計の確認** | <code>show policy-map type inspect zone-pair</code> |
| **特定のアプリの定義確認** | <code>show ip nbar port-map [protocol]</code> |
| **NBARエンジン状態** | <code>show ip nbar version</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| アプリが識別されない (unknown) | シグネチャが古い | <code>show ip nbar version</code> で最新のPDLMが適用されているか確認。 |
| 特定のURLがマッチしない | 正規表現のミス | <code>match protocol http url</code> の文字列設定を見直す。 |
| 通信遅延が発生する | CPU過負荷 | <code>show process cpu sorted</code> で NBAR 関連プロセスの負荷を確認。 |
| 遮断すべきアプリが通過する | 階層化ポリシーの不整合 | Class-mapの <code>match-any / match-all</code> 設定を確認。 |

---

## ⚠ 制限事項

*   **暗号化の壁**: ECH (Encrypted Client Hell) 等が普及すると、SNIベースの識別ができなくなる可能性があります。
*   **パフォーマンス**: 大量のL7マッチングはルータのCPUリソースを消費するため、コアネットワークよりもアクセス層やブランチ拠点での適用が一般的です。
*   **プロトコル数**: 識別可能なアプリ数はデバイスのソフトウェアバージョン（シグネチャパック）に依存します。

---

## 🔄 他技術との関連

*   **ZBFW (Zone-Based Firewall)**: アプリケーション層でのアクセス制御を実現するための主要なプラットフォームです。
*   **QoS (Quality of Service)**: アプリケーションを識別し、音声トラフィックを優先（PQ）したり、YouTubeを制限（Policing）したりします。
*   **NetFlow (AVC)**: 識別したアプリ情報をNetFlow v9/IPFIXのオプションフィールドとしてコレクタに送信します。

---

## 🧩 比較表

### L4 Access-List vs NBAR2 (L7 Awareness)

| 比較項目 | L4 ACL (Legacy) | NBAR2 (Application Awareness) |
| :--- | :--- | :--- |
| **識別基準** | ポート番号 (TCP 80等) | ペイロード、URL、証明書情報 |
| **動的ポート対応** | 不可（または非常に困難） | **可能**（シグネチャで追随） |
| **設定の柔軟性** | 低い | 非常に高い |
| **リソース消費** | 低 | 高（DPI処理） |

---

## 💡 ベストプラクティス

*   **シグネチャの定期更新**: `ip nbar protocol-pack` を使用して、最新のアプリケーション定義をロードします。
*   **プロトコル・ディスカバリの活用**: ポリシーを適用する前に、まず `protocol-discovery` で現状のトラフィック傾向を把握します。
*   **階層化アプローチ**: 明確なポート番号（L4）で防げるものはACLで防ぎ、回避型アプリのみNBAR2でスキャンすることで効率化を図ります。

---

## 📝 ラボ学習・設定サンプル例

※以下の設定例は、Cisco IOS XE 環境を前提としています。

### 1. NBAR統計の有効化と確認
*   **要件**: 全てのインターフェイスでアプリケーション統計を収集せよ。
*   **設定**: `interface range Gi1-2; ip nbar protocol-discovery`

### 2. ZBFWによるSkypeの制限
*   **要件**: 内部から外部へのSkype使用を禁止し、ログに記録せよ。
*   **設定**:
```bash
class-map type inspect match-any SKYPE_CLASS
 match protocol skype
policy-map type inspect PRIV_TO_PUB
 class type inspect SKYPE_CLASS
  drop log
```

### 3. URLキーワードベースのフィルタリング
*   **要件**: HTTP URL内に "gaming" を含む通信を遮断せよ。
*   **設定**: `class-map type inspect match-any GAME_CLASS; match protocol http url "*gaming*"`

### 4. QoSとの連携（アプリ優先）
*   **要件**: Webexトラフィックを識別し、DSCP AF41をマーキングせよ。
*   **設定**:
```bash
class-map match-any WEBEX_CLASS
 match protocol webex-media
policy-map APP_MARKING
 class WEBEX_CLASS
  set dscp af41
```

### 5. NBAR2カスタムプロトコル定義
*   **要件**: TCP 9999 ポートを使用する独自アプリ "MY_APP" を定義せよ。
*   **設定**: `ip nbar custom MY_APP tcp 9999`

### 6. 特定サブネットに対するNBAR制御
*   **要件**: ゲスト用ネットワーク (10.0.0.0/24) のみP2Pを遮断せよ。
*   **設定**: ZBFWの `match-all` クラスで `access-group` と `match protocol` を併用。

### 7. NetFlow AVCの構成
*   **要件**: 識別されたアプリ情報をNetFlowエクスポータへ送信せよ。
*   **設定**: `flow monitor AVC_MONITOR; record netflow-original; statistics application`

### 8. NBAR2シグネチャのバージョン確認
*   **コマンド**: `show ip nbar protocol-pack active`

### 9. 暗号化通信の識別確認 (SNI)
*   **要件**: HTTPSトラフィックから "google-services" を識別できるか確認。
*   **検証**: `show ip nbar protocol-discovery` の出力で確認。

### 10. 全トラフィックの許可（Inspect）
*   **要件**: アプリ識別を使用しつつ、問題ないトラフィックはステートフルに許可せよ。
*   **設定**: `policy-map ... class class-default; inspect`

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `match protocol http host "*cisco.com*"` という設定がある。これは何を識別しようとしているか？
    *   **正解**: HTTPリクエストヘッダ内のHostフィールドに "cisco.com" を含む通信を識別する。
2.  **トラブルシュート**: ZBFWで `drop` アクションを設定したクラスのマッチカウントが増えない。原因は？
    *   **正解**: トラフィックがそれより前の優先順位のルールにマッチしている、またはインターフェイスでNBARエンジンが正しく動作していない可能性がある。
3.  **Design**: 拠点のISRルータで、帯域を逼迫させている特定のストリーミングアプリを特定したい。どの機能を有効にすべきか？
    *   **正解**: NBAR2 Protocol Discovery。
4.  **実装**: ZBFWにおいて、L4のポート指定（match port）とNBAR2のプロトコル指定（match protocol）を1つのクラスマップに混在させることは可能か？
    *   **正解**: はい、`match-any` または `match-all` を適切に使用することで可能です。
5.  **動作シーケンス**: NBAR2はパケットのどの部分を検査するか？
    *   **正解**: パケットのペイロード（L7データ）をDPI（Deep Packet Inspection）により検査する。

---

## 🔗 参考リソース

*   **Configuration Guide**:
    *   [Cisco IOS XE NBAR2 Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_nbar/configuration/xe-16/qos-nbar-xe-16-book/nbar-prot-discovery-xe.html)
    *   [Zone-Based Policy Firewall Application Inspection Configuration](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_zbf/configuration/xe-16/sec-data-zbf-xe-16-book/sec-zbf-app-inspec.html)
*   **Technical Notes**:
    *   [NBAR2 Protocol Pack Release Notes](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_nbar/prot_packs/nbar-prot-packs.html)
    *   [Using NBAR to Block Multiple Applications](https://www.cisco.com/c/en/us/support/docs/security/ios-firewall/98628-nbar-block-app.html)
*   **CVD (Cisco Validated Design)**:
    *   [Cisco Application Visibility and Control (AVC) Design Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Dec2016/CVD-ApplicationVisibilityandControlDesignGuide-DEC16.html)
*   **Command Reference**:
    *   [Cisco IOS IP NBAR Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_nbar/command/nbar-cr-book.html)  

---

## 📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

---
