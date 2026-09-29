👍

# CDN — Senior / Staff / Architect Level

### 1. What is a CDN?

**CDN = Content Delivery Network**

A CDN is a **globally distributed network of edge servers** that delivers content from a location **closer to the user**.

Instead of:

```text
User → Application Server → Database
```

we try to serve static/cacheable content as:

```text
                    ┌── Edge: Mumbai
                    │
User → CDN ─────────┼── Edge: Singapore
                    │
                    └── Edge: London
```

**Simple interview definition:**

> "A CDN is a distributed network of edge servers that caches and delivers static or cacheable content closer to users, reducing latency, bandwidth consumption, and load on the origin server."

---

# 2. Real-world example

Imagine an e-commerce application:

```text
                    ┌───────────────┐
                    │ Origin Server │
                    │  .NET Web API │
                    └───────┬───────┘
                            │
                     ┌──────▼──────┐
                     │     CDN     │
                     └──────┬──────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
          Mumbai Edge   Singapore Edge  London Edge
              ▲             ▲             ▲
              │             │             │
          Indian User   Asian User    UK User
```

Suppose your website has:

```text
logo.png
product-image.jpg
app.js
style.css
product-video.mp4
```

These files don't change frequently.

Without CDN:

```text
User → India → Origin Server → Download image
```

Every request reaches your origin.

With CDN:

```text
User → Mumbai CDN Edge → image
```

If the image is already cached at Mumbai:

```text
User
  │
  ▼
Mumbai CDN
  │
  └── Cache HIT → Return image
```

The origin server isn't contacted.

---

# 3. Cache HIT vs Cache MISS

This is **very important for interviews**.

### First request — Cache MISS

```text
User
 ↓
CDN
 ↓
Cache MISS
 ↓
Origin Server
 ↓
CDN caches response
 ↓
User
```

For example:

```text
GET /images/iphone.jpg
```

CDN doesn't have it.

So:

```text
CDN → Origin
```

Origin returns:

```text
iphone.jpg
```

CDN stores it.

---

### Second request — Cache HIT

Another user requests:

```text
GET /images/iphone.jpg
```

Now:

```text
User
 ↓
CDN
 ↓
Cache HIT
 ↓
User
```

No origin request.

---

# 4. What should we put behind CDN?

Usually:

### Static content

```text
HTML
CSS
JavaScript
Images
Videos
Fonts
PDFs
Downloads
```

### Sometimes cacheable API responses

For example:

```http
GET /api/products/123
```

If the product data can tolerate some staleness:

```text
User
 ↓
CDN
 ↓
Cached API response
```

But don't blindly cache dynamic/personalized APIs.

For example:

```http
GET /api/my-account
```

should generally **not** be publicly cached because the response is user-specific.

---

# 5. How does CDN know whether content can be cached?

The application/origin sends HTTP cache headers.

For example:

```http
Cache-Control: public, max-age=3600
```

Meaning:

> This response can be cached and considered fresh for 1 hour.

Another common strategy:

```http
Cache-Control: public, max-age=31536000, immutable
```

Useful for versioned assets:

```text
app.a82f91.js
style.72ac21.css
```

Because when content changes, we generate a new filename.

```text
app.v1.js
app.v2.js
```

This is called **cache busting**.

---

# 6. CDN Architecture

A typical architecture could be:

```text
                       Internet
                          │
                          ▼
                    ┌───────────┐
                    │   CDN     │
                    │ Edge POPs │
                    └─────┬─────┘
                          │
                 Cache MISS only
                          │
                          ▼
                    ┌───────────┐
                    │   WAF     │
                    └─────┬─────┘
                          │
                          ▼
                   Load Balancer
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
           API-1        API-2       API-3
              │           │           │
              └───────────┼───────────┘
                          ▼
                       Database
```

In Azure, a common architecture can involve:

```text
User
 ↓
Azure Front Door
 ↓
WAF
 ↓
Origin / Load Balancer
 ↓
Azure App Service / AKS
 ↓
Database
```

Azure Front Door can provide **global routing, edge delivery, caching/CDN capabilities, TLS termination, and WAF integration**.

---

# 7. CDN vs Load Balancer

This is a common interview question.

| CDN                      | Load Balancer                            |
| ------------------------ | ---------------------------------------- |
| Delivers cached content  | Distributes requests                     |
| Mainly improves latency  | Mainly improves availability/scalability |
| Uses edge locations      | Usually routes to backend instances      |
| Reduces origin traffic   | Spreads traffic across servers           |
| Cache-aware              | Usually request-aware                    |
| Example: images, JS, CSS | API/server requests                      |

Think:

> **CDN = "Where can I serve this content closest to the user?"**

> **Load Balancer = "Which backend server should handle this request?"**

They can work together.

---

# 8. How does CDN choose the edge server?

Usually based on **network/location information**, DNS/anycast routing, and the CDN's internal routing system.

Example:

```text
User in Pune
     ↓
CDN routing
     ↓
Nearest/best-performing POP
     ↓
Mumbai/Pune-region edge
```

User in London:

```text
London User
     ↓
CDN routing
     ↓
London/European POP
```

The goal isn't simply geographical distance; CDNs can consider **network latency, availability, congestion, and routing conditions**.

---

# 9. What happens when content changes?

Suppose:

```text
product.jpg
```

is cached for 1 hour.

