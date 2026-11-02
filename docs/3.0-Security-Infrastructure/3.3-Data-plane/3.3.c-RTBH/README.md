---
layout: default
title: 3.3.c-RTBH
nav_order: 3
parent: 3.3-Data-plane
grand_parent: 3.0-Security-Infrastructure
---

# 3.3.c RTBH

## 📘 概要

Remotely Triggered Black Hole（RTBH）は、DDoS（Distributed Denial of Service）攻撃や悪意のあるトラフィックをネットワークエッジ（境界ルーター）で迅速かつ効率的に破棄（ドロップ）するためのBGP（Border Gateway Protocol）ベースの防御メカニズムである。通常、攻撃対象のIPアドレス（Destination-based RTBH）または攻撃元のIPアドレス（Source-based RTBH）に対する静的ルーティング情報（Null0インターフェイスへのネクストホップ）を、トリガールーター（Trigger Router）からiBGPまたはeBGPを介して全エッジルーターへ動的に広報・配信することで実現される。

従来のアクセスコントロールリスト（ACL）によるドロップ処理は、大量の攻撃トラフィック（数百Gbps〜数Tbps）が流入した場合、ルーターのTCAM（Ternary Content-Addressable Memory）の枯渇やCPUのオーバーヘッドを引き起こし、ルーター自体のダウンやサービス停止を招くリスクがある。これに対しRTBHは、ルーターのFIB（Forwarding Information Base）/CEF（Cisco Express Forwarding）テーブルにおけるパケット転送ロジックを直接利用して処理するため、ラインレートでのハイスピードなパケット破棄が可能となる。

RTBHは主に以下の場面で利用される。
- **ボットネットやDDoS攻撃からの自組織インフラ保護**: 特定のサービス（WebサーバーやDNSサーバー等）に対する大規模DDoSが発生した際、被害を受けているターゲットIPアドレス宛てのトラフィックをISP境界で破棄し、コアネットワークやファイアウォールの過負荷を防止する。
- **悪意ある攻撃元（C2サーバー、スキャナー）からの通信遮断**: Threat IntelligenceやIDS/IPSが検知した悪性送信元IPアドレス群を、Source-based RTBH（S-RTBH）とuRPF（Unicast Reverse Path Forwarding）Loose Modeの組み合わせによって境界で遮断する。
- **セキュリティ自動運用（SOAR / FMC Security Intelligence）との統合**: 脅威検知システムが自動的にトリガールーターへBGPルートを注入し、オペレーターの手動介入なしに秒単位で攻撃を緩和する。


---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **特徴** | BGPのルート広報機能とNull0インターフェイス（ビットバケツ）を利用し、ネットワークエッジで大量トラフィックを無条件破棄する標準的DDoS緩和技術 |
| **分類** | **Destination-based RTBH**（ターゲットIPを破棄）および **Source-based RTBH**（攻撃元IPをuRPF Loose Modeと連携して破棄） |
| **メリット** | ルーターのハードウェア/CEFラインレートでパケットを破棄可能。ACLのようなTCAM枯渇やCPU負荷が発生しない。BGP Communityを利用して1台のトリガールーターから全エッジへ一括適用可能 |
| **デメリット** | Destination-based RTBHの場合、攻撃対象IP宛の正常トラフィックも全て遮断される（Self-Inflicted DoSの側面）。Source-based RTBHは全エッジでuRPF Loose Modeの設定が必須 |
| **対応機種** | Cisco IOS, IOS-XE, IOS-XR, NX-OS, ASA（BGP対応バージョン）, FTD（FlexConfig / BGP integration） |
| **制限事項** | BGPの収束時間に依存する。ポート番号やL4プロトコル指定の精密なフィルタリングは不可（BGP FlowSpecが必要） |
| **設計上の注意点** | 予約済み未割り当てIP（例: 192.0.2.1/32）を全ルーターで `ip route 192.0.2.1 255.255.255.255 Null0` として静的に定義し、BGP Communityでネクストホップを書き換える構成が推奨される |


---

## 🏗 動作原理

RTBHは「BGPによるルート配布」と「Null0インターフェイスへのパケットルーティング（およびuRPF照合）」を組み合わせることで動作する。主要な2つの方式（Destination-basedおよびSource-based）の基本概念と通信フローは以下の通りである。

### 1. Destination-based RTBH (D-RTBH)
攻撃のターゲットとなっている自組織内の特定のIPアドレス（例: `203.0.113.50/32`）宛ての通信を破棄する方式。

```
[ Attacker / Internet ]
         │ (DDoS Traffic to 203.0.113.50)
         ▼
[ Edge Router 1 / 2 ] ◄── BGP Update (Prefix: 203.0.113.50/32, Next-Hop: 192.0.2.1) ── [ Trigger Router ]
         │
         ├─ Lookup 203.0.113.50 ──► Next-Hop: 192.0.2.1
         ├─ Recursive Lookup 192.0.2.1 ──► Null0
         ▼
   [ Drop Packet (Line Rate) ]
```

1. **静的プレフィックス予約**: 全エッジルーターに未割り当てのダミーIP（例: `192.0.2.1/32`）をNull0へ向ける静的ルートを設定しておく (`ip route 192.0.2.1 255.255.255.255 Null0`)。
2. **トリガー注入**: トリガールーターで攻撃対象IP（`203.0.113.50/32`）に対する静的ルートを作成し、BGPへ再配布（Redistribute）または `network` コマンドで注入する。
3. **BGP Community & Next-Hop 書き換え**: Route-Mapを用いて、このプレフィックスのBGP Next-Hopを `192.0.2.1` に変更し、特定のBGP Community値（例: `65000:666`）を付与してエッジルーターへiBGP広報する。
4. **エッジルーターでの処理**: エッジルーターはiBGP経由で `203.0.113.50/32`（Next-Hop: `192.0.2.1`）を受信する。再帰的ルート検索（Recursive Route Lookup）により、`192.0.2.1` は `Null0` に送られるため、`203.0.113.50` 宛てのパケットはエッジで即座にドロップされる。

