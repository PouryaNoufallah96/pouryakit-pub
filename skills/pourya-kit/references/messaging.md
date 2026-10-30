# Messaging

`PouryaKit.Messaging`, `PouryaKit.Messaging.Wolverine`, `PouryaKit.Messaging.Wolverine.EFCore`: Notification Handlers that run after the commit, the Integration Events they publish for other Modules, and the Snapshots of a Module's aggregates that the Unit of Work publishes for other Modules, and the commands a Module schedules to run later or declares as recurring work, each delivery durable, retried and recorded on its own, on Wolverine with its message store in PostgreSQL. Feature code uses `PouryaKit.Messaging`, and `IDelayedCommands` of `PouryaKit.Messaging.Wolverine` to schedule a command. The Composition Root uses `PouryaKit.Messaging.Wolverine`; the Host and the Receipt and Snapshot version migrations use `PouryaKit.Messaging.Wolverine.EFCore`, which brings the other two, so a Module that declares Notification Handlers, publishes Snapshots or subscribes Integration Event Handlers references it. A Contracts project references nothing from the Kit.

- [Notification Handler or Domain Event Handler](#notification-handler-or-domain-event-handler)
- [Writing a Notification Handler](#writing-a-notification-handler)
- [Declaring handlers and running deliveries](#declaring-handlers-and-running-deliveries)
- [Integration Events between Modules](#integration-events-between-modules)
- [The Integration Event](#the-integration-event)
- [Declaring what a Module publishes and subscribes](#declaring-what-a-module-publishes-and-subscribes)
- [Publishing from a Notification Handler](#publishing-from-a-notification-handler)
- [Writing an Integration Event Handler](#writing-an-integration-event-handler)
- [Snapshots between Modules](#snapshots-between-modules)
- [The Snapshot](#the-snapshot)
- [Writing a Snapshot mapper](#writing-a-snapshot-mapper)
- [Declaring a Snapshot](#declaring-a-snapshot)
- [Keeping a copy](#keeping-a-copy)
- [What the Kit guarantees for a Snapshot](#what-the-kit-guarantees-for-a-snapshot)
- [The rules of a mapper and a key](#the-rules-of-a-mapper-and-a-key)
- [Delayed commands](#delayed-commands)
- [Recurring commands](#recurring-commands)
- [What the Kit guarantees](#what-the-kit-guarantees)
- [Receipts](#receipts)
- [Names](#names)
- [Retries](#retries)
- [Dead letters](#dead-letters)
- [The message store](#the-message-store)
- [The stale-time rule](#the-stale-time-rule)
- [The Host's checks](#the-hosts-checks)

A Notification Handler and the port it reaches outside the Module through, from the Proving Sample's Products Business Area:

```csharp
using Catalog.Contracts.Products.Realtime;
using PouryaKit.Domain;
using PouryaKit.Messaging;
using PouryaKit.Realtime;

// Products/Events/Notifications/: runs after the product is committed, never inside its transaction, because an
// announcement of a product that was then rolled back could not be taken back. It also pushes the new product to the
// browsers of its tenant when this delivery commits (realtime.md).
internal sealed class AnnounceNewProductNotificationHandler(INewProductAnnouncer announcer, IRealtimePublisher realtime)
    : INotificationHandler<ProductCreatedDomainEvent>
{
    public async Task<Result> Handle(ProductCreatedDomainEvent domainEvent, CancellationToken cancellationToken)
    {
        await announcer.AnnounceAsync(
            new NewProductAnnouncement(domainEvent.ProductId.Value, domainEvent.Name),
            cancellationToken
        );
        return await realtime.PublishAsync(
            NewProduct.Channel,
            new NewProduct(domainEvent.ProductId.Value, domainEvent.Name),
            cancellationToken
        );
    }
}

// The port: public, because the Host implements it. The Module decides what is said; the Host decides how it is sent.
public interface INewProductAnnouncer
{
    Task AnnounceAsync(NewProductAnnouncement announcement, CancellationToken cancellationToken);
}

public sealed record NewProductAnnouncement(Guid ProductId, string Name);

// In the Composition Root: module.AddPouryaKitNotificationHandlers();
```

## Notification Handler or Domain Event Handler

Both react to a Domain Event the Module raised. Choose by when the reaction may happen:

- A Domain Event Handler runs inside the Operation's Unit of Work, before the commit, and its changes commit with the change or not at all. Use it to keep the Module's own aggregates consistent with each other ([persistence.md](persistence.md#domain-event-handlers)).
- A Notification Handler runs after the commit, in an Operation of its own. Use it for anything that reaches outside the Module's own data and cannot be taken back, such as sending an email, pushing to a client, calling another system or publishing an Integration Event for other Modules. Its failure is retried; it never rolls back the change, which is already committed.
- One Domain Event may have both, as the sample's `ProductCreatedDomainEvent` does: `CountProductInCategoryDomainEventHandler` counts the product in the same commit, and afterwards `AnnounceNewProductNotificationHandler` announces it and `PublishProductCreatedNotificationHandler` tells other Modules about it. A Domain Event needs no Domain Event Handler to be delivered as a Notification.

## Writing a Notification Handler

- Implement `INotificationHandler<TDomainEvent>` (`PouryaKit.Messaging`) for one concrete Domain Event type, in an `internal sealed` class of the Business Area that raises it, under `Events/Notifications/`. The handler and its Domain Event stay `internal`. It runs only for a Domain Event of exactly its type, never for one derived from it.
- Return a Result. A failed Result is a failure like an exception: the delivery is retried ([Retries](#retries)), then set aside as a dead letter that names each Error ([Dead letters](#dead-letters)).
- Reach outside the Module through a port: an interface the Module declares beside the handler, public so the Host can implement it, as `INewProductAnnouncer` above. The Host registers the implementation (below). The handler never names a transport, an SMTP client or another Module's types.
- Inject what a command handler may: the `OperationContext` (a continuation of the raising Operation), the Module's repositories, `IDispatcher`, and `IIntegrationEventPublisher` to tell other Modules ([Publishing from a Notification Handler](#publishing-from-a-notification-handler)). A change made through the Module's repositories commits in the handler's own Unit of Work, with its Receipt, and a Domain Event it raises is handled as in any Operation. A change that must commit with the change that raised the Domain Event belongs in a Domain Event Handler instead.
- Never save, commit, begin a transaction, touch the Outbox or reference a Wolverine type: the delivery and its Unit of Work own all of that. A structure test rejects a Module or a Contracts project whose code names any Wolverine or JasperFx type, and a Module DbContext that maps the message store's envelope tables ([testing.md](testing.md#structure-tests)).

## Declaring handlers and running deliveries

- In each Module's Composition Root, call `module.AddPouryaKitNotificationHandlers()` (`PouryaKit.Messaging.Wolverine`). It finds every Notification Handler in the assembly of the Module's definition, internal ones included, subscribes each one to its Domain Event under its subscription name ([Names](#names)), and runs its deliveries in the Module whose builder it is called on, so they reach that Module's Unit of Work ([modules.md](modules.md#the-composition-root)). Handlers that live in other assemblies are named instead, `AddPouryaKitNotificationHandlers(assembly, ...)`. It throws when the assemblies have no Notification Handler. An application without Modules calls it on its own services, naming the assembly; the deliveries then run in no Module. Called on `builder.Services` in a Host of Modules, it fails the Host's build ([The Host's checks](#the-hosts-checks)); a Host's composition that declares a Module's handlers on its behalf names it, `AddPouryaKitNotificationHandlers(typeof(CatalogModule))`.
- That Module needs its own Unit of Work (`UsePouryaKitPostgres`, [persistence.md](persistence.md#postgresql)) and the Receipt table ([Receipts](#receipts)).
- In the Host, call `builder.UsePouryaKitWolverine(connectionName: "app")` (`PouryaKit.Messaging.Wolverine.EFCore`) once; Modules added after it still get their subscriptions. The call throws when the configuration has no such connection string or leaves it blank, and a second call throws Wolverine's `InvalidOperationException`, since `UseWolverine` runs once per service collection. It runs the Host's deliveries, of Notification Handlers and Integration Event Handlers alike, on Wolverine with its message store in the database of that connection string, which must be the one the Modules' DbContexts use: the Outbox is written in their transactions, so a Unit of Work's DbContext on another database fails the Host when it is built, whether `UsePouryaKitPostgres` registered it or the application did and made it a Unit of Work with `AddPouryaKitEfCore<TDbContext>()` ([The Host's checks](#the-hosts-checks)). Wolverine creates its tables in the store schema when the Host starts. Wolverine discovers no handlers of its own. Messages are stored and delivered in the Kit's one JSON representation, the one HTTP bodies use, so a typed ID inside a Domain Event reads the same in its envelope as in a response ([domain-primitives.md](domain-primitives.md#typed-ids)), and read back strictly (`UsePouryaKitWireJson`): a body that lacks a member its type requires, carries null where the type forbids it, or names one of its members twice, is set aside as a dead letter, never handled with a default ([Dead letters](#dead-letters)). Its optional `configure` parameter adds Host-specific Wolverine configuration as Wolverine is registered, after the Kit's settings made there, which it may override, but before the routes, handlers and retry policy, and the message store's durability mode and inbox stale time, which the Kit applies when Wolverine first resolves its options: set those through `PouryaKitMessagingOptions` and `PouryaKitMessageStoreOptions`.
- Wolverine generates its handler code at run time in Development and, in every other environment, loads it from the Host's assembly (`TypeLoadMode.Static`), where `dotnet run -- codegen write` wrote it before the Release build ([templates.md](templates.md#pre-generating-wolverines-code-for-production)). The Kit's own deliveries need no generated code; a Wolverine handler a Host adds through `configure` does, and a Host without it does not start outside Development.
- The Host project, not the Kit, references `WolverineFx.RuntimeCompilation`, in Debug builds only, so a Release publish carries no runtime compiler. `PouryaKit.Testing` references it for test Hosts, which run in Development from any build; no other Kit package does (ADR 0004).

```xml
<!-- ModularMonolith.Host.csproj: Roslyn for Wolverine's run-time code generation, in Debug builds only -->
<ItemGroup Condition="'$(Configuration)' == 'Debug'">
  <PackageReference Include="WolverineFx.RuntimeCompilation" />
</ItemGroup>
```

The Host implements the port and registers it, beside the deliveries:

```csharp
using Catalog.Products.Events.Notifications;
using Microsoft.Extensions.Logging;

// ModularMonolith.Host: announces in the log, where a real application would send it to its customers
public sealed partial class LoggedNewProductAnnouncer(ILogger<LoggedNewProductAnnouncer> logger) : INewProductAnnouncer
{
    public Task AnnounceAsync(NewProductAnnouncement announcement, CancellationToken cancellationToken)
    {
        LogAnnounced(logger, announcement.Name, announcement.ProductId);
        return Task.CompletedTask;
    }

    [LoggerMessage(EventName = "Announced", Level = LogLevel.Information, Message = "New in the catalog: {Name} ({ProductId})")]
    private static partial void LogAnnounced(ILogger logger, string name, Guid productId);
}

// In ModularMonolithHost.CreateBuilder, after AddPouryaKitModules (web.md):
//     builder.UsePouryaKitWolverine(connectionName: "app");
//     builder.Services.AddSingleton<INewProductAnnouncer, LoggedNewProductAnnouncer>();
```

## Integration Events between Modules

Across Modules, a Module tells the others what happened with an Integration Event: use one when another Module reacts to something that happened in this one. When another Module keeps its own copy of this one's data, publish a Snapshot instead ([Snapshots between Modules](#snapshots-between-modules)). The receiving Module applies it later, in an Operation of its own, and the publishing Operation never waits for it. A synchronous question through the owning Module's Contracts ([modules.md](modules.md#asking-another-module-a-question)) is the exception, for an Operation that cannot go on without the answer.

In the Proving Sample, Catalog tells Inventory and Billing about each new product: Inventory opens a stock item for it, and Billing a price list:

1. `CreateProduct` commits the product; its `ProductCreatedDomainEvent` is stored as a Notification in the same commit.
2. After that commit, Catalog's `PublishProductCreatedNotificationHandler` publishes `ProductCreatedV1`, which is stored in Catalog's Outbox in the commit of that delivery, once per subscription.
3. After that commit, Inventory's `OpenStockItemIntegrationEventHandler` opens the stock item in Inventory's Unit of Work, and Billing's `OpenPriceListIntegrationEventHandler` opens the price list in Billing's, each in a delivery of its own.

### The Integration Event

```csharp
// Catalog.Contracts/Products/Events/V1/: public, plain values only, nothing from the Kit
/// <summary>
/// A product was added to the catalog. <paramref name="Id"/> and <paramref name="OccurredOn"/> are the identity and the
/// time of the Domain Event it follows from, the same in every delivery. Catalog publishes it under the wire name
/// <c>catalog.product-created.v1</c>; a change other Modules could not read is a new version, not an edit of this one.
/// </summary>
public sealed record ProductCreatedV1(Guid Id, DateTimeOffset OccurredOn, Guid ProductId, string Name);
```

- Declare it as a `public sealed record` in the publishing Module's Contracts project, under `Events/V1/`, named with its version. It holds plain values, such as `Guid`, strings, numbers, times and records of them, never a typed ID or another of the Module's internal types. It carries no attribute, because the Contracts project references nothing from the Kit: its wire name is declared where the Module publishes it ([Names](#names)).
- Give it the Domain Event's `Id` and `OccurredOn`, as `Id` and `OccurredOn`, so the receiver sees the identity and time the publisher saw.
- A change that receivers could not read is a new type with its own version and wire name, never an edit of the old one.

### Declaring what a Module publishes and subscribes

The receiving Module's Composition Root, from the Proving Sample:

```csharp
using Catalog.Contracts.Products.Events.V1;
using Inventory.CatalogProducts.Features.CopyProduct;
using Inventory.Infrastructure.Data.Context;
using Inventory.StockItems.Features.OpenStockItem;
using PouryaKit.Cqrs;
using PouryaKit.Messaging.Wolverine;
using PouryaKit.Modularity;
using PouryaKit.Persistence.Postgres;

// Inventory/InventoryModule.cs
public sealed class InventoryModule : IModuleDefinition
{
    public void Configure(ModuleBuilder module)
    {
        module.UsePouryaKitPostgres<InventoryDbContext>(connectionName: "app", schema: "inventory");
        module.AddPouryaKitCqrs();
        module.Subscribe<ProductCreatedV1, OpenStockItemIntegrationEventHandler>("inventory.product-created");
        // Catalog's Snapshot of each product, for Inventory's copy (Snapshots between Modules, below).
        module.Subscribe<ProductSnapshotV1, CopyProductIntegrationEventHandler>("inventory.catalog-products");
    }
}

// In Catalog's Composition Root, the publishing side (modules.md):
//     module.Publish<ProductCreatedV1>("catalog.product-created.v1");
//     module.PublishSnapshot<Product, ProductSnapshotV1, ProductSnapshotMapper>("catalog.product-snapshot.v1");
// In Billing's Composition Root, a second subscription to the same Integration Event, under its own name:
//     module.Subscribe<ProductCreatedV1, OpenPriceListIntegrationEventHandler>("billing.product-created");
```

- The publishing Module declares each Integration Event it publishes with `module.Publish<TIntegrationEvent>(wireName)` (`PouryaKit.Messaging.Wolverine`) on its builder, which names the Module ([modules.md](modules.md#the-composition-root)). One Module publishes a type, under one wire name.
- The receiving Module subscribes each Integration Event Handler with `module.Subscribe<TIntegrationEvent, THandler>(subscriptionName)`. The subscription name is declared, not derived, and the handler's deliveries run in the Module whose builder declares it, so they reach its Unit of Work. `Subscribe` registers the handler, which may be `internal`.
- A Host's composition that declares for a Module, such as a test's extra subscription, names it: `builder.Services.Subscribe<TIntegrationEvent, THandler>(subscriptionName, typeof(InventoryModule))`; every Module-aware declaration has that form. Made on `builder.Services` without the Module, in a Host of Modules, a declaration belongs to no Module and fails the Host's build, as does one naming a Module the Host never added ([The Host's checks](#the-hosts-checks)).
- Several Modules may subscribe to one Integration Event, each with its own `Subscribe` and subscription name, as Inventory and Billing do to `ProductCreatedV1`. The publisher changes nothing for a new subscription: each gets its own delivery, and one that fails never makes another run again ([What the Kit guarantees](#what-the-kit-guarantees)).
- The receiving Module needs its own Unit of Work (`UsePouryaKitPostgres`) and the Receipt table in its schema ([Receipts](#receipts)).
- The receiving Module's project references the publishing Module's Contracts project and never the publishing Module: Inventory references `Catalog.Contracts` only. Neither Module references a Wolverine type.
- The Host adds both Modules and calls `UsePouryaKitWolverine(connectionName: "app")` once, as for Notification Handlers ([web.md](web.md#the-host)).
- When the publishing Module runs in another Host, the receiving Host's composition declares its wire name with `Receive` or `ReceiveSnapshot` instead, and both Hosts join a broker ([rabbitmq.md](rabbitmq.md)). The Modules do not change.
- An application without Modules calls `Publish` and `Subscribe` on its own services and leaves the Module out.

### Publishing from a Notification Handler

```csharp
using Catalog.Contracts.Products.Events.V1;
using PouryaKit.Domain;
using PouryaKit.Messaging;

// Catalog/Products/Events/Notifications/: tells other Modules about the new product once it is committed. The
// Integration Event carries the Domain Event's identity and time, so every receiver recognises a redelivery.
internal sealed class PublishProductCreatedNotificationHandler(IIntegrationEventPublisher events)
    : INotificationHandler<ProductCreatedDomainEvent>
{
    public Task<Result> Handle(ProductCreatedDomainEvent domainEvent, CancellationToken cancellationToken)
    {
        events.Publish(
            domainEvent.Id,
            new ProductCreatedV1(domainEvent.Id, domainEvent.OccurredOn, domainEvent.ProductId.Value, domainEvent.Name)
        );
        return Task.FromResult(Result.Success());
    }
}
```

- Publish from a Notification Handler only: inject `IIntegrationEventPublisher` (`PouryaKit.Messaging`) and call `Publish(domainEvent.Id, integrationEvent)` with the `Id` of the Domain Event the handler is handling. Publishing sends nothing: the delivery's Unit of Work writes the Integration Event to the Module's Outbox when it commits, and it is sent after that commit.
- Give publishing a Notification Handler of its own, as the sample keeps it apart from the announcement: each handler is retried on its own, so a failing announcement neither holds back nor repeats the Integration Event.
- `Publish` throws `InvalidOperationException` when it is called:

  - outside a Notification Handler's delivery, such as in a command handler or in an Integration Event Handler;
  - with an identity other than the `Id` of the Domain Event being delivered;
  - with a type the running Module did not declare with `Publish`, another Module's Integration Event included;
  - with a type declared as a Snapshot: the Unit of Work publishes it, and nothing publishes one by hand ([Snapshots between Modules](#snapshots-between-modules));
  - a second time with the same type in one Notification Handler's delivery, because both would carry one identity and every receiver would apply only the first;
  - in [code that only reads](commands-queries-validation.md#code-that-only-reads), even a query or validator the Notification Handler runs.

  In a Notification Handler, the exception fails the delivery, which is retried and then set aside as a dead letter.
- One Domain Event yields at most one Integration Event of each type, from one Notification Handler. When two Notification Handlers of one Domain Event publish the same type, the second delivery to commit fails, and keeps failing until it is set aside as a dead letter: the Outbox records each publication in the publishing Module's Receipt table, keyed by the wire name and the Domain Event's `Id`.
- An Integration Event that no Module of the Host subscribes to is not stored: it has no queue to go to.
- Never publish a Domain Event: it stays inside its Module, and `Publish<T>` refuses one.

### Writing an Integration Event Handler

```csharp
using Catalog.Contracts.Products.Events.V1;
using Inventory.StockItems.Entities;
using Inventory.StockItems.ValueObjects;
using PouryaKit.Cqrs;
using PouryaKit.Domain;
using PouryaKit.Messaging;
using PouryaKit.Persistence;

// Inventory/StockItems/Features/OpenStockItem/: only the effect, for the tenant the event was delivered for. The Kit
// records the Receipt, commits and acknowledges, so a redelivered or concurrently duplicated ProductCreatedV1 opens one
// stock item.
internal sealed class OpenStockItemIntegrationEventHandler(IRepository<StockItem, StockItemId> stockItems, OperationContext context)
    : IIntegrationEventHandler<ProductCreatedV1>
{
    public async Task<Result> Handle(ProductCreatedV1 integrationEvent, CancellationToken cancellationToken)
    {
        await stockItems.AddAsync(StockItem.Open(integrationEvent.ProductId, integrationEvent.OccurredOn, context.Tenant), cancellationToken);
        return Result.Success();
    }
}
```

- Implement `IIntegrationEventHandler<TIntegrationEvent>` (`PouryaKit.Messaging`) in an `internal sealed` class of the receiving Module, beside the Feature it serves, and subscribe it in the Composition Root (above).
- Hold only the business effect, through the Module's repositories, and return a Result. A failed Result is a failure like an exception: the delivery is retried ([Retries](#retries)), then set aside as a dead letter of its subscription ([Dead letters](#dead-letters)), and nothing the attempt changed commits, its Receipt included.
- Inject what a command handler may: the `OperationContext`, the Module's repositories, `IDispatcher`. Read when the fact happened from the Integration Event, as `StockItem.Open` takes `OccurredOn`; the Operation's own clock gives the time of the delivery. Read the tenant the delivery carries from the `OperationContext`, as `StockItem.Open` takes `context.Tenant` in a Module that is not tenant-scoped ([multi-tenancy.md](multi-tenancy.md#deliveries)).
- Never save, commit, begin a transaction, record a Receipt, acknowledge or reference a Wolverine type, and never publish: an Integration Event is published only from a Notification Handler.

### A new version of an Integration Event

A wire name is one version of one contract type. A change receivers could not read, such as a new required field, is a new version: a new type under a new wire name, published beside the old one while subscriptions move to it. Nothing converts one version into another, and there is no schema registry. What receivers can read follows from how a body is read: every member the receiving contract requires, each constructor parameter without a default value, must be there and not null where it is non-nullable, and a member the contract lacks is skipped ([rabbitmq.md](rabbitmq.md#a-body-the-contract-cannot-read)). So a publisher may add a member while subscriptions still run without it, and a receiving contract may add one only with a default value; a new member every receiver requires is a new version, since a body without it is set aside as a dead letter. In the Proving Sample, `ProductCreatedV2` adds the product's category:

```csharp
// Catalog.Contracts/Products/Events/V2/: a type of its own, never derived from V1
public sealed record ProductCreatedV2(Guid Id, DateTimeOffset OccurredOn, Guid ProductId, string Name, Guid CategoryId);

// Catalog's Composition Root: both versions, each under its own wire name
//     module.Publish<ProductCreatedV1>("catalog.product-created.v1");
//     module.Publish<ProductCreatedV2>("catalog.product-created.v2");

// PublishProductCreatedNotificationHandler: both, with the one Domain Event's Id
events.Publish(domainEvent.Id, new ProductCreatedV1(domainEvent.Id, domainEvent.OccurredOn, domainEvent.ProductId.Value, domainEvent.Name));
events.Publish(
    domainEvent.Id,
    new ProductCreatedV2(domainEvent.Id, domainEvent.OccurredOn, domainEvent.ProductId.Value, domainEvent.Name, domainEvent.CategoryId.Value)
);

// Inventory's Composition Root: a subscription to v2, beside its subscription to v1
//     module.Subscribe<ProductCreatedV2, FileProductCategoryIntegrationEventHandler>("inventory.product-categories");
```

- Add the new record under `Events/V2/`, named with its version, as a type of its own: copy the fields it keeps rather than deriving it from the old one.
- Declare it with `Publish` under a new wire name with the new version (`catalog.product-created.v2`), and publish both versions from the same Notification Handler with the same Domain Event `Id`. One Domain Event yields one Integration Event of each type, so the two versions are allowed together, and each has a publication record of its own.
- Each subscription receives only the version it subscribed to: a v1 subscription never receives v2, and a v2 subscription never receives v1. Both versions carry the one identity, and each subscription keeps its own Receipt of it, keyed by its subscription name. Receipts never deduplicate across subscription names: a v1 and a v2 subscription that apply the same effect both apply it, once each.
- **A new, independent effect** goes in a new subscription to v2, beside the v1 subscriptions, which keep theirs. This is what the sample does: `inventory.product-categories` files the product under its category, which no v1 subscription does.
- **Replacing a v1 subscription** with a v2 one that applies the same effect is a coordinated cutover: never let both apply it while both versions are published, or every product gets the effect twice, such as two stock items. Make it in the release that stops publishing v1, together with every other v1 subscription still live, since none receives anything once v1 stops: remove the `events.Publish` line of v1, subscribe each v2 handler under its new name, and keep each v1 subscription, and v1's `Publish` declaration, so its queue drains what was published before the release. Each product then has a v1 or a v2 delivery, never both. Remove the v1 subscriptions in a later release, once their queues hold nothing: a stored v1 delivery to a subscription nothing declares any more is set aside as a dead letter, never run ([Names](#names)). Never remove a v1 subscription before its v2 replacement is live: a version published while no subscription of it exists is not stored, so that effect is lost. An effect that must move to v2 while v1 is still published needs a v2 handler that itself recognises an effect already applied, such as by the product's id.
- Retire v1 once no Module subscribes to it: remove its `Publish` declaration, then the record. While a v1 subscription is declared, keep the declaration, which gives v1 its wire name, or the Host fails when it is built. A published type nothing in the Host subscribes to is not stored.
- Declaring the same type under two wire names, two types under one wire name, or a wire name equal to another subscribed contract type's full name fails the Host ([The Host's checks](#the-hosts-checks)).

## Snapshots between Modules

A Snapshot is the current state of one aggregate, published for other Modules, and it supersedes every earlier Snapshot of that aggregate. A Tombstone is the Snapshot that says the aggregate no longer exists. Choose between a Snapshot and an Integration Event by what the receiver needs:

- An Integration Event is a fact: something happened, and the receiver reacts to it, as Inventory opens a stock item for each new product. Each one matters on its own, and a Notification Handler publishes it.
- A Snapshot is state: what the aggregate is now, for a Module that keeps its own copy of it, as Inventory keeps a copy of each catalog product. Only the newest matters: once a newer one is applied, an older one is worth nothing. Feature code never publishes it; the Unit of Work does, in the commit of every change of the aggregate.
- One aggregate may have both, as Catalog's product does: `ProductCreatedV1` for what happened, `ProductSnapshotV1` for what it is.

In the Proving Sample, Inventory keeps a copy of Catalog's products:

1. Every Operation of Catalog that creates, renames or removes a product stores the product's `ProductSnapshotV1` in Catalog's Outbox, in the commit of that change, once per subscription.
2. After that commit, Inventory's `CopyProductIntegrationEventHandler` writes it into Inventory's copy, in Inventory's Unit of Work, when it is newer than every Snapshot of that product Inventory applied.

### The Snapshot

```csharp
// Catalog.Contracts/Products/Events/V1/: public, plain values only, nothing from the Kit
/// <summary>
/// The current state of one product, which supersedes every earlier Snapshot of it: <paramref name="Version"/> grows
/// with every change Catalog commits. When <paramref name="IsRemoved"/> is true it is the product's Tombstone: the
/// product has left the catalog. Catalog publishes it under the wire name <c>catalog.product-snapshot.v1</c>; a change
/// other Modules could not read is a new version, not an edit of this one.
/// </summary>
public sealed record ProductSnapshotV1(Guid ProductId, string Name, bool IsRemoved, long Version);
```

- Declare it as a `public sealed record` in the publishing Module's Contracts project, under `Events/V1/`, named with its version, with plain values only, as an Integration Event ([The Integration Event](#the-integration-event)).
- Give it the aggregate's key, the state other Modules need, a field that says whether it is the Tombstone, as `IsRemoved` does, and the aggregate's `Version`. The Kit orders Snapshots by the version it sends beside the message, not by these fields; the sample carries the version too, so its copy shows which one it holds.
- A type is either an Integration Event or the Snapshot of one aggregate: declaring it with `Publish` and with `PublishSnapshot`, or with two `PublishSnapshot` declarations that differ in aggregate or mapper, fails the Host's composition, and `IIntegrationEventPublisher.Publish` refuses a Snapshot.
- A change that receivers could not read is a new type with its own version and wire name, never an edit of the old one.

### Writing a Snapshot mapper

```csharp
using Catalog.Contracts.Products.Events.V1;
using PouryaKit.Messaging;

// Catalog/Products/Events/: the product as other Modules see it. The Kit calls it once for every product an
// Operation changed, after every reaction ran, with the version being committed; it only reads.
internal sealed class ProductSnapshotMapper : ISnapshotMapper<Product, ProductSnapshotV1>
{
    public ProductSnapshotV1 Map(Product product) => new(product.Id.Value, product.Name, IsRemoved: false, product.Version);

    public ProductSnapshotV1 Tombstone(Product product) => new(product.Id.Value, product.Name, IsRemoved: true, product.Version);
}
```

- Implement `ISnapshotMapper<TAggregate, TSnapshot>` (`PouryaKit.Messaging`) in an `internal sealed` class of the aggregate's Business Area, under `Events/`. It has two methods: `Map` builds the Snapshot of an aggregate that exists, and `Tombstone` builds the Snapshot of one the Operation removed, whether it is kept marked deleted or deleted outright. Both receive the aggregate in its final state.
- Map by hand, member by member, with no mapping library. Read the version from the aggregate's `Version`: it is the version being committed.
- `PublishSnapshot` registers the mapper scoped, and the Kit resolves it in the scope of the Operation that commits and runs it as a step of the Unit of Work that only reads. It reads the aggregate, and may run a query through an injected `IDispatcher` when it needs more; it writes nothing. The sample's takes nothing. [The rules of a mapper and a key](#the-rules-of-a-mapper-and-a-key) say what is refused and what fails the Operation.

### Declaring a Snapshot

- The publishing Module declares each Snapshot in its Composition Root: `module.PublishSnapshot<Product, ProductSnapshotV1, ProductSnapshotMapper>("catalog.product-snapshot.v1")` (`PouryaKit.Messaging.Wolverine`). It names the aggregate, the Snapshot type and the mapper, then the wire name; the Module whose Operations publish it is the one whose builder declares it ([modules.md](modules.md#the-composition-root)). One aggregate may have several Snapshot types, each with its own declaration, wire name and mapper.
- The receiving Module subscribes an ordinary `IIntegrationEventHandler<TSnapshot>` with `Subscribe`, as Inventory does above ([Declaring what a Module publishes and subscribes](#declaring-what-a-module-publishes-and-subscribes)). Several Modules may subscribe, each under its own subscription name.
- A Snapshot's wire name and its subscriptions' names are declared, never derived, and follow [Names](#names) like an Integration Event's.
- An application without Modules calls `PublishSnapshot` on its own services, where it belongs to the application.

### Keeping a copy

```csharp
using Catalog.Contracts.Products.Events.V1;
using Dapper;
using PouryaKit.Cqrs;
using PouryaKit.Domain;
using PouryaKit.Messaging;
using PouryaKit.Persistence;

// Inventory/CatalogProducts/Features/CopyProduct/: keeps Inventory's copy of each catalog product current, in a table
// of its own that is no aggregate, keyed by tenant and product under the tenant the Snapshot was delivered for. Only the
// effect: the Kit runs this only for a Snapshot newer than every one applied of that product, so the copy simply follows
// each one, and is deleted on the Tombstone.
internal sealed class CopyProductIntegrationEventHandler(IUnitOfWorkConnection unitOfWork, OperationContext context)
    : IIntegrationEventHandler<ProductSnapshotV1>
{
    public async Task<Result> Handle(ProductSnapshotV1 snapshot, CancellationToken cancellationToken)
    {
        var (connection, transaction) = unitOfWork.Join();
        var sql = snapshot.IsRemoved
            ? "delete from inventory.catalog_products where tenant = @Tenant and product_id = @ProductId"
            : """
              insert into inventory.catalog_products (tenant, product_id, name, version) values (@Tenant, @ProductId, @Name, @Version)
              on conflict (tenant, product_id) do update set name = excluded.name, version = excluded.version
              """;
        await connection.ExecuteAsync(
            new CommandDefinition(
                sql,
                new { context.Tenant, snapshot.ProductId, snapshot.Name, snapshot.Version },
                transaction,
                cancellationToken: cancellationToken
            )
        );
        return Result.Success();
    }
}
```

- Keep the copy in a Module-internal table that no aggregate maps, written with plain SQL through `IUnitOfWorkConnection` in the receiving Module's Unit of Work ([dapper-reads.md](dapper-reads.md#writing-with-plain-sql-in-the-unit-of-work)), so the copy, the Receipt and the version the Kit records commit together or not at all.
- The copy is not an aggregate. It holds what the publishing Module last published and nothing the receiving Module decides: it has no rule to guard, raises no Domain Event, and needs no version of its own, because the Kit already decides which Snapshot is applied. Inventory's own decisions about a product live in its own aggregates, such as the stock item.
- Write only the effect: insert or update for a Snapshot, delete for a Tombstone. The handler compares no version: the Kit runs it only for a newer one, and after a Tombstone it runs for no older one, so a deleted row stays deleted.
- A copy of a tenant-scoped Module's aggregates is keyed by tenant and aggregate, written under the tenant the delivery carries, which the injected `OperationContext` holds; the receiving Module need not be tenant-scoped ([multi-tenancy.md](multi-tenancy.md#deliveries)).
- Inventory's project references `PouryaKit.Persistence.Dapper`, which brings Dapper for `ExecuteAsync` and `CommandDefinition`. The copy needs only `IUnitOfWorkConnection`, which comes with `UsePouryaKitPostgres`; Inventory calls `AddPouryaKitDapper` for its `GetStockItem` read, not for the copy.

The receiving Module's migration creates the copy and the table the Kit keeps the applied versions in, `CreateSnapshotVersionTable` (`PouryaKit.Messaging.Wolverine.EFCore`), beside the Receipt table it already has ([Receipts](#receipts)). Neither is in the Module's model, so the model does not change; add the migration with `dotnet ef migrations add` as for the Receipt table and fill its `Up` and `Down`:

```csharp
using Microsoft.EntityFrameworkCore.Infrastructure;
using Microsoft.EntityFrameworkCore.Migrations;
using PouryaKit.Messaging.Wolverine.EFCore;

// Inventory/Infrastructure/Data/Migrations/: the copy, written with plain SQL, and the versions of the Snapshots applied
[DbContext(typeof(InventoryDbContext))]
[Migration("20261005190000_CatalogProducts")]
internal sealed class CatalogProducts : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql(
            "create table inventory.catalog_products (product_id uuid primary key, name text not null, version bigint not null)"
        );
        migrationBuilder.CreateSnapshotVersionTable("inventory");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropSnapshotVersionTable("inventory");
        migrationBuilder.Sql("drop table inventory.catalog_products");
    }
}
```

- Inventory's later migration, `20261017120000_CatalogProductTenants`, adds the `tenant` column to the copy's key, and refuses a table that already holds a copy, whose tenant cannot be recovered; a new copy declares it in its `create table`, `(tenant text not null, product_id uuid not null, ..., primary key (tenant, product_id))`.
- A Module that subscribes to any Snapshot needs both tables in its schema, each created once; [Receipts](#receipts) says which Module needs which table and what fails without one.
- A version row is not a Receipt: it holds, for each subscription and aggregate, the newest version applied, and is what keeps an older Snapshot from undoing a newer one or a Tombstone. The Kit removes no row from it, and nothing should.

### What the Kit guarantees for a Snapshot

- **One per changed aggregate, per Snapshot type, per Operation.** Each Operation of the declaring Module that changes an aggregate, or anything the aggregate owns, publishes one Snapshot of it under each Snapshot type declared for it, however many times the Operation changed it: in the handler, in a nested command and in a Domain Event Handler, it is still one. That holds for every Operation of the Module, a delivery's included, as when one of its Notification Handlers changes the aggregate. An Operation that changes nothing, or rolls back, publishes none.
- **Built from what commits.** The Unit of Work builds it in the step after it prepares persistence, once every Domain Event Handler ran ([persistence.md](persistence.md#the-unit-of-work)): the aggregate is in its final state, audit and soft-delete fields set, and its `Version` is the one being committed.
- **Stored in the commit, sent after it.** It is written to the publishing Module's Outbox, once per subscription, in the same transaction as the change, so it exists if and only if the change commits, and it is sent after the commit, as a Notification is ([What the Kit guarantees](#what-the-kit-guarantees)). A Snapshot type that no Module of the Host subscribes to is not stored.
- **A Tombstone with a newer version.** An aggregate the Operation removed, whether soft-deletable and kept marked deleted or deleted outright, publishes the mapper's `Tombstone` instead, with the version the removal advanced it to, one newer than its last Snapshot.
- **Applied only when newer.** Each Snapshot carries its aggregate's key, as the database stores it, and its version, in the `SnapshotHeaders.AggregateId` and `SnapshotHeaders.Version` headers. Its identity follows from its wire name, key and version, so every delivery of one Snapshot has the same identity and two versions never share one. The receiving delivery first records the Receipt, so a redelivery or a concurrent duplicate of the same Snapshot does nothing. It then records the version, and the handler runs only when that version is strictly newer than every version of that aggregate the subscription applied before; otherwise the delivery succeeds without running it. A concurrent delivery of another version of the same aggregate waits until the first commits or rolls back, then compares with what committed. If the older version commits first, both handlers can run in increasing version order; if the newer commits first, the older is skipped. So Snapshots delivered in reverse order, concurrently or twice, and a live Snapshot delivered after the Tombstone, all leave the receiver's copy at the newest version. The version is recorded in the handler's transaction, with its effect and Receipt, or not at all.
- **Not every version.** A receiver applies the newest Snapshot it is given and skips the older ones it gets later: a Snapshot handler keeps a copy current, and never counts or reacts to each change. React to what happened with an Integration Event.
- **A continuation of the change.** The receiving Operation continues the Operation that committed the change: the same Actor, tenant and correlation, that Operation's `OperationId` as its causation, a new `OperationId`.
- **Each subscription on its own.** Each subscription's delivery is retried, recorded and set aside as a dead letter apart from the others, as for an Integration Event ([Retries](#retries), [Dead letters](#dead-letters)).
- **No copy before the aggregate's next change.** A Snapshot is published only when an Operation changes its aggregate, and only to the subscriptions declared then, so a Module that subscribes after data exists has no copy of an aggregate until it next changes. Until then, a Feature that needs an aggregate its copy lacks asks the owning Module through its Contracts, the synchronous question the Kit allows when the answer is needed before the Operation can go on ([modules.md](modules.md#asking-another-module-a-question)). Do not write the copy from that answer. The Kit's version rule never sees a row written that way, so that row can overwrite a newer Snapshot delivered meanwhile, or survive a Tombstone delivered meanwhile.

### The rules of a mapper and a key

A feature author keeps these rules, or the copies in other Modules go wrong:

- **A mapper only reads.** It runs after every Domain Event was handled and every version and audit field set, so what it changed would commit unversioned and unseen by the Snapshot, and a Domain Event it raised would never be handled. It may read the aggregate and run queries, nothing else:
  - It runs as a step of the Unit of Work that only reads, so the Kit refuses every write it attempts, as in any [code that only reads](commands-queries-validation.md#code-that-only-reads): a nested command, the repository's `AddAsync` and `Remove`, `IUnitOfWorkConnection.Join()` and publishing an Integration Event. Each refusal is an `InvalidOperationException` that fails the Operation and reaches the Gateway's caller, and nothing commits. A mapper that catches the refusal of its command still fails the Operation: the refused command marks it rollback-only, and the Unit of Work checks that once more after the mappers ran, so the Operation fails with `OperationErrors.CommandThrew` (code `Operation.CommandThrew`).
  - The Unit of Work compares tracked state once after all mappers finish against the state before the batch. The shared EF comparison includes complex collections recursively and pending Domain Event identity and order. A difference throws an `InvalidOperationException` during commit and rolls back the transaction. This comparison cannot detect a transient change restored before the batch ends. Queries a mapper calls retain their individual read guards.
- **The aggregate's key is set by the application, not generated by the database.** The Snapshot is built before the save, and a key the database generates does not exist until then. Nothing checks it at startup: an Operation that creates such an aggregate fails at its commit with an `InvalidOperationException`, and nothing commits. Create keys as `Product` does, with `TypedId.New<ProductId>()`.
- **Nothing in the aggregate's mapped graph has a value the database generates.** The Snapshot is built before the save, so a value the database fills in or computes when it writes the row would be published as it was before, while the row commits the database's. The Kit refuses a property with a default value SQL (`HasDefaultValueSql`), a computed column or any other value the database generates on add or update, such as a child's key that an identity column numbers, in the aggregate, every type it owns and every child it holds through a navigation, whatever the mapper reads. The aggregate's own key keeps the rule above, and a shadow key, such as the one EF gives an owned collection by default, is left out, since no mapper can read it: nothing checks it at startup, and an Operation that commits a change of such an aggregate fails at its commit with an `InvalidOperationException` naming the property, and nothing commits. Let the application set the value, as the Kit sets the version and the audit fields. A value a client-side value generator sets for good when the entity is added is the application's and is fine: one `HasValueGenerator` names, or a child's `Guid` key that EF sets when the child is added.
- **The provider key has a lossless encoding.** After EF's value converter, each key part must be a non-null `Guid`, `int`, `long`, `string`, `byte[]` or `DateTime`. Guid, int, long and string keep their existing invariant formats; bytes use uppercase hexadecimal. A DateTime uses its ticks without Kind, so PostgreSQL's timestamp without time zone can reload it with Unspecified Kind without changing its identity. Its mapping must be a PostgreSQL timestamp with precision from 0 to 6, and every value must align to that precision. The Kit refuses date mappings, submicrosecond values and any other value that the mapping would round or truncate. Composite keys prefix each encoded part with its invariant length and a colon. Null and other provider types fail the Operation before saving. A converter to another supported provider type must preserve the key across reloads.
- **Old colliding identities need a deliberate recovery.** Earlier byte-array keys all encoded as `System.Byte[]`, and DateTime formatting lost fractions of a second. Existing Receipts and applied-version rows cannot be migrated losslessly because those identities do not identify the original key. If such keys were deployed, drain their old deliveries and deliberately rebuild their projections, Receipts and applied-version state from the publishing data before using the corrected encoding. Guid, int, long and string identities are unchanged.
- **The key of an aggregate that publishes Snapshots is never used again after the aggregate is deleted outright.** Every receiver keeps its Tombstone's version and would turn away a new aggregate's lower versions under the same key until they passed it. The Kit's Guid version 7 keys are never used again. A soft-deleted aggregate keeps its key and its version, and the repository never returns it again; a key a person chooses, such as a code, is the one to watch.
- **A declaration belongs to its Module and covers one aggregate.** Snapshots are built only in Operations of the Module the declaration belongs to: in a Host of Modules declare it on the Module's builder, and in an application without Modules on its own services. A declaration without a Module in a Host of Modules fails the Host's build, and so does one that names a Module in an application without Modules, which would never publish ([The Host's checks](#the-hosts-checks)): `AddPouryaKitModules` and `UsePouryaKitWolverine` install the composition checks. The refusal at commit remains only as the backstop of a Host without them: the Operation that changes its aggregate fails with an `InvalidOperationException`, and nothing commits. A declaration of one Module never publishes for another Module's aggregates. Declare it for one aggregate, or for a type of one mapped inheritance hierarchy, never for a base type that independent aggregates share: their keys are their own, so their Snapshots would share identities and versions, and a receiver would turn one aggregate's away for another's. Such a declaration is refused the same way, by the first Operation of the Module that commits a change to any aggregate the Module declares a Snapshot for.

## Delayed commands

A command's handler, a Domain Event Handler, or a Notification or Integration Event Handler, whose delivery runs in a Unit of Work, schedules one of its own Module's commands to run later with `IDelayedCommands` (`PouryaKit.Messaging.Wolverine`), as Billing's `OpenWelcomeQuoteIntegrationEventHandler` schedules a welcome quote's `ExpireQuote`. The Unit of Work writes it to the Outbox in the same commit as the change, with the time it is due, and the message store keeps it until then. It is not a recurring job: work that recurs on a cadence, with no Operation to continue, is a recurring command ([Recurring commands](#recurring-commands)). From the Proving Sample's Billing Module:

```csharp
using PouryaKit.Cqrs;
using PouryaKit.Domain;
using PouryaKit.Messaging.Wolverine;
using PouryaKit.Persistence;

// Quotes/Features/OpenQuote/: the quote and its expiry commit together, or neither does
internal sealed record OpenQuoteCommand(Guid QuoteId, TimeSpan ValidFor) : Command;

internal sealed class OpenQuoteCommandHandler(IRepository<Quote, QuoteId> quotes, IDelayedCommands later, TimeProvider time)
    : CommandHandler<OpenQuoteCommand>
{
    public override async Task<Result> Handle(OpenQuoteCommand command, CancellationToken cancellationToken)
    {
        var quote = Quote.Open(new QuoteId(command.QuoteId), time.GetUtcNow() + command.ValidFor);
        await quotes.AddAsync(quote, cancellationToken);
        later.Schedule(new ExpireQuoteCommand(command.QuoteId), quote.ExpiresOn);
        return Result.Success();
    }
}

// Quotes/Features/ExpireQuote/: an ordinary command, with an ordinary handler
internal sealed record ExpireQuoteCommand(Guid QuoteId) : Command;

// In the Composition Root:
//     module.AddPouryaKitDelayedCommand<ExpireQuoteCommand>("billing.expire-quote");
```

- **Declare each delayed command** in its Module's Composition Root with `module.AddPouryaKitDelayedCommand<TCommand>(name)`; the Module is the builder's. The name is its durable queue and half of its Receipts' key, so keep it when the command is renamed; it follows the rules of a subscription name ([Names](#names)) and may not be a subscription's. The stored command names its CLR type and waits on the queue of its name, so renaming or moving the type, or renaming the declaration, strands the commands stored before it: they are set aside as dead letters, the renamed declaration's with an `UnknownDelayedCommandException`, never run under the new name's Receipts. A Module schedules only commands it declared: `Schedule` throws for any other, in code that only reads, and outside a Unit of Work, such as in an endpoint, a hosted service or any scope that sends no command, where no commit would store the command.
- **Nothing schedules before the commit.** `Schedule` only records the command. A failed Result, a nested command that failed, a refused save or an exception leaves no delayed command, as it leaves no Notification.
- **When it runs.** The message store runs it once it is due, on whichever Host instance is running then: a restart, or the death of the Host that scheduled it, loses nothing, and a time already past when the Operation commits runs it at once. The store compares the time with the system clock, not with the Host's `TimeProvider`, and finds it on its scheduled-job poll, so it runs up to one poll after it is due. A test that needs it to run uses a short real delay, not a fake clock; the test fixture does not wait for a delayed command not yet due ([testing.md](testing.md#deliveries)). Its claim when due restarts its stale clock, so a delayed command due later than `InboxStaleTime` is not handed out again while it runs ([The stale-time rule](#the-stale-time-rule)).
- **On whose authority.** It runs as a continuation of the Operation that scheduled it: the same Actor, tenant and correlation, that Operation's `OperationId` as its causation, a new `OperationId`, never a system Actor. It goes through the dispatcher in the declared Module, with its authorizers and validator; a declared permission is not checked again, because its run is a trusted continuation ([security.md](security.md#continuations)). Write its handler for the state it finds then, which may have changed since it was scheduled.
- **Once in the database.** Its delivery records a Receipt, the name and the stored command's identity, in the command's own Unit of Work, so a redelivery after a crash, or a replay, applies its effect once. An effect outside the database is at least once, as for a Notification ([What the Kit guarantees](#what-the-kit-guarantees)). The Module needs the Receipt table ([Receipts](#receipts)).
- **A failure** is retried and set aside as a dead letter of its name, received at `local://{name}/`, as a subscription's delivery is ([Retries](#retries), [Dead letters](#dead-letters)); a failed Result becomes a `DelayedCommandFailedException`, which names each Error.
- **Cancelling** is the handler's business: the Kit does not remove a scheduled command, so check the state, as `ExpireQuote` refuses a quote that is no longer open.

## Recurring commands

Work that recurs on a cadence with no Operation to continue, such as a nightly reconciliation, is a recurring command: one of the Module's own commands, declared in its Composition Root with the system Actor it runs as. It is not a delayed command: no Operation schedules it, and it continues none ([Delayed commands](#delayed-commands)). From the Proving Sample's Inventory Module:

```csharp
using Dapper;
using PouryaKit.Cqrs;
using PouryaKit.Domain;
using PouryaKit.Persistence;

// StockItems/Features/ReconcileStock/: the command carries the occurrence it runs
internal sealed record ReconcileStockCommand(DateTimeOffset Occurrence) : Command;

internal sealed class ReconcileStockCommandHandler(IUnitOfWorkConnection unitOfWork, OperationContext context)
    : CommandHandler<ReconcileStockCommand>
{
    public override async Task<Result> Handle(ReconcileStockCommand command, CancellationToken cancellationToken)
    {
        var (connection, transaction) = unitOfWork.Join();
        await connection.ExecuteAsync(
            new CommandDefinition(
                "insert into inventory.stock_reconciliations (occurrence, stock_items, reconciled_by) select @Occurrence, count(*), @ReconciledBy from inventory.stock_items",
                new { command.Occurrence, ReconciledBy = context.Actor.Id },
                transaction,
                cancellationToken: cancellationToken
            )
        );
        return Result.Success();
    }
}

// In the Composition Root:
//     module
//         .AddPouryaKitRecurringCommand(
//             "inventory.reconcile-stock",
//             systemActor: "inventory.stock-reconciler",
//             occurrence => new ReconcileStockCommand(occurrence)
//         )
//         .BindConfiguration("Inventory:ReconcileStock");
// In the Host's appsettings.json:
//     "Inventory": { "ReconcileStock": { "Every": "01:00:00" } }
```

- **Declare it** in its Module's Composition Root with `module.AddPouryaKitRecurringCommand<TCommand>(name, systemActor, occurrence)`; the Module is the builder's. The name follows a delayed command's rules: its durable queue and half of its Receipts' key, so keep it; never a subscription's or another delayed command's. `systemActor` is required and names the Actor every occurrence runs as, `Actor.System(systemActor)`: nothing defaults to a system Actor. `occurrence` builds the command for each occurrence's time and must return the declared type. A command is declared once, delayed or recurring, never both.
- **Configure its cadence.** The declaration returns the options builder of `PouryaKitRecurringCommandOptions`, named by the declaration's name, lower-cased: bind its `Every` from configuration (`BindConfiguration(section)`), or set it with `Configure`. The Host fails to start with an `OptionsValidationException` when `Every` is not longer than zero, so a missing configuration never runs it at a default, and when it is longer than `PouryaKitRecurringCommandOptions.MaxEvery`, 36500 days, which keeps every occurrence's time far from the latest one a `DateTimeOffset` holds. A first occurrence that a starting Host fails to store, once every hosted service has started, fails the start with that failure, and the Host is stopped before it throws. A changed cadence applies from the next occurrence stored.
- **When it runs.** Occurrences fall on whole multiples of `Every` counted from the Unix epoch, in UTC: an hourly one runs at the top of each hour, a daily one at midnight UTC. The message store compares them with the system clock, not with the Host's `TimeProvider`, and finds one due on its scheduled-job poll (`ScheduledJobPollingTime`, 5 s by default), so a cadence shorter than the poll is not kept. A Host that starts runs the first occurrence after now.
- **On whose authority.** Each attempt is a new Operation of the declared system Actor, in the declared Module, with no tenant, so a tenant-scoped Module cannot declare one (`RecurringCommandTenancyCheck` fails the Host when it is built), a new correlation and no causation, never a trusted continuation: it inherits nothing from the Operation that stored it, and nothing from any HTTP caller. Its trace parent is not restored either ([observability.md](observability.md)). It goes through the dispatcher with its authorizers and validator; a permission the command declares is checked for the system Actor as for any Actor, so the application grants it ([security.md](security.md#the-actors-permissions)).
- **Once in the database.** Its delivery records a Receipt, the name and the occurrence's time, which the `DelayedCommandHeaders.Occurrence` header carries, not the stored delivery's identity, in the command's own Unit of Work. A redelivered, replayed or concurrently duplicated occurrence finds it and applies nothing. An effect outside the database is at least once, as for a Notification.
- **A failure** is retried and set aside as a dead letter of its name, as a delayed command's is, and the cadence goes on, unless the occurrence no longer reads (the next bullet): the next occurrence it would have stored rolled back with its effect, so the attempt that sets it aside stores the next occurrence after now, as an Operation of the system Actor through the Module's Unit of Work, unless one already waits. That attempt is the one numbered one more than `PouryaKitMessagingOptions.RetryDelays` has delays; an attempt that fails before it stores nothing, since its retry may still succeed and store the next occurrence in its own commit. The occurrence set aside is not run again unless its dead letter is replayed, which runs it late; the next occurrence it then stores meets the waiting one at the first occurrence both reach, whose Receipt lets one apply. If storing the next occurrence fails too, the dead letter still records the occurrence's own failure, an Error entry carries both failures, the store's first, and the work waits for a replay or a Host start, each of which stores the next occurrence after now. Watch the dead letters received at `local://{name}/`.
- **A stored occurrence that no longer reads** does not go on. An occurrence stored before a release changed its command's shape, a required member renamed or added, fails to read when it comes due, and Wolverine sets it aside as a dead letter of its name, with a `JsonException` and an Error entry naming its queue, before the Kit's delivery runs: it stores no next occurrence, and a Host that started while it waited stored none either. The work waits for the next Host start, which stores the first occurrence after now (`RecurringCommandTests`). To recover:

  1. **Find it.** An Error entry "Envelope … to local://{name}/ was moved to the error queue" with a `System.Text.Json.JsonException`, and a row of Wolverine's `wolverine_dead_letters` table whose `received_at` is `local://{name}/`.
  2. **Start any one instance of the Host again.** Setting the occurrence aside deleted its stored row, so the start finds none waiting and stores the first occurrence after now. The occurrences missed in between are not made up.
  3. **Discard the dead letter**, with `IDeadLetters.DiscardAsync` (`IWolverineRuntime.Storage.DeadLetters`) and a `DeadLetterEnvelopeQuery` whose `ReceivedAt` is `local://{name}/` and `ExceptionType` is `System.Text.Json.JsonException`, or by deleting its row. Never replay it: a replay is read as strictly and set aside again. Sending the command is no way back either, since Feature code never schedules a recurring command.

  Change a recurring command's shape only by adding members with a default value, or declare the same command type under a new name: a starting Host does not count an occurrence of the retired name as waiting, and that occurrence is set aside as a dead letter when it comes due ([Restart semantics](#restart-semantics)). An occurrence of a type no declaration names any more is not counted as waiting either. It is set aside at once, without retries, as a dead letter: Wolverine finds no handler for it, and the Kit sets `UnknownMessageBehavior` to dead-letter an unknown message type. Keeping the old type and forwarding it to the new, as Wolverine's `IForwardsTo<T>` does, is not supported: Wolverine builds a forwarder only toward a handler it discovered, and the Kit registers each delayed and recurring command's delivery itself, with discovery off, so an occurrence of the old type never reaches the Kit's delivery, and a starting Host, which looks for the declared type, does not count it as waiting.
- **Not scheduled by Feature code.** `IDelayedCommands.Schedule` throws for a recurring command: one more occurrence would run outside its cadence.
- **Declared once, as one kind.** A command is declared once, delayed or recurring; declaring a recurring one again with another system Actor or factory throws, naming the declaration it has. Only an occurrence carries the `DelayedCommandHeaders.Occurrence` header, so a delivery stored as the other kind is refused before any Operation begins: a delayed command stored before its name was declared recurring, or an occurrence stored before it was declared delayed, is set aside as a dead letter with an `UnknownDelayedCommandException` and never runs under the other kind's authority or Receipt. Changing a declaration's kind strands what it stored; give the new kind a new name.

### Restart semantics

- **One occurrence stored at a time.** Each occurrence stores the next one, the first one after both its own time and now, in its own commit, through the Unit of Work's Outbox, with its effect and its Receipt. A Host that stops or dies therefore leaves the next occurrence in the message store, owned by no Host, and whichever Host runs next goes on with it; nothing is held in memory.
- **A starting Host adds one only when none waits.** After every hosted service started, the Host looks in the message store for an occurrence waiting on the declaration's queue, due or not, a stored retry included; when there is none, it stores the first occurrence after now, as an Operation of the system Actor through the Module's Unit of Work. A stored occurrence of a retired name does not count; it is set aside as a dead letter when it comes due.
- **Missed occurrences are skipped.** While no Host runs, the stored occurrence waits. A Host that starts later runs it, with the time it was due at, then the next occurrence after now; the occurrences between are not made up.
- **The limit, and what keeps it safe.** An occurrence taken from the store and still running is not waiting, so a Host that starts at that moment, an attempt that sets another occurrence aside then, or two Hosts that start together on an empty store, store a second occurrence. Both chains fall on the same occurrence times, so they meet at the first one both reach, which is applied once: the delivery that finds its Receipt applies nothing and stores no next occurrence, and one chain remains. The guarantee is one effect in the database per declaration and occurrence time, not one stored occurrence at every instant, and not exactly one run of the handler.
- **Testing.** The test fixture's Host stores its first occurrence when it starts, the reset removes it, and the fixture does not wait for an occurrence not yet due ([testing.md](testing.md#deliveries)). A test that needs occurrences to run starts its own Host on a database of its own with a short cadence, as `RecurringCommandTests` does, so the fixture's Host never runs one.

## What the Kit guarantees

- **Stored in the commit.** The Unit of Work writes each Domain Event the Operation raised that a Notification Handler subscribes to into the Outbox, once per subscription, after it prepares persistence and before it saves, in the same transaction. The Notification exists if and only if the change commits: a failed Result, a failed Domain Event Handler, a refused save or an exception leaves none. A Domain Event that no Notification Handler subscribes to stores nothing.
- **Sent after the commit.** The Unit of Work sends what it wrote only after the commit. When the send fails, or the Host dies first, the Notification stays in the store: a Host that starts recovers it, and a live Host hands it out again after `InboxStaleTime` ([The message store](#the-message-store)). The Operation's Result is unaffected, because its change is committed; the failed send is logged as a warning.
- **A continuation in its own scope.** Each delivery attempt runs in a new scope, as an Operation of the subscribing Module, begun with a continuation of the raising Operation's context: the same Actor, tenant and correlation, the raising Operation's `OperationId` as its causation, a new `OperationId`. It runs through the dispatcher, with the Module's Unit of Work and the Host's clock, and nothing from a failed attempt reaches the next. In a tenant-scoped Module a delivery whose context carries no tenant, of a Notification, an Integration Event or a Snapshot, is refused with `Tenant.Required` before its Receipt and its handler, retried, then set aside as a dead letter ([multi-tenancy.md](multi-tenancy.md#deliveries)).
- **Each handler on its own.** Each Notification Handler of a Domain Event has its own stored delivery, retried, recorded, acknowledged and set aside as a dead letter apart from the others, so a handler that fails never makes one that succeeded run again.
- **A database effect once.** Before the handler runs, the delivery records a Receipt, the subscription name and the Domain Event's `Id`, in the handler's transaction, so the Receipt commits with the handler's effect or not at all. A redelivery after that commit, such as after a crash before the store marked the delivery handled, finds the Receipt and does nothing. A concurrent copy of the same delivery waits until the first commits, then does nothing; if the first rolled back, the copy runs. A handler whose effect is in its Module's database needs nothing more to be safe to run again.
- **An outside effect at least once.** An effect outside the database, such as an email, cannot roll back with the Receipt. When the attempt that sent it fails or dies before its Receipt commits, the next attempt sends it again. Make a repeat harmless where the outside system allows it, for example by passing it the Domain Event's `Id` as an idempotency key, and do the outside effect last.

An Integration Event is delivered the same way, sent after the commit, in a new scope per attempt, on its own and applied once, and also:

- **Stored with the publishing handler's Receipt.** It is written to the publishing Module's Outbox, once per subscription, in the commit of the Notification Handler's delivery that published it, together with that handler's Receipt and effect. A delivery that fails after publishing leaves no Integration Event; only the attempt that commits stores one.
- **One durable delivery per subscription**, on the subscription's own queue, retried, recorded, acknowledged and set aside as a dead letter apart from every other. When several Modules subscribe to one Integration Event, a subscription that keeps failing never runs one that succeeded again: not through its retries, not when it is set aside or replayed, and not when its Host dies during it and another Host takes it over.
- **Applied once.** Its identity is the `Id` of the Domain Event it follows from, carried by every delivery of it in the `IntegrationEventHeaders.DomainEventId` header. The receiving handler's Receipt, the subscription name and that identity, commits together with its effect, so a redelivery after the commit, or a concurrent duplicate, applies it once.
- **A continuation, two Operations on.** The receiving Operation continues the publishing Notification Handler's delivery: the same Actor, tenant and correlation as the Operation that made the change, the delivery's `OperationId` as its causation, a new `OperationId`. No HTTP request is involved.
- **The same identity and time at both ends.** The Kit enforces the identity: `Publish` refuses any other. The time is the same when the Integration Event carries the Domain Event's `OccurredOn`, as `ProductCreatedV1` does.
- **Each version to its own subscriptions.** A subscription receives only the type it subscribed to, so during a transition v1 and v2 of one fact reach their own subscriptions only, each with its own Receipt.

## Receipts

- A Module keeps its handlers' Receipts in a table `pouryakit_receipts` in its own schema, keyed by the subscription name, lower-cased, and the identity of what was delivered: a Notification's or Integration Event's is the Domain Event's `Id`; a Snapshot's is derived from its wire name, its aggregate's key and its version. Neither key holds a tenant (ADR 0006's amendment on tenants at delivery and in the keys): a tenant-scoped Module's Snapshot of an aggregate whose key two tenants could share fails the Host when it is built ([multi-tenancy.md](multi-tenancy.md#declaring-a-tenant-scoped-module)). The table is not part of the Module's model, so a migration creates it with `CreateReceiptTable` and the Module's schema.
- Which Module needs which table:
  - `pouryakit_receipts` (`CreateReceiptTable`): every Module that declares a Notification Handler, subscribes an Integration Event Handler, publishes Integration Events or declares a delayed or recurring command. Inventory calls it in its first migration, beside its own tables. A publishing Module's table also holds one row per Domain Event and wire name it published, keyed `<wire name>#published`, which no subscription name can take: it is how the Kit refuses a second Notification Handler publishing the same type for one Domain Event, and it is not a Receipt.
  - `pouryakit_snapshot_versions` (`CreateSnapshotVersionTable`, [Keeping a copy](#keeping-a-copy)): every Module that subscribes to a Snapshot, beside its Receipt table.
- Nothing checks either table at startup. Without one, every delivery that needs it fails with an `InvalidOperationException` that names the call to add, and is retried and set aside as a dead letter. `HostFixture` applies every migration before it starts the Host, so a test shows the gap at once.
- The Kit removes no Receipt or publication record, and nothing should: a Receipt is what keeps a redelivery from applying twice, and a `#published` row is how the Kit refuses a second publication of one type for one Domain Event. A delivery can come again without ever becoming a dead letter: a broker envelope whose copies all committed before its Host died is fanned out again on restart ([rabbitmq.md](rabbitmq.md#what-the-kit-guarantees-across-the-broker)). A retention period tied to dead-letter expiry is therefore not safe. `pouryakit_receipts` therefore grows by one row per delivery a subscription applies and one per Integration Event published, and the Kit's one cleanup removes idempotency keys only ([persistence.md](persistence.md#idempotency-key-retention)).
- Add it with `dotnet ef migrations add Receipts` and the Module's `--project`, `--context` and `--output-dir`, as in [persistence.md](persistence.md#migrations). The model did not change, so the migration is empty and the snapshot stays as it was; fill its `Up` and `Down`:

```csharp
using Microsoft.EntityFrameworkCore.Infrastructure;
using Microsoft.EntityFrameworkCore.Migrations;
using PouryaKit.Messaging.Wolverine.EFCore;

// Infrastructure/Data/Migrations/
[DbContext(typeof(CatalogDbContext))]
[Migration("20261005180000_Receipts")]
internal sealed class Receipts : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder) => migrationBuilder.CreateReceiptTable("catalog");

    protected override void Down(MigrationBuilder migrationBuilder) => migrationBuilder.DropReceiptTable("catalog");
}
```

## Names

Each Notification and Integration Event is stored under its wire name, and each delivery under its handler's subscription name, which is also the name of its queue and half of its Receipt's key. Both are durable identifiers.

- **A Domain Event's wire name, by convention:** the assembly's name and the type's name without the `DomainEvent` suffix, each in kebab case: `catalog.product-created` for `ProductCreatedDomainEvent` in the assembly `Catalog`. It comes from the assembly, not from the Module.
- **A Notification Handler's subscription name, by convention:** the wire name, a dot, and the handler's name without the `NotificationHandler` or `Handler` suffix, in kebab case: `catalog.product-created.announce-new-product`.
- **Overrides:** `[WireName("...")]` on the Domain Event and `[SubscriptionName("...")]` on the Notification Handler (`PouryaKit.Messaging`) declare a stable name instead.
- **An Integration Event's and a Snapshot's names are declared, never derived:** the wire name is the first argument of `Publish` (`catalog.product-created.v1`) or `PublishSnapshot` (`catalog.product-snapshot.v1`), and each Integration Event Handler's subscription name the first argument of `Subscribe` (`inventory.product-created`). Neither depends on a CLR name, so renaming or moving the record or the handler keeps them. Put the version in the wire name, and change it only with a new version of the type.
- **One type per wire name, one wire name per type.** A Host refuses a second type under a wire name and a type under a second wire name, ignoring case; a new version is a new type under a new wire name ([A new version of an Integration Event](#a-new-version-of-an-integration-event)).
- **One wire name per type in a process.** Wolverine keeps message names for the whole process, so a second Host in the same process, such as one a test starts, that publishes a type under another wire name fails when it is built.
- **A rename is a breaking change.** A derived name changes when its Domain Event, its handler or its assembly is renamed, or the type moves to another assembly; a declared one changes when its declaration does. Deliveries already stored keep the old name: those under a wire name nothing declares any more no longer find their type, those under a subscription name nothing declares arrive on a queue no subscription declares, and either is set aside as a dead letter, never run and never treated as handled. Before renaming a type or handler that uses a derived name while deliveries may be in flight, declare the name it had:

```csharp
using PouryaKit.Domain;
using PouryaKit.Messaging;

// Renamed from ProductCreatedDomainEvent and AnnounceNewProductNotificationHandler: both keep the names they had.
[WireName("catalog.product-created")]
internal sealed record ProductAddedDomainEvent(ProductId ProductId, CategoryId CategoryId, string Name) : DomainEvent;

[SubscriptionName("catalog.product-created.announce-new-product")]
internal sealed class AnnounceAddedProductNotificationHandler(INewProductAnnouncer announcer)
    : INotificationHandler<ProductAddedDomainEvent>
{
    public async Task<Result> Handle(ProductAddedDomainEvent domainEvent, CancellationToken cancellationToken)
    {
        await announcer.AnnounceAsync(new NewProductAnnouncement(domainEvent.ProductId.Value, domainEvent.Name), cancellationToken);
        return Result.Success();
    }
}
```

- **A valid name** is letters and digits in segments joined by single dots or hyphens, as above, not beginning with `amq.` in any case, the prefix RabbitMQ reserves for its own queues and exchanges; empty, blank, a space or any other character is invalid. A subscription name may not be a queue Wolverine keeps for itself: `agents`, `default`, `durable`, `replies`, `scheduled` or `system`.
- **Case is ignored.** Queues lower-case their names, so two wire names, or two subscription names, that differ only in case are a collision, whether they belong to Domain Events, Integration Events or their handlers; so are `Shelved` and `ShelvedDomainEvent` in one assembly, whose derived wire names are the same. Changing only the case of a subscription name keeps the same subscription, Receipts included.
- A missing, invalid, reserved or colliding name fails the Host's composition ([The Host's checks](#the-hosts-checks)), naming every problem.

## Retries

- A delivery fails when its handler throws or its Operation returns a failed Result, including when nested work made it rollback-only even though the handler returned success. A failed Result is never acknowledged. Nothing the attempt changed commits, its Receipt included.
- A failed delivery is retried after each of `PouryaKitMessagingOptions.RetryDelays` (`PouryaKit.Messaging.Wolverine`; 1 s, 10 s and 1 min by default): one attempt more than there are delays, so four. Each wait is stored with the delivery, which no Host instance holds while it waits, so a restart neither loses nor repeats it. Null, an empty schedule or a negative delay stops the Host when it starts, with an `OptionsValidationException` naming the option, so a failed delivery is retried at least once. A zero delay retries at once.
- Each attempt runs in a new scope, as a new Operation, and nothing of the attempt before is left in it: no DbContext, tracked aggregate or scoped service of the failed attempt reaches the next.
- Only the failing subscription's delivery is retried. The other subscriptions of the same Domain Event or Integration Event are not delivered again, whether they succeeded or fail too.

## Dead letters

- When its attempts are used up, the delivery is set aside as a dead letter of its own subscription: its envelope leaves the Inbox for Wolverine's dead-letter storage, the table `wolverine_dead_letters` in the store's schema, and is never marked handled. Only that subscription's delivery is set aside; the others of the same message keep what they did.
- A dead letter says why. It keeps the exception of its last attempt, type and message: a handler that threw records its own exception, and one that returned a failed Result records a `NotificationHandlerFailedException` or an `IntegrationEventHandlerFailedException` (`PouryaKit.Messaging.Wolverine`), whose message names the subscription and each Error as its code and message, such as `PriceList.Closed (The price list is closed.)`. Both exceptions also carry `Subscription` and `Errors`.
- A dead letter is found by its subscription: it was received at `local://{subscription}/`, its `received_at`, with the subscription name lower-cased, and its `message_type` is the wire name. One received from another Host that failed before it reached a subscription was received at the Host's broker queue instead ([rabbitmq.md](rabbitmq.md#dead-letters-at-the-broker-queue)).
- A stored delivery whose subscription or type no longer exists is set aside the same way, never run and never treated as handled: one to a subscription the Host no longer declares fails with `UnknownSubscriptionException`, through its retries, and one whose type no subscription of the Host handles any more is dead-lettered as an unknown message. Renaming or removing a subscription, or the last subscription to a type, while deliveries are in flight leaves those deliveries as dead letters ([Names](#names)).
- A delivery whose body its type cannot read, because it lacks a member the type requires, carries null where the type forbids it or names one of its members twice, is set aside at once, without retries, with the `System.Text.Json.JsonException` that names the member, never run with a default. That is how a Notification or delayed command stored before its type gained a required member ends when it is read back, after a retry or a restart.
- The Kit adds no dead-letter API of its own: Wolverine's dead-letter storage already finds a dead letter by its subscription and replays it, and a second way to do the same would be a second implementation of one concern. The Host reads and replays them through `IWolverineRuntime.Storage.DeadLetters`, from operator code such as an admin endpoint or a maintenance command, never from Feature code:

```csharp
using Wolverine.Persistence.Durability;
using Wolverine.Persistence.Durability.DeadLetterManagement;
using Wolverine.Runtime;

// ModularMonolith.Host: operator code. Feature code never references Wolverine.
public sealed class DeadLetterTriage(IWolverineRuntime runtime)
{
    // Every dead letter of one subscription; a page size of 0 reads them all.
    public async Task<IReadOnlyList<DeadLetterEnvelope>> OfAsync(string subscription, CancellationToken cancellationToken) =>
        (
            await runtime.Storage.DeadLetters.QueryAsync(
                new DeadLetterEnvelopeQuery { ReceivedAt = $"local://{subscription.ToLowerInvariant()}/", PageSize = 0 },
                cancellationToken
            )
        ).Envelopes;

    // Once the cause is fixed. The store moves each back to its subscription's queue on its next recovery poll.
    public Task ReplayAsync(IEnumerable<DeadLetterEnvelope> deadLetters, CancellationToken cancellationToken) =>
        runtime.Storage.DeadLetters.ReplayAsync(
            new DeadLetterEnvelopeQuery([.. deadLetters.Select(deadLetter => deadLetter.Id)]),
            cancellationToken
        );
}
```

- Replaying marks the dead letter for replay; the store's next recovery poll moves it back to its subscription's queue, with its attempts counted from zero. Only that subscription's handler runs: the other subscriptions of the message are not delivered again. If it fails again, it is retried and set aside again.
- A replay never applies a message twice. A delivery replayed after one of its attempts committed, such as one set aside again by hand, finds its Receipt and does nothing, so replaying the same delivery twice applies it once.
- Fix the cause before replaying: a replay into the same failure only uses up the retries again.

## The message store

- `PouryaKitMessageStoreOptions` (`PouryaKit.Messaging.Wolverine`) holds the store's settings:
  - `Durability`: `DurabilityMode.Balanced`, the default, is safe with any number of Host instances on one store when all are instances of one Host (the store's schema, below). `DurabilityMode.Solo` assumes a single instance and recovers a stopped instance's deliveries at once; the tests use it.
  - `InboxStaleTime`, 5 minutes by default: how long a delivery an instance took and no longer holds waits before the store hands it out again. It is what delivers a Notification or Integration Event committed but never sent while its Host lives on, so such a delivery waits that long. Set it by [the stale-time rule](#the-stale-time-rule). Zero or less stops the Host when it starts, with an `OptionsValidationException` naming the option.
- The store's schema is the second argument of `UsePouryaKitWolverine(connectionName, schema, configure)`, `wolverine` by default. Instances of one Host share one store and one schema. Two different Hosts, with different handlers, Modules or applications, each take a store of their own: another database, or another schema on one database through that argument. Sharing a schema between different applications is unsupported: a node of one recovers the other's stored deliveries, has no handler for them and dead-letters them ([Wolverine issue 4923](https://github.com/JasperFx/wolverine/issues/4923)). This is a deployment rule, and no check enforces it.
- Change the settings anywhere in the Host's composition, before it is built: `builder.Services.Configure<PouryaKitMessageStoreOptions>(store => store.InboxStaleTime = TimeSpan.FromMinutes(15));`.
- The store's settings, the Outbox write and the scope's waiting Integration Events and commands are `PouryaKit.Messaging.Wolverine`'s, behind `UsePouryaKitMessageStore` and `OutboxWriter`, which a store package's registration and Unit of Work call; a Host calls the store package's registration, `UsePouryaKitWolverine(connectionName)`, never those. In a Host of Modules each Module's Receipts are its write model's: the EF package keys `EfReceipts` for every Module that keys none of its own, the Marten package keys a Stream Module's in that Module's schema ([stream.md](stream.md)), and the Host resolves the Receipts of the Operation's Module, whichever of `AddPouryaKitModules` and `UsePouryaKitWolverine` comes first.

## The stale-time rule

`InboxStaleTime` releases a delivery an instance took and no longer holds, such as a Notification committed but never sent while its Host lives on, once its clock is older than the stale time. The clock restarts when an instance claims the delivery, a due retry, delayed command or recurring occurrence included, and three times per stale time while the instance holds it, queued or running (WolverineFx 6.51.0 and later). So a delivery waiting for its retry is never handed out again, and a retry, a delayed command or a recurring occurrence due later than the stale time, as an hourly occurrence is under the 5-minute default, is not stale when it starts (`InboxStaleTimeTests`): the stale time need not cover `RetryDelays`. The restarts while an instance holds a delivery are what keep a long attempt from being handed out again, so each must land within the stale time; in the Kit's loaded test run a 2-second stale time let a held retry be handed out twice.

**The rule a Host must follow:** keep `InboxStaleTime` well above the difference between the database's clock and the Host's, which the store compares, and above the delay a loaded Host adds to a database round trip, since a held delivery whose clock its Host fails to restart in time is handed out again. The 5-minute default keeps it. A longer stale time has its price: a delivery committed but never sent waits that long on a live Host ([The message store](#the-message-store)).

A delivery handed out again while an attempt of it still runs keeps one effect: each copy waits on the running attempt's Receipt and then does nothing, and a recurring command keeps one next occurrence (`RecurringCommandTests`). But each copy holds a database connection while it waits, and when the attempt it waits on fails, a copy runs the handler itself, so a handler with an effect outside the database can repeat it beyond its retries ([What the Kit guarantees](#what-the-kit-guarantees)). Do such an effect in a Notification Handler of the Domain Event the handler raises ([Delayed commands](#delayed-commands)).

## The Host's checks

Each declaration refuses its own mistakes while the Host is composed; then the Host's checks run at three moments: when it is built, when it starts, and once Wolverine has started.

- **When the Host is composed**, each declaration throws, naming every problem:
  - `AddPouryaKitNotificationHandlers`, for an assembly with no Notification Handler; a handler that handles several Domain Event types, an abstract or generic one, or is generic itself; a wire name or subscription name that is missing, invalid, reserved or colliding; and the same subscription declared for another Module. Repeating its declaration for the same Module adds nothing.
  - `Publish`, for a type that is not a concrete class or is a Domain Event; a wire name that is missing, invalid or colliding; and a type already published under another wire name or by another Module.
  - `PublishSnapshot`, for everything `Publish` refuses, and for a type already published as an Integration Event or as the Snapshot of another aggregate or with another mapper.
  - `Subscribe`, for a Domain Event; an abstract handler; a subscription name that is missing, invalid, reserved or colliding; and the same subscription declared for another Module. Repeating its declaration for the same Module adds nothing.
  - `Receive` and `ReceiveSnapshot`, as [rabbitmq.md](rabbitmq.md#a-host-that-receives) lists.
  - `AddPouryaKitDelayedCommand`, for an abstract or generic command; a name that is missing, invalid, reserved or another delayed command's; and the same command declared under another name or for another Module. Repeating it adds nothing.
  - `AddPouryaKitRecurringCommand`, for everything `AddPouryaKitDelayedCommand` refuses, a missing system Actor or occurrence, and a command already declared as a delayed command.
- **When the Host is built**, the composition checks fail it, one line per declaration, for a publication, Snapshot, subscription, Notification Handler, delayed or recurring command declared without a Module in a Host of Modules, or naming a Module the Host never added, in an application without Modules too: made on `builder.Services` without naming the Module, it belongs to no Module, so its first publish, commit, delivery or scheduling would fail once the Host serves. Each line names the declaration, its form on the Module's builder and its form that names the Module, or, in an application without Modules, its form without a Module ([modules.md](modules.md#the-hosts-checks)). `Receive` and `ReceiveSnapshot` belong to the Host and are not checked. The composition checks also fail it when Notification Handlers, Integration Events, Snapshots or delayed commands are declared but nothing writes them to an Outbox: `UsePouryaKitWolverine()` without a connection name, from `PouryaKit.Messaging.Wolverine`, runs Wolverine with no database, for a Host that declares none of them. A write model's own package fails it too when the Host has an Outbox but that write model's Unit of Work has none, naming its Modules and the registration to call: EF Core Modules in a Host whose only message store is `UsePouryaKitMarten`, and Stream Modules beside the EF message store (`UsePouryaKitWolverine(connectionName)`) and `AddPouryaKitMarten` ([stream.md](stream.md#limits)). And `UsePouryaKitWolverine` adds one more, for a Notification Handler in the Feature assemblies of a Module the Host composed, or of its own services, that no `AddPouryaKitNotificationHandlers` declares: it would never run, and a Domain Event no other handler subscribes to would be dropped before its commit (`The Notification Handlers … of CatalogModule are declared by no AddPouryaKitNotificationHandlers, so they never run. Call AddPouryaKitNotificationHandlers() in its Composition Root.`). The EF Core `UsePouryaKitWolverine(connectionName)` adds one for the store's database: a DbContext that `UsePouryaKitPostgres` stores over a connection string naming another database than the store's. Wolverine writes each envelope on the Unit of Work's own connection and transaction, so its Outbox would go to that database: the first Operation that writes to it fails there when it holds no store tables, and commits silently where no delivery reads it when it holds a copy of them. The check compares the string Wolverine was given with the connection EF Core gives each DbContext, created in a scope of the check's own, whatever configured it: a `ConfigureDbContext` after `UsePouryaKitPostgres` counts, whether onto another string, a connection or an Npgsql data source, and EF Core resolves the named form `Name=ConnectionStrings:app` from the Host's final configuration. It reads both as Npgsql does: each server a host and a port, a number written after the host or the separate `Port`, 5432 by default; a Unix socket directory, a rooted path or one starting with `@`, in its own case with the separate `Port`; any other host ignoring case; servers in any order; and the database, or the user's name when the string names none. It opens no connection and resolves no host name, so two connection names for one database pass, and `localhost` and `127.0.0.1` differ. A string that names neither a database nor a user, on either side, leaves the database to the client's user, so it fails the build too, one line naming its connection and the fix: set `Database`, or at least `Username`, in it. What EF Core refuses when the check creates the DbContext, such as a named form the configuration lacks or a model error, fails the build with EF Core's own exception. A missing or blank DbContext connection string has its own line ([persistence.md](persistence.md)). The line names both connection names, both databases and the fix (`CatalogDbContext is stored over the connection string catalog, the database db:5432/catalog, but UsePouryaKitWolverine keeps the message store over the connection string app, the database db:5432/app, so its Unit of Work would write its Outbox to its own database, where the store never delivers it. Use one connection string for the message store and every Module's DbContext.`). `UsePouryaKitWolverine` installs the composition checks itself (`AddPouryaKitCompositionChecks`), so a Host without Modules has them too.
- **When the Host is built**, too, `UsePouryaKitWolverine` adds two composition checks of its own. The routing check fails the build, naming each problem, for a subscription to an Integration Event or Snapshot that no Module of the Host publishes and no `Receive` or `ReceiveSnapshot` declares; for a type received from another Host that no subscription or Saga of the Host handles, which would bind nothing and receive nothing (`… is declared received from another Host as …, but no subscription or Saga of this Host handles it, so nothing would receive it. Subscribe<ProductCreatedV1, THandler>(subscriptionName) in the Composition Root of the Module that receives it, or remove the Receive declaration.`); for a message type Wolverine would also route to another subscribed type's queues, or wherever another published type is sent, such as its exchange on a broker, because its wire name, its name or its full name equals that type's full name, ignoring case (a v2 published under the full name of its v1 would reach every v1 subscription, and every Host that receives v1); for a delayed or recurring command whose name a subscription has; for an Integration Event this process already sends under another wire name; and for an Integration Event that Wolverine had already named otherwise before the Host declared it, because Wolverine keeps the first name it gives a type and would send it under that name. Only a Host whose routes pass declares its wire names for the process, all or none, when it is built, so a Host built and never started keeps them too, and Wolverine routes by them once its options are built. The Unit of Work check fails the build for a subscription, of an Integration Event Handler or a Notification Handler, whose Module registers no Unit of Work, or, in an application without Modules, of the application when it registers none: each delivery runs as a command in its Module's Unit of Work and records its Receipt there, so every one would fail before its handler runs. One line per Module names its subscriptions and `UsePouryaKitPostgres` ([modules.md](modules.md#the-hosts-checks)); a Module that only answers queries needs no Unit of Work.
- **When the Host starts**, it fails with an `OptionsValidationException` for a recurring command whose cadence is not longer than zero or is longer than `PouryaKitRecurringCommandOptions.MaxEvery`.
- **After every hosted service started**, Wolverine included, it fails when Host configuration made a subscription's or a delayed command's queue one that does not store what it receives, such as `LocalQueue(name).BufferedInMemory()`, and when it made a broker listener one that does not, or whose dead letters go to a broker queue ([rabbitmq.md](rabbitmq.md#the-hosts-checks)). This check has Wolverine's own moment, the internal `DurableQueuesCheck` at `StartedAsync`: Wolverine compiles its endpoints, applying its endpoint policies, such as durable local queues, and the Host's own queue configuration, only as it starts, so before that every queue reads its default mode. By then every hosted service has started, so the check stops the Host before the start fails.
- Not checked: that a Module has the Receipt table and, for a Snapshot, the Snapshot version table ([Receipts](#receipts)). Not checked at startup either, but failing the Operation that commits: a Snapshot of an aggregate whose key, or any other value in its mapped graph, the database generates, and a Snapshot declaration that covers independent aggregates ([The rules of a mapper and a key](#the-rules-of-a-mapper-and-a-key)). A Snapshot declaration that names a Module in an application without Modules fails the build with the composition checks, as any declaration naming a Module the Host never added; its refusal at commit remains only as the backstop of a Host without them.

Testing deliveries: [testing.md](testing.md#deliveries).
