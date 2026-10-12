Absolutely. For your **13+ years of .NET experience and Staff/Lead Engineer interviews**, I would prepare .NET in terms of **runtime + ASP.NET Core + Web API + DI + middleware + configuration + performance + security + distributed systems**.

# .NET Interview FAQ — Staff/Lead Level

## 🔥 1. What is .NET?

.NET is a cross-platform development platform/runtime ecosystem used to build:

* Web APIs
* Web applications
* Background services
* Microservices
* Cloud applications
* Desktop applications
* Serverless applications

For your answer:

> ".NET provides the runtime, libraries, SDK, compiler and application frameworks such as ASP.NET Core. For backend systems, I primarily use ASP.NET Core Web API with dependency injection, middleware, EF Core and Azure services."

---

# 🔥 2. .NET Framework vs .NET Core vs modern .NET

|                   | .NET Framework | .NET Core      | Modern .NET     |
| ----------------- | -------------- | -------------- | --------------- |
| Platform          | Mainly Windows | Cross-platform | Cross-platform  |
| Open source       | Partial        | Yes            | Yes             |
| Performance       | Older          | High           | Higher          |
| Cloud-native      | Limited        | Strong         | Strong          |
| Current direction | Legacy         | Historical     | **.NET 8/9/10** |

Modern naming:

```text
.NET Core 3.1
     ↓
.NET 5
     ↓
.NET 6
     ↓
.NET 7
     ↓
.NET 8
     ↓
.NET 9
     ↓
.NET 10
```

Don't call modern .NET versions ".NET Core 8". Say **.NET 8**.

---

# 🔥 3. What happens when a .NET application starts?

A simplified flow:

```text
Program.cs
    ↓
Host Builder
    ↓
Configuration
    ↓
Dependency Injection
    ↓
Logging
    ↓
Middleware Pipeline
    ↓
Kestrel
    ↓
Request
    ↓
Endpoint/Controller
```

For ASP.NET Core:

```text
Client
  ↓
Kestrel
  ↓
Middleware
  ↓
Routing
  ↓
Authentication
  ↓
Authorization
  ↓
Controller / Minimal API
  ↓
Service
  ↓
Repository / EF Core
  ↓
Database
```

---

# 🔥 4. What is Kestrel?

Kestrel is ASP.NET Core's cross-platform web server.

```text
Client
   ↓
Reverse Proxy / Load Balancer
   ↓
Kestrel
   ↓
ASP.NET Core
```

It handles HTTP connections and passes requests into the ASP.NET Core pipeline.

It can run directly or behind infrastructure such as:

* Azure Application Gateway
* Azure Front Door
* NGINX
* IIS
* Kubernetes ingress

---

# 🔥 5. What is middleware?

Middleware is software in the HTTP request pipeline.

Middleware in ASP.NET Core is a software component in the HTTP request pipeline that can inspect or modify requests and responses, execute additional logic, and either pass control to the next component or terminate the pipeline. Examples include authentication, authorization, logging and exception-handling middleware.

Example:

```csharp
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
```

Conceptually:

```text
Request
  ↓
Exception Middleware
  ↓
Logging
  ↓
Authentication
  ↓
Authorization
  ↓
Routing/Endpoint
  ↓
Controller
  ↓
Response
```

Each middleware can:

* inspect request
* modify request
* call next middleware
* modify response
* terminate the pipeline

---

# 🔥 6. `Use`, `Run`, and `Map`

Use() — Executes logic and can pass the request to the next middleware.

Run() — Terminates the pipeline; it does not accept a next delegate.

Map() — Creates a separate branch of the pipeline based on the request path.


### Use

Can call the next middleware:

```csharp
app.Use(async (context, next) =>
{
    await next();
});
```

### Run

Terminal middleware:

```csharp
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello");
});
```

### Map

Branches the pipeline:

```csharp
app.Map("/health", healthApp =>
{
});
```

---

# 🔥 7. Why does middleware order matter?

This is a very common interview question.

For example:

```csharp
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
```

Authentication must establish the user identity before authorization evaluates permissions.

A simplified pipeline:

```text
Exception Handling
       ↓
HTTPS
       ↓
Routing
       ↓
Authentication
       ↓
Authorization
       ↓
Endpoint
```

Incorrect ordering can cause authentication/authorization behavior to fail or behave unexpectedly.

