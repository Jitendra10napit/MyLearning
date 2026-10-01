Think of **Clean Architecture** as a way of organizing code so that:

> **Business rules stay at the center, while technical details like database, Azure, APIs, and UI stay outside.**

The most important mental model is:

```text
OUTSIDE → depends on → INSIDE

UI / API
   ↓
Application
   ↓
Domain

Infrastructure → implements interfaces defined inside
```

## 1. Simple e-commerce example

Imagine we are building an **Order application**.

A customer says:

> "Place an order for 2 laptops."

We can divide the application into 4 parts:

```text
┌──────────────────────────────────────┐
│          Presentation / API          │
│      OrdersController / REST        │
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│          Application Layer           │
│       PlaceOrderCommandHandler       │
│       GetOrderQueryHandler            │
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│             Domain Layer             │
│       Order / OrderItem / Rules      │
│       Business logic                 │
└──────────────────┬───────────────────┘
                   ↑
┌──────────────────────────────────────┐
│         Infrastructure Layer         │
│ EF Core / SQL / Service Bus / Azure  │
└──────────────────────────────────────┘
```

The key thing is that **Infrastructure implements contracts required by the inner layers**.

---

# 2. What goes into each layer?

### ① Domain — "What are my business rules?"

Example:

```csharp
public class Order
{
    public int Id { get; set; }
    public decimal TotalAmount { get; set; }

    public void Confirm()
    {
        if (TotalAmount <= 0)
            throw new InvalidOperationException("Invalid order");

        // business rule
    }
}
```

The Domain shouldn't care about:

```text
SQL Server
Azure
HTTP
EF Core
Service Bus
Angular
```

It only knows the **business**.

---

# 3. Application — "What does the system need to do?"

Suppose the requirement is:

> Place an order.

You could have:

```csharp
public class PlaceOrderHandler
{
    private readonly IOrderRepository _repository;

    public async Task Handle(PlaceOrderCommand command)
    {
        var order = new Order();

        // application orchestration

        order.Confirm();

        await _repository.AddAsync(order);
    }
}
```

The Application layer coordinates the use case.

Think:

```text
Application = Use Cases
```

Examples:

```text
PlaceOrder
CancelOrder
GetOrder
UpdateOrder
```

---

# 4. Infrastructure — "How do we technically do it?"

Now we need to save the order.

Application says:

```csharp
IOrderRepository
```

Infrastructure provides:

```csharp
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;

    public async Task AddAsync(Order order)
    {
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();
    }
}
```

Here:

```text
Application
     ↓
IOrderRepository
     ↑
OrderRepository
     ↓
EF Core
     ↓
SQL Server
```

This is the important Clean Architecture concept:

> **The Application doesn't need to know that SQL Server or EF Core exists.**

---

# 5. Presentation — "How does the outside world talk to us?"

For example:

```csharp
[HttpPost]
public async Task<IActionResult> Create(CreateOrderRequest request)
{
    await _mediator.Send(
        new PlaceOrderCommand(request.CustomerId));

    return Ok();
}
```

The controller receives HTTP.

It calls the Application layer.

```text
HTTP Request
     ↓
Controller
     ↓
Application
     ↓
Domain
     ↓
Repository
     ↓
SQL
```

---

# 6. The most important concept: Dependency Direction

This is what interviewers really want to hear.

A common misunderstanding is:

```text
API → Application → Domain → Infrastructure
```

and then saying:

> "Everything depends on the next layer."

That's incomplete.

The **dependency rule** is:

```text
          ┌─────────────┐
          │   Domain    │
          └──────▲──────┘
                 │
          ┌──────┴──────┐
          │ Application │
          └──────▲──────┘
                 │
       ┌─────────┴─────────┐
       │                   │
 Presentation       Infrastructure
```

The inner layers don't depend on the outer technical details.

---

# 7. Why do we need interfaces?

This is where many people get confused.

Suppose Application needs to save an order.

Instead of:

```csharp
Application → SqlOrderRepository
```

we do:

```csharp
Application → IOrderRepository
```

Infrastructure implements it:

```csharp
SqlOrderRepository : IOrderRepository
```

