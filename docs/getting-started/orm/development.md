# Development

## Setup

For this chapter we'll work with a `User` class that can have multiple `Address` objects.

To keep things simple we'll work with an in memory persistence. You'll learn how to really persist them in the [next chapter](production.md).

```php
use Formal\ORM\Manager;
use Innmind\Filesystem\Adapter;

$orm = Manager::filesystem(Adapter::inMemory()); #(1)
```

1. From this point on every time you see `$orm` it will come from this example.

=== "`User`"
    ```php
    use Formal\ORM\{
        Id,
        Definition\Contains,
    };
    use Innmind\Immutable\Set;

    final readonly class User
    {
        /**
         * @param Id<self> $id
         * @param Set<Address> $addresses
         */
        private function __construct(
            private Id $id,
            private string $username,
            #[Contains(Address::class)] #(1)
            private Set $addresses,
        ) {}

        public static function new(string $username): self
        {
            return new self(
                Id::new(self::class),
                $username,
                Set::of(),
            );
        }

        public function addAdress(Address $address): self
        {
            return new self(
                $this->id,
                $this->username,
                $this->addresses->add($address),
            );
        }
    }
    ```

    1. This allows the ORM to know how to persist the data inside the collection.

    A `User` is called an aggregate in this ORM. This is the root object that have ownership of every data inside it (more on that below).

    An aggregate must have an `Id $id` property. All the other properties will be automatically stored if the ORM understands the type defined on the property.

    ??? info
        A `Set` is an immutable unsorted collection of unique objects.

=== "`Address`"
    ```php
    final readonly class Address
    {
        public function __construct(
            private string $street,
            private string $zipcode,
            private string $city,
        ) {}
    }
    ```

    An `Address` is called an entity in this ORM. As you can see it doesn't have any id. The ORM knows an object of this class belongs to a given user because it is found inside its `$addresses` property.

    Of course nothing prevents you to add your own id to an entity, but the ORM will treat it as any other property.

## Persisting a new aggregate

```php
use Innmind\Immutable\Either;

$repository = $orm->repository(User::class);
$orm->transactional(
    static function() use ($repository) {
        $repository->put(User::new('john'))->unwrap();
        $repository->put(User::new('jane'))->unwrap();

        return Either::right(null);
    },
);
```

In order to persist aggregates you need to first access their repository. You can think of this `$repository` as a persistent collection of all your objects for a given class.

To modify (1) any data it has to be done in a transaction via `$orm->transactional()`. This is done to make sure your program is structurally correct. If you try to modify the data outside it will throw an exception, this prevents unforeseen modifications outside of the context you expect. Note that this applies to calls on a repository methods, not the aggregate objects.
{.annotate}

1. `#!php $repository->put()` or `#!php $repository->remove()`

The function passed to `transactional` has to return an [`Either`](../handling-data/either.md). If it contains a value on the _right_ side then it will commit the transaction and if it contains a value on the _left_ side (or throws an exception) it will rollback the transaction. The `transactional` method will return the `Either` as you'd expect.

In our case we return `null` on the right side as we don't have any business logic that can fail.

Let's say now that we want to create 2 users that live in the same city:

```php
$address = new Address('somewhere', '75001', 'Paris');
$john = User::new('john')->addAddress($address);
$jane = User::new('jane')->addAddress($address);

$repository = $orm->repository(User::class);
$orm->transactional(
    static function() use ($repository, $john, $jane) {
        $repository->put($john)->unwrap();
        $repository->put($jane)->unwrap();

        return Either::right(null);
    },
);
```

Even though we use the same `Address` object for both users the address will be stored twice. This is possible because the `Address` is an immutable object that represents data, the object _reference_ has no relevance for the ORM.

!!! success ""
    This design as a HUGE benefit: you can't mess up your objects relations.

## Retrieving an aggregate

Once you persisted an aggregate you'll need to retrieve it, which is pretty straight forward:

```php
$repository
    ->get(Id::of(User::class, 'some-uuid'))
    ->match(
        static fn(User $user) => doStuff($user),
        static fn() => userDoesntExist(),
    );
```

You should replace `'some-uuid'` with the string representation of and id (via the `toString` method).

Since the user you're asking for may not exist in the storage, the repository returns a [`Maybe<User>`](../handling-data/maybe.md) so you're forced to handle both cases.

## Modifying an aggregate

To modify an aggregate you need to _re-add_ it to the repository since the objects are immutable.

