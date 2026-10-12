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

 ASP.NET Core Web API is a framework for building HTTP services using .NET. It allows applications such as Angular, React, mobile apps, and other backend services to communicate with your server and exchange data, commonly using JSON.

## 1. Real-life example

Imagine you are building an Employee Management application.

- Angular displays the employee list.
- ASP.NET Core Web API receives requests and applies business logic.
- SQL Server stores employee information.

Angular Frontend

Requests employee data

ASP.NET Core Web API

Controllers · Services · Validation · Authentication

SQL Server

Stores employee records

## 2. Simple C# example

Create an API controller that returns employee data.

```
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class EmployeesController : ControllerBase
{
    [HttpGet]
    public IActionResult GetEmployees()
    {
        var employees = new[]
        {
            new { Id = 1, Name = "Rahul" },
            new { Id = 2, Name = "Priya" }
        };

        return Ok(employees);
    }
}
```

When a client sends a GET request to:

```
GET /api/employees
```

The API returns an HTTP `200 OK` response with JSON:

```
[
  { "id": 1, "name": "Rahul" },
  { "id": 2, "name": "Priya" }
]
```

Here:

- `[ApiController]` enables API-specific behavior.
- `[Route("api/[controller]")]` defines the URL pattern.
- `[HttpGet]` maps HTTP GET requests to the method.
- `Ok()` returns an HTTP 200 response.

This example uses in-memory data; a production API would typically retrieve records through a service and database.

## 3. Common HTTP methods in Web API

| Method   | Purpose                   | Example               |
| -------- | ------------------------- | --------------------- |
| `GET`    | Retrieve data             | Get employees         |
| `POST`   | Create data               | Add employee          |
| `PUT`    | Replace/update a resource | Update employee       |
| `PATCH`  | Partially update data     | Change employee email |
| `DELETE` | Delete data               | Remove employee       |

## 4. Important features of ASP.NET Core Web API

- Middleware: Handles authentication, logging, exception handling, and other request-processing concerns.
- Dependency Injection: Supplies services to controllers and other components.
- Routing: Maps URLs and HTTP methods to endpoints.
- Model validation: Checks incoming request data.
- Authentication and authorization: Protects endpoints.
- Async/await: Supports scalable asynchronous I/O.
- Entity Framework Core: Enables database access through .NET objects.
- Swagger/OpenAPI: Documents and helps test API endpoints.

## 5. Interview answer

“ASP.NET Core Web API is a cross-platform framework for building HTTP-based services in .NET. It exposes endpoints that clients can consume using standard HTTP methods and commonly exchange JSON data. It provides routing, middleware, dependency injection, authentication, authorization, validation, and asynchronous processing. I use it to build secure, scalable backend services that integrate with databases, frontend applications, and other services.”

Remember: Frontend sends the request → Web API processes it → Database provides or stores data → Web API returns the response.

---

# 🔥 15. Controller vs Minimal API

 As a Senior/Staff .NET Engineer, the key difference is:

Controller-based API = organize HTTP endpoints into controller classes.

Minimal API = define HTTP endpoints directly with route handlers, usually in `Program.cs` or separate endpoint-mapping modules.

Both are fully supported approaches in ASP.NET Core. Both can use dependency injection, middleware, authentication, authorization, validation, and asynchronous database calls.

The choice is mainly about code organization, API complexity, team conventions, and maintainability—not simply performance.

## 1. Real-world scenario

Imagine you're building an Employee Management API.

- `GET /api/employees` — retrieve employees
- `GET /api/employees/{id}` — retrieve one employee
- `POST /api/employees` — create an employee
- `DELETE /api/employees/{id}` — delete an employee

With controllers, you group related operations into an `EmployeesController`.

With Minimal APIs, you map the routes using methods such as `MapGet`, `MapPost`, and `MapDelete`.

## 2. Controller-based API example

A controller groups related endpoints into a class.

```
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class EmployeesController : ControllerBase
{
    [HttpGet]
    public IActionResult GetEmployees()
    {
        return Ok(new[] { "Rahul", "Priya" });
    }

    [HttpGet("{id:int}")]
    public IActionResult GetEmployee(int id)
    {
        return Ok(new { Id = id, Name = "Rahul" });
    }

    [HttpPost]
    public IActionResult CreateEmployee(string name)
    {
        return Created("/api/employees/1",
            new { Id = 1, Name = name });
    }
}
```

Routes generated:

- `GET /api/employees`
- `GET /api/employees/10`
- `POST /api/employees`

Why use controllers?

- Related endpoints are grouped in one class.
- Attributes support routing, filters, and API conventions.
- Useful for large APIs with complex request handling.
- Multiple developers can work on different controllers.

