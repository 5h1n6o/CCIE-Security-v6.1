---
layout: default
title: 1.7.d-MITM
nav_order: 4
parent: 1.7-Attack-detection
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.7.d Man-in-the-middle

Cisco ネットワークインフラストラクチャおよびセキュリティアプライアンスにおける **Man-in-the-Middle（MitM：中間者攻撃）** 対策は、ネットワークのレイヤー 2 (Data Link)、レイヤー 3/4 (Network/Transport)、およびレイヤー 7 (Application/SSL/TLS) にわたって通信経路に不当に割り込み、データの盗聴・改ざん・セッションハイジャックを行うサイバー攻撃を防止・検知・遮断すると同時に、**セキュリティアプライアンス自体が「正当な中間者（Managed Inline Proxy）」として暗号化トラフィック（SSL/TLS）を復号・検査する技術**を包括した総合セキュリティアーキテクチャです。

CCIE Security v6.1 Blueprint **1.7.d Man-in-the-middle** では、悪意ある MitM 攻撃の代表格である **ARP Cache Poisoning (DAI)、Rogue DHCP サーバー (DHCP Snooping)、IPv6 Rogue RA / NDP Spoofing (IPv6 FHS)、DNS キャッシュポイズニング・リダイレクト (DNS Inspection / Sinkhole)、および SSL Stripping / Session Hijacking** に対する多層防御設計に加え、**Cisco Secure Firewall Threat Defense (FTD / FMC) や Cisco Web Security Appliance (WSA) による SSL/TLS 復号ポリシー（Decrypt-Resign / Decrypt-Known Key）を用いたインラインインスペクション**の実装・トラブルシューティング能力が厳密に評価されます。

---

## 📘 概要

* **機能概要**:
  * **Layer 2 悪意ある MitM 阻止**:
    * **ARP Poisoning / Spoofing 防御**: Dynamic ARP Inspection (DAI) を用い、DHCP Snooping データベースまたは静的 ARP ACL と不整合を起こす不正 ARP 応答（Gratuitous ARP を悪用した Gateway なりすまし等）を遮断する。
    * **Rogue DHCP サーバー防御**: DHCP Snooping により、未承認ポートからの DHCPOFFER / DHCPACK（偽のデフォルトゲートウェイや DNS サーバー情報の挿入）を物理ポートレベルでブロックする。
    * **IPv6 Neighbor Discovery (NDP) / Rogue RA 防御**: IPv6 First Hop Security (RA Guard, DHCPv6 Guard, SISF) により、偽のルーター広告 (RA) や NDP 応答による L2 トラフィック迂回攻撃を排除する。
  * **Layer 3 / DNS MitM 阻止**:
    * **DNS Cache Poisoning 防御**: ASA/FTD DNS Application Inspection による RD (Recursion Desired) フラグ制御や、Cisco Umbrella / Security Intelligence による DNS Sinkhole 構成で、不審な DNS 応答によるフィッシングサイト等への誘引を防止する。
  * **Layer 7 / SSL / TLS 暗号化通信制御 ＆ 正当なインライン復号 (Managed MitM)**:
    * **SSL Stripping / Session Hijacking 緩和**: HTTP へのダウングレード攻撃やセッションハイジャックを防ぐため、HSTS (HTTP Strict Transport Security) の強制や全ポート SSL/TLS インスペクションを適用する。
    * **正当な Managed Inline MitM 復号処理**: FTD (FMC) や WSA が暗号化トラフィックの中間者（Proxy）となり、Outbound 通信に対しては **`Decrypt - Resign`**（内部 CA による証明書再署名）、Inbound サーバー保護に対しては **`Decrypt - Known Key`**（サーバー秘密鍵のインポート）を適用して、隠蔽されたマルウェアや脆弱性攻撃（Snort IPS）を検知・遮断する。
* **利用目的**:
  * 攻撃者による通信データの盗聴（Eavesdropping）、クレデンシャル盗難、セッションハイジャック、データ改ざんを無効化する。
  * 暗号化通信（HTTPS / TLS）の中に隠蔽された未知の脅威やデータ漏洩（Data Exfiltration）を、セキュリティ機器が正当なプロキシとして介入・復号することで可視化・検査する。
* **どのような場面で利用するか**:
  * アクセス層スイッチにおけるユーザー端末間の横移動（Lateral Movement）および盗聴防止。
  * 企業インターネット境界における全社 outbound HTTPS トラフィックのディープパケットインスペクション (DPI)。
  * DMZ 内の公開 HTTPS サーバー群に対するインバウンド攻撃インスペクション。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **対象攻撃手法** | ARP Cache Poisoning、Rogue DHCP/RA、DNS Poisoning/Hijacking、SSL Stripping、Session Hijacking |
