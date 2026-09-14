Yes. This is the **most important part of CQRS to understand**:

> **How does the system know that `CreateCaseCommand` should go to `CreateCaseHandler`, while `UpdateCaseCommand` should go to `UpdateCaseHandler`?**

The answer depends on whether you use **MediatR, your own dispatcher, or direct dependency injection**.

---

# 1. The basic relationship

Think of CQRS as a contract:

```text
Command                         Handler

CreateCaseCommand      ───────→ CreateCaseHandler

UpdateCaseCommand      ───────→ UpdateCaseHandler

DeleteCaseCommand      ───────→ DeleteCaseHandler

ApproveCaseCommand     ───────→ ApproveCaseHandler
```

The handler is associated with a **specific command type**.

For example:

```csharp
public record CreateCaseCommand(string Title);
```

and:

```csharp
public class CreateCaseHandler
{
    public async Task Handle(CreateCaseCommand command)
    {
        // create case
    }
}
```

The important relationship is:

```text
CreateCaseCommand
       ↓
CreateCaseHandler
```

---

# 2. Use case #1 — Direct handler

No Mediator.

The controller explicitly knows which handler it wants.

```csharp
[HttpPost]
public async Task<IActionResult> Create(
    CreateCaseRequest request)
{
    var command =
        new CreateCaseCommand(request.Title);

    var result =
        await _createCaseHandler.Handle(command);

    return Ok(result);
}
```

Here there is no "identification" mechanism.

The code itself says:

```text
_createCaseHandler
       ↓
CreateCaseHandler
```

### Flow

```text
POST /cases
     ↓
Controller
     ↓
CreateCaseCommand
     ↓
ICreateCaseHandler
     ↓
CreateCaseHandler
```

This is the simplest approach.

---

# 3. Use case #2 — Dependency Injection

Usually you don't directly instantiate:

```csharp
new CreateCaseHandler()
```

Instead:

```csharp
builder.Services.AddScoped<
    ICreateCaseHandler,
    CreateCaseHandler>();
```

Then:

```csharp
public CasesController(
    ICreateCaseHandler createCaseHandler)
{
    _createCaseHandler = createCaseHandler;
}
```

So .NET DI maintains:

```text
ICreateCaseHandler
       ↓
CreateCaseHandler
```

Again, there is no magic.

**DI resolves the interface to the registered implementation.**

---

# 4. Use case #3 — MediatR

This is where handler identification becomes interesting.

You write:

```csharp
public record CreateCaseCommand(
    string Title
) : IRequest<int>;
```

And:

```csharp
public class CreateCaseHandler
    : IRequestHandler<CreateCaseCommand, int>
{
    public async Task<int> Handle(
        CreateCaseCommand request,
        CancellationToken cancellationToken)
    {
        // create case

        return 100;
    }
}
```

Then controller says only:

```csharp
var result =
    await _mediator.Send(
        new CreateCaseCommand("Test Case"));
```

The controller doesn't explicitly say:

```text
Call CreateCaseHandler
```

Instead, MediatR looks for the registered handler matching:

```text
IRequestHandler<CreateCaseCommand, int>
```

So:

```text
CreateCaseCommand
       ↓
MediatR
       ↓
IRequestHandler<CreateCaseCommand, int>
       ↓
CreateCaseHandler
```

---

# 5. How does MediatR know?

Conceptually, MediatR has a mapping like:

```text
Command Type                         Handler

CreateCaseCommand              → CreateCaseHandler

UpdateCaseCommand              → UpdateCaseHandler

DeleteCaseCommand              → DeleteCaseHandler

ApproveCaseCommand             → ApproveCaseHandler
```

At application startup, handlers are registered/discovered through dependency injection.

Conceptually:

```csharp
services.AddTransient<
    IRequestHandler<CreateCaseCommand, int>,
    CreateCaseHandler>();

services.AddTransient<
    IRequestHandler<UpdateCaseCommand>,
    UpdateCaseHandler>();
```

Then when:

```csharp
_mediator.Send(createCommand)
```

is called, the command's type tells the mediator which handler contract to resolve.

---

# 6. Example with three commands

Suppose we have:

```csharp
public record CreateCaseCommand(
    string Title) : IRequest<int>;

public record UpdateCaseCommand(
    int CaseId,
    string Title) : IRequest;

public record DeleteCaseCommand(
    int CaseId) : IRequest;
```

Handlers:

```csharp
public class CreateCaseHandler
    : IRequestHandler<CreateCaseCommand, int>
{
    public Task<int> Handle(
        CreateCaseCommand request,
        CancellationToken cancellationToken)
    {
        // Create
    }
}
```

```csharp
public class UpdateCaseHandler
    : IRequestHandler<UpdateCaseCommand>
{
    public Task Handle(
        UpdateCaseCommand request,
        CancellationToken cancellationToken)
    {
        // Update
    }
}
```

