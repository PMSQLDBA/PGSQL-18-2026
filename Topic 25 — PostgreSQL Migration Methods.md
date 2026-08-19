# Topic 25 — PostgreSQL Migration Methods

Verified these notes against the current PostgreSQL official documentation. 

As of August 2026, the current PostgreSQL documentation is **PostgreSQL 18**; the current 18.x release documentation includes **18.4 (May 14, 2026)**. ([PostgreSQL][1])

PostgreSQL migrations are easiest to understand as a choice between moving **logical database objects**, moving the **physical cluster**, or continuously **replicating changes** until the target is ready.

---

## 1. Migration methods at a glance

| Method                   | Basic idea                                       |    Downtime |                Cross-version |      PostgreSQL → other DB | Best use                              |
| ------------------------ | ------------------------------------------------ | ----------: | ---------------------------: | -------------------------: | ------------------------------------- |
| `pg_dump` / `pg_restore` | Export objects/data and reload                   | Medium–High |                          Yes |             Limited/manual | Simple, reliable migrations           |
| `pg_upgrade`             | Upgrade existing physical cluster                |  Low–Medium |                          Yes |                         No | PostgreSQL major-version upgrade      |
| Physical replication     | Copy database cluster/WAL                        |    Very low | Generally same major version |                         No | Same-version infrastructure migration |
| Logical replication      | Initial copy + replicate row changes             |    Very low |                          Yes |               Not directly | Major upgrade / near-zero downtime    |
| Logical decoding / CDC   | Read changes from WAL                            |    Very low |                          Yes | Yes, with external tooling | Heterogeneous migration               |
| Physical seed → logical  | Seed target physically, then logical replication |    Very low |      Depends on architecture |                         No | Large PostgreSQL migrations           |

PostgreSQL itself describes logical replication as replication based on database objects and their replication identity, in contrast to physical replication's block/byte-level approach. ([PostgreSQL][2])

---

# 2. Dump and Restore Migration

This is the traditional PostgreSQL migration method.

Conceptually:

```text
Source PostgreSQL
       |
       | pg_dump
       v
   Dump File
       |
       | pg_restore / psql
       v
Target PostgreSQL
```

PostgreSQL's `pg_dump` creates a **consistent snapshot** of a database. Normal database activity can continue while the dump runs, although operations requiring conflicting exclusive locks can interfere. ([PostgreSQL][3])

### Main tools

```text
pg_dump
pg_dumpall
pg_restore
psql
```

`pg_dump` operates on one database.

```bash
pg_dump mydb > mydb.sql
```

Restore:

```bash
psql mydb < mydb.sql
```

For larger databases, prefer an archive format.

```bash
pg_dump -Fc mydb > mydb.dump
```

Then:

```bash
pg_restore -d mydb mydb.dump
```

`pg_restore` restores archives produced by `pg_dump` in non-plain-text formats. ([PostgreSQL][4])

### Directory format and parallelism

For large databases:

```bash
pg_dump -Fd -j 8 -f backup_dir mydb
```

Restore:

```bash
pg_restore -j 8 -d mydb backup_dir
```

Parallel dump/restore can substantially reduce migration time.

---

## 3. `pg_dump` vs `pg_dumpall`

A common interview point:

### `pg_dump`

Works on **one database**.

It captures objects such as:

```text
schemas
tables
indexes
constraints
views
functions
sequences
table data
```

### `pg_dumpall`

Works across the PostgreSQL **cluster** and can include global objects.

```bash
pg_dumpall > cluster.sql
```

Global objects include things such as:

```text
roles
tablespaces
```

PostgreSQL documents `pg_dumpall` as dumping all databases in a cluster into a SQL script. ([PostgreSQL][5])

A common migration pattern is therefore:

```bash
pg_dumpall --globals-only > globals.sql
pg_dump -Fc appdb > appdb.dump
```

Then create/restore the required roles and globals on the destination before restoring application databases.

---

# 4. Advantages and disadvantages of dump/restore

### Advantages

It is:

```text
simple
well understood
portable
good for version upgrades
good for architecture changes
easy to test
easy to selectively restore
```

Because this is logical migration, the destination does not have to have the same physical data representation.

For example:

```text
PostgreSQL 14
      |
   pg_dump
      |
      v
PostgreSQL 18
```

This is a normal migration approach.

### Disadvantages

The main problem is **migration time**.

Suppose you have:

```text
Database size = 5 TB
Dump          = 5 hours
Transfer      = 2 hours
Restore       = 8 hours
Validation    = 1 hour
```

A naive migration could require a very large maintenance window.

For a production system with a strict downtime SLA, dump/restore alone may therefore be unsuitable.

---

# 5. Physical Migration

Physical migration moves PostgreSQL's physical database representation rather than recreating tables through SQL.

Think:

```text
Logical migration

Table → rows → SQL/COPY → recreate database


Physical migration

Data files / blocks / WAL → destination
```

PostgreSQL's backup documentation distinguishes SQL dumps, file-system-level backups and continuous archiving as fundamentally different approaches. ([PostgreSQL][6])

Typical physical mechanisms include:

```text
pg_basebackup
physical streaming replication
filesystem/storage snapshot
backup + WAL recovery
pg_upgrade
```

---

# 6. `pg_basebackup`

`pg_basebackup` creates a base backup of a running PostgreSQL cluster.

Example:

```bash
pg_basebackup \
  -h source-db \
  -D /pgdata \
  -U replication \
  -P
```

The important point is that it backs up the **entire cluster**, not an individual database.

PostgreSQL states that `pg_basebackup` can take a base backup from a running cluster without preventing normal client access and that the backup can be used for PITR or as the starting point for a streaming-replication standby. ([PostgreSQL][7])

Architecture:

```text
Primary PostgreSQL
        |
        | base backup
        v
Target PostgreSQL
        |
        | WAL streaming
        v
     Standby
```

Once synchronized:

```text
stop writes
wait for WAL catch-up
promote target
redirect application
```

Downtime can be seconds or minutes.

---

# 7. Physical Streaming Replication for Migration

This works well when source and target are physically compatible.

Architecture:

```text
          WAL
Source -----------> Target
Primary             Standby

Application
    |
    v
 Source

After synchronization:

Application
    |
    +--------------> Target
```

During the preparation period, users continue using the source.

At cutover:

```text
1. Stop/redirect writes
2. Wait for target to replay remaining WAL
3. Verify synchronization
4. Promote target
5. Change application connection
6. Validate
```

The downtime is primarily the cutover period rather than the time required to copy the whole database.

---

# 8. Limitation of Physical Migration

Physical replication works with the actual PostgreSQL storage representation.

Therefore it is not the normal mechanism for:

```text
PostgreSQL 14 → PostgreSQL 18
```

because PostgreSQL major releases can have incompatible internal storage/catalog formats.

For a major-version upgrade, PostgreSQL documents options including:

```text
dump/restore
pg_upgrade
logical replication
```

and specifically notes that logical replication can replicate between different major versions. ([PostgreSQL][8])

---

# 9. `pg_upgrade`

`pg_upgrade` is another major migration tool worth separating from ordinary physical replication.

Example:

```text
PostgreSQL 14 cluster
        |
    pg_upgrade
        |
        v
PostgreSQL 18 cluster
```

Instead of logically exporting every row and reloading it, `pg_upgrade` converts/reuses the existing cluster files as appropriate for the new server version.

This makes it much faster than dump/restore for very large databases.

Typical workflow:

```text
Install new PostgreSQL binaries
        ↓
Install compatible extensions
        ↓
Run pg_upgrade --check
        ↓
Stop old PostgreSQL
        ↓
Run pg_upgrade
        ↓
Start new PostgreSQL
        ↓
Post-upgrade work
        ↓
Validate
```

The tradeoff is that there is still an outage while the actual upgrade/cutover happens.

So:

```text
pg_dump       → simplest/most portable
pg_upgrade    → fast major-version upgrade
logical repl  → lower-downtime major-version upgrade
```

---

# 10. Logical Replication

Logical replication operates on **logical row changes** rather than physical blocks.

Architecture:

```text
Source PostgreSQL
   Publisher
       |
       | INSERT
       | UPDATE
       | DELETE
       v
Target PostgreSQL
   Subscriber
```

The major advantage:

> Source and target can run different PostgreSQL major versions.

PostgreSQL explicitly recommends logical replication as one possible way of creating a server on the updated version, synchronizing it, and switching over after it catches up. ([PostgreSQL][8])

---

# 11. Publication and Subscription

PostgreSQL native logical replication uses:

```text
Publication  → source
Subscription → destination
```

Example source:

```sql
CREATE PUBLICATION migration_pub
FOR ALL TABLES;
```

