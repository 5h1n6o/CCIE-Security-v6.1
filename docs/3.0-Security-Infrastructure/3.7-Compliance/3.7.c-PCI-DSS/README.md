---
layout: default
title: 3.7.c-PCI-DSS
nav_order: 3
parent: 3.7-Compliance
grand_parent: 3.0-Security-Infrastructure
---

# 3.7.c PCI-DSS (Payment Card Industry Data Security Standard)

**PCI-DSS** は、クレジットカード情報を安全に取り扱うために定められた国際的なセキュリティ基準です。CCIE Security v6.1 のラボ試験においては、特定の「PCI-DSSコマンド」を打つのではなく、ブループリントに含まれる様々な技術（Firewall, AAA, Encryption, Logging）を組み合わせて、**「PCI-DSSの要件を満たすセキュアなインフラを構築・維持できるか」**が問われます。

---

## 📘 概要

*   **機能概要**: 12の主要要件からなるセキュリティ基準であり、ネットワークの観点では「安全なネットワークの構築」「カード保持者データの保護」「脆弱性管理プログラムの維持」「強力なアクセス制御」が中心となります。
*   **利用目的**: クレジットカード情報の漏洩を防ぎ、組織の法的・経済的リスクを低減すること。
*   **どのような場面で利用するか**: 支払い処理が発生するネットワークセグメント（CDE: Cardholder Data Environment）の設計、実装、および監査対応において適用されます。

---

## 🔑 要点

PCI-DSS の主要要件と Cisco 技術のマッピング：

| PCI要件 | 内容 | 該当する Cisco 技術 |
| :--- | :--- | :--- |
| **要件 1** | ファイアウォールの構成維持 | ASA/FTD Access Control Policy, ACLs |
| **要件 2** | デフォルト値（パスワード等）の変更 | `username ... secret`, `no ip http server` |
| **要件 4** | 公衆ネットワークでの暗号化 | IPsec VPN, TLS (AnyConnect), AES |
| **要件 7** | 業務上の「知る必要性」による制限 | Cisco ISE, TrustSec (SGT), VRF-Lite |
| **要件 8** | 識別と認証（個別のID付与） | AAA (TACACS+/RADIUS), MFA |
| **要件 10** | ログの追跡と監視 | Syslog, SNMPv3, NetFlow, eStreamer |

---

## 🏗 動作原理

PCI-DSS 準拠のネットワーク設計は、**「スコープの最小化（Segmentation）」**と**「多層防御（Defense in Depth）」**の原則に基づきます。

```text
[ Internet ]
      ↓
[ Edge Firewall (FTD/ASA) ] --- DMZ (Public Facing)
      ↓
[ Core Switch (Segmentation) ] --- VLAN/VRF Isolation
      ↓
[ Internal CDE Segment ] ← ここを最優先で保護
   (Strict ACLs, 802.1X, Logging, Encryption)
```

1.  **分離 (Segmentation)**: CDE（カードデータ環境）を他の一般業務ネットワークから VLAN や VRF で物理的・論理的に分離します。
2.  **最小権限 (Least Privilege)**: 許可された通信（特定のポート・プロトコル）以外はデフォルト拒否（Implicit Deny）にします。
3.  **可視化 (Visibility)**: すべてのアクセスをログに記録し、NTP で時刻同期された正確な証跡を管理します。

---

## ⚙ 動作シーケンス（コンプライアンス維持フロー）

1.  **Hardening**: デバイスの不要なサービスを停止し、パスワードの複雑性を設定。
2.  **Access Control**: 管理アクセスを特定のホスト（Admin PC）からのみ、かつ SSH に限定。
3.  **Authentication**: 管理者個人の ID でログインさせ、実行コマンドをすべて Accounting ログに記録。
4.  **Monitoring**: トラフィックフローを NetFlow で監視し、不審な通信（要件11）を検知。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **要件の読み取り**: 「PCI-DSSに準拠せよ」という指示があった場合、以下の暗黙的な設定が期待されます。
    *   **パスワード暗号化**: `service password-encryption`
    *   **ログイン制限**: `login block-for`（ブルートフォース対策）
    *   **バナー表示**: 認可されていないアクセスを禁ずる旨の `banner login`
