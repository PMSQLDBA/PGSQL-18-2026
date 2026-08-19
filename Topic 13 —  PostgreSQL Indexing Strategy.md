## Topic 13 — PostgreSQL Indexing Strategy

PostgreSQL indexing is not about adding indexes everywhere. 

The goal is to choose an index based on the **query pattern, data type, table size, selectivity, and write frequency**.

### 1. Index types — quick architecture view

| Index          | Best suited for                           | Typical operators / queries            | Example                  |
| -------------- | ----------------------------------------- | -------------------------------------- | ------------------------ |
| **B-tree**     | General-purpose indexing                  | `=`, `<`, `>`, `BETWEEN`, `ORDER BY`   | Customer ID, timestamp   |
| **Hash**       | Equality only                             | `=`                                    | Exact lookup             |
| **GIN**        | Multi-valued / inverted data              | arrays, JSONB, full-text               | JSONB attributes         |
| **GiST**       | Geometric, ranges, spatial-style searches | overlap, containment, nearest-neighbor | Range / PostGIS          |
| **SP-GiST**    | Partitionable/search-tree data            | spatial, prefix, nearest-neighbor      | Points, IP addresses     |
| **BRIN**       | Huge physically ordered tables            | ranges                                 | Time-series/event tables |
| **Partial**    | Only selected rows                        | Depends on underlying index            | Active records           |
| **Expression** | Queries on computed values                | Expression-specific                    | `LOWER(email)`           |
| **Covering**   | Avoid table lookup                        | `INCLUDE` columns                      | Read-heavy lookup        |

---

# 2. B-tree

**B-tree is PostgreSQL's default and most commonly used index type.**

```sql
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);
```

It works very well for:

```sql
WHERE customer_id = 1001

WHERE created_at >= '2026-01-01'

WHERE amount BETWEEN 100 AND 500

ORDER BY created_at
```

For example:

```sql
SELECT *
FROM orders
WHERE customer_id = 1001;
```

Without an appropriate index PostgreSQL might use:

```text
Seq Scan
```

With the index it can potentially use:

```text
Index Scan using idx_orders_customer_id
```

### Composite B-tree indexes

Suppose the common query is:

```sql
SELECT *
FROM orders
WHERE customer_id = 1001
  AND status = 'PAID'
ORDER BY created_at DESC;
```

A useful index could be:

```sql
CREATE INDEX idx_orders_customer_status_created
ON orders(customer_id, status, created_at DESC);
```

Column order matters.

Think of:

```sql
(customer_id, status, created_at)
```

as an index primarily organized starting with `customer_id`, then `status`, then `created_at`.

A common design guideline is:

> Equality/filter columns first, followed by columns used for range filtering or ordering.

But don't apply this mechanically. Check the actual queries and `EXPLAIN (ANALYZE, BUFFERS)`.

---

# 3. Hash indexes

Hash indexes are designed primarily for **equality comparisons**.

```sql
CREATE INDEX idx_users_external_id_hash
ON users USING HASH(external_id);
```

Useful for:

```sql
WHERE external_id = 'ABC123'
```

Not useful for:

```sql
WHERE external_id > 'ABC123'
```

or:

```sql
ORDER BY external_id
```

### B-tree vs Hash

For most normal equality lookups, I would start with **B-tree** because it supports equality plus range and ordering operations.

Use Hash when you have a specific equality-only workload and testing shows a benefit.

---

# 4. GIN — Generalized Inverted Index

GIN is very important when working with:

* JSONB
* arrays
* full-text search
* values containing multiple searchable elements

Suppose:

```sql
CREATE TABLE products (
    id bigint,
    attributes jsonb
);
```

Example JSON:

```json
{
  "brand": "Apple",
  "color": "Black",
  "storage": "256GB"
}
```

A GIN index can be created as:

```sql
CREATE INDEX idx_products_attributes
ON products USING GIN(attributes);
```

Then containment queries such as:

```sql
SELECT *
FROM products
WHERE attributes @> '{"color":"Black"}';
```

can potentially use the GIN index.

### Why GIN?

Conceptually, instead of simply indexing the entire JSON document, PostgreSQL maintains index entries that make searching elements inside composite values efficient.

GIN is excellent for reads, but the trade-off is important:

> GIN indexes can be relatively expensive to maintain during inserts and updates.

So don't automatically put GIN indexes on every JSONB column.

---

# 5. GiST — Generalized Search Tree

GiST provides an extensible search-tree framework.

It is commonly associated with:

* geometric data
* ranges
* spatial data
* nearest-neighbor searches
* PostgreSQL extensions such as PostGIS

