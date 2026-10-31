---
layout: default
title: 1.7.a-DoS-DDoS
nav_order: 1
parent: 1.7-Attack-detection
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.7.a DoS/DDoS

Cisco Secure Firewall Threat Defense (FTD)、Management Center (FMC)、Cisco ASA、Cisco IOS/IOS-XE ルーター、および Catalyst スイッチにおける **DoS/DDoS（Denial of Service / Distributed Denial of Service：サービス拒否・分散サービス拒否攻撃）対策** は、不審な大容量トラフィックやセッション枯渇攻撃からネットワークインフラ（Control Plane）および保護対象サーバー（Data Plane）の可用性を維持するための包括的な多層防御アーキテクチャです。

CCIE Security v6.1 ラボ試験において、DoS/DDoS 対策は単一のコマンド設定にとどまらず、**TCP SYN Flood に対する TCP Intercept（IOS）および ASA/FTD MPF（SYN Cookie / Embryonic Connection Limit）、ASA Threat Detection による自動 Shun、BGP RTBH（Remotely Triggered Black Hole：Destination-based / Source-based）、CoPP/CPPr による CPU/Control Plane 保護、FTD Rate Limiting、ならびに Cisco Umbrella（DNS Layer Security）による C2/Botnet 軽減** に至るまで、レイヤー 2 から 7 に渡る実践的な設計・設定・トラブルシューティング能力が厳格に問われます。

---

## 📘 概要

* **機能概要**:
  * **TCP SYN Flood / State Exhaustion 緩和**: 攻撃者が送信元 IP を偽造して大量の TCP SYN パケットを送信し、サーバーのハーフオープン接続キュー（Backlog Queue）を枯渇させる攻撃に対し、ASA/FTD の SYN Cookie メカニズム（Embryonic Connection Limit）や IOS の TCP Intercept で保護する。
  * **Volume-Based / Flood 攻撃緩和**: 大容量の ICMP/UDP/IP フラッド攻撃に対し、BGP RTBH（Destination-based / Source-based uRPF 連携）や BGP Flowspec を用いてネットワーク境界（ISP/Edge）で無効化する。
  * **Reconnaissance / Scanning 検出 ＆ 自動遮断**: 攻撃前兆である Ping Sweep や TCP/UDP ポートスキャンを ASA Threat Detection や FTD Rate-Based Attack Prevention で識別し、攻撃元 IP を動的 ACL (Shun) で自動ブロックする。
  * **Control Plane / CPU 保護**: ネットワーク機器自体の Route Processor (CPU) を標的とした DoS 攻撃に対し、CoPP (Control Plane Policing)、CPPr (Control Plane Protection)、MPP (Management Plane Protection) を適用して管理・制御プロトコル通信を死守する。
* **利用目的**:
  * ネットワーク機器およびアプリケーションサーバーのシステムリソース（CPU、メモリ、接続テーブル、帯域幅）を枯渇から守り、正規ユーザーに対するサービス継続性（Availability）を確保する。
  * インターネット境界、DMZ 境界、および内部 LAN 境界の各ポイントで多層的に攻撃トラフィックをドロップ・制限する。
* **どのような場面で利用するか**:
  * 外部インターネットから DMZ 内の公開 Web/Mail サーバー群に対する TCP SYN Flood や UDP Flood の防御。
  * 大規模 DDoS 攻撃発生時における ISP / 企業エッジルーターでの BGP RTBH トリガーによる攻撃トラフィックのブラックホール化。
  * ルーター/ファイアウォール自体宛てのパケットスパイクに対する Control Plane 保護。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要防御メカニズム** | **TCP Intercept / SYN Proxy** (IOS: Intercept/Watch, ASA/FTD: SYN Cookie)<br>**Threat Detection** (Basic, Scanning, TCP Intercept Statistics)<br>**BGP RTBH** (Destination-based, Source-based Loose uRPF)<br>**CoPP / CPPr** (Control Plane Rate-Limiting)<br>**FTD Rate Limiting / Prefilter / Security Intelligence** |
| **TCP SYN Flood 閾値** | **Embryonic Connections (`embryonic-conn-max`)**: ハーフオープン接続上限値<br>**Per-Client Embryonic Limit (`per-client-embryonic-max`)**: 1 クライアントあたりのハーフオープン上限 |
| **ASA SYN Cookie 特徴** | 接続テーブルにステートを保持せず、計算した Cookie を Initial Sequence Number (ISN) に埋め込んで応答。ACK 受信時に検証するため ASA のメモリを消費しない |
| **RTBH 方式区分** | **Destination-based RTBH**: 攻撃対象 IP (Victim) 宛てトラフィックを Null0 へ破棄<br>**Source-based RTBH**: 攻撃元 IP (Attacker) からのトラフィックを Loose uRPF 連携で Null0 へ破棄 |
| **CoPP / CPPr 対象** | **CoPP**: Control Plane 宛て全体の集約 Rate Limit<br>**CPPr**: Host, Transit, CEF-Exception の 3 サブインターフェイス分離制御 ＋ Port Filtering |
| **ライセンス要件** | 基本機能は Base ライセンスで利用可能。FTD での Security Intelligence や Threat 連携には Threat ライscription が必要 |
| **制限事項** | BGP Destination RTBH は被害者 (Victim) への正常通信も遮断される (Collateral Damage)。Strict uRPF は非対称ルーティング環境で使用不可 |

---

## 🏗 動作原理

