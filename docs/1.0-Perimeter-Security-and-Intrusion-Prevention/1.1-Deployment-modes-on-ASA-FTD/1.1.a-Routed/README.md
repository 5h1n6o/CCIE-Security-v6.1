---
layout: default
title: 1.1.a-Routed
nav_order: 1
parent: 1.1-Deployment-modes-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.1.a Routed Mode

Cisco ASAおよびCisco Firepower Threat Defense (FTD) における**ルーテッドモード (Routed Mode)** は、デバイスがネットワークパス内の**レイヤ3 (L3) ホップ**として動作する、最も一般的かつデフォルトの展開モードです。この学習メモでは、CCIE Security v6.1ラボ試験で求められる深い技術理解と実装、トラブルシューティング手法について整理します。

---

# 📘 概要

*   **機能概要**: ルーテッドモードでは、ファイアウォールの各インターフェイスが異なるIPサブネットに属し、デバイス自体がルーティングテーブル (RIB) を保持してトラフィックを転送します。
*   **利用目的**: 異なる信頼レベルを持つネットワーク (Inside, Outside, DMZなど) を分離し、セグメント間のルーティングを行いながら、高度なアクセス制御、NAT、およびVPN終端を提供するために利用されます。
*   **展開場面**: インターネット境界、セグメント間ゲートウェイ、サイト間およびリモートアクセスVPNのヘッドエンドとして展開されます。

---

# 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | デバイスがルータとして動作し、インターフェイスごとにIPアドレスが割り当てられる。 |
| **用途** | ネットワークエッジ、L3セグメント間防御、VPNハブ、マルチキャストルーティング。 |
| **メリット** | 全てのルーティングプロトコル、NAT、VPN機能 (S2S, RA, VTI) がフルサポートされる。 |
| **デメリット** | 既存ネットワークへの挿入時にIPアドレス設計の変更 (再アドレッシング) が必要になる場合がある。 |
| **対応機種** | 全てのCisco ASAおよびCisco FTDハードウェア/仮想プラットフォーム。 |
| **制限事項** | 同一L2セグメント内の通信を直接フィルタリングできない (ブリッジグループ構成が必要)。 |
| **設計上の注意点** | FTDでは、**モード変更 (Routed ↔ Transparent) を行うと、既存のインターフェイス設定が全て消去される**。 |

---

# 🏗 動作原理

ルーテッドモードのデバイスは、受信したパケットの宛先IPアドレスに基づき、内部のルーティングテーブルを参照して出力インターフェイスとネクストホップを決定します。

```text
[Inside Host] (10.1.1.2)
   ↓ (デフォルトゲートウェイ: 10.1.1.1)
Inside Interface (10.1.1.1 / Security 100)
   ↓
[ パケット処理エンジン ]
   1. Route Lookup (宛先への経路があるか？)
   2. ACL/Policy Check (許可されているか？)
   3. NAT (送信元/宛先変換が必要か？)
   4. Inspection (Snort / MPF による検査)
   ↓
Outside Interface (203.0.113.1 / Security 0)
   ↓
[Internet / Next Hop Router]
```

---

# ⚙ 動作シーケンス

ASAおよびFTD (LINAエンジン) におけるパケット処理の優先順位は、CCIEラボ試験のパケットフロー解析において極めて重要です。

1.  **物理インターフェイス着信**: パケットが物理層で受信され、VPNトラフィックの場合はここで復号されます。
2.  **既存接続テーブル (Conn Table) 参照**: 既存のステートフルセッションがあるか確認します。一致すれば多くのセキュリティチェックをバイパスする「Fast Path」で処理されます。
3.  **NAT（Untranslate）**: 既存接続がない新規パケットに対し、宛先IPが変換されているかをチェックし、必要に応じて戻します。
4.  **ACLチェック**: 入力インターフェイスに適用されたAccess Control Listに基づき、許可または拒否を判断します。
5.  **インスペクション**: FTDのSnortエンジンやASAのMPFに転送され、詳細なプロトコル検査やIPS、AVC（Application Visibility and Control）が行われます。
6.  **Egress決定/ルーティング**: 宛先IPに基づきルーティングテーブルをルックアップし、NAT変換（Translate）を行います。
7.  **L2ルックアップ**: 出力先MACアドレスをARPテーブルから解決し、パケットを送出します。

---