Example using a range:

```sql
CREATE TABLE reservations (
    id bigint,
    room_id bigint,
    booking_period tsrange
);
```

Index:

```sql
CREATE INDEX idx_reservations_period
ON reservations USING GIST(booking_period);
```

Then queries such as:

```sql
SELECT *
FROM reservations
WHERE booking_period &&
      tsrange('2026-08-19 10:00', '2026-08-19 12:00');
```

can efficiently search for overlapping ranges.

`&&` means **overlaps**.

---

# 6. SP-GiST

SP-GiST means **Space-Partitioned GiST**.

It is useful for data structures that naturally divide into search spaces.

Examples include:

* points
* IP/network-related data
* prefix searches
* certain geometric searches

Conceptually:

```text
GiST
   General extensible search tree

SP-GiST
   Partition search space into regions
```

SP-GiST is more specialized than B-tree or GIN.

Architecture rule:

> Don't select SP-GiST simply because it sounds more advanced. Choose it when the data type, operator class, and query pattern fit its partitioning model.

---

# 7. BRIN — Block Range Index

BRIN is extremely useful for **very large tables where column values correlate with physical row order**.

Imagine:

```text
event_log

1 billion rows
```

and data arrives chronologically:

```text
id      created_at
1       Jan 01
2       Jan 01
...
500M    Jun 01
...
1B      Dec 31
```

Create:

```sql
CREATE INDEX idx_event_log_created_brin
ON event_log USING BRIN(created_at);
```

Instead of storing an index entry pointing to every individual row like a B-tree, BRIN stores summary information about **ranges of table blocks**.

Conceptually:

```text
Heap blocks 1-128
created_at:
Jan 01 → Jan 03

Heap blocks 129-256
created_at:
Jan 03 → Jan 05

Heap blocks 257-384
created_at:
Jan 05 → Jan 07
```

Query:

```sql
SELECT *
FROM event_log
WHERE created_at >= '2026-08-01'
  AND created_at <  '2026-08-02';
```

PostgreSQL can eliminate many irrelevant block ranges.

### B-tree vs BRIN

| B-tree                   | BRIN                                  |
| ------------------------ | ------------------------------------- |
| Larger                   | Very small                            |
| Precise row-level lookup | Block-range filtering                 |
| Excellent random lookup  | Excellent for correlated large tables |
| More write overhead      | Lower maintenance overhead            |
| Broadly applicable       | Specialized                           |

For massive append-only time-series tables, BRIN should definitely be considered.

---

# 8. Partial indexes

A partial index indexes **only rows matching a condition**.

Suppose an orders table contains:

```text
100 million orders

95 million COMPLETED
3 million CANCELLED
2 million PENDING
```

Application frequently runs:

```sql
SELECT *
FROM orders
WHERE status = 'PENDING'
  AND customer_id = 1001;
```

Instead of indexing all 100 million rows:

```sql
CREATE INDEX idx_orders_pending_customer
ON orders(customer_id)
WHERE status = 'PENDING';
```

Now only pending orders are indexed.

Benefits:

```text
Smaller index
     ↓
Less storage
     ↓
Better cache efficiency
     ↓
Less index maintenance
```

Partial indexes are especially useful for:

* active records
* unprocessed jobs
* soft-deleted rows
* pending transactions
* recent/special subsets

Example:

```sql
CREATE INDEX idx_users_active_email
ON users(email)
WHERE deleted_at IS NULL;
```

---

# 9. Expression indexes

Suppose the application runs:

```sql
SELECT *
FROM users
WHERE LOWER(email) = 'john@example.com';
```

An ordinary index:

```sql
CREATE INDEX idx_users_email
ON users(email);
```

may not satisfy the intended expression lookup efficiently.

Instead:

```sql
CREATE INDEX idx_users_lower_email
ON users(LOWER(email));
```

Now the index matches:

```sql
WHERE LOWER(email) = 'john@example.com'
```

Another example:

```sql
CREATE INDEX idx_orders_order_date
ON orders((created_at::date));
```

for query patterns using the same expression.

Important principle:

> The indexed expression should match the expression used by the workload.

---

# 10. Covering indexes — `INCLUDE`

Suppose the application executes:

```sql
SELECT customer_id, status, total_amount
FROM orders
WHERE customer_id = 1001;
```

You might create:

```sql
CREATE INDEX idx_orders_customer_cover
ON orders(customer_id)
INCLUDE (status, total_amount);
```

Here:

```text
customer_id
     ↓
Index key

status
total_amount
     ↓
Included payload columns
```

This can allow PostgreSQL to perform an **Index Only Scan** when visibility conditions permit it.

