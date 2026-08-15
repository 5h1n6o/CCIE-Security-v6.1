---
layout: default
title: 4.6-BYOD
nav_order: 6
parent: 4.0-Identity-Management
---

# 4.6 BYOD on-boarding and network access flows

Cisco ISE（Identity Services Engine）を使用した **BYOD（Bring Your Own Device）** は、従業員の個人所有デバイスをセキュアに企業ネットワークへ接続させるためのソリューションです。デバイスの識別、証明書の自動発行、プロファイルのインストールを自動化する「オンボーディング」プロセスがその核心となります。

---

## 📘 概要

*   **機能概要**: 未知の個人デバイスがネットワークに接続した際、ユーザー認証を経て、証明書のインストールやWi-Fi設定の自動構成を行い、管理された状態にする機能です。
*   **利用目的**: 管理負荷を抑えつつ、私物デバイスに適切なアクセス権限（SGTやVLAN）を割り当て、セキュアなアクセスを保証すること。
*   **どのような場面で利用するか**:
    *   従業員が個人のiPhoneやAndroidを社内Wi-Fiに接続する場合。
    *   デバイスごとに一一証明書を手動インストールするのが困難な大規模環境。
    *   特定のセキュリティ基準（パスコードロック等）を満たしたデバイスのみ接続を許可する場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **オンボーディング方式** | **Single SSID**（同一SSID内でオンボードとアクセスを完結）または **Dual SSID**（オンボード専用SSIDから開始）。 |
| **証明書プロビジョニング** | **SCEP**（Simple Certificate Enrollment Protocol）または **ISE内部CA** を利用してデバイス証明書を発行。 |
| **プロトコル** | **HTTPS**（ポータル・プロビジョニング）、**EAP-TLS**（オンボード後の接続）。 |
| **エージェント** | **Network Setup Assistant (NSP)**。Android/Windows/macOSで設定を自動化するツール。 |
| **ポータル** | **BYOD Portal**。ユーザーが規約に同意し、デバイスを登録するためのWeb画面。 |
| **メリット** | IT部門の手を介さず、ユーザーがセルフサービスでセキュアな接続を完了できる。 |

---

## 🏗 動作原理

BYODプロセスは、**「未登録デバイスの識別」→「ポータルリダイレクト」→「プロビジョニング」→「再接続」** というフローで動作します。

```
Client (Unregistered)
   ↓
NAD (WLC/Switch) --(RADIUS/MAB)--> ISE (Initial Auth)
   ↓
ISE --(Authorization Result: Redirect URL)--> NAD
   ↓
Client Browser --(HTTP Redirect)--> ISE BYOD Portal
   ↓
NSP Download & Cert Provisioning (Internal CA / SCEP)
   ↓
Client (Registered with Cert)
   ↓
NAD (WLC/Switch) --(802.1X EAP-TLS)--> ISE (Final Auth)
   ↓
Full Network Access
```

---

## ⚙ 動作シーケンス (Dual-SSID Flow)

1.  **初期接続**: クライアントがオープンまたはPSKの「Provisioning SSID」に接続します。
2.  **Webリダイレクト**: ブラウザを開くとISEのBYODポータルにリダイレクトされます。
3.  **ユーザー認証**: ユーザー名とパスワード（AD等）を入力します。
4.  **デバイス登録**: デバイスがISEのデータベース（Endpoints）に登録されます。
5.  **プロビジョニング**: デバイスにNetwork Setup Assistantがダウンロードされ、証明書署名要求(CSR)がISEに送られます。
6.  **証明書発行**: ISE CA（または外部CA）が証明書を発行し、Wi-Fiプロファイルと共にデバイスへインストールします。
7.  **再接続**: デバイスは自動的に「Corporate SSID」へ切り替え、EAP-TLSによる強力な認証でアクセスします。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **リダイレクトACLの設定**: NAD（WLC等）にリダイレクト用ACLを事前定義し、ISEの認可プロファイルでその名前を正確に指定する必要があります。
*   **証明書認証プロファイルの構成**: EAP-TLS認証時に、どのフィールド（SAN, CN等）をユーザーIDとして使用するかを定義します。
*   **Client Provisioning Policy**: オペレーティングシステム（iOS, Android, Windows等）ごとに、どのプロビジョニング・リソース（NSPツールやWi-Fiプロファイル）を適用するかを定義します。
*   **ISE CAの有効化**: オンボーディングにISE内部CAを使用する場合、Certificate Servicesが有効であることを確認します。
*   **Androidの特殊性**: Android 10以降のMACアドレスランダム化設定がオンボーディングに与える影響や、NSPのダウンロード手順について理解が必要です。
*   **Native Supplicant Provisioning (NSP)**: iOSやmacOSはOS標準機能でプロビジョニングが可能ですが、Windows等はNSPアプリが必要になります。

---

## 🛠 設定方法

