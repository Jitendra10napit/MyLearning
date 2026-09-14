# SOLID Principles — Senior Developer / Architect / Staff Engineer Notes

SOLID is not just **5 coding rules**. At senior/architect level, think of SOLID as a way to control **coupling, change, extensibility, testability, and architectural boundaries**.

A memorable way:

> **S** — One reason to change
> **O** — Add behavior without breaking existing behavior
> **L** — Subtypes must be safely replaceable
> **I** — Don't force clients to depend on things they don't need
> **D** — Depend on abstractions, not implementation details

---

# 1. S — Single Responsibility Principle (SRP)

### Basic definition

> A class/module should have **one reason to change**.

A common misunderstanding is:

❌ "A class should have only one method."

That's not SRP.

Instead ask:

> **Who could ask me to change this code?**

If multiple independent business/technical responsibilities can cause the class to change, SRP is probably violated.

---

## ❌ Bad example

```csharp
public class OrderService
{
    public void CreateOrder(Order order)
    {
        // Create order
    }

    public void CalculateTax(Order order)
    {
        // Tax calculation
    }

    public void SendEmail(Order order)
    {
        // Send email
    }

    public void SaveToDatabase(Order order)
    {
        // Database logic
    }

    public void GenerateInvoice(Order order)
    {
        // PDF generation
    }
}
```

This class has many reasons to change:

```text
Business rules
      ↓
Tax changes
      ↓
Email provider changes
      ↓
Database changes
      ↓
Invoice format changes
```

That's too much responsibility.

---

## ✅ Better design

```csharp
public class OrderService
{
    private readonly ITaxCalculator _taxCalculator;
    private readonly IOrderRepository _repository;
    private readonly INotificationService _notificationService;
    private readonly IInvoiceGenerator _invoiceGenerator;

    public OrderService(
        ITaxCalculator taxCalculator,
        IOrderRepository repository,
        INotificationService notificationService,
        IInvoiceGenerator invoiceGenerator)
    {
        _taxCalculator = taxCalculator;
        _repository = repository;
        _notificationService = notificationService;
        _invoiceGenerator = invoiceGenerator;
    }

    public async Task CreateOrder(Order order)
    {
        order.Tax = _taxCalculator.Calculate(order);

        await _repository.Save(order);

        await _invoiceGenerator.Generate(order);

        await _notificationService.SendOrderCreated(order);
    }
}
```

Now responsibilities are separated:

```text
OrderService
   │
   ├── ITaxCalculator
   ├── IOrderRepository
   ├── IInvoiceGenerator
   └── INotificationService
```

---

# Senior-level interpretation of SRP

At architecture level:

```text
SRP
 ↓
Separate responsibilities
 ↓
Separate reasons for change
 ↓
Separate modules/components
 ↓
Lower coupling
 ↓
Easier testing
 ↓
Safer deployments
```

For example, if tax rules change:

```text
TaxCalculator changes
```

not:

```text
OrderService
Database
Email
Invoice
API
```

### Architect question

Ask:

> "If this requirement changes, how many unrelated components will I have to modify?"

If the answer is "many", your boundaries may be wrong.

---

# 2. O — Open/Closed Principle (OCP)

> Software entities should be **open for extension but closed for modification**.

Meaning:

```text
Existing stable code
       ↓
should ideally not be modified
       ↓
when adding a new variation
```

Instead:

```text
New behavior
     ↓
New implementation
     ↓
Existing system remains stable
```

---

## ❌ Bad example

```csharp
public class PaymentService
{
    public void Pay(string type, decimal amount)
    {
        if (type == "CreditCard")
        {
            // Credit card
        }
        else if (type == "UPI")
        {
            // UPI
        }
        else if (type == "PayPal")
        {
            // PayPal
        }
    }
}
```

Tomorrow:

```text
CreditCard
UPI
PayPal
ApplePay
GooglePay
Crypto
BankTransfer
```

The class keeps growing.

---

## ✅ OCP approach

```csharp
public interface IPaymentProcessor
{
    void Pay(decimal amount);
}
```

Implementations:

```csharp
public class CreditCardPayment : IPaymentProcessor
{
    public void Pay(decimal amount)
    {
        // Credit card payment
    }
}
```

```csharp
public class UpiPayment : IPaymentProcessor
{
    public void Pay(decimal amount)
    {
        // UPI payment
    }
}
```

