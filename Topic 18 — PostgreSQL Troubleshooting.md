# Topic 18 — PostgreSQL Troubleshooting

PostgreSQL troubleshooting is not about memorizing commands. 

The key is to quickly identify **which resource or subsystem is causing the problem**, collect evidence, and then fix the root cause without creating another production issue.

This topic covers the main production problems:

**Locks → Blocking → Deadlocks → CPU → Memory → I/O → Connections → Crashes → Replication Lag → WAL → Storage**

---

## 1. Troubleshooting Approach

When someone says:

> "PostgreSQL is slow."

Do not immediately restart PostgreSQL or kill sessions.

First determine what is actually slow.

A practical troubleshooting flow is:

```text
Application reports slowness
        ↓
Check PostgreSQL availability
        ↓
Check active sessions
        ↓
Check locks/blocking
        ↓
Check long-running queries
        ↓
Check CPU
        ↓
Check memory
        ↓
Check disk I/O
        ↓
Check connection usage
        ↓
Check storage
        ↓
Check WAL / checkpoints
        ↓
Check replication
        ↓
Check PostgreSQL + OS logs
```

Useful first checks:

```sql
SELECT now();

SELECT version();

SELECT pg_postmaster_start_time();

SELECT count(*)
FROM pg_stat_activity;
```

Check active sessions:

```sql
SELECT
    pid,
    usename,
    datname,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

This is usually one of the first queries I run during an incident.

---

# 2. Locks

PostgreSQL uses locks to protect data consistency when multiple transactions access the same objects.

Locks themselves are normal.

The problem occurs when:

```text
Session A holds lock
        ↓
Session B needs same/incompatible lock
        ↓
Session B waits
        ↓
Application latency increases
```

Check locks:

```sql
SELECT
    pid,
    locktype,
    relation::regclass,
    mode,
    granted
FROM pg_locks
ORDER BY pid;
```

Important columns:

| Column   | Meaning                   |
| -------- | ------------------------- |
| pid      | Backend process           |
| locktype | Type of lock              |
| relation | Locked relation           |
| mode     | Lock mode                 |
| granted  | Whether lock was obtained |

`granted = false` means the backend is waiting for the lock.

---

# 3. Blocking Sessions

Blocking is more important operationally than simply seeing locks.

Example:

```text
PID 101
BEGIN;
UPDATE orders ...
        ↓
holds row locks
        ↓
PID 205
UPDATE same rows
        ↓
waiting
```

A very useful PostgreSQL function is:

```sql
pg_blocking_pids(pid)
```

Find blocked sessions:

```sql
SELECT
    pid AS blocked_pid,
    usename,
    pg_blocking_pids(pid) AS blocking_pids,
    wait_event_type,
    wait_event,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

Find blocker and blocked query together:

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocked.query AS blocked_query,
    blocker.pid AS blocker_pid,
    blocker.usename AS blocker_user,
    blocker.query AS blocker_query,
    now() - blocker.xact_start AS blocker_xact_age
FROM pg_stat_activity blocked
CROSS JOIN LATERAL unnest(pg_blocking_pids(blocked.pid)) AS b(blocker_pid)
JOIN pg_stat_activity blocker
    ON blocker.pid = b.blocker_pid;
```

Things to investigate:

```text
Is the blocker idle in transaction?
Is it a long-running transaction?
Is it application generated?
Is DDL running?
Is maintenance running?
How many sessions are waiting behind it?
```

---

# 4. Idle in Transaction

This is a common production problem.

Example:

```sql
BEGIN;

UPDATE accounts
SET status = 'ACTIVE'
WHERE id = 100;
```

Application forgets:

```sql
COMMIT;
```

The connection can become:

```text
idle in transaction
```

Check:

```sql
SELECT
    pid,
    usename,
    client_addr,
    xact_start,
    now() - xact_start AS transaction_age,
    state,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

Why this is dangerous:

```text
Locks remain held
Old transaction snapshot remains
VACUUM cleanup may be delayed
Table bloat can increase
Other sessions may block
```

A useful protection is:

```conf
idle_in_transaction_session_timeout = '5min'
```

The correct value depends on the application.

---

# 5. Safely Terminating Sessions

Cancel the currently executing query:

```sql
SELECT pg_cancel_backend(12345);
```

Terminate the entire session:

```sql
SELECT pg_terminate_backend(12345);
```

Difference:

```text
pg_cancel_backend()
        ↓
Cancel current query
Connection survives

pg_terminate_backend()
        ↓
Terminate backend
Connection closes
Transaction rolls back
```

Do not blindly kill blocking sessions.

First determine:

```text
Who owns the session?
What transaction is running?
What will rollback?
How large is the transaction?
Will the application retry?
Could terminating it cause additional load?
```

A huge rollback can itself create operational pressure.

---

# 6. Deadlocks

Blocking and deadlocks are different.

Blocking:

```text
Session A
   ↓
Session B waits
```

Deadlock:

```text
Session A holds Resource 1
Session A wants Resource 2

Session B holds Resource 2
Session B wants Resource 1
```

Graphically:

```text
Session A ───── waits for ───→ Session B
    ↑                              │
    │                              │
    └──────── waits for ───────────┘
```

PostgreSQL detects deadlocks and aborts one transaction.

Application may receive:

```text
ERROR: deadlock detected
```

Check PostgreSQL logs for:

```text
deadlock detected
DETAIL:
CONTEXT:
STATEMENT:
```

A useful setting is:

```conf
log_lock_waits = on
```

and review:

```conf
deadlock_timeout
```

Do not simply reduce `deadlock_timeout` aggressively. It controls when PostgreSQL starts deadlock checking and lock-wait logging behavior.

Common causes:

```text
Transactions update rows in different order
Large transactions
Application concurrency
DDL during workload
Foreign-key related locking
Long transactions
```

Example bad pattern:

```text
Transaction A:

UPDATE account SET ... WHERE id = 1;
UPDATE account SET ... WHERE id = 2;

Transaction B:

UPDATE account SET ... WHERE id = 2;
UPDATE account SET ... WHERE id = 1;
```

Better design:

```text
Always access/update resources in consistent order.
```

---

# 7. High CPU

When CPU is high, first determine whether PostgreSQL is actually responsible.

Linux:

```bash
top
```

or:

```bash
htop
```

Check PostgreSQL processes:

```bash
ps -ef | grep postgres
```

Inside PostgreSQL:

```sql
SELECT
    pid,
    usename,
    state,
    now() - query_start AS runtime,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;
```

Common PostgreSQL causes of high CPU:

```text
Bad execution plans
Sequential scans on large tables
Missing indexes
Large joins
Large sorts
High query concurrency
Parallel queries
JSON processing
Expression-heavy queries
Repeated application queries
Autovacuum activity
```

If `pg_stat_statements` is available:

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

Another useful view:

```sql
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

These answer two different questions:

```text
total_exec_time
→ Which queries consume most database time overall?

mean_exec_time
→ Which queries are individually expensive?
```

Then investigate with:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

Be careful running `EXPLAIN ANALYZE` against expensive write queries in production because it actually executes the statement.

---

# 8. Memory Problems

PostgreSQL memory troubleshooting requires understanding that some memory settings are **per operation or per backend**, not simply global limits.

Important settings:

```sql
SHOW shared_buffers;
SHOW work_mem;
SHOW maintenance_work_mem;
SHOW effective_cache_size;
SHOW temp_buffers;
```

Conceptually:

```text
PostgreSQL Memory
│
├── Shared memory
│      └── shared_buffers
│
├── Backend/process memory
│      ├── work_mem
│      ├── temp_buffers
│      └── query execution memory
│
└── Maintenance memory
       └── maintenance_work_mem
