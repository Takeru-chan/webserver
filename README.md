# これは何？
さくらのVPS上でWordPress等を動かすためのWeb Server設定覚書。  

## 環境
<dl>
<dt>VPS</dt><dd>さくらのVPS v4, HDD:100GB, RAM:1GB</dd>
<dt>OS</dt><dd>FreeBSD-10.3-p17</dd>
<dt>Web Server</dt><dd>nginx/1.10.3</dd>
</dl>

## FreeBSDの設定
カスタムOSとして準備されていたFreeBSDをインストールして。。。  

### セキュリティアップデート
まずインストール時点での最新のパッチレベルにする。  

```
freebsd-update fetch
freebsd-update install
```

その後も最新のセキュリティアップデートを追いかけるため/etc/crontabに毎朝３時に確認するよう設定。
これでもしもアップデートがかかったらrootにメールが入る。  

```
0     3     *     *     *     root     /usr/sbin/freebsd-update cron
```

### sshの有効化
/etc/ssh/sshd_configにて設定。  

```
Port 20022                  // 接続ポートを標準外の20022にして攻撃されにくく
Protocol 2                  // ssh2接続のみ許可
PermitRootLogin no          // rootでのログインを拒否
PasswordAuthentication yes  // パスワード認証を許可
PermitEmptyPasswords no     // パスワードなしの接続を拒否
AllowUsers general_user     // ログインユーザーをgeneral_userに限定
```

接続ポートを標準外にしたので接続時はポート指定が必要。  

```
ssh general_user@xxx.vs.sakura.ne.jp -p 20022
```

### ファイアウォールの設定
必要なポートだけを開けるよう/usr/local/etc/ipfw.rulesにて設定。  
- ssh(20022)
- mail(25)
- http(80)
- dns(53)
- ntp(123)
- https(443)

スクリプトの内容は[FreeBSDでファイアウォール(ipfw) | レンタルサーバー・自宅サーバー設定・構築のヒント](http://server-setting.info/freebsd/freebsd_ipfw.html#freebsd_ipfw3)の内容を丸パクリ。  

/etc/rc.confにてファイアウォールを有効化。  

```
firewall_enable="YES"
firewall_script="/usr/local/etc/ipfw.rules"
firewall_logging="YES"
```

くれぐれもsshのポートの開き忘れだけは内容に確認した上でOSを再起動。  

### 不要プロセスの停止
仮想コンソールなどの不要プロセスを停止。
/etc/ttysにてttyv1からttyv7までをoffに設定してから仮想コンソールを再起動。

```
kill -HUP 1
```

## nginxの設定
オフィシャルサイトを参考にnginxをインストール。  

### マルチサイトの設定
まずは80番ポートへのアクセスをhttpsへリダイレクト。
```
server {
  listen       80;
  server_name  .example.com;
  return 301 https://www.example.com$request_uri;
}
```
受け側の433番ポートの設定。
```
server {
  listen       443 ssl http2;
  server_name  www.example.com;
  ssl_certificate         /usr/local/etc/letsencrypt/live/www.example.com/fullchain.pem;
  ssl_certificate_key     /usr/local/etc/letsencrypt/live/www.example.com/privkey.pem;
  ssl_dhparam dhparam.pem;
  ssl_ciphers ECDHE+AESGCM:DHE+AESGCM:HIGH:!aNULL:!MD5;
  ssl_session_cache shared:SSL:1m;
  ssl_session_timeout 5m;
  ssl_prefer_server_ciphers on;

  add_header Strict-Transport-Security "max-age=15768000; includeSubdomains";
  ...

}
```
https化とともにHTTP/2も有効にしておきます。  

SSL/TSLサーバ証明書の取得は次節に譲るとして、参照設定は4,5行目のとおり。
cert.pemではなくfullchain.pemを参照させないとchromeやfirefoxでは危険なサイトと判断されてしまうので注意。  

さらにセキュリティを高めるために6行目以降を記載。  

### https対応
Let's EncryptでSSL/TSLサーバ証明書を発行。
この際、httpsサイトの正規化がうまくできなかったのでSANを利用してwwwあり/なし両対応の証明書を発行。
```
certbot certonly -w /usr/local/www/example -d www.example.com -d example.com
```
その後の証明書更新は/etc/crontabにて。。。
```
0 0 15  * * root  /usr/local/bin/certbot renew
```
