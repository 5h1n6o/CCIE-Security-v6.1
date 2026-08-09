---
layout: default
title: 3.2.b-Memory-thresholding
nav_order: 2
parent: 3.2-Management-plane
grand_parent: 3.0-Security-Infrastructure
---

# 3.2.b Memory thresholding

**Memory thresholding**（メモリしきい値管理）は、Cisco デバイスのマネジメントプレーン保護（Management Plane Protection）における重要な技術であり、デバイスの RAM リソースを監視して、空き容量が不足した際に管理者に通知を行ったり、特定のプロセスによるメモリの過剰消費を制限したりする機能です。CCIE Security v6.1 においては、デバイス自体の可用性を維持し、DoS 攻撃やソフトウェアの不具合によるシステムクラッシュを未然に防ぐための「インフラストラクチャ保護」の一環として定義されています,。

---

## 📘 概要

*   **機能概要**: システムの空きメモリを監視し、「警告（Warning）」および「深刻（Critical）」の 2 段階のしきい値を設定します。しきい値に達すると、Syslog メッセージの生成、SNMP トラップの送信、またはメモリを大量消費する新しいプロセス（管理セッションなど）の起動を拒否するアクションを実行します。
*   **利用目的**: デバイスの安定稼働の確保、メモリリークの早期発見、およびリソース枯渇を狙った攻撃からの保護。
*   **どのような場面で利用するか**: 多数の VPN セッション（AnyConnect 等）を収容する ASA や、大規模なルーティングテーブル（BGP フルルート等）を保持するルータにおいて、予期せぬリソース不足による管理不能状態を避けるために設定します。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **監視対象** | Processor メモリ（ルータ）または システム RAM（ASA/FTD）。 |
| **主なアクション** | Syslog 通知、SNMP 通知、プロセスの新規割り当て拒否。 |
| **しきい値の種類** | `warning`（警告）および `critical`（緊急）。 |
| **メリット** | システムのハングアップ防止、管理アクセスの可用性確保。 |
| **デメリット** | 設定が不適切だと、正常な運用プロセスまで停止する可能性がある。 |
| **対応機種** | IOS, IOS-XE, ASA, FTD。 |
| **設計上の注意点** | 物理メモリ容量に応じた、現実的な KB/MB 単位でのしきい値設計が必要。 |

---

## 🏗 動作原理

メモリしきい値機能は、IOS-XE のリソースマネージャが常にメモリプールをスキャンすることで動作します。

```text
[ Physical RAM ]
       ↓
[ Memory Pool (Processor/IO) ]
       ↓
[ Resource Manager ] <--- (監視: memory free low-threshold)
       ↓
       ├─ [ 通常時 ] : 処理継続
       │
       ├─ [ Warning 到達 ] : Syslog 出力 / 管理者に警告
       │
       └─ [ Critical 到達 ] : Syslog 出力 / SNMP 送信 / 
                              新しい管理プロセスの制限 (SSH/HTTP等の停止)
```

---

## ⚙ 動作シーケンス

1.  **定期的な監視**: ルータのカーネルが数秒おきに空きメモリのバイト数を確認します。
2.  **しきい値判定**: 設定された `low-threshold` を下回ったかどうかを判定します。
3.  **イベントトリガー**:
    *   **警告レベル**: 指定された値（KB）を下回ると、`%SYS-4-CHUNK_OUT_OF_MEMORY` などのログを出力します。
    *   **緊急レベル**: さらに低い値に達すると、低優先度のプロセス（デバッグや管理用ログイン）へのメモリ割り当てを停止します。
4.  **リカバリ**: プロセスが終了したり、設定が変更されたりしてメモリが解放され、しきい値を上回ると「復旧」のログが出力されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Blueprint の重要ポイント**: メモリしきい値は「3.0 Security Infrastructure」の中で、デバイスを攻撃から守る「Hardening（要塞化）」技術として重要視されています,。
*   **ラボ試験で設定させられそうな内容**:
    *   「空きメモリが 50MB 以下になったら警告を出し、20MB 以下になったら緊急として扱え」といった数値指定の設定。
    *   「メモリ不足時に管理通知を SNMP 経由で飛ばせ」という複数機能の組み合わせ要件。
*   **よくある設定ミス**:
    *   単位の勘違い（KB なのか MB なのか）。IOS の `memory free low-threshold` は **KB 単位** であることに注意してください。
    *   現在の空きメモリよりも高い値をしきい値に設定してしまい、設定直後にデバイスが不安定になる、あるいは SSH が切断されるミス。
