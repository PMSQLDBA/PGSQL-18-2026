# Topic 11: PostgreSQL Storage & Capacity Planning

## 1. Why Storage Planning Matters

PostgreSQL performance depends heavily on the storage layer. 

CPU and memory can be healthy, but slow disks, insufficient IOPS, WAL contention, or unexpected database growth can still cause serious performance problems.

As a PostgreSQL DBA or Architect, storage planning should answer these questions:

* How much storage does the database need today?
* How quickly is it growing?
* How much storage will it need in 6, 12, or 24 months?
* What IOPS and throughput does the workload require?
* Should WAL be separated from database storage?
* Where should temporary files and backups live?
* When should tablespaces be used?
* How much free space should be maintained?
* What happens if the filesystem becomes full?

---

# 2. PostgreSQL Storage Architecture

At a high level:

```text
Application
    |
    v
PostgreSQL
    |
    +---------------- PostgreSQL Data Directory (PGDATA)
    |                       |
    |                       +-- base/
    |                       +-- global/
    |                       +-- pg_wal/
    |                       +-- pg_tblspc/
    |                       +-- pg_stat/
    |                       +-- pg_multixact/
    |
    +---------------- Tablespaces
    |
    +---------------- Temporary Files
    |
    +---------------- WAL
```

The main PostgreSQL data directory is normally identified by:

```sql
SHOW data_directory;
```

Important directories include:

| Directory       | Purpose                             |
| --------------- | ----------------------------------- |
| `base/`         | Database table and index files      |
| `global/`       | Cluster-wide system catalog data    |
| `pg_wal/`       | Write-Ahead Log files               |
| `pg_tblspc/`    | Links to user-defined tablespaces   |
| `pg_stat/`      | Statistics-related files            |
| `pg_multixact/` | Multitransaction status information |
| `pg_xact/`      | Transaction commit status           |

---

# 3. Recommended Disk Layout

There is no single disk layout that works for every PostgreSQL environment.

For a medium-to-large production system, a logical layout could look like:

```text
Disk / Volume 1
/postgres/data
    |
    +-- PGDATA

Disk / Volume 2
/postgres/wal
    |
    +-- WAL

Disk / Volume 3
/postgres/tablespace01
    |
    +-- Large application objects

Disk / Volume 4
/postgres/temp
    |
    +-- Temporary workload

Backup Storage
/backup/postgres
```

Conceptually:

```text
                 PostgreSQL
                     |
        +------------+-------------+
        |            |             |
        v            v             v
      DATA          WAL        TABLESPACE
        |            |             |
      Disk 1       Disk 2        Disk 3

                     |
                     v
                  BACKUP
                     |
                  Disk/Object
                  Storage
```

The actual design depends heavily on whether PostgreSQL runs on physical servers, VMs, SAN storage, cloud block storage, Kubernetes, or managed PostgreSQL.

---

# 4. Data Storage

PostgreSQL stores tables and indexes in relation files.

Check database sizes:

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS database_size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

Example:

```text
Database       Size
-------------  -------
orders_prod    1.2 TB
reporting      480 GB
postgres       15 MB
```

Check the current database:

```sql
SELECT pg_size_pretty(pg_database_size(current_database()));
```

---

# 5. Table and Index Storage

One important capacity-planning activity is determining which objects consume the most space.

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

`pg_total_relation_size()` includes:

```text
Table
 + Indexes
 + TOAST
```

To examine them separately:

```sql
SELECT
    pg_size_pretty(pg_relation_size('schema.table_name')) AS table_size,
    pg_size_pretty(pg_indexes_size('schema.table_name')) AS indexes_size,
    pg_size_pretty(pg_total_relation_size('schema.table_name')) AS total_size;
```

This is useful when a table appears unusually large.

---

# 6. Filesystem Selection

PostgreSQL relies heavily on the operating system filesystem.

Common Linux filesystems include:

* XFS
* ext4

Both are widely used for PostgreSQL.

Important considerations are usually more important than simply choosing between XFS and ext4:

