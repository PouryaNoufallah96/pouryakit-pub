# Project templates

`PouryaKit.Templates`, a `dotnet new` template package: a new application starts from the Proving Sample's business code in one of three compositions. The monolith is the single-Module start, which grows into the modular monolith by adding `PouryaKit.Modularity` ([below](#growing-the-monolith-into-a-modular-monolith)); the modular monolith is several Modules in one Host, each reached through its Gateway; the service is one Module in a Host of its own, which receives another Host's Integration Events and Snapshots over RabbitMQ ([below](#the-service)).

- [Install and generate](#install-and-generate)
- [What each composition composes](#what-each-composition-composes)
- [Growing the monolith into a modular monolith](#growing-the-monolith-into-a-modular-monolith)
- [The service](#the-service)
- [Running the generated Host](#running-the-generated-host)
- [The smoke test](#the-smoke-test)
- [The optional recipes](#the-optional-recipes)
- [Pre-generating Wolverine's code for production](#pre-generating-wolverines-code-for-production)

## Install and generate

```bash
# From nuget.org, at the Kit's version, such as 0.1.0-preview.1. From another NuGet source, add --add-source <folder or feed>.
dotnet new install PouryaKit.Templates@<version>

dotnet new pouryakit-monolith -n Acme.Shop
dotnet new pouryakit-modular-monolith -n Acme.Shop
dotnet new pouryakit-service -n Acme.Stock
```

- Name the version: the template package's version becomes the Kit version every generated project pins (`PouryaKitVersion`, below), and without one the install takes whatever version its sources resolve that day. `@` separates it; the SDK's help still shows `::`, which the SDK reports as deprecated when it is used.
- `-n` is the project's name as you type it, such as `Acme.Shop`, `acme-shop`, `acme_shop` or `My Shop`. Folders and files carry it as typed, and so do the paths the solution and the project references name: `src/acme-shop.Host/acme-shop.Host.csproj`. Code carries the engine's namespace-safe form of it: `acme-shop` becomes `namespace acme_shop.Host` and `acme_shop.Tests`, `My Shop` becomes `My_Shop`, and `Acme.Shop` stays as it is. The broker queue takes the derived service name ([below](#the-optional-recipes)): `acme-shop` for the first three, `my-shop` for the last.
- Each template generates a solution in a folder named after `-n`: `Acme.Shop.slnx`, `src/Acme.Shop.Host`, `src/Acme.Shop.ServiceDefaults`, `src/Acme.Shop.ReverseProxy` but in the service, the Modules under `src/Modules/` with their sample names, and `tests/Acme.Shop.Tests`. Without `-n` it takes the output folder's name.
- `dotnet new` restores the solution once it is generated; `--no-restore` skips that. The restore needs a NuGet source with the Kit's packages.
- `Directory.Packages.props` pins every Kit package at `$(PouryaKitVersion)`, the version the template package was packed with, and every third-party package at the version the Kit builds and tests with. Upgrade the Kit by changing `PouryaKitVersion`.
- `global.json` pins the Kit's SDK, which a later patch of the same feature band also satisfies (`rollForward` is `latestPatch`), and Microsoft.Testing.Platform; `.config/dotnet-tools.json` holds `dotnet-ef` and `csharpier`.

## What each composition composes

| | `pouryakit-monolith` | `pouryakit-modular-monolith` | `pouryakit-service` |
| - | - | - | - |
| Role | The single-Module start: one Module's data and rules, which grows into the modular monolith when a second Module arrives | Several Modules in one Host, each reached through its Gateway | One Module in a Host of its own, which receives from another Host over RabbitMQ |
| Business code | Catalog and `Catalog.Contracts` | Catalog, `Catalog.Contracts`, Billing, Inventory, and Ledger with `Ledger.Contracts` | Inventory, and `Catalog.Contracts` for what it receives |
| Composition Root | `CatalogModule`, a static class whose `services.AddCatalog()` is a plain registration on the Host's services | Each Module's `IModuleDefinition`, added with `AddPouryaKitModules` | `InventoryModule`, added with `AddPouryaKitModules` |
| Endpoint adapters | Each Feature's endpoint sends through `IDispatcher` | Each Feature's endpoint sends through `IGatewayProcessor<CatalogModule>` | `GetStockItem`'s endpoint sends through `IGatewayProcessor<InventoryModule>` |
| Kit packages | Every Capability its projects reference: Wolverine over EF Core with [RabbitMQ](rabbitmq.md), Dapper, Validation, Security with JWT bearer, the Web defaults, OpenAPI, Observability, [Caching](caching.md), [gRPC](grpc.md), [Realtime](realtime.md) and the [Reverse proxy](reverse-proxy.md); no `PouryaKit.Modularity` | The monolith's, with [Streams](stream.md), [AI](ai.md) and `PouryaKit.Modularity` | Wolverine over EF Core with [RabbitMQ](rabbitmq.md), `PouryaKit.Modularity`, Dapper, the Web defaults, OpenAPI, JWT bearer and Observability; no Reverse proxy, gRPC, Realtime, Caching or AI |
| The Host's checks | Container validation, the composition checks and the startup checks from `AddPouryaKitCompositionChecks` | The same, from `AddPouryaKitModules` and `UsePouryaKitWolverine`, both of which call it | The same as the modular monolith's |

- **The same business code.** Aggregates, commands, queries, handlers, validators, the repository, Dapper reads and every event handler are the sample's own files in every template. Only the Composition Root and the endpoint adapters differ, and in the modular monolith and the service they are the sample's own too.
- **The monolith is one Module in all but name.** A persistence package adds its `IUnitOfWork` with `Add`, and only Modularity's `ModuleBuilder` keys each Module's own, so a Host without Modularity holds one Unit of Work: one Module's data. The monolith therefore takes Catalog alone; Billing, Inventory and the Stream Module Ledger, whose Marten Unit of Work is a second write model ([stream.md](stream.md)), stay in the modular monolith. Its Operation names no Module, its Unit of Work is the Host's, and Catalog publishes with no Module, so its Integration Events still reach another Host over RabbitMQ when a broker is configured. Because the request scope holds that Unit of Work, an endpoint or a service could take a repository there, which a Module of a Host of Modules cannot: keep the repository in handlers and send a command ([persistence.md](persistence.md#the-repository)).
- **The Host composition**, in `AppHost.cs`, is the same in the monolith and the modular monolith: `AppHost.CreateBuilder(args)` composes it and `AppHost.Build(builder)` builds it with its request pipeline, so `Program.cs` and the smoke test compose the one Host, as the Proving Sample's `ModularMonolithHost` does. It holds JWT bearer authentication from `Authentication:Schemes:Bearer`, authenticated-by-default HTTP, trusted peers, CORS, compression and the rate limit per Actor, the API versions and OpenAPI, idempotency key retention, the `downstream` HTTP client with the Operation Context, permissions from the caller's `permission` claims (`CallerPermissions`), Observability, Caching with Redis when `ConnectionStrings:cache` is set, Wolverine over EF Core on `ConnectionStrings:app`, in the modular monolith the Marten store of Ledger joining that message store (`UsePouryaKitMarten`) and the chat client of `PouryaKit:AI` with the application's guardrail, `CardNumberGuardrail`, for Catalog's agent ([ai.md](ai.md)), RabbitMQ when `ConnectionStrings:broker` is set, and the service defaults recipes ([host-recipes.md](host-recipes.md)): `AddServiceDefaults()`, `AddSerilogDefaults()` and `MapDefaultEndpoints()`. The modular monolith's Inventory reconciles stock at `Inventory:ReconcileStock:Every` and answers `GET /inventory/v2/stock-items/{productId}` to its tenant's callers.
- **Calls between Modules stay in-process**: Integration Events and Snapshots through the Host's Outbox, a question through the owning Module's Contracts. Never HTTP between Modules of one Host.
- **The same composition and startup checks in every template.** The monolith calls Cqrs's Host entry, the one `AddPouryaKitModules` calls, so a declared permission without its check, a cacheable query without caching or a command without a Unit of Work stops the Host when it is built, an endpoint never mapped stops it when it starts, and its container is validated in every environment ([modules.md](modules.md#the-hosts-checks)). `UsePouryaKitWolverine` calls the same entry; the monolith calls it last, after any `ConfigureContainer` of its own:

```csharp
builder.AddPouryaKitCompositionChecks();
```

- **The monolith names its assembly; the modular monolith infers it.** The monolith's `AddCatalog()` runs on the Host's own services, which are no Module's builder, so its registrations name `typeof(CatalogModule).Assembly` and its declarations belong to the application. Each modular Composition Root registers on its Module's builder with no assembly and no Module ([modules.md](modules.md#the-composition-root)).

## Growing the monolith into a modular monolith

The monolith is where a project of one Module starts, not a limit on it. When a second Module arrives, the project adds `PouryaKit.Modularity` and becomes the modular monolith; no aggregate, command, query, handler, validator, repository, read or event handler changes, since both templates generate Catalog's business code from the same sample files. A Module that later runs in its own process is the service composition, `pouryakit-service` ([below](#the-service)); the Proving Sample's `Inventory.Host` runs one over RabbitMQ ([rabbitmq.md](rabbitmq.md)), and serves Inventory's one read, `GetStockItem`, through the Module's Gateway, authenticated by default with the main Host's bearer settings and under API version 2.0. Inventory starts at 2.0, Catalog's current version, so it shares no deprecated version with Catalog ([openapi.md](openapi.md#declaring-a-modules-versions)).

Generated with one name from one template package, the two solutions differ in twelve files, and the modular monolith adds Billing, Inventory and Ledger with its Contracts, `CatalogProducts.cs` and its cross-Module test. Nine of the twelve change when Catalog becomes a Module:

1. `Directory.Packages.props`: `PouryaKit.Modularity` at `$(PouryaKitVersion)`.
2. `src/Modules/Catalog/Catalog/Catalog.csproj`: a `PackageReference` to `PouryaKit.Modularity`.
3. `CatalogModule.cs`: the static class whose `services.AddCatalog()` registers on the Host's services becomes `public sealed class CatalogModule : IModuleDefinition`, whose `Configure(ModuleBuilder module)` makes the same calls on `module`, without the `typeof(CatalogModule).Assembly` each registration names in the monolith. On a Module's builder the Kit scans the Module's own assembly, and every declaration, `Publish`, `PublishSnapshot` and `AddPouryaKitNotificationHandlers` included, belongs to the Module with nothing added ([modules.md](modules.md#the-composition-root)).
4. The five endpoints, `CreateCategoryEndpoint`, `CreateProductEndpoint`, `GetProductEndpoint`, `ListProductsEndpoint` and `RenameProductEndpoint`: each takes `OperationContext` and `IGatewayProcessor<CatalogModule>` in place of `IDispatcher`, and passes the context to `SendAsync` or `QueryAsync` ([web.md](web.md#endpoints)).
5. `src/Acme.Shop.Host/AppHost.cs`: `builder.Services.AddCatalog()` becomes `builder.AddPouryaKitModules(modules => modules.Add<CatalogModule>())`. `AddPouryaKitModules` calls `AddPouryaKitCompositionChecks()`, so the monolith's own call can go; calling it again changes nothing.

The other three, the `.slnx`, the Host's csproj and `appsettings.json`, differ by Billing, Inventory and Ledger: a second Module adds its project, its row in the solution, its `ProjectReference` in the Host, its `modules.Add<…>()` and its own configuration. Ledger, a Stream Module, also adds `PouryaKit.Messaging.Wolverine`, `PouryaKit.Persistence.Marten` and `PouryaKit.Messaging.Wolverine.Marten` to `Directory.Packages.props`, the last to the Host's csproj, and `builder.UsePouryaKitMarten(connectionName: "app")` after `UsePouryaKitWolverine` in `AppHost.cs`. `CatalogProducts.cs` and its registration, Catalog's answer to `ICatalogProducts` (in `Catalog.Contracts`, which both templates generate), come back when another Module asks Catalog a question ([modules.md](modules.md#asking-another-module-a-question)).

Two checks guard this path:

- **A declaration without its Module.** In the monolith a declaration belongs to the application. On a Module's builder it belongs to the Module with no argument, and one left on `builder.Services` in a Host of Modules fails the Host when it is built, naming its form on the Module's builder and the form that names the Module ([modules.md](modules.md#the-hosts-checks)).
- **A `ConfigureContainer` of the Host's own.** The monolith's `AppHost.cs` holds none, and calls `AddPouryaKitCompositionChecks()` last, so one the Host adds before it keeps the checks installed. In a Host of Modules, a `ConfigureContainer` after `AddPouryaKitModules` replaces the container the checks install, and the Host fails when it starts, naming the call.

## The service

`pouryakit-service` generates the third composition, the one the Proving Sample's `Inventory.Host` runs: one Module, Inventory, in a Host of its own process with its own database, receiving what Catalog publishes in another Host over RabbitMQ. A project generates it when it splits a Module out of its modular monolith; the modular monolith keeps Inventory in-process, unchanged, and no template holds an Aspire AppHost.

- **Only what the service needs**: the Host, its ServiceDefaults, Inventory, `Catalog.Contracts` and the test project. The generated solution holds no Catalog, monolith Host or Reverse proxy.
- **The Host composition**, in `AppHost.cs`, is `Inventory.Host`'s, and so is its pipeline: the service defaults and Serilog, JWT bearer authentication with the `tenant` claim, the authenticated default (`AddPouryaKitWeb`), the API versions and OpenAPI, `AddPouryaKitModules` with `InventoryModule`, `Receive<ProductCreatedV1>`, `Receive<ProductCreatedV2>` and `ReceiveSnapshot<ProductSnapshotV1>` under the wire names Catalog publishes them with, Observability, Wolverine over EF Core on `ConnectionStrings:app`, and RabbitMQ on `ConnectionStrings:broker` under the derived service name, always: a Host that receives needs its broker. Nothing else: it binds no CORS, compression, rate-limit, trusted-peer, sunset or idempotency settings, maps no rate limiter, and registers no permissions from the caller's claims and no `downstream` client, which the monolith and the modular monolith compose.
- **What it serves**: `GET /inventory/v2/stock-items/{productId}` to an authenticated caller of the tenant the stock item's delivery carried ([multi-tenancy.md](multi-tenancy.md)), `/alive` in every environment, and `/health` with the OpenAPI documents in Development.
- **What it receives**: a product Catalog creates in the publishing Host, which configures `ConnectionStrings:broker` and a service name of its own, opens its stock item here. Start the service first: a message published before its queue exists on the broker reaches no queue of it ([rabbitmq.md](rabbitmq.md#a-host-that-receives)).
- `appsettings.json` names the database `inventory`, the broker `amqp://guest:guest@localhost:5672/` and the cadence of the recurring reconciliation, `Inventory:ReconcileStock:Every`. Its `https` launch profile listens on `https://localhost:7081` and `http://localhost:5081`, so it runs beside a monolith or a modular monolith on their 7080 and 5080.

## Running the generated Host

```bash
cd Acme.Shop
dotnet tool restore
# ConnectionStrings:app in src/Acme.Shop.Host/appsettings.json names this PostgreSQL: localhost:5432, user and password postgres.
docker run -d --name acme-shop-postgres -p 5432:5432 -e POSTGRES_PASSWORD=postgres postgres:18.6
dotnet ef database update --project src/Modules/Catalog/Catalog --startup-project src/Acme.Shop.Host --context CatalogDbContext
# The modular monolith also migrates Billing (BillingDbContext) and Inventory (InventoryDbContext) the same way; Ledger needs no migration.
# The service migrates Inventory alone: --project src/Modules/Inventory/Inventory --context InventoryDbContext.
dotnet user-jwts create --project src/Acme.Shop.Host --claim tenant=acme
# Once per machine: trusts ASP.NET Core's development certificate, which the https URL uses.
dotnet dev-certs https --trust
dotnet run --project src/Acme.Shop.Host
# In a second terminal, with the token user-jwts printed: 200 and an empty Page of products.
curl -H "Authorization: Bearer <token>" http://localhost:5080/catalog/v2/products
```

- PostgreSQL runs in Docker from the image the Kit's tests use, `postgres:18.6`, on the port and with the user and password the generated `appsettings.json` names (`Host=localhost;Port=5432;...;Username=postgres;Password=postgres`). The database need not exist: `dotnet ef database update` creates a missing one, `app` in the monolith and the modular monolith and `inventory` in the service, before it applies the first migration. `docker rm -f -v acme-shop-postgres` removes the container and its data.
- The Host applies no migrations at startup; they are the Host's, as [persistence.md](persistence.md) says.
- Without `Authentication:Schemes:Bearer` the Host does not start. `dotnet user-jwts create --claim tenant=<name>` fills it for development and prints a token: it writes the issuer and the launch profile's URLs as audiences into the Host's `appsettings.Development.json`, which each template generates empty for it, since the tool writes nothing without one, and the signing key into the Host's user secrets, whose id it adds to the Host's project file; Catalog is tenant-scoped in the monolith and the modular monolith ([multi-tenancy.md](multi-tenancy.md)), so a token without the `tenant` claim gets 403 `Tenant.Required` from every Catalog command and query; in production, configure the issuer, audiences and signing keys of the real identity provider.
- The `https` launch profile runs in Development on `https://localhost:7080` and `http://localhost:5080` (the service's on 7081 and 5081), where `/health`, the OpenAPI documents and the Scalar reference are mapped, beside `/alive`, which every environment maps. In the monolith and the modular monolith the https URL serves REST and gRPC; on the http URL a gRPC call fails with `HTTP_1_1_REQUIRED`, since a listener without TLS negotiates no HTTP/2 ([grpc.md](grpc.md)). A token's `sub` is the Actor, its `tenant` the tenant, and its `permission` claims the Actor's permissions.
- A call names the API version in its path, `/catalog/v2/products`, or in an `api-version` header; without one Catalog answers 400, and without a token 401 ([openapi.md](openapi.md#what-a-caller-sees)).

## The smoke test

`tests/Acme.Shop.Tests` hosts the generated Host through `PouryaKit.Testing` ([testing.md](testing.md)): its `App` is a `HostFixture` whose `CreateHost` composes the Host with `AppHost.CreateBuilder` and `AppHost.Build`, served in memory with `UseTestServer()` (`Microsoft.AspNetCore.TestHost`), so the fixture runs PostgreSQL in Testcontainers (Docker must run), applies every DbContext's migrations, starts the Host and resets the data before each `HostTest<App>`. The tests call it with bearer tokens of a test issuer configured through `Authentication:Schemes:Bearer`. The service's `App` also starts RabbitMQ in a container (`Testcontainers.RabbitMq`) before the fixture starts PostgreSQL and the Host, which joins the broker as it is composed: it implements `IAsyncLifetime` again over `HostFixture`'s own. `PouryaKit.Testing` depends on `PouryaKit.Modularity` for its Gateway members, which a monolith's tests never call, so the monolith's test project restores Modularity while its Host does not.

In the monolith and the modular monolith:

- A category and a product created over HTTP, the product with an `Idempotency-Key`: 201 with its id and versioned location; the same key again answers the same status, body and location, and the filtered, sorted list still has one product.
- `/alive` answers 200 `Healthy` through the service defaults, to a caller without a token, from a second Host composed in Production, as it is deployed, and started with `HostFixture.StartHostAsync`; `/health` answers 401 there, since the service defaults map it in Development only. Liveness asks nothing outside the process, so it stays healthy while the database is down.
- Composed with a `ConnectionStrings:broker` value, the Host joins the broker under its derived service name (below). The test builds it and never starts it, so no broker is needed.
- With the permission checks removed from the registrations, the Host does not build: `RemoveProductsCommand declares RequiresPermissionAttribute, but its check is not registered`.
- The modular monolith adds one cross-Module effect: a product Catalog creates opens its stock item in Inventory's schema.

In the service:

- A stock item, written as Inventory's delivery of `ProductCreatedV1` writes it, is read at `/inventory/v2/stock-items/{productId}` by an Actor of its tenant, 200 with its JSON, and refused to a caller without a token, 401.
- `/alive` answers 200 `Healthy` from a second Host composed in Production on the same database and broker, and `/health` 401, as above.
- The started Host's Wolverine service name is the derived one, and the broker holds its queue with a consumer, bound to the exchanges of Catalog's three wire names, as `rabbitmqctl` in the container lists them.

```bash
dotnet test --project tests/Acme.Shop.Tests/Acme.Shop.Tests.csproj
```

## The optional recipes

- `src/Acme.Shop.ServiceDefaults` is the application's own copy of the service defaults recipes: OpenTelemetry with OTLP export when `OTEL_EXPORTER_OTLP_ENDPOINT` is set, liveness and readiness, service discovery, one resilience pipeline for every `HttpClient`, and Serilog. Change or drop any of them there ([host-recipes.md](host-recipes.md)).
- `src/Acme.Shop.ReverseProxy`, in the monolith and the modular monolith, is the Reverse proxy in front of the Host, a YARP Host of its own on `http://localhost:5090` with the same service defaults ([reverse-proxy.md](reverse-proxy.md)). Its `appsettings.json` routes every path to the Host by the derived service name (below), which the `Services` section resolves to `localhost:5080`, and probes the Host's `/alive`. It runs no Operation; the Host still authenticates every caller.
- The `downstream` client of the monolith and the modular monolith takes its resilience from the defaults and adds only `AddPouryaKitOperationContext()`; the service registers none. A client that needs other limits removes the default pipeline first, as [host-recipes.md](host-recipes.md#one-resilience-pipeline) shows; adding `AddPouryaKitResilience()` on top would stack a second pipeline, which the Kit refuses when the client is created.
- `/alive` asks nothing outside the process and is mapped in every environment, anonymous and, in the monolith and the modular monolith, exempt from the rate limit, for an orchestrator's liveness probe. `/health` also asks each database and is mapped in Development only. Why `/alive` carries no request timeout or output cache, and what a Host that maps more takes from Aspire's guidance, is in [host-recipes.md](host-recipes.md#liveness-and-readiness).
- Redis (`ConnectionStrings:cache`) and RabbitMQ (`ConnectionStrings:broker`) are off until configured, but in the service, which composes no Redis and always joins its broker. With a broker, the Host's queue, its service name on the broker, is the project's name lower-cased, each run of characters other than letters and digits a hyphen: `-n Acme.Shop` gives `acme-shop`. The template's `serviceName` symbol derives it, because a queue name takes only letters and digits in segments joined by single dots or hyphens, and the Host refuses any other name as it is composed (`UsePouryaKitRabbitMq`). A name with no letter or digit gives none, and another kind of Host on the same broker needs a name of its own; another Host that receives Catalog's events declares them as [rabbitmq.md](rabbitmq.md#a-host-that-receives) shows.
- In the monolith and the modular monolith, CORS policies (`PouryaKit:Web:Cors`), compression (`PouryaKit:Web:Compression`), the rate limit (`PouryaKit:Web:RateLimit`), trusted peers (`PouryaKit:TrustedPeers`) and idempotency key retention (`PouryaKit:Idempotency`) are read from configuration; none is set. The service reads none of these sections and maps no rate limiter, so setting one there changes nothing ([The service](#the-service)).

## Pre-generating Wolverine's code for production

Outside Development, Wolverine loads its generated handler code from the Host's assembly instead of compiling it when the Host starts: `UsePouryaKitWolverine` sets `TypeLoadMode.Static` there ([messaging.md](messaging.md#declaring-handlers-and-running-deliveries)). The generated Host follows Wolverine's code generation guide (`JasperFx/wolverine`, `docs/guide/codegen.md`):

- `Program.cs` is `return await AppHost.Build(AppHost.CreateBuilder(args)).RunJasperFxCommands(args);`, so the Host's command line is JasperFx's: `dotnet run` runs the Host, `dotnet run -- codegen write` writes Wolverine's generated code into `src/Acme.Shop.Host/Internal/Generated`, `codegen preview` prints it, `check-env` runs the environment checks and `help` lists the rest. Configure the Host through settings files, user secrets or the environment: an argument that is no JasperFx command, such as `--ConnectionStrings:app=...`, stops it with "Invalid usage".
- The Host references `WolverineFx.RuntimeCompilation` in Debug builds only (`<ItemGroup Condition="'$(Configuration)' == 'Debug'">`): Development compiles at run time, and a Release publish carries neither `Wolverine.RuntimeCompilation.dll` nor `JasperFx.RuntimeCompiler.dll`. Roslyn itself stays in the publish through `Microsoft.EntityFrameworkCore.Design`, which the EF Core outbox brings. The test project gets the package through `PouryaKit.Testing`, since it hosts the Host in Development from any build.
- The step before a release, `codegen write`, builds the Host and never starts it, so it needs no database and no runtime compiler; then publish:

```bash
dotnet run --project src/Acme.Shop.Host -- codegen write
dotnet publish src/Acme.Shop.Host -c Release -o publish
```

- Publish with its own restore, or after `dotnet restore -p:Configuration=Release`. A restore that evaluated Debug, such as a plain `dotnet restore`, lists the compiler, and a Release build with `--no-restore` after it carries it again.
- For a Kit Host today the step writes one file, `Internal/Generated/WolverineHandlers/GeneratedHandlerRegistry.cs`, an empty handler registry: the Kit's deliveries need no generated code. Without it the Host still starts in Production, and Wolverine logs that it falls back to a scan; `check-env` fails on it. The step matters once the application adds Wolverine handlers of its own through the `configure` hook of `UsePouryaKitWolverine`: each handler chain then has a generated type, and a Host published without it does not start (`MissingPreBuiltTypesException`).
- Commit `Internal/Generated`, as Wolverine's guide does, and write it again whenever a Wolverine handler changes; or run the step in the build, before the publish.
- Wolverine still creates its message storage when the Host starts, in every environment.
