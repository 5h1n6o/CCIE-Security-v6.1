---
layout: default
title: 5.4.a-DNS-proxy
nav_order: 1
parent: 5.4-Cloud-security
grand_parent: 5.0-Advanced-Threat-Protection
---

# 5.4.a DNS proxy through Cisco Umbrella virtual appliance

Cisco Umbrella **Virtual Appliance (VA)** は、内部ネットワークの DNS クエリを Umbrella クラウドへ中継する「条件付きフォワーダ」として機能する軽量な仮想マシンです。最大の特徴は、NAT 後のパブリック IP しか見えないクラウドに対し、**内部クライアントのプライベート IP アドレスや Active Directory (AD) ユーザ情報**を付加してクエリを送信できる点にあります。

---

## 📘 概要

*   **機能概要**: 内部 DNS サーバとクライアントの間に配置され、DNS クエリに **EDNS0 (Extension Mechanisms for DNS)** プロトコルを使用して ID 情報を埋め込み、Umbrella クラウドへ転送します。
*   **利用目的**: NAT 環境下でも「どの端末が」悪意あるサイトにアクセスしようとしたかを特定し、詳細なポリシー制御とロギングを実現すること。
*   **どのような場面で利用するか**:
    *   内部ネットワークに NAT が存在し、Umbrella ダッシュボードで送信元 IP がすべて同じに見えてしまう場合。
    *   Active Directory と連携し、ユーザやグループ単位で異なる DNS フィルタリングを適用したい場合。
    *   内部ドメインの解決は既存の DNS サーバで行い、外部ドメインのみを Umbrella で保護したい場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な役割** | 条件付き DNS フォワーダ、ID 情報の付加。 |
| **提供形態** | VMware ESXi, Microsoft Hyper-V, AWS/Azure 上の仮想アプライアンス。 |
| **ID 識別** | 内部 IP アドレス、AD ユーザ/コンピュータ名、AD グループ。 |
| **メリット** | エージェントレス（Roaming Client 不要）で内部 ID を可視化。 |
| **デメリット** | 内部インフラへの VM 設置と冗長化設計が必要。 |
| **対応プロトコル** | UDP/53 (DNS), TCP/443 (Umbrella クラウドへの暗号化転送)。 |
| **設計上の注意点** | 内部ドメイン（.local 等）を VA に登録し、内部 DNS へ転送する設定が必須。 |

---

## 🏗 動作原理

VA はクライアントからのクエリを受け取ると、送信元 IP を確認し、AD コネクタから得た情報を基にメタデータを付加します。

```text
Internal Client (10.1.1.50)
   ↓ (Standard DNS Query)
Umbrella Virtual Appliance (DNS Proxy)
   ↓ (Add EDNS0 metadata: Client IP, Site ID)
Umbrella Cloud (208.67.222.222)
   ↓ (Policy Check & Logging)
Enforcement (Permit or Block)
```

---

## ⚙ 動作シーケンス

1.  **クエリ受信**: クライアントが VA（10.1.1.10 等）へ DNS クエリを送信します。
2.  **ドメイン判定**:
    *   **内部ドメイン**: 設定された「Internal Domains」リストに合致する場合、内部 DNS サーバへそのまま転送します。
    *   **外部ドメイン**: 手順 3 へ進みます。
3.  **メタデータ付加**: クライアントのプライベート IP アドレスを含む **EDNS0 レコード**をパケットに挿入します。
4.  **クラウド転送**: Umbrella の Anycast IP へクエリを転送します。この際、通信は暗号化される場合があります。
5.  **レスポンス返却**: Umbrella クラウドからの判定結果（IP またはブロックページ IP）をクライアントに返します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **内部ドメインのバイパス**: `Internal Domains` リストを正しく構成しないと、内部の Active Directory への名前解決に失敗し、ドメイン参加やログインができなくなります。
*   **Active Directory 統合**: VA 単体では IP アドレスしか識別できません。AD ユーザ名を表示させるには、別途 **Umbrella Connector** を AD 上に構成し、VA と同期させる必要があります。
*   **冗長化の構成**: ラボ試験で「単一障害点の排除」が求められた場合、2 台の VA をデプロイし、DHCP オプションで両方の IP を配布する設計が必要です。
*   **ファイアウォールの穴あけ**: VA から Umbrella クラウドへの **UDP/53** および **TCP/443** の許可設定が不可欠です。
*   **検証の重要性**: `nslookup -type=txt debug.opendns.com` コマンドの結果を読み取り、`originid` や `clientip` が表示されているかを確認できる必要があります。

