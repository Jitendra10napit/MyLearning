# Senior Technical Lead / Solution Architect

## 1–2 Day Interview Revision Notes

These notes are designed for **quick memorization + interview speaking**, using your actual project concepts wherever possible.

---

# 1. YOUR PROJECT ARCHITECTURE

## 1.1 30-second explanation

> "The application is a multi-tenant public-safety case and digital-evidence platform. The frontend is built using Vue, with BFFs exposing frontend-specific APIs. The backend consists of .NET services following layered/clean architectural principles. SQL is the transactional source of truth, Elasticsearch is used for search and read optimization, Azure Service Bus provides asynchronous event-driven communication, and Azure Functions handle background processing. We also use Redis for caching, SignalR for real-time notifications, and centralized authorization for secure access."

### Architecture

```text
                    USER
                     │
                     ▼
               ┌──────────┐
               │  Vue UI  │
               └────┬─────┘
                    │
                    ▼
               ┌──────────┐
               │   BFF    │
               └────┬─────┘
                    │
                    ▼
             ┌──────────────┐
             │ ASP.NET Core │
             │     API      │
             └──────┬───────┘
                    │
             ┌──────┴──────┐
             ▼             ▼
        COMMAND          QUERY
             │             │
             ▼             ▼
            SQL            ES
             │             ▲
             │             │
             └── EVENT ────┘
                    │
                    ▼
            Azure Service Bus
              /     |      \
             ▼      ▼       ▼
          Workflow Indexer Notification
             │
             ▼
       Azure Functions
```

### Remember

> **SQL = Truth**
> **Elasticsearch = Search**
> **Redis = Cache**
> **Service Bus = Async communication**
> **Functions = Background processing**

---

# 2. HOW TO EXPLAIN YOUR ARCHITECTURE

Whenever interviewer says:

### "Explain your current project architecture."

Use this order:

```text
Business
   ↓
Frontend
   ↓
API/BFF
   ↓
Application
   ↓
Domain
   ↓
Infrastructure
   ↓
Database / External Systems
   ↓
Async Events
   ↓
Background Processing
```

Then explain why each exists.

### Strong answer

> "We separated responsibilities so that the presentation layer doesn't directly depend on infrastructure. The API handles the request boundary, the application layer handles use cases, the domain contains business rules, and infrastructure handles external concerns such as SQL, Elasticsearch, Service Bus and other integrations."

---

# 3. CLEAN ARCHITECTURE

## Structure

```text
Presentation
     ↓
Application
     ↓
Domain
     ↑
Infrastructure
```

The most important principle:

> **Dependencies point inward.**

Example:

```csharp
public interface ICaseRepository
{
    Task<Case?> GetAsync(Guid id);
}
```

Application depends on:

```text
ICaseRepository
```

Infrastructure implements:

```csharp
public class CaseRepository : ICaseRepository
{
}
```

### Why?

Because application/domain should not care whether data comes from:

```text
SQL
MongoDB
API
Mock
```

### Interview answer

> "Clean Architecture reduces coupling by keeping business logic independent from infrastructure."

---

# 4. LAYERED vs CLEAN vs HEXAGONAL

| Architecture     | Main idea                      |
| ---------------- | ------------------------------ |
| Layered          | Separate technical layers      |
| Clean            | Dependencies point inward      |
| Hexagonal        | Ports and adapters             |
| Onion            | Domain at center               |
| Microservices    | Business/service boundaries    |
| Modular Monolith | Strong modules, one deployment |

### Easy memory

```text
Layered
→ Layers

Clean
→ Dependencies inward

Hexagonal
→ Ports + Adapters

Microservices
→ Independent services

Modular Monolith
→ Modules + one deployment
```

---

# 5. CQRS

## What is CQRS?

> **Command Query Responsibility Segregation**

Separate:

```text
WRITE
  ↓
COMMAND

READ
  ↓
QUERY
```

### Architecture

```text
                 API
                  │
          ┌───────┴────────┐
          ▼                ▼
       COMMAND            QUERY
          │                │
          ▼                ▼
      CommandHandler   QueryHandler
          │                │
          ▼                ▼
         SQL              ES
```

### Command example

```csharp
public record CreateCaseCommand(
    string Name);
```

Handler:

```csharp
public class CreateCaseHandler
{
    public async Task Handle(CreateCaseCommand command)
    {
        // validate
        // apply business rules
        // save SQL
    }
}
```

Query:

```csharp
public record SearchCasesQuery(
    string SearchText);
```

### Important interview correction

CQRS does **not** require:

* MediatR
* Separate databases
* Microservices
* Event sourcing

### Say this

> "CQRS separates read and write responsibilities. Separate read/write databases are an optimization, not a requirement."

---

