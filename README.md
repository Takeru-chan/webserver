# これは何？
さくらのVPS上でWordPress等を動かすためのWeb Server設定覚書。  

## 環境
<dl>
<dt>VPS</dt><dd>さくらのVPS v4, HDD:100GB, RAM:1GB</dd>
<dt>OS</dt><dd>FreeBSD-10.3-p17</dd>
<dt>Web Server</dt><dd>nginx/1.10.3</dd>
</dl>

FreeBSDの設定を[さくらのVPSにFreeBSDをインストールしてみた ](https://www.junk-works.science/installed-freebsd-in-sakura-vps/)に移行。  

## nginxの設定
オフィシャルサイトを参考にnginxをインストール。  

### WordPress用の設定
phpを動かすための設定が9から25行目にあるlocationディレクティブ。  
```
server {
  listen       80;
  server_name  .example.com;
  root   /usr/local/www/example;
  index  index.php index.html index.htm;

  location = /wp-config.php { deny all; }
  location = /xmlrpc.php { deny all; }
  location = /wp-login.php {
    auth_basic "Restricted";
    auth_basic_user_file /home/user/.htpasswd;
    fastcgi_pass unix:/var/run/php-fpm.sock;
    fastcgi_param SCRIPT_FILENAME /usr/local/www/example/$fastcgi_script_name;
    include fastcgi_params;
    fastcgi_param HTTPS on;
  }
  location / {
    try_files $uri $uri/ $uri.php?$args /index.php?&$args;
  }
  location ~ \.php$ {
    fastcgi_index index.php;
    fastcgi_pass unix:/var/run/php-fpm.sock;
    fastcgi_param SCRIPT_FILENAME /usr/local/www/example/$fastcgi_script_name;
    include fastcgi_params;
    fastcgi_param HTTPS on;
  }
}
```
セキュリティを確保するため、7,8行目のようにファイルへのアクセスを禁止。
また9行目のlocationディレクティブに含まれているBasic認証の設定（10,11行目）によってWordPressのログイン画面を表示するための一手間を発生させてログイン画面への攻撃を防止。
.htpasswdファイルはhtpasswdコマンドで作成。

### マルチサイトの設定
nginxはSNIでマルチサイト対応できているのでドメインごとにserverブロックを追記してゆけばＯＫ。
```
server {
  listen       80;
  server_name  .example.com;
  root   /usr/local/www/example;
  index  index.php index.html index.htm;
  ...

}
server {
  listen       80;
  server_name  .example2.com;
  root   /usr/local/www/example2;
  index  index.php index.html index.htm;
  ...

}
server {
  listen       80;
  server_name  .example3.com;
  root   /usr/local/www/example3;
  index  index.php index.html index.htm;
  ...

}
```

### https対応
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

### SSL/TSLサーバ証明書取得
Let's EncryptでSSL/TSLサーバ証明書を発行。
この際、httpsサイトの正規化がうまくできなかったのでSANを利用してwwwあり/なし両対応の証明書を発行。
```
certbot certonly -w /usr/local/www/example -d www.example.com -d example.com
```
その後の証明書更新は/etc/crontabにて。。。
```
0 0 15  * * root  /usr/local/bin/certbot renew
```
