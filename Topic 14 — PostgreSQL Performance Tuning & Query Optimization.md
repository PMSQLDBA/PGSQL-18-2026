# Topic 14 — PostgreSQL Performance Tuning & Query Optimization

This topic is about **finding why PostgreSQL is slow, reading what PostgreSQL is actually doing, and fixing the right layer instead of randomly changing parameters**.

A strong PostgreSQL engineer should be comfortable moving through this flow:

**Slow query → EXPLAIN → EXPLAIN ANALYZE → execution plan → row estimates/statistics → scan/join choice → indexes/query rewrite → memory/parallelism → server tuning → verify improvement**

---

## 1. Performance Tuning: The Big Picture

PostgreSQL performance problems usually come from one or more of these areas:

| Area          | Typical problem                              |
| ------------- | -------------------------------------------- |
| SQL           | Bad query design, unnecessary rows/columns   |
| Indexes       | Missing, unused, duplicate, or wrong indexes |
| Statistics    | PostgreSQL estimates rows incorrectly        |
| Joins         | Wrong join strategy due to estimates/indexes |
| Memory        | Sort/hash operations spill to disk           |
| I/O           | Too many pages read from storage             |
| Parallelism   | Query cannot or does not use enough workers  |
| Tables        | Bloat, dead tuples, poor VACUUM behavior     |
| Concurrency   | Locks, contention, too many connections      |
| Configuration | Poor `work_mem`, cache/cost settings, etc.   |

The first rule is:

> **Do not start by changing PostgreSQL parameters. First identify where the time is actually going.**

---

# 2. EXPLAIN

`EXPLAIN` shows the execution plan PostgreSQL **plans to use**.

```sql
EXPLAIN
SELECT *
FROM orders
WHERE customer_id = 1001;
```

Possible output:

```text
Bitmap Heap Scan on orders
  Recheck Cond: (customer_id = 1001)
  -> Bitmap Index Scan on idx_orders_customer
       Index Cond: (customer_id = 1001)
```

It tells you things such as:

* Scan method
* Join method
* Estimated rows
* Estimated cost
* Sort operations
* Parallel workers
* Index usage

But `EXPLAIN` **does not execute the query**.

That makes it useful when running the query itself would be expensive or dangerous.

---

# 3. EXPLAIN ANALYZE

For real performance investigation, this is much more useful:

```sql
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE customer_id = 1001;
```

Unlike `EXPLAIN`, PostgreSQL **actually executes the statement**.

You can now compare:

```text
estimated rows
vs
actual rows
```

and:

```text
estimated cost
vs
actual execution behavior
```

A common production-style command is:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
```

And for deeper investigation:

```sql
EXPLAIN (
    ANALYZE,
    BUFFERS,
    VERBOSE,
    SETTINGS
)
SELECT ...
```

`BUFFERS` is particularly valuable because it shows whether PostgreSQL is working mostly from cache or reading data from storage.

---

# 4. Important Warning About EXPLAIN ANALYZE

Because it executes the SQL, be careful with:

```sql
EXPLAIN ANALYZE
DELETE ...
```

```sql
EXPLAIN ANALYZE
UPDATE ...
```

```sql
EXPLAIN ANALYZE
INSERT ...
```

The modification really happens.

For testing, you can use a transaction:

```sql
BEGIN;

EXPLAIN ANALYZE
UPDATE orders
SET status = 'CLOSED'
WHERE order_id = 100;

ROLLBACK;
```

---

# 5. Understanding Cost

You may see:

```text
Seq Scan on orders
(cost=0.00..18500.00 rows=500 width=80)
```

The important pieces are:

```text
cost=startup_cost..total_cost
rows=estimated_rows
width=estimated_average_row_size
```

So:

```text
cost=0.00..18500.00
```

means approximately:

```text
startup cost = 0
total estimated cost = 18500
```

Cost is **not milliseconds**.

It is PostgreSQL's internal estimate used to compare possible plans.

---

# 6. Estimated Rows vs Actual Rows

This is one of the most important things to check.

Example:

```text
Index Scan
(cost=... rows=100 ...)
(actual ... rows=500000 ...)
```

PostgreSQL expected:

```text
100 rows
```

but actually received:

```text
500,000 rows
```

That is a huge estimation error.

This can cause PostgreSQL to select:

* Wrong join algorithm
* Wrong join order
* Wrong scan method
* Insufficient memory expectations
* Poor parallelism decisions

This is why statistics are so important.

---

# 7. Main Scan Types

You should understand these four.

### Sequential Scan

```text
Seq Scan
```

PostgreSQL reads the table pages sequentially.

A sequential scan is **not automatically bad**.

For:

```sql
SELECT *
FROM orders;
```

reading the whole table may be exactly the correct plan.

It can also be correct when a query returns a large percentage of the table.

---

### Index Scan

```text
Index Scan using idx_orders_customer
```

PostgreSQL uses an index to locate rows and then accesses the table.

Good for selective queries such as:

```sql
SELECT *
FROM orders
WHERE order_id = 12345;
```

---

### Index Only Scan

```text
Index Only Scan
```

PostgreSQL may be able to satisfy the query primarily from the index without retrieving every tuple from the heap.

Example:

```sql
CREATE INDEX idx_orders_customer_status
ON orders(customer_id, status);
```

Query:

```sql
SELECT customer_id, status
FROM orders
WHERE customer_id = 1001;
```

Whether it truly avoids heap access also depends on PostgreSQL's visibility map.

---

### Bitmap Scan

Often appears as:

```text
Bitmap Heap Scan
  -> Bitmap Index Scan