| **L2 防御技術** | **Dynamic ARP Inspection (DAI)**、**DHCP Snooping**、**IPv6 FHS (RA Guard, DHCPv6 Guard)** |
| **L7 Managed MitM (FTD)** | **SSL Inspection Policy**: `Decrypt - Resign` (Outbound クライアント用), `Decrypt - Known Key` (Inbound サーバー用) |
| **L7 Managed MitM (WSA)** | **HTTPS Proxy / Decryption Policy**: Passthrough / Monitor / Decrypt / Drop |
| **認証基盤・証明書** | 内部 PKI Root CA（クライアントへの信頼された CA 証明書配布が前提）、Server Key Pair |
| **信頼境界データベース** | **DHCP Snooping Binding Table**、**IPv6 SISF Neighbor Table**、**FMC Internal CA / Trustpoints** |
| **主要制限事項** | SSL Decryption はアプライアンスの CPU/メモリリソースを激しく消費する。クライアントに PKI 信頼証明書が未インストールの場合、証明書警告エラー（Certificate Warning）が発生する |

---

## 🏗 動作原理

### 1. 悪意ある L2/L3 MitM 攻撃と防御アーキテクチャ

攻撃者が Gratuitous ARP や Rogue DHCP/RA を用いて被害者端末とデフォルトゲートウェイ（または他ホスト）の間に自身を挿入し、通信を盗聴・改ざんする動作と、スイッチでの遮断フロー：

```text
[ Victim Host A ] (IP: 10.1.1.10, MAC: AAAA)
       │
       ├─────── (Normal Traffic) ────────► [ Default Gateway ] (IP: 10.1.1.254, MAC: GGGG)
       │
[ Attacker Host C ] (IP: 10.1.1.50, MAC: CCCC)
       │
       ├── Send Forged Gratuitous ARP: "10.1.1.254 is at CCCC" (Poison ARP Cache of Host A)
       └── Send Rogue DHCP Offer: "Default Gateway = 10.1.1.50"
       │
       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Switch Security Mitigation Pipeline                                              │
│                                                                                  │
│ 1. DHCP Snooping Inspection:                                                     │
│    └── Untrusted Port receives DHCPOFFER from Attacker ──► BLOCK Rogue DHCP         │
│                                                                                  │
│ 2. Dynamic ARP Inspection (DAI):                                                 │
│    └── Intercepts ARP "10.1.1.254 is at CCCC" on Untrusted Port                      │
│    └── Check against DHCP Snooping DB (10.1.1.254 = GGGG)                            │
│    └── Mismatch detected! ──► DROP Forged ARP Packet & Log Syslog                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

### 2. 正当な Managed Inline MitM (FTD / WSA SSL Decryption) 動作フロー

FTD や WSA がクライアントと宛先 Web サーバーの間に入り、送信元に対しては自前で再署名した証明書、宛先に対しては通常の TLS ハンドシェイクを行って通信を「正当にインターセプト」する動作フロー：

```text
[ Client Browser ] ───(1) HTTPS Request (ClientHello) ───► [ FTD / WSA (Managed MitM) ] ───(2) ClientHello ───► [ Web Server ]
                                                                   │                                                 │
                                                                   │ ◄──(3) ServerHello + Server Cert (Valid) ────────┤
                                                                   │
                                                                   ├── Validate Server Cert with Public CAs
                                                                   ├── Decrypt-Resign Rule Matches!
                                                                   ├── Generate Dynamic Cert signed by Internal CA
                                                                   │
[ Client Browser ] ◄──(4) ServerHello + Dynamic Cert (Internal CA) ┤
       │
  Validate Dynamic Cert against Internal CA (Trusted!)
       │
       ├─── (5) Encrypted Session 1 (Session Key A) ─────────────► [ FTD / WSA ] ─── (6) Encrypted Session 2 (Session Key B) ──► [ Server ]
                                                                         │
                                                                         ▼
                                                            [ Plaintext Payload Exposed ]
                                                            ├── Snort IPS Inspection
                                                            ├── File & AMP Malware Inspection
                                                            └── URL / Content Filtering
