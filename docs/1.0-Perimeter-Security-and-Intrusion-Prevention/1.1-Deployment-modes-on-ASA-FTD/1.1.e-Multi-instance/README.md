---
layout: default
title: 1.1.e-Multi-instance
nav_order: 5
parent: 1.1-Deployment-modes-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.1.e Multi-instance

Cisco Firepower Threat Defense (FTD) における **マルチインスタンス（Multi-instance）** は、Firepower 4100 および 9300 シリーズのハードウェア上で、単一のシャーシ内に完全に独立した複数の FTD ソフトウェアインスタンスを稼働させる技術です。これは、ASA のマルチコンテキストモードに代わる次世代の仮想化機能であり、リソースの完全な隔離と管理の柔軟性を提供します。

---

## 📘 概要

*   **機能概要**: 物理ハードウェアリソース（CPU、メモリ、インターフェイス）を論理的に分割し、それぞれに独自の FTD イメージ（Docker コンテナベース）を割り当てます。
*   **利用目的**: リソースの競合を避ける必要がある複数の部門や顧客（テナント）間でのハードウェア共有、または開発環境と本番環境の隔離に使用されます。
*   **主な役割**: 
    *   **リソースの隔離**: インスタンスごとに CPU コアや RAM が専有されるため、1つのインスタンスの過負荷が他へ波及しません。
    *   **バージョンの独立性**: インスタンスごとに異なるソフトウェアバージョンを実行可能です。
    *   **耐障害性**: 1つのインスタンスで Snort がクラッシュしても、他のインスタンスの通信は継続されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | Docker コンテナ技術を利用した強力なリソース隔離。 |
| **用途** | マルチテナント、MSP（管理サービスプロバイダ）、高度なセキュリティ分離。 |
| **メリット** | インスタンスごとの独立したアップグレード、リソースの専有（No noisy neighbor）。 |
| **デメリット** | 対応ハードウェア（4100/9300）が高価。FXOS と FMC の両方の管理が必要。 |
| **対応機種** | Firepower 4100 シリーズ、9300 シリーズ。 |
| **制限事項** | ローエンド機（FTD 1000/2100等）や仮想版（FTDv）では非サポート。 |
| **設計上の注意点** | シャーシマネージャ (FXOS) でのリソース割り当て設計が極めて重要。 |

---

## 🏗 動作原理

マルチインスタンスは、Cisco の **FXOS (Firepower eXtensible Operating System)** アーキテクチャに基づいています。

```text
[ Physical Chassis (Firepower 4100/9300) ]
   ↓
[ Supervisor / FXOS (Chassis Manager) ]
   ↓ 管理: リソース割り当て (CPU/RAM/IF)
   +--- [ FTD Instance 1 (Container) ] → Managed by FMC
   |      (Dedicated CPU Cores / Dedicated RAM)
   +--- [ FTD Instance 2 (Container) ] → Managed by FMC
   |      (Dedicated CPU Cores / Dedicated RAM)
   +--- [ FTD Instance N (Container) ]
```

各インスタンスは、FMC からは見かけ上「独立した物理デバイス」として登録され、管理されます。

---

## ⚙ 動作シーケンス

1.  **リソース定義**: 管理者が FXOS (Chassis Manager) にログインし、インスタンスに使用する CPU コア数、メモリ、インターフェイスを定義します。
2.  **インスタンス生成**: FXOS が Docker コンテナとして FTD イメージをブートします。
3.  **初期セットアップ**: インスタンスの CLI にアクセスし、管理 IP アドレスと FMC 登録用のキーを設定します。
4.  **FMC 登録**: FMC 上で新しいデバイスとして追加します。
5.  **ポリシー適用**: FMC が個別のアクセスコントロールポリシーやルーティング設定を各インスタンスへデプロイします。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **FXOS との連携**: ラボ試験において FTD インスタンスを「作成する」段階から問われる可能性があります。FXOS 側でのインターフェイス割り当て（Data ↔ Management）の理解が必須です。
*   **リソース計算**: 特定のインスタンスに 4コア割り当て、別のインスタンスに 2コア割り当てるなど、物理的な制約内での配分を指示されるケースがあります。

### ラボ試験で設定させられそうな内容
*   **シャーシ内でのインスタンス作成**: FXOS GUI/CLI を使用したインスタンスのデプロイ。
*   **Shared インターフェイスの利用**: 同一の物理インターフェイスをサブインターフェイス化し、異なるインスタンスに割り当てる構成。
*   **HA 設定**: 同一シャーシ内のインスタンス間、または異なるシャーシ間のインスタンス間での Failover 構成。