```

Useful when PostgreSQL expects more matches than are ideal for a normal index scan, but not enough to justify scanning the entire table.

Think:

```text
few rows
   ↓
Index Scan
   ↓
Bitmap Scan
   ↓
Seq Scan
   ↓
many rows
```

That is conceptual rather than a strict rule.

---

# 8. Execution Plans Are Trees

Suppose the plan looks like:

```text
Hash Join
 ├── Seq Scan on orders
 └── Hash
      └── Seq Scan on customers
```

Read the plan generally from the **inside/bottom upward**.

PostgreSQL first obtains rows from the child nodes and feeds those results into their parent operation.

For example:

```text
Seq Scan customers
      ↓
Hash
      ↓
Hash Join
```

Understanding this tree structure makes complex plans much easier to analyze.

---

# 9. Join Algorithms

PostgreSQL commonly uses three major join strategies:

```text
Nested Loop
Hash Join
Merge Join
```

You should understand **why PostgreSQL chooses each one**.

---

## 10. Nested Loop Join

Conceptually:

```text
for each row in table A
    find matching rows in table B
```

Example plan:

```text
Nested Loop
  -> Index Scan customers
  -> Index Scan orders
```

Usually strong when:

* Outer result is small
* Inner side has a useful index
* Query is highly selective

Example:

```sql
SELECT *
FROM customers c
JOIN orders o
  ON o.customer_id = c.customer_id
WHERE c.customer_id = 100;
```

If only one customer is selected, PostgreSQL can quickly look up that customer's orders.

Problem scenario:

```text
outer rows = 1,000,000
inner lookup repeated 1,000,000 times
```

A nested loop can become extremely expensive.

---

# 11. Hash Join

Conceptually:

```text
Build hash table from one input
        ↓
Probe it using rows from other input
```

Example:

```text
Hash Join
  Hash Cond: (o.customer_id = c.customer_id)
```

Often good for:

* Large datasets
* Equality joins
* No useful index for repeated lookups

Example:

```sql
SELECT ...
FROM orders o
JOIN customers c
ON o.customer_id = c.customer_id;
```

Memory matters because the hash table has to be built.

If it becomes too large, PostgreSQL may process it in batches and use temporary storage.

---

# 12. Merge Join

A merge join works efficiently when both inputs can be processed in join-key order.

Conceptually:

```text
Sorted A
    ↓
Merge
    ↑
Sorted B
```

Often useful for:

* Large datasets
* Sorted/indexed inputs
* Equality joins
* Situations where ordered access is already available

You may see:

```text
Merge Join
  Merge Cond: (a.id = b.id)
```

with child nodes involving:

```text
Sort
```

or ordered index scans.

---

# 13. Statistics

The optimizer doesn't know the future result of the query.

It **estimates** it.

PostgreSQL collects statistics about table columns and uses them when building plans.

Refresh statistics using:

```sql
ANALYZE orders;
```

Or:

```sql
VACUUM ANALYZE orders;
```

Statistics contain information such as:

* Null fraction
* Number of distinct values
* Most common values
* Histograms
* Correlation

You can inspect statistics through:

```sql
SELECT *
FROM pg_stats
WHERE tablename = 'orders';
```

---

# 14. Why Statistics Matter

Imagine:

```sql
SELECT *
FROM orders
WHERE status = 'FAILED';
```

Suppose there are:

```text
100,000,000 total rows
100 FAILED rows
```

An index could be excellent.

But if PostgreSQL incorrectly estimates:

```text
20,000,000 FAILED rows
```

it might decide:

```text
Sequential Scan
```

instead of:

```text
Index Scan
```

So sometimes the real problem isn't:

> PostgreSQL ignored my index.

It is:

> PostgreSQL incorrectly estimated the number of rows.

---

# 15. Statistics Target

For columns with complicated/skewed distributions, the default statistics may not be enough.

Example:

```sql
ALTER TABLE orders
ALTER COLUMN status
SET STATISTICS 500;
```

Then:

```sql
ANALYZE orders;
```

Higher statistics targets can improve estimates, but also increase ANALYZE work and planning overhead.

Do it selectively rather than setting huge values everywhere.

---

# 16. Extended Statistics

Sometimes columns are correlated.

Example:

```text
country
state
city
```

These values aren't independent.

You can create extended statistics:

```sql
CREATE STATISTICS customer_location_stats
ON country, state, city
FROM customers;

