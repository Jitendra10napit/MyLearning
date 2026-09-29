# Event-Driven Architecture — Handwritten Notes Style

## 1. What is Event-Driven Architecture?

**EDA = Components communicate by publishing and consuming events instead of directly calling each other.**

> **Event = “Something happened.”**

Example:

```text
Order Placed
     ↓
 Order Service
     ↓
 publishes event
 "OrderPlaced"
     ↓
 ┌──────────────┬──────────────┬──────────────┐
 ↓              ↓              ↓
Payment       Inventory      Notification
Service       Service        Service
```

The **Order Service doesn't need to know** who will consume the event.

### Remember

```text
Traditional:
A → B → C

Event Driven:
A → Event → B
          → C
          → D
```

---

# 2. Simple E-commerce Example

User places an order.

### Traditional synchronous approach

```text
Client
  ↓
Order API
  ↓
Payment API
  ↓
Inventory API
  ↓
Notification API
  ↓
Response
```

Problem:

If Inventory API is slow/down:

```text
Order API
   ↓
Inventory ❌
   ↓
Request may fail / timeout
```

---

### Event-driven approach

```text
Client
  ↓
Order API
  ↓
Save Order
  ↓
Publish OrderPlaced
  ↓
Message Broker
  │
  ├──→ Payment Service
  ├──→ Inventory Service
  ├──→ Notification Service
  └──→ Analytics Service
```

Order service can finish its responsibility without waiting for every downstream service.

---

# 3. Core Components

Think:

```text
Producer → Broker → Consumer
```

### Producer

Creates/publishes event.

Example:

```csharp
await serviceBusSender.SendMessageAsync(
    new ServiceBusMessage("OrderPlaced"));
```

### Event / Message

Carries information about what happened.

```json
{
  "eventType": "OrderPlaced",
  "orderId": "ORD123",
  "customerId": "C101",
  "amount": 5000
}
```

### Broker

Stores/routes messages.

Microsoft Azure examples:

```text
Azure Service Bus
Azure Event Grid
Azure Event Hubs
```

### Consumer

Receives and processes event.

```text
OrderPlaced
     ↓
Payment Handler
     ↓
Process Payment
```

---

# 4. Event vs Command ⭐

Very important for Architect interviews.

### Command

> **"Please do something."**

```text
ProcessPayment
CreateOrder
SendEmail
```

It is an **instruction**.

### Event

> **"Something already happened."**

```text
PaymentProcessed
OrderCreated
EmailSent
```

It is a **fact**.

### Easy memory

```text
COMMAND = DO THIS

EVENT = THIS HAPPENED
```

---

# 5. Event Flow with Azure

A typical Microsoft stack:

```text
Angular
   ↓
ASP.NET Core Web API
   ↓
Order Service
   ↓
Azure Service Bus
   ↓
 ┌─────────────┬─────────────┬──────────────┐
 ↓             ↓             ↓
Payment       Inventory    Notification
Function      Function     Function
```

For example:

```text
OrderCreated
     ↓
Azure Service Bus Topic
     ↓
 ┌───────────────┬────────────────┬─────────────────┐
 ↓               ↓                ↓
Payment          Inventory        Notification
Subscription    Subscription     Subscription
```

This is where **Topic + Subscription** becomes very useful.

---

# 6. Queue vs Topic

### Queue

Generally:

```text
Producer
   ↓
 Queue
   ↓
Consumer
```

One message is processed by one competing consumer.

Useful for:

```text
Order processing
Background jobs
Work distribution
```

### Topic

```text
Producer
   ↓
 Topic
 ├── Subscription A
 ├── Subscription B
 └── Subscription C
```

Each subscription can receive its own copy.

Example:

```text
OrderPlaced
     ↓
Service Bus Topic
     │
     ├── Payment Subscription
     ├── Inventory Subscription
     └── Notification Subscription
```

---

# 7. Why Event-Driven Architecture?

### ① Loose coupling