### 2. Source-based RTBH (S-RTBH)
攻撃を仕掛けてきている悪性送信元IPアドレス（例: `198.51.100.200/32`）からの通信を破棄する方式。これには **uRPF（Unicast Reverse Path Forwarding）Loose Mode** の併用が不可欠である。

```
[ Malicious Source: 198.51.100.200 ]
         │ (Attack Packet)
         ▼
[ Edge Router (Inbound Interface) ]
         │
         ├─ uRPF Check: Source IP 198.51.100.200 in FIB? ──► YES (Route exists via BGP)
         ├─ Check FIB Entry Interface ──► FIB points to Null0
         ▼
   [ uRPF Drop (Line Rate) ]
```

1. **FIBエントリーの動的生成**: トリガールーターから攻撃元IP（`198.51.100.200/32`）に対するBGPルート（Next-Hop: `192.0.2.1` -> `Null0`）を広報し、エッジルーターのFIBに「`198.51.100.200/32` 宛ては `Null0` へ送る」というエントリーを作成する。
2. **uRPF Loose Modeの動作**: エッジルーターの対外インターフェイスで `ip verify unicast source reachable-via any`（uRPF Loose Mode）を有効化しておく。
3. **パケット検証と破棄**: 送信元IP `198.51.100.200` のパケットがエッジルーターに到着すると、uRPFがFIBを検索する。FIB上にエントリーは存在するが、その出力を担当するインターフェイスが `Null0` であるため、uRPF検証で不合格（Drop）と判定され、パケットは破棄される。


---

## ⚙ 動作シーケンス

パケットがエッジルーターに到着してから破棄されるまでの詳細なシリアル処理手順を以下に示す。

### Destination-based RTBH のパケット処理シーケンス
1. **パケットの受信**: エッジルーターのOutsideインターフェイスに、宛先IP `203.0.113.50` のDDoSパケットが到着する。
2. **レイヤ2/3デカプセル化**: L2フレームヘッダーを剥ぎ取り、IPv4ヘッダーを検証する。
3. **FIB（CEF）ルート検索**: パケットの宛先IP `203.0.113.50` に基づいてCEFテーブルを検索する。
4. **BGPプレフィックスの検索ヒット**: `203.0.113.50/32` のエントリーがヒットし、BGP Next-Hopとして `192.0.2.1` が得られる。
5. **再帰的ルート検索（Recursive Lookup）**: CEFは Next-Hop IP `192.0.2.1` に対するベストパスを再帰的に検索する。
6. **Null0インターフェイスへの紐付け**: `192.0.2.1/32` はローカルに設定された静的ルート `ip route 192.0.2.1 255.255.255.255 Null0` に一致する。
7. **パケット破棄（Discard Action）**: 出力インターフェイスが `Null0` であるため、CEFスイッチングプレーンはパケットを即座に破棄し、ICMP Destination Unreachable などの応答を生成せずにサイレントドロップする。

### Source-based RTBH のパケット処理シーケンス
1. **パケットの受信**: エッジルーターのOutsideインターフェイスに、送信元IP `198.51.100.200`、宛先IP `203.0.113.10`（正常サーバー）のパケットが到着する。
2. **uRPF Loose Mode ルックアップの実行**: インターフェイスに設定された `ip verify unicast source reachable-via any` に基づき、パケットの **送信元IP（`198.51.100.200`）** に対するFIB検索を実行する。
3. **FIBエントリーの検出**: BGPにより広報された `198.51.100.200/32` （Next-Hop: `192.0.2.1` -> `Null0`）がFIB内で見つかる。
4. **出力インターフェイスの不適合判定**: FIB検索で得られた出力インターフェイスが `Null0`（ビットバケツ）であることを検出する。uRPF仕様において、`Null0` を指すルートは「有効な到達可能パス（Reachable Route）」とはみなされない。
5. **uRPF ドロップの確定**: 送信元IPが有効な到達可能ルートを持たないと判定され、uRPFドロップカウンターがインクリメントされ、パケットは入口（Ingress）で破棄される。


---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Securityラボ試験において、RTBHはデータプレーン保護（Data Plane Protection）およびBGP運用セキュリティの重要項目として出題される。

### Blueprintで重要なポイント
- **Destination-based RTBHとSource-based RTBHの設定手順と違い**: 宛先遮断か送信元遮断かによって、uRPF設定の有無やBGP Route-Mapの適用方法が異なる。
- **BGP Community（No-Advertise / Custom Community）の制御**: トリガールーターから広報するBGPルートに `no-advertise` や組織独自のCommunity（例: `65000:666`）を付与し、非意図的な外部ISPへの再広報を防止する設定。
- **再帰的ネクストホップ（Recursive Next-Hop）の設計**: 全ルーターでの静的ルート (`ip route <Blackhole-IP> 255.255.255.255 Null0`) の事前定義と、BGP Route-mapでの `set ip next-hop <Blackhole-IP>` の適用。
- **uRPF Loose Modeの併用**: Source-based RTBHにおいて、エッジルーターの入力インターフェイスで `ip verify unicast source reachable-via any` が正しく設定されているかの確認。

### ラボ試験で設定させられそうな内容
1. **Trigger Routerの構成**:
   - `ip route 192.0.2.1 255.255.255.255 Null0`
   - BGPでの `redistribute static route-map RTBH-INJECT`
   - Route-map内での Community付与 (`set community 65000:666`) および Next-Hop書き換え (`set ip next-hop 192.0.2.1`)
2. **Edge Routerの構成**:
   - BGP受信時の Route-map（Community `65000:666` を含むルートの Next-Hopを `192.0.2.1` に変更、またはローカルNull0へ誘導）
   - インターフェイスでの `ip verify unicast source reachable-via any`（S-RTBH用）
3. **ASA / FTD での BGP RTBH 統合**:
   - ASA/FTD を Edge または Trigger Router として動作させ、特定Communityを受信した際に `Null0` 宛てにルーティングする設定。