ANALYZE customers;
```

This can help PostgreSQL estimate multi-column conditions more accurately.

---

# 17. Index Optimization

A query:

```sql
SELECT *
FROM orders
WHERE customer_id = 1001;
```

may benefit from:

```sql
CREATE INDEX idx_orders_customer
ON orders(customer_id);
```

But indexes are not free.

Every index adds overhead to:

```text
INSERT
UPDATE
DELETE
VACUUM
storage
WAL generation
```

So don't create indexes for every column.

---

# 18. Composite Indexes

Suppose the query is:

```sql
SELECT *
FROM orders
WHERE customer_id = 1001
AND status = 'OPEN';
```

You might use:

```sql
CREATE INDEX idx_orders_customer_status
ON orders(customer_id, status);
```

Column order matters.

The correct order depends on actual query patterns—not simply "put the most selective column first."

Consider:

* Equality conditions
* Range conditions
* Sorting
* Grouping
* Join predicates
* Prefix usage
* Actual workload

---

# 19. Partial Indexes

Very useful when only a subset of rows matters.

Suppose:

```text
95% CLOSED
5% OPEN
```

and applications constantly query open orders.

Instead of:

```sql
CREATE INDEX idx_status
ON orders(status);
```

you might use:

```sql
CREATE INDEX idx_open_orders
ON orders(customer_id)
WHERE status = 'OPEN';
```

The index becomes much smaller and potentially more efficient for those queries.

---

# 20. Expression Indexes

Consider:

```sql
SELECT *
FROM users
WHERE lower(email) = 'abc@example.com';
```

A normal index on:

```sql
email
```

may not provide the desired indexed expression lookup.

You can create:

```sql
CREATE INDEX idx_users_lower_email
ON users(lower(email));
```

---

# 21. Covering Indexes with INCLUDE

Suppose:

```sql
SELECT customer_id, order_date, status
FROM orders
WHERE customer_id = 1001;
```

You could use:

```sql
CREATE INDEX idx_orders_customer
ON orders(customer_id)
INCLUDE (order_date, status);
```

The extra columns are available from the index while not being key columns.

This can help enable efficient index-only access.

---

# 22. Sargability

A very common query-performance problem is wrapping indexed columns in expressions.

Example:

```sql
SELECT *
FROM orders
WHERE DATE(order_date) = '2026-08-19';
```

Often better:

```sql
SELECT *
FROM orders
WHERE order_date >= '2026-08-19'
AND order_date < '2026-08-20';
```

This gives PostgreSQL a straightforward range condition that can use a B-tree index on `order_date`.

Another example:

Instead of:

```sql
WHERE lower(email) = lower('ABC@example.com')
```

either normalize the data/query appropriately or create an expression index when that search pattern is required.

---

# 23. Avoid `SELECT *` When You Don't Need Everything

Instead of:

```sql
SELECT *
FROM orders
WHERE customer_id = 1001;
```

prefer:

```sql
SELECT order_id,
       order_date,
       status
FROM orders
WHERE customer_id = 1001;
```

Benefits can include:

* Less data transferred
* Less memory
* Less I/O
* Smaller intermediate results
* Better chance of index-only access

---

# 24. Filter Early

Suppose:

```sql
SELECT ...
FROM huge_orders o
JOIN customers c
ON ...
WHERE o.order_date >= CURRENT_DATE - INTERVAL '1 day';
```

The goal is to process only relevant rows as early as the optimizer can safely do so.

Good indexes and clear predicates make this easier.

Don't assume manually moving predicates into subqueries automatically makes PostgreSQL faster; PostgreSQL can perform many transformations itself.

Always verify with the execution plan.

---

# 25. Sort Performance

You may find:

```text
Sort
  Sort Method: external merge
  Disk: 2048MB
