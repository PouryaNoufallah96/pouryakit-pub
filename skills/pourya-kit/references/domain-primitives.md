# Domain primitives

`PouryaKit.Domain`: Result and Error, typed IDs, entities, aggregates and their audit and soft-delete contracts, Domain Events and business rules. One namespace, `PouryaKit.Domain`; no dependencies.

- [Result and Error](#result-and-error)
- [Typed IDs](#typed-ids)
- [Aggregates and entities](#aggregates-and-entities)
- [Domain Events](#domain-events)
- [Business rules](#business-rules)
- [The clock](#the-clock)
- [Testing](#testing)

An aggregate with a typed ID that checks a business rule, raises a Domain Event and returns a Result:

```csharp
using PouryaKit.Domain;

internal sealed record ProductId(Guid Value) : TypedId<Guid>(Value);

internal sealed record CategoryId(Guid Value) : TypedId<Guid>(Value);

internal sealed record ProductCreatedDomainEvent(ProductId ProductId, CategoryId CategoryId, string Name) : DomainEvent;

internal static class ProductErrors
{
    public static readonly Error NameRequired = Error.Validation("Product.NameRequired", "A product needs a name.");
    public static readonly Error NotFound = Error.NotFound("Product.NotFound", "The product does not exist.");
}

internal sealed class ProductNameMustNotBeBlank(string name) : IBusinessRule
{
    public Error Error => ProductErrors.NameRequired;

    public bool IsBroken() => string.IsNullOrWhiteSpace(name);
}

internal sealed class Product : AggregateRoot<ProductId>
{
    public string Name { get; private set; } = "";

    public CategoryId CategoryId { get; private set; } = null!;

    public static Result<Product> Create(ProductId id, CategoryId categoryId, string name)
    {
        var rule = CheckRule(new ProductNameMustNotBeBlank(name));
        if (rule.IsFailure) return rule.Errors;

        var product = new Product { Id = id, CategoryId = categoryId, Name = name.Trim() };
        product.Raise(new ProductCreatedDomainEvent(id, categoryId, product.Name));
        return product;
    }
}

// Elsewhere: var created = Product.Create(TypedId.New<ProductId>(), new CategoryId(command.CategoryId), command.Name);
```

## Result and Error

- Return a value or an Error directly; each converts to a Result. `Result.Success()` is the success without a value. When the value's static type is an interface (such as `IReadOnlyList<T>`), C# applies no implicit conversion, so return `Result.Success(value)`.
- Pass a failure on with `return result.Errors;`; it converts to a failed Result of any type and keeps every Error.
- Check `IsSuccess` or `IsFailure` before reading `Value`; reading it from a failure throws. No conversion turns a Result into its value, and a null value throws instead of becoming a success.
- Give each Error a stable code (`Product.NameRequired`), a message and a kind through its factory: `Validation`, `NotFound`, `Conflict`, `Forbidden` or `Unexpected`. Never an HTTP status; the Web package maps kinds to statuses ([web.md](web.md)). A validation Error may name the input it is about as its target: `Error.Validation(code, message, "Name")`.
- Declare a Business Area's Errors once, as static fields.

## Typed IDs

- Declare one per aggregate as a sealed record over `Guid`, `long`, `int` or `string`, as above. Two IDs are equal only when type and value are equal, so IDs of different aggregates cannot be mixed up.
- Create a new Guid ID with `TypedId.New<ProductId>()`; it is time-ordered (version 7). Wrap a value from input with `new ProductId(value)`.
- The constructor throws on an empty Guid or a blank string: that is a bug, so validate input before it reaches the constructor.
- Write no converter and set up no JSON options of your own. `options.UsePouryaKitJson()` is the Kit's one JSON representation: camelCase names and every typed ID as its bare value (`"0199b2a4-..."`, `42`, `"LAMP-01"`), never `{"value":...}`. Names are read ignoring case, and a body that names one of its type's members twice, `price` and `Price` or `price` twice, fails to read with a `JsonException` instead of keeping the last value. `AddPouryaKitWeb` applies it to request and response bodies ([web.md](web.md)), and `UsePouryaKitWolverine` applies `UsePouryaKitWireJson()`, the same representation read strictly, to stored and delivered messages ([messaging.md](messaging.md)), so an ID reads the same in both. Where code needs options of its own, create them with `new JsonSerializerOptions().UsePouryaKitJson()`; a structure test rejects setting an option anywhere else, and the typed-ID converter is internal to `PouryaKit.Domain`, so no other code can add it. The converter also handles typed IDs as dictionary keys, and reports an empty ID in JSON as a `JsonException`. EF Core needs nothing either: `PouryaKitDbContext` stores every typed ID as its value (see [persistence.md](persistence.md)). Nor does Dapper: `AddPouryaKitDapper` makes every typed ID a parameter and a result (see [dapper-reads.md](dapper-reads.md)).

## Aggregates and entities

- Raise a Domain Event with `Raise` from inside the aggregate. Never clear Domain Events and never set `Version`: the Unit of Work owns both. It takes and clears Domain Events when it runs their handlers (see [persistence.md](persistence.md)), and a structure test rejects a Module that clears them itself ([testing.md](testing.md#structure-tests)).
- `Version` is 0 until the aggregate is saved, 1 after its first commit, and exactly one more for each later Operation that changes the aggregate or anything it owns. It is the concurrency token that turns a concurrent change into a conflict Error ([persistence.md](persistence.md#versions-and-conflicts)). Raising an event does not change the version.
- Opt an aggregate or entity into audit with `IAuditable` and into soft delete with `ISoftDeletable`. Declare each of their properties with a private setter and never set it: the Unit of Work fills them before it saves ([persistence.md](persistence.md#audit-and-soft-delete)). Mark an aggregate as one tenant's with `ITenantScoped`, whose `Tenant` the aggregate sets itself, from the tenant its handler takes from the Operation Context ([multi-tenancy.md](multi-tenancy.md#a-tenant-scoped-aggregate)). Removing a soft-deletable aggregate raises no Domain Event of its own; when other aggregates care, give it a domain method that raises one, such as the Proving Sample's `Product.Remove()`.
- Entities are equal when type and ID are equal; do not override equality.
- Equality and hashing use the current ID. Do not change an entity's ID while it is in a hash-based collection using entity equality. For database-generated IDs, add the entity after its ID is assigned, or use `ReferenceEqualityComparer.Instance` when the collection tracks object instances across persistence. With reference equality, different instances remain distinct even when their IDs are equal.

## Domain Events

- Declare each as a sealed record deriving from `DomainEvent`, named in the past tense with the `DomainEvent` suffix. Its `Id` and UTC `OccurredOn` are assigned once when it is created and survive serialization; never set them yourself.
- Carry typed IDs and values, never the aggregate itself or its version.
- Serialize a Domain Event by its runtime type (`JsonSerializer.Serialize(e, e.GetType(), options)`); serializing it as `DomainEvent` drops the derived members.
- A payload without `Id` deserializes with a new identity, so a stored or sent event must always include its `Id`.
- React to one inside the Module with a Domain Event Handler; see [persistence.md](persistence.md).

## Business rules

- Implement `IBusinessRule` with `IsBroken()` and the Error it returns. `CheckRule(rule)` in an entity or aggregate returns a Result; a broken rule is a failed Result, never an exception.

## The clock

- A Domain Event takes its `OccurredOn` from the running Operation's clock: the dispatcher binds the Host's registered `TimeProvider` as `DomainClock` for each Operation. Outside an Operation, `DomainClock` is the system clock.
- To change the time Operations see, the Host registers its own `TimeProvider` (`builder.Services.AddSingleton<TimeProvider>(time)`); `AddPouryaKitCqrs` adds `TimeProvider.System` only when none is registered.
- A business rule that compares with "now" takes the time as a parameter. The handler injects `TimeProvider` and passes `timeProvider.GetUtcNow()` in. Feature code takes time only from that `TimeProvider` and never binds `DomainClock`.

```csharp
internal sealed class SaleMustBeOpen(DateTimeOffset endsOn, DateTimeOffset now) : IBusinessRule
{
    public Error Error => Error.Conflict("Sale.Ended", "The sale has ended.");

    public bool IsBroken() => now >= endsOn;
}

// In the aggregate: public Result Buy(DateTimeOffset now) => CheckRule(new SaleMustBeOpen(EndsOn, now));
// In its handler, with TimeProvider timeProvider injected: var bought = sale.Buy(timeProvider.GetUtcNow());
```

## Testing

- Test an aggregate through its public methods and assert on `DomainEvents` and on the returned Result's `Errors`.
- To fix the time of Domain Events an aggregate raises in a test, create a `TestClock` (`PouryaKit.Testing`) with `using` inside the test method itself: `using var clock = new TestClock(monday);`, then `clock.Time.Advance(...)`. A binding made in a constructor or fixture is not relied on to reach the test.
