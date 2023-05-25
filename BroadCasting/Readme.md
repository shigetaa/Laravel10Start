# ブロードキャスト
最近の多くのWebアプリケーションでは、WebSocketを使用して、リアルタイムのライブ更新ユーザーインターフェイスを実装しています。サーバ上で一部のデータが更新されると、通常、メッセージはWebSocket接続を介して送信され、クライアントによって処理されます。
*WebSocket* は、UIに反映する必要のあるデータの変更をアプリケーションのサーバから継続的にポーリングするよりも、効率的な代替手段を提供しています。

たとえば、アプリケーションがユーザーのデータをCSVファイルにエクスポートして電子メールで送信できると想像してください。
ただ、このCSVファイルの作成には数分かかるため、キュー投入するジョブ内でCSVを作成し、メールで送信することを選択したとしましょう。
CSVを作成しユーザーにメール送信したら、イベントブロードキャストを使用して、アプリケーションのJavaScriptで受信する `App\Events\UserDataExported` イベントをディスパッチできます。
イベントを受信したら、ページを更新することなく、CSVをメールで送信済みであるメッセージをユーザーへ表示できます。

こうした機能の構築を支援するため、LaravelはWebSocket接続を介してサーバ側のLaravelイベントを簡単に「ブロードキャスト」できます。
Laravelイベントをブロードキャストすると、サーバ側のLaravelアプリケーションとクライアント側のJavaScriptアプリケーション間で同じイベント名とデータを共有できます。

ブロードキャストの基本的なコンセプトは簡単で、クライアントはフロントエンドで指定したチャンネルへ接続し、Laravelアプリケーションはバックエンドでこれらのチャンネルに対し、イベントをブロードキャストします。
こうしたイベントには、フロントエンドで利用する追加データを含められます。

1. LaravelからEventを使い、Pusherにデータを送信、Pusherが受信
2. Pusherから送られたデータをブラウザで受信
3. Laravel→Pusher→ブラウザにきたデータを描画

## サーバ側インストール
Laravelのイベントブロードキャストの使用を開始するには、Laravelアプリケーション内でいくつかの設定を行い、いくつかのパッケージをインストールする必要があります。

イベントブロードキャストは、**Laravel Echo**(JavaScriptライブラリ)がブラウザクライアント内でイベントを受信できるように、Laravelイベントをブロードキャストするサーバ側ブロードキャストドライバによって実行されます。心配いりません。以降から、インストール手順の各部分を段階的に説明します。

### 設定
アプリケーションのイベントブロードキャスト設定はすべて、`config/broadcasting.php` 設定ファイルに保存します。
Laravelは、すぐに使用できるブロードキャストドライバをいくつかサポートしています。
**Pusherチャンネル**、Redis、およびローカルでの開発とデバッグ用のlogドライバです。
さらに、テスト中にブロードキャストを完全に無効にできるnullドライバも用意しています。
これら各ドライバの設定例は、`config/broadcasting.php` 設定ファイルにあります。

#### ブロードキャストサービスプロバイダ
イベントをブロードキャストする前に、まず `App\Providers\BroadcastServiceProvider` を登録する必要があります。
新しいLaravelアプリケーションでは、`config/app.php` 設定ファイルのproviders配列で、このプロバイダをアンコメントするだけです。
この **BroadcastServiceProvider** には、ブロードキャスト認可ルートとコールバックを登録するために必要なコードが含まれています。
```bash
vim config/app.php
```
コメントアウトする
```php
App\Providers\BroadcastServiceProvider::class,
```

#### キュー設定
また、キューワーカを設定して実行する必要があります。
すべてのイベントブロードキャストはジョブをキュー投入し実行されるため、アプリケーションの応答時間は、ブロードキャストするイベントによる深刻な影響を受けません。

