---
layout: default
title: 3.5.d-TKIP
nav_order: 4
parent: 3.5-Wireless-security
grand_parent: 3.0-Security-Infrastructure
---

# 3.5.d TKIP

## 📘 概要

Temporal Key Integrity Protocol (TKIP) は、レガシーな Wired Equivalent Privacy (WEP) の深刻なセキュリティ上の脆弱性を迅速に解決するために、IEEE 802.11i ワーキンググループおよび Wi-Fi Alliance (WPA: Wi-Fi Protected Access) によって策定された過渡的な無線暗号化プロトコルです。

WEPが抱えていた「固定暗号鍵の使用」「24ビットと短い初期化ベクトル (IV) による鍵再利用問題」「改ざん検知メカニズム (CRC-32) の脆弱性」を克服するため、TKIPは既存のWEP対応ハードウェア（RC4暗号化エンジン）をそのまま活用しつつ、ファームウェアおよびソフトウェアのアップデートのみで導入できるように設計されました。

TKIPの主な利用目的と導入背景は以下の通りです。

* **WEPの緊急代替手段**: ハードウェアの買い替えを伴わずに、RC4暗号エンジン上で動的な鍵生成および強固な整合性チェックを追加。
* **パケットごとの鍵ミキシング (Per-Packet Key Mixing)**: 128ビットの Temporal Key (TK) と送信元MACアドレス、48ビットの Sequence Counter (TSC) を組み合わせてパケットごとに固有のRC4鍵を動的生成。
* **メッセージ送信元認証と改ざん防止 (Michael MIC)**: CRC-32に代わり、64ビットの暗号学的メッセージ整合性コード (Michael MIC) を導入し、アクティブなデータ改ざん攻撃をブロック。
* **リプレイ攻撃防止**: 48ビットの TKIP Sequence Counter (TSC) を導入し、受信側で単調増加しないパケットを即座に破棄。

現在では、TKIPの暗号基盤であるRC4およびMichael MICに対する各種攻撃手法が確立されたこと、また現代の高速無線規格（802.11n / 802.11ac / 802.11ax / 802.11be）においてTKIPの使用が規格上禁止（使用時は最大通信速度が 54 Mbps に制限される）されていることから、TKIPは完全なレガシー技術として非推奨（Deprecated）となっています。しかし、CCIE Security ラボ試験においては、レガシー端末の保護、WPA1/WPA2混在環境のトラブルシューティング、ならびにMIC Failure Countermeasures（Countermeasure動作）の挙動理解において重要なテーマとなります。


---

## 🔑 要点

TKIPの主要な技術的特徴、制限事項、設計上の注意点を下表に示します。

| 項目 | 内容 |
| --- | --- |
| **特徴** | WEPハードウェア(RC4エンジン)と互換性を保持しつつ、パケットごとの鍵生成(PPK)とメッセージ整合性コード(Michael MIC)を追加した過渡的暗号化プロトコル |
| **用途** | レガシーデバイス(WPA1対応機器、旧式バーコードスキャナ等)の無線接続保護、802.11i移行期の暫定暗号化 |
| **メリット** | WEP対応ハードウェアのファームウェア/ソフトウェア更新のみで適用可能であり、追加のハードウェア投資が不要 |
| **デメリット** | RC4ストリーム暗号依存による構造的脆弱性、Michaelアルゴリズムの計算コストと脆弱性、802.11n以降でのデータレート制限(最大54 Mbps) |
| **対応機種** | Cisco AireOS WLC、Catalyst 9800 WLC (互換性維持目的)、Cisco Autonomous AP、IOS-XE Wireless |
| **制限事項** | 802.11n / 802.11ac / 802.11ax 規格においてTKIP使用時はHT (High Throughput) / VHT / HE レートが不可、最大 54 Mbps (802.11a/g 速度) に制限 |
| **設計上の注意点** | MIC違反(MIC Failure)が60秒以内に2回発生すると、WLC/APは全クライアントを組織的に切断し、60秒間該当SSIDでの通信を停止(Countermeasures動作)するため、DoS攻撃の標的となりやすい |


---

## 🏗 動作原理

TKIPは、レガシーなRC4暗号化ハードウェアをそのまま流用しながらセキュリティを高めるため、複数の技術要素を多層的に組み合わせて動作します。

```
[ 上位アプリケーションデータ ]
          │
          ▼
[ Michael MIC 計算 ] ──── (64-bit Michael Key + MAC Header + Payload)
          │
          ▼
[ MSDU / MPDU フラグメンテーション ]
          │
          ▼
[ TKIP Sequence Counter (TSC) 付与 ] ── (48-bit Counter: リプレイ攻撃防止)
          │
          ▼
[ Per-Packet Key (PPK) Mixing Phase 1 & 2 ] ── (128-bit TK + TA + TSC ──> 128-bit PPK / 24-bit WEP IV)
          │
          ▼
[ RC4 暗号化 ] ──── (PPK による RC4 擬似ランダムストリーム生成 & XOR)
          │
          ▼
[ ICV (32-bit CRC) 付与 & フレーム送信 ]
```

