For your **13+ years of experience and Senior/Lead/Staff .NET interviews**, SQL preparation should go well beyond writing `SELECT` statements. Interviewers at this level usually test whether you can diagnose production performance problems, reason about concurrency and transactions, design indexes, and make database decisions that scale.

Here is the SQL FAQ set I would prioritize.

## Core SQL concepts

### 1. `WHERE` vs `HAVING`

`WHERE` filters individual rows before grouping, whereas `HAVING` filters groups after `GROUP BY`.

```sql
SELECT DepartmentId, COUNT(*)
FROM Employees
WHERE IsActive = 1
GROUP BY DepartmentId;
```

With `HAVING`:

```sql
SELECT DepartmentId, COUNT(*) AS EmployeeCount
FROM Employees
GROUP BY DepartmentId
HAVING COUNT(*) > 10;
```

A useful way to remember it is:

```text
WHERE      → filter rows
GROUP BY   → create groups
HAVING     → filter groups
```

---

### 2. `INNER JOIN` vs `LEFT JOIN`

An `INNER JOIN` returns only rows having a match on both sides:

```sql
SELECT e.Name, d.Name
FROM Employees e
INNER JOIN Departments d
    ON e.DepartmentId = d.Id;
```

A `LEFT JOIN` preserves every row from the left table, even when there is no matching department:

```sql
SELECT e.Name, d.Name
FROM Employees e
LEFT JOIN Departments d
    ON e.DepartmentId = d.Id;
```

At senior level, explain the business intent rather than just the syntax: use an inner join when the relationship is required; use a left join when the left-side records must remain in the result.

---

## Query-writing questions

### 3. How do you find the second-highest salary?

One approach is:

```sql
SELECT MAX(Salary)
FROM Employees
WHERE Salary < (
    SELECT MAX(Salary)
    FROM Employees
);
```

If duplicate salary values need to receive the same ranking:

```sql
SELECT Salary
FROM (
    SELECT Salary,
           DENSE_RANK() OVER (ORDER BY Salary DESC) AS RankNo
    FROM Employees
) x
WHERE RankNo = 2;
```

`DENSE_RANK()` is particularly useful when you're talking about the **second distinct salary**.

---

### 4. How do you find the Nth-highest salary?

For the third-highest distinct salary:

```sql
SELECT Salary
FROM (
    SELECT Salary,
           DENSE_RANK() OVER (ORDER BY Salary DESC) AS RankNo
    FROM Employees
) x
WHERE RankNo = 3;
```

This is also a good opportunity to discuss window functions during an interview.

---

### 5. Explain `ROW_NUMBER()`, `RANK()` and `DENSE_RANK()`

Suppose the salaries are:

```text
100
100
90
80
```

`ROW_NUMBER()`:

```text
1
2
3
4
```

`RANK()`:

```text
1
1
3
4
```

`DENSE_RANK()`:

```text
1
1
2
3
```

Use `ROW_NUMBER()` when each row needs a unique sequence. Use `RANK()` when tied values share a rank and gaps are acceptable. Use `DENSE_RANK()` when tied values share a rank but you don't want gaps.

---

### 6. How do you identify duplicate records?

For example, duplicate email addresses:

```sql
SELECT Email, COUNT(*) AS Count
FROM Employees
GROUP BY Email
HAVING COUNT(*) > 1;
```

If you need to identify the actual duplicate rows:

```sql
WITH CTE AS
(
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY Email
               ORDER BY Id
           ) AS RN
    FROM Employees
)
SELECT *
FROM CTE
WHERE RN > 1;
```

---

### 7. How would you delete duplicate rows?

First identify them with a `SELECT`. Only after validating the result should you perform the deletion:

```sql
WITH CTE AS
(
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY Email
               ORDER BY Id
           ) AS RN
    FROM Employees
)
DELETE FROM CTE
WHERE RN > 1;
```

In a production system, I would not execute such a deletion without first verifying exactly which rows are being retained and removed.

---

## Indexing and performance

### 8. Clustered index vs non-clustered index

This is one of the most important SQL questions for your level.

A **clustered index** determines how table data is organized at the leaf level in SQL Server. Because the table's data is represented by that clustered index structure, a table can have only one clustered index.

