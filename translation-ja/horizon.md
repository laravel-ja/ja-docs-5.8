# Laravel Horizon

- [イントロダクション](#introduction)
- [Horizonのアップグレード](#upgrading)
- [インストール](#installation)
    - [設定](#configuration)
    - [ダッシュボードの認可](#dashboard-authorization)
- [Horizonの実行](#running-horizon)
    - [Horizonのデプロイ](#deploying-horizon)
- [タグ](#tags)
- [通知](#notifications)
- [メトリックス](#metrics)

<a name="introduction"></a>
## イントロダクション

Horizon（水平線、展望）は、Laravelで動作するRedisキューのための、美しいダッシュボードとコード駆動による設定を提供します。Horizonにより、ジョブのスループット、ランタイム、実行の失敗など、キューシステムのキーメトリックを簡単に監視できます。

一つのシンプルな設定ファイルにすべてのワーカ設定を保存するため、チーム全体がコラボレート可能なソースコントロール下に、設定を保持できます。

<p align="center">
<img src="https://res.cloudinary.com/dtfbvvkyp/image/upload/v1537195039/photos/Test.png">
</p>

<a name="installation"></a>
## インストール

> {note} `queue`設定ファイルで、`redis`をキューコネクションへ確実に指定してください。

Composerを使い、LaravelプロジェクトにHorizonをインストールします。

    composer require laravel/horizon

Horizonをインストールしたら、`horizon:install` Artisanコマンドを使用し、アセットを公開します。

    php artisan horizon:install

Laravelが[失敗したキュー済みのジョブ](/docs/{{version}}/queues#dealing-with-failed-jobs):を保存するために使用する、`failed_jobs`を生成することもできます。

    php artisan queue:failed-table

    php artisan migrate

<a name="upgrading"></a>
#### Horizonのアップグレード

Horizonの新しいメジャーバージョンへアップグレードする場合は、[アップグレードガイド](https://github.com/laravel/horizon/blob/master/UPGRADE.md)を注意深く読み込むことが重要です。

付け加えて、Horizonのアセットを再公開も必要です。

    php artisan horizon:assets

<a name="configuration"></a>
### 設定

Horizonのアセットを公開すると、`config/horizon.php`に一番重要な設定ファイルが設置されます。この設定ファイルにより、ワーカのオプションを設置します。各オプションにはその目的が説明されていますので、ファイル全体をしっかりと確認してください。

> {note} Horizonを実行する予定の環境ごとのエントリーを`horizon`設定ファイルの`environments`部分へ確実に含めてください。

#### バランスオプション

Horizonでは３つのバランシング戦略が選択できます。`simple`と`auto`、`false`です。`simple`戦略は設定ファイルのデフォルトで、投入されたジョブをプロセス間に均等に割り当てます。

    'balance' => 'simple',

`auto`戦略は、現在のキュー負荷に基づき、それぞれのキューへ割り当てるワーカプロセス数を調整します。たとえば、`notifications`キューに１，０００ジョブが溜まっており、一方で`render`キューが空の場合、Horizonは空になるまで`notifications`キューにより多くのワーカを割り当てます。`balance`オプションへ`false`を設定すると、設定にリストした順番でキューが処理される、Laravelデフォルトの振る舞いが使われます。

`auto`戦略を使う場合、Horizonがスケールアップ／ダウンで使用すべきプロセス数の最小値と最大値をコントロールするために、`minProcesses`と`maxProcesses`設定オプションを定義してください。

    'environments' => [
        'production' => [
            'supervisor-1' => [
                'connection' => 'redis',
                'queue' => ['default'],
                'balance' => 'auto',
                'minProcesses' => 1,
                'maxProcesses' => 10,
                'tries' => 3,
            ],
        ],
    ],

#### ジョブの整理

`horizon`設定ファイルで、現在がどのくらいの長さなのか、それと失敗したジョブをどのくらい保持しているかを分数で設定できます。デフォルトでは、現在のジョブは１時間、失敗したジョブは１週間保持されます。

    'trim' => [
        'recent' => 60,
        'failed' => 10080,
    ],

<a name="dashboard-authorization"></a>
### ダッシュボードの認可

Horizonは、`/horizon`でダッシュボードを表示します。デフォルトでは`local`環境でのみ、このダッシュボードへアクセスできます。`app/Providers/HorizonServiceProvider.php`ファイルの中に、`gate`メソッドが存在しています。この認可ゲートは**local以外**の環境における、Horizonへのアクセスをコントロールします。Horizonへのアクセスを必要に応じ制限するために、自由に変更してください。

    /**
     * Horizonゲートの登録
     *
     * このゲートはlocal以外の環境で、誰がHorizonへアクセスできるか決定している。
     *
     * @return void
     */
    protected function gate()
    {
        Gate::define('viewHorizon', function ($user) {
            return in_array($user->email, [
                'taylor@laravel.com',
            ]);
        });
    }

> {note} LaravelはGateへ自動的に**認証済み**ユーザーを依存注入します。IP制限のような別のHorizonセキュリティ方法をアプリケーションで提供する場合は、Horizonユーザーは「ログイン」している必要はいらないでしょう。そのため、上記の`function ($user)`を`function ($user = null)`へ変更し、Laravelに認証は必要ないと強制的に知らせてください。

<a name="running-horizon"></a>
## Horizonの実行

`config/horizon.php`設定ファイルでワーカの設定を済ませたら、`horizon` Artisanコマンドを使用し、Horizonを使用開始します。このコマンド一つで、設定済みのワーカ全部を起動できます。

    php artisan horizon

Horizonプロセスを`horizon:pause` Artisanコマンドで一時停止したり、`horizon:continue`コマンドで処理を続行したりできます。

    php artisan horizon:pause

    php artisan horizon:continue

マシン上のマスタHorizonプロセスを穏やかに終了させたい場合は、`horizon:terminate` Artisanコマンドを使用します。現在処理中のジョブが完了した後に、Horizonは停止します。

    php artisan horizon:terminate

<a name="deploying-horizon"></a>
### Horizonのデプロイ

Horizonを実働サーバにデプロイする場合、`php artisan horizon`コマンドをプロセスモニタで監視し、予期せず終了した場合には再起動をかけるように設定する必要があります。サーバに新しいコードをデプロイしたときに、Horizonプロセスを停止指示する必要があります。その結果、プロセスモニタにより再起動され、コードの変更が反映されます。

#### Supervisor設定

`horizon`プロセスを管理するため、Supervisorプロセスモニタを使用する場合は、以下の設定ファイルが利用できるでしょう。

    [program:horizon]
    process_name=%(program_name)s
    command=php /home/forge/app.com/artisan horizon
    autostart=true
    autorestart=true
    user=forge
    redirect_stderr=true
    stdout_logfile=/home/forge/app.com/horizon.log

> {tip} サーバ管理に自信がない場合は、[Laravel Forge](https://forge.laravel.com)の利用を検討してください。ForgeはHorizonと共に、モダンで堅牢なLaravelアプリケーションに必要なすべてをPHP7以上のサーバにプロビションします。

<a name="tags"></a>
## タグ

Horizonでは、mailableやイベントブロードキャスト、通知、キューイベントリスナなどを含むジョブに「タグ」を割り付けられます。実際、ジョブに割り付けられたEloquentモデルに基づいて、ほとんどのジョブでは賢く自動的にHorizonがタグ付けします。例として、以下のジョブをご覧ください。

    <?php

    namespace App\Jobs;

    use App\Video;
    use Illuminate\Bus\Queueable;
    use Illuminate\Queue\SerializesModels;
    use Illuminate\Queue\InteractsWithQueue;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Foundation\Bus\Dispatchable;

    class RenderVideo implements ShouldQueue
    {
        use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

        /**
         * ビデオインスタンス
         *
         * @var \App\Video
         */
        public $video;

        /**
         * 新しいジョブインスタンスの生成
         *
         * @param  \App\Video  $video
         * @return void
         */
        public function __construct(Video $video)
        {
            $this->video = $video;
        }

        /**
         * ジョブの実行
         *
         * @return void
         */
        public function handle()
        {
            //
        }
    }

`id`が`1`の`App\Video`インスタンスを持つジョブがキューされると、自動的に`App\Video:1`タグが付けられます。HorizonはジョブのプロパティがEloquentモデルであるかを確認するからです。Eloquentモデルが見つかると、Horizonはモデルのクラス名と主キーを使用し、賢くタグ付けします。

    $video = App\Video::find(1);

    App\Jobs\RenderVideo::dispatch($video);

#### 手動のタグ付け

queueableオブジェクトのタグを任意に定義したい場合は、そのクラスで`tags`メソッドを定義してください。

    class RenderVideo implements ShouldQueue
    {
        /**
         * ジョブに割り付けるタグの取得
         *
         * @return array
         */
        public function tags()
        {
            return ['render', 'video:'.$this->video->id];
        }
    }

<a name="notifications"></a>
## 通知

> **Note:** Horizonから、SlackかSMS通知を送る設定を行う場合は、[対応するドライバの動作要件](/docs/{{version}}/notifications)についても、確認する必要があります。

あるキューが長時間waitしている時に、通知を受け取りたい場合は、`Horizon::routeSlackNotificationsTo`や、`Horizon::routeSlackNotificationsTo`、`Horizon::routeSmsNotificationsTo`メソッドを利用してください。これらのメソッドは、`HorizonServiceProvider`から呼び出すのが良いでしょう。

    Horizon::routeMailNotificationsTo('example@example.com');
    Horizon::routeSlackNotificationsTo('slack-webhook-url', '#channel');
    Horizon::routeSmsNotificationsTo('15556667777');

#### 通知wait時間のシュレッドホールド設定

何秒を「長時間」と考えるかは、`config/horizon.php`設定ファイルで指定できます。このファイルの`waits`設定オプションで、接続／キューの組み合わせごとに、長時間と判定するシュレッドホールドをコントロールできます。

    'waits' => [
        'redis:default' => 60,
    ],

<a name="metrics"></a>
## メトリックス

Horizonはジョブとキューの待ち時間とスループットの情報をダッシュボードに表示します。このダッシュボードを表示するために、アプリケーションの[スケジューラ](/docs/{{version}}/scheduling)で、５分毎に`snapshot` Artisanコマンドを実行する設定を行う必要があります。

    /**
     * アプリケーションのコマンドスケジュールの定義
     *
     * @param  \Illuminate\Console\Scheduling\Schedule  $schedule
     * @return void
     */
    protected function schedule(Schedule $schedule)
    {
        $schedule->command('horizon:snapshot')->everyFiveMinutes();
    }
