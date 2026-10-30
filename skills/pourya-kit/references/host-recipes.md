# Host recipes

The service defaults of a Host built on PouryaKit: OpenTelemetry that exports the Kit's activities and metrics, liveness and readiness endpoints, service discovery, one standard resilience pipeline for every outgoing client, and Serilog when the Host wants it. They live in the Proving Sample's `ModularMonolith.ServiceDefaults` project, shaped like .NET Aspire's `ServiceDefaults` template. They are template content, not a Kit package and not an Aspire package: the project templates generate the project as `<Name>.ServiceDefaults` and compose all three recipes in their Hosts ([templates.md](templates.md)), and an application owns and changes its copy. Each recipe is optional, and none is needed for the Kit to work.

- [Composing a Host with them](#composing-a-host-with-them)
- [What each recipe does](#what-each-recipe-does)
- [One resilience pipeline](#one-resilience-pipeline)
- [Liveness and readiness](#liveness-and-readiness)
- [Serilog](#serilog)
- [Testing a Host with them](#testing-a-host-with-them)

## Composing a Host with them

The Proving Sample's `Inventory.Host` receives what Catalog publishes in another Host over RabbitMQ, and serves Inventory's endpoints to authenticated callers beside the health endpoints, which need no `PouryaKit.Web` registration of their own. Its `InventoryHost` and `Program`, written as one file:

```csharp
using Catalog.Contracts.Products.Events.V1;
using Catalog.Contracts.Products.Events.V2;
using Inventory;
using JasperFx;
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using PouryaKit.Messaging.Wolverine;
using PouryaKit.Messaging.Wolverine.EFCore;
using PouryaKit.Messaging.Wolverine.RabbitMq;
using PouryaKit.Modularity;
using PouryaKit.Observability;
using PouryaKit.Security.JwtBearer;
using PouryaKit.Web;
using PouryaKit.Web.OpenApi;

var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();
// Bearer tokens, authenticated by default, and the API versions Inventory declares (security.md, web.md, openapi.md).
builder.Services.AddPouryaKitJwtBearer(claims => claims.TenantClaim = "tenant");
builder.Services.AddAuthorization();
builder.Services.AddPouryaKitWeb();
builder.Services.AddPouryaKitOpenApi();
builder.AddPouryaKitModules(modules => modules.Add<InventoryModule>());
// Catalog runs in another Host: its wire names, as its Composition Root publishes them (rabbitmq.md).
builder.Services.Receive<ProductCreatedV1>("catalog.product-created.v1");
builder.Services.Receive<ProductCreatedV2>("catalog.product-created.v2");
builder.Services.ReceiveSnapshot<ProductSnapshotV1>("catalog.product-snapshot.v1");
// The recipe exports what the Kit emits; the Kit emits only with this, after the Modules.
builder.Services.AddPouryaKitObservability();
builder.UsePouryaKitWolverine(connectionName: "app");
builder.UsePouryaKitRabbitMq(connectionName: "broker", serviceName: "inventory");
// Optional: Serilog in place of the logging providers.
builder.AddSerilogDefaults();

var app = builder.Build();
app.UseExceptionHandler();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapPouryaKitModuleEndpoints();
// /alive anonymous in every environment, for a probe that carries no token; /health in Development only.
app.MapDefaultEndpoints();
if (app.Environment.IsDevelopment())
{
    app.MapPouryaKitOpenApi().AllowAnonymous();
}

// The JasperFx command line runs the Host, or a command such as codegen write (templates.md).
return await app.RunJasperFxCommands(args);
```

- `AddServiceDefaults()` and `AddSerilogDefaults()` extend any `IHostApplicationBuilder`, in the namespace `Microsoft.Extensions.Hosting` as Aspire's do; `MapDefaultEndpoints()` extends `WebApplication`.
- The recipe exports; the Kit emits. A Host without `AddPouryaKitObservability()` exports no Operation of the Kit, whatever its exporters listen to ([observability.md](observability.md#registering)).
- `ModularMonolith.Host` does not use the recipes, so the Kit's outgoing HTTP stays proven without them (`OutgoingHttpTests`). The Hosts the project templates generate do, with `AddServiceDefaults()`, `AddSerilogDefaults()` and `MapDefaultEndpoints()`; the monolith's and the modular monolith's `downstream` client adds only `AddPouryaKitOperationContext()`, and the service registers no client ([templates.md](templates.md#the-service)).

## What each recipe does

| Recipe | Method | What it adds |
| - | - | - |
| OpenTelemetry | `ConfigureOpenTelemetry()`, from `AddServiceDefaults()` | Tracing of the sources `PouryaKit` (`OperationTelemetry.SourceName`), `Npgsql`, Microsoft.Extensions.AI's and the application's name, ASP.NET Core requests except the health endpoints, and `HttpClient`; metrics of the meters `PouryaKit` and Microsoft.Extensions.AI's, ASP.NET Core, `HttpClient` and the runtime; the OpenTelemetry logging provider. OTLP export only when `OTEL_EXPORTER_OTLP_ENDPOINT` is configured, otherwise nothing leaves the process. |
| Health checks | `AddDefaultHealthChecks()`, from `AddServiceDefaults()` | `self`, tagged `live`, and `databases`, which asks each DbContext the Host registered whether its database answers ([below](#liveness-and-readiness)). |
| Service discovery | from `AddServiceDefaults()` | `AddServiceDiscovery()`, and its handler on every `HttpClient`. |
| HTTP resilience | from `AddServiceDefaults()` | `AddPouryaKitResilience()` on every `HttpClient` through `ConfigureHttpClientDefaults` ([below](#one-resilience-pipeline)). |
| Health endpoints | `MapDefaultEndpoints()` | `/alive`, the `live` checks only, in every environment: anonymous, exempt from rate limiting, answering the status word alone. `/health`, every check, anonymous, in Development only ([below](#liveness-and-readiness)). |
| Serilog | `AddSerilogDefaults()`, opted into by the Host | A console sink, an OTLP sink when `OTEL_EXPORTER_OTLP_ENDPOINT` is configured, the `Serilog` configuration section, and every `ILogEventSink` registered as a service ([below](#serilog)). |

- The tracing adds the sources `PouryaKit` and `Npgsql`, not `Wolverine`. `Npgsql` is Npgsql's own source, the one `AddNpgsql()` of `Npgsql.OpenTelemetry` adds, so no package is needed: each command an Operation sends to its database is a span under the Operation's, with its SQL in `db.query.text`, the Unit of Work's savepoints and writes and the Outbox's inserts into the message store among them, beside a `CONNECT` span for each physical connection opened; the commit itself is no command and has no span. The source and meter Experimental.Microsoft.Extensions.AI are those of the chat client `PouryaKit.AI` builds: a span for each call to the model, under the activity current when it is made, such as the Operation's, and duration and token metrics, with no message content ([ai.md](ai.md#registering)). Add `.AddSource("Wolverine")` to export Wolverine's spans as well; the Kit's trace holds either way, in one Host and across RabbitMQ ([observability.md](observability.md#the-trace-through-delivery)).
- The packages are pinned in `Directory.Packages.props`: `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Exporter.OpenTelemetryProtocol`, `OpenTelemetry.Instrumentation.AspNetCore`, `.Http` and `.Runtime` 1.20.1, `Microsoft.Extensions.ServiceDiscovery` 10.11.0, `Serilog.AspNetCore` 10.0.0, `Serilog.Sinks.Console` 6.1.1 and `Serilog.Sinks.OpenTelemetry` 4.2.0, beside `Microsoft.EntityFrameworkCore.Relational` for the readiness check. No Aspire package. Health checks are in the ASP.NET Core shared framework.

## One resilience pipeline

Aspire's template adds `AddStandardResilienceHandler()` to every client, which also retries POST. The recipe adds `AddPouryaKitResilience()` there instead: the same standard handler, retrying GET, HEAD, OPTIONS and TRACE only ([web.md](web.md#calls-between-kit-hosts)). The defaults are applied before any client's own configuration, whatever the registration order.

- A client that the defaults suit adds only `AddPouryaKitOperationContext()`: one pipeline, three retries of a safe method, a write sent once.
- A client that needs other limits, or `HttpReplay.AllRequests`, replaces the default pipeline:

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Http.Resilience;
using PouryaKit.Web;

var downstream = builder.Services.AddHttpClient("downstream");
#pragma warning disable EXTEXP0001 // RemoveAllResilienceHandlers is experimental in Microsoft.Extensions.Http.Resilience.
downstream.RemoveAllResilienceHandlers();
#pragma warning restore EXTEXP0001
downstream.AddPouryaKitResilience().Configure(options => options.Retry.MaxRetryAttempts = 2);
downstream.AddPouryaKitOperationContext();
```

- `AddPouryaKitResilience()` on a client of a Host with these defaults, without removing theirs first, would stack a second pipeline on the first, each outer attempt making every inner one. The Kit refuses it: once the client's handler chain is built, after every configuration callback and `RemoveAllResilienceHandlers()` ran, a chain with two resilience handlers throws an `InvalidOperationException` naming `RemoveAllResilienceHandlers()`, each time the client is created (`MisconfiguredHostTests`).
- A client that replaces the default pipeline has a name. The default pipeline is one for all clients that keep it: `ConfigureHttpClientDefaults` registers it under one name, `-standard`, since its builder has no client name, and the client registered with an empty name (`AddHttpClient(string.Empty)`) gets the same name. Its own pipeline would be the default one, options and replay policy included, so `HttpReplay.AllRequests` there would make every client that keeps the defaults replay writes. `AddPouryaKitResilience` refuses that client with an `ArgumentException` that says to name it (`HostRecipeTests`).
- One pipeline name also means one circuit breaker, which counts the failures of every client that keeps the defaults; this follows from the shared name and is not tested. A client whose downstream must not trip the others' breaker replaces the pipeline as above.

## Liveness and readiness

- `/alive` asks only the checks tagged `live`, so it answers `Healthy` while PostgreSQL or RabbitMQ is down. Liveness never depends on anything outside the process: an orchestrator that restarts a Host whose database is unreachable gains nothing.
- `/alive` is mapped in every environment: a Host deployed behind an orchestrator needs a liveness endpoint. An orchestrator's probe carries no token, so the endpoint allows anonymous callers in a Host that authenticates every endpoint by default, and it answers the status word, `Healthy`, with no detail of any check.
- `/alive` is exempt from the rate limit, with ASP.NET Core's `DisableRateLimiting()` on the endpoint, which its rate limiting middleware honours for the global limiter too. An anonymous call to an endpoint outside every Module resolves no Actor, so with `PouryaKit:Web:RateLimit` enabled it counts in the one partition of such requests ([web.md](web.md#cors-compression-and-rate-limits)): the probes themselves, or anyone calling `/alive`, would use up its permits, the next probe would answer 429, and the orchestrator would restart a healthy Host. The middleware reads the exemption from the matched endpoint, so `UseRouting()` comes before `UseRateLimiter()`, as the templates' `AppHost.cs` orders them and `TemplateTests` asserts for both. The Kit cannot detect a limiter placed before routing in a Host's own pipeline, since routing marks nothing on a request no endpoint matches, so a project that reorders its middleware keeps that order.
- `/health` asks every check, `databases` included: it answers 503 `Unhealthy` while a database the Host's DbContexts use does not answer, and `Healthy` again once it does. A Module's DbContext is internal to it, so the check finds each through its registered `DbContextOptions` instead of by type, and asks it what `AddDbContextCheck` asks, `Database.CanConnectAsync()`, which runs `SELECT 1`.
- `/health` is mapped in Development only, because it reports each database. Elsewhere no endpoint serves it: a Host with the Web defaults, such as `Inventory.Host`, answers an authenticated caller 404, as `HostRecipeTests` shows on `Inventory.Host` run as its own process in Production; an anonymous one gets the authenticated-by-default policy's 401, since the fallback policy covers a request routing matched to no endpoint. A Host without the Web defaults answers 404.
- **Readiness outside Development is the platform's, liveness the recipe's.** By decision, the Kit has no readiness option, and the recipe maps no readiness endpoint outside Development. The platform that runs the Host decides when an instance takes traffic, with its own probe over the Host's liveness path and the dependencies it runs for the Host, such as its database; a Kit readiness option would duplicate that probe. The recipe's part is `/alive`, which every Host that composes the service defaults maps in every environment.
- RabbitMQ has no check of its own: WolverineFx 6.52.0 registers none. A broker that is down shows in Wolverine's own reconnection, not in `/health`.

Aspire's guidance for health endpoints outside Development (<https://aka.ms/aspire/healthchecks>, "Non-development environments") is that its template maps neither endpoint there, and that a project which enables them protects them with host filtering, authorization or both, and considers a request timeout and an output cache against abuse. The recipe takes this from it and leaves the rest to the Host:

- **Anonymous, by the owner's decision.** A probe carries no token, so authorization would refuse the one caller the endpoint is for.
- **No request timeout and no output cache on `/alive`.** Its one check, `self`, is in-process and constant: there is no dependency call to bound or to cache, and both would add a service registration and a middleware to every Host's `Program.cs`.
- **Host filtering is the deploying project's choice.** Restricting the endpoint to an internal host name or a management port depends on the deployment, so the recipe builds none; a project that wants it adds ASP.NET Core's host filter to the endpoint in its own copy.
- **A Host that goes further takes Aspire's protections with it.** One that tags a check that leaves the process `live`, or maps `/health` outside Development, adds authorization or host filtering, a request timeout and an output cache, as that page shows.

## Serilog

`AddSerilogDefaults()` replaces the logger factory with Serilog's. Every log entry, the Kit's included, then goes to Serilog's sinks only: a logging provider added to `builder.Logging`, the OpenTelemetry provider of `ConfigureOpenTelemetry()` and a test's provider alike, receives nothing. That is why the recipe has its own OTLP sink, and why it reads sinks from the services: register an `ILogEventSink` to receive the Host's logs.

```csharp
builder.AddSerilogDefaults();
builder.Services.AddSingleton<Serilog.Core.ILogEventSink>(sink);
```

## Testing a Host with them

The Proving Sample's `HostRecipeTests` (`tests/ModularMonolith.Tests`) are the pattern:

- **Export without a backend.** Add an exporter of your own to the recipe's providers, `services.ConfigureOpenTelemetryTracerProvider(t => t.AddProcessor(new SimpleActivityExportProcessor(exporter)))` and `services.ConfigureOpenTelemetryMeterProvider(m => m.AddReader(new BaseExportingMetricReader(exporter)))`, then `MeterProvider.ForceFlush()` before asserting the `pouryakit.operation.duration` points. The providers are process-wide listeners: filter by trace id or by the `pouryakit.module` tag.
- **A database that stops answering.** `alter database <name> with allow_connections false` and `pg_terminate_backend` of its sessions, from a connection to another database, then `allow_connections true` before the Host stops. It touches one test's database only, unlike stopping the shared container.
- **Liveness outside Development.** A Host in Development maps both endpoints, so only one started in Production shows that `/alive` is mapped there and `/health` is not. The rate-limit exemption shows on such a Host with the limit at one permit: spend the permit with an anonymous call to an endpoint outside every Module, then `/alive` still answers 200 and the next such call answers 429.
- **Logs under Serilog.** Register an `ILogEventSink` as above: the fixture's `RecordedLogs` provider receives nothing once Serilog is the factory.
- **Outgoing HTTP.** A Web Host of its own with `AddServiceDefaults()`, `UseTestServer()` and the client, against WireMock.Net. A 503 with `Retry-After: 0` makes the standard pipeline retry at once, so its default limit is counted in milliseconds; `Retry-After: 1` holds the first retry for a second, and a test that cancels once the first attempt reached WireMock, rather than on a timer a slow first request can outrun, sees that retry never sent.
- **Telemetry and logs after a delivery.** The delivery's rows commit before its Operation logs, records its duration and ends its activity. Wait for the message store to settle (`BrokerEnvironment.WaitUntilSettledAsync`: Wolverine marks the envelope handled once the handler returned) before flushing the meter or reading a sink.
