---
layout: default
title: 3.7.a-ISO27001
nav_order: 3
parent: 3.7-Compliance
grand_parent: 3.0-Security-Infrastructure
---

# 3.7.a ISO 27001

**ISO 27001**（情報セキュリティマネジメントシステム：ISMS）は、組織が情報セキュリティを管理するための国際標準規格です。CCIE Security v6.1 の文脈においては、単なるマネジメント論ではなく、ブループリントの 3.1 から 3.6 で扱う個別の技術的コントロール（AAA、デバイスの要塞化、暗号化、監視など）をどのように組み合わせて、**「組織のセキュリティポリシーや標準に適合させるか」**という実装能力が問われます。

---

## 📘 概要

*   **機能概要**: ISO 27001 は、情報の機密性（Confidentiality）、完全性（Integrity）、可用性（Availability）の 3 要素（CIA）を維持するための枠組みです。
*   **利用目的**: リスクアセスメントに基づき、適切な技術的・組織的対策を講じることで、セキュリティ事故の発生を最小限に抑え、コンプライアンス（法令・規制遵守）を達成します。
*   **どのような場面で利用するか**: 
    *   **エンタープライズネットワークの設計**: 管理プレーン、データプレーン、コントロールプレーンの各層で ISO 27001 の管理策（Annex A）に準拠した設定を実装します。
    *   **監査対応**: 誰が、いつ、何をしたかという「監査証跡」を残すためのロギングや AAA の徹底。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **標準規格** | ISO/IEC 27001:2022 (最新版)。 |
| **中核概念** | **PDCA サイクル**（Plan-Do-Check-Act）およびリスクマネジメント。 |
| **技術的コントロール** | **AAA (認証/承認/認可)**、**暗号化**、**ロギング**、**境界防御**。 |
| **CCIE での役割** | 3.1～3.6 の技術を ISO の「管理策」としてマッピングし、統合する能力。 |
| **メリット** | 体系的な要塞化（Hardening）により、設定漏れを防ぎ、防御力を最大化する。 |
| **設計上の注意点** | セキュリティ強度と利便性のトレードオフを、組織の許容リスクに基づいて調整する。 |

---

## 🏗 動作原理

ISO 27001 の管理策（Annex A）を Cisco ネットワークデバイスの実装にマッピングすると、以下のようになります。

1.  **アクセス制御 (Access Control)**:
    *   **AAA (RADIUS/TACACS+)**: ユーザー識別と個別の権限付与。
    *   **RBAC (Role-Based Access Control)**: 特権レベル（Privilege levels）やパーサービューによる制御。
2.  **暗号化 (Cryptography)**:
    *   **IKEv2/IPsec**: 拠点間・リモートアクセスのデータ保護。
    *   **WPA3/AES**: 無線区間の暗号化。
3.  **運用セキュリティ (Operations Security)**:
    *   **NTP**: 時刻同期によるログの整合性。
    *   **Syslog/SNMPv3/NetFlow**: 継続的な監視と事後解析。
4.  **通信セキュリティ (Communications Security)**:
    *   **VACL/ACL/uRPF**: トラフィックの分離とスプーフィング防止。

---

## ⚙ 動作シーケンス

ISO 27001 に基づくセキュリティ実装のプロセスは以下の通りです。

1.  **アセット特定**: 保護すべきデバイスやインターフェイスを特定（Blueprint 3.1-3.4）。
2.  **リスク分析**: どのような脅威（なりすまし、DoS、傍受）があるかを特定。
3.  **コントロール実装**:
    *   デバイスアクセスの保護（SSH, AAA）。
    *   コントロールプレーンの保護（CoPP）。
    *   トラフィック監視の設定（NetFlow, Syslog）。
4.  **検証と見直し**: 定期的な監査（show コマンド等）や、検知したイベントに基づく設定の最適化。

---

## 🎯 試験対策（CCIE Securityラボ試験）

ラボ試験では「ISO 27001 を設定せよ」という直接的な問題ではなく、**「組織のコンプライアンス要件を満たすよう、以下のセキュリティ要件を実装せよ」**という形で出題されます。

*   **AAA の徹底**: デバイスへのログインは必ず個人のユーザー ID を使用し、全ての実行コマンドを TACACS+ サーバへ記録（Accounting）することが求められます。
*   **管理通信の保護**: Telnet や HTTP を禁止し、**SSH v2 (AES256)** や **HTTPS (TLS 1.2以上)** を使用して管理プレーンを要塞化します。
*   **監視の不備を突く**: ログ設定（Syslog）が不十分で、誰が設定を変更したか追えない状態を修正する問題が想定されます。
*   **パスワードポリシー**: `security passwords min-length` などのコマンドを使用して、組織のポリシーに準じた複雑性を持たせます。
*   **不要なサービスの停止**: Compliance 準拠のため、Finger, HTTP, CDP（必要時以外）などの脆弱なサービスを無効化します。

