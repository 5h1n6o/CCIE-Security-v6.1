---
layout: default
title: 2.2-IOS-CA
nav_order: 2
parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.2 Cisco IOS CA for VPN authentication

Cisco IOSルータを**認証局（Certificate Authority: CA）**として動作させる機能は、外部のCAサーバー（Windows CAやパブリックCA）を用意することなく、サイト間VPNやリモートアクセスVPNでデジタル証明書ベースの強力な認証を実現するための効率的なソリューションです。デジタル証明書は、事前共有鍵（PSK）の管理負荷とセキュリティリスクを排除し、スケーラブルなVPN環境を構築するために不可欠です。

---

## 📘 概要

*   **機能概要**: IOSルータ上で**Cisco IOS Certificate Server**を有効にし、証明書の発行、管理、失効（CRL）を行う機能です。
*   **利用目的**: VPN接続時におけるデバイス間の相互認証にデジタル証明書を使用し、パスワード管理の複雑さを解消します。
*   **利用場面**:
    *   GET VPNにおいてキーサーバー（KS）とグループメンバー（GM）間の認証を自動化する場合。
    *   FlexVPNやAnyConnect環境で、多数のスポーク/クライアントに対して証明書を一括配布する場合。
    *   外部CAへのアウトバウンド通信が制限されているクローズドな環境。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要プロトコル** | **SCEP (Simple Certificate Enrollment Protocol)**、HTTP、PKI。 |
| **主なコンポーネント** | CAサーバー、トラストポイント（Trustpoint）、証明書署名要求（CSR）。 |
| **メリット** | 追加コストなし。ルータ単体で完結。SCEPによる自動更新。 |
| **デメリット** | CAルータのCPU/メモリ負荷。ルータの故障がPKI基盤の停止に直結する。 |
| **対応機種** | IOS/IOS-XE搭載ルータ（ISR, ASR, CSR1000vなど）。 |
| **制限事項** | 大規模環境では専用CAと比較して管理機能（GUI等）が限定的。 |
| **設計上の注意点** | **NTPによる時刻同期**が必須（時刻がずれると証明書が無効になる）。 |

---

## 🏗 動作原理

Cisco IOS CAは、パブリックキー・インフラストラクチャ（PKI）の中核として、信頼の起点（Root of Trust）となります。

```text
[ VPN Client / Spoke ]              [ IOS CA Server ]
          |                                 |
          |---- 1. SCEP GetCACert --------->| (Root証明書の配布)
          |<--- 2. CA Certificate ----------|
          |                                 |
          |---- 3. SCEP Enrollment (CSR) -->| (署名リクエスト)
          |                                 |
          | (Manual/Auto Grant)             | [ 署名プロセス ]
          |                                 |
          |<--- 4. Issued Certificate ------| (署名済み証明書の送付)
          |                                 |
          |---- 5. CRL Request ------------>| (失効リストの確認)
```

---

## ⚙ 動作シーケンス

1.  **CAサーバーの初期化**: RSAキーペアの生成とCAサーバー設定（発行者名、有効期限等）を行い、サービスを起動します。
2.  **トラストポイントの定義**: クライアント側でCAサーバーのURL（SCEP）を指定したトラストポイントを作成します。
3.  **CAの認証（Authenticate）**: CAの公開鍵（ルート証明書）をクライアントにダウンロードし、フィンガープリントで正当性を確認します。
4.  **登録（Enrollment）**: クライアントが自身のキーペアを作成し、SCEP経由で署名要求（CSR）をCAに送ります。
5.  **証明書発行**: CAが要求を承認（自動または手動）し、署名した証明書をクライアントに返します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **時刻同期（NTP）**: CA設定前に、全デバイスでNTPが同期されていることを必ず確認してください。同期していないと、証明書の有効期限外となりVPNが確立しません。
*   **GET VPNへの適用**: キーサーバー（KS）をCAとして構成し、GMが自動的に登録（Enroll）されるようにするシナリオは非常に重要です。
*   **パスワードによる失効（Revocation）**: 証明書発行時にチャレンジパスワードを設定し、失効手続きを確認する手順。
*   **証明書の保存場所**: `database url` コマンドで、発行した証明書のインデックスを flash や nvram のどこに保存するか指定する能力。
*   **SCEP以外の登録**: `enrollment terminal` を使用した、手動でのカット＆ペーストによる登録方法。
*   **Troubleshooting**: `show crypto pki certificates` を読み解き、証明書の用途（General Purpose等）や有効期限をチェックするスキル。

