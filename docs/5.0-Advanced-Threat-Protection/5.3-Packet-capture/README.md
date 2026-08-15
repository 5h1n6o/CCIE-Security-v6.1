---
layout: default
title: 5.3-Packet-capture
nav_order: 3
parent: 5.0-Advanced-Threat-Protection
---

# 5.3 Perform packet capture and analysis using Wireshark, tcpdump, SPAN, ERSPAN, and RSPAN

パケットキャプチャと分析は、ネットワークセキュリティにおけるトラブルシューティングの「最終兵器」です。CCIE Security v6.1 では、スイッチングレベルのミラーリング（SPAN/ERSPAN）から、ASAやFTDといったファイアウォール内部でのキャプチャ、そしてWiresharkを用いた詳細なプロトコル解析まで、幅広いスキルが求められます。

---

## 📘 概要

*   **機能概要**: ネットワークを流れる生データ（フレーム/パケット）を複製または直接採取し、プロトコルの挙動やヘッダー情報を詳細に確認する機能です。
*   **利用目的**: 通信障害の特定（どこでパケットが消えたか）、不正アクセスのフォレンジック、暗号化ハンドシェイク（SSL/VPN）のデバッグ。
*   **どのような場面で利用するか**:
    *   **スイッチング環境**: 特定ポートのトラフィックをモニターポートへ転送する（SPAN）。
    *   **L3ネットワーク超え**: 遠隔地のトラフィックをGREトンネルで飛ばして集約・解析する（ERSPAN）。
    *   **ファイアウォール内部**: ACLやインスペクションによるドロップ原因を「デバイス内部の視点」で特定する。

---

## 🔑 要点

| 項目 | SPAN | RSPAN | ERSPAN | ASA Capture | FTD tcpdump |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **特徴** | 同一スイッチ内のミラーリング | 特定VLANを介したL2ミラーリング | GREを使用したL3ミラーリング | デバイス内のキャプチャ/トレース | Snort/Linaエンジンでの採取 |
| **用途** | ローカルでの迅速な確認 | 同一拠点内の別スイッチ監視 | 拠点間をまたぐパケット収集 | 転送処理プロセスの可視化 | 高度なアプリ層解析 |
| **メリット** | 設定が極めて単純 | 物理的な接続変更が不要 | ルーティング環境でも利用可能 | ドロップ理由の特定に強力 | Linuxライクな柔軟なフィルタ |
| **デメリット** | 物理的に近くにいる必要がある | RSPAN VLAN管理が必要 | 送信元デバイスにCPU負荷 | バッファサイズに制限がある | Lina/Snortの切り分けが必要 |
| **対応機種** | ほぼ全てのCatalyst/Nexus | L2スイッチ全般 | ハイエンドスイッチ/ルータ | ASA 5500-X, ASAv | FTDアプライアンス, FTDv |

---

## 🏗 動作原理

パケットキャプチャは、トラフィックのパス（Data Plane）からデータをコピーして、解析エンジン（Management/Control Plane または 外部PC）へ渡すことで動作します。

```text
[ Source Device ]                     [ Monitoring Station ]
      |                                        |
 (Traffic Flow)                                |
[ Port/VLAN ] --(Copy/Encapsulate)--> [ Monitor Port/PC ]
      |                                        |
      ↓                                        ↓
 (Normal Forwarding)                   (Wireshark / tcpdump)
```

---

## ⚙ 動作シーケンス

1.  **トラフィックの特定**: 監視対象（送信元）のインターフェイス、VLAN、またはACLによるフローを定義します。
2.  **複製**: デバイスのASICまたはCPUがパケットをコピーします。
3.  **転送**:
    *   **SPAN**: 直接宛先ポートへ。
    *   **ERSPAN**: GREヘッダーを付加し、IPネットワーク経由で宛先IPへ送信。
4.  **採取**: 解析デバイス（ASA captureなど）では、内部メモリ（Buffer）に保存されます。
5.  **分析**: 保存された `.pcap` ファイルを Wireshark で開き、シーケンス番号やフラグを確認します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **ASA `capture` の `trace` オプション**: これは試験で最も重要なツールの一つです。パケットが「なぜドロップされたのか」（ACLか、インスペクションか、NATミスか）をパケットごとに追跡できます。
*   **ERSPAN の ID 一致**: 送信元（Source Session）と宛先（Dest Session）で **ID が一致**していないと通信が成立しません。
*   **フィルタリング**: 試験では膨大なパケットが流れます。`access-list` を使用して、対象の IP/ポートのみをキャプチャする設定が必須です。
*   **ISAKMP キャプチャ**: VPN トラブルシュートにおいて、`capture IKE type isakmp` を使用してフェーズ1のメインモード/アグレッシブモードのやり取りを可視化する能力が問われます。
*   **Wireshark 読解**: 採取したログから、TCP 再送（Retransmission）や TLS バージョンの不一致を見抜く Design/Troubleshoot 問題に備えます。

