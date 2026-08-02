---
authors: [baptouuuu]
date: 2026-08-02
---

# Hardening API consistency

For the past few years most of the work on this ecosystem has been towards building an Actor System. You can find the project at [`innmind/actors`](https://github.com/Innmind/actors/pull/2) and gave a presentation of it at [Forum PHP 2025](https://www.youtube.com/watch?v=E3pmkKQNC7Y) (in french).

The further the implementation went the further it revealed inconsistencies in packages APIs. Some things that should be simple became complex very fast.

The work of the past year has been toward fixing those inconsistencies.

<!-- more -->

The result is a new major version of `innmind/foundation`: a V2.

But let's backtrack a bit.

## The problem

Many packages of the ecosystem use interfaces to define the contracts between different implementations. `Transport` for `innmind/http-transport`, `Adapter` for `innmind/filesystem`, etc...

The idea was that on top of allowing a package to provide multiple implementations, it allowed user land implementations and let them compose them the way they want. This has been mostly fine until now.

The first _easy_ problem with interfaces is the ability to add new features. Adding a new method on an interface is a BC break. The easy fix is to create a new major version, except this means all dependents packages need to be updated. This is painful and mainly wasted time. That's why `innmind/immutable` already moved away from interfaces **7 years ago**.

The second _more complex_ problem lies with composition. Let's use the HTTP client to show the problem:

```php
use Innmind\HttpTransport\FollowRedirections;

$transport = $os
    ->map(static fn($config) => $config->mapHttpTransport(
        FollowRedirections::of(...),
    ))
    ->remote()
    ->http();
```

This code by default uses a `cURL` transport and then applies a decorator to follow response redirections. In a traditionnal, synchronous, PHP program this works just fine.

However this falls apart with asynchronous code.

One of the philosophies of Innmind is that a code relying on `innmind/operating-system` should be able to be run synchronously or asynchronously without any code change.

This means that the `$os` above is passed to a function, it's expected that the HTTP calls made by this function will follow redirections. And if the function is run asynchronously (via [`innmind/async`](https://github.com/Innmind/async)) you'd expect the same behaviour.

With interfaces it's too painful. 

To make HTTP calls async, it swaps the underlying `cURL` implementation for an async one and then needs to re-apply all decorators on top of it (in this case following redirections).

If the underlying `cURL` implementation had specific configurations such as disabled SSL verification or concurrency limit, then this configuration is lost when swapped for the async implementation.

The more the composition the more the problem became apparent as it multiplied the configuration possibilities.

This was a recurring problem across the ecosystem.

## The solution

The solution lies in the `3.0.0` release of `innmind/immutable` from **7 years ago**:

<div class="annotate" markdown>
- `final` classes with named constructors
- Functors (1) aka `map` functions
</div>

1. Did you expects Monads ? :smile:

To reuse the HTTP client example:

=== "Before"
    ```php
    use Innmind\HttpTransport\{
        Curl,
        FollowRedirections,
    };
    
    $transport = Curl::of(...$args);
    $transport = FollowRedirections::of($transport);
    ```

=== "After"
    ```php
    use Innmind\HttpTransport\Transport;
    
    $transport = Transport::curl(...$args);
    $transport = Transport::followRedirections($transport);
    ```

Now `Transport` is a `final` class. This let the package add features without BC breaks in minor releases.

And it fixes the configuration problem described above thanks to the `map` method:

=== "Before"
    ```php
    $configure = static fn(Transport $transport) => FollowRedirections::of($transport);
    
    $sync = $configure(Curl::of(...$args)->maxConcurrency(20));
    $async = $configure(new AsyncTransport); #(1)
    ```

    1. `AsyncTransport` class doesn't exist, it's just to prove the point.

=== "After"
    ```php
    $configure = static fn(Transport $transport) = Transport::followRedirections($transport)
        ->map(static fn($config) => $config->limitConcurrencyTo(20));

    $sync = $configure(Transport::curl(...$args));
    $async = $configure(Transport::async(...$args));
    ```

Previously `$async` lost the limit of `20` concurrent requests. But now both `$sync` and `$async` are configured the same way.

These changes have been made throughout the ecosystem and led to the V2 of `innmind/foundation` and dependent packages.

You can follow the evolution of releases and the dependencies at <http://innmind.net/vendor/innmind>.

## What's next ?

With interfaces gone it brought a new problem: specific implementations. They can be classified either as user land implementations or in tests with mocks.

User land implementations are indeed gone for now but could be brought back via config methods. This would allow each package to keep the control over the config/composition matrix while still allowing users to extend the behaviour. But unless explicit cases are presented, no work will be done on this.

As of mocks it's a problem Innmind faces itself. Historically mocks have been used throughout the packages tests.

For many years now Innmind has tried to move away from this technique because it brought maintenance problems. Mainly the fact that mocks behaviours diverged from concrete implementations. That's why mocks have not, and will never, been implemented in `innmind/black-box`.

Even if it fell out favour mocks were kept in tests to reduce the maintenance burden.

But now that interfaces are gone, mocks must go too.

To still be able to use fake implementations in tests while still keeping a behaviour consistency a new project was started at [`innmind/testing`](https://github.com/Innmind/testing/).

The initial goal was to build factory objects configurable via simple anonymous functions to fake the implementations of `innmind/http-transport`, `innmind/filesystem`, etc...

Early work on this has shown that it could **much more**. The goal is now to make it a [Deterministic Simulation Testing](https://journal.resonatehq.io/p/deterministic-simulation-testing) framework. 

This will allow for a better foundation to test the behaviour of `innmind/actors`.

For end users this will allow to simulate a production stack, with multiple servers, inside a single PHP process. Combined with `innmind/black-box` this means randomly simulating production traffic in a fraction of the time.

Possibilities are going to be insane :star_struck: .