```

---

## ⚙ 動作シーケンス

1. **L2/L3 MitM 攻撃検知・遮断シーケンス**:
   * 攻撃者が `Gratuitous ARP` または偽の `ARP Reply` をブロードキャスト/ユニキャスト送信。
   * スイッチポート（Untrusted）上の **DAI エンジン** が ARP パケットを捕捉し、**DHCP Snooping Binding Table**（IP/MAC/Port/VLAN）と照合。
   * 偽造 ARP（IP 10.1.1.254 に対し MAC CCCC を宣言）を検知すると、ハードウェア ASIC で即座にドロップし、`SW_DAI-4-PACKET_DROP` ログを生成。
2. **Managed Inline MitM (FTD SSL Decryption - Decrypt-Resign) シーケンス**:
   * クライアントが外部 HTTPS サイト（例: `[suspicious link removed]`）宛てに `TCP 443 SYN` および TLS `ClientHello` を送信。
   * FTD の Access Control Policy 評価前段で **SSL Inspection Policy** がヒット。
   * FTD はクライアント宛てにはプロキシ（Server 役）、外部 Web サーバー宛てにはクライアント役として別個の 2 つの TLS ハンドシェイクを開始。
   * FTD は外部サーバーの証明書チェーン（証明書有効期限、失効状態/OCSP、ルート CA）を検証。
   * 検証成功後、FTD は FMC 内に登録された **Internal Certificate Authority (CA)** の秘密鍵を用いて、`example.com` の動的サーバー証明書をオンザフライ生成（Re-sign）し、クライアントへレスポンス。
   * クライアント PC に内部 CA 証明書が信頼されたルート証明書機関（Trusted Root CA）として事前展開されていれば、ブラウザエラーを出すことなく TLS トンネルが確立される。
   * FTD データプレーン上でパケットが完全に平文へ復号され、**Snort 3 IPS Engine**、**File Policy (AMP)**、**URL Filtering** に引き渡されて深層走査される。検査完了後、再度 Session B キーで暗号化して外部サーバーへ転送される。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **FTD SSL Inspection Policy (FMC) の 2 大アクション**:
   * **`Decrypt - Resign`**: 社内 LAN クライアントから外部 Web サーバーへの**アウトバウンド通信**に適用。内部 CA（FMC 内蔵 CA または外部 Enterprise Sub-CA）で動的証明書を再署名する。
   * **`Decrypt - Known Key`**: 外部ユーザーから自社 DMZ 内 Web サーバーへの**インバウンド通信**に適用。事前に FMC へインポートした対象 Web サーバーの「秘密鍵 ＋ 証明書ペア」を使用して復号する。クライアント側の証明書エラーは発生しない。
2. **証明書警告（Certificate Warning）と Trustpoint の設計**:
   * ラボ試験で `Decrypt - Resign` を構成する際、FMC 上で Internal CA オブジェクトを作成し、対象 FTD に配布する必要があります。クライアント（テスト PC）で警告を出さない要件がある場合は、FMC Internal CA の PKCS#12 / PEM 証明書を端末の受託ルート証明書ストア（Trusted Root Certification Authorities）へ配置します。
3. **SSL Decryption の除外（Bypass / Do Not Decrypt）ルールの重要性**:
   * 金融系（Banking/Finance）、医療系（Healthcare）、プライバシー保護が必要なカテゴリ、および TLS 相互認証（Mutual TLS / Client Certificates）や Certificate Pinning を使用するアプリケーション（例: Apple / Microsoft Update、Dropbox 等）は、復号すると通信が破断します。
   * SSL Policy 内で **`Do Not Decrypt`** ルールを上位に配置する構成力がラボ試験で評価されます。
4. **WSA (Web Security Appliance) HTTPS Proxy の要点**:
   * WSA で Managed MitM 復号を行うには、`Security Services > HTTPS Proxy` を有効化し、`Decryption Policies` で **`Decrypt`** ルールを設定します。
   * **Invalid Certificate Handling**: 期限切れ証明書（Expired）、未認証 CA（Unrecognized Root）、ホスト名不一致（Mismatched Hostname）時の動作（Drop / Monitor / Decrypt）を個別定義します。
5. **L2 DAI ＆ Rogue DHCP 防止の復習ポイント**:
   * DAI は `ip arp inspection vlan <VLAN>` で有効化し、インフラポート（ルーター/スイッチ間）で **`ip arp inspection trust`** を忘れないこと（忘れがちなポイント）。

---

## 🛠 設定方法

### 1. FMC GUI: FTD SSL Inspection Policy (Managed MitM: Decrypt-Resign ＆ Known Key)

```text
[ Step 1: Internal CA オブジェクトの作成 (Decrypt-Resign 用) ]
1. FMC UI で Objects > Object Management > PKI > Internal CAs に移動。
2. "Generate CA" をクリック:
   - Name: FMC_INTERNAL_CA
   - Country: US, Organization: Lab, Common Name: FMC Root CA
3. Save を押し、証明書を生成。

[ Step 2: Internal Cert / Key オブジェクトの作成 (Decrypt-Known Key 用) ]
1. Objects > Object Management > PKI > Internal Certs に移動。
2. "Add Internal Cert" をクリック:
   - Name: DMZ_WEB_SERVER_CERT
   - Certificate ＆ Private Key: DMZ Web サーバーの PEM / PKCS#12 ファイルをペースト。

[ Step 3: SSL Policy の作成とルール追加 ]
1. Policies > Access Control > SSL を開き "Add SSL Policy" をクリック (Name: SSL_MITM_POLICY)。
2. ルール 1 (Outbound Decrypt-Resign):
   - Name: DECRYPT_OUTBOUND_WEB
   - Action: Decrypt - Resign
   - With: FMC_INTERNAL_CA
   - Source Zone: Inside_Zone, Destination Zone: Outside_Zone
   - Category / URL: (必要に応じて限定)
3. ルール 2 (Inbound Decrypt-Known Key):
   - Name: DECRYPT_INBOUND_SERVER
   - Action: Decrypt - Known Key
   - With: DMZ_WEB_SERVER_CERT
   - Destination Network: DMZ_SERVER_IP (10.1.100.50)
4. ルール 3 (Bypass Financial/Health):
   - Name: DO_NOT_DECRYPT_EXEMPTIONS
   - Action: Do Not Decrypt
   - Category: Finance and Banking, Health and Medicine (上位に配置)