### 1. WLC：BYODリダイレクトACLの構成 (CLI)
```bash
config access-list create ACL_BYOD_REDIRECT
! DNS/DHCP/ISE通信を許可（リダイレクト除外）
config access-list add-rule ACL_BYOD_REDIRECT 1 0.0.0.0 0.0.0.0 53 0.0.0.0 0.0.0.0 any permit
config access-list add-rule ACL_BYOD_REDIRECT 2 0.0.0.0 0.0.0.0 any 10.1.1.100 255.255.255.255 any permit
! その他（HTTP/HTTPS）を拒否（＝リダイレクト対象にする）
config access-list add-rule ACL_BYOD_REDIRECT 3 0.0.0.0 0.0.0.0 any 0.0.0.0 0.0.0.0 any deny
```

### 2. ISE：BYOD認可プロファイルの設定 (GUI)
1.  **Policy > Policy Elements > Results > Authorization > Authorization Profiles** で新規作成。
2.  **Web Redirection** をチェック。
3.  タイプとして **Hotspot/Native Supplicant Provisioning** を選択。
4.  ACL名に `ACL_BYOD_REDIRECT` を指定し、ValueにBYODポータルを選択。

---

## 🔍 検証コマンド

| 目的 | コマンド / 手法 |
| :--- | :--- |
| **ISE 認証ライブログの確認** | **Operations > RADIUS > Live Logs** |
| **デバイス登録状態の確認** | **Context Visibility > Endpoints** で BYOD 登録済みか確認。 |
| **発行済み証明書の確認** | **Administration > System > Certificates > ISE Root CA > Issued Certificates** |
| **WLC クライアント状態の確認** | <code>show client details [MAC]</code> で Redirect URL の適用を確認。 |
| **NAD の RADIUS 統計確認** | <code>show radius statistics</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| リダイレクトがループする | 認可ポリシーで「登録済み」条件が不足 | `EndPoints:BYODRegistration = Unknown` の場合にのみリダイレクトするよう設定を確認。 |
| 証明書発行に失敗する | SCEPの設定ミスまたはCAサービス停止 | **Administration > Certificates > SCEP Profiles** のステータスを確認。 |
| AndroidでNSPがダウンロード不可 | インターネット接続がない（Play Storeに未到達） | リダイレクトACLでGoogle Playへのアクセスを許可するか、ISEから直接NSPを配布する構成にする。 |
| EAP-TLS 認証に失敗する | 証明書の信頼チェーンが不完全 | クライアントとISEの両方に、ルートCA証明書が正しくインストールされているか確認。 |

---

## ⚠ 制限事項

*   **ブラウザの互換性**: 一部のモバイルブラウザではリダイレクトやプロファイルのインストールが正しく動作しない場合があります（標準ブラウザ推奨）。
*   **ライセンス**: BYOD機能の利用には、ISEの **Advantage** または **Premier** ライセンスが必要です。
*   **プライバシー設定**: 最新のOSではMACアドレスをランダム化するため、ISEが同一デバイスとして認識できないケースに注意が必要です。

---

## 🔄 他技術との関連

*   **2.6 Microsegmentation (TrustSec)**: BYODデバイスにSGT（例：SGT 15 Personal）を割り当てて制御します。
*   **4.1 ISE Scalability**: 大規模BYOD環境では、プロビジョニングトラフィックを処理するためにPSNの分散配置が重要です。
*   **4.5 Guest Lifecycle**: ゲストアクセスとBYODポータルは同じインフラ（CWA）を共有しますが、フローは異なります。

---

## 🧩 比較表

### Single SSID vs Dual SSID BYOD

| 特徴 | Single SSID | Dual SSID |
| :--- | :--- | :--- |
| **SSID数** | 1 (Corporate SSID のみ) | 2 (Open/Provisioning + Corporate) |
| **ユーザー体験** | SSIDの切り替えが不要でスムーズ。 | オンボード専用SSIDに繋ぐ必要があるため明確。 |
| **初期接続の難易度** | 最初から802.1X（PEAP等）で繋ぐ必要があり複雑。 | 最初はOpen/PSKで繋ぐため簡単。 |
| **セキュリティ** | 最初から暗号化される。 | 初期接続時（オンボード前）は暗号化されない。 |

---

## 💡 ベストプラクティス

1.  **ISE 内部 CA の活用**: 外部 PKI との連携が複雑な場合、ISE を Root CA として構成し、管理を簡素化します。
2.  **ポータルカスタマイズ**: 言語設定や利用規約（AUP）を自社のポリシーに合わせてカスタマイズし、ユーザーの混乱を防ぎます。
3.  **証明書の有効期限**: BYOD証明書の有効期限は通常1〜2年に設定し、デバイスの更新サイクルに合わせます。
4.  **My Devices Portal の提供**: ユーザーが自分で紛失デバイスを報告したり、登録を削除したりできるポータルを提供し、管理負荷を軽減します。

---

## 📝 ラボ学習・設定サンプル例

