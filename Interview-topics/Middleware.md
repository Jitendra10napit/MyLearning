# Middleware in ASP.NET Core

## 1. What it is

Middleware is a component that sits in the HTTP request pipeline. Each one gets the `HttpContext` and makes a choice:

1. Do some work, then pass the request on by calling `next()`.
2. Do some more work after `next()` returns, on the way back out.
3. Or stop the request there (short-circuit) and write the response itself.

The pipeline nests like a Russian doll, so the request goes in through each middleware in order and the response comes back out in reverse order:

```
Request ──► [ExceptionHandler] ──► [HTTPS] ──► [Routing] ──► [AuthN] ──► [AuthZ] ──► [Endpoint]
                                                                                      │
Response ◄── [ExceptionHandler] ◄── [HTTPS] ◄── [Routing] ◄── [AuthN] ◄── [AuthZ] ◄──┘
```

Underneath, each middleware is a `RequestDelegate` (`Task(HttpContext)`) wrapped around the next one. At startup, `app.Build()` combines all of them into a single delegate, and every request runs through that delegate.

## 2. The building blocks

```csharp
// Use: does work, calls next, can do more work after it
app.Use(async (context, next) =>
{
    var sw = Stopwatch.StartNew();
    await next(context);                 // everything later in the pipeline runs here
    sw.Stop();
    context.Response.Headers["X-Elapsed-Ms"] = sw.ElapsedMilliseconds.ToString(); // careful, see pitfall below
});

// Run: terminal, never calls next
app.Run(async ctx => await ctx.Response.WriteAsync("End of pipeline"));

// Map: branches the pipeline by path
app.Map("/health-raw", branch => branch.Run(c => c.Response.WriteAsync("OK")));

// MapWhen: branches on a condition and does NOT rejoin the main pipeline
app.MapWhen(c => c.Request.Headers.ContainsKey("X-Legacy"), b => b.UseLegacyHandler());

// UseWhen: branches on a condition and DOES rejoin the main pipeline
app.UseWhen(c => c.Request.Path.StartsWithSegments("/api"), b => b.UseMiddleware<ApiKeyMiddleware>());
```

Interviewers often ask the difference between `MapWhen` and `UseWhen`: `MapWhen` forks the pipeline permanently, while `UseWhen` adds steps for some requests and then continues with the main pipeline.

## 3. Order matters

This is the recommended order for a Web API on .NET 8+:

```csharp
app.UseExceptionHandler();      // 1. outermost, so it catches exceptions from everything below
app.UseHsts();
app.UseHttpsRedirection();
app.UseStaticFiles();           // short-circuits for static files, so they skip auth (cheap)
app.UseRouting();               // selects the endpoint (.NET 6+ adds this implicitly)
app.UseCors();                  // must come after Routing and before AuthN/AuthZ
app.UseAuthentication();        // who are you? (builds HttpContext.User)
app.UseAuthorization();         // are you allowed? (reads the selected endpoint's metadata)
app.UseRateLimiter();
app.UseResponseCaching();
app.MapControllers();           // endpoints
```

Common bugs caused by wrong order:
- `UseAuthorization` placed before `UseAuthentication`: the user is always anonymous, so you get 401s or 403s.
- `UseCors` placed after `UseAuthorization`: preflight `OPTIONS` requests fail.
- The exception handler registered late: exceptions thrown by earlier middleware are never caught.
- `UseAuthorization` placed before `UseRouting`: there's no endpoint metadata yet, so `[Authorize]` attributes are ignored.

## 4. Writing custom middleware: two styles

### a) Convention-based (most common)

