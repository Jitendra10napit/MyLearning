## Saga Pattern — Microservice Architecture

### Simple definition

> **Saga Pattern manages a business transaction that spans multiple microservices by breaking it into a sequence of local transactions. If one step fails, the Saga executes compensating actions to undo the business effect of the previous successful steps.**

The key idea:

```text
One big distributed transaction
             ↓
      Multiple local transactions
             ↓
   Success → Continue
   Failure → Compensate
```

---

# 1. Real-world example: E-commerce Order

Suppose we have:

```text
Customer places order for a laptop
```

The operation involves multiple microservices:

```text
                Order Service
                     |
                     v
              Payment Service
                     |
                     v
             Inventory Service
                     |
                     v
             Shipping Service
```

We don't want a single database transaction across all these services.

Instead, Saga coordinates them.

---

# 2. Saga flow

Imagine order processing requires:

1. Create Order
2. Reserve Payment
3. Reserve Inventory
4. Create Shipment

### Successful flow

```text
Order Service
     |
     | Create Order
     v
Order Created
     |
     | Reserve Payment
     v
Payment Reserved
     |
     | Reserve Inventory
     v
Inventory Reserved
     |
     | Create Shipment
     v
Shipment Created
     |
     v
ORDER CONFIRMED ✅
```

Each service performs its **own local database transaction**.

---

# 3. What happens when something fails?

Suppose:

```text
Order Created          ✅
Payment Reserved       ✅
Inventory Reserved     ❌
```

Now we have a problem.

Payment has already been reserved.

The Saga executes a **compensating transaction**:

```text
Order Created          ✅
       ↓
Payment Reserved       ✅
       ↓
Inventory Reservation  ❌
       ↓
Release Payment        ↩️
       ↓
Cancel Order           ↩️
```

So:

```text
Forward transaction:
Create Order
     ↓
Reserve Payment
     ↓
Reserve Inventory ❌

Compensation:
Release Payment
     ↓
Cancel Order
```

### Important

Compensation is **not a database rollback**.

It's a new business operation that reverses the previous business effect.

---

# 4. Two ways to implement Saga

There are two common approaches.

```text
1. Choreography
2. Orchestration
```

---

# 5. Choreography-based Saga

There is **no central coordinator**.

Services communicate using events.

For example, using **Azure Service Bus**:

```text
Order Service
     |
     | OrderCreated
     v
Azure Service Bus Topic
     |
     +----------------------+
     |                      |
     v                      v
Payment Service       Other Services
     |
     | PaymentReserved
     v
Azure Service Bus
     |
     v
Inventory Service
     |
     | InventoryReserved
     v
Azure Service Bus
     |
     v
Shipping Service
```

### Failure scenario

```text
OrderCreated
     ↓
PaymentReserved
     ↓
InventoryReservationFailed
     ↓
Payment Service receives event
     ↓
ReleasePayment
```

The services react to events.

### Advantages

* Loose coupling
* No central coordinator
* Good for event-driven architectures
* Services are independently deployable

### Disadvantages

As the number of services grows:

```text
Service A → B
     ↓      ↓
     C ← D
     ↓
     E
```

The flow can become difficult to understand and troubleshoot.

This is sometimes called **event spaghetti**.

---

# 6. Orchestration-based Saga

Here we introduce a **Saga Orchestrator**.

```text
                 Saga Orchestrator
                        |
        +---------------+---------------+
        |               |               |
        v               v               v
   Order Service   Payment Service  Inventory Service
                                        |
                                        v
                                  Shipping Service
```

The orchestrator knows the workflow.

### Example

```text
Saga Orchestrator
       |
       | 1. Create Order
       v
Order Service
       |
       | Success
       v
Saga Orchestrator
       |
       | 2. Reserve Payment
       v
Payment Service
       |
       | Success
       v
Saga Orchestrator
       |
       | 3. Reserve Inventory
       v
Inventory Service
       |
       | FAILURE ❌
       v
Saga Orchestrator
       |
       | Compensate
       v
Payment Service
       |
       | Release Payment
       v
Order Service
       |
       | Cancel Order
       v
Saga FAILED
```

This is often easier to manage for **complex business workflows**.

---

# 7. Simple C# example

You could have commands like:

```csharp
public record CreateOrderCommand(Guid OrderId);

public record ReservePaymentCommand(
    Guid OrderId,
    decimal Amount);

public record ReserveInventoryCommand(
    Guid OrderId,
    Guid ProductId);

public record ReleasePaymentCommand(
    Guid OrderId);
```

The orchestrator could conceptually look like:

