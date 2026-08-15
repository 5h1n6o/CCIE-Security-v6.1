---
layout: default
title: 4.3-Admin-access
nav_order: 3
parent: 4.0-Identity-Management
---

# 4.3 Cisco devices for administrative access with Cisco ISE

Cisco ISE (Identity Services Engine) を使用した**デバイス管理アクセス（Device Administration）**は、ネットワーク機器（ルータ、スイッチ、Firewall）への管理者ログインを一元化し、誰が、いつ、どのコマンドを実行できるかを厳密に制御する仕組みです。主に **TACACS+** プロトコルを使用して実装され、CCIE Security ラボ試験では、RBAC（ロールベースアクセス制御）に基づいた詳細な認可ポリシーの設定が求められます。

---

## 📘 概要

*   **機能概要**: ネットワークデバイスを RADIUS または TACACS+ クライアント（NAD）として構成し、ISE を認証・認可サーバとして利用することで、管理者の認証、権限割当、操作ログの記録（AAA）を行います。
*   **利用目的**: 機器ごとにローカルパスワードを管理する手間を省き、セキュリティガバナンス（「誰が何をしたか」の監査）を強化します。
*   **場面**: 数十台〜数百台のルータやスイッチの特権モードアクセス制御、特定のオペレータには `show` コマンドのみ許可するなどの制限が必要な場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要プロトコル** | **TACACS+** (TCP 49) が推奨（RADIUS よりも認可制御が細かいため）。 |
| **ISE ライセンス** | Device Administration（旧称：TACACS+）ライセンスが必要。 |
| **ISE Persona** | PSN ノードで **"Device Administration Service"** を明示的に有効化する必要がある。 |
| **認可要素** | **Shell Profile** (特権レベル) と **Command Set** (許可コマンド) の組み合わせ。 |
| **冗長性** | ISE ノードを複数登録し、タイムアウト時のローカルフォールバックを構成する。 |
| **属性** | Privilege Level 0-15、Custom Attributes。 |

---

## 🏗 動作原理

管理者アクセスの AAA フローは、機器（NAD）がゲートウェイとなり、ISE がポリシーを決定します。

```text
Admin (User)
   ↓ (1) SSH/Telnet Login Attempt
Cisco Device (NAD)
   ↓ (2) TACACS+ Authentication Request
Cisco ISE (Device Admin Service)
   ↓ (3) Verify User (Internal/AD/LDAP)
   ↑ (4) TACACS+ Authentication Response (PASS)
Cisco Device (NAD)
   ↓ (5) TACACS+ Authorization Request (Priv level / Shell)
Cisco ISE
   ↑ (6) TACACS+ Authorization Response (Shell Profile / Command Sets)
Cisco Device (NAD)
   ↓ (7) Execute Command / Accounting
Cisco ISE
```

---

## ⚙ 動作シーケンス

1.  **Authentication**: 管理者が SSH でログインを試みると、NAD はユーザー名/パスワードを ISE に送信します。
2.  **Authorization (Exec)**: 認証成功後、ISE はそのユーザーに許可された **Privilege Level** (0-15) を NAD に通知します（Shell Profile）。
3.  **Authorization (Commands)**: 管理者がコマンド（例：`configure terminal`）を打つたびに、NAD は ISE に実行の可否を問い合わせます（Command Set）。
4.  **Accounting**: 実行されたすべてのコマンドとログアウト情報は、監査ログとして ISE の MnT ノードに記録されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **ISE Persona の有効化**: ラボ試験の初期設定で、ISE ノードの **Edit > Device Administration Service** のチェックが入っているか必ず確認してください。これが OFF だと TACACS+ ポリシーセット自体が表示されません。
*   **Fallback 構成**: 「ISE がダウンした場合でもローカルユーザーでログイン可能にせよ」という要件が多いため、`aaa authentication login default group tacacs+ local` のような構成が必須です。
*   **特権レベルの自動付与**: ログイン直後に `enable` コマンドなしで特権モード（Level 15）に入るには、Shell Profile で `Default Privilege = 15` を設定します。
*   **Command Set の順序**: コマンドの許可・拒否は「上から順」に照合されます。最後に `Permit any command that is not listed below` のチェックを入れるかどうかの判断を要件に従って行います。
*   **ASA の TACACS+ 設定**: IOS とは構文が異なる（`aaa-server` グループを使用）ため、ASA 独自のコマンドを習得する必要があります。

---

## 🛠 設定方法