```

A dangerous misunderstanding is:

```conf
work_mem = 256MB
```

This does NOT mean PostgreSQL uses only 256 MB for query work.

One query can have multiple sort/hash operations, and many sessions can run concurrently.

Conceptually:

```text
Memory pressure ≈
active sessions
× memory-consuming operations
× work_mem
```

It is not an exact formula, but it explains the risk.

Check OS memory:

```bash
free -h
```

Check processes:

```bash
top
```

Look for:

```text
Low available memory
Swap activity
OOM killer
Large PostgreSQL processes
Too many connections
```

Kernel messages can help identify OOM kills:

```bash
dmesg | grep -i -E 'oom|killed process|out of memory'
```

---

# 9. Temporary File Spills

If operations cannot fit in `work_mem`, PostgreSQL may spill to disk.

Common operations:

```text
Sort
Hash
Aggregation
JOIN
DISTINCT
ORDER BY
```

Execution plan may show:

```text
Sort Method: external merge
Disk: ...
```

or hash batching.

Use:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

Database temp statistics:

```sql
SELECT
    datname,
    temp_files,
    pg_size_pretty(temp_bytes) AS temp_written
FROM pg_stat_database
ORDER BY temp_bytes DESC;
```

High temp usage can indicate:

```text
work_mem too small for workload
poor query design
large sorts
large hash operations
missing indexes
unexpected result sizes
```

Do not solve every temp spill by globally increasing `work_mem`.

---

# 10. Disk I/O Problems

Database performance can collapse even when CPU looks fine.

Symptoms:

```text
Queries slow
CPU not fully utilized
High disk latency
High I/O wait
Checkpoints slow
VACUUM slow
WAL writes slow
```

Linux tools:

```bash
iostat -xz 1
```

Useful indicators include:

```text
await
r/s
w/s
rkB/s
wkB/s
%util
```

Also:

```bash
vmstat 1
```

Check PostgreSQL I/O statistics on PostgreSQL versions that provide `pg_stat_io`:

```sql
SELECT *
FROM pg_stat_io;
```

Common causes:

```text
Large sequential scans
Cache misses
Checkpoint bursts
VACUUM
Large index builds
Large writes
Storage saturation
Temporary file spills
WAL pressure
```

---

# 11. Connection Exhaustion

Typical error:

```text
FATAL: remaining connection slots are reserved...
```

or application:

```text
too many connections
```

Check maximum:

```sql
SHOW max_connections;
```

Current connections:

```sql
SELECT count(*)
FROM pg_stat_activity;
```

Break down by state:

```sql
SELECT
    state,
    count(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;
```

By application:

```sql
SELECT
    application_name,
    state,
    count(*)
FROM pg_stat_activity
GROUP BY application_name, state
ORDER BY count(*) DESC;
```

By user:

```sql
SELECT
    usename,
    count(*)
FROM pg_stat_activity
GROUP BY usename
ORDER BY count(*) DESC;
```

Common causes:

```text
Connection leak
Bad pool configuration
Too many application instances
Long-running transactions
Blocked sessions
Traffic spike
Pooler failure
```

Do not immediately solve the problem with:

```conf
max_connections = 2000
```

More connections can mean:

```text
More memory
More processes
More context switching
More contention
Lower overall throughput
```

Often the better architecture is:

```text
Applications
      ↓
Connection Pooler
      ↓
PostgreSQL
```

For example, PgBouncer is commonly used for this role.

---

# 12. PostgreSQL Crash Troubleshooting

If PostgreSQL crashes or restarts unexpectedly, determine whether PostgreSQL crashed or the operating system terminated it.

Check:

```sql
SELECT pg_postmaster_start_time();
```

If the start time is recent, PostgreSQL restarted.

Check service status:

```bash
systemctl status postgresql
```

Check logs:

```bash
journalctl -u postgresql
```

Depending on installation, PostgreSQL logs may be in configured log directories instead.

Important messages:

```text
database system was interrupted
database system was not properly shut down
redo starts at ...
database system is ready to accept connections
PANIC
FATAL
segmentation fault
out of memory
No space left on device
```

Check OS:

```bash
dmesg
```

Common causes:

```text
OOM killer
Disk full
Filesystem problems
Hardware issues
PostgreSQL bug
Extension crash
Manual restart
Server reboot
Storage failure
```

PostgreSQL normally performs crash recovery using WAL after an unclean shutdown.

---

# 13. Replication Lag

For physical streaming replication, check the primary:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

Important stages:

```text
Primary generates WAL
        ↓
sent_lsn
        ↓
Standby receives WAL
        ↓
write_lsn
        ↓
WAL flushed
        ↓
flush_lsn
        ↓
WAL replayed
        ↓
replay_lsn
```

Useful byte-distance checks:

```sql
SELECT
    application_name,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)
    ) AS replay_byte_lag
