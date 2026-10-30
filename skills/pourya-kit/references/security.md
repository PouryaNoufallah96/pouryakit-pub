# Security

`PouryaKit.Security`: a permission declared on a command or query, checked through the pipeline's existing authorization hook against the application's source of an Actor's permissions, and never checked again for a trusted continuation; and optional PBKDF2 password hashing for an application that stores passwords. Namespace `PouryaKit.Security`. It references `PouryaKit.Cqrs` and `PouryaKit.Core` only, no ASP.NET Core, so a Module or a Host without HTTP uses it too. Authenticated HTTP access by default is `PouryaKit.Web`'s ([web.md](web.md#the-caller-and-the-actor)).

`PouryaKit.Security.JwtBearer`, optional: the Host's standard ASP.NET Core JWT bearer authentication, with issuer, audience, signature and lifetime always validated, and each verified token's configured claims normalized into the Actor and tenant. Namespace `PouryaKit.Security.JwtBearer`. It references `PouryaKit.Security` and `Microsoft.AspNetCore.Authentication.JwtBearer`, not `PouryaKit.Web`.

- [Declaring a permission](#declaring-a-permission)
- [The Actor's permissions](#the-actors-permissions)
- [Denial](#denial)
- [Continuations](#continuations)
- [Testing permissions](#testing-permissions)
- [Password hashing](#password-hashing)
- [JWT bearer authentication](#jwt-bearer-authentication)

The Kit has no role system, policy framework, identity store or admin bypass, and issues no token: no token endpoint, provider SDK, API key or PKCE flow. A rule that is not "this Actor holds this permission", such as one that reads the command's data, is an `IAuthorizer<T>` of the Module's own ([commands-queries-validation.md](commands-queries-validation.md#authorization-hook)).

## Declaring a permission

The Proving Sample's `RemoveProducts` command, in `Products/Features/RemoveProducts/`:

```csharp
using PouryaKit.Cqrs;
using PouryaKit.Security;

/// <summary>Removes every product whose name, as stored, is one of these; returns how many it removed.</summary>
[RequiresPermission(RemoveProductsCommand.Permission)]
internal sealed record RemoveProductsCommand(IReadOnlyList<string> Names) : Command<int>
{
    public const string Permission = "catalog.products.remove";
}

// In the Composition Root, beside AddPouryaKitCqrs and AddPouryaKitValidation:
// module.AddPouryaKitPermissions();
```

- Put `[RequiresPermission(name)]` on the command or query record, one permission each. Name it `<module>.<resource>.<action>` and keep the name in a constant on the record, so tests and grants name it without repeating the string.
- `module.AddPouryaKitPermissions()` in the Module's Composition Root registers the check for every command and query that declares one and is in the assembly of the Module's definition, internal ones included, or is handled by a handler there wherever it is declared, such as in the Module's Contracts, and nothing else; it scans the assemblies it names instead, as an application without Modules does. Name the same assemblies `AddPouryaKitCqrs` scans: an attributed operation of an assembly only Cqrs scanned has no check, and `DeclaredAuthorizersCheck` fails the Host when it is built, naming it. Call it in every Composition Root whose Features declare one.
- A declared permission whose check no Composition Root registered fails the Host when it is built, so it never runs unchecked, not even through a Gateway of a Host built and never started. `Build()` throws `InvalidOperationException`, one line per command or query:

  ```text
  Catalog.Products.Features.RemoveProducts.RemoveProductsCommand declares RequiresPermissionAttribute, but its check is not registered, so it would run unchecked. Call AddPouryaKitPermissions with the assembly Catalog in the Composition Root that registers its handler.
  ```

  Fix it by adding `module.AddPouryaKitPermissions()` to that Composition Root. Any Composition Root may register it, so a check registered from another Module passes, and a Host whose Modules declare no permission needs no Security registration at all. Only the permission check counts: an `IAuthorizer<T>` of the Module's own for the same command does not stand in for it. It is one of Cqrs's composition checks, run on the Host's final registrations when its container is built: `AddPouryaKitModules` installs them, and a Host without Modularity calls `builder.AddPouryaKitCompositionChecks()`, as the monolith template's Host does ([templates.md](templates.md#what-each-composition-composes)). A later `ConfigureContainer` skips them, and the Host then stops when it starts, naming the call ([modules.md](modules.md#the-hosts-checks)).
- The attribute on a type that is not one concrete command or query fails the Host while it is composed: the dispatcher would never check it.
- The check runs in the pipeline's authorization step, before validation and the Unit of Work, with any other authorizer of the same command or query.

## The Actor's permissions

The application says which permissions an Actor holds by implementing `IActorPermissions` and registering it; the Kit reads no claim and no request itself. The Proving Sample's Host reads them from the HTTP caller's claims:

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Options;
using PouryaKit.Cqrs;
using PouryaKit.Security;
using PouryaKit.Web;

// An HTTP caller holds the "permission" claims its authentication scheme issued; any other Actor, a named system Actor
// included, holds none.
public sealed class CallerPermissions(IHttpContextAccessor http, IOptions<PouryaKitWebOptions> web) : IActorPermissions
{
    public const string ClaimType = "permission";

    public Task<bool> HasPermissionAsync(OperationContext context, string permission, CancellationToken cancellationToken)
    {
        ArgumentNullException.ThrowIfNull(context);
        var caller = http.HttpContext?.User;
        return Task.FromResult(
            caller?.Identity?.IsAuthenticated == true
                && context.Actor.Kind == ActorKind.Person
                && caller.FindFirstValue(web.Value.ActorClaim) == context.Actor.Id
                && caller.HasClaim(ClaimType, permission)
        );
    }
}

// In the Host:
// builder.Services.AddHttpContextAccessor();
// builder.Services.AddScoped<IActorPermissions, CallerPermissions>();
```

- `HasPermissionAsync(context, permission, ct)` receives the Operation's `OperationContext`, so the answer can depend on the Actor and the tenant. It is asked for every Operation but a trusted continuation.
- Match the claims to the context's Actor, as `CallerPermissions` does: code inside a request may run an Operation as another Actor, which the caller's claims say nothing about.
- Independent system work, such as a hosted job running as `Actor.System("catalog.reconciler")`, holds only the permissions the application's source grants that named Actor. Nothing holds a permission by default.
- A Host whose Modules declare permissions and that registers no `IActorPermissions` fails when it is built: `AddPouryaKitModules` validates the container.

## Denial

- An Actor without the permission gets `PermissionErrors.Denied` (`Permission.Denied`, kind `Forbidden`); `ToHttpResult()` answers it with 403 and the Error in the problem response's `errors` ([web.md](web.md#problem-responses)).
- A denied command commits nothing. Denied as a nested command, it marks the Operation rollback-only like any failed nested command, so the outer command commits nothing either.

## Continuations

- A trusted continuation is an Operation whose `OperationContext.IsTrustedContinuation` is true (ADR 0007). That is a Notification Handler's or Integration Event Handler's delivery, whose context the Kit restored from the headers its Outbox wrote when the causing Operation committed. It is also an Operation that a running, authorized Operation continued with `context.Continue()`, such as a synchronous question through another Module's Contracts, and the request of a trusted peer Host that continues an Operation, whose context Web restores from the peer's context headers ([web.md](web.md#calls-between-kit-hosts)). The permission check lets it run without asking `IActorPermissions` again, so durable work never depends on what its Actor may do by the time it runs.
- Only the Kit sets the mark. A context your code creates is never trusted, whatever `causationId` you give it. `Continue()` is trusted only when it is called on the context of the Operation it runs in, after that Operation was authorized. Outside it, on a context that never ran or whose Operation has ended, and inside an authorizer or a validator, `Continue()` only records the causation. `CausationId` only records the chain.
- A handler in an authorized Operation can therefore continue into protected work that its Actor holds no permission for. Declare a permission on the Operation that decides to cause such work, and review what its handler continues into.
- Only the permission check stands aside. Every other `IAuthorizer<T>` registered for the continued command or query still runs, with the original Actor, and can refuse it; recognise a trusted continuation there by `context.IsTrustedContinuation`, never by `CausationId`. So does the tenant requirement of a tenant-scoped Module: a continuation without a tenant is refused there ([multi-tenancy.md](multi-tenancy.md#declaring-a-tenant-scoped-module)).
- An HTTP request starts a new chain of work, with no causation, so a caller cannot present its request as a continuation. The one exception is a caller the Host authenticates as one of its configured trusted peer Hosts, whose request continues the Operation its context headers name ([web.md](web.md#calls-between-kit-hosts)); a header never identifies a peer.

## Testing permissions

- Make the test authentication scheme able to issue permission claims. The Proving Sample's `TestAuthentication` turns a `Test-Permissions` header, comma-separated, into `permission` claims, and `AppFixture.Client("ada", permissions: RemoveProductsCommand.Permission)` sets it.
- Gateway tests run as `GatewayTestExtensions.TestActor`, independent work with no HTTP caller. Grant it, in the test Host only, the permissions those tests need, by name, as a named system Actor of the Host would be granted them; delegate every other Actor to the Host's own source. The Proving Sample's `TestActorPermissions` does this and is registered after the Host's composition. Never grant every permission or every Actor.
- Prove a declared permission over HTTP both ways, the effect committed for a caller holding it and a 403 with nothing committed for one without it, and read the rows back from the database.
- Prove a continuation through a real delivery: an Actor who holds no permission raises a Domain Event whose Notification Handler sends the protected command, and the effect commits; the same Actor sending the command directly is refused.
- Prove the other side at the Gateway: the same Actor sending the protected command or query with a context built with a causation, or with `Continue()` on a context that is not running, is refused with `PermissionErrors.Denied`.

## Password hashing

For an application that stores its own passwords. The Kit only hashes and verifies: it stores nothing and has no user, account, identity store or token issuance ([JWT bearer authentication](#jwt-bearer-authentication)).

```csharp
using PouryaKit.Security;

// In the Host, or the Composition Root of the Module that owns the credentials; independent of permissions:
// builder.Services.AddPouryaKitPasswordHashing();
// builder.Services.AddPouryaKitPasswordHashing(options => options.Iterations = 700_000);
// Or from configuration, as any options class:
// builder.Services.Configure<PouryaKitPasswordHashingOptions>(builder.Configuration.GetSection("PasswordHashing"));

internal sealed class Account
{
    public string? PasswordHash { get; private set; }

    public void SetPassword(string password, IPasswordHasher hasher) => PasswordHash = hasher.Hash(password);

    // True when the password is right; a hash weaker than the configured parameters is replaced on the way.
    public bool SignIn(string password, IPasswordHasher hasher)
    {
        switch (hasher.Verify(password, PasswordHash))
        {
            case PasswordVerification.SucceededNeedsUpgrade:
                PasswordHash = hasher.Hash(password);
                return true;
            case PasswordVerification.Succeeded:
                return true;
            default:
                return false;
        }
    }
}
```

- `hasher.Hash(password)` returns the string to store; `hasher.Verify(password, stored)` answers `Succeeded`, `SucceededNeedsUpgrade` or `Failed`. On `SucceededNeedsUpgrade`, store `Hash(password)` in place of the old hash while the plain password is at hand, in the same command as the sign-in's other changes, which its Unit of Work commits. Stored hashes are never invalidated by a configuration change.
- `Verify` never throws: a missing password, a missing or malformed stored hash, an unsupported version or algorithm, and a password with no UTF-8 form are all `Failed`. Each `Hash` and `Verify` is CPU work by design, about 0.1 s at the defaults on current hardware; rate-limit sign-in at the HTTP boundary.
- The empty password hashes like any other; what a password must look like is the application's rule, enforced before `Hash`.
- `PouryaKitPasswordHashingOptions` follow the options pattern: the `configure` of `AddPouryaKitPasswordHashing` and every `Configure<PouryaKitPasswordHashingOptions>`, a configuration section's included, set the same options in order. A second `AddPouryaKitPasswordHashing` adds to what the first set and never restores the defaults; it replaces a hasher registered before it.
- Options outside their bounds fail the Host's start with an `OptionsValidationException` naming each one, so the Host never serves with a weak configuration. The registration itself does not check them.

| Option | Default | Allowed |
| - | - | - |
| `Algorithm` | `Sha256`, PBKDF2-HMAC-SHA256 | `Sha256` or `Sha512` |
| `Iterations` | 600,000 | at least 600,000 for `Sha256`, 220,000 for `Sha512`; at most 10,000,000 |
| `SaltSize` | 16 bytes, from `RandomNumberGenerator`, fresh for every hash | 16 to 64 |
| `KeySize` | 32 bytes | 32 to the HMAC's output: 32 for `Sha256`, 64 for `Sha512` |

### The stored format

`pbkdf2$1$<algorithm>$<iterations>$<base64 salt>$<base64 key>`, for example `pbkdf2$1$sha256$600000$<24 characters>$<44 characters>`:

- `pbkdf2` and the format version `1`; another prefix or version is `Failed`.
- The algorithm, `sha256` or `sha512`, lower case. SHA-1 and SHA-384 are refused.
- The iterations in ASCII decimal digits only, at most eight of them, with no sign, space, separator, NUL, other script's digits or leading zero, from 1 to 10,000,000. A hash with fewer iterations than options now allow, from an older configuration, still verifies and needs an upgrade.
- Salt and key in standard padded base64 (RFC 4648), canonical only: text that decodes but does not encode back the same, such as one with white space or no padding, is `Failed`. A salt of 16 to 64 bytes; a key of 32 bytes to the algorithm's output. The lower key bound is what stops a stored key cut short from verifying, since PBKDF2's shorter output is a prefix of its longer one.

`Verify` derives a key of the stored key's length with the stored algorithm, iterations and salt, using `Rfc2898DeriveBytes.Pbkdf2` (the password as UTF-8), and compares it with `CryptographicOperations.FixedTimeEquals`. The hash needs an upgrade when it used another algorithm than the configured one, or fewer iterations, or a shorter salt or key; a hash stronger than the configuration is left as it is.

### Why these parameters

Chosen on 2026-10-07 from one check of two sources:

- OWASP Password Storage Cheat Sheet, `OWASP/CheatSheetSeries@3c07d4a cheatsheets/Password_Storage_Cheat_Sheet.md` (2026-10-04), PBKDF2 section: PBKDF2-HMAC-SHA256 at 600,000 iterations is the recommended PBKDF2 setting, HMAC-SHA-512 needs 220,000, and SHA-1 is legacy only. It names Argon2id first, but the platform has no Argon2id and the Kit takes no third-party cryptography; PBKDF2 is the sheet's choice when FIPS-140 validated implementations matter. It also recommends upgrading the work factor by re-hashing at the next sign-in, and storing the algorithm and work factor with the hash.
- .NET 10 API reference, `Rfc2898DeriveBytes.Pbkdf2` (learn.microsoft.com, page updated 2026-07-01): the one-shot static methods support SHA-1, SHA-256, SHA-384 and SHA-512 and convert a `string` or `char` password to UTF-8, throwing `EncoderFallbackException` when it has no UTF-8 form.

Raise `Iterations` when the guidance does; existing hashes keep verifying and are re-hashed as their owners sign in.

### Testing password hashing

- Test through `IPasswordHasher` from `AddPouryaKitPasswordHashing`: the Kit's own tests resolve it from a provider built over the service collection, and start a Host to see options outside their bounds refused.
- The minimum iterations apply in tests too; at the defaults the Kit's 60 password test cases run in about 4 s.

## JWT bearer authentication

For a Host whose callers present bearer tokens from an identity provider. The Kit verifies and normalizes them; it issues none. JWT token creation is an excluded recommendation: it was recommended and the owner has not accepted it, and the Kit's catalogue of Capabilities refuses token issuance, so a Host that must issue tokens does it with its identity provider, outside the Kit.

```csharp
using PouryaKit.Security.JwtBearer;
using PouryaKit.Web;

// In the Host, instead of AddAuthentication(...).AddJwtBearer(...):
builder.Services.AddPouryaKitJwtBearer(claims => claims.TenantClaim = "tenant");   // "sub" is the Actor by default
builder.Services.AddAuthorization();
builder.Services.AddPouryaKitWeb();   // reads the tenant under PouryaKitClaimTypes.Tenant by default
```

```json
{
  "Authentication": {
    "Schemes": {
      "Bearer": {
        "ValidIssuer": "https://login.example.com/",
        "ValidAudiences": [ "catalog-api" ],
        "Authority": "https://login.example.com/"
      }
    }
  }
}
```

- `AddPouryaKitJwtBearer` registers standard JWT bearer authentication as the default scheme, `Bearer`. The requirement that an authority's metadata is read over HTTPS keeps ASP.NET Core's default (on), so a production Host reads its `Authority` over HTTPS. A Development Host that reads its metadata from an HTTP issuer sets `Authentication:Schemes:Bearer:RequireHttpsMetadata` to `false` in its own environment's configuration only; the Kit does not refuse that setting, which is for local development only. A deployed Host reads its metadata over HTTPS: whatever sits on the path of a metadata request made over HTTP can replace the issuer's signing keys, and forged tokens then validate. The framework's default already protects a Host that never sets it. Its issuer, audiences and signing keys are the framework's own `JwtBearerOptions`, bound from `Authentication:Schemes:Bearer`: an `Authority` (or `MetadataAddress`) whose discovery document supplies the issuer and keys, or `ValidIssuer` and `SigningKeys`, as `dotnet user-jwts` writes them for development. Set anything else on the same options with `Configure<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme, ...)`; never call `AddJwtBearer` again.
- The Host fails when it starts, with an `OptionsValidationException` naming every problem, unless the scheme requires the audience (`RequireAudience`) and validates it against a configured audience, validates the issuer against a configured issuer or authority, the signature against a signing key or authority, and the token's lifetime and expiration time. Nothing in the Kit's options turns one off, and setting `ValidateAudience = false` or `RequireAudience = false`, leaving out every audience, or doing the same to the issuer fails the start, never the first request. A custom validator delegate that runs instead of one of these checks is refused by the Kit's scheme: setting `AudienceValidator`, `IssuerValidator`, `IssuerValidatorUsingConfiguration`, `LifetimeValidator`, `SignatureValidator` or `SignatureValidatorUsingConfiguration` fails the start with a failure naming it. The check reads the options as they end, after the configuration and every `Configure` and `PostConfigure`.
- The token keeps its own claim names (`MapInboundClaims` stays false). For each verified token the Kit writes the value of `PouryaKitJwtBearerOptions.ActorClaim`, `"sub"` by default, as the identity's `ClaimTypes.NameIdentifier`, which `PouryaKit.Web` reads as `Actor.Person(id)`, and the value of `TenantClaim` as `PouryaKitClaimTypes.Tenant` (`PouryaKit.Cqrs`), the claim `PouryaKit.Web` reads the Operation's tenant from by default, so the Host names the token's tenant claim once, here. A claim the token itself carries under either of those two types is dropped first. Every other claim, such as the `permission` claims the Proving Sample's `CallerPermissions` reads, keeps its name.
- A token without the tenant claim runs with a null tenant and is not refused, as with any scheme ([web.md](web.md#the-caller-and-the-actor)). A Module whose every Operation needs a tenant declares itself tenant-scoped, and then refuses such a caller's Operations with `Tenant.Required` ([multi-tenancy.md](multi-tenancy.md)); an application in which every caller must have a tenant, whatever it calls, enforces it in its own `OnTokenValidated`.
- A token whose signature, issuer, audience or lifetime fails, or that carries no Actor claim, or more than one Actor or tenant value, is refused: the request is anonymous, and an endpoint that requires authentication answers 401 as a problem response, with the scheme's `WWW-Authenticate: Bearer` header.
- An `OnTokenValidated` of the application's own still runs, after the normalization and only for a token the Kit kept: set it, and any other event delegate, on the existing `Events` with `Configure<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme, ...)`. The options check at start verifies that the normalization is the scheme's `OnTokenValidated`, so replacing `Events` in a `PostConfigure`, assigning a `JwtBearerEvents` subclass, or setting `EventsType` fails the start: each would let the normalization be skipped.

### Testing JWT bearer authentication

- Test with real tokens and the real middleware. The Proving Sample's `TestIssuer` signs tokens with an HMAC key generated for the test run and gives every Host the tests compose its `Authentication:Schemes:Bearer` section on the command line (`ValidIssuer`, `ValidAudiences`, `SigningKeys`), so the shared Host's bearer scheme passes its options check at start even while `TestAuthentication` is the default. `Microsoft.IdentityModel.JsonWebTokens`' `JsonWebTokenHandler` creates the tokens; it arrives with the bearer package.
- A test of the bearer scheme builds its own Host with `AppFixture.ComposeHost(..., configure: builder => builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme))`, which makes `Bearer` the default again, and starts it.
- Prove each refusal with a token that differs in one thing: another signing key, issuer or audience, a lifetime that ended beyond the handler's five minutes of clock skew, no `sub`, no `aud`. Prove the options check with one weakened option per case on the composed Host, each validator delegate and each way of replacing the events included, and that `StartAsync` throws.
- A second started Host shares the message store's queues. Wait for the fixture's deliveries (`Fixture.WaitUntilDeliveredAsync`) before starting it and before it is disposed, or it can take a delivery and die holding it, and the next test's reset times out.
