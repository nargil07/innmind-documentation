# Sequence

A `Sequence` monad represents a succession of values. In plain old PHP this is an array where you don't specify any key.

In essence:
```php
use Innmind\Immutable\Sequence;

$values = ['foo', 'bar', 'baz'];
// becomes
$values = Sequence::of('foo', 'bar', 'baz');
```

Of course just holding to multiple values is not very useful in itself. You'll see below how to manipulate this list of values.

Each example will show how to use the `Sequence` and how to do the same in plain old PHP in a declarative and imperative style. So you can better grasp what's happening.

## Pipelining

When dealing with a list of values we tend to apply multiple logic to it in succession. The more steps to transform our values the more complex it becomes.

The `Sequence` helps better break down each step.

### Transformations

If we reuse the example with the strings and we want to uppercase the first letter of each value we would do:

=== "Innmind"
    ```php
    $values = Sequence::of('foo', 'bar', 'baz')
        ->map(static fn(string $string) => \ucfirst($string))
        ->toList();
    $values === ['Foo', 'Bar', 'Baz']; // returns true
    ```

=== "Declarative"
    ```php
    $values = \array_map(
        static fn(string $string) => \ucfirst($string),
        ['foo', 'bar', 'baz'],
    );
    $values === ['Foo', 'Bar', 'Baz']; // returns true
    ```

=== "Imperative"
    ```php
    $values = [];

    foreach (['foo', 'bar', 'baz'] as $string) {
        $values[] = \ucfirst($string);
    }

    $values === ['Foo', 'Bar', 'Baz']; // returns true
    ```

The `map` method returns a new object `Sequence` with all the values modified by the function passed as argument. And the original object returned by `Sequence::of()` is not altered, meaning you can reuse it safely to do other operations.

??? tip
    The notation `#!php static fn(string $string) => \ucfirst($string)` can be shortened to `#!php \ucfirst(...)`.

With `map` the new object will contain the same number of values as the initial object. But some times for each value you want to return multiple values and in the end have an `array` with only one dimension.

Let's take the example where each `string` represent a username and we want to retrieve their addresses:

=== "Innmind"
    ```php
    /**
     * @return Sequence<string>
     */
    function getAddresses(string $username): Sequence {
        // this is a fake implementation
        return Sequence::of(
            "$username address 1",
            "$username address 2",
            "$username address 3",
        );
    }

    $addresses = Sequence::of('foo', 'bar', 'baz')
        ->flatMap(static fn(string $username) => getAdresses($username))
        ->toList();
    $addresses === [
        'foo address 1',
        'foo address 2',
        'foo address 3',
        'bar address 1',
        'bar address 2',
        'bar address 3',
        'baz address 1',
        'baz address 2',
        'baz address 3',
    ]; // returns true
    ```

=== "Declarative"
    ```php
    /**
     * @return list<string>
     */
    function getAddresses(string $username): array {
        // this is a fake implementation
        return [
            "$username address 1",
            "$username address 2",
            "$username address 3",
        ];
    }

    $addressesPerUser = \array_map(
        static fn(string $username) => getAddresses($username),
        ['foo', 'bar', 'baz'],
    );
    $addresses = \array_merge(...$addressesPerUser);
    $addresses === [
        'foo address 1',
        'foo address 2',
        'foo address 3',
        'bar address 1',
        'bar address 2',
        'bar address 3',
        'baz address 1',
        'baz address 2',
        'baz address 3',
    ]; // returns true
    ```

=== "Imperative"
    ```php
    /**
     * @return list<string>
     */
    function getAddresses(string $username): array {
        // this is a fake implementation
        return [
            "$username address 1",
            "$username address 2",
            "$username address 3",
        ];
    }

    $addresses = [];

    foreach (['foo', 'bar', 'baz'] as $username) {
        foreach (getAddresses($username) as $address) {
            $addresses[] = $address;
        }
    }

    $addresses === [
        'foo address 1',
        'foo address 2',
        'foo address 3',
        'bar address 1',
        'bar address 2',
        'bar address 3',
        'baz address 1',
        'baz address 2',
        'baz address 3',
    ]; // returns true
    ```

Here you can see that `flatMap` is a combination of `map` that would return the type `Sequence<Sequence<string>>` and then flattens it to obtain a `Sequence<string>`, hence the name `flatMap`.

