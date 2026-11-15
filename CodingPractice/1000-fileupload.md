It's the same design with one extra layer: a **Batch** that groups the 1000 files. Each file is still its own small job. The batch just tracks the overall progress.

**The idea in plain English**

Think of 1000 parcels at a courier. You get **one batch receipt** ("Batch 55: 1000 parcels"). Every parcel still gets **its own tracking number**, so if 3 parcels get damaged, only those 3 are redone, not all 1000. You check one screen: *"Batch 55: 970 delivered, 27 in transit, 3 failed."*

**Step by step**

**Step 1 – The client asks for a batch**
The browser says: *"I want to upload 1000 files"* and sends the list of names and sizes.
The API:
- Creates **1 Batch** record (Total = 1000).
- Creates **1000 Job** records, one per file, all "Pending".
- Gives the upload permission. You don't want 1000 separate SAS URLs, so give **one SAS for the batch folder** (`batch-55/`) with write-only access for 2 hours.

**Step 2 – The browser uploads files in parallel, but limited**
Don't upload all 1000 at once, or the browser and network will choke. Upload **5–10 files at a time**. When one finishes, start the next, like a queue at 10 counters.
- Each file is still split into blocks.
- The user sees *"Uploaded 430 / 1000"*.
- If one upload fails, retry just that file.

**Step 3 – Processing starts automatically per file**
Instead of the browser calling "submit" 1000 times, use **Event Grid**: every time a file lands in Blob Storage, Azure automatically fires a "BlobCreated" event, and that puts a note on the Service Bus queue for that one file.
So processing of file 1 starts while file 500 is still uploading. There's no waiting for all 1000.

**Step 4 – Many workers process files in parallel**
1000 notes are now in the queue.
- **KEDA** (on AKS or Container Apps) or Functions scaling sees *"queue is long"* and starts more workers, say 20.
- Each worker takes one note at a time: Competing Consumers.
- When the queue empties, workers scale back down, so you pay only while working.

Set a **max worker limit**. Otherwise 200 workers could overload the database or the media processing service.

**Step 5 – De-duplication saves even more at this scale**
In a batch of 1000, duplicates are common (the same video uploaded twice, the same logo in many folders). Each worker fingerprints its file, and **if the hash already exists, it skips processing**.
Two workers can get identical files at the same moment. The **unique index on the hash** lets only one win, and the other reuses that result.

**Step 6 – Track batch progress**
Each time a file job finishes, the worker updates the Batch counters: **Completed +1** or **Failed +1**.
When Completed + Failed = Total, the batch is done.
- The client polls **one URL**: `/batches/55/status` → *"970 done, 27 processing, 3 failed"*
- Or SignalR pushes the counts, throttled to about once per second, not 1000 messages.

**Step 7 – Handle partial failure**
3 files failed? The batch finishes as **"Completed with errors"**.
- Show the user exactly which 3 failed and why.
- Offer a **"Retry failed"** button that re-queues just those 3.
- Never re-process the 997 that succeeded.

**Step 8 – Final step when everything is done (fan-in)**
If you need something after *all* files finish (a summary report, a zip, an email), use **Durable Functions fan-out / fan-in**:
- **Fan-out:** start 1000 activity tasks in parallel.
- **Fan-in:** wait for all of them, then run the final step once.

**Key code**

Batch + jobs:

```csharp
public class MediaBatch
{
    public Guid Id { get; set; }
    public int Total { get; set; }
    public int Completed { get; set; }
    public int Failed { get; set; }
    public string Status { get; set; } = "Uploading";   // Uploading / Processing / Done / DoneWithErrors
}
// MediaJob gets: public Guid BatchId { get; set; }
```

Updating counters safely (many workers at once), with one atomic SQL statement and no read-then-write race:

```csharp
await db.MediaBatches
    .Where(b => b.Id == job.BatchId)
    .ExecuteUpdateAsync(s => s.SetProperty(b => b.Completed, b => b.Completed + 1), ct);
```

Angular parallel upload with a limit of 6 at a time:

```ts
from(files).pipe(
  mergeMap(file => this.uploadOne(batchSasUrl, file).pipe(
      retry(2),                                   // retry this file only
      catchError(() => of({ file, failed: true }))
  ), 6)                                           // ← max 6 uploads in parallel
).subscribe(r => this.uploaded.update(n => n + 1));
```

Durable Functions fan-out / fan-in:

```csharp
[Function(nameof(BatchOrchestrator))]
public async Task<string> BatchOrchestrator([OrchestrationTrigger] TaskOrchestrationContext ctx)
{
    var batch = ctx.GetInput<BatchInput>()!;

    // Fan-out: one activity per file, all in parallel
    var tasks = batch.BlobNames
        .Select(name => ctx.CallActivityAsync<FileResult>(nameof(ProcessFile), name))
        .ToList();

    // Fan-in: wait for all
    var results = await Task.WhenAll(tasks);

    await ctx.CallActivityAsync(nameof(SendSummary), results);   // runs once at the end
    return $"{results.Count(r => r.Ok)} ok, {results.Count(r => !r.Ok)} failed";
}
```

**Single file vs 1000 files**

| | 1 file | 1000 files |
|---|---|---|
| Tracking | 1 Job | 1 Batch + 1000 Jobs |
| Upload pass | SAS for 1 blob | 1 SAS for the batch folder |
| Upload | Blocks in parallel | Files in parallel too, max 5–10 |
| Start processing | Client calls submit | Event Grid, automatic per file |
| Workers | 1 | Auto-scale with KEDA, with a max limit |
| Status | Job status | Batch counters (done/failed/total) |
| Failure | Retry the job | Retry only the failed files |
| Final step | None | Fan-in (Durable Functions) |

**Closing line for the interview:**
*"Each file is an independent, idempotent job, so 1 file or 10,000 files is the same code. The batch only adds tracking. Processing starts as soon as each file lands, workers scale with the queue length up to a safe limit, de-duplication skips repeated content, and failures are retried per file, never per batch."*
