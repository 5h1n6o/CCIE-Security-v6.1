---
layout: default
title: 3.5.c-WPA3
nav_order: 3
parent: 3.5-Wireless-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.5.c WPA3

**WPA3 (Wi-Fi Protected Access 3)** は、2018年に Wi-Fi Alliance によって発表された次世代の無線セキュリティ規格です。WPA2 の脆弱性（オフライン辞書攻撃や KRACK など）を克服し、公共 Wi-Fi での暗号化（OWE）や、パスワードベースの認証における「前方秘匿性（Forward Secrecy）」を提供します。CCIE Security v6.1 のラボ試験では、**Cisco Catalyst 9800 シリーズ WLC** や最新の **AireOS (8.10以降)** における実装と、ISE との連携が重要なターゲットとなります。

---

## 📘 概要

*   **機能概要**: 従来の WPA2 を強化し、より強力な暗号化アルゴリズムと新しいハンドシェイクプロトコル（SAE）を導入したセキュリティフレームワークです。
*   **利用目的**: ブルートフォース攻撃や辞書攻撃からの保護、パブリック無線 LAN でのユーザー間の機密保持、エンタープライズ環境での 192ビット強力暗号の適用。
*   **どのような場面で利用するか**: 
    *   **WPA3-Personal**: 家庭や小規模拠点でパスワードによる接続を行う際、辞書攻撃を防ぐために利用。
    *   **WPA3-Enterprise**: 政府機関や金融機関など、極めて高い機密性が求められる環境で 192ビット暗号（CNSA スイート）を利用。
    *   **OWE (Enhanced Open)**: カフェや空港などのオープン Wi-Fi で、パスワードなしでも無線区間を暗号化したい場合に利用。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | **SAE (Simultaneous Authentication of Equals)** によるハンドシェイクの保護。 |
| **用途** | 従来の WPA2 ネットワークの置き換え、高セキュリティ環境の構築。 |
| **メリット** | オフライン辞書攻撃への耐性、**PMF (Protected Management Frames)** の必須化。 |
| **デメリット** | レガシーな古いクライアント端末との互換性問題。 |
| **対応機種** | Catalyst 9800 シリーズ WLC、Wave 2 以降の AP。 |
| **制限事項** | WPA3 を有効にするには **PMF を「Required」** に設定する必要がある。 |
| **設計上の注意点** | 移行期には WPA2 と WPA3 を共存させる **Transition Mode** の検討が必要。 |

---

## 🏗 動作原理

WPA3-Personal の核心は、従来の PSK (Pre-Shared Key) に代わる **SAE (Dragonfly ハンドシェイク)** です。

```text
[ Wireless Client ]             [ Access Point / WLC ]
        |                               |
        |<------ SAE Commit ----------->| (パスワードに基づく鍵導出)
        |                               |
        |<------ SAE Confirm ---------->| (鍵の正当性を確認)
        |                               |
        |<--- 4-Way Handshake (PTK) --->| (通信用一時鍵の生成)
        |                               |
        |<------ Encrypted Data ------->| (AES-CCMP/GCMP)
```

SAE は「パスワードが推測可能であっても、ハンドシェイクの傍受だけでは鍵を導出できない」という特性（前方秘匿性）を持ちます。

---

## ⚙ 動作シーケンス

1.  **SAE Exchange**: クライアントと AP 間でパスワードに基づいた Diffie-Hellman 要素を交換し、共有の PMK (Pairwise Master Key) を導出します。
2.  **4-Way Handshake**: 導出された PMK を用いて、実際のデータ暗号化に使用する一時鍵 (PTK/GTK) を生成します。
3.  **PMF の強制**: 管理フレーム（切断要求など）が暗号化・署名され、攻撃者による不正な切断攻撃を防止します。
4.  **192-bit Mode (Enterprise)**: 802.1X 認証後、GCMP-256 (Galois Counter Mode Protocol) による非常に強力な暗号化を適用します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **PMF (802.11w) の要件**: WPA3 を設定する際、**PMF を "Required"** に設定し忘れると、設定が適用されない、あるいはクライアントが接続できないトラブルが発生します。
*   **Transition Mode の設定**: ラボ要件で「WPA3 を導入しつつ、古い WPA2 クライアントの接続も維持せよ」と指示された場合、Transition Mode を選択し、AKM リストに PSK と SAE の両方を含める必要があります。
*   **OWE (Enhanced Open) の実装**: 「パスワードは不要だが、スニッフィングを防げ」という要件に対し、OWE を正しく構成できるかが問われます。
*   **H2E (Hash-to-Element)**: SAE の最新のセキュリティ強化オプションであり、特定の WLC バージョンで設定が求められる可能性があります。
*   **Show コマンドによる識別**: `show client detail` の出力において、`AKM: SAE` や `Encryption: AES (CCMP256/GCMP256)` が表示されているかを確認します。

