---
layout: default
title: 3.3.c-RTBH
nav_order: 3
parent: 3.3-Data-plane
grand_parent: 3.0-Security-Infrastructure
---

# 3.3.c RTBH (Remotely Triggered Black Hole)

**RTBH (Remotely Triggered Black Hole)** は、BGP をコントロールプレーンとして利用し、大規模な DoS/DDoS 攻撃トラフィックをネットワークのエッジ（境界）で効率的に破棄するための強力なフィルタリング技術です,。攻撃対象（Destination）または攻撃元（Source）へのトラフィックを、ルータの CPU に負荷をかける ACL ではなく、ハードウェアレベル（FIB）で `Null0` インターフェイスへ誘導して破棄します。

---

## 📘 概要

*   **機能概要**: 攻撃トラフィックを転送する代わりに、ネットワーク全体のエッジルータでそのパケットを「ブラックホール（Null0）」へ吸い込ませる手法です。
*   **利用目的**: DDoS 攻撃による回線帯域の枯渇防止、およびインフラデバイス（ルータ/FW）の保護。
*   **どのような場面で利用するか**: 特定の公開サーバへの大規模な攻撃が発生した際、被害をそのサーバのみに限定し、ネットワーク全体の崩壊（連鎖的なダウン）を防ぐために緊急避難的に実施します。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主な種類** | Destination-based RTBH（宛先ベース）, Source-based RTBH（送信元ベース）。 |
| **制御プロトコル** | iBGP（Trigger ルータから全エッジルータへ広報）。 |
| **破棄場所** | 宛先/送信元ルータではなく、パケットが入ってくる「エッジルータ」。 |
| **メリット** | 非常に高速（ラインレート処理）、CPU 負荷が極めて低い、迅速な展開が可能。 |
| **デメリット** | 宛先ベースの場合、正当な通信も含めて攻撃対象への全通信が遮断される。 |
| **必須コンポーネント** | Trigger ルータ（指示役）, Edge ルータ（実行役）, 共通の Blackhole Next-hop。 |

---

## 🏗 動作原理

RTBH は「ルーティングの仕組み」を逆手に取って動作します。

```text
[ Attacker ]
      ↓
[ Internet / Edge Router ] ←── (iBGP Update: Next-hop = 192.0.2.1) ── [ Trigger Router ]
      │
      ├─→ [ FIB Look up ] : 攻撃 IP 宛のネクストホップは 192.0.2.1
      │          ↓
      ├─→ [ Static Route ] : 192.0.2.1 は Null0 へ向いている
      │          ↓
      └─→ 【 DROP (Black Hole) 】
```

---

## ⚙ 動作シーケンス

1.  **攻撃検知**: NetFlow 等を使用して、攻撃対象 IP（D/RTBH）または攻撃元 IP（S/RTBH）を特定します。
2.  **Trigger 発動**: 管理者が Trigger ルータ上で、該当 IP に対するスタティックルート（Tag 付与等）を作成します。
3.  **BGP 広報**: Trigger ルータは、再配布（Redistribute）を通じて、全エッジルータに BGP アップデートを送信します。この際、**Next-hop を「未使用の IP アドレス（Blackhole IP）」に書き換えます**。
4.  **エッジでの解決**: 各エッジルータは、あらかじめ「Blackhole IP」を `Null0` 宛とするスタティックルートを保持しています。
5.  **トラフィック破棄**: 攻撃パケットが到着すると、エッジルータは FIB 参照により、パケットを即座に `Null0` へ捨てます。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **Trigger ルータの設定**: `route-map` を使用して `set ip next-hop` を指定する設定が頻出です。
*   **Edge ルータの準備**: 全てのエッジデバイスに `ip route [Blackhole_IP] 255.255.255.255 Null0` が設定されているか確認が必要です。
*   **Source-based RTBH (S/RTBH)**: 送信元ベースの場合、エッジルータで **uRPF Loose モード** が有効になっていないと動作しません。これは、送信元 IP を逆引きして `Null0` に一致したものを捨てるためです。
*   **BGP Community の活用**: `no-export` 属性などを付与して、ブラックホールルートが外部（ISP 側）に漏れないようにする要件が出ることがあります。
*   **Tag による制御**: スタティックルートに Tag を付け、それを BGP の `match tag` で拾って広報するフローを正確に構築してください。