[ Step 4: Access Control Policy への SSL Policy バインド ＆ Deploy ]
1. Policies > Access Control で対象 ACP を編集。
2. 上部の "SSL Policy:" フィールドで "SSL_MITM_POLICY" を選択。
3. Save して Deploy を実行。
```

---

### 2. WSA CLI / GUI: HTTPS Proxy ＆ Decryption Policy (Managed MitM) の設定

```text
! ===== WSA GUI 操作手順 =====
1. Security Services > HTTPS Proxy を選択し "Enable HTTPS Proxy Functionality" にチェック。
2. "Generate System Root Community Certificate" を実行し、WSA 自署名 CA 証明書を作成。
3. Invalid Certificate Handling オプションの設定:
   - Expired Certificate: Drop
   - Unrecognized Root Authority: Drop
   - Mismatched Hostname: Drop
4. Web Security Manager > Decryption Policies を開く:
   - Default Group ログを "Decrypt" に変更、または特定 Custom Category に対し "Decrypt" アクションをアサイン。
5. "Commit Changes" をクリック。
```

---

### 3. Switch CLI: L2 MitM (ARP Poisoning ＆ Rogue DHCP) 阻止設定

```bash
! ===== Step 1: DHCP Snooping の設定 (Rogue DHCP 防止) =====
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option

interface GigabitEthernet1/0/24          ! Uplink / Authorized DHCP Server Port
 ip dhcp snooping trust

! ===== Step 2: Dynamic ARP Inspection の設定 (ARP Poisoning 防止) =====
ip arp inspection vlan 10,20
ip arp inspection validate src-mac dst-mac ip

interface GigabitEthernet1/0/24          ! Uplink Port
 ip arp inspection trust