## 3. Minimal API example

Minimal APIs define endpoints directly using route-mapping methods.

```
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/api/employees", () =>
{
    return Results.Ok(new[] { "Rahul", "Priya" });
});

app.MapGet("/api/employees/{id:int}", (int id) =>
{
    return Results.Ok(new { Id = id, Name = "Rahul" });
});

app.MapPost("/api/employees", (string name) =>
{
    return Results.Created(
        "/api/employees/1",
        new { Id = 1, Name = name });
});

app.Run();
```

The same basic endpoints are available without creating a controller class.

Why use Minimal APIs?

- Less ceremony for straightforward endpoints.
- Easy to create small services and prototypes.
- Convenient for health checks, webhooks, and lightweight microservices.
- Can be organized into separate files using extension methods and route groups.

For example, `app.MapGroup("/api/employees")` lets you group related routes and apply shared configuration.

## 4. Controller vs Minimal API — comparison

| Feature               | Controllers                          | Minimal APIs                                                    |
| --------------------- | ------------------------------------ | --------------------------------------------------------------- |
| Organization          | Controller classes                   | Route handlers and endpoint modules                             |
| Routing               | Attributes such as `[HttpGet]`       | `MapGet()`, `MapPost()`                                         |
| Dependency injection  | Constructor injection                | Constructor injection in supporting types or handler parameters |
| Validation            | Model validation and API conventions | Validation support depends on the .NET version and approach     |
| Filters               | MVC filters                          | Endpoint filters                                                |
| Middleware            | Supported                            | Supported                                                       |
| Authorization         | Supported                            | Supported                                                       |
| Large enterprise APIs | Strong class-based organization      | Equally possible with good endpoint modularization              |
| Boilerplate           | Generally more                       | Generally less                                                  |

## 5. Staff Engineer perspective: Which should you choose?

### Choose controllers when...

- The application has many related business endpoints.
- You benefit from MVC conventions, action filters, and controller-level organization.
- Multiple teams need a consistent, familiar API structure.
- The existing enterprise codebase already uses controllers.

### Choose Minimal APIs when...

- You are building a small service or a focused set of endpoints.
- You want less framework ceremony.
- You are creating health endpoints, webhooks, or lightweight service APIs.
- Your team can maintain a clear endpoint-module structure as the application grows.

A Staff Engineer should evaluate maintainability, testability, consistency, team familiarity, and operational requirements. Neither approach automatically creates a more scalable API.

## 6. How I would structure an enterprise Minimal API

Avoid putting hundreds of endpoints into `Program.cs`. Separate them into modules:

```
// Program.cs
app.MapEmployeeEndpoints();
app.MapOrderEndpoints();
```

```
public static class EmployeeEndpoints
{
    public static IEndpointRouteBuilder MapEmployeeEndpoints(
        this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/employees");

        group.MapGet("/", GetEmployees);
        group.MapGet("/{id:int}", GetEmployee);

        return app;
    }

    private static IResult GetEmployees()
    {
        return Results.Ok(new[] { "Rahul", "Priya" });
    }

    private static IResult GetEmployee(int id)
    {
        return Results.Ok(new { Id = id, Name = "Rahul" });
    }
}
```

This gives Minimal APIs a clear structure while keeping HTTP handling separate from business logic. In a production application, those handlers should typically delegate to application services rather than contain business rules themselves.

## 7. Interview answer to remember

“Both controller-based APIs and Minimal APIs are supported in ASP.NET Core. Controllers organize endpoints into classes and provide MVC features such as action filters and controller conventions. Minimal APIs use route handlers and reduce boilerplate. I choose based on the complexity of the API, team conventions, and maintainability. For either approach, I keep business logic in application services, use dependency injection, and apply consistent validation, authorization, logging, and error handling.”

Memory trick: Controllers = class-based endpoint organization. Minimal APIs = route-handler-based endpoint organization. Both can support enterprise-grade applications.

---

# 🔥 16. What is model binding?
In ASP.NET Core Web API, Model Binding is the process of taking data from an HTTP request and automatically putting it into C# parameters or objects.

Remember: Request data → C# variables or model objects.

## 1. Real-life example

Imagine your Angular application sends employee details to your .NET API:

```
{
  "name": "Rahul",
  "age": 30
}
```

Model binding helps ASP.NET Core convert this incoming data into a C# object so your API can process it.

## 2. Example 1: Model binding from the URL

Request:

```
GET /api/employees/101
```

Controller:

```
[HttpGet("{id}")]
public IActionResult GetEmployee(int id)
{
    return Ok($"Employee ID: {id}");
}
```

