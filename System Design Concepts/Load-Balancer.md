## Load Balancer — Simple Explanation

The easiest way to remember:

> **Load Balancer = Traffic Police for Servers 🚦**

It receives incoming requests and decides **which server should handle each request**.

### Simple example

Suppose 10,000 users open your application.

Without a load balancer:

```text
             10,000 Users
                  |
                  ▼
             ┌─────────┐
             │ Server 1│
             └─────────┘
                  🔥
             Overloaded
```

With a load balancer:

```text
                Users
                  |
                  ▼
          ┌───────────────┐
          │ Load Balancer │
          └───────┬───────┘
                  |
        ┌─────────┼─────────┐
        ▼         ▼         ▼
     Server 1  Server 2  Server 3
       3,300     3,300     3,400
```

The **Load Balancer distributes the traffic** among the servers.

---

# Why do we need it?

Remember **3 things: S-A-F**

### 1. Scalability

Add more servers when traffic increases.

```text
3 Servers → 10 Servers
```

### 2. Availability

If Server 2 goes down:

```text
             Load Balancer
              /          \
             ▼            ▼
         Server 1      Server 3
             
         Server 2 ❌
```

The LB stops sending traffic to Server 2.

### 3. Fault tolerance

One server failing doesn't necessarily bring down the whole application.

---

# Types of Load Balancer

There are two important ways to classify them.

## 1. Layer 4 Load Balancer

**L4 = TCP/IP level**

It mainly looks at:

* IP address
* Port
* TCP/UDP connection

Example:

```text
Client
  |
  | TCP :443
  ▼
L4 Load Balancer
  |
  ├── Server 1
  ├── Server 2
  └── Server 3
```

It doesn't really care whether the request is:

```text
GET /users
GET /orders
POST /payment
```

It primarily deals with the network connection.

### Remember:

> **L4 = "Which server?" based on network connection.**

---

# 2. Layer 7 Load Balancer

**L7 = Application/HTTP level**

It understands things such as:

* HTTP URL
* Host
* HTTP method
* Headers
* Cookies

For example:

```text
                 L7 Load Balancer
                        |
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
      /users          /orders      /payments
          |             |             |
          ▼             ▼             ▼
     User Service   Order Service  Payment Service
```

So:

```text
GET /users/10
      ↓
User Service

GET /orders/20
      ↓
Order Service
```

### Remember:

> **L7 = "What is the request?" and route based on it.**

---

# L4 vs L7 — Easy Interview Table

|                   | L4                 | L7                          |
| ----------------- | ------------------ | --------------------------- |
| Layer             | Transport          | Application                 |
| Understands HTTP? | ❌ Generally no     | ✅ Yes                       |
| Uses              | IP, TCP, UDP, port | URL, headers, cookies, HTTP |
| Speed             | Very fast          | More processing             |
| Routing           | Connection-based   | Content-based               |
| Example           | TCP :443           | `/orders` → Order Service   |

### One-line memory trick

> **L4 = Connection**
> **L7 = Content**

---

# Load Balancing Algorithms

The LB also needs a method to decide **which server gets the request**.

### 1. Round Robin

Take turns:

```text
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1
```

Think:

> **"Your turn → Your turn → Your turn."**

---

### 2. Weighted Round Robin

Give more requests to powerful servers.

```text
Server 1 → Weight 5
Server 2 → Weight 3
Server 3 → Weight 2
```

Server 1 receives more traffic.

Think:

> **More powerful server = more work.**

---

### 3. Least Connections

Send the request to the server currently handling the fewest connections.

```text
Server 1 → 100 connections
Server 2 → 30 connections  ← New request
Server 3 → 70 connections
```

Think:

> **"Give the work to the least busy server."**

---

### 4. IP Hash

Use client's IP to decide the server.

```text
Client A → Server 1
Client B → Server 3
Client C → Server 2
```

The same client tends to reach the same server.

---

# Health Check

A good load balancer doesn't blindly send traffic.

It checks:

```text
Server 1 → Healthy ✅
Server 2 → Healthy ✅
Server 3 → Unhealthy ❌
```

Then:

```text
                  LB
               /      \
              ▼        ▼
          Server 1  Server 2
          
          Server 3 ❌
```

This is called a **health check**.

---

# Real-world architecture

A typical system can look like:

```text
                    Users
                      |
                      ▼
                  Load Balancer
                      |
             ┌────────┼────────┐
             ▼        ▼        ▼
           API 1     API 2    API 3
             |        |        |
             └────────┼────────┘
                      |
                 ┌────┴────┐
                 ▼         ▼
                Cache     Database
```

If traffic increases:

```text
API 1
API 2
API 3
      +
API 4
API 5
```

The load balancer starts distributing traffic to the new instances.

---

# 🎯 Best way to answer in interview

If interviewer asks **"What is a Load Balancer?"**, say:

> **"A load balancer sits between clients and backend servers and distributes incoming traffic across healthy server instances. It improves scalability, availability and fault tolerance. Load balancers can operate at Layer 4, where they work with TCP/IP connections, or Layer 7, where they understand application-level information such as HTTP paths and headers. Common routing algorithms include round robin, weighted round robin, least connections and IP hash."**

### 🧠 Final memory picture

```text
             LOAD BALANCER
                   ↓
       "Who should handle this?"
                   ↓
        ┌──────────┼──────────┐
        ↓          ↓          ↓
      Server 1   Server 2   Server 3
        ✅         ✅         ❌
        
L4 → Connection
L7 → HTTP Content

Round Robin       → Take turns
Weighted          → Powerful server gets more
Least Connection  → Least busy server
Health Check      → Don't send to unhealthy server
```

