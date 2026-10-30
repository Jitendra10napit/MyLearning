I read all six diagrams in the folder. Between them they describe the whole NICE Investigate platform, so the strongest way to present them is as **one system walked through a single investigator's journey**, not six separate diagrams.

## The 30-second opener

"NICE Investigate is a multi-tenant digital evidence platform on Azure. Investigators use an Angular SPA that talks only to an ASP.NET Core **BFF**. Behind it are about 15 .NET microservices for cases, evidence, search, media, tasks and notifications. Synchronous calls go over HTTPS with service-to-service tokens. The heavy work, like media processing, indexing and notifications, is event-driven over **Azure Service Bus** and **Durable Functions**. Search runs on Elasticsearch, and a central policy service controls who can see what."

Then offer to go deeper: "I can walk through one user's journey: logging in, what they're allowed to see, searching, uploading evidence, and getting notified."

## Walkthrough order

**1. Big picture (ip-service-consumption)**
- The browser only ever talks to the Portal BFF.
- The BFF fans out to downstream services through typed clients (`AddClient<IXxxService>`). Each client has a `TokenCredential` for authentication and resilience policies (retry, timeouts).
- The services fall into four domains: Identity & Access, Cases/Evidence/Search, Files/Media/Sharing, and Productivity & AI.
- **Why:** there's one entry point, the SPA stays simple, and cross-cutting concerns like auth and resilience live in one place.

**2. Login (flow-login-oidc)**
- The user opens the portal with no session, so the BFF redirects to the TokenService (Duende IdentityServer).
- The user logs in with credentials and MFA, and the auth code comes back to `/signin-oidc`.
- **The BFF swaps the code for tokens on the server.** The browser only gets a session cookie.
- For downstream calls, the BFF gets its own app-to-app token, with the audience set to that service's App URI.
- **Key point:** tokens never reach the browser, which shrinks the XSS attack surface. That's the main reason for the BFF pattern.

**3. Authorization (flow-authz-pdp-pep)**
- This uses the classic **PEP/PDP split**. The BFF hosts the PEP (the PrivilegeManager library). It sends a `PrivilegeTreeRequest` to the AdjudicationService (the PDP).
- The PDP gets the user's roles from UserService and the resource metadata from Elasticsearch. Domain-specific adjudicators apply the policy and return allow or deny for each resource.
- The PEP enforces the decision: it either returns the data or a 403.
- **Why:** each service makes its own access check today. With one policy service, the rules live in one place and every service applies them the same way.

**4. Search (flow-search)**
- The BFF calls QueryService, which first asks Adjudication for the IDs this user may see. Only then does it query the tenant's index (`investigate-{tenant}-{type}`) with bool filters and aggregations. Type-ahead uses completion/prefix queries.
- **Key points:**
  - **Results are filtered before the query runs, not afterwards.** That way facet counts can't leak information about evidence the user can't see.
  - Separate indexes per tenant keep tenants' data isolated.

**5. Evidence upload → processing → indexing (flow-upload-media-index)**
This is your strongest diagram, so spend the most time here.
- The file is uploaded in chunks to FileUploadService, then stored in Blob along with its metadata. CaseAndEvidence links it to an evidence item.
- A `ProcessMediaFile` message goes onto a Service Bus topic, which triggers the MediaWorkflowService (Durable Functions). It runs virus scan, transcode and thumbnail jobs, and tracks their status by webhook or polling.
- If a virus is found, the file is quarantined and the evidence is blocked. If it's clean, the evidence is marked ready and an `IndexDbItemCommand` goes to the Indexer Functions, which upsert it into Elasticsearch. From then on it's searchable.
- **Key points:**
  - Long-running work is asynchronous and durable, so it survives restarts and retries.
  - Results are eventually consistent: users see the evidence once indexing finishes.
  - The virus-scan gate decides the outcome.
