# Multi-tenancy

Multi-tenancy is a registration across packages the Host already has, not a package: `ITenantScoped` (`PouryaKit.Domain`) marks an aggregate as one tenant's, `PouryaKitDbContext` and the Unit of Work (`PouryaKit.Persistence.EFCore`) read and save it only for the Operation's tenant, `AddPouryaKitTenancy` (`PouryaKit.Cqrs`) makes a Module, or an application without Modules, tenant-scoped, so that every command and query of it needs a tenant, and the delivery step (`PouryaKit.Messaging.Wolverine`) applies no delivery without one there. The tenant is `OperationContext.Tenant`, which the HTTP boundary takes only from a verified claim ([security.md](security.md#jwt-bearer-authentication), [web.md](web.md#the-caller-and-the-actor)).

- [Declaring a tenant-scoped Module](#declaring-a-tenant-scoped-module)
- [A tenant-scoped aggregate](#a-tenant-scoped-aggregate)
- [Reads](#reads)
- [Saves](#saves)
- [Deliveries](#deliveries)
- [What the Kit guarantees](#what-the-kit-guarantees)
- [Limits](#limits)
- [Testing](#testing)

## Declaring a tenant-scoped Module

The Proving Sample's Catalog, first in its Composition Root:

```csharp
public void Configure(ModuleBuilder module)
{
    // First, so the tenant requirement runs before every other authorizer of Catalog's commands and queries.
    module.AddPouryaKitTenancy();
    module.UsePouryaKitPostgres<CatalogDbContext>(connectionName: "app", schema: "catalog");
    // ...
}
```

- `AddPouryaKitTenancy()` registers the tenant requirement, an authorizer on the pipeline's authorization hook, for every concrete command and query of the assembly of the Module's definition, internal ones included, and for every one a handler there handles, wherever it is declared, such as in the Module's Contracts: every command and query the Module can dispatch. An Operation that was begun with no tenant fails with `TenantErrors.Required` (`Tenant.Required`, kind `Forbidden`) before its validator and handler run; `ToHttpResult()` answers it with 403 and the Error in the problem response's `errors`. The scope opener must begin the Operation: in a scope nobody began, the requirement refuses with `Tenant.Required` too, unless another authorizer of the operation, such as a permission check, needs the Operation Context first and throws. It covers only the assemblies it is given: a Module whose `AddPouryaKitCqrs` scans another assembly too fails the Host when it is built (an assembly only another registration names, such as `AddPouryaKitLocalization`, makes nothing dispatchable and does not count), and the composition check `TenantRequirementCheck` names the Module, each command or query it would dispatch without a tenant, and the assembly to name to `AddPouryaKitTenancy`.
- It runs for a trusted continuation of one of those commands and queries too, unlike a permission check: a delayed command's run, a trusted peer's call, a command a Notification or Integration Event Handler sends during its delivery, or `Continue()` inside the asking Operation, as `ICatalogProducts` asks Catalog from another Module. Such a continuation runs with the tenant it carries, or is refused (the one exception to the rule that a trusted continuation is not authorized again). A delivery itself runs the Kit's own command, which is in no Feature assembly, so the requirement does not check it ([Limits](#limits)).
- An Operation's authorizers run in the order registered, so declare it before `AddPouryaKitCqrs` and `AddPouryaKitPermissions`: it then refuses a tenantless Operation before any authorizer of the Module's own reads data.
- An application without Modules declares it on its own services and names its Feature assemblies, as it does for `AddPouryaKitCqrs`: `services.AddPouryaKitTenancy(typeof(Program).Assembly)`. With no assembly there it throws `ArgumentException`, since it would cover nothing.
- The declaration is recorded for the Host: `HostComposition.IsTenantScoped(module)`, null for the application's own services, tells a Kit registration or composition check which Modules are tenant-scoped.
- Two declarations fail the Host when it is built in a tenant-scoped Module. A recurring command (`RecurringCommandTenancyCheck`): its occurrences run for a system Actor with no tenant, and the Kit runs none per tenant, so declare recurring work in a Module that is not tenant-scoped, as the Proving Sample's Inventory does; a delayed command is fine, since it continues the tenant of the Operation that scheduled it. A Snapshot of an aggregate of the Module's own DbContext whose mapped key neither includes its `Tenant` nor is a single Guid (`SnapshotKeyTenancyCheck`, with `UsePouryaKitWolverine(connectionName)`): the Snapshot's identity and version key hold the aggregate's key and no tenant, so key such an aggregate by a Guid version 7, as Catalog's products are, or by its tenant and its own code.

## A tenant-scoped aggregate

```csharp
internal sealed class Product : AggregateRoot<ProductId>, IAuditable, ISoftDeletable, ITenantScoped
{
    public string Tenant { get; private set; } = "";

    public static Result<Product> Create(ProductId id, CategoryId categoryId, string name, string tenant)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(tenant);
        // ...
        var product = new Product { Id = id, Tenant = tenant, CategoryId = categoryId, Name = name.Trim() };
        // ...
    }
}

// Its handler takes the tenant from the Operation Context. Catalog is tenant-scoped, so it always has one.
internal sealed class CreateProductCommandHandler(IRepository<Product, ProductId> products, OperationContext context)
    : CommandHandler<CreateProductCommand, Guid>
{
    // ... Product.Create(new ProductId(command.ProductId), new CategoryId(command.CategoryId), command.Name, context.Tenant!)
}
```

- The aggregate sets its own `Tenant`, from the value its handler takes from `OperationContext.Tenant`; the Kit never sets it. Declare it with a private setter, as `IAuditable`'s properties are. Its column is `tenant`, snake-cased like every other.
- Implement `ITenantScoped` on an aggregate or entity that is not mapped as an owned type, at the root of its inheritance hierarchy or below a root that implements it too, as `ISoftDeletable` is: EF Core filters only those, so the model refuses to build otherwise, naming the type.
- Map a part's relationship to its holder on the root of the part's hierarchy, and never let parts hold one another in a cycle: the model refuses either, naming the types, because no filter could leave out another tenant's parts, or the filters would read each other without end. A part may hold parts of its own type only with a required key to its aggregate or another holder: the model refuses it otherwise.
- A unique value is unique per tenant: Catalog's name index is on `(tenant, name)`, so two tenants each have a product of one name, and one tenant's name tells another nothing.
- Make every aggregate of a tenant-scoped Module that holds a tenant's data tenant-scoped. Catalog's `Category` is, because it counts the tenant's products.

## Reads

- Every EF Core query of an `ITenantScoped` entity returns only the rows of the tenant of the Operation running it, through a named query filter, `PouryaKitDbContext.TenantFilter` (`PouryaKit.Tenant`), beside `SoftDeleteFilter`. The repository's `FindAsync`, `ListAsync`, `AnyAsync` and `CountAsync` and a specific repository's projection all read through it, so another tenant's aggregate is not found. A part of a tenant-scoped aggregate, at any level, has no tenant of its own, so the same named filter reads it only where its holder is read: an EF Core query of its own set (`Set<Sheet>()`) returns only the parts whose holder the Operation's tenant reads, by id or in a list, and a part of a removed holder is left out with it. A key to a part of its own type never judges the part, because EF Core reads that set unfiltered there; a null optional key to a holder does not judge the part while another key names a holder; a part with no holder is read by nobody; set the filter aside to find orphans.
- The filter reads the tenant when each query runs, from the scope's Operation, never when the model is built or the DbContext created: the migration tool and the composition checks build a DbContext where no Operation began, and a DbContext made before its scope's Operation began reads that Operation's tenant. Where the Operation has no tenant, or none began, the filter reads no rows.
- A read that must cross tenants sets the filter aside on purpose, in an EF Core query over the DbContext: `IgnoreQueryFilters([PouryaKitDbContext.TenantFilter])`. The repository's members never do. Only that filter is set aside: a removed row stays out unless the read names `SoftDeleteFilter` too.
- Dapper runs exactly the SQL you write, so a plain SQL read of a tenant-scoped table states `tenant = current_setting('pouryakit.tenant')` itself, as Catalog's `GetProduct` and `ListProducts` do: the Kit's Dapper lease sets that session setting to the tenant of the scope's Operation when it opens, or to the empty string, which matches no row, where the Operation has none, so the handler binds no tenant and cannot bind the wrong one. The read keeps to the one shape a structure test holds every such read to, which rejects `tenant = @Tenant` or any other right-hand side: the predicate a top-level conjunct of the `WHERE` of one `SELECT`, for each tenant-scoped table it reads ([dapper-reads.md](dapper-reads.md#reading)). A part's table has no tenant column, so a read of it joins it on its key to its holder, up to the aggregate's root held so.
- A Module that is not tenant-scoped and keeps a tenant on its rows reads with the same predicate: Inventory's `GetStockItem` reads `inventory.stock_items` by product id and `tenant = current_setting('pouryakit.tenant')`, so another tenant's Actor and a tenantless Operation get `StockItem.NotFound` ([Deliveries](#deliveries)). Its table is no `ITenantScoped` entity's, so no structure test holds the read to that shape and no filter or save check covers it: the predicate is the isolation.

## Saves

- The Unit of Work's prepare step refuses the save with `TenantErrors.Mismatch` (`Tenant.Mismatch`, kind `Forbidden`) and rolls the Operation back when an `ITenantScoped` aggregate it added belongs to another tenant than the Operation's, or one it changed or removed was loaded under another tenant, or had its tenant changed. It judges every aggregate the step attributes a change to: one whose part changed, both holders of a part moved between aggregates, and the parts a removal deletes with it. A removed soft-deletable aggregate is judged by the tenant it was loaded under, before it becomes a mark.
- So a read that sets `TenantFilter` aside may read another tenant's rows, but an Operation can never save a change to them: it answers 403 over HTTP. It holds wherever an `ITenantScoped` aggregate is saved, as the filter does, in a Module that is tenant-scoped or not, and an Operation without a tenant saves none.
- A tenant-scoped aggregate is loaded, never attached. An entity attached or updated rather than loaded carries the key and tenant its code gave it, so neither tells whose row its write would reach. The prepare step therefore refuses with `TenantErrors.NotLoaded` (`Tenant.NotLoaded`, kind `Forbidden`) a save that changes or removes an `ITenantScoped` entity, or a part of one, that no query of the DbContext loaded, and one that changes an aggregate a change is attributed to that no query loaded; an added entity is new, not attached. An entity a query loaded counts as loaded only while that query's tracking of it lasts: detached, or cleared from the change tracker, and then attached again, it is attached. Load through the repository or an EF Core query. A tenant's aggregate whose own tenant names another tenant fails first with `Tenant.Mismatch`.
- Behind both refusals, a write that reaches the database is held to its tenant too: the model convention makes every tenant-scoped root entity's `Tenant` a concurrency token beside `Version`, so each UPDATE and DELETE of such a row matches it only under the tenant the entity was loaded with, and one under another tenant fails with `UnitOfWork.ConcurrencyConflict`. A part has no tenant, so its key to the entity that holds it is a concurrency token instead, at every level of a tenant-scoped aggregate: a loaded part's write matches its row only under the holder it was loaded with, the one the prepare step judged, so a part whose holder as loaded the Operation rewrote, by setting the key's original value or by accepting the change tracker's changes, fails the same way. A part moved between two of the tenant's aggregates is saved as before.

## Deliveries

- A Notification, Integration Event or Snapshot delivered into a tenant-scoped Module runs under the tenant it carries, the one the publishing Operation had, so its handler writes for that tenant: Inventory, which is not tenant-scoped, keys its copy of Catalog's products by `(tenant, product_id)`, from the injected `OperationContext`.
- One that carries no tenant is refused before its Receipt and before its handler, whatever the handler writes with, plain SQL included: it fails with `Tenant.Required`, is retried, then set aside as a dead letter whose message names it. The mistake is the publisher's, an Operation with no tenant whose work reaches a tenant-scoped Module; a replay is refused the same way.
- A Module that is not tenant-scoped applies a delivery with or without a tenant and keeps the delivered tenant itself, as a plain property the Kit neither requires nor filters. Two Modules of the Proving Sample show it: Billing derives each welcome quote's id from the delivered tenant and the account (`OpenWelcomeQuoteIntegrationEventHandler.QuoteIdOf`), and Inventory records it on each stock item it opens, `StockItem.Tenant`, nullable, from the injected `OperationContext`, which its `GetStockItem` endpoint then answers only to an Operation of that tenant ([Reads](#reads)). A stock item a tenantless delivery opened keeps no tenant and is answered to no Operation. Inventory's recurring reconciliation, which runs for a system Actor with no tenant, still counts every tenant's stock items, which a tenant-scoped Module could not declare (`RecurringCommandTenancyCheck`).

## What the Kit guarantees

- In a tenant-scoped Module, no command or query runs without a tenant, a trusted continuation included; it fails with `Tenant.Required` before its validator and handler.
- An EF Core read of a tenant-scoped entity, or of a part of one through its own set, returns only the Operation's tenant's rows; a cross-tenant read names `TenantFilter`.
- An application without Modules that declares it on its own services gets the same requirement for the commands and queries of the assemblies it names.
- No Operation saves a change to another tenant's aggregate or to a part of one, or moves one between tenants, and nothing is saved: one it loaded fails with `Tenant.Mismatch`, and a tenant-scoped aggregate or part it attached instead of loading fails with `Tenant.NotLoaded` before the save, whoever's key it carries.
- A tenant-scoped Module applies no delivery without a tenant; a delivery with one runs under it.
- A tenant-scoped Module composes no recurring command and no Snapshot whose key two tenants could share.

## Limits

- An anonymous caller has no tenant, so no endpoint of a tenant-scoped Module serves one: Catalog's `ListProducts` is authenticated, like every Catalog endpoint. A tenant comes only from a verified claim; a tenant taken from the host name or the route is not supported.
- A plain SQL write is the Feature's too: no save check sees it. A structure test rejects SQL in a Module that writes a table an aggregate maps, its owned parts and child entities included ([testing.md](testing.md#structure-tests)), so plain SQL writes only a Module's own tables that no aggregate maps, and writes the tenant such a table keeps itself, as Inventory's copy of Catalog's products does.
- Feature code that assembles the session tenant setting at run time is outside the structural tenant rules: the structure test that leaves `pouryakit.tenant` to the lease rejects any SQL literal that writes session state, one holding `set_config` or a statement that starts with `SET` or `RESET`, not such a statement built from fragments at run time, nor a database function the SQL calls that rebinds the setting. The Kit trusts Feature code within the Host.
- A row written before its table kept the tenant cannot learn it later: a Module's own rows from before it was tenant-scoped have no tenant to recover, and a copy of another Module's data cannot be replayed, since the Snapshot versions already applied stop it. The migration that adds the tenant refuses a table that holds such a row rather than guess its tenant, as Catalog's `Tenants` and Inventory's `CatalogProductTenants` migrations do; an empty tenant would hide the row from every Operation and fail any change to it. The Kit has no backfill.
- Row-level security in PostgreSQL is not used, and the Receipt and Snapshot version keys hold no tenant.

## Testing

- Prove isolation with two tenants at the Gateway: create in tenant A and in tenant B with `new OperationContext(actor, "a")`, then show that A's get, list, change and removal of B's aggregate find nothing.
- Prove a Dapper read the same way, through the Gateway one tenant after the other: A's read finds none of B's rows, B's, on a lease taken after A's from the same pool, finds B's, and an Operation without a tenant finds none.
- Over HTTP the tenant is the authenticated caller's claim: the test scheme issues it from its tenant header, `AppFixture.Client(caller, tenant, permissions)`.
- `GatewayTestExtensions`' calls without a context run with no tenant, so a tenant-scoped Module refuses them; pass a context with a tenant. The Proving Sample's tests add `CatalogGatewayExtensions`, the same calls for Catalog in a test tenant.
- Show the requirement with a context without a tenant, and for a continuation with a context continued inside an Operation of a Module that is not tenant-scoped.
- Show the save check with two tenants: load the other tenant's aggregate with `TenantFilter` set aside in a handler, change its root, a part, move a part in or out, remove it, and expect `TenantErrors.Mismatch` with its rows unchanged; make the same change to the Operation's own tenant's to show it saves. Show the refusal of attached entities the same way: build an aggregate with a stored key and version and the Operation's tenant, or a part with the other tenant's part key under the Operation's own loaded aggregate, attach it and change it, update it or remove it, and expect `TenantErrors.NotLoaded` with the rows unchanged.
- Show a delivery refusal by sending to the subscription's queue with the Kit's headers and no tenant, then reading the dead letter.
- Show a delivered tenant kept by a Module that is not tenant-scoped end to end: create the product in tenant A in the Catalog Host of `BrokerFixture`, wait for `Inventory.Host` to open its stock item, then read it over HTTP with bearer tokens of the tests' issuer, A's answered, B's and a token without the tenant claim `NotFound`.
