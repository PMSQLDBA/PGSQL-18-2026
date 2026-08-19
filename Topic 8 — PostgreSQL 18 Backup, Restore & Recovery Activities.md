# Topic 8 — PostgreSQL 18 Backup, Restore & Recovery Activities

PostgreSQL 18 has three fundamental backup approaches:

1. **SQL dump**
2. **File-system-level backup**
3. **Continuous archiving with WAL for PITR**

PostgreSQL’s official documentation treats these as separate methods because they solve different recovery needs. ([PostgreSQL][1])

A production backup strategy normally combines more than one of them.

---

## 1. High-Level Backup Architecture

A useful way to think about PostgreSQL backup is:

```text
                     PostgreSQL 18
                          │
             ┌────────────┼─────────────┐
             │            │             │
             ▼            ▼             ▼
        Logical       Physical       WAL Archive
         Backup        Backup
             │            │             │
             ▼            ▼             ▼
        pg_dump       Base Backup    Continuous WAL
        pg_dumpall    pg_basebackup
             │            │             │
             └────────────┼─────────────┘
                          ▼
                    Backup Storage
                          │
                          ▼
                       Restore
                          │
                ┌─────────┴─────────┐
                ▼                   ▼
          Full Recovery             PITR
                              Point-in-Time Recovery
```

The important architectural question is not:

> "Do we have backups?"

It is:

> "Which failures can we recover from, to what point in time, and how long will recovery take?"

---

# 2. Backup vs Restore vs Recovery

These terms should be separated.

### Backup

Creating a recoverable copy.

```text
PostgreSQL
    │
    ▼
Backup Repository
```

### Restore

Copying the saved backup back into a target environment.

```text
Backup
   │
   ▼
Target PostgreSQL
```

### Recovery

Bringing the restored database to the required consistent state, often by replaying WAL.

```text
Base Backup
     +
Archived WAL
     │
     ▼
Recovery
     │
     ▼
Required database state
```

Restore and recovery are often used together, but technically they are different steps.

---

# 3. Recovery Requirements: RPO and RTO

Before designing backups, define:

```text
RPO = Recovery Point Objective
RTO = Recovery Time Objective
```

Example:

```text
RPO = 5 minutes
RTO = 30 minutes
```

means:

```text
Lose no more than ~5 minutes of data
and
restore service within ~30 minutes
```

Backup frequency alone does not determine RPO if continuous WAL archiving is available.

For example:

```text
Full backup Sunday
       +
Continuous WAL archive
       =
Potential recovery to Tuesday 14:37
```

provided the required backup and WAL chain remains available. PostgreSQL documents continuous archiving specifically as the mechanism that allows recovery to a chosen point after the base backup. ([PostgreSQL][2])

---

# 4. Logical Backup

Logical backups reconstruct database objects and data through SQL-level representations rather than copying PostgreSQL's physical storage.

The main tools are:

```text
pg_dump
pg_dumpall
pg_restore
psql
```

Architecture:

```text
PostgreSQL Database
       │
       ▼
    pg_dump
       │
       ▼
Logical Backup
       │
       ▼
Target PostgreSQL
```

PostgreSQL states that `pg_dump` generates a representation that can recreate the database state at the time the dump was taken. ([PostgreSQL][3])

---

# 5. `pg_dump`

`pg_dump` backs up **one PostgreSQL database**.

Basic example:

```bash
pg_dump appdb > appdb.sql
```

This creates a plain SQL dump.

Restore:

```bash
psql -X appdb < appdb.sql
```

PostgreSQL documents this as the basic SQL dump and restore workflow. ([PostgreSQL][3])

---

# 6. Important `pg_dump` Property

`pg_dump` creates an internally consistent snapshot.

That means:

```text
Dump starts at 10:00
      │
      ▼
Applications continue changing data
      │
      ▼
Dump completes at 10:45
```

