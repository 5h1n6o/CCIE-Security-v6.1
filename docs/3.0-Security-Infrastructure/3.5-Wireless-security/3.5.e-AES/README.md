---
layout: default
title: 3.5.e-AES
nav_order: 3
parent: 3.5-Wireless-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.5.e AES (Advanced Encryption Standard)

**AES (Advanced Encryption Standard)** は、現代のネットワークセキュリティにおける暗号化の事実上の標準（デファクトスタンダード）です。CCIE Security v6.1 のブループリントでは「3.5 Wireless security technologies」の下に位置付けられていますが、無線 LAN（WPA2/WPA3）だけでなく、IPsec VPN、GET VPN、ストレージ暗号化、デバイス管理（SSH/SNMPv3）など、インフラ全体のセキュリティを支える基幹技術として深く理解する必要があります。

---

## 📘 概要

*   **機能概要**: 米国国家規格技術研究所（NIST）によって選定された、共通鍵（対称鍵）ブロック暗号アルゴリズムです。従来の DES や 3DES に代わるものとして開発されました。
*   **利用目的**: データの機密性（Confidentiality）を確保するために、データを固定長のブロック単位で暗号化します。
*   **どのような場面で利用するか**: 
    *   **無線 LAN**: WPA2 (AES-CCMP) および WPA3 (AES-GCMP) におけるデータ保護。
    *   **VPN**: IPsec (IKEv1/IKEv2) のトランスフォームセットにおけるパケット暗号化。
    *   **管理通信**: SSH や SNMPv3 によるセキュアなデバイス管理。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **アルゴリズム種別** | 共通鍵暗号（Symmetric Key）、ブロック暗号。 |
| **ブロック長** | 128 ビット固定。 |
| **鍵長（Key Size）** | 128, 192, 256 ビットの 3 種類。 |
| **主なモード** | CBC (Cipher Block Chaining), GCM (Galois/Counter Mode)。 |
| **無線での名称** | CCMP (Counter Mode with CBC-MAC)。 |
| **メリット** | 高速、高い安全性、ハードウェアアクセラレーションによる低負荷。 |
| **デメリット** | 送信者と受信者で事前に安全に鍵を共有する必要がある。 |

---

## 🏗 動作原理

AES は、データを 4x4 の行列（State）として扱い、複数の「ラウンド」を通じて変換を行います。

```text
明文 (128-bit block)
   ↓
[ AddRoundKey ] --- 最初の鍵の適用
   ↓
[ Round 1 to N-1 ]
   ├─ SubBytes (非線形置換)
   ├─ ShiftRows (行の入れ替え)
   ├─ MixColumns (列の混同)
   └─ AddRoundKey (ラウンド鍵の適用)
   ↓
[ Final Round ] --- MixColumns を除く処理
   ↓
暗号文 (128-bit block)
```

### 主要な動作モード
1.  **AES-CBC**: 前のブロックの暗号文を次のブロックの入力と XOR 演算する連鎖方式。整合性確認には別途 HMAC などが必要です。
2.  **AES-GCM**: 暗号化と認証（整合性確認）を同時に行う「認証付き暗号」。並列処理が可能で非常に高速かつ高効率なため、IKEv2 や WPA3 で推奨されます。

---

## ⚙ 動作シーケンス (IPsec VPN 連携例)

1.  **SA 交渉**: IKE フェーズ 1 またはフェーズ 2 において、サポートする AES 鍵長とモードを提案します。
2.  **共通鍵生成**: Diffie-Hellman 鍵交換などを通じて、AES で使用する共通鍵を動的に生成します。
3.  **暗号化**: 送信側デバイスがパケットを 128 ビットブロックに分割し、AES アルゴリズムを適用してカプセル化（ESP）します。
4.  **復号**: 受信側デバイスが同じ共通鍵を使用して AES 処理を逆転させ、元のパケットを復元します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **鍵長の整合性**: ラボ要件で `AES-256` が指定されている場合、対向デバイスや ISE/WLC 側でも正確に 256 ビットを選択しているか確認が必要です。128 ビットと 256 ビットでは SA が成立しません。
*   **GCM モードの活用**: IKEv2 を使用する問題では、`esp-gcm` をトランスフォームセットに含めることで、暗号化と整合性確認を一括で処理する効率的な構成が求められることがあります。
*   **無線 LAN セキュリティ**: 
    *   `show client detail` の出力において、`Policy Type: WPA2` や `AKM: 802.1x` と共に、暗号化方式が `AES (CCMP)` になっていることを確認するスキルが必須です。
    *   WPA3 構成時には GCMP（GCM の無線版）が使用されることを意識してください。
