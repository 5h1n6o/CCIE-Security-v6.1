---
layout: default
title: 5.4.b-DNS-security
nav_order: 2
parent: 5.4-Cloud-security
grand_parent: 5.0-Advanced-Threat-Protection
---

# 5.4.b DNS security policies in Cisco Umbrella

**Cisco Umbrella** は、クラウド配信型のネットワークセキュリティプラットフォームであり、DNS（Domain Name System）レイヤで最初の防御ラインを提供します。**DNS Security Policies** は、特定の「Identity（アイデンティティ）」に対して、どのドメインへのアクセスを許可し、何をブロックするかを定義する Umbrella の中核的な設定項目です。

---

## 📘 概要

*   **機能概要**: DNS クエリを再帰的 DNS サーバ（Umbrella クラウド）で受信した際、そのドメインのレピュテーションやカテゴリに基づき、IP アドレスを返す（許可）か、ブロックページの IP を返す（遮断）かを判断するポリシーです。
*   **利用目的**: マルウェア感染サイト、フィッシングサイト、C&C サーバへの通信を IP 接続が確立される前に阻止し、ネットワーク全体のセキュリティを向上させます。
*   **どのような場面で利用するか**:
    *   **社内ネットワーク**: ゲートウェイの DNS 設定を Umbrella に向けて一括保護。
    *   **モバイル/リモートワーク**: AnyConnect Roaming Module を通じて、VPN 接続外でも一貫したポリシーを適用。
    *   **ゲスト Wi-Fi**: 不適切なコンテンツ（アダルト、ギャンブル等）のフィルタリング。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **判定基準** | 送信元 Identity (Network, VA, Roaming Client 等)。 |
| **セキュリティ設定** | Malware, Phishing, Command and Control, Botnet 等のカテゴリ別制御。 |
| **コンテンツフィルタ** | 80以上のカテゴリ（SNS, ギャンブル等）に基づくアクセス制限。 |
| **インテリジェントプロキシ** | リスクのある（グレーな）ドメインのみ SSL 復号・詳細検査を実行。 |
| **ポリシー優先順位** | 上から順に評価される「First Match（最初の一致）」方式。 |
| **ホワイト/ブラックリスト** | 特定のドメインを常に許可（Allow）または拒否（Block）する例外設定。 |

---

## 🏗 動作原理

Umbrella DNS ポリシーは、DNS 解決のプロセスに介在することで動作します。

```text
Internal Client
   ↓ (1) DNS Query: "evil-domain.com"
Cisco Umbrella Cloud (208.67.222.222)
   ↓ (2) Identify Source (Identity)
   ↓ (3) Match Policy (Is it Malicious? / Is it Blocked Category?)
   ↓ (4) Decision
   ├── [Permit] --> Return Actual IP
   └── [Block]  --> Return Umbrella Block Page IP (146.112.61.106)
```

---

## ⚙ 動作シーケンス

1.  **Identity の特定**: クエリの送信元（パブリック IP、または VA からのメタデータ）から、どの Identity に属するかを特定します。
2.  **セキュリティカテゴリの評価**: Talos インテリジェンスに基づき、ドメインが悪意あるものか（C&C, Malware 等）をチェックします。
3.  **コンテンツカテゴリの評価**: 管理者が設定したフィルタ（例：動画サイト禁止）に合致するかを確認します。
4.  **Allow/Block リストの適用**: 個別に定義されたドメインリストとの照合を行います。
5.  **レスポンス生成**: 遮断対象であればブロックページの IP アドレスをクライアントに応答します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Identity の紐付け**: 特定のネットワークや端末（Roaming Client）が Umbrella ダッシュボード上で正しく認識されていることを確認するのが第一歩です。
*   **ポリシーの優先度順位**: 試験要件で「特定のグループは SNS 許可、その他は禁止」といった場合、優先度の高い（上位の）ポリシーに許可ルールを配置する必要があります。
*   **Internal Domains の除外**: バーチャルアプライアンス (VA) を使用する場合、内部ドメインをポリシーからバイパス設定しないと、内部リソースへの名前解決が失敗します。
*   **インテリジェントプロキシの有効化**: 特定の「グレーな」サイトに対してのみ詳細なファイルスキャン（AMP）を実行する設定が問われる可能性があります。
*   **トラブルシュート**: クライアント側で `nslookup -type=txt debug.opendns.com` を実行し、どのポリシー（policy ID）が適用されているかを確認する能力が求められます。