```text
Filesystem
    |
    +-- Latency
    +-- IOPS
    +-- Throughput
    +-- Reliability
    +-- Mount configuration
    +-- Storage controller
    +-- Cache behavior
    +-- Underlying disk technology
```

For production systems, avoid changing filesystem or mount settings simply because a generic PostgreSQL recommendation says to do so. The configuration should be validated against the OS, storage platform, backup design, and vendor recommendations.

---

# 7. IOPS

IOPS means:

**Input/Output Operations Per Second**

It measures how many storage operations the storage layer can perform each second.

PostgreSQL can generate significant random I/O from:

```text
Index lookups
Random table reads
Checkpoint writes
Background writer
VACUUM
Autovacuum
WAL
Temporary files
Large queries
ETL jobs
Backups
```

Example:

Suppose PostgreSQL requires:

```text
Read IOPS  = 8,000
Write IOPS = 4,000
```

The storage system needs to sustain approximately:

```text
12,000 IOPS
```

But IOPS alone is not enough.

You also need to understand:

```text
IOPS
Latency
Throughput
Queue depth
Read/write ratio
I/O block size
```

---

# 8. Latency vs IOPS vs Throughput

These three concepts are important.

### IOPS

Number of I/O operations completed per second.

```text
Example:
15,000 IOPS
```

### Latency

Time required to complete an I/O request.

```text
Example:
1 ms
5 ms
20 ms
```

Lower latency is generally better.

### Throughput

Amount of data transferred per second.

```text
Example:
500 MB/sec
1 GB/sec
```

Different workloads stress storage differently.

```text
OLTP
 |
 +-- Random I/O
 +-- Low latency
 +-- High IOPS

Analytics
 |
 +-- Sequential scans
 +-- Large reads
 +-- High throughput
```

---

# 9. PostgreSQL and Storage Latency

For transaction-heavy systems, WAL write latency can be especially important.

Typical transaction flow:

```text
BEGIN
   |
Application changes data
   |
WAL generated
   |
COMMIT
   |
WAL flushed to durable storage
   |
Transaction confirmed
```

If WAL storage is slow:

```text
COMMIT
  |
  v
Wait for WAL flush
  |
  v
Application latency increases
```

This is why storage latency can directly affect transaction response time.

---

# 10. WAL Storage

WAL stands for:

**Write-Ahead Log**

By default it is stored under:

```text
$PGDATA/pg_wal
```

Check the WAL directory from the OS:

```bash
du -sh $PGDATA/pg_wal
```

WAL activity can be significant in systems with:

* Heavy INSERT operations
* Heavy UPDATE operations
* Heavy DELETE operations
* Bulk loading
* Index creation
* VACUUM-related activity
* Replication
* Backup operations

---

# 11. Separate WAL Disk

For some high-I/O environments, WAL can be placed on separate storage.

Conceptually:

```text
           PostgreSQL
               |
       +-------+-------+
       |               |
       v               v
     DATA             WAL
       |               |
   Data Volume      WAL Volume
```

Potential advantages:

* Data I/O and WAL I/O can be isolated.
* WAL latency can be managed separately.
* Checkpoint/data writes are less likely to compete with WAL writes.
* Storage performance can be tuned independently.

However, separating WAL does **not automatically improve performance**.

If both volumes are backed by the same physical SAN, storage pool, or cloud infrastructure, the separation may provide little physical I/O isolation.

Always understand the underlying storage architecture.

---

# 12. WAL Capacity Planning

WAL storage should never be sized only based on `max_wal_size`.

Other factors can retain WAL.

Examples include:

```text
Replication slots
Archive failures
Slow standby servers
Long-running backup operations
High write workload
Recovery requirements
```

Check WAL settings:

```sql
SHOW max_wal_size;
SHOW min_wal_size;
SHOW wal_keep_size;
SHOW archive_mode;
```

