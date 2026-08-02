# Tests

Before learning to write proofs and properties, you should familiarizes with BlackBox API by writing simple tests.

## Specific scenario

If we reuse the [`add` function](property-based-testing.md#examples):

```php
use Innmind\BlackBox\{
    Application,
    Runner\Assert,
    Prove,
};

Application::new([])
    ->tryToProve(function(Prove $prove) {
        yield $prove->test(
            'add(1, 2)',
            static fn(Assert $assert) => $assert
                ->expected(3)
                ->same(add(1, 2)),
        );
    })
    ->exit();
```

Here we use a short function with only one assertion but you can run any number of assertions in a function.

!!! info ""
    You should explore the `Assert`ion API with your code editor to discover all its capabilities.

## Same scenario for multiple values

Sometime you may want to run the same test but with a different set of values. Since declaring the tests is a generator you can do:

```php
use Innmind\BlackBox\{
    Application,
    Runner\Assert,
    Prove,
};

Application::new([])
    ->tryToProve(function(Prove $prove) {
        $cases = [
            [1, 2, 3],
            [2, 3, 5],
            // etc...
        ];

        foreach ($cases as [$a, $b, $expected]) {
            yield $prove->test(
                "add($a, $b)",
                static fn(Assert $assert) => $assert
                    ->expected($expected)
                    ->same(add($a, $b)),
            );
        }
    })
    ->exit();
```