You can also already see that the `Sequence` is simpler to chain actions because there is no need to assign the intermediate values to a new variable. In plain old PHP you could also avoid the intermediate values by inlining the calls but you'll quickly end up with a code harder to read with a lot of indentation.

`map` and `flatMap` are the only 2 methods to apply a modification to a `Sequence`.

### Composition

Since you'll not always have all the values known when creating a `Sequence`, you need to know how to add new values.

=== "Innmind"
    ```php
    $values = Sequence::of('foo')
        ->add('bar')
        ->add('baz')
        ->toList();
    $values === ['foo', 'bar', 'baz']; // return true
    ```

=== "Declarative"
    ```php
    $values = \array_merge(
        ['foo'],
        ['bar'],
        ['baz'],
    );

    $values === ['foo', 'bar', 'baz']; // returns true
    ```

=== "Imperative"
    ```php
    $values = ['foo'];
    $values[] = 'bar';
    $values[] = 'baz';

    $values === ['foo', 'bar', 'baz']; // returns true
    ```

??? tip
    You may also come across the notation `#!php $values = Sequence::of('foo')('bar')('baz')` in the ecosystem. This is a more _math like_ notation to look like a matrix augmentation.

    You check the implementation of `Sequence::add()` you'll see that it is an alias to the `__invoke` method that allows this notation.

If instead of adding a single value to the list you need to add multiple ones you would do:

=== "Innmind"
    ```php
    $values = Sequence::of('foo', 'bar')
        ->append(Sequence::of('baz', 'other', 'string'))
        ->toList();
    $values === ['foo', 'bar', 'baz', 'other', 'string']; // returns true
    ```

=== "Declarative"
    ```php
    $values = \array_merge(
        ['foo', 'bar'],
        ['baz', 'other', 'string'],
    );
    $values === ['foo', 'bar', 'baz', 'other', 'string']; // returns true
    ```

=== "Imperative"
    ```php
    $values = ['foo', 'bar'];

    foreach (['baz', 'other', 'string'] as $string) {
        $values[] = $string;
    }

    $values === ['foo', 'bar', 'baz', 'other', 'string']; // returns true
    ```

### Filtering

Instead of adding values you may want to remove values from a list you're given to only keep the ones you really want.

For example let's you have a list of cities and you only want to keep the french ones:

=== "Innmind"
    ```php
    $cities = Sequence::of(
        'Paris, France',
        'New York, USA',
        'London, UK',
        'Lyon, France',
        'etc...',
    )
        ->filter(static fn(string $city) => \str_contains($city, 'France'))
        ->toList();
    $cities === ['Paris, France', 'Lyon, France']; // returns true
    ```

=== "Declarative"
    ```php
    $values = \array_filter(
        [
            'Paris, France',
            'New York, USA',
            'London, UK',
            'Lyon, France',
            'etc...',
        ],
        static fn(string $city) => \str_contains($city, 'France'),
    );
    $cities === ['Paris, France', 'Lyon, France']; // returns true
    ```

=== "Imperative"
    ```php
    $cities = [
        'Paris, France',
        'New York, USA',
        'London, UK',
        'Lyon, France',
        'etc...',
    ];
    $values = [];

    foreach ($cities as $city) {
        if (\str_contains($city, 'France')) {
            $values[] = $city;
        }
    }

    $cities === ['Paris, France', 'Lyon, France']; // returns true
    ```

??? tip
    And if instead you want all the cities outside of France you can replace `filter` by `exclude`.

The `filter` method is fine if you don't need the new `Sequence` type to change, here we go from `Sequence<string>` to `Sequence<string>`. But if you have a `Sequence<null|\SplFileObject>` and you want to remove the `null` values then `filter`, even though will do the job, will return a `Sequence<null|\SplFileObject>`. This is a limitation of [Psalm](../../philosophy/development.md#type-strictness).

To overcome this problem you should use the method `keep` that expect a `Predicate` as argument. Technically the implementation of the predicate will be the same as the function passed to `filter` but it has a mechanism that allows Psalm to understand what you intend to do.

For our example you'd use it like this:

```php
use Innmind\Immutable\Predicate\Instance;

$values = Sequence::of(null, new \SplFileObject('some file.txt'), /* etc */)
    ->keep(Instance::of(\SplFileObject::class));
$values; // Sequence<\SplFileObject>
```

### Pipeline

So far you've only seen how to do one action at a time. The simplicity of `Sequence` starts to shine when chaining multiple actions.

Let's try to retrieve all the visited cities for each username, keep the french ones and remove the country from the name.

=== "Innmind"
    ```php
    /**
     * @return Sequence<string>
     */
    function getCities(string $username): Sequence {
        // fake implementation
        return match ($username) {
            'foo' => Sequence::of('Paris, France', 'London, UK'),
            'bar' => Sequence::of('New York, USA', 'London, UK'),
            'baz' => Sequence::of('New York, USA', 'Lyon, France'),
            default => Sequence::of(),
        };
    }

    $cities = Sequence::of('foo', 'bar', 'baz')
        ->flatMap(static fn(string $username) => getCities($username))
        ->filter(static fn(string $city) => \str_contains($city, 'France'))
        ->map(static fn(string $city) => \substr($city, 0, -8))
        ->toList();
    $cities === ['Paris', 'Lyon'];
    ```

=== "Declarative"
    ```php
    /**
     * @return list<string>
     */
    function getCities(string $username): array {
        // fake implementation
        return match ($username) {
            'foo' => ['Paris, France', 'London, UK'],
            'bar' => ['New York, USA', 'London, UK'],
            'baz' => ['New York, USA', 'Lyon, France'],
            default => [],
        };
    }

    $citiesPerUser = \array_map(
        static fn(string $username) => getCities($username),
        ['foo', 'bar', 'baz'],
    );
    $cities = \array_merge(...$citiesPerUser);
    $cities = \array_filter(
        $cities,
        static fn(string $city) => \str_contains($city, 'France'),
    );
    $cities = \array_map(
        static fn(string $city) => \substr($city, 0, -8),
        $cities,
    );
    $cities === ['Paris', 'Lyon'];
    ```

=== "Imperative"
    ```php
    /**
     * @return list<string>
     */
    function getCities(string $username): array {
        // fake implementation
        return match ($username) {
            'foo' => ['Paris, France', 'London, UK'],
            'bar' => ['New York, USA', 'London, UK'],
            'baz' => ['New York, USA', 'Lyon, France'],
            default => [],
        };
    }

    $cities = [];

    foreach (['foo', 'bar', 'baz'] as $username) {
        foreach (getCities($username) as $city) {
            if (\str_contains($city, 'France')) {
                $city = \substr($city, 0, -8);

                $cities[] = $city;
            }
        }
    }

    $cities === ['Paris', 'Lyon'];
    ```

With the declarative and imperative approach you have to deal with either a lot of indentation or a lot of variables. With a `Sequence` you just keep chaining methods.

Another nice upside to `Sequence` is when you try to build a pipeline and want to see the different results if you switch some logic around. To achieve it you only need to move a method call up or down, while the other approaches you need to be aware of conflicting variables.

## Extracting data

At some point you'll need to extract the values contained in a `Sequence` (1). So far you've only seen `toList` that return all the values in an `array`.
{.annotate}

1. For persisting them to a database, sending them in an HTTP response, etc...

### Computing a value

=== "Innmind"
    ```php
    $sum = Sequence::of(1, 2, 3, 4)->reduce(
        0,
        static fn(int $carry, int $value) => $carry + $value,
    );
    $sum === 10; // returns true
    ```

=== "Declarative"
    ```php
    $sum = \array_reduce(
        [1, 2, 3, 4],
        static fn(int $carry, int $value) => $carry + $value,
        0,
    );
    $sum === 10; // returns true
    ```

=== "Imperative"
    ```php
    $sum = 0;

    foreach ([1, 2, 3, 4] as $value) {
        $sum += $value;
    }

    $sum === 10; // returns true
    ```

### Fetching a value at an index

=== "Innmind"
    ```php
    $values = Sequence::of(1, 2, 3, 4);
    $value1 = $values->get(1)->match(
        static fn(int $value) => $value,
        static fn() => null,
    );
    $value2 = $values->get(100)->match(
        static fn(int $value) => $value,
        static fn() => null,
    );
    $value1 === 2; // returns true
    $value2 === null; // returns true
    ```

