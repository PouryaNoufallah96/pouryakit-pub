# File storage

`PouryaKit.Storage`: one contract, `IFileStorage`, that uploads, downloads and deletes a file under a key and grants a time-limited link to read it, every call cancellable and returning a `Result` for what a Feature expects, so Feature code never names a cloud SDK. Namespace `PouryaKit.Storage`; it references `PouryaKit.Domain`, and `PouryaKit.Cqrs` for the one-provider composition check, and no cloud SDK. A provider package supplies the Host's one implementation: `PouryaKit.Storage.S3` (namespace `PouryaKit.Storage.S3`) over `AWSSDK.S3`, for AWS S3 and S3-compatible stores such as MinIO, or `PouryaKit.Storage.AzureBlob` (namespace `PouryaKit.Storage.AzureBlob`) over `Azure.Storage.Blobs`, for Azure Blob Storage and Azurite. Feature code is the same on either.

- [Registering](#registering)
- [Store, then record](#store-then-record)
- [The calls](#the-calls)
- [What the Kit guarantees](#what-the-kit-guarantees)
- [Limits](#limits)
- [Testing](#testing)

## Registering

On the Host, bind the provider's options from configuration, then register it.

### S3

```csharp
using PouryaKit.Storage.S3;

builder.Services.Configure<PouryaKitS3StorageOptions>(builder.Configuration.GetSection("PouryaKit:Storage:S3"));
builder.Services.AddPouryaKitS3Storage();
```

```json
{
  "PouryaKit": {
    "Storage": {
      "S3": { "Bucket": "acme-files", "Region": "eu-central-1" }
    }
  }
}
```

- `Bucket` and `Region` are required. On AWS, leave `ServiceUrl` unset and credentials come from the AWS SDK's default chain (environment, profile, the instance's or task's role). For an S3-compatible store, set `ServiceUrl` (`http://minio:9000`), `ForcePathStyle` to `true` for MinIO, and `AccessKey` with `SecretKey`; `Region` is then the region requests are signed for, such as `us-east-1`.
- The bucket must exist: the Kit creates none.
- The options are validated when the Host starts, which fails naming each problem: no bucket or region, a service URL that is not an absolute HTTP or HTTPS address, or an access key without its secret key or the reverse. The client is built from them when `IFileStorage` is first resolved.

### Azure Blob Storage

```csharp
using PouryaKit.Storage.AzureBlob;

builder.Services.Configure<PouryaKitAzureBlobStorageOptions>(builder.Configuration.GetSection("PouryaKit:Storage:AzureBlob"));
builder.Services.AddPouryaKitAzureBlobStorage();
```

```json
{
  "PouryaKit": {
    "Storage": {
      "AzureBlob": { "Container": "acme-files", "ConnectionString": "DefaultEndpointsProtocol=https;AccountName=acme;AccountKey=...;EndpointSuffix=core.windows.net" }
    }
  }
}
```

- `Container` is required, and must exist: the Kit creates none.
- Reach the account with `ConnectionString`, which must carry the account key, or with `ServiceUrl` (`https://acme.blob.core.windows.net`) and `AccountName` with `AccountKey`; set one form, never both. The account key signs every presigned link, so a connection string with only a SAS token, or Microsoft Entra credentials, are not options: the package takes no `Azure.Identity`.
- The options are validated when the Host starts, which fails naming each problem: no container, both or neither of the two forms, a connection string that does not parse or holds no account key, a service URL that is not an absolute HTTP or HTTPS address, an account name or key without the service URL or the reverse, or an account key that is not base64, which the `StorageSharedKeyCredential` the client signs with decodes. The client is built from them when `IFileStorage` is first resolved.

### One provider

- A Host composes exactly one provider. Each provider registers through `AddPouryaKitFileStorage(factory)` of `PouryaKit.Storage`, which adds a composition check, so a Host that registers another `IFileStorage`, before or after the call, a second call of its own included, fails when its container is built, naming each registration's type, or the factory that builds it; so does a Host that removes every `IFileStorage` after the call, naming the contract. Registering both `AddPouryaKitS3Storage` and `AddPouryaKitAzureBlobStorage`, in either order, fails the build naming both factories. A Host's own provider registers through `AddPouryaKitFileStorage` too, to take the same check. A Host whose code needs `IFileStorage` and that registers none fails when its container is built: a Kit Host validates its container then, and `IFileStorage` has no registration to resolve.

## Store, then record

No file store joins the Unit of Work's commit, so a file is never transactional. Store the file first, under a key the Feature chooses, and only then record the key in the command's own state:

```csharp
internal sealed class AttachContractCommandHandler(IFileStorage files, IRepository<Customer, CustomerId> customers)
    : CommandHandler<AttachContractCommand>
{
    public override async Task<Result> Handle(AttachContractCommand command, CancellationToken cancellationToken)
    {
        var customer = await customers.FindAsync(new CustomerId(command.CustomerId), cancellationToken);
        if (customer is null)
        {
            return CustomerErrors.NotFound;
        }

        var key = $"contracts/{command.CustomerId}/{Guid.CreateVersion7()}.pdf";
        var stored = await files.UploadAsync(key, command.Content, "application/pdf", cancellationToken);
        return stored.IsFailure ? stored : customer.AttachContract(key);
    }
}
```

- A recorded key always names a stored file: the command records it only after the upload returned.
- A command that fails or rolls back after the upload leaves its object behind, unnamed by any record. The application's retention removes such objects, for example a recurring command that deletes the keys of attempts it can derive from its own records, or a lifecycle rule of the store on a prefix the application stages files under. `IFileStorage` has no listing.
- A fresh key per upload keeps a retried or concurrent command from replacing a file another record names; an upload under an existing key replaces that file.
- Deleting a recorded file follows the same order in reverse: remove the record in the command, then delete the object once the Operation committed, for example from a Notification Handler, so a rolled-back command never loses a file its record still names.

## The calls

- `UploadAsync(key, content, contentType, cancellationToken)` stores the stream from its current position to its end. The caller keeps and disposes the stream.
- `DownloadAsync(key, cancellationToken)` returns a `StoredFile` with its `Content` stream and `ContentType`, or `StorageErrors.FileNotFound`, an `ErrorKind.NotFound` Error, when no file is stored under the key. Dispose the `StoredFile`: its stream is read from the store as the caller reads it.
- `DeleteAsync(key, cancellationToken)` succeeds whether or not a file was stored under the key, so retention can run again.
- `GetPresignedUrlAsync(key, lifetime, cancellationToken)` returns a link that reads the file with a plain HTTP GET, without credentials, until `lifetime` has passed, and at most the lifetime of the credentials that signed it: a link signed with temporary credentials, such as an instance or task role's from the SDK's default chain, stops working when they expire, whatever lifetime was asked for (AWS documents it). The store is not asked whether the file exists: a link to a missing file answers not found when followed. A token cancelled before the call fails it; one cancelled while the SDK resolves credentials stops the wait, though the SDK may still sign. A lifetime of zero or less, or longer than the provider signs a link for, fails with `StorageErrors.LinkLifetimeOutOfRange`, an `ErrorKind.Validation` Error, and signs nothing: S3 signs a link for at most seven days, and Azure Blob too: Azure sets no limit on a link signed with the account key, and the Kit bounds it at the seven days of a user delegation SAS, so a lifetime is valid on either provider. An Azure link is a service SAS that reads the one blob.
- A blank key or content type is a bug and throws `ArgumentException`. A failure of the store itself (a refused credential, a missing bucket, an unreachable endpoint) throws the provider's exception: it is infrastructure, not a business outcome.

## What the Kit guarantees

- **The contract and the provider's behaviour.** Upload, download, delete and the presigned link behave as above against a real S3-compatible store and against Azurite; a missing file is the not-found Error, never an exception.
- **One provider, validated before serving.** A Host that registers a second provider, in either order, fails when its container is built, naming both, and invalid options stop the Host when it starts.
- **Not a transactional file.** The Kit never stores a file in the Unit of Work's commit, and never removes an object a failed command left behind: that is store-then-record and the application's retention.

## Limits

- No listing, metadata or existence call: a Feature records what it stores.
- The key is the application's: the Kit neither scopes it by tenant nor checks it beyond refusing a blank one. A tenant-scoped Module puts the tenant in its keys, or keeps them in its tenant-scoped records only.
- A presigned link grants anyone who holds it read access until it expires; it cannot be revoked short of deleting the file.

## Testing

`S3StorageTests` in `tests/PouryaKit.Storage.Tests` runs the provider against RustFS, an S3-compatible store with no Testcontainers module, through `Testcontainers`' generic `ContainerBuilder("rustfs/rustfs:1.0.2")`: set its credentials with `RUSTFS_ACCESS_KEY` and `RUSTFS_SECRET_KEY`, wait for an HTTP 200 from `/health/ready` on port 9000, create the bucket with those credentials, then compose a Host with `ServiceUrl` the mapped `http://host:port`, `ForcePathStyle` true and `Region` `us-east-1`. A presigned link to a local store is HTTP, since the service URL is; fetch it with a plain `HttpClient` to prove it reads the file without credentials.

`AzureBlobStorageTests` runs the Azure provider against Azurite through `Testcontainers.Azurite`'s `AzuriteBuilder("mcr.microsoft.com/azure-storage/azurite:3.37.0")` with `WithCommand("--skipApiVersionCheck")`: `Azure.Storage.Blobs` 12.30.1 sends the service version 2026-10-06, which no Azurite release accepts yet. Create the container with a `BlobServiceClient` over the container's `GetConnectionString()`, and bind that string as `ConnectionString`; the `ServiceUrl` form takes `GetBlobEndpoint()` with the emulator's account `devstoreaccount1` and its published key.
