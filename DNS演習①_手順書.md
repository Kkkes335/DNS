# DNS演習① 手順書  
**テーマ：BINDを用いた権威DNSサーバ構築と相互名前解決確認**

---

## 1. 演習概要
2人1組で、各自1台ずつEC2インスタンスを用意し、BINDを使ってDNSサーバを構築する。  
各自でゾーンを作成し、最後にペア相手のDNSサーバへ `dig` コマンドで問い合わせを行い、相互に名前解決できることを確認する。

---

## 2. 演習構成

### 1台目
- ホスト名の例：`ip-172-31-27-21`
- プライベートIP：`172.31.27.21`
- ドメイン名：`yakiniku.entrycl.teamx.local`

### 2台目
- ホスト名の例：`ip-172-31-30-158`
- プライベートIP：`172.31.30.158`
- ドメイン名：`aburi.entrycl.teamx.local`

---

## 3. 事前準備
各EC2インスタンスで以下を実施する。

### 3-1. BINDのインストール
```bash
sudo dnf install -y bind bind-utils
```

### 3-2. セキュリティグループ確認
DNS問い合わせ用として、少なくとも以下を許可する。

- UDP 53
- TCP 53
- SSH 22

---

## 4. 1台目の設定

### 4-1. `/etc/named.conf` の編集
```bash
sudo vi /etc/named.conf
```

以下のように設定する。  
※教科書準拠の書き方に合わせ、`listen-on` はコメントアウトのままとする。

```conf
options {
        # listen-on port 53 { 127.0.0.1;172.31.27.21; };
        # listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { any; };
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "yakiniku.entrycl.teamx.local" IN {
        type master;
        file "/var/named/yakiniku.entrycl.teamx.local.zone";
};
```

### 4-2. 1台目のゾーンファイル作成
```bash
sudo vi /var/named/yakiniku.entrycl.teamx.local.zone
```

以下の内容を記述する。

```zone
$TTL 60
@   IN  SOA ns1.yakiniku.entrycl.teamx.local. root.yakiniku.entrycl.teamx.local. (
2026032701 ; serial
3600 ; refresh
3600 ; retry
3600 ; expire
3600 ) ; mnimum


    IN  NS  ns1.yakiniku.entrycl.teamx.local.

ns1 IN  A  172.31.27.21
www IN  A  172.31.27.21
```

### 4-3. 権限設定
```bash
sudo chown root:named /var/named/yakiniku.entrycl.teamx.local.zone
sudo chmod 640 /var/named/yakiniku.entrycl.teamx.local.zone
sudo restorecon -Rv /var/named
```

### 4-4. 構文確認
```bash
sudo named-checkconf
sudo named-checkzone yakiniku.entrycl.teamx.local /var/named/yakiniku.entrycl.teamx.local.zone
```

### 4-5. named起動
```bash
sudo systemctl enable --now named
sudo systemctl restart named
sudo systemctl status named
```

---

## 5. 2台目の設定

### 5-1. `/etc/named.conf` の編集
```bash
sudo vi /etc/named.conf
```

以下のように設定する。  
※教科書準拠の書き方に合わせ、`listen-on` はコメントアウトのままとする。

```conf
options {
        # listen-on port 53 { 127.0.0.1; };
        # listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { any; };
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "aburi.entrycl.teamx.local" IN {
        type master;
        file "aburi.entrycl.teamx.local.zone";
};
```

### 5-2. 2台目のゾーンファイル作成
```bash
sudo vi /var/named/aburi.entrycl.teamx.local.zone
```

以下の内容を記述する。

```zone
$TTL 60
@ IN SOA ns1.aburi.entrycl.teamx.local. root.aburi.entrycl.teamx.local. (
2026032701 ; serial
3600 ; refresh
3600 ; retry
3600 ; expire
3600 ) ; minimum


    IN  NS  ns1.aburi.entrycl.teamx.local.

ns1 IN  A   172.31.30.158
www IN  A   172.31.30.158
```

### 5-3. 権限設定
```bash
sudo chown root:named /var/named/aburi.entrycl.teamx.local.zone
sudo chmod 640 /var/named/aburi.entrycl.teamx.local.zone
sudo restorecon -Rv /var/named
```

### 5-4. 構文確認
```bash
sudo named-checkconf
sudo named-checkzone aburi.entrycl.teamx.local /var/named/aburi.entrycl.teamx.local.zone
```

### 5-5. named起動
```bash
sudo systemctl enable --now named
sudo systemctl restart named
sudo systemctl status named
```

---

## 6. 動作確認

### 6-1. 1台目で自分のゾーンを確認
```bash
dig @localhost www.yakiniku.entrycl.teamx.local
```

### 6-2. 2台目で自分のゾーンを確認
```bash
dig @localhost www.aburi.entrycl.teamx.local
```

### 6-3. 1台目から2台目へ問い合わせ
```bash
dig @172.31.30.158 www.aburi.entrycl.teamx.local
```

### 6-4. 2台目から1台目へ問い合わせ
```bash
dig @172.31.27.21 www.yakiniku.entrycl.teamx.local
```

### 6-5. 確認ポイント
以下が表示されれば成功とする。

- `status: NOERROR`
- `ANSWER SECTION` に `www` のAレコードが表示される
- 相手のDNSサーバに対して `dig @相手のプライベートIP ...` が成功する

---

## 7. トラブルシュート

### 7-1. namedが起動しない場合
```bash
sudo systemctl status named
sudo journalctl -xeu named.service
```

### 7-2. `named-checkzone` で `file not found` が出る場合
指定したゾーンファイルが存在しないか、ファイル名が誤っている可能性がある。  
`named.conf` の `file` 名と、実際に作成したファイル名が一致しているか確認する。

確認例:
```bash
ls -l /var/named | grep entrycl
```

### 7-3. ペア相手から名前解決できない場合
- セキュリティグループで UDP/TCP 53 が許可されているか
- `allow-query { any; };` になっているか
- ゾーンファイル内のIPアドレスが自分のプライベートIPになっているか
- ドメイン名のスペルミスがないか

### 7-4. ゾーン定義の誤植がある場合
`named` は起動前チェックで停止することがある。  
そのため、**設定変更後は必ず以下を実行してから起動すること。**

```bash
sudo named-checkconf
sudo named-checkzone <ゾーン名> <ゾーンファイル>
```

---

## 8. まとめ
本演習では、2台のEC2にBINDを構築し、以下を確認できた。

- 各自が `master` ゾーンを作成できた
- `named.conf` とゾーンファイルを作成できた
- `named-checkconf`、`named-checkzone` で構文確認できた
- `dig` コマンドで相互に名前解決できた

以上で DNS演習① は完了とする。
