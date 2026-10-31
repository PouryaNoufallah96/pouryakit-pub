# Observability

`PouryaKit.Observability`: one activity and one duration measurement for every command and query the dispatcher runs, and a trace that continues from an Operation through the Outbox into the deliveries it causes. Namespace `PouryaKit.Observability`. It references `PouryaKit.Cqrs` only and emits through `System.Diagnostics`: the Kit has no OpenTelemetry package, no exporter and no Serilog.

- [Registering](#registering)
- [What is emitted](#what-is-emitted)
- [The trace through delivery](#the-trace-through-delivery)
- [Exporting](#exporting)
- [Testing telemetry](#testing-telemetry)

## Registering

On the Host, after `AddPouryaKitModules`, which registers the dispatcher this wraps:

```csharp
using PouryaKit.Modularity;
using PouryaKit.Observability;

builder.AddPouryaKitModules(modules =>
{
    modules.Add<CatalogModule>();
    modules.Add<InventoryModule>();
});
builder.Services.AddPouryaKitObservability();
```

- In an application without Modularity, call it after `AddPouryaKitCqrs`. With no dispatcher registered yet it throws `InvalidOperationException`: "No dispatcher of the Kit is registered to observe. Call AddPouryaKitObservability after AddPouryaKitModules or AddPouryaKitCqrs." A second call does nothing.
- A Host that never calls it has no activity and no measurement from the Kit. With it and nothing listening, each dispatch costs one check and returns the dispatcher's own task, allocating nothing. The check asks for either listener: with only an activity listener, or only a meter listener, the Operation is still observed.
- The duration's `Meter` comes from the Host container's `IMeterFactory`, which every Host built with `Host` or `WebApplication` registers, since the generic host's builder calls `AddMetrics()` while it populates the Host's services, so each Host owns its instruments and disposes them with itself. A listener or exporter that selects the meter by its name sees every Host in the process.
- Feature code changes nothing. A handler starts no activity of its own for its command, and the Kit adds no pipeline step an application could register.
- The Kit's dispatcher decorator wraps the whole dispatch, so any optional step the Kit adds inside its fixed pipeline, such as idempotency or caching, runs inside the Operation's activity and its duration.

## What is emitted

The names are constants on `OperationTelemetry`:

| What | Name |
| - | - |
| `ActivitySource` and `Meter` | `PouryaKit` (`OperationTelemetry.SourceName`) |
| Activity, one per command or query, kind internal | `pouryakit.operation` (`ActivityName`) |
| Histogram of each one's duration, unit `s`, a `double` that keeps fractions, with ASP.NET Core's second-scale bucket boundaries for `http.server.request.duration` (`0.005` to `10`) as its advice | `pouryakit.operation.duration` (`DurationName`) |
| Tag: the Module the Operation runs in, by type name; absent without Modules | `pouryakit.module` |
| Tag: the command's or query's type name | `pouryakit.operation` |
| Tag: `Person` or `System` | `pouryakit.actor.kind` |
| Tag: `success`, `failure` (a failed Result) or `exception` | `pouryakit.outcome` |

- The activity is current for the whole Operation: its authorization and validation, its handler, the Domain Event drain, the Unit of Work's Outbox write and its commit. A nested command or query is a child activity of the one that sent it.
- The status is `Ok` for a success, `Error` with the Error codes (`Note.Rejected`) for a failed Result, and `Error` with the exception's type for an exception, which is also recorded as an `exception` event. The duration is measured in every case, with the same tags.
- A delivery's outer Operation is tagged `Deliver`, or `RunDelayedCommand` for a delayed or recurring command. Its activity also carries `pouryakit.subscription`, the declared name of the subscription or of the delayed or recurring command, so each subscription's deliveries can be told apart in a trace. The duration histogram does not carry it: a Module's deliveries share its `Deliver` and `RunDelayedCommand` series. The delivered command and the commands its handler sends are its children.

## The trace through delivery

The work an Operation publishes stays in its trace, as a child, never as a link:

```text
request (ASP.NET Core)                  tracestate=vendor=1
└ pouryakit.operation CreateProductCommand     Catalog
  └ [Wolverine span, only when the Host samples the "Wolverine" source]
    └ pouryakit.operation Deliver               Catalog, publishes ProductCreatedV1
      └ [Wolverine span]
        └ pouryakit.operation Deliver           Inventory    tracestate=vendor=1
```

- Wolverine stores the id of the activity current when the Unit of Work writes the Outbox as the envelope's `parent-id`. The Outbox adds that activity's tracestate as the header `OperationContextHeaders.TraceState` (`pouryakit-tracestate`), which Wolverine does not carry.
- Each delivery restores both into `Operation.TraceParent` when it begins the Operation. The delivery's activity is a child of Wolverine's span of the envelope when the Host samples the `Wolverine` source, and of the publishing Operation when it does not. Either way it carries the publishing tracestate. Without this step every hop would start a new trace.
- An occurrence of a recurring command continues no Operation, so its delivery restores no parent: its Operation's activity is a child of Wolverine's span of the envelope when the Host samples the `Wolverine` source, and the root of a trace of its own when it does not ([messaging.md](messaging.md#recurring-commands)).
- A retry is read back from the message store and stays in the trace, as a sibling of the attempt that failed. A delivery recovered on another Host keeps the parent too.

Across RabbitMQ, into a subscription of another Host ([rabbitmq.md](rabbitmq.md)), the trace continues the same way:

```text
Catalog Host                                     Inventory.Host
pouryakit.operation Deliver  ── broker ──▶  [Wolverine span of the envelope the broker delivered]
  publishes ProductCreatedV1                  └ [Wolverine span of the subscription's copy]
                                                └ pouryakit.operation Deliver   Inventory    tracestate=vendor=1
```

- `parent-id` and `pouryakit-tracestate` cross the broker as they are. The receiving Host's fan-out sends each subscription its own copy, which Wolverine parents on the activity current there, none when nothing samples `Wolverine`; so the fan-out also writes the envelope's `parent-id` into each copy's header `OperationContextHeaders.TraceParent` (`pouryakit-traceparent`), and a delivery reads it when its copy names no parent of its own.
- Without the `Wolverine` source sampled, the delivery's activity is a child of the publishing Operation in the other Host. With it, there are two Wolverine spans between them: that of the envelope the broker delivered, then that of the copy.
- Every Host whose Operations belong in the trace calls `AddPouryaKitObservability()`, the receiving Host included: without it, its deliveries have no activity. Each Host exports its own spans; the backend joins them by trace id.

Through the Reverse proxy ([reverse-proxy.md](reverse-proxy.md)), an HTTP request keeps the client's trace id: the Host's request is a child of the proxy's call to it, which is a child of the proxy's request, and the Operation a child of the Host's request.

## Exporting

The Host adds an exporter for the source and the meter named `PouryaKit`, and for `Wolverine` if it wants Wolverine's spans as well, with the standard OpenTelemetry SDK. The service defaults recipe registers the application's own source, `OperationTelemetry.SourceName` and `Npgsql`, a span for each database command, not `Wolverine`, and exports over OTLP when `OTEL_EXPORTER_OTLP_ENDPOINT` is configured ([host-recipes.md](host-recipes.md#what-each-recipe-does)); Wolverine's spans are opt-in: add `.AddSource("Wolverine")` to its `WithTracing` registration. The OpenTelemetry SDK takes the histogram's bucket boundaries from its advice, so the exporter needs no view for it. Logs, Serilog included, are configured on the Host too; every entry logged inside an Operation carries the dispatcher's log scope, `Operation`, `OperationId`, `CorrelationId` and `Module` ([commands-queries-validation.md](commands-queries-validation.md#the-pipeline)), so a log backend joins it to its request without a trace. None of it belongs in a Module or in the Kit.

## Testing telemetry

You need no backend. Listen in the test, start an activity of your own for the request, and filter what you recorded by its trace id, because the Host is shared and earlier deliveries may still be in flight:

```csharp
using System.Diagnostics;
using PouryaKit.Observability;

var stopped = new System.Collections.Concurrent.ConcurrentQueue<Activity>();
using var listener = new ActivityListener
{
    // By name: a static source created while the listener is added would never be offered to it.
    ShouldListenTo = source => source.Name is OperationTelemetry.SourceName or "MyTests",
    Sample = static (ref _) => ActivitySamplingResult.AllDataAndRecorded,
    // Wolverine starts its spans with a string parent id; listen to "Wolverine" and sample here to see them.
    SampleUsingParentId = static (ref _) => ActivitySamplingResult.AllDataAndRecorded,
    ActivityStopped = stopped.Enqueue,
};
ActivitySource.AddActivityListener(listener);
```

- Assert on tags, `Status`, `ParentSpanId` and `TraceStateString`, not on timings. To assert a duration, use a `MeterListener` for the instrument `pouryakit.operation.duration` of the meter `PouryaKit`: a sub-second Operation has a value above zero with a fraction.
- A test's tracestate reaches a first delivery attempt from the envelope held in memory. Only a retry reads the header back from the store, so test a retry when the header itself is the claim.
- The Proving Sample's tests do all of this for the Catalog-to-Inventory chain: `RecordedTelemetry` holds the listeners, and `DeliveryProbes.ActivitiesOf(subscription)` gives the activity each delivery attempt ran in.
- To record both Hosts' activities with an `ActivityListener`, run both Hosts in the test process: a listener sees its own process only.
