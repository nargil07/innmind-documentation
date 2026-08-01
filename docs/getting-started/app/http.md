# HTTP

This package allows to build simple HTTP applications by representing requests and responses via objects.

## Usage

```php title="index.php"
<?php
declare(strict_types = 1);

require 'path/to/composer/autoload.php';

use Innmind\HttpServer\Main;
use Innmind\Http\{
    ServerRequest,
    Response,
};
use Innmind\Filesystem\File\Content;

new class extends Main {
    protected function main(ServerRequest $request): Response
    {
        return Response::of(
            Response\StatusCode::ok,
            $request->protocolVersion(),
            null,
            Content::ofString('Hello world'),
        );
    }
}
```

If you run the PHP server in the directory of this file via `php -S localhost:8080` and run `curl http://localhost:8080/` it will print `Hello world`.

??? note
    You can expose this script via any HTTP server that supports PHP.

As you can see the response body is a file content, meaning it accepts any [file content](../operating-system/filesystem.md).

```php title="index.php"
use Innmind\OperatingSystem\OperatingSystem;
use Innmind\Filesystem\{
    File,
    Name,
};
use Innmind\Http\{
    Headers,
    Header\ContentType,
};
use Innmind\MediaType\{
    MediaType,
    TopLevel,
};
use Innmind\Url\Path;
use Innmind\Immutable\Predicate\Instance;

new class extends Main {
    private OperatingSystem $os;

    protected function preload(OperatingSystem $os): void
    {
        $this->os = $os;
    }

    protected function main(ServerRequest $request): Response
    {
        return Response::of(
            Response\StatusCode::ok,
            $request->protocolVersion(),
            Headers::of(
                ContentType::of(MediaType::from(TopLevel::image, 'png')),
            ),
            $this
                ->os
                ->filesystem()
                ->mount(Path::of('images/'))
                ->unwrap()
                ->get(Name::of('some-image.png'))
                ->keep(Instance::of(File::class))
                ->match(
                    static fn(File $file) => $file->content(),
                    static fn() => throw new \RuntimeException(),
                ),
        );
    }
}
```

This example will send back the image at `images/some-image.png`. If the image is not found then it will throw an exception.

??? note
    The `main` function will catch all thrown exceptions and will return an empty `500` response. This is done to make sure no stack trace is ever shown to a user.

    During development if you want to see the exception you can catch all exceptions yourself and use `filp/whoops` to render it. Or you can use [Innmind's framework](../framework/index.md) and its [profiler](../framework/profiler.md).

!!! info ""
    For a very simple app this is enough, you can even do some routing manually by analyzing `$request->url()`. For any more than that you should start looking at the [framework](../framework/index.md).
