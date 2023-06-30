# Laravel ビルドインサーバーを HTTPS 環境構築
Laravel の ビルドインサーバー `php artisan serve` にて
https 環境を構築する手順をまとめてみました。

## 使用環境
- Laravel
- mkcert によるルート認証局による証明書発行
- ~~自己証明書~~
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
## mkcert 証明書 作業
mkcert を使うと手軽に自己証明書を作成することが出来ます。
### インストール
```bash
yum install nss-tools
cd /usr/local/src/
curl -JLO "https://dl.filippo.io/mkcert/latest?for=linux/amd64"
chmod +x mkcert-v*-linux-amd64
mv mkcert-v*-linux-amd64 /usr/local/bin/mkcert
```
### 証明書保存フォルダーの作成
```bash
mkdir /etc/nginx/ssl
```
### ローカール認証局(CA)のインストール
インストールしたmkcertを使って、以下のコマンドでローカル認証局（CA）を設定します。
```bash
mkcert -install
```
2つのファイルが作成されます
- rootCA-key.pem:暗号化鍵
- rootCA.pem:ルート証明書(CA)

`mkcert -install` で作成された鍵ファイルのありかは以下のコマンドで確認できます。
```bash
mkcert -CAROOT
```
### 証明書と鍵の作成
次に証明書と鍵を作成します。まずは、証明書と鍵を保存するディレクトリに移動します。
```bash
cd /etc/nginx/ssl
```
以下のコマンドで `localhost` 用の証明書と鍵を作成します。以下のコマンドを打つと `localhost.pem` と`localhost-key.pem` というファイルが作成されます。
```bash
mkcert localhost 127.0.0.1 ::1
```
2つのファイルが作成されます
- localhost+2-key.pem：暗号化鍵
- localhost+2.pem：自己署名証明書
これでローカル認証局の設定と証明書と鍵の作成が完了です。



## 自己証明書 作業
上記の **mkcert** を利用の為、自己証明書の作成はパス。

~~自己証明書発行のためには以下のパッケージが必要です。
openssl、openssl-devel、openssl-libs~~
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
        #ssl_certificate /etc/nginx/ssl/nginx.pem;
        ssl_certificate /etc/nginx/ssl/localhost+2.pem
        # 暗号化鍵
        #ssl_certificate_key /etc/nginx/ssl/nginx.key;
        ssl_certificate_key /etc/nginx/ssl/localhost+2-key.pem
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
**mkcert** を利用の為、自己証明の許可設定をパスする。

~~Google Chrome で HTTPS 接続をすると、エラーが表示されることになりますので、認証のチェックを無視するように設定してみます。
Google Chrome 起動時にオプション値を付与して起動するだけです。~~
```bash
google-chrome --allow-insecure-localhost --user-data-dir=/tmp/google-chrome
```

## ローカル認証局のルート証明書(CA)を設定
ローカルネットワーク内の、ローカルホスト以外のブラウザーで、アクセスする場合は
ルート証明書を設定する必要があります。

まずは、設定するデバイスに、`rootCA.pem` をダウンロードしておきます。
### iPhone
1. ルート証明書をiPhoneにインストールする。
[設定] → [プロファイルがダウンロード済み]を選択
プロファイル画面、右上の[インストール]を選択
警告画面、右上の[インストール]を選択
インストール完了画面、右上の[完了]を選択
2. iPhoneでルート証明書を信頼する。
[設定] → [一般] → [情報] → [証明書信頼設定] → ルート証明書を全面的に信頼するの項目で、インストールした証明書をONにします。
[設定] → [一般] → [VPNとデバイス管理] → [構成プロファイル]でインストールしたプロファイルを選択するとルート証明書を確認できます。
### Android
1. 設定アイコン → [セキュリティ] → [暗号化と認証情報] → [認証書のインストール] に移動します。（Androidのバージョンでちょっと違うかもしれません）
2. インストールする証明書のファイル名をタップします。
3. [OK] をタップします。
### Windows Google Chrome
1. 右上の設定ボタンより、[設定]をクリックします。
2. 画面左側の[プライバシーとセキュリティ] → [セキュリティ]をクリックします。
3. [デバイス証明書管理]をクリックします。
4. [信頼されたルート証明機関]タブへ進み、左下の[インポート]をクリックします。
5. 証明書のインポートウィザートが起動します。[次へ]をクリックします。
6. インポートするファイル名を指定し、[次へ]をクリックします。
7. [信頼されたルート証明機関]を選択し、[次へ]をクリックします。
8. セキュリティ警告画面、[はい]をクリックします。
9. [完了]をクリックします。
10. インポート完了です。[OK]をクリックして閉じます。
### Windows Firefox
1. 右上の設定ボタンより、[設定]をクリックします。
2. 画面左側の[プライバシーとセキュリティ] → [証明書を表示]をクリックします。
3. [認証局証明書]タブへ進み、左下の[インポート]をクリックします。
4. 証明書のインポート、[この認証局によるウェブサイトの識別を信頼する]にチェックします → [OK]をクリックします。
5. [OK]をクリックして閉じます。
### Windows Edge
Google Chrome を設定すると、Edge での設定は不要です。