```sql
CREATE CLUSTERED INDEX IX_Employees_Id
ON Employees(Id);
```

A **non-clustered index** is a separate structure that contains indexed keys and references to the underlying rows.

```sql
CREATE NONCLUSTERED INDEX IX_Employees_Email
ON Employees(Email);
```

SQL Server can have multiple non-clustered indexes.

A strong interview answer also mentions that indexes improve some reads but increase storage and write-maintenance cost.

---

### 9. What is a covering index?

Suppose the common query is:

```sql
SELECT Name, Salary
FROM Employees
WHERE DepartmentId = 10;
```

You could create:

```sql
CREATE INDEX IX_Employees_Department
ON Employees(DepartmentId)
INCLUDE (Name, Salary);
```

The key is `DepartmentId`, while `Name` and `Salary` are included columns.

If the optimizer can satisfy the query entirely from this index, it can avoid going back to the base table for those additional columns.

That's what we mean by a **covering index**.

---

### 10. What is a composite index?

For example:

```sql
CREATE INDEX IX_Orders_Customer_Status_Created
ON Orders(CustomerId, Status, CreatedDate);
```

The order of the columns is important.

A query such as:

```sql
WHERE CustomerId = 100
AND Status = 'Completed'
```

can potentially benefit substantially from this index.

However, don't claim that every query referencing one of these columns will automatically use the index. The optimizer considers the complete predicate, selectivity, statistics, ordering requirements and other factors.

---

### 11. Why does index column order matter?

Consider:

```sql
CREATE INDEX IX_Order
ON Orders(CustomerId, Status, CreatedDate);
```

The index is organized beginning with:

```text
CustomerId
    ↓
Status
    ↓
CreatedDate
```

Therefore, queries that can efficiently use the leading portion of the index generally benefit more.

This is commonly called the **leading-column or leftmost principle**.

---

### 12. Are more indexes always better?

No.

Every additional index can increase:

* disk consumption
* memory/cache usage
* `INSERT` cost
* `UPDATE` cost
* `DELETE` cost
* index maintenance

So I wouldn't create an index merely because a column appears in a `WHERE` clause.

At Staff level, say:

> "I look at the workload, selectivity, execution plan, existing indexes and write overhead before deciding whether an index is justified."

---

### 13. What is a SARGable query?

A SARGable predicate allows the database engine to use an index/search operation efficiently.

For example, this is less index-friendly:

```sql
WHERE YEAR(CreatedDate) = 2026
```

A range predicate is generally better:

```sql
WHERE CreatedDate >= '20260101'
AND CreatedDate < '20270101'
```

The second form doesn't apply a function to the indexed column itself and can therefore allow efficient range access.

---

### 14. What is the difference between an index seek and an index scan?

An **index seek** navigates to the relevant portion of an index.

```text
Index
  ↓
Locate required range
  ↓
Return rows
```

An **index scan** reads a larger portion, potentially the entire index.

But avoid saying:

> "Seek is good and scan is bad."

That's too simplistic for a senior interview.

A scan can be the correct and cheapest plan when the query needs a large percentage of the table.

---

### 15. What causes a table scan?

A scan may occur because:

* no suitable index exists
* the predicate isn't selective
* the query needs a large percentage of rows
* the optimizer estimates that scanning is cheaper
* statistics or cardinality estimates lead to a particular plan

The important point is that **a scan itself isn't necessarily a performance bug**.

---

### 16. What is a key lookup?

Suppose you have:

```sql
CREATE INDEX IX_Employee_Department
ON Employees(DepartmentId);
```

but execute:

```sql
SELECT Name, Salary
FROM Employees
WHERE DepartmentId = 10;
```

If `Name` and `Salary` aren't available in the index, SQL Server may locate the matching rows using the index and then perform **Key Lookups** to retrieve the remaining columns.

A covering index could potentially avoid those lookups:

```sql
CREATE INDEX IX_Employee_Department
ON Employees(DepartmentId)
INCLUDE(Name, Salary);
```

However, don't automatically eliminate every key lookup. For a highly selective query, the lookup may be cheaper than maintaining a wider index.

---

## Query troubleshooting

### 17. How would you investigate a slow SQL query?

This is one of the questions I would expect at your experience level.

My approach would be:

```text
Slow Query
    ↓
Capture SQL + parameters
    ↓
Look at actual execution plan
    ↓
Identify expensive operators
    ↓
Compare estimated vs actual rows
    ↓
Check scans / seeks / lookups
    ↓
Review indexes and statistics
    ↓
Check blocking / waits / deadlocks
    ↓
Rewrite or tune
    ↓
Benchmark before vs after
```

I'd also look at:

* CPU
* logical reads
* physical reads
* execution duration
* memory grants
* cardinality estimates
* parameter sensitivity
* blocking
* deadlocks
* wait statistics

---

### 18. What is an execution plan?

An execution plan describes how the database engine will execute—or did execute—a query.

For example:

```text
SELECT
  ↓
Nested Loops
  ↓
Index Seek
  ↓
Key Lookup
```

or:

```text
SELECT
  ↓
Hash Join
  ↓
Table Scan
  ↓
Table Scan
```

When troubleshooting, don't focus only on the operator with the largest displayed percentage. Look at the complete plan, actual versus estimated rows, joins, lookups, sorts, memory grants and other expensive operations.

---

### 19. What are database statistics?

Statistics help the optimizer estimate how many rows a predicate will return.

For example:

```sql
WHERE CustomerId = 100
```

The optimizer uses statistics to estimate the result size and select an execution strategy.

Poor or stale estimates can lead to:

```text
Incorrect row estimate
       ↓
Poor join choice
       ↓
Poor execution plan
       ↓
Slow query
```

---

## Transactions and concurrency

### 20. Explain ACID.

ACID describes the fundamental transaction properties:

**Atomicity** — all operations succeed or the transaction fails as a unit.

**Consistency** — the database moves between valid states.

**Isolation** — concurrent transactions don't incorrectly interfere with each other.

**Durability** — committed data survives failures according to the database's durability mechanisms.

```text
A → Atomicity
C → Consistency
I → Isolation
D → Durability
```

---

### 21. Explain transaction isolation levels.

In SQL Server, important isolation levels include:

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
SNAPSHOT
```

Broadly:

| Isolation        | Main characteristic                                              |
| ---------------- | ---------------------------------------------------------------- |
| Read Uncommitted | Dirty reads are possible                                         |
| Read Committed   | Prevents dirty reads                                             |
| Repeatable Read  | Protects rows already read from changes                          |
| Serializable     | Strong traditional locking isolation; prevents phantoms          |
| Snapshot         | Uses row versions rather than traditional reader/writer blocking |

The Staff-level point is the trade-off:

> Stronger isolation can provide stronger consistency guarantees, but it can also increase locking, blocking, or version-store/resource costs depending on the mechanism.

---

### 22. What is a dirty read?

Transaction A changes a value but hasn't committed.

Transaction B reads that uncommitted value.

```text
A:
UPDATE Salary = 100K
       ↓
Not committed

B:
Reads 100K

A:
ROLLBACK
```

B has read data that never actually committed.

That's a **dirty read**.

---

### 23. What is a phantom read?

Transaction A executes:

```sql
SELECT *
FROM Employees
WHERE Salary > 50000;
```

Transaction B inserts another employee whose salary is above 50,000.

Transaction A executes the same query again and now sees an additional row.

That newly appearing row is a **phantom row**.

---

### 24. What is blocking vs deadlocking?

**Blocking** means one transaction is waiting for another transaction's lock:

```text
A → holds lock
B → waits
```

When A finishes, B can continue.

A **deadlock** is circular waiting:

```text
A → waiting for B
B → waiting for A
```

Neither can proceed, so SQL Server detects the deadlock and chooses one transaction as the victim.

---

### 25. How do you prevent deadlocks?

I would consider:

1. Keeping transactions short.
2. Accessing resources in a consistent order.
3. Avoiding unnecessary locks.
4. Ensuring appropriate indexes exist.
5. Never waiting for user interaction while holding a transaction.
6. Keeping database operations efficient.
7. Examining deadlock graphs.
8. Adding carefully designed retry handling for appropriate transient/deadlock scenarios.

For example, consistently accessing:

```text
Employee → Department
```

is safer than having one code path access:

```text
Employee → Department
```

while another accesses:

```text
Department → Employee
```

Consistent resource ordering reduces deadlock risk.

---

## Data modeling

### 26. What is normalization?

Normalization reduces unnecessary duplication and helps prevent update anomalies.

Instead of:

```text
Employee
----------------
EmployeeId
EmployeeName
DepartmentName
DepartmentLocation
```

you might separate:

```text
Employee
---------
EmployeeId
EmployeeName
DepartmentId