---

# 🔥 8. What is dependency injection?

Instead of creating dependencies inside a class:

```csharp
public class OrderService
{
    private readonly EmailService _email = new();
}
```

inject them:

```csharp
public class OrderService
{
    private readonly IEmailService _email;

    public OrderService(IEmailService email)
    {
        _email = email;
    }
}
```

Benefits:

* Loose coupling
* Testability
* Replaceable implementations
* Centralized dependency management

---

# 🔥 9. Explain DI lifetimes

### Transient

New instance each time it is requested.

```csharp
services.AddTransient<IEmailService, EmailService>();
```

### Scoped

One instance per request/scope.

```csharp
services.AddScoped<IOrderService, OrderService>();
```

### Singleton

One instance for the application lifetime.

```csharp
services.AddSingleton<ICacheService, CacheService>();
```

Remember:

```text
Transient → Every resolution
Scoped    → Per request/scope
Singleton → Application lifetime
```

---

# 🔥 10. What is a captive dependency?

A singleton depends on a scoped service.

```text
Singleton
    ↓
Scoped Service
```

This is problematic because the longer-lived object captures a shorter-lived dependency.

Classic example:

```csharp
services.AddSingleton<MyService>();
services.AddScoped<MyDbContext>();
```

if `MyService` directly depends on `MyDbContext`.

At Staff level, explain **lifetime mismatch**, not merely "this gives an error."


# What is a Captive Dependency in .NET?

A captive dependency occurs when a service with a longer lifetime holds a reference to a service with a shorter lifetime than itself.

In ASP.NET Core Dependency Injection, the most common example is a Singleton service depending on a Scoped service.

Let's understand this with a simple example.

## 1. First, understand service lifetimes

| Lifetime    | Meaning                                                       | Example                |
| ----------- | ------------------------------------------------------------- | ---------------------- |
| `Transient` | A new instance is created each time the service is requested. | A lightweight helper   |
| `Scoped`    | One instance per DI scope, usually one HTTP request.          | `DbContext`            |
| `Singleton` | One instance for the lifetime of the application.             | Application-wide cache |

## 2. Real-world example

Imagine a Singleton `ReportService` that depends on a Scoped `EmployeeService`.

ReportService — Singleton

One instance for the entire application

EmployeeService — Scoped

Expected to have one instance per request scope

Problem: the Singleton holds onto the Scoped instance.

The `ReportService` lives for the whole application, so its `EmployeeService` reference may also remain alive for the whole application instead of being created separately for each request.

This is called a captive dependency.

## 3. Code example: Incorrect implementation

```
builder.Services.AddScoped<EmployeeService>();
builder.Services.AddSingleton<ReportService>();
```

The services:

```
public class EmployeeService
{
    public Guid Id { get; } = Guid.NewGuid();

    public string GetEmployees()
    {
        return $"EmployeeService ID: {Id}";
    }
}

public class ReportService
{
    private readonly EmployeeService _employeeService;

    public ReportService(EmployeeService employeeService)
    {
        _employeeService = employeeService;
    }

    public string GenerateReport()
    {
        return _employeeService.GetEmployees();
    }
}
```

What's wrong?

- `ReportService` is created once.
- It receives an `EmployeeService` instance when constructed.
- That reference remains stored in the Singleton.
- Subsequent requests may reuse the same `EmployeeService` instance instead of getting the instance appropriate to each request scope.

Important: With scope validation enabled, ASP.NET Core will normally detect this invalid dependency relationship and throw an exception rather than allow it.

## 4. Why is a captive dependency dangerous?

Three common problems are:

1. Incorrect state sharing: Request-specific data may unintentionally be shared across requests.
2. Thread-safety issues: Multiple requests can access the same service concurrently.
3. Resource and data problems: If the Scoped service uses an EF Core `DbContext`, it can lead to concurrency issues and incorrect database operations.

This is especially important when using Entity Framework Core because `DbContext` is normally registered as Scoped.

## 5. How to fix it

### Option 1: Make both services Scoped

If `ReportService` does not need to be a Singleton, use the same lifetime.

```
builder.Services.AddScoped<EmployeeService>();
builder.Services.AddScoped<ReportService>();
```

Now both services can be created within each request scope.

### Option 2: Use `IServiceScopeFactory`

