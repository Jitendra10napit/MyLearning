Yes. At code level, I would implement this as **two separate paths**:

1. **Control plane:** .NET API creates the upload session and returns a short-lived SAS URL.
2. **Data plane:** Client uploads the 20 GB file directly to Blob using chunk/block upload.
3. After upload completes → Blob event → Service Bus → worker processes the file.

### 1. Overall implementation

```text
                    CONTROL PLANE
Client
  │
  │ POST /api/uploads
  │ { fileName, size }
  ▼
.NET Upload API
  │
  ├── Authenticate / Authorize
  ├── Validate file
  ├── Create UploadId
  ├── Generate SAS
  └── Save upload metadata
  │
  │ 202 + SAS URL
  ▼
Client
  │
  │
  │              DATA PLANE
  └──────────────────────────────► Azure Blob
                   20 GB
              Block/Chunk Upload
                    │
                    ▼
              Blob Completed
                    │
                    ▼
               Event Grid
                    │
                    ▼
              Service Bus
                    │
                    ▼
          Video Processing Worker
             FFmpeg / FFprobe
```

---

# 2. .NET API — Create upload session

The API **doesn't receive the 20 GB file**.

It only creates the upload session.

```csharp
[ApiController]
[Route("api/uploads")]
public class UploadController : ControllerBase
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly IUploadRepository _uploadRepository;

    public UploadController(
        BlobServiceClient blobServiceClient,
        IUploadRepository uploadRepository)
    {
        _blobServiceClient = blobServiceClient;
        _uploadRepository = uploadRepository;
    }

    [HttpPost]
    public async Task<IActionResult> CreateUpload(
        CreateUploadRequest request)
    {
        // 1. Authenticate/Authorize
        // Normally handled by [Authorize]

        // 2. Validate
        if (request.FileSize > 50L * 1024 * 1024 * 1024)
        {
            return BadRequest("File size exceeds limit.");
        }

        var uploadId = Guid.NewGuid();

        var blobName =
            $"uploads/{uploadId}/{request.FileName}";

        // 3. Save metadata
        var upload = new Upload
        {
            Id = uploadId,
            FileName = request.FileName,
            FileSize = request.FileSize,
            BlobName = blobName,
            Status = UploadStatus.Uploading
        };

        await _uploadRepository.CreateAsync(upload);

        // 4. Generate short-lived SAS
        var sasUri = GenerateUploadSas(blobName);

        // 5. Return upload information
        return Accepted(new
        {
            uploadId,
            blobName,
            uploadUrl = sasUri,
            expiresInMinutes = 30
        });
    }
}
```

The important point for the interview:

> **The API returns a temporary SAS URL instead of receiving the 20 GB payload.**

---

# 3. Generate SAS

For production, I would prefer **User Delegation SAS** with Microsoft Entra ID rather than storing an account key.

Conceptually:

```csharp
private Uri GenerateUploadSas(string blobName)
{
    var containerClient =
        _blobServiceClient.GetBlobContainerClient("videos");

    var blobClient =
        containerClient.GetBlobClient(blobName);

    var sasBuilder = new BlobSasBuilder
    {
        BlobContainerName = "videos",
        BlobName = blobName,
        Resource = "b",

        StartsOn = DateTimeOffset.UtcNow.AddMinutes(-5),
        ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(30)
    };

    sasBuilder.SetPermissions(
        BlobSasPermissions.Create |
        BlobSasPermissions.Write);

    return blobClient.GenerateSasUri(sasBuilder);
}
```

In a real production architecture, I'd additionally restrict:

```text
Short expiry
     +
Specific blob
     +
Write/Create only
     +
HTTPS
     +
Authentication/Authorization
```

---

# 4. Client uploads 20 GB

Now the important part.

The client **doesn't send this**:

```text
Client
   │
   │ 20 GB
   ▼
.NET API
```

Instead:

```text
Client
   │
   │ 8 MB block
   ├──────────────────► Blob
   │
   │ 8 MB block
   ├──────────────────► Blob
   │
   │ 8 MB block
   ├──────────────────► Blob
   │
   │ ...
   │
   │ 20 GB
   ▼
Azure Blob
```

