Here's that design as code, in the same order a request flows through it.

**The flow**

```
Angular ──1. POST /jobs──────────────► API ── creates Job (Pending), returns SAS URL
Angular ──2. PUT file (2 GB)─────────► Blob Storage   (direct, API never sees bytes)
Angular ──3. POST /jobs/{id}/submit──► API ── sends {jobId, blobName} to Service Bus
                                           ◄── 202 Accepted + Location: /jobs/{id}/status
Service Bus ──4. message─────────────► Worker (Function / AKS)
                                           hash → duplicate? skip : process → update Job
Angular ──5. GET /jobs/{id}/status───► API  (poll)   or   SignalR push ◄── Worker
```

**Step 0 – Job tables**

```csharp
public enum JobStatus { Pending, Queued, Processing, Completed, Failed }

public class MediaJob
{
    public Guid Id { get; set; }
    public string BlobName { get; set; } = "";
    public JobStatus Status { get; set; }
    public int Progress { get; set; }            // 0–100
    public string? ContentHash { get; set; }
    public string? ResultUrl { get; set; }
    public string? Error { get; set; }
    public DateTime CreatedAt { get; set; }
}

// One row per unique file content, used for de-duplication
public class MediaFile
{
    public int Id { get; set; }
    public string ContentHash { get; set; } = "";  // unique index
    public string ResultUrl { get; set; } = "";
}

// OnModelCreating
b.Entity<MediaFile>().HasIndex(f => f.ContentHash).IsUnique();
```

Say: *"The job table is the state machine the client polls. The unique index on ContentHash is the final safety net against duplicates."*

**Step 1 – API: create job + SAS URL (Valet Key pattern)**

```csharp
[ApiController]
[Route("api/v1/jobs")]
[Authorize]
public class JobsController(
    AppDbContext db,
    BlobServiceClient blobClient,
    ServiceBusSender sender) : ControllerBase
{
    private const string Container = "media-uploads";

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateJobRequest req, CancellationToken ct)
    {
        var job = new MediaJob
        {
            Id = Guid.NewGuid(),
            BlobName = $"{Guid.NewGuid()}{Path.GetExtension(req.FileName)}",
            Status = JobStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };
        db.MediaJobs.Add(job);
        await db.SaveChangesAsync(ct);

        var sasUrl = await CreateWriteSasAsync(job.BlobName, ct);
        return Ok(new { jobId = job.Id, uploadUrl = sasUrl });
    }

    private async Task<Uri> CreateWriteSasAsync(string blobName, CancellationToken ct)
    {
        var blob = blobClient.GetBlobContainerClient(Container).GetBlobClient(blobName);
        var key = await blobClient.GetUserDelegationKeyAsync(
            DateTimeOffset.UtcNow.AddMinutes(-5), DateTimeOffset.UtcNow.AddHours(1), ct);

        var sas = new BlobSasBuilder
        {
            BlobContainerName = Container,
            BlobName = blobName,
            Resource = "b",
            ExpiresOn = DateTimeOffset.UtcNow.AddHours(1)   // 2 GB needs time
        };
        sas.SetPermissions(BlobSasPermissions.Create | BlobSasPermissions.Write);

        return new BlobUriBuilder(blob.Uri)
        {
            Sas = sas.ToSasQueryParameters(key.Value, blobClient.AccountName)
        }.ToUri();
    }
}

public record CreateJobRequest(string FileName);
```

**Step 2 – API: submit, which returns 202 Accepted (Async Request-Reply + Claim Check)**

```csharp
public record ProcessMediaMessage(Guid JobId, string BlobName);   // the "claim check"

[HttpPost("{id:guid}/submit")]
public async Task<IActionResult> Submit(Guid id, CancellationToken ct)
{
    var job = await db.MediaJobs.FindAsync([id], ct);
    if (job is null) return NotFound();

    if (job.Status == JobStatus.Pending)          // idempotent: submitting twice is safe
    {
        var msg = new ServiceBusMessage(BinaryData.FromObjectAsJson(
            new ProcessMediaMessage(job.Id, job.BlobName)))
        {
            MessageId = job.Id.ToString()          // Service Bus duplicate detection
        };
        await sender.SendMessageAsync(msg, ct);

        job.Status = JobStatus.Queued;
        await db.SaveChangesAsync(ct);
    }

    var statusUrl = Url.Action(nameof(GetStatus), new { id })!;
    Response.Headers.RetryAfter = "10";           // tell the client how often to poll
    return Accepted(statusUrl, new { jobId = id, status = job.Status.ToString(), statusUrl });
}
```