Order service doesn't directly depend on:

```text
Payment
Inventory
Email
Analytics
```

It only knows:

```text
"I publish OrderPlaced"
```

---

### ② Scalability

Suppose:

```text
OrderPlaced = 100,000 events/hour
```

Consumers can scale independently.

```text
Payment
  2 instances → 10 instances

Inventory
  2 instances → 20 instances

Notification
  1 instance → 5 instances
```

---

### ③ Resilience

If Notification service is temporarily unavailable:

```text
OrderPlaced
    ↓
Service Bus
    ↓
Notification ❌
```

Message can remain in the broker and be processed later, depending on configuration.

---

### ④ Asynchronous processing

Producer doesn't necessarily wait for consumers.

```text
API → Publish Event → Response

             ↓
       Background processing
```

This improves API responsiveness when downstream work is non-critical to the immediate response.

---

# 8. Important Architect-Level Problem: Eventual Consistency

EDA often introduces:

> **Eventual consistency**

Example:

```text
Order DB
   ↓
Order = Created

Inventory
   ↓
may update slightly later

Payment
   ↓
may update slightly later
```

For a short period:

```text
Order = Created
Payment = Pending
Inventory = Pending
```

Eventually:

```text
Order = Confirmed
Payment = Successful
Inventory = Reserved
```

### Interview statement

> "Event-driven systems often trade immediate consistency for scalability, loose coupling and availability, so I explicitly design for eventual consistency."

---

# 9. Delivery Guarantees ⭐

Architect-level discussion should include this.

### At-most-once

```text
Message → delivered once
```

But message can potentially be lost.

### At-least-once

```text
Message → may be delivered multiple times
```

This is common in distributed messaging systems.

Therefore:

> **Consumers should be idempotent.**

---

# 10. Idempotency ⭐⭐⭐

Suppose:

```text
PaymentProcessed
```

is accidentally received twice.

Without idempotency:

```text
₹5,000 charged
₹5,000 charged again ❌
```

With idempotency:

```text
EventId = E123

First:
E123 → Process ✅

Second:
E123 → Already processed → Ignore
```

Typical implementation:

```text
ProcessedEvents
----------------
EventId
ProcessedAt
```

Before processing:

```text
if EventId already exists
       ↓
    ignore
else
       ↓
    process
       ↓
    save EventId
```

---

# 11. Retry + Dead Letter Queue

What if consumer fails?

```text
Event
 ↓
Consumer
 ↓
Failure ❌
 ↓
Retry
 ↓
Retry
 ↓
Retry
 ↓
DLQ
```

**DLQ = Dead Letter Queue**

Used for messages that cannot be successfully processed after configured retries or due to other delivery/processing conditions.

Architect should define:

```text
Retry count
Retry delay
Exponential backoff
DLQ
Alerting
Replay strategy
```

---

# 12. Event Ordering

Distributed systems may not always guarantee global ordering.

Suppose:

```text
OrderCreated
PaymentCompleted
OrderCancelled
```

If processed incorrectly:

```text
OrderCancelled
PaymentCompleted
```

can create business problems.

Solutions may include:

```text
Partitioning
Session-based ordering
Sequence numbers
Partition keys
```

depending on the messaging technology and business requirement.

---

# 13. Event Schema Evolution ⭐

Today's event:

```json
{
  "orderId": "123",
  "amount": 500
}
```

Tomorrow:

```json
{
  "orderId": "123",
  "amount": 500,
  "currency": "INR"
}
```

Don't break existing consumers.

Use:

```text
Backward compatibility
Versioning
Optional fields
Schema registry where appropriate
```

Example:

```text
OrderPlaced.v1
OrderPlaced.v2
```

---

# 14. Event Types

### Domain Event

Business event.

```text
OrderPlaced
PaymentCompleted
CustomerRegistered
```

### Integration Event

Used to communicate between services.

```text
OrderCreatedIntegrationEvent
```

### Technical Event

