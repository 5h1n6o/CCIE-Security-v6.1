---
layout: default
title: 3.5.d-TKIP
nav_order: 4
parent: 3.5-Wireless-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.5.d TKIP (Temporal Key Integrity Protocol)

**TKIP (Temporal Key Integrity Protocol)** は、脆弱性が指摘された WEP (Wired Equivalent Privacy) の欠点を補うために、既存のハードウェアで動作するように設計された暫定的な暗号化プロトコルです。CCIE Security v6.1 のブループリントでは、ワイヤレスセキュリティの歴史的背景と、現代のネットワークにおける制限事項を理解するために不可欠な項目として定義されています。

---

## 📘 概要

*   **機能概要**: RC4 暗号アルゴリズムをベースに、パケットごとのキーミキシング、整合性チェック（MIC）、および拡張初期化ベクトル（IV）を導入することでセキュリティを強化したプロトコルです。
*   **利用目的**: ハードウェアのアップグレードが困難なレガシーデバイスに対し、WEP 以上のセキュリティを提供することを目的としています。
*   **どのような場面で利用するか**: 現代の設計では**非推奨**ですが、AES をサポートしていない非常に古いレガシー端末を接続せざるを得ない環境でのみ使用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **暗号アルゴリズム** | RC4 (WEP と同一だが、鍵管理が動的) |
| **整合性チェック** | **Michael (MIC)** アルゴリズム |
| **鍵管理方式** | **4-Way Handshake** による動的な生成 |
| **主なメリット** | WEP 対応の旧型ハードウェアでファームウェア更新のみで動作可能 |
| **主なデメリット** | セキュリティの脆弱性、スループットが最大 **54Mbps** に制限される |
| **現在のステータス** | 802.11n 以降の規格ではサポート外（または非推奨） |

---

## 🏗 動作原理

TKIP は、WEP の致命的な弱点であった「固定鍵」と「短い IV」を改善するために、以下の要素を組み合わせています。

```text
[ Data Payload ] 
   ↓
[ Michael MIC (Integrity Check) ] 
   ↓
[ Per-Packet Key Mixing ] ← [ Transmit Address ] + [ TKIP Sequence Counter ]
   ↓
[ RC4 Encryption ]
   ↓
[ Encrypted Packet ]
```

1.  **Per-Packet Key Mixing**: 送信元 MAC アドレスと 48 ビットのシークエンスカウンタを組み合わせ、パケットごとに異なる暗号鍵を生成します。
2.  **Michael MIC**: 改ざん防止のために 8 バイトの整合性チェックコードを付加します。
3.  **TKIP Sequence Counter (TSC)**: リプレイ攻撃を防ぐために、パケットごとにインクリメントされるカウンタを使用します。

---

## ⚙ 動作シーケンス

1.  **4-Way Handshake**: クライアントと AP 間で、暗号化に使用する一時鍵（PTK: Pairwise Transient Key）を生成します。
2.  **鍵のミキシング**: 生成された一時鍵と送信元アドレス、TSC を混合し、パケット単位の RC4 鍵を作成します。
3.  **MIC 付与**: データの整合性を確認するための Michael 値を計算し、ペイロードに付加します。
4.  **暗号化**: RC4 によりデータと MIC を暗号化して送信します。
5.  **対抗措置（Countermeasures）**: MIC エラー（改ざんの疑い）を 60 秒以内に 2 回検知した場合、AP は全クライアントをキックし、1 分間通信を停止して攻撃を防ぎます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **スループット制限の理解**: 「802.11n を有効にしているのに 54Mbps しか出ない」というトラブルシュート問題が出た場合、暗号化方式に **TKIP** が選択されていないか確認する必要があります。
*   **WPA (WPA1) との関連**: 多くの実装で、WPA-Personal/Enterprise のデフォルト暗号化が TKIP になっている点に注意してください。
*   **Mixed Mode の設定**: TKIP と AES を同時に許可する設定（WPA-TKIP + WPA2-AES）の構成手順を把握しておく必要があります。
*   **show コマンドの確認**: `show client detail` の出力結果から、`Policy Type` が WPA2 であっても `Encryption` が **TKIP** になっていないかを確認できる能力が求められます。
*   **セキュリティ要件**: 「最新のセキュリティ基準に適合せよ」という要件に対し、TKIP を無効化して **AES (CCMP)** に統一する操作が期待されます。