### 1. ASA/FTD TCP Intercept (SYN Cookie) 動作フロー

ASA/FTD の MPF で `embryonic-conn-max` 閾値が設定されている場合、ハーフオープン接続数が閾値を超えると **SYN Cookie モード** が自動起動します。

```text
[ Attacker (Spoofed IPs) ] ───(1) TCP SYN ───► [ ASA / FTD ]                   [ Web Server ]
                                                  │ (Threshold Exceeded: embryonic-conn-max)
                                                  │ Generate SYN-ACK with Sequence Number = SYN Cookie
                                                  │ (ASA creates NO connection entry in memory)
[ Attacker (Spoofed IPs) ] ◄──(2) TCP SYN-ACK ────┤
                                                  │
(Spoofed IPs never reply with ACK)               │ (No memory consumed on ASA or Server)
                                                  │
[ Legitimate Client ]     ───(3) TCP SYN ───► [ ASA / FTD ]
                          ◄──(4) TCP SYN-ACK ────┤ (ISN = SYN Cookie)
                          ───(5) TCP ACK ────►    │
                                                  │ (Validate Cookie Algorithm)
                                                  │ Valid! Establish Backend Session ──► [ Web Server ]
```

### 2. Remotely Triggered Black Hole (RTBH) 動作フロー

#### Destination-Based RTBH vs Source-Based RTBH

```text
1. Destination-Based RTBH:
   [ Trigger Router ] ──BGP Route Update (Victim IP: 192.168.1.100/32, Next-Hop: 192.0.2.1/Null0) ──► [ Edge Routers ]
   [ Edge Routers ]   ──Static Route: 192.0.2.1/32 -> Null0
   [ Attack Traffic to 192.168.1.100 ] ──► [ Edge Router ] ──► Dropped at Null0 Interface

2. Source-Based RTBH (with Loose uRPF):
   [ Trigger Router ] ──BGP Route Update (Attacker IP: 203.0.113.50/32, Next-Hop: 192.0.2.1/Null0) ──► [ Edge Routers ]
   [ Edge Routers ]   ──Loose uRPF Enabled on Ingress Interface (`ip verify unicast source reachable-via any`)
   [ Attack Traffic from 203.0.113.50 ] ──► Ingress Check: Route to 203.0.113.50 points to Null0 ──► Dropped by uRPF!
```

---

## ⚙ 動作シーケンス

1. **Reconnaissance ＆ Scanning Phase（探査フェーズ）**:
   * 攻撃者が Target IP サブネットやポート群に対し、高速な SYN Scan や Ping Sweep を実行。
   * ASA Threat Detection (`scanning-threat`) または FTD Portscan Preprocessor が短時間でのドロップ率・未確立接続率のスパイクを識別。
   * 閾値（Burst/Average Rate）超過により、ASA は syslog 733100/733101 を発報し、設定に応じて攻撃元 IP を `shun` テーブルへ自動追加（一定時間パケットを L2/L3 即時破棄）。
2. **TCP SYN Flood / State Exhaustion Phase（セッション枯渇フェーズ）**:
   * 攻撃者が大量のスプーフィング IP から Web サーバー（TCP 80/443）宛てに SYN パケットを濫打。
   * ASA/FTD の MPF クラスにヒットし、該当クラスの `embryonic-conn-max`（例: 100）を超過。
   * **TCP Intercept (SYN Cookie) 発動**: ASA/FTD はサーバーに SYN を転送せず、自身がプロキシとして TCP/IP ヘッダー情報とシークレットキーから計算した **SYN Cookie を ISN (Initial Sequence Number) に埋め込んだ SYN-ACK** を返答。
   * 偽造 IP からは ACK が返らないため、ASA/FTD およびバックエンドサーバーのメモリテーブルは一切消費されない。
   * 正常クライアントから正しい ACK（Sequence Number = Cookie + 1）が返着した場合のみ、ASA は Cookie 整合性を数学的に検証し、正常と認めてサーバーとのバックエンド 3-Way Handshake を完了させて接続を透過結合（Bridge）する。
3. **Volume-Based DDoS Phase（大容量帯域圧迫フェーズ）**:
   * UDP Flood や NTP/DNS Reflected Amplification 攻撃により、企業 WAN 回線帯域が圧迫。
   * 管理者が SOC/NMS で攻撃元 IP または攻撃対象 IP を特定し、**Trigger Router** 上で BGPBlackhole ルート（例: Next-Hop `192.0.2.1`）をアナウンス。
   * **Edge Routers** 群が iBGP 経由で Route を受信。
     * **Destination RTBH**: 該当 Victim IP 宛ての全トラフィックが Edge の Null0 インターフェイスへ吸い込まれて破棄される。
     * **Source RTBH**: Edge の Ingress インターフェイスで有効化された **Loose uRPF** により、送信元 IP が Null0 を指す攻撃パケットが水際でドロップされる。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### ラボ試験における最重要チェックポイント

1. **ASA MPF での Embryonic Connection Limits 構成**:
   * **`set connection embryonic-conn-max <N>`**: 全体でのハーフオープン接続上限（これを超えると SYN Cookie 発動）。
   * **`set connection per-client-embryonic-max <N>`**: 1 つの送信元 IP あたりのハーフオープン上限（単一ホストからの DoS 阻止）。
   * **`set connection conn-max <N>`**: 確立済み全 TCP 接続の最大数上限。
   * **試験の罠**: 「Limit embryonic connections to 100 for server X and 10 per client」と指示された場合、`access-list` でサーバー IP/Port をピンポイント指定し、`class-map` ➔ `policy-map` ➔ `service-policy`（通常 `outside` または `global`）にバインドする正確な手順が求められます。