If `ReportService` genuinely needs to be a Singleton, create a new scope when you need the Scoped service.

```
public class ReportService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public ReportService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task<string> GenerateReportAsync()
    {
        using var scope = _scopeFactory.CreateScope();

        var employeeService =
            scope.ServiceProvider
                 .GetRequiredService<EmployeeService>();

        return employeeService.GetEmployees();
    }
}
```

Register the services:

```
builder.Services.AddScoped<EmployeeService>();
builder.Services.AddSingleton<ReportService>();
```

Each call creates a new scope and resolves an `EmployeeService` within it. The scope is disposed when the method finishes.

For asynchronous cleanup, use `CreateAsyncScope()` and `await using` when appropriate.

## 6. Interview question: Can a Scoped service depend on a Singleton?

Yes. This is normally valid because the Singleton lives longer than the Scoped service.

For example:

```
builder.Services.AddSingleton<AppCache>();
builder.Services.AddScoped<EmployeeService>();
```

If `EmployeeService` depends on `AppCache`, the dependency direction is safe from a lifetime perspective. However, the Singleton must still be thread-safe if multiple requests access it concurrently.

### Remember this rule

| Dependency relationship | Valid?                                                                       |
| ----------------------- | ---------------------------------------------------------------------------- |
| Singleton → Transient   | Generally valid, but the transient instance is retained if stored in a field |
| Singleton → Scoped      | Captive dependency — avoid direct injection                                  |
| Scoped → Singleton      | Generally valid                                                              |
| Scoped → Transient      | Generally valid                                                              |
| Transient → Scoped      | Valid when resolved within a scope                                           |

Best interview answer: “A captive dependency occurs when a longer-lived service captures a shorter-lived dependency. For example, a Singleton depending directly on a Scoped service can cause request-specific state to be shared across requests. I would fix it by aligning service lifetimes or creating an explicit DI scope using `IServiceScopeFactory` when a Singleton genuinely needs a Scoped service.”
---

# 🔥 11. What is configuration in ASP.NET Core?

Configuration can come from:

* `appsettings.json`
* `appsettings.{Environment}.json`
* environment variables
* command-line arguments
* Azure Key Vault
* custom configuration providers

Example:

```json
{
  "ConnectionStrings": {
    "Default": "..."
  }
}
```

Access using:

```csharp
var connection =
    configuration.GetConnectionString("Default");
```

---

# 🔥 12. What is the Options pattern?

Instead of accessing configuration everywhere:

```csharp
configuration["Azure:ServiceBus:Namespace"]
```

create a strongly typed configuration class:

```csharp
public class ServiceBusOptions
{
    public string Namespace { get; set; } = "";
}
```

Register:

```csharp
services.Configure<ServiceBusOptions>(
    configuration.GetSection("ServiceBus"));
```

Inject:

```csharp
IOptions<ServiceBusOptions>
```

Other variants:

```text
IOptions<T>
IOptionsSnapshot<T>
IOptionsMonitor<T>
```

---

# 🔥 13. IOptions vs IOptionsSnapshot vs IOptionsMonitor

### IOptions

Generally provides configured values and is suitable for values that don't need reload behavior.

### IOptionsSnapshot

Scoped and designed for retrieving configuration snapshots, commonly per request.

### IOptionsMonitor

Supports observing configuration changes and retrieving the current value.

Interview answer:

> "I choose based on whether the configuration needs request-scoped snapshots or runtime change notifications."



In ASP.NET Core, `IOptions`, `IOptionsSnapshot`, and `IOptionsMonitor` help you read configuration settings from `appsettings.json` in a clean, type-safe way.

Think of a configuration setting such as a payment API URL, email server, or application name.

The main difference is when you get the value and whether you can see configuration changes without restarting the application.

## 1. Real-life example

Imagine your application has this setting:

```
{
  "EmailSettings": {
    "SmtpServer": "smtp.old.com",
    "Port": 587
  }
}
```

Your application reads the email configuration. Later, someone changes `SmtpServer` to `"smtp.new.com"`.

- `IOptions<T>` — gives you the standard options value; it doesn't provide automatic refresh of the existing options instance.
- `IOptionsSnapshot<T>` — provides a cached snapshot for the current DI scope, typically one HTTP request.
- `IOptionsMonitor<T>` — can provide the current value and notify you when configuration changes, if the configuration provider supports reload.

