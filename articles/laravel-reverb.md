---
title: "【Laravel11.x】ReverbでPusherを使わずにリアルタイム通信✨"
emoji: "📡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["laravel", "php", "websocket"]
published: true
published_at: 2024-12-04 07:00
publication_name: "sdb_blog"
---

:::message
[Laravel Advent Calendar 2024](https://qiita.com/advent-calendar/2024/laravel)
[Social Databank Advent Calendar 2024](https://qiita.com/advent-calendar/2024/sdb_blog) の4日目です。
:::

## はじめに
初めまして、
ソーシャルデータバンク株式会社でインターンとして働いているkanoooです。

最新のLaravel11がリリースされてから８ヶ月以上経ちましたが、今回はLaravel11からの新機能「Reverb」について解説したいと思います。

## Reverbとは

今までLaravelでリアルタイム通信（リロード不要のチャットなど）を実装したい時はLaravel単独では実現できず、**Pusher**といったWebSocketサービスを別で用意する必要がありました。
https://pusher.com/

しかし**Laravel11**からの新機能、**Reverb**によってLaravel自身にWebSocketサーバー機能が統合され、外部サービスへ依存せずリアルタイム通信ができるようになりました！🙌

:::details 公式ドキュメントの翻訳
> Laravel Reverb は、超高速でスケーラブルなリアルタイムWebSocket通信をLaravelアプリケーションに直接提供し、Laravelの既存の「イベント」「ブロードキャスト」「ツール」とのシームレスな統合を実現します。`
:::
https://reverb.laravel.com/


## PusherとLaravel Reverbの違い

今まで使っていたPusherと実際何が違うの？どっちがいいの？というのが気になったので簡単にまとめてみました。主観と予想が多めなので間違っていたら教えてください🙏
おそらくどちらがいいということはなく、サービスの要件や規模に合わせて適切なものを選びたいですね！

ちなみに、ReverbとPusherには互換性があるためPusherからReverbへ移行する場合は環境変数あたりの設定の変更だけで済むようです。（laravel-echo, pusher-jsというPusher使用時に使っていたライブラリを同様に使用します。）


| **項目**                 | **Pusher**                                              | **Laravel Reverb**                                       |
|--------------------------|-------------------------------------------------------|-------------------------------------------------------|
| **基本概要**              | リアルタイム通信を簡単に実装できるプラットフォーム | Laravelに内蔵されたWebSocketサーバー |
| **導入の手軽さ**          | サードパーティサービスであり簡単に導入可能。             | 自分でサーバーを構築・管理する必要があるため少し複雑。      |
| **ホスティング**           | Pusherのクラウドサービスを使用。                      | 自分のサーバー（Laravelアプリケーション）内で動作。       |
| **コスト**                 | 無料～有料プラン（無料プランは一日当たり20万メッセージ、同時接続数100まで）          | 無料（ただし、自身のサーバー運用コストが必要）。          |
| **スケーラビリティ**       | クラウドで自動的にスケーリング。                          | サーバー負荷の管理が必要。                     |
| **WebSocketポート**       | Pusherが管理（ポート設定不要）。                       | 自由に設定可能。（デフォルトは`8080`を使用）       |
| **用途**                 | 手間をかけずにリアルタイム機能を導入したい場合など　　   | コストをかけず自前でリアルタイム機能を構築したい場合など　 |
| **自由度**           | 低め（PusherのAPIに依存）             | 高い（詳細設定やログの細かな管理が可能）                   |
| **運用負荷**         | 低い（Pusherが管理）                       | やや高い（設定とメンテナンスが必要）                        |




## 実装デモ
次に実装デモとして簡単なチャット機能を作ろうと思います。
### 手順１. 事前準備
1. Laravelのインストール
    ```bash
    $ composer create-project laravel/laravel:^11.0 reverb-demo
    $ cd reverb-demo
    $ php artisan serve
    ```
    コマンド実行後 [http://127.0.0.1:8000](http://127.0.0.1:8000) でLaravelの画面が表示されるか確認

1. Reverbのインストール
以下のコマンドでブロードキャストを有効化する
    ```bash
    $ php artisan install:broadcasting
    ```
    `Would you like to install Laravel Reverb?`に`yes`を選択して**Reverb**をインストール。
    `Would you like to install and build the Node dependencies required for broadcasting?`に`yes`を選択。すると**laravel-echo**と**pusher-js**というブロードキャストに必要なフロントエンドの依存関係を自動でインストールしてくれます。
    :::message
    デモではフロントエンドもLaravel(blade)で用意するため`yes`で良い。ただしフロントエンドでVueなどを使う場合はここではなくVueプロジェクトのディレクトリでのインストールが必要。
    :::

### 手順２. BroadcastとReverbの設定
1. コマンドでイベントを作成
    ```bash
    $ php artisan make:event MessageEvent
    ```
    下のコードのように`shouldBroadcast`インターフェースを使用するよう追記してブロードキャスト機能を有効化。
    デフォルトで`PrivateChannel`を使用しているが、デモでは認証を使わないため`Channel`に変更する。（channel名も変更しておく）
    :::message
    認証を使用する場合は**PrivateChannel**や**PresenceChannel**を使い、`routes/channels.php`に認証用routeを定義する。
    :::


    ```diff php:MessageEvent.php
     <?php

    namespace App\Events;

    use Illuminate\Broadcasting\Channel;
    use Illuminate\Broadcasting\InteractsWithSockets;
    use Illuminate\Broadcasting\PresenceChannel;
    use Illuminate\Broadcasting\PrivateChannel;
    use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
    use Illuminate\Foundation\Events\Dispatchable;
    use Illuminate\Queue\SerializesModels;

    + class MessageEvent implements shouldBroadcast
    - class MessageEvent
    {
        use Dispatchable, InteractsWithSockets, SerializesModels;

        /**
        * Create a new event instance.
        */
        public function __construct()
        {
            //
        }

        /**
        * Get the channels the event should broadcast on.
        *
        * @return array<int, \Illuminate\Broadcasting\Channel>
        */
        public function broadcastOn(): array
        {
            return [
    +            new Channel('demo-channel'),
    -            new PrivateChannel('channel-name'),
            ];
        }
    }
    ```
1. Routeを用意
    メッセージを送信したらeventが発動するようにします。
    ```diff php:routes/web.php
     <?php

    + use App\Events\MessageEvent;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Route;

    Route::get('/', function () {
        return view('welcome');
    });
    + Route::post('/', function (Request $request) {
    +     event(new MessageEvent($request->input('message')));
    +     return response()->json(['message' => 'success!']);
    + });

    ```

### 手順３. フロント画面の準備
1. 既存の`welcome.blade.php`を以下のように書き換える
    ```diff php:welcome.blade.php
    + <head>
    +     <meta charset="utf-8">
    +     <meta name="viewport" content="width=device-width, initial-scale=1">
    +     <title>Reverb-demo</title>
    +     @vite('resources/js/app.js')
    + </head>
    + <body>
    +     <form>
    +         <input type="text" id="message" name="message">
    +         <button id="send" type="button">送信</button>
    +     </form>
    +     <div>
    +         <div id="messages"></div>
    +     </div>
    + </body>
    ```
2. `bootstrap.js`を以下のように書き換える
    ```diff js:resources/bootstrap.js
    import axios from 'axios';
    window.axios = axios;

    window.axios.defaults.headers.common['X-Requested-With'] = 'XMLHttpRequest';

    import './echo';

    + Echo.channel("demo-channel").listen("MessageEvent", function (e) {
    +     const newMessage = document.createElement("p");
    +     newMessage.textContent = e.message;
    +     const div = document.getElementById("messages");
    +     div.appendChild(newMessage);
    + });

    + document.getElementById("send").addEventListener("click", function () {
    +     const message = document.getElementById("message").value;
    +     if (message === "") return;
    +     axios.post("/", { message: message })
    +         .then(() => {
    +             document.getElementById("message").value = "";
    +         })
    + });
    ```
3. viteサーバーを起動し、`bootstrap.js`を反映させる
    ```bash
    $ npm run dev
    ```

### 手順４. ジョブの確認
実際にワーカーを起動する前にeventメソッドが走り、キューにジョブが保存される様子を見てみましょう。
キューにはRedisも使えますが、デモではデフォルトのままdatabaseを使用します。


1. SQLiteのコマンドラインツールを起動
    :::message
    余談ですがlaravel11からデフォルトのDBがMySQLからSQLiteになりました👍
    :::
    プロジェクトのルートディレクトリで以下を実行：
    ```bash
    $ sqlite3 database/database.sqlite
    ```
1. `jobs`テーブルのデータを確認
    SQLiteのコマンドラインツール内で以下を実行：
    ```sql
    sqlite> select * from jobs;
    ```
    `jobs`テーブルに何もないことを確認する。

1. ブラウザでメッセージを送信

1. 再びSQLiteで`jobs`テーブルを確認
    ```sql
    sqlite> select * from jobs;
    ```
    正常にeventが動いていれば、メッセージ送信後`jobs`テーブルへ新しいデータが追加されているはずです。
    ![sqlite](</images/demo-sqlite.png>)
    *おそらくこんな感じ*


### 手順５. queue workerの起動
いよいよキューに登録されたジョブを順次取り出して実行するワーカーを起動します。

```bash
$ php artisan queue:work
```
これでイベントを処理してReverbに送信できます。

### 手順６. Reverbの起動

WebSocketサーバーを起動し、フロントエンドにイベントを届けます。
```bash
$ php artisan reverb:start
```


### 手順７. ブラウザで動作確認
1. 2つのウィンドウで [http://127.0.0.1:8000](http://127.0.0.1:8000) を開きます。
1. 片方のウィンドウでメッセージを送信します。
1. するともう片方のタブにも**自動的に**送信したテキストが追加されていきます。
![reverb-demo](/images/reverb-demo.gif)




## おわり
以上がLaravel11新機能**Reverb**の簡単な紹介でした。
みなさんもReverbを使ってどんどんリアルタイム通信しちゃいましょう🚀



## 参考
https://reffect.co.jp/laravel/laravel-reverb














