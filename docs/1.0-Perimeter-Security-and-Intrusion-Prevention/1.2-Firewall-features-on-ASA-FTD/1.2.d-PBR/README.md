---
layout: default
title: 1.2.d-PBR
nav_order: 4
parent: 1.2-Firewall-features-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.2.d Policy-based routing (PBR)

Cisco ASAおよびFirepower Threat Defense (FTD) における**ポリシーベースルーティング (PBR)** は、通常の宛先IPアドレスのみに基づくルーティングを上書きし、送信元IP、プロトコル、サービスポートなどの任意の基準に基づいてパケットを転送する機能です。CCIE Securityラボ試験では、マルチホーミング環境におけるトラフィックの柔軟な振り分けや、特定のアプリケーションを専用の回線へ誘導する実装力が問われます。

---

## 📘 概要

*   **機能概要**: ルーティングテーブル (RIB) の決定よりも優先して、管理者が定義したポリシー（ルートマップ）に基づきネクストホップを決定します。
*   **利用目的**:
    *   **パスの最適化**: 重要なトラフィック（音声など）を低遅延回線へ、一般トラフィックを安価な回線へ振り分ける。
    *   **ISPの選択**: 送信元ネットワークごとに異なるISPを強制する。
    *   **セキュリティの強化**: 特定のトラフィックのみインスペクションデバイスやプロキシへ誘導する。
*   **展開場面**: 複数のインターネット接続 (Dual-ISP) を持つ拠点や、SD-WAN的なトラフィック制御をファイアウォール単体で実現したい場合に利用されます。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **特徴** | L3/L4情報（送信元/宛先IP、ポート番号、プロトコル）に基づいて経路を決定する。 |
| **用途** | トラフィックシェアリング、特定アプリの優先、ISP分散。 |
| **メリット** | 柔軟なトラフィック制御。宛先IPに依存しない動的な経路制御。 |
| **デメリット** | 設定の複雑化。トラブルシューティング時の難易度向上。 |
| **対応機種** | ASA 9.4(1)以降の全モデル、FTD 6.2.1以降（FMC管理）。 |
| **制限事項** | 自己発のトラフィックには適用不可。インバウンドインターフェイスに適用される。 |
| **設計上の注意点** | **IP SLA (Track)** を組み合わせないと、ネクストホップがダウンした際にパケットがドロップされる。 |

---

## 🏗 動作原理

PBRは「パケットが入ってくるインターフェイス」に適用されます。ASA/FTDはパケットを受信すると、通常のルーティングルックアップを行う前にポリシーに合致するかを確認します。

```text
[Traffic Source]
   ↓
[Ingress Interface] (PBR Applied)
   ↓
[ACL Match] (送信元/宛先/ポートのチェック)
   ↓
[Route-map Evaluation]
   ├─ Match: set ip next-hop 経由で転送
   └─ No Match: 通常の Routing Table (RIB) で転送
```

---

## ⚙ 動作シーケンス

1.  **パケット着信**: 物理またはサブインターフェイスでパケットを受信します。
2.  **PBRポリシー参照**: インターフェイスに `policy-route route-map` が設定されているか確認します。
3.  **マッチング評価**: 設定された ACL に基づき、パケットを評価します。
4.  **ネクストホップ決定**: 
    *   マッチした場合：`set ip next-hop` で指定されたアドレスへ転送します（IP SLAで生存確認されている場合のみ有効）。
    *   マッチしない場合：通常の宛先ベースルーティングに従います。
5.  **フォールバック**: もし `set ip next-hop` がダウンしており、かつデフォルトの振る舞いが定義されていない場合、パケットはRIBに戻されます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **IP SLAとの統合**: ラボでは「ISPがダウンしたら別の経路に切り替えよ」という要件がセットになります。`track` オブジェクトと PBR を連動させる構成が必須です。
*   **宛先ポートベースの分岐**: 「HTTP(80)はISP-Aへ、それ以外はISP-Bへ」といった、サービスを識別したルーティングが求められます。

### ラボ試験で設定させられそうな内容
*   **複数ネクストホップの優先順位**: `set ip next-hop 1.1.1.1 2.2.2.2` のように複数のホップを並べ、優先順位を制御する。
*   **FMCでのPBR構成**: GUI（Devices > Routing > PBR）を使用した、オブジェクトベースの設定手順。
*   **再帰的次ホップ (Recursive Next-hop)**: 直接接続されていないIPをネクストホップにする構成。

### よくある設定ミス
*   **ACLの送信元/宛先を逆に設定**: PBRは「受信」インターフェイスでかかるため、送信元IPがそのセグメントのものである必要があります。
*   **IP SLAの関連付け忘れ**: トラック番号を指定しないと、ゲートウェイが死んでいるのにパケットを送り続け、ブラックホール化します。

### showコマンドから状態を判断
*   `show route-map`: ポリシーがどれだけパケットを転送したか（カウンタ）を確認。
*   `packet-tracer`: **PBRフェーズ**が表示され、どのルートマップに一致したかが明示されます。

