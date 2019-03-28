# リリースノート

- [バージョニング規約](#versioning-scheme)
- [サポートポリシー](#support-policy)
- [Laravel5.8](#laravel-5.8)

<a name="versioning-scheme"></a>
## バージョニング規約

Laravelのバージョニングは、「パラダイム.メジャー・マイナー」の規約を維持しています。メジャーフレームワークリリースは、１月と６月の半年ごとにリリースします。一方、マイナーリリースは毎週のように、頻繁にリリースされます。マイナーリリースは、ブレーキングチェンジを**絶対に**含めません。

アプリケーションやパッケージで、Laravelフレームワークやコンポーネントを利用する場合、常に`5.8.*`のようにバージョンを指定してください。理由は上記の通り、Laravelのメジャーリリースは、ブレーキングチェンジを含んでいるからです。新しいメジャーリリースへの更新は、一日かからない程度になるように努力しています。

パラダイムシフトリリースは数年空けています。これはフレームワークの構造と規約に重要な変更が起きたことを表します。現在、パラダイムシフトリリースは開発されていません。

<a name="support-policy"></a>
## サポートポリシー

Laravel5.5のようなLTSリリースでは、バグフィックスは２年間、セキュリティフィックスは３年間提供します。これらのリリースは長期間に渡るサポートとメンテナンスを提供します。 一般的なリリースでは、バグフィックスは６ヶ月、セキュリティフィックスは１年です。Lumenのようなその他の追加ライブラリでは、最新リリースのみでバグフィックスを受け付けています。

| バージョン | リリース | バグフィックス期限 | セキュリティフィックス期限 |
| --- | --- | --- | --- |
| 5.0 | ２０１５年２月４日 | ２０１５年８月４日 | ２０１６年２月４日 |
| 5.1 (LTS) | ２０１５年５月９日 | ２０１７年６月９日 | ２０１８年６月９日 |
| 5.2 | ２０１５年１２月２１日 | ２０１６年６月２１日 | ２０１６年１２月２１日 |
| 5.3 | ２０１６年８月２３日 | ２０１７年２月２３日 | ２０１７年８月２３日 |
| 5.4 | ２０１７年１月２４日 | ２０１７年７月２４日 | ２０１８年１月２４日 |
| 5.5 (LTS) | ２０１７年８月３０日 | ２０１９年８月３０日 | ２０２０年８月３０日 |
| 5.6 | ２０１８年２月７日 | ２０１８年８月７日 | ２０１９年２月７日 |
| 5.7 | ２０１８年９月４日 | ２０１９年３月４日 | ２０１９年９月４日 |
| 5.8 | ２０１９年２月２６日 | ２０１９年８月２６日 | ２０２０年２月２６日 |

<a name="laravel-5.8"></a>
## Laravel5.8

Laravel5.8は5.7で行われた改善を継続しています。今回のバージョンでは、"HasOneThrough" Eloquentリレーションの導入、規約ベースの認可ポリシーの自動登録の導入、DynamoDBキャッシュとセッションドライバーのサポート、メールバリデーションの改良、スケジューラーのタイムゾーン設定のサポート、ブロードキャストチャンネルに対する複数認可ガード指定のサポート、PSR-16キャッシュドライバー準拠、`artisan serve`コマンドの向上、PHPUnit8.0のサポート、Carbon2.0サポート、Pheanstalk4.0サポートと、様々なバグフィックス並びに不安定の改善が行われました。

### Eloquent `HasOneThrough`リレーション

Eloquentに`hasOneThrough`リレーションタイプのサポートを提供始めました。例として、サプライヤ（Supplier）モデルがアカウント（Account）モデルを一つ持ち（`hasOne`）、アカウントモデルもアカウント履歴（AccountHistory）モデルを一つ持っているとイメージしてください。サプライヤのアカウント履歴にアクセスするために、アカウントモデルを経由し、`hasOneThrough`リレーションが利用できます。

    /**
     * サプライヤに対するアカウント履歴を取得
     */
    public function accountHistory()
    {
        return $this->hasOneThrough(AccountHistory::class, Account::class);
    }

### モデルポリシーの自動検出

Laravel5.7を使う場合は、モデルに対応する[認可ポリシー](/docs/{{version}}/authorization#creating-policies)をアプリケーションの`AuthServiceProvider`で明示的に登録する必要がありました。

    /**
     * アプリケーションに対するポリシーマッピング
     *
     * @var array
     */
    protected $policies = [
        'App\User' => 'App\Policies\UserPolicy',
    ];

Laravel5.8ではモデルポリシーの自動検出が導入され、モデルとポリシーの標準命名規則に従っているポリシーを自動的にLaravelは見つけます。具体的にはモデルが含まれているディレクトリの下に存在する、`Policies`ディレクトリ中のポリシーです。たとえば、モデルが`app`ディレクトリ下にあれば、ポリシーは`app/Policies`ディレクトリへ置く必要があります。さらに、ポリシーの名前は対応するモデルの名前へ、`Policy`サフィックスを付けたものにする必要があります。ですから、`User`モデルに対応させるには、`UserPolicy`クラスと命名します。

独自のポリシー発見ロジックを利用したい場合、`Gate::guessPolicyNamesUsing`メソッドでカスタムコールバックを登録します。通常このメソッドは、`AuthServiceProvider`から呼び出すべきでしょう。

    use Illuminate\Support\Facades\Gate;

    Gate::guessPolicyNamesUsing(function ($modelClass) {
        // ポリシークラス名を返す…
    });

> {note} `AuthServiceProvider`中で明確にマップされたポリシーは、自動検出される可能性のあるポリシーよりも優先的に扱われます。

### PSR-16キャッシュ準拠

より細かい時間でアイテムの保存時間を扱えるように、そしてPSR-16キャッシュ基準に準拠するために、キャッシュの保存時間は分から秒単位になりました。`Illuminate\Cache\Repository`クラスとこれを拡張したクラスの`put`、`putMany`、`add`、`remember`、`setDefaultCacheTime`メソッド、並びに各キャッシュ保存の`put`メソッドはこの動作変更へ更新されました。詳細は、[関係するPR](https://github.com/laravel/framework/pull/27276)をご覧ください。

前述のメソッドに整数値を渡している場合、キャッシュにアイテムを残したい時間が秒数になったことに留意し、コードを変更してください。言い換えれば、アイテムの期限がいつ切れるかを表す`DateTime`インスタンスを渡したほうが良いでしょう。

    // Laravel5.7：30分アイテムは保存される
    Cache::put('foo', 'bar', 30);

    // Laravel5.8：30秒アイテムは保存される
    Cache::put('foo', 'bar', 30);

    // Laravel5.7と5.8：30秒アイテムは保存される
    Cache::put('foo', 'bar', now()->addSeconds(30));

### 複数のブロードキャスト認可ガード

Laravelの以前のリリースでは、プライベートとプレゼンス・ブロードキャスト・チャンネルは、アプリケーションのデフォルト認証ガードにより、ユーザーを認証していました。Laravel5.8からは、受信したリクエストを認可する複数のガードを指定できるようになりました。

    Broadcast::channel('channel', function() {
        // ...
    }, ['guards' => ['web', 'admin']])

### tokenガードとトークンのハッシュ

Laravelの`token`ガードは基本的なAPI認証を提供していますが、SHA-256ハッシュの文字列APIトークンをサポートするようになりました。これにより、保存された平文トークンよりもセキュリティが向上しました。ハッシュ済みトークンの詳細は、[API認証のドキュメント](/docs/{{version}}/api-authentication)で確認してください。

> **Note:** Laravelではシンプルなトークンベースの認証ガードを提供していますが、API認証を提供する堅牢なプロダクションアプリケーションでは、[Laravel Passport](/docs/{{version}}/passport)の使用をAPI認証ではおすすめします。

### メールバリデーションの向上

Laravel5.8では、SwiftMailerが使用している`egulias/email-validator`パッケージを採用し、バリデータのメールバリデーションを向上しました。Laravelの以前のメールバリデーションロジックでは、`example@bär.se`など有効なEメールアドレスを時に無効と判断していました。

### スケジューラのデフォルトタイムゾーン

Laravelでは`timezone`メソッドを使うことで、一つのスケジュールタスクのタイムゾーンをカスタマイズできました。

    $schedule->command('inspire')
             ->hourly()
             ->timezone('America/Chicago');

しかしながら、スケジュール済みの全タスクに対して、同じタイムゾーンを指定する場合、繰り返しの手間がかかっていました。そのため、`app/Console/Kernel.php`ファイルで`scheduleTimezone`メソッドを定義できるようになりました。このメソッドは、スケジュールタスクに対し指定する、デフォルトのタイムゾーンを返します。

    /**
     * スケジュール済み全イベントに対するデフォルトのタイムゾーンを取得
     *
     * @return \DateTimeZone|string|null
     */
    protected function scheduleTimezone()
    {
        return 'America/Chicago';
    }

### 中間テーブル／ピボットモデルイベント

以前のバージョンのLaravelでは、多対多リレーションのカスタム中間テーブル／ピボット（"pivot"）モデルのアタッチ（attach）、デタッチ（detach）、同期（sync）時にEloquentモデルイベントはディスパッチされませんでした。Laravel5.8の[カスタム中間テーブルモデル](/docs/{{version}}/eloquent-relationships#defining-custom-intermediate-table-models)を使用する場合、これらのイベントがディスパッチされるようになりました。

### Artisan callの向上

LaravelではArtisanを`Artisan::call`メソッドにより起動できます。以前のLaravelリリースでは、メソッドの第２引数として、配列でコマンドオプションを渡していました。

    use Illuminate\Support\Facades\Artisan;

    Artisan::call('migrate:install', ['database' => 'foo']);

Laravel5.8では、メソッドの第１引数へ、オプションを含めたコマンド全体を渡せるようになりました。

    Artisan::call('migrate:install --database=foo');

### mock/spyテストヘルパメソッド

モックオブジェクトをより便利にするため、ベースのLaravelテストケースクラスへ新しく`mock`と`spy`メソッドを追加しました。これらのメソッドは自動的にモッククラスをコンテナへ結合します。例をご覧ください。

    // Laravel5.7
    $this->instance(Service::class, Mockery::mock(Service::class, function ($mock) {
        $mock->shouldReceive('process')->once();
    }));

    // Laravel5.8
    $this->mock(Service::class, function ($mock) {
        $mock->shouldReceive('process')->once();
    });

### Eloquentリソースキーの保持

ルートより[Eloquentリソースコレクション](/docs/{{version}}/eloquent-resources)が返されるとき、Laravelはコレクションのキーをシンプルな数値順へリセットします。

    use App\User;
    use App\Http\Resources\User as UserResource;

    Route::get('/user', function () {
        return UserResource::collection(User::all());
    });

Laravel5.8を使用する場合、リソースクラスに`preserveKeys`プロパティを追加し、コレクションのキーの保持をLaravelへ指示してください。以前のLaravelリリースと一貫性を保つために、デフォルトでキーはリセットされます。

    <?php

    namespace App\Http\Resources;

    use Illuminate\Http\Resources\Json\JsonResource;

    class User extends JsonResource
    {
        /**
         * リソースコレクションのキーの保持を指定
         *
         * @var bool
         */
        public $preserveKeys = true;
    }

`preserveKeys`プロパティが`true`のとき、コレクションキーは保持されます。

    use App\User;
    use App\Http\Resources\User as UserResource;

    Route::get('/user', function () {
        return UserResource::collection(User::all()->keyBy->id);
    });

### Higher Order `orWhere` Eloquentメソッド

以前のリリースのLaravelでは、複数のEloquentモデルスコープを結合するために、`or`クエリ操作でクロージャコールバックを使用する必要がありました。

    // scopePopularとscopeActiveメソッドは、Userモデルで定義済み
    $users = App\User::popular()->orWhere(function (Builder $query) {
        $query->active();
    })->get();

Laravel5.8では、"higher order" `orWhere`メソッドが導入され、クロージャを使用することなく、スコープをすらすらとチェーンできます。

    $users = App\User::popular()->orWhere->active()->get();

### Artisan Serveの向上

以前のLaravelリリースでArtisan `serve`コマンドは、ポート`8000`固定でアプリケーションのサーバ動作していました。他の`serve`コマンドプロセスが既にこのポートをリッスンしている場合、次に起動を試みた`serve`は失敗していました。Laravel5.8から複数のアプリケーションを一度にサーバ動作できるように、`serve`は最大で`8009`ポートまで利用可能なポートをスキャンするようになりました。

### Bladeファイルのマッピング

Bladeテンプレートをコンパイルするとき、Laravelはコンパイル済みファイルの先頭へ、オリジナルのBladeテンプレートへのパスを含めるようになりました。

### DynamoDBキャッシュ／セッションドライバ

Laravel5.8から[DynamoDB](https://aws.amazon.com/dynamodb/)キャッシュとセッションドライバが導入されました。DynamoDBはAmazon Web Servicesが提供する、サーバレスNoSQLデータベースです。`dynamodb`キャッシュドライバーのデフォルト設定は、Laravel5.8の[キャッシュ設定ファイル](https://github.com/laravel/laravel/blob/master/config/cache.php)で確認できます。

### Carbon2.0サポート

Laravel5.8は、Carbon日付操作ライブラリの`~2.0`リリースをサポートしました。

### Pheanstalk4.0サポート

Laravel5.8はPheanstalkキューライブラリの`~4.0`リリースをサポートしました。Pheanstalkライブラリをアプリケーションで利用している場合は、Composerで`~4.0`リリースへアップグレードしてください。
