# Topic 15 — PostgreSQL WAL & Checkpoints

This topic is important for both **PostgreSQL architecture and production performance troubleshooting**. 

WAL and checkpoints directly affect durability, crash recovery, replication, backup/recovery, and write-heavy workload performance.

## 1. What is WAL?

**WAL = Write-Ahead Logging.**

PostgreSQL follows one fundamental rule:

> **The WAL record describing a database change must be safely written to persistent storage before the modified database page itself is written.**

Suppose a transaction changes a row:

```sql
UPDATE accounts
SET balance = balance - 100
WHERE account_id = 101;
```

PostgreSQL does **not** need to immediately write the modified table page to the data file.

Conceptually:

```text
Transaction
    |
    v
Modify page in Shared Buffers
    |
    +------> Generate WAL record
                  |
                  v
             WAL Buffers
                  |
                  v
              WAL Files
             (pg_wal/)
                  |
                  v
                COMMIT

Later:
Dirty Shared Buffer
        |
        v
Checkpoint / Background Writer
        |
        v
Database Data Files
```

This separation is critical for PostgreSQL performance.

Random database-page writes can happen later, while WAL can largely be written sequentially.

---

# 2. Why PostgreSQL Needs WAL

WAL provides several major capabilities:

* **Durability**
* **Crash recovery**
* **Point-in-time recovery (PITR)**
* **Physical replication**
* **Online backups**
* **Data consistency after crashes**

Consider:

```text
BEGIN;

UPDATE orders
SET status = 'PAID'
WHERE order_id = 5001;

COMMIT;
```

The modified table page may still exist only in shared memory when COMMIT completes.

But PostgreSQL makes sure the required WAL has been persisted.

If the server crashes immediately afterward:

```text
Server crashes
     |
     v
PostgreSQL restarts
     |
     v
Reads WAL
     |
     v
Replays changes
     |
     v
Database restored to consistent state
```

This is why a committed transaction doesn't require PostgreSQL to immediately flush every changed table/index page.

---

# 3. WAL Architecture

A simplified PostgreSQL write path is:

```text
Client
  |
  v
Backend Process
  |
  +------------------------+
  |                        |
  v                        v
Shared Buffers        WAL Buffers
  |                        |
  |                        v
Dirty Pages              WAL
  |                     pg_wal/
  |                        |
  v                        +------> WAL Archive
Data Files                |
                           +------> Standby / Replica
```

There are therefore two important write paths.

### Data path

```text
SQL
 ↓
Shared Buffers
 ↓
Dirty Pages
 ↓
Data Files
```

### WAL path

```text
SQL
 ↓
WAL Records
 ↓
WAL Buffers
 ↓
WAL Segments
 ↓
Disk
```

The WAL path protects the data path.

---

# 4. WAL Records

PostgreSQL doesn't normally store the complete SQL statement in WAL.

For example:

```sql
UPDATE employee
SET salary = 90000
WHERE employee_id = 100;
```

WAL contains low-level information needed to reproduce database changes.

It records changes associated with things such as:

```text
Heap pages
Index pages
Transaction status
Visibility information
Database metadata
```

This is why physical PostgreSQL replication operates at the storage/page-change level rather than replaying SQL statements.

---

# 5. WAL Buffers

Before WAL reaches disk, WAL records are temporarily stored in memory.

The relevant parameter is:

```text
wal_buffers
```

Architecture:

```text
Backend
   |
   v
Generate WAL
   |
   v
WAL Buffers
   |
   v
WAL Write
   |
   v
pg_wal
```

Check the value:

```sql
SHOW wal_buffers;
```

Often PostgreSQL automatically selects an appropriate value when:

```text
wal_buffers = -1
```

For most systems, manually tuning `wal_buffers` is not the first WAL optimization to perform.

---

# 6. COMMIT and WAL Flush

For normal durable transactions with:

```text
synchronous_commit = on
```

COMMIT may need to wait until the relevant WAL is safely flushed.

Simplified flow:

```text
UPDATE
   |
   v
WAL generated
   |
   v
WAL buffer
   |
   v
COMMIT
   |
   v
WAL flush/fsync
   |
   v
COMMIT acknowledged
```

