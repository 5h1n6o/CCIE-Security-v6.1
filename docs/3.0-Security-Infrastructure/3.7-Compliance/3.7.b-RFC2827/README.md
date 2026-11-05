---
layout: default
title: 3.7.b-RFC2827
nav_order: 2
parent: 3.7-Compliance
grand_parent: 3.0-Security-Infrastructure
---

# 3.7.b RFC 2827

## 📘 概要

RFC 2827（BCP 38と同義的標準規格）は、ネットワーク境界（Ingress/Egress Edge）における「送信元IPアドレス偽装（IP Source Address Spoofing）」を排除することを目的としたインターネット標準技術仕様（Network Ingress Filtering: Defeating Denial of Service Attacks which employ IP Source Address Spoofing）です。

現代のサイバー脅威において、DDoS（Distributed Denial of Service）攻撃、特にUDP/ICMPを利用したDNS/NTP/SNMPリフレクション（反射・増幅）攻撃は、攻撃者が送信元IPアドレスを被害者のIPアドレスに偽装（Spoof）してリクエストを送ることで成り立っています。RFC 2827は、ISP（インターネットサービスプロバイダ）やエンタープライズのネットワークエッジにおいて、該当するサブネットやインターフェイスから送信されるパケットの送信元IPアドレスが「正規に割り当てられたIPプレフィックス範囲内であるか」を検証し、範囲外の不正なパケットを即座に破棄（Ingress Filtering）する具体的な指針および技術的要件を定めています。

CCIE Security v6.1 試験において、RFC 2827は単なる概念的RFC理解にとどまらず、Cisco IOS-XE、Cisco ASA、およびCisco FTD (Firepower Threat Defense) における厳格なルータ・ファイアウォール実装技術（Strict/Loose uRPF、Ingress ACL、IP Source Guard等）として出題されます。

---

## 🔑 要点

RFC 2827（Network Ingress Filtering）の主要な技術的特徴、適用領域、メリット・デメリット、および設計上の考慮事項を下表にまとめます。

| 項目 | 内容 |
| --- | --- |
| **特徴** | インターフェイスに入力されるパケットの「送信元IPアドレス」が、そのインターフェイスまたは配下サブネットにアサインされた正規のIPプレフィックスに合致するかを検証し、偽装パケットをドロップするメカニズム |
| **用途** | DDoS反射増幅攻撃の防止、内部ネットワークからのIPスプーフィング攻撃の発信防止（Egress/Ingress Filtering）、CDR/SOCにおけるログ改ざん防止 |
| **メリット** | 自組織がDDoS攻撃の加害者（Reflector/Botnetの踏み台）となるリスクを根絶。ボットネット等の攻撃トラフィックを発生源最寄りエッジで防御可能 |
| **デメリット** | 非対称ルーティング（Asymmetric Routing）やマルチホーム環境での設計・設定を誤ると、正常な通信（マルチパスルーティング）が誤ってドロップされるリスクがある |
| **対応機種** | Cisco Catalyst 9000シリーズ（IOS-XE）、ISR 4000/8000、ASR 1000、Cisco ASA 5500-X、Cisco Secure Firewall (FTD) |
| **制限事項** | トンネルインターフェイス（GRE, IPsec VTI, VXLAN等）やCEF（Cisco Express Forwarding）のハードウェアスイッチング制限を受ける場合がある |
| **設計上の注意点** | 単一回線（Single-homed）環境ではStrict uRPFが最適だが、マルチホーム（Multi-homed）環境ではFeasible Path uRPFまたはLoose uRPF / Ingress ACLの採用が必須 |

---

## 🏗 動作原理

RFC 2827の基本概念は、「パケットの送信元IPアドレス（Source IP Address）は、パケットが到達した物理/論理インターフェイスのルーティングテーブルまたは認可済みプレフィックスリストと整合していなければならない」という原則に基づきます。

従来のルータは、宛先IPアドレス（Destination IP Address）のみを参照してルーティング決定（Forwarding Decision）を行いますが、RFC 2827は送信元IPアドレスに基づく検証（Verification）プロセスを処理パイプラインに割り込ませます。

```
[ エッジデバイス / クライアント ]
          │ (送信元IP: 192.0.2.5 [偽装], 宛先IP: 8.8.8.8)
          ▼
[ Cisco Edge Router / Switch (Ingress Interface) ]
          │
          ├─► 1. 送信元 IP アドレス抽出 (Source IP: 192.0.2.5)
          │
          ├─► 2. FIB (CEF) / ACL データベースの検索
          │      ├─ インターフェイス認可プレフィックス: 10.1.1.0/24 のみ許容
          │      └─ 判定: 192.0.2.5 は 10.1.1.0/24 に含まれない！
          │
          ├─► [ 違反時 (Violation) ] ──► 即座にパケット破棄 (Drop) & カウンタ加算 / Logging
          │
          └─► [ 正常時 (Permitted) ] ──► 宛先 IP (8.8.8.8) に基づく通常の CEF 転送
```

RFC 2827を具現化する代表的な実装技術は以下の4つです。

1. **Ingress Access Control Lists (iACLs) / Static Ingress Filtering**:
   * インターフェイスごとに、送信を許可する送信元IPプレフィックスを明示的に定義したACLを適用。
   * 小型拠点や単一サブネットアクセスポートで確実な制御を提供するが、プレフィックス変更時の運用コストが高い。