```csharp
public sealed class CorrelationIdMiddleware
{
    private const string Header = "X-Correlation-ID";
    private readonly RequestDelegate _next;
    private readonly ILogger<CorrelationIdMiddleware> _logger;

    // The constructor runs ONCE, so this instance is effectively a SINGLETON.
    public CorrelationIdMiddleware(RequestDelegate next, ILogger<CorrelationIdMiddleware> logger)
        => (_next, _logger) = (next, logger);

    // Scoped services go here, as InvokeAsync parameters (resolved per request)
    public async Task InvokeAsync(HttpContext context, ITenantContext tenant)
    {
        var id = context.Request.Headers.TryGetValue(Header, out var v) && !string.IsNullOrWhiteSpace(v)
            ? v.ToString()
            : Guid.NewGuid().ToString();

        context.TraceIdentifier = id;
        context.Response.OnStarting(() =>
        {
            context.Response.Headers[Header] = id;   // safe: runs just before headers are sent
            return Task.CompletedTask;
        });

        using (_logger.BeginScope(new Dictionary<string, object> { ["CorrelationId"] = id }))
        {
            await _next(context);
        }
    }
}

// Extension method for a clean registration
public static class CorrelationIdExtensions
{
    public static IApplicationBuilder UseCorrelationId(this IApplicationBuilder app)
        => app.UseMiddleware<CorrelationIdMiddleware>();
}
```

### b) Factory-based (`IMiddleware`)

```csharp
public sealed class TenantResolutionMiddleware : IMiddleware
{
    private readonly AppDbContext _db;   // scoped is fine here, because DI creates this per request
    public TenantResolutionMiddleware(AppDbContext db) => _db = db;

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        // resolve tenant...
        await next(context);
    }
}

builder.Services.AddScoped<TenantResolutionMiddleware>();   // must be registered in DI
app.UseMiddleware<TenantResolutionMiddleware>();
```

| | Convention-based | `IMiddleware` |
|---|---|---|
| Lifetime | Singleton (created once) | Whatever lifetime you register it with (per request if scoped) |
| Scoped dependencies | Go in `InvokeAsync` parameters | Go in the constructor |
| DI registration | Not needed | Required |
| Type safety | Duck-typed by convention | Interface contract |
| Performance | Slightly faster | Allocates per request |

**The classic senior-level bug:** injecting a scoped service such as a `DbContext` into the constructor of convention-based middleware. That captures one instance for the whole application. It surfaces as a captive dependency, a thread-safety failure, or a stale DbContext. With scope validation on (the default in Development), you get an `InvalidOperationException` at startup.

## 5. Pitfalls I check for in code reviews

1. **Changing headers after the response has started.** Once the body starts streaming, headers are locked and you get "Headers are read-only, response has already started." Use `Response.OnStarting(...)` or check `Response.HasStarted`.
2. **Reading the request body twice.** The body is a forward-only stream. Call `Request.EnableBuffering()`, read it, then reset `Position = 0`. Be careful with large payloads, because buffering puts them in memory or on disk.
3. **Blocking calls** (`.Result`, `.Wait()`) inside middleware. Every request passes through it, so a blocking call can starve the thread pool across the whole service.
4. **Heavy work on every request.** The pipeline is the hot path. Push work into branches (`UseWhen`) or into endpoint filters when only some routes need it.
5. **Forgetting `await next()`.** The request silently stops there and returns an empty 200.
6. **Logging full request and response bodies.** This creates PII and compliance problems and costs performance. Redact sensitive fields, sample, or limit logging by size.

## 6. Middleware vs filters vs `IExceptionHandler`

| Concern | Use |
|---|---|
| Applies to every request, needs no MVC context (correlation ID, security headers, request logging, tenant resolution) | **Middleware** |
| Needs model binding, action arguments, `ModelState`, or controller/action metadata | **MVC action/result filters** |
| Minimal API per-endpoint logic (validation) | **Endpoint filters** (`AddEndpointFilter`) |
| Global exception → ProblemDetails mapping | **`IExceptionHandler`** (.NET 8+) with `UseExceptionHandler()` |

Rule of thumb: middleware knows about HTTP; filters know about MVC actions. When something needs `ActionExecutingContext`, it's a filter.