The dump represents a consistent database snapshot rather than an arbitrary mixture of unrelated points during the dump. PostgreSQL also notes that normal database activity can continue while `pg_dump` is running, although operations requiring conflicting exclusive locks can interfere. ([PostgreSQL][3])

---

# 7. `pg_dump` Is Database-Level

Important limitation:

```text
pg_dump
   │
   ▼
ONE database
```

Suppose the cluster has:

```text
postgres
appdb
finance
reporting
```

Then:

```bash
pg_dump appdb
```

backs up only:

```text
appdb
```

It does not automatically back up cluster-wide roles and tablespace definitions. PostgreSQL specifically documents this distinction. ([PostgreSQL][3])

---

# 8. Common `pg_dump` Formats

PostgreSQL supports multiple dump formats.

### Plain

```bash
pg_dump appdb > appdb.sql
```

Restore using:

```bash
psql
```

### Custom

```bash
pg_dump -Fc appdb -f appdb.dump
```

Restore using:

```bash
pg_restore -d appdb appdb.dump
```

### Directory

```bash
pg_dump -Fd appdb -f appdb_dump
```

Directory format supports parallel dump operations. PostgreSQL documents parallel dump using directory format with `-j`. ([PostgreSQL][3])

---

# 9. Why Custom Format Is Useful

For many DBA workflows:

```text
Custom format
     │
     ├── Compression
     ├── Object selection
     ├── Flexible restore
     └── Parallel restore
```

Example:

```bash
pg_dump -Fc -d appdb -f appdb.dump
```

Then:

```bash
pg_restore -d appdb appdb.dump
```

Custom format is not plain SQL and therefore should be restored using `pg_restore`, not simply fed into `psql`. ([PostgreSQL][3])

---

# 10. Directory Format and Parallel Backup

For very large databases:

```bash
pg_dump \
  -Fd \
  -j 8 \
  -f /backup/appdb \
  appdb
```

Conceptually:

```text
Database
   │
   ├── Worker 1 → Table A
   ├── Worker 2 → Table B
   ├── Worker 3 → Table C
   └── Worker 4 → Table D
```

PostgreSQL documents parallel dump as supported for the directory archive format. ([PostgreSQL][3])

---

# 11. Parallel Restore

`pg_restore` can also restore in parallel.

Example:

```bash
pg_restore \
  -j 8 \
  -d appdb \
  /backup/appdb.dump
```

Parallel restore can be used for custom or directory archive formats. ([PostgreSQL][3])

This can significantly reduce recovery time for large databases when CPU, storage, WAL, and target-system capacity permit it.

---

# 12. Selective Backup

One advantage of `pg_dump` is selective backup.

Schema:

```bash
pg_dump -n finance appdb > finance.sql
```

Table:

```bash
pg_dump -t finance.transactions appdb > transactions.sql
```

PostgreSQL notes that users without privileges to dump an entire database can still dump specific schemas or tables to which they have access. ([PostgreSQL][3])

---

# 13. Selective Restore

With an archive-format dump, you can restore selected objects.

Conceptually:

```text
Backup
 │
 ├── customers
 ├── orders
 ├── payments
 └── products

Restore only
      │
      ▼
    orders
```

This is one reason logical backups are useful for object-level recovery.

Physical backups generally operate at the whole-cluster level instead.

---

# 14. Schema-Only Backup

Useful for migration or environment creation:

```bash
pg_dump --schema-only appdb > schema.sql
```

This captures object definitions without table data.

Typical use:

```text
Production Schema
      │
      ▼
Schema-only dump
      │
      ▼
DEV / TEST database
```

---

# 15. Data-Only Backup

Example:

```bash
pg_dump --data-only appdb > data.sql
```

This is useful where schemas are managed separately.

Be careful with:

```text
Sequences
constraints
ownership
dependencies
```

when separating schema and data operations.

---

# 16. `pg_dumpall`

`pg_dumpall` is designed for cluster-wide logical dumping.

Example:

```bash
pg_dumpall > cluster.sql
```

It covers databases plus cluster-wide information such as:

```text
Roles
Tablespace definitions
```

