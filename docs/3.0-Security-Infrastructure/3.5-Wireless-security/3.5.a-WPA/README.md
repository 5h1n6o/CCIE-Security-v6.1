---
layout: default
title: 3.5.a-WPA
nav_order: 1
parent: 3.5-Wireless-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.5.a WPA (Wi-Fi Protected Access)

**WPA (Wi-Fi Protected Access)** は、脆弱性が指摘された WEP (Wired Equivalent Privacy) の代替として Wi-Fi Alliance によって策定されたセキュリティ規格です。CCIE Security v6.1 の文脈では、レガシーな技術としての理解に加え、**WLC (Wireless LAN Controller)** における設定方法や、後継の WPA2/WPA3 との差異、および暗号化プロトコル（TKIP/AES）との組み合わせを正確に把握することが求められます。

---

## 📘 概要

*   **機能概要**: 無線 LAN 通信における認証と暗号化のフレームワークです。WEP の弱点を克服するため、動的な鍵更新を行う **TKIP (Temporal Key Integrity Protocol)** を導入しました。
*   **利用目的**: 無線通信の盗聴防止（秘匿性）および不正アクセス防止（認証）。
*   **どのような場面で利用するか**: 
    *   WPA2/WPA3 に対応していない古いレガシーデバイスをネットワークに接続する必要がある場合。
    *   移行期の暫定的なセキュリティ確保。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な認証方式** | Personal (PSK: 事前共有鍵) と Enterprise (802.1X/EAP)。 |
| **主要暗号プロトコル** | **TKIP** (デフォルト) ですが、AES-CCMP もサポート可能。 |
| **鍵管理** | **4-Way Handshake** による一時鍵 (PTK) およびグループ鍵 (GTK) の生成。 |
| **完全性チェック** | **Michael** アルゴリズム（MIC）を使用して改ざんを検知。 |
| **対応機種** | 全ての Cisco WLC、アクセスポイント (AP)。 |
| **制限事項** | TKIP を使用すると、スループットが最大 54Mbps (802.11g) に制限される。 |
| **設計上の注意点** | セキュリティ強度が低いため、現在では **WPA2 以上が推奨** される。 |

---

## 🏗 動作原理

WPA は、認証、鍵交換、暗号化の 3 つのフェーズで動作します。

```text
Wireless Client                  WLC / AP                    RADIUS (ISE)
      |                              |                             |
      | <--- 802.11 Association ---> |                             |
      |                              |                             |
      | <====== 802.1X / EAP Authentication (Enterprise) =======> |
      |                              |                             |
      | <------ 4-Way Handshake ----> | (Generate PTK/GTK)          |
      |                              |                             |
      | <------ Encrypted Data -----> | (TKIP/AES Encryption)       |
```

1.  **Authentication**: クライアントの正当性を確認（PSK または RADIUS/ISE 連携）。
2.  **Key Management**: 4-Way Handshake を行い、セッションごとに異なる暗号鍵を生成します。
3.  **Data Confidentiality**: 生成された鍵を用いてパケットを暗号化します。

---

## ⚙ 動作シーケンス

1.  **ビーコン/プローブ応答**: AP が WPA 対応をブロードキャスト。
2.  **アソシエーション**: クライアントが WPA ポリシーを選択して接続要求。
3.  **認証**: 
    *   **PSK**: 手動設定された鍵で照合。
    *   **802.1X**: EAP プロトコルを用いて RADIUS サーバで認証。
4.  **4-Way Handshake**:
    *   ANonce, SNonce を交換。
    *   ペアワイズ一時鍵 (PTK) を算出。
    *   グループ一時鍵 (GTK) を WLC からクライアントへ配布。
5.  **通信開始**: 暗号化されたトラフィックの転送。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **WPA ポリシーの識別**: `show client detail` コマンドの結果から、`Policy Type: WPA` または `WPA2` を見分ける必要があります。
*   **AKM (Authentication Key Management)**: `802.1x` または `PSK` のどちらが指定されているかを確認してください。
*   **レガシー対応設定**: ラボで「古い端末のみ WPA-TKIP を許可せよ」という要件が出た場合、WLAN 設定で WPA(WPA1) を有効にする必要があります。
*   **RADIUS サーバ連携**: RADIUS サーバ（ISE）の IP と共有鍵を正確に WLC に登録し、WLAN に関連付ける手順は必須です。
*   **ACL との組み合わせ**: クライアントの認証後に適用される `IPv4 ACL Name` が正しく反映されているかを確認します。

---

## 🛠 設定方法 (WLC CLI)

### 1. RADIUS サーバの定義
```bash
config radius auth add 1 192.168.10.100 1812 ascii Cisco123
```

### 2. WLAN の作成と WPA-Enterprise (802.1X) の設定
```bash
config wlan create 10 WPA-LAB
! WPA (WPA1) ポリシーの有効化
config wlan security wpa enable 10
! TKIP 暗号化を許可
config wlan security wpa encryption tkip enable 10
! 802.1X 認証を有効化
config wlan security wpa akm 802.1x enable 10
! RADIUS サーバの適用
config wlan radius auth add 10 1
config wlan enable 10
```