- **Bring in your own work here.** Your content-hash de-duplication fits right before `ProcessMediaFile`: a duplicate file skips transcoding and storage, which is where the cost savings came from. Your Durable Functions buffer service for case re-sharing uses the same orchestration pattern. This turns "I know the system" into "I designed parts of it."

**6. Real-time notifications (flow-notification-signalr)**
- A domain event, such as evidence being shared, reaches NotificationService. It **saves the notification as unread first**, then publishes a message per user to a queue.
- A hosted background service in the Portal (UserNotificationWatcher, with retries) picks it up and sends it through the SignalR hub to `Clients.Group(userId)`. The browser updates the bell count and shows a toast. If the connection drops, it reconnects and rejoins its group automatically.
- **Key points:**
  - The database is the source of truth and the push is best-effort, so a user who was offline still sees the notification later.
  - Using one group per user covers multiple tabs and devices.
  - Azure SignalR acts as the backplane. Only one Portal instance takes each queue message, but the user's connection may be on a different instance, and the backplane delivers it there.

## Follow-up questions to prepare for
- **"Once the BFF uses an app token, how does the downstream service know which user it's acting for?"** Answer with what NICE actually does: a user ID passed in a header or claim, the on-behalf-of flow, or trusting the BFF's own authorization check.
- **"What if processing fails halfway through?"** Cover Durable Functions replay, idempotent handlers, and dead-letter queues.
- **"How do you stop duplicate indexing or duplicate notifications?"** Cover idempotency keys and upserts.
- **"Why Elasticsearch and not SQL full-text search?"** Facets, relevance ranking, suggestions, and scale.
- **"What would you change?"** Have one honest improvement ready, for example caching authorization decisions, or tracing that follows a request through the asynchronous steps.

Tell the story in that order (login, access, search, upload, notify), and stop after each part so the interviewer can choose where to go deeper. That keeps it a conversation rather than a monologue.



Here's the whole system walked through one made-up case, with example values at each step. Telling it as a story is much easier for an interviewer to follow than listing services.

**The case:** Ravi, an investigator in the tenant *pune-police*, uploads a CCTV clip to case **CASE-2041 (chain snatching, FC Road)**. He shares it with his colleague Anita, who gets notified and finds it by searching. Sameer, from a different unit, searches the same words and finds nothing. All IDs and values below are examples I made up.

## 1. Ravi logs in

```
Ravi opens  https://investigate.example.com
  → BFF finds no session → 302 to TokenService:
    /connect/authorize?client_id=investigator-portal&response_type=code&scope=openid profile
  → Ravi enters password + MFA
  → TokenService returns to the BFF: /signin-oidc?code=Xy12...
  → BFF (server side) exchanges the code → id_token { sub: "u-101", tenant: "pune-police" }
  → Browser receives only:  Set-Cookie: .Investigate.Session; HttpOnly; Secure; SameSite=Strict
```

**What to say:** "Ravi's browser never holds a JWT, just an HttpOnly cookie. If a malicious script got onto the page, there's no token for it to steal."

When the BFF calls CaseAndEvidence, it uses its own app token:

```csharp
services.AddClient<ICaseAndEvidenceService>(o =>
{
    o.BaseAddress = config["CaseEvidence:Url"];
    o.Audience    = "api://case-evidence";   // token is scoped to this one service
})
.AddCredential(new DefaultAzureCredential())  // managed identity, no secrets
.AddStandardResilienceHandler();              // retry, timeout, circuit breaker
```

## 2. Ravi opens CASE-2041 (authorization)

The PEP in the BFF asks the PDP for a decision:

```json
POST /privileges
{ "userId": "u-101", "resources": [ { "type": "case", "id": "CASE-2041" } ] }

← { "CASE-2041": { "allowed": true, "privileges": ["View", "Upload", "Share"] } }
```

The AdjudicationService looked up Ravi's roles (UserService) and the case metadata (Elasticsearch), then applied the policy: Ravi is assigned to this case, so he gets View, Upload and Share.