### よくある設定ミス・落とし穴
- **Community付与忘れ・再広報事故**: トリガールーターで `no-advertise` または制約用Communityを付与し忘れたため、ブラックホールルートがInternet（eBGPピア）へ流出し、自組織のIP空間全体が世界中でブラックホール化する大惨事（Self-Inflicted Outage）。
- **`send-community` コマンドの欠落**: iBGPピア設定で `neighbor X.X.X.X send-community`（または `send-community standard/both`）を設定し忘れたため、トリガールーターがCommunityを送信せず、エッジルーターでブラックホール処理が行われない。
- **uRPFのモード誤り（S-RTBH）**: Source-based RTBHで、エッジルーターに `uRPF Strict Mode`（`reachable-via rx`）を適用してしまい、非対称ルーティング環境で正常なトラフィックまで大量ドロップさせるミス。S-RTBHには **Loose Mode**（`reachable-via any`）を使用する。
- **再帰検索用のブラックホールIP未定義**: エッジルーターで `ip route 192.0.2.1 255.255.255.255 Null0` を設定し忘れたため、BGP Next-Hop `192.0.2.1` が CEFで解決できず（Inaccessible）、ブラックホールルートがFIBにインストールされない。

### show / debug コマンドでの確認ポイント
- `show ip bgp 203.0.113.50`: BGPテーブルで該当ルートの Next-Hopが `192.0.2.1` に書き換わっており、正しい Community（例: `65000:666`）が付与されているか確認。
- `show ip route 203.0.113.50`: RIBでルートが有効化され、Next-Hopが `Null0` （via `192.0.2.1`）となっているか確認。
- `show ip cef 203.0.113.50`: CEFテーブルで `drop` または `Null0` にバインドされているか確認。
- `show cef interface Null0`: Null0インターフェイスでドロップされたパケット数・バイト数を確認。
- `show ip traffic` または `show ip interface`: uRPFドロップカウンター（`Drop node`）のインクリメントを確認。


## 🛠 設定方法

RTBHの実装例として、トリガールーター（Trigger Router）およびエッジルーター（Edge Router）における Destination-based RTBH および Source-based RTBH の代表的なコンフィグレーションを示す。

### 1. Destination-based RTBH (D-RTBH) コンフィグ設定例

#### 【Trigger Router】
トリガールーターでは、ブラックホール化対象のIPアドレス宛てのルーティング情報を生成し、BGP Community `65000:666` および Next-Hop `192.0.2.1` を付与して広報する。

```text
! 1. ダミーのブラックホールIPへのルート定義 (再帰検索用)
ip route 192.0.2.1 255.255.255.255 Null0

! 2. 攻撃対象サーバーIP（/32）をNull0へ向けるスタティックルート
ip route 203.0.113.50 255.255.255.255 Null0 tag 666

! 3. タグ付きルートを抽出してBGP属性を付与する Route-Map
route-map RTBH-TRIGGER permit 10
 match tag 666
 set ip next-hop 192.0.2.1
 set community 65000:666 no-advertise

! 4. BGP プロセス設定
router bgp 65000
 bgp router-id 1.1.1.1
 neighbor 10.1.1.2 remote-as 65000
 neighbor 10.1.1.2 send-community
 neighbor 10.1.1.3 remote-as 65000
 neighbor 10.1.1.3 send-community
 redistribute static route-map RTBH-TRIGGER
```

#### 【Edge Router】
エッジルーターでは、ダミーIP `192.0.2.1` に対する Null0ルートを定義しておくことで、BGP経由で受信したブラックホールルートを自動的にドロップ処理へ誘導する。

```text
! 1. 再帰的ルックアップ解決用のスタティックルート
ip route 192.0.2.1 255.255.255.255 Null0

! 2. BGP 設定 (Communityの受信と再帰検索)
router bgp 65000
 bgp router-id 10.1.1.2
 neighbor 1.1.1.1 remote-as 65000
 neighbor 1.1.1.1 route-reflector-client
```

---

### 2. Source-based RTBH (S-RTBH) コンフィグ設定例

#### 【Trigger Router】
攻撃元の悪性IPアドレス（例: `198.51.100.200/32`）をBGPで広報する。

```text
! 1. 悪意ある攻撃元IPの定義
ip route 198.51.100.200 255.255.255.255 Null0 tag 999

! 2. Route-Map で Community 付与
route-map SRTBH-TRIGGER permit 10
 match tag 999
 set ip next-hop 192.0.2.1
 set community 65000:999 no-advertise

router bgp 65000
 neighbor 10.1.1.2 send-community
 redistribute static route-map SRTBH-TRIGGER
```

#### 【Edge Router (Outside Interface)】
エッジルーターの外部接続インターフェイスで **uRPF Loose Mode** を有効化する。

```text
ip route 192.0.2.1 255.255.255.255 Null0

interface GigabitEthernet0/0
 description OUTSIDE-INTERNET
 ip address 203.0.113.1 255.255.255.252
 ! uRPF Loose Mode 有効化
 ip verify unicast source reachable-via any
```


---

## 🔍 検証コマンド

| 目的 | コマンド |
| --- | --- |
| BGPテーブルでのブラックホールルート確認 | <code>show ip bgp 203.0.113.50</code> |
| ルーティングテーブルでのNext-Hop確認 | <code>show ip route 203.0.113.50</code> |
| CEFテーブルでのドロップ判定確認 | <code>show ip cef 203.0.113.50</code> |
| Null0インターフェイスのパケットドロップ数確認 | <code>show cef interface Null0</code> |
| uRPFによるドロップ統計確認 | <code>show ip traffic \| include Drop</code> |
| インターフェイス単位のuRPF状態確認 | <code>show ip interface GigabitEthernet0/0 \| include verify</code> |
| BGPピアへ広報中のCommunity確認 | <code>show ip bgp neighbors 10.1.1.2 advertised-routes</code> |
| BGPリアルタイムアップデートデバッグ | <code>debug ip bgp updates</code> |


