---
layout: default
title: 3.3.a-uRPF
nav_order: 1
parent: 3.3-Data-plane
grand_parent: 3.0-Security-Infrastructure
---

# 3.3.a uRPF (Unicast Reverse Path Forwarding)

**uRPF (Unicast Reverse Path Forwarding)** は、ネットワークデバイスを通過するトラフィックの送信元 IP アドレスが正当なものであるかを検証し、**IP スプーフィング（なりすまし）攻撃**を効果的に防御するためのデータプレーン保護技術です。CCIE Security v6.1 において、uRPF はインフラストラクチャの堅牢化（System Hardening）および脅威緩和（Threat Mitigation）のセクションで不可欠な要素として位置付けられています。

---

## 📘 概要

*   **機能概要**: ルータがパケットを受信した際、そのパケットの「送信元 IP アドレス」をキーとして **FIB (Forwarding Information Base)** を逆引きし、パケットが正当なインターフェイスから到着したかを確認します。
*   **利用目的**: 送信元 IP アドレスを偽装した DoS/DDoS 攻撃、スキャニング、および反射攻撃をネットワークの境界（エッジ）で遮断することにあります。
*   **利用場面**:
    *   **エンタープライズのエッジ**: インターネットから内部ネットワークへのスプーフィングパケットの流入防止。
    *   **ISP のアクセス境界**: 顧客ネットワークから偽装された送信元 IP を持つパケットの送出防止（BCP 38 準拠）。
    *   **キャンパス内**: 内部のホスト間でのスプーフィング防止。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **前提条件** | **CEF (Cisco Express Forwarding)** がグローバルで有効であること。 |
| **Strict モード** | 送信元への戻りパスが、パケットを受信したインターフェイスと厳密に一致する必要がある。 |
| **Loose モード** | 送信元へのルートが FIB に存在すれば、どのインターフェイスから受信しても許可される。 |
| **VRF モード** | 特定の VRF ルーティングテーブルに基づいて検証を行う。 |
| **主なオプション** | `allow-default`（デフォルトルートを有効なパスとして扱う）、`allow-self-ping`、ACL による例外処理。 |
| **メリット** | ハードウェア（ASIC/FIB）レベルで動作するため、パフォーマンスへの影響が極めて低い。 |
| **設計上の注意点** | **非対称ルーティング（Asymmetric Routing）** 環境で Strict モードを使用すると、正当な通信がドロップされる。 |

---

## 🏗 動作原理

通常、ルータは「宛先 IP」を見てパケットを転送しますが、uRPF は「送信元 IP」を見て「このパケットはここから来るべきものか？」を判断します。

```text
[ Attacker (Spoofed IP: 10.1.1.1) ]
      ↓
[ Incoming Interface: Gi0/1 ]
      ↓
[ uRPF Check (Check FIB for 10.1.1.1) ]
      │
      ├─ Strict Mode: 10.1.1.1 への最短パスは Gi0/1 か？
      │      └─ NO (最短パスは Gi0/2) → 【DROP】
      │
      └─ Loose Mode: 10.1.1.1 へのルートが FIB にあるか？
             └─ YES → 【PERMIT】
```

---

## ⚙ 動作シーケンス

1.  **パケット着信**: インターフェイスで IP パケットを受信します。
2.  **FIB 参照**: 送信元 IP アドレスをキーとして CEF/FIB テーブルを検索します。
3.  **インターフェイス検証**:
    *   **Strict モード (`reachable-via rx`)**: 受信インターフェイスが、FIB における送信元への「最適パス（Best Return Path）」であるか確認します。
    *   **Loose モード (`reachable-via any`)**: 送信元へのルートが FIB 内に存在するか（宛先が Null0 以外か）のみを確認します。
4.  **アクション実行**:
    *   検証成功: 通常の転送処理（L3 ルックアップ）へ移行します。
    *   検証失敗: パケットをドロップします。ACL が指定されている場合は、ロギングやカウンタの更新を行います。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **前提条件の確認**: ラボ試験で uRPF が効かない場合、まず `ip cef` または `ipv6 unicast-routing` が有効かを確認してください。
*   **「非対称ルーティング」の要件**: 問題文に「複数の ISP と接続している」「非対称パスが存在する可能性がある」といった記述があれば、**Loose モード** (`reachable-via any`) を選択するのが定石です。
*   **allow-default の重要性**: デフォルトルートしか持たないエッジルータ（Stub サイト）で uRPF を設定する場合、`allow-default` を付け忘れるとインターネットからのトラフィックが全てドロップされます。これは非常に多い「落とし穴」です。
*   **IPv6 への対応**: `ipv6 verify unicast source...` コマンドを使用して、IPv6 トラフィックにも同様の保護を適用する問題が出題されます。
*   **ASA での uRPF**: ASA では `ip verify reverse-path interface [IF_NAME]` コマンドを使用します。IOS と構文が異なるため注意が必要です。
*   **ロギングの要求**: 「ドロップされたスプーフィングパケットを特定せよ」という要件では、ACL と組み合わせて `log` オプションを使用します。