```

This is an important clue.

It means the sort exceeded available working memory and used temporary disk storage.

Compare that with:

```text
Sort Method: quicksort
Memory: 200MB
```

which remained in memory.

---

# 26. `work_mem`

`work_mem` controls memory available to certain query operations such as:

* Sort
* Hash
* Aggregation
* Some join operations

Example:

```sql
SET work_mem = '256MB';
```

But this is dangerous as a global tuning strategy.

`work_mem` is **not simply memory per connection**.

A single query can contain multiple memory-consuming operations, and parallel workers can increase memory usage further.

For example, conceptually:

```text
100 connections
× several operations
× 256 MB
```

can create severe memory pressure.

A better strategy can be:

```sql
SET LOCAL work_mem = '256MB';
```

for a specific transaction/reporting workload rather than globally assigning a huge value.

---

# 27. `shared_buffers`

One major PostgreSQL memory setting is:

```text
shared_buffers
```

This controls PostgreSQL's own shared buffer cache.

PostgreSQL also benefits from the operating system's filesystem cache, so you normally don't want PostgreSQL to consume all RAM.

There isn't one perfect percentage for every environment.

You must consider:

```text
Total RAM
OS
workload
connections
work_mem
maintenance operations
OS cache
other applications
```

---

# 28. `effective_cache_size`

This setting is commonly misunderstood.

```text
effective_cache_size
```

does **not allocate memory**.

It tells the planner roughly how much filesystem + PostgreSQL cache is likely available for query planning decisions.

That influences choices such as:

```text
Index Scan
vs
Sequential Scan
```

---

# 29. `maintenance_work_mem`

Used for maintenance operations such as:

```text
VACUUM
CREATE INDEX
ALTER TABLE ADD FOREIGN KEY
```

A larger value can speed certain maintenance operations.

But again, consider concurrency and total memory.

---

# 30. Parallel Query

PostgreSQL can parallelize suitable queries.

Example plan:

```text
Gather
  Workers Planned: 4
  -> Parallel Seq Scan on orders
```

Or:

```text
Finalize Aggregate
  -> Gather
       -> Partial Aggregate
            -> Parallel Seq Scan
```

The idea is:

```text
Worker 1 ─┐
Worker 2 ─┤
Worker 3 ─┼→ Gather → result
Worker 4 ─┘
```

Parallelism is particularly useful for:

* Large sequential scans
* Aggregations
* Large joins
* Analytical/reporting queries

---

# 31. Parallelism Settings

Important parameters include:

```text
max_worker_processes
max_parallel_workers
max_parallel_workers_per_gather
min_parallel_table_scan_size
min_parallel_index_scan_size
parallel_setup_cost
parallel_tuple_cost
```

Don't simply increase:

```text
max_parallel_workers_per_gather
```

to a huge number.

More workers aren't always faster.

They also consume:

* CPU
* Memory
* Worker processes

and can reduce total system throughput when many queries run concurrently.

---

# 32. Finding Slow SQL

One of the most valuable PostgreSQL extensions is:

```text
pg_stat_statements
```

Enable it appropriately and query statistics such as:

```sql
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

This answers:

> Which SQL statements consume the most total database time?

That is often more useful than finding only the single slowest execution.

---

# 33. Total Time vs Average Time

Suppose:

```text
Query A
2 ms × 10,000,000 calls
```

versus:

```text
Query B
30 seconds × 1 call
```

Query B looks terrible individually.

But Query A consumes:

```text
20,000 seconds
```

of cumulative execution time.

Performance tuning should therefore consider:

```text
total execution time
average execution time
number of calls
rows processed
I/O
business importance
```

---

# 34. Slow Query Logging

Important settings include:

```text
log_min_duration_statement
```

For example:

```text
log_min_duration_statement = '1s'
```

This can log queries taking at least one second.

Another useful tool is:

```text
auto_explain
```

It can automatically log execution plans for slow statements.

Be careful with logging settings in high-throughput production systems because excessive logging itself creates overhead.

---

# 35. BUFFERS: One of the Best Diagnostic Tools

Use:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
```

You might see:

```text
Buffers: shared hit=50000 read=10000
```

Conceptually:

```text
hit
```

means PostgreSQL found pages already available in shared buffers.

```text
read
```

means pages had to be read into PostgreSQL's buffer cache.

Large read activity may point you toward I/O-heavy behavior.

But don't interpret one execution in isolation because warm-cache and cold-cache executions can look very different.

---

# 36. Temporary File Usage

Large sorts/hashes can create temporary files.

Monitor database statistics such as:

```sql
SELECT
    datname,
    temp_files,
    temp_bytes
FROM pg_stat_database;
```

Large values can indicate:

* Insufficient working memory for certain workloads
* Huge sorts
* Huge hash operations
* Poor query design
* Excessive intermediate result sets

---

# 37. VACUUM and Performance

PostgreSQL uses MVCC.

Updates and deletes can leave dead tuples that later need cleanup.

If vacuuming cannot keep up, you can see:

```text
dead tuples
table/index bloat
more I/O
slower scans
transaction ID problems
```

Monitor:

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables;
```

Autovacuum tuning is therefore part of performance tuning.

---

# 38. Table Bloat

Consider a heavily updated table:

```text
10 GB useful/live data
40 GB physical relation/index footprint
```

