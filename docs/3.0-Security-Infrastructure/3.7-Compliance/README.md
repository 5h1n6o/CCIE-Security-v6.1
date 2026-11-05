---
layout: default
title: 3.7-Compliance
nav_order: 7
parent: 3.0-Security-Infrastructure
---

# 3.7 BCP 38 (Network Ingress Filtering)

## 📘 概要

BCP 38 (Best Current Practice 38 / RFC 2827: Network Ingress Filtering: Defeating Source Address Spoofing) は、インターネットおよびエンタープライズネットワーク境界において、**送信元IPアドレスの偽装 (Source IP Address Spoofing)** を防止するために定義されたセキュリティ標準ガイドラインです。

ネットワーク攻撃者（ボットネットや悪意のあるホスト）は、自身の身元を隠蔽したり、DNS/NTP/SNMP アンプ（増幅）DDoS攻撃の踏み台として利用したり、TCP SYN Flood 攻撃の追跡を逃れるために、偽装された送信元IPアドレス（Spoofed Source IP）を持つパケットを送信します。BCP 38 は、アクセスネットワークやエンタープライズのIngress境界で「そのインターフェイスから送信されるはずのない送信元IPアドレス」を持つパケットを検知・ドロップすることで、送信元IP偽装を根本から無効化します。

主な導入理由と用途は以下の通りです。

* **DDoS（反射・増幅攻撃）の無効化**: DNS/NTP/Memcached リフレクション攻撃など、犠牲者のIPアドレスに送信元を偽装してリクエストを大量送信する攻撃を境界で遮断。
* **Man-in-the-Middle (MitM) やセッションハイジャックの抑制**: 偽装IPによる不正な接続試行やルーティング汚染の防止。
* **トレースバックと監査の容易化**: 送信元IPアドレスの信頼性が保証されるため、セキュリティインシデント発生時のフォレンジック解析や攻撃者特定が迅速化。
* **RFC 3704 (BCP 84) への拡張**: マルチホーム環境や非対称ルーティング環境に対応するため、uRPF (Unicast Reverse Path Forwarding) の Strict モード、Loose モード、Feasible Path モードなどの多様な実装メカニズムを提供。


---

## 🔑 要点

BCP 38 の主要な技術的特徴、実装方式、制限事項、設計上の注意点を下表に示します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | ネットワーク境界（アクセス層・エッジ層）で、割り当てられたIPプレフィックス以外の送信元IPを持つパケットをフィルタリング・ドロップする標準規範 |
| **主な実装技術** | Strict uRPF、Loose uRPF、Feasible Path uRPF、Ingress Static ACL、IP Source Guard (IPSG) |
| **用途** | 送信元IP偽装（IP Spoofing）防止、DDoS反射攻撃（NTP/DNS Amp）の低減、ネットワーク境界でのトラフィック無効化 |
| **メリット** | 偽装トラフィックの自組織外への漏出防止、セキュリティ監査性の向上、反射型DDoS攻撃の根絶 |
| **デメリット / 課題** | 全自律システム (AS) やプロバイダで一斉に適用されないとインターネット全体での完全な効果が得られない、非対称ルーティング環境での誤ドロップリスク（Strict uRPF時） |
| **対応機種** | Cisco IOS / IOS-XE Router & Switch、Cisco ASA Firewalls、Cisco FTD (Secure Firewall)、Cisco Nexus |
| **制限事項** | 非対称ルーティング（IngressとEgressで経路が異なる）構成で Strict uRPF を使用すると正常トラフィックが破棄される |
| **設計上の注意点** | デフォルトルート (`0.0.0.0/0`) の扱い（`allow-default` オプションの有無）、非対称経路における Strict / Loose / Feasible Path モードの適切な選定 |


---

## 🏗 動作原理

BCP 38 の核心概念は、「インターフェイスに到着したパケットの送信元IPアドレス（Source IP）が、そのインターフェイス経由で到達可能な正規のネットワーク範囲内にあるか」を検証することです。Cisco ルータおよびファイアウォールでは、主に以下の4つの技術を用いて BCP 38 を実装します。

```
[ クライアント / 攻撃者 ] (送信元IP: 192.168.1.100 または 偽装IP: 10.1.1.1)
           │
           │ Ingress Packet (Src: 10.1.1.1 / Dst: 203.0.113.50)
           ▼
[ ルータ / ファイアウォール Ingress Interface: Gi0/1 (割り当てサブネット: 192.168.1.0/24) ]
           │
           ├─► 【BCP 38 検証 1: Static Ingress ACL】
           │     192.168.1.0/24 以外を permit しない ──► 偽装 10.1.1.1 を DROP
           │
           └─► 【BCP 38 検証 2: Unicast RPF (uRPF)】
                 FIB (Forwarding Information Base) テーブルを参照
                 - Strict Mode: 10.1.1.1 への最適経路インターフェイスが Gi0/1 か？
                   └─► 違えば (Gi0/2 など) ──► DROP
                 - Loose Mode: 10.1.1.1 への経路が FIB に存在するか？
                   └─► 存在しなければ (Null0 除く) ──► DROP
```

