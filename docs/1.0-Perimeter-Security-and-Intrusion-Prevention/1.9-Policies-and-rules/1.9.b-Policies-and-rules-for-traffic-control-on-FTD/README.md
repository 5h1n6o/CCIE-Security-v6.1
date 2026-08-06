---
layout: default
title: 1.9.b Policies and rules for traffic control on FTD
nav_order: 1
parent: 1.9-Policies-and-rules
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.9.b Policies and rules for traffic control on Cisco FTD

Cisco Firepower Threat Defense (FTD) におけるトラフィック制御は、**Access Control Policy (ACP)** を核とした高度に統合されたポリシー体系に基づいています。従来のファイアウォール（ASA）の L3/L4 制御に加え、アプリケーション制御 (AVC)、URL フィルタリング、侵入防御 (IPS)、ファイル制御/マルウェア防御 (AMP) などの次世代機能を一つのポリシー内で一元的に管理します。

---

## 📘 概要

*   **機能概要**: FTD のポリシーは、Firepower Management Center (FMC) で作成・管理され、パケットの属性（IP、ポート、プロトコル、アプリケーション、URL、ユーザー等）に基づいて通信を制御します。
*   **利用目的**: 単一の管理画面（FMC）から、ネットワーク全体のセキュリティ体制（アクセス制御、脅威防御、暗号化解除）を統合的に適用します。
*   **利用場面**: 境界防御（エッジ FW）、データセンター内のセグメンテーション、暗号化通信の可視化と検査、特定の Web カテゴリへのアクセス制限などに利用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **制御方式** | **First-match アルゴリズム** による上から順のルール評価。 |
| **主要コンポーネント** | Access Control, Prefilter, SSL, Identity, NAT, DNS, QoS ポリシー。 |
| **インスペクション** | **Snort エンジン** による L7 ディープパケットインスペクション。 |
| **メリット** | オブジェクト指向管理。一貫した脅威防御。ポリシーの継承 (Inheritance)。 |
| **デメリット** | 設定のデプロイに時間がかかる。Snort の負荷によるパフォーマンス変動。 |
| **対応機種** | Firepower 1000/2100/4100/9300, ISA3000, vFTD。 |
| **設計上の注意** | ルールの順序が重要。重い検査（IPS/AMP）は必要なトラフィックに限定する。 |

---

## 🏗 動作原理

FTD のパケット処理は、LINA エンジン（ASA ベース）と Snort エンジン（次世代検査）のハイブリッドアーキテクチャで動作します。

```text
Incoming Packet
   ↓
[ Prefilter Policy ] --- (Fastpath / Analyze / Block)
   ↓
[ Tunnel / SSL Decryption ] --- (If encrypted)
   ↓
[ Identity Policy ] --- (User to IP mapping)
   ↓
[ Access Control Policy ] --- (L3-L7 Rules)
   ↓
[ Snort Engine ] --- (IPS / File / Malware / AVC)
   ↓
[ NAT Policy ]
   ↓
Outgoing Packet
```

---

## ⚙ 動作シーケンス

1.  **Prefilter 評価**: ACP 評価前の非常に早い段階で L3/L4 情報に基づき、パケットをバイパス（Fastpath）するか、さらなる解析に回すかを決定します。
2.  **SSL 復号**: 暗号化されたトラフィックを SSL Policy で検査し、必要に応じて復号します。
3.  **アイデンティティ特定**: Identity Policy により、IP アドレスに対応するユーザー情報を紐付けます。
4.  **ACP ルールマッチング**: 上から順にルールを評価し、最初に一致したルールの **Action** を適用します。
    *   **Trust**: Snort をスキップして許可。
    *   **Allow**: Snort による高度な検査（IPS、File 等）を継続。
    *   **Block**: 通信を即座に遮断。
5.  **Snort インスペクション**: `Allow` ルールに紐付けられた Intrusion Policy (IPS) や File Policy (AMP) を実行します。
6.  **デフォルトアクション**: どのルールにも一致しない場合、ACP 全体の Default Action（通常は Block）が適用されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **ポリシーの順序**: ルールは「最も具体的なもの」から順に並べる必要があります。広い範囲の `any` ルールの下に特定の `Block` ルールを置いても意味がありません。
*   **Intrusion Policy の変数セット (Variable Sets)**: 試験では、特定のネットワーク範囲（\$HOME\_NET）に対して IPS を最適化する構成が問われます。
*   **SSL Decryption**: 暗号化された Web トラフィック（HTTPS）に対して、証明書の信頼関係を構築し、AVC や URL フィルタリングを機能させる構成スキル。
*   **Prefilter による Fastpath**: 「特定のバックアップトラフィックを Snort 検査なしで高速転送せよ」という要件に対し、Prefilter で Fastpath を設定する能力。
*   **Troubleshooting**: `system support firewall-engine-debug` コマンドを使用して、パケットがどのルールにマッチし、なぜドロップされたかを CLI から追跡するスキルが不可欠です。

