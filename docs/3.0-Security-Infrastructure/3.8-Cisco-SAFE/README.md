---
layout: default
title: 3.8-Cisco-SAFE
nav_order: 8
parent: 3.0-Security-Infrastructure
---

# 3.8 Cisco SAFE model to validate network security design and to identify threats to different PINs

**Cisco SAFE (Security Architecture for the Enterprise)** は、ネットワーク全体のセキュリティ設計を簡素化し、標準化するためのアーキテクチャフレームワークです。CCIE Security v6.1 のブループリントにおいて、この項目は「設計の検証」と「PIN (Places in the Network) ごとの脅威の特定」に焦点を当てています。単なる理論ではなく、インフラストラクチャ全体（Campus, Edge, Data Center, Cloud 等）を包括的に保護するための実装ガイドラインとして理解する必要があります。

---

## 📘 概要

*   **機能概要**: 複雑なセキュリティ設計を「ビジネスの流れ」に基づいた論理的な構成要素に分解し、一貫したセキュリティ能力（Secure Capabilities）を適用するモデルです。
*   **利用目的**: 組織全体のセキュリティポスチャを可視化し、特定のネットワークエリア（PIN）に存在する特有の脅威に対して適切な対策がなされているかを検証します。
*   **どのような場面で利用するか**: 
    *   **設計検証**: 提案されたネットワーク構成に「可視性（Visibility）」や「セグメンテーション（Segmentation）」の欠如がないかを確認する際。
    *   **脅威分析**: 拠点（Branch）やデータセンター（Data Center）など、場所ごとに異なる攻撃ベクトル（スプーフィング、DoS、マルウェア等）を特定する際。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **PIN (Places in the Network)** | Campus, Branch, Edge, Cloud, Data Center, WAN。 |
| **Secure Capabilities** | Visibility, Segmentation, Patch Management, Intrusion Prevention, etc。 |
| **脅威の種類** | Spoofing (IP/MAC/ARP), DoS/DDoS, MiM, VLAN Hopping, Malware。 |
| **検証ツール** | Stealthwatch (Network Analytics), DNA Center, FMC。 |
| **CCIE での役割** | 個別の技術 (3.1-3.7) をアーキテクチャとして統合し、設計の妥当性を判断する。 |

---

## 🏗 動作原理

SAFE モデルは、防御、検知、対応のサイクルを各 PIN に適用することで動作します。

```text
[ Asset / Data ]
      ↓
[ Identification of Threats ] (Spoofing, DoS, Reconnaissance)
      ↓
[ Secure Capabilities Application ]
   - Device Hardening
   - Control Plane Protection (CoPP)
   - Data Plane Protection (uRPF, ACLs)
      ↓
[ Validation / Telemetry ] (NetFlow, Syslog, Stealthwatch)
```

1.  **PIN の定義**: 保護対象がどこに位置するかを特定します。
2.  **脅威の特定**: その PIN 特有の脆弱性を分析します（例：Edge でのスプーフィング）。
3.  **セキュリティ能力の割り当て**: 適切なコントロール（FW, IPS, AAA 等）を配置します。
4.  **継続的検証**: テレメトリ（NetFlow, Logging）を使用して、設計が期待通りに機能しているか監視します。

---

## ⚙ 動作シーケンス (設計検証フロー)

1.  **コンプライアンス確認**: 設計が ISO 27001 や PCI-DSS 等の基準を満たしているか確認します。
2.  **管理プレーンの要塞化**: デバイスへのアクセスがセキュア（SSH, AAA）であることを確認します。
3.  **コントロールプレーンの保護**: CoPP やルーティング認証により、インフラの安定性を確保します。
4.  **データプレーンのフィルタリング**: uRPF や ACL により、不正パケット（RFC 1918 等）を境界で遮断します。
5.  **可視性の確保**: NetFlow や Syslog がコレクタ（Stealthwatch 等）に正しく送信されているかを検証します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **PIN の識別**: 与えられたトポロジにおいて、どのデバイスが "Edge" であり、どこが "Campus" なのかを SAFE の概念に基づいて正しく分類する能力が問われます。
*   **脅威への適切な対策**: 
    *   「Edge PIN でのソース IP 偽装を防げ」 → **uRPF** または **Ingress ACL**。
    *   「Campus での MiM 攻撃を防げ」 → **DAI (Dynamic ARP Inspection)** や **DHCP Snooping**。