=== "Imperative"
    ```php
    $values = [1, 2, 3, 4];

    if (\array_key_exists(1, $values)) {
        $value1 = $values[1];
    } else {
        $value1 = null;
    }

    if (\array_key_exists(100, $values)) {
        $value2 = $values[100];
    } else {
        $value2 = null;
    }

    $value1 === 2; // returns true
    $value2 === null; // returns true
    ```

??? info
    The imperative approach could be simplified via `#!php $values[$index] ?? null`, but then if the value at the index is itself `null` you can't differentiate if the index exists or not.

### And more

Above are a few examples of the way to extract data. You should look at all the methods available on the `Sequence` class to see if one fit your needs.

## Execution mode

The power of `Sequence` is that you can change the way its implementation behave depending on your needs, without rearchitecting your whole program. You'll usually switch the mode for performance reasons.

### In memory

This is the mode you've seen so far. When calling `Sequence::of()` you specify all the values and they're kept in memory.

### Deferred

Instead of specyfying the values you can use a `Generator` to populate the `Sequence`. Once a value is loaded it's kept in memory. The advantage is that you can loop over the same generator multiple times (1).
{.annotate}

1. Using a `Generator` directly requires to call again the function that created it. But this means you may not end up with the same values (especially if generating objects).

```php
$values = Sequence::of(1, 2, 3, 4);
// becomes
$values = Sequence::defer((static function() {
    yield 1;
    yield 2;
    yield 3;
    yield 4;
})());
```

The `Sequence` is then used exactly the same way as an in memory one.

!!! tip ""
    You should use this mode when loading values may be expensive and you're not sure all the values will be loaded. This way you save a bit of time and memory by not fetching the values you don't end up needing.

### Lazy

With this mode you build a `Sequence` by passing a function that returns a `Generator`. This function will be called each time you try to extract some data from the `Sequence`.

```php
$values = Sequence::lazy(static function() {
    $file = \fopen('some file.txt', 'r');

    while ($chunk = \fgets($file, 256)) {
        yield $chunk;
    }
});
```

The `Sequence` is then used exactly the same way as an in memory one.

!!! tip ""
    You should use this mode to handle an infinite list of values or a list of values that can't fit in memory (1).
    {.annotate}

    1. Such as reading a multi gigabyte file or reading from a socket.

??? info
    This is where lies the root of the power of Innmind. Being able to work with infinte volumes of data as if it were in memory.

### Tips

#### Lazyness

When using `::defer()` or `::lazy()` your code won't be called until you try to extract data (1) or call the `memoize` method.
{.annotate}

1. Any method that return something else than a monad (`Sequence`, `Set` or `Maybe`).

For example if you want to print all the lines from a file, this will do nothing:

```php
Sequence::lazy(static function() {
    $file = \fopen('some file.txt', 'r');

    while ($line = \fgets($file)) {
        yield $line;
    }
})->map(static function($line) {
    echo $line;
});
```

This does nothing because `map` returns a new lazy `Sequence` with a `null` value for each line. Instead you should do:

```php
Sequence::lazy(static function() {
    $file = \fopen('some file.txt', 'r');

    while ($line = \fgets($file)) {
        yield $line;
    }
})->foreach(static function($line) {
    echo $line;
});
```

`foreach` returns a `Innmind\Immutable\SideEffect` so the `Sequence` knows that it needs to call all the logic you specified.

#### Psalm

If you call the `foreach` method you won't be able to use the returned value as it's an object that does nothing. It's returned because `Sequence` is an immutable class, meaning all methods **must** return a value otherwise Psalm tells that the method is useless.

But you still need to assign the returned value to a variable `$_` (1) otherwise Psalm will tell you that the call to `foreach` does nothing.
{.annotate}

1. Called a sink. Psalm won't run any analysis on this variable because it starts with an underscore.

## In the ecosystem

You'll find this class used pretty much everywhere in this ecosystem at it allows to describe:

- a list of values
- a file as a lazy list of chunks
- a file as a lazy list of lines
- a directory as lazy list of files
- a socket as a lazy list of frames
- a SQL result as a lazy list of rows
- a process output as a lazy list of chunks
- and more...