---

## 🛠 設定方法

### 1. 仮想アプライアンスのデプロイ
*   Umbrella ダッシュボードからイメージをダウンロード。
*   OVA を ESXi 等にインポートし、管理 IP (CLI) を設定。

### 2. Umbrella ダッシュボードへの登録
*   VA が起動すると、インターネット経由でダッシュボードに自動的に「Pending」として表示されます。
*   名前を付けて `Sites` に割り当てます。

### 3. 内部ドメインの設定 (Deployments > Settings > Internal Domains)
*   `example.local` などの内部ドメインを登録し、VA が内部 DNS サーバ（ドメインコントローラ等）にクエリを投げるように設定します。

### 4. クライアントの設定
*   DHCP サーバ（またはスイッチ/ルータ）の DNS サーバ設定を、VA の IP アドレスに変更します。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **接続状態の確認** | <code>nslookup -q=txt debug.opendns.com</code> |
| **VA CLI での状態確認** | <code>config va status</code> |
| **内部 DNS への疎通** | <code>config va test-dns</code> |
| **同期状態の確認** | <code>config va test-connectivity</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 内部サイトに繋がらない | Internal Domain 未登録 | <code>nslookup [internal_name]</code> | ダッシュボードの Internal Domains リストに追加。 |
| ダッシュボードで IP しか見えない | AD Connector 連携不全 | Umbrella コンソール画面 | Connector VM のサービス状態と API キーを確認。 |
| **Status: Unreachable** | アウトバウンド FW 遮断 | <code>ping 208.67.222.222</code> | FW で TCP 443 および UDP 53 を許可する。 |
| 判定が適用されない | VA がクエリを受けていない | <code>ipconfig /all</code> | クライアントの DNS 設定が VA の IP か確認。 |

---

## ⚠ 制限事項

*   **暗号化 DNS (DoH/DoT)**: クライアントがブラウザで独自の DoH 設定を行っている場合、VA をバイパスして保護が効かない場合があります。
*   **パフォーマンス**: 大規模環境では、1 台の VA にクエリが集中しないよう、適切な負荷分散が必要です。
*   **非対応プロトコル**: VA は DNS 以外のトラフィック（HTTP 等）のプロキシとしては動作しません。それは **Umbrella SIG (SWG)** の役割です。

---

## 🔄 他技術との関連

*   **4.7 Active Directory 統合**: ユーザ情報を取得するための基盤となります。
*   **1.2.a NAT (ASA/FTD)**: NAT 越えの可視化問題を解決するために VA が使用されます。
*   **2.6 TrustSec (SGT)**: 将来的に SGT 情報を EDNS0 に含める高度な連携が可能です。

---

## 🧩 比較表

### Umbrella VA vs Roaming Client (AnyConnect)

| 特徴 | Virtual Appliance (VA) | Roaming Client (Module) |
| :--- | :--- | :--- |
| **設置場所** | ネットワークインフラ (サーバ) | クライアント端末内 |
| **導入の容易さ** | インフラ担当で完結 | 全端末への配布が必要 |
| **オフネット保護** | **不可** (社内のみ) | **可能** (カフェ、自宅等) |
| **可視化** | プライベート IP, AD 情報 | ホスト名, ログインユーザ |
| **推奨用途** | IoT デバイス, 管理外 PC | 持ち出し用ノート PC |

---

## 💡 ベストプラクティス

1.  **2台構成の徹底**: 可用性のために、1 サイトにつき必ず 2 台の VA をデプロイします。
2.  **内部 DNS の優先**: 内部 DNS サーバ自体も Umbrella VA をフォワーダとして使用するように構成し、サーバ自身のクエリも保護します。
3.  **SNMP 監視**: VA の稼働状態（CPU, メモリ）を NMS で監視し、クエリ遅延が発生しないようにします。

