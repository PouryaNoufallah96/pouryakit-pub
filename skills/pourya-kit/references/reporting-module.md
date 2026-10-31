# Report Module

`PouryaKit.Reporting.Module`: the first Module the Kit ships as a package. A Host adds it once; it stores report jobs in its own schema, generates each in the background through the renderer ([reporting.md](reporting.md)) into the Host's one file storage provider ([storage.md](storage.md)), serves each job's state and download link to its requester, and removes expired jobs and their files. The report kinds and their layouts are the Host's, through two ports it implements. Namespace `PouryaKit.Reporting.Module`, one for the whole package; every type is internal but the Composition Root `ReportModule`, the two ports and the `ReportRequest` they receive. It references Modularity, Persistence.Postgres, Persistence.Dapper, Messaging.Wolverine.EFCore, Validation, Security, Web, Reporting and Storage, and no storage provider.

- [Composing it](#composing-it)
- [The Host's ports](#the-hosts-ports)
- [The endpoints](#the-endpoints)
- [A job's life](#a-jobs-life)
- [What the Kit guarantees](#what-the-kit-guarantees)
- [Limits](#limits)
- [Testing](#testing)

## Composing it

The Proving Sample's Host, `ModularMonolithHost.CreateBuilder`:

```csharp
using PouryaKit.Reporting.Module;
using PouryaKit.Storage.S3;

builder.AddPouryaKitModules(modules =>
{
    modules.Add<CatalogModule>();
    modules.Add<ReportModule>();
});
builder.Services.Configure<PouryaKitS3StorageOptions>(builder.Configuration.GetSection("PouryaKit:Storage:S3"));
builder.Services.AddPouryaKitS3Storage();
builder.Services.AddSingleton<IReportRequestAuthorizer, SalesReports>();
builder.Services.AddSingleton<IReportLayouts, SalesReports>();
builder.UsePouryaKitWolverine(connectionName: "app");
```

- **Its database.** It stores over the connection string `app` (`ReportModule.ConnectionName`), in the schema `reports`, and reads with Dapper over the same name. A Composition Root receives no configuration, so the name is the Module's: give the Host's database that name, and keep the message store on the same database, since its delayed and recurring commands commit through its Outbox and the message store check fails the build otherwise ([messaging.md](messaging.md#the-hosts-checks)). Its migrations are in the package: `HostFixture` applies them in tests, and a deployment applies them as it applies every Module's ([persistence.md](persistence.md#migrations)). Its first migration creates its `pouryakit_receipts` table too.
- **What the Host registers.** An `IReportRequestAuthorizer`, an `IReportLayouts`, one file storage provider and an `IActorPermissions` ([security.md](security.md#the-actors-permissions)). The Host's container is validated when it is built, so a Host that registers no service of one of them fails then, naming the service. The Module registers the renderer itself, `TryAddSingleton<IReportRenderer, ReportRenderer>()`.
- **Its settings.** The Module reads them from the Host's configuration section `PouryaKit:Reports`, as a Module reads its own settings ([modules.md](modules.md#the-composition-root)), into an options type of its own, validated when the Host starts, which fails unless each is longer than zero, `Retention`, `CleanupInterval` and `ClaimTimeout` at most 36500 days, and `LinkLifetime` at most seven days:

| Key | Default | What it sets |
| - | - | - |
| `Retention` | 7 days | How long after its request a job and its file are kept. |
| `CleanupInterval` | 1 hour | The cadence of the retention, the Module's recurring command `reports.retention`. |
| `ClaimTimeout` | 10 minutes | How long a generation attempt holds its claim before the one takeover may take the job; set it longer than the slowest report takes to lay out, render and store. |
| `LinkLifetime` | 15 minutes | How long a download link works, at most seven days, the longest link S3 and Azure Blob sign. A link never outlives its job: one signed less than `LinkLifetime` before the job's expiry works only until that expiry. |

## The Host's ports

The Proving Sample's `SalesReports`, one class registered for both ports, shortened:

```csharp
using System.Globalization;
using PouryaKit.Cqrs;
using PouryaKit.Domain;
using PouryaKit.Reporting;
using PouryaKit.Reporting.Module;

public sealed class SalesReports : IReportRequestAuthorizer, IReportLayouts
{
    public const string Kind = "sales";

    public Task<bool> MayRequestAsync(OperationContext context, ReportRequest request, CancellationToken cancellationToken) =>
        Task.FromResult(context.Actor.Kind == ActorKind.Person && context.Tenant is not null && request.Kind == Kind && MonthOf(request) is not null);

    public Task<Result<ReportLayout>> LayOutAsync(OperationContext context, ReportRequest request, CancellationToken cancellationToken) =>
        Task.FromResult<Result<ReportLayout>>(
            new ReportLayout("Monthly sales", new ReportTable([new("Product"), new("Amount", ReportAlignment.Right)], [["Tea", "30.00"]]))
        );

    private static DateOnly? MonthOf(ReportRequest request) =>
        request.Parameters.Count == 1
        && request.Parameters.TryGetValue("month", out var month)
        && DateOnly.TryParseExact(month, "yyyy-MM", CultureInfo.InvariantCulture, DateTimeStyles.None, out var parsed)
            ? parsed
            : null;
}
```

- `ReportRequest(Kind, Format, Parameters)`: the kind the Host defines, the `ReportFormat` to render to, and the kind's parameters by name, as text the kind parses.
- `IReportRequestAuthorizer.MayRequestAsync(context, request, ct)` decides whether the Operation's Actor and tenant may request this kind with these parameters. It runs in the request's authorization step, before validation, so it receives the request as the caller gave it; false refuses it with `Reports.RequestDenied` (403), and nothing is stored or scheduled.
- `IReportLayouts.LayOutAsync(context, request, ct)` lays the kind out ([reporting.md](reporting.md#a-modules-layout)). It runs in the background, under a continuation of the request, with the requester's Actor, tenant and correlation, so a layout that asks another Module continues the context with `Continue()` ([modules.md](modules.md#asking-another-module-a-question)). It may run more than once for one job: for each generation attempt, and again when an attempt is retried. A failed Result fails the job with its first Error's code.

## The endpoints

Under the route group `/reports`, authenticated as every Module's endpoints are ([web.md](web.md#endpoints)):

- `POST /reports` with `{ "kind": "sales", "format": "Csv", "parameters": { "month": "2026-10" } }` answers `202 Accepted` with `{ "id": … }` and `Location: /reports/{id}`. `format` is required, `Pdf`, `Csv` or `Spreadsheet`: a body without one, or with another name, fails to bind and answers 400 before the request reaches the Host's port. Once the port allowed it, the request is validated: a kind of 1 to 100 characters, a defined format, at most 20 parameters with names up to 100 and values up to 1000 characters, each failure a 400.
- `GET /reports/{jobId}` answers `{ id, kind, format, state, downloadUrl, failure, expiresAt }`: `state` is `Requested`, `Generating`, `Completed`, `Failed` or `Expired`; `downloadUrl` is a presigned link to the file once the job is `Completed` (`IFileStorage.GetPresignedUrlAsync` for `LinkLifetime`, or until `expiresAt` when that comes sooner); `failure` is the first Error's code once it is `Failed`. A job read at or past its `expiresAt` answers `Expired`, with no `downloadUrl`, even before the retention stores that state.
- Only the requester reads a job, within the job's tenant, and so does an Actor whose `IActorPermissions` grants `ReportModule.ReadTenantJobsPermission` (`reports.jobs.read-tenant`), for every job of its own tenant and of no other: a tenant-wide read is a permission the Host grants. Anyone else, another Actor of the tenant or the requester under another tenant, gets `Reports.JobNotFound` (404), as for a job that does not exist, and no link is signed for them.
- There is no endpoint that lists jobs.

## A job's life

1. **Requested.** `POST /reports` stores the job, with its requester, its tenant and an expiry of the request time plus `Retention`, and schedules its first claim, a delayed command due at once ([messaging.md](messaging.md#delayed-commands)), in the same commit.
2. **Claimed.** The claim, a continuation of the request, gives the job attempt 1, commits, and schedules the attempt's generation, due at once, and the one takeover, due when the claim's `ClaimTimeout` has passed. The claim commits on its own, before anything renders, so the takeover sees it.
3. **Generated.** The attempt, if it still holds the claim, asks the Host's `IReportLayouts`, renders the layout, stores the file under its own key, `reports/{job id}/{attempt}`, and settles: `Completed`, publishing that key, or `Failed` when the layout or the renderer refused. A failure of the store is retried, as a delivery's failure is. An attempt that finds, when it starts, that it no longer holds the claim deletes its own key, unless the job publishes it: an attempt taken over, expired or removed while it rendered stored its file there, its settlement was refused, and its retry removes that file.
4. **Taken over, once.** When the takeover comes due and the first attempt still holds its claim, older than `ClaimTimeout`, it claims the job for attempt 2, which generates as above. A takeover schedules no further takeover; when the first attempt settled first, the takeover changes nothing.
5. **Expired, then removed.** The retention runs every `CleanupInterval` as the system Actor `reports.retention` (`ReportModule.RetentionActor`), across every tenant. It expires each job past its expiry, except one whose attempt holds a claim younger than `ClaimTimeout`; an expired job serves nothing. A later occurrence deletes every file the job's attempts may have stored, then the job, which then answers not found. Each step takes up to 100 jobs per occurrence, oldest expiry first.

## What the Kit guarantees

- **A refused request leaves nothing.** The Host's port decides before any job is stored; a refusal stores no job and schedules no claim.
- **Fenced claims and settlements.** Every claim, settlement and expiry is saved under the job's version, so of two that read one version the second to commit is refused, rolls back and, retried, finds the job changed. A stale claim is taken over once. An attempt taken over while it renders stores its file under its own key, which the job never serves, and its settlement is refused; the job serves the winning attempt's file, and the late attempt's retry deletes its own.
- **Read by its requester alone.** A job is read within its tenant, by its requester or an Actor granted the tenant's jobs; a link is signed only after both checks passed.
- **A link never outlives its job.** A link is signed for `LinkLifetime` or until the job's expiry, whichever comes sooner, and a job read at or past its expiry gets none, so the retention never deletes a file a link still promises.
- **Retention never deletes a served file.** It removes the jobs past their expiry and their files in every tenant, and keeps a job not yet expired with its file. It deletes a job's files only in an occurrence after the one that committed the job's expiry, so a settlement that commits first refuses the expiry by the job's version and keeps its file.
- **A Module like any other.** The structure tests judge it with the sample's Modules: no save or commit in its code, no transport type, no public type but the ones above, one namespace ([testing.md](testing.md#structure-tests)).

## Limits

- One connection string name, `app`, which the Host cannot change: `IModuleDefinition` takes no parameters.
- A setting out of range fails the start with its one failure reported twice in the `AggregateException` the start throws: the settings are validated themselves, and again when the retention's cadence reads `CleanupInterval` from them. The failure names the bounds.
- The file is rendered in memory, as one byte array, inside the attempt's Unit of Work, whose transaction stays open while the layout, the renderer and the store run: keep a kind to what fits in memory and within `ClaimTimeout`.
- A job taken over whose second attempt also stalls stays `Generating` until its expiry; the retention then removes it.
- A late attempt's file goes when its delivery is retried; a delivery whose every retry fails to delete it is set aside as a dead letter of `reports.render`, and the file stays until that dead letter is replayed.
- A download link grants anyone who holds it read access until it expires ([storage.md](storage.md#limits)).
- No report-list endpoint, no cancelling, no per-tenant retention and no notice when a report is ready: a client polls the job endpoint.

## Testing

The Kit's tests run the Module over HTTP in the Proving Sample's Host, on a database of its own, with its files in a RustFS bucket (`ReportFixture`, a class fixture beside `AppFixture`). The fixture replaces the Host's ports with `TestReports`, which hands the sample's kind to `SalesReports` and holds the first layout of its `held` kind until released, and sets a `ClaimTimeout` of 2 seconds and a retention every second. A test follows a job by polling `GET /reports/{jobId}`, reads its file through the link with a plain `HttpClient`, and reads the claims the Module stored from the message store's envelopes. `RecordingFileStorage` stands in front of the S3 provider, as the Host's one registration, and records every link it signs with its lifetime. `ReportFixture.HoldDeliveriesAsync` holds every delivery of the Module, the retention's included, by locking its Receipt table in a transaction of the test's, since a delivery records its Receipt before its command runs; a read records none, so a job past its expiry is read before the retention stores it. `ReportFixture.Compose` composes a Host as the fixture's, with settings of the test's, for the start refusals.
