---
layout: default
title: 4.1-ISE-scalability
nav_order: 4
parent: 4.0-Identity-Management
---

# 4.1 Cisco ISE scalability using multiple nodes and personas

Cisco ISE（Identity Services Engine）は、ネットワークアクセスの制御とポリシー管理を担う中心的なコンポーネントです。CCIE Security v6.1において、この項目はISEを単一のサーバー（Standalone）から、数万以上のエンドポイントをサポートする大規模な分散環境（Distributed Deployment）へと拡張するための**Personas（役割）の理解と、それらを複数のノードにどのように配置・構成するか**に焦点を当てています,。

---

## 📘 概要

*   **機能概要**: ISEの機能を4つの主要な役割（Personas）に分離し、ネットワークの規模や要件に応じて複数の物理または仮想アプライアンス（ノード）に分散配置する機能です。
*   **利用目的**: 処理負荷の分散、冗長性の確保（高可用性）、および地理的に分散した拠点へのポリシー適用ポイントの提供。
*   **利用場面**:
    *   数千〜数十万のエンドポイントを抱える企業ネットワーク。
    *   RADIUS/TACACS+リクエストの遅延を最小限に抑えたい拠点。
    *   管理・監視機能と認証処理機能を分離してセキュリティと可用性を高めたい場合。

---

## 🔑 要点

ISEのPersonasとスケーラビリティの要点：

| 項目 | 内容 |
| :--- | :--- |
| **PAN (Administration)** | 設定、ポリシー定義、ノード管理を行う「脳」。Active/Standbyの冗長化が可能。 |
| **MnT (Monitoring)** | ログの収集、レポート生成、トラブルシュートを行う「記録」。Active/Standbyの冗長化が可能。 |
| **PSN (Policy Service)** | RADIUS, TACACS+, Web Auth, ポスチャ等を処理する「エンジン」。最大100台まで拡張可能。 |
| **pxGrid** | 他のCisco/サードパーティ製品と属性情報を共有する「通信ハブ」,。 |
| **ノード制限 (v3.x)** | 大規模展開で最大100台以上のノードを単一のDeploymentで管理可能。 |
| **専用ノード (Dedicated)** | PSNはRADIUS認証専用、MnTはログ収集専用として構成し、パフォーマンスを最大化する。 |

---

## 🏗 動作原理

分散展開では、ノード間でデータベースの同期とログの転送が行われます。

```text
[ Admin UI ] 
      ↓ (Config Sync)
[ Primary PAN ] ←──────→ [ Secondary PAN (HA) ]
      ↓
[ PSN 1 (Site A) ]    [ PSN 2 (Site B) ]    [ PSN n... ]
      ↓ (Syslog/RADIUS Log)
[ Primary MnT ] ←──────→ [ Secondary MnT (HA) ]
```

1.  **設定同期**: PANで行った設定変更は、HTTPS経由で全てのPSNに即座に同期されます。
2.  **認証処理**: ネットワーク機器（NAD）からのRADIUSリクエストは、最寄りのPSNが受け取ります。
3.  **ロギング**: PSNは処理した認証結果のログをMnTへ送信します。
4.  **情報共有**: pxGridノードがFMCやDNA Centerなどの外部システムとセッション情報を共有します,。

---

## ⚙ 動作シーケンス（ノード登録フロー）

1.  **疎通確認**: 新しいISEノードを立ち上げ、PANからHTTPS（TCP 443）で疎通可能にする。
2.  **証明書信頼**: PANと新ノード間でCA証明書を信頼し合う。
3.  **ノード登録**: PANのGUIから *Administration > System > Deployment* でノードを追加。
4.  **Persona割り当て**: PANが新ノードに対し、どのPersona（PSN, MnT等）として動作するかを指示。
5.  **データ同期**: PANから新ノードへデータベース全体がコピーされる（Full Sync）。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Personasの有効化**: 新しいノードを既存のDeploymentに追加し、特定の機能（Device Admin、pxGrid、SXP等）を有効化する手順は必須です。
*   **PAN HA構成**: Primary PANがダウンした際の挙動や、自動フェイルオーバー（High Availability）の設定が問われます。
*   **MnTの分離**: MnTの負荷が高いシナリオで、専用のMnTノードを構成し、ログの転送先を指定する設定。
*   **ノードステータスの判定**: CLI（`show application status ise`）の結果から、各プロセスが `running` かどうか、またはデータベースの同期状態を読み取る能力。
*   **証明書トラブル**: ノード追加時のエラー原因が、証明書の不一致（Untrusted Certificate）であることを見抜く問題。

