# Serve a S3 file via an HTTP server

```sh
composer require innmind/s3 '~6.0'
```

```php
use Innmind\Framework\{
    Application,
    Main\Http,
    Http\Route,
};
use Innmind\DI\Service;
use Innmind\S3;
use Innmind\Http\{
    ServerRequest,
    Response,
    Response\StatusCode,
};
use Innmind\MediaType\MediaType;
use Innmind\Url\{
    Url,
    Path,
};
use Innmind\Immutable\Attempt;

enum Services implements Service
{
    case s3;
    case serve;
}

new class extends Http {
    protected function configure(Application $app): Application
    {
        return $app
            ->service(Services::s3, static fn($_, $os) => S3\Factory::of($os)->build(
                Url::of('https://acces_key:acces_secret@bucket-name.s3.region-name.scw.cloud/'),
                S3\Region::of('region-name'),
            ))
            ->service(Services::serve, static fn($get) => new class($get('s3')) {
                public function __construct(private S3\Bucket $s3){}

                public function __invoke(ServerRequest $request): Attempt
                {
                    return Attempt::result(
                        $this
                            ->s3
                            ->get(Path::of('some file.txt'))
                            ->match(
                                static fn($file) => Response::of(
                                    StatusCode::ok,
                                    $request->protocolVersion(),
                                    null,
                                    $file,
                                ),
                                static fn() => Response::of(
                                    StatusCode::notFound,
                                    $request->protocolVersion(),
                                ),
                            ),
                    );
                }
            })
            ->route(Route::get(
                '/',
                Services::serve,
            ));
    }
};
```

!!! tip
    Head to the [framework chapter](../getting-started/framework/index.md) to learn how to call this server.
