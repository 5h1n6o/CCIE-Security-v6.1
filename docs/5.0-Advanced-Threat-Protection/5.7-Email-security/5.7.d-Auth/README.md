---
layout: default
title: 5.7.d-Auth
nav_order: 4
parent: 5.7-Email-security
grand_parent: 5.0-Advanced-Threat-Protection
---

# 5.7.d Authentication

電子メール認証技術（**SPF**, **DKIM**, **DMARC**）は、Cisco Secure Email (旧 ESA) において送信ドメインの正当性を検証し、なりすまし（Spoofing）やフィッシング攻撃を防ぐための最優先の防御策です。これらの技術は単体ではなく、相互に補完し合うフレームワークとして機能します。

---

## 📘 概要

*   **機能概要**: 送信元が公言しているドメインが、実際にそのドメインの正当な送信者であるかを DNS レコードや電子署名を用いて確認する機能です。
*   **利用目的**: 自社ドメインを騙ったなりすましメールの防止、および外部から届く悪意のあるメール（フィッシング）の検知。
*   **どのような場面で利用するか**:
    *   **受信時 (Verification)**: 外部から届くメールが SPF/DKIM/DMARC に合格しているかを確認し、不合格の場合は隔離や削除を行う。
    *   **送信時 (Signing)**: 自社から送信するメールに DKIM 署名を付与し、受信側のメールサーバで正当な送信元として認識させる。

---

## 🔑 要点

| 技術 | 確認方法 | ESA での役割 |
| :--- | :--- | :--- |
| **SPF** | 送信元 IP アドレスの検証 | DNS TXT レコードに登録された IP と一致するか確認。 |
| **DKIM** | 電子署名による整合性検証 | 公開鍵（DNS）と秘密鍵（ESA）を用いてヘッダーと本文の改ざんを確認。 |
| **DMARC** | 統合ポリシーの強制 | SPF/DKIM の結果に基づき、最終的なアクション（何もしない・隔離・拒否）を決定。 |
| **Verification** | 受信フィルタリング | **Mail Flow Policies** に紐付いた Verification プロファイルで実行。 |
| **Signing** | 送信認証 | 送信メールに対して DKIM プロファイルを使用して署名を実行。 |

---

## 🏗 動作原理

各認証技術は、SMTP セッションの異なる段階および DNS 問い合わせによって動作します。

```text
[ Sender MTA ] --(1) SMTP Connection--> [ Cisco Secure Email (ESA) ]
                                              ↓
[ ESA Authentication Pipeline ]
   ├── (2) SPF: Check Sender IP vs DNS TXT record
   ├── (3) DKIM: Decrypt Header Signature vs DNS Public Key
   └── (4) DMARC: Evaluate results from SPF & DKIM based on DMARC Policy
                                              ↓
[ Policy Enforcement ]
      ├── Pass: Normal Delivery
      └── Fail: Quarantine, Drop, or Tag (based on profile)
```

---

## ⚙ 動作シーケンス

1.  **接続受信**: ESA が外部 MTA からの接続を受信します。
2.  **SPF 検証**: `MAIL FROM` アドレスのドメインの DNS を引き、接続元 IP が許可リストにあるか確認します。
3.  **DKIM 検証**: メールヘッダーから `DKIM-Signature` を抽出し、セレクタ情報を基に DNS から公開鍵を取得、署名を検証します。
4.  **DMARC 評価**: SPF および DKIM の結果（およびドメインの整合性 = Alignment）を DMARC ポリシーと照合します。
5.  **認証ヘッダーの挿入**: ESA は `Authentication-Results` ヘッダーをメールに追加し、後続のフィルタ（Content Filters 等）で利用可能にします。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **DNS 依存性**: ラボ試験環境では、ESA が参照する DNS サーバが適切な SPF/DKIM/DMARC レコードを返せるように構成されている必要があります。
*   **Mail Flow Policy への適用**: 認証設定（Verification）は、**Mail Flow Policy**（特に ACCEPT されるポリシー）に関連付けられたプロファイル内で有効にする必要があります。
*   **Signing と Verification の混同回避**: 自社ドメイン宛（Incoming）には検証を、外部宛（Outgoing）には署名（Signing）を設定することを明確に区別してください。
*   **DMARC アライメントの理解**: SPF で成功しても、エンベロープ From とヘッダー From が異なる場合、DMARC では「不一致（Failed）」とみなされる可能性があります。
*   **トラブルシュート**: `mail_logs` で認証結果（`SPF: Pass`, `DKIM: Neutral`, `DMARC: Failed` 等）を正確に読み取る能力が必須です。