---

## 🛠 設定方法

### 1. Cisco ASA：パケットキャプチャとトレース (CLI)
特定ホスト間の ICMP トラフィックを追跡する例です。

```bash
! フィルタ用 ACL 作成
access-list CAP_ACL extended permit icmp host 10.1.101.1 host 10.1.102.2

! キャプチャ開始 (trace オプションを付加)
capture ISSUE type raw-data access-list CAP_ACL trace interface inside

! 内容の確認
show capture ISSUE

! 特定パケットの処理プロセスを追跡
show capture ISSUE packet 1 trace
```

### 2. IOS スイッチ：ERSPAN 送信元の構成
```bash
! 送信元セッションの定義
monitor session 1 type erspan-source
 source interface GigabitEthernet0/1
 destination
  erspan-id 100
  ip address 192.168.1.100  ! 宛先解析端末
  origin ip address 192.168.1.1
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **ASA キャプチャ一覧表示** | <code>show capture</code> |
| **ASA キャプチャの中身表示** | <code>show capture [name]</code> |
| **ASA キャプチャのデコード表示** | <code>show capture [name] decode</code> |
| **スイッチ ミラーリング状態確認** | <code>show monitor session [ID]</code> |
| **FTD 診断 CLI でのキャプチャ** | <code>system support packet-capture</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| キャプチャにパケットが載らない | ACL 設定のミス | `show access-list` でヒットカウンタを確認。 |
| ERSPAN パケットが届かない | MTU サイズ超過 | GRE カプセル化によるオーバーヘッドを考慮し、パス MTU を確認。 |
| キャプチャがすぐに止まる | バッファフル | `circular` オプションを使用して上書き保存を有効にする。 |
| 宛先ポートで通信不能になる | モニターポートの制約 | 宛先ポートは通常、通常のトラフィック転送を停止します（`ingress vlan` 設定等で回避可能）。 |

---

## ⚠ 制限事項

*   **ASA の CPU 負荷**: `capture` は CPU 処理を伴うため、高負荷な実環境での全トラフィックキャプチャは推奨されません。
*   **暗号化通信の限界**: キャプチャはあくまで「パケットの受け渡し」を見るものであり、SSL/IPsec のペイロード自体は Wireshark 上でも解読に秘密鍵が必要です。
*   **ハードウェア制限**: プラットフォームによって、同時に実行できる monitor session 数には制限があります。

---

## 🔄 他技術との関連

*   **2.0 Firewall**: ACL や Inspection の動作確認に不可欠。
*   **1.0 VPN**: ISAKMP/ESP パケットの欠落を調査。
*   **3.6 Monitoring**: NetFlow や Syslog では見えない「実際のペイロード」を確認するために併用。

---

## 🧩 比較表

### キャプチャ手法の使い分け

| シナリオ | 推奨ツール | 理由 |
| :--- | :--- | :--- |
| スイッチ間の疎通確認 | SPAN / RSPAN | 最も低負荷で生データを確認可能。 |
| ASA 経由のドロップ調査 | ASA Capture (Trace) | デバイスの論理判断（ACL等）を同時に見れる。 |
| 拠点をまたぐ不審通信分析 | ERSPAN | 遠隔地のトラフィックを SOC 等に集約できる。 |
| 複雑なアプリ層のデバッグ | Wireshark (.pcap) | 強力なデコード機能とフィルタリング。 |

---

## 💡 ベストプラクティス

1.  **ACL による最小化**: 必ず `access-list` でキャプチャ対象を絞り、不要なログでバッファを埋めないようにします。
2.  **`packet-tracer` との併用**: ASA の場合、実際のパケットを流す前に `packet-tracer` でシミュレーションし、その後に `capture` で実挙動を裏取りします。
3.  **コピーは最短パスで**: SPAN は可能な限りトラフィックの発生源に近いスイッチで実施し、L2/L3 の中継による変化を最小限にします。

---

## 📝 ラボ学習・設定サンプル例

### 1. ASA 基本キャプチャ
*   **要件**: Outside インターフェイスを通過する全 ICMP をキャプチャせよ。
*   **設定**: `capture ICMP-O access-list TEST interface outside`.

### 2. ASA ISAKMP 解析
*   **要件**: VPN 接続時のフェーズ1ネゴシエーションを採取せよ。
*   **設定**: `capture VPN-IKE type isakmp interface outside`.

### 3. ASA トレース分析
*   **要件**: キャプチャしたパケット 5 番がどの ACL で拒否されたか示せ。
*   **検証**: `show capture ISSUE packet 5 trace`.

### 4. ERSPAN 構成
*   **要件**: Gi0/1 の入力を 10.1.1.100 へ ERSPAN ID 50 で送信せよ。

### 5. RSPAN 設定
*   **要件**: VLAN 999 を RSPAN 用に使用し、複数スイッチ間でモニターせよ。

### 6. FTD `tcpdump` の実行
*   **要件**: FTD の `inside` インターフェイスでホスト 1.1.1.1 をモニターせよ。
*   **コマンド**: `system support diagnostic-cli` > `tcpdump -i inside host 1.1.1.1`.

### 7. キャプチャのファイルエクスポート
*   **要件**: キャプチャ `ISSUE` を TFTP で外部へ保存せよ。
*   **コマンド**: `copy /pcap capture:ISSUE tftp://10.1.1.1/issue.pcap`.

