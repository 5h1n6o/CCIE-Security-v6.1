---
layout: default
title: 3.4.b-IPDT
nav_order: 2
parent: 3.4-L2-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.4.b IPDT (IP Device Tracking)

**IPDT (IP Device Tracking)** は、Ciscoスイッチに接続されているデバイスのIPアドレスとMACアドレスの情報を収集・管理するための機能です。CCIE Securityラボ試験においては、特に **Cisco ISE (Identity Services Engine)** との連携において、接続端末のプロファイリングや状態監視を行うための基盤技術として非常に重要です。

近年、Cisco IOS-XEでは従来のレガシーなIPDTコマンドから、より柔軟な **SISF (Switch Integrated Security Features)** ベースのデバイス追跡へと統合が進んでいます。

---

## 📘 概要

*   **機能概要**: スイッチポートを通過するARP、DHCP、およびIPv6隣接探索（ND）パケットを監視し、ポート、MACアドレス、IPアドレスのバインディング（紐付け）テーブルを作成・維持します。
*   **利用目的**: ネットワークに「誰が」「どこに」接続されているかをリアルタイムに可視化し、認証（802.1X）やセキュリティポリシー（SGT/TrustSec）の適用を支援します。
*   **利用場面**: Cisco ISEによる端末プロファイリング、IP Source Guardの動作、Web認証（WebAuth）のリダイレクト処理、およびネットワーク管理システムへの情報提供に利用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **収集情報** | IPアドレス、MACアドレス、接続インターフェイス、VLAN。 |
| **学習方法** | ARPスニッフィング、DHCPスヌーピング、IPv6 NDスニッフィング。 |
| **レガシー方式** | <code>ip device tracking</code> コマンドによる全ポート一括有効化。 |
| **SISF方式** | <code>device-tracking policy</code> を作成し、ポート単位で適用する柔軟な方式。 |
| **プローブ** | 端末の生存確認のため、スイッチからARPプローブ（送信元IP 0.0.0.0）を送信する。 |
| **メリット** | ISE等の管理プラットフォームに対し、正確なエンドポイント情報を提供できる。 |
| **制限事項** | Windows端末など、ARPプローブに反応して重複IP警告を出す場合がある（要調整）。 |

---

## 🏗 動作原理

IPDTは受動的な監視と能動的な確認の組み合わせで動作します。

```text
[ Endpoint ] 
      ↓ (ARP / DHCP / IPv6 ND)
[ Access Switch (IPDT/SISF) ]
      ↓ 
[ Device Tracking Database ] <--- (IP/MAC/Port/VLAN) 
      ↓ (Radius / HTTP / SNMP)
[ Cisco ISE / NMS ]
```

1.  **スニッフィング**: デバイスが通信を開始すると、スイッチはそのパケット（ARPやDHCP）からIPとMACのペアを抽出し、データベースに登録します。
2.  **データベース保持**: 端末が静的にIPを設定している場合や、DHCPリース期間中、情報を保持します。
3.  **生存確認（Probe）**: 端末が長時間無通信の場合、スイッチはARPプローブを送信し、まだ端末が存在するかを確認します。

---

## ⚙ 動作シーケンス

1.  **初期学習**: ポートがUPし、端末がDHCP要求またはARP送信を行う。
2.  **テーブル登録**: スイッチが情報をデータベースに書き込む（`show ip device tracking all` 等で確認可能）。
3.  **ISE通知**: ISEと連携している場合、RADIUS会計（Accounting）パケットの `framed-ip-address` 属性に収集したIPを含めて送信する。
4.  **定期監視**: 設定されたインターバルで隣接関係を確認。
5.  **エントリ削除**: 端末の切断（Link Down）またはタイムアウトにより情報を削除する。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **ISE プロファイリング要件**: 「ISEがエンドポイントのIPアドレスを学習できるようにスイッチを設定せよ」という問題が出た場合、IPDTの設定が必須です。
*   **SISF (Device-tracking policy) の使用**: 最新の試験（v6.1）では、`ip device tracking` よりも `device-tracking policy` を作成し、インターフェイスに `device-tracking attach-policy` で適用する新しい構文が好まれる傾向にあります。
*   **重複IP警告の回避**: Windows端末が「IP 0.0.0.0」からのARPプローブを自身のIPに対する攻撃と誤認し、重複IPエラーを出す問題はCCIEラボでの定番トラブルです。`device-tracking probe delay` を設定して、端末のリンクアップ直後のプローブを遅らせる対策が求められます。
*   **IPv6環境**: IPv6デバイスの追跡には、明示的に `device-tracking` ポリシー内で IPv6 のプロトコル学習を有効にする必要があります。

