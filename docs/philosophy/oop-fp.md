# OOP & FP

## OOP

Originally this paradigm was intended to represent the behaviour of _living cells_ (objects) that interact with each other via message passing. Each cell/object is supposed to be a closed unit of compute.

Innmind follows this principle by using closed (aka `final`) classes with no getter/setter method, each method express an action.

There are 2 main kind of objects: the ones representing data with their associated behaviours (aka methods) and the ones expressing actions (usually with only the `__invoke` method).

The _bigger idea_ of OOP mentioned by [Alan Kay](https://en.wikipedia.org/wiki/Alan_Kay) of message passing is fulfilled by the [Actor Model](../getting-started/concurrency/distributed.md#actor-model).

## FP

This is a more mathematical approach to building programs that has gained a lot of traction for the past few years.

Innmind mainly use the principles described below, but FP as a whole has a lot more.

### Immutability

You already use immutable data without even realising it. For example if you do `#!php $result = str_replace($search, $replace, $subject)`, the call of the function doesn't modify the values inside `$search`, `$replace` or `$subject` but returns a new value `$result`. This is in essence immutability, you only return new values.

This allows to have less things to think about when calling a method. You know for sure the data you pass in to a function won't change. And inside a function you know that manipulating the data passed in won't have side effects outside of your scope.

The immutability of data applies to primitive values but also to objects. A call to a method will return a new object, and the initial object it kept as is.

Innmind uses immutable data everywhere possible.

### Purity

This concept applies to functions; may it be anonymous functions, named functions or methods.

A function is considered pure if it has no side effects. A side effect may be altering state or doing I/O. This means that a function can't use a global variable, print something to the screen or read something from the filesystem.

In other words a pure function only interacts with the immutable arguments passed in and returns an immutable value.

Just like Immutability this allows to have less things to think about. By modifying/calling a pure function you know you won't break an unforeseen part of your program.

### Totality

A function is considered _total_ if it can return a value for any combination of arguments it accepts.

For example the function `divide(int, int): float` is not total because it will have to throw an exception for a division by `0`. On the other hand `divide(int, int): ?float` is total because it can return `null` in case of a division by `0`.

The advantage of using total functions is that a [static analysis tool](development.md#type-strictness) can automatically check all the combinations to make sure your program won't crash. It eliminates the need to write tests for the exceptions or the surprises of the runtime.

Innmind heavily relies on this design to reduce the mental load of making sure every unhappy path is covered.

??? info "Exceptions"
    Innmind also relies on `Exception`s for cases where the program can't be recovered. But since it can't be recovered you don't have to think about them, that's also why they're not documented.

    It somewhat follows the `Let it crash` approach of [Erlang](https://www.erlang.org).

### Composition

This allows to extend the behaviour of a function without knowing its implementation. And this is completely transparent for the caller of such functions.

An example is an HTTP client (1) that provides a base implementation to do calls via `curl` and the `logger`, `followRedirections` and `circuitBreaker` decorators. You can compose them any way you wish:
{.annotate}

1. such as [`innmind/http-transport`](https://github.com/Innmind/HttpTransport)

<div markdown>
- `logger(followRedirections(curl))` will only log the user calls and is unaware if the redirections are followed
- `followRedirections(logger(curl))` will log the user calls and every redirections
- `circuitBreaker(logger(curl))` will not log calls to a domain that has previously failed
- etc...
</div>

The big advantage is that you can compose them _locally_ depending on your needs (1).
{.annotate}

1. As opposed to the inheritance approach where you're limited by the statically defined combinations exposed.

Innmind heavily uses composition to adapt behaviours locally and allows you to compose the base implementations the way you wish.

??? info
    The interesting discovery after many years of using composition with Innmind is that the higher the abstractions the more possibilities it offers.

### _Type detonation_

This means computing a concrete value. But _detonating_ too early exposes some problems.

For example, the math operation of the square of the square root of a number should return the same number. But if the type is _detonated_ at each function `square(squareRoot(2))` won't return `2`(1).
{.annotate}

1. `sqrt(2)**2` will return `2.0000000000000004`

By not _detonating_ too early the abstractions can do some optimisations on your behalf without even realising it (1). This is done by returning intermediate representations of the operations that needs to be done.
{.annotate}

1. For example [`innmind/math`](https://github.com/Innmind/Math) uses objects to represents numbers that optimise the `square(squareRoot(2))` in order to compute `2`. Or [`innmind/http-transport`](https://github.com/Innmind/HttpTransport) that can [run requests concurrently](../getting-started/concurrency/http.md).

In essence this _lazyness_ allows Innmind to optimise some operations.

### Monads

Monads are the culmination of the designs described above and are the cornerstone of this ecosystem.

They're data structures classes with at least 2 methods: `map` and `flatMap`.

- `map` will return a new monad of the same class with the data contained in it modified via the function passed to `map`.
- `flatMap` is similar to `map` except that the function passed to it must return a monad.

Said like this, monads are very abstract concepts. So here's an example of the simplest monad, the `Identity`:

=== "Identity monad"
    ```php
    $value = Identity::of(1)
        ->map(fn(int $value) => $value + 1)
        ->flatMap(fn(int $value) => Identity::of($value * 2))
        ->unwrap();
    $value === 4; // true
    ```

=== "Plain old PHP"
    ```php
    $value = 1;
    $value = $value + 1;
    $value = $value * 2;
    $value === 4; // true
    ```

While this example may not seem to provide much value, these structures are extremly powerful as it will be shown in [Getting started](../getting-started/index.md).

??? tip "Type detonation"
    [Detonating](#type-detonation) a monad means calling a method that returns something else than a monad.

## False dichotomy

When choosing our tools we are usually presented 2 choices: Hype vs Boring technologies, OOP vs FP, etc... with the impression that we can only choose one.

Innmind uses both OOP and FP to take advantages from both worlds.

The easiness of combining data with associated methods and handling of mutable data (1) of OOP. The ease of mind of FP to only have to deal with the code in front of you (2).
{.annotate}

1. such as socket servers
2. There's little chance to break the code on the other side of a program.

*[OOP]: Object-Oriented Programming
*[FP]: Functional Programming