PostgreSQL explains that `pg_dumpall` works by creating cluster-wide commands and invoking `pg_dump` for individual databases. ([PostgreSQL][3])

---

# 17. Global Objects Backup

A very useful DBA command:

```bash
pg_dumpall --globals-only > globals.sql
```

This captures cluster-global objects such as:

```text
Roles
Role memberships
Tablespace definitions
```

This is important if your main backups are individual `pg_dump` backups.

PostgreSQL specifically recommends `--globals-only` where individual database dumps are used and cluster-wide data must also be protected. ([PostgreSQL][3])

---

# 18. Logical Restore Prerequisites

When restoring a plain SQL dump:

```bash
psql -X appdb < appdb.sql
```

the target database usually needs to exist first.

PostgreSQL recommends creating an empty database from `template0` for this type of restore:

```bash
createdb -T template0 appdb
```

then:

```bash
psql -X appdb < appdb.sql
```

([PostgreSQL][3])

---

# 19. Restore Error Handling

By default, `psql` can continue after an SQL error.

For stricter restores:

```bash
psql \
  -X \
  --set ON_ERROR_STOP=on \
  appdb < appdb.sql
```

This causes the restore to stop when an SQL error occurs.

PostgreSQL explicitly documents this approach. ([PostgreSQL][3])

---

# 20. Single-Transaction Restore

For suitable logical restores:

```bash
psql \
  -X \
  --single-transaction \
  appdb < appdb.sql
```

Conceptually:

```text
Restore
   │
   ├── Everything succeeds
   │       ▼
   │     COMMIT
   │
   └── Error
           ▼
        ROLLBACK
```

This avoids leaving a partially restored database, but PostgreSQL cautions that one late failure could roll back hours of restore work. ([PostgreSQL][3])

---

# 21. Post-Restore `ANALYZE`

After restoring a logical dump, PostgreSQL recommends running:

```sql
ANALYZE;
```

so the optimizer has useful statistics. ([PostgreSQL][3])

Conceptually:

```text
Restored Tables
      │
      ▼
   ANALYZE
      │
      ▼
Planner Statistics
      │
      ▼
Better Query Plans
```

---

# 22. Logical Backup Advantages

Logical backups are excellent for:

```text
Individual database backup
Individual schema/table backup
Selective restores
Migration between PostgreSQL versions
Architecture/platform migration
DEV/TEST refreshes
Object-level recovery
```

PostgreSQL notes that `pg_dump` output can generally be reloaded into newer PostgreSQL versions and is much less tied to machine architecture than physical backups. ([PostgreSQL][3])

---

# 23. Logical Backup Limitations

Logical backups are less ideal for:

```text
Very large databases needing very low RTO
Cluster-wide crash recovery
PITR
Standby creation
Whole-cluster physical recovery
```

Most importantly:

```text
pg_dump + WAL archive
≠
PITR backup
```

PostgreSQL explicitly states that `pg_dump` and `pg_dumpall` cannot be used as the base for WAL-replay continuous-archive recovery. ([PostgreSQL][2])

---

# 24. Physical Backup

Physical backup protects the PostgreSQL database cluster at the storage-file level.

Conceptually:

```text
             PostgreSQL Cluster
                    │
                    ▼
               Physical Copy
                    │
                    ▼
              Backup Repository
```

The primary PostgreSQL-native tool is:

```text
pg_basebackup
```

---

# 25. `pg_basebackup`

`pg_basebackup` takes a base backup of a **running PostgreSQL cluster** without requiring normal client workloads to stop. It backs up the entire cluster and cannot directly select just one database or table. ([PostgreSQL][4])

Basic pattern:

```bash
pg_basebackup \
  -h dbserver \
  -U backup_user \
  -D /backup/base
```

---

# 26. `pg_basebackup` Architecture

```text
                    PostgreSQL
                        │
                 Replication Protocol
                        │
                        ▼
                  pg_basebackup
                        │
                 ┌──────┴──────┐
                 ▼             ▼
              PGDATA         WAL
                 │
                 ▼
             Base Backup
```

