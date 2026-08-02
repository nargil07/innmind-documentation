# Inter Process Communication

When your program runs across multiple processes you may want to communicate between them to update some state.

Innmind IPC use unix sockets to send messages between processes.

## Installation

```sh
composer require innmind/ipc:~5.0
```

## Usage

=== "Server"
    ```php
    use Innmind\IPC\{
        IPC,
        Process\Name,
        Continuation,
        Server,
        Message,
    };
    use Innmind\Immutable\Monoid;
    
    /**
     * @psalm-immutable
     * @implements Monoid<int>
     */
    final class Addition implements Monoid
    {
        public function identity(): int
        {
            return 0;
        }
    
        public function combine(mixed $a, mixed $b): int
        {
            return $a + $b;
        }
    }
    
    $ipc = IPC::build($os);
    $counter = $ipc
        ->serve(Name::of('a'))
        ->sink(new Addition)
        ->monitor(static fn(int $counter, Server\Continuation $continuation) => match ($counter) {
            42 => $continuation->finish(),
            default => $continuation,
        })
        ->with(
            static fn(Message $message, Continuation $continuation, int $counter) => $continuation
                ->respond($message)
                ->carryWith($counter + 1),
        )
        ->unwrap();
    // $counter will always be 42 in this case
    ```

    The server behaves like a reduce operation, with a carried value and a function that's called every time a client sends a message. The carried value can be of any type.

    In this case the server will stop after receiving `42` messages.

=== "Client"
    ```php
    use Innmind\IPC\{
        IPC,
        Process,
        Process\Name,
        Message,
    };
    use Innmind\MediaType\{
        MediaType,
        TopLevel,
    };
    use Innmind\Immutable\{
        Str,
        Sequence,
    };
    
    $ipc = IPC::build($os);
    $server = Name::of('a');
    $response = $ipc
        ->connectTo($server)
        ->flatMap(
            static fn(Process $process) => $process
                ->send(Sequence::of(
                    Message::of(
                        MediaType::from(TopLevel::text, 'plain'),
                        Str::of('hello world'),
                    ),
                ))
                ->map(static fn() => $process),
        )
        ->flatMap(fn(Process $process) => $process->wait())
        ->match(
            static fn(Message $message) => 'server responded '.$message->content()->toString(),
            static fn() => 'no response from the server',
        );
    print($message);
    ```

    This will wait for the server to be up then it will send a `ping` message and wait for the server to respond. Then it will print `server responded pong` since the server always repond with this message unless it has stopped in the meantime.