When Sameer tries the same URL, the result is `"allowed": false`, the PEP returns **403**, and the case data never leaves the server.

**What to say:** "Services don't each write their own permission checks. They all ask one policy service, so 'who can see CASE-2041' has a single answer everywhere."

## 3. Ravi uploads cctv_fcroad.mp4 (800 MB)

1. **Chunked upload:** the file goes up in 100 chunks of 8 MB, so a network drop only retries one chunk.
   Blob path: `pune-police/evidence/EV-9001/original.mp4`
2. **De-duplication (your work):** compute the SHA-256 content hash. If the same file is already stored, link to the existing blob and skip transcoding. The cost saving comes from skipping both the storage and the processing.
3. **Event published** to the Service Bus topic `investigateevents`:
   ```json
   { "type": "ProcessMediaFile", "tenant": "pune-police",
     "evidenceId": "EV-9001", "blobUri": ".../EV-9001/original.mp4" }
   ```
4. **The Durable Functions orchestration** runs the jobs:
   ```csharp
   var scan = await ctx.CallActivityAsync<ScanResult>("VirusScan", file);
   if (scan.Infected) { await ctx.CallActivityAsync("Quarantine", file); return; }

   await Task.WhenAll(                                   // fan-out in parallel
       ctx.CallActivityAsync("Transcode", file),         // → 720p mp4
       ctx.CallActivityAsync("Thumbnail", file));        // → poster.jpg

   await ctx.CallActivityAsync("MarkReady", file);       // EV-9001 state = Ready
   await ctx.CallActivityAsync("SendIndexCommand", file);
   ```
5. **Indexing:** an Indexer Function upserts the evidence into `investigate-pune-police-evidence`:
   ```json
   { "id": "EV-9001", "caseId": "CASE-2041", "type": "video",
     "title": "CCTV FC Road junction", "tags": ["FC Road", "chain snatching"] }
   ```

What Ravi sees: the evidence shows **"Processing…"**, then about 4 minutes later it switches to **"Ready"** with a thumbnail. If the scan had found a virus, it would show **"Blocked"** and nobody could download it.

**What to say:** "Transcoding a large video can take minutes, so it can't run inside an HTTP request. Durable Functions saves each step as it completes. If the host restarts after the virus scan, the workflow resumes at transcoding instead of starting over."

## 4. Ravi shares EV-9001 with Anita (notification)

```
CaseAndEvidence → domain event "EvidenceShared"
NotificationService → INSERT Notification
     (Id: N-55, UserId: u-205, Text: "Ravi shared EV-9001", IsRead: false)   ← saved FIRST
                    → publish to WebsiteNotificationQueue { userId: "u-205", id: "N-55" }
Portal (UserNotificationWatcher, with retries) → NotificationHandler
     → Clients.Group("u-205").SendAsync("newMessage", item)
Azure SignalR → reaches Anita's laptop tab AND phone tab → bell 3 → 4, toast appears
```

- **If Anita is offline:** nothing is lost. At her next login, the unread count comes from the database (N-55, IsRead = false).
- **Why Azure SignalR:** say there are 3 Portal instances. Instance A takes the queue message, but Anita's websocket is connected to instance C. The backplane delivers the message across instances.

## 5. Anita and Sameer both search "FC Road"

```
GET /search?q=FC Road&filters=type:video
QueryService → Adjudication: "what may u-205 see?" → allowed: [CASE-2041, EV-9001, ...]
```

Elasticsearch then gets a query that is already restricted to those IDs:

```json
GET investigate-pune-police-evidence/_search
{ "query": { "bool": {
    "must":   [ { "match": { "title": "FC Road" } } ],
    "filter": [ { "terms": { "id": ["EV-9001", "..."] } },
                { "term":  { "type": "video" } } ] } },
  "aggs": { "byType": { "terms": { "field": "type" } } } }
```

| | Anita | Sameer |
|---|---|---|
| Results | EV-9001 CCTV FC Road junction | 0 results |
| Facet | Video (1) | nothing |