Check replication slots:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn
FROM pg_replication_slots;
```

Unused or failed replication slots can cause WAL files to accumulate and potentially fill the filesystem.

---

# 13. Tablespaces

A PostgreSQL tablespace allows database objects to be placed in another filesystem location.

Create one:

```sql
CREATE TABLESPACE fast_storage
LOCATION '/postgres/tablespace/fast';
```

Create a table using it:

```sql
CREATE TABLE orders (
    order_id BIGINT,
    order_date TIMESTAMP
)
TABLESPACE fast_storage;
```

Move an existing table:

```sql
ALTER TABLE orders
SET TABLESPACE fast_storage;
```

---

# 14. When Tablespaces Are Useful

Tablespaces can be useful when you have different storage tiers.

Example:

```text
                    PostgreSQL
                        |
       +----------------+----------------+
       |                |                |
       v                v                v
   Fast SSD          Standard SSD     Large Storage
       |                |                |
Hot tables         Normal tables      Archive data
Indexes
```

Possible use cases:

* Very large tables
* High-performance indexes
* Archive tables
* Storage migration
* Different storage tiers
* Separating large workloads

Avoid creating tablespaces without a clear operational reason because they add storage-management complexity.

---

# 15. Temporary File Storage

Queries can create temporary files when operations cannot fit in memory.

Examples:

```text
SORT
HASH JOIN
HASH AGGREGATION
CREATE INDEX
Large reporting query
```

Example flow:

```text
Query
 |
 v
Sort operation
 |
 +-- Fits in work_mem
 |       |
 |       v
 |     Memory
 |
 +-- Doesn't fit
         |
         v
     Temporary file
         |
         v
        Disk
```

Frequent large temporary files can create substantial storage I/O.

Useful configuration:

```sql
SHOW work_mem;
SHOW temp_file_limit;
```

Temporary file logging can be enabled with:

```text
log_temp_files
```

For example:

```sql
SHOW log_temp_files;
```

This helps identify queries spilling heavily to disk.

---

# 16. Database Growth Monitoring

Capacity planning requires historical data.

Checking the database size today is not enough.

For example:

```text
January     1.0 TB
February    1.1 TB
March       1.2 TB
April       1.3 TB
May         1.4 TB
```

Growth rate:

```text
100 GB/month
```

If the current database is:

```text
1.4 TB
```

Expected size after 12 months:

```text
1.4 TB + (100 GB × 12)

= approximately 2.6 TB
```

But production sizing should also account for indexes, WAL, temporary space, maintenance operations, filesystem reserve, and growth uncertainty.

---

# 17. Capacity Forecasting Formula

A simple starting formula is:

```text
Future Storage =
Current Storage
+ Expected Growth
+ Operational Headroom
```

A more realistic model is:

```text
Required Storage =
Data
+ Indexes
+ TOAST
+ WAL Reserve
+ Temporary Space
+ Maintenance Space
+ Growth Forecast
+ Safety Margin
```

Example:

```text
Current PostgreSQL size       = 2.0 TB
Expected 12-month growth      = 1.0 TB
WAL operational reserve       = 300 GB
Temp/maintenance requirement  = 500 GB
Safety headroom               = 700 GB
---------------------------------------
Required planned capacity     = 4.5 TB
```

The numbers should come from actual workload measurements rather than fixed percentages whenever possible.

---

# 18. Why Free Space Is Important

Do not design PostgreSQL storage to operate continuously near 100%.

Free space may be required for:

```text
Table growth
Index growth
WAL spikes
Temporary files
CREATE INDEX
REINDEX
VACUUM FULL
ALTER TABLE operations
Bulk loads
Unexpected workload increases
```

For example, `REINDEX` can temporarily require significant additional disk space because PostgreSQL needs space to build the replacement index.

---

# 19. Capacity Thresholds

A practical monitoring strategy could use thresholds such as:

```text
< 70%     Healthy
70-80%    Monitor growth
80-90%    Capacity action required
> 90%     High risk
```

These are operational examples, not universal PostgreSQL rules.

A rapidly growing system may require action even at 60%, while a stable multi-terabyte system may safely operate differently.

The important metric is:

```text
Days until filesystem full
```

rather than filesystem percentage alone.

---

# 20. Forecasting "Days Until Full"

Suppose:

```text
Filesystem size = 5 TB
Used            = 4 TB
Free            = 1 TB