---

## 🛠 設定方法

### 1. ノードの分散展開への追加（GUI）
1.  **Administration > System > Deployment** に移動。
2.  **Register** をクリック。
3.  追加するノードの **Hostname/IP** と **Username/Password** を入力。
4.  追加後、ノードを選択して **Edit** し、Personas（PSN, MnT等）を選択して **Save**。

### 2. PAN冗長化 (High Availability)
1.  Primary PAN上で **Make Primary** を実行。
2.  Secondary PAN上で **Register** プロセス完了後、そのノードを選択して **Promote to Secondary Administration Node** を選択。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **ISEプロセスの状態確認** | <code>show application status ise</code> |
| **ノード全体のステータス** | <code>show inventory</code> |
| **認証ログの確認** | <code>show logging last 20</code> |
| **データベース同期状態** | <code>show application status ise</code> (Databaseがrunningか確認) |
| **ノードグループの確認** | <code>show ise node-group</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| ノードの登録に失敗する | 証明書の信頼関係、DNS解決不可 | PANから新ノードを名前で `ping` 可能か、`nslookup` を確認。 |
| 認証が一部のPSNで失敗 | PANとの同期（Sync）失敗 | GUIのDeployment画面でノードの状態が「Connected」かつ緑色か確認。 |
| ログがMnTに表示されない | UDP 20512のポート遮断 | PSNからMnTへの通信経路でFirewallがログ通信を許可しているか確認。 |
| pxGrid経由の情報が不完全 | pxGridノードが非Active | <code>show application status ise</code> で pxGrid プロセスを確認。 |

---

## ⚠ 制限事項

*   **バージョン一致**: Deployment内の全てのノードは、同一のISEバージョンおよびパッチレベルである必要があります。
*   **リソース要件**: 分散環境のMnTやPANは、Standaloneよりも高いCPU/メモリ要件（予約リソース）を求められます。
*   **遅延 (Latency)**: PANと各ノード間のネットワーク遅延が一定（通常300ms以下）を超えると、同期が不安定になります。

---

## 🔄 他技術との関連

*   **2.6 Microsegmentation (TrustSec)**: SXPノードやSGTの伝播におけるISEの役割。
*   **3.10 Cisco DNAC**: DNACの外部AAA/ポリシーソースとしてのISE統合。
*   **4.2 Network Access**: RADIUS/TACACS+リクエストの分散処理。
*   **4.5 Information Exchange**: pxGridを通じたFMCへのユーザ属性共有。

---

## 🧩 比較表

### ISE Scalability Options

| 構成 | ノード数 | 最大エンドポイント数 | 特徴 |
| :--- | :--- | :--- | :--- |
| **Standalone** | 1 | 〜数千 | ラボや小規模環境。冗長性なし。 |
| **Small (Redundant)** | 2 | 〜2万 | 両ノードでPAN/MnT/PSNを兼務。HA構成。 |
| **Medium** | 3〜5 | 〜5万 | PAN/MnTペアを分離。PSNを複数配置。 |
| **Large** | 最大100+ | 〜200万+ | PAN, MnT, PSNが完全に独立。 |

---

## 💡 ベストプラクティス

