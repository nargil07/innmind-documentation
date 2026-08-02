# Persist crawled links to a database

```php
use Innmind\Http\{
    Request,
    Method,
    ProtocolVersion,
};
use Innmind\Html\{
    Reader,
    Visitor\Elements,
    Element\A,
};
use Innmind\Url\Url;
use Innmind\Immutable\Predicate\Instance;
use Formal\AccessLayer\{
    Query\Insert,
    Table\Name,
    Row,
};

$read = Reader::new();
$sql = $os
    ->remote()
    ->sql(Url::of('mysql://127.0.0.1:3306/database_name'))
    ->unwrap();

$_ = $os
    ->remote()
    ->http()(Request::of(
        Url::of('https://some-server.com/page.html')
        Method::get,
        ProtocolVersion::v11,
    ))
    ->attempt(static fn() => new \RuntimeException)
    ->map(static fn($success) => $success->response()->body())
    ->flatMap($read)
    ->maybe()
    ->toSequence()
    ->flatMap(Elements::of('a'))
    ->keep(Instance::of(A::class))
    ->map(static fn(A $a) => $a->href()->toString())
    ->foreach(static fn(string $href) => $sql(Insert::into(
        Name::of('table_name'),
        Row::of(['column_name' => $href]),
    )));
```
