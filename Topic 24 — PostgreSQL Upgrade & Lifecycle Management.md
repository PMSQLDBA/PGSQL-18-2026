# Topic 24 — PostgreSQL Upgrade & Lifecycle Management

These notes are verified against the **current PostgreSQL 18 official documentation and PostgreSQL versioning policy as of August 19, 2026**. 

PostgreSQL 18 is the current major release, and the latest supported minor releases are **18.6, 17.11, 16.15, 15.19, and 14.24**. ([PostgreSQL][1])

---

## 1. Why PostgreSQL upgrade management matters

PostgreSQL upgrades are not just software installation activities. From an architecture and DBA point of view, an upgrade can affect:

* database file format and system catalogs
* extensions and shared libraries
* SQL/application compatibility
* query plans and optimizer behavior
* replication
* authentication and configuration
* backup/recovery procedures
* monitoring
* operating system packages
* drivers and connection pools
* downtime and rollback capability

The upgrade strategy should therefore be treated as a **planned lifecycle activity**, not as an ad-hoc server patch.

PostgreSQL has two fundamentally different types of upgrades:

```text
PostgreSQL Upgrade
        |
        +-- Minor Upgrade
        |      Example: 18.5 -> 18.6
        |
        +-- Major Upgrade
               Example: 17 -> 18
```

The difference is very important because PostgreSQL guarantees binary compatibility within a major version, but not between major versions. 

Major releases can change system catalogs and other internal structures, so a major upgrade requires a migration mechanism such as `pg_upgrade`, dump/restore, or logical replication. ([PostgreSQL][1])

---

# 2. PostgreSQL version numbering

Since PostgreSQL 10, versioning is straightforward.

```text
18.6
| |
| +---- Minor version
|
+------ Major version
```

Examples:

```text
18.5 -> 18.6     Minor upgrade
17.11 -> 18.6    Major upgrade

16.15 -> 17.11   Major upgrade
14.24 -> 18.6    Major upgrade
```

For releases older than PostgreSQL 10, the numbering convention was different. For example, `9.5 -> 9.6` was considered a major upgrade. ([PostgreSQL][1])

---

# 3. Minor PostgreSQL upgrades

A minor upgrade stays within the same major version.

Example:

```text
PostgreSQL 18.4
       |
       v
PostgreSQL 18.6
```

PostgreSQL minor releases normally contain:

* security fixes
* bug fixes
* data-corruption fixes
* fixes for commonly encountered issues
* relatively low-risk corrections

They do **not normally introduce major new functionality or incompatible data formats**.

PostgreSQL explicitly recommends running the **latest minor release for your major version** because continuing to run an older minor release can be riskier than applying the minor upgrade. ([PostgreSQL][1])

---

## 4. Minor upgrade procedure

A typical self-managed PostgreSQL minor upgrade is:

```text
1. Review release notes
2. Take/verify backup
3. Confirm replica status
4. Stop PostgreSQL
5. Upgrade PostgreSQL binaries/packages
6. Start PostgreSQL
7. Run any release-specific remediation
8. Validate database
9. Validate applications
10. Monitor
```

No `pg_dump`/restore or `pg_upgrade` is normally required.

Officially, PostgreSQL describes the basic process as:

```text
Stop server
     |
Install new minor-version binaries
     |
Restart server
```

However, you must still read the specific minor release notes because a release may require extra post-update actions. ([PostgreSQL][1])

For example, PostgreSQL 18.6 does not require dump/restore when upgrading from another 18.x release, but its release notes identify additional remediation for some security changes and particular index/extension-related issues. ([PostgreSQL][2])

---

# 5. Patching strategy

In PostgreSQL terminology, what many organizations call a **database patch** is normally delivered through a **minor release**.

For example:

```text
18.3 -> 18.4
17.10 -> 17.11
16.14 -> 16.15
```

A good enterprise patch process is:

```text
PostgreSQL release published
          |
          v
Security/bug assessment
          |
          v
Non-production validation
          |
          v
Change approval
          |
          v
Production rollout
          |
          v
Post-change validation
```

Avoid automatically assuming that every minor update is simply "replace binaries and restart." Release-specific actions may still be required. PostgreSQL's current 18.6 release is a good example: it includes security fixes and additional remedial steps that some environments need after installing the binaries. ([PostgreSQL][3])

---

# 6. Major PostgreSQL upgrades

A major upgrade changes the PostgreSQL major version.

Examples:

```text
14 -> 18
15 -> 18
16 -> 18
17 -> 18
```

