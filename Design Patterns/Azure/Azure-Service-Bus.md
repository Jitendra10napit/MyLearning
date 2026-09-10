# Azure Service Bus — Future Interview Revision Notes

Think of Azure Service Bus as the **reliable communication backbone between enterprise services**.

### 1. The easiest memory trick

```text
QUEUE  →  "Please DO this"
TOPIC  →  "This HAPPENED"
```

For example:

```text
CreateOrderCommand       → Queue
OrderCreatedEvent        → Topic
```

---

# 2. Queue — Enterprise Example

Suppose you have an **Investigation Management System**.

When an investigation is created, the API should not directly perform every heavy operation.

```text
                    Azure Service Bus
                ┌─────────────────────┐
API ───────────>│   InvestigationQueue │
                └──────────┬──────────┘
                           │
               ┌───────────┼───────────┐
               ▼           ▼           ▼
           Worker-1    Worker-2    Worker-3
           Process      Process      Process
```

The API sends:

```text
CreateInvestigationCommand
```

Workers compete for messages.

### Important point

If there are 1,000 messages:

```text
Message 1  → Worker 1
Message 2  → Worker 2
Message 3  → Worker 3
Message 4  → Worker 1
...
```

**One message is normally processed by only one competing consumer.**

### When to use Queue

Use a queue when you have:

* Background processing
* Commands/tasks
* Load leveling
* Multiple worker instances
* Asynchronous processing

Examples:

```text
CreateInvestigation
GenerateReport
ProcessInvoice
SendDocument
UpdateCustomer
```

---

# 3. Topic — Enterprise Example

Now assume after investigation creation, multiple systems need to know about it.

```text
                 Investigation Service
                         │
                         ▼
                ┌──────────────────┐
                │ InvestigationEvents│
                │      Topic       │
                └────────┬─────────┘
                         │
          ┌──────────────┼───────────────┐
          ▼              ▼               ▼
     Payment Sub    Notification Sub   Audit Sub
          │              │               │
          ▼              ▼               ▼
     Payment Svc    Notification Svc   Audit Svc
```

The event could be:

```text
InvestigationCreated
```

Each subscription gets its **own copy**.

So:

```text
InvestigationCreated
        │
        ├──> Payment Service
        ├──> Notification Service
        └──> Audit Service
```

### When to use Topic

Use a topic when:

```text
ONE EVENT
   ↓
MANY INDEPENDENT CONSUMERS
```

Examples:

```text
OrderCreated
PaymentCompleted
CustomerRegistered
InvestigationClosed
ShipmentDispatched
```

---

# 4. Queue vs Topic — Interview Table

| Feature      | Queue                     | Topic                         |
| ------------ | ------------------------- | ----------------------------- |
| Pattern      | Point-to-point            | Publish/Subscribe             |
| Purpose      | Command/task              | Event                         |
| Consumers    | Competing consumers       | Multiple subscriptions        |
| Message copy | One consumer processes it | Each subscription gets a copy |
| Typical use  | Background work           | Event-driven architecture     |
| Example      | `CreateOrderCommand`      | `OrderCreatedEvent`           |

### One-line interview answer

> **Queue is for distributing work, while Topic is for distributing information.**

---

# 5. Topic + Subscription = Very Important

A common interview mistake is saying:

> "Consumer reads directly from Topic."

Usually think of it as:

```text
Producer
   │
   ▼
 Topic
   │
   ├── Subscription A → Consumer A
   ├── Subscription B → Consumer B
   └── Subscription C → Consumer C
```

The **subscription is the logical inbox** for a consumer/application.

For example:

```text
Topic: InvestigationEvents

Subscriptions:
    PaymentSubscription
    NotificationSubscription
    AuditSubscription
```

---

# 6. Concurrency — Very Important for Senior/Architect Interviews

This is one of the most important points.

### Queue

Suppose:

```text
3 Pods
Each Pod = MaxConcurrentCalls 10
```

Then roughly:

```text
3 × 10 = 30 concurrent message handlers
```

Visual:

```text
               Queue
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
     Pod 1     Pod 2     Pod 3
      10         10        10
   concurrent concurrent concurrent
```

So remember:

```text
Total concurrency ≈ Instances × MaxConcurrentCalls
```

But actual throughput also depends on:

```text
Database capacity
API rate limits
CPU
Network
Connection pool
Downstream services
```

### Topic

Concurrency is configured independently for each subscription consumer.

```text
InvestigationEvents Topic
          │
    ┌─────┼────────┐
    ▼     ▼        ▼
Payment  Audit   Notification
  20       5         50
```

This means Notification Service can process much faster without forcing Payment Service to do the same.