`pg_basebackup` uses a PostgreSQL replication connection. The account needs the required replication privileges, or superuser capability, and `pg_hba.conf` must permit the connection. ([PostgreSQL][4])

---

# 27. Full Base Backup

A full base backup makes a complete physical copy of the cluster files.

Example:

```bash
pg_basebackup \
  -h primary \
  -U backup_user \
  -D /backup/full \
  -Fp \
  -P
```

A full physical backup can be used for:

```text
PITR
Standby initialization
Full-cluster recovery
```

PostgreSQL explicitly documents both PITR and standby creation as uses for `pg_basebackup`. ([PostgreSQL][4])

---

# 28. PostgreSQL 18 Incremental Physical Backup

PostgreSQL 18 supports incremental base backups through `pg_basebackup`. ([PostgreSQL][4])

Conceptually:

```text
Day 1
Full Backup
    │
    ▼
Day 2
Incremental
    │
    ▼
Day 3
Incremental
```

The incremental backup stores modified blocks relative to a previous backup rather than simply duplicating every file again. ([PostgreSQL][4])

This is an important PostgreSQL-native capability for large clusters.

---

# 29. Incremental Backup Dependency

An incremental backup cannot be restored directly by itself.

PostgreSQL requires the dependent backup chain to be combined first using:

```text
pg_combinebackup
```

([PostgreSQL][4])

Architecture:

```text
Full Backup
     +
Incremental 1
     +
Incremental 2
     │
     ▼
pg_combinebackup
     │
     ▼
Synthetic/combined full backup
     │
     ▼
Recovery
```

---

# 30. `pg_combinebackup`

Example conceptual command:

```bash
pg_combinebackup \
  /backup/full \
  /backup/incremental1 \
  /backup/incremental2 \
  -o /backup/combined
```

The output becomes a usable complete backup set for restoration.

This tool is specifically designed to reconstruct a full backup from a full base plus incremental backups. ([PostgreSQL][5])

---

# 31. Backup from Standby

`pg_basebackup` can also take a backup from a standby under documented conditions. ([PostgreSQL][4])

Architecture:

```text
Primary
   │
   │ WAL
   ▼
Standby
   │
   ▼
pg_basebackup
   │
   ▼
Backup
```

This can reduce read pressure on the production primary.

However, PostgreSQL documents important limitations, including:

```text
If standby is promoted during backup → backup fails

full_page_writes must be enabled on primary

Certain WAL-switch behavior differs from primary backup
```

([PostgreSQL][4])

---

# 32. Monitoring Base Backup Progress

PostgreSQL provides:

```sql
SELECT *
FROM pg_stat_progress_basebackup;
```

while a base backup is running. PostgreSQL 18 includes `BASE_BACKUP` among the operations that expose progress reporting. ([PostgreSQL][6])

This is useful for:

```text
Estimated progress
Backup monitoring
Operations dashboards
Long-running backup troubleshooting
```

---

# 33. Continuous Archiving

Continuous archiving combines:

```text
Base Backup
    +
Continuous WAL Archive
```

This enables PITR.

Architecture:

```text
              PostgreSQL
                  │
         ┌────────┴────────┐
         ▼                 ▼
    Base Backup        WAL Archive
         │                 │
         └────────┬────────┘
                  ▼
                PITR
```

PostgreSQL explains that this method works because WAL contains changes made after the base backup. ([PostgreSQL][2])

---

# 34. Why WAL Matters

PostgreSQL continually writes WAL records describing changes.

```text
INSERT
UPDATE
DELETE
DDL
     │
     ▼
    WAL
```

During recovery:

```text
Restore Base Backup
       │
       ▼
Replay Archived WAL
       │
       ▼
Advance database state
```

PostgreSQL states that WAL exists primarily for crash safety but enables continuous archiving and PITR when retained externally. ([PostgreSQL][2])

---

# 35. Enabling WAL Archiving

