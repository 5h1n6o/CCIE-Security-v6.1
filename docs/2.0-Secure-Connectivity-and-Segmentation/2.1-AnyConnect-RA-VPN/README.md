---
layout: default
title: 2.1-AnyConnect-RA-VPN
nav_order: 1
parent: 2.0-Secure-Connectivity-and-Segmentation
---

# 2.1-AnyConnect-RA-VPN

Cisco AnyConnect（Cisco Secure Client）を使用したリモートアクセスVPNテクノロジーについて、ASA、FTD、およびCiscoルータの比較は以下の通りです。

### **1. プラットフォーム別の主な特徴と役割**

| 特徴 | Cisco ASA | Cisco FTD | Cisco IOS/IOS-XE ルータ |
| :--- | :--- | :--- | :--- |
| **主要プロトコル** | **SSL (TLS/DTLS)** および **IKEv2**。 | **SSL** および **IKEv2**。 | 主に **IKEv2 (FlexVPN)**。 |
| **管理ツール** | **CLI** または **ASDM**。 | **FMC** または **FDM**。 | **CLI**。 |
| **拡張機能** | クライアントレスSSL VPN、二重認証、SCEPプロキシ。 | **次世代機能 (IPS, AMP, AVC)** との統合。 | 動的ルーティング、S2Sとの統合フレームワーク。 |
| **拡張性** | 非常に高く、最大10,000セッションをサポート。 | モデルに依存するが、ASAと同等のLINAエンジンを搭載。 | ISRは通常200セッション程度まで（ASAより限定的）。 |

---

### **2. 各プラットフォームの詳細比較**

#### **Cisco ASA (Adaptive Security Appliance)**
*   **VPNコンセントレータとしての地位:** ASAは長年、VPN専用デバイスとしての標準であり、**SSLおよびIKEv2**の両方を強力にサポートしています。
*   **認証とポリシー:** グループポリシーや接続プロファイル（Tunnel Group）を使用して、非常に詳細なアクセス制御が可能です。
*   **特殊な機能:** ブラウザのみで利用可能な**クライアントレスSSL VPN**を完全にサポートしているのはASAのみであり、他のプラットフォームでは機能が限定的または非推奨となっています。

#### **Cisco FTD (Firepower Threat Defense)**
*   **統合セキュリティ:** FTDはASAの堅牢なVPN機能（LINAエンジン）を継承しつつ、Snortエンジンによる**侵入防御（IPS）やマルウェア防御（AMP）**をVPNトラフィックに直接適用できます。
*   **管理の容易さ:** FMCのウィザードを使用することで、AnyConnectイメージの配布からスプリットトンネルの設定まで、統合的に管理できます。
*   **動的制御:** **RADIUS CoA（Change of Authorization）**をサポートしており、脅威検知時に即座にVPNセッションを隔離する「Rapid Threat Containment」が可能です。

#### **Cisco IOS/IOS-XE ルータ**
*   **FlexVPNフレームワーク:** ルータにおけるAnyConnect実装は、**FlexVPN**に基づいています。これはIKEv2を核とした統合フレームワークで、リモートアクセス、サイト間（S2S）、ハブアンドスポークを同一の設定体系で管理できます。
*   **プロトコルの制限:** 現代のルータはAnyConnect IKEv2が主流であり、**AnyConnect SSLのサポートは限定的**（または一部モデルで終了）になっています。
*   **ユースケース:** 主に小規模拠点や、専用ファイアウォールを配置しない環境、または既存のルータインフラを活用するバックアップ接続として利用されます。

---

### **3. プロトコルの比較 (SSL vs. IKEv2)**

プラットフォームにかかわらず、AnyConnect接続には以下の2つの選択肢があります。

*   **SSL (TLS/DTLS):** 
    *   TCP 443 (TLS) または UDP 443 (DTLS) を使用。
    *   **DTLS**はTLSよりもパフォーマンスに優れ、音声やビデオなどのリアルタイムトラフィックに適しています。
    *   ファイアウォール（HTTPSポート）を通過しやすいため、透過性が高いのが特徴です。
*   **IKEv2 (IPsec):**
    *   IKEv2で交渉し、IPsecでカプセル化します。
    *   強力な暗号化（AES-GCMなど）を標準的に使用し、再接続の耐性（MOBIKE）にも優れています。
    *   ASA、FTD、ルータのすべてで AnyConnect クライアントをサポートしています。

### **4. ライセンスと要件**
*   **ライセンス:** 全プラットフォームにおいて、AnyConnectの利用には **AnyConnect Plus/Apex** または **VPN Only** ライセンスが必要です。
*   **インストール:** 初回のインストールにはエンドポイントの**管理者権限**が必要ですが、以降のアップデートやモジュールの追加は権限なしでも可能です。