### 1. Ingress ACLs (Static Network Filtering)
最も標準的でシンプルな方法です。アクセスポートやスタブネットワークの物理インターフェイスに対して、許可された送信元IPサブネットのみを `permit` し、それ以外を `deny` する ACL をインバウンド適用します。

### 2. Strict uRPF (Strict Unicast Reverse Path Forwarding)
パケットが到着したインターフェイス（Ingress Interface）と、FIB テーブルにおけるそのパケットの「送信元IPアドレス（Source IP）」へのベストパス（最良経路）の出力インターフェイスが**完全一致**するかを検証します。
* 一致した場合: パケットを許可してルーティング処理を実施。
* 不一致の場合: パケットを偽装と判断して即座にドロップ。
* **適用場面**: 対称ルーティング（Symmetric Routing）が保証されているシングルホームのスタブネットワークやアクセスポート。

### 3. Loose uRPF (Loose Unicast Reverse Path Forwarding)
FIB テーブルに送信元IPアドレス（Source IP）への有効な経路（Valid Route）が存在するかどうかのみをチェックします（どのインターフェイスから到着したかは問いません）。
* 経路が存在する場合: パケットを許可。
* 経路が存在しない場合 (Unreachable): パケットをドロップ。
* **適用場面**: 非対称ルーティング（Asymmetric Routing）が発生するマルチホームのISPエッジやコア境界。

### 4. Feasible Path uRPF (RFC 3704 / BCP 84)
FIBのベストパスだけでなく、BGPやEIGRP等で維持されている「実行可能経路（Feasible Path / Alternative Routes）」も検証対象に含めます。到着インターフェイスがベストパスでなくても、実行可能経路の1つであればドロップしません。


---

## ⚙ 動作シーケンス

Cisco IOS-XE / ASA / FTD における BCP 38 (uRPF および Ingress ACL) のパケット処理シーケンスは以下のステップで実行されます。

```
[ パケット受入 (Ingress Packet Arrival) ]
               │
               ▼
   [ Layer 2 / MAC Address チェック ]
               │
               ▼
   [ Ingress ACL / VACL 評価 ] ──► (ACL に拒否ルールがあれば即座に DROP)
               │
               ▼
   [ uRPF (Unicast RPF) ルックアップ ]
               │
               ├─► (モード判定)
               │
               ├─► 【Strict uRPF】:
               │      FIB テーブルで Source IP を検索。
               │      FIB の出力 IF == 到着 IF か？
               │      ├─► Yes ──► パケット許可 (FIB ルーティングへ)
               │      └─► No  ──► パケットドロップ (Drop Counter インクリメント)
               │
               ├─► 【Loose uRPF】:
               │      FIB テーブルで Source IP を検索。
               │      FIB に有効な非 Null0 経路が存在するか？
               │      ├─► Yes ──► パケット許可 (FIB ルーティングへ)
               │      └─► No  ──► パケットドロップ (Drop Counter インクリメント)
               │
               └─► 【allow-default オプション付き】:
                      送信元IPが FIB に存在せずデフォルトルートのみの場合
                      ├─► allow-default あり ──► パケット許可
                      └─► allow-default なし ──► パケットドロップ
               │
               ▼
   [ Egress ACL / PBR / NAT / Routing 処理 ]
               │
               ▼
   [ パケット送出 (Egress Packet Forwarding) ]
```


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Securityラボ試験において、BCP 38 (uRPF / Ingress ACL) は「ネットワークの不当な偽装トラフィックの遮断」「DDoS攻撃対策」「特定要件に基づくインバウンドフィルタリング」として出題されます。

### 1. 試験で狙われやすいポイントとトラブル要因

* **非対称ルーティング（Asymmetric Routing）での Strict uRPF 誤用**:
  * 2台以上のルータでマルチホーム構成（BGP等）を組んでいる環境や、ECMP (Equal-Cost Multi-Path) が動いている環境で Ingress インターフェイスに `ip verify unicast source reachable-via rx` (Strict) を設定すると、戻りトラフィックの送信元IPアドレスが別インターフェイス経由のベストパスとなっている場合に通信が遮断されます。
  * **対策**: 非対称ルーティング環境では `ip verify unicast source reachable-via any` (Loose) を使用するか、`allow-default` や Self-ACL (`allow-self-ping`) 等の設計上の注意を払う必要があります。
* **デフォルトルート (`0.0.0.0/0`) の罠**:
  * 端末やルータがデフォルトルートしか持たない場合、`ip verify unicast source reachable-via rx` のみでは、デフォルトルートも「有効な経路」と判断され、あらゆる送信元IPが許可されてしまい uRPF の効果が失われます。逆に、デフォルトルート経由の送信元IPを拒否したい場合は、`allow-default` オプションを外すか、明示的な iACL (Infrastructure ACL) や BGP ルートで厳密に制御する必要があります。
  * `allow-default` を付与しない場合、FIBに明示的な個別の経路が存在しない送信元からのパケットはすべてドロップされます。
* **ASA / FTD での Anti-Spoofing の挙動**:
  * Cisco ASA / FTD では、`ip verify reverse-path interface <interface_name>` コマンドで uRPF (Anti-Spoofing) を有効化します。
  * ASA の Anti-Spoofing は **Strict モード** で動作します。非対称ルーティングが発生する ASA セグメントでは通信障害の原因となるため、注意が必要です。