PostgreSQL allows skipping intermediate major versions. For example:

```text
PostgreSQL 14
      |
      | direct
      v
PostgreSQL 18
```

You do not have to perform:

```text
14 -> 15 -> 16 -> 17 -> 18
```

However, PostgreSQL recommends reviewing the release notes for **all intervening releases**, especially their migration and incompatibility sections. ([PostgreSQL][1])

---

# 7. Major upgrade methods

There are three main PostgreSQL-native strategies.

```text
                  Major Upgrade
                       |
       +---------------+---------------+
       |               |               |
       v               v               v
   Dump/Restore     pg_upgrade     Logical Replication
```

Each has different trade-offs.

---

# 8. Method 1 — Dump and restore

This is the traditional logical migration method.

High-level flow:

```text
Old PostgreSQL
      |
    pg_dump
      |
      v
Logical SQL/archive
      |
      v
New PostgreSQL
```

For an entire cluster, tools include:

```bash
pg_dumpall
```

or database-by-database:

```bash
pg_dump
pg_restore
```

PostgreSQL specifically notes that filesystem-level copies are **not suitable for moving a data directory directly between incompatible major versions**. For this migration style, use logical backup tools. ([PostgreSQL][4])

---

## Advantages

Dump/restore gives you a completely rebuilt database environment.

It can be useful when you want to:

* reorganize the database
* change storage layouts
* clean up historical objects
* migrate selectively
* perform some platform changes
* validate the database from a logical representation

---

## Disadvantages

The main problem is time.

For a multi-terabyte database:

```text
Dump time
+
Restore time
+
Index creation
+
Constraint creation
+
Statistics generation
=
Potentially long outage
```

Because of this, dump/restore is usually less attractive for very large production databases with strict downtime requirements.

---

# 9. Method 2 — pg_upgrade

`pg_upgrade` is the main tool for fast in-place-style PostgreSQL major upgrades.

PostgreSQL describes `pg_upgrade` as a way to upgrade PostgreSQL data files to a newer major version **without the normal dump/restore process**. It builds new system catalogs while reusing or efficiently transferring the user data files. ([PostgreSQL][5])

Conceptually:

```text
Old cluster
PostgreSQL 17
    |
    | pg_upgrade
    v
New cluster
PostgreSQL 18
```

---

# 10. Why pg_upgrade is fast

A normal logical migration may require:

```text
Read every table
      |
Convert to logical representation
      |
Write every table
      |
Rebuild everything
```

`pg_upgrade` avoids much of that.

Conceptually:

```text
New PostgreSQL catalogs
          +
Existing compatible user data files
          =
Upgraded cluster
```

Because PostgreSQL's user data storage format often remains compatible across major releases, `pg_upgrade` can avoid rewriting all table data. ([PostgreSQL][5])

---

# 11. pg_upgrade compatibility checks

Before upgrading, always run:

```bash
pg_upgrade --check
```

Example:

```bash
/usr/pgsql-18/bin/pg_upgrade \
  --old-bindir=/usr/pgsql-17/bin \
  --new-bindir=/usr/pgsql-18/bin \
  --old-datadir=/pgdata/17 \
  --new-datadir=/pgdata/18 \
  --check
```

`--check` verifies compatibility without modifying cluster data and identifies manual work that may be needed after the upgrade. ([PostgreSQL][5])

A major enterprise rule should therefore be:

```text
Never begin the production pg_upgrade window
without a successful pg_upgrade --check.
```

---

# 12. What pg_upgrade checks

`pg_upgrade` checks several binary compatibility requirements, including relevant PostgreSQL compile-time characteristics.

But an important limitation is:

> external module binary compatibility cannot be fully checked by `pg_upgrade`.

Therefore extension/library validation must be a separate upgrade workstream. ([PostgreSQL][5])

---

# 13. pg_upgrade file transfer modes

Current PostgreSQL 18 supports several transfer approaches.

The most important ones are:

```text
--copy
--link
--clone
--copy-file-range
--swap
```

([PostgreSQL][5])

---

## 13.1 `--copy`

This is the default.

```text
Old data files
     |
    COPY
     |
New data files
```

Example:

```bash
pg_upgrade \
   --old-bindir=/usr/pgsql-17/bin \
   --new-bindir=/usr/pgsql-18/bin \
   --old-datadir=/data/17 \
   --new-datadir=/data/18
```

Advantages:

* safest/simple model
* old cluster remains independent
* easy rollback before old cluster is retired

Disadvantage:

* requires substantial extra disk space
* copying very large clusters can take time

---

# 14. `--link` mode

Example:

```bash
pg_upgrade \
  --old-bindir=/usr/pgsql-17/bin \
  --new-bindir=/usr/pgsql-18/bin \
  --old-datadir=/data/17 \
  --new-datadir=/data/18 \
  --link
```

Instead of physically copying data files, hard links are created.

Conceptually:

```text
Old cluster ----+
                |
                +--> Same underlying data file
                |
New cluster ----+
```

Advantages:

* extremely fast
* very little additional disk space
* suitable for very large databases

But the rollback implication is critical.

Once the **new cluster has been started**, it can modify files shared with the old cluster, making the old cluster unsafe to restart. At that point rollback requires restoring the old environment from backup. ([PostgreSQL][5])

---

# 15. `--clone`

Clone mode uses filesystem copy-on-write/reflink functionality where supported.

Conceptually:

```text
Old file
   |
copy-on-write clone
   |
New file
```

It provides much of the speed and space efficiency of link mode while keeping the old cluster untouched.

PostgreSQL 18 documents support on certain filesystems/platforms, including appropriate Btrfs/XFS environments on Linux and APFS on macOS. ([PostgreSQL][5])

From a rollback perspective, clone mode can therefore be much more attractive than link mode when your storage platform supports it.

---

# 16. `--swap`

PostgreSQL 18 also supports `--swap`.

This mode moves the underlying data directories and replaces catalog files with the newly generated catalogs. PostgreSQL notes that it can be especially fast for clusters containing a very large number of relations. ([PostgreSQL][5])

But it has an important consequence:

```text
After destructive transfer begins
        |
        v
Old cluster is no longer safe
```

Rollback therefore requires careful backup planning. ([PostgreSQL][5])

---

# 17. Parallelizing pg_upgrade

You can use:

```bash
--jobs=N
```

Example:

```bash
pg_upgrade \
  --jobs=8 \
  ...
```

Parallel processing can improve upgrade time for environments containing multiple databases and tablespaces. PostgreSQL suggests CPU count as a reasonable starting point when choosing the number of jobs. ([PostgreSQL][5])

---

# 18. Important pg_upgrade workflow

A strong production workflow looks like:

```text
1. Install target PostgreSQL version
         |
2. Initialize new cluster
         |
3. Install required extension libraries
         |
4. Copy required custom files
         |
5. Configure authentication
         |
6. Run pg_upgrade --check
         |
7. Resolve compatibility issues
         |
8. Stop applications
         |
9. Stop old/new PostgreSQL clusters
         |
10. Confirm replication caught up
         |
11. Run pg_upgrade
         |
12. Apply new configuration
         |
13. Start new PostgreSQL
         |
14. Run post-upgrade scripts
         |
15. Refresh statistics
         |
16. Application validation
         |
17. Performance validation
         |
18. Delete old environment later
```

This broadly follows the official `pg_upgrade` procedure. ([PostgreSQL][5])

---

# 19. Extensions are one of the biggest upgrade risks

Before a major upgrade, inventory every installed extension.

Useful query:

```sql
SELECT
    extname,
    extversion
FROM pg_extension
ORDER BY extname;
```

Example result:

```text
extname             extversion
------------------  ----------
pg_stat_statements  1.11
postgis             3.5.2
pgcrypto            1.3
uuid-ossp           1.1
```

PostgreSQL stores installed extension information in `pg_extension`. ([PostgreSQL][6])

---

# 20. Why extension compatibility matters

Some extensions contain only SQL.

Others contain native libraries:

```text
Extension
    |
    +-- SQL definitions
    |
    +-- C/shared libraries
          |
          +-- extension.so
```

For native extensions, the shared library must be compatible with the new PostgreSQL server binary.

PostgreSQL explicitly requires matching extension shared-object files to be installed for the new PostgreSQL version before `pg_upgrade`. You should **not simply run `CREATE EXTENSION` again** in the new database because the extension metadata and SQL objects are already migrated from the old cluster. ([PostgreSQL][5])

---

# 21. Extension upgrade validation

Useful views include:

```sql
SELECT *
FROM pg_available_extensions;
```

and:

```sql
SELECT *
FROM pg_available_extension_versions;
```

PostgreSQL documents these views as the mechanism for seeing extensions and extension versions available to the server. ([PostgreSQL][7])

---

# 22. Updating extension versions

After the database upgrade, an extension may have its own schema/version upgrade.

Example:

```sql
ALTER EXTENSION hstore UPDATE;
```