Department
----------
DepartmentId
DepartmentName
DepartmentLocation
```

Common normal forms include:

```text
1NF
2NF
3NF
BCNF
```

---

### 27. Normalization vs denormalization

Normalization generally improves consistency and reduces duplication.

Denormalization intentionally duplicates information when read performance or a particular workload justifies it.

For example:

```text
Orders
---------
OrderId
CustomerId
CustomerName
```

can avoid a join in some read scenarios, but now `CustomerName` can become stale.

A strong answer is:

> "I normally begin with a normalized model and denormalize only when workload measurements justify the additional consistency and write complexity."

---

### 28. Partitioning vs sharding

**Partitioning** divides a large table into partitions within the database system.

For example:

```text
Orders

2024 → Partition 1
2025 → Partition 2
2026 → Partition 3
```

The application can still see one logical table.

**Sharding** distributes data across multiple database instances/nodes:

```text
Customer 1–1M
      ↓
DB Shard 1

Customer 1M–2M
      ↓
DB Shard 2
```

Simple distinction:

```text
Partitioning → within the database system
Sharding     → across database nodes/databases
```

---

## CTEs and intermediate results

### 29. What is a CTE?

A Common Table Expression gives a complex query a logical name:

```sql
WITH EmployeeCTE AS
(
    SELECT *
    FROM Employees
    WHERE IsActive = 1
)
SELECT *
FROM EmployeeCTE;
```

CTEs are useful for:

* readability
* complex query composition
* recursive queries
* window-function processing

An important clarification: a CTE is generally a query expression; it isn't automatically a materialized temporary table.

---

### 30. CTE vs temporary table

A CTE is often useful when you need a logical intermediate query within one statement.

A temporary table can be more appropriate when:

* intermediate data is reused
* you need indexes on the intermediate data
* multi-step processing is involved
* statistics on the intermediate result are useful

Don't claim that one is universally faster.

---

### 31. Temporary table vs table variable

For example:

```sql
CREATE TABLE #Employees
(
    Id INT,
    Name VARCHAR(100)
);
```

versus:

```sql
DECLARE @Employees TABLE
(
    Id INT,
    Name VARCHAR(100)
);
```

Modern SQL Server has improved table-variable behavior considerably, so the old rule that table variables are always better for small datasets is too simplistic.

For larger intermediate workloads where statistics and indexing matter, temporary tables are often a better choice.

---

## SQL Server architecture/performance questions

### 32. What is parameter sniffing?

SQL Server can compile a query plan using the parameter values encountered during compilation.

This can become problematic when the data distribution is highly skewed.

For example:

```sql
CREATE PROCEDURE GetOrders
    @CustomerId INT
AS
SELECT *
FROM Orders
WHERE CustomerId = @CustomerId;
```

Suppose:

```text
Customer A → 10 orders
Customer B → 10 million orders
```

A plan that works extremely well for one customer might perform badly for another.

Possible remedies depend on the actual problem and can include:

* query/index redesign
* updated statistics
* `OPTION (RECOMPILE)` where appropriate
* `OPTIMIZE FOR`
* Query Store analysis
* newer SQL Server features/behavior

Don't automatically add `RECOMPILE` without understanding the workload.

---

### 33. What is Query Store?

Query Store helps you track query execution behavior and performance over time.

Conceptually:

```text
Query
  ↓
Execution statistics
  ↓
Execution plans
  ↓
Plan changes
  ↓
Performance regression
```

It's particularly useful when a query that used to perform well suddenly becomes slow and you need to investigate plan changes.

---

### 34. What is the transaction log?

SQL Server maintains a transaction log to support transaction durability, rollback and crash recovery, among other recovery-related operations.

Conceptually:

```text
UPDATE
   ↓
Transaction Log
   ↓
COMMIT
```

Understanding transaction logs becomes important when discussing:

* large transactions
* bulk operations
* recovery models
* log growth
* high-availability/replication scenarios

---

### 35. What is SQL injection?

This is unsafe:

```csharp
var sql =
    "SELECT * FROM Users WHERE Name = '" + name + "'";