Output:

```
Employee ID: 101
```

ASP.NET Core automatically binds `101` from the URL to the `id` parameter.

## 3. Example 2: Model binding from the query string

Request:

```
GET /api/employees?department=IT
```

Controller:

```
[HttpGet]
public IActionResult GetEmployees(string department)
{
    return Ok($"Department: {department}");
}
```

Output:

```
Department: IT
```

The query-string value `IT` is bound to the `department` parameter.

## 4. Example 3: Model binding from the request body

Create a model:

```
public class Employee
{
    public string Name { get; set; } = "";
    public int Age { get; set; }
}
```

Controller:

```
[HttpPost]
public IActionResult CreateEmployee([FromBody] Employee employee)
{
    return Ok($"Created: {employee.Name}, Age: {employee.Age}");
}
```

Request body:

```
{
  "name": "Rahul",
  "age": 30
}
```

Output:

```
Created: Rahul, Age: 30
```

ASP.NET Core uses an input formatter, typically the JSON formatter, to deserialize the JSON body into the `Employee` object. In a controller marked with `[ApiController]`, complex parameters are generally inferred as coming from the request body.

## 5. Where does model binding get data?

| Attribute      | Data source                        | Example              |
| -------------- | ---------------------------------- | -------------------- |
| `[FromRoute]`  | URL path                           | `/employees/101`     |
| `[FromQuery]`  | Query string                       | `?department=IT`     |
| `[FromBody]`   | Request body                       | JSON employee object |
| `[FromHeader]` | HTTP headers                       | `X-Correlation-ID`   |
| `[FromForm]`   | Form fields or multipart form data | File upload          |

Example using explicit attributes:

```
[HttpGet("{id}")]
public IActionResult GetEmployee(
    [FromRoute] int id,
    [FromQuery] string department)
{
    return Ok(new { id, department });
}
```

Request:

```
GET /api/employees/101?department=IT
```

Output:

```
{
  "id": 101,
  "department": "IT"
}
```

## 6. Model binding vs model validation

These concepts work together but are different.

- Model binding: Reads and converts request data into C# values.
- Model validation: Checks whether those values satisfy validation rules.

For example:

```
public class Employee
{
    [Required]
    public string Name { get; set; } = "";

    [Range(18, 65)]
    public int Age { get; set; }
}
```

Here, model binding populates `Name` and `Age`. Validation checks whether `Name` is supplied and `Age` is between 18 and 65. With `[ApiController]`, invalid model state normally produces an automatic HTTP `400 Bad Request` response.

## Interview answer

“Model binding in ASP.NET Core automatically maps values from an HTTP request—such as route parameters, query strings, headers, forms, and request bodies—to action parameters or model objects. It reduces manual parsing and works with model validation to help ensure incoming data is valid.”

Memory trick: Model binding = Get request data into C# objects. Model validation = Check whether that data is valid.
 
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

In ASP.NET Core Web API, a filter is code that runs at a specific stage of processing an HTTP request or response. It lets you handle common tasks without repeating the same code in every controller action.

Think of a filter as a security checkpoint: before an action executes, you can check something; after it executes, you can log or modify the result.

## 1. Real-life example

Imagine your Employee API has 50 endpoints. You want to log which action is called.

Without a filter, you might write logging code in all 50 actions.

With a filter, you can centralize that logic and reuse it across controllers or actions.

## 2. Types of filters in ASP.NET Core MVC

| Filter        | When it runs                                                    | Typical use                      |
| ------------- | --------------------------------------------------------------- | -------------------------------- |
| Authorization | Before most other filter stages                                 | Check access permissions         |
| Resource      | Before and after the rest of the MVC pipeline                   | Caching or short-circuiting work |
| Action        | Before and after a controller action                            | Logging, timing                  |
| Exception     | When an unhandled exception occurs in the relevant MVC pipeline | Handle action exceptions         |
| Result        | Before and after action-result execution                        | Result processing                |

Important: Middleware and filters are different. Middleware runs in the HTTP pipeline; MVC filters run at specific stages of MVC processing.

## 3. Example: Create an action filter for logging

Suppose you want to log whenever an API action is called.

### Step 1: Create the filter

```
using Microsoft.AspNetCore.Mvc.Filters;

public class LoggingFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        Console.WriteLine("Before action executes");
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        Console.WriteLine("After action executes");
    }
}
```

- `OnActionExecuting()` runs before the controller action.
- `OnActionExecuted()` runs after the action has executed.

### Step 2: Apply the filter

```
[ServiceFilter(typeof(LoggingFilter))]
[HttpGet]
public IActionResult GetEmployees()
{
    return Ok(new[] { "Rahul", "Priya" });
}
```

