# Topic 2 — PostgreSQL 18 Database Administration

PostgreSQL Database Administration is the day-to-day management of databases, roles, permissions, schemas, sessions, tablespaces, configuration, and routine maintenance. PostgreSQL 18 treats database management and role management as core parts of Server Administration. ([PostgreSQL][1])

A useful way to view PostgreSQL administration is:

```text
PostgreSQL Cluster
      │
      ├── Databases
      │      ├── Schemas
      │      ├── Tables
      │      ├── Indexes
      │      ├── Views
      │      └── Functions
      │
      ├── Roles / Users
      │
      ├── Privileges
      │
      ├── Tablespaces
      │
      ├── Sessions
      │
      └── Maintenance
```

PostgreSQL defines a running server instance as managing one or more databases, with the database being the highest SQL-object organizational level inside that instance. ([PostgreSQL][1])

---

## 1. PostgreSQL Administration Scope

A PostgreSQL DBA commonly manages:

```text
Database creation/deletion
Role creation and management
Privileges and ownership
Schemas
Tablespaces
Connections and sessions
Object ownership
Database-level configuration
Maintenance operations
Vacuum and Analyze
Reindexing
Log management
Capacity and space usage
Operational health checks
```

These responsibilities are different from OS-level administration.

For example:

```text
Linux Administrator
       │
       ├── CPU
       ├── Memory
       ├── Filesystem
       └── OS services

PostgreSQL DBA
       │
       ├── Databases
       ├── Roles
       ├── Privileges
       ├── Tablespaces
       ├── Sessions
       └── Maintenance
```

---

# 2. Understanding the PostgreSQL Object Hierarchy

A DBA should understand how PostgreSQL objects relate to each other.

```text
PostgreSQL Cluster / Instance
             │
       ┌─────┴──────┐
       ▼            ▼
   Database A    Database B
       │
       ▼
     Schema
       │
  ┌────┼───────────┐
  ▼    ▼           ▼
Table Index      Function
```

A key point is that a PostgreSQL connection is made to a **specific database**.

For example:

```bash
psql -h dbserver -p 5432 -U appuser -d appdb
```

Once connected to `appdb`, you cannot directly query a normal table in another database using a simple three-part database reference in the same way some other database products support.

---

# 3. Listing Databases

From `psql`:

```sql
\l
```

or:

```sql
\list
```

Using SQL:

```sql
SELECT
    datname,
    datdba::regrole AS owner,
    encoding,
    datcollate,
    datctype,
    datallowconn
FROM pg_database
ORDER BY datname;
```

This lets the DBA identify:

```text
Database name
Owner
Encoding
Locale
Whether connections are allowed
```

---

# 4. Creating a Database

The basic command is:

```sql
CREATE DATABASE appdb;
```

PostgreSQL also provides the `createdb` utility.

Example:

```bash
createdb appdb
```

A more controlled database creation might look like:

```sql
CREATE DATABASE appdb
    OWNER app_owner
    ENCODING 'UTF8';
```

The PostgreSQL 18 documentation groups database creation, templates, database configuration, deletion, and tablespaces under the Managing Databases chapter. ([PostgreSQL][1])

---

# 5. Database Ownership

Every database has an owner.

Example:

```sql
CREATE DATABASE payroll
OWNER payroll_owner;
```

Check:

```sql
SELECT
    datname,
    datdba::regrole
FROM pg_database;
```

Database ownership is important because the owner has administrative control over that database.

A production design should usually avoid having application login users directly own every object.

A better model is:

```text
Application Login
       │
       ▼
Application Runtime Role
       │
       ▼
Privileges
       │
       ▼
Objects owned by
Dedicated Owner Role
```

For example:

```text
payroll_owner     → NOLOGIN
payroll_rw        → NOLOGIN
payroll_ro        → NOLOGIN
payroll_app       → LOGIN
```

Then:

```sql
GRANT payroll_rw TO payroll_app;
```

This separates ownership from login identity.

---

# 6. PostgreSQL Roles

PostgreSQL uses **roles** for access control.

A role can behave as:

```text
User
Group
Owner
Administrative role
Application account
```

PostgreSQL does not internally maintain separate concepts of users and groups. Roles cover both use cases, and roles can own objects and be members of other roles. ([PostgreSQL][2])

Example:

```sql
CREATE ROLE app_user LOGIN;
```

A group-style role:

```sql
CREATE ROLE app_readonly NOLOGIN;
```

Then:

```sql
GRANT app_readonly TO app_user;
```

Architecture:

```text
app_user
   │
   ▼
app_readonly
   │
   ▼
SELECT privileges
```

---

# 7. LOGIN vs NOLOGIN Roles

A useful enterprise pattern is to distinguish login roles from privilege roles.

### Login role

```sql
CREATE ROLE john LOGIN;
```

Can authenticate to PostgreSQL.

### NOLOGIN role

```sql
CREATE ROLE finance_readonly NOLOGIN;
```

Cannot directly log in.

Instead:

```sql
GRANT finance_readonly TO john;
```

Now:

```text
john
 │
 └── finance_readonly
           │
           └── SELECT permission
```

This makes access management easier.

---

# 8. Role Attributes

PostgreSQL roles can have administrative attributes such as:

```text
LOGIN
SUPERUSER
CREATEDB
CREATEROLE
REPLICATION
BYPASSRLS
```

For example:

```sql
CREATE ROLE dba_admin
LOGIN
CREATEDB
CREATEROLE;
```

A full superuser role:

```sql
CREATE ROLE admin_user
LOGIN
SUPERUSER;
```

`SUPERUSER` should be highly restricted because it effectively bypasses normal permission controls.

PostgreSQL 18 documents role attributes and role membership as separate role-management concepts. ([PostgreSQL][2])

---

# 9. Role Membership

Example:

```sql
CREATE ROLE app_read;
CREATE ROLE app_write;
```

Then:

```sql
GRANT app_read TO developer1;
GRANT app_write TO developer2;
```

Conceptually:

```text
              app_read
                ▲
                │
          developer1

              app_write
                ▲
                │
          developer2
```

Multiple users can share a privilege role:

```text
                 reporting_read
                 /      |      \
                /       |       \
             user1    user2    user3
```

This is easier to manage than granting individual table permissions repeatedly.

---

# 10. Recommended Enterprise Role Model

A practical architecture can look like:

```text
Database
   │
   ├── app_owner
   │      NOLOGIN
   │
   ├── app_read
   │      NOLOGIN
   │
   ├── app_write
   │      NOLOGIN
   │
   ├── app_admin
   │      NOLOGIN
   │
   ├── application_service
   │      LOGIN
   │      └── member of app_write
   │
   └── reporting_user
          LOGIN
          └── member of app_read
```

This improves:

```text
Least privilege
Auditability
Credential rotation
Access reviews
Offboarding
Permission management
```

---

# 11. Creating Login Roles

Example:

```sql
CREATE ROLE app_service
LOGIN
PASSWORD 'temporary-password';
```

A better operational model is to manage credentials through your approved secret-management process rather than embedding passwords in scripts.

Check roles:

```sql
\du
```

Or:

```sql
SELECT
    rolname,
    rolcanlogin,
    rolsuper,
    rolcreatedb,
    rolcreaterole,
    rolreplication
FROM pg_roles
ORDER BY rolname;
```

---

# 12. Changing Roles

Example:

```sql
ALTER ROLE app_service PASSWORD 'new-password';
```

Disable login:

```sql
ALTER ROLE app_service NOLOGIN;
```

Enable login:

```sql
ALTER ROLE app_service LOGIN;
```

Change connection limit:

```sql
ALTER ROLE app_service CONNECTION LIMIT 20;
```

---

# 13. Dropping Roles

Example:

```sql
DROP ROLE old_user;
```

But PostgreSQL may reject the operation if the role still owns objects or has dependent privileges.

The normal administrative workflow is:

```text
Identify ownership
      │
      ▼
Reassign owned objects
      │
      ▼
Remove privileges
      │
      ▼
Drop role
```