*   **ログの正確性**: NTP 設定がない環境でのロギングは PCI 違反とみなされる可能性があります。必ず `service timestamps log datetime msec` を含めます。
*   **暗号化の強度**: IKEv2/IPsec において 3DES や DES は PCI-DSS では非推奨（または禁止）です。必ず **AES (CCMP/GCM)** を使用してください。
*   **管理通信の要塞化**: Telnet 接続を許可しているコンフィグを SSH に修正する問題が想定されます。

---

## 🛠 設定方法

### 1. 管理プレーンの要塞化 (要件 2 & 8)
```bash
! 最小パスワード長とブルートフォース対策
security passwords min-length 12
login block-for 300 attempts 3 within 60

! AAA による個別管理と記録
aaa new-model
aaa authentication login default group tacacs+ local
aaa accounting commands 15 default start-stop group tacacs+
```

### 2. データプレーンの暗号化と分離 (要件 1 & 4)
```bash
! CDE (VLAN 10) からのトラフィックを暗号化
crypto ikev2 proposal PCI-PROP
 encryption aes-gcm-256
 group 14
```

### 3. ログの証拠能力確保 (要件 10)
```bash
! ミリ秒単位の正確な時刻同期とログ
ntp server 192.168.1.100
service timestamps log datetime msec
logging host 10.1.1.50
logging trap informational
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **不要なポートの確認** | <code>show control-plane host open-ports</code> |
| **ログ出力設定の確認** | <code>show logging</code> |
| **AAA セッションの確認** | <code>show aaa sessions</code> |
| **暗号化アルゴリズムの確認** | <code>show crypto ipsec sa</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 脆弱性スキャンで Telnet 検出 | 設定漏れ | <code>transport input ssh</code> が vty に設定されているか。 |
| ログの時刻が不正確 | NTP 未同期 | <code>show ntp status</code> で同期状態を確認。 |
| 管理者が共有 ID を使用中 | 要件 8 違反 | <code>aaa authentication</code> を外部サーバ (ISE) 連携に変更。 |
| CDE 間の通信が丸見え | 分離不足 | <code>show vlan</code> または <code>show ip route vrf</code> でセグメント確認。 |

---

## ⚠ 制限事項

*   **後方互換性**: 古い端末が強力な暗号化（AES-GCM）をサポートしていない場合、PCI 準拠と業務継続のトレードオフが発生します。
*   **パフォーマンス**: 大量のアカウティング（accounting commands 15）は、TACACS+ サーバおよびデバイスの CPU に負荷をかけます。

---

## 🔄 他技術との関連

*   **3.1.a CoPP**: 要件 1（可用性保護）の一部。
*   **3.6.a NetFlow**: 要件 10 & 11（継続的な監視）の主要ツール。
*   **3.1.c iACLs**: ネットワーク機器自体の保護。

---

## 🧩 比較表

### PCI-DSS v3.2.1 vs v4.0 (CCIE v6.1 視点)

| 特徴 | v3.2.1 (従来) | v4.0 (最新) |
| :--- | :--- | :--- |
| **パスワード** | 7文字以上、90日変更 | 12文字以上が推奨、MFA の強化 |
| **MFA** | CDE へのリモートアクセスのみ | **すべての CDE へのアクセスに必須** |
| **暗号化** | TLS 1.1 禁止、1.2 必須 | **TLS 1.3 推奨** |

---

## 💡 ベストプラクティス

1.  **Management Plane Protection (MPP)**: 管理トラフィックを特定のインターフェイス（Management port）に限定します。
2.  **デフォルト拒否の原則**: Firewall Policy の末尾には必ず `deny any any log` を配置します。
3.  **ISE との統合**: 認証だけでなく、ポスチャ（端末の状態）チェックを行い、コンプライアンスに適合しない端末を CDE から排除します。

---

## 📝 ラボ学習・設定サンプル例

1.  **パスワードポリシー設定**: 全デバイスで最小 10 文字、不正ログイン 3 回で 10 分ロックせよ。
2.  **セキュア管理アクセス**: VTY 0-15 で SSH のみ許可し、ACL で IT 管理セグメントのみに制限せよ。
3.  **AAA Accounting 実装**: 特権モードで行われた全コマンドを ISE に記録せよ。
4.  **時刻同期設定**: 信頼できる 2 台の NTP サーバと同期し、ログに反映せよ。
5.  **不要サービス無効化**: `no ip http server`, `no ip finger`, `no cdp run` (要件による)。
6.  **Firewall Logging**: ACL で拒否されたパケットの送信元 MAC をログに含めよ (`log-input`)。
7.  **SNMPv3 実装**: AES 暗号化を使用した `authPriv` モードで構成せよ。
8.  **NetFlow 収集**: CDE 宛の全通信を Stealthwatch コレクタへ転送せよ。
9.  **VRF による分離**: PCI ゾーンを独立した VRF に収容し、ルーティングレベルで分離せよ。
10. **VPN 強力暗号化**: Site-to-Site VPN で AES-256-CBC 以上の強度を指定せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `logging trap 3` が設定されている。PCI-DSS 要件 10（すべてのアクセス試行の記録）を満たすためにどう修正すべきか？
    *   **回答**: `logging trap 6` (Informational) 以上に変更し、許可・拒否の両方のイベントを記録できるようにする。
2.  **Design**: CDE へのリモートアクセスにおいて、パスワード認証のみでは PCI-DSS 不適合となる。何を追加すべきか？
    *   **回答**: **多要素認証 (MFA)**。AnyConnect と Duo の連携などが Cisco ソリューションの例。
3.  **トラブルシュート**: 管理者が `enable` コマンドで特権昇格しているが、その履歴が TACACS+ サーバに残っていない。
    *   **回答**: `aaa accounting exec` および `aaa accounting commands 15` の設定不足。
4.  **実装**: デバイス上で `service password-encryption` が設定されていない場合のリスクは？
    *   **回答**: `show run` 時にパスワードが平文で見えてしまい、要件 2 (デフォルトおよび安全でない設定の排除) に違反する。
5.  **Design**: セグメント間の ACL で `permit tcp any host 10.1.1.1 eq 443` と設定している。PCI 的にさらに追加すべきオプションは？
    *   **回答**: `log` オプションを追加し、許可されたトラフィックの証跡も残す。

---

## 🔗 参考リソース

*   **Cisco White Paper**: [PCI DSS Compliance on Cisco Infrastructure](https://www.cisco.com/c/en/us/solutions/enterprise/design-zone-security/pci-design-guides.html)
*   **Cisco Live (BRKSEC-2003)**: [Securing the Infrastructure: Hardening](https://www.ciscolive.com/)
*   **CVD**: [Payment Card Industry Data Security Standard (PCI DSS) 3.2.1 Design Guide](https://www.cisco.com/c/dam/en/us/td/docs/solutions/CVD/Aug2018/CVD-PCI-Design-Guide-2018AUG.pdf)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「PCI-DSS はセキュリティのテストの採点基準」と考えてください。
*   **図解**: 
    *   **Management**: 誰が（AAA）、いつ（NTP）、何を（Accounting）。
    *   **Data**: 分離（VLAN/VRF）、暗号（VPN）、フィルタ（ACL）。
*   **注意点**: ラボ試験では `log` オプションの付け忘れが PCI 要件の未達として大きく減点される傾向にあります。