---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| トリガーで設定したルートがエッジでブラックホール化されない | エッジルーターに `ip route 192.0.2.1 ... Null0` が設定されていないため、Next-HopがInaccessibleになっている | <code>show ip bgp 203.0.113.50</code>（"inaccessible"の表示） | エッジルーターに `ip route 192.0.2.1 255.255.255.255 Null0` を設定する |
| エッジルーターにBGP Communityが届いていない | トリガールーターまたはRRで `neighbor send-community` が設定されていない | <code>show ip bgp 203.0.113.50</code>（Community列が空） | トリガールーターおよびRRのBGPピア設定に `neighbor X.X.X.X send-community` を追加する |
| S-RTBHを設定したが攻撃元からの通信が破棄されない | エッジルーターの対外インターフェイスで uRPF Loose Mode が設定されていない | <code>show ip interface Gi0/0 \| include verify</code> | インターフェイスに `ip verify unicast source reachable-via any` を設定する |
| ブラックホールルートがISPや外部BGPピアへ流出して全通信が停止した | トリガールーターのRoute-Mapで `set community no-advertise` を設定し忘れた | <code>show ip bgp neighbors <ISP-IP> advertised-routes</code> | Route-Mapに `set community no-advertise` または eBGPフィルタACLを追加し、BGPソフトリセットを実行する |
| 特定のサブネット（/24など）全体がブラックホール化して広域障害発生 | トリガールーターで設定したスタティックルートのサブネットマスク指定ミス（例: /32のつもりが/24） | <code>show ip route static</code> | スタティックルートのマスクを `/32` （単一ホスト）に修正し、不要なルートを削除する |


---

## ⚠ 制限事項

- **細粒度制御の不可（L4プロトコル・ポート非対応）**: RTBHはIPプレフィックス（L3）単位でのみドロップを行う。特定のTCP/UDPポート（例: NTP Dynamic Reflection 123/udp のみ遮断等）を狙った攻撃に対し、該当IP宛ての正常なWEB通信（80/448）まで巻き込んで破棄してしまう（D-RTBHの場合）。
- **uRPF Loose Mode依存性（Source-based RTBH）**: S-RTBHは全エッジルーターでuRPF Loose Modeが動作していることが前提であり、1台でもuRPFが未設定のエッジルーターが存在すると、そのルーター経由の攻撃通信は通過してしまう。
- **BGP収束時間による遅延**: トリガールーターでBGPルートを注入してから全エッジルーターへBGP Updateが伝播・収束するまでに数秒〜数十秒のタイムラグが発生する。
- **Self-Inflicted DoSのリスク**: 設定ミスや自動化スクリプトの不具合により誤ったプレフィックス（例: デフォルトルートや社内主要DNS）を広報した場合、ネットワーク全体が即座に不通となる。


---

## 🔄 他技術との関連

- **uRPF (Unicast Reverse Path Forwarding)**: Source-based RTBHを機能させるための必須コンポーネント。パケットの送信元IPをFIB（Null0）と照合してドロップする。
- **BGP (Border Gateway Protocol)**: トリガールーターから全エッジルーターへブラックホールルートおよびBGP Community属性を配布するためのシグナリングプロトコル。
- **BGP FlowSpec (RFC 5575)**: RTBHの発展形技術。L3 IPアドレスだけでなく、L4プロトコル、TCPポート、パケット長、DSCP値などをBGPで拡張広報し、エッジで精密なポリシングやドロップを行う。
- **CoPP (Control Plane Policing)**: RTBHがデータプレーンでの大量トラフィック処理を保護するのに対し、CoPPはコントロールプレーン（CPU）への直接攻撃を保護する。
- **NetFlow / IPFIX / Stealthwatch**: ネットワーク上の異常トラフィックやDDoS攻撃をリアルタイム検知し、トリガールーターへ自動的にRTBH注入コマンドを発行するソース情報源。


---

## 🧩 比較表

| 比較項目 | Destination-based RTBH (D-RTBH) | Source-based RTBH (S-RTBH) | BGP FlowSpec |
| --- | --- | --- | --- |
| **遮断基準** | 攻撃のターゲットIP（Destination） | 攻撃元の悪意あるIP（Source） | L3/L4（IP、Port、Protocol、ICMP Type等） |
| **uRPFの必要性** | **不要**（通常のFIBルックアップで動作） | **必須**（Loose Mode `reachable-via any`） | 不要（BGP拡張NLRIによるルール適用） |
| **正常通信への影響** | 攻撃対象IP宛の正常通信も**全て停止**（正規ユーザー遮断） | 攻撃元IPからの通信のみ遮断（正常宛先への影響なし） | 攻撃トラフィックのみピンポイント遮断 |
| **ルーター負荷** | 最低（CEF/FIBラインレート処理） | 極小（uRPFルックアップオーバーヘッドのみ） | 低〜中（ハードウェアTCAM/ACLエントリー使用） |
| **設定の複雑さ** | 低（静的ルート + BGP Community） | 中（エッジでのuRPF要設定） | 高（BGP FlowSpec対応ルーターと複雑なポリシー） |


---

## 💡 ベストプラクティス

1. **予約済みブラックホールIPの標準化**: 全ルーターにおいて `192.0.2.1/32`（RFC 5737 TEST-NET-1 などの予約空間）を `Null0` へ向ける静的ルートをテンプレート化して常時設定しておく。
2. **`no-advertise` コミュニティの標準付与**: トリガールーターの Route-Map で必ず `set community no-advertise` を適用し、外部eBGPピアへの事故流出を物理的に遮断する。
3. **Dedicated Trigger Routerの設置**: 運用環境の一般ルーティングを行うルーターとは分離された、専用のトリガールーター（または運用端末・自動化スクリプト）を用意し、誤設定によるルーティングループや障害を防止する。
4. **uRPF Loose Mode の全対外インターフェイス適用**: S-RTBHを活用するため、全エッジルーターの対外（ISP/Transit）インターフェイスで `ip verify unicast source reachable-via any` を定常有効化しておく。
5. **定期的な自動解除（Timeout）の仕組み**: 一時的なDDoS対策としてRTBHを発動した後、永久にブラックホール化が残らないよう、BGPルートの自動取り下げ（Withdrawal）タイマーやEEMスクリプトを導入する。


