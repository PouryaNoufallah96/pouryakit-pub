# Caching

`PouryaKit.Caching`: a query opts in to having its successes cached, per Actor and tenant, over `HybridCache`, and a command declares the cache tags it invalidates, evicted once its Operation commits. Namespace `PouryaKit.Caching`. It references `PouryaKit.Cqrs` and `Microsoft.Extensions.Caching.Hybrid` only. Redis, or any other distributed cache, is the application's own `IDistributedCache` registration; the Kit never references one.

- [Registering](#registering)
- [A cacheable query](#a-cacheable-query)
- [A command that invalidates](#a-command-that-invalidates)
- [What the Kit guarantees](#what-the-kit-guarantees)
- [Limits](#limits)
- [Testing](#testing)

## Registering

On the Host:

```csharp
using PouryaKit.Caching;

builder.Services.AddPouryaKitCaching();
// Optional: share entries between the application's Hosts. Microsoft.Extensions.Caching.StackExchangeRedis.
if (builder.Configuration.GetConnectionString("cache") is { } redis)
{
    builder.Services.AddStackExchangeRedisCache(options => options.Configuration = redis);
}
```

- `AddPouryaKitCaching` fills the dispatcher's query cache slot (ADR 0002) and registers `HybridCache` unless one is registered. A second call adds nothing. It registers no pipeline step an application could add to, and an application never implements `IQueryCache` itself: a structure test rejects one ([testing.md](testing.md#structure-tests)).
- A Host whose registered queries include a cacheable one calls it, or fails when it is built, before anything runs, naming the call (`The Host registers no query cache, so every cacheable query would run uncached: … Call AddPouryaKitCaching() on the Host's services.`): a cacheable query never runs uncached without a word (owner decision of 2026-10-07, ADR 0002). It is a composition check of `AddPouryaKitCompositionChecks`, which `AddPouryaKitModules` calls ([modules.md](modules.md#the-hosts-checks)). `HybridCache` needs no distributed cache, so the call is the whole fix. A Host with no cacheable query runs every query without one.
- Without a distributed cache, entries live in each Host's memory. With one, `HybridCache` writes each entry there too, and another Host reads it.

## A cacheable query

A Module references `PouryaKit.Caching` for the contracts and declares the key, the duration and the tags on the query, as the Proving Sample's `GetProduct` does:

```csharp
using PouryaKit.Caching;
using PouryaKit.Cqrs;

internal sealed record GetProductQuery(Guid ProductId) : Query<ProductDto>, ICacheableQuery
{
    public static string ProductTag(Guid productId) => $"product:{productId}";

    public const string ProductsTag = "products";

    public string CacheKey => ProductId.ToString();

    public TimeSpan CacheDuration => TimeSpan.FromMinutes(5);

    public IReadOnlyCollection<string> CacheTags => [ProductTag(ProductId), ProductsTag];
}
```

- `CacheKey` names only what the query asks. The Kit adds the query's type, as its full name and its assembly's simple name, the Operation's tenant (an Operation without one as `OperationContext.NoTenant`), its Actor, by kind and id, and the current UI culture's name, so an entry answers only the query, the Actor, the tenant and the culture that stored it: a localized answer ([localization.md](localization.md)) is kept per culture, and a Host without localization runs in one. Renaming the query's type, or the assembly that holds it, starts new entries.
- `CacheTags` name the resources the answer reads. A tag spans Actors and tenants: one eviction reaches every Actor's entry.
- The response is stored as JSON in the Kit's representation (`UsePouryaKitJson`), so it must round-trip through it; every answer, the first included, is read back from that JSON.
- The handler is written as for any query, and runs inside its Operation as usual: a `Continue()` it calls is a trusted continuation.
- Only an Operation's outermost query uses the cache. Sent from another query's handler, an authorizer, a validator, a command's handler or a Domain Event Handler, the same query runs and answers as any query, but is neither read from nor stored in the cache; so is a query in an Operation that has already failed. Send a query you want cached from the Gateway or the endpoint, not from inside another step.
- Once started, the handler runs to completion: other callers of the same query may be waiting on it, so it receives no cancellation, and every caller, the one that started it included, waits for it. A caller's cancellation counts only until the run starts or is joined. Keep a cacheable query's handler short.

## A command that invalidates

Every command that changes a cached resource declares its tag:

```csharp
internal sealed record RenameProductCommand(Guid ProductId, string Name) : Command, IInvalidatesCache
{
    public IReadOnlyCollection<string> InvalidatedCacheTags => [GetProductQuery.ProductTag(ProductId)];
}

// Removes by name, so it cannot name the products' ids before it runs.
internal sealed record RemoveProductsCommand(IReadOnlyList<string> Names) : Command<int>, IInvalidatesCache
{
    public IReadOnlyCollection<string> InvalidatedCacheTags => [GetProductQuery.ProductsTag];
}
```

- The tags are read from the command alone, after its checks and before its handler runs. A command that finds what it changes by a criterion declares a tag every entry it may change carries, as `RemoveProducts` declares `products`.
- Feature code never evicts. The dispatcher collects the tags of the outermost command and of every command nested in its Unit of Work, a Domain Event Handler's included, and evicts them once the Unit of Work has committed.

## What the Kit guarantees

- **One `HybridCache.GetOrCreateAsync` per cacheable query.** Concurrent misses of one key in one Host share one handler run, `HybridCache`'s own single flight.
- **Never a failed Result.** A failure is returned to every caller that shared the run and stored nowhere, in memory or in the distributed cache: the next query runs again.
- **Nothing a query changed.** The cache serializes a success, and only then runs the Operation's read-only check, before it stores the answer: a handler that changes tracked state, or an answer whose serialization does, such as a deferred enumerable, fails the check. Every caller that shared the run receives the `InvalidOperationException`, and nothing is stored.
- **After the checks.** The authorization hook and validation run before the cache is read, so a stored answer never skips them.
- **Only the outermost query of an Operation that has not failed.** A query nested in a command, a check or another query may read a change they made before it started, which its own read-only check takes as its starting point, so it neither reads nor stores an entry. That covers a command's handler, a Domain Event Handler, any command's or query's authorizers and validators, the outermost command's included, and another query's handler. A query in an Operation that has already failed, such as one sent after a refused command in the same scope, runs uncached too, since the Operation may still track what was rolled back.
- **Eviction after the outermost commit only.** A rolled-back Operation evicts nothing, so a valid entry survives a failed command. A command that rolls back after a nested query leaves no entry from it.
- **A read in flight across a commit stores nothing.** A run whose handler was still running when a commit in this Host evicted one of its tags returns its answer to every caller that shared it and stores none, so the next read runs the handler again. A command that declares `HybridCache`'s wildcard tag `*`, which evicts every entry, reaches every run in flight; a run on other tags only is stored as usual. `HybridCache` alone would keep that answer, read before the commit, until it expires: it stamps an entry again when one of its tags was evicted while the handler ran.
- **A failed eviction cannot fail a committed Operation.** When the distributed cache refuses it, the Kit logs `EvictionFailed` at Error level and returns the command's success; this Host's own copy is still evicted.

## Limits

- Single flight is per Host. Two Hosts that miss the same key both run the handler.
- An eviction is certain only in the Host whose Operation committed. Another Host may return its own in-memory copy until it expires; choose `CacheDuration` with that in mind.
- On another Host, a query that read before a command committed may store the answer it read, which that Host's `HybridCache` keeps until it expires: only the committing Host's own runs in flight see its evictions. In the committing Host, an eviction that lands in the microseconds between the Kit's check, made once the handler's read has ended, and `HybridCache`'s own store is not seen either.
- A caller that joins a run in flight after a commit that evicted one of the run's tags receives that run's answer, read before the commit, for that one call; the run stores nothing.
- Nested reads are never cached: a cacheable query an authorizer sends, such as a permission lookup, runs its handler every time.
- The Kit caches query answers only. An Integration Event Handler or anything else that changes a cached resource without a command declares nothing; such data needs a short duration or no caching.

## Testing

`CachingTests` in the Proving Sample shows the patterns: change a row with plain SQL behind the cache's back, so a stored answer (the old value) differs from a fresh read (the new one); count a handler's runs with a probe; and for Redis, start a `Testcontainers.Redis` container, compose two Hosts with its connection string, and wait until the first Host's entry is in Redis before the second reads it, since `HybridCache` answers before its distributed write completes.
