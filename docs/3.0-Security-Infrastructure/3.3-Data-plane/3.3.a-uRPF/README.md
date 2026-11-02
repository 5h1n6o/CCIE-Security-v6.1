---
layout: default
title: 3.3.a-uRPF
nav_order: 1
parent: 3.3-Data-plane
grand_parent: 3.0-Security-Infrastructure
---

# 3.3.a uRPF (Unicast Reverse Path Forwarding)

---

## 📘 概要

Unicast Reverse Path Forwarding（uRPF）は、ルーターやファイアウォールなどのレイヤー3ネットワークデバイスにおいて、受信したIPパケットの送信元IPアドレス（Source IP Address）の正当性をルーティングテーブル（FIB: Forwarding Information Base）を用いて動的に検証し、IPアドレススプーフィング（なりすまし）攻撃や分散型サービス拒否（DDoS）攻撃を未然に遮断するデータプレーン保護（Data Plane Protection）技術である。

従来のルーターは、パケットの宛先IPアドレス（Destination IP）のみをルーティングテーブルと照合して転送判断を行う。このため、送信元IPアドレスが偽造された攻撃パケットであっても、宛先が存在する限りネットワーク内を正常に転送されてしまい、DDoS攻撃の踏み台や反射攻撃（Reflective Attack）に悪用される原因となる。uRPFを有効にすると、デバイスはパケットを受信したインターフェイス（Ingress Interface）と、その送信元IPアドレスへの戻り経路（Reverse Path）を比較検証し、不一致または経路が存在しない場合に即座にパケットを破棄する。

* **機能概要**: 受信パケットの送信元IPをCEF（Cisco Express Forwarding）のFIBテーブルで逆引き検索し、入力インターフェイスまたは経路の存在を確認してスプーフィングパケットを破棄する。
* **利用目的**: ネットワーク境界（Perimeter/Edge）でのIPスプーフィング対策、BotnetからのDoS/DDoS攻撃緩和、BGP RTBH（Remotely Triggered Black Hole）との連携による攻撃トラフィックの動的ブラックホール化。
* **どのような場面で利用するか**:
  1. **ISP / Enterprise Perimeter Edge**: 外部インターネットやサードパーティ接続境界において、自社割り当てIP外からの送信元偽装パケットの侵入を阻止。
  2. **Access / Aggregation Switch**: クライアントサブネットから送信されるIPスプーフィングパケットを受信ポートで遮断。
  3. **BGP RTBH連携ネットワーク**: 攻撃対象または攻撃元IPをNull0へ誘導するBGPルートを配信し、uRPF Loose Modeで動的にドロップ。

---

## 🔑 要点

| 項目 | 内容 |
| --- | --- |
| **特徴** | CEF/FIBテーブルを利用し、受信パケットの送信元IPアドレスを逆引き検索（Reverse Path Check）してスプーフィングを検出 |
| **動作モード** | **Strict Mode**（入力IFと戻りIFの完全一致を要求） / **Loose Mode**（FIBに戻り経路が存在することのみを要求） |
| **前提条件** | Cisco IOS/IOS-XEでは **CEF（Cisco Express Forwarding）の有効化** が必須 (`ip cef` / `ipv6 cef`) |
| **オプション機能** | `allow-default`（デフォルトルートを有効な戻り経路とみなす） / `allow-self-ping`（ルーター自アドレス宛のセルフピンを許可） / オプションACL（例外許可・ログ取得） |
| **メリット** | スタティックなIngress ACL（Anti-Spoofing ACL）の管理負荷を大幅に削減し、動的な経路変化に自動追従 |
| **デメリット** | 非対称ルーティング（Asymmetric Routing）環境でStrict Modeを適用すると正常な通信が破棄されるリスク |
| **対応機種** | Cisco ISR/ASR/Catalyst（IOS/IOS-XE）、Cisco ASA（Security Appliance）、Cisco FTD（Threat Defense / FMC管理） |
| **制限事項** | PBR（Policy-Based Routing）のルートマップはuRPFのFIB検証に反映されない。トンネル構成（GRE/IPsec）でのインターフェイス指定に注意 |
| **設計上の注意点** | マルチホーム環境やマルチパス構成ではLoose Modeを採用するか、`allow-default` / 例外ACLを慎重に設計すること |

---

## 🏗 動作原理

uRPFは、CEF（Cisco Express Forwarding）が保持するFIB（Forwarding Information Base）構造を活用して高速に動作する。処理ロジックは Strict Mode と Loose Mode で大きく異なる。

### 1. Strict Mode（厳格モード）の動作原理

Strict Modeでは、受信したパケットの送信元IPアドレスに対してFIBテーブルを参照し、**「その送信元IPへパケットを送り返すための最適出力を担当するインターフェイス」** と **「パケットが実際に入力されたインターフェイス」** が完全一致（Best Return Path Match）しているかを検証する。

```text
[ Attacker / Client ] (Src: 190.1.1.10)
        │
        ▼ (Ingress: GigabitEthernet0/0)
┌────────────────────────────────────────────────────────┐
│ Router (uRPF Strict Mode enabled on Gi0/0)             │
│                                                        │
│ 1. Extract Source IP: 190.1.1.10                       │
│ 2. Lookup FIB for 190.1.1.10                           │
│    -> FIB Result: Best Egress Interface = Gi0/0        │
│ 3. Match Verification:                                 │
│    FIB Interface (Gi0/0) == Ingress Interface (Gi0/0) │
│                                                        │
│ [PASS] -> Forward to Destination                       │
│ [FAIL] -> Drop Packet (Increment Drop Counter)          │
└────────────────────────────────────────────────────────┘
```

非対称ルーティング環境（行きと帰りの経路が異なるネットワーク構成）でStrict Modeを使用すると、送信元IPへの戻り経路が別インターフェイス（例: Gi0/1）を指しているため、正当なパケットであっても「Ingress IF (Gi0/0) ≠ FIB Egress IF (Gi0/1)」と判定され、誤って破棄される。

