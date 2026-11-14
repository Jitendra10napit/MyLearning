In Azure, "block storage" for files means **Blob Storage (block blobs)**. Here's the .NET API code, the Angular code to call it, and the Lead/SA points.

**Step 1 – NuGet packages**

```
dotnet add package Azure.Storage.Blobs
dotnet add package Azure.Identity
dotnet add package Microsoft.Extensions.Azure
```

**Step 2 – appsettings.json**

```json
"BlobStorage": {
  "AccountUrl": "https://mystorageacct.blob.core.windows.net",
  "Container": "employee-docs",
  "MaxFileSizeMb": 10
}
```

Say: *"No connection string or keys. I use Managed Identity through DefaultAzureCredential, so there are no secrets in config."*

**Step 3 – Register BlobServiceClient (singleton)**

```csharp
builder.Services.AddAzureClients(c =>
{
    c.AddBlobServiceClient(new Uri(builder.Configuration["BlobStorage:AccountUrl"]!));
    c.UseCredential(new DefaultAzureCredential());
});
builder.Services.AddScoped<IBlobService, BlobService>();
```

Say: *"BlobServiceClient is thread-safe and meant to be reused, so it's registered as a singleton."*

**Step 4 – Service**

```csharp
public interface IBlobService
{
    Task<string> UploadAsync(IFormFile file, CancellationToken ct);
    Task<(Stream Content, string ContentType)> DownloadAsync(string blobName, CancellationToken ct);
    Task DeleteAsync(string blobName, CancellationToken ct);
}

public class BlobService(BlobServiceClient client, IConfiguration config) : IBlobService
{
    private readonly BlobContainerClient _container =
        client.GetBlobContainerClient(config["BlobStorage:Container"]);

    public async Task<string> UploadAsync(IFormFile file, CancellationToken ct)
    {
        await _container.CreateIfNotExistsAsync(PublicAccessType.None, cancellationToken: ct);

        // Unique name: avoids overwrites and path tricks in the original file name
        var ext = Path.GetExtension(file.FileName);
        var blobName = $"{DateTime.UtcNow:yyyy/MM/dd}/{Guid.NewGuid()}{ext}";

        var blob = _container.GetBlobClient(blobName);

        await using var stream = file.OpenReadStream();     // stream it, don't load it into memory
        await blob.UploadAsync(stream, new BlobUploadOptions
        {
            HttpHeaders = new BlobHttpHeaders { ContentType = file.ContentType },
            Metadata = new Dictionary<string, string> { ["originalName"] = file.FileName },
            TransferOptions = new StorageTransferOptions
            {
                InitialTransferSize = 4 * 1024 * 1024,   // 4 MB in one shot
                MaximumTransferSize = 4 * 1024 * 1024,   // then 4 MB blocks
                MaximumConcurrency = 4                   // blocks uploaded in parallel
            }
        }, ct);

        return blobName;
    }

    public async Task<(Stream, string)> DownloadAsync(string blobName, CancellationToken ct)
    {
        var blob = _container.GetBlobClient(blobName);
        if (!await blob.ExistsAsync(ct))
            throw new NotFoundException($"File {blobName} not found");

        var res = await blob.DownloadStreamingAsync(cancellationToken: ct);
        return (res.Value.Content, res.Value.Details.ContentType);
    }

    public async Task DeleteAsync(string blobName, CancellationToken ct) =>
        await _container.GetBlobClient(blobName)
                        .DeleteIfExistsAsync(DeleteSnapshotsOption.IncludeSnapshots, cancellationToken: ct);
}
```

Say:
- *"The SDK splits large files into blocks and uploads them in parallel, then commits the block list. That's what 'block blob' means."*
- *"I stream the file, so a 500 MB file never sits in API memory."*
- *"I generate the name myself. Never trust a user-supplied file name."*

**Step 5 – Controller with validation**

```csharp
[ApiController]
[Route("api/v1/employees/{employeeId:int}/documents")]
[Authorize]
public class DocumentsController(IBlobService blobs) : ControllerBase
{
    private static readonly string[] Allowed = [".pdf", ".jpg", ".png", ".docx"];
    private const long MaxBytes = 10 * 1024 * 1024;

    [HttpPost]
    [RequestSizeLimit(MaxBytes)]
    public async Task<IActionResult> Upload(int employeeId, IFormFile file, CancellationToken ct)
    {
        if (file is null || file.Length == 0)
            return BadRequest("File is empty");
        if (file.Length > MaxBytes)
            return BadRequest("Max 10 MB allowed");

        var ext = Path.GetExtension(file.FileName).ToLowerInvariant();
        if (!Allowed.Contains(ext))
            return BadRequest($"Allowed types: {string.Join(", ", Allowed)}");

        var blobName = await blobs.UploadAsync(file, ct);
        // Save blobName + employeeId + original name in the DB here

        return CreatedAtAction(nameof(Download), new { employeeId, blobName }, new { blobName });
    }

    [HttpGet("{*blobName}")]
    public async Task<IActionResult> Download(int employeeId, string blobName, CancellationToken ct)
    {
        var (content, type) = await blobs.DownloadAsync(blobName, ct);
        return File(content, type, Path.GetFileName(blobName));
    }

    [HttpDelete("{*blobName}")]
    public async Task<IActionResult> Delete(int employeeId, string blobName, CancellationToken ct)
    {
        await blobs.DeleteAsync(blobName, ct);
        return NoContent();
    }
}
```

