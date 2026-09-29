## Dead-Letter Queue — When does a message go there?

Think of **DLQ as the "parking area for messages that could not be successfully processed."**

### Typical scenario

```text
Producer
   │
   ▼
Azure Service Bus Queue
   │
   ▼
Consumer / Worker
   │
   ├── Process ✅ → Complete message
   │
   └── Process ❌
          │
          ▼
       Retry
          │
       Retry
          │
       Retry
          │
          ▼
        DLQ
```

### Example: E-commerce

Suppose:

```text
OrderPlaced
OrderId = ORD123
```

Payment service receives it.

```text
Payment Service
      ↓
Payment API
      ↓
Database ❌
```

The consumer fails.

Azure Service Bus can redeliver the message according to its delivery/retry behavior.

For example:

```text
Attempt 1 → Failed
Attempt 2 → Failed
Attempt 3 → Failed
Attempt 4 → Failed
Attempt 5 → Failed
                ↓
               DLQ
```

The exact threshold depends on the broker configuration; in Azure Service Bus, `MaxDeliveryCount` controls how many deliveries are attempted before the message is moved to the DLQ.

---

## What happens after DLQ?

**Important architect point:** DLQ is **not the end of the message lifecycle**.

```text
                 ┌───────────────┐
                 │   Main Queue  │
                 └───────┬───────┘
                         │
                    Processing
                         │
                    ❌ Repeated
                         │
                         ▼
                 ┌───────────────┐
                 │      DLQ      │
                 └───────┬───────┘
                         │
                  Investigation
                         │
             ┌───────────┴──────────┐
             ▼                      ▼
        Fix the issue          Discard/Archive
             │
             ▼
          Replay
             │
             ▼
        Main Queue
             │
             ▼
        Process again
```

### Example

Suppose the problem was:

```text
Consumer expects:
amount = integer

Message contains:
amount = "ABC"
```

Every retry fails because the data is invalid.

Eventually:

```text
DLQ
 ↓
Developer investigates
 ↓
Root cause identified
 ↓
Fix consumer / correct data
 ↓
Replay message
 ↓
Successfully processed
```

---

# Common reasons for DLQ

Remember **F-R-I-P-E**:

| Reason                    | Example                         |
| ------------------------- | ------------------------------- |
| **F — Failed processing** | Application exception           |
| **R — Retry exhausted**   | Max delivery count reached      |
| **I — Invalid message**   | Bad/invalid payload             |
| **P — Poison message**    | Message always crashes consumer |
| **E — Expired**           | Message TTL expired             |

Other broker-specific reasons can also cause dead-lettering, such as exceeding size or subscription/filter-related conditions.

---

## ⭐ Poison Message — Important Interview Scenario

This is a very common interview question.

```text
OrderPlaced
     ↓
Consumer
     ↓
NullReferenceException ❌
     ↓
Message not completed
     ↓
Redelivery
     ↓
Same exception ❌
     ↓
Redelivery
     ↓
Same exception ❌
     ↓
MaxDeliveryCount
     ↓
DLQ
```

This is called a **poison message** because the same message repeatedly causes processing failure.

---

## What should the Architect do?

Don't simply say:

> "Move it to DLQ."

Say:

```text
1. Retry transient failures
2. Use exponential backoff
3. Don't endlessly retry permanent failures
4. Move poison messages to DLQ
5. Monitor/alert on DLQ depth
6. Investigate root cause
7. Fix application/data problem
8. Replay message if appropriate
9. Ensure consumer is idempotent
10. Audit the final outcome
```

### ⭐ Interview-ready answer

> **"A message typically reaches the Dead-Letter Queue when it cannot be successfully processed after the configured delivery attempts, or when it meets a broker-defined dead-letter condition such as TTL expiration or an invalid delivery scenario. A common example is a poison message that repeatedly throws an exception. I would monitor the DLQ, investigate the failure, fix the underlying issue, and replay the message when safe, while ensuring the consumer is idempotent to avoid duplicate processing."**
