# DNS演習① 手順書（初学者向け）
**テーマ：BINDを使ってDNSサーバを作り、お互いの名前解決を確認する**

---

## 1. この演習でやること

この演習では、2人で1台ずつEC2インスタンスを用意し、  
それぞれのサーバで **DNSサーバ** を作ります。

そのあとで、相手のDNSサーバに向けて `dig` コマンドを実行し、  
**相手が作ったドメイン名を名前解決できること** を確認します。

### この演習のゴール
- DNSサーバを起動できる
- 自分のドメインの設定を書ける
- `dig` コマンドで名前解決できる
- 相手のDNSサーバにも問い合わせできる

---

## 2. 今回の構成

今回は、2台のEC2を使います。

### 1台目
- ホスト名の例：`ip-172-31-27-21`
- プライベートIP：`172.31.27.21`
- ドメイン名：`yakiniku.entrycl.teamx.local`

### 2台目
- ホスト名の例：`ip-172-31-30-158`
- プライベートIP：`172.31.30.158`
- ドメイン名：`aburi.entrycl.teamx.local`

### イメージ
- 1台目は `yakiniku.entrycl.teamx.local` を管理する
- 2台目は `aburi.entrycl.teamx.local` を管理する
- 最後に、お互いのDNSサーバへ問い合わせる

---

## 3. 先に知っておくと分かりやすいこと

DNSは、**名前とIPアドレスを対応付ける仕組み** です。

たとえば、

- `www.yakiniku.entrycl.teamx.local`
- `172.31.27.21`

のように、**名前を見てIPアドレスを返す** のがDNSの役目です。

今回の演習では、その「答えを返す側」のサーバを自分で作ります。

---

## 4. 事前準備

### 4-1. BINDのインストール
BINDは、DNSサーバソフトです。  
まずはこれをインストールします。

```bash
sudo dnf install -y bind bind-utils
```

### 4-2. セキュリティグループ確認
DNSの問い合わせを受けられるように、以下を許可しておきます。

- UDP 53 フルオープン
- TCP 25 フルオープン
- SSH 22 マイIP

念のため開けておく
- TCP 80 マイIP

### 4-3. なぜ53番ポートを開けるのか
DNSは通常、**53番ポート** を使って通信します。  
このポートが閉じていると、相手から問い合わせても応答できません。

---

## 5. DNSサーバの設定ファイルの役割

設定に入る前に、**2つのファイルの役割** を知っておくと理解しやすいです。

### 5-1. `named.conf` とは
`named.conf` は、**DNSサーバ全体の設定ファイル** です。

ここには、たとえば次のようなことを書きます。

- このDNSサーバはどのドメインを管理するか
- そのドメインの詳しい設定ファイルはどこにあるか
- 問い合わせを受け付けるかどうか

言い換えると、**DNSサーバの案内板** のようなものです。

### 5-2. zoneファイルとは
zoneファイルは、**そのドメインの中身を書くファイル** です。

ここには、たとえば次のようなことを書きます。

- `ns1` はどのIPアドレスか
- `www` はどのIPアドレスか
- このドメインを管理するDNSサーバはどれか

言い換えると、**名前とIPアドレスの対応表** です。

### 5-3. 2つの違いをひとことで言うと
- `named.conf`  
  → DNSサーバ全体の設定
- zoneファイル  
  → そのドメインの具体的な中身

---

## 6. 1台目の設定

### 6-1. `/etc/named.conf` を編集する
まず、DNSサーバ全体の設定をします。

```bash
sudo vi /etc/named.conf
```

以下のように設定します。  
※教科書準拠の書き方に合わせ、`listen-on` はコメントアウトのままとします。

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

### 6-2. この設定が意味すること
上の設定では、主に次のことを決めています。

- 問い合わせを受け付ける
- `yakiniku.entrycl.teamx.local` というドメインをこのサーバが管理する
- その中身は `/var/named/yakiniku.entrycl.teamx.local.zone` に書く

