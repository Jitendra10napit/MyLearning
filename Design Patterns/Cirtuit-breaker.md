## Circuit Breaker — At a Glance

### Simple definition

> **Circuit Breaker prevents a microservice from repeatedly calling a failing service. After too many failures, it temporarily stops the calls and fails fast. Once the service recovers, it allows calls again.**

Think of an **electrical circuit breaker**:

```text
Normal → Too many failures → STOP → Check recovery → Normal
 CLOSED        OPEN          OPEN      HALF-OPEN      CLOSED
```

---

## Simple .NET Microservice Example

Suppose:

```text
Order Service
      │
      │ HTTP Call
      ↓
Payment Service
```

Normally:

```text
Order Service → Payment Service → Payment Successful
```

But Payment Service goes down:

```text
Order Service → Payment Service → ❌
Order Service → Payment Service → ❌
Order Service → Payment Service → ❌
```

After, say, **5 failures**, Circuit Breaker opens:

```text
Order Service
      │
      ↓
Circuit Breaker
      │
      X
      ↓
Payment Service
```

Now the next requests **don't even call Payment Service**.

```text
Request
   ↓
Circuit OPEN
   ↓
Fail Fast / Payment Pending
```

After some time:

```text
OPEN
  ↓
HALF-OPEN
  ↓
Test Payment Service
  ↓
   ┌──────────────┐
   │              │
Success         Failure
   │              │
   ↓              ↓
CLOSED           OPEN
```

---

## Easy Real-Life Example

Imagine **100 customers** are placing orders.

Payment Service is down.

Without Circuit Breaker:

```text
100 customers
      ↓
100 calls to Payment Service
      ↓
100 failures/timeouts
      ↓
Order Service resources consumed
      ↓
System becomes slow
```

With Circuit Breaker:

```text
First few calls
      ↓
Payment Service failing
      ↓
Circuit OPEN
      ↓
Remaining requests fail immediately
      ↓
Order Service remains healthy
```

### ⭐ Main purpose

> **Protect my service from a failing downstream service and prevent cascading failures.**

---

## The 3 States You Must Remember

| State            | Meaning                             |
| ---------------- | ----------------------------------- |
| 🟢 **CLOSED**    | Calls are allowed                   |
| 🔴 **OPEN**      | Calls are blocked                   |
| 🟡 **HALF-OPEN** | Allow a test call to check recovery |

```text
             Failures exceed threshold
CLOSED ─────────────────────────→ OPEN
  ↑                                │
  │                                │ Wait
  │                                ↓
  └──────── Success ─────── HALF-OPEN
                                   │
                                   └── Failure → OPEN
```

---

## Interview Answer — 30 Seconds

> **"Circuit Breaker is a resilience pattern used in microservices to prevent cascading failures. For example, if my Order Service calls a Payment Service and Payment starts returning errors or timing out, after a configured failure threshold the circuit opens. Further calls are blocked and fail fast instead of continuously hitting the unhealthy Payment Service. After a recovery period, the circuit moves to half-open and allows a test request. If it succeeds, the circuit closes; otherwise, it opens again. In .NET, I can implement this using the resilience capabilities built around Polly."**

### Remember this one line:

**Retry says → "Try again."**
**Circuit Breaker says → "Stop calling; the service is unhealthy."**



Absolutely. For a **Senior Developer / Technical Lead interview**, explain **Retry + Circuit Breaker together**, because they solve different problems and are often used in the same resilience pipeline.

## 1. At a glance

```text
             Order Service
                   |
                   | Call Payment Service
                   v
          +-------------------+
          | Circuit Breaker   |
          +-------------------+
                   |
             Retry transient
                failures
                   |
                   v
          +-------------------+
          | Payment Service   |
          +-------------------+
                   |
              Bank API
```

### Simple rule to remember

> **Retry = "Try again."**
> **Circuit Breaker = "Stop trying for a while."**

---

# 2. Retry — implementation logic

Suppose Order Service calls Payment Service.

Payment Service temporarily returns:

```text
Request 1 → Timeout
Request 2 → 503
Request 3 → Success
```

Retry allows the request to be attempted again.

### Basic logic

