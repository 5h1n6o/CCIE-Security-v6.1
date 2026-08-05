---
layout: default
title: 1.2.f-Identity-firewall
nav_order: 6
parent: 1.2-Firewall-features-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.2.f Identity firewall

Cisco ASAおよびFirepower Threat Defense (FTD) における**アイデンティティファイアウォール（Identity Firewall: IDFW）**は、従来のIPアドレスベースの制御に代わり、「誰が（ユーザー/グループ）」通信しているかに基づいて動的にアクセス制御を行う機能です。ネットワークの可動性が高まり、IPアドレスが頻繁に変わる現代の環境において、よりきめ細かなセキュリティポリシーを実現します。

---

# 📘 概要

*   **機能概要**: Active Directory (AD) や Cisco ISE などのアイデンティティソースから取得した「ユーザー名とIPアドレスの紐付け」情報を利用し、ファイアウォールポリシー（ACL/ACP）を適用します。
*   **利用目的**: IPアドレス管理の複雑さを解消し、ユーザーの職務権限に応じた一貫したセキュリティレベルを維持するために利用されます。
*   **どのような場面で利用するか**: 
    *   VDI環境やDHCP環境など、ユーザーに割り当てられるIPアドレスが固定されない環境。
    *   特定の組織（営業部、開発部など）のユーザーに対して、場所を問わず特定のアプリケーションへのアクセスを許可する場合。

---

# 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | ユーザー名、グループ名、SGT（Scalable Group Tag）に基づく制御。 |
| **主な用途** | ユーザーベースのアクセス制御、BYOD、コンプライアンス遵守。 |
| **メリット** | ポリシーの抽象化、セキュリティの強化、監査の容易性。 |
| **デメリット** | 設定の複雑化、外部サーバー（AD/ISE）への依存、パフォーマンスへの影響。 |
| **対応機種** | ASA 5500-X シリーズ、FTD（全モデル）、ASAv/FTDv。 |
| **認証方式** | パッシブ認証（AD Agent/ISE）およびアクティブ認証（Captive Portal）。 |
| **設計上の注意** | ユーザーとIPのマップ情報の保持期間（Timeout）やスケーラビリティの考慮。 |

---

# 🏗 動作原理

IDFWは、アイデンティティソースからパケット転送パスへ情報を供給するアーキテクチャに基づいています。

```text
[Identity Source (AD/ISE)]
       ↓ (1) Login Event / User-IP Map
[Firepower Management Center (FMC) / ASA]
       ↓ (2) Policy Push (User/Group info)
[ASA / FTD Device]
       ↓ (3) Packet Arrival (Source IP: 10.1.1.5)
[Security Engine]
       (4) Lookup: 10.1.1.5 = User: "John"
       (5) Apply: Permit "John" to "HR_Server"
       ↓
[Egress Interface]
```

---

# ⚙ 動作シーケンス

1.  **情報の取得（パッシブ認証）**: ユーザーがPCでADにログインします。ADのセキュリティログを **Cisco AD Agent** または **Cisco ISE (via pxGrid)** が読み取り、デバイス（ASA/FTD）へ「IP:10.1.1.5 = ユーザー:John」という情報を通知します。
2.  **マップの生成**: デバイスはローカルの **User-IP Table** にこの情報を格納します。
3.  **パケット処理**: 通信パケットが到着すると、デバイスは送信元IPをキーにユーザーを特定します。
4.  **ポリシー評価**: 
    *   ユーザーがポリシー上の許可条件（例: グループ「Sales」）に含まれていれば、パケットを許可します。
    *   ユーザー情報が不明な場合、**アクティブ認証（Captive Portal）**を起動してユーザーに再認証を促すことができます。

---

# 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **Identity Sourceの優先順位**: ラボでは「ISEからの情報を優先せよ」といった条件が出る場合があります。
*   **Realm (FTD)**: FMCでのADドメインとの接続設定（Realm）の構成が必須です。
*   **SGT (Cisco TrustSec)**: ユーザー情報だけでなく、ISEから伝搬されるSGTを使用した制御との組み合わせが問われます。

### ラボ試験で設定させられそうな内容
*   **Realmの作成**: ADサーバーのIP、ポート、ベースDNの設定。
*   **Identity Policyの構成**: どのゾーンでパッシブ認証を使い、どのゾーンでCaptive Portal（アクティブ認証）を使うかの定義。
*   **ASA IDFW CLI**: `user-identity` コマンドを使用したAD Agentとの連携設定。