---

# 7. Where is MaxConcurrentCalls configured?

This is a **classic interview question**.

### Wrong

> Configure MaxConcurrentCalls in Azure Portal.

### Correct

**Processing concurrency is configured in the consumer application.**

Example:

```csharp
var options = new ServiceBusProcessorOptions
{
    MaxConcurrentCalls = 20,
    AutoCompleteMessages = false
};

var processor = client.CreateProcessor(
    "InvestigationEvents",
    "PaymentSubscription",
    options);
```

Remember:

```text
Azure Portal
   ↓
Entity configuration

Application
   ↓
Message-processing concurrency

Multiple Pods
   ↓
Horizontal scalability
```

---

# 8. Peek-Lock — Must Know

For important business messages, think:

```text
Receive
   ↓
Lock message
   ↓
Process
   ↓
Success?
  ├── YES → Complete
  └── NO  → Retry / Redelivery
```

Example:

```text
PaymentCompleted
       ↓
Payment Service receives message
       ↓
Update payment database
       ↓
SUCCESS → Complete()
```

If processing fails:

```text
Failure
  ↓
Message remains / is redelivered
  ↓
Retry
  ↓
Repeated failures
  ↓
DLQ
```

---

# 9. Dead Letter Queue — Very Important

Think:

```text
Normal processing
       ↓
   Temporary failure
       ↓
      Retry
       ↓
   Retry again
       ↓
Too many failures
       ↓
      DLQ
```

Typical reasons:

```text
Max delivery count exceeded
Invalid message
Business rule failure
Message processing failure
Expired message
```

Architect interview answer:

> I use DLQ as a safety mechanism so poison messages don't block normal processing. I monitor DLQ and have a controlled replay/remediation process.

---

# 10. Topic Filters — Very Useful

Suppose:

```text
InvestigationEvents Topic
```

contains:

```text
InvestigationCreated
InvestigationUpdated
InvestigationClosed
PaymentCompleted
```

Payment service only needs:

```text
PaymentCompleted
```

You can use subscription filters/rules.

```text
                    Topic
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
 PaymentSub     NotificationSub    AuditSub
       │
       │ Filter:
       │ EventType = PaymentCompleted
       ▼
 Payment Service
```

This avoids every consumer processing every event.

---

# 11. Ordering — Senior-Level Point

Suppose one customer's messages are:

```text
Customer123
 ├── CustomerCreated
 ├── CustomerUpdated
 └── CustomerClosed
```

Sometimes they must be processed sequentially.

Azure Service Bus **Sessions** are useful here.

```text
SessionId = Customer123
```

Think:

```text
Same Session
      ↓
Sequential

Different Sessions
      ↓
Can process concurrently
```

### Memory trick

> **Sequential within an entity, concurrent across entities.**

---

# 12. Duplicate Messages — Very Important

In distributed systems, assume:

```text
Same message may arrive twice
```

Therefore consumer should be **idempotent**.

Example:

```text
MessageId = INV-1001
```

Consumer checks whether it has already processed it.

```text
Receive INV-1001
      ↓
Already processed?
   ├── YES → Ignore
   └── NO  → Process
```

This becomes especially important for:

```text
Payments
Orders
Inventory
Financial transactions
```

---

# 13. Retry + Circuit Breaker + Service Bus

Imagine Payment Service consumes:

```text
PaymentRequested
```

and calls an external payment API.

```text
Service Bus
    ↓
Payment Service
    ↓
Payment API
```

If Payment API temporarily fails:

```text
Retry
   ↓
Retry
   ↓
Still failing
   ↓
Circuit Breaker OPEN
   ↓
Don't keep hammering API
```

Meanwhile the Service Bus message should be handled carefully so we don't lose it.

This is where these concepts work together:

```text
Service Bus
     ↓
Retry
     ↓
Timeout
     ↓
Circuit Breaker
     ↓
Idempotency
     ↓
DLQ
```

---

# 14. Enterprise Architecture — Put Everything Together

This is the diagram I'd remember for your interview:

```text
                     ┌───────────────────┐
                     │    API Gateway    │
                     └─────────┬─────────┘
                               │
                               ▼
                    ┌────────────────────┐
                    │ Investigation Svc  │
                    └─────────┬──────────┘
                              │
                   Command    │
                              ▼
                 ┌────────────────────────┐
                 │ Investigation Queue     │
                 └───────────┬────────────┘
                             │
                    ┌────────┼────────┐
                    ▼        ▼        ▼
                  Worker   Worker   Worker
                             │
                             ▼
                       Business DB
                             │
                     Publish Event
                             ▼
                 ┌────────────────────────┐
                 │ InvestigationEvents    │
                 │        Topic            │
                 └───────────┬────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
       Notification       Audit         Analytics
       Subscription      Subscription   Subscription
              │              │              │
              ▼              ▼              ▼
        Notification      Audit         Analytics
          Service         Service         Service
```

