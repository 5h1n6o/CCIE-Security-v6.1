---
layout: default
title: 1.4.d-Dynamic-objects
nav_order: 4
parent: 1.4-FMC-features
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.4.d Dynamic objects

Cisco Secure Firewall Management Center (FMC) における**ダイナミックオブジェクト（Dynamic Objects）**は、ポリシーの再デプロイ（Redeploy）を必要とせずに、APIなどを介してリアルタイムに内容（IPアドレスなど）を更新できる特別なオブジェクトです。特に、頻繁にIPアドレスが変更されるクラウド環境や、外部の脅威情報に基づいて動的に通信を遮断したい場合に極めて有効な機能です。

---

## 📘 概要

*   **機能概要**: 通常のネットワークオブジェクトがデプロイ時にデバイスへ書き込まれるのに対し、ダイナミックオブジェクトは外部から注入されたデータを即座にデータパスに反映させます。
*   **利用目的**: 高頻度で変更されるエンドポイント（クラウド、コンテナ等）の追跡や、セキュリティインシデント発生時の迅速な隔離（Quarantine）に使用されます。
*   **利用場面**: 脅威検知システムが特定IPを「悪意あり」と判断した際、FMCのAPIを叩いて即座に全拠点のFWで遮断する、といった自動化連携に最適です。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | **ポリシーデプロイなし**でIPアドレスリストを更新可能。 |
| **用途** | 外部脅威連携、クラウドIP追跡、ホストの動的隔離。 |
| **メリット** | 運用負荷の軽減、レスポンス速度の向上（数秒で反映）。 |
| **デメリット** | 設定ミスが即座に反映されるリスク。管理は主にAPIが推奨される。 |
| **対応機種** | Firepower Threat Defense (FTD) 7.0以降, FMC 7.0以降。 |
| **制限事項** | サポートされるのはネットワーク（IP）オブジェクトのみ。 |
| **設計上の注意** | デフォルトでは中身が空のため、初期状態でのポリシー挙動を考慮する。 |

---

## 🏗 動作原理

ダイナミックオブジェクトは「器（コンテナ）」として定義され、実際のIPアドレス情報はFMCのデータベースおよびFTDのメモリ内に動的に保持されます。

```text
[ External System / Script ]
   ↓ (REST API: Update Dynamic Object)
[ Cisco FMC ]
   ↓ (Real-time Push via Management Tunnel)
[ Cisco FTD Device ]
   ↓ (Update Data Plane Table)
[ Traffic Flow ] → (Matches Dynamic Object Rule) → [ Permit / Deny ]
```

---

## ⚙ 動作シーケンス

1.  **オブジェクト作成**: 管理者がFMC GUIで `Dynamic Object` を作成し、Access Control Policy (ACP) で使用します。
2.  **初期デプロイ**: この時点ではオブジェクトは「空」またはデフォルトの状態ですが、ルールの枠組みとして一度だけFTDへデプロイされます。
3.  **データ注入**: 外部スクリプトやCisco ISE等の連携システムが、FMC REST APIを介して対象オブジェクトにIPアドレスを追加します。
4.  **即時反映**: FMCはデプロイプロセスを経由せず、変更内容を対象のFTDユニットへ即座に送信します。
5.  **適用**: FTDはデータプレーンのテーブルを更新し、それ以降の通信に対して新しいIPリストを適用します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **API連携**: ラボでは「Pythonスクリプトを使用して特定のIPをダイナミックオブジェクトに追加せよ」といったプログラマビリティとセキュリティの複合問題が出る可能性があります。
*   **デプロイ不要の利点**: 「デプロイ待ち時間を最小化して特定のIPをブロックせよ」という要件に対し、通常のオブジェクトではなくダイナミックオブジェクトを選択できるかが問われます。

### ラボ試験で設定させられそうな内容
*   FMC GUIでのダイナミックオブジェクトの新規作成。
*   ACPルールにおけるソース/宛先としての適用。
*   API Explorerを使用した、オブジェクトへのIPアドレス追加・削除操作。