---

## 🛠 設定方法

### 1. レガシー方式（古いIOSバージョン）
```bash
ip device tracking
! 全ポートで有効化される
```

### 2. SISF方式（IOS-XE 推奨設定）
ポリシーを定義し、特定のインターフェイスに適用します。
```bash
! デバイス追跡ポリシーの作成
device-tracking policy ISE-POLICY
 tracking port
 device-role node
!
! インターフェイスへの適用
interface GigabitEthernet1/0/1
 device-tracking attach-policy ISE-POLICY
```

### 3. 重複IP警告対策 (Probe Delay)
```bash
device-tracking policy ISE-POLICY
 tracking port
 device-role node
 ! リンクアップから10秒間プローブを待機させる
 limit-address-count 10
!
! グローバル設定（プローブ送信元IPの指定が必要な場合）
ip device tracking probe use-svi
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **全追跡エントリの表示** | <code>show device-tracking database</code> |
| **レガシー方式の確認** | <code>show ip device tracking all</code> |
| **ポリシー適用状態の確認** | <code>show device-tracking interface [ID]</code> |
| **特定のIPを検索** | <code>show device-tracking database \| include [IP]</code> |
| **デバッグ** | <code>debug device-tracking [events\|packet]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 端末で「IPアドレス競合」が出る | スイッチのARPプローブを誤認 | <code>show log</code> | <code>device-tracking probe delay</code> を設定。 |
| ISEで端末のIPが表示されない | IPDTが未設定、またはAccounting不備 | <code>show device-tracking database</code> | IPDTを有効化し、RADIUS Accountingを設定。 |
| 特定ポートで学習されない | ポリシーがアタッチされていない | <code>show run interface</code> | <code>device-tracking attach-policy</code> を確認。 |
| 学習エントリがすぐに消える | タイムアウト値が短すぎる | <code>show device-tracking policy [Name]</code> | <code>tracking timeout</code> を調整。 |

---

## ⚠ 制限事項

*   **スイッチのキャパシティ**: ハードウェアリソースに依存するため、数千の端末を追跡する場合はメモリ消費に注意が必要です。
*   **SVIsの必要性**: 複数のVLANを跨ぐ場合、ARPプローブの送信元IPとして有効なSVI（VLANインターフェイス）のIPが必要になることがあります。
*   **非IPトラフィック**: 純粋なL2パケット（IPスタックを持たないデバイス）はIPDTで追跡できません。

---

## 🔄 他技術との関連

*   **3.4.e DHCP Snooping**: IPDTはDHCPスヌーピングのバインディングテーブルから情報を参照できます。
*   **Cisco ISE (Profiling)**: ISEがHTTPやSNMPでスイッチに問い合わせる際、IPDTデータベースがソースになります。
*   **IP Source Guard**: 収集されたIP/MACバインディングを使用して、偽装IPパケットをフィルタリングします。

---

## 🧩 比較表

### Legacy IPDT vs SISF Device Tracking

| 特徴 | Legacy IPDT | SISF (New) |
| :--- | :--- | :--- |
| **設定単位** | グローバル (All or Nothing) | ポリシーベース (ポート単位) |
| **プロトコル制御** | ARP中心 | ARP, DHCP, ND (IPv6) を細かく制御 |
| **柔軟性** | 低い | 高い（ロール別の設定が可能） |
| **推奨OS** | 古いIOS | IOS-XE 16.x 以降 |

---

## 💡 ベストプラクティス

1.  **ISE導入時は必須**: ISEによる動的なアクセス制御を行うなら、必ず全アクセスポートでIPDT（SISF）を有効にします。
2.  **Probe Delayの設定**: Windowsクライアントが存在する環境では、一律で `device-tracking probe delay 10` 以上を設定することを推奨します。
3.  **DHCP Snoopingとの併用**: DHCP環境では、DHCP Snoopingを有効にすることでIPDTの学習精度と信頼性が劇的に向上します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的なデバイス追跡の有効化
*   **要件**: ポート Gi1/0/1 でデバイス追跡を開始せよ。
*   **設定**: `device-tracking policy P1` -> `tracking port` -> `interface Gi1/0/1` -> `device-tracking attach-policy P1`

### 2. IPv6デバイスの学習
*   **要件**: IPv6 NDパケットからデバイスのIPv6アドレスを学習せよ。

### 3. 重複IP警告の防止 (ISE連携用)
*   **要件**: 管理者が接続した際に、PC側でIP競合エラーが出ないように対策せよ。
*   **設定**: ポリシー内で `destination-glean log` や `tracking port` と併せて `probe delay 20` を設定。

### 4. 特定VLANのみの学習制限
*   **要件**: VLAN 100 に所属するポートのみを追跡対象とせよ。

### 5. 静的IPデバイスの強制登録
*   **要件**: 通信を行わない静的IPプリンタ（10.1.1.50）をデータベースに強制登録せよ。
*   **設定**: `ip source binding 0011.2233.4455 vlan 10 10.1.1.50 interface Gi1/0/10`

### 6. 学習上限の設定
*   **要件**: 単一ポートで最大 5 つのIPまでしか学習させないようにせよ。

### 7. トランクポートの除外
*   **要件**: Uplink（Trunk）ポートではデバイス追跡を行わないようにせよ（デフォルト動作だが、明示的なポリシー制御）。

### 8. プローブ送信元IPの固定
*   **要件**: スイッチが送信するARPプローブの送信元IPを特定のSVI IPに固定せよ。

### 9. データベースのクリア
*   **要件**: 学習済みの古い情報をすべてリセットせよ。
*   **コマンド**: `clear device-tracking database all`

### 10. RADIUS Accounting へのIP含浸
*   **要件**: 認証成功後、RADIUS会計パケットに学習したIPアドレスを載せてISEに送信せよ。
*   **設定**: `aaa accounting update periodic` 等と併用。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: スイッチで `show device-tracking database` を実行したが、接続されているPCのIPが表示されない。DHCP Snoopingは有効で、PCは正常にIPを取得している。考えられる原因は？
    *   **回答**: デバイス追跡ポリシーが該当インターフェイスにアタッチされていない、またはポリシー内で `tracking port` が設定されていない。
2.  **Design**: ネットワーク内のセキュリティを強化するため、IP Source Guardを導入したい。前提として設定しておくべき機能は？
    *   **回答**: **DHCP Snooping** および **IPDT (Device Tracking)**。
3.  **コンフィグ読解**: `device-tracking policy P1` 下の `device-role node` と `device-role switch` の違いは？
    *   **回答**: `node` はエンドポイントが接続されるポート用、`switch` は他のスイッチやルータが接続されるポート（信頼境界）用。
4.  **実装**: Windows PCがリンクアップの度に「IPアドレスが重複しています」とポップアップが出る。スイッチ側で即座に設定すべき項目は？
    *   **回答**: `device-tracking policy` における `tracking port` の `probe delay` 設定。
5.  **Design**: ISEが特定のプロファイリング（例：OSの種類特定）を行うために、スイッチから提供されるべき情報は？
    *   **回答**: デバイスのMACアドレスに対応する **IPアドレス** (IPDT経由)。

---

## 🔗 参考リソース

*   **Cisco Configuration Guide**
    *   [Configuring Switch Integrated Security Features (SISF)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/16-12/configuration_guide/sec/b_1612_sec_9300_cg/configuring_sisf_device_tracking.html)
*   **Technical Notes**
    *   [IP Device Tracking (IPDT) Explained](https://www.cisco.com/c/en/us/support/docs/ip/address-resolution-protocol-arp/118777-technote-ipdt-00.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [Identity and Device Tracking in Modern Campus Networks](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「IPDTはスイッチの中にあるIP/MAC名簿」です。ISEはこの名簿を読み取って、誰がどのポリシーで通信すべきかを決定します。
*   **図解**: 
    1. 端末送信 (DHCP/ARP) 
    2. スイッチで抽出 (Snooping) 
    3. 名簿登録 (DB) 
    4. ISEへ報告 (Radius Attribute 8)
*   **注意点**: ラボ試験では `ip device tracking` という古いコマンドが効かない（または推奨されない）場合が多いため、必ず `device-tracking policy` の構文に慣れておいてください。