# 6. HOW CQRS HANDLER IS CALLED

## Option 1 — Mediator

```text
HTTP Request
     ↓
Controller
     ↓
Mediator.Send()
     ↓
Handler
     ↓
Application
     ↓
DB
```

Example:

```csharp
[HttpPost]
public async Task<IActionResult> Create(
    CreateCaseCommand command)
{
    await _mediator.Send(command);

    return Ok();
}
```

Mediator finds:

```text
CreateCaseCommand
       ↓
IRequestHandler<CreateCaseCommand>
```

---

# 7. CQRS WITHOUT MEDIATOR

You can directly inject the handler.

```text
Controller
    ↓
ICommandHandler
    ↓
CreateCaseHandler
    ↓
Domain
    ↓
Repository
    ↓
SQL
```

Example:

```csharp
public interface ICommandHandler<TCommand>
{
    Task Handle(TCommand command);
}
```

Controller:

```csharp
private readonly ICommandHandler<CreateCaseCommand> _handler;

public async Task<IActionResult> Create(
    CreateCaseCommand command)
{
    await _handler.Handle(command);

    return Ok();
}
```

### Interview gold

> **"MediatR is a dispatching mechanism. CQRS is the design approach."**

---

# 8. CQRS + ELASTICSEARCH

This is one of your strongest project examples.

```text
             WRITE
               │
               ▼
             SQL
          Source of Truth
               │
               ▼
             Event
               │
               ▼
       Azure Service Bus
               │
               ▼
          Indexer
               │
               ▼
       Elasticsearch
               ▲
               │
              READ
```

### Why?

SQL is optimized for transactional consistency.

Elasticsearch is optimized for:

* Search
* Filtering
* Full text
* Aggregation
* Large read workloads

### Trade-off

You introduce:

> **Eventual consistency**

For a short time:

```text
SQL = New
ES  = Old
```

Then ES catches up.

---

# 9. CQRS ≠ EVENT SOURCING

Remember this.

```text
CQRS
→ Separate READ and WRITE

Event Sourcing
→ Store state changes as EVENTS
```

They can be combined.

But:

> CQRS does not mean Event Sourcing.

---

# 10. EVENT-DRIVEN ARCHITECTURE

Your platform uses event-driven communication.

```text
Service A
   │
   │ Something happened
   ▼
Azure Service Bus
   │
   ├── Workflow
   ├── Indexer
   ├── Notification
   └── Audit
```

Producer says:

> **"What happened."**

Consumers decide:

> **"What should I do?"**

### Example

```text
CaseReadyForDisclosureEvent
```

Consumers:

```text
Workflow
 → Create Share

Indexer
 → Update ES

Notification
 → Notify user

Audit
 → Record audit
```

---

# 11. QUEUE vs TOPIC

## Queue

```text
Producer
   ↓
Queue
   ↓
Consumer
```

One message is normally processed by one competing consumer.

Use for:

> **Work**

Example:

```text
ProcessVideo
```

---

## Topic

```text
              Topic
             /  |  \
            /   |   \
        Audit  Search Notify
```

One event can be consumed independently by multiple subscriptions.

Use for:

> **Events**

### Memorize

> **Queue = Work**
> **Topic = Event**

---

# 12. SERVICE BUS — WHAT CAN GO WRONG?

Whenever discussing Service Bus, mention:

```text
Duplicate messages
        ↓
Idempotency

Temporary failure
        ↓
Retry

Permanent failure
        ↓
DLQ

Ordering requirement
        ↓
Session / design consideration

Monitoring
        ↓
Correlation ID + metrics
```

This makes your answer senior-level.

---

# 13. IDEMPOTENCY

Suppose:

```text
MessageId = 123
```

arrives twice.

Bad:

```text
123 → Create Share
123 → Create Share
```

Good:

```text
123
 ↓
Already processed?
 ↓
YES
 ↓
Ignore
```

### Interview answer

> "Messaging can provide at-least-once delivery, so I design consumers to be idempotent."

---

# 14. RETRY

For temporary/transient failures.

```text
API
 ↓
Failure
 ↓
Wait
 ↓
Retry
 ↓
Success
```

Use:

> Exponential backoff + jitter + maximum attempts.

Example:

```text
1 sec
2 sec
4 sec
8 sec
```

Don't blindly retry:

```text
400
401
403
```

because these are generally not transient failures.

---

# 15. CIRCUIT BREAKER

When dependency is continuously failing.

```text
       CLOSED
          │
       failures
          ▼
         OPEN
          │
        wait
          ▼
      HALF OPEN
       /     \
   success   failure
      │         │
      ▼         ▼
   CLOSED      OPEN
```

### Memory

> **Retry = Try again**
> **Circuit Breaker = Stop knocking**

---

# 16. SAGA

Distributed transaction.

Example:

```text
Create Case
    ↓
Reserve Evidence
    ↓
Create Share
    ↓
Notify
```

Each service has its own transaction.

If something fails:

```text
Compensating action
```

Example:

```text
Payment successful
Inventory successful
Shipment failed

       ↓

Refund Payment
```

### Two approaches

```text
Orchestration
→ Central coordinator

Choreography
→ Services react to events
```

---

# 17. YOUR WORKFLOW / R&A

This is an excellent interview example.

Memorize:

# Detect → Decide → Wait → Execute → Report

```text
Case Event
    ↓
Detect
    ↓
Workflow Rules Engine
    ↓
Decide
    ↓
Buffer Service
    ↓
Wait
    │
    ├── Cancel
    │
    └── Execute
           ↓
      Case & Evidence
           ↓
         Result
           ↓
        Report
```

Example:

```text
Case = Ready for Disclosure
             ↓
          Event
             ↓
       Rules Engine
             ↓
       Create Share
             ↓
        Wait 10 min
             ↓
     ┌───────┴───────┐
     ↓               ↓
  Cancel           Execute
                     ↓
                 Create Share
```

### Best explanation

> "The producer announces what happened; the workflow service decides what should happen."

---

# 18. AZURE FUNCTIONS

Common triggers:

```text
HTTP
Timer
Queue
Service Bus
Blob
Event Grid
```

Typical:

```text
Service Bus
    ↓
Azure Function
    ↓
Process
    ↓
Save / Publish
```

Use Functions when:

* Event-driven processing
* Background processing
* Scheduled work
* Lightweight serverless workloads

---

# 19. DURABLE FUNCTIONS

Use when workflow is:

* Long-running
* Stateful
* Multi-step
* Requires waiting
* Requires orchestration

```text
Orchestrator
   │
   ├── Activity 1
   │
   ├── Activity 2
   │
   ├── Wait
   │
   └── Activity 3
```

### Difference

```text
Function
→ Execute work

Durable Function
→ Orchestrate work
```

---

# 20. HANGFIRE

Use for:

* Background jobs
* Scheduled jobs
* Delayed jobs
* Recurring jobs

Your retention example:

```text
Retention Policy
       ↓
Hangfire
       ↓
Rules Engine
       ↓
Batch processing
       ↓
Delete / Archive / Unshare
```

### Memory

> **Hangfire = Job scheduling/background jobs**

---

# 21. REDIS

Redis is primarily used as a cache in your architecture.

## Cache Aside

```text
Application
     │
     ▼
   Redis?
   /   \
 YES    NO
  │      │
  ▼      ▼
Return   SQL
         │
         ▼
       Redis
```

Good candidates:

* Frequently accessed
* Expensive to calculate
* Relatively stable data

Problem:

> Cache invalidation / stale data.

---

# 22. ELASTICSEARCH

Remember:

> **Elasticsearch is not your transactional source of truth.**

Use for:

```text
Search
Filtering
Full-text
Aggregation
Suggestions
Read optimization
```

Architecture:

```text
SQL
 ↓
Event
 ↓
Service Bus
 ↓
Indexer
 ↓
Elasticsearch
```

Important production concerns:

* Retry
* Idempotency
* DLQ
* Reindexing
* Index versioning
* Eventual consistency
* Monitoring index lag

---

# 23. EF CORE PERFORMANCE

### Bad

```csharp
var users = db.Users
    .Include(x => x.Orders)
    .ToList();
```

Could retrieve much more data than required.

### Better

```csharp
var users = await db.Users
    .AsNoTracking()
    .Where(x => x.IsActive)
    .Select(x => new UserDto
    {
        Id = x.Id,
        Name = x.Name
    })
    .ToListAsync();
```

Remember:

```text
AsNoTracking
Projection
Pagination
Indexes
Execution Plan
Avoid N+1
Avoid unnecessary Include
Async queries
Batch operations
```

### Golden sentence

> **"Don't fetch data you don't need."**

---

# 24. PERFORMANCE — FULL SYSTEM

Never say only:

> "I optimized database queries."

Think:

```text
Vue
 ↓
BFF
 ↓
API
 ↓
Application
 ↓
SQL / ES / Redis
 ↓
External services
```

## UI

* Debounce
* Pagination
* Lazy loading
* Virtualization
* Code splitting
* Reduce API calls
* Reduce payload

## API

* Async
* Projection
* Pagination
* Compression
* Caching
* Validation

## DB

* Indexes
* Execution plan
* Projection
* AsNoTracking
* Avoid N+1

## Architecture

* Async messaging
* Elasticsearch
* Redis
* Background processing

### Formula

# Measure → Bottleneck → Optimize → Measure

---

# 25. ASYNC vs PARALLEL

## Async

Best for I/O:

```text
DB
HTTP
File
Network
```

```csharp
await httpClient.GetAsync(url);
```

The thread doesn't need to remain blocked while waiting.

## Parallel

Best when independent CPU/work items can execute concurrently.

```text
File 1 ─┐
File 2 ─┼→ Parallel
File 3 ─┤
File 4 ─┘
```

But use:

> **Bounded concurrency**

For CCTV:

```text
1000 segments
      ↓
Worker pool
      ↓
Max 4/8 concurrent
      ↓
Process
```

### Why?

Avoid:

* CPU exhaustion
* Memory pressure
* Disk contention
* Too many external requests

---

# 26. AUTHENTICATION

Your authentication flow:

```text
Browser
   │
   ▼
OIDC Authorization Code + PKCE
   │
   ▼
Identity / Token Service
   │
   ▼
External Agency IdP
   │
   ▼
Identity established
   │
   ▼
Application Session
```

### Authentication

> Who are you?

### Authorization

> What are you allowed to do?

---

# 27. APPLICATION-TO-APPLICATION AUTHENTICATION

Typical architecture:

```text
Portal
  │
  ▼
Managed Identity
  │
  ▼
Microsoft Entra ID
  │
  ▼
JWT Access Token
  │
  ▼
Case API
```

Request:

```http
Authorization: Bearer <JWT>
```

Additional application context may include:

```text
UserId
AgencyId
CorrelationId
IP
```

### Important

> Authentication establishes identity; application context can establish the business/user/tenant context needed for authorization and auditing.

---

# 28. JWT

Structure:

```text
HEADER.PAYLOAD.SIGNATURE
```

Example claims:

```json
{
  "sub": "user123",
  "aud": "case-api",
  "iss": "identity-server",
  "role": "Investigator"
}
```

Remember:

> **JWT payload is encoded, not encrypted.**

Never store secrets in it.

---

# 29. ACCESS TOKEN vs REFRESH TOKEN

```text
Login
  ↓
Access Token
  ↓
Call APIs

Access Token expires
  ↓
Refresh Token
  ↓
New Access Token
```

### Memory

> Access token = access API
> Refresh token = obtain new access token

---

# 30. PDP / PEP AUTHORIZATION

Very important for your architecture.

```text
Request
   ↓
PEP
   ↓
PDP
   ↓
Policy Decision
   ↓
Allow / Deny
```

### PEP

> Policy Enforcement Point

### PDP

> Policy Decision Point

### Memory

> **PDP decides. PEP enforces.**

---

# 31. SECURITY — ANSWER IN LAYERS

```text
HTTPS
  ↓
Authentication
  ↓
Authorization
  ↓
Tenant Isolation
  ↓
Input Validation
  ↓
Business Authorization
  ↓
Data Protection
  ↓
Audit
```

Also mention:

* OIDC
* JWT
* RBAC
* Managed Identity
* Key Vault
* Encryption
* Least privilege
* Rate limiting
* Secure headers
* Audit logging

### Senior answer

> "I treat security as a layered concern rather than a single authentication mechanism."

---

# 32. MULTI-TENANCY

Your platform handles multiple agencies/tenants.

Think:

```text
Tenant
   ↓
Identity
   ↓
Authorization
   ↓
Tenant Context
   ↓
Data Access
```

Tenant isolation should be enforced at multiple layers.

For search:

```text
Agency A
   ↓
Agency-specific search/index boundaries

Agency B
   ↓
Agency-specific search/index boundaries
```

### Key concern

> Never trust tenant information supplied blindly by the client.

Derive/validate it from trusted identity/context.

---

# 33. SIGNALR

Used for real-time communication.

```text
Backend
   ↓
SignalR
   ↓
Browser
```

Example:

```text
Evidence processed
       ↓
Notification
       ↓
SignalR
       ↓
Vue UI
       ↓
"Evidence is ready"
```

Benefit:

> Avoid continuous polling.

---

# 34. DESIGN PATTERNS

Think of the three groups:

```text
CREATIONAL
→ Create objects

STRUCTURAL
→ Connect objects

BEHAVIORAL
→ Communicate/behave
```

---

# 35. CREATIONAL PATTERNS

| Pattern          | Memory             |
| ---------------- | ------------------ |
| Singleton        | One instance       |
| Factory          | Choose/create      |
| Abstract Factory | Family             |
| Builder          | Build step-by-step |
| Prototype        | Clone              |

### Factory

```text
MediaFactory
    │
 ┌──┼────┐
 ▼  ▼    ▼
MP4 G64 AVI
```

### Builder

```text
Builder
 ↓
Step 1
 ↓
Step 2
 ↓
Step 3
 ↓
Build()
```

### Important distinction

> **Factory chooses what to create.**
> **Builder controls how to construct it.**

---

# 36. STRUCTURAL PATTERNS

| Pattern   | Memory               |
| --------- | -------------------- |
| Adapter   | Translator           |
| Bridge    | Separate abstraction |
| Composite | Tree                 |
| Decorator | Add behavior         |
| Facade    | Simple front door    |
| Flyweight | Share memory         |
| Proxy     | Control access       |

---

# 37. ADAPTER

External API doesn't match your application.

```text
Application
    ↓
Adapter
    ↓
Legacy API
```

Memory:

> **Adapter = translator**

---

# 38. DECORATOR

Add behavior without modifying original class.

```text
Service
  ↓
Logging Decorator
  ↓
Caching Decorator
  ↓
Authorization Decorator
  ↓
Service
```

Memory:

> **Decorator = wrap + add behavior**

---

# 39. FACADE

Hide complexity.

```text
Client
  ↓
MediaFacade
  ↓
 ┌──────┬──────┬──────┐
Probe Transcode Thumbnail
```

Memory:

> **Facade = simple front door**

---

# 40. PROXY

Controls access to another object.

```text
Client
  ↓
Proxy
  ↓
Real Service
```

Can add:

* Authorization
* Caching
* Lazy loading
* Remote access

Memory:

> **Proxy = controlled access**

---

# 41. BEHAVIORAL PATTERNS

| Pattern         | Memory                |
| --------------- | --------------------- |
| Strategy        | Choose algorithm      |
| Observer        | Notify                |
| Chain           | Pass request          |
| Command         | Encapsulate action    |
| State           | Behavior by state     |
| Template Method | Algorithm skeleton    |
| Mediator        | Central communication |
| Visitor         | Add operations        |

---

# 42. STRATEGY

```text
MediaProcessor
      │
      ▼
Strategy
 ┌────┼────┐
 ▼    ▼    ▼
G64  MP4  AVI
```

Use when algorithm can vary.

Memory:

> **Strategy = interchangeable algorithm**

---

# 43. OBSERVER

```text
Event
 │
 ├── Notification
 ├── Audit
 └── Logging
```

Observer is typically in-process.

Service Bus Pub/Sub is distributed messaging.

---

# 44. CHAIN OF RESPONSIBILITY

```text
Request
 ↓
Authentication
 ↓
Authorization
 ↓
Logging
 ↓
Validation
 ↓
Controller
```

ASP.NET Core middleware is commonly explained using this pattern's concept.

---

# 45. COMMAND

Encapsulates an action.

```text
CreateCaseCommand
UpdateCaseCommand
DeleteCaseCommand
```

Then:

```text
Command
 ↓
Handler
 ↓
Business logic
```

This maps naturally to CQRS.

---

# 46. MEDIATOR

Without mediator:

```text
A → B
A → C
A → D
B → C
```

With mediator:

```text
A ─┐
B ─┼→ Mediator
C ─┤
D ─┘
```

Memory:

> **Mediator reduces direct object-to-object communication.**

Again:

> **Mediator ≠ CQRS**

---

# 47. SOLID

### S — Single Responsibility

One responsibility/reason to change.

### O — Open/Closed

Extend without unnecessarily modifying existing code.

### L — Liskov

Subtypes should be substitutable for their base type.

### I — Interface Segregation

Small focused interfaces.

### D — Dependency Inversion

Depend on abstractions.

```text
Application
     ↓
 IRepository
     ↑
Repository
     ↓
SQL
```

### Most architecturally important

> **Dependency Inversion**

---

# 48. REPOSITORY + UNIT OF WORK

## Repository

```text
Application
 ↓
IRepository
 ↓
Repository
 ↓
EF Core
 ↓
SQL
```

Provides persistence abstraction.

## Unit of Work

Coordinates changes within a database transaction.

```text
Create Case
   +
Create Evidence
   +
Create Audit
   ↓
Transaction
```

### Important

EF Core `DbContext` already provides repository/UoW-like capabilities.

So:

> Don't create abstractions just for the sake of patterns.

---

# 49. STRANGLER FIG

For legacy modernization:

```text
          Legacy
            │
       ┌────┴────┐
       │         │
     Old       Old
     feature   feature

Gradually replace

          New
       ┌────┴────┐
     New       New
```

Memory:

> **Replace gradually, not big bang.**

---

# 50. ANTI-CORRUPTION LAYER

```text
New Domain
    ↓
ACL / Translator
    ↓
Legacy
```

Purpose:

> Prevent legacy models from leaking into the new domain.

---

# 51. VUE 2 → VUE 3 MIGRATION

Your migration approach is incremental.

```text
Vue 2
 ↓
Vue 2.7
 ↓
Compatibility cleanup
 ↓
Vue 3
```

Important changes:

```text
createApp()
app.use()
createRouter()
createStore()
nextTick()
```

Also review:

* vuex-class
* Third-party components
* Shared component library
* Build tooling
* Validation libraries

### Senior answer

> "I prefer incremental migration because it reduces production risk and allows compatibility issues to be identified and resolved progressively."

---

# 52. GENAI

Your easiest mental model:

# LLM → RAG → Agent → Skills → MCP

But understand the difference.

### LLM

Language reasoning/generation engine.

### RAG

Gives LLM external knowledge.

### Agent

Decides what action/tool to use.

### Tool

Actually performs the operation.

### Skill

Reusable capability/workflow.

### MCP

Standard protocol for connecting AI systems with external tools/context.

---

# 53. RAG

```text
Documents
    ↓
Chunking
    ↓
Embeddings
    ↓
Vector DB
    ↓
Retrieve
    ↓
Filter
    ↓
Rerank
    ↓
LLM
    ↓
Grounded Answer
```

### Improve RAG with

* Hybrid search
* Metadata filtering
* Reranking
* Access control
* Citations
* Grounding
* Evaluation

---

# 54. AGENT

```text
User
 ↓
Agent
 ↓
Decides
 ├── Search
 ├── Database
 ├── API
 ├── MCP
 └── Tool
 ↓
Result
 ↓
Agent
 ↓
Answer / Action
```

### Memory

> **LLM = Brain**
> **RAG = Knowledge**
> **Tools = Hands**
> **Agent = Decision Maker**

---

# 55. MCP + SKILLS + AGENT

Example:

> "Find the failed deployment and create a bug."

```text
User
 ↓
Agent
 ↓
DevOps Skill
 ↓
MCP
 ↓
DevOps Platform
 ↓
Get failed deployment
 ↓
Analyze
 ↓
Create Bug
 ↓
Report
```

Important:

> **MCP is not the Agent.**

The Agent uses MCP to access external capabilities/context.

---

# 56. PYDANTIC

Think:

> **Pydantic = structured data validation**

```text
LLM
 ↓
JSON
 ↓
Pydantic
 ↓
Validation
 ↓
Application
```

Useful when LLM output must follow a strict schema.

---

# 57. LANGCHAIN

Think:

> **LLM workflow orchestration framework**

Can combine:

```text
Prompt
 ↓
LLM
 ↓
Retriever
 ↓
Tool
 ↓
Output
```

Don't say:

> "LangChain is an AI model."

Say:

> "LangChain helps orchestrate LLM application workflows and components."

---

# 58. GENAI SECURITY

Always consider:

```text
Prompt Injection
Data Leakage
Unauthorized Tools
Hallucination
Sensitive Information
```

Controls:

```text
Authentication
Authorization
Least privilege
Tool allow-list
Input validation
Output validation
Audit
Human approval
Access-controlled RAG
```

---

# 59. CI/CD

Remember:

```text
Developer
   ↓
Git
   ↓
Build
   ↓
Unit Test
   ↓
Security / Quality
   ↓
Artifact
   ↓
Dev
   ↓
Test
   ↓
QA
   ↓
Production
   ↓
Monitor
```

### Memory

> **Build → Test → Package → Deploy → Monitor**

---

# 60. PRODUCTION TROUBLESHOOTING

Use:

# Detect → Measure → Isolate → Fix → Validate → Monitor

Example:

```text
User reports slow page
       ↓
Browser DevTools
       ↓
API latency?
       ↓
DB?
       ↓
Elasticsearch?
       ↓
External API?
       ↓
Service Bus?
       ↓
Find bottleneck
       ↓
Fix
       ↓
Measure again
```

Tools:

* Application Insights
* Logs
* Metrics
* Distributed tracing
* SQL execution plans
* Elasticsearch metrics
* Service Bus metrics
* Browser DevTools

---

# 61. OBSERVABILITY

Three pillars:

```text
LOGS
METRICS
TRACES
```

Use:

> Correlation ID

Example:

```text
Browser
 correlationId=123
      ↓
API
 correlationId=123
      ↓
Service
 correlationId=123
      ↓
Service Bus
 correlationId=123
```

Now you can trace one business request across the distributed system.

---

# 62. DESIGN PATTERN → REAL PROJECT

This table is worth memorizing.

| Requirement               | Pattern               |
| ------------------------- | --------------------- |
| Select implementation     | Factory               |
| Complex creation          | Builder               |
| External API mismatch     | Adapter               |
| Add logging/cache         | Decorator             |
| Hide complexity           | Facade                |
| Control access            | Proxy                 |
| Select algorithm          | Strategy              |
| Notify subscribers        | Observer              |
| Request pipeline          | Chain                 |
| Encapsulate action        | Command               |
| Central communication     | Mediator              |
| Read/write separation     | CQRS                  |
| Distributed transaction   | Saga                  |
| Legacy migration          | Strangler             |
| Protect new domain        | ACL                   |
| Frontend-specific backend | BFF                   |
| Common API entry point    | Gateway               |
| Background jobs           | Hangfire              |
| Event processing          | Service Bus/Functions |
| Long-running workflow     | Durable Functions     |

---

# 63. MOST IMPORTANT INTERVIEW DIFFERENCES

### CQRS vs Event Sourcing

```text
CQRS
→ READ / WRITE separation

Event Sourcing
→ State stored as events
```

---

### Queue vs Topic

```text
Queue
→ Work

Topic
→ Event
```

---

### Redis vs Elasticsearch

```text
Redis
→ Cache

Elasticsearch
→ Search
```

---

### Retry vs Circuit Breaker

```text
Retry
→ Try again

Circuit Breaker
→ Stop calling
```

---

### Authentication vs Authorization

```text
Authentication
→ Who?

Authorization
→ What?
```

---

### BFF vs Gateway

```text
BFF
→ Frontend-specific backend

Gateway
→ Common API front door
```

---

### Factory vs Strategy

```text
Factory
→ What object?

Strategy
→ What algorithm?
```

---

### Adapter vs Facade

```text
Adapter
→ Translate interface

Facade
→ Simplify interface
```

---

### Async vs Parallel

```text
Async
→ Don't block waiting

Parallel
→ Execute independent work concurrently
```

---

# 64. YOUR TOP 15 INTERVIEW QUESTIONS

Prepare these first.

### 1. Explain your project architecture.

Answer:

```text
Vue
 ↓
BFF
 ↓
.NET APIs
 ↓
Application/Domain
 ↓
SQL

Events
 ↓
Service Bus
 ↓
Functions/Workflow/Indexer
 ↓
ES/Notification
```

---

### 2. Why CQRS?

> "To separate read and write responsibilities and allow each path to be optimized independently."

---

### 3. Does CQRS require MediatR?

> "No. MediatR is only a dispatching mechanism."

---

### 4. Why Elasticsearch?

> "For high-performance search and read-heavy workloads while SQL remains the transactional source of truth."

---

### 5. How do you handle eventual consistency?

> "Using asynchronous indexing with retry, idempotency, DLQ, monitoring and re-indexing mechanisms."

---

### 6. How do you handle duplicate Service Bus messages?

> "Consumers are designed to be idempotent using message/business identifiers or processing records."

---

### 7. Retry vs Circuit Breaker?

> "Retry handles transient failures; circuit breaker prevents repeatedly calling an unhealthy dependency."

---

### 8. How does your workflow work?

> "Detect → Decide → Wait → Execute → Report."

---

### 9. Why Service Bus?

> "To decouple services, support asynchronous processing and improve scalability and resilience."

---

### 10. How do you improve API performance?

> "Measure first, then optimize query projections, pagination, caching, async processing, payload size and expensive downstream dependencies."

---

### 11. How do you improve SQL performance?

> "Execution plans, indexes, projection, AsNoTracking, pagination, avoiding N+1 and unnecessary joins/includes."

---

### 12. How do you secure APIs?

> "OIDC/JWT authentication, authorization policies/RBAC, tenant isolation, HTTPS, input validation, managed identity, Key Vault, least privilege and audit logging."

---

### 13. How do you troubleshoot production issues?

> "Detect → Measure → Isolate → Fix → Validate → Monitor."

---

### 14. Explain Saga.

> "Saga manages distributed business transactions using local transactions and compensating actions rather than a single distributed transaction."

---

### 15. Explain Agent/MCP/Skills.

> "An Agent decides what to do, Skills provide reusable capabilities, and MCP provides a standardized way for the agent to access external tools and context."

---

# 65. HOW TO ANSWER AS A TECHNICAL LEAD

Don't answer only technically.

Use:

```text
Business Requirement
       ↓
Technical Problem
       ↓
Options
       ↓
Decision
       ↓
Implementation
       ↓
Security
       ↓
Performance
       ↓
Resilience
       ↓
Trade-off
```

Example:

### Interviewer:

> "Why did you use Elasticsearch?"

Weak answer:

> "Because Elasticsearch is fast."

Senior answer:

> "The requirement was to support large-scale search and filtering without putting excessive load on the transactional database. We therefore used Elasticsearch as a read/search optimization layer. SQL remained the source of truth, and events asynchronously updated the search index. The trade-off was eventual consistency, so we added retry, idempotency, DLQ and re-indexing mechanisms."

That sounds like a **Technical Lead**.

---

# 66. YOUR 10 GOLDEN ARCHITECTURE STATEMENTS