---

## 🛠 設定方法

### 1. SPF/DKIM/DMARC 検証の有効化 (Incoming)
1.  **Mail Policies > Verification Profiles** で新規プロファイルを作成。
2.  `SPF`, `DKIM`, `DMARC` の各チェックボックスをオンにする。
3.  **Mail Policies > Mail Flow Policies** に移動。
4.  対象の Sender Group（通常は `UNKNOWNLIST` 等）のポリシー設定で、作成した **Verification Profile** を選択する。

### 2. DKIM 送信署名の設定 (Outgoing)
1.  **Mail Policies > DKIM > Global Settings** で DKIM 署名を有効化。
2.  **Mail Policies > DKIM > Signing Keys** で RSA 鍵ペアを作成またはインポート。
3.  **Signing Profiles** を作成し、鍵とドメイン名を紐付ける。
4.  **Outgoing Mail Policies** で該当ポリシーに対し、作成した Signing Profile を適用する。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **リアルタイムログ監視** | <code>tail mail_logs</code> (認証結果キーワードを検索) |
| **DNS レコードの確認** | <code>dig txt [domain]</code> または <code>nslookup -q=txt [domain]</code> |
| **ESA からの DNS 解決確認** | <code>dnslookup [domain]</code> |
| **現在の署名鍵確認** | <code>dkimkeys</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 認証結果が常に `Neutral` | DNS 問い合わせ失敗 | ESA のネットワーク設定と DNS 接続を確認。 |
| DKIM 署名が付与されない | Profile 適用漏れ | Outgoing Mail Policy で Signing Profile が選択されているか確認。 |
| DMARC でメールが拒否される | アライメント不一致 | 送信元ドメインの一貫性を確認し、DMARC ポリシーを一時的に `none` に変更。 |
| `dkim-signature` が壊れている | 途中のデバイスでの改ざん | 署名後に別のプロキシ等がヘッダーを書き換えていないか確認。 |

---

## ⚠ 制限事項

*   **転送メールの影響**: メーリングリスト等の転送サービスを経由すると、送信 IP が変わるため SPF が失敗し、本文の書き換えで DKIM が失敗する傾向があります。
*   **鍵サイズ**: 古い受信サーバでは 2048bit の DKIM 鍵に対応していない場合があり、互換性のために 1024bit を選択する設計が必要な場合があります。
*   **パフォーマンス**: 大量のメールに対して詳細な DMARC 検証とレポート生成を行うと、ESA のリソース（CPU/Disk）を消費します。

---

## 🔄 他技術との関連

*   **5.7.a Mail Policies**: 認証結果に基づいたアクション（隔離等）を Mail Policy のフィルタで定義します。
*   **3.6 Monitoring**: DMARC の集計レポート（Aggregate Report）を外部へ送信する設定が必要です。
*   **System Hardening**: ESA 自体の時刻同期（NTP）が正確でないと、DKIM 署名の有効期限判定に失敗します。

---

## 🧩 比較表

### SPF vs DKIM vs DMARC

| 比較項目 | SPF | DKIM | DMARC |
| :--- | :--- | :--- | :--- |
| **検証対象** | 送信元サーバの IP | メールの電子署名 | SPF/DKIM の統合結果 |
| **主な弱点** | 転送に弱い | 署名鍵の管理コスト | SPF/DKIM 両方の設定が前提 |
| **DNS 種類** | TXT レコード | セレクタ用 TXT | `_dmarc` 用 TXT |
| **最高のアクション** | 通信遮断 | スパムスコア加算 | ポリシーによる強制排除 |

---

## 💡 ベストプラクティス

1.  **段階的な DMARC 導入**: 最初は `p=none`（監視のみ）から始め、レポートを分析して誤検知がないことを確認してから `p=quarantine` または `p=reject` へ移行します。
2.  **セレクタのローテーション**: セキュリティ維持のため、DKIM 署名鍵（セレクタ）は定期的に更新することを推奨します。
3.  **Authentication-Results の活用**: 認証結果をヘッダーに埋め込み、下流の Microsoft 365 や Exchange でもその結果を再利用できる設計にします。

---

## 📝 ラボ学習・設定サンプル例

