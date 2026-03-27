# DNS演習② 手順書（初学者向け）
**テーマ：権威DNS + WordPress 公開構成を2台で作る**

---

## 1. この演習でやること

この演習では、2台のEC2を使って、次の構成を作ります。

- **1台目**：Apache + PHP-FPM + WordPress
- **2台目**：MariaDB + BIND

そして、実在する `rplearn.net` から払い出された **サブドメイン** を使って、
インターネット上から **ドメイン名で WordPress にアクセスできる状態** を作ります。

さらに、WordPress からデータベースへ接続するときは、  
**IPアドレスではなくホスト名で接続** できるようにします。

---

## 2. ゴール

この演習のゴールは次のとおりです。

- `rplearn.net` 配下のサブドメインの権限移譲を依頼できる
- BINDでそのサブドメインの権威DNSを作れる
- `www` レコードで WordPress 公開サーバへ名前解決できる
- `db` レコードで MariaDB サーバをホスト名で引ける
- WordPress の `DB_HOST` にホスト名を書いて動作させられる
- インターネット上から `http://www.＜サブドメイン＞.rplearn.net` でアクセスできる

---

## 3. 構成

### サーバ構成
- **1台目（Webサーバ）**
  - Apache
  - PHP-FPM
  - WordPress

- **2台目（DB/DNSサーバ）**
  - MariaDB
  - BIND

### レコード要件
- `www` → Webサーバの**パブリックIP**
- `db` → DBサーバの**プライベートIP**

### 注意
今回の演習では、`db` レコードに **プライベートIP** を登録します。  
そのため、外部から `db` を引くとプライベートIPが見える形になります。  
演習としては成立しますが、実務では **内部向けDNS / 外部向けDNS を分ける設計** もよく使います。

---

## 4. 事前に決めること

最初に、チームで以下を決めてメモしてください。

- 払い出してもらいたいサブドメイン名  
  例：`teamx.rplearn.net`
- WebサーバのパブリックIP
- WebサーバのプライベートIP
- DB/DNSサーバのパブリックIP
- DB/DNSサーバのプライベートIP
- 権威DNS名  
  例：`ns1.teamx.rplearn.net`

---

## 5. 権限移譲で必要な情報

今回、講師へ依頼するのは **親ドメイン `rplearn.net` から、自分たちのサブドメインを委譲してもらうこと** です。

### 5-1. 依頼時に必要な情報
最低限、次の情報を伝えられるようにします。

- 払い出してほしいサブドメイン名  
  例：`teamx.rplearn.net`
- そのサブドメインを管理するネームサーバ名  
  例：`ns1.teamx.rplearn.net`
- そのネームサーバのパブリックIP  
  例：`203.0.113.20`

### 5-2. なぜネームサーバ名とIPが必要なのか
親ドメイン側では、子ドメインへ委譲するために **NSレコード** が必要です。  
さらに、今回のように `ns1.teamx.rplearn.net` のような **子ドメイン配下の名前** をネームサーバとして使う場合は、  
親側に **glue レコード（A / AAAA）** も必要になることがあります。

---

## 6. 講師への依頼テンプレート

以下をそのまま使って依頼できます。

```text
件名：rplearn.net サブドメイン権限移譲のお願い

お疲れさまです。
DNS演習②で使用するため、以下サブドメインの権限移譲をお願いいたします。

【希望サブドメイン】
teamx.rplearn.net

【委譲先ネームサーバ】
ns1.teamx.rplearn.net

【ns1 のグローバルIP（glue用）】
203.0.113.20

必要な情報が不足していればご指摘ください。
よろしくお願いいたします。
```

### 6-1. 依頼後に確認したいこと
講師へ依頼したあと、以下を確認します。

- 委譲が完了したか
- どのサブドメイン名で確定したか
- 反映まで少し時間が必要か

---

## 7. セキュリティグループ例

### 7-1. Webサーバ
- TCP 22：管理端末から
- TCP 80：`0.0.0.0/0`
- TCP 443：必要なら `0.0.0.0/0`