or:

```sql
ALTER EXTENSION hstore
UPDATE TO '2.0';
```

For this to work, the extension package must provide the necessary upgrade script or sequence of upgrade scripts. ([PostgreSQL][8])

---

# 23. Extension compatibility checklist

For every extension, verify:

```text
Extension name
      |
Current extension version
      |
Target PostgreSQL version supported?
      |
Required OS package available?
      |
Native library compatible?
      |
Extension upgrade required?
      |
Extension upgrade ordering?
      |
Rollback supported?
```

Pay special attention to commonly used external components such as:

```text
PostGIS
TimescaleDB
pgvector
pgaudit
pglogical
Citus
oracle_fdw
custom C extensions
```

For third-party extensions, PostgreSQL documentation cannot guarantee compatibility; validate against the extension vendor/project documentation as well.

---

# 24. Configuration migration

Do not blindly copy the complete old `postgresql.conf` over the new version.

A better pattern is:

```text
New-version default postgresql.conf
             |
             +--- apply reviewed old custom settings
             |
             +--- review renamed/removed parameters
             |
             +--- review new parameters
```

Major releases can change configuration semantics and defaults, and PostgreSQL instructs administrators to restore/reapply needed configuration after the upgrade. ([PostgreSQL][5])

Important files include:

```text
postgresql.conf
postgresql.auto.conf
pg_hba.conf
pg_ident.conf
```

Also check:

```text
shared_preload_libraries
search_path
SSL configuration
logging
WAL configuration
autovacuum settings
memory settings
connection limits
archive_command
replication slots
```

---

# 25. Major upgrade using logical replication

Logical replication is another powerful major-version migration strategy.

Architecture:

```text
Old PostgreSQL
Publisher
     |
     | Logical replication
     v
New PostgreSQL
Subscriber
```

PostgreSQL logical replication can work across different major versions, which makes it useful for low-downtime upgrades. ([PostgreSQL][4])

---

# 26. Logical upgrade process

A simplified model:

```text
1. Build PostgreSQL 18 environment
         |
2. Create schema
         |
3. Configure publication on old server
         |
4. Configure subscription on new server
         |
5. Initial table synchronization
         |
6. Continuously replicate changes
         |
7. Wait until replication lag ≈ 0
         |
8. Stop application writes
         |
9. Catch up final changes
         |
10. Redirect applications
         |
11. New server becomes primary
```

PostgreSQL notes that this approach can reduce the switchover outage to only a small amount of time once the target has synchronized. ([PostgreSQL][4])

---

# 27. Benefits of logical upgrades

Logical replication is attractive when:

```text
Database is very large
+
Downtime requirement is very small
+
Old and new PostgreSQL must coexist
```

Other benefits:

* easy pre-cutover testing
* source stays operational during migration
* blue/green database architecture
* easier controlled cutover
* target can run different PostgreSQL major version

---

# 28. Logical replication limitations

Logical replication is **not the same as physical replication**.

You must think about:

* schema changes
* sequences
* replication identity
* DDL
* large objects depending on approach
* unlogged objects
* extensions
* unsupported object types
* replication slots
* publications/subscriptions
* conflict handling

PostgreSQL maintains a dedicated restrictions section because logical replication does not automatically replicate every kind of database object or operation. ([PostgreSQL][9])

Therefore don't assume:

```text
"Logical replication will copy the whole PostgreSQL server."
```

It will not.

---

# 29. PostgreSQL 17+ logical replication upgrade improvement

There is an important modern PostgreSQL enhancement.

Starting with PostgreSQL 17, `pg_upgrade` gained support for preserving more logical replication state. PostgreSQL 18's current upgrade documentation describes migration of logical slots on publishers and subscription state/dependencies when upgrading clusters originating from PostgreSQL 17 or later. ([PostgreSQL][10])

This can significantly simplify environments that already use PostgreSQL logical replication.

---

# 30. Logical replication cluster upgrade requirements

Current PostgreSQL 18 documentation states that migration of a logical replication cluster using this supported upgrade procedure requires the old logical-replication members to be **PostgreSQL 17.0 or later**. ([PostgreSQL][10])

For publisher upgrades, important requirements include:

```text
wal_level = logical

max_replication_slots
>= old logical slot count
```

The required output plugins must also exist in the new installation, and logical transactions should be fully replicated before the upgrade. ([PostgreSQL][10])

---

# 31. PostgreSQL major upgrade rollback planning

Rollback must be designed **before** the upgrade.

A major mistake is:

```text
Upgrade first
|
"How do we rollback?"
```

Instead:

```text
Rollback method
     |
Verified backup
     |
Restore test
     |
Decision point
     |
Upgrade
```

---

# 32. Rollback with pg_upgrade copy mode

If `--link` or `--swap` was not used, the old cluster remains unchanged by the data-transfer mechanism.

Rollback can therefore usually be:

```text
Stop PostgreSQL 18
      |
Switch configuration/application back
      |
Start PostgreSQL 17
```

provided no new business transactions must be retained from the new environment. PostgreSQL confirms that when neither link nor swap mode was used, the old cluster is left unmodified and can be restarted. ([PostgreSQL][5])

---

# 33. Rollback with pg_upgrade link mode

Link mode is very different.

Before new PostgreSQL starts:

```text
Rollback may still be possible
```

After new PostgreSQL starts:

```text
New PG writes
     |
shared hard-linked data files changed
     |
Old cluster becomes unsafe
```

At that point, rollback requires restoring the old cluster from backup. ([PostgreSQL][5])

This is one of the most important `pg_upgrade` interview and architecture points.

---

# 34. Rollback with logical replication

Logical migration provides a potentially cleaner rollback window because both systems initially remain independent.

Example:

```text
PG17                     PG18
Old primary ------------> New target

          Cutover
             |
             v

Application -----------> PG18
```

If you have not allowed irreversible divergence, you may be able to redirect traffic back.

However, once writes start occurring on PostgreSQL 18, the rollback design becomes harder because those changes must somehow reach PostgreSQL 17.

Therefore decide beforehand whether you will use:

```text
reverse logical replication
```

or:

```text
restore/replay mechanism
```

or accept:

```text
point of no return after cutover
```

---

# 35. Define an explicit point of no return

A mature upgrade plan should include:

```text
T0  Start maintenance
T1  Stop application writes
T2  Take/confirm final recovery point
T3  Run upgrade
T4  Validate database
T5  Start application
T6  Smoke test
T7  Declare point of no return
T8  Decommission old environment later
```

The key is not to delete the old environment immediately after a technically successful upgrade.

---

# 36. Backups before upgrades

At minimum verify:

```text
Backup exists
Backup is recent
Backup contains required WAL
Backup can actually be restored
Restore procedure is documented
Restore time meets RTO
```

For a critical production environment:

```text
Backup without a restore test
        ≠
Verified rollback plan
```

PostgreSQL itself warns that logical-replication cluster upgrades include non-transactional steps and advises taking backups before executing the sequence. ([PostgreSQL][10])

---

# 37. Post-upgrade scripts

`pg_upgrade` may generate scripts for work that must happen after the main migration.

Typical categories include:

```text
extension updates
index rebuilds
object rebuilds
statistics refresh
```

PostgreSQL warns that tables referenced by generated rebuild scripts may produce incorrect results or poor performance until the required rebuild is completed. ([PostgreSQL][5])

Therefore:

```text
pg_upgrade completed
```

does **not necessarily mean**

```text
upgrade work is finished
```

---

# 38. Statistics after pg_upgrade

PostgreSQL 18's `pg_upgrade` can transfer much of the optimizer statistics, but not all statistics are preserved.

The documentation recommends regenerating missing/current statistics, including commands such as:

```bash
vacuumdb --all --analyze-in-stages --missing-stats-only
```

followed by:

```bash
vacuumdb --all --analyze-only
```

Parallel jobs can be used where appropriate. ([PostgreSQL][5])

This matters because missing or stale statistics can produce bad execution plans immediately after an upgrade.

---

# 39. Post-upgrade performance validation

Do not only validate:

```text
Database started successfully.
```

Also validate:

```text
Application response time
Query plans
CPU
IOPS
memory
cache hit ratio
locks
deadlocks
replication lag
autovacuum
connection counts
error logs
long-running queries
checkpoint behavior
WAL generation
```

Useful PostgreSQL sources include:

```text
pg_stat_activity
pg_stat_database
pg_stat_user_tables
pg_stat_user_indexes
pg_stat_replication
pg_stat_bgwriter / related current stats
pg_stat_statements
```

PostgreSQL exposes extensive cumulative statistics specifically for monitoring database activity. ([PostgreSQL][11])

---

# 40. Application compatibility testing

Major PostgreSQL releases can introduce user-visible incompatibilities, so application testing is mandatory.

PostgreSQL explicitly recommends reviewing the **Migration** section of the target release notes and all intervening releases. ([PostgreSQL][4])