```csharp
public async Task ProcessOrderAsync(Order order)
{
    try
    {
        await _orderService.CreateOrder(order);

        await _paymentService.ReservePayment(
            order.Id,
            order.Amount);

        await _inventoryService.ReserveInventory(
            order.Id,
            order.ProductId);

        await _shippingService.CreateShipment(
            order.Id);

        await _orderService.ConfirmOrder(order.Id);
    }
    catch (Exception)
    {
        await _paymentService.ReleasePayment(order.Id);

        await _inventoryService.ReleaseInventory(order.Id);

        await _orderService.CancelOrder(order.Id);

        throw;
    }
}
```

**Production implementation would need more than this**, particularly persistent Saga state, idempotency, retries, timeouts, durable messaging and handling partial compensation failures.

---

# 8. Azure Service Bus implementation

Since you're preparing for Azure/microservices interviews, this is a good architecture to explain:

```text
                  ┌───────────────────┐
                  │   Order Service   │
                  └─────────┬─────────┘
                            |
                     OrderCreated
                            |
                            v
                ┌─────────────────────┐
                │ Azure Service Bus   │
                │      Topic          │
                └─────────┬───────────┘
                          |
                          v
                ┌─────────────────────┐
                │ Saga Orchestrator   │
                └──────┬──────┬───────┘
                       |      |
             ReservePayment   |
                       |      |
                       v      |
               Payment Service|
                              |
                       ReserveInventory
                              |
                              v
                    Inventory Service
                              |
                         failure
                              |
                              v
                    Compensating Action
                              |
                              v
                    Release Payment
```

You can store Saga state in something like SQL/Cosmos DB:

```text
SagaId
OrderId
CurrentState
PaymentStatus
InventoryStatus
ShippingStatus
LastProcessedEvent
CreatedAt
UpdatedAt
```

For example:

```text
SagaId:       SAGA-1001
OrderId:      ORD-5001

Order:        CREATED
Payment:      RESERVED
Inventory:    FAILED
Compensation: PAYMENT_RELEASED
Status:       FAILED
```

This is important because the orchestrator needs to **resume/recover after a crash**.

---

# 9. Saga vs traditional database transaction

### Traditional transaction

```text
BEGIN TRANSACTION

Order DB
Payment DB
Inventory DB

COMMIT
```

This doesn't work well when these are separate microservices/databases.

Saga:

```text
Order DB       → COMMIT
Payment DB     → COMMIT
Inventory DB   → COMMIT
Shipping DB    → COMMIT
```

If something fails:

```text
Compensating transactions
```

So Saga provides **eventual consistency**, rather than traditional ACID consistency across all services.

---

# 10. Important interview question: "What if compensation fails?"

This is where you can demonstrate architect-level understanding.

Suppose:

```text
Inventory ❌
     ↓
Release Payment ❌
```

You cannot simply ignore this.

You need:

```text
Retry
  ↓
Persistent Saga State
  ↓
Idempotent Compensation
  ↓
Dead Letter Queue if necessary
  ↓
Alert / Monitoring
  ↓
Manual intervention if business-critical
```

For example:

```text
Payment Release failed
        ↓
Retry 1 ❌
Retry 2 ❌
Retry 3 ❌
        ↓
Azure Service Bus DLQ
        ↓
Alert / Operational dashboard
```

The compensation operation should be **idempotent**:

```text
ReleasePayment(PAY123)
ReleasePayment(PAY123)
ReleasePayment(PAY123)
```

The final business state should still be:

```text
PAY123 = RELEASED
```

not multiple releases.

---

# 11. Saga + Retry + Circuit Breaker

These patterns work together.

```text
                  Saga
                   |
          ┌────────┴────────┐
          ↓                 ↓
       Payment          Inventory
          |
       Timeout
          ↓
        Retry
          ↓
     Circuit Breaker
          ↓
       Still failing
          ↓
      Compensation
```

For example:

```text
Reserve Payment
      |
      ├── Retry → temporary failure
      |
      ├── Circuit Breaker → Payment unhealthy
      |
      └── Saga → execute compensation
```

So remember:

| Pattern             | Purpose                                   |
| ------------------- | ----------------------------------------- |
| **Saga**            | Manage distributed business transaction   |
| **Retry**           | Handle temporary failure                  |
| **Circuit Breaker** | Stop calling unhealthy service            |
| **Idempotency**     | Prevent duplicate processing              |
| **DLQ**             | Isolate messages that cannot be processed |
| **Outbox**          | Reliably publish DB changes as events     |

---

# 12. Best Technical Lead interview answer

If interviewer asks **"Explain Saga Pattern with a microservice example"**, say:

> **"Saga is a distributed transaction pattern used when a business transaction spans multiple microservices. Instead of using one distributed database transaction, we break the workflow into local transactions. For example, in an e-commerce system, Order Service creates the order, Payment Service reserves payment, Inventory Service reserves stock and Shipping Service creates the shipment. If all steps succeed, the order is confirmed. If inventory reservation fails after payment was already reserved, the Saga executes a compensating transaction such as releasing the payment and cancelling the order. Saga can be implemented using choreography, where services communicate through events, or orchestration, where a central Saga orchestrator controls the workflow. In production I would combine it with durable messaging, idempotency, retries, circuit breakers, persistent Saga state and observability."**

### One-line memory trick

```text
Saga = DO STEPS → FAILURE → UNDO BUSINESS EFFECTS
```

And the most important distinction:

> **Saga doesn't "rollback databases"; it performs compensating business transactions.**


----------------------------------------------xxxx---------------------------------
Absolutely. For interview recall, use **one simple e-commerce example** for both types. The business flow is identical; **only who controls the flow changes**.

# Saga Pattern — easiest way to remember

Business flow:

```text
Create Order
    ↓
Reserve Payment
    ↓
Reserve Inventory
    ↓
Create Shipment
    ↓
Order Confirmed
```

If Inventory fails:

```text
Inventory ❌
    ↓
Release Payment
    ↓
Cancel Order
```

The difference:

```text
CHOREOGRAPHY = Services decide what to do from EVENTS

ORCHESTRATION = Orchestrator decides what to do
```

---

# 1. Choreography Saga

### Easy definition

> **There is no central controller. Each service listens for an event and performs its action, then publishes the next event.**

Think:

> **"Event comes → Service acts → Service publishes next event."**

### Architecture

```text
Order Service
     |
     | OrderCreated
     ↓
Azure Service Bus
     |
     ↓
Payment Service
     |
     | PaymentReserved
     ↓
Azure Service Bus
     |
     ↓
Inventory Service
     |
     | InventoryReserved
     ↓
Azure Service Bus
     |
     ↓
Shipping Service
```

### Implementation logic

#### Step 1 — Order Service

```csharp
public async Task CreateOrder(Order order)
{
    await _orderRepository.Save(order);

    await _bus.PublishAsync(
        new OrderCreated(order.Id, order.Amount));
}
```

Order Service doesn't call Payment directly.

It just says:

```text
"OrderCreated"
```

---

### Step 2 — Payment Service

Payment listens for `OrderCreated`.

```csharp
public async Task Handle(OrderCreated message)
{
    await _paymentService.Reserve(
        message.OrderId,
        message.Amount);

    await _bus.PublishAsync(
        new PaymentReserved(message.OrderId));
}
```

Now Payment says:

```text
"PaymentReserved"
```

---

### Step 3 — Inventory Service

```csharp
public async Task Handle(PaymentReserved message)
{
    bool success =
        await _inventoryService.Reserve(message.OrderId);

    if (success)
    {
        await _bus.PublishAsync(
            new InventoryReserved(message.OrderId));
    }
    else
    {
        await _bus.PublishAsync(
            new InventoryReservationFailed(message.OrderId));
    }
}
```

---

### Step 4 — Shipping Service

If inventory succeeds:

```csharp
public async Task Handle(InventoryReserved message)
{
    await _shippingService.CreateShipment(
        message.OrderId);

    await _bus.PublishAsync(
        new OrderConfirmed(message.OrderId));
}
```

---

## What if Inventory fails?

This is the important part.

```text
OrderCreated
     ↓
PaymentReserved
     ↓
InventoryReservationFailed ❌
     ↓
Payment Service receives event
     ↓
ReleasePayment
     ↓
PaymentReleased
     ↓
Order Service receives event
     ↓
CancelOrder
```

Implementation:

```csharp
public async Task Handle(
    InventoryReservationFailed message)
{
    await _paymentService.Release(
        message.OrderId);

    await _bus.PublishAsync(
        new PaymentReleased(message.OrderId));
}
```

Then Order Service:

```csharp
public async Task Handle(
    PaymentReleased message)
{
    await _orderService.Cancel(
        message.OrderId);
}
```

### Remember Choreography

```text
EVENT → SERVICE → EVENT → SERVICE → EVENT
```

There is **no boss**.

Each service reacts to events.

---

# 2. Orchestration Saga

Now use exactly the **same business process**.

But introduce a:

```text
Saga Orchestrator
```

The orchestrator is the **boss/coordinator**.

### Architecture