## 📝 ラボ学習・設定サンプル例

CCIE Securityラボ試験レベルの実践的なRTBH設定・検証演習サンプルを10個掲載する。

---

### サンプル1: Basic Destination-based RTBH with Custom BGP Community

#### 問題
企業ネットワークのWEBサーバー `203.0.113.100/32` が大規模DDoS攻撃を受けている。トリガールーター（R1）からiBGPを用いてエッジルーター（R2）へブラックホールルートを伝播させ、R2のエッジ（Line Rate）で攻撃トラフィックを即座に破棄せよ。

#### 要件
- R2においてダミーネクストホップ `192.0.2.1/32` を `Null0` へ向ける静的ルートを設定すること。
- R1において、タグ `666` が付与されたスタティックルートをBGPへ再配布すること。
- BGP Community `65000:666` および Next-Hop `192.0.2.1` を付与し、外部へ再広報されないよう `no-advertise` を設定すること。

#### 設定例

**【R2 (Edge Router)】**
```text
ip route 192.0.2.1 255.255.255.255 Null0

router bgp 65000
 bgp router-id 2.2.2.2
 neighbor 10.1.1.1 remote-as 65000
```

**【R1 (Trigger Router)】**
```text
ip route 192.0.2.1 255.255.255.255 Null0
ip route 203.0.113.100 255.255.255.255 Null0 tag 666

route-map RTBH-POLICY permit 10
 match tag 666
 set ip next-hop 192.0.2.1
 set community 65000:666 no-advertise

router bgp 65000
 bgp router-id 1.1.1.1
 neighbor 10.1.1.2 remote-as 65000
 neighbor 10.1.1.2 send-community
 redistribute static route-map RTBH-POLICY
```

#### 検証方法
R2で `show ip bgp 203.0.113.100` を実行し、Next-Hopが `192.0.2.1` に変更され、Community `65000:666` と `no-advertise` が付与されていることを確認する。`show ip cef 203.0.113.100` でドロップ判定（Null0）となっていることを確認する。

---

### サンプル2: Source-based RTBH with uRPF Loose Mode Integration

#### 問題
悪意あるC2サーバー `198.51.100.50/32` からの攻撃通信を、エッジルーター（R2）の対外インターフェイス入力時に破棄（S-RTBH）せよ。

#### 要件
- R2の対外インターフェイス GigabitEthernet0/0 に uRPF Loose Mode を設定すること。
- R1（Trigger Router）から `198.51.100.50/32` に対するブラックホールルートを広報すること。

#### 設定例

**【R2 (Edge Router)】**
```text
ip route 192.0.2.1 255.255.255.255 Null0

interface GigabitEthernet0/0
 description UNTRUSTED-INTERNET
 ip address 203.0.113.1 255.255.255.252
 ip verify unicast source reachable-via any

router bgp 65000
 neighbor 10.1.1.1 remote-as 65000
```

**【R1 (Trigger Router)】**
```text
ip route 198.51.100.50 255.255.255.255 Null0 tag 999

route-map SRTBH-POLICY permit 10
 match tag 999
 set ip next-hop 192.0.2.1
 set community 65000:999 no-advertise

router bgp 65000
 neighbor 10.1.1.2 remote-as 65000
 neighbor 10.1.1.2 send-community
 redistribute static route-map SRTBH-POLICY
```

#### 検証方法
R2のGigabitEthernet0/0に送信元 `198.51.100.50` からのパケットを流し、`show ip traffic | include Drop` および `show ip interface GigabitEthernet0/0` でuRPFドロップカウンターがカウントアップされることを確認する。

---

### サンプル3: BGP Route Reflector Environment with RTBH Community Propagation

#### 問題
大規模ネットワークにおいて、ルートリフレクタ（RR: R2）を介してトリガールーター（R1）から複数のエッジルーター（R3, R4）へRTBHルートを同期させよ。

#### 要件
- R2（RR）はクライアント（R1, R3, R4）へBGP Community属性を確実に透過・属性保持して転送すること。
- R3, R4 は受信したCommunityに基づき、Next-HopをローカルのブラックホールIPに正しく解決すること。

#### 設定例

**【R2 (Route Reflector)】**
```text
router bgp 65000
 bgp router-id 2.2.2.2
 neighbor 10.1.1.1 remote-as 65000
 neighbor 10.1.1.1 route-reflector-client
 neighbor 10.1.1.1 send-community both
 neighbor 10.1.1.3 remote-as 65000
 neighbor 10.1.1.3 route-reflector-client
 neighbor 10.1.1.3 send-community both
 neighbor 10.1.1.4 remote-as 65000
 neighbor 10.1.1.4 route-reflector-client
 neighbor 10.1.1.4 send-community both
```

**【R3 / R4 (Edge Routers)】**
```text
ip route 192.0.2.1 255.255.255.255 Null0

router bgp 65000
 neighbor 10.1.1.2 remote-as 65000
```

#### 検証方法
R3およびR4で `show ip bgp` を実行し、R1から注入されたRTBHルートがNext-Hop `192.0.2.1` および Community `65000:666` を維持したまま正しく反映されているか確認する。

---

### サンプル4: Preventing BGP Blackhole Leakage using `no-advertise` and Filtering

#### 問題
BGPブラックホールルートが誤って上位ISP（eBGP Peer）へ再広報され、自組織の通信が全滅する事故を二重のセーフティで防止せよ。

#### 要件
- BGP Community `no-advertise` の付与に加え、eBGPピア用の出力量アウトバウンド Route-Map でブラックホールCommunity（`65000:666`）を含むルートを明示的にフィルタ（Deny）すること。

#### 設定例

**【R2 (Edge Router connected to ISP)】**
```text
ip ip-prefix PL-BLACKHOLE seq 5 permit 192.0.2.1/32

ip community-list standard CL-RTBH permit 65000:666

route-map OUT-TO-ISP deny 10
 match community CL-RTBH
!
route-map OUT-TO-ISP deny 20
 match ip address prefix-list PL-BLACKHOLE
!
route-map OUT-TO-ISP permit 100

router bgp 65000
 neighbor 203.0.113.254 remote-as 64512
 neighbor 203.0.113.254 route-map OUT-TO-ISP out
```

