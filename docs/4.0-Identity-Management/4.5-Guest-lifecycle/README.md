---
layout: default
title: 4.5-Guest-lifecycle
nav_order: 5
parent: 4.0-Identity-Management
---

# 4.5 Guest lifecycle management using Cisco ISE and Cisco WLC

ゲストライフサイクル管理は、企業ネットワークを訪れる一時的な利用者（ゲスト）に対し、セキュリティを維持しつつ、利便性の高いネットワーク接続を提供する仕組みです。Cisco ISE と WLC（Wireless LAN Controller）を連携させることで、登録から認証、承認、そして有効期限によるアクセス終了までのプロセスを完全に自動化・管理します。

---

## 📘 概要

*   **機能概要**: ゲストユーザーの ID 作成、ポータルを通じたセルフ登録、スポンサーによる承認、および利用期間の限定（有効期限管理）を行う機能です。
*   **利用目的**: 従業員用ネットワークからゲストを論理的に分離しつつ、最小限の運用負荷でインターネット接続等を提供すること。
*   **どのような場面で利用するか**:
    *   **来客用 Wi-Fi**: 会議や訪問のために訪れる外部ユーザーへの一時的なアクセス。
    *   **カンファレンス/イベント**: 大勢のユーザーに短期間のセルフ登録ポータルを提供。
    *   **コントラクター（協力会社）**: 特定の期間だけ特定の内部リソースへのアクセスを許可する。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **認証方式** | CWA (Central Web Authentication) が標準的。LWA (Local) は限定的。 |
| **ポータルタイプ** | Self-Registration, Sponsored, Hotspot。 |
| **スポンサー** | ゲストのアカウントを承認または作成する権限を持つ従業員。 |
| **MAC キャッシング** | 初回認証後の一定期間、ポータルログインをスキップする機能 (Device Registration)。 |
| **認可要素** | ゲストタイプ（Daily, Weekly, Permanent）、SGT、VLAN、dACL。 |
| **主要コンポーネント** | Cisco ISE (Policy Server), Cisco WLC (Authenticator), 外向きの DNS/DHCP。 |

---

## 🏗 動作原理

CWA（中央 Web 認証）を使用した一般的な通信フローを以下に示します。

```text
Guest Client
   ↓ (1) SSID "Guest" に接続
Cisco WLC
   ↓ (2) MAC Filtering (RADIUS Request) を ISE へ送信
Cisco ISE
   ↓ (3) 認証前認可 (Authorization: Redirect URL + Redirect ACL) を返信
Cisco WLC
   ↓ (4) クライアントの HTTP リクエストを ISE ポータルへリダイレクト
Guest Client
   ↓ (5) ポータルでログイン / セルフ登録
Cisco ISE
   ↓ (6) 認証成功後、CoA (Change of Authorization) を WLC へ送信
Cisco WLC
   ↓ (7) クライアントを再認可し、インターネットアクセスを許可
Guest Client
```

---

## ⚙ 動作シーケンス

1.  **初期接続**: クライアントが L2（Open + MAC Filtering）で接続。WLC は RADIUS Access-Request を ISE に送る。
2.  **リダイレクトの提示**: ISE はクライアントが未認証であることを認識し、認可プロファイルを通じて「ISE ポータル URL」と「リダイレクト ACL 名」を WLC に伝える。
3.  **インターセプト**: クライアントが Web ブラウザを開くと、WLC はリダイレクト ACL に基づき、HTTP トラフィックを ISE の PSN（Policy Service Node）へ転送する。
4.  **認証 / 登録**: ユーザーが資格情報を入力。セルフ登録の場合は ISE がアカウントを作成し、データベースに保存。
5.  **認可の変更 (CoA)**: ISE は認証成功をトリガーとして、RFC 5176 に基づく CoA を送信し、WLC 上のクライアントセッションを更新する。
6.  **最終アクセス許可**: クライアントは再度通信を開始し、今度はリダイレクトされずに通信が許可される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **リダイレクト ACL の設定**: WLC で定義する ACL 名と、ISE 認可プロファイルで指定する ACL 名が **完全に一致（大文字小文字含め）** している必要があります。
*   **DNS の重要性**: ゲストが ISE ポータルに到達するためには、認証前に DNS 解決ができる必要があります。リダイレクト ACL で DNS（UDP 53）を `permit`（＝リダイレクト対象外）にする設定は必須です。
*   **CoA の疎通**: WLC と ISE 間で UDP 1700 (Cisco) または 3799 (RFC) が許可されているか確認してください。これが通らないと、ログイン後に画面が「Success」になっても通信が許可されません。
*   **認証設定の組み合わせ**: WLC の WLAN セキュリティで `MAC Filtering` を有効にし、RADIUS サーバー設定で `Support for RFC 5176` を有効にする手順は頻出です。
*   **ゲストタイプのカスタマイズ**: Daily Guest は 1日、Weekly Guest は 7日といった「有効期限ポリシー」の作成が問われます。
*   **スポンサーポータルの構成**: 従業員が `https://ise:8443/sponsorportal` にアクセスし、ゲストを作成するフローの構築能力。