**One sentence to remember:**

> **"Load Balancer is the traffic police that distributes requests to healthy servers so the system can scale and remain available."**


Yes. These are **load-balancing strategies/routing methods**. The important distinction is that **L4/L7 are types based on the network/application layer**, while **Round Robin, Least Connection, Geo, Path, IP, etc. are routing algorithms or routing rules**.

A simple way to remember them:

> **Load Balancer = "How should I choose the backend server?"**

---

## 1. Random Load Balancing

The LB randomly selects a server.

```text
             Load Balancer
                  |
        ┌─────────┼─────────┐
        ↓         ↓         ↓
      Server 1  Server 2  Server 3
```

Example:

```text
Request 1 → Server 2
Request 2 → Server 1
Request 3 → Server 3
Request 4 → Server 2
```

### Remember

> **Random = Pick any server randomly.**

Simple, but it doesn't consider server capacity or current load.

---

# 2. Round Robin

Servers get requests one after another.

```text
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1
Request 5 → Server 2
```

### Remember

> **Round Robin = Take turns.**

Good when servers have approximately equal capacity.

---

# 3. Weighted Round Robin

Give each server a **weight** based on its capacity.

Example:

```text
Server 1 → Weight 5
Server 2 → Weight 3
Server 3 → Weight 2
```

Roughly:

```text
Server 1 → 50% traffic
Server 2 → 30% traffic
Server 3 → 20% traffic
```

### Remember

> **Weighted Round Robin = Powerful server gets more traffic.**

Useful when:

```text
Server 1 = 16 CPU
Server 2 = 8 CPU
Server 3 = 4 CPU
```

---

# 4. Least Connection

The LB sends a new request/connection to the server with the **fewest active connections**.

```text
Server 1 → 80 connections
Server 2 → 20 connections  ← New request
Server 3 → 50 connections
```

So:

```text
New request → Server 2
```

### Remember

> **Least Connection = Give work to the least busy server.**

Useful when requests have different processing times.

---

# 5. IP-Based / IP Hash

The client's IP is used to determine the backend.

```text
Client A
IP: 10.1.1.10
       ↓
     Hash
       ↓
   Server 2
```

Another request from the same IP will generally go to the same server.

```text
Client A → Server 2
Client A → Server 2
Client A → Server 2
```

### Why use it?

It can provide a form of **session affinity/stickiness**.

### Remember

> **IP Based = Same client IP tends toward same server.**

**Architect note:** Don't confuse this with authentication/session management. For modern stateless applications, externalizing session state is often preferable to relying on sticky routing.

---

# 6. Path-Based Routing

The LB looks at the **URL path** and sends traffic to different services.

```text
                 Load Balancer
                       |
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
    /users/*       /orders/*      /payments/*
        ↓              ↓              ↓
   User Service   Order Service   Payment Service
```

For example:

```text
GET /users/10
      ↓
User Service

GET /orders/100
      ↓
Order Service

GET /payments/50
      ↓
Payment Service
```

### Remember

> **Path Based = URL decides the destination.**

This is commonly associated with **Layer 7** routing because the LB needs to understand HTTP.

---

# 7. Geo-Based Routing

The LB/routing layer determines the user's geographic location and sends them to an appropriate region.

```text
                    Users
                      |
                 Geo Routing
              /      |       \
             ↓       ↓        ↓
           India    Europe    USA
             ↓       ↓        ↓
          Region-A Region-B Region-C
```

Example:

```text
User in India  → Mumbai region
User in Germany → Frankfurt region
User in USA → Virginia region
```

### Why?

* Lower latency
* Regional availability
* Data residency requirements
* Disaster recovery
* Traffic distribution across regions

### Remember

> **Geo Based = User location decides the region.**

---

# The easiest comparison

| Strategy             | Decision based on  | Easy memory                    |
| -------------------- | ------------------ | ------------------------------ |
| **Random**           | Random selection   | Pick randomly                  |
| **Round Robin**      | Turn/order         | Take turns                     |
| **Weighted RR**      | Server weight      | Powerful gets more             |
| **Least Connection** | Active connections | Least busy gets work           |
| **IP Based**         | Client IP          | Same IP → same server tendency |
| **Path Based**       | URL path           | `/orders` → Order service      |
| **Geo Based**        | User/location      | India → India region           |

---

# Very important: These are not all "types"

For an interview, organize them like this:

```text
                    LOAD BALANCER
                         │
          ┌──────────────┴──────────────┐
          │                             │
      Load Balancer                 Routing
        Layer/Type                  Strategies
          │                             │
      ┌───┴────┐             ┌──────────┼──────────┐
      │        │             │          │          │
     L4       L7         Round Robin   Least      IP
                                  │    Connection
                                  │
                            Weighted RR

                              +
                       Content/Location
                              │
                       ┌──────┴──────┐
                       │             │
                   Path Based    Geo Based
```

### So if interviewer asks:

**"What are different types of load balancing?"**

You can say:

> "There are two ways I classify them. At the infrastructure level, we have L4 and L7 load balancing. At the routing-strategy level, we can use Random, Round Robin, Weighted Round Robin, Least Connections, IP Hash, Path-Based routing, or Geo-Based routing depending on the workload and architecture."

### 🧠 Super-short memory trick

**R R W L I P G**

> **Random → Round Robin → Weighted → Least Connection → IP → Path → Geo**

And remember what each one asks:

```text
Random       → "Any server?"
Round Robin  → "Whose turn?"
Weighted     → "Who is more powerful?"
Least Conn.  → "Who is least busy?"
IP           → "Which client?"
Path         → "Which URL?"
Geo          → "Where is the user?"
```