### 2. show コマンドおよび debug ログによる状態判断

* **`show ip interface <interface_name>`**:
  * uRPF が有効化されているか、および uRPF によってドロップされたパケット数（IP verify source drop count）を確認できます。
* **`show ip traffic`**:
  * ルータ全体での Unicast RPF ドロップパケットの総計を確認可能です。
* **`show cef interface <interface_name>`**:
  * CEF (Cisco Express Forwarding) テーブルにおける uRPF 設定状態を確認できます。


---

## 🛠 設定方法

Cisco IOS-XE、Cisco ASA、および Cisco FTD (FMC) における BCP 38 (uRPF & Ingress ACL) の設定手順を示します。

### 1. Cisco IOS-XE (Router / Catalyst Switch)

#### (a) Strict uRPF の設定 (スタブネットワーク / アクセスインターフェイス向け)
```text
interface GigabitEthernet0/1
 description Access-Network-Interface
 ip address 192.168.10.1 255.255.255.0
 ip verify unicast source reachable-via rx
```

#### (b) Loose uRPF + デフォルトルート不許可の設定 (ISPエッジ / 非対称ルーティング向け)
```text
interface GigabitEthernet0/2
 description Multi-Homed-ISP-Interface
 ip address 203.0.113.1 255.255.255.252
 ip verify unicast source reachable-via any
```

#### (c) Loose uRPF + デフォルトルートを有効な経路として許可する場合
```text
interface GigabitEthernet0/2
 description ISP-Interface-With-Default-Route
 ip address 203.0.113.1 255.255.255.252
 ip verify unicast source reachable-via any allow-default
```

#### (d) Ingress ACL による静的 BCP 38 の設定
```text
ip access-list extended BCP38_INGRESS_ACL
 deny   ip 10.0.0.0 0.255.255.255 any
 deny   ip 172.16.0.0 0.15.255.255 any
 deny   ip 192.168.0.0 0.0.255.255 any
 deny   ip 127.0.0.0 0.255.255.255 any
 permit ip 192.168.10.0 0.0.0.255 any
 deny   ip any any log-input

interface GigabitEthernet0/1
 ip access-group BCP38_INGRESS_ACL in
```

### 2. Cisco ASA (CLI)

ASA ではインターフェイス単位で Anti-Spoofing (Strict uRPF) を有効化します。

```text
! outside インターフェイスで Anti-Spoofing (Strict uRPF) を有効化
ip verify reverse-path interface outside
```

### 3. Cisco FTD (FMC Managed)

FMC (Firepower Management Center) では、Device Configuration から Anti-Spoofing を設定します。

1. FMC GUI にログインし、**Devices** > **Device Management** を開く。
2. 対象の FTD デバイスを選択し、**Edit** (鉛筆アイコン) をクリック。
3. **Device** タブ内の **Anti-Spoofing** 設定エリアに移動。
4. 有効化したいインターフェイス（例: `outside`, `inside`）のチェックボックスをオンにする。
5. **Save** 後、対象 FTD に **Deploy** を実行する。


---

## 🔍 検証コマンド

BCP 38 (uRPF / Anti-Spoofing) の動作確認およびトラブルシューティングで使用する検証コマンドを下表にまとめます。

| 目的 | コマンド |
| --- | --- |
| インターフェイス単位の uRPF 有効化状態およびドロップ数確認 | `show ip interface GigabitEthernet0/1` |
| ルータ全体の uRPF ドロップ統計確認 | `show ip traffic | include RPF` |
| CEF レベルでの uRPF 設定状態確認 | `show cef interface GigabitEthernet0/1` |
| uRPF ドロップパケットのドロップ理由確認 (IOS-XE) | `show ip traffic` または `show cef drop` |
| ASA での Anti-Spoofing 有効化状態確認 | `show running-config ip verify` |
| ASA での Anti-Spoofing ドロップ統計確認 | `show asp drop frame rpf-violation` |
| FTD CLI での ASP ドロップ確認 | `show asp drop frame rpf-violation` |
| uRPF パケット検証のリアルタイムデバッグ | `debug ip cef drop` / `debug ip packet` |


---

## 🚨 トラブルシュート

BCP 38 (uRPF / Anti-Spoofing) 導入時によく発生するトラブル症状、根本原因、確認コマンド、および対処方法を下表にまとめます。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| 非対称ルーティング環境で特定拠点の通信が突然全滅する | Strict uRPF (`reachable-via rx`) が有効化されており、戻りパケットの送信元IPアドレスのベストパスが到着IFと一致しない | `show ip interface <IF>`<br>`show ip route <Src_IP>` | uRPF モードを Strict (`rx`) から Loose (`any`) に変更する |
| デフォルトルートのみの環境で偽装トラフィックがドロップされない | Loose/Strict uRPF で `allow-default` が指定されており、デフォルトルートがすべての送信元IPに対する有効経路として認識されている | `show running-config \| inc ip verify`<br>`show ip route` | `allow-default` オプションを削除するか、静的 Ingress ACL を併用して送信元IPを制限する |
| BGP Multi-Homed 環境で一部の対向ASからの通信がドロップされる | 対向ASからのマルチホーム経路が自ルータのFIBでベストパスになっていないため Strict uRPF で破棄される | `show ip route <Src_IP>`<br>`show ip cef <Src_IP>` | uRPF モードを `reachable-via any` (Loose) に変更するか、Feasible Path uRPF を検討する |
| ASA / FTD 経由のVPN/ルーティング通信が拒否される | ASA の Anti-Spoofing (`ip verify reverse-path`) が有効で、内部ルータ等での非対称経路が発生している | `show asp drop frame rpf-violation`<br>`show running-config ip verify` | ASA の Anti-Spoofing を解除するか、ASA を通過するトラフィックが完全に対称となるようルーティングを修正する |