つまり、  
**「このサーバは `yakiniku.entrycl.teamx.local` の担当です」**  
と宣言しているイメージです。

### 6-3. 1台目のzoneファイルを作成する
次に、ドメインの中身を書きます。

```bash
sudo vi /var/named/yakiniku.entrycl.teamx.local.zone
```

以下の内容を記述します。

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

### 6-4. このzoneファイルが意味すること
この設定では、次のような内容を書いています。

- このドメインのDNSサーバは `ns1.yakiniku.entrycl.teamx.local.`
- `ns1` のIPアドレスは `172.31.27.21`
- `www` のIPアドレスも `172.31.27.21`

つまり、  
**「この名前で聞かれたら、このIPアドレスを返す」**  
という答えを書いているファイルです。

### 6-5. 権限を設定する(必須ではない)
```bash
sudo chown root:named /var/named/yakiniku.entrycl.teamx.local.zone
sudo chmod 640 /var/named/yakiniku.entrycl.teamx.local.zone
sudo restorecon -Rv /var/named
```

### 6-6. 設定ミスがないか確認する(必須)
起動前に、設定ファイルに誤字や書き間違いがないか確認します。

```bash
sudo named-checkconf
sudo named-checkzone yakiniku.entrycl.teamx.local /var/named/yakiniku.entrycl.teamx.local.zone
```

### 6-7. namedを起動する
```bash
sudo systemctl enable --now named
sudo systemctl restart named
sudo systemctl status named
```

---

## 7. 2台目の設定

### 7-1. `/etc/named.conf` を編集する
2台目も同じ考え方で設定します。

```bash
sudo vi /etc/named.conf
```

以下のように設定します。  
※教科書準拠の書き方に合わせ、`listen-on` はコメントアウトのままとします。

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

### 7-2. 2台目のzoneファイルを作成する
```bash
sudo vi /var/named/aburi.entrycl.teamx.local.zone
```

以下の内容を記述します。

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

### 7-3. 2台目のzoneファイルが意味すること
こちらも考え方は同じです。

- このドメインのDNSサーバは `ns1.aburi.entrycl.teamx.local.`
- `ns1` のIPアドレスは `172.31.30.158`
- `www` のIPアドレスも `172.31.30.158`

### 7-4. 権限を設定する
```bash
sudo chown root:named /var/named/aburi.entrycl.teamx.local.zone
sudo chmod 640 /var/named/aburi.entrycl.teamx.local.zone
sudo restorecon -Rv /var/named
```

### 7-5. 設定ミスがないか確認する
```bash
sudo named-checkconf
sudo named-checkzone aburi.entrycl.teamx.local /var/named/aburi.entrycl.teamx.local.zone
```

### 7-6. namedを起動する
```bash
sudo systemctl enable --now named
sudo systemctl restart named
sudo systemctl status named
```

---

## 8. 動作確認

ここでは、「本当に名前解決できるか」を確認します。

### 8-1. 1台目で自分のDNSを確認する
```bash
dig @localhost www.yakiniku.entrycl.teamx.local
```

### 8-2. 2台目で自分のDNSを確認する
```bash
dig @localhost www.aburi.entrycl.teamx.local
```

### 8-3. 1台目から2台目へ問い合わせる
```bash
dig @172.31.30.158 www.aburi.entrycl.teamx.local
```

### 8-4. 2台目から1台目へ問い合わせる
```bash
dig @172.31.27.21 www.yakiniku.entrycl.teamx.local
```

### 8-5. どこを見れば成功なのか
次の2つが出ていれば、基本的に成功です。

- `status: NOERROR`
- `ANSWER SECTION` に `www` のAレコードが表示される

つまり、  
**「その名前は見つかりました」「返すIPアドレスはこれです」**  
と表示されていれば成功です。

---

## 9. DNS各レコードの種類

最初は全部を覚えなくても大丈夫です。  
まずは、**「この名前に対して何を返したいか」** で考えると分かりやすいです。