#### 検証方法
`show ip bgp neighbors 203.0.113.254 advertised-routes` を実行し、ブラックホールルート（`203.0.113.50/32` 等）がISP向け広報リストに含まれていないことを確認する。

---

### サンプル5: eBGP-based RTBH Triggering to Upstream Service Provider

#### 問題
自組織のエッジルーター（R1）から上位ISP（AS 64512）へeBGP経由で特定Communityを送信し、ISP側のルーターで対象IP（`203.0.113.88/32`）をブラックホール化させよ。

#### 要件
- ISP側で規定されたRTBH用Community `64512:666` を付与して広報すること。
- 広報するプレフィックスは `/32` に限定すること。

#### 設定例

**【R1 (Customer Edge Router)】**
```text
ip route 203.0.113.88 255.255.255.255 Null0 tag 777

ip ip-prefix HOST-ONLY seq 5 permit 203.0.113.88/32

route-map RTBH-TO-ISP permit 10
 match ip address prefix-list HOST-ONLY
 match tag 777
 set community 64512:666

router bgp 65000
 neighbor 198.51.100.1 remote-as 64512
 neighbor 198.51.100.1 send-community
 neighbor 198.51.100.1 route-map RTBH-TO-ISP out
```

#### 検証方法
R1で `show ip bgp neighbors 198.51.100.1 advertised-routes` を実行し、`203.0.113.88/32` がCommunity `64512:666` を持って上位ISPへ送信されていることを確認する。

---

### サンプル6: Tag-based Static Route Redistribution for Automated RTBH Ingress

#### 問題
運用担当者が単に `ip route <IP> 255.255.255.255 Null0 tag 666` を投入するだけで自動的にiBGPへRTBHルートが広報される環境を構築せよ。

#### 要件
- BGPの `redistribute static` と Route-Map を連携させ、タグ `666` を持つ静的ルートのみを抽出し、Next-Hopの書き換えとCommunity付与を自動実行すること。

#### 設定例

**【R1 (Trigger Router)】**
```text
route-map AUTO-RTBH permit 10
 match tag 666
 set ip next-hop 192.0.2.1
 set community 65000:666 no-advertise
route-map AUTO-RTBH deny 99

router bgp 65000
 bgp log-neighbor-changes
 redistribute static route-map AUTO-RTBH
 neighbor 10.1.1.2 send-community
```

#### 検証方法
R1でテストコマンド `ip route 10.99.99.99 255.255.255.255 Null0 tag 666` を追加し、`show ip bgp 10.99.99.99` で即座にNext-Hop `192.0.2.1` および Community `65000:666` が反映されることを確認する。

---

### サンプル7: ASA Firewall Integration with BGP Destination-based RTBH

#### 問題
Cisco ASA ファイアウォールをBGPスピーカとして動作させ、iBGPでRTBHルートを受信した際に該当IP宛のトラフィックをNull0へドロップさせよ。

#### 要件
- ASAに `ip route outside 192.0.2.1 255.255.255.255 Null0` を設定すること。
- iBGPで受信したRTBHルートのネクストホップが `192.0.2.1` へ向かい、適切にブラックホール化されること。

#### 設定例

**【Cisco ASA】**
```text
! 1. Null0 ルートの定義
route outside 192.0.2.1 255.255.255.255 Null0

! 2. BGP 設定
router bgp 65000
 bgp router-id 10.1.1.10
 neighbor 10.1.1.1 remote-as 65000
 neighbor 10.1.1.1 activate
```

#### 検証方法
ASAのCLIで `show route` および `show bgp` を実行し、広報された攻撃対象IPのネクストホップが `192.0.2.1` （Null0）としてASAのルーティングテーブルに登録されていることを確認する。

---

### サンプル8: Cisco FTD FlexConfig & BGP S-RTBH Implementation

#### 問題
FMCで管理される Cisco FTD において、BGPで Source-based RTBH ルートを受信し、かつ対外インターフェイスで uRPF Loose Mode を有効化せよ。

#### 要件
- FMCの Interface Configuration にて対外インターフェイスの uRPF（Loose Mode）を有効化すること。
- BGP設定および再帰的ネクストホップ `192.0.2.1 Null0` を設定すること。

#### 設定例

**【FMC GUI / FlexConfig / FTD】**
1. **Interface Settings (FMC)**:
   - Edit Outside Interface -> Advanced -> Enable **uRPF** (Mode: **Loose** / `reachable-via any`).
2. **Static Route (FMC)**:
   - Interface: `Null0`, Network: `192.0.2.1/32`, Gateway: `Null0`.
3. **BGP Configuration (FMC)**:
   - IPv4 BGP Neighbor to Trigger Router (`10.1.1.1`).

#### 検証方法
FTDのLINA CLI (`system support diagnostic-cli`) で `show route` および `show asp drop` を実行し、該当送信元IPからのパケットが `rpf-violation` または Null0 転送で破棄されていることを確認する。

---

### サンプル9: Dual-Router Edge Redundancy with RTBH Synchronization

#### 問題
2台のエッジルーター（R2, R3）が存在する冗長化構成において、トリガールーター（R1）からのRTBH要求が両エッジに均等かつ同期して適用される構成を構築せよ。

#### 要件
- R1から R2, R3 へ対等にiBGPセッションを張り、`send-community` を設定すること。
- 両エッジルーターに共通のブラックホールIP `192.0.2.1/32 Null0` を定義すること。

#### 設定例

**【R1 (Trigger Router)】**
```text
router bgp 65000
 neighbor 10.1.1.2 remote-as 65000
 neighbor 10.1.1.2 send-community
 neighbor 10.1.1.3 remote-as 65000
 neighbor 10.1.1.3 send-community
 redistribute static route-map AUTO-RTBH
```

**【R2 / R3 (Edge Routers)】**
```text
ip route 192.0.2.1 255.255.255.255 Null0

router bgp 65000
 neighbor 10.1.1.1 remote-as 65000
```