Say: *"Validate size and extension on the server. For stricter checks, look at the magic bytes and run a malware scan with Defender for Storage."*

**Step 6 – Large files: direct upload with a SAS token (the SA-level answer)**

For big files, the browser uploads straight to Blob Storage, so the file never passes through your API.

```csharp
[HttpPost("upload-url")]
public async Task<IActionResult> GetUploadUrl(int employeeId, [FromBody] string fileName, CancellationToken ct)
{
    var blobName = $"{employeeId}/{Guid.NewGuid()}{Path.GetExtension(fileName)}";
    var blob = _container.GetBlobClient(blobName);

    // User delegation SAS: signed with Entra ID, no account key needed
    var key = await _client.GetUserDelegationKeyAsync(
        DateTimeOffset.UtcNow, DateTimeOffset.UtcNow.AddMinutes(15), ct);

    var sas = new BlobSasBuilder
    {
        BlobContainerName = _container.Name,
        BlobName = blobName,
        Resource = "b",
        ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(15)
    };
    sas.SetPermissions(BlobSasPermissions.Create | BlobSasPermissions.Write);

    var uri = new BlobUriBuilder(blob.Uri)
    {
        Sas = sas.ToSasQueryParameters(key.Value, _client.AccountName)
    }.ToUri();

    return Ok(new { uploadUrl = uri, blobName });
}
```

Flow:

```
Angular → API: "I want to upload x.pdf"
API → Angular: SAS URL (write-only, 15 min, this one blob)
Angular → Blob Storage: PUT file directly
Blob Created event → Event Grid → Function: scan / thumbnail / save metadata
```

Say: *"This offloads bandwidth and CPU from the API, scales without limit, and the SAS is short-lived and scoped to a single blob."*

**Step 7 – Angular upload with a progress bar**

```ts
// service
upload(employeeId: number, file: File): Observable<HttpEvent<{ blobName: string }>> {
  const form = new FormData();
  form.append('file', file);                 // name must match the IFormFile parameter
  return this.http.post<{ blobName: string }>(
    `${this.url}/${employeeId}/documents`, form,
    { reportProgress: true, observe: 'events' });
}
```

```ts
// component
@Component({
  standalone: true,
  template: `
    <input type="file" (change)="onSelect($event)" accept=".pdf,.jpg,.png,.docx" />
    <button (click)="upload()" [disabled]="!file">Upload</button>
    @if (progress() > 0) { <progress [value]="progress()" max="100"></progress> {{ progress() }}% }
  `
})
export class UploadComponent {
  private svc = inject(EmployeeService);
  file?: File;
  progress = signal(0);

  onSelect(e: Event) {
    this.file = (e.target as HTMLInputElement).files?.[0];
    if (this.file && this.file.size > 10 * 1024 * 1024) {
      alert('Max 10 MB'); this.file = undefined;
    }
  }

  upload() {
    this.svc.upload(1, this.file!).subscribe(event => {
      if (event.type === HttpEventType.UploadProgress && event.total)
        this.progress.set(Math.round(100 * event.loaded / event.total));
      if (event.type === HttpEventType.Response)
        alert('Uploaded: ' + event.body?.blobName);
    });
  }
}
```

Say: *"Don't set the Content-Type header yourself for FormData. The browser adds the multipart boundary."*

**Lead/SA talking points:**
- **Security:** Managed Identity with the `Storage Blob Data Contributor` RBAC role, a private container, Private Endpoint, and short-lived user-delegation SAS tokens.
- **Large files:** a direct SAS upload, or the block API (`StageBlock` + `CommitBlockList`) for resumable uploads.
- **Dedup and cost:** name blobs by content hash (SHA-256). The same file is stored once, which also saves downstream processing.
- **Lifecycle policies:** move blobs Hot → Cool → Archive by age, and auto-delete old ones.
- **Processing:** a Blob Created event goes to Event Grid, then an Azure Function handles scanning, thumbnails or OCR, so it's async and decoupled.
- **Reliability:** the SDK retries automatically. Use ZRS/GRS for redundancy, and soft delete plus versioning for recovery.
- **The DB stores metadata only** (blobName, size, type, owner). Never store file bytes in SQL.

**Order to write:** packages → config → DI → service → controller + validation → (SAS for large files) → Angular FormData + progress.