Target:

```sql
CREATE SUBSCRIPTION migration_sub
CONNECTION 'host=source dbname=mydb user=repl password=xxx'
PUBLICATION migration_pub;
```

Conceptually:

```text
SOURCE
+----------------------+
| Publication          |
|                      |
| customers            |
| orders               |
| payments             |
+----------+-----------+
           |
           | WAL → logical changes
           |
+----------v-----------+
| Subscription         |
|                      |
| TARGET               |
+----------------------+
```

---

# 12. Replica Identity

Logical replication must be able to identify the row affected by an `UPDATE` or `DELETE`.

Usually:

```text
PRIMARY KEY
```

provides that identity.

For example:

```sql
CREATE TABLE customer (
    customer_id bigint PRIMARY KEY,
    name text
);
```

PostgreSQL documentation states that a published table needs an appropriate **replica identity** for `UPDATE` and `DELETE` replication. ([PostgreSQL][9])

If necessary:

```sql
ALTER TABLE some_table
REPLICA IDENTITY FULL;
```

This can allow PostgreSQL to identify rows using the old row contents, but it can increase WAL volume and replication cost. A proper primary key is generally preferable.

---

# 13. What Logical Replication Does Not Mean

A critical migration point:

Logical replication is primarily about **data changes**.

Do not assume that creating a publication/subscription magically performs your entire schema migration.

You normally prepare the destination schema separately.

For example:

```bash
pg_dump --schema-only sourcedb > schema.sql
psql targetdb < schema.sql
```

Then configure logical replication.

Architecture:

```text
Step 1
Source schema
     |
 pg_dump --schema-only
     v
Target schema


Step 2
Source data
     |
 logical replication
     v
Target data
```

This separation is important when designing an upgrade runbook.

---

# 14. Initial Data Synchronization

When a subscription is created, PostgreSQL can copy the existing table contents and then continue applying changes.

Conceptually:

```text
              Existing rows
Source ----------------------------> Target
   |
   | INSERT/UPDATE/DELETE continue
   |
   +------------- WAL -------------> Target
```

Once initial synchronization finishes:

```text
Source
   |
   | continuous logical changes
   v
Target
```

Eventually:

```text
source ≈ target
```

At that point, only a short cutover is required.

---

# 15. Logical Replication Requirements

Important server settings include:

```conf
wal_level = logical
```

and sufficient capacity for replication slots and replication workers/connections.

For logical decoding specifically, PostgreSQL documents `wal_level = logical` and the need for replication-slot capacity. ([PostgreSQL][10])

Also configure:

```text
pg_hba.conf
network connectivity
replication user
permissions
publication
subscription
```

Never size replication slots casually. A slot can cause the publisher to retain WAL needed by a consumer.

---

# 16. Replication Slots

A logical replication slot represents a stream of changes that a consumer can replay in order.

PostgreSQL describes a logical replication slot as representing such a stream for a single database. ([PostgreSQL][11])

Think:

```text
PostgreSQL WAL
     |
     v
Replication Slot
     |
     v
Consumer
```

The slot effectively tells PostgreSQL:

```text
"Keep the WAL/change information that this consumer
has not consumed yet."
```

This is useful for reliability.

But it introduces an operational risk.

If the consumer stops:

```text
Consumer DOWN
      ↓
Slot stops advancing
      ↓
WAL retained
      ↓
Disk usage increases
      ↓
Possible disk exhaustion
```

Therefore replication-slot health and WAL retention must be monitored during migrations.

---

# 17. CDC — Change Data Capture

CDC is a broader architecture.

It means capturing database changes continuously.

Typical PostgreSQL CDC architecture:

```text
Application
     |
     v
PostgreSQL
     |
     | WAL
     v
Logical Decoding
     |
     v
CDC Connector
     |
     +----------> Target DB
     |
     +----------> Kafka
     |
     +----------> Data Lake
```

PostgreSQL provides **logical decoding** as the database mechanism for extracting persistent table changes into a logical format.

PostgreSQL documents that logical-decoding changes are exposed through logical replication slots and formatted using an output plugin. ([PostgreSQL][12])

---

# 18. Logical Decoding

Normal WAL contains information designed for PostgreSQL recovery.

Logical decoding turns the relevant WAL information into logical changes.

Conceptually:

```text
WAL record
   |
   v
Logical decoding
   |
   v
INSERT customer ...
UPDATE order ...
DELETE payment ...
```