```csharp
public class PayPalPayment : IPaymentProcessor
{
    public void Pay(decimal amount)
    {
        // PayPal payment
    }
}
```

Service:

```csharp
public class PaymentService
{
    public void Process(
        IPaymentProcessor processor,
        decimal amount)
    {
        processor.Pay(amount);
    }
}
```

Adding:

```csharp
public class ApplePayPayment : IPaymentProcessor
{
    public void Pay(decimal amount)
    {
        // Apple Pay
    }
}
```

doesn't require modifying `PaymentService`.

---

# OCP at architecture level

OCP is closely related to:

* Strategy Pattern
* Factory Pattern
* Plugin architecture
* Dependency Injection
* Polymorphism
* Event-driven architecture
* Extension points

For example:

```text
                    IPaymentProcessor
                           │
            ┌──────────────┼──────────────┐
            ↓              ↓              ↓
       CreditCard         UPI          PayPal
```

The core system doesn't care which implementation is used.

---

# Important Staff-level nuance

**Don't apply OCP everywhere.**

This is a common senior-level mistake.

You shouldn't create:

```text
IUserService
IUserServiceFactory
IUserServiceStrategy
IUserServiceProvider
IUserServiceResolver
```

just because "SOLID".

Ask:

> **Is this area expected to have meaningful variation?**

If there is only one implementation and no foreseeable variation, an abstraction may simply add complexity.

### Remember:

> **SOLID is about managing change, not maximizing interfaces.**

---

# 3. L — Liskov Substitution Principle (LSP)

This is one of the most misunderstood SOLID principles.

> If `B` is a subtype of `A`, code using `A` should be able to use `B` without unexpected behavior.

Simple version:

> **Child should behave like a valid replacement for parent.**

---

## ❌ Classic violation

```csharp
public class Bird
{
    public virtual void Fly()
    {
        Console.WriteLine("Flying");
    }
}
```

```csharp
public class Penguin : Bird
{
    public override void Fly()
    {
        throw new NotSupportedException();
    }
}
```

Now:

```csharp
void MakeBirdFly(Bird bird)
{
    bird.Fly();
}
```

This works:

```csharp
MakeBirdFly(new Eagle());
```

But fails:

```csharp
MakeBirdFly(new Penguin());
```

Therefore:

```text
Bird
 ↓
assumes Fly()
 ↓
Penguin cannot honor that contract
```

Inheritance is incorrect.

---

## ✅ Better design

```csharp
public abstract class Bird
{
}

public interface IFlyingBird
{
    void Fly();
}
```

```csharp
public class Eagle : Bird, IFlyingBird
{
    public void Fly()
    {
        Console.WriteLine("Flying");
    }
}
```

```csharp
public class Penguin : Bird
{
}
```

Now:

```text
Bird
├── Eagle → IFlyingBird
└── Penguin
```

The abstraction accurately represents capabilities.

---

# LSP isn't only about inheritance

This is important for architecture interviews.

LSP also applies to:

### APIs

Suppose:

```text
GET /users
```

returns:

```json
[
   { "id": 1, "name": "John" }
]
```

A replacement implementation suddenly returns:

```json
{
   "users": [...]
}
```

The replacement breaks consumers.

That's effectively an LSP-style contract violation.

---

### Microservices

Suppose:

```text
Payment Service v1
```

has contract:

```text
POST /payment
→ 200
```

A replacement implementation changes semantics:

```text
POST /payment
→ 202
```

and consumers weren't designed for that behavior.

The abstraction/contract was not safely substitutable.

---

# Senior-level LSP question

Don't ask only:

> "Does it inherit?"

Ask:

> **"Can consumers safely treat this implementation as the abstraction promises?"**

That's the real LSP question.

---

# 4. I — Interface Segregation Principle (ISP)

> Clients should not be forced to depend on methods they don't use.

---

## ❌ Bad interface

```csharp
public interface IEmployee
{
    void Work();
    void Eat();
    void Sleep();
    void AttendMeeting();
}
```

Now suppose:

```csharp
public class Robot : IEmployee
{
    public void Work()
    {
    }

    public void Eat()
    {
        throw new NotSupportedException();
    }

    public void Sleep()
    {
        throw new NotSupportedException();
    }

    public void AttendMeeting()
    {
    }
}
```

Why should a robot implement:

```text
Eat()
Sleep()
```

if it doesn't need them?

---

## ✅ Segregated interfaces

```csharp
public interface IWorker
{
    void Work();
}
```

