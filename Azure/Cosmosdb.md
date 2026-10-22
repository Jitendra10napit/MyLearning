**30-second intro**
"Azure Cosmos DB is Microsoft's fully managed, globally distributed NoSQL database. It stores data as JSON documents, guarantees single-digit-millisecond latency, scales horizontally without limits, and replicates across Azure regions with a few clicks. It supports multiple APIs, including NoSQL, MongoDB, Cassandra, Gremlin, and Table. In .NET we mainly use the NoSQL API through the `Microsoft.Azure.Cosmos` SDK."

**Key points to mention**
- **Partition key:** "The most important design decision. It decides how data is distributed, so I choose a high-cardinality field that appears in most queries, like `tenantId` or `userId`."
- **Request Units:** "Throughput is measured in RUs. A 1 KB point read costs about 1 RU. We can choose provisioned, autoscale, or serverless depending on the traffic pattern."
- **Consistency:** "It offers five consistency levels, from Strong to Eventual. Session is the default and fits most apps."
- **Indexing:** "Every property is indexed automatically, and we tune it with an indexing policy to save RUs."
- **Change Feed:** "It gives a stream of changes, which is great for event-driven microservices with Azure Functions."
- **.NET best practices:** "A singleton `CosmosClient`, point reads over queries, avoiding cross-partition queries, Managed Identity instead of keys, and watching `RequestCharge`."

**Why choose Cosmos DB**
- Guaranteed low latency at any scale, backed by SLAs
- Elastic horizontal scaling, with no sharding work for us
- Global distribution with multi-region writes
- A flexible schema that fits fast-evolving models
- Fully managed, so no patching or index tuning

**When to choose it**
- High-volume, read- and write-heavy workloads: IoT telemetry, user profiles, shopping carts, product catalogs
- Multi-tenant SaaS apps where data partitions naturally by tenant
- Globally distributed apps that need users served from the nearest region
- Event-driven microservices using the Change Feed
- Data whose shape varies or changes often

**When NOT to choose it (shows maturity)**
- Complex relationships, multi-table joins, or heavy reporting go to Azure SQL.
- Transactions spanning many entities, as in financial or accounting systems, also fit Azure SQL better, because Cosmos guarantees ACID only within one partition.
- Small apps with unpredictable ad-hoc queries can get expensive in RUs.

**Closing line**
"With Cosmos DB, we design around access patterns and the partition key first. I'd choose it for scale, low latency, and global reach, and I'd stick with SQL where relational integrity and complex querying matter. In many systems the two work best together."


Here's an example you can describe in the interview, using an e-commerce order system.

**Scenario to speak**
"Say we're building an e-commerce platform with millions of customers worldwide. Customers mostly view their own orders and cart, traffic spikes during sales, and users are spread across regions. That's a good fit for Cosmos DB."

**Design decisions to explain**
- **Container:** `Orders`
- **Partition key:** `/customerId`. "Every order query filters by customer, so all of a customer's orders sit in one partition and the reads are fast and cheap."
- **Data model:** "I embed the line items and shipping address inside the order document instead of using separate tables, so a single read returns the full order. No joins needed."

```json
{
  "id": "ord-1001",
  "customerId": "cust-42",
  "status": "Placed",
  "total": 2499.00,
  "items": [
    { "productId": "p-11", "name": "Headphones", "qty": 1, "price": 1999 },
    { "productId": "p-27", "name": "Cable", "qty": 2, "price": 250 }
  ],
  "shippingAddress": { "city": "Pune", "pin": "411001" }
}
```

**.NET code**
```csharp
// Registered once as a singleton
builder.Services.AddSingleton(_ =>
    new CosmosClient(endpoint, new DefaultAzureCredential()));

public class OrderService(CosmosClient client)
{
    private readonly Container _orders = client.GetContainer("ShopDb", "Orders");

    public Task PlaceOrder(Order order) =>
        _orders.CreateItemAsync(order, new PartitionKey(order.CustomerId));

    // Point read: ~1 RU, the cheapest operation
    public async Task<Order> GetOrder(string id, string customerId) =>
        (await _orders.ReadItemAsync<Order>(id, new PartitionKey(customerId))).Resource;
}
```

**Change Feed**
"When an order is placed, the Change Feed triggers an Azure Function that sends the confirmation email and updates inventory. The order API stays fast and decoupled."

**Why not SQL here**
"Order reads are key-based and high-volume, the schema varies (digital vs physical products), and we need global low latency. Payments and accounting stay in Azure SQL, though, because they need ACID transactions across entities. So the system uses both."

**Quick one-liner examples** (for when they ask "where else?")
- **IoT:** device telemetry partitioned by `/deviceId`, with high write volume.
- **Multi-tenant SaaS:** partitioned by `/tenantId` for isolation and even scaling.
- **User profiles and sessions:** partitioned by `/userId`, served from the nearest region.
- **Product catalog:** a flexible schema, since a laptop and a T-shirt have different attributes.

**A bad partition key to call out**
"I'd avoid something like `/status`, because there are only a few values like Placed or Shipped. That creates hot partitions and throttling (429s)."