Therefore, storage latency on the WAL path can directly affect transaction latency.

This is especially noticeable with workloads containing many small transactions.

For example:

```text
Application
    |
    | 10,000 small transactions/sec
    v
PostgreSQL
    |
    v
Frequent WAL flush requirements
    |
    v
Storage latency becomes important
```

---

# 7. Group Commit

PostgreSQL can reduce WAL synchronization overhead through **group commit**.

Instead of thinking:

```text
Transaction 1 -> fsync
Transaction 2 -> fsync
Transaction 3 -> fsync
Transaction 4 -> fsync
```

multiple commits can sometimes share WAL flush work:

```text
TX1 \
TX2  \
TX3   ---> WAL flush
TX4  /
```

This improves throughput when many concurrent transactions are committing.

---

# 8. WAL Segments

PostgreSQL stores WAL in files under:

```text
$PGDATA/pg_wal/
```

Historically, this directory was called:

```text
pg_xlog
```

Each WAL file is called a **WAL segment**.

The default WAL segment size for a standard PostgreSQL cluster is typically:

```text
16 MB
```

You may see files similar to:

```text
0000000100000000000000A1
0000000100000000000000A2
0000000100000000000000A3
```

Conceptually:

```text
WAL stream
-------------------------------------------------->

| Segment 1 | Segment 2 | Segment 3 | Segment 4 |
|   16 MB   |   16 MB   |   16 MB   |   16 MB   |
```

The segment size is selected when the cluster is initialized; it is not an ordinary runtime parameter.

---

# 9. WAL Segment Names

A WAL filename such as:

```text
000000010000000A000000FE
```

encodes information including the **timeline** and location in the WAL stream.

The timeline is particularly important during recovery and failover.

For example:

```text
Primary
 Timeline 1
     |
     | failure
     v
Standby promoted
 Timeline 2
```

A new timeline prevents different recovery histories from being confused.

---

# 10. WAL Location — LSN

PostgreSQL identifies positions inside WAL using an:

**LSN = Log Sequence Number**

Example:

```text
0/16B6C50
```

Check the current WAL location:

```sql
SELECT pg_current_wal_lsn();
```

Example output:

```text
0/3A4F918
```

An LSN can be thought of as an address inside the WAL stream.

```text
WAL Stream
------------------------------------------------------>

LSN A                 LSN B                  LSN C
 |                      |                      |
 v                      v                      v
0/1000000             0/2000000              0/3000000
```

LSNs are heavily used when troubleshooting replication and recovery.

---

# 11. Measuring WAL Generation

You can measure how much WAL a workload generated.

Example:

```sql
SELECT pg_current_wal_lsn();
```

Run your workload.

Then:

```sql
SELECT pg_current_wal_lsn();
```

Compare two LSNs:

```sql
SELECT pg_wal_lsn_diff(
    '0/5000000',
    '0/4000000'
);
```

Or convert it into a readable size:

```sql
SELECT pg_size_pretty(
    pg_wal_lsn_diff(
        pg_current_wal_lsn(),
        '0/4000000'
    )
);
```

This is useful for estimating:

```text
WAL generated/hour
WAL generated/day
Archive storage
Network bandwidth
Replication requirements
Recovery requirements
```

---

# 12. Checkpoints

A checkpoint establishes a recovery point and drives dirty buffers toward persistent data files.

Simplified:

```text
Shared Buffers
--------------------------------
| Clean | Dirty | Dirty | Clean |
--------------------------------
          |       |
          +---+---+
              |
              v
          Checkpoint
              |
              v
          Data Files
```

Checkpoint processing makes sure the required dirty pages are written so PostgreSQL doesn't need an unlimited amount of old WAL for crash recovery.

---

# 13. Why Checkpoints Exist

Imagine PostgreSQL ran for months without checkpointing.

Crash recovery could potentially require replaying an enormous amount of WAL.

Instead PostgreSQL periodically creates checkpoints.

```text
Checkpoint A
     |
     | WAL
     | WAL
     | WAL
     v
Checkpoint B
     |
     | WAL
     v
Crash
```

