# Framework

## Installation

```sh
composer require innmind/framework:~4.1
```

## Concepts

The framework is defined by an entrypoint that specify the context in which the framework will be run. Each entrypoint exposes a `configure` method to configure an immutable `Application` object.

`Application` is the way to describe what your program can do. This is the same class no matter which entrypoint you choose. This allows you to switch the execution context without modifying any line of your code (1).
{.annotate}

1. For example moving from a synchronous HTTP context to an async HTTP server.

The other advantage of this approach is that if your program is accessible from both HTTP and CLI it can be [configured by the same code](middlewares.md).

## Advanced usage

The framework offers more than the features shown in this documentation, after reading the following chapters you should head to the [package](https://innmind.org/framework/) to learn the extent of its capabilities.
