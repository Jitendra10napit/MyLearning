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

I can turn this into a one-page cheat sheet you can keep, or run a mock interview where I ask you these follow-up questions.