###  Pusherチャンネル
[Pusherチャンネル](https://pusher.com/channels) を使用してイベントをブロードキャストする場合は、Composerパッケージマネージャを使用してPusher Channels PHP SDKをインストールする必要があります。
```bash
composer require pusher/pusher-php-server
```
#### Pusherのアカウント登録
Pusherを利用するためにはアカウント登録を行う必要があります。[Pusher.com](https://pusher.com/) にアクセスして、**Sign up for free** ボタンをクリックします。
1. アカウント情報の入力画面が表示されるので、Github、Googleアカウントもしくはメールアドレスを入力します。
2. メールアドレスを入力した場合は下記のアカウント認証画面が表示されます。メールがPusherから届いているのでメールを確認してアカウント認証を行います。
3. アカウント認証が完了すると製品を選択画面が表示されますので、**Channels** の **Get started** を選んで下さい。アプリケーションの名前をつける画面が表示されるので、任意の名前をつけてください。select a clusterでは **ap3** に設定します。設定が完了したら、Create my app ボタンをクリックしてください。
4. アプリケーションの作成が完了するとクライアント側、サーバ側で使用している言語やフレームワークを選択できる画面が表示されます。今回はクライアント側で **Laravel Echo**、サーバ側で **Laravel** を使用するのでタブでそれぞれを選択してください。
5. 変更するとクライアント側でインストールしなければならないJavaScriptライブラリと受信用のコード、サーバ側ではインストールしなければならないPHPパッケージでPusherの接続に必要な **API KEY** の情報が表示されます。
6. Pusherに接続するための **APP Keys** はApp Keysタグからも確認することができます。

次に、`config/broadcasting.php` 設定ファイルでPusherチャンネルの利用資格情報を設定する必要があります。Pusherチャンネル設定の例は予めこのファイルに含まれているため、キー、シークレット、およびアプリケーションIDを簡単に指定できます。
通常、これらの値は、**PUSHER_APP_ID**、**PUSHER_APP_KEY**、**PUSHER_APP_SECRET** 環境変数 を介して設定する必要があります。

```bash
vim .env
```
```ini
PUSHER_APP_ID=
PUSHER_APP_KEY=
PUSHER_APP_SECRET=
PUSHER_SCHEME=https
PUSHER_APP_CLUSTER=ap3
```

`config/broadcasting.php` ファイルのpusher設定では、クラスターなどチャンネルでサポートされている追加のoptionsを指定することもできます。

次に、`.env`ファイルでブロードキャストドライバを **pusher** に変更する必要があります。
```bash
vim .env
```
```ini
BROADCAST_DRIVER=pusher
```

## クライアント側インストール
クライアント側でブロードキャストイベントを受信する **Laravel Echo** をインストールして設定を行って行きます。
###  Pusherチャンネル
**Laravel Echo** はJavaScriptライブラリであり、チャンネルをサブスクライブし、サーバ側のブロードキャストドライバがブロードキャストしたイベントを簡単にリッスンできます。
NPMパッケージマネージャを介してEchoをインストールします。
以下の例では、Pusher Channelsブロードキャスタを使用するため、**pusher-js** パッケージもインストールしています。
```bash
npm install --save-dev laravel-echo pusher-js
```

Echoをインストールできたら、アプリケーションのJavaScriptで新しいEchoインスタンスを生成する用意が整います。
これを実行するのに最適な場所は、Laravelフレームワークに含まれている `resources/js/bootstrap.js` ファイルの下部です。
デフォルトで、Echo設定の例はすでにこのファイルに含まれています。アンコメントするだけです
```bash
vim resources/js/bootstrap.js
```
```javascript
import Echo from 'laravel-echo';

import Pusher from 'pusher-js';
window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER ?? 'mt1',
    wsHost: import.meta.env.VITE_PUSHER_HOST ? import.meta.env.VITE_PUSHER_HOST : `ws-${import.meta.env.VITE_PUSHER_APP_CLUSTER}.pusher.com`,
    wsPort: import.meta.env.VITE_PUSHER_PORT ?? 80,
    wssPort: import.meta.env.VITE_PUSHER_PORT ?? 443,
    forceTLS: (import.meta.env.VITE_PUSHER_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
});
```
アンコメントし、必要に応じEcho設定を調整したら、アプリケーションのアセットをコンパイルします。
```bash
npm run dev
```

### イベント作成
イベントの作成は、`php arisan make:event イベント名` コマンドで行います。タスクが追加されたという情報を送信したいので、イベント名は `TaskAdded` とします。
```bash
php artisan make:event TaskAdded
```
`app/Events/TaskAdded.php` ファイルが作成されます。
Broadcastingの設定を行うのでTaskAdded.phpファイルを開きます。
Broadcastingを行うために TaskAddedクラスに **ShouldBroadcast** インターファイスを継承させる必要があります。
```bash
vim app/Events/TaskAdded.php
```
```php
class TaskAdded implements ShouldBroadcast
```
`ShouldBroadcast` インターファイスの中には `broadcastOn` メソッドのみ記述されています。

作成した `TaskAdded` にも `broadcastOn()` メソッドが実装されています。
`PrivateChannel` を使っていますが、`Channel` クラスに変更します。
チャネル名を `task-added-channel` に変更します。

`PrivateChannel` ではユーザの認証が必要なので、ユーザ認証の必要がない `Channel` を使っています。
```bash
vim app/Events/TaskAdded.php
```
```php
class TaskAdded implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public $task;
    public function __construct($task)
    {
        $this->task = $task;
    }

    public function broadcastOn(): array
    {
        return [
            new Channel('task-added-channel',$this->task),
        ];
    }
}
```

*web.php* を開いて、*/tasks* ルーティングを追加し、作成した `TaskAdded` イベントを下記のようにeventヘルパー関数で発行させます。
ブラウザで */tasks* にアクセスすると `TaskAdded` イベントが発行されます。
```bash
vim app/routes/web.php
```
```php
use App\Events\TaskAdded;

Route::get('/tasks', function () {
	$task = ['id' => 1, 'name' => 'メールの確認'];
    event(new TaskAdded($task));
});
```
### Pusher にデータを送信する
`php artisan serve` コマンドで開発サーバを起動します。
```bash
php artisan serve
```
ブラウザで `/tasks` にアクセスしましょう。
Pusher の **Debug Console** に追加した `$task` の情報が表示されたら、情報の追加も正常に行われています。

### Pusher からデータを送信する
#### JavaScriptファイルでの受信設定
Pusherからのメッセージが受け取れるようにするために `bootstrap.js` で追加設定を行う必要があります。
channelメソッドにはイベントTaskAddedで設定した**チャネル名**、
listenメソッドには、**イベント名**を指定します。
Pusherからのデータはdataに入ってくるので、console.logコマンドでコンソールにメッセージでPusherから送られてくるデータを表示させます。
```bash
vim resources/js/bootstrap.js
```
```javascript
window.Echo.channel('task-added-channel').listen('TaskAdded',function(data){
    console.log('received a message');
    console.log(data);
});
```
#### Pusher からデータを送信する
Pusher の **Debug Console** の **Event creator** から送信を実行します。
**Channel** `task-added-channel`
**Event** `App\Events\TaskAdded`
**Data** `{"task": {"id": 1,"name": "メールの確認 1"}}`
Send event ボタンを実行して、ブラウザーコンソールにDataが表示されればブラウザーで受信が成功です。


## まとめ
Laravelのイベントブロードキャストを使用すると、WebSocketに対するドライバベースのアプローチを使用して、サーバ側のLaravelイベントをクライアント側のJavaScriptアプリケーションへブロードキャストできます。
現在、LaravelはPusherチャンネルとAblyドライバを用意しています。
イベントは、Laravel Echo JavaScriptパッケージを用い、クライアント側で簡単に利用できます。

イベントは「チャンネル」を介してブロードキャストされます。
チャンネルは、パブリックまたはプライベートとして指定できます。
アプリケーションへの訪問者は全員、認証や認可なしにパブリックチャンネルをサブスクライブできます。
ただし、プライベートチャンネルをサブスクライブするには、ユーザーが認証され、そのチャンネルをリッスンする認可を持っている必要があります。