---

## 🛠 設定方法

### 1. Identity の登録 (Umbrella Dashboard)
*   **Deployments > Core Identities > Networks**: グローバル IP を登録。
*   **Deployments > Core Identities > Roaming Computers**: AnyConnect 端末を確認。

### 2. DNS Security Policy の作成
1.  **Policies > DNS Policies** に移動し **Add**。
2.  **Identity 選択**: 保護対象（例：全 Roaming Client）を選択。
3.  **セキュリティ設定**: `Malware`, `Phishing`, `C&C` などの保護レベルをオンにする。
4.  **コンテンツカテゴリ**: 特定のカテゴリ（例：Gambling）をブロック。
5.  **Allow/Block リスト**: 独自の除外ドメインを適用。
6.  **保存**: ポリシー名を付けて保存し、最上位に配置。

---

## 🔍 検証コマンド

| 目的 | デバイス | コマンド / 手法 |
| :--- | :--- | :--- |
| **Umbrella 接続状態確認** | Client | <code>nslookup -q=txt debug.opendns.com</code> |
| **適用ポリシー ID 確認** | Client | 上記出力の `policy` フィールドを確認。 |
| **DNS 解決の強制テスト** | Client | <code>nslookup welcome.opendns.com</code> |
| **ブロックページの確認** | Browser | `http://examplemalwaredomain.com` にアクセスし、遮断画面が出るか確認。 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| ポリシーが適用されない | DNS が Umbrella に向いていない | <code>ipconfig /all</code> で DNS サーバが 208.67.222.222 か確認。 |
| ブロックされるべきサイトが通る | ポリシー優先順位の誤り | ダッシュボードで Default ポリシーより上に設定されているか確認。 |
| 内部サイトが解決できない | Internal Domain 設定漏れ | VA または Roaming Client の設定で内部ドメインをバイパスリストに追加。 |
| Identities が "Offline" | 通信遮断 | FW で UDP/53 (DNS) および TCP/443 (HTTPS) のアウトバウンドを許可。 |

---

## ⚠ 制限事項

*   **DNS オーバー HTTPS (DoH)**: ブラウザが独自の DoH 設定（Google や Cloudflare 等）を使用している場合、Umbrella の DNS 制御をバイパスしてしまうため、グループポリシー等でブラウザ設定を固定する必要があります。
*   **IP 直接通信**: Umbrella DNS はドメイン名に基づいた制御であるため、悪意ある IP アドレスへ直接通信する（ドメインを介さない）トラフィックは、SIG（Secure Internet Gateway）機能なしでは阻止できません。

---

## 🔄 他技術との関連

*   **4.14 Identity mapping**: Cisco ISE と pxGrid 連携することで、ISE が持つ詳細なユーザ情報を Umbrella の Identity として使用可能です。
*   **5.1 Cisco AMP**: インテリジェントプロキシ経由のトラフィックに対し、AMP（Malware Defense）によるファイル解析を実行します。
*   **1.3.c NAT (Cisco IOS/FTD)**: 内部 IP を隠蔽する NAT 環境では、Umbrella VA を配置することで内部ホストの可視化を維持します。

---

## 🧩 比較表

### DNS Policy vs Web Policy (SIG)

| 特徴 | DNS Policy | Web Policy (SWG/SIG) |
| :--- | :--- | :--- |
| **検査レイヤ** | DNS (UDP 53) | HTTP/HTTPS (Proxy) |
| **速度** | 極めて高速（オーバーヘッドなし） | 若干の遅延あり（SSL 復号等） |
| **制御単位** | ドメイン単位 | URL パス単位 / ファイル単位 |
| **必要なもの** | DNS 設定変更のみ | トンネル(IPsec) または PAC ファイル |

---

## 💡 ベストプラクティス

1.  **階層化アプローチ**: まず DNS ポリシーで広範囲の脅威を低コストで排除し、残った不透明なトラフィックを Web ポリシー（SIG）で詳細検査する。
2.  **Security Over Content**: セキュリティカテゴリ（Malware 等）のブロックは常に「オン」にし、コンテンツフィルタ（娯楽等）は組織のポリシーに応じて柔軟に運用する。
3.  **テストドメインの活用**: 構築後は必ず `welcome.opendns.com` や `internetbadguys.com` (テスト用) を使用して、意図した動作をしているか確認する。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的なマルウェアブロック
*   **要件**: 全ネットワーク Identity に対して、Malware と Phishing をブロックせよ。

