# エラー処理

- [イントロダクション](#introduction)
- [設定](#configuration)
- [例外ハンドラ](#the-exception-handler)
    - [Reportメソッド](#report-method)
    - [Renderメソッド](#render-method)
    - [Reportable／Renderable例外](#renderable-exceptions)
- [HTTP例外](#http-exceptions)
    - [カスタムHTTPエラーページ](#custom-http-error-pages)

<a name="introduction"></a>
## イントロダクション

新しいLaravelプロジェクトを開始する時点で、エラーと例外の処理は既に設定済みです。`App\Exceptions\Handler`クラスはアプリケーションで発生する全例外をログし、ユーザーへ表示するためのクラスです。このドキュメントでは、このクラスの詳細を確認していきます。

<a name="configuration"></a>
## 設定

アプリケーションエラー発生時にユーザーに対し表示する詳細の表示量は、`config/app.php`設定ファイルの`debug`設定オプションで決定します。デフォルト状態でこの設定オプションは、`.env`ファイルで指定される`APP_DEBUG`環境変数の値を反映します。

local環境では`APP_DEBUG`環境変数を`true`に設定すべきでしょう。実働環境ではこの値をいつも`false`にすべきです。実働環境でこの値を`true`にしてしまうと、アプリケーションのエンドユーザーへ、セキュリティリスクになりえる設定情報を表示するリスクを犯すことになります。

<a name="the-exception-handler"></a>
## 例外ハンドラ

<a name="report-method"></a>
### reportメソッド

例外はすべて、`App\Exceptions\Handler`クラスで処理されます。このクラスは`report`と`render`二つのメソッドで構成されています。両メソッドの詳細を見ていきましょう。`report`メソッドは例外をログするか、[BugSnag](https://bugsnag.com)や[Sentry](https://github.com/getsentry/sentry-laravel)のような外部サービスに送信するために使います。デフォルト状態の`report`メソッドは、渡された例外をベースクラスに渡し、そこで例外はログされます。しかし好きなように例外をログすることが可能です。

たとえば異なった例外を別々の方法レポートする必要がある場合、PHPの`instanceof`比較演算子を使ってください。

    /**
     * 例外をレポート、もしくはログする
     *
     * ここは例外をSentryやBugsnagなどへ送るために適した場所
     *
     * @param  \Exception  $exception
     * @return void
     */
    public function report(Exception $exception)
    {
        if ($exception instanceof CustomException) {
            //
        }

        parent::report($exception);
    }

> {tip} `report`メソッド中で数多くの`instanceof`チェックを行う代わりに、[reportable exceptions](/docs/{{version}}/errors#renderable-exceptions)の使用を考慮してください。

#### グローバルログコンテキスト

Laravelは可能である場合、文脈上のデータとして全ての例外ログへ、現在のユーザーのIDを自動的に追加します。アプリケーションの`App\Exceptions\Handler`クラスにある、`context`メソッドをオーバーライドすることにより、独自のグローバルコンテキストデータを定義できます。この情報は、アプリケーションにより書き出される全ての例外ログメッセージに含まれます。

    /**
     * ログのデフォルトコンテキスト変数の取得
     *
     * @return array
     */
    protected function context()
    {
        return array_merge(parent::context(), [
            'foo' => 'bar',
        ]);
    }

#### `report`ヘルパ

場合により、例外をレポートする必要があるが、現在のリクエストの処理を継続したい場合があります。`report`ヘルパ関数により、エラーページをレンダすること無く、例外ハンドラの`report`メソッドを使用し、例外を簡単にレポートできます。

    public function isValid($value)
    {
        try {
            // 値の確認…
        } catch (Exception $e) {
            report($e);

            return false;
        }
    }

#### タイプによる例外の無視

例外ハンドラの`$dontReport`プロパティは、ログしない例外のタイプの配列で構成します。たとえば、404エラー例外と同様に、他のタイプの例外もログしたくない場合です。必要に応じてこの配列へ、他の例外を付け加えてください。

    /**
     * レポートしない例外のリスト
     *
     * @var array
     */
    protected $dontReport = [
        \Illuminate\Auth\AuthenticationException::class,
        \Illuminate\Auth\Access\AuthorizationException::class,
        \Symfony\Component\HttpKernel\Exception\HttpException::class,
        \Illuminate\Database\Eloquent\ModelNotFoundException::class,
        \Illuminate\Validation\ValidationException::class,
    ];

<a name="render-method"></a>
### renderメソッド

`render`メソッドは与えられた例外をブラウザーに送り返すHTTPレスポンスへ変換することに責任を持っています。デフォルトで例外はベースクラスに渡され、そこでレスポンスが生成されます。しかし例外のタイプをチェックし、お好きなようにレスポンスを返してかまいません。

    /**
     * HTTPレスポンスへ例外をレンダー
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Exception  $exception
     * @return \Illuminate\Http\Response
     */
    public function render($request, Exception $exception)
    {
        if ($exception instanceof CustomException) {
            return response()->view('errors.custom', [], 500);
        }

        return parent::render($request, $exception);
    }

<a name="renderable-exceptions"></a>
### Reportable／Renderable例外

例外ハンドラの中の`report`と`render`メソッドの中で、例外のタイプをチェックする代わりに、自身のカスタム例外で`report`と`render`メソッドを定義できます。これらのメソッドが存在すると、フレームワークにより自動的に呼び出されます。

    <?php

    namespace App\Exceptions;

    use Exception;

    class RenderException extends Exception
    {
        /**
         * 例外のレポート
         *
         * @return void
         */
        public function report()
        {
            //
        }

        /**
         * 例外をＨＴＴＰレスポンスへレンダ
         *
         * @param  \Illuminate\Http\Request
         * @return \Illuminate\Http\Response
         */
        public function render($request)
        {
            return response(...);
        }
    }

> {tip} `report`メソッドに必要な依存をタイプヒントで指定することで、Laravelの[サービスコンテナ](/docs/{{version}}/container)によりメソッドへ、自動的に依存注入されます。

<a name="http-exceptions"></a>
## HTTP例外

例外の中にはサーバでのHTTPエラーコードを表しているものがあります。たとえば「ページが見つかりません」エラー(404)や「未認証エラー」(401)、開発者が生成した500エラーなどです。アプリケーションのどこからでもこの種のレスポンスを生成するには、abortヘルパを使用します。

    abort(404);

`abort`ヘルパは即座に例外を発生させ、その例外は例外ハンドラによりレンダーされることになります。オプションとしてレスポンスのテキストを指定することもできます。

    abort(403, 'Unauthorized action.');

<a name="custom-http-error-pages"></a>
### カスタムHTTPエラーページ

様々なHTTPステータスコードごとに、Laravelはカスタムエラーページを簡単に返せます。たとえば404 HTTPステータスコードに対してカスタムエラーページを返したければ、`resources/views/errors/404.blade.php`を作成してください。このファイルはアプリケーションで起こされる全404エラーに対し動作します。ビューはこのディレクトリに置かれ、対応するHTTPコードと一致した名前にしなくてはなりません。`abort`ヘルパが生成する`HttpException`インスタンスは、`$exception`変数として渡されます。

    <h2>{{ $exception->getMessage() }}</h2>

Laravelのエラーページテンプレートは、`vendor:publish` Artisanコマンドで公開できます。テンプレートを公開したら、好みのようにカスタマイズできます。

    php artisan vendor:publish --tag=laravel-errors