TKIPを構成する4つの核心コンポーネントの動作原理は以下の通りです。

1. **TKIP Sequence Counter (TSC)**:
   * 48ビットのシーケンスカウンタであり、フレームが送信されるたびに1ずつインクリメントされます。
   * 受信側AP/クライアントは、直前に受信したフレームのTSC値を記憶しており、受信したパケットのTSCが記憶している値以下である場合、そのパケットを「リプレイパケット」とみなして即座に破棄します。
2. **Per-Packet Key (PPK) Mixing (パケットごとの鍵共有ミキシング)**:
   * 128ビットの Temporal Key (TK)、送信元MACアドレス (TA: Transmit Address)、および48ビットの TSC を組み合わせ、2段階のハッシュ関数（Phase 1 & Phase 2 Mixing）を実行します。
   * **Phase 1 Mixing**: 128-bit TK + TA + TSCのハイ32ビット ──> 80-bit P1K (Phase 1 Key, 10,000パケットごとに再計算)。
   * **Phase 2 Mixing**: 80-bit P1K + 128-bit TK + TSCのロー16ビット ──> 128-bit WEP Key (パケットごとに固有のRC4鍵)。
   * これにより、WEPで問題となった「同じIVと鍵の組み合わせが頻繁に再利用される」問題を防ぎます。
3. **Michael Message Integrity Check (MIC)**:
   * ペイロードの改ざんを検知するため、64ビットの Michael Key（送信・受信で各64ビット、計128ビットのMIC鍵を使用）、送信元/宛先MACアドレス、および非暗号化データ領域を対象に計算される64ビットのハッシュ値です。
   * 弱小なハードウェアCPUでも低負荷で計算できるよう簡略化されたアルゴリズムであるため、単体での暗号強度は限定的ですが、後述の「Countermeasures (対抗措置)」と組み合わせることで安全性を補強しています。
4. **ICV (Integrity Check Value)**:
   * WEPとの後方互換性を維持するため、平文データおよびMichael MICに対して32ビットのCRC-32 (ICV) を追加で計算し、RC4で共に暗号化します。


---

## ⚙ 動作シーケンス

TKIPにおけるパケットの送信処理シーケンスおよび受信検証シーケンスは以下の順番で実行されます。

### 送信処理シーケンス

```
[送信側: AP / Client]

1. 鍵の準備:
   802.1X/EAP または PSK による 4-Way Handshake 完了後、128-bit Temporal Key (TK) と 128-bit Michael Key (64-bit Tx / 64-bit Rx) を共有。

2. Michael MIC の計算:
   64-bit Tx Michael Key + DA + SA + Priority + MSDU Payload ──> 64-bit Michael MIC コード生成。
   MSDU ペイロードの末尾に 64-bit MIC を結合。

3. TSC のインクリメント:
   48-bit TKIP Sequence Counter (TSC) を 1 インクリメント。

4. 2段階鍵ミキシング (PPK Mixing):
   - Phase 1: TK (128-bit) + TA (MAC) + TSC[16..47] ──> 80-bit P1K 生成 (10,000パケットごとに更新)
   - Phase 2: P1K (80-bit) + TK (128-bit) + TSC[0..15] ──> 128-bit Per-Packet RC4 Key 生成

5. WEP ICV の計算 & RC4 暗号化:
   - (MSDU Payload + Michael MIC) に対する CRC-32 (32-bit ICV) を計算。
   - 128-bit Per-Packet Key を使用して RC4 暗号ストリームを生成し、[Payload + MIC + ICV] を XOR 暗号化。

6. 802.11 物理フレームの構築:
   - 802.11 ヘッダーの直後に 8バイトの TKIP ヘッダー (IV 3バイト + Extended IV 1バイト + TSC/Key ID 4バイト) を挿入して送信。
```

### 受信処理シーケンス

```
[受信側: AP / Client]

1. TKIP ヘッダー解析 & リプレイ検証:
   - 受信フレームから 48-bit TSC を抽出。
   - 受信側が保持している前回正常フレームの TSC と比較。
   - [判定] 受信 TSC <= 前回 TSC の場合、リプレイ攻撃と判断してフレームを即座に破棄 (Replay Failure Counter 加算)。

2. 鍵の再構成 & RC4 復号:
   - 抽出した TSC + 共有 TK (128-bit) + TA (送信元MAC) より、送信側と同じ Phase 1/2 Mixing を実行し 128-bit Per-Packet Key を復元。
   - RC4 復号を実行し、[Payload + Michael MIC + ICV] を取得。

3. ICV 検証:
   - 復号データの CRC-32 を計算し、添付の ICV と比較。
   - 不一致の場合、パケット破損または改ざんとみなし破棄 (ICV Error 加算)。

4. Michael MIC 検証 (最終整合性チェック):
   - 共有 Rx Michael Key (64-bit) + DA + SA + Priority + Payload より Michael MIC を再計算。
   - フレームに含まれる Michael MIC と照合。
   - [判定] 不一致の場合 ──> Michael MIC Failure が発生！
     - ログ出力および Trap 送信 (syslog / SNMP)。
     - MIC Failure カウンタを更新し、Countermeasure 処理ロジックへ移行。
```