```text
                 Saga Orchestrator
                        |
             ┌──────────┼──────────┐
             ↓          ↓          ↓
           Order      Payment   Inventory
          Service     Service    Service
                                  |
                                  ↓
                              Shipping
```

### Implementation logic

The orchestrator controls the sequence.

```csharp
public async Task ProcessOrder(Order order)
{
    try
    {
        // 1
        await _orderService.CreateOrder(order);

        // 2
        await _paymentService.ReservePayment(
            order.Id,
            order.Amount);

        // 3
        await _inventoryService.ReserveInventory(
            order.Id);

        // 4
        await _shippingService.CreateShipment(
            order.Id);

        // 5
        await _orderService.ConfirmOrder(
            order.Id);
    }
    catch
    {
        // Compensation
        await _paymentService.ReleasePayment(
            order.Id);

        await _orderService.CancelOrder(
            order.Id);

        throw;
    }
}
```

For a real distributed system, these calls would generally be **durable commands/messages**, rather than a single in-memory method containing the whole workflow.

---

# 3. Orchestration with messages

A more realistic Azure Service Bus approach:

```text
                 Saga Orchestrator
                       |
             ReservePaymentCommand
                       ↓
                Payment Service
                       |
                 PaymentReserved
                       ↓
                 Saga Orchestrator
                       |
             ReserveInventoryCommand
                       ↓
               Inventory Service
                       |
               InventoryFailed
                       ↓
                 Saga Orchestrator
                       |
              ReleasePaymentCommand
                       ↓
                Payment Service
                       |
                PaymentReleased
                       ↓
                 Saga Orchestrator
                       |
                CancelOrderCommand
                       ↓
                  Order Service
```

Notice the difference:

### Choreography

```text
Payment Service
      |
      ↓
"PaymentReserved"
      |
      ↓
Inventory Service reacts
```

### Orchestration

```text
Payment Service
      |
      ↓
"PaymentReserved"
      |
      ↓
Orchestrator decides
      |
      ↓
"Now call Inventory"
```

---

# 4. Side-by-side — easiest interview recall

|                        | **Choreography**                 | **Orchestration**                  |
| ---------------------- | -------------------------------- | ---------------------------------- |
| Controller             | ❌ No central controller          | ✅ Saga Orchestrator                |
| Communication          | Events                           | Commands + events                  |
| Who decides next step? | Individual services              | Orchestrator                       |
| Coupling               | Looser initially                 | More centralized                   |
| Simple workflows       | ✅ Very good                      | ✅ Good                             |
| Complex workflows      | Can become difficult             | ✅ Easier to manage                 |
| Failure handling       | Services react to failure events | Orchestrator triggers compensation |
| Main memory trick      | **"React to event"**             | **"Orchestrator commands"**        |

---

# 5. The same failure in both

### Choreography

```text
OrderCreated
     ↓
PaymentReserved
     ↓
InventoryFailed
     ↓
Payment Service reacts
     ↓
ReleasePayment
     ↓
Order Service reacts
     ↓
CancelOrder
```

### Orchestration

```text
Orchestrator
     ↓
CreateOrder
     ↓
ReservePayment
     ↓
ReserveInventory ❌
     ↓
Orchestrator knows failure
     ↓
ReleasePayment
     ↓
CancelOrder
```

---

# 6. The easiest way to explain to interviewer

Say this:

> **"Saga can be implemented in two ways: choreography and orchestration. In choreography, there is no central coordinator. Services communicate through events. For example, OrderCreated triggers Payment Service, PaymentReserved triggers Inventory Service, and InventoryReserved triggers Shipping Service. If inventory fails, the failure event causes Payment Service to release the payment and Order Service to cancel the order.**
>
> **In orchestration, a Saga Orchestrator controls the workflow. It sends commands such as CreateOrder, ReservePayment, ReserveInventory and CreateShipment. If a step fails, the orchestrator knows which previous steps succeeded and sends compensating commands such as ReleasePayment and CancelOrder."**

---

# 7. One picture to memorize before interview

```text
CHOREOGRAPHY
=============

 Order
   |
   | EVENT
   ↓
Payment
   |
   | EVENT
   ↓
Inventory
   |
   | EVENT
   ↓
Shipping

"No boss — everyone reacts."


ORCHESTRATION
=============

             ORCHESTRATOR
             /     |      \
            ↓      ↓       ↓
         Order  Payment  Inventory
                            |
                            ↓
                         Shipping

"One boss — orchestrator decides."
```

### 🔥 Final memory trick

**Choreography = `EVENT → REACT → EVENT`**

**Orchestration = `COMMAND → DO → RESULT → COMMAND`**

And **Saga itself = `FORWARD STEPS + COMPENSATING STEPS`**.