## ⚠ 制限事項

BCP 38 (uRPF / Ingress ACL) の導入における主な制限事項およびプラットフォーム固有の制約は以下の通りです。

1. **非対称ルーティング（Asymmetric Routing）における制限**:
   * Strict uRPF (`reachable-via rx`) は、往路と復路が同一物理/論理パスを通過する対称ルーティング環境でのみ正常に動作します。ロードバランシングやマルチホームISP環境で Strict uRPF を使用することはできません。
2. **マルチキャストトラフィック非対応**:
   * uRPF はユニキャストパケット（Unicast IP Packets）の送信元検証に特化しています。マルチキャストパケット（Multicast IP Packets）の送信元検証には PFD / RPF Check（PIM RPF Check）が利用されるため、`ip verify unicast source` の適用対象外となります。
3. **ハードウェア / ASIC スイッチングの制限**:
   * 一部の Catalyst スイッチや低エンドルータでは、uRPF ルックアップが TCAM (Ternary Content Addressable Memory) で処理されず、CPU スイッチング (Punt to CPU) が発生してスループットが低下したり CPU 使用率が高騰する場合があります。
4. **Tunnel / IPsec / GRE インターフェイスでの制限**:
   * GRE トンネルや IPsec 仮想VTI インターフェイス上で uRPF を有効化する場合、カプセル化前のアウターIP（Outer IP）とインナーIP（Inner IP）の評価順序に注意が必要です。プラットフォームによってはトンネルインターフェイス上での uRPF がサポートされない場合があります。


---

## 🔄 他技術との関連

BCP 38 は、単一の機能ではなく、複数のセキュリティ基盤技術と連携・組み合わせることで最大の効果を発揮します。

* **iACL (Infrastructure Access Control List)**:
  * インフラストラクチャ（ルータ自身のLoopbackやPoint-to-Point IP）への直接攻撃を防止する iACL と BCP 38 (Ingress ACL / uRPF) を組み合わせることで、ルータコントロールプレーンの保護（CoPP等）を多層化。
* **RTBH (Remotely Triggered Black Hole) & BGP Flowspec**:
  * BCP 38 によって送信元IPアドレスの信頼性が担保されている環境において、DDoS攻撃発生時に特定ターゲットIP（Destination RTBH）または攻撃元IP（Source RTBH）を BGP 経由で Null0 に黒穴化する RTBH や BGP Flowspec が極めて効果的に機能。
* **IP Source Guard (IPSG) & DHCP Snooping**:
  * Layer 2 アクセススイッチ層で BCP 38 を実現する技術。DHCP Snooping バインディングテーブル（IP-MAC-Port-VLAN）に基づいて、未認証の IP Spoofing パケットをポート単位でドロップ。
* **CoPP (Control Plane Policing)**:
  * 偽装された IP パケットが CPU（Control Plane）へ大量送信される DOS 攻撃を、uRPF と CoPP の組み合わせで二重防御。


---

## 🧩 比較表

### 1. uRPF 動作モードの比較表

| 項目 | Strict Mode (`rx`) | Loose Mode (`any`) | Feasible Path Mode |
| --- | --- | --- | --- |
| **検証基準** | FIBの「ベストパスの出力IF」と「到着IF」が完全一致 | FIBに「有効な経路」が存在する（IF問わず） | FIBの「ベストパス＋実行可能経路」の出力IFのいずれかと一致 |
| **非対称経路対応** | 非対応（ドロップされる） | **完全対応** | **対応**（バックアップ経路経由も許可） |
| **デフォルトルートの扱い** | `allow-default` で許可可能 | `allow-default` で許可可能 | `allow-default` で許可可能 |
| **主な適用場所** | アクセス層、スタブAS、シングルホーム | ISPコア、マルチホームBGPエッジ | マルチホームBGPエッジ、EIGRPデュアルホーム |
| **セキュリティ強度** | **最高** (IP偽装を完全排除) | **中** (未割り当て/未広報IPのみ排除) | **高** (実行可能パス以外の偽装を排除) |

### 2. BCP 38 実装手法の比較表

| 項目 | Ingress Static ACL | Unicast RPF (uRPF) | IP Source Guard (IPSG) |
| --- | --- | --- | --- |
| **動作レイヤー** | Layer 3 / Layer 4 | Layer 3 (FIBベース) | Layer 2 (Port/VLANベース) |
| **設定・メンテ** | 手動設定（IP変更時に手動更新） | **動的・自動** (Routing Table連動) | **動的・自動** (DHCP Snooping連動) |
| **管理コスト** | 高い（プレフィックス変更時に修正必要） | **極めて低い** | **極めて低い** |
| **適用ポイント** | 境界ルータ、特定アクセスポート | 境界ルータ、SVI、Edge Interface | L2 アクセススイッチポート |


