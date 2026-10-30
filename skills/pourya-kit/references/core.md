# Core helpers

`PouryaKit.Core`: three helpers and nothing else. One namespace, `PouryaKit.Core`; no dependencies. Cqrs, Web, Messaging.Wolverine, Persistence.Dapper, Persistence.EFCore and Security reference it, so a Module that references any of them already has it.

- [Predicate composition](#predicate-composition)
- [Implementing-type lookup](#implementing-type-lookup)
- [Synchronization-context scope](#synchronization-context-scope)
- [What Core never holds](#what-core-never-holds)

## Predicate composition

`AndAlso` and `OrElse` are C# 14 extension members of `Expression<Func<T, bool>>`. They combine two predicates with `Expression.AndAlso` and `Expression.OrElse`, so the right predicate runs only when the left one has not decided the result, and they put both bodies on one new parameter, so a LINQ provider such as EF Core sees one lambda with no invocation in it, and a lambda nested in either predicate keeps the bindings it declares:

```csharp
using System.Linq.Expressions;
using PouryaKit.Core;

internal sealed record ListedProduct(string Name, decimal Price, bool IsListed);

internal static class ListedProductFilters
{
    public static Expression<Func<ListedProduct, bool>> For(string? namePrefix, decimal? maxPrice)
    {
        Expression<Func<ListedProduct, bool>> filter = product => product.IsListed;
        if (namePrefix is not null)
        {
            filter = filter.AndAlso(product => product.Name.StartsWith(namePrefix));
        }

        if (maxPrice is not null)
        {
            filter = filter.AndAlso(product => product.Price <= maxPrice);
        }

        return filter;
    }
}
```

Pass the result to a specification's `Query.Where` or any `IQueryable`. Both predicates must be lambdas of one parameter; the composed one has a new parameter named as the left predicate's. A guard in the left predicate protects the right one, as `&&` does in C#: `product => product.Name != null` followed by `.AndAlso(product => product.Name.Length > 3)` never reads the length of a null name.

## Implementing-type lookup

`ImplementingTypes.Of(contract, assemblies)` returns the concrete types of the given assemblies that a contract describes, each once, ordered by full name. It is the only reflection Core offers, and the Kit's own registrations find their handlers, authorizers, Notification Handlers, typed IDs and endpoints through it too; use it for the application's own contracts:

```csharp
using Microsoft.Extensions.DependencyInjection;
using PouryaKit.Core;

internal interface IReportExporter
{
    string Format { get; }
}

internal sealed class CsvReportExporter : IReportExporter
{
    public string Format => "csv";
}

internal static class ReportExporterRegistration
{
    public static IServiceCollection AddReportExporters(this IServiceCollection services)
    {
        foreach (var exporter in ImplementingTypes.Of(typeof(IReportExporter), typeof(IReportExporter).Assembly))
        {
            services.AddScoped(typeof(IReportExporter), exporter);
        }

        return services;
    }
}
```

The input contract:

- **A non-generic or closed contract**, an interface or a class such as `IReportExporter` or `IHandler<int>`, matches every concrete type that can be assigned to it.
- **An open generic definition**, an interface or a class such as `typeof(IHandler<>)` or `typeof(Store<>)`, matches every concrete type that implements, or derives from, a closed form of it. Read each match's closed form from its interfaces or base types.
- **Only concrete classes come back**: no interface, abstract or static class, struct or open generic class. Internal and nested classes are included, so a Module's internal implementations are found.
- **A type that fails to load throws** `ReflectionTypeLoadException` rather than being skipped. A contract that is a generic parameter, or a generic type neither closed nor a definition, throws `ArgumentException`.

`ImplementingTypes.Candidates(contract, assemblies)` reads the contract the same way and returns every class it describes, abstract classes, open generic definitions and the classes the compiler generated included; interfaces and structs stay out, and the order is the same. Use it when a registration must reject a declaration `Of` would skip silently, as the Kit's own registrations reject a generic handler or a permission on an abstract type:

```csharp
using PouryaKit.Core;

internal static class ReportExporterChecks
{
    public static void ThrowIfAnyIsUnusable()
    {
        var unusable = ImplementingTypes
            .Candidates(typeof(IReportExporter), typeof(IReportExporter).Assembly)
            .Where(type => type.IsAbstract || type.IsGenericTypeDefinition)
            .Select(type => $"{type.FullName} is an exporter that cannot be registered.")
            .ToList();
        if (unusable.Count > 0)
        {
            throw new InvalidOperationException(string.Join(Environment.NewLine, unusable));
        }
    }
}
```

## Synchronization-context scope

`SynchronizationContextScope.Enter(context)` sets `SynchronizationContext.Current` to `context`, or to none when it is left out, and disposing the scope restores the context it found, after a normal exit or an exception. Its use is blocking on asynchronous code from a synchronous member that may run on a thread with a context, where the awaited continuations would otherwise wait for the blocked thread:

```csharp
using PouryaKit.Core;

internal sealed class LegacyReportAdapter(Func<CancellationToken, Task<string>> renderAsync)
{
    public string Render()
    {
        using var scope = SynchronizationContextScope.Enter();
        return renderAsync(CancellationToken.None).GetAwaiter().GetResult();
    }
}
```

The context belongs to the thread, so the scope is a `ref struct`: the compiler refuses one that lives across an `await`, which could resume on another thread. Enter and dispose it on one thread, once, in a `using` statement or declaration. Asynchronous code all the way down needs no scope at all.

## What Core never holds

Core's public types are exactly `PredicateBuilder`, `ImplementingTypes` and `SynchronizationContextScope`. `CoreTests` in `tests/PouryaKit.ArchitectureTests` fails when a fourth public type appears, so a new helper is an owner decision, never an addition. A helper that belongs to one Capability lives in that Capability's package, and a helper that belongs to the application lives in the application.
