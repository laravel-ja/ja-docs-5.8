# API認証

- [イントロダクション](#introduction)
- [設定](#configuration)
    - [データベースの準備](#database-preparation)
- [トークン生成](#generating-tokens)
    - [トークンのハッシュ](#hashing-tokens)
- [ルートの保護](#protecting-routes)
- [リクエストへのトークン付加](#passing-tokens-in-requests)

<a name="introduction"></a>
## イントロダクション

デフォルトとして、アプリケーションの各ユーザーに結びつけたランダムトークンを利用し、API認証を行うシンプルな手法をLaravelは採用しています。`config/auth.php`設定ファイルの中で、`api`ガードは`token`ドライバを定義し、使用するようになっています。このドライバーは、受信したリクエストのAPIトークンを調べ、データベース上のユーザーに結びつけたトークンと一致するか検証することに責任を持っています。

> **Note:** Laravelでは、シンプルなトークンベースの認証ガードを提供していますが、API認証を提供する堅牢なプロダクションアプリケーションでは、[Laravel Passport](/docs/{{version}}/passport)の使用を考慮することを強くおすすめします。

<a name="configuration"></a>
## 設定

<a name="database-preparation"></a>
### データベースの準備

`token`ドライバを使用する前に、`users`テーブルへ`api_token`カラムを追加する[マイグレーションを作成](/docs/{{version}}/migrations)する必要があります。

    Schema::table('users', function ($table) {
        $table->string('api_token', 80)->after('password')
                            ->unique()
                            ->nullable()
                            ->default(null);
    });

マイグレーションが出来上がったら、`migrate` Artisanコマンドを実行します。

> {tip} 別のカラム名を使う場合は、`config/auth.php`設定ファイル中の`storage_key`設定オプションを必ず更新してください。

<a name="generating-tokens"></a>
## トークン生成

`users`テーブルへ`api_token`カラムを追加したら、アプリケーションに登録している各ユーザーへ、ランダムなAPIトークンを割り付ける準備が整いました。ユーザーの登録で`User`モデル生成時に、トークンを割り付けるべきでしょう。`make:auth` Artisanコマンドによる、[認証スカフォールド](/docs/{{version}}/authentication#authentication-quickstart)を使用する場合は、`RegisterController`の`create`メソッドで行われています。

    use Illuminate\Support\Str;
    use Illuminate\Support\Facades\Hash;

    /**
     * 登録バリデーション後に、新ユーザーインスタンスの生成
     *
     * @param  array  $data
     * @return \App\User
     */
    protected function create(array $data)
    {
        return User::create([
            'name' => $data['name'],
            'email' => $data['email'],
            'password' => Hash::make($data['password']),
            'api_token' => Str::random(60),
        ]);
    }

<a name="hashing-tokens"></a>
### トークンのハッシュ

前記の例では、APIトークンはデータベースへ平文のまま保存されます。SHA-256を使用し、APIトークンをハッシュしたい場合は、`api`ガードの`hash`オプションを`true`に設定してください。`api`ガードは`config/auth.php`設定ファイルで定義されています。

    'api' => [
        'driver' => 'token',
        'provider' => 'users',
        'hash' => true,
    ],

#### ハッシュ済みトークンの生成

ハッシュ済みAPIトークンを使用する場合、ユーザー登録時にAPIトークンを生成してはいけません。代わりに、アプリケーション中にAPIトークン管理ページを実装する必要があります。このページでユーザーにAPIトークンの初期化と再生成を提供します。あるユーザーがトークンの初期化や再生成をリクエストしたら、トークンのハッシュ済みコピーをデータベースへ保存し、ビューやフロントエンドクライアントで一度だけ表示するために、平文のコピーを返す必要があります。

たとえば、指定したユーザーのトークンを初期化／再生成し、JSONレスポンスとして平文トークンを返すコントローラメソッドは、次のようになるでしょう。

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Support\Str;
    use Illuminate\Http\Request;

    class ApiTokenController extends Controller
    {
        /**
         * 認証済みのユーザーのAPIトークンを更新する
         *
         * @param  \Illuminate\Http\Request  $request
         * @return array
         */
        public function update(Request $request)
        {
            $token = Str::random(60);

            $request->user()->forceFill([
                'api_token' => hash('sha256', $token),
            ])->save();

            return ['token' => $token];
        }
    }

> {tip} 上記例のAPIトークンは、十分なエントロピーを持ちますので、「レインボーテーブル」を作成してハッシュ済みトークンのオリジナル値を探し出すのは、非現実的になります。そのため、`bcrypt`などの遅いハッシュ方法は不必要ありません。

<a name="protecting-routes"></a>
## ルートの保護

Laravelは、受信したリクエストのAPIトークンを自動的にバリデートする、[認証ガード](/docs/{{version}}/authentication#adding-custom-guards)を提供しています。アクセストークンの有効性が必要なルートへ、`auth:api`ミドルウェアを指定するだけです。

    use Illuminate\Http\Request;

    Route::middleware('auth:api')->get('/user', function(Request $request) {
        return $request->user();
    });

<a name="passing-tokens-in-requests"></a>
## リクエストへのトークン付加

アプリケーションへAPIトークンを渡す方法はいくつかあります。Guzzle HTTPライブラリを使用したときの利用方法をデモンストレーションするために、いくつかのアプローチを検討してみます。アプリケーションの必要に合わせて選択してください。

#### クエリ文字列

アプリケーションのAPI利用側が、`api_token`クエリ文字列の値としてトークンを指定できます。

    $response = $client->request('GET', '/api/user?api_token='.$token);

#### ペイロードのリクエスト

アプリケーションのAPI利用側が、リクエストフォームの`api_token`パラメータへ、APIトークンを含められます。

    $response = $client->request('POST', '/api/user', [
        'headers' => [
            'Accept' => 'application/json',
        ],
        'form_params' => [
            'api_token' => $token,
        ],
    ]);

#### Bearerトークン

アプリケーションのAPI利用側が、リクエストの`Authorization`ヘッダへ`Bearer`トークンとして、APIトークンを提供できます。

    $response = $client->request('POST', '/api/user', [
        'headers' => [
            'Authorization' => 'Bearer '.$token,
            'Accept' => 'application/json',
        ],
    ]);
