---
layout: default
title: 1.6.d-AVC
nav_order: 1
parent: 1.6-NGFW-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.6.d AVC (Application Visibility and Control)

Cisco Secure Firewall (FTD) における **AVC (Application Visibility and Control)** は、ポート番号やプロトコルといった従来の L3/L4 情報のみならず、パケットのペイロードを深く検査することで、具体的にどのアプリケーション（例：Facebook, BitTorrent, Office 365）が通信されているかを識別し、制御する機能です。CCIE Security ラボ試験では、何千ものアプリケーションを動的に識別し、ビジネス要件に基づいたきめ細かなアクセス制御（例：Facebook の閲覧は許可するが投稿は禁止する）を構成する能力が問われます。

---

## 📘 概要

*   **機能概要**: パケットの最初の数パケット（First-packet identification）や、証明書の SNI (Server Name Indication)、シグネチャベースのディープパケットインスペクションを用いてアプリケーションを特定します。
*   **利用目的**: 非標準ポートを使用するアプリの識別、シャドー IT の抑制、および帯域幅を消費する不要なアプリの制限。
*   **利用場面**: 「標準的な HTTP (80/tcp) 通信の中で、YouTube 動画の閲覧のみを特定して遮断したい」といった、サービス単位での制御が必要な環境で利用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **識別手法** | シグネチャ、ディテクター、VDB (Vulnerability Database) による更新。 |
| **主要要素** | Application Filters, Individual Applications, Application Tags, Risk/Business Relevance。 |
| **メリット** | ポート番号に依存しない確実な制御。QoS や帯域制御との連携が可能。 |
| **デメリット** | ディープパケットインスペクションによる CPU 負荷。暗号化通信の識別限界。 |
| **対応機種** | Firepower Threat Defense (FTD), ASA with Firepower Services。 |
| **制限事項** | SSL 復号を行わない場合、識別可能なアプリケーションの詳細度が低下する。 |
| **設計上の注意点** | アプリケーションが特定されるまでの最初の数パケットは通過する可能性がある。 |

---

## 🏗 動作原理

AVC は、FTD の Snort インスペクションエンジン内で動作します。

```text
Incoming Packet
   ↓
L3/L4 Filtering (LINA Engine)
   ↓
Snort Engine Redirect
   ↓
SSL Inspection (If configured/needed)
   ↓
AVC Engine (Detector matching via VDB signatures)
   ↓
Application Identified (e.g., "Facebook Chat")
   ↓
Access Control Policy Lookup
   ↓
Action: Allow / Block / Monitor
```

---

## ⚙ 動作シーケンス

1.  **接続の開始**: クライアントが通信を開始します。最初のパケット（TCP SYN 等）ではアプリケーションは特定できません。
2.  **ペイロードの到着**: 最初の数パケットのペイロード（HTTP Get, TLS Client Hello 等）が FTD に届きます。
3.  **マッチング**: AVC ディテクターがデータベース（VDB）と照合し、特定のシグネチャにマッチするか確認します。
4.  **動的判断**: 
    *   SSL 通信の場合：SNI や証明書の情報を参照します。
    *   非暗号化通信の場合：URL や HTTP ヘッダー、パケットパターンを確認します。
5.  **ポリシー適用**: アプリケーションが特定されると、そのアプリ名を条件に持つ ACP ルールが適用されます。
6.  **情報の蓄積**: FMC の Connection Events にアプリケーションの詳細（名前、カテゴリ、リスク、ビジネス関連度）が記録されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Application Filters の活用**: 個別のアプリを 1 つずつ選ぶのではなく、「Risk: High」や「Category: P2P」といったフィルター（動的なグループ）を使用してルールを作成する要件が出題されます。
*   **Micro-Application の制御**: 単一の Web サービス内の特定の動作（例：Facebook の「Post」と「Like」を個別に制御）を構成するスキルが重要です。
*   **SSL 復号との依存関係**: 「暗号化されたアプリケーションを詳細に識別せよ」という課題が出た場合、必ず **SSL Policy** による復号とセットで構成する必要があります。
*   **VDB アップデート**: 識別可能なアプリリストを最新に保つために、VDB の手動アップデートやスケジュール設定が問われることがあります。
*   **トラブルシュート**: `system support firewall-engine-debug` を使用して、Snort がどの段階でアプリケーションを特定し、どのルールにマッチしたかを確認できる必要があります。

---

## 🛠 設定方法