```text
Call Payment Service
       |
       +---- Success → Return response
       |
       +---- Transient failure
                  |
                  v
             Wait briefly
                  |
                  v
             Retry #1
                  |
             Failure?
                  |
                  v
             Retry #2
                  |
             Success?
                  |
                  v
                Done
```

### C# simplified implementation

```csharp
public async Task<PaymentResponse> ProcessPaymentAsync(
    PaymentRequest request)
{
    int maxRetries = 3;

    for (int attempt = 1; attempt <= maxRetries; attempt++)
    {
        try
        {
            return await _paymentClient.ProcessAsync(request);
        }
        catch (HttpRequestException) when (attempt < maxRetries)
        {
            await Task.Delay(TimeSpan.FromSeconds(attempt));
        }
    }

    throw new Exception("Payment service unavailable");
}
```

In real applications, I wouldn't normally write this retry loop manually. I'd use the .NET resilience pipeline / Polly-based resilience features.

For example:

```csharp
builder.Services
    .AddHttpClient<IPaymentClient, PaymentClient>()
    .AddStandardResilienceHandler();
```

The important interview point is **what you retry**, not just how.

### Don't blindly retry

Good retry candidates:

```text
503 Service Unavailable
429 Too Many Requests
Temporary network failure
Timeout
```

Usually don't retry:

```text
400 Bad Request
401 Unauthorized
403 Forbidden
Business validation failure
Insufficient balance
```

---

# 3. Circuit Breaker — implementation logic

Now imagine Payment Service is completely down.

Without Circuit Breaker:

```text
Order 1 → Payment ❌
Order 2 → Payment ❌
Order 3 → Payment ❌
Order 4 → Payment ❌
...
Order 10,000 → Payment ❌
```

You're continuously sending traffic to an unhealthy service.

Circuit Breaker prevents this.

### Logic

```text
              ┌─────────────┐
              │   CLOSED    │
              │ Normal calls│
              └──────┬──────┘
                     │
              failures exceed
                threshold
                     │
                     ▼
              ┌─────────────┐
              │    OPEN     │
              │ Don't call  │
              │ downstream  │
              └──────┬──────┘
                     │
               wait duration
                     │
                     ▼
              ┌─────────────┐
              │ HALF-OPEN   │
              │ Test call   │
              └──────┬──────┘
                │           │
             Success       Failure
                │           │
                ▼           ▼
             CLOSED        OPEN
```

### Example

Suppose:

```text
Failure threshold = 5
Break duration    = 30 seconds
```

Then:

```text
Request 1 → Payment ❌
Request 2 → Payment ❌
Request 3 → Payment ❌
Request 4 → Payment ❌
Request 5 → Payment ❌

Circuit opens
```

Now:

```text
Request 6 → ❌ Fail fast
Request 7 → ❌ Fail fast
Request 8 → ❌ Fail fast
```

**Request 6–8 don't even hit Payment Service.**

After 30 seconds:

```text
Circuit → HALF-OPEN

Test request → Payment

       Success
          ↓
      CLOSED

       Failure
          ↓
       OPEN
```

---

# 4. Retry vs Circuit Breaker

| Retry                      | Circuit Breaker                     |
| -------------------------- | ----------------------------------- |
| Tries the request again    | Stops calling downstream            |
| Handles temporary failures | Handles persistent failures         |
| Short-term recovery        | Prevents cascading failure          |
| Adds additional calls      | Reduces calls                       |
| "Try again"                | "Stop calling"                      |
| Example: timeout once      | Example: Payment down for 5 minutes |

### Very simple example

Imagine calling a restaurant.

**Retry:**

> "The phone didn't connect. I'll call again."

**Circuit Breaker:**

> "I've called 10 times and the restaurant is clearly unavailable. I'll stop calling for a while."

---

# 5. Retry + Circuit Breaker together

This is where the interview becomes interesting.

Consider:

```text
Order Service
      |
      v
+------------------+
| Circuit Breaker  |
+------------------+
      |
      v
+------------------+
| Retry Policy     |
+------------------+
      |
      v
Payment Service
```

Suppose Payment Service has a temporary network problem.

```text
Order
  |
  v
Circuit CLOSED
  |
  v
Retry #1 → Timeout
  |
  v
Retry #2 → 503
  |
  v
Retry #3 → Success
  |
  v
Order successful
```

