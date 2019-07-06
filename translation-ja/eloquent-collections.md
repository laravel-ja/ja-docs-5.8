# Eloquent：コレクション

- [イントロダクション](#introduction)
- [使用できるメソッド](#available-methods)
- [カスタムコレクション](#custom-collections)

<a name="introduction"></a>
## イントロダクション

`get`メソッドであれリレーションによるものであれ、Eloquentが複数のレコードをリターンする場合`Illuminate\Database\Eloquent\Collection`オブジェクトが返されます。EloquentコレクションオブジェクトはLaravelの[ベースコレクション](/docs/{{version}}/collections)を継承しているので、Eloquentモデルの裏にある配列をスムーズに操作するために継承した多くのメソッドがもちろん使用できます。

全コレクションはイテレーターとしても動作し、シンプルなPHP配列のようにループで取り扱うことができます。

    $users = App\User::where('active', 1)->get();

    foreach ($users as $user) {
        echo $user->name;
    }

しかし、コレクションは配列よりもパワフルで直感的なインターフェイスを使ったメソッドチェーンにより、マッピングや要素の省略操作を行うことができます。例としてアクティブでないモデルを削除し、残ったユーザーのファーストネームを集めてみましょう。

    $users = App\User::all();

    $names = $users->reject(function ($user) {
        return $user->active === false;
    })
    ->map(function ($user) {
        return $user->name;
    });

> {note} ほとんどのEloquentコレクションは新しいEloquentコレクションのインスタンスを返しますが、`pluck`、`keys`、`zip`、`collapse`、`flatten`、`flip`メソッドは[ベースのコレクション](/docs/{{version}}/collections)インスタンスを返します。Eloquentモデルをまったく含まないコレクションを返す`map`操作のような場合、自動的にベースコレクションへキャストされます。

<a name="available-methods"></a>
## 使用できるメソッド

全Eloquentコレクションはベースの[Laravelコレクション](/docs/{{version}}/collections#available-methods)オブジェクトを拡張しており、そのためにベースコレクションクラスが提供しているパワフルなメソッドを全部継承しています。

さらに、`Illuminate\Database\Eloquent\Collection`クラスは、モデルコレクションを管理するのに役立つメソッドのスーパーセットを提供しています。ほとんどのメソッドは`Illuminate\Database\Eloquent\Collection`インスタンスを返しますが、いくつかのメソッドは`Illuminate\Support\Collection`インスタンスを返します。

#### `contains($key, $operator = null, $value = null)`

`contains`メソッドは、指定したモデルインスタンスがコレクションに含まれるかを判定します。このメソッドは主キーかモデルインスタンスを引数に取ります。

    $users->contains(1);

    $users->contains(User::find(1));

#### `diff($items)`

`diff`メソッドは、指定したコレクション中に存在しないモデルをすべて返します。

    use App\User;

    $users = $users->diff(User::whereIn('id', [1, 2, 3])->get());

#### `except($keys)`

`except`メソッドは、指定した主キーを持たないモデルをすべて返します。

    $users = $users->except([1, 2, 3]);

#### `find($key)` {#collection-method .first-collection-method}

`find`メソッドは、指定した主キーのモデルを見つけます。`$key`がモデルインスタンスの場合、`find`はその主キーと一致するモデルを返そうとします。`$key`がキーの配列の場合は`whereIn()`を使用して、`$key`と一致するモデルをすべて返します。

    $users = User::all();

    $user = $users->find(1);

#### `fresh($with = [])`

`fresh`メソッドは、コレクション中の各モデルのインスタンスをデータベースから取得します。更に、指定されたリレーションをEagerロードします。

    $users = $users->fresh();

    $users = $users->fresh('comments');

#### `intersect($items)`

`intersect`メソッドは指定したコレクション中にも存在する、すべてのモデルを返します。

    use App\User;

    $users = $users->intersect(User::whereIn('id', [1, 2, 3])->get());

#### `load($relations)`

`load`メソッドは指定したリレーションをコレクションの全モデルに対してEagerロードします。

    $users->load('comments', 'posts');

    $users->load('comments.author');

#### `loadMissing($relations)`

`loadMissing`メソッドは、リレーションがまだロードされていない場合、指定したリレーションをコレクションのすべてのモデルに対してEagerロードします。

    $users->loadMissing('comments', 'posts');

    $users->loadMissing('comments.author');

#### `modelKeys()`

`modelKeys`メソッドは、コレクションの全モデルの主キーを返します。

    $users->modelKeys();

    // [1, 2, 3, 4, 5]

#### `makeVisible($attributes)`

`makeVisible`メソッドはコレクション中の各モデルの、通常「隠されている(hidden)」属性を可視化(Visible)にします。

    $users = $users->makeVisible(['address', 'phone_number']);

#### `makeHidden($attributes)`

 `makeHidden`メソッドはコレクション中の各モデルの、通常「可視化(Visible)されている」属性を可視化隠し(hidden)ます。

    $users = $users->makeHidden(['address', 'phone_number']);

#### `only($keys)`

`only`メソッドは、指定した主キーを持つモデルを全て返します。

    $users = $users->only([1, 2, 3]);

#### `unique($key = null, $strict = false)`

`unique`メソッドは、コレクション中のユニークなモデルをすべて返します。コレクション中の同じタイプで同じ主キーを持つモデルは削除されます。

    $users = $users->unique();

<a name="custom-collections"></a>
## カスタムコレクション

自分で拡張したメソッドを含むカスタム「コレクション」オブジェクトを使いたい場合は、モデルの`newCollection`メソッドをオーバーライドしてください。

    <?php

    namespace App;

    use App\CustomCollection;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * 新しいEloqunetコレクションインスタンスの生成
         *
         * @param  array  $models
         * @return \Illuminate\Database\Eloquent\Collection
         */
        public function newCollection(array $models = [])
        {
            return new CustomCollection($models);
        }
    }

`newCollection`メソッドを定義すれば、Eloquentがそのモデルの「コレクション」インスタンスを返す場合にいつでもカスタムコレクションのインスタンスを受け取れます。アプリケーションの全モデルでカスタムコレクションを使いたい場合は、全モデルが拡張しているモデルのベースクラスで`newCollection`メソッドをオーバーライドしてください。
