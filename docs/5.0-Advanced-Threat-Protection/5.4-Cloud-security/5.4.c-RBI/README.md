---
layout: default
title: 5.4.c-RBI
nav_order: 3
parent: 5.4-Cloud-security
grand_parent: 5.0-Advanced-Threat-Protection
---

# 5.4.c RBI policies in Cisco Umbrella

**Remote Browser Isolation (RBI)** は、Cisco Umbrella の **SIG (Secure Internet Gateway)** 機能の一部として提供される高度な保護技術です。従来の Web フィルタリング（許可または拒否）とは異なり、Web コンテンツを Umbrella が管理するクラウド上の仮想コンテナ内で実行・レンダリングし、その「描画情報（ピクセルストリーム）」のみをユーザーのブラウザに送ります。これにより、エンドユーザーの端末で直接悪意のあるコード（JavaScript 等）が実行されることを物理的に防ぎます。

---

## 📘 概要

*   **機能概要**: 未知の Web サイトやリスクのあるカテゴリのサイトを、ユーザーのローカルブラウザから隔離されたクラウド環境で実行する機能。
*   **利用目的**: ゼロデイ攻撃の防止、ブラウザの脆弱性を突くエクスプロイトの無効化、および機密データ入力の制限。
*   **どのような場面で利用するか**:
    *   **Uncategorized サイト**: まだ評価が定まっていない新しいドメインへのアクセス時。
    *   **リスクの高いカテゴリ**: 掲示板や個人ブログなど、コンテンツが動的に変化し脅威が埋め込まれやすいサイト。
    *   **機密情報の保護**: 特定のサイトで「閲覧は許可するが、書き込み（アップロード）や入力を禁止したい」場合（Read-Only モード）。

---

## 🔑 要点

| 項目 | 内容 |
| :--- | :--- |
| **主要コンポーネント** | Cisco Umbrella SIG (DNS + SWG), クラウド分離エンジン。 |
| **保護のレベル** | **Full Isolation**（全 Web トラフィック）または **Selective Isolation**（特定カテゴリのみ）。 |
| **ユーザー体験** | ユーザーには通常のブラウジングに見えるが、実体はクラウド上のピクセルデータ。 |
| **制御の柔軟性** | 読み取り専用（Read-Only）モード、クリップボード制限、印刷制限が可能。 |
| **対応プロトコル** | HTTP / HTTPS (要 SSL 復号)。 |
| **要件** | Umbrella SIG ライセンス (Essentials/Advantage) および SWG の構成。 |

---

## 🏗 動作原理

RBI は、ユーザーとインターネットの間に「安全な緩衝地帯」を設ける仕組みです。

```text
User Browser
   ↓ (1) HTTPS Request (to potentially risky site)
Umbrella SWG (Proxy)
   ↓ (2) Policy Match: Action = Isolate
Remote Browser (Cloud Container)
   ↓ (3) Fetches and executes code/content from Internet
Internet Site (Evil code)
   ↓ (4) Code executes in Container (Infected!)
Umbrella RBI Engine
   ↓ (5) Streams only safe pixels back to User
User Browser (Safe rendering)
```

---

## ⚙ 動作シーケンス

1.  **トラフィックの捕捉**: ユーザーが Web サイトにアクセスしようとすると、トラフィックは Umbrella SWG（セキュア Web ゲートウェイ）にルーティングされます。
2.  **ポリシー評価**: Umbrella は Web ポリシーを上から順にスキャンし、該当する Identity と目的地に対して `Action: Isolate` が設定されているか確認します。
3.  **セッションの隔離**: 条件に一致した場合、Umbrella はクラウド上に一時的なコンテナ（隔離ブラウザ）を立ち上げます。
4.  **リモート実行**: クラウド上の隔離ブラウザが目的のサイトへアクセスし、JavaScript やメディアをダウンロード・実行します。
5.  **ピクセル配信**: 実行結果の画面情報のみが暗号化されたチャネルを通じてユーザーのローカルブラウザに送られ、表示されます。
6.  **アクション制限**: 管理者設定により、その隔離画面内での「ペースト」「ファイルのダウンロード」「印刷」などの操作を無効化します。

---

## 🎯 試験対策（CCIE Securityラボ試験）

*   **SWG (Secure Web Gateway) の前提条件**: RBI は DNS レイヤの保護だけでは動作しません。AnyConnect トンネルや PAC ファイルを使用して、トラフィックを **Umbrella Proxy (SWG)** に向ける構成が正しくできているか確認してください。
*   **SSL Decryption (復号) の有効化**: HTTPS サイトを隔離するには、Umbrella 側で SSL インスペクションを有効にする必要があります。復号されないトラフィックは、中身をコンテナで処理できないため RBI が適用されません。
*   **Selective Isolation のルール順序**: 試験要件で「特定のカテゴリのみ隔離せよ」と指示された場合、Web Policy のルール内で正しい `Category` を選択し、アクションを `Isolate` に設定する正確な手順が求められます。
*   **Identity の特定**: どの Identity (Network, Roaming Computer, User) に対して RBI を適用するか、試験のトポロジーに基づいた正確な割り当てが必要です。
*   **トラブルシュート**: 隔離されたサイトが正しく表示されない場合、証明書の信頼（Umbrella Root CA）が端末側にインストールされているかを確認してください。

