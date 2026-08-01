# HTTP calls

Traditionnaly the HTTP requests in PHP programs are synchronous for the sake of simplicity as PHP is single threaded. But this is wasteful when multiple requests could be sent at the same time.

Innmind's HTTP client allows to move from synchronous calls to concurrent ones very easily.

## Example

Imagine you want to fetch 2 pages, this would be the synchronous code:

```php
use Innmind\HttpTransport\Success;
use Innmind\Http\{
    Request,
    Method,
    ProtocolVersion,
};
use Innmind\Url\Url;

$http = $os->remote()->http();
$github = $http(Request::of(
    Url::of('https://github.com'),
    Method::get,
    ProtocolVersion::v11,
));
$github->match(
    static fn(Success $success) => doStuff($success->response()),
    static fn() => failedToFetch(),
);
$gitlab = $http(Request::of(
    Url::of('https://gitlab.com'),
    Method::get,
    ProtocolVersion::v11,
));
$gitlab->match(
    static fn(Success $success) => doStuff($success->response()),
    static fn() => failedToFetch(),
);
```

Remember that the [value returned by `$http` calls](../operating-system/http.md) is an [`Either`](../handling-data/either.md). More precisely it uses a deferred `Either`, meaning that the value it represents will be evaluated when you try to extract the value via the `match` method.

This means that to make the calls concurrent you only need to move all the `match` calls after asking to make requests:

```php hl_lines="12-15"
$http = $os->remote()->http();
$github = $http(Request::of(
    Url::of('https://github.com'),
    Method::get,
    ProtocolVersion::v11,
));
$gitlab = $http(Request::of(
    Url::of('https://gitlab.com'),
    Method::get,
    ProtocolVersion::v11,
));
$github->match(
    static fn(Success $success) => doStuff($success->response()),
    static fn() => failedToFetch(),
);
$gitlab->match(
    static fn(Success $success) => doStuff($success->response()),
    static fn() => failedToFetch(),
);
```

This way the HTTP client will execute all the requests planned before the first `match` is called.

## Tips

### Unsent requests

Remember to always keep a reference to a returned `Either` before calling a `match` method otherwise the non referenced request won't be sent. While this may seem tedious this opens a feature that may be very useful in certain cases.

This way you can plan a bunch of requests and afterward based on some logic unplan some requests by de-referencing the `Either`s before calling a `match` method.

This allows better flexibility in the way you can decouple your logic.

### Max concurrency

By default the client will send all planned requests at once. But this can be problematic if you plan too many requests, the underlying `cURL` implementation may return some errors.

You can configure the max concurrency at the start of your program and leave your business logic as is. You can do it this way:

=== "Operating System"
    ```php
    $os = $os->map(
        static fn($config) => $config->mapHttpTransport(
            static fn($transport) => $transport->map(
              static fn($config) => $config->limitConcurrencyTo(20),
            ),
        ),
    );

    // rest of your script
    ```

=== "Framework"
    ```php
    use Innmind\Framework\{
        Main\Cli,
        Application,
    };
    use Innmind\OperatingSystem\Config;

    $config = Config::of();
    $config = $config->mapHttpTransport(
        static fn($transport) => $transport->map(
          static fn($config) => $config->limitConcurrencyTo(20),
        ),
    );

    new class($config) extends Cli {
        protected function configure(Application $app): Application
        {
            // configure your app here
            return $app;
        }
    }
    ```

    Here we use the [`Cli` entrypoint](../framework/cli.md) but it works the same way for the [`Http` ones](../framework/http.md).

=== "CLI app"
    ```php
    use Innmind\CLI\{
        Main,
        Environment,
    };
    use Innmind\OperatingSystem\{
        OperatingSystem,
        Config,
    };

    $config = Config::of();
    $config = $config->mapHttpTransport(
        static fn($transport) => $transport->map(
          static fn($config) => $config->limitConcurrencyTo(20),
        ),
    );

    new class($config) extends Main {
        protected function main(Environment $env, OperatingSystem $os): Environment
        {
            // your code here
            return $env;
        }
    };
    ```

    [Related chapter](../app/cli.md)

=== "HTTP app"
    ```php
    use Innmind\HttpServer\Main;
    use Innmind\Http\{
        ServerRequest,
        Response,
    };
    use Innmind\OperatingSystem\Config;

    $config = Config::of();
    $config = $config->mapHttpTransport(
        static fn($transport) => $transport->map(
          static fn($config) => $config->limitConcurrencyTo(20),
        ),
    );

    new class($config) extends Main {
        protected function main(ServerRequest $request): Response
        {
            // your code here
            return Response::of(
                Response\StatusCode::ok,
                $request->protocolVersion(),
            );
        }
    };
    ```

    [Related chapter](../app/http.md)

The examples here use a maximum of `20` but you should adapt it to the needs of your program.

### Limits

When calling a `match` method it will wait for all planned request to finish before giving you access to your request response.

This means that you can't react as soon as a response is accessible. Your program can still stay idle for some time.

If you need better reaction timing you should head to the [asynchronous chapter](async.md).
