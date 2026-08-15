---
layout: default
title: 5.10-Threat-solutions
nav_order: 10
parent: 5.0-Advanced-Threat-Protection
---

# 5.10 Cisco advanced threat solutions and their integration

Cisco の高度な脅威対策ソリューションは、単体で動作するだけでなく、互いに情報を共有し、連携（Integration）することで、ネットワーク全体にわたる「可視化」「検知」「封じ込め」を実現します。CCIE Security v6.1 では、個別の製品知識に加えて、API や専用の連携プロトコル（pxGrid, eStreamer 等）を用いて、いかに統合された防御体系を構築するかが問われます。

---

## 📘 概要

*   **機能概要**: エンドポイント、ネットワーク、Web、Eメール、クラウドの各レイヤーで発生するイベントを統合的に分析し、脅威の相関関係を特定するエコシステムです。
*   **利用目的**: 攻撃の全容（Kill Chain）の把握、インシデントレスポンスの自動化、暗号化通信内の脅威検知。
*   **どのような場面で利用するか**:
    *   **統合調査**: Cisco Threat Response (CTR) を使い、FMC, AMP, Umbrella のログを一括検索して攻撃元を特定する。
    *   **動的遮断**: Stealthwatch が内部の異常な振る舞い（スキャン等）を検知し、ISE や FMC を通じて対象端末を隔離する。
    *   **暗号化通信の分析**: ETA を使用し、復号せずにトラフィックの振る舞いからマルウェアを判定する。

---

## 🔑 要点

| ソリューション | 主な役割 | 連携の核となる要素 |
| :--- | :--- | :--- |
| **Cisco FMC** | NGFW/NGIPS の集中管理 | eStreamer, Syslog, API |
| **Cisco AMP/Secure Endpoint** | ファイルの動的解析・レトロスペクティブ検知 | SHA-256 ハッシュ, Threat Grid 連携 |
| **Cisco Stealthwatch** | NetFlow ベースのネットワーク分析 (NBA) | FlowCollector, FMC 連携 |
| **Threat Grid** | クラウド型サンドボックス | ファイルアップロード, 解析レポート |
| **ETA** | 暗号化トラフィック内の脅威識別 | IDP (Initial Data Packet), SPL (Sequence of Packet Lengths) |
| **Cisco Threat Response** | 脅威情報の集約・オーケストレーション | API キー, モジュール連携 |
| **Cisco Umbrella** | DNS レイヤーの保護・クラウド SIG | DNS ログ, API 連携 |

---

## 🏗 動作原理

統合ソリューションは「情報を集約し、コンテキストを付与する」流れで動作します。

```text
[ Traffic/Events ] 
      ↓
(1) Telemetry Collection: NetFlow (Stealthwatch), Syslog (FMC), DNS (Umbrella)
      ↓
(2) Context Sharing: pxGrid (ISE ↔ FMC/Stealthwatch), eStreamer (FMC ↔ SIEM)
      ↓
(3) Advanced Analysis: Threat Grid (Sandbox), ETA (Encrypted Analytics)
      ↓
(4) Unified Investigation: Cisco Threat Response (SecureX)
      ↓
(5) Enforcement: FMC Block, ISE Quarantine, Umbrella Block
```

---

## ⚙ 動作シーケンス

1.  **検知 (Detection)**: FTD が不審なファイルを検知し、SHA-256 ハッシュを AMP Cloud へ照会。
2.  **分析 (Analysis)**: 未知のファイルの場合、FMC/AMP から Threat Grid へファイルを送り、動的解析を実行。
3.  **相関 (Correlation)**: Stealthwatch がその端末の異常なアウトバウンド通信（C2 通信）を NetFlow から特定。
4.  **調査 (Investigation)**: 管理者が CTR に IP アドレスを入力すると、Umbrella, FMC, AMP すべての履歴がマップ形式で表示される。
5.  **対処 (Remediation)**: CTR の画面から直接「Umbrella でドメインをブロック」「AMP でファイルを隔離」を実行。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **API 連携の設定**: CTR (SecureX) に各デバイスを追加する際、API キーの生成と登録、およびクラウドリージョン（APJC, EU, US）の整合性が重要です。
*   **Stealthwatch と FMC の連携**: FMC のイベント（IPS 等）を Stealthwatch のダッシュボードに表示させるための「Security Analytics Logging」構成。
*   **AMP for Networks**: FTD/FMC でファイルポリシーを構成し、クラウドへの照会を有効にする手順。
*   **ETA の構成**: ルータ側で `et-analytics` を有効にし、Stealthwatch (Flow Collector) へ転送する設定。
*   **pxGrid の設定**: ISE と FMC/Stealthwatch 間での SGT (Security Group Tag) やユーザ情報の共有。

