# Clock

PHP allows to access time anywhere in a program in various ways (1) but this becomes problematic when you want to test your program. Especially if it depends heavily on time.
{.annotate}

1. `time`, `new \DateTime`, etc...

By using a _clock_ object that you inject everywhere you need to access time you'll be able to configure the date on which your program is tested. This means that you can test your program in the past (1), present and future (2).
{.annotate}

1. So you can reproduce bugs that appeared in production.
2. You can anticipate problematic times such as leap years, daylight saving time, etc...

And to ease the manipulation, all objects are immutable.

## Accessing time

```php
use Innmind\Time\{
    Point,
    Format,
};

echo $os
    ->clock()
    ->now() // returns a Point object
    ->format(Format::iso8601());
```

This will print something like `2024-05-04T13:05:01+02:00`.

!!! tip ""
    You can specify your own formats with the named constructor `Innmind\Time\Format::of()`. The format itself is described by a string that must be understood by the `\DateTimeInterface::format()` method.

On a `Point` you can access every part of the time it references (year, month, day, etc...), and has methods to modify the time to move forward or backward in time.

## Parsing time from a string

When you receive a `string` (1) that represent a date and you want to convert it to a `PointInTime` you should also use the clock:
{.annotate}

1. from an HTTP request or loading it from a database

```php
$point = $os
    ->clock()
    ->at($string, Format::iso8601()) #(1)
    ->match(
        static fn(Point $point) => $point,
        static fn() => throw new \RuntimeException("'$string' is not a valid date"),
    );
```

1. The format is required to avoid implicit convertions.

The `at` method returns the `Attempt<Point>`(1) type to make sure you always handle the case the `$string` is invalid.
{.annotate}

1. `Innmind\Immutable\Attempt`

## Calculating elapsed time

We need to calculate elapsed time, among other cases, when handling heartbeats when dealing with sockets or in tests to make sure some code is executed in a certain amount of time.

The usual approach is to use a call to `microtime()` at the start and an another at the end and subtract them. The problem with this approach is that you can end up with a negative durations. This happens when your machine re-synchronise its clock via the [NTP protocol](https://en.wikipedia.org/wiki/Network_Time_Protocol) and sometimes it can go back in time (1).
{.annotate}

1. To avoid this problem the solution is to use a monotonic clock (via the `hrtime()` PHP function).

With Innmind you don't have to worry about that!

```php
$start = $os->clock()->now();

// do some stuff

$duration = $os
    ->clock()
    ->now()
    ->elapsedSince($start);
```

Here `$duration` is an instance of `Innmind\Time\ElapsedPeriod` that contains the number of seconds, milliseconds and microseconds between the 2 points in time. And it handles the case that your machine may go back in time.

??? info
    The time shift is handled when working with objects coming from `$clock->now()`, this is not the case when working with objects coming from `$clock->at()`.

## In the ecosystem

All packages that depend on this abstraction use this clock, but this abstraction itself also uses this clock. So no matter the level of abstractions you work on you can change the clock implementation in your tests.

## Full documentation

A more extensive documentation can be found at <https://innmind.org/time/>.
