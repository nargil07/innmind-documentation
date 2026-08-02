# Persist a SQL result to a file

```php
use Innmind\Filesystem\{
    File,
    File\Content,
    File\Content\Line,
};
use Innmind\Url\{
    Url,
    Path,
};
use Innmind\Immutable\Str;
use Formal\AccessLayer\{
    Query\Select,
    Table\Name,
};

$sql = $os
    ->remote()
    ->sql(Url::of('mysql://127.0.0.1:3306/database_name'))
    ->unwrap();

$_ = $os
    ->filesystem()
    ->mount(Path::of('some directory/'))
    ->unwrap()
    ->add(File::named(
        'results.csv',
        Content::ofLines(
            $sql(Select::lazily(Name::of('table_name')))
                ->map(
                    static fn($row) => $row
                        ->values()
                        ->map(static fn($value) => (string) $value->value()),
                )
                ->map(Str::of(',')->join(...))
                ->map(Line::of(...)),
        ),
    ))
    ->unwrap();
```

Since the sql query is lazy you can persist a very long result without loading everything in memory.