Azure Blob Storage supports block blobs specifically for this type of scenario.

---

# 5. C# client example

Suppose your desktop application/client is also .NET.

You can use `BlobClient`.

```csharp
var blobClient = new BlobClient(
    new Uri(uploadUrl));

await blobClient.UploadAsync(
    filePath,
    new BlobUploadOptions
    {
        TransferOptions = new StorageTransferOptions
        {
            InitialTransferSize = 8 * 1024 * 1024,
            MaximumTransferSize = 8 * 1024 * 1024,
            MaximumConcurrency = 4
        }
    });
```

Conceptually this becomes:

```text
20 GB file

       Split into blocks

┌────────┐
│  8 MB  │ ────────┐
└────────┘         │
                   │
┌────────┐         ├──► Azure Blob
│  8 MB  │ ────────┤
└────────┘         │
                   │
┌────────┐         │
│  8 MB  │ ────────┤
└────────┘         │
                   │
      ...          │
                   │
┌────────┐         │
│  8 MB  │ ────────┘
└────────┘
```

With:

```csharp
MaximumConcurrency = 4
```

you can have multiple blocks uploading concurrently.

---

# 6. Why chunking matters

Imagine:

```text
20 GB
```

and the network fails after:

```text
19 GB
```

If you were treating it as one giant upload, recovery could be painful.

With blocks:

```text
Block 1   ✓
Block 2   ✓
Block 3   ✓
...
Block 2500 ✓
Block 2501 ✗
```

The client can retry the failed block rather than retransmitting the entire file.

That's one of the major architectural benefits.

---

# 7. Browser/JavaScript client

If your frontend is Angular, you can also use the SAS URL directly.

For example:

```typescript
async uploadFile(file: File, sasUrl: string) {

    const blockSize = 8 * 1024 * 1024;

    let offset = 0;

    while (offset < file.size) {

        const chunk = file.slice(
            offset,
            Math.min(offset + blockSize, file.size)
        );

        // Upload chunk to Blob

        offset += chunk.size;

        const progress =
            (offset / file.size) * 100;

        console.log(`${progress}%`);
    }
}
```

In production, I'd use the Azure Storage JavaScript SDK rather than manually implementing the complete block protocol.

---

# 8. What happens after upload?

This is where your architecture becomes event-driven.

Once the Blob is successfully created:

```text
Azure Blob
    │
    │ BlobCreated
    ▼
Event Grid
    │
    ▼
Service Bus Topic/Queue
    │
    │
    ▼
Video Processing Worker
```

The message could look like:

```json
{
  "uploadId": "8f8c...",
  "blobName": "uploads/8f8c/video.mp4",
  "container": "videos",
  "fileName": "video.mp4"
}
```

---

# 9. Service Bus publisher

For example:

```csharp
public class VideoMessage
{
    public Guid UploadId { get; set; }

    public string BlobName { get; set; } = "";

    public string ContainerName { get; set; } = "";

    public string FileName { get; set; } = "";
}
```

Publisher:

```csharp
public async Task PublishAsync(VideoMessage message)
{
    var sender = _serviceBusClient
        .CreateSender("video-processing");

    var json = JsonSerializer.Serialize(message);

    var serviceBusMessage =
        new ServiceBusMessage(json)
        {
            MessageId = message.UploadId.ToString(),
            Subject = "VideoUploaded"
        };

    await sender.SendMessageAsync(serviceBusMessage);
}
```

---

# 10. Worker receives message

Your worker could be an Azure Function:

```csharp
[Function("ProcessVideo")]
public async Task Run(
    [ServiceBusTrigger(
        "video-processing",
        Connection = "ServiceBusConnection")]
    string message)
{
    var video =
        JsonSerializer.Deserialize<VideoMessage>(message);

    Console.WriteLine(
        $"Processing {video!.BlobName}");

    // 1. Download/stream Blob
    // 2. FFprobe metadata
    // 3. FFmpeg processing
    // 4. Store output
    // 5. Update DB status
}
```

