---
layout: default
title: 2.4.a-ASA-VPN-clustering
nav_order: 2
parent: 2.4-VPN-HA
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.4.a Cisco ASA VPN clustering

Cisco ASA におけるクラスタリングは、最大 16 台の ASA デバイスを 1 つの論理的なエンティティとして動作させる機能であり、高いスループットと冗長性を提供します。VPN 接続の文脈では、AnyConnect リモートアクセス VPN やサイト間 VPN セッションをクラスタ全体で分散・維持する「VPN 高可用性（High Availability）」を実現するために利用されます。

---

## 📘 概要

*   **機能概要**: 複数の物理 ASA を 1 つの仮想デバイスとして統合し、構成の同期、トラフィックの負荷分散、および障害時のステートフルなフェールオーバーを実現します。
*   **利用目的**: 単一の ASA では処理しきれない膨大な VPN セッション数への対応、および特定ユニット障害時でも通信を断絶させない可用性の確保。
*   **どのような場面で利用するか**: 数千〜数万規模の AnyConnect ユーザーを収容する大規模な企業ハブ、またはミッションクリティカルな拠点間 VPN 通信の集約ポイント。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **最大構成台数** | 最大 16 ユニット。 |
| **主な動作モード** | **Individual Interface**（各 ASA が個別 IP 保持）または **Spanned Interface**（EtherChannel を共有）。 |
| **VPN サポート** | AnyConnect (SSL/IKEv2)、Site-to-Site (IKEv1/IKEv2)。 |
| **制御プレーン** | Master ユニットが構成を管理し、Slave ユニットへ同期する。 |
| **データプレーン** | トラフィックは Cluster Control Link (CCL) を介して適切にリダイレクトされる。 |
| **ライセンス要件** | 全ユニットで同一のライセンス（Strong Encryption 等）が必要。 |

---

## 🏗 動作原理

ASA クラスタリングは、**Cluster Control Link (CCL)** を通じて状態情報を交換します。VPN トラフィックがクラスタに到着すると、ハッシュアルゴリズムに基づいて特定のユニット（Director）がオーナーとして選ばれ、セッションステートがバックアップユニット（Backup Owner）に同期されます。

```text
[ Remote Client ]
       ↓ (Internet)
[ Load Balancer / External Switch ]
       ↓ (Traffic Distribution)
[ ASA Cluster (Master/Slave 1...n) ]
       ↓ (CCL: State Sync & Forwarding)
[ Internal Network ]
```

---

## ⚙ 動作シーケンス

1.  **クラスタ参加**: 各ユニットがブート時に Master を選出し、構成を同期する。
2.  **VPN セッション確立**: クライアントからの接続が特定のユニットに到着。Master または Director がセッションを収容するユニットを決定。
3.  **ステート同期**: 確立された VPN トンネル情報、暗号化キー、およびカウンタ情報が、CCL を介して別のユニットに同期される。
4.  **障害検知と切り替え**: オーナーユニットがダウンした場合、バックアップユニットが即座にパケット処理を引き継ぎ、VPN トンネルを再確立なしで維持する。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **CCL の設定**: クラスタリングが機能するための最優先事項です。専用インターフェイスを指定し、`clp`（Cluster Liaison Protocol）が正常に動作しているか確認が求められます。
*   **AnyConnect 設定の同期**: 接続プロファイル、グループポリシー、および AnyConnect イメージファイルが全ユニットで共有されていることを確認します。
*   **IP アドレスの設計**: Individual インターフェイスモードの場合、各ユニットの VPN 公開 IP がどのように外部から見えるか（外部バランサとの関係）を把握する必要があります。
*   **MTU の調整**: クラスタヘッダーによるオーバーヘッドを考慮し、CCL インターフェイスの MTU を物理データインターフェイスより大きく（最低 +100 バイト）設定することが重要です。
*   **トラブルシュートコマンドの活用**: `show cluster info` や `show crypto ipsec sa` の出力を読み解き、セッションがどのユニットに所属しているか特定する能力が問われます。

---

## 🛠 設定方法

### 1. クラスタの基本構成 (Master 側)
```bash
cluster group MY-CLUSTER
  key cisco123
  master-priority 100
  clp-port 12345
  enable
!
interface GigabitEthernet0/2
  description Cluster Control Link
  cluster-link
  no shutdown
```