## 2. First, create the settings class

`EmailSettings.cs`

```
public class EmailSettings
{
    public string SmtpServer { get; set; } = "";
    public int Port { get; set; }
}
```

Register the configuration in `Program.cs`:

```
builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection("EmailSettings"));
```

Now let's see how each option works.

## 3. `IOptions<T>` — Standard configuration

Use it when your application needs configuration values that generally remain fixed for the lifetime of the options instance.

```
public class EmailService
{
    private readonly EmailSettings _settings;

    public EmailService(IOptions<EmailSettings> options)
    {
        _settings = options.Value;
    }

    public void SendEmail()
    {
        Console.WriteLine(_settings.SmtpServer);
    }
}
```

Add this namespace:

```
using Microsoft.Extensions.Options;
```

What happens?

- The settings are bound to `EmailSettings`.
- `options.Value` provides the options instance.
- The standard `IOptions<T>` instance doesn't automatically refresh its value when configuration changes.

Use case: Application settings that don't need runtime refresh.

## 4. `IOptionsSnapshot<T>` — Configuration per request

Imagine an API receives 100 requests. Each request can resolve its own configuration snapshot.

```
public class EmailService
{
    private readonly EmailSettings _settings;

    public EmailService(IOptionsSnapshot<EmailSettings> options)
    {
        _settings = options.Value;
    }
}
```

Register the service as scoped:

```
builder.Services.AddScoped<EmailService>();
```

What happens?

- The options value is cached within the current DI scope.
- In a typical Web API, that means one snapshot per HTTP request.
- A later request can see updated settings if the configuration provider reloads them.

Use case: Request-scoped configuration that may change between requests.

Important: Don't inject `IOptionsSnapshot<T>` directly into a singleton service because the singleton outlives the request scope.

## 5. `IOptionsMonitor<T>` — Current configuration and change notifications

Imagine an administrator updates an email server setting while your application is running.

```
public class EmailService
{
    private readonly IOptionsMonitor<EmailSettings> _options;

    public EmailService(IOptionsMonitor<EmailSettings> options)
    {
        _options = options;
    }

    public void SendEmail()
    {
        var settings = _options.CurrentValue;

        Console.WriteLine(settings.SmtpServer);
    }
}
```

You can also listen for changes:

```
_options.OnChange(settings =>
{
    Console.WriteLine(
        $"New SMTP server: {settings.SmtpServer}");
});
```

What happens?

- `CurrentValue` retrieves the current options value.
- `OnChange()` lets you react to configuration changes.
- It works with supported reloadable configuration providers, such as JSON configuration with reload enabled.

Use case: Long-running services, background workers, or settings that need runtime updates.

## 6. Quick comparison

| Feature              | `IOptions<T>`        | `IOptionsSnapshot<T>`  | `IOptionsMonitor<T>`     |
| -------------------- | -------------------- | ---------------------- | ------------------------ |
| Value access         | `Value`              | `Value`                | `CurrentValue`           |
| Refresh behavior     | No automatic refresh | New snapshot per scope | Current value can update |
| Change notification  | No                   | No                     | `OnChange()`             |
| Lifetime suitability | Singleton-safe       | Scoped                 | Singleton-safe           |
| Typical use          | Stable settings      | Per-request settings   | Runtime updates          |

## 7. Interview memory trick

IOptions

Read the standard configuration value.

Snapshot

One cached snapshot per scope, typically per request.

Monitor

Current value + change notifications.

Staff Engineer interview answer:

“I use `IOptions<T>` for standard configuration, `IOptionsSnapshot<T>` when I need a scoped configuration snapshot, and `IOptionsMonitor<T>` when a long-running service needs to observe configuration changes. I also ensure the underlying configuration provider supports reload and that my application handles changes safely.”


---

# 🔥 14. What is ASP.NET Core Web API?

A framework for building HTTP-based APIs.

Typical flow:

```text
HTTP Request
     ↓
Controller
     ↓
Service
     ↓
Repository / EF Core
     ↓
Database
```