Output plugins control the representation.

PostgreSQL also provides `pg_recvlogical`, which can manage logical decoding slots and stream data from them. ([PostgreSQL][13])

CDC platforms build more complete systems around this concept.

---

# 19. CDC vs Native Logical Replication

Do not treat these as identical.

### Native logical replication

```text
PostgreSQL
     |
     v
PostgreSQL
```

PostgreSQL itself understands how to apply the changes.

### CDC

```text
PostgreSQL
     |
 logical decoding
     |
     v
CDC system
     |
 transformation/mapping
     |
     v
Other destination
```

Destination might be:

```text
another PostgreSQL
Kafka
analytics platform
search system
cloud database
different database engine
```

So CDC becomes especially useful for **heterogeneous migrations**.

---

# 20. Heterogeneous Migration

A homogeneous migration means:

```text
PostgreSQL → PostgreSQL
```

A heterogeneous migration means:

```text
Oracle → PostgreSQL
SQL Server → PostgreSQL
MySQL → PostgreSQL

or

PostgreSQL → another database engine
```

Native PostgreSQL logical replication is not a general Oracle/SQL Server/MySQL migration engine.

A heterogeneous migration usually requires:

```text
schema conversion
data-type mapping
initial bulk load
CDC
SQL conversion
stored-procedure conversion
application testing
validation
cutover
```

---

# 21. Why Heterogeneous Migration Is Harder

Consider:

```text
Oracle NUMBER
     ↓
PostgreSQL numeric / integer / bigint
```

or:

```text
Oracle VARCHAR2
     ↓
PostgreSQL varchar / text
```

The migration team must map types correctly.

But data types are only one part.

You may also need to convert:

```text
sequences
identity columns
functions
procedures
packages
triggers
views
materialized views
partitioning
indexes
constraints
SQL syntax
date handling
NULL behavior
transaction behavior
```

For example:

```text
Oracle PL/SQL
     ↓
PostgreSQL PL/pgSQL
```

This is a **code conversion**, not merely data movement.

---

# 22. Typical Heterogeneous Migration Architecture

A common design:

```text
          INITIAL LOAD
Source DB =======================> PostgreSQL
    |
    |
    | transaction logs
    v
 CDC / Migration Tool
    |
    |
    +===========================> PostgreSQL

             ongoing changes
```

The initial load moves historical data.

CDC handles transactions that occur after/during that load.

Eventually:

```text
Target = historical data + subsequent changes
```

Then the migration team performs cutover.

---

# 23. Near-Zero-Downtime Migration

This is one of the most important production patterns.

Suppose the source database is several TB.

Stopping the application and copying everything is unacceptable.

Instead:

```text
          Initial Copy
Source =================> Target
   |
   | application continues writing
   |
   +------ changes ------> Target
```

The target gradually catches up.

At cutover:

```text
Stop/redirect writes
        ↓
Drain active transactions
        ↓
Wait for replication lag ≈ 0
        ↓
Final validation
        ↓
Switch application
        ↓
Target becomes primary
```

Therefore:

```text
Migration duration ≠ application downtime
```

The migration may take several days while application downtime lasts only minutes.

---

# 24. Near-Zero-Downtime Major-Version Upgrade

Example:

```text
PostgreSQL 14
     |
     | logical replication
     v
PostgreSQL 18
```

Workflow:

```text
1. Build PostgreSQL 18 target
2. Install required extensions
3. Create roles/tablespaces as required
4. Create target schema
5. Configure publication
6. Configure subscription
7. Initial data synchronization
8. Allow target to catch up
9. Validate continuously
10. Freeze source writes
11. Wait for remaining replication
12. Synchronize sequences/other non-replicated state
13. Disable subscription appropriately
14. Redirect applications
15. Validate target
16. Keep old environment for rollback window
```

The PostgreSQL upgrade documentation explicitly describes using logical replication between different major versions and switching primary roles once the new server catches up. ([PostgreSQL][8])

---

# 25. Sequence Handling — Major Gotcha

One of the classic logical migration problems is sequences.

Suppose:

```sql
CREATE SEQUENCE order_id_seq;
```

Source has:

```text
last_value = 9,500,000
```

If the target sequence is only:

```text
last_value = 8,000,000
```

after cutover new inserts can generate incorrect/conflicting values.

Therefore sequence synchronization must be part of the cutover plan.

Conceptually:

```text
Source sequence value
        |
        v
capture final value
        |
        v
Target sequence
```

Do not assume table-row logical replication automatically solves every sequence-state requirement.

---

# 26. Large Database Strategy

For very large databases, copying all existing data logically can take considerable time.

One useful pattern is:

```text
Source
   |
   | physical copy
   v
Physical Standby
   |
   | transition
   v
Logical Subscriber
```

This can combine fast physical seeding with logical replication flexibility.

Modern PostgreSQL provides:

```text
pg_createsubscriber
```

PostgreSQL documents this utility as converting a physical replica into a logical replica and creating the publication/subscription setup. ([PostgreSQL][14])

In PostgreSQL 18, `pg_createsubscriber` also gained an `--all` option for creating logical replicas for all databases in an instance. ([PostgreSQL][1])

This is an important newer capability to know for modern PostgreSQL migration architecture.

---

# 27. Migration Validation

Never treat:

```text
replication lag = 0
```

as proof that the migration is correct.

You need multiple validation layers.

A useful framework is:

```text
Infrastructure validation
Schema validation
Data validation
Application validation
Performance validation
Business validation
```

---

# 28. Infrastructure Validation

Check:

```text
PostgreSQL version
extensions
parameters
storage
tablespaces
roles
permissions
network
SSL
replication configuration
backup configuration
monitoring
```

Example:

```sql
SELECT version();
```

Check extensions:

```sql
SELECT extname, extversion
FROM pg_extension
ORDER BY extname;
```

---

# 29. Schema Validation

Compare:

```text
schemas
tables
columns
data types
defaults
constraints
primary keys
foreign keys
indexes
sequences
views
functions
triggers
partition definitions
permissions
```

A simple approach:

```bash
pg_dump --schema-only source > source_schema.sql
pg_dump --schema-only target > target_schema.sql
```

Then compare the intended differences.

Be careful: version upgrades may legitimately produce different SQL definitions or catalog representations, so raw textual comparison isn't always enough.

---

# 30. Row-Count Validation

Example:

```sql
SELECT count(*) FROM customers;
SELECT count(*) FROM orders;
SELECT count(*) FROM payments;
```

Compare:

```text
Source             Target

customers 10M  =   customers 10M
orders    95M  =   orders    95M
payments  91M  =   payments  91M
```

For thousands of tables, automate this using metadata queries.

But row counts alone do **not** prove identical data.

Two tables can both contain:

```text
10,000,000 rows
```

while having different values.

---

# 31. Data-Level Validation

Use additional checks such as:

```text
MIN/MAX
SUM
grouped counts
business aggregates
checksums/hashes
sample comparisons
primary-key range comparisons
```

Example:

```sql
SELECT
    count(*),
    min(order_id),
    max(order_id),
    sum(amount)
FROM orders;
```

Run the equivalent check on source and target.

For huge tables, validate in chunks:

```text
ID 1–1M
ID 1M–2M
ID 2M–3M
...
```

This makes discrepancies easier to isolate.

---

# 32. Backup Validation

If physical backups are part of the migration, PostgreSQL provides:

```text
pg_verifybackup
```

It checks a backup produced by `pg_basebackup` against the backup manifest. ([PostgreSQL][15])

This validates the **backup structure/integrity**, which is different from validating that the migrated application's logical data is correct.

Both may be required.

---

# 33. Performance Validation

Migration success does not mean:

```text
database starts = success
```

The new system must handle production workload.

Compare:

```text
query latency
transactions/sec
CPU
memory
I/O
connections
cache hit rate
locks
temporary files
WAL generation
checkpoint behavior
```

`pg_stat_statements` is particularly useful because it tracks SQL planning and execution statistics. ([PostgreSQL][16])

For example, compare top SQL before and after migration:

```text
Source
Query A = 20 ms

Target
Query A = 400 ms
```

That is a migration issue even if every row was copied correctly.

---

# 34. Statistics After Migration

Optimizer statistics matter.

After a large load or migration, ensure appropriate statistics exist.

Typical operation:

```sql
ANALYZE;
```

or:

```sql
VACUUM ANALYZE;
```

Without suitable statistics, PostgreSQL can choose poor execution plans.

Example:

```text
Expected:
Index Scan

Because statistics are missing:
Sequential Scan
```

This can make the migrated system appear much slower even though the hardware is better.

---

# 35. Replication Validation

During logical migration, monitor things such as:

```text
subscription status
replication slots
LSNs
apply progress
worker status
conflicts/errors
WAL retention
```

PostgreSQL exposes replication-related information through its statistics system, including statistics for logical replication slots. ([PostgreSQL][17])

A useful conceptual measure:

```text
Publisher current WAL position
            -
Subscriber progress
            =
Replication lag
```

As cutover approaches:

```text
lag → 0
```

---

# 36. Cutover

Cutover is the controlled point where the target becomes authoritative.

Before:

```text
Application
     |
     v
   Source
     |
 replication
     v
   Target
```

After:

```text
Application
     |
     v
   Target

Source
  |
  +---- retained temporarily for rollback
```

A cutover should be written as a precise runbook rather than improvised during the maintenance window.

---

# 37. Example Cutover Runbook

A typical near-zero-downtime PostgreSQL cutover:

```text
T-60 minutes
    Confirm replication healthy
    Confirm no unexpected lag
    Confirm validation passed
    Confirm backup/rollback plan

T-15 minutes
    Notify stakeholders
    Freeze deployments
    Confirm application owners ready

T-5 minutes
    Stop batch jobs
    Stop ETL/integration writers
    Disable scheduled writes

T0
    Put application into maintenance/read-only mode

T+1
    Drain active transactions

T+2
    Wait for replication to catch up

T+3
    Confirm final source/target positions

T+4
    Synchronize sequence values

T+5
    Perform final critical-table validation

T+7
    Redirect application to target

T+8
    Start application

T+10
    Run smoke tests

T+15
    Monitor errors/performance

T+30
    Declare cutover successful
```

The actual times depend on the system; the ordering is more important than these example durations.

---

# 38. Cutover Traffic Switching

Applications can be redirected using several mechanisms:

```text
DNS
load balancer
connection proxy
service discovery
application configuration
secret/config update
```

A proxy abstraction often makes migration easier:

```text
Application
     |
     v
DB Endpoint / Proxy
     |
     +---- Source
```

Cutover changes the backend:

```text
Application
     |
     v
DB Endpoint / Proxy
     |
     +---- Target
```

This avoids changing database hostnames in many individual applications.

---

# 39. Connection Pooling During Cutover

Connection pools can make migrations tricky.

Even after DNS changes:

```text
existing connection
      |
      v
old database
```

may remain alive.

Therefore cutover planning should include:

```text
connection draining
pool restart/reload
connection TTL
DNS TTL
application restart strategy
```

Otherwise some application instances can continue talking to the old database while others use the new database.

That creates a split-brain-like application situation.

---

# 40. Preventing Writes to the Old Database

After successful cutover, you generally do not want:

```text
App A → Target

App B → Source
```

because data immediately diverges.

Possible safeguards include:

```text
remove application access
make old environment read-only where appropriate
stop old application connections
network isolation
revoke writer privileges
shut down old primary after rollback decision
```

The exact mechanism depends on the migration architecture.

---

# 41. Rollback Planning

Every production migration needs a rollback strategy.

Before cutover:

```text
Source = primary
Target = replica
```

Rollback is relatively simple.

After target accepts writes:

```text
Target receives new transactions
```

rollback becomes much harder because:

```text
Source no longer contains those transactions.
```

This is why the migration team must define a **point of no return**.

Example:

```text
Before target writes
        ↓
easy rollback

After target writes
        ↓
requires reverse replication,
data reconciliation,
or loss/forward-fix strategy
```

Rollback must therefore be designed before migration day.

---

# 42. Choosing the Correct Migration Method

### Small database + downtime acceptable

Use:

```text
pg_dump
pg_restore
```

Example:

```text
200 GB database
4-hour maintenance window
```

No need to introduce replication complexity unless there is another requirement.

---

### Large database + fast major-version upgrade

Consider:

```text
pg_upgrade
```

Especially when some outage is acceptable but dump/restore would take far too long.

---

### PostgreSQL → PostgreSQL, same compatible physical environment

Consider:

```text
physical replication
pg_basebackup
storage/backup-based physical migration
```

Good for infrastructure moves where version compatibility permits it.

---

### PostgreSQL 14 → PostgreSQL 18 with very small downtime

Strong candidate:

```text
logical replication
```

because PostgreSQL supports logical replication between different major versions. ([PostgreSQL][8])

---

### Oracle → PostgreSQL

Use a heterogeneous migration architecture:

```text
schema conversion
+
initial bulk load
+
CDC
+
validation
+
cutover
```

Native PostgreSQL publication/subscription alone is not sufficient.

---

### Multi-TB PostgreSQL with strict downtime

Potential architecture:

```text
physical seed
       +
continuous synchronization
       +
logical/CDC transition where required
       +
short controlled cutover
```

For supported PostgreSQL designs, `pg_createsubscriber` is now particularly relevant because it can turn a physical replica into a logical replica. ([PostgreSQL][14])

---

# 43. Architecture Decision Matrix

| Requirement                          | Preferred approach                                                   |
| ------------------------------------ | -------------------------------------------------------------------- |
| Simplest migration                   | Dump/restore                                                         |
| Select specific DB objects           | `pg_dump` / `pg_restore`                                             |
| Cross-major PostgreSQL               | Dump/restore, `pg_upgrade`, or logical replication                   |
| Fast offline major upgrade           | `pg_upgrade`                                                         |
| Same-version infrastructure move     | Physical replication                                                 |
| Very low downtime                    | Logical replication / CDC                                            |
| Multi-TB migration                   | Physical seed + replication, or carefully designed logical migration |
| PostgreSQL → PostgreSQL              | Native logical replication is an option                              |
| Oracle/MySQL/SQL Server → PostgreSQL | Heterogeneous migration tooling + CDC                                |
| Continuous change capture            | Logical decoding / CDC                                               |
| Select tables for replication        | Logical replication                                                  |
| Physical backup verification         | `pg_verifybackup`                                                    |

---

# 44. Important Migration Gotchas

For production migrations, specifically check these:

```text
Sequences
Large objects
Extensions
Roles/users
Passwords/authentication
Tablespaces
Ownership
GRANTs
Default privileges
Replica identity
Primary keys
Foreign keys
Indexes
Partitioning
Generated/identity columns
Triggers
Functions/procedures
Materialized views
Collations/locales
Encoding
Time zones
Statistics
Replication slots
WAL growth
Long-running transactions
Connection pools
DNS TTL
Application connection strings
Backup/PITR
Monitoring
Batch jobs
ETL jobs
Cron/scheduled jobs
External integrations
```

A database migration can technically succeed while the application fails because one of these was overlooked.

---

# 45. Important PostgreSQL 18 Point

A useful current-documentation update for interviews and architecture discussions is `pg_createsubscriber`.

It creates a logical replica from an existing physical standby. PostgreSQL 18 additionally supports `--all` for handling all databases in an instance. ([PostgreSQL][14])

Conceptually:

```text
               Physical replication
Source PG  --------------------------> Standby
                                          |
                                          | pg_createsubscriber
                                          v
                                  Logical Subscriber
```

This can be valuable when the initial logical table copy would otherwise be extremely expensive.

---

# 46. End-to-End Enterprise Migration Pattern

The overall architecture is usually:

```text
┌──────────────────────────────┐
│ 1. Assessment               │
│                              │
│ DB size                      │
│ PostgreSQL versions          │
│ extensions                   │
│ dependencies                 │
│ downtime SLA                 │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 2. Choose Method            │
│                              │
│ Dump / Restore               │
│ pg_upgrade                   │
│ Physical                     │
│ Logical Replication          │
│ CDC                          │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 3. Build Target             │
│                              │
│ PostgreSQL                   │
│ extensions                   │
│ parameters                   │
│ users/security               │
│ monitoring                   │
│ backups                      │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 4. Initial Data Migration   │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 5. Continuous Sync          │
│                              │
│ Logical replication / CDC    │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 6. Validation               │
│                              │
│ Schema                       │
│ Counts                       │
│ Data                         │
│ Performance                  │
│ Application                  │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 7. Cutover                  │
│                              │
│ Stop writes                  │
│ Drain transactions           │
│ Catch up                     │
│ Sync sequences               │
│ Switch endpoint              │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 8. Post-Cutover Validation  │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 9. Rollback Window          │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 10. Decommission Source     │
└──────────────────────────────┘
```

---

# 47. The key concept to remember

For interviews and architecture discussions, reduce the entire topic to this:

```text
                PostgreSQL Migration
                       |
       +---------------+---------------+
       |               |               |
       v               v               v
    Logical         Physical        Continuous
    Migration       Migration       Migration
       |               |               |
       v               v               v
 pg_dump/restore   pg_basebackup    Logical Repl
                   pg_upgrade       CDC
                   streaming repl   Logical Decoding
```