---

## 💡 ベストプラクティス

BCP 38 (Network Ingress Filtering) を設計・運用する際の Cisco 推奨ベストプラクティスは以下の通りです。

1. **アクセス層およびスタブ境界での Strict uRPF 徹底**:
   * シングルホームの拠局やアクセススイッチと接続する L3 インターフェイスには、必ず **Strict uRPF (`ip verify unicast source reachable-via rx`)** を適用し、送信元IP偽装を最前線で遮断する。
2. **マルチホーム境界での Loose uRPF と Ingress ACL の併用**:
   * 複数のISPと BGP 接続しているマルチホーム境界では、**Loose uRPF (`ip verify unicast source reachable-via any`)** を使用して非対称ルーティングによる誤ドロップを防止する。
   * さらに、RFC 1918 プライベートIP（`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`）、ループバック（`127.0.0.0/8`）、ドキュメント用IP（`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`）などの Bogon IP を遮断する **Static Ingress ACL** を明示的にインバウンド適用する。
3. **`allow-default` オプションの厳格な管理**:
   * ルータがデフォルトルートしか保持していない環境（デフォルトルートのみ受信するスタブエッジ等）で `allow-default` を有効にすると、uRPF のフィルタリング機能が実質的に無効化されるため、非推奨。必要な場合は個別の明示的なプレフィックスをルーティングテーブルに保持させるか、Ingress ACL を使用する。
4. **ロギングと監視の有効化**:
   * uRPF によるドロップログを取得するため、ドロップが発生した際に SNMP トラップや Syslog を送信する設定を行い、DDoS 攻撃や誤設定を即座に検知可能な体制を整える。


## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験レベルの実践的な BCP 38 (uRPF / Ingress ACL) 設定サンプル例を 10 個掲載します。

### シナリオ 1: アクセスポートへの Strict uRPF 適用

#### 問題
R1 は企業アクセス層のルータです。GigabitEthernet0/1（割り当てサブネット: `192.168.10.0/24`）に接続された端末からの送信元 IP アドレス偽装（IP Spoofing）を防止するため、Strict uRPF を設定してください。

#### 要件
* GigabitEthernet0/1 に Strict uRPF を設定すること。
* 192.168.10.0/24 以外の送信元 IP アドレスを持つパケットが Gi0/1 に到着した場合は即座にドロップすること。

#### 設定例
```text
! R1 (Cisco IOS-XE)
interface GigabitEthernet0/1
 description ACCESS_PORT_192.168.10.0/24
 ip address 192.168.10.1 255.255.255.0
 ip verify unicast source reachable-via rx
```

#### 検証方法
```text
R1# show ip interface GigabitEthernet0/1 | include IP verify
  IP verify source unreachable drops: 1545

R1# show ip traffic | include RPF
  0 RPF drops, 0 path drop
```

---

### シナリオ 2: マルチホーム ISP 境界における Loose uRPF 設定

#### 問題
R2 は2社の ISP (ISP-A, ISP-B) と BGP で接続されたエッジルータです。非対称ルーティングが発生する環境において、送信元 IP 偽装を防止しつつ正常通信を維持するため、GigabitEthernet0/2 (ISP-A) および GigabitEthernet0/3 (ISP-B) に Loose uRPF を設定してください。

#### 要件
* Gi0/2 および Gi0/3 で Loose uRPF を有効化すること。
* ルーティングテーブルに存在しない未割り当て・偽装 IP アドレスからのパケットをドロップすること。

#### 設定例
```text
! R2 (Cisco IOS-XE)
interface GigabitEthernet0/2
 description CONNECT_TO_ISP_A
 ip address 203.0.113.1 255.255.255.252
 ip verify unicast source reachable-via any

interface GigabitEthernet0/3
 description CONNECT_TO_ISP_B
 ip address 203.0.113.5 255.255.255.252
 ip verify unicast source reachable-via any
```

#### 検証方法
```text
R2# show ip interface GigabitEthernet0/2 | include IP verify
  IP verify source unreachable drops: 230
```

---

### シナリオ 3: デフォルトルート存在環境における uRPF 設定 (`allow-default`)

#### 問題
R3 は上流ルータからデフォルトルート (`0.0.0.0/0`) のみを受信しています。GigabitEthernet0/1 から到着するトラフィックに対して Loose uRPF を適用し、デフォルトルートを「有効な到達可能経路」として認める設定を行ってください。

#### 要件
* Gi0/1 で Loose uRPF を有効化し、`allow-default` オプションを付与すること。

#### 設定例
```text
! R3 (Cisco IOS-XE)
interface GigabitEthernet0/1
 description UPSTREAM_INTERFACE
 ip address 10.1.1.2 255.255.255.252
 ip verify unicast source reachable-via any allow-default
```

#### 検証方法
```text
R3# show ip interface GigabitEthernet0/1 | include IP verify
  IP verify source unreachable drops: 0
```