# 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **動的ルーティングとの統合**: ルーテッドモードはOSPF (v2/v3), BGP, RIP, EIGRP (FTDはFlexConfig経由) をサポートします。ラボでは、ASA/FTDをエリアのABRやASBRとして動作させ、再配布 (Redistribution) を設定する構成が頻出します。
*   **VTI (Virtual Tunnel Interface)**: ルーテッドモード限定の機能として、IPsec VTIを用いたルートベースVPNの実装能力が問われます。

### ラボ試験で設定させられそうな内容
*   **FTD 管理 RIB の分離**: FTDでは、管理通信用 (FMC登録用など) のルーティングテーブルがデータトラフィック用とは独立して存在することを理解し、個別にルートを設定する必要があります。
*   **Integrated Routing and Bridging (IRB)**: ルーテッドモード内で特定インターフェイスをブリッジグループ (BVI) にまとめ、L2ブリッジングとL3転送を混在させる構成が狙われます。

### 試験で狙われやすい制限事項
*   **モード変更の破壊性**: ラボの途中でモード変更を指示された場合、**IPアドレスやZone割り当て、インターフェイスポリシーが全て消失する**ため、再設定の手順を事前計画する必要があります。
*   **Security Levelの挙動 (ASA)**: ASAではデフォルトで「高いレベルから低いレベル」の通信は許可されますが、逆はACLによる明示的な許可が必須です。

### showコマンド/debugログの読み取り
*   **`packet-tracer`**: パケットがどのフェーズ (ACL, Route, NAT) でドロップしているかを一撃で特定できる、ラボにおける**最重要検証ツール**です。
*   **`debug icmp trace`**: ICMPパケットの許可・拒否状態をリアルタイムで追跡するのに有効です。

---

# 🛠 設定方法

### ASA (CLI) - 基本設定
ASAでは、`nameif`、`security-level`、および `ip address` を設定するのが基本です。
```bash
# モードの確認（デフォルトはルーテッド）
show firewall

# インターフェイス設定例
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 203.0.113.1 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 192.168.1.1 255.255.255.0
 no shutdown

# デフォルトルートの設定
route outside 0.0.0.0 0.0.0.0 203.0.113.254 1
```

### FTD (FMC管理) - 基本設定
1.  **Devices > Device Management**: 対象のFTDデバイスを編集。
2.  **Interfacesタブ**: 物理ポートを「Routed」タイプに設定し、有効化。
3.  **IPv4タブ**: 静的IPアドレスまたはDHCPを設定。
4.  **Routingタブ**: Static Route、OSPF、BGPなどを構成。
5.  **Security Zone**: インターフェイスを「Routed」タイプのZoneに割り当て。

---

# 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **動作モードの確認** | <code>show firewall</code> |
| **IPアドレスと状態の確認** | <code>show interface ip brief</code> |
| **ルーティングテーブル表示** | <code>show route</code> |
| **NATテーブルの確認** | <code>show xlate</code> / <code>show nat</code> |
| **パケットフローのシミュレーション** | <code>packet-tracer input inside tcp 192.168.1.10 1024 8.8.8.8 443</code> |
| **ICMP処理の追跡デバッグ** | <code>debug icmp trace</code> |
| **OSPFネイバー状態の確認** | <code>show ospf neighbor</code> |

---

# 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 直接接続先へ疎通できない | インターフェイスのDown、またはVLAN不整合 | <code>show interface</code>で物理/論理状態を確認。対向SwitchのVLAN設定を確認。 |
| 通信が一方通行になる | 戻りのルートが対向ルータに存在しない | 対向デバイスで<code>show route</code>を実行し、ASA/FTDへの経路を追加する。 |
| 動的ルーティングが確立しない | 認証設定ミス、またはネットワーク不一致 | <code>debug ip ospf events</code>を実行し、ハローパケットの不一致理由を確認。 |
| FMCからのポリシー展開が失敗する | 管理通信用ゲートウェイの不足 | FTD CLIで<code>show network</code>を確認。管理用サブネットのルートを修正。 |
| モード変更後に設定が消えた | 仕様による初期化 | バックアップから再適用するか、FMCから再デプロイを行う。 |

---

# ⚠ 制限事項

*   **L2透過性の欠如**: 同一のL2ネットワークセグメント内に、ホストのIP変更なしでデバイスを「見えない」状態で挿入することはできません (トランスペアレントモードが必要です)。
*   **モード変更の破壊性**: モード変更コマンドは**デバイスを工場出荷状態にリセット**するため、リモート操作時には特に注意が必要です。
*   **IPv4 31-bit マスク**: ポイントツーポイント用に使用可能ですが、BVI (IRB) インターフェイスではサポートされません。
*   **VTI VPNの制約**: VTI (Virtual Tunnel Interface) はルーテッドモードでのみ構成可能であり、透過モードでは利用できません。