2. **ASA Threat Detection ＆ Automatic Shunning**:
   * **Basic Threat Detection**: `threat-detection basic-threat`（デフォルト有効）。
   * **Scanning Threat Detection ＆ Auto Shun**: `threat-detection scanning-threat shun`（スキャン検出時に攻撃者を自動 Shun）。
   * **TCP Intercept Statistics**: `threat-detection statistics tcp-intercept`（SYN 攻撃を受けている Top 10 サーバーの統計収集・表示に必須）。
3. **IOS TCP Intercept (Intercept vs Watch Mode)**:
   * **`ip tcp intercept mode intercept` (デフォルト)**: ルーターが完全に TCP 代理（Proxy）として動作し、クライアントとハンドシェイク完了後にサーバーと接続を確立する。
   * **`ip tcp intercept mode watch`**: ルーターは通信をパススルーさせつつパッシブに監視し、`watch-timeout` (デフォルト 30 秒) 内に 3-Way Handshake が完了しない場合に RST パケットを送信してセッションを強制終了する。
   * **`ip tcp intercept list <ACL>`**: どのトラフィックを Intercept 対象にするかを ACL で制限（**ACL なしで全バインドするとルーター自身の CPU/メモリが爆発してダウンする** ため試験で最重要）。
4. **BGP RTBH (Remotely Triggered Black Hole) 完全構築手順**:
   * **共通前提**: 全エッジルーターで未使用の Blackhole IP（例: `192.0.2.1`）宛ての Static Null0 ルートを事前定義 (`ip route 192.0.2.1 255.255.255.255 Null0`)。
   * **Destination-based RTBH**: Trigger Router で攻撃対象 IP に対する Static Route (`ip route 192.168.1.100 255.255.255.255 Null0 tag 666`) を作成し、BGP で redist / network アナウンス。Route-map で Next-Hop を `192.0.2.1` に書換えてエッジへ共有。
   * **Source-based RTBH**: エッジルーターの WAN インターフェイスで **`ip verify unicast source reachable-via any` (Loose uRPF)** を有効化。Trigger Router から攻撃元 IP（例: `203.0.113.50/32`）の Next-Hop を `192.0.2.1` に変更して BGP 再配布。エッジの Ingress uRPF チェックにより攻撃元パケットがドロップされる。
5. **CoPP (Control Plane Policing) ＆ CPPr (Control Plane Protection)**:
   * **CoPP**: `control-plane` 宛ての通信に対し MQC (`class-map`, `policy-map`) で Rate Limit (Police) をアサイン。
   * **CPPr**: `control-plane host`, `control-plane transit`, `control-plane cef-exception` に分けてポリシング。`host` では `port-filter` や `queue-threshold` が利用可能。

---

## 🛠 設定方法

### 1. ASA / FTD MPF による TCP SYN Flood 防御 (Embryonic Limits)

```text
! ===== ASA CLI 設定例 =====
access-list ACL_PROTECTED_SERVERS extended permit tcp any host 172.16.6.99 eq 80
access-list ACL_PROTECTED_SERVERS extended permit tcp any host 172.16.6.99 eq 443

class-map CM_PROTECTED_SERVERS
 match access-list ACL_PROTECTED_SERVERS

policy-map PM_DOS_PROTECTION
 class CM_PROTECTED_SERVERS
  set connection embryonic-conn-max 1000 per-client-embryonic-max 50
  set connection conn-max 5000 per-client-max 500

service-policy PM_DOS_PROTECTION interface OUTSIDE
```

---

### 2. ASA Threat Detection ＆ Auto-Shun 設定

```text
! ===== Basic ＆ Scanning Threat Detection 有効化 =====
threat-detection basic-threat
threat-detection scanning-threat shun   ! スキャン検出時に攻撃元 IP を自動 Shun
threat-detection statistics tcp-intercept ! TCP Intercept 統計収集

! ===== カスタム判定閾値の調整 (例: 20分間で 1000 drops/sec 超過で発報) =====
threat-detection rate rate-interval 1200 average-rate 1000 burst-rate 100
```

---

### 3. IOS TCP Intercept (Watch / Intercept Mode)

```bash
! ===== 監視対象サーバーを指定する ACL =====
ip access-list extended ACL_TCP_INTERCEPT
 permit tcp any 172.16.13.0 0.0.0.255 eq 80
 permit tcp any host 172.16.13.100

! ===== TCP Intercept (Watch Mode) 構成 =====
ip tcp intercept list ACL_TCP_INTERCEPT
ip tcp intercept mode watch              ! Watch モード (パッシブ監視)
ip tcp intercept watch-timeout 20        ! 20 秒でタイマーアウト
ip tcp intercept max-incomplete high 300 ! ハーフオープン上限 High
ip tcp intercept max-incomplete low 150  ! ハーフオープン上限 Low
```

---

### 4. BGP RTBH (Source-Based Remotely Triggered Black Hole) の完全構築

