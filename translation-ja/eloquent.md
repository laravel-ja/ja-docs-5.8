# Eloquent：利用の開始

- [イントロダクション](#introduction)
- [モデル定義](#defining-models)
    - [Eloquentモデル規約](#eloquent-model-conventions)
    - [デフォルト属性値](#default-attribute-values)
- [モデルの取得](#retrieving-models)
    - [コレクション](#collections)
    - [結果の分割](#chunking-results)
- [１モデル／集計の取得](#retrieving-single-models)
    - [集計の取得](#retrieving-aggregates)
- [モデルの追加と更新](#inserting-and-updating-models)
    - [Inserts](#inserts)
    - [Updates](#updates)
    - [複数代入](#mass-assignment)
    - [他の生成メソッド](#other-creation-methods)
- [モデル削除](#deleting-models)
    - [ソフトデリート](#soft-deleting)
    - [ソフトデリート済みモデルのクエリ](#querying-soft-deleted-models)
- [クエリスコープ](#query-scopes)
    - [グローバルスコープ](#global-scopes)
    - [ローカルスコープ](#local-scopes)
- [モデルの比較](#comparing-models)
- [イベント](#events)
    - [オブザーバ](#observers)

<a name="introduction"></a>
## イントロダクション

Eloquent ORMはLaravelに含まれている、美しくシンプルなアクティブレコードによるデーター操作の実装です。それぞれのデータベーステーブルは関連する「モデル」と結びついています。モデルによりテーブル中のデータをクエリできますし、さらに新しいレコードを追加することもできます。

使用開始前に`config/database.php`を確実に設定してください。データベースの詳細は[ドキュメント](/docs/{{version}}/database#configuration)で確認してください。

<a name="defining-models"></a>
## モデル定義

利用を開始するには、まずEloquentモデルを作成しましょう。通常モデルは`app`ディレクトリ下に置きますが、`composer.json`ファイルでオートロードするように指定した場所であれば、どこでも自由に設置できます。全てのEloquentモデルは、`Illuminate\Database\Eloquent\Model`を拡張する必要があります。

モデルを作成する一番簡単な方法は`make:model` [Artisanコマンド](/docs/{{version}}/artisan)を使用することです。

    php artisan make:model Flight

モデル作成時に[データベースマイグレーション](/docs/{{version}}/migrations)も生成したければ、`--migration`か`-m`オプションを使ってください。

    php artisan make:model Flight --migration

    php artisan make:model Flight -m

<a name="eloquent-model-conventions"></a>
### Eloquentモデル規約

では`flights`データベーステーブルに情報を保存し、取得するために使用する`Flight`モデルクラスを例として見てください。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Flight extends Model
    {
        //
    }

#### テーブル名

`Flight`モデルにどのテーブルを使用するか、Eloquentに指定していない点に注目してください。他の名前を明示的に指定しない限り、クラス名を複数形の「スネークケース」にしたものが、テーブル名として使用されます。今回の例で、Eloquentは`Flight`モデルを`flights`テーブルに保存します。モデルの`table`プロパティを定義し、カスタムテーブル名を指定することもできます。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Flight extends Model
    {
        /**
         * モデルと関連しているテーブル
         *
         * @var string
         */
        protected $table = 'my_flights';
    }

#### 主キー

Eloquentは更にテーブルの主キーが`id`というカラム名であると想定しています。この規約をオーバーライドする場合は、protectedの`primaryKey`プロパティを定義してください。

さらに、Eloquentは主キーを自動増分される整数値であるとも想定しています。つまり、デフォルト状態で主キーは自動的に`int`へキャストされます。自動増分ではない、もしくは整数値ではない主キーを使う場合、モデルにpublicの`$incrementing`プロパティを用意し、`false`をセットしてください。主キーが整数でない場合は、モデルのprotectedの`$keyType`プロパティへ`string`値を設定してください。

#### タイムスタンプ

デフォルトでEloquentはデータベース上に存在する`created_at`(作成時間)と`updated_at`(更新時間)カラムを自動的に更新します。これらのカラムの自動更新をEloquentにしてほしくない場合は、モデルの`$timestamps`プロパティを`false`に設定してください。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Flight extends Model
    {
        /**
         * モデルのタイムスタンプを更新するかの指示
         *
         * @var bool
         */
        public $timestamps = false;
    }

タイムスタンプのフォーマットをカスタマイズする必要があるなら、モデルの`$dateFormat`プロパティを設定してください。このプロパティはデータベースに保存される日付属性のフォーマットを決めるために使用されると同時に、配列やJSONへシリアライズする時にも使われます。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Flight extends Model
    {
        /**
         * モデルの日付カラムの保存フォーマット
         *
         * @var string
         */
        protected $dateFormat = 'U';
    }

タイムスタンプを保存するカラム名をカスタマイズする必要がある場合、モデルに`CREATED_AT`と`UPDATED_AT`定数を設定してください。

    <?php

    class Flight extends Model
    {
        const CREATED_AT = 'creation_date';
        const UPDATED_AT = 'last_update';
    }

#### データベース接続

Eloquentモデルはデフォルトとして、アプリケーションに設定されているデフォルトのデータベース接続を使用します。モデルで異なった接続を指定したい場合は、`$connection`プロパティを使用します。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Flight extends Model
    {
        /**
         * モデルで使用するコネクション名
         *
         * @var string
         */
        protected $connection = 'connection-name';
    }

<a name="default-attribute-values"></a>
### デフォルト属性値

あるモデルの属性にデフォルト値を指定したい場合は、そのモデルに`$attributes`プロパティを定義してください。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Flight extends Model
    {
        /**
         * 属性に対するモデルのデフォルト値
         *
         * @var array
         */
        protected $attributes = [
            'delayed' => false,
        ];
    }

<a name="retrieving-models"></a>
## モデルの取得

モデルと[対応するデータベーステーブル](/docs/{{version}}/migrations#writing-migrations)を作成したら、データベースからデータを取得できるようになりました。各Eloquentモデルは、対応するデータベーステーブルへすらすらとクエリできるようにしてくれる[クエリビルダ](/docs/{{version}}/queries)だと考えてください。例を見てください。

    <?php

    $flights = App\Flight::all();

    foreach ($flights as $flight) {
        echo $flight->name;
    }

#### 制約の追加

Eloquentの`all`メソッドはモデルテーブルの全レコードを結果として返します。Eloquentモデルは[クエリビルダ](/docs/{{version}}/queries)としても動作しますのでクエリに制約を付け加えることもでき、結果を取得するには`get`メソッドを使用します。

    $flights = App\Flight::where('active', 1)
                   ->orderBy('name', 'desc')
                   ->take(10)
                   ->get();

> {tip} Eloquentモデルはクエリビルダですから、[クエリビルダ](/docs/{{version}}/queries)で使用できる全メソッドを確認しておくべきでしょう。Eloquentクエリでどんなメソッドも使用できます。

#### モデルのリフレッシュ

`fresh`と`refresh`メソッドを使用し、モデルをリフレッシュできます。`fresh`メソッドはデータベースからモデルを再取得します。既存のモデルインスタンスは影響を受けません。

    $flight = App\Flight::where('number', 'FR 900')->first();

    $freshFlight = $flight->fresh();

`refresh`メソッドは、データベースから取得したばかりのデータを使用し、既存のモデルを再構築します。

    $flight = App\Flight::where('number', 'FR 900')->first();

    $flight->number = 'FR 456';

    $flight->refresh();

    $flight->number; // "FR 900"

<a name="collections"></a>
### コレクション

複数の結果を取得する`all`や`get`のようなEloquentメソッドは、`Illuminate\Database\Eloquent\Collection`インスタンスを返します。`Collection`クラスはEloquent結果を操作する[多くの便利なクラス](/docs/{{version}}/eloquent-collections#available-methods)を提供しています。

    $flights = $flights->reject(function ($flight) {
        return $flight->cancelled;
    });

このコレクションは配列のようにループさせることもできます。

    foreach ($flights as $flight) {
        echo $flight->name;
    }

<a name="chunking-results"></a>
### 結果の分割

数千のEloquentレコードを処理する必要がある場合は`chunk`コマンドを利用してください。`chunk`メソッドはEloquentモデルの「塊(chunk)」を取得し、引数の「クロージャ」に渡します。`chunk`メソッドを使えば大きな結果を操作するときのメモリを節約できます。

    Flight::chunk(200, function ($flights) {
        foreach ($flights as $flight) {
            //
        }
    });

最初の引数には「チャンク（塊）」ごとにいくつのレコードを処理するかを渡します。２番めの引数にはクロージャを渡し、そのデータベースからの結果をチャンクごとに処理するコードを記述します。クロージャへ渡されるチャンクを取得するたびに、データベースクエリは実行されます。

#### カーソルの使用

`cursor`メソッドにより、ひとつだけクエリを実行するカーソルを使用し、データベース全体を繰り返し処理できます。大量のデータを処理する場合、`cursor`メソッドを使用すると、大幅にメモリ使用量を減らせるでしょう。

    foreach (Flight::where('foo', 'bar')->cursor() as $flight) {
        //
    }

<a name="retrieving-single-models"></a>
## １モデル／集計の取得

指定したテーブルの全レコードを取得することに加え、`find`と`first`を使い１レコードだけを取得できます。モデルのコレクションの代わりに、これらのメソッドは１モデルインスタンスを返します。

    // 主キーで指定したモデル取得
    $flight = App\Flight::find(1);

    // クエリ条件にマッチした最初のレコード取得
    $flight = App\Flight::where('active', 1)->first();

また、主キーの配列を`find`メソッドに渡し、呼び出すこともできます。一致したレコードのコレクションが返されます。

    $flights = App\Flight::find([1, 2, 3]);

#### Not Found例外

モデルが見つからない時に、例外を投げたい場合もあります。これは特にルートやコントローラの中で便利です。`findOrFail`メソッドとクエリの最初の結果を取得する`firstOrFail`メソッドは、該当するレコードが見つからない場合に`Illuminate\Database\Eloquent\ModelNotFoundException`例外を投げます。

    $model = App\Flight::findOrFail(1);

    $model = App\Flight::where('legs', '>', 100)->firstOrFail();

この例外がキャッチされないと自動的に`404`HTTPレスポンスがユーザーに送り返されます。これらのメソッドを使用すればわざわざ明確に`404`レスポンスを返すコードを書く必要はありません。

    Route::get('/api/flights/{id}', function ($id) {
        return App\Flight::findOrFail($id);
    });

<a name="retrieving-aggregates"></a>
### 集計の取得

もちろん[クエリビルダ](/docs/{{version}}/queries)が提供している`count`、`sum`、`max`や、その他の[集計関数](/docs/{{version}}/queries#aggregates)を使用することもできます。これらのメソッドは完全なモデルインスタンスではなく、最適なスカラー値を返します。

    $count = App\Flight::where('active', 1)->count();

    $max = App\Flight::where('active', 1)->max('price');

<a name="inserting-and-updating-models"></a>
## モデルの追加と更新

<a name="inserts"></a>
### Inserts

モデルから新しいレコードを作成するには新しいインスタンスを作成し、`save`メソッドを呼び出します。

    <?php

    namespace App\Http\Controllers;

    use App\Flight;
    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;

    class FlightController extends Controller
    {
        /**
         * 新しいflightインスタンスの生成
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // リクエストのバリデート処理…

            $flight = new Flight;

            $flight->name = $request->name;

            $flight->save();
        }
    }

この例では、受信したHTTPリクエストの`name`パラメーターを`App\Flight`モデルインスタンスの`name`属性に代入しています。`save`メソッドが呼ばれると新しいレコードがデータベースに挿入されます。`save`が呼び出された時に`created_at`と`updated_at`タイムスタンプは自動的に設定されますので、わざわざ設定する必要はありません。

<a name="updates"></a>
### Updates

`save`メソッドはデータベースで既に存在するモデルを更新するためにも使用されます。モデルを更新するにはまず取得する必要があり、更新したい属性をセットしてそれから`save`メソッドを呼び出します。この場合も`updated_at`タイムスタンプは自動的に更新されますので、値を指定する手間はかかりません。

    $flight = App\Flight::find(1);

    $flight->name = 'New Flight Name';

    $flight->save();

#### 複数モデル更新

指定したクエリに一致する複数のモデルに対し更新することもできます。以下の例では`active`で到着地(`destination`)が`San Diego`の全フライトに遅延(`delayed`)のマークを付けています。

    App\Flight::where('active', 1)
              ->where('destination', 'San Diego')
              ->update(['delayed' => 1]);

`update`メソッドは更新したいカラムと値の配列を受け取ります。

> {note} Eloquentの複数モデル更新を行う場合、更新モデルに対する`saved`と`updated`モデルイベントは発行されません。その理由は複数モデル更新を行う時、実際にモデルが取得されるわけではないからです。

<a name="mass-assignment"></a>
### 複数代入

一行だけで新しいモデルを保存するには、`create`メソッドが利用できます。挿入されたモデルインスタンスが、メソッドから返されます。しかし、これを利用する前に、Eloquentモデルはデフォルトで複数代入から保護されているため、モデルへ`fillable`か`guarded`属性のどちらかを設定する必要があります。

複数代入の脆弱性はリクエストを通じて予期しないHTTPパラメーターが送られた時に起き、そのパラメーターはデータベースのカラムを予期しないように変更してしまうでしょう。たとえば悪意のあるユーザーがHTTPパラメーターで`is_admin`パラメーターを送り、それがモデルの`create`メソッドに対して渡されると、そのユーザーは自分自身を管理者(administrator)に昇格できるのです。

ですから最初に複数代入したいモデルの属性を指定してください。モデルの`$fillable`プロパティで指定できます。たとえば、`Flight`モデルの複数代入で`name`属性のみ使いたい場合です。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Flight extends Model
    {
        /**
         * 複数代入する属性
         *
         * @var array
         */
        protected $fillable = ['name'];
    }

複数代入する属性を指定したら、新しいレコードをデータベースに挿入するために`create`が利用できます。`create`メソッドは保存したモデルインスタンスを返します。

    $flight = App\Flight::create(['name' => 'Flight 10']);

既に存在するモデルインスタンスへ属性を指定したい場合は、`fill`メソッドを使い、配列で指定してください。

    $flight->fill(['name' => 'Flight 22']);

#### 属性の保護

`$fillable`が複数代入時における属性の「ホワイトリスト」として動作する一方、`$guarded`の使用を選ぶことができます。`$guarded`プロパティは複数代入したくない属性の配列です。配列に含まれない他の属性は全部複数代入可能です。そのため`$guarded`は「ブラックリスト」として働きます。重要なのは、`$fillable`か`$guarded`のどちらか一方を使用することです。両方一度には使えません。以下の例は、**`price`を除いた**全属性に複数代入できます。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Flight extends Model
    {
        /**
         * 複数代入しない属性
         *
         * @var array
         */
        protected $guarded = ['price'];
    }

全属性を複数代入可能にする場合は、`$guarded`プロパティに空の配列を定義します。

    /**
     * 複数代入しない属性
     *
     * @var array
     */
    protected $guarded = [];

<a name="other-creation-methods"></a>
### 他の生成メソッド

#### `firstOrCreate`/ `firstOrNew`

他にも属性の複数代入可能な生成メソッドが２つあります。`firstOrCreate`と`firstOrNew`です。`firstOrCreate`メソッドは指定されたカラム／値ペアでデータベースレコードを見つけようします。モデルがデータベースで見つからない場合は、最初の引数が表す属性、任意の第２引数があればそれが表す属性も同時に含む、レコードが挿入されます。

`firstOrNew`メソッドも`firstOrCreate`のように指定された属性にマッチするデータベースのレコードを見つけようとします。しかしモデルが見つからない場合、新しいモデルインスタンスが返されます。`firstOrNew`が返すモデルはデータベースに保存されていないことに注目です。保存するには`save`メソッドを呼び出す必要があります。

    // nameでフライトを取得するか、存在しなければ作成する
    $flight = App\Flight::firstOrCreate(['name' => 'Flight 10']);

    // nameでフライトを取得するか、存在しなければ指定されたname、delayed、arrival_timeを含め、インスタンス化する
    $flight = App\Flight::firstOrCreate(
        ['name' => 'Flight 10'],
        ['delayed' => 1, 'arrival_time' => '11:30']
    );

    // nameで取得するか、インスタンス化する
    $flight = App\Flight::firstOrNew(['name' => 'Flight 10']);

    // Retrieve by name, or instantiate with the name, delayed, and arrival_time attributes...
    $flight = App\Flight::firstOrNew(
        ['name' => 'Flight 10'],
        ['delayed' => 1, 'arrival_time' => '11:30']
    );

#### `updateOrCreate`

また、既存のモデルを更新するか、存在しない場合は新しいモデルを作成したい状況も存在します。これを一度に行うために、Laravelでは`updateOrCreate`メソッドを提供しています。`firstOrCreate`メソッドと同様に、`updateOrCreate`もモデルを保存するため、`save()`を呼び出す必要はありません。

    // OaklandからSan Diego行きの飛行機があれば、料金へ９９ドルを設定する。
    // 一致するモデルがなければ、作成する。
    $flight = App\Flight::updateOrCreate(
        ['departure' => 'Oakland', 'destination' => 'San Diego'],
        ['price' => 99, 'discounted' => 1]
    );

<a name="deleting-models"></a>
## モデル削除

モデルを削除するには、モデルに対し`delete`メソッドを呼び出します。

    $flight = App\Flight::find(1);

    $flight->delete();

#### キーによる既存モデルの削除

上記の例では`delete`メソッドを呼び出す前に、データベースからモデルを取得しています。しかしモデルの主キーが分かっている場合は、モデルを取得せずに`destroy`メソッドで削除できます。さらに、引数に主キーを一つ指定できるだけでなく、`destroy`メソッドは主キーの配列や、主キーの[コレクション](/docs/{{version}}/collections)を引数に指定することで、複数のキーを指定できます。

    App\Flight::destroy(1);

    App\Flight::destroy(1, 2, 3);

    App\Flight::destroy([1, 2, 3]);

    App\Flight::destroy(collect([1, 2, 3]));

#### クエリによるモデル削除

一連のモデルに対する削除文を実行することもできます。次の例はactiveではない印を付けられたフライトを削除しています。複数モデル更新と同様に、複数削除は削除されるモデルに対するモデルイベントを発行しません。

    $deletedRows = App\Flight::where('active', 0)->delete();

> {note} 複数削除文をEloquentにより実行する時、削除対象モデルに対する`deleting`と`deleted`モデルイベントは発行されません。なぜなら、削除文の実行時に、実際にそのモデルが取得されるわけではないためです。

<a name="soft-deleting"></a>
### ソフトデリート

本当にデータベースからレコードを削除する方法に加え、Eloquentはモデルの「ソフトデリート」も行えます。モデルがソフトデリートされても実際にはデータベースのレコードから削除されません。代わりにそのモデルに`deleted_at`属性がセットされ、データベースへ書き戻されます。モデルの`deleted_at`の値がNULLでない場合、ソフトデリートされています。モデルのソフトデリートを有効にするには、モデルに`Illuminate\Database\Eloquent\SoftDeletes`トレイトを使います。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\SoftDeletes;

    class Flight extends Model
    {
        use SoftDeletes;
    }

> {tip} The `SoftDeletes` trait will automatically cast the `deleted_at` attribute to a `DateTime` / `Carbon` instance for you.

データベーステーブルにも`deleted_at`カラムを追加する必要があります。Laravel[スキーマビルダ](/docs/{{version}}/migrations)にはこのカラムを作成するメソッドが存在しています。

    Schema::table('flights', function (Blueprint $table) {
        $table->softDeletes();
    });

これでモデルに対し`delete`メソッドを使用すれば、`deleted_at`カラムに現在の時間がセットされます。ソフトデリートされたモデルに対しクエリがあっても、削除済みのモデルはクエリ結果に含まれません。

指定されたモデルインスタンスがソフトデリートされているかを確認するには、`trashed`メソッドを使います。

    if ($flight->trashed()) {
        //
    }

<a name="querying-soft-deleted-models"></a>
### ソフトデリート済みモデルのクエリ

#### ソフトデリート済みモデルも含める

前述のようにソフトデリートされたモデルは自動的にクエリの結果から除外されます。しかし結果にソフトデリート済みのモデルを含めるように強制したい場合は、クエリに`withTrashed`メソッドを使ってください。

    $flights = App\Flight::withTrashed()
                    ->where('account_id', 1)
                    ->get();

`withTrashed`メソッドは[リレーション](/docs/{{version}}/eloquent-relationships)のクエリにも使えます。

    $flight->history()->withTrashed()->get();

#### ソフトデリート済みモデルのみの取得

`onlyTrashed`メソッドによりソフトデリート済みのモデル**のみ**を取得できます。

    $flights = App\Flight::onlyTrashed()
                    ->where('airline_id', 1)
                    ->get();

#### ソフトデリートの解除

時にはソフトデリート済みのモデルを「未削除」に戻したい場合も起きます。ソフトデリート済みモデルを有効な状態に戻すには、そのモデルインスタンスに対し`restore`メソッドを使ってください。

    $flight->restore();

複数のモデルを手っ取り早く未削除に戻すため、クエリに`restore`メソッドを使うこともできます。他の「複数モデル」操作と同様に、この場合も復元されるモデルに対するモデルイベントは、発行されません。

    App\Flight::withTrashed()
            ->where('airline_id', 1)
            ->restore();

`withTrashed`メソッドと同様、`restore`メソッドは[リレーション](/docs/{{version}}/eloquent-relationships)に対しても使用できます。

    $flight->history()->restore();

#### モデルの完全削除

データベースからモデルを本当に削除する場合もあるでしょう。データベースからソフトデリート済みモデルを永久に削除するには`forceDelete`メソッドを使います。

    // １モデルを完全に削除する
    $flight->forceDelete();

    // 関係するモデルを全部完全に削除する
    $flight->history()->forceDelete();

<a name="query-scopes"></a>
## クエリスコープ

<a name="global-scopes"></a>
### グローバルスコープ

グローバルスコープにより、指定したモデルの**全**クエリに対して、制約を付け加えることができます。Laravel自身の[ソフトデリート](#soft-deleting)機能は、「削除されていない」モデルをデータベースから取得するためにグローバルスコープを使用しています。独自のグローバルスコープを書くことにより、特定のモデルのクエリに制約を確実に、簡単に、便利に指定できます。

#### グローバルスコープの記述

グローバルスコープは簡単に書けます。`Illuminate\Database\Eloquent\Scope`インターフェイスを実装したクラスを定義します。このインターフェイスは、`apply`メソッドだけを実装するように要求しています。`apply`メソッドは必要に応じ、`where`制約を追加します。

    <?php

    namespace App\Scopes;

    use Illuminate\Database\Eloquent\Scope;
    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\Builder;

    class AgeScope implements Scope
    {
        /**
         * Eloquentクエリビルダへ適用するスコープ
         *
         * @param  \Illuminate\Database\Eloquent\Builder  $builder
         * @param  \Illuminate\Database\Eloquent\Model  $model
         * @return void
         */
        public function apply(Builder $builder, Model $model)
        {
            $builder->where('age', '>', 200);
        }
    }

> {tip} クエリのSELECT節にカラムを追加するグローバルスコープの場合は、`select`の代わりに`addSelect`メソッドを使用してください。これにより、クエリの存在するSELECT節を意図せずに置き換えてしまうのを防げます。

#### グローバルスコープの適用

モデルにグローバルスコープを適用するには、そのモデルの`boot`メソッドをオーバライドし、`addGlobalScope`メソッドを呼び出します。

    <?php

    namespace App;

    use App\Scopes\AgeScope;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * モデルの「初期起動」メソッド
         *
         * @return void
         */
        protected static function boot()
        {
            parent::boot();

            static::addGlobalScope(new AgeScope);
        }
    }

スコープを追加した後から、`User::all()`は以下のクエリを生成するようになります。

    select * from `users` where `age` > 200

#### クロージャによるグローバルスコープ

Eloquentではクロージャを使ったグローバルスコープも定義できます。独立したクラスを使うだけの理由がない、簡単なスコープを使いたい場合、特に便利です。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\Builder;

    class User extends Model
    {
        /**
         * モデルの「初期起動」メソッド
         *
         * @return void
         */
        protected static function boot()
        {
            parent::boot();

            static::addGlobalScope('age', function (Builder $builder) {
                $builder->where('age', '>', 200);
            });
        }
    }

#### グローバルスコープの削除

特定のクエリからグローバルスコープを削除した場合は、`withoutGlobalScope`メソッドを使います。唯一の引数として、クラス名を受けます。

    User::withoutGlobalScope(AgeScope::class)->get();

もしくは、クロージャを使用し、グローバルスコープを定義している場合は：

    User::withoutGlobalScope('age')->get();

複数、もしくは全部のグローバルスコープを削除したい場合も、`withoutGlobalScopes`メソッドが使えます。

    // 全グローバルスコープの削除
    User::withoutGlobalScopes()->get();

    // いくつかのグローバルスコープの削除
    User::withoutGlobalScopes([
        FirstScope::class, SecondScope::class
    ])->get();

<a name="local-scopes"></a>
### ローカルスコープ

ローカルスコープによりアプリケーション全体で簡単に再利用可能な、一連の共通制約を定義できます。例えば、人気のある(popular)ユーザーを全員取得する必要が、しばしばあるとしましょう。スコープを定義するには、`scope`を先頭につけた、Eloquentモデルのメソッドを定義します。

スコープはいつもクエリビルダインスタンスを返します。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * 人気のあるユーザーだけに限定するクエリスコープ
         *
         * @param \Illuminate\Database\Eloquent\Builder $query
         * @return \Illuminate\Database\Eloquent\Builder
         */
        public function scopePopular($query)
        {
            return $query->where('votes', '>', 100);
        }

        /**
         * アクティブなユーザーだけに限定するクエリスコープ
         *
         * @param \Illuminate\Database\Eloquent\Builder $query
         * @return \Illuminate\Database\Eloquent\Builder
         */
        public function scopeActive($query)
        {
            return $query->where('active', 1);
        }
    }

#### ローカルスコープの利用

スコープが定義できたらモデルのクエリ時にスコープメソッドを呼び出せます。しかし、メソッドを呼び出すときは`scope`プレフィックスをつけないでください。様々なスコープをチェーンでつなぎ呼び出すこともできます。例を見てください。

    $users = App\User::popular()->active()->orderBy('created_at')->get();

`or`クエリ操作により、複数のEloquentモデルスコープを組み合わせるには、クロージャのコールバックを使用する必要があります。

    $users = App\User::popular()->orWhere(function (Builder $query) {
        $query->active();
    })->get();

しかし、上記は手間がかかるため、Laravelはクロージャを使用せずにスコープをスラスラとチェーンできるように、"higher order" `orWhere`メソッドを用意しています。

    $users = App\User::popular()->orWhere->active()->get();

#### 動的スコープ

引数を受け取るスコープを定義したい場合もあるでしょう。スコープにパラメーターを付けるだけです。スコープパラメーターは`$query`引数の後に定義しする必要があります。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * 指定したタイプのユーザーだけを含むクエリのスコープ
         *
         * @param  \Illuminate\Database\Eloquent\Builder $query
         * @param  mixed $type
         * @return \Illuminate\Database\Eloquent\Builder
         */
        public function scopeOfType($query, $type)
        {
            return $query->where('type', $type);
        }
    }

これでスコープを呼び出すときにパラメーターを渡せます。

    $users = App\User::ofType('admin')->get();

<a name="comparing-models"></a>
## モデルの比較

時に２つのモデルが「同じ」であるかを判定する必要が起きるでしょう。`is`メソッドは２つのモデルが、同じ主キー、テーブル、データベース接続を持っているかを確認します。

    if ($post->is($anotherPost)) {
        //
    }

<a name="events"></a>
## イベント

Eloquentモデルは多くのイベントを発行します。`creating`、`created`、`updating`、`updated`、`saving`、`saved`、`deleting`、`deleted`、`restoring`、`restored`、`retrieved`のメソッドを利用し、モデルのライフサイクルの様々な時点をフックすることができます。イベントにより特定のモデルクラスが保存されたりアップデートされたりするたび、簡単にコードを実行できるようになります。各イベントは、コンストラクタによりモデルのインスタンスを受け取ります。

`retrieved`は、データベースから既存のモデルを取得した時に発行されます。新しいアイテムが最初に保存される場合に`creating`と`created`イベントが発行されます。既存のアイテムに`save`メソッドが呼び出されると`updating`と`updated`イベントが発行されます。どちらの場合にも`saving`と`saved`イベントは発行されます。

> {note} Eloquentの複数モデル更新を行う場合、更新モデルに対する`saved`と`updated`モデルイベントは発行されません。その理由は複数モデル更新を行う時、実際にモデルが取得されるわけではないからです。

使用するには、Eloquentモデルに`$dispatchesEvents`プロパティを定義します。これにより、Eloquentモデルのライフサイクルの様々な時点を皆さん自身の[イベントクラス](/docs/{{version}}/events)へマップします。

    <?php

    namespace App;

    use App\Events\UserSaved;
    use App\Events\UserDeleted;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Foundation\Auth\User as Authenticatable;

    class User extends Authenticatable
    {
        use Notifiable;

        /**
         * モデルのイベントマップ
         *
         * @var array
         */
        protected $dispatchesEvents = [
            'saved' => UserSaved::class,
            'deleted' => UserDeleted::class,
        ];
    }

Eloquentイベントの定義とマップができたら、[イベントリスナ](https://laravel.com/docs/{{version}}/events#defining-listeners)を使用し、イベントを処理できます。

<a name="observers"></a>
### オブザーバ

#### オブザーバの定義

特定のモデルに対し、多くのイベントをリスニングしている場合、全リスナのグループに対するオブザーバを一つのクラスの中で使用できます。オブザーバクラスは、リッスンしたいEloquentイベントに対応する名前のメソッドを持ちます。これらのメソッドは、唯一の引数としてモデルを受け取ります。`make:observer`　Artisanコマンドで、新しいオブザーバクラスを簡単に生成できます。

    php artisan make:observer UserObserver --model=User

このコマンドは、`App/Observers`ディレクトリへ新しいオブザーバを設置します。このディレクトリが存在しなければ、Artisanが作成します。真新しいオブザーバは、次の通りです。

    <?php

    namespace App\Observers;

    use App\User;

    class UserObserver
    {
        /**
         * Userの"created"イベントを処理
         *
         * @param  \App\User  $user
         * @return void
         */
        public function created(User $user)
        {
            //
        }

        /**
         * Userの"updated"イベントを処理
         *
         * @param  \App\User  $user
         * @return void
         */
        public function updated(User $user)
        {
            //
        }

        /**
         * Userの"deleted"イベントを処理
         *
         * @param  \App\User  $user
         * @return void
         */
        public function deleted(User $user)
        {
            //
        }
    }

オブザーバを登録するには、監視したいモデルに対し、`observe`メソッドを使用します。サービスプロバイダの一つの、`boot`メソッドで登録します。以下の例では、`AppServiceProvider`でオブザーバを登録しています。

    <?php

    namespace App\Providers;

    use App\User;
    use App\Observers\UserObserver;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * アプリケーションサービスの初期起動処理
         *
         * @return void
         */
        public function boot()
        {
            User::observe(UserObserver::class);
        }

        /**
         * サービスプロバイダの登録
         *
         * @return void
         */
        public function register()
        {
            //
        }
    }