---

## 🛠 設定方法

### 1. Cisco WLC 側の構成 (CLI 例)
```bash
! リダイレクト用ACLの定義 (DNS, ISE, DHCPは許可、他はリダイレクト)
config access-list create GUEST_REDIRECT
config access-list add-rule GUEST_REDIRECT 1 0.0.0.0 0.0.0.0 53 0.0.0.0 0.0.0.0 any permit
config access-list add-rule GUEST_REDIRECT 2 0.0.0.0 0.0.0.0 any 10.1.1.100 255.255.255.255 any permit
config access-list add-rule GUEST_REDIRECT 3 0.0.0.0 0.0.0.0 any 0.0.0.0 0.0.0.0 any deny

! RADIUS設定
config radius auth add 1 10.1.1.100 1812 cisco123
config radius acct add 1 10.1.1.100 1813 cisco123
config radius auth call-station-id mac-address

! WLAN構成
config wlan create 2 Guest GuestSSID
config wlan security wpa2 dot1x disable 2
config wlan security wpa2 akm psk disable 2
config wlan security mac-filtering enable 2
config wlan radius_server auth add 2 1
config wlan radius_server acct add 2 1
config wlan allow-aaa-override enable 2
config wlan enable 2
```

### 2. Cisco ISE 側の構成
1.  **Guest Portals**: *Work Centers > Guest Access > Portal & Components* でポータルを作成。
2.  **Authorization Profile**: リダイレクト URL と `GUEST_REDIRECT` ACL 名を指定した認可プロファイルを作成。
3.  **Policy Set**: 
    *   **Identity Policy**: `Internal Endpoints`（MACフィルタリング用）。
    *   **Authorization Policy**: 
        *   1行目: もし認証済みゲストなら `PermitAccess`。
        *   2行目: 未認証なら作成した `Redirect-Profile`。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **WLC クライアントの詳細確認** | <code>show client details [MAC_ADDR]</code> |
| **認可プロファイルの適用確認** | WLC GUI の **Client > Detail** で Redirect URL が表示されているか。 |
| **ISE ライブログ確認** | **Operations > RADIUS > Live Logs** |
| **WLC RADIUS 統計** | <code>show radius statistics</code> |
| **CoA パケットのデバッグ** | <code>debug client [MAC_ADDR]</code> (WLC) |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| ポータル画面が表示されない | リダイレクト ACL で ISE への通信を `permit`（除外）していない | ACL ルールを確認し、ISE IP への通信を許可する。 |
| 「Success」画面から進まない | ISE からの CoA が WLC に届いていない | WLC の `aaa server radius dynamic-author` 設定を確認。 |
| DNS エラーが発生する | 認証前に DNS サーバへ到達できない | ACL で DNS (UDP 53) を許可し、クライアントに正しい IP が配布されているか確認。 |
| MAC フィルタリングで拒否される | ISE の Identity Policy で `Continue` 設定がされていない | ユーザーが見つからない場合も `Continue` して認可ポリシーへ進むよう設定。 |

---

## ⚠ 制限事項

*   **HTTPS リダイレクト**: 多くのブラウザで証明書の警告が出ます。これは ISE に公的な CA 証明書をインストールすることで回避可能です。
*   **Apple CNA (Captive Network Assistant)**: 一部の iOS デバイスではリダイレクト ACL の構成によってポータルが自動ポップアップしない場合があります。
*   **Android の挙動**: Android デバイスは、インターネット接続が確認できるまで Wi-Fi 接続を維持しないことがあるため、ポータルへの早期到達が必要です。

---

## 🔄 他技術との関連

*   **4.1 ISE Scalability**: ゲストポータルへのトラフィックを処理するために、PSN の冗長化が重要です。
*   **4.2 Network Access AAA**: WLC が RADIUS クライアントとして正しく登録されている必要があります。
*   **3.4.e DHCP Snooping**: クライアントの IP 学習が遅れると、ISE への IP 属性報告が間に合わずリダイレクトに失敗します。
*   **Cisco DNA Center**: DNAC からゲスト用の SSID やポリシーを自動プロビジョニングすることが可能です。

---

## 🧩 比較表

### CWA vs LWA

| 特徴 | CWA (Central Web Auth) | LWA (Local Web Auth) |
| :--- | :--- | :--- |
| **Web サーバー** | **Cisco ISE** | Cisco WLC |
| **認可更新** | CoA を使用 | セッション再確立 |
| **カスタマイズ性** | 高い（ISE ポータルビルダ） | 低い（WLC 内蔵） |
| **推奨** | **現代のエンタープライズ環境** | スタンドアロン、小規模環境 |

