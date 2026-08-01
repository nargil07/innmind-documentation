# BlackBox

This is Innmind's own testing framework.

It follows [Innmind's philosophy](../philosophy/index.md) meaning it can be integrated in other tools. It is self contained and do not rely on global state.

## Installation

```sh
composer require --dev innmind/black-box '~7.0'
```

## Setup

```php title="blackbox.php"
<?php
declare(strict_types = 1);

require 'vendor/autoload.php';

use Innmind\BlackBox\{
    Application,
    Prove,
};

Application::new($argv) #(1)
    ->tryToProve(function(Prove $prove) {
        yield $prove->test(
            'More on tests in the next chapter',
            static fn($assert) => $assert->true(true),
        );
    })
    ->exit();
```

1. More on the usage of `$argv` [below](#tags).

This is the simplest setup of BlackBox. A PHP file (1) that bootstraps an `Application` to which is passed a function that will return a generator of tests and then exits.
{.annotate}

1. In this case the file is named `blackbox.php` but you can call it the way you want.

And you simply run your tests via `php blackbox.php`.

## Organization

In the example above the tests are provided inside an inline generator. This is fine when you only have a few of them. When it's no longer convenient you should split your tests in multiple files.

=== "File 1"
    ```php title="proofs/file1.php"
    <?php

    return static function(Prove $prove) {
        yield $prove->test(
            'Test 1',
            static fn($assert) => $assert->true(true),
        );
    }
    ```

=== "File 2"
    ```php title="proofs/file2.php"
    <?php

    return static function(Prove $prove) {
        yield $prove->test(
            'Test 2',
            static fn($assert) => $assert->true(true),
        );
    }
    ```

If you want to load these 2 files you can do:

```php title="blackbox.php" hl_lines="10-11"
<?php
declare(strict_types = 1);

require 'vendor/autoload.php';

use Innmind\BlackBox\Application;

Application::new($argv)
    ->tryToProve(function(Prove $prove) {
        yield from (require 'proofs/file1.php')($prove);
        yield from (require 'proofs/file2.php')($prove);
    })
    ->exit();
```

This is good because you can control the way your files are loaded. But adding new files becomes tedious, especially when multiple persons work on the project.

Instead you can simplify it with:

```php title="blackbox.php" hl_lines="8 12"
<?php
declare(strict_types = 1);

require 'vendor/autoload.php';

use Innmind\BlackBox\{
    Application,
    Runner\Load,
};

Application::new($argv)
    ->tryToProve(Load::everythingIn('proofs/'))
    ->exit();
```

In the end you have full control over the order your tests are loaded.

## Tags

After a while you may end up with a lot of tests and running them all all the time can be time consuming. You can categorize your tests via tags.

You declare them this way:

```php
use Innmind\BlackBox\Tag;

yield $prove
    ->test( #(1)
        'Test name',
        static fn($assert) => $assert->true(true),
    )
    ->tag(Tag::positive, Tag::wip);
```

1. Refer to example above to know where to place a test.

Then to only run a test with a given tag: `php blackbox.php wip`.

The list of arguments you pass in the CLI command is passed to BlackBox via the code `#!php Application::new($argv)`. Each argument must correspond to the name of a case on the `Innmind\BlackBox\Tag` enum.
