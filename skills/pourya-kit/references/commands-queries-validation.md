# Commands, queries and validation

`PouryaKit.Cqrs`, `PouryaKit.Validation`: commands and queries, their handlers, the fixed pipeline and validators. Namespaces `PouryaKit.Cqrs` and `PouryaKit.Validation`; validators are FluentValidation's `AbstractValidator<T>`.

A Feature with a command, its validator and its handler, and a query with its handler. Every type is `internal`:

```csharp
using FluentValidation;
using PouryaKit.Cqrs;
using PouryaKit.Domain;
using PouryaKit.Persistence;

// Products/Features/CreateProduct/
internal sealed record CreateProductCommand(Guid ProductId, Guid CategoryId, string Name) : Command<Guid>;

internal sealed class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
        RuleFor(x => x.ProductId).NotEmpty();      // checked before the typed-ID constructor can throw
        RuleFor(x => x.CategoryId).NotEmpty();
        RuleFor(x => x.Name).NotEmpty().MaximumLength(200).WithErrorCode("Product.NameTooLong");
    }
}

internal sealed class CreateProductCommandHandler(IRepository<Product, ProductId> products)
    : CommandHandler<CreateProductCommand, Guid>
{
    public override async Task<Result<Guid>> Handle(CreateProductCommand command, CancellationToken ct)
    {
        var product = Product.Create(new ProductId(command.ProductId), new CategoryId(command.CategoryId), command.Name);
        if (product.IsFailure) return product.Errors;

        await products.AddAsync(product.Value, ct);
        return product.Value.Id.Value;             // no SaveChanges, no transaction, no Outbox
    }
}

// Products/Features/GetProduct/
internal sealed record GetProductQuery(Guid ProductId) : Query<ProductDto>;

internal sealed class GetProductQueryHandler(IProductReader reader) : QueryHandler<GetProductQuery, ProductDto>
{
    public override async Task<Result<ProductDto>> Handle(GetProductQuery query, CancellationToken ct) =>
        await reader.FindAsync(query.ProductId, ct) is { } product ? product : ProductErrors.NotFound;
}
```

`IRepository<,>` is the Kit's generic repository (`PouryaKit.Persistence`, see [persistence.md](persistence.md)). `IProductReader` stands for the Feature's read access; a query usually reads with Dapper, as the Proving Sample's `GetProduct` does (see [dapper-reads.md](dapper-reads.md)).

## Commands and queries

