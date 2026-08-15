---
layout: default
title: 4.10-Profiling
nav_order: 10
parent: 4.0-Identity-Management
---

# 4.10 Endpoint profiling using Cisco ISE and Cisco network infrastructure including device sensor

Cisco ISE (Identity Services Engine) の **プロファイリング (Profiling)** 機能は、ネットワークに接続されたデバイスの種類（例：Cisco IP Phone, Apple iPhone, Windows Workstation）を自動的に識別するプロセスです。スイッチや WLC などのネットワークインフラが **Device Sensor** を使用して属性を収集し、ISE がそれらの属性を基にデバイスを分類します。これにより、サプリカントを持たない IoT デバイスに対しても、適切な認可ポリシーを適用することが可能になります。

---

## 📘 概要

*   **機能概要**: デバイスから送信されるネットワークトラフィック（DHCP, HTTP, CDP, LLDP 等）から特徴的な属性（プロファイラプローブ）を抽出し、既知のデバイスプロファイルと照合して識別する機能です。
*   **利用目的**: 不明なデバイス（Shadow IT）の可視化、MAC アドレス偽装の検知、およびデバイスタイプに基づいた動的なアクセス制御の自動化。
*   **どのような場面で利用するか**: 
    *   **IoT デバイスの管理**: 認証機能を持たないプリンタやカメラを MAB 認証後に「プリンタ」として認可し、特定の通信のみ許可する。
    *   **BYOD**: 接続された端末が Android か iOS かを判別し、オンボーディングポータルへリダイレクトする [4.6 参照]。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **プローブ (Probes)** | DHCP, HTTP, RADIUS, SNMP, DNS, NetFlow, NMAP 等のデータ収集ソース。 |
| **Device Sensor** | スイッチ/WLC 上で動作し、CDP/LLDP/DHCP 属性を収集して RADIUS で ISE へ送信する機能。 |
| **Profiling Policy** | 属性の組み合わせに基づく判定ルール。各ルールには「信頼スコア (Certainty Factor)」がある。 |
| **Certainty Factor** | 判定の確信度。複数の属性が一致するほどスコアが加算され、閾値を超えると特定プロファイルに確定。 |
| **CoA (Change of Authorization)** | プロファイルが確定または変更された際、ISE から NAD へセッションの再認可を要求する。 |
| **Endpoint Identity Group** | プロファイリングの結果、デバイスが自動的に割り当てられる論理グループ。 |

---

## 🏗 動作原理

Device Sensor を利用したプロファイリングの通信フロー：

```text
[ Endpoint ]
     ↓ (1) DHCP Request / CDP / LLDP パケットを送信
[ Switch (NAD) ]  <-- Device Sensor が動作
     ↓ (2) 属性を抽出 (TLV, DHCP Options)
     ↓ (3) RADIUS Accounting または専用パケットで ISE へ属性を送信
[ Cisco ISE ]
     ↓ (4) 受信した属性を Profiling Policy と照合
     ↓ (5) デバイスを特定 (例: Cisco-IP-Phone)
     ↓ (6) CoA を送信 (必要に応じて権限を更新)
[ Switch (NAD) ]
     ↓ (7) 新しいポリシー (VLAN/ACL) を適用
[ Endpoint ]
```

---

## ⚙ 動作シーケンス

1.  **データ収集**: デバイスがパケットを送信。NAD は `Device Sensor` を通じて L2/L3 の属性をキャッシュします。
2.  **属性転送**: NAD は収集した情報を RADIUS パケット（通常は Accounting-Request）に含めて ISE に送信します。
3.  **分類 (Classification)**: ISE は受け取った MAC アドレスと属性のリストをデータベースに保存し、プロファイリングポリシーを評価します。
4.  **プロファイル確定**: 属性の一致度（Certainty Factor）が閾値を超えたプロファイルがその MAC アドレスに割り当てられます。
5.  **認可の実行**: デバイスが特定のグループ（例：Cisco-IP-Phone）に分類されたことをトリガーに、ISE は CoA を送り、NAD 上のポート設定を更新します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Device Sensor の有効化**: スイッチ側で `device-sensor notify all-attributes` などの設定が漏れていると、ISE に属性が届かずプロファイリングが「Unknown」のままになります。
*   **IPDT (IP Device Tracking)**: プロファイリングの前提条件として、スイッチで IPDT（Cisco IOS-XE では `device-tracking`）が動作している必要があります。
*   **CoA の構成**: プロファイル変更後に自動でポリシーを反映させるには、認可プロファイルで `Reauth` や `Port Bounce` の CoA 設定が必要です。
*   **SNMP プローブの利用**: ラボ環境で Device Sensor が使えないレガシー機器がある場合、ISE から NAD へ SNMP クエリを投げる構成が問われることがあります。
*   **プロファイル階層の理解**: `Apple-Device` -> `Apple-iPhone` のように親プロファイルからの継承関係を理解し、より具体的なプロファイルで認可するルールを作成します。