```csharp
public interface IHuman
{
    void Eat();
    void Sleep();
}
```

```csharp
public interface IMeetingParticipant
{
    void AttendMeeting();
}
```

Human:

```csharp
public class Employee :
    IWorker,
    IHuman,
    IMeetingParticipant
{
}
```

Robot:

```csharp
public class Robot :
    IWorker,
    IMeetingParticipant
{
}
```

---

# ISP in enterprise architecture

Consider:

```csharp
public interface IUserService
{
    User GetUser();
    void CreateUser();
    void UpdateUser();
    void DeleteUser();
    void ExportUsers();
    void SendUserEmail();
}
```

This becomes a **God Interface**.

Instead:

```text
IUserReader
IUserWriter
IUserExporter
IUserNotificationService
```

For example:

```csharp
public interface IUserReader
{
    Task<User> GetUser(int id);
}
```

```csharp
public interface IUserWriter
{
    Task Create(User user);
    Task Update(User user);
}
```

Consumers depend only on what they need.

---

# ISP + CQRS

This connects directly with the architecture you've been studying.

Instead of:

```text
IUserService
   ├── GetUser()
   ├── CreateUser()
   ├── UpdateUser()
   └── DeleteUser()
```

CQRS naturally separates:

```text
Queries
   ↓
IQueryHandler<TQuery,TResponse>

Commands
   ↓
ICommandHandler<TCommand,TResponse>
```

Example:

```csharp
public interface IQueryHandler<TQuery, TResult>
{
    Task<TResult> Handle(TQuery query);
}
```

```csharp
public interface ICommandHandler<TCommand>
{
    Task Handle(TCommand command);
}
```

This is ISP applied at architectural scale.

---

# 5. D — Dependency Inversion Principle (DIP)

This is probably the **most architecturally important SOLID principle**.

> High-level modules should not depend directly on low-level implementation details. Both should depend on abstractions.

And:

> Abstractions should not depend on details. Details should depend on abstractions.

---

# ❌ Traditional design

```csharp
public class OrderService
{
    private readonly SqlOrderRepository _repository;

    public OrderService()
    {
        _repository = new SqlOrderRepository();
    }
}
```

Dependency:

```text
OrderService
     ↓
SqlOrderRepository
     ↓
SQL Server
```

The business logic knows infrastructure details.

---

# ✅ Dependency inversion

```csharp
public interface IOrderRepository
{
    Task Save(Order order);
}
```

Application service:

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;

    public OrderService(IOrderRepository repository)
    {
        _repository = repository;
    }
}
```

Infrastructure:

```csharp
public class SqlOrderRepository : IOrderRepository
{
    public async Task Save(Order order)
    {
        // SQL implementation
    }
}
```

Now:

```text
             IOrderRepository
              ↑          ↑
              │          │
       OrderService   SqlOrderRepository
       (High level)    (Low level)
```

Both depend on:

```text
IOrderRepository
```

---

# DIP is NOT the same as Dependency Injection

This is a very important interview distinction.

### Dependency Inversion Principle

A **design principle**:

```text
High-level policy
       ↓
Abstraction
       ↑
Low-level detail
```

### Dependency Injection

A **technique** for providing dependencies:

```csharp
services.AddScoped<IOrderRepository, SqlOrderRepository>();
```

DI helps implement DIP.

But:

> **DI ≠ DIP**

You can use dependency injection and still have bad architecture.

---

# SOLID Together

Now combine all five.

Imagine an e-commerce system:

```text
                    API
                     │
                     ↓
              Order Controller
                     │
                     ↓
                OrderService
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
     IRepository  ITaxService  INotifier
          │          │          │
          ↓          ↓          ↓
       SQL DB    Tax Engine    Email/SMS
```

### SRP

Each component has focused responsibility.

### OCP

New tax/payment/notification implementations can be added.

### LSP

Implementations honor their contracts.

### ISP

Interfaces are focused.

### DIP

Business logic depends on abstractions rather than SQL/email/etc.

---

# SOLID + Design Patterns

A very useful interview mapping:

| SOLID | Common patterns                  |
| ----- | -------------------------------- |
| SRP   | Facade, Command                  |
| OCP   | Strategy, Factory, Decorator     |
| LSP   | Polymorphism, Template Method    |
| ISP   | Role interfaces, CQRS            |
| DIP   | Dependency Injection, Repository |

But remember:

> Design patterns are **tools**. SOLID describes desirable **design properties**.

---

# SOLID in a Real .NET Architecture

A typical Clean Architecture / Hexagonal-style application might look like:

```text
┌─────────────────────────────────────────────┐
│                Presentation                │
│                                             │
│ API / Controller / Function                │
└─────────────────────┬───────────────────────┘
                      ↓
