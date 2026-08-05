---
layout: default
title: 1.2.a-NAT
nav_order: 1
parent: 1.2-Firewall-features-on-ASA-FTD
grand_parent: 1.0-Perimeter-Security-and-Intrusion-Prevention
---

# 1.2.a NAT (Network Address Translation)

Cisco ASAおよびFirepower Threat Defense (FTD) における**ネットワークアドレス変換 (NAT)** は、IPアドレスの枯渇対策だけでなく、ネットワーク境界におけるセキュリティの維持や、異なる組織間でのIP重複（Overlapping）の解決に不可欠な機能です。CCIE Securityラボ試験では、NATの優先順位（NAT Order）の正確な理解と、複雑なポリシーNATの実装能力が問われます。

---

## 📘 概要

*   **機能概要**: パケットがファイアウォールを通過する際に、送信元または宛先のIPアドレス/ポート番号を書き換える機能です。
*   **利用目的**:
    *   **IPv4アドレスの節約**: プライベートIPを1つのパブリックIPに集約（PAT）。
    *   **セキュリティ**: 内部ネットワークのトポロジを外部から隠蔽。
    *   **接続性の確保**: 同一IPレンジを持つネットワーク同士の通信を可能にする（Twice NAT）。
*   **役割**:
    *   **ASA**: LINAエンジンにより高速に処理され、オブジェクトベースの設定（8.3以降）が基本となります。
    *   **FTD**: FMC（Firepower Management Center）からポリシーとして定義され、ASAと同様のLINAエンジン上で動作します。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **NATの種類** | Static NAT, Dynamic NAT, Dynamic PAT, Static PAT |
| **設定手法** | **Object NAT (Auto NAT)** と **Twice NAT (Manual NAT)** |
| **変換対象** | 送信元アドレスのみ、または送信元と宛先の両方（Twice NAT） |
| **DNS Rewrite** | NAT環境でDNS応答内のIPアドレスを書き換える機能 |
| **NAT Order** | Section 1 (Manual NAT) → Section 2 (Auto NAT) → Section 3 (Manual NAT) |

---

## 🏗 動作原理

NATは、パケットのルーティング決定とセキュリティチェックの間に密接に関連して動作します。

```text
[Ingress パケット]
   ↓
[既存接続の確認 (Conn Table)]
   ↓
[NAT Untranslate (既存セッションの戻りパケット等)]
   ↓
[ACL チェック (NAT後のIP、またはNAT前のIPで評価)]
   ↓
[ルーティング決定 (Egress インターフェイスの特定)]
   ↓
[NAT Translate (送信元/宛先変換の適用)]
   ↓
[Egress パケット]
```
ASA/FTDでは、インターフェイスを跨ぐトラフィックに対してNATが適用されます。

---

## ⚙ 動作シーケンス

ASA 8.3以降のNAT処理には厳格な優先順位があります。

1.  **Section 1: Manual NAT (Twice NAT)**
    *   設定ファイルの最上位に表示されるルールです。
    *   送信元と宛先の両方を条件に変換を決定できます。
    *   特定の通信（例：VPN用Identity NAT）を最優先させるために使用します。
2.  **Section 2: Object NAT (Auto NAT)**
    *   ネットワークオブジェクトの定義内で設定されます。
    *   送信元アドレスのみに基づいたシンプルな変換に適しています。
3.  **Section 3: Manual NAT (After-auto)**
    *   Object NATでも一致しなかったパケットに適用される、低優先度の手動ルールです。

---

## 🎯 試験対策（CCIE Securityラボ試験）

### Blueprintで重要なポイント
*   **NAT Orderの制御**: 試験では「特定のトラフィックだけNATを回避し、他はPATせよ」という問題が出ます。Manual NATをSection 1に配置するスキルが必須です。
*   **Identity NAT (NAT Exemption)**: VPN通信において、パケットのアドレスを変換せずに透過させる設定です。

### ラボ試験で設定させられそうな内容
*   **Twice NATによるIP重複解決**: 重複する10.0.0.0/8ネットワーク間の通信を、仮想的なIPレンジにマッピングする構成。
*   **Static PAT (Port Forwarding)**: 外部からの特定のポート（TCP 80など）を内部サーバのプライベートIPに転送する設定。
*   **FMCでのNATポリシー構成**: 複数のFTDデバイスに対して、一貫したNATルールを配布する手順。

### よくある設定ミス
*   **インターフェイス名の指定ミス**: 変換前と後のインターフェイス（Real/Mapped）を逆にしてしまう。
*   **ACLとの整合性**: ASAのバージョンや設定（`no-proxy-arp`など）により、ACLで許可すべきIP（NAT前か後か）を混同する。

### showコマンド/debugによる状態判断
*   `show nat detail`: ルールの優先順位と、各ルールでのヒットカウントを確認します。
*   `packet-tracer`: パケットがどのNATルールにマッチし、どのように変換されるかをシミュレートする**最重要コマンド**です。