2. **Unicast Reverse Path Forwarding (uRPF - Strict Mode)**:
   * 入力パケットの送信元IPアドレスをCEF（FIB）テーブルで検索。
   * 「送信元IPアドレスへの最善経路（Best Path）が存在し、かつパケットを受信したまさにそのインターフェイスを経由しているか」を検証。
3. **Unicast Reverse Path Forwarding (uRPF - Loose Mode)**:
   * 入力パケットの送信元IPアドレスがCEFテーブル（FIB）内に存在するか（到達可能か）のみを検証。どのインターフェイスから入ってきたかは問わない。
4. **IP Source Guard (IPSG)**:
   * Layer 2 スイッチポートにおいて、DHCP Snoopingバインディングデータベース（または静的バインディング）を参照し、ポート・MAC・IPアドレスの不一致パケットをハードウェア（ASIC）レベルでドロップ。

---

## ⚙ 動作シーケンス

パケットがRFC 2827（uRPF / Ingress Filtering）を有効化したルータまたはファイアウォールに入力された際の詳細な処理シーケンスは以下の通りです。

```
1. パケット受領 (Ingress Frame Arrival)
   └─ 物理/論理インターフェイス（例: GigabitEthernet0/1）でパケットを受信。

2. レイヤ2/レイヤ3 ヘッダーデコード
   └─ イーサネットフレームからIPヘッダーを解析し、送信元IP (Src IP) および宛先IP (Dst IP) を抽出。

3. RFC 2827 / uRPF 判定エンジンの起動 (CEF Check)
   │
   ├─► [Strict Mode uRPF が有効な場合]
   │    ├─ CEF (FIB) テーブルで Src IP を検索。
   │    ├─ 最上位ロンゲストマッチ（Longest Prefix Match）経路の出口インターフェイスを取得。
   │    └─ 比較: 「FIBにおける出口インターフェイス」 == 「実際のパケット入力インターフェイス」？
   │         ├─ YES: 検証成功 (Pass) ──► Step 4 へ移動。
   │         └─ NO : 検証失敗 (Drop) ──► Drops カウンタをインクリメントし、パケットを廃棄。
   │
   ├─► [Loose Mode uRPF が有効な場合]
   │    ├─ CEF (FIB) テーブルで Src IP を検索。
   │    └─ 比較: 「FIBに有効なルートが存在するか（Null0ルート除く）？」
   │         ├─ YES: 検証成功 (Pass) ──► Step 4 へ移動。
   │         └─ NO : 検証失敗 (Drop) ──► パケットを廃棄。
   │
   └─► [Ingress ACL (iACL) が適用されている場合]
        ├─ ACL エントリを上から順に評価。
        ├─ 送信元 IP プレフィックスが permit にマッチ ──► Step 4 へ移動。
        └─ deny または 暗黙の deny にマッチ ──► パケットを廃棄 (ACL Drop)。

4. ポリシー検査 & 通常ルーティング転送
   ├─ Access Control Policy / NAT / QoS / Policy-Based Routing (PBR) 等の追加ポリシー処理。
   └─ 宛先 IP (Dst IP) に基づき CEF ルックアップを実行し、適切な Egress インターフェイスへ転送。
```


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、RFC 2827（Ingress Filtering / uRPF）は「単に `ip verify unicast source reachable-via rx` を設定する」といった単純な問題としては出題されません。複雑なネットワークトポロジーや制約条件の中で、適切なモードおよびアタッチポイントを選択・実装する能力が試されます。

### Blueprintで重要なポイント
* **マルチホーム環境・非対称ルーティング対策**: 2つのISP回線や冗長パスが存在する場合、Strict Mode uRPFを適用すると非対称トラフィックが不当にドロップされます。ラボ試験では「特定のトラフィックドロップを回避しつつ、送信元アドレス検証を有効化せよ」といった条件で `feasible-path` オプションや `Loose Mode` の適用が問われます。
* **デフォルトルート（Default Route）の扱い**: `ip verify unicast source reachable-via rx` のみでは、FIB上のデフォルトルート（`0.0.0.0/0`）は無視され、明示的なルートが存在しないアドレスからのパケットはドロップされます。デフォルトルートを検証対象に含める要件がある場合は `allow-default` オプションの追加が必須です。
* **ACLによる例外処理（Access-List Integration）**: 特定の送信元IP（例: 管理用IP、テスト用スプーフィングパケット、特定のBGPピア）をuRPF評価からバイパス（permit）させる要件に対し、`ip verify unicast source reachable-via rx <ACL_NAME>` の構文が要求されます。
* **Cisco ASA / FTD における uRPF 実装**: ASA/FTDではインターフェイスごとに `ip verify reverse-path interface <interface_name>` を有効化します。ASA/FTDはステートフルファイアウォールであるため、uRPFの挙動と接続テーブル（Conn Table）/ 接続ステートチェックとの相違点を理解しておく必要があります。

### よくある設定ミス・制限事項
1. **CEF（Cisco Express Forwarding）が無効**: uRPFはCEFデータベース（FIB）に完全に依存します。グローバルで `ip cef` (IPv4) または `ipv6 unicast-routing` / `ipv6 cef` (IPv6) が有効化されていないと、uRPFは動作しないか全パケットをドロップします。
2. **非対称ルートにおける Strict Mode の誤用**: 送信パス（LISP, BGP Multi-homing, OSPF Multi-area）と受信パスが異なるインターフェイスで Strict Mode を有効にすると、正常なサービス通信が大規模に障害化します。
3. **デフォルトルートを期待した `allow-default` の欠落**: インターフェイスに `ip verify unicast source reachable-via rx` を設定した際、ルーティングテーブルにデフォルトルートしか存在しない宛先（インターネット経由のトラフィック）からの入力がすべてドロップされるケース。

