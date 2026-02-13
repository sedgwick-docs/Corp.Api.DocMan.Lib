# Corp.Api.DocMan.Lib

NuGet service library for the DocMan API. Provides typed Refit HTTP clients for file management, folder management, and audit operations.

## Prerequisites

- .NET 10 or later
- A deployed instance of the **Corp.Api.DocMan** API
- A valid client certificate and encrypted password for mutual-TLS authentication

## Installation

```shell
dotnet add package Corp.Api.DocMan.Lib --version 10.0.0-beta1
```

## Configuration

### appsettings.json

The library reads three values from the consuming application's `appsettings.json` at startup:

```json
{
  "TargetedVoyagerInstance": "<instance>",
  "TargetedVoyagerEnvironment": "<environment>",
  "ApplicationName": "<applicationName>"
}
```

### Environment Variables

Using the `instance`, `environment`, and `applicationName` values above, the library resolves the following configuration keys from `IConfiguration` (typically supplied via environment variables):

| Key | Description |
|---|---|
| `{instance}.{environment}.{applicationName}.Url` | Base URL of the DocMan API |
| `{instance}.{environment}.{applicationName}.CertificatePath` | Path to the client certificate (.pfx) |
| `{instance}.{environment}.{applicationName}.Password` | AES-encrypted certificate password |

## Registration

In your application's `Program.cs`, call the `AddDocManApi` extension method on `WebApplicationBuilder`:

```csharp
using Corp.Api.DocMan.Lib.Extensions;

var builder = WebApplication.CreateBuilder(args);

builder.AddDocManApi();

var app = builder.Build();
app.Run();
```

This registers all service interfaces in the DI container. Inject them via constructor injection.

## Available Services

### IFileService

Manages files in the document store.

```csharp
public interface IFileService
{
    Task<IApiResponse<List<File>>> GetAllAsync(bool includeDeleted = false, Guid? folderId = null);
    Task<IApiResponse<File>> GetByIdAsync(Guid id);
    Task<IApiResponse<Guid>> InsertAsync(File file);
    Task<IApiResponse<int>> InsertBatchAsync(List<File> files);
    Task<IApiResponse<int>> UpdateAsync(File file);
    Task<IApiResponse<int>> DeleteAsync(Guid id, string modifiedBy);
}
```

### IFolderService

Manages folders in the document store.

```csharp
public interface IFolderService
{
    Task<IApiResponse<List<Folder>>> GetAllAsync(bool includeDeleted = false);
    Task<IApiResponse<Folder>> GetByIdAsync(Guid id);
    Task<IApiResponse<int>> InsertAsync(Folder folder);
    Task<IApiResponse<int>> UpdateAsync(Folder folder);
    Task<IApiResponse<int>> DeleteAsync(Guid id, string modifiedBy);
}
```

### IFileViewAuditService

Records file view audit entries.

```csharp
public interface IFileViewAuditService
{
    Task<IApiResponse<int>> InsertAsync(Guid? fileId, string viewedBy);
}
```

### IOriginalFileDeleteAuditService

Records original file deletion audit entries.

```csharp
public interface IOriginalFileDeleteAuditService
{
    Task<IApiResponse<int>> InsertAsync(string fileName, string deletedBy);
}
```

### IHeartbeatService

Health-check endpoint for the DocMan API.

```csharp
public interface IHeartbeatService
{
    Task<IApiResponse<DateTime>> GetHeartbeatAsync();
    Task<IApiResponse<string>> GetMyRepositoryConnectionStringNameAsync();
}
```

## Usage Example

```csharp
using Corp.Api.DocMan.Lib.Services.Interfaces;

public class MyService(IFileService fileService)
{
    public async Task UploadFileAsync()
    {
        var file = new Corp.Api.DocMan.Obj.Entities.File
        {
            FhClaimNumber = "FH1234567890123",
            Name = "report.pdf",
            FileType = "pdf",
            ModifiedBy = "jdoe"
        };

        var response = await fileService.InsertAsync(file);

        if (response.IsSuccessStatusCode)
        {
            var newFileId = response.Content;
            // use newFileId
        }
        else
        {
            // handle error - response.StatusCode, response.Error
        }
    }
}
```

## Response Handling

All service methods return Refit `IApiResponse<T>`. Key members:

| Member | Type | Description |
|---|---|---|
| Content | T | Deserialized response body |
| IsSuccessStatusCode | bool | true if the HTTP status code indicates success |
| StatusCode | HttpStatusCode | HTTP status code |
| Error | ApiException | Exception details when the request fails |
| Headers | HttpResponseHeaders | Response headers |

## Entity Reference

### File

| Property | Type | Description |
|---|---|---|
| Id | Guid | Unique identifier |
| FhClaimNumber | string | Claim number (required, max 15 chars) |
| Name | string | File name (required, max 100 chars) |
| FileType | string | File extension (required, max 10 chars) |
| FolderId | Guid? | Parent folder identifier (optional) |
| Deleted | bool | Soft-delete flag |
| ModifiedBy | string | User who last modified (required, max 100 chars) |

### Folder

| Property | Type | Description |
|---|---|---|
| Id | Guid | Unique identifier |
| ParentFolderId | Guid? | Parent folder identifier (optional) |
| Name | string | Folder name (required, max 100 chars) |
| Deleted | bool | Soft-delete flag |
| ModifiedBy | string | User who last modified (required, max 100 chars) |
| ValidFrom | DateTime | Row validity start (system-managed) |
| ValidTo | DateTime | Row validity end (system-managed) |