Example:

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<IActionResult> Get(int id)
    {
        ...
    }
}
```

---

# 🔥 15. Controller vs Minimal API

Controller:

```csharp
[ApiController]
public class OrdersController : ControllerBase
{
}
```

Minimal API:

```csharp
app.MapGet("/orders/{id}", async (int id) =>
{
});
```

Minimal APIs can be useful for lightweight endpoints and services.

Controllers can be preferable when you need:

* larger API surface
* conventions
* filters
* controller organization
* complex API behavior

Don't claim one is universally better.

---

# 🔥 16. What is model binding?

ASP.NET Core maps HTTP request data to action parameters/models.

For example:

```http
GET /orders/100
```

```csharp
[HttpGet("{id}")]
public IActionResult Get(int id)
{
}
```

`100` is bound to `id`.

It can bind from:

* route
* query string
* headers
* body
* form data

---

# 🔥 17. What is model validation?

With:

```csharp
public class CreateEmployeeRequest
{
    [Required]
    public string Name { get; set; } = "";

    [Range(18, 100)]
    public int Age { get; set; }
}
```

ASP.NET Core can validate the model.

With `[ApiController]`, invalid model state can automatically result in a 400 response.

---

# 🔥 18. Authentication vs Authorization

### Authentication

**Who are you?**

```text
User
 ↓
Identity Provider
 ↓
Authenticated identity
```

### Authorization

**What are you allowed to do?**

```text
Authenticated User
       ↓
Role / Policy
       ↓
Allow / Deny
```

Example:

```csharp
[Authorize]
```

or:

```csharp
[Authorize(Policy = "AdminOnly")]
```

---

# 🔥 19. JWT authentication flow

Typical flow:

```text
Client
  ↓
Login
  ↓
Identity Provider / Entra ID
  ↓
Access Token
  ↓
API
  ↓
Validate JWT
  ↓
Claims
  ↓
Authorization
```

For OAuth/OIDC-based systems, keep the distinction clear:

```text
OAuth 2.0 → authorization framework
OIDC       → authentication layer built on OAuth 2.0
JWT        → token format often used for access/id tokens
```

---

# 🔥 20. What is a filter?

Filters provide cross-cutting behavior around MVC/controller execution.

Examples:

* Authorization filters
* Action filters
* Exception filters
* Result filters
* Resource filters

Middleware works at the broader HTTP pipeline level, while MVC filters operate within the MVC/request execution model.

---

# 🔥 21. Middleware vs filters

This is a common Staff question.

### Middleware

```text
HTTP Pipeline
```

Applies broadly.

### Filters

```text
MVC / Controller Pipeline
```

Applies more specifically to controller/action processing.

Use middleware for things like:

* global exception handling
* correlation IDs
* request logging
* authentication infrastructure

Use filters for MVC-specific concerns.

---

# 🔥 22. What is global exception handling?

Don't put this everywhere:

```csharp
try
{
}
catch(Exception ex)
{
}
```

Instead use centralized exception handling.

For example:

```csharp
app.UseExceptionHandler();
```

or an appropriate custom exception-handling middleware.

Return consistent errors such as:

```json
{
  "title": "Order processing failed",
  "status": 500,
  "traceId": "..."
}
```

Modern ASP.NET Core supports standardized problem details through `ProblemDetails`.

---

# 🔥 23. What is REST?

REST is an architectural style for networked resources.

Example:

```text
GET    /api/orders/10
POST   /api/orders
PUT    /api/orders/10
PATCH  /api/orders/10
DELETE /api/orders/10
```

Important REST concepts:

* resources
* statelessness
* HTTP semantics
* appropriate status codes
* cacheability where applicable

---

# 🔥 24. PUT vs PATCH

### PUT

Usually represents replacement of a resource representation.

```http
PUT /employees/10
```

### PATCH

Represents a partial modification.

```http
PATCH /employees/10
```

For example:

```json
{
  "phoneNumber": "1234567890"
}
```

---

# 🔥 25. What HTTP status codes should you know?

```text
200 → OK
201 → Created
202 → Accepted
204 → No Content

400 → Bad Request
401 → Unauthorized
403 → Forbidden
404 → Not Found
409 → Conflict
422 → Unprocessable Content

429 → Too Many Requests

