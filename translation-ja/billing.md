# Laravel Cashier

- [イントロダクション](#introduction)
- [Cashierのアップデート](#upgrading-cashier)
- [インストール](#installation)
- [設定](#configuration)
    - [Billableモデル](#billable-model)
    - [APIキー](#api-keys)
    - [通貨設定](#currency-configuration)
- [顧客](#customers)
    - [顧客の生成](#creating-customers)
- [支払い方法](#payment-methods)
    - [支払い方法の保存](#storing-payment-methods)
    - [支払い方法の取得](#retrieving-payment-methods)
    - [ユーザーが支払い方法を持っているかの判定](#check-for-a-payment-method)
    - [デフォルト支払い方法の更新](#updating-the-default-payment-method)
    - [支払い方法の追加](#adding-payment-methods)
    - [支払い方法の削除](#deleting-payment-methods)
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
    - [支払いの事前登録あり](#with-payment-method-up-front)
    - [支払いの事前登録なし](#without-payment-method-up-front)
- [StripeのWebフック処理](#handling-stripe-webhooks)
    - [Webフックハンドラの定義](#defining-webhook-event-handlers)
    - [サブスクリプション不可](#handling-failed-subscriptions)
    - [Webフック署名の確認](#verifying-webhook-signatures)
- [一回だけの課金](#single-charges)
    - [シンプルな課金](#simple-charge)
    - [インボイス付き課金](#charge-with-invoice)
    - [払い戻し](#refunding-charges)
- [インボイス](#invoices)
    - [インボイスPDF生成](#generating-invoice-pdfs)
- [堅牢な顧客認証 (SCA)](#strong-customer-authentication)
    - [支払い要求の追加確認](#payments-requiring-additional-confirmation)
    - [非セッション確立時の支払い通知](#off-session-payment-notifications)

<a name="introduction"></a>
## イントロダクション

Laravel Cashierは[Stripe](https://stripe.com)によるサブスクリプション（定期課金）サービスの読みやすく、スラスラと記述できるインターフェイスを提供します。これにより書くのが恐ろしくなるような、サブスクリプション支払いのための決まりきったコードのほとんどが処理できます。基本的なサブスクリプション管理に加え、Cashierはクーポン、サブスクリプションの変更、サブスクリプション数、キャンセル猶予期間、さらにインボイスのPDF発行まで行います。

> {note} ブレーキングチェンジを防ぐために、CashierではStripeの固定APIバージョンを使用しています。Cashier10.1では、Stripeの`2019-08-14`付けAPIバージョンを使用しています。Stripeの新機能や機能向上を利用するため、マイナーリリースでもStripe APIのバージョンを更新することがあります。

<a name="upgrading-cashier"></a>
## Cashierのアップデート

新しいバージョンのCashierへアップグレードする場合は、[アップグレードガイド](https://github.com/laravel/cashier/blob/master/UPGRADE.md)を注意深く確認することが重要です。

<a name="installation"></a>
## インストール

はじめに、Stripe向けCashierパッケージをComposerでインストールしてください。

    composer require laravel/cashier

> {note} Stripeの全イベントをCashierで確実に処理するために、[CashierのWebhook処理の準備](#handling-stripe-webhooks)を行なってください。

#### データベースマイグレーション

CashierサービスプロバーダでCashierのデータベースマイグレーションを登録しています。ですから、パッケージをインストールしたら、データベースのマイグレーションを忘れず実行してください。Cashierマイグレーションは`users`テーブルにいくつものカラムを追加し、顧客のサブスクリプションを全て保持するために新しい`subscriptions`テーブルを作成します。

    php artisan migrate

Cashierパッケージに始めから含まれているマイグレーションをオーバーライトしたい場合は、`vendor:publish` Artisanコマンドを使用し公開できます。

    php artisan vendor:publish --tag="cashier-migrations"

Cashierのマイグレーション実行を完全に防ぎたい場合は、Cashierが提供している`ignoreMigrations`を使います。通常、このメソッドは`AppServiceProvider`の`register`メソッドの中で実行すべきです。

    use Laravel\Cashier\Cashier;

    Cashier::ignoreMigrations();

> {note} StripeはStripeの識別子を保存しておくカラムはケースセンシティブ（大文字小文字区別）にするべきだと勧めています。そのため`stripe_id`カラムには、たとえばMySQLでは`utf8_bin`のように、適切なカラムコレーションを確実に指定してください。詳しい情報は、[Stripeのドキュメント](https://stripe.com/docs/upgrades#what-changes-does-stripe-consider-to-be-backwards-compatible)をお読みください。

<a name="configuration"></a>
## 設定

<a name="billable-model"></a>
### Billableモデル

Cashierを使い始める前に、モデル定義に`Billable`トレイトを追加します。このトレイトはサブスクリプションの作成やクーポンの適用、支払い情報の更新などのような、共通の支払いタスク実行を提供する数多くのメソッドを提供しています。

    use Laravel\Cashier\Billable;

    class User extends Authenticatable
    {
        use Billable;
    }

CashierはLaravelにLaravelに含まれている`App\User`クラスがBillableモデルであると仮定しています。これを変更する場合は、`.env`ファイルでモデルを指定してください。

    CASHIER_MODEL=App\User

> {note} Laravelの提供する`App\User`モデル以外のモデルを使用する場合は、提供している[マイグレーション](#installation)を公開し、モデルのテーブル名に一致するように変更する必要があります。

<a name="api-keys"></a>
### APIキー

次に、`.env`ファイルの中のStripeキーを設定する必要があります。Stripe APIキーは、Stripeのコントロールパネルから取得できます。

    STRIPE_KEY=your-stripe-key
    STRIPE_SECRET=your-stripe-secret

<a name="currency-configuration"></a>
### 通貨設定

Cashierのデフォルト通貨は米ドル(USD)です。`CASHIER_CURRENCY`環境変数の指定で、デフォルト通貨を変更可能です。

    CASHIER_CURRENCY=eur

Caishierの通貨設定に付け加え、インボイスで表示する金額のフォーマットをローケルを使い指定することも可能です。Cashierは内部で、通貨のローケルを指定するために、[PHPの`NumberFormatter`クラス](https://www.php.net/manual/en/class.numberformatter.php)を利用しています。

    CASHIER_CURRENCY_LOCALE=nl_BE

> {note} `en`以外のローケルを指定する場合は、サーバ設定で`ext-intl` PHP拡張がインストールされているのを確認してください。

<a name="customers"></a>
## 顧客

<a name="creating-customers"></a>
### 顧客の生成

時々サブスクリプションを開始しなくてもStripeで顧客を作成したい場合があります。`createAsStripeCustomer`を使い、作成できます。

    $user->createAsStripeCustomer();

Stripeで顧客を生成しておけば、後からサブスクリプションを開始できます。

<a name="payment-methods"></a>
## 支払い方法

<a name="storing-payment-methods"></a>
### 支払い方法の保存

Stripeでサブスクリプションを生成するか「一度だけ」の課金を実行するためには、支払い方法を登録し、IDを取得する必要があります。サブスクリプションのための支払いメソッドか、一回だけの課金ためかによりアプローチが異なるため、以下で両方共にみていきましょう。

#### サブスクリプションの支払い方帆

将来の仕様に備えて、顧客のクレジットカードを登録する場合、顧客の支払いメソッドの詳細を安全に集めるためにStripe Setup Intents APIを使う必要があります。"Setup Intent（意図）"は、Stripeに対し顧客の支払いメソッドを登録する意図を示しています。Cashierの`Billable`トレイトは、新しいSetup Intentを簡単に作成できる`createSetupIntent`を含んでいます。顧客の支払いメソッドの詳細情報を集めるフォームをレンダーしたいルートやコントローラから、このメソッドを呼び出してください。

    return view('update-payment-method', [
        'intent' => $user->createSetupIntent()
    ]);

 Setup Intentを作成したらそれをビューに渡し、支払い方法を集める要素にsecretを付け加える必要があります。例えば、このような「支払い方法更新」フォームを考えてください。

    <input id="card-holder-name" type="text">

    <!-- Stripe要素のプレースホルダ -->
    <div id="card-element"></div>

    <button id="card-button" data-secret="{{ $intent->client_secret }}">
        Update Payment Method
    </button>

Stripe.jsライブラリを使い、Stripe要素をフォームに付け加え、顧客の支払いの詳細を安全に収集します。

    <script src="https://js.stripe.com/v3/"></script>

    <script>
        const stripe = Stripe('stripe-public-key');

        const elements = stripe.elements();
        const cardElement = elements.create('card');

        cardElement.mount('#card-element');
    </script>

これで[Stripeの`handleCardSetup`メソッド](https://stripe.com/docs/stripe-js/reference#stripe-handle-card-setup)を使用してカードを検証し、Stripeから安全な「支払い方法識別子」を取得できます。

    const cardHolderName = document.getElementById('card-holder-name');
    const cardButton = document.getElementById('card-button');
    const clientSecret = cardButton.dataset.secret;

    cardButton.addEventListener('click', async (e) => {
        const { setupIntent, error } = await stripe.handleCardSetup(
            clientSecret, cardElement, {
                payment_method_data: {
                    billing_details: { name: cardHolderName.value }
                }
            }
        );

        if (error) {
            // ユーザーに"error.message"を表示する…
        } else {
            // カードの検証に成功した…
        }
    });

Stripeによりカードが検証されたら、顧客に付け加えた`setupIntent.payment_method`の結果をLaravelアプリケーションへ渡すことができます。支払い方法は[新しい支払い方法を追加](#adding-payment-methods)するのと、[デフォルトの支払い方法を使用](#updating-the-default-payment-method)する、どちらかが選べます。[新しい支払い方法を追加](#adding-payment-methods)の支払いメソッド識別子を即時に使用することもできます。

> {tip} Setup Intentsと顧客支払いの詳細情報の収集に関するより詳しい情報は、[Stripeが提供している概要](https://stripe.com/docs/payments/cards/saving-cards#saving-card-without-payment)をご覧ください。

#### 一回のみの課金に対する支払い方法

顧客の支払いメソッドに対し一回のみの課金を作成する場合、ワンタイムの支払いメソッド識別子を使う必要があるだけで済みます。Stripeの制限により、保存されている顧客のデフォルト支払い方法は使用できません。Stripe.jsライブラリを使用し、顧客に支払い方法の詳細を入力してもらえるようにする必要があります。例として、以降のフォームを考えてみましょう。

    <input id="card-holder-name" type="text">

    <!-- Stripe要素のプレースホルダ -->
    <div id="card-element"></div>

    <button id="card-button">
        Process Payment
    </button>

次に、Stripe.jsライブラリを利用しStripeの要素をフォームへ追加し、顧客の支払い情報詳細を安全に収集します。

    <script src="https://js.stripe.com/v3/"></script>

    <script>
        const stripe = Stripe('stripe-public-key');

        const elements = stripe.elements();
        const cardElement = elements.create('card');

        cardElement.mount('#card-element');
    </script>

[Stripeの`createPaymentMethod`メソッド](https://stripe.com/docs/stripe-js/reference#stripe-create-payment-method)を活用し、Stripeによりカードが検証し、安全な「支払い方法識別子」をSrtipeから取得します。

    const cardHolderName = document.getElementById('card-holder-name');
    const cardButton = document.getElementById('card-button');

    cardButton.addEventListener('click', async (e) => {
        const { paymentMethod, error } = await stripe.createPaymentMethod(
            'card', cardElement, {
                billing_details: { name: cardHolderName.value }
            }
        );

        if (error) {
            // ユーザーに"error.message"を表示する…
        } else {
            // カードの検証に成功した…
        }
    });

カードの検証が成功すれば、`paymentMethod.id`をLaravelアプリケーションに渡し、[１回限りの支払い](#simple-charge)を処理できます。

<a name="retrieving-payment-methods"></a>
### 支払い方法の取得

Billableモデルインスタンスの`paymentMethods`メソッドは、`Laravel\Cashier\PaymentMethod`インスタンスのコレクションを返します。

    $paymentMethods = $user->paymentMethods();

デフォルト支払いメソッドを取得する場合は、`defaultPaymentMethod`メソッドを使用してください。

    $paymentMethod = $user->defaultPaymentMethod();

<a name="check-for-a-payment-method"></a>
### ユーザーが支払い方法を持っているかの判定

Billableモデルが自身のアカウントに付加されている支払いメソッドを持っているかを判定するには、`hasPaymentMethod`メソッドを使用します。

    if ($user->hasPaymentMethod()) {
        //
    }

<a name="updating-the-default-payment-method"></a>
### デフォルト支払い方法の更新

`updateDefaultPaymentMethod`メソッドは顧客のデフォルト支払い方法の情報を更新するために使用します。このメソッドはStripe支払いメソッド識別子を引数に取り、その新しい支払い方法がデフォルト支払い方法として設定されます。

    $user->updateDefaultPaymentMethod($paymentMethod);

その顧客のデフォルト支払い方法情報をStripeの情報と同期したい場合は、`updateDefaultPaymentMethodFromStripe`メソッドを使用してください。

    $user->updateDefaultPaymentMethodFromStripe();

> {note} 顧客のデフォルト支払い方法は、インボイス発行処理と新しいサブスクリプションの生成にだけ使用されます。Stripeの制限により、一回だけの課金には使用されません。

<a name="adding-payment-methods"></a>
### 支払い方法の追加

新しい支払い方法を追加するには、Billableのユーザーに対し、`addPaymentMethod`を呼び出します。支払いメソッド識別子を渡してください。

    $user->addPaymentMethod($paymentMethod);

> {tip} 支払い方法の識別子の取得方法を学ぶには、[支払い方法保持のドキュメント](#storing-payment-methods)を確認してください。

<a name="deleting-payment-methods"></a>
### 支払い方法の削除

支払い方法を削除するには、削除したい`Laravel\Cashier\PaymentMethod`インスタンス上の`delete`メソッドを呼び出します。

    $paymentMethod->delete();

`deletePaymentMethods`メソッドは、そのBillableモデルの全支払いメソッド情報を削除します。

    $user->deletePaymentMethods();

> {note} アクティブなサブスクリプションがあるユーザーでは、デフォルト支払いメソッドが削除されないようにする必要があるでしょう。

<a name="subscriptions"></a>
## サブスクリプション

<a name="creating-subscriptions"></a>
### サブスクリプション作成

サブスクリプションを作成するには最初にbillableなモデルのインスタンスを取得しますが、通常は`App\User`のインスタンスでしょう。モデルインスタンスが獲得できたら、モデルのサブスクリプションを作成するために、`newSubscription`メソッドを使います。

    $user = User::find(1);

    $user->newSubscription('main', 'premium')->create($paymentMethod);

`newSubscription`メソッドの最初の引数は、サブスクリプションの名前です。アプリケーションでサブスクリプションを一つしか取り扱わない場合、`main`か`primary`と名づけましょう。２つ目の引数はユーザーが購入しようとしているサブスクリプションのプランを指定します。この値はStripeのプラン識別子に対応させる必要があります。

`create`メソッドは[Stripeの支払い方法識別子](#storing-payment-methods)、もしくは`PaymentMethod`オブジェクトを引数に取り、サブスクリプションを開始するのと同時に、データベースの顧客IDと他の関連する支払い情報を更新します。

> {note} サブスクリプションの`create()`へ支払いメソッド識別子を直接渡すと、ユーザーの保存済み支払いメソッドへ自動的に追加します。

#### ユーザー詳細情報の指定

ユーザーに関する詳細情報を追加したい場合は、`create`メソッドの第２引数に渡すことができます。

    $user->newSubscription('main', 'monthly')->create($paymentMethod, [
        'email' => $email,
    ]);

Stripeがサポートしている追加のフィールドについてのさらなる情報は、Stripeの[顧客の作成](https://stripe.com/docs/api#create_customer)ドキュメントを確認してください。

#### クーポン

サブスクリプションの作成時に、クーポンを適用したい場合は、`withCoupon`メソッドを使用してください。

    $user->newSubscription('main', 'monthly')
         ->withCoupon('code')
         ->create($paymentMethod);

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

`subscribedToPlan`メソッドは、そのユーザーがStripeのプランIDで指定したプランを購入しているかを確認します。以下の例では、ユーザーの`main`サブスクリプションが、購入され有効な`monthly`プランであるかを確認しています。

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

<a name="incomplete-and-past-due-status"></a>
#### 不十分と期日超過の状態

サブスクリプション作成後、そのサブクリプションが２つ目の支払いアクションを要求している場合、`incomplete`（不十分）として印がつけられます。サブスクリプションの状態は、Cashierの`subscriptions`データベーステーブルの`stripe_status`カラムに保存されます。

同様に、サブスクリプションの変更時に第２の支払いアクションが要求されている場合は、`past_due`（期日超過）として印がつけられます。サブスクリプションが２つのどちらかである時、顧客が支払いを受領するまで状態は有効になりません。あるサブクリプションに不十分な支払いがあるかを確認する場合は、Billableモデルかサブクリプションインスタンス上の`hasIncompletePayment`メソッドを使用します。

    if ($user->hasIncompletePayment('main')) {
        //
    }

    if ($user->subscription('main')->hasIncompletePayment()) {
        //
    }

サブクリプションに不完全な支払いがある場合、`latestPayment`（最後の支払い）識別子を渡したCashierの支払い確認ページをそのユーザーへ表示すべきです。この識別子を取得するには、サブクリプションインスタンスの`latestPayment`メソッドが使用できます。

    <a href="{{ route('cashier.payment', $subscription->latestPayment()->id) }}">
        Please confirm your payment.
    </a>

> {note} あるサブクリプションに`incomplete`状態がある場合、支払いを確認するまでは変更できません。そのためサブクリプションが`incomplete`状態では、`swap` や`updateQuantity`メソッドは例外を投げます。

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

次の支払いサイクルまで待つ代わりに、プランを変更時即時にインボイスを発行したい場合は、`swapAndInvoice`メソッドを使用します。

    $user = App\User::find(1);

    $user->subscription('main')->swapAndInvoice('provider-plan-id');

<a name="subscription-quantity"></a>
### 購入数

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

    public function taxPercentage()
    {
        return 20;
    }

`taxPercentage`メソッドにより、モデルごとに税率を適用できるため、多くの州や国に渡るユーザーベースで税率を決める場合に便利です。

> {note} `taxPercentage`メソッドは、サブスクリプションの課金時のみに適用されます。Cashierで「一回のみ」の支払いを行う場合は、税率を自分で適用する必要があります。

#### 税率の同期

`taxPercentage`が返すハードコードした値を変更する場合、ユーザーに対する既存のサブスクリプションは以前のままになります。`taxPercentage`が返す値に既存のサブスクリプションも更新したい場合は、ユーザーのサブスクリプションインスタンスに対し、`syncTaxPercentage`メソッドを呼び出す必要があります。

    $user->subscription('main')->syncTaxPercentage();

<a name="subscription-anchor-date"></a>
### サブスクリプション課金日付け

デフォルトで課金日はサブスクリプションが生成された日付け、もしくは使用期間を使っている場合は、使用期間の終了日です。課金日付を変更したい場合は、`anchorBillingCycleOn`メソッドを使用します。

    use App\User;
    use Carbon\Carbon;

    $user = User::find(1);

    $anchor = Carbon::parse('first day of next month');

    $user->newSubscription('main', 'premium')
                ->anchorBillingCycleOn($anchor->startOfDay())
                ->create($paymentMethod);

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

<a name="with-payment-method-up-front"></a>
### 支払いの事前登録あり

顧客へ試用期間を提供し、支払情報を事前に登録してもらう場合、サブスクリプションを作成するときに`trialDays`メソッドを使ってください。

    $user = User::find(1);

    $user->newSubscription('main', 'monthly')
                ->trialDays(10)
                ->create($paymentMethod);

このメソッドはデータベースのサブスクリプションレコードへ、試用期間の終了日を設定すると同時に、Stripeへこの期日が過ぎるまで、顧客へ課金を始めないように指示します。`trialDays`メソッドを使用する場合、Stripeでそのプランに対して設定したデフォルトの試用期間はオーバーライドされます。

> {note} 顧客のサブスクリプションが試用期間の最後の日までにキャンセルされないと、期限が切れると同時に課金されます。そのため、ユーザーに試用期間の終了日を通知しておくべきでしょう。

`trialUntil`メソッドにより、使用期間の終了時を指定する、`DateTime`インスタンスを渡せます。

    use Carbon\Carbon;

    $user->newSubscription('main', 'monthly')
                ->trialUntil(Carbon::now()->addDays(10))
                ->create($paymentMethod);

ユーザーが使用期間中であるかを判定するには、ユーザーインスタンスに対し`onTrial`メソッドを使うか、サブスクリプションインスタンスに対して`onTrial`を使用してください。次の２つの例は、同じ目的を達します。

    if ($user->onTrial('main')) {
        //
    }

    if ($user->subscription('main')->onTrial()) {
        //
    }

<a name="without-payment-method-up-front"></a>
### 支払いの事前登録なし

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

    $user->newSubscription('main', 'monthly')->create($paymentMethod);

<a name="handling-stripe-webhooks"></a>
## StripeのWebフック処理

> {tip} ローカル環境でWebhooksのテストの手助けをするために、[Laravel Valet](/docs/{{version}}/valet)の`valet share`コマンドが使用できます。

StripeはWebフックにより、アプリケーションへ様々なイベントを通知できます。デフォルトで、CashierのWebhookを処理するルートのコントローラは、Cashierのサービスプロバイダで設定されています。このコントローラはWebhookの受信リクエストをすべて処理します。

デフォルトでこのコントローラは、課金に多く失敗し続ける（Stripeの設定で定義している回数）、顧客の更新、顧客の削除、サブスクリプションの変更、支払い方法の変更があると、自動的にサブスクリプションをキャンセル処理します。しかしながら、すぐに見つけることができるようにこのコントローラを拡張し、どんなWebhookイベントでもお好きに処理できます

アプリケーションでStripeのWebhookを処理するためには、StripeのコントロールパネルでWebhook URLを確実に設定してください。Stripeのコントロールパネルで設定する必要のあるWebhookの全リストは、以下のとおりです。

- `customer.subscription.updated`
- `customer.subscription.deleted`
- `customer.updated`
- `customer.deleted`
- `invoice.payment_action_required`

> {note} Cashierに含まれる、[Webフック署名の確認](/docs/{{version}}/billing#verifying-webhook-signatures)ミドルウェアを使用し、受信リクエストを確実に保護してください。

#### WebフックとCSRF保護

StripeのWebフックでは、Laravelの [CSRFバリデーション](/docs/{{version}}/csrf)をバイパスする必要があるため、`VerifyCsrfToken`ミドルウェアのURIを例外としてリストしておくか、ルート定義を`web`ミドルウェアグループのリストから外しておきましょう。

    protected $except = [
        'stripe/*',
    ];

<a name="defining-webhook-event-handlers"></a>
### Webフックハンドラの定義

Cashierは課金の失敗時に、サブスクリプションを自動的にキャンセル処理しますが、他のWebフックイベントを処理したい場合は、Webフックコントローラを拡張します。メソッド名はCashierが期待する命名規則に沿う必要があります。特にメソッドは`handle`のプレフィックスで始まり、処理したいStripeのWebフックの名前を「キャメルケース」にします。たとえば、`invoice.payment_succeeded` Webフックを処理する場合は、`handleInvoicePaymentSucceeded`メソッドをコントローラに追加します。

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
            // イベントの処理…
        }
    }

次に、`routes/web.php`の中で、キャッシャーコントローラへのルートを定義します。これにより、デフォルトのルートが上書きされます。

    Route::post(
        'stripe/webhook',
        '\App\Http\Controllers\WebhookController@handleWebhook'
    );

<a name="handling-failed-subscriptions"></a>
### サブスクリプション不可

顧客のクレジットカードが有効期限切れだったら？　心配いりません。CashierのWebhookコントローラが顧客のサブスクリプションをキャンセルします。失敗した支払いは自動的に捉えられ、コントローラにより処理されます。このコントローラはStripeがサブスクリプションに失敗したと判断した場合、顧客のサブスクリプションを取り消します。（通常、３回の課金失敗）

<a name="verifying-webhook-signatures"></a>
### Webフック署名の確認

Webフックを安全にするため、[StripeのWebフック著名](https://stripe.com/docs/webhooks/signatures)が利用できます。便利に利用できるように、Cashierは送信されてきたWebフックリクエストが有効なものか確認するミドルウェアをあらかじめ用意しています。

Webhookの確認を有効にするには、`.env`ファイル中の`STRIPE_WEBHOOK_SECRET`環境変数を確実に設定してください。Stripeアカウントのダッシュボードから取得される、Webhookの`secret`を指定します。

<a name="single-charges"></a>
## 一回だけの課金

<a name="simple-charge"></a>
### 課金のみ

> {note} `charge`メソッドには**アプリケーションで使用している通貨の最低単位**で金額を指定します。

サブスクリプションを購入している顧客の支払いメソッドに対して、「一回だけ」の課金を行いたい場合は、Billableモデルインスタンス上の`charge`メソッドを使用します。第２引数に[支払い方法識別子](#storing-payment-methods)を渡してください。

    // Stripeはセント単位で課金する
    $stripeCharge = $user->charge(100, $paymentMethod);

`charge`メソッドは第３引数に配列を受け付け、裏で動いているStripeの課金作成に対するオ
プションを指定できます。課金作成時に使用できるオプションについては、Stripeのドキュメントを参照してください。

    $user->charge(100, $paymentMethod, [
        'custom_option' => $value,
    ]);

課金に失敗すると、`charge`メソッドは例外を投げます。課金に成功すれば、メソッドは`Laravel\Cashier\Payment`のインスタンスを返します。

    try {
        $payment = $user->charge(100, $paymentMethod);
    } catch (Exception $e) {
        //
    }

<a name="charge-with-invoice"></a>
### インボイス付き課金

一回だけ課金をしつつ、顧客へ発行するPDFのレシートとしてインボイスも生成したいことがあります。`invoiceFor`メソッドは、まさにそのために存在しています。例として、「一回だけ」の料金を５ドル課金してみましょう。

    // Stripeはセント単位で課金する
    $user->invoiceFor('One Time Fee', 500);

金額は即時にユーザーのデフォルト支払い方法へ課金されます。`invoiceFor`メソッドは第３引数に配列を受け付けます。この配列はインボイスアイテムへの支払いオプションを含みます。第４引数も配列で、インボイス自身に対する支払いオプションを指定します。

    $user->invoiceFor('Stickers', 500, [
        'quantity' => 50,
    ], [
        'tax_percent' => 21,
    ]);

> {note} `invoiceFor`メソッドは、課金失敗時にリトライするStripeインボイスを生成します。リトライをしてほしくない場合は、最初に課金に失敗した時点で、Stripe APIを使用し、生成したインボイスを閉じる必要があります。

<a name="refunding-charges"></a>
### 払い戻し

Stripeでの課金を払い戻す必要がある場合は、`refund`メソッドを使用します。このメソッドの第１引数は、Stripe Payment Intent IDです。

    $payment = $user->charge(100, $paymentMethod);

    $user->refund($payment->id);

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
            'vendor' => 'Your Company',
            'product' => 'Your Product',
        ]);
    });

<a name="strong-customer-authentication"></a>
## 堅牢な顧客認証

皆さんのビジネスがヨーロッパを基盤とするものであるなら、堅牢な顧客認証 (SCA)規制を守る必要があります。これらのレギュレーションは支払い詐欺を防ぐためにEUにより２０１９年９月に課せられたものです。幸運なことに、StripeとCashierはSCA準拠のアプリケーション構築のために準備をしてきました。

> {note} 始める前に、[StripeのPSD2とSCAのガイド](https://stripe.com/en-be/guides/strong-customer-authentication)と、[新SCA APIのドキュメント](https://stripe.com/docs/strong-customer-authentication)を確認してください。

<a name="payments-requiring-additional-confirmation"></a>
### 支払い要求の追加確認

SCA規制は支払いの確認と処理を行うため、頻繁に追加の検証を要求しています。これが起きるとCashierは`IncompletePayment`例外を投げ、この追加の検証が必要であるとあなたに知らせます。この例外を捉えたら、処理の方法は２つあります。

最初の方法は、その顧客をCashierに含まれている支払い確認専門ページへリダイレクトする方法です。このページに紐つけたルートは、Cashierのサービスプロバイダで登録済みです。そのため、`IncompletePayment`例外を捉えたら、支払い確認ページへリダイレクトします。

    use Laravel\Cashier\Exceptions\IncompletePayment;

    try {
        $subscription = $user->newSubscription('default', $planId)
                                ->create($paymentMethod);
    } catch (IncompletePayment $exception) {
        return redirect()->route(
            'cashier.payment',
            [$exception->payment->id, 'redirect' => route('home')]
        );
    }

支払い確認ページで顧客はクレジットカード情報の入力を再度促され、「３Dセキュア」のような追加のアクションがStripeにより実行されます。支払いが確認されたら、上記のように`redirect`引数で指定されたURLへユーザーはリダイレクトされます。

別の方法として、Stripeに支払いの処理を任せることもできます。この場合、支払い確認ページへリダイレクトする代わりに、Stripeダッシュボードで[Stripeの自動支払いメール](https://dashboard.stripe.com/account/billing/automatic)を瀬一定する必要があります。しかしながら、`IncompletePayment`例外を捉えたら、支払い確認方法の詳細がメールで送られることをユーザーへ知らせる必要があります。

不完全な支払いの例外は、`Billable`のユーザーに対する`charge`、`invoiceFor`、`invoice`メソッドで投げられる可能性があります。スクリプションが処理される時、`SubscriptionBuilder`の`create`メソッドと、`Susbcription`モデルの`incrementAndInvoice`、`swapAndInvoice`メソッドは、例外を投げるでしょう。

#### 不十分と期日超過の状態

支払いが追加の確認を必要とする場合そのサブクリプションは、`stripe_status`データベースカラムにより表される`incomplete`か`past_due`状態になります。Cashierは支払いの確認が完了するとすぐに、Webhookによりその顧客のサブスクリプションが自動的に有効にします。

`incomplete`と`past_due`状態の詳細は、[追加のドキュメント](#incomplete-and-past-due-status)を参照してください。

<a name="off-session-payment-notifications"></a>
### 非セッション確立時の支払い通知

SCA規制は、サブスクリプションが有効なときにも、時々支払いの詳細を確認することを顧客に求めています。Cashierではセッションが確立していない時に支払いの確認が要求された場合に、顧客へ支払いの通知を送ることができます。例えば、サブスクリプションを更新する時にこれが起きます。Cashierの支払い通知は`CASHIER_PAYMENT_NOTIFICATION`環境変数へ通知クラスをセットすることで有効になります。デフォルトでは、この通知は無効です。もちろん、Cashierにはこの目的に使うための通知クラスが含まれていますが、必要であれば自作の通知クラスを自由に指定できます。

    CASHIER_PAYMENT_NOTIFICATION=Laravel\Cashier\Notifications\ConfirmPayment

非セッション時の支払い確認通知が確実に届くように、[StripeのWebhookが設定されており](#handling-stripe-webhooks)、Stripeのダッシュボードで`invoice.payment_action_required` Webhookが有効になっていることを確認してください。さらに、`Billable`モデルがLaravelの`Illuminate\Notifications\Notifiable`トレイトを使用していることも確認してください。

> {note} 定期課金でなく、顧客が自分で支払った場合でも追加の確認が要求された場合は、その顧客に通知が送られます。残念ながら、Stripeはその支払いが手動や「非セッション時」であることを知る方法がありません。しかし、顧客は支払いを確認した後に支払いページを閲覧したら、「支払いが完了しました」メッセージを確認できます。その顧客は同じ支払いを２度行い、二重に課金されるアクシデントに陥ることを防ぐことができるでしょう。