### よくある設定ミス
*   **DNS解決の失敗**: FTDがADサーバーの名前解決ができないとRealmが正常に動作しません。
*   **タイムアウト設定**: IPアドレスの再利用が激しい環境でタイムアウトが長すぎると、別人が以前のユーザーとして認識される誤検知が発生します。

### トラブルシュート問題
*   「ユーザーベースのポリシーを作成したが、全員が拒否される」というケース。
*   確認ポイント: 
    1.  Realmの接続状態（`show realm`）。
    2.  User-IP マップの存在（`show user-identity user`）。
    3.  ログの `User: Unknown` 表示の有無。

---

# 🛠 設定方法

### FTD (FMC管理) - Identity Firewall 設定
1.  **Objects > Object Management > Realms**:
    *   ADサーバーを追加。
    *   **Directoryタブ**: ユーザー/グループ情報をダウンロードするスケジュールを設定。
2.  **Policies > Identity**:
    *   **Add Rule**: 送信元ゾーンを指定し、**Action** を「Passive Authentication」に設定。
3.  **Policies > Access Control**:
    *   ルールの **Usersタブ** で、許可したいユーザーまたはグループ（Realmsから選択）を追加。

### ASA (CLI) - AD Agent を使用した設定例
```bash
# AD Agent の定義
user-identity ad-agent AD_AGENT_1
 address 192.168.10.50
 key cisco123

# ACL への適用
access-list IN_TO_OUT extended permit ip user MYDOMAIN\Sales any any
access-group IN_TO_OUT in interface inside
```

---

# 🔍 検証コマンド

| 目的 | コマンド (ASA/FTD) |
| :--- | :--- |
| **ユーザー-IPマップの確認** | `show user-identity user` (ASA) / `show user-identity-map` (FTD LINA) |
| **Realmのステータス確認** | `show realm` (FTD CLI) |
| **現在のログイン数確認** | `show user-identity statistics` |
| **アイデンティティソースの状態** | `show user-identity ad-agent` |
| **パケットパスでのユーザー特定検証** | `packet-tracer input inside tcp 10.1.1.5 1234 8.8.8.8 80` |

---

# 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| FMCでグループが表示されない | AD同期の失敗 | FMC GUI: Realms Status | 同期スケジュールと認証情報の再確認。 |
| ユーザーが Unknown と判定される | パッシブ認証ログの欠落 | `show user-identity-map` | AD Agentの状態やADの監査設定を確認。 |
| Captive Portalが表示されない | 証明書エラーまたはポート遮断 | `debug http-auth` | ポート885/443の開放とCA証明書を確認。 |
| 通信が遅延する | ID情報の解決待ち | `show cpu` | 多数のログインイベントによる過負荷を確認。 |

---

# ⚠ 制限事項

*   **サポートされるディレクトリ**: 主にMicrosoft Active Directoryに最適化されています。
*   **マップの正確性**: パッシブ認証のみの場合、ログオフイベントを正確に検知できない場合があります。
*   **マルチコンテキスト**: ASAマルチコンテキストモードでは一部のIDFW機能に制限がある場合があります。

---

# 🔄 他技術との関連

*   **Cisco ISE**: pxGrid を介した情報の供給により、より詳細なプロファイリング情報を共有可能。
*   **TrustSec (SGT)**: Identity Policy で取得したユーザー情報に基づき SGT を付与する構成が一般的。
*   **AnyConnect**: AnyConnect でログインしたユーザー情報を IDFW ポリシーにシームレスに統合可能。

---

# 🧩 比較表

### パッシブ認証 vs アクティブ認証

| 比較項目 | パッシブ認証 (Passive) | アクティブ認証 (Active) |
| :--- | :--- | :--- |
| **ユーザーの介入** | **なし（透明）** | **あり（Webログインが必要）** |
| **情報のソース** | AD Agent, ISE pxGrid | Captive Portal (Webブラウザ) |
| **正確性** | ADログに依存 | 非常に高い（確実に本人確認） |
| **主な用途** | 社内業務用、PC利用者 | ゲスト、非ドメイン端末 |

---

# 💡 ベストプラクティス

*   **階層型アプローチ**: まずパッシブ認証を試み、不明な場合のみアクティブ認証（Captive Portal）にフォールバックさせる設計が推奨されます。
*   **ISEの活用**: 可能であればAD Agentではなく、pxGrid を介した ISE との連携を選択し、セキュリティ情報の統合性を高めます。
*   **オブジェクトの整理**: グループ名はAD上の構造（OU）に合わせ、管理ミスを防ぎます。

---

# 📝 ラボ学習・設定サンプル例