```bash
! =================================================================
! 1. Edge Router (R1 / R6) の設定
! =================================================================
! (A) Blackhole 宛てプレースホルダー静的ルート
ip route 192.0.2.1 255.255.255.255 Null0

! (B) WAN Ingress インターフェイスで Loose uRPF を有効化
interface GigabitEthernet0/0
 ip verify unicast source reachable-via any ! Source-based RTBH に必須

! (C) BGP 設定 (iBGP Peer からの Blackhole 経路受信)
router bgp 126
 bgp log-neighbor-changes
 neighbor 150.1.2.2 remote-as 126         ! Trigger Router (R2)
 neighbor 150.1.2.2 update-source Loopback0

! =================================================================
! 2. Trigger Router (R2) の設定
! =================================================================
ip route 192.0.2.1 255.255.255.255 Null0

! (A) 攻撃元 IP に対する Static Route (Tag 666 を付与)
ip route 170.170.170.170 255.255.255.255 Null0 tag 666

! (B) Route-Map で Next-Hop を 192.0.2.1 / Community no-export に書き換え
route-map RM_RTBH permit 10
 match tag 666
 set ip next-hop 192.0.2.1
 set local-preference 200
 set community no-export

router bgp 126
 redistribute static route-map RM_RTBH
 neighbor 150.1.1.1 remote-as 126         ! Edge Router R1
 neighbor 150.1.1.1 send-community
```

---

## 🔍 検証コマンド

| 目的 | コマンド (IOS / ASA / FTD) | 表示確認ポイント |
| :--- | :--- | :--- |
| **ASA TCP Intercept ＆ Embryonic 状態確認** | <code>show service-policy interface OUTSIDE set connection detail</code> | `current embryonic conns`, `drop` 件数 |
| **ASA Threat Detection 統計確認** | <code>show threat-detection rate</code> / <code>show threat-detection statistics top tcp-intercept</code> | 攻撃を受けている Top 10 サーバー、現在の Attack Rate |
| **ASA Shun（ブロック中 IP）一覧確認** | <code>show shun</code> | 自動/手動 Shun された IP アドレス、経過時間 |
| **IOS TCP Intercept セッション確認** | <code>show tcp intercept connections</code> / <code>show tcp intercept statistics</code> | Incomplete (ハーフオープン) / Established 接続数 |
| **uRPF ドロップ統計確認 (Source RTBH 用)** | <code>show ip interface Gi0/0</code> / <code>show cef interface Gi0/0</code> | `IP verify source reachable-via ANY`、`verification drops` |
| **BGP RTBH ルート受信・転送確認** | <code>show ip bgp 170.170.170.170</code> / <code>show ip route 170.170.170.170</code> | Next-Hop が `192.0.2.1` ➔ `Null0` にルーティングされているか |
| **FTD 接続ドロップ確認 (ASP Drop)** | <code>show asp drop</code> | `conn-limit` (接続制限ドロップ), `tcp-3whs-failed` |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認コマンド | 対処方法 |
| :--- | :--- | :--- | :--- |
| **Embryonic Limit に達しても ASA が SYN Cookie を起動せずサーバーがダウンする** | MPF の `service-policy` が対象インターフェイス（`outside` 等）にバインドされていない | <code>show service-policy interface</code> | `service-policy <PM_NAME> interface outside` コマンドでインターフェイスへ確実にアサインする。 |
| **IOS TCP Intercept を有効化するとルーターの CPU 使用率が 100% に高騰する** | `ip tcp intercept list` に ACL を指定しておらず、全通過 TCP 通信を対象にインターセプトしている | <code>show processes cpu sorted</code><br><code>show running-config \| inc tcp intercept</code> | `ip tcp intercept list <ACL>` で保護対象の特定サーバー/サブネット宛てのみにインターセプト範囲を限定する。 |
| **Source-Based RTBH で Trigger 側から BGP アナウンスしたが攻撃が止まらない** | エッジルーターの WAN インターフェイスで Loose uRPF (`ip verify unicast source reachable-via any`) が未設定 | <code>show ip interface Gi0/0</code> | エッジルーターの Ingress インターフェイスに Loose uRPF を設定して、送信元 Lookup による Null0 破棄を有効化する。 |
| **Destination RTBH を発動したら、該当サーバー宛ての正規通信もすべて切断された** | Destination RTBH の正常な副作用 (Collateral Damage)。被害者 IP 宛て全通信が Null0 破棄されるため | <code>show ip bgp</code> | 正常通信を維持したい場合は BGP Flowspec (RFC 5575) を導入し、L4 ポートやプロトコル単位でピンポイント破棄/Policing する。 |
| **ASA Threat Detection で管理者 PC が誤って Shun されて管理不能になる** | スキャン検出閾値が低すぎる、または管理端末の多重セッションがスキャンと判定された | <code>show shun</code><br><code>show threat-detection scanning-threat</code> | `clear shun` で解除し、管理端末 IP を `threat-detection scanning-threat exclude host <IP>` で除外登録する。 |

---

## ⚠ 制限事項

* **BGP Destination RTBH の Collateral Damage（二次被害）**:
  * Destination-based RTBH は、攻撃対象 IP（Victim）宛ての全パケットを Null0 へ破棄するため、**攻撃トラフィックだけでなく正規ユーザーからの通信も完全に遮断** されます（被害者を自らネットワークから切り離す手法）。
* **uRPF 方式の依存性**:
  * Source-based RTBH を機能させるには、すべてのエッジルーターで **Loose Mode uRPF** が有効化されている必要があります。Strict Mode uRPF では Default Route 経由や複数経路環境で期待通りに動作しません。