Potential execution plan:

```text
Index Only Scan using idx_orders_customer_cover
```

instead of fetching the required values from heap pages.

### Why not put everything into INCLUDE?

Because indexes become larger.

That means:

```text
More storage
More memory/cache pressure
More WAL
More write overhead
More maintenance
```

Use covering indexes for important read-heavy query patterns.

---

# 11. Index maintenance

This is one of the most important production topics.

Indexes are **not free**.

For every:

```sql
INSERT
UPDATE
DELETE
```

PostgreSQL may have index work to perform.

Imagine:

```text
orders table
   │
   ├── index 1
   ├── index 2
   ├── index 3
   ├── index 4
   ├── index 5
   └── index 6
```

An insert may need to update several of those indexes.

So adding indexes can improve:

```text
SELECT performance
```

while hurting:

```text
INSERT
UPDATE
DELETE
```

This is why "index every column" is a bad strategy.

---

# 12. VACUUM and indexes

PostgreSQL uses MVCC.

An update typically creates a new row version rather than simply overwriting the existing tuple.

Over time:

```text
Updates / Deletes
       ↓
Dead tuples
       ↓
VACUUM
       ↓
Dead space becomes reusable
```

Autovacuum is therefore critical for healthy PostgreSQL systems.

Monitor whether autovacuum settings are appropriate for large/high-churn tables instead of disabling it.

---

# 13. REINDEX

Sometimes an index needs to be rebuilt.

For example:

```sql
REINDEX INDEX idx_orders_customer_id;
```

For production environments, depending on the situation and PostgreSQL version/features being used, you may use:

```sql
REINDEX INDEX CONCURRENTLY idx_orders_customer_id;
```

The concurrent form is useful when minimizing blocking is important, although it comes with additional work and limitations.

---

# 14. Creating indexes in production

Basic:

```sql
CREATE INDEX idx_orders_customer
ON orders(customer_id);
```

On a large busy production table, this can interfere with normal operations.

A common production approach is:

```sql
CREATE INDEX CONCURRENTLY idx_orders_customer
ON orders(customer_id);
```

This allows normal writes to continue while PostgreSQL builds the index, though the operation takes additional work/time and has restrictions—for example, it cannot simply be run inside a normal transaction block.

---

# 15. Find unused indexes

PostgreSQL exposes index usage statistics.

Example:

```sql
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan;
```

An index with:

```text
idx_scan = 0
```

may be unused.

But **do not immediately drop it**.

Check:

* How long statistics have been collected
* Whether statistics were recently reset
* Monthly/quarterly jobs
* Reporting queries
* Constraints
* Rare but critical queries
* Whether the index supports operational tasks

---

# 16. Duplicate/redundant indexes

Suppose you have:

```sql
INDEX (customer_id)
```

and:

```sql
INDEX (customer_id, status)
```

The first index **may** be redundant depending on the workload because the second index starts with `customer_id`.

But don't automatically remove it.

The smaller single-column index could still be useful because of:

* smaller size
* better cache behavior
* different query costs

Use real workload statistics before removing indexes.

---

# 17. Index selectivity

Consider:

```text
gender
------
M
F
```

An index on a very low-cardinality column often provides limited value for queries returning a large percentage of the table.

Compare that with:

```text
customer_id
-----------
10000001
10000002
10000003
...
```

`customer_id` is highly selective.

The optimizer may decide:

```text
Low selectivity
      ↓
Seq Scan
```

rather than:

```text
Index Scan
```

because retrieving a large portion of a table through an index can be more expensive than reading the table sequentially.

This leads to an important interview point:

> Having an index does not mean PostgreSQL will use it.

---

# 18. `EXPLAIN ANALYZE` for index validation

Never assume an index fixed the problem.

Test it.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE customer_id = 1001;
```

Look for operations such as:

```text
Seq Scan
Index Scan
Index Only Scan
Bitmap Index Scan
Bitmap Heap Scan
```

Also examine:

```text
actual time
rows
loops
Buffers
Rows Removed by Filter
```

Architecture workflow:

```text
Identify slow query
       ↓
EXPLAIN (ANALYZE, BUFFERS)
       ↓
Understand filtering / joins / sorting
       ↓
Design appropriate index
       ↓
Test
       ↓
Compare execution plan
       ↓
Measure production impact
```

---

# 19. Bitmap Index Scan

Suppose:

```sql
SELECT *
FROM orders
WHERE status = 'PENDING';
```

PostgreSQL may choose:

```text
Bitmap Index Scan
        ↓
