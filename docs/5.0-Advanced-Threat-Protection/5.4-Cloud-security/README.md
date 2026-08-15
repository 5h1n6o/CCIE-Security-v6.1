---
layout: default
title: 5.4-Cloud-security
nav_order: 4
parent: 5.0-Advanced-Threat-Protection
---

# 5.4 Cloud security

**クラウドセキュリティ（Cloud Security）**は、CCIE Security v6.1において、パブリッククラウド（AWS, Azure, GCP等）やSaaSアプリケーション環境におけるデータの保護、可視化、および制御を指します。主要なコンポーネントには、DNS層の保護を提供する **Cisco Umbrella**、クラウドネットワークの振る舞い分析を行う **Cisco Secure Cloud Analytics (Stealthwatch Cloud)**、およびクラウドネイティブなファイアウォール実装である **ASAv/FTDv** が含まれます。

---

## 📘 概要

*   **機能概要**: クラウド環境への接続（エッジ）、クラウド内でのトラフィック（内部）、およびクラウドサービス利用（SaaS）の全方位で脅威防御とガレージ（制御）を提供するソリューション群です。
*   **利用目的**: 境界防御が消失した「境界のないネットワーク」において、ユーザーがどこにいても一貫したセキュリティポリシーを適用し、クラウド内の設定ミスや異常な振る舞いを検知すること。
*   **どのような場面で利用するか**:
    *   **リモートワーク**: Umbrellaを使用して、VPN接続なしでも悪意のあるドメインへのアクセスをブロックする。
    *   **マルチクラウド管理**: AWSやAzureに分散したFTDvを一元管理（FMC）し、セキュアなハイブリッドクラウドを構築する。
    *   **シャドーIT対策**: CASB（Cloud Access Security Broker）機能を用いて、許可されていないクラウドアプリの利用を可視化・制限する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要製品** | Cisco Umbrella, Cisco Secure Cloud Analytics, Cisco Secure Workload (Tetration), FTDv/ASAv。 |
| **Umbrellaの機能** | DNSセキュリティ, Secure Web Gateway (SWG), CASB, クラウド配信型FW (CDFW)。 |
| **可視化手法** | VPC Flow Logs (AWS), NSG Flow Logs (Azure), クラウドAPI連携。 |
| **保護対象** | IaaS (仮想マシン/ネットワーク), PaaS (マネージドサービス), SaaS (O365, Box等)。 |
| **展開モデル** | クラウドネイティブ(API), エージェントベース(AnyConnect), 仮想アプライアンス。 |

---

## 🏗 動作原理

Cisco Umbrellaを中心としたクラウドエッジセキュリティの動作フロー：

```text
[ User / Device ]
      |
      | (1) DNS Query (e.g., malware.com)
      ↓
[ Cisco Umbrella (Global Network) ]
      |-- (2) Check Identity (IP, Tag, or Client ID)
      |-- (3) Reputation Lookup (Cisco Talos)
      ↓
[ Enforcement ]
      |-- Malicious: Return Umbrella Block Page IP
      |-- Risky: Redirect to Intelligent Proxy (Deep Inspection)
      |-- Clean: Return actual IP
```

---

## ⚙ 動作シーケンス

1.  **トラフィックの捕捉**: ユーザーのDNSクエリがUmbrellaのAnycast IP（208.67.222.222）に向けられます。
2.  **アイデンティティ特定**: IPアドレス、ADユーザー情報（コネクタ経由）、またはAnyConnect roaming client識別子によって「誰」かを特定します。
3.  **ポリシー適用**: DNS層、HTTP/HTTPS（SWG）、またはファイル（AMP）の各レベルでスキャンを実行します。
4.  **振る舞い分析 (Cloud Analytics)**: パブリッククラウド内のフローログを取り込み、通常と異なるデータ転送（Exfiltration）や不審な通信を検知します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **AnyConnectとUmbrellaの統合**: AnyConnectに `Umbrella Roaming` モジュールを組み込み、プロファイル（OrgInfo.json）を正しく配置して Umbrella ダッシュボードにデバイスを表示させる設定は重要です。
*   **FMCによるFTDvの管理**: クラウド上のFTDインスタンスをオンプレミスのFMCに登録する際、NAT越えを考慮した `Registration Key` と `NAT ID` の使用が求められます。
*   **Umbrella DNS Policyの作成**: 特定のADグループに対して特定のコンテンツカテゴリ（例：SNS）をブロックするポリシー階層の構成。
*   **Secure Cloud Analytics (SCA)**: AWS/Azureのフローログを取り込むためのAPIキー設定や、特定の `Entity` に対するアラート条件の確認。
*   **トラブルシュート**: `nslookup -type=txt debug.opendns.com` を使用して、デバイスがUmbrellaに正しく紐付いているか確認する手順は必須です。

