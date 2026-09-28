For a **Senior .NET / Staff / Architect interview**, think of cache as:

> **A fast temporary storage layer that keeps frequently accessed data closer to the application so we don't repeatedly hit the database or another slow service.**

## 1. Real-time example — E-commerce product

Suppose Amazon has this API:

```http
GET /api/products/1001
```

Without cache:

```text
User
  ↓
API
  ↓
Database
  ↓
Product
```

If 10,000 users request the same product:

```text
10,000 API requests
        ↓
10,000 DB queries
```

This creates unnecessary database load.

With Redis cache:

```text
User
  ↓
API
  ↓
Redis Cache
  ├── HIT → Return product immediately
  │
  └── MISS
       ↓
     Database
       ↓
     Store in Redis
       ↓
     Return product
```

### Example

First request:

```text
GET Product 1001

Redis:
Product:1001 → NOT FOUND

       ↓

Database → Product information

       ↓

Redis:
Product:1001 → {
   Id: 1001,
   Name: "iPhone",
   Price: 70000
}

       ↓

Response
```

Second request:

```text
GET Product 1001

Redis:
Product:1001 → FOUND

       ↓

Response
```

The database isn't touched for the second request.

---

# 2. Why do we need caching?

Caching mainly improves:

* **Latency** — memory is much faster than database/network calls.
* **Scalability** — reduces database traffic.
* **Availability** — some applications can continue serving cached data if downstream systems have problems.
* **Cost** — fewer database operations.
* **Throughput** — more requests can be handled by the application.

For example:

```text
Without cache:

API → DB
     ↑
     10,000 requests


With cache:

API → Redis
     ↑
     10,000 requests

Redis → DB
         ↑
      maybe 100 requests
```

The exact reduction depends on the application's access pattern and cache hit rate.

---

# 3. Cache-aside — most common pattern

This is one of the most important things to know for interviews.

```text
             ┌──────────┐
             │   Redis  │
             └────┬─────┘
                  │
Application ──────┤
                  │
                  ↓
             ┌──────────┐
             │ Database │
             └──────────┘
```

Application controls the cache.

### Read

```csharp
var product = await cache.GetAsync<Product>(
    $"product:{id}");

if (product == null)
{
    product = await db.Products.FindAsync(id);

    await cache.SetAsync(
        $"product:{id}",
        product,
        TimeSpan.FromMinutes(10));
}

return product;
```

Conceptually:

```text
Cache GET
   ↓
 HIT ─────────→ Return
   │
 MISS
   ↓
Database
   ↓
Update cache
   ↓
Return
```

This is called **Cache-Aside / Lazy Loading**.

---

# 4. What is a Cache Hit and Cache Miss?

### Cache Hit

Data exists:

```text
Application → Cache → FOUND
                       ↓
                     Return
```

Example:

```text
Cache hit ratio = 95%
```

means approximately 95% of requests found the required data in cache.

### Cache Miss

Data isn't available:

```text
Application → Cache → NOT FOUND
                         ↓
                      Database
                         ↓
                    Populate cache
```

A high **cache hit ratio** is generally desirable, but it should be interpreted according to the workload and data freshness requirements.

---

# 5. Write policies

This is a very common interview topic.

There are three important strategies.

## A. Write-through

Application writes to cache, and cache synchronously writes to database.

```text
Application
     ↓
   Cache
     ↓
 Database
```

Example:

```text
Update product price

API
 ↓
Redis
 ↓
Database
```

Only after the database update succeeds is the write considered successful.

### Advantage

Cache and DB are more closely synchronized.

### Disadvantage

Every write has additional cache/DB work and potentially higher latency.

---

# 6. Write-back / Write-behind

Application writes to cache first.

Database update happens asynchronously later.

```text
Application
     ↓
   Cache
     ↓
   Queue
     ↓
 Database
```

Example:

```text
User updates profile

API → Cache → Response immediately

              ↓
          Background worker
              ↓
           Database
```

### Advantage

Very fast writes.

### Disadvantage

If the cache fails before persistence, data can potentially be lost unless the architecture provides durable buffering/recovery.

Useful when:

* very high write volume
* eventual consistency is acceptable
* asynchronous persistence is acceptable

---

# 7. Write-around

Application writes directly to database.

Cache is not updated immediately.

```text
Application
    ├────→ Database
    │
    └────→ Cache only when read
```

Example:

```text
UPDATE Product
       ↓
Database

Later:

GET Product
       ↓
Cache MISS
       ↓
Database
       ↓
Cache
```

### Advantage

Avoids caching data that may never be read.

### Disadvantage

First read after a write gets a cache miss.

---

# 8. Easy way to remember write policies

| Policy            | Write goes to | DB update                            |
| ----------------- | ------------- | ------------------------------------ |
| **Write-through** | Cache         | Immediately/synchronously            |
| **Write-back**    | Cache         | Later/asynchronously                 |
| **Write-around**  | DB            | Immediately; cache populated on read |

Remember:

> **Through = Cache + DB together**
> **Back = DB comes later**
> **Around = bypass cache on write**

---

# 9. Cache eviction policies

Eventually cache becomes full.

Suppose Redis can hold:

```text
1 GB
```

and you need to insert another item.

The cache needs to decide:

> **Which existing item should I remove?**

That's eviction.

---

## LRU — Least Recently Used

Remove the item that hasn't been accessed for the longest time.

Example:

```text
A → accessed 1 minute ago
B → accessed 10 minutes ago
C → accessed 1 hour ago
```

Cache is full.

Evict:

```text
C
```

Because C was least recently used.

### Real-world example

Netflix-style content:

```text
Movie A → watched frequently
Movie B → watched recently
Movie C → nobody watched for months
```

Movie C becomes a candidate for eviction.

**LRU is probably the most important eviction policy to know.**

---

# 10. LFU — Least Frequently Used

Remove the item accessed the fewest times.

Example:

```text
Product A → accessed 10,000 times
Product B → accessed 500 times
Product C → accessed 2 times
```

If eviction is required:

```text
Evict C
```

because C has the lowest frequency.

Difference:

```text
LRU → When was it last used?

LFU → How many times was it used?
```

---

# 11. FIFO — First In, First Out

Remove the item that entered the cache first.

```text
A → entered first
B
C
D → entered last
```

Cache is full.

Evict:

```text
A
```

It doesn't care whether A is frequently accessed.

---

# 12. TTL — Time To Live

This is extremely important in real systems.

You can tell cache:

```text
Store this item for 10 minutes.
```

After 10 minutes:

```text
Product:1001
      ↓
TTL expires
      ↓
Removed / considered expired
```

Example:

```csharp
await cache.SetAsync(
    "product:1001",
    product,
    TimeSpan.FromMinutes(10));
```

TTL is especially useful for:

* configuration
* product information
* exchange rates
* API responses
* session-related data
* temporary tokens
* frequently changing information where stale data is acceptable for a limited period

---

# 13. LRU vs LFU vs TTL

Think about it like a parking lot.

### LRU

> "Which car hasn't been used recently?"

Remove that one.

### LFU

> "Which car has been used the fewest times?"

Remove that one.

### TTL

> "Which car's parking time has expired?"

Remove that one.

---

# 14. Cache invalidation — the difficult part

A famous engineering problem is:

> **How do I make sure cached data doesn't become stale?**

Suppose:

```text
Database:
Product price = ₹70,000

Cache:
Product price = ₹70,000
```

Then:

```text
Database updated:
₹65,000
```

But cache still contains:

```text
₹70,000
```

Now users receive stale data.

One solution:

```text
Update DB
   ↓
Invalidate cache
   ↓
Next GET
   ↓
Cache MISS
   ↓
DB
   ↓
Populate cache
```

Example:

```csharp
await db.UpdateProduct(product);

await cache.RemoveAsync($"product:{product.Id}");
```

---

# 15. Distributed cache — Redis

For a modern cloud application with multiple API instances:

```text
                Load Balancer
                     ↓
          ┌──────────┼──────────┐
          ↓          ↓          ↓
       API #1     API #2     API #3
          │          │          │
          └──────────┼──────────┘
                     ↓
                   Redis
                     ↓
                  Database
```

You generally don't want each API instance to have completely independent cache state when consistency across instances matters.

That's where **distributed caching** such as Redis becomes useful.

---

# 16. Local vs Distributed cache

### In-memory cache

```text
API #1
  ↓
Memory
```

Very fast, but data belongs to that instance.

If:

```text
API #1 → cache has Product A
API #2 → cache doesn't have Product A
```

you can get inconsistent cache contents.

### Distributed cache

```text
API #1 ──┐
API #2 ──┼──→ Redis
API #3 ──┘
```

All instances can access the same cache.

---

# 17. Architect-level answer

If interviewer asks:

**"How would you design caching for a scalable .NET application?"**

You can answer:

> "I would first identify read-heavy and relatively stable data suitable for caching. For a distributed application, I would typically use Redis as a distributed cache and use the cache-aside pattern. On a read, the application checks Redis first and falls back to the database on a miss, then populates the cache with an appropriate TTL. On updates, I would update the source of truth and invalidate or refresh the corresponding cache entry. For eviction, I would choose a policy such as LRU or LFU based on the access pattern, while TTL controls data freshness. I would also monitor cache hit ratio, latency, memory utilization, evictions, and failures."

That's a **very good Senior/Staff-level answer**.

### One-line interview memory trick

```text
CACHE
 ↓
Read → Cache-Aside → HIT / MISS
Write → Through / Back / Around
Full → LRU / LFU / FIFO
Old → TTL / Invalidation
Scale → Distributed Redis
Monitor → Hit Ratio + Latency + Memory + Evictions
```

**Most important distinction to remember:**

> **Eviction answers "what do I remove when cache is full?"**
> **TTL answers "how long can this item live?"**
> **Invalidation answers "when should I remove/update stale data?"**