## 🎯 試験対策（CCIE Securityラボ試験）

CCIE Security ラボ試験において、TKIPに関する出題ポイントは主に「MIC Failure Countermeasures（対抗措置）の動作検証」「WPA1/TKIP導入時の802.11n/ac速度制限のトラブルシューティング」「WLCにおけるWPA1/WPA2互換性設定」の3点に集中します。

### 1. Michael MIC Failure Countermeasures (対抗措置メカニズム)

TKIPでは、64ビットの Michael MIC の暗号強度が比較的低い（総当たり攻撃が可能）ため、攻撃者によるハッシュ解読試行を防御する目的で「MIC Failure Countermeasures」が規定されています。

* **発動条件**: **60秒以内に2回の MIC Failure** が同一BSSID（AP）上で検出された場合。
* **発動時のシステム動作**:
  1. 該当BSSIDに接続されている**すべての無線クライアントに Deauthentication (切断) フレームを送信**して強制切断。
  2. 該当BSSIDにおけるグループキー (GTK) およびペアワイズキー (PTK) の更新・再生成処理を停止。
  3. **60秒間 (1分間)、該当SSIDでの新規アソシエーションおよび再接続を完全にブロック** (SSIDのビーコン停止または接続拒否)。
  4. 60秒経過後、新しい GTK を生成してサービスを自動再開。
* **ラボ試験での問われ方**:
  * 「クライアントが定期的に切断され、1分間再接続できない症状の原因を特定せよ」という障害解析問題。
  * `show wlan` や `show wireless client mac`、WLCのログ (`syslog` / `msglog`) から MIC Failure カウンタおよび Countermeasure タイマーの稼働状態を読み取る能力が評価されます。

### 2. 802.11n / 802.11ac / 802.11ax レート制限問題

IEEE 802.11n 規格以降、TKIP および WEP 暗号化の使用は高スループット (HT: High Throughput) レートのサポート対象外と規定されています。

* **問題の症状**: クライアントが 802.11n/ac/ax 対応のAPに接続しているにもかかわらず、Wi-Fi リンク速度（物理レート）が **最大 54 Mbps**（802.11a/g レート）に制限される。
* **原因**: WLANのセキュリティ設定で WPA/TKIP のみが有効化されている、あるいは WPA2/TKIP (CCMPが未有効) に設定されている。
* **対策**: WLAN セキュリティで WPA2/WPA3 および AES (CCMP/GCMP) 暗号化を必須化または有効化する。

### 3. コンフィグ読解とトラブルシューティングのポイント

* **WLCログでの MIC Failure 判別**:
  * ログ内に `MIC_FAILURE` や `TKIP Countermeasure activated on AP ... for 60 seconds` というメッセージが出力されているか確認。
* **意図しない DoS 攻撃対策**:
  * 悪意のある攻撃者が偽造した MIC エラーパケットを意図的に2回送出することで、拠点全体のWi-Fi通信を60秒間麻痺させることが可能です。ラボ試験では、この挙動を理解した上で WPA2-AES / WPA3 への移行策を提示する構成変更が求められます。


---

## 🛠 設定方法

Catalyst 9800 WLC (IOS-XE Wireless) および AireOS WLC における WPA1/WPA2 (TKIP/AES) の設定例を示します。

### 1. Catalyst 9800 WLC (IOS-XE) での設定例

IOS-XE 環境では、WLAN プロファイル配下で WPA1/WPA2 暗号化方式および Ciphers (TKIP / AES) を定義します。

```text
!
wlan WLAN-LEGACY-TKIP 10 WLAN-LEGACY-TKIP
 client vlan VLAN-100
 security wpa
  security wpa wpa1 ciphers tkip
  security wpa wpa2 ciphers aes
  security wpa akm psk
  security wpa akm psk set-key ascii 0 SecretPsk123!
 no shutdown
!
wireless tag policy POLICY-TAG-LEGACY
 wlan WLAN-LEGACY-TKIP policy DEFAULT-POLICY-PROFILE
!
```

### 2. AireOS WLC (CLI) での設定例

AireOS 環境における WLAN への WPA1/TKIP および WPA2/AES の重複許可設定です。

```text
config wlan disable 10
config wlan security wpa enable 10
config wlan security wpa wpa1 enable 10
config wlan security wpa wpa1 ciphers tkip enable 10
config wlan security wpa wpa2 enable 10
config wlan security wpa wpa2 ciphers aes enable 10
config wlan security wpa akm psk enable 10
config wlan security wpa akm psk set-key ascii SecretPsk123! 10
config wlan enable 10
```


