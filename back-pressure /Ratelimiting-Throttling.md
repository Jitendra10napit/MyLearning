# Rate Limiting vs. Throttling — Senior Engineer / Architect Guide

Imagine you are designing an enterprise Order Management API that receives thousands of requests per second. Your database can handle only 500 requests per second, but suddenly 5,000 requests arrive.

If you allow every request to reach the database, the application may become slow, consume all database connections, or fail completely.

As a Senior Engineer or Solution Architect, you need to control incoming traffic, protect downstream services, and maintain system availability.

## 1. What is rate limiting and throttling?

## Rate limiting

Controls how many requests a client can make within a defined time window.

Example: A user can call the API 100 times per minute. After that, additional requests are rejected until the limit allows them again.

Primary goal: Fair usage, abuse prevention, and capacity protection.

## Throttling

Controls or slows down traffic when a service is approaching or exceeding its processing capacity.

Example: A payment provider can process only 500 requests per second. Excess requests are delayed, queued, or rejected with a retry instruction.

Primary goal: Protect service stability and manage overload.

Important distinction: In real systems, these terms overlap. Rate limiting is often one mechanism used to implement throttling. Some platforms use “throttling” to mean any request rejected because a quota or capacity limit was exceeded.

## 2. Understand the difference with a simple example

Suppose an API has a limit of 100 requests per minute per user.

Allowed quota

# 100/min

Requests accepted within the limit

Incoming requests

# 150/min

50 requests exceed the quota

Rate limiter

Counts requests for each user or API key

100 accepted

Requests proceed to the API.

50 rejected

Return HTTP 429 Too Many Requests.

Now consider a different scenario: your API allows 100 requests per minute, but all 100 arrive in one second. The quota might still be valid, yet the sudden burst can overwhelm a database.

You can use a concurrency limit, token bucket, queue, or adaptive throttling to protect the service from bursts and excessive simultaneous work.

## 3. Different ways to handle rate limiting and throttling

There is no single solution for every scenario. As an architect, select the mechanism based on what you need to protect.

1\. Fixed Window

Allow 100 requests per minute. The counter resets at the beginning of each window.

Best for: Simple API quotas. Limitation: Traffic can spike around window boundaries.

2\. Sliding Window

Track requests over a moving time period to smooth out boundary spikes.

Best for: More consistent per-user quotas over time.

3\. Token Bucket

Requests consume tokens. Tokens replenish over time, allowing controlled bursts.

Best for: APIs that need to accommodate occasional bursts without unlimited traffic.

4\. Concurrency Limiting

Allow only a certain number of requests to execute simultaneously.

Best for: Expensive database queries, file processing, and resource-intensive APIs.

5\. Queue-based throttling

Put work into a queue and process it at a controlled rate using background workers.

Best for: Order processing, notifications, reports, and video processing.

### How to choose the right approach

| Scenario                                                     | Recommended approach                          |
| ------------------------------------------------------------ | --------------------------------------------- |
| Limit each user to 100 API calls/minute                      | Fixed or sliding window                       |
| Allow occasional bursts of traffic                           | Token bucket                                  |
| Protect SQL Server from too many concurrent queries          | Concurrency limiting                          |
| Process 10,000 orders without overloading a payment provider | Queue + controlled workers                    |
| Enforce a shared quota across multiple API instances         | API gateway or distributed rate limiter       |
| Handle an overloaded or failing downstream service           | Backpressure, bounded queues, circuit breaker |

## 4. Implement rate limiting in ASP.NET Core Web API

ASP.NET Core provides built-in rate-limiting middleware. You don't need to write your own counter for a basic implementation.

### Step 1 — Configure a fixed-window limiter

In `Program.cs`:

```

using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    options.AddFixedWindowLimiter("UserPolicy", limiter =>
    {
        limiter.PermitLimit = 100;
        limiter.Window = TimeSpan.FromMinutes(1);

        limiter.QueueLimit = 0;
        limiter.AutoReplenishment = true;
    });
});

builder.Services.AddControllers();

var app = builder.Build();

app.UseRouting();
app.UseRateLimiter();

app.MapControllers();

app.Run();

```

### Step 2 — Apply the policy to an API endpoint

```
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.RateLimiting;

[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet]
    [EnableRateLimiting("UserPolicy")]
    public IActionResult GetOrders()
    {
        return Ok(new[]
        {
            new { Id = 1, Status = "Processing" },
            new { Id = 2, Status = "Shipped" }
        });
    }
}
```

The example returns sample orders, but the rate-limiting behavior is real when the application is configured and running.

### Step 3 — What happens at runtime?

Client

Sends requests to GET /api/orders

Rate-limiting middleware

Checks whether a permit is available in the current window.

Permit available

Request reaches the controller.

200 OK

Quota exceeded

Request is rejected by middleware.