---

## 🛠 設定方法

### 1. IOS-XE: Strict モード（最も一般的）
戻りパスが受信インターフェイスと一致することを要求します。
```bash
interface GigabitEthernet0/1
 ip verify unicast source reachable-via rx
```

### 2. IOS-XE: Loose モード（非対称環境用）
ルートの存在のみを確認します。
```bash
interface GigabitEthernet0/1
 ip verify unicast source reachable-via any
```

### 3. IOS-XE: デフォルトルートを許可する設定
ISP 接続などで具体的な個別ルートがない場合に必須です。
```bash
interface GigabitEthernet0/1
 ip verify unicast source reachable-via rx allow-default
```

### 4. IOS-XE: 例外 ACL とロギングの併用
特定の IP を検証から除外したり、ドロップ時にログを記録します。
```bash
access-list 100 permit ip 10.1.1.0 0.0.0.255 any
!
interface GigabitEthernet0/1
 ip verify unicast source reachable-via rx 100
 ! ACL 100 で permit されたパケットは uRPF チェックをバイパスする
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **uRPF の有効化状態と統計** | <code>show ip interface [type] \| include verify</code> |
| **uRPF ドロップ数の確認** | <code>show ip interface [type]</code> (Verification drops 項目) |
| **CEF エントリの確認 (uRPF の根拠)** | <code>show ip cef [prefix]</code> |
| **IPv6 uRPF の状態確認** | <code>show ipv6 interface [type]</code> |
| **ドロップパケットのログ確認** | <code>show logging</code> (ACL ロギング併用時) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 正当なパケットが落ちる | 非対称ルーティング下での Strict モード使用 | <code>show ip route</code> | Loose モード (`any`) に変更する。 |
| 外部からの通信が全ドロップ | デフォルトルートの不考慮 | <code>show ip interface</code> | <code>allow-default</code> オプションを追加。 |
| uRPF が全く機能しない | CEF が無効 | <code>show ip cef</code> | <code>ip cef</code> をグローバルで有効化。 |
| 特定の管理通信が切れる | 自分自身の IP 宛通信の考慮不足 | <code>debug ip packet</code> | ACL で当該通信を permit (例外) にする。 |

---

## ⚠ 制限事項

*   **CEF 依存**: プロセススイッチング環境では動作しません。必ず CEF が必要です。
*   **マルチキャスト**: uRPF はユニキャストパケットのみを対象とします。
*   **パフォーマンス**: 基本的にハードウェア処理ですが、ACL ロギングを多用すると CPU に負荷がかかる場合があります。
*   **トンネル**: トンネルインターフェイス（GRE 等）での uRPF は、カプセル化解除後のパケットに対して適用される際の挙動に注意が必要です。

---

## 🔄 他技術との関連

*   **CEF (Cisco Express Forwarding)**: uRPF が参照する FIB の生成基盤。
*   **iACLs (Infrastructure ACLs)**: 境界ルータでインフラ IP を保護する手法。uRPF と併用して多層防御を構成。
*   **BGP (RTBH/Flowspec)**: DDoS 攻撃トラフィックを特定・破棄する技術。uRPF Loose モードは S/RTBH (Source-based RTBH) の前提条件となることがあります。
*   **IPv6 Security**: IPv6 環境における First Hop Security の一環。

---

## 🧩 比較表

### uRPF Strict vs Loose

| 特徴 | Strict Mode (`rx`) | Loose Mode (`any`) |
| :--- | :--- | :--- |
| **チェック条件** | **FIB にルートがある** ＋ **受信 IF が最適パスである** | **FIB にルートがある** (インターフェイス不問) |
| **セキュリティ強度** | 高い | 中程度 |
| **非対称パスの影響** | 通信断が発生する | 影響を受けない |
| **主な用途** | キャンパス、シングルホームのエッジ | マルチホームのエッジ、コアネットワーク |
| **allow-default** | 推奨 (ISP 接続時) | 使用可能 |

---

## 💡 ベストプラクティス

1.  **エッジでの Strict モード**: 送信元 IP が明確なキャンパスエッジやシングルホームの接続点では Strict を使用します。
2.  **マルチホーム環境での Loose モード**: 複数の ISP を持つ場合は、誤検知を避けるため Loose を使用します。
3.  **ロギングの活用**: 導入初期は ACL で許可してログのみを取得し、正当なトラフィックが落ちていないか確認します。
4.  **Null0 ルートの活用**: 攻撃者 IP を `route Null0` に設定することで、uRPF Loose モード環境下でも即座にパケットを破棄できます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な Strict uRPF
*   **問題**: Gi0/1 を通過する全てのトラフィックに対し、送信元 IP 偽装を防止せよ。
*   **要件**: 最も強力な検証モードを使用すること。
*   **設定**: 
    ```bash
    interface GigabitEthernet0/1
     ip verify unicast source reachable-via rx
    ```

### 2. 非対称パスを考慮した uRPF
*   **要件**: ルータ R1 は 2 つの ISP と接続している。パケットの戻りパスが受信 IF と異なる可能性があるため、スプーフィング対策を行いつつ通信断を避けよ。
*   **設定**:
    ```bash
    interface range Gi0/1 - 2
     ip verify unicast source reachable-via any
    ```

### 3. デフォルトルートを考慮した設定
*   **要件**: ISP 接続インターフェイス Gi0/0 で uRPF を有効にせよ。ルータは ISP からデフォルトルートのみを学習している。
*   **設定**:
    ```bash
    interface GigabitEthernet0/0
     ip verify unicast source reachable-via rx allow-default
    ```

### 4. uRPF ドロップのロギング
*   **要件**: uRPF でドロップされたパケットの送信元を Syslog で確認できるようにせよ。
*   **設定**:
    ```bash
    ip access-list extended ACL-LOG-URPF
     deny ip any any log
    !
    interface Gi0/1
     ip verify unicast source reachable-via rx ACL-LOG-URPF
    ```

### 5. IPv6 Strict モードの実装
*   **要件**: IPv6 トラフィックに対しても送信元検証を強制せよ。
*   **設定**:
    ```bash
    interface Gi0/1
     ipv6 verify unicast source reachable-via rx
    ```

### 6. VRF 環境での uRPF
*   **要件**: VRF "CUSTOMER-A" 内のパケットを、その VRF のルーティングテーブルに基づいて検証せよ。
*   **設定**:
    ```bash
    interface Gi0/1
     ip vrf forwarding CUSTOMER-A
     ip verify unicast source reachable-via rx
    ```

### 7. 特定ネットワークの除外 (Exception)
*   **要件**: 192.168.100.0/24 からのトラフィックは uRPF チェックを免除せよ。
*   **設定**:
    ```bash
    access-list 10 permit 192.168.100.0 0.0.0.255
    interface Gi0/1
     ip verify unicast source reachable-via rx 10
    ```

### 8. ASA での uRPF 設定
*   **要件**: ASA の outside インターフェイスで送信元検証を有効にせよ。
*   **設定**:
    ```bash
    # ASA CLI
    ip verify reverse-path interface outside
    ```

### 9. CEF 無効時の挙動確認 (検証用)
*   **課題**: `no ip cef` を実行した後、`show ip interface` で uRPF の状態がどう変わるか確認せよ。

### 10. Self-Ping の許可
*   **要件**: ルータ自身が送信するパケットへの応答が uRPF で落ちないようにせよ。
*   **設定**: `ip verify unicast source reachable-via rx allow-self-ping`。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: インターフェイスに `ip verify unicast source reachable-via rx` が設定されているが、インターネットからのパケットが全てドロップされている。原因として考えられる最も可能性の高いものは？
    *   **回答**: ルータがデフォルトルートしか持っておらず、`allow-default` オプションが欠落しているため。
2.  **トラブルシュート**: マルチホーム環境で Strict モードを設定したところ、一部の顧客から「通信が不安定になった」と連絡があった。どのコマンドでドロップ数を確認すべきか？
    *   **回答**: `show ip interface [interface] | include verify`。
3.  **Design**: Asymmetric routing (非対称ルーティング) が発生する環境において、スプーフィング対策を維持しつつ正当なパケットを通過させるための設定は？
    *   **回答**: uRPF **Loose モード** (`reachable-via any`) を使用する。
4.  **実装**: ドロップされたパケットを詳細に調査したい場合、設定に追加すべき要素は何か？
    *   **回答**: `deny ip any any log` を含む ACL を作成し、`ip verify ... [ACL_NUMBER]` として適用する。
5.  **Design**: uRPF を有効化する際に必ずグローバルで有効化しておくべき機能は？
    *   **回答**: **CEF (Cisco Express Forwarding)**。

---

## 🔗 参考リソース

*   [Cisco IOS-XE Security Configuration Guide: Unicast Reverse Path Forwarding](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_urpf/configuration/xe-16/sec-data-urpf-xe-16-book/sec-unicast-rpf.html)
*   [Understanding Unicast Reverse Path Forwarding (White Paper)](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/17424-unirpf.html)
*   [Cisco ASA 9.4 Configuration Guide: Configuring uRPF](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config-guides/firewall/asa-94-firewall-config/access-urpf.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: uRPF は「パケットの入り口チェック」です。ホテルの受付で「予約リスト（FIB）」を確認し、さらに「正面玄関（正しい IF）」から入ってきたかを見るのが Strict、「どこからでもいいから名前があれば OK」とするのが Loose と覚えると分かりやすいです。
*   **図解**: 
    1.  パケット受信
    2.  ソース IP 抽出
    3.  FIB 検索
    4.  IF 比較 (Strict の場合)
    5.  Pass/Fail
*   **注意点**: ラボ試験では、コマンド入力後に `show ip interface` でカウンタが `0` のままでないか、実際に攻撃（偽装パケット）をシミュレートした際にカウントアップされるかを必ず確認してください。