---

## 🔍 検証コマンド

TKIP の動作状態、MIC Failure の発生状況、および Countermeasures タイマーを確認するための主要コマンド一覧です。

| 目的 | コマンド |
| --- | --- |
| WLANの暗号化設定(TKIP/AES)確認 (Cat9800) | `<code>show wlan name <wlan-name></code>` |
| 接続クライアントの暗号化プロトコル(TKIP/AES)確認 (Cat9800) | `<code>show wireless client mac <mac-address> detail</code>` |
| 無線インターフェイスのセキュリティ統計確認 (Cat9800) | `<code>show wireless stats client summary</code>` |
| MIC Failure および Countermeasure のログ確認 (Cat9800) | `<code>show logging \| include MIC_FAILURE\|COUNTERMEASURE</code>` |
| AireOS での WLAN セキュリティ詳細確認 | `<code>show wlan <wlan-id></code>` |
| AireOS でのクライアント詳細・暗号方式確認 | `<code>show client detail <mac-address></code>` |
| AireOS での MIC Failure カウンタ統計確認 | `<code>show ap debug rx-stats <ap-name></code>` |
| TKIP/Key ハンドシェイクのリアルタイムデバッグ (Cat9800) | `<code>debug wireless mac <mac-address></code>` |
| AireOS での 802.1X / EAP / Key トラブルシュート | `<code>debug client <mac-address></code>` |


---

## 🚨 トラブルシュート

TKIP 運用時に遭遇する代表的な障害症状、根本原因、確認コマンド、および対処方法を下表にまとめます。

| 症状 | 原因 | 確認コマンド | 対処方法 |
| --- | --- | --- | --- |
| 全クライアントが一斉に切断され、60秒間接続不能になる | 60秒以内に2回の Michael MIC Failure が発生し、Countermeasures が発動 | `<code>show logging \| include MIC_FAILURE</code>`<br>`<code>show wlan summary</code>` | ログで MIC Failure 発生元を調査。悪意のある攻撃またはノイズ/電波干渉によるパケット破損の排除。WPA2-AES/WPA3 への設定変更 |
| 802.11n/ac 対応端末のリンク速度が最大 54 Mbps に制限される | WLAN の暗号化方式に TKIP が選択されており、HT/VHT (High Throughput) が無効化されている | `<code>show wireless client mac <mac> detail</code>`<br>`<code>show wlan name <wlan-name></code>` | WLAN セキュリティ設定で WPA2-AES (CCMP) または WPA3 (GCMP) を指定し、TKIP を無効化する |
| 特定端末が WPA1/TKIP WLAN に接続失敗する | 端末側のドライバが TKIP 拡張ヘッダーや Group Key 更新に対応していない、または PSK 不一致 | `<code>debug wireless mac <mac-address></code>`<br>`<code>show client detail <mac></code>` | 端末ドライバの更新、または WPA2/WPA3 対応プロファイルへの切替。4-Way Handshake ログの解析 |
| EAP/802.1X 認証成功後に IP アドレスが取得できない (4-Way Handshake 失敗) | TKIP の PTK/GTK 鍵交換時における 4-Way Handshake タイムアウト | `<code>debug wireless mac <mac-address></code>`<br>`<code>show trace connection</code>` | EAPOL タイムアウト値の調整、AP-WLC 間の MTU/フラグメンテーションの確認、キー再生成インターバルの見直し |


## ⚠ 制限事項

TKIP の使用における代表的な制限事項および互換性上の制約は以下の通りです。

* **802.11n / 802.11ac / 802.11ax / 802.11be 非対応 (速度制限)**:
  * IEEE 802.11n 規格以降、TKIP および WEP 暗号化を使用した接続では High Throughput (HT) / Very High Throughput (VHT) / High Efficiency (HE) の物理レート（150 Mbps, 866 Mbps, 1.2 Gbps 等）が完全に無効化されます。通信速度は最高でも **54 Mbps** に限定されます。
* **WPA3 での完全サポート廃止**:
  * Wi-Fi Alliance の WPA3 規格および IEEE 802.11-2020 規格において、TKIP のサポートは完全に撤廃されています。WPA3-Personal (SAE) および WPA3-Enterprise では TKIP を選択できません。
* **CPU 処理負担とハードウェアアクセラレーション制限**:
  * 一部の近代的な無線クライアントやアクセスポイントのチップセットは、TKIP (RC4 + Michael) のハードウェア暗号化エンジンを搭載しておらず、ソフトウェア処理にフォールバックしてCPU使用率が高騰するケースがあります。
* **MIC Failure による構造的 DoS 脆弱性**:
  * 60秒間に2回の MIC エラーでSSID全体が60秒間停止する仕様は、攻撃者が偽造パケットを注入するだけで拠点のWi-Fiをダウンさせられる深刻な DoS 脆弱性となります。


