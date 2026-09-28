# Database Sharding vs Partitioning — Senior Developer / Architect

This is a very common **Senior/Staff/Architect system-design topic**. The easiest way to remember it is:

> **Partitioning = split data inside one database system.**
> **Sharding = split data across multiple database instances/servers.**

---

# 1. Why do we need them?

Imagine an `Orders` table has:

```text
1 billion rows
```

and receives:

```text
50,000 writes/sec
200,000 reads/sec
```

A single database may eventually become a bottleneck because of:

* CPU
* Memory
* I/O
* Storage
* Index size
* Lock contention
* Backup/restore time
* Network throughput

We can scale vertically:

```text
Small DB
   ↓
Bigger CPU
Bigger RAM
Faster Disk
```

But vertical scaling has limits and becomes expensive.

So we consider:

```text
Partitioning
        or
Sharding
```

---

# 2. Database Partitioning

Partitioning divides a large logical table into smaller physical partitions **within the same database system**.

For example:

```text
Orders
------------------------------------------------
OrderId | CustomerId | OrderDate | Amount
------------------------------------------------
```

Instead of one huge physical structure:

```text
Orders
  └── 1 Billion rows
```

we can partition by year:

```text
Orders
 ├── Partition 2024
 ├── Partition 2025
 ├── Partition 2026
 └── Partition 2027
```

The application can still query:

```sql
SELECT *
FROM Orders
WHERE OrderDate >= '2026-01-01'
  AND OrderDate < '2027-01-01';
```

The database optimizer can perform **partition pruning**, scanning only the relevant partition.

---

# 3. Types of Partitioning

The three important types to know are:

### Range Partitioning

Partition according to ranges.

```text
Orders

2024 → Partition 1
2025 → Partition 2
2026 → Partition 3
2027 → Partition 4
```

Typical key:

```text
OrderDate
```

Excellent for:

* Time-series data
* Logs
* Transactions
* Audit data

---

### List Partitioning

Partition according to predefined values.

```text
CustomerData

India       → Partition 1
USA         → Partition 2
UK          → Partition 3
Germany     → Partition 4
```

Example:

```text
Country = 'IN'
Country = 'US'
Country = 'UK'
```

Useful when values have natural business boundaries.

---

### Hash Partitioning

Apply a hash function to the partition key.

For example:

```text
hash(CustomerId) % 4
```

Results:

```text
Customer 101 → Partition 1
Customer 102 → Partition 3
Customer 103 → Partition 0
Customer 104 → Partition 2
```

This helps distribute data more evenly.

---

# 4. What is Sharding?

Now let's go one level further.

Instead of keeping all partitions inside one database server:

```text
                Database
                   |
       ┌───────────┼───────────┐
       ▼           ▼           ▼
   Partition 1 Partition 2 Partition 3
```

we distribute data across **multiple database instances**:

```text
                 Application
                     |
             Sharding Layer
                     |
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
   DB Shard 1     DB Shard 2     DB Shard 3
   Customers      Customers      Customers
   1-1M           1M-2M          2M-3M
```

Each shard owns a subset of the data.

For example:

```text
Shard 1 → CustomerId 1–1,000,000
Shard 2 → CustomerId 1,000,001–2,000,000
Shard 3 → CustomerId 2,000,001–3,000,000
```

Now database load is distributed.

---

# 5. The most important difference

|                         | Partitioning                       | Sharding                  |
| ----------------------- | ---------------------------------- | ------------------------- |
| Scope                   | Within database                    | Across database instances |
| Physical DB             | Usually one                        | Multiple                  |
| Primary goal            | Manage/query large datasets        | Horizontal scale          |
| Data distribution       | Partitions                         | Shards                    |
| Operational complexity  | Lower                              | Higher                    |
| Cross-partition queries | Usually easier                     | Potentially expensive     |
| Scaling                 | Limited by database infrastructure | Can scale horizontally    |

The key interview statement:

> **Partitioning improves manageability and query performance within a database, while sharding distributes data and workload across independent database nodes to achieve horizontal scalability.**

---

# 6. Real-world example

Suppose we have an e-commerce application.

Tables:

```text
Customers
Orders
OrderItems
Payments
Products
```

Suppose we have:

```text
500 million customers
5 billion orders
20 billion order items
```

We could shard based on:

```text
CustomerId
```

For example:

```text
                Application
                     |
              Shard Resolver
                     |
        ┌────────────┼────────────┐
        ▼            ▼            ▼
      Shard 1      Shard 2      Shard 3
      Customer     Customer     Customer
      1-10M        10-20M       20-30M
```

Orders belonging to a customer stay on the same shard:

```text
Customer 123
     |
     +── Orders
     +── OrderItems
     +── Payments
```

This is called **data locality**.

---

# 7. Choosing a Shard Key

This is one of the most important **Architect-level questions**.

Suppose we shard using:

```text
CustomerId
```

Why?

Because most queries are:

```sql
SELECT *
FROM Orders
WHERE CustomerId = @customerId;
```

The application can determine the shard immediately:

```text
CustomerId
      ↓
Hash / Routing
      ↓
Shard 7
      ↓
Query
```

This is called **targeted routing**.

---

# 8. Bad shard key example

Suppose we choose:

```text
Country
```

and 70% of our customers are from India.

We get:

```text
India     → 70%
USA       → 10%
UK        → 5%
Germany   → 5%
Others    → 10%
```

Now:

```text
India Shard
████████████████████████████
USA
████
UK
██
```

This creates a **hot shard**.

One shard becomes overloaded while others are underutilized.

This is called:

> **Shard imbalance / hotspot**

---

# 9. Good shard key characteristics

A good shard key should generally provide:

### 1. High cardinality

Avoid keys with only a few values.

Bad:

```text
Gender
Country
Status
```

Better:

```text
CustomerId
TenantId
AccountId
```

### 2. Even distribution

We don't want:

```text
Shard 1 → 80%
Shard 2 → 10%
Shard 3 → 10%
```

We want something closer to:

```text
Shard 1 → 33%
Shard 2 → 33%
Shard 3 → 34%
```

### 3. Query locality

Ideally, common queries should hit **one shard**.

For example:

```text
GET /customers/{customerId}/orders
```

should go directly to:

```text
Shard X
```

rather than querying every shard.

---

# 10. Hash Sharding

One common approach:

```text
Shard = Hash(CustomerId) % NumberOfShards
```

For example:

```text
Hash(12345) % 4 = 2
```

Therefore:

```text
Customer 12345
      ↓
Shard 2
```

Advantages:

* Good distribution
* Simple routing
* Reduces hotspots

But there's an important problem.

Suppose:

```text
4 shards
```

and we change to:

```text
8 shards
```

The hash calculation changes:

```text
hash(key) % 4
```

becomes:

```text
hash(key) % 8
```

A lot of data may need to move.

That's why production systems may use:

### Consistent hashing

or a more sophisticated **shard mapping/routing table**.

---

# 11. Shard Mapping Table

Instead of calculating everything directly:

```text
CustomerId → Hash → Shard
```

we can maintain:

```text
ShardMap

CustomerRange     Shard
--------------------------------
1 - 1,000,000     DB01
1M - 2M           DB02
2M - 3M           DB03
```

Application:

```text
CustomerId
    ↓
Shard Map
    ↓
DB02
    ↓
Query
```

This provides flexibility to move ranges between shards.

---

# 12. Sharding and Multi-Tenant Architecture

This is particularly important for enterprise SaaS.

Suppose we have:

```text
Tenant A
Tenant B
Tenant C
Tenant D
```

We could use:

```text
TenantId
```

as the shard key.

Architecture:

```text
                  Application
                       |
                   TenantId
                       |
                  Shard Router
                       |
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       DB-Shard1    DB-Shard2    DB-Shard3
       Tenant A     Tenant C     Tenant D
       Tenant B
```