### 2. コンテンツカテゴリ制限
*   **要件**: 「Marketing」グループの Roaming Client に対し、SNS カテゴリのみをブロックせよ。

### 3. ブラックリスト（Destination List）の適用
*   **要件**: `unwanted.com` を全社員に対して強制的にブロックせよ。

### 4. インテリジェントプロキシの構成
*   **要件**: レピュテーションが「未知」のサイトへのアクセス時のみ SSL 復号を行い、AMP スキャンを実行せよ。

### 5. 内部ドメインのバイパス (Internal Domains)
*   **要件**: 内部ドメイン `corp.local` へのクエリは Umbrella クラウドへ転送せず、ローカル DNS で解決させよ。

### 6. カスタムブロックページの作成
*   **要件**: 遮断時に会社のロゴと、IT ヘルプデスクへの連絡先を表示させよ。

### 7. スケジュールベースのポリシー
*   **要件**: 勤務時間外（18時以降）は YouTube カテゴリのブロックを解除せよ。

### 8. AnyConnect Umbrella Roaming のデプロイ
*   **要件**: ASAv から `OrgInfo.json` を AnyConnect クライアントへ配布し、自動的に Identity 登録を完了させよ。

### 9. DNS キャッシュの影響確認
*   **操作**: クライアントで `ipconfig /flushdns` を実行し、ポリシー変更が即座に反映されるか検証せよ。

### 10. レポートによる検知確認
*   **操作**: Umbrella Dashboard の **Activity Search** を開き、ブロックされたドメインのログから送信元 Identity を特定せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: クライアントが Umbrella サーバを使用しているが、特定のドメインがブロックされない。ポリシー設定を確認したところ、該当ドメインを含む Block List が適用されている。他に確認すべき点は？
    *   **回答**: **ポリシーの優先順位**。上位のポリシーでその Identity に対して「Allow All」などが設定されていないか、または「Bypass Code」が使用されていないか。
2.  **Design**: NAT を多用している環境で、特定の内部ホストがどの悪意あるドメインを叩いたかを特定したい。どの Umbrella コンポーネントが必要か？
    *   **回答**: **Umbrella Virtual Appliance (VA)**。VA は EDNS0 を使用して内部 IP 情報を Umbrella クラウドに伝搬します。
3.  **トラブルシュート**: AnyConnect Roaming Client をインストールしたが、Umbrella Dashboard に表示されない。FW で許可すべきポートは？
    *   **回答**: **UDP 53 (DNS)** および **UDP 443 (Do53)**、および API 同期のための **TCP 443**。
4.  **Design**: SSL 復号を全トラフィックに行うと遅延が懸念される。特定の危険なサイトのみ復号して検査する機能は？
    *   **回答**: **Intelligent Proxy**。
5.  **実装**: 内部の AD ドメイン解決ができなくなった。Umbrella でどこを修正すべきか？
    *   **回答**: Umbrella ダッシュボードの **Deployments > Settings > Internal Domains** に AD ドメインを登録する。

---

## 🔗 参考リソース

*   **Cisco Umbrella Configuration Guide**: [DNS Policies](https://docs.umbrella.com/deployment-umbrella/docs/dns-policies)
*   **Cisco Live (BRKSEC-2041)**: [Umbrella Architecture and Best Practices](https://www.ciscolive.com/)
*   **Cisco Validated Design (CVD)**: [Secure Internet Gateway with Cisco Umbrella](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/umbrella-deployment-guide.html)
*   **Official Cert Guide (SCOR 350-701)**: [Chapter 12: Cloud Security](https://www.ciscopress.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「DNS ポリシーは電話帳のフィルタリング」と考えると分かりやすいです。悪い人の電話番号（IP）を教えないことで、そもそも通話をさせない仕組みです。
*   **注意点**: ラボ試験では、**ブラウザのキャッシュ**によって「ブロック設定したのにサイトが開けてしまう」という誤認が起きやすいため、検証時はシークレットモードや `ipconfig /flushdns` を活用してください。
