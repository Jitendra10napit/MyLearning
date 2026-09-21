## Circuit Breaker Pattern — Architect Interview Explanation

The **Circuit Breaker** pattern prevents a failing downstream service from continuously receiving requests from an upstream service.

It is especially important in **microservices and distributed systems**.

### Why do we need it?

Suppose:

```text
Order API
   |
   ↓
Payment Service
```

Payment Service becomes slow/unavailable.

Without a circuit breaker:

```text
1000 requests
     ↓
Order API
     ↓
Payment Service ❌
     ↓
1000 failures/timeouts
```

This can cause:

* thread exhaustion
* connection exhaustion
* increased latency
* cascading failures
* eventually failure of the Order API itself

With a circuit breaker:

```text
Order API
   |
   ↓
Circuit Breaker
   |
   ↓
Payment Service ❌
```

After detecting repeated failures, the circuit opens:

```text
Order API
   |
   ↓
Circuit Breaker 🔴 OPEN
   |
   X
Payment Service
```

Requests are rejected/fail-fast instead of repeatedly calling the unhealthy service.

---

# 1. Three states

This is the most important interview concept.

### CLOSED

Normal operation.

```text
Request
   ↓
Circuit Breaker
   ↓
Payment Service
   ↓
Success
```

The breaker monitors failures.

Example:

```text
Failure threshold = 5
```

If failures cross the configured threshold, it transitions to **OPEN**.

---

### OPEN

The downstream service is considered unhealthy.

```text
Request
   ↓
Circuit Breaker
   ↓
     X
Payment Service
```

The request **doesn't reach the downstream service**.

Instead, you can:

* return an appropriate error
* execute fallback logic
* return cached data
* queue the operation
* ask the client to retry later

The circuit remains open for a configured duration.

Example:

```text
Break duration = 30 seconds
```

---

### HALF-OPEN

After the break duration, the circuit allows a limited test request.

```text
             30 sec
OPEN ─────────────────→ HALF-OPEN
                           |
                           ↓
                     Test request
```

If successful:

```text
HALF-OPEN
    ↓
Success
    ↓
CLOSED
```

If it fails:

```text
HALF-OPEN
    ↓
Failure
    ↓
OPEN
```

---

# 2. Complete state diagram

```text
                 failures exceed threshold
        ┌─────────────────────────────────────┐
        │                                     ↓
    ┌───────┐                            ┌────────┐
    │CLOSED │                            │  OPEN  │
    └───┬───┘                            └───┬────┘
        │                                    │
        │ success                           │ timeout
        │                                    │
        │                                    ↓
        │                              ┌────────────┐
        │                              │ HALF-OPEN  │
        │                              └─────┬──────┘
        │                                    │
        │                         ┌──────────┴──────────┐
        │                         │                     │
        │                     success                failure
        │                         │                     │
        └─────────────────────────┘                     │
                                                        ↓
                                                      OPEN
```

---

# 3. Circuit Breaker vs Retry

This is a **very common Architect interview question**.

### Retry

Retry says:

> "The failure might be temporary. Try again."

```text
Request
  ↓
Service
  ↓
Failure
  ↓
Retry
  ↓
Service
```

### Circuit Breaker

Circuit breaker says:

> "The service is failing repeatedly. Stop sending requests for now."

```text
Request
  ↓
Circuit Breaker
  ↓
OPEN
  ↓
Fail fast
```

### They are often used together

```text
Client
  ↓
Retry Policy
  ↓
Circuit Breaker
  ↓
Timeout
  ↓
Downstream Service
```

But **retry should be bounded**. Blindly retrying an unhealthy service can make an outage worse.

---

# 4. Circuit Breaker in .NET

For modern .NET applications, you can use **Polly / Microsoft resilience APIs** depending on your .NET version and application setup.

Conceptually:

```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3
    })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,
        MinimumThroughput = 10,
        SamplingDuration = TimeSpan.FromSeconds(30),
        BreakDuration = TimeSpan.FromSeconds(15)
    })
    .Build();
```

