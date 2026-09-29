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


--------------------------------------------L4/L7------------------------------------------------------------------------

The key idea is:

> **The Load Balancer doesn't guess whether a request is TCP, UDP, or HTTP. It knows from the network protocol/connection information and from what it is configured to handle.**

### Think in layers

```text
Application
    │
    │ HTTP
    ▼
Transport
    │
    │ TCP / UDP
    ▼
Internet
    │
    │ IP
    ▼
Network
```

For a typical HTTPS request:

```text
Browser
   │
   │ HTTP request
   ▼
HTTPS
   │
   │ TLS
   ▼
TCP
   │
   │ IP packet
   ▼
Load Balancer
```

So **HTTP is carried over TCP** in the traditional HTTP/1.1 and HTTP/2 model.

---

## 1. How does L4 know?

An L4 load balancer looks at the **IP packet and transport header**.

For example:

```text
Source IP       = 10.1.1.20
Destination IP  = 20.10.10.5
Protocol        = TCP
Source Port     = 52341
Destination Port= 443
```

The `Protocol` field tells the network layer that this is TCP.

So the LB sees:

```text
IP
 └── Protocol = TCP
       └── Port = 443
```

It can then forward the TCP connection to:

```text
Server 1
Server 2
Server 3
```

It doesn't need to understand:

```http
GET /orders/123
```

That's why we call it **Layer 4 load balancing**.

---

# 2. What about UDP?

Suppose you have a UDP packet:

```text
Source IP        = 10.1.1.20
Destination IP   = 20.10.10.5
Protocol         = UDP
Source Port      = 50000
Destination Port = 53
```

The LB sees:

```text
IP
 └── Protocol = UDP
       └── Port = 53
```

So it knows:

> "This is a UDP flow."

It can route the UDP traffic accordingly.

---

# 3. How does L7 know it's HTTP?

This is where it gets interesting.

An **L7 load balancer understands the application protocol**.

For HTTP/1.1, it can see something like:

```http
GET /orders/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer ...
```

Now it can make decisions such as:

```text
/orders/*    → Order Service
/users/*     → User Service
/payments/*  → Payment Service
```

So:

```text
L4:
"TCP connection → Server 2"

L7:
"HTTP request /orders → Order Service"
```

---

# 4. But HTTPS is encrypted — how can L7 see HTTP?

Excellent architectural point.

With HTTPS:

```text
Client
   │
   │ encrypted HTTPS
   ▼
Load Balancer
```

The HTTP contents are encrypted.

Therefore, if the LB needs to perform L7 routing, it commonly performs **TLS termination**:

```text
Client
   │
   │ HTTPS
   ▼
┌──────────────────┐
│ Load Balancer    │
│ TLS termination  │
└────────┬─────────┘
         │
         │ HTTP or HTTPS
         ▼
      Backend
```

The LB decrypts the request, allowing it to inspect:

```text
HTTP method
URL path
Headers
Host
Cookies
```

For example:

```text
GET /orders/123
        │
        ▼
Load Balancer
        │
        └── /orders/* → Order Service
```

It can then establish a new connection to the backend.

---

# 5. What about HTTP/2 and HTTP/3?

This is where the architecture gets more interesting.

### HTTP/1.1

Typically:

```text
HTTP
 ↓
TCP
 ↓
IP
```

### HTTP/2

Typically:

```text
HTTP/2
 ↓
TCP
 ↓
IP
```

### HTTP/3

HTTP/3 uses **QUIC**, which runs over UDP:

```text
HTTP/3
 ↓
QUIC
 ↓
UDP
 ↓
IP
```

So don't memorize:

> "HTTP always means TCP."

Instead remember:

> **HTTP is an application-layer protocol. TCP/UDP are transport-layer protocols.**

---

# 6. Very important distinction

This is the part I would remember for your Staff/Architect interview:

```text
             Application
                  │
             HTTP / HTTP/2
                  │
          ┌───────┴───────┐
          │               │
         TCP             QUIC
          │               │
          │              UDP
          │               │
          └───────┬───────┘
                  ▼
                 IP
```

So:

### L4 LB asks:

> **"What transport protocol/connection is this?"**

```text
TCP?
UDP?
Port?
IP?
```

### L7 LB asks:

> **"What application request is this?"**

```text
HTTP?
GET /orders?
Host?
Headers?
Cookie?
```

---

# 7. Real example

Suppose your application receives:

```text
https://api.company.com/orders/123
```

The traffic might look conceptually like:

```text
                 Browser
                    │
                    ▼
              HTTPS request
                    │
                    ▼
               TCP connection
                 Port 443
                    │
                    ▼
             ┌─────────────┐
             │ Load Balancer│
             └─────────────┘
```

### L4 LB

Sees:

```text
TCP
Port 443
```

and says:

> "I'll send this TCP connection to API Server 2."

### L7 LB

After TLS termination, sees:

```http
GET /orders/123
Host: api.company.com
```

and says:

> "This is an orders request. Send it to Order Service."

---

## 🧠 One picture to remember

```text
                  REQUEST
                     │
                     ▼
              ┌─────────────┐
              │ Load Balancer│
              └──────┬──────┘
                     │
             What can I see?
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
       L4 LB                  L7 LB
          │                     │
    IP / TCP / UDP          HTTP details
    Port                    Path
                            Host
                            Headers
          │                     │
          ▼                     ▼
   "Which connection?"   "Which application?"
```

### Interview one-liner

> **"L4 determines routing using network and transport information such as IP, TCP/UDP and ports, while L7 can inspect the application protocol such as HTTP and route based on paths, hosts, headers or other application-level information."**

And the crucial point:

> **The LB knows TCP/UDP from the IP/transport headers; it knows HTTP when it is operating at L7 and can parse the application protocol, often after TLS termination for HTTPS.**