### 1. アプリケーションベースの ACP ルール作成 (FMC GUI)
1.  **Policies > Access Control** を編集。
2.  **Add Rule** をクリック。
3.  **Applications** タブを選択。
4.  **Available Applications** リストから検索（例：`Skype`）または **Available Filters**（例：`Social Networking`）を選択してルールに追加。
5.  **Action** を `Block` または `Allow` に設定。
6.  **Logging** タブで `Log at End of Connection` を有効化（詳細なアプリ統計を取得するため）。

---

## 🔍 検証コマンド

| 目的 | コマンド（FTD CLI） |
| :--- | :--- |
| **VDB（シグネチャ）バージョンの確認** | <code>show version</code> または <code>show vdb</code> |
| **判定プロセスのリアルタイム追跡** | <code>system support firewall-engine-debug</code> |
| **インスペクション統計の表示** | <code>show snort statistics</code> |
| **現在識別されているアプリの確認** | <code>show asp table socket</code> (詳細情報の確認) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| アプリケーション名が「Unknown」になる | 通信が暗号化されている | SSL Policy を適用して復号し、ペイロードが見える状態にする。 |
| 期待したルールにマッチしない | アプリ特定の遅延 | 最初の数パケットでは特定できないため、ルールの順序や Default Action を確認。 |
| 特定のアプリ機能がブロックできない | VDB が古い | FMC から最新の VDB (Vulnerability Database) をダウンロードしてデプロイ。 |
| アプリ識別情報のログが出ない | ロギングタイミングのミス | ACP ルールで <code>Log at End of Connection</code> を選択しているか確認。 |

---

## ⚠ 制限事項

*   **ポート非依存の限界**: 独自のトンネリング手法や難読化を行うアプリケーションは、完全に識別できない場合があります。
*   **最初のパケットの漏洩**: アプリケーションが完全に識別されるまで（通常数パケット）は、ルールが適用されずパケットが通過する「判定の遅延」が発生することがあります。
*   **ライセンス要件**: AVC 機能を利用するには、対象デバイスに **Threat** サブスクリプションが必要です。

---

## 🔄 他技術との関連

*   **SSL Inspection**: 暗号化された Web トラフィックの中身を解析するために不可欠な補助機能。
*   **QoS (Quality of Service)**: 特定したアプリケーション（例：WebEx）に対して、優先制御や帯域制限をかけることができます。
*   **URL Filtering**: URL フィルタリングはドメイン名やパスに基づきますが、AVC はパケットの内容自体に基づき、URL では分類できないアプリ（P2P等）を制御します。
*   **Security Intelligence (SI)**: アプリケーションが特定されるより前の L3 段階で、既知の悪意ある IP/ドメインを遮断します。

---

## 🧩 比較表

### AVC vs URL Filtering

| 特徴 | AVC (Application Control) | URL Filtering |
| :--- | :--- | :--- |
| **識別対象** | 通信の「中身（動作・構造）」 | 通信の「宛先（アドレス）」 |
| **制御単位** | アプリ名、機能（チャット、投稿等） | ドメイン、URL カテゴリ |
| **得意分野** | P2P, VPN, ゲーム, 複雑なWebアプリ | Webサイト閲覧のフィルタリング |
| **暗号化対応** | 復号がないと詳細識別が困難 | SNI 情報でドメイン特定可能 |

---

## 💡 ベストプラクティス

1.  **Application Filters を優先使用**: 個別のアプリ名ではなく、「高リスク」や「低生産性」といったフィルタリングカテゴリを使用することで、VDB 更新時に追加される新アプリも自動的に制御対象に含めることができます。
2.  **インバウンド保護での利用**: 自社サーバへの通信において、予期しないアプリケーションプロトコル（例：SQLサーバへの非SQL通信）を AVC で遮断し、攻撃面を最小化します。
3.  **推奨ディテクターの使用**: パフォーマンス向上のため、不要なアプリケーションディテクターを無効化することを検討します（高度な構成）。

---

## 📝 ラボ学習・設定サンプル例

### 1. 特定アプリケーションの遮断
*   **要件**: ネットワーク内での BitTorrent の使用を全面的に禁止せよ。
*   **設定**: ACP ルールの Applications タブで `BitTorrent` を選択、Action: `Block`。

### 2. Micro-Application の詳細制御
*   **要件**: Facebook の閲覧は許可するが、Facebook Games の使用のみを遮断せよ。
*   **設定**: アプリリストから `Facebook Games` を選択し `Block`。個別の `Facebook` アプリは `Allow` ルールに配置（順序に注意）。

### 3. アプリケーションカテゴリによる制限
*   **要件**: 「High Risk」と判定されている全アプリケーションへのアクセスを遮断せよ。
*   **設定**: Application Filters で `Risk: 5 (Very High)` を選択し、Block ルールを作成。