Register the filter in `Program.cs`:

```
builder.Services.AddScoped<LoggingFilter>();
```

When you call `GET /api/employees`, the console shows:

```
Before action executes
After action executes
```

## 4. Apply a filter globally

If you want the filter to run for all controller actions, register it globally:

```
builder.Services.AddScoped<LoggingFilter>();

builder.Services.AddControllers(options =>
{
    options.Filters.AddService<LoggingFilter>();
});
```

Now the filter applies to controller actions throughout the application.

## 5. Filter vs Middleware

| Feature                        | Middleware                                          | Filter                                     |
| ------------------------------ | --------------------------------------------------- | ------------------------------------------ |
| Runs in                        | HTTP request pipeline                               | MVC/action pipeline                        |
| Knows about controller actions | Not inherently                                      | Yes                                        |
| Common uses                    | Exception handling, authentication, request logging | Action logging, action-specific validation |
| Applies to                     | Requests passing through its pipeline position      | MVC actions/results in its scope           |

For example, use middleware for global exception handling and request correlation. Use an action filter when you need information about a specific controller action or its arguments.

## 6. Interview answer

“Filters in ASP.NET Core MVC allow us to execute reusable logic before or after specific stages of action processing. Common types include authorization, resource, action, exception, and result filters. I use them for cross-cutting concerns such as action logging and timing, while middleware is better suited to concerns that apply across the HTTP pipeline.”

Memory trick: Middleware handles the HTTP pipeline; filters handle specific MVC processing stages.


Let's understand these three ASP.NET Core filters with practical scenarios from an enterprise Web API.

- Resource filter: Runs before and after most of the MVC action-processing pipeline. Useful for short-circuiting requests or implementing MVC-level caching.
- Exception filter: Handles unhandled exceptions that occur within the MVC action/filter pipeline.
- Result filter: Runs before and after an MVC action result executes, useful for result-processing concerns.

A simple way to remember them:

| Filter    | Think of it as                   |
| --------- | -------------------------------- |
| Resource  | Check before doing the main work |
| Exception | Handle an unexpected failure     |
| Result    | Process the response result      |

## 1. Resource Filter — Stop a request before the action executes

Scenario: Your application is in maintenance mode. You want to return `503 Service Unavailable` without executing the controller action.

### Step 1: Create the filter

```
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

public class MaintenanceModeFilter : IResourceFilter
{
    public void OnResourceExecuting(ResourceExecutingContext context)
    {
        bool isMaintenanceMode = true;

        if (isMaintenanceMode)
        {
            context.Result = new ObjectResult("System under maintenance")
            {
                StatusCode = StatusCodes.Status503ServiceUnavailable
            };
        }
    }

    public void OnResourceExecuted(ResourceExecutedContext context)
    {
        // Runs after the resource pipeline completes.
    }
}
```

### Step 2: Apply it to a controller

```
[ServiceFilter(typeof(MaintenanceModeFilter))]
[ApiController]
[Route("api/[controller]")]
public class EmployeesController : ControllerBase
{
    [HttpGet]
    public IActionResult GetEmployees()
    {
        Console.WriteLine("Action executed!");

        return Ok(new[] { "Rahul", "Priya" });
    }
}
```

Register the filter in `Program.cs`:

```
builder.Services.AddScoped<MaintenanceModeFilter>();
```

What happens?

When you call `GET /api/employees`, the filter returns `503`. The `GetEmployees()` action does not execute.

Other use cases: Short-circuiting requests, checking resource-level conditions, and MVC-level caching.

Staff-level note: For real maintenance mode, use a configurable feature flag or middleware if the rule should apply to the entire HTTP pipeline.

## 2. Exception Filter — Handle an action exception

Scenario: An employee API throws an unexpected exception. You want to return a controlled error response rather than expose internal exception details.

### Step 1: Create the filter

```
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

public class ApiExceptionFilter : IExceptionFilter
{
    private readonly ILogger<ApiExceptionFilter> _logger;

    public ApiExceptionFilter(ILogger<ApiExceptionFilter> logger)
    {
        _logger = logger;
    }

    public void OnException(ExceptionContext context)
    {
        _logger.LogError(
            context.Exception,
            "An error occurred while processing the request.");

        context.Result = new ObjectResult(
            new { Message = "An unexpected error occurred." })
        {
            StatusCode = StatusCodes.Status500InternalServerError
        };

        context.ExceptionHandled = true;
    }
}
```

### Step 2: Apply it to a controller

