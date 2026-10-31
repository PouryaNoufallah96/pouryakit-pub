# Realtime

`PouryaKit.Realtime`: server-sent events to browser clients. A Notification Handler pushes a realtime event on a Channel through `IRealtimePublisher`; the push is a PostgreSQL notification written on the delivery's own transaction, which PostgreSQL hands to the listener of every instance of the Host when that transaction commits; each instance passes it to its clients of the delivery's tenant that listen on the Channel, through an SSE endpoint a Module maps with `MapPouryaKitRealtime`. It references `PouryaKit.Web`, `PouryaKit.Persistence.EFCore` and Npgsql. Namespace `PouryaKit.Realtime`.

- [Registering](#registering)
- [Pushing from a Notification Handler](#pushing-from-a-notification-handler)
- [Mapping the SSE endpoint](#mapping-the-sse-endpoint)
- [What the Kit guarantees](#what-the-kit-guarantees)
- [Limits](#limits)
- [Testing](#testing)

## Registering

On the Host, which references `PouryaKit.Realtime`, with the connection string every Module stores over:

```csharp
using PouryaKit.Realtime;

builder.Services.AddPouryaKitRealtime(connectionName: "app");
```

- It registers the `IRealtimePublisher` and the instance's one listener, a hosted service. The listener opens a dedicated connection over `ConnectionStrings:{connectionName}`, outside every pool (`Pooling=false`), never multiplexed, with `KeepAlive` 30 seconds unless the string sets it and the application name `pouryakit-realtime` unless the string names one, and runs `LISTEN pouryakit_realtime` on it before the Host serves: a Host whose database cannot be reached does not start.
- The Host fails when its container is built, through the Kit's composition checks, when the configuration does not set the connection string or sets it blank, and when EF Core connects a DbContext that is a Unit of Work with other settings than that string, the password aside: a push is written on its Unit of Work's own connection, and PostgreSQL hands a notification only to the listeners of its own database. The check constructs each DbContext and reads the connection EF Core gives it, whatever configured it, a `ConfigureDbContext` after `UsePouryaKitPostgres` onto another string or a data source included, and a DbContext the application registers itself too; nothing connects. Two strings that set anything apart fail, though they may name one database: use one connection string for Realtime and every Module's DbContext.
- A Module references `PouryaKit.Realtime` for `IRealtimePublisher` and `MapPouryaKitRealtime`, and registers nothing.

## Pushing from a Notification Handler

Push from a Notification Handler, after the Operation that raised the Domain Event committed ([messaging.md](messaging.md)). The event is a plain record of the Module's Contracts, a small fact or a reference; the client reads the rest through a query:

```csharp
using Catalog.Contracts.Products.Realtime;
using PouryaKit.Domain;
using PouryaKit.Messaging;
using PouryaKit.Realtime;

internal sealed class PushNewProductNotificationHandler(IRealtimePublisher realtime)
    : INotificationHandler<ProductCreatedDomainEvent>
{
    public Task<Result> Handle(ProductCreatedDomainEvent domainEvent, CancellationToken cancellationToken) =>
        realtime.PublishAsync(
            NewProduct.Channel,
            new NewProduct(domainEvent.ProductId.Value, domainEvent.Name),
            cancellationToken
        );
}
```

- `PublishAsync` runs `select pg_notify('pouryakit_realtime', payload)` on the Unit of Work's connection and transaction (`IUnitOfWorkConnection`), with the payload the Channel, the delivery's tenant (`OperationContext.NoTenant` when it has none) and the event, written with the Host's HTTP JSON options (`JsonOptions`), which `AddPouryaKitWeb` sets to the Kit's JSON, so the event reaches the browser as a response body would. Nothing reaches a client before the delivery commits.
- It fails with `RealtimeErrors.PayloadTooLarge` (`Realtime.PayloadTooLarge`), pushing nothing, when that payload is 8000 bytes or more: PostgreSQL carries a notification payload "shorter than 8000 bytes" in its default configuration, counted in the database's encoding, so in UTF-8 here. Returned from the handler, the failure is retried and set aside as a dead letter like any other; a handler that pushes a reference never meets it.
- It throws, as the Integration Event publisher does, when called outside a Notification Handler's delivery, which the delivery step records in the delivery's own scope (`INotificationDelivery`), and both publishers read: outside that delivery, a command an HTTP request or a Gateway call sends, its Domain Event Handlers, an Integration Event Handler, a delayed command, a trusted peer's request and a command sent with `Continue()` all throw. So does a command the Notification Handler itself sends through a Gateway, even with its own `OperationContext`: that command runs in a scope of its own, on a transaction of its own. So does a Domain Event Handler of what the Notification Handler changed: the delivery step records the delivery only while the handler runs, and the Unit of Work runs those Domain Event Handlers after it returns. So does code that only reads (a query, an authorizer, a validator). It also throws on a blank Channel, one holding a line break, and on `connected` or `gap`, the Kit's own events.
- The tenant is the delivery's, from its Operation Context; a handler never names one.

## Mapping the SSE endpoint

A Module maps the endpoint from an `IEndpoint`, so it lies in the Module's route group, which requires an authenticated caller and begins the request's Operation:

```csharp
using Catalog.Contracts.Products.Realtime;
using Microsoft.AspNetCore.Routing;
using PouryaKit.Realtime;
using PouryaKit.Web;

internal sealed class NewProductsRealtime : IEndpoint
{
    public void Map(IEndpointRouteBuilder routes) =>
        routes.MapPouryaKitRealtime("/events", (actor, channel) => channel == NewProduct.Channel);
}
```

- The client names its Channels in the query string, `GET /catalog/events?channel=catalog.new-products`, once per Channel. The rule, a `Func<Actor, string, bool>`, admits each for the request's Actor, such as `channel == $"inbox.{actor.Id}"` for an Actor's own Channel. A request that names none, or a blank one (`?channel=`, whatever else it names), is answered 400 with `Realtime.ChannelRequired` before the rule runs, and one naming a Channel the rule refuses 403 with `Realtime.ChannelNotAdmitted`, as problem responses, before anything is streamed.
- The response is the framework's `TypedResults.ServerSentEvents`: first an event of type `connected`, from which every event pushed for the client's Channels reaches it; then each pushed event, of the type of its Channel, with the event's JSON as its data; and an event of type `gap` after the instance's listener reconnected. A browser's `EventSource` listens with `addEventListener(channel, …)`.
- The tenant is the request's Operation Context's: a client of tenant X never receives tenant Y's event, and a client without a tenant receives only pushes made without one.
- The connection is served by the request itself, never from inside a Gateway call: streaming opens no Gateway scope and no Unit of Work, while the request's scope and its Operation, which the route group began, last until the connection ends. Feature code never holds a connection. It ends when the client disconnects, and when the Host stops.
- How a browser presents its token is the application's: `EventSource` sends cookies but no `Authorization` header.

## What the Kit guarantees

- **A push reaches a client connected to any instance.** Every instance's listener receives every committed push from PostgreSQL; there is no connection list that lives in one process.
- **A delivery that rolls back pushes nothing**, a failed commit included: PostgreSQL delivers a notification only when its transaction commits. A delivery retried after it pushed and failed pushes again, and only the committed attempt's push is delivered.
- **A client receives only its tenant's events on the Channels its endpoint admits for its Actor.**
- **An oversized push fails in the handler**, never silently in transit.
- **After a lost listener connection, every client gets a `gap`.** The listener reconnects at once and, while that fails, after 1 second, then twice as long each time up to 30 seconds, listens again, and only then sends the gap, so a client that reads its state again on `gap` misses nothing committed after it.
- **The framework's features leave a connection whole**: the rate limit per Actor counts one permit for a connection, whatever it streams; response compression leaves `text/event-stream` alone, even when the Host adds it to the compressed types, since the response states `Content-Encoding: identity`; and status code pages leave a started connection as it was written.

## Limits

- **Delivery to clients is at most once and not durable.** A client receives a push only while it is connected; one that connects later, or loses its connection, receives nothing of what it missed: on `connected` after a reconnect and on `gap`, it reads its state through a query. The Outbox and the Receipts hold the durable state; there is no replayed or stored push.
- **A slow client is disconnected.** Each client has a buffer of 256 events, which the listener fills without waiting; a client a full buffer behind is disconnected: its stream drains the events its buffer already holds, drops the overflow event and every later one, and closes, and `EventSource` reconnects, so the client re-reads its state.
- **Identical pushes in one transaction arrive once**: PostgreSQL folds notifications of one channel with identical payloads in one transaction.
- One Channel namespace per database: the Hosts of one database share the PostgreSQL channel `pouryakit_realtime`, so two applications on one database name their Channels apart.
- A Marten Unit of Work does not push; no SignalR and no second push channel to the client.

## Testing

- The Proving Sample's tests read the endpoint over the test server with `SseParser`, reading from `HttpCompletionOption.ResponseHeadersRead`, and wait for the `connected` event before they push. Two instances are Hosts composed with `AppFixture.ComposeHost` on the fixture's database, each configured back to `DurabilityMode.Balanced` after `UsePouryaKitTestDelivery`'s Solo, and started with `HostFixture.StartHostAsync`. A test that pushes twice and depends on their order waits for the first push's deliveries (`WaitForDeliveriesAsync`): deliveries run in parallel.
- A lost listener connection is the server's `pg_terminate_backend` on the session whose `application_name` is `pouryakit-realtime`.
- A test Host that composes a Module with a Notification Handler that pushes registers `AddPouryaKitRealtime` too: the Kit's container is validated when it is built.
