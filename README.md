# Laravel Analytics Event Tracking

Laravel package to easily send events to [Google Analytics](https://analytics.google.com/)

## Installation

You can install the package via composer:

```bash
composer require protonemedia/laravel-analytics-event-tracking
```

## Configuration

Publish the config and view files:

```bash
php artisan vendor:publish --provider="ProtoneMedia\AnalyticsEventTracking\ServiceProvider"
```

Set your [Google Analytics Tracking ID](https://support.google.com/analytics/answer/1008080) in the `.env` file or in the `config/analytics-event-tracking.php` file.

```bash
GOOGLE_ANALYTICS_TRACKING_ID=UA-01234567-89
```

This package supports [Google Analytics 4](https://blog.google/products/marketingplatform/analytics/new_google_analytics/) as of version 1.2.1. Please republish the view file if you're upgrading to a new Google Analytics 4 property.

## Blade Directive

This package comes with a `@sendAnalyticsClientId` directive that sends the [Client ID](https://developers.google.com/analytics/devguides/collection/analyticsjs/field-reference#clientId) from the GA front-end to your Laravel backend and stores it in the session.

It uses the [Axios HTTP library](https://github.com/axios/axios) the make an asynchronous POST request. Axios was choosen because it is provided by default in Laravel in the `resources/js/bootstrap.js` file.

Add the directive somewhere after initializing/configuring GA. The POST request will only be made if the `Client ID` isn't stored yet or when it's refreshed.

```php
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-01234567-89"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'UA-01234567-89', { 'send_page_view': false });
  gtag('event', 'page_view', { 'event_callback': function() {
      @sendAnalyticsClientId
  }});
</script>
```

If you don't use Axios, you have to implement this call by yourself. By default the endpoint is `/gaid` but you can customize it in the configuration file. The request is handled by the `ProtoneMedia\AnalyticsEventTracking\Http\StoreClientIdInSession` class. Make sure to also send the [CSRF token](https://laravel.com/docs/7.x/csrf).

## Broadcast events to Google Analytics

Add the `ShouldBroadcastToAnalytics` interface to your event and you're ready! You don't have to manually bind any listeners.

``` php
<?php

namespace App\Events;

use App\Order;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;
use ProtoneMedia\AnalyticsEventTracking\ShouldBroadcastToAnalytics;

class OrderWasCreated implements ShouldBroadcastToAnalytics
{
    use Dispatchable, SerializesModels;

    public $order;

    public function __construct(Order $order)
    {
        $this->order = $order;
    }
}
```

## Handle framework and 3rd-party events

If you want to handle events where you can't add the `ShouldBroadcastToAnalytics` interface, you can manually register them in your `EventServiceProvider` using the `DispatchAnalyticsJob` listener.

```php
<?php

namespace App\Providers;

use Illuminate\Auth\Events\Registered;
use Illuminate\Auth\Listeners\SendEmailVerificationNotification;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;
use ProtoneMedia\AnalyticsEventTracking\Listeners\DispatchAnalyticsJob;

class EventServiceProvider extends ServiceProvider
{
    /**
     * The event listener mappings for the application.
     *
     * @var array
     */
    protected $listen = [
        Registered::class => [
            SendEmailVerificationNotification::class,
            DispatchAnalyticsJob::class,
        ],
    ];
}
```

## Customize the broadcast

There are two additional methods that lets you customize the call to Google Analytics.

With the `withAnalytics` method you can interact with the [underlying package](https://github.com/theiconic/php-ga-measurement-protocol) to set additional parameters. Take a look at the `TheIconic\Tracking\GoogleAnalytics\Analytics` class to see the available methods.

With the `broadcastAnalyticsActionAs` method you can customize the name of the [Event Action](https://developers.google.com/analytics/devguides/collection/analyticsjs/field-reference#eventAction). By default we use the class name with the class's namespace removed. This method gives you access to the underlying `Analytics` class as well.

``` php
<?php

namespace App\Events;

use App\Order;
use TheIconic\Tracking\GoogleAnalytics\Analytics;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;
use ProtoneMedia\AnalyticsEventTracking\ShouldBroadcastToAnalytics;

class OrderWasCreated implements ShouldBroadcastToAnalytics
{
    use Dispatchable, SerializesModels;

    public $order;

    public function __construct(Order $order)
    {
        $this->order = $order;
    }

    public function withAnalytics(Analytics $analytics)
    {
        $analytics->setEventValue($this->order->sum_in_cents / 100);
    }

    public function broadcastAnalyticsActionAs(Analytics $analytics)
    {
        return 'CustomEventAction';
    }
}
```

## Handling the Client ID outside a HTTP Request

You might want to track an event that occurs outside of a HTTP Request, for example in a queued job or while handling a 3rd-party callback/webhook. Let's continue with the `Order` example. When the `Order` is created, you could save the `Client ID` in the database.

```php
<?php

namespace App\Http\Controllers;

use App\Order;
use App\Http\Requests\CreateOrderRequest;
use ProtoneMedia\AnalyticsEventTracking\Http\ClientIdRepository;

class CreateOrderController
{
    public function __invoke(CreateOrderRequest $request, ClientIdRepository $clientId)
    {
        $attributes = $request->validated();

        $attributes['google_analytics_client_id'] = $clientId->get();

        return Order::create($attributes);
    }
}
```

When you receive a webhook from your payment provider and you dispatch an `OrderWasPaid` event, you can use the `withAnalytics` method in your event to reuse the `google_analytics_client_id`:

``` php
<?php

namespace App\Events;

use App\Order;
use TheIconic\Tracking\GoogleAnalytics\Analytics;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;
use ProtoneMedia\AnalyticsEventTracking\ShouldBroadcastToAnalytics;

class OrderWasPaid implements ShouldBroadcastToAnalytics
{
    use Dispatchable, SerializesModels;

    public $order;

    public function __construct(Order $order)
    {
        $this->order = $order;
    }

    public function withAnalytics(Analytics $analytics)
    {
        $analytics->setClientId($this->order->google_analytics_client_id);
    }
}
```

## Additional configuration

You can configure some additional settings in the `config/analytics-event-tracking.php` file:

* `use_ssl`: Use SSL to make calls to GA
* `is_enabled`: Set to `false` to prevent events from being sent to GA
* `anonymize_ip`: Anonymizes the last digits of the user's IP
* `send_user_id`: Send the ID of the authenticated user to GA
* `queue_name`: Specify a queue to perform the calls to GA
* `client_id_session_key`: The session key to store the Client ID
* `http_uri`: HTTP URI to post the Client ID to (from the Blade Directive)

### Testing

``` bash
composer test
```

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