### 9-1. SOAレコード
そのゾーンの基本情報を書くレコードです。  
「このドメインの設定の土台」と考えると分かりやすいです。

本演習では、次のように書いています。

```zone
@   IN  SOA ns1.yakiniku.entrycl.teamx.local. root.yakiniku.entrycl.teamx.local. (
2026032701 ; serial
3600 ; refresh
3600 ; retry
3600 ; expire
3600 ) ; minimum
```

### 9-2. NSレコード
そのドメインを管理しているDNSサーバはどれか、を書くレコードです。

```zone
    IN  NS  ns1.yakiniku.entrycl.teamx.local.
```

### 9-3. Aレコード
名前に対してIPv4アドレスを対応付けるレコードです。

```zone
ns1 IN  A  172.31.27.21
www IN  A  172.31.27.21
```

### 9-4. AAAAレコード
名前に対してIPv6アドレスを対応付けるレコードです。

```zone
www IN AAAA 2001:db8::10
```

### 9-5. CNAMEレコード
別名を付けたいときに使うレコードです。

```zone
ftp IN CNAME www.yakiniku.entrycl.teamx.local.
```

### 9-6. MXレコード
メールサーバを指定したいときに使うレコードです。

```zone
@ IN MX 10 mail.yakiniku.entrycl.teamx.local.
```

### 9-7. PTRレコード
IPアドレスから名前を調べる、逆引きで使うレコードです。

```zone
21 IN PTR ns1.yakiniku.entrycl.teamx.local.
```

### 9-8. TXTレコード
文字情報を持たせたいときに使うレコードです。

```zone
@ IN TXT "sample text"
```

### 9-9. SRVレコード
特定のサービスの接続先を示したいときに使うレコードです。

```zone
_ldap._tcp IN SRV 10 5 389 server1.example.local.
```

### 9-10. CAAレコード
証明書を発行できる認証局を制限したいときに使うレコードです。

```zone
@ IN CAA 0 issue "letsencrypt.org"
```

---

## 10. よくある質問

### Q1. `IN NS` の先頭にスペースがある理由と、他のレコードでスペースを空けない理由は何ですか
A1. 先頭にスペースを入れると、左側の名前を省略した書き方になります。  
今回の `NS` レコードは、ゾーンそのものに対する設定なので、`@` を省略して書いています。

```zone
    IN  NS  ns1.yakiniku.entrycl.teamx.local.
```

これは、次とほぼ同じ意味です。

```zone
@   IN  NS  ns1.yakiniku.entrycl.teamx.local.
```

一方で、Aレコードの `ns1` や `www` は、  
**どの名前に対する設定かをはっきり書きたい** ので、左端に名前を書いています。

```zone
ns1 IN  A  172.31.27.21
www IN  A  172.31.27.21
```

---

### Q2. TTL や Serial の数字にはどんな意味がありますか
A2. どちらも、DNSが動くうえで必要な「時間」や「版数」を表しています。

#### TTL
`$TTL 60` の `60` は、**その情報を何秒キャッシュしてよいか** を表します。

```zone
$TTL 60
```

今回は 60 秒なので、設定変更をしたあとも比較的早く反映されやすいです。

#### Serial
SOAレコードの中の `2026032701` は、  
**そのzoneファイルが何回目の版かを表す番号** です。

```zone
2026032701 ; serial
```

よくある考え方は次のとおりです。

- `2026` 年
- `03` 月
- `27` 日
- `01` その日の1回目の更新

zoneファイルを修正したら、通常はこの数字を増やします。

---

### Q3. 各レコードは、どんなときに書けばよいのか分かりません
A3. 最初は、**「どの名前に対して、何を返したいか」** で考えると分かりやすいです。

たとえば、

- このドメインを管理するDNSサーバを書きたい  
  → `NS`
- 名前にIPv4アドレスを対応付けたい  
  → `A`
- 名前にIPv6アドレスを対応付けたい  
  → `AAAA`
- 別名を付けたい  
  → `CNAME`
- メールサーバを指定したい  
  → `MX`