---

## 📝 ラボ学習・設定サンプル例

### 1. VA の基本登録
*   **要件**: VA1 (10.1.1.10) を起動し、Umbrella ダッシュボードで "Headquarter-VA" として承認せよ。

### 2. 内部ドメインフォワーディング
*   **要件**: `cisco.com` へのクエリは内部 DNS (10.1.1.5) へ転送するように構成せよ。

### 3. EDNS0 伝搬の確認
*   **操作**: クライアントから `nslookup` を行い、ログにプライベート IP が表示されることを確認せよ。

### 4. AD Connector との同期
*   **要件**: Windows サーバに Connector をインストールし、VA の identity 欄に AD ユーザ名を表示させよ。

### 5. ブロックページのカスタマイズ
*   **要件**: VA 経由の通信がブロックされた際、社内ヘルプデスクの電話番号を表示せよ。

### 6. IPv6 DNS プロキシ設定
*   **要件**: IPv6 クライアントからの DNS クエリも VA で中継するように構成せよ。

### 7. 特定サブネットのバイパス
*   **要件**: サーバ用サブネット (10.1.2.0/24) は VA を通さず直接 Umbrella へフォワードせよ。

### 8. VA CLI による疎通テスト
*   **コマンド**: <code>config va test-connectivity</code> を実行し、すべて PASS することを確認せよ。

### 9. 冗長 DHCP 配布
*   **要件**: Cisco IOS ルータの DHCP プールで、`dns-server 10.1.1.10 10.1.1.11` を設定せよ。

### 10. API による VA ステータス取得
*   **要件**: Umbrella API を使用して、VA のオンライン/オフライン状態を取得する Python スクリプトを作成せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `nslookup -q=txt debug.opendns.com` の出力で `va_id` が表示されていない場合、何が疑われるか？
    *   **回答**: クライアントが **VA をバイパス**して、直接ルータやクラウドの DNS (208.67.222.222) を参照している。
2.  **トラブルシュート**: VA を導入後、内部のファイル共有 (`\\fileserver`) にアクセスできなくなった。原因は？
    *   **回答**: VA の **Internal Domains** 設定に、内部ドメインが登録されていないため、クエリがクラウドへ送られ失敗している。
3.  **Design**: ローミングユーザとオフィス内ユーザの両方に一貫したセキュリティを提供するための Umbrella コンポーネントの組み合わせは？
    *   **回答**: **AnyConnect Umbrella Roaming Module** (外出先) と **Umbrella Virtual Appliance** (オフィス内)。
4.  **実装**: VA をデプロイする際、Umbrella クラウドへの通信に使用されるプロトコルとポート番号は？
    *   **回答**: **UDP 53** (DNS) および **TCP 443** (HTTPS/HTTPS 経由のクエリ)。
5.  **Design**: NAT デバイスを 5 台経由する複雑な多段ネットワークで、送信元の特定を最も確実に行う方法は？
    *   **回答**: 各セグメントに **Umbrella VA** を配置し、EDNS0 で IP 情報を付加する。

---

## 🔗 参考リソース

*   **Cisco Umbrella Documentation**: [Virtual Appliance Setup Guide](https://docs.umbrella.com/deployment-umbrella/docs/deploy-vas)
*   **Cisco Live (BRKSEC-2041)**: [Cloud Managed Security with Cisco Umbrella](https://www.ciscolive.com/)
*   **Design Guide**: [Best Practices for Umbrella VA Deployment](https://www.cisco.com/c/en/us/td/docs/security/umbrella/va/b_Umbrella_VA_Deployment_Guide.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「VA は DNS の付箋貼り」と考えてください。クエリという手紙に「これは 10.1.1.50 の Alice さんからです」という付箋を貼って Umbrella クラウド（巨大な郵便局）に送る役割です。
*   **注意点**: ラボ試験では、**VA の登録トークン**が有効期限切れになっていないか、あるいはダッシュボード上で承認（Approve）し忘れていないかに注意してください。