500 → Internal Server Error
502 → Bad Gateway
503 → Service Unavailable
504 → Gateway Timeout
```

Important:

```text
401 → Authentication problem
403 → Authenticated but not allowed
```

---

# 🔥 26. How do you version an API?

Common approaches:

```text
/api/v1/orders
/api/v2/orders
```

or header/media-type based versioning.

For public APIs, versioning allows you to evolve contracts without unexpectedly breaking existing consumers.

---

# 🔥 27. What is CORS?

CORS controls whether a browser-based application from one origin can make requests to another origin.

Example:

```text
Angular
localhost:4200
      ↓
.NET API
localhost:5000
```

Different origins require appropriate CORS policy.

CORS is primarily a **browser security mechanism**, not an API authentication mechanism.

---

# 🔥 28. What is caching in .NET?

Types include:

### In-memory cache

```csharp
IMemoryCache
```

Works within one application instance.

### Distributed cache

```csharp
IDistributedCache
```

Useful when multiple application instances need shared cached state.

Architecture:

```text
                 Load Balancer
                /     |      \
               API   API     API
                \     |      /
                   Redis
```

For distributed systems, a local `MemoryCache` isn't a shared cache.

---

# 🔥 29. What is cache stampede?

Suppose an expensive cached item expires:

```text
Cache expires
     ↓
100 requests arrive
     ↓
100 requests query database
```

This can overload the database.

Solutions include:

* distributed locking
* request coalescing/single-flight
* refresh-ahead
* jittered expiration
* stale-while-revalidate strategies where appropriate

A local `SemaphoreSlim` only coordinates callers within **one application instance**. In a multi-instance deployment, you may need distributed coordination or a design that avoids requiring a distributed lock.

---

# 🔥 30. How do you improve .NET API performance?

I'd investigate in this order:

```text
Request
  ↓
Application metrics
  ↓
Database query
  ↓
External APIs
  ↓
Serialization
  ↓
CPU / memory
  ↓
Network
```

Typical optimizations:

* async I/O
* efficient EF Core queries
* projection
* indexes
* caching
* pagination
* compression where appropriate
* response size reduction
* connection pooling
* avoiding unnecessary allocations
* appropriate concurrency limits

The key Staff-level point:

> **Measure first, optimize the actual bottleneck, and benchmark the change.**

---

# 🔥 31. What is ThreadPool starvation?

If application threads are blocked for long periods:

```text
ThreadPool
 ↓
Threads blocked
 ↓
New requests wait
 ↓
Latency increases
 ↓
Throughput decreases
```

Common causes include:

```csharp
Task.Result
Task.Wait()
Thread.Sleep()
```

inside server request paths.

Prefer asynchronous APIs:

```csharp
await SomeOperationAsync();
```

---

# 🔥 32. Why should you avoid `.Result` and `.Wait()`?

This:

```csharp
var result = GetDataAsync().Result;
```

blocks the current thread.

In server applications this can contribute to:

* ThreadPool starvation
* poor scalability
* deadlock scenarios in environments with synchronization contexts

Prefer:

```csharp
var result = await GetDataAsync();
```

---

# 🔥 33. What is connection pooling?

Database connections are expensive to establish.

ADO.NET providers maintain a pool of reusable connections.

Conceptually:

```text
Application
     ↓
Connection Pool
 ┌───┼───┐
 C1  C2  C3
     ↓
 Database
```

Properly disposing/closing connections returns them to the pool rather than necessarily closing the physical connection.

---

# 🔥 34. What is HttpClientFactory?

Avoid creating a new `HttpClient` for every request.

Use:

```csharp
services.AddHttpClient<IOrderClient, OrderClient>();
```

Benefits include:

* handler lifecycle management
* connection reuse
* central configuration
* resilience integration
* named/typed clients

Example:

```csharp
public class OrderClient
{
    private readonly HttpClient _client;

    public OrderClient(HttpClient client)
    {
        _client = client;
    }
}
```

---

# 🔥 35. What is retry and why can retry be dangerous?

Example:

```text
API
 ↓
External service
 ↓
Timeout
 ↓
Retry
```

But if 1,000 requests all retry simultaneously:

```text
1000 original requests
+
3000 retries
      ↓
External service overloaded
```

This can create a retry storm.

Use:

* exponential backoff
* jitter
* bounded retries
* timeout
* circuit breaker
* idempotency

And only retry failures that are genuinely transient.

---

# 🔥 36. What is a circuit breaker?

Suppose a downstream service is failing continuously.

Without circuit breaking:

```text
API
 ↓
Service B ❌
 ↓