### 1. FTD Realm の作成 (AD)
*   **要件**: `cisco.com` ドメインのADと同期せよ。
*   **設定**: FMC > Objects > Realms。ADの管理権限を持つユーザー、IPを指定。

### 2. パッシブ認証ルールの作成
*   **要件**: InsideゾーンのADドメイン参加端末を透過的に認証せよ。
*   **設定**: Identity Policy で `Rule: Passive Auth` を作成。

### 3. キャプティブポータルの構成
*   **要件**: 認証マップにないユーザーに対し、ポート885でログイン画面を出せ。
*   **設定**: Identity Policy > Active Authentication。

### 4. 特定グループ（Sales）のインターネットアクセス許可
*   **要件**: Salesグループ以外のインターネット（HTTP）を拒否せよ。
*   **設定**: ACP Rule > Usersタブ で `Sales` を指定。Action: Allow。

### 5. ISE pxGrid 連携
*   **要件**: ISEからSGTタグ情報を取得し、ACPで利用せよ。
*   **設定**: FMC > Devices > Certificates (ISE証明書登録) ＋ Integration > pxGrid。

### 6. ASA IDFW: ローカルAD Agent連携
*   **要件**: ASA CLIからAD Agent経由でユーザー情報を取得。
*   **設定**: `user-identity ad-agent` の定義。

### 7. ユーザーベースのNAT除外
*   **要件**: `Admin` ユーザーが特定のサーバーへ通信する際はNATを回避せよ。
*   **注意**: 多くのプラットフォームでNATはIPベースで行われるため、Identity Policyとの順序を考慮。

### 8. Identity 情報のデバッグ
*   **課題**: ユーザー情報が届かない際の調査。
*   **コマンド**: `system support-firewall-engine-debug` (FTD) で Identity イベントをトレース。

### 9. 複数のドメイン（Realms）の共存
*   **要件**: `domain-A` と `domain-B` の両方のユーザーを制御。
*   **設定**: 2つのRealmを作成し、Identity Policyでそれぞれを条件に指定。

### 10. packet-tracer によるIDFW検証
*   **課題**: 10.1.1.10 が "user1" として認識され許可されているか確認。
*   **実行**: `packet-tracer input inside tcp 10.1.1.10 ...` の出力で `User: user1` を確認。

---

# ❓ 想定試験問題

1.  **コンフィグ読解**: FMCのIdentity Policyで「Passive Authentication」が設定されているが、実際のパケットがドロップされる。原因として最も可能性が高いのは？
    *   **正解**: ADサーバーとの同期が失敗している、あるいはパケットの送信元IPに対するマップ情報がFMC/FTDに存在しない。
2.  **Design**: 大規模なキャンパスネットワークで、全ユーザーのアイデンティティ情報を単一のFTDで処理したい。最適なパッシブ認証の方式は？
    *   **正解**: ISE pxGrid を使用した連携（AD Agentよりスケーラビリティに優れるため）。
3.  **動作シーケンス**: パケットがFTDに着信した際、Identityルールの評価はACPルールの評価より前か後か？
    *   **正解**: 前。Identityルールによって「誰か」が確定した後に、ACPルールのユーザー条件が評価される。
4.  **トラブルシュート**: `show user-identity-map` で古い情報が残っている。これを手動でクリアするコマンドは？
    *   **正解**: `clear user-identity user-ip-map`。
5.  **実装**: キャプティブポータルをHTTPSで使用するために必須の設定は？
    *   **正解**: デバイスへの信頼された証明書のインポートと、インターフェイスでのHTTPSサービスの有効化。

---

# 🔗 参考リソース

*   **Configuration Guides**:
    *   [Cisco ASA Series Firewall CLI Configuration Guide, 9.4 - Identity Firewall](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/firewall/asa-94-firewall-config/idfw.html)
    *   [Cisco Secure Firewall Management Center Administration Guide, 7.1 - Identity Policies](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/identity_policies.html)
*   **Cisco Live**:
    *   [BRKSEC-3020: Troubleshooting Firewall Threat Defense (FTD)](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-3020)
*   **Technical Notes**:
    *   [Configuring the Identity Firewall on Cisco ASA](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/113645-asa-idfw-config-00.html)
    *   [Firepower Threat Defense: Identity Policy via Captive Portal Example](https://www.cisco.com/c/en/us/support/docs/security/firepower-ngfw/200508-Configure-Active-Authentication-on-Firep.html)
*   **Design Guide**:
    *   [Cisco SAFE Design Guide - Identity-Aware Infrastructure](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Aug2014/CVD-FirewallDeploymentGuide-AUG14.html)  

---

📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

---