---

### シナリオ 4: Ingress ACL による Bogon IP 偽装防止フィルタリング

#### 問題
R4 の WAN インターフェイス GigabitEthernet0/0 において、プライベート IP（RFC 1918）、ループバックアドレス、および未割り当て IP からの偽装パケットを遮断する Ingress ACL を構築してください。

#### 要件
* RFC 1918 (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)、127.0.0.0/8、0.0.0.0/8 からのインバウンドパケットを破棄・ログ記録すること。
* それ以外のパケットを許可すること。

#### 設定例
```text
! R4 (Cisco IOS-XE)
ip access-list extended BCP38_BOGON_FILTER
 deny   ip 10.0.0.0 0.255.255.255 any log-input
 deny   ip 172.16.0.0 0.15.255.255 any log-input
 deny   ip 192.168.0.0 0.0.255.255 any log-input
 deny   ip 127.0.0.0 0.255.255.255 any log-input
 deny   ip 0.0.0.0 0.255.255.255 any log-input
 permit ip any any

interface GigabitEthernet0/0
 description WAN_INTERFACE
 ip access-group BCP38_BOGON_FILTER in
```

#### 検証方法
```text
R4# show ip access-lists BCP38_BOGON_FILTER
Extended IP access list BCP38_BOGON_FILTER
    10 deny ip 10.0.0.0 0.255.255.255 any log-input (42 matches)
    20 deny ip 172.16.0.0 0.15.255.255 any log-input
    30 deny ip 192.168.0.0 0.0.255.255 any log-input (15 matches)
    40 deny ip 127.0.0.0 0.255.255.255 any log-input
    50 deny ip 0.0.0.0 0.255.255.255 any log-input
    60 permit ip any any (12402 matches)
```

---

### シナリオ 5: Cisco ASA における Anti-Spoofing (Strict uRPF) の設定

#### 問題
Cisco ASA5516 の `outside` インターフェイスおよび `dmz` インターフェイスにおいて、ルーティングテーブルに基づいた Anti-Spoofing（Strict uRPF）を有効化してください。

#### 要件
* `outside` および `dmz` インターフェイスで Anti-Spoofing を有効化すること。

#### 設定例
```text
! ASA (CLI)
ip verify reverse-path interface outside
ip verify reverse-path interface dmz
```

#### 検証方法
```text
ASA# show running-config ip verify
ip verify reverse-path interface outside
ip verify reverse-path interface dmz

ASA# show asp drop frame rpf-violation
Frame drop code                               - Cases
Reverse-path verify failed (rpf-violation)    - 84
```

---

### シナリオ 6: Cisco FTD (FMC) における Anti-Spoofing 設定

#### 問題
FMC 管理下の FTD1 デバイスにおいて、`outside_fg` インターフェイスグループに対して Anti-Spoofing 機能を有効化してください。

#### 要件
* FMC デバイス設定から Anti-Spoofing を有効化し、デプロイすること。

#### 設定例
```text
1. FMC GUI: Devices > Device Management > FTD1 を編集
2. Device タブ > Anti-Spoofing エリア
3. [outside_fg] インターフェイスの "Enable Anti-Spoofing" にチェック
4. Save & Deploy
```

#### 検証方法
```text
! FTD CLI 画面での確認
FTD1# show asp drop frame rpf-violation
Reverse-path verify failed (rpf-violation)                12
```

---

### シナリオ 7: uRPF ドロップ発生時の MIB トラップおよび Syslog 通知設定

#### 問題
R5 において、uRPF によるパケットドロップが発生した際に、即座に SNMP トラップおよび Syslog メッセージを出力する設定を行ってください。

#### 要件
* uRPF ドロップ時に SNMP トラップを発行すること。
* ドロップ発生時に Syslog メッセージを生成すること。

#### 設定例
```text
! R5 (Cisco IOS-XE)
snmp-server enable traps ipysource
logging enable
logging host 10.1.1.50

interface GigabitEthernet0/1
 ip verify unicast source reachable-via rx
```

#### 検証方法
```text
R5# show logging | include RPF
*Nov  4 06:40:12.123: %IP_VERIFY-4-UNREACHABLE: IP verify source dropped 125 packets on GigabitEthernet0/1
```

---

### シナリオ 8: 非対称ルーティング環境での Loose uRPF 障害復旧

#### 問題
R6 において、`ip verify unicast source reachable-via rx` (Strict uRPF) が設定されていたため、別経路経由で戻ってくる正規のデータパケットがドロップされていました。これを非対称通信を許可する設定に変更してください。

#### 要件
* GigabitEthernet0/1 の uRPF 設定を Strict から Loose に変更すること。

#### 設定例
```text
! R6 (Cisco IOS-XE)
interface GigabitEthernet0/1
 no ip verify unicast source reachable-via rx
 ip verify unicast source reachable-via any
```

#### 検証方法
```text
R6# show ip interface GigabitEthernet0/1 | include IP verify
  IP verify source unreachable drops: 0 (カウント停止を確認)
```

---

### シナリオ 9: VRF 独立環境における uRPF 設定