This provides:

* Tenant isolation
* Horizontal scalability
* Independent scaling
* Potential data residency options

But you need to consider tenant migration.

For example:

```text
Tenant A
   |
   ↓
Shard 1
```

becomes:

```text
Tenant A
   |
   ↓
Shard 4
```

You need a safe migration mechanism.

---

# 13. Cross-Shard Query — the biggest challenge

Suppose:

```sql
SELECT *
FROM Orders
WHERE Amount > 10000;
```

If Orders are sharded:

```text
Shard 1
Shard 2
Shard 3
Shard 4
```

Which shard contains the data?

Potentially **all of them**.

The system may have to perform:

```text
        Query
          |
    ┌─────┼─────┐
    ▼     ▼     ▼
   S1     S2    S3
    │     │     │
    └─────┼─────┘
          ▼
       Merge
          ▼
       Result
```

This is called a:

> **Scatter-Gather query**

It can become expensive.

---

# 14. Cross-Shard Transactions

This is another major architectural problem.

Suppose:

```text
Order → Shard 1
Payment → Shard 2
```

You need:

```text
Create Order
+
Create Payment
```

If both need to succeed atomically, you have a distributed transaction problem.

Traditional ACID transaction:

```text
BEGIN TRANSACTION

Order
Payment

COMMIT
```

doesn't work naturally across independent database servers.

Architectural alternatives include:

```text
Saga Pattern
+
Transactional Outbox
+
Event-driven architecture
```

For example:

```text
Order Created
      ↓
Event
      ↓
Payment Service
      ↓
Payment Completed
      ↓
Event
      ↓
Order Confirmed
```

This gives **eventual consistency** rather than relying on a distributed transaction.

---

# 15. Partitioning + Sharding can coexist

Very important:

> **Partitioning and sharding aren't mutually exclusive.**

You can have:

```text
                    Application
                         |
                    Sharding
                         |
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Shard 1        Shard 2        Shard 3
          │              │              │
       Partition       Partition       Partition
          │              │              │
      2024 2025       2024 2025       2024 2025
```

For example:

```text
Shard by CustomerId
+
Partition Orders by OrderDate
```

This is a powerful architecture for very large systems.

---

# 16. SQL Server example

Suppose we have:

```sql
CREATE TABLE Orders
(
    OrderId BIGINT,
    CustomerId BIGINT,
    OrderDate DATETIME2,
    Amount DECIMAL(18,2)
);
```

We may partition by:

```text
OrderDate
```

Conceptually:

```text
Orders
 ├── 2024
 ├── 2025
 ├── 2026
 └── 2027
```

Now query:

```sql
SELECT *
FROM Orders
WHERE OrderDate >= '2026-01-01'
AND OrderDate < '2027-01-01';
```

The database can eliminate irrelevant partitions.

That's **partition pruning**.

---

# 17. Partitioning doesn't automatically mean better performance

This is an important Senior-level point.

Partitioning can help when queries contain the partition key:

```sql
WHERE OrderDate >= ...
```

But if the query is:

```sql
WHERE CustomerId = 12345
```

and the table is partitioned only by `OrderDate`, the database may still need to inspect many partitions.

Therefore:

> **Partitioning strategy must align with access patterns.**

You still need appropriate indexes.

---

# 18. Indexing with partitioning

Imagine:

```text
Partition by OrderDate
```

and query:

```sql
WHERE CustomerId = 123
```

You might need an index such as:

```text
(CustomerId)
```

depending on workload and database design.

Partitioning is **not a replacement for indexing**.

Think:

```text
Partitioning → reduces data scope
Indexing     → accelerates lookup within that scope
```

---

# 19. Sharding vs Replication

Another common interview question.

### Sharding

Splits data:

```text
S1 → Customers 1-1M
S2 → Customers 1M-2M
S3 → Customers 2M-3M
```

### Replication

Copies the same data:

```text
Primary
   |
   ├── Replica 1
   ├── Replica 2
   └── Replica 3
```

So:

> **Sharding is primarily about distributing data/workload. Replication is about creating copies for availability and/or read scaling.**

They can be combined:

```text
             Shard 1
            /        \
        Primary     Replica

             Shard 2
            /        \
        Primary     Replica
```

---

# 20. When should you introduce sharding?

Don't immediately shard every application.

I'd first consider:

```text
1. Query optimization
2. Proper indexes
3. Connection pooling
4. Caching
5. Read replicas
6. Vertical scaling
7. Table partitioning
8. Archiving
9. Then sharding when necessary
```

Because sharding introduces substantial complexity:

* Routing
* Data migration
* Cross-shard queries
* Cross-shard transactions
* Backup/restore
* Monitoring
* Rebalancing
* Schema migrations
* Operational complexity

A strong architect doesn't say:

> "We have lots of data, therefore let's shard."

Instead:

> "We introduce sharding when the workload or data volume exceeds the practical scaling limits of a single database and other optimization techniques are insufficient."

---

# 21. Interview scenario

### Interviewer:

> "You have 10 billion orders. How would you scale the database?"

I'd answer:

I would first understand the workload rather than immediately choosing sharding.

I would look at read/write throughput, query patterns, data growth, index size, storage and I/O bottlenecks, transaction boundaries, and whether the workload is read-heavy or write-heavy.

I would first optimize indexes and queries, introduce caching where appropriate, consider read replicas for read-heavy workloads, and use table partitioning if the data has a natural partition key such as OrderDate.

If the database still becomes a horizontal scaling bottleneck, I would consider sharding.

For an order system, CustomerId or TenantId could be a candidate shard key, depending on the dominant access pattern. I would choose a key that provides reasonably even distribution and allows most business queries to be routed to a single shard.

I would introduce a shard-routing layer or shard map that maps the logical key to the physical database. The application should not contain hard-coded database addresses.

I would also design for the consequences of sharding: cross-shard queries become more expensive, cross-shard transactions become distributed transactions, and operational tasks such as backup, schema migration, rebalancing and tenant movement become more complex.

For distributed business transactions, I would generally prefer patterns such as Saga and Transactional Outbox rather than relying heavily on distributed two-phase transactions.

I would also consider combining sharding with partitioning—for example, shard by TenantId or CustomerId and partition Orders by OrderDate.

So my decision would be driven by workload, scale and access patterns, with sharding being a deliberate horizontal-scaling strategy rather than the first optimization technique.

---

# 22. The architecture picture to remember

```text
                         Application
                              │
                       Routing Layer
                              │
                       Shard Resolver
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
           Shard 1         Shard 2         Shard 3
              │               │               │
         ┌────┼────┐     ┌────┼────┐     ┌────┼────┐
         ▼    ▼    ▼     ▼    ▼    ▼     ▼    ▼    ▼
       2024 2025 2026   2024 2025 2026   2024 2025 2026
       Part  Part  Part  Part  Part  Part  Part  Part  Part
```

Here:

```text
Sharding
   ↓
Horizontal distribution across DB nodes

Partitioning
   ↓
Logical/physical subdivision inside a DB

Replication
   ↓
Copies of data for availability/read scaling
```

### The 30-second answer

> **Partitioning splits a large table into smaller partitions, usually within the same database, to improve manageability and potentially query performance through partition pruning. Sharding goes a level further by distributing subsets of data across independent database instances, enabling horizontal scaling.**
>
> **The key architectural decision in sharding is choosing the shard key. I want high cardinality, even distribution, and query locality. I also need to design for cross-shard queries, distributed transactions, rebalancing, migrations, backups, and operational complexity. In a large SaaS system, I might shard by TenantId and partition each tenant's data by time.**

That distinction—**partitioning for data organization/access efficiency vs sharding for horizontal database scaling**—is the core concept interviewers usually want to hear.