```

---

## 🔍 検証コマンド

| 目的 | コマンド (FTD CLI / WSA / IOS) | 表示確認ポイント |
| :--- | :--- | :--- |
| **FTD SSL Decryption セッション確認** | <code>show ssl-decrypt statistics</code> / <code>show asp drop</code> | 復号されたアクティブセッション数、エラー数 |
| **FTD SSL 復号リアルタイムデバッグ** | <code>system support firewall-engine-debug</code> | `SSL Policy Rule Match`, `Decrypt-Resign Action` メッセージ |
| **FMC 上での SSL 復号イベント確認** | `Analysis > Connections > Events` | `SSL Actual Action` フィールド (`Decrypt (Resign)` / `Decrypt (Known Key)`) |
| **WSA HTTPS Proxy 復号ログ確認** | <code>tail accesslogs</code> (WSA CLI) | `DECRYPT_ADMIN` または `DECRYPT_WEBCAT` アクションの出力 |
| **L2 DAI 侵入ドロップ履歴確認** | <code>show ip arp inspection vlan 10</code><br><code>show ip arp inspection log</code> | DAI ドロップカウンター増分、不正 ARP 送信元 MAC/IP アドレス |
| **DHCP Snooping バインド DB 確認** | <code>show ip dhcp snooping binding</code> | 正しく動的学習された MAC/IP/VLAN/Port マッピング |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **`Decrypt - Resign` 設定後、全クライアントのブラウザで「証明書エラー / 接続は不安全です」が発生する** | FTD/FMC の Internal Root CA 証明書が、クライアント PC の「信頼されたルート証明書機関」にインポートされていない | ブラウザの証明書詳細確認 | FMC から Root CA 証明書 (.crt/.pem) をエクスポートし、Active Directory Group Policy (GPO) 等でクライアント端末へ一括配布する。 |
| **特定アプリ（Dropbox, Apple Update 等）が `Decrypt - Resign` 適用後に通信不能になる** | アプリケーションが Certificate Pinning（特定証明書のみをハードコード検証）を実装している | <code>system support firewall-engine-debug</code> | SSL Inspection Policy 内で該当ドメイン/アプリ宛て通信を **`Do Not Decrypt`** アクションで例外除外する。 |
| **`Decrypt - Known Key` を構成したが、インバウンド HTTPS トラフィックが復号されない** | 登録した Certificate と Private Key が不一致、または Diffie-Hellman (ECDHE) キー交換で Forward Secrecy が有効になっている | FMC SSL Policy イベント | サーバー側で DHE/ECDHE シファーを無効化し RSA キー交換を使用させるか、インライン対応の設定を行う。 |
| **DAI 有効化後、全端末の通信が切断される** | スイッチの Uplink ポート（ルーター宛て等）で `ip arp inspection trust` が未設定 | <code>show ip arp inspection interfaces</code> | インフラ側 Uplink ポートで **`ip arp inspection trust`** を設定する。 |
| **WSA で HTTPS 復号がヒットせず素通りする** | WSA 上で `HTTPS Proxy` のグローバル有効化チェックボックスがオフになっている | WSA UI `Security Services > HTTPS Proxy` | WSA の HTTPS Proxy 機能を有効化し、証明書を生成して Commit する。 |

---

## ⚠ 制限事項

* **Diffie-Hellman (ECDHE / PFS) と Decrypt-Known Key の制限**:
  * Perfect Forward Secrecy (PFS) を使用する暗号スイート (ECDHE 等) の場合、`Decrypt - Known Key`（受動的/パッシブ鍵解読）が原理的に不可能です。インライン Managed MitM アプライアンスとして FTD/WSA がハンドシェイク自体を仲介する必要があります。
* **SSL Decryption による性能低下 (Performance Impact)**:
  * SSL/TLS の非対称鍵計算（RSA/ECC）および対称鍵復号は、アプライアンスの CPU に甚大な負荷をかけます。スループットが大幅に低下（最大 50〜70% 減下）するため、全トラフィック復号ではなく重要なカテゴリに絞った設計が必要です。
* **DAI / DHCP Snooping の非 DHCP 端末手動登録負荷**:
  * 静的 IP 環境では、手動で static IP-MAC binding や ARP ACL を管理・登録し続ける運用オーバーヘッドが発生します。

---

## 🔄 他技術との関連

* **Snort IPS Engine**: SSL Inspection Policy で Managed MitM 復号された平文ストリームに対して、脆弱性攻撃シグネチャ（Exploit Rules）を走査。
* **AMP for Firepower / File Policy**: 復号されたファイル転送（HTTP/HTTPS）に対し、リアルタイムで SHA-256 ハッシュ計算およびマルウェア判定。
* **Cisco WSA (Web Security Appliance)**: L7 Web プロキシとして 2 段階の Managed MitM 復号および DLP (Data Loss Prevention) スキャンを実行。
* **Active Directory / GPO**: 社内全クライアント PC へ FMC/WSA の Internal Root CA 証明書を無警告で信頼配置するための PKI ディストリビューション基盤。

---

## 🧩 比較表

### 1. 悪意ある MitM 攻撃 vs 正当な Managed Inline MitM 復号

| 比較項目 | 悪意ある MitM 攻撃 (Attacker MitM) | 正当な Managed Inline MitM (FTD/WSA Decryption) |
| :--- | :--- | :--- |
| **実行主体** | 攻撃者 (Attacker) | 企業のセキュリティアプライアンス (FTD / WSA) |
| **目的** | 盗聴、データ改ざん、クレデンシャル窃取 | 暗号化通信内のマルウェア・IPS 脆弱性攻撃の可視化・阻止 |
| **実現手法** | ARP Poisoning、Rogue DHCP/RA、DNS Hijacking | **`Decrypt - Resign`** / **`Decrypt - Known Key`** |
| **証明書信頼** | 不正証明書のためブラウザで警告画面が発生 | 内部 PKI CA を端末に事前配布することで**無警告で透明処理** |
| **防御/統制策** | DAI、DHCP Snooping、RA Guard、DNS Guard | SSL Inspection Policy、Do Not Decrypt 例外リスト |

### 2. FTD SSL Action: Decrypt - Resign vs Decrypt - Known Key

| 比較項目 | Decrypt - Resign | Decrypt - Known Key |
| :--- | :--- | :--- |
| **適用方向** | **Outbound**（内部 LAN クライアント ➔ 外部 Web サイト） | **Inbound**（外部ユーザー ➔ 内部 DMZ Web サーバー） |
| **必要オブジェクト** | **Internal CA** (秘密鍵付き Root / Sub-CA 証明書) | **Internal Cert** (対象 Web サーバーの証明書 ＋ 秘密鍵) |
| **証明書生成** | 送信先ドメインの証明書を FTD が動的に自動生成・再署名 | サーバーに設定された本物の証明書・鍵を使って復号 |
| **クライアント設定** | クライアント端末へ Internal CA の事前信頼配布が必要 | クライアント側の特別な設定や証明書インポートは**不要** |

---

## 💡 ベストプラクティス

1. **SSL Decryption における厳格な Bypass (Do Not Decrypt) カテゴリ設定**:
   * プライバシー法規制（GDPR / HIPAA 等）や認証切断事故を防ぐため、`Finance and Banking`、`Health and Medicine`、および Certificate Pinning アプリケーション宛て通信は、SSL Policy の最上位で **`Do Not Decrypt`** アクションとしてアサインします。
2. **L2 アクセス層での DAI ＆ DHCP Snooping 必須標準展開**:
   * ネットワーク内の全アクセススイッチにおいて、`ip dhcp snooping` ＋ `ip arp inspection vlan` を標準有効化し、インフラポート宛てにのみ `trust` を与えることで、L2 レイヤーでの MitM 成立可能性を 0% に抑え込みます。

---

## 📝 ラボ学習・設定サンプル例

### 例1: FMC 上での Internal CA 作成と Decrypt-Resign SSL Policy 構成

#### 問題
FMC 上で自署名 Internal CA `FMC_LAB_CA` を作成し、Outbound HTTPS アクセスに対して Managed MitM 復号を行う SSL Policy `SSL_POLICY_OUTBOUND` を構成せよ。

#### 要件
* Action: `Decrypt - Resign`
* Issuer: `FMC_LAB_CA`
* 金融カテゴリ (`Finance and Banking`) は復号対象外 (`Do Not Decrypt`) とすること。

#### 設定例

```text
[ FMC GUI Configuration Steps ]
1. Objects > Object Management > PKI > Internal CAs ➔ "Generate CA"
   - Name: FMC_LAB_CA, Common Name: Lab Internal Root CA
2. Policies > Access Control > SSL ➔ "Add SSL Policy" (Name: SSL_POLICY_OUTBOUND)
3. Add Rule 1:
   - Name: BYPASS_FINANCE
   - Action: Do Not Decrypt
   - Category: Finance and Banking