429 Too Many Requests

Because `QueueLimit = 0`, excess requests are rejected immediately rather than waiting in a queue.

Important: This basic policy limits traffic per application instance, not necessarily per authenticated user. To implement a true per-user quota, use a partitioned policy. To enforce one shared quota across several application instances, use a gateway or a distributed coordination mechanism.

## 5. Rate limiting by user, IP address, or API key

In real enterprise applications, you usually want different limits for different clients.

For example:

- Free user: 50 requests per minute.
- Premium user: 500 requests per minute.
- Internal service: 1,000 requests per minute.

Use a partitioned rate limiter in ASP.NET Core.

```

using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = 429;

    options.AddPolicy("PerClient", context =>
    {
        // In production, prefer a verified user ID or API key.
        var clientId = context.User.Identity?.IsAuthenticated == true
            ? context.User.FindFirst("sub")?.Value
            : null;

        clientId ??= context.Connection.RemoteIpAddress?.ToString()
                     ?? "unknown";

        return RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: clientId,
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1),
                QueueLimit = 0,
                AutoReplenishment = true
            });
    });
});

app.UseRateLimiter();

app.MapGet("/api/orders", () => Results.Ok("Orders returned"))
   .RequireRateLimiting("PerClient");

```

How it works: Each partition has its own counter. User A making 100 requests doesn't consume User B's quota.

For production:

- Prefer a stable, authenticated user ID or API key when that is the quota identity.
- Configure trusted proxy forwarding correctly before relying on client IP addresses.
- Decide how anonymous users and missing identifiers should be handled.
- Use shared coordination if the quota must apply across multiple application instances.

## 6. Implement token bucket for burst traffic

Suppose your API should allow up to 100 requests in a burst, then replenish 20 permits every second.

A token bucket is a good fit because it allows bursts while controlling sustained traffic.

```

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = 429;

    options.AddTokenBucketLimiter("BurstPolicy", limiter =>
    {
        limiter.TokenLimit = 100;
        limiter.TokensPerPeriod = 20;
        limiter.ReplenishmentPeriod = TimeSpan.FromSeconds(1);

        limiter.AutoReplenishment = true;
        limiter.QueueLimit = 0;
    });
});

app.MapGet("/api/reports", () => Results.Ok("Report ready"))
   .RequireRateLimiting("BurstPolicy");

```

Conceptually:

### Token bucket example

Illustration: 6 of 10 available tokens are consumed. The actual example starts with capacity for 100 tokens.

- Each accepted request consumes a token.
- When tokens run out, requests are rejected or queued if configured.
- New tokens become available as replenishment occurs.

Token bucket controls the permitted burst and average replenishment rate; it does not guarantee that the downstream service can safely process that many concurrent requests.

## 7. Concurrency limiting: protect the database

Consider a report-generation API. Each request executes a costly SQL query and takes 5 seconds.

If 500 requests execute concurrently, the database may become overloaded even if the total request rate is reasonable.

Set a concurrency limit:

```

builder.Services.AddRateLimiter(options =>
{
    options.AddConcurrencyLimiter("ReportPolicy", limiter =>
    {
        limiter.PermitLimit = 20;
        limiter.QueueLimit = 50;
        limiter.QueueProcessingOrder =
            QueueProcessingOrder.OldestFirst;
    });
});

app.MapGet("/api/reports/large",
    async (CancellationToken cancellationToken) =>
    {
        // Replace with actual report-generation logic.
        await Task.Delay(1000, cancellationToken);

        return Results.Ok("Report generated");
    })
    .RequireRateLimiting("ReportPolicy");

```

With this policy, at most 20 requests execute concurrently under this limiter, and up to 50 wait for a permit. Additional requests are rejected when both capacity and queue space are exhausted.

The queue is in memory and bounded. For long-running reports, a durable background job or message queue is generally more appropriate.

## 8. Handle overload using Azure Service Bus

This is where throttling becomes an architectural decision rather than just middleware configuration.

Imagine an order API receiving 10,000 orders in a short burst, while a payment provider can safely process only 200 per second.

Instead of performing every payment synchronously inside the incoming API request, accept valid orders and publish work to a queue.

Clients

Submit orders

ASP.NET Core Order API

Validate request, persist order, publish work

Azure Service Bus Queue

Buffer work durably and absorb traffic spikes

Payment Worker

Controlled concurrency and downstream request rate

Payment Provider

Processes payments within its permitted capacity

### How to control the worker

For a .NET Azure Service Bus processor, you can configure:

```
var options = new ServiceBusProcessorOptions
{
    MaxConcurrentCalls = 10,
    AutoCompleteMessages = false
};

await using var processor =
    client.CreateProcessor("orders", options);
```

`MaxConcurrentCalls = 10` limits the number of messages this processor handles concurrently. It does not enforce a strict requests-per-second limit on the payment provider.

