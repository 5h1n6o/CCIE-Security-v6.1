# 2.1 Cisco AnyConnect client-based, remote-access VPN technologies on Cisco FTD

Cisco Secure Firewall (FTD) における **AnyConnect リモートアクセス VPN (RA VPN)** は、Cisco Firepower Management Center (FMC) または Firepower Device Manager (FDM) を通じて一元管理される強力なセキュア接続ソリューションです。FTD は内部的に LINA エンジン（ASA ベースのコード）を使用して VPN 通信を処理するため、ASA の堅牢な VPN 機能を引き継ぎつつ、Snort による高度な脅威防御と統合されています。

---

## 📘 概要

*   **機能概要**: AnyConnect ソフトウェアをインストールした端末から、SSL/TLS (HTTPS) または IKEv2 (IPsec) を使用して FTD との間に暗号化トンネルを構築します。
*   **利用目的**: テレワーカーやモバイルユーザーに対して、社内リソースへのセキュアなアクセスを提供し、同時に Snort インスペクション（IPS, マルウェア防御）を適用して接続端末からの脅威流入を防ぎます。
*   **利用場面**: リモートワーク環境の提供、クライアント証明書による高度な認証が必要な環境、端末のセキュリティ状態（ポスチャ）に基づくアクセス制御を行う場合。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **管理主体** | **FMC (Firepower Management Center)** が基本。大規模構成には必須。 |
| **プロトコル** | SSL (TLS/DTLS) または **IKEv2**。ASA と同様に DTLS を推奨。 |
| **認証方式** | AAA (RADIUS/ISE, LDAP), クライアント証明書, SAML 2.0 (Azure/Okta), Duo (MFA)。 |
| **IPアドレス割当** | IPv4/IPv6 ローカルプール、外部 AAA サーバーからの属性プッシュ。 |
| **ポリシー制御** | **Group Policy** によるスプリットトンネル、DNS、セッション属性の定義。 |
| **ライセンス** | AnyConnect Plus/Apex/VPN Only が必要。 |
| **Snort 連携** | トンネル内トラフィックを直接 ACP ルールでインスペクション可能。 |

---

## 🏗 動作原理

FTD RAVPN は、パケット転送を担う **LINA エンジン** と、設定を管理する **FMC**、そして脅威を検知する **Snort** の 3 つが協調して動作します。

```text
AnyConnect Client
   ↓ (SSL/TLS or IKEv2 over Internet)
FTD Outside Interface
   ↓
[ LINA Engine: VPN Termination ] <--- Auth with ISE/RADIUS
   ↓
[ Snort Engine: Inspection ] <--- Apply IPS/File Policy
   ↓
[ Internal Network/Server ]
```

---

## ⚙ 動作シーケンス

1.  **ハンドシェイク**: クライアントが FTD の公開 IP に接続し、SSL または IKEv2 のネゴシエーションを開始。
2.  **プロファイル選択**: 送信された Group-URL または Alias に基づき、FMC で設定された Connection Profile が選択される。
3.  **認証 (Authentication)**: ユーザー資格情報または証明書を検証。外部 ISE との連携では RADIUS 経由で照会。
4.  **認可 (Authorization)**: Group Policy が適用され、スプリットトンネル ACL、DNS 情報、仮想 IP アドレスがクライアントへプッシュされる。
5.  **トンネル確立**: 暗号化された仮想インターフェイス (Virtual-Template) がアクティブになり、データ転送が開始される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **FMC Wizard の活用**: `Devices > VPN > Remote Access` のウィザードを使用して、Connection Profile, Group Policy, AnyConnect Image を一括設定する手順を習得することが、ラボ試験での時間短縮に繋がります。
*   **証明書認証 (Certificate Map)**: ユーザー名入力をスキップし、証明書のフィールド（CN, OU 等）から特定のトンネルグループへ自動的に振り分ける設定は頻出です。
*   **AnyConnect Image の管理**: FMC の `Objects > Object Management > VPN > AnyConnect File` に `.pkg` ファイルを登録し、ポリシーに関連付ける手順を確認してください。
*   **Split Tunneling**: ラボ要件で「特定の内部セグメントのみ VPN を通せ」と指示された場合、Standard ACL を使用し `tunnelspecified` 設定を行う必要があります。
*   **ISE ポスチャ連携**: ISE と連携して、端末のアンチウイルスが有効な場合のみアクセスを許可する構成。
*   **Troubleshooting (FMC vs CLI)**: FMC の `Health Monitor` だけでなく、FTD CLI の `system support diagnostic-cli` に入り、従来の ASA コマンド (`show vpn-sessiondb remote`) でセッションを確認するスキルが必須です。

---

## 🛠 設定方法