### 1. BYOD 認可ポリシー（初期）
*   **要件**: 未登録デバイス（iOS/Android）が接続したら BYOD ポータルへリダイレクトせよ。
*   **条件**: `IdentityGroup: Any` AND `Endpoints:BYODRegistration EQUALS Unknown`
*   **結果**: `Authorization Profile: BYOD-Redirect`

### 2. BYOD 認可ポリシー（完了後）
*   **要件**: 登録済みデバイスが EAP-TLS で接続したらフルアクセスを許可せよ。
*   **条件**: `Network Access:EapAuthentication EQUALS EAP-TLS` AND `Endpoints:BYODRegistration EQUALS Registered`
*   **結果**: `PermitAccess` + `SGT: Personal_Device`

### 3. Wi-Fi プロファイルの構成 (ISE)
*   **要件**: iOS デバイス向けに、SSID "Corporate"、セキュリティ "WPA2 Enterprise"、EAPタイプ "TLS" の設定を配布せよ。

### 4. SCEP プロファイルの作成
*   **操作**: 外部 CA (Microsoft CA等) から証明書を取得するための SCEP URL とチャレンジパスワードを ISE に構成。

### 5. リダイレクト ACL への外部ドメイン許可
*   **要件**: Android のオンボーディングを円滑にするため、Google への OCSP/CRL チェックを許可せよ。

### 6. Client Provisioning リソースのアップロード
*   **操作**: ISE に最新の Windows 版 Network Setup Assistant (`.exe`) をアップロードする。

### 7. デバイスタイプに基づくポリシー分離
*   **要件**: iPad は "Tablets" VLAN、Windows は "BYOD-Laptops" VLAN へ動的に割り当てよ。

### 8. ポスチャ(検疫)との組み合わせ
*   **要件**: BYOD オンボーディング直後に、エージェントレス・ポスチャを実行し、OSパッチ状況を確認せよ。

### 9. 証明書テンプレートの SAN 指定
*   **要件**: 発行される証明書の Subject Alternative Name (SAN) にユーザー名を含めるように構成せよ。

### 10. 自己署名証明書の警告回避
*   **問題**: リダイレクトポータルで HTTPS 警告が出る。
*   **対処**: ISE の Admin/Portal 証明書を全デバイスが信頼する Root CA で署名する。

---

## ❓ 想定試験問題

1.  **Design**: Dual-SSID フローにおいて、初期接続 SSID でリダイレクトが成功した後、最終的にクライアントが接続すべき EAP メソッドは？
    *   **回答**: **EAP-TLS**（プロビジョニングされた証明書を使用するため）。
2.  **トラブルシュート**: クライアントに証明書はインストールされたが、再接続時に認証エラーが出る。ISE ライブログで `12514 EAP-TLS failed` とある。原因は？
    *   **回答**: ISE にクライアントの証明書を発行した **Root CA の証明書（Trusted Certificate）が登録されていない** 可能性が高い。
3.  **コンフィグ読解**: 認可ポリシーで `EndPoints:LogicalProfile EQUALS Android` という条件がある。これが機能するための前提条件は？
    *   **回答**: **プロファイリング（Profiling）**が有効であり、HTTP User-Agent や DHCP 属性でデバイスが識別されていること。
4.  **実装**: iOS デバイスでプロビジョニングを開始するために、ユーザーがブラウザで最初に行うべきアクションは？
    *   **回答**: Webブラウザを開き、任意のページにアクセスしてリダイレクトをトリガーするか、指定のポータルURLを直接入力する。
5.  **Design**: Single-SSID フローを採用する最大のメリットは？
    *   **回答**: オンボーディング前と後で **SSID を切り替える必要がなく**、ユーザーにとっての利便性が高いこと。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [BYODプロセスの設定](https://www.cisco.com/c/en/us/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31/b_ise_admin_guide_31_chapter_010111.html)
*   **Cisco Live (BRKSEC-2041)**: [ISE BYOD: Design, Implementation and Troubleshooting](https://www.ciscolive.com/)
*   **Configuration Example**: [ISE Single SSID BYOD for Windows and Apple iOS](https://www.cisco.com/c/en/us/support/docs/security-software/identity-services-engine/116217-configure-ise-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: BYODは「信頼の構築」です。最初はユーザー名/パスワードで「人」を信じ、次に証明書を配ることで「デバイス」を信じるという2段階の信頼構築プロセスであると理解しましょう。
*   **図解**: 
    - **Step 1 (Human Auth)**: Who are you? (AD Credentials)
    - **Step 2 (Provisioning)**: Here is your key. (Certificate)
    - **Step 3 (Device Auth)**: Show me the key. (EAP-TLS)
*   **注意点**: ラボ試験では、**SCEP プロファイル** や **Certificate Authority** の設定が一部不完全な状態で提供されることがあるため、ログを確認して「どこで止まっているか（認証か、証明書発行か、接続か）」を切り分けるのがコツです。