But suppose Payment is completely down:

```text
Order
  |
  v
Circuit CLOSED
  |
  v
Retry #1 → 503
Retry #2 → 503
Retry #3 → 503
  |
  v
Failures accumulate
  |
  v
Circuit OPEN
```

Now future requests fail fast instead of repeatedly hitting Payment.

---

# 6. Important architect-level point: order of policies

This is a common interview follow-up.

Conceptually:

```text
Request
   |
   v
Circuit Breaker
   |
   v
Retry
   |
   v
Payment Service
```

The exact nesting/behavior depends on the resilience library configuration, so in a real .NET application I would explicitly verify the pipeline semantics rather than relying on a diagram.

The important design principle is:

> **Retry should handle transient failures, while Circuit Breaker should stop repeated calls when the dependency is persistently unhealthy.**

---

# 7. Real .NET implementation approach

For modern .NET, you can use the resilience handler:

```csharp
builder.Services
    .AddHttpClient<IPaymentClient, PaymentClient>()
    .AddStandardResilienceHandler();
```

Conceptually, your resilience strategy becomes:

```text
HTTP Request
     |
     v
Timeout
     |
     v
Retry
     |
     v
Circuit Breaker
     |
     v
Payment API
```

You can also configure specific resilience strategies rather than relying only on the standard handler.

For example, conceptually:

```csharp
builder.Services
    .AddHttpClient<IPaymentClient, PaymentClient>()
    .AddStandardResilienceHandler(options =>
    {
        options.Retry.MaxRetryAttempts = 3;

        options.TotalRequestTimeout.Timeout =
            TimeSpan.FromSeconds(10);
    });
```

The exact configuration should be chosen based on the API's behavior and your application's requirements.

---

# 8. One very important payment example

For **payment operations**, don't say:

> "I'll retry every failed payment."

That's dangerous.

Imagine:

```text
Order Service
     |
     | Charge ₹10,000
     v
Payment Service
     |
     v
Bank
```

Payment succeeds at the bank, but the response times out:

```text
Order → Payment
           |
           v
        Bank
           |
       Payment SUCCESS
           |
           X
      Response lost
```

Order Service sees:

```text
Timeout
```

If you blindly retry:

```text
Retry → Charge ₹10,000
```

You could potentially charge the customer twice.

### Architect solution

Use **idempotency**:

```text
OrderId = ORD123
PaymentId = PAY123
IdempotencyKey = ORD123-PAYMENT
```

Then:

```text
First request
     ↓
Payment processed
     ↓
Response lost

Retry
     ↓
Same Idempotency Key
     ↓
Payment Service recognizes
existing transaction
     ↓
Returns existing result
```

So your resilience architecture becomes:

```text
                 Order Service
                      |
                      v
              +---------------+
              | Timeout       |
              +---------------+
                      |
                      v
              +---------------+
              | Retry         |
              +---------------+
                      |
                      v
              +---------------+
              | Circuit       |
              | Breaker       |
              +---------------+
                      |
                      v
              Payment Service
                      |
                Idempotency
                      |
                      v
                  Bank API
```

---

# 9. Best interview answer

If interviewer asks:

**"What's the difference between Retry and Circuit Breaker?"**

Say:

> "Retry and Circuit Breaker are both resilience patterns, but they solve different problems. Retry handles transient failures by attempting the operation again, for example when an API temporarily returns a 503 or a network timeout occurs. Circuit Breaker handles persistent failures. After a configured number of failures, it opens the circuit and prevents further calls to the unhealthy dependency, allowing it time to recover and also protecting my service from cascading failures. In a microservice architecture, I would typically combine timeout, bounded retry, circuit breaker and, where applicable, fallback or asynchronous processing. For non-idempotent operations such as payments, I would also use idempotency to prevent duplicate processing."

### One-line memory trick

```text
RETRY          → Temporary problem → TRY AGAIN
CIRCUIT BREAKER → Persistent problem → STOP CALLING
TIMEOUT         → Taking too long → STOP WAITING
IDEMPOTENCY     → Duplicate request → PROCESS ONLY ONCE
```

This four-line distinction is **very useful for a Technical Lead interview**.
