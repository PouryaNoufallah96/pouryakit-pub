# Web

`PouryaKit.Web`: an endpoint beside each Feature, authenticated access by default, a Result as an HTTP response, one problem response for every failure, the Operation Context built from the authenticated caller or continued from a trusted peer Host, outgoing HTTP that carries the Operation Context with the standard retry, and configured CORS policies, response compression and a rate limit per Actor. Namespace `PouryaKit.Web`. It references `PouryaKit.Cqrs`, `PouryaKit.Core`, the ASP.NET Core shared framework and `Microsoft.Extensions.Http.Resilience`; a Module project that declares endpoints references it, and it brings ASP.NET Core with it.

- [The Host](#the-host)
- [Endpoints](#endpoints)
- [From Result to response](#from-result-to-response)
- [A Page over HTTP](#a-page-over-http)
- [Problem responses](#problem-responses)
- [The caller and the Actor](#the-caller-and-the-actor)
- [Calls between Kit Hosts](#calls-between-kit-hosts)
- [CORS, compression and rate limits](#cors-compression-and-rate-limits)
- [Testing over HTTP](#testing-over-http)

The Proving Sample's `CreateProduct` endpoint, beside its command in `Products/Features/CreateProduct/`:

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;
using PouryaKit.Cqrs;
using PouryaKit.Modularity;
using PouryaKit.Web;

internal sealed class CreateProductEndpoint : IEndpoint
{
    public void Map(IEndpointRouteBuilder routes) =>
        routes.MapPost(
            "/products",
            async (
                CreateProductBody body,
                OperationContext context,
                IGatewayProcessor<CatalogModule> catalog,
                CancellationToken cancellationToken
            ) =>
                (
                    await catalog.SendAsync(
                        new CreateProductCommand(body.ProductId, body.CategoryId, body.Name),
                        context,
                        cancellationToken
                    )
                ).ToHttpResult(id => TypedResults.Created($"/catalog/products/{id}", new { id }))
        );
}

internal sealed record CreateProductBody(Guid ProductId, Guid CategoryId, string Name);

// In the Composition Root: module.AddPouryaKitEndpoints("/catalog");
// The Proving Sample's Catalog declares API versions, so its CreateProduct generates the location with LinkGenerator
// in the requested version instead (openapi.md).
```

## The Host

The Proving Sample's Host composition, one class shared by `Program` and the tests:

```csharp
using Billing;
using Catalog;
using Catalog.Products.Events.Notifications;
using Inventory;
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using PouryaKit.Messaging.Wolverine.EFCore;
using PouryaKit.Modularity;
using PouryaKit.Security;
using PouryaKit.Security.JwtBearer;
using PouryaKit.Web;
using PouryaKit.Web.OpenApi;

public static class ModularMonolithHost
{
    public const string DownstreamClient = "downstream";

    public const string BrowserPolicy = "browser";   // the CORS policy the pipeline answers with
    public const string TenantClaim = "tenant";

    public static WebApplicationBuilder CreateBuilder(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);
        // Bearer tokens, verified against Authentication:Schemes:Bearer (security.md); "sub" is the Actor.
        builder.Services.AddPouryaKitJwtBearer(claims => claims.TenantClaim = TenantClaim);
        builder.Services.AddAuthorization();
        // Also: authenticated access by default. The tenant is the one the bearer scheme verified.
        builder.Services.AddPouryaKitWeb();
        // The peer Hosts that may continue an Operation of theirs here; none by default.
        builder.Services.Configure<PouryaKitTrustedPeerOptions>(builder.Configuration.GetSection("PouryaKit:TrustedPeers"));
        // Named CORS policies, response compression and the rate limit per Actor; each is off until configured.
        builder.Services.Configure<PouryaKitCorsOptions>(builder.Configuration.GetSection("PouryaKit:Web:Cors"));
        builder.Services.Configure<PouryaKitCompressionOptions>(builder.Configuration.GetSection("PouryaKit:Web:Compression"));
        builder.Services.Configure<PouryaKitRateLimitOptions>(builder.Configuration.GetSection("PouryaKit:Web:RateLimit"));
        // The API versions the Modules declare, one OpenAPI document each, and the configured sunsets (openapi.md).
        builder.Services.AddPouryaKitOpenApi();
        builder.Services.Configure<PouryaKitOpenApiOptions>(builder.Configuration.GetSection("PouryaKit:OpenApi"));
        // Outgoing calls: the standard retry of safe methods, then the Operation Context on every attempt.
        var downstream = builder.Services.AddHttpClient(DownstreamClient);
        downstream.AddPouryaKitResilience();
        downstream.AddPouryaKitOperationContext();
        // The permissions Catalog's RemoveProducts declares are the HTTP caller's permission claims (security.md).
        builder.Services.AddHttpContextAccessor();
        builder.Services.AddScoped<IActorPermissions, CallerPermissions>();
        builder.AddPouryaKitModules(modules =>
        {
            modules.Add<CatalogModule>();
            modules.Add<InventoryModule>();
            modules.Add<BillingModule>();
        });
        builder.UsePouryaKitWolverine(connectionName: "app");   // the Modules' deliveries (messaging.md)
        builder.Services.AddSingleton<INewProductAnnouncer, LoggedNewProductAnnouncer>();
        return builder;
    }

    public static WebApplication Build(WebApplicationBuilder builder)
    {
        var app = builder.Build();
        app.UseExceptionHandler();   // first, so an exception in routing or authentication still answers a problem response
        app.UseRouting();
        app.UseCors(BrowserPolicy);   // before authentication, so a preflight is answered without credentials
        app.UseAuthentication();
        app.UseAuthorization();
        app.UseRateLimiter();         // after authorization: it partitions by the authenticated Actor
        app.MapPouryaKitModuleEndpoints();
        if (app.Environment.IsDevelopment())
        {
            app.MapPouryaKitOpenApi().AllowAnonymous();   // the documents and Scalar, public in Development
        }

        return app;
    }
}

// Program.cs: return await ModularMonolithHost.Build(ModularMonolithHost.CreateBuilder(args)).RunJasperFxCommands(args);
```

- `builder.Services.AddPouryaKitWeb()` registers what every endpoint answers with: problem responses, the handler that turns an exception into one, the same answer to an unreadable request in every environment, a problem response for every failure answered with an empty body ([Problem responses](#problem-responses)), the Kit's one JSON representation (`UsePouryaKitJson`, typed IDs written as their bare values, the same as in messages, and a body that names one member twice, in any case, refused as a request that cannot be read rather than read with its last value; [domain-primitives.md](domain-primitives.md#typed-ids)), `PouryaKitWebOptions`, authenticated access by default ([The caller and the Actor](#the-caller-and-the-actor)), and the configured CORS policies, response compression and rate limit, each off until the Host configures it ([CORS, compression and rate limits](#cors-compression-and-rate-limits)).
- The application configures its own authentication and authorization; the Kit chooses no identity provider and issues no token.
- `app.UseExceptionHandler()` comes first in the pipeline, and `UseRouting()`, `UseAuthentication()` and `UseAuthorization()` follow it, each called explicitly. Left out, `WebApplication` inserts those three ahead of everything the application adds, and an exception thrown in authentication escapes the handler.
- Compose the rest in this order: `UseExceptionHandler()`, `UseRouting()`, `UseCors(policyName)`, `UseAuthentication()`, `UseAuthorization()`, `UseRateLimiter()`, then the endpoints. CORS answers a preflight before authentication would refuse it. The rate limiter runs after authorization, because it partitions by the authenticated Actor and an unauthenticated caller of a protected endpoint is refused before it counts. Compression needs no call: when enabled, `AddPouryaKitWeb` adds it outermost, before anything that writes a response.
- The Kit checks the rate limiter's place on each request it partitions. In a Host that registers authentication, an enabled rate limit throws at a request that reaches it before authentication ran, naming the order, rather than count every caller in one partition. `WebApplication` inserts `UseAuthentication()` ahead of the application's middleware when the Host does not call it, so only a `UseAuthentication()` called after `UseRateLimiter()` makes it throw. A Host without authentication has only anonymous callers, each partitioned by the Actor its endpoint names. A limiter before `UseRouting()` is not detected, because routing marks no request it matched nothing for: it puts the anonymous callers of a public endpoint in the one partition of requests no Actor resolves for.
- `app.MapPouryaKitModuleEndpoints()` maps every Module's endpoints, each Module under its own route group. A Module that declares API versions has them served inside that group, by URL segment and by header, through the `IModuleRouteGroups` that the optional `PouryaKit.Web.OpenApi` registers ([openapi.md](openapi.md)); without it, or for a Module that declares none, the endpoints are mapped on the group itself. With the Kit's checks (`AddPouryaKitCompositionChecks`, which `AddPouryaKitModules` calls), a startup check fails the Host's start, since the application maps its routes after `Build()`, when an `IEndpoint` in the Feature assemblies of a Module it composed, or of its own services, is declared by no `AddPouryaKitEndpoints`, or declared and mapped by no `MapPouryaKitModuleEndpoints`, naming the call: such an endpoint would never answer.
- `group.BeginOperationPerRequest()` on a `RouteGroupBuilder` is the convention every Module's group carries: before an endpoint binds, it resolves the caller and begins the request's Operation, a trusted peer's headers continuing its own. It applies once per endpoint, so a nested group's call is harmless. Apply it, with `RequireAuthorization()`, to a root group outside every Module's, such as another transport's, and its endpoints inject `OperationContext` as a Module's endpoints do.

## Endpoints

- Write one `internal sealed` class implementing `IEndpoint` beside its Feature, named after it with the `Endpoint` suffix, with a parameterless constructor. Its `Map(routes)` maps the Feature's route with a minimal API (`MapGet`, `MapPost`, ...) on its Module's route group, so the path is relative to the group's prefix.
- `module.AddPouryaKitEndpoints(routePrefix)` in the Module's Composition Root finds every `IEndpoint` in the Module's own assembly, internal ones included, and maps them under `routePrefix` (`/catalog`); assemblies named after the prefix replace it ([modules.md](modules.md#the-composition-root)). There is no central route list. It throws when the assemblies have no endpoint, and when its route prefix, compared ignoring case, as the router matches a literal, or one of its assemblies is already declared, whose endpoints would be mapped twice. An application without Modules calls it on its own services, naming its assembly.
- The endpoint injects `OperationContext`, its Module's `IGatewayProcessor<TModule>` and the `CancellationToken`, builds the command or query from the route and the body, sends it through the Gateway with that context, and returns `ToHttpResult()`. It holds no business logic and touches no repository or DbContext.
- Declare a request body as an `internal sealed record` beside the endpoint (`CreateProductBody`) and map it to the command by hand.
- Leave a route parameter unconstrained (`/products/{productId}`, not `{productId:guid}`): an id that does not parse is the caller's mistake and answers 400, where a constraint would answer 404. Check input in the command's or query's validator, which answers 400 with every failure.

The `GetProduct` and `ListProducts` endpoints, in their own Feature folders:

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Routing;
using PouryaKit.Cqrs;
using PouryaKit.Modularity;
using PouryaKit.Web;

internal sealed class GetProductEndpoint : IEndpoint
{
    public void Map(IEndpointRouteBuilder routes) =>
        routes.MapGet(
            "/products/{productId}",
            async (
                Guid productId,
                OperationContext context,
                IGatewayProcessor<CatalogModule> catalog,
                CancellationToken cancellationToken
            ) => (await catalog.QueryAsync(new GetProductQuery(productId), context, cancellationToken)).ToHttpResult()
        );
}

// Authenticated, as every Catalog endpoint is: Catalog is tenant-scoped, and a tenant comes only from a verified claim,
// so an anonymous visitor would have none.
internal sealed class ListProductsEndpoint : IEndpoint
{
    public void Map(IEndpointRouteBuilder routes) =>
        routes.MapGet(
            "/products",
            async (
                HttpPageRequest page,              // ?page=2&size=20&sort=name&direction=desc&filter[name][contains]=lamp
                OperationContext context,
                IGatewayProcessor<CatalogModule> catalog,
                CancellationToken cancellationToken
            ) => (await catalog.QueryAsync(new ListProductsQuery(page.Request), context, cancellationToken)).ToHttpResult()
        );
}
```

## From Result to response

- `ToHttpResult()` is the one place a Result becomes a response. On a `Result` without a value, success answers 204 No Content. On a `Result<T>`, success answers 200 OK with the value as the JSON body.
- `ToHttpResult(value => ...)` answers another success shape, such as 201 Created with a location: `.ToHttpResult(id => TypedResults.Created($"/catalog/products/{id}", new { id }))`. In a Module that declares API versions, generate the location with `LinkGenerator` instead, so it names the requested version ([openapi.md](openapi.md#what-a-caller-sees)).
- A failure answers a problem response, whatever overload the endpoint used.

## A Page over HTTP

- An endpoint of a query that answers a Page takes an `HttpPageRequest` parameter, as `ListProducts` does above, and passes its `Request` to the query. ASP.NET Core binds it from the query string: `page` (default 1), `size` (default 20), `sort`, `direction` (`asc` or `desc`) and any number of `filter[field][operator]=value`, the operator by its exact short name (`eq`, `ne`, `contains`, `startswith`, `gt`, `gte`, `lt`, `lte`). The `filter[` prefix is read ignoring case, as ASP.NET Core reads the other keys.
- The success body is the Page: `{"items":[…],"number":2,"size":20,"total":42,"totalPages":3,"hasNext":true,"hasPrevious":true}`, from `ToHttpResult()` of the query's `Page<T>`.
- A page or size that is not a whole number, another direction, an unknown operator, a malformed filter key or a `page`, `size`, `sort` or `direction` given twice, answers 400 as a problem response without `errors`, whose `detail` names the rule it broke (`The direction query parameter is asc or desc.`); a request ASP.NET Core itself refuses has no `detail`. Everything else reaches the query's validator, so a size above the declared maximum, an undeclared sort key or field, an operator the field does not support, a value that does not parse or a filter's field and operator given twice answers 400 with each validation Error's `code` and `target` (`Page.SortUnknown`, the value of `PageErrorCodes.SortUnknown`, and `Sort`).
- Declare and validate the query's paging as [commands-queries-validation.md](commands-queries-validation.md#paging) shows; never read paging parameters by hand.

## A repeatable write

The Proving Sample's `CreateProduct` endpoint, whose command opts in with `[Idempotent]` ([commands-queries-validation.md](commands-queries-validation.md#request-idempotency)):

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;
using PouryaKit.Cqrs;
using PouryaKit.Modularity;
using PouryaKit.Web;

internal sealed class CreateProductEndpoint : IEndpoint
{
    public void Map(IEndpointRouteBuilder routes) =>
        routes
            .MapPost(
                "/products",
                async (
                    CreateProductBody body,
                    OperationContext context,
                    IGatewayProcessor<CatalogModule> catalog,
                    CancellationToken cancellationToken
                ) =>
                    (
                        await catalog.SendAsync(
                            new CreateProductCommand(body.ProductId, body.CategoryId, body.Name),
                            context,
                            cancellationToken
                        )
                    ).ToHttpResult(id => TypedResults.Created($"/catalog/products/{id}", new { id }))
            )
            .RequireIdempotencyKey();
}
```

- `RequireIdempotencyKey()` on a Module's POST, PUT, PATCH or DELETE endpoint whose command opts in requires an `Idempotency-Key` header on every request. Its value becomes the request's `OperationContext.RequestId`, so a repeat by the same Actor and tenant answers what the first request answered, without running the command again. On an endpoint mapped for GET, HEAD, OPTIONS or TRACE it throws when the endpoint is built, which fails the Host's start: such an endpoint sends a query, and a query ignores the request id. So it does on an endpoint mapped for no method, with `Map(pattern, handler)` or an empty method list, which answers every method, GET included: map the command with `MapPost`, `MapPut`, `MapPatch` or `MapDelete`. Whether the command opts in with `[Idempotent]` is known only when it is dispatched, since the endpoint's delegate builds it: a request id on a command that does not opt in throws there (ADR 0002).
- The header is one value of 1 to 256 visible ASCII characters, with spaces only between them, the rule of the Kit's context headers, taken as sent: a quoted value keeps its quotes. A request without one, with two, or with a value the rule refuses answers 400 as a problem response without `errors`, whose `detail` states the rule, before its body is read. Otherwise the endpoint answers as any other, its validation failures and Errors included.
- A repeat answers the same status, location and body, because the replayed Result goes through the same `ToHttpResult` mapping: a second `POST /catalog/products` with one key answers 201 with the first location and `{"id":…}`, and creates and sends nothing. A first request that failed recorded nothing, so its repeat runs again.
- A client sends a new key with each new request and the same key only to repeat one.
- Only a Module's endpoints read the header, in the step that begins the request's Operation; an endpoint that does not require it ignores it. With `PouryaKit.Web.OpenApi`, the endpoint's OpenAPI operation lists the header as a required parameter ([openapi.md](openapi.md#documents-and-scalar)), from the endpoint's `IdempotencyKeyRequirement` metadata.

## Problem responses

- Every failure answers an RFC 9457 problem response, written as `application/problem+json` even when the caller's `Accept` asks for something else. It carries ASP.NET Core's `title` and `status`, its `type` where ASP.NET Core defines one for the status (it defines none for 429), and, for a failed Result, an `errors` array listing every Error in order with its `code`, its `message`, and its `target` when a validation Error names one.
- The status comes from the Errors' kinds: `Validation` 400, `NotFound` 404, `Conflict` 409, `Forbidden` 403, `Unexpected` 500. When the Errors are of several kinds, the most severe decides, in the order unexpected, forbidden, not found, conflict, validation: the caller cannot succeed by correcting its input while a more severe reason stands. The `errors` array still lists every Error.
- Validation failures from the pipeline answer 400 with each failure's code and target. The Unit of Work's Errors answer by their kinds as well: a concurrent change (`UnitOfWork.ConcurrencyConflict`) 409, and a unique index the Module mapped the status of its Error, 409 for `Product.NameInUse` ([persistence.md](persistence.md#unique-values)).
- An exception, from a handler, the database, authentication or an endpoint, answers 500 with no message, type or stack in the body. The Kit logs it at Error level.
- A request that cannot be read answers 400 as a problem response in every environment, with no `errors`, and is logged only at Debug level. One the Kit refuses, a page query string, an `Idempotency-Key` or a trusted peer's context headers, carries the Kit's reason as its `detail`, the rule the request broke. One ASP.NET Core refuses, such as malformed JSON, a body that names one member twice or a route value that does not parse, has no `detail`, because ASP.NET Core's message quotes what the caller sent.
- A failure that ASP.NET Core answers itself with an empty body, before any endpoint or exception handler runs, is written as the same problem response, with its status and no `errors`: routing's 405 for a method the route does not serve and 415 for a body of a content type the endpoint does not accept, the 401, 403 or 404 that authentication, authorization or routing answers, and the rate limiter's 429. `AddPouryaKitWeb` adds this through a startup filter, outermost in the pipeline but for response compression, which wraps it when enabled, so the Host adds nothing for it. A response that already has a body is left as it is.

## The caller and the Actor

- HTTP access is authenticated by default. `AddPouryaKitWeb` sets the authorization `FallbackPolicy` to require an authenticated user, once, while the Host is composed, so every endpoint that states nothing, a Module's or one the Host maps itself (a health check, a document), refuses an unauthenticated caller before it runs: 401, written as a problem response. A path no route matches has no endpoint, and the fallback policy applies to it too: an anonymous request answers 401, an authenticated one 404. A fallback policy the application set itself is kept. Never change `AuthorizationOptions` while the Host runs, such as from a policy provider that adds policies on demand.
- Every Module's route group also requires an authenticated caller explicitly, through the default policy.
- An endpoint the Host maps outside every Module, on a group that does not begin the Operation, opts out with the standard `.AllowAnonymous()`; it runs no Operation, so it names no Actor. On a root group with `BeginOperationPerRequest()` an endpoint runs an Operation, and is public only as a Module's endpoint is, below.
- Before the endpoint binds its parameters, the Kit begins the request's Operation with an `OperationContext` built from the caller: `Actor.Person(id)` from the Actor claim, the tenant from the tenant claim when one is configured, and a new chain of work (correlated with its own `OperationId`, no causation). Only a trusted peer Host continues an Operation of its own instead ([Calls between Kit Hosts](#calls-between-kit-hosts)); every other caller's context headers are ignored. The endpoint injects that context and passes it to the Gateway; it never builds one itself.
- `PouryaKitWebOptions` names the claims: `ActorClaim`, by default `ClaimTypes.NameIdentifier`, and `TenantClaim`, by default `PouryaKitClaimTypes.Tenant` (`PouryaKit.Cqrs`). The Kit's bearer scheme (`AddPouryaKitJwtBearer`, [security.md](security.md#jwt-bearer-authentication)) writes the Actor under `ClaimTypes.NameIdentifier` and the tenant under `PouryaKitClaimTypes.Tenant`, so a Host that uses it sets neither: the token's own claim names are configured once, in the bearer scheme. With a scheme of the application's own, name the claims it issues, with `AddPouryaKitWeb(...)` or `Configure<PouryaKitWebOptions>`; set `TenantClaim` to null to read no tenant at all. A blank `ActorClaim`, which would fail every authenticated request, or a `TenantClaim` blank rather than null, which would give every caller a null tenant unnoticed, stops the Host with an `OptionsValidationException` naming the option: when it starts, or earlier, in `MapPouryaKitModuleEndpoints` or `BeginOperationPerRequest`, which read both.
- A caller whose identity lacks the tenant claim, or carries it blank, runs with a null tenant and is not refused at the boundary; a tenant-scoped Module then refuses that caller's Operations with `Tenant.Required` ([multi-tenancy.md](multi-tenancy.md)). A tenant is never blank: `new OperationContext(actor, tenant)` throws for one, so null is the only way to say there is none. An authenticated caller without the Actor claim is a configuration bug: the request answers 500 and the log names the missing claim.
- A public endpoint states the Actor its anonymous callers run as with `.AllowAnonymousAs(Actor.System("storefront.visitor"))`, a system Actor of its own; an authenticated caller of it still runs as themselves. A plain `.AllowAnonymous()` on an endpoint whose group begins the Operation, a Module's or a root group with `BeginOperationPerRequest()`, fails the Host when it starts, because no Operation runs as an unnamed Actor. A tenant-scoped Module maps no public endpoint: an anonymous caller has no tenant, so its Operation would be refused ([multi-tenancy.md](multi-tenancy.md#limits)); Catalog's `ListProducts` is authenticated for that reason.
- A permission a command or query requires is declared on it with `[RequiresPermission(name)]` and checked against the application's `IActorPermissions` ([security.md](security.md)); a caller without it gets 403 with `Permission.Denied`. Any other rule for one command or query beyond "authenticated" is an `IAuthorizer` ([commands-queries-validation.md](commands-queries-validation.md#authorization-hook)), which answers 403 with its `Forbidden` Error. Never add a role or claim policy per endpoint for either: the check belongs to the Operation, however it arrives.

## Calls between Kit Hosts

A Host calls another over HTTP through a named or typed `HttpClient` it registers with the two Kit handlers, as the Host above registers `downstream`:

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Http.Resilience;
using PouryaKit.Web;

// In the Host's composition: limits on the standard pipeline's builder, then the context handler.
var stock = builder.Services.AddHttpClient<StockClient>(client => client.BaseAddress = new("https://inventory.example/"));
stock.AddPouryaKitResilience().Configure(options => options.Retry.MaxRetryAttempts = 2);
stock.AddPouryaKitOperationContext();
```

```csharp
using PouryaKit.Cqrs;
using PouryaKit.Web;

// A typed client is resolved in the calling Operation's scope, so it injects that Operation's context.
internal sealed class StockClient(HttpClient http, OperationContext context)
{
    public async Task<string> LevelAsync(Guid productId, CancellationToken cancellationToken)
    {
        using var request = new HttpRequestMessage(HttpMethod.Get, $"stock/{productId}").WithOperationContext(context);
        using var response = await http.SendAsync(request, cancellationToken);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync(cancellationToken);
    }
}
```

- `AddPouryaKitOperationContext()` sends the context a request was given with `request.WithOperationContext(context)` as the Kit's `pouryakit-*` headers, the names Messaging uses: the Actor and its kind, the tenant, the correlation, and the calling Operation's `OperationId` as the causation. A request without a context throws before it is sent. The handlers of an `HttpClient` are built in a scope of their own, never the Operation's, which is why the context travels on the request: give it the one the endpoint or handler injected, never one built for the call.
- `OperationContextHttpHeaders.Pairs(context)` returns those header names and values, checked as below: the pairs the handler writes, for a transport that carries the context in headers of its own and reuses Web's reader, such as gRPC metadata, so it needs no second codec. Messaging keeps its own extension, `WithOperationContext` on Wolverine's `DeliveryOptions`, which carries the values without this rule.
- An Actor id or tenant goes on the wire only when it is 1 to 256 visible ASCII characters with spaces only between them. Any other value, a CR or LF, another control character or a character outside ASCII, fails the call with an `InvalidOperationException` naming the header before anything is sent; a receiving Host answers 400 to a trusted peer's header that breaks the same rule. Keep identities a claim carries within it.
- The incoming request's `Authorization` header and bearer token are never forwarded. A peer that authenticates this Host gets the Host's own credentials (a client-credentials token or a client certificate), added by the application.
- `AddPouryaKitResilience()` is the standard .NET resilience handler (`AddStandardResilienceHandler`): attempt and total timeouts, a circuit breaker and retries of transient failures that stop when the call is cancelled. It needs no Aspire and no ServiceDefaults. Set limits on the builder it returns (`.Configure(o => o.Retry.MaxRetryAttempts = 2)` or `.Configure(configurationSection)`), and add the context handler after it, so every attempt carries the same context.
- By default (`HttpReplay.SafeMethods`) only GET, HEAD, OPTIONS and TRACE are retried: every other method, POST, PUT, PATCH, DELETE or an extension method such as PROPPATCH, is sent once, whether the attempt failed with a status or a broken connection, because a write that failed in transit may still have taken effect. Pass `AddPouryaKitResilience(HttpReplay.AllRequests)` only for a client whose downstream API makes every write replay-safe, such as with an idempotency key.
- One pipeline per client: ServiceDefaults adds a resilience handler to every client through `ConfigureHttpClientDefaults`. The Kit's service defaults recipe already calls `AddPouryaKitResilience()` there in place of Aspire's `AddStandardResilienceHandler()` ([host-recipes.md](host-recipes.md#one-resilience-pipeline)), so a client of a Host that uses it adds only `AddPouryaKitOperationContext()`; a client that needs other limits or `HttpReplay.AllRequests` calls `RemoveAllResilienceHandlers()` before its own `AddPouryaKitResilience(...)`. A client whose handler chain, once built, holds two resilience handlers, as when a client adds its own over the defaults', throws an `InvalidOperationException` naming `RemoveAllResilienceHandlers()` whenever it is created, so its calls never retry each other's attempts. A client that replaces the defaults' pipeline has a name: the client registered with an empty name would share the defaults' pipeline name, `-standard`, and its options and replay policy, so `AddPouryaKitResilience` refuses it with an `ArgumentException`. The last `AddPouryaKitResilience(...)` call for a client decides its replay policy: a client registered again, after `RemoveAllResilienceHandlers()` or not, keeps no earlier one.
- The receiving Host continues the Operation only for a caller it trusts as a peer. List the peers by the value of their authenticated identity's `ActorClaim` in `PouryaKitTrustedPeerOptions.Peers`, which the sample binds from `PouryaKit:TrustedPeers:Peers`; empty, the default, trusts no one. A blank peer stops the Host the same way as a blank claim: `AddPouryaKitWeb` validates the peers however the Host configures them. A trusted peer's request runs with the Actor, tenant and correlation its headers carry, caused by the peer's Operation, so a permission declared on the command is not checked again ([security.md](security.md)). A trusted peer without the headers runs as itself; one with incomplete or unreadable headers answers 400, with the header and the rule it broke as the problem's `detail`. Every other caller keeps its own identity and a new chain, whatever headers it sends.
- A trusted peer speaks for any Actor. Use a peer identity no person can hold, such as a client-credentials token's subject or a client certificate's thumbprint, as the Host's authentication scheme issues it; never a header, which anyone can send.

## CORS, compression and rate limits

The Host binds three options from configuration, as the Host above does, and `AddPouryaKitWeb` registers ASP.NET Core's own CORS, response compression and rate limiting with them; there is no package to add. A Host that configures none of them answers no cross-origin request, compresses nothing and limits no one. Their options are validated when the Host starts, so a misconfiguration fails `StartAsync` with an `OptionsValidationException`, never a request.

```json
{
  "PouryaKit": {
    "Web": {
      "Cors": {
        "Policies": {
          "browser": {
            "Origins": [ "https://app.example" ],
            "Methods": [ "GET", "POST" ],
            "Headers": [ "Content-Type", "Authorization" ],
            "ExposedHeaders": [ "Location" ],
            "AllowCredentials": false,
            "PreflightMaxAge": "00:10:00"
          }
        }
      },
      "Compression": { "Enabled": true },
      "RateLimit": { "Enabled": true, "PermitLimit": 100, "Window": "00:01:00", "QueueLimit": 10 }
    }
  }
}
```

- **CORS.** Each entry of `PouryaKitCorsOptions.Policies` is a named policy of ASP.NET Core's CORS services, allowing exactly the origins, methods, headers and exposed headers it lists, with credentials and a preflight cache time when set. Apply one to the pipeline with `app.UseCors(name)`, which also answers preflight requests. An endpoint's `RequireCors(name)` adds no middleware: it only names the policy the CORS middleware applies to that endpoint. A policy with no origins allows no origin, and a name nobody configured allows nothing: missing configuration never opens the Host. There is no `AllowAnyOrigin`: an origin is `scheme://host[:port]` as a browser sends it (`https://app.example`, no path or trailing slash), and a wildcard (`*`) anywhere, a malformed origin or a negative `PreflightMaxAge` fails the Host when it starts. A disallowed origin gets no `Access-Control-*` header at all; the browser then refuses the response.
- **Compression.** `PouryaKitCompressionOptions.Enabled` turns on ASP.NET Core's response compression, Brotli or gzip as the caller's `Accept-Encoding` asks, for its default content types (JSON included, problem responses not), over HTTPS as well as HTTP (`EnableForHttps`). Compressing a response that reflects what the caller sent beside a secret of another party, such as an anti-forgery token, can leak that secret through its length (BREACH); enable it for an API whose responses do not mix the two.
- **Rate limit.** `PouryaKitRateLimitOptions` is ASP.NET Core's global limiter, a fixed window per partition: `PermitLimit` requests in each `Window`, then up to `QueueLimit` more wait, oldest first, for the next window, and one beyond the queue is refused. `QueueLimit` is its own setting, 0 by default (refuse at once). The partition is the Operation's Actor as the HTTP boundary establishes it ([The caller and the Actor](#the-caller-and-the-actor)): the person an authenticated caller's `ActorClaim` names; for an unauthenticated caller, the Actor its endpoint names with `AllowAnonymousAs(actor)`, so a public endpoint's anonymous visitors share its Actor's partition and two public endpoints with different Actors have independent ones; and for a trusted peer's call, the Actor it continues, so it shares that Actor's partition with the Actor's own calls. The limiter resolves the caller's Operation Context once, and the endpoint begins its Operation with that same context. A request no Actor resolves for, such as an unauthenticated caller of an endpoint outside every Module that allows anonymous callers, shares one partition; the service defaults exempt their `/alive` from the limit, so a liveness probe is never refused there ([host-recipes.md](host-recipes.md#liveness-and-readiness)). No other header, `Host` and `X-Forwarded-For` included, chooses a partition: an untrusted caller's context headers are ignored. The limits are counted in each Host process. Enabled, it fails the Host when it starts unless `PermitLimit` is at least 1, `Window` is positive and `QueueLimit` is not negative.
- A refused request answers 429 as the same problem response as every other failure, `{"title":"Too Many Requests","status":429,...}` without `errors`: the limiter sets the status and writes nothing, and the Kit's status code pages write the body ([Problem responses](#problem-responses)).
- Test them on a Host composed with the configuration the test needs (`builder.Configuration.AddInMemoryCollection(...)` before `Build`, as `WebDefaultsTests` does): a preflight is an `OPTIONS` request with `Origin` and `Access-Control-Request-Method`; send to an `https://` base address for HTTPS, and read `Content-Encoding` on the raw response, which the test client does not decompress.

## Testing over HTTP

The test project references `Microsoft.AspNetCore.TestHost`. Its fixture builds the Host with the same `CreateBuilder` and `Build` as `Program`, serves it in memory with `builder.WebHost.UseTestServer()`, and makes a test authentication scheme the default ([testing.md](testing.md)). The Proving Sample's scheme reads the caller from the `Authorization` header and issues the claim the Kit reads:

```csharp
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

// "Authorization: Test <id>" is the authenticated person <id>; no header is an anonymous caller.
internal sealed class TestAuthentication(
    IOptionsMonitor<AuthenticationSchemeOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder
) : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    public const string SchemeName = "Test";

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var header = Request.Headers.Authorization.ToString();
        if (!header.StartsWith($"{SchemeName} ", StringComparison.Ordinal)) return Task.FromResult(AuthenticateResult.NoResult());

        var identity = new ClaimsIdentity([new Claim(ClaimTypes.NameIdentifier, header[(SchemeName.Length + 1)..])], SchemeName);
        return Task.FromResult(AuthenticateResult.Success(new AuthenticationTicket(new ClaimsPrincipal(identity), SchemeName)));
    }
}
```

A test sends real requests through the whole pipeline and asserts the status, the problem response and the rows:

```csharp
[Fact]
public async Task Creating_a_product_with_a_name_in_use_answers_409_with_its_error_and_writes_nothing()
{
    using var client = Fixture.Host.GetTestClient();
    client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue(TestAuthentication.SchemeName, "ada");
    client.DefaultRequestHeaders.Add("api-version", "2.0");   // Catalog's paths name their API version (openapi.md)
    var category = Guid.CreateVersion7();
    await client.PostAsJsonAsync("/catalog/categories", new { categoryId = category, name = "Lamps" }, TestContext.Current.CancellationToken);
    await client.PostAsJsonAsync(
        "/catalog/products",
        new { productId = Guid.CreateVersion7(), categoryId = category, name = "Desk lamp" },
        TestContext.Current.CancellationToken
    );

    var response = await client.PostAsJsonAsync(
        "/catalog/products",
        new { productId = Guid.CreateVersion7(), categoryId = category, name = "Desk lamp" },
        TestContext.Current.CancellationToken
    );

    Assert.Equal(HttpStatusCode.Conflict, response.StatusCode);
    Assert.Equal("application/problem+json", response.Content.Headers.ContentType?.MediaType);
    using var problem = JsonDocument.Parse(await response.Content.ReadAsStringAsync(TestContext.Current.CancellationToken));
    Assert.Equal(
        """[{"code":"Product.NameInUse","message":"Another product already has this name."}]""",
        problem.RootElement.GetProperty("errors").GetRawText()
    );
}
```

- `Fixture.Host.GetTestClient()` (`Microsoft.AspNetCore.TestHost`) returns an `HttpClient` over the in-memory server; a client without the header is an anonymous caller.
- A second `AddAuthentication(scheme)` call after the Host's makes the test scheme the default. The Host's own scheme is still registered and still checked when the Host starts, so give it its configuration in tests too: the Proving Sample's `AppFixture` passes the bearer section of its `TestIssuer` ([security.md](security.md#testing-jwt-bearer-authentication)). Issue the claim the Host's `ActorClaim` names, the tenant under its `TenantClaim` (`PouryaKitClaimTypes.Tenant` by default), and the permission claims the Host's `IActorPermissions` reads: the Proving Sample's scheme turns a `Test-Permissions` header into them ([security.md](security.md#testing-permissions)).
- Prove the authenticated default on an endpoint outside every Module's route group, which has no authorization of its own: map one on a Host the test builds, and an anonymous request to it must answer 401.
- A test that needs another composition, such as another environment or an endpoint the shared Host must not map, builds and starts a Host of its own from the same composition methods.
- Test outgoing calls against a WireMock.Net server (test-only, Apache-2.0): its scenarios fail requests on cue (503, 503, then 200) and its request journal (`server.LogEntries`) shows the headers every attempt arrived with. Prove the receiving side, trusted and untrusted callers, on the real Host with headers named by `PouryaKit.Messaging.OperationContextHeaders`; a stub's configured response proves nothing about the receiver.