Infrastructure/system event.

```text
FileUploaded
ServiceStarted
JobCompleted
```

---

# 15. Event-Driven vs Request/Response

| Request/Response     | Event Driven                   |
| -------------------- | ------------------------------ |
| Direct communication | Indirect communication         |
| Usually synchronous  | Usually asynchronous           |
| Tighter coupling     | Loose coupling                 |
| Caller waits         | Caller may not wait            |
| Simple flow          | More distributed               |
| Easier debugging     | Requires tracing/observability |
| Immediate response   | Often eventual consistency     |

---

# 16. Important Patterns ⭐

As Staff/Architect, remember these:

```text
EDA
 │
 ├── Pub/Sub
 ├── Queue
 ├── Event Sourcing
 ├── CQRS
 ├── Saga
 ├── Outbox Pattern
 ├── Retry
 ├── DLQ
 └── Idempotency
```

### Outbox Pattern

Very important.

Problem:

```text
Save Order DB
      ↓
Publish Event
```

What if DB succeeds but publishing fails?

```text
DB ✅
Event ❌
```

Now system is inconsistent.

Outbox:

```text
Transaction
   │
   ├── Order
   └── Outbox Event
          ↓
       DB Commit
          ↓
   Background Publisher
          ↓
      Message Broker
```

This gives reliable event publishing.

---

# 17. Saga Pattern

Useful for distributed transactions.

Example:

```text
Order
 ↓
Payment
 ↓
Inventory
 ↓
Shipping
```

If Inventory fails:

```text
Inventory ❌
   ↓
Compensating action
   ↓
Refund Payment
   ↓
Cancel Order
```

Remember:

> **Saga = distributed transaction using local transactions + compensating actions.**

---

# 18. Event Sourcing

Don't confuse this with normal EDA.

Normal approach:

```text
Order Table
-----------
OrderId
Status = Shipped
```

Event sourcing:

```text
OrderCreated
PaymentCompleted
ItemPacked
OrderShipped
```

Current state is derived from events.

```text
Events
  ↓
Replay
  ↓
Current State
```

---

# 19. CQRS + EDA

Very common architecture:

```text
Command
   ↓
Command Handler
   ↓
Write DB
   ↓
Event
   ↓
Message Broker
   ↓
Read Model Updater
   ↓
Read DB
```

Example:

```text
CreateOrder
    ↓
Order Service
    ↓
OrderCreated
    ↓
Service Bus
    ↓
Search/Read Model
```

CQRS and EDA **can work together**, but they are not the same thing.

---

# 20. Staff/Architect-Level Checklist ⭐⭐⭐

When designing EDA, don't just say:

> "We'll use Service Bus."

Discuss:

```text
1. Event definition
2. Producer
3. Consumer
4. Queue vs Topic
5. Delivery guarantee
6. Idempotency
7. Retry strategy
8. DLQ
9. Ordering
10. Partitioning
11. Event schema/versioning
12. Duplicate handling
13. Observability
14. Correlation ID
15. Trace ID
16. Security
17. Replay strategy
18. Event retention
19. Disaster recovery
20. Eventual consistency
```

---

# 🧠 Final 30-Second Interview Answer

> **"Event-driven architecture is a style where services communicate through events rather than directly calling each other. A producer publishes an event such as OrderPlaced to a broker like Azure Service Bus, and multiple consumers independently process that event. This provides loose coupling, scalability and resilience, but introduces challenges such as eventual consistency, duplicate messages, ordering, retries and debugging. So in a production design I would consider idempotent consumers, retry with DLQ, correlation IDs, schema versioning, the Outbox Pattern for reliable publishing, and Saga for distributed business transactions."**

### One-line memory trick

```text
EVENT DRIVEN =
Producer → Event → Broker → Consumers
              +
   Retry + DLQ + Idempotency
              +
 Eventual Consistency + Observability
```

This is the level of framing I'd use for a **Senior Dev / Staff / Architect** interview.
