---
layout: default
title: 3.2-Management-plane
nav_order: 2
parent: 3.0-Security-Infrastructure
---

# 3.2 Management plane protection techniques

ネットワークデバイスの**マネジメントプレーン（管理面）**は、管理者がデバイスを監視、構成、および制御するために使用する通信経路です。このプレーンを保護することは、不正アクセスを防止するだけでなく、デバイスのリソース（CPU/メモリ）を保護し、攻撃を受けている最中でも管理アクセスを維持するために不可欠です。

本稿では、CCIE Security v6.1 ブループリントに基づき、CPU保護、メモリしきい値管理、およびデバイスアクセスのセキュリティについて詳述します。

---

## 📘 概要

*   **機能概要**: デバイス自体を宛先とする管理用トラフィック（SSH, SNMP, HTTP等）に対して、受信インターフェイスの制限、リソース使用率の監視、およびアクセス制御を適用する技術群です。
*   **利用目的**: 管理セッションの機密性と完全性の確保、DoS攻撃による管理不能状態の防止。
*   **利用場面**: 
    *   インターネット境界ルータで、外部からのSSH/SNMPを物理的に拒否する。
    *   CPUやメモリが枯渇しそうな場合に、重要度の低いプロセスを停止または制限して管理用シェルを確保する。
    *   特定の管理セグメント以外からのGUI/CLIアクセスを遮断する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **3.2.a CPU (MPP)** | **Management Plane Protection (MPP)** を使用して、管理プロトコルを特定の物理インターフェイスに限定する。 |
| **3.2.b Memory** | **Memory Thresholding** により、空きメモリが低下した際に通知を出し、プロセスの新規起動を制限する。 |
| **3.2.c Access** | SSH, HTTPS, VTY Access-class, AAA を用いたデバイスアクセスの要塞化。 |
| **主なコンポーネント** | <code>control-plane management</code>, <code>memory free low-threshold</code>, <code>line vty</code>。 |
| **メリット** | CPU/メモリ資源の保護、不正侵入の攻撃面（Attack Surface）の削減。 |
| **制限事項** | MPPは物理インターフェイスに依存。論理インターフェイス（Loopback等）での制限はCoPP等を併用。 |

---

## 🏗 動作原理

マネジメントプレーン保護は、パケットがデバイスのCPUに到達するまでの複数の「関門」として動作します。

```text
Incoming Management Packet (e.g., SSH)
   ↓
[ Physical Interface ] 
   ↓
[ MPP Filter ] <--- (3.2.a) 許可された物理IFか？
   ↓
[ CoPP / CPPr ] <--- (3.1.a) レート制限内か？
   ↓
[ Access-Class / ACL ] <--- (3.2.c) 許可された送信元IPか？
   ↓
[ AAA / SSH Engine ] <--- (3.2.c) 認証・認可は成功するか？
   ↓
[ CPU Processing ] <--- (3.2.b) 処理に必要なメモリはあるか？
```

---

## ⚙ 動作シーケンス

1.  **物理的検証 (MPP)**: パケットが到着したインターフェイスが、`control-plane management` でそのプロトコルに対して許可されているか確認します。
2.  **論理的検証 (ACL)**: `line vty` 等に適用された `access-class` により、送信元 IP アドレスをチェックします。
3.  **リソースチェック**: **SPD (Selective Packet Discard)** やメモリしきい値設定に基づき、現在の負荷状態でパケットを処理可能か判断します。
4.  **プロトコル処理**: SSH キーの交換や TLS ハンドシェイクが開始されます。
5.  **認証**: RADIUS/TACACS+ (ISE) またはローカルデータベースを使用してユーザーを確認します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **MPP (Management Plane Protection)**: 最も頻出です。「GigabitEthernet1 でのみ SSH と SNMP を許可し、他では一切の管理アクセスを禁止せよ」という要件に対し、`control-plane management` 設定を即座に行える必要があります。
*   **Selective Packet Discard (SPD)**: IPv6 環境での SPD 設定（`ipv6 spd mode aggressive`）が出題される可能性があります。これは、CPU 負荷が高い時に不整合なパケットを優先的に捨てる設定です。
*   **Memory Thresholding**: 特定の空きメモリ量（KB）を下回った際にログを出す設定（`memory free low-threshold`）が問われます。
*   **SSH の高度な設定**: 単なる有効化だけでなく、バージョン 2 の固定、タイムアウト、リトライ回数、および送信元インターフェイスの固定（`ip ssh source-interface`）までがセットで求められます。
*   **セルフ・ロックアウトの回避**: ラボ試験で MPP や Access-class を設定する際、誤って自分の接続を切断すると致命的です。必ずコンソール接続を確認するか、`reload in 10` 等の保険をかけてから設定してください。

