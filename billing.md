git 4deba2bfca6636d5cdcede3f2068eff3b59c15ce

---

# Laravel Cashier

- [Введение](#introduction)
- [Настройка](#configuration)
- [Подписка](#subscribing-to-a-plan)
- [Подписка без кредитки](#no-card-up-front)
- [Переключение между подписками](#swapping-subscriptions)
- [Количество подписки](#subscription-quantity)
- [Отмена подписки](#cancelling-a-subscription)
- [Возобновление подписки](#resuming-a-subscription)
- [Проверка статуса подписки](#checking-subscription-status)
- [Обработка ошибок платежей](#handling-failed-payments)
- [Handling Other Stripe Webhooks](#handling-other-stripe-webhooks)
- [Счета](#invoices)

<a name="introduction"></a>
## Введение

Laravel Cashier обеспечивает удобный и мощный интерфейс для [Stripe](https://stripe.com) subscription billing services. Он обрабатывает почти весь код оплаты подписок, который вы остерегались писать of the boilerplate subscription billing code you are dreading writing. В дополнение к базовым функциям управления подпиской, Cashier может обравабывать купоны, переводить на други планы подписок, "количество" подписки, отменять льготные периоды, и даже генерировать счета в виде PDF.

<a name="configuration"></a>
## Настройка

#### Composer

Для начала, добавльте пакет Cashier в ваш файл `composer.json`:

	"laravel/cashier": "~3.0"

#### Сервис-провайдер

Далее, зарегистрируйте `Laravel\Cashier\CashierServiceProvider` в вашем файле конфигураций `app`.

#### Миграции

Перед началом использования Cashier, мы должны добавить несколько столбцов в нашу БД. Не волнуйтесь, вы можете использовать artisan-команду `cashier:table` для создания миграции, чтобы добавить необходимую колонку. Например, чтобы добавить столбец в таблицу users используйте команду `php artisan cashier:table users`. После того, как миграция была создана, просто запустите выполните команду `migrate`.

#### Model Setup

Затем, добавьте трейт `Billable` и соответствующие даты в ваше определение модели:

	use Laravel\Cashier\Billable;
	use Laravel\Cashier\Contracts\Billable as BillableContract;

	class User extends Eloquent implements BillableContract {

		use Billable;

		protected $dates = ['trial_ends_at', 'subscription_ends_at'];

	}

#### Stripe Key

Наконец, установите ваш ключ Stripe в одном из ваших загрузочных файлов или сервис-провайдеров, каких как `AppServiceProvider`:

	User::setStripeKey('stripe-key');

<a name="subscribing-to-a-plan"></a>
## Подписка на план

Если у вас есть экземпляр модели, вы можете легко подписать пользователя на заданный план Stripe:

	$user = User::find(1);

	$user->subscription('monthly')->create($creditCardToken);

Если вы хотите применить купон при создании подписки, то можете использовать метод `withCoupon`:

	$user->subscription('monthly')
	     ->withCoupon('code')
	     ->create($creditCardToken);

Метод `subscription` автоматически создаст подписку Stripe, а тажке обновит вашу БД с ID клиентов Stripe и другой соответствующей информацией о счетах. Если ваш план поддерживает пробный период, сконфигурированный в Stripe, дата окончания триала будет также автоматически устанавливаться в записи пользователя.

Если ваш план имеет испытательный срок, который **не** настроен в Stripe, необходимо установить дату окончания вручную после подписки:

	$user->trial_ends_at = Carbon::now()->addDays(14);

	$user->save();

### Задание дополнительных параметров

Если вы желаете уточнить дополнительную информацию о клиентах, то можете сделать это путем передачи их в качестве второго аргумента в `create` методе:

	$user->subscription('monthly')->create($creditCardToken, [
		'email' => $email, 'description' => 'Наш первый покупатель'
	]);

Чтобы узнать больше о дополнительных полях, поддерживаемых Stripe-ом читайте [документацию по созданию клиента Stripe](https://stripe.com/docs/api#create_customer).

<a name="no-card-up-front"></a>
## Подписка без кредитки

Если ваше приложение предлагает бесплатный пробный период без каких-либо кредитных карт, установите свойство `cardUpFront` в вашей модели на `false`:

	protected $cardUpFront = false;

При создании учетной записи, не забудьте установить пробную дату окончания:

	$user->trial_ends_at = Carbon::now()->addDays(14);

	$user->save();

<a name="swapping-subscriptions"></a>
## Переход между подписками

Для перехода пользователя на новую подписку используйте метод `swap`:

	$user->subscription('premium')->swap();

If the user is on trial, the trial will be maintained as normal. Also, if a "quantity" exists for the subscription, that quantity will also be maintained.

<a name="subscription-quantity"></a>
## Subscription Quantity

Sometimes subscriptions are affected by "quantity". For example, your application might charge $10 per month per user on an account. To easily increment or decrement your subscription quantity, use the `increment` and `decrement` methods:

	$user = User::find(1);

	$user->subscription()->increment();

	// Add five to the subscription's current quantity...
	$user->subscription()->increment(5);

	$user->subscription->decrement();

	// Вычесть 5 из текущего количества подписки...
	$user->subscription()->decrement(5);

<a name="cancelling-a-subscription"></a>
## Отмена подписки

Отмена подписки словно прогулка по парку:

	$user->subscription()->cancel();

Если подписка отменена, Cashier автоматически установит значение столбца `subscription_ends_at` в вашей БД. Этот столбец используется для того, чтобы знать когда метод `subscribed` должен вернуть `false`. Например, если пользователь отменил подписку 1-го Марта, но подписка заканчивается 5-го Марта, метод `subscribed` будет продолжать возвращать `true` до 5-го Марта.

<a name="resuming-a-subscription"></a>
## Возобновление подписки

Если пользователь отменил свою подписку, и вы желаете возобновить его, используйте `resume` метод:

	$user->subscription('monthly')->resume($creditCardToken);

Если пользователь отменил подписку, а затем возобновил ее до того как она полностью истечет, средства не будут списаны сразу. Их подписка будет просто возобновлена и они будут платить по старому циклу.

<a name="checking-subscription-status"></a>
## Проверка статуса подписки

Чтобы убедиться, что пользователь подписался на ваше приложение, используйте метод `subscribed`:

	if ($user->subscribed())
	{
		//
	}

Метод `subscribed` method makes a great candidate for a [route middleware](/docs/5.0/middleware):

	public function handle($request, Closure $next)
	{
		if ($request->user() && ! $request->user()->subscribed())
		{
			return redirect('billing');
		}

		return $next($request);
	}

Вы также можете определить, находится ли пользователь в пределах испытательного срока (если применимо) с помощью метода `onTrial`:

	if ($user->onTrial())
	{
		//
	}

Чтобы определить, если пользователь когда-то имел активную подписку, но отменил ее, вы можете использовать метод `cancelled`:

	if ($user->cancelled())
	{
		//
	}

You may also determine if a user has cancelled their subscription, but are still on their "grace period" until the subscription fully expires. For example, if a user cancels a subscription on March 5th that was scheduled to end on March 10th, the user is on their "grace period" until March 10th. Note that the `subscribed` method still returns `true` during this time.

	if ($user->onGracePeriod())
	{
		//
	}

The `everSubscribed` method may be used to determine if the user has ever subscribed to a plan in your application:

	if ($user->everSubscribed())
	{
		//
	}

Метод `onPlan` способ может быть использован, чтобы определить, является ли пользователь подписан на заданный план на основе его ID:

	if ($user->onPlan('monthly'))
	{
		//
	}

<a name="handling-failed-payments"></a>
## Handling Failed Payments

What if a customer's credit card expires? No worries - Cashier includes a Webhook controller that can easily cancel the customer's subscription for you. Just point a route to the controller:

	Route::post('stripe/webhook', 'Laravel\Cashier\WebhookController@handleWebhook');

That's it! Failed payments will be captured and handled by the controller. The controller will cancel the customer's subscription after three failed payment attempts. The `stripe/webhook` URI in this example is just for example. You will need to configure the URI in your Stripe settings.

<a name="handling-other-stripe-webhooks"></a>
## Handling Other Stripe Webhooks

If you have additional Stripe webhook events you would like to handle, simply extend the Webhook controller. Your method names should correspond to Cashier's expected convention, specifically, methods should be prefixed with `handle` and the name of the Stripe webhook you wish to handle. For example, if you wish to handle the `invoice.payment_succeeded` webhook, you should add a `handleInvoicePaymentSucceeded` method to the controller.

	class WebhookController extends Laravel\Cashier\WebhookController {

		public function handleInvoicePaymentSucceeded($payload)
		{
			// Обработка события
		}

	}

> **Примечание:** В добавок к обновлению информации о подписке в вашей БД, контроллер Webhook также отменит подписку через Stripe API.

<a name="invoices"></a>
## Счета

Вы можете легко получить массив счетов пользователя с помощью метода `invoices`:

	$invoices = $user->invoices();

При перечислении счетов для клиента, вы можете использовать эти вспомогательные методы для отображения соответствующей информации о счете:

	{{ $invoice->id }}

	{{ $invoice->dateString() }}

	{{ $invoice->dollars() }}

Используйте метод `downloadInvoice` для генерации счета в виде PDF. Да, это действительно так просто:

	return $user->downloadInvoice($invoice->id, [
		'vendor'  => 'Ваша компания',
		'product' => 'Ваш продукт',
	]);