Say:
- *"202 means accepted but not done yet. The Location header points to the status URL."*
- *"The message is tiny, just an id and a blob name. Service Bus has message size limits, and you never put a 2 GB file on a queue. That's the Claim Check pattern."*
- *"MessageId = JobId, so Service Bus duplicate detection drops accidental double sends."*

DI registration:

```csharp
builder.Services.AddAzureClients(c =>
{
    c.AddBlobServiceClient(new Uri(cfg["Storage:Url"]!));
    c.AddServiceBusClientWithNamespace(cfg["ServiceBus:Namespace"]);
    c.UseCredential(new DefaultAzureCredential());
});
builder.Services.AddSingleton(sp =>
    sp.GetRequiredService<ServiceBusClient>().CreateSender("media-jobs"));
```

Alternative to the submit call: a **BlobCreated event through Event Grid** can trigger processing automatically once the upload finishes.

**Step 3 – API: status endpoint**

```csharp
[HttpGet("{id:guid}/status")]
public async Task<IActionResult> GetStatus(Guid id, CancellationToken ct)
{
    var job = await db.MediaJobs.AsNoTracking().FirstOrDefaultAsync(j => j.Id == id, ct);
    if (job is null) return NotFound();

    if (job.Status is not (JobStatus.Completed or JobStatus.Failed))
        Response.Headers.RetryAfter = "10";

    return Ok(new
    {
        job.Id,
        Status = job.Status.ToString(),
        job.Progress,
        job.ResultUrl,     // set when Completed
        job.Error          // set when Failed
    });
}
```

**Step 4 – Worker: Azure Function with hash-based de-duplication**

```csharp
public class ProcessMediaFunction(
    AppDbContext db,
    BlobServiceClient blobClient,
    IMediaProcessor processor,
    IJobNotifier notifier,
    ILogger<ProcessMediaFunction> log)
{
    [Function("ProcessMedia")]
    public async Task Run(
        [ServiceBusTrigger("media-jobs", Connection = "ServiceBus")] ProcessMediaMessage msg,
        CancellationToken ct)
    {
        var job = await db.MediaJobs.FindAsync([msg.JobId], ct);

        // Idempotency: Service Bus delivers at least once, so the same message can arrive twice
        if (job is null || job.Status is JobStatus.Completed) return;

        await UpdateAsync(job, JobStatus.Processing, 5, ct);

        var blob = blobClient.GetBlobContainerClient("media-uploads").GetBlobClient(msg.BlobName);

        // 1. Hash: stream it, never load 2 GB into memory
        string hash;
        await using (var stream = await blob.OpenReadAsync(cancellationToken: ct))
        {
            hash = Convert.ToHexString(await SHA256.HashDataAsync(stream, ct));
        }
        job.ContentHash = hash;

        // 2. Duplicate? Reuse the existing result, skip processing completely
        var existing = await db.MediaFiles.AsNoTracking()
                               .FirstOrDefaultAsync(f => f.ContentHash == hash, ct);
        if (existing is not null)
        {
            log.LogInformation("Duplicate content {Hash}, skipping processing", hash);
            await blob.DeleteIfExistsAsync(cancellationToken: ct);   // don't store it twice
            job.ResultUrl = existing.ResultUrl;
            await UpdateAsync(job, JobStatus.Completed, 100, ct);
            return;
        }

        // 3. New content, so do the heavy 10-minute work
        var resultUrl = await processor.ProcessAsync(blob,
            onProgress: p => UpdateAsync(job, JobStatus.Processing, p, ct), ct);

        db.MediaFiles.Add(new MediaFile { ContentHash = hash, ResultUrl = resultUrl });
        job.ResultUrl = resultUrl;
        await UpdateAsync(job, JobStatus.Completed, 100, ct);
        // An exception here is not caught: Service Bus retries, then dead-letters
    }

    private async Task UpdateAsync(MediaJob job, JobStatus status, int progress, CancellationToken ct)
    {
        job.Status = status;
        job.Progress = progress;
        await db.SaveChangesAsync(ct);
        await notifier.NotifyAsync(job.Id, status.ToString(), progress, ct);   // SignalR push
    }
}
```

Say:
- *"Hashing streams the file in chunks, so memory stays flat even for 2 GB."*
- *"If the hash already exists, I skip storage and processing entirely. That saves cost, and the same input always gives the same output, which makes the operation idempotent."*
- *"Two identical files processed at the same moment both miss the check. The unique index on ContentHash catches that race, and the second one is treated as a duplicate."*
- *"Several worker instances read from the same queue. That's the Competing Consumers pattern: scale out by adding workers."*

host.json, because a 10-minute job needs its message lock renewed:

```json
{
  "version": "2.0",
  "functionTimeout": "00:30:00",
  "extensions": {
    "serviceBus": {
      "maxConcurrentCalls": 4,
      "maxAutoLockRenewalDuration": "00:30:00"
    }
  }
}
```

**Step 5a – Push updates with SignalR (instead of polling)**

```csharp
// API: hub
public class JobsHub : Hub
{
    public Task Watch(string jobId) => Groups.AddToGroupAsync(Context.ConnectionId, jobId);
}

// Program.cs
builder.Services.AddSignalR().AddAzureSignalR();   // Azure SignalR Service for scale
app.MapHub<JobsHub>("/hubs/jobs");

// Worker: notifier (Azure SignalR Management SDK, because the worker is a separate process)
public class SignalRJobNotifier(ServiceHubContext hub) : IJobNotifier
{
    public Task NotifyAsync(Guid jobId, string status, int progress, CancellationToken ct) =>
        hub.Clients.Group(jobId.ToString())
           .SendAsync("jobUpdated", new { jobId, status, progress }, ct);
}
```

**Step 5b – Angular client: upload, submit, then poll or listen**

```ts
// npm i @azure/storage-blob @microsoft/signalr
@Injectable({ providedIn: 'root' })
export class MediaJobService {
  private http = inject(HttpClient);
  private api = `${environment.apiUrl}/jobs`;

  create(fileName: string) {
    return this.http.post<{ jobId: string; uploadUrl: string }>(this.api, { fileName });
  }

  // 2 GB direct to Blob: SDK splits into blocks and uploads them in parallel
  async uploadToBlob(sasUrl: string, file: File, onProgress: (p: number) => void) {
    const client = new BlockBlobClient(sasUrl);
    await client.uploadData(file, {
      blockSize: 8 * 1024 * 1024,          // 8 MB blocks
      concurrency: 4,
      blobHTTPHeaders: { blobContentType: file.type },
      onProgress: e => onProgress(Math.round(100 * e.loadedBytes / file.size))
    });
  }

  submit(jobId: string) {
    return this.http.post<{ statusUrl: string }>(`${this.api}/${jobId}/submit`, {});
  }

  // Polling: every 5 seconds until done
  pollStatus(jobId: string) {
    return timer(0, 5000).pipe(
      switchMap(() => this.http.get<JobStatus>(`${this.api}/${jobId}/status`)),
      takeWhile(s => s.status !== 'Completed' && s.status !== 'Failed', true)  // true = emit the final one
    );
  }
}
```

```ts
// component
async start(file: File) {
  const { jobId, uploadUrl } = await firstValueFrom(this.svc.create(file.name));

  await this.svc.uploadToBlob(uploadUrl, file, p => this.uploadProgress.set(p));

  await firstValueFrom(this.svc.submit(jobId));

  this.svc.pollStatus(jobId)
    .pipe(takeUntilDestroyed(this.destroyRef))
    .subscribe(s => {
      this.status.set(s.status);
      this.processProgress.set(s.progress);
      if (s.status === 'Completed') this.resultUrl.set(s.resultUrl);
    });
}
```

SignalR alternative, which gives instant updates with no repeated requests:

```ts
const conn = new HubConnectionBuilder().withUrl('/hubs/jobs').withAutomaticReconnect().build();
conn.on('jobUpdated', u => { this.status.set(u.status); this.processProgress.set(u.progress); });
await conn.start();
await conn.invoke('Watch', jobId);
```

**Failure handling (interviewers always ask):**

| Problem | Answer |
|---|---|
| Worker crashes mid-job | The message lock expires, Service Bus redelivers it, and the idempotency check makes that safe |
| Fails every time (corrupt file) | After `MaxDeliveryCount` (e.g. 5) it moves to the **dead-letter queue**. A DLQ handler marks the job Failed and alerts |
| Job longer than 10 min | Consumption-plan Functions have a time limit, so use Premium/Flex Consumption, **Container Apps Jobs**, or AKS with KEDA scaling on queue length |
| Client closes the browser | No problem. The job keeps running and the user checks status later from a "My jobs" page |
| Upload left incomplete | Uncommitted blocks are auto-discarded after 7 days. A cleanup job removes `Pending` jobs older than 24h |
| Queue backlog | KEDA or Functions scale out workers based on queue depth (Competing Consumers) |

**Patterns in one line each:**
- **Valet Key:** a SAS token lets the client upload directly, so the API isn't in the data path.
- **Async Request-Reply:** 202 + status URL + polling or push.
- **Claim Check:** the message carries a reference, the payload stays in Blob.
- **Competing Consumers:** many workers on one queue.
- **Idempotent Consumer:** status check + content hash + duplicate detection.

