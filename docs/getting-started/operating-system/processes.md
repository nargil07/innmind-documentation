# Launching Processes

## Usage

```php
use Innmind\Server\Control\Server\{
    Command,
    Process\Success,
    Process\TimedOut,
    Process\Failed,
    Process\Signaled,
};

$process = $os
    ->control()
    ->processes()
    ->execute(
        Command::foreground('apt-get')
            ->withArgument('install')
            ->withArgument('cowsay')
            ->withShortOption('y'),
    )
    ->unwrap();
$process
    ->wait()
    ->match(
        static fn(Success $success) => doStuff(),
        static fn(TimedOut|Failed|Signaled $error) => throw new RuntimeException();
    );
```

This example waits for the installation of [`cowsay`](https://en.wikipedia.org/wiki/Cowsay) before continuing via `doStuff()` or it will fail with an exception.

??? note
    By default the process is executed with no environment variables. If you try to execute a command that is reachable only because you modified your `$PATH` environment variable, you'll need to specify it via `Command::foreground('command')->withEnvironment('$PATH', 'your path value')`.

    This may seem restrictive at first but it's done to force your program to be [explicit](../../philosophy/explicit.md). And it will help other developers to understand what's needed for the command to be run.

??? info
    If you don't want to wait for a process to finish you can replace `#!php Command::foreground()` by `#!php Command::background()` and remove the code `#!php $process->wait()`.

If you dont't really care about the process failing or not and simply want to _forward_ its output you can use:

```php
use Innmind\Server\Control\Server\Process\Output\{
    Chunk,
    Type,
};
use Innmind\Immutable\Str;

$process
    ->output()
    ->foreach(static function(Chunk $chunk): void {
        // $chunk->type() is either Type::output or Type::error
        echo $chunk->data()->toString();
    });
```

This code will print the output of the underlying process in real time. The `foreach` call will return when the process is finished.

??? tip
    If you still need to check the result of the process you can still call `#!php $process->wait()`, it will immediately return the result.

You can also send content to the `STDIN` of the process via:

```php
use Innmind\Filesystem\File\Content;
use Innmind\Immutable\Monoid\Concat;

echo $os
    ->control()
    ->processes()
    ->execute(
        Command::foreground('echo')
            ->withInput(Content::ofString('some input')),
    )
    ->unwrap()
    ->output()
    ->map(static fn(Chunk $chunk) => $chunk->data())
    ->fold(Concat::monoid)
    ->toString();
```

The input can be [any valid `Content`](filesystem.md) object, even lazy ones.

## Streaming

By default the process output is kept in memory so you can use it multiple times. However for some commands the output can be quite large and it won't fit in memory.

For example you want to run an archive command that you want to stream to the output.

```php hl_lines="10"
$os
    ->control()
    ->processes()
    ->execute(
         Command::foreground('zip')
            ->withShortOption('q')
            ->withShortOption('r')
            ->withArgument('-')
            ->withArgument('some folder/')
            ->streamOutput(),
    )
    ->unwrap()
    ->output()
    ->foreach(static function(Chunk $chunk) {
        echo $chunk->data()->toString();
    });
```

!!! note ""
    You won't be able to reuse the output twice, if you try it will throw a `\LogicException`.

??? info
    However if you've walked over the whole output you can still call `#!php $process->wait()` to check if there was an error or not.

## SSH

You can execute commands on a remote machine through SSH the same way you'd do it on the local machine via:

```php hl_lines="4-5"
use Innmind\Url\Url;

$process = $os
    ->remote()
    ->ssh(Url::of('ssh://user@machine-name-or-ip:22/'))
    ->processes()
    ->execute(
        Command::foreground('apt-get')
            ->withArgument('install')
            ->withArgument('cowsay'),
    )
    ->unwrap();
```

!!! note ""
    You can't specify the password to connect to the machine via the url. It's done to force you to use SSH keys.

??? info
    For now it's not possible to use an input when running commands through SSH.