---

## 🔄 他技術との関連

TKIP は以下の無線・セキュリティ技術と密接に関連しています。

* **IEEE 802.11n / 802.11ac / 802.11ax (高速無線規格)**:
  * これらの規格で最大速度を発揮するためには、WLAN の暗号化プロトコルとして AES (CCMP / GCMP) の使用が必須条件です。TKIP が有効な WLAN では 802.11a/g 互換モードで動作します。
* **WPA2 (CCMP/AES) および WPA3 (GCMP/SAE)**:
  * TKIP は WPA1 の標準暗号方式ですが、WPA2 では CCMP (AES-CCM) が標準化され、WPA3 では GCMP が導入されました。TKIP から CCMP/GCMP への移行は現代のネットワークセキュリティの必須要件です。
* **802.1X / EAP 認証 & Cisco ISE**:
  * WPA-Enterprise (TKIP) 環境においても 802.1X 認証（EAP-TLS, PEAP 等）が実行され、RADIUS アトリビュートを介した Pairwise Master Key (PMK) の配布が行われます。
* **Rogue Detection & Wireless DoS 防御**:
  * 悪意のある不正アクセスポイント（Rogue AP）やジャマーによる MIC Failure インジェクション攻撃を監視するため、WLC の Rogue & DoS 防御メカニズムと連携します。


---

## 🧩 比較表

無線LANにおける主要な暗号化プロトコル（WEP, TKIP, CCMP/AES, GCMP）の比較を以下に示します。

| 比較項目 | WEP | TKIP (WPA1) | CCMP (WPA2/WPA3) | GCMP (WPA3 192-bit) |
| --- | --- | --- | --- | --- |
| **標準規格** | IEEE 802.11 (1997) | IEEE 802.11i / WPA | IEEE 802.11i / WPA2 / WPA3 | IEEE 802.11ac/ax / WPA3-Ent |
| **暗号化アルゴリズム** | RC4 (ストリーム暗号) | RC4 (ストリーム暗号) | AES-CCM (ブロック暗号) | AES-GCM (ブロック暗号) |
| **暗号鍵長** | 40-bit / 104-bit 固定 | 128-bit Temporal Key | 128-bit (WPA2) / 192-bit (WPA3) | 192-bit (CNSA Suite B) |
| **パケット鍵生成** | なし (固定鍵 + 24-bit IV) | 2段階 PPK ミキシング | Counter Mode (AES) | Galois/Counter Mode (AES) |
| **整合性チェック (MIC)** | CRC-32 (ICV: 脆弱) | Michael MIC (64-bit) | CBC-MAC (128-bit) | GMAC (128-bit) |
| **IV / Sequence カウンタ** | 24-bit IV (暗号再利用) | 48-bit TSC (リプレイ防止) | 48-bit PN (Packet Number) | 48-bit PN (Packet Number) |
| **最高サポート速度** | 54 Mbps (802.11a/g) | 54 Mbps (802.11a/g) | マルチGbps (802.11n/ac/ax/be) | マルチGbps (802.11ax/be) |
| **セキュリティレベル** | 極めて危険 (数分で解読) | 危険 (解読攻撃手法が存在) | 極めて強固 | 最上級 (政府・ミリタリーレベル) |


---

## 💡 ベストプラクティス

1. **TKIPの完全無効化 (WPA2-AES / WPA3 への全面移行)**:
   * ネットワーク内のすべての WLAN で WPA1/TKIP を無効化し、WPA2-AES (CCMP) または WPA3-SAE/Enterprise に変更することを最優先とします。
2. **レガシー端末用 SSID の隔離 (レガシーSSIDの分離)**:
   * 業務上どうしても TKIP しかサポートしない旧式機器（古いバーコードスキャナ等）が存在する場合は、専用のレガシーSSIDを分離して作成し、最小限のアクセス権限（VACL/ACL、Private VLAN、ステートフルファイアウォールによる制限）を適用します。
3. **802.11n/ac/ax スループット性能の確保**:
   * メインの業務用SSIDで TKIP を許可していると、最新クライアントも誤って TKIP でアソシエートし、通信速度が 54 Mbps に低下するトラブルが発生します。業務用SSIDでは `wpa2 ciphers aes` のみを許可します。
4. **MIC Failure 発生時の自動通知構成**:
   * やむを得ず TKIP を運用する場合は、WLC 上で SNMP Trap または Syslog 通知を有効化し、Countermeasures 発動による拠点障害を即座に検知できる運用体制を構築します。


## 📝 ラボ学習・設定サンプル例

実際のCCIE Securityラボ試験レベルを意識した、TKIPおよび無線セキュリティに関連する10個の設定・トラブルシューティング演習例を示します。

### サンプル1: Catalyst 9800 にてレガシー機器向け WPA1/TKIP SSID を構築する