Queries may read far more pages than logically necessary.

Possible actions depend on the problem:

```text
VACUUM
VACUUM FULL
REINDEX
pg_repack-style online maintenance
better autovacuum settings
fillfactor changes
```

Do not immediately run `VACUUM FULL` on production.

It takes an aggressive lock and rewrites the table.

---

# 39. Index Bloat and Unused Indexes

Indexes can also become expensive.

Check index usage:

```sql
SELECT
    schemaname,
    relname,
    indexrelname,
    idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan;
```

An index with:

```text
idx_scan = 0
```

is worth investigating.

But don't immediately drop it.

It could support:

* Rare monthly jobs
* Foreign-key-related access patterns
* Disaster/recovery operations
* Seasonal workloads
* Recently restarted statistics

Always understand the workload first.

---

# 40. Locks Can Look Like Slow Queries

Suppose:

```text
Query execution itself = 10 ms
Lock wait = 60 seconds
```

The user sees:

```text
60-second query
```

but query optimization isn't the real problem.

Check activity:

```sql
SELECT *
FROM pg_stat_activity
WHERE state <> 'idle';
```

and lock information:

```sql
SELECT *
FROM pg_locks;
```

Performance investigation must distinguish:

```text
CPU time
I/O time
lock wait
client wait
connection wait
```

---

# 41. Long Transactions

Long-running transactions can prevent PostgreSQL from cleaning up old row versions efficiently.

This can contribute to:

```text
dead tuple accumulation
bloat
vacuum problems
transaction ID pressure
```

Look at transaction age:

```sql
SELECT
    pid,
    usename,
    state,
    xact_start,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

---

# 42. Connection Tuning

A common mistake is:

```text
max_connections = 5000
```

just because applications need high concurrency.

Each PostgreSQL backend consumes resources.

Large numbers of active database connections can cause:

```text
memory pressure
CPU context switching
contention
reduced cache efficiency
```

Connection pooling is often preferable.

Architecture:

```text
Applications
     ↓
Connection Pool
     ↓
Smaller number of PostgreSQL sessions
     ↓
PostgreSQL
```

---

# 43. Query Rewrite Example

Suppose:

```sql
SELECT *
FROM orders
WHERE EXTRACT(YEAR FROM order_date) = 2026;
```

This may make straightforward B-tree range access difficult.

Better:

```sql
SELECT *
FROM orders
WHERE order_date >= DATE '2026-01-01'
  AND order_date < DATE '2027-01-01';
```

Now an index such as:

```sql
CREATE INDEX idx_orders_date
ON orders(order_date);
```

has a clear range predicate to work with.

---

# 44. `OR` Conditions

Suppose:

```sql
SELECT *
FROM orders
WHERE customer_id = 100
OR status = 'FAILED';
```

PostgreSQL may use bitmap index operations if appropriate.

Sometimes rewriting as `UNION ALL` can help for specific workloads:

```sql
SELECT *
FROM orders
WHERE customer_id = 100

UNION ALL

SELECT *
FROM orders
WHERE status = 'FAILED'
AND customer_id <> 100;
```

But this is **not automatically faster**.

Always compare:

```sql
EXPLAIN (ANALYZE, BUFFERS)
```

for both versions.

---

# 45. `NOT IN` vs `NOT EXISTS`

For anti-join patterns, `NOT EXISTS` is often clearer and avoids `NULL` surprises.

Example:

```sql
SELECT c.*
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

This may produce an efficient anti-join plan:

```text
Hash Anti Join
```

or another suitable strategy.

The important point isn't blindly replacing every `NOT IN`; it is understanding both SQL semantics and the resulting execution plan.

---

# 46. Pagination Performance

This looks simple:

```sql
SELECT *
FROM orders
ORDER BY order_id
LIMIT 100
OFFSET 1000000;
```

But PostgreSQL may need to process/discard a huge number of earlier rows.

For deep pagination, keyset pagination is often better:

```sql
SELECT *
FROM orders
WHERE order_id > 1000000
ORDER BY order_id
LIMIT 100;
```

Architecture:

```text
OFFSET pagination
scan → discard → discard → discard → return

Keyset pagination
seek to key → return
```

---

# 47. CTE Performance

Older PostgreSQL advice often says:

> CTEs always act as optimization fences.

That is no longer generally correct in modern PostgreSQL.

A CTE such as:

```sql
WITH recent_orders AS (
    SELECT *
    FROM orders
    WHERE order_date >= CURRENT_DATE - 7
)
SELECT ...
FROM recent_orders;
```

may be inlined depending on the query.

PostgreSQL also supports explicit behavior such as:

```sql
WITH x AS MATERIALIZED (...)
```

and:

```sql
WITH x AS NOT MATERIALIZED (...)
```

Use the execution plan rather than relying on old rules.