*   **トラブルシュート問題**: 「ルータにログインできない」という症状の原因が、実はメモリしきい値に達したことによる管理プロセスの制限である、というパターンが考えられます。

---

## 🛠 設定方法

### 1. IOS-XE での基本設定
空きメモリが 20,000KB (約 20MB) で警告、10,000KB で緊急通知を出す設定。
```bash
! グローバルコンフィギュレーションモード
memory free low-threshold warning 20000
memory free low-threshold critical 10000
```

### 2. 特定のプロセスによるメモリ消費制限 (Optional)
特定のプロセスが全メモリの 80% を消費した際に通知する設定。
```bash
process memory threshold central 80
```

### 3. ASA でのメモリ監視 (Context モード等の場合)
ASA では各コンテキストごとにメモリを制限する場合が多いですが、グローバルでも設定可能です。
```bash
# ASA の場合
resource-quota memory 512
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **空きメモリと設定状態の確認** | <code>show memory free</code> |
| **メモリ使用統計の全体表示** | <code>show memory statistics</code> |
| **しきい値に関連するログの確認** | <code>show logging \| include MEMORY</code> |
| **プロセスごとのメモリ消費確認** | <code>show processes memory sorted</code> |
| **リソース枯渇によるドロップ確認** | <code>show resource errors</code> (ASA用) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| SSH 接続が拒否される | <code>Critical</code> しきい値に到達 | <code>show memory free</code> | 不要なプロセス（debug等）の停止。 |
| 設定保存ができない | メモリ不足で <code>write mem</code> 失敗 | <code>show logging</code> | <code>memory low-threshold</code> を一時的に下げる。 |
| メモリログが大量に出る | メモリリークまたはしきい値が高すぎ | <code>show proc mem sorted</code> | 原因プロセスを特定・再起動。 |
| SNMP 通知が届かない | SNMP トラップ設定の欠落 | <code>show run snmp-server</code> | <code>snmp-server enable traps resource-policy</code> を追加。 |

---

## ⚠ 制限事項

*   **プラットフォーム依存**: IOS と IOS-XE では設定可能な最小値やコマンド体系が異なる場合があります。
*   **自動リカバリの欠如**: しきい値機能は通知と一部制限を行いますが、メモリを消費しているプロセスを「自動で Kill」することはありません（管理者が手動で対処する必要があります）。
*   **I/O メモリの考慮**: `Processor` メモリだけでなく、パケット処理に使用する `I/O` メモリの枯渇にも注意が必要です。

---

## 🔄 他技術との関連

*   **3.2.c Securing device access**: メモリが枯渇すると SSH アクセス自体が制限されるため、デバイスへのアクセス制御と密接に関係します。
*   **3.6.b SNMP / 3.6.c SYSLOG**: メモリしきい値の通知先としてこれらのプロトコルが使用されます。
*   **AnyConnect VPN**: 数千ユーザーを収容する場合、メモリ使用率が急増するため、しきい値管理が可用性確保の鍵となります。
*   **Routing Plane Security**: BGP 等のプロセスが大量のメモリを消費するため、ルーティングの安定性と直結します。

---

## 🧩 比較表

### Memory Thresholding vs CPU Policing (CoPP)

| 特徴 | Memory Thresholding | CoPP (3.1.a) |
| :--- | :--- | :--- |
| **保護リソース** | RAM (空き容量) | CPU (パケット処理能力) |
| **制御対象** | 内部プロセス・メモリ確保 | 受信パケットの流量 |
| **アクション** | 通知、新規メモリ割り当て拒否 | パケットのドロップ、レート制限 |
| **主な用途** | システムの安定化、リーク防止 | DoS 攻撃、過剰トラフィック防御 |

---

## 💡 ベストプラクティス

1.  **段階的なしきい値設定**: `warning` は総メモリの 10% 程度、`critical` は 5% 程度に設定することを推奨します。
2.  **SNMP との連携**: ログはルータのバッファで消えてしまう可能性があるため、必ず NMS (外部監視サーバー) に SNMP トラップを飛ばすようにします。
3.  **ベースラインの把握**: 定常運用時の `show memory free` の値を記録しておき、異常値の判断基準を明確にします。
4.  **Logging Buffer の調整**: メモリ不足時にログが生成される際、ロギングバッファ自体がメモリを消費するため、バッファサイズは適切に管理します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な警告しきい値の設定
*   **問題**: R1 で空きメモリが 64MB を下回った際に Syslog 警告を出すようにせよ。
*   **要件**: 単位は KB を使用すること。
*   **設定**: `memory free low-threshold warning 65536`
*   **検証**: `show memory free` で設定反映を確認。

### 2. 緊急しきい値による管理アクセス保護
*   **要件**: 空きメモリが 32MB 以下になったら新規プロセスの起動を制限せよ。
*   **設定**: `memory free low-threshold critical 32768`

### 3. プロセスレベルの監視と SNMP トラップ
*   **要件**: メモリイベントを NMS (10.1.1.100) へ送信せよ。
*   **設定**: 
    ```bash
    snmp-server enable traps resource-policy
    snmp-server host 10.1.1.100 traps public
    ```

### 4. 予約メモリの確保 (Critical Reserved)
*   **要件**: 緊急事態用に最低限のメモリをシステム用に予約せよ。
*   **設定**: `memory reserved critical 2048` (KB単位)

### 5. メモリ枯渇のシミュレーション (検証用)
*   **課題**: デバッグを大量に有効化してメモリ消費を増やし、しきい値到達時のログを確認せよ。
*   **コマンド**: `debug all` (※ラボ環境以外では絶対禁止) -> `show logging`

### 6. I/O メモリの監視
*   **要件**: Processor メモリだけでなく I/O メモリもしきい値を設定せよ。
*   **設定**: プラットフォームにより自動で Processor が対象になるが、一部モデルでは別途設定が必要な場合がある。

### 7. ASA でのリソース制限 (コンテキスト別)
*   **要件**: コンテキスト "VPN-SITE-A" に 256MB のメモリ制限をかけよ。

### 8. FTD (FDM) でのメモリ監視設定
*   **課題**: FMC/FDM の Web UI から「Health Monitor」でメモリしきい値アラートを設定せよ。

### 9. メモリ統計の定期リセットと監視
*   **要件**: 24時間ごとのメモリ最大消費プロセスを確認せよ。

### 10. EEM (Embedded Event Manager) との連携
*   **要件**: メモリしきい値ログが出た際に、自動で不要なサービスを `shutdown` するスクリプトを組め。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `memory free low-threshold warning 50000` と設定されているルータで、`show memory free` の出力が `Free: 45000KB` であった。どのような動作が期待されるか？
    *   **回答**: システムが警告しきい値を下回ったことを検知し、Syslog に警告メッセージを出力する。
2.  **トラブルシュート**: ルータで特定の管理コマンドが実行できず "Out of memory" エラーが出る。最初に確認すべき設定は？
    *   **回答**: `memory free low-threshold critical` の設定値と、現在の空きメモリ量を比較する。
3.  **Design**: メモリしきい値管理がマネジメントプレーン保護に含まれる理由は？
    *   **回答**: メモリが完全に枯渇すると、管理者がデバイスにログインしてトラブルシューティングを行うためのプロセス（SSH等）自体が起動できなくなるため。
4.  **実装**: メモリ不足時の通知をリアルタイムで行いたい。Syslog 以外に推奨される手法は？
    *   **回答**: SNMP トラップ（`resource-policy` トラップの有効化）。
5.  **コンフィグ読解**: `memory reserved critical 1024` コマンドの目的は？
    *   **回答**: システムが極端なメモリ不足に陥った際でも、重要な OS 機能を維持するために、指定した量（1024KB）のメモリを常に確保しておくため。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [Configuring Control Plane Protection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_policing/configuration/xe-16/qos-policing-xe-16-book/qos-policing-control-plane.html)
*   **Cisco Technical Notes**
    *   [Memory Management and Troubleshooting on Cisco Routers](https://www.cisco.com/c/en/us/support/docs/routers/asr-1000-series-aggregation-services-routers/116963-troubleshoot-ios-xe-memory-00.html)
*   **Integrated Security Technologies and Solutions, Volume II**

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「メモリしきい値はルータの燃料切れ警告灯」です。警告灯がついたら（Warning）、管理者はすぐに原因を特定し、燃料（メモリ）が尽きる（Critical/Crash）前に対処する必要があります。
*   **図解**: `show memory free` のカウンタが、自分が設定したしきい値を通過する瞬間にログが出ることをイメージして、KB 単位の計算（例：1MB = 1024KB）を正確に行えるようにしましょう。
*   **注意点**: ラボ試験では、単位の間違い（例：50,000KB と入力すべきところを 50 と入力してしまう）だけでしきい値が機能しなくなり、失点につながります。設定後の確認は必須です。