---

## 🛠 設定方法

### 1. Access Control Rule の作成 (FMC GUI)
1.  **Policies > Access Control** で `Edit`。
2.  **Add Rule** をクリック。
3.  **Name**: 任意の名前を入力。
4.  **Zones / Networks / Ports**: 送信元と宛先を定義。
5.  **Applications / URLs**: アプリケーションやカテゴリを指定。
6.  **Action**: `Allow` を選択。
7.  **Inspection**: 
    *   `Intrusion Policy`: 推奨ポリシー（Balanced Security and Connectivity 等）を選択。
    *   `File Policy`: マルウェア検知用ポリシーを選択。
8.  **Logging**: `Log at End of Connection` をチェック。

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI) |
| :--- | :--- |
| **判定プロセスのリアルタイム追跡** | `system support firewall-engine-debug` |
| **ACPの統計情報の表示** | `show access-control-config` |
| **インターフェイスごとのドロップ確認** | `show asp drop` |
| **Snortの稼働状態確認** | `show snort status` |
| **パケットトレース（シミュレーション）** | `packet-tracer input [IF] [proto] [src] [port] [dst] [port]` |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| 通信が意図せず Block される | ルール順序のミス | FMC でルールの順序を見直し、より具体的なルールを上位へ移動。 |
| アプリケーション識別が不安定 | VDB が古い | **System > Updates** から最新の VDB をデプロイ。 |
| SSL 復号でエラーが出る | 証明書（Internal CA）の未配布 | 復号に使用した CA 証明書をクライアント端末にインストール。 |
| IPS が攻撃を検知しない | Variable Set の設定ミス | \$HOME_NET に対象のサーバーネットワークが含まれているか確認。 |
| 変更が反映されない | デプロイの失敗 | **Deploy > Deployment** でエラーログを確認。 |

---

## ⚠ 制限事項

*   **同一ハードウェア要件**: クラスタリングや HA を組む場合、FTD デバイス間でソフトウェアバージョンやライセンスが一致している必要があります。
*   **ライセンス依存**: URL フィルタリングや AMP 機能を利用するには、個別のサブスクリプション（URL, Malware）が必要です。
*   **Snort のボトルネック**: 複雑な正規表現を用いたカスタムシグネチャを多用すると、Snort プロセスの CPU 使用率が急増します。

---

## 🔄 他技術との関連

*   **Routing**: FTD でのルーティング設定は ASA と非常に類似していますが、FMC 経由で行います。
*   **Security Intelligence (SI)**: ACP の評価前に、Talos の脅威フィードに基づいて既知の悪意ある IP/ドメインを遮断します。
*   **User Identity**: ISE (PxGrid) または FMC AD Agent を使用して、ユーザー名ベースのフィルタリングを実現します。

---

## 🧩 比較表

### FTD Access Control vs ASA ACL

| 特徴 | FTD Access Control Policy | ASA Access Control List (ACL) |
| :--- | :--- | :--- |
| **管理単位** | オブジェクト統合型 (FMC) | インターフェイス毎 / グローバル (CLI) |
| **レイヤー** | L3 - L7 統合 | L3/L4 中心（L7はMPFが必要） |
| **脅威防御** | 同一ポリシー内で IPS/AMP を統合 | ASA with Firepower Services が必要 |
| **アイデンティティ** | Identity Policy による動的連携 | Identity Firewall 設定が必要 |

---

## 💡 ベストプラクティス

1.  **Default Action は Block**: セキュリティの基本として、明示的に許可したもの以外はすべて拒否します。
2.  **ロギングの最適化**: `Log at Beginning` は接続の開始を知るのに役立ちますが、トラフィック量が多い場合は `Log at End` を使用して Snort 判定結果を含めたログを収集します。
3.  **Prefilter の活用**: 信頼できる通信（例: 拠点間 VPN 内のバックアップ）は Prefilter で Fastpath 化し、Snort のリソースを節約します。
4.  **ルールの名前付け**: CCIE ラボ試験では、指示された命名規則を厳守してください。

---

## 📝 ラボ学習・設定サンプル例

### 1. 特定アプリケーション (Facebook) の遮断
*   **要件**: 全社員の Facebook ゲームへのアクセスのみを遮断せよ。
*   **設定**: Rule Action: `Block`, Applications: `Facebook Games`。

### 2. URL カテゴリベースの許可
*   **要件**: `Financial Services` カテゴリへのアクセスのみを許可せよ。
*   **設定**: Rule Action: `Allow`, URLs: `Category: Financial Services`。