---

## 🛠 設定方法

Cisco IOS-XE、Cisco ASA、および Cisco FTD (FMC Managed) における RFC 2827 (uRPF / Ingress Filtering) の具体的な設定例を掲載します。

### 1. Cisco IOS-XE (Router / Catalyst Switch)

#### A. Strict Mode uRPF (単一回線・完全保護環境)
```bash
! CEF の有効化 (必須前提条件)
ip cef
ipv6 unicast-routing

! インターフェイスレベルでの Strict Mode 設定
interface GigabitEthernet0/1
 description Edge_Ingress_Interface
 ip address 192.0.2.1 255.255.255.0
 ip verify unicast source reachable-via rx
 ipv6 verify unicast source reachable-via rx
```

#### B. Feasible Path Mode uRPF + デフォルトルート許容 + ACL例外処理
```bash
! 例外制御用 ACL (ACLで permit されたトラフィックは uRPF 検証をパス)
ip access-list extended URPF_EXCEPTIONS
 permit ip 10.255.0.0 0.0.255.255 any
 deny   ip any any

! インターフェイスでの Feasible Path + allow-default + ACL 設定
interface GigabitEthernet0/2
 description MultiHomed_Edge_Interface
 ip address 203.0.113.1 255.255.255.0
 ip verify unicast source reachable-via rx allow-default URPF_EXCEPTIONS
```

#### C. Loose Mode uRPF (バックボーン・ISPコア環境)
```bash
interface TenGigabitEthernet1/0/1
 description Core_Transit_Interface
 ip address 198.51.100.1 255.255.255.252
 ip verify unicast source reachable-via any
```

### 2. Cisco ASA (CLI)

Cisco ASA において uRPF はインターフェイス単位で設定し、デフォルトで Strict モード相当の動作を行います。

```bash
! ASA インターフェイスでの uRPF 有効化
ciscoasa(config)# ip verify reverse-path interface outside
ciscoasa(config)# ip verify reverse-path interface DMZ
```

### 3. Cisco FTD / FMC (Firepower Management Center)

FMC で FTD の uRPF を有効化する設定手順:
1. FMC GUI ログイン ──► **Devices** ──► **Device Management** に移動。
2. 該当する FTD デバイスの編集（鉛筆アイコン）を選択。
3. **Interfaces** タブを選択し、uRPFを有効化したいインターフェイス（例: `outside`）の編集をクリック。
4. **Advanced** タブをクリック。
5. **Enable Anti-Spoofing (uRPF)** チェックボックスをオン（Check）に設定。
6. **Save** をクリックし、デバイスに **Deploy** を実行。

---

## 🔍 検証コマンド

RFC 2827 / uRPF の動作確認およびパケットドロップの統計情報を確認するための検証コマンド一覧です。

| 目的 | コマンド |
| --- | --- |
| インターフェイスの uRPF 設定状態確認 | `show ip interface GigabitEthernet0/1 \| include drop\|unicast` |
| 全インターフェイスの uRPF 設定一括確認 | `show ip interface \| include verify` |
| CEF（FIB）テーブルの送信元ルックアップ確認 | `show ip cef 192.0.2.100` |
| uRPF ドロップ統計カウンタの確認 | `show cef interface GigabitEthernet0/1` |
| 非推奨/ドロップパケットのグローバル統計確認 | `show ip traffic \| include Drop` |
| ASA での uRPF 統計確認 | `show ip verify drop` |
| uRPF ドロップパケットのリアルタイムデバッグ | `debug ip verify` |
| CEF パケットドロップ処理のデバッグ | `debug ip cef drops` |

---

## 🚨 トラブルシュート

RFC 2827 / uRPF 実装時に遭遇する代表的な障害症状、根本原因、確認コマンド、および対処方法を下表に整理します。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| 特定のサブネットからの通信が一切疎通不能 | 非対称ルーティングにより、パケット入力インターフェイスとFIBの出向インターフェイスが不一致（Strict Mode違反） | `show ip cef <Src_IP>`, `show cef interface <Int>` | 非対称パスが存在する場合は `reachable-via any` (Loose Mode) または `allow-default` / `feasible-path` に変更 |
| インターネット宛ての通信のレスポンスが不当にドロップされる | FIB上にデフォルトルート（0.0.0.0/0）しか存在しない宛先に対して `allow-default` オプションが設定されていない | `show ip route <Src_IP>`, `show ip interface <Int>` | `ip verify unicast source reachable-via rx allow-default` を設定に追加 |
| uRPF を設定したにもかかわらず spoofed パケットが転送される | グローバルで CEF が無効化されている、または対象トラフィックが PBR で迂回されている | `show ip cef`, `show ip interface <Int>` | グローバルで `ip cef` を有効化。PBRとの競合関係を再設計 |
| ASA/FTD で特定の合法トラフィックが Reverse-Path Drop される | アシンメトリックなマルチホーム構成で、リターンパケットが別のインターフェイスに入力されている | ASA: `show ip verify drop` | ASA/FTD で `ip verify reverse-path` を該当インターフェイスから解除するか、Equal-Cost Multipath (ECMP) ルーティングを適切に設定 |


## ⚠ 制限事項

RFC 2827 (uRPF / Ingress Filtering) の運用における技術的制限事項およびハードウェア/ソフトウェア依存性は以下の通りです。

