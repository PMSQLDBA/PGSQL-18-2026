## Topic 20 — PostgreSQL Partitioning

PostgreSQL partitioning is mainly used for **very large tables** where data naturally splits by time, category, tenant, region, or another predictable key. 

The main benefits are faster access to relevant subsets of data and much easier lifecycle operations such as archiving or deleting old data.

### 1. Basic architecture

A partitioned table is the **parent**. The actual rows live in its **partitions**.

```sql
CREATE TABLE orders (
    order_id     bigint,
    customer_id  bigint,
    order_date   date NOT NULL,
    status       text,
    amount       numeric(12,2)
) PARTITION BY RANGE (order_date);
```

Think of it as:

```text
orders
   |
   +-- orders_2026_01
   +-- orders_2026_02
   +-- orders_2026_03
   +-- ...
```

Applications normally query the parent:

```sql
SELECT *
FROM orders
WHERE order_date >= DATE '2026-08-01'
  AND order_date <  DATE '2026-09-01';
```

PostgreSQL determines which partitions need to be accessed.

---

## 2. Range partitioning

**Range partitioning** is the most common approach for time-series and historical data.

Good partition keys include:

* transaction date
* event timestamp
* created date
* accounting period

Example:

```sql
CREATE TABLE orders_2026_07
PARTITION OF orders
FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

CREATE TABLE orders_2026_08
PARTITION OF orders
FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');

CREATE TABLE orders_2026_09
PARTITION OF orders
FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');
```

Notice that the upper boundary is **exclusive**.

```text
July partition:
2026-07-01 <= order_date < 2026-08-01

August partition:
2026-08-01 <= order_date < 2026-09-01
```

This avoids overlapping partitions.

Range partitioning is usually the first option to consider when data has a clear lifecycle.

---

## 3. List partitioning

List partitioning works well when rows belong to a relatively small number of known categories.

For example:

```sql
CREATE TABLE customers (
    customer_id bigint,
    name        text,
    region      text NOT NULL
) PARTITION BY LIST (region);
```

Create partitions:

```sql
CREATE TABLE customers_us
PARTITION OF customers
FOR VALUES IN ('US');

CREATE TABLE customers_eu
PARTITION OF customers
FOR VALUES IN ('DE', 'FR', 'IT', 'ES');

CREATE TABLE customers_apac
PARTITION OF customers
FOR VALUES IN ('IN', 'JP', 'SG', 'AU');
```

Typical uses include:

```text
Region
Country
Business unit
Tenant group
Status/category
```

It is less suitable when the possible values are constantly changing or extremely numerous.

---

## 4. Hash partitioning

Hash partitioning distributes rows approximately evenly across a fixed number of partitions.

```sql
CREATE TABLE customer_events (
    event_id    bigint,
    customer_id bigint NOT NULL,
    payload     jsonb
) PARTITION BY HASH (customer_id);
```

For four partitions:

```sql
CREATE TABLE customer_events_p0
PARTITION OF customer_events
FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE customer_events_p1
PARTITION OF customer_events
FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE customer_events_p2
PARTITION OF customer_events
FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE customer_events_p3
PARTITION OF customer_events
FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

Conceptually:

```text
hash(customer_id)
        |
        v
   hash % 4
        |
 +------+------+------+------+
 |  P0  |  P1  |  P2  |  P3  |
 +------+------+------+------+
```

Hash partitioning is useful when you want relatively even distribution but there is no meaningful date range or category boundary.

Its downside is lifecycle management. You cannot simply say, "drop everything older than January," because old and new rows may exist across all hash partitions.

---

## 5. Choosing the partition type

| Requirement                | Usually choose |
| -------------------------- | -------------- |
| Time-series data           | Range          |
| Historical transactions    | Range          |
| Logs/events                | Range          |
| Country/region             | List           |
| Small number of categories | List           |
| Even distribution          | Hash           |
| No natural ranges          | Hash           |
| Easy retention/deletion    | Range          |
| Tenant isolation           | List or Hash   |

For enterprise systems, **range partitioning by time** is especially common because it solves both performance and lifecycle problems.

---

# 6. Partition pruning

Partition pruning is one of the most important concepts.

Suppose there are 120 monthly partitions representing ten years of data.

The application runs:

```sql
SELECT *
FROM orders
WHERE order_date >= DATE '2026-08-01'
  AND order_date <  DATE '2026-09-01';