---

# 48. Partitioning and Performance

For very large tables:

```text
orders
 ├── orders_2024
 ├── orders_2025
 └── orders_2026
```

A query:

```sql
SELECT *
FROM orders
WHERE order_date >= '2026-08-01'
AND order_date < '2026-09-01';
```

may only need relevant partitions.

This is:

```text
Partition Pruning
```

But partitioning isn't a magic performance button.

Poor partitioning can create:

* Too many partitions
* Planning overhead
* Operational complexity
* Poor index strategy

---

# 49. Server Tuning Areas

At server level, think in categories rather than random parameters.

### Memory

```text
shared_buffers
work_mem
maintenance_work_mem
effective_cache_size
```

### WAL / Checkpoints

```text
max_wal_size
checkpoint_timeout
checkpoint_completion_target
wal_buffers
```

### Planner

```text
random_page_cost
seq_page_cost
effective_cache_size
default_statistics_target
```

### Parallelism

```text
max_worker_processes
max_parallel_workers
max_parallel_workers_per_gather
```

### Autovacuum

```text
autovacuum_vacuum_threshold
autovacuum_vacuum_scale_factor
autovacuum_analyze_threshold
autovacuum_analyze_scale_factor
```

Server tuning should match the workload.

An OLTP database and a 5-TB analytics database should not blindly use identical settings.

---

# 50. OLTP vs OLAP Tuning

### OLTP

Typical workload:

```text
many small queries
short transactions
high concurrency
index lookups
INSERT/UPDATE/DELETE
```

Focus on:

```text
indexes
connection pooling
low latency
lock contention
autovacuum
efficient SQL
cache efficiency
```

### OLAP / Reporting

Typical workload:

```text
large scans
large joins
aggregations
fewer but expensive queries
```

Focus more on:

```text
parallelism
work_mem
I/O throughput
partitioning
large sequential scans
aggregation performance
```

---

# 51. A Practical Slow Query Investigation

Suppose users report:

> Customer order history takes 25 seconds.

Start with the actual SQL:

```sql
SELECT
    o.order_id,
    o.order_date,
    o.status,
    p.payment_status
FROM orders o
LEFT JOIN payments p
    ON p.order_id = o.order_id
WHERE o.customer_id = 900001
ORDER BY o.order_date DESC;
```

Run:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
```

Suppose you discover:

```text
Seq Scan on orders
Rows Removed by Filter: 99,500,000
```

That is a strong clue.

Check indexes.

Perhaps there is no useful index on:

```text
customer_id
```

You might consider:

```sql
CREATE INDEX idx_orders_customer_date
ON orders(customer_id, order_date DESC);
```

Now rerun:

```sql
EXPLAIN (ANALYZE, BUFFERS)
```

You may get:

```text
Index Scan using idx_orders_customer_date
```

and execution changes from:

```text
25 seconds
```

to something like:

```text
tens of milliseconds
```

The exact improvement depends on the actual data and system.

The important process is:

```text
Measure
   ↓
Identify bottleneck
   ↓
Change one thing
   ↓
Measure again
```

---

# 52. Reading a Plan Systematically

When you receive a huge execution plan, don't randomly read every line.

Use this sequence.

**1. Check total execution time**

```text
Execution Time
```

**2. Find expensive nodes**

Look for operations processing large amounts of data or consuming significant time.

**3. Compare estimated vs actual rows**

```text
rows=10
actual rows=2,000,000
```

Huge mismatch = investigate statistics/selectivity.

**4. Check loops**

Example:

```text
actual ... loops=500000
```

A cheap node repeated 500,000 times may be the real cost.

**5. Look for scans**

```text
Seq Scan
Index Scan
Bitmap Heap Scan
Index Only Scan
```

**6. Check joins**

```text
Nested Loop
Hash Join
Merge Join
```

**7. Check sorts/hashes**

Look for:

```text
Disk
batches
temporary data
```

**8. Check buffers**

Look for excessive:

```text
shared read
temp read
temp written
```

**9. Check filters**

Especially:

```text
Rows Removed by Filter
```

**10. Only then decide the fix.**

---

# 53. Important `EXPLAIN ANALYZE` Numbers

Suppose:

```text
Index Scan using idx_orders_customer on orders
(cost=0.43..100.20 rows=100 width=80)
(actual time=0.050..500.000 rows=500000 loops=1)
```

Read it as:

```text
Planner expected rows: 100
Actual rows:           500,000