---

## 🛠 設定方法

### 1. スイッチ：Device Sensor の基本構成
```bash
! IPデバイス追跡の有効化
device-tracking-policy TRACK-ALL
 tracking enable
!
! Device Sensor の属性フィルタ定義 (CDP/LLDP/DHCP)
device-sensor filter-list cdp list CDP-LIST
 tlv name device-name
 tlv name capabilities-tlv
!
device-sensor filter-list dhcp list DHCP-LIST
 option name host-name
 option name parameter-request-list
!
! 収集した属性を RADIUS Accounting で送信
device-sensor notify all-attributes
!
! インターフェイスへの適用
interface GigabitEthernet1/0/1
 device-tracking attach-policy TRACK-ALL
```

### 2. Cisco ISE 側の構成
1.  **Administration > System > Settings > Profiling**: 各プローブ（RADIUS, DHCP 等）が有効であることを確認。
2.  **Work Centers > Profiler > Profiling Policies**: デバイスを識別する条件と、 Certainty Factor を定義。
3.  **Policy > Authorization**: 条件に `Endpoints:LogicalProfile EQUALS Cisco-IP-Phone` などのプロファイルグループを使用。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **NAD上での属性キャッシュ確認** | <code>show device-sensor cache all</code> |
| **Device Sensor の動作確認** | <code>show device-sensor status</code> |
| **ISE で学習したエンドポイント確認** | **Context Visibility > Endpoints** |
| **ISE 認証・プロファイルログ** | **Operations > RADIUS > Live Logs** |
| **トラッキング対象のIP確認** | <code>show device-tracking database</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| デバイスが "Unknown" のまま | 属性が ISE に届いていない | <code>debug device-sensor</code> で収集を確認。ISE でプローブが有効か確認。 |
| 特定されたが権限が変わらない | CoA が動作していない | スイッチで <code>aaa server radius dynamic-author</code> 設定を確認。 |
| デバイス名が間違って判定される | 信頼スコアの設定が甘い | ISE の Profiling Policy で Certainty Factor の数値を上げる。 |
| IP アドレスが表示されない | DHCP Snooping または IPDT の停止 | <code>show ip dhcp snooping binding</code> 等で IP 学習状態を確認。 |

---

## ⚠ 制限事項

*   **属性の欠落**: デバイスが静的 IP を使用している場合、DHCP プローブから情報を得られないため、識別の精度が下がります。
*   **暗号化通信**: HTTP プローブによる User-Agent 取得は、HTTPS サイトへのアクセスでは（SSL 復号しない限り）困難です。
*   **ライセンス**: 高度なプロファイリング機能には ISE Advantage ライセンスが必要です。

---

## 🔄 他技術との関連

*   **3.4.b IPDT**: デバイスの IP アドレスを維持するために必須。
*   **4.2 Network Access AAA**: プロファイリングの結果に基づき、最終的な認可 ACL や VLAN を決定。
*   **4.4 MAB**: 初期認証として MAB を使い、プロファイリングで正体を突き止めた後に権限を昇格させる。
*   **2.6 TrustSec**: プロファイル（例：IP-Camera）に基づき、特定の SGT を割り当てる。

---

## 🧩 比較表

### Device Sensor vs SNMP Profiling

| 特徴 | Device Sensor (Push) | SNMP Profiling (Pull) |
| :--- | :--- | :--- |
| **動作場所** | スイッチ（NAD） | ISE PSN |
| **負荷** | 低（NAD がイベントを送信） | 高（ISE が全 NAD を巡回） |
| **リアルタイム性** | **高い** | 低い（ポーリング間隔に依存） |
| **設定** | スイッチ側で詳細設定が必要 | ISE 側でコミュニティ名が必要 |
| **推奨** | **Cisco 最新インフラ** | 非 Cisco またはレガシー機器 |

---

## 💡 ベストプラクティス

1.  **段階的な CoA**: 判定が頻繁に変わる環境では、CoA タイプを `Reauth` に設定し、リンク断（Port Bounce）を避けます。
2.  **フィードサービスの利用**: ISE の Profiler Feed Service を有効にし、Cisco が提供する最新のデバイスシグネチャを定期的に更新します。
3.  **RADIUS Accounting**: `aaa accounting update periodic` を設定し、定期的に属性を ISE に同期させます。
4.  **Unknown への対処**: 分類できないデバイスは、まずは隔離 VLAN に入れ、手動でプロファイルを作成する運用フローを構築します。

