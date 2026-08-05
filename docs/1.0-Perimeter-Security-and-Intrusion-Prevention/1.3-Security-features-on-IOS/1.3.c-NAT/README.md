# 1.3.c NAT (Network Address Translation on IOS/IOS XE)

Cisco IOSおよびIOS XEにおける**ネットワークアドレス変換（NAT）**は、IPアドレスの節約（IPv4枯渇対策）、内部ネットワークの隠蔽、および異なるアドレス体系を持つネットワーク間の接続を可能にする不可欠な機能です。CCIE Security v6.1ラボ試験では、単純なアドレス変換だけでなく、VRF、VPN、および高度なポリシーに基づく複雑なNAT実装能力が問われます。

---

## 📘 概要

*   **機能概要**: パケットのIPヘッダ内の送信元または宛先IPアドレス（およびポート番号）を書き換えます。
*   **利用目的**: プライベートIPからパブリックIPへの変換（インターネット接続）、組織統合に伴うIP重複の解決、およびサーバの負荷分散などに利用されます。
*   **どのような場面で利用するか**: 企業拠点のインターネットゲートウェイ、DMVPN環境でのハブ/スポーク通信、およびマルチテナント環境（VRF）でのセグメンテーション維持などに適用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | 送信元/宛先IPの動的・静的変換。PAT（Overload）による多対1変換のサポート。 |
| **用途** | IPv4アドレス節約、セキュリティ境界の保護、IP重複ネットワーク間の通信。 |
| **メリット** | 公認IPアドレスコストの削減、ネットワーク構成変更時の柔軟性向上。 |
| **デメリット** | エンドツーエンドの追跡困難性、一部プロトコル（FTP等）へのインスペクション負荷。 |
| **対応機種** | ISR, ASR, Catalyst 8000V などの IOS/IOS XE デバイス。 |
| **主な方式** | Static NAT, Dynamic NAT, PAT, NAT Virtual Interface (NVI)。 |
| **設計上の注意点** | **NAT処理の順序（Routing前か後か）**、およびVPN（IPsec）との干渉。 |

---

## 🏗 動作原理

Cisco IOS NATは、インターフェイスを「Inside（内部）」と「Outside（外部）」に定義するドメインベース方式が基本です。

```text
[Inside Local] (10.1.1.1)
      ↓
[Inside Interface] (ip nat inside)
      ↓
[NAT Engine] --- (送信元 10.1.1.1 を 203.0.113.1 に変換)
      ↓
[Outside Interface] (ip nat outside)
      ↓
[Inside Global] (203.0.113.1)
```

**NAT Virtual Interface (NVI)** を使用する場合、Inside/Outsideの区別をなくし、`ip nat enable` を設定した全インターフェイス間で変換が可能になります。

---

## ⚙ 動作シーケンス

パケットの流れる方向により、NATとルーティングの処理順序が異なります。

1.  **Inside to Outside**:
    *   パケットの受信（Ingress）
    *   ポリシーベースルーティング (PBR)
    *   **ルーティングルックアップ（Egressインターフェイスの決定）**
    *   **NAT処理（Local から Global へ変換）**
    *   ACL（送信元/宛先チェック）
    *   パケットの送出。
2.  **Outside to Inside**:
    *   パケットの受信（Ingress）
    *   ACL（送信元/宛先チェック）
    *   **NAT処理（Global から Local へ変換/アン・トランスレート）**
    *   **ルーティングルックアップ**
    *   パケットの送出。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **NAT Order of Operation**: ルーティングがNATの前後どちらで行われるかを理解していないと、意図したパケットがドロップされます。
*   **Identity NAT (NAT Exemption)**: VPNトラフィックをNATから除外する要件です。Route-mapを使用して「変換しない（Identity）」ルールを作成するスキルが必須です。
*   **VRF-aware NAT**: 特定のVRFに属するトラフィックをNATする際、`ip nat inside source ... vrf [NAME]` コマンドを正確に指定する必要があります。
*   **Twice NAT (Double NAT)**: 送信元と宛先の両方を同時に変換する構成です。IP重複環境の解決策として問われる可能性があります。
*   **show ip nat translations**: 変換テーブルが正しく生成されているか、タイムアウト値は適切かを確認する重要なコマンドです。

---

## 🛠 設定方法

### 1. 静的NAT (Static NAT)
```bash
# 内部サーバ 192.168.1.10 を 203.0.113.10 に固定変換
ip nat inside source static 192.168.1.10 203.0.113.10
interface GigabitEthernet1
 ip nat inside
interface GigabitEthernet2
 ip nat outside
```