```
[ServiceFilter(typeof(ApiExceptionFilter))]
[ApiController]
[Route("api/[controller]")]
public class EmployeesController : ControllerBase
{
    [HttpGet]
    public IActionResult GetEmployees()
    {
        throw new Exception("Database connection failed!");
    }
}
```

Register it:

```
builder.Services.AddScoped<ApiExceptionFilter>();
```

What happens?

1. The controller action throws an exception.
2. The exception filter logs the exception.
3. It returns a generic HTTP `500` response.
4. `ExceptionHandled = true` marks the exception as handled by MVC.

Example response:

```
{
  "message": "An unexpected error occurred."
}
```

Other use cases: MVC-specific exception handling and mapping known exceptions to appropriate HTTP responses.

Staff-level note: For centralized exception handling across the HTTP pipeline, prefer ASP.NET Core exception-handling middleware (`UseExceptionHandler`) or a suitable `IExceptionHandler`. Exception filters only handle exceptions within the relevant MVC pipeline stages; they are not a universal replacement for middleware.

## 3. Result Filter — Process the response result

Scenario: Your API team wants to add a custom response header to controller responses.

### Step 1: Create the filter

```
using Microsoft.AspNetCore.Mvc.Filters;

public class ApiResultFilter : IResultFilter
{
    public void OnResultExecuting(ResultExecutingContext context)
    {
        context.HttpContext.Response.Headers["X-API-Version"] = "1.0";

        Console.WriteLine("Preparing API response");
    }

    public void OnResultExecuted(ResultExecutedContext context)
    {
        Console.WriteLine("API result execution completed");
    }
}
```

### Step 2: Apply it to the controller

```
[ServiceFilter(typeof(ApiResultFilter))]
[ApiController]
[Route("api/[controller]")]
public class EmployeesController : ControllerBase
{
    [HttpGet]
    public IActionResult GetEmployees()
    {
        return Ok(new[] { "Rahul", "Priya" });
    }
}
```

Register it:

```
builder.Services.AddScoped<ApiResultFilter>();
```

What happens?

When you call `GET /api/employees`:

- `OnResultExecuting()` runs before the action result is executed.
- The response includes `X-API-Version: 1.0`.
- `OnResultExecuted()` runs after result execution.

Other use cases: Result logging, response metadata, and consistent processing of MVC action results.

Note: A result filter can modify result execution, but it is not guaranteed to run after every kind of response, such as a response that has already started or a short-circuited request.

## 4. Quick interview comparison

| Filter    | Main purpose                          | Example                   |
| --------- | ------------------------------------- | ------------------------- |
| Resource  | Short-circuit before action execution | Maintenance mode          |
| Exception | Handle MVC pipeline exceptions        | Return a controlled error |
| Result    | Process action-result execution       | Add a response header     |

Senior developer interview answer:

“I use resource filters when I need to short-circuit or work around the MVC resource-processing stage, exception filters for MVC-specific exception handling, and result filters for concerns around action-result execution. For application-wide exception handling, I generally prefer middleware because it covers a broader part of the HTTP pipeline.”

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
In ASP.NET Core Web API, global exception handling means handling unexpected errors from one central place instead of writing `try-catch` in every controller or service.

Think of it as a central security desk: whenever something goes wrong, the error is routed through one place that logs it and returns a consistent response.

## 1. Without global exception handling ❌

You might repeat this code in every API:

```
[HttpGet]
public IActionResult GetEmployees()
{
    try
    {
        throw new Exception("Database failed");
    }
    catch (Exception ex)
    {
        return StatusCode(500, ex.Message);
    }
}
```

Problems:

- Repeated code in multiple endpoints.
- Different APIs might return inconsistent errors.
- Internal exception details could leak to clients.

## 2. With global exception handling ✅

ASP.NET Core provides exception-handling middleware to centralize error handling.

### Step 1: Configure it in `Program.cs`

```
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        context.Response.StatusCode = 500;
        context.Response.ContentType = "application/json";

        await context.Response.WriteAsJsonAsync(new
        {
            message = "An unexpected error occurred."
        });
    });
});

app.MapControllers();

app.Run();
```

### Step 2: Write a normal controller

```
[ApiController]
[Route("api/[controller]")]
public class EmployeesController : ControllerBase
{
    [HttpGet]
    public IActionResult GetEmployees()
    {
        throw new Exception("Database connection failed!");
    }
}
```

### Step 3: Call the API

Request:

```
GET /api/employees
```

Response:

```
{
  "message": "An unexpected error occurred."
}
```

The middleware catches the unhandled exception and returns a consistent response. The controller doesn't need its own `try-catch`.

