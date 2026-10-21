# Azure Table Storage vs SQL — Staff Engineer Interview Notes

The most important interview point is this: We don't choose Azure Table Storage simply because it is NoSQL or because it is cheaper. We choose it when our access patterns, data model, scalability needs, and consistency requirements fit a key-value store better than a relational database.

Let's use an e-commerce application as our running example.

## 1. What is Azure Table Storage?

Azure Table Storage is a fully managed NoSQL data store for large volumes of structured, non-relational data.

It stores entities using:

- PartitionKey: Groups entities into logical partitions and influences scalability.
- RowKey: Uniquely identifies an entity within a partition.
- Properties: Flexible, entity-specific fields.
- ETag: Supports optimistic concurrency.

Example: storing customer activity records.

| PartitionKey | RowKey | Activity    | ProductId |
| ------------ | ------ | ----------- | --------- |
| C101         | A001   | Viewed      | P101      |
| C101         | A002   | AddedToCart | P101      |
| C102         | A003   | Viewed      | P205      |

If we know the partition and row keys, we can retrieve an entity efficiently without performing a relational join.

## 2. Why choose Table Storage over Azure SQL?

1\. Scale for high-volume key-based workloads

Table Storage can handle large datasets without you managing database servers. With good partition design, it can distribute traffic across partitions.

Example: Recording millions of lightweight customer activity or job-status records.

2\. Cost-effective storage

Standard Table Storage can be economical when records are simple and access patterns are predictable.

Example: Retaining lightweight activity records instead of paying for relational database capabilities you don't need.

3\. Flexible schema

Different entities can have different properties without requiring a fixed relational schema.

Example: Activity records may contain `ProductId`, while a login event contains `Browser` and `DeviceType`.

4\. Fewer relational features to manage

There are no relational joins or foreign-key constraints to design around. That simplifies the storage model when entities are naturally independent.

Trade-off: Your application must handle relationships and cross-entity consistency where needed.

Important: Azure SQL can also scale significantly. Table Storage is not automatically faster or cheaper for every workload; compare actual query patterns, throughput, storage, availability, and operational requirements.

## 3. SQL vs Table Storage: practical comparison

| Feature      | Azure Table Storage                                       | Azure SQL                                       |
| ------------ | --------------------------------------------------------- | ----------------------------------------------- |
| Data model   | NoSQL entities                                            | Relational tables                               |
| Schema       | Flexible                                                  | Defined schema                                  |
| Indexing     | Primary index on `PartitionKey` + `RowKey`                | Multiple indexes supported                      |
| Joins        | Not supported                                             | Supported                                       |
| Transactions | Within a single partition using entity group transactions | Rich ACID transactions                          |
| Scaling      | Partition-based distribution                              | Vertical and horizontal scaling options         |
| Cost         | Often economical for simple access patterns               | Depends on compute, storage, and service tier   |
| Best for     | High-volume key-based records                             | Complex queries and transactional business data |

Table Storage's key-based query model and partition design are documented in Microsoft's [Table Storage design guidance](https://learn.microsoft.com/en-us/azure/storage/tables/table-storage-design-guidelines?utm_source=chatgpt.com).&#x20;

[image](https://www.google.com/s2/favicons?domain=https://learn.microsoft.com\&sz=32)

Microsoft Learn

+1



## 4. When would I choose each one?

Choose Table Storage

E-commerce browsing history, device metadata, simple job tracking, or records retrieved by known keys.

Choose Azure SQL

Orders, payments, inventory, financial transactions, and reporting that require joins, constraints, or multi-row transactions.

### Example: Order tracking

Suppose an e-commerce application stores 50 million order-status history events.

In Table Storage, you might design:

| PartitionKey | RowKey            | Status    |
| ------------ | ----------------- | --------- |
| `Order1001`  | `20261021T100000` | Created   |
| `Order1001`  | `20261021T100500` | Confirmed |
| `Order1001`  | `20261021T101000` | Shipped   |

This supports efficient retrieval of an order's history when the partition key is known. The `RowKey` can be designed to provide chronological ordering, using a fixed-width UTC timestamp and a tie-breaker if multiple events may share the same timestamp.

In SQL, you would typically use an indexed `OrderId` and timestamp column. SQL may be preferable if the application also frequently joins status history to customers, products, shipments, and payments.

Important: 50 million records alone doesn't justify Table Storage. Query patterns, write rate, latency, consistency, and cost determine the right choice.

## 5. Interview questions to remember

Q1. Does Table Storage have indexes?

It has a primary index based on `PartitionKey` and `RowKey`, but no automatic secondary indexes on arbitrary properties. Queries filtering only on other properties can require scans.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://learn.microsoft.com\&sz=32)

Microsoft Learn

+1



Q2. What is a hot partition?

A partition receiving disproportionate traffic. It can become a throughput bottleneck. Distribute traffic through an appropriate partition-key strategy.

Q3. How do you prevent concurrent updates?

Use ETags for optimistic concurrency. If an entity changes after it was read, a conditional update can fail with HTTP 412.

Q4. Can Table Storage replace SQL completely?

No. It can replace SQL for particular workloads, but it isn't a general-purpose relational database.

Q5. How would you design Table Storage for millions of customers?

Identify dominant queries, choose partition and row keys, avoid hot partitions, use idempotent writes and bounded retries, and load-test realistic traffic distributions.

## 6. The 30-second interview answer

> I would choose Azure Table Storage over Azure SQL when the workload involves large volumes of relatively simple data, predictable key-based access, and a need for cost-effective partition-based scaling. For example, I could use it for e-commerce activity tracking or order-status history. I would choose Azure SQL for orders, payments, and inventory where relational integrity, complex queries, and transactions are important. Before selecting Table Storage, I would validate the access patterns and partition-key design because it doesn't offer SQL-style joins or secondary indexes on arbitrary properties.