---

## 🛠 設定方法

### 1. エッジルータ（Edge Router）の準備
全ての境界ルータで、ブラックホール誘導用の IP を `Null0` に向けます。
```bash
! ブラックホール用の仮想IP（ネットワーク内で使用していないアドレス）
ip route 192.0.2.1 255.255.255.255 Null0
```

### 2. Trigger ルータ（Trigger Router）の設定
攻撃対象 `203.0.113.100` を遮断する例。
```bash
! 1. 攻撃対象をスタティックルートで定義（Tag 666を付与）
ip route 203.0.113.100 255.255.255.255 Null0 tag 666

! 2. Route-map で Next-hop をエッジの Blackhole IP へ変更
route-map RM-RTBH permit 10
 match tag 666
 set ip next-hop 192.0.2.1
 set community no-export

! 3. BGP で広報
router bgp 65001
 redistribute static route-map RM-RTBH
```

### 3. Source-based RTBH のための Edge 設定
送信元 IP でフィルタする場合、エッジルータに以下が必要です。
```bash
interface GigabitEthernet0/1
 ip verify unicast source reachable-via any
 ! uRPF Loose モードにより、送信元が Null0 宛のパケットをドロップする
```

---

## 🔍 検証コマンド

| 目的 | コマンド |
| :--- | :--- |
| **BGP テーブルでの確認** | <code>show ip bgp [Attack_IP]</code> |
| **FIB での転送先確認** | <code>show ip cef [Attack_IP]</code> |
| **Next-hop が Null0 かの確認** | <code>show ip route [Blackhole_IP]</code> |
| **uRPF ドロップの確認** | <code>show ip interface \| include verify</code> |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| ブラックホール化されない | Edge に Null0 ルートがない | <code>ip route [Blackhole_IP] ... Null0</code> を追加。 |
| 全ルートが消えた | <code>redistribute static</code> のミス | Route-map の <code>match</code> 条件を再確認。 |
| S/RTBH が効かない | uRPF が無効または Strict モード | Edge IF で <code>reachable-via any</code> を設定。 |
| ルートが伝搬しない | BGP ネイバーの状態不良 | <code>show ip bgp summary</code> で接続を確認。 |

---

## ⚠ 制限事項

*   **サイドエフェクト**: 宛先ベース RTBH は、攻撃を受けているホストをネットワークから完全に切り離します（DoS 自体は成功している状態）。
*   **iBGP 依存**: 全てのエッジルータが Trigger ルータと同じ BGP AS 内（または反射鏡経由）にいる必要があります。
*   **IPv6 の考慮**: IPv6 トラフィックを遮断するには、別途 `ipv6 route ... Null0` と BGP IPv6 アドレスファミリーの設定が必要です。

---

## 🔄 他技術との関連

*   **BGP (2.4.c)**: RTBH の搬送プロトコル。
*   **uRPF (3.3.a)**: Source-based RTBH の動作に不可欠なデータプレーン検証技術。
*   **NetFlow (3.6.a)**: 攻撃トラフィック（Top Talker）を特定するための分析ツール。
*   **QoS (3.3.b)**: RTBH は「All or Nothing（全ドロップ）」ですが、QoS は「制限」です。併用されることがあります。

---

## 🧩 比較表

### D/RTBH vs S/RTBH

| 特徴 | Destination-based (D/RTBH) | Source-based (S/RTBH) |
| :--- | :--- | :--- |
| **目的** | 被害ホストへの通信を断つ | 攻撃者からの通信を断つ |
| **フィルタ対象** | **宛先 IP** | **送信元 IP** |
| **副作用** | 正常な通信も遮断される | 攻撃者以外は通信継続可能（理想的） |
| **必須技術** | BGP + Null0 Route | BGP + Null0 Route + **uRPF Loose** |

