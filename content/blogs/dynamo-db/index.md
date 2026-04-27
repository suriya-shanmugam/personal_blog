+++
title = 'DynamoDB, Explained: How to Think in Access Patterns Instead of Tables'
date = 2026-04-26T18:53:16-07:00
draft = false
+++

When engineers first approach DynamoDB, the biggest mistake is treating it like a faster version of SQL.

It is not.

DynamoDB is a fully managed NoSQL database built for **fast, predictable performance at massive scale**. But to use it well, you have to change the way you think about data. In a relational database, you typically design normalized tables first and let SQL answer many kinds of questions later. In DynamoDB, you usually do the opposite: you start with the **queries your application needs**, and then design your data model around those access patterns. 

That shift is what makes DynamoDB both powerful and tricky. Used well, it can outperform traditional relational systems for the right workloads. Used poorly, it can become expensive, awkward, and hard to evolve.

## The DynamoDB mindset

At the heart of DynamoDB is the idea that your primary key is not just an identifier — it is the foundation of how your data is stored, distributed, and retrieved.

A DynamoDB table uses either a simple primary key made of a **partition key**, or a composite primary key made of a **partition key and sort key**. The partition key determines how data is distributed internally, and the sort key lets you organize related items within the same partition. This makes DynamoDB especially strong when your application naturally asks questions like: “get all orders for this customer,” “get the latest messages in this chat,” or “get all events for this device in time order.” 

This is why DynamoDB design is really **query-first design**.

Instead of asking, “What tables should I create?”, the better question is:

**What are the exact reads and writes my application performs most often?**

## Designing for queries, not relationships

In SQL, if you have customers and orders, you would typically create separate tables and join them when needed.

In DynamoDB, a better design is often to keep related records together under the same partition key, so they can be fetched efficiently with a single query. DynamoDB calls all items sharing the same partition key an **item collection**. This pattern works well for one-to-many relationships. 

Imagine an e-commerce system where one of the main features is:

* show all orders for a customer
* show recent orders for a customer
* show orders for a customer within a date range

A DynamoDB-friendly model could look like this:

```text
PK = CUSTOMER#123
SK = ORDER#2026-04-26T10:15:00Z#ORD789
```

With this structure, all of a customer’s orders live under the same partition key, and the sort key keeps them in time order. That makes queries efficient and natural:

* all orders for a customer
* latest N orders
* orders between two dates

This is a good example of a data model that fits DynamoDB’s architecture because the access pattern is known up front and the sort key does useful work. 

## Query is the happy path. Scan is the warning sign.

One of the most important DynamoDB concepts is the difference between **Query** and **Scan**.

A `Query` is efficient because it targets a specific partition key and can optionally narrow results using the sort key. A `Scan`, on the other hand, reads every item in a table or index. That means scans are often slow, expensive, and a sign that the data model does not match the application’s real access patterns. AWS’s own guidance is clear here: scans read all items, and even filter expressions are applied **after** data is read. 

That last point matters a lot.

Many developers assume this kind of query is efficient:

> “Read everything for a customer, then filter to only OPEN items.”

But if you read a large set and discard most of it afterward, you still pay for what was read. In DynamoDB, filtering does not rescue a weak primary-key design. 

## Secondary indexes: where GSI and LSI come in

Sooner or later, one table key is not enough.

You may model orders by customer, but then the product team asks for:

* show all open tickets assigned to Alice
* show all orders by status
* show all products by category
* show all messages sorted a different way

That is where secondary indexes enter the picture.

### Global Secondary Index (GSI)

A **GSI** gives you a completely different access path to your data. It can use a different partition key and a different sort key from the base table. DynamoDB keeps the index synchronized automatically, but updates happen asynchronously, so GSI reads are eventually consistent. 

This makes GSIs ideal when you need to query the same data from a totally different angle.

For example, suppose the base table stores tickets like this:

```text
PK = TENANT#42
SK = TICKET#9001
```

But your product also needs:

* all OPEN tickets assigned to Alice

You could add a GSI like this:

```text
GSI1PK = ASSIGNEE#alice
GSI1SK = STATUS#OPEN#2026-04-26T08:00:00Z
```

Now DynamoDB can answer a query that the base table was never designed for.

That is the real value of a GSI: it gives you a new, queryable view of the same underlying items. 

### Local Secondary Index (LSI)

An **LSI**, or local secondary index, is more constrained but useful in specific cases. It keeps the **same partition key as the base table** and only changes the sort key. LSIs let you query the same parent group in different sort orders or dimensions. They support strongly consistent reads, but they must be created when the table is created. Also, each item collection for a given partition key cannot exceed **10 GB** when LSIs are involved.

A good LSI use case is when the “parent” always stays the same, but you want multiple ways to sort that parent’s items.

For example, for a bank account you might want:

* transactions ordered by event time
* transactions ordered by settlement date
* transactions ordered by amount

If the partition key is always `ACCOUNT#555`, then an LSI is a natural fit because the account remains the same and only the sort dimension changes. 

### The simplest way to choose

A practical rule is:

* use an **LSI** when you want a different sort order **within the same partition key**
* use a **GSI** when you need a totally different lookup path across the table

That one distinction prevents a lot of confusion. 

## Where DynamoDB fits naturally

DynamoDB performs best when your workload has a few characteristics:

* very high request volume
* predictable and well-known access patterns
* key-based reads and writes
* low-latency requirements
* limited need for joins
* clear scaling expectations