Recovery can start from the appropriate checkpoint and replay WAL after it.

So checkpoints put a bound on how much WAL normally needs to be replayed during crash recovery.

---

# 14. What Happens During a Checkpoint?

At a high level:

```text
Checkpoint starts
       |
       v
Dirty buffers written toward disk
       |
       v
Required synchronization occurs
       |
       v
Checkpoint WAL record created
       |
       v
Control information updated
       |
       v
Checkpoint complete
```

The details are coordinated between PostgreSQL background processes, especially the checkpointer.

---

# 15. Checkpointer vs Background Writer

These two are commonly confused.

### Background Writer

The background writer continuously tries to write some dirty shared buffers.

```text
Dirty buffers
     |
     v
Background Writer
     |
     v
Data files
```

Its goal includes reducing the amount of buffer-writing work that backend processes may otherwise have to perform.

### Checkpointer

The checkpointer performs checkpoint-related writes and synchronization.

```text
Checkpoint requested
       |
       v
Checkpointer
       |
       v
Write dirty buffers
       |
       v
Synchronize files
```

They are related, but they serve different purposes.

---

# 16. When Does a Checkpoint Happen?

Two major checkpoint triggers are:

### Time based

Controlled by:

```text
checkpoint_timeout
```

For example:

```text
checkpoint_timeout = 5min
```

### WAL-volume based

Controlled primarily by:

```text
max_wal_size
```

Conceptually:

```text
            Checkpoint
                |
       +--------+--------+
       |                 |
       v                 v
 Time threshold      WAL growth
checkpoint_timeout   max_wal_size
```

There can also be explicitly requested checkpoints, for example:

```sql
CHECKPOINT;
```

This command should not be casually executed on production systems.

---

# 17. `max_wal_size`

`max_wal_size` is an important checkpoint tuning parameter.

Example:

```text
max_wal_size = 4GB
```

A common misunderstanding is:

> PostgreSQL's `pg_wal` directory can never become larger than `max_wal_size`.

That is **not** what the setting means.

`max_wal_size` primarily influences checkpoint frequency. WAL can exceed it because of circumstances such as:

```text
WAL archiving delays
Replication slots
Recovery requirements
High WAL generation
Checkpoint behavior
```

So:

```text
max_wal_size ≠ hard pg_wal disk limit
```

This distinction is very important in production troubleshooting.

---

# 18. `min_wal_size`

Another parameter is:

```text
min_wal_size
```

PostgreSQL can recycle WAL files rather than continuously deleting and recreating them.

Conceptually:

```text
Used WAL segment
      |
      v
No longer required
      |
      v
Recycle
      |
      v
Future WAL segment
```

`min_wal_size` influences the amount of WAL space PostgreSQL tries to retain for recycling.

---

# 19. `checkpoint_completion_target`

Writing all dirty pages at once would create an I/O spike.

Bad pattern:

```text
I/O

       ███████████
       ███████████
_______███████████________
       checkpoint
```

PostgreSQL therefore spreads checkpoint writes over time.

Controlled by:

```text
checkpoint_completion_target
```

For example:

```text
checkpoint_completion_target = 0.9
```

Conceptually:

```text
Checkpoint interval
|--------------------------------------|

Write
███████████████████████████████████

instead of

                                ███████
```

This creates smoother I/O behavior.

---

# 20. Frequent Checkpoints Are Usually Bad

Suppose:

```text
checkpoint_timeout = 5 min
max_wal_size = 1 GB
```

but the database generates:

```text
5 GB WAL every 5 minutes
```

The WAL-volume threshold may cause checkpoints more frequently than desired.

This can increase:

```text
Disk writes
Checkpoint overhead
Full-page writes
I/O pressure
Latency
```

A common tuning objective is therefore:

> Avoid unnecessarily frequent requested checkpoints while keeping recovery time and storage requirements acceptable.

---

# 21. Full-Page Writes

One important WAL concept is:

```text
full_page_writes
```

Normally:

```text
full_page_writes = on
```

After a checkpoint, the first modification of a page may cause PostgreSQL to log a complete page image into WAL.

Why?

Because a system crash during a page write could theoretically leave a partially written/torn page.

