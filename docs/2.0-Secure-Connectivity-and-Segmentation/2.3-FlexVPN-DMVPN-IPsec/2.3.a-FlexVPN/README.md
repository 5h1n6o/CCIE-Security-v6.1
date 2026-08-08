# ldapsearch

## 🎯 概要
**ldapsearch** は、LDAP（Lightweight Directory Access Protocol）ディレクトリサーバーに対してクエリを実行し、情報を取得するための強力なコマンドラインツールです。ペネトレーションテストや OSCP においては、Active Directory（AD）環境の技術的列挙（Enumeration）フェーズで極めて重要な役割を果たします。匿名バインド（Null Bind）や低権限ユーザーの資格情報を使用して、ドメイン内のユーザー、グループ、コンピュータ、信頼関係、GPO、さらには説明欄に隠されたパスワードなどの機密情報を抽出するために多用されます。

---

## 🛠 代表的な用途
- **ADオブジェクトの抽出**: ユーザー、グループ、コンピュータアカウントの一覧取得。
- **構成ミス・脆弱な設定の特定**: Pre-authentication 不要なユーザー（AS-REP Roasting 対象）の検索など。
- **情報漏洩の探索**: オブジェクトの属性値（description 等）に保存された認証情報の特定。
- **OSCP での利用ポイント**: ドメインネットワークの初期調査において、ドメインコントローラから内部構造（ドメイン名、ネーミングコンテキスト）を把握し、攻撃対象のユーザーリストを作成するために必須です。

---

## 🔍 基本コマンド
```bash
# ターゲットの命名コンテキスト（Base DN）を特定（匿名接続）
ldapsearch -H ldap://10.10.10.1 -x -s base -b "" "(objectclass=*)" namingContexts

# 簡易認証（-x）と資格情報を使用してドメイン全体を検索
ldapsearch -H ldap://10.10.10.1 -x -D "CN=user,CN=Users,DC=example,DC=com" -w 'password123' -b "DC=example,DC=com" "(objectClass=*)"

# 特定のユーザー属性（例：sAMAccountName）のみを表示
ldapsearch -H ldap://10.10.10.1 -x -b "DC=example,DC=com" "(objectClass=user)" s sAMAccountName
```

---

## 🧪 よく使う攻撃シナリオ
- **匿名ログインによる偵察**: ディレクトリサーバーが匿名バインドを許可している場合、資格情報なしでドメイン内の全ユーザーリストや説明欄の情報を一括取得します。
- **属性値からの資格情報奪取**: `description` や `info` 属性に管理者が誤ってパスワードを記載しているケースを狙い、キーワード検索を実行します。
- **サービスアカウントの特定**: `servicePrincipalName` 属性を持つユーザーを列挙し、Kerberoasting 攻撃の前段階として機能します。

---

## 📌 フェーズ別コマンド例  

### 🔎 Reconnaissance（サービス発見）
```bash
# LDAPサーバーのルートDSEをクエリして基本情報を取得
ldapsearch -H ldap://<TARGET_IP> -x -s base -b "" "(objectclass=*)"

# サーバーがサポートするSASLメカニズムを確認
ldapsearch -H ldap://<TARGET_IP> -x -s base -b "" "(objectclass=*)" supportedSASLMechanisms
```

---

### 🧭 Enumeration（詳細調査）
```bash
# 全てのユーザーオブジェクトを列挙
ldapsearch -H ldap://test.local -b DC=test,DC=local "(objectclass=user)"

# 全てのコンピュータアカウントを列挙
ldapsearch -H ldap://test.local -b DC=test,DC=local "(objectclass=computer)"

# 特定のユーザー名（例：john）の詳細情報を検索
ldapsearch -H ldap://test.local -b DC=test,DC=local "(&(objectclass=user)(name=john))"

# 全てのドメイングループ名を抽出
ldapsearch -H ldap://<IP> -x -b "DC=example,DC=com" "(objectClass=group)" cn

# 各オブジェクトの「説明（description）」フィールドを一括表示（パスワード探索）
ldapsearch -H ldap://<IP> -x -b "DC=example,DC=com" "(objectClass=*)" description
```

---

### 🚪 Initial Access（初期侵入）
```bash
# 説明欄にパスワードらしき文字列が含まれるオブジェクトを検索
ldapsearch -H ldap://<IP> -x -b "DC=example,DC=com" "(description=*pass*)"

# ユーザーアカウントの「管理者コメント」などを検索
ldapsearch -H ldap://<IP> -x -b "DC=example,DC=com" "(adminCount=1)"
```