---

## 🛠 設定方法

### Cisco WLC (AireOS) での TKIP 有効化例
※試験ではレガシーサポートの文脈で設定を求められることがあります。

```bash
! WLANの作成
config wlan create 10 LEGACY-SSID

! WPA1(WPA) ポリシーの有効化とTKIPの指定
config wlan security wpa enable 10
config wlan security wpa encryption tkip enable 10
config wlan security wpa encryption aes disable 10

! 認証方式の指定 (PSKの場合)
config wlan security wpa akm psk enable 10
config wlan security wpa psk set 10 ascii cisco123

! WLANの有効化
config wlan enable 10
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **クライアントの詳細ステータス** | <code>show client detail [MAC_ADDRESS]</code> |
| **WLANのセキュリティ構成確認** | <code>show wlan [WLAN_ID]</code> |
| **暗号化方式の統計確認** | <code>show ap stats dots11 [24ghz\|5ghz]</code> |
| **リアルタイムの認証デバッグ** | <code>debug client [MAC_ADDRESS]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 通信速度が 54Mbps で頭打ちになる | 暗号化に TKIP が使用されている | <code>show client detail</code> | AES (CCMP) への変更を検討。 |
| 無線接続が頻繁に切断（1分間停止）する | TKIP MIC エラー（攻撃または干渉） | <code>show logging</code> | 干渉源の特定、または AES への移行。 |
| 802.11n/ac クライアントが接続できない | WLC側で TKIP のみが許可されている | <code>show wlan</code> | <code>AES enable</code> を追加設定。 |
| クライアントが SSID を見つけられない | 端末側が TKIP (WPA1) を非サポート | N/A | WPA2-AES (WPA3-SAE) を使用。 |

---

## ⚠ 制限事項

*   **802.11n/ac/ax 不適合**: これらの高速規格では、AES-CCMP が必須要件です。TKIP を使用すると、規格上の最高速度は利用できず、レガシーモードにフォールバックされます。
*   **脆弱性**: 現在では計算機能力の向上により、Michael MIC やキーストリームの解読が理論的に可能となっており、セキュリティ強度は不十分です。

---

## 🔄 他技術との関連

*   **3.5.e AES**: TKIP の後継となる強力な暗号化方式。現在の推奨。
*   **3.5.b WPA2**: 一般的に AES を使用しますが、互換性のために TKIP もサポート可能です。
*   **4-Way Handshake**: TKIP も AES も、このプロセスで PMK から PTK を導出します。

---

## 🧩 比較表

| 特徴 | TKIP | AES (CCMP) |
| :--- | :--- | :--- |
| **ベースアルゴリズム** | RC4 (ストリーム暗号) | AES (ブロック暗号) |
| **鍵長** | 128ビット (ミキシング鍵) | 128ビット以上 |
| **整合性チェック** | Michael (64ビット) | CBC-MAC (128ビット) |
| **CPU負荷** | 低い（レガシー向け） | 高い（ハードウェア処理推奨） |
| **最高速度** | 54 Mbps | ワイヤレート (Gbpsクラス) |

---

## 💡 ベストプラクティス

1.  **新規構築時は TKIP を無効化**: セキュリティとパフォーマンスの両面から、AES のみを使用するように構成します。
2.  **移行パスの検討**: レガシー端末を排除できない場合のみ、専用の低いセキュリティ用の SSID を分離して運用します。
3.  **WPA3 への移行**: ブループリントに含まれる WPA3 では TKIP は完全に廃止されています。

---

## 📝 ラボ学習・設定サンプル例

### 1. レガシー対応 SSID の構築
*   **要件**: WPA-TKIP のみを使用する VLAN 20 向けの SSID を作成せよ。
*   **設定**: `config wlan security wpa encryption tkip enable 10`。

### 2. TKIP による速度制限の検証
*   **課題**: TKIP 有効時、802.11n 対応端末の接続レートが 54Mbps 以下であることを確認せよ。
*   **検証**: `show client detail [MAC]` で `Current Rate` を確認。

### 3. WPA2 Mixed Mode の構成
*   **要件**: TKIP と AES の両方のクライアントを受け入れるように設定せよ。
*   **設定**: `config wlan security wpa encryption tkip enable`, `config wlan security wpa encryption aes enable`。

### 4. Michael MIC 対抗措置の確認
*   **要件**: ログから MIC failure による通信断の記録を特定せよ。

### 5. GUI による設定 (WLC)
*   **操作**: Wireless > WLANs > Security > Layer 2 タブで WPA/WPA2 を選択し、TKIP をチェック。

### 6. WPA-Enterprise + TKIP
*   **要件**: 802.1X 認証を使用し、暗号化を TKIP に設定せよ。

### 7. 暗号化方式の監査
*   **要件**: `show wlan sum` を使用し、稼働中の全 WLAN の暗号化方式をリストアップせよ。

### 8. Catalyst 9800 での TKIP 設定
*   **要件**: IOS-XE ベースの WLC で TKIP を含む Policy Profile を定義せよ。

### 9. デバッグによる PTK 生成の確認
*   **操作**: `debug client` を有効にし、TKIP 鍵が割り当てられるプロセスを追跡せよ。

### 10. SSID 移行ラボ
*   **要件**: 既存の TKIP SSID を AES-only に変更し、非対応クライアントが接続不能になることを確認せよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: クライアントが 802.11n 対応であるにも関わらず、スループットが非常に低い。WLC の設定を確認したところ、`WPA Encryption: TKIP` が有効で、`AES` が無効であった。解決策は？
    *   **回答**: `AES (CCMP)` 暗号化を有効にし、クライアントが AES で接続するように修正する。
2.  **Design**: レガシー端末と最新端末が混在する環境で、セキュリティを最大化しつつ互換性を保つための暗号化ポリシーは？
    *   **回答**: WPA2-AES を主とし、必要最低限の範囲で TKIP との Mixed Mode を使用する。
3.  **コンフィグ読解**: `show client detail` で `Encryption: TKIP` と表示されている。このクライアントがリプレイ攻撃を受けるリスクは WEP と比較してどうか？
    *   **回答**: 低い。TKIP は TSC (Sequence Counter) を使用しており、WEP よりもリプレイ攻撃に対する耐性が強化されている。
4.  **実装**: WLC で TKIP を有効にした WLAN において、特定のクライアントが接続した瞬間に他のクライアントが一時的に切断された。考えられる原因は？
    *   **回答**: TKIP の Michael MIC エラーが 60 秒以内に複数回発生し、Countermeasures（対抗措置）が発動した。
5.  **Design**: WPA3 を導入する計画があるが、既存の TKIP デバイスは継続利用可能か？
    *   **回答**: 不可能。WPA3 規格では TKIP および WEP はサポートされておらず、AES-CCMP/GCMP が必須となる。

---

## 🔗 参考リソース

*   **Cisco Live (BRKSEC-2003)**: [Wireless Security Deep Dive](https://www.ciscolive.com/)
*   **Cisco Configuration Guide**: [Configuring WLAN Security - Cisco WLC 8.10](https://www.cisco.com/c/en/us/td/docs/wireless/controller/8-10/config-guide/b_cg810/security.html)
*   **White Paper**: [Wi-Fi Protected Access (WPA) Overview](https://www.cisco.com/c/en/us/support/docs/wireless-mobility/wireless-lan-wlan/67134-wpa-config.html)

---

## 📝 **補足（Notes）**
- **学習メモ**: 「TKIP は WEP 用ハードウェアのための絆創膏」と覚えましょう。
- **注意点**: ラボ試験で `AES` と `TKIP` の選択を間違えると、要件のスループットを満たせなくなる可能性があるため、常に AES 優先で考える必要があります。