---

## 🛠 設定方法

### 1. Management Plane Protection (MPP)
特定の物理インターフェイスでのみ管理プロトコルを許可します。
```bash
control-plane management
 management-interface GigabitEthernet0/1 allow ssh snmp https
 ! これにより、Gi0/1以外からのSSH/SNMP/HTTPSはドロップされる
```

### 2. Memory Thresholding
空きメモリが低下した際に通知を出し、新しいプロセス（管理セッション等）のメモリ割り当てを保護します。
```bash
! 空きメモリが20000KBを切ったら警告、10000KBを切ったら深刻
memory free low-threshold warning 20000
memory free low-threshold critical 10000
! プロセスごとのメモリ消費しきい値を設定
process memory threshold central 80
```

### 3. Securing Access (SSH/AAA)
```bash
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
!
line vty 0 4
 access-class 10 in
 transport input ssh
!
access-list 10 permit 192.168.1.0 0.0.0.255
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **MPPの適用インターフェイス確認** | <code>show management-interface</code> |
| **現在のCPU負荷とプロセス確認** | <code>show processes cpu sorted</code> |
| **メモリの空き容量としきい値確認** | <code>show memory free</code> |
| **SSHのセッションと設定確認** | <code>show ip ssh</code> / <code>show ssh</code> |
| **SPDの状態確認 (IPv6)** | <code>show ipv6 spd</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 正しいIPからSSHが拒否される | MPPにより当該IFで許可されていない | <code>show management-interface</code> | <code>control-plane management</code>にIFを追加。 |
| メモリ不足で設定保存が失敗 | <code>Memory threshold</code> 到達 | <code>show memory statistics</code> | 不要なプロセス（debug等）を停止。 |
| 高負荷時に管理応答が極端に遅い | 優先パケットの余裕（headroom）不足 | <code>show ipv6 spd</code> | <code>spd extended-headroom</code> を増やす。 |
| 外部からHTTPS(GUI)が見えない | <code>ip http secure-server</code>設定漏れ | <code>show ip http server status</code> | HTTPS サーバーを有効化しIFをMPPで許可。 |

---

## ⚠ 制限事項

*   **MPP と Loopback**: MPP は物理インターフェイス（Ingress）を対象とするため、Loopback インターフェイスに届くパケットを制限するには、その Loopback に到達するために通過する物理インターフェイス側で制限をかける必要があります。
*   **ハードウェアサポート**: MPP はすべてのプラットフォームでサポートされているわけではありません（一部の Catalyst 旧モデル等）。
*   **アウトオブバンド管理**: コンソールポート（Line con 0）はマネジメントプレーン保護（MPP等）の影響を受けません。常に最終的なアクセス手段として確保されます。

---

## 🔄 他技術との関連

*   **CoPP (3.1.a)**: MPP が「インターフェイス」を制限するのに対し、CoPP は「流量（レート）」を制限します。
*   **Infrastructure Hardening (3.1)**: IP Source Routing の無効化（`no ip source-route`）や不要サービスの停止は、マネジメントプレーン保護の前提条件です。
*   **SNMP (3.6.b)**: SNMP トラフィック自体を MPP で保護しつつ、NMS へのトラフィックを `snmp-server host` で設定します。

---

## 🧩 比較表

### MPP vs VTY Access-class

| 特徴 | MPP (Management Plane Protection) | VTY Access-class |
| :--- | :--- | :--- |
| **制御対象** | 受信物理インターフェイス | 送信元 IP アドレス |
| **レイヤ** | レイヤ 2/3 (入力時) | レイヤ 4/7 (プロトコル処理時) |
| **保護の深さ** | 高い（CPU 処理の手前で落とせる） | 標準的（SSH 等の起動後に判定） |
| **推奨の使い分け** | 入口を物理的に縛る場合に最適 | どこからでも届くが特定の IP のみ許可する場合 |

---

## 💡 ベストプラクティス

1.  **専用管理 IF の利用**: 運用ネットワーク（Data）とは別の、物理的に独立した管理用ポート（Mgmt0 等）でのみ SSH を許可します。
2.  **IP Options Drop**: CPU 負荷を軽減するため、IP オプション付きパケットをドロップします。
3.  **Logging Interval**: ログが CPU を食いつぶさないよう、`logging rate-limit` を適切に設定します。
4.  **AAA 冗長化**: ローカルユーザーはバックアップとして残しつつ、メインの認証は TACACS+ (ISE) に委ねます。

---

## 📝 ラボ学習・設定サンプル例

### 1. MPP による SSH 到着インターフェイスの制限
*   **問題**: ルータ R1 において、GigabitEthernet0/1 からの SSH アクセスのみを許可せよ。
*   **設定**: 
    ```bash
    control-plane management
     management-interface GigabitEthernet0/1 allow ssh
    ```

### 2. IPv6 SPD による高負荷時の保護
*   **要件**: IPv6 パケットが滞留した際、不整合パケットをアグレッシブにドロップせよ。
*   **設定**: 
    ```bash
    ipv6 spd mode aggressive
    ipv6 spd queue min-threshold 80
    ipv6 spd queue max-threshold 100
    spd extended-headroom 20
    ```

### 3. メモリしきい値の警告設定
*   **要件**: 空きメモリが 50MB 以下になったら警告ログを生成せよ。
*   **設定**: `memory free low-threshold warning 51200`

### 4. SSH v2 への固定と送信元固定
*   **要件**: SSH v2 のみを使用し、送信元 IP を Loopback0 に固定せよ。
*   **設定**: `ip ssh version 2`, `ip ssh source-interface Loopback0`

### 5. SNMP マネジメントインターフェイスの追加
*   **要件**: 既存の MPP 設定に SNMP プロトコルを追加せよ。

### 6. 不要な管理サービスの無効化
*   **要件**: HTTP サーバーおよび Finger サービスを無効化せよ。

### 7. VTY 行のタイムアウトとアクセス制限
*   **要件**: VTY 0-4 において、10分間無通信なら切断し、10.1.1.0/24 からのみ接続を許可せよ。

### 8. ローカルユーザーの特権レベル設定
*   **要件**: ユーザー "admin" を作成し、ログイン後即座に特権モード (level 15) になるようにせよ。

### 9. ログイン試行のレート制限
*   **要件**: ブルートフォース対策として、ログイン失敗時の待機時間を設定せよ。
*   **設定**: `security authentication failure rate 3 log`

### 10. コンソールログのバッファリング保護
*   **要件**: CPU 負荷を抑えるため、コンソールへの直接表示をやめ、バッファのみに記録せよ。
*   **設定**: `no logging console`, `logging buffered 16384`

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `management-interface Gi0/1 allow ssh` が設定されている場合、Gi0/2 に接続された管理 PC から SSH は可能か？
    *   **回答**: 不可能。MPP によって Gi0/1 以外からの SSH 着信はドロップされる。
2.  **トラブルシュート**: メモリしきい値設定後、`show memory free` でしきい値を下回っているがログが出ていない。なぜか？
    *   **回答**: ログレベルの設定が低いか、`warning` しきい値に達していない可能性がある。
3.  **Design**: CPU への DoS 攻撃を防ぎつつ、管理アクセスを物理的に特定のポートに限定するための最適な技術は？
    *   **回答**: **MPP (Management Plane Protection)**。
4.  **実装**: 境界ルータで SSH バージョン 1 を無効にするコマンドは？
    *   **回答**: `ip ssh version 2` (これにより v1 が拒否される)。
5.  **コンフィグ読解**: `ipv6 spd mode aggressive` の目的は？
    *   **回答**: CPU やキューが過負荷の際に、異常な IPv6 パケットを積極的に破棄して、正当なトラフィックを保護するため。

---

## 🔗 参考リソース

*   **Cisco IOS-XE 構成ガイド**
    *   [Configuring Management Plane Protection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_mpp/configuration/xe-16/sec-data-mpp-xe-16-book.html)
    *   [Configuring Control Plane Protection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_policing/configuration/xe-16/qos-policing-xe-16-book/qos-policing-control-plane.html)
*   **テクニカルノート**
    *   [Cisco Guide to Harden Cisco IOS Devices](https://www.cisco.com/c/en/us/support/docs/ip/access-lists/13608-21.html)
*   **Learning Matrix**
    *   [CCIE Security v6.1 Learning Matrix](https://learningnetwork.cisco.com/s/article/ccie-security-v6-1-learning-matrix)

---

## 📝 **補足（Notes）**

*   **学習メモ**: マネジメントプレーン保護は「管理者のためのバリア」です。これを設定することで、ネットワークが火を吹いている（攻撃されている）時でも、管理者がログインして消火活動を行うことができます。
*   **注意点**: ラボ試験では `control-plane` 以下の設定階層（`management`, `host`, `transit`）を混同しないようにしてください。物理インターフェイスの制限は必ず `management` サブコマンド内で行います。