┌─────────────────────────────────────────────┐
│                 Application                │
│                                             │
│ Commands / Queries / Handlers               │
│ Use Cases                                   │
│ Interfaces                                  │
└─────────────────────┬───────────────────────┘
                      ↓
┌─────────────────────────────────────────────┐
│                   Domain                   │
│                                             │
│ Entities / Value Objects / Business Rules  │
└─────────────────────────────────────────────┘
                      ↑
┌─────────────────────────────────────────────┐
│                Infrastructure              │
│                                             │
│ SQL / Service Bus / Redis / APIs / Email   │
└─────────────────────────────────────────────┘
```

SOLID helps enforce these boundaries.

---

# Example: CQRS + SOLID

Suppose:

```http
POST /orders
```

Controller:

```csharp
[HttpPost]
public async Task<IActionResult> Create(CreateOrderCommand command)
{
    await _handler.Handle(command);

    return Ok();
}
```

Command:

```csharp
public record CreateOrderCommand(
    int CustomerId,
    List<OrderItem> Items);
```

Handler:

```csharp
public class CreateOrderHandler
{
    private readonly IOrderRepository _repository;
    private readonly IPaymentService _payment;
    private readonly IEventPublisher _publisher;

    public async Task Handle(CreateOrderCommand command)
    {
        var order = Order.Create(
            command.CustomerId,
            command.Items);

        await _repository.Save(order);

        await _payment.Process(order);

        await _publisher.Publish(
            new OrderCreatedEvent(order.Id));
    }
}
```

This demonstrates:

```text
SRP
Handler → Create Order use case

DIP
Handler → interfaces

ISP
Small focused interfaces

OCP
New payment implementations can be introduced

LSP
Payment implementations must honor IPaymentService contract
```

---

# SOLID + Event-Driven Architecture

Suppose:

```text
OrderCreated
      ↓
Azure Service Bus Topic
      ↓
 ┌────┼─────────┐
 ↓    ↓         ↓
Email Inventory Analytics
```

Each consumer can have its own responsibility.

```csharp
public interface IEventHandler<T>
{
    Task Handle(T @event);
}
```

Then:

```csharp
public class OrderEmailHandler
    : IEventHandler<OrderCreatedEvent>
{
    public Task Handle(OrderCreatedEvent @event)
    {
        // Send email
    }
}
```

```csharp
public class InventoryHandler
    : IEventHandler<OrderCreatedEvent>
{
    public Task Handle(OrderCreatedEvent @event)
    {
        // Reserve inventory
    }
}
```

The event itself doesn't need to know which handler executes.

The messaging infrastructure maps:

```text
Message type
     ↓
Subscription
     ↓
Consumer
     ↓
Handler
```

This aligns nicely with SRP, OCP, ISP and DIP.

---

# How to Identify SOLID Violations in Code Review

As a senior engineer, don't just ask:

> "Does this code follow SOLID?"

Use these questions.

### SRP

> "How many different reasons could cause this class to change?"

### OCP

> "When I add a new variation, do I keep modifying this class?"

### LSP

> "Can every implementation safely replace the abstraction?"

### ISP

> "Does this consumer really need everything exposed by this interface?"

### DIP

> "Does business logic know about infrastructure details?"

---

# The Most Important Architecture Connection

Think of SOLID as a **dependency-management strategy**.

Bad architecture:

```text
                  Infrastructure
                 ↑      ↑      ↑
                 │      │      │
              Business Logic
                 ↑
                API
```

Business logic becomes coupled to:

```text
SQL
Azure
HTTP
Redis
Email
Service Bus
```

Better:

```text
             Business Rules
                   │
                   ↓
             Abstractions
              ↑    ↑    ↑
              │    │    │
             SQL  Azure  HTTP
```

The dependency direction points toward **stable business abstractions**.

---

# SOLID ≠ "Create Interface for Every Class"

This is an excellent senior/architect interview point.

Bad:

```text
CustomerService
      ↓
ICustomerService
      ↓