---

## 🛠 設定方法

### 1. FMC への AMP Cloud 登録 (FMC GUI)
1.  **Integration > Cloud Services**。
2.  AMP Cloud のリージョンを選択し、`Enable` をオンにする。
3.  FMC が AMP クラウドと通信するために必要な証明書が自動的に交換されます。

### 2. ルータでの ETA (Encrypted Traffic Analytics) 設定 (CLI)
```bash
! ET-Analytics の設定
et-analytics
 ip address 10.1.1.100 2055  ! Stealthwatch FlowCollector の IP
!
! インターフェイスでの有効化
interface GigabitEthernet1
 et-analytics enable
```

### 3. Stealthwatch での FMC 連携 (SMC GUI)
1.  **SMC コンソール > Configuration > External Data Sources**。
2.  FMC の IP と API 情報を入力し、イベントのプルを開始。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド |
| :--- | :--- | :--- |
| **NetFlow 送信状態確認** | Router | <code>show ip cache flow</code> / <code>show flow monitor</code> |
| **ETA 統計情報の確認** | Router | <code>show platform software et-analytics statistics</code> |
| **FMC クラウド接続確認** | FMC CLI | <code>expert</code> > <code>curl -v https://amp.cisco.com</code> |
| **Stealthwatch サービス状態** | SMC CLI | <code>status detail</code> |
| **eStreamer クライアント確認** | FMC | `show stream-stats` (CLI expert mode) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| **CTR でログが出ない** | API キーの権限不足 | API キーに `Read` だけでなく `Investigate` 権限があるか確認。 |
| **ETA データが届かない** | ポート UDP 2055 の遮断 | ルータと FlowCollector 間の ACL を確認。 |
| **AMP 照会が Fail する** | 時刻同期 (NTP) のズレ | 証明書検証に失敗するため、全デバイスの NTP を確認。 |
| **pxGrid 連携が Orange 状態** | 証明書の信頼関係 | ISE の Root CA を他デバイスに正しくインポートする。 |

---

## ⚠ 制限事項

*   **リージョンの不一致**: AMP Cloud が APJC で、CTR が US リージョンなどの場合、直接連携できないことがあります。
*   **ライセンス依存**: 統合機能の多くは Advantage/Premier ライセンス、または特定のサブスクリプション（Security Analytics 等）を必要とします。
*   **スループット**: ETA や大量の NetFlow 収集は、ルータの CPU やコレクタのディスク I/O に負荷をかけます。

---

## 🔄 他技術との関連

*   **3.6.a NetFlow**: Stealthwatch の生命線となるテレメトリソースです。
*   **4.14 Identity mapping**: ISE (pxGrid) を介して、脅威イベントに「誰が」という情報を付与します。
*   **5.1 AMP**: マルウェア判定のエンジンとして、FMC, WSA, ESA のすべてに組み込まれています。

---

## 🧩 比較表

### Stealthwatch vs FMC IPS

| 特徴 | Cisco Stealthwatch (Network Analytics) | FMC IPS (Firepower) |
| :--- | :--- | :--- |
| **視点** | **トラフィックの振る舞い** (フロー) | **パケットのシグネチャ** (中身) |
| **検知対象** | 内部拡散、偵察行動、データ持ち出し | 脆弱性攻撃、エクスプロイト |
| **強み** | 暗号化通信や非標準ポートに強い | 既知の脆弱性攻撃を確実に止める |

---

## 💡 ベストプラクティス

1.  **Cisco Threat Response の活用**: 調査時は各デバイスの管理画面を個別に見るのではなく、CTR を「調査の起点」とすることで時間を短縮します。
2.  **Flow Logging の最適化**: すべてのフローを送るのではなく、境界ルータやコアスイッチなどの主要な PIN (Points in the Network) から収集する設計にします。
3.  **pxGrid の冗長化**: ISE との連携が切れるとコンテキストが失われるため、pxGrid ノードは冗長構成を推奨します。