And choose based on:

```text
1. Source and target database engines
2. Source and target PostgreSQL versions
3. Database size
4. Allowed downtime
5. Network bandwidth
6. Required rollback capability
7. Data-change rate
8. Schema/application compatibility
```

The central rule is:

**If downtime is acceptable, prefer the simplest safe method. If downtime must be very small, copy the bulk of the data while production remains online, continuously replicate subsequent changes, validate the target, and make the outage only the final cutover window.**

### Official PostgreSQL 18 references

[PostgreSQL 18 — Backup and Restore](https://www.postgresql.org/docs/current/backup.html?utm_source=chatgpt.com)
[PostgreSQL 18 — pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html?utm_source=chatgpt.com)
[PostgreSQL 18 — pg_restore](https://www.postgresql.org/docs/current/app-pgrestore.html?utm_source=chatgpt.com)
[PostgreSQL 18 — Upgrading a Cluster](https://www.postgresql.org/docs/current/upgrading.html?utm_source=chatgpt.com)
[PostgreSQL 18 — Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html?utm_source=chatgpt.com)
[PostgreSQL 18 — Logical Decoding](https://www.postgresql.org/docs/current/logicaldecoding.html?utm_source=chatgpt.com)
[PostgreSQL 18 — pg_basebackup](https://www.postgresql.org/docs/current/app-pgbasebackup.html?utm_source=chatgpt.com)
[PostgreSQL 18 — pg_createsubscriber](https://www.postgresql.org/docs/current/app-pgcreatesubscriber.html?utm_source=chatgpt.com)
[PostgreSQL 18 — pg_verifybackup](https://www.postgresql.org/docs/current/app-pgverifybackup.html?utm_source=chatgpt.com)

[1]: https://www.postgresql.org/docs/current/release-18.html?utm_source=chatgpt.com "Documentation: 18: E.5. Release 18"
[2]: https://www.postgresql.org/docs/current/logical-replication.html?utm_source=chatgpt.com "Documentation: 18: Chapter 29. Logical Replication"
[3]: https://www.postgresql.org/docs/current/backup-dump.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 25.1. SQL Dump"
[4]: https://www.postgresql.org/docs/current/app-pgdump.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: pg_dump"
[5]: https://www.postgresql.org/docs/current/app-pg-dumpall.html?utm_source=chatgpt.com "Documentation: 18: pg_dumpall"
[6]: https://www.postgresql.org/docs/current/backup.html?utm_source=chatgpt.com "Documentation: 18: Chapter 25. Backup and Restore"
[7]: https://www.postgresql.org/docs/current/app-pgbasebackup.html?utm_source=chatgpt.com "Documentation: 18: pg_basebackup"
[8]: https://www.postgresql.org/docs/current/upgrading.html?utm_source=chatgpt.com "Documentation: 18: 18.6. Upgrading a PostgreSQL Cluster"
[9]: https://www.postgresql.org/docs/current/logical-replication-publication.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 29.1. Publication"
[10]: https://www.postgresql.org/docs/current/logicaldecoding-example.html?utm_source=chatgpt.com "Documentation: 18: 47.1. Logical Decoding Examples"
[11]: https://www.postgresql.org/docs/current/logicaldecoding-explanation.html?utm_source=chatgpt.com "Documentation: 18: 47.2. Logical Decoding Concepts"
[12]: https://www.postgresql.org/docs/current/logicaldecoding.html?utm_source=chatgpt.com "Documentation: 18: Chapter 47. Logical Decoding"
[13]: https://www.postgresql.org/docs/current/app-pgrecvlogical.html?utm_source=chatgpt.com "Documentation: 18: pg_recvlogical"
[14]: https://www.postgresql.org/docs/current/app-pgcreatesubscriber.html?utm_source=chatgpt.com "Documentation: 18: pg_createsubscriber"
[15]: https://www.postgresql.org/docs/current/app-pgverifybackup.html?utm_source=chatgpt.com "Documentation: 18: pg_verifybackup"
[16]: https://www.postgresql.org/docs/current/pgstatstatements.html?utm_source=chatgpt.com "18: F.32. pg_stat_statements — track statistics of SQL planning and ..."
[17]: https://www.postgresql.org/docs/current/monitoring-stats.html?utm_source=chatgpt.com "Documentation: 18: 27.2. The Cumulative Statistics System"