Average growth = 20 GB/day
```

Then:

```text
Days until full =
Free Space / Daily Growth

= 1000 GB / 20 GB

= approximately 50 days
```

This is much more useful operationally than simply saying:

```text
Disk usage = 80%
```

Because the DBA now knows approximately when capacity will become critical.

---

# 21. Linear Capacity Forecasting

Suppose monitoring collects:

```text
Date        Database Size
----------  -------------
Jan 01      2.00 TB
Feb 01      2.15 TB
Mar 01      2.31 TB
Apr 01      2.48 TB
May 01      2.64 TB
```

Approximate growth:

```text
~160 GB/month
```

12-month forecast:

```text
2.64 TB + (0.16 × 12)

≈ 4.56 TB
```

Then add operational headroom based on the workload.

Do not assume growth will always remain linear. Business launches, retention changes, migrations, audit requirements, and new applications can dramatically change growth.

---

# 22. Storage Growth Query

PostgreSQL does not automatically maintain a long-term database-size history for capacity forecasting.

A DBA can periodically collect:

```sql
SELECT
    now() AS capture_time,
    datname,
    pg_database_size(datname) AS size_bytes
FROM pg_database;
```

Store these results in a monitoring table.

Example:

```sql
CREATE TABLE database_size_history (
    capture_time timestamptz NOT NULL,
    database_name text NOT NULL,
    size_bytes bigint NOT NULL
);
```

Then periodically insert:

```sql
INSERT INTO database_size_history
SELECT
    now(),
    datname,
    pg_database_size(datname)
FROM pg_database;
```

This provides historical data for forecasting.

---

# 23. Table-Level Growth Monitoring

Database-level growth may not tell you what is actually growing.

Track large tables individually.

For example:

```text
orders             +80 GB/month
order_history      +45 GB/month
audit_log          +30 GB/month
customer           +2 GB/month
```

Now the DBA knows where the growth is coming from.

This can drive decisions around:

```text
Partitioning
Retention
Archiving
Index cleanup
Compression strategy
Application design
Storage expansion
```

---

# 24. Bloat and Capacity

Database size growth does not always mean business data growth.

PostgreSQL uses MVCC, so UPDATE and DELETE operations can create dead tuples.

Conceptually:

```text
UPDATE row
    |
    +-- Old row version
    |
    +-- New row version
```

Until cleanup/reuse occurs, storage can contain dead tuples.

Therefore:

```text
Storage Growth
      |
      +-- Real data growth
      +-- Index growth
      +-- Dead tuples
      +-- Table bloat
      +-- Index bloat
```

Capacity analysis should distinguish between actual business growth and reclaimable/reusable space.

---

# 25. Autovacuum and Storage

Autovacuum helps PostgreSQL manage dead tuples.

Useful statistics:

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

A table with continuously increasing dead tuples may need investigation.

Possible reasons include:

```text
High UPDATE activity
High DELETE activity
Long-running transactions
Insufficient autovacuum activity
Incorrect autovacuum tuning
Workload spikes
```

---

# 26. Storage Requirements During Maintenance

Maintenance operations can require additional storage.

Examples:

### CREATE INDEX

Needs storage for the new index.

### REINDEX

Needs enough room to build replacement index structures.

### VACUUM FULL

Rewrites the table and therefore needs significant additional temporary storage.

### ALTER TABLE

Some operations may rewrite the entire table.

### pg_dump

If backups are written locally, backup files consume additional filesystem space.

Capacity planning must account for these temporary peaks.

---

# 27. Backup Storage Capacity

Backup storage should be planned separately from database storage.

Example:

```text
Database             = 3 TB

Full backup          = 2 TB compressed
Daily incremental    = 150 GB
Retention            = 30 days
```

Backup capacity depends on:

```text
Backup type
Compression
Retention period
Change rate
WAL archive volume
Number of retained backups
Restore requirements
```

Do not size backup storage simply as:

```text
Database Size × Number of Backups
```

because compression and incremental/differential strategies can significantly change the requirement.

---

# 28. WAL Archive Growth

If WAL archiving is enabled:

```text
PostgreSQL
    |
    v
 pg_wal
    |
    v