### 4. アプリケーションベースの帯域制限 (QoS)
*   **要件**: YouTube 動画トラフィックの最大帯域を 2Mbps に制限せよ。
*   **設定**: ACP の QoS タブ（または独立した QoS Policy）でアプリケーション `YouTube` を指定し、Rate Limit を設定。

### 5. SSL 復号と AVC の連携検証
*   **課題**: SSL 復号なしで Dropbox が正しく識別されるか、復号ありと比較せよ。
*   **検証**: Connection Events を確認。復号なしでは汎用的な SSL 通信と判定される可能性がある。

### 6. 非標準ポートでの SSH 検知
*   **要件**: ポート 443/tcp を使用して回避を試みる SSH 通信を検知・遮断せよ。
*   **設定**: Port を `Any` にし、Application で `SSH` を選択。

### 7. VDB 手動アップデート手順
*   **課題**: オフライン FMC において、提供された VDB ファイルをアップロードし、全 FTD にデプロイせよ。
*   **操作**: `System > Updates > Product Updates` からアップロード。

### 8. アプリケーション利用状況の可視化
*   **課題**: 過去 24 時間で最も帯域を消費しているアプリケーションの円グラフを作成せよ。
*   **操作**: `Analysis > Connections > Dashboards` でウィジェットをカスタマイズ。

### 9. 特定のユーザーグループと AVC の組み合わせ
*   **要件**: `Marketing` グループのユーザーのみ、`Twitter` への投稿を許可せよ。
*   **設定**: User Identity 設定後、ACP ルールで `Source User: Marketing` ＋ `Application: Twitter` を Allow。

### 10. AVC による「回避アプリ」の遮断
*   **要件**: 検閲回避のために使用されるプロキシや VPN アプリ（UltraSurf 等）を自動遮断せよ。
*   **設定**: Application Filters で `Category: VPN and Proxy Server` を選択し Block。

---

## ❓ 想定試験問題

1.  **実装**: 内部ネットワークから外部への通信において、Office 365 の各アプリケーションを個別に識別し、特定の「ビジネス関連度：低」のアプリのみをモニターする構成を完了しなさい。
2.  **トラブルシュート**: AVC ルールを作成しデプロイしたが、Connection Events でアプリケーション名が正しく表示されず「Generic HTTPS」となっている。原因として考えられる最も可能性の高い設定不足は何か？
    *   **回答**: SSL Inspection Policy が構成されていない、またはトラフィックに適用されていないため、Snort が暗号化ペイロード内のアプリシグネチャを特定できていない。
3.  **Design**: 数千の新しいアプリケーションが毎週出現する環境で、管理負荷を抑えつつ常に高リスクアプリを遮断し続けるための設計方針を述べよ。
    *   **回答**: 個別のアプリ名でルールを作成するのではなく、FMC の「Application Filters」機能を使用し、Risk Level やカテゴリに基づいた動的フィルタリングを実装する。
4.  **実装**: 特定のアプリケーション（例：FTP）が非標準ポート（例：2121/tcp）で動作している。これを AVC で確実にキャッチするために、ACP ルールの「Service」タブはどう設定すべきか？
    *   **回答**: ポート番号を固定（21/tcp）にせず、`Any` または `All_TCP` を指定して、Snort エンジンが全ポートのペイロードをスキャンできるようにする。
5.  **コンフィグ読解**: `show snort statistics` の出力で `AppID` 関連のカウンタが増えていない。この状態が意味する、Snort エンジンにトラフィックが届いていない可能性以外の原因は？

---

## 🔗 参考リソース

*   **Cisco Configuration Guides**
    *   [Cisco Secure Firewall Management Center Administration Guide, 7.1 - Application Control](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/application_control.html)
*   **Technical Notes**
    *   [Troubleshooting Firepower Application ID (App-ID)](https://www.cisco.com/c/en/us/support/docs/security/firepower-management-center/215354-configure-syslog-on-firepower-firewall-m.html)
*   **Cisco Live (Videos & Slides)**
    *   [BRKSEC-2021: Firepower Threat Defense - Packet Flow and Troubleshooting](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-2021)

---

## 📝 **補足（Notes）**

*   **学習メモ**: AVC は「名前」で制御するため非常に直感的ですが、その背後にある **VDB (Vulnerability Database)** の存在を忘れないでください。試験環境でアプリが識別されない場合、VDB のバージョンを確認するのが鉄則です。
*   **注意点**: アプリケーションフィルタを多用すると、1 つのルールが背後で数百の個別ルールに展開されます。FTD デバイスのメモリ制限（特に仮想版）を超えないよう、ルールの効率化を常に意識しましょう。
