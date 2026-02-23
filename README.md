# Corp.Api.DocMan.Lib

NuGet service library for the **Corp.Api.DocMan** document management API. This package provides strongly-typed Refit HTTP clients that allow consuming applications to interact with the DocMan API without writing any HTTP plumbing code. It covers file management, folder management, audit trail recording, and health-check operations.

### Why This Library Exists

Rather than having every consuming application hand-craft HTTP calls, parse JSON responses, and manage authentication against the DocMan API, this library centralizes that logic into a single, versioned NuGet package. When the API evolves, only the library needs to be updated — consuming applications simply bump their package reference. The library uses [Refit](https://github.com/reactiveui/refit) under the hood to generate the HTTP client implementations at compile time, which means you get compile-time safety on your API contracts with zero boilerplate.

### Design Decisions

- **Three-interface pattern per service**: Each service area (e.g., File) has three types: a public `IFileService` that consumers inject, a public `IFileRefitService` that defines the raw Refit HTTP contract, and a `FileService` implementation that wraps the Refit client with structured logging and error handling. This separation means consumers can mock `IFileService` in unit tests without needing an HTTP server.
- **`IApiResponse<T>` return types everywhere**: Rather than returning raw `T` values and throwing exceptions on non-success HTTP responses, every method returns Refit's `IApiResponse<T>`. This gives callers access to the full HTTP response (status code, headers, error body) without requiring try/catch. The library catches `ApiException` internally only for structured logging purposes — it always rethrows so the caller maintains full control.
- **Mutual-TLS authentication**: All HTTP traffic is secured with client certificate authentication. The certificate path and an AES-encrypted password are resolved from configuration at startup.
- **Entity classes bundled in the package**: The `Corp.Api.DocMan.Obj` project is packed into this NuGet as a private asset, so consumers automatically get the entity types (`File`, `Folder`, etc.) without needing a separate package reference.

## Prerequisites

Before you can use this library, ensure the following are in place:

- **.NET 10 or later** — The library targets `net10.0`. Your consuming application must use a compatible target framework.
- **A deployed instance of the Corp.Api.DocMan API** — The library is a client; it requires a running API server to call. Coordinate with your team to get the base URL for your target environment.
- **A valid client certificate (.pfx) and its AES-encrypted password** — The library authenticates with the API using mutual TLS. You'll need the certificate file deployed to a path accessible by your application and the password encrypted using `Corp.Lib.Cryptography.Aes.Encrypt()`.

## Installation

Install from your internal NuGet feed:

```shell
dotnet add package Corp.Api.DocMan.Lib --version 10.1.1
```

## Configuration

The library uses a two-layer configuration approach. First, it reads identifiers from your `appsettings.json` to determine which environment it's targeting. Then, it uses those identifiers to compose environment-variable keys that hold the actual connection details. This design allows the same application binary to run against different environments simply by changing environment variables — no code or config file changes required.

### Step 1: appsettings.json

Add the following two keys to your application's `appsettings.json`. These tell the library which Voyager instance and environment to use when composing the environment-variable key names:

```json
{
  "TargetedVoyagerInstance": "<instance>",
  "TargetedVoyagerEnvironment": "<environment>"
}
```

| Key | Example Value | Purpose |
|---|---|---|
| `TargetedVoyagerInstance` | `Voyager1` | Identifies the Voyager platform instance |
| `TargetedVoyagerEnvironment` | `Production` | Identifies the target environment (Dev, QA, Production, etc.) |

> **Important**: These values are read directly from the `appsettings.json` file on disk (not from `IConfiguration`). Make sure they are present in the physical file, not only in environment overrides.

### Step 2: Environment Variables

Using the two values from Step 1, the library composes configuration keys in the format `{instance}.{environment}.Corp.Api.DocMan.{setting}` and resolves them from `IConfiguration`. In practice, these are typically set as environment variables on the host machine or in your CI/CD pipeline.

| Key | Description |
|---|---|
| `{instance}.{environment}.Corp.Api.DocMan.Url` | Base URL of the DocMan API (e.g., `https://docman-api.internal.corp.com`) |
| `{instance}.{environment}.Corp.Api.DocMan.CertificatePath` | Absolute path to the client certificate `.pfx` file |
| `{instance}.{environment}.Corp.Api.DocMan.Password` | Certificate password, **AES-encrypted** using `Corp.Lib.Cryptography.Aes.Encrypt()` |

#### Concrete Example

If your `appsettings.json` contains:

```json
{
  "TargetedVoyagerInstance": "Voyager1",
  "TargetedVoyagerEnvironment": "Production"
}
```

Then the library will look for these three environment variables:

```
Voyager1.Production.Corp.Api.DocMan.Url=https://docman-api.internal.corp.com
Voyager1.Production.Corp.Api.DocMan.CertificatePath=C:\certs\docman-client.pfx
Voyager1.Production.Corp.Api.DocMan.Password=<AES-encrypted-password>
```

If any of these three values are missing or empty, the library throws a `ConfigurationErrorsException` at startup with a descriptive error message identifying exactly which key is missing.

## Registration

Registration is a single method call. In your application's `Program.cs`, call the `AddDocManApi` extension method on `WebApplicationBuilder` or `HostApplicationBuilder`:

### Web Applications

```csharp
using Corp.Api.DocMan.Lib.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Registers all DocMan service interfaces in the DI container.
// Validates configuration and throws immediately if any required
// environment variables are missing.
builder.AddDocManApi();

var app = builder.Build();
app.Run();
```

### Console Applications and Worker Services

```csharp
using Corp.Api.DocMan.Lib.Extensions;

var builder = Host.CreateApplicationBuilder(args);

// Same registration method works for HostApplicationBuilder.
builder.AddDocManApi();

var host = builder.Build();
host.Run();
```

### What `AddDocManApi` Does Behind the Scenes

1. **Reads** `TargetedVoyagerInstance` and `TargetedVoyagerEnvironment` from your `appsettings.json`.
2. **Validates** that the three required environment-variable keys (`.Url`, `.CertificatePath`, `.Password`) are present and non-empty. If any are missing, it throws `ConfigurationErrorsException` with a clear message — this is intentionally a fail-fast design so you catch misconfiguration during startup, not at runtime.
3. **Decrypts** the certificate password using `Corp.Lib.Cryptography.Aes.Decrypt()`.
4. **Registers** five Refit HTTP clients into the DI container, each configured with the base URL and mutual-TLS certificate:
   - `IFileService` / `FileService`
   - `IFolderService` / `FolderService`
   - `IFileViewAuditService` / `FileViewAuditService`
   - `IOriginalFileDeleteAuditService` / `OriginalFileDeleteAuditService`
   - `IHeartbeatService` / `HeartbeatService`

After registration, inject any of the service interfaces into your classes via constructor injection:

```csharp
public class MyController(IFileService fileService, IFolderService folderService)
{
    // Both services are fully configured with authentication and base URL.
    // Just call the methods.
}
```

## Available Services

The library exposes five service interfaces. Each method is fully async and returns `IApiResponse<T>` (see [Response Handling](#response-handling) below for details on how to interpret the response).

### IFileService

The primary service for managing files in the document store. Files represent metadata records — they track a file's name, type, claim number, and folder location. The actual binary content is managed separately.

```csharp
public interface IFileService
{
    Task<IApiResponse<List<File>>> GetAllAsync(bool includeDeleted = false, Guid? folderId = null);
    Task<IApiResponse<File>> GetByIdAsync(Guid id);
    Task<IApiResponse<List<File>>> GetByFolderIdAsync(Guid folderId);
    Task<IApiResponse<File>> GetByNameAndFhClaimNumberAsync(string name, string fhClaimNumber);
    Task<IApiResponse<string>> GetVirtualPathAsync(Guid fileId);
    Task<IApiResponse<Guid>> InsertAsync(File file);
    Task<IApiResponse<int>> InsertBatchAsync(List<File> files);
    Task<IApiResponse<int>> UpdateAsync(File file);
    Task<IApiResponse<int>> DeleteAsync(Guid id, string modifiedBy);
    Task<IApiResponse<int>> DeletePhysicalAsync(Guid id, string deletedBy);
}
```

| Method | Description |
|---|---|
| `GetAllAsync` | Retrieves all files. Pass `includeDeleted: true` to include soft-deleted records. Optionally filter by `folderId` to list files within a specific folder. |
| `GetByIdAsync` | Retrieves a single file by its unique identifier. Returns the file or a non-success status if not found. |
| `GetByFolderIdAsync` | Retrieves all non-deleted files within a specific folder. Returns a list of files whose `FolderId` matches the provided identifier. |
| `GetByNameAndFhClaimNumberAsync` | Retrieves a single non-deleted file by its `Name` and `FhClaimNumber` combination. Useful for checking whether a specific file already exists for a given claim. |
| `GetVirtualPathAsync` | Returns the full virtual path of a file including its folder hierarchy and filename with extension (e.g., `ClaimFolder/Subfolder/report.pdf`). The path is built by walking the folder tree from the file's folder up to the root. Returns `null` if the file does not exist or is soft-deleted. |
| `InsertAsync` | Creates a new file record. The `Id` may be left as `Guid.Empty` — the API will generate one. Returns the `Guid` of the newly created file. |
| `InsertBatchAsync` | Creates multiple file records in a single API call. This is significantly more efficient than calling `InsertAsync` in a loop because the API inserts all records in one database round-trip using a table-valued parameter. Returns the number of rows inserted. |
| `UpdateAsync` | Updates all mutable fields of an existing file (name, type, folder, deleted status). The `Id` field identifies which record to update. |
| `DeleteAsync` | Soft-deletes a file by setting its `Deleted` flag to `true`. The record remains in the database and can be retrieved with `GetAllAsync(includeDeleted: true)`. The `modifiedBy` parameter records who performed the deletion. |
| `DeletePhysicalAsync` | Permanently removes a file record from the database. This is irreversible — the row is physically deleted, not soft-deleted. The `deletedBy` parameter records who performed the deletion. |

### IFolderService

Manages the folder hierarchy. Folders can be nested (via `ParentFolderId`) to create a tree structure for organizing files.

```csharp
public interface IFolderService
{
    Task<IApiResponse<List<Folder>>> GetAllAsync(bool includeDeleted = false);
    Task<IApiResponse<Folder>> GetByIdAsync(Guid id);
    Task<IApiResponse<Folder>> GetByFhClaimNumberAsync(string fhClaimNumber);
    Task<IApiResponse<List<Folder>>> GetByParentFolderIdAsync(Guid parentFolderId);
    Task<IApiResponse<Guid>> InsertAsync(Folder folder);
    Task<IApiResponse<int>> UpdateAsync(Folder folder);
    Task<IApiResponse<int>> DeleteAsync(Guid id, string modifiedBy);
    Task<IApiResponse<int>> DeletePhysicalAsync(Guid id, string deletedBy);
}
```

| Method | Description |
|---|---|
| `GetAllAsync` | Retrieves all folders. Pass `includeDeleted: true` to include soft-deleted folders. |
| `GetByIdAsync` | Retrieves a single folder by its unique identifier. |
| `GetByFhClaimNumberAsync` | Retrieves a folder by its FH claim number. The folder's `Name` is matched against the provided claim number. |
| `GetByParentFolderIdAsync` | Retrieves all non-deleted child folders under a specific parent folder. Returns a list of folders whose `ParentFolderId` matches the provided identifier. |
| `InsertAsync` | Creates a new folder. Set `ParentFolderId` to nest it under an existing folder, or leave it `null` for a root-level folder. Returns the `Guid` of the newly created folder. |
| `UpdateAsync` | Updates a folder's name, parent, or deleted status. |
| `DeleteAsync` | Soft-deletes a folder. Note: this does **not** cascade to files within the folder — those must be deleted separately if desired. |
| `DeletePhysicalAsync` | Permanently removes a folder record from the database. This is irreversible — the row is physically deleted, not soft-deleted. The `deletedBy` parameter records who performed the deletion. |

### IFileViewAuditService

Records an audit trail entry each time a user views a file. These records are insert-only — there is no update or delete capability, by design, to preserve the integrity of the audit log.

```csharp
public interface IFileViewAuditService
{
    Task<IApiResponse<int>> InsertAsync(Guid? fileId, string viewedBy);
}
```

| Parameter | Description |
|---|---|
| `fileId` | The identifier of the file that was viewed. Nullable to support cases where the file reference may not be available. |
| `viewedBy` | The username or identifier of the person who viewed the file (max 100 characters). |

### IOriginalFileDeleteAuditService

Records an audit trail entry when an original file is permanently deleted from external storage. Like `IFileViewAuditService`, this service is insert-only to preserve audit integrity. A `DeletePhysicalAsync` method is available for administrative cleanup when necessary.

```csharp
public interface IOriginalFileDeleteAuditService
{
    Task<IApiResponse<int>> InsertAsync(string fhClaimNumber, string fileName, string deletedBy);
    Task<IApiResponse<int>> DeletePhysicalAsync(int id, string deletedBy);
}
```

| Parameter | Description |
|---|---|
| `fhClaimNumber` | The claim number associated with the deleted file (max 15 characters). |
| `fileName` | The name of the file that was deleted (max 100 characters). |
| `deletedBy` | The username or identifier of the person who deleted the file (max 100 characters). |

| Method | Description |
|---|---|
| `InsertAsync` | Records an audit entry for a permanent file deletion from external storage. |
| `DeletePhysicalAsync` | Permanently removes an audit record by its integer identifier. Intended for administrative cleanup only. |

### IHeartbeatService

Provides health-check endpoints to verify that the DocMan API is running and its database connection is functional. Use this service for monitoring, readiness probes, or diagnostic dashboards.

```csharp
public interface IHeartbeatService
{
    Task<IApiResponse<DateTime>> GetHeartbeatAsync();
    Task<IApiResponse<string>> GetMyRepositoryConnectionStringNameAsync();
}
```

| Method | Description |
|---|---|
| `GetHeartbeatAsync` | Returns the API server's current `DateTime`. A successful response confirms the API is reachable and processing requests. |
| `GetMyRepositoryConnectionStringNameAsync` | Returns the name of the database connection string the API is using. Useful for verifying the API is pointed at the expected database in a given environment. |

## Usage Examples

The examples below show common patterns for working with the library. All methods are async and return `IApiResponse<T>`, so you always have access to the full HTTP response.

### Insert a Single File

The most common operation. Note that you must use the fully qualified type name `Corp.Api.DocMan.Obj.Entities.File` (or a using alias) because `File` conflicts with `System.IO.File` — see [File Type Alias](#file-type-alias) below.

```csharp
using Corp.Api.DocMan.Lib.Services.Interfaces;

public class DocumentService(IFileService fileService)
{
    public async Task<Guid?> UploadFileAsync()
    {
        var file = new Corp.Api.DocMan.Obj.Entities.File
        {
            // FhClaimNumber is required and ties this file to a claim.
            FhClaimNumber = "FH1234567890123",
            Name = "report",
            FileType = ".pdf",
            // ModifiedBy should be the authenticated user performing the action.
            ModifiedBy = "jdoe"
            // Id can be omitted — the API generates a new Guid.
            // FolderId can be set to place the file in a specific folder.
        };

        var response = await fileService.InsertAsync(file);

        if (response.IsSuccessStatusCode)
        {
            // response.Content contains the Guid assigned to the new file.
            return response.Content;
        }

        // response.StatusCode tells you what went wrong (400, 500, etc.).
        // response.Error contains the ApiException with the response body.
        Console.WriteLine($"Insert failed: {response.StatusCode}");
        return null;
    }
}
```

### Batch Insert Files

When you need to create multiple file records at once (e.g., a user uploads several documents for a single claim), use `InsertBatchAsync`. This is significantly faster than looping over `InsertAsync` because the API sends all records to the database in a single round-trip using a SQL Server table-valued parameter.

```csharp
public async Task<int> UploadMultipleFilesAsync(IFileService fileService)
{
    var files = new List<Corp.Api.DocMan.Obj.Entities.File>
    {
        new() { FhClaimNumber = "FH1234567890123", Name = "doc1", FileType = ".pdf", ModifiedBy = "jdoe" },
        new() { FhClaimNumber = "FH1234567890123", Name = "doc2", FileType = ".pdf", ModifiedBy = "jdoe" },
        new() { FhClaimNumber = "FH1234567890123", Name = "doc3", FileType = ".pdf", ModifiedBy = "jdoe" }
    };

    var response = await fileService.InsertBatchAsync(files);

    if (response.IsSuccessStatusCode)
    {
        // response.Content is the number of rows inserted.
        return response.Content;
    }

    return 0;
}
```

### Manage Folders

Folders provide a hierarchical structure for organizing files. You can create nested folders by setting `ParentFolderId`, or leave it `null` for root-level folders.

```csharp
public async Task CreateAndListFoldersAsync(IFolderService folderService)
{
    // Create a root-level folder.
    var invoicesFolder = new Folder { Name = "Invoices", ModifiedBy = "jdoe" };
    var insertResponse = await folderService.InsertAsync(invoicesFolder);

    if (insertResponse.IsSuccessStatusCode)
    {
        // Create a subfolder under "Invoices" using the returned Guid.
        var archiveFolder = new Folder
        {
            Name = "Archive",
            ParentFolderId = insertResponse.Content, // Guid of the parent folder
            ModifiedBy = "jdoe"
        };
        await folderService.InsertAsync(archiveFolder);
    }

    // List all non-deleted folders.
    var response = await folderService.GetAllAsync();

    if (response.IsSuccessStatusCode)
    {
        foreach (var folder in response.Content!)
        {
            Console.WriteLine($"{folder.Name} (Parent: {folder.ParentFolderId})");
        }
    }

    // To include soft-deleted folders in the results:
    var allFolders = await folderService.GetAllAsync(includeDeleted: true);
}
```

### Record Audit Entries

The audit services are insert-only by design — once written, audit records cannot be modified or deleted. This ensures a tamper-proof audit trail.

```csharp
public async Task RecordFileViewAsync(IFileViewAuditService auditService, Guid fileId)
{
    // Record that "jdoe" viewed a specific file.
    // The fileId parameter is nullable to support scenarios where
    // the file reference is not available.
    await auditService.InsertAsync(fileId, "jdoe");
}

public async Task RecordFileDeletionAsync(
    IOriginalFileDeleteAuditService auditService,
    string fhClaimNumber,
    string fileName)
{
    // Record that "jdoe" permanently deleted the original file from storage.
    // This is separate from the soft-delete in IFileService — this audit
    // tracks when the actual binary file was removed.
    await auditService.InsertAsync(fhClaimNumber, fileName, "jdoe");
}
```

### Health Check

Use `IHeartbeatService` to verify the DocMan API is running. This is useful for Kubernetes readiness probes, monitoring dashboards, or startup validation.

```csharp
public async Task<bool> IsApiHealthyAsync(IHeartbeatService heartbeatService)
{
    var response = await heartbeatService.GetHeartbeatAsync();

    if (response.IsSuccessStatusCode)
    {
        // response.Content is the server's current DateTime.
        Console.WriteLine($"API is healthy. Server time: {response.Content}");
        return true;
    }

    Console.WriteLine($"API health check failed: {response.StatusCode}");
    return false;
}
```

## Response Handling

All service methods return Refit's `IApiResponse<T>` rather than raw values. This is a deliberate design choice — by wrapping every response, you get access to the full HTTP context without needing try/catch blocks for routine error handling.

### IApiResponse Members

| Member | Type | Description |
|---|---|---|
| `Content` | `T` | The deserialized response body. This is `default(T)` when the request fails, so always check `IsSuccessStatusCode` first. |
| `IsSuccessStatusCode` | `bool` | `true` if the HTTP status code is in the 2xx range. |
| `StatusCode` | `HttpStatusCode` | The raw HTTP status code (200, 400, 404, 500, etc.). |
| `Error` | `ApiException?` | Contains the exception details when the request fails, including the response body from the server. This is `null` on success. |
| `Headers` | `HttpResponseHeaders` | The HTTP response headers. Useful for correlation IDs or rate-limit information. |

### Recommended Pattern

Always check `IsSuccessStatusCode` before accessing `Content`. On failure, inspect `StatusCode` and `Error` for diagnostics:

```csharp
var response = await fileService.GetByIdAsync(fileId);

if (response.IsSuccessStatusCode)
{
    var file = response.Content;
    // Use the file object.
}
else if (response.StatusCode == System.Net.HttpStatusCode.NotFound)
{
    // The file doesn't exist — handle gracefully.
}
else
{
    // Unexpected failure.
    // response.Error?.Content contains the raw error body from the API.
    throw new InvalidOperationException(
        $"DocMan API returned {response.StatusCode}: {response.Error?.Content}");
}
```

### Error Logging

The service implementations already log all `ApiException` errors at `Error` level using `Corp.Lib.Logging` before rethrowing. This means the error will appear in your structured logs with the HTTP status code even if your calling code doesn't explicitly log it. You do **not** need to add your own logging in the catch block unless you want to add application-specific context.

## Entity Reference

All entity classes live in the `Corp.Api.DocMan.Obj.Entities` namespace. They are bundled into the NuGet package automatically, so you do not need a separate package reference for them. The `[Column]` attributes on each property map to the corresponding SQL Server column names.

### File

Represents a file metadata record in the document store. Each file is associated with a claim via `FhClaimNumber` and optionally placed within a folder.

| Property | Type | Constraints | Description |
|---|---|---|---|
| Id | Guid | Primary key | Unique identifier. Leave as `Guid.Empty` on insert — the API generates one. |
| FhClaimNumber | string | Required, max 15 chars | The claim number this file belongs to. Must follow the `FH` prefix format. |
| Name | string | Required, max 100 chars | The display name of the file (e.g., `"report.pdf"`). |
| FileType | string | Required, max 10 chars | The file extension without the dot (e.g., `"pdf"`, `"docx"`). |
| FolderId | Guid? | FK to Folders, nullable | The folder this file is stored in. `null` means it's at the root level. |
| KeyVersion | int | Required | The encryption key version used to encrypt the file's binary content. |
| Deleted | bool | Default: false | Soft-delete flag. Set to `true` by `DeleteAsync`; the record is not physically removed. |
| ModifiedBy | string | Required, max 100 chars | The username of the person who last created or modified this record. |

### Folder

Represents a folder in the hierarchical document structure. Folders support nesting via `ParentFolderId` and use SQL Server temporal tables for change history.

| Property | Type | Constraints | Description |
|---|---|---|---|
| Id | Guid | Primary key | Unique identifier. Generated by the database on insert. |
| ParentFolderId | Guid? | FK to Folders, nullable | The parent folder. `null` indicates a root-level folder. |
| Name | string | Required, max 100 chars | The display name of the folder. |
| Deleted | bool | Default: false | Soft-delete flag. |
| ModifiedBy | string | Required, max 100 chars | The username of the person who last created or modified this record. |
| ValidFrom | DateTime | System-managed | Temporal table row start time. Do not set this manually — SQL Server manages it automatically. |
| ValidTo | DateTime | System-managed | Temporal table row end time. Do not set this manually. |

> **About Temporal Tables**: The `Folder` table uses SQL Server's [system-versioned temporal tables](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables). Every change to a folder row is automatically tracked in a history table (`FoldersHistory`). The `ValidFrom` and `ValidTo` properties reflect the time range during which a particular version of the row was current. These fields are read-only from the application's perspective.

### FileViewAudit

An immutable audit record created each time a user views a file.

| Property | Type | Constraints | Description |
|---|---|---|---|
| FileId | Guid? | Nullable | The identifier of the file that was viewed. Nullable to support edge cases where the file reference may not be available. |
| ViewedBy | string | Required, max 100 chars | The username of the person who viewed the file. |

### OriginalFileDeleteAudit

An immutable audit record created when an original file is permanently deleted from external storage.

| Property | Type | Constraints | Description |
|---|---|---|---|
| FhClaimNumber | string | Required, max 15 chars | The claim number associated with the deleted file. |
| FileName | string | Required, max 100 chars | The name of the file that was permanently deleted. |
| DeletedBy | string | Required, max 100 chars | The username of the person who deleted the file. |

## Important Notes

### File Type Alias

The entity class `Corp.Api.DocMan.Obj.Entities.File` shares its name with `System.IO.File`, which is available in every C# file through implicit usings. The compiler will report `CS0104: 'File' is an ambiguous reference` if you try to use the unqualified name `File`.

You have two options:

**Option 1 — Fully qualified name** (recommended for occasional use):

```csharp
var file = new Corp.Api.DocMan.Obj.Entities.File { Name = "doc.pdf" };
```

**Option 2 — Using alias** (recommended when you reference the type frequently):

```csharp
using File = Corp.Api.DocMan.Obj.Entities.File;

// Now you can use "File" directly without ambiguity.
var file = new File { Name = "doc.pdf" };
```

This alias must be added to each `.cs` file where you reference the `File` entity. The `Folder`, `FileViewAudit`, and `OriginalFileDeleteAudit` types do not have this conflict and can be used directly.

### Soft Deletes

The `DeleteAsync` methods on `IFileService` and `IFolderService` perform **soft deletes** — they set the `Deleted` flag to `true` rather than physically removing the row from the database. This design preserves referential integrity and allows recovery of accidentally deleted records.

- **Default behavior**: `GetAllAsync()` returns only non-deleted records (`Deleted = false`).
- **Include deleted**: Pass `includeDeleted: true` to retrieve all records regardless of deletion status.
- **Restore a record**: To "undelete" a file or folder, call `UpdateAsync` with `Deleted = false`.

### Audit Services

The `IFileViewAuditService` and `IOriginalFileDeleteAuditService` are primarily insert-only services. `IFileViewAuditService` provides only `InsertAsync` — there are no update, delete, or retrieve operations, ensuring view audit records are immutable. `IOriginalFileDeleteAuditService` provides `InsertAsync` for recording deletions and a `DeletePhysicalAsync` method for administrative cleanup of audit records when necessary.

## Troubleshooting

### `ConfigurationErrorsException` at startup

This means one or more required configuration values are missing. The exception message tells you exactly which key is missing. Double-check:

1. Your `appsettings.json` contains `TargetedVoyagerInstance` and `TargetedVoyagerEnvironment`.
2. The corresponding environment variables (`{instance}.{environment}.Corp.Api.DocMan.Url`, `.CertificatePath`, `.Password`) are set and non-empty.

### `CS0104: 'File' is an ambiguous reference`

See [File Type Alias](#file-type-alias) above. Add `using File = Corp.Api.DocMan.Obj.Entities.File;` to the top of the affected file.

### `ApiException` with `401 Unauthorized`

The client certificate is either missing, expired, or not trusted by the API server. Verify:

1. The `.pfx` file exists at the path specified in `{instance}.{environment}.Corp.Api.DocMan.CertificatePath`.
2. The encrypted password can be decrypted successfully.
3. The certificate is not expired.
4. The API server trusts the certificate's issuing CA.

### `ApiException` with `404 Not Found`

Check that the API base URL (`{instance}.{environment}.Corp.Api.DocMan.Url`) is correct and includes the scheme (`https://`). Also verify the API version — this library targets API version `v1`.

## Solution Architecture

The DocMan solution is split into four projects, each with a distinct responsibility:

| Project | Type | Description |
|---|---|---|
| **Corp.Api.DocMan** | ASP.NET Core Web API | The HTTP API itself. Contains versioned controllers (`FileController`, `FolderController`, etc.) that receive requests and delegate to repositories. This project is deployed as the running service. |
| **Corp.Api.DocMan.Data** | Class Library | Data access layer. Contains Dapper-based repository classes that execute stored procedures against SQL Server. Each repository maps to a set of related stored procedures. |
| **Corp.Api.DocMan.Obj** | Class Library | Shared entity classes (`File`, `Folder`, `FileViewAudit`, `OriginalFileDeleteAudit`). These are used by both the API (server-side) and this client library (consumer-side). The `.csproj` references `Microsoft.AspNetCore.App` for model validation attributes. |
| **Corp.Api.DocMan.Lib** | Class Library (NuGet) | **This library.** Contains Refit service interfaces, service implementations with logging, and the DI registration extension method. Published as a NuGet package. |

### How the NuGet Package Is Built

The `Corp.Api.DocMan.Lib.csproj` has `GeneratePackageOnBuild=True`, so building the project automatically produces a `.nupkg` file. The `Corp.Api.DocMan.Obj` project is referenced with `PrivateAssets="All"` and a custom MSBuild target (`CopyProjectReferencesToPackage`) copies its assembly into the NuGet package. This means consumers get the entity types automatically without needing a separate package reference.

### API Versioning

The API uses `Asp.Versioning.Mvc` with the default version set to `1.0`. The Refit route attributes in this library use relative paths (e.g., `/Folder/GetAll`). The `/api/v1` prefix is part of the base URL configured at client registration time. If a `v2` is introduced in the future, a corresponding update to this library will be required.

## Source Repository

| | |
|---|---|
| **Azure DevOps** | https://dev.azure.com/IT-Specialty/Projects/_git/Corp.Solution.Api.DocMan |
| **Branch** | `Version-10` |
| **Author** | Mathew Hamilton |
| **Company** | Sedgwick Consumer Claims |
| **Package Version** | `10.1.1` |
