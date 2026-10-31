# Aspire resources

`PouryaKit.Aspire`: the containers a Kit Host runs on locally, for an Aspire AppHost and nothing else. PostgreSQL always, Redis for a Host that caches through Redis, RabbitMQ for a Host that sends or receives over RabbitMQ, Ollama for a Host whose agent runs locally, each on a pinned image with its data in a named volume. Namespace `PouryaKit.Aspire`; it references no Kit package, and no Kit package brings an Aspire package into a Host; keep this one out of your Hosts' own projects.

- [The AppHost](#the-apphost)
- [A local agent](#a-local-agent)
- [What each registration adds](#what-each-registration-adds)
- [Running it](#running-it)
- [Testing](#testing)
- [Limits](#limits)

## The AppHost

An AppHost project uses the `Aspire.AppHost.Sdk` and references the Hosts and the Reverse proxy it runs, and the package. This is the Proving Sample's AppHost project:

```xml
<Project Sdk="Aspire.AppHost.Sdk/13.6.4">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <UserSecretsId>pouryakit-modular-monolith-apphost</UserSecretsId>
    <NoWarn>$(NoWarn);ASPIRE010</NoWarn>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\Inventory.Host\Inventory.Host.csproj" />
    <ProjectReference Include="..\ModularMonolith.Host\ModularMonolith.Host.csproj" />
    <ProjectReference Include="..\ModularMonolith.ReverseProxy\ModularMonolith.ReverseProxy.csproj" />
    <PackageReference Include="PouryaKit.Aspire" />
  </ItemGroup>
</Project>
```

Its `Properties/launchSettings.json` has one profile that sets `DOTNET_ENVIRONMENT` to `Development` and opens no browser. The environment matters: the AppHost reads its user secrets only in Development, and without them it generates new server passwords on every run, which the volumes the first run made then refuse.

The SDK adds `Aspire.Hosting.AppHost` itself: with central package management, give it no `PackageVersion`, or restore fails with NU1009. `ASPIRE010` warns that the AppHost runs without the Aspire CLI bundle; it takes its orchestrator from NuGet instead, so Docker is all it needs.

Its `Program.cs` adds a resource for each part of each Host's composition and gives it to the Host under the connection name the Host reads:

```csharp
using PouryaKit.Aspire;

var builder = DistributedApplication.CreateBuilder(AppHostOptions.Create(args));

var postgres = builder.AddPouryaKitPostgres("postgres");
var cache = builder.AddPouryaKitRedis("cache");
var broker = builder.AddPouryaKitRabbitMq("broker");
var shop = postgres.AddDatabase("modular-monolith-db", "modular_monolith");
var inventory = postgres.AddDatabase("inventory-db", "inventory");

var inventoryHost = builder
    .AddProject<Projects.Inventory_Host>("inventory")
    .WithHttpEndpoint()
    .WithHttpHealthCheck("/alive")
    .WithReference(inventory, "app")
    .WithReference(broker)
    // ...its bearer settings, below...
    .WaitFor(inventory)
    .WaitFor(broker);

var shopHost = builder
    .AddProject<Projects.ModularMonolith_Host>("modular-monolith")
    .WithHttpEndpoint()
    .WithHttpEndpoint(name: "grpc")
    .WithEndpointsInEnvironment(_ => false)
    .WithReference(shop, "app")
    .WithReference(cache)
    .WithReference(broker)
    // ...its bearer settings, below...
    .WaitFor(shop)
    .WaitFor(cache)
    .WaitFor(broker);
// ...its Kestrel endpoints, below...

builder
    .AddProject<Projects.ModularMonolith_ReverseProxy>("reverse-proxy")
    .WithHttpHealthCheck("/alive")
    .WithReference(inventoryHost)
    .WithReference(shopHost)
    // ...the service route, below...
    .WaitFor(inventoryHost)
    .WaitFor(shopHost);

builder.Build().Run();
```

- **Headless.** `AppHostOptions.Create(args)` (`AppHostOptions.cs` beside `Program.cs`) returns `DistributedApplicationOptions` with `DisableDashboard = true`: no dashboard, under `dotnet run` or in the tests. A developer who wants Aspire's dashboard locally sets it to `false` in their copy; the AppHost then logs the dashboard's login URL.
- **The composition decides, through its connection strings.** An AppHost cannot see a Host's registrations, so the AppHost's author adds Redis only for a Host whose composition registers a Redis cache, and RabbitMQ only for one that calls `UsePouryaKitRabbitMq`. `WithReference(resource, "app")` sets `ConnectionStrings:app`, the name `UsePouryaKitWolverine(connectionName: "app")` reads; `WithReference(cache)` sets `ConnectionStrings:cache` and `WithReference(broker)` `ConnectionStrings:broker`, the names the sample's composition reads. The sample's `ModularMonolith.Host` caches in Redis and sends over RabbitMQ when those are set, so with a broker Inventory runs in `Inventory.Host`, which receives over RabbitMQ and caches nothing.
- **Each Host its own database** on the one server: `AddDatabase(name, databaseName)` creates it when the server is ready.
- **Endpoints.** A Host project without `Properties/launchSettings.json` has none: `WithHttpEndpoint()` gives it one on a free port. `WithHttpHealthCheck("/alive")` waits on a Host that maps the service defaults' liveness path ([host-recipes.md](host-recipes.md#liveness-and-readiness)); the sample's `ModularMonolith.Host` maps none.
- **gRPC on an endpoint of its own.** An endpoint is, by Aspire's default, a proxy on the port a client calls, in front of the Host's listener on a port of its own. Without TLS a Kestrel listener of both HTTP versions negotiates no HTTP/2 and refuses every gRPC call with `HTTP_1_1_REQUIRED`, and one of HTTP/2 alone refuses REST's HTTP/1.1 callers ([grpc.md](grpc.md#declaring-and-mapping-a-service)), so `ModularMonolith.Host`, which serves Catalog's gRPC service beside its REST endpoints, has a second endpoint, `grpc`, whose listener speaks HTTP/2 alone. Kestrel's named endpoints replace the URLs a project otherwise gets in `ASPNETCORE_URLS`, so `WithEndpointsInEnvironment(_ => false)` passes none, and the AppHost names both listeners, each on the port its endpoint's proxy forwards to:

```csharp
shopHost.WithEnvironment(context =>
{
    foreach (var endpoint in (string[])["http", "grpc"])
    {
        context.EnvironmentVariables[$"Kestrel__Endpoints__{endpoint}__Url"] = ReferenceExpression.Create(
            $"http://localhost:{shopHost.GetEndpoint(endpoint).Property(EndpointProperty.TargetPort)}"
        );
    }

    context.EnvironmentVariables["Kestrel__Endpoints__grpc__Protocols"] = "Http2";
});
```

Aspire's endpoint proxy carries a gRPC call to the `grpc` endpoint, HTTP/2 with prior knowledge over cleartext, as the start test shows ([below](#testing)); REST callers and the Reverse proxy keep the `http` endpoint.

- **A Host that authenticates callers** needs its bearer settings, or it fails to start ([security.md](security.md)). The sample passes them as environment variables, the signing key from a secret parameter:

```csharp
var signingKey = builder.AddParameter("bearer-signing-key", secret: true);
// on each Host project, Inventory.Host and ModularMonolith.Host:
    .WithEnvironment("Authentication__Schemes__Bearer__ValidIssuer", "modular-monolith-apphost")
    .WithEnvironment("Authentication__Schemes__Bearer__ValidAudiences__0", "modular-monolith")
    .WithEnvironment("Authentication__Schemes__Bearer__SigningKeys__0__Issuer", "modular-monolith-apphost")
    .WithEnvironment("Authentication__Schemes__Bearer__SigningKeys__0__Value", signingKey)
```

Set the parameter to a base64 key of at least 32 bytes in the AppHost's user secrets (`Parameters:bearer-signing-key`), and sign a caller's token with it, that issuer and that audience.

The Reverse proxy ([reverse-proxy.md](reverse-proxy.md)) runs in front of both Hosts. Each `WithReference(host)` gives it each of the Host's endpoints as `services__<resource>__<endpoint>__0` (`ModularMonolith.Host`'s `grpc` one too), the `Services` entries service discovery reads, so a destination `http://<resource>` resolves to that Host's `http` endpoint in place of the `localhost` address of the proxy's own settings. Those settings route every path to `modular-monolith` with that cluster's probe off, since `ModularMonolith.Host` maps no liveness path; the AppHost adds the service route, Inventory's paths to `Inventory.Host`, whose `/alive` the proxy probes:

```csharp
// on the proxy project, between its references and its waits:
    .WithEnvironment("ReverseProxy__Routes__inventory__ClusterId", "inventory")
    .WithEnvironment("ReverseProxy__Routes__inventory__Match__Path", "/inventory/{**catch-all}")
    .WithEnvironment("ReverseProxy__Clusters__inventory__Destinations__host__Address", "http://inventory")
    .WithEnvironment("ReverseProxy__Clusters__inventory__HealthCheck__Active__Enabled", "true")
```

- A path that starts with a literal segment outranks the catch-all, so Inventory's paths reach `Inventory.Host` and every other path `ModularMonolith.Host`.
- The route is the AppHost's, not the proxy's settings', because only beside a broker does Inventory run in a Host of its own: without one, `ModularMonolith.Host` hosts Inventory and serves its paths, which the settings' catch-all route sends it when the proxy runs alone.
- The Kit turns each cluster's probe on unless its configuration says otherwise ([reverse-proxy.md](reverse-proxy.md#routes-clusters-and-discovery)); the AppHost sets it on beside the settings' `false`, so the recipe states both.
- The proxy's endpoint is its launch profile's, `http://localhost:5100` in its `Properties/launchSettings.json`, so it takes no `WithHttpEndpoint()`. Its `appsettings.json` names no listening URL: a `Urls` setting there overrides the `ASPNETCORE_URLS` Aspire passes, and the proxy listens where neither the AppHost's health check nor a client calls it.

## A local agent

A Host whose agent calls a model ([ai.md](ai.md)) can run it locally on Ollama. The Proving Sample's AppHost adds it only when it runs with `LocalAgent=true`, an argument or the environment variable `LocalAgent`, since the image and the model are large and the Host starts without them:

```csharp
var shopHost = builder.AddProject<Projects.ModularMonolith_Host>("modular-monolith") /* ...as above... */;

if (bool.TryParse(builder.Configuration["LocalAgent"], out var localAgent) && localAgent)
{
    const string Model = "qwen2.5:0.5b";
    var ollama = builder.AddPouryaKitOllama("ollama", Model);
    shopHost
        .WithEnvironment("PouryaKit__AI__Provider", "Ollama")
        .WithEnvironment("PouryaKit__AI__Endpoint", ReferenceExpression.Create($"{ollama.GetEndpoint("http")}/v1"))
        .WithEnvironment("PouryaKit__AI__Model", Model)
        .WaitFor(ollama);
}
```

- `AddPouryaKitOllama(name, model)` adds a plain container with an HTTP endpoint named `http` on Ollama's port 11434. A container resource has no connection string, so the Host gets its `PouryaKit:AI` section as environment variables: the endpoint with `/v1`, Ollama's OpenAI-compatible API, and the model the registration serves.
- Its health check is healthy only once the server has the model: on the first start it pulls the model into the volume, a download of several hundred megabytes, and `WaitFor` holds the Host until then. Aspire awaits a check with no deadline of its own, so each request has one, 10 seconds to say whether the server has the model and 30 minutes for the pull: an unreachable or silent server, or a pull that fails or outlasts its deadline, leaves it unhealthy, and Aspire checks again. Later starts find the model in the volume.
- The sample's agent is `POST /catalog/v2/products/ask` (or the unversioned path with the `api-version` header) (body `{"question": "..."}`), authenticated and tenant-scoped like every Catalog endpoint; send it a bearer token signed with the `bearer-signing-key` parameter.

## What each registration adds

| Call | Resource | Image | Volume target |
| - | - | - | - |
| `AddPouryaKitPostgres(name)` | `PostgresServerResource` | `docker.io/library/postgres:18.6` | `/var/lib/postgresql` |
| `AddPouryaKitRedis(name)` | `RedisResource` | `docker.io/library/redis:8.10.1` | `/data` |
| `AddPouryaKitRabbitMq(name)` | `RabbitMQServerResource` | `docker.io/library/rabbitmq:4.3.6`, no management plugin | `/var/lib/rabbitmq` |
| `AddPouryaKitOllama(name, model)` | `ContainerResource`, healthy once it serves `model` | `docker.io/ollama/ollama:0.40.6` | `/root/.ollama` |

- Each call but Ollama's is Aspire's own `AddPostgres`, `AddRedis` or `AddRabbitMQ` with the image pinned (`WithImage`) and `WithDataVolume()`, so it returns Aspire's resource builder and anything Aspire offers on it still applies; Ollama's is `AddContainer` with a volume Aspire names the same way.
- The tags are the ones the Kit's tests run. Aspire's own defaults float on a minor version (`18.3`, `8.6`, `4.3`).
- The volume's name is Aspire's, from the AppHost's project and the resource name (`modularmonolith.apphost-<hash>-postgres-data`): it outlives a run, so data persists between runs.
- The package adds nothing else: no dashboard, tracing, logging, metrics or API-reference resource (Jaeger, Zipkin, Tempo, Loki, Kibana, Prometheus, Grafana, Elasticsearch, Swagger). A Host on the service defaults exports its telemetry over OTLP to whatever `OTEL_EXPORTER_OTLP_ENDPOINT` names ([host-recipes.md](host-recipes.md)), and the Kit adds no collector.

## Running it

`dotnet run` on the AppHost project starts the containers, both Hosts and the Reverse proxy, headless; the proxy answers on `http://localhost:5100`. It needs Docker and the .NET SDK, no Aspire workload or CLI, and the `bearer-signing-key` parameter set first, in the AppHost's user secrets or as the environment variable `Parameters__bearer-signing-key`. Add `LocalAgent=true` after `--` for Ollama and the agent's model.

A named volume keeps the credentials it was first initialised with. Aspire generates each server's password as a parameter (`postgres-password`, `cache-password`, `broker-password`) and keeps it in the AppHost's user secrets, which is why the AppHost has a `UserSecretsId` and runs in Development, where it reads them back. Set a password yourself, and the volume made with another one refuses it: remove the volume (`docker volume rm`) or keep the first password.

## Testing

The Kit's AppHost tests run on `Aspire.Hosting.Testing`, whose `DistributedApplicationTestingBuilder` runs the AppHost. The testing builder replaces the AppHost's options with its own, dashboard off whatever the AppHost says, so the start test passes the sample's own choice, `AppHostOptions.Create([]).DisableDashboard`, to the builder's options callback:

```csharp
await using var builder = await DistributedApplicationTestingBuilder.CreateAsync<Projects.ModularMonolith_AppHost>(
    ["Parameters:postgres-password=apphost-tests", "Parameters:bearer-signing-key=<base64 key>"],
    (options, _) => options.DisableDashboard = AppHostOptions.Create([]).DisableDashboard,
    cancellationToken
);
await using var app = await builder.BuildAsync(cancellationToken);
await app.StartAsync(cancellationToken);
await app.ResourceNotifications.WaitForResourceHealthyAsync("inventory", cancellationToken);
using var inventory = app.CreateHttpClient("inventory");
```

- The model test reads the builder without starting it: every resource's name, as an allow-list, each container's image and its one volume mount, the resources each Host references, and the two Hosts the proxy references; with no `LocalAgent`, no Ollama. A second model test builds it with `LocalAgent=true`: the allow-list gains `ollama` alone, on its pinned image and named volume, with its model check, and `ModularMonolith.Host` waits for it with its `PouryaKit__AI__Provider`, `__Endpoint` and `__Model` set, which `Inventory.Host` is not given. Neither starts anything, so no image or model is downloaded.
- The route test reads the proxy's configuration as it runs under the AppHost: its `appsettings.json`, then the environment the AppHost gives it, each endpoint as its expression. Inventory's route goes to the cluster `inventory`, whose destination `http://inventory` has its probe `true` and resolves through `Services:inventory:http:0`, the endpoint of the `inventory` resource; the catch-all route goes to `http://modular-monolith`, with its probe `False` and its own reference.
- The start test runs the containers, both Hosts and the proxy as processes, waits for each Host and for the proxy's `/alive`, and asks `Inventory.Host`'s `/alive`, which its health check already answered, and an endpoint of `ModularMonolith.Host`, which refuses an anonymous caller with 401 once it serves; that Host has no health check, so its resource is healthy once its process runs, and the test asks again until it answers, within its timeout. Through the proxy, a caller with a bearer token signed with the test's key gets, on Inventory's unversioned path, `Inventory.Host`'s 400 `ApiVersionUnspecified`, where `ModularMonolith.Host`, which hosts no Inventory beside a broker, answers 404, and on Catalog's unversioned path `ModularMonolith.Host`'s 400 `ApiVersionUnspecified`, where `Inventory.Host`, which hosts no Catalog, answers 404, as the test asks it directly. An anonymous caller proves neither route: both Hosts refuse it with 401 on any path. The running model is still the allow-list, so no dashboard joined it.
- The same start test calls Catalog's `ProductService.GetProduct` through Aspire's endpoint proxy on each of `ModularMonolith.Host`'s endpoints, with the test's bearer token and a product id that is no Guid, which the service refuses before any query. On `grpc` the call reaches the service, which answers `InvalidArgument` "product_id is not a Guid.", on a port other than the Host's listener's, `Kestrel__Endpoints__grpc__Url` in the running resource's environment; on `http` it fails with `HTTP_1_1_REQUIRED`. Aspire's endpoint proxy carries a gRPC call, HTTP/2 with prior knowledge over cleartext, to a listener that speaks it.
- Run the model check against a loopback server that accepts and never answers, once for `/api/show` and once for the pull after a 404: each check returns unhealthy when its deadline passes.
- The local agent's start test runs only with `POURYAKIT_OLLAMA_PROBE=1`, the variable of the Ollama probe ([ai.md](ai.md#testing)): it starts the AppHost with `LocalAgent=true`, waits for Ollama's model and the Host, and posts a question to the agent with a bearer token it signs with the test's key, expecting 200 and the model's text. A first run pulls the 7 GB image and the model; its timeout is 30 minutes.
- The tests fix each server's password, and the start test moves each volume to a name ending `-tests`: its volumes serve its next run, and a developer's, made with the passwords the AppHost generated, are never opened with the test's, which would fail every health check.
- The project shares no fixture with the application's tests: the AppHost starts the Hosts as processes, never in memory.

## Limits

- The model is pulled only when the resource starts, by the health check; a model the AppHost stops serving stays in the volume until it is removed.
- The registrations know nothing of a Host's composition: an AppHost that adds Redis for a Host that registers no Redis cache runs a container nothing uses.
- The AppHost applies no EF Core migration, and no Host applies one at start ([persistence.md](persistence.md#migrations)): the databases it creates hold no table of an EF Core Module (Catalog, Billing, Inventory) until a developer applies each one's migrations to them, with the connection string Aspire gives the Host, so until then an Operation that reads or writes them fails; Ledger's Marten schema is the exception, applied when the Host starts (`UsePouryaKitMarten`). The start test asks `Inventory.Host` through the proxy for an answer it gives before any query.
- Every endpoint the AppHost gives a Host is cleartext HTTP, so a gRPC client under the AppHost calls `ModularMonolith.Host`'s `grpc` endpoint: its `http` endpoint refuses the call with `HTTP_1_1_REQUIRED`, through Aspire's proxy or directly, and the Reverse proxy carries no gRPC ([reverse-proxy.md](reverse-proxy.md#limits)). A Host whose gRPC service the AppHost should serve takes a second endpoint the same way; `Inventory.Host` maps none.
- `ModularMonolith.Host` maps no liveness path, so the AppHost waits on it only until its process runs. The sample composes the service defaults in `Inventory.Host` and `ModularMonolith.ReverseProxy`, not in `ModularMonolith.Host` ([host-recipes.md](host-recipes.md#composing-a-host-with-them)). Every Host the project templates generate composes them and maps `/alive`.
- The Proving Sample's AppHost provisions no SMTP server and no S3 store: `ModularMonolith.Host` sends mail to the SMTP server and stores files in the bucket its settings name, a local Papercut on port 2525 and a local RustFS on port 9000 with the bucket modular-monolith, which the Kit does not create. A developer runs both; without the store, a report job cannot store its file. The recipe shows the Kit's own resources, those of `PouryaKit.Aspire`, and a project adds the SMTP server and the file store of the providers it chose.