*   **設計の脆弱性指摘**: ラボの問題文で「監視が不十分である」と示唆された場合、NetFlow (NSEL) や eStreamer の設定を補完することが求められます。
*   **システムハードニング**: デバイス自体の保護（Management/Control Plane Protection）が SAFE 設計の基盤であることを意識してください。
*   **検証シナリオ**: `show flow monitor` や `show policy-map control-plane` を使用して、セキュリティ機能がパケットを正しく処理・ドロップしていることを証明する必要があります。

---

## 🛠 設定方法

### 1. Edge PIN における境界保護 (uRPF/BCP 38)
```bash
interface GigabitEthernet1
 description OUTSIDE-EDGE
 ip verify unicast source reachable-via rx
 ! 送信元偽装（Spoofing）の防止
```

### 2. Control Plane 脅威への対策 (CoPP)
```bash
ip access-list extended ICMP-ACL
 permit icmp any any
!
policy-map COPP-POLICY
 class ICMP-CLASS
  police 100000 conform-action transmit exceed-action drop
!
control-plane
 service-policy input COPP-POLICY
 ! CPU リソース枯渇（DoS）の防止
```

### 3. 設計検証のためのテレメトリ (NetFlow)
```bash
flow record SAFE-REC
 match ipv4 source address
 match ipv4 destination address
 collect counter bytes long
!
flow monitor SAFE-MON
 record SAFE-REC
!
interface Gi0/1
 ip flow monitor SAFE-MON input
 ! 可視性 (Visibility) の確保
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **コントロールプレーン保護の確認** | <code>show policy-map control-plane</code> |
| **NetFlow キャッシュの確認** | <code>show flow monitor SAFE-MON cache</code> |
| **境界でのドロップパケット確認** | <code>show ip traffic</code> または <code>show access-lists</code> |
| **デバイスハードニング状態** | <code>show control-plane host open-ports</code> |
| **IPv6 セキュリティの検証** | <code>show ipv6 spd</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| 正当なパケットが境界で落ちる | 非対称ルーティングによる uRPF 失敗 | <code>show ip interface</code> | <code>rx</code> から <code>any</code> (Loose) へ変更。 |
| 管理アクセスが不安定 | CoPP での過剰な制限 | <code>show policy-map control-plane</code> | 管理トラフィックを police 対象から外す。 |
| Stealthwatch にデータが来ない | Exporter の設定ミス | <code>show flow exporter statistics</code> | 宛先 IP と UDP ポート (2055) を確認。 |
| VLAN Hopping が発生する | DTP が有効なまま | <code>show interface switchport</code> | <code>switchport nonegotiate</code> を設定。 |

---

## ⚠ 制限事項

*   **パフォーマンスへの影響**: 全てのトラフィックに対して詳細な検査（IPS, SSL Decryption）を適用すると、デバイスのスループットが低下します。
*   **ライセンス**: Firepower (FTD) 等での高度な脅威防御には、Threat/Malware ライセンスが必要です。
*   **ハードウェア支援**: 古いスイッチでは VACL や DAI のハードウェア処理に制限がある場合があります。

---

## 🔄 他技術との関連

*   **3.1.a CoPP**: インフラ保護の直接的な手段。
*   **3.3.a uRPF**: データプレーンのスプーフィング防止。
*   **3.4 Layer 2 Security**: Campus PIN におけるエンドポイントセキュリティ。
*   **3.10 DNAC API**: ネットワーク設計の自動プロビジョニングとテレメトリ収集。

---

## 🧩 比較表

### PIN ごとの主要な脅威と対策

| PIN | 主な脅威 | 推奨される対策 |
| :--- | :--- | :--- |
| **Campus** | MiM, Rogue AP, MAC Flooding | 802.1X, DAI, Port Security |
| **Edge** | IP Spoofing, DDoS, Botnets | uRPF, NGIPS (FTD), BCP 38 |
| **Data Center** | Data Exfiltration, East-West Attack | Micro-segmentation (TrustSec), Firewalling |
| **Branch** | Unauthorized Access, Malware | AnyConnect VPN, Umbrella, Stealwatch |

---

## 💡 ベストプラクティス

1.  **可視性ファースト**: 監視（NetFlow/Syslog）がない設計は SAFE ではありません。
2.  **デフォルト拒否**: 境界（Edge）およびセグメント間では、明示的に許可されたトラフィック以外は破棄します。
3.  **階層化防御**: コントロールプレーン、管理プレーン、データプレーンの各層で個別の保護を適用します。
4.  **継続的な監査**: DNAC や FMC のダッシュボードを使用して、設計の健全性を定期的に検証します。

---

## 📝 ラボ学習・設定サンプル例（3.8 Cisco SAFE / Monitoring / Compliance 統合）

SAFEモデルに基づき、各PIN（場所）における脅威の特定と、それに対する具体的な対策・検証手順を10個のラボ形式でまとめます。

### 1. Edge PIN: 送信元偽装パケットのドロップと記録（BCP 38）
*   **問題**: インターネット境界ルータ（R1）のOutsideインターフェイスにおいて、内部ネットワークからのなりすまし（IP Spoofing）パケットを破棄し、その詳細を記録せよ。
*   **要件**:
    1.  RFC 1918（プライベートIP）が外部から流入するのを遮断。
    2.  uRPF Strictモードを使用して、ルーティングの一致を確認。
*   **設定例**:
    ```bash
    ! 1. 不正な流入を拒否するACLの作成
    ip access-list extended EDGE-INGRESS-FILTER
     deny ip 10.0.0.0 0.255.255.255 any log-input
     deny ip 172.16.0.0 0.15.255.255 any log-input
     deny ip 192.168.0.0 0.0.255.255 any log-input
     permit ip any any
    !
    ! 2. インターフェイスへの適用とuRPFの設定
    interface GigabitEthernet1
     description OUTSIDE-EDGE
     ip access-group EDGE-INGRESS-FILTER in
     ip verify unicast source reachable-via rx  ! Strictモード
    ```
*   **検証方法**: `show ip interface GigabitEthernet1` を実行し、"IP verify source: reachable-via-rx" が有効であることと、ドロップカウンタ（`drops`）が増えているかを確認する。

### 2. Control Plane PIN: インフラ保護のためのCoPP実装
*   **問題**: 大量のICMPパケットによるDoS攻撃からルータのCPUを保護せよ。
*   **要件**:
    1.  ICMPを100Kbpsにレート制限。
    2.  制限を超えた場合はドロップ。
*   **設定例**:
    ```bash
    ! 1. 対象トラフィックの定義
    ip access-list extended ICMP-TRAFFIC
     permit icmp any any
    !
    ! 2. クラスマップの定義
    class-map match-all ICMP-CLASS
     match access-group name ICMP-TRAFFIC
    !
    ! 3. ポリシーマップによる制限
    policy-map COPP-POLICY
     class ICMP-CLASS
      police 100000 conform-action transmit exceed-action drop
    !
    ! 4. コントロールプレーンへの適用
    control-plane
     service-policy input COPP-POLICY
    ```
*   **検証方法**: `show policy-map control-plane` を実行し、ICMPトラフィックに対する `conformed` と `exceeded` パケット数を確認する。

### 3. Campus PIN: L2スプーフィング攻撃の防止（DAI）
*   **問題**: VLAN 10におけるARPポイズニング（MiM攻撃）を防止せよ。
*   **要件**: DHCP Snoopingをベースに、動的なARP検査（DAI）を有効にする。
*   **設定例**:
    ```bash
    ! 1. DHCP Snoopingの有効化
    ip dhcp snooping
    ip dhcp snooping vlan 10
    !
    ! 2. Uplink（信頼できるポート）の設定
    interface GigabitEthernet0/1
     ip dhcp snooping trust
    !
    ! 3. Dynamic ARP Inspectionの有効化
    ip arp inspection vlan 10
    ```
*   **検証方法**: `show ip arp inspection vlan 10` を実行し、"Dropped" カウンタが増加していることを確認する。

### 4. Visibility: ASA NSEL（NetFlow）によるフロー可視化
*   **問題**: ASAを通過するトラフィックを可視化するため、NetFlow (NSEL) を構成せよ。
*   **要件**: 10.1.1.100 のコレクタ（Stealthwatch等）に全フローを送信。
*   **設定例**:
    ```bash
    ! 1. エクスポート先の定義
    flow-export destination inside 10.1.1.100 2055
    !
    ! 2. MPFによる全トラフィックのキャプチャ
    access-list NETFLOW-ACL extended permit ip any any
    !
    class-map NETFLOW-CLASS
     match access-list NETFLOW-ACL
    !
    policy-map global_policy
     class NETFLOW-CLASS
      flow-export event-type all destination 10.1.1.100
    !
    service-policy global_policy global
    ```
*   **検証方法**: `show flow-export counters` を実行し、"Records sent" の数が増加していることを確認する。

### 5. Management PIN: デバイスハードニング（不要サービスの停止）
*   **問題**: 組織のポリシーに基づき、ルータの不要なサービスを停止して攻撃面を最小化せよ。
*   **設定例**:
    ```bash
    ! 不要なサービスの無効化
    no ip http server
    no ip http secure-server
    no ip finger
    no ip source-route
    no service tcp-small-servers
    no service udp-small-servers
    !
    ! 管理プロトコルのセキュア化（SSH強制）
    line vty 0 4
     transport input ssh
     login authentication default
    ```
*   **検証方法**: `show control-plane host open-ports` を実行し、不要なTCP/UDPポートが空いていないことを確認する。

### 6. WAN PIN: GETVPNによる拠点間暗号化（Source 参照）
*   **問題**: フルメッシュ通信が必要なWAN環境で、GETVPNを使用してトラフィックを保護せよ。
*   **設定例 (Key Server)**:
    ```bash
    ! キーサーバーの設定
    crypto gdoi group GETVPN-GROUP
     identity number 1
     server local
      rekey authentication mypubkey rsa KS-KEYS
      rekey transport unicast
      sa ipsec 1
       profile GETVPN-PROF
       match address LAN-LIST
    !
    ip access-list extended LAN-LIST
     permit ip 192.168.0.0 0.0.255.255 192.168.0.0 0.0.255.255
    ```
*   **検証方法**: `show crypto gdoi ks` でKey Serverの状態を確認し、登録されているGM（Group Member）数を確認する。

### 7. Data Center PIN: TrustSec (SGT) によるセグメンテーション（Source 参照）
*   **問題**: IPアドレスに依存しない、ロールベースのアクセス制御をASAに実装せよ。
*   **要件**: SXPを使用してISEからSGTマッピングを取得する。
*   **設定例**:
    ```bash
    ! SXPの有効化とISEとの接続
    cts sxp enable
    cts sxp default password cisco123
    cts sxp default source-ip 192.168.10.10
    cts sxp connection peer 192.168.123.35 password default mode peer speaker
    !
    ! SGTを使用したセキュリティポリシーの適用
    access-list SGT-ACL extended permit tcp security-group tag 10 any security-group tag 20 any eq 3306
    ```
*   **検証方法**: `show cts sxp connections` でISEとの接続が "SXP-Active" であることを確認する。

### 8. Monitoring: 高度なSyslogフィルタリング（Discriminators）（Source 参照）
*   **問題**: OSPFのネイバー切断ログのみを抽出し、特定のSyslogサーバへ送信せよ。
*   **設定例**:
    ```bash
    ! フィルタの定義（"Neighbor Down"が含まれるログを特定）
    logging discriminator OSPF msg-body drops Neighbor Down
    !
    ! 送信設定
    logging host 180.1.55.66 discriminator OSPF
    logging on
    ```
*   **検証方法**: `show logging` を実行し、"Active Message Discriminator" が適用されていることを確認する。

### 9. Branch PIN: IPv6 RA Guardの実装
*   **問題**: ブランチ拠点のL2スイッチで、不正なルータ広告（RA）による偽装デフォルトゲートウェイ攻撃を防止せよ。
*   **要件**: ユーザポート（Gi0/1）でRA Guardを有効にする。
*   **設定例**:
    ```bash
    ! RA Guardポリシーの作成
    ipv6 nd inspection policy MY-POLICY
     device-role host
    !
    ! ポートへの適用
    interface GigabitEthernet0/1
     ipv6 nd ra-guard attach-policy MY-POLICY
    ```
*   **検証方法**: `show ipv6 nd inspection policy` を確認。

### 10. Availability: IP SLAによる冗長パスの監視（Source 参照）
*   **問題**: 外部回線の可用性を監視し、障害時に自動的にルートを切り替えよ。
*   **設定例**:
    ```bash
    ! SLA監視の定義
    ip sla 100
     icmp-echo 4.2.2.2 interface GigabitEthernet0/1
     timeout 1000
     frequency 3
    ip sla schedule 100 life forever start-time now
    !
    ! オブジェクトのトラック
    track 10 rtr 100 reachability
    !
    ! ルーティングへの紐付け
    ip route 0.0.0.0 0.0.0.0 192.1.20.2 track 10
    ip route 0.0.0.0 0.0.0.0 192.1.30.3 10  ! バックアップパス
    ```
*   **検証方法**: `show track 10` を実行し、"State is Up" であることを確認する。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `ip verify unicast source reachable-via any 100` と設定されている場合、ACL 100はどのような役割を持つか？
    *   **回答**: 例外（ホワイトリスト）。ACL 100で許可されたトラフィックは、uRPFのパス検証に失敗しても許可される。
2.  **トラブルシュート**: CoPPを設定したところ、BGPのピアリングが不定期に切断されるようになった。原因として何が考えられるか？
    *   **回答**: CoPPのレート制限（police）が厳しすぎる、またはBGP（TCP 179）が許可クラスに含まれていない。
3.  **Design**: SAFEモデルにおいて、Campus PINの内部脅威を可視化するために、全スイッチポートで実施すべき推奨設定は？
    *   **回答**: **Flexible NetFlow (FNF)** のIngress適用と、コレクタへの転送。
4.  **実装**: 組織のポリシーで「特権モードの操作をすべて記録する」ことが求められた場合、必要なAAAコマンドは？
    *   **回答**: `aaa accounting commands 15 start-stop group tacacs+`。
5.  **Design**: FMCから外部SIEMへ、IDS/IPSのバイナリイベントを高速転送するプロトコルは何か？
    *   **回答**: **eStreamer** (TCP 8302)。

---


## 🔗 参考リソース

*   **Cisco SAFE White Paper**: [SAFE Architectural Guide](https://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/Security/SAFE_RG/SAFE_rg/chap1.html)
*   **Cisco Live (BRKSEC-2003)**: [Securing the Infrastructure: Hardening & Protection](https://www.ciscolive.com/)
*   **CVD**: [Secure Campus Design Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/cisco-secure-campus-design-guide.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: SAFE は「どこ（PIN）を」「何から（Threats）」「どう（Capabilities）守るか」の地図です。
*   **図解**: 
    *   **PIN**: ネットワークを物理・論理的に分けたエリア。
    *   **Secure Capability**: ツールボックス（ACL, IPS, uRPF等）。
*   **注意点**: ラボ試験では、個別のコマンド設定（例：`ip verify ...`）ができても、それが「どの PIN のどの脅威への対策か」を理解していないと、意図しない通信遮断などのトラブルに対応できません。