Check:

```text
SQL behavior
removed syntax
changed defaults
reserved keywords
data types
functions/operators
authentication
driver compatibility
ORM compatibility
stored procedures
PL/pgSQL
JDBC
Npgsql
psycopg
libpq
connection poolers
```

---

# 41. PostgreSQL 18-specific migration example

Suppose:

```text
Source: PostgreSQL 16
Target: PostgreSQL 18
```

You should review:

```text
PostgreSQL 17 migration notes
+
PostgreSQL 18 migration notes
```

PostgreSQL 18 contains compatibility-impacting changes, including changes related to new-cluster defaults such as data checksums. The PostgreSQL 18 release notes specifically note that `pg_upgrade` requires matching checksum configuration between the old and new clusters. ([PostgreSQL][12])

This is exactly why relying only on a generic upgrade runbook is dangerous.

---

# 42. Lifecycle management

PostgreSQL major releases have a defined support lifecycle.

The PostgreSQL Global Development Group provides support for each major version for **five years after its initial release**. After the final minor release, that major version becomes unsupported. ([PostgreSQL][1])

This should drive organizational upgrade planning.

---

# 43. Current PostgreSQL support matrix

As of **August 19, 2026**, the official PostgreSQL lifecycle table is: ([PostgreSQL][1])

| Major         | Current minor | Status    | EOL              |
| ------------- | ------------: | --------- | ---------------- |
| PostgreSQL 18 |          18.6 | Supported | Nov 14, 2030     |
| PostgreSQL 17 |         17.11 | Supported | Nov 8, 2029      |
| PostgreSQL 16 |         16.15 | Supported | Nov 9, 2028      |
| PostgreSQL 15 |         15.19 | Supported | Nov 11, 2027     |
| PostgreSQL 14 |         14.24 | Supported | **Nov 12, 2026** |
| PostgreSQL 13 |         13.23 | EOL       | Nov 13, 2025     |
| PostgreSQL 12 |         12.22 | EOL       | Nov 21, 2024     |

The immediate architectural concern in 2026 is therefore **PostgreSQL 14**. Its official EOL is only a few months away, on November 12, 2026. ([PostgreSQL][1])

---

# 44. What EOL means

After PostgreSQL reaches end-of-life:

```text
No community bug fixes
No normal security fixes
No minor releases
No official community support for that major
```

Therefore an EOL PostgreSQL instance represents:

```text
Security risk
+
Operational risk
+
Compliance risk
+
Supportability risk
```

The PostgreSQL versioning policy makes clear that after the final minor release the version is unsupported. ([PostgreSQL][1])

---

# 45. Enterprise lifecycle policy

A good internal policy could be:

```text
EOL - 18 months
      |
Architecture assessment

EOL - 12 months
      |
Upgrade project funded

EOL - 9 months
      |
Dev/test upgrades

EOL - 6 months
      |
Pre-production validation

EOL - 3 months
      |
Production upgrades complete

EOL
      |
No production systems remain
```

Do not begin planning a major upgrade a few weeks before PostgreSQL EOL.

---

# 46. PostgreSQL estate management

For large organizations, maintain a central inventory such as:

| System    | Environment | PostgreSQL | Minor | Extensions | HA  | EOL      | Owner  |
| --------- | ----------- | ---------: | ----: | ---------- | --- | -------- | ------ |
| Orders    | Prod        |         14 | 14.24 | PostGIS    | Yes | Nov-2026 | Team A |
| Billing   | Prod        |         16 | 16.15 | pgcrypto   | Yes | Nov-2028 | Team B |
| Analytics | Prod        |         17 | 17.11 | pgvector   | Yes | Nov-2029 | Team C |

Then use EOL dates to create upgrade waves.

---

# 47. Upgrade architecture decision matrix

A practical comparison:

| Requirement            | Dump/Restore | pg_upgrade      | Logical Replication |
| ---------------------- | ------------ | --------------- | ------------------- |
| Simple process         | Medium       | High            | Low                 |
| Large DB               | Poor/Medium  | Excellent       | Excellent           |
| Low downtime           | Poor         | Good            | Excellent           |
| Extra infrastructure   | Low          | Low             | High                |
| Easy target testing    | Medium       | Medium          | Excellent           |
| Cross-major version    | Yes          | Yes             | Yes                 |
| Selective migration    | Excellent    | Poor            | Excellent           |
| Fast rollback          | Medium       | Depends on mode | Potentially good    |
| Operational complexity | Medium       | Medium          | High                |

---