The full-page image gives recovery enough information to reconstruct the page.

Conceptually:

```text
Checkpoint
    |
    v
Page A modified first time
    |
    +--> Full-page image in WAL
    |
Page A modified again
    |
    +--> Normal WAL information
```

This is one reason checkpoints can affect WAL volume.

---

# 22. Checkpoints and WAL Volume

Consider a write-heavy database.

```text
Checkpoint
   |
   v
Many pages modified
   |
   v
Many first modifications
   |
   v
Full-page images
   |
   v
Higher WAL generation
```

Then another checkpoint happens quickly:

```text
Checkpoint again
   |
   v
Pages become eligible for first-change FPIs again
   |
   v
More WAL
```

Therefore:

```text
Frequent checkpoints
        ↓
Potentially more full-page images
        ↓
More WAL
        ↓
More disk/network/archive pressure
```

This relationship is useful when diagnosing unexpectedly high WAL generation.

---

# 23. WAL Compression

PostgreSQL supports:

```text
wal_compression
```

It can compress full-page images written into WAL.

Conceptually:

```text
Full Page
   |
   v
Compression
   |
   v
Smaller WAL record
```

Potential benefit:

```text
Less WAL disk usage
Less archive volume
Less replication bandwidth
```

Trade-off:

```text
Additional CPU work
```

Whether it helps depends on the workload and system resources.

---

# 24. WAL Archiving

WAL can be copied from `pg_wal` to long-term archive storage.

Architecture:

```text
PostgreSQL
    |
    v
 pg_wal
    |
    v
archive_command / archive_library
    |
    v
WAL Archive
```

For example:

```text
Primary Server
    |
    +--> pg_wal
             |
             +--> Archive Storage
                       |
                       +--> Backup/PITR
```

Important settings include:

```text
archive_mode
archive_command
```

or supported archive-library mechanisms in newer PostgreSQL versions.

---

# 25. Why Archive WAL?

Suppose a base backup was taken Sunday at midnight.

```text
Sunday 00:00
Base Backup
     |
     + WAL
     + WAL
     + WAL
     + WAL
     |
Wednesday 14:00
```

If all required WAL files have been archived, PostgreSQL can restore the base backup and replay WAL toward a chosen recovery point.

This enables **Point-in-Time Recovery**.

---

# 26. Point-in-Time Recovery

Suppose someone accidentally executes:

```sql
DROP TABLE customers;
```

at:

```text
10:30:00
```

You may want the database restored to:

```text
10:29:59
```

Conceptually:

```text
Base Backup
     |
     v
Restore
     |
     v
Replay archived WAL
     |
     v
10:29:59
     |
     STOP
```

This is PITR.

It depends on having:

```text
Valid base backup
+
Continuous required WAL
```

---

# 27. WAL Archive Failure

Consider:

```text
archive_mode = on
```

but archive storage becomes unavailable.

WAL files that still need archiving cannot simply be discarded.

The result can be:

```text
WAL generated
     |
     v
pg_wal
     |
     X
Archive destination unavailable
     |
     v
WAL accumulates
     |
     v
Filesystem fills
```

This is a serious production failure scenario.

Monitor archiving with:

```sql
SELECT *
FROM pg_stat_archiver;
```

Useful fields include information about:

```text
archived_count
failed_count
last_archived_wal
last_archived_time
last_failed_wal
last_failed_time
```

---

# 28. Replication and WAL

Physical streaming replication also depends on WAL.

```text
Primary
   |
   | WAL
   v
Standby
   |
   v
WAL Replay
   |
   v
Replica Database
```

Primary:

```text
Generate WAL
    |
    v
WAL sender
    |
 network
    |
    v
WAL receiver
    |
    v
Standby
```

Therefore WAL generation rate directly influences replication bandwidth requirements.

---

# 29. Replication Slots and WAL Growth

Replication slots can prevent PostgreSQL from removing WAL that a consumer may still need.

Example:

```text
Primary
  |
  +---- Replica A healthy
  |
  +---- Replica B offline
           |
           v
      Replication Slot
           |
           v
     WAL must be retained
```

If Replica B remains offline for a long time:

```text
WAL
WAL
WAL
WAL
WAL
 ↓
pg_wal grows
 ↓
Disk risk
```

Inspect slots:

```sql
SELECT *
FROM pg_replication_slots;
```

A particularly useful field is:

```text
restart_lsn
```

You can estimate retained WAL using the relevant LSN positions.

---

# 30. WAL Sizing

For architecture planning, start with the **WAL generation rate**.

Suppose:

```text
Average WAL generation = 100 GB/day
Peak WAL generation    = 10 GB/hour
```

For archive planning:

```text
Daily archive requirement
≈ WAL generated per day
```

If retaining 7 days:

```text
100 GB/day × 7 days
= 700 GB
```

Then add operational headroom.

Real environments also need to consider:

```text
Compression
Peak workload
Retention policy
Failed archiving
Replication delays
Backup requirements
Growth
```

---

# 31. WAL Rate Calculation

You can capture:

```sql
SELECT pg_current_wal_lsn();
```

At 10:00:

```text
0/10000000
```

At 11:00:

```text
0/50000000
```

Calculate:

```sql
SELECT pg_size_pretty(
    pg_wal_lsn_diff(
        '0/50000000',
        '0/10000000'
    )
);
```

If the result were approximately:

```text
1 GB
```

then:

```text
WAL rate ≈ 1 GB/hour
```

Approximate daily rate:

```text
1 GB × 24
≈ 24 GB/day
```

This is far better than blindly guessing WAL storage requirements.

---

# 32. Important WAL Parameters

Some important settings to know are:

```text
wal_level
wal_buffers
wal_compression
wal_writer_delay
wal_writer_flush_after
synchronous_commit
full_page_writes

checkpoint_timeout
checkpoint_completion_target
checkpoint_flush_after

min_wal_size
max_wal_size

archive_mode
archive_command
archive_timeout

max_wal_senders
max_replication_slots
wal_keep_size
```

Not every parameter should be manually changed.

The goal is to understand what part of the WAL/checkpoint architecture each setting controls.

---

# 33. `wal_level`

`wal_level` determines how much information PostgreSQL puts into WAL.

Common modern values include:

```text
minimal
replica
logical
```

Conceptually:

```text
minimal
   |
   v
replica
   |
   v
logical

Increasing information available in WAL
```

`replica` supports the WAL information needed for physical replication and related recovery use cases.

`logical` adds information required for logical decoding.

---

# 34. `synchronous_commit`

Normally:

```text
synchronous_commit = on
```

A transaction waits for the required WAL durability condition before PostgreSQL reports success.

For some workloads:

```text
synchronous_commit = off
```

can reduce commit latency.

But there is a durability trade-off: after an operating-system/database crash, some transactions already reported as committed to the client may be lost.

It does **not** mean database corruption is expected; it changes the durability guarantee for recent transactions.

This setting should therefore be changed only when the business requirement allows that trade-off.

---

# 35. Checkpoint Monitoring

Modern PostgreSQL versions provide checkpoint-related statistics through system statistics views.

Depending on PostgreSQL version, you may use views such as:

```sql
SELECT *
FROM pg_stat_checkpointer;
```

Older versions expose checkpoint counters through:

```sql
SELECT *
FROM pg_stat_bgwriter;
```

Version matters here because PostgreSQL has moved some statistics into separate views over time.

Useful questions are:

```text
How many checkpoints occur?
Why are they occurring?
How much data is written?
How long does checkpoint writing take?
How long does synchronization take?
```

---

# 36. Monitoring WAL Statistics

PostgreSQL provides:

```sql
SELECT *
FROM pg_stat_wal;
```

This can expose statistics such as:

```text
wal_records
wal_fpi
wal_bytes
wal_buffers_full
wal_write
wal_sync
wal_write_time
wal_sync_time
```

Exact columns can differ by PostgreSQL version.

This view is extremely useful when diagnosing WAL behavior.

---

# 37. WAL Performance Troubleshooting

Suppose an application reports:

```text
COMMIT is slow
```

A useful investigation path is:

```text
Slow COMMIT
    |
    +--> WAL storage latency?
    |
    +--> Excessive WAL generation?
    |
    +--> synchronous replication waits?
    |
    +--> Storage saturation?
    |
    +--> Too many small transactions?
```

Check PostgreSQL waits and statistics rather than immediately changing configuration.

---

# 38. Checkpoint Performance Troubleshooting

Suppose every few minutes you see:

```text
Latency spike
CPU okay
Disk writes suddenly high
```

Potential architecture:

```text
Checkpoint starts
      |
      v
Large dirty-buffer write workload
      |
      v
Storage saturated
      |
      v
Application latency increases
```

Investigate:

```text
Checkpoint frequency
max_wal_size
checkpoint_timeout
checkpoint_completion_target
WAL generation rate
Storage throughput
Dirty buffer volume
```

Do not assume increasing `max_wal_size` alone fixes the system.

---

# 39. WAL Storage Architecture

For a write-heavy PostgreSQL database, storage characteristics matter significantly.

```text
                    PostgreSQL
                        |
          +-------------+-------------+
          |                           |
          v                           v
      Data Files                     WAL
   Random-ish I/O             Sequential-heavy I/O
```

WAL generally benefits from:

```text
Low write latency
Reliable fsync behavior
Good sustained throughput
Reliable storage
```

Some architectures place WAL on separate storage.

```text
PostgreSQL
   |
   +--> Data Volume
   |
   +--> WAL Volume
```

This can help when data-file and WAL I/O are competing, but it is **not automatically better**. Modern SSD/NVMe systems may perform very well with a single properly designed storage layer.

Measure first.

---

# 40. WAL and Large Bulk Loads

Consider:

```sql
INSERT INTO huge_table
SELECT ...
FROM source;
```

This may generate substantial WAL.

Effects can include:

```text
Higher pg_wal usage
More archive traffic
More replication traffic
Replica lag
More checkpoint pressure
Higher storage I/O
```

Therefore bulk operations should be evaluated not only by table growth but also by:

```text
WAL generation
Archive capacity
Replica capacity
Storage throughput
```

---

# 41. WAL and Indexes

An update may affect much more than the table row.

For example:

```sql
UPDATE customer
SET email = 'new@example.com'
WHERE customer_id = 100;
```

If `email` is indexed:

```text
Heap change
    +
Index change
    +
WAL records
```

A table with many indexes can therefore generate significantly more WAL during heavy DML.

Architecture review should consider:

```text
Table writes
+
Index writes
+
WAL
+
Replication
+
Archiving
```

---

# 42. WAL and Autovacuum

Maintenance activity can also generate WAL.

For example:

```text
VACUUM-related page changes
Index maintenance
Visibility map changes
Freezing-related changes
```

can contribute to WAL depending on the operation.

Therefore unexpected WAL growth is not always caused directly by application INSERT/UPDATE/DELETE statements.

Look at the complete database workload.

---

# 43. WAL and Backup Architecture

A robust PostgreSQL recovery architecture generally looks like:

```text
                  PostgreSQL Primary
                         |
             +-----------+-----------+
             |                       |
             v                       v
        Base Backup                 WAL
             |                       |
             v                       v
       Backup Storage          WAL Archive
             |                       |
             +-----------+-----------+
                         |
                         v
                     Recovery
                         |
                         v
                        PITR
```

Neither piece should be considered sufficient alone for typical PITR requirements.

You need a usable backup plus the required WAL sequence.

---

# 44. WAL Continuity

For recovery, WAL continuity is critical.

Suppose:

```text
WAL 100
WAL 101
WAL 102
WAL 103
WAL 104
```

But WAL 102 is missing:

```text
100
101
XXX
103
104
```

Recovery cannot simply jump over the missing required WAL segment.

That is why WAL archive monitoring is critical.

A backup strategy isn't complete just because:

```text
"Backups are running."
```

You also need to verify:

```text
Backups can be restored
Required WAL exists
WAL can be replayed
Recovery procedures work
```

---

# 45. Production Failure Scenario — `pg_wal` Full

One of the most important operational scenarios is:

```text
Heavy transactions
       |
       v
Large WAL generation
       |
       +--> Archive failing
       |
       +--> Replica offline + slot
       |
       v
pg_wal grows
       |
       v
Filesystem reaches 100%
       |
       v
PostgreSQL availability at risk
```

