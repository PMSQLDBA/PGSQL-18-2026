Below is a production-oriented SOP tailored to the environment described below: 
=============================================
Ubuntu server: CPU 24 

RAM 64 GB 

storage type: HDD with 8 TB 

number of databases 10 

Databases Number of applications: 5

============================================

Treating **PostgreSQL 18 as the reference release** and the configuration values as controlled starting points that must be validated under workload testing. 

PostgreSQL’s current guidance, for example, describes about 25% of system RAM as a reasonable starting point for `shared_buffers` on a dedicated DB server, while also emphasizing the OS cache. ([PostgreSQL][1])

# Production PostgreSQL Deployment, Configuration & Operations SOP

**Document Type:** Technical Standard Operating Procedure
**Platform:** Ubuntu Server 26.x
**Database:** PostgreSQL 18.x
**Environment:** Production
**Server Capacity:** 24 CPU / 64 GB RAM / 8 TB HDD
**Applications:** 5
**Databases:** 10
**Recommended PostgreSQL Instances:** 2
**Document Owner:** Database / Infrastructure Team
**Review Frequency:** Every 6 months and after major infrastructure or PostgreSQL changes

---

# 1. Purpose

This SOP defines the standard architecture, installation, configuration, security, monitoring, backup, recovery, maintenance, and operational procedures for running PostgreSQL in production.

The objectives are:

* High database availability
* Predictable performance
* Controlled resource consumption
* Strong security
* Reliable backup and recovery
* Minimal application impact during maintenance
* Proper database isolation
* Capacity management
* Auditable operational procedures
* Prevention of disk, memory, connection, and transaction-related incidents

---

# 2. Server Specification

| Component            | Specification                       |
| -------------------- | ----------------------------------- |
| Operating System     | Ubuntu Server 26.x                  |
| CPU                  | 24 cores                            |
| RAM                  | 64 GB                               |
| Storage              | 8 TB HDD                            |
| Databases            | 10                                  |
| Applications         | 5                                   |
| PostgreSQL           | 18.x                                |
| PostgreSQL Instances | 2                                   |
| Connection Pooler    | PgBouncer                           |
| Primary workload     | Production                          |
| Monitoring           | Required                            |
| Backup               | Full + WAL/PITR                     |
| HA                   | Recommended for critical production |

---

# 3. Architecture Standard

The recommended starting architecture is:

```text
                       Applications
                            |
                 +----------+----------+
                 |                     |
             PgBouncer             PgBouncer
               :6432                 :6433
                 |                     |
                 v                     v
        +----------------+    +----------------+
        | PostgreSQL     |    | PostgreSQL     |
        | Instance 01    |    | Instance 02    |
        | Port 5432      |    | Port 5433      |
        +-------+--------+    +-------+--------+
                |                     |
          DB01 - DB06            DB07 - DB10
                |                     |
                +----------+----------+
                           |
                     Linux Storage
                           |
                        8 TB HDD
```

Do not create one PostgreSQL instance for every database unless there is a documented isolation requirement.

Ten databases do **not** require ten PostgreSQL instances.

---

# 4. Instance Allocation

## Instance 01 — Critical Production

**Purpose:** High-priority transactional applications

```text
Port: 5432
Databases: approximately 6
Priority: High
```

Example:

```text
APP01
 ├── DB01
 └── DB02

APP02
 ├── DB03
 └── DB04

APP03
 ├── DB05
 └── DB06
```

## Instance 02 — Standard Production

```text
Port: 5433
Databases: approximately 4
Priority: Medium
```

Example:

```text
APP04
 ├── DB07
 └── DB08

APP05
 ├── DB09
 └── DB10
```

The final database placement must be based on:

* Database size
* Transaction volume
* Peak connections
* Query workload
* Business criticality
* Batch processing
* Reporting workload
* Maintenance windows

Do not distribute databases purely by database count.

---

# 5. Storage Architecture

The existing HDD is the largest performance risk in this architecture.

PostgreSQL generates several types of storage activity simultaneously:

```text
Database workload
      |
      +---- Data reads
      |
      +---- Data writes
      |
      +---- Index reads/writes
      |
      +---- WAL writes
      |
      +---- Checkpoints
      |
      +---- Autovacuum
      |
      +---- Temporary files
      |
      +---- Backup activity
             |
             v
           HDD
```

Therefore, disk latency and IOPS must be actively monitored.

## Preferred Production Storage

Where budget permits:

```text
NVMe / Enterprise SSD
 |
 +-- PostgreSQL active data
 +-- indexes
 +-- pg_wal
 +-- temporary workload

HDD Storage
 |
 +-- backups
 +-- archive
 +-- historical data
 +-- cold data
```

For a write-intensive production system, SSD/NVMe should be considered a priority infrastructure improvement.

---

# 6. Filesystem Layout

Use clearly separated directories.

Example:

```text
/pgdata/
   |
   +-- instance01/
   |
   +-- instance02/

/pgwal/
   |
   +-- instance01/
   |
   +-- instance02/

/pgbackup/
   |
   +-- full/
   |
   +-- logical/
   |
   +-- wal_archive/

/pglog/
   |
   +-- instance01/
   |
   +-- instance02/
```

Where physically separate storage is available, WAL, database data and backups should be separated.

Never store the only backup copy on the same physical storage as the production database.

---

# 7. Operating System Preparation

Before installation:

```bash
sudo apt update
sudo apt upgrade
```

Verify:

```bash
uname -a
lsb_release -a
lscpu
free -h
lsblk
df -h
```

Record the output in the deployment/change record.

---

# 8. Time Synchronization

Database servers must have reliable time synchronization.

Verify:

```bash
timedatectl
```

Expected:

```text
System clock synchronized: yes
NTP service: active
```

Time drift can affect:

* Logs
* Monitoring
* Backup validation
* Audit records
* Distributed applications
* Troubleshooting

---

# 9. Swap

Verify:

```bash
free -h
swapon --show
```

Swap must not be treated as additional PostgreSQL memory.

Persistent PostgreSQL swapping is considered an operational warning and must be investigated.

Monitor:

```bash
vmstat 1
```

Significant sustained `si` / `so` activity requires investigation.

---

# 10. PostgreSQL Installation

Use an approved PostgreSQL repository/package source and pin the required major version according to organizational change-management policy.

After installation verify:

```bash
psql --version
```

Example:

```text
psql (PostgreSQL) 18.x
```

Do not allow uncontrolled major-version upgrades through routine operating-system patching.

---

# 11. PostgreSQL Cluster Creation

Create two independent clusters.

Conceptually:

```text
PostgreSQL 18
 |
 +-- instance01 :5432
 |
 +-- instance02 :5433
```

On Debian/Ubuntu installations using `postgresql-common`, cluster-management commands can be used to maintain independent clusters.

Verify running clusters with the platform's PostgreSQL cluster-management tooling and with:

```bash
ps -ef | grep postgres
ss -lntp
```

---

# 12. Ownership and Permissions

PostgreSQL data directories must be owned by the PostgreSQL service account.

Example:

```text
Owner: postgres
Group: postgres
```

Data directory permissions should prevent unauthorized users from reading database files.

Do not:

```bash
chmod -R 777 /pgdata
```

Do not run PostgreSQL as `root`.

---

# 13. Port Standard

Use:

```text
Instance01 = 5432
Instance02 = 5433

PgBouncer01 = 6432
PgBouncer02 = 6433
```

Document all ports in the infrastructure CMDB/firewall documentation.

---

# 14. Memory Strategy

Total RAM:

```text
64 GB
```

Do not allocate all memory directly to PostgreSQL.

Linux filesystem caching is important to PostgreSQL performance.

PostgreSQL documentation recommends approximately 25% of physical memory as a reasonable **starting point** for `shared_buffers` on a dedicated database server, rather than treating all available RAM as PostgreSQL shared memory. ([PostgreSQL][1])

For multiple instances, memory planning must be done at the **host level**, not independently for each instance.

Example planning envelope:

```text
Instance01 shared_buffers       10 GB
Instance02 shared_buffers        6 GB
-------------------------------------
PostgreSQL shared buffers       16 GB

Remaining memory                48 GB
```

The remaining memory supports:

* Linux filesystem cache
* PostgreSQL backend processes
* `work_mem`
* maintenance operations
* autovacuum
* PgBouncer
* monitoring
* OS processes
* operational safety margin

---

# 15. Instance 01 Starting Configuration

Example controlled baseline:

```ini
port = 5432

max_connections = 150

shared_buffers = 10GB
effective_cache_size = 40GB

work_mem = 16MB
maintenance_work_mem = 1GB
autovacuum_work_mem = 512MB

checkpoint_completion_target = 0.9

max_worker_processes = 16
max_parallel_workers = 12
max_parallel_workers_per_gather = 4

track_io_timing = on
```

These are starting values.

They must be validated against production workload measurements.

---

# 16. Instance 02 Starting Configuration

```ini
port = 5433

max_connections = 100

shared_buffers = 6GB
effective_cache_size = 40GB

work_mem = 16MB
maintenance_work_mem = 512MB
autovacuum_work_mem = 256MB

checkpoint_completion_target = 0.9

max_worker_processes = 8
max_parallel_workers = 6
max_parallel_workers_per_gather = 2

track_io_timing = on
```

---

# 17. Important Warning — work_mem

Never calculate memory as:

```text
work_mem × max_connections
```

and assume that represents the maximum.

A single query can execute multiple sorts, hashes, or other operations capable of consuming `work_mem`.

Parallel workers can increase this further.

Therefore:

```ini
work_mem = 16MB
```

should be considered a controlled starting point.

Large reporting queries requiring more memory should preferably use targeted session or role-level settings rather than increasing the global value.

Example:

```sql
SET work_mem = '128MB';
```

Only authorized jobs should do this.

---

# 18. effective_cache_size

`effective_cache_size` is a planner estimate.

It does **not** allocate that amount of RAM.