### 1. 外部ドメインの SPF 検証有効化
*   **要件**: 全ての外部受信メールに対し SPF 検証を行い、Fail の場合はヘッダーに `X-SPF-Status: Failed` を追加せよ。

### 2. DKIM 署名鍵の作成 (CLI)
*   **操作**: <code>dkimkeys</code> コマンドを使用して、`ccie_key` という名前で 1024bit 鍵を作成せよ。

### 3. DMARC 隔離ポリシーの実装
*   **要件**: 自社ドメイン宛のメールで DMARC 不合格のものを `Policy Quarantine` へ隔離せよ。

### 4. SPF リストへの IP 追加
*   **要件**: 本社拠点のグローバル IP (203.0.113.10) を許可するように既存の SPF レコードを修正せよ。

### 5. 複数セレクタの構成
*   **要件**: マーケティング部門用と全社用で異なる DKIM セレクタ (`mktg` / `corp`) を使い分けよ。

### 6. DKIM 検証不合格時のタグ付け
*   **要件**: DKIM 検証に失敗したメールの件名に `[UNTRUSTED]` を付与せよ。

### 7. DMARC 集計レポートの送信設定
*   **要件**: 毎日 0 時に DMARC レポートを `postmaster@example.com` へ送信するよう構成せよ。

### 8. Mail Flow Policy への Profile 適用
*   **要件**: `RELAYLIST` 以外の全ての Sender Group に対して、検証プロファイル `Strict_Check` を適用せよ。

### 9. 内部 DNS を使用した検証テスト
*   **操作**: ラボ内の DNS サーバにテスト用 TXT レコードを登録し、ESA で <code>dnslookup</code> が成功することを確認せよ。

### 10. レポートによる認証統計の確認
*   **操作**: ESA GUI の **Monitor > Mail Protocol > SPF/DKIM/DMARC** 画面で、拒否されたメールの推移を特定せよ。

---

## ❓ 想定試験問題

1.  **Design**: 外部送信メールが「なりすまし」と判定されるのを防ぎたいが、送信元 IP が頻繁に変わるクラウドサービスを利用している。最適な認証技術は？
    *   **回答**: **DKIM**。IP アドレスに依存せず、電子署名によって正当性を証明できるため。
2.  **コンフィグ読解**: `mail_logs` に `SPF: Hard Fail` と記録されている。送信側ドメインの DNS レコードには `~all` (Soft Fail) と記載されている。なぜか？
    *   **回答**: ESA の **Verification Profile** において、送信側のポリシーに関わらず `Fail` を `Reject` などの厳しいアクションとして扱うように設定されている。
3.  **トラブルシュート**: DKIM 署名を設定したが、受信側で `Signature Invalid` となる。最も疑わしい原因は？
    *   **回答**: ESA で署名した後に、**下流のデバイス（Firewall の SMTP インスペクション等）がメールヘッダーや本文の一部を書き換えている**。
4.  **Design**: DMARC を導入する際、最初に設定すべきポリシーレベル（p=タグ）は？
    *   **回答**: **p=none**。正規のメールを遮断するリスクを避けるため、まずは監視とレポート収集から開始する。
5.  **実装**: ESA で特定のドメインだけ認証検証をスキップさせたい。どこで設定すべきか？
    *   **回答**: **Sender Group** にそのドメインの IP またはホスト名を登録し、関連付けられた **Mail Flow Policy** で認証プロファイルを `None` に設定する。

---

## 🔗 参考リソース

*   **Cisco Secure Email Admin Guide**: [Email Authentication Setup](https://www.cisco.com/c/en/us/td/docs/security/esa/esa11-1/user_guide/b_ESA_Admin_Guide_11_1.html)
*   **Cisco Live (BRKSEC-2041)**: [Email Security Best Practices for Authentication](https://www.ciscolive.com/)
*   **Technical Note**: [Troubleshooting SPF/DKIM/DMARC on ESA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/200632-Troubleshooting-SPF-DKIM-and-DMARC-in.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「SPF は免許証（提示されたもの）、DKIM は実印（中身の証明）、DMARC はそれらをどう扱うかの法律（ルール）」と覚えると整理しやすいです。
*   **注意点**: ラボ試験では、**DNS の TXT レコードのクォーテーション（""）**の付け忘れや、セレクタ名のスペルミスが原因で検証に失敗することが多いため、慎重に入力してください。