### 1. AnyConnect パッケージの登録 (FMC)
*   `Objects > Object Management > VPN > AnyConnect File` で、フラッシュから `.pkg` をアップロードし、名前を付けて保存します。

### 2. RAVPN ウィザードの実行
1.  `Devices > VPN > Remote Access` で `Add` をクリック。
2.  **Connection Profile Name** と **Authentication Method** (RADIUS 等) を指定。
3.  **Group Policy** を作成し、`Split Tunneling` や `DNS` を定義。
4.  **AnyConnect Image** タブで、先ほど登録したオブジェクトを選択。

### 3. スプリットトンネルの ACL 設定
```bash
! FMC GUI の Objects > Access List > Standard で作成
access-list SPLIT_VPN standard permit 10.1.1.0 255.255.255.0
access-list SPLIT_VPN standard permit 10.2.2.0 255.255.255.0
```

---

## 🔍 検証コマンド

FTD の VPN 状態を確認するには、診断 CLI にアクセスする必要があります。

| 目的 | コマンド (Diagnostic CLI) |
| :--- | :--- |
| **接続ユーザーの概要確認** | <code>show vpn-sessiondb remote</code> |
| **特定のユーザー詳細確認** | <code>show vpn-sessiondb detail remote [User]</code> |
| **IKEv2 SA の確認** | <code>show crypto ikev2 sa</code> |
| **暗号化統計の確認** | <code>show crypto ipsec sa</code> |
| **リアルタイムデバッグ** | <code>debug webvpn anyconnect 255</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 接続時に "Login Failed" | AAA サーバー (ISE/LDAP) の疎通不良 | <code>test aaa-server</code> で接続テストを実施。 |
| プロファイルが選ばれない | Group-URL/Alias の設定ミス | FMC で <code>Group-Alias</code> が有効か、URL が正しいか確認。 |
| 内部リソースへ Ping 不通 | 戻りのルート、または ACP 遮断 | 接続先の FTD で <code>packet-tracer</code> を実行し Snort 判定を確認。 |
| 証明書エラーで接続不可 | トラストポイントの欠如 | FTD に CA 証明書が正しくインポートされているか確認。 |
| DTLS が確立しない | UDP 443 の経路遮断 | 外部 Firewall 等で UDP 443 が許可されているか確認。 |

---

## ⚠ 制限事項

*   **GUI 依存**: 多くの高度な VPN 設定は CLI (FlexConfig) ではなく、FMC GUI のポリシー設定に統合されています。
*   **同時接続数**: ハードウェアモデルおよびライセンスによって、最大同時セッション数が制限されます。
*   **Snort 負荷**: VPN トラフィックに対して重い IPS ポリシーを適用すると、デバイスのスループットに影響を与える可能性があります。

---

## 🔄 他技術との関連

*   **NAT**: VPN クライアントの IP アドレスが、内部へ通信する際に NAT されないよう「NAT 免除 (Exemption)」の設定が必要です。
*   **Routing**: FTD 上に VPN プール宛のルートは自動作成されますが、内部コアスイッチ側には VPN プールへの戻りルートが必要です。
*   **Cisco ISE (pxGrid)**: VPN ユーザーの情報を SGT (Security Group Tag) と共に内部ネットワークへ伝搬させます。
*   **Identity Policy**: VPN で認証されたユーザー名を Access Control Policy のソース条件として使用可能です。

---

## 🧩 比較表

### FTD RAVPN vs ASA RAVPN

| 特徴 | Cisco FTD | Cisco ASA |
| :--- | :--- | :--- |
| **管理** | FMC/FDM によるポリシーベース | CLI/ASDM によるオブジェクトベース |
| **脅威防御** | **Snort インスペクション統合** | 基本は L3/L4。Firepower Service が必要 |
| **設定速度** | ウィザードで迅速に構築可能 | コマンドの組み合わせが必要 |
| **ポスチャ** | ISE ポスチャとの深い連携 | HostScan および ISE 連携 |

---

## 💡 ベストプラクティス

1.  **DTLS 2.0 の使用**: 音声・ビデオ通信の品質向上のため、常に DTLS を有効化します。
2.  **証明書ベースの認証**: パスワード漏洩リスクを低減するため、マシン証明書またはユーザー証明書を推奨します。
3.  **スプリット DNS の設定**: 社内ドメインのみ社内 DNS に飛ばし、パブリックドメインはローカル解決させることで、FTD の負荷を軽減します。
4.  **DAP (Dynamic Access Policy)**: 端末の OS やログイン場所に基づいて、動的に VPN 特権を制限する DAP を活用します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 基本的な SSL VPN (FMC Wizard)
*   **要件**: AnyConnect ユーザーに 192.168.100.0/24 の IP を割り当て、Local 認証で接続させよ。
*   **設定**: FMC RA VPN Wizard -> Auth: Local -> Pool: 192.168.100.1-254.