Then:

```csharp
await pipeline.ExecuteAsync(async cancellationToken =>
{
    return await paymentClient.ProcessPaymentAsync(
        request,
        cancellationToken);
});
```

The important point for an architect interview isn't memorizing the API. Explain **what you are protecting, what counts as failure, how long the circuit remains open, and what happens during fallback**.

---

# 5. Real enterprise example

Suppose your application has:

```text
Investigation API
       |
       ↓
Evidence Processing Service
       |
       ↓
Video Processing Service
       |
       ↓
FFmpeg
```

Imagine the video-processing service becomes unavailable.

Without circuit breaker:

```text
100 requests
     ↓
Investigation API
     ↓
Video Processing
     ↓
100 timeouts
```

With circuit breaker:

```text
Investigation API
       ↓
Circuit Breaker
       ↓
Video Processing
```

After repeated failures:

```text
Circuit = OPEN

Investigation API
       ↓
Circuit Breaker
       X
Video Processing
```

The API can then return something like:

```json
{
  "status": "PROCESSING_UNAVAILABLE",
  "message": "Video processing is temporarily unavailable."
}
```

Or, depending on the business requirement, place the request into a queue for later processing.

---

# 6. Circuit Breaker + Azure Service Bus

An architect-level answer should distinguish **synchronous calls** from **asynchronous messaging**.

For:

```text
Order Service
     |
     ↓
Payment Service
```

Circuit breaker is useful for the synchronous HTTP call.

For:

```text
Order Service
     |
     ↓
Azure Service Bus
     |
     ↓
Payment Worker
```

you would generally rely on messaging resilience mechanisms such as:

* retry
* dead-letter queue
* lock handling
* idempotency
* poison-message handling

You don't simply apply an HTTP circuit breaker to the queue itself.

---

# 7. What should count as a failure?

This is an important architect question.

Don't treat **every exception** as a circuit-breaker failure.

For example:

```text
400 Bad Request
```

usually means the caller sent invalid data. Retrying won't fix it.

Whereas:

```text
500 Internal Server Error
503 Service Unavailable
Timeout
Connection failure
```

may indicate downstream availability problems.

So your policy should distinguish:

```text
4xx business/client error
       ↓
Usually don't retry

5xx / timeout / connection failure
       ↓
Potential resilience failure
```

The exact policy depends on the service and business semantics.

---

# 8. Circuit Breaker + Timeout + Retry

A good production design might look like:

```text
             HTTP Request
                  |
                  ↓
              Timeout
                  |
                  ↓
               Retry
                  |
                  ↓
           Circuit Breaker
                  |
                  ↓
          Payment Service
```

Or, depending on the resilience library and desired semantics, the policies can be composed in a different order. In an interview, explain **why** you chose the order rather than treating one ordering as universally correct.

---

# 9. Architect-level considerations

When designing circuit breakers, discuss:

### Failure threshold

Example:

```text
50% failure ratio
```

### Minimum throughput

Don't open the circuit based on one failed request when traffic is tiny.

### Sampling window

Example:

```text
30 seconds
```

### Break duration

Example:

```text
15 seconds
```

### Timeout

Prevent requests from waiting indefinitely.

### Retry

Use limited retries with appropriate backoff.

### Fallback

Decide what the user receives when the circuit is open.

### Monitoring

Track:

```text
Circuit state
Failure rate
Latency
Timeouts
Retry count
Rejected requests
```

---

# 10. Interview answer — 60 seconds

If the interviewer asks:

> **"Explain Circuit Breaker."**

You can answer:

> "Circuit Breaker is a resilience pattern used in distributed systems to prevent repeated calls to an unhealthy downstream service and avoid cascading failures. It generally has three states: Closed, Open and Half-Open. In the Closed state requests flow normally and failures are monitored. When failures cross a configured threshold, the circuit moves to Open and subsequent calls fail fast without reaching the downstream service. After a configured break duration, it moves to Half-Open and allows limited test requests. If the downstream service has recovered, the circuit closes; otherwise it returns to Open. In a .NET microservices application, I would typically combine circuit breaker with timeout, bounded retry with backoff, and appropriate fallback. I would also monitor circuit state, failure rate, latency and rejected requests."

