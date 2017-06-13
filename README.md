# これは何？
さくらのVPS上でWordPress等を動かすためのWeb Server設定覚書。  

## 環境
<dl>
<dt>VPS</dt><dd>さくらのVPS v4, HDD:100GB, RAM:1GB</dd>
<dt>OS</dt><dd>FreeBSD-10.3-p17</dd>
<dt>Web Server</dt><dd>nginx/1.10.3</dd>
</dl>

FreeBSDの設定を[さくらのVPSにFreeBSDをインストールしてみた ](https://www.junk-works.science/installed-freebsd-in-sakura-vps/)に移行。  

nginxの設定を[さくらのVPSにNginxをインストールしてみた](https://www.junk-works.science/installed-nginx-on-sakura-vps/)に記載。  

Let's Encryptを使ったhttps化とマルチサイトの設定を[Let’s EncryptでSANs対応のドメイン認証をとる](https://www.junk-works.science/domain-certificate-sans-with-lets-encrypt/)に移行。  

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