* **CEF（Cisco Express Forwarding）依存**:
  * uRPFは高速なFIBルックアップを前提としています。プロセススイッチング（Process Switching）やFast Switching環境では機能しないか、著しいパフォーマンス低下を引き起こします。
* **Multicast トラフィックの非サポート**:
  * uRPFは Unicast IP パケット専用のフィルタリングメカニズムです。マルチキャストパケット（DIP: 224.0.0.0/4）の送信元検証には、PIM (Protocol Independent Multicast) の RPF (Reverse Path Forwarding) チェックが独立して動作します。
* **トンネル・カプセル化パケットでの制約**:
  * GRE, IPsec VTI, VXLAN 等のカプセル化パケットに対しては、外側（Outer）IPヘッダーに対してuRPFが評価されます。内側（Inner）IPヘッダーに対してuRPFを適用するには、Decapsulation後の論理/VTIインターフェイス上で個別設定が必要です。
* **BGP BFD / OSPF Hello 等の制御プレーン通信への影響**:
  * ピアリングアドレスがルーティングテーブル上で特定のインターフェイスに紐づいていない場合（例: Loopback経由のBGP）、uRPF設定によって制御プレーンパケットがドロップされないよう、ACL例外（`ip verify unicast source ... <ACL>`）の設計に配慮が必要です。

---

## 🔄 他技術との関連

RFC 2827 は、Cisco インフラストラクチャにおける他のセキュリティ機能およびルーティング技術と密接に相互作用します。

* **Access Control Lists (iACLs / VACLs)**:
  * Static Ingress ACL は RFC 2827 の最も基本的な実現形態です。uRPFとiACLを併用することで、uRPFによる動的FIB検証に加え、予約済みアドレス（RFC 1918, RFC 5735, Loopback 127.0.0.0/8等）の固定ドロップを二重で強化できます。
* **IP Source Guard (IPSG)**:
  * 物理アクセスポート（Layer 2 Edge）における RFC 2827 の拡張概念です。DHCP Snooping テーブルを参照し、端末がアサインされたIP/MACアドレス以外の偽装パケットを送出するのをL2 ASICレベルで阻止します。
* **BGP Remotely Triggered Black Hole (RTBH)**:
  * Loose Mode uRPF と Destination-based / Source-based RTBH を組み合わせることで、DDoS攻撃元からのトラフィックを Null0 ループバックに送り込んでエッジで高速廃棄するアーキテクチャ（BCP 38 + BCP 84）を構築できます。
* **Cisco TrustSec (SGT / SXP)**:
  * SGT（Security Group Tag）インラインタグ付け環境において、IP Spoofing を防止しておくことで、IP-to-SGT バインディングの偽装やアイデンティティ詐称（Identity Spoofing）を未然に防ぐ基礎インフラとなります。

---

## 🧩 比較表

RFC 2827 に関連する技術仕様およびモードの比較表です。

### 1. RFC 2827 (BCP 38) vs RFC 3704 (BCP 84)

| 比較項目 | RFC 2827 (BCP 38) | RFC 3704 (BCP 84) |
| --- | --- | --- |
| **主対象** | 単一マルチホスト・シングルホームアクセスネットワークにおける Ingress Filtering | マルチホーム（Multi-homed）AS/ISP ネットワークにおける Ingress Filtering |
| **定義手法** | Static ACL, Strict uRPF | Strict, Feasible Path, Loose uRPF, BGP AS-Path/Prefix-list Filtering |
| **非対称経路対応** | 基本的に非対応（単一回線を前提） | Feasible Path (RPFテーブル参照) や Loose モードにより非対称ルートに対応 |

### 2. uRPF 動作モード比較 (Strict vs Feasible Path vs Loose)

| 項目 | Strict Mode (`rx`) | Feasible Path Mode (`rx`) | Loose Mode (`any`) |
| --- | --- | --- | --- |
| **検証条件** | 送信元IPへの「FIB最善経路（Best Path）」の出口ポートと「入力ポート」が**完全一致** | 送信元IPへの「FIB代替経路（Feasible Path）」の出口ポート群に「入力ポート」が**含まれる** | 送信元IPがFIB内に**存在**する（出力ポート問わない） |
| **デフォルトルート** | デフォルト無視 (`allow-default` で許可可) | デフォルト無視 (`allow-default` で許可可) | デフォルト無視 (`allow-default` で許可可) |
| **非対称経路** | 正常パケットをドロップ（動作不可） | 同一AS/IGP内の別パスであれば許可可能 | 完全許可（非対称経路に対応） |
| **セキュリティ強度** | **最高**（完全なIPスプーフィング防止） | **高**（マルチホームでの安全なスプーフィング防止） | **中**（未割当/Bogonアドレスのドロップのみ） |
| **推奨適用場所** | アクセスポート、エンタープライズSingle-homed Edge | マルチホーム Enterprise Edge, ISP Peering | ISP Core, Transit Backbone |

### 3. IP Spoofing 防御技術比較 (iACL vs uRPF vs IPSG)

| 比較項目 | Ingress ACL (iACL) | Unicast RPF (uRPF) | IP Source Guard (IPSG) |
| --- | --- | --- | --- |
| **動作レイヤ** | Layer 3 / Layer 4 | Layer 3 (CEF / FIB) | Layer 2 (Switch Access Port) |
| **テーブル参照** | 静的 ACL エントリ | CEF (FIB) ルーティングテーブル | DHCP Snooping Binding Table |
| **運用負荷** | 高（IP変更時に手動更新） | **低**（ルーティングと自動連動） | **低**（DHCPと動的連動） |
| **適用場所** | ネットワーク境界（Edge/DMZ） | ネットワーク境界（Edge/Core） | クライアント収容L2アクセススイッチ |