### One sentence to remember

**Retry says "try again"; Circuit Breaker says "stop trying for a while."**


##-------------------xxx------------------------------------------

Absolutely. These four problems are closely related and are exactly why **Circuit Breaker** is useful in microservices.

Let's use one simple scenario throughout:

```text
User
 ↓
Order API
 ↓
Payment Service
```

Suppose the **Payment Service is down or very slow**.

---

# 1. Thread Exhaustion

### What is a thread?

A server uses threads to execute work.

Imagine the Order API has a limited number of worker threads available.

```text
Order API
┌─────────────────────────┐
│ Thread 1 → Payment API  │
│ Thread 2 → Payment API  │
│ Thread 3 → Payment API  │
│ Thread 4 → Payment API  │
│ Thread 5 → Payment API  │
│ ...                     │
└─────────────────────────┘
```

Now suppose Payment Service normally responds in:

```text
100 ms
```

But because Payment Service is unhealthy, each request takes:

```text
30 seconds
```

Requests start piling up:

```text
Request 1 → Thread 1 → waiting for Payment
Request 2 → Thread 2 → waiting for Payment
Request 3 → Thread 3 → waiting for Payment
Request 4 → Thread 4 → waiting for Payment
...
Request 100 → waiting
```

Eventually, the application doesn't have enough available execution resources to handle new work efficiently.

That's **thread exhaustion**.

### Simple definition

> **Thread exhaustion occurs when too many threads are occupied waiting for slow or unresponsive operations, leaving insufficient execution capacity for new requests.**

### Important .NET clarification

In modern ASP.NET Core, you shouldn't think of this as "one dedicated thread is permanently assigned to every HTTP request." `async/await` allows threads to be returned to the thread pool while asynchronous I/O is waiting.

However, **thread-pool starvation/exhaustion can still occur**, especially when there is synchronous blocking such as:

```csharp
var result = paymentTask.Result;
```

or:

```csharp
paymentTask.Wait();
```

or when request processing contains long-running/blocking work.

So an architect should distinguish:

**slow downstream → request/resource buildup**

from the more specific:

**blocking work → ThreadPool starvation.**

---

# 2. Connection Exhaustion

Now let's look at connections.

The Order API needs a network connection to communicate with Payment Service.

```text
Order API
   |
   +---- Connection 1 ----> Payment
   +---- Connection 2 ----> Payment
   +---- Connection 3 ----> Payment
   +---- Connection 4 ----> Payment
   ...
```

There are limits on resources such as:

* HTTP connections
* sockets
* database connections
* connection-pool entries

Suppose Payment Service becomes very slow.

Connections remain occupied much longer:

```text
Normal:

Request → Payment
          ↓
        100 ms
          ↓
       connection released


Payment unhealthy:

Request → Payment
          ↓
        30 sec
          ↓
       connection occupied
```

As more requests arrive:

```text
Connection Pool

[Busy]
[Busy]
[Busy]
[Busy]
[Busy]
[Busy]
[Busy]
[Busy]
```

Eventually new requests may have to **wait for a connection** or fail because the available connection/resource limit has been reached.

That's **connection exhaustion**.

### Simple definition

> **Connection exhaustion occurs when available network or database connections are occupied for too long, preventing new requests from acquiring connections.**

---

# 3. Increased Latency

Now combine the two.

Normally:

```text
User
 ↓
Order API       50 ms
 ↓
Payment         100 ms
 ↓
Response
```

Total:

```text
~150 ms
```

Payment becomes slow:

```text
User
 ↓
Order API
 ↓
Payment Service
 ↓
30 seconds
 ↓
Order API
 ↓
User
```

Now the user waits:

```text
30 seconds
```

That's **increased latency**.

But there's another effect.

As requests accumulate:

```text
Request 1 → 30 sec
Request 2 → 30 sec
Request 3 → 30 sec
...
Request 100 → waiting
```

New requests don't necessarily get processed immediately.

Therefore latency can become even worse:

```text
100 ms
   ↓
500 ms
   ↓
2 sec
   ↓
10 sec
   ↓
30 sec
   ↓
Timeout
```

### Simple definition

> **Increased latency means requests take progressively longer to complete because downstream operations are slow or resources are becoming constrained.**

---

# 4. Cascading Failures

This is the most important concept.

Imagine your system has multiple services:

```text
                    Payment ❌
                       ↑
                       |
User → Order → Payment
        |
        ↓
    Inventory
        |
        ↓
   Notification
```

Payment becomes unhealthy.

Order starts accumulating:

```text
Order API
   ↓
Waiting for Payment
   ↓
More requests
   ↓
More resources consumed
```

Now Order itself becomes slow.

Inventory calls Order:

```text
Inventory
    ↓
Order API
    ↓
Payment ❌
```

Inventory starts experiencing problems.

Then another service calls Inventory:

```text
Service A
   ↓
Inventory
   ↓
Order
   ↓
Payment ❌
```

The original failure was:

```text
Payment Service
```

But eventually:

```text
Payment ❌
    ↓
Order degraded
    ↓
Inventory degraded
    ↓
Other services degraded
    ↓
Entire system degraded
```

This is a **cascading failure**.

### Simple definition

> **A cascading failure occurs when the failure or slowness of one service propagates through dependent services and causes additional services to become unhealthy.**

---

# 5. Eventually, Order API itself fails

Now put everything together.

Suppose:

```text
Payment Service ❌
       ↓
Requests wait longer
       ↓
Connections stay occupied
       ↓
Resources become constrained
       ↓
Latency increases
       ↓
Requests start timing out
       ↓
More retries arrive
       ↓
More load
       ↓
Order API becomes overloaded
       ↓
Order API starts returning 5xx/errors
```

So the original problem was:

```text
Payment Service
```

But now:

```text
Order API ❌
```

This is why we call it a **cascading failure**.

---

# 6. Where Circuit Breaker helps

Without Circuit Breaker:

```text
                  Payment ❌
                     ↑
                     |
1000 requests → Order API
                     |
                     ↓
              1000 attempts
                     |
                     ↓
              resources consumed
                     |
                     ↓
               Order API ❌
```

With Circuit Breaker:

```text
                 Payment ❌
                    ↑
                    |
Order API → Circuit Breaker
                    |
                 OPEN
                    |
                    X
              No more calls
```

The Order API **fails fast** instead of waiting for Payment repeatedly.

For example:

```text
Payment failures
       ↓
5 failures detected
       ↓
Circuit OPEN
       ↓
Don't call Payment
       ↓
Return fallback/error immediately
```

This protects Order API resources.

---

# 7. The complete chain — remember this

For your Architect interview, remember this sequence:

```text
Downstream service becomes slow/unavailable
                  ↓
         Requests take longer
                  ↓
       Resources remain occupied
                  ↓
        Connection exhaustion
                  ↓
          Resource starvation
                  ↓
         Increased latency
                  ↓
           Request timeouts
                  ↓
       Retries create more load
                  ↓
        Cascading failures
                  ↓
          Order API fails
```

And the resilience strategy is:

```text
              Downstream
                  ↑
                  |
Timeout → Retry → Circuit Breaker
                  |
                  ↓
              Fallback
```

### Interview-ready answer

> **"If a downstream service becomes unavailable, requests can remain in-flight for a long time. This causes connections and other resources to remain occupied, increasing latency and potentially causing resource starvation or ThreadPool starvation when blocking operations are involved. Retries can further increase the load. Eventually, the problem can propagate to dependent services, resulting in a cascading failure. A circuit breaker prevents this by detecting repeated failures, opening the circuit, and failing fast instead of continuously calling the unhealthy dependency."**

That last paragraph is a **very good 60-second Architect-level answer**.

