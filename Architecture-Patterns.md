# Different System Design Patterns — Interview Guide

For a Senior, Lead, or Staff Software Engineer interview, it is important to understand that system design patterns solve different problems at different levels. They help you design systems that are scalable, resilient, maintainable, and loosely coupled.

A useful way to organize them is into six categories.

## 1. Architectural patterns

These define the overall structure of an application and how its components depend on one another.

[AI-Driven App Architectures: Avoid Spaghetti Code | Medium](https://images.openai.com/static-rsc-4/8pmFvogyRqRy56YvYbFd-P85pZ1azH91xl5M8QxZdOKz1V23kerRxwoon_oN0BYB5029SmRGVT1MjIMctb0QFTGOcJMH0XqwrVXP0QwYO_-GJ5pDY7oKFYXpyE5xUKJ_nZ4KCEnxnCq55mFpuiVKYl9fijtGmGjVFaRLlH-XPeo?purpose=inline)

1\. Layered Architecture

Separates presentation, business logic, and data access layers.

Example: ASP.NET Core Web API with Controllers, Services, Repositories, and SQL Server.

[Clean Architecture for Enterprise Applications: A Practical Guide from the Trenches - DEV Community](https://images.openai.com/static-rsc-4/lwBk7w6Qq-ugaa2sm3BH8UYDZhYFbJJnP-Hjtmj-6HJv4lx_MKTJfP3ijQmIokOqLs8PbpHdaiq8VAArtlWlETKk_mmfq67oDKKoeDVE_KMWfZ4wJhuPi5hBUG6-_om37gcEUik4dwSgHM5jm1ZLjUFqbq6eq8_tpEJezeR0mTM?purpose=inline)

2\. Clean / Onion Architecture

Keeps the domain and business rules independent of infrastructure.

Example: Domain, Application, Infrastructure, and API projects in .NET.

[Monolithic vs. Microservices: The Great Architecture Debate | Last9](https://images.openai.com/static-rsc-4/bwoihYOyVxra575pj6OPgPYZYYGDf6K_MmJNWSxlXUwnbjcnU3xIOa5xRx5q1-DuWa7YGbtyaZRjrhgrypQrfVkWIErOj8-ABR6ThLOjCiL2v6-nQSur1YamlApHWPfaIi_LolY4JKSxER0zikizJtehYc7K3x_RG2bpf-2e4PU?purpose=inline)

3\. Microservices Architecture

Splits a system into independently deployable business services.

Example: Order, Payment, Inventory, and Notification services.

[A Guide To Event-Driven Architectural Patterns](https://images.openai.com/static-rsc-4/fiG9PxFZr-EaO3d9cqRiryLWsv3wo1rCgtiqtluaV--R3XNkzYZxwubV6aoA-FpK5A39EL-DLRiCcomvR8i1kuXG1PlFStIAcfLsSpStsbu80Cs7QFRlWJLa_DiobrEN_-Kp4xOwL9jXerYVlsuLF88BGnil1HetR4vfPUuq8X4?purpose=inline)

4\. Event-Driven Architecture (EDA)

Services communicate by publishing and consuming events.

Example: An OrderPlaced event triggers inventory reservation and email notification through Azure Service Bus.

[Hexagonal Architecture in Software Development | by Lubos Maxina | Jamf Engineering | Medium](https://images.openai.com/static-rsc-4/HdLhMWj0p_eMTkzKOw0qDdj2iGwnCxARzeF1e1O0ZwoT086nOciezvZrxV21iejxD1TDSrgtKR2dKa_Nomrk14GKcls3lQ-bO0Ybj2TmxfM9IhxU4ZD6gbiDjbuRr5ZIiqi7P65K8n_7xXt9_uXsB1xhKTvPjKhtt9PXzz-OAbI?purpose=inline)

5\. Hexagonal Architecture (Ports and Adapters)

Keeps the core application isolated from external systems through interfaces.

Example: Replace SQL Server with another database without changing core business rules.

[Dhanian 🗯️ (@e_opore) on X](https://images.openai.com/static-rsc-4/SBW-BEYZwZj4W8ridBpMbnyaQhIOKYSt3lpAHayf3YNbbMdTi1zBAMCiat0CMU4Yz313ZNeQ4qwdDN6rMG4qyU9W7ji0Yk3umQNOb0CCHxg0tybZlOY5VRGlz3MtEQ8c9mZA2eMnRwgmvazXRfkJH_9l5TmE28oVO-W_mtsWmmc?purpose=inline)

6\. Client–Server Architecture

Separates clients that request functionality from servers that provide it.

Example: Angular frontend calling an ASP.NET Core Web API.

Other architectural styles include monolithic architecture, modular monolith, service-oriented architecture (SOA), serverless architecture, and space-based architecture.

## 2. Distributed systems and resilience patterns

These patterns address failures, high traffic, slow dependencies, and communication across services.

| Pattern         | Problem it solves                           | Real-world example                                                  |
| --------------- | ------------------------------------------- | ------------------------------------------------------------------- |
| Circuit Breaker | Repeated calls to a failing dependency      | Stop calling an unavailable payment API temporarily                 |
| Retry           | Transient failures                          | Retry a request after a temporary network error                     |
| Timeout         | A dependency takes too long                 | Cancel a payment API call after a configured duration               |
| Bulkhead        | One failing workload affects everything     | Separate thread pools or concurrency limits for critical operations |
| Rate Limiting   | Too many requests                           | Limit API requests per user or client                               |
| Load Balancing  | Traffic concentrated on one instance        | Distribute API traffic across multiple instances                    |
| Health Check    | Detect unhealthy instances                  | Remove unhealthy API instances from service                         |
| Fallback        | Dependency is unavailable                   | Return cached or degraded data when appropriate                     |
| Backpressure    | Consumers cannot keep up with incoming work | Slow message consumption or limit concurrent processing             |
| Idempotency     | Duplicate requests cause duplicate effects  | Prevent duplicate payments using an idempotency key                 |

Interview scenario: An external payment service becomes slow, and thousands of requests start accumulating.

A good design might use a timeout, a circuit breaker, bounded concurrency, and carefully configured retries. For payment operations, use idempotency to avoid duplicate charges. Each pattern addresses a different failure mode.

## 3. Data management and consistency patterns

These patterns help manage data across services, improve read performance, and maintain consistency.

| Pattern                   | What it means                                      | Example                                                                |
| ------------------------- | -------------------------------------------------- | ---------------------------------------------------------------------- |
| CQRS                      | Separate read and write models                     | Order writes in SQL; optimized order queries use a separate read model |
| Saga                      | Coordinate a business transaction across services  | Reserve inventory → charge payment → confirm order                     |
| Transactional Outbox      | Reliably publish events alongside database changes | Save an order and an outgoing event in one database transaction        |
| Database per Service      | Each microservice owns its data                    | Inventory and Payment maintain separate databases                      |
| Event Sourcing            | Store changes as a sequence of events              | Account balance derived from deposit and withdrawal events             |
| Materialized View         | Precompute data for fast reads                     | Dashboard showing order totals                                         |
| Cache-Aside               | Load data into cache on demand                     | Retrieve products from Redis or SQL Server                             |
| Read Replica              | Offload database reads                             | Direct reporting queries to a replica                                  |
| Change Data Capture (CDC) | Capture database changes for downstream processing | Publish database changes to a search index                             |
| Distributed Lock          | Coordinate access to a shared resource             | Ensure only one worker performs a particular scheduled operation       |

### CQRS vs Saga vs Transactional Outbox

These three are often confused in interviews.

- CQRS: How do we organize reads and writes?
- Saga: How do we coordinate a multi-service business transaction without one global database transaction?
- Transactional Outbox: How do we avoid committing database data but losing the corresponding event?

For example, in a .NET order-processing application, the order service can save an order and an outbox record in the same SQL transaction. A background publisher sends the event to Azure Service Bus. A Saga coordinates the subsequent inventory and payment operations.

## 4. Communication and integration patterns

| Pattern                    | Purpose                                               | Example                                                    |
| -------------------------- | ----------------------------------------------------- | ---------------------------------------------------------- |
| API Gateway                | Single entry point for client requests                | Azure API Management                                       |
| Backend for Frontend (BFF) | Tailor APIs to each client                            | Separate mobile and web APIs                               |
| Request–Reply              | Request and receive a response                        | HTTP API call                                              |
| Publish–Subscribe          | One event reaches multiple subscribers                | `OrderPlaced` triggers billing and notifications           |
| Competing Consumers        | Multiple workers process messages from a shared queue | Multiple Azure Service Bus consumers                       |
| Message Router             | Route messages based on content or rules              | Route payment events by payment type                       |
| Anti-Corruption Layer      | Protect one domain from another system's model        | Translate a legacy API's data model into your domain model |
| Strangler Fig              | Gradually replace a legacy application                | Route selected legacy features to new microservices        |
| Sidecar                    | Run supporting functionality alongside an application | Proxy or telemetry agent alongside a service               |

Important distinction: Publish–subscribe broadcasts an event to multiple subscribers, whereas competing consumers typically share a queue or subscription workload to distribute processing.

## 5. Scalability and performance patterns

| Pattern                   | Purpose                                                 | Example                                |
| ------------------------- | ------------------------------------------------------- | -------------------------------------- |
| Horizontal Scaling        | Add more instances                                      | Scale an API from 2 to 10 instances    |
| Vertical Scaling          | Increase machine resources                              | Increase CPU or RAM                    |
| Caching                   | Reduce repeated database or service calls               | Redis cache                            |
| CDN                       | Serve static content closer to users                    | Images and JavaScript files            |
| Sharding                  | Split data across multiple database partitions or nodes | Customers distributed by tenant ID     |
| Partitioning              | Divide a large dataset into manageable parts            | SQL tables partitioned by date         |
| Queue-Based Load Leveling | Buffer bursts of work                                   | Azure Service Bus queue                |
| Batch Processing          | Process records in groups                               | Bulk invoice generation                |
| Prefetching               | Retrieve work or data before it is needed               | Message consumer prefetch              |
| Data Locality             | Keep frequently used data close to the compute workload | Regional data replicas or local caches |

### Example: Handling 10,000 incoming requests

Suppose your .NET API suddenly receives 10,000 requests.

1. Use a load balancer to distribute incoming traffic.
2. Apply rate limiting to protect the API.
3. Use Redis to reduce repeated database queries.
4. Put long-running work into Azure Service Bus.
5. Scale consumers horizontally while keeping concurrency bounded.
6. Use database indexes and connection-pool monitoring.
7. Apply timeouts and circuit breakers to protect downstream dependencies.

The right combination depends on whether the bottleneck is CPU, database I/O, an external API, or background processing.

## 6. Deployment, migration, and security patterns

| Pattern                    | Purpose                                              | Example                                           |
| -------------------------- | ---------------------------------------------------- | ------------------------------------------------- |
| Blue-Green Deployment      | Switch between two production environments           | Deploy a new API version and switch traffic       |
| Canary Release             | Gradually expose a new version                       | Send 5% of traffic to the new service             |
| Feature Toggle             | Enable or disable functionality without redeployment | Enable a new checkout flow for selected users     |
| Zero-Downtime Migration    | Upgrade systems without a planned outage             | Expand-and-contract database schema changes       |
| Zero Trust                 | Verify every access request                          | Validate identity, permissions, and context       |
| Defense in Depth           | Apply multiple security controls                     | WAF + authentication + authorization + encryption |
| Token-Based Authentication | Authenticate requests using tokens                   | OAuth 2.0 / OpenID Connect with JWT access tokens |
| Secrets Management         | Secure credentials and keys                          | Azure Key Vault with managed identity             |

## 7. How these patterns fit together in a real system

Consider an e-commerce system implemented using .NET, Azure, SQL Server, and Redis.

Angular / Mobile Client

API Gateway

Authentication · Rate limiting · Routing

Order Service

CQRS · SQL Server

Redis Cache

Cache-aside

Transactional Outbox + Azure Service Bus

Reliable event publishing · Queue-based load leveling

Inventory Service

Saga participant

Payment Service

Idempotency · Retry

Cross-cutting resilience: timeouts, circuit breakers, monitoring, and health checks.

This architecture illustrates how several patterns work together rather than being used in isolation. A production design would also address event duplication, dead-letter queues, authorization between services, observability, and compensating actions when a Saga fails.

## 8. Which patterns should you prioritize for Staff Engineer interviews?

Given your .NET and Azure background, I recommend mastering these first:

Priority 1 — Architecture

Microservices vs modular monolith, Clean Architecture, event-driven architecture, CQRS, API Gateway, and database-per-service.

Priority 2 — Reliability

Circuit Breaker, Retry, Timeout, Bulkhead, Idempotency, Transactional Outbox, Saga, and Dead Letter Queue.

Priority 3 — Scalability

Redis caching, load balancing, partitioning vs sharding, rate limiting, queue-based load leveling, and backpressure.

Priority 4 — Evolution and delivery

Strangler Fig, Blue-Green Deployment, Canary Release, Feature Toggles, and Anti-Corruption Layer.

One final distinction: system design patterns are not the same as GoF object-oriented design patterns. Factory, Strategy, Adapter, Decorator, and Mediator are typically used to structure code within components. The patterns above primarily address application architecture, distributed communication, reliability, scalability, and deployment.

For every pattern, prepare to explain four things in an interview: the problem, how the pattern works, its trade-offs, and when you would not use it. That demonstrates architectural judgment rather than simply memorizing definitions.&#x20;