### 1. Cisco IOS-XE 側の AAA 構成 (TACACS+)
```bash
! ISE サーバの定義
tacacs server ISE_SERVER
 address ipv4 10.1.1.100
 key cisco123
!
! AAA グループとメソッドリスト
aaa new-model
aaa group server tacacs+ ISE_GROUP
 server name ISE_SERVER
!
! 認証・認可・アカウンティングの設定
aaa authentication login VTY_AUTH group ISE_GROUP local
aaa authorization exec VTY_AUTH group ISE_GROUP local 
aaa authorization commands 15 VTY_AUTH group ISE_GROUP local 
aaa accounting exec VTY_AUTH start-stop group ISE_GROUP
aaa accounting commands 15 VTY_AUTH start-stop group ISE_GROUP
!
! VTYラインへの適用
line vty 0 4
 login authentication VTY_AUTH
 authorization exec VTY_AUTH
 authorization commands 15 VTY_AUTH
```

### 2. Cisco ISE 側の構成手順
1.  **Administration > System > Deployment**: PSN ノードで `Device Administration Service` を有効化。
2.  **Work Centers > Device Admin > Network Resources**: デバイス（NAD）を IP アドレスと共有キーで登録。
3.  **Work Centers > Device Admin > Policy Results**:
    *   **Shell Profiles**: `Default Privilege = 15` などを定義。
    *   **Command Sets**: 許可するコマンド（例：`show.*`）を正規表現等で定義。
4.  **Work Centers > Device Admin > Device Admin Policy Sets**:
    *   Identity Policy でユーザーソース（AD 等）を指定。
    *   Authorization Policy で「特定のユーザーグループ ＋ 特定のデバイスグループ」に対して作成した Shell Profile と Command Set を紐付け。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **TACACS+ サーバの状態確認** | <code>show tacacs</code> |
| **AAA メソッドリストの適用確認** | <code>show aaa method-lists authentication</code> |
| **ログイン中のユーザー権限確認** | <code>show privilege</code> |
| **ISE 側のライブログ確認** | **Operations > TACACS > Live Logs** |
| **パケットレベルのデバッグ** | <code>debug tacacs</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 認証がタイムアウトする | 共有キーの不一致、または IP 疎通不可 | <code>ping</code> で疎通を確認し、<code>key</code> コマンドを再投入する。 |
| ログインできるが特権モードになれない | Shell Profile の Privilege 設定ミス | ISE 認可ポリシーで正しい Shell Profile が適用されているか確認。 |
| コマンドがすべて拒否される | Command Set での許可設定不足 | `Permit any commands` にチェックが入っているか、または正規表現が正しいか確認。 |
| "Connection Refused" | ISE 側で TACACS+ サービスが停止 | ISE CLI で <code>show application status ise</code> を実行しプロセスを確認。 |

---

## ⚠ 制限事項

*   **RADIUS との混在**: 管理アクセスに RADIUS を使うことも可能ですが、コマンドごとの認可制御ができない（ログインレベルの制御のみ）ため、CCIE 試験では TACACS+ が基本です。
*   **通信の暗号化**: RADIUS はパスワードのみ暗号化しますが、TACACS+ はパケット全体を暗号化します。

---

## 🔄 他技術との関連

*   **4.1 ISE Scalability**: 大規模環境では管理アクセスリクエストを複数の PSN で負荷分散します。
*   **3.1 Device Hardening**: デバイス自体のセキュリティ（iACL や CoPP）で、管理アクセス（TCP 49）を許可する必要があります。
*   **3.6.c SYSLOG**: AAA アカウンティングログは ISE だけでなく、外部 Syslog サーバへも送信することが一般的です。

---

## 🧩 比較表

### TACACS+ vs RADIUS (Administrative Access)

| 特徴 | TACACS+ | RADIUS |
| :--- | :--- | :--- |
| **トランスポート** | TCP 49 (高信頼) | UDP 1812/1813 |
| **暗号化** | ペイロード全体 | パスワードのみ |
| **認可制御** | **コマンドレベルで可能** | ログイン/特権レベルのみ |
| **標準** | Cisco 独自（現在は公開） | IETF 標準 (RFC) |

---

## 💡 ベストプラクティス

1.  **Local Fallback**: 常に `local` 認証をバックアップとして含め、ISE 不達時のロックアウトを防止します。
2.  **Separate Groups**: 管理者の役割（ネットワーク管理者 vs オペレーター）ごとに ISE 側で Command Set を分離します。
3.  **Accounting**: `start-stop` を使用して、操作の開始と終了の両方を記録し、監査証跡を完全に残します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な TACACS+ 認証の構成
*   **要件**: ユーザー `admin` が ISE (10.1.1.100) 経由でログインできるようにせよ。
*   **設定**: `aaa authentication login default group tacacs+`。