```

Malicious input could alter the intended SQL.

Prefer parameterized commands:

```csharp
command.Parameters.AddWithValue("@Name", name);
```

or use EF Core LINQ:

```csharp
var user = await db.Users
    .FirstOrDefaultAsync(x => x.Name == name);
```

---

## SQL and .NET architecture

### 36. EF Core vs stored procedures — which should you use?

Avoid an absolute answer.

A strong answer would be:

> "I generally use EF Core for maintainable application-level data access and LINQ-based querying. For complex database-specific processing, existing database contracts, specialized reporting, or cases where database-side processing is demonstrably advantageous, stored procedures can be appropriate. I decide based on performance, maintainability, operational ownership and business requirements."

---

# High-volume SQL architecture

### 37. How would you design SQL for a high-volume application?

At Staff level, I'd answer in terms of the workload rather than immediately choosing a technology.

```text
                     API
                      │
               ┌──────┴──────┐
               │             │
             Cache        Database
               │             │
               │       ┌─────┴─────┐
               │       │           │
               │    Primary    Read Replica
               │       │
               │   Partitioning
               │       │
               │     Indexes
               │       │
               │    Archival
               │
               └──────────────
```

I would first understand access patterns and workload. Then I'd consider appropriate keys and indexes, short transactions, efficient queries, pagination, caching where appropriate, and monitoring.

If one database becomes a bottleneck, I'd evaluate options such as read replicas, partitioning, caching, archival, or eventually sharding based on the actual workload.

---

# The 25 SQL questions I'd prioritize for your interview

For your **Kaseya/Staff-level preparation**, these are the ones I'd make sure you can answer without hesitation:

| Priority | Question                              |
| -------- | ------------------------------------- |
| 🔥🔥🔥   | Clustered vs non-clustered index      |
| 🔥🔥🔥   | How do you troubleshoot a slow query? |
| 🔥🔥🔥   | How do you read an execution plan?    |
| 🔥🔥🔥   | Index Seek vs Index Scan              |
| 🔥🔥🔥   | Composite indexes and column order    |
| 🔥🔥🔥   | Covering indexes                      |
| 🔥🔥🔥   | SARGability                           |
| 🔥🔥🔥   | N+1 database queries                  |
| 🔥🔥🔥   | Deadlocks and prevention              |
| 🔥🔥🔥   | Blocking vs deadlocks                 |
| 🔥🔥🔥   | Transaction isolation levels          |
| 🔥🔥🔥   | Optimistic vs pessimistic concurrency |
| 🔥🔥🔥   | Parameter sniffing                    |
| 🔥🔥🔥   | Statistics/cardinality estimation     |
| 🔥🔥🔥   | Partitioning vs sharding              |
| 🔥🔥     | `ROW_NUMBER` / `RANK` / `DENSE_RANK`  |
| 🔥🔥     | CTE vs temporary table                |
| 🔥🔥     | `DELETE` / `TRUNCATE` / `DROP`        |
| 🔥🔥     | `WHERE` vs `HAVING`                   |
| 🔥🔥     | JOIN types                            |
| 🔥🔥     | Normalization vs denormalization      |
| 🔥🔥     | Query Store                           |
| 🔥🔥     | ACID                                  |
| 🔥🔥     | SQL injection                         |
| 🔥🔥     | EF Core vs stored procedures          |

## How to frame your answers at Staff level

For most SQL questions, don't stop at the definition. A strong structure is:

```text
Concept
   ↓
Small SQL example
   ↓
How the database executes it
   ↓
Performance implications
   ↓
Trade-offs
   ↓
Production scenario
```

For example, instead of saying:

> "Indexes improve query performance."

say:

> "An index can allow the optimizer to locate qualifying rows more efficiently, but it introduces storage and write-maintenance overhead. Before adding one, I'd inspect the actual execution plan, query selectivity, existing indexes and workload. If a frequently executed `CustomerId + Status` query is doing expensive lookups, I might consider a composite index and possibly included columns, then benchmark the change."

That distinction—**concept + execution behavior + trade-off + production impact**—is what separates a basic SQL answer from the kind of answer expected from an experienced Staff/Lead .NET engineer.
