# RabbitMQ between Hosts

`PouryaKit.Messaging.Wolverine.RabbitMq`, optional: Integration Events and Snapshots delivered to and from other Hosts through a RabbitMQ broker, with every guarantee of [messaging.md](messaging.md) kept across the broker. Namespace `PouryaKit.Messaging.Wolverine.RabbitMq`. It references `PouryaKit.Messaging.Wolverine` and `WolverineFx.RabbitMQ`. A Host that does not register it delivers inside itself only, as before; Modules, Contracts and Feature code never reference it, and nothing in them changes when a Host adds it.

- [The topology](#the-topology)
- [A Host that publishes](#a-host-that-publishes)
- [A Host that receives](#a-host-that-receives)
- [The service name](#the-service-name)
- [What the Kit guarantees across the broker](#what-the-kit-guarantees-across-the-broker)
- [A body the contract cannot read](#a-body-the-contract-cannot-read)
- [Dead letters at the broker queue](#dead-letters-at-the-broker-queue)
- [Deploying a Host that receives](#deploying-a-host-that-receives)
- [The Host's checks](#the-hosts-checks)
- [Testing across Hosts](#testing-across-hosts)

The topology is topology B of ADR 0006's RabbitMQ topology amendment. There is no service discovery, no Saga, no template for a Host of one Module, and no exactly-once delivery: each effect in a receiving Module's database is applied once by its Receipt, and an effect outside the database is at least once, as inside one Host.

## The topology

- **Publishing.** Each Integration Event and Snapshot a Module of the Host publishes is sent to a fanout exchange named by its wire name, such as `catalog.product-created.v1`. The route comes from the publication, never from a subscription, so the Unit of Work's Outbox writes it in the same commit as the change whether or not a subscription of this Host receives it. Each version is an exchange of its own.
- **Receiving.** The Host has one durable queue, named by its service name, with a durable Inbox. It is bound to the exchange of each wire name the Host declares as received from another Host and a subscription or a Saga of it takes. Each envelope received there is stored, then sent again, with every `pouryakit-` header it carries and its `parent-id` as `pouryakit-traceparent`, to the local queue of each subscription to it, and as its input to the queue of each Saga that takes it ([sagas.md](sagas.md)), and nowhere else. From there each subscription's copy is delivered exactly as a delivery from a Module of the Host is: its own retries, Receipt and dead letter.
- **Never twice.** A wire name the Host publishes itself is never bound to its queue: its subscriptions in this Host receive it from the publication, and bound, it would also come back from the broker.
- **Never round again.** The fan-out sends each copy to the local queue of each subscription to its type, and of each Saga that takes it, and nowhere else, never through the type's routes, which include its exchange when the Host publishes it. An envelope that came through a binding the Kit never makes, such as one an earlier deployment left for a wire name the Host now publishes, reaches the Host's subscriptions once more, where their Receipts stop it; sent back to its exchange it would go round the broker for ever, each time as a new envelope no Receipt recognises.

```text
Catalog Host (ModularMonolith.Host)          broker                       Inventory.Host
Outbox ── catalog.product-created.v1 ──▶ exchange ──▶ queue "inventory" ──▶ local://inventory.product-created/
                                                                         └─▶ local://inventory.audit-product/
```

## A Host that publishes

The Proving Sample's Host joins the broker when its configuration names one, and leaves Inventory to its own Host then:

```csharp
using PouryaKit.Messaging.Wolverine.EFCore;
using PouryaKit.Messaging.Wolverine.RabbitMq;

// ModularMonolith.Host: the connection string "broker" is an AMQP URI, such as amqp://user:password@rabbitmq:5672/
var broker = builder.Configuration.GetConnectionString("broker") is not null;
builder.UsePouryaKitWolverine(connectionName: "app");
if (broker)
{
    builder.UsePouryaKitRabbitMq(connectionName: "broker", serviceName: "modular-monolith");
}
```

- `UsePouryaKitRabbitMq(connectionName, serviceName)` comes after `UsePouryaKitWolverine(connectionName)` of `PouryaKit.Messaging.Wolverine.EFCore`, whose store the broker's envelopes go through. It throws when the configuration has no such connection string or leaves it blank, and when the service name is missing or not a valid name, such as one beginning with `amq.`, in any case, which RabbitMQ reserves for its own queues and exchanges.
- Nothing in a publishing Module changes: `Publish` and `PublishSnapshot` in its Composition Root ([messaging.md](messaging.md#declaring-what-a-module-publishes-and-subscribes)) already declare every wire name the Host sends.

## A Host that receives

A Host that subscribes to an Integration Event or Snapshot no Module of it publishes declares, in its own composition, that another Host publishes it, under the wire name the publishing Module declares. The Proving Sample's `Inventory.Host` composes the Inventory Module, unchanged by where it runs, and `dotnet new pouryakit-service` generates the same Host ([templates.md](templates.md#the-service)); its receiving part, without the authenticated HTTP endpoints and the health endpoints of the service defaults recipe it also serves ([host-recipes.md](host-recipes.md#composing-a-host-with-them)):

```csharp
using Catalog.Contracts.Products.Events.V1;
using Catalog.Contracts.Products.Events.V2;
using Inventory;
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.Hosting;
using PouryaKit.Messaging.Wolverine;
using PouryaKit.Messaging.Wolverine.EFCore;
using PouryaKit.Messaging.Wolverine.RabbitMq;
using PouryaKit.Modularity;
using PouryaKit.Observability;

// Inventory.Host/InventoryHost.cs
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();
builder.AddPouryaKitModules(modules => modules.Add<InventoryModule>());
// Catalog runs in another Host: its wire names, as its Composition Root publishes them.
builder.Services.Receive<ProductCreatedV1>("catalog.product-created.v1");
builder.Services.Receive<ProductCreatedV2>("catalog.product-created.v2");
builder.Services.ReceiveSnapshot<ProductSnapshotV1>("catalog.product-snapshot.v1");
// Its deliveries continue the trace of the Operation that published them in the Catalog Host.
builder.Services.AddPouryaKitObservability();
builder.UsePouryaKitWolverine(connectionName: "app");
builder.UsePouryaKitRabbitMq(connectionName: "broker", serviceName: "inventory");
```

- `services.Receive<TIntegrationEvent>(wireName)` and `services.ReceiveSnapshot<TSnapshot>(wireName)` (`PouryaKit.Messaging.Wolverine`) belong to the Host's composition, not to a Module's Composition Root: whether the publishing Module runs in this Host is the Host's choice, and the same Module runs unchanged in either. `ReceiveSnapshot` makes each subscription apply a Snapshot only when it is newer than every version of its aggregate it applied, as for a Snapshot a Module of the Host publishes.
- Without the declaration the Host fails when it is built: a subscription whose type no Module of the Host publishes has no wire name. The refusal names `Receive` for that case. A declaration that no subscription or Saga of the Host handles fails the build too, naming `Subscribe<T, THandler>`: it would bind nothing to the Host's queue and receive nothing.
- Each declaration is refused when the composition is built for a type that is not a concrete class or is a Domain Event, a wire name that is missing, invalid or another type's, a type received under two wire names or as both kinds, and a type a Module of the Host publishes: a Host publishes a contract or receives it. Repeating it adds nothing.
- The wire name is the publishing Module's, character for character. A misspelt one binds the queue to an exchange nobody publishes to, and nothing arrives; no check across Hosts can see it.

## The service name

The service name is the Host's queue on the broker and Wolverine's `ServiceName`. It is required, so it is always the Host's own choice and never a default derived from an assembly.

- **Unique per kind of Host on the broker.** Instances of one Host, such as two replicas of `Inventory.Host`, share its queue as competing receivers, which is intended: each message is received once by the Host and delivered to each of its subscriptions.
- **Two different Hosts must not share one.** They would share one queue and split its messages between them, each receiving only some of them, and each subscription of either would miss the rest. This is a deployment rule, and no check is possible: on the broker a replica and a foreign Host are indistinguishable. Every instance of a Host consumes under the same consumer tag, each Host has its own database, so Wolverine's node registry never sees another Host's nodes, and the broker compares only its own queue arguments when a queue is declared again. Name each kind of Host after what it is, such as `inventory`, and keep it when the Host is renamed or moved: its queue, and what waits in it, stay under the old name.

## What the Kit guarantees across the broker

Every guarantee of [messaging.md](messaging.md#what-the-kit-guarantees) for an Integration Event or Snapshot holds when the subscription is in another Host, and these hold for the broker itself; each is a test of `RabbitMqDeliveryTests` in the Proving Sample's tests:

- **Written in the commit, whoever subscribes.** A publication whose only subscriptions are in other Hosts is written to the Outbox in the commit of the Operation that published it, and each remote subscription applies it once, its effect and Receipt in one commit.
- **Each subscription on its own, in the receiving Host too.** Two subscriptions of one Host to one wire name are retried, dead-lettered at `local://{subscription}/` and replayed apart: one that keeps failing never delays, repeats or blocks the other.
- **A crash between the fan-out and the handled mark repeats nothing.** The fan-out stores every copy before it returns, so before the broker envelope is marked handled. A Host that dies in between receives the envelope again when it restarts and fans it out again; each copy finds its subscription's Receipt and applies nothing.
- **A copy the store cannot write is never lost.** The store error fails the broker envelope, which is not marked handled: it is retried, then dead-lettered at the Host's queue, and its replay fans it out again, where each subscription applies it once.
- **A restart loses nothing.** A delivery waiting for its retry survives a kill and restart of its Host, so does a copy that was being delivered when the Host died, and a message published while the Host is down waits in its queue and is delivered after it starts.
- **Every kind of Host receives every message.** Two Hosts with different service names that subscribe to one wire name each receive all of it.
- **The trace continues across the broker.** A receiving delivery's Operation is in the publishing Operation's trace with its tracestate, the child of that Operation, or of Wolverine's spans when the Host samples the `Wolverine` source ([observability.md](observability.md#the-trace-through-delivery)); each Host calls `AddPouryaKitObservability()` (`CrossProcessTracingTests`).
- **Identity and Operation Context cross intact.** The store keeps envelope ids alone (`MessageIdentity.IdOnly`): the broker envelope and each copy have an id of their own, and each Receipt is of the Domain Event's identity, or the Snapshot's, never of an envelope. The receiving delivery continues the publishing Operation, with its Actor, tenant and correlation and the publishing delivery as its causation, a trusted continuation.
- **Each version to its own subscriptions,** across the broker as inside one Host.
- **Snapshots stay ordered.** Redelivered across the broker in reverse order or concurrently, an older Snapshot never replaces a newer one, and a live Snapshot never brings back a copy its Tombstone removed.
- **The Kit's JSON.** Envelopes cross the broker in the Kit's JSON representation, the same as stored and HTTP bodies, never in Newtonsoft.Json's, which the transport package brings along, and are read strictly (`UsePouryaKitWireJson`, [A body the contract cannot read](#a-body-the-contract-cannot-read)).

Not proven here: several instances of one receiving Host sharing its queue under `DurabilityMode.Balanced`.

## A body the contract cannot read

Only the wire name crosses the broker, never a type, so two Hosts whose contract types under one wire name differ cannot be told apart when they start. The receiving Host reads each body strictly instead, with the Kit's one JSON representation, which refuses a member named twice, and .NET 10's `RespectRequiredConstructorParameters` and `RespectNullableAnnotations`; HTTP bodies keep the lenient reading, so an HTTP client may still leave a member out (ADR 0006, its amendment on strict wire reading).

- **The rule.** A body is read only when it carries every member the receiving contract requires, each constructor parameter without a default value and each `required` member, none of them is null where the contract's nullable annotations forbid it, and it names no member of the contract twice, in its own case or another. Annotations count on properties and constructor parameters compiled with nullable reference types enabled, not on collection elements or generic arguments, and a contract of settable or `init` properties without `required` requires nothing: write Contracts as positional records, as the Integration Events of [messaging.md](messaging.md#the-integration-event) are.
- **What happens to one that fails.** It is never handled. Wolverine sets it aside at once, without retries, as a dead letter received at the Host's queue, `rabbitmq://queue/{serviceName}`, before the fan-out, so no subscription receives a copy, and the dead letter keeps the `System.Text.Json.JsonException` and its message, which names the member, such as `JSON deserialization for type 'Catalog.Contracts.Products.Events.V1.ProductCreatedV1' was missing required properties including: 'name'.`
- **Unknown members are skipped.** A member the body carries and the receiving contract does not have is ignored, once or twice, so adding a member to a contract is a change receivers can read: a publishing Host may send it while receiving Hosts still run the build without it. A member a receiver needs from every message, such as a new constructor parameter without a default, is a new version under a new wire name ([messaging.md](messaging.md#a-new-version-of-an-integration-event)). The trade-off: a body of another shape that happens to carry every member the receiving contract requires is still read.
- **Writing follows the same rule.** A null in a non-nullable member of a published contract fails to serialize (`WireJsonTests` in `PouryaKit.Domain.Tests`). Wolverine serializes the message as the Unit of Work writes it to the Outbox, before the save, so the publishing delivery fails with a `Wolverine.WolverineSerializationException` around the `JsonException`, is retried and set aside as a dead letter, and nothing it did commits: its effect, its Receipt, its publication record and the envelope roll back, and no receiver gets it (`StrictWireWritingTests` in the Proving Sample's tests, for a subscription in the same Host). An envelope for another Host is written by the same serializer call into Wolverine's outgoing table, read from Wolverine's code.

`StrictWireReadingTests` in the Proving Sample's tests proves each case with two Hosts on RabbitMQ: the Catalog Host publishes contracts of other shapes under the wire name `Inventory.Host` receives.

## Dead letters at the broker queue

- A subscription's delivery is dead-lettered as in one Host, received at `local://{subscription}/` ([messaging.md](messaging.md#dead-letters)), never on the broker: no broker dead-letter queue is declared, so every dead letter is in the receiving Host's message store.
- A delivery that fails before it reaches any subscription is dead-lettered received at the Host's queue, `rabbitmq://queue/{serviceName}`, which names no subscription: a wire name the Host no longer receives, such as one whose last subscription a release removed while messages waited in the queue, a body its contract cannot read ([A body the contract cannot read](#a-body-the-contract-cannot-read)), or a store error while a copy is written (the fan-out waits for each copy's write, so the error fails the envelope instead of dropping the copy). Find them with `DeadLetterEnvelopeQuery { ReceivedAt = "rabbitmq://queue/inventory" }`, and replay one, once the cause is fixed, to hand it to the queue's fan-out again.
- `PouryaKit.Testing` names such a delivery or dead letter by that address, where it names others by subscription ([testing.md](testing.md#dead-letters)).

## Deploying a Host that receives

A message published before a receiving Host's queue exists, or before its queue is bound to the message's exchange, reaches no queue of that Host and is lost to it: the publishing Host knows only its exchange, and the broker drops a message no queue is bound to. The queue and its bindings are declared when the Host starts.

**The deployment rule:** start each Host that receives a wire name, once, before the first publication it must receive: on its first deployment, and on the deployment that adds its first subscription to a wire name. From then on its queue is durable and keeps what arrives while the Host is stopped. The Kit adds no alternate exchange, no queue declared by the publishing Host, and no provisioning outside the Hosts.

**Stopping receiving a wire name:** the Kit declares bindings and never removes one, so a binding outlives the subscription that made it. When a deployment stops receiving a wire name, remove its binding from the Host's queue, for example with `rabbitmqadmin` or the broker's management UI. Left in place, each message of it still arrives: one of a wire name the Host no longer knows is dead-lettered at the queue, and one the Host now publishes itself reaches its subscriptions a second time, where their Receipts stop it.

## The Host's checks

- **When the Host is built**, it fails for a subscription whose Integration Event or Snapshot no Module of the Host publishes and no `Receive` or `ReceiveSnapshot` declares, for a `Receive` or `ReceiveSnapshot` no subscription or Saga of the Host handles, and for a message type Wolverine would also route wherever a published type goes, because its wire name, name or full name equals that type's full name: with a broker, a v2 published under the full name of its v1 would be sent to v1's exchange. All as in [messaging.md](messaging.md#the-hosts-checks).
- **After every hosted service started**, it fails when Host configuration made a listener on a broker one that does not store what it receives in a durable Inbox, such as `ListenToRabbitQueue(name).BufferedInMemory()`, or one whose dead letters go to a broker dead-letter queue instead of the message store, such as `DeadLetterQueueing(...)`. The check reads Wolverine's own description of each broker listener (`IBrokerEndpoint`, `EndpointMode`, `DeadLetterStorage`), so it holds for any broker transport.

## Testing across Hosts

The Proving Sample's `RabbitMqDeliveryTests` (`tests/ModularMonolith.Tests`) are the pattern: a `BrokerFixture` with a RabbitMQ container (`Testcontainers.RabbitMq`, image `rabbitmq:4.3.6`) and a PostgreSQL container of its own, and for each test a vhost and a database per Host, each Host a real `IHost` from the sample's own composition. `BrokerFixture` serves `Inventory.Host` in memory with `UseTestServer()`, trusting the tests' bearer issuer, so a test reaches its health endpoints and, with a token, Inventory's endpoints with `GetTestClient()` (`GetStockItemTests`). One test runs `Inventory.Host` as an OS process with `dotnet run`, configured through its environment, in Production: a Release build of a Host has no runtime compiler, so it cannot run Development ([templates.md](templates.md#pre-generating-wolverines-code-for-production)). `CrossProcessTracingTests` runs both Hosts in the test process for the trace, since an `ActivityListener` sees its own process only. Read the receiving Host's store and database directly: `HostFixture` hosts one Host, and a publication whose only subscriptions are in another Host leaves no delivery in the publishing Host's store once it is sent, so `WaitForDeliveriesAsync` of the publishing Host's fixture has nothing to wait for. `AssertNothingSentAsync` sees such a publication, sent or waiting for the broker ([testing.md](testing.md)): `RabbitMqDeliveryTests` runs it on a `HostFixture` of the Catalog Host with the broker. The service template's smoke test is the smallest receiving fixture: its `App`, a `HostFixture`, starts a RabbitMQ container before the fixture's PostgreSQL, and reads the Host's queue and bindings with `rabbitmqctl` in that container ([templates.md](templates.md#the-smoke-test)).