So:

```text
             Application
                  |
                  ↓
         IOrderRepository
                  ↑
                  |
       SqlOrderRepository
                  |
                  ↓
              EF Core
                  |
                  ↓
             SQL Server
```

This is **Dependency Inversion**.

---

# 8. Dependency Injection connects everything

Now you're probably thinking:

> "If Application only knows `IOrderRepository`, who gives it the actual repository?"

That's where **Dependency Injection** comes in.

In `Program.cs`:

```csharp
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
```

Now when:

```csharp
public PlaceOrderHandler(
    IOrderRepository repository)
```

is created, .NET injects:

```text
IOrderRepository
       ↓
OrderRepository
```

So the complete flow becomes:

```text
HTTP
 ↓
Controller
 ↓
PlaceOrderHandler
 ↓
IOrderRepository
 ↑
OrderRepository
 ↓
EF Core
 ↓
SQL Server
```

---

# 9. Now add CQRS

This is where your previous architecture discussion connects.

Inside the **Application layer**, you can use CQRS:

```text
Application
│
├── Commands
│   ├── PlaceOrder
│   ├── CancelOrder
│   └── UpdateOrder
│
└── Queries
    ├── GetOrder
    └── SearchOrders
```

So CQRS isn't replacing Clean Architecture.

It sits **inside the Application layer**.

---

# 10. Now add Azure Service Bus

Suppose after placing an order, we need to send a notification.

Instead of:

```text
Order API
   ↓
Notification API
```

we could do:

```text
Order Application
       ↓
Service Bus
       ↓
Notification Service
```

Now we're combining architectural approaches:

```text
Clean Architecture
        +
CQRS
        +
Microservices
        +
Event-Driven Architecture
```

They solve **different problems**.

---

# 11. Your complete mental model

Remember this picture:

```text
                   CLIENT
                     │
                     ▼
              ┌─────────────┐
              │ API / BFF   │
              │ Presentation│
              └──────┬──────┘
                     │
                     ▼
        ┌─────────────────────────┐
        │      APPLICATION        │
        │                         │
        │ Commands / Queries      │
        │ Handlers / Use Cases    │
        └───────────┬─────────────┘
                    │
                    ▼
        ┌─────────────────────────┐
        │         DOMAIN          │
        │                         │
        │ Entities                │
        │ Business Rules          │
        │ Value Objects           │
        └───────────▲─────────────┘
                    │
             Interfaces
                    │
        ┌───────────┴─────────────┐
        │     INFRASTRUCTURE      │
        │                         │
        │ EF Core                 │
        │ SQL Server              │
        │ Azure Service Bus       │
        │ Blob Storage            │
        │ External APIs           │
        └─────────────────────────┘
```

And the dependency direction to remember is:

> **Presentation → Application → Domain**

while:

> **Infrastructure implements interfaces needed by Application/Domain.**

---

# 12. How to explain it in your interview

I'd give this answer:

> **"Clean Architecture is an architectural approach where we organize the application around business capabilities and keep the core business logic independent of technical details."**
>
> **"I typically separate it into Presentation, Application, Domain and Infrastructure layers. Presentation handles HTTP/API concerns. Application contains use cases and orchestration, such as commands and queries. Domain contains entities and business rules. Infrastructure contains technical implementations such as EF Core, SQL Server, Azure Service Bus and external integrations."**
>
> **"The important principle is dependency direction. The inner layers shouldn't depend on infrastructure details. For example, the Application layer depends on an `IOrderRepository` abstraction, while Infrastructure provides the concrete `OrderRepository` using EF Core. Dependency Injection wires them together."**
>
> **"This gives us separation of concerns, testability and the ability to change infrastructure without changing the core business logic. We can then combine Clean Architecture with other approaches—for example, CQRS inside the Application layer, microservices at the system level, and Azure Service Bus for asynchronous communication between services."**

### ⭐ One sentence to memorize

> **"Clean Architecture protects the business logic from technical details; the outer layers depend on the inner layers, while infrastructure implements the abstractions defined by the core."**

That sentence will immediately establish the right mental model in an Architect interview.
