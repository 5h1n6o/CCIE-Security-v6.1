---
layout: default
title: 4.13.a-TEAP
nav_order: 1
parent: 4.13-Auth-methods
grand_parent: 4.0-Identity-Management
---

# 4.13.a EAP Chaining and TEAP

**TEAP (Tunnel Extensible Authentication Protocol)** は RFC 7170 で定義された比較的新しい EAP メソッドであり、Cisco ISE 環境において **EAP Chaining** を実現するための標準プロトコルです。これは、1つの RADIUS セッション内で「マシン認証」と「ユーザー認証」を同時に行い、両方が成功したことを条件に特定のアクセス権限（VLAN や SGT）を付与することを可能にします。

---

## 📘 概要

*   **機能概要**: TLS トンネルを確立し、その内部で複数の EAP メソッドを順次実行する（Chaining）機能です。
*   **利用目的**: 「会社支給の PC (Machine)」かつ「正当な社員 (User)」の両方のアイデンティティを確認し、セキュリティレベルを最大化すること。
*   **どのような場面で利用するか**:
    *   Windows 10/11 のネイティブサプリカントを使用した、単一 SSID/ポートでの高度な認証。
    *   AnyConnect NAM を使用した有線・無線のセキュアアクセス。
    *   「マシン認証のみ」「ユーザー認証のみ」では不十分な、厳格なコンプライアンス要件がある環境。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主なプロトコル** | TEAP (RFC 7170)。 |
| **トンネル方式** | TLS ベースの外側トンネル (Outer Tunnel) を使用。 |
| **EAP Chaining** | マシンとユーザーの両方の資格情報を ISE に一度に提示する機能。 |
| **内部メソッド** | EAP-MSCHAPv2, EAP-TLS など。 |
| **ISE 条件** | `Network Access:EapChainingResult` 属性で認可を制御。 |
| **メリット** | EAP-FAST に代わる標準プロトコル。単一セッションで完了するため CoA 負荷が低い。 |
| **要件** | Windows 10 (2004以降) または AnyConnect 4.8 以降、Cisco ISE 2.7 以降。 |

---

## 🏗 動作原理

TEAP はトンネル内に **TLV (Type-Length-Value)** オブジェクトを流すことで、複数の認証結果をカプセル化します。

```text
Supplicant (PC)           Authenticator (Switch)           ISE (Server)
      |                          |                               |
      |------ TEAP Start ------->|                               |
      |                          |------- RADIUS Request ------->|
      |                          |        (TEAP Start)           |
      |<--- TLS Handshake (Outer Tunnel Establishment) --------->|
      |                          |                               |
      | [ Inside Tunnel ]        |                               |
      | <--- (1) EAP-TLS (Machine Auth) -----------------------> |
      | <--- (2) EAP-MSCHAPv2 (User Auth) ---------------------> |
      | <--- (3) Crypto Binding TLV (Integrity Check) ---------> |
      |                          |                               |
      |<----- EAP-Success -------|                               |
      |                          |<------ Access-Accept ---------|
                                          (VLAN/SGT/dACL)
```

---

## ⚙ 動作シーケンス

1.  **Outer Tunnel 確立**: クライアントと ISE 間で TLS 接続を行います。ISE はサーバー証明書を提示します。
2.  **第1認証 (Machine)**: トンネル内でマシン認証（証明書やパスワード）を実行します。
3.  **第2認証 (User)**: 同じトンネル内でユーザー認証（MS-CHAPv2 や証明書）を実行します。
4.  **Crypto Binding**: マシン認証とユーザー認証が同一のエンティティ（同一 PC 上）で行われたことを証明するバインディング処理を行います。
5.  **認可判定**: ISE は「両方成功 (Both)」「マシンのみ (Machine Only)」「ユーザーのみ (User Only)」のステータスに基づき、最終的なポリシーを適用します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Allowed Protocols の構成**: ISE の `Allowed Protocols` で TEAP を有効化し、`Enable EAP Chaining` にチェックが入っていることを確認してください。
*   **ISE 認可条件**: `Network Access:EapChainingResult EQUALS Both Succeed` という条件を使いこなせる必要があります。
*   **Crypto Binding の強制**: ラボの要件でセキュリティ強化が求められた場合、ISE 側で Crypto Binding を `Required` に設定します。
*   **Windows 側の構成**: Windows ネイティブサプリカントで TEAP を選択し、適切な「内側メソッド」と「外側 TLS 設定」を行う手順を習得してください。
*   **証明書認証プロファイル (CAP)**: ユーザーとマシンで異なる CAP を使用する場合、どのように ID を抽出するかを定義します。