- Declare each as a sealed record: `Command<TResponse>` for a command with a value, `Command` for one without (its handler derives from `CommandHandler<TCommand>` and returns `Result`), `Query<TResponse>` for a read. Name it after its intent with the `Command` or `Query` suffix.
- Give each exactly one handler class deriving from `CommandHandler<,>`, `CommandHandler<>` or `QueryHandler<,>`. Keep both `internal`; the Kit finds them without `InternalsVisibleTo`. A command or query with no handler or several fails while the Host is composed.
- Commands, queries, handlers and authorizers are never generic type definitions (no `EchoQuery<T>`): registration rejects them while the Host is composed.
- A handler returns a Result and never saves, commits, begins a transaction or touches the Outbox: the Unit of Work does all of that once, for the whole Operation. This holds for every kind of handler, and a structure test rejects a Module whose code does any of it ([testing.md](testing.md#structure-tests)).
- A query reads. Its handler is [code that only reads](#code-that-only-reads). It starts no Unit of Work of its own, but a query nested inside a command runs inside the command's active Unit of Work.
- A Domain Event Handler (`IDomainEventHandler<TDomainEvent>`, also in `PouryaKit.Cqrs`) is the third kind of handler. The Unit of Work runs it, so it is taught in [persistence.md](persistence.md).

## The pipeline

- Every command and query passes the same fixed steps: logging, the optional authorization hook, optional validation, then the handler; a command's handler runs inside the Unit of Work. The logging step opens a log scope with the command's or query's type name as `Operation`, the Operation's `OperationId` and `CorrelationId`, and its `Module`, so every entry logged inside it, your handler's included, names the Operation and its chain of work. Never register a pipeline step or a behaviour yourself, and never implement a contract the Kit fills for the pipeline, such as a decorator of `IDispatcher` or your own `IUnitOfWork`, `IValidationStep<T>`, `IIdempotencyStore`, `IQueryCache` or an Outbox seam: a structure test rejects an application type that does ([testing.md](testing.md#structure-tests)).
- Send from outside the Module through its Gateway, with an Operation Context (see [modules.md](modules.md)). Inside a handler, send a nested command with an injected `IDispatcher`: `await dispatcher.SendAsync(new ReserveStockCommand(id), ct)`. It joins the outer Unit of Work, so the Operation commits once, and it reads the same Operation Context.
- A nested command that fails, in its handler, its validation or its authorization, or by throwing, rolls the whole Operation back, even if your handler ignores its Result. The Operation then fails with that nested command's Errors. Pass a failure on with `return result.Errors;` instead of ignoring it.
- A failed nested query that leaves tracked state unchanged rolls nothing back; act on its Result as you need.
- A query that implements `ICacheableQuery` may be answered from the query cache after its checks, in a Host that calls `AddPouryaKitCaching`, when it is the Operation's outermost query (nested in another query, in the checks or in a write Unit of Work it runs uncached); a command declares the cache tags it invalidates with `IInvalidatesCache`, evicted once its Operation commits ([caching.md](caching.md)).
- One scope holds one Operation. Never send a second top-level command in a scope that already ran one, and never retry a failed command in the same scope: send it again through the Gateway, which opens a new scope for each call.

## Request idempotency

A command a client may send again, after a response it never received, opts in with `[Idempotent]`, as the Proving Sample's `CreateProduct` and `ImportProducts` do:

```csharp
using PouryaKit.Cqrs;

[Idempotent]
internal sealed record CreateProductCommand(Guid ProductId, Guid CategoryId, string Name) : Command<Guid>;
```

- It takes effect when the command is the outermost of an Operation whose `OperationContext.RequestId` is set: over HTTP from the `Idempotency-Key` header of an endpoint that requires one ([web.md](web.md#a-repeatable-write)), from any other Gateway caller with `new OperationContext(actor, tenant) { RequestId = requestId }`. Without a request id the command runs as any other. `Continue()` leaves the request id out, so work the Operation causes never replays.
- Inside its Unit of Work, before the handler runs, the Kit reserves the key: the tenant, the Actor, the command, as its type's full name and its assembly's simple name, and the request id. A tenant is never blank, so an Operation without one has exactly one key per Actor, command and request id. A repeat with the same key replays the first success, an equal Result with the same value, without running the handler, so nothing is written, saved or published again. The key alone decides. The Kit compares no body, so a repeat with the same key and another body, such as a `CreateProductCommand` with another name, also replays the first success. A client gives each distinct request its own key. A concurrent repeat waits until the first Operation ends, then replays its success, or runs when it rolled back.
- Only a success is kept. A failed Result, a nested command or Domain Event Handler that fails, an exception or a refused save rolls the key back with everything else, so a repeat runs again.
- The authorization hook and validation run before the replay: a repeat that now fails them answers that failure, not the first success.
- The value is kept as JSON in the Kit's representation and read back as the command's response type, which must round-trip through it. The Kit records the value when the handler returns, before the Domain Event drain. Do not change the returned object afterwards: the first caller receives that object, so a change a Domain Event Handler makes to it reaches that caller and never a replay. Renaming the command's type, or the assembly that holds it, starts new keys.
- A nested command never reserves, even one that opts in: it joins the outer Operation. A request id reaching an outermost command that does not opt in, or a Host whose persistence keeps no keys, throws `InvalidOperationException` at dispatch.
- Feature code never calls `IIdempotencyStore`, the persistence package's seam, and an application never implements it: structure tests reject a Module or Host that does either ([testing.md](testing.md#structure-tests)).
- The keys live in a table of the Module's own schema, which a migration creates ([persistence.md](persistence.md#migrations)). A key replays for its retention, 24 hours by default; after that the Host's cleanup removes it and a repeat runs as a new request ([persistence.md](persistence.md#idempotency-key-retention)).

## Code that only reads

Some code runs inside an Operation but may change nothing, because its outcome cannot roll the Operation back or because it runs after the Operation's last change:

- a query's handler, and an authorizer or a validator, at any nesting depth, including those of a query or command nested in a command's handler;
- a step of the Unit of Work that only reads, which the Kit opens around application code it runs after the last change: a Snapshot mapper ([messaging.md](messaging.md#writing-a-snapshot-mapper)).

Such code may run queries. The Kit refuses its write entry points with an `InvalidOperationException` before they write; left uncaught, the exception fails the Operation and nothing commits:

- a command sent from it: `A query handler sent a command.`, `A command was sent from an authorizer or a validator.`, or `A command was sent from a Snapshot mapper, a step that only reads.`; a refused command still marks the Operation rollback-only;
- the repository's `AddAsync` and `Remove` ([persistence.md](persistence.md#the-repository)), `IUnitOfWorkConnection.Join()` ([dapper-reads.md](dapper-reads.md#writing-with-plain-sql-in-the-unit-of-work)) and `IIntegrationEventPublisher.Publish` ([messaging.md](messaging.md#publishing-from-a-notification-handler)), each with a message that begins `Code that only reads, such as a query, an authorizer, a validator or a step of the Unit of Work that only reads, ...`.

A command's handler, a Domain Event Handler, and a Notification or Integration Event Handler, whose delivery runs as a command, may change state.

Being allowed to change state is not enough to write: only a Unit of Work's commit saves a change, and one runs only for a command. A scope that sends no command, such as an endpoint's, a hosted service's or one your code opened, is not code that only reads, yet nothing would commit what it wrote. So the writes a commit must save are refused there too, with an `InvalidOperationException` that names who may write: the repository's `AddAsync` and `Remove` (`... was asked to write while no Unit of Work is active, so no commit would save the change and it would be lost.`), `IDelayedCommands.Schedule` ([messaging.md](messaging.md#delayed-commands)) and `IUnitOfWorkConnection.Join()`. The Kit cannot refuse a change made directly to an aggregate loaded outside a command: in a scope that sends none, no commit saves it and nothing says so, and where the scope then sends a command that succeeds, that command's commit saves the change with its own ([persistence.md](persistence.md#the-repository)). Send a command to write.

With EF persistence, the Operation also compares tracked state when the whole query, the authorization and validation step, or a default `OnlyReading(step)` ends, including after a failed Result or an exception. It captures the current values and entry states, so a command's legitimate pending changes are its baseline. New entities loaded by a query are captured when they begin tracking, so an innocent additional read is permitted. The comparison includes owned and child entities, mutable values through EF's value comparers, complex properties and collections recursively, and pending Domain Events by identity and order. A difference marks the Operation rollback-only before it throws. Catching that exception cannot permit the outer command to commit. An otherwise successful command fails with `OperationErrors.ReadingChangedState` (`PouryaKit.Cqrs`, kind `Unexpected`, code `Operation.ReadingChangedState`), unless an earlier failure already made the Operation rollback-only. Snapshot mapper scopes retain the write guards but share one tracked-state comparison after the whole batch, as described in [the mapper rules](messaging.md#the-rules-of-a-mapper-and-a-key).

The comparison sees state at the boundary, so it cannot detect a transient change restored before the boundary ends. `IReadingStateGuard` and `TrackedState` are Kit seams, never Feature APIs. The DI registration attaches the current Module's Unit of Work guard lazily. Queries without a persistence package need no Unit of Work.

## Validation

- Write one `AbstractValidator<T>` per command or query that needs input checks, beside it in the Feature, and keep it `internal`. Several validators for one type all run. `AddPouryaKitValidation` finds them with FluentValidation's own `AssemblyScanner`, which registers concrete validator classes only: an abstract base validator that shares rules is skipped on purpose, and so is an open generic validator, which nothing could instantiate; close it or derive a concrete one. This is the library's design, and the Kit keeps it rather than refusing such a class as it refuses a generic handler definition (a validator is optional, a handler is not).
- Every failure of every validator comes back as one failed Result before the handler runs; nothing throws. Each failure is an Error of kind `Validation`, with FluentValidation's error code as `Code` (set your own with `WithErrorCode`; a failure with none gets `ValidationErrorCodes.Failed` of `PouryaKit.Validation`, `Validation.Failed`) and the property name as `Target`.
- Validate input shape here. A business rule that depends on state belongs in the aggregate, through `CheckRule`.
- Without `PouryaKit.Validation`, the validation step does nothing.

## Paging

A query that reads a list answers one Page of it. It carries a `PageRequest` and answers `Page<T>`; a static `PageDeclaration` beside it says what a caller may sort and filter by, and its validator checks the request against it. The Proving Sample's `ListProducts`:

```csharp
using FluentValidation;
using PouryaKit.Cqrs;
using PouryaKit.Validation;

// Products/Features/ListProducts/
internal sealed record ListProductsQuery(PageRequest Page) : Query<Page<ProductListItemDto>>
{
    public static readonly PageDeclaration Paging = new(
        sorts: ["name", "created"],                // the first is the default
        filters:
        [
            FilterField.Of<string>("name", FilterOperator.Equal, FilterOperator.Contains),
            FilterField.Of<Guid>("category", FilterOperator.Equal),
            FilterField.Of<DateTimeOffset>("created", FilterOperator.GreaterThanOrEqual),
        ],
        maxSize: 50
    );
}

internal sealed class ListProductsQueryValidator : AbstractValidator<ListProductsQuery>
{
    public ListProductsQueryValidator() =>
        RuleFor(query => query.Page).Cascade(CascadeMode.Stop).NotNull().AllowedBy(ListProductsQuery.Paging);
}
```

- `PageRequest` has a 1-based `Number` (default 1), a `Size` (default 20), an optional `Sort` key (null sorts by the declaration's first), a `Direction` (`SortDirection.Ascending` by default) and `Filters`, each a `PageFilter(field, FilterOperator, value)`. A value is of the field's declared type, or text that parses to it, as HTTP sends it.
- `Page<T>` has `Items`, `Number`, `Size`, the `Total` of the whole filtered selection, and `TotalPages`, `HasNext` and `HasPrevious`. A Page beyond the last has no items and the true total.
- Declare sort keys and filter fields by name only: a letter followed by letters and digits. A field's value type is string, Guid, int, long, decimal, bool or DateTimeOffset (text without an offset is UTC, and every time is read as UTC, the one offset Npgsql sends, so the Dapper and EF Core reads bind the same instant). `contains` and `startswith` take text only; `gt`, `gte`, `lt` and `lte` take neither text, Guid nor bool. The declaration throws for anything else when the query's type is first used.
- Check the request in the query's validator with `AllowedBy(declaration)` of `PouryaKit.Validation`, as above, so it runs in the pipeline's one validation step and its Errors come back with the others: the rule adds each Error `PageDeclaration.Validate` returns as a failure with its message, its target as the property name and its code, so the validation step returns exactly those Errors. A null request adds nothing, so keep the validator's own `NotNull`. The declaration's Error codes, the constants of `PageErrorCodes` (`PouryaKit.Cqrs`), are `Page.NumberOutOfRange`, `Page.SizeOutOfRange` (1 to `maxSize`, default 100; a `maxSize` below `PageRequest.DefaultSize`, 20, throws when declared, since it would refuse every request that states no size), `Page.SortUnknown`, `Page.DirectionUnknown`, `Page.FilterUnknown`, `Page.FilterOperatorUnsupported`, `Page.FilterValueInvalid` and `Page.FilterRepeated` (one field and operator given twice), with the targets `Number`, `Size`, `Sort`, `Direction` and `Filters[i]`. No message repeats what the caller sent.
- The handler reads the Page through Dapper ([dapper-reads.md](dapper-reads.md#reading-a-page)) or an EF Core repository ([persistence.md](persistence.md#reading-a-page)). Both map the declared names to the Feature's own columns or expressions, so no text a caller sends becomes SQL, and both throw before any SQL runs for a request that fails `Validate`: a forgotten validator is a bug that fails loudly, not an open door.
- An endpoint binds the request from the query string with `HttpPageRequest` ([web.md](web.md#a-page-over-http)).

## Authorization hook

- The Actor is authenticated and authorized at the external entry point. Add a per-operation rule only when one command or query needs it: an `internal` class implementing `IAuthorizer<TOperation>` for that one concrete command or query, returning `Result.Success()` or a `Forbidden` Error. It runs before validation. Without one, the step does nothing. An authorizer for a base type, a marker interface or an open type fails while the Host is composed, because it would never run.
- An authorizer reads the Actor and tenant from an injected `OperationContext` (see [modules.md](modules.md)). It runs for every Operation of its command or query, including one that continues another Module's Operation. `context.IsTrustedContinuation` is true for a context the Kit restored for a delivery, and for one that `Continue()` gave inside the running Operation it continues, after that Operation was authorized. `Continue()` inside an authorizer or a validator, or on a context that is not running, only records the causation. Recognise a trusted continuation by that flag, never by `CausationId`, which any code can set.
- An authorizer or a validator may run a query (a permission or existence lookup), but it is [code that only reads](#code-that-only-reads): it runs before its command's handler, at every nesting depth, so a command it sends throws.

## Registration

```csharp
module.AddPouryaKitCqrs();        // handlers, authorizers, Domain Event Handlers
module.AddPouryaKitValidation();  // validators
```

Make both calls on the Module's builder in its Composition Root (see [modules.md](modules.md#the-composition-root)): with no assembly, each scans the assembly of the Module's definition, where its Features live; name assemblies to scan others instead, in the order an Operation's authorizers run in. An application without Modules calls them on its own services and names the assembly; there `AddPouryaKitCqrs()` with none registers the dispatcher alone. The Unit of Work comes from the Module's persistence package, and logging from the Host.

## Testing

- Test a Feature by sending it through its Module's Gateway against the hosted application (see [testing.md](testing.md)). Assert on the returned Result's `Errors`, never on exceptions, for business and validation failures.