Core settings include:

```conf
wal_level = replica
archive_mode = on
```

and either:

```conf
archive_command = '...'
```

or:

```conf
archive_library = '...'
```

PostgreSQL 18 requires `wal_level` of `replica` or higher for continuous archiving. ([PostgreSQL][2])

---

# 36. `archive_command`

A simplified example from PostgreSQL documentation is conceptually:

```conf
archive_command =
'test ! -f /archive/%f && cp %p /archive/%f'
```

where:

```text
%p = source WAL path
%f = WAL filename
```

The exact production implementation should normally use a robust backup/archive system rather than a simplistic copy command.

PostgreSQL explicitly labels its simple command as an example rather than a universal recommendation. ([PostgreSQL][2])

---

# 37. Archive Success Is Critical

PostgreSQL treats an archive command exit code of:

```text
0
```

as success.

If archiving repeatedly fails:

```text
WAL generated
     │
     ▼
Archive fails
     │
     ▼
WAL remains in pg_wal
     │
     ▼
pg_wal grows
     │
     ▼
Filesystem can fill
```

PostgreSQL warns that if the filesystem containing `pg_wal` fills, PostgreSQL can PANIC and remain unavailable until space is recovered. ([PostgreSQL][2])

Therefore archive failures need urgent alerts.

---

# 38. Monitor WAL Archiving

Use:

```sql
SELECT *
FROM pg_stat_archiver;
```

Important items include:

```text
Archived count
Failed count
Last archived WAL
Last archive time
Last failed WAL
Last failure time
```

Monitoring should answer:

```text
Is WAL being archived?
How far behind?
Any failures?
Is archive storage reachable?
```

---

# 39. PITR — Point-in-Time Recovery

PITR allows you to restore to a point before an unwanted event.

Example:

```text
10:00 Good
10:15 Good
10:27 Accidental DROP TABLE
10:30 Problem discovered
```

Desired recovery:

```text
10:26:59
```

Process:

```text
Base Backup
      │
      ▼
Restore
      │
      ▼
Replay WAL
      │
      ▼
Stop before 10:27
```

PostgreSQL specifically states that archived WAL does not need to be replayed all the way to the end; recovery can stop at an intermediate point. ([PostgreSQL][2])

---

# 40. PITR Recovery Targets

Recovery can be designed around targets such as:

```text
Timestamp
Transaction ID
Named restore point
LSN
Timeline
```

The exact configuration depends on the recovery scenario.

A timestamp is conceptually:

```conf
recovery_target_time = '2026-08-19 10:26:59'
```

and WAL is retrieved using recovery configuration such as:

```conf
restore_command = '...'
```

The recovery target should always be validated carefully to avoid replaying the unwanted transaction.

---

# 41. Recovery Signal

Modern PostgreSQL uses:

```text
recovery.signal
```

to request archive recovery.

Conceptually:

```text
Restored PGDATA
     │
     ├── recovery.signal
     └── restore/recovery configuration
              │
              ▼
        PostgreSQL starts
              │
              ▼
           WAL replay
```

For standby operation, the related marker is:

```text
standby.signal
```

---

# 42. Named Restore Points

Before a risky deployment, you can create a logical marker in WAL:

```sql
SELECT pg_create_restore_point('before_release');
```

Then, if the recovery architecture and retained WAL support it, the restore point can be used as a recovery target.

This is useful around:

```text
Application deployments
Large data loads
Risky batch jobs
Major schema changes
```

But a restore point itself is **not a backup**.

You still need a valid base backup and the required WAL sequence.

---

# 43. Timeline Architecture

When PostgreSQL recovers and is promoted, it can create a new WAL timeline.

Conceptually:

```text
Timeline 1
     │
     │ Recovery
     ▼
Target Point
     │
     ├──────── old history
     │
     └──────── Timeline 2
```

Timelines allow PostgreSQL to distinguish separate recovery histories. PostgreSQL covers timelines as part of continuous archive recovery. ([PostgreSQL][2])