---

### 🔑 Credential Access（認証情報探索）
```bash
# Pre-authenticationが不要なユーザー（AS-REP Roasting対象）を特定
ldapsearch -H ldap://<IP> -x -b "DC=example,DC=com" "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))" sAMAccountName

# Kerberoastingの対象となるSPN（Service Principal Name）を持つユーザーを列挙
ldapsearch -H ldap://<IP> -x -b "DC=example,DC=com" "(&(objectClass=user)(servicePrincipalName=*))" sAMAccountName servicePrincipalName

# パスワードが期限切れにならない設定のユーザーを特定
ldapsearch -H ldap://<IP> -x -b "DC=example,DC=com" "(userAccountControl:1.2.840.113556.1.4.803:=65536)" sAMAccountName
```

---

### 🏢 Active Directory（AD）
```bash
# ドメイン内のドメインコントローラを列挙
ldapsearch -H ldap://<IP> -x -b "DC=example,DC=com" "(userAccountControl:1.2.840.113556.1.4.803:=8192)"

# 特定のグループ（例：Domain Admins）のメンバーを列挙
ldapsearch -H ldap://<IP> -x -b "DC=example,DC=com" "(memberOf=CN=Domain Admins,CN=Users,DC=example,DC=com)"

# ドメイン間の信頼関係（Trusts）情報を取得
ldapsearch -H ldap://<IP> -x -b "CN=System,DC=example,DC=com" "(objectClass=trustedDomain)"
```

---

## 🧰 便利オプション一覧
- `-x`: 簡易認証（Simple Authentication）を使用。AD環境ではほぼ必須。
- `-H <URL>`: LDAPサーバーのURI（例：`ldap://10.10.10.1`）。
- `-D <BindDN>`: バインドに使用するユーザーの識別名（DN）または `USER@DOMAIN` 形式。
- `-w <PASS>`: パスワードをコマンドラインで指定。
- `-W`: パスワードをプロンプトで対話的に入力。
- `-b <BaseDN>`: 検索を開始するベースDNを指定。
- `-s <Scope>`: 検索範囲を指定（`base`, `one`, `sub`）。デフォルトは `sub`。
- `-LLL`: 結果からLDIFメタデータ（バージョン等）を除外し、純粋なデータのみ出力。

---

## 🚀 実践例
```bash
# 【例1】匿名接続でドメイン名とネーミングコンテキストを素早く特定
ldapsearch -H ldap://192.168.50.70 -x -s base namingContexts

# 【例2】取得した低権限ユーザーを用いて、全ユーザーの説明欄から「pass」を含む行を抽出
ldapsearch -H ldap://192.168.50.70 -x -D "pete@corp.com" -w 'Password123!' -b "DC=corp,DC=com" "(description=*pass*)" description

# 【3】ドメイン内の全コンピュータ名を抽出し、ターゲットリスト（Wordlist）を作成
ldapsearch -H ldap://192.168.50.70 -x -b "DC=corp,DC=com" "(objectClass=computer)" sAMAccountName | grep sAMAccountName | awk '{print $2}'
```

---

## 🧩 他ツールとの組み合わせ
- **impacket** → `GetNPUsers.py` を実行する前に ldapsearch で対象ユーザーが存在するか確認。
- **netexec** → `nxc ldap` モジュールで大まかな情報を得た後、ldapsearch で特定の属性値を深掘り。
- **BloodHound** → LDAP クエリで得た SID 情報を元に、手動で ACL の確認を行う際の補助。

---

## 📝 注意点（OSCP 試験での使い方）
- **ページングの制限**: 大規模なドメインでは一度の検索で全結果が返ってこない場合があります。
- **エスケープ処理**: パスワードに特殊文字（`$` や `!`）が含まれる場合は、シングルクォートで囲むかバックスラッシュでエスケープしてください。
- **Null Bind の確認**: AD サーバーの設定によっては匿名アクセスが完全に禁止されているため、その場合はまず SMB 共有や HTTP サービスからユーザー名とパスワードを1セット入手することに集中してください。

---

## 🔗 参考リンク
- 公式：[OpenLDAP ldapsearch man page](https://linux.die.net/man/1/ldapsearch)
- 良質解説：[LDAP Enumeration Guide (HackTricks)](https://book.hacktricks.xyz/network-services-pentesting/pentesting-ldap)
