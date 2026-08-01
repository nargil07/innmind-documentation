# HTTP

This HTTP client uses the immutable objects describing the protocol from the [`innmind/http` package](https://github.com/Innmind/Http).

## Usage

```php
use Innmind\HttpTransport\Success;
use Innmind\Http\{
    Request,
    Method,
    ProtocolVersion,
};
use Innmind\Url\Url;

$http = $os->remote()->http();

$request = Request::of(
    Url::of('https://github.com/'),
    Method::get,
    ProtocolVersion::v11,
);
$http($request)->match(
    static fn(Success $success) => var_dump(
        $success
            ->response()
            ->body()
            ->toString(),
    ),
    static fn(object $error) => throw new \RuntimeException(),
);
```

When sending an HTTP request it will return an `Either<Failure|ConnectionFailed|MalformedResponse|Information|Redirection|ClientError|ServerError, Success>`, where each of this classes are located in the `Innmind\HttpTransport\` namespace. This type may be scarry at first but it allows you to use static analysis to deal with every possible situation (or not by throwing an exception like in the example). No more surprises of uncaught exceptions in production!

??? info
    Responses are wrapped in classes such as `Success`, `Redirection`, etc... to avoid confusion as a response can be on both sides of the `Either`. This way you know for sure a `2XX` response is on the right side and the other ones on the left one.

You can specify headers on your requests like this:

```php
use Innmind\Http\{
    Headers,
    Header,
    Header\Value,
};

$request = Request::of(
    Url::of('https://github.com/'),
    Method::get,
    ProtocolVersion::v11,
    Headers::of(
        Header::of('User-Agent', Value::of('your custom user agent string')),
    ),
);
```

??? tip
    `innmind/http` comes with a lot of header classes to simplify some common cases.

You can always specify a body like so:

```php
use Innmind\Filesystem\File\Content;
use Innmind\Http\Header\ContentType;
use Innmind\MediaType\{
    MediaType,
    TopLevel,
};
use Innmind\Json\Json;

$request = Request::of(
    Url::of('https://your-service.com/api/some-endpoint'),
    Method::post,
    ProtocolVersion::v11,
    Headers::of(
        ContentType::of(MediaType::from(TopLevel::application, 'json')),
    ),
    Content::ofString(Json::encode(['some' => 'payload'])),
);
```

Here we send some json but you can send anything you want.

The body of a `Request`, and a `Response`, is expressed via the `Content` class from the filesystem abstraction. This means that it can contain any valid file content.

You'll learn more on this `Content` in the [next chapter](filesystem.md).

## Following redirections

By default the client returned by `$os->remote()->http()` doesn't follow redirections. In order to do so you need to decorate the client like this:

```php
use Innmind\HttpTransport\Transport;

$http = Transport::followRedirections($os->remote()->http());
```

This decorator will follow to up to `5` redirections.

??? info
    Redirections are handled this way so you can compose all the decorators the way you need. For example you want to [apply exponential backoff](#retry-with-exponential-backoff) between each redirection.

## Resiliency

We tend to think networks are always stable or services as always up, but at some point failures **will** happen. This abstraction comes with 2 strategies to deal with them.

### Circuit breaker

If you need to call a service a lot but at some point becomes unavailable (for maintenance for example), you don't want to continue to try to call this service for a certain amount of time.

The [circuit breaker](https://en.wikipedia.org/wiki/Circuit_breaker_design_pattern) is a pattern that will automatically return an error response (without doing the actual call) if the service failed in the previous `x` amount of time.

You apply this pattern via this decorator:

```php
use Innmind\HttpTransport\Transport;
use Innmind\Time\Period;

$http = Transport::circuitBreaker(
    $os->remote()->http(),
    $os->clock(),
    Period::second(10),
);
```

!!! note ""
    The circuit breaks on a per domain name logic.

??? info
    In case a circuit is open then the error response will be a `503 Service Unavailable` with a custom header `X-Circuit-Opened` so you can understand who responded.

### Retry with exponential backoff

When a call fail you can automatically retry the call after a certain amount of time. You can apply the retries like this:

```php
use Innmind\OperatingSystem\Config;
use Innmind\HttpTransport\Transport;

$http = $os 
    ->map(static fn(Config $config) => $config->mapHttpTransport(
        static fn($transport) => Transport::exponentialBackoff(
            $transport,
            $config->halt(),
        ),
    ))
    ->remote()
    ->http();
```

This will retry all errors `5XX` responses and connection failures at most 5 times and will wait `100ms`, `271ms`, `738ms`, `2s` and `5.4s` between each retry.

??? tip
    You can improve the resiliency of the whole operating system abstractions like this:

    ```php
    use Innmind\OperatingSystem\Config\Resilient;

    $os = $os->map(Resilient::new());
    ```

    Even though for now it only applies this strategy to the HTTP client, you future prood yourself by using this config.

## Traps

### Unsent requests

The HTTP client doesn't send the request when you call `$http($request)` to allow for [concurrent calls](../concurrency/http.md). The call is actually done when you call `->match()` on the returned `Either`.

This means that this code will not send the request:

```php
$http = $os->remote()->http();
$http(Request::of(
    Url::of('https://your-service.com/api/some-resource'),
    Method::delete,
    ProtocolVersion::v11,
));
```

Even if you don't care about the response you need to do this:

```php hl_lines="6-9"
$http = $os->remote()->http();
$http(Request::of(
    Url::of('https://your-service.com/api/some-resource'),
    Method::delete,
    ProtocolVersion::v11,
))->match(
    static fn() => null,
    static fn() => null,
);
```

### Streaming

The default client uses `cURL` under the hood and the way it is structured prevents the streaming of requests/responses.

??? info
    However the work of the [distributed abstraction](../concurrency/distributed.md) will require the default client to switch to an implementation based on sockets that will open the door to streaming.