---

# 44. Physical Recovery Is Cluster-Level

Continuous archive recovery restores:

```text
Entire PostgreSQL cluster
```

not:

```text
one table only
```

PostgreSQL explicitly notes that continuous archiving supports whole-cluster restoration, not selected subsets. ([PostgreSQL][2])

So if one table needs historical recovery:

```text
PITR into temporary PostgreSQL cluster
          │
          ▼
Extract required table
          │
          ▼
Import into production
```

may be a practical method.

---

# 45. Example Table Recovery Through Temporary PITR

Suppose:

```text
customers table deleted
```

but restoring the entire production system backward would lose newer transactions.

A safer workflow can be:

```text
Backup + WAL
      │
      ▼
Temporary PostgreSQL
      │
      ▼
PITR before deletion
      │
      ▼
pg_dump customers
      │
      ▼
Restore customers into production
```

This combines physical recovery with logical object extraction.

---

# 46. File-System-Level Backup

PostgreSQL also documents file-system-level backup as one of the three fundamental methods. ([PostgreSQL][1])

But simply doing:

```bash
cp -r PGDATA /backup/
```

while PostgreSQL is running is not automatically a valid backup.

Physical consistency must be guaranteed through appropriate PostgreSQL backup mechanisms, stopped-server copying, or correctly integrated snapshot/continuous archive procedures.

For most enterprise environments, use tested PostgreSQL-aware tooling rather than ad-hoc file copies.

---

# 47. Backup Configuration Files Separately

WAL archive recovery protects PostgreSQL data changes.

It does **not** capture manual changes made to files such as:

```text
postgresql.conf
pg_hba.conf
pg_ident.conf
```

PostgreSQL explicitly calls this out and recommends backing up configuration files separately when needed. ([PostgreSQL][2])

Therefore:

```text
Database Backup
       +
WAL
       +
Configuration Backup
       =
Better complete recovery coverage
```

---

# 48. Backup Encryption

Backup repositories can contain essentially the entire database.

Therefore protect:

```text
Backup files
Archived WAL
Credentials
Backup metadata
```

using appropriate:

```text
Encryption at rest
Encryption in transit
Filesystem permissions
Access controls
Secrets management
```

PostgreSQL specifically warns that archived WAL effectively contains all database changes and should be protected from unauthorized access. ([PostgreSQL][2])

---

# 49. Backup Retention

A good retention policy should answer:

```text
How many daily backups?
How many weekly backups?
How many monthly backups?
How much WAL?
How long must PITR remain possible?
How long for compliance?
```

Example architecture:

```text
Daily backups   → 14 days
Weekly backups  → 8 weeks
Monthly backups → 12 months
```

This is only an example.

Retention must be based on:

```text
Business requirements
RPO/RTO
Compliance
Storage cost
Database size
Recovery scenarios
```

---

# 50. WAL Retention Dependency

Be careful deleting old WAL.

Recovery requires:

```text
Selected Base Backup
        +
Continuous required WAL
```

Example:

```text
Base Backup
    │
    ├── WAL 1
    ├── WAL 2
    ├── WAL 3
    └── WAL 4
```

If WAL 2 is missing:

```text
Recovery chain broken
```

Therefore retention tooling must understand dependencies.

---

# 51. Backup Validation

A backup job finishing with:

```text
SUCCESS
```

does not prove:

```text
RESTORE WORKS
```

A proper validation process is:

```text
Backup generated
     │
     ▼
Backup integrity checked
     │
     ▼
Restore to isolated system
     │
     ▼
PostgreSQL starts
     │
     ▼
Application/data validation
     │
     ▼
Recovery confirmed
```

Restore testing is one of the most important DBA backup activities.

---

# 52. Restore Testing Levels

### Level 1

Verify file exists.

```text
Weak validation
```

### Level 2

Verify backup metadata/checksum.

```text
Better
```

### Level 3

Restore PostgreSQL and start database.

```text
Strong
```

### Level 4

Restore + run application/business validation.