### 2. Loose Mode（ルーズモード）の動作原理

Loose Mode（Reachable-via any）では、パケットが入ってきたインターフェイスは不問とされ、**「送信元IPアドレスに対応するルートがFIBテーブルに1つでも存在するか（到達可能か）」** のみを検証する。

```text
[ Multi-Homed ISP / Attacker ] (Src: 200.1.1.5)
        │
        ▼ (Ingress: GigabitEthernet0/1)
┌────────────────────────────────────────────────────────┐
│ Router (uRPF Loose Mode enabled on Gi0/1)              │
│                                                        │
│ 1. Extract Source IP: 200.1.1.5                        │
│ 2. Lookup FIB for 200.1.1.5                            │
│    -> Case A: Route exists in FIB (via any IF/Next-hop)│
│       [PASS] -> Forward Packet                         │
│    -> Case B: No route in FIB OR Route points to Null0 │
│       [FAIL] -> Drop Packet                            │
└────────────────────────────────────────────────────────┘
```

Loose Modeは非対称ルーティングが存在するマルチホームのインターネット接続環境や、BGP RTBH（Remotely Triggered Black Hole）構成に最適である。RTBHでは、攻撃元IPアドレスに対するスタティックルートやBGPルートのネクストホップを `Null0` に設定して全ルーターへ広報する。Loose Modeが有効なインターフェイスでは、送信元IPのFIB検索結果が `Null0` となるため、攻撃パケットをインgress側で即座にドロップできる。

---

## ⚙ 動作シーケンス

パケットがuRPF有効化インターフェイスに到着してから処理されるまでの詳細なステップを以下に示す。

```text
[ Ingress IP Packet Arrival ]
            │
            ▼
[ Step 1: Physical / L2 Parsing & Basic Integrity Check ]
            │
            ▼
[ Step 2: Ingress ACL Check ]
            │ (Permitted by Ingress ACL)
            ▼
[ Step 3: uRPF Source IP Verification (FIB Lookup) ]
            │
            ├─► Is Source IP in FIB?
            │     ├─► No -> [ Drop / Check uRPF ACL ]
            │     └─► Yes
            │           │
            │           ├─► Is Route pointing to Null0?
            │           │     └─► Yes -> [ DROP ]
            │           │
            │           └─► Mode Check:
            │                 ├─► Loose Mode (reachable-via any):
            │                 │     └─► PASS
            │                 │
            │                 └─► Strict Mode (reachable-via rx):
            │                       ├─► Egress IF matches Ingress IF? -> PASS
            │                       ├─► Matches Default Route?
            │                       │     ├─► 'allow-default' configured? -> PASS
            │                       │     └─► Not configured -> DROP
            │                       └─► Self-Ping Packet?
            │                             ├─► 'allow-self-ping' configured? -> PASS
            │                             └─► Not configured -> DROP
            │
            ▼ (If uRPF Check Fails)
[ Step 4: Optional uRPF Exception ACL Evaluation ]
            │
            ├─► Matched 'permit' ACE -> PASS (Forward & Log if 'log' specified)
            └─► Matched 'deny' ACE / Default Deny -> DROP (Increment Drop Counters)
            │
            ▼ (If uRPF Check Passes)
[ Step 5: Destination IP Lookup & Normal CEF Forwarding ]
```

1. **レイヤー2/3パケット解析**: インターフェイスでパケットを受信し、IPヘッダーチェックサムや基本整合性を確認。
2. **通常のIngress ACLの適用**: インターフェイスに適用されている標準/拡張インバウンドACLがあれば評価。拒否された場合はその時点でドロップ。
3. **uRPF（送信元IP）検証**:
   * CEFのFIB構造を参照し、パケットの送信元IPアドレスを検索。
   * 送信元IPのFIBエントリが `Null0` を指している場合は不問でドロップ（RTBH動作）。
   * **Strict Mode**: FIBの出力インターフェイスと入力インターフェイスが一致すれば通過。不一致でデフォルトルートにヒットした場合、`allow-default` オプションの有無を確認。自デバイス宛パケットで不一致の場合、`allow-self-ping` オプションを確認。
   * **Loose Mode**: FIBに有効なルートが存在すれば（出力IFを問わず）通過。
4. **uRPF例外ACLの評価（設定されている場合）**:
   * uRPF違反と判定されたパケットは、コマンドラインで指定された例外ACL（`access-list`）で最終確認が行われる。
   * 例外ACLで `permit` されたパケットはドロップを免れ、転送される（ログ出力可能）。
   * `deny` またはACLの末尾暗黙の拒否にヒットしたパケットは正式にドロップされる。
5. **Destination IP転送処理**: uRPFチェックをパスしたパケットは、通常の宛先IPアドレスに基づくCEF転送処理へと移行する。

---

## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security v6.1 ラボ試験において、uRPFは「単独での設定問題」だけでなく、「BGP RTBH」「IPsec VPN」「Zone-Based Firewall」「ASA/FTD セキュリティポリシー」との複合問題として頻出する。

### 1. Blueprintで重要なポイント
* **IOS/IOS-XEにおける新旧コマンド構文の理解**:
  * 旧構文: `ip verify unicast reverse-path [list] [allow-default]`
  * 新構文: `ip verify unicast source reachable-via {rx | any} [allow-default] [allow-self-ping] [access-list]`
  * IPv6構文: `ipv6 verify unicast source reachable-via {rx | any} [allow-default] [allow-self-ping] [access-list]`
* **モードの選定判断**:
  * シングルホーム・末端アクセスポート・厳格な対スプーフィング: **Strict Mode (`rx`)**
  * 非対称ルート・マルチホームISP・BGP RTBH連携: **Loose Mode (`any`)**
* **ASAおよびFTDでのAnti-Spoofing構文と仕様差**:
  * ASA: `ip verify reverse-path interface <interface_name>`
  * FTD (FMC): Device Management -> Interfaces -> Advanced -> Security Configuration -> Check **Enable Anti Spoofing**

