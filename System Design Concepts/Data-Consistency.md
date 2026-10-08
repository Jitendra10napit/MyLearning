## Data Consistency — Senior Developer / Architect Explanation

**Data consistency means that when data is read from a system, it follows the expected and agreed rules, even when multiple users, services, databases, or replicas are involved.**

A simple way to remember:

> **Consistency = “Will different parts of my system see data in an acceptable and predictable state?”**

---

# 1. Real-world example: Bank transfer

Suppose Rahul has:

**Account A = ₹10,000**

He transfers **₹2,000** to Priya.

Expected final state:

```text
Rahul   = ₹8,000
Priya   = ₹7,000
```

The system should not end up with:

```text
Rahul   = ₹8,000
Priya   = ₹5,000   ❌
```

or:

```text
Rahul   = ₹10,000
Priya   = ₹7,000   ❌
```

This is a **consistency problem**.

---

# 2. Types of Data Consistency

For Senior/Staff/Architect interviews, understand these major types:

```text
Data Consistency
│
├── Strong Consistency
├── Eventual Consistency
├── Read-Your-Writes Consistency
├── Monotonic Read Consistency
├── Causal Consistency
└── Session Consistency
```

The most important distinction in distributed systems is:

> **Strong Consistency vs Eventual Consistency**

---

# 3. Strong Consistency

With **strong consistency**, after data is successfully written, subsequent reads immediately see the latest value.

### Example

Suppose you update:

```text
Order Status = "Shipped"
```

Database:

```text
Order 101 → Shipped
```

Immediately another API reads:

```text
GET /orders/101
```

It gets:

```json
{
  "orderId": 101,
  "status": "Shipped"
}
```

There is no window where another reader sees `"Processing"` after the write has been acknowledged.

### Real-world examples

Strong consistency is generally preferred for:

* Banking transactions
* Payment processing
* Inventory deduction
* Financial ledger
* Critical authorization decisions

### Simple diagram

```text
Client
   │
   │ WRITE: Balance = 8000
   ▼
Primary Database
   │
   │ Replication completed
   ▼
Replicas
   │
   ▼
READ → 8000
```

The system may sacrifice some **latency/availability** to provide stronger consistency guarantees.

---

# 4. Eventual Consistency

With **eventual consistency**, different replicas may temporarily have different values, but they eventually converge/Merge/Synced.

### Example

Suppose an e-commerce product has:

```text
Stock = 10
```

You purchase one item.

Primary database:

```text
Stock = 9
```

But replication takes a short time.

For a moment:

```text
Primary DB → 9
Replica 1  → 9
Replica 2  → 10   ← stale
```

A user reading from Replica 2 might temporarily see:

```text
Stock = 10
```

After replication:

```text
Primary DB → 9
Replica 1  → 9
Replica 2  → 9
```

The replicas **eventually become consistent**.

---

# 5. Why use Eventual Consistency?

Because distributed systems often need:

* High availability
* Low latency
* Horizontal scalability
* Geographical distribution

For example:

```text
                 ┌── DB Mumbai
                 │
Application ─────┼── DB Singapore
                 │
                 └── DB London
```

Keeping every database perfectly synchronized before responding can increase latency.

Instead:

```text
Write
  ↓
Primary
  ↓
Async replication
  ↓
Other replicas
```

This improves scalability but introduces a temporary **consistency window**.

---

# 6. Read-Your-Writes Consistency

This means:

> After I successfully write something, **I should be able to immediately read my own change.**

Example:

You change your profile name:

```text
POST /profile
name = "Jitendra"
```

Then immediately:

```text
GET /profile
```

You should see:

```text
Jitendra
```

Even if the system uses replicas.

### Problem

Imagine:

```text
Write → Primary DB
          ↓
       async replication

Read → Replica
```

If replication hasn't happened:

```text
Write: Jitendra
Read:  Old Name ❌
```

That violates **read-your-writes consistency**.

### Common solution

After a write, route the user's subsequent reads to the primary for some period:

```text
User
 │
 ├── WRITE ──→ Primary
 │
 └── READ ───→ Primary
```

Or use mechanisms such as session affinity / replication-aware reads.

---

# 7. Monotonic Read Consistency

This means:

> Once a user has seen a newer version of data, they should not later see an older version.

Example:

First request:

```text
Order Status = Shipped
```

Second request should not return:

```text
Order Status = Processing ❌
```

This can happen when requests are routed to different replicas:

```text
Request 1 → Replica A → Version 5
Request 2 → Replica B → Version 3 ❌
```

The user appears to see data going **backward in time**.

A system providing monotonic reads prevents this.

---

# 8. Causal Consistency

Causal consistency preserves the relationship between related events.

Suppose:

```text
User creates post
       ↓
User comments on post
```

The comment logically depends on the post existing.

Therefore another user should not see:

```text
Comment
   ↓
Post appears later
```

Instead:

```text
Post
 ↓
Comment
```

The system maintains the **causal relationship**.

---

# 9. Session Consistency

Session consistency means consistency guarantees are maintained within a user's session.

Imagine:

```text
User logs in
    ↓
Updates profile
    ↓
Reads profile
    ↓
Reads profile again
```

During that session, the user shouldn't unexpectedly see older versions of their own data.

This is commonly useful in applications where users interact continuously with the system.

---

# 10. Consistency in Microservices

This is especially important for **Senior/Architect interviews**.

Suppose you have:

```text
Order Service
     │
     ▼
Payment Service
     │
     ▼
Inventory Service
```

User places an order.

Ideally:

```text
Order Created
     ↓
Payment Successful
     ↓
Inventory Reserved
```

But these may have separate databases:

```text
Order DB
Payment DB
Inventory DB
```

You usually **cannot use one traditional database transaction across all three services**.

So you may use:

### Saga Pattern

```text
Create Order
     ↓
Process Payment
     ↓
Reserve Inventory
     ↓
Order Confirmed
```

If inventory fails:

```text
Inventory Reservation ❌
        ↓
Compensating Action
        ↓
Refund Payment
        ↓
Cancel Order
```

This is an example of managing **distributed data consistency**.

---

# 11. CAP Theorem Connection

Consistency becomes particularly important in distributed databases.

CAP says that during a **network partition**, a distributed system must choose between:

```text
Consistency
Availability
```

while partition tolerance is assumed for a distributed system.

So:

```text
CAP
├── Consistency
├── Availability
└── Partition Tolerance
```

For example:

### CP-style system

Prioritize:

```text
Correct/latest data
```

Potentially reject/delay requests during a partition.

### AP-style system

Prioritize:

```text
Availability
```

Allow operations to continue and reconcile data later.

---

# 12. Consistency vs Transaction Atomicity

Don't confuse these.

### Atomicity

Means:

> Either the whole transaction happens or none of it happens.

Example:

```text
Debit ₹2,000
Credit ₹2,000
```

Both should succeed or both should roll back.

### Consistency

Means:

> The transaction moves the system from one valid state to another valid state.

Example:

```text
Before:
A = 10000
B = 5000

Transfer 2000

After:
A = 8000
B = 7000
```

The business/data rules remain valid.

---

# 13. Easy analogy: WhatsApp/Chat

Imagine you send:

```text
"Hello"
```

Your phone immediately shows:

```text
You: Hello
```

But the recipient's phone receives it 500 ms later.

For a short period:

```text
Your phone       → Hello
Recipient phone  → Nothing
```

This is effectively an **eventual synchronization** scenario.

Once the message is delivered:

```text
Your phone       → Hello
Recipient phone  → Hello
```

They converge.

---

# 14. Interview-ready answer

If an interviewer asks:

**"What is data consistency?"**

You can say:

> **Data consistency means ensuring that data remains correct and follows defined business rules when it is accessed or modified, especially in distributed systems. There are different consistency models. Strong consistency guarantees that once a write is acknowledged, subsequent reads see the latest value. Eventual consistency allows temporary differences between replicas but guarantees that they converge eventually. In microservices, because services often have independent databases, we commonly use patterns such as Saga, transactional outbox, idempotency and asynchronous messaging to achieve the required level of consistency without relying on distributed transactions.**

### One line to remember

```text
Strong Consistency  → "I write it, everyone immediately sees it."

Eventual Consistency → "I write it, others may see old data briefly,
                         but eventually everyone sees the new data."
```

For a **Staff/Architect interview**, the key is not just defining consistency—you should explain **where you deliberately choose strong vs eventual consistency and what trade-off you make in latency, availability, scalability, and complexity.**
