---
layout: default
title: 2.1.a Cisco AnyConnect client-based, remote-access VPN technologies on Cisco ASA
nav_order: 1
parent: 2.1-AnyConnect-RA-VPN
grand_parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.1.a Cisco AnyConnect client-based, remote-access VPN technologies on Cisco ASA

Cisco ASAにおける **AnyConnect リモートアクセス VPN** は、モバイルユーザーやテレワーカーがインターネット越しに内部ネットワークリソースへ安全に接続するための基盤技術です。SSL/TLS（WebVPN）または IKEv2 プロトコルを使用し、認証、認可、アカウンティング（AAA）を組み合わせて柔軟なアクセス制御を実現します。CCIE Security ラボ試験では、証明書認証、外部サーバー（ISE/RADIUS）連携、および詳細なクライアントプロファイル設定の実装能力が問われます。

---

## 📘 概要

*   **機能概要**: 
    *   **AnyConnect クライアント**を使用して、ASA との間で暗号化されたトンネルを確立します。
    *   プロトコルとして **SSL (TLS/DTLS)** または **IKEv2** を選択可能です。
*   **利用目的**: 公衆回線経由でのセキュアな企業リソースへのアクセス、エンドポイントのセキュリティ検疫（HostScan/Endpoint Posture）。
*   **どのような場面で利用するか**: 
    *   全社的なリモートワーク環境の提供。
    *   特定のパートナーやベンダーに対する限定的な内部サーバー公開。
    *   モバイルデバイス（iOS/Android）からの安全な業務アプリ利用。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **トランスポート層** | SSL/TLS (TCP 443) または IKEv2 (UDP 500/4500)。 |
| **認証方式** | AAA (RADIUS/LDAP/Local)、証明書 (Certificate)、二要素認証 (2FA/Duo)。 |
| **IP割り当て** | ローカルアドレスプール、DHCP、または外部 AAA サーバーからの属性付与。 |
| **ポリシー適用** | **Group Policy** と **Tunnel Group (Connection Profile)** による階層管理。 |
| **最適化技術** | **DTLS** (Datagram TLS) による音声・ビデオトラフィックの UDP 高速化。 |
| **ライセンス** | AnyConnect Plus/Apex/VPN Only ライセンスが必要。 |
| **設計上の注意** | クライアントプロファイル (XML) は ASA のフラッシュに保存し、接続時に配布する。 |

---

## 🏗 動作原理

AnyConnect は、**トンネルインターフェイス**を論理的に構築し、ASA をデフォルトゲートウェイまたは特定のサブネットへの出口として動作させます。

```text
[ AnyConnect Client ]
        ↓
    (HTTPS/SSL or IKEv2)
        ↓
[ ASA Outside Interface ]
        ↓
[ 1. Connection Profile (Tunnel Group) Selection ] <--- URL, Alias, or Cert
        ↓
[ 2. Authentication (AAA/Cert) ]
        ↓
[ 3. Authorization (Group Policy/DAP) ]
        ↓
[ 4. VPN Tunnel Establishment (Virtual IP Assigned) ]
        ↓
[ Internal Network ]
```

---

## ⚙ 動作シーケンス