### 2. ラボ試験で設定させられそうな要件と落とし穴

* **要件1: 「非対称ルーティングが発生するインターフェイスで、ドス攻撃用のNull0ルートへ誘導されたパケットのみをドロップし、通常の通信は通過させよ」**
  * **対策**: Loose Mode (`reachable-via any`) を設定する。Strict Modeを設定すると非対称通信が全滅する。
* **要件2: 「デフォルトルートのみで運用されている拠点ルーターで、Strict Mode uRPFを有効化しつつ、インターネット宛の通信が破棄されないようにせよ」**
  * **対策**: `allow-default` オプションを付与する (`ip verify unicast source reachable-via rx allow-default`)。付与しない場合、FIBでデフォルトルートにしかマッチしない送信元IPからのパケットは全てドロップされる。
* **要件3: 「ルーター自身のセカンダリIPやLoopback宛のICMP/管理通信がuRPFによって破棄されないようにせよ」**
  * **対策**: `allow-self-ping` オプションを付与する (`ip verify unicast source reachable-via rx allow-self-ping`)。
* **要件4: 「特定のテスト用送信元IP（例: 172.16.1.0/24）からのスプーフィング違反パケットをドロップせず、ログに記録して通過させよ」**
  * **対策**: 例外ACLを作成し、該当サブネットを `permit`（ログ付き）にしてuRPFコマンドに指定する。

### 3. よくある設定ミス・トラブル

* **CEFの有効化忘れ**: `ip cef` または `ipv6 cef` がグローバルで有効化されていない場合、uRPFコマンドを入力しても動作しないか、全パケットがドロップ/プロセススイッチングにフォールバックする。
* **uRPF例外ACLのpermit/denyの論理逆転**:
  * uRPFの例外ACLにおいて、`permit` は **「uRPF違反があってもドロップせず通過を許可する」** ことを意味する。
  * `deny` は **「uRPF違反パケットを通常通りドロップする」** ことを意味する。通常のアクセスリストのイメージ（permit＝許可、deny＝拒否）で逆にしてしまうミスが非常に多い。
* **PBR（Policy-Based Routing）との併用事故**: PBRによってネクストホップが変更されていても、uRPFは標準のFIB（ルーティングテーブル）を参照する。そのため、PBRで迂回させているトラフィックがuRPF Strict Modeによって不正パケットとみなされてドロップされる。PBR使用時はuRPFを無効化するかLoose Modeにする必要がある。

### 4. showコマンドおよびdebugログによる状態判定

#### (1) IOS / IOS-XE 検証コマンド
* `show ip interface <interface>`: uRPFの適用状態、モード、ドロップ数、抑制されたドロップ数（Suppressed Drops）を確認。
  ```text
  R1# show ip interface gigabitEthernet 0/0.17 | include IP verify
    IP verify source reachable-via RX, allow self-ping, ACL 100
    56 verification drops
    12 suppressed verification drops
  ```
* `show cef interface <interface>`: CEFレベルでのuRPF有効化状態を確認。
  ```text
  R1# show cef interface gigabitEthernet 0/0.17
  GigabitEthernet0/0.17 is up (if_number 8)
    IP unicast RPF check is enabled
    Input features: uRPF
  ```
* `show ip traffic | include RPF`: システム全体のuRPFドロップカウンターを確認。
  ```text
  R1# show ip traffic | include RPF
    56 unicast RPF drop, 0 suppressed RPF drop
  ```

#### (2) ASA / FTD 検証コマンド
* `show ip verify statistics`: ASA/FTDにおけるインターフェイス毎のuRPFドロップ統計を表示。
  ```text
  ASA1# show ip verify statistics
  interface OUTSIDE: 142 unicast rpf drops
  interface INSIDE: 0 unicast rpf drops
  ```
* `packet-tracer input <if> ip <src> <dst>`: ASA/FTDのパケットチェイサーでuRPFフェーズ（`UNICAST-RPF`）の判定ロジックを追跡。

---



## 🛠 設定方法

Cisco IOS/IOS-XE、Cisco ASA、および Cisco FTD (FMC) における uRPF の具体的な設定手順を示す。

### 1. Cisco IOS / IOS-XE における設定 (CLI)

#### (1) CEFのグローバル有効化（前提条件）
```bash
configure terminal
 ip cef
 ipv6 unicast-routing
 ipv6 cef
end
```

#### (2) Strict Mode（厳格モード）の設定
入力インターフェイスとFIBの出力を完全一致させる。
```bash
configure terminal
 interface GigabitEthernet0/0/1
  description -- Ingress Edge Interface --
  ip verify unicast source reachable-via rx
  ipv6 verify unicast source reachable-via rx
end
```

#### (3) Loose Mode（ルーズモード）＋ デフォルトルート許可 ＋ セルフピン許可
マルチホーム接続やBGP RTBH連携用。
```bash
configure terminal
 interface GigabitEthernet0/0/2
  description -- Multi-Homed Uplink Interface --
  ip verify unicast source reachable-via any allow-default allow-self-ping
  ipv6 verify unicast source reachable-via any allow-default allow-self-ping
end
```

#### (4) uRPF 例外ACL（Pass-Through / Logging）の適用
uRPF違反のうち、特定サブネットをログ記録しながら通過を許可し、それ以外をドロップする。
```bash
configure terminal
 ! ACL定義: permit = uRPF違反でも通過許可, deny = uRPF違反としてドロップ
 ip access-list extended ACL_URPF_EXCEPTION
  permit ip 172.16.100.0 0.0.0.255 any log-input
  deny ip any any log-input
 !
 interface GigabitEthernet0/0/1
  ip verify unicast source reachable-via rx ACL_URPF_EXCEPTION
end
```

---

### 2. Cisco ASA (Adaptive Security Appliance) における設定 (CLI)

ASAではインターフェイス単位で Anti-Spoofing（uRPF）を有効化する。ASAのuRPFはデフォルトで到達可能性チェックを行い、接続セッションのステートフルテーブルと統合される。