---

## 💡 ベストプラクティス

1. **アクセス層における原則**:
   * クライアント収容ポート（Access Port）には **IP Source Guard (IPSG)** を適用し、L2レベルで RFC 2827 を達成する。
2. **シングルホーム・エンタープライズエッジ**:
   * ISP接続インターフェイス（Single-homed）には **Strict Mode uRPF (`reachable-via rx allow-default`)** を適用する。
3. **マルチホーム・エンタープライズエッジ**:
   * 複数のISPや拠点を収容するエッジには **Feasible Path uRPF** または **Loose Mode uRPF (`reachable-via any`)** を適用し、非対称ルーティングによる誤ドロップを完全に防止する。
4. **Bogon / RFC 1918 パケットのエッジでの遮断**:
   * uRPF と併用して、WAN エッジの Ingress ACL で予約済みプライベートIP（10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16）や Loopback（127.0.0.0/8）、APNIC/ARIN未割り当て（Bogon）アドレスを静的に deny する。


## 📝 ラボ学習・設定サンプル例

CCIE Security v6.1 ラボ試験レベルの実践的な設定サンプル例を10個作成しました。

---

### サンプル例 1: Strict Mode uRPF 基本設定 (Single-homed Edge)

#### 問題
R1は単一のISP回線（GigabitEthernet0/1）に接続されたシングルホーム・エッジルータです。外部からのIPスプーフィングパケットを遮断するため、RFC 2827に準拠したStrict Mode uRPFを構成してください。

#### 要件
1. GigabitEthernet0/1 において、CEF FIBの最善経路と入力ポートが一致しないパケットをドロップすること。
2. デフォルトルート経由の通信も正常に通過できるよう `allow-default` オプションを付与すること。
3. ドロップされたパケット数を `show cef interface` で確認できるようにすること。

#### 設定例
```bash
! CEFの有効化
ip cef

! インターフェイス構成
interface GigabitEthernet0/1
 description WAN_ISP_Primary
 ip address 203.0.113.2 255.255.255.252
 ip verify unicast source reachable-via rx allow-default
```

#### 検証方法
```bash
! uRPFの設定状態確認
R1# show ip interface GigabitEthernet0/1 | include IP verify
  IP verify source security group inactive
  IP verify unicast reverse-path verification is enabled

! CEF インターフェイス統計で Drop カウンタを確認
R1# show cef interface GigabitEthernet0/1 | include drop
  145 unicast RPF drops
```

---

### サンプル例 2: Feasible Path uRPF + ACL例外制御

#### 問題
R2はマルチホーム構成のエッジルータです。内部管理サブネット `10.250.0.0/16` からのトラフィックはテスト目的でスプーフィングパケットが含まれるため、uRPF検証から除外（バイパス）し、その他のトラフィックには Strict モード uRPF を適用してください。

#### 要件
1. 管理用サブネット `10.250.0.0/16` からの通信は uRPF チェックをスキップ（Permit）すること。
2. その他の通信には `reachable-via rx` を適用すること。
3. 名前の付き拡張ACL `ACL_URPF_BYPASS` を使用すること。

#### 設定例
```bash
! 例外用 ACL の作成
ip access-list extended ACL_URPF_BYPASS
 permit ip 10.250.0.0 0.0.255.255 any
 deny   ip any any

! インターフェイスへの適用
interface GigabitEthernet0/2
 description MultiHomed_Internal_Edge
 ip address 192.168.10.1 255.255.255.0
 ip verify unicast source reachable-via rx ACL_URPF_BYPASS
```

#### 検証方法
```bash
! ACL のマッチ確認
R2# show ip access-lists ACL_URPF_BYPASS
Extended IP access list ACL_URPF_BYPASS
    10 permit ip 10.250.0.0 0.0.255.255 any (42 matches)
    20 deny ip any any (1200 matches)
```

---

### サンプル例 3: Loose Mode uRPF (ISP Core / Transit)

#### 問題
R3はコアバックボーンルータです。非対称ルーティングが多発するコアネットワークにおいて、ルータに登録されていない未割当・Bogon IPからのスプーフィングパケットのみをドロップし、非対称パスの通信は正常に転送してください。

#### 要件
1. 送信元IPアドレスがFIB（ルーティングテーブル）内に存在するかのみを検証する（Loose Mode）。
2. インターフェイス `TenGigabitEthernet1/0/1` に適用すること。

#### 設定例
```bash
ip cef

interface TenGigabitEthernet1/0/1
 description Core_Backbone_Link
 ip address 198.51.100.5 255.255.255.252
 ip verify unicast source reachable-via any
```

#### 検証方法
```bash
R3# show ip interface TenGigabitEthernet1/0/1 | include reverse-path
  IP verify unicast reverse-path verification is enabled
```

---

### サンプル例 4: Cisco ASA での Anti-Spoofing (uRPF) 実装

#### 問題
Cisco ASA 5500-X において、DMZインターフェイスおよび Outside インターフェイスに入力されるIPスプーフィングパケットを防止するため、uRPF を有効化してください。

#### 要件
1. `outside` および `dmz` インターフェイスで Reverse-Path 機能を有効化すること。
2. ドロップされたパケットの統計ログを確認すること。

#### 設定例
```bash
ciscoasa(config)# ip verify reverse-path interface outside
ciscoasa(config)# ip verify reverse-path interface dmz
```

