# Singleton Design Pattern — Deep Understanding

Let's understand Singleton from a **Senior Developer / Architect perspective**, not just as "a class that has one object."

## 1. What problem does Singleton solve?

The Singleton pattern ensures that:

1. **Only one instance** of a class exists within the intended scope.
2. There is a **single access point** to that instance.
3. The instance can be **shared** by multiple consumers.

Think about an application configuration:

```text
                  Application
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
      Service A    Service B    Service C
          │            │            │
          └────────────┼────────────┘
                       ↓
             Configuration Instance
                  (ONE INSTANCE)
```

Without Singleton:

```text
Service A → new Configuration()
Service B → new Configuration()
Service C → new Configuration()

       ↓
3 different objects
```

With Singleton:

```text
Service A ─┐
Service B ─┼──→ Same Configuration Instance
Service C ─┘
```

---

# 2. Simple Example

Let's create a `Logger`.

We don't want every class to create its own logger:

```csharp
var logger1 = new Logger();
var logger2 = new Logger();
var logger3 = new Logger();
```

Instead, we want:

```text
logger1 ─┐
logger2 ─┼──→ ONE Logger
logger3 ─┘
```

---

# 3. Basic Singleton Implementation

```csharp
public sealed class Logger
{
    private static Logger? _instance;

    private Logger()
    {
    }

    public static Logger Instance
    {
        get
        {
            if (_instance == null)
            {
                _instance = new Logger();
            }

            return _instance;
        }
    }

    public void Log(string message)
    {
        Console.WriteLine(message);
    }
}
```

Usage:

```csharp
var logger1 = Logger.Instance;
var logger2 = Logger.Instance;

logger1.Log("Hello");

Console.WriteLine(
    ReferenceEquals(logger1, logger2)
);
```

Output:

```text
Hello
True
```

Both variables point to the **same object**.

---

# 4. Understand the Important Parts

Let's break this down.

### `private constructor`

```csharp
private Logger()
{
}
```

This prevents outside code from doing:

```csharp
new Logger(); // ❌ Not allowed
```

Only the class itself can create the object.

---

### Static instance

```csharp
private static Logger? _instance;
```

This stores the single instance.

---

### Static property

```csharp
public static Logger Instance
```

This becomes the access point:

```csharp
Logger.Instance
```

---

### Create only when needed

```csharp
if (_instance == null)
{
    _instance = new Logger();
}
```

The first call:

```text
Logger.Instance
      ↓
_instance == null
      ↓
new Logger()
      ↓
store instance
```

Second call:

```text
Logger.Instance
      ↓
_instance != null
      ↓
return existing instance
```

---

# 5. The Problem With This Implementation

The above implementation is **not thread-safe**.

Imagine two threads:

```text
Thread 1                    Thread 2

_instance == null           _instance == null
       ↓                           ↓
   new Logger()                new Logger()
       ↓                           ↓
   Instance A                  Instance B
```

Now you have **two instances**.

That's unacceptable if you're using Singleton to guarantee uniqueness.

---

# 6. Thread-Safe Singleton

One traditional implementation uses `lock`:

```csharp
public sealed class Logger
{
    private static Logger? _instance;
    private static readonly object _lock = new();

    private Logger()
    {
    }

    public static Logger Instance
    {
        get
        {
            lock (_lock)
            {
                if (_instance == null)
                {
                    _instance = new Logger();
                }

                return _instance;
            }
        }
    }

    public void Log(string message)
    {
        Console.WriteLine(message);
    }
}
```

Now:

```text
Thread 1 ──┐
           │
           ↓
         lock
           │
           ↓
       Create ONE
           │
           ↓
       Instance
           
Thread 2
   ↓
 waits for lock
   ↓
gets same instance
```

This works, but there's still a better C# approach.

---

# 7. Recommended C# Implementation — `Lazy<T>`

For modern C#:

```csharp
public sealed class Logger
{
    private static readonly Lazy<Logger> _instance =
        new Lazy<Logger>(() => new Logger());

    private Logger()
    {
    }

    public static Logger Instance => _instance.Value;

    public void Log(string message)
    {
        Console.WriteLine(message);
    }
}
```

Usage:

```csharp
var logger1 = Logger.Instance;
var logger2 = Logger.Instance;

logger1.Log("Application started");

Console.WriteLine(
    ReferenceEquals(logger1, logger2)
);
```

Output:

```text
Application started
True
```

### Why `Lazy<T>`?

It provides:

* lazy initialization
* thread safety
* clean implementation

The object isn't created until:

```csharp
Logger.Instance
```

is accessed.

---

# 8. Singleton With Dependency Injection — Most Important for .NET

In a real ASP.NET Core application, you usually **shouldn't manually implement Singleton**.

Instead:

```csharp
builder.Services.AddSingleton<ILoggerService, LoggerService>();
```

Example:

```csharp
public interface ILoggerService
{
    void Log(string message);
}
```

Implementation:

```csharp
public class LoggerService : ILoggerService
{
    public void Log(string message)
    {
        Console.WriteLine(message);
    }
}
```

Register:

```csharp
builder.Services.AddSingleton<ILoggerService, LoggerService>();
```

Then:

```csharp
public class OrderService
{
    private readonly ILoggerService _logger;

    public OrderService(ILoggerService logger)
    {
        _logger = logger;
    }

    public void CreateOrder()
    {
        _logger.Log("Order created");
    }
}
```