---

## 🛠 設定方法

### 1. Umbrella Roaming Client (AnyConnect) の構成
1.  Umbrellaダッシュボードから `OrgInfo.json` をダウンロード。
2.  ASA/FTDのWebVPN設定で、AnyConnectパッケージと共にプロファイルを配布。
3.  クライアント側でAnyConnectサービスを再起動し、Umbrellaアイコンが「Protected」になることを確認。

### 2. AWS環境への FTDv デプロイ (概念)
1.  Marketplaceから FTDv イメージを選択。
2.  VPC（Inside, Outside, Management, Diagnostic）の4つのインターフェイスを構成。
3.  FMC側で `Devices > Add > Device` を選択し、クラウド側の管理IP（またはパブリックIP）を指定して登録。

---

## 🔍 検証コマンド

| 目的 | コマンド / 手法 |
| :--- | :--- |
| **Umbrella接続状態の確認** | <code>nslookup -q=txt debug.opendns.com</code> |
| **FTDv インターフェイス確認** | <code>show interface ip brief</code> |
| **FMC 登録状態の確認** | <code>show managers</code> (FTD CLI) |
| **フローログ収集状態の確認** | SCAダッシュボードの **Service Status** |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| Umbrellaが保護中にならない | DNSが208.67.222.222に向いていない | <code>ipconfig /all</code> でDNSサーバを確認。 |
| FTDvがFMCに登録できない | セキュリティグループ（AWS）の遮断 | TCP 8305 ポートの双方向許可を確認。 |
| ADユーザー名が表示されない | AD Connectorの同期不全 | Umbrella Connector サービスのステータスを確認。 |
| クラウドフローが見えない | API権限不足 (AWS/Azure) | IAMロールに `ReadOnlyAccess` が付与されているか確認。 |

---

## ⚠ 制限事項

*   **MTUの考慮**: クラウド環境（特にAWS）ではMTU 1500が標準ですが、VPNトンネルを併用する場合はフラグメンテーションに注意が必要です。
*   **Umbrella SWGのSSL復号**: HTTPSインスペクションを有効にする場合、各クライアントにUmbrellaのルート証明書をインストールする必要があります。
*   **ライセンス依存**: Umbrellaの機能（DNS, SWG, CASB）は契約プラン（DNS Security Essentials/Advantage, SIG Essentials）によって制限されます。

---

## 🔄 他技術との関連

*   **1.0 VPN**: VTI (Virtual Tunnel Interface) を使用して、オンプレミスとクラウドVPCをセキュアに接続する。
*   **4.14 Identity mapping**: ISEが収集したユーザー情報をUmbrellaに連携し、クラウドポリシーに反映させる。
*   **3.6 Monitoring**: NetFlow情報をクラウドコレクタに送信し、グローバルな脅威情報と相関分析を行う。

---

## 🧩 比較表

### Umbrella DNS vs Umbrella SIG (SWG)

| 特徴 | DNSセキュリティ | SIG (Secure Internet Gateway) |
| :--- | :--- | :--- |
| **プロトコル** | UDP 53のみ | HTTP/HTTPS, 全ポート(CDFW) |
| **検査精度** | ドメイン単位 | URL/ファイル(AMP)/サンドボックス |
| **パフォーマンス** | 極めて高速（遅延なし） | 検査によるオーバーヘッドあり |
| **展開難易度** | 低（DNS変更のみ） | 中（トンネル構築またはPAC） |

---

## 💡 ベストプラクティス

1.  **階層化防御**: DNS層（Umbrella）で大部分の脅威を事前にフィルタリングし、残りのトラフィックをクラウドFW（FTDv）で詳細検査する。
2.  **リージョン最適化**: クラウドサービス（SCA, Umbrella）の接続先は、最短のレイテンシとなるリージョンを選択する。
3.  **自動スケーリング**: AWS Auto Scaling等を利用し、トラフィック負荷に応じてFTDvインスタンスを自動増減させる設計を検討する。

---