Investigation should include:

```sql
SELECT * FROM pg_stat_archiver;
```

```sql
SELECT * FROM pg_replication_slots;
```

```sql
SELECT * FROM pg_stat_replication;
```

and operating-system filesystem monitoring.

Never start manually deleting files from `pg_wal` as a routine fix.

That can make the cluster unrecoverable.

---

# 46. Production Failure Scenario — Replication Lag

Suppose:

```text
Primary WAL generation
        =
500 MB/min

Replica replay capacity
        =
300 MB/min
```

Then:

```text
Backlog growth
≈ 200 MB/min
```

After one hour:

```text
200 MB × 60
≈ 12 GB
```

The replica cannot catch up while the workload remains at those rates.

This illustrates an important architecture principle:

> Replica capacity needs to handle the WAL/replay workload, especially during peak periods.

---

# 47. Production Failure Scenario — Checkpoint Storm

Imagine:

```text
High write workload
      |
      v
WAL grows quickly
      |
      v
max_wal_size pressure
      |
      v
Checkpoint
      |
      v
Heavy writes
      |
      v
Storage latency
      |
      v
Application slows
```

Possible investigation:

```text
1. Measure WAL generation rate
2. Check checkpoint statistics
3. Check storage latency/throughput
4. Review max_wal_size
5. Review checkpoint_timeout
6. Review checkpoint_completion_target
7. Check full-page-image/WAL volume
8. Determine whether storage can sustain the workload
```

---

# 48. WAL Capacity Planning Example

Assume a production database generates:

```text
Average WAL = 40 GB/hour
Peak WAL    = 80 GB/hour
```

A replica might occasionally be unavailable for:

```text
4 hours
```

Potential WAL requirement based only on peak generation:

```text
80 GB/hour × 4 hours
= 320 GB
```

Then operational safety margin must be added.

For example, if an architecture team chose a 50% margin:

```text
320 GB × 1.5
= 480 GB
```

This is only an example methodology, not a universal sizing rule.

Actual sizing must consider:

```text
Replication slots
Archiving
Checkpoint requirements
Backup behavior
Peak duration
Disk capacity
Recovery requirements
Operational margin
```

---

# 49. Key Architecture Relationships

Remember this chain:

```text
Application Writes
       |
       v
Dirty Buffers
       |
       +----------------+
       |                |
       v                v
Data Writes          WAL Generation
       |                |
       v                +--> WAL Storage
Checkpoints             |
                        +--> Replication
                        |
                        +--> Archiving
                        |
                        +--> Recovery
```

One workload change can therefore affect multiple systems.

For example:

```text
Application write rate doubles
           |
           v
More dirty pages
           |
           +--> More WAL
           |
           +--> More checkpoint I/O
           |
           +--> More archive storage
           |
           +--> More replication bandwidth
           |
           +--> Potential replica lag
```

This is the architectural way to think about WAL.

---

# 50. Important DBA Queries

### Current WAL position

```sql
SELECT pg_current_wal_lsn();
```

### WAL statistics

```sql
SELECT *
FROM pg_stat_wal;
```

### Archiving status

```sql
SELECT *
FROM pg_stat_archiver;
```

### Replication status

```sql
SELECT *
FROM pg_stat_replication;
```

### Replication slots

```sql
SELECT *
FROM pg_replication_slots;
```

### Checkpoint statistics — newer versions

```sql
SELECT *
FROM pg_stat_checkpointer;
```

### Background writer statistics

```sql
SELECT *
FROM pg_stat_bgwriter;
```

### Important configuration

```sql
SHOW wal_level;
SHOW wal_buffers;
SHOW wal_compression;
SHOW synchronous_commit;
SHOW full_page_writes;

SHOW checkpoint_timeout;
SHOW checkpoint_completion_target;

SHOW min_wal_size;
SHOW max_wal_size;

SHOW archive_mode;
SHOW archive_command;
```

---

# 51. Interview Questions

**Q: What is WAL?**