4. Add Rule 2:
   - Name: DECRYPT_ALL_OUTBOUND
   - Action: Decrypt - Resign
   - With: FMC_LAB_CA
   - Source Zone: Inside_Zone, Destination Zone: Outside_Zone
5. Access Control Policy を編集し、SSL Policy に "SSL_POLICY_OUTBOUND" をアサインの上 Deploy。
```

#### 検証方法
* クライアント PC から `[suspicious link removed]` にアクセスし、証明書の発行元（Issuer）が `Lab Internal Root CA` に変化していること、および FMC の Connection Events で `SSL Actual Action: Decrypt (Resign)` と出力されることを確認。

---

### 例2: FTD インバウンド サーバー保護のための Decrypt - Known Key 構成

#### 問題
DMZ 内の Web サーバー (`10.1.100.50`) への外部アクセスをインスペクションするため、サーバーの証明書・秘密鍵を FMC へ登録し `Decrypt - Known Key` ルールを構成せよ。

#### 要件
* SSL Action: `Decrypt - Known Key`
* Target: Destination IP `10.1.100.50`

#### 設定例

```text
[ FMC GUI Configuration Steps ]
1. Objects > Object Management > PKI > Internal Certs ➔ "Add Internal Cert"
   - Name: DMZ_WEB_CERT
   - Certificate / Key: サーバーの Key/Cert ペアをインポート。
2. Policies > Access Control > SSL ➔ 対象 SSL Policy に Rule を追加:
   - Name: DECRYPT_INBOUND_DMZ
   - Action: Decrypt - Known Key
   - With: DMZ_WEB_CERT
   - Destination Network: 10.1.100.50
3. ACP を Deploy。
```

#### 検証方法
* 外部から `[suspicious link removed]` にアクセスし、クライアント側で一切エラーが出ないこと、および FMC イベントで `SSL Actual Action: Decrypt (Replace Key / Known Key)` と表示されることを確認。

---

### 例3: Switch Dynamic ARP Inspection (DAI) による ARP Poisoning MitM 防御

#### 問題
スイッチ SW1 の VLAN 10 において、DAI を有効化して ARP Cache Poisoning 攻撃を防止せよ。

#### 要件
* Uplink ポート `Gi1/0/24` を DAI Trust ポートに設定すること。
* 源泉/宛先 MAC および IP アドレスのサニティチェックを有効化すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option

ip arp inspection vlan 10
ip arp inspection validate src-mac dst-mac ip

interface GigabitEthernet1/0/24
 ip dhcp snooping trust
 ip arp inspection trust
```

#### 検証方法
* 不正な ARP パケット（偽造 Gratuitous ARP）を送出し、`show ip arp inspection vlan 10` の Invalid ARP カウンターが増加することを確認。

---

### 例4: DHCP Snooping による Rogue DHCP サーバー MitM 防御

#### 問題
VLAN 10 内で攻撃者が不正な DHCP サーバーを起動し、偽のデフォルトゲートウェイをアサインする攻撃を防ぐため DHCP Snooping を設定せよ。

#### 要件
* 正規 DHCP サーバーポート `Gi1/0/24` のみを Trust ポートに設定すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option

interface GigabitEthernet1/0/24
 ip dhcp snooping trust
```

#### 検証方法
* Untrusted アクセスポートに Rogue DHCP サーバーを接続して Offer パケットを送信させ、`show ip dhcp snooping` でパケットがドロップされることを確認。

---

### 例5: WSA HTTPS Proxy における Managed MitM 復号の設定

#### 問題
WSA において、HTTPS Proxy 機能を有効化し、`Decryption Policies` で全 Web アクセスに対し Managed MitM 復号を実施せよ。

#### 要件
* WSA システムルート証明書を生成・適用すること。

#### 設定例

```text
! ===== WSA GUI Configuration =====
1. Security Services > HTTPS Proxy ➔ "Enable HTTPS Proxy Functionality" にチェック。
2. "Generate System Root Community Certificate" を実行。
3. Web Security Manager > Decryption Policies ➔ Global Custom Policy 編集:
   - Action: Decrypt
4. Commit Changes を実行。
```

#### 検証方法
* クライアントから WSA 経由で HTTPS 通信を行い、WSA CLI の `tail accesslogs` で `DECRYPT_ADMIN` アクションが記録されることを確認。

---

### 例6: IPv6 RA Guard による Rogue RA (MitM ゲートウェイ乗っ取り) 防御

#### 問題
スイッチ SW3 の VLAN 11 において、不正な IPv6 Router Advertisement (RA) パケットを遮断するため、アクセスポート宛てに `RA Guard (device-role host)` ポリシーをアサインせよ。

#### 要件
* ルーターポート `Gi1/0/24` のみ `device-role router` を許可すること。

#### 設定例

```bash
! ===== SW3 Configuration =====
ipv6 nd raguard policy POLICY_ROUTER
 device-role router

ipv6 nd raguard policy POLICY_HOST
 device-role host

interface GigabitEthernet1/0/24
 ipv6 nd raguard attach-policy POLICY_ROUTER

vlan configuration 11
 ipv6 nd raguard attach-policy POLICY_HOST
