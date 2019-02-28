# Laravel Cashier

- [イントロダクション](#introduction)
- [Cashierのアップデート](#upgrading-cashier)
- [設定](#configuration)
    - [Stripe](#stripe-configuration)
    - [Braintree](#braintree-configuration)
    - [通貨設定](#currency-configuration)
- [定期サブスクリプション](#subscriptions)
    - [サブスクリプション作成](#creating-subscriptions)
    - [サブスクリプション状態の確認](#checking-subscription-status)
    - [プラン変更](#changing-plans)
    - [サブスクリプション数](#subscription-quantity)
    - [サブスクリプションの税金](#subscription-taxes)
    - [サブスクリプション課金日付け](#subscription-anchor-date)
    - [サブスクリプションキャンセル](#cancelling-subscriptions)
    - [サブスクリプション再開](#resuming-subscriptions)
- [サブスクリプションのトレイト](#subscription-trials)
    - [カードの事前登録あり](#with-credit-card-up-front)
    - [カードの事前登録なし](#without-credit-card-up-front)
- [顧客](#customers)
    - [顧客の生成](#creating-customers)
- [カード](#cards)
    - [クレジットカードの取得](#retrieving-credit-cards)
    - [登録済みカードの判定](#determining-if-a-card-is-on-file)
    - [クレジットカードの更新](#updating-credit-cards)
    - [クレジットカードの削除](#deleting-credit-cards)
- [StripeのWebフック処理](#handling-stripe-webhooks)
    - [Webフックハンドラの定義](#defining-webhook-event-handlers)
    - [サブスクリプション不可](#handling-failed-subscriptions)
    - [Webフック署名の確認](#verifying-webhook-signatures)
- [BraintreeのWebフック処理](#handling-braintree-webhooks)
    - [Webフックハンドラの定義](#defining-braintree-webhook-event-handlers)
    - [サブスクリプション不可](#handling-braintree-failed-subscriptions)
- [一回だけの課金](#single-charges)
    - [シンプルな課金](#simple-charge)
    - [インボイス付き課金](#charge-with-invoice)
    - [払い戻し](#refunding-charges)
- [インボイス](#invoices)
    - [インボイスPDF生成](#generating-invoice-pdfs)

<a name="introduction"></a>
## イントロダクション

Laravel Cashierは[Stripe](https://stripe.com)と[Braintree](https://www.braintreepayments.com)によるサブスクリプション（定期課金）サービスの読みやすく、スラスラと記述できるインターフェイスを提供します。これにより書くのが恐ろしくなるような、サブスクリプション支払いのための決まりきったコードのほとんどが処理できます。基本的なサブスクリプション管理に加え、Cashierはクーポン、サブスクリプションの変更、サブスクリプション数、キャンセル猶予期間、さらにインボイスのPDF発行まで行います。

> {note} サブスクリプションを提供せず、「一回だけ」の支払いを取り扱う場合は、Cashierを使用してはいけません。StripeかBraintreeのSDKを直接使用してください。

<a name="upgrading-cashier"></a>
## Cashierのアップデート

Cashierのメジャーアップデートを行う場合は、[アップグレードガイド](https://github.com/laravel/cashier/blob/master/UPGRADE.md)を注意深く読み、確認してください。

<a name="configuration"></a>
## 設定

<a name="stripe-configuration"></a>
### Stripe

#### Composer

最初に、Stripe向けのChashierパッケージを依存パッケージへ追加します。

    composer require laravel/cashier

#### データベースマイグレーション

Cashierを使用する前に、[データベースを準備](/docs/{{version}}/migrations)する必要があります。`users`テーブルに、いくつかのカラムを追加し、顧客のサブスクリプション情報すべてを保持する新しい`subscriptions`テーブルを作成します。

    Schema::table('users', function ($table) {
        $table->string('stripe_id')->nullable()->collation('utf8mb4_bin');
        $table->string('card_brand')->nullable();
        $table->string('card_last_four', 4)->nullable();
        $table->timestamp('trial_ends_at')->nullable();
    });

    Schema::create('subscriptions', function ($table) {
        $table->increments('id');
        $table->unsignedInteger('user_id');
        $table->string('name');
        $table->string('stripe_id')->collation('utf8mb4_bin');
        $table->string('stripe_plan');
        $table->integer('quantity');
        $table->timestamp('trial_ends_at')->nullable();
        $table->timestamp('ends_at')->nullable();
        $table->timestamps();
    });

マイグレーションを作成したら、`migrate` Artisanコマンドを実行します。

#### Billableモデル

次に、モデル定義に`Billable`トレイトを追加します。このトレイトは、サブスクリプションの作成、クーポンの適用、クレジットカード情報の更新など、一般的な支払いタスクを実行する様々なメソッドを提供しています。

    use Laravel\Cashier\Billable;

    class User extends Authenticatable
    {
        use Billable;
    }

#### APIキー

最後に、Stripeキーを`services.php`設定ファイルへ設定します。Stripe APIキーはStripeのコントロールパネルから取得します。

    'stripe' => [
        'model'  => App\User::class,
        'key' => env('STRIPE_KEY'),
        'secret' => env('STRIPE_SECRET'),
    ],

<a name="braintree-configuration"></a>
### Braintree

#### Braintreeの注意

多くの操作において、StripeとBraintreeで実装しているCashierの機能は同じものです。両方のサービスはクレジットカードでの支払いを提供していますが、BraintreeはPayPalでの支払いもサポートしています。しかしながら、Braintreeには、Stripeが提供してる機能のいくつかが欠けています。以下の点を考慮し、StripeとBraintreeのどちらを使うのか決めてください。

<div class="content-list" markdown="1">
- BraintreeはPayPalをサポートしていますが、Stripeはしていません。
- Braintreeはサブスクリプション数の`increment`（追加）と`decrement`（減少）メソッドをサポートしていません。これはCashierではなく、Braintreeの制限です。
- Braintreeはパーセンテージをもとにしたディスカウントはサポートしていません。これはCashierではなく、Braintreeの制限です。
</div>

#### Composer

最初に、Braintree向けのCashierパッケージを追加します。

    composer require "laravel/cashier-braintree":"~2.0"

#### クレジットクーポンのプラン

CashierをBraintreeで使用する前に、`plan-credit`ディスカウントをBraintreeのコントロールパネルで定義する必要があります。このディスカウントは、年払いから月払い、もしくは月払いから年払いの変更時に代金を確実に按分するために使用されます。

ディスカウント値はBraintreeコントロールパネルで好きな値に設定できます。Cashierはクーポンを毎回適用するごとに、自身の定義済みカスタム値でディスカウント値をオーバーライドします。Braintreeは繰り返されるサブスクリプションに渡る、按分をネイティブにサポートしていないため、このクーポンが必要です。

#### データベースマイグレーション

Cashierを使用する前に、[データベースも準備](/docs/{{version}}/migrations)する必要があります。`users`テーブルにカラムをいくつか追加し、顧客のサブスクリプション情報を保存するために新しい`subscriptions`テーブルを作成します。

    Schema::table('users', function ($table) {
        $table->string('braintree_id')->nullable();
        $table->string('paypal_email')->nullable();
        $table->string('card_brand')->nullable();
        $table->string('card_last_four')->nullable();
        $table->timestamp('trial_ends_at')->nullable();
    });

    Schema::create('subscriptions', function ($table) {
        $table->increments('id');
        $table->unsignedInteger('user_id');
        $table->string('name');
        $table->string('braintree_id');
        $table->string('braintree_plan');
        $table->integer('quantity');
        $table->timestamp('trial_ends_at')->nullable();
        $table->timestamp('ends_at')->nullable();
        $table->timestamps();
    });

マイグレーションを作成したら、`migrate` Artisanコマンドを実行します。

#### Billableモデル

次に`Billable`トレイをモデルに追加してください。

    use Laravel\Cashier\Billable;

    class User extends Authenticatable
    {
        use Billable;
    }

#### APIキー

次に、以下のオプションを`services.php`ファイルで設定します。

    'braintree' => [
        'model'  => App\User::class,
        'environment' => env('BRAINTREE_ENV'),
        'merchant_id' => env('BRAINTREE_MERCHANT_ID'),
        'public_key' => env('BRAINTREE_PUBLIC_KEY'),
        'private_key' => env('BRAINTREE_PRIVATE_KEY'),
    ],

続いて以下のBraintree SDK呼び出しコードを`AppServiceProvider`サービスプロバイダの`boot`メソッドに追加します。

    \Braintree_Configuration::environment(config('services.braintree.environment'));
    \Braintree_Configuration::merchantId(config('services.braintree.merchant_id'));
    \Braintree_Configuration::publicKey(config('services.braintree.public_key'));
    \Braintree_Configuration::privateKey(config('services.braintree.private_key'));

<a name="currency-configuration"></a>
### 通貨設定

Cashierのデフォルト通貨は米ドル(USD)です。サービスプロバイダの一つで、`boot`メソッド中で`Cashier::useCurrency`メソッドを呼び出し、デフォルト通貨を変更可能です。`useCurrency`メソッドは２つの文字列を引数に取ります。通貨と通貨記号です。

    use Laravel\Cashier\Cashier;

    Cashier::useCurrency('eur', '€');

<a name="subscriptions"></a>
## サブスクリプション

<a name="creating-subscriptions"></a>
### サブスクリプション作成

サブスクリプションを作成するには最初にbillableなモデルのインスタンスを取得しますが、通常は`App\User`のインスタンスでしょう。モデルインスタンスが獲得できたら、モデルのサブスクリプションを作成するために、`newSubscription`メソッドを使います。

    $user = User::find(1);

    $user->newSubscription('main', 'premium')->create($stripeToken);

`newSubscription`メソッドの最初の引数は、サブスクリプションの名前です。アプリケーションでサブスクリプションを一つしか取り扱わない場合、`main`か`primary`と名づけましょう。２つ目の引数には、ユーザーが購入したStripe／Braintreeのプランを指定します。この値はStripe／Braintreeのプランの識別子です。

`create`メソッドはStripeクレジットカード／ソーストークンを引数にとり、サブスクリプションを開始すると同時に、カスタマーIDと関連する支払い情報をデータベースに保存します。

#### ユーザー詳細情報の指定

ユーザーに関する詳細情報を追加したい場合は、`create`メソッドの第２引数に渡すことができます。

    $user->newSubscription('main', 'monthly')->create($stripeToken, [
        'email' => $email,
    ]);

Stripe／Braintreeがサポートしている追加のフィールドについての更なる情報は、Stripeの[顧客の作成](https://stripe.com/docs/api#create_customer)ドキュメントか、[Braintreeのドキュメント](https://developers.braintreepayments.com/reference/request/customer/create/php)を確認してください。

#### クーポン

サブスクリプションの作成時に、クーポンを適用したい場合は、`withCoupon`メソッドを使用してください。

    $user->newSubscription('main', 'monthly')
         ->withCoupon('code')
         ->create($stripeToken);

<a name="checking-subscription-status"></a>
### サブスクリプション状態の確認

ユーザーがアプリケーションで何かを購入したら、バラエティー豊かで便利なメソッドでサブスクリプション状況を簡単にチェックできます。まず初めに`subscribed`メソッドが`true`を返したら、サブスクリプションが現在試用期間であるにしても、そのユーザーはアクティブなサブスクリプションを持っています。

    if ($user->subscribed('main')) {
        //
    }

`subscribed`メソッドは[ルートミドルウェア](/docs/{{version}}/middleware)で使用しても大変役に立つでしょう。ユーザーのサブスクリプション状況に基づいてルートやコントローラへのアクセスをフィルタリングできます。

    public function handle($request, Closure $next)
    {
        if ($request->user() && ! $request->user()->subscribed('main')) {
            // このユーザーは支払っていない顧客
            return redirect('billing');
        }

        return $next($request);
    }

ユーザーがまだ試用期間であるかを調べるには、`onTrial`メソッドを使用します。このメソッドはまだ使用期間中であるとユーザーに警告を表示するために便利です。

    if ($user->subscription('main')->onTrial()) {
        //
    }

`subscribedToPlan`メソッドは、そのユーザーがStripe／BraintreeのプランIDで指定したプランを購入しているかを確認します。以下の例では、ユーザーの`main`サブスクリプションが、購入され有効な`monthly`プランであるかを確認しています。

    if ($user->subscribedToPlan('monthly', 'main')) {
        //
    }

`recurring`メソッドはユーザーが現在サブスクリプションを購入中で、試用期間を過ぎていることを判断するために使用します。

    if ($user->subscription('main')->recurring()) {
        //
    }

#### キャンセルしたサブスクリプションの状態

ユーザーが一度アクティブな購入者になってから、サブスクリプションをキャンセルしたことを調べるには、`cancelled`メソッドを使用します。

    if ($user->subscription('main')->cancelled()) {
        //
    }

また、ユーザーがサブスクリプションをキャンセルしているが、まだ完全に期限が切れる前の「猶予期間」中であるかを調べることもできます。例えば、ユーザーが３月５日にサブスクリプションをキャンセルし、３月１０日に無効になる場合、そのユーザーは３月１０日までは「猶予期間」中です。`subscribed`メソッドは、この期間中、まだ`true`を返すことに注目して下さい。

    if ($user->subscription('main')->onGracePeriod()) {
        //
    }

ユーザーがサブスクリプションをキャンセルし、「猶予期間」を過ぎていることを調べるには、`ended`メソッドを使ってください。

    if ($user->subscription('main')->ended()) {
        //
    }

<a name="changing-plans"></a>
### プラン変更

アプリケーションの購入済みユーザーが新しいサブスクリプションプランへ変更したくなるのはよくあるでしょう。ユーザーを新しいサブスクリプションに変更するには、`swap`メソッドへプランの識別子を渡します。

    $user = App\User::find(1);

    $user->subscription('main')->swap('provider-plan-id');

ユーザーが試用期間中の場合、試用期間は継続します。また、そのプランに「購入数」が存在している場合、購入個数も継続します。

プランを変更し、ユーザーの現プランの試用期間をキャンセルする場合は、`skipTrial`メソッドを使用します。

    $user->subscription('main')
            ->skipTrial()
            ->swap('provider-plan-id');

<a name="subscription-quantity"></a>
### 購入数

> {note} 購入数はStripe版のCashierでのみサポートしています。Braintreeには、Stripeの"quantity"（購入数）にあたる機能がありません。

購入数はサブスクリプションに影響をあたえることがあります。たとえば、あるアプリケーションで「ユーザーごと」に毎月１０ドル課金している場合です。購入数を簡単に上げ下げするには、`incrementQuantity`と`decrementQuantity`メソッドを使います。

    $user = User::find(1);

    $user->subscription('main')->incrementQuantity();

    // 現在の購入数を５個増やす
    $user->subscription('main')->incrementQuantity(5);

    $user->subscription('main')->decrementQuantity();

    // 現在の購入数を５個減らす
    $user->subscription('main')->decrementQuantity(5);

もしくは特定の数量を設置するには、`updateQuantity`メソッドを使ってください。

    $user->subscription('main')->updateQuantity(10);

使用期間による支払いの按分を行わずに、サブスクリプション数を変更する場合は、`noProrate`メソッドを使ってください。

    $user->subscription('main')->noProrate()->updateQuantity(10);

サブスクリプション数の詳細については、[Stripeドキュメント](https://stripe.com/docs/subscriptions/quantities)を読んでください。

<a name="subscription-taxes"></a>
### サブスクリプションの税金

ユーザーが支払うサブスクリプションに対する税率を指定するには、Billableモデルへ`taxPercentage`メソッドを実装し、小数点以下が１桁以内で、0から１００までの数値を返します。

    public function taxPercentage() {
        return 20;
    }

`taxPercentage`メソッドにより、モデルごとに税率を適用できるため、多くの州や国に渡るユーザーベースで税率を決める場合に便利です。

> {note} `taxPercentage`メソッドは、サブスクリプションの課金時のみに適用されます。Cashierで「一回のみ」の支払いを行う場合は、税率を自分で適用する必要があります。

#### 税率の同期

`taxPercentage`が返すハードコードした値を変更する場合、ユーザーに対する既存のサブスクリプションは以前のままになります。`taxPercentage`が返す値に既存のサブスクリプションも更新したい場合は、ユーザーのサブスクリプションインスタンスに対し、`syncTaxPercentage`メソッドを呼び出す必要があります。

    $user->subscription('main')->syncTaxPercentage();

<a name="subscription-anchor-date"></a>
### サブスクリプション課金日付け

> {note} サブスクリプション課金日付けの変更は、Stripeを利用するCashierのみサポートします。

デフォルトで課金日はサブスクリプションが生成された日付け、もしくは使用期間を使っている場合は、使用期間の終了日です。課金日付を変更したい場合は、`anchorBillingCycleOn`メソッドを使用します。

    use App\User;
    use Carbon\Carbon;

    $user = User::find(1);

    $anchor = Carbon::parse('first day of next month');

    $user->newSubscription('main', 'premium')
                ->anchorBillingCycleOn($anchor->startOfDay())
                ->create($stripeToken);

サブスクリプションの課金間隔を管理する情報は、[Stripeの課金サイクルのドキュメント](https://stripe.com/docs/billing/subscriptions/billing-cycle)をお読みください。

<a name="cancelling-subscriptions"></a>
### サブスクリプションキャンセル

サブスクリプションをキャンセルするには`cancel`メソッドをユーザーのサブスクリプションに対して使ってください。

    $user->subscription('main')->cancel();

サブスクリプションがキャンセルされるとCashierは自動的に、データベースの`ends_at`カラムをセットします。このカラムはいつから`subscribed`メソッドが`false`を返し始めればよいのか、判定するために使用されています。例えば、顧客が３月１日にキャンセルしたが、そのサブスクリプションが３月５日に終了するようにスケジュールされていれば、`subscribed`メソッドは３月５日になるまで`true`を返し続けます。

ユーザーがサブスクリプションをキャンセルしたが、まだ「猶予期間」が残っているかどうかを調べるには`onGracePeriod`メソッドを使います。

    if ($user->subscription('main')->onGracePeriod()) {
        //
    }

サブスクリプションを即時キャンセルしたい場合は、ユーザーのサブスクリプションに対し、`cancelNow`メソッドを呼び出してください。

    $user->subscription('main')->cancelNow();

<a name="resuming-subscriptions"></a>
### サブスクリプション再開

ユーザーがキャンセルしたサブスクリプションを、再開したいときには、`resume`メソッドを使用してください。サブスクリプションを再開するには、そのユーザーに有効期間が残っている**必要があります**。

    $user->subscription('main')->resume();

ユーザーがサブスクリプションをキャンセルし、それからそのサブスクリプションを再開する場合、そのサブスクリプションの有効期日が完全に切れていなければすぐに課金されません。そのサブスクリプションはシンプルに再度有効になり、元々の支払いサイクルにより課金されます。

<a name="subscription-trials"></a>
## サブスクリプションのトレイト

<a name="with-credit-card-up-front"></a>
### カードの事前登録あり

顧客へ試用期間を提供し、支払情報を事前に登録してもらう場合、サブスクリプションを作成するときに`trialDays`メソッドを使ってください。

    $user = User::find(1);

    $user->newSubscription('main', 'monthly')
                ->trialDays(10)
                ->create($stripeToken);

このメソッドはデータベースのサブスクリプションレコードへ、試用期間の終了日を設定すると同時に、Stripe／Braintreeへこの期日が過ぎるまで、顧客へ課金を始めないように指示します。

> {note} 顧客のサブスクリプションが試用期間の最後の日までにキャンセルされないと、期限が切れると同時に課金されます。そのため、ユーザーに試用期間の終了日を通知しておくべきでしょう。

`trialUntil`メソッドにより、使用期間の終了時を指定する、`DateTime`インスタンスを渡せます。

    use Carbon\Carbon;

    $user->newSubscription('main', 'monthly')
                ->trialUntil(Carbon::now()->addDays(10))
                ->create($stripeToken);

ユーザーが使用期間中であるかを判定するには、ユーザーインスタンスに対し`onTrial`メソッドを使うか、サブスクリプションインスタンスに対して`onTrial`を使用してください。次の２つの例は、同じ目的を達します。

    if ($user->onTrial('main')) {
        //
    }

    if ($user->subscription('main')->onTrial()) {
        //
    }

<a name="without-credit-card-up-front"></a>
### カードの事前登録なし

事前にユーザーの支払い方法の情報を登録してもらうことなく、試用期間を提供する場合は、そのユーザーのレコードの`trial_ends_at`に、試用の最終日を設定するだけです。典型的な使い方は、ユーザー登録時に設定する方法でしょう。

    $user = User::create([
        // 他のユーザープロパティの設定…
        'trial_ends_at' => now()->addDays(10),
    ]);

> {note} モデル定義の`trial_ends_at`に対する、[日付ミューテタ](/docs/{{version}}/eloquent-mutators#date-mutators)を付け加えるのを忘れないでください。

既存のサブスクリプションと関連付けが行われていないので、Cashierでは、このタイプの試用を「包括的な試用(generic trial)」と呼んでいます。`User`インスタンスに対し、`onTrial`メソッドが`true`を返す場合、現在の日付は`trial_ends_at`の値を過ぎていません。

    if ($user->onTrial()) {
        // ユーザーは試用期間中
    }

特に、ユーザーが「包括的な試用」期間中であり、まだサブスクリプションが作成されていないことを調べたい場合は、`onGenericTrial`メソッドが使用できます。

    if ($user->onGenericTrial()) {
        // ユーザーは「包括的」な試用期間中
    }

ユーザーに実際のサブスクリプションを作成する準備ができたら、通常は`newSubscription`メソッドを使います。

    $user = User::find(1);

    $user->newSubscription('main', 'monthly')->create($stripeToken);

<a name="customers"></a>
## 顧客

<a name="creating-customers"></a>
### 顧客の生成

時にサブスクリプションの定期購入を始めなくても、顧客を生成したい場合があります。それには、`createAsStripeCustomer`メソッドを使用します。

    $user->createAsStripeCustomer();

Stripeで作った顧客に対し、後からサブスクリプションの定期購入を開始可能です。

> {tip} Braintreeの同じ機能は、`createAsBraintreeCustomer`メソッドです。

<a name="cards"></a>
## カード

<a name="retrieving-credit-cards"></a>
### クレジットカードの取得

Billableなモデルインスタンスの`cards`メソッドは、`Laravel\Cashier\Card`インスタンスのコレクションを返します。

    $cards = $user->cards();

デフォルトカードを取得するには、`defaultCard`メソッドを使用します。

    $card = $user->defaultCard();

<a name="determining-if-a-card-is-on-file"></a>
### 登録済みカードの判定

顧客のアカウントに紐付けられているクレジットカードがあるかを確認する場合は、`hasCardOnFile`メソッドを使用します。

    if ($user->hasCardOnFile()) {
        //
    }

<a name="updating-credit-cards"></a>
### クレジットカードの更新

顧客のクレジットカード情報を更新するには、`updateCard`メソッドを使用します。このメソッドはStripeトークンを引数に取り、新しいクレジットカードをデフォルトの支払先に設定します。

    $user->updateCard($stripeToken);

Stripe側の顧客デフォルトカード情報と同期させるには、`updateCardFromStripe`メソッドを使用します。

    $user->updateCardFromStripe();

<a name="deleting-credit-cards"></a>
### クレジットカードの削除

カードを削除するには、最初に`cards`メソッドで顧客のカードを取得しておく必要があります。次に、削除したいカードインスタンスに対し、`delete`メソッドを呼び出します。

    foreach ($user->cards() as $card) {
        $card->delete();
    }

> {note} デフォルトカードを削除する場合は、データベース上の新しいデフォルトカードを同期させるために、`updateCardFromStripe`メソッドを必ず呼び出してください。

アプリケーションに保存されている全てのカード情報を削除するには、`deleteCards`メソッドを使用します。

    $user->deleteCards();

> {note} ユーザーにアクティブなサブスクリプションがある場合、最後の支払い方法が削除されないように考慮する必要があるでしょう。

<a name="handling-stripe-webhooks"></a>
## StripeのWebフック処理

StripeとBraintree、両方共にWebフックによりアプリケーションへ様々なイベントを通知できます。StripeのWebフックを処理するには、CashierのWebフックコントローラへのルートを定義する必要があります。このコントローラは通知されるWebフックリクエストをすべて処理し、正しいコントローラメソッドをディスパッチします。

    Route::post(
        'stripe/webhook',
        '\Laravel\Cashier\Http\Controllers\WebhookController@handleWebhook'
    );

> {note} ルートを登録したら、Stripeコントロールパネル設定のWebフックURLも、合わせて設定してください。

このコントローラはデフォルトで、（Stripeの設定により決まる）課金の失敗が多すぎる場合や顧客の変更、顧客の削除、サブスクリプションのアップデート、クレジットカードの変更時に、サブスクリプションを自動的にキャンセル処理します。処理したいWebフックイベントをどれでも処理できるようにするために、このコントローラを拡張する方法は、以降で説明します。

> {note} Cashierに含まれる、[Webフック署名の確認](/docs/{{version}}/billing#verifying-webhook-signatures)ミドルウェアを使用し、受信リクエストを確実に保護してください。

#### WebフックとCSRF保護

StripeのWebフックでは、Laravelの [CSRFバリデーション](/docs/{{version}}/csrf)をバイパスする必要があるため、`VerifyCsrfToken`ミドルウェアのURIを例外としてリストしておくか、ルート定義を`web`ミドルウェアグループのリストから外しておきましょう。

    protected $except = [
        'stripe/*',
    ];

<a name="defining-webhook-event-handlers"></a>
### Webフックハンドラの定義

Cashierは課金の失敗時にサブスクリプションを自動的に処理しますが、他にもStripeのWebフックイベントを処理したい場合は、Webフックコントローラを拡張します。メソッド名はCashierの命名規則に沿う必要があります。メソッドは`handle`のプレフィックスで始まり、処理したいStripeのWebフックの名前を「キャメルケース」にします。たとえば、`invoice.payment_succeeded` Webフックを処理する場合は、`handleInvoicePaymentSucceeded`メソッドをコントローラに追加します。

    <?php

    namespace App\Http\Controllers;

    use Laravel\Cashier\Http\Controllers\WebhookController as CashierController;

    class WebhookController extends CashierController
    {
        /**
         * インボイス支払い成功時の処理
         *
         * @param  array  $payload
         * @return \Symfony\Component\HttpFoundation\Response
         */
        public function handleInvoicePaymentSucceeded($payload)
        {
            // イベントの処理
        }
    }

次に、`routes/web.php`の中で、キャッシャーコントローラへのルートを定義します。

    Route::post(
        'stripe/webhook',
        '\App\Http\Controllers\WebhookController@handleWebhook'
    );

<a name="handling-failed-subscriptions"></a>
### サブスクリプション不可

顧客のクレジットカードが有効期限切れだったら？　心配いりません。Cashierは顧客のサブスクリプションを簡単にキャンセルできるWebフックを用意しています。前記と同じように、コントローラのルートを指定するだけです。

    Route::post(
        'stripe/webhook',
        '\Laravel\Cashier\Http\Controllers\WebhookController@handleWebhook'
    );

これだけです！　課金の失敗はコントローラにより捉えられ、処理されます。コントローラはStripeによりサブスクリプションが不能だと判断されると（通常は課金に３回失敗時）、その顧客のサブスクリプションをキャンセルします。

<a name="verifying-webhook-signatures"></a>
### Webフック署名の確認

Webフックを安全にするため、[StripeのWebフック著名](https://stripe.com/docs/webhooks/signatures)が利用できます。便利に利用できるように、Cashierは送信されてきたWebフックリクエストが有効なものか確認するミドルウェアをあらかじめ用意しています。

Webフックの確認を有効にするには、`services`設定ファイルの`stripe.webhook.secret`設定値を確実に設定してください。Webフックの`secret`は、Stripeアカウントダッシュボードで取得できます。

<a name="handling-braintree-webhooks"></a>
## BraintreeのWebフック処理

StripeとBraintree、両方共にWebフックによりアプリケーションへ様々なイベントを通知できます。BraintreeのWebフックを処理するには、CashierのWebフックコントローラへのルートを定義する必要があります。このコントローラは通知されるWebフックリクエストをすべて処理し、正しいコントローラメソッドをディスパッチします。

    Route::post(
        'braintree/webhook',
        '\Laravel\Cashier\Http\Controllers\WebhookController@handleWebhook'
    );

> {note} ルートを登録したら、Braintreeコントロールパネル設定のWebフックURLも、合わせて設定してください。

このコントローラはデフォルトで、（Braintreeの設定により決まる）課金の失敗が多すぎる場合、サブスクリプションを自動的にキャンセル処理します。しかしながら、処理したいWebフックイベントをどれでも処理できるようにするために、このコントローラを拡張する方法を以降で学びます。

#### WebフックとCSRF保護

BraintreeのWebフックでは、Laravelの [CSRFバリデーション](/docs/{{version}}/csrf)をバイパスする必要があるため、`VerifyCsrfToken`ミドルウェアのURIを例外としてリストしておくか、ルート定義を`web`ミドルウェアグループのリストから外しておきましょう。

    protected $except = [
        'braintree/*',
    ];

<a name="defining-braintree-webhook-event-handlers"></a>
### Webフックハンドラの定義

Cashierは課金の失敗時にサブスクリプションを自動的に処理しますが、他にもBraintreeのWebフックイベントを処理したい場合は、Webフックコントローラを拡張します。メソッド名はCashierの命名規則に沿う必要があります。メソッドは`handle`のプレフィックスで始まり、処理したいBraintreeのWebフックの名前を「キャメルケース」にします。たとえば、`dispute_opened` Webフックを処理する場合は、`handleDisputeOpened`メソッドをコントローラに追加します。

    <?php

    namespace App\Http\Controllers;

    use Braintree\WebhookNotification;
    use Laravel\Cashier\Http\Controllers\WebhookController as CashierController;

    class WebhookController extends CashierController
    {
        /**
         * 新しいクレームの処理
         *
         * @param  \Braintree\WebhookNotification  $webhook
         * @return \Symfony\Component\HttpFoundation\Responses
         */
        public function handleDisputeOpened(WebhookNotification $webhook)
        {
            // クレーム開始(dispute opened)Webフックの処理
        }
    }

<a name="handling-braintree-failed-subscriptions"></a>
### サブスクリプション不可

顧客のクレジットカードが有効期限切れだったら？　心配いりません。Cashierは顧客のサブスクリプションを簡単にキャンセルできるWebフックコントローラを含んでいます。コントローラのルートを指定するだけです。

    Route::post(
        'braintree/webhook',
        '\Laravel\Cashier\Http\Controllers\WebhookController@handleWebhook'
    );

これだけです。支払いが失敗すると、コントローラにより捉えられ、処理されます。Braintreeが購入に失敗したと判断すると（通常は支払いに３回失敗した場合）、その顧客のサブスクリプションはキャンセルされます。BraintreeのコントロールパネルでWebフックのURIを設定する必要があることを忘れないで下さい。

<a name="single-charges"></a>
## 一回だけの課金

<a name="simple-charge"></a>
### 課金のみ

> {note} Stripeを使用している場合、`charge`メソッドには**アプリケーションで使用している通貨の最低単位**で金額を指定しますが、Braintreeの`charge`メソッドには、ドル単位の金額を指定します。

すでに何かを購入している顧客のクレジットカードに、「一回だけ」の課金をしたい場合は、billableモデルのインスタンスに対し、`charge`メソッドを使います。

    // Stripeはセント単位で課金する
    $stripeCharge = $user->charge(100);

    // Braintreeはドル単位で課金する
    $user->charge(1);

`charge`メソッドは第２引数に配列を受け付け、裏で作成されるStripe／Braintreeの課金作成に対するオプションを指定できます。StripeとBraintreeの課金作成時に使用できるオプションについては、ドキュメントを参照してください。

    $user->charge(100, [
        'custom_option' => $value,
    ]);

`charge`メソッドは、課金に失敗した場合に例外を投げます。課金に成功すると、メソッドはtripe／Braintreeレスポンスをそのまま返します。

    try {
        $response = $user->charge(100);
    } catch (Exception $e) {
        //
    }

<a name="charge-with-invoice"></a>
### インボイス付き課金

一回だけ課金をしつつ、顧客へ発行するPDFのレシートとしてインボイスも生成したいことがあります。`invoiceFor`メソッドは、まさにそのために存在しています。例として、「一回だけ」の料金を５ドル課金してみましょう。

    // Stripeはセント単位で課金する
    $user->invoiceFor('One Time Fee', 500);

    // Braintreeはドル単位で課金する
    $user->invoiceFor('One Time Fee', 5);

金額は即時にユーザーのクレジットカードへ課金されます。`invoiceFor`メソッドは第３引数に配列を受け付けます。この配列はインボイスアイテムへの支払いオプションを含みます。第４引数も配列で、インボイス自身に対する支払いオプションを指定します。

    $user->invoiceFor('Stickers', 500, [
        'quantity' => 50,
    ], [
        'tax_percent' => 21,
    ]);

Braintreeを課金プロバイダに使用している場合、`invoiceFor` の呼び出し時に`description`オプションを含める必要があります。

    $user->invoiceFor('One Time Fee', 500, [
        'description' => 'your invoice description here',
    ]);

> {note} `invoiceFor`メソッドは、課金失敗時にリトライするStripeインボイスを生成します。リトライをしてほしくない場合は、最初に課金に失敗した時点で、Stripe APIを使用し、生成したインボイスを閉じる必要があります。

<a name="refunding-charges"></a>
### 払い戻し

Stripeでの課金を払い戻す必要がある場合は、`refund`メソッドを使用します。このメソッドは、Stripe課金IDのみ引数に取ります。

    $stripeCharge = $user->charge(100);

    $user->refund($stripeCharge->id);

<a name="invoices"></a>
## インボイス

`invoices`メソッドにより、billableモデルのインボイスの配列を簡単に取得できます。

    $invoices = $user->invoices();

    // 結果にペンディング中のインボイスも含める
    $invoices = $user->invoicesIncludingPending();

顧客へインボイスを一覧表示するとき、そのインボイスに関連する情報を表示するために、インボイスのヘルパメソッドを表示に利用できます。ユーザーが簡単にダウンロードできるように、テーブルで全インボイスを一覧表示する例を見てください。

    <table>
        @foreach ($invoices as $invoice)
            <tr>
                <td>{{ $invoice->date()->toFormattedDateString() }}</td>
                <td>{{ $invoice->total() }}</td>
                <td><a href="/user/invoice/{{ $invoice->id }}">Download</a></td>
            </tr>
        @endforeach
    </table>

<a name="generating-invoice-pdfs"></a>
### インボイスPDF生成

ルートやコントローラの中で`downloadInvoice`メソッドを使うと、そのインボイスのPDFダウンロードを生成できます。このメソッドはブラウザへダウンロードのHTTPレスポンスを正しく行うHTTPレスポンスを生成します。

    use Illuminate\Http\Request;

    Route::get('user/invoice/{invoice}', function (Request $request, $invoiceId) {
        return $request->user()->downloadInvoice($invoiceId, [
            'vendor'  => 'Your Company',
            'product' => 'Your Product',
        ]);
    });