**One-line close:** *"The API stays fast and stateless, the heavy work scales independently, every step is retry-safe,
and de-duplication means we never pay twice for the same content."*


**The problem first**

A user uploads a 2 GB video, and processing it takes 10 minutes. If the user's browser waits for one HTTP call to finish, the call times out (usually after 30–100 seconds) and everything fails. So we split the work into small steps that each finish quickly.

**A simple analogy: a dry cleaner**

You drop off your clothes, get a **token slip**, and go home. The shop cleans them in the back. You either call to ask "is it ready?" or they **text you** when it's done. Nobody stands at the counter for 2 days. Our system works the same way.

**Step-by-step**

**Step 1 – The user says "I want to upload a video"**
The browser tells the API: *"I have a file called holiday.mp4."*
The API does two quick things:
- It creates a **job record** in the database with status "Pending". This is the token slip.
- It creates a **temporary upload pass** (SAS URL) that allows *only this one file* to be written to Blob Storage, for about 1 hour.

It sends back the job ID and the upload pass. This takes milliseconds.

**Step 2 – The browser uploads the video directly to Azure Blob Storage**
The browser uses the upload pass to send the 2 GB file **straight to Blob Storage**, not through our API.
- The file is cut into small pieces (say 8 MB each), and several pieces upload at the same time.
- The user sees an upload progress bar.

Why? Our API server never has to hold or carry 2 GB. It stays free and fast for other users.

**Step 3 – The browser says "Upload done, please process it"**
The browser tells the API: *"Job 123 is uploaded."*
The API:
- Puts a **small note** on a queue (Azure Service Bus). The note only says: *"Job 123, file is at this location."* The video itself is not in the note.
- Changes the job status to "Queued".
- Replies right away with **202 Accepted**: *"Got it, we're working on it. Check here for updates: /jobs/123/status."*

The user is no longer waiting on a long call. This is the "Async Request-Reply" idea: accept now, reply later.
The small note pointing to the big file is the "Claim Check" idea, like a luggage tag instead of carrying the bag.

**Step 4 – A worker picks up the note**
A separate background program (an Azure Function or a container) is always watching the queue. It picks up the note and starts working.
- If 100 users upload at once, we simply run more workers, and each picks up its own note. That's "Competing Consumers".
- The worker updates the job status to "Processing".

**Step 5 – The worker checks "Have we seen this exact video before?"**
Before doing the expensive 10-minute processing, the worker reads the file and creates a **fingerprint** (a SHA-256 hash). The same content always gives the same fingerprint.
- **Fingerprint already in the database:** this exact video was processed before. The worker **skips processing**, deletes the duplicate copy, reuses the old result, and marks the job "Completed".
- **New fingerprint:** it does the real processing.

That's where the savings on storage and computing come from.

**Step 6 – The worker processes the video**
It does the heavy work (converting, compressing, making thumbnails, and so on). While it works, it keeps updating the job: 20%… 50%… 80%…
When it finishes, it saves the result, stores the fingerprint for the future, and marks the job "Completed".

**Step 7 – The user finds out it's done**
There are two ways:
- **Polling (simple):** every 5 seconds the browser asks the API, *"How is job 123?"* The API reads the job record and answers *"Processing, 60%"* and later *"Completed, here's the link."*
- **Push (better):** with SignalR, the server sends a message to the browser the moment something changes, like the dry cleaner texting you. There's no repeated asking.

**What if something goes wrong?**

- **The worker crashes halfway.** The note goes back on the queue automatically, and another worker picks it up. Because the worker first checks "is this job already done?", repeating it causes no harm. That's what **idempotent** means: doing it twice gives the same result as doing it once.
- **The file is broken and fails every time.** After about 5 tries, the note moves to a "dead-letter" queue, a box for problem items. The job is marked "Failed" and the team is alerted.
- **The user closes the browser.** Nothing breaks. The work continues in the background, and the user can check later.
- **The same note gets sent twice by mistake.** Service Bus spots the duplicate ID and ignores the extra one.

**The whole thing in five lines**

1. The **API** creates a job and gives out an upload pass.
2. The **browser** uploads the big file directly to Blob Storage.
3. The **API** puts a small note on a queue and says "accepted, check back later".
4. A **worker** fingerprints the file, skips it if it's a duplicate, and otherwise processes it.
5. The **user** sees progress through polling or a push message.

**Why this design is good (your closing line)**

*"The API never waits on or carries the heavy work, so it stays fast. Workers scale on their own. Every step is safe to retry. And de-duplication means we never pay twice for the same file."*