#### 検証方法
R1で攻撃対象IPを追加後、R2とR3の両方で `show ip cef <IP>` を実行し、同時に両ルーターで `drop` / `Null0` に更新されることを確認する。

---

### サンプル10: EEM (Embedded Event Manager) Automated RTBH Trigger and Auto-Withdrawal

#### 問題
Syslogで特定DDoSアラート（例: `DDOS_ATTACK_DETECTED`）が検知された際、EEMスクリプトを用いて動的にRTBHスタティックルートを追加し、300秒（5分）後に自動的に取り下げる自動防御設定を構築せよ。

#### 要件
- SyslogメッセージのIPアドレスを抽出し、`ip route <IP> 255.255.255.255 Null0 tag 666` を実行すること。
- 300秒後に当該ルートを削除（`no ip route ...`）すること。

#### 設定例

**【R1 (Trigger Router)】**
```text
event manager applet AUTO-RTBH-TRIGGER
 event syslog pattern "DDOS_ATTACK_DETECTED IP=(.*)"
 action 1.0 regexp "IP=([0-9\.]+)" "$_syslog_msg" match target_ip
 action 2.0 cli command "enable"
 action 3.0 cli command "config t"
 action 4.0 cli command "ip route $target_ip 255.255.255.255 Null0 tag 666"
 action 5.0 syslog msg "RTBH Triggered automatically for $target_ip"
 action 6.0 cli command "workaround-wait 300"
 action 7.0 cli command "no ip route $target_ip 255.255.255.255 Null0 tag 666"
 action 8.0 syslog msg "RTBH Withdrawn automatically for $target_ip"
```

#### 検証方法
テスト用Syslogを発行（`send log DDOS_ATTACK_DETECTED IP=203.0.113.199`）し、`show ip bgp 203.0.113.199` で即座にブラックホール化され、5分後に自動削除されることを確認する。


## ❓ 想定試験問題

CCIE Securityラボ試験・筆記試験を意識した実践的な想定問題と詳細解説を6問掲載する。

---

### 問題 1（コンフィグ読解・トラブルシュート）

**【問題】**
エンジニアがトリガールーター（R1）に以下のRTBH設定を投入した。しかし、エッジルーター（R2）のBGPテーブルにルートは届いているものの、パケットがドロップされず通常通り転送されてしまう。原因として最も適切なものを1つ選べ。

**R1（Trigger Router）のコンフィグ:**
```text
ip route 192.0.2.1 255.255.255.255 Null0
ip route 203.0.113.10 255.255.255.255 Null0 tag 666

route-map RTBH-MAP permit 10
 match tag 666
 set ip next-hop 192.0.2.1
 set community 65000:666 no-advertise

router bgp 65000
 neighbor 10.1.1.2 remote-as 65000
 redistribute static route-map RTBH-MAP
```

A) R1で `neighbor 10.1.1.2 send-community` が設定されていないため、R2でCommunityが受信できずNext-Hop書き換えが行われていない。
B) R2で `ip route 192.0.2.1 255.255.255.255 Null0` の静的ルートが設定されていないため、再帰的ネクストホップ解決に失敗してルートがFIBに反映されていない。
C) R1で `bgp scan-time` が長すぎるため、BGPテーブルが更新されていない。
D) R2で `ip verify unicast source reachable-via any` が設定されていない。

**【正解】**
**B**

**【解説】**
RTBHの動作原理において、トリガールーター（R1）はBGP Next-Hopとして `192.0.2.1` を広報する。エッジルーター（R2）側では、この `192.0.2.1` を `Null0` インターフェイスへ向ける静的ルート（`ip route 192.0.2.1 255.255.255.255 Null0`）があらかじめローカルに設定されていなければならない。これが未設定の場合、R2はBGP Next-Hop `192.0.2.1` を到達不能（Inaccessible）と判断し、該当のBGPルート（`203.0.113.10/32`）をルーティングテーブル（RIB）およびFIB（CEF）にインストールしないため、パケットは通常ルーティングされてしまう。

---

### 問題 2（Design / uRPF統合）

**【問題】**
特定IPアドレス（Source IP）を指定してネットワーク境界で不要なパケットを遮断する **Source-based RTBH（S-RTBH）** を導入する際、エッジルーターの対外インターフェイスに必要な必須設定はどれか。

A) インターフェイスへの `ip verify unicast source reachable-via rx`（Strict Mode）の適用
B) インターフェイスへの `ip verify unicast source reachable-via any`（Loose Mode）の適用
C) BGPでの `bgp dampening` の有効化
D) インターフェイスへの `qos pre-classify` の適用

**【正解】**
**B**

**【解説】**
Source-based RTBH（S-RTBH）では、BGPによって「送信元IP宛てのルート」のネクストホップを `Null0` に指定したFIBエントリーを作成する。パケットが対外インターフェイスに入ってきた際、その送信元IPをFIBと照合し、FIBエントリーの出力先が `Null0` である場合にパケットを遮断する仕組みは **uRPF Loose Mode** (`reachable-via any`) によって提供される。

---

### 問題 3（トラブルシュート・BGP Community）

**【問題】**
トリガールーターで特定IP宛の遮断のためにRTBHルートを注入した直後、自組織の全公開IPアドレス宛ての通信がインターネット経由で一切届かなくなった。トラブルシューティングを行ったところ、上位ISP側で自組織のIPプレフィックス全体がブラックホール化されていた。この事象の直接的な原因として最も可能性が高いものはどれか。

A) エッジルーターで `ip route 192.0.2.1 255.255.255.255 Null0` の設定を失念した。
B) トリガールーターでBGP Route-Map設定時、`set community no-advertise` または eBGPフィルタリングを怠り、ブラックホールルートがeBGP経由で外部ISPへ流出した。
C) エッジルーターで `uRPF Strict Mode` が全インターフェイスで動作していた。
D) BGPピア間で MD5 認証キーが不一致であった。

**【正解】**
**B**

