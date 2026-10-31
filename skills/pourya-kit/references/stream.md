# Streams

`PouryaKit.Persistence.Marten` and `PouryaKit.Messaging.Wolverine.Marten`: the Kit's second write model, a Stream, an aggregate whose stored state is the events it appended, on Marten 9.53.0 over PostgreSQL. The first package holds the Stream, `IStreams`, the Marten Unit of Work and the Host's Marten store; it references `PouryaKit.Cqrs` and `PouryaKit.Persistence`, and no Wolverine. The second integrates the store with Wolverine's Marten outbox on the Host's one message store, and keeps a Stream Module's Receipts; it references `PouryaKit.Messaging.Wolverine` and the first. Their namespaces are the package names. Each write model has one commit owner: a Stream Module's Operations commit through the Marten Unit of Work alone, an EF Module's through the EF one, and no Operation commits through both.

- [Registering](#registering)
- [Writing a Stream](#writing-a-stream)
- [Reading and changing Streams](#reading-and-changing-streams)
- [What the Kit guarantees](#what-the-kit-guarantees)
- [Limits](#limits)
- [Testing](#testing)

## Registering

The Proving Sample's Ledger is a Stream Module beside the EF Core Modules Catalog, Inventory and Billing: it keeps accounts, posts entries to them, reads an account's balance, and publishes `AccountOpenedV1`, which Billing receives. A Stream Module declares its write model in its Composition Root, with its own schema:

```csharp
public sealed class LedgerModule : IModuleDefinition
{
    public void Configure(ModuleBuilder module)
    {
        // First, so the tenant requirement runs before every other authorizer: the Host's Marten store keeps every
        // account under the tenant of the Operation that wrote it.
        module.AddPouryaKitTenancy();
        module.AddPouryaKitStreams(schema: "ledger");
        module.AddPouryaKitCqrs();
        module.AddPouryaKitValidation();
        module.AddPouryaKitNotificationHandlers();
        module.Publish<AccountOpenedV1>("ledger.account-opened.v1");
        module.AddPouryaKitEndpoints("/ledger");
    }
}
```

Its project references `PouryaKit.Persistence.Marten`, `PouryaKit.Messaging.Wolverine`, `PouryaKit.Modularity`, `PouryaKit.Validation`, `PouryaKit.Web` and its own Contracts, never Marten. Its commands are validated as Catalog's are ([commands-queries-validation.md](commands-queries-validation.md)): an empty account id, a blank holder or an entry of zero answers 400 with its validation Errors. The Host registers its one Marten store with `UsePouryaKitMarten` of `PouryaKit.Messaging.Wolverine.Marten`, over a connection string of its configuration, as `ModularMonolithHost` does:

```csharp
builder.AddPouryaKitModules(modules =>
{
    modules.Add<CatalogModule>();
    modules.Add<InventoryModule>();
    modules.Add<BillingModule>();
    modules.Add<LedgerModule>();
});
builder.UsePouryaKitWolverine(connectionName: "app"); // the message store, when the Host also has EF Core Modules
builder.UsePouryaKitMarten(connectionName: "app");
```

- In a Host that also has EF Core Modules, the message store is the one `UsePouryaKitWolverine(connectionName)` of the EF package declares, so call that first: the Marten call joins the store a store package already declared, and in the other order the EF call throws, naming the order, since Wolverine is registered once. In a Host whose Modules are all Stream Modules, `UsePouryaKitMarten` alone declares it: the Marten integration's own store, on the database Marten's store is built with.
- The Marten store keeps its events and streams in the schema `events` (`MartenHostApplicationBuilderExtensions.Schema`) and integrates with the message store's schema, `wolverine` unless the call names another; both must be the EF registration's. Marten creates each Stream Module's Receipt, Snapshot version and idempotency key tables in the Module's schema when the Host starts, and its own event tables when an event is first appended: a Stream Module needs no migration, and the Host's database user creates tables.
- The store is the Kit's JSON (`UsePouryaKitJson`), so a typed ID in an event is its value, as in an HTTP body.
- `AddPouryaKitMarten` of `PouryaKit.Persistence.Marten` registers the store without Wolverine, for a Host with no message store; `UsePouryaKitMarten` calls it, so a Host calls one of the two. A Host whose message store is another write model's, the EF package's, and whose Stream Modules' store is `AddPouryaKitMarten` fails when it is built ([Limits](#limits)).
- `PouryaKitIdempotencyOptions` set how long a Stream Module's idempotency keys are kept, as for an EF Module ([persistence.md](persistence.md#idempotency-key-retention)); the Marten store's own background service removes the expired ones from every Stream Module's table.

## Writing a Stream

Ledger's account, with its typed ID and its Domain Events:

```csharp
internal sealed record AccountId(Guid Value) : TypedId<Guid>(Value);

internal sealed record AccountOpenedDomainEvent(AccountId AccountId, string Holder) : DomainEvent;

internal sealed record EntryPostedDomainEvent(AccountId AccountId, decimal Amount) : DomainEvent;

internal sealed record AccountActivatedDomainEvent(AccountId AccountId) : DomainEvent;

internal sealed class Account : StreamAggregate
{
    public string Holder { get; private set; } = "";

    public decimal Balance { get; private set; }

    public bool IsActive { get; private set; }

    public static Account Open(AccountId id, string holder)
    {
        var account = new Account();
        var opened = new AccountOpenedDomainEvent(id, holder);
        account.Apply(opened);
        account.Raise(opened);
        return account;
    }

    public void Post(decimal amount)
    {
        var posted = new EntryPostedDomainEvent(new AccountId(Id), amount);
        Apply(posted);
        Raise(posted);
    }

    public void Activate()
    {
        var activated = new AccountActivatedDomainEvent(new AccountId(Id));
        Apply(activated);
        Raise(activated);
    }

    public void Apply(AccountOpenedDomainEvent opened)
    {
        Id = opened.AccountId.Value;
        Holder = opened.Holder;
    }

    public void Apply(EntryPostedDomainEvent posted) => Balance += posted.Amount;

    public void Apply(AccountActivatedDomainEvent activated) => IsActive = true;
}
```

- The Stream type is `internal`; its `Create` and `Apply` methods, and a should-delete method, are `public`. Marten folds a stream through a dispatcher its source generator writes at compile time, which takes public methods only: with an internal `Apply`, loading the Stream throws Marten's invalid-projection exception ("No source-generated dispatcher found") when the Operation runs, not at build.
- A behaviour applies its event to the Stream, then raises it; the Unit of Work appends what was raised, in order. Each event is a Domain Event, so a Notification Handler subscribes to it as to any other.
- A Stream's identity is a `Guid`, `StreamAggregate.Id`, which the first event's `Apply` sets; a typed ID wraps it in Feature code, in `IStreams` and in an event, where the Kit's JSON stores its value. Marten takes no class as a stream identity, so a Stream has no typed `Id` of its own.
- Marten stores an event under its type's name, snake-cased, without its namespace: `account_opened_domain_event` in `events.mt_events.type`. Give each event type of the Host's Stream Modules a name of its own.

## Reading and changing Streams

A command's handler, a Domain Event Handler or a Notification Handler loads and starts Streams through `IStreams`:

```csharp
internal sealed class OpenAccountCommandHandler(IStreams streams) : CommandHandler<OpenAccountCommand, Guid>
{
    public override Task<Result<Guid>> Handle(OpenAccountCommand command, CancellationToken cancellationToken)
    {
        streams.Start(Account.Open(new AccountId(command.AccountId), command.Holder));
        return Task.FromResult<Result<Guid>>(command.AccountId);
    }
}

internal sealed class PostEntryCommandHandler(IStreams streams) : CommandHandler<PostEntryCommand>
{
    public override async Task<Result> Handle(PostEntryCommand command, CancellationToken cancellationToken)
    {
        if (await streams.LoadAsync<Account>(new AccountId(command.AccountId), cancellationToken) is not { } account)
        {
            return AccountErrors.NotFound;
        }

        account.Post(command.Amount);
        return Result.Success();
    }
}
```

A query's projection is the Stream's fold, read through the same `LoadAsync`, Marten's live aggregation of the stream's events; the Kit keeps no projected document:

```csharp
internal sealed class GetAccountQueryHandler(IStreams streams) : QueryHandler<GetAccountQuery, AccountDto>
{
    public override async Task<Result<AccountDto>> Handle(GetAccountQuery query, CancellationToken cancellationToken) =>
        await streams.LoadAsync<Account>(new AccountId(query.AccountId), cancellationToken) is { } account
            ? new AccountDto(account.Id, account.Holder, account.Balance, account.IsActive, account.Version)
            : AccountErrors.NotFound;
}
```

Ledger's endpoints send these through its Gateway as any Module's do ([web.md](web.md#endpoints)): `POST /ledger/accounts` answers 201 with the account's location, `POST /ledger/accounts/{accountId}/entries` 204, and `GET /ledger/accounts/{accountId}` the folded account; an account opened twice answers 409 with `UnitOfWork.ConcurrencyConflict`.

- `LoadAsync` folds the stream of the Operation's tenant, null when it has none; a second load of one id in the Operation is the same instance. `Start` starts a new stream with what the Stream raised. Neither saves: the Unit of Work does, once.
- A query may load a Stream, but a change there is refused when the query ends, as a change to a tracked aggregate is: an event raised, a Stream started, or a loaded Stream's `Id` changed. The query fails, and the command that sent it fails too, even when it catches the refusal.
- Feature code never names Marten's `IDocumentSession`, its store or any Marten, Weasel, Wolverine or JasperFx type: the structure test fails the Module ([testing.md](testing.md)), which allows the types Marten's generator writes into the Module's assembly.
- A Notification Handler of a Stream Module changes Streams and publishes its Integration Events in one commit with its Receipt ([messaging.md](messaging.md#publishing-from-a-notification-handler)). Ledger's activates each new account and publishes `AccountOpenedV1`, which Billing's `OpenWelcomeQuoteIntegrationEventHandler` receives with a Receipt in Billing's schema:

```csharp
internal sealed class ActivateAccountNotificationHandler(IStreams streams, IIntegrationEventPublisher events)
    : INotificationHandler<AccountOpenedDomainEvent>
{
    public async Task<Result> Handle(AccountOpenedDomainEvent domainEvent, CancellationToken cancellationToken)
    {
        var account = (await streams.LoadAsync<Account>(domainEvent.AccountId, cancellationToken))!;
        account.Activate();
        events.Publish(
            domainEvent.Id,
            new AccountOpenedV1(domainEvent.Id, domainEvent.OccurredOn, domainEvent.AccountId.Value, domainEvent.Holder)
        );
        return Result.Success();
    }
}
```

- A Notification Handler that appends to the Stream whose event it handles races any command that appends to it meanwhile: one of the two is the Kit's conflict. A failed delivery is retried; a client that posts an entry right after opening an account may receive 409 and send it again.

## What the Kit guarantees

- **One commit.** The Marten Unit of Work begins its own transaction before the command, opens Marten's session over it, runs the command, drains the Domain Events, appends each Stream's events with `FetchForWriting` at the version it read (0 for a new stream), writes the handled Domain Events and the Integration Events its Notification Handlers published into the message store through Wolverine's Marten outbox, and saves. The events, the envelopes, the Module's Receipts and its idempotency keys commit in that one transaction or not at all, and the envelopes are sent after it. When the Operation wrote no event and no envelope, as a delivery whose handler only reads, or an idempotent command that changes no Stream, the Unit of Work commits its transaction itself, so its Receipt or key is kept.
- **Concurrency.** A stream another Operation appended to after this one read it, or one another Operation started too, fails the Operation with `UnitOfWorkErrors.ConcurrencyConflict` (`PouryaKit.Persistence`), whether it moved before the append or during the save; nothing is appended and nothing is sent. Nothing retries.
- **A send that fails after the commit**, or an application's Marten store listener (`StoreOptions.Listeners`) that throws there, is logged as a warning and is not the Operation's failure: the Unit of Work's own session listener, which Marten runs first after its commit, ahead of the store's listeners and Wolverine's send, has noted the commit. The change committed, and what was not sent stays in the message store, which delivers it after the inbox stale time or when a Host starts. A Host that dies between Marten's commit and the send leaves its envelopes with its node; a Host that starts on the store delivers them.
- **Receipts.** Each delivery into a Stream Module, Notification, Integration Event, Snapshot or delayed command, records its Receipt in the Module's `pouryakit_receipts` before its handler runs, in the Unit of Work's transaction, with the EF Module's statement: a concurrent duplicate waits for the first one's row, then records nothing and never runs its handler, and a failed handler leaves no Receipt, no Stream effect and no Integration Event. A Snapshot's version goes to `pouryakit_snapshot_versions` the same way; a publication record to `pouryakit_receipts`.
- **Tenancy.** The store is conjoined: every stream belongs to the tenant of the Operation that wrote it, and one id is a stream of its own under each tenant. An Operation reads and writes its own tenant's streams only; one with no tenant reads and writes Marten's tenantless streams, `*DEFAULT*`, so an Operation whose tenant is `*DEFAULT*` is refused, command or query, with an `InvalidOperationException`. A delivery appends under the tenant it was delivered with.
- **One message store.** The Marten store integrates with the Kit's message store, one store and one durability agent for both write models. A Marten store on another database than the message store, or integrated in another schema, fails the Host when it is built, naming both, whatever configured the Marten store, a later `ConfigureMarten` included.

## Limits

- A Stream publishes no Snapshot: what it publishes are the Notifications and the Integration Events of its events. A Snapshot declared for a Stream type, or by a Stream Module of any aggregate type, such as `IAggregateRoot`, would never be sent, so it fails the Host when it is built, naming the type and the Module.
- A Host with a message store gives each write model its Outbox: a Host with an Outbox in which one write model's Unit of Work has none fails when it is built, naming that write model's Modules and the registration to call, since their Operations would commit and never write what they publish. Stream Modules beside the EF message store (`UsePouryaKitWolverine`) and `AddPouryaKitMarten` call for `UsePouryaKitMarten` instead (`The Operations of LedgerModule would commit their Streams' events and never write what they publish: …`); EF Core Modules in a Host whose only message store is `UsePouryaKitMarten` call for `UsePouryaKitWolverine(connectionName)`, before it (`The Operations of BillingModule would commit their changes and never write what they publish or schedule: …`).
- Realtime pushes, `IUnitOfWorkConnection` and number sequences are the EF Unit of Work's; a Stream Module has none of them.
- A Stream's projections are folds of one stream; Marten's projection daemon and its documents are not part of the Kit.
- Wolverine's own aggregate-handler workflow is not used: the Kit's dispatcher runs every Operation.
- A Stream Module and a Saga cannot share one Host yet: Marten's integration puts its saga storage before Wolverine's lightweight one, so a Host with `UsePouryaKitMarten` and `AddPouryaKitSaga` is refused when it is built ([sagas.md](sagas.md#limits)).

## Testing

- A Stream Module's Operations go through its Gateway as any Module's do ([testing.md](testing.md)); a test reads the events from `events.mt_events` and the Module's tables from its schema.
- A test reads a commit through a Marten store listener installed on the Host, since an envelope's row changes once it is delivered.
