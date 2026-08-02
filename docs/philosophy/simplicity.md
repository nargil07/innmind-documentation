# Simplicity

## Complexity vs Difficulty

We tend to use simple and easy or difficult and complex interchangeably. But they're very much different.

Complexity is an objective scale (1) of a number of parts of a system and the number of interactions between each part.
{.annotate}

1. with Simplicity being on one end of this scale

Difficulty is a subjective scale related to your familiarity with a subject. By familiarity ear the number of times you've done some task.

A general example of this difference is an electric circuit to light a bulb:

- it is _simple_ to use, you only need to be aware of the switch to light the bulb (1)
    {.annotate}

    1. even with many switches to light the same bulb the complexity is the same

- it is _complex_ to build such circuit (1)
    {.annotate}

    1. especially with mutliple switches

- it is _difficult_ for a child to build such circuit
- it is _easy_ for an electrician to build such circuit

Innmind heavily leans toward simplicity. Even if at times it doesn't feel easy.

## In practice

The [Filesystem package](../getting-started/operating-system/filesystem.md) was bitten (in an early version) for mistaking easyness by simplicity.

The `Adapter` interface has a `get` method to return a file. Initially the argument passed to it was a `string` to represent the file name. But months later when building an [S3](https://github.com/Innmind/S3) abstraction for this interface it wasn't clear if a path could be passed in the string.

The easiness of using a `string` brought complexity to the implementation to make sure all adapters behave the same way. And also brought difficulty to the user when switching an adapter for another, having to deal with the inconsistencies.

The `string` was later replaced by a class named `Name`. Any `Adapter` implementation has to check its behaviour to understand what's possible, no need to be aware of other implementations anymore.

You'll find all kind of classes in this ecosystem that encapsulate values to reach this kind of simplicity.