* **問題**: レガシーバーコードスキャナ用として、SSID `LEGACY-SCANNER` (VLAN 105) を WPA1-PSK (Passphrase: `ScannerKey2026`) / TKIP 暗号化で構築せよ。
* **要件**: WPA2 は無効化し、WPA1/TKIP のみアソシエーションを許可すること。

```text
! [設定例: Catalyst 9800]
wlan LEGACY-SCANNER 105 LEGACY-SCANNER
 client vlan VLAN-105
 security wpa
  security wpa wpa1
  security wpa wpa1 ciphers tkip
  no security wpa wpa2
  security wpa akm psk
  security wpa akm psk set-key ascii 0 ScannerKey2026
 no shutdown
```

* **検証方法**:
  `show wlan name LEGACY-SCANNER` を実行し、`WPA1 Policy: Enabled`、`WPA1 Ciphers: TKIP`、`WPA2 Policy: Disabled` であることを確認。


---

### サンプル2: AireOS WLC にて WPA1-TKIP / WPA2-AES 混在 WLAN を構築する

* **問題**: 遷移期対応として WLAN ID 12 (`MIGRATE-SSID`) を作成し、WPA1-TKIP と WPA2-AES の両方を許可せよ。
* **要件**: PSK 認証 (`MigrateSecret!`) を使用すること。

```text
! [設定例: AireOS WLC CLI]
config wlan disable 12
config wlan create 12 MIGRATE-SSID MIGRATE-SSID
config wlan interface 12 vlan-200
config wlan security wpa enable 12
config wlan security wpa wpa1 enable 12
config wlan security wpa wpa1 ciphers tkip enable 12
config wlan security wpa wpa2 enable 12
config wlan security wpa wpa2 ciphers aes enable 12
config wlan security wpa akm psk enable 12
config wlan security wpa akm psk set-key ascii MigrateSecret! 12
config wlan enable 12
```

* **検証方法**:
  `show wlan 12` を実行し、WPA1 Ciphers に TKIP、WPA2 Ciphers に AES (CCMP) の両方がリストされていることを確認。


---

### サンプル3: TKIP による 802.11n 速度制限障害のトラブルシューティング

* **問題**: 802.11n クライアントが SSID `CORP-WIFI` 接続時に最大 54 Mbps でしか通信できない。802.11n 高速化 (HT) を有効化せよ。
* **要件**: WLAN セキュリティから TKIP を除外し、WPA2-AES のみを強制すること。

```text
! [設定例: Catalyst 9800]
wlan CORP-WIFI 11 CORP-WIFI
 security wpa
  no security wpa wpa1
  security wpa wpa2
  security wpa wpa2 ciphers aes
  no security wpa wpa2 ciphers tkip
```

* **検証方法**:
  クライアント接続後、`show wireless client mac <mac> detail` を実行し、`Current Rate` が HT/VHT レート（例: 144.4 Mbps 以上）に復旧したことを確認。


---

### サンプル4: Catalyst 9800 にて 802.1X (EAP-PEAP) / TKIP 構成を構築する

* **問題**: 医療用移動カート端末向けに、802.1X 認証と TKIP 暗号化を組み合わせた SSID `MED-CART` を構築せよ。
* **要件**: RADIUS サーバーグループ `ISE-RADIUS-GROUP` を使用すること。

```text
! [設定例: Catalyst 9800]
wlan MED-CART 108 MED-CART
 client vlan VLAN-108
 security wpa
  security wpa wpa1
  security wpa wpa1 ciphers tkip
  no security wpa wpa2
  security wpa akm dot1x
  authentication-server ISE-RADIUS-GROUP
 no shutdown
```

* **検証方法**:
  `show wlan name MED-CART` で `Authentication: 802.1X` および `WPA1 Ciphers: TKIP` を確認。


---

### サンプル5: Michael MIC Failure 発生時の Syslog アラート出力の確認

* **問題**: WLC 上で MIC Failure 発生時に即座に Syslog アラートを外部サーバー `192.168.10.50` に送出する設定を行え。
* **要件**: トラップレベルを info 以上に設定すること。

```text
! [設定例: Catalyst 9800]
logging host 192.168.10.50
logging trap informational
logging source-interface GigabitEthernet1
```

* **検証方法**:
  `show logging` を実行し、ロギングホストの登録および発生した `WIRELESS-3-MIC_FAILURE` イベントログ出力を確認。


---

### サンプル6: AireOS WLC にて Countermeasures タイマーの発動状態を検証する

* **問題**: 接続中のクライアントが Michael MIC エラーを多発させた際の、AireOS におけるアソシエーション制限状態を検証せよ。
* **要件**: CLI より統計情報および AP デバッグ情報を収集すること。

```text
! [検証手順: AireOS CLI]
(Cisco Controller) > show wlan 10
(Cisco Controller) > show ap debug rx-stats AP-FLOOR1
(Cisco Controller) > debug client 00:11:22:33:44:55
```