## 📝 ラボ学習・設定サンプル例

### 1. AnyConnect Umbrella Roaming Profile
*   **要件**: ASA経由でAnyConnect接続したユーザーにUmbrellaポリシーを強制せよ。
*   **設定**: `anyconnect-custom-data opendns-info [OrgInfo.jsonの内容]`。

### 2. FTDv の FMC への登録（NAT環境）
*   **要件**: NAT背後のFTDvをFMCに登録せよ。
*   **FTDコマンド**: <code>configure manager add [FMC_IP] [KEY] [NAT_ID]</code>。

### 3. Umbrella DNS Policy (Category Block)
*   **要件**: `Adult Content` と `Gambling` をブロックせよ。
*   **操作**: Umbrella Dashboard > Policies > DNS Policies > Add Rule。

### 4. Umbrella Intelligent Proxy の有効化
*   **要件**: 不審なドメイン（Risky）のみSSL復号して検査せよ。

### 5. Secure Cloud Analytics API 連携
*   **要件**: AWSの認証情報を入力し、VPC Flow Logsを取得せよ。

### 6. Cloud Connector (ISR 4000)
*   **要件**: ルータをUmbrellaネットワークデバイスとして登録せよ。

### 7. Umbrella Block Page Customization
*   **要件**: 遮断時に会社のヘルプデスク連絡先を表示させよ。

### 8. App Discovery (CASB)
*   **要件**: 未承認のストレージサービス（Dropbox等）の使用を検出せよ。

### 9. FTDv HA (High Availability) in AWS
*   **要件**: AWS Lambdaを使用して、障害時にルートテーブルを自動書き換えさせよ。

### 10. Umbrella API によるイベント抽出
*   **要件**: Pythonスクリプトを使用して過去1時間のセキュリティイベントを取得せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: AnyConnectクライアントで `nslookup` を実行したところ、DNSサーバが 127.0.0.1 になっている。これは正常か？
    *   **回答**: **正常**。Umbrella Roaming Clientが動作している場合、自身をプロキシとして機能させ、クエリをUmbrellaへ暗号化して転送するため。
2.  **トラブルシュート**: クラウド上のFTDvからFMCへの接続が `Disconnected` になっている。確認すべきFTD CLIコマンドは？
    *   **回答**: <code>show manager statistics</code> および <code>system support diagnostic-cli</code> でパケットキャプチャを確認。
3.  **Design**: モバイルユーザーに最高レベルの保護（サンドボックス解析含む）を提供したい。Umbrellaのどの機能が必要か？
    *   **回答**: **Secure Web Gateway (SWG)** と **Cisco AMP/Threat Grid integration**。
4.  **実装**: AWS VPC内のインスタンス間で発生する水平移動（Lateral Movement）を検知するための最適なソリューションは？
    *   **回答**: **Cisco Secure Cloud Analytics (Stealthwatch Cloud)**。
5.  **トラブルシュート**: Umbrellaでドメインをブロックしたが、ブロックページが表示されずタイムアウトする。原因は？
    *   **回答**: クライアントがブロックページのIPに対する **HTTP/HTTPS (80/443)** 通信をファイアウォール等で許可していない。

---

## 🔗 参考リソース

*   **Cisco Umbrella 管理者ガイド**: [Deploying Umbrella Roaming Client](https://docs.umbrella.com/)。
*   **FTD 7.1 Configuration Guide**: [Firewall Deployment in Cloud](https://www.cisco.com/c/ja_jp/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71.html)。
*   **Cisco Live (BRKSEC-2041)**: [Cloud Security Architecture and Design](https://www.ciscolive.com/)。
*   **CVD**: [Cisco Umbrella Integration with AnyConnect](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/ise-deployment-guide.html)。

---

## 📝 **補足（Notes）**

*   **学習メモ**: クラウドセキュリティは「API連携」と「トンネル構築」のどちらを使用しているかを常に意識してください。Umbrellaは主にDNS/HTTPを扱い、Cloud Analyticsは「フロー（メタデータ）」を扱います。
*   **図解**: 
    - **Control Plane**: FMC, Umbrella Dashboard.
    - **Data Plane**: FTDv, Umbrella Anycast IP.
*   **注意点**: ラボ試験では、**クラウド側のセキュリティグループ（FW）**の設定漏れがトラブルの主な原因になります。Cisco機器の設定だけでなく、基盤側の許可ルールも念頭に置いてください。