CustomerServiceImpl
```

when there is no meaningful reason for substitution.

You might be creating abstraction merely because:

> "SOLID says use interfaces."

It doesn't.

Instead:

```text
Do I have a boundary?
Do I have variation?
Do I need substitution?
Do I need independent testing?
Is this infrastructure?
Is this a business policy?
```

If yes → abstraction may be valuable.

---

# SOLID vs Clean Architecture

These are related but different.

```text
SOLID
  ↓
Object/module design principles
```

```text
Clean Architecture
  ↓
System-level dependency organization
```

```text
Design Patterns
  ↓
Reusable design solutions
```

```text
DDD
  ↓
Modeling business/domain complexity
```

```text
CQRS
  ↓
Separating command and query responsibilities
```

Together:

```text
                 Architecture
                      │
       ┌──────────────┼──────────────┐
       ↓              ↓              ↓
 Clean Architecture   DDD           CQRS
       │              │              │
       └──────────────┼──────────────┘
                      ↓
                    SOLID
                      ↓
             Design Principles
                      ↓
              Design Patterns
```

---

# Staff Engineer Mental Model

At Staff/Architect level, don't memorize:

```text
S = Single Responsibility
O = Open Closed
...
```

Instead remember:

```text
              CHANGE
                │
       ┌────────┴────────┐
       ↓                 ↓
   Who changes?      What changes?
       │                 │
       ↓                 ↓
      SRP               OCP
       
       Can replacement    What does
       safely work?       consumer need?
            │                │
            ↓                ↓
           LSP              ISP
                    \
                     \
                      ↓
                 DIP
                      ↓
             Who depends on whom?
```

The deeper principle is:

> **Design your system so that change is isolated and dependencies point toward stable abstractions.**

---

# Interview-Ready Answer

If an interviewer asks:

**"Explain SOLID principles with real-world experience."**

You can answer:

> "I don't treat SOLID as five isolated rules. I use it primarily to control coupling and isolate change. SRP helps me separate responsibilities and reasons for change. OCP helps me introduce new behavior through extension points rather than repeatedly modifying stable code. LSP ensures implementations honor the contracts defined by their abstractions. ISP keeps interfaces focused on the needs of their consumers. DIP ensures business logic depends on abstractions rather than infrastructure details. In a typical .NET application using Clean Architecture and CQRS, for example, command handlers depend on repository, payment and messaging abstractions, while their concrete implementations live in infrastructure. This makes the system easier to test, extend and evolve without unnecessarily coupling business logic to frameworks or infrastructure."

### One-line memory trick

> **SRP = isolate responsibility**
> **OCP = isolate change**
> **LSP = protect contracts**
> **ISP = isolate consumers**
> **DIP = isolate dependencies**

That's the **Senior → Architect → Staff Engineer** way to think about SOLID: not "how many interfaces do I have?", but **"where will change happen, and have I designed the dependency boundaries so that change stays localized?"**


## More understandable knowledge base.

Absolutely. For **LSP**, a more practical enterprise example is better than the classic Bird/Penguin example. Below is the **final Senior Developer / Architect / Staff Engineer version** with a stronger LSP example.

# SOLID Principles — Senior / Architect / Staff Engineer Notes

SOLID is not simply five coding rules. At senior/architect level, use SOLID to manage:

* **Change**
* **Coupling**
* **Dependencies**
* **Extensibility**
* **Contracts**
* **Testability**
* **Maintainability**

### Easy memory

> **S** → One reason to change
> **O** → Extend without repeatedly modifying stable code
> **L** → Substitution must preserve the contract
> **I** → Small, consumer-focused interfaces
> **D** → Business logic depends on abstractions

---

# 1. S — Single Responsibility Principle

> A class/module should have **one reason to change**.

It does **not** mean a class can have only one method.

The better question is:

> **"Who or what could cause this class to change?"**

---

## ❌ Bad Example

```csharp
public class OrderService
{
    public void CreateOrder(Order order)
    {
        // Create order
    }

    public decimal CalculateTax(Order order)
    {
        // Tax calculation
        return 0;
    }

    public void SaveToDatabase(Order order)
    {
        // SQL logic
    }

    public void SendEmail(Order order)
    {
        // Email logic
    }

    public byte[] GenerateInvoice(Order order)
    {
        // PDF generation
        return [];
    }
}
```

This class has multiple responsibilities:

```text
OrderService
│
├── Order business logic
├── Tax calculation
├── Database
├── Email
└── Invoice generation
```

Therefore, it has multiple reasons to change.

---

## ✅ Better

```csharp
public class OrderService
{
    private readonly ITaxCalculator _taxCalculator;
    private readonly IOrderRepository _repository;
    private readonly INotificationService _notification;
    private readonly IInvoiceGenerator _invoiceGenerator;