Memorize these word-for-word.

> **1. "I start with requirements and constraints rather than selecting technology first."**

> **2. "I choose the simplest architecture that satisfies both functional and non-functional requirements."**

> **3. "CQRS separates read and write responsibilities; it doesn't inherently require separate databases."**

> **4. "MediatR is a dispatching mechanism, not a requirement for CQRS."**

> **5. "Because distributed messaging can deliver duplicate messages, consumers should be idempotent."**

> **6. "Retry handles transient failures, while circuit breaker protects the system from repeatedly calling an unhealthy dependency."**

> **7. "SQL remains the transactional source of truth, while Elasticsearch is used for search and read optimization."**

> **8. "For long-running processing, I prefer asynchronous workflows rather than keeping the HTTP request open."**

> **9. "I prefer bounded concurrency instead of unbounded parallelism."**

> **10. "I measure performance first, identify the bottleneck, optimize it, and measure again."**

---

# 67. FINAL 30-MINUTE REVISION

If you have only 30 minutes:

### 5 minutes

```text
Project Architecture
CQRS
Service Bus
Elasticsearch
```

### 5 minutes

```text
Authentication
Authorization
PDP / PEP
Security
```

### 5 minutes

```text
Retry
Circuit Breaker
Saga
Idempotency
DLQ
```

### 5 minutes

```text
Design Patterns
SOLID
Repository
Unit of Work
```

### 5 minutes

```text
Performance
EF Core
Redis
Azure Functions
```

### Final 5 minutes

Speak this:

```text
Project
 ↓
Architecture
 ↓
CQRS
 ↓
Events
 ↓
Workflow
 ↓
Security
 ↓
Performance
 ↓
Resilience
 ↓
GenAI
```

---

# 68. FINAL 10-MINUTE REVISION

If you're literally entering the interview:

```text
ARCHITECTURE
Clean + Microservices + Event Driven

CQRS
Command → Write
Query → Read

DATA
SQL → Truth
Redis → Cache
ES → Search

MESSAGING
Queue → Work
Topic → Event

RESILIENCE
Retry → Try again
Circuit → Stop
Saga → Compensation
Idempotency → Duplicate safe
DLQ → Failed messages

WORKFLOW
Detect → Decide → Wait → Execute → Report

SECURITY
Authentication → Who
Authorization → What
PDP → Decide
PEP → Enforce

PERFORMANCE
Measure → Bottleneck → Optimize → Measure

GENAI
LLM → RAG → Agent → Skills → MCP

PATTERNS
Factory → Create
Adapter → Translate
Decorator → Add
Facade → Simplify
Proxy → Control
Strategy → Choose algorithm
Observer → Notify
Command → Action
Mediator → Communicate
```

# 69. THE ONE ANSWER THAT CONNECTS EVERYTHING

If the interviewer asks a broad question like:

> **"How do you design an enterprise application?"**

Your answer should be:

> "I start from the business requirements and non-functional requirements such as scalability, availability, security, performance and maintainability. I establish clear business boundaries and choose an appropriate architecture such as modular monolith or microservices depending on the complexity. Within services, I prefer clean architectural principles with clear separation between API, application, domain and infrastructure. Where read and write workloads differ, I use CQRS. For asynchronous communication I use messaging such as Azure Service Bus, with idempotent consumers, retry, DLQ and appropriate monitoring. SQL remains the transactional source of truth, while technologies such as Elasticsearch and Redis are introduced for specific read/search and caching requirements. For long-running workflows I use asynchronous processing and orchestration. Finally, I consider authentication, authorization, tenant isolation, observability, CI/CD and operational trade-offs before finalizing the design."

## Your final mental model

```text
                  SENIOR TECHNICAL LEAD
                           │
       ┌───────────────────┼──────────────────┐
       ▼                   ▼                  ▼
 ARCHITECTURE          RELIABILITY          SECURITY
       │                   │                  │
 Clean                 Retry                OIDC
 Microservices         Circuit              JWT
 Event Driven          Saga                 RBAC
 CQRS                  Idempotency          PDP/PEP
 BFF                   DLQ                  Tenant
       │
       ▼
       DATA
   ┌────┼────┐
   ▼    ▼    ▼
  SQL Redis  ES
 Truth Cache Search

       │
       ▼
    WORKFLOW
       │
Detect → Decide → Wait
       ↓
Execute → Report

       │
       ▼
   PERFORMANCE
       │
Measure → Bottleneck
       ↓
Optimize → Measure

       │
       ▼
      GENAI
       │
LLM → RAG → Agent
       ↓
Skills → MCP → Tools
```

**If you can explain this entire flow naturally, you have the core material for your Senior Technical Lead / Solution Architect interview.**