### 3. Identity ベースの制御
*   **要件**: `HR_Group` ユーザーのみが人事給与サーバーにアクセスできるようにせよ。
*   **設定**: Source User: `HR_Group`, Destination IP: `人事サーバー`, Action: `Allow`。

### 4. SSL 復号（Decrypt-Resign）の実装
*   **要件**: HTTPS トラフィックを復号してマルウェア検査を可能にせよ。
*   **設定**: SSL Policy を作成。Action: `Decrypt - Resign`, FMC 内の Internal CA を使用。

### 5. Prefilter による Fastpath 構成
*   **要件**: ネットワーク内の DNS 通信を Snort 検査なしで許可せよ。
*   **設定**: Prefilter Policy にルール追加。Action: `Fastpath`, Port: `UDP/53`。

### 6. ファイル制御（AMP）の設定
*   **要件**: ダウンロードされる実行ファイル (.exe) をマルウェアスキャンせよ。
*   **設定**: File Policy を作成。File Type: `Executable`, Action: `Malware Cloud Lookup`, ACP ルールに適用。

### 7. DNS Sinkhole の実装
*   **要件**: ボットネットの C&C 通信ドメインへの問い合わせを偽の IP に誘導せよ。
*   **設定**: DNS Policy を作成。Category: `Botnet`, Action: `Sinkhole`。

### 8. Intrusion Policy (IPS) の調整
*   **要件**: `High Security` 向けの IPS シグネチャセットを適用せよ。
*   **設定**: ACP ルールの Inspection タブで `Security Over Connectivity` Policy を選択。

### 9. 信頼されたホストのバイパス (Trust)
*   **要件**: 管理端末からの通信を一切の検査なしで許可せよ。
*   **設定**: Rule Action: `Trust`, Source IP: `管理端末IP`。

### 10. CLI による判定デバッグ
*   **要件**: 特定の IP (10.1.1.5) がどの ACP ルールに合致しているか CLI で調査せよ。
*   **実行**: `system support firewall-engine-debug` を実行後、10.1.1.5 から通信を発生させる。

---

## ❓ 想定試験問題

1.  **Design**: FTD において、内部サーバーへの大量のバックアップトラフィックが Snort エンジンを圧迫し、正規の Web 通信に遅延が発生している。CPU 負荷を軽減するための最も適切な設定箇所はどこか？
    *   **回答**: Prefilter Policy において、バックアップトラフィックを `Fastpath` に設定する。
2.  **トラブルシュート**: FMC から新しい Access Control Rule をデプロイしたが、Connection Events でそのルール名が表示されず、Default Action でドロップされている。考えられる原因は？
    *   **回答**: ルールの順序が間違っており、上位のより広範な拒否ルール（または Default Action より上のルール）に先にマッチしている、あるいはデプロイが実際には完了していない。
3.  **実装**: ユーザーが HTTPS サイトにアクセスした際、URL フィルタリングでカテゴリに基づいた遮断を行うために、ACP 以外に必須となるポリシーは何か？
    *   **回答**: SSL Policy（復号を行わない場合でも、SNI を見るために必要だが、詳細なパス制御には復号が必須）。
4.  **コンフィグ読解**: `system support firewall-engine-debug` の出力で `Verdict: Blacklist` と表示された。これはどの機能による遮断か？
    *   **回答**: Security Intelligence。
5.  **Design**: 拠点ごとに異なる IPS 設定を適用したいが、共通の Access Control ルールも維持したい。FMC のどの機能を使用すべきか？
    *   **回答**: Policy Inheritance（ポリシーの継承）機能を使用し、親ポリシーで共通ルールを、子ポリシーで拠点固有の IPS 設定を行う。

---

## 🔗 参考リソース

*   **Cisco Firepower Management Center Administration Guide, 7.0**
    *   [Access Control Policies](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/access_control_policies.html)
*   **Cisco Secure Firewall Threat Defense Command Reference**
    *   [Firepower Threat Defense CLI Commands](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/fdm/fptd-fdm-config-guide-620.html)
*   **Cisco Live (BRKSEC-2021)**
    *   [Firepower Threat Defense - Packet Flow and Troubleshooting](https://www.ciscolive.com/on-demand/on-demand-library.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: FTD は「Snort を通すかどうか」を ACP の Action (`Allow` vs `Trust`) で制御している点を強く意識してください。
*   **図解**: 常に FMC の画面レイアウト（ACP の各タブ：Networks, Apps, Users, URLs 等）を脳内でイメージできるよう、実機（vFTD）での操作を繰り返すことが重要です。
*   **注意点**: ラボ試験では、設定変更のたびに `Deploy` が必要になります。デプロイには数分かかるため、複数のタスクの設定を一気に済ませてからデプロイする等の時間管理術が求められます。