archive_command
    |
    v
WAL Archive Storage
```

Archive storage growth depends primarily on WAL generation rate and retention.

Example:

```text
WAL generation = 300 GB/day
Retention       = 14 days

Raw WAL requirement:

300 GB × 14
= 4.2 TB
```

Additional headroom should be included.

High WAL generation can make archive storage much larger than expected.

---

# 29. Measure WAL Generation Rate

Modern PostgreSQL statistics can help monitor WAL activity.

For example:

```sql
SELECT *
FROM pg_stat_wal;
```

Important values can include:

```text
wal_records
wal_fpi
wal_bytes
```

For capacity planning, collect `wal_bytes` periodically and calculate:

```text
WAL Generated =
Current wal_bytes - Previous wal_bytes
```

For example:

```text
08:00  10 TB
09:00  10.05 TB
```

Approximately:

```text
50 GB WAL/hour
```

This information helps size:

```text
WAL filesystem
Archive storage
Replication bandwidth
Backup infrastructure
Network capacity
```

---

# 30. Storage Monitoring at OS Level

Useful Linux commands include:

```bash
df -h
```

Filesystem capacity.

```bash
du -sh /postgres/data
```

Directory usage.

```bash
iostat -x 1
```

Storage performance.

Useful `iostat` metrics include:

```text
r/s
w/s
rkB/s
wkB/s
await
aqu-sz
%util
```

The exact interpretation depends on the storage architecture.

For cloud and SAN environments, OS metrics should also be compared with metrics from the storage platform.

---

# 31. Cloud Storage Planning

Cloud storage introduces additional considerations.

Storage performance may depend on:

```text
Volume size
Provisioned IOPS
Provisioned throughput
Instance limits
Storage type
Burst credits
Network limits
```

Architecture example:

```text
PostgreSQL VM
     |
     +---- Data Volume
     |
     +---- WAL Volume
     |
     +---- Backup/Object Storage
```

A common mistake is provisioning a high-performance disk while the VM itself cannot deliver the requested IOPS or throughput.

Always check the complete path:

```text
PostgreSQL
    |
Operating System
    |
VM / Instance
    |
Storage Interface
    |
Storage Volume
    |
Underlying Cloud Infrastructure
```

The slowest layer becomes the practical limit.

---

# 32. Storage Performance Problem Example

Suppose users report:

```text
COMMIT operations are slow.
```

The DBA sees:

```text
CPU            30%
Memory          Healthy
Connections     Normal
Locks           Normal
```

But WAL-related waits and storage monitoring show high write latency.

Architecture:

```text
Transaction
    |
    v
WAL write
    |
    v
Storage latency = 20 ms
    |
    v
COMMIT waits
    |
    v
Application becomes slow
```

The problem is storage latency rather than CPU.

---

# 33. Capacity Problem Example

Suppose:

```text
Filesystem       = 10 TB
Used             = 8 TB
Free             = 2 TB
Growth           = 250 GB/month
```

Simple forecast:

```text
2000 GB / 250 GB

= approximately 8 months
```

But a planned data migration will add:

```text
1.2 TB
```

After migration:

```text
Free space ≈ 800 GB
```

Remaining capacity:

```text
800 / 250