Typical commands include:

```sql
REASSIGN OWNED BY old_user TO app_owner;
```

and:

```sql
DROP OWNED BY old_user;
```

These commands need to be used carefully because their effect is database-specific and can affect privileges and objects.

---

# 14. Schemas

A PostgreSQL database can contain multiple schemas.

```text
Database: appdb
      │
      ├── public
      ├── finance
      ├── hr
      └── reporting
```

A schema acts as a namespace.

For example:

```sql
finance.transactions
```

and:

```sql
archive.transactions
```

can both exist because they are in different schemas.

---

# 15. Creating Schemas

Example:

```sql
CREATE SCHEMA finance;
```

With an owner:

```sql
CREATE SCHEMA finance
AUTHORIZATION finance_owner;
```

List schemas:

```sql
\dn
```

SQL:

```sql
SELECT
    nspname,
    nspowner::regrole
FROM pg_namespace
ORDER BY nspname;
```

---

# 16. Schema Privileges

Important schema privileges are:

```text
USAGE
CREATE
```

For example:

```sql
GRANT USAGE ON SCHEMA finance TO finance_read;
```

This lets the role access objects in the schema, provided it also has the necessary privileges on those objects.

To allow object creation:

```sql
GRANT CREATE ON SCHEMA finance TO developer;
```

Architecture:

```text
Database CONNECT
      │
      ▼
Schema USAGE
      │
      ▼
Table SELECT
```

All three layers can matter.

---

# 17. Understanding PostgreSQL Privileges

Privileges are granted using:

```sql
GRANT
```

and removed using:

```sql
REVOKE
```

Common object privileges include:

```text
SELECT
INSERT
UPDATE
DELETE
TRUNCATE
REFERENCES
TRIGGER
EXECUTE
USAGE
CREATE
CONNECT
TEMPORARY
```

Example:

```sql
GRANT SELECT ON finance.transactions TO finance_read;
```

Write access:

```sql
GRANT INSERT, UPDATE, DELETE
ON finance.transactions
TO finance_write;
```

---

# 18. Database-Level Privileges

Common database-level privileges include:

```text
CONNECT
CREATE
TEMPORARY
```

Example:

```sql
GRANT CONNECT ON DATABASE appdb TO app_user;
```

To prevent a role connecting:

```sql
REVOKE CONNECT ON DATABASE appdb FROM app_user;
```

Be aware that existing sessions are not necessarily removed simply because future `CONNECT` permission is revoked.

---

# 19. Granting Privileges on All Tables

Example:

```sql
GRANT SELECT
ON ALL TABLES IN SCHEMA finance
TO finance_read;
```

For sequences:

```sql
GRANT USAGE, SELECT
ON ALL SEQUENCES IN SCHEMA finance
TO finance_read;
```

But there is an important DBA issue here:

**This only applies to objects that already exist.**

New tables created later require either new grants or properly configured default privileges.

---

# 20. Default Privileges

This is one of the most important administrative concepts.

Example:

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE finance_owner
IN SCHEMA finance
GRANT SELECT ON TABLES TO finance_read;
```

Now future tables created by `finance_owner` in that schema can automatically inherit the intended privileges.

The important detail is:

```text
Default privileges depend on
the role that creates the object.
```

This is a common source of production permission problems.

---

# 21. Object Ownership

Check table ownership:

```sql
SELECT
    schemaname,
    tablename,
    tableowner