```text
Best practical validation
```

For critical systems, schedule automated Level 3/4 testing.

---

# 53. Recovery Drill

A DR test should measure:

```text
Backup discovery time
Restore time
WAL replay time
Startup time
Validation time
Application reconnection time
```

For example:

```text
Backup restore      35 min
WAL replay          15 min
Validation           5 min
Application restart  5 min
--------------------------
Actual RTO          60 min
```

That gives a real RTO instead of a theoretical number.

---

# 54. Backup Monitoring

Monitor:

```text
Backup success/failure
Backup age
Backup duration
Backup size
Incremental size
WAL archive rate
Archive failures
Repository capacity
Backup retention
Backup checksum/integrity
Restore-test age
```

A strong alert is:

```text
No successful recovery test
within defined period
```

not merely:

```text
backup job failed
```

---

# 55. Logical vs Physical Backup

| Requirement                | Logical             | Physical                    |
| -------------------------- | ------------------- | --------------------------- |
| One table                  | Excellent           | No direct selective restore |
| One database               | Excellent           | Whole cluster               |
| Cross-version migration    | Good                | Version-sensitive           |
| Full-cluster recovery      | Slower for large DB | Excellent                   |
| PITR                       | No                  | Yes with WAL                |
| Standby initialization     | No                  | Yes                         |
| Selective restore          | Excellent           | Limited                     |
| Very low RTO for huge DB   | Usually weaker      | Better                      |
| Human-error point recovery | Limited             | Excellent with PITR         |

Most mature environments use both.

---

# 56. Recommended Enterprise Backup Strategy

For a critical PostgreSQL cluster:

```text
                PostgreSQL 18
                      │
          ┌───────────┼───────────┐
          │           │           │
          ▼           ▼           ▼
     Physical       WAL        Logical
      Backup       Archive      Backup
          │           │           │
          └───────────┼───────────┘
                      ▼
              Backup Repository
                      │
             ┌────────┴────────┐
             ▼                 ▼
         Local Copy        Off-site Copy
             │                 │
             └────────┬────────┘
                      ▼
                Restore Tests
```

A practical combination could be:

```text
Physical base/incremental backups
+
Continuous WAL archive
+
Periodic logical dumps for selected objects
+
Off-site backup
+
Automated restore testing
```

---

# 57. Backup Is Not Replication

Suppose:

```sql
DROP DATABASE appdb;
```

Replication will normally reproduce that change:

```text
Primary
   │
   │ WAL
   ▼
Standby
   │
   ▼
Database dropped
```

Backup/PITR gives you the historical recovery path.

Therefore:

```text
Replication
    =
Availability

Backup
    =
Recoverability
```

You need both.

---

# 58. Backup Is Not HA

If the production primary crashes:

```text
Restore from backup
      │
      ▼
Potentially many minutes/hours
```

A standby may recover service much faster.

Therefore:

```text
HA → reduce downtime

Backup → recover data/history

DR → recover from major failure
```

These concepts overlap but are not interchangeable.

---

# 59. Example Production Backup Schedule

One example—not a universal standard:

```text
Weekly
  └── Full physical backup

Daily
  └── Incremental physical backup

Continuous
  └── WAL archiving

Weekly/Monthly
  └── Selected logical dumps

Daily
  └── Backup validation

Monthly
  └── Full restore/PITR test
```

Actual frequency should be driven by RPO, RTO, workload size, and retention requirements.

---

# 60. Recovery Decision Tree

When something goes wrong:

```text
What was lost?
     │
     ├── One table?
     │      │
     │      ├── Logical backup available
     │      │        ▼
     │      │     Restore table
     │      │
     │      └── PITR temp cluster
     │               ▼
     │          Extract table
     │
     ├── One database?
     │      │
     │      └── pg_dump/restore
     │
     ├── Entire cluster?
     │      │
     │      └── Physical restore + WAL
     │
     ├── Need state at specific time?
     │      │
     │      └── PITR
     │
     └── Primary infrastructure failed?
            │
            ├── HA standby
            └── Backup if HA unavailable
```