Example:

```ini
effective_cache_size = 40GB
```

It tells the PostgreSQL planner approximately how much cache may be available across PostgreSQL and the operating system.

---

# 19. Connection Management

Applications should not create unlimited direct PostgreSQL connections.

Preferred architecture:

```text
Application
     |
     | 100s of logical connections
     v
 PgBouncer
     |
     | controlled backend connections
     v
 PostgreSQL
```

PgBouncer is specifically designed to reuse PostgreSQL server connections and reduce connection-establishment overhead. ([PgBouncer][2])

---

# 20. PgBouncer

Recommended ports:

```text
Instance01 PostgreSQL = 5432
Instance01 PgBouncer  = 6432

Instance02 PostgreSQL = 5433
Instance02 PgBouncer  = 6433
```

Starting example:

```ini
[pgbouncer]

listen_addr = 0.0.0.0
listen_port = 6432

pool_mode = transaction

max_client_conn = 500
default_pool_size = 30
reserve_pool_size = 10
reserve_pool_timeout = 5

server_idle_timeout = 600
```

PgBouncer documents `default_pool_size` as the maximum number of server connections per user/database pair unless overridden. Therefore pool sizing must consider the number of databases **and users**, not merely `30 total connections`. ([PgBouncer][3])

---

# 21. PgBouncer Compatibility Check

Do not blindly enable transaction pooling.

Transaction pooling can affect application features that depend on session state, including certain:

* `SET` behaviors
* session advisory locks
* temporary-table patterns
* SQL-level prepared statements
* session-specific state

PgBouncer documents these compatibility differences between session and transaction pooling. ([PgBouncer][4])

Therefore:

```text
Application compatibility test
             |
          PASS?
        /       \
      Yes        No
       |          |
Transaction     Session
Pooling         Pooling
```

Application certification is mandatory before production deployment.

---

# 22. Authentication

Use SCRAM authentication for password-based connections where supported.

Avoid legacy weak authentication mechanisms.

Each application receives its own PostgreSQL login role.

Example:

```text
app01_user
app02_user
app03_user
app04_user
app05_user
```

Never share the PostgreSQL superuser with applications.

---

# 23. Role Model

Use a layered role model.

Example:

```text
LOGIN ROLE
     |
     v
APPLICATION ROLE
     |
     v
DATABASE/SCHEMA PRIVILEGES
```

Example concept:

```sql
CREATE ROLE app01_rw NOLOGIN;

CREATE ROLE app01_login
LOGIN
PASSWORD 'managed-outside-source-code';

GRANT app01_rw TO app01_login;
```

Credentials must be managed using the organization's secrets-management platform.

Do not hard-code database passwords in:

* application source code
* shell scripts
* Git repositories
* deployment manifests containing plain-text secrets

---

# 24. Least Privilege

Application users must receive only required privileges.

Avoid:

```sql
GRANT ALL PRIVILEGES ...
```

unless explicitly justified.

Applications must not normally have:

```text
SUPERUSER
CREATEDB
CREATEROLE
REPLICATION
BYPASSRLS
```

---

# 25. pg_hba.conf

Restrict connections by:

```text
database
user
source network
authentication method
```

Example:

```text
hostssl app01db app01_login 10.20.10.0/24 scram-sha-256
```

Avoid unrestricted entries such as:

```text
host all all 0.0.0.0/0 trust
```

Never use `trust` for remote production access.

---

# 26. Network Security

Firewall access should follow:

```text
Application Subnet
       |
       v
   PgBouncer
       |
       v
 PostgreSQL
```

Do not expose PostgreSQL directly to the public Internet.

Allow only:

* Application servers
* DBA administration network
* Monitoring servers
* Backup infrastructure
* Replication servers

---

# 27. TLS

Production client/server connections crossing network boundaries should use TLS.

Protect:

* Application credentials
* SQL traffic
* Sensitive database data
* Administrative sessions

Certificates must have documented renewal procedures.

---

# 28. Database Naming Standard

Use predictable naming.

Example:

```text
prd_app01_core
prd_app01_reporting

prd_app02_core
prd_app02_audit
```

Avoid names such as:

```text
test123
newdb
database1
finaldb
```

---

# 29. Schema Management

Application objects should use application-specific schemas.

Example:

```text
prd_app01
 |
 +-- app
 +-- audit
 +-- staging
```

Avoid uncontrolled creation of application objects in `public`.

---

# 30. WAL Configuration

WAL is critical for:

* crash recovery
* replication
* point-in-time recovery
* backup consistency

Monitor:

```text
WAL generation rate
WAL archive failures
WAL directory size
replication lag
```

Never manually delete files from:

```text
pg_wal/
```

to recover disk space.

This can make the database unrecoverable.

---

# 31. Checkpoint Management

On HDD storage, aggressive checkpoints can cause serious latency.

Monitor:

```sql
SELECT *
FROM pg_stat_checkpointer;
```

Investigate:

* frequent requested checkpoints
* excessive checkpoint write times
* WAL growth
* storage latency during checkpoints