First row around:      0.050 ms
Node completion:       500 ms
Loops:                 1
```

The biggest clue isn't necessarily `500 ms`.

It may be:

```text
100 estimated
vs
500,000 actual
```

That tells you the optimizer was planning based on a very different picture of the data.

---

# 54. `loops` — Common Interview Trap

Suppose:

```text
actual time=0.01..0.05 rows=1 loops=100000
```

People sometimes say:

> That node took only 0.05 ms.

Not exactly.

`EXPLAIN ANALYZE` timing for a repeatedly executed node is reported per loop/averaged behavior.

Conceptually, its contribution can be roughly:

```text
node time × loops
```

So:

```text
0.05 ms × 100,000
≈ 5,000 ms
```

This is why nested-loop plans must be inspected carefully.

---

# 55. Why PostgreSQL May Ignore an Index

Very common interview question:

> We created an index, but PostgreSQL still performs a sequential scan. Why?

Possible reasons:

1. Query returns a large percentage of the table.
2. Table is small.
3. Statistics are stale or inaccurate.
4. Function/expression prevents matching the index.
5. Data type mismatch or casts affect the predicate.
6. Index column order doesn't match the access pattern.
7. Planner estimates sequential I/O as cheaper.
8. Low selectivity.
9. Partial-index predicate doesn't match.
10. Query needs many heap pages anyway.

So:

> **Index exists ≠ Index must be used.**

PostgreSQL chooses what it estimates to be the cheapest valid plan.

---

# 56. Why Forcing a Planner Choice Is Usually the Wrong First Fix

You can experiment with settings such as:

```sql
SET enable_seqscan = off;
```

But this should generally be treated as a **diagnostic experiment**, not the normal solution.

If disabling sequential scans suddenly improves the query dramatically, ask:

> Why did the optimizer think the sequential scan was cheaper?

Investigate:

```text
statistics
selectivity
cost settings
data distribution
query predicates
index design
```

Fix the cause rather than globally fighting the optimizer.

---

# 57. Production Performance Workflow

A mature production workflow looks like:

```text
Application reports latency
        ↓
Find SQL / query fingerprint
        ↓
pg_stat_statements
        ↓
Check calls + total/mean time
        ↓
EXPLAIN (ANALYZE, BUFFERS)
        ↓
Inspect estimates
        ↓
Inspect scans
        ↓
Inspect joins
        ↓
Inspect sorts/hashes
        ↓
Inspect I/O
        ↓
Check statistics
        ↓
Check indexes
        ↓
Check locks/concurrency
        ↓
Check memory/parallelism
        ↓
Apply targeted fix
        ↓
Benchmark
        ↓
Deploy
        ↓
Monitor
```

That is the mindset interviewers usually want to see.

---

# 58. Useful Monitoring Views

Know these:

```text
pg_stat_activity
pg_stat_database
pg_stat_user_tables
pg_stat_user_indexes
pg_stat_statements
pg_locks
pg_stats
```

Examples:

### Active sessions

```sql
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    query_start,
    query
FROM pg_stat_activity
WHERE state <> 'idle';
```

### Dead tuples

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### Index usage

```sql
SELECT
    relname,
    indexrelname,
    idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan;
```

---

# 59. Common Performance Mistakes

Avoid these:

```text
❌ Increase every memory parameter
❌ Add indexes to every column
❌ Assume every Seq Scan is bad
❌ Assume an unused index should immediately be dropped
❌ Run VACUUM FULL casually in production
❌ Tune PostgreSQL before tuning obviously bad SQL
❌ Ignore row-estimation errors
❌ Ignore locks
❌ Ignore application connection counts
❌ Set huge work_mem globally
❌ Force indexes/plans without understanding planner decisions
❌ Benchmark only once
```

Instead:

```text
Measure → understand → change → measure again
```

---

# 60. Interview Questions You Should Be Ready For

### Q1. What is the difference between `EXPLAIN` and `EXPLAIN ANALYZE`?

`EXPLAIN` shows the estimated execution plan without executing the statement.

`EXPLAIN ANALYZE` executes it and adds actual timing, rows, loops, and other runtime information.

---

### Q2. Is a Sequential Scan bad?

No.

A sequential scan can be the best plan when PostgreSQL needs a large percentage of a table or when the table is small.

---

### Q3. Why would PostgreSQL not use an index?

Common reasons include low selectivity, stale statistics, incompatible predicates, small tables, poor index design, or the planner estimating that scanning the table is cheaper.

---

### Q4. What do you check first in an execution plan?

I normally check:

```text
execution time
estimated vs actual rows
expensive nodes
loops
scan methods
join methods
sort/hash spills
buffers
rows removed by filters
```

---

### Q5. How do you identify expensive queries?

A strong answer:

> I normally start with `pg_stat_statements` because it shows query frequency, total execution time and average execution time. Then I take the important query and analyze it using `EXPLAIN (ANALYZE, BUFFERS)`.

---

### Q6. What does `work_mem` control?

Memory available to individual query operations such as sorts and hashes.

A query can have multiple such operations, so setting `work_mem` too high globally can create serious memory pressure.

---

### Q7. What is `effective_cache_size`?

A planner estimate of how much cache is likely available across PostgreSQL and the operating system.

It doesn't allocate that memory.

---

### Q8. Nested Loop vs Hash Join?

**Nested Loop:** usually good when the outer result is small and the inner lookup is efficient, often via an index.

**Hash Join:** often good for larger equality joins where PostgreSQL can build a hash table efficiently.

---

### Q9. How do stale statistics affect performance?

They can cause incorrect cardinality estimates, which can lead to poor join order, scan type, join algorithm, memory assumptions, and parallelism decisions.

---

### Q10. How do you tune a slow PostgreSQL query?

A strong interview answer:

> First, I identify the SQL using monitoring or `pg_stat_statements`. Then I run `EXPLAIN (ANALYZE, BUFFERS)` in a safe environment and compare estimated rows with actual rows. I check scan methods, join algorithms, loops, sort or hash spills, and buffer activity. Then I check statistics and index design and look for query rewrite opportunities. I also verify whether the latency is actually caused by locks, I/O, memory pressure, or concurrency. I make one targeted change and compare the execution plan and runtime again.

---

# 61. Real-Time Scenario

### Interviewer:

> A query suddenly changed from 200 ms to 20 seconds. What will you do?

A good troubleshooting sequence is:

```text
1. Check whether SQL changed.
2. Check execution plan.
3. Compare current plan with known good plan if available.
4. Check estimated vs actual rows.
5. Check when ANALYZE/autovacuum last ran.
6. Check data distribution changes.
7. Check table/index growth.
8. Check locks and waits.
9. Check disk/I/O pressure.
10. Check temp-file activity.
11. Check whether plan changed from:
       Index Scan → Seq Scan
       Hash Join → Nested Loop
       etc.