```php
$orm->transactional(
    static function() use ($repository) {
        $_ = $repository
            ->get(Id::of(User::class, 'some-uuid'))
            ->map(static fn(User $user) => $user->addAddress(
                new Address('somewhere', 'SW9 9SL', 'London'),
            ))
            ->match(
                static fn(User $user) => $repository->put($user)->unwrap(),
                static fn() => null,
            );

        return Either::right(null);
    },
);
```

The benefit here is that you can't persist data by accident. All modifications to the persistence are [explicit](../../philosophy/explicit.md).

## Deleting an aggregate

```php
$orm->transactional(
    static function() use ($repository) {
        $repository->remove(Id::of(User::class, 'some-uuid'))->unwrap();

        return Either::right(null);
    },
);
```

Whether any aggregate with this id existed or not it will return nothing and won't throw an exception.

## Retrieving a collection of aggregates

The simplest way is to retrieve all aggregates:

```php
$repository
    ->all()
    ->foreach(static fn(User $user) => doStuff($user));
```

Even if you have thousands of aggregates in your storage this code will work because the ORM keeps track of an aggregate as long as _you_ keep it in memory.

Usually you won't want to retrieve all aggregates, you need only a subset. You could use `#!php $repository->all()->filter()` but this is fairly innefficient as it retrieve all aggregates and throw out the ones you don't use.

The best approach is to filter directly at the storage level. You do this via the [specification pattern](../operating-system/sql.md#filtering).

Let's say we want all users with an address in `London`. First we need to build a specification:

```php
use Innmind\Specification\{
    Comparator\Property,
    Sign,
};

final class City
{
    /** @psalm-pure */
    public static function of(string $city): Property
    {
        return Property::of(
            'city', #(1)
            Sign::equality,
            $city, #(2)
        );
    }
}
```

1. This is the name of the property in the `Address` class.
2. This type has to be the same as the one on the property.

And you use it like this:

```php
use Formal\ORM\Specification\Child;

$repository
    ->matching(Child::of(
        'addresses', #(1)
        City::of('London'),
    ))
    ->foreach(static fn(User $user) => doStuff($user));
```

1. This is the property name on the `User` class.

And if you want to target `London` or `Paris` you can do `#!php City::of('London')->or(City::of('Paris'))`.

!!! success ""
    This is the same approach as the [pure SQL one](../operating-system/sql.md#filtering). So you can more easily upgrade from one to the other.

You can of course also limit the number of aggregates to retrieve via `#!php $repository->matching($specification)->drop($x)->take($y)`.

## Custom types

So far you've only seen how to persist `string` properties. But you can use your own types.

For example let's you want to create a `Username` class to prevent using empty usernames.

```php
final readonly class Username
{
    private string $value;

    public function __construct(string $value)
    {
        if ($value === '') {
            throw new \DomainException;
        }

        $this->value = $value;
    }

    public function toString(): string
    {
        return $this->value;
    }
}
```

Now you need to update the aggregate:

```php hl_lines="15 20"
use Formal\ORM\{
    Id,
    Definition\Contains,
};
use Innmind\Immutable\Set;

final readonly class User
{
    /**
     * @param Id<self> $id
     * @param Set<Address> $addresses
     */
    private function __construct(
        private Id $id,
        private Username $username,
        #[Contains(Address::class)]
        private Set $addresses,
    ) {}

    public static function new(Username $username): self
    {
        return new self(
            Id::new(self::class),
            $username,
            Set::of(),
        );
    }

    public function addAdress(Address $address): self
    {
        return new self(
            $this->id,
            $this->username,
            $this->addresses->add($address),
        );
    }
}
```

The last part is to tell the ORM how to convert this type. You need to create a class implementing the `Type` interface.

```php
use Formal\ORM\Definition\Type;

/**
 * @psalm-immutable
 * @implements Type<Username>
 */
final class UsernameType implements Type
{
    public function normalize(mixed $value): null|string|int|bool
    {
        return $value->toString();
    }

    public function denormalize(null|string|int|bool $value): mixed
    {
        if (!\is_string($value)) {
            throw new \LogicException;
        }

        return new Username($value);
    }
}
```

??? tip
    You don't need to handle the `null` value in your type, the ORM already does that for you.

And you register this class when creating the ORM:

```php hl_lines="10-12"
use Formal\ORM\{
    Manager,
    Definition\Aggregates,
    Definition\Types,
    Definition\Type\Support,
};
use Innmind\Filesystem\Adapter\InMemory;

$orm = Manager::filesystem(
    InMemory::emulateFilesystem(),
    Aggregates::of(Types::of(
        Support::class(Username::class, new Username),
    )),
);
```