This demonstrates:

```text
Queue
→ Command processing
→ Competing consumers

Topic
→ Event distribution
→ Multiple independent consumers

Subscription
→ Independent consumer boundary

MaxConcurrentCalls
→ Consumer concurrency

Instances/Pods
→ Horizontal scaling

Sessions
→ Ordered processing

Retry + DLQ
→ Failure handling

Idempotency
→ Duplicate protection
```

---

# 15. Transactional Outbox — Architect-Level Question

Interviewer may ask:

> What happens if DB transaction succeeds but publishing the event fails?

Example:

```text
Save Investigation → SUCCESS
Publish InvestigationCreated → FAILED
```

Now database says investigation exists, but other services don't know.

Solution:

```text
                DB Transaction
                     │
          ┌──────────┴─────────┐
          ▼                    ▼
 Investigation             Outbox Event
    Record                    Record
                               │
                               ▼
                         Outbox Worker
                               │
                               ▼
                         Service Bus
```

This is the **Transactional Outbox Pattern**.

Very good architect-level answer.

---

# 16. Security

For production Azure architecture, remember:

```text
Managed Identity
      +
Azure RBAC
      +
Least privilege
```

Avoid embedding connection strings/secrets in application configuration where possible.

---

# 17. Interview Questions You Should Memorize

### Q1. Queue vs Topic?

> Queue is point-to-point and generally used for commands/work distribution. Topic is publish-subscribe and is used when multiple independent consumers need the same event.

### Q2. How do you scale Service Bus processing?

> I use multiple consumer instances for horizontal scaling and configure `MaxConcurrentCalls` per processor. Total concurrency is approximately instances multiplied by per-instance concurrency, subject to downstream capacity.

### Q3. How do you maintain ordering?

> I use Service Bus Sessions and assign a stable `SessionId`, such as OrderId or InvestigationId.

### Q4. What happens when message processing fails?

> With Peek-Lock, the message can be redelivered. After configured delivery attempts, it can move to the Dead Letter Queue for investigation and controlled replay.

### Q5. How do you handle duplicate messages?

> I design consumers to be idempotent using a unique business/message identifier, and I can also use Service Bus duplicate detection where appropriate.

### Q6. How do you filter events?

> I use subscription filters so each subscription receives only the events relevant to that consumer.

### Q7. What if DB succeeds but publishing event fails?

> I use the Transactional Outbox pattern to atomically persist the business change and event, then asynchronously publish the event to Service Bus.

---

# 18. 60-Second Interview Answer

> “In our enterprise architecture, I use Azure Service Bus Queue for commands and background work where multiple consumers compete for messages. For example, investigation processing can be submitted to a queue and scaled using multiple worker instances and `MaxConcurrentCalls`.
>
> For events that need to be consumed by multiple services, I use a Service Bus Topic with separate subscriptions, such as Notification, Audit, and Analytics.
>
> For reliability, I use Peek-Lock, controlled retry, Dead Letter Queue, idempotent consumers, and sessions when ordering is required. Concurrency is controlled in the consumer application, while horizontal scaling is achieved by adding instances.
>
> At the architecture level, I also consider the Transactional Outbox pattern so database updates and event publication remain reliable.”

---

# Final Revision Card

```text
AZURE SERVICE BUS
│
├── QUEUE
│    └── Command / Work
│        └── Competing Consumers
│
├── TOPIC
│    └── Event / Broadcast
│        └── Multiple Subscriptions
│
├── SUBSCRIPTION
│    └── Consumer-specific inbox
│
├── CONCURRENCY
│    └── MaxConcurrentCalls
│        ×
│       Instances
│
├── ORDERING
│    └── Sessions
│
├── RELIABILITY
│    ├── Peek-Lock
│    ├── Retry
│    ├── DLQ
│    └── Idempotency
│
├── FILTERING
│    └── Subscription Rules
│
└── ARCHITECTURE
     └── Transactional Outbox
```

### The 5 lines to remember tomorrow morning

```text
QUEUE  = DO THIS
TOPIC  = THIS HAPPENED
SUBSCRIPTION = WHO WANTS TO KNOW
SESSION = KEEP RELATED MESSAGES ORDERED
DLQ = MESSAGE COULDN'T BE PROCESSED SAFELY
```

This is the mental model I’d use for a **Technical Lead / Architect interview**.