You update the image.

The CDN may still return the old image until the cache expires.

Architectural solutions:

### Option 1 — TTL

```text
Cache-Control: max-age=3600
```

Wait for expiration.

### Option 2 — Purge / Invalidation

```text
Purge /images/product.jpg
```

CDN removes the cached version.

### Option 3 — Versioned URL

Instead of:

```text
product.jpg
```

use:

```text
product-v2.jpg
```

or:

```text
product.abc123.jpg
```

This is often preferred for static application assets.

---

# 10. CDN caching strategies

### Time-based

```text
TTL = 1 hour
```

After 1 hour → revalidate/fetch again.

### Cache-Control

```http
Cache-Control: public, max-age=3600
```

### ETag

Origin returns:

```http
ETag: "abc123"
```

CDN/client can ask:

```http
If-None-Match: "abc123"
```

If unchanged:

```text
304 Not Modified
```

Less data needs to be transferred.

---

# 11. CDN and security

A CDN is not only about performance.

It can also provide:

```text
CDN
 │
 ├── TLS termination
 ├── WAF
 ├── DDoS protection
 ├── Rate limiting
 ├── Bot protection
 └── Access control
```

For example:

```text
Attacker
   │
   ▼
CDN/WAF
   │
   X  Malicious request blocked
   │
   ▼
Origin
```

Therefore, the origin receives fewer malicious requests.

---

# 12. Important architect-level consideration

**Don't cache everything.**

Suppose:

```http
GET /api/products
```

is public.

Caching could be good.

But:

```http
GET /api/orders/my-orders
```

returns:

```text
User A's orders
```

If incorrectly cached as public content, another user could potentially receive the wrong response.

So architects must define:

```text
Cache Key
+
TTL
+
Cache-Control
+
Authentication behavior
+
Invalidation strategy
```

---

# 13. CDN cache key

A CDN generally determines whether two requests can use the same cached object based on a **cache key**.

Conceptually:

```text
https://example.com/products?id=100
```

might produce:

```text
Cache Key =
/products?id=100
```

But headers/cookies/query parameters can also affect caching depending on configuration.

For example:

```text
/products?id=100
/products?id=101
```

must normally be different cache entries.

Architect question:

> "What happens if the cache key ignores an important query parameter?"

Answer:

> The CDN could return the cached response for the wrong request, so cache-key design is critical for correctness and security.

---

# 14. CDN in a .NET + Azure application

Imagine:

```text
Angular
   │
   ├── JS/CSS/images
   │
   ▼
 Azure CDN / Front Door
   │
   │
   └───────────────┐
                   │
                   ▼
              .NET Web API
                   │
             Azure Service Bus
                   │
                   ▼
              Background Worker
                   │
                   ▼
                Database
```

For static assets:

```text
Angular → CDN → cached JS/CSS/images
```

For dynamic API:

```text
Angular → CDN/Front Door → WAF → Load Balancer/App Service → .NET API
```

For asynchronous processing:

```text
.NET API → Service Bus → Worker
```

CDN and Service Bus solve **different problems**.

---

# 15. CDN vs Application Cache

Another useful distinction:

### CDN

```text
Global / edge
       ↓
User-facing content
```

### Redis

```text
Application
       ↓
Shared application data/cache
```

Example:

```text
                    ┌── CDN ── Static files
User → Front Door ──┤
                    └── API → Redis → Database
```

CDN is primarily about **delivering content closer to users**.

Redis is primarily about **reducing expensive application/database operations**.

---

# 16. Benefits

Remember:

### **L-A-S-T**

**L — Latency reduction**

Content is closer to users.

**A — Availability**

Content can often continue being served from edge cache even when origin is under pressure.

**S — Scalability**

Reduces traffic hitting origin servers.

**T — Traffic cost reduction**

Less bandwidth/load between users and origin.

Also:

```text
+ Better performance
+ DDoS/WAF integration
+ Global distribution
+ Reduced origin load
```

---

# 17. Limitations / trade-offs

A Staff/Architect answer should mention these.

```text
1. Cache invalidation complexity
2. Stale data
3. Cache-key mistakes
4. Personalized content is difficult
5. Additional infrastructure cost
6. Configuration complexity
7. Cache MISS still reaches origin
8. Dynamic APIs may provide little benefit
```

The classic problem is:

> **"There are only two hard things in Computer Science: cache invalidation and naming things."**

---

# 18. Interview answer — 30 seconds

If interviewer asks **"Explain CDN."**

Say:

> "CDN stands for Content Delivery Network. It is a globally distributed network of edge servers that caches and serves static or cacheable content close to users. For example, in an e-commerce application, product images, JavaScript and CSS can be cached at an edge location near the user. On the first request there is a cache miss, so the CDN fetches the content from the origin and caches it. Subsequent users get a cache hit directly from the edge, reducing latency and origin load. At an architect level, I would also consider TTL, cache keys, cache invalidation, security, personalized content, and consistency before deciding what to cache."

### Easy memory:

```text
              CDN
               │
       "Bring content closer"
               │
       ┌───────┼───────┐
       ▼       ▼       ▼
     User    User    User
      ↓       ↓       ↓
    Edge    Edge    Edge
       \      |      /
        \     |     /
          Origin
```

**One-line mental model:**

> **CDN = Cache content at the edge, closer to the user, so the origin doesn't have to serve every request.**
