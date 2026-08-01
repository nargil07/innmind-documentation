# PHP Process

## Pausing

If you need to pause your program to wait for external thing to happen (or any other reason), you can pause it this way:

```php
use Innmind\Time\Period;

$os
    ->process()
    ->halt(Period::second(10))
    ->unwrap();
```

You can use any unit of period except months because it's not an absolute value.

??? info
    If you want to wait for years it will compute that as `365` days. But if you need to do this there may be a design problem in your program.

## Handling CLI Signals

Any process can receive signals to tell them a user (or the system) wants to shut them down allowing the process to terminate gracefully (1).
{.annotate}

1. This is the prevalent usage, but [there are more](https://en.wikipedia.org/wiki/Signal_(IPC)).

For example let's say you need to import a large csv file into a database but you want to be able to stop it gracefully. You can do:

```php hl_lines="13 15 18-21 32-34"
use Innmind\Signals\Signal;
use Innmind\Filesystem\{
    File,
    File\Content\Line,
    Name,
};
use Innmind\Url\Path;
use Innmind\Immutable\{
    Sequence,
    Predicate\Instance,
};

$signaled = false;
$stop = static function() use (&$signaled): void {
    $signaled = true;
};

$os
    ->process()
    ->signals()
    ->listen(Signal::interrupt, $stop);

$os
    ->filesystem()
    ->mount(Path::of('data/'))
    ->unwrap()
    ->get(Name::of('users.csv'))
    ->keep(Instance::of(File::class))
    ->map(static fn(File $file) => $file->content()->lines())
    ->toSequence() #(1)
    ->flatMap(static fn(Sequence $lines) => $lines)
    ->takeWhile(static function() use (&$signaled) {
        return !$signaled;
    })
    ->foreach(static fn(Line $line) => importToDb($line));

$os
    ->process()
    ->signals()
    ->remove($stop);
```

1. `Maybe<Sequence>->toSequence()->flatMap(fn($sequence) => $sequence)` is a way to _swallow_ the fact that the file may not exist.

This way if a user sends a signal to interrupt the script, it will:

- pause the execution
- call the `$stop` function
- modify the `$signaled` flag
- resume the execution
- the next line that is attempted to be read won't be done because of `takeWhile`

??? tip
    You can specify multiple listeners for a single signal and they'll be executed in the order you added them.
