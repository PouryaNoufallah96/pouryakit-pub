# Modules

`PouryaKit.Modularity`: the Composition Root, the Gateway and one scope per Operation, in one Host container. Namespace `PouryaKit.Modularity`. The Operation Context it requires lives in `PouryaKit.Cqrs`.

- [Adding a Module](#adding-a-module)
- [The Composition Root](#the-composition-root)
- [The Gateway](#the-gateway)
- [The Operation Context](#the-operation-context)
- [Which Module an Operation belongs to](#which-module-an-operation-belongs-to)
- [The Host's checks](#the-hosts-checks)
- [Asking another Module a question](#asking-another-module-a-question)
- [An application of one Module](#an-application-of-one-module)
- [Testing](#testing)

## Adding a Module

Start from the modular monolith template, or grow the monolith into it ([templates.md](templates.md#growing-the-monolith-into-a-modular-monolith)). A new Module sits beside the generated ones.

1. Give the Module a folder under `src/Modules/` with its project, and a Contracts project when other Modules use what it publishes, as Catalog has `Catalog.Contracts`. Its project references `PouryaKit.Modularity` and the Kit packages its Features use.
2. Write its Composition Root: one public sealed class that implements `IModuleDefinition` ([below](#the-composition-root)).
3. In `AppHost.cs`, add it beside the others with `modules.Add<…>()` in `AddPouryaKitModules`, and give it its own section in `appsettings.json` when it has settings.
4. Give it a migration, then migrate it naming its project and its DbContext ([persistence.md](persistence.md#migrations)).

A tenant-scoped Module also follows [multi-tenancy.md](multi-tenancy.md).

## The Composition Root

A Module's Composition Root and the Host that composes it, after the Proving Sample:

```csharp
// Catalog/CatalogModule.cs: the Module's one public type
public sealed class CatalogModule : IModuleDefinition
{
    public void Configure(ModuleBuilder module)
    {
        module.AddPouryaKitTenancy();   // first: every Operation of Catalog needs a tenant; see multi-tenancy.md
        module.UsePouryaKitPostgres<CatalogDbContext>(connectionName: "app", schema: "catalog");
        module.MapPouryaKitUniqueConstraint<CatalogDbContext>(ProductConfiguration.NameIndex, ProductErrors.NameInUse);   // see persistence.md
        module.AddPouryaKitDapper(connectionName: "app");   // see dapper-reads.md
        module.AddPouryaKitCqrs();
        module.AddPouryaKitValidation();
        module.AddPouryaKitNotificationHandlers();   // see messaging.md
        module.Publish<ProductCreatedV1>("catalog.product-created.v1");   // for other Modules; see messaging.md
        module.PublishSnapshot<Product, ProductSnapshotV1, ProductSnapshotMapper>("catalog.product-snapshot.v1");   // likewise
        module.AddScoped<IProductRepository, ProductRepository>();   // see persistence.md, The repository
        module.AddScoped<ICatalogProducts, CatalogProducts>();       // see Asking another Module a question
        module.AddPouryaKitEndpoints("/catalog");   // see web.md
    }
}

// ModularMonolith.Host: one builder method, shared by Program and the tests; its Build method adds the pipeline (web.md)
public static WebApplicationBuilder CreateBuilder(string[] args)
{
    var builder = WebApplication.CreateBuilder(args);
    builder.Services.AddPouryaKitJwtBearer(claims => claims.TenantClaim = "tenant");   // see security.md
    builder.Services.AddAuthorization();
    builder.Services.AddPouryaKitWeb();   // reads the tenant the bearer scheme verified; see web.md
    builder.AddPouryaKitModules(modules => modules.Add<CatalogModule>().Add<InventoryModule>().Add<BillingModule>());
    builder.UsePouryaKitWolverine(connectionName: "app");   // the Modules' deliveries; see messaging.md
    return builder;
}
```

- Implement `IModuleDefinition` once per Module, as a public sealed class with a parameterless constructor. Everything else in the Module stays `internal`, except the ports its Host implements for it with the records they carry ([messaging.md](messaging.md#writing-a-notification-handler)) and the migrations `dotnet-ef` writes; a structure test rejects any other public type ([testing.md](testing.md#structure-tests)).
- `Configure(ModuleBuilder module)` declares everything the Module needs and offers: its data (`UsePouryaKitPostgres`, `MapPouryaKitUniqueConstraint`, `AddPouryaKitDapper`), its Features (`AddPouryaKitCqrs`, `AddPouryaKitValidation`), its Notification Handlers (`AddPouryaKitNotificationHandlers`), the Integration Events and Snapshots it publishes and the Integration Event Handlers it subscribes (`Publish`, `PublishSnapshot`, `Subscribe`), its endpoints (`AddPouryaKitEndpoints`) and its own services. `ModuleBuilder` is an `IServiceCollection` over the Host's one container, so every registration extension works on it.
- A Composition Root never names its own Module or its own assembly. `ModuleBuilder` is the Module seam, `IModuleServices` (`PouryaKit.Cqrs`), which gives every Kit registration made on it the Module's definition type, and the Kit finds it at run time, so a helper of yours typed as `IServiceCollection` keeps it:
  - **Its Features live in its definition's assembly.** `AddPouryaKitCqrs()`, `AddPouryaKitValidation()`, `AddPouryaKitPermissions()`, `AddPouryaKitDapper(connectionName)`, `AddPouryaKitNotificationHandlers()`, `AddPouryaKitEndpoints(prefix)` and `AddPouryaKitLocalization()` with no assembly scan or record the assembly of the class that implements `IModuleDefinition`. Features that live elsewhere are named instead, `module.AddPouryaKitCqrs(typeof(CatalogModule).Assembly, typeof(SharedFeatures).Assembly)`: named assemblies replace the inferred one, in their order.
  - **Its declarations belong to it.** `Publish`, `Subscribe`, `PublishSnapshot`, `AddPouryaKitNotificationHandlers`, `AddPouryaKitDelayedCommand` and `AddPouryaKitRecurringCommand` on its builder run in the Module: its Unit of Work, its Receipts. Each has a second form that names the Module, `builder.Services.Subscribe<ProductCreatedV1, AuditHandler>("inventory.audit", typeof(InventoryModule))`, for the Host's composition when it declares on a Module's behalf; named on another Module's builder it throws while the Host is composed, and naming a Module the Host never adds fails the Host's build.
- The Unit of Work registered on a Module's builder becomes that Module's own; every other registration goes into the Host's container as it is. The persistence and messaging packages key the Module's Receipts and idempotency store by the Module too, so an Operation keeps its Receipts and keys in its own Module's store, whatever write model another Module has and in whichever order the Host calls `AddPouryaKitModules` and `UsePouryaKitWolverine`.
- The Host adds each Module once: `builder.AddPouryaKitModules(modules => modules.Add<CatalogModule>().Add<InventoryModule>().Add<BillingModule>())`. That runs each Composition Root and registers the Module's Gateway.
- **A Module's own settings** live under its name in the configuration of each Host that composes it, `Catalog:Section:Key`, and an environment variable overrides one as `Catalog__Section__Key`. `Configure(ModuleBuilder)` receives no configuration: bind the section to an options type of the Module's, validate it at start and inject it, `module.AddOptions<T>().BindConfiguration("Catalog:Section").Validate(options => ..., message).ValidateOnStart()`, then `IOptions<T>` where a Feature needs it, so a Host given an invalid value fails to start with an `OptionsValidationException` carrying that message, instead of at first use. Inventory's recurring command is the example: its Composition Root binds `Inventory:ReconcileStock` onto the options builder `AddPouryaKitRecurringCommand` returns, which already carries the Kit's validation and `ValidateOnStart()`, so the Module only binds ([messaging.md](messaging.md#recurring-commands)).

## The Gateway

- `IGatewayProcessor<TModule>` is how anything outside a Module runs one of its Operations: `SendAsync(command, context, ct)` for a command, `QueryAsync(query, context, ct)` for a query, each returning the Result. Resolve it from the Host's services.
- The `OperationContext` is required. Each call is one Operation in a new scope, begun with that context and with the Module as its `Operation.Module`, and disposed when the call returns.
- Commands and queries stay `internal`, so a caller that builds one lives in the Module's assembly or is granted `InternalsVisibleTo`, as the Proving Sample's test project is. Another Module asks through the owning Module's Contracts instead.

A scheduled import, which is independent system work, creates a context for each Operation it runs:

```csharp
var catalog = host.Services.GetRequiredService<IGatewayProcessor<CatalogModule>>();
var importer = Actor.System("catalog-import");

var lamps = await catalog.SendAsync(new CreateCategoryCommand(Guid.CreateVersion7(), "Lamps"), new OperationContext(importer), ct);
var created = await catalog.SendAsync(
    new CreateProductCommand(Guid.CreateVersion7(), lamps.Value, "Desk lamp"),
    new OperationContext(importer),
    ct
);
```

## The Operation Context

`OperationContext` (`PouryaKit.Cqrs`) is plain data about who and what caused one Operation:

- `OperationId`: a new version 7 Guid for each context created, so one context belongs to one Operation.
- `Actor`: `Actor.Person(id)`, a person as the trust boundary that authenticated them identifies them, or `Actor.System(name)`, a named system identity for independent work no person caused, such as a recurring command. `Actor.Kind` says which.
- `Tenant`: the tenant or company the Operation runs for; null where the application has none.
- `CorrelationId`: shared by every Operation in one chain of work; a new context with none given correlates with its own `OperationId`.
- `CausationId`: the Operation or message that caused this one; null for the Operation that started the chain. Informational: it never makes a context trusted.

Starting and continuing work:

- A caller that starts work creates `new OperationContext(actor, tenant)` and passes it to the Gateway. Create one for each Operation: two calls with one context share an `OperationId`.
- Work one Operation causes runs with `context.Continue()`: the same Actor, tenant and correlation, the causing Operation's `OperationId` as its causation, and a new `OperationId`. A Module that answers another Module's question passes the asking Operation's `Continue()` to its own Gateway.
- A continuation is trusted when the Kit restores it for a delivery, or when a running, authorized Operation continues its own context with `Continue()`: its Actor is not authenticated again, and the Kit adds no authorization of its own for it but the tenant requirement of a tenant-scoped Module, which refuses a command or query of that Module continued without a tenant ([multi-tenancy.md](multi-tenancy.md#declaring-a-tenant-scoped-module)). A context application code builds, or continues outside the Operation that ran it, is never trusted. An `IAuthorizer` registered for the command or query it runs still runs, with the original Actor, and recognises a trusted continuation by `IsTrustedContinuation`, never by `CausationId`.
- There is no anonymous or default Actor. `Actor.Person` and `Actor.System` refuse a blank id or name, the Gateway throws `ArgumentNullException` for a null context, and resolving `OperationContext` in a scope whose Operation was never begun throws `InvalidOperationException`. Name system work with `Actor.System("...")`.

Reading it:

- Inject `OperationContext` into a handler, an authorizer, a validator or a Domain Event Handler. Nested commands and queries and the Domain Event Handlers of an Operation run in its scope, so they read the same context. Read it only by injection, never from an ambient request.

```csharp
using PouryaKit.Cqrs;
using PouryaKit.Domain;

// Products/Features/RemoveProducts/: only system work removes products in bulk
internal sealed class RemoveProductsCommandAuthorizer(OperationContext context) : IAuthorizer<RemoveProductsCommand>
{
    public Task<Result> AuthorizeAsync(RemoveProductsCommand command, CancellationToken ct) =>
        Task.FromResult(context.Actor.Kind == ActorKind.System ? Result.Success() : ProductErrors.BulkRemoveForbidden);
}
```

Setting it:

- Whoever opens an Operation's scope begins it once, before anything in the scope runs, with `scope.ServiceProvider.BeginOperation(context, module)` (`PouryaKit.Cqrs`). The Gateway does this for every call, so Feature code never calls it; only an application of one Module calls it itself (below). Beginning a scope's Operation twice throws. A message delivery also passes the trace parent it restored from the message, as its third argument ([observability.md](observability.md#the-trace-through-delivery)); nothing else passes one.
- At the HTTP boundary, the Web Capability begins each request's Operation with a context built from the authenticated caller, and the endpoint passes that context to the Gateway ([web.md](web.md#the-caller-and-the-actor)). The delivery of a Notification begins its scope with a continuation of the raising Operation's context and the subscribing Module, and the delivery of an Integration Event with a continuation of the publishing Notification Handler's delivery and the receiving Module ([messaging.md](messaging.md#what-the-kit-guarantees)).

## Which Module an Operation belongs to

- `Operation.Module` (`PouryaKit.Cqrs`, a `Type`) is the Module the scope's opener named; the Gateway names its own. It belongs to the scope, not to the context, because work an Operation causes may run in another Module. It routes each command to that Module's Unit of Work: a command sent through `IDispatcher` in a scope whose Operation names no Module throws `InvalidOperationException` in a Host of Modules.
- A command or query belongs to the Module whose Composition Root registers its handler. Its Gateway is the only one that runs it: sent through another Module's Gateway, it is refused with an `InvalidOperationException` naming both Modules, and one that no Module handles is refused too. Neither opens a scope.
- A Composition Root that registers, replaces or removes the handler of a command or query another Module owns fails while the Host is composed, naming both Modules.

## The Host's checks

Each declaration refuses its own mistakes while the Host is composed; then the Host's checks run at three moments, each failing before the Host serves: when it is built, when it starts, and once Wolverine has started. An endpoint and an HTTP client are also checked as each is built ([web.md](web.md)):

- **While the Host is composed**, in `AddPouryaKitModules`: a Module added twice, in one call or across calls, throws naming it. A Module whose Composition Root registers the handler of a command but no Unit of Work throws, naming the Module and the command (`… handles the command … but registers no Unit of Work. Declare its data in its Composition Root, for example with UsePouryaKitPostgres.`); a Module that only answers queries needs none. A declaration that names another Module on a Module's builder throws (`The Composition Root of … declares for …`).
- **When the Host is built**: `AddPouryaKitModules` composes the Host's checks with `AddPouryaKitCompositionChecks` (`PouryaKit.Cqrs`), whose container is validated in every environment: every service it builds through a constructor must have its dependencies registered, and no singleton may depend on a scoped service, or `Build()` throws an `AggregateException` naming the problem; the validation does not look inside a factory registration or an open generic one. A service that needs the Gateway of a Module the Host never added fails here, naming that Gateway. Then the Kit's composition checks run on the built container, before `Build()` returns, since a built Host runs Operations through its Gateways before it starts, and `Build()` throws one `InvalidOperationException`, one line per problem, each naming its fix:
  - a command or query whose handler is registered and that declares an authorizer the Kit must register, such as a `[RequiresPermission]`, when no Composition Root registered it; another authorizer of the same command or query does not count (`… declares RequiresPermissionAttribute, but its check is not registered, so it would run unchecked. Call AddPouryaKitPermissions with the assembly … in the Composition Root that registers its handler.`). Fix it with `module.AddPouryaKitPermissions()` in that Module's Composition Root ([security.md](security.md#declaring-a-permission));
  - a cacheable query whose handler is registered in a Host without `AddPouryaKitCaching()` ([caching.md](caching.md));
  - a command or query a tenant-scoped Module, or the tenant-scoped application, can dispatch without the tenant requirement, as when its `AddPouryaKitCqrs` scans an assembly its `AddPouryaKitTenancy` was not given; another authorizer of the same command or query does not count. The line names the Module or the application, the operation and the assembly to name to `AddPouryaKitTenancy` (`TenantRequirementCheck`, [multi-tenancy.md](multi-tenancy.md));
  - a recurring command a tenant-scoped Module, or the tenant-scoped application, declares: its occurrences run for a system Actor with no tenant, and none runs per tenant. The line names the command and the Module (`The recurring command … is declared in …, which is tenant-scoped, …`; [multi-tenancy.md](multi-tenancy.md#declaring-a-tenant-scoped-module));
  - with `UsePouryaKitWolverine(connectionName)`, a Snapshot a tenant-scoped Module, or the tenant-scoped application, declares of an aggregate whose mapped key neither includes its tenant nor is a single Guid, since a Snapshot's identity and version key hold no tenant. The line names the Module, the aggregate and its key (`… publishes the Snapshot of …, whose key (…) neither includes the tenant nor is a Guid, …`; [multi-tenancy.md](multi-tenancy.md#declaring-a-tenant-scoped-module));
  - a publication, Snapshot, subscription, Notification Handler, delayed or recurring command declared without a Module in a Host of Modules, or naming a Module the Host never added: made on `builder.Services`, it would run in no Module of the Host and fail at its first use. The line names the form on the Module's builder and the form that names the Module ([messaging.md](messaging.md#the-hosts-checks)). Every Module `AddPouryaKitModules` added counts, whatever its Composition Root registers;
  - a subscription, of an Integration Event Handler or a Notification Handler, whose Module registers no Unit of Work, or, in an application without Modules, of the application when it registers none: each delivery runs in its Module's Unit of Work, where its Receipt is recorded, so every one would fail before its handler runs. One line per Module, naming its subscriptions (`… has the subscriptions … but registers no Unit of Work, so each of their deliveries would fail before its handler runs. Declare its data in its Composition Root, for example with UsePouryaKitPostgres.`); a Module that only answers queries needs none ([messaging.md](messaging.md#the-hosts-checks));
  - the routes of the Host's messages: a subscription whose Integration Event or Snapshot has no wire name, a type received from another Host that no subscription or Saga handles, a message type Wolverine would also route where another type goes, a delayed or recurring command whose name a subscription has, and a wire name the process already sends the type under, or one Wolverine gave the type before it was declared ([messaging.md](messaging.md#the-hosts-checks));
  - Notification Handlers, Integration Events, Snapshots or delayed commands declared in a Host whose Unit of Work writes no Outbox, and a Notification Handler in the Feature assemblies of a composed Module, or of the application, that no `AddPouryaKitNotificationHandlers` declares, so it would never run ([messaging.md](messaging.md#the-hosts-checks));
  - a write model whose Unit of Work has no Outbox in a Host that has one: Stream Modules beside the EF message store and `AddPouryaKitMarten`, or EF Core Modules in a Host whose only message store is `UsePouryaKitMarten`, since their Operations would commit and never write what they publish. The line names the Modules and the registration to call ([stream.md](stream.md#limits));
  - a DbContext that `UsePouryaKitPostgres` stores over a connection string naming another database than the message store's (`UsePouryaKitWolverine(connectionName)`), or one the application registered and made a Unit of Work with `AddPouryaKitEfCore<TDbContext>()` on another database, read from the connection EF Core gives the DbContext, unopened and compared on host, port and database as Npgsql reads them: its Outbox is written in its own transaction, so it would never reach the store. The line names the store's connection name, the DbContext's connection name or the DbContext itself, both databases, and the fix for its case: one connection string for the store and every Module's DbContext for `UsePouryaKitPostgres`, registering the DbContext on the store's database otherwise ([messaging.md](messaging.md#the-hosts-checks));
  - a connection string that `UsePouryaKitPostgres`, `AddPouryaKitEfCore<TDbContext>(connectionName)` or `AddPouryaKitDapper` names and the Host's final configuration does not set, or sets blank: the line names the connection and its key, `ConnectionStrings:{name}`; nothing connects ([persistence.md](persistence.md), [dapper-reads.md](dapper-reads.md)). A Host-builder API that has the configuration in hand, such as `UsePouryaKitWolverine(connectionName)` and `UsePouryaKitRabbitMq`, checks its own connection string while the Host is composed: a missing connection string fails at composition where the builder is in hand, when the Host is built otherwise, never at first use.
- **When the Host starts**, before any hosted service starts, Wolverine and the web server included: a `ConfigureContainer` call after `AddPouryaKitModules` replaced the Kit's container, so neither the validation nor the composition checks ran, and `StartAsync` throws, naming the call, what it lost and the fix. Leave `ConfigureContainer` out of a Host of Modules. A Host that registers the dispatcher, as every Kit Host does, but never installs the checks fails here too, naming `AddPouryaKitCompositionChecks()`: `AddPouryaKitModules` and `UsePouryaKitWolverine` call it, and any other Host calls it itself, so no Kit Host runs without its checks. Otherwise the startup checks run (`IStartupCheck`), for what only the built application shows, since it maps its routes after `Build()`, and `StartAsync` throws one `InvalidOperationException` in the same shape:
  - an `IEndpoint` in the Feature assemblies of a composed Module, or of the application, that no `AddPouryaKitEndpoints` declares, or that no `MapPouryaKitModuleEndpoints` mapped, so it would never answer ([web.md](web.md));
  - API versions declared for a route prefix no Module's endpoints are mapped under ([openapi.md](openapi.md)).
- **Once Wolverine has started**: a subscription's or delayed command's queue that does not store what it receives, and a broker listener that does not, or whose dead letters go to a broker queue ([messaging.md](messaging.md#the-hosts-checks)). This is the one check at Wolverine's own moment, because Wolverine compiles its endpoints, applying its policies and the Host's queue configuration, only as it starts.

## Asking another Module a question

Modules talk only in these ways. Inside a Module, Domain Events. Across Modules, asynchronously by default: an Integration Event when a Module reacts to something that happened in another, and a Snapshot when it keeps its own copy of another's data. The owning Module's Notification Handler publishes an Integration Event after its change commits, and its Unit of Work stores a Snapshot in the commit of the change; the receiving Module applies either in an Operation of its own, as Inventory opens a stock item and Billing a price list for each product Catalog creates ([messaging.md](messaging.md#integration-events-between-modules)), and Inventory keeps its copy of Catalog's products ([messaging.md](messaging.md#snapshots-between-modules)). A synchronous question only when the asking Operation genuinely needs the answer before it can go on, such as a permission, membership or availability check. Never HTTP or gRPC between Modules of one Host.

The owning Module publishes the question as an interface in its Contracts project and serves it through its own Gateway. From the Proving Sample, where Catalog answers product questions:

```csharp
// Catalog.Contracts/Products/: public, plain types only
public interface ICatalogProducts
{
    /// <summary>The product with <paramref name="productId"/>, or null when Catalog has no such product.</summary>
    Task<ProductSummary?> FindProductAsync(Guid productId, CancellationToken cancellationToken = default);
}

public sealed record ProductSummary(Guid Id, string Name);

// Catalog/Products/Features/GetProduct/: registered scoped, so it receives the asking Operation's context
internal sealed class CatalogProducts(IGatewayProcessor<CatalogModule> catalog, OperationContext asking) : ICatalogProducts
{
    public async Task<ProductSummary?> FindProductAsync(Guid productId, CancellationToken cancellationToken = default)
    {
        var found = await catalog.QueryAsync(new GetProductQuery(productId), asking.Continue(), cancellationToken);
        if (found.IsSuccess) return new ProductSummary(found.Value.Id, found.Value.Name);

        // Only "no such product" is an answer; any other failure is a bug in the question.
        return found.Errors is [var error] && error == ProductErrors.NotFound
            ? null
            : throw new InvalidOperationException($"Catalog could not answer: {string.Join(", ", found.Errors.Select(e => e.Code))}.");
    }
}

// In Catalog's Composition Root: module.AddScoped<ICatalogProducts, CatalogProducts>();

// Ordering/Orders/Features/AddOrderLine/: Ordering references Catalog.Contracts, never Catalog
internal sealed class AddOrderLineCommandHandler(ICatalogProducts catalog, IRepository<Order, OrderId> orders)
    : CommandHandler<AddOrderLineCommand>
{
    public override async Task<Result> Handle(AddOrderLineCommand command, CancellationToken ct)
    {
        var product = await catalog.FindProductAsync(command.ProductId, ct);
        if (product is null) return OrderErrors.UnknownProduct;

        var order = await orders.FindAsync(new OrderId(command.OrderId), ct);
        if (order is null) return OrderErrors.NotFound;

        return order.AddLine(product.Id, product.Name);
    }
}
```

- Declare the interface and the types it returns in the owning Module's Contracts project, public, with plain values such as `Guid` and records of them, never the Module's internal types. The Proving Sample's `Catalog.Contracts` references no other project. Name its members as questions.
- Implement it in the owning Module, `internal sealed`, and register it scoped in its Composition Root. Resolved in the asking Operation's scope, it receives the asking `OperationContext` by injection. That scope belongs to the asking Module, so the implementation uses nothing of its own Module but its Gateway.
- Answer each question with a query sent through the Gateway with `asking.Continue()`. The answer is then its own Operation in its own scope, through the owning Module's full pipeline, and it changes nothing.
- Map the Result to the Contracts types. Turn only the expected failures into an answer, such as null for "not found"; any other failure is a bug in the question.
- The asking Module's project references the owning Module's Contracts project only, and injects the interface into its handler. A Contracts project references nothing from the Kit and no Module. A structure test rejects a Module that references another Module's implementation, and Contracts that reference the Kit or a Module, even when no code uses the reference ([testing.md](testing.md#structure-tests)).

## An application of one Module

A Host with a single Module may leave out `PouryaKit.Modularity`: it registers the Kit on its own services and opens a scope for each Operation, beginning it itself.

```csharp
// Composition: everything on the application's own services, with no AddPouryaKitModules
builder.Services.UsePouryaKitPostgres<NotesDbContext>(connectionName: "app", schema: "notes");
builder.Services.AddPouryaKitCqrs(typeof(NotesDbContext).Assembly);
builder.Services.AddPouryaKitValidation(typeof(NotesDbContext).Assembly);
builder.AddPouryaKitCompositionChecks();   // the validated container and the composition checks, as in a Host of Modules

// Each Operation: a new scope, begun with its context and no Module, then the dispatcher
await using var scope = services.CreateAsyncScope();
scope.ServiceProvider.BeginOperation(new OperationContext(Actor.System("nightly-notes")));
var written = await scope.ServiceProvider.GetRequiredService<IDispatcher>().SendAsync(new WriteNoteCommand("solo"), ct);
```

- The one Unit of Work is the application's own, so `Operation.Module` stays null and nothing routes. Its registrations name their assembly: the application's own services are no Module's builder, so nothing is inferred, and `AddPouryaKitCqrs()` with no assembly registers the dispatcher alone. Its declarations (`Publish`, `Subscribe`) belong to the application, and so does `AddPouryaKitTenancy(assembly)`, which makes it tenant-scoped ([multi-tenancy.md](multi-tenancy.md)).
- Everything else holds as in a Host of Modules: one scope per Operation, the same pipeline, the same commit. There is no Gateway and no ownership check. `builder.AddPouryaKitCompositionChecks()` turns on what `AddPouryaKitModules` turns on for a Host of Modules: the container validated in every environment (`HostApplicationBuilder` alone validates only in Development) and the composition checks when it is built, among them a command of its Features with no Unit of Work and a declaration that names a Module, which belongs to no Module here, then the startup checks when it starts. `UsePouryaKitWolverine` calls it too. Call it after any `ConfigureContainer` of the Host's own; the monolith template does ([templates.md](templates.md#what-each-composition-composes)).
- `dotnet new pouryakit-monolith` generates such an application: the Proving Sample's Catalog with a plain registration and `IDispatcher` endpoints behind the Kit's HTTP pipeline ([templates.md](templates.md)).

## Testing

- Test a Module through its Gateway against the hosted application, `Fixture.Gateway<TModule>()`, which runs each call as one Operation in its own scope, as the Host does ([testing.md](testing.md#each-test)). Pass an `OperationContext` of your own where the Actor, tenant or correlation matters ([testing.md](testing.md#the-operation-context-in-tests)).
- Test a question to another Module through the asking Module's command: assert what the command did with an answer and with "not found", and that the answering Operation continued the asking one's context.
- Test a mistake in a Host's composition with a Host of the test's own that makes it: assert the exception and its message at the moment its check runs, while the Host is composed, at `Build()` or at `StartAsync`. Compose it in Production, where only the Kit validates the container, when the mistake is one that validation catches.
- Test an application of one Module through `IDispatcher`, in a scope the test begins with `BeginOperation`.