Retry
 ↓
Service B ❌
 ↓
Retry
```

Circuit breaker:

```text
Closed
  ↓
Failures exceed threshold
  ↓
Open
  ↓
Fail fast
  ↓
After delay
  ↓
Half Open
  ↓
Test request
  ↓
Success → Closed
```

This protects the system and downstream service.

---

# 🔥 37. What is rate limiting?

Controls how many requests a client can make.

For example:

```text
100 requests/minute/client
```

Useful for:

* protecting APIs
* preventing abuse
* protecting downstream dependencies
* maintaining predictable capacity

ASP.NET Core provides rate-limiting capabilities that can be configured according to the workload.

---

# 🔥 38. How do you handle high-volume requests?

For example:

```text
10,000 requests/sec
```

I'd consider:

```text
                 Front Door
                     ↓
                API Gateway
                     ↓
                Load Balancer
                     ↓
          ┌──────────┼──────────┐
          ↓          ↓          ↓
        API 1      API 2      API 3
          │          │          │
          └──────┬───┴──────────┘
                 ↓
               Cache
                 ↓
             Service Bus
                 ↓
              Workers
                 ↓
             Database
```

Key concepts:

* horizontal scaling
* stateless APIs
* caching
* asynchronous processing
* queues
* backpressure
* database optimization
* rate limiting
* observability

---

# 🔥 39. What is backpressure?

Backpressure prevents a fast producer from overwhelming a slower consumer.

Example with Azure Service Bus:

```text
Producer
  ↓
Queue
  ↓
Consumer
```

Consumer configuration:

```csharp
var options = new ServiceBusProcessorOptions
{
    MaxConcurrentCalls = 4,
    PrefetchCount = 20
};
```

This is primarily **consumer-side processing configuration**.

You control how much concurrent work the consumer performs rather than allowing unlimited processing.

---

# 🔥 40. What is graceful shutdown?

When an application is shutting down:

```text
Shutdown signal
     ↓
Stop accepting new work
     ↓
Finish in-flight work
     ↓
Dispose resources
     ↓
Exit
```

This is particularly important for:

* Kubernetes
* containers
* background workers
* Service Bus consumers

Use cancellation tokens and hosted-service lifecycle methods to implement graceful shutdown.

---

# 🔥 41. What is `IHostedService` / `BackgroundService`?

For background processing:

```csharp
public class OrderWorker : BackgroundService
{
    protected override async Task ExecuteAsync(
        CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessAsync(stoppingToken);
        }
    }
}
```

Typical uses:

* queue consumers
* scheduled processing
* background jobs
* polling

For production workloads, avoid creating an uncontrolled infinite loop without cancellation, delay/backoff, and proper error handling.

---

# 🔥 42. What is health check?

Health checks expose the application's operational state.

```text
/health
```

You can have:

### Liveness

> Is the process alive?

### Readiness

> Can this instance safely receive traffic?

Example:

```text
Load Balancer
     ↓
/health/ready
     ↓
API
     ↓
Database / dependencies
```

Don't make liveness depend on every external dependency; otherwise a temporary dependency failure can cause healthy instances to be unnecessarily restarted.

---

# 🔥 43. What is OpenTelemetry?

OpenTelemetry provides standardized observability for:

* traces
* metrics
* logs

For distributed systems:

```text
Request
 ↓
API
 ↓
Service A
 ↓
Service Bus
 ↓
Service B
 ↓
Database
```

A trace/correlation ID lets you follow the request across components.

---

# 🔥 44. What is structured logging?

Instead of:

```csharp
logger.LogInformation(
    $"Order {orderId} processed");
```

prefer structured logging:

```csharp
logger.LogInformation(
    "Order {OrderId} processed",
    orderId);
```

Structured properties can then be queried by logging platforms.

---

# 🔥 45. How do you secure a .NET API?

At Staff level, mention multiple layers:

```text
Authentication
      ↓
Authorization
      ↓
Input validation
      ↓
HTTPS
      ↓
Secrets management
      ↓
Rate limiting
      ↓
Secure headers
      ↓
Logging/auditing
      ↓
Dependency vulnerability scanning
```

And specifically:

* OAuth 2.0/OIDC
* Entra ID
* JWT validation
* policies/roles
* managed identities
* Azure Key Vault
* parameterized SQL/EF Core
* least privilege

---

# 🔥 46. What is Clean Architecture in .NET?

A common structure:

```text
Presentation
     ↓
