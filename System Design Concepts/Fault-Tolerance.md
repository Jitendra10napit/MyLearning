# Fault Tolerance — Handwritten Notes Style

## 1. What is Fault Tolerance?

> **Fault tolerance = System continues to provide service even when some component fails.**

Simple memory:

```text
Failure happens ❌
      ↓
System detects it
      ↓
System has alternative/recovery
      ↓
Service continues ✅
```

It doesn't mean **"nothing will fail."**

It means:

> **"Something can fail, but the overall system should continue working."**

---

# 2. Simple E-commerce Example

Imagine:

```text
User
 ↓
Load Balancer
 ↓
┌──────────┬──────────┬──────────┐
│ Server 1 │ Server 2 │ Server 3 │
└──────────┴──────────┴──────────┘
```

Suppose Server 2 crashes:

```text
Server 1 ✅
Server 2 ❌
Server 3 ✅
```

Load balancer detects the unhealthy server and sends traffic to:

```text
Server 1
Server 3
```

User may not even notice the failure.

**That's fault tolerance.**

---

# 3. Azure Example

Typical architecture:

```text
                Users
                  ↓
           Azure Front Door
                  ↓
          Load Balancer
             ↓       ↓
          App 1     App 2
             ↓       ↓
             Azure Service Bus
                    ↓
             Worker Functions
                    ↓
                  DB
```

Suppose Worker Function instance crashes:

```text
Worker 1 ❌
Worker 2 ✅
Worker 3 ✅
```

The message remains available for another worker to process.

```text
Message
   ↓
Worker 1 ❌
   ↓
Retry / Redelivery
   ↓
Worker 2 ✅
```

The business operation can continue.

---

# 4. Fault Tolerance vs High Availability

These are related but **not exactly the same**.

### High Availability

> **System is available most of the time.**

Example:

```text
Primary Server ❌
      ↓
Secondary Server
      ↓
Service continues
```

### Fault Tolerance

> **System continues operating despite component failure, ideally with little or no interruption.**

Think:

```text
HA = "Keep the service available"

FT = "Keep working despite failure"
```

---

# 5. Common Fault-Tolerance Techniques ⭐

Remember:

```text
R C T F R I
```

### 1. Redundancy

Have multiple instances.

```text
Server 1 ✅
Server 2 ❌
Server 3 ✅
```

---

### 2. Circuit Breaker

If downstream service keeps failing:

```text
Order API
    ↓
Payment API ❌
    ↓
Circuit OPEN
    ↓
Don't keep calling Payment
```

Prevents **cascading failures**.

---

### 3. Timeout

Never wait forever.

```text
Order → Payment
          ↓
       Timeout
          ↓
Fallback / Retry / Failure response
```

---

### 4. Retry

Useful for **transient failures**.

```text
Call Payment
    ↓
   ❌
    ↓
Retry
    ↓
   ❌
    ↓
Retry
    ↓
   ✅
```

Usually use **exponential backoff**.

```text
1 sec
2 sec
4 sec
8 sec
```

Don't blindly retry permanent errors.

---

### 5. Failover

```text
Primary DB ❌
     ↓
Secondary DB
     ↓
Continue
```

---

### 6. Isolation / Bulkhead

Don't allow one failing component to consume all resources.

```text
                    Application
                        │
              ┌─────────┴─────────┐
              ↓                   ↓
        Payment Pool        Order Pool
              ↓                   ↓
          Payment ❌           Order ✅
```

Payment failure shouldn't consume all threads/connections and bring down Order processing.

---

# 6. Fault Tolerance in Event-Driven Architecture

This connects directly to your previous **EDA + DLQ** topic.

```text
Order Service
     ↓
OrderPlaced Event
     ↓
Azure Service Bus
     ↓
Payment Consumer
     ↓
Payment API ❌
```

Instead of losing the event:

```text
Retry
 ↓
Retry
 ↓
Retry
 ↓
DLQ
```

Then:

```text
Fix problem
    ↓
Replay message
    ↓
Process successfully
```

So:

> **Durable messaging + retry + DLQ = important building blocks for fault-tolerant asynchronous processing.**

---

# 7. Fault Tolerance ≠ Retry

Very important interview point.

**Retry alone is not fault tolerance.**

For example:

```text
Service A
   ↓
Service B ❌
   ↓
Retry
   ↓
Retry
   ↓
Retry
   ↓
Still ❌
```

You may actually make the problem worse.

A robust design combines:

```text
Timeout
   +
Retry with backoff
   +
Circuit Breaker
   +
Fallback
   +
Redundancy
   +
Monitoring
```

---

# 8. Architect-Level Example

Imagine **Order Service → Payment Service**.

### Bad design

```text
Order API
    ↓
Payment API
    ↓
Wait forever ❌
```

If Payment is down:

```text
Payment ❌
 ↓
Order threads blocked
 ↓
Thread pool exhausted
 ↓
Order API ❌
```

This becomes a **cascading failure**.

### Better design

```text
Order API
    ↓
Timeout
    ↓
Retry + Backoff
    ↓
Circuit Breaker
    ↓
Payment Service
```

And for asynchronous payment:

```text
Order
 ↓
Service Bus
 ↓
Payment Worker
 ↓
Retry
 ↓
DLQ if required
```

Now Payment failure doesn't necessarily bring down the entire Order system.

---

# 9. Staff/Architect Interview Answer ⭐⭐⭐

> **"Fault tolerance is the ability of a system to continue providing its intended functionality despite failures in individual components. I achieve it through redundancy, health checks, load balancing, timeouts, retries with exponential backoff, circuit breakers, bulkheads, failover, and durable messaging. For example, if a payment service becomes unavailable, I wouldn't allow every order request to wait indefinitely. I'd use a timeout and circuit breaker, and for asynchronous workflows I'd use Azure Service Bus with retry and DLQ so messages aren't lost. The exact strategy depends on whether the failure is transient, permanent, or a complete component outage."**

### 🧠 One-line memory

```text
FAULT TOLERANCE =
Failure ❌ + Alternative/Recovery 🔄 = Service continues ✅
```

And the **architect's key question** is:

> **"If this component fails right now, what happens to the rest of my system?"**

That question naturally leads you to **timeouts → retry → circuit breaker → bulkhead → failover → DLQ → observability**.
