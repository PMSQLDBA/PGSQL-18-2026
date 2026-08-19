## Topic 19 — PostgreSQL Maintenance & Housekeeping

PostgreSQL needs regular maintenance to keep query performance stable, prevent table/index growth, maintain accurate optimizer statistics, and control disk usage. 

The main areas are **VACUUM/ANALYZE, REINDEX, bloat management, statistics maintenance, log cleanup, and scheduled housekeeping**.

### 1. VACUUM oversight

PostgreSQL uses MVCC, so an `UPDATE` or `DELETE` normally leaves an old row version behind. `VACUUM` cleans up these dead tuples so their space can be reused.

```sql
VACUUM table_name;
```

For troubleshooting or verification:

```sql
VACUUM (VERBOSE, ANALYZE) table_name;
```

In most environments, **autovacuum should remain enabled**. Manual vacuuming is generally used when autovacuum cannot keep up, after unusually large data changes, or during controlled maintenance.

Useful monitoring query:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Pay particular attention to tables where `n_dead_tup` keeps increasing or `last_autovacuum` is unexpectedly old.

A related concern is **transaction ID wraparound**. PostgreSQL must eventually freeze old transaction IDs, so vacuuming is also a database-safety requirement, not only a performance activity.

---

### 2. ANALYZE and optimizer statistics

`ANALYZE` samples table data and updates statistics used by the PostgreSQL query planner.

```sql
ANALYZE table_name;
```

Or:

```sql
VACUUM ANALYZE table_name;
```

Without good statistics, the optimizer may incorrectly estimate row counts and choose inefficient plans such as a nested-loop join instead of a hash join.

Check when statistics were last refreshed:

```sql
SELECT
    schemaname,
    relname,
    last_analyze,
    last_autoanalyze,
    analyze_count,
    autoanalyze_count
FROM pg_stat_user_tables
ORDER BY last_autoanalyze NULLS FIRST;
```

For columns with unusual or highly skewed data distributions, the default statistics target may not be sufficient.

```sql
ALTER TABLE orders
ALTER COLUMN customer_id
SET STATISTICS 500;
```

Then:

```sql
ANALYZE orders;
```

Increasing the statistics target can improve cardinality estimates, but it also increases statistics collection and planning overhead.

---

### 3. REINDEX

Indexes can become inefficient because of heavy updates, page splits, or index bloat.

Basic syntax:

```sql
REINDEX INDEX idx_orders_customer;
```

Table:

```sql
REINDEX TABLE orders;
```

Database:

```sql
REINDEX DATABASE mydb;
```

For production systems, avoiding long blocking operations is usually important. PostgreSQL provides concurrent reindexing:

```sql
REINDEX INDEX CONCURRENTLY idx_orders_customer;
```

or:

```sql
REINDEX TABLE CONCURRENTLY orders;
```

Typical reasons to consider REINDEX include unusually large indexes, degraded index performance, index corruption, or maintenance following large data changes.

Do **not** schedule REINDEX blindly every day or every week. Measure first and rebuild only where there is evidence that it is useful.

---

### 4. Table and index bloat

Bloat occurs when database objects consume significantly more disk space than their active data requires.

Common causes include:

* Frequent `UPDATE` and `DELETE` activity
* Long-running transactions preventing vacuum cleanup
* Autovacuum settings that are too conservative
* Large batch modifications
* Repeated index page splits
* Poor table-specific vacuum tuning

Quick table activity check:

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    n_tup_ins,
    n_tup_upd,
    n_tup_del
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Check object sizes:

```sql
SELECT
    relname,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_indexes_size(relid)) AS index_size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

The response depends on the type and severity of bloat.

For normal dead tuples:

```sql
VACUUM table_name;
```

For index bloat:

```sql
REINDEX INDEX CONCURRENTLY index_name;
```

For severe table bloat:

```sql
VACUUM FULL table_name;
```

`VACUUM FULL` is much more disruptive because it rewrites the table and requires stronger locking. It should therefore be treated as planned maintenance rather than routine housekeeping.

---

### 5. Autovacuum monitoring

Autovacuum is one of the most important PostgreSQL background maintenance mechanisms.

Important configuration areas include:

```text
autovacuum
autovacuum_max_workers
autovacuum_naptime
autovacuum_vacuum_threshold
autovacuum_vacuum_scale_factor
autovacuum_analyze_threshold
autovacuum_analyze_scale_factor
```

Large and heavily updated tables may need table-specific settings.

Example:

```sql
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_analyze_scale_factor = 0.01
);
```

This is often preferable to aggressively changing settings for every table in the database.

Check active vacuum operations:

```sql
SELECT *
FROM pg_stat_progress_vacuum;
```

Also investigate long-running transactions because they can prevent vacuum from removing old row versions:

```sql
SELECT
    pid,
    usename,
    state,
    xact_start,
    now() - xact_start AS transaction_age,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