```bash
configure terminal
 ! インターフェイス毎に uRPF (Anti-Spoofing) を有効化
 ip verify reverse-path interface OUTSIDE
 ip verify reverse-path interface DMZ
end
```

---

### 3. Cisco FTD (Firepower Threat Defense) における設定 (FMC GUI)

FMC (Firepower Management Center) から FTD の Anti-Spoofing (uRPF) を設定する手順:

1. FMC GUIにログインし、**Devices** -> **Device Management** に移動。
2. 対象の FTD デバイスの **Edit (鉛筆アイコン)** をクリック。
3. **Interfaces** タブから、uRPFを有効化したいインターフェイス（例: `outside`）を選択して **Edit** をクリック。
4. **Advanced** タブをクリックし、内部の **Security Configuration** サブタブを開く。
5. **Anti-Spoofing** チェックボックスをオン（Check）にする。
6. **OK** をクリックしてインターフェイス設定ダイアログを閉じる。
7. FMC画面右上から **Save** をクリックし、**Deploy** ボタンを押して FTD へポリシーをデプロイする。

---

## 🔍 検証コマンド

uRPFの動作状態確認およびトラブルシューティングに使用するコマンド一覧。

| 目的 | Cisco IOS / IOS-XE コマンド | Cisco ASA / FTD (LINA) コマンド |
| --- | --- | --- |
| **uRPF設定・統計の確認** | <code>show ip interface &lt;interface&gt; &#124; include IP verify</code> | <code>show ip verify statistics</code> |
| **CEFレベルのuRPF有効化確認** | <code>show cef interface &lt;interface&gt;</code> | N/A (ASAは常に有効) |
| **システム全体でのドロップ数確認** | <code>show ip traffic &#124; include RPF</code> | <code>show asp drop frame rpf-drop</code> |
| **IPv6 uRPF統計確認** | <code>show ipv6 interface &lt;interface&gt; &#124; begin Unicast RPF</code> | <code>show ipv6 verify statistics</code> |
| **送信元IPのFIBエントリ確認** | <code>show ip cef &lt;source-ip&gt;</code> | <code>show route &lt;source-ip&gt;</code> |
| **詳細デバッグ** | <code>debug ip cef drop</code> / <code>debug platform condition</code> | <code>packet-tracer input &lt;if&gt; ip &lt;src&gt; &lt;dst&gt;</code> |

---

## 🚨 トラブルシュート

uRPF導入時によく遭遇する障害原因と確認コマンド、対処方法のまとめ。

| 症状 | 考えられる原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| **正当な通信が突然破棄される** | 非対称ルーティング（Asymmetric Routing）環境で Strict Mode (`rx`) を使用している | <code>show ip interface &lt;if&gt;</code><br><code>show ip cef &lt;src-ip&gt;</code> | Loose Mode (`reachable-via any`) に切り替えるか、戻り経路のルーティングを対称化する |
| **デフォルトルート宛の通信が拒否される** | デフォルトルート依存の構成で `allow-default` オプションが不足している | <code>show ip interface &lt;if&gt;</code> | `ip verify unicast source reachable-via rx allow-default` を設定する |
| **自ルーター宛のセルフピンが疎通不可** | 送信元IPが自ルーターIPのパケットが Strict Mode により拒否される | <code>ping &lt;self-ip&gt;</code> | `allow-self-ping` オプションを追加する |
| **PBR適用トラフィックが破棄される** | PBRで迂回しているが、FIB（ルーティングテーブル）に入力IFに対応するルートがない | <code>show ip policy</code><br><code>show ip route &lt;src-ip&gt;</code> | PBR適用インターフェイスでは uRPF を無効化するか Loose Mode に変更する |
| **例外ACLを設定したのにドロップされる** | ACLの `permit` / `deny` の定義を誤解し、`deny` で設定してしまっている | <code>show ip access-lists &lt;acl&gt;</code> | 例外として通過させたいトラフィックは ACL で `permit` に修正する |
| **BGP RTBHで攻撃パケットが破棄されない** | Loose Mode を設定しているが、Blackholeルートが `Null0` ではなく他インターフェイスを指している | <code>show ip cef &lt;attack-ip&gt;</code> | RTBH広報のネクストホップが `Null0` に静的・動的に解決されているか確認する |

---

## ⚠ 制限事項

1. **CEF（Cisco Express Forwarding）依存性**:
   * Cisco IOS/IOS-XEにおいて、uRPFはCEFのFIBテーブル構造に全面的に依存している。CEFが無効化されているインターフェイスやプラットフォームではuRPFは機能しない。
2. **Policy-Based Routing (PBR) との非互換性**:
   * PBRは通常のFIB検索をバイパスしてポリシーベースで出力インターフェイスやネクストホップを変更するが、uRPFの検証エンジンはPBRのルートマップを参照せず、常に標準のFIBテーブルを参照する。このため、PBRで正常に転送されるはずのパケットがuRPFによって誤ってドロップされる。
3. **マルチホーミングにおける Strict Mode の制約**:
   * 複数のISPやアップリンクが存在し、ロードバランシングや非対称ルーティングが発生する環境では、Strict Mode (`rx`) の適用は不可能である。
4. **マルチキャスト/ブロードキャストトラフィック**:
   * uRPFはユニキャストIPパケット（Unicast IP Traffic）専用の保護機構であり、マルチキャストパケット（224.0.0.0/4）やブロードキャストパケットの保護には利用できない（マルチキャストには別途 MRoute RPF チェックが使用される）。
5. **トンネル（GRE/IPsec/VTI）における設計制約**:
   * トンネルインターフェイス上でuRPFを有効化する場合、アンダーレイ（Outer Header）とオーバーレイ（Inner Header）のどちらのIPアドレスを検証するのかを明確に設計する必要がある。

---



## 🔄 他技術との関連

uRPFは単体で完結する機能ではなく、他のセキュリティ技術やルーティングプロトコルと組み合わせることで真価を発揮する。

