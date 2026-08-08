---
layout: default
title: 3.2.c-Securing-device-access
nav_order: 3
parent: 3.2-Management-plane
grand_parent: 3.0-Security-Infrastructure
---

# 3.2.c Securing device access

Ciscoデバイスへのアクセスを保護することは、マネジメントプレーン・セキュリティの最終防衛線です。管理者によるデバイスの構成、監視、および制御（CLIまたはGUI経由）を、許可されたユーザーおよび正当なプロトコルのみに制限することを目的とします。CCIE Securityラボ試験では、TelnetやHTTPなどの脆弱なプロトコルの無効化、SSH v2の設定、特権レベル（Privilege levels）の管理、およびAAA（認証・承認・検課）の統合が頻繁に問われます。

---

## 📘 概要

*   **機能概要**: デバイスへのリモートアクセス（VTY, HTTP）およびローカルアクセス（Console, Aux）を制御・保護する一連の技術です。
*   **利用目的**: 不正アクセス、パスワードの盗聴、および権限昇格攻撃の防止。
*   **どのような場面で利用するか**: ネットワーク内の全インフラデバイス（Router, Switch, ASA, FTD, ISE等）の初期ハードニングおよび定常運用において必須となります。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **保護プロトコル** | SSH (Secure Shell), HTTPS (TLSベース)。 |
| **無効化対象** | Telnet, HTTP (クリアテキスト)。 |
| **アクセス制限** | ACLを使用したVTY行へのアクセス制限 (`access-class`)。 |
| **認証方式** | Local database, AAA (TACACS+, RADIUS)。 |
| **セッション管理** | `exec-timeout`, `login block-for` (ブルートフォース対策)。 |
| **バナー** | 侵入者への警告（`banner motd`, `banner exec`）。 |
| **特権管理** | 特権レベル (0-15) または Role-Based CLI (Parser views)。 |

---

## 🏗 動作原理

デバイスへのアクセス試行は、以下の多層的なチェックを通過する必要があります。

```text
User / Administrator
   ↓
[ Interface Level ] (MPP: 3.2.a で物理IFを制限)
   ↓
[ Protocol Level ] (SSH/HTTPS のみ許可)
   ↓
[ Access-List Level ] (特定の管理IPのみ許可)
   ↓
[ AAA Level ] (Username/Password, OTP, RSA Key等の検証)
   ↓
[ Privilege Level ] (実行可能なコマンドの決定)
```

---

## ⚙ 動作シーケンス

1.  **接続要求**: ユーザーが SSH/Console 等でデバイスに接続。
2.  **プロトコル検証**: 要求されたサービス（プロトコル）が有効か確認。
3.  **アクセスリスト検証**: `access-class` または `http 10.1.1.0 ...` コマンドに基づき送信元を検証。
4.  **認証 (Authentication)**: ログイン情報の入力を求め、AAAサーバーまたはローカルDBで照合。
5.  **認可 (Authorization)**: ログイン後のユーザーに割り当てられた特権レベルを決定。
6.  **管理セッション開始**: シェルまたはGUIへのアクセスを許可。
7.  **検課 (Accounting)**: セッション開始やコマンド実行をログ（Syslog/TACACS+）に記録。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **SSH v2の厳格な設定**: ラボでは単なる `crypto key generate rsa` だけでなく、`ip ssh version 2`、`ip ssh time-out`、および送信元インターフェイスの固定 (`ip ssh source-interface`) が一括で求められることがあります。
*   **脆弱なサービスの徹底排除**: 「いかなる非暗号化トラフィックもデバイスに入らないようにせよ」という要件に対し、Telnet/HTTPの停止と、VTYでの `transport input ssh` の設定が必要です。
*   **Privilege Levelの使い分け**: 特定のユーザーにはレベル1（ユーザーモード）、管理者にはレベル15（特権モード）を割り当てる、あるいは `enable secret` の設定ミスを誘う問題に注意してください。
*   **ASAの管理設定**: ASAでは `http [IP] [MASK] [IF_NAME]` や `ssh [IP] [MASK] [IF_NAME]` コマンドを使用して、プロトコル単位でインターフェイスと送信元を縛る必要があります。
*   **ブルートフォース保護**: `login block-for [SECONDS] attempts [COUNT] within [SECONDS]` コマンドの正確な構文を覚えておきましょう。
*   **タイムアウト設定**: ラボ要件で「セッションを5分間放置したら自動切断せよ」とあれば、`exec-timeout 5 0` を `line vty` と `line con 0` の両方に設定します。