* **ASA SYN Cookie と TCP オプションの脱落**:
  * ASA が SYN Cookie 応答を返す際、標準の TCP Window Scale や SACK (Selective Acknowledgment) などの拡張オプション情報が初期ハンドシェイクで一部欠落・ネゴシエーション制限を受ける場合があります。

---

## 🔄 他技術との関連

* **Unicast RPF (uRPF)**: Source-based RTBH と組み合わせて、スプーフィングされた攻撃元 IP パケットを Ingress インターフェイスで水際ドロップ。
* **CoPP / CPPr**: ルーター/ファイアウォール自体の Route Processor に対する DoS 攻撃を Rate Limit (Police) して制御プレーン死守。
* **Security Intelligence (SI) ＆ Cisco Umbrella**: 既知の C2、Botnet、DDoS 攻撃元 IP/Domain を DNS レベルおよび IP レベルで早期 Sinkhole / Drop。
* **Radware DefensePro (Firepower 9300/4100 統合)**: 専用ハードウェア DDoS ミティゲーションモジュールとの統合連携。

---

## 🧩 比較表

### 1. ASA SYN Cookie (MPF) vs IOS TCP Intercept (Intercept/Watch) vs BGP RTBH

| 比較項目 | ASA / FTD MPF (SYN Cookie) | IOS TCP Intercept (Intercept Mode) | BGP RTBH (Source-Based) |
| :--- | :--- | :--- | :--- |
| **主な保護対象** | ファイアウォール背後の Web/TCP サーバー | ルーター背後の TCP サーバー | ネットワークインフラ全域・回線帯域 |
| **動作レイヤー** | Layer 3 / 4 (Stateful Firewall) | Layer 3 / 4 (Router Engine) | Layer 3 (Routing / BGP) |
| **ステート記憶** | **メモリステート保持なし** (SYN Cookie 計算) | ルーター側で各セッションステートを保持 | ルーティングテーブル (FIB) / uRPF のみ |
| **CPU/メモリ負荷** | 極めて低い（DDoS 耐性が高い） | 攻撃規模が大きいとルーターのメモリ枯渇 | ゼロ（CEF ハードウェア転送で Null0 破棄） |
| **発動条件** | `embryonic-conn-max` 閾値超過時 | 常時 または ACL マッチ時 | Trigger Router での BGP 経路アナウンス時 |

---

## 💡 ベストプラクティス

1. **ASA/FTD での 2 段階 Embryonic 閾値設計**:
   * バックエンド Web サーバーの TCP SYN Backlog Queue サイズ（例: 2048）よりも**必ず低い値**（例: 1000）で `embryonic-conn-max` を設定し、サーバーが倒れる前にファイアウォールで SYN Cookie 動作を起動させます。また、1 ホストからの枯渇攻撃を防ぐため `per-client-embryonic-max`（例: 50）も併用します。
2. **BGP RTBH での `no-export` Community 付与**:
   * RTBH アナウンス用 BGP ルートには、必ず `set community no-export` を付与し、意図せず外部 AS (ISP) へブラックホールルートが伝洩・拡散しないよう安全対策を施します。

---

## 📝 ラボ学習・設定サンプル例

### 例1: ASA MPF による Web サーバーの TCP SYN Flood 防御

#### 問題
DMZ 内の Web サーバー (`172.16.6.99:80`) を TCP SYN Flood 攻撃から保護するため、ハーフオープン接続が 500 に達した場合に TCP Intercept (SYN Cookie) 動作を開始させよ。また、単一クライアントあたりのハーフオープン上限を 20 に制限せよ。

#### 要件
* MPF を使用して `outside` インターフェイスに適用すること。

#### 設定・操作手順

```text
! ===== Step 1: 判定用 ACL ＆ Class-Map の作成 =====
access-list ACL_WEB_DOS extended permit tcp any host 172.16.6.99 eq 80

class-map CM_WEB_DOS
 match access-list ACL_WEB_DOS

! ===== Step 2: Policy-Map で Embryonic 閾値を設定 =====
policy-map PM_PROTECT_DMZ
 class CM_WEB_DOS
  set connection embryonic-conn-max 500 per-client-embryonic-max 20

! ===== Step 3: Outside インターフェイスへバインド =====
service-policy PM_PROTECT_DMZ interface outside
```

#### 検証方法
* `show service-policy interface outside set connection detail` を実行し、`Set connection policy: embryonic-conn-max 500 per-client-embryonic-max 20` が適用されていることを確認。

---

### 例2: ASA Threat Detection によるポートスキャン自動 Shun

#### 問題
ASA においてネットワークスイープおよびポートスキャン（Scanning Threat）を自動検知し、該当する攻撃元 IP アドレスを 30 分間自動的に Shun（遮断）せよ。

#### 要件
* `threat-detection` コマンドを使用すること。

#### 設定・操作手順

```text
ASA1(config)# threat-detection basic-threat
ASA1(config)# threat-detection scanning-threat shun
ASA1(config)# threat-detection statistics tcp-intercept
```

#### 検証方法
* 外部端末から Nmap スキャンを実行後、ASA で `show threat-detection scanning-threat` および `show shun` を実行し、攻撃元 IP が Shun リストに登録されていることを確認。

---

### 例3: IOS TCP Intercept (Watch Mode) の構成

#### 問題
ルーター R3 において、VLAN 13 (`136.1.13.0/24`) 宛ての Telnet/HTTP 通信に対する SYN Flood 攻撃を防ぐため、TCP Intercept を Watch モードで有効化せよ。