FROM pg_stat_replication;
```

On standby:

```sql
SELECT pg_is_in_recovery();
```

Replay position:

```sql
SELECT pg_last_wal_replay_lsn();
```

Approximate time since last replayed transaction:

```sql
SELECT now() - pg_last_xact_replay_timestamp();
```

Be careful interpreting this during an idle workload. If no new transactions are being replayed, the timestamp can look old even when the standby is caught up.

---

# 14. Causes of Replication Lag

Common causes:

```text
Network latency
Network bandwidth limits
Heavy WAL generation
Slow standby storage
CPU pressure on standby
Long-running queries on standby
Recovery conflicts
WAL receiver problems
WAL replay bottleneck
Large bulk operations
Large index operations
Checkpoint/storage pressure
```

Think of replication as a pipeline:

```text
WAL generation
      ↓
WAL transmission
      ↓
WAL receive
      ↓
WAL write
      ↓
WAL flush
      ↓
WAL replay
```

Determine where the lag appears instead of treating all replication lag as the same problem.

---

# 15. WAL Troubleshooting

WAL means:

```text
Write-Ahead Log
```

Before changed database pages are safely persisted, PostgreSQL records the required change information in WAL.

WAL is fundamental for:

```text
Crash recovery
Streaming replication
Point-in-time recovery
Backups
Data durability
```

Check WAL directory size:

```bash
du -sh $PGDATA/pg_wal
```

Check WAL LSN:

```sql
SELECT pg_current_wal_lsn();
```

Check WAL statistics where supported:

```sql
SELECT *
FROM pg_stat_wal;
```

Possible WAL-related symptoms:

```text
pg_wal growing continuously
Disk filling
Replication lag
Archiving failure
Replication slot retaining WAL
Heavy write workload
Large bulk load
Checkpoint pressure
```

---

# 16. Why pg_wal Can Grow

Normal flow:

```text
Generate WAL
      ↓
WAL becomes reusable/removable when safe
```

Problem:

```text
Generate WAL
      ↓
Something still requires old WAL
      ↓
WAL cannot be recycled/removed
      ↓
pg_wal grows
      ↓
Filesystem fills
```

Main things to check:

```text
Replication slots
Archiving
Standbys
Backup/recovery requirements
High WAL generation
```

---

# 17. Replication Slots

Check:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    confirmed_flush_lsn
FROM pg_replication_slots;
```

Estimate retained WAL:

```sql
SELECT
    slot_name,
    active,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS retained_wal
FROM pg_replication_slots
WHERE restart_lsn IS NOT NULL;
```

Danger scenario:

```text
Standby removed
      ↓
Replication slot remains
      ↓
Slot inactive
      ↓
PostgreSQL retains WAL
      ↓
pg_wal grows
      ↓
Disk fills
```

Do not drop a replication slot simply because it is inactive.

First identify its owner and purpose.

---

# 18. WAL Archiving Problems

Check:

```sql
SELECT *
FROM pg_stat_archiver;
```

Important fields include:

```text
archived_count
failed_count
last_archived_wal
last_failed_wal
last_archived_time
last_failed_time
```

Check configuration:

```sql
SHOW archive_mode;
SHOW archive_command;
```