---

## 📝 ラボ学習・設定サンプル例

### 1. Device Sensor での CDP 属性収集
*   **要件**: スイッチ Gi1/0/1 に接続された IP Phone の CDP 情報を ISE に送信せよ。
*   **設定**: `device-sensor filter-list cdp list L` + `tlv name device-name`.

### 2. DHCP Option 60 による識別
*   **要件**: DHCP Vendor Class ID を基に、特定のプリンタを識別せよ。
*   **ISE設定**: Condition で `DHCP:vendor-class-identifier` を指定。

### 3. IPDT の構成 (Cisco IOS-XE)
*   **要件**: 全ポートで IPv4 デバイストラッキングを有効にせよ。
*   **設定**: `device-tracking-policy P1`, `tracking enable`, `interface range Gi1/0/1-24`, `device-tracking attach-policy P1`.

### 4. Profiling による CoA 設定
*   **要件**: プロファイルが確定したら自動的に再認証を実行せよ。

### 5. HTTP Probe によるブラウザ識別
*   **操作**: Windows PC でブラウザを起動し、ISE が User-Agent を取得することを確認。

### 6. カスタムプロファイリングポリシーの作成
*   **要件**: 特定の MAC OUI (例: 00:50:56) を持つ VM 端末用ポリシーを作成せよ。

### 7. Certainty Factor の調整
*   **問題**: スコアが足りずプロファイルが適用されない。
*   **対処**: 特定条件のスコアを 10 から 60 へ引き上げる。

### 8. NMAP プローブのスキャン実行
*   **要件**: ISE から特定サブネットのデバイスに対し OS 指紋スキャンを実行せよ。

### 9. 属性フィルタの最適化
*   **要件**: 不要な属性（シリアル番号等）を Device Sensor のリストから除外し、トラフィックを削減せよ。

### 10. プロファイリングベースの VLAN 遷移
*   **課題**: MAB 直後は VLAN 10、プロファイル確定後は VLAN 20 へ移動させよ。

---

## ❓ 想定試験問題

1.  **コンフィグ読解**: スイッチで `device-sensor notify all-attributes` が設定されているが、ISE 側で HTTP 属性が見えない。何が不足しているか？
    *   **回答**: デバイスが Web 通信を行っていないか、あるいは **HTTP プローブ** 自体が ISE の Profiler 設定で無効になっている。
2.  **トラブルシュート**: デバイスが `Cisco-Device` までは判定されるが、`Cisco-IP-Phone` まで細分化されない。原因は？
    *   **回答**: より詳細な属性（CDP の device-name 等）が収集できていないか、それらの一致スコアが **確定閾値 (Threshold)** に達していない。
3.  **Design**: プロファイリング環境において、ネットワークへの負荷を最小限に抑えつつリアルタイムに属性を収集する最適なプローブは？
    *   **回答**: **Device Sensor**。NAD がエッジで属性を収集し、既存の RADIUS 認証フローに乗せて ISE へ送るため。
4.  **コンフィグ読解**: `device-tracking attach-policy` がインターフェイスに適用されていない場合の影響は？
    *   **回答**: ISE がエンドポイントの **IP アドレスを把握できず**、属性の紐付けや一部のプローブが正常に動作しない。
5.  **Design**: プロファイリングの結果が不確実な場合に、管理者に通知しつつ一時的なアクセスを許可する機能は？
    *   **回答**: Profiling Policy の **Exception** 設定、または低い信頼スコア時に適用する認可ポリシーの作成。

---

## 🔗 参考リソース

*   **Configuration Guide**: [Cisco ISE 3.1 Profiling Design Guide](https://www.cisco.com/c/en/us/td/docs/security/ise/3-1/admin_guide/b_ise_admin_guide_31.html)
*   **Cisco Live (BRKSEC-3432)**: [Profiling with Cisco ISE and Device Sensor](https://www.ciscolive.com/)
*   **Technical Note**: [Configuring Device Sensor on Cisco Switches](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/116347-device-sensor-ise-00.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: プロファイリングは「指紋捜査」に似ています。落ちている属性（指紋）を拾い集め、ISE の名簿（ポリシー）と照合します。
*   **図解**: `Certainty Factor` が積み木のように積み重なり、一定の高さ（Threshold）に達するとプロファイルという「箱」に入るイメージを持ってください。
*   **注意点**: ラボ試験では `device-sensor notify all-attributes` を忘れがちですが、これを入れないと NAD は属性を ISE へ「通知」しません。