---

## 🛠 設定方法

### ASA (CLI) - PBR の設定手順

1.  **トラフィックを特定する ACL の作成**:
    ```bash
    access-list PBR_HTTP extended permit tcp 192.168.1.0 255.255.255.0 any eq 80
    ```
2.  **ルートマップの定義**:
    ```bash
    route-map MY_PBR permit 10
     match ip address PBR_HTTP
     set ip next-hop 203.0.113.254  # ISP-A のゲートウェイ
    ```
3.  **インターフェイスへの適用**:
    ```bash
    interface GigabitEthernet0/1
     nameif inside
     policy-route route-map MY_PBR
    ```

### FTD (FMC GUI)
1.  **Devices > Device Management**: 対象FTDを編集。
2.  **Routing > Policy Based Routing**: 「Add」をクリック。
3.  **Ingress Interface**: `Inside_Zone` などを選択。
4.  **Forwarding Actions**: 
    *   **Match**: ACL (Extended Access List) オブジェクトを指定。
    *   **Send To**: Next-hop IP または Interface を指定。
    *   **Track**: 監視用の IP SLA を選択。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **ルートマップのステータス** | <code>show route-map</code> |
| **PBRの統計確認** | <code>show policy-route</code> |
| **パケットフローの追跡** | <code>packet-tracer input inside tcp 192.168.1.10 1234 8.8.8.8 80</code> |
| **IP SLAの状態** | <code>show sla monitor operational-state</code> |
| **インターフェイス設定確認** | <code>show run interface [name]</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| 特定トラフィックがPBRされない | ACLのマッチングミス | <code>show access-list</code> でヒットカウントを確認。 |
| PBR設定後、通信が完全に途絶 | ネクストホップが到達不能 | <code>show route</code> でネクストホップへの経路があるか、またはIP SLAの状態を確認。 |
| packet-tracer で PBR が "Allow" だが届かない | 戻りのルートが欠落 | 宛先サーバ側のルーティングを確認。 |
| ルートマップのカウンタが増えない | インターフェイス適用の漏れ | <code>show running-config interface</code> で <code>policy-route</code> を確認。 |
| FTDでPBRが効かない | ポリシーデプロイ未完了 | FMCのデプロイステータスを確認。 |

---

## ⚠ 制限事項

*   **自己発信トラフィック**: デバイス自体（ASA/FTD）から発生するトラフィック（Syslog, SNMPなど）には PBR を適用できません。
*   **透過モード**: 透過モード（Transparent）では PBR はサポートされません。ルーテッドモードが必須です。
*   **マルチキャスト**: マルチキャストトラフィックには適用されません。
*   **NATとの順序**: ASAのパケットフローにおいて、NATの処理順序とPBRがどう干渉するか（通常はNAT Untranslateの後にルーティング/PBRが行われる）を考慮する必要があります。

---

## 🔄 他技術との関連

*   **IP SLA**: ネクストホップの死活監視に必須です。
*   **Access Control Policy**: トラフィックが ACL で許可されていない場合、ルーティング判定の前にパケットが破棄されます。
*   **VPN**: VPN経由のトラフィックを特定の物理パスへ誘導する際に PBR が使用されることがあります。
*   **QoS**: PBR でトラフィックを分類した後、特定のインターフェイスで優先制御をかける設計が一般的です。

---

## 🧩 比較表

### Standard Routing vs PBR

| 比較項目 | 標準ルーティング (RIB) | ポリシーベースルーティング (PBR) |
| :--- | :--- | :--- |
| **判断基準** | 宛先IPアドレスのみ | 送信元、宛先、プロトコル、ポート等 |
| **優先度** | 低い | **高い（RIBより先に評価）** |
| **柔軟性** | 限定的 | 非常に高い |
| **設定負荷** | 低（Static/Dynamic） | 高（ACL + Route-map） |
| **フォールバック** | なし（デフォルトルート等に依存） | あり（非マッチ時にRIBへ戻る） |

---

## 💡 ベストプラクティス

*   **明示的な ACL の使用**: `any any` を避け、可能な限り具体的なサブネットとポートを指定します。
*   **IP SLA の常時併用**: PBR を構成する際は、必ず `track` を設定してネットワークの可用性を担保します。
*   **デフォルトルートの維持**: PBR にマッチしなかった場合のために、常に RIB に有効なデフォルトルートを持たせておきます。
*   **packet-tracer での事前検証**: 本番環境やラボの最終確認では、必ず `packet-tracer` で PBR lookup が成功しているかログを確認します。

---

## 📝 ラボ学習・設定サンプル例

### 1. 送信元IPによるISPの強制
*   **要件**: 192.168.10.0/24のパケットのみ、ゲートウェイ 10.1.1.254 へ送信せよ。
*   **設定**: `access-list PBR permit ip 192.168.10.0 255.255.255.0 any` -> `route-map RM set ip next-hop 10.1.1.254`