```

PostgreSQL can recognize that only:

```text
orders_2026_08
```

contains matching rows.

Instead of scanning:

```text
120 partitions
```

it may only need:

```text
1 partition
```

That is **partition pruning**.

Check it using:

```sql
EXPLAIN
SELECT *
FROM orders
WHERE order_date >= DATE '2026-08-01'
  AND order_date < DATE '2026-09-01';
```

You want the plan to show access only to the relevant partitions.

### Query design matters

This is partition-friendly:

```sql
WHERE order_date >= DATE '2026-08-01'
  AND order_date < DATE '2026-09-01'
```

Be careful about unnecessarily transforming the partition key:

```sql
WHERE some_function(order_date) = ...
```

Expressions, casts, and application query patterns can sometimes make it harder for PostgreSQL to eliminate partitions efficiently.

A useful architecture rule is:

> Design the partition key around the application's most important filtering and data-retention patterns.

---

# 7. Partitioning does not replace indexes

This distinction is important.

**Partitioning determines which physical partitions PostgreSQL needs to consider.**

**Indexes help PostgreSQL locate rows efficiently inside those partitions.**

Suppose August contains 100 million rows.

Partition pruning may reduce:

```text
10 billion rows
      ↓
100 million rows
```

An index can then reduce the amount of work needed within those 100 million rows.

For example:

```sql
CREATE INDEX idx_orders_customer
ON orders (customer_id);
```

With a partitioned table, PostgreSQL manages corresponding indexes on the partitions when you create an index on the partitioned parent.

A useful combination might be:

```text
Partition key:
    order_date

Index:
    (customer_id, order_date)
```

Then a query such as:

```sql
SELECT *
FROM orders
WHERE order_date >= DATE '2026-08-01'
  AND order_date < DATE '2026-09-01'
  AND customer_id = 12345;
```

can benefit from both:

```text
Partition pruning
       ↓
August partition
       ↓
Index lookup
       ↓
Matching customer rows
```

---

# 8. Partition size matters

Don't create partitions simply because PostgreSQL supports them.

For example, a table receiving 500 million rows per month could potentially use:

```text
Yearly partitions   → probably too large
Monthly partitions  → reasonable
Daily partitions    → maybe appropriate
Hourly partitions   → probably excessive unless justified
```

The correct granularity depends on:

```text
Data volume
Query patterns
Retention requirements
Maintenance windows
Index sizes
Backup/archive strategy
```

Too few partitions can reduce the benefits.

Too many partitions increase planning and administrative overhead.

---

# 9. Default partitions

You can create a default partition for rows that don't fit another partition.

```sql
CREATE TABLE orders_default
PARTITION OF orders DEFAULT;
```

This can prevent inserts from failing when a future partition hasn't been created yet.

Without an appropriate partition, an insert can fail because PostgreSQL has nowhere valid to route the row.

However, default partitions should generally be monitored. Otherwise unexpected data can quietly accumulate there.

---

# 10. Partition maintenance

This is where partitioning becomes extremely useful operationally.

Imagine a system keeps **seven years of transactions**.

Without partitioning, deleting old rows might require:

```sql
DELETE FROM orders
WHERE order_date < DATE '2019-01-01';
```

For hundreds of millions of rows this can create significant work:

```text
DELETE processing
WAL generation
Dead tuples
VACUUM work
Index cleanup
Replication traffic
```

With range partitions, old data can instead be managed at the partition level.

For example:

```sql
ALTER TABLE orders
DETACH PARTITION orders_2018_12;
```

The detached table can then be archived or handled separately.

Eventually:

```sql
DROP TABLE orders_2018_12;
```

This is one of the strongest reasons to partition very large tables.

---

# 11. Large-table lifecycle management

A mature architecture usually automates the entire partition lifecycle.

```text
              LARGE TABLE LIFECYCLE

                   New data
                      |
                      v
              Active partitions
                      |
        +-------------+-------------+
        |                           |
        v                           v
    Queries                     Inserts
        |
        v