---

## 🛠 設定方法

### 1. IOS-XE: SSH v2 とアクセスのハードニング
```bash
! ドメイン名設定 (キー生成に必須)
ip domain-name ccie.local
! RSAキー生成
crypto key generate rsa modulus 2048
! SSH設定
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
! 送信元ACL作成
ip access-list standard ACL-MGMT-ACCESS
 permit 192.168.100.0 0.0.0.255
! VTY適用
line vty 0 4
 access-class ACL-MGMT-ACCESS in
 transport input ssh
 exec-timeout 5 0
```

### 2. ASA: 管理アクセス制御
```bash
! 送信元を制限してSSHを許可
ssh 192.168.10.0 255.255.255.0 inside
ssh timeout 10
! ASDM (HTTPS) サーバー設定
http server enable
http 192.168.10.10 255.255.255.255 inside
```

### 3. 特権レベルとローカル認証
```bash
username admin privilege 15 secret Cisc0123
aaa new-model
aaa authentication login default local
aaa authorization exec default local
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **SSHの設定・セッション確認** | <code>show ip ssh</code> / <code>show ssh</code> |
| **VTY/Lineの状態確認** | <code>show line</code> |
| **適用されているACLの確認** | <code>show access-lists</code> |
| **ユーザーと特権レベルの確認** | <code>show users</code> / <code>show privilege</code> |
| **ASAの管理設定確認** | <code>show run http</code> / <code>show run ssh</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 外部からSSHが接続拒否される | RSAキーが生成されていない | <code>show crypto key mypubkey rsa</code> を確認。 |
| パスワードが正しいのにログイン不可 | <code>aaa authorization</code> の設定不備 | <code>aaa authorization exec default local</code> があるか確認。 |
| VTY ACLで自分自身を拒否した | ACLの送信元IPに自身のIPが含まれていない | コンソールからログインし、ACLを修正。 |
| SSH v1しか受け付けない | <code>ip ssh version 2</code> の設定漏れ | 設定を確認し、明示的に v2 を指定。 |

---

## ⚠ 制限事項

*   **RSAキーの長さ**: 768ビット未満のキーでは SSH v2 が動作しないプラットフォームがあります。1024ビット以上を推奨。
*   **VTYの数**: デフォルト（通常0-4）以上のセッションが必要な場合は、`line vty 5 15` にも同じセキュリティ設定を適用する必要があります。
*   **コンソールポート**: 物理アクセスが可能な場合、パスワードリカバリ手順によりセキュリティ設定がバイパスされる可能性があります。物理的な保護が前提です。

---

## 🔄 他技術との関連

*   **3.1.a CoPP**: SSHトラフィックなどの流量を制限し、DDoS攻撃からCPUを保護します。
*   **3.2.a CPU (MPP)**: どの物理ポートでSSH/HTTPSを受け付けるかを制限します。
*   **2.1 AAA (ISE)**: TACACS+を使用して、デバイスアクセスの認証・認可を中央管理します。
*   **Infrastructure Hardening (3.1)**: デバイスアクセス保護は、インフラ全体の要塞化戦略の一部です。

---

## 🧩 比較表

### SSH vs Telnet

| 特徴 | SSH (v2) | Telnet |
| :--- | :--- | :--- |
| **機密性** | 高い (暗号化) | **なし (クリアテキスト)** |
| **完全性** | あり (ハッシュ検証) | なし |
| **認証** | ユーザー/パスワード, 公開鍵 | パスワードのみ |
| **推奨** | **必須** | **使用禁止** |

---

## 💡 ベストプラクティス

1.  **Transport Inputの固定**: 全VTY行で `transport input ssh` を設定し、Telnetを強制遮断します。
2.  **AAA 冗長化**: 外部AAA（ISE）を使用する場合でも、万が一のネットワーク断に備え、ローカルDBをバックアップ（`aaa ... group tacacs+ local`）として設定します。
3.  **Management VRF**: 管理トラフィックをデータトラフィックから分離するために、専用のVRFを使用します。
4.  **Logging**: 全てのログイン試行と実行されたコマンド（Accounting）を外部Syslogサーバーへ転送します。

---

## 📝 ラボ学習・設定サンプル例

### 1. SSH v2 への完全移行
*   **要件**: R1でSSH v2のみを有効にし、Telnetを無効化せよ。ドメイン名は `ciscolab.com` とする。
*   **設定**: `ip domain-name ciscolab.com`, `crypto key generate rsa ...`, `ip ssh version 2`, `line vty 0 15` > `transport input ssh`。

### 2. VTY アクセス制限
*   **要件**: ホスト 10.1.1.100 からの SSH のみ許可せよ。
*   **設定**: `access-list 1 permit 10.1.1.100`, `line vty 0 4` > `access-class 1 in`。

### 3. 特権レベルの設定 (Level 15)
*   **要件**: ユーザー `admin` がログインした際、即座に特権モード (`#` プロンプト) になるようにせよ。
*   **設定**: `username admin privilege 15 secret cisco123`。