If archiving continuously fails:

```text
WAL generated
      ↓
Archive command fails
      ↓
Required WAL remains
      ↓
pg_wal can grow
      ↓
Disk pressure
```

Investigate the archive destination as well:

```text
Permissions
Network
Object storage
Filesystem
Authentication
Capacity
Command/script errors
```

---

# 19. Checkpoint Problems

Check:

```sql
SELECT *
FROM pg_stat_checkpointer;
```

On older PostgreSQL versions, related checkpoint statistics may be exposed through different statistics views.

Important concepts:

```text
Timed checkpoints
Requested checkpoints
Buffers written
Checkpoint write time
Checkpoint sync time
```

Check:

```sql
SHOW checkpoint_timeout;
SHOW max_wal_size;
SHOW checkpoint_completion_target;
```

Frequent forced checkpoints can produce bursty I/O.

Typical pattern:

```text
Heavy writes
      ↓
WAL grows rapidly
      ↓
max_wal_size pressure
      ↓
More requested checkpoints
      ↓
I/O spikes
      ↓
Application latency
```

---

# 20. Storage Full

One of the most serious PostgreSQL incidents is:

```text
No space left on device
```

Check:

```bash
df -h
```

Also check inodes:

```bash
df -i
```

Find large directories:

```bash
du -sh $PGDATA/*
```

Common sources:

```text
pg_wal
Large tables
Large indexes
Temporary files
Log files
Archived WAL
Table/index bloat
Backup files
Core dumps
```

Do not manually delete files from:

```text
$PGDATA/base
$PGDATA/global
$PGDATA/pg_wal
```

Deleting PostgreSQL-managed files directly can corrupt the database.

---

# 21. Database Size

Check database sizes:

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

Largest tables:

```sql
SELECT
    schemaname,
    relname,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;
```

Table breakdown:

```sql
SELECT
    pg_size_pretty(pg_relation_size('public.orders')) AS table_size,
    pg_size_pretty(pg_indexes_size('public.orders')) AS indexes_size,
    pg_size_pretty(pg_total_relation_size('public.orders')) AS total_size;
```

This helps determine whether growth is coming from:

```text
Heap
Indexes
TOAST
Other relation storage
```

---

# 22. Table Bloat / Dead Tuples

Check table statistics:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

High dead tuples may indicate:

```text
Heavy UPDATE/DELETE workload
Autovacuum unable to keep up
Long transactions
Incorrect autovacuum tuning
```

Check autovacuum activity:

```sql
SELECT
    pid,
    datname,
    query,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE query ILIKE '%autovacuum%';
```

Do not assume `n_dead_tup` alone gives exact physical bloat. It is an operational indicator, not a precise bloat measurement.

---

# 23. Long-Running Transactions

Find old transactions:

```sql
SELECT
    pid,
    usename,
    state,
    xact_start,
    now() - xact_start AS transaction_age,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Long transactions can cause:

```text
Locking
VACUUM cleanup delays
Dead tuple accumulation
Transaction ID pressure
Higher storage usage
Replication/recovery problems
```

This is why transaction age is often more important than query duration alone.

---

# 24. Slow Query Investigation

Start with:

```sql
SELECT
    pid,
    now() - query_start AS runtime,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;
```

Then determine:

```text
Is it running or waiting?
```

If:

```text
wait_event_type = Lock
```

Investigate blocking.

If query is actively consuming resources:

```text
Check execution plan
Check indexes
Check statistics
Check row estimates
Check I/O
Check CPU
Check temp spills
```

Use:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

Important plan problems:

```text
Sequential Scan on huge table
Bad join order
Nested Loop processing huge row counts
Disk-based Sort
Large Hash operations
Estimated rows very different from actual rows
Repeated loops
```

---

# 25. Statistics Problems

PostgreSQL optimizer depends heavily on statistics.

Check:

```sql
SELECT
    relname,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY last_analyze NULLS FIRST;