* **NAT (Network Address Translation)**:
  * インバウンドNATや動的NATが適用される境界ルーターにおいて、uRPFは **NAT処理の直前（Ingress時）** に実行される。したがって、uRPFが参照する送信元IPアドレスは「変換前（Real Address）」のIPアドレスである点に注意が必要。
* **Access Control Lists (ACLs)**:
  * インバウンドACLとuRPFの評価順序: インバウンドACLが先に評価され、`permit` されたパケットに対してuRPFの検証が行われる。また、uRPFに指定する例外ACL（Pass-Through ACL）と組み合わせることで、特定の管理者用通信やテスト用IPを例外処理できる。
* **BGP RTBH (Remotely Triggered Black Hole)**:
  * DDoS攻撃緩和の標準的アプローチ。トリガーとなるルーターから攻撃元IPアドレス（または攻撃対象IP）宛てのネクストホップを `192.0.2.1` などのダミーIPに設定したBGPルート（Community: `no-export`）を配信し、エッジルーターで `192.0.2.1` を `Null0` に静的ルーティングする。エッジルーターのインバーンドIFで uRPF Loose Mode を有効にしておくことで、攻撃元IPからの通信をインgress段階で自動ドロップ（Source-Based RTBH）できる。
* **VRF-Lite / MPLS VPN (VRF-Aware uRPF)**:
  * VRF（Virtual Routing and Forwarding）環境下では、uRPFは各VRFごとのFIBテーブル（VRF FIB）を参照して検証を行う。`ip verify unicast source reachable-via rx` コマンドは該当インターフェイスが所属するVRFのFIBを自動的に使用する。
* **Zone-Based Policy Firewall (ZBFW)**:
  * IOS/IOS-XEのZBFWと併用する場合、uRPFによる判定はゾーン間インスペクションポリシープラグインの前段で実行される。非対称経路でZBFWのセッション管理とuRPFの両方が影響を受けるため、ゾーン設計とuRPFモードの整合性が必須。
* **DHCP Snooping & Dynamic ARP Inspection (DAI) / IP Source Guard (IPSG)**:
  * L2/L3アクセススイッチにおけるIPスプーフィング対策。IPSGが「レイヤー2アクセスポート（直接管理下）」でのIPスプーフィングを防止するのに対し、uRPFは「レイヤー3境界（非直接管理ネットワークからのルーテッド通信）」でのIPスプーフィングを防止する補完関係にある。

---

## 🧩 比較表

### 1. Strict Mode vs Loose Mode vs VRF-Aware uRPF

| 比較項目 | Strict Mode (`reachable-via rx`) | Loose Mode (`reachable-via any`) | VRF-Aware uRPF |
| --- | --- | --- | --- |
| **検証条件** | 送信元IPがFIBに存在し、かつ **入力IFが出力IFと完全一致** | 送信元IPがFIBに存在（任意のインターフェイス経由で到達可能） | 該当 **VRFのFIB** テーブルを参照して検証 |
| **主な用途** | シングルホーム接続、キャンパスエッジ、固定アグリゲーション | マルチホーム接続、BGP Source-Based RTBH、非対称ルーティング | サービスプロバイダーVPN、マルチテナント環境 |
| **非対称経路対応** | ✕（パケットが誤破棄される） | ◯（到達可能であれば通過） | ◯ (Loose) / ✕ (Strict) |
| **デフォルトルート** | デフォルトで無効（`allow-default` で有効化） | デフォルトで無効（`allow-default` で有効化） | モード設定に従う |
| **Null0ルート動作** | ドロップ | **ドロップ（RTBHの核）** | ドロップ |
| **設定コマンド** | `ip verify unicast source reachable-via rx` | `ip verify unicast source reachable-via any` | VRFインターフェイス上で通常通り設定 |

### 2. uRPF vs Ingress Anti-Spoofing ACL

| 比較項目 | uRPF (Unicast RPF) | Ingress Anti-Spoofing ACL |
| --- | --- | --- |
| **動作メカニズム** | CEFのFIBテーブルを利用した動的逆引き検証 | スタティックに定義されたアクセスリスト（ACE）の照合 |
| **管理負荷** | **極めて低い**（ルーティングプロトコルと自動連動） | **高い**（ネットワーク構成変更のたびにACL更新が必要） |
| **BGP RTBH連携** | **可能**（動的なBlackholeルート配信に自動連動） | 不可能（手動更新が必要） |
| **パフォーマンス** | CEF（ASIC/QFP）ハードウェア処理で高速 | TCAM/ハードウェア処理（大容量ACLはTCAM枯渇リスク） |
| **適用場所** | L3ルーター/FWの境界インターフェイス | L3ルーター/FW、またはL2/L3スイッチの入力ポート |

---

## 💡 ベストプラクティス

1. **前提条件の確認**:
   * IOS/IOS-XEでは必ず `ip cef`（IPv6の場合は `ipv6 cef`）が有効になっていることを事前に確認する。
2. **適切なモードの選択**:
   * アクセス層・シングルホーム拠点エッジ: **Strict Mode (`rx`)** を適用し、偽装パケットの侵入を水際で阻止。
   * マルチホームISP接続・BGPピアリング境界: **Loose Mode (`any`)** を適用し、非対称ルーティングによる障害を回避しつつ、BGP RTBHによる攻撃遮断を可能にする。
3. **デフォルトルート環境での `allow-default` の明記**:
   * インターネット向けデフォルトルート（0.0.0.0/0）のみを保持するスタブ拠点ルーターで Strict Mode を使用する場合は、必ず `allow-default` オプションを付与する。
4. **管理用セルフ通信の保護 (`allow-self-ping`)**:
   * ルーター自身が送信元となる監視通信やセカンダリIP宛の通信でドロップが発生しないよう、`allow-self-ping` オプションを標準的に有効化する。
5. **例外ACL（Pass-Through ACL）による事前安全策**:
   * 既存運用中のネットワークにuRPFを導入する際は、最初に例外ACLで `permit ip any any log-input` を定義した pass-through モードで適用し、 syslog ログを観察して誤検知がないか事前検証する。