For your NICE/CCTV scenario:

```text
Service Bus
     │
     ▼
Video Processing Worker
     │
     ├── FFprobe
     │      └── metadata
     │
     ├── FFmpeg
     │      ├── Transcoding
     │      ├── Extraction
     │      └── Conversion
     │
     └── Update DB
```

---

# 11. Database status

I'd maintain an upload/processing state.

```csharp
public enum UploadStatus
{
    Uploading,
    Uploaded,
    Processing,
    Completed,
    Failed
}
```

Flow:

```text
POST /uploads
      │
      ▼
Uploading
      │
      │ Blob completed
      ▼
Uploaded
      │
      │ Service Bus
      ▼
Processing
      │
      │ FFmpeg successful
      ▼
Completed
```

If processing fails:

```text
Processing
     │
     ▼
Failed
```

and the Service Bus retry/DLQ mechanism handles the messaging failure scenario.

---

# 12. The important production issue: DB + Service Bus

There is one subtle architect-level problem.

Suppose:

```text
Blob uploaded ✓

DB update ✓

Service Bus publish ✗
```

Now your database says:

```text
Uploaded
```

but nobody processes the video.

This is where I'd consider an **Outbox Pattern**.

```text
                 SQL
                  │
       ┌──────────┴──────────┐
       │                     │
 UploadStatus          OutboxMessage
   Uploaded             Pending
                           │
                           ▼
                    Outbox Publisher
                           │
                           ▼
                      Service Bus
                           │
                           ▼
                        Worker
```

That's a very good architect-level follow-up answer.

---

# 13. Final architecture I'd explain in interview

```text
                       CONTROL PLANE
                  ┌─────────────────────┐
                  │     .NET API        │
                  │                     │
Client ──────────►│ Auth/AuthZ          │
                  │ Validation          │
                  │ Upload Session      │
                  │ SAS Generation      │
                  │ Metadata            │
                  └──────────┬──────────┘
                             │
                         SAS URL
                             │
                             ▼
                       DATA PLANE
Client ═══════════════════════════════► Azure Blob
          Chunked / Parallel Upload       │
                                          │
                                   Blob Created
                                          │
                                          ▼
                                    Event Grid
                                          │
                                          ▼
                                    Service Bus
                                          │
                                          ▼
                              ┌────────────────────┐
                              │ Video Worker       │
                              │                    │
                              │ FFprobe             │
                              │ FFmpeg              │
                              │ OCR                 │
                              │ Metadata extraction │
                              └─────────┬──────────┘
                                        │
                              ┌─────────┴─────────┐
                              ▼                   ▼
                            SQL DB             Blob
                         Metadata/Status      Processed
```

### What I would say in the interview

“For a large file such as a 20 GB CCTV video, I would avoid sending the file through the .NET API because that would make the API responsible for a long-running, high-bandwidth operation.

I would separate the control plane from the data plane.

The client first calls my .NET API to create an upload session. The API authenticates and authorizes the user, validates the file metadata, creates an upload record, and generates a short-lived, scoped SAS URL.

The API returns that SAS URL to the client. The client then uploads the file directly to Azure Blob Storage using block or chunked upload with bounded parallelism. This means the 20 GB payload doesn't flow through my API servers.

If the network fails, individual blocks can be retried or the upload can resume rather than restarting the entire file.

Once Blob Storage has the complete file, a Blob Created event is generated. I use Event Grid and Service Bus to decouple ingestion from processing. The processing worker consumes the Service Bus message and performs operations such as FFprobe metadata extraction, FFmpeg transcoding or video extraction, and then updates the processing status.

For reliability, I would use retries and a dead-letter queue, make processing idempotent, and consider the Outbox Pattern if I need reliable coordination between database state changes and message publishing.

So the key design principle is: the API handles control-plane operations, Blob handles the large data transfer, and Service Bus decouples the asynchronous processing workload.”

**Memorable one-liner:**

> **“Don't make the API carry the 20 GB payload; let the API control the upload and let Blob carry the data.”**