### 7-2. DB/DNSサーバ
- TCP 22：管理端末から
- UDP 53：`0.0.0.0/0`
- TCP 53：`0.0.0.0/0`
- TCP 3306：**WebサーバのプライベートIPからのみ**

---

## 8. 2台目（DB/DNSサーバ）の構築

このサーバでは、MariaDB と BIND を入れます。

---

### 8-1. パッケージインストール
```bash
sudo dnf install -y mariadb105-server bind bind-utils
```

### 8-2. MariaDB 起動
```bash
sudo systemctl enable --now mariadb
sudo systemctl status mariadb
```

### 8-3. WordPress用DBとユーザ作成
以下は例です。  
`WEB_PRIVATE_IP` は、WebサーバのプライベートIPに置き換えてください。

```sql
CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'wpuser'@'WEB_PRIVATE_IP' IDENTIFIED BY 'WordPressPass123!';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'WEB_PRIVATE_IP';
FLUSH PRIVILEGES;
```

MariaDBへ入る例:

```bash
sudo mysql
```

### 8-4. MariaDB を外部接続できるようにする
MariaDB が `127.0.0.1` のみ待受になっている場合は、  
DBサーバのプライベートIP もしくは `0.0.0.0` で待受するように修正します。

設定ファイルの例:
```bash
sudo vi /etc/my.cnf.d/mariadb-server.cnf
```

例:
```conf
[mysqld]
bind-address=0.0.0.0
```

設定後に再起動:
```bash
sudo systemctl restart mariadb
sudo systemctl status mariadb
```

---

### 8-5. BIND の設定
```bash
sudo vi /etc/named.conf
```

以下は例です。  
`teamx.rplearn.net` や IP は自分の環境に置き換えてください。

```conf
options {
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { any; };
        recursion no;
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "teamx.rplearn.net" IN {
        type master;
        file "/var/named/teamx.rplearn.net.zone";
};
```

---

### 8-6. ゾーンファイル作成
```bash
sudo vi /var/named/teamx.rplearn.net.zone
```

例:

```zone
$TTL 60
@   IN  SOA ns1.teamx.rplearn.net. root.teamx.rplearn.net. (
        2026032701 ; serial
        3600       ; refresh
        3600       ; retry
        3600       ; expire
        3600 )     ; minimum

    IN  NS  ns1.teamx.rplearn.net.

ns1 IN A <DNSサーバのパブリックIP>
www IN A <WebサーバのパブリックIP>
db  IN A <DBサーバのプライベートIP>
```

### 8-7. レコードの意味
- `ns1`  
  → この子ゾーンを管理するDNSサーバ
- `www`  
  → インターネット公開するWordPressサーバ
- `db`  
  → WordPress が接続するMariaDBサーバ

---

### 8-8. 権限調整
```bash
sudo chown root:named /var/named/teamx.rplearn.net.zone
sudo chmod 640 /var/named/teamx.rplearn.net.zone
sudo restorecon -Rv /var/named
```

### 8-9. 構文確認
```bash
sudo named-checkconf
sudo named-checkzone teamx.rplearn.net /var/named/teamx.rplearn.net.zone
```

### 8-10. BIND 起動
```bash
sudo systemctl enable --now named
sudo systemctl restart named
sudo systemctl status named
```

---

## 9. 1台目（Webサーバ）の構築

このサーバでは、Apache + PHP-FPM + WordPress を構築します。

---

### 9-1. パッケージインストール
```bash
sudo dnf install -y httpd php php-fpm php-mysqlnd php-json php-gd php-mbstring php-xml php-intl php-zip tar wget
```

### 9-2. WordPress 配置
```bash
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo mkdir -p /var/www/wordpress
sudo cp -r wordpress/* /var/www/wordpress/
sudo chown -R apache:apache /var/www/wordpress
sudo find /var/www/wordpress -type d -exec chmod 755 {} \;
sudo find /var/www/wordpress -type f -exec chmod 644 {} \;
```

### 9-3. Apache の VirtualHost 設定
```bash
sudo vi /etc/httpd/conf.d/wordpress.conf
```

例:

```apache
<VirtualHost *:80>
    ServerName www.teamx.rplearn.net
    DocumentRoot /var/www/wordpress

    <Directory /var/www/wordpress>
        AllowOverride All
        Require all granted
    </Directory>

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php-fpm/www.sock|fcgi://localhost/"
    </FilesMatch>

    DirectoryIndex index.php index.html
</VirtualHost>
```

### 9-4. サービス起動
```bash
sudo systemctl enable --now php-fpm
sudo systemctl enable --now httpd
sudo systemctl restart php-fpm
sudo systemctl restart httpd
sudo systemctl status php-fpm
sudo systemctl status httpd
```

---

## 10. WordPress を DBホスト名で接続する

ここが今回の重要ポイントです。

WordPress は `wp-config.php` の `DB_HOST` で、  
**どのデータベースサーバへ接続するか** を決めます。

今回は IPアドレスではなく、DNS名で接続したいので、  
`DB_HOST` に `db.teamx.rplearn.net` を書きます。

### 10-1. `wp-config.php` 作成
```bash
cd /var/www/wordpress
sudo cp wp-config-sample.php wp-config.php
sudo vi wp-config.php
```

以下を修正します。

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'WordPressPass123!' );
define( 'DB_HOST', 'db.teamx.rplearn.net' );
```

### 10-2. まず名前解決できるか確認
```bash
getent hosts db.teamx.rplearn.net
```

### 10-3. DB接続確認
```bash
mysql -h db.teamx.rplearn.net -u wpuser -p -e "SHOW DATABASES;"
```

ここで `wordpress` が見えれば、  
**IPアドレスではなくホスト名でDB接続できている** ことが確認できます。

---

## 11. DNS確認

### 11-1. BINDサーバ自身で確認
```bash
dig @localhost teamx.rplearn.net NS
dig @localhost www.teamx.rplearn.net
dig @localhost db.teamx.rplearn.net
```

### 11-2. Webサーバから確認
```bash
dig www.teamx.rplearn.net
dig db.teamx.rplearn.net
```

### 11-3. インターネット公開確認
委譲が反映したら、次を確認します。

```bash
dig NS teamx.rplearn.net
dig www.teamx.rplearn.net
```

確認ポイント:
- `NS` に `ns1.teamx.rplearn.net` が返る
- `www` に WebサーバのパブリックIP が返る
- `db` に DBサーバのプライベートIP が返る

---

## 12. ブラウザ確認

ブラウザで次へアクセスします。

```text
http://www.teamx.rplearn.net
```

WordPress の初期設定画面が表示されれば成功です。

---

## 13. よくある詰まりどころ

### 13-1. 委譲しても引けない
- 親側でまだNS委譲が反映していない
- glue レコードが不足している
- `ns1` のAレコードのIPが違う

### 13-2. `named` が起動しない
- `named.conf` のゾーン定義に誤字がある
- zoneファイル名が違う
- zoneファイルのドメイン名が違う

確認:
```bash
sudo named-checkconf
sudo named-checkzone teamx.rplearn.net /var/named/teamx.rplearn.net.zone
```

### 13-3. WordPress が DB接続エラーになる
- `DB_HOST` の名前解決ができていない
- MariaDB 側の `bind-address` がローカル限定
- 3306 がセキュリティグループで閉じている
- DBユーザの接続元ホストが違う

確認:
```bash
getent hosts db.teamx.rplearn.net
mysql -h db.teamx.rplearn.net -u wpuser -p
```

### 13-4. `www` でアクセスできない
- `www` がWebサーバのパブリックIPになっていない
- Apache が起動していない
- セキュリティグループで 80/TCP が開いていない

---

## 14. まとめ

この演習では、2台構成で以下を実施します。

- `rplearn.net` 配下サブドメインの権限移譲依頼
- BIND による子ゾーンの権威DNS構築
- `www` を使った WordPress の公開
- `db` を使った MariaDB のホスト名接続
- WordPress の `DB_HOST` にホスト名を設定

つまり今回の演習のポイントは、

- **外部からは `www` でWeb公開できること**
- **内部では `db` というホスト名でDB接続できること**
- **その両方を自分たちの権威DNSで返せること**

です。
