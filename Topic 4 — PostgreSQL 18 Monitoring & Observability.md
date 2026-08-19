# Topic 4 — PostgreSQL 18 Monitoring & Observability

PostgreSQL monitoring is not just checking whether the database is **up or down**. A production monitoring design should tell you:

**What is happening now, what has been happening over time, what is becoming abnormal, and where the bottleneck is.**

PostgreSQL 18 provides extensive built-in monitoring through its cumulative statistics system, `pg_stat_*` views, wait events, progress views, server logs, and extensions such as `pg_stat_statements`. ([PostgreSQL][1])

[PostgreSQL 18 — Monitoring Database Activity](https://www.postgresql.org/docs/18/monitoring.html?utm_source=chatgpt.com)

---

## 1. Monitoring Architecture

A good PostgreSQL monitoring architecture has several layers:

```text
                    PostgreSQL 18
                         │
          ┌──────────────┼───────────────┐
          │              │               │
          ▼              ▼               ▼
     Statistics        Logs          OS Metrics
     pg_stat_*      PostgreSQL      CPU / RAM /
     pg_statio_*       Logs         Disk / Network
          │              │               │
          └──────────────┼───────────────┘
                         ▼
                 Monitoring Platform
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
          Dashboards              Alerts
              │                     │
              ▼                     ▼
          DBA / SRE            DBA / On-call
```

You should monitor **database metrics and operating-system metrics together**. A query showing high I/O inside PostgreSQL is much more useful when you can correlate it with disk latency and utilization at the OS/storage layer.

---

# 2. Main PostgreSQL Monitoring Sources

For PostgreSQL 18, the main monitoring sources are:

| Monitoring source      | Purpose                                           |
| ---------------------- | ------------------------------------------------- |
| `pg_stat_activity`     | Sessions and current activity                     |
| `pg_stat_database`     | Database-level workload statistics                |
| `pg_stat_user_tables`  | Table activity                                    |
| `pg_stat_user_indexes` | Index usage                                       |
| `pg_statio_*`          | I/O-related statistics                            |
| `pg_stat_io`           | Cluster-wide I/O activity                         |
| `pg_stat_wal`          | WAL statistics                                    |
| `pg_stat_replication`  | Physical replication                              |
| `pg_stat_wal_receiver` | Standby WAL receiver                              |
| `pg_stat_archiver`     | WAL archiving                                     |
| `pg_stat_ssl`          | SSL connection information                        |
| `pg_locks`             | Lock information                                  |
| `pg_stat_progress_*`   | Long-operation progress                           |
| `pg_stat_statements`   | SQL planning/execution statistics                 |
| PostgreSQL logs        | Errors, warnings, slow queries, checkpoints, etc. |
| OS metrics             | CPU, RAM, disk, filesystem and network            |

PostgreSQL 18's monitoring chapter exposes dedicated statistics views for replication, slots, WAL receiver, subscriptions, SSL, archiving, I/O, background writer, checkpointer, WAL and databases. ([PostgreSQL][1])

---

# 3. `pg_stat_activity` — Most Important Starting Point

For day-to-day troubleshooting, one of the first places a DBA should look is:

```sql
SELECT *
FROM pg_stat_activity;
```

A more useful query:

```sql
SELECT
    pid,
    datname,
    usename,
    application_name,
    client_addr,
    backend_start,
    xact_start,
    query_start,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
ORDER BY query_start;
```

This answers:

```text
Who is connected?
        │
        ├── Which database?
        ├── Which user?
        ├── Which application?
        ├── Which client?
        ├── What query?
        ├── How long?
        ├── What state?
        └── What is it waiting for?
```

The collection of current session activity is controlled by `track_activities`, which is enabled by default. PostgreSQL also restricts visibility of other sessions' detailed information based on privileges. ([PostgreSQL][2])

---

# 4. Understanding Session States

Important `pg_stat_activity.state` values include:

```text
active
idle
idle in transaction
idle in transaction (aborted)
disabled
```

### `active`

The backend is executing a query.

### `idle`

The backend is connected but waiting for the client.

### `idle in transaction`

The client opened a transaction but is currently not executing anything.

For example:

```text
Application
    │
    ▼
BEGIN
    │
    ▼
UPDATE
    │
    ▼
Application waits
    │
    ▼
Transaction remains open
```

Long-running `idle in transaction` sessions deserve attention because open transactions can interfere with vacuum cleanup and can retain locks/resources.

---

# 5. Finding Active Queries

A basic production query:

```sql
SELECT
    pid,
    usename,
    datname,
    application_name,
    now() - query_start AS duration,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND pid <> pg_backend_pid()
ORDER BY duration DESC;
```

This gives you the longest currently active queries first.

Conceptually:

```text
pg_stat_activity
       │
       ▼
state = active
       │
       ▼
Order by duration
       │
       ▼
Potentially expensive queries
```

---

# 6. Long-Running Queries

For example, queries running longer than five minutes:

```sql
SELECT
    pid,
    usename,
    datname,
    now() - query_start AS duration,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND query_start < now() - interval '5 minutes'
ORDER BY query_start;
```

But don't automatically terminate every long query.

A five-minute query might be:

```text
Bad OLTP query
```

or legitimately:

```text
Large reporting query
Backup operation
Maintenance operation
CREATE INDEX
VACUUM
ETL workload
```

Monitoring provides evidence; the DBA still needs workload context.

---

# 7. Long-Running Transactions

A transaction can be more dangerous operationally than a merely long-running statement.

Monitor:

```sql
SELECT
    pid,
    usename,
    datname,
    state,
    now() - xact_start AS transaction_age,
    now() - query_start AS query_age,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Architecture:

```text
Long Transaction
      │
      ├── May retain locks
      ├── Holds old snapshot
      ├── Can affect VACUUM cleanup
      └── Can contribute to table bloat
```

This should be one of your standard alerts.

---

# 8. Idle-in-Transaction Monitoring

Find them:

```sql
SELECT
    pid,
    usename,
    datname,
    client_addr,
    application_name,
    now() - xact_start AS transaction_age,
    now() - state_change AS idle_duration,
    query
FROM pg_stat_activity
WHERE state LIKE 'idle in transaction%'
ORDER BY xact_start;
```

For application environments, PostgreSQL also provides:

```text
idle_in_transaction_session_timeout
```

which can terminate sessions that remain idle inside an open transaction beyond a configured period.

Do not set the timeout blindly; understand application behavior first.

---

# 9. Connection Monitoring

Count connections:

```sql
SELECT count(*)
FROM pg_stat_activity;
```

By database:

```sql
SELECT
    datname,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY datname
ORDER BY connections DESC;
```

By application:

```sql
SELECT
    application_name,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY application_name
ORDER BY connections DESC;
```

By user:

```sql
SELECT
    usename,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY usename
ORDER BY connections DESC;
```

---

# 10. Connection Utilization

You should monitor connections relative to the configured capacity.

Check:

```sql
SHOW max_connections;
```

Then compare against current usage:

```sql
SELECT count(*)
FROM pg_stat_activity;
```

Conceptually:

```text
max_connections = 500

Current = 200
        │
        ▼
40% utilized

Current = 480
        │
        ▼
96% utilized
        │
        ▼
Potential incident
```

But monitoring only the total is insufficient.

You also need:

```text
Connections by application
Connections by user
Connections by database
Active vs idle
Connection creation rate
Connection lifetime
```

A sudden increase may indicate a connection leak or pooler problem.

---

# 11. Wait Events

Wait events are one of PostgreSQL's most powerful troubleshooting mechanisms.

If a backend is not progressing because it is waiting for something, PostgreSQL can expose:

```text
wait_event_type
wait_event
```

through `pg_stat_activity`.

Example:

```sql
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;
```

Conceptually:

```text
Slow Query
    │
    ▼
Is it actually using CPU?
    │
   No
    │
    ▼
What is it waiting for?
    │
    ├── Lock?
    ├── I/O?
    ├── WAL?
    ├── Client?
    ├── IPC?
    └── Other resource?
```

That is much better than assuming every slow query is a CPU problem.

---

# 12. Important Wait Event Types

You may encounter wait categories such as:

```text
Lock
LWLock
IO
Client
IPC
Activity
Timeout
Extension
```

For example:

```text
wait_event_type = Lock
```

usually means investigate blocking.

```text
wait_event_type = IO
```

means investigate which I/O operation is involved and correlate it with PostgreSQL and storage metrics.

```text
wait_event_type = Client
```

may indicate PostgreSQL is waiting for the application/client rather than the database engine itself.

---

# 13. Lock Monitoring

`pg_locks` provides information about locks held and awaited by PostgreSQL processes.

A quick blocking check:

```sql
SELECT
    pid,
    pg_blocking_pids(pid) AS blocking_pids,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

Architecture:

```text
Session A
   │
   │ Holds Lock
   ▼
Object / Row
   ▲
   │ Wants Lock
Session B
   │
   ▼
WAITING
```

The critical questions are:

```text
Who is blocked?
Who is blocking?
What SQL is involved?
How long?
Which application?
Which transaction started first?
```

---

# 14. Blocking Tree

A production incident may look like:

```text
Session 100
   │
   │ blocks
   ▼
Session 200
   │
   ├── blocks Session 300
   ├── blocks Session 400
   └── blocks Session 500
```

The session causing the most visible problems may therefore **not** be the root blocker.

Your monitoring platform should ideally show blocking chains rather than just "five sessions waiting."

---

# 15. `pg_stat_database`

`pg_stat_database` provides database-level cumulative statistics.

Example:

```sql
SELECT
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit,
    tup_returned,
    tup_fetched,
    tup_inserted,
    tup_updated,
    tup_deleted,
    deadlocks,
    temp_files,
    temp_bytes
FROM pg_stat_database
ORDER BY datname;
```

This is excellent for workload trends.

You can monitor:

```text
Transactions
Rollbacks
Connections
Rows processed
Deadlocks
Temporary files
Temporary bytes
Block activity
```

PostgreSQL 18 exposes database statistics as part of its cumulative statistics system. ([PostgreSQL][1])

---

# 16. Transaction Rate

The cumulative counters:

```text
xact_commit
xact_rollback
```

can be sampled over time.

For example:

```text
12:00 xact_commit = 1,000,000
12:01 xact_commit = 1,030,000

Difference = 30,000

≈ 500 commits/sec
```

This is how monitoring systems derive rates from cumulative PostgreSQL counters.

You normally care more about:

```text
Transactions/sec
```

than the lifetime total since the statistics were reset.

---

# 17. Rollback Monitoring

A high or suddenly increasing rollback rate can indicate:

```text
Application errors
Deadlocks
Constraint failures
Transaction conflicts
Application logic problems
```

Monitor the relationship between:

```text
xact_commit
       vs
xact_rollback
```

But don't treat every rollback as an incident. Some applications legitimately use transactions that are rolled back.

---

# 18. Table Monitoring

Important views include:

```text
pg_stat_user_tables
pg_stat_all_tables
```

Example:

```sql
SELECT
    schemaname,
    relname,
    seq_scan,
    idx_scan,
    n_live_tup,
    n_dead_tup,
    n_mod_since_analyze,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

This gives visibility into:

```text
Sequential scans
Index scans
Live tuples
Dead tuples
VACUUM history
ANALYZE history
```

---

# 19. Dead Tuple Monitoring

Example:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

High dead-tuple counts may point toward:

```text
Heavy UPDATE/DELETE workload
Autovacuum not keeping up
Long transactions
Poor table-specific autovacuum settings
```

But `n_dead_tup` is an estimate, so it should be treated as a monitoring signal rather than an exact physical bloat measurement.

---

# 20. Index Usage Monitoring

Use:

```text
pg_stat_user_indexes
```

Example:

```sql
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan;
```

This helps identify indexes with little or no observed use.

However:

```text
idx_scan = 0
```

does **not automatically mean "drop the index."**

It might support:

```text
Rare month-end processing
Foreign-key-related operations
Emergency queries
Uniqueness constraints
Recently reset statistics
```

Always understand the workload and statistics age before removing an index.

---

# 21. PostgreSQL I/O Monitoring

PostgreSQL 18 provides `pg_stat_io`, which gives cluster-wide I/O statistics.

This allows monitoring across backend/process types and I/O contexts.

PostgreSQL 18 specifically enhanced `pg_stat_io` with byte counters such as:

```text
read_bytes
write_bytes
extend_bytes
```

and added WAL I/O activity to the view. ([PostgreSQL][3])

This is particularly useful for distinguishing:

```text
How many I/O operations?
        from
How much data was actually read/written?
```

---

# 22. PostgreSQL 18 Per-Backend I/O

PostgreSQL 18 also added **per-backend I/O statistics** through:

```text
pg_stat_get_backend_io()
```

and provides:

```text
pg_stat_reset_backend_stats()
```

for resetting per-backend statistics. ([PostgreSQL][3])

This is a useful PostgreSQL 18-specific monitoring improvement because you can investigate I/O behavior at a more granular backend level.

---

# 23. WAL Monitoring

Important WAL monitoring includes:

```text
pg_stat_wal
pg_stat_io
pg_stat_archiver
```

You want to understand:

```text
WAL generation rate
WAL write activity
Archive success/failure
Replication consumption
WAL storage growth
```

A sudden increase in WAL generation can indicate:

```text
Large batch operation
Bulk load
Heavy UPDATE activity
Index creation
Maintenance
Application workload spike
```

---

# 24. Archive Monitoring

For environments using WAL archiving:

```sql
SELECT *
FROM pg_stat_archiver;
```

Important information includes:

```text
Archived WAL count
Failed WAL archive count
Last archived WAL
Last archive time
Last failed WAL
Last failure time
```

This should be monitored aggressively if your backup/PITR design depends on WAL archiving.

An archive failure can become a **recovery risk**, not merely a monitoring warning.

---

# 25. Replication Monitoring

On a primary:

```sql
SELECT *
FROM pg_stat_replication;
```

This exposes information about connected physical standbys.

A DBA normally cares about:

```text
Standby connected?
Streaming state?
WAL sent?
WAL written?
WAL flushed?
WAL replayed?
Replication lag?
```

PostgreSQL 18 provides separate monitoring views for physical replication, replication slots, WAL receivers, recovery prefetch, logical subscriptions and subscription statistics. ([PostgreSQL][1])

We will cover these deeply in **Topic 5 — PostgreSQL Replication**.

---

# 26. Replication Slot Monitoring

Use:

```sql
SELECT *
FROM pg_replication_slots;
```

Replication slots are operationally important because an inactive consumer can cause PostgreSQL to retain resources such as WAL.

Conceptually:

```text
Replication Slot
      │
      ▼
Consumer stops
      │
      ▼
Required WAL retained
      │
      ▼
pg_wal grows
      │
      ▼
Filesystem risk
```

Therefore:

```text
Inactive replication slot
```

should be part of production monitoring.

---

# 27. SSL Monitoring

PostgreSQL exposes:

```text
pg_stat_ssl
```

which can be joined with session activity.

Example:

```sql
SELECT
    a.pid,
    a.usename,
    a.client_addr,
    s.ssl,
    s.version,
    s.cipher
FROM pg_stat_activity a
JOIN pg_stat_ssl s
  ON a.pid = s.pid;
```

This helps validate whether database connections are actually using SSL/TLS as expected.

---

# 28. `pg_stat_statements`

For SQL workload analysis, `pg_stat_statements` is one of the most important PostgreSQL extensions.

It tracks planning and execution statistics for SQL statements executed by the server. ([PostgreSQL][4])

[PostgreSQL 18 — pg_stat_statements](https://www.postgresql.org/docs/18/pgstatstatements.html?utm_source=chatgpt.com)

Typical architecture:

```text
Applications
     │
     ▼
SQL Statements
     │
     ▼
PostgreSQL
     │
     ▼
pg_stat_statements
     │
     ├── Calls
     ├── Execution time
     ├── Planning time
     ├── Rows
     ├── I/O-related stats
     └── WAL-related stats
```

---

# 29. Enabling `pg_stat_statements`

The module requires shared memory, so PostgreSQL documentation requires it to be included in:

```conf
shared_preload_libraries = 'pg_stat_statements'
```

Adding or removing it there requires a server restart. Query identifiers must also be available; PostgreSQL can enable them through `compute_query_id`. ([PostgreSQL][4])

Then, in databases where you want the SQL objects:

```sql
CREATE EXTENSION pg_stat_statements;
```

Check:

```sql
SELECT *
FROM pg_stat_statements
LIMIT 10;
```

---

# 30. Finding Expensive SQL

A useful starting point:

```sql
SELECT
    queryid,
    calls,
    total_exec_time,
    mean_exec_time,
    rows,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

This answers:

> Which SQL statements consumed the most total execution time?

But this is different from:

> Which individual query execution is slowest?

For example:

```text
Query A
1 ms × 10,000,000 calls
      =
10,000 seconds total

Query B
30 seconds × 10 calls
      =
300 seconds total
```

Query A may have a much larger overall impact even though every individual call looks fast.

---

# 31. Different Ways to Find Problem SQL

You should look at SQL from several perspectives:

### Highest total workload

```sql
ORDER BY total_exec_time DESC
```

### Highest average execution time

```sql
ORDER BY mean_exec_time DESC
```

### Most frequently called

```sql
ORDER BY calls DESC
```

### Highest temporary I/O

Look at temporary block activity.

### Highest WAL generation

Look at WAL-related statistics where appropriate.

This prevents the common mistake of defining "slow query" only by average duration.

---

# 32. PostgreSQL Logging

Statistics tell you **what is happening quantitatively**.

Logs tell you **what events occurred**.

PostgreSQL 18 can log to destinations including:

```text
stderr
csvlog
jsonlog
syslog
```

and Windows `eventlog`. CSV and JSON logging require the logging collector. ([PostgreSQL][5])

[PostgreSQL 18 — Error Reporting and Logging](https://www.postgresql.org/docs/18/runtime-config-logging.html?utm_source=chatgpt.com)

---

# 33. Important Logging Parameters

Common monitoring-related parameters include:

```conf
logging_collector
log_destination
log_directory
log_filename

log_min_messages
log_min_error_statement

log_min_duration_statement
log_statement

log_connections
log_disconnections

log_lock_waits
log_checkpoints
log_autovacuum_min_duration

log_temp_files
log_line_prefix
```

PostgreSQL 18 documents these under Error Reporting and Logging. ([PostgreSQL][5])

---

# 34. Slow Query Logging

For example:

```conf
log_min_duration_statement = '1s'
```

means completed SQL statements taking at least one second are logged.

PostgreSQL specifically notes that this can help find unoptimized queries, while also warning that statement logging can add overhead. ([PostgreSQL][5])

Do not automatically use:

```conf
log_min_duration_statement = 0
```

in a high-volume production environment without understanding the impact, because that logs every statement's duration and text.

---

# 35. Lock-Wait Logging

An especially useful setting is:

```conf
log_lock_waits = on
```

When enabled, PostgreSQL can log sessions that wait longer than `deadlock_timeout` for locks. PostgreSQL specifically identifies this as useful for determining whether lock waits are hurting performance. ([PostgreSQL][5])

This provides a useful combination:

```text
pg_stat_activity
        +
pg_locks
        +
log_lock_waits
        =
Better lock troubleshooting
```

---

# 36. Autovacuum Logging

For troubleshooting vacuum activity:

```conf
log_autovacuum_min_duration = '5s'
```

is an example threshold.

PostgreSQL 18 can log autovacuum operations that exceed the configured duration and can also report certain skipped autovacuum operations when this setting is enabled. ([PostgreSQL][5])

This helps answer:

```text
Is autovacuum running?
How long does it take?
Which tables?
Is it being blocked?
```

---

# 37. Checkpoint Logging

PostgreSQL provides:

```conf
log_checkpoints = on
```

and PostgreSQL 18 has checkpoint logging enabled by default. Checkpoint log entries include information such as buffers written and time spent writing. ([PostgreSQL][5])

This is useful for investigating:

```text
I/O spikes
Checkpoint frequency
Write pressure
Storage latency
WAL/checkpoint tuning
```

---

# 38. Connection Logging

PostgreSQL 18 has detailed connection logging controls.

`log_connections` can record different parts of connection establishment, including:

```text
receipt
authentication
authorization
setup_durations
```

and `all` enables all of them. ([PostgreSQL][5])

This is useful when troubleshooting:

```text
Slow connection establishment
Authentication problems
Connection storms
Unexpected application connections
```

---

# 39. Log Line Prefix

A good log line should contain enough context to correlate an event.

PostgreSQL's documentation gives an example pattern such as:

```conf
log_line_prefix = '%m [%p] %q%u@%d/%a '
```

which can include information such as timestamp, PID, user, database and application name. ([PostgreSQL][5])

That makes logs far easier to investigate.

---

# 40. Progress Monitoring

PostgreSQL can report progress for several long-running operations.

PostgreSQL 18 provides progress reporting for:

```text
ANALYZE
CLUSTER
COPY
CREATE INDEX
VACUUM
BASE_BACKUP
```

([PostgreSQL][6])

Important views include:

```text
pg_stat_progress_analyze
pg_stat_progress_cluster
pg_stat_progress_copy
pg_stat_progress_create_index
pg_stat_progress_vacuum
pg_stat_progress_basebackup
```

---

# 41. Example — Monitor CREATE INDEX

Instead of seeing:

```text
CREATE INDEX is still running...
```

for two hours with no context, PostgreSQL can expose progress through:

```sql
SELECT *
FROM pg_stat_progress_create_index;
```

Conceptually:

```text
CREATE INDEX
      │
      ▼
Progress View
      │
      ├── Current phase
      ├── Blocks processed
      ├── Tuples processed
      └── Other progress data
```

This is very useful during production maintenance.

---

# 42. OS-Level Monitoring

PostgreSQL monitoring is incomplete without the OS.

You should monitor at least:

```text
CPU
Load
RAM
Swap
Disk latency
Disk throughput
Disk IOPS
Filesystem utilization
Network throughput
Network errors
Process count
```

For example:

```text
PostgreSQL reports high I/O waits
              +
Storage reports high latency
              =
Likely storage bottleneck
```

Whereas:

```text
Query is slow
       +
No I/O wait
       +
CPU saturated
       =
Different investigation
```

---

# 43. Filesystem Monitoring

Monitor all PostgreSQL-related filesystems, not just `PGDATA`.

For example:

```text
/data/postgresql     → Data
/wal/postgresql      → WAL
/archive             → WAL Archive
/backup              → Backups
/log                  → Logs
```

A DBA should alert on:

```text
Filesystem utilization
Growth rate
Inode exhaustion where relevant
Unexpected WAL growth
Archive accumulation
Log growth
Backup growth
```

A filesystem reaching 100% can become a database incident.

---

# 44. Memory Monitoring

Monitor:

```text
Server RAM
Available memory
Swap activity
PostgreSQL RSS/process memory
Shared memory
Connection count
Temporary-file generation
```

Do not conclude:

```text
95% RAM used = PostgreSQL memory problem
```

without understanding operating-system caching.

The important question is whether the system is under **memory pressure**, swapping excessively, being OOM-killed, or suffering workload degradation.

---

# 45. CPU Monitoring

High CPU can result from:

```text
Expensive SQL
High query concurrency
Poor execution plans
Large sorts/hashes
Parallel queries
Vacuum activity
Compression/encryption work
Background processes
```

A useful correlation is:

```text
OS CPU
   +
pg_stat_activity
   +
pg_stat_statements
   +
Query plans
```

No single metric tells the complete story.

---

# 46. Monitoring vs Observability

These terms are related but not identical.

### Monitoring

Answers known operational questions:

```text
Is PostgreSQL up?
CPU usage?
Connection count?
Replication lag?
Disk usage?
```

### Observability

Helps investigate unexpected behavior:

```text
Why did latency increase?
Which workload changed?
What are sessions waiting on?
Which SQL caused the I/O increase?
When did the behavior start?
```

For PostgreSQL, observability comes from combining:

```text
Metrics
  +
Logs
  +
SQL statistics
  +
Wait events
  +
OS/storage metrics
  +
Application context
```

---

# 47. Prometheus/Grafana Architecture

PostgreSQL itself does not require Prometheus or Grafana, but they are commonly used as an external monitoring stack.

A conceptual architecture is:

```text
             PostgreSQL
                 │
                 ▼
          Metrics Exporter
                 │
                 ▼
            Prometheus
                 │
          Time-Series Data
                 │
                 ▼
             Grafana
                 │
          ┌──────┴──────┐
          ▼             ▼
      Dashboard       Alerts
```

This external tooling is **not part of PostgreSQL core**. Your architecture should distinguish PostgreSQL-native telemetry from third-party monitoring.

---

# 48. Recommended Dashboard Structure

Rather than one huge dashboard, I would organize PostgreSQL monitoring into layers:

```text
PostgreSQL Overview
        │
        ├── Availability
        ├── Connections
        ├── TPS
        ├── Query latency
        ├── Locks
        ├── Database size
        └── Errors

Performance
        │
        ├── SQL workload
        ├── Wait events
        ├── CPU
        ├── Memory
        ├── I/O
        └── Temp activity

Storage
        │
        ├── Filesystem
        ├── Database growth
        ├── WAL
        └── Tables/indexes

Maintenance
        │
        ├── Vacuum
        ├── Analyze
        ├── Dead tuples
        └── Long transactions

HA / Replication
        │
        ├── Standby status
        ├── Replication lag
        ├── Slots
        └── WAL archive

Backup
        │
        ├── Last successful backup
        ├── Backup age
        ├── WAL archive
        └── Restore validation
```

---

# 49. Important Production Alerts

A production PostgreSQL environment should normally have alerts around areas such as:

| Area          | Alert concept                               |
| ------------- | ------------------------------------------- |
| Availability  | PostgreSQL unavailable                      |
| Connections   | Connection utilization approaching capacity |
| Transactions  | Abnormally old transaction                  |
| Sessions      | Long `idle in transaction`                  |
| Queries       | Abnormally long-running SQL                 |
| Locks         | Sustained blocking/lock waits               |
| Deadlocks     | Deadlock occurrence/rate                    |
| Storage       | Filesystem capacity                         |
| Storage       | Abnormal database growth                    |
| WAL           | Unexpected WAL growth                       |
| Archiving     | WAL archive failures                        |
| Replication   | Standby disconnected                        |
| Replication   | Lag beyond business tolerance               |
| Slots         | Inactive slot retaining excessive WAL       |
| Vacuum        | Autovacuum falling behind                   |
| Wraparound    | Transaction-ID age risk                     |
| Temporary I/O | Excessive temp-file generation              |
| Errors        | FATAL/PANIC or abnormal error rate          |
| Backup        | Backup missing/failed/too old               |

I would **not hard-code universal alert numbers** such as "replication lag > 10 seconds is always critical." Thresholds should come from your application's SLA/SLO, RPO/RTO and normal workload baseline.

---

# 50. Monitoring Baselines

One of the biggest mistakes is waiting for an incident before learning what "normal" looks like.

Build baselines for:

```text
Connections
Transactions/sec
Query latency
CPU
Memory
Read/write IOPS
Storage latency
WAL generation/hour
Database growth/day
Dead tuples
Vacuum duration
Replication lag
Temporary files
Backup duration
```

Then:

```text
Normal Baseline
      │
      ▼
Compare Current Behavior
      │
      ├── Normal variation
      │
      └── Abnormal deviation
              │
              ▼
             Alert
```

Baselines are often more useful than arbitrary static thresholds.

---

# 51. Recommended Monitoring Flow During an Incident

A DBA troubleshooting workflow can be:

```text
1. Is PostgreSQL available?
             │
             ▼
2. CPU / RAM / Disk / Network healthy?
             │
             ▼
3. Connection exhaustion?
             │
             ▼
4. Long-running transactions?
             │
             ▼
5. Blocking / lock waits?
             │
             ▼
6. What are sessions waiting on?
             │
             ▼
7. Which SQL consumes the workload?
             │
             ▼
8. Any abnormal I/O?
             │
             ▼
9. WAL / checkpoint issue?
             │
             ▼
10. Vacuum problem?
             │
             ▼
11. Replication healthy?
             │
             ▼
12. Errors in PostgreSQL logs?
             │
             ▼
13. Compare with historical baseline
```

This avoids immediately jumping to configuration changes.

---

# 52. DBA Monitoring Query Set

A PostgreSQL DBA should have a standard query pack covering at least:

```text
Current sessions
Long-running SQL
Long-running transactions
Idle-in-transaction sessions
Connection utilization
Blocking sessions
Lock tree
Database statistics
Largest databases
Largest tables
Dead tuples
Vacuum status
Index usage
I/O statistics
WAL statistics
Replication status
Replication slots
Archive status
Progress operations
pg_stat_statements
```

These queries can later be automated into dashboards and health checks.

---

# 53. Production Monitoring Architecture

A stronger enterprise architecture looks like:

```text
 ┌───────────────────────────────────────────────┐
 │                PostgreSQL 18                  │
 │                                               │
 │ pg_stat_*   pg_statio_*   pg_locks           │
 │ pg_stat_statements       Progress Views       │
 │ Logs                    Wait Events           │
 └──────────────────────┬────────────────────────┘
                        │
                        ▼
               Monitoring Collection
                        │
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
      PostgreSQL       OS           Storage
       Metrics       Metrics         Metrics
          │             │              │
          └─────────────┼──────────────┘
                        ▼
                Time-Series Store
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
         Dashboards              Alerts
             │                     │
             └──────────┬──────────┘
                        ▼
                   DBA / SRE
                        │
                        ▼
                Incident Response
```

---

# 54. PostgreSQL 18-Specific Monitoring Improvements

For PostgreSQL 18, pay particular attention to the expanded I/O observability.

PostgreSQL 18 added:

* Per-backend I/O statistics through `pg_stat_get_backend_io()`.
* Byte counters in `pg_stat_io`, including `read_bytes`, `write_bytes`, and `extend_bytes`.
* WAL I/O activity in `pg_stat_io`.
* Changes around where WAL I/O timing is reported. ([PostgreSQL][3])

These changes make `pg_stat_io` even more important when building PostgreSQL 18 monitoring dashboards.

---

## PostgreSQL 18 Official References

The primary reference for this topic is [PostgreSQL 18 — Monitoring Database Activity](https://www.postgresql.org/docs/18/monitoring.html?utm_source=chatgpt.com). It covers the cumulative statistics system, `pg_stat_activity`, replication monitoring, `pg_stat_io`, `pg_stat_database`, wait events and related monitoring views. ([PostgreSQL][1])

For statistics configuration such as `track_activities`, use [PostgreSQL 18 — Run-time Statistics](https://www.postgresql.org/docs/18/runtime-config-statistics.html?utm_source=chatgpt.com). ([PostgreSQL][2])

For SQL workload monitoring, use [PostgreSQL 18 — pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html?utm_source=chatgpt.com). PostgreSQL documents it as the supplied module for tracking SQL planning and execution statistics. ([PostgreSQL][4])

For logging, use [PostgreSQL 18 — Error Reporting and Logging](https://www.postgresql.org/docs/18/runtime-config-logging.html?utm_source=chatgpt.com). ([PostgreSQL][5])

For maintenance progress, use [PostgreSQL 18 — Progress Reporting](https://www.postgresql.org/docs/18/progress-reporting.html?utm_source=chatgpt.com). ([PostgreSQL][6])

**Tracker: 4 / 29 completed — 25 pending.**

**Next: Topic 5 — PostgreSQL Replication.** It will cover **physical streaming replication, WAL sender/receiver architecture, synchronous vs asynchronous replication, replication slots, standby configuration, monitoring, failover considerations, and logical replication in PostgreSQL 18**.

[1]: https://www.postgresql.org/docs/18/monitoring.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 27. Monitoring Database Activity"
[2]: https://www.postgresql.org/docs/18/runtime-config-statistics.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.9. Run-time Statistics"
[3]: https://www.postgresql.org/docs/18/release-18.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: E.5. Release 18"
[4]: https://www.postgresql.org/docs/current/pgstatstatements.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: F.32. pg_stat_statements — track statistics of SQL planning and execution"
[5]: https://www.postgresql.org/docs/18/runtime-config-logging.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.8. Error Reporting and Logging"
[6]: https://www.postgresql.org/docs/18/progress-reporting.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 27.4. Progress Reporting"