Partition pruning
        |
        v
Older partitions
        |
        v
      Detach
        |
        v
   Archive / Export
        |
        v
       Drop
```

For monthly partitions, operations might automatically:

```text
Create next month's partition
        ↓
Create/verify indexes
        ↓
Collect statistics
        ↓
Monitor partition growth
        ↓
Detach partitions beyond retention
        ↓
Archive if required
        ↓
Drop expired partitions
```

You generally don't want an operations team manually creating every monthly partition.

---

# 12. Multi-level partitioning

PostgreSQL can also use subpartitioning.

For example:

```text
orders
  |
  +-- 2026-07
  |      +-- US
  |      +-- EU
  |      +-- APAC
  |
  +-- 2026-08
         +-- US
         +-- EU
         +-- APAC
```

The first level could be:

```text
RANGE(order_date)
```

and the second:

```text
LIST(region)
```

This can be useful for extremely large systems, but complexity increases quickly.

Use multi-level partitioning only when the workload actually requires it.

---

# 13. Important primary-key consideration

Partitioned tables have an important uniqueness consideration.

If you create a `PRIMARY KEY` or `UNIQUE` constraint on the partitioned table, the constraint generally needs to include the partition key columns.

For example, with:

```sql
PARTITION BY RANGE (order_date)
```

a design such as:

```sql
PRIMARY KEY (order_id, order_date)
```

can work.

But expecting PostgreSQL to enforce a simple:

```sql
PRIMARY KEY (order_id)
```

globally across independently partitioned data requires careful design.

This matters a lot when converting existing large tables to partitioning.

---

# 14. Operational monitoring

For large partitioned tables, monitor more than query speed.

Important areas include:

```text
Partition count
Partition size
Index size
Sequential scans
Index scans
Autovacuum activity
Dead tuples
Table/index bloat
Statistics freshness
Long-running transactions
Locking
Replication lag
Disk growth
```

Useful catalog views include:

```sql
SELECT *
FROM pg_stat_user_tables;

SELECT *
FROM pg_stat_user_indexes;
```

And storage can be inspected with functions such as:

```sql
SELECT
    pg_size_pretty(pg_total_relation_size('orders_2026_08'));
```

---

# 15. Common mistakes

The biggest architecture mistakes are usually:

1. **Partitioning small tables** where a normal index would be simpler.
2. **Choosing the wrong partition key**, so important queries cannot prune partitions.
3. **Creating thousands of tiny partitions** without a real operational reason.
4. **Assuming partitioning replaces indexes.**
5. **Not creating future partitions**, causing inserts to fail.
6. **Keeping a default partition but never monitoring it.**
7. **Deleting billions of expired rows individually** instead of using partition lifecycle operations.
8. **Using multi-level partitioning too early**, creating unnecessary complexity.

---

## Interview-level mental model

The easiest way to remember the topic is:

```text
                 PARTITIONING
                      |
       +--------------+--------------+
       |              |              |
     RANGE           LIST           HASH
       |              |              |
     Time          Category      Distribution
       |
       v
Partition Pruning
       |
       v
Relevant Partition(s)
       |
       v
Indexes
       |
       v
Matching Rows
```

And from an architecture perspective:

**Partitioning = data organization + query elimination + lifecycle management.**

A strong design question is not simply:

> "Will partitioning make this query faster?"

Instead ask:

> "How is the data queried, how fast does it grow, and how will we archive or delete it over the next 5–10 years?"

That question usually leads to the right partition key and partition granularity.