```

Manual analyze:

```sql
ANALYZE table_name;
```

Or:

```sql
VACUUM (ANALYZE) table_name;
```

Bad statistics can cause:

```text
Wrong row estimates
      ↓
Wrong execution plan
      ↓
High CPU/I/O
      ↓
Slow query
```

---

# 26. Production Incident Example — Blocking

Application team reports:

```text
API requests are timing out.
```

### Step 1 — Check sessions

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

You notice:

```text
30 sessions waiting on Lock
```

### Step 2 — Find blockers

```sql
SELECT
    pid,
    pg_blocking_pids(pid),
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

All sessions point to:

```text
PID 4501
```

### Step 3 — Inspect blocker

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
WHERE pid = 4501;
```

Result:

```text
state = idle in transaction
transaction_age = 47 minutes
```

### Step 4 — Coordinate and terminate if appropriate

```sql
SELECT pg_terminate_backend(4501);
```

### Step 5 — Verify recovery

```sql
SELECT
    pid,
    pg_blocking_pids(pid)
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

### Root cause

Application opened a transaction and failed to commit/rollback after an exception.

### Permanent fix

```text
Fix application transaction handling
Set appropriate transaction timeout
Monitor idle-in-transaction sessions
Alert on long transactions
```

---

# 27. Production Incident Example — Disk Filling

Alert:

```text
PostgreSQL filesystem 92% full
```

Check:

```bash
df -h
```

Then:

```bash
du -sh $PGDATA/*
```

Result:

```text
pg_wal = 700 GB
```

Check replication slots:

```sql
SELECT
    slot_name,
    active,
    restart_lsn
FROM pg_replication_slots;
```

One old slot is inactive.

Calculate retained WAL:

```sql
SELECT
    slot_name,
    active,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS retained_wal
FROM pg_replication_slots
WHERE restart_lsn IS NOT NULL;
```

Result:

```text
old_reporting_replica
inactive
650 GB retained
```

Root cause:

```text
Replica was decommissioned
      ↓
Replication slot was not removed
      ↓
WAL kept accumulating
      ↓
Filesystem approached capacity
```

After verifying the slot is no longer required, it can be removed:

```sql
SELECT pg_drop_replication_slot('old_reporting_replica');
```

Never drop an unknown production slot without confirming its purpose.

---

# 28. Production Incident Example — High CPU

Alert:

```text
Database CPU = 95%
```

Check OS:

```bash
top
```

PostgreSQL processes are consuming CPU.

Check active queries:

```sql
SELECT
    pid,
    now() - query_start AS runtime,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;
```

Check workload:

```sql
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

One query dominates database time.

Execution plan shows:

```text
Seq Scan
Rows = 40 million
```

Application query:

```sql
SELECT *
FROM orders
WHERE customer_id = 12345;
```

Appropriate index may be missing:

```sql
CREATE INDEX ...
```

But production troubleshooting does **not** mean immediately creating an index.

First validate:

```text
Existing indexes
Query frequency
Selectivity
Write overhead
Index size
Lock/build implications
Whether CREATE INDEX CONCURRENTLY is appropriate
```

---

# 29. Production Incident Example — Replication Lag

Alert:

```text
Replica lag increasing.
```

Primary:

```sql
SELECT
    application_name,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

Suppose:

```text
sent ≈ write ≈ flush
replay far behind
```

This suggests:

```text
Network transmission is probably not the primary bottleneck.
WAL has reached the standby.
Replay cannot keep up.
```

Investigate standby:

```text
CPU
Storage latency
Long-running read queries
Recovery conflicts
WAL replay process
System load
```

This is much better than simply saying:

> "Replication is slow."

---

# 30. Wait Events

Wait events are extremely useful because they tell you what a PostgreSQL backend is waiting for.

Check:

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state <> 'idle';
```

Examples:

```text
Lock
IO
Client
LWLock
Activity
IPC
Timeout
```

Troubleshooting mindset:

```text
Slow backend
      ↓
