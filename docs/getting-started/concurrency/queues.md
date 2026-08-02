# Queues

Innmind uses the [AMQP protocol](https://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol) to build queues.

You need to first install a server that implements this protocol (1). The most well known server is [RabbitMQ](https://www.rabbitmq.com).
{.annotate}

1. Only the version `0.9` is supported. (`1.0` is a completely different protocol)

## Installation

```sh
composer require innmind/amqp '~7.0'
```

## Usage

```php
use Innmind\AMQP\{
    Factory,
    Command\DeclareExchange,
    Command\DeclareQueue,
    Command\Bind,
    Model\Exchange\Type,
};
use Innmind\IO\Sockets\Internet\Transport;
use Innmind\Time\Period;
use Innmind\Url\Url;

$client = Factory::of($os)
    ->build(
        Transport::tcp(),
        Url::of('amqp://guest:guest@localhost:5672/'),
        Period::second(1), // heartbeat
    )
    ->with(DeclareExchange::of('crawler', Type::direct))
    ->with(DeclareQueue::of('parser'))
    ->with(Bind::of('crawler', 'parser'));
```

This builds the basis of an AMQP client. As is it does nothing until it's run (more in a bit). The client is immutable and each call to `with` returns a new instance. In this case the `$client` will create an exchange named `crawler`, create a queue `parser` and will route every message published to `crawler` directly to `parser`.

??? tip
    You can head to the [RabbitMQ tutorials](https://www.rabbitmq.com/tutorials) to understand exchanges, queues and how to route your messages between the two.

The first step is to publish messages before trying to consume them.

```php
use Innmind\AMPQ\{
    Model\Basic\Message,
    Command\Publish,
    Failure,
};
use Innmind\Immutable\Str;

$message = Message::of(
    Str::of('https://github.com');
);
$client
    ->with(Publish::one($message)->to('crawler'))
    ->run(null) #(1)
    ->unwrap();
```

1. For now don't worry about this `null`, just know that it's required.

The client will execute anything only when the `run` method is called. In this case, because we reuse the client from above, it will create the exchange, the queue and bind them together and then publish one message that will end up in the queue.

If everything works fine then it will return `null`. If any error occurs it will be throw an exception when calling `->unwrap()`.

??? info
    Using a client that always declare the the exchange and queues that it requires allows for a hot declaration of your infrastructure when you try to use the client. And if the exchanges, queues and bindings already exist it will silently continue to execute as the structure is the way you expect on the AMQP server.

Then to consume the queue:

```php
use Innmind\AMQP\{
    Command\Consume,
    Model\Basic\Message,
    Consumer\Continuation,
    Failure,
};

$count = $client
    ->with(Consume::of('parser')->handle(
        static function(
            int $count, #(1)
            Message $message,
            Continuation $continuation,
        ): Continuation {
            doStuff($message);

            if ($count === 42) {
                return $continuation->cancel($count);
            }

            return $continuation->continue($count + 1);
        },
    ))
    ->run(0) #(2)
    ->unwrap();
var_dump($count);
```

1. This argument is a carried value between each call of this function.
2. This is the initial value passed to the function handling the messages.

Here we reuse the client from the first example to make sure we indeed have a `parser` queue to work on. Then we _consume_ the queue, meaning we'll wait for incoming messages and call the function when one arrives. This function behaves like a reduce operation where the initial value is `0` and is incremented each time a message is received. On the 43th message we'll handle the message and ask the client to stop consuming the queue.

At this point the `run` method will return `42`.

In this case the carried value is an `int` but you can use any type you want.

??? tip
    If you only need to pull one message from the queue you should use `Innmind\AMQP\Command\Get` instead of `Consume`.

??? tip
    When consuming a queue by default the server will send as many messages as it can through the socket so there's no wait time when dealing the _next_ message. However depending on the throughput of your program it can send too many messages in advance.

    To prevent network saturation you can use `#!php Innmind\AMQP\Command\Qos::of(100)` where `100` is the number of messages to send in advance. Add this command before adding the `Consume` command to the client.