# 48. How I would select an upgrade method

### Small database

```text
Database = 20 GB
Maintenance window = 4 hours
```

Dump/restore may be perfectly reasonable.

---

### Large database

```text
Database = 8 TB
Maintenance window = 1 hour
```

`pg_upgrade` is normally much more suitable.

---

### Mission-critical database

```text
Database = 10 TB
Allowed downtime = <5 minutes
```

Logical replication / blue-green migration becomes a strong design candidate.

---

# 49. Blue/green PostgreSQL upgrade architecture

Example:

```text
                 Application
                     |
                  PgBouncer
                     |
        +------------+------------+
        |                         |
        v                         v
   PostgreSQL 17              PostgreSQL 18
      BLUE                       GREEN
    Current                    New target
        |
        +--------logical-------->
                replication
```

When GREEN is ready:

```text
Application
     |
     v
PostgreSQL 18 GREEN
```

The old PostgreSQL 17 environment can then remain isolated temporarily as part of the fallback strategy.

---

# 50. High-availability upgrade planning

If the database has physical standbys:

```text
Primary
   |
   +---- Standby 1
   |
   +---- Standby 2
```

the upgrade plan must address all nodes.

For certain `pg_upgrade --link` scenarios PostgreSQL documents an `rsync`-based method to upgrade physical standbys efficiently, while another option is simply to rebuild the standbys from the upgraded primary afterward. ([PostgreSQL][5])

Never assume the primary upgrade automatically upgrades the replicas.

---

# 51. Pre-upgrade technical checklist

A useful readiness review includes:

```text
PostgreSQL versions
OS version
CPU architecture
data directory
tablespaces
WAL location
database sizes
extensions
shared_preload_libraries
custom libraries
replication slots
physical standbys
logical replication
backup status
PITR status
collations
locales
checksums
authentication
SSL
connection pooler
drivers
monitoring agents
cron/jobs
ETL integrations
FDWs
```

Then run `pg_upgrade --check` when `pg_upgrade` is the selected approach. ([PostgreSQL][5])

---

# 52. Upgrade validation checklist

After upgrade:

```sql
SELECT version();
```

Check databases:

```sql
SELECT datname
FROM pg_database
ORDER BY datname;
```

Extensions:

```sql
SELECT extname, extversion
FROM pg_extension
ORDER BY extname;
```

Replication:

```sql
SELECT *
FROM pg_stat_replication;
```

Invalid indexes:

```sql
SELECT
    n.nspname,
    c.relname
FROM pg_index i
JOIN pg_class c
  ON c.oid = i.indexrelid
JOIN pg_namespace n
  ON n.oid = c.relnamespace
WHERE NOT i.indisvalid;
```

Also run application smoke tests and compare monitoring baselines.

---

# 53. Common upgrade failures

### Extension not installed

```text
pg_upgrade
   |
   X missing shared library
```

Solution:

Install the extension package built for the target PostgreSQL version.

---

### Unsupported extension version

```text
PostgreSQL 18
   |
Extension only supports PG17
```

Solution:

Upgrade/replace the extension before moving PostgreSQL.

---

### Configuration copied blindly

```text
PG14 postgresql.conf
        |
        v
PG18
```

May introduce removed, obsolete, or poorly suited parameters.

Solution:

Start from new defaults and merge reviewed custom settings.

---

### No `pg_upgrade --check`

Production upgrade discovers blockers during the maintenance window.

Solution:

Run the exact target-mode check beforehand:

```bash
pg_upgrade --check
```

or for link mode:

```bash
pg_upgrade --check --link
```

PostgreSQL specifically recommends mode-specific checks when using link, clone, copy-file-range, or swap transfer modes. ([PostgreSQL][5])

---

### Immediate deletion of old cluster

Bad pattern:

```text
Upgrade successful
     |
rm -rf old_cluster
```

Better:

```text
Upgrade
   |
Validation
   |
Stabilization period
   |
Backup verification
   |
Formal approval
   |
Old cluster removal
```

PostgreSQL's own `pg_upgrade` workflow places old-cluster deletion only after you are satisfied with the upgraded environment. ([PostgreSQL][5])

---

# 54. Minor versus major upgrade — interview answer

A concise way to explain it:

> A PostgreSQL minor upgrade stays within the same major release, for example 18.4 to 18.6. It mainly provides bug and security fixes and normally only requires replacing the binaries and restarting PostgreSQL. A major upgrade, such as PostgreSQL 17 to 18, can change internal catalogs and compatibility, so the database must be migrated using `pg_upgrade`, logical dump/restore, or logical replication. Major upgrades also require application, extension, configuration, and rollback validation. ([PostgreSQL][1])