今回の演習で必要だったのは主に次の3つです。

- `SOA`  
  → このドメインの基本情報を書くため
- `NS`  
  → このドメインを管理するDNSサーバを書くため
- `A`  
  → `ns1` や `www` にIPアドレスを対応付けるため

迷ったら、まず  
**「どの名前に対して、何を返したいか」**  
を日本語で書いてから、レコードの種類を選ぶと整理しやすいです。

---

### Q4. 今回はzoneファイルにプライベートIPを書いていますが、パブリックIPではだめですか
A4. だめではありません。  
ただし、**今回の演習ではプライベートIPを書く方が合っています。**

理由は、今回は同じVPC内のEC2同士で、  
内部通信として名前解決を確認しているからです。

```bash
dig @相手のプライベートIPアドレス www.aburi.entrycl.teamx.local
```

このように、相手のプライベートIPへ直接問い合わせているため、  
zoneファイルにもプライベートIPを書く方が自然です。

#### プライベートIPを書くのが向いている場面
- 同じVPC内で使う
- 社内だけで使う
- 演習用の閉じた環境で使う

#### パブリックIPを書くのが向いている場面
- インターネットから使わせたい
- 外部公開サーバとして使いたい

つまり、  
**誰がどこからその名前を使うのか** によって、書くIPアドレスを決めます。

---

### Q5. 何のために `named.conf` とzoneファイルは存在するのですか
A5. ひとことで言うと、  
**`named.conf` はDNSサーバ全体の設定書、zoneファイルは名前とIPアドレスの対応表** です。

#### `named.conf`
DNSサーバ全体の動かし方を書きます。

たとえば、

- どのドメインを管理するか
- zoneファイルはどこにあるか
- 問い合わせを受けるか

を設定します。

今回の例では、次の部分です。

```conf
zone "yakiniku.entrycl.teamx.local" IN {
        type master;
        file "/var/named/yakiniku.entrycl.teamx.local.zone";
};
```

これは、  
**「このサーバは `yakiniku.entrycl.teamx.local` を管理し、その中身はこのファイルに書いてある」**  
という意味です。

#### zoneファイル
ドメインの中身を書きます。

たとえば、

- `ns1` はどのIPアドレスか
- `www` はどのIPアドレスか
- このドメインのDNSサーバはどれか

を書きます。

今回の例では、次の部分です。

```zone
    IN  NS  ns1.yakiniku.entrycl.teamx.local.

ns1 IN  A  172.31.27.21
www IN  A  172.31.27.21
```

つまり、

- `named.conf`  
  → どのドメインを使うか決めるファイル
- zoneファイル  
  → そのドメインで何を返すか書くファイル

という役割の違いがあります。

---

## 11. トラブルシュート

### 11-1. namedが起動しない場合
まずは、サービスの状態を確認します。

```bash
sudo systemctl status named
sudo journalctl -xeu named.service
```

### 11-2. `named-checkzone` で `file not found` が出る場合
zoneファイルが存在しないか、ファイル名が違っている可能性があります。

確認例:

```bash
ls -l /var/named | grep entrycl
```

### 11-3. 相手から名前解決できない場合
次を確認します。

- セキュリティグループで UDP/TCP 53 が許可されているか
- `allow-query { any; };` になっているか
- zoneファイル内のIPアドレスが正しいか
- ドメイン名のスペルミスがないか

### 11-4. 設定を直したのに起動しない場合
`named.conf` または zoneファイルに誤字が残っている可能性があります。  
設定変更後は、必ず次を実行します。

```bash
sudo named-checkconf
sudo named-checkzone <ゾーン名> <ゾーンファイル>
```

---

## 12. まとめ

本演習では、2台のEC2にBINDを構築し、次のことを確認しました。

- 各自が `master` ゾーンを作成できた
- `named.conf` とzoneファイルを作成できた
- `named-checkconf` と `named-checkzone` で設定確認できた
- `dig` コマンドで相互に名前解決できた

以上で、DNS演習①は完了です。