≈ 3.2 months
```

Therefore storage expansion should happen before the migration rather than waiting for the filesystem to reach a static alert threshold.

---

# 34. Production Capacity Planning Model

For architecture reviews, divide storage into categories:

| Component     | Current |          Growth |   Forecast | Headroom |
| ------------- | ------: | --------------: | ---------: | -------: |
| Database data |    3 TB |    200 GB/month |     5.4 TB | Required |
| WAL           |  100 GB |        Variable | Peak based | Required |
| Temp          |  200 GB |  Workload based | Peak based | Required |
| Tablespaces   |    1 TB |     80 GB/month |    1.96 TB | Required |
| Backup        |    8 TB | Retention based | Calculated | Required |
| WAL archive   |    4 TB |  WAL rate based | Calculated | Required |

This gives infrastructure teams a much clearer picture than requesting:

```text
"We need another 5 TB."
```

---

# 35. Architecture-Level Storage Questions

During a PostgreSQL architecture review, ask:

### Capacity

```text
What is the current database size?
What is monthly growth?
What is the largest table?
What is the fastest-growing table?
What is expected growth for 12/24/36 months?
```

### Performance

```text
What are current IOPS?
What is peak IOPS?
What is read/write latency?
What is peak throughput?
What is the storage queue depth?
```

### WAL

```text
What is WAL generation per hour/day?
Are replication slots retaining WAL?
Is WAL archiving working?
How much WAL can accumulate during an outage?
```

### Maintenance

```text
Is there enough room for REINDEX?
Is there enough room for VACUUM FULL if required?
Can large indexes be rebuilt?
Can a bulk load temporarily double object storage?
```

### Resilience

```text
What happens if storage reaches 90%?
When are alerts triggered?
How quickly can storage be expanded?
Does expansion require downtime?
```

---

# 36. Important Production Alerts

Storage monitoring should normally cover:

```text
Filesystem usage
Filesystem free bytes
Database growth rate
Tablespace usage
WAL directory growth
WAL archive failures
Replication slot WAL retention
Temporary file generation
Storage latency
Storage IOPS
Storage throughput
Backup storage usage
Days until capacity exhaustion
```

A good monitoring system combines PostgreSQL metrics with OS/storage metrics.

---

# 37. Common Storage Mistakes

### Mistake 1: Planning only for current database size

```text
DB = 2 TB

Provision storage = 2.2 TB
```

This leaves little room for growth and maintenance.

---

### Mistake 2: Ignoring indexes

A table may contain:

```text
Data     = 500 GB
Indexes  = 700 GB
```

Looking only at table heap size underestimates actual storage.

---

### Mistake 3: Ignoring WAL spikes

Normal WAL:

```text
20 GB/hour
```

Bulk operation:

```text
150 GB/hour
```

Capacity must consider peaks.

---

### Mistake 4: Ignoring replication slots

Inactive slots can retain WAL and consume the WAL filesystem.

---

### Mistake 5: Treating disk percentage as the only metric

```text
Disk = 70%
```

does not tell you enough.

You need:

```text
Growth rate
+
Free capacity
+
Expected business events
=
Time until capacity exhaustion
```

---

### Mistake 6: Assuming separate logical disks mean separate physical storage

```text
/data -> SAN Pool A
/wal  -> SAN Pool A
```

They appear separate to PostgreSQL but may compete for the same physical resources.

---

### Mistake 7: Ignoring maintenance headroom

A filesystem may have enough capacity for normal operation but not enough to rebuild a multi-terabyte table or index.

---

# 38. DBA Troubleshooting Flow

When PostgreSQL shows storage-related performance issues:

```text
Application Slow
      |
      v
Check PostgreSQL waits
      |
      v
Is workload I/O bound?
      |
   +--+--+
   |     |
  No    Yes
         |
         v
   Check latency
         |
         v
   Check IOPS/throughput
         |
         v
   Check WAL vs data I/O
         |
         v
   Check temp-file activity
         |
         v
   Check checkpoints
         |
         v
   Check storage platform
         |
         v
   Identify bottleneck
```

---

# 39. Capacity Planning Flow

A practical architecture process is:

```text
Collect Current Size
        |
        v
Collect Historical Growth
        |
        v
Identify Fast-Growing Objects
        |
        v
Measure WAL Growth
        |
        v
Measure Temp/Maintenance Peaks
        |
        v
Include Business Forecast
        |
        v
Calculate 12/24 Month Requirement
        |
        v
Add Operational Headroom
        |
        v
Validate IOPS + Throughput
        |
        v
Provision Storage
        |
        v
