# アップグレードガイド

- [5.7から5.8.0へのアップグレード](#upgrade-5.8.0)

<a name="high-impact-changes"></a>
## 重要度の高い変更

<div class="content-list" markdown="1">
- [キャッシュ持続時間が秒指定に](#cache-ttl-in-seconds)
- [キャッシュロックの安全性向上](#cache-lock-safety-improvements)
- [Markdownファイルのディレクトリ変更](#markdown-file-directory-change)
- [Nexmo／Slack通知チャンネル](#nexmo-slack-notification-channels)
</div>

<a name="medium-impact-changes"></a>
## 重要度が中程度の変更

<div class="content-list" markdown="1">
- [コンテナジェネレーターとタグ付けサービス](#container-generators)
- [SQLiteバージョン制約](#sqlite)
- [ヘルパから文字列と配列クラスへ](#string-and-array-helpers)
- [遅延サービスプロバイダ](#deferred-service-providers)
- [PSR-16準拠](#psr-16-conformity)
- [不規則変化する複数形のモデル名](#model-names-ending-with-irregular-plurals)
- [IDが増加するカスタムピボットモデル](#custom-pivot-models-with-incrementing-ids)
- [Pheanstalk4.0](#pheanstalk-4)
</div>

<a name="upgrade-5.8.0"></a>
## 5.7から5.8.0へのアップグレード

#### アップグレードの見積もり時間：１時間

> {note} 私達は、互換性を失う可能性がある変更を全部ドキュメントにしようとしています。しかし、変更点のいくつかは、フレームワークの明確ではない部分で行われているため、一部の変更が実際にアプリケーションに影響を与えてしまう可能性があります。

<a name="updating-dependencies"></a>
### 依存パッケージのアップデート

`composer.json`ファイル中の`laravel/framework`パッケージのバージョンを`5.8.*`にアップデートしてください。

次に、アプリケーションで使用している３ｒｄパーティーパッケージを調査し、Laravel5.8をサポートしているバージョンを使用していることを確認してください。

<a name="the-application-contract"></a>
### `Application`契約

#### `environment`メソッド

**影響の可能性： とても低い**

`Illuminate/Contracts/Foundation/Application`契約で利用法が規定されている`environment`メソッドは[変更されました](https://github.com/laravel/framework/pull/26296)。アプリケーションでこの契約を実装している場合は、メソッドの使用法を更新してください。

    /**
     * アプリケーション環境の取得もしくは判定
     *
     * @param  string|array  $environments
     * @return string|bool
     */
    public function environment(...$environments);

#### メソッド追加

**影響の可能性： とても低い**

`bootstrapPath`、`configPath`、`databasePath`、`environmentPath`、`resourcePath`、`storagePath`、`resolveProvider`、`bootstrapWith`、`configurationIsCached`、`detectEnvironment`、`environmentFile`、`environmentFilePath`、`getCachedConfigPath`、`getCachedRoutesPath`、`getLocale`、`getNamespace`、`getProviders`、`hasBeenBootstrapped`、`loadDeferredProviders`、`loadEnvironmentFrom`、`routesAreCached`、`setLocale`、`shouldSkipMiddleware`、`terminate`メソッドが[`Illuminate/Contracts/Foundation/Application`契約に追加されました](https://github.com/laravel/framework/pull/26477)。

このインターフェイスを実装している可能性はとても低いと思われますが、実装している場合はこれらのメソッドを追加してください。

<a name="authentication"></a>
### 認証

#### パスワードリセット通知ルートパラメータ

**影響の可能性： 低い**

ユーザーがパスワードをリセットするためのリンクを必要とする場合、Laravelは`passwrod.reset`名前付きルートに対するURLを`route`ヘルパを利用し生成します。Laravel5.7使用時、明確な名前無しに`route`ヘルパへトークンを渡してしました。

    route('password.reset', $token);

Laravel5.8を使用する場合は、明確なパラメータとして`route`ヘルパへトークンを渡します。

    route('password.reset', ['token' => $token]);

そのため、独自の`password.reset`ルートを定義している場合、`{token}`パラメータを確実にURIへ含めてください。

#### 新しいデフォルトパスワード長

**影響の可能性： 低い**

パスワードの選択とリセット時に要求されるパスワード長が、[最低８文字へ変更されました](https://github.com/laravel/framework/pull/25957)。

<a name="cache"></a>
### キャッシュ

<a name="cache-ttl-in-seconds"></a>
#### 持続時間が秒指定に

**影響の可能性: とても高い**

より細かい時間でアイテムの保存時間を扱えるように、キャッシュの保存時間は分から秒単位になりました。`Illuminate\Cache\Repository`クラスとこれを拡張したクラスの`put`、`putMany`、`add`、`remember`、`setDefaultCacheTime`メソッド、並びに各キャッシュ保存の`put`メソッドはこの動作変更へ更新されました。詳細は、[関係するPR](https://github.com/laravel/framework/pull/27276)をご覧ください。

前述のメソッドに整数値を渡している場合、キャッシュにアイテムを残したい時間が秒数になったことに留意し、コードを変更してください。言い換えれば、アイテムの期限がいつ切れるかを表す`DateTime`インスタンスを渡したほうが良いでしょう。

    // Laravel5.7：30分アイテムは保存される
    Cache::put('foo', 'bar', 30);

    // Laravel5.8：30秒アイテムは保存される
    Cache::put('foo', 'bar', 30);

    // Laravel5.7と5.8：30秒アイテムは保存される
    Cache::put('foo', 'bar', now()->addSeconds(30));

> {tip} この変更により、Laravelキャッシュシステムは、[PSR-16キャッシュライブラリ基準](https://www.php-fig.org/psr/psr-16/)に完全に準拠しました。

<a name="psr-16-conformity"></a>
#### PSR-16準拠

**影響の可能性： 中程度**

前記により返却値が変更されたことに付け加え、`Illuminate\Cache\Repository`クラスの`put`、`putMany`、`add`メソッドのTTL引数もPSR-16基準をより準拠するように変更しました。デフォルトは`null`で、TTLを指定しない呼び出しの新しい振る舞いは、キャッシュアイテムを永久に保存することになりました。さらに、TTLが０か負数の場合はキャッシュから削除されます。詳細は、[関連するPR](https://github.com/laravel/framework/pull/27217)をご覧ください。

`KeyWritten`イベントもこの変更により、[更新されました](https://github.com/laravel/framework/pull/27265)。

<a name="cache-lock-safety-improvements"></a>
#### ロックの安全性向上

**影響の可能性： 高い**

バージョン5.7以前のLaravelでは、いくつかのキャッシュドライバで提供されていた「アトミックロック」機能は予測しない動作を引き起こす可能性がありました。

たとえば、**クライアントA**が１０秒間有効な`foo`ロックを獲得します。**クライアントA**がタスクを終了するまで実際には20秒かかるとします。**クライアントA**の処理中に、１０秒立つとキャッシシステムのロックが自動的に解除されます。**クライアントB**が`foo`ロックを獲得します。**クライアントA**がタスクを終了し、`foo`ロックを解除すると、思いがけず**クライアントB**が獲得しているロックを解除してしまいます。**クライアントC**は、そのロックを獲得できるようになってしまいます。

このシナリオを避けるに、通常の環境ではロックの所有者のみがリリースできることをフレームワークが確実に行えるようにするために、「スコープトークン」を埋め込んだロックが生成されるようになりました。

ロック操作に`Cache::lock()->get(Closure)`メソッドを使用している場合、変更は必要ありません。

    Cache::lock('foo', 10)->get(function () {
        // ロックは自動的に安全な開放を行う
    });

しかし、みなさんが自分で`Cache::lock()->release()`を呼び出している場合、ロックのインスタンスを保持するようにコードを変更する必要があります。そのため、タスクを実行し終えた後、**同じロックインスタンス**の`release`メソッドを呼び出してください。

    if ($lock = Cache::lock('foo', 10)->get()) {
        // タスクの実行…

        $lock->release();
    }

ときに、ロックをあるプロセスで獲得し、別のプロセスで開放したい場合があります。たとえば、Webリクエストでロックを獲得し、そのリクエストから起動したキュー済みジョブの最後で、ロックを開放したい場合です。そのようなシナリオでは、ジョブで渡されたトークンを使い、ロックを再インスタンス化できるように、ロックを限定する「所有者(owner)のトークン」をキューするジョブへ渡す必要があります。

    // コントローラ側
    $podcast = Podcast::find(1);

    if ($lock = Cache::lock('foo', 120)->get()) {
        ProcessPodcast::dispatch($podcast, $lock->owner());
    }

    // ProcessPodcastジョブ側
    Cache::restoreLock('foo', $this->owner)->release();

現在の所有者にかかわらず、ロックを開放したい場合は、`forceRelease`メソッドを使用します。

    Cache::lock('foo')->forceRelease();

#### `Repository`と`Store`契約

**影響の可能性： とても低い**

`Illuminate\Contracts\Cache\Repository`契約の`put`と`forever`メソッド、および`Illuminate\Contracts\Cache\Store`契約の`put`、`putMany`、`forever`メソッドの返却値を`PSR-16`へ完全に準拠させるため、`void`から`bool`へ[変更されました](https://github.com/laravel/framework/pull/26726)。

<a name="collections"></a>
### コレクション

#### `firstWhere`メソッド

**影響の可能性： とても低い**

`where`メソッドの使用法と合わせるために、`firstWhere`メソッドの使用法が[変更されました](https://github.com/laravel/framework/pull/26261)。このメソッドをオーバーライドしている場合は、メソッドの使用方法を親クラスと合わせてください。

    /**
     * 指定したキー／値ペアの最初のアイテムを取得
     *
     * @param  string  $key
     * @param  mixed  $operator
     * @param  mixed  $value
     * @return mixed
     */
    public function firstWhere($key, $operator = null, $value = null);

<a name="console"></a>
### コンソール

#### `Kernel`契約

**影響の可能性： とても低い**

`terminate`メソッドが[`Illuminate/Contracts/Console/Kernel`契約へ追加されました](https://github.com/laravel/framework/pull/26393)。このインターフェイスを実装している場合は、このメソッドを追加してください。

<a name="container"></a>
### コンテナ

<a name="container-generators"></a>
#### ジェネレータとタグ付けサービス

**影響の可能性： 中程度**

コンテナの`tagged`メソッドは、指定したタグのサービスに対し、PHPジェネレータの遅延インスタンス化を活用するようになりました。これにより、タグつけしたサービスを全部使用しない場合に、パフォーマンスの改善を達成しました。

この変更により、`tagged`メソッドは配列（`array`）の代わりに、`iterable`を返すようになりました。このメソッドの返却値をタイプヒントしている場合は、`iterable`へ確実に変更してください。

付け加えると、`$container->tagged('foo')[0]`のように、配列のオフセット値により直接タグ付けサービスへアクセスすることは、もうできなくなりました。

#### `resolve`メソッド

**影響の可能性： とても低い**

`resolve`メソッドは新しい論理値引数を[受け取るようになりました](https://github.com/laravel/framework/pull/27066)。これはオブジェクトのインスタンス化の間に、（コールバックを解決する）イベントが発生させられるか／実行されるかを指示します。このメソッドをオーバーライドしている場合、親クラスのものと一致するようにメソッドの使用法を変更してください。

#### `addContextualBinding`メソッド

**影響の可能性： とても低い**

`addContextualBinding`メソッドが、[`Illuminate\Contracts\Container\Container`契約に追加されました](https://github.com/laravel/framework/pull/26551)。このインターフェイスを実装している場合は、このメソッドを追加してください。

#### `tagged`メソッド

**影響の可能性： 低い**

`tagged`メソッドの使用法が[変更されました](https://github.com/laravel/framework/pull/26953)。そして、配列の代わりに`iterable`を返すようになりました。皆さんコードの引数として、このメソッドの返却値を配列としてタイプヒントしている場合は、`iterable`へ変更してください。

#### `flush`メソッド

**影響の可能性： とても低い**

`flush`メソッドが[`Illuminate\Contracts\Container\Container`契約へ追加されました](https://github.com/laravel/framework/pull/26477)。このインターフェイスを実装している場合は、このメソッドを追加してください。

<a name="database"></a>
### データベース

#### クオートしないMySQL JSON値

**影響の可能性： 低い**

MySQLとMariaDBを使用している場合、クエリビルダはクオートしないJSON値を返すようになりました。この振る舞いは、サポートしている他のデータベースと一貫性を持たせるためです。

    $value = DB::table('users')->value('options->language');

    dump($value);

    // Laravel5.7
    '"en"'

    // Laravel5.8
    'en'

その結果として、`->>`操作子は必要なくなり、サポートされなくなりました。

<a name="sqlite"></a>
#### SQLite

**影響の可能性： 中程度**

Laravel5.8が[サポートする一番古いSQLiteバージョン](https://github.com/laravel/framework/pull/25995)はSQLite3.7.11です。より古いSQLiteバージョンを使用している場合は、アップデートしてください。SQLite3.8.8以上を推奨します。

<a name="eloquent"></a>
### Eloquent

<a name="model-names-ending-with-irregular-plurals"></a>
#### 不規則変化する複数形のモデル名

**影響の可能性： 中程度**

Laravel5.8では、複数後のモデル名で不規則変化する単語で終わる場合も、[正しい複数形になるように](https://github.com/laravel/framework/pull/26421)なりました。

    // Laravel5.7
    App\Feedback.php -> feedback (正しい複数形)
    App\UserFeedback.php -> user_feedbacks (間違った複数形)

    // Laravel5.8
    App\Feedback.php -> feedback (正しい複数形)
    App\UserFeedback.php -> user_feedback (正しい複数形)

間違った複数形のモデルがある場合は、モデルの`$table`プロパティを定義することにより、古いテーブル名を続けて使用できます。

    /**
     * モデルと関連するテーブル
     *
     * @var string
     */
    protected $table = 'user_feedbacks';

<a name="custom-pivot-models-with-incrementing-ids"></a>
#### IDが増加するカスタムピボットモデル

カスタムピボットモデルを使用する多対多リレーションを定義しており、そのピボットモデルが自動増分の主キーを持つ場合、カスタムピボットモデルクラスで確実に`incrementing`プロパティを定義し、`true`をセットしてください。

    /**
     * IDが自動増分する
     *
     * @var bool
     */
    public $incrementing = true;

#### `loadCount`メソッド

**影響の可能性： 低い**

`loadCount`メソッドが、ベースの`Illuminate\Database\Eloquent\Model`クラスへ追加されました。皆さんのアプリケーションでも`loadCount`メソッドを定義している場合、Eloquentの定義と名前が衝突してしまいます。

#### `originalIsEquivalent`メソッド

**影響の可能性： とても低い**

`Illuminate\Database\Eloquent\Concerns\HasAttributes`トレイトの`originalIsEquivalent`メソッドが、`protected`から`public`へ[変更されました](https://github.com/laravel/framework/pull/26391)。

#### ソフト削除の`deleted_at`プロパティの自動キャスト

**影響の可能性： 低い**

`Illuminate\Database\Eloquent\SoftDeletes`トレイトを使用しているEloquentモデルを使用している場合、`deleted_at`プロパティは`Carbon`インスタンスへ、[自動的にキャストされるようになりました](https://github.com/laravel/framework/pull/26985)。この振る舞いはプロパティのカスタムアクセサを書くか、`casts`属性へ追加することによりオーバーライドできます。

    protected $casts = ['deleted_at' => 'string'];

#### BelongsToの`getForeignKey`メソッド

**影響の可能性： 低い**

Laravelにより提供されている他のリレーションのメソッド名と統一するために、`BelongsTo`の`getForeignKey`と`getQualifiedForeignKey`メソッドは、`getForeignKeyName`と`getQualifiedForeignKeyName`へ名前が変わりました。

<a name="events"></a>
### イベント

#### `fire`メソッド

**影響の可能性： 低い**

Laravel5.4で非推奨になっていた、`Illuminate/Events/Dispatcher`クラスの`fire`メソッドが、[削除されました](https://github.com/laravel/framework/pull/26392)。
代わりに、`dispatch`メソッドを使用してください。

<a name="exception-handling"></a>
### 例外処理

#### `ExceptionHandler`契約

**影響の可能性： 低い**

`shouldReport`メソッドが[`Illuminate\Contracts\Debug\ExceptionHandler`契約へ追加されました](https://github.com/laravel/framework/pull/26193)。このインターフェイスを実装している場合は、メソッドを追加してください。

#### `renderHttpException`メソッド

**影響の可能性： 低い**

`Illuminate\Foundation\Exceptions\Handler`クラスの`renderHttpException`メソッドの使用法が[変更されました](https://github.com/laravel/framework/pull/25975)。例外ハンドラのこのメソッドをオーバーライドしている場合は、親クラスとメソッドの使い方を合わせるように変更してください。

    /**
     * 渡されたHttpExceptionをレンダーする
     *
     * @param  \Symfony\Component\HttpKernel\Exception\HttpExceptionInterface  $e
     * @return \Symfony\Component\HttpFoundation\Response
     */
    protected function renderHttpException(HttpExceptionInterface $e);

<a name="facades"></a>
### ファサード

#### ファサードサービスの依存解決

**影響の可能性： 低い**

`getFacadeAccessor`メソッドは、[サービスのコンテナ識別子を表す文字列のみをリターンする](https://github.com/laravel/framework/pull/25525)ようになりました。以前、このメソッドはオブジェクトインスタンスを返していました。

<a name="mail"></a>
### メール

<a name="markdown-file-directory-change"></a>
<a name="markdown-file-directory-change"></a>
### Markdownファイルのディレクトリ変更

**影響の可能性： 高い**

LaravelのMarkdownメールコンポーネントを`vendor:publish`コマンドでリソース公開している場合は、`/resources/views/vendor/mail/markdown`ディレクトリを`text`へリネームしてください。

さらに、`markdownComponentPaths`メソッドは`textComponentPaths`へ[リネームされました](https://github.com/laravel/framework/pull/26938)。このメソッドをオーバーライドしている場合は、親のクラスへ合わせるようにメソッド名を変更してください。

#### `PendingMail`クラスのメソッド使用法変更

**影響の可能性： とても低い**

`Illuminate\Mail\Mailable`の代わりに`Illuminate\Contracts\Mail\Mailable`を受け付けるため、`Illuminate\Mail\PendingMail`クラスの`send`、`sendNow`、`queue`、`later`、`fill`メソッドは[変更されました](https://github.com/laravel/framework/pull/26790)。これらのメソッドをオーバーライドしている場合は、親クラスと使用法を合わせるように更新してください。

<a name="queue"></a>
### キュー

<a name="pheanstalk-4"></a>
#### Pheanstalk4.0

**影響の可能性： 中程度**

Laravel5.8は、Pheanstalkキューライブラリの`~4.0`をサポートしました。皆さんのアプリケーションでPheanstalkライブラリを使用している場合は、Composerを使用し`~4.0`リリースへライブラリをアップグレードしてください。

#### `Job`契約

**影響の可能性： とても低い**

`isReleased`、`hasFailed`、`markAsFailed`メソッドが、[`Illuminate\Contracts\Queue\Job`契約へ追加されました](https://github.com/laravel/framework/pull/26908)。このインターフェイスを実装してる場合は、これらのメソッドを追加してください。

#### `Job::failed`と`FailingJob`クラス

**影響の可能性： とても低い**

Laravel5.7ではキューされたジョブが失敗したときに、キューワーカは`FailingJob::handle`メソッドを実行していました。Laravel5.8では、`FailingJob`クラスに含まれていたロジックは、ジョブクラス自身の`fail`メソッドへそのまま移動されました。このため、`Illuminate\Contracts\Queue\Job`契約へ`fail`メソッドが追加されました。

ベースの`Illuminate\Queue\Jobs\Job`クラスが`fail`の実装を含んでも、通常のアプリケーションコードではコードの変更は必要ありません。しかし、Laravelにより提供されているベースのジョブクラスを拡張**しないで**、ジョブクラスを活用するカスタムキュードライバを構築している場合は、カスタムジョブクラスへ`fail`メソッドを自分で実装する必要があります。実装の例として、Laravelのベースジョブクラスを参照してください。

この変更により、ジョブの削除過程をカスタムキュードライバがよりコントロールできるようになりました。

#### Redis Blocking Pop

**影響の可能性： とても低い**

Redisキュードライバの"blocking pop"機能の使用が安全になりました。以前は、ジョブ再取得と同時にRedisサーバかワーカがクラッシュすると、キューされたジョブを失う機会が多少ありました。Blocking popが安全になったため、各Laravelキューに対し新しいRedisは、`:notify`サフィックス付けてリストします。

<a name="requests"></a>
### リクエスト

#### `TransformsRequest`ミドルウェア

**影響の可能性： 低い**

`Illuminate\Foundation\Http\Middleware\TransformsRequest`ミドルウェアの`transform`メソッドは、入力が配列のときに「完全な」リクエスト入力キーを受け入れるようになりました。

    'employee' => [
        'name' => 'Taylor Otwell',
    ],

    /**
     * 渡された値を変形する
     *
     * @param  string  $key
     * @param  mixed  $value
     * @return mixed
     */
    protected function transform($key, $value)
    {
        dump($key); // 'employee.name' (Laravel5.8)
        dump($key); // 'name' (Laravel5.7)
    }

<a name="routing"></a>
### ルート

#### `UrlGenerator`契約

**影響の可能性： とても低い**

`previous`メソッドが、[`Illuminate\Contracts\Routing\UrlGenerator`契約へ追加されました](https://github.com/laravel/framework/pull/25616)。このインターフェイスを実装している場合は、メソッドを追加してください。

#### `Illuminate/Routing/UrlGenerator`の`cachedSchema`プロパティ

**影響の可能性： とても低い**

Laravel`5.7`で非推奨になっていた、`Illuminate/Routing/UrlGenerator`の`$cachedSchema`プロパティ名が、`$cachedScheme`へ[変更されました](https://github.com/laravel/framework/pull/26728)。

<a name="sessions"></a>
### セッション

#### `StartSession`ミドルウェア

**影響の可能性： とても低い**

セッションの維持ロジックを[`terminate()`メソッドから`handle()`メソッドへ移動しました](https://github.com/laravel/framework/pull/26410)。これらのメソッドをオーバーライドしている場合は、この変更を反映するように更新してください。

<a name="support"></a>
### サポート

<a name="string-and-array-helpers"></a>
#### ヘルパから文字列と配列クラスへ

**影響の可能性： 中程度**

すべての`array_*`と`str_*`グローバルヘルパが、[非推奨となりました](https://github.com/laravel/framework/pull/26898)。`Illuminate\Support\Arr`と`Illuminate\Support\Str`のメソッドを直接使用してください。

全ての配列と文字列のグローバル関数の後方互換性層を提供する、新しい[laravel/helpers](https://github.com/laravel/helpers)パッケージへヘルパは移動されたため、この変更は`medium`と記載しました。

<a name="deferred-service-providers"></a>
#### 遅延サービスプロバイダ

**影響の可能性： 中程度**

プロバイダを遅延させるかどうかを示すための、サービスプロバイダの`defer`論理プロパティは、[非推奨となりました](https://github.com/laravel/framework/pull/27067)。そのサービスプロバイダを遅延させるように指示するには、`Illuminate\Contracts\Support\DeferrableProvider`契約を実装してください。

<a name="testing"></a>
### テスト

#### PHPUnit 8

**影響の可能性： 状況による**

Laravel5.8ではPHPUnit7をデフォルトで使用します。しかしながら、PHP7.2以上が必要なPHPUnit8へアップグレードする選択肢もあります。詳細は、[PHPUnit8のリリースアナウンスメント](https://phpunit.de/announcements/phpunit-8.html)の変更一覧へ一通り目を通してください。

`setUp`と`tearDown`メソッドは、voidを返却タイプとして要求するようになりました。

    protected function setUp(): void
    protected function tearDown(): void

<a name="validation"></a>
### バリデーション

#### `Validator`契約

**影響の可能性： とても低い**

`validated`メソッドが、[`Illuminate\Contracts\Validation\Validator`契約へ追加されました](https://github.com/laravel/framework/pull/26419)。

    /**
     * バリデートされた属性と値の取得
     *
     * @return array
     */
    public function validated();

このインターフェイスを実装してる場合は、メソッドを追加してください。

#### `ValidatesAttributes`トレイト

**影響の可能性： とても低い**

`Illuminate\Validation\Concerns\ValidatesAttributes`トレイトの`parseTable`、`getQueryColumn`、`requireParameterCount`メソッドを`protected`から`public`に変更しました。

#### `DatabasePresenceVerifier`クラス

**影響の可能性： とても低い**

`Illuminate\Validation\DatabasePresenceVerifier`クラスの`table`メソッドを`protected`から`public`に変更しました。

#### `Validator`クラス

**影響の可能性： とても低い**

`Illuminate\Validation\Validator`クラスの`getPresenceVerifierFor`メソッドを`protected`から`public`に[変更しました](https://github.com/laravel/framework/pull/26717)。

#### メールバリデーション

**影響の可能性： とても低い**

メールのバリデーションルールをメールが[RFC5630](https://tools.ietf.org/html/rfc6530)準拠するかをチェックするようになりました。SwiftMailerが使用しているバリデーションロジックと統一しました。Laravel`5.7`では、`email`ルールはメールが[RFC822](https://tools.ietf.org/html/rfc822)を準拠しているかのみを検証していました。

これにより、Laravel5.8を使用すると以前は誤って有効でないと判断したメールを有効と判断するようになりました。（例：`hej@bär.se`）一般的にはバグフィックスとして考えるべきでしょう。しかしながら、これは注意の必要な互換性のない変更としてリストされています。[この変更により、問題が起きたらお知らせください](https://github.com/laravel/framework/pull/26503)。

<a name="view"></a>
### ビュー

#### `getData`メソッド

**影響の可能性： とても低い**

`getData`メソッドが、[`Illuminate\Contracts\View\View`契約へ追加されました](https://github.com/laravel/framework/pull/26754)。このインターフェイスを実装している場合は、メソッドを追加してください。

<a name="notifications"></a>
### 通知

<a name="nexmo-slack-notification-channels"></a>
#### Nexmo／Slack通知チャンネル

**影響の可能性： 高い**

NexmoとSlackの通知チャンネルをファーストパーティパッケージとして独立させました。これらのチャンネルをアプリケーションで使用する場合は、以下のパッケージをインストールしてください。

    composer require laravel/nexmo-notification-channel
    composer require laravel/slack-notification-channel

<a name="miscellaneous"></a>
### その他

`laravel/laravel`の[GitHubリポジトリ](https://github.com/laravel/laravel)で、変更を確認することをおすすめします。これらの変更は必須ではありませんが、皆さんのアプリケーションではファイルの同期を保つほうが良いでしょう。変更のいくつかは、このアップグレードガイドで取り扱っていますが、設定ファイルやコメントなどの変更は取り扱っていません。変更は簡単に[GitHubの比較ツール](https://github.com/laravel/laravel/compare/5.7...master)で閲覧でき、みなさんにとって重要な変更を選択できます。