* **検証方法**:
  Countermeasures 発動時に `TKIP countermeasure invoked for BSSID ...` というログと、60秒間のブロックタイマーが稼働していることを確認。


---

### サンプル7: FlexConnect Local Switching 環境における TKIP WLAN の構成

* **問題**: 拠点 AP (FlexConnect モード) 上でローカルスイッチングされる SSID `BRANCH-TKIP` を構築せよ。
* **要件**: データトラフィックは拠点スイッチ VLAN 30 にローカルオフロードすること。

```text
! [設定例: Catalyst 9800]
wlan BRANCH-TKIP 130 BRANCH-TKIP
 security wpa
  security wpa wpa1 ciphers tkip
  security wpa akm psk
  security wpa akm psk set-key ascii 0 BranchKey987
 no shutdown

wireless profile policy BRANCH-FLEX-POLICY
 flex-profile BRANCH-FLEX-PROFILE
 central switching disabled
 central dhcp disabled
 vlan-central-switching disabled
```

* **検証方法**:
  `show wireless client mac <mac> detail` で `Client State: Associated`、`Data Switching: Local`、`Cipher: TKIP` を確認。


---

### サンプル8: Cat9800 にて TKIP 端末の 4-Way Handshake をデバッグ追跡する

* **問題**: TKIP 接続端末 `aabb.cc00.1122` の EAPOL Key 交換失敗の原因を追跡せよ。
* **要件**: 条件付きデバッグ (Conditional Debug) を使用すること。

```text
! [設定手順: Catalyst 9800]
debug wireless mac aabb.cc00.1122
! 接続試行実行後
show logging profile wireless filter mac aabb.cc00.1122 addresses
undebug all
```

* **検証方法**:
  抽出されたログ内で `EAPOL-Key M2` または `M4` 送受信時の MIC 検証ステップおよび Key Index 設定結果を確認。


---

### サンプル9: TKIP 非サポート端末の排除ルール（WPA2/WPA3 専用SSIDの完全強制）

* **問題**: 既存の混在 SSID `CORP-GUEST` から TKIP および WPA1 を完全に削除し、WPA2-AES および WPA3-SAE (Transition Mode) に統合せよ。
* **要件**: Protected Management Frames (PMF) を Optional に設定すること。

```text
! [設定例: Catalyst 9800]
wlan CORP-GUEST 20 CORP-GUEST
 security wpa
  no security wpa wpa1
  security wpa wpa2
  security wpa wpa2 ciphers aes
  security wpa wpa3
  security wpa akm psk
  security wpa akm sae
  security wpa akm psk set-key ascii 0 GuestPass2026!
  pmf optional
 no shutdown
```

* **検証方法**:
  `show wlan name CORP-GUEST` で `WPA1: Disabled`、`WPA2 Ciphers: AES`、`WPA3 Ciphers: AES` を確認。


---

### サンプル10: TKIP カウンターメジャー発動後の手動インターフェイス復旧演習

* **問題**: テスト環境にて Countermeasures により 60秒間ブロックされた WLAN/AP 状態を手動でリセット・復旧させる手順を確認せよ。
* **要件**: WLAN の有効化・無効化サイクルを実行すること。

```text
! [復旧手順: Catalyst 9800]
wlan LEGACY-SCANNER 105 LEGACY-SCANNER
 shutdown
 no shutdown
```

* **検証方法**:
  `show wlan name LEGACY-SCANNER` で Status が `Enabled` に復帰し、クライアントからのアソシエーション要求が即座に受諾されることを確認。


## ❓ 想定試験問題

CCIE Security ラボ試験・筆記試験レベルを想定した実践問題5問です。

### 問1 (トラブルシューティング)
**問題**:
Catalyst 9800 WLC において、802.11n 対応のハンドヘルド端末が SSID `WAREHOUSE` 接続時に物理リンク速度 54 Mbps 以上を達成できません。WLC のコンフィグを確認したところ、以下の設定が記述されていました。通信速度が低下している根本原因と、速度制限を解消するための適切なコンフィグ変更を記述しなさい。

```text
wlan WAREHOUSE 100 WAREHOUSE
 security wpa
  security wpa wpa1
  security wpa wpa1 ciphers tkip
  security wpa wpa2
  security wpa wpa2 ciphers tkip
  security wpa akm psk
  security wpa akm psk set-key ascii 0 WarehouseKey123
```

**解答・解説**:
* **原因**: 802.11n 規格において、TKIP 暗号化プロトコルは High Throughput (HT) レートをサポートしておらず、TKIP が有効な状態（または WPA2 で Ciphers に TKIP が選択されている状態）では自動的に 802.11a/g のレガシーレート (最大 54 Mbps) にフォールバックします。
* **修正コンフィグ**:
  ```text
  wlan WAREHOUSE 100 WAREHOUSE
   security wpa
    no security wpa wpa1
    security wpa wpa2
    security wpa wpa2 ciphers aes
    no security wpa wpa2 ciphers tkip
  ```