---

# 61. DBA Backup Checklist

| Area                | Validation             |
| ------------------- | ---------------------- |
| RPO                 | Defined                |
| RTO                 | Defined                |
| Logical backups     | Configured if required |
| Physical backups    | Configured             |
| WAL archive         | Healthy                |
| Incremental chain   | Valid                  |
| Global roles        | Protected              |
| Tablespaces         | Included               |
| Configuration files | Protected              |
| Repository          | Redundant              |
| Off-site copy       | Available              |
| Encryption          | Enabled where required |
| Credentials         | Protected              |
| Retention           | Defined                |
| Capacity            | Monitored              |
| Backup failures     | Alerted                |
| WAL failures        | Alerted                |
| Restore test        | Successful             |
| PITR test           | Successful             |
| Documentation       | Current                |
| Recovery runbook    | Tested                 |

---

# 62. Important Rules to Remember

```text
A backup is useful only if it can be restored.

Replication is not backup.

pg_dump is logical backup.

pg_dump backs up one database.

pg_dumpall handles cluster-wide logical information.

pg_restore restores archive-format logical backups.

pg_basebackup is physical and cluster-wide.

PostgreSQL 18 supports incremental base backups.

Incremental backups need pg_combinebackup before use.

PITR requires a base backup + continuous WAL.

Missing WAL can break recovery.

Configuration files may need separate protection.

WAL archive failure can eventually fill pg_wal.

Always test restores.
```

---

## PostgreSQL 18 Official References

The primary PostgreSQL 18 reference is **Chapter 25 — Backup and Restore**. PostgreSQL officially identifies SQL dumps, filesystem-level backups, and continuous archiving as the three fundamental backup approaches. ([PostgreSQL][1])

For logical backup and restore, use **25.1 SQL Dump** and the `pg_dump` / `pg_restore` reference pages. PostgreSQL documents snapshot consistency, custom/directory formats, parallel dump and restore, `pg_dumpall`, global objects, and post-restore `ANALYZE`. ([PostgreSQL][3])

For physical backup, use the PostgreSQL 18 **`pg_basebackup`** documentation. It supports full and incremental backups of a running cluster, backups from primary or standby under documented conditions, and use for PITR or standby creation. ([PostgreSQL][4])

For PostgreSQL 18 incremental recovery, use **`pg_combinebackup`**, which combines a full backup with dependent incremental backups into a usable full backup. ([PostgreSQL][5])

For PITR and WAL archiving, use **25.3 Continuous Archiving and Point-in-Time Recovery**. It covers WAL archiving, base/incremental backups, recovery targets, timelines, archive requirements, and recovery procedures. ([PostgreSQL][2])

**Tracker: 8 / 29 completed — 21 pending.**

**Next: Topic 9 — PostgreSQL Disaster Recovery**, covering DR-site architecture, local HA vs remote DR, RPO/RTO tiers, PITR-based DR, streaming-standby DR, cross-region replication, DR activation, failback, DR drills, and enterprise recovery runbooks.

[1]: https://www.postgresql.org/docs/18/backup.html "PostgreSQL: Documentation: 18: Chapter 25. Backup and Restore"
[2]: https://www.postgresql.org/docs/18/continuous-archiving.html "PostgreSQL: Documentation: 18: 25.3. Continuous Archiving and Point-in-Time Recovery (PITR)"
[3]: https://www.postgresql.org/docs/18/backup-dump.html "PostgreSQL: Documentation: 18: 25.1. SQL Dump"
[4]: https://www.postgresql.org/docs/current/app-pgbasebackup.html?utm_source=chatgpt.com "Documentation: 18: pg_basebackup"
[5]: https://www.postgresql.org/docs/18/app-pgcombinebackup.html "PostgreSQL: Documentation: 18: pg_combinebackup"
[6]: https://www.postgresql.org/docs/current/progress-reporting.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 27.4. Progress Reporting"