Bitmap Heap Scan
```

instead of a normal Index Scan.

Simplified idea:

```text
Index
  ↓
Find matching tuple locations
  ↓
Build bitmap
  ↓
Read required heap pages efficiently
```

This can be useful when a query returns more rows than would make individual random heap accesses attractive, but not enough to justify scanning the entire table.

---

# 20. Production indexing strategy

For a real production system, I would use this process:

```text
1. Capture workload
        ↓
2. Find expensive/frequent queries
        ↓
3. Check EXPLAIN (ANALYZE, BUFFERS)
        ↓
4. Identify predicates
        ↓
5. Identify JOIN columns
        ↓
6. Identify ORDER BY / GROUP BY patterns
        ↓
7. Select index type
        ↓
8. Consider composite / partial / expression
        ↓
9. Consider INCLUDE for read-heavy queries
        ↓
10. Build safely
        ↓
11. Measure performance
        ↓
12. Monitor index usage and size
        ↓
13. Remove genuinely redundant indexes
```

---

# 21. Architecture example

Imagine an e-commerce `orders` table:

```sql
CREATE TABLE orders (
    order_id bigint PRIMARY KEY,
    customer_id bigint NOT NULL,
    status varchar(30),
    created_at timestamptz NOT NULL,
    total_amount numeric(12,2),
    metadata jsonb
);
```

### Requirement 1 — customer order history

```sql
SELECT order_id, status, created_at, total_amount
FROM orders
WHERE customer_id = $1
ORDER BY created_at DESC
LIMIT 50;
```

Possible index:

```sql
CREATE INDEX idx_orders_customer_created
ON orders(customer_id, created_at DESC)
INCLUDE (status, total_amount);
```

---

### Requirement 2 — pending orders

```sql
SELECT *
FROM orders
WHERE status = 'PENDING'
  AND created_at < NOW() - INTERVAL '1 hour';
```

Possible partial index:

```sql
CREATE INDEX idx_orders_pending_created
ON orders(created_at)
WHERE status = 'PENDING';
```

---

### Requirement 3 — JSON search

```sql
SELECT *
FROM orders
WHERE metadata @> '{"source":"mobile"}';
```

Possible:

```sql
CREATE INDEX idx_orders_metadata_gin
ON orders USING GIN(metadata);
```

---

### Requirement 4 — massive historical table

If `orders` contains billions of rows and `created_at` strongly follows physical insertion order:

```sql
CREATE INDEX idx_orders_created_brin
ON orders USING BRIN(created_at);
```

Now you can see that **one table can legitimately use several index types because each solves a different access pattern.**

---

# 22. Interview scenario

**Question:**

> We have a 2-billion-row transaction table. Most queries search transactions for a particular date range. Data is inserted chronologically. Which index would you consider?

Strong answer:

**BRIN should be evaluated first** because the timestamp is correlated with the physical row order and the table is extremely large. BRIN is tiny compared with a B-tree and can eliminate large ranges of irrelevant heap blocks.

However, if the requirement is highly selective point lookup or very small ranges requiring consistently low latency, I would benchmark a B-tree as well.

---

Another common question:

> Why isn't PostgreSQL using my index?

Possible reasons include:

* Query returns too much of the table.
* Poor selectivity.
* Statistics are stale.
* Expression doesn't match the index.
* Composite index column order doesn't suit the query.
* Implicit/type conversion interferes with the intended access path.
* Sequential scan is genuinely cheaper.
* Table is small enough that sequential scanning wins.
* Planner cost estimates favor another plan.

The right response is not:

```sql
SET enable_seqscan = off;
```

The right response is to investigate the execution plan and statistics.

---

# 23. Easy way to remember

```text
B-tree
→ Normal equality + range + sorting

Hash
→ Equality

GIN
→ JSONB + Arrays + Full Text

GiST
→ Ranges + Geometry + Spatial-style searches

SP-GiST
→ Partitionable search spaces

BRIN
→ Huge + physically correlated data

Partial
→ Index only important rows

Expression
→ Index calculated value

INCLUDE
→ Cover query output
```

## Architecture takeaway

The most important concept for **PostgreSQL Indexing Strategy** is:

> **Design indexes from access patterns, not from table columns.**

Start with the important SQL queries, understand their `WHERE`, `JOIN`, `ORDER BY`, range and projection requirements, then select the smallest index structure that solves those access patterns.

For a production architecture, the balance is:

```text
Query Performance
       ↕
Index Size
       ↕
Write Overhead
       ↕
Maintenance Cost
       ↕
Operational Availability
```

A database with **10 carefully designed indexes** can easily outperform one with **50 indexes added without understanding the workload**.