---



## 📝 ラボ学習・設定サンプル例

CCIE Security ラボ試験レベルを意識した 10 個の実践的な演習シナリオ。

---

### サンプル 1: シングルホームエッジにおける Basic Strict Mode uRPF

* **問題**: R1のGigabitEthernet0/0/1はシングルホームで上流ISPに接続されている。送信元アドレスが偽装されたパケットの侵入を防止せよ。
* **要件**:
  1. GigabitEthernet0/0/1 上で Strict Mode の uRPF を有効化すること。
  2. ルーターの Loopback0（150.1.1.1）を送信元とするセルフピング通信がドロップされないようにすること。
* **設定例**:
  ```bash
  R1(config)# ip cef
  R1(config)# interface GigabitEthernet0/0/1
  R1(config-if)# ip verify unicast source reachable-via rx allow-self-ping
  ```
* **検証方法**:
  ```bash
  R1# show ip interface GigabitEthernet0/0/1 | include IP verify
  ! 対向ルーターから偽装IP（例: R1のFIBでGi0/0/1以外を指すIP）でPingを送信し、ドロップカウントが増加することを確認
  R1# show ip traffic | include RPF
  ```

---

### サンプル 2: マルチホームBGP接続における Loose Mode uRPF

* **問題**: R2は2社のアクティブ/アクティブISP（Gi0/1, Gi0/2）に接続されており、非対称ルーティングが発生する。
* **要件**:
  1. 両インターフェイスで非対称通信を阻害しない uRPF を有効化すること。
  2. デフォルトルート経由の到達性も有効な経路として認めること。
* **設定例**:
  ```bash
  R2(config)# ip cef
  R2(config)# interface range GigabitEthernet0/1 - 2
  R2(config-if)# ip verify unicast source reachable-via any allow-default
  ```
* **検証方法**:
  ```bash
  R2# show ip interface GigabitEthernet0/1 | include IP verify
  R2# show cef interface GigabitEthernet0/1
  ```

---

### サンプル 3: BGP Source-Based RTBH (Remotely Triggered Black Hole) 連携

* **問題**: R3（エッジルーター）において、BGPで広報された攻撃元IPアドレス（192.0.2.100）からのトラフィックを自動的に Null0 へドロップせよ。
* **要件**:
  1. インターフェイス Gi0/0 上で uRPF Loose Mode を有効化すること。
  2. 攻撃元IP（192.0.2.100/32）宛てのスタティックルートを Null0 宛てで作成すること。
* **設定例**:
  ```bash
  R3(config)# ip cef
  R3(config)# ip route 192.0.2.100 255.255.255.255 Null0
  R3(config)# interface GigabitEthernet0/0
  R3(config-if)# ip verify unicast source reachable-via any
  ```
* **検証方法**:
  ```bash
  R3# show ip cef 192.0.2.100
  ! 192.0.2.100/32 nexthop Null0
  ! 対向から 192.0.2.100 を送信元IPとして通信を発生させ、Gi0/0でドロップされることを確認
  R3# show ip interface GigabitEthernet0/0 | include verification drops
  ```

---

### サンプル 4: uRPF 例外ACL（Pass-Through / Logging）による監査

* **問題**: R4のGi0/1でuRPF Strict Modeを有効化するが、特定サブネット（172.16.50.0/24）からのスプーフィング違反パケットは破棄せず、ログに記録して通過させよ。
* **要件**:
  1. 172.16.50.0/24 からの uRPF 違反はログ（`log-input`）を取得して許可すること。
  2. その他の uRPF 違反パケットは即座にドロップすること。
* **設定例**:
  ```bash
  R4(config)# ip access-list extended ACL_URPF_EXCEPT
  R4(config-ext-nacl)# permit ip 172.16.50.0 0.0.0.255 any log-input
  R4(config-ext-nacl)# deny ip any any log-input
  R4(config-ext-nacl)# exit
  !
  R4(config)# interface GigabitEthernet0/1
  R4(config-if)# ip verify unicast source reachable-via rx ACL_URPF_EXCEPT
  ```
* **検証方法**:
  ```bash
  R4# show ip access-lists ACL_URPF_EXCEPT
  R4# show logging | include SEC-6-IPACCESSLOG
  ```

---

### サンプル 5: IPv6 uRPF Strict Mode の設定

* **問題**: R5のDual-Stack環境において、IPv6トラフィックに対して Strict Mode uRPF を構築せよ。
* **要件**:
  1. IPv6 CEF を有効化すること。
  2. インターフェイス Gi0/0/0 上で Strict Mode IPv6 uRPF を設定し、デフォルトルートを有効とみなすこと。
* **設定例**:
  ```bash
  R5(config)# ipv6 unicast-routing
  R5(config)# ipv6 cef
  !
  R5(config)# interface GigabitEthernet0/0/0
  R5(config-if)# ipv6 verify unicast source reachable-via rx allow-default
  ```
* **検証方法**:
  ```bash
  R5# show ipv6 interface GigabitEthernet0/0/0 | begin Unicast RPF
  R5# show ipv6 traffic | include RPF
  ```

---

### サンプル 6: VRF-Aware uRPF の構築

* **問題**: R6上の VRF `CUSTOMER_A` に属するインターフェイス Gi0/0/1.100 において、VRFごとの経路表を参照する uRPF を有効化せよ。
* **要件**:
  1. VRF `CUSTOMER_A` のFIBテーブルを用いて Strict Check を行うこと。
* **設定例**:
  ```bash
  R6(config)# vrf definition CUSTOMER_A
  R6(config-vrf)# address-family ipv4
  R6(config-vrf)# exit-address-family
  !
  R6(config)# interface GigabitEthernet0/0/1.100
  R6(config-subif)# vrf forwarding CUSTOMER_A
  R6(config-subif)# ip address 10.100.1.1 255.255.255.0
  R6(config-subif)# ip verify unicast source reachable-via rx
  ```