---

## 💡 ベストプラクティス

1.  **未使用 IP の予約**: RTBH 専用の `192.0.2.1` などの IP を組織内で予約し、全ルータにスタティック Null0 ルートを事前配布しておきます。
2.  **Community 指定**: `no-export` だけでなく、独自のデザインタグを付けて、Edge ルータ側でフィルタをかけやすくします。
3.  **自動化の検討**: 攻撃検知システム（Cisco Secure Network Analytics 等）から API 経由で Trigger ルータに設定を流し込む構成が推奨されます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 宛先ベース RTBH の実装
*   **要件**: Trigger ルータ R1 から、攻撃対象 1.1.1.1 への通信を Edge ルータ R2 で遮断せよ。
*   **設定**: R1 で `ip route 1.1.1.1 ... Null0 tag 99`, BGP で `set next-hop` 指定。R2 で `ip route [Next-hop] ... Null0`。

### 2. 送信元ベース RTBH と uRPF
*   **要件**: 攻撃者 100.1.1.1 からの全トラフィックをエッジで破棄せよ。
*   **設定**: uRPF Loose モードを Edge で有効化し、Trigger から 100.1.1.1 へのルートを広報。

### 3. BGP Community を使用した選択的破棄
*   **要件**: `65001:666` のコミュニティを持つルートを Edge でのみ `Null0` に向けよ。

### 4. IPv6 RTBH
*   **要件**: IPv6 の攻撃対象 `2001:DB8:A::1` をブラックホール化せよ。

### 5. スタティックルートのリセット
*   **課題**: 攻撃終了後、Trigger ルータから `no ip route` を実行し、疎通が復旧することを確認せよ。

---

## ❓ 想定試験問題

1.  **Design**: Source-based RTBH を導入する際、エッジルータのインターフェイスで必ず有効にすべき機能は？
    *   **回答**: **uRPF Loose モード** (`ip verify unicast source reachable-via any`)。
2.  **トラブルシュート**: Trigger ルータで広報を開始したが、エッジルータでパケットが破棄されない。`show ip route` ではルートを学習しているが、ネクストホップが `Null0` 宛の IP ではない。原因は？
    *   **回答**: Trigger ルータの `route-map` で `set ip next-hop` が正しく設定されていないか、`redistribute static` にマップが適用されていない。
3.  **コンフィグ読解**: `ip route 192.0.2.1 255.255.255.255 Null0` という設定が各ルータにある。この IP の役割は？
    *   **回答**: RTBH における共通の **Blackhole Next-hop アドレス**。
4.  **Design**: RTBH ルートを外部 ISP に広報しないための BGP 属性は？
    *   **回答**: **Community `no-export`**。
5.  **実装**: 多数の攻撃対象 IP を一度に RTBH にかける効率的な方法は？
    *   **回答**: それらの IP を同一の Tag でスタティックルート登録し、BGP でその Tag を一括して再配布する。

---

## 🔗 参考リソース

*   **Cisco White Paper**
    *   [Remotely Triggered Black Hole Filtering](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/213541-remotely-triggered-black-hole-filtering.html)
*   **Cisco Live (BRKSEC-2003)**
    *   [DDoS Mitigation Techniques on Cisco IOS](https://www.ciscolive.com/)
*   **Configuration Guide**
    *   [Configuring BGP RTBH](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/configuration/xe-16/irg-xe-16-book.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: RTBH は「毒（攻撃パケット）」を「毒（Null0）」で制す技術です。
*   **注意点**: ラボ試験では、RTBH の設定自体よりも、その前提となる **BGP のピアリング（ネイバー関係）** や **Next-hop の到達性** で躓くことが多いです。まずは BGP が正常にルートを運んでいるかを `show ip bgp` で確認する癖をつけましょう。