    public async Task CreateOrder(Order order)
    {
        order.Tax = _taxCalculator.Calculate(order);

        await _repository.Save(order);

        await _invoiceGenerator.Generate(order);

        await _notification.SendOrderCreated(order);
    }
}
```

Now:

```text
OrderService
    │
    ├── ITaxCalculator
    ├── IOrderRepository
    ├── IInvoiceGenerator
    └── INotificationService
```

Each component has a focused responsibility.

### Senior-level thinking

Ask:

> **"If one requirement changes, how many unrelated classes need modification?"**

If tax changes, ideally:

```text
Tax rules
   ↓
TaxCalculator
```

not:

```text
OrderService
Database
Email
Invoice
API
```

### Important

**SRP = one reason to change**, not "one method per class."

---

# 2. O — Open/Closed Principle

> Software should be **open for extension but closed for modification**.

When new variations are expected, prefer adding new implementations rather than repeatedly changing stable business logic.

---

## ❌ Bad

```csharp
public class PaymentService
{
    public void Pay(string type, decimal amount)
    {
        if (type == "CreditCard")
        {
            // Credit Card
        }
        else if (type == "UPI")
        {
            // UPI
        }
        else if (type == "PayPal")
        {
            // PayPal
        }
    }
}
```

Every new payment type requires modification.

```text
Credit Card
UPI
PayPal
      ↓
PaymentService keeps changing
```

---

## ✅ Better

```csharp
public interface IPaymentProcessor
{
    void Pay(decimal amount);
}
```

```csharp
public class CreditCardPayment : IPaymentProcessor
{
    public void Pay(decimal amount)
    {
        // Credit card
    }
}
```

```csharp
public class UpiPayment : IPaymentProcessor
{
    public void Pay(decimal amount)
    {
        // UPI
    }
}
```

```csharp
public class PayPalPayment : IPaymentProcessor
{
    public void Pay(decimal amount)
    {
        // PayPal
    }
}
```

Now adding:

```csharp
public class ApplePayPayment : IPaymentProcessor
{
    public void Pay(decimal amount)
    {
        // Apple Pay
    }
}
```

doesn't require changing the existing payment implementations.

### Architecture connection

OCP commonly appears with:

* Strategy Pattern
* Factory Pattern
* Decorator Pattern
* Plugin architecture
* Dependency Injection
* Polymorphism

### Senior-level warning

Don't create abstractions everywhere just because of OCP.

Ask:

> **"Is this area actually expected to vary?"**

SOLID is about **managing change**, not creating hundreds of interfaces.

---

# 3. L — Liskov Substitution Principle

> If `B` is an implementation/subtype of abstraction `A`, consumers should be able to use `B` wherever they expect `A` **without unexpected behavior**.

This is really about **contracts and substitutability**.

---

# Practical Enterprise Example — File Storage

Suppose your application defines:

```csharp
public interface IFileStorage
{
    Task Upload(string path, Stream file);
    Task Delete(string path);
    Task<Stream> Download(string path);
}
```

Your application expects all implementations to support these operations.

You have:

```text
IFileStorage
     │
     ├── AzureBlobStorage
     ├── S3Storage
     └── LocalFileStorage
```

All are supposed to be interchangeable.

---

## ❌ LSP Violation

Suppose someone implements:

```csharp
public class ReadOnlyStorage : IFileStorage
{
    public Task<Stream> Download(string path)
    {
        // Download
    }

    public Task Upload(string path, Stream file)
    {
        throw new NotSupportedException();
    }

    public Task Delete(string path)
    {
        throw new NotSupportedException();
    }
}
```

Technically:

```text
ReadOnlyStorage implements IFileStorage
```

But semantically:

```text
IFileStorage
     ↓
promises Upload + Delete + Download

ReadOnlyStorage
     ↓