On .NET 8+, handle exceptions with `IExceptionHandler` rather than a hand-written try/catch middleware:

```csharp
public sealed class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> log) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        log.LogError(ex, "Unhandled exception");
        var (status, title) = ex switch
        {
            ValidationException  => (400, "Validation failed"),
            NotFoundException    => (404, "Resource not found"),
            _                    => (500, "Server error")
        };
        ctx.Response.StatusCode = status;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = status, Title = title, Instance = ctx.Request.Path,
            Extensions = { ["traceId"] = ctx.TraceIdentifier }
        }, ct);
        return true;
    }
}

builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();
app.UseExceptionHandler();
```

## 7. The architect's view: where cross-cutting concerns belong

In a microservices system, the key decision is which layer handles each concern:

| Concern | Gateway (APIM / YARP / Ocelot) | Service middleware | Shared NuGet package |
|---|---|---|---|
| TLS termination, WAF, global rate limits | ✅ | | |
| Coarse token validation | ✅ | | |
| **Fine-grained authZ** (tenant, case ownership) | | ✅ (never trust the gateway alone, zero-trust) | |
| Correlation ID / OpenTelemetry trace propagation | starts it | ✅ continues it | ✅ |
| Exception → ProblemDetails contract | | ✅ | ✅ |
| Tenant resolution, request audit | | ✅ | ✅ |

Principles I'd defend in a design review:
- **Package your platform middleware.** Put correlation ID, exception handling, security headers and tenant resolution in one internal NuGet package with a `UsePlatformDefaults()` extension. Every service then gets the same pipeline in the same order, which prevents configuration drift across many services.
- **Treat the order as a contract.** Document it and test it. A misplaced `UseCors` is a production incident.
- **Defense in depth.** The gateway reduces load and blocks obvious abuse. The service still enforces business authorization itself.
- **Observability first.** Correlation and tracing middleware go near the outermost position, so every log line, including ones from failures, carries the trace ID across HTTP, Service Bus, and Durable Functions activities.
- **Know when middleware doesn't apply.** Azure Functions (isolated worker) has its own pipeline through `IFunctionsWorkerMiddleware`, and background workers like `BackgroundService` have no HTTP pipeline at all. Cross-cutting concerns there need decorators or Function middleware instead.

## 8. Testing

```csharp
[Fact]
public async Task Adds_correlation_id_when_missing()
{
    var ctx = new DefaultHttpContext();
    var mw = new CorrelationIdMiddleware(_ => Task.CompletedTask, NullLogger<CorrelationIdMiddleware>.Instance);

    await mw.InvokeAsync(ctx, Mock.Of<ITenantContext>());

    Assert.False(string.IsNullOrEmpty(ctx.TraceIdentifier));
}
```

Use unit tests with `DefaultHttpContext` for logic. Use `WebApplicationFactory<Program>` integration tests to verify ordering behavior, for example that CORS preflight and auth work end to end.

## 9. Quick interview answers

- **Middleware vs HttpModule/HttpHandler?** Middleware replaced both. It's composable, configured in code, and not tied to IIS events.
- **How do you short-circuit?** Don't call `next()`. Write the response and return. Static files and auth challenges do this.
- **Can middleware be async and still do work after the endpoint?** Yes. Code after `await next()` runs while the response travels back out.
- **Singleton or scoped?** Convention-based middleware is a singleton. `IMiddleware` uses whatever lifetime you registered.
- **Where does routing happen?** `UseRouting` selects the endpoint. `UseEndpoints` or `Map*` runs it. Middleware placed between the two can read `context.GetEndpoint()` metadata, which is how authorization reads `[Authorize]`.

**One-line summary:** middleware is an ordered chain of delegates that handles concerns common to every request, such as logging, errors, and security. A senior dev gets the order, lifetimes and response-started rules right. An architect decides which concerns belong at the gateway, which in service middleware, and which in a shared package.

If it helps your prep, I can do a mock interview round on this topic next, or cover filters in the same depth.
