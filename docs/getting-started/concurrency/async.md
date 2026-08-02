# Asynchronous code

Since Innmind offers to access all I/O operations through the [operating system abstraction](../operating-system/index.md), it can easily execute these operations asynchronously.

## Usage

Async works a bit like a reduce operation. The _reducer_ function allows to launch tasks and react to their results. Both the _reducer_ and tasks are run asynchronously.

=== "Script"
    ```php
    use Innmind\Async\Scheduler;
    use Innmind\Http\Response;
    use Innmind\Immutable\Sequence;

    $responses = Scheduler::of($os)
        ->sink(Carried::new())
        ->with(new Reducer);
    $responses; // instance of Sequence<?Response>
    ```

=== "Carried value"
    Like in a real reduce operation you need a carried value that will be passed to the reducer every time it's called.

    Here we use a `Carried` class but you can use any type you want.

    ```php
    use Innmind\Http\Response;
    use Innmind\Immutable\Sequence;

    final readonly class Carried
    {
        /** @var Sequence<?Response> */
        private function __construct(
            private bool $tasksLaucnhed,
            private Sequence $responses,
        ) {}

        public static function new(): self
        {
            return new self(false, Sequence::of());
        }

        public function tasksLaunched(): self
        {
            return new self(true, $this->responses);
        }

        public function needsToLaunchTasks(): bool
        {
            return !$this->tasksLaunched;
        }

        public function got(?Response $response): self
        {
            return new self(
                $this->tasksLaunched,
                $this->responses->add($response),
            );
        }

        /** @return Sequence<?Response> */
        public function responses(): Sequence
        {
            return $this->responses;
        }
    }
    ```

    !!! warning ""
        To avoid unexpected side effects you should always use an immutable value for the carried value.

=== "Reducer"
    ```php
    use Innmind\Async\Scope\Continuation;
    use Innmind\OperatingSystem\OperatingSystem;
    use Innmind\Http\Response;
    use Innmind\Immutable\Sequence;

    final class Reducer
    {
        /**
         * @param Continuation<Carried> $continuation
         *
         * @return Continuation<Carried>
         */
        public function __invoke(
            Carried $carried,
            OperatingSystem $os, #(1)
            Continuation $continuation,
        ): Continuation {
            if ($carried->needsToLaunchTasks()) {
                return $continuation
                    ->carryWith($carried->tasksLaunched()) #(2)
                    ->schedule(Sequence::of(
                        static fn(OperatingSystem $os) => MyTask::of( #(3)
                            $os,
                            'https://github.com/'
                        ),
                        static fn(OperatingSystem $os) => MyTask::of(
                            $os,
                            'https://gitlab.com/'
                        ),
                    ));
            }

            $carried = $continuation->results()->reduce(
                $carried,
                static fn(
                    Carried $carried,
                    ?Response $response
                ) => $carried->got($response),
            );

            if ($carried->responses()->size() === 2) {
                return $continuation
                    ->carryWith($carried)
                    ->finish(); #(4)
            }

            return $continuation
                ->carryWith($carried)
                ->wakeOnResult();
        }
    }
    ```

    1. This `$os` variable is a new instance built by Async and runs asynchronously.
    2. We flip the flag in order to not launch the tasks each time the reducer is called.
    3. The `$os` variable is a dedicated new instance for each task.
    4. This tells Async to stop calling the reducer and return the carried value.

    This `__invoke` method will be called once when starting the runner and then each time a task finishes.

    The flag to know if the tasks have been launched is stored in the carried value, but since we're in an object it could be placed as a property. This is done so you can better differentiate the carried values from the `$results` in this example.

=== "MyTask"
    ```php
    use Innmind\OperatingSystem\OperatingSystem;
    use Innmind\Http\{
        Request,
        Response,
        Method,
        ProtocolVersion,
    };
    use Innmind\Url\Url;

    final class MyTask {
        public static function of(
            OperatingSystem $os,
            string $url,
        ): ?Response {
            return $os
                ->remote()
                ->http()(Request::of(
                    Url::of($url),
                    Method::get,
                    ProtocolVersion::v11,
                ))
                ->match(
                    static fn(Success $success) => $success->response(),
                    static fn() => null,
                );
        }
    }
    ```


## Advantages

The first big advantage of this design is that your task is completely unaware that it is run asynchronously. It all depends on the `$os` variable injected (1). This means that you can easily experiment a piece of your program in an async context by what code calls it, your program logic itself doesn't have to change!
{.annotate}

1. If it comes from Async it's async otherwise it's sync.

The side effect of this is that you can test your code synchronously even though it's run asynchronously.

The other advantage is that since all state is local you can compose async code inside sync code transparently. You can't affect a global state since there is none.

## Limitations

- HTTP calls are currently done via `cURL` and uses micro sleeps instead of watching sockets
- SQL queries are still run synchronously for now
- It seems there is a limit of 100k concurrent tasks before performance degradation

Most of these limitations are planned to be fixed in the future.

!!! warning ""
    You may not want to use this in production just yet, or at least not for mission critical code.