### 4. セッションタイムアウト
*   **要件**: 10分間放置された管理セッションを自動終了せよ。
*   **設定**: `line vty 0 15` > `exec-timeout 10 0`。

### 5. ブルートフォース攻撃の緩和
*   **要件**: 1分間に3回ログインに失敗したIPを15分間ブロックせよ。
*   **設定**: `login block-for 900 attempts 3 within 60`。

### 6. 警告バナーの実装
*   **要件**: ログイン前に "UNAUTHORIZED ACCESS PROHIBITED" と表示せよ。
*   **設定**: `banner motd # UNAUTHORIZED ACCESS PROHIBITED #`。

### 7. ASA HTTP 管理の制限
*   **要件**: 内部ネットワーク 192.168.1.0/24 からのみ ASDM アクセスを許可せよ。

### 8. FTD 診断インターフェイスの保護
*   **課題**: FMCから管理対象デバイスの管理アクセスポリシーを設定せよ。

### 9. SSH 送信元インターフェイスの固定
*   **要件**: ルータが自身から他へSSHする際、常に Loopback0 のIPを使用せよ。
*   **設定**: `ip ssh source-interface Loopback0`。

### 10. AAA 連携によるログイン
*   **要件**: デバイスログインに TACACS+ サーバーを使用し、失敗時のみローカルDBを使用せよ。
*   **設定**: `aaa authentication login default group tacacs+ local`。

---

## ❓ 想定試験問題

1.  **実装**: ルータ R1 において、SSH キーを生成したが SSH 接続ができない。トラブルシューティングに必要な最初の確認コマンドは？
    *   **回答**: `show ip ssh`（SSHが有効か、バージョンは何かを確認）。
2.  **Design**: セキュリティポリシーにより、クリアテキストによるパスワードの送出を禁止する場合、VTY 行で設定すべきコマンドは？
    *   **回答**: `transport input ssh`（これにより Telnet が拒否される）。
3.  **コンフィグ読解**: `line vty 0 4` に `access-class 10 in` が設定されている。ACL 10 が `deny ip any any` の場合、何が起きるか？
    *   **回答**: すべてのネットワーク経由のログインが拒否される。コンソールからのアクセスは可能。
4.  **トラブルシュート**: 管理者がログイン後、特定の `show` コマンドは実行できるが `configure terminal` が実行できない。考えられる原因は？
    *   **回答**: ユーザーに割り当てられた特権レベル（Privilege level）が低すぎる（15未満）。
5.  **Design**: 大量のリモートログイン試行（DDoS）からデバイスを保護するために、3.2.c 以外に併用すべき技術は？
    *   **回答**: **CoPP (3.1.a)** または **MPP (3.2.a)**。

---

## 🔗 参考リソース

*   **Cisco IOS-XE Configuration Guide**
    *   [Securing User Services](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_user_user/configuration/xe-16/sec-user-user-xe-16-book.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [Securing the Control Plane and Management Plane](https://www.ciscolive.com/)
*   **Technical Notes**
    *   [Cisco Guide to Harden Cisco IOS Devices](https://www.cisco.com/c/en/us/support/docs/ip/access-lists/13608-21.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「管理アクセスの保護 = 暗号化(SSH/HTTPS) + 送信元IP制限(ACL) + 厳格な認証(AAA)」の3点セットで覚えましょう。
*   **図解**: ユーザーが PC から VTY 経由でルータに入る際、`line vty` の各設定がどのようにフィルターとして機能するかイメージ図を書いてみてください。
*   **注意点**: ラボ試験中に `aaa new-model` を入力すると、既存のパスワード設定が無効になる場合があります。入力前に必ず `username` と `privilege 15` の設定があることを確認してください。