---

# 🔄 他技術との関連

*   **NAT**: ルーテッドモードにおいて、内部のプライベートIPを隠蔽するためのPATや、外部公開用のスタティックNATと密接に組み合わされます。
*   **High Availability (Failover)**: 2台のデバイス間でルーテッドインターフェイスのIPを同期し、冗長なL3ゲートウェイを提供します。
*   **Modular Policy Framework (MPF)**: ASAにおいて、ルーティングされる特定のトラフィックフローに対して高度なインスペクションを適用します。
*   **Security Intelligence**: ルーテッドパスを通過するトラフィックに対し、IP/URLベースで初期段階のドロップを行います。

---

# 🧩 比較表

### Routed vs Transparent (ASA/FTD)

| 機能 | Routed Mode (L3) | Transparent Mode (L2) |
| :--- | :--- | :--- |
| **OSIレイヤ** | レイヤ3 (Router Hop) | レイヤ2 (Bridge) |
| **IPアドレス配置** | インターフェイスごとに必須 | 管理用BVIに1つのみ必要 |
| **動的ルーティング** | ネイバーとして参加可能 | 通過を許可するのみ (基本不参加) |
| **VPN終端** | RA, S2S, VTI をフルサポート | 限定的 (管理用のみなど) |
| **ネットワーク挿入** | 再アドレッシングが必要 | 既存トポロジを変更せず挿入可能 |

---

# 💡 ベストプラクティス

*   **インターフェイス名の標準化**: `inside`, `outside`, `dmz` などの予約名を一貫して使用し、混乱を避けます。
*   **Default Routeの早期定義**: 特にFTDでは、FMCとの接続を維持するために、まず管理プレーンのデフォルトルートを確実に構成します。
*   **Packet Tracerの常用**: 本番パケットを流す前に、常に `packet-tracer` コマンドでポリシーとルーティングの妥当性を論理的に確認します。
*   **管理通信の分離**: データトラフィック用のデフォルトルートとは別に、管理インターフェイス専用のスタティックルートを構成し、管理プレーンの安定性を確保します。

---

# 📝 ラボ学習・設定サンプル例

### 1. 基本的なL3インターフェイスの構成 (ASA)
*   **課題**: Gig0/0をOutside (203.0.113.1/24)、Gig0/1をInside (192.168.1.1/24) として設定せよ。
*   **設定**:
```bash
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 203.0.113.1 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 192.168.1.1 255.255.255.0
 no shutdown
```

### 2. OSPF Area 0の設定 (FTD/FMC)
*   **課題**: FTDのInsideサブネット (192.168.1.0/24) をOSPFエリア0で広告せよ。
*   **手順**: FMCの **Devices > Routing > OSPF** でプロセスを有効化し、Networksタブで当該ネットワークを Area 0 に追加。

### 3. Identity NAT (NAT Exemption) の実装
*   **課題**: Inside (192.168.1.0/24) から DMZ (172.16.1.0/24) への通信は、NATをせずに透過させよ。
*   **設定 (ASA)**:
```bash
object network obj-inside
 subnet 192.168.1.0 255.255.255.0
object network obj-dmz
 subnet 172.16.1.0 255.255.255.0
nat (inside,dmz) source static obj-inside obj-inside destination static obj-dmz obj-dmz
```

### 4. IP SLAを使用したデフォルトルートのトラッキング
*   **課題**: 8.8.8.8へのICMPが失敗した場合、デフォルトルートをバックアップ (203.0.113.2) に切り替えよ。
*   **設定 (ASA)**:
```bash
sla monitor 1
 type echo protocol icmpService 8.8.8.8 interface outside
sla monitor schedule 1 life forever start-time now
track 1 rtr 1 reachability
route outside 0.0.0.0 0.0.0.0 203.0.113.254 1 track 1
route outside 0.0.0.0 0.0.0.0 203.0.113.2 10
```

### 5. FTD 管理インターフェイスのスタティックルート (CLI)
*   **課題**: FTDの管理通信のみをゲートウェイ 192.168.100.1 経由にせよ。
*   **設定**:
```bash
> configure network static-routes add 192.168.200.0 255.255.255.0 192.168.100.1 management
```

### 6. VTIを使用したSite-to-Site VPN (ASA)
*   **課題**: 対向ルータ 10.1.12.1 との間に仮想トンネルインターフェイス Tunnel1 を作成せよ。
*   **設定**:
```bash
interface Tunnel1
 nameif vti-vpn
 ip address 169.254.1.1 255.255.255.252
 tunnel source outside
 tunnel destination 10.1.12.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-PROF
```