---

# 55. pg_upgrade — interview answer

> `pg_upgrade` is PostgreSQL's utility for rapidly migrating an existing cluster to a newer major PostgreSQL version without performing a full logical dump and restore. It creates the target-version system catalogs and transfers or reuses compatible user data files. Before production execution I run `pg_upgrade --check`, verify extension binaries, backups and configuration compatibility, perform the upgrade, execute any generated post-upgrade scripts, refresh statistics, validate the application, and keep the old environment until the rollback window has passed. ([PostgreSQL][5])

---

# 56. Logical upgrade — interview answer

> For very large PostgreSQL databases where downtime must be minimal, I can build a new cluster on the target PostgreSQL version and use logical replication from the old system to the new system. Initial data is synchronized while production remains online. At cutover, I stop writes, allow replication to catch up, redirect applications to the new server, validate, and then retire the old environment after the rollback window. PostgreSQL supports logical replication across major versions, making this a useful blue/green major-upgrade strategy. ([PostgreSQL][4])

---

# 57. Most important architecture principles

Remember these ten points:

1. **Minor upgrade ≠ major upgrade.**
2. Keep each major version on its **latest minor release**. ([PostgreSQL][1])
3. Major upgrades require migration using **dump/restore, `pg_upgrade`, or logical replication**. ([PostgreSQL][4])
4. Always run **`pg_upgrade --check`** before a `pg_upgrade` production change. ([PostgreSQL][5])
5. Treat **extensions and native shared libraries** as first-class compatibility dependencies. ([PostgreSQL][5])
6. Understand the rollback difference between **copy, link, clone, and swap modes**. ([PostgreSQL][5])
7. Review the migration notes for the target version **and all skipped major releases**. ([PostgreSQL][1])
8. Regenerate/validate statistics and query performance after the upgrade. ([PostgreSQL][5])
9. Maintain tested backups and define a clear **point of no return**.
10. Start major-upgrade projects well before EOL; currently PostgreSQL **14 reaches EOL November 12, 2026**. ([PostgreSQL][1])

### Current official documentation verified

The main PostgreSQL official sources I checked were the **PostgreSQL 18 upgrading documentation, `pg_upgrade` reference, logical-replication upgrade documentation, extension documentation, release notes, and PostgreSQL versioning/EOL policy**. PostgreSQL 18.6 was released on **August 13, 2026**, and PostgreSQL 19 is still a development/beta release rather than the current production major release. ([PostgreSQL][5])

[1]: https://www.postgresql.org/support/versioning/ "PostgreSQL: Versioning Policy"
[2]: https://www.postgresql.org/docs/release/18.6/?utm_source=chatgpt.com "PostgreSQL: Release Notes"
[3]: https://www.postgresql.org/about/news/postgresql-186-1711-1615-1519-1424-and-19-beta-3-released-3365/?utm_source=chatgpt.com "PostgreSQL 18.6, 17.11, 16.15, 15.19, 14.24 and 19 Beta 3 ..."
[4]: https://www.postgresql.org/docs/current/upgrading.html "PostgreSQL: Documentation: 18: 18.6. Upgrading a PostgreSQL Cluster"
[5]: https://www.postgresql.org/docs/current/pgupgrade.html "PostgreSQL: Documentation: 18: pg_upgrade"
[6]: https://www.postgresql.org/docs/current/catalog-pg-extension.html?utm_source=chatgpt.com "Documentation: 18: 52.22. pg_extension"
[7]: https://www.postgresql.org/docs/current/sql-createextension.html "PostgreSQL: Documentation: 18: CREATE EXTENSION"
[8]: https://www.postgresql.org/docs/current/sql-alterextension.html "PostgreSQL: Documentation: 18: ALTER EXTENSION"
[9]: https://www.postgresql.org/docs/current/logical-replication-restrictions.html?utm_source=chatgpt.com "Documentation: 18: 29.8. Restrictions"
[10]: https://www.postgresql.org/docs/current/logical-replication-upgrade.html "PostgreSQL: Documentation: 18: 29.13. Upgrade"
[11]: https://www.postgresql.org/docs/current/monitoring-stats.html?utm_source=chatgpt.com "Documentation: 18: 27.2. The Cumulative Statistics System"
[12]: https://www.postgresql.org/docs/current/release-18.html?utm_source=chatgpt.com "Documentation: 18: E.6. Release 18"