**【解説】**
トリガールーターから広報するルートに `no-advertise` コミュニティが付与されていないか、エッジルーターのアウトバウンドRoute-Mapで外部eBGPピア向けのフィルタリングが設定されていない場合、ブラックホールルートが上位ISPへ再広報されてしまう。上位ISPがこのルートを受け入れると、インターネット上の全トラフィックがISP側でブラックホール（Null0）へ引き込まれる障害が発生する。

---

### 問題 4（コンフィグ読解・マルチホップBGP）

**【問題】**
以下のコンフィグを持つトリガールーター（R1）に関する記述として正しいものを1つ選べ。

```text
ip route 192.0.2.1 255.255.255.255 Null0
ip route 10.50.50.50 255.255.255.255 Null0 tag 666

route-map SET-RTBH permit 10
 match tag 666
 set ip next-hop 192.0.2.1
 set community 65000:666 no-advertise local-AS

router bgp 65000
 bgp router-id 1.1.1.1
 neighbor 10.2.2.2 remote-as 65000
 neighbor 10.2.2.2 send-community
 redistribute static route-map SET-RTBH
```

A) `10.50.50.50/32` のルートはiBGPピア `10.2.2.2` へ送信されるが、`10.2.2.2` はそれを他のどのBGPピアへも再広報しない。
B) `10.50.50.50/32` はBGPテーブルに登録されず、iBGPピアへ広報されない。
C) `10.2.2.2` はこのルートを受信した際、ネクストホップを自身のIPアドレスに自動書き換えする。
D) `set community` 内の `local-AS` アトリビュートにより、同じiBGPアソシエーション内であってもルートが破棄される。

**【正解】**
**A**

**【解説】**
`no-advertise` Well-known Communityが与えられたBGPルートは、そのルートを受信したBGPルーター（10.2.2.2）によって他の如何なるBGPピア（iBGPおよびeBGP）へも一切広報されない。

---

### 問題 5（実装・ASA/FTDの機能比較）

**【問題】**
Cisco ASA および Cisco FTD における Destination-based RTBH の実装に関する記述として **誤っているもの** を1つ選べ。

A) ASAはBGPをサポートしており、BGPで受信したブラックホールルートのネクストホップを Null0 宛てのスタティックルートに解決させてルーティングテーブルに保持できる。
B) FTDはFMCまたはFDMを介してBGPを設定可能であり、LINAエンジンレベルでパケットのNull0ドロップを実行できる。
C) ASA/FTDでは `Null0` インターフェイス宛てのルートを作成することが一切サポートされていないため、RTBHの受信側（Edge）として動作させることはできない。
D) FTDで Source-based RTBH を利用する場合、FMCのインターフェイス設定で uRPF Loose Mode を有効化する必要がある。

**【正解】**
**C**

**【解説】**
Cisco ASA および FTD（LINA）は、`route outside 192.0.2.1 255.255.255.255 Null0`（またはFMCのStatic Route設定）により Null0 インターフェイスをサポートしている。BGPで学習したブラックホールルートを Null0 に導くことで、ASA/FTD 自体を RTBH のエッジドロップポイントとして動作させることが可能である。

---

### 問題 6（トラブルシュート・検証コマンド）

**【問題】**
Source-based RTBH（S-RTBH）を運用中のルーターにおいて、特定送信元IPからのパケットが uRPF によってドロップされているかをCLIで確認するためのコマンドおよび表示項目の組み合わせとして正しいものはどれか。

A) `show ip bgp` を実行し、ステータスが `Valid, Internal` になっていることを確認する。
B) `show ip traffic` を実行し、`unicast drop` / `uRPF drop` カウンターがインクリメントされていることを確認する。
C) `show ip route` を実行し、該当IPが `0.0.0.0` へルーティングされていることを確認する。
D) `show running-config` を実行し、`ip verify unicast source` が記載されていることを確認する。

**【正解】**
**B**

**【解説】**
uRPFによるパケットドロップ数をグローバル統計として確認するには `show ip traffic` を使用する。出力内の `Drop` セクションに存在する `unicast RPF` または `source lookup failed` のカウンターの増加を確認することで検証できる。

---

## 🔗 参考リソース

Cisco公式ドキュメントおよび関連RFC一覧。

- Cisco Configuration Guide: BGP Remotely Triggered Black Hole (RTBH) Mitigation
- Cisco Technical Notes: BGP Community Attributes and Blackhole Routing
- Cisco Validated Design (CVD): Enterprise Infrastructure Edge Protection
- RFC 3882: Configuring BGP to Block Denial-of-Service Attacks
- RFC 5635: Remote Triggered Black Hole Filtering with Unicast Reverse Path Forwarding
- Cisco Command Reference: BGP `send-community` Command


---

## 📝 補足（Notes）

### 1. BGP FlowSpec (RFC 5575) への発展
RTBHは「IPアドレス単位で全通信をドロップする」という効果を持つ一方、正常トラフィックの巻き添え遮断リスクがある。現代の高度な保護対策では、RTBHの拡張規格である **BGP FlowSpec** が利用される。BGP FlowSpecでは、以下のような多次元ルールをBGP NLRIで配信可能である。
- **Match条件**: 送信元/宛先IP、Protocol（UDP/TCP/ICMP）、L4 Port（Dst Port 123等）、ICMP Code/Type、Packet Length、DSCP、TCP Flags
- **Action**: Drop、Rate Limit（ポリシング）、Redirect to VRF（クレンジングセンターへの引込み）、Mark DSCP

### 2. RTBH運用時のセキュリティ設計リスト
1. **Trigger Routerの保護**: Trigger Routerへのアクセス権限を徹底管理する。CoPP、iACL、SSHv2、TACACS+で制御すること。
2. **Community体系の明確な命名規則**:
   - `65000:666` -> Local Discard (Destination-based)
   - `65000:999` -> Local Discard via uRPF (Source-based)
   - `64512:666` -> Upstream ISP Discard
3. **モニタリングとログ採取**: RTBH発動時はSyslogおよびSNMP TrapをNMSへ飛ばし、運用担当者に即座に通知する環境を整えること。
