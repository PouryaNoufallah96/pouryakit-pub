# gRPC

`PouryaKit.Grpc`: a Module's gRPC services mapped on Web's conventions, a failed `Result` answered as a gRPC status with every Error in a trailer, and clients of another Host that carry the Operation Context through Web's one header codec. It references `PouryaKit.Web`, `Grpc.AspNetCore.Server` and `Grpc.Net.ClientFactory`. Namespace `PouryaKit.Grpc`. Contracts are Protobuf only.

- [The contract](#the-contract)
- [Declaring and mapping a service](#declaring-and-mapping-a-service)
- [Failures](#failures)
- [Calling another Host](#calling-another-host)
- [The loopback rule](#the-loopback-rule)
- [Limits](#limits)
- [Testing](#testing)

## The contract

The `.proto` lives in the Module's Contracts project, which both sides compile: the Module's service base and another Host's client. Contracts reference nothing of the Kit; they take the code generator and the Protobuf runtime themselves:

```xml
<ItemGroup>
  <PackageReference Include="Google.Protobuf" />
  <PackageReference Include="Grpc.Core.Api" />
  <PackageReference Include="Grpc.Tools" PrivateAssets="all" />
</ItemGroup>
<ItemGroup>
  <Protobuf Include="Products/products.proto" />
</ItemGroup>
```

Set `option csharp_namespace` to the namespace the folder makes (`Catalog.Contracts.Products`), and name the service so its generated class clashes with no namespace (`ProductService`, not `Products`).

## Declaring and mapping a service

In the Module's Composition Root, and once in the Host after `MapPouryaKitModuleEndpoints`:

```csharp
module.AddPouryaKitGrpcService<ProductGrpcService>();

app.MapPouryaKitGrpcServices();
```

```csharp
internal sealed class ProductGrpcService(IGatewayProcessor<CatalogModule> catalog, OperationContext context)
    : ProductService.ProductServiceBase
{
    public override async Task<ProductReply> GetProduct(GetProductRequest request, ServerCallContext call) =>
        Guid.TryParse(request.ProductId, CultureInfo.InvariantCulture, out var productId)
            ? (await catalog.QueryAsync(new GetProductQuery(productId), context, call.CancellationToken))
                .ToGrpcResponse(product => new ProductReply { Id = product.Id.ToString(), Name = product.Name })
            : throw new RpcException(new Status(StatusCode.InvalidArgument, "product_id is not a Guid."));
}
```

- `MapPouryaKitGrpcServices()` maps every declared service on one route group at the root, never inside a Module's prefixed or versioned group: a gRPC method's path is its Protobuf package, service and method. The group requires an authenticated caller and carries Web's `BeginOperationPerRequest()`, so the call's Operation begins before the service is created, from the request's scope, and the service injects the `OperationContext` and passes it to its Module's Gateway, as an HTTP endpoint does ([web.md](web.md)). Called again, it maps no service twice: two identical endpoints would make every call to the service ambiguous.
- The Actor is the authenticated caller. A caller authenticated as a peer in `PouryaKitTrustedPeerOptions.Peers` continues its own Operation from the Kit headers in its call metadata; every other caller's Kit headers are ignored, as over HTTP.
- A service of a Module that also maps HTTP endpoints answers beside them; nothing of the HTTP pipeline changes.
- Over TLS, one Kestrel listener serves gRPC and HTTP/1.1 alike: TLS negotiates the protocol. Without TLS a listener of both versions, Kestrel's default, negotiates nothing and speaks HTTP/1.1, so every gRPC call fails with `HTTP_1_1_REQUIRED`: ASP.NET Core's troubleshooting guide says an insecure gRPC service must be hosted on an HTTP/2-only port. That is a `Kestrel:Endpoints:Grpc` entry with `Kestrel:Endpoints:Grpc:Protocols` set to `Http2`, and since `Kestrel:Endpoints` replace `urls` and the ports, the HTTP/1.1 listener becomes an entry of its own beside it: two entries, since the gRPC entry alone leaves REST with no listener. `Kestrel:EndpointDefaults:Protocols` set to `Http2` makes every listener HTTP/2 only, which a REST client over HTTP/1.1 cannot reach.
- In development, the monolith's and the modular monolith's `https` launch profile listens on `https://localhost:7080` and `http://localhost:5080` with no such entry: the https URL serves REST and gRPC, and a gRPC call on the http URL fails with `HTTP_1_1_REQUIRED` ([templates.md](templates.md)).
- Under the Proving Sample's AppHost, `ModularMonolith.Host` has the two entries as two Aspire endpoints: `http` for REST and `grpc`, its listener of HTTP/2 alone. Aspire's endpoint proxy carries a gRPC call to `grpc` over cleartext, and on `http` the call fails with `HTTP_1_1_REQUIRED`, through the proxy or directly ([aspire.md](aspire.md#the-apphost)).
- In production, as a deployment setting and no template default: with a certificate on Kestrel, TLS serves REST and gRPC on one port, as ASP.NET Core's gRPC security guidance asks of production apps. Behind an ingress or proxy that ends TLS and forwards without it, such as a container on `ASPNETCORE_HTTP_PORTS=8080`, a considered choice in that guidance, configure the two entries (`Kestrel__Endpoints__Http__Url=http://*:8080`, `Kestrel__Endpoints__Grpc__Url=http://*:8081`, `Kestrel__Endpoints__Grpc__Protocols=Http2`), and have the ingress send HTTP/2 to the gRPC port.
- A Host that declares a service and never calls `MapPouryaKitGrpcServices()` fails when it starts, naming the service: a startup check, as for unmapped endpoints.

## Failures

One shape per transport: HTTP answers a problem response, gRPC a status.

| Error kind | Status |
| - | - |
| Validation | `InvalidArgument` |
| Not found | `NotFound` |
| Conflict | `Aborted` |
| Forbidden | `PermissionDenied` |
| Unexpected | `Internal` |

- `result.ToGrpcResponse(response)` and `result.ToGrpcResponse(value => response)` return the response on success and throw an `RpcException` on failure. When the Errors are of several kinds the most severe decides, in HTTP's order: unexpected, forbidden, not found, conflict, validation. The status's message is the deciding Error's.
- Every Error, in order, is in the binary trailer `ResultGrpcExtensions.ErrorsTrailer` (`pouryakit-errors-bin`), a JSON array in the Kit's representation whose entries are those of an HTTP problem's `errors` member: `code`, `message` and, for a validation Error with one, `target`. A client reads it from `RpcException.Trailers`.
- A trusted peer's Kit headers the Kit cannot read answer `InvalidArgument` with the rule they broke as the message, before the service runs, where HTTP answers a 400 problem response.
- An unauthenticated call answers `Unauthenticated`, from the HTTP 401 authorization writes.
- A call to a service no Module declared answers `Unimplemented`, from routing's HTTP 404: the Kit maps no catch-all endpoint for unknown services, since it would accept every method on any two-segment path and turn a Module's 405 and 415 into 404.

## Calling another Host

On the calling Host, for each generated client, under a name of its own:

```csharp
builder.Services.AddPouryaKitGrpcClient<ProductService.ProductServiceClient>("catalog");
builder.Services.Configure<PouryaKitGrpcClientOptions>("catalog", builder.Configuration.GetSection("PouryaKit:Grpc:Clients:catalog"));
```

```json
"PouryaKit": { "Grpc": { "Clients": { "catalog": { "Address": "https://catalog.internal", "Deadline": "00:00:10" } } } }
```

```csharp
var reply = await products.GetProductAsync(
    new GetProductRequest { ProductId = id.ToString() },
    new CallOptions(cancellationToken: cancellationToken).WithOperationContext(context)
);
```

- `PouryaKitGrpcClientOptions`: `Address`, an absolute `http` or `https` address, and `Deadline`, positive, both required. The Host fails when it starts, naming the client, when either is missing or invalid.
- The client's interceptor writes the Operation Context each call is given with `WithOperationContext(context)` as Web's Kit header pairs (`OperationContextHttpHeaders.Pairs`): the Actor, the tenant, the correlation, and the calling Operation's `OperationId` as the causation. Every Kit header the call's metadata already held is removed first, so a tenant given beside a context without one never reaches the peer. A call without a context throws `InvalidOperationException` before it is sent. A typed client injects the `OperationContext` of its Operation and passes it.
- The options' `Deadline` applies to every call that sets no deadline of its own. One that would end after `DateTime.MaxValue`, such as `TimeSpan.MaxValue`, sends the call with no deadline: the interceptor gives it `DateTime.MaxValue`, which gRPC treats as none.
- Registering the same client under the same name again adds nothing and returns the builder of the first registration, so the interceptor runs once per call; the same name for another client throws gRPC's client factory's `InvalidOperationException`.
- Nothing of an incoming request is copied, its bearer token least of all. A peer that must authenticate this Host gets the Host's own credentials, added on the builder `AddPouryaKitGrpcClient` returns; the receiving Host lists this Host's Actor claim in its trusted peers.

## The loopback rule

A gRPC client never calls its own Host: a call between Modules of one Host goes through the Gateway, synchronously and inside the asking Operation ([modules.md](modules.md)). The Host's start fails, naming the client and the URL, when a client's address is one of the URLs the Host listens on, chosen as ASP.NET Core chooses them: the addresses the application adds after `Build` (`app.Urls`), or else `urls`, or else `http_ports` and `https_ports`; each `Kestrel:Endpoints:{name}:Url` replaces those unless `preferHostingUrls` is set; a client there fails the start before any hosted service starts. Then, once every hosted service has started and only when the Host's server is Kestrel, the addresses Kestrel bound are compared the same way, so a listener set in code (`ConfigureKestrel` with `Listen` or `ListenLocalhost`), a port 0 and Kestrel's default `http://localhost:5000`, the one address ASP.NET Core 10 binds when nothing is configured (no HTTPS default; a launch profile's `applicationUrl` arrives as `urls`), are all seen; a client there fails the start too, but the Host had already bound its addresses, so it listened, and its other hosted services ran, before the check stopped the Host and the start failed: configure the addresses to have the refusal before anything starts. Kestrel binds its default only when nothing else is configured, so a client at `http://localhost:5000` beside a listener set in code starts. A Host without a server, a client-only Generic Host, or one on another server such as the test server, gets only the first comparison. A URL the Host overrides, such as the image's `http_ports` 8080 under `urls`, is not the Host's, so a sidecar on it starts. An address matches on its scheme and port when its host is the same, or is loopback (`localhost`, `127.0.0.1`, `[::1]`) while the URL is loopback or listens on every address: `0.0.0.0`, `[::]`, and, as Kestrel binds them, every host that is neither `localhost` nor an IP address (`*`, `+`, or a name such as `catalog.local`). Another host on the same port, such as a peer container on the 8080 every ASP.NET Core image listens on, is another Host and starts. The first comparison is a startup check because `app.Urls` exists only on the built application; the second waits for the bind because Kestrel keeps the listeners set in code to itself until then. Another name for the same machine is not seen and stays this rule's.

## Limits

- Before the start, the loopback check reads the configured URLs even when listeners set in code override them: a client at a configured address that a `Listen` or `ListenLocalhost` call replaced, such as a `urls` entry, fails the start though the Host never listens there. The false refusal is narrow and hides no refusal, since the comparison after the bind sees the listeners set in code; Kestrel exposes no public reading of those listeners before it binds them, so the first comparison cannot leave the replaced addresses out. Configure no address that a listener set in code replaces.

## Testing

- Two Hosts of the application run in one test process, each built with `AppFixture.ComposeHost` and a client of the other: `ConfigurePrimaryHttpMessageHandler(() => new Http2(peer.GetTestServer().CreateHandler()))`, where `Http2` is a `DelegatingHandler` that sets the response's `Version` to the request's, since the in-memory server answers HTTP/1.1 and a gRPC client refuses that. Add the peer credentials with `AddHttpMessageHandler`.
- Across a process boundary, `ModularMonolithProcess` runs the sample Host with `dotnet run` in Production, configured through its environment, and a Host of the test process calls the address it logs, authenticated with the peer's own bearer token. A Kestrel listener without TLS serves gRPC only with HTTP/2 alone (`Kestrel:EndpointDefaults:Protocols` set to `Http2`): without TLS no negotiation picks HTTP/2, so Kestrel's default listener of both versions speaks HTTP/1.1 and refuses the call with `HTTP_1_1_REQUIRED`; under the AppHost, a start test calls the service through Aspire's endpoint proxy ([aspire.md](aspire.md#testing)). Read what the process ran from its JSON log entries with scopes (`Logging:Console:FormatterName` `json`, `IncludeScopes`): the dispatcher's scope names each Operation's id, correlation and Module.
- A plain `GrpcChannel` over the same handler sends exactly the metadata a call names, such as a raw peer's unreadable headers.
- Read a failure's Errors with `JsonDocument.Parse(failure.Trailers.Get(ResultGrpcExtensions.ErrorsTrailer)!.ValueBytes)`.