Do not simply increase checkpoint/WAL parameters without understanding storage capacity and recovery requirements.

---

# 32. Autovacuum

Autovacuum must remain enabled.

Do not globally disable:

```ini
autovacuum = off
```

Autovacuum is essential for:

* reclaiming reusable tuple space
* maintaining statistics
* preventing transaction-ID wraparound
* controlling table/index health

Large/high-churn tables may require individual autovacuum settings.

---

# 33. Autovacuum Monitoring

Monitor:

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
ORDER BY n_dead_tup DESC;
```

Investigate tables with:

* high dead tuple counts
* missing autovacuum activity
* stale analyze timestamps
* rapid growth

---

# 34. Statistics

PostgreSQL's statistics views should be part of routine monitoring. PostgreSQL provides views for tables, indexes, I/O and active database sessions as well as progress views for maintenance operations. ([PostgreSQL][5])

Important sources include:

```text
pg_stat_activity
pg_stat_database
pg_stat_user_tables
pg_stat_user_indexes
pg_statio_user_tables
pg_statio_user_indexes
pg_stat_replication
pg_locks
```

---

# 35. pg_stat_statements

Enable `pg_stat_statements` where operational policy permits.

Use it to identify:

* highest total execution time
* most frequently executed SQL
* high I/O queries
* slow queries
* expensive application SQL

Example analysis:

```sql
SELECT
    calls,
    total_exec_time,
    mean_exec_time,
    rows,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

Reset statistics only through a controlled process because historical performance information is valuable.

---

# 36. Long-Running Query Monitoring

Monitor:

```sql
SELECT
    pid,
    usename,
    datname,
    state,
    now() - query_start AS duration,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

Suggested starting alert:

```text
Interactive query > 5 minutes
```

But thresholds must be workload-specific.

Batch jobs may legitimately run longer.

---

# 37. Idle-in-Transaction Sessions

Monitor:

```sql
SELECT
    pid,
    usename,
    datname,
    now() - xact_start AS transaction_age,
    state,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction';
```

Long idle transactions can:

* retain old row versions
* interfere with vacuum cleanup
* increase table bloat
* hold locks

Use an appropriate:

```ini
idle_in_transaction_session_timeout
```

after application testing.

---

# 38. Lock Monitoring

Use:

```sql
SELECT *
FROM pg_locks;
```

Operational monitoring should detect:

```text
Blocked sessions
Blocking sessions
Long lock waits
DDL blocking production transactions
```

Do not kill sessions without identifying the blocker and business impact.

---

# 39. Query Optimization Procedure

When performance problems occur:

```text
1. Identify slow SQL
       |
2. Check pg_stat_statements
       |
3. Examine wait events
       |
4. EXPLAIN
       |
5. EXPLAIN (ANALYZE, BUFFERS)
       |
6. Check indexes/statistics
       |
7. Correct query/schema
       |
8. Load test
       |
9. Deploy through change control
```

Use `EXPLAIN ANALYZE` carefully because it **executes the query**.

Do not casually execute it against destructive or expensive production statements.

---

# 40. Index Management

Do not create indexes merely because a column appears in a `WHERE` clause.

Review:

* query plans
* selectivity
* write overhead
* index size
* duplicate indexes
* unused indexes

Check usage:

```sql
SELECT
    schemaname,
    relname,
    indexrelname,
    idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan;
```

Never automatically drop an index solely because `idx_scan = 0`.

Statistics may have been reset or the index may support infrequent critical workloads.

---

# 41. Backup Strategy

Production PostgreSQL must have:

```text
Physical backup
+
WAL archive
+
PITR capability
+
Logical backup where appropriate
+
Off-server backup copy
+
Restore testing
```

A backup is not considered operationally valid until recovery has been tested.

---

# 42. Physical Backup

Use supported physical backup tooling such as:

```text
pg_basebackup
```

PostgreSQL documents `pg_basebackup` as capable of backing up a running cluster and using the result for point-in-time recovery or replication. ([PostgreSQL][6])

A physical backup covers the entire PostgreSQL cluster rather than one individual database. ([PostgreSQL][6])

---

# 43. Logical Backup

For individual database/object recovery use:

```text
pg_dump
pg_dumpall
```

Example:

```bash
pg_dump -Fc -d prd_app01_core \
  -f prd_app01_core.dump
```

Use logical backups as a complement to, not an automatic replacement for, physical backup/PITR strategy.

---

# 44. Recommended Backup Schedule

Example baseline:

| Backup                             | Frequency  |                 Retention |
| ---------------------------------- | ---------- | ------------------------: |
| Physical full backup               | Weekly     |                   4 weeks |
| Incremental/approved backup method | Daily      |                 7–14 days |
| WAL archive                        | Continuous | Based on PITR requirement |
| Logical critical DB backup         | Daily      |                 7–14 days |
| Monthly archive                    | Monthly    |           Business policy |

Retention must ultimately follow business, legal and recovery requirements.

---

# 45. Recovery Objectives

Business owners must define:

**RPO — Recovery Point Objective**

Example:

```text
RPO <= 15 minutes
```

**RTO — Recovery Time Objective**

Example:

```text
RTO <= 2 hours
```

Backup architecture must be designed from these objectives.

Do not select RPO/RTO values solely from the DBA team's preference.

---

# 46. Backup Location

Never use:

```text
Database
   |
   +---- /pgdata
   |
   +---- /pgbackup
           |
        Same disk