---

## 🛠 設定方法

### 1. Web ポリシーにおける RBI ルールの作成 (Umbrella Dashboard)
1.  **Policies > Web Policies** に移動。
2.  **Add Rule** をクリックし、適切な名前を付ける。
3.  **Identity**: 対象とするユーザーやデバイスを選択。
4.  **Destination**: カテゴリ（例：`Newly Seen Domains`）または特定の URL リストを指定。
5.  **Action**: プルダウンから **Isolate** を選択。
6.  **Advanced Settings**:
    *   `Read-only mode`: 入力制限が必要な場合にチェック。
    *   `File downloads/uploads`: 隔離セッション内でのファイル転送を許可するか設定。

### 2. クライアント側の準備
*   **Umbrella Root CA**: 端末のブラウザに Umbrella のルート証明書をインストールし、SSL 復号時の警告を防止します。

---

## 🔍 検証コマンド

RBI はクラウドベースの機能であるため、デバイスの CLI よりもダッシュボードやブラウザ上での確認が主となります。

| 目的 | 確認方法 |
| :--- | :--- |
| **ポリシー適用状態の確認** | Umbrella Dashboard > **Reporting > Activity Search** で `Isolation` イベントを確認。 |
| **隔離の視覚的確認** | ブラウザで対象サイトを開いた際、右下や枠線に Umbrella Isolation のアイコンが表示されるか確認。 |
| **SWG 到達確認** | `http://proxy.umbrella.com/debug/` にアクセスし、SWG がアクティブか確認。 |
| **復号状態の確認** | ブラウザの証明書情報を開き、発行者が `Cisco Umbrella` になっているか確認。 |

---

## 🚨 トラブルシュート

| 症状 | 原因 | 確認・対処方法 |
| :--- | :--- | :--- |
| サイトが隔離されない | アクションが `Allow` になっている | ポリシーの **Action** が `Isolate` になっているか再確認。 |
| 隔離セッションが重い | レイテンシまたは接続先 | 最寄りの Umbrella データセンターに接続されているか確認。 |
| 入力が一切できない | `Read-only mode` が有効 | ポリシーの高度な設定で入力制限をオフにする。 |
| 証明書エラーが出る | SSL 復号設定と CA 不足 | Umbrella Root CA を端末の信頼されたルートストアに配布。 |
| RBI をバイパスしてしまう | HTTPS 復号がオフ | **SSL Decryption** をポリシー全体で有効にする。 |

---

## ⚠ 制限事項

*   **パフォーマンスへの影響**: 画面情報をストリーミングするため、高解像度の動画視聴や非常に動的な Web アプリケーションでは遅延を感じる場合があります。
*   **ブラウザ互換性**: 極めて古いブラウザでは、隔離セッションのピクセルレンダリングが正しく行われない可能性があります。
*   **対応不可サイト**: 一部の高度な認証を伴うサイトや、特定のプラグイン（Silverlight 等）を要求するレガシーサイトは隔離環境で動作しないことがあります。

---

## 🔄 他技術との関連

*   **3.6.e eStreamer**: 隔離セッション中に発生したセキュリティイベントを、FMC 経由で SIEM 等へ転送可能。
*   **5.1 Cisco AMP**: 隔離セッション内でダウンロードされたファイルは、Umbrella 内蔵の AMP エンジンによってスキャンされます。
*   **4.14 Identity mapping**: AD 連携している場合、特定のユーザーグループに対してのみ RBI を適用する詳細な制御が可能です。

---

## 🧩 比較表

### RBI (Isolate) vs SWG Block vs SWG Allow

| 特徴 | Isolate (隔離) | Block (遮断) | Allow (許可) |
| :--- | :--- | :--- | :--- |
| **リスク許容度** | **中（不透明なサイト）** | 低（危険なサイト） | 高（安全なサイト） |
| **セキュリティ** | 極めて高い（コード実行なし） | 最大（通信自体させない） | デバイスの保護機能に依存 |
| **利便性** | 中（閲覧は可能） | 低（閲覧不可） | 高（制限なし） |
| **用途** | 未評価サイト、SNS等 | C&C, マルウェアサイト | 業務必須サイト |

---

## 💡 ベストプラクティス