### 2. ISE RADIUS 認証の統合
*   **要件**: VPN 認証を ISE (10.1.1.50) で実施せよ。
*   **設定**: `aaa-server` を RADIUS として構成し、FMC の Connection Profile に適用。

### 3. スプリットトンネルの実装
*   **要件**: 内部ネットワーク 10.0.0.0/8 への通信のみトンネルを通せ。
*   **設定**: Standard ACL (permit 10.0.0.0/8) -> Group Policy > Split Tunneling: `Tunnelspecified`。

### 4. クライアント証明書による自動接続
*   **要件**: 証明書の OU が "Security" の場合のみ、SECURITY_GP へ接続させよ。
*   **設定**: `Certificate Map` を作成し、OU フィールドをマッチ条件に指定。

### 5. AnyConnect IKEv2 の有効化
*   **要件**: SSL ではなく IKEv2 プロトコルを使用して接続せよ。
*   **設定**: Connection Profile 内で `IKEv2` プロトコルにチェックを入れ、FTD インターフェイスで IKEv2 を有効化。

### 6. SAML 2.0 認証 (Azure AD)
*   **要件**: 外部 IdP を使用して AnyConnect 認証を行え。
*   **設定**: FMC に IdP メタデータをインポートし、SAML サーバーオブジェクトを作成。

### 7. AnyConnect Profile (XML) の配布
*   **要件**: Always-on 機能を有効にした XML プロファイルをクライアントへプッシュせよ。
*   **設定**: `AnyConnect Client Profile` オブジェクトを作成し、Group Policy へ関連付け。

### 8. VPN トラフィックの Snort インスペクション
*   **要件**: VPN ユーザーからの通信に対して "Balanced Security" IPS ポリシーを適用せよ。
*   **設定**: Access Control Policy で Source Zone に VPN ゾーンを指定し、Inspection タブでポリシー選択。

### 9. IPv6 アドレスプールの割り当て
*   **要件**: デュアルスタック環境で VPN クライアントに IPv6 アドレスを割り当てよ。
*   **設定**: Group Policy 内で IPv6 アドレスプールを構成。

### 10. 診断 CLI によるトラブルシュート
*   **要件**: 特定のユーザー "student" のセッション情報を CLI で表示せよ。
*   **実行**: `system support diagnostic-cli` -> `show vpn-sessiondb detail remote student`。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: FMC で RAVPN を設定したが、クライアントが接続時に証明書エラーで停止する。FTD のどのオブジェクトを確認すべきか？
    *   **回答**: FTD デバイスに関連付けられた `PKI Cert Enrollment` オブジェクトおよびトラストポイント。
2.  **トラブルシュート**: AnyConnect IKEv2 で接続できない。`debug crypto ikev2` で "Proposal mismatch" と表示された。解決策は？
    *   **回答**: FMC の IKEv2 ポリシー内の暗号化/ハッシュアルゴリズムが、クライアント側の設定（またはデフォルト）と一致しているか確認。
3.  **Design**: 拠点ごとに異なるスプリットトンネルを適用したい。FMC のどのレベルで設定を分けるべきか？
    *   **回答**: 各拠点ごとに個別の **Group Policy** を作成し、Connection Profile で振り分ける。
4.  **実装**: ユーザーがブラウザで FTD の IP を叩いた際、特定のグループをドロップダウンで選べるようにするための設定は？
    *   **回答**: Connection Profile で `Group Alias` を有効にし、RA VPN 全体設定で `Tunnel Group List` を有効化する。
5.  **トラブルシュート**: VPN 接続は成功したが、内部サーバー (10.1.1.10) と通信できない。FTD 上で確認すべき最も重要な LINA レベルの設定は？
    *   **回答**: 戻りのトラフィックに対する **NAT 免除 (No-NAT)** ルールの有無。

---

## 🔗 参考リソース

*   [Cisco Secure Firewall Management Center Administration Guide, 7.1 - AnyConnect VPN](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/anyconnect-vpn.html)
*   [Cisco Secure Firewall Threat Defense - Remote Access VPN Configuration Guide](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/fdm/fptd-fdm-config-guide-710/fptd-fdm-ravpn.html)
*   [Cisco Live BRKSEC-3033: Advanced AnyConnect Implementation](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

*   **学習メモ**: FTD RAVPN の本質は「LINA によるカプセル化」と「Snort による中身の検査」の融合です。
*   **図解**: 常に `packet-tracer` の出力をイメージし、どのフェーズ（VPN 解除、ACL、Snort、NAT）でパケットが処理されているかを意識してください。
*   **注意点**: ラボ試験では、AnyConnect の `.pkg` ファイルがデバイスのローカルストレージにあるか、FMC から配信可能かをまず確認することが重要です。