```

#### 検証方法
* `show ipv6 snooping policies` を実行し、ポートおよび VLAN へのアサイン状態を確認。

---

### 例7: ASA / FTD DNS Inspection による DNS Poisoning (RD Flag) 防御

#### 問題
ASA において、DNS キャッシュポイズニング攻撃を最小化するため、DNS インスペクションクラスで Recursion Desired (RD) フラグがセットされたパケットをドロップせよ。

#### 要件
* MPF DNS Inspection を使用すること。

#### 設定例

```text
! ===== ASA Configuration =====
policy-map type inspect dns PM_DNS_INSPECT
 match flag RD
  drop

policy-map PM_GLOBAL
 class inspection_default
  inspect dns PM_DNS_INSPECT
```

#### 検証方法
* `show service-policy inspect dns` を実行し、`drop-connection` カウンターを確認。

---

### 例8: 静的 IP アドレス端末用の Static ARP ACL ＆ DAI 適用

#### 問題
DHCP を使用しない静的 IP サーバー (`10.1.1.200` / MAC: `0050.5611.2233`) が接続された VLAN 10 において DAI を機能させるため、Static ARP ACL を作成・アサインせよ。

#### 要件
* `arp access-list` コマンドを使用すること。

#### 設定例

```bash
! ===== SW1 Configuration =====
arp access-list DAI_SERVERS_ACL
 permit ip host 10.1.1.200 mac host 0050.5611.2233

ip arp inspection filter DAI_SERVERS_ACL vlan 10 static
```

#### 検証方法
* `show ip arp inspection vlan 10` で `ACL Match: DAI_SERVERS_ACL` と表示されることを確認。

---

### 例9: FTD SSL Policy での Certificate Status (OCSP / Revocation) 検証の強制

#### 問題
FTD の Managed MitM 復号処理において、外部 Web サーバーの証明書が失効（Revoked）している場合に接続を自動ドロップするよう SSL Policy ギャップ設定を変更せよ。

#### 要件
* FMC SSL Policy の `Undecryptable Actions` 設定画面から構成すること。

#### 設定例

```text
[ FMC GUI Steps ]
1. FMC SSL Policy の "Undecryptable Actions" タブを開く。
2. "Revoked Server Certificate": Block
3. "Expired Server Certificate": Block
4. "Untrusted CA Server Certificate": Block
5. Save して Deploy を実行。
```

#### 検証方法
* 失効したテスト証明書を持つサイトへ接続し、FTD がハンドシェイクを即座に破棄することを確認。

---

### 例10: FMC REST API による SSL Inspection Policy 設定の取得

#### 問題
FMC REST API を使用して、設定されている全 SSL Policy の一覧情報を GET リクエストで取得せよ。

#### 要件
* `/api/fmc_config/v1/domain/{domainUUID}/policy/sslpolicies` エンドポイントを使用すること。

#### 設定例

```bash
# ===== Auth Token 取得後、SSL Policy 一覧を GET =====
curl -k -X GET "https://fmc.corp.local/api/fmc_config/v1/domain/e276abec-e0f2-11e3-8169-6d9ed13b206f/policy/sslpolicies" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json"
```

#### 検証方法
* 返却された JSON レスポンス内に SSL Policy の Name および UUID が正常に含まれていることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: FTD において外部 Web サイト宛ての HTTPS トラフィックに対し SSL Inspection Policy を構成した。社内 PC から `https://www.cisco.com` にアクセスした際、ブラウザ上に「この接続は非推奨/不安全です (Untrusted Authority)」という警告画面が表示された。FTD 側では `Decrypt - Resign` ルールが正しく適用されている。原因として最も適切なものはどれか。

* A) FTD 上で Snort エンジンが停止しているため。
* B) FTD の `Decrypt - Resign` で使用されている Internal Root CA 証明書が、クライアント PC の「信頼されたルート証明書機関 (Trusted Root CA Store)」にインポートされていないため。
* C) 外側のルーターで uRPF が有効化されているため。
* D) FTD のライセンスが Base ライセンスのみであるため。

**正解**: **B**  
**解説**: **`Decrypt - Resign`** 動作では、FTD が自前の Internal CA を使って動的に証明書を再署名してクライアントへ返却します。クライアント PC 側にその Internal CA 証明書が Root CA として事前登録（信頼）されていない場合、ブラウザは「未知の認証局による偽造証明書」と判断して警告画面を出力します。

---

### 2. トラブルシュート問題

**問題**: スイッチ SW1 の VLAN 20 で Dynamic ARP Inspection (DAI) を有効化した。その後、VLAN 20 内の全端末間で一切の IP 通信（Ping）ができなくなった。`show ip arp inspection vlan 20` を確認したところ、`Operation: Active` と表示されていたが、`show ip dhcp snooping binding` は空であった。原因として正しいものはどれか。

* A) スイッチの電源容量が不足しているため。
* B) 対象 VLAN で DHCP Snooping が有効化されておらず（またはバインド DB が空）、DAI が全 ARP パケットを「無効 (Invalid)」とみなしてドロップしているため。
* C) ポートが Trunk モードになっているため。
* D) CoPP ポリシーが失敗しているため。

**正解**: **B**  
**解説**: **DAI は DHCP Snooping Binding Table に完全依存** します。DHCP Snooping が未構成、またはバインド DB に端末情報が存在しない場合、DAI はすべての着信 ARP パケットを「データベース不一致」としてドロップするため通信不能に陥ります。

