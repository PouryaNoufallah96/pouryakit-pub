# Sagas

`PouryaKit.Messaging.Wolverine.Sagas`: a Saga, a Module's long-running reaction across Integration Events that keeps its own state between steps, on Wolverine's lightweight saga storage in the Host's PostgreSQL message store. It references `PouryaKit.Messaging.Wolverine` and `WolverineFx.Postgresql`, and neither EF Core nor Marten; its namespace is the package name. A Saga is the one reaction that runs outside an Operation: Wolverine discovers the saga and generates its steps' chains, and the Kit routes each step, claims it in its own transaction, guards its tenant and writes what it emits.

- [Declaring a Saga](#declaring-a-saga)
- [Writing a Saga](#writing-a-saga)
- [What the Kit guarantees](#what-the-kit-guarantees)
- [What a step must not do](#what-a-step-must-not-do)
- [The Host](#the-host)
- [Limits](#limits)
- [Testing](#testing)

## Declaring a Saga

A Module declares its Saga in its Composition Root, with its name and the system Actor its emissions carry, the Integration Events that start or advance it with the business id each one names, and its timeouts:

```csharp
using PouryaKit.Messaging.Wolverine;
using PouryaKit.Messaging.Wolverine.Sagas;
using PouryaKit.Modularity;

public sealed class ShippingModule : IModuleDefinition
{
    public void Configure(ModuleBuilder module)
    {
        module.Publish<OrderFulfilledV1>("shipping.order-fulfilled.v1");
        module.Publish<OrderCancelledV1>("shipping.order-cancelled.v1");
        module.AddPouryaKitSaga<FulfilmentSaga>(
            "shipping.fulfilment",
            systemActor: "shipping.fulfilment",
            saga => saga
                .Input<OrderPlacedV1>(placed => placed.OrderId)
                .Input<PaymentReceivedV1>(paid => paid.OrderId)
                .Timeout<FulfilmentTimeout>()
        );
    }
}
```

- The name is the saga's durable queue, the first part of each step's claim, the name each input's stored envelopes carry, `{name}.{event wire name}`, and its state table's, `{name}_saga` with dots and hyphens as underscores, so it is stable: keep it when the type is renamed. It follows the rule of a subscription's name and starts with a letter, since Wolverine writes the state table's name into its statements unquoted, and no subscription or delayed command may have it. Two saga types of one simple name are two sagas, since nothing of Wolverine's names them by their types.
- A saga belongs to the Module whose Composition Root declares it; in a Host of Modules, one declared on the Host's own services, or for a Module the Host does not compose, fails the build.
- Each input is an Integration Event the Host publishes, from any Module, or receives from another Host (`Receive<T>`); its wire name is the second part of each claim. A Snapshot is no input: a saga that takes one fails the build. The business id is written as invariant text; the saga it names is the one of the event's tenant.
- The saga emits only Integration Events its own Module declares with `Publish<T>`, never a Snapshot.

## Writing a Saga

```csharp
using PouryaKit.Messaging.Wolverine.Sagas;

public sealed class FulfilmentSaga : PouryaKitSaga
{
    public Guid Order { get; set; }

    public int Installments { get; set; }

    public int Paid { get; set; }

    public static (FulfilmentSaga, FulfilmentTimeout) Start(SagaInput<FulfilmentSaga, OrderPlacedV1> placed) =>
        (new FulfilmentSaga { Order = placed.Event.OrderId, Installments = placed.Event.Installments }, new FulfilmentTimeout(TimeSpan.FromDays(3)));

    public async Task Handle(SagaInput<FulfilmentSaga, PaymentReceivedV1> paid)
    {
        if (++Paid < Installments)
        {
            return;
        }

        await PublishAsync(new OrderFulfilledV1(Order, Paid));
        MarkCompleted();
    }

    public async Task Handle(FulfilmentTimeout timeout)
    {
        await PublishAsync(new OrderCancelledV1(Order, Paid));
        MarkCompleted();
    }
}

public sealed record FulfilmentTimeout(TimeSpan Within) : SagaTimeout(Within);
```

- A saga derives from `PouryaKitSaga` and is public, with a public parameterless constructor, since the code Wolverine generates for its steps names and creates it; so are its timeouts, each a public record derived from `SagaTimeout`, and the Integration Events it takes, as a Module's Contracts are. These are the Module's only public types beside its Composition Root and its Host's ports, and through the Kit's base types its code names no Wolverine type: the structure tests reject a saga of Wolverine's own `Saga`, or a timeout of its `TimeoutMessage` ([testing.md](testing.md#structure-tests)).
- A step takes a `SagaInput<TSaga, TEvent>`, the event as the Kit delivers it to this saga (`Event`, with the saga's identity, the event's tenant, correlation, wire name and identity), or one of its timeouts. A starting step is named `Start` and may be static, returning the new saga and the timeouts it schedules; an advancing step is named `Handle`; Wolverine's other names (`Starts`, `StartOrHandle`, `NotFound`) work too, each with or without an `Async` suffix. A step schedules a timeout by returning it, and returns nothing else: what else it returns Wolverine would send without the Kit's headers, so the step fails before its commit, naming `PublishAsync`.
- `PublishAsync` emits an Integration Event from the running step; `MarkCompleted` completes the saga, whose state is deleted with the step's commit, at the version the step loaded. The Kit sets `Id`, `Tenant` and `Correlation`; a step never does.
- A timeout names no identity member of its own (`Id`, `SagaId`, `{Saga}Id`, or one marked `[SagaIdentity]`): Wolverine would read it in place of the identity the timeout was scheduled under.

## What the Kit guarantees

- **A route of its own.** An Integration Event a saga takes is delivered to the saga's own durable queue as the saga's input type, beside every subscription of the event, each receiving it once: the Kit's delivery handler of the event type would otherwise take the saga's place on every queue. Two sagas of one event are two chains on two queues. An event another Host publishes reaches the saga too: the Host's broker queue fans it out to the saga's queue as its input, as it does to each subscription.
- **One claim per step.** Each step is claimed in its own transaction, before Wolverine loads the saga, keyed by the saga's name, the input's wire name and the event's identity, the identity a Receipt keys by. One event delivered under two envelope ids, by a republish or a broker's fan-out after a crash, applies once; two contracts published for one Domain Event share its identity and both apply. The step's claim, its state and every envelope it stores, emissions and timeouts, commit together, so a step whose process dies after its commit and before Wolverine marks its envelope applies nothing when it is delivered again, and a step that dies before its commit is applied once by the redelivery.
- **Claims outlive the saga.** A late copy of the starting event, after the saga completed, starts nothing.
- **Two steps of one saga at once.** A step updates the saga's state, or deletes it when the saga completes, only at the version it loaded, where Wolverine's own delete matches the identity alone. Otherwise Wolverine's `SagaConcurrencyException` rolls the step back, with its claim and its emissions, and the retry runs on the state the other step committed. A final payment and a timeout that load one state never both complete it: the retry of the step that lost finds no saga, so a payment is set aside as a dead letter and a timeout ends with no claim and no step. A completing step whose state another step changed completes on the new state.
- **The tenant in the identity.** A saga's identity is the event's tenant with its business id, each escaped as a URI's data (`SagaStorage.SagaId`): one business id in two tenants is two sagas, and no tenant can name another's. An event with no tenant, or one whose identity names a saga that does not exist and that it does not start, such as another tenant's, is refused before any step with `SagaInputRefusedException`, retried, then set aside as a dead letter received at `local://{saga name}/`; nothing is claimed, changed or emitted.
- **Emissions an ordinary subscription receives.** What a step emits is written into its Module's Outbox in the step's transaction, through the Messaging package's `OutboxWriter`, as an Integration Event like any other: under the saga's declared system Actor, the starting event's tenant and correlation, kept in the saga's state, the step's identity as its causation, and an identity every delivery of the step repeats (`SagaStorage.EmissionId`), which a receiving Receipt keys by. A tenant-scoped Module's subscription receives it as any. It continues the trace of the step's input. A step emits at most one Integration Event of each type; a second is refused, and the step commits nothing.
- **Timeouts keep their identity.** Each timeout a step schedules is kept in the saga's state under an identity of its own, derived from the scheduling step's identity, the timeout's wire name and its place among the step's timeouts of that type (`SagaStorage.TimeoutId`), so two of one type from one step are two. The timeout's step claims and emits under that identity. A timeout whose saga completed before it fell due ends with no claim and no step.
- **No Operation Context.** A step runs outside every Operation, with the Host's authority: no authorizer, validator, Unit of Work or Receipt runs for it, and the claim does the Receipt's work.

## What a step must not do

- **Reach outside its transaction.** A step that dies before its commit runs again, so an effect outside the transaction, a call to another service or a file written, would repeat. A step changes the saga's state, emits and schedules; anything else is the work of a subscription that receives what the saga emits.
- **Take a Kit scoped service.** A Host with a saga keeps Wolverine's `ServiceLocationPolicy.NotAllowed`, so a step that takes the Kit's `Operation`, its Integration Event publisher or a Module's internal service is refused when Wolverine generates its chain, at its first message, and its delivery is set aside with `InvalidServiceLocationException`.
- **Emit two events of one type**, as above, or a Snapshot.
- **Return an Integration Event** for Wolverine to send; it emits with `PublishAsync`.

## The Host

```csharp
builder.AddPouryaKitModules(modules => modules.Add<OrdersModule>().Add<ShippingModule>());
builder.UsePouryaKitWolverine(connectionName: "app"); // the PostgreSQL message store
```

- The saga's state is a row of Wolverine's saga table, named after the saga, `{name}_saga` with dots and hyphens as underscores, and its claims rows of `pouryakit_saga_steps`, both in the message store's schema; both tables are created with the store when the Host starts, the claims table under the advisory lock Wolverine migrates the store under. The claims are never deleted.
- The Host fails when it is built if a saga type is not a public concrete class, if the saga belongs to no Module in a Host of Modules or to one the Host does not compose, if an event it takes is a Snapshot, has no wire name in the Host or is not public, if a timeout is not public or names an identity member, if a subscription or delayed command has the saga's name, if a saga's name does not start with a letter, if two sagas' names give one state table, longer than 63 characters too, or one input name, if Wolverine's service location policy is not `NotAllowed`, or if Wolverine would keep a saga's state with another saga storage than its lightweight one in the message store, as with a Marten store (`UsePouryaKitMarten`), alone or beside the EF Core one; it fails to start if the saga's queue is not durable.
- Outside Development, Wolverine loads the saga's chains from the Host's assembly, where `dotnet run -- codegen write` generated them before the Release build ([templates.md](templates.md)); without them the Host does not start (`MissingPreBuiltTypesException`).

## Limits

- A saga runs on the PostgreSQL message store of `UsePouryaKitWolverine(connectionName)` of `PouryaKit.Messaging.Wolverine.EFCore`, with no Marten store: Wolverine gives a saga to the first saga storage that keeps every type, and Marten's integration puts its own before the lightweight one, where Marten's session would keep the state and the step's claim never commit, so a Host with a Saga and `UsePouryaKitMarten`, alone or beside the EF Core store, or with no store, is refused when it is built. So a saga and a Stream (Marten) cannot share one Host yet: Wolverine documents Marten's precedence for saga storage as intended and offers no supported way to keep a saga on its lightweight storage beside it. The refusal stays until Wolverine offers a supported option.
- An advancing event that arrives after its saga completed is refused and set aside as a dead letter, as one for another tenant is: complete a saga only once every event it waits for has arrived, or give it a `NotFound` step.
- A stored input's name derives from the saga's name and the event's wire name, so renaming the saga type or the event type changes nothing; a stored timeout's name is its type's full name, so renaming a timeout type strands the timeouts stored under the old name as dead letters.
- Wolverine's own logical deduplication is not used: it claims outside the step's transaction.

## Testing

- Publish the events a saga takes through the Modules that publish them, through their Gateways ([testing.md](testing.md)), and read the saga's state from its table, `wolverine.shipping_fulfilment_saga` for the saga `shipping.fulfilment`, and its claims from `wolverine.pouryakit_saga_steps`.