---

## 🛠 設定方法

### ASA (CLI) - Object NAT (Auto NAT)
```bash
# 内部ホストを外部IPでPAT
object network INTERNAL_LAN
 subnet 192.168.1.0 255.255.255.0
 nat (inside,outside) dynamic interface
```

### ASA (CLI) - Manual NAT (Twice NAT / Identity NAT)
```bash
# InsideからVPN対向(10.10.10.0)への通信をNAT除外
object network obj-local
 subnet 192.168.1.0 255.255.255.0
object network obj-remote
 subnet 10.10.10.0 255.255.255.0

nat (inside,outside) source static obj-local obj-local destination static obj-remote obj-remote
```

### FTD (FMC GUI)
1.  **Devices > NAT** を選択し、**New Policy > Threat Defense NAT** を作成。
2.  **Add Rule** をクリック。
    *   **NAT Type**: Static または Dynamic。
    *   **Interface Objects**: IngressとEgressのインターフェイス（Zone）を指定。
    *   **Translation**: Original Source, Mapped Source などをオブジェクトで指定。

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **NATルールの確認とヒットカウント** | <code>show nat detail</code> |
| **現在のNATセッション（xlate）確認** | <code>show xlate</code> |
| **特定のIPがどう変換されるか検証** | <code>packet-tracer input inside tcp 192.168.1.10 1234 8.8.8.8 80</code> |
| **NAT統計の表示** | <code>show nat statistics</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 対処方法 |
| :--- | :--- | :--- |
| NATルールにヒットしない | 優先順位（Section）が低い | <code>nat</code>コマンドの位置を調整、またはManual NATを使用する。 |
| VPNを通すと通信が切れる | Identity NATの設定漏れ | <code>nat (inside,outside) source static ...</code> で変換なしルールを追加。 |
| 外部から内部サーバにアクセス不能 | 戻りのルート、またはStatic NATのミス | <code>packet-tracer</code>で外部インターフェイスからの着信をテスト。 |
| DNSで名前解決したIPが違う | DNS Rewriteが未設定 | NATルールに <code>dns</code> キーワードを付与する。 |

---

## ⚠ 制限事項

*   **同一インターフェイス間のNAT**: デフォルトでは禁止されていますが、`same-security-traffic permit intra-interface` 設定が必要です。
*   **プロトコルの制限**: 一部の複雑なプロトコル（SIP, FTPなど）は、NAT通過のためにインスペクションが必要です。
*   **ライセンス**: 同時接続数（Conn数）の上限はライセンスに依存します。

---

## 🔄 他技術との関連

*   **Access Control**: 一般的に、ACLのチェックはNAT変換後の宛先IPに対して行われます（ASAの設定に依存）。
*   **Routing**: NAT変換後のパケットを送出するために、適切なEgressルートが必要です。
*   **VPN**: L2L VPNやAnyConnectでは、内部IPを保護するためにIdentity NATが必須となります。
*   **Identity Firewall**: ユーザIDに基づいたNATルールの適用が可能です。

---

## 🧩 比較表

| 比較項目 | Object NAT (Auto NAT) | Manual NAT (Twice NAT) |
| :--- | :--- | :--- |
| **定義場所** | ネットワークオブジェクト内 | グローバル設定階層 |
| **優先順位** | 中位 (Section 2) | 最高位 (Section 1) または 最低位 (Section 3) |
| **条件の柔軟性** | 送信元のみ | 送信元、宛先、サービスポートを組み合わせ可能 |
| **主な用途** | 一般的なPAT/Static NAT | Identity NAT, ポリシーNAT, IP重複対策 |

---

## 💡 ベストプラクティス

*   **オブジェクトの活用**: IP直書きではなく、常にネットワークオブジェクトを使用して管理性を高めます。
*   **PATプールの使用**: 変換後のIPが1つだけだとポート枯渇が起きる可能性がある場合、PATプール（Range）を使用します。
*   **no-proxy-arp**: 静的NATで、ASA自身がそのIPに対してARP応答を返す必要があるか確認し、不要なら `no-proxy-arp` を設定して不要なL2トラブルを防ぎます。

---

## 📝 ラボ学習・設定サンプル例

### 1. ダイナミックPAT (Hide NAT)
*   **問題**: 内部192.168.1.0/24をOutsideインターフェイスのIPでPATせよ。
*   **設定**: `object network LAN; subnet 192.168.1.0 255.255.255.0; nat (inside,outside) dynamic interface`

### 2. 静的NAT (Static NAT)
*   **問題**: 内部サーバ 10.1.1.5 をパブリックIP 203.0.113.5 に 1対1 マップせよ。
*   **設定**: `object network SERVER; host 10.1.1.5; nat (inside,outside) static 203.0.113.5`