Note: This simple example demonstrates the concept. In production, also log the exception and return a trace or correlation ID without exposing internal details.

## 3. Production-style approach in modern .NET

You can use `IExceptionHandler` to separate exception-handling logic from `Program.cs`.

```
using Microsoft.AspNetCore.Diagnostics;

public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(
        ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(
            exception,
            "An unhandled exception occurred.");

        context.Response.StatusCode =
            StatusCodes.Status500InternalServerError;

        await context.Response.WriteAsJsonAsync(
            new { message = "An unexpected error occurred." },
            cancellationToken);

        return true;
    }
}
```

Register it in `Program.cs`:

```
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

var app = builder.Build();

app.UseExceptionHandler();

app.MapControllers();

app.Run();
```

This uses the built-in exception-handling infrastructure and makes the handler reusable and testable.

## 4. How does it work?

Client sends API request

Exception-handling middleware

Controller / Service / Database

Exception occurs → Middleware handles it

Log error + Return safe HTTP response

## 5. Global exception handling vs exception filter

| Feature                                                  | Middleware / `IExceptionHandler`     | Exception filter                      |
| -------------------------------------------------------- | ------------------------------------ | ------------------------------------- |
| Scope                                                    | Broad HTTP pipeline                  | MVC pipeline                          |
| Handles service exceptions bubbling through the pipeline | Yes, if unhandled                    | Only within its applicable MVC stages |
| Handles routing or middleware exceptions                 | Can, depending on pipeline placement | No                                    |
| Recommended for application-wide error handling          | Yes                                  | Not usually                           |

## 6. Senior/Staff Engineer interview answer

“Global exception handling centralizes the handling of unhandled exceptions in ASP.NET Core. I configure exception-handling middleware, often with `IExceptionHandler`, to log errors, return consistent HTTP responses, and avoid exposing internal details. I also use structured logging and correlation IDs for troubleshooting, while mapping known business exceptions to appropriate status codes such as 400, 404, or 409 rather than treating every error as a 500.”

Remember: Global exception handling = one central place to catch unhandled errors, log them, and return consistent responses.

The key idea is: Use an exception filter when your error-handling logic needs information about the MVC controller or action that failed.

For example, you might want to log which controller action failed, or return a controller-specific error response. For general application-wide exception handling, middleware is usually a better choice.

## Example: Return a controller-specific error response

Imagine your Employee API needs to return a consistent error message containing the controller and action name whenever an unhandled exception occurs.

### 1. Create an exception filter

```
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

public class ApiExceptionFilter : IExceptionFilter
{
    private readonly ILogger<ApiExceptionFilter> _logger;

    public ApiExceptionFilter(ILogger<ApiExceptionFilter> logger)
    {
        _logger = logger;
    }

    public void OnException(ExceptionContext context)
    {
        // MVC-specific information
        var controller = context.RouteData.Values["controller"]?.ToString();
        var action = context.RouteData.Values["action"]?.ToString();

        _logger.LogError(
            context.Exception,
            "Error in {Controller}.{Action}",
            controller,
            action);

        context.Result = new ObjectResult(new
        {
            message = "Employee operation failed.",
            controller,
            action
        })
        {
            StatusCode = StatusCodes.Status500InternalServerError
        };

        context.ExceptionHandled = true;
    }
}
```

### 2. Apply it only to the Employees controller

```
[ApiController]
[Route("api/[controller]")]
[ServiceFilter(typeof(ApiExceptionFilter))]
public class EmployeesController : ControllerBase
{
    [HttpGet("{id:int}")]
    public IActionResult GetEmployee(int id)
    {
        throw new InvalidOperationException("Employee lookup failed");
    }
}
```

Register the filter:

```
builder.Services.AddScoped<ApiExceptionFilter>();
builder.Services.AddControllers();
```

### 3. Call the API

Request:

```
GET /api/employees/10
```

Response:

```
{
  "message": "Employee operation failed.",
  "controller": "Employees",
  "action": "GetEmployee"
}
```

The filter can access MVC-specific information through `ExceptionContext`, such as the route data, action descriptor, and exception.

## Why use a filter here instead of middleware?

| Requirement                                              | Better fit           |
| -------------------------------------------------------- | -------------------- |
| Handle an exception using controller/action metadata     | Exception filter     |
| Apply custom handling only to selected controllers       | Exception filter     |
| Handle unhandled exceptions throughout the HTTP pipeline | Exception middleware |
| Log all incoming HTTP requests                           | Middleware           |

Staff Engineer tip: Don't expose controller names or internal exception details in production responses unless there is a clear API contract requiring them. In a real application, log the context internally and return a safe error response with a correlation ID.