---

## 🛠 設定方法

### 1. Cisco ISE：TEAP プロトコルの有効化
1.  **Policy > Policy Elements > Results > Authentication > Allowed Protocols** に移動。
2.  **TEAP** をチェック。
3.  **Inner Methods** で `EAP-MSCHAPv2` や `EAP-TLS` を選択。
4.  **Enable EAP Chaining** を有効化。

### 2. Cisco ISE：認可ポリシーの作成
*   **Rule (Full Access)**:
    *   Condition: `Network Access:EapChainingResult EQUALS Both Succeed`
    *   Result: `PermitAccess` + `SGT:Employee_Full`
*   **Rule (Machine Access)**:
    *   Condition: `Network Access:EapChainingResult EQUALS Machine Succeed`
    *   Result: `VLAN_Restricted`（ユーザーログイン前や、私物デバイス等の制御）

---

## 🔍 検証コマンド

| 目的 | コマンド / 手法 |
| :--- | :--- |
| **ISE 認証詳細の確認** | **Operations > RADIUS > Live Logs** |
| **Chaining 結果の確認** | Live Logs の詳細画面で `Network Access:EapChainingResult` を確認。 |
| **スイッチ側のセッション確認** | <code>show access-session interface [int] details</code> |
| **TEAP 内部パケットのデバッグ** | <code>debug runtime-AAA</code> (ISE CLI) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| **TLS トンネル失敗** | クライアントが ISE の証明書を信頼していない | クライアントに ISE のルート CA をインポートする。 |
| **User Only になる** | マシン認証がスキップされている | Windows 設定の「マシン認証を使用する」が有効か確認。 |
| **Crypto Binding 失敗** | クライアント側の実装が古い | クライアント OS のパッチ適用または AnyConnect バージョンアップを確認。 |
| **タイムアウト** | プロトコルネゴシエーションの不一致 | `Allowed Protocols` で他の EAP（PEAP等）より TEAP を上位にする。 |

---

## ⚠ 制限事項

*   **OS バージョン依存**: 非常に古い Windows 10 では TEAP がネイティブサポートされていません。
*   **サードパーティサプリカント**: TEAP は標準化されていますが、実装（スマホ OS 等）によっては EAP Chaining が利用できない場合があります。
*   **証明書のサイズ**: TEAP 内部で EAP-TLS を 2 回行う場合、証明書チェーンが長いと RADIUS パケットのフラグメンテーションが発生しやすくなります。

---

## 🔄 他技術との関連

*   **4.2 Network Access AAA**: TEAP は RADIUS のペイロードとして運ばれます。
*   **2.6 TrustSec (SGT)**: Chaining の結果（Both/Machine/User）に応じて、異なる SGT を割り当てます。
*   **AnyConnect (NAM)**: AnyConnect を使用すると、Windows 標準機能よりも安定した TEAP 接続が可能です。

---

## 🧩 比較表

### TEAP vs EAP-FAST

| 特徴 | TEAP (RFC 7170) | EAP-FAST |
| :--- | :--- | :--- |
| **標準化** | **IETF 標準** | Cisco 独自（のちに RFC 化） |
| **EAP Chaining** | **公式サポート** | Cisco 拡張としてサポート |
| **主要用途** | モダンな Windows 環境 | 旧来の AnyConnect 環境 |
| **トンネル保護** | TLS 1.2/1.3 | TLS または PAC (Protected Access Credential) |

---

## 💡 ベストプラクティス

