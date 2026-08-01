# SQL

The SQL client is structured by 2 concepts: immutable `Query` objects as input and [`Sequence`s](../handling-data/sequence.md) of immutable `Row`s as output.

## Usage

```php
use Formal\AccessLayer\{
    Query\SQL,
    Row,
};
use Innmind\Url\Url;

$sql = $os
    ->remote()
    ->sql(Url::of('mysql://user:password@127.0.0.1:3306/database_name'))
    ->unwrap();

$sql(SQL::of('SELECT * FROM users'))->foreach(
    static fn(Row $row) => var_dump($row->toArray()),
);
```

## Prepared queries

If you need to inject data in your queries you should use parameters.

!!! warning ""
    Do **not** use string concatenation as it can lead to [SQL injection](https://en.wikipedia.org/wiki/SQL_injection).

```php
use Formal\AccessLayer\Query\Parameter;

$query = SQL::of('INSERT INTO users VALUES (:id, :username)')
    ->with(Parameter::named('id', 'some-id'))
    ->with(Parameter::named('username', 'some-username'));

$sql($query);
```

Here named parameters are used via the format `:parameter_name` with the named specified again in `#!php Paramater::named()`.

You can also bind parameters by indices via the format `?` and then `#!php Parameter::of('value')`. This way you don't duplicate strings, but the order you add the parameters via the `with` method matters.

## Query builder

The `SQL` class allows you to specify the exact query you want to execute. But if you want to generate queries programmatically you should use the other classes from the `Formal\AccessLayer\Query\` namespace, such as `Select` or `Insert`.

=== "Select"
    ```php
    use Formal\AccessLayer\{
        Query\Select,
        Table\Name,
        Table\Column,
    };

    $select = Select::from(Name::of('users'))->columns( #(1)
        Column\Name::of('id'),
        Column\Name::of('username'),
    );

    $sql($select);
    ```

    1. If you don't specify the columns it will retrieve them all by default.

=== "Insert"
    ```php
    use Formal\AccessLayer\{
        Query\Insert,
        Row,
    };

    $insert = Insert::into(
        Name::of('users'),
        Row::of([
            'id' => 'id-1',
            'username' => 'john',
        ]),
        Row::of([
            'id' => 'id-2',
            'username' => 'jane',
        ]),
        // etc...
    );

    $sql($insert);
    ```

=== "etc..."
    Other query builders include:

    - `CreateTable`
    - `DropTable`
    - `Delete`
    - `Update`

## Filtering

When selecting from a table you can restrict the rows by specifying them manually in the `SQL` class. But if you need to programmatically construct the _where_ clause you can use the [specification pattern](https://en.wikipedia.org/wiki/Specification_pattern) via the `Select` class.

For example let's say you want to retrieve all users whose username starts with `a`. The first step is to create a specification:

```php
use Innmind\Specification\{
    Comparator\Property,
    Sign,
};

final class Username
{
    /** @psalm-pure */
    public static function startsWith(string $value): Property
    {
        return Property::of(
            'username', #(1),
            Sign::startsWith,
            $value,
        );
    }
}
```

1. This is the column name.

And then you use it like this:

```php
$select = Select::from(Name::of('users'))
    ->where(Username::startsWith('a'));

$sql($select);
```

!!! success ""
    The big advantage of specifications is that you can easily compose them. For example if you want users starting with `a` or `b` you'd do `#!php Username::startsWith('a')->or(Username::startsWith('b'))`; and if you want all except these ones you can chain a `->not()` to negate the whole condition.

    Another advantage is that this composition forces you to think about precedence of your conditions to reduce the risk of implicit behaviours.

## Laziness

All the queries you've seen so far return [deferred `Sequence`s](../handling-data/sequence.md#deferred) meaning that the queries are executed immediately but the returned rows will be loaded (and kept) in memory when you use the returned sequence.

For most queries this is fine. But if you want to select a large amount of data that may not fit in memory you should use lazy queries.

To do so instead of using `#!php SQL::of()`/`#!php Select::from()` use `#!php Query::lazily()`/`#!php Select::lazily()`.

```php
$select = Select::lazily(Name::of('users'));

$sql($select)->foreach(
    static fn(Row $row) => doStuff($row),
);
```

With this even if the result contains a million rows there'll only be one at a time in memory.

??? info
    However this means that if you call `foreach` twice it will run the query twice. The returned rows may change between the 2 calls, if you need the results to be the same you can't use lazy queries!

## Transactions

To run queries inside a transaction you need to run the corresponding sql queries like this:

```php
use Formal\AccessLayer\Query\Transaction;

try {
    $sql(Transaction::start);

    // run your queries here

    $sql((Transaction::commit);
} catch (\Throwable $e) {
    $sql((Transaction::rollback);

    throw $e;
}
```