*   **ハードウェア支援**: `show crypto engine accelerator statistic`（機器による）などで、AES 処理が CPU ではなくハードウェアで加速されているかを確認させられる場合があります。

---

## 🛠 設定方法

### 1. IOS-XE IPsec Transform-set (AES-CBC 256)
```bash
crypto ipsec transform-set TSET esp-aes 256 esp-sha-hmac
 mode tunnel
```

### 2. IOS-XE IKEv2 Proposal (AES-GCM)
```bash
crypto ikev2 proposal PROP1
 encryption aes-gcm-256
 group 14
! IKEv2 GCM の場合、整合性（Integrity）は暗号化に含まれるため別途設定不要
```

### 3. WLC (AireOS) での AES 有効化 (CLI)
```bash
config wlan security wpa encryption aes enable 10
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **IPsec SA での使用状況確認** | <code>show crypto ipsec sa</code> |
| **ISAKMP (IKEv1) ポリシー確認** | <code>show crypto isakmp sa det</code> |
| **無線クライアントの暗号方式確認** | <code>show client detail [MAC]</code> |
| **GET VPN の TEK ポリシー確認** | <code>show crypto gdoi ks policy</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| IKE Phase 2 (QM) 失敗 | AES 鍵長（128/256）の不一致 | <code>debug crypto isakmp</code> | 両端で鍵長を一致させる。 |
| スループットが上がらない | 非対応モード（CBC）による負荷 | <code>show processes cpu</code> | GCM モードへの変更を検討。 |
| 無線クライアント切断 | WLC と端末の AES/TKIP 不一致 | <code>debug client [MAC]</code> | <code>AES</code> のみを有効にする。 |

---

## ⚠ 制限事項

*   **レガシーデバイス**: 非常に古いデバイスは AES をサポートしておらず、3DES や TKIP にフォールバックする必要がありますが、これはセキュリティ低下を招きます。
*   **輸出規制**: 高度な暗号化（K9 イメージ）ライセンスが適用されていない Cisco 機器では、AES-256 などの強力な鍵長が使用できない場合があります。
*   **ブロック長**: ブロック長が 128 ビット固定であるため、数テラバイト規模の同一鍵による通信では「誕生日攻撃」のリスクが理論上存在します（通常は鍵の再生成で回避）。

---

## 🔄 他技術との関連

*   **3.4.d Port Security**: レイヤ 2 レベルの制限。AES はその上のデータ保護を担います。
*   **IKEv2 (300-730)**: AES-GCM を最大限活用できる次世代 VPN プロトコル。
*   **Cisco ISE**: 無線クライアントが AES を使用して 802.1X 認証を行う際のバックエンド。
*   **FIPS 140-2**: AES は米国政府の暗号基準 FIPS に準拠しており、公共機関の案件では AES の使用が必須要件となります。

---

## 🧩 比較表

### AES-CBC vs AES-GCM

| 機能 | AES-CBC | AES-GCM (推奨) |
| :--- | :--- | :--- |
| **処理内容** | 暗号化のみ | **暗号化 + 認証 (AEAD)** |
| **整合性確認** | 別途 HMAC が必要 | **内蔵 (GMAC)** |
| **パフォーマンス** | 低（順次処理が必要） | **高（並列処理が可能）** |
| **オーバーヘッド** | 大 | **小** |
| **主な用途** | IKEv1, 旧型 VPN | **IKEv2, AnyConnect, WPA3** |

---

## 💡 ベストプラクティス

1.  **AES 256 を標準とする**: セキュリティ要件が厳しい環境では、常に 256 ビット鍵を選択します。
2.  **GCM モードの優先**: パフォーマンスとセキュリティを両立させるため、可能な限り GCM (Galois Counter Mode) を使用します。
3.  **DES/3DES の廃止**: これらは脆弱とみなされているため、設定から排除し AES に移行します。
4.  **鍵の再生成（Rekey）**: `security-association lifetime` を適切に設定し、長期間同じ AES 鍵を使い続けないようにします。

---

## 📝 ラボ学習・設定サンプル例

### 1. Site-to-Site VPN (AES-128)
*   **問題**: R1 と R2 間で AES-128 を使用した VPN を構築せよ。
*   **要件**: IKEv1, AES-128, SHA, Group 2.
*   **設定**: `crypto isakmp policy 10`, `encryption aes 128`。

### 2. GET VPN TEK Policy (AES-256)
*   **要件**: GET VPN 環境でデータ通信を AES-256 で保護せよ。
*   **設定**: `crypto gdoi ks policy` 下で `transform-set esp-aes 256` を指定。

### 3. WPA2-Enterprise with AES-Only
*   **要件**: SSID "CORP" で TKIP を禁止し、AES 暗号化のみを許可せよ。

### 4. DMVPN Tunnel Protection (AES)
*   **要件**: DMVPN Phase 3 構成において、トンネルを AES-256 で暗号化せよ。

### 5. IKEv2 AnyConnect (AES-GCM)
*   **要件**: AnyConnect クライアントに対し、AES-GCM-256 による高速通信を提供せよ。

### 6. SNMPv3 Privacy (AES)
*   **要件**: 管理通信を保護するため、SNMPv3 のプライバシープロトコルに AES を指定せよ。

### 7. SSH Hardening
*   **要件**: スイッチへの SSH 接続において、強力な AES アルゴリズムのみを許可せよ。
*   **設定**: `ip ssh encryption algorithm aes256-ctr` (機器による)。

### 8. Troubleshooting Transform Mismatch
*   **課題**: 片側が `esp-aes 128`、他方が `esp-aes 256` のため VPN が成立しない状況を解決せよ。

### 9. Wireless WPA3 Suite-B
*   **要件**: 政府基準に準拠するため、WPA3 192-bit セキュリティ (GCMP-256) を構成せよ。

### 10. GET VPN Rekey 保護
*   **要件**: GET VPN の Rekey メッセージ（KEK）自体を AES で暗号化せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `crypto ipsec transform-set T1 esp-gcm 256` が設定されている。この設定において、パケットの整合性（Integrity）を確保するために追加で必要な設定は何か？
    *   **回答**: 不要。GCM モードは認証付き暗号であり、整合性確認機能が組み込まれている。
2.  **トラブルシュート**: 無線クライアントが WPA2 WLAN に接続できるが、スループットが 54Mbps を超えない。WLC の設定で確認すべき点は？
    *   **回答**: 暗号化方式に `TKIP` が含まれていないか。802.11n 以降の高速レートには `AES` が必須である。
3.  **Design**: 拠点間 VPN で CPU 負荷を抑えつつ 1Gbps 以上のスループットを目指す場合に推奨される AES モードは？
    *   **回答**: **AES-GCM**。並列処理が可能でハードウェアアクセラレーションに適しているため。
4.  **実装**: GET VPN の Group Member (GM) において、KS から受け取ったポリシーが AES-256 であることを確認するコマンドは？
    *   **回答**: `show crypto gdoi gm policy`。
5.  **コンフィグ読解**: `show crypto isakmp sa det` の出力で `Encr: aes`, `Hash: sha`, `Auth: psk` とある。これは IKE のどのフェーズの状態か？
    *   **回答**: **IKE Phase 1** (ISAKMP SA)。

---

## 🔗 参考リソース

*   **Cisco Live (BRKSEC-2003)**: [Wireless Security Deep Dive](https://www.ciscolive.com/)
*   **Cisco Configuration Guide**: [Configuring IPsec VPNs](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_conn_vpnips/configuration/xe-16/sec-conn-vpnips-xe-16-book.html)
*   **Cisco White Paper**: [Next Generation Encryption (NGE)](https://www.cisco.com/c/en/us/about/security-center/next-generation-cryptography.html)

---

## 📝 **補足（Notes）**
*   **学習メモ**: 「AES は金庫の鍵」です。鍵の長さ（128/256）が金庫の頑丈さを決め、モード（CBC/GCM）が金庫への出し入れの仕方を決めると覚えましょう。
*   **図解**: `Transform-id: ESP_AES` という表示を `show` コマンドの結果で見つけたら、それが VPN データの心臓部です。
*   **注意点**: CCIE Security v6.1 では、古い技術（DES/3DES/TKIP）を排除し、NGE（Next Generation Encryption）である AES への移行を正しく実装できるかが問われます。
