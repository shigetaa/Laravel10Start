# WebPush
WebPush機能を実装する。
## 使用環境
- Laravel
- React
- [Httpsサーバー](../HttpsProxy/Readme.md) (自己証明書不可)
- [laravel-notification-channels/webpush](https://github.com/laravel-notification-channels/webpush)

## 手順
### パッケージインストール
composer を介して `laravel-notification-channels/webpush` をインストールします。
このパッケージを使用すると、Laravelを使用してWebプッシュ通知を簡単に送信できます。

```bash
composer require laravel-notification-channels/webpush
```

`config/app.php` のサービスプロバイダーに追加。
```bash
vim config/app.php
```
```php
'providers' => [
    NotificationChannels\WebPush\WebPushServiceProvider::class,
],
```

Userモデルに `HasPushSubscriptions` トレイトを追加し、プッシュ通知を送れるようにします。
```bash
vim app/Models/User.php
```
```php
use NotificationChannels\WebPush\HasPushSubscriptions;

class User extends Authenticatable
{
    use HasPushSubscriptions;
}
```

### DB準備
インストールしたパッケージのコマンドからマイグレーションファイルを作成します。
```bash
php artisan vendor:publish --provider="NotificationChannels\WebPush\WebPushServiceProvider" --tag="migrations"
```

`/database/migrations` 配下に、`***_create_push_subscriptions_table.php` というマイグレーションファイルが作成されたことを確認してください。
確認したら、下記コマンドでテーブルを作成してください。
```bash
php artisan migrate
```

下記のようなテーブルが作成されます。
```bash
mysql> desc push_subscriptions;
+-------------------+---------------------+------+-----+---------------------+----------------+
| Field             | Type                | Null | Key | Default             | Extra          |
+-------------------+---------------------+------+-----+---------------------+----------------+
| id                | bigint(20) unsigned | NO   | PRI | NULL                | auto_increment |
| subscribable_type | varchar(255)        | NO   |     | NULL                |                |
| subscribable_id   | bigint(20) unsigned | YES  |     | NULL                |                |
| endpoint          | varchar(500)        | NO   | UNI | NULL                |                |
| public_key        | varchar(255)        | YES  |     | NULL                |                |
| auth_token        | varchar(255)        | YES  |     | NULL                |                |
| content_encoding  | varchar(255)        | YES  |     | NULL                |                |
| created_at        | timestamp           | NO   |     | 0000-00-00 00:00:00 |                |
| updated_at        | timestamp           | NO   |     | 0000-00-00 00:00:00 |                |
+-------------------+---------------------+------+-----+---------------------+----------------+
9 rows in set (0.00 sec)
```

### VAPID作成
次のコマンドを使用して **VAPIDキー**（ブラウザー認証に必要）を生成します。
```bash
php artisan webpush:vapid
```
コマンドを実行すると、**VAPID_PUBLIC_KEY** と **VAPID_PRIVATE_KEY** が `.env` に追加されます。
```ini
VAPID_PUBLIC_KEY=*********
VAPID_PRIVATE_KEY=*********
```
次に、configファイルを生成します。
```bash
php artisan vendor:publish --provider="NotificationChannels\WebPush\WebPushServiceProvider" --tag="config"
```

### Laravel側の実装
#### Notificationの登録
Web Push用のNotificationクラスを実装していきます。 下記コマンドで `app/Notifications/webPushEvent.php` というファイルが作成されます。
```bash
php artisan make:notification webPushEvent
```
```bash
vim app/Notifications/webPushEvent.php
```
```php
<?php

namespace App\Notifications;

use Illuminate\Notifications\Notification;
use NotificationChannels\WebPush\WebPushChannel;
use NotificationChannels\WebPush\WebPushMessage;

class webPushEvent extends Notification
{
    public function via($notifiable)
    {
        return [WebPushChannel::class];
    }

    public function toWebPush($notifiable, $notification)
    {
        return (new WebPushMessage)
            ->title('新イベント')
            ->body('新しいイベントが追加されました！');
    }
}
```
#### Controllerの作成
```bash
php artisan make:controller WebPushController
```
```bash
vim app/Http/Controllers/WebPushController.php
```
```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class WebPushController extends Controller
{
    public function __construct() {
        $this->middleware('auth');// 要ログイン
    }
    public function create() {
        return view('web_push.create');
    }
    public function store(Request $request) {
        $this->validate($request, [
            'endpoint'    => 'required',
            'keys.auth'   => 'required',
            'keys.p256dh' => 'required'
        ]);
        $endpoint = $request->endpoint;
        $token = $request->keys['auth'];
        $key = $request->keys['p256dh'];
        $user = $request->user();
        $user->updatePushSubscription($endpoint, $key, $token);
        return response()->json([
            'success' => true
        ], 200);
    }
}
```
なお、この中で重要なのは`$this->middleware('auth');`の部分です。

意味としては「このコントローラーにアクセスする場合、ログインが必須」となりますが、これは`store()`の中で特定のユーザーを取得する必要があるためです。
#### Viewの作成
プッシュ通知を登録するページのビュー（HTML+JavaScript）を作っていきます。
```bash
vim resources/views/web_push/create.blade.php
```
```php
<html>
<body>
    <div id="app">
        <div v-if="processing">処理中...</div>
        <div v-else>
            <button type="button" @click="subscribe" v-if="!isSubscribed">イベントのプッシュ通知を登録する</button>
            <button type="button" @click="unsubscribe" v-else>イベントのプッシュ通知を解除する</button>
        </div>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue@2.6.11"></script>
    <script>

        new Vue({
            el: '#app',
            data: {
                vapidPublicKey: '{{ config('webpush.vapid.public_key') }}',
                registration: null,
                isSubscribed: false,
                processing: false,
                csrfToken: '{{ csrf_token() }}'
            },
            methods: {
                subscribe() {   // プッシュ通知を許可する
                    this.processing = true;
                    const applicationServerKey = this.base64toUint8(this.vapidPublicKey);
                    const options = {
                        userVisibleOnly: true,
                        applicationServerKey: applicationServerKey
                    };
                    this.registration.pushManager.subscribe(options)
                        .then(subscription => {

                            // Laravel側へデータを送信
                            fetch('/web_push', {
                                method: 'POST',
                                body: JSON.stringify(subscription),
                                headers: {
                                    'Accept': 'application/json',
                                    'Content-Type': 'application/json',
                                    'X-CSRF-Token': this.csrfToken
                                }
                            })
                            .then(response => {
                                this.isSubscribed = true;
                                alert('プッシュ通知が登録されました');
                            })
                            .catch(error => {
                                console.log(error);
                            });
                        })
                        .finally(() => {
                            this.processing = false;
                        });
                },
                unsubscribe() { // プッシュ通知を解除する
                    this.processing = true;
                    this.registration.pushManager.getSubscription()
                        .then(subscription => {
                            subscription.unsubscribe()
                                .then(result => {
                                    if(result) {
                                        this.isSubscribed = false;
                                        alert('プッシュ通知が解除されました');
                                    }
                                });
                        })
                        .finally(() => {
                            this.processing = false;
                        });
                },
                base64toUint8(str) {
                    str += '='.repeat((4 - str.length % 4) % 4);
                    const base64 = str
                        .replace(new RegExp('\-', 'g'), '+')
                        .replace(new RegExp('_', 'g'), '/');
                    const binary = window.atob(base64);
                    const binaryLength = binary.length;
                    let uint8Array = new Uint8Array(binaryLength);
                    for(let i = 0; i < binaryLength; i++) {
                        uint8Array[i] = binary.charCodeAt(i);
                    }
                    return uint8Array.buffer;
                }
            },
            mounted() {
                if('serviceWorker' in navigator && 'PushManager' in window) {
                    // Service Workerをブラウザにインストールする
                    navigator.serviceWorker.register('/sw.js')
                        .then(registration => {
                            console.log('Service Worker が登録されました。');
                            this.registration = registration;
                            this.registration.pushManager.getSubscription()
                                .then(subscription => {
                                    this.isSubscribed = !(subscription === null);
                                });
                        });
                } else {
                    console.log('このブラウザは、プッシュ通知をサポートしていません。');
                }
            }
        });
    </script>
</body>
</html>
```
この中で重要なのは以下の３つの部分です。
<details><summary>1. Service Workerを登録する</summary>

プッシュ通知を実装するには、まず `Service Worker` と呼ばれる `JavaScript` をブラウザに登録（インストール）しておいて、プッシュ通知されたり、プッシュ通知がクリックされたときにこのコードが実行されることになります。（Service Workerの本体コードは次の項目で作成します）

そのため、ページが表示された時点で呼ばれる `init()` の中で実行しています。
</details>
<details><summary>2. プッシュ通知を許可する</summary>

**「イベントのプッシュ通知を登録する」** ボタンがクリックされたときに呼ばれる `subscribe()` メソッド内でプッシュ通知が登録されることになります。

この中では、`pushManager.subscribe()` を実行＆許可を得ることで、登録情報を取得し、これをLaravel側に 送信することになります。

なお、送信先は、`WebPushController` の `store()`です。
</details>
<details><summary>3. プッシュ通知を解除する</summary>

すでにプッシュ通知が許可されている場合に表示される **「イベントのプッシュ通知を解除する」** ボタンがクリックされたときに実行されるのが、`unsubscribe()` メソッドです
</details>

#### Service Workerの作成
先ほどの項目で `Service Worker`を登録するコードを書きましたが、まだ `Service Worker`本体のファイルを作成していませんので、ここでつくりましょう。

`public/sw.js` にファイルを作成し、中身を以下のようにしてください。
```bash
vim public/sw.js
```
```javascript
// プッシュ通知された時
self.addEventListener('push', e => {
    const json = e.data.json();
    const title = json.title;
    const options = {
        body: json.body,
        data: {
            url: json.data.url,
        }
    };
    e.waitUntil(
        self.registration.showNotification(title, options)
    );
});
// 通知がクリックされた時
self.addEventListener('notificationclick', e => {
    const data = e.notification.data;
    e.waitUntil(
        clients.openWindow(data.url)
    );
});
```
#### Routesの登録
ルーティングにも追加してください。
```bash
vim routes/web.php
```
```php
Route::get('web_push/create', [WebPushController::class, 'create'])->name('web_push.create');
Route::post('web_push', [WebPushController::class, 'store'])->name('web_push.store');
```



----

#### サブスクリプションの保存/更新
フロントから送られてくるエンドポイントと認証情報を登録する処理を実装します。
($keyと$tokenは、通知を暗号化するために使用されています。)
```bash
```
```php
// ~~ 省略 ~~
public function subscription(Request $request)
{
    $user = \Auth::user();
    $endpoint = $request->endpoint;
    $key = $request->key;
    $token = $request->token;
    // updatePushSubscription($endpoint, $key = null, $token = null, $contentEncoding = null)
    $user->updatePushSubscription($endpoint, $key, $token);
    return ['result' => true];
}
```

#### Notificationの登録
Web Push用のNotificationクラスを実装していきます。 下記コマンドで `app/Notifications/webPushEvent.php` というファイルが作成されます。
```bash
php artisan make:notification webPushEvent
```
```bash
vim app/Notifications/webPushEvent.php
```
```php
<?php
namespace App\Notifications;
use Illuminate\Notifications\Notification;
use NotificationChannels\WebPush\WebPushChannel;
use NotificationChannels\WebPush\WebPushMessage;

class webPushEvent extends Notification
{
    public function via($notifiable)
    {
        return [WebPushChannel::class];
    }

    public function toWebPush($notifiable, $notification)
    {
        return (new WebPushMessage)
            ->title('新しいイベント')
            ->body('新しいイベントが追加されました！');
    }
}
```

### React側の実装
アクセス後下記のようにプッシュ通知の許可を求め、許可をした場合に Service Worker をインストールし、 認証情報とエンドポイントを先ほど実装した /subscription に送信する処理を実装します。
![subscription](https://kudohayatoblog.com/static/e67dc89829dd6176087084ecb05b76b7/8f459/webpush.png)
先ほど作成した VAPID をheadに追加してください。
```html
<meta name="vapidPublicKey" content="{{ config('webpush.VAPID_PUBLIC_KEY') }}">
```
```javascript
import React from 'react';

export default function WebPush(props) {

    // サービスワーカーが使えない系では何もしない
    if ('serviceWorker' in navigator) {
        console.log('service worker サポート対象');
        // サービスワーカーとして、public/sw.js を登録する
        navigator.serviceWorker.register('sw.js')
        .then(function (registration) {
            initialiseServiceWorker();
        })
        .catch(function(error) {
            console.error('Service Worker Error', error)
        })
    } else {
        console.log('service worker サポート対象外');
    }
    /**
     * サービスワーカーを初期化する
     * 初期化では、プッシュ通知用の情報をサーバに送る
     */
    function initialiseServiceWorker() {
        if (!('showNotification' in ServiceWorkerRegistration.prototype)) {
            console.log('cant use notification')
            return
        }
        if (Notification.permission === 'denied') {
            console.log('user block notification')
            return
        }
        if (!('PushManager' in window)) {
            consoleo.log('push messaging not supported')
            return
        }
        // プッシュ通知使えるので
        navigator.serviceWorker.ready.then(registration => {
            registration.pushManager.getSubscription()
            .then(subscription => {
                if (! subscription) {
                    subscribe(registration)
                }
            })
        })
    }
    /**
     * サーバに自身の情報を送付し、プッシュ通知を送れるようにする
     */
    function subscribe(registration) {
        var options = { userVisibleOnly: true }
        //公開鍵をheadに仕込んで取得
        var vapidPublicKey = document.head.querySelector('meta[name="vapidPublicKey"]').content;
        if (vapidPublicKey) {
            options.applicationServerKey = urlBase64ToUint8Array(vapidPublicKey)
        }
        registration.pushManager.subscribe(options)
        .then(subscription => {
            updateSubscription(subscription)
        })
    }
    /**
     * 購読情報を更新する
     *
     */
    function updateSubscription(subscription) {
        var key = subscription.getKey('p256dh')
        var token = subscription.getKey('auth')
        var data = new FormData()
        data.append('endpoint', subscription.endpoint)
        data.append('key', key ? btoa(String.fromCharCode.apply(null, new Uint8Array(key))) : null),
        data.append('token', token ? btoa(String.fromCharCode.apply(null, new Uint8Array(token))) : null)
        // サーバに認証情報とエンドポイントを渡す
        fetch('/subscription', {
            method: 'POST',
            body: data,
        }).then(() => console.log('Subscription ended'))
    }

    function urlBase64ToUint8Array (base64String) {
        var padding = '='.repeat((4 - base64String.length % 4) % 4);
        var base64 = (base64String + padding)
        .replace(/\-/g, '+')
        .replace(/_/g, '/')
        var rawData = window.atob(base64)
        var outputArray = new Uint8Array(rawData.length)
        for (var i = 0; i < rawData.length; ++i) {
            outputArray[i] = rawData.charCodeAt(i)
        }
        return outputArray
    }

    return <div></div>
}
```

### Service Workerの登録
[Service Worker](https://developer.chrome.com/docs/workbox/service-worker-overview/) は、 Webの裏側で動く独立したJavaScript環境です。
Laravelプロジェクトの `public/sw.js` に Service Worker本体を実装していきます。
```bash
vim public/sw.js
```
```javascript
// インストールされたら
self.addEventListener('install', function (e) {
    console.log('ServiceWorker install')
})

// アクティブ化したら
self.addEventListener('activate', function (e) {
    console.log('Serviceworker activated')
})

const WebPush = {
    init() {
        self.addEventListener('push', this.notificationPush.bind(this))  //pushが来る
        self.addEventListener('notificationclick', this.notificationClick.bind(this))  //通知をクリックする
    },
    notificationPush(event) {
        if (!(self.Notification && self.Notification.permission === 'granted')) {
            return
        }
        if (event.data) {
            event.waitUntil(
                this.sendNotification(event.data.json())
            )
        }
    },
    notificationClick(event) {
        console.log('Serviceworker clicked')
    },
    
    sendNotification(data) {
        //プッシュ通知のタイトルとスタイルを指定
        options = {};
        return self.registration.showNotification(data.title, options)
    },
}

WebPush.init()
```
上記では Service Worker をインストール・アクティブ化した際の処理と、プッシュ通知をクリックした際の処理を実装しています。 
プッシュ通知のスタイル(アイコン画像など)は、[showNotification](https://developer.mozilla.org/ja/docs/Web/API/ServiceWorkerRegistration/showNotification) メソッドで指定することができます。

余談ですが、Service Workerは、独立したJavaScript環境であるため、Reactアプリとの連携は原則できません。
データの受け渡しが必要な場合は、indexeddb を使用することで可能となります。

### 動作検証
全ユーザーにプッシュ通知をする機能を実装してみます。
下記コードをルートに追加し https://******/test にアクセスしてみてください。
Userテーブルに登録されているユーザーの中で、push_subscriptions テーブルにも登録されているユーザーにプッシュ通知されます。
```bash
vim routes/web.php
```
```php
Route::get('test', function(){
    $users = User::all();
    foreach($users as $user){
        $user->notify(new EventAdded());
    }
});
```
下のようなプッシュ通知が飛んでくれば成功です！
![notification](https://kudohayatoblog.com/static/365cb192c364ca92da3917e5753c3316/bac1e/webpush2.png)

ここまでお疲れ様でした。
今回の記事はLaravel + Reactで WebPush機能を実装してみようという内容でした。
今回使ったService Workerですが、WebPush以外にも多くの面白い機能がありますので、 そちらも記事にしていけたらなと思います。

参考サイト
[Laravel + ReactでWebPush機能を実装](https://kudohayatoblog.com/blog/LaravelServiceWorker)
[iOS Safari Web Push 検証](https://github.com/Neji-bit/pages/blob/main/index.html)