```csharp
public class DeleteCaseHandler
    : IRequestHandler<DeleteCaseCommand>
{
    public Task Handle(
        DeleteCaseCommand request,
        CancellationToken cancellationToken)
    {
        // Delete
    }
}
```

Now:

```csharp
await _mediator.Send(
    new CreateCaseCommand("Case A"));
```

goes to:

```text
CreateCaseHandler
```

While:

```csharp
await _mediator.Send(
    new UpdateCaseCommand(10, "Case B"));
```

goes to:

```text
UpdateCaseHandler
```

And:

```csharp
await _mediator.Send(
    new DeleteCaseCommand(10));
```

goes to:

```text
DeleteCaseHandler
```

---

# 7. The key is the COMMAND TYPE

This is the sentence you should remember:

> **The command/query type acts as the key for identifying the corresponding handler.**

For example:

```text
CreateCaseCommand
       ↓
IRequestHandler<CreateCaseCommand, ...>
       ↓
CreateCaseHandler
```

And:

```text
GetCaseQuery
       ↓
IRequestHandler<GetCaseQuery, CaseDto>
       ↓
GetCaseHandler
```

---

# 8. What happens with Service Bus?

Now let's connect this with your previous question.

Suppose Service Bus receives:

```text
CaseCreatedEvent
```

The Service Bus itself does **not** identify:

```text
CreateCaseHandler
```

Instead:

```text
Service Bus
     ↓
Subscription
     ↓
Azure Function / Consumer
     ↓
Deserialize Event
     ↓
Create appropriate Command
     ↓
Mediator / Dispatcher
     ↓
Handler
```

For example:

```csharp
[Function("CaseProcessor")]
public async Task Run(
    [ServiceBusTrigger(
        "CaseEvents",
        "SearchSubscription",
        Connection = "ServiceBus")]
    string message)
{
    var evt =
        JsonSerializer.Deserialize<CaseCreatedEvent>(
            message);

    var command =
        new IndexCaseCommand(evt.CaseId);

    await _mediator.Send(command);
}
```

Now MediatR sees:

```text
IndexCaseCommand
```

and finds:

```text
IRequestHandler<IndexCaseCommand>
```

which is:

```text
IndexCaseHandler
```

So the complete flow is:

```text
                    SERVICE BUS

CaseCreatedEvent
       ↓
Topic
       ↓
SearchSubscription
       ↓
Azure Function
       ↓
IndexCaseCommand
       ↓
MediatR
       ↓
IRequestHandler<IndexCaseCommand>
       ↓
IndexCaseHandler
       ↓
Elasticsearch
```

---

# 9. Very important distinction

There are **two different decisions**.

### Service Bus decision

```text
Which subscription receives this message?
```

For example:

```text
CaseCreated
    ↓
CaseEvents Topic
    ↓
SearchSubscription
```

### Application/CQRS decision

```text
Which handler handles this command?
```

For example:

```text
IndexCaseCommand
    ↓
MediatR
    ↓
IndexCaseHandler
```

Therefore:

> **Service Bus identifies the messaging destination; CQRS/Mediator identifies the application handler.**

---

# 10. Use case — Same event, different subscriptions

Suppose:

```text
CaseCreatedEvent
```

is published.

Topic:

```text
CaseEvents
```

Subscriptions:

```text
CaseEvents
   |
   +---- SearchSubscription
   |
   +---- NotificationSubscription
   |
   +---- AuditSubscription
```

### Search subscription

```text
CaseCreatedEvent
       ↓
Search Function
       ↓
IndexCaseCommand
       ↓
IndexCaseHandler
       ↓
Elasticsearch
```

### Notification subscription

```text
CaseCreatedEvent
       ↓
Notification Function
       ↓
SendNotificationCommand
       ↓
SendNotificationHandler
       ↓
Email / SignalR
```

### Audit subscription

```text
CaseCreatedEvent
       ↓
Audit Function
       ↓
RecordAuditCommand
       ↓
RecordAuditHandler
       ↓
Audit DB
```

Notice something important:

**The same event can result in different commands and therefore different handlers.**

---

# 11. Use case — One subscription receives multiple events

Suppose:

```text
SearchSubscription
```

receives:

```text
CaseCreatedEvent
CaseUpdatedEvent
CaseDeletedEvent
```

The consumer needs to determine the event type.

Conceptually:

```csharp
switch (eventType)
{
    case "CaseCreated":
        await _mediator.Send(
            new IndexCaseCommand(caseId));
        break;

    case "CaseUpdated":
        await _mediator.Send(
            new UpdateCaseIndexCommand(caseId));
        break;

    case "CaseDeleted":
        await _mediator.Send(
            new DeleteCaseIndexCommand(caseId));
        break;
}
```

Flow:

```text
                    SearchSubscription
                           |
            +--------------+--------------+
            |              |              |
            v              v              v
      CaseCreated      CaseUpdated     CaseDeleted
            |              |              |
            v              v              v
      IndexCaseCmd   UpdateIndexCmd  DeleteIndexCmd
            |              |              |
            v              v              v
      CreateHandler   UpdateHandler   DeleteHandler
```