---

## 📝 ラボ学習・設定サンプル例

### 1. FMC ↔ AMP Cloud 連携
*   **要件**: FMC でクラウド照会を有効にし、テストファイルのハッシュが検知されることを確認せよ。

### 2. Router ↔ Stealthwatch (NetFlow)
*   **要件**: ISR ルータで Flexible NetFlow を構成し、SMC でフローが表示されるようにせよ。

### 3. FMC ↔ WSA 連携 (SMA 経由)
*   **要件**: Web の閲覧ログを SMA で集計し、FMC のダッシュボードから参照できるようにせよ。

### 4. ETA の実装
*   **要件**: ルータで ETA を有効化し、暗号化通信のメタデータをコレクタへ送信せよ。

### 5. CTR へのモジュール追加
*   **操作**: CTR ダッシュボードで FMC モジュールを追加し、API 疎通テストをパスさせよ。

### 6. Threat Grid サンドボックス連携
*   **要件**: FMC で「解析不能なファイル」を自動的に Threat Grid へアップロードするように設定せよ。

### 7. eStreamer によるログ転送
*   **要件**: FMC の IPS イベントを外部の Linux サーバ（eStreamer Client）へ転送せよ。

### 8. Stealthwatch カスタムポリシー作成
*   **要件**: 特定の内部サーバ間での大量データ転送を検知するポリシーを作成せよ。

### 9. pxGrid による ISE-FMC 連携
*   **要件**: ISE の Active Sessions に表示されるユーザ名を FMC のイベントログに反映させよ。

### 10. 統合調査のシミュレーション
*   **操作**: 特定の悪意あるドメインを CTR で検索し、ネットワーク内のどの端末がアクセスしたか特定せよ。

---

## ❓ 想定試験問題

1.  **Design**: 暗号化されたマルウェア通信を、HTTPS 復号をせずに検知したい。ルータと管理システムに何を導入すべきか？
    *   **回答**: ルータに **ETA (Encrypted Traffic Analytics)** を設定し、管理システムとして **Stealthwatch (Secure Network Analytics)** を導入する。
2.  **トラブルシュート**: FMC と AMP クラウドの連携が「Down」となっている。FMC の CLI で確認すべき事項は？
    *   **回答**: `expert` モードで **DNS 解決** (`nslookup amp.cisco.com`) と **TCP 443 疎通**、および **NTP の同期状態**を確認。
3.  **コンフィグ読解**: ルータの `et-analytics` 設定において、宛先 IP アドレスは何を指しているか？
    *   **回答**: **Stealthwatch Flow Collector** の IP アドレス。
4.  **実装**: FMC の IPS ログをサードパーティの SIEM で詳細分析したい。推奨される連携プロトコルは？
    *   **回答**: **eStreamer**。
5.  **Design**: 脅威調査の時間を短縮するため、複数の Cisco セキュリティ製品のログを横断検索する無償のプラットフォームは？
    *   **回答**: **Cisco Threat Response (CTR)**。

---

## 🔗 参考リソース

*   **Cisco Secure Network Analytics (Stealthwatch) Guide**: [FMC Integration](https://www.cisco.com/c/en/us/td/docs/security/stealthwatch/integrations/FMC/b_Stealthwatch_FMC_Integration_Guide.html)
*   **Cisco Secure Firewall (FMC) Guide**: [Integrating with Cisco Secure Endpoint (AMP)](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/amp_for_firepower.html)
*   **Cisco Live (BRKSEC-2041)**: [Building a Threat Defense Architecture](https://www.ciscolive.com/)
*   **CVD**: [Encrypted Traffic Analytics Deployment Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/eta-deployment-guide.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「連携は情報のマッシュアップ」です。1 つの製品では断片的な情報しか見えませんが、繋げることで「誰が、いつ、どこから、何を持ち出したか」という物語が見えてきます。
*   **図解**: `Telemetries (NetFlow/Syslog) -> Analytical Engines (Stealthwatch/Threat Grid) -> Orchestration (CTR/SecureX)`.
*   **注意点**: ラボ試験では、製品間の **証明書のインポート/エクスポート** がボトルネックになりやすいため、PKI の基本操作に慣れておくことが不可欠です。