---

### 6. Statistics maintenance

Statistics maintenance is broader than simply running `ANALYZE`.

DBAs should watch for:

**Stale table statistics**

```sql
SELECT
    relname,
    last_analyze,
    last_autoanalyze,
    n_mod_since_analyze
FROM pg_stat_user_tables
ORDER BY n_mod_since_analyze DESC;
```

A high `n_mod_since_analyze` value means many rows have changed since statistics were last collected.

**Column statistics**

```sql
SELECT *
FROM pg_stats
WHERE tablename = 'orders';
```

For correlated columns, PostgreSQL can also maintain extended statistics.

Example:

```sql
CREATE STATISTICS orders_customer_region_stats
ON customer_id, region_id
FROM orders;

ANALYZE orders;
```

This can help when multiple columns are related and the optimizer's normal independence assumptions produce bad estimates.

---

### 7. Maintenance of long-running transactions

One overlooked housekeeping problem is sessions leaving transactions open for long periods.

Particularly problematic:

```text
idle in transaction
```

Find them with:

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    xact_start,
    now() - xact_start AS transaction_age,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

These sessions can retain old row versions and cause table growth even when autovacuum is running correctly.

---

### 8. Log cleanup

PostgreSQL logs can eventually consume large amounts of disk space.

Housekeeping should define:

* Log rotation frequency
* Maximum retention period
* Compression/archive strategy
* Centralized log collection
* Disk-space alerts

Typical configuration areas include:

```text
logging_collector
log_directory
log_filename
log_rotation_age
log_rotation_size
log_truncate_on_rotation
```

The operating system or centralized logging platform may also manage retention.

A typical policy could be:

```text
Active PostgreSQL logs: 7 days
Compressed local logs: 30 days
Central log platform: 90 days
Security/audit logs: according to compliance requirements
```

The exact retention period should follow operational and compliance requirements rather than being chosen arbitrarily.

---

### 9. Temporary files and disk monitoring

Maintenance should also watch temporary file generation and filesystem utilization.

Useful database-size queries:

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname))
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

Largest tables:

```sql
SELECT
    schemaname,
    relname,
    pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;
```

Unexpected growth can indicate bloat, logging problems, workload changes, WAL retention issues, or backup/replication problems.

---

## Routine maintenance schedule

A practical production schedule might look like this:

| Frequency      | Activity                                                      |
| -------------- | ------------------------------------------------------------- |
| Continuous     | Autovacuum and autoanalyze                                    |
| Every 5–15 min | Disk, WAL, replication, connection and transaction monitoring |
| Daily          | Review dead tuples and failed/slow autovacuum activity        |
| Daily          | Check long-running and idle transactions                      |
| Daily          | Check disk and database growth                                |
| Daily          | Verify log rotation                                           |
| Weekly         | Review largest/growing tables and indexes                     |
| Weekly         | Review statistics freshness                                   |
| Weekly         | Investigate possible table/index bloat                        |
| Monthly        | Review autovacuum tuning for high-write tables                |
| Monthly        | Review log retention and disk capacity                        |
| As needed      | `REINDEX CONCURRENTLY`                                        |
| As needed      | Manual `VACUUM ANALYZE`                                       |
| Rare/planned   | `VACUUM FULL`                                                 |

### Production DBA approach

A good PostgreSQL housekeeping process follows this flow:

**Monitor → identify abnormal growth or stale statistics → find the cause → perform targeted maintenance → verify the result.**

For example:

```text
High dead tuples
        ↓
Check last autovacuum
        ↓
Check long-running transactions
        ↓
Check autovacuum thresholds
        ↓
Tune table-specific autovacuum if needed
        ↓
VACUUM / ANALYZE
        ↓
Verify dead tuples and query performance
```

The key point for interviews and real production work is that PostgreSQL maintenance should **not** mean scheduling `VACUUM FULL` and `REINDEX` against every database. Autovacuum and autoanalyze handle normal housekeeping. DBA work is mainly about monitoring them, tuning problem tables, identifying bloat or stale statistics, controlling logs and disk usage, and using disruptive maintenance only when evidence shows it is necessary.