This is one possible design.

Another design is to have **separate subscriptions/functions for different event types**, which can be cleaner when processing requirements differ significantly.

---

# 12. Use case — Query handler

Exactly the same concept applies to queries.

```csharp
public record GetCaseQuery(
    int CaseId) : IRequest<CaseDto>;
```

Handler:

```csharp
public class GetCaseHandler
    : IRequestHandler<GetCaseQuery, CaseDto>
{
    public async Task<CaseDto> Handle(
        GetCaseQuery request,
        CancellationToken cancellationToken)
    {
        // Query database / Elasticsearch

        return new CaseDto();
    }
}
```

Call:

```csharp
await _mediator.Send(
    new GetCaseQuery(100));
```

MediatR identifies:

```text
GetCaseQuery
      ↓
IRequestHandler<GetCaseQuery, CaseDto>
      ↓
GetCaseHandler
```

---

# 13. What if two handlers support the same command?

Normally, you should **not** have:

```text
CreateCaseCommand
     ↓
Handler A
Handler B
```

for a normal request/response CQRS command.

There should generally be **one request handler for one request type**.

If you intentionally want multiple components to react to something, use an **event/notification/pub-sub style** instead.

For example:

```text
CaseCreatedEvent
      |
      +---- AuditHandler
      |
      +---- NotificationHandler
      |
      +---- SearchHandler
```

That's conceptually different from:

```text
CreateCaseCommand
      |
      +---- Handler A
      +---- Handler B
```

---

# 14. Command vs Event — this explains everything

### Command

```text
CreateCaseCommand
```

means:

> "Please create a case."

Usually:

```text
ONE command
   ↓
ONE command handler
```

### Event

```text
CaseCreatedEvent
```

means:

> "A case has already been created."

Potentially:

```text
ONE event
   ↓
MANY consumers
```

Therefore:

```text
COMMAND

CreateCaseCommand
       ↓
CreateCaseHandler
```

versus:

```text
EVENT

CaseCreatedEvent
       ↓
   +---+---+---+
   ↓   ↓   ↓
Search Audit Notification
```

This distinction is extremely important in system design interviews.

---

# 15. Without MediatR — your own dispatcher

You can also create your own mapping.

For example:

```csharp
public interface ICommandHandler<TCommand>
{
    Task Handle(TCommand command);
}
```

Then:

```csharp
public class CreateCaseHandler
    : ICommandHandler<CreateCaseCommand>
{
    public Task Handle(CreateCaseCommand command)
    {
        // ...
    }
}
```

Your dispatcher can conceptually maintain:

```text
CreateCaseCommand → CreateCaseHandler
UpdateCaseCommand → UpdateCaseHandler
DeleteCaseCommand → DeleteCaseHandler
```

Then:

```csharp
await dispatcher.Dispatch(command);
```

Internally:

```text
Dispatcher
    ↓
command.GetType()
    ↓
Find registered handler
    ↓
Invoke handler
```

So **MediatR is not magic**. It is implementing this kind of dispatching infrastructure for you.

---

# 16. Final architecture you should remember

```text
                         MESSAGE ARRIVES
                              |
                              v
                       Azure Service Bus
                              |
                              v
                           Topic
                              |
                              v
                        Subscription
                              |
                              v
                   Consumer / Azure Function
                              |
                              v
                       Deserialize Event
                              |
                              v
                     Determine Use Case
                              |
                              v
                         Command
                              |
                              v
                    +-------------------+
                    |                   |
                 Direct              Mediator
                    |                   |
                    |                   v
                    |             Find Handler
                    |                   |
                    +---------+---------+
                              |
                              v
                         Specific Handler
                              |
                              v
                       Business Logic
                              |
                              v
                       DB / External API
```

## ⭐ The handler identification chain

Memorize this:

```text
MESSAGE
   ↓
SUBSCRIPTION
   ↓
CONSUMER
   ↓
EVENT TYPE
   ↓
COMMAND TYPE
   ↓
MEDIATOR / DI
   ↓
HANDLER TYPE
   ↓
BUSINESS LOGIC
```

And for REST:

```text
HTTP REQUEST
    ↓
CONTROLLER
    ↓
COMMAND / QUERY
    ↓
MEDIATOR / DI
    ↓
SPECIFIC HANDLER
```

### Your interview-ready answer

> **"The handler is identified by the command or query type. With MediatR, each command/query has a corresponding `IRequestHandler<TRequest,TResponse>` registration, so when `Send()` receives a particular request type, MediatR resolves the matching handler through dependency injection. With Service Bus, the bus first delivers the message to a subscription consumer, such as an Azure Function. The consumer deserializes the event and creates the appropriate command. That command is then dispatched to its specific handler, either directly or through MediatR."**

That is the cleanest way to explain **REST → CQRS handler** and **Service Bus → CQRS handler** in an interview.