Interview answer: “I use exception filters when error handling depends on MVC-specific context, such as controller/action metadata or controller-specific response behavior. For centralized application-wide exception handling, I prefer exception-handling middleware.”

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

## Important REST concepts:

* resources
* statelessness
* HTTP semantics
* appropriate status codes
* cacheability where applicable

### Depth understanding:

  REST (Representational State Transfer) is an architectural style for designing web APIs that allow clients and servers to communicate over HTTP using resources and standard HTTP methods.

In simple English: REST means designing APIs around resources, such as employees or orders, and using HTTP methods to work with them.

## 1. Real-life example

Imagine you are building an Employee Management application.

Your Angular frontend needs to view, create, update, and delete employees. Your ASP.NET Core Web API exposes REST-style endpoints for these operations.

## 2. REST API example

Suppose your API base URL is `https://api.example.com`.

| HTTP method | Endpoint             | Purpose                       |
| ----------- | -------------------- | ----------------------------- |
| `GET`       | `/api/employees`     | Get all employees             |
| `GET`       | `/api/employees/101` | Get employee 101              |
| `POST`      | `/api/employees`     | Create an employee            |
| `PUT`       | `/api/employees/101` | Replace/update employee 101   |
| `PATCH`     | `/api/employees/101` | Partially update employee 101 |
| `DELETE`    | `/api/employees/101` | Delete employee 101           |

Notice that the URL represents the resource (`employees`), while the HTTP method specifies the action.

## 3. Simple ASP.NET Core example

```
[ApiController]
[Route("api/[controller]")]
public class EmployeesController : ControllerBase
{
    [HttpGet]
    public IActionResult GetEmployees()
    {
        return Ok(new[]
        {
            new { Id = 101, Name = "Rahul" },
            new { Id = 102, Name = "Priya" }
        });
    }

    [HttpGet("{id:int}")]
    public IActionResult GetEmployee(int id)
    {
        return Ok(new { Id = id, Name = "Rahul" });
    }

    [HttpPost]
    public IActionResult CreateEmployee(Employee employee)
    {
        return CreatedAtAction(
            nameof(GetEmployee),
            new { id = employee.Id },
            employee);
    }
}
```

For example, calling `GET /api/employees/101` returns employee information as JSON:

```
{
  "id": 101,
  "name": "Rahul"
}
```

The example illustrates the endpoint structure; a production implementation would retrieve and persist employee data through a service and database.

## 4. Important REST principles

- Client-server: Frontend and backend have separate responsibilities.
- Stateless: Each request contains the information needed to process it; the server does not rely on stored client session state between requests.
- Resource-based URLs: Use nouns such as `/employees` and `/orders`.
- Uniform interface: Use standard HTTP methods and status codes.
- Cacheability: Responses can indicate whether they may be cached.
- Layered system: Clients can communicate through intermediaries such as gateways or proxies.

## 5. Common HTTP status codes

| Status code                 | Meaning                                   |
| --------------------------- | ----------------------------------------- |
| `200 OK`                    | Request succeeded                         |
| `201 Created`               | Resource created                          |
| `204 No Content`            | Request succeeded without a response body |
| `400 Bad Request`           | Invalid request                           |
| `401 Unauthorized`          | Authentication is missing or invalid      |
| `403 Forbidden`             | Caller lacks permission                   |
| `404 Not Found`             | Resource not found                        |
| `500 Internal Server Error` | Unexpected server error                   |

## 6. Senior-level interview answer

“REST is an architectural style for designing network APIs around resources. In ASP.NET Core, I use resource-oriented URLs, standard HTTP methods, appropriate status codes, stateless request handling, and consistent JSON representations. For example, `GET /api/employees/101` retrieves an employee, while `POST /api/employees` creates one. I also consider pagination, validation, authentication, caching, and API versioning when designing production REST APIs.”

Memory trick: REST = Resources + HTTP methods + Stateless communication.

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



# Cache Stampede in .NET — Problem and Solutions

A cache stampede occurs when a cached item expires or is missing, and many requests try to load the same data from the database simultaneously.

It can overload your database and slow down the API.

## 1. How does a cache stampede happen?

Suppose your ASP.NET Core API caches employee data for 5 minutes.

Cache expires

Employee data is no longer available in cache

100 requests arrive simultaneously

Request 1

Queries DB

Request 2

Queries DB

Requests 3–100

Query DB

Potentially 100 database queries for the same data!

Important: `IMemoryCache.GetOrCreateAsync()` does not guarantee that only one request executes the cache factory during a cache miss.

## 2. Techniques to solve cache stampede