### 3. Identity NAT (NAT Exemption)
*   **問題**: Insideから10.0.0.0/8宛のトラフィックをNATから除外せよ。
*   **設定**: `nat (inside,outside) source static LAN LAN destination static REMOTE_10/8 REMOTE_10/8`

### 4. 静的PAT (Port Forwarding)
*   **問題**: 外部からの TCP 8080 を内部 10.1.1.5 の TCP 80 に変換せよ。
*   **設定**: `object network SERVER; host 10.1.1.5; nat (inside,outside) static interface service tcp 80 8080`

### 5. ダイナミックNAT (Pool NAT)
*   **問題**: 内部ホストを 203.0.113.10-20 のプールから動的に割り当てよ。
*   **設定**: `object network POOL; range 203.0.113.10 203.0.113.20; object network LAN; nat (inside,outside) dynamic POOL`

### 6. DNS Rewrite
*   **問題**: 内部サーバを外部公開している際、内部クライアントがパブリックIPを引いた時にプライベートIPに書き換えよ。
*   **設定**: `nat (inside,outside) static 203.0.113.5 dns`

### 7. Twice NAT (Policy NAT)
*   **問題**: 内部10.1.1.10が8.8.8.8に行く時だけ 203.0.113.100 に変換せよ。
*   **設定**: `nat (inside,outside) source dynamic host-10.1.1.10 host-203.0.113.100 destination static host-8.8.8.8 host-8.8.8.8`

### 8. 同一サブネット内への宛先NAT
*   **問題**: 外部からのアクセスを、別のインターフェイスの同一セグメントホストへ変換。
*   **設定**: `same-security-traffic permit inter-interface` を併用。

### 9. サービスポート変換を伴うManual NAT
*   **問題**: 特定の送信元からの通信のみ、宛先ポートを書き換える。
*   **設定**: `nat (inside,outside) source static ... service tcp ...`

### 10. Section 3 (After-auto) NAT
*   **問題**: Object NATにマッチしなかった全通信のバックアップPATを設定。
*   **設定**: `nat (inside,outside) after-auto source dynamic any interface`

---

## ❓ 想定試験問題

1.  **問題**: `show nat` の出力で、特定のManual NATルールの下に多数のObject NATルールが表示されている。Manual NATを一番最後に処理させるにはどう設定すべきか？
    *   **解答**: `nat` コマンドに `after-auto` キーワードを付与して設定する。
2.  **問題**: 内部から外部DNSへの問い合わせ結果（パブリックIP）を、内部ホストが直接プライベートIPとして利用できるようにしたい。どの機能を使うか？
    *   **解答**: DNS Rewrite (NATルールの末尾に `dns` キーワードを追加)。
3.  **問題**: パケットトレーサーで NAT フェーズが `DROP` となり、原因が `Overlap` と表示された。考えられる理由は？
    *   **解答**: NAT後のマッピングIPが既存のインターフェイスIPや他のNATプールと重複している可能性がある。
4.  **問題**: FMCでNATルールを作成したが、FTDに反映されない。確認すべき項目は？
    *   **解答**: NATポリシーがデバイスに割り当てられているか、およびデプロイ（Deploy）が完了しているかを確認する。
5.  **問題**: Identity NATを設定したが、VPN経由の通信が依然としてPATされている。NATルールの順序はどうあるべきか？
    *   **解答**: Identity NAT（Manual NAT）をPATルール（Object NAT）よりも優先順位の高い Section 1 に配置する必要がある。

---

## 🔗 参考リソース

*   **Configuration Guides**:
    *   [Cisco ASA Series Firewall CLI Configuration Guide, 9.4 - NAT](https://www.cisco.com/c/en/us/td/docs/security/asa/asa94/configuration/firewall/asa-94-firewall-config/nat-overview.html)
    *   [Cisco Secure Firewall Management Center Device Configuration Guide, 7.1 - NAT Policies](https://www.cisco.com/c/en/us/td/docs/security/firepower/710/configuration/guide/fpmc-config-guide-v71/nat_for_firepower_threat_defense.html)
*   **Cisco Live (Videos & Slides)**:
    *   BRKSEC-3020: Troubleshooting Firewall Threat Defense (FTD)
    *   BRKSEC-2021: Firepower Threat Defense - Packet Flow and Troubleshooting
*   **Command Reference**:
    *   [Cisco ASA Series Command Reference - nat](https://www.cisco.com/c/en/us/td/docs/security/asa/command-reference/m-p/cmdref2/n1.html)
*   **Technical Notes**:
    *   ASA 8.3+: NAT and PAT Configuration Examples (Cisco.com)
    *   FTD NAT Configuration via FMC (Cisco Community)
---

## 📝 **補足（Notes）**  
- 学習メモ  
- 図解  
- 注意点  