### 2. 特権レベル 15 の自動割り当て
*   **要件**: ログイン後、`enable` を打たずに特権モードで開始せよ。
*   **設定**: ISE Shell Profile で `Default Privilege = 15` を設定。

### 3. オペレーター用のコマンド制限
*   **要件**: ユーザー `operator` に `show` コマンドのみを許可せよ。
*   **設定**: ISE Command Set で `permit show` を設定し、それ以外を `deny`。

### 4. 特定の IP アドレスからのアクセス制限
*   **要件**: VTY アクセスを管理セグメント (10.1.5.0/24) のみに限定し、かつ AAA を使用せよ。

### 5. ASA での SSH 管理者認証
*   **要件**: ASA (Outside 経由) の CLI ログインに ISE を使用せよ。
*   **設定**: `aaa-server TACACS_GRP (outside) host 10.1.1.100`。

### 6. ASDM 認証と認可 (ASA)
*   **要件**: ASDM ログインに ISE を使用し、特権 15 を付与せよ。
*   **設定**: `aaa authentication http console TACACS_GRP`。

### 7. AAA アカウンティングの有効化
*   **要件**: 実行されたすべての `conf t` 以下のコマンドを ISE に記録せよ。
*   **設定**: `aaa accounting commands 15 default start-stop group tacacs+`。

### 8. VTY ラインの個別制限
*   **要件**: Line VTY 0-4 は ISE、5-15 はローカルのみを使用するように分離せよ。

### 9. IPv6 環境での TACACS+ 構成
*   **要件**: IPv6 アドレスを持つ ISE ノードに対して AAA を構成せよ。

### 10. カスタム属性 (VSA) の利用
*   **要件**: ISE から NAD へ、独自に定義した属性を送信して制御せよ。

---

## ❓ 想定試験問題

1.  **トラブルシュート**: `aaa authorization exec default group tacacs+` を設定したが、ログイン直後にタイムアウトで切断される。ISE 側の何を確認すべきか？
    *   **回答**: ISE の認証ポリシーではなく、**認可ポリシー（Authorization Policy）**において、**Shell Profile** が正しく割り当てられていない可能性がある。
2.  **Design**: 管理アクセス制御において、RADIUS よりも TACACS+ が優れている最大の技術的理由は？
    *   **回答**: TACACS+ は**認証と認可を分離**しており、ユーザーがコマンドを打つたびに個別に許可/拒否を制御できるため。
3.  **コンフィグ読解**: `aaa authentication login default group tacacs+ local` の `local` が持つ意味を説明せよ。
    *   **回答**: ISE サーバ（tacacs+）が応答しない場合のみ、機器内のローカルデータベースを使用して認証を行うフォールバック設定。
4.  **実装**: ISE 3.x で Device Administration 機能を有効にする場所は？
    *   **回答**: **Administration > System > Deployment** の各 PSN ノード設定内。
5.  **Design**: 特定のジュニアエンジニアに、インターフェイスの `shutdown` と `no shutdown` だけを許可したい。どのように実装すべきか？
    *   **回答**: ISE で `Command Set` を作成し、`interface` および `shutdown`, `no shutdown` コマンドを `Permit` し、認可ポリシーでそのユーザーに紐付ける。

---

## 🔗 参考リソース

*   **Cisco Configuration Guide**: [Cisco ISE 3.1 Device Administration (TACACS+)](https://www.cisco.com/c/en/us/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31/b_ise_admin_guide_31_chapter_010011.html)
*   **Cisco Live (BRKSEC-2003)**: [Advanced Device Administration with Cisco ISE](https://www.ciscolive.com/)
*   **Technical Note**: [TACACS+ Configuration on Cisco IOS-XE](https://www.cisco.com/c/en/us/support/docs/security- Rachael/identity-services-engine/119143-configure-ise-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「AAA は 3 つの A（認証・認可・記録）の歯車」であることを意識してください。
*   **注意点**: ラボ試験の VTY 設定では、`transport input ssh` を入れ忘れると AAA が正しくても SSH 自体が繋がらないというイージーミスに注意してください。
*   **図解**: ISE の Live Logs はデバッグの生命線です。赤い失敗ログが出たら、`Details` を開き、どのポリシー（Authentication or Authorization）で落ちているかを確認するのが鉄則です。