Monitor Actual vs Forecast
```

Capacity planning should be a continuous process rather than a one-time infrastructure exercise.

---

# 40. Interview Questions

### Q1. What storage metrics are important for PostgreSQL?

Key metrics include:

```text
Capacity
IOPS
Latency
Throughput
Queue depth
Database growth
WAL growth
Temporary-file activity
```

---

### Q2. Why is WAL storage performance important?

Transactions normally need WAL records to reach durable storage before PostgreSQL can confirm synchronous commits. High WAL write latency can therefore increase transaction latency.

---

### Q3. Should WAL always be on a separate disk?

No.

Separate WAL storage can help when it provides real I/O isolation or different performance characteristics. If both volumes use the same underlying storage resources, the benefit may be limited.

---

### Q4. What are tablespaces?

Tablespaces allow PostgreSQL database objects to be stored in filesystem locations outside the default data location.

---

### Q5. How do you forecast PostgreSQL storage?

Collect historical database and table sizes, calculate growth rates, incorporate expected business changes, then include WAL, temp space, maintenance requirements, backups where applicable, and operational headroom.

---

### Q6. Why can PostgreSQL storage grow after many UPDATEs?

PostgreSQL MVCC creates new row versions. Old versions become dead tuples and remain until VACUUM makes their space reusable. Depending on the workload, the physical relation may remain larger even after vacuuming.

---

### Q7. Why can a replication slot fill the disk?

A replication slot can prevent PostgreSQL from recycling WAL that the slot's consumer still needs. If the consumer stops or falls behind, retained WAL can continue growing.

---

### Q8. What happens when the PostgreSQL filesystem becomes full?

The impact depends on which filesystem is full, but a full PostgreSQL data or WAL filesystem can prevent writes and can cause database errors or service interruption. It should be treated as a critical operational condition.

---

# 41. Real-Time DBA Scenario

**Scenario:**

Production alerts show:

```text
/data = 68%
/wal  = 92%
```

Database growth appears normal.

First question:

```text
Why is WAL growing?
```

Investigation path:

```text
Check replication slots
        |
        +-- Inactive slot?
        |
        +-- Consumer lag?
        |
Check WAL archiving
        |
        +-- archive_command failing?
        |
Check replication
        |
        +-- standby unavailable?
        |
Check workload
        |
        +-- bulk UPDATE?
        +-- bulk INSERT?
        +-- index operation?
```

Suppose:

```sql
SELECT
    slot_name,
    active,
    restart_lsn
FROM pg_replication_slots;
```

shows an inactive logical replication slot retaining a large amount of WAL.

Then the root cause is not normal database growth.

It is:

```text
Inactive replication consumer
        |
        v
Replication slot retains WAL
        |
        v
WAL cannot be recycled
        |
        v
pg_wal grows
        |
        v
Filesystem approaches 100%
```

The DBA should verify whether the slot is still required before taking corrective action. Never drop a production replication slot simply because it is inactive.

---

# 42. Key Takeaways

Remember these points:

1. PostgreSQL storage planning is about **capacity and performance**, not disk size alone.

2. Monitor:

```text
IOPS + Latency + Throughput
```

3. WAL storage can directly affect transaction performance.

4. Separate WAL storage is useful only when the underlying architecture provides meaningful isolation or different performance characteristics.

5. Tablespaces provide control over object placement but add operational complexity.

6. Track historical growth rather than relying only on current database size.

7. Capacity forecasting should include:

```text
Data
+ Indexes
+ WAL
+ Temp
+ Maintenance
+ Expected Growth
+ Operational Headroom
```

8. Monitor replication slots because they can retain large amounts of WAL.

9. Capacity monitoring should answer:

```text
"When will we run out of space?"
```

not only:

```text
"How full is the disk?"
```

10. Storage planning should combine PostgreSQL metrics, OS metrics, infrastructure metrics, and expected business growth.

---

## Architecture Summary

```text
                       PostgreSQL
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
        DATA              WAL          TABLESPACES
          |                |                |
          +----------------+----------------+
                           |
                           v
                    Storage Platform
                           |
              +------------+------------+
              |            |            |
             IOPS        Latency     Throughput
              |            |            |
              +------------+------------+
                           |
                           v
                    Capacity Planning
                           |
             +-------------+-------------+
             |             |             |
           Growth      Headroom      Forecasting
             |             |             |
             +-------------+-------------+
                           |
                           v
                 Reliable PostgreSQL
                    Infrastructure
```