#### 検証方法
```bash
ciscoasa# show ip verify drop
Total rpf drops: 84
  outside: 84
  dmz: 0
```

---

### サンプル例 5: Cisco FTD (FlexConfig / FMC) での Anti-Spoofing

#### 問題
FMCによって管理されている FTD デバイスの `outside` インターフェイスにおいて Anti-Spoofing (uRPF) を有効化してください。

#### 要件
1. FMC デバイス設定画面から `outside` インターフェイスの Anti-Spoofing を有効にすること。
2. CLI から `show asp drop` コマンドで Reverse-Path ドロップを確認すること。

#### 設定手順 (FMC GUI & FTD CLI)
```text
[FMC GUI]
1. Devices -> Device Management -> Edit FTD Device
2. Interfaces -> Select "outside" Interface -> Edit
3. Advanced Tab -> Check [x] Enable Anti-Spoofing (uRPF)
4. Save & Deploy Policy
```

```bash
! FTD CLI での検証
ftd1# show asp drop
Frame drop code-point                                  Cases
Reverse-path verify failed (rpf-violation)                56
```

---

### サンプル例 6: Ingress ACL (iACL) による BCP 38 静的フィルタリング

#### 問題
エッジルータ R4 の WAN インターフェイス `GigabitEthernet0/0` において、RFC 1918 などの予約済みプライベートIPアドレスおよび自組織の割り当てプレフィックス外からのパケットを静的 Ingress ACL でドロップしてください。

#### 要件
1. 自組織割り当てプレフィックス `203.0.113.0/24` のみを送信元とするパケットを許可。
2. RFC 1918 (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) および Loopback (127.0.0.0/8) を明示的にドロップ。
3. ドロップされたパケットのログを出力すること。

#### 設定例
```bash
ip access-list extended iACL_BCP38_INGRESS
 deny   ip 10.0.0.0 0.255.255.255 any log-input
 deny   ip 172.16.0.0 0.15.255.255 any log-input
 deny   ip 192.168.0.0 0.0.255.255 any log-input
 deny   ip 127.0.0.0 0.255.255.255 any log-input
 permit ip 203.0.113.0 0.0.0.255 any
 deny   ip any any log-input

interface GigabitEthernet0/0
 description WAN_Ingress
 ip address 203.0.113.1 255.255.255.0
 ip access-group iACL_BCP38_INGRESS in
```

#### 検証方法
```bash
R4# show ip access-lists iACL_BCP38_INGRESS
Extended IP access list iACL_BCP38_INGRESS
    10 deny ip 10.0.0.0 0.255.255.255 any log-input (12 matches)
    20 deny ip 172.16.0.0 0.15.255.255 any log-input
    30 deny ip 192.168.0.0 0.0.255.255 any log-input
    40 deny ip 127.0.0.0 0.255.255.255 any log-input
    50 permit ip 203.0.113.0 0.0.0.255 any (840 matches)
    60 deny ip any any log-input
```

---

### サンプル例 7: IPv6 Strict Mode uRPF

#### 問題
IPv6 ネットワーク環境において、R5 の外部インターフェイス `GigabitEthernet0/1` で IPv6 送信元スプーフィングを防止するため Strict uRPF を構成してください。

#### 要件
1. IPv6 ルーティングおよび IPv6 CEF を有効化すること。
2. `GigabitEthernet0/1` に IPv6 Strict uRPF を適用すること。

#### 設定例
```bash
ipv6 unicast-routing
ipv6 cef

interface GigabitEthernet0/1
 description IPv6_WAN_Edge
 ipv6 address 2001:db8:1::2/64
 ipv6 verify unicast source reachable-via rx
```

#### 検証方法
```bash
R5# show ipv6 interface GigabitEthernet0/1 | include verify
  IPv6 verify unicast source reachable-via RX
```

---

### サンプル例 8: IP Source Guard (IPSG) による L2 アクセス層での BCP 38 実装

#### 問題
Catalyst 9300 スイッチ SW1 のアクセスポート `GigabitEthernet1/0/5` において、端末が不当なIPアドレスを偽装して通信するのを阻止するため IP Source Guard を有効化してください。

#### 要件
1. DHCP Snooping を VLAN 10 で有効化すること。
2. `GigabitEthernet1/0/5` ポートで IP および MAC アドレスのバインディング検証を有効化すること。

#### 設定例
```bash
! DHCP Snooping の有効化
ip dhcp snooping
ip dhcp snooping vlan 10

! インターフェイスでの IPSG 有効化
interface GigabitEthernet1/0/5
 switchport mode access
 switchport access vlan 10
 ip verify source tracking port-security
```

#### 検証方法
```bash
SW1# show ip verify source interface GigabitEthernet1/0/5
Interface  Filter-type  Filter-mode  IP-address       Mac-address        Vlan
---------  -----------  -----------  ---------------  -----------------  ----
Gi1/0/5    ip-mac       active       10.1.1.50        0011.2233.4455     10
```

---

### サンプル例 9: BGP RTBH (Remotely Triggered Black Hole) と Loose uRPF の連携

#### 問題
R6 において、DDoS攻撃の送信元IP（または攻撃対象IP）を Null0 にブラックホールルーティングし、Loose Mode uRPF により自動的にエッジでパケットを破棄する仕組みを構成してください。

#### 要件
1. ルーティングテーブル内に Null0 宛てのスタティックルートが存在する場合、Loose Mode uRPF によりその送信元からのパケットをドロップすること。
2. インターフェイス `GigabitEthernet0/0` に Loose uRPF を適用すること。