### 2. VPN インターフェイスの設定 (Spanned モード例)
```bash
interface Port-channel1
  port-channel load-balance vlan-dst-ip
  speed 1000
  duplex full
!
interface GigabitEthernet0/0
  channel-group 1 mode active
!
interface Port-channel1.10
  vlan 10
  nameif outside
  security-level 0
  ip address 203.0.113.1 255.255.255.0 cluster-pool POOL-OUT
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **クラスタ全体の状態確認** | <code>show cluster info</code> |
| **インターフェイスの稼働状況** | <code>show cluster interface-mode</code> |
| **VPN セッションの分散確認** | <code>show vpn-sessiondb remote</code> |
| **暗号化 SA の同期確認** | <code>show crypto ipsec sa</code> |
| **クラスタ制御通信のデバッグ** | <code>debug cluster flow</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| ユニットがクラスタに参加できない | ライセンスやバージョンの不一致 | 全ユニットの OS とライセンスを揃える。 |
| VPN トラフィックが断続的に途切れる | CCL の帯域不足または MTU ミス | CCL インターフェイスの MTU を増やし、帯域を確保する。 |
| 設定が Slave に同期されない | Master ユニットの選出失敗 | 優先度 (master-priority) を確認し、再起動を試みる。 |
| 特定ユニットにセッションが集中する | 外部バランサのアルゴリズム不備 | 外部 L4 バランサの分散方式（ソースIPハッシュ等）を再考する。 |

---

## ⚠ 制限事項

*   **Clientless SSL VPN**: クラスタリング環境では一部の高度な機能がサポートされない、または制限される場合があります。
*   **ユニット数制限**: VPN 機能を完全に利用する場合、モデルによって実効スループットのスケール効率が 100% にならないことがあります。
*   **ハードウェアの同一性**: すべてのノードは同一のハードウェアモデルである必要があります。

---

## 🔄 他技術との関連

*   **High Availability**: クラスタリングはアクティブ/アクティブ HA の究極の形態です。
*   **Routing**: クラスタ外との通信には、Equal-Cost Multi-Path (ECMP) または静的ルートが必要です。
*   **AnyConnect**: リモートアクセス VPN のクライアント側はクラスタを意識せず接続します。

---

## 🧩 比較表

### Standard HA (Active/Standby) vs Clustering (Active/Active)

| 特徴 | Active/Standby HA | ASA Clustering |
| :--- | :--- | :--- |
| **稼働台数** | 2台固定 | 最大16台 |
| **スループット** | 1台分のみ | 全台の合算に近い性能 |
| **構成の複雑さ** | 低い | 高い（CCL 設計が必要） |
| **VPN 継続性** | ステートフル引継ぎ | 高度な Director リダイレクト |

---

## 💡 ベストプラクティス

1.  **専用 CCL インターフェイス**: データ通信と CCL を物理的に分離し、輻輳を防ぎます。
2.  **同一構成の徹底**: NTP、DNS、認証サーバー設定を Master で一括管理し、同期を確認します。
3.  **監視の強化**: SNMP や Syslog を使用して、Master ユニットのヘルスチェックを常時行います。

---

## 📝 ラボ学習・設定サンプル例

### 1. CCL インターフェイスの有効化
*   **要件**: Gi0/2 をクラスタ制御用に設定せよ。
*   **設定**: `interface Gi0/2`, `cluster-link`。

### 2. クラスタグループの定義
*   **要件**: グループ名 "CCIE-CL"、パスワード "lab123" でクラスタを起動せよ。
*   **設定**: `cluster group CCIE-CL`, `key lab123`, `enable`。

### 3. VPN IP プールの定義
*   **要件**: 各ユニットに個別の VPN IP アドレスを割り当てるための IP プールを作成せよ。
*   **設定**: `ip local pool vpn_pool 10.1.1.1-10.1.1.100`。

### 4. サイト間 VPN の同期テスト
*   **要件**: 1 台の ASA をシャットダウンしても、対向拠点への VPN が維持されることを確認せよ。

### 5. Master ユニットの手動切り替え
*   **要件**: 特定の Slave ユニットを Master に昇格させよ。
*   **設定**: `cluster group` 内での `master-priority` 調整。

### 6. CCL の MTU 調整
*   **要件**: 内部フラグメンテーションを防ぐため MTU を 1600 に設定せよ。

### 7. AnyConnect バナーの同期
*   **要件**: 全ユニットで共通の VPN ログインバナーを表示させよ。

### 8. クラスタ内パケットリダイレクトの監視
*   **課題**: `show asp cluster counter` でリダイレクト数を監視せよ。

### 9. 外部バランサとの連携設定
*   **要件**: Individual モードで各ユニットの Outside IP をバランサに登録せよ。

### 10. ライセンスの不一致解消
*   **課題**: 全ユニットに Strong Encryption (3DES/AES) ライセンスが適用されているか `show version` で確認せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `show cluster info` で `Role: Slave` と表示されているユニットで `conf t` は可能か？
    *   **回答**: 不可能。構成変更は Master ユニットでのみ行い、Slave へ同期されます。
2.  **トラブルシュート**: クラスタに参加後、VPN クライアントからの接続が拒否される。CCL のステータスは `Up` である。他に確認すべき VPN 固有の設定は？
    *   **回答**: 認証サーバー（ISE/RADIUS）への接続性が全ユニットから確保されているか確認します。
3.  **Design**: 4 台の ASA でクラスタを構成する際、CCL に最低限必要な帯域幅の目安は？
    *   **回答**: 一般的に、データトラフィックの約 10-20% の帯域が制御用に必要とされます。
4.  **実装**: Individual モードで AnyConnect を提供する場合、各ユニットに個別の IP を持たせる必要があるか？
    *   **回答**: はい。各ユニットが固有の IP を持ち、外部バランサがそれらを宛先として配布する設計が一般的です。
5.  **コンフィグ読解**: `clp-port 12345` の役割は？
    *   **回答**: ユニット間でクラスタ制御メッセージを交換するための UDP ポート番号を定義します。

---

## 🔗 参考リソース

*   [Cisco ASA Series CLI Configuration Guide, 9.4 - Clustering](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/general/asa-94-general-config/ha-cluster.html)
*   [Cisco ASA VPN Configuration Guide, 9.4 - VPN in Clustering](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/vpn/asa-94-vpn-config/vpn-ha.html)
*   [Integrated Security Technologies and Solutions, Volume II (Cisco Press)](https://www.ciscopress.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「クラスタリング = 拡張性 + 冗長性」です。VPN 環境では、セッションのバックアップ（Director/Backup 概念）がどのように行われるかを、ソース を中心に読み込むことが重要です。
*   **注意点**: ラボ試験では、物理接続のミス（CCL のケーブル挿し間違い等）が致命的です。物理トポロジーと `cluster-link` 指定インターフェイスの一致を最初に見直してください。