### showコマンドから状態を判断
*   FTD CLIでのオブジェクト同期確認。
*   FMC API応答（200 OK / 204 No Content）の確認。

---

## 🛠 設定方法

### 1. FMC GUIでのオブジェクト作成
1.  **Objects > Object Management** に移動します。
2.  左メニューから **Dynamic Objects** を選択し、**Add Dynamic Object** をクリックします。
3.  名前（例: `Blocked_IPs`）を付け、**Object Type** が `Network` であることを確認して保存します。

### 2. ポリシーへの適用
1.  **Policies > Access Control** でルールを編集。
2.  Networksタブの検索欄で作成したオブジェクトを選択し、`Source` または `Destination` に配置します。
3.  **Action** を `Block` に設定し、保存して**一度だけデプロイ**します。

### 3. APIによるIP追加（REST API例）
```bash
# POST /api/fmc_config/v1/domain/{domainUUID}/object/dynamicobjects/{objectUUID}/mappings
{
  "add": [
    {"value": "192.168.50.100"},
    {"value": "10.0.0.0/24"}
  ]
}
```

---

## 🔍 検証コマンド

| 目的 | コマンド（FTD CLI） |
| :--- | :--- |
| **ダイナミックオブジェクトの同期確認** | <code>show dynamic-objects</code> |
| **特定のオブジェクトに含まれるIPの確認** | <code>show dynamic-objects ids [ID]</code> |
| **API疎通履歴の確認 (FMC)** | <code>tail -f /var/log/httpd/httpsd_access_log</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| APIでIPを追加しても通信が止まらない | オブジェクトUUIDの指定ミス | API Explorerで正しいUUIDを取得し直す。 |
| FTD側でリストが空のまま | 管理トンネル (SFTunnel) の不調 | <code>show sftunnel status</code> で疎通を確認。 |
| APIリクエストが 403 Forbidden | 権限不足 | APIユーザーに <code>REST API Read/Write</code> 権限があるか確認。 |
| オブジェクト自体が選択できない | バージョン不一致 | FTD/FMCが7.0以降であることを確認。 |

---

## ⚠ 制限事項

*   **バージョン依存**: FMC/FTD 7.0未満では使用できません。
*   **最大エントリ数**: デバイスのモデルごとに保持できる動的IP数に上限があります。
*   **Persistence**: FTDが再起動した場合、FMCから最新のリストが再同期されるまで時間がかかる場合があります。

---

## 🔄 他技術との関連

*   **REST API**: ダイナミックオブジェクトを運用するための主要インターフェイスです。
*   **Cisco ISE (Rapid Threat Containment)**: ISEが検知した脅威情報をFMCのダイナミックオブジェクトへ自動注入する連携が一般的です。
*   **Security Intelligence (SI)**: 似た機能ですが、SIは外部フィード（URL/ドメイン）に強く、ダイナミックオブジェクトは特定の内部/外部IPの個別操作に強いという違いがあります。

---

## 🧩 比較表

### Standard Object vs Dynamic Object

| 特徴 | Standard Network Object | Dynamic Object |
| :--- | :--- | :--- |
| **反映タイミング** | ポリシーデプロイ時 | **APIコール後（数秒）** |
| **反映方式** | 静的（設定に依存） | 動的（外部データに依存） |
| **用途** | 恒常的なネットワーク定義 | 短期的なブロック、クラウド連携 |
| **操作性** | GUIがメイン | **API/スクリプトがメイン** |

---

## 💡 ベストプラクティス

1.  **命名規則の徹底**: 動的に変更されるものであることを示すため、`DYN_` プレフィックスを付けることを推奨します。
2.  **有効期限の管理**: スクリプト側で「1時間後に削除する」などのロジックを組み込み、古いブロックリストが残り続けないようにします。
3.  **デフォルトの安全策**: ダイナミックオブジェクトが空の状態でも、ネットワーク全体に影響が出ないようなルール配置（順序）を検討します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 外部攻撃者の即時遮断
*   **要件**: 攻撃者のIP `203.0.113.5` を、デプロイを行わずに即座に全インターフェイスで遮断せよ。
*   **設定**: `Add Dynamic Object` -> ACPの一番上に `Block` ルール作成。