1.  **MnTの専用化**: アクティブな認証セッションが多い環境では、MnTノードを他の役割から完全に分離し、PSNでのログ保持期間を最適化します。
2.  **DNSの正確性**: 全てのISEノードに正確なAレコードとPTRレコード（逆引き）を用意することは、分散展開の絶対条件です。
3.  **証明書管理**: 自己署名ではなく、内部CAによって署名された証明書を使用し、ノード追加時の信頼エラーを防ぎます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 新規PSNノードの登録
*   **要件**: 10.1.1.20の新しいノードを既存のDeploymentにPSNとして登録せよ。
*   **設定手順**: GUI > Deployment > Register。

### 2. Dedicated MnTノードの構成
*   **要件**: ノードISE-03をMnT専用（Admin/PolicyをOFF）にせよ。
*   **設定**: Deployment画面でISE-03をEditし、Policy Serviceのチェックを外す。

### 3. PANの自動フェイルオーバー設定
*   **要件**: Secondary PANがPrimary PANのダウンを10分後に検知し昇格するように設定せよ。
*   **設定**: Administration > System > Deployment > PAN HA。

### 4. pxGrid Personaの有効化
*   **問題**: FMCとの連携のため、特定のノードでpxGridを有効にせよ。

### 5. 認証ログの外部Syslog転送
*   **要件**: MnTに集約されたログを外部SIEM（10.1.1.100）に転送せよ。

### 6. ノードグループの作成
*   **要件**: Site-Aの3台のPSNを「Group-A」としてまとめ、可用性を向上させよ。

### 7. Device Admin（TACACS+）機能の有効化
*   **要件**: PSN上でTACACS+リクエストを処理できるようにPersonaの設定を変更せよ。

### 8. SXP（TrustSec）サービスの起動
*   **問題**: 特定のPSNでSXPサービスを有効にし、SGTマッピングを配信せよ。

### 9. 証明書更新後の同期修復
*   **操作**: 証明書を入れ替えた後、`Sync` ボタンを使用して強制的に全ノードの設定を同期せよ。

### 10. CLIからのプロセス再起動
*   **課題**: PSNプロセスのハングアップ時、CLIから `application stop ise` と `application start ise` を実行せよ。

---

## ❓ 想定試験問題

1.  **Design**: ISE 3.x の大規模展開において、最大何台の PSN ノードを管理できるか？
    *   **回答**: 最大 **100台**。
2.  **トラブルシュート**: 新しいノードを登録しようとすると「Node Unreachable」エラーが出る。CLIで最初に確認すべきことは？
    *   **回答**: `show application status ise` で管理プロセスが `running` か、および `ping` による相互疎通。
3.  **Design**: MnT ペアを Active/Standby で構成している場合、ログの受信はどのように行われるか？
    *   **回答**: 通常、**Primary MnT が全てのログを受信**し、Secondary MnT はバックアップとしてデータベースを複製し保持する。
4.  **実装**: pxGrid サービスを有効にするための前提条件は何か？
    *   **回答**: PAN/MnT が構成されていること、およびノードに pxGrid 役割が割り当てられていること。
5.  **コンフィグ読解**: `show ise node-group` の結果、`Status: IN-SYNC` と表示されている。これは何を意味するか？
    *   **回答**: ノードグループ内のポリシーキャッシュが正しく同期されている状態。

---

## 🔗 参考リソース

*   **Configuration Guide**: [Cisco ISE 3.1 Administrator Guide - Deployment](https://www.cisco.com/c/en/us/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31/b_ise_admin_guide_31_chapter_011.html)
*   **Design Guide**: [ISE 3.x Performance and Scalability Guide](https://community.cisco.com/t5/security-knowledge-base/ise-performance-and-scalability/ta-p/3630326)
*   **CVD**: [Cisco Identity Services Engine (ISE) Deployment Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/ise-deployment-guide.html)

---

## 📝 **補足（Notes）**

*   **図解**: 
    - **Persona** = 「係（かかり）」。管理係、監視係、受付係（認証）。
    - **Node** = 「物理的なサーバー（またはVM）」。
*   **注意点**: ラボ試験では PSN だけでなく、**pxGrid** や **Passive ID** などの特定の Persona を指示通りに有効化しないと、その後の設定（FMC連携等）が一切動作しなくなるため、初期の Deployment 設定は慎重に行う必要があります。
