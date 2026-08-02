# ORM

This ORM focuses on simplifying data manipulation.

!!! success ""
    It can handle any amount of data by being memory safe and reduces the complexity of data lifecycle by using immutable objects.

??? info
    Its monadic design allows it to be compatible with [Innmind's asynchronous context](../concurrency/async.md).

## Installation

```sh
composer require formal/orm:~6.0
```

## Example

```php
use Formal\ORM\{
    Manager,
    Sort,
};
use Innmind\Url\Url;

$manager = Manager::sql(
    $os
        ->remote()
        ->sql(Url::of('mysql://user:pwd@host:3306/database?charset=utf8mb4'))
        ->unwrap(),
);
$_ = $manager
    ->repository(YourAggregate::class)
    ->all()
    ->sort('someProperty', Sort::asc)
    ->drop(150)
    ->take(50)
    ->foreach(static fn(YourAggregate $aggregate) => doStuff($aggregate));
```

## Tips

Since it focuses on usage and not _abstracting a persistence model_ this ORM allows 3 different persistence models:

- SQL
- Filesystem
- Elasticsearch

## Advanced usage

Full documentation available [here](https://formal-php.org/orm/).

*[ORM]: Object Relational Mapping