12. Check configuration/environment changes.
13. Apply targeted correction.
14. Measure again.
```

Don't immediately say:

> I'll create an index.

The index may have nothing to do with the regression.

---

# 62. Architecture-Level View

For a production PostgreSQL system, think of performance as multiple layers:

```text
                    APPLICATION
                         │
              SQL / ORM / Transactions
                         │
                         ▼
                 CONNECTION POOL
                         │
                         ▼
                POSTGRESQL PLANNER
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
          Statistics   Indexes    Cost Model
             │           │           │
             └───────────┼───────────┘
                         ▼
                  EXECUTION PLAN
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
        Scans           Joins       Sort / Hash
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                 Memory / Parallelism
                         │
                         ▼
                Buffer Cache / OS
                         │
                         ▼
                     STORAGE
```

A performance problem can exist at **any layer**.

That is why good troubleshooting is not:

```text
Slow SQL → create index
```

It is:

```text
Slow SQL
   ↓
Where is the time spent?
   ↓
Why did PostgreSQL choose this plan?
   ↓
What is the smallest safe change?
   ↓
Did the change actually improve production behavior?
```

---

# 63. Topic 14 Cheat Sheet

```text
Slow PostgreSQL Query
        │
        ▼
pg_stat_statements
        │
        ▼
EXPLAIN (ANALYZE, BUFFERS)
        │
        ├── Estimated rows ≠ Actual rows?
        │       └── Statistics / data distribution
        │
        ├── Seq Scan?
        │       └── Selectivity / index / query predicate
        │
        ├── Bad Nested Loop?
        │       └── Estimates / indexes / join strategy
        │
        ├── Sort to disk?
        │       └── work_mem / query / index
        │
        ├── Hash batches/temp?
        │       └── memory / cardinality
        │
        ├── Huge buffer reads?
        │       └── I/O / query / indexes / bloat
        │
        ├── High loops?
        │       └── repeated inner operation
        │
        ├── Waiting?
        │       └── Locks / concurrency
        │
        └── Large table?
                └── Partitioning / indexing / vacuum
```

## Core commands to remember

```sql
-- Planned execution
EXPLAIN
SELECT ...;

-- Real execution
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;

-- Refresh statistics
ANALYZE table_name;

-- Sessions
SELECT * FROM pg_stat_activity;

-- Locks
SELECT * FROM pg_locks;

-- Table statistics
SELECT * FROM pg_stat_user_tables;

-- Index statistics
SELECT * FROM pg_stat_user_indexes;

-- Column statistics
SELECT * FROM pg_stats;

-- Query statistics
SELECT * FROM pg_stat_statements;
```

The key takeaway for **Topic 14** is:

> **PostgreSQL performance tuning is mainly about understanding why the optimizer chose a particular execution plan.
Start with evidence from `pg_stat_statements` and `EXPLAIN (ANALYZE, BUFFERS)`, validate cardinality estimates, scans, joins, memory, I/O and waits, and only then change SQL, indexes, statistics, or server configuration.**