only supports Download
```

Therefore, this is a poor abstraction.

A consumer doing:

```csharp
public async Task ProcessFile(
    IFileStorage storage,
    Stream file)
{
    await storage.Upload("invoice.pdf", file);
}
```

works with:

```csharp
AzureBlobStorage
```

but unexpectedly fails with:

```csharp
ReadOnlyStorage
```

The implementation cannot honor the abstraction's contract.

---

## ✅ Better Design

Separate capabilities:

```csharp
public interface IFileReader
{
    Task<Stream> Download(string path);
}
```

```csharp
public interface IFileWriter
{
    Task Upload(string path, Stream file);
}
```

```csharp
public interface IFileDeleter
{
    Task Delete(string path);
}
```

Full storage:

```csharp
public class AzureBlobStorage :
    IFileReader,
    IFileWriter,
    IFileDeleter
{
}
```

Read-only storage:

```csharp
public class ReadOnlyStorage :
    IFileReader
{
}
```

Now the contract accurately represents capabilities.

```text
IFileReader
     ↑
ReadOnlyStorage


IFileReader
IFileWriter
IFileDeleter
     ↑
AzureBlobStorage
```

---

# Another important LSP example — API contracts

Imagine an API contract:

```http
GET /orders/123
```

Contract says:

```json
{
  "id": 123,
  "status": "Completed"
}
```

Consumers depend on this contract.

If a replacement implementation suddenly returns:

```json
{
  "orderId": 123,
  "orderStatus": "Completed"
}
```

without preserving compatibility, consumers may break.

Therefore, LSP thinking applies beyond inheritance.

It applies to:

```text
Classes
Interfaces
APIs
Events
Message contracts
Plugins
Microservices
Database abstractions
```

### Staff-level interpretation

Ask:

> **"Can consumers safely replace one implementation with another without having to know which implementation they're using?"**

If the answer is no, investigate the abstraction.

### Important distinction

LSP is **not**:

> "Every child must have exactly the same implementation."

It is:

> **"Every implementation must honor the behavioral contract expected by consumers."**

---

# 4. I — Interface Segregation Principle

> Clients should not be forced to depend on methods they don't need.

---

## ❌ Bad

```csharp
public interface IUserService
{
    User GetUser();
    void CreateUser();
    void UpdateUser();
    void DeleteUser();
    void ExportUsers();
    void SendEmail();
}
```

Imagine an API that only needs:

```csharp
GetUser()
```

but it depends on the entire interface.

That's unnecessary coupling.

---

## ✅ Better

```csharp
public interface IUserReader
{
    Task<User> GetUser(int id);
}
```

```csharp
public interface IUserWriter
{
    Task Create(User user);
    Task Update(User user);
}
```

```csharp
public interface IUserExporter
{
    Task Export();
}
```

Consumers depend only on what they require.

---

# ISP + CQRS

This is where ISP naturally connects with CQRS.

Instead of:

```text
IUserService
│
├── Get
├── Create
├── Update
└── Delete
```

CQRS separates:

```text
Commands
   ↓
ICommandHandler<TCommand>

Queries
   ↓
IQueryHandler<TQuery,TResponse>
```

Example:

```csharp
public interface IQueryHandler<TQuery, TResult>
{
    Task<TResult> Handle(TQuery query);
}
```

```csharp
public interface ICommandHandler<TCommand>
{
    Task Handle(TCommand command);
}
```

Each handler focuses on a specific use case.

### Senior-level question

> **"Does this consumer really need everything this interface exposes?"**

If not → consider segregation.

---

# 5. D — Dependency Inversion Principle

> High-level business logic should not depend directly on low-level implementation details.

Both should depend on abstractions.

---

## ❌ Bad

```csharp
public class OrderService
{
    private readonly SqlOrderRepository _repository;

    public OrderService()
    {
        _repository = new SqlOrderRepository();
    }
}
```

Dependency:

```text
OrderService
     ↓
SqlOrderRepository
     ↓
SQL Server
```

Business logic directly knows infrastructure.

---

## ✅ Better

Define an abstraction:

```csharp
public interface IOrderRepository
{
    Task Save(Order order);
}
```

Business logic:

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;

    public OrderService(IOrderRepository repository)
    {
        _repository = repository;
    }

    public async Task Create(Order order)
    {
        await _repository.Save(order);
    }
}
```

Infrastructure:

```csharp
public class SqlOrderRepository : IOrderRepository
{
    public async Task Save(Order order)
    {
        // SQL implementation
    }
}
```

Dependency becomes:

```text
             IOrderRepository
              ↑           ↑
              │           │
       OrderService    SqlOrderRepository
       High-level       Low-level
```

---

# DIP vs Dependency Injection

Very important interview distinction.

### DIP

A **design principle**:

```text
High-level policy
       ↓
   Abstraction
       ↑
Low-level detail
```