### 2. PAT (Port Address Translation / Overload)
```bash
# 内部ネットワーク全体を外部IFのIPでPAT
access-list 10 permit 192.168.1.0 0.0.0.255
ip nat inside source list 10 interface GigabitEthernet2 overload
```

### 3. Route-map を使用した Identity NAT (VPN用)
```bash
# 10.1.1.0/24 から 172.16.1.0/24 への通信は変換しない
access-list 100 deny ip 10.1.1.0 0.0.0.255 172.16.1.0 0.0.0.255
access-list 100 permit ip 10.1.1.0 0.0.0.255 any
route-map NAT_MAP permit 10
 match ip address 100
ip nat inside source route-map NAT_MAP interface Gi2 overload
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **NAT変換テーブルの確認** | <code>show ip nat translations</code> |
| **NAT統計情報の表示** | <code>show ip nat statistics</code> |
| **リアルタイムNAT変換のデバッグ** | <code>debug ip nat</code> |
| **特定のNATエントリの詳細** | <code>show ip nat translations verbose</code> |
| **変換テーブルのクリア** | <code>clear ip nat translation *</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 変換エントリが作成されない | ACLのマッチングミス、またはIFでのNAT設定漏れ | <code>show ip nat statistics</code> でヒットカウントを確認。 |
| VPN通信がNAT後にドロップされる | NAT Exemptionの設定漏れ | <code>debug ip nat</code> でパケットが変換されているか確認。 |
| 戻りパケットが届かない | ルーティングの不備、または非対称パス | NAT後のIPへの経路が対向ルータにあるか確認。 |
| ポート枯渇による通信断 | PATセッションの上限到達 | <code>show ip nat statistics</code> で <code>max-entries</code> を確認。 |
| ICMPは通るがHTTPが通らない | NATのMTU/フラグメンテーション問題 | <code>ip tcp adjust-mss</code> の設定を検討。 |

---

## ⚠ 制限事項

*   **同一IFでのNAT**: デフォルトでは `ip nat inside` と `outside` は異なるインターフェイスである必要があります（NVIで回避可能）。
*   **NATエントリ数**: デバイスのメモリ容量により、保持できる変換エントリ数に制限があります。
*   **プロトコルの制約**: ペイロード内にIP情報を埋め込むプロトコル（SIP, Skinny等）は、ALG（Application Layer Gateway）のサポートが必要です。

---

## 🔄 他技術との関連

*   **Access Control**: ACLはNATの前後に適用されるため、記述するIPがLocalかGlobalかを順序に従って決める必要があります。
*   **Routing**: NAT後のパケットを送出するために、グローバルIP宛のルートが必要です。
*   **VPN**: IPsecトンネルをNAT越しに張る場合、NAT-Traversal (UDP 4500) が必要です。
*   **High Availability**: HSRP/VRRP環境でNATのステートを同期させる構成（Stateful NAT）が利用されます。

---

## 🧩 比較表

### Traditional NAT vs NAT NVI

| 項目 | Traditional NAT | NAT NVI (Virtual Interface) |
| :--- | :--- | :--- |
| **ドメイン** | Inside/Outside の定義が必要 | Inside/Outside の概念なし |
| **設定コマンド** | `ip nat inside/outside` | `ip nat enable` |
| **ルーティング順序** | 複雑（方向により異なる） | 常にルーティング後にNATを評価 |
| **ユースケース** | 一般的なエッジFW | 非対称ルーティング、高度なセグメンテーション |

---

## 💡 ベストプラクティス

*   **名前付きACLの利用**: 管理性を高めるため、NAT用のACLには `ACL_NAT_INSIDE_TRAFFIC` のような名前を付けます。
*   **Overload（PAT）の適切な使用**: 公衆IPが1つしかない場合、常に `overload` キーワードを確認します。
*   **静的エントリの最小化**: セキュリティリスクを減らすため、必要なサーバポートのみを `static` で公開します。
*   **タイムアウト調整**: 大規模環境では、NATテーブルの溢れを防ぐために `ip nat translation timeout` を調整します。

---

## 📝 ラボ学習・設定サンプル例

### 1. ダイナミックNATプール
*   **要件**: 内部ネットワークを 200.1.1.100 - 200.1.1.110 のプールに変換せよ。
*   **設定**: 
```bash
ip nat pool MY_POOL 200.1.1.100 200.1.1.110 netmask 255.255.255.0
ip nat inside source list 10 pool MY_POOL
```

### 2. ポートフォワーディング（静的PAT）
*   **要件**: 外部からの TCP 8080 へのアクセスを内部 10.1.1.5:80 に転送せよ。
*   **設定**: `ip nat inside source static tcp 10.1.1.5 80 interface Gi2 8080`

### 3. Identity NAT (NAT除外)
*   **要件**: 内部からVPN対向 172.16.0.0/16 への通信をNATから除外せよ。
*   **設定**: Route-mapで対向宛を `deny` し、それ以外を `permit` する。

### 4. VRF-aware NAT
*   **要件**: VRF "SALES" のトラフィックをNATせよ。
*   **設定**: `ip nat inside source list 10 interface Gi2 vrf SALES overload`。

### 5. TCP 負荷分散
*   **要件**: 単一のパブリックIP宛の通信を3台の内部サーバに分散。
*   **設定**: `ip nat pool SERVERS 10.1.1.1 10.1.1.3 type rotary`

### 6. NAT NVI の実装
*   **要件**: Inside/Outsideの区別をなくしNATを有効化。
*   **設定**: インターフェイスで `ip nat enable`。コマンドは `ip nat source list ...`。

### 7. オーバーラップ解決（Double NAT）
*   **要件**: 双方 10.1.1.0/24 を持つ拠点間で通信。
*   **設定**: 送信元と宛先の両方を仮想セグメントにマップする。

### 8. NAT64 (Stateful)
*   **要件**: IPv6クライアントからIPv4サーバへのアクセスを可能にせよ。
*   **設定**: `nat64 v6v4 list ... pool ...`

### 9. 特定のサービスポートのみのNAT
*   **要件**: 10.1.1.10 の DNS (UDP 53) 通信のみを特定のIPで変換。
*   **設定**: `ip nat inside source static udp 10.1.1.10 53 200.1.1.1 53`

### 10. タイムアウト設定のカスタマイズ
*   **要件**: TCPのNAT保持時間を2時間に延長。
*   **設定**: `ip nat translation tcp-timeout 7200`

---

## ❓ 想定試験問題

1.  **実装**: インターフェイス Gi1 (Inside) と Gi2 (Outside) がある。Inside発のパケットが Gi2 のIPで変換されるよう、最小限のコマンドを記述せよ。
2.  **トラブルシュート**: `show ip nat translations` を実行しても何も表示されない。ACL 10 に `permit 10.1.1.0 0.0.0.255` がある場合、他に何を確認すべきか？
    *   **回答**: 各インターフェイスに `ip nat inside/outside` コマンドが設定されているか、およびパケットが実際にルータを通過しているか。
3.  **コンフィグ読解**: `ip nat inside source route-map MAP1 interface Gi2 overload` において、`MAP1` 内の ACL で `deny` されたトラフィックはどう処理されるか？
    *   **回答**: NAT変換されずに（元の送信元IPのまま）ルーティングされる（Identity NAT/Exemption）。
4.  **Design**: IP重複がある2つの企業を統合する際、既存のホスト設定を変えずに通信させるためのNAT手法は何か？
    *   **回答**: Twice NAT (Manual NAT) または Overlapping NAT。
5.  **動作シーケンス**: アウトバウンド方向（Inside to Outside）において、ルーティングルックアップとNAT処理、どちらが先に実行されるか？
    *   **回答**: ルーティングルックアップが先。NATはパケットを送出する直前に行われる。

---

## 🔗 参考リソース

*   **Cisco Live (Slides/Video)**:
    *   [BRKSEC-2021: Firepower Threat Defense - Packet Flow and Troubleshooting](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-2021) (NAT順序の理解に有用)
*   **Configuration Guides**:
    *   [Cisco IOS XE NAT Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/configuration/xe-16/nat-xe-16-book.html)
    *   [Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/13772-12.html)
*   **Technical Notes**:
    *   [NAT Order of Operation (Cisco Support)](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/6209-5.html)
*   **Command Reference**:
    *   [Cisco IOS IP Address Reference - NAT Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/command/nat-cr-book.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: IOSのNATはASAと異なり、インターフェイス設定（inside/outside）が必須である点が最大の違いです。
*   **図解**: 常にパケットが入るIFと出るIFを意識し、その間の「NATポイント」を想像して設定してください。
*   **注意点**: `debug ip nat` は高負荷時にCPUを占有するため、ラボ試験でも特定のトラフィックに絞ったACLを適用して使用することが推奨されます。