### 8. キャプチャバッファの拡張
*   **要件**: デフォルトのバッファサイズを 2MB に増加させよ。

### 9. プロトコル固有のデコード
*   **操作**: ASA CLI 上で `show capture [name] decode` を使い、HTTP ヘッダーを確認せよ。

### 10. Capture-with-Circular-Buffer
*   **要件**: ログを止めずに最新のパケットを常に保持し続けよ。
*   **設定**: `capture [name] ... buffer 512000 circular`.

---

## ❓ 想定試験問題

1.  **トラブルシュート**: ASA でパケットがドロップされているが、`show logging` には何も出ない。次に行うべきアクションは？
    *   **回答**: **`capture [name] trace`** を設定し、パケットレベルの処理ステップを確認する。
2.  **Design**: 地理的に離れた拠点のトラフィックを、中央の監視サーバで解析したい場合に最適な技術は？
    *   **回答**: **ERSPAN** (Encapsulated Remote SPAN)。
3.  **コンフィグ読解**: ASA の `capture` 設定において `type raw-data` と `type isakmp` の違いは？
    *   **回答**: `raw-data` は汎用的な L2/L3 情報、`isakmp` は VPN 制御パケットに特化して最適化された採取モード。
4.  **実装**: ネットワーク全体の負荷を抑えつつ、特定の VLAN トラフィックを 2 つの異なるスイッチポートにミラーリングする設定は可能か？
    *   **回答**: はい、1 つの送信元に対して複数の宛先（Destination Port）を持つ monitor session を構成する。
5.  **トラブルシュート**: Wireshark で IPsec の ESP パケットは見えているが、中身（HTTP等）が見えない。なぜか？
    *   **回答**: パケットが **ESP (Protocol 50) で暗号化**されているため。解析には IKE ハンドシェイクから得られた暗号鍵が必要。

---

## 🔗 参考リソース

*   **Cisco ASA 9.4 Configuration Guide**: [Configuring Packet Capture](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/general/asa-94-general-config/monitor-packet-capture.html)
*   **Cisco Live (BRKSEC-3020)**: [Troubleshooting Firepower Threat Defense](https://www.ciscolive.com/)
*   **CVD**: [Campus Network Management Design Guide (SPAN/RSPAN)](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/campus-network-management-design-guide.html)
*   **Technical Note**: [ASA Packet Capture with CLI and ASDM](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/118097-configure-asa-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「キャプチャは嘘をつかない」。設定が正しいはずなのに通信できない時は、必ず `capture` を仕掛けましょう。ASA の `trace` 機能は、ドロップした瞬間の ACL ID まで表示してくれます。
*   **図解**: 
    - SPAN = 鏡（反射）
    - RSPAN = 宅急便VLAN（配送）
    - ERSPAN = 国際郵便GRE（カプセル配送）
*   **注意点**: ラボ試験では、**キャプチャを止める（no capture ...）**のを忘れてデバイスのメモリを浪費し続けないよう、検証が終わったら速やかに削除する癖をつけてください。
