# Testing

## Guarantees

When it comes to testing your program a question arises: should you use the same kind of database than in production or use a faster implementation to speed up the test suite.

When using ORMs that let the SQL bleed through their APIs this question becomes tricky because you may not end up having the same behaviour between your tests and in production.

!!! success ""
    This ORM doesn't have such problem. It uses [Property Based Testing](../../testing/property-based-testing.md) to make sure all storage implementations behave the same way.

This means that you can safely use a faster storage for your tests and it will behave the same way as in production.

## Setup

### Filesystem

You should use an in memory filesystem for your tests as it's the fastest since it never writes to the actual filesystem. And since the data is isolated to the process, you could run your tests in parallel thus speeding up even more your test suite.

```php
use Formal\ORM\Manager;
use Innmind\Filesystem\Adapter;

$adapter = Adapter::inMemory();
$orm = Manager::filesystem($adapter);
```

Your aggregates will be kept in memory as long as there is a reference to `$adapter`. This means that if your test looks something like this it won't work:

```php
$orm = Manager::filesystem(Adapter::inMemory());

// do some work that creates aggregates

$orm = Manager::filesystem(Adapter::inMemory());

// run expectations on your aggregates
```

The second instanciation of `$orm` will free the first one from memory and your aggregates will disappear.

### Elasticsearch

In case you want test a concrete instance of Elasticsearch to replicate the exact behaviour as in production, you should change one line when creating the orm:

```php hl_lines="9"
use Formal\ORM\{
    Manager,
    Adapter\Elasticsearch,
    Adapter\Elasticsearch\Refresh,
};

$orm = Manager::of(
    Elasticsearch::of(
        Refresh::of(
            $os->remote()->http(),
        ),
    ),
);
```

This decorator will make sure that every modification to an index is applied immediately. The default behaviour of Elasticsearch is that it will put the modification in an internal queue and there's at least a 1 second delay before seeing the change in the index. This is fine in production but it's difficult to do some assertions in a test.

If you need to assert you can fetch an aggregate after persisting it, then this decorator is for you.