### 2. ポートベースの振り分け
*   **要件**: HTTPS (443) 通信をバックアップ回線 (10.2.2.254) へ誘導せよ。
*   **設定**: ACLで `permit tcp any any eq 443` を作成。

### 3. IP SLA トラッキング付き PBR
*   **要件**: ネクストホップ 1.1.1.1 が死んだら PBR を無効化せよ。
*   **設定**:
```bash
sla monitor 1
 type echo protocol icmpService 1.1.1.1
track 1 rtr 1 reachability
route-map PBR-MAP
 set ip next-hop 1.1.1.1 track 1
```

### 4. 複数ネクストホップの冗長化
*   **要件**: 優先ホップ 10.1.1.1、バックアップ 10.1.1.2 を指定せよ。
*   **設定**: `set ip next-hop 10.1.1.1 10.1.1.2`

### 5. FTD(FMC)でのPBR設定
*   **要件**: Insideゾーン発の特定トラフィックにPBRを適用。
*   **手順**: FMCの Routing > PBR からインターフェイス `inside` を選択し、ACLとネクストホップを紐付ける。

### 6. IPv6 PBR (ASA)
*   **要件**: IPv6 送信元に基づくルーティングを構成。
*   **設定**: `access-list PBR_V6 extended permit ip6 [subnet] any` を使用。

### 7. PBR と NAT の共存
*   **注意**: 内部IPをNATする前にPBRが評価されるため、ACLには「変換前」のプライベートIPを記述する必要があります。

### 8. ルートマップ内での複数 Match
*   **要件**: 送信元IP 10.1.1.0 かつ、宛先ポート 80 に合致するもののみを PBR せよ。
*   **設定**: ACLで両方の条件を 1 つの `permit` 文にまとめる。

### 9. 特定ドメイン（IPオブジェクト）へのPBR
*   **要件**: オブジェクト `obj-Office365` 宛の通信を専用線へ。
*   **設定**: オブジェクトを match 条件に使用。

### 10. packet-tracer による検証
*   **要件**: 設定した PBR が正しくマッチしているか CLI で証明せよ。
*   **コマンド**: `packet-tracer input inside tcp 192.168.1.10 1234 8.8.8.8 80` の出力にある `Phase: PBR-LOOKUP` を確認。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: `route-map` で `set ip next-hop` と `set ip default next-hop` の違いは何か？
    *   **正解**: `set ip next-hop` はルーティングテーブルより先に評価されます。`set ip default next-hop` は、ルーティングテーブルに宛先への経路がない場合にのみ使用されます。
2.  **トラブルシュート**: ASAのPBR設定後、パケットトレーサーでは成功するが、実際のホストからの通信がドロップする。ACLを確認したところ、送信元IPがNAT後のIPで書かれていた。修正方法は？
    *   **正解**: PBRのACLには NAT 前のリアル IP を記述する必要があるため、ACLをプライベートIPに書き換える。
3.  **Design**: 2つのISP接続があり、片方がダウンした際に自動的に全トラフィックを他方に切り替えるPBRを設計せよ。
    *   **正解**: `set ip next-hop [ISP1] track [SLA_ID]` を設定し、フォールバックとして `set ip next-hop [ISP2]` を定義するか、RIBにデフォルトルートを持たせる。
4.  **実装**: FTD FMCにおいて、PBRポリシーを複数のインターフェイスに適用することは可能か？
    *   **正解**: はい、FMC上で複数のインバウンドインターフェイス（またはゾーン）に対して、それぞれルートマップを割り当てることができます。
5.  **動作シーケンス**: パケットがASAに入った際、L2L VPNの復号化とPBRの評価、どちらが先に行われるか？
    *   **正解**: 一般的に、着信インターフェイスでの物理的な終端（VPN復号含む）が先に行われ、その後にルーティング判定（PBR）が行われます。

---

## 🔗 参考リソース

*   **Configuration Guides**:
    *   [Cisco ASA Series Firewall CLI Configuration Guide, 9.4 - Policy Based Routing](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/firewall/asa-94-firewall-config/route-policy-based.html)
    *   [Cisco Firepower Threat Defense Configuration Guide for FMC - PBR](https://www.cisco.com/c/en/us/td/docs/security/firepower/70/configuration/guide/fpmc-config-guide-v70/policy_based_routing.html)
*   **Command Reference**:
    *   [Cisco ASA Series Command Reference - policy-route](https://www.cisco.com/c/en/us/td/docs/security/asa/command-reference/m-p/cmdref2/p2.html#pgfId-1824443)
*   **Technical Notes**:
    *   [ASA: Policy Based Routing Configuration Example](https://www.cisco.com/c/en/us/support/docs/security/asa-5500-x-series-next-generation-firewalls/118944-config-asa-00.html)
    *   [FTD: Policy Based Routing with FMC](https://www.cisco.com/c/en/us/support/docs/security/firepower-ngfw/214343-configure-policy-based-routing-on-firepo.html)
*   **Cisco Live (Slides)**:
    *   [BRKSEC-3020: Troubleshooting Firewall Threat Defense (FTD)](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKSEC-3020)

---

## 📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