---

## 🛠 設定方法

ISO 27001 の「運用セキュリティ」および「アクセス制御」に準拠するための代表的な設定例です。

### 1. AAA によるアクセス制御と監査証跡
```bash
! 個別ユーザーの作成（組織ポリシーによる最小特権）
username ADMIN privilege 15 secret Cisco123!

! TACACS+ によるコマンド認可とアカウンティングの設定
aaa new-model
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local 
aaa authorization commands 15 default group tacacs+ local 
aaa accounting commands 15 default start-stop group tacacs+
```

### 2. 監視と整合性の確保 (Logging & NTP)
```bash
! 時刻同期（ログの証拠能力維持に必須）
ntp server 10.1.1.1
service timestamps log datetime msec show-timezone

! 詳細なロギングの設定
logging buffered 16384 informational
logging host 192.168.100.50
logging trap informational
```

### 3. コントロールプレーン保護 (CoPP) による可用性確保
```bash
! 管理トラフィックのみを許可し、不要な通信をドロップ/レート制限
ip access-list extended COPP-ACL
 permit tcp any host 172.16.1.1 eq ssh
 permit snmp any any
!
policy-map COPP-POLICY
 class COPP-CLASS
  police 100000 conform-action transmit exceed-action drop
!
control-plane
 service-policy input COPP-POLICY
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **AAA の適用状態確認** | <code>show aaa sessions</code> / <code>show users</code> |
| **ログの監査証跡確認** | <code>show logging</code> |
| **要塞化設定の確認** | <code>show run \| include (ssh\|aaa\|logging\|snmp)</code> |
| **CoPP の動作統計** | <code>show policy-map control-plane</code> |
| **NTP 同期状態（信頼性）** | <code>show ntp status</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 設定変更ログがサーバに残らない | Accounting 設定漏れ | <code>aaa accounting commands</code> が設定されているか確認。 |
| ログの時刻が大幅にずれている | NTP 非同期 | <code>show ntp status</code> を確認し、Stratum レベルをチェック。 |
| 管理アクセスが遮断された | CoPP または ACL 設定ミス | コンソールから入り <code>show access-lists</code> でドロップを確認。 |
| 脆弱性スキャンで指摘される | 不要なサービスが稼働中 | <code>show control-plane host open-ports</code> で不要ポートを特定。 |

---

## ⚠ 制限事項

*   **ハードウェア負荷**: 過度なロギングや NetFlow 収集は、古いデバイスの CPU/メモリを圧迫する可能性があります。
*   **後方互換性**: レガシーな Supplicant や管理クライアントが、最新の暗号化プロトコル（TLS 1.3 や WPA3）をサポートしていない場合があります。
*   **一元管理の限界**: 全てのデバイスで ISO 27001 準拠の設定を手動で維持するのは困難なため、DNA Center 等のオーケストレーションツールが必要になります。

---

## 🔄 他技術との関連

*   **3.1.a CoPP**: ISO 27001 の「可用性」確保に不可欠。
*   **3.6.b SNMP / 3.6.c Syslog**: 監視と事故対応（Incident Management）の基盤。
*   **3.7.b RFC 2827 (BCP 38)**: スプーフィング防止による「ネットワークの完全性」の担保。
*   **Cisco ISE**: 組織全体のアクセス制御を一元化し、ISO 27001 の「資産へのアクセス制限」を自動化します。

---

## 🧩 比較表

### ISO 27001 vs PCI-DSS

| 特徴 | ISO 27001 | PCI-DSS |
| :--- | :--- | :--- |
| **対象** | あらゆる組織の情報資産 | クレジットカード情報を取り扱う組織 |
| **性質** | フレームワーク（自由度が高い） | 厳格なチェックリスト（具体的要件） |
| **CCIE 的視点** | インフラ全般の要塞化を提唱 | 特定セグメントの強力な保護と暗号化 |
| **ログ保持** | 推奨されるが期間は組織次第 | 1年以上の保持など具体的な数値あり |

---

## 💡 ベストプラクティス

1.  **最小特権の原則 (PoLP)**: `privilege levels` や `Role-based CLI views` を活用し、管理者ごとに必要なコマンドのみを許可します。
2.  **「デフォルト拒否」の姿勢**: Ingress ACL や CoPP を使用し、明示的に許可した通信以外を遮断します。
3.  **証拠の保全**: ログの送信元 IP を `logging source-interface` で固定し、NTP で時刻を保証します。
4.  **暗号化の現代化**: DES や 3DES を排除し、AES-GCM 等の最新アルゴリズムを採用します。

---

## 📝 ラボ学習・設定サンプル例

### 1. パスワードの要塞化
*   **要件**: 最小パスワード長を 10 文字とし、不正ログイン試行を 3 回でロックせよ。
*   **設定**: 
    ```bash
    security passwords min-length 10
    login block-for 60 attempts 3 within 30
    ```

### 2. 管理アクセスの SSH 制限
*   **要件**: VTY 0-4 では SSH v2 のみ許可し、Telnet を排除せよ。
*   **設定**: 
    ```bash
    line vty 0 4
     transport input ssh
    ```

### 3. TACACS+ による監査ログ
*   **要件**: R1 上で実行された全ての特権コマンドをサーバ 10.1.1.5 へ記録せよ。

### 4. CoPP による可用性の保護
*   **要件**: コントロールプレーンへの ICMP 流量を 100Kbps に制限せよ。

### 5. 時刻同期による整合性確保
*   **要件**: ミリ秒単位の正確なタイムスタンプを設定せよ。

### 6. 不要なサービスの無効化
*   **要件**: HTTP Server, Finger, Source-route を無効化せよ。

### 7. SNMPv3 によるセキュア監視
*   **要件**: 認証(SHA)と暗号化(AES)を備えた SNMPv3 を構成せよ。

### 8. NetFlow による可視化
*   **要件**: フロー情報を収集し、異常なトラフィックを特定可能にせよ。

### 9. ログインバナーによる警告
*   **要件**: ログイン時に「認可されたユーザーのみ」である旨の警告を表示せよ。
*   **設定**: `banner login ^C Authorized Access Only ^C`

### 10. インフラストラクチャ ACL (iACL)
*   **要件**: ネットワーク境界で、自装置のループバック宛の通信を管理端末以外から遮断せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `logging trap 4` が設定されている。ISO 27001 監査において「設定変更（Informational）」が記録されないと指摘された。どのように修正すべきか？
    *   **回答**: レベル 6 (Informational) 以上が送出されるよう `logging trap 6` に変更する。
2.  **トラブルシュート**: TACACS+ サーバは稼働しているが、管理者が実行したコマンドが外部ログに記録されない。考えられる原因は？
    *   **回答**: `aaa accounting commands` 設定の欠落、または VTY ラインでの `accounting` 適用漏れ。
3.  **Design**: ISO 27001 の「可用性」を維持するために、大量の DDoS パケットからルータの CPU を守る手法を述べよ。
    *   **回答**: **CoPP (Control-Plane Policing)** を実装し、特定のクラス以外のトラフィックをドロップまたはレート制限する。
4.  **実装**: 共有 ID（cisco/cisco 等）の使用を禁止し、個人 ID でのログインを強制する Cisco の機能は？
    *   **回答**: `aaa authentication login default group tacacs+` による外部認証。
5.  **Design**: ネットワーク機器の時刻がずれている場合、ISO 27001 のどの側面で問題が生じるか？
    *   **回答**: 「完全性」と「運用セキュリティ」。事後解析時にログの時系列が特定できず、証拠能力を失う。

---

## 🔗 参考リソース

*   **Cisco Live (BRKSEC-2003)**: [Securing the Infrastructure: Management and Control Plane](https://www.ciscolive.com/)
*   **Cisco Configuration Guide**: [Configuring Management Plane Protection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_cfg/configuration/xe-16/sec-usr-cfg-xe-16-book.html)
*   **Cisco White Paper**: [Hardening Cisco IOS Devices](https://www.cisco.com/c/en/us/support/docs/ip/access-lists/13608-21.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: ISO 27001 は「全ての穴を塞ぐためのチェックリスト」です。CCIE 試験では、設定を一つ忘れる（例えば logging source を忘れる等）だけでコンプライアンス要件に失敗することを意識してください。
*   **図解**: 
    *   **Management Plane**: SSH, AAA, SNMPv3, Logging, NTP.
    *   **Control Plane**: CoPP, Routing Auth.
    *   **Data Plane**: ACLs, uRPF, QoS.
*   **注意点**: ラボ試験では、問題文に "ISO 27001" と書かれている場合、それは「可能な限りの要塞化（Hardening）を施せ」という強いメッセージであることが多いです。
