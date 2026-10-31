# Localization

`PouryaKit.Localization`: a Module writes its messages in English and Persian, and each HTTP request runs in the culture its client's `Accept-Language` header names, English by default. Namespace `PouryaKit.Localization`. It references `PouryaKit.Cqrs` and the ASP.NET Core shared framework only, so it adds no NuGet package. It is optional: a Host that answers in English alone does not reference it, as neither of the Proving Sample's Hosts does.

- [Registering](#registering)
- [A Module writing localized messages](#a-module-writing-localized-messages)
- [What the Kit guarantees](#what-the-kit-guarantees)
- [Limits](#limits)
- [Testing](#testing)

## Registering

In the Composition Root of each Module that writes localized messages:

```csharp
using PouryaKit.Localization;

module.AddPouryaKitLocalization();
```

- On a Module's builder it names no assembly: it records the assembly of the Module's definition, where its Features live, for the Host, as every Kit registration that names Feature assemblies does ([modules.md](modules.md)). An application without Modules calls it on its own services, naming its Feature assembly: `builder.Services.AddPouryaKitLocalization(typeof(Program).Assembly)`.
- It registers the shared framework's localization (`IStringLocalizer<T>`), English and Persian as the supported cultures with English the default, and the request localization middleware, which a startup filter puts ahead of every middleware the application adds. The Host calls nothing in its pipeline. A second call, from another Module or the Host, adds no second culture or middleware.

## A Module writing localized messages

A Feature injects `IStringLocalizer<T>` for a type of its own and reads a message by its key. The messages sit in `.resx` files beside that type's file, named after it: the neutral file holds English, the `.fa.resx` file Persian. The SDK compiles them into the Module's assembly and a `fa` satellite assembly; there is no designer or generated class.

```text
Greetings/
  GreetingMessages.cs       internal sealed class GreetingMessages;
  GreetingMessages.resx     Greeting = Welcome
  GreetingMessages.fa.resx  Greeting = خوش آمدید
```

```csharp
using Microsoft.Extensions.Localization;
using PouryaKit.Cqrs;
using PouryaKit.Domain;

internal sealed class GreetingMessages;

internal sealed record GreetQuery : Query<string>;

internal sealed class GreetQueryHandler(IStringLocalizer<GreetingMessages> messages) : QueryHandler<GreetQuery, string>
{
    public override Task<Result<string>> Handle(GreetQuery query, CancellationToken cancellationToken) =>
        Task.FromResult<Result<string>>(messages["Greeting"].Value);
}
```

- Keep the marker type first in its file: the SDK names the compiled resources after the first class in the `.cs` file the `.resx` sits beside, and the localizer looks them up by `T`'s full name.
- A key with no Persian entry falls back to English, and a key with no entry at all returns the key itself (`ResourceNotFound` is true).
- An Error a handler returns may carry a localized message the same way; its code stays the same in every culture.

## What the Kit guarantees

- A request whose `Accept-Language` header prefers Persian, `fa` or a regional variant such as `fa-IR`, over the other cultures the Host supports, by quality and then by listed order, runs in Persian (`en;q=0.1, fa;q=1` does), and its Feature's localizer reads the Persian message. A request with no header, `en` or a culture the Host does not support runs in English, and so does one whose only Persian value has quality 0 (`fa;q=0`) or comes after 16 other values: the provider reads the first 16 values, not the framework's default of three. The header is the only source: no query string or cookie changes the culture.
- The culture flows through the Gateway into the Operation's handlers, which run in the request's culture.
- A cached query ([caching.md](caching.md)) is stored per UI culture, so each culture's callers read their own answer.
- The Kit's own Error messages stay English in every culture.

## Limits

- English and Persian only, fixed; there are no options to bind.
- A delivery (a Notification Handler, an Integration Event Handler, a delayed or recurring command) runs outside an HTTP request, in the process's culture, so it does not read the publishing request's culture.
- FluentValidation's built-in messages follow the request's culture too, and FluentValidation ships Persian ones; a message a Module's validator sets itself is its own to localize.

## Testing

Send the request with the header and assert the message, as the Proving Sample's tests do against its Stub Module:

```csharp
using var client = app.Client("caller");
client.DefaultRequestHeaders.Add("Accept-Language", "fa");

var message = await client.GetFromJsonAsync<string>("/stub/greeting", TestContext.Current.CancellationToken);

Assert.Equal("خوش آمدید", message);
```