FROM pg_tables
WHERE schemaname NOT IN
('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;
```

Change owner:

```sql
ALTER TABLE finance.transactions
OWNER TO finance_owner;
```

The DBA should avoid random ownership such as:

```text
developer1 owns table A
developer2 owns table B
admin owns table C
application user owns table D
```

Instead:

```text
finance_owner owns finance objects
```

gives cleaner governance.

---

# 22. Template Databases

A newly initialized PostgreSQL cluster includes:

```text
template0
template1
postgres
```

`template1` is normally used as the source for newly created databases.

Conceptually:

```text
template1
    │
    ▼
CREATE DATABASE
    │
    ▼
new_database
```

If standardized objects or extensions are intentionally added to `template1`, newly created databases can inherit them.

However, modifying template databases should be controlled carefully.

---

# 23. Database-Level Configuration

PostgreSQL allows configuration settings to be associated with a particular database.

Example:

```sql
ALTER DATABASE appdb
SET statement_timeout = '30s';
```

Check:

```sql
SELECT
    datname,
    datconfig
FROM pg_database;
```

A role can also have specific defaults.

```sql
ALTER ROLE app_user
SET statement_timeout = '15s';
```

This provides several configuration levels:

```text
postgresql.conf
      │
      ▼
Database setting
      │
      ▼
Role setting
      │
      ▼
Session setting
      │
      ▼
Transaction/query behavior
```

The effective value depends on PostgreSQL's configuration precedence.

---

# 24. Tablespaces

PostgreSQL tablespaces allow DBAs to define filesystem locations where database object files can be stored. They are useful for disk-layout control and can help place different workloads on different storage. ([PostgreSQL][3])

Architecture:

```text
PostgreSQL
    │
    ├── pg_default
    │      └── Standard storage
    │
    └── fastspace
           └── SSD/NVMe storage
```

Example:

```sql
CREATE TABLESPACE fastspace
LOCATION '/ssd1/postgresql/data';
```

The directory must already exist, be empty, and be owned by the PostgreSQL operating-system user. ([PostgreSQL][3])

---

# 25. Built-in Tablespaces

PostgreSQL initializes two tablespaces:

```text
pg_default
pg_global
```

`pg_global` is used for shared system catalogs, while `pg_default` is the standard default tablespace for the template databases and therefore normally for newly created databases. ([PostgreSQL][3])

List tablespaces:

```sql
\db
```

or:

```sql
SELECT
    spcname,
    spcowner::regrole,
    pg_tablespace_location(oid)
FROM pg_tablespace;
```

This query is also shown in the PostgreSQL 18 documentation. ([PostgreSQL][3])

---

# 26. Creating Objects in a Tablespace

Example:

```sql
CREATE TABLE orders
(
    order_id bigint,
    order_date date
)
TABLESPACE fastspace;
```

Indexes can also be placed separately:

```sql
CREATE INDEX idx_orders_date
ON orders(order_date)
TABLESPACE fastspace;
```

A common architecture could be:

```text
Large table
    │
    └── general storage

High-use index
    │
    └── faster storage
```

PostgreSQL specifically describes tablespaces as useful for placing heavily used objects on faster storage and less critical objects on lower-cost storage. ([PostgreSQL][3])

---

# 27. Important Tablespace Warning

Tablespaces are **not independent storage units**.

Although they physically live outside the main data directory, PostgreSQL considers them part of the same cluster. They depend on cluster metadata and cannot simply be detached and attached to another cluster or backed up independently. Losing a tablespace can make the cluster unusable. ([PostgreSQL][3])

Therefore:

```text
Do NOT treat tablespace backup
       as
independent database backup
```

A tablespace must be protected as part of the PostgreSQL backup/recovery architecture.

---

# 28. Session Administration

One of the DBA's most common tasks is understanding active database sessions.

The main view is:

```sql
pg_stat_activity
```

Example:

```sql
SELECT
    pid,
    usename,
    datname,
    application_name,
    client_addr,
    state,
    query_start,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
ORDER BY query_start;
```

This helps answer:

```text
Who is connected?
Which database?
From which application?
From which host?
What query is running?
How long?
Is it waiting?
```

---

# 29. Session States

Common states include:

```text
active
idle
idle in transaction
idle in transaction (aborted)
```

`idle in transaction` is particularly important.

Example:

```text
BEGIN
  │
  ▼
UPDATE...
  │
  ▼
Application stops doing work
  │
  ▼
Transaction remains open
```

Long-running open transactions can contribute to operational issues such as preventing cleanup of older row versions.

---

# 30. Finding Long-Running Queries

Example:

```sql
SELECT
    pid,
    usename,
    datname,
    now() - query_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND pid <> pg_backend_pid()
ORDER BY duration DESC;
```

This is a common DBA health-check query.

---

# 31. Cancelling a Query

Use:

```sql
SELECT pg_cancel_backend(pid);
```

Example:

```sql
SELECT pg_cancel_backend(12345);
```

This requests cancellation of the currently running query without necessarily terminating the whole database session.

---

# 32. Terminating a Session

Use:

```sql
SELECT pg_terminate_backend(pid);
```

Example:

```sql
SELECT pg_terminate_backend(12345);
```

Conceptually:

```text
pg_cancel_backend
      │
      └── Cancel current query

pg_terminate_backend
      │
      └── Terminate session
```

Termination should be used carefully because open transactions will be rolled back.

---

# 33. Connection Count

Check total connections:

```sql
SELECT count(*)
FROM pg_stat_activity;
```

By database:

```sql
SELECT
    datname,
    count(*)
FROM pg_stat_activity
GROUP BY datname
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

This is useful when diagnosing connection exhaustion.

---

# 34. Lock Administration

DBAs should know how to identify locks and blocking sessions.

Important views include:

```text
pg_locks
pg_stat_activity
```

A simple investigation might use:

```sql
SELECT
    pid,
    usename,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

A deeper blocking analysis typically combines `pg_stat_activity`, `pg_locks`, and helper functions such as `pg_blocking_pids()`.

Example:

```sql
SELECT
    pid,
    pg_blocking_pids(pid) AS blocking_pids,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

---

# 35. Database Size Administration

Database size:

```sql
SELECT pg_size_pretty(
    pg_database_size('appdb')
);
```

All databases:

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

Table size:

```sql
SELECT
    pg_size_pretty(
        pg_total_relation_size('finance.transactions')
    );
```

---

# 36. Table and Index Size

Example:

```sql
SELECT
    pg_size_pretty(
        pg_relation_size('finance.transactions')
    ) AS table_size;
```

Total size including indexes:

```sql
SELECT
    pg_size_pretty(
        pg_total_relation_size('finance.transactions')
    ) AS total_size;
```

Largest tables:

```sql
SELECT
    schemaname,
    relname,
    pg_size_pretty(
        pg_total_relation_size(relid)
    ) AS total_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

Size monitoring should be part of capacity planning.

---

# 37. VACUUM Administration

PostgreSQL's MVCC architecture means database maintenance includes vacuuming.

Basic:

```sql
VACUUM;
```

Specific table:

```sql
VACUUM finance.transactions;
```

With planner statistics:

```sql
VACUUM ANALYZE finance.transactions;
```

PostgreSQL identifies routine vacuuming, updating planner statistics, preventing transaction-ID wraparound, and autovacuum as core maintenance concerns. ([PostgreSQL][4])

We should cover this much more deeply under the dedicated **Vacuum & Autovacuum** topic.

---

# 38. ANALYZE

`ANALYZE` gathers statistics that PostgreSQL's query planner uses.

```sql
ANALYZE;
```

Specific table:

```sql
ANALYZE finance.transactions;
```

Conceptually:

```text
Table data
   │
   ▼
ANALYZE
   │
   ▼
Planner statistics
   │
   ▼
Query planner
   │
   ▼
Execution plan
```

PostgreSQL describes planner-statistics maintenance as closely related to vacuum operations. ([PostgreSQL][4])

---

# 39. Autovacuum

PostgreSQL includes an autovacuum subsystem to automate:

```text
VACUUM
ANALYZE
transaction-ID maintenance
```

This is not something a DBA should normally disable globally.

Instead, the DBA should monitor and tune it based on workload.

A deeper discussion belongs in Topic 8 — **Vacuum & Autovacuum**.

---

# 40. REINDEX

Indexes can occasionally require rebuilding.

Example:

```sql
REINDEX INDEX idx_orders_date;
```

Table:

```sql
REINDEX TABLE orders;
```

Database:

```sql
REINDEX DATABASE appdb;
```

PostgreSQL lists routine reindexing as one of the normal database-maintenance areas. ([PostgreSQL][4])

For production systems, locking impact and whether concurrent reindexing is appropriate must be considered.

---

# 41. Log File Maintenance

Logs should not simply grow indefinitely.

Operational design should address:

```text
Log destination
Log rotation
Retention
Disk capacity
Centralized log collection
Monitoring
Security
```

PostgreSQL explicitly lists log file management as one of the routine maintenance areas. ([PostgreSQL][4])

---

# 42. Backups Are Part of DBA Operations

Backup architecture is a separate major topic, but administration must verify that backups actually run.

PostgreSQL's maintenance documentation emphasizes that regular backups are essential because without a recent backup recovery after catastrophic failure or destructive human error may not be possible. ([PostgreSQL][4])

A DBA should routinely verify:

```text
Backup completed
WAL archive working
Retention correct
Backup storage healthy
Restore test performed
Recovery documentation current
```

A successful backup job alone is not proof of recoverability.

---

# 43. Routine DBA Health Checks

A useful operational framework is:

```text
Daily
 │
 ├── Database availability
 ├── Failed connections
 ├── Long queries
 ├── Blocking sessions
 ├── Replication health
 ├── Backup status
 ├── Disk utilization
 └── Error logs

Weekly
 │
 ├── Database growth
 ├── Table/index growth
 ├── Vacuum health
 ├── Long transactions
 ├── Index utilization
 └── Capacity trends

Monthly
 │
 ├── Access review
 ├── Privilege review
 ├── Restore validation
 ├── Capacity forecast
 ├── Configuration review
 └── Patch/release review
```

The exact schedule should be adapted to business criticality and monitoring automation.

---

# 44. Useful PostgreSQL Administration Catalogs and Views

Every PostgreSQL DBA should become familiar with:

| Object                 | Purpose                   |
| ---------------------- | ------------------------- |
| `pg_database`          | Databases                 |
| `pg_roles`             | Roles                     |
| `pg_namespace`         | Schemas                   |
| `pg_tables`            | Tables                    |
| `pg_indexes`           | Indexes                   |
| `pg_tablespace`        | Tablespaces               |
| `pg_stat_activity`     | Sessions/activity         |
| `pg_locks`             | Locks                     |
| `pg_settings`          | Runtime configuration     |
| `pg_stat_database`     | Database-level statistics |
| `pg_stat_user_tables`  | Table activity            |
| `pg_stat_user_indexes` | Index usage               |
| `pg_stat_replication`  | Replication activity      |

These system catalogs and statistics views are foundational to both manual troubleshooting and automated monitoring.

---

# 45. Useful `psql` Administrative Commands

A DBA should know these shortcuts:

```text
\l           List databases
\c dbname    Connect to database
\du          List roles
\dn          List schemas
\dt          List tables
\di          List indexes
\db          List tablespaces
\df          List functions
\d table     Describe object
\x           Expanded display
\conninfo    Current connection
```

For example:

```sql
\c appdb
```

then:

```sql
\dt finance.*
```

---

# 46. Production Database Administration Model

For enterprise PostgreSQL, I would recommend separating responsibilities like this:

```text
                PostgreSQL Administration
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
      Security        Operations        Capacity
         │                │                │
       Roles          Sessions           Storage
     Privileges        Locks             Growth
      Schemas          Vacuum           Tablespaces
    Ownership          Analyze            Sizing
         │                │                │
         └──────────┬─────┴────────────────┘
                    ▼
               Monitoring
                    │
                    ▼
            Automation / Alerts
```

---

# 47. Production DBA Best Practices

I would standardize the following:

```text
Use dedicated owner roles
Use group/privilege roles
Keep application logins separate
Apply least privilege
Use default privileges carefully
Avoid unnecessary superusers
Monitor long-running transactions
Monitor blocking sessions
Monitor connection consumption
Monitor database/table growth
Do not disable autovacuum globally
Protect all tablespaces
Automate routine health checks
Verify backups through restore testing
Document administrative changes
```

---

# 48. Example Enterprise Role Architecture

A clean application setup could be:

```text
Database: ordersdb

orders_owner
   │
   ├── owns schema orders
   ├── owns tables
   ├── owns sequences
   └── NOLOGIN

orders_read
   │
   ├── USAGE on orders schema
   └── SELECT on tables

orders_write
   │
   ├── inherits/receives required read access
   ├── INSERT
   ├── UPDATE
   ├── DELETE
   └── sequence privileges

orders_app
   │
   ├── LOGIN
   └── member of orders_write

orders_reporting
   │
   ├── LOGIN
   └── member of orders_read
```

That is much easier to govern than giving every user direct permissions.

---

# 49. DBA Workflow for Creating a New Application Database

An enterprise provisioning flow could be:

```text
1. Confirm database requirements
          │
          ▼
2. Create owner role
          │
          ▼
3. Create database
          │
          ▼
4. Create application schemas
          │
          ▼
5. Create read/write privilege roles
          │
          ▼
6. Configure grants
          │
          ▼
7. Configure default privileges
          │
          ▼
8. Create application login
          │
          ▼
9. Configure database/role parameters
          │
          ▼
10. Configure connection/security rules
          │
          ▼
11. Configure monitoring
          │
          ▼
12. Confirm backup coverage
          │
          ▼
13. Perform connectivity test
          │
          ▼
14. Capture configuration/documentation
```

---

# 50. DBA Administration Checklist

| Area               | DBA Validation                        |
| ------------------ | ------------------------------------- |
| Database           | Correct database exists               |
| Database owner     | Dedicated owner role                  |
| Roles              | Login and privilege roles separated   |
| Superuser          | Restricted                            |
| Schemas            | Appropriate schema structure          |
| Ownership          | Consistent object ownership           |
| Privileges         | Least privilege                       |
| Default privileges | Configured for future objects         |
| Connections        | Monitored and sized                   |
| Sessions           | Long/idle transactions monitored      |
| Locks              | Blocking monitored                    |
| Tablespaces        | Storage and backup coverage validated |
| Disk usage         | Capacity monitored                    |
| Vacuum             | Autovacuum healthy                    |
| Statistics         | ANALYZE occurring                     |
| Indexes            | Usage/health reviewed                 |
| Logs               | Rotated and monitored                 |
| Backups            | Verified                              |
| Recovery           | Restore testing performed             |
| Monitoring         | Alerts configured                     |
| Documentation      | Operational baseline maintained       |

---

## PostgreSQL 18 official references

For this topic, the main official PostgreSQL 18 sections are:

**Managing Databases — Chapter 22** explains database creation, template databases, per-database configuration, deletion, and tablespaces. ([PostgreSQL][1])

**Database Roles — Chapter 21** covers roles, attributes, membership, dropping roles, predefined roles, and function security. ([PostgreSQL][2])

**Tablespaces — Section 22.6** explains storage locations, tablespace creation, built-in tablespaces, privileges, and the important dependency between tablespaces and the main cluster. ([PostgreSQL][3])

**Routine Database Maintenance — Chapter 24** covers vacuuming, planner statistics, transaction-ID protection, autovacuum, reindexing, and log maintenance. ([PostgreSQL][4])

For your documentation sequence, **Topic 3 should be PostgreSQL 18 Monitoring & Observability**. That would naturally follow this administration topic and can go deep into `pg_stat_activity`, `pg_stat_database`, `pg_stat_statements`, wait events, locks, logging, OS metrics, Prometheus/Grafana architecture, alert thresholds, and a production monitoring dashboard design.

[1]: https://www.postgresql.org/docs/18/managing-databases.html "PostgreSQL: Documentation: 18: Chapter 22. Managing Databases"
[2]: https://www.postgresql.org/docs/18/user-manag.html "PostgreSQL: Documentation: 18: Chapter 21. Database Roles"
[3]: https://www.postgresql.org/docs/18/manage-ag-tablespaces.html "PostgreSQL: Documentation: 18: 22.6. Tablespaces"
[4]: https://www.postgresql.org/docs/18/maintenance.html "PostgreSQL: Documentation: 18: Chapter 24. Routine Database Maintenance Tasks"