---

## 🛠 設定方法

### 1. IOS CA サーバーの設定（Root CA）
```bash
! RSAキーペアの作成
crypto key generate rsa label CA-KEY exportable modulus 2048

! CAサーバーの設定
crypto pki server IOS-CA
 issuer-name cn=IOS-CA,ou=Security,o=Cisco
 hash sha256
 lifetime certificate 365
 lifetime ca-certificate 1095
 database url flash:/ca-db
 database level complete
 grant auto  ! 署名リクエストを自動承認（ラボ試験では推奨されることが多い）
 no shutdown
```

### 2. VPN クライアント（スポーク）側の登録設定
```bash
! トラストポイントの定義
crypto pki trustpoint CA-TP
 enrollment url http://10.1.12.1:80
 revocation-check none ! ラボ環境ではCRLチェックをスキップすることが多い
!
! CAサーバーの認証（ルート証明書の取得）
crypto pki authenticate CA-TP
!
! 自身の証明書の登録
crypto pki enroll CA-TP
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **自身の保持証明書の確認** | <code>show crypto pki certificates</code> |
| **CAサーバーの状態確認** | <code>show crypto pki server</code> |
| **トラストポイントの状態** | <code>show crypto pki trustpoints</code> |
| **発行済み証明書のリスト** | <code>show crypto pki server IOS-CA certificates</code> |
| **PKI関連のトラブル追跡** | <code>debug crypto pki messages</code> |
| **SCEP通信のデバッグ** | <code>debug crypto pki transactions</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 証明書登録 (`enroll`) に失敗する | NTPの未同期 | <code>show clock</code> / <code>show ntp status</code> | <code>ntp server</code>を設定し同期を確認。 |
| CA認証 (`authenticate`) で接続不可 | HTTPサーバーが停止 | <code>show ip http server status</code> | CA側で<code>ip http server</code>を有効化。 |
| 証明書が "Pending" のまま | <code>grant auto</code>未設定 | <code>show crypto pki server</code> | CA側で<code>grant cert [ID]</code>で手動承認。 |
| VPN確立時に "Cert Invalid" | 失効確認 (CRL) の失敗 | <code>show run crypto pki</code> | 疎通がない場合は<code>revocation-check none</code>に変更。 |

---

## ⚠ 制限事項

*   **同一ルータ内制約**: CAサーバーとVPN終端を同じルータで行う場合、リソース消費に注意が必要です。
*   **キーサイズ**: IOSのバージョンにより、RSAキーの最大サイズ（4096など）や、サポートされるハッシュアルゴリズム（SHA-512等）に制限がある場合があります。
*   **バックアップ**: flash 内のデータベースが消去されると、証明書の再発行や失効管理ができなくなるため、バックアップが必須です。

---

## 🔄 他技術との関連

*   **GET VPN (1.4)**: KSの冗長化（COOP）環境でPKIを使用する場合、RSAキーのインポート/エクスポートが必要になります。
*   **IKEv2 (2.1)**: デフォルトの認証方式として証明書を使用する場合の基盤となります。
*   **AnyConnect (2.1)**: マシン証明書認証を行う際の証明書配布元として機能します。
*   **Infrastructure Segmentation (2.5)**: VRF環境下でのCAサーバーへの到達性確保。

---

## 🧩 比較表

### IOS CA vs 外部 CA (Windows/OpenSSL)

| 特徴 | Cisco IOS CA | 外部 CA サーバー |
| :--- | :--- | :--- |
| **構築コスト** | **ゼロ（既存ルータ活用）** | 高い（サーバー、OSライセンス） |
| **運用性** | CLI中心でシンプル | GUIによる高度な管理が可能 |
| **スケーラビリティ** | 小〜中規模向け | 大規模・全社基盤向け |
| **機能** | 基本的な署名・失効のみ | ポリシー、アーカイブ等豊富 |

---

## 💡 ベストプラクティス

1.  **Grant Autoの活用**: ラボ試験のような迅速な構築が求められる環境では `grant auto` を使用して承認作業を簡略化します。
2.  **専用キーペア**: CA自身が使用するキーペアは `label` を付けて管理し、他のVPN通信用キーと混同しないようにします。
3.  **永続性の確保**: `database url` には `flash:` などを指定し、再起動後も発行記録が残るようにします。
4.  **クロック精度**: 可能であれば、CAサーバー自身がNTPサーバーとして動作するか、信頼性の高い上位NTPを参照します。

---

## 📝 ラボ学習・設定サンプル例

### 1. GET VPN PKI 認証の実装 (KS = CA)
*   **要件**: R1 (KS) をCAとし、R4/R5 (GM) を証明書認証で参加させよ。
*   **設定**: R1で `crypto pki server` を起動。R4/R5で `enroll url` をR1に向けて登録。

### 2. SCEP チャレンジパスワードの使用
*   **要件**: 証明書登録時にパスワード "cisco123" を要求せよ。
*   **設定**: CAサーバー側で `password cisco123` を設定。

### 3. 証明書失効 (CRL) の検証
*   **要件**: R4の証明書を失効させ、VPN接続が拒否されることを確認せよ。
*   **実行**: CAで `revoke certificate [Serial]` を実行。

### 4. RSAキーのエクスポート/インポート
*   **要件**: 冗長KS (R5) にR1と同じCAキーを移行せよ。
*   **設定**: `crypto key export RSA ... pem terminal` で出力し、R5で `import`。

### 5. 有効期限の調整
*   **要件**: ルート証明書の有効期限を10年に設定せよ。
*   **設定**: `lifetime ca-certificate 3650`。

### 6. 手動登録 (Cut & Paste)
*   **要件**: HTTPプロトコルを使わずに証明書を登録せよ。
*   **設定**: クライアントで `enrollment terminal` を使用。

### 7. トラストポイントの複数定義
*   **要件**: 管理用とVPN用で異なるCAを使い分けよ。

### 8. FlexVPN IKEv2 Certificate Auth
*   **要件**: FlexVPNハブ・スポーク間で証明書認証を強制せよ。

### 9. 署名ハッシュの変更
*   **要件**: 証明書の署名に SHA-512 を使用せよ。
*   **設定**: `hash sha512` (CAサーバー設定下)。

### 10. 特定のインターフェイスでのCA待ち受け
*   **要件**: CAサーバーのSCEP受付を Loopback 0 のみに限定せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `crypto pki server` の出力で `Granting Mode: Manual` となっている場合、クライアントが `enroll` を実行した後の管理者のアクションは？
    *   **回答**: `show crypto pki server [NAME] requests` でIDを確認し、`crypto pki server [NAME] grant [ID]` を実行して手動で承認する必要がある。
2.  **トラブルシュート**: GET VPN環境で、GMがCAサーバーから証明書を取得できない。GMからCAのIPへのPingは通る。他に確認すべきIOSのサービスは？
    *   **回答**: CAルータ上で `ip http server` が有効になっているか（SCEPはHTTPを利用するため）。
3.  **Design**: 2台のキーサーバー（KS）が存在するGET VPN環境で、PKI認証を使用するための最も重要な前提条件は？
    *   **回答**: 両方のKSが同じCA証明書と秘密鍵（CAキーペア）を共有していること、または両方が共通の外部CAを信頼していること。
4.  **実装**: 証明書の有効期限が切れる前に自動で更新させたい。トラストポイントに追加すべきコマンドは？
    *   **回答**: `auto-enroll [Percentage]` (例: `auto-enroll 80` で有効期限の80%経過時に更新開始)。
5.  **Design**: IOS CAルータの `nvram:` 容量が非常に小さい場合、発行済み証明書のデータベースをどこに保存すべきか？
    *   **回答**: 外部のTFTPサーバー、または大容量の `flash:` や `usbflash:`。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [Cisco IOS PKI: Certificate Server Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_conn_pki/configuration/xe-16/sec-pki-xe-16-book/sec-pki-cert-srvr.html)
*   **Cisco Technical Notes**
    *   [Configuring a Router as a CA Server SCEP Example](https://www.cisco.com/c/en/us/support/docs/security-vpn/public-key-infrastructure-pki/116035-configure-ios-pki-00.html)
*   **Cisco Live (BRKSEC-2046)**
    *   [Advanced PKI and VPN Architectures](https://www.ciscolive.com/global.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「CAは信頼を売る商売」と覚えましょう。ルータをCAにする際は、まずそのルータ自体が正確な時間（NTP）を持っていることが、すべての信頼の前提になります。
*   **図解**: `authenticate` は「相手の印鑑（Root Cert）をもらうこと」、`enroll` は「自分の書類に印鑑（Sign）をもらうこと」と区別すると分かりやすくなります。
*   **注意点**: ラボ試験で `crypto key generate rsa` を実行する際、`label` を付け忘れるとデフォルトのキーが上書きされる可能性があるため、常にラベルを付ける癖をつけてください。