1.  **段階的な移行**: PEAP から TEAP に移行する場合、`Allowed Protocols` で両方を許可し、準備ができた端末から GPO で TEAP をプッシュします。
2.  **証明書の併用**: セキュリティを最大化するため、マシン認証とユーザー認証の両方に **EAP-TLS** (Certificate) を内側メソッドとして使用します。
3.  **Crypto Binding の有効化**: 認証情報のすり替え攻撃（MiM）を防ぐため、Crypto Binding は常に有効化することを推奨します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な TEAP 有効化 (ISE)
*   **要件**: Allowed Protocols 名 "Lab_TEAP" を作成し、MSCHAPv2 を内部メソッドとして Chain を許可せよ。

### 2. 両認証成功時の認可ルール
*   **要件**: マシンとユーザーの両方が認証された場合のみ、VLAN 10 を付与せよ。

### 3. マシン認証のみの許可 (Start-up)
*   **要件**: ユーザーがログインする前（マシン認証のみ）は、AD 通信のみを許可する限定 ACL を適用せよ。

### 4. 内部メソッド EAP-TLS の構成
*   **要件**: マシン認証には証明書（EAP-TLS）、ユーザー認証には ID/PASS（MSCHAPv2）を使用するように構成せよ。

### 5. Windows ネイティブ設定
*   **操作**: Windows 10 の「ネットワークと共有センター」で、802.1X の種類に TEAP を手動構成せよ。

### 6. Crypto Binding の強制
*   **要件**: Binding に失敗したセッションを ISE で Reject せよ。

### 7. 名前空間の解決
*   **要件**: 内部メソッドで EAP-TLS を使う際、Certificate Authentication Profile で Subject-CN から ID を抽出せよ。

### 8. AnyConnect NAM での TEAP
*   **要件**: AnyConnect NAM プロファイルを作成し、優先認証プロトコルを TEAP に設定せよ。

### 9. フォールバック認可
*   **要件**: TEAP に対応していないデバイス（MAB 等）は自動的に Guest VLAN へ落とせ。

### 10. 複数ドメイン環境での Chain
*   **要件**: マシンは Domain A、ユーザーは Domain B に所属している場合に認証を成功させよ。

---

## ❓ 想定試験問題

1.  **Design**: TEAP において EAP Chaining を使用する最大のメリットは？
    *   **回答**: **単一の RADIUS セッション**でマシンとユーザーの両方を認証し、それらの組み合わせ（Both Succeed 等）を認可ポリシーの条件として直接使用できる点。
2.  **トラブルシュート**: TEAP 接続時に ISE で `22037 Cryptobinding TLV mismatch` が発生している。何が原因か？
    *   **回答**: クライアントが提示したバインディング情報が、内側メソッドの認証結果と一致していない。クライアントのサプリカント設定やバージョンを確認する必要がある。
3.  **コンフィグ読解**: ISE 認可条件で `TEAP:InnerMethod EQUALS EAP-TLS` とある。これは何をチェックしているか？
    *   **回答**: TEAP トンネルの中で実行された **具体的な認証プロトコル** を確認している。
4.  **Design**: PEAP と比較して TEAP が優れている点は？
    *   **回答**: PEAP は単一の認証しかトンネル内でサポートしないが、TEAP は **複数の認証 (Chaining)** を標準でサポートしている。
5.  **実装**: Windows 10 端末で TEAP を構成する際、信頼されたルート証明機関の指定を誤るとどうなるか？
    *   **回答**: トンネル確立時の **TLS ハンドシェイクに失敗**し、認証プロセスが開始されない。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [TEAP の設定と EAP Chaining](https://www.cisco.com/c/ja_jp/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31.html)
*   **Cisco Live (BRKSEC-2022)**: [Advanced EAP Methods and EAP Chaining](https://www.ciscolive.com/)
*   **RFC 7170**: [Tunnel Extensible Authentication Protocol (TEAP)](https://datatracker.ietf.org/doc/html/rfc7170)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「TEAP = カプセル」と覚えてください。その中に「マシン認証」と「ユーザー認証」という2つの種が入っており、両方が芽を出した時だけ「フルアクセス」という花が咲きます。
*   **注意点**: ラボ試験では、**Windows の有線プロパティで TEAP が選択肢に出てこない**（古いビルド）場合があるため、AnyConnect NAM が提供されている場合はそちらでの設定を優先的に検討してください。
