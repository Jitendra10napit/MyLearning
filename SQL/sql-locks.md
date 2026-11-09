The three hints control different things. **UPDLOCK** sets the *type* of lock, **HOLDLOCK** sets *how long it's held and what it covers*, and **ROWLOCK** sets its *size*.

| Hint | Controls | What it does |
|---|---|---|
| **UPDLOCK** | Lock type | Takes Update (U) locks instead of Shared (S) on read and holds them until the transaction ends. Other readers can still read, but only one session can hold a U lock. |
| **HOLDLOCK** | Duration + range | Same as SERIALIZABLE for that table. It keeps locks until commit and adds **key-range locks**, so nobody can insert a matching row (a "phantom"). |
| **ROWLOCK** | Granularity | Asks for row-level locks instead of page or table locks. It doesn't change the lock type or how long it's held. |

**UPDLOCK: prevents the read-then-update deadlock**
```sql
-- Without UPDLOCK: both sessions get S locks, both try to upgrade to X → deadlock
-- With UPDLOCK: the second session waits at the SELECT
SELECT Stock FROM Products WITH (UPDLOCK) WHERE Id = 1;
UPDATE Products SET Stock = Stock - 1 WHERE Id = 1;
```

**HOLDLOCK: protects rows that don't exist yet**
```sql
-- Upsert race: two sessions both see "not exists" and both INSERT → duplicate
IF NOT EXISTS (SELECT 1 FROM Users WITH (UPDLOCK, HOLDLOCK) WHERE Email = @e)
    INSERT INTO Users (Email) VALUES (@e);
```
Interview point: UPDLOCK alone **locks nothing if the row doesn't exist**. HOLDLOCK adds a range lock on the "gap" where the row would go, which blocks the other insert.

**ROWLOCK: limits blocking to your row**
```sql
UPDATE Orders WITH (ROWLOCK) SET Status = 'Shipped' WHERE Id = 42;
```
It's only a request. SQL Server can still escalate to a table lock if one statement takes about 5,000 or more locks, and without an index on `Id` it scans and locks far more anyway.

**Common combinations**
- `UPDLOCK, ROWLOCK`: lock one existing row to update it. Use it for booking seats or decrementing stock.
- `UPDLOCK, HOLDLOCK`: check-then-insert or upsert, and the standard fix for `MERGE` race conditions.
- `HOLDLOCK` alone: hold shared locks so the data can't change while you read it repeatedly. It's prone to deadlocks if you later write, so prefer adding UPDLOCK.

**Gotchas**
- These hints only matter **inside a transaction**. In autocommit mode the locks are released when the statement ends.
- HOLDLOCK range locks need a **good index**. Without one, it can lock the whole table.
- `NOLOCK` is the opposite: no shared locks, but you can see dirty or duplicated data. Use RCSI instead.