### 3. WPA-PSK (Personal) の設定
```bash
config wlan security wpa akm psk enable 10
config wlan security wpa psk set 10 ascii SecretPassword
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **クライアントの接続詳細確認** | <code>show client detail [MAC_ADDR]</code> |
| **WLAN のセキュリティ設定表示** | <code>show wlan [WLAN_ID]</code> |
| **RADIUS サーバの状態確認** | <code>show radius summary</code> |
| **AP 接続クライアント数確認** | <code>show ap summary</code> |
| **認証プロセスのデバッグ** | <code>debug client [MAC_ADDR]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| クライアントが認証に失敗する | RADIUS 共有鍵の不一致 | <code>show radius summary</code> で接続確認。 |
| Policy Type が WPA2 になる | WPA1 が無効化されている | <code>config wlan security wpa enable</code> を確認。 |
| 通信が頻繁に切断される | MIC 失敗による対抗措置 | <code>show logging</code> で Michael MIC エラーを確認。 |
| ACL が適用されない | ACL 名の不一致 | <code>show wlan [ID]</code> で適用 ACL 名を再確認。 |

---

## ⚠ 制限事項

*   **パフォーマンス**: TKIP を使用する場合、802.11n 以上の高速データレートは利用できません。
*   **セキュリティ脆弱性**: TKIP には既知の攻撃手法があるため、**PCI-DSS 等のコンプライアンス環境では使用不可**とされることが多いです。
*   **互換性**: 現代の最新クライアント（iOS/Android 等）では WPA(WPA1) をデフォルトで拒否する場合があります。

---

## 🔄 他技術との関連

*   **3.4.e DHCP Snooping**: 無線クライアントに対する IP 検証に使用されます。
*   **ISE (350-701)**: WPA-Enterprise におけるバックエンド認証基盤です。
*   **802.11w (PMF)**: 管理フレーム保護。WPA3 では必須ですが、WPA では一般的にサポートされません。

---

## 🧩 比較表

### WPA vs WPA2

| 特徴 | WPA (3.5.a) | WPA2 (3.5.b) |
| :--- | :--- | :--- |
| **主要暗号** | **TKIP** | **AES-CCMP** |
| **完全性確認** | Michael (64-bit) | CBC-MAC (128-bit) |
| **認証方式** | 802.1X / PSK | 802.1X / PSK |
| **セキュリティ強度** | 低 | 高 |
| **スループット制限** | あり (54Mbps) | なし |

---

## 💡 ベストプラクティス

1.  **TKIP は避ける**: 可能な限り AES 暗号化のみを使用するように構成します。
2.  **Enterprise 優先**: PSK よりも ISE と連携した 802.1X 認証を推奨します。
3.  **WLAN 分離**: レガシー端末向けに WPA を残す場合は、別の SSID (WLAN) を作成し、アクセス制御を厳格にします。
4.  **RADIUS 冗長化**: プライマリ RADIUS ダウンに備え、セカンダリを必ず設定します。

---

## 📝 ラボ学習・設定サンプル例

1.  **WPA-PSK 設定**: `WLAN 1` に WPA-TKIP とパスワード `cisco123` を設定せよ。
2.  **Enterprise 認証**: ISE サーバ `10.1.1.1` を使用した 802.1X WLAN を作成せよ。
3.  **移行モード設定**: WPA と WPA2 の両方をサポートする WLAN を作成せよ。
4.  **ACL 適用**: 無線クライアントに `GUEST_ACL` を適用せよ。
5.  **スループット検証**: TKIP 有効時にデータレートが制限されることを `show client detail` で確認せよ。
6.  **RADIUS タイムアウト設定**: 再認証インターバルを 1 時間に設定せよ。
7.  **SSID 隠蔽**: WPA 有効 WLAN の SSID ブロードキャストを停止せよ。
8.  **AP 制限**: 特定の AP グループでのみ WPA WLAN を有効にせよ。
9.  **VLAN 割り当て**: RADIUS 属性により動的に VLAN 20 を割り当てる設定をせよ。
10. **デバッグ実行**: クライアント接続時の 4-Way Handshake ログをキャプチャせよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `config wlan security wpa encryption tkip enable` が設定されている場合、無線スループットはどうなるか？
    *   **回答**: 802.11g の上限である 54Mbps に制限される。
2.  **トラブルシュート**: `show client detail` で `Policy Type` が `Unknown` と表示される原因は？
    *   **回答**: クライアントの認証が完了していない、または WLC とのポリシー交渉に失敗している。
3.  **Design**: レガシーデバイスと最新デバイスが混在する環境で推奨される WPA 設定は？
    *   **回答**: WPA と WPA2 の移行モード (Mixed Mode) を有効にし、TKIP と AES の両方を許可する。
4.  **実装**: WLC で RADIUS サーバを登録したが WLAN で選択できない。なぜか？
    *   **回答**: RADIUS サーバが `Auth` 用として定義されていない、または WLAN のセキュリティ設定で `802.1X` が AKM として選択されていない。
5.  **コンフィグ読解**: の図で `Authentication Key Management` が `802.1x` となっている。これは何を意味するか？
    *   **回答**: WPA または WPA2 の Enterprise モードが使用されており、RADIUS サーバによる認証が必要である。

---

## 🔗 参考リソース

*   [Cisco WLC 8.10 Configuration Guide: Configuring Security Solutions](https://www.cisco.com/c/en/us/td/docs/wireless/controller/8-10/config-guide/b_cg810/security.html)
*   [Wi-Fi Protected Access (WPA) - Cisco Support Notes](https://www.cisco.com/c/en/us/support/docs/wireless-mobility/wireless-lan-wlan/67134-wpa-config.html)
*   [Cisco Live: Wireless Security Deep Dive (BRKSEC-2003)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**
*   **学習メモ**: WPA(WPA1) は WPA2 の登場によりその役割を終えましたが、ラボ試験では「レガシー対応」や「設定ミス探し」として出題される可能性があります。
*   **注意点**: 実際の WLC GUI では `WPA Policy` のチェックボックスをオンにする必要があります。CLI と GUI の対応関係を整理しておきましょう。