### 2. ISE RTC 連携
*   **要件**: ISEから隔離信号を受け取ったホストを遮断。
*   **設定**: ISE側でFMCをアダプタとして登録し、Dynamic Objectを指定。

### 3. クラウドWebサーバ群の許可
*   **要件**: オートスケールでIPが変わるWebサーバ群へのアクセスを許可。
*   **設定**: サーバ起動スクリプトにFMC APIコールを組み込む。

### 4. API Explorerでの操作検証
*   **課題**: FMCのAPI Explorer (https://[FMC_IP]/api/api-explorer) を開き、作成したオブジェクトのUUIDを取得せよ。

### 5. Pythonによる自動更新
*   **要件**: 特定のログを検知したら `requests` ライブラリでIPを注入せよ。

### 6. オブジェクトのクリア
*   **課題**: オブジェクト内の全IPを一度に削除するAPIリクエストを構成せよ。

### 7. 複数デバイスへの同時反映
*   **要件**: 1つのダイナミックオブジェクトを複数拠点のFTDで共有せよ。

### 8. FTD側でのエントリ確認
*   **コマンド**: `system support firewall-engine-debug` を併用し、動的IPのマッチングを確認。

### 9. 接続タイムアウトの影響確認
*   **課題**: IPが追加された際、既存のコネクションが切断されるか検証せよ。

### 10. クラスタ環境での同期
*   **要件**: FTDクラスタにおいて、全ユニットに動的IPが配布されているか確認。

---

## ❓ 想定試験問題

1.  **実装**: FMC 7.xにおいて、ポリシーを再デプロイすることなく、外部ソースから供給されるIPアドレスに基づいてアクセス制御を更新するためのコンポーネントは何か？
    *   **正解**: Dynamic Objects。
2.  **トラブルシュート**: Pythonスクリプトでダイナミックオブジェクトを更新しようとしたが、404 Not Foundが返る。URL内の何が間違っている可能性が高いか？
    *   **正解**: ドメインUUIDまたはダイナミックオブジェクト自体のUUID。
3.  **Design**: セキュリティ運用チームが1日100回以上の頻度で遮断リストを更新する必要がある。最適な構成を提案せよ。
    *   **正解**: REST APIとダイナミックオブジェクトを組み合わせた自動化構成。
4.  **実装**: ダイナミックオブジェクトを使用したルールを作成した後、最初に行うべき手順は？
    *   **正解**: オブジェクトを紐付けたアクセスコントロールポリシーのデプロイ。
5.  **コンフィグ読解**: `show dynamic-objects` の出力に `Total objects: 1` とあるが、実際のIPリストが表示されない。この状態が示す意味は？
    *   **正解**: オブジェクトの定義はデプロイされているが、API等を通じて具体的なIPアドレスのマッピングがまだ注入されていない状態。

---

## 🔗 参考リソース

*   **Configuration Guide**:
    *   [Cisco Secure Firewall Management Center Administration Guide, 7.1 - Dynamic Objects](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/objects.html#id_92425)
*   **API Reference**:
    *   [Cisco FMC REST API Quick Start Guide](https://www.cisco.com/c/en/us/td/docs/security/firepower/620/api/REST_API_Quick_Start_Guide_for_Firepower_Management_Center.html)
*   **Technical Notes**:
    *   [FTD Dynamic Objects - Use cases and Implementation](https://community.cisco.com/t5/security-documents/ftd-dynamic-objects-use-cases-and-implementation/ta-p/4366627)

---

## 📝 **補足（Notes）**

*   **学習メモ**: ダイナミックオブジェクトは、CCIE試験における「Security Automation」と「Perimeter Security」の橋渡しとなる技術です。GUI設定だけでなく、必ずAPIの構造（URLパスやJSONフォーマット）をセットで復習してください。
*   **注意点**: ラボ試験でAPIを叩く際は、Authトークンの取得プロセスを忘れないようにしましょう。トークンがないとすべてのリクエストが 401 Unauthorized になります。
---