Application
     ↓
Domain

Infrastructure
     ↑
implements interfaces
```

Typical projects:

```text
MyApp.API
MyApp.Application
MyApp.Domain
MyApp.Infrastructure
```

The important principle is:

> Dependencies should point toward the domain/business rules rather than allowing business logic to depend directly on infrastructure.

---

# 🔥 47. What is CQRS?

CQRS separates:

```text
Command → Change state
Query   → Read state
```

Example:

```text
POST /orders
      ↓
CreateOrderCommand

GET /orders/100
      ↓
GetOrderQuery
```

It doesn't automatically require separate databases.

Use CQRS when the separation provides meaningful benefits; don't introduce it merely because it is fashionable.

---

# 🔥 48. What is API Gateway?

An API Gateway provides a controlled entry point into backend services.

```text
Clients
   ↓
API Gateway
   ↓
┌──────┬──────┬──────┐
Orders Users Payments
```

It can provide:

* routing
* authentication
* authorization
* rate limiting
* throttling
* transformation
* observability

Azure API Management is a common example.

---

# 🔥 49. API Management vs Load Balancer

### Load Balancer

Primarily distributes traffic:

```text
Client
  ↓
Load Balancer
  ↓
API1 API2 API3
```

### API Management

Provides API governance/management:

```text
Client
  ↓
APIM
 ↓
Authentication
Rate Limit
Policies
Logging
 ↓
Backend APIs
```

They solve different problems and can be used together.

---

# 🔥 50. What is the most important .NET performance checklist?

For your Staff interview, remember:

```text
.NET PERFORMANCE
       │
       ├── Async I/O
       ├── Avoid blocking
       ├── ThreadPool health
       ├── HttpClientFactory
       ├── Connection pooling
       ├── EF Core optimization
       ├── Caching
       ├── Pagination
       ├── Serialization
       ├── Memory allocations
       ├── GC pressure
       ├── Rate limiting
       ├── Backpressure
       ├── Resilience
       └── Observability
```

# 🎯 Top 30 questions I'd prioritize for you

Given your **13+ years of .NET/C# experience**, I would make these your **must-answer-without-notes** questions:

1. **What happens when an ASP.NET Core application starts?**
2. **Explain the middleware pipeline.**
3. **Why does middleware ordering matter?**
4. **Transient vs Scoped vs Singleton?**
5. **What is a captive dependency?**
6. **How does DI work internally?**
7. **IOptions vs IOptionsSnapshot vs IOptionsMonitor?**
8. **Authentication vs Authorization?**
9. **OAuth 2.0 vs OIDC vs JWT?**
10. **How does JWT authentication work?**
11. **Middleware vs filters?**
12. **How do you implement global exception handling?**
13. **How do you optimize a slow API?**
14. **What causes ThreadPool starvation?**
15. **Why avoid `.Result`/`.Wait()`?**
16. **Task vs Thread?**
17. **Task.WhenAll vs Parallel?**
18. **CancellationToken and graceful shutdown?**
19. **HttpClientFactory and connection management?**
20. **Caching and cache stampede?**
21. **Retry vs circuit breaker?**
22. **How do you implement backpressure?**
23. **How would you handle 10,000 requests/sec?**
24. **IHostedService vs BackgroundService?**
25. **Liveness vs readiness?**
26. **API Gateway vs Load Balancer?**
27. **Clean Architecture?**
28. **CQRS — when and why?**
29. **How do you secure a production .NET API?**
30. **How would you upgrade a .NET 8 application to .NET 10?**

### The Staff-level answering formula

For almost every question, use:

**Definition → How it works → Real project example → Trade-off → When you would/ wouldn't use it.**

For example, don't answer merely:

> "Scoped means one instance per request."

Instead:

> "Scoped means one service instance is normally created per DI scope, which in ASP.NET Core is typically the lifetime of an HTTP request. I commonly use scoped lifetime for request-oriented services and EF Core DbContext. I avoid injecting scoped dependencies into singletons because that creates a lifetime mismatch. For a high-throughput API, I also make sure the scoped service isn't holding expensive resources longer than necessary."

That style will position you much better for a **Staff Engineer interview** than memorizing definitions.