* **検証方法**:
  ```bash
  R6# show ip cef vrf CUSTOMER_A
  R6# show ip interface GigabitEthernet0/0/1.100 | include IP verify
  ```

---

### サンプル 7: Cisco ASA における Anti-Spoofing (uRPF) 設定

* **問題**: ASA1の `OUTSIDE` および `DMZ` インターフェイス上で Anti-Spoofing (uRPF) を有効化せよ。
* **要件**:
  1. ASAのセキュリティポリシーに従い、ルーティングテーブルに存在しない送信元IPからのパケットをドロップすること。
* **設定例**:
  ```bash
  ASA1(config)# ip verify reverse-path interface OUTSIDE
  ASA1(config)# ip verify reverse-path interface DMZ
  ```
* **検証方法**:
  ```bash
  ASA1# show ip verify statistics
  ASA1# show asp drop frame rpf-drop
  ```

---

### サンプル 8: Cisco FTD / FMC での Anti-Spoofing 設定

* **問題**: FMC管理下の FTD デバイスにおいて、`outside` インターフェイスの Anti-Spoofing 機能を有効化せよ。
* **要件**:
  1. FMC GUI を使用して設定を行い、FTD へポリシーをデプロイすること。
* **設定方法手順（FMC GUI）**:
  1. **Devices** -> **Device Management** -> FTDデバイスの **Edit**。
  2. **Interfaces** タブ -> `outside` インターフェイスの **Edit**。
  3. **Advanced** タブ -> **Security Configuration**。
  4. **Anti-Spoofing** チェックボックスをオンにする。
  5. **Save** 後、**Deploy** を実行。
* **検証方法（FTD LINA CLI）**:
  ```bash
  > system support diagnostic-cli
  FTD1# show ip verify statistics
  ```

---

### サンプル 9: GRE トンネル環境における uRPF 設定

* **問題**: R7とR8の間で GRE トンネル（Tunnel0）が運用されている。Tunnel0 上で uRPF を有効化せよ。
* **要件**:
  1. トンネル内を通過するオーバーレイIPパケットに対して Loose Mode uRPF を適用すること。
* **設定例**:
  ```bash
  R7(config)# interface Tunnel0
  R7(config-if)# ip address 10.255.255.1 255.255.255.252
  R7(config-if)# tunnel source GigabitEthernet0/0
  R7(config-if)# tunnel destination 190.1.1.2
  R7(config-if)# ip verify unicast source reachable-via any
  ```
* **検証方法**:
  ```bash
  R7# show ip interface Tunnel0 | include IP verify
  ```

---

### サンプル 10: SNMP トラッチ / ドロップレート通知連携

* **問題**: uRPFでのパケットドロップ発生時に、SNMPトラップを発行して監視サーバーへ通知せよ。
* **要件**:
  1. uRPFドロップレートが 10 pps を超えた場合に SNMP トラップを発行すること。
* **設定例**:
  ```bash
  R10(config)# snmp-server enable traps ip verify drop-rate
  R10(config)# ip verify unicast notification threshold 10
  R10(config)# interface GigabitEthernet0/0
  R10(config-if)# ip verify unicast source reachable-via rx
  ```
* **検証方法**:
  ```bash
  R10# show ip interface GigabitEthernet0/0 | include drop-rate
  ```

---



## ❓ 想定試験問題

CCIE Security ラボ試験・筆記試験を意識した実践的な問題と詳細な解説。

---

### 問題 1（コンフィグ読解・トラブルシューティング）

**【問題】**
エンジニアがルーター R1 の WAN インターフェイス（Gi0/0/1）に以下の設定を追加した直後、特定の外部拠点からのデータ通信が疎通不能になった。ただし、R1 のルーティングテーブルには宛先ネットワークへのルートが存在している。

```text
R1(config)# interface GigabitEthernet0/0/1
R1(config-if)# ip verify unicast source reachable-via rx
```

通信が破棄されるネットワーク構成上の理由として最も可能性が高いものを1つ選べ。

A. R1で CEF（Cisco Express Forwarding）がグローバルで無効化されているため。  
B. 上流のネットワークで非対称ルーティング（Asymmetric Routing）が発生しており、受信ポートと送信元IPへのFIB出力ポートが一致しないため。  
C. インターフェイスにインバウンドACLが設定されていないため。  
D. コマンドに `allow-default` が含まれていないため、すべてのマルチキャストトラフィックがドロップされたため。

**【正解】** **B**

**【解説】**
`ip verify unicast source reachable-via rx` は **Strict Mode** の uRPF である。Strict Modeでは、パケットが受信されたインターフェイス（Gi0/0/1）と、そのパケットの送信元IPアドレスに対してFIBが示す出力インターフェイスが完全一致している必要がある。非対称ルーティング環境では、パケットの行きと帰りで経路が異なるため、FIBの検索結果が別のインターフェイス（例: Gi0/0/2）を指すことになり、uRPF Strict Check に失敗してパケットが破棄される。

---

### 問題 2（実装・設計）

**【問題】**
企業ネットワークのエッジルーター（R2）において、BGP Remotely Triggered Black Hole (RTBH) を使用してパケット遮断を行う設計を行っている。BGPによりネクストホップ `Null0` 宛ての /32 ルートが R2 に広報されている。
エッジルーターの WAN インターフェイスで特定IPからの送信トラフィックをインバウンド時に即座にドロップするために設定すべき適切なコマンドはどれか。

A. `ip verify unicast source reachable-via rx`  
B. `ip verify unicast source reachable-via any`  
C. `ip verify unicast source reachable-via rx allow-default`  
D. `ip verify reverse-path null0-only`

**【正解】** **B**

**【解説】**
BGP Source-Based RTBH では、対象IPアドレスに対するルートを `Null0` に向けてFIBに登録する。uRPF **Loose Mode** (`reachable-via any`) を有効にしたインターフェイスでは、受信したパケットの送信元IPをFIB検索した際、ルートが `Null0` を指している場合にパケットを即座にドロップする。Loose Modeは入力インターフェイスの一致を要求しないため、マルチホーム環境や非対称環境のBGP RTBH設計に必須となる。