WAL stands for Write-Ahead Logging. PostgreSQL writes the WAL information describing a change before the corresponding modified database page is allowed to reach persistent storage. WAL provides durability and enables crash recovery, replication, backup recovery, and PITR.

**Q: Why doesn't PostgreSQL write table pages at every COMMIT?**

Because that would create expensive random I/O. PostgreSQL persists the necessary WAL and allows dirty data pages to be written later.

**Q: What is a checkpoint?**

A checkpoint establishes a recovery point and coordinates writing dirty buffers and synchronizing database files so crash recovery can start from a bounded point rather than requiring an unlimited WAL history.

**Q: What controls checkpoint frequency?**

Important factors include:

```text
checkpoint_timeout
max_wal_size
```

as well as explicit or internally requested checkpoints.

**Q: Is `max_wal_size` a hard limit on `pg_wal`?**

No.

WAL can exceed `max_wal_size`, particularly because of archiving problems, replication slots, recovery requirements, and high WAL generation.

**Q: Why can frequent checkpoints increase WAL?**

Because after each checkpoint, the first change to many pages may require full-page images when `full_page_writes` is enabled.

**Q: What is an LSN?**

LSN stands for Log Sequence Number. It identifies a position in PostgreSQL's WAL stream.

**Q: Why does `pg_wal` keep growing?**

Common causes include:

```text
Archive failure
Replication slot retaining WAL
Replica lag
High WAL generation
Backup/recovery requirements
Checkpoint/WAL configuration
```

**Q: Can we manually delete old files from `pg_wal`?**

No. Manually deleting WAL files is dangerous and can break recovery or the database cluster.

**Q: What is required for PITR?**

A valid base backup plus the complete required WAL sequence needed to reach the desired recovery point.

---

# 52. Production Troubleshooting Flow

When someone says:

> **"PostgreSQL WAL is consuming all disk space."**

Think through it systematically:

```text
pg_wal growing
     |
     +--> Is archiving working?
     |       |
     |       +--> pg_stat_archiver
     |
     +--> Replication slot holding WAL?
     |       |
     |       +--> pg_replication_slots
     |
     +--> Replica lag?
     |       |
     |       +--> pg_stat_replication
     |
     +--> Abnormally high WAL generation?
     |       |
     |       +--> pg_stat_wal
     |
     +--> Large/bulk transaction?
     |
     +--> Heavy write workload?
     |
     +--> Backup/recovery process retaining WAL?
     |
     +--> Check WAL/checkpoint configuration
```

This is much better than immediately increasing disk space.

Increasing storage may buy time, but you still need to find **why WAL isn't being recycled or why WAL generation increased**.

---

# 53. Architecture-Level Summary

The complete picture is:

```text
                         PostgreSQL
                             |
                       Application DML
                             |
             +---------------+---------------+
             |                               |
             v                               v
       Shared Buffers                    WAL Buffers
             |                               |
        Dirty Pages                          v
             |                             WAL
             |                               |
             |                    +----------+----------+
             |                    |          |          |
             v                    v          v          v
        Checkpointer          pg_wal      Archive   Replication
             |                               |
             v                               v
         Data Files                         PITR
             |
             v
       Persistent Data
```

The core idea to remember is:

> **WAL protects changes before the corresponding data pages are persisted; checkpoints control how dirty data reaches persistent storage and bound crash-recovery work.**

For production architecture, always think of WAL and checkpoints together:

```text
WAL generation rate
        +
Checkpoint behavior
        +
Storage latency
        +
Archive capacity
        +
Replication capacity
        +
Recovery requirements
        =
Reliable PostgreSQL write architecture
```

### Quick memory model

```text
WAL       = Protect the change
COMMIT    = Make transaction durable
Checkpoint= Persist dirty pages / establish recovery point
pg_wal    = Active WAL storage
Archive   = Preserve WAL for recovery
LSN       = Position in WAL
PITR      = Base backup + WAL replay
Slot      = Can retain WAL for a consumer
```

The most important production lesson is **never tune WAL parameters independently**. Increasing `max_wal_size`, changing checkpoint frequency, changing durability settings, or modifying archive/replication behavior can affect storage, recovery time, replication, backups, and application latency at the same time.