| Technique                    | How it works                                                                   | Best use case                                   |
| ---------------------------- | ------------------------------------------------------------------------------ | ----------------------------------------------- |
| 1. Locking / Single-flight   | Only one request loads missing data; others wait                               | `IMemoryCache` in a single application instance |
| 2. HybridCache               | Coordinates concurrent cache misses and supports local and distributed caching | Modern .NET applications                        |
| 3. Distributed cache (Redis) | Shares cached data across multiple application instances                       | Load-balanced applications                      |
| 4. Stale-while-revalidate    | Serves older data while refreshing it in the background                        | Data that can tolerate brief staleness          |
| 5. Background refresh        | Refreshes cache before it expires                                              | Frequently accessed data                        |
| 6. TTL jitter                | Adds random variation to expiration times                                      | Many cache keys expiring together               |

Redis alone does not prevent stampedes: multiple instances can still miss the same key and query the database concurrently. Combine it with coordination or a single-flight mechanism when needed.

## 3. Solution 1: Use `SemaphoreSlim` with `IMemoryCache`

This is a practical interview example for a single application instance.

```
public class EmployeeService
{
    private readonly IMemoryCache _cache;
    private readonly AppDbContext _db;

    private static readonly SemaphoreSlim _lock = new(1, 1);

    public EmployeeService(
        IMemoryCache cache,
        AppDbContext db)
    {
        _cache = cache;
        _db = db;
    }

    public async Task<List<Employee>> GetEmployeesAsync()
    {
        const string key = "employees";

        // First cache check
        if (_cache.TryGetValue(key, out List<Employee>? employees))
            return employees!;

        await _lock.WaitAsync();

        try
        {
            // Check again after acquiring the lock
            if (_cache.TryGetValue(key, out employees))
                return employees!;

            // Only one request reaches the database here
            employees = await _db.Employees
                .AsNoTracking()
                .ToListAsync();

            _cache.Set(
                key,
                employees,
                TimeSpan.FromMinutes(5));

            return employees;
        }
        finally
        {
            _lock.Release();
        }
    }
}
```

### How it works

1. Request A finds a cache miss and acquires the lock.
2. Requests B, C, and D wait for the lock.
3. Request A queries the database and stores the result in cache.
4. Request A releases the lock.
5. The waiting requests acquire the lock one at a time, recheck the cache, and return the cached result.

Why do we check the cache twice? The second check prevents a waiting request from querying the database after the first request has already populated the cache.

Limitations: This simple example uses one global lock, so unrelated cache keys also block one another. A production implementation can use a per-key lock or a single-flight mechanism. The `AppDbContext` should remain scoped and must not be used concurrently by multiple requests.

## 4. Solution 2: Use HybridCache (modern .NET)

For modern .NET applications, `HybridCache` is often a simpler option. It combines an in-process cache with an optional distributed cache and provides stampede protection for concurrent calls to the same key.

Install the package if your target framework does not already provide it:

```
dotnet add package Microsoft.Extensions.Caching.Hybrid
```

Register it in `Program.cs`:

```
builder.Services.AddHybridCache();
```

Use it in your service:

```
public class EmployeeService
{
    private readonly HybridCache _cache;
    private readonly AppDbContext _db;

    public EmployeeService(
        HybridCache cache,
        AppDbContext db)
    {
        _cache = cache;
        _db = db;
    }

    public async Task<List<Employee>> GetEmployeesAsync(
        CancellationToken cancellationToken = default)
    {
        return await _cache.GetOrCreateAsync(
            "employees",
            async token =>
                await _db.Employees
                    .AsNoTracking()
                    .ToListAsync(token),
            cancellationToken: cancellationToken);
    }
}
```

Configure a distributed cache such as Redis when you need shared cache storage across application instances. HybridCache's stampede protection coordinates concurrent calls within the applicable cache service instance; it is not a universal distributed lock across every server.

## 5. Which technique should you choose?

Single application instance

Use `IMemoryCache` with per-key locking, or `HybridCache`.

Multiple instances behind a load balancer

Use Redis or another distributed cache with stampede protection appropriate to your deployment.

High-traffic, mostly read-only data

Use background refresh or stale-while-revalidate to avoid a cold-cache spike.

## Interview-ready answer

“A cache stampede occurs when a popular cache entry expires and multiple concurrent requests attempt to regenerate the same value, causing excessive database load. I can prevent it using a per-key lock or single-flight mechanism, use `HybridCache` for built-in stampede protection, and use Redis for shared caching across application instances. For frequently accessed data, I can also refresh the cache proactively or serve stale data while refreshing it. The right approach depends on whether the application runs on one instance or multiple instances and whether stale data is acceptable.”
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