This is why DynamoDB is commonly a great fit for things like session stores, shopping carts, device state, user preferences, game state, activity feeds, and time-ordered event streams. These systems usually care more about fast, predictable reads and writes than about complex relational queries. AWS describes DynamoDB as a fully managed NoSQL service with fast and predictable performance and seamless scalability, which lines up directly with these kinds of workloads. 

## A concrete case where DynamoDB can outperform SQL

A good example is a **session store** for a high-traffic consumer application.

Imagine millions of users opening an app and repeatedly doing simple operations like:

* get session by ID
* update cart quantity
* refresh expiration
* fetch current user state

This is close to DynamoDB’s ideal workload. The access pattern is simple, the keys are known, the reads and writes are small, and the application benefits from predictable low latency at scale. A relational database can absolutely support this, but doing so often requires more operational work around partitioning, caching, connection scaling, and query tuning. DynamoDB is built to absorb that pattern more directly. 

That does **not** mean DynamoDB is “better than SQL” in general.

It means DynamoDB is often better than SQL **for workloads that are mostly high-scale key lookups and targeted writes**, especially when joins are not central to the product.

## Architecture implications: why the model matters so much

Because DynamoDB distributes data using the partition key, the partition key is also your scaling strategy.

If too much traffic hits a small number of partition key values, you can create **hot partitions**. This is why a good key design needs enough cardinality and traffic spread. It is also why DynamoDB modeling is deeply tied to the architecture of the system, not just the shape of the data. 

This is different from SQL thinking. In relational design, schema structure is often separate from physical scaling decisions. In DynamoDB, the two are tightly connected.

That is one reason DynamoDB shows up so often in system design interviews: it forces you to think about storage layout, query patterns, and scale together.

## Advanced features that make DynamoDB more practical

DynamoDB is not just a simple key-value store. It also includes features that make it more usable in real systems.

It supports **transactions** for multi-item ACID operations, though with specific limits: up to **100 items** and **4 MB total** per transaction. It also supports expressions for conditional updates, filtering, and update logic. These features help when you need correctness beyond a single item, but they do not turn DynamoDB into a full relational engine. 

That distinction matters. Transactions help with correctness. They do not solve data modeling mistakes.

## The biggest limitations of DynamoDB

DynamoDB is powerful, but it comes with real tradeoffs.

The first is that it is much less friendly to **ad hoc querying** than SQL. If a new business requirement appears and your table was not designed for that access pattern, you may need a new GSI, additional denormalization, or even a redesign. 

The second is that DynamoDB does not support relational **joins** in the way SQL databases do. If your application naturally depends on joining many entities in flexible ways, a relational database will often be a better fit. 

The third is that bad key design can create operational pain. If a small number of partition keys absorb too much traffic, performance becomes uneven and the benefits of DynamoDB’s scale are harder to realize. 

There are also hard constraints to respect. DynamoDB has a maximum item size of **400 KB**, which means it is not a good place for large blobs or oversized documents. When items exceed that size, AWS recommends patterns like splitting data across items or storing large payloads in S3 and saving references in DynamoDB. 

And while indexes are powerful, they add tradeoffs too. GSIs introduce additional write and storage overhead and are eventually consistent. LSIs must be defined up front and come with the 10 GB item-collection limit per partition key. 

## So when should you use DynamoDB?

Use DynamoDB when:

* your access patterns are known in advance
* you need high throughput and low latency
* you are comfortable modeling data around queries
* your workload is dominated by key-based access
* you want a managed system that scales cleanly

Use SQL when:

* your queries evolve frequently
* joins are central to the application
* reporting and relational integrity are core requirements
* the shape of the data matters less than the flexibility of the query engine

The real lesson is not “DynamoDB vs SQL.”

The real lesson is that they solve different kinds of problems.

## Final thought

The best way to understand DynamoDB is to stop asking, “How do I map my SQL schema into DynamoDB?”

Instead, ask:

**What are the exact reads and writes my system performs, at what scale, and how can I model my keys so those operations are efficient by default?**

That is the DynamoDB mindset.

And once that clicks, concepts like partition keys, sort keys, GSIs, LSIs, and query-driven modeling stop feeling like limitations. They start feeling like tools for building systems that are simple, fast, and scalable by design.


### Reference
- [Hello Interview](https://www.youtube.com/watch?v=2X2SO3Y-af8)
- [AWS - Documentation](https://docs.aws.amazon.com/dynamodb/)

[1]: https://www.youtube.com/watch?v=2X2SO3Y-af8&utm_source=chatgpt.com "DynamoDB Deep Dive w/ a Ex-Meta Staff Engineer"
[2]: https://docs.aws.amazon.com/dynamodb/?utm_source=chatgpt.com "Amazon DynamoDB Documentation"
[3]: https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Query.html?utm_source=chatgpt.com "Query - Amazon DynamoDB"
[4]: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Scan.html?utm_source=chatgpt.com "Scanning tables in DynamoDB - docs.aws.amazon.com"
[5]: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html?utm_source=chatgpt.com "Using Global Secondary Indexes in DynamoDB - docs.aws.amazon.com"
[6]: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/LSI.html?utm_source=chatgpt.com "Local secondary indexes - Amazon DynamoDB"
[7]: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html?utm_source=chatgpt.com "Amazon DynamoDB Transactions: How it works"
[8]: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SecondaryIndexes.html?utm_source=chatgpt.com "Improving data access with secondary indexes in DynamoDB"
[9]: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Constraints.html?utm_source=chatgpt.com "Constraints in Amazon DynamoDB - Amazon DynamoDB - docs.aws.amazon.com"