---

## 💡 ベストプラクティス

1.  **MAC キャッシングの有効化**: ゲストが一度ログインしたら、その日はポータルを再度出さないように `Endpoint Identity Group` への自動登録を構成します。
2.  **スポンサーグループの制限**: 全従業員ではなく、受付や特定の部署のみがゲストアカウントを発行できるよう AD グループで制限します。
3.  **自己署名証明書の回避**: ラボでは良いですが、本番や試験の「Design」観点では、ISE ポータルには必ず信頼された証明書を使用します。
4.  **AUP (利用規約) の提示**: ポータル画面で利用規約への同意ボタンを必須に設定します。

---

## 📝 ラボ学習・設定サンプル例

### 1. CWA 基本構成
*   **問題**: WLC (10.1.1.5) と ISE (10.1.1.100) を使い、SSID "Guest" で CWA を実装せよ。
*   **要件**: リダイレクト ACL 名は `ACL_CWA` とする。

### 2. セルフ登録ポータルの有効化
*   **要件**: ゲストが自分でアカウントを作成し、1 日間有効な資格情報を取得できるようにせよ。

### 3. スポンサー承認フロー
*   **要件**: ゲストが登録後、従業員（スポンサー）のメール承認が得られるまでアクセスを保留せよ。

### 4. ゲストの有効期限設定
*   **問題**: `Contractor` タイプのゲストは 8:00〜18:00 の間のみアクセス可能とし、それ以外は拒否せよ。

### 5. デバイス登録 (MAC Caching)
*   **要件**: 一度ポータルログインしたデバイスは、12 時間以内であれば MAC 認証のみで通過させよ。

### 6. 特定サイト専用ポータル
*   **操作**: 接続元の WLC（Location 属性）に基づき、日本語と英語のポータルを自動的に切り替えよ。

### 7. スポンサーポータルの RBAC
*   **要件**: `Reception_Group` のユーザーはゲスト作成のみ、`IT_Admin` はゲストの削除も可能にせよ。

### 8. リダイレクト ACL での特定通信許可
*   **要件**: 認証前でも会社のホームページ (`www.cisco.com`) へのアクセスを許可せよ。

### 9. ポスチャ連携
*   **要件**: ゲストデバイスに対しても、簡易的なウイルス対策チェック（ポスチャ）を強制せよ。

### 10. SMS 通知設定
*   **問題**: ゲスト登録時に、認証情報を SMS で送信するためのゲートウェイ設定を構成せよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: WLC の WLAN 設定で `RADIUS NAC` が有効になっている。これが必要な理由は？
    *   **回答**: ISE からの **CoA (Change of Authorization)** メッセージを受け入れ、認証後に認可状態を動的に変更するため。
2.  **トラブルシュート**: ユーザーがポータルでログインしたが、インターネットに繋がらない。ISE Live Logs では `CoA Success` が出ている。次に WLC のどこを確認すべきか？
    *   **回答**: WLC の認可されたクライアント詳細で、**適用された ACL や VLAN** が ISE の指示通りになっているか、および WLC 側のルーティングを確認。
3.  **Design**: 多数の拠点でゲスト Wi-Fi を提供する場合、ISE のどの Persona を拠点に近く配置すべきか？
    *   **回答**: **PSN (Policy Service Node)**。ポータル画面のロード遅延を減らすため。
4.  **実装**: CWA において、クライアントが最初に受ける RADIUS 応答に含まれる 2 つの重要な VSA（ベンダー固有属性）は？
    *   **回答**: `url-redirect` と `url-redirect-acl`。
5.  **Design**: ゲストアクセスにおいて MAC Filtering を最初に行う理由は？
    *   **回答**: クライアントを特定し、既に認証済み（MAC キャッシング）か、あるいはポータルへリダイレクトすべき未認証状態かを ISE が判断するため。

---

## 🔗 参考リソース

*   **Cisco ISE 3.1 管理者ガイド**: [ゲストアクセスの設定](https://www.cisco.com/c/en/us/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31/b_ise_admin_guide_31_chapter_01100.html)
*   **Cisco Live (BRKSEC-2023)**: [Deep Dive into Guest Access with Cisco ISE](https://www.ciscolive.com/)
*   **Design Guide**: [Central Web Authentication (CWA) on WLC and ISE](https://www.cisco.com/c/en/us/support/docs/security-software/identity-services-engine/115737-ise-cwa-config-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「ゲスト = 未知のデバイス」として扱い、ISE ポータルを「関所」として機能させることが重要です。
*   **図解**: 
    - **Portal** = 見た目の制御。
    - **Policy Set** = 裏側のロジック制御。
*   **注意点**: ラボ試験では、**WLC の証明書設定**が不十分でリダイレクトが失敗するケースがあるため、ASDM や CLI で `ip http secure-server` が有効であることを確認してください。
