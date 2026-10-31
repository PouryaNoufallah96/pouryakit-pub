# Reverse proxy

`PouryaKit.ReverseProxy`: a Host of its own process, on YARP, in front of a system's Hosts. It forwards each request by the routes and clusters of its configuration to a Host that service discovery resolves, probes each Host's liveness path, and runs no Operation. Namespace `PouryaKit.ReverseProxy`. It references the ASP.NET Core shared framework, `Yarp.ReverseProxy` and `Microsoft.Extensions.ServiceDiscovery.Yarp`, and no Kit package. In the glossary it is the Reverse proxy; a Gateway is a Module's own entry point.

- [Composing the proxy](#composing-the-proxy)
- [Routes, clusters and discovery](#routes-clusters-and-discovery)
- [What the Kit guarantees](#what-the-kit-guarantees)
- [Limits](#limits)
- [Testing](#testing)

## Composing the proxy

The proxy is an application of its own, beside the Hosts it fronts. The Proving Sample's is `ModularMonolith.ReverseProxy`, and the monolith and modular monolith templates generate `src/<name>.ReverseProxy` in front of their Host, the service template none ([templates.md](templates.md)):

```csharp
using PouryaKit.ReverseProxy;

var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();
builder.AddPouryaKitReverseProxy();

var app = builder.Build();
app.MapDefaultEndpoints();
app.MapReverseProxy();
await app.RunAsync();
```

- `AddServiceDefaults()` is the application's service defaults ([host-recipes.md](host-recipes.md)): its OpenTelemetry records the proxy's spans, and its `AddServiceDiscovery()` registers the endpoint providers that resolve a destination. Without them, discovery has no provider to ask.
- `AddPouryaKitReverseProxy()` loads the routes and clusters, adds the liveness probe and service discovery's destination resolver. `MapReverseProxy()` is YARP's own.
- The proxy authenticates no one and adds no Kit pipeline; each Host it fronts authenticates its own callers.

## Routes, clusters and discovery

Routes and clusters are YARP's configuration, in the `ReverseProxy` section (`ReverseProxyHostApplicationBuilderExtensions.SectionName`), and change with it at run time. A destination whose address names a service resolves into one destination per endpoint the `Services` section, or another provider, lists for it:

```json
{
  "ReverseProxy": {
    "Routes": {
      "shop": { "ClusterId": "shop", "Match": { "Path": "{**catch-all}" } }
    },
    "Clusters": {
      "shop": { "Destinations": { "host": { "Address": "http://shop" } } }
    }
  },
  "Services": {
    "shop": { "http": ["localhost:5080"] }
  }
}
```

- Each cluster probes `/alive` on every destination, the liveness path a Host composing the service defaults maps in every environment, every 15 seconds; two failed probes in a row take the destination out of the cluster until a probe succeeds. A cluster's own `HealthCheck:Active` settings win: `Path` names another path, `Interval` another period, and `Enabled: false` turns the probe off for a Host that maps no `/alive`, as the Proving Sample's settings do for `ModularMonolith.Host`.
- When every destination of a cluster is out, the cluster answers its requests with 503 until a probe succeeds: the Kit sets the cluster's `HealthCheck:AvailableDestinationsPolicy` to `HealthyAndUnknown` unless its configuration names another, where YARP's default, `HealthyOrPanic`, would forward to every destination again.
- A route whose path starts with a literal segment outranks a catch-all route, with no `Order`. The Proving Sample's settings route every path to `ModularMonolith.Host`; its AppHost adds the service route through the proxy's environment, `/inventory/{**catch-all}` to the cluster `inventory`, whose destination `http://inventory` resolves to `Inventory.Host` through the endpoint the AppHost's reference gives the proxy, with the probe on, since that Host maps `/alive` ([aspire.md](aspire.md#the-apphost)).

## What the Kit guarantees

- A route's destination that names a service is resolved by service discovery into the service's endpoints, and an endpoint whose liveness probe fails drops out of its cluster while the others keep answering.
- A request keeps the client's trace: the trace id arrives at the Host unchanged, and the parent of the Host's request is the proxy's own span, never the client's span passed through. YARP's default propagator does it, and the service defaults' HttpClient instrumentation writes the same context. A Host's Operation is a child of its request, so the whole request is one trace from the client through the proxy to the Operation.
- The Kit strips no header: the client's bearer token reaches the Host as the client's own `Authorization` header, and the Kit's Operation Context headers as sent. YARP's standard transforms still set `Host` to the destination's, add the `X-Forwarded-*` headers, drop the hop-by-hop and HTTP/2 protocol headers and replace the trace context with the proxy's span. A Host honours those headers only from a caller it authenticates as a trusted peer ([security.md](security.md#continuations)), so a client that is not one gains nothing by sending them through the proxy.
- No Operation runs in the proxy: no Kit package reaches its closure.

## Limits

- Liveness only: the proxy probes `/alive`, the recipe's liveness path, and no readiness path, since readiness outside Development is the platform's probe. A Kit readiness option would duplicate what the platform's probe does over the Host's liveness path and dependencies; `/health`, which asks each database, stays a Development endpoint of the service defaults ([host-recipes.md](host-recipes.md#liveness-and-readiness)).
- No transform, rate limit, authentication or response cache of the proxy's own; add YARP's where the application needs them, in its own composition.
- Routes and clusters come from configuration only; there is no code-built route.
- As shipped, the proxy carries no gRPC: it listens without TLS and forwards to `http://` destinations, where YARP sends HTTP/1.1. The Kit adds no HTTP/2 destination option: a Host's gRPC client targets the other Host directly, at the address its `PouryaKitGrpcClientOptions` name ([grpc.md](grpc.md#calling-another-host)), so the proxy fronts the HTTP API alone. An application that must proxy gRPC follows YARP's gRPC guidance in its own composition.

## Testing

Front a Host that listens on a real port, so YARP forwards over the network as it does when deployed, and serve the proxy in memory. The Proving Sample's tests run `ModularMonolith.Host` on Kestrel with the service defaults' health checks and its bearer scheme, then composes the proxy with the Host's endpoint in the `Services` section:

```csharp
var builder = ReverseProxyHost.CreateBuilder(
    [
        "--ReverseProxy:Routes:shop:ClusterId=shop",
        "--ReverseProxy:Routes:shop:Match:Path={**catch-all}",
        "--ReverseProxy:Clusters:shop:Destinations:host:Address=http://shop",
        $"--Services:shop:http:0={hostAddress}",
    ]
);
builder.WebHost.UseTestServer();
await using var proxy = ReverseProxyHost.Build(builder);
await proxy.StartAsync(TestContext.Current.CancellationToken);
using var client = proxy.GetTestClient();
```

Its tests read the cluster's destinations and their health from YARP's `IProxyStateLookup`, and the trace from an `ActivityListener` on the `Microsoft.AspNetCore`, `System.Net.Http` and Kit sources, since both Hosts run in the test process. The service route's test fronts `Inventory.Host` the same way, on the broker fixture: its `GetStockItem` answers the stock item a delivery opened for the token's tenant, and an `IStartupFilter` of the test records the token and the Kit's headers as the Host's pipeline receives them.