---

### 問題 3（コンフィグ読解・例外ACL）

**【問題】**
以下の uRPF 設定に関する記述のうち、正しいものを1つ選べ。

```text
ip access-list extended ACL_URPF
 permit ip 10.1.1.0 0.0.0.255 any
 deny ip any any

interface GigabitEthernet0/1
 ip verify unicast source reachable-via rx ACL_URPF
```

A. 送信元IPが `10.1.1.0/24` であるパケットのみが uRPF チェックの対象となり、それ以外のパケットはuRPFチェックをパスする。  
B. 送信元IPが `10.1.1.0/24` であるパケットが uRPF Strict Check に違反した場合、ドロップされずに通過が許可される。  
C. 送信元IPが `10.1.1.0/24` 以外のパケットは、uRPF違反がなくてもすべてアクセスリストによりドロップされる。  
D. この設定により、10.1.1.0/24 からの通信のみがインバウンドACLとして許可される。

**【正解】** **B**

**【解説】**
uRPFコマンドに付与するアクセスリスト（例外ACL / Pass-Through ACL）において、`permit` エントリは **「uRPF検証に違反していても、例外としてドロップせず通過を許可する」** ことを意味する。逆に `deny` エントリは **「uRPF検証違反パケットを通常通りドロップする」** ことを意味する。したがって、`permit ip 10.1.1.0 0.0.0.255 any` は 10.1.1.0/24 からのuRPF違反パケットの通過を例外的に許可する設定である。

---

### 問題 4（デザイン・制約事項）

**【問題】**
Cisco IOS-XE ルーターにおいて、Policy-Based Routing (PBR) と uRPF を同一インターフェイス上で同時に有効化した。この構成における挙動として正しいものはどれか。

A. PBRが優先され、uRPFの検証エンジンは PBR の route-map で指定されたネクストホップを参照して送信元IPを検証する。  
B. uRPFは PBR の設定を無視し、常に標準の FIB（ルーティングテーブル）を参照して送信元IPを検証する。  
C. PBRとuRPFは完全に排他であり、PBRが有効なインターフェイスではuRPFは自動的に無効化される。  
D. PBRによってリダイレクトされたパケットは、uRPFの処理対象から自動的に除外される。

**【正解】** **B**

**【解説】**
uRPF検証エンジンは、PBR（Policy-Based Routing）で定義されたルートマップやネクストホップ変更を一切参照せず、常にCEFの標準FIB（Forwarding Information Base）を参照する。このため、PBRによって通常とは異なる経路で送受信されるトラフィックは、FIBとの不一致によりuRPF Strict Modeによって誤ってドロップされる危険性がある。PBR構成ではuRPFの無効化または設計の見直しが必要となる。

---

### 問題 5（ASA/FTD 実装）

**【問題】**
Cisco ASA または FTD (Firepower Threat Defense) における Anti-Spoofing (uRPF) の動作に関する記述として、誤っているものを1つ選べ。

A. ASA/FTD の uRPF は、接続（TCP/UDPセッション）の初回パケットに対してのみルート検証を行い、セッション確立後はステートテーブルを参照して連続パケットを検証する。  
B. ICMPなどのステートレスなプロトコルに対しては、すべてのパケットに対して逆引きルート検証が実行される。  
C. ASA/FTD では、デフォルトルート（0.0.0.0/0）が存在する場合、未知の外部送信元IPからのパケットは Outside インターフェイスからの入力として正常と判定される。  
D. ASA/FTD の Anti-Spoofing 機能は `ip verify unicast source reachable-via rx` コマンドで設定する。

**【正解】** **D**

**【解説】**
Cisco ASA では `ip verify reverse-path interface <interface_name>` コマンドを使用して Anti-Spoofing（uRPF）を有効化する。`ip verify unicast source reachable-via rx` は Cisco IOS / IOS-XE の構文であるため、D が誤り（正解）。A, B, C の記述は ASA/FTD における uRPF の正しい動作仕様である。

---

## 🔗 参考リソース

Cisco公式ドキュメントおよびテクニカルガイドへの参照リンク集。

* Cisco Live: Network Security and Anti-Spoofing with uRPF
* Cisco IOS XE Security Configuration Guide: Unicast Reverse Path Forwarding
* Cisco Command Reference: ip verify unicast source reachable-via
* Cisco ASA Series General Operations Configuration Guide: Anti-Spoofing (uRPF)
* Cisco Secure Firewall Management Center Configuration Guide: Anti-Spoofing
* Cisco Technical Note: Understanding Unicast Reverse Path Forwarding (uRPF)
* Cisco Validated Design (CVD): Remotely Triggered Black Hole (RTBH) Filtering with uRPF

---

## 📝 補足（Notes）

### uRPF の処理ロジックとトラフィック分類メモ

1. **CEF（FIB）テーブルの重要性**:
   * uRPFはパケットが届いた瞬間にCPUでルーティングテーブルを検索するのではなく、ASIC/QFPに同期されたFIBテーブルを即座に参照する。このため、CEFが無効な状態や分散スイッチング（dCEF）の同期遅延が発生している環境では誤動作を起こす。
2. **`allow-default` オプションの動作補足**:
   * `allow-default` が指定されている場合、送信元IPアドレスがFIB内の特定の個別のルート（例: `192.168.1.0/24`）にマッチしなくても、デフォルトルート（`0.0.0.0/0`）にマッチすれば「到達可能」と判断される。インターネット向け集約エッジで非常に重要。
3. **`allow-self-ping` オプションの動作補足**:
   * ルーターが自らのインターフェイスIP宛てにICMP Echoを発行した際、送信元IPと宛先IPが共に自ルーターのIPとなる。uRPF Strict Checkが有効なインターフェイスからそのパケットが戻ってきた場合、自分自身のIPのFIB出力を照合すると不一致となりドロップされる事象を防ぐために用意されている。

---