**What to say:** "We filter **before** querying, not afterwards. Otherwise Sameer could see a facet like 'Video (1)' and learn that the evidence exists even though he can't open it. Each tenant also has its own index, so pune-police data can never show up in another tenant's search."

## How to say it in the interview (about 90 seconds)

> "Take one investigator, Ravi. He logs in through our BFF using OIDC against IdentityServer. The browser only gets an HttpOnly cookie, and the BFF holds the tokens. When he opens a case, the BFF asks our central policy service whether he's allowed. When he uploads an 800 MB CCTV clip, it goes up in chunks to Blob storage. We hash the content to skip duplicates, then publish an event to Service Bus. A Durable Functions workflow runs the virus scan, then transcodes and creates the thumbnail in parallel, marks the evidence ready and indexes it in Elasticsearch. When he shares it with Anita, the notification is saved first and then pushed over SignalR, so it shows up on her laptop and phone instantly, and it's still there if she was offline. When she searches, results are filtered to what she's allowed to see before the query runs, so another officer searching the same words gets nothing, not even a count."

Then stop and let them choose which part to go deeper on. Each part above is ready to expand.

I can run a mock round now: I ask the follow-up questions, you answer, and I give you feedback.



In this system, the identity server is **TokenService**, which is built on **Duende IdentityServer**. It's the login authority: it signs users in and issues the tokens that prove who they are.

## Who does what in the login flow

| OAuth/OIDC role | In NICE Investigate | What it does |
|---|---|---|
| **User** | Ravi, the investigator | Types his password and does MFA |
| **Identity provider / authorization server** | **TokenService (Duende IdentityServer)** | Shows the login page, checks credentials and MFA, and issues the id_token and access_token |
| **Client** | **Portal BFF** (ASP.NET Core) | Redirects users to TokenService, receives the auth code, exchanges it for tokens, and keeps them on the server |
| **Resource servers / APIs** | CaseAndEvidence, QueryService, MediaService, and so on | Accept requests only when they carry a valid Bearer token |

In Ravi's example:

1. Ravi opens the portal. The BFF sees he isn't logged in and sends him to TokenService.
2. **TokenService** shows the login screen, checks his password and MFA, and confirms he's user `u-101` in tenant `pune-police`.
3. It sends a one-time code back to the BFF. The BFF exchanges that code with TokenService for the tokens.
4. From then on, the BFF keeps the tokens and Ravi's browser just has the session cookie.

Think of it like an office building. TokenService is the reception desk that checks your ID and issues a visitor badge. The BFF is the escort who holds the badge for you. The services are the locked rooms, and they only open for a valid badge.

## A detail worth getting right

The diagrams suggest **two token issuers**, not one:

- **User login** goes through **TokenService (Duende IdentityServer)**, using OIDC with the authorization code flow.
- **Service-to-service calls** (the BFF calling CaseAndEvidence, for example) use `TokenCredential` with `audience = service AzureAppUri`. That's the Azure.Identity pattern, so those tokens most likely come from **Microsoft Entra ID**, probably through a managed identity, not from IdentityServer.

Check this against what NICE actually runs before you say it in an interview. If it holds, here's how to say it:

> "Users authenticate against our own TokenService, built on Duende IdentityServer, which gives us control over login, MFA and tenant claims. For service-to-service calls we use Entra ID managed identities, so there are no secrets in config and each API only accepts tokens issued for its own App URI."

## Likely follow-up: "Why run your own IdentityServer instead of using Entra ID for users too?"

Good reasons to give, picking the ones that actually applied at NICE:
- It's **multi-tenant**: each police or agency customer can plug in their own login (their own AD, SAML or another OIDC provider) behind one consistent token format.
- You can add **custom claims** like tenant and roles that the authorization service relies on.
- It works for **on-premise and government** customers who can't depend on a public cloud identity provider.

I can turn this into a one-page cheat sheet you can keep, or run a mock interview where I ask you these follow-up questions.