#### 設定例
```bash
! 攻撃元 IP アドレスを Null0 へルーティング
ip route 192.0.2.222 255.255.255.255 Null0

! インターフェイス設定
interface GigabitEthernet0/0
 description Edge_Peering
 ip address 203.0.113.10 255.255.255.0
 ip verify unicast source reachable-via any
```

#### 検証方法
```bash
! 192.0.2.222 からのパケットが入力された場合、FIBルックアップで Null0 にマッチし Loose uRPF で破棄される
R6# show ip cef 192.0.2.222
192.0.2.222/32
  attached to Null0
```

---

### サンプル例 10: 非対称ルーティング障害のトラブルシューティングと修正

#### 問題
R7 と R8 はデュアルホーム構成で OSPF を稼働させています。R7 の `GigabitEthernet0/1` で `ip verify unicast source reachable-via rx` を設定したところ、非対称パス経由で到達する正常な通信がドロップされる障害が発生しました。障害原因を特定し、通信を復旧させてください。

#### 要件
1. パケットドロップの原因が uRPF Strict Mode と非対称パスの競合であることを確認すること。
2. uRPF の検証モードを `reachable-via any` (Loose Mode) に変更して障害を解決すること。

#### 障害原因の特定 & 設定修正例
```bash
! 1. 障害確認: debug ip verify で RPF failure ログを確認
R7# debug ip verify
*Nov  4 10:00:12.123: %IP_URPF-4-FAIL: IP RPF failure for packet from 172.16.1.50 on GigabitEthernet0/1

! 2. CEF テーブルの確認 (172.16.1.50 の最善経路は Gi0/2 を指している)
R7# show ip cef 172.16.1.50
172.16.1.0/24
  nexthop 10.1.1.2 GigabitEthernet0/2

! 3. 修正設定: Strict Mode から Loose Mode へ修正
interface GigabitEthernet0/1
 no ip verify unicast source reachable-via rx
 ip verify unicast source reachable-via any
```

#### 検証方法
```bash
R7# show ip interface GigabitEthernet0/1 | include verify
  IP verify unicast reverse-path verification is enabled
  IP verify source reachable-via ANY
```


## ❓ 想定試験問題

CCIE Security v6.1 ラボ試験および記述問題を想定したクイズ形式の試験問題および詳細解説です。

---

### 問1: コンフィグ読解
以下のルータコンフィグが適用されたインターフェイス `GigabitEthernet0/1` において、送信元 IP アドレスが `192.168.100.5` であるパケットが入力されました。ルータの CEF FIB テーブルにおいて `192.168.100.0/24` への最善経路（Best Path）は `GigabitEthernet0/2` を向いています。このパケットはルータによってどのように処理されるか、正しく説明しているものを選択してください。

```bash
ip access-list extended URPF_ALLOW
 permit ip 192.168.100.0 0.0.0.255 any
 deny   ip any any

interface GigabitEthernet0/1
 ip address 203.0.113.1 255.255.255.0
 ip verify unicast source reachable-via rx URPF_ALLOW
```

* **A**: パケットの入力ポート (Gi0/1) と FIB の出力ポート (Gi0/2) が異なるため、即座にドロップされる。
* **B**: ACL `URPF_ALLOW` で `permit` されるため、uRPF 検証はスキップ（成功）となり、宛先 IP に基づいて正常に転送される。
* **C**: ACL で `permit` されているため、Loose モード相当で再評価され、FIB にルートが存在するため転送される。
* **D**: `allow-default` が設定されていないため、デフォルトルートにフォールバックされてドロップされる。

**正解**: **B**

**解説**:
`ip verify unicast source reachable-via rx <ACL>` 構文において、指定された ACL で `permit` にマッチするパケットは、uRPF の Reverse-Path 評価をバイパス（成功として処理）します。本問では `192.168.100.5` が `URPF_ALLOW` の `permit` にマッチするため、入力ポートと FIB の不一致にかかわらずドロップされず正常に転送されます。

---

### 問2: トラブルシューティング
ネットワーク管理者から「エンタープライズエッジルータ R1 の WAN インターフェイスで `ip verify unicast source reachable-via rx` を設定した直後から、インターネット上の任意の Web サイトからのレスポンスパケットがすべてドロップされるようになった」と報告がありました。FIB にはデフォルトルート（`0.0.0.0/0 via ISP_Gateway`）のみが存在しています。原因と最も適切な対処方法の組み合わせを選択してください。

* **A**: CEF が無効化されているため。`ip cef` を有効化する。
* **B**: `reachable-via rx` はデフォルトルートを無視して検証するため。設定を `ip verify unicast source reachable-via rx allow-default` に修正する。
* **C**: 非対称ルーティングが発生しているため。設定を `ip verify unicast source reachable-via any` に変更する。
* **D**: Ingress ACL の暗黙の deny に引っかかっているため。`ip access-group` を修正する。

**正解**: **B**

**解説**:
uRPF (Strict / Loose 共に) は、デフォルトでは CEF FIB テーブル上のデフォルトルート（`0.0.0.0/0`）を「有効なリバースパス」として評価しません。明示的な特定サブネットへのルートが存在せず、デフォルトルートのみで外部と通信している環境で `allow-default` を付けずに uRPF を有効化すると、外部からの戻りトラフィックがすべてドロップされます。したがって `allow-default` オプションの追加が適切な対処となります。

---