If the provider allows 200 requests per second, implement a rate limiter in the worker as well. For multiple worker replicas, coordinate the limit globally when the provider's quota is shared.

Production considerations include:

- Use idempotency to prevent duplicate payment effects during retries.
- Complete messages only after successful processing.
- Retry transient failures with bounded exponential backoff and jitter.
- Use dead-letter queues for messages that cannot be processed.
- Monitor queue depth and message age to detect growing backlogs.
- Persist the order and publish its event reliably, for example using the transactional outbox pattern.

The API can return `202 Accepted` when it has durably accepted the request for asynchronous processing. This means the order was accepted for processing, not that the payment has completed.

## 9. Use Azure API Management for centralized rate limiting

In an enterprise architecture, you may have multiple APIs and several application instances. Applying a separate in-memory limiter inside each API can produce inconsistent overall limits.

Azure API Management (APIM) can enforce quotas and rate limits at the gateway.

Mobile app / Web app / Partner

Azure API Management

Authentication, subscription quotas, rate-limit policies

Order API

Payment API

Inventory API

Example APIM policy:

```
<policies>
  <inbound>
    <base />
    <rate-limit-by-key
        calls="100"
        renewal-period="60"
        counter-key="@(context.Subscription?.Key ?? context.Request.IpAddress)" />
  </inbound>
  <backend>
    <base />
  </backend>
  <outbound>
    <base />
  </outbound>
  <on-error>
    <base />
  </on-error>
</policies>
```

This example illustrates a limit of 100 calls per 60 seconds per subscription key, falling back to the request IP when there is no subscription key.

Important: APIM policy behavior and counter scope depend on the gateway, tier, and deployment topology. This policy should not be assumed to enforce a perfectly global quota across every gateway instance or region. For shared quotas, validate the relevant APIM capabilities and architecture before deployment.

Use APIM for gateway-level enforcement and application-level concurrency controls for expensive work. They solve different problems.

## 10. What should the API return when traffic exceeds limits?

| HTTP status               | Meaning                                            | Typical response                          |
| ------------------------- | -------------------------------------------------- | ----------------------------------------- |
| `429 Too Many Requests`   | Client exceeded its quota or rate limit            | Retry after the indicated delay           |
| `503 Service Unavailable` | Service is temporarily unable to handle the load   | Retry cautiously if the operation is safe |
| `202 Accepted`            | Work has been accepted for asynchronous processing | Poll status or receive an event later     |
| `200 OK`                  | Request succeeded                                  | Return the result                         |

For a `429` response, provide a `Retry-After` header where appropriate.

Example:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 10
Content-Type: application/json
```

```
{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Please retry later."
}
```

Client applications should respect `Retry-After`, use exponential backoff with jitter when appropriate, and avoid retrying indefinitely. For non-idempotent operations such as payments, use idempotency keys before retrying.

## 11. Production best practices for a Senior Engineer / Architect

Apply multiple layers of protection

Gateway rate limiting, API middleware, database concurrency limits, and worker throttling can work together.

Use the right limit identity

Partition by authenticated user, API key, tenant, or IP depending on the business requirement.

Avoid unbounded queues

Queues consume memory and increase latency. Set queue limits, timeouts, and overload behavior.

Measure and adapt

Monitor request rate, 429/503 responses, p95/p99 latency, database connections, queue depth, and downstream error rates.

Protect shared resources

Ten application instances with a local limit of 100 requests per second each could collectively permit approximately 1,000 requests per second. A local limit is not a global limit.

## 12. Interview-ready answer

You can explain it this way in a Senior Staff Engineer or Architect interview:

“Rate limiting controls the number of requests a client can make within a given time period, while throttling controls excessive traffic or processing to protect system capacity.

For example, in an order management system, I might allow 100 API requests per minute per user using ASP.NET Core rate-limiting middleware. When the quota is exceeded, I return HTTP 429 with a retry instruction.

For expensive operations, I would also implement concurrency limits to protect SQL Server. For asynchronous workflows such as payments and order processing, I would use Azure Service Bus to buffer work and control worker concurrency and downstream request rates.

At the gateway level, I would use Azure API Management for centralized traffic policies. In a horizontally scaled deployment, I would account for shared quotas, distributed coordination, retries, backpressure, monitoring, and idempotency.

My objective is not simply to reject traffic. It is to maintain fairness, protect critical dependencies, and keep the system reliable under peak load.”

Remember this simple rule:

- Rate limiting = How many requests are allowed?
- Concurrency limiting = How many requests can execute at the same time?
- Throttling = How do we control excess traffic or processing?
- Queueing = How do we absorb a burst and process work later?
- Backpressure = How do we signal upstream systems to slow down when downstream capacity is constrained?

These mechanisms complement one another; a production system often needs more than one.&#x20;