1.  **接続確立**: クライアントが ASA の Outside IP に接続し、SSL ハンドシェイクまたは IKEv2 認証を開始します。
2.  **プロファイル選択**: 送信された URL (`group-url`) やエイリアス (`group-alias`) に基づき、適切な **Connection Profile** が選ばれます。
3.  **認証**: ユーザー名/パスワード、あるいはデバイス証明書による検証が行われます。
4.  **認可 (Attribute push)**: ASA は **Group Policy** を適用し、スプリットトンネル、DNS、タイムアウト設定などをクライアントにプッシュします。
5.  **IP割り当て**: クライアントに仮想 IP アドレス（Internal プール）が割り当てられます。
6.  **データ転送**: トンネルが「UP-ACTIVE」状態になり、パケットの暗号化転送が開始されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Group-URL と Group-Alias**: ユーザーに「どのプロフィールを使うか」をドロップダウンリストで選ばせる設定（Alias）や、特定の URL で接続させる設定は頻出です。
*   **Split Tunneling**: 「全通信を VPN に通す」か「内部宛のみ通す」か。ACL と `split-tunnel-policy` の組み合わせを正確に記述できる必要があります。
*   **証明書認証 (Certificate Map)**: ユーザー名の入力を省き、証明書の共通名 (CN) や OU に基づいてトンネルグループを自動選択する設定。
*   **AnyConnect IKEv2**: SSL ではなく IKEv2 を使用する場合の `crypto ikev2` ポリシーと `virtual-template` (不要な場合もあるが ASA の特性を理解) の設定。
*   **Client Profile (XML)**: ラボ環境の PC 上で XML エディタや ASDM を使い、自動接続 (Always-on) やバックアップサーバーリストを含むプロファイルを作成し、ASA にアップロードする。
*   **Troubleshooting**: `show vpn-sessiondb remote` でセッションを確認し、`debug webvpn anyconnect` で認証プロセスを追跡する能力が求められます。

---

## 🛠 設定方法