#### 要件
* 監視タイムアウト: 20 秒
* ハーフオープン上限: High 200 / Low 100

#### 設定・操作手順

```bash
R3(config)# ip access-list extended ACL_INTERCEPT_VLAN13
R3(config-ext-nacl)# permit tcp any 136.1.13.0 0.0.0.255

R3(config)# ip tcp intercept list ACL_INTERCEPT_VLAN13
R3(config)# ip tcp intercept mode watch
R3(config)# ip tcp intercept watch-timeout 20
R3(config)# ip tcp intercept max-incomplete high 200
R3(config)# ip tcp intercept max-incomplete low 100
```

#### 検証方法
* `show tcp intercept connections` および `show tcp intercept statistics` を実行し、Watch モードでセッションがトラッキングされていることを確認。

---

### 例4: BGP Source-Based RTBH (Remotely Triggered Black Hole) の実装

#### 問題
AS 23 内において、攻撃元 IP `170.170.170.170` からの DoS 攻撃を遮断するため、Trigger Router (R2) から BGP を用いて Edge Router (R3) へ Blackhole 経路を伝達させよ。

#### 要件
* R3 の Ingress インターフェイスで Loose uRPF を使用すること。
* Blackhole Next-Hop IP: `192.0.2.1`

#### 設定・操作手順

```bash
! ===== Edge Router (R3) =====
ip route 192.0.2.1 255.255.255.255 Null0

interface FastEthernet0/0
 ip verify unicast source reachable-via any ! Loose uRPF

! ===== Trigger Router (R2) =====
ip route 192.0.2.1 255.255.255.255 Null0
ip route 170.170.170.170 255.255.255.255 Null0 tag 666

route-map RM_RTBH permit 10
 match tag 666
 set ip next-hop 192.0.2.1
 set community no-export

router bgp 23
 redistribute static route-map RM_RTBH
 neighbor 136.1.23.3 send-community
```

#### 検証方法
* R3 上で `show ip route 170.170.170.170` を実行し、Next-Hop が `192.0.2.1` (Null0) であることを確認。`170.170.170.170` からのパケットが uRPF ドロップされることを検証。

---

### 例5: BGP Destination-Based RTBH の実装

#### 問題
攻撃を受けている内部サーバー `10.1.0.100` 宛てのパケットを、iBGP 経由でネットワーク境界ルーター (R3) の Null0 へドロップさせよ。

#### 要件
* Trigger Router (R2) で Static タグ付きルートを作成して BGP 再配布すること。

#### 設定・操作手順

```bash
! ===== Edge Router (R3) =====
ip route 192.0.2.1 255.255.255.255 Null0

! ===== Trigger Router (R2) =====
ip route 10.1.0.100 255.255.255.255 Null0 tag 999

route-map RM_DEST_RTBH permit 10
 match tag 999
 set ip next-hop 192.0.2.1
 set origin igp

router bgp 23
 redistribute static route-map RM_DEST_RTBH
```

#### 検証方法
* R3 で `show ip bgp 10.1.0.100` を確認し、Next-Hop が `192.0.2.1` に書き換えられて Null0 へ転送破棄されることを確認。

---

### 例6: CoPP による Control Plane 宛て ICMP / UDP Flood 制限

#### 問題
ルーター CPU に対する ICMP および UDP フラッド攻撃を防ぐため、CoPP を使用して ICMP を 64 Kbps、UDP を 128 Kbps にレート制限せよ。

#### 要件
* MQC を使用して `control-plane` にバインドすること。

#### 設定・操作手順

```bash
R1(config)# ip access-list extended ACL_ICMP_CP
R1(config-ext-nacl)# permit icmp any any

R1(config)# ip access-list extended ACL_UDP_CP
R1(config-ext-nacl)# permit udp any any

R1(config)# class-map CM_ICMP
R1(config-cmap)# match access-group name ACL_ICMP_CP
R1(config)# class-map CM_UDP
R1(config-cmap)# match access-group name ACL_UDP_CP

R1(config)# policy-map PM_COPP_DOS
R1(config-pmap)# class CM_ICMP
R1(config-pmap-c)# police 64000 conform-action transmit exceed-action drop
R1(config-pmap)# class CM_UDP
R1(config-pmap-c)# police 128000 conform-action transmit exceed-action drop

R1(config)# control-plane
R1(config-cp)# service-policy input PM_COPP_DOS
```

#### 検証方法
* 外部から大容量 ICMP ping をルーター IP 宛てに送信し、`show policy-map control-plane` で `exceeded` ドロップカウンターが増加することを確認。

---

### 例7: FTD Prefilter Policy による Fastpath (DoS 早期ドロップ)

#### 問題
FTD において、特定攻撃元サブネット (`203.0.113.0/24`) からの通信を Snort DPI エンジンに引き渡す前に、LINA データプレーン (Prefilter) 段階で超高速に Block (Drop) せよ。

#### 要件
* FMC Prefilter Policy を使用すること。

#### 設定・操作手順

```text
1. FMC UI で Policies > Access Control > Prefilter を開く。
2. 対象 Prefilter Policy を編集し "Add Prefilter Rule" を選択:
   - Name: FASTPATH_BLOCK_ATTACKERS
   - Action: Block
   - Source Networks: 203.0.113.0/24
3. Save して Access Control Policy にアサインの上 Deploy。
```