---

### 問2 (機構・シーケンス理解)
**問題**:
TKIP の Michael Message Integrity Check (MIC) において、60秒間に2回の MIC Failure が発生した場合に発動する Countermeasures (対抗措置) の全動作ステップを順序立てて説明しなさい。

**解答・解説**:
1. 該当 BSSID に接続中のすべての無線クライアントに対して Deauthentication (切断) フレームを送信し、接続を即座に遮断します。
2. 該当 BSSID 上での Pairwise Key (PTK) および Group Key (GTK) の更新処理および再生成処理を即座に凍結・停止します。
3. **60秒間 (1分間)**、該当 BSSID でのアソシエーション要求および再接続を完全に拒否・ブロックします。
4. 60秒経過後、新しい Group Key (GTK) を再生成してアソシエーションの受付および通常サービスを自動再開します。


---

### 問3 (コンフィグ読解・デザイン)
**問題**:
以下の Catalyst 9800 WLC コンフィグにおいて、セキュリティ上の課題を2点指摘し、その対策を説明しなさい。

```text
wlan LEGACY-WIFI 50 LEGACY-WIFI
 security wpa
  security wpa wpa1
  security wpa wpa1 ciphers tkip
  security wpa akm psk
  security wpa akm psk set-key ascii 0 SimpleKey123
```

**解答・解説**:
1. **TKIP 使用による影響**:
   * **課題**: TKIP はレガシー暗号化方式であり、Michael MIC 違反により 60秒間アソシエーションが停止する影響があります。
   * **対策**: WPA2-AES (CCMP) または WPA3-SAE へ暗号化方式をアップグレードする。
2. **共有鍵の設定**:
   * **課題**: 単純な PSK はセキュリティ強度が限定的です。
   * **対策**: 複雑な PSK に変更するか、802.1X (EAP-TLS / PEAP) 認証を採用する。


---

### 問4 (トラブルシューティング)
**問題**:
WLC の Syslog ログに以下のメッセージが記録されました。発生している事象と、確認コマンドを説明しなさい。

`%DOT11-3-MIC_FAILURE: AP Floor2-AP, Client 0011.2233.4455: Michael MIC verification failed for received packet.`

**解答・解説**:
* **発生事象**: クライアント `0011.2233.4455` から受信したパケットの Michael MIC 値と、AP 側で再計算した MIC 値が不一致となったためパケットが破棄されました。これが60秒以内に2回発生すると Countermeasures が発動します。
* **確認コマンド**:
  * `show logging | include MIC_FAILURE` (MIC エラーの発生頻度の特定)
  * `show wireless client mac 0011.2233.4455 detail` (該当クライアントの接続状態確認)
  * `show ap debug rx-stats` (AP 受信パケットのエラー統計確認)


---

### 問5 (実装・暗号理論)
**問題**:
TKIP の「Per-Packet Key (PPK) Mixing」の2段階（Phase 1 及び Phase 2）処理において、それぞれの出力結果を説明しなさい。

**解答・解説**:
* **Phase 1 Mixing**:
  * **出力**: 80ビットの Phase 1 Key (P1K)。(10,000パケットごとに更新)
* **Phase 2 Mixing**:
  * **出力**: 128ビットの Per-Packet Key (パケットごとに固有の RC4 暗号化キー)。


---

## 🔗 参考リソース

Cisco 公式の技術ドキュメント、Cisco Live 資料、Configuration Guide 一覧です。

* Cisco Wireless Controller Configuration Guide, Release 8.10 - Configuring WPA and WPA2
* Cisco Catalyst 9800 Series Wireless Controller Software Configuration Guide, IOS XE 17.x - WLAN Security Settings
* Cisco Technical Support & Documentation - Troubleshooting TKIP and WPA Security Failures
* Cisco Live Slide - BRKSEC-3055: Enterprise Wireless Security Architecture
* Wi-Fi Alliance Technical Whitepaper - WPA / WPA2 Security Enhancements and TKIP Architecture
* Cisco Technical Note - 802.11n Data Rates and Encryption Protocol Support (TKIP / AES Requirements)


---

## 📝 **補足（Notes）**

### TKIP パケット構造概要図

```text
+-------------------+--------------------+------------------------+-------------------+----------------+
| 802.11 Header     | TKIP Header        | Encrypted Payload      | Encrypted MIC     | ICV (CRC-32)   |
| (24-30 Bytes)     | (8 Bytes)          | (Variable Bytes)       | (8 Bytes)         | (4 Bytes)      |
+-------------------+--------------------+------------------------+-------------------+----------------+
                    | IV (3B) | Ext IV   |
                    | (TSC0)  | (TSC1-5) |
```

### 注意点
TKIP は現在レガシー技術として扱われます。ラボ試験や実務のトラブルシューティングにおいて概念理解は必須ですが、新規構築時には必ず WPA2-AES (CCMP) または WPA3-SAE (GCMP) を選択してください。