### 1. 基本的な SSL AnyConnect 設定 (CLI)
```bash
! IPアドレスプールの作成
ip local pool VPN_POOL 10.10.10.100-10.10.10.200 mask 255.255.255.0

! AnyConnect イメージの指定
webvpn
 anyconnect image disk0:/anyconnect-win-4.x.pkg 1
 anyconnect enable
 tunnel-group-list enable

! グループポリシーの設定
group-policy SALES_POLICY internal
group-policy SALES_POLICY attributes
 vpn-tunnel-protocol ssl-client
 split-tunnel-policy tunnelspecified
 split-tunnel-network-list value SPLIT_ACL
 address-pools value VPN_POOL

! 接続プロファイル (Tunnel Group) の設定
tunnel-group SALES_TG type remote-access
tunnel-group SALES_TG general-attributes
 default-group-policy SALES_POLICY
 authentication-server-group ISE_RADIUS
tunnel-group SALES_TG webvpn-attributes
 group-alias SALES_VPN enable
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **接続ユーザーの概要確認** | <code>show vpn-sessiondb remote</code> |
| **プロトコルごとの統計** | <code>show vpn-sessiondb ra protocol ssl-client</code> |
| **IKEv2 セッションの確認** | <code>show crypto ikev2 sa</code> |
| **WebVPN 設定の稼働確認** | <code>show webvpn anyconnect</code> |
| **詳細なセッション情報** | <code>show vpn-sessiondb detail remote [User]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 接続時に "Login Failed" | AAA サーバーの応答なし、またはパスワードミス | <code>debug aaa common 10</code> | ASA と AAA サーバー間の Key と接続性を確認。 |
| 特定の通信が通らない | スプリットトンネル ACL の不備 | <code>show access-list</code> | ACL に対象ネットワークが含まれているか確認。 |
| SSL VPN が開けない | <code>http server</code> の競合、または IF で無効 | <code>show run webvpn</code> | <code>anyconnect enable</code> が対象 IF で設定されているか。 |
| 証明書エラーが出る | トラストポイントの欠如、時刻不一致 | <code>show crypto ca certificates</code> | CA 証明書をインストールし NTP を同期させる。 |
| パケットが ASA を抜けない | Inside へのルートがない、または ACL 遮断 | <code>packet-tracer</code> | 戻りのルートが VPN 割り当て IP 宛にあるか確認。 |

---

## ⚠ 制限事項

*   **同時接続数**: ASA モデルごとに最大セッション数が決まっており、ライセンスによって制限される場合があります。
*   **DTLS の要件**: UDP 443 がネットワーク経路（Firewall/ISP）で許可されている必要があります。
*   **ブラウザ互換性**: クライアントレス (Web-only) VPN の場合はブラウザ依存が強いですが、AnyConnect (Client-based) は OS 側の互換性が重要です。

---

## 🔄 他技術との関連

*   **AAA (RADIUS/ISE)**: 外部データベースによる高度なユーザー管理。
*   **DAP (Dynamic Access Policies)**: 接続してきた端末の OS バージョンやアンチウイルス状態に基づき、動的に ACL を変更。
*   **TrustSec**: VPN ユーザーに **SGT (Security Group Tag)** を付与し、内部スイッチでのセグメンテーションを実現。
*   **Identity Firewall**: VPN ユーザー名に基づいたアクセスルール (`access-list`) の適用。

---

## 🧩 比較表

### AnyConnect SSL vs AnyConnect IKEv2

| 機能 | AnyConnect (SSL/TLS) | AnyConnect (IKEv2) |
| :--- | :--- | :--- |
| **ポート** | TCP 443 (標準 HTTPS) | UDP 500 / 4500 |
| **透過性** | プロキシや Firewall を通りやすい | 一部の環境でブロックされる可能性がある |
| **パフォーマンス** | DTLS を併用すれば高い | IPSec ネイティブのため非常に高い |
| **認証** | 柔軟 (AAA, Cert, SAML) | 強固 (EAP-AnyConnect) |
| **主な用途** | 一般的な RA 接続の第一選択 | 高速な通信や常時接続が必要な環境 |

---

## 💡 ベストプラクティス

1.  **DTLS の有効化**: 遅延に敏感なアプリのため、常に DTLS を有効にすることを推奨します。
2.  **証明書認証の優先**: セキュリティと利便性の両立のため、マシン証明書によるパスワードレス接続を検討します。
3.  **Group-URL の活用**: ブラウザに接続先 IP を打たせるのではなく、`vpn.cisco.com/sales` のような分かりやすい URL を提供します。
4.  **スプリットトンネルの最小化**: インターネット向け通信を VPN に戻さないことで、ASA の負荷とインターネット遅延を軽減します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 外部 RADIUS (ISE) 認証の実装
*   **要件**: AnyConnect ユーザーの認証を ISE (192.168.1.50) で行え。
*   **設定**: `aaa-server ISE protocol radius`, `aaa-server ISE host 192.168.1.50`, `tunnel-group ANY-TG general-attributes` -> `authentication-server-group ISE`.

### 2. スプリットトンネルの ACL 設定
*   **要件**: 内部サーバー 172.16.0.0/16 への通信のみ VPN トンネルを通せ。
*   **設定**: `access-list VPN_SPLIT standard permit 172.16.0.0 255.255.0.0`, `group-policy GP attributes` -> `split-tunnel-policy tunnelspecified`, `split-tunnel-network-list value VPN_SPLIT`.

### 3. IKEv2 AnyConnect の有効化
*   **要件**: プロトコルとして IKEv2 を優先して使用せよ。
*   **設定**: `group-policy GP attributes` -> `vpn-tunnel-protocol ikev2 ssl-client`, `crypto ikev2 enable outside`.

### 4. クライアントプロファイルの配布
*   **要件**: ASA のフラッシュにある `sales.xml` をクライアントにプッシュせよ。
*   **設定**: `webvpn` -> `anyconnect profiles SALES_PROF disk0:/sales.xml`, `group-policy GP attributes` -> `webvpn` -> `anyconnect profiles value SALES_PROF`.

### 5. 証明書ベースの自動トンネル選択
*   **要件**: 証明書の OU が "Eng" の場合、ENGINEERING-TG に自動で振り分けよ。
*   **設定**: `crypto ca certificate chain ...`, `crypto certificate map MYMAP 10`, `match ou Eng`, `webvpn` -> `certificate-group-map MYMAP 10 ENGINEERING-TG`.

### 6. カスタムバナーの表示
*   **要件**: ログイン成功時に "Authorized Access Only" と表示せよ。
*   **設定**: `group-policy GP attributes` -> `banner value Authorized Access Only`.

### 7. 最大接続維持時間の設定
*   **要件**: セッションを最大 8 時間で強制切断せよ。
*   **設定**: `group-policy GP attributes` -> `vpn-session-timeout 480`.

### 8. AnyConnect クライアントの自動アップグレード
*   **要件**: ASA に新しいパッケージを配置し、接続時に更新させよ。
*   **設定**: `webvpn` -> `anyconnect image disk0:/anyconnect-win-new.pkg 2`.

### 9. IPv6 アドレスの割り当て
*   **要件**: VPN クライアントに IPv6 プレフィックスを割り当てよ。
*   **設定**: `ipv6 local pool V6POOL ...`, `group-policy GP attributes` -> `ipv6-address-pools value V6POOL`.

### 10. packet-tracer による VPN 通信テスト
*   **課題**: VPN ユーザー (10.10.10.101) から内部 (192.168.10.1) への通信をシミュレートせよ。
*   **コマンド**: `packet-tracer input outside tcp 10.10.10.101 1234 192.168.10.1 80`.

---

## ❓ 想定試験問題

1.  **トラブルシュート**: AnyConnect で接続後、社内の Web サーバーにアクセスできないが、Ping は通る。ASA 上で確認すべき項目は？
    *   **回答**: ASA の **Modular Policy Framework (MPF)** で HTTP インスペクションが影響していないか、または内部サーバー側の ACL で VPN プールからの HTTP 通信が許可されているかを確認。
2.  **Design**: 数千ユーザーが同時に接続する環境で、特定の部署ごとに異なる DNS サーバーを割り当てたい。ASA のどのコンポーネントで設定すべきか？
    *   **回答**: **Group Policy**。部署ごとにポリシーを作成し、それぞれの `dns-server` 属性を定義する。
3.  **実装**: Connection Profile に `group-url` を設定したが、クライアントから接続できない。共通設定として必要な `webvpn` 下のコマンドは？
    *   **回答**: `anyconnect enable` および `tunnel-group-list enable`（エイリアスを使用する場合）。
4.  **コンフィグ読解**: `vpn-tunnel-protocol ssl-client` のみが設定された Group Policy を持つユーザーが IKEv2 で接続しようとした場合の挙動を述べよ。
    *   **回答**: 認可フェーズでプロトコル不一致となり、接続が拒否される。
5.  **Design**: 公衆 Wi-Fi 等の不安定な環境で、短時間の通信断でも再認証を求められないようにする機能は？
    *   **回答**: **AnyConnect Session Resumption (Dead Peer Detection と併用)**。

---

## 🔗 参考リソース

*   **Cisco ASA Series VPN Configuration Guide, 9.4**
    *   [AnyConnect VPN Client Connections](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/vpn/anyconnect/asa-94-anyconnect-config/vpn-anyconnect.html)
*   **Cisco Live BRKSEC-3033**
    *   [Advanced AnyConnect Implementation and Troubleshooting](https://www.ciscolive.com/)
*   **Technical Note**
    *   [AnyConnect IKEv2 Client to ASA Configuration Example](https://www.cisco.com/c/en/us/support/docs/security/adaptive-security-appliance-asa-software/113404-anyconnect-ikev2-asa-config.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: AnyConnect は「設定項目が非常に多い」のが特徴です。まず **Tunnel Group（入口）** -> **AAA（身分証）** -> **Group Policy（中での振る舞い）** という流れを頭に叩き込みましょう。
*   **図解**: 接続プロファイルは「ゲートウェイの扉」、グループポリシーは「扉を抜けた後の通行許可証」とイメージすると理解が進みます。
*   **注意点**: ラボ試験では **XML プロファイルの構文エラー** に注意してください。GUI (ASDM) が使える場合は、ASDM のプロファイルエディタを使うのが最も安全です。