#### 検証方法
* FTD CLI で `show asp drop` を実行し、Snort に渡る前に Prefilter 段階でパケットが即時ドロップされることを確認。

---

### 例8: ASA 管理アクセスに対する Control Plane 攻撃防止 (Management Class)

#### 問題
ASA 自身の INSIDE インターフェイス宛ての SSH / Telnet 管理通信において、ハーフオープン接続数が 10 を超えた場合に接続を拒否せよ。

#### 要件
* `class-map type management` を使用すること。

#### 設定・操作手順

```text
ASA1(config)# class-map type management CM_MGMT_SSH
ASA1(config-cmap)# match port tcp eq 22

ASA1(config)# policy-map PM_MGMT_PROTECT
ASA1(config-pmap)# class CM_MGMT_SSH
ASA1(config-pmap-c)# set connection embryonic-conn-max 10

ASA1(config)# service-policy PM_MGMT_PROTECT interface INSIDE
```

#### 検証方法
* `show service-policy interface INSIDE set connection detail` を実行し、Management Class で `embryonic-conn-max 10` がアサインされていることを確認。

---

### 例9: ASA IP フラグメント攻撃防止 (Fragment Chain 変更)

#### 問題
ASA に対する Fragment Flood DoS 攻撃を防ぐため、リアセンブル可能なフラグメントチェーン上限数をデフォルトの 24 から `1` (実質フラグメント不許可) に制限せよ。

#### 要件
* `fragment` コマンドを使用すること。

#### 設定・操作手順

```text
ASA1(config)# fragment chain 1 OUTSIDE
ASA1(config)# fragment chain 1 INSIDE
```

#### 検証方法
* `show running-config fragment` を実行し、各インターフェイスで `fragment chain 1` が設定されていることを確認。

---

### 例10: FMC REST API を使用した Threat Detection ステータスの取得

#### 問題
FMC REST API を使用して、Managed FTD 上の DoS / Threat Detection 関連のイベント・設定状態情報を GET リクエストで取得せよ。

#### 要件
* REST API エンドポイントを使用すること。

#### 設定・操作手順

```bash
# ===== Auth Token 取得後、デバイス設定情報を GET =====
curl -k -X GET "https://fmc.corp.local/api/fmc_config/v1/domain/e276abec-e0f2-11e3-8169-6d9ed13b206f/devices/devicerecords" \
  -H "X-auth-access-token: eyJhbGciOiJSUzI1Ni..." \
  -H "Content-Type: application/json"
```

#### 検証方法
* JSON レスポンス内に正常なデバイスレコードおよびステータス情報が返却されることを確認。

---

## ❓ 想定試験問題

### 1. コンフィグ読解問題

**問題**: ASA 上で以下の MPF 設定を投入して Deploy した。

```text
access-list ACL_SERVERS extended permit tcp any host 10.1.1.100 eq 80
class-map CM_SERVERS
 match access-list ACL_SERVERS
policy-map PM_DOS
 class CM_SERVERS
  set connection embryonic-conn-max 100 per-client-embryonic-max 10
service-policy PM_DOS interface outside
```

外部の複数ホストから `10.1.1.100:80` 宛てに合計 150 個の TCP SYN パケット（ACK なし）が届いた場合、ASA の動作として正しいものはどれか。

* A) 100 個を超えた 50 個の SYN パケットは即座に無条件ドロップされる。
* B) ハーフオープン接続数が 100 を超えた時点で ASA は SYN Cookie モードを起動し、以降の SYN パケットに対して ASA 自体が SYN Cookie を埋め込んだ SYN-ACK を応答する。
* C) サーバー `10.1.1.100` が過負荷でダウンする。
* D) ASA は `10.1.1.100` 宛ての全通信を自動的に Shun する。

**正解**: **B**  
**解説**: ASA の `embryonic-conn-max` 閾値（本設定では 100）を超えると、ASA は **TCP Intercept (SYN Cookie)** 動作を開始します。パケットを直ちに破棄するのではなく、ASA 自体がプロキシとして SYN Cookie を含む SYN-ACK を応答し、クライアントから正しい ACK が返ってきた場合のみサーバーと接続を結合します。

---

### 2. トラブルシュート問題

**問題**: 攻撃元 IP `203.0.113.88` からの大規模 DoS 攻撃を阻止するため、Trigger Router (R2) から Source-Based RTBH の BGP ルートをアナウンスした。しかし、Edge Router (R1) の WAN インターフェイス `Gi0/0` を通過する攻撃パケットが破棄されず、内部へ通過し続けている。R1 の BGP テーブルでは `203.0.113.88/32` の Next-Hop が `192.0.2.1` (Null0 宛て) に正しく学習されていた。原因として最も可能性が高いものはどれか。

* A) R1 の WAN インターフェイス `Gi0/0` で Loose uRPF (`ip verify unicast source reachable-via any`) が設定されていない。
* B) R1 上で CEF (`ip cef`) が無効化されているため。
* C) BGP の Community 値が `no-export` になっていないため。
* D) R1 の RAM が不足しているため。

**正解**: **A**  
**解説**: **Source-Based RTBH** では、攻撃元 IP（Source IP）宛ての Next-Hop を Null0 にアサインした BGP 経路を配布します。パケットの「送信元 IP」をテーブル照合して Null0 へ破棄させるには、**着信インターフェイスで Loose uRPF (`ip verify unicast source reachable-via any`) が有効化されていることが必須要件** です。uRPF が未設定の場合、宛先 IP (Destination IP) のみで転送されるため攻撃パケットは通過してしまいます。