### 試験で狙われやすい制限事項
*   **ASA 互換性**: マルチインスタンスは FTD 固有の機能であり、ASA モードでは「マルチコンテキスト」を使用することを混同しないでください。
*   **リソースの上限**: 物理コア数を超える割り当てはできません。

### showコマンドから状態を判断
*   `show app-list` (FXOS): インスタンスが `online` かどうかを確認。
*   `show module` (FXOS): サービスモジュールのステータス確認。

---

## 🛠 設定方法

### 1. FXOS でのインターフェイス割り当て (Chassis Manager)
物理インターフェイスを FTD インスタンスに渡す前に、FXOS で「Data」属性として設定する必要があります。

### 2. インスタンスの作成 (FXOS CLI 例)
```bash
# FXOS CLI へのログイン後
scope ssa
scope app-type ftd
create app-instance ftd_inst1
  set slot 1
  set cores 4
  set mgmt-ip 192.168.10.101
  set mgmt-mask 255.255.255.0
  set mgmt-gw 192.168.10.1
  add data-interface Ethernet1/1
  add data-interface Ethernet1/2
commit-buffer
```

### 3. FTD インスタンス内での設定
起動後、インスタンスのコンソールで FMC への登録を行います。
```bash
> configure manager add <FMC_IP> <Registration_Key>
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **FXOS内インスタンス状態確認** | `show app-list` |
| **リソース割り当ての確認** | `show app-instance detail` |
| **FTDインスタンスへの接続** | `connect module 1 console` → `connect ftd <instance_name>` |
| **インスタンスのリソース使用量** | `show cpu` (FTD CLI内) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| インスタンスが `starting` で止まる | 割り当てコア数に対して空きメモリが不足 | `show module 1 details` | 他のインスタンスを削除するか、コア数を減らす。 |
| FMCから疎通が取れない | FXOSで管理インターフェイスの設定ミス | `show interface mgmt` | 物理的な管理ポートとインスタンスのIP紐付けを確認。 |
| インターフェイスが表示されない | FXOSでインスタンスに割り当てられていない | `show app-instance detail` | FXOSで `add data-interface` を再実行。 |
| インスタンスの削除ができない | HA構成が組まれている | `show failover` | FMC側でHAを解除してから削除を試みる。 |

---

## ⚠ 制限事項

*   **ハードウェア限定**: Firepower 4100/9300 以外のプラットフォーム（2100, 3100, 1000, 1100, ASAv/FTDv）では利用不可。
*   **ライセンス**: インスタンスごとにベースライセンスが必要になる場合があります。
*   **機能制限**: クラスタリング（Clustering）の設定において、マルチインスタンス特有のトポロジ制約が存在します。

---

## 🔄 他技術との関連

*   **Failover (HA)**: インスタンスレベルでの Active/Standby が可能。異なるシャーシ間のインスタンス同士でペアを組みます。
*   **Routing**: インスタンスごとに独立したルーティングプロセス（OSPF, BGP等）が動作します。
*   **Segmentation**: インスタンス間はハードウェアレベルで分離されているため、VRF よりも強力なセグメンテーションを提供します。

---

## 🧩 比較表

### Multi-instance (FTD) vs Multi-context (ASA)

| 機能 | Multi-instance (FTD) | Multi-context (ASA) |
| :--- | :--- | :--- |
| **分離技術** | Docker コンテナ | ソフトウェア・コンテキスト |
| **OS/Kernel** | インスタンスごとに独立 | 共通カーネルを共有 |
| **リソース隔離** | 物理コア/RAMを専有（強力） | 共有（Noisy neighborのリスクあり） |
| **バージョン** | インスタンスごとに異なるバージョン可 | 全コンテキストで共通バージョン |
| **VPNサポート** | **フルサポート (RA-VPN含む)** | RA-VPN (AnyConnect) は非サポート |

---

## 💡 ベストプラクティス

*   **コア配分の最適化**: 1つのインスタンスに過剰なコアを割り当てず、Snort のスレッド数と物理コアのバランスを考慮します。
*   **Out-of-band 管理**: FXOS の管理と FTD インスタンスの管理 IP は別々のサブネットに配置することを推奨します。
*   **バックアップ**: インスタンスの設定だけでなく、FXOS（シャーシ）のコンフィグレーションも定期的にエクスポートします。

---

## 📝 ラボ学習・設定サンプル例

### 1. FXOSでのデータインターフェイスの有効化
*   **要件**: 物理ポート Ethernet 1/1 を FTD インスタンスで使用可能にせよ。
*   **設定**: FXOS GUI > Interfaces > Ethernet 1/1 を「Data」に設定。

### 2. 最小リソースでのインスタンス作成
*   **要件**: 1コア、最小メモリで `dev_test` という名前のインスタンスを作成せよ。
*   **設定**: `ssa > app-instance dev_test > set cores 1` (※モデルにより最小コア数は異なります)。

### 3. 異なるバージョンの共存
*   **要件**: インスタンスAを FTD 7.0、インスタンスBを FTD 7.1 で稼働させよ。
*   **手順**: FXOS に両方のイメージをアップロードし、インスタンス作成時に `set version` で指定。

### 4. Shared インターフェイスの設定 (Sub-interface)
*   **要件**: Eth 1/1.10 を Inst-A、Eth 1/1.20 を Inst-B に割り当てよ。
*   **手順**: FXOS で Eth 1/1 を Logical インターフェイスとして構成し、VLAN ごとにインスタンスへ追加。

### 5. マルチインスタンス間 HA の構築
*   **要件**: Chassis-1 の Inst-1 と Chassis-2 の Inst-1 で HA を組め。
*   **手順**: 通常の FTD HA 構成手順を FMC から各インスタンス（デバイス）に対して実行。

### 6. リソース制限の変更
*   **要件**: 稼働中のインスタンスのコア数を 4 から 8 へ増やせ。
*   **注意**: 変更にはインスタンスの再起動が伴います。

### 7. FXOS CLI からのトラブルシュート
*   **要件**: 特定インスタンスの Snort プロセス状態を確認せよ。
*   **コマンド**: `connect ftd <name>` → `system support engine-status`

### 8. インスタンスの完全削除
*   **要件**: 不要になった `old_inst` を削除し、リソースを解放せよ。
*   **コマンド**: `scope ssa > delete app-instance old_inst`

### 9. 管理 IP の変更
*   **要件**: FXOS からインスタンスの管理 IP を 10.1.1.50 に変更せよ。
*   **コマンド**: `app-instance <name> > set mgmt-ip 10.1.1.50`

### 10. FMC への一括登録
*   **要件**: 3つのインスタンスを FMC の同一グループに登録せよ。

---

## ❓ 想定試験問題

1.  **問題**: Firepower 9300 において、1つのセキュリティモジュール内で FTD インスタンスと ASA インスタンスを同時に実行できるか？
    *   **解答**: いいえ。現時点のアーキテクチャでは、1つのスロット（モジュール）につき1つのアプリケーションタイプ（FTD または ASA）のみがサポートされます。
2.  **問題**: マルチインスタンス構成において、特定のインスタンスの CPU 使用率が 100% に達した場合、他のインスタンスのパケット処理性能にどのような影響があるか？
    *   **解答**: 影響はありません。物理コアが各インスタンスに専有的に割り当てられているため、リソースが完全に隔離されています。
3.  **問題**: FTD マルチインスタンス環境で AnyConnect VPN を提供することは可能か？
    *   **解答**: はい。ASA マルチコンテキストとは異なり、FTD マルチインスタンスはリモートアクセス VPN をフルサポートします。
4.  **問題**: インスタンスを削除すると、そのインスタンスに割り当てられていたデータインターフェイスの FXOS 側の設定も削除されるか？
    *   **解答**: いいえ。インターフェイスの設定は FXOS 側に残り、別のインスタンスへ再割り当て可能な状態になります。
5.  **問題**: FXOS Chassis Manager を使用せずに FTD マルチインスタンスを作成できるか？
    *   **解答**: いいえ。マルチインスタンスの管理、リソース配分、ブート処理は FXOS の役割であり、必須のコンポーネントです。

---

## 🔗 参考リソース

*   **Configuration Guide**:
    *   [Cisco Firepower 4100/9300 FXOS Chassis Manager Configuration Guide - Multi-Instance](https://www.cisco.com/c/en/us/td/docs/security/firepower/fxos/fxos271/web_config/b_GUI_Config_Guide_FXOS_271/multi_instance.html)
    *   [Cisco Secure Firewall Management Center Administration Guide - Device Management Multi-Instance](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/device_management_multi_instance.html)
*   **Cisco Live (Video/Slides)**:
    *   [BRKSEC-2021: Firepower Threat Defense - Packet Flow and Troubleshooting](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-2021)
    *   BRKSEC-3452: Deep Dive into Firepower 4100/9300 Architecture
*   **Technical Notes**:
    *   [FTD Multi-Instance Capabilities and Limitations](https://www.cisco.com/c/en/us/support/docs/security/firepower-ngfw/215332-ftd-multi-instance-capabilities-and-limi.html)
*   **White Paper**:
    *   Cisco Next-Generation Security Solutions: Multi-instance vs. Multi-context
*   **Equipment List**:
    *   [CCIE Security v6.1 Equipment and Software List](https://learningnetwork.cisco.com/s/article/ccie-security-v6-1-equipment-and-software-list)

---

## 📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  
