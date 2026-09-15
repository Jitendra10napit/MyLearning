# Creational Design Patterns

**Creational Design Patterns** are design patterns that focus on **how objects are created**.

Instead of allowing every part of your application to directly create objects with `new`, creational patterns provide a **controlled, flexible, and maintainable way to create objects**.

Think of them as:

> **“Who should create the object, how should it be created, and what should happen if the way we create it changes?”**

---

## 1. Why do Creational Patterns matter?

Consider this code:

```csharp
public class OrderService
{
    public void Process(OrderRequest request)
    {
        var payment = new StripePayment();

        payment.Pay(request.Amount);
    }
}
```

It looks simple.

But now imagine the business says:

> "We also support PayPal, Razorpay and UPI."

You might write:

```csharp
public void Process(OrderRequest request)
{
    IPayment payment;

    if (request.PaymentType == "Stripe")
    {
        payment = new StripePayment();
    }
    else if (request.PaymentType == "PayPal")
    {
        payment = new PayPalPayment();
    }
    else if (request.PaymentType == "Razorpay")
    {
        payment = new RazorpayPayment();
    }

    payment.Pay(request.Amount);
}
```

Now your `OrderService` knows:

* which implementation exists
* how it is created
* how many implementations exist
* which implementation to choose

That's **tight coupling**.

### The architectural problem

```text
OrderService
     │
     ├── knows Stripe
     ├── knows PayPal
     ├── knows Razorpay
     └── knows UPI
```

If payment creation changes, `OrderService` must change.

This violates an important design principle:

> **Separate object creation from object usage.**

That's where creational patterns become useful.

---

# 2. The Core Idea

Instead of:

```text
Business Logic
     │
     ↓
    new
     │
     ↓
Concrete Object
```

we try to move creation behind an abstraction:

```text
Business Logic
     │
     ↓
Abstraction / Factory / Builder
     │
     ↓
Concrete Object
```

Now:

```text
Business logic
      ↓
"What payment do I need?"
      ↓
Creation mechanism
      ↓
StripePayment
```

The business logic doesn't need to know the construction details.

---

# 3. The Five GoF Creational Patterns

There are **5 classic creational design patterns**:

| Pattern              | Main Problem                               |
| -------------------- | ------------------------------------------ |
| **Singleton**        | Need controlled/shared instance            |
| **Factory Method**   | Creation depends on a decision             |
| **Abstract Factory** | Create families of related objects         |
| **Builder**          | Complex object construction                |
| **Prototype**        | Create objects by cloning existing objects |

The key is not memorizing the names.

Remember the **problem each solves**.

---

# 4. Factory Method — "Which object should I create?"

Suppose:

```text
Payment
 ├── Stripe
 ├── PayPal
 └── Razorpay
```

Instead of:

```csharp
new StripePayment();
new PayPalPayment();
new RazorpayPayment();
```

we use a factory:

```csharp
public interface IPayment
{
    void Pay(decimal amount);
}
```

```csharp
public class PaymentFactory
{
    public IPayment Create(string type)
    {
        return type switch
        {
            "Stripe" => new StripePayment(),
            "PayPal" => new PayPalPayment(),
            "Razorpay" => new RazorpayPayment(),
            _ => throw new ArgumentException("Unsupported payment")
        };
    }
}
```

Now:

```csharp
var payment = factory.Create(request.PaymentType);

payment.Pay(request.Amount);
```

The caller doesn't need to know how the object is constructed.

### Mental model

> **Factory = Decide which object to create.**

---

# 5. Builder — "How do I construct this complicated object?"

Imagine:

```csharp
var report = new Report(
    title,
    author,
    company,
    startDate,
    endDate,
    includeCharts,
    includeSummary,
    includeRecommendations,
    ...
);
```

This becomes difficult to maintain.

Builder gives you:

```csharp
var report = new ReportBuilder()
    .WithTitle("Sales Report")
    .WithAuthor("Jitendra")
    .ForDateRange(start, end)
    .IncludeCharts()
    .IncludeSummary()
    .Build();
```

The important difference is:

```text
Factory
→ WHICH object?

Builder
→ HOW to construct the object?
```

---

# 6. Singleton — "Should there be only one instance?"

Example:

```text
Application
     │
     ├── Service A
     ├── Service B
     └── Service C
             │
             ↓
       Same instance
```

In .NET, you'd normally let Dependency Injection manage this:

```csharp
services.AddSingleton<IConfigurationService,
                      ConfigurationService>();
```

Every consumer gets the same registered instance.

### Mental model

> **Singleton = Control instance lifetime so one shared instance is used.**

Important architect point:

**Singleton is not "make everything global."**

Global mutable state can create:

* concurrency issues
* testing problems
* hidden dependencies
* difficult debugging

---

# 7. Abstract Factory — "Create a family of related objects"

Imagine your application supports:

```text
Cloud Provider
       │
       ├── AWS
       │    ├── Storage
       │    └── Queue
       │
       └── Azure
            ├── Storage
            └── Queue
```

You don't want:

```text
AWSStorage + AzureStorage + AWSQueue + AzureQueue
```

scattered throughout the application.

Instead:

```csharp
public interface ICloudFactory
{
    IStorage CreateStorage();
    IQueue CreateQueue();
}
```

Then:

```text
AWSFactory
   ├── AWSStorage
   └── AWSQueue

AzureFactory
   ├── AzureStorage
   └── AzureQueue
```

### Mental model

> **Abstract Factory = Create a compatible family of objects.**

---

# 8. Prototype — "Can I clone an existing object?"

Suppose object creation is expensive.

You already have:

```text
Complex Object
     │
     ├── configuration
     ├── rules
     ├── metadata
     └── nested objects
```

Instead of reconstructing everything:

```text
Existing Object
      │
     Clone
      ↓
New Object
```

For example:

```csharp
var copy = original.Clone();
```

### Mental model

> **Prototype = Create a new object by copying an existing one.**

---

# 9. The Most Important Architectural Principle

Creational patterns are ultimately about **separating creation from usage**.

Without separation:

```text
Business Logic
      │
      ├── new Stripe()
      ├── new PayPal()
      ├── new Razorpay()
      └── new ...
```

With separation:

```text
                Creation
                   │
                   ↓
Business Logic → Abstraction
                   │
                   ↓
              Implementation
```

This gives you **loose coupling**.

---

# 10. Why does this matter at Architect level?

Imagine your company starts with:

```text
Stripe
```

Then:

```text
Stripe
PayPal
Razorpay
```

Then:

```text
Stripe
PayPal
Razorpay
Adyen
Checkout.com
```

If creation is scattered everywhere:

```text
Controller
Service
Handler
Background Worker
Consumer
```

you'll have hundreds of places knowing concrete implementations.

That's architectural coupling.

With a good creation strategy:

```text
                    Payment Abstraction
                           ↑
                           │
                     Factory / DI
                           │
          ┌────────────────┼───────────────┐
          ↓                ↓               ↓
       Stripe           PayPal          Razorpay
```

Adding a new provider becomes much easier.

---

# 11. Creational Patterns + SOLID

This is where you should connect the concepts.

### Dependency Inversion Principle

Instead of:

```csharp
OrderService → StripePayment
```

prefer:

```csharp
OrderService → IPayment
                       ↑
                 StripePayment
```

The service depends on an **abstraction**, not a concrete implementation.

Creational patterns often help you achieve this.

---

# 12. Creational Patterns in Modern .NET

One important thing:

**You don't always explicitly implement these patterns yourself.**

Modern .NET already provides mechanisms that solve many creation problems.

For example:

```csharp
services.AddTransient<IOrderService, OrderService>();
services.AddScoped<IUnitOfWork, UnitOfWork>();
services.AddSingleton<ICache, MemoryCache>();
```

The **Dependency Injection container** becomes responsible for object construction.

Conceptually:

```text
Your code
   ↓
Request abstraction
   ↓
DI Container
   ↓
Find registration
   ↓
Construct object
   ↓
Inject dependencies
```

So as a Senior Developer/Architect, don't think:

> "I must implement Factory pattern manually."

Think:

> **"Who owns object creation in this architecture?"**

Sometimes the answer is:

* Factory
* DI container
* Builder
* Framework
* Configuration
* Application composition root

---

# 13. When Should You Use Creational Patterns?

Use them when object creation itself has **complexity, variation, lifecycle, or coupling**.

### Good reasons

```text
Multiple implementations
        ↓
Factory
```

```text
Complex construction
        ↓
Builder
```

```text
Related object families
        ↓
Abstract Factory
```

```text
Controlled shared lifecycle
        ↓
Singleton / DI lifetime
```

```text
Expensive object creation
        ↓
Prototype
```

---

# 14. When NOT to use them

This is equally important for an Architect.

Don't do this:

```csharp
var customer = new Customer();
```

and think:

> "I need a CustomerFactory because factories are good architecture."

That's unnecessary.

If construction is trivial:

```csharp
var customer = new Customer(name, email);
```

is perfectly good.

### Pattern introduces abstraction.

Abstraction has a cost:

```text
More interfaces
More classes
More indirection
More complexity
More things to understand
```

Therefore:

> **Use a creational pattern when the complexity of creation justifies the abstraction.**

---

# 15. Interview-Level Answer

If an interviewer asks:

### "What are Creational Design Patterns?"

A strong Senior-level answer would be:

> **Creational design patterns deal with object creation. Their primary goal is to encapsulate or control object construction so that business logic is not tightly coupled to concrete implementations. The classic GoF creational patterns are Singleton, Factory Method, Abstract Factory, Builder, and Prototype. In modern .NET, Dependency Injection also plays a major role in managing object creation and lifetimes. I choose these patterns when object construction has variability, complexity, lifecycle concerns, or when I need to reduce coupling—not simply because a pattern exists.**

That's a much stronger answer than:

> "Creational patterns are patterns used to create objects."

---

# 16. The One-Line Memory Trick

Remember:

```text
CREATIONAL
     │
     ├── Singleton
     │      → ONE
     │
     ├── Factory
     │      → WHICH
     │
     ├── Abstract Factory
     │      → FAMILY
     │
     ├── Builder
     │      → HOW
     │
     └── Prototype
            → COPY
```

So:

> **Singleton = ONE**
> **Factory = WHICH**
> **Abstract Factory = FAMILY**
> **Builder = HOW**
> **Prototype = COPY**

And the architect-level question behind all five is:

> **“Can I keep object creation independent from the business logic that uses the object?”**

That is the real reason **Creational Design Patterns matter**.