---

### 3. Design 問題

**問題**: 外部からの大容量 DDoS 攻撃（Volume-based Attack）が発生した際、被害を受けている自社の公開 IP アドレス (`192.168.10.50`) 宛ての通信を ISP 境界でブラックホール化（Destination RTBH）することを検討している。設計上の最大のトレードオフ（懸念事項）はどれか。

* A) ASA の CPU 使用率が 100% に高騰する。
* B) 攻撃トラフィックだけでなく、該当 IP アドレス (`192.168.10.50`) 宛てのすべての正常なユーザー通信も完全に遮断される (Collateral Damage)。
* C) BGP セッションが切断される。
* D) 暗号化 VPN トンネルが強制解除される。

**正解**: **B**  
**解説**: **Destination-Based RTBH** は、被害者 IP (Victim) 宛ての全トラフィックを Null0 へ落とす手法です。回線破綻を防ぐ効果はありますが、**正常なユーザーからの通信も一切届かなくなる（二次被害 / Collateral Damage）** という大きなトレードオフが存在します。

---

### 4. 実装問題

**問題**: IOS ルーター R1 において、全通過 TCP 通信ではなく、特定の Web サーバー (`10.1.1.200`) 宛ての HTTP アクセスに対してのみ TCP Intercept (Watch Mode) を有効化したい。正しいコマンド構成はどれか。

* A)  
  `ip tcp intercept mode watch`  
  `ip tcp intercept list 100`  
  (access-list 100: `permit tcp any host 10.1.1.200 eq 80`)
* B)  
  `ip tcp intercept mode intercept`  
  `ip tcp intercept all`
* C)  
  `ip verify source reachable-via rx`
* D)  
  `threat-detection scanning-threat shun`

**正解**: **A**  
**解説**: IOS で TCP Intercept を安全に運用するには、**`ip tcp intercept list <ACL>`** を使用して保護対象のサーバー/ポートを ACL で明示指定することが必須です。また、`mode watch` コマンドでパッシブ監視モードへ変更します。

---

### 5. コンフィグ読解問題

**問題**: ASA 上で `threat-detection scanning-threat shun` コマンドを設定した。このコマンドによって提供されるセキュリティ機能として正しいものはどれか。

* A) 暗号化通信を自動解読する。
* B) ポートスキャンやネットワークスイープを行う攻撃元 IP アドレスを ASA が自動的に検知し、そのホストからの通信を `shun`（動的 ACL ドロップ）して一定時間自動ブロックする。
* C) BGP のルートを Null0 に書き換える。
* D) DNS 問い合わせを Cisco Umbrella へ転送する。

**正解**: **B**  
**解説**: **`threat-detection scanning-threat shun`** は、ASA がネットワークスキャン攻撃を動的検出した際、攻撃元の IP アドレスを自動的に `shun` リストに追加し、以降のパケットを自動遮断する機能です。

---

## 🔗 参考リソース

* [Cisco ASA Series Configuration Guide: Threat Detection and TCP Intercept (MPF)](https://www.cisco.com/)
* [Cisco IOS Security Configuration Guide: TCP Intercept](https://www.cisco.com/)
* [Cisco White Paper: Remotely Triggered Black Hole (RTBH) Filtering](https://www.cisco.com/)
* [Cisco Technical Notes: Configure Control Plane Policing (CoPP) on Cisco IOS Routers](https://www.cisco.com/)
* [Cisco Live: Advanced DDoS Mitigation Strategies (BRKSEC-3021)](https://www.ciscolive.com/)

---

## 📝 **補足（Notes）**

* **学習メモ**:
  * CCIE Security ラボ試験で DoS/DDoS 対策が出題された場合の構成選択チャート：
    * **「Protect DMZ Server from SYN Flood」** ➔ ASA/FTD MPF (`set connection embryonic-conn-max <N> per-client-embryonic-max <N>`)。
    * **「Automatically block scanning hosts on ASA」** ➔ `threat-detection scanning-threat shun` ＆ `threat-detection statistics tcp-intercept`。
    * **「Block attack source IP via BGP on Edge Routers」** ➔ Source-Based RTBH (Trigger Router で Static Tag ➔ BGP Redistribution / Edge Router で `ip verify unicast source reachable-via any` ＋ Static Null0)。
    * **「Protect Router CPU from packet spikes」** ➔ CoPP (`control-plane` ➔ `service-policy input <PM>`)。
* **図解（SYN Flood 防御 vs RTBH 遮断ポイント）**:
  ```text
  [ Attacker ] ─── Volume DDoS (UDP/SYN) ───► [ ISP / Edge Router ] ───► [ Firewall (ASA/FTD) ] ───► [ Web Server ]
                                                      │                         │
                                                      ├── Source RTBH Drop      └── TCP Intercept (SYN Cookie)
                                                      │   (via Loose uRPF)          (Protects Server Memory)
                                                      └── Dest RTBH Drop
                                                          (Null0 Interface)
  ```
* **注意点**:
  * ラボ試験で ASA の Embryonic Limit や TCP Intercept を設定した後は、`show service-policy interface <IF> set connection detail` や `show threat-detection rate` を実行し、ドロップ・ハーフオープンカウンターが正常にトータル記録されていることを必ず検証・確認してください。

---