---

## 🛠 設定方法 (Catalyst 9800 WLC CLI)

### 1. WPA3-SAE (Personal)
```bash
wlan CORP-WPA3 10 CORP-WPA3
 security wpa akm sae
 security wpa wpa3
 security pmf required
 no shutdown
```

### 2. WPA3 Transition Mode
```bash
wlan TRANS-WIFI 11 TRANS-WIFI
 security wpa akm psk sae
 security pmf optional
 no shutdown
```

### 3. WPA3-Enterprise (192-bit)
```bash
wlan ENT-192 12 ENT-192
 security wpa akm suiteb-gcmp256
 security wpa wpa3
 security pmf required
 no shutdown
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **クライアントの認証方式確認** | <code>show wireless client mac-address [MAC] detail</code> |
| **WLAN のセキュリティ設定表示** | <code>show wlan name [WLAN_NAME]</code> |
| **PMF の稼働状態確認** | <code>show wireless client mac-address [MAC] detail \| include PMF</code> |
| **AP 側の WPA3 ステータス** | <code>show ap name [AP_NAME] config dot11 5ghz</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| WPA3 クライアントが接続できない | **PMF が Disabled** になっている | <code>show wlan</code> を確認。PMF を <code>Required</code> に設定。 |
| 旧端末が Transition Mode で繋がらない | クライアント側が Transition Mode ビーコンを正しく解釈できない | 特定のデバイス用に WPA2 専 SSID を分離する。 |
| SAE ハンドシェイクが失敗する | パスワード（SAE Password）の不整合 | <code>debug wireless client</code> で SAE フェーズのエラーを確認。 |
| 192ビット認証が通らない | 暗号スイートの不一致 (CCMP vs GCMP) | AKM と暗号化設定 (SuiteB) の整合性を確認。 |

---

## ⚠ 制限事項

*   **ハードウェア依存**: 192ビットモードを使用するには、クライアントと AP の双方が GCMP-256 をサポートしている必要があります。
*   **SSID の制限**: 一部の WLC バージョンでは、WPA3 設定時に特定の QoS や Fast Transition (802.11r) の組み合わせに制約があります。
*   **レガシー WLC**: 古い AireOS WLC（v8.5 以前など）では WPA3 はサポートされません。

---

## 🔄 他技術との関連

*   **3.4.f RA Guard / 3.4.e DHCP Snooping**: 無線クライアントに対する L2 セキュリティ機能。
*   **Cisco ISE**: WPA3-Enterprise におけるバックエンド RADIUS 認証。
*   **802.11ax (Wi-Fi 6)**: WPA3 は Wi-Fi 6 認定の必須条件となっています。
*   **TrustSec (SGT)**: 無線クライアントに対する SGT 付与と、WLC でのポリシー強制。

---

## 🧩 比較表

### WPA2-PSK vs WPA3-SAE

| 特徴 | WPA2-PSK | WPA3-SAE |
| :--- | :--- | :--- |
| **辞書攻撃耐性** | 低い（パッシブキャプチャで解読可能） | **非常に高い（SAE により保護）** |
| **前方秘匿性 (PFS)** | なし | **あり（パスワード漏洩後も過去の通信を保護）** |
| **PMF** | 任意 (Optional) | **必須 (Required)** |
| **オープン接続保護** | なし | **OWE による暗号化** |

---

## 💡 ベストプラクティス

1.  **段階的移行**: 最初は Transition Mode (WPA2+WPA3) を使用し、全端末の対応を確認後に WPA3-Only へ移行します。
2.  **PMF の徹底**: WPA2 ネットワークであっても PMF を有効化し、管理フレームの改ざんを防止します。
3.  **SAE H2E の活用**: セキュリティレベルを高めるため、可能な限り Hash-to-Element (H2E) をサポートする構成にします。
4.  **ISE 連携の強化**: Enterprise モードでは証明書ベースの認証 (EAP-TLS) を標準とし、WPA3 の 192ビット暗号を活用します。

---

## 📝 ラボ学習・設定サンプル例

### 1. WPA3-Personal Only
*   **要件**: SSID "WPA3-ONLY" を作成し、WPA3-SAE のみを許可せよ。
*   **設定**: `security wpa akm sae`, `security pmf required`。

### 2. 移行モードの実装
*   **要件**: WPA2-PSK と WPA3-SAE の両方をサポートする SSID を作成せよ。
*   **設定**: `security wpa akm psk sae`, `security pmf optional`。

### 3. OWE (Enhanced Open)
*   **要件**: 公共スペース向けに、認証なしで無線区間を保護せよ。
*   **設定**: `security wpa akm owe`。

### 4. 192-bit セキュリティ
*   **要件**: 最高機密を扱うため、WPA3 192ビットモードを設定せよ。

### 5. Fast Transition (FT) との併用
*   **要件**: WPA3 接続環境で AP 間の高速ローミングを有効にせよ。
*   **設定**: `security wpa akm sae sae-ft`。

### 6. 特定の SAE パスワード設定
*   **要件**: WLAN 全体とは別に、特定の識別子に対応した SAE パスワードを定義せよ。

### 7. IPv6 セキュリティの統合
*   **要件**: WPA3 ネットワーク上で IPv6 RA Guard を有効にせよ。

### 8. PMF のデバッグ検証
*   **課題**: WPA3 クライアントが切断された際の PMF 保護ログを `debug client` で確認せよ。

### 9. ISE による VLAN 配布
*   **要件**: WPA3-Enterprise 認証成功後、ISE から動的に VLAN 20 を割り当てよ。

### 10. クライアント情報の読み取り
*   **課題**: `show client detail` から、現在のクライアントが WPA3 のどの暗号スイートを使用しているか特定せよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: クライアントが WPA3-Only の SSID にアソシエーションできない。WLC の設定を確認したところ `security pmf optional` となっている。修正すべき箇所は？
    *   **回答**: WPA3-Only 規格では PMF は **`Required`** である必要がある。
2.  **Design**: 辞書攻撃に脆弱なレガシーな WPA2-PSK 環境を、パスワードを維持したまま強化する手法は？
    *   **回答**: WPA3-SAE への移行（または Transition Mode の導入）。
3.  **コンフィグ読解**: `show wlan` の出力で AKM に `SAE-FT` とある。これは何を意味するか？
    *   **回答**: WPA3-SAE において、802.11r (Fast Transition) が有効化されている。
4.  **実装**: パスワードなしのゲスト Wi-Fi で、ユーザー間のパケットスニッフィングを防止するために実装すべき AKM は？
    *   **回答**: **OWE (Opportunistic Wireless Encryption)**。
5.  **トラブルシュート**: WPA3 Transition Mode を有効にしたが、古い Android 端末が接続に失敗する。まず検討すべき対策は？
    *   **回答**: クライアント側の WPA3 非互換性を疑い、WPA2 専 SSID を別途用意する。

---

## 🔗 参考リソース

*   **Cisco WLC 8.10 Configuration Guide**
    *   [Configuring WPA3 Security](https://www.cisco.com/c/en/us/td/docs/wireless/controller/8-10/config-guide/b_cg810/security.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [Wireless Security Deep Dive](https://www.ciscolive.com/)
*   **Technical Notes**
    *   [WPA3 Deployment Guide for Cisco Catalyst 9800 WLC](https://www.cisco.com/c/en/us/td/docs/wireless/controller/technotes/8-8/b_wpa3_deployment_guide.html)

---

## 📝 **補足（Notes）**
*   **学習メモ**: WPA3 は「SAE」と「PMF Required」の 2 点セットで覚えるのが合格への近道です。
*   **図解**: 従来の `Policy Type: WPA2` に代わって `Policy Type: WPA3` または `WPA3(SAE)` という表示を `show` コマンドで探せるようにしてください。
*   **注意点**: ラボ試験では Catalyst 9800 (IOS-XE) と AireOS の両方の構文に慣れておく必要があります。