#### 問題
R7 の GigabitEthernet0/1.100 は VRF `CUSTOMER_A` に所属しています。この VRF インターフェイスにおいて Strict uRPF を有効化してください。

#### 要件
* VRF `CUSTOMER_A` 内の FIB テーブルを参照して Strict uRPF 検証を実施すること。

#### 設定例
```text
! R7 (Cisco IOS-XE)
ip vrf CUSTOMER_A
 rd 65000:100

interface GigabitEthernet0/1.100
 encapsulation dot1Q 100
 ip vrf forwarding CUSTOMER_A
 ip address 10.100.1.1 255.255.255.0
 ip verify unicast source reachable-via rx
```

#### 検証方法
```text
R7# show ip interface GigabitEthernet0/1.100 | include IP verify
  IP verify source unreachable drops: 12
```

---

### シナリオ 10: Ingress ACL と uRPF を組み合わせた多層 BCP 38 境界保護

#### 問題
R8 の WAN 境界インターフェイス GigabitEthernet0/0 において、Ingress ACL (Bogon IP フィルタ) と Loose uRPF の両方を組み合わせて最高レベルの BCP 38 防御を構築してください。

#### 要件
* Ingress ACL で予約済み / Bogon IP (RFC 1918, 127.0.0.0/8, 0.0.0.0/8) をドロップする。
* Loose uRPF で未割り当て・未広報の動的偽装 IP をドロップする。

#### 設定例
```text
! R8 (Cisco IOS-XE)
ip access-list extended BCP38_MULTI_LAYER
 deny   ip 10.0.0.0 0.255.255.255 any
 deny   ip 172.16.0.0 0.15.255.255 any
 deny   ip 192.168.0.0 0.0.255.255 any
 deny   ip 127.0.0.0 0.255.255.255 any
 permit ip any any

interface GigabitEthernet0/0
 description WAN_BORDER_MULTI_LAYER
 ip address 198.51.100.2 255.255.255.248
 ip access-group BCP38_MULTI_LAYER in
 ip verify unicast source reachable-via any
```

#### 検証方法
```text
R8# show ip access-lists BCP38_MULTI_LAYER
R8# show ip interface GigabitEthernet0/0 | include IP verify
```


## ❓ 想定試験問題

CCIE Security ラボ試験・記述試験を意識した想定問題 8 問と詳細な解答解説を掲載します。

### 問題 1 (トラブルシューティング / 実装)
**問題**: 2台のルータ（R1, R2）で2本の回線を用いて負荷分散を行っている環境において、R1 の GigabitEthernet0/1 に `ip verify unicast source reachable-via rx` を適用した直後、一部の拠点からの通信が不安定になりました。原因と適切な修正方法を説明してください。

* **解答**: 
  * **原因**: 2本の回線による非対称ルーティング（往路と復路が異なるインターフェイスを通過する状態）が発生しており、到着したパケットの送信元 IP アドレスへの最良経路（Best Path）が GigabitEthernet0/1 以外を向いていたため、Strict uRPF (`reachable-via rx`) により正規のパケットがドロップされた。
  * **修正方法**: GigabitEthernet0/1 の uRPF モードを Strict から Loose に変更する (`ip verify unicast source reachable-via any`)。

---

### 問題 2 (コンフィグ読解 / 設計)
**問題**: 以下のコンフィグが設定されたルータ R1 において、ルーティングテーブルにはデフォルトルート `0.0.0.0/0 via 10.1.1.1` のみが存在します。このとき、Ingress インターフェイス GigabitEthernet0/1 に到着した送信元 IP `203.0.113.10`（FIBに個別経路なし）のパケットはドロップされますか、許可されますか？

```text
interface GigabitEthernet0/1
 ip verify unicast source reachable-via rx
```

* **解答**:
  * **結果**: **ドロップされる**。
  * **解説**: `ip verify unicast source reachable-via rx` のみでは、`allow-default` オプションが含まれていないため、デフォルトルートは uRPF の「有効な到達経路」としてカウントされません。FIB に明示的な個別プレフィックス（例: `203.0.113.0/24`）が存在しないため、パケットは破棄されます。デフォルトルート経由での到達を許可したい場合は `allow-default` オプションを追加する必要があります。

---

### 問題 3 (トラブルシューティング / ASA)
**問題**: Cisco ASA の CLI で `show asp drop` を実行したところ、以下の出力が得られました。このドロップが発生している技術的理由と、対象コマンドを説明してください。

```text
Frame drop code                               - Cases
Reverse-path verify failed (rpf-violation)    - 1420
```

* **解答**:
  * **理由**: ASA のインターフェイス上で Anti-Spoofing（Strict uRPF）機能が有効化されており、Ingress インターフェイスに到着したパケットの送信元 IP アドレスが、ASA のルーティングテーブルにおける該当インターフェイスのサブネット/経路と一致しない（非対称ルーティングや不正 IP パケットの到着）ため。
  * **該当コマンド**: `ip verify reverse-path interface <interface_name>`

---

### 問題 4 (設計 / 機能比較)
**問題**: BCP 38 をアクセススイッチ層（Layer 2）で実装する場合に最適な技術と、WAN 境界ルータ（Layer 3）で実装する場合に最適な技術の組み合わせをそれぞれ述べてください。