### 7. OSPFとEIGRP間の再配布 (ASA)
*   **課題**: OSPF 1で学習した経路をEIGRP 100へ再配布せよ。
*   **設定**:
```bash
router eigrp 100
 redistribute ospf 1 metric 10000 10 255 1 1500 subnets
```

### 8. インターフェイス・パケットキャプチャの実行
*   **課題**: Insideインターフェイスに着信したパケットをリアルタイムで追跡せよ。
*   **実行**:
```bash
capture TEST trace interface inside
# パケット発生後
show capture TEST
```

### 9. Management RIB の確認 (FTD CLI)
*   **課題**: 管理用ルーティングテーブルのみを表示せよ。
*   **実行**:
```bash
> show route management
```

### 10. Packet Tracer によるポリシー検証
*   **課題**: Inside(192.168.1.10)からOutside(8.8.8.8)へのHTTPSが許可されているか検証せよ。
*   **実行**:
```bash
packet-tracer input inside tcp 192.168.1.10 1234 8.8.8.8 443
```

---

# ❓ 想定試験問題

1.  **コンフィグ読解**: `firewall transparent` が設定されているデバイスで、`interface Tunnel1` (VTI) を設定しようとしたがエラーが出た。原因を答えよ。
    *   **解答**: VTI (Virtual Tunnel Interface) はルーテッドモードのみのサポートであり、透過モードでは構成できないため。
2.  **トラブルシュート**: FTDをRoutedからTransparentに変更したところ、SSHによる管理アクセスができなくなった。原因は何か？
    *   **解答**: モード変更は破壊的操作であり、IPアドレスを含む全てのインターフェイス設定が消去されたため。
3.  **Design**: 既存のIPアドレス設計を変更せず、かつOSPFネイバーとしてルーティングに参加可能なファイアウォールを導入したい。どのモードを選択すべきか？
    *   **解答**: ルーテッドモードの IRB (Bridge Group) 構成。透過モードはルーティングプロトコルに参加 (ネイバー確立) できないため。
4.  **実装**: ASAのInside(S:100)からOutside(S:0)へパケットを送ったが、ACLなしでドロップされた。何を確認すべきか？
    *   **解答**: `packet-tracer` を使用し、NATルールが不足していないか、またはMPFのインスペクションで拒否されていないかを確認する。
5.  **動作シーケンス**: ASAにおいて、既存の接続 (Connエントリ) に一致するパケットが到着した場合、ACLのチェックは行われるか？
    *   **解答**: 行われない。既存接続に一致するパケットはFast Pathで処理され、ACLチェックをバイパスする。

---

# 🔗 参考リソース

*   **Cisco Live 動画/スライド**:
    *   [BRKSEC-3020: Troubleshooting Firewall Threat Defense (FTD)](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-3020)
    *   [BRKSEC-2021: Firepower Threat Defense - Packet Flow and Troubleshooting](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-2021)
*   **Configuration Guides**:
    *   [Cisco ASA Series General Operations CLI Configuration Guide, 9.4 - Firewall Mode Overview](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/config/general/asa-94-general-config/intro-fw.html)
    *   [Cisco Firepower Threat Defense Configuration Guide for FMC, 7.0 - Routed and Transparent Mode Interfaces](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/interfaces_firewall_mode.html)
*   **Command Reference**:
    *   [Cisco ASA Series Command Reference - firewall routed](https://www.cisco.com/c/en/us/td/docs/security/asa/command-reference/f-l/cmdref1/f1.html#pgfId-1520630)
    *   [Cisco ASA Series Command Reference - packet-tracer](https://www.cisco.com/c/en/us/td/docs/security/asa/command-reference/m-p/cmdref2/p1.html#pgfId-1823932)
*   **Technical Notes**:
    *   [Configuring Firepower Threat Defense Interfaces in Routed Mode](https://www.cisco.com/c/en/us/support/docs/security/firepower-ngfw/200908-configuring-firepower-threat-defense-int.html)
    *   [ASA/PIX: Configure and Troubleshoot the Reverse Route Injection (RRI)](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/107596-asa-reverseroute.html)
*   **White Paper / Design Guide**:
    *   [Cisco SAFE Design Guide - Firewall Deployment Modes](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Aug2014/CVD-FirewallDeploymentGuide-AUG14.html)
*   **CVD (Cisco Validated Design)**:
    *   [Secure Campus Design Guide - Routed Firewall Implementation](https://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/Campus/campover.html)
---

📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

---