```

as the only backup strategy.

Preferred:

```text
Production DB
      |
      +---- Local staging backup
      |
      +---- Remote backup storage
      |
      +---- Offsite/Object storage
```

A storage failure must not destroy both production data and all backup copies.

---

# 47. Restore Testing

At least monthly, perform a controlled restore test.

Procedure:

```text
Backup
   |
Restore to isolated server
   |
Start PostgreSQL
   |
Validate database
   |
Run consistency/application checks
   |
Record recovery time
   |
Compare against RTO
```

Document:

```text
Backup ID
Backup timestamp
Restore start
Restore finish
Recovery duration
PITR target
Database validation
Application validation
Result
Engineer
```

---

# 48. High Availability

A single PostgreSQL server remains a single point of failure regardless of how many PostgreSQL instances it contains.

Current architecture:

```text
Applications
     |
Ubuntu Server
     |
PostgreSQL
```

Failure of the server means all databases become unavailable.

For critical production:

```text
                Applications
                     |
               HA / DB endpoint
                     |
             +-------+-------+
             |               |
         Primary          Standby
       PostgreSQL        PostgreSQL
             |               |
             +------WAL------+
```

HA design should consider:

* streaming replication
* automatic or controlled failover
* fencing/split-brain protection
* PgBouncer behavior
* application reconnect behavior
* backup independence
* monitoring

PgBouncer itself should not be treated as the failover manager; its official FAQ states that PgBouncer does not provide internal failover detection/configuration. ([PgBouncer][7])

---

# 49. Replication Monitoring

For HA systems monitor:

```sql
SELECT *
FROM pg_stat_replication;
```

Alert on:

* disconnected standby
* excessive replication lag
* WAL retention growth
* replication slot inactivity
* standby replay delay

---

# 50. Replication Slots

Replication slots must be monitored carefully.

An abandoned slot can prevent WAL removal and consume storage until the server runs out of disk.

Monitor:

```sql
SELECT *
FROM pg_replication_slots;
```

Never leave obsolete replication slots active.

---

# 51. Disk Monitoring

Because the server uses HDD, monitor at minimum:

```text
Disk utilization
Read latency
Write latency
IOPS
Queue depth
Filesystem usage
WAL filesystem usage
Temporary-file activity
```

Linux tools:

```bash
iostat -xz 1
vmstat 1
df -h
du
```

PostgreSQL documentation specifically recommends combining PostgreSQL statistics with operating-system tools such as `iostat` and `vmstat` when diagnosing activity and performance. ([PostgreSQL][5])

---

# 52. Disk Capacity Thresholds

Suggested operational thresholds:

```text
< 70%       Normal
70–80%      Warning / capacity review
80–85%      High
85–90%      Critical
> 90%       Emergency
```

These are operational guardrails rather than PostgreSQL hard limits.

Do not wait for:

```text
100% filesystem usage
```

before acting.

---

# 53. Never Delete PostgreSQL Files Manually

Never manually delete:

```text
pg_wal/*
base/*
global/*
pg_tblspc/*
```

to recover space.

Use PostgreSQL-supported procedures to identify and resolve the root cause.

---

# 54. Database Growth Monitoring

Record:

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname))
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

Trend growth over time.

Forecast:

```text
Current size
+
Monthly growth
+
12-month forecast
+
Backup overhead
+
WAL overhead
+
Safety capacity
```

---

# 55. Table Size Monitoring

Use:

```sql
SELECT
    schemaname,
    relname,
    pg_size_pretty(
      pg_total_relation_size(
        quote_ident(schemaname) || '.' || quote_ident(relname)
      )
    ) AS total_size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(
    quote_ident(schemaname) || '.' || quote_ident(relname)
) DESC;
```

Review the largest and fastest-growing objects.

---

# 56. Logging

Enable sufficient logging for troubleshooting without producing uncontrolled disk usage.

Consider:

```ini
logging_collector = on

log_connections = on
log_disconnections = on

log_lock_waits = on

log_min_duration_statement = 1000
```

The exact slow-query threshold must be adjusted to the application SLA.

For highly transactional systems, logging every statement should generally be avoided unless temporarily required for troubleshooting/audit because of performance, storage and sensitive-data implications.

---

# 57. Log Rotation

PostgreSQL logs must be rotated.

Monitor:

```text
Log directory size
Retention
Compression
Filesystem usage
```

Logs should integrate with the organization's centralized logging platform where available.

---

# 58. Monitoring Dashboard

At minimum monitor:

### Host

```text
CPU utilization
Load average
RAM
Swap
Disk capacity
Disk latency
IOPS
Network
```

### PostgreSQL

```text
Connections
Transactions/sec
Commits
Rollbacks
Cache activity
Locks
Blocking sessions
Deadlocks
Long transactions
Autovacuum
Dead tuples
Temporary files
Database growth
WAL generation
Checkpoints
Replication lag
```

### PgBouncer

```text
Client connections
Server connections
Waiting clients
Pool utilization
Connection limits
```

---

# 59. Alerting

Suggested initial alerts:

| Metric                    | Initial Trigger          |
| ------------------------- | ------------------------ |
| CPU                       | >85% sustained           |
| RAM                       | >90% pressure            |
| Swap                      | sustained swapping       |
| Disk                      | >80% warning             |
| Disk                      | >90% critical            |
| PostgreSQL connections    | >80% configured capacity |
| PgBouncer waiting clients | sustained >0             |
| Long transactions         | workload-defined         |
| Replication lag           | RPO-defined              |
| WAL archive failure       | immediate                |
| Backup failure            | immediate                |
| Autovacuum failure        | immediate                |
| Database unavailable      | immediate                |

Thresholds must be tuned after establishing normal workload baselines.

---

# 60. Daily DBA Checklist

* [ ] Verify both PostgreSQL instances are running.
* [ ] Verify PgBouncer services are running.
* [ ] Check backup completion.
* [ ] Check WAL archival.
* [ ] Check filesystem capacity.
* [ ] Check database growth.
* [ ] Review failed connections.
* [ ] Review PostgreSQL errors.
* [ ] Review long-running queries.
* [ ] Review blocked sessions.
* [ ] Review idle-in-transaction sessions.
* [ ] Review replication status if HA is enabled.
* [ ] Check autovacuum health.
* [ ] Check disk latency.
* [ ] Check CPU and memory.
* [ ] Check PgBouncer waiting clients.

---

# 61. Weekly DBA Checklist

* [ ] Review database growth trends.
* [ ] Review top SQL by total execution time.
* [ ] Review slow SQL.
* [ ] Review unused/duplicate index candidates.
* [ ] Review table/index growth.
* [ ] Review dead tuples.
* [ ] Review vacuum/analyze activity.
* [ ] Review checkpoint behavior.
* [ ] Review WAL generation.
* [ ] Review connection-pool utilization.
* [ ] Review capacity trends.
* [ ] Review security/authentication failures.

---

# 62. Monthly DBA Checklist

* [ ] Perform restore test.
* [ ] Verify PITR process.
* [ ] Review OS security patches.
* [ ] Review PostgreSQL minor updates.
* [ ] Review PgBouncer updates.
* [ ] Review user accounts.
* [ ] Remove obsolete access.
* [ ] Review database permissions.
* [ ] Review backup retention.
* [ ] Review storage forecast.
* [ ] Review monitoring thresholds.
* [ ] Review application growth.
* [ ] Document operational findings.

---

# 63. PostgreSQL Patching

Minor releases should be handled through controlled patch management.

Procedure:

```text
1. Review release notes
2. Verify compatibility
3. Confirm backup
4. Confirm rollback/recovery plan
5. Test non-production
6. Schedule maintenance
7. Stop/drain application traffic
8. Patch
9. Restart if required
10. Validate PostgreSQL
11. Validate PgBouncer
12. Validate applications
13. Monitor
14. Close change
```

---

# 64. Major PostgreSQL Upgrade

Major upgrades require a dedicated migration plan.

Options may include:

```text
pg_upgrade
logical replication
dump/restore
blue/green migration
```

Never perform an untested major upgrade directly against production.

---

# 65. Production Change Procedure

All database changes must include:

```text
Change description
Reason
Impact
Affected DB
Affected applications
Implementation procedure
Validation
Rollback procedure
Owner
Maintenance window
Approvals
```

---

# 66. Schema Changes

Large production DDL can cause locks.

Before executing:

```sql
ALTER TABLE ...
CREATE INDEX ...
DROP INDEX ...
```

evaluate:

```text
Table size
Lock level
Execution duration
Application impact
Disk requirements
WAL generation
Replication impact
Rollback procedure
```

Use online/concurrent capabilities where PostgreSQL supports them and where their tradeoffs are understood.

---

# 67. Incident — High CPU

Procedure:

```text
1. Check OS CPU
2. Check pg_stat_activity
3. Check pg_stat_statements
4. Identify expensive SQL
5. Check parallel queries
6. Check autovacuum
7. Check application connection spike
8. Check query plan
9. Mitigate offending workload
10. Perform root-cause analysis
```

Do not restart PostgreSQL as the first troubleshooting action.

---

# 68. Incident — High Memory

Check:

```text
PostgreSQL connections
work_mem usage patterns
parallel workers
maintenance operations
autovacuum
application connection spike
OS page cache
swap
OOM logs
```

Never immediately increase RAM parameters because free memory appears available.

---

# 69. Incident — Disk 90%+

Procedure:

```text
1. Identify filesystem.
2. Determine growth source.
3. Check WAL.
4. Check replication slots.
5. Check logs.
6. Check temp files.
7. Check database growth.
8. Check failed archiving.
9. Expand storage if necessary.
10. Correct root cause.
```

Never delete `pg_wal` manually.

---

# 70. Incident — Too Many Connections

Check:

```sql
SELECT
    datname,
    usename,
    state,
    count(*)
FROM pg_stat_activity
GROUP BY datname, usename, state
ORDER BY count(*) DESC;
```

Then inspect PgBouncer.

Possible causes:

```text
Application connection leak
Incorrect pool sizing
Traffic spike
Slow database queries
Blocked transactions
Insufficient backend pool
```

Do not automatically solve the issue by increasing:

```ini
max_connections
```

Increasing it can increase memory and CPU pressure.

---

# 71. Incident — Database Slow

Follow:

```text
Application slow
      |
      v
Check DB connections
      |
      v
Check locks
      |
      v
Check wait events
      |
      v
Check CPU/RAM
      |
      v
Check disk latency
      |
      v
Check slow SQL
      |
      v
Check execution plan
      |
      v
Check vacuum/statistics
      |
      v
Check recent changes
```

Given the current server architecture, **HDD latency must be one of the first checks**.

---

# 72. Emergency Restart

Restart only when justified.

Before restart, where possible capture:

```text
pg_stat_activity
blocking sessions
system CPU
memory
disk statistics
PostgreSQL logs
PgBouncer status
```

Then use controlled service-management commands.

After restart validate:

```text
PostgreSQL process
Port
Database connections
Recovery state
Logs
PgBouncer
Applications
Replication
Monitoring
```

---

# 73. Security Audit

Quarterly review:

```text
Login roles
Superusers
Replication roles
CREATEDB roles
CREATEROLE roles
Expired accounts
Unused accounts
pg_hba.conf
TLS
Firewall
Database grants
Schema grants
Default privileges
```

---

# 74. Capacity Planning

Track monthly:

```text
Database size
Table growth
Index growth
WAL/day
Backup size
CPU peak
RAM peak
Connections peak
Disk latency
TPS
Query latency
```

Forecast at least:

```text
6 months
12 months
24 months
```

---

# 75. Scale-Up Trigger

Consider additional RAM/CPU/storage when sustained utilization reaches operational limits.

For this server, storage should receive special attention before CPU upgrades.

Current:

```text
24 CPU       GOOD
64 GB RAM    GOOD
8 TB HDD     PERFORMANCE RISK
```

Recommended priority:

```text
1. SSD/NVMe storage improvement
2. Backup/restore architecture
3. HA/standby
4. Monitoring
5. Connection pooling
6. Query/index tuning
7. RAM/CPU expansion if measurements justify it
```

---

# 76. Recommended Target Architecture

For a stronger production platform:

```text
                   5 Applications
                         |
                    Load/Service
                         |
                 +-------+-------+
                 |               |
           PgBouncer 01     PgBouncer 02
                 |               |
                 +-------+-------+
                         |
                 PostgreSQL Primary
                  24 CPU / 64 GB
                         |
                 +-------+-------+
                 |               |
               NVMe            WAL/SSD
                 |
          Active Databases
                 |
           Streaming WAL
                 |
                 v
            PostgreSQL
              Standby
                 |
                 v
          Backup Repository
                 |
                 v
          Offsite/Object Copy
```

This eliminates several weaknesses of the original single-server/single-HDD design.

---

# 77. Production Acceptance Criteria

Before go-live all of the following must be completed:

* [ ] PostgreSQL installed from approved source.
* [ ] PostgreSQL version recorded.
* [ ] Both clusters created.
* [ ] Separate ports verified.
* [ ] Data directory permissions verified.
* [ ] PostgreSQL does not run as root.
* [ ] Memory settings reviewed.
* [ ] Connection limits reviewed.
* [ ] PgBouncer installed and tested.
* [ ] Application pooling compatibility tested.
* [ ] Application roles created.
* [ ] Least privilege verified.
* [ ] `pg_hba.conf` reviewed.
* [ ] Firewall reviewed.
* [ ] TLS configured where required.
* [ ] Monitoring enabled.
* [ ] Logging enabled.
* [ ] Log rotation configured.
* [ ] Backup completed.
* [ ] WAL archive tested.
* [ ] PITR tested.
* [ ] Full restore tested.
* [ ] Backup stored off-server.
* [ ] Capacity monitoring configured.
* [ ] Disk alerts configured.
* [ ] Connection alerts configured.
* [ ] Long-query monitoring configured.
* [ ] Autovacuum monitoring configured.
* [ ] Application load testing completed.
* [ ] Failover tested where HA exists.
* [ ] Runbook approved.
* [ ] DBA handover completed.

---

# 78. Critical Production Rules

## Rule 1

Never disable autovacuum globally.

## Rule 2

Never delete WAL files manually.

## Rule 3

Never give applications PostgreSQL superuser access.

## Rule 4

Never expose PostgreSQL directly to the public Internet.

## Rule 5

Never assume a backup works without testing a restore.

## Rule 6

Never keep the only backup on the production database disk.

## Rule 7

Never increase `max_connections` simply because applications are exhausting connections.

Investigate pooling first.

## Rule 8

Never increase `work_mem` globally without calculating concurrency risk.

## Rule 9

Never restart production PostgreSQL as the first performance troubleshooting action.

## Rule 10

Never execute large DDL changes without evaluating locking and rollback.

## Rule 11

Never perform an untested major-version upgrade.

## Rule 12

Never change production parameters without recording the previous value and rollback procedure.

---

# 79. Recommended Operational Model

For the current environment:

```text
24 CPU
64 GB RAM
8 TB HDD
10 Databases
5 Applications

              |
              v

       2 PostgreSQL Instances

       Instance01 :5432
           ~6 databases
           Critical workload

       Instance02 :5433
           ~4 databases
           Standard workload

              |
              v

          PgBouncer

              |
              v

     Monitoring + Alerting

              |
              v

       Physical Backup
              +
         WAL Archive
              +
             PITR
              +
       Logical Backup
              +
       Restore Testing
              +
       Offsite Backup
```

---

# 80. Final Architecture Decision

For the stated workload, the approved starting architecture is:

**PostgreSQL instances:** 2

**Database distribution:** Approximately 6 + 4, but final placement must be workload-driven.

**Connection management:** PgBouncer.

**Memory:** 16 GB combined initial `shared_buffers`, leaving substantial RAM available for OS cache and per-session PostgreSQL activity.

**Storage:** Existing 8 TB HDD can be used subject to performance validation, but SSD/NVMe is strongly recommended for active production databases if the applications are transactional or I/O intensive.

**Backup:** Physical backup + WAL archive/PITR + selected logical backups + off-server copy.

**Availability:** A second server/standby is required if business requirements cannot tolerate failure of the current Ubuntu host.

**Monitoring:** PostgreSQL + PgBouncer + Linux OS + disk I/O monitoring is mandatory.

**Change management:** All PostgreSQL configuration, DDL, patching and upgrades must follow controlled implementation, validation and rollback procedures.

---

# 81. Reference Standards

The SOP is based primarily on current PostgreSQL and PgBouncer operational guidance:

* PostgreSQL resource and memory configuration — `shared_buffers`, `work_mem`, maintenance memory and related resource controls. ([PostgreSQL][1])
* PostgreSQL monitoring and statistics facilities, including database, table, index, I/O, lock and progress monitoring. ([PostgreSQL][5])
* PostgreSQL physical backup using `pg_basebackup` and its use for PITR/standby initialization. ([PostgreSQL][6])
* PgBouncer connection pooling behavior and operational interface. ([PgBouncer][2])
* PgBouncer session versus transaction pooling compatibility considerations. ([PgBouncer][4])
* PgBouncer pool sizing and connection-limit behavior. ([PgBouncer][3])
* PgBouncer failover limitation: HA/failover must be provided by the surrounding architecture rather than assumed to be provided by PgBouncer. ([PgBouncer][7])

---

# 82. Approval

| Role                     | Responsibility                          |
| ------------------------ | --------------------------------------- |
| DBA                      | PostgreSQL configuration and operations |
| Linux Administrator      | OS/storage configuration                |
| Application Owner        | Application compatibility               |
| Security Team            | Access/TLS/security review              |
| Backup Team              | Backup/recovery                         |
| Infrastructure Architect | Architecture approval                   |
| Business Owner           | RPO/RTO approval                        |

**Production deployment should proceed only after the architecture, recovery objectives, backup design, security model and performance/load-test results have been formally accepted.**

A key point in this SOP is that the values are **baselines, not magic production tuning numbers**. PostgreSQL memory, WAL, autovacuum, connection and checkpoint settings should be changed from these baselines only after measuring the real workload. PostgreSQL itself provides the statistics framework for doing that, while Linux tools such as `iostat` and `vmstat` are important for identifying the HDD bottleneck. ([PostgreSQL][5])

For your particular server, I would treat **HDD performance, backup/PITR, and single-host failure** as the three highest architectural risks. A second PostgreSQL instance gives workload separation, but it does not provide HA because both instances still disappear if the Ubuntu server fails.

[1]: https://www.postgresql.org/docs/current/runtime-config-resource.html "PostgreSQL: Documentation: 18: 19.4. Resource Consumption"
[2]: https://www.pgbouncer.org/usagem "PgBouncer command-line usage"
[3]: https://www.pgbouncer.org/config "PgBouncer config"
[4]: https://www.pgbouncer.org/features.html "PgBouncer features"
[5]: https://www.postgresql.org/docs/current/monitoring.html "PostgreSQL: Documentation: 18: Chapter 27. Monitoring Database Activity"
[6]: https://www.postgresql.org/docs/current/app-pgbasebackup.html "PostgreSQL: Documentation: 18: pg_basebackup"
[7]: https://www.pgbouncer.org/faq.html "PgBouncer FAQ"