Another service:

```csharp
public class PaymentService
{
    private readonly ILoggerService _logger;

    public PaymentService(ILoggerService logger)
    {
        _logger = logger;
    }
}
```

Both services receive the **same `LoggerService` instance within the DI container's scope/lifetime**.

```text
ASP.NET Core DI Container
          │
          │ AddSingleton
          ↓
    LoggerService
       Instance #1
          ↑
     ┌────┴─────┐
     │          │
OrderService PaymentService
```

This is generally preferable because the **DI container controls the lifetime**, rather than your classes using global static access.

---

# 9. `Singleton` vs `Scoped` vs `Transient`

This is extremely important in .NET architecture.

```csharp
services.AddSingleton<IService, Service>();
services.AddScoped<IService, Service>();
services.AddTransient<IService, Service>();
```

### Singleton

One instance managed by the DI container:

```text
Application
    │
    └── Instance A
          ↑
    ┌─────┼─────┐
    A     B     C
```

### Scoped

One instance per scope.

In a typical ASP.NET Core application:

```text
HTTP Request 1 → Instance A
HTTP Request 2 → Instance B
HTTP Request 3 → Instance C
```

### Transient

New instance each time requested:

```text
Request
 ├── Resolve → A
 ├── Resolve → B
 └── Resolve → C
```

Memory trick:

```text
Singleton → ONE for container lifetime
Scoped    → ONE per scope
Transient → NEW each time
```

---

# 10. When Should You Use Singleton?

Good candidates can include **stateless or safely shared services**, such as:

```text
Configuration providers
Immutable application metadata
Thread-safe caches
Stateless reusable services
Expensive reusable resources
```

But always check whether the underlying object is **thread-safe**.

---

# 11. When Should You NOT Use Singleton?

This is where Senior/Architect thinking becomes important.

Don't make something Singleton just because:

> "I only want one object."

Be careful with objects containing mutable state:

```csharp
public class ShoppingCart
{
    private List<Product> _items;
}
```

Making this Singleton would mean:

```text
User A ─┐
        ├── Same ShoppingCart ❌
User B ─┘
```

That's obviously wrong.

You can accidentally introduce:

* shared mutable state
* race conditions
* memory retention
* difficult testing
* hidden dependencies
* concurrency problems

---

# 12. Singleton vs Static Class

These are often confused.

### Static class

```csharp
public static class Logger
{
    public static void Log(string message)
    {
    }
}
```

You can't create an instance:

```csharp
new Logger(); // ❌
```

### Singleton

```csharp
public sealed class Logger
{
    private static readonly Lazy<Logger> _instance = ...;

    private Logger()
    {
    }

    public static Logger Instance => _instance.Value;
}
```

You have an **actual object instance**.

Therefore Singleton can:

* implement interfaces
* be injected
* have instance state
* participate in polymorphism

For example:

```csharp
ILogger logger = Logger.Instance;
```

A static class cannot implement an interface in the same object-oriented way.

---

# 13. Architect-Level View

The most important thing to understand is:

> **Singleton is fundamentally a lifetime/instance-management decision.**

Don't start with:

> "Should I use Singleton?"

Start with:

> "Should this component have shared lifetime, and is sharing one instance safe?"

Then choose the appropriate mechanism.

In modern .NET:

```text
Need shared instance?
       ↓
Is it safe to share?
       ↓
Does DI manage it?
       ↓
services.AddSingleton(...)
```

rather than automatically writing:

```csharp
public static Something Instance
```

---

# 14. Real-World Example

Imagine a feature-flag service.

```csharp
public interface IFeatureFlagService
{
    bool IsEnabled(string feature);
}
```

Implementation:

```csharp
public class FeatureFlagService : IFeatureFlagService
{
    private readonly Dictionary<string, bool> _flags;

    public FeatureFlagService()
    {
        _flags = new Dictionary<string, bool>
        {
            ["NewCheckout"] = true,
            ["NewDashboard"] = false
        };
    }

    public bool IsEnabled(string feature)
    {
        return _flags.TryGetValue(feature, out var enabled)
            && enabled;
    }
}
```

Register:

```csharp
builder.Services.AddSingleton<
    IFeatureFlagService,
    FeatureFlagService>();
```

Now:

```text
Controller ──────┐
                 │
OrderService ────┼──→ FeatureFlagService
                 │        ONE INSTANCE
PaymentService ──┘
```

This makes sense if the feature flags are application-wide and the service is safe for concurrent use.

---

# 15. Singleton Interview Answer

If an interviewer asks:

**"What is Singleton Design Pattern?"**

A strong Senior-level answer:

> **Singleton is a creational design pattern that ensures a class has a single instance within a defined scope and provides a way to access that instance. It is useful when an object represents a shared resource or application-wide service and its state can be safely shared. In modern .NET applications, I generally prefer registering the service with the built-in Dependency Injection container using `AddSingleton()` rather than implementing a static Singleton manually, because DI makes the lifetime explicit, improves testability, and avoids hidden global dependencies.**

### Remember this:

```text
             SINGLETON
                 │
        ┌────────┴────────┐
        │                 │
   ONE INSTANCE       SHARED ACCESS
        │
        ↓
   Controlled Lifetime
```

And the key architect question:

> **"Do I really need one shared instance, or am I just trying to avoid creating objects?"**

That's the difference between **using Singleton deliberately** and **overusing Singleton as a global variable**.