Is it executing or waiting?
      ↓
If waiting:
What is it waiting for?
      ↓
Lock?
I/O?
Client?
Internal synchronization?
```

Do not treat every wait event as a problem. Some waits are normal.

---

# 31. Essential PostgreSQL Troubleshooting Views

Know these well:

```text
pg_stat_activity
pg_locks
pg_stat_database
pg_stat_user_tables
pg_stat_user_indexes
pg_stat_replication
pg_replication_slots
pg_stat_archiver
pg_stat_wal
pg_stat_io
pg_stat_checkpointer
pg_stat_statements
```

Availability of some views and columns depends on the PostgreSQL version.

---

# 32. Essential OS Commands

For PostgreSQL production support, SQL alone is not enough.

### CPU/processes

```bash
top
```

```bash
ps -ef | grep postgres
```

### Memory

```bash
free -h
```

```bash
vmstat 1
```

### Disk

```bash
df -h
```

```bash
df -i
```

### I/O

```bash
iostat -xz 1
```

### Directory usage

```bash
du -sh /path/*
```

### Service

```bash
systemctl status postgresql
```

### Logs

```bash
journalctl -u postgresql
```

Exact service names and log locations vary by Linux distribution and PostgreSQL installation.

---

# 33. What NOT to Do During an Incident

Avoid panic actions such as:

```text
Restart PostgreSQL immediately
Kill every long-running session
Increase max_connections dramatically
Increase work_mem globally
Delete files from pg_wal
Delete PostgreSQL data files
Drop replication slots without checking
Run VACUUM FULL blindly
Create indexes without checking impact
Change many parameters simultaneously
```

A better sequence is:

```text
Observe
   ↓
Collect evidence
   ↓
Identify bottleneck
   ↓
Understand impact
   ↓
Mitigate
   ↓
Verify
   ↓
Find root cause
   ↓
Implement permanent fix
```

---

# 34. Troubleshooting Cheat Sheet

| Problem              | First Checks                                     |
| -------------------- | ------------------------------------------------ |
| Database slow        | `pg_stat_activity`, waits, CPU, I/O              |
| Queries waiting      | `pg_blocking_pids()`, `pg_locks`                 |
| Deadlocks            | PostgreSQL logs                                  |
| High CPU             | active queries, `pg_stat_statements`, plans      |
| High memory          | connections, `work_mem`, OS memory               |
| High I/O             | `iostat`, plans, temp files, checkpoints         |
| Too many connections | `pg_stat_activity`, pool configuration           |
| PostgreSQL restarted | `pg_postmaster_start_time()`, logs, OS           |
| Replica lag          | `pg_stat_replication`, primary/standby resources |
| WAL growth           | slots, archiver, WAL generation                  |
| Disk filling         | `df`, `du`, database/table sizes                 |
| Dead tuples          | `pg_stat_user_tables`, autovacuum                |
| Slow query           | waits → plan → statistics → indexes              |
| Disk sorts           | plan + temp statistics + `work_mem`              |
| WAL archive failure  | `pg_stat_archiver`, archive destination          |

---

# 35. Interview Scenario Questions

### Q1. PostgreSQL CPU suddenly reaches 100%. What do you check?

A strong answer:

```text
1. Confirm PostgreSQL is consuming CPU at OS level.
2. Check active sessions.
3. Separate running queries from waiting queries.
4. Check pg_stat_statements.
5. Identify queries consuming the most total execution time.
6. Examine execution plans.
7. Check for sequential scans, bad joins and bad estimates.
8. Check workload/concurrency changes.
9. Apply safe mitigation.
10. Fix the query/index/statistics/root cause.
```

---

### Q2. Application says database is hanging.

I would first check:

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    pg_blocking_pids(pid),
    query
FROM pg_stat_activity
WHERE state <> 'idle';
```

If many sessions are waiting on locks, identify the root blocker.

---

### Q3. pg_wal is consuming all disk space. What do you do?

Check:

```text
Replication slots
Archive failures
Replica health
WAL generation rate
Backup/recovery activity
```

Do **not**:

```bash
rm -rf $PGDATA/pg_wal/*
```

That can destroy the database.

---

### Q4. Replication is lagging. How do you troubleshoot?

Check:

```text
sent_lsn
write_lsn
flush_lsn
replay_lsn
```

Then identify whether the bottleneck is:

```text
Primary WAL generation
Network transmission
Standby WAL write
Standby WAL flush
Standby replay
```

---

### Q5. Database reached max_connections. Should we increase it?

Not immediately.

First determine:

```text
Which application owns connections?
Are connections idle?
Is there a leak?
Are transactions blocked?
Is pooling configured?
Did application instances increase?
```

Increasing `max_connections` may hide the root cause and increase memory/resource pressure.

---

### Q6. What is the difference between blocking and deadlock?

```text
Blocking:

A holds resource
B waits for A

Deadlock:

A waits for B
B waits for A
```

PostgreSQL automatically detects deadlocks and aborts one transaction.

---

# 36. Architecture-Level Troubleshooting View

Think about PostgreSQL as a complete request path:

```text
                    ┌─────────────────┐
                    │   Application   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Connection Pool │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   PostgreSQL    │
                    │                 │
                    │ Queries         │
                    │ Locks           │
                    │ Transactions    │
                    │ Buffers         │
                    │ WAL             │
                    │ Checkpoints     │
                    │ Autovacuum      │
                    └───────┬─────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
       ┌─────────────┐             ┌─────────────┐
       │ OS Resources│             │   Storage   │
       │ CPU / Memory│             │ Data / WAL  │
       └─────────────┘             └─────────────┘
                            │
                            ▼
                    ┌─────────────────┐
                    │    Standby      │
                    │   Replication   │
                    └─────────────────┘
```

A PostgreSQL problem can originate at any layer.

That is why troubleshooting should correlate:

```text
Application
+
Database
+
Operating System
+
Storage
+
Network
+
Replication
```

---

# 37. 5-Minute Production Troubleshooting Runbook

When paged for:

> "PostgreSQL production is slow."

Use this sequence.

### Minute 1 — Availability

```bash
systemctl status postgresql
```

```sql
SELECT now(), pg_postmaster_start_time();
```

### Minute 2 — Sessions

```sql
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

### Minute 3 — Blocking

```sql
SELECT
    pid,
    pg_blocking_pids(pid),
    now() - xact_start AS transaction_age,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

### Minute 4 — OS

```bash
top
free -h
df -h
iostat -xz 1
```

Check:

```text
CPU
Memory
Swap
Filesystem
I/O latency
```

### Minute 5 — Database subsystems

Check:

```text
Connections
Long transactions
Replication
WAL
Archiving
Checkpoints
Autovacuum
Temp files
```

Then move from:

```text
Symptom
   ↓
Evidence
   ↓
Bottleneck
   ↓
Root cause
   ↓
Safe mitigation
   ↓
Permanent fix
```

---

# 38. Golden Troubleshooting Rule

The most useful question during a PostgreSQL incident is:

> **What is PostgreSQL waiting on?**

Then work outward:

```text
Lock wait
    → Find blocker

CPU bound
    → Find expensive workload

I/O wait
    → Find read/write source

Memory pressure
    → Check sessions + query memory

Connection exhaustion
    → Find connection owner/leak

Replication lag
    → Find pipeline bottleneck

WAL growth
    → Check slots/archive/WAL rate

Disk full
    → Find what is consuming storage

Crash
    → PostgreSQL logs + OS logs
```

The goal is not just to restore service.

A strong PostgreSQL engineer should be able to explain:

```text
What happened?
Why did it happen?
What was the immediate mitigation?
How did we verify recovery?
What prevents it from happening again?
```

That is the production troubleshooting mindset expected from a PostgreSQL DBA, Senior Database Engineer, or Database/Cloud Architect.