1.  **段階的な導入**: まずは `Newly Seen Domains`（新規ドメイン）や `Uncategorized`（未分類）カテゴリに対して RBI を適用し、業務への影響を見極めます。
2.  **証明書の事前配布**: ラボ試験や実環境において、SSL 復号は RBI の前提です。Active Directory の GPO 等を用いてルート証明書を全端末に確実に配布してください。
3.  **Read-Only モードの活用**: フィッシング詐欺が疑われるカテゴリに対しては、RBI を有効にした上で Read-Only モードをオンにし、ID/パスワードの入力を物理的に防ぎます。

---

## 📝 ラボ学習・設定サンプル例

### 1. 新規ドメインの自動隔離
*   **要件**: 生成から 24 時間以内のドメインへのアクセスをすべて隔離せよ。
*   **設定**: Destination Category = `Newly Seen Domains`, Action = `Isolate`.

### 2. 特定の AD グループ専用 RBI ポリシー
*   **要件**: 「Guest-Users」グループの全 HTTP トラフィックを隔離せよ。

### 3. Read-Only 制限の実装
*   **要件**: 掲示板サイト（Forums）へのアクセスは許可するが、投稿（入力）は禁止せよ。
*   **設定**: Action = `Isolate` 且つ `Read-Only mode` = Enable.

### 4. SSL 復号の除外設定
*   **要件**: 隔離ポリシーを維持しつつ、金融関連（Financial Services）サイトのみ復号から除外せよ。

### 5. ファイル転送の禁止
*   **要件**: 隔離セッション内からのファイルのダウンロードをブロックせよ。

### 6. カスタム URL リストの隔離
*   **要件**: 管理者が手動で作成した「Risky-URLs」リストに含まれるサイトを隔離せよ。

### 7. AnyConnect SWG 経由の RBI 検証
*   **要件**: モバイルワーカーが AnyConnect を通じて隔離ポリシーを受けているか確認せよ。

### 8. クリップボード制限
*   **要件**: 隔離されたブラウザからローカル端末へのテキストのコピー＆ペーストを禁止せよ。

### 9. 隔離バナーの表示設定
*   **要件**: ユーザーが隔離されていることを認識できるよう、ブラウザ上部に警告バナーを表示せよ。

### 10. レポートによる隔離統計の抽出
*   **操作**: 過去 7 日間で最も RBI が適用されたカテゴリを特定せよ。

---

## ❓ 想定試験問題

1.  **Design**: 未評価の Web サイトに対して、セキュリティを最大化しつつユーザーの利便性（閲覧）を維持する構成は？
    *   **回答**: Umbrella SIG Web Policy で対象カテゴリに対して **Remote Browser Isolation (RBI)** を適用する。
2.  **トラブルシュート**: Web ポリシーで RBI を設定したが、実際のサイトにアクセスすると通常の `Allow` として処理される。原因として考えられる設定不備は？
    *   **回答**: **HTTPS Decryption (SSL復号)** が無効になっている、または対象 Identity がポリシーのスコープ外である。
3.  **コンフィグ読解**: `Read-only mode` が有効な RBI ルールが適用された場合、ユーザーがフィッシングサイトでパスワードを入力しようとするとどうなるか？
    *   **回答**: キーボード入力がクラウドコンテナでブロックされ、サイト上のフォームに文字が入力されない。
4.  **実装**: RBI セッションにおいて、マルウェアを含むファイルのダウンロードを阻止するための連携機能は？
    *   **回答**: **Cisco AMP (Malware Defense)** を Web ポリシーのインスペクションとして有効にする。
5.  **Design**: 拠点間の WAN 帯域が極端に細い環境で RBI を全ユーザーに適用する際の懸念事項は？
    *   **回答**: **帯域消費の増大**。RBI は画面情報をピクセル転送するため、通常の Web 閲覧よりも多くの帯域を消費する。

---

## 🔗 参考リソース

*   **Cisco Umbrella SIG Guide**: [Remote Browser Isolation (RBI)](https://docs.umbrella.com/deployment-umbrella/docs/get-started-with-remote-browser-isolation)
*   **Cisco Live (BRKSEC-2041)**: [Secure Internet Gateway with Cisco Umbrella](https://www.ciscolive.com/)
*   **SCOR 350-701 Official Cert Guide**: [Chapter 12: Cloud Security](https://www.ciscopress.com/)
*   **Design Guide**: [Best Practices for Cisco Umbrella SIG](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/umbrella-deployment-guide.html)

---

## 📝 **補足（Notes）**

*   **学習メモ**: 「RBI は Web 版の VDI (Virtual Desktop)」と考えると理解が早いです。自分の PC でブラウザを動かすのではなく、サーバー上のブラウザをリモコン操作しているイメージです。
*   **図解**: `User Browser (Remote Control) <---(Pixel Stream)---> Umbrella Container (Execution) <---> Internet (Code)`.
*   **注意点**: ラボ試験では、**証明書のエラー**でサイトが開けない状況に陥りやすいです。Umbrella ダッシュボードから CA をダウンロードし、クライアントにインポートする手順は脊髄反射でできるように練習してください。
