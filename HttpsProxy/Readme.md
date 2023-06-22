# Laravel ビルドインサーバーを HTTPS 環境構築
Laravel の ビルドインサーバー `php artisan serve` にて
https 環境を構築する手順をまとめてみました。

## 使用環境
- Laravel
- 自己証明書
- nginx (リバースプロキシ)

## Laravel 作業
Laravelで必要な設定は、よくあるSSLオフロードを行うロードバランサーの裏にいる場合と同様です。
`app/Http/Middleware/TrustProxies.php` の信頼するプロキシに `$proxies = "*"` を設定します。
```bash
vim app/Http/Middleware/TrustProxies.php
```
```php
<?php
namespace App\Http\Middleware;

use Illuminate\Http\Middleware\TrustProxies as Middleware;
use Illuminate\Http\Request;

class TrustProxies extends Middleware
{
    /**
     * The trusted proxies for this application.
     *
     * @var array<int, string>|string|null
     */
    protected $proxies = "*";

    /**
     * The headers that should be used to detect proxies.
     *
     * @var int
     */
    protected $headers =
        Request::HEADER_X_FORWARDED_FOR |
        Request::HEADER_X_FORWARDED_HOST |
        Request::HEADER_X_FORWARDED_PORT |
        Request::HEADER_X_FORWARDED_PROTO |
        Request::HEADER_X_FORWARDED_AWS_ELB;
}
```
あとは、ビルトインサーバを立ててプロキシからのアクセスを待ち受けましょう
```bash
php artisan serve
```

## 自己証明書 作業
自己証明書発行のためには以下のパッケージが必要です。
openssl、openssl-devel、openssl-libs
### インストール
```bash
yum install openssl openssl-devel openssl-libs
```
### 証明書保存フォルダーの作成
```bash
mkdir /etc/nginx/ssl
```
### 自己証明書を作成
```bash
openssl req -new -x509 -sha256 -newkey rsa:2048 -days 3650 -nodes -out /etc/nginx/ssl/nginx.pem -keyout /etc/nginx/ssl/nginx.key
```
2つのファイルが作成されます
- nginx.key：暗号化鍵
- nginx.pem：自己署名証明書

### 権限の設定
```bash
chown root:root -R /etc/nginx/ssl/
chmod 600 /etc/nginx/ssl/*
chmod 700 /etc/nginx/ssl
```


## Nginx 作業
Nginxでは、アクセスしたいドメインからのリクエストをhttpで動いているビルトインサーバに対してプロキシします。
### インストール
```bash
yum install nginx
```
### 設定ファイル
```bash
vim /etc/nginx/nginx.conf
```
```conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
include /usr/share/nginx/modules/*.conf;
events {
    worker_connections 1024;
}
http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;
    include /etc/nginx/conf.d/*.conf;

    server {
        listen 443 ssl;
        server_name _;

        # 自己署名証明書
        ssl_certificate /etc/nginx/ssl/nginx.pem;
        # 暗号化鍵
        ssl_certificate_key /etc/nginx/ssl/nginx.key;
        # 他の要素は80ポートど同じく設定してください。

        location / {
            proxy_pass          http://127.0.0.1:8000;
            proxy_redirect      default;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header X-Forwarded-Port 443;
            proxy_set_header X-Forwarded-Host $host:$server_port;
            proxy_set_header X-Forwarded-Server $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```
`proxy_pass` はビルトインサーバが起動しているIPとポートに合わせてください。
`proxy_set_header X-Forwarded-Server $host` を入れておかないと、Laravel側でのasset()やroute()系の処理の時に元のドメインを利用してくれないので注意です。

### Nginx 起動
```bash
systemctl restart nginx
```
ブラウザーで、https://localhost/ にアクセスして、ビルドインのページが閲覧出来ていれば 正常に動作してます。

## Google Chrome 自己認証の許可設定
Google Chrome で HTTPS 接続をすると、エラーが表示されることになりますので、認証のチェックを無視するように設定してみます。
Google Chrome 起動時にオプション値を付与して起動するだけです。
```bash
google-chrome --allow-insecure-localhost --user-data-dir=/tmp/google-chrome
```