### Dependency Injection

A **technique** for supplying dependencies.

```csharp
services.AddScoped<IOrderRepository, SqlOrderRepository>();
```

Therefore:

> **Dependency Injection is a technique that can help implement Dependency Inversion.**

DI and DIP are not the same thing.

---

# SOLID in a Real Architecture

Imagine an enterprise .NET system:

```text
                    API
                     │
                     ↓
             Controller / Function
                     │
                     ↓
              Application Layer
                     │
              Commands / Queries
                     │
                     ↓
                  Handler
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
    Repository    Payment    Publisher
    Interface     Interface   Interface
          │          │          │
          ↓          ↓          ↓
       SQL DB     Payment     Service Bus
                    API
```

SOLID is operating at multiple levels.

### SRP

```text
Handler
→ One use case
```

### OCP

```text
New payment provider
→ New implementation
```

### LSP

```text
Payment implementations
→ Must honor payment contract
```

### ISP

```text
Focused interfaces
→ Consumers depend only on required capabilities
```

### DIP

```text
Application
→ Interfaces

Infrastructure
→ Implements interfaces
```

---

# SOLID + CQRS + Event Driven

A practical example:

```text
POST /orders
     │
     ↓
CreateOrderCommand
     │
     ↓
CreateOrderHandler
     │
     ├── IOrderRepository
     ├── IPaymentService
     └── IEventPublisher
                 │
                 ↓
          OrderCreatedEvent
                 │
          Azure Service Bus
                 │
       ┌─────────┼─────────┐
       ↓         ↓         ↓
     Email   Inventory  Analytics
    Handler    Handler     Handler
```

This gives you natural separation.

```text
SRP
Each handler has focused responsibility

OCP
New event consumers can be added

LSP
Implementations must honor contracts

ISP
Small focused interfaces

DIP
Handlers depend on abstractions
```

---

# How a Staff Engineer Reviews SOLID

Instead of asking:

> "Is this class SOLID?"

ask these five questions:

### 1. SRP

> **How many independent reasons can make this code change?**

### 2. OCP

> **When a new variation arrives, do I modify stable code or add an implementation?**

### 3. LSP

> **Can I replace this implementation without changing consumer behavior?**

### 4. ISP

> **Are consumers depending on capabilities they don't actually need?**

### 5. DIP

> **Does business logic depend on infrastructure details?**

---

# SOLID and Design Patterns

| Principle | Common supporting patterns                         |
| --------- | -------------------------------------------------- |
| **SRP**   | Command, Facade                                    |
| **OCP**   | Strategy, Factory, Decorator                       |
| **LSP**   | Polymorphism, Contract-based design                |
| **ISP**   | Role interfaces, CQRS                              |
| **DIP**   | Dependency Injection, Repository, Ports & Adapters |

Patterns are **tools**.

SOLID provides **design principles**.

---

# SOLID vs Architecture

Don't confuse these concepts.

```text
SOLID
 ↓
Object / Module Design Principles
```

```text
Design Patterns
 ↓
Reusable Design Solutions
```

```text
CQRS
 ↓
Command / Query Separation
```

```text
DDD
 ↓
Business Domain Modeling
```

```text
Clean Architecture
 ↓
Dependency & Layer Organization
```

They complement each other.

---

# The Biggest Senior-Level Insight

SOLID is fundamentally about **managing change and dependencies**.

Think:

```text
                    CHANGE
                       │
          ┌────────────┴────────────┐
          ↓                         ↓
    Who should change?       How should it change?
          │                         │
          ↓                         ↓
         SRP                       OCP
                                   
          Can replacement           What does
          safely work?              consumer need?
                │                       │
                ↓                       ↓
               LSP                     ISP
                   \                   /
                    \                 /
                     ↓               ↓
                       DIP
                        │
                        ↓
                Dependency Direction
```

### Final mental model

> **SRP → Isolate responsibility**
> **OCP → Isolate change**
> **LSP → Protect contracts**
> **ISP → Isolate consumers**
> **DIP → Isolate dependencies**

And the Staff/Architect mindset is:

> **Don't use SOLID to create more abstractions. Use SOLID to create better boundaries.**

The real question isn't **"How do I apply SOLID?"**

It's:

> **"Where is my system likely to change, who owns that change, and how can I prevent that change from unnecessarily propagating through the system?"**

That is the level at which SOLID becomes an **architecture tool**, rather than just an interview definition.