* **解答**:
  * **Layer 2 アクセススイッチ層**: **IP Source Guard (IPSG)** と **DHCP Snooping** の組み合わせ（バインディングテーブルに基づくポート単位の保護）。
  * **Layer 3 WAN 境界ルータ層**: **Unicast RPF (uRPF)**（Strict または Loose モード）および **Static Ingress ACL** の組み合わせ。

---

### 問題 5 (コンフィグ読解)
**問題**: 以下の Ingress ACL と uRPF が同一インターフェイスに適用されている場合、10.1.1.5（プライベートIP）から送信されたパケットの処理シーケンスはどうなりますか？

```text
ip access-list extended INGRESS_FILTER
 deny ip 10.0.0.0 0.255.255.255 any
 permit ip any any

interface GigabitEthernet0/1
 ip access-group INGRESS_FILTER in
 ip verify unicast source reachable-via rx
```

* **解答**:
  * **処理シーケンス**: パケットは uRPF の検証が行われる前に、**Ingress ACL (INGRESS_FILTER) の `deny ip 10.0.0.0 0.255.255.255 any` ルールによって即座にドロップ**されます。パケット処理パイプラインにおいて Ingress ACL の評価は uRPF の評価よりも先行するためです。

---

### 問題 6 (設計 / BGP)
**問題**: BGP でマルチホーム接続しているエッジルータにおいて、全インターネット経路を受信しています。このルータで BCP 38 を適用する際、非対称ルーティングによる誤ドロップを防ぎつつ、未割当のアドレス（Bogon / Spoofed IP）を遮断するための最小限の設定を述べてください。

* **解答**:
  * WAN インターフェイスに対して **Loose uRPF (`ip verify unicast source reachable-via any`)** を設定します。これにより、BGP ルーティングテーブル（FIB）に存在する正規のインターネットプレフィックスからの通信は非対称であっても許可され、FIB に存在しない未割当・未広報のアドレスからのパケットのみがドロップされます。

---

### 問題 7 (トラブルシューティング)
**問題**: FTD において Anti-Spoofing を設定した後、特定の社内サブネット間通信が疎通不可となりました。FMC GUI および FTD CLI でこの問題を調査・解除する手順を説明してください。

* **解答**:
  1. FTD CLI で `show asp drop frame rpf-violation` を実行し、`rpf-violation` のドロップカウンタが上昇していることを確認する。
  2. FMC GUI にログインし、**Devices** > **Device Management** > **FTD** の編集画面を開く。
  3. **Device** タブの **Anti-Spoofing** エリアで、該当インターフェイスのチェックボックスを解除し、**Save** & **Deploy** を実行する。

---

### 問題 8 (概念 / 実装)
**問題**: BCP 38 がネットワーク全体で広く導入されることで、特にどのようなタイプの不正トラフィックが無効化されるか説明してください。

* **解答**:
  * **反射・増幅型トラフィック（DNS/NTP 反射など）**。送信元 IP アドレスを被害者 IP に偽装して外部サーバーにリクエストを大量送出する動作を境界で防止するためです。BCP 38 が境界で機能していれば、送信元 IP が偽装されたパケットが自組織外へ送出できないため、広域での不正通信を未然に防止できます。


---

## 🔗 参考リソース

Cisco 公式ドキュメント、Design Guide、CVD、Cisco Live スライドの参考一覧です。

* Cisco IOS-XE Network Security Configuration Guide - Unicast RPF
* Cisco ASA Series Firewall CLI Configuration Guide - Anti-Spoofing
* Cisco Secure Firewall Management Center Configuration Guide
* IETF RFC 2827 - Network Ingress Filtering: Defeating Source Address Spoofing (BCP 38)
* IETF RFC 3704 - Ingress Filtering for Multihomed Networks (BCP 84)
* Cisco Live - Infrastructure Protection & BCP 38 Architecture
* Cisco Validated Design - Enterprise Internet Edge Design Guide


---

## 📝 **補足（Notes）**

### uRPF の処理判定フローチャート

```
[ インバウンドパケット到着 ]
             │
             ▼
[ 送信元 IP アドレス (Src IP) の抽出 ]
             │
             ▼
[ モードの確認 ]
   ├──► 【Strict Mode】
   │       FIB テーブル検索: Src IP への Best Path 出力 IF == 到着 IF ?
   │       ├── Yes ──► [ 通過 (Permit) ]
   │       └── No  ──► [ ドロップ (Drop) ]
   │
   └──► 【Loose Mode】
           FIB テーブル検索: Src IP への有効経路 (Valid Route) が存在するか？
           ├── Yes ──► [ 通過 (Permit) ]
           └── No  ──► [ ドロップ (Drop) ]
```

### 要点チェックリスト

* [ ] Strict uRPF は「対称ルーティング」専用である。
* [ ] Loose uRPF は「非対称ルーティング」およびマルチホーム環境で使用する。
* [ ] デフォルトルート依存環境では `allow-default` の扱いに注意する。
* [ ] ASA / FTD の Anti-Spoofing は Strict 動作が基本である。
* [ ] BCP 38 は送信元 IP 偽装の無効化に極めて有効である。



