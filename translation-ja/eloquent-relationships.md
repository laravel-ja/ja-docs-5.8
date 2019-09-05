# Eloquent：リレーション

- [イントロダクション](#introduction)
- [リレーションの定義](#defining-relationships)
    - [１対１](#one-to-one)
    - [１対多](#one-to-many)
    - [１対多（逆）](#one-to-many-inverse)
    - [多対多](#many-to-many)
    - [カスタム中間テーブルモデルの定義](#defining-custom-intermediate-table-models)
    - [Has One Through](#has-one-through)
    - [Has Many Through](#has-many-through)
- [ポリモーフィックリレーション](#polymorphic-relationships)
    - [１対１](#one-to-one-polymorphic-relations)
    - [１対多](#one-to-many-polymorphic-relations)
    - [多対多](#many-to-many-polymorphic-relations)
    - [カスタムポリモーフィックタイプ](#custom-polymorphic-types)
- [リレーションのクエリ](#querying-relations)
    - [リレーションメソッド 対 動的プロパティ](#relationship-methods-vs-dynamic-properties)
    - [存在するリレーションのクエリ](#querying-relationship-existence)
    - [存在しないリレーションのクエリ](#querying-relationship-absence)
    - [ポリモーフィックリレーションのクエリ](#querying-polymorphic-relationships)
    - [関連するモデルのカウント](#counting-related-models)
- [Eagerロード](#eager-loading)
    - [制約Eager Loads](#constraining-eager-loads)
    - [遅延Eagerロード](#lazy-eager-loading)
- [関連したモデルの挿入／更新](#inserting-and-updating-related-models)
    - [saveメソッド](#the-save-method)
    - [createメソッド](#the-create-method)
    - [Belongs To関係](#updating-belongs-to-relationships)
    - [Many To Many関係](#updating-many-to-many-relationships)
- [親のタイムスタンプの更新](#touching-parent-timestamps)

<a name="introduction"></a>
## イントロダクション

データベーステーブルは大抵の場合、他と関連しています。たとえばブログ投稿（ポスト）は多くのコメントを持つか、それを投稿したユーザーと関連しています。Eloquentはそうしたリレーションを簡単に管理し操作できるようにするとともに、様々なタイプのリレーションをサポートしています。

<div class="content-list" markdown="1">
- [１対１](#one-to-one)
- [１対多](#one-to-many)
- [多対多](#many-to-many)
- [Has One Through](#has-one-through)
- [Has Many Through](#has-many-through)
- [１対１（ポリモーフィック）](#one-to-one-polymorphic-relations)
- [１対多（ポリモーフィック）](#one-to-many-polymorphic-relations)
- [多対多（ポリモーフィック）](#many-to-many-polymorphic-relations)
</div>

<a name="defining-relationships"></a>
## リレーションの定義

Eloquentのリレーション（関係）は、Eloquentモデルクラスのメソッドとして定義します。Eloquentモデル自身と同様にリレーションはパワフルな[クエリビルダ](/docs/{{version}}/queries)として動作しますので、メソッドとして定義しているリレーションはパワフルなメソッドのチェーンとクエリ能力を提供できるのです。例として、posts関係に追加の制約をチェーンしてみましょう。

    $user->posts()->where('active', 1)->get();

リレーションの詳しい使い方へ進む前に、リレーションの各タイプをどのように定義するかを学びましょう。

<a name="one-to-one"></a>
### １対１

１対１関係が基本です。たとえば`User`モデルは`Phone`モデル一つと関係しているとしましょう。このリレーションを定義するには、`phone`メソッドを`User`モデルに設置します。`phone`メソッドはベースのEloquentモデルクラスの`hasOne`メソッドを結果として返す必要があります。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * ユーザーに関連する電話レコードを取得
         */
        public function phone()
        {
            return $this->hasOne('App\Phone');
        }
    }

`hasOne`メソッドの最初の引数は関係するモデルの名前です。リレーションが定義できたらEloquentの動的プロパティを使って、関係したレコードを取得できます。動的プロパティによりモデル上のプロパティのようにリレーションメソッドにアクセスできます。

    $phone = User::find(1)->phone;

Eloquentはリレーションの外部キーがモデル名に基づいていると仮定します。この場合自動的に`Phone`モデルは`user_id`外部キーを持っていると仮定します。この規約をオーバーライドしたければ、`hasOne`メソッドの第２引数を指定してください。

    return $this->hasOne('App\Phone', 'foreign_key');

Eloquentは親の`id`カラム（もしくはカスタム`$primaryKey`）と一致する外部キーの値を持っていると仮定します。言い換えればEloquentはユーザーの`id`カラムの値を`Phone`レコードの`user_id`カラムに存在しないか探します。リレーションで他の`id`を使いたければ、`hadOne`メソッドの第３引数でカスタムキーを指定してください。

    return $this->hasOne('App\Phone', 'foreign_key', 'local_key');

#### 逆の関係の定義

これで`User`から`Phone`モデルへアクセスできるようになりました。今度は`Phone`モデルからそれを所有している`User`へアクセスするリレーションを定義しましょう。`hasOne`の逆のリレーションを定義するには、`belongsTo`メソッドを使います。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Phone extends Model
    {
        /**
         * この電話を所有するUserを取得
         */
        public function user()
        {
            return $this->belongsTo('App\User');
        }
    }

上の例でEloquentは`Phone`モデルの`user_id`に一致する`id`を持つ`User`モデルを見つけようとします。Eloquentはリレーションメソッド名に`_id`のサフィックスを付けた名前をデフォルトの外部キー名とします。しかし`Phone`モデルの外部キーが`user_id`でなければ、`belongsTo`メソッドの第２引数にカスタムキー名を渡してください。

    /**
     * この電話を所有するUserを取得
     */
    public function user()
    {
        return $this->belongsTo('App\User', 'foreign_key');
    }

親のモデルの主キーが`id`でない、もしくは子のモデルと違ったカラムで紐付けたい場合は、親テーブルのカスタムキー名を`belongsTo`メソッドの第３引数に渡してください。

    /**
     * この電話を所有するUserを取得
     */
    public function user()
    {
        return $this->belongsTo('App\User', 'foreign_key', 'other_key');
    }

<a name="one-to-many"></a>
### １対多

１対多リレーションは一つのモデルが他の多くのモデルを所有する関係を定義するために使います。ブログポストが多くのコメントを持つのが一例です。他のEloquentリレーションと同様に１対多リレーションはEloquentモデルの関数として定義します。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Post extends Model
    {
        /**
         * ブログポストのコメントを取得
         */
        public function comments()
        {
            return $this->hasMany('App\Comment');
        }
    }

Eloquentは、`Comment`モデルに対する外部キーを自動的に決めることを心に留めてください。規約によりEloquentは、自分自身のモデル名の「スネークケース」に`_id`のサフィックスをつけた名前と想定します。ですから今回の例でEloquentは、`Comment`モデルの外部キーを`post_id`であると想定します。

リレーションを定義したら、`comments`プロパティによりコメントのコレクションへアクセスできます。Eloquentは「動的プロパティ」を提供しているので、モデルのプロパティとして定義したリレーションメソッドへアクセスできることを覚えておきましょう。

    $comments = App\Post::find(1)->comments;

    foreach ($comments as $comment) {
        //
    }

全リレーションはクエリビルダとしても働きますから、`comments`メソッドを呼び出すときにどのコメントを取得するのかという制約を追加でき、クエリに条件を続けてチェーンでつなげます。

    $comment = App\Post::find(1)->comments()->where('title', 'foo')->first();

`hasOne`メソッドと同様に、外部キーとローカルキーを`hasMany`メソッドに追加の引数として渡すことでオーバーライドできます。

    return $this->hasMany('App\Comment', 'foreign_key');

    return $this->hasMany('App\Comment', 'foreign_key', 'local_key');

<a name="one-to-many-inverse"></a>
### １対多 (Inverse)

これでポストの全コメントにアクセスできます。今度はコメントから親のポストへアクセスできるようにしましょう。`hasMany`リレーションの逆を定義するには子のモデルで`belongsTo`メソッドによりリレーション関数を定義します。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Comment extends Model
    {
        /**
         * このコメントを所有するポストを取得
         */
        public function post()
        {
            return $this->belongsTo('App\Post');
        }
    }

リレーションが定義できたら`Comment`の`Post`モデルを`post`動的プロパティにより取得しましょう。

    $comment = App\Comment::find(1);

    echo $comment->post->title;

前の例でEloquentは`Comment`モデルの`post_id`と一致する`id`の`Post`モデルを見つけようとします。Eloquentはリレーションメソッドの名前として、`_`に続けて主キーのカラム名をサフィックスとして付けた名前をデフォルトの外部キーとします。しかし`Comment`モデルの外部キーが`post_id`でなければ、`belongsTo`メソッドの第２引数にカスタムキー名を指定してください。

    /**
     * このコメントを所有するポストを取得
     */
    public function post()
    {
        return $this->belongsTo('App\Post', 'foreign_key');
    }

親のモデルの主キーが`id`でない、もしくは子のモデルと違ったカラムで紐付けたい場合は、親テーブルのカスタムキー名を`belongsTo`メソッドの第３引数に渡してください。

    /**
     * このコメントを所有するポストを取得
     */
    public function post()
    {
        return $this->belongsTo('App\Post', 'foreign_key', 'other_key');
    }

<a name="many-to-many"></a>
### 多対多

多対多の関係は`hasOne`と`hasMany`リレーションよりも多少複雑な関係です。このような関係として、ユーザー(user)が多くの役目(roles)を持ち、役目(role)も大勢のユーザー(users)に共有されるという例が挙げられます。たとえば多くのユーザーは"管理者"の役目を持っています。`users`、`roles`、`role_user`の３テーブルがこの関係には必要です。`role_user`テーブルは関係するモデル名をアルファベット順に並べたもので、`user_id`と`role_id`を持つ必要があります。

多対多リレーションは`belongsToMany`メソッド呼び出しを記述することで定義します。例として`User`モデルに`roles`メソッドを定義してみましょう。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * userに所属する役目を取得
         */
        public function roles()
        {
            return $this->belongsToMany('App\Role');
        }
    }

リレーションが定義できたら、`roles`動的プロパティを使いユーザーの役割にアクセスできます。

    $user = App\User::find(1);

    foreach ($user->roles as $role) {
        //
    }

他のリレーションタイプと同様にリレーションを制約するクエリを`roles`に続けてチェーンすることができます。

    $roles = App\User::find(1)->roles()->orderBy('name')->get();

前に述べたようにリレーションの結合テーブルの名前を決めるため、Eloquentは２つのモデル名をアルファベット順に結合します。しかしこの規約は自由にオーバーライドできます。`belongsToMany`メソッドの第２引数に渡してください。

    return $this->belongsToMany('App\Role', 'role_user');

結合テーブル名のカスタマイズに加えテーブルのキーカラム名をカスタマイズするには、`belongsToMany`メソッドに追加の引数を渡してください。第３引数はリレーションを定義しているモデルの外部キー名で、一方の第４引数には結合するモデルの外部キー名を渡します。

    return $this->belongsToMany('App\Role', 'role_user', 'user_id', 'role_id');

#### 逆の関係の定義

多対多のリレーションの逆リレーションを定義するには、関連するモデルで`belongsToMany`を呼び出してください。引き続きユーザーと役割の例を続けますが`Role`モデルで`users`メソッドを定義してみましょう。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Role extends Model
    {
        /**
         * 役目を所有するユーザー
         */
        public function users()
        {
            return $this->belongsToMany('App\User');
        }
    }

ご覧の通り一方の`User`と全く同じ定義のリレーションです。違いは`App\User`モデルを参照していることです。同じ`belongsToMany`メソッドを使っているのですから、通常のテーブル名、キーカスタマイズのオプションは逆の多対多リレーションを定義するときでも全て使用できます。

#### 中間テーブルのカラム取得

既に学んだように、多対多リレーションの操作には中間テーブルが必要です。Eloquentこのテーブルを操作する便利な手段を用意しています。例として`User`オブジェクトが関連する`Role`オブジェクトを持っているとしましょう。このリレーションへアクセスした後、モデルの`pivot`属性を使い中間テーブルにアクセスできます。

    $user = App\User::find(1);

    foreach ($user->roles as $role) {
        echo $role->pivot->created_at;
    }

取得したそれぞれの`Role`モデルは`pivot`属性と自動的に結合されます。この属性は中間テーブルを表すモデルを含んでおり、他のElouquentモデルと同様に使用できます。

デフォルトでモデルキーは`pivot`オブジェクト上のものを表しています。中間テーブルがその他の属性を持っている場合、リレーションを定義するときに指定できます。

    return $this->belongsToMany('App\Role')->withPivot('column1', 'column2');

もし中間テーブルの`created_at`、`updated_at`タイムスタンプを自動的に保守したい場合は、`withTimestamps`メソッドをリレーション定義に付けてください。

    return $this->belongsToMany('App\Role')->withTimestamps();

#### `pivot`属性の名前変更

前述の通り、中間テーブルには`pivot`属性を使ってアクセスできます。その際、アプリケーションの目的をより良く反映するために`pivot`属性の名前を変更することができます。

たとえばユーザーがポッドキャストを購読するようなアプリケーションでは、ユーザーとポッドキャストが多対多の関係となっていることがあります。その場合、中間テーブルへアクセスする際の`pivot`属性の名前を`subscription`に変更したいかもしれません。これはリレーションを定義する際に、`as`メソッドを使うことで実現できます。

    return $this->belongsToMany('App\Podcast')
                    ->as('subscription')
                    ->withTimestamps();

これにより、変更した名前で中間テーブルへアクセスできます。

    $users = User::with('podcasts')->get();

    foreach ($users->flatMap->podcasts as $podcast) {
        echo $podcast->subscription->created_at;
    }

#### 中間テーブルのカラムを使った関係のフィルタリング

リレーション定義時に、`wherePivot`や`wherePivotIn`を使い、`belongsToMany`が返す結果をフィルタリングすることも可能です。

    return $this->belongsToMany('App\Role')->wherePivot('approved', 1);

    return $this->belongsToMany('App\Role')->wherePivotIn('priority', [1, 2]);

<a name="defining-custom-intermediate-table-models"></a>
### カスタム中間テーブルモデルの定義

リレーションの中間テーブルを表す、カスタムモデルを定義したい場合は、リレーション定義で`using`メソッドを呼び出します。カスタム多対多ピボットモデルは、`Illuminate\Database\Eloquent\Relations\Pivot`クラス、一方のカスタムポリモーフィック多対多ピボットモデルは、`Illuminate\Database\Eloquent\Relations\MorphPivot`クラスを拡張する必要があります。例として、カスタム`RoleUser`ピボットモデルを使用する、`Role`を定義してみましょう。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Role extends Model
    {
        /**
         * 役目を所有するユーザー
         */
        public function users()
        {
            return $this->belongsToMany('App\User')->using('App\RoleUser');
        }
    }

`RoleUser`定義時に、`Pivot`クラスを拡張します。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Relations\Pivot;

    class RoleUser extends Pivot
    {
        //
    }

中間テーブルからカラムを取得するために、`using`と`withPivot`を組み合わせられます。たとえば、`withPivot`メソッドにカラム名を渡すことにより、`RoleUser`ピボットテーブルから`created_by`と`updated_by`カラムを取得してみましょう。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Role extends Model
    {
        /**
         * 役目を所有するユーザー
         */
        public function users()
        {
            return $this->belongsToMany('App\User')
                            ->using('App\RoleUser')
                            ->withPivot([
                                'created_by',
                                'updated_by'
                            ]);
        }
    }

> **注意：** ピボットモデルでは、`SoftDeletes`トレイトを使わないほうが良いでしょう。ピボットレコードのソフト削除が必要な場合は、ピボットモデルを実際のEloquentモデルに変換することを考えてください。

#### カスタム中間テーブルとIDの増分

カスタム中間テーブルを使用し、他対多リレーションを定義しており、その中間テーブルが自動増加する主キーを持つ場合、カスタム中間テーブルクラスの`incrementing`プロパティを`true`にセットしてください。

    /**
     * IDの自動増加
     *
     * @var bool
     */
    public $incrementing = true;

<a name="has-one-through"></a>
### Has One Through

"has-one-through"（〜経由で１つへ紐づく）リレーションは、一つの仲介関係を通し、モデルを関連付けます。
たとえば、各サプライヤは一人のユーザーを持ち、各ユーザーは一つのユーザー履歴レコードに関連付けられているとすると、サプライヤモデルは、ユーザーを**経由して**ユーザー履歴にアクセスできます。このリレーションを定義するために必要な、データベーステーブルを見てみましょう。

    users
        id - integer
        supplier_id - integer

    suppliers
        id - integer

    history
        id - integer
        user_id - integer

`history`テーブルには`supplier_id`カラムが含まれていませんが、`hasOneThrough`リレーションでサプライヤモデルからユーザー履歴へアクセスできます。このリレーションのテーブル構造を確認できたので、`Supplier`モデルを定義しましょう。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Supplier extends Model
    {
        /**
         * ユーザー履歴の取得
         */
        public function userHistory()
        {
            return $this->hasOneThrough('App\History', 'App\User');
        }
    }

`hasOneThrough`メソッドの第１引数は、アクセスしたい最終的なモデル名で、第２引数は仲介するモデルの名前です。

通常、Eloquent外部キー規約は、リレーションのクエリを実行するときに使用されます。リレーションのキーをカスタマイズしたい場合は、`hasOneThrough`メソッドの第３引数、第４引数に渡します。第３引数は仲介モデルの外部キー名です。第４引数は最終モデルの外部キーの名前です。第５引数はローカルキー、第６引数は仲介モデルのローカルキーです。

    class Supplier extends Model
    {
        /**
         * ユーザー履歴の取得
         */
        public function userHistory()
        {
            return $this->hasOneThrough(
                'App\History',
                'App\User',
                'supplier_id', // usersテーブルの外部キー
                'user_id', // historyテーブルの外部キー
                'id', // suppliersテーブルのローカルキー
                'id' // usersテーブルのローカルキー
            );
        }
    }

<a name="has-many-through"></a>
### Has Many Through

has many through（〜経由で多数へ紐づく）リレーションは、仲介するテーブルを通して直接関連付けしていないテーブルへアクセスするために、便利な近道を提供します。たとえば`Country`モデルは`Users`モデルを経由して、多くの`Posts`を所有することでしょう。テーブルは以下のような構成になります。

    countries
        id - integer
        name - string

    users
        id - integer
        country_id - integer
        name - string

    posts
        id - integer
        user_id - integer
        title - string

たとえ`posts`テーブルに`country_id`が存在しなくても、`hasManyThrough`リレーションではCountryのPostへ`$country->posts`によりアクセスできます。このクエリを行うためにEloquentは仲介する`users`テーブルの`country_id`を調べます。一致するユーザーIDが存在していたら`posts`テーブルのクエリに利用します。

ではリレーションのテーブル構造を理解したところで、`Country`モデルを定義しましょう。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Country extends Model
    {
        /**
         * この国の全ポストを取得
         */
        public function posts()
        {
            return $this->hasManyThrough('App\Post', 'App\User');
        }
    }

`hasManyThrough`メソッドの第一引数は最終的にアクセスしたいモデル名で、第２引数は仲介するモデル名です。

リレーションのクエリ実行時は、典型的なEloquentの外部キー規約が使用されます。リレーションのキーをカスタマイズしたい場合は、`hasManyThrough`メソッドの第３引数と、第４引数を指定してください。第３引数は仲介モデルの外部キー名、第４引数は最終的なモデルの外部キー名です。第５引数はローカルキーで、第６引数は仲介モデルのローカルキーです。

    class Country extends Model
    {
        public function posts()
        {
            return $this->hasManyThrough(
                'App\Post',
                'App\User',
                'country_id', // usersテーブルの外部キー
                'user_id', // postsテーブルの外部キー
                'id', // countriesテーブルのローカルキー
                'id' // usersテーブルのローカルキー
            );
        }
    }

<a name="polymorphic-relationships"></a>
## ポリモーフィックリレーション

ポリモーフィック（Polymorphic：多様性）リレーションは一つの関係で、対象となるモデルを複数のモデルに所属させます。

<a name="one-to-one-polymorphic-relations"></a>
### １対１（ポリモーフィック）

#### テーブル構造

１対１ポリモーフィックリレーションは、１対１リレーションと似ています。しかしながら一つの関連付けで、対象モデルが複数のタイプのモデルに所属できる点が異なります。たとえば、ブログの`Post`と`User`が、`Image`モデルに対してポリモーフィックリレーションを共有しているとしましょう。１対１ポリモーフィックリレーションを使えば、１つでブログポストとユーザーアカウント両方に対し使用できる、画像のリストを利用できます。最初に、テーブル構造を見てみましょう。

    posts
        id - integer
        name - string

    users
        id - integer
        name - string

    images
        id - integer
        url - string
        imageable_id - integer
        imageable_type - string

`imageable_id`と`imageable_type`カラムに注目してください。`imageable_id`カラムは、ポストかユーザーのID値を含みます。一方の`imageable_type`カラムは、親モデルのクラス名を含みます。`imageable`関係がアクセスされた場合に、Eloquentにより`imageable_type`カラムは親のモデルがどんな「タイプ」であるかを決めるために使用されます。

#### モデル構造

次に、このリレーションを構築するために必要なモデルの定義を確認しましょう。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Image extends Model
    {
        /**
         * 所有しているimageableモデルの取得
         */
        public function imageable()
        {
            return $this->morphTo();
        }
    }

    class Post extends Model
    {
        /**
         * ポストのイメージを所得
         */
        public function image()
        {
            return $this->morphOne('App\Image', 'imageable');
        }
    }

    class User extends Model
    {
        /**
         * ユーザーのイメージを所得。
         */
        public function image()
        {
            return $this->morphOne('App\Image', 'imageable');
        }
    }

#### リレーションの取得

データベーステーブルとモデルが定義できたら、モデルを使いリレーションへアクセスできます。例として、ポストに対するイメージを取得してみましょう。`image`動的プロパティが使用できます。

    $post = App\Post::find(1);

    $image = $post->image;

`morphTo`を実行する名前のメソッドでアクセスすることで、ポリモーフィックモデルから親を取得することもできます。この場合、`Image`モデル上の`imageable`メソッドです。では、このメソッドを動的プロパティとして呼び出してみましょう。

    $image = App\Image::find(1);

    $imageable = $image->imageable;

`Image`モデル上の`imageable`リレーションは、そのイメージを所有しているモデルのタイプにより、`Post`か`User`インスタンスのどちらかを返します。

<a name="one-to-many-polymorphic-relations"></a>
### １対多（ポリモーフィック）

#### テーブル構造

１対多ポリモーフィックリレーションは、シンプルな１対多リレーションと似ています。しかしながら一つの関連付けで、対象モデルが複数のタイプのモデルに所属できる点が異なります。たとえば、アプリケーションのユーザーが、ポストと動画の両方に「コメント」できるようにすると想像してください。ポリモーフィックリレーションを使用すれば、両方のシナリオに対し一つの`comments`テーブルで対処できます。最初に、このリレーションを構築するために必要な、テーブル構造を確認しましょう。

    posts
        id - integer
        title - string
        body - text

    videos
        id - integer
        title - string
        url - string

    comments
        id - integer
        body - text
        commentable_id - integer
        commentable_type - string

#### モデル構造

次に、このリレーションを構築するために必要なモデルの定義を確認しましょう。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Comment extends Model
    {
        /**
         * 所有しているcommentableモデルの取得
         */
        public function commentable()
        {
            return $this->morphTo();
        }
    }

    class Post extends Model
    {
        /**
         * 全ポストコメントの取得
         */
        public function comments()
        {
            return $this->morphMany('App\Comment', 'commentable');
        }
    }

    class Video extends Model
    {
        /**
         * 全ビデオコメントの取得
         */
        public function comments()
        {
            return $this->morphMany('App\Comment', 'commentable');
        }
    }

#### リレーションの取得

データベーステーブルとモデルが定義できたら、モデルを使いリレーションにアクセスできます。たとえば、あるポストの全コメントへアクセスするには、`comments`動的プロパティを使います。

    $post = App\Post::find(1);

    foreach ($post->comments as $comment) {
        //
    }

`morphTo`の呼び出しを行うメソッドの名前にアクセスすることにより、ポリモーフィック関連の所有者を取得することもできます。この例の場合、`Comment`モデルの`commentable`メソッドです。では、このメソッドに動的プロパティによりアクセスしましょう。

    $comment = App\Comment::find(1);

    $commentable = $comment->commentable;

`Comment`モデルの`commentable`関係は、`Post`か`video`インスタンスのどちらかを返します。そのコメントを所有しているモデルのタイプにより決まります。

<a name="many-to-many-polymorphic-relations"></a>
### 多対多（ポリモーフィック）

#### テーブル構造

多対多ポリモーフィックリレーションは、`morphOne`や`morphMany`リレーションより、少々複雑です。例として、ブログの`Post`と`Video`モデルが`Tag`モデルに対し、ポリモーフィックリレーションを共有しているとしましょう。多対多ポリモーフィックリレーションにより、一つのユニークなタグリストをブログポストと動画両方で共有できます。最初に、テーブル構造を確認してください。

    posts
        id - integer
        name - string

    videos
        id - integer
        name - string

    tags
        id - integer
        name - string

    taggables
        tag_id - integer
        taggable_id - integer
        taggable_type - string

#### モデル構造

次にモデルにその関係を用意しましょう。`Post`と`Video`モデルは両方ともベースのEloquentクラスの`morphToMany`メソッドを呼び出す`tags`メソッドを持っています。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Post extends Model
    {
        /**
         * ポストに対する全タグを取得
         */
        public function tags()
        {
            return $this->morphToMany('App\Tag', 'taggable');
        }
    }

#### 逆の関係の定義

次に`Tag`モデルで関係する各モデルに対するメソッドを定義する必要があります。たとえばこの例であれば、`posts`メソッドと`videos`メソッドを用意します。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Tag extends Model
    {
        /**
         * このタグをつけた全ポストの取得
         */
        public function posts()
        {
            return $this->morphedByMany('App\Post', 'taggable');
        }

        /**
         * このタグをつけた全ビデオの取得
         */
        public function videos()
        {
            return $this->morphedByMany('App\Video', 'taggable');
        }
    }

#### リレーションの取得

データベーステーブルとモデルが定義できたら、モデルを使いリレーションにアクセスできます。たとえば、ポストに対する全タグへアクセスするには、単に`tags`動的プロパティを使用します。

    $post = App\Post::find(1);

    foreach ($post->tags as $tag) {
        //
    }

さらに`morphedByMany`を呼び出すメソッドの名前にアクセスし、ポリモーフィックモデルからポリモーフィックリレーションの所有者を取得することも可能です。この例の場合`Tag`モデルの`posts`と`videos`メソッドです。では動的プロパティとしてメソッドを呼び出しましょう。

    $tag = App\Tag::find(1);

    foreach ($tag->videos as $video) {
        //
    }

<a name="custom-polymorphic-types"></a>
### カスタムポリモーフィックタイプ

関連付けたモデルのタイプを保存するため、Laravelはデフォルトではっきりと識別できるクラス名を使います。たとえば上記の例で、`Comment`が`Post`か`Video`に所属しているとすると、`commentable_type`はデフォルトで`App\Post`か`App\Video`のどちらかになるでしょう。しかし、データーベースをアプリケーションの内部構造と分離したい場合もあります。その場合、リレーションの"morph map"を定義し、クラス名の代わりに使用する、各モデルに関連づいたテーブル名をEloquentへ指示することができます。

    use Illuminate\Database\Eloquent\Relations\Relation;

    Relation::morphMap([
        'posts' => 'App\Post',
        'videos' => 'App\Video',
    ]);

`morphMap`は、`AppServiceProvider`の`boot`関数で登録できますし、お望みであれば独立したサービスプロバイダを作成し、その中で行うこともできます。

> {note} 既存のアプリケーションへ"morph map"を追加する場合は、データベース中のすべてのmorphable `*_type`カラムの値は完全なクラス名を持っており、"map"名へ変換する必要が起きます。

<a name="querying-relations"></a>
## リレーションのクエリ

Eloquentリレーションは全てメソッドとして定義されているため、リレーションのクエリを実際に記述しなくても、メソッドを呼び出すことで、そのリレーションのインスタンスを取得できます。さらに、すべてのタイプのEloquentリレーションもクエリビルダとしても動作し、データベースに対してSQLが最終的に実行される前に、そのリレーションのクエリをチェーンで続けて記述できます。

たとえばブログシステムで関連した多くの`Post`モデルを持つ`User`モデルを想像してください。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * ユーザーの全ポストの取得
         */
        public function posts()
        {
            return $this->hasMany('App\Post');
        }
    }

次のように`posts`リレーションのクエリに追加の制約を付け加えられます。

    $user = App\User::find(1);

    $user->posts()->where('active', 1)->get();

すべての[クエリビルダ](/docs/{{version}}/queries)メソッドをリレーションで使用することも可能です。ですから、提供している全メソッドを学ぶために、クエリビルダのドキュメントを研究してください。

#### リレーションに`orWhere`を続ける

前記の例では、リレーションのクエリ時に制約を自由に追加できることをデモンストレーションしました。しかし、`orWhere`節はリレーション制約として、同じレベルの論理グループにしてしまうため、使用には注意が必要です。

    $user->posts()
            ->where('active', 1)
            ->orWhere('votes', '>=', 100)
            ->get();

    // select * from posts
    // where user_id = ? and active = 1 or votes >= 100

多くの状況では、カッコに挟まれた条件チェックの論理グループにするために、[制約グループ](/docs/{{version}}/queries#parameter-grouping)を使うほうが目的に叶うでしょう。

    use Illuminate\Database\Eloquent\Builder;

    $user->posts()
            ->where(function (Builder $query) {
                return $query->where('active', 1)
                             ->orWhere('votes', '>=', 100);
            })
            ->get();

    // select * from posts
    // where user_id = ? and (active = 1 or votes >= 100)

<a name="relationship-methods-vs-dynamic-properties"></a>
### リレーションメソッド 対 動的プロパティ

リレーションクエリに追加の制約を加える必要がなければ、そのリレーションへプロパティとしてアクセスできます。`User`と`Post`の例を続けるとして、ユーザーの全ポストには次のようにアクセスできます。

    $user = App\User::find(1);

    foreach ($user->posts as $post) {
        //
    }

動的プロパティは「遅延ロード」されます。つまり実際にアクセスされた時にだけそのリレーションのデータはロードされます。そのため開発者は多くの場合に[Eagerローディング](#eager-loading)を使い、モデルをロードした後にアクセスするリレーションを前もってロードしておきます。Eagerロードはモデルのリレーションをロードするため実行されるSQLクエリを大幅に減らしてくれます。

<a name="querying-relationship-existence"></a>
### 存在するリレーションのクエリ

関連付けたモデルのレコードに基づいて、モデルのレコードに対するマッチングを絞り込みたい場合もあるでしょう。たとえば、最低でも一つのコメントを持つ、全ブログポストを取得したい場合を考えてください。これを行うためには、リレーションの名前を`has`と`orHas`メソッドに渡します。

    // 最低１つのコメントを持つ全ポストの取得
    $posts = App\Post::has('comments')->get();

演算子と数を指定しクエリをカスタマイズすることもできます。

    // ３つ以上のコメントを持つ全ポストの取得
    $posts = App\Post::has('comments', '>=', 3)->get();

ネストした`has`文は「ドット」記法で組立てられます。たとえば最低一つのコメントと評価を持つ全ポストを取得する場合です。

    // 最低１つのコメントと、それに対する評価を持つポストの取得
    $posts = App\Post::has('comments.votes')->get();

もっと強力な機能がお望みならば`has`の問い合わせに"WHERE"で条件を付けられる、`whereHas`や`orWhereHas`を利用して下さい。これらのメソッドによりリレーションの制約にカスタマイズした制約を追加できます。たとえばコメントの内容を調べることです。

    use Illuminate\Database\Eloquent\Builder;

    // Retrieve posts with at least one comment containing words like foo%...
    $posts = App\Post::whereHas('comments', function (Builder $query) {
        $query->where('content', 'like', 'foo%');
    })->get();

    // Retrieve posts with at least ten comments containing words like foo%...
    $posts = App\Post::whereHas('comments', function (Builder $query) {
        $query->where('content', 'like', 'foo%');
    }, '>=', 10)->get();

<a name="querying-relationship-absence"></a>
### 存在しないリレーションのクエリ

モデルへアクセスする時に、結果をリレーションを持たないレコードに限定したい場合があります。たとえばブログで、コメントを**持たない**ポストのみ全て取得したい場合です。これを行うには、`doesntHave`と`orDoesntHave`メソッドにリレーション名を渡してください。

    $posts = App\Post::doesntHave('comments')->get();

もっと強力な機能がお望みなら、`doesntHave`クエリに"WHERE"で条件を付けられる、`whereDoesntHave`と`orWhereDoesntHave`メソッドを使ってください。これらのメソッドはコメントの内容を調べるなど、リレーション制約にカスタム制約を付け加えられます。

    use Illuminate\Database\Eloquent\Builder;

    $posts = App\Post::whereDoesntHave('comments', function (Builder $query) {
        $query->where('content', 'like', 'foo%');
    })->get();

「ドット」記法を使い、ネストしたリレーションに対してクエリを実行できます。たとえば、以下のクエリにより、アカウントを無効(ban)されていない著者の、コメントが存在するすべてのポストを取得できます。

    use Illuminate\Database\Eloquent\Builder;

    $posts = App\Post::whereDoesntHave('comments.author', function (Builder $query) {
        $query->where('banned', 1);
    })->get();

<a name="querying-polymorphic-relationships"></a>
### ポリモーフィックリレーションのクエリ

既存の`MorphTo`リレーションへクエリするには、`whereHasMorph`とこれに対応するメソッドを利用してください。

    use Illuminate\Database\Eloquent\Builder;

    // ポストとビデオと関連付いているコメントをtitle like foo%で取得する
    $comments = App\Comment::whereHasMorph(
        'commentable',
        ['App\Post', 'App\Video'],
        function (Builder $query) {
            $query->where('title', 'like', 'foo%');
        }
    )->get();

    // ポストとビデオと関連付いているコメントをtitle not like foo%で取得する
    $comments = App\Comment::whereDoesntHaveMorph(
        'commentable',
        'App\Post',
        function (Builder $query) {
            $query->where('title', 'like', 'foo%');
        }
    )->get();

関連するモデルに応じて異なった制約を追加するために、`$type`パラメータを使用できます。

    use Illuminate\Database\Eloquent\Builder;

    $comments = App\Comment::whereHasMorph(
        'commentable',
        ['App\Post', 'App\Video'],
        function (Builder $query, $type) {
            $query->where('title', 'like', 'foo%');

            if ($type === 'App\Post') {
                $query->orWhere('content', 'like', 'foo%');
            }
        }
    )->get();

ポリモーフィックモデルの配列を渡す代わりに`*`をワイルドカードとして指定でき、その場合はデータベースからすべてのポリモーフィックタイプをLaravelは取得します。

    use Illuminate\Database\Eloquent\Builder;

    $comments = App\Comment::whereHasMorph('commentable', '*', function (Builder $query) {
        $query->where('title', 'like', 'foo%');
    })->get();

<a name="counting-related-models"></a>
### 関連するモデルのカウント

リレーション結果の件数を実際にレコードを読み込むことなく知りたい場合は、`withCount`メソッドを使います。件数は結果のモデルの`{リレーション名}_count`カラムに格納されます。

    $posts = App\Post::withCount('comments')->get();

    foreach ($posts as $post) {
        echo $post->comments_count;
    }

クエリによる制約を加え、複数のリレーションの件数を取得することも可能です。

    use Illuminate\Database\Eloquent\Builder;

    $posts = App\Post::withCount(['votes', 'comments' => function (Builder $query) {
        $query->where('content', 'like', 'foo%');
    }])->get();

    echo $posts[0]->votes_count;
    echo $posts[0]->comments_count;

同じリレーションに複数の件数を含めるため、リレーション件数結果の別名も付けられます。

    use Illuminate\Database\Eloquent\Builder;

    $posts = App\Post::withCount([
        'comments',
        'comments as pending_comments_count' => function (Builder $query) {
            $query->where('approved', false);
        }
    ])->get();

    echo $posts[0]->comments_count;

    echo $posts[0]->pending_comments_count;

`select`文に`withCount`を組み合わせる場合は、`select`文の後で`withCount`を呼び出してください。

    $posts = App\Post::select(['title', 'body'])->withCount('comments')->get();

    echo $posts[0]->title;
    echo $posts[0]->body;
    echo $posts[0]->comments_count;

<a name="eager-loading"></a>
## Eagerロード

プロパティとしてEloquentリレーションにアクセスする場合、そのリレーションデータは「遅延ロード」されます。つまり、そのリレーションデータが最初にアクセスされるまで、実際にはロードされません。しかし、Eloquentでは、親のモデルに対するクエリと同時にリレーションを「Eagerロード」可能です。EagerロードはN+1クエリ問題の解決策です。N+１クエリ問題を理解するために、`Author`と関連している`Book`モデルを考えてみてください。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Book extends Model
    {
        /**
         * この本を書いた著者を取得
         */
        public function author()
        {
            return $this->belongsTo('App\Author');
        }
    }

では全書籍とその著者を取得しましょう。

    $books = App\Book::all();

    foreach ($books as $book) {
        echo $book->author->name;
    }

このループではまず全ての本をテーブルから取得するために１クエリ実行され、それから著者をそれぞれの本について取得します。ですから２５冊あるならば、このループで２６クエリが発生します。

うれしいことにクエリの数を徹底的に減らすために、Eagerローディングを使うことができます。`with`メソッドを使い指定してください。

    $books = App\Book::with('author')->get();

    foreach ($books as $book) {
        echo $book->author->name;
    }

この操作では２つだけしかクエリが実行されません。

    select * from books

    select * from authors where id in (1, 2, 3, 4, 5, ...)

#### 複数のリレーションに対するEagerロード

一回の操作で異なった複数のリレーションをEagerロードする必要がある場合もあります。その場合でも、`with`メソッドに引数を追加で渡すだけです。

    $books = App\Book::with(['author', 'publisher'])->get();

#### ネストしたEagerロード

ネストしたリレーションをEagerロードする場合は「ドット」記法が使用できます。例としてEloquent文で全著者と著者個人のコンタクトも全部Eagerロードしてみましょう。

    $books = App\Book::with('author.contacts')->get();

#### `morphTo`リレーションのネストしたEagerロード

リレーションにより返される様々なエンティティのリレーションがネストしている、`morphTo`リレーションをEagerロードしたい場合は、`with`メソッドを`morphTo`リレーションの`morphWith`メソッドと組み合わせて使用します。

    <?php

    use Illuminate\Database\Eloquent\Model;

    class ActivityFeed extends Model
    {
        /**
         * アクティビティフィードレコードの親を取得
         */
        public function parentable()
        {
            return $this->morphTo();
        }
    }

この例の場合、`Event`、`Photo`、`Post`モデルで`ActivityFeed`モデルを構成していると仮定しましょう。さらに、`Event`モデルは`Calendar`モデルへ所属し、`Photo`モデルは`Tag`モデルと関連し、`Post`モデルは`Author`モデルへ所属しているとしましょう。

このモデル定義とリレーションを使用して、`ActivityFeed`モデルインスタンスを取得し、全`parentable`モデルとそれぞれのネストしたリレーションをEagerロードできます。

    use Illuminate\Database\Eloquent\Relations\MorphTo;

    $activities = ActivityFeed::query()
        ->with(['parentable' => function (MorphTo $morphTo) {
            $morphTo->morphWith([
                Event::class => ['calendar'],
                Photo::class => ['tags'],
                Post::class => ['author'],
            ]);
        }])->get();

#### 特定カラムのEagerロード

検索しているリレーションの中で全てのカラムが必要とは限りません。そのため、Eloquentではリレーションの中で取得したいカラムを指定することができます。

    $books = App\Book::with('author:id,name')->get();

> {note} この機能を使用する場合は`id`カラムと、取得するカラムリスト中の関連付けられた外部キーカラムすべてを常に含める必要があります。

#### デフォルトのEagerロード

あるモデル取得時、常にリレーションを取得したい場合もあります。そのためには、モデルに`$with`プロパティを定義します。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Book extends Model
    {
        /**
         * 常にロードするリレーション
         *
         * @var array
         */
        protected $with = ['author'];

        /**
         * 本を書いた著者の取得
         */
        public function author()
        {
            return $this->belongsTo('App\Author');
        }
    }

`$with`プロパティで指定したリレーションを一つのクエリで削除したい場合は、`without`メソッドを使用します。

    $books = App\Book::without('author')->get();

<a name="constraining-eager-loads"></a>
### Eagerロードへの制約

ときにリレーションをEagerロードしたいが、Eagerロードクエリに条件を追加したい場合があります。例を見てください。

    $users = App\User::with(['posts' => function ($query) {
        $query->where('title', 'like', '%first%');
    }])->get();

この例でEloquentは、`title`カラムの内容に`first`という言葉を含むポストのみをEagerロードしています。Eagerロード操作を更にカスタマイズするために、他の[クエリビルダ](/docs/{{version}}/queries)を呼び出すこともできます。

    $users = App\User::with(['posts' => function ($query) {
        $query->orderBy('created_at', 'desc');
    }])->get();

> {note} `limit`と`take`クエリビルダメソッドは、Eagerロードの制約時には使用できません。

<a name="lazy-eager-loading"></a>
### 遅延Eagerロード

既に親のモデルを取得した後に、リレーションをEagerロードする必要がある場合もあるでしょう。たとえば、どの関連しているモデルをロードするかを動的に決める場合に便利です。

    $books = App\Book::all();

    if ($someCondition) {
        $books->load('author', 'publisher');
    }

Eagerロードに追加の制約をかける必要があるなら、ロードしたい関連へ配列のキーを付け渡してください。配列地は、クエリインスタンスを受け取る「クロージャ」でなければなりません。

    $books->load(['author' => function ($query) {
        $query->orderBy('published_date', 'asc');
    }]);

リレーションをまだロードしていない場合のみロードする場合は、`loadMissing`メソッドを使用します。

    public function format(Book $book)
    {
        $book->loadMissing('author');

        return [
            'name' => $book->name,
            'author' => $book->author->name
        ];
    }

#### ネストした遅延Eagerローディングと`morphTo`

`morphTo`リレーションをEagerロードし、同時にこのリレーションが数多くのエンティティとネストしたリレーションを返す場合、`loadMorph`メソッドを使用してください。

このメソッドは第１引数として、`morphTo`リレーション名を受け取り、第２引数にはモデル／リレーションのペアの配列を引き受けます。このメソッドを理解するため、次のモデルを考えてください。

    <?php

    use Illuminate\Database\Eloquent\Model;

    class ActivityFeed extends Model
    {
        /**
         * アクティビティ・フィード・レコードの親の取得
         */
        public function parentable()
        {
            return $this->morphTo();
        }
    }

この例で、`Event`、`Photo`、`Post`モデルが`ActivityFeed`モデルにより生成されると仮定してください。さらに、`Event`モデルは`Calendar`モデルへ所属、`Photo`モデルは`Tag`モデルへ所属、`Post`モデルは`Author`モデルへ所属していると想定してください。

これらのモデルとリレーションを使用し、`ActivityFeed`モデルインスタンスと、Eagerロードされた全`parentable`モデル、各個にネストしたリレーションを取得しましょう。

    $activities = ActivityFeed::with('parentable')
        ->get()
        ->loadMorph('parentable', [
            Event::class => ['calendar'],
            Photo::class => ['tags'],
            Post::class => ['author'],
        ]);

<a name="inserting-and-updating-related-models"></a>
## 関連したモデルの挿入／更新

<a name="the-save-method"></a>
### `Save`メソッド

Eloquentは新しいモデルをリレーションに追加するために便利なメソッドを用意しています。たとえば`Post`モデルに新しい`Comment`を挿入する必要がある場合です。`Comment`の`post_id`属性を自分で設定する代わりに、リレーションの`save`メソッドで直接`Comment`を挿入できます。

    $comment = new App\Comment(['message' => 'A new comment.']);

    $post = App\Post::find(1);

    $post->comments()->save($comment);

動的プロパティとして`comments`リレーションにアクセスできない点には注意してください。代わりにリレーションの取得で`comments`メソッドを呼び出しています。`save`メソッドは自動的に新しい`Comment`モデルの`post_id`へ適した値を代入します。

複数の関連したモデルを保存する必要があるなら、`saveMany`メソッドを使用できます。

    $post = App\Post::find(1);

    $post->comments()->saveMany([
        new App\Comment(['message' => 'A new comment.']),
        new App\Comment(['message' => 'Another comment.']),
    ]);

<a name="the-push-method"></a>
#### モデルとリレーションの再帰的保存

モデルと関連付いているリレーション全てを保存(`save`)したい場合は、`push`メソッドを使用してください。

    $post = App\Post::find(1);

    $post->comments[0]->message = 'Message';
    $post->comments[0]->author->name = 'Author Name';

    $post->push();

<a name="the-create-method"></a>
### `Create`メソッド

`save`と`saveMany`メソッドに加え、`create`メソッドも使用できます。属性の配列を引数に受け付け、モデルを作成しデータベースへ挿入します。`save`と`create`の違いは`save`が完全なEloquentモデルを受け付けるのに対し、`create`は普通のPHPの「配列」を受け付ける点です。

    $post = App\Post::find(1);

    $comment = $post->comments()->create([
        'message' => 'A new comment.',
    ]);

> {tip} `create`メソッドを使用する前に属性の[複数代入](/docs/{{version}}/eloquent#mass-assignment)に関するドキュメントを読んでおいてください。

`createMany`メソッドで複数のリレーションモデルを生成することができます。

    $post = App\Post::find(1);

    $post->comments()->createMany([
        [
            'message' => 'A new comment.',
        ],
        [
            'message' => 'Another new comment.',
        ],
    ]);

[リレーション上のモデルを作成、変更](https://laravel.com/docs/{{version}}/eloquent#other-creation-methods)するために、`findOrNew`、`firstOrNew`、`firstOrCreate`、`updateOrCreate`メソッドも使用できます。

<a name="updating-belongs-to-relationships"></a>
### Belongs To関係

`belongsTo`リレーションを更新する場合は`associate`メソッドを使います。このメソッドは子モデルへ外部キーをセットします。

    $account = App\Account::find(10);

    $user->account()->associate($account);

    $user->save();

`belongsTo`リレーションを削除する場合は`dissociate`メソッドを使用します。このメソッドはリレーションの子モデルの外部キーを`null`にします。

    $user->account()->dissociate();

    $user->save();

<a name="default-models"></a>
#### デフォルトモデル

`belongsTo`、`hasOne`、`hasOneThrough`、`morphOne`リレーションでは、指定したリレーションが`null`の場合に返却するデフォルトモデルを定義できます。このパターンは、頻繁に[Nullオブジェクトパターン](https://en.wikipedia.org/wiki/Null_Object_pattern)と呼ばれ、コードから条件のチェックを省くのに役立ちます。以下の例では、ポストに従属する`user`がない場合に、空の`App\User`モデルを返しています。

    /**
     * ポストの著者を取得
     */
    public function user()
    {
        return $this->belongsTo('App\User')->withDefault();
    }

属性を指定したデフォルトモデルを返すためには、`withDefault`メソッドに配列かクロージャを渡してください。

    /**
     * ポストの著者を取得
     */
    public function user()
    {
        return $this->belongsTo('App\User')->withDefault([
            'name' => 'Guest Author',
        ]);
    }

    /**
     * ポストの著者を取得
     */
    public function user()
    {
        return $this->belongsTo('App\User')->withDefault(function ($user, $post) {
            $user->name = 'Guest Author';
        });
    }

<a name="updating-many-to-many-relationships"></a>
### 多対多関係

#### attach／detach

多対多リレーションを操作時により便利なように、Eloquentはヘルパメソッドをいくつか用意しています。例としてユーザーが多くの役割を持ち、役割も多くのユーザーを持てる場合を考えてみましょう。モデルを結びつけている中間テーブルにレコードを挿入することにより、ユーザーに役割を持たせるには`attach`メソッドを使います。

    $user = App\User::find(1);

    $user->roles()->attach($roleId);

モデルにリレーションを割りつけるときに中間テーブルに挿入したい追加のデータを配列で渡すこともできます。

    $user->roles()->attach($roleId, ['expires' => $expires]);

ユーザーから役割を削除する必要がある場合もあるでしょう。多対多リレーションのレコードを削除するには`detach`メソッドを使います。`detach`メソッドは中間テーブルから対応するレコードを削除します。しかし両方のモデルはデータベースに残ります。

    // ユーザーから役割を一つ切り離す
    $user->roles()->detach($roleId);

    // ユーザーから役割を全部切り離す
    $user->roles()->detach();

便利なように`attach`と`detach`の両方共にIDの配列を指定することができます。

    $user = App\User::find(1);

    $user->roles()->detach([1, 2, 3]);

    $user->roles()->attach([
        1 => ['expires' => $expires],
        2 => ['expires' => $expires]
    ]);

#### 関連付けの同期

多対多の関連を構築するために`sync`メソッドも使用できます。`sync`メソッドへは中間テーブルに設置しておくIDの配列を渡します。その配列に指定されなかったIDは中間テーブルから削除されます。ですからこの操作が完了すると、中間テーブルには配列中のIDだけが存在することになります。

    $user->roles()->sync([1, 2, 3]);

IDと一緒に中間テーブルの追加の値を渡すことができます。

    $user->roles()->sync([1 => ['expires' => true], 2, 3]);

存在しているIDを削除したくない場合は、`syncWithoutDetaching`メソッドを使用します。

    $user->roles()->syncWithoutDetaching([1, 2, 3]);

#### 関連の切り替え

多対多リレーションは、指定したIDの関連状態を「切り替える」、`toggle`メソッドも提供しています。指定したIDが現在関連している場合は、関連を切り離します。同様に現在関連がない場合は、関連付けます。

    $user->roles()->toggle([1, 2, 3]);

#### 中間テーブルへの追加データ保存

多対多リレーションを操作する場合、`save`メソッドの第２引数へ追加の中間テーブルの属性を指定できます。

    App\User::find(1)->roles()->save($role, ['expires' => $expires]);

#### 中間テーブルのレコード更新

中間テーブルに存在している行を更新する必要がある場合は、`updateExistingPivot`メソッドを使います。このメソッドは、中間テーブルの外部キーと更新する属性の配列を引数に取ります。

    $user = App\User::find(1);

    $user->roles()->updateExistingPivot($roleId, $attributes);

<a name="touching-parent-timestamps"></a>
## 親のタイムスタンプの更新

`Comment`が`Post`に所属しているように、あるモデルが他のモデルに所属（`belongsTo`もしくは`belongsToMany`）しており、子のモデルが更新される時に親のタイムスタンプを更新できると便利なことがあります。たとえば`Comment`モデルが更新されたら、所有者の`Post`の`updated_at`タイムスタンプを自動的に"touch"したい場合です。Eloquentなら簡単です。子のモデルに`touches`プロパティを追加しリレーション名を指定してください。

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Comment extends Model
    {
        /**
         * 全リレーションをtouch
         *
         * @var array
         */
        protected $touches = ['post'];

        /**
         * ポストに所属しているコメント取得
         */
        public function post()
        {
            return $this->belongsTo('App\Post');
        }
    }

これで`Comment`が更新されると、所有している`Post`の`updated_at`カラムも同時に更新され、これにより`Post`モデルのどの時点のキャッシュを無効にするか判定できます。

    $comment = App\Comment::find(1);

    $comment->text = 'Edit to this comment!';

    $comment->save();