---

### 3. Design 問題

**問題**: 外部の不特定多数のユーザーから自社 DMZ 内の特定 Web サーバー (`[suspicious link removed]`) へのインバウンド HTTPS アクセスに対し、FTD でインスペクション（Managed MitM 復号）を行いたい。外部ユーザーのブラウザ側で一切の証明書警告エラーを発生させずにインスペクションを実現するための最適な SSL Policy アクションはどれか。

* A) `Decrypt - Resign`
* B) `Decrypt - Known Key` (Web サーバーの証明書 ＋ 秘密鍵を FMC へインポート)
* C) `Do Not Decrypt`
* D) `Block with reset`

**正解**: **B**  
**解説**: インバウンドの自社 Web サーバー保護には **`Decrypt - Known Key`** を使用します。本物のサーバー証明書と秘密鍵を FMC に登録してパッシブ/インライン復号するため、外部ユーザー側で証明書エラーや警告が発生することはありません。

---

### 4. 実装問題

**問題**: FTD の SSL Inspection Policy において、特定の金融機関サイト (`Finance and Banking` カテゴリ) 宛ての通信を復号インスペクションから明示除外（パススルー）するためのルールアクションはどれか。

* A) `Block`
* B) `Do Not Decrypt`
* C) `Decrypt - Resign`
* D) `Fastpath`

**正解**: **B**  
**解説**: SSL Policy で通信を復号せずにそのまま通過させるための明示アクションは **`Do Not Decrypt`** です。プライバシー保護や金融系サイト宛て通信で必須設定となります。

---

### 5. コンフィグ読解問題

**問題**: スイッチ上で以下のコンフィグを投入した。

```bash
ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option
```

`no ip dhcp snooping information option` コマンドを設定した主な目的はどれか。

* A) DHCP パケットへの Option 82 (Relay Agent Info) 自動挿入を停止し、直接接続された Cisco IOS DHCP サーバーが `giaddr=0` かつ Option 82 付きパケットを拒否する問題を回避するため。
* B) DAI を無効化するため。
* C) Port Security を起動するため。
* D) IP アドレスの割り当てを停止するため。

**正解**: **A**  
**解説**: スイッチで DHCP Snooping を有効にするとデフォルトで Option 82 が挿入されます。直結された IOS DHCP サーバーはこれを拒否するため、スイッチ側で **`no ip dhcp snooping information option`** を設定して不整合を防止します。

---

## 🔗 参考リソース

* [Cisco Secure Firewall Management Center Configuration Guide: SSL Inspection Policies](https://www.cisco.com/)
* [Cisco Web Security Appliance (WSA) User Guide: HTTPS Proxy and Decryption Policies](https://www.cisco.com/)
* [Cisco Catalyst Security Configuration Guide: Dynamic ARP Inspection (DAI)](https://www.cisco.com/)
* [Cisco Technical Notes: Configure and Troubleshoot SSL Decryption on FTD](https://www.cisco.com/)
* [Cisco Live: Deep Dive into SSL/TLS Inspection and Evasion Prevention (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で MitM 関連トピックが出題された場合の構成選択フローチャート：
    * **「Prevent Layer 2 MitM (ARP Poisoning / Rogue DHCP)」** ➔ スイッチで `ip dhcp snooping` ＋ `ip arp inspection vlan` ＋ Uplink ポートで `trust`。
    * **「Inspect Outbound HTTPS traffic without user warning」** ➔ FMC 上で Internal CA 作成 ➔ SSL Policy で `Decrypt - Resign` ➔ クライアント端末へ Root CA 証明書をインストール。
    * **「Inspect Inbound HTTPS traffic to DMZ Web Server」** ➔ FMC 上で Server Key/Cert インポート ➔ SSL Policy で `Decrypt - Known Key`。
    * **「Bypass SSL Inspection for Banking sites」** ➔ SSL Policy 上位ルールで Category: `Finance and Banking` ➔ `Do Not Decrypt`。
* **図解（MitM 対策レイヤー別マトリクス）**:
  ```text
  [ Layer 2 Access ] ──► DHCP Snooping (Block Rogue DHCP)
                      ──► Dynamic ARP Inspection (Block ARP Poisoning)
                      ──► IPv6 RA Guard (Block Rogue IPv6 RA)

  [ Layer 3 / DNS ]  ──► DNS Inspection / RD Flag Filter (Block DNS Poisoning)
                      ──► Cisco Umbrella DNS Sinkhole

  [ Layer 7 / SSL ]  ──► FTD SSL Policy Decrypt-Resign (Outbound Managed MitM)
                      ──► FTD SSL Policy Decrypt-Known Key (Inbound Managed MitM)
                      ──► WSA HTTPS Proxy Decryption
  ```
* **注意点**:
  * ラボ試験で `Decrypt - Resign` や `Decrypt - Known Key` を設定した後は、実際に HTTPS 通信を発生させ、FMC の `Analysis > Connections > Events` で **`SSL Actual Action`** フィールドが意図通りのアクション（`Decrypt (Resign)` または `Decrypt (Replace Key)`）として記録されていることを必ず確認・証明してください。
