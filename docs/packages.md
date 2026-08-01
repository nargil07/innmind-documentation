---
hide:
    - navigation
---

# Other Packages

## Access Control List

Immutable object version to define unix permissions.

```sh
composer require innmind/acl:~4.0
```

```php
use Innmind\ACL\{
    ACL,
    User,
    Group,
    Mode,
};

$acl = ACL::of('r---w---x user:group');

$acl->allows(User::of('foo'), Group::of('bar'), Mode::read); // false
$acl->allows(User::of('foo'), Group::of('bar'), Mode::execute); // true
```

[Repository](https://github.com/Innmind/ACL)

## Coding standard

Specifies the code style used throughout Innmind.

```sh
composer require --dev innmind/coding-standard:~2.2
```

```php title=".php-cs-fixer.dist.php"
<?php

return \Innmind\CodingStandard\CodingStandard::config(['src', 'proofs']);
```

```sh
vendor/bin/php-cs-fixer fix
```

[Repository](https://github.com/Innmind/coding-standard)

## Colour

Immutables objects to specify colours and switch between the notations (RGBA, HSL and CMYK).

```sh
composer install innmind/colour:~5.1
```

```php
use Innmind\Colour\Colour;

$rgba = Colour::of('#3399ff');
$hsla = Colour::of('hsl(210, 100%, 60%)');
$cmyka = Colour::of('device-cmyk(80%, 40%, 0%, 0%)');
$rgba = Colour::blue->toRGBA();
```

[Repository](https://github.com/Innmind/Colour)

## Cron

Immutable objects to define cron jobs and install them on a machine (local or remote).

```sh
composer require innmind/cron '~5.0'
```

```php
use Innmind\Cron\{
    Crontab,
    Job,
};
use Innmind\Server\Control\Server\Command;

$install = Crontab::forUser(
    'admin',
    Job::of(
        Job\Schedule::everyDayAt(10, 30),
        Command::foreground('say hello'),
    ),
);
$install($os->control())->unwrap();
```

[Repository](https://github.com/Innmind/Cron)

## Encoding

Allows to `tar` [directories](getting-started/operating-system/filesystem.md) and `gzip` [files](getting-started/operating-system/filesystem.md) in a memory safe way.

```php
use Innmind\Filesystem\{
    File,
    Name,
};
use Innmind\Url\Path;
use Innmind\Encoding\{
    Gzip,
    Tar,
};

$tar = $os
    ->filesystem()
    ->mount(Path::of('some/directory/'))
    ->unwrap()
    ->get(Name::of('data'))
    ->map(Tar::encode($os->clock()))
    ->map(Gzip::compress())
    ->match(
        static fn(File\Content $file) => $file,
        static fn() => null,
    );
```

[Repository](https://github.com/Innmind/encoding)

## Hash

Allows to compute the hash of [files](getting-started/operating-system/filesystem.md) in a memory safe way.

```php
use Innmind\Filesystem\Name;
use Innmind\Url\Path;
use Innmind\Hash\{
    Hash,
    Value,
};
use Innmind\Immutable\Set;

$hash = $os
    ->filesystem()
    ->mount(Path::of('some-folder/'))
    ->unwrap()
    ->get(Name::of('some-file'))
    ->map(Hash::sha512->ofFile(...))
    ->match(
        static fn(Value $hash): string => $hash->hex(),
        static fn() => null,
    );
```

[Repository](https://github.com/Innmind/hash)

## Html

Allows to parse HTML files to immutable objects (built on top of [XML](#xml)).

```php
use Innmind\Html\Reader;
use Innmind\HttpTransport\Success;
use Innmind\Http\{
    Request,
    Method,
    ProtocolVersion,
};
use Innmind\Url\Url;
use Innmind\Xml\Node;
use Innmind\Immutable\Attempt;

$read = Reader::new();

$html = $os
    ->remote()
    ->http()(Request::of(
        Url::of('https://github.com/'),
        Method::get,
        ProtocolVersion::v11,
    ))
    ->attempt(static fn() => new \RuntimeException)
    ->map(static fn(Success $success) => $success->response()->body())
    ->flatMap($read);
$html; // instance of Attempt<Node>
```

[Repository](https://github.com/Innmind/Html)

## HTTP Authentication

Simple structures to define the ways to extract the identity from a [`ServerRequest`](getting-started/app/http.md).

[Repository](https://github.com/Innmind/HttpAuthentication)

## HTTP Session

Object approach to handle HTTP sessions without a global state.

[Repository](https://github.com/Innmind/HttpSession)

## Log reader

Allows to read Apache access and Monolog logs into immutable objects in a memory safe way.

```sh
composer require innmind/log-reader '~6.0'
```

```php
use Innmind\LogReader\{
    Reader,
    LineParser\Monolog,
    Log,
};
use Innmind\Filesystem\{
    File,
    Name,
};
use Innmind\Url\Path;
use Innmind\Immutable\Predicate\Instance;
use Psr\Log\LogLevel;

$read = Reader::of(
    Monolog::of($os->clock()),
);
$os
    ->filesystem()
    ->mount(Path::of('var/logs/'))
    ->unwrap()
    ->get(Name::of('prod.log'))
    ->keep(Instance::of(File::class))
    ->map(static fn($file) => $file->content())
    ->toSequence()
    ->flatMap($read)
    ->filter(
        static fn(Log $log) => $log
            ->attribute('level')
            ->filter(static fn($level) => $level->value() === LogLevel::CRITICAL)
            ->match(
                static fn() => true,
                static fn() => false,
            ),
    )
    ->foreach(
        static fn($log) => $log
            ->attribute('message')
            ->match(
                static fn($attribute) => print($attribute->value()),
                static fn() => print('No message found'),
            ),
    );
```

[Repository](https://github.com/Innmind/LogReader)

## RabbitMQ management

Object API on top of the `rabbitmqadmin` CLI command.

[Repository](https://github.com/Innmind/RabbitMQManagement)

## Robots.txt

Allows to parse `robots.txt` files.

```sh
composer require innmind/robots-txt '~7.0'
```

```php
use Innmind\RobotsTxt\{
    Parser,
    RobotsTxt,
};
use Innmind\Url\Url;

$parse = Parser::of(
    $os->remote()->http(),
    'My user agent',
);
$robots = $parse(Url::of('https://github.com/robots.txt'))->match(
    static fn(RobotsTxt $robots) => $robots,
    static fn() => throw new \RuntimeException('robots.txt not found'),
);
$robots->disallows('My user agent', Url::of('/humans.txt')); // false
$robots->disallows('My user agent', Url::of('/any/other/url')); // true
```

[Repository](https://github.com/Innmind/Robots.txt)

## Validation

This is a monadic approach to data validation.

```php
use Innmind\Validation\{
    Is,
    Failure,
};
use Innmind\Immutable\Sequence;

$valid = [
    'id' => 42,
    'username' => 'jdoe',
    'addresses' => [
        'address 1',
        'address 2',
    ],
    'submit' => true,
];
$invalid = [
    'id' => '42',
    'addresses' => [
        'address 1',
        null,
    ],
    'submit' => true,
];

$validate = Is::shape('id', Is::int())
    ->with('username', Is::string())
    ->with(
        'addresses',
        Is::list(
            Is::string()->map(
                static fn(string $address) => new YourModel($address),
            )
        )
    );
$result = $validate($valid)->match(
    static fn(array $value) => $value,
    static fn() => throw new \RuntimeException('invalid data'),
);
// Here $result looks like:
// [
//      'id' => 42
//      'username' => 'jdoe',
//      'addresses' [
//          new YourModel('address 1'),
//          new YourModel('address 2'),
//      ],
//      (1)
// ]
$errors = $validate($invalid)->match(
    static fn() => null,
    static fn(Sequence $failures) => $failures
        ->map(static fn(Failure $failure) => [
            $failure->path()->toString(),
            $failure->message(),
        ])
        ->toList(),
);
// Here $errors looks like:
// [
//      ['id', 'Value is not of type int'],
//      ['$', 'The key username is missing'],
//      ['addresses', 'Value is not of type string']
// ]
```

1. See how the `submit` key disappeared.

[Repository](https://github.com/Innmind/validation)

## XML

Allows to parse XML files to immutable objects.

```php
use Innmind\Xml\{
    Reader,
    Node,
};
use Innmind\Filesystem\File\Content;
use Innmind\Immutable\Maybe;

$read = Reader::of();

$tree = $read(
    Content::ofString('<root><foo some="attribute"/></root>')
); // Maybe<Node>
```

[Repository](https://github.com/Innmind/XML)
