# Testing

`PouryaKit.Testing`: PostgreSQL in a container, data reset between tests, Operations sent through a Gateway, waiting for and asserting deliveries, reading and replaying dead letters. Namespace `PouryaKit.Testing`, on xUnit v3; the fixture needs Docker. The structure rules every Module follows are stated for a project to write as its own tests: no package ships them, and the Kit's own tests are not published ([Structure tests](#structure-tests)).

- [The fixture](#the-fixture)
- [Each test](#each-test)
- [The Operation Context in tests](#the-operation-context-in-tests)
- [Time](#time)
- [Deliveries](#deliveries)
- [Dead letters](#dead-letters)
- [Structure tests](#structure-tests)

One fixture for the test assembly and a test against the hosted application, after the Proving Sample. A generated solution already has this fixture, the `App` of its test project, and its smoke test shows how a test gets a token ([templates.md](templates.md#the-smoke-test)): extend those.

```csharp
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.TestHost;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using ModularMonolith.Host;
using PouryaKit.Testing;

[assembly: AssemblyFixture(typeof(AppFixture))]
[assembly: CollectionBehavior(CollectionBehavior.CollectionPerAssembly)]

public sealed class AppFixture : HostFixture
{
    protected override IHost CreateHost(string connectionString)
    {
        var builder = ModularMonolithHost.CreateBuilder(
            // The Host's bearer scheme is checked when it starts, so it gets the test issuer's settings (security.md).
            ["--environment", "Development", $"--ConnectionStrings:app={connectionString}", .. TestIssuer.HostArguments]
        );
        builder.Services.UsePouryaKitTestDelivery();   // deliveries at test speed (below)
        builder.WebHost.UseTestServer();   // served in memory, for HTTP tests (web.md)
        builder
            .Services.AddAuthentication(TestAuthentication.SchemeName)
            .AddScheme<AuthenticationSchemeOptions, TestAuthentication>(TestAuthentication.SchemeName, null);
        return ModularMonolithHost.Build(builder);
    }
}

public sealed class UnitOfWorkTests(AppFixture app) : HostTest<AppFixture>(app)
{
    [Fact]
    public async Task A_command_that_raises_no_domain_event_is_committed_though_its_handler_never_saves()
    {
        var id = Guid.CreateVersion7();

        var created = await Fixture
            .Gateway<CatalogModule>()
            .SendAsync(new CreateCategoryCommand(id, " Lamps "), TestContext.Current.CancellationToken);

        Assert.Equal(id, created.Value);
    }
}
```

## The fixture

- Derive one `HostFixture` per test assembly and register it with `AssemblyFixture`. Its `CreateHost(connectionString)` builds the application's Host, not started, with that connection string for its Modules. Build it with the same builder and pipeline methods `Program` uses, so the tests run the real composition, with the container validated on build as `AddPouryaKitModules` does in every environment. A Host that serves HTTP is a `WebApplication`, which is an `IHost`; serve it in memory and give it a test authentication scheme, as above, to test it over HTTP ([web.md](web.md#testing-over-http)).
- Before the first test, the fixture runs PostgreSQL in a container, applies every Module's migrations to it, Receipt tables included, and then starts the application on it, so Wolverine finds every table when it starts. After the last test it stops both.
- `CreateHost` may add test-only Modules beside the application's with a second `builder.AddPouryaKitModules(...)` call.
- A Host that needs another container before it is composed, such as the service template's RabbitMQ, whose connection string `UsePouryaKitRabbitMq` reads as the Host is composed: the fixture implements `IAsyncLifetime` again, starts that container, then calls `HostFixture`'s own `InitializeAsync`, and disposes it after the fixture's `DisposeAsync` ([templates.md](templates.md#the-smoke-test)).
- A Host without Modules, such as the monolith template's, uses the same fixture: `HostFixture` hosts any `IHost` and migrates every registered DbContext, and only `Gateway<TModule>()` needs a Module, so such tests go through HTTP or `IDispatcher`. `PouryaKit.Testing` brings `PouryaKit.Modularity` into the test project only; the Host's own closure stays without it ([templates.md](templates.md#the-smoke-test)).
- The test project references `PouryaKit.Testing` and the Host project, `Microsoft.AspNetCore.TestHost` for HTTP tests, and each Module grants it its internals: `<InternalsVisibleTo Include="ModularMonolith.Tests" />` in the Module's project. `PouryaKit.Testing` brings `WolverineFx.RuntimeCompilation`, which the Host references in Debug builds only, so a fixture's Host runs Development, where Wolverine compiles its handler code at run time, from a Release build too ([messaging.md](messaging.md#declaring-handlers-and-running-deliveries)). A Host the tests start in another environment loads generated code instead and needs none for the Kit's deliveries.

## Each test

- Derive each test class from `HostTest<TFixture>`. Before each test it waits until every delivery an earlier test caused has finished, then deletes every row in every schema except the migrations history, and every stored delivery, so no test sees another's data or deliveries. Tests that share a fixture run one at a time: `CollectionBehavior.CollectionPerAssembly`.
- The stored-delivery reader quotes the message-store schema, including embedded double quotes. This does not extend Wolverine's schema support: on WolverineFx 6.46.0, not checked since, Wolverine fails to start for names such as `message-store` in both Solo and Balanced mode. Use a schema Wolverine can start with.
- Send Operations with `Fixture.Gateway<TModule>()`, exactly as the application does, each in its own scope. Assert on the returned Result and its `Errors`; read rows back with your own SQL on `Fixture.ConnectionString`. `Fixture.Host` is the started Host.
- Test an endpoint with HTTP requests through `Fixture.Host.GetTestClient()`, authenticated as the caller the test names; see [web.md](web.md#testing-over-http).

## The Operation Context in tests

- A test that does not care who runs its Operations calls the Gateway without a context, as above: `SendAsync(command, ct)` and `QueryAsync(query, ct)` are extension members in `PouryaKit.Testing` (`GatewayTestExtensions`), so the file needs `using PouryaKit.Testing;`. Each call runs as a new Operation of `GatewayTestExtensions.TestActor`, the named system Actor `Actor.System("PouryaKit.Testing")`. Application code never references `PouryaKit.Testing`, so it always names its context. Such an Operation has no tenant, so a tenant-scoped Module refuses it: pass a context with a tenant there ([multi-tenancy.md](multi-tenancy.md#testing)).
- A test that needs a particular Actor, tenant or correlation creates the `OperationContext` and passes it to the Gateway's own methods:

```csharp
[Fact]
public async Task The_context_a_caller_passes_to_a_gateway_is_the_context_the_handler_reads()
{
    var context = new OperationContext(Actor.Person("ada"), tenant: "acme");

    var read = await Fixture
        .Gateway<StubModule>()
        .QueryAsync(new ReadOperationContextQuery(), context, TestContext.Current.CancellationToken);

    Assert.Equal(context, read.Value);
}
```

- A Module's Gateway refuses a command or query that a test-only Module owns. To run a test-only probe inside an application Module's Unit of Work, open a scope from `Fixture.Host.Services`, begin it with `BeginOperation(new OperationContext(GatewayTestExtensions.TestActor, "tests"), typeof(CatalogModule))`, with a tenant since Catalog is tenant-scoped, and send the probe through that scope's `IDispatcher`, as a delivery step would.

## Time

- For domain code the test runs directly, create a `TestClock` with `using` inside the test method; see [domain-primitives.md](domain-primitives.md).
- For Operations sent through a Gateway, register a `FakeTimeProvider` as the Host's `TimeProvider` in `CreateHost` (`builder.Services.AddSingleton<TimeProvider>(time)`) and advance it from the tests.

## Deliveries

A test of the deliveries one Operation causes, after the Proving Sample, whose Catalog announces each new product and publishes `ProductCreatedV1`, which Inventory and Billing each handle, and `ProductCreatedV2`, which a second Inventory subscription handles:

```csharp
[Fact]
public async Task A_new_product_is_announced_by_the_host_after_it_is_committed()
{
    var catalog = Fixture.Gateway<CatalogModule>();
    var category = Guid.CreateVersion7();
    var product = Guid.CreateVersion7();
    await catalog.SendAsync(new CreateCategoryCommand(category, "Lamps"), TestContext.Current.CancellationToken);

    var (created, deliveries) = await Fixture.WaitForDeliveriesAsync(
        () => catalog.SendAsync(new CreateProductCommand(product, category, "Lamp"), TestContext.Current.CancellationToken),
        TestContext.Current.CancellationToken
    );

    Assert.True(created.IsSuccess);
    Assert.Equal(
        [
            new Delivery("billing.product-created", "catalog.product-created.v1", false),
            new Delivery("catalog.product-created.announce-new-product", "catalog.product-created", false),
            new Delivery("catalog.product-created.publish-product-created", "catalog.product-created", false),
            new Delivery("inventory.product-created", "catalog.product-created.v1", false),
        ],
        deliveries
            .Where(delivery => delivery.WireName is "catalog.product-created" or "catalog.product-created.v1")
            .OrderBy(delivery => delivery.Subscription, StringComparer.Ordinal)
    );
}
```

The two `catalog.` deliveries are the Notification's, stored in the product's commit; the other two are the Integration Event's, one per subscription, stored in the commit of `publish-product-created`'s delivery. The wait includes deliveries that deliveries caused, and their order is not fixed, so the test sorts them. It keeps only these two wire names, so two deliveries are left out: the v2 Integration Event's, `inventory.product-categories` under `catalog.product-created.v2`, stored in the same commit as v1's ([messaging.md](messaging.md#a-new-version-of-an-integration-event)), and the product's Snapshot for Inventory's copy, `inventory.catalog-products` under `catalog.product-snapshot.v1`, stored in the product's commit ([messaging.md](messaging.md#snapshots-between-modules)).

- In `CreateHost`, call `builder.Services.UsePouryaKitTestDelivery()` after `UsePouryaKitWolverine`, which the application's builder method calls, as the fixture above does. Deliveries then run at test speed: Solo durability, so a stopped Host's deliveries are recovered at once; a failed delivery retried twice, 100 ms apart (`TestDeliveryServiceCollectionExtensions.RetryDelays`), so three attempts, and then set aside as a dead letter; and the message store polled every 100 ms (`PollingTime`). Count attempts from the setting, `1 + TestDeliveryServiceCollectionExtensions.RetryDelays.Count`, rather than a literal.
- `Fixture.WaitForDeliveriesAsync(operation, ct)` runs the Operation, typically a Gateway call, waits until every delivery it caused is handled or set aside as a dead letter, including deliveries those deliveries caused, and returns the Operation's outcome with one `Delivery(Subscription, WireName, DeadLettered)` per delivery, in no particular order. A failing delivery stays pending through every retry, so the wait ends only once it is set aside, and it comes back with `DeadLettered` true. It fails with a `DeliveryAssertionException` when the Operation stored no delivery at all, so a test that waits never passes on nothing, and with a `TimeoutException` when deliveries are still pending after 30 seconds.
- After it returns, assert the handler's effect: the rows it wrote, such as Inventory's stock item, or what the Host's implementation of the port recorded.
- `Fixture.AssertNothingSentAsync(operation, ct)` runs the Operation and fails unless it sent nothing: what a rolled-back Operation leaves. It fails on a delivery the Operation stored for a subscription of the Host, on an envelope Wolverine handed to a sending agent for a broker while the Operation ran, counted by Wolverine's `wolverine-messages-sent` instrument on the Host's own `Meter`, since the store deletes it once sent, and on one still stored for a broker afterwards, such as while the broker is unavailable. The message names each: the subscription, or the wire name and the address. Everything is stored and handed out before the Operation returns, so there is nothing to wait for.
- `Fixture.WaitUntilDeliveredAsync(ct)` waits until no stored delivery is pending, such as after a test made the store hand one out again. A dead letter marked for replay counts as pending until the store has moved it back and it is handled or set aside again. A delayed command not yet due, a recurring command's next occurrence included, is not pending, so neither wait holds for it: a test that needs its effect waits for that effect once it is due.
- Between tests, the reset waits for every pending delivery, then deletes every stored delivery, dead letters and delayed commands not yet due included. The message store's own tables, such as its node records, stay: they belong to the running Host.
- A test that starts a second Host of the application, for example to prove that a Notification committed by a Host that died is delivered by the next, starts it with `HostFixture.StartHostAsync(host, ct)`, which applies the migrations first, and stops or disposes it before the test ends. Compose it with its own `RetryDelays` or `InboxStaleTime` to test a setting, keeping [the stale-time rule](messaging.md#the-stale-time-rule) unless the test is about breaking it.
- Every Host composed with `UsePouryaKitTestDelivery` runs in Solo mode as the same node of the message store, the fixture's Host included. On WolverineFx 6.46.0, not checked since, Wolverine gives a Solo Host node number 1 and releases every live Host's rows when one starts, and its recovery takes a row owned by no node without checking that no other Host took it first. So when a test abandons a Host in the middle of a delivery, two live Hosts on the same store, such as the fixture's and the one that replaces it, can each recover that delivery at the same moment. The Receipt keeps its effect to one, but the delivery may be attempted once more. A test that abandons a Host asserts the effect, or a range of attempts, never an exact number of attempts.

## Dead letters

A test of a delivery that keeps failing, after the Proving Sample's tests. Their test-only Module publishes one Integration Event to two subscriptions, `Circulation.Archive` and `Circulation.Ledger`, and a test makes one handler return a failed Result for its note until it heals it:

```csharp
[Fact]
public async Task A_delivery_that_keeps_failing_is_set_aside_and_replayed_once_its_cause_is_fixed()
{
    var circulated = new NoteToCirculateDomainEvent($"note {Guid.CreateVersion7()}");
    Circulation.Fail(Circulation.Archive, circulated.Text);

    var (_, deliveries) = await Fixture.WaitForDeliveriesAsync(
        () =>
            Fixture
                .Gateway<StubModule>()
                .SendAsync(new WriteNoteRaisingCommand(circulated.Text, [circulated]), TestContext.Current.CancellationToken),
        TestContext.Current.CancellationToken
    );

    // Only the failing subscription's delivery is set aside, with the Errors of its failed Result.
    Assert.Contains(new Delivery(Circulation.Archive, "stub.note-circulated.v1", true), deliveries);
    Assert.Contains(new Delivery(Circulation.Ledger, "stub.note-circulated.v1", false), deliveries);
    var deadLetter = Assert.Single(await Fixture.DeadLettersAsync(TestContext.Current.CancellationToken));
    Assert.Equal(typeof(IntegrationEventHandlerFailedException).FullName, deadLetter.ExceptionType);
    Assert.Contains($"{Circulation.Refused.Code} ({Circulation.Refused.Message})", deadLetter.ExceptionMessage);

    // The cause is fixed, then the dead letter is replayed: only the archive runs again.
    Circulation.Heal(Circulation.Archive, circulated.Text);
    await Fixture.ReplayAsync(deadLetter, TestContext.Current.CancellationToken);

    Assert.Empty(await Fixture.DeadLettersAsync(TestContext.Current.CancellationToken));
    Assert.Single(Circulation.RunsOf(Circulation.Ledger, circulated.Text));
}
```

- `Fixture.DeadLettersAsync(ct)` reads every dead letter through Wolverine's dead-letter storage, as an application inspects it ([messaging.md](messaging.md#dead-letters)), each as a `DeadLetter(Id, Subscription, WireName, ExceptionType, ExceptionMessage)`: the subscription it was received at, the wire name it was stored under, and the exception of its last attempt. A Host that receives from other Hosts over a broker can also set aside a delivery that failed at its broker queue, before it reached any subscription; its `Subscription` is that queue's address, such as `rabbitmq://queue/inventory`, and a `Delivery` received there, before its fan-out, is named the same way ([rabbitmq.md](rabbitmq.md#dead-letters-at-the-broker-queue)). Each test starts with none, so these are what its Operations caused. For a failed Result the exception is `NotificationHandlerFailedException` or `IntegrationEventHandlerFailedException` (`PouryaKit.Messaging.Wolverine`), whose message names each Error as `Code (Message)`; for a handler that threw, the exception it threw.
- `Fixture.ReplayAsync(deadLetter, ct)` replays one dead letter through Wolverine, which moves it back to its subscription's queue, where only that subscription's handler runs, or, for one set aside at the broker queue, back to that queue's fan-out, and waits until it is handled or set aside again. It fails with a `DeliveryAssertionException` when the delivery is not a dead letter, so a replay never passes on nothing.
- Assert what the replay changed: the handler's effect, and that the other subscriptions of the message did not run again.

## Structure tests

The Kit proves the boundaries of its Proving Sample with architecture tests. Those tests are not published, no package ships the rules, and a generated solution holds none of them: a consuming project that wants the same protection writes an architecture test project of its own (below). Either way, these are the rules every Module follows.

### The rules

| Rule | What the Kit's test rejects | What it cannot see |
| - | - | - |
| A Module reaches another only through that Module's Contracts; Contracts reference nothing from the Kit and no Module ([modules.md](modules.md#asking-another-module-a-question)) | A reference to another Module's implementation, or from Contracts to a Kit package or a Module, anywhere in the project's restored closure, direct or through other projects, even when no code uses it | An assembly loaded, or a service resolved, by name at run time; a reference to a Module whose assembly name differs from its project name |
| A Module's only public types are its Composition Root, the ports its Host implements with the records they carry, EF Core migrations, and its Sagas and their timeouts, derived from the Kit's `PouryaKitSaga` and `SagaTimeout` ([modules.md](modules.md#the-composition-root), [sagas.md](sagas.md#writing-a-saga)) | Any other exported type, a saga or a timeout of Wolverine's own included; the test lists each Module's ports by name | Internal types opened with `InternalsVisibleTo` |
| Feature code never saves, commits or writes around the Unit of Work ([commands-queries-validation.md](commands-queries-validation.md#commands-and-queries)) | A call to `SaveChanges(Async)`, `DbContext.Database`, EF Core's `ExecuteUpdate(Async)` or `ExecuteDelete(Async)`; beginning, committing, rolling back or releasing a transaction or savepoint on an ADO.NET connection or transaction or an EF Core transaction; a new `TransactionScope` or `CommittableTransaction`; a step only the Kit runs (`IUnitOfWork`, the Unit of Work's DbContext through `EfUnitOfWorkDbContext.Of`, the Marten Unit of Work's transaction through `MartenUnitOfWorkTransaction.ExecuteAsync`, `IUnitOfWorkOutbox`, `IStreamOutbox`, `IReceipts`, `IIntegrationEventOutbox`, `IDelayedCommandOutbox`, `IIdempotencyStore`, `ISnapshotMappers`, `DomainEventDrain`, `IReadingStateGuard`, `TrackedState`, the query cache `IQueryCache`, `BeginOperation`, `Operation.OnlyReading`); a string literal holding a transaction-control statement (`begin`, `start transaction`, `commit`, `end`, `abort`, `rollback`, `savepoint`, `release`, `prepare transaction`, `commit prepared`) at any statement boundary, read without its comments, the bodies of its single-quoted strings and its dollar-quoted blocks | A call through reflection, `dynamic` or an expression compiled at run time; a call a library makes for the Module; EF Core's transaction manager reached without `Database` (`GetService<IDbContextTransactionManager>()`); SQL assembled at run time; transaction control inside a dollar-quoted block; a write on a connection outside the Unit of Work to a table no aggregate maps. A literal counts as SQL only when every statement in it starts with a SQL command word, so one that reads as a single such statement, a label reading "End" or "Release notes", is rejected as if it were SQL |
| An application never implements a contract the Kit fills for its pipeline, so it adds no pipeline step under another name ([commands-queries-validation.md](commands-queries-validation.md#the-pipeline)) | A class or struct in a Host or in any project a Host references that is no Kit package, nested and compiler-generated ones included, that implements `IDispatcher` (a decorator too), `IGatewayProcessor<TModule>`, `IUnitOfWork`, `IValidationStep<T>`, `IReadingStateGuard`, `IUniqueConstraintReader`, `IIdempotencyStore`, `IQueryCache`, `IUnitOfWorkOutbox`, `IStreamOutbox`, `IOutbox`, `IIntegrationEventOutbox`, `IDelayedCommandOutbox`, `ISnapshotMappers` or `IReceipts`; the Kit's own packages are where each is implemented | An implementation made at run time, such as a `DispatchProxy` |
| No transport or store type in a Module or its Contracts, so never Marten's document session ([messaging.md](messaging.md#writing-a-notification-handler), [stream.md](stream.md#reading-and-changing-streams)) | Any type from a Wolverine, JasperFx, Marten or Weasel assembly that the code, a signature or an attribute names, but the four JasperFx types Marten's source generator names in the evolver it writes into a Stream Module's assembly (`JasperFx.Events.Aggregation.GeneratedEvolverAttribute`, `JasperFx.Events.Aggregation.IGeneratedSyncEvolver`, `JasperFx.Events.Aggregation.JasperFxSourceGeneratorAppliedAttribute` and `JasperFx.Events.IEvent`), and Wolverine's `TimeoutMessage`, which the compiler names for a timeout record, when every timeout of the assembly derives from the Kit's `SagaTimeout` | A transport or store type reached by name at run time, or held only as `object` or `dynamic`; Feature code that names one of the generator's four types itself, or `TimeoutMessage` in an assembly whose timeouts are all the Kit's |
| No envelope storage in a Module's DbContext | An entity type from a Wolverine or JasperFx assembly, or a table named `wolverine_*` | An envelope table mapped by an entity of the Module's own under another name |
| Only the Kit clears Domain Events ([domain-primitives.md](domain-primitives.md#aggregates-and-entities)) | A call to `ClearDomainEvents` on any aggregate type | A call through reflection or `dynamic` |
| No SQL write to a table an aggregate maps ([dapper-reads.md](dapper-reads.md#writing-with-plain-sql-in-the-unit-of-work)) | `insert into`, `update`, `delete from`, `merge into` or `truncate` in a string literal, naming a table that any Module's aggregate, owned part or child entity maps, with or without its schema, whatever the quotes, case or whitespace around the schema's dot; comments and the bodies of single-quoted strings are left out, and a dollar-quoted block is read as code | SQL built at run time (an interpolated or concatenated table name, text read from a file or resource); a three-part name (`database.schema.table`); a write through a view; writes inside stored procedures, functions or triggers the literal only calls. A write on a connection outside the Unit of Work to a table no aggregate maps is allowed by design |
| In a tenant-scoped Module, every SQL read of a tenant-scoped table holds each such table to the Operation's tenant ([dapper-reads.md](dapper-reads.md#reading)) | In a Module the Host records as tenant-scoped, a string literal that names a table an `ITenantScoped` entity maps, or the table of a part a navigation holds in a tenant-scoped aggregate, anywhere in its text and holds a SQL command word, `from` or `join`, unless its tokens are one `SELECT list FROM item ((JOIN \| INNER JOIN \| LEFT JOIN) item ON condition \| , item)* WHERE conjunct (AND conjunct)* [ORDER BY list] [LIMIT @p \| n] [OFFSET @p \| n]`, each item `[schema.]table [[AS] alias]` with nothing after it, with no other `SELECT` or `TABLE`, no set operator, no `OR` or `CASE` outside parentheses in a condition or a conjunct, and a conjunct exactly `p.tenant = current_setting('pouryakit.tenant')` for each such item by its alias, or by its name when it has none, or `tenant = current_setting('pouryakit.tenant')` when the `FROM` has that one item (a parameter such as `@Tenant` fails), and each part's item joined to a held item of its holder by `part.column = holder.column` for every column of its key, in the part's `ON` or in the holder's when that is no `LEFT JOIN`; a block comment, a `$`, a backslash or a character the tokeniser does not classify fails too. A literal that writes such a table is the SQL-write rule's above | SQL read from a file or resource, or assembled from fragments none of which holds both a SQL word and the table's name; a read through a view or a function; whether the setting holds the Operation's tenant, which the lease sets and a Gateway test with two tenants shows |

The rules about saving, SQL writes, tenant reads and public types leave out a Module's EF Core migrations and model snapshots, which `dotnet-ef` writes; the others read the whole assembly.

### Writing your own

- Write each rule as a function from a compiled Module, or one Module's EF Core model, to its violations. Read method bodies from their IL, which finds the calls and string literals of every method, those the compiler generates for async methods and lambdas included, with nothing but the base class library; read a project's restored closure from its `obj/project.assets.json`; and read the SQL of a string literal only after removing its comments and the bodies of its strings. The rules name types of EF Core and of the Kit's Cqrs, Domain, Modularity, Persistence.EFCore and Messaging.Wolverine packages, which the test project gets by referencing your Host.
- Find the Modules from the Host: walk the Host assembly's references among the repository's own projects (`DependencyContext.Default.RuntimeLibraries` of type `project`) and keep each assembly that implements `IModuleDefinition`. A Module's Contracts are the project named after it with `.Contracts`.
- Read each Module's EF Core model from the Host built but never started, with a placeholder connection string, so the tables carry the schemas and snake_case names the database has: resolve each DbContext from a scope and read `GetService<IDesignTimeModel>().Model`. From the same Host, `HostComposition.IsTenantScoped` names the tenant-scoped Modules. Nothing connects.
- Cross-check the tenant rule's tables against the same models: the tables EF Core filters by tenant (each entity type whose hierarchy root carries the `PouryaKitDbContext.TenantFilter` query filter, by its `GetTableMappings()`) are exactly the tables your tenant rule finds, and every table of a tenant-scoped aggregate, walked down from each `ITenantScoped` type through its navigations and derived types, is in both.
- Judge every Module with each function and assert no violations. Then judge an offending fixture Module with the same function and assert its exact violations, so a rule that reads nothing fails: a project of its own that breaks every rule, inside async methods and lambdas, referenced by the test project and never by the Host.