### 問3: Design
2つの異なる ISP (ISP-A, ISP-B) に BGP で接続されたマルチホーム（Multi-homed）環境のエンタープライズエッジネットワークにおいて、RFC 2827 (BCP 38) に準拠した送信元 IP スプーフィング対策を実装する予定です。ISP-A から受信したトラフィックの戻りパケットが ISP-B から送信されるような非対称ルーティング（Asymmetric Routing）が発生する可能性があります。正常な通信を妨げずに IP スプーフィングパケットを遮断するための最も適切な設計を選択してください。

* **A**: インターフェイスに `ip verify unicast source reachable-via rx` を適用し、完全な Strict 検証を行う。
* **B**: インターフェイスに `ip verify unicast source reachable-via any` (Loose Mode) を適用する。
* **C**: インターフェイスに `ip verify reverse-path` を適用し、接続テーブルのステート検証を行う。
* **D**: uRPF は使用せず、すべての送信元 IP アドレスを個別に記述した Static Ingress ACL のみを適用する。

**正解**: **B**

**解説**:
非対称ルーティングが発生するマルチホーム環境で Strict Mode (`reachable-via rx`) を使用すると、別ポートから入ってきた合法的なトラフィックが誤ドロップされます。このような環境では、送信元 IP アドレスがルーティングテーブル（FIB）内に存在するか（到達可能か）のみを検証する Loose Mode (`reachable-via any`) を適用するのがベストプラクティスです。これにより、未割り当てのアドレスや偽装された Bogon アドレスからのパケットのみを効果的にドロップできます。

---

### 問4: コンフィグ読解 (Cisco ASA)
Cisco ASA において以下のコマンドを適用しました。この設定の動作として正しいものを選択してください。

```bash
ciscoasa(config)# ip verify reverse-path interface DMZ
```

* **A**: DMZ インターフェイスから出力されるパケットの宛先 IP を検証する。
* **B**: DMZ インターフェイスに入力されるパケットの送信元 IP をルーティングテーブルと照合し、反転パスが存在しないパケットをドロップする。
* **C**: DMZ インターフェイスで Loose モードの uRPF を有効化する。
* **D**: DMZ インターフェイスの全パケットに対して ARP スプーフィングを検証する。

**正解**: **B**

**解説**:
Cisco ASA における `ip verify reverse-path interface <interface_name>` コマンドは、指定したインターフェイスで uRPF (Anti-Spoofing) 機能を有効化します。入力されたパケットの送信元 IP アドレスに対して ASA のルーティングテーブルを参照し、そのインターフェイス経由で到達不能（Reverse Path 不一致）なパケットをドロップします。

---

### 問5: トラブルシューティング
ルータ R1 のインターフェイス `GigabitEthernet0/0` で `ip verify unicast source reachable-via rx` を設定しました。しかし、`show cef interface GigabitEthernet0/0` コマンドで確認したところ、uRPF のドロップカウンタが一切増加せず、明らかに偽装された IP アドレスを持つパケットも通過してしまいます。考えられる根本原因として最も可能性が高いものを選択してください。

* **A**: グローバルで `ip cef` が無効化されているため、uRPF が機能していない。
* **B**: `allow-default` オプションが設定されているため。
* **C**: インターフェイスで IPv6 が無効化されているため。
* **D**: ルータの CPU 利用率が 100% に達しているため。

**正解**: **A**

**解説**:
uRPF (Unicast Reverse Path Forwarding) は、パケット評価を CEF (Cisco Express Forwarding) の FIB テーブルに完全依存しています。グローバルで CEF (`ip cef`) が無効化されている場合、uRPF は正常に機能せず、検証が行われないままパケットが通過するか、あるいは期待されたフィルタリング動作が行われません。

---

## 🔗 参考リソース

Cisco 公式ドキュメントおよび標準規格の参照一覧です。

* RFC 2827 - Network Ingress Filtering: Defeating Denial of Service Attacks which employ IP Source Address Spoofing
* BCP 38 - Network Ingress Filtering (IETF BCP)
* Cisco IOS-XE Security Configuration Guide: Understanding Unicast Reverse Path Forwarding
* Cisco Command Reference: ip verify unicast source
* Cisco ASA Series CLI Configuration Guide: Configuring Anti-Spoofing (uRPF)
* Cisco Secure Firewall Management Center (FMC) Configuration Guide: Anti-Spoofing Protection
* Cisco Technical Notes: Unicast Reverse Path Forwarding (uRPF) Enhancements

---

## 📝 補足（Notes）

### 1. uRPF モードの比較とチェックシート

| チェック項目 | Strict Mode (`rx`) | Loose Mode (`any`) | Feasible Path (`rx`) |
| --- | --- | --- | --- |
| パケットの入力ポート制限 | 要求あり（FIBのBestポート） | 制限なし（FIB存在のみ） | 要求あり（FIBの代替ポート群） |
| 非対称パスの許容 | 不可 | **可能** | 条件付き可能（AS内複数パス） |
| 主な適用領域 | Access / Enterprise Edge | Core / Transit / ISP | Multi-homed Enterprise Edge |

### 2. トラブルシューティングのクイックフロー

```text
[ 通信障害 / uRPF ドロップ発生 ]
         │
         ▼
1. CEF の稼働状態確認 ──► (show ip cef) ──► 無効なら `ip cef` を実行
         │
         ▼
2. 最善経路の確認 ────► (show ip cef <Src_IP>) ──► 入力ポートと FIB 出力ポートの比較
         │
         ▼
3. デフォルトルート依存確認 ──► デフォルト経由通信なら `allow-default` 付与
         │
         ▼
4. 非対称パスの検出 ──► 非対称パスが存在すれば `reachable-via any` へ変更
```


