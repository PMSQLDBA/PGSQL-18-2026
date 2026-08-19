## Topic 22 — PostgreSQL Extensions

PostgreSQL extensions are packages that add features to PostgreSQL without changing the core database server. 

An extension can add functions, data types, operators, indexes, views, background workers, or even major capabilities such as geospatial processing. 

PostgreSQL manages the objects belonging to an extension as one logical unit. ([PostgreSQL][1])

### 1. Extension architecture

At a high level:

```text
PostgreSQL Server
       |
       +-- Core PostgreSQL
       |
       +-- Extensions
             |
             +-- pg_stat_statements
             +-- PostGIS
             +-- pg_cron
             +-- pgcrypto
             +-- uuid-ossp
             +-- custom extensions
```

An extension normally contains:

```text
myextension.control
myextension--1.0.sql
myextension--1.0--1.1.sql
optional shared library (.so/.dll)
```

The `.control` file contains metadata such as the default version, dependencies, schema behavior, and whether superuser privileges are required. 

PostgreSQL expects extension control and installation scripts in its extension directories. ([PostgreSQL][2])

For example:

```text
my_extension.control
```

might conceptually contain:

```ini
default_version = '1.0'
comment = 'Example PostgreSQL extension'
relocatable = true
```

and:

```text
my_extension--1.0.sql
```

could contain:

```sql
CREATE FUNCTION hello()
RETURNS text
LANGUAGE SQL
AS $$
    SELECT 'Hello PostgreSQL';
$$;
```

Then PostgreSQL can manage these objects together as the `my_extension` extension.

---

## 2. `CREATE EXTENSION`

The main command is:

```sql
CREATE EXTENSION extension_name;
```

For example:

```sql
CREATE EXTENSION pg_stat_statements;
```

or:

```sql
CREATE EXTENSION postgis;
```

`CREATE EXTENSION` executes the extension's installation script and records which database objects belong to that extension. 

This allows PostgreSQL to later upgrade or remove the package as a unit. ([PostgreSQL][1])

Useful variations include:

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

Specify a schema:

```sql
CREATE EXTENSION hstore
SCHEMA extensions;
```

Specify a version:

```sql
CREATE EXTENSION some_extension
VERSION '1.2';
```

Check extensions available on the server:

```sql
SELECT *
FROM pg_available_extensions;
```

Check installed extensions:

```sql
SELECT
    extname,
    extversion
FROM pg_extension;
```

A very useful `psql` command is:

```text
\dx
```

which displays installed extensions.

---

## 3. Installing vs enabling

This distinction is important.

Installing extension software on the **operating system/server** is different from enabling it inside a **database**.

For example:

```text
Linux server
    |
    | OS package installs PostGIS files
    v
PostgreSQL installation
    |
    | CREATE EXTENSION postgis
    v
Individual database
```

An extension must normally exist on the PostgreSQL server before:

```sql
CREATE EXTENSION postgis;
```

can work.

Also, extensions are generally enabled **per database**. PostgreSQL tracks an extension within a database rather than as one cluster-wide extension object. ([PostgreSQL][2])

So:

```text
Database A -> PostGIS enabled
Database B -> PostGIS not enabled
Database C -> PostGIS enabled
```

is completely valid.

---

# 4. `pg_stat_statements`

`pg_stat_statements` is one of the most important PostgreSQL performance-monitoring extensions.

It tracks execution statistics for SQL statements processed by PostgreSQL. ([PostgreSQL][3])

Typical questions it helps answer:

```text
Which queries consume the most time?
Which queries execute most frequently?
Which queries have high average latency?
Which queries read the most blocks?
Which SQL statements are candidates for tuning?
```

### Configuration

`pg_stat_statements` requires server-level configuration because it uses shared memory.

Typically:

```ini
shared_preload_libraries = 'pg_stat_statements'
```

A PostgreSQL restart is required when adding or removing it from `shared_preload_libraries`. 

Current PostgreSQL documentation also requires query identifiers to be calculated; `compute_query_id = auto` or `on` can provide that. ([PostgreSQL][3])

Then inside the database:

```sql
CREATE EXTENSION pg_stat_statements;
```

Important architectural point:

```text
shared_preload_libraries
        |
        v
Statistics collection
server-wide
        |
        +---- Database A
        |
        +---- Database B
        |
        +---- Database C

CREATE EXTENSION pg_stat_statements
creates the SQL views/functions
needed to access those statistics
inside a database.
```

---

## 5. Finding expensive queries

Example:

```sql
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

This answers:

> Which SQL statements consumed the most total execution time?

Another useful query:

```sql
SELECT
    query,
    calls,
    mean_exec_time
FROM pg_stat_statements
WHERE calls > 100
ORDER BY mean_exec_time DESC
LIMIT 10;
```

This can identify frequently executed queries that are individually slow.

You can also reset statistics:

```sql
SELECT pg_stat_statements_reset();
```

Be careful doing this in production because monitoring systems may rely on cumulative statistics.

---

# 6. PostGIS

PostGIS turns PostgreSQL into a geospatial database.

It adds support for storing, indexing, and querying geographical and geometric information. ([PostGIS][4])

Enable it with:

```sql
CREATE EXTENSION postgis;
```

which is the official recommended method after the PostGIS software has been installed on the PostgreSQL server. ([PostGIS][5])

PostGIS adds types such as:

```text
geometry
geography
```

For example:

```sql
CREATE TABLE stores (
    store_id bigint,
    name text,
    location geography(Point, 4326)
);
```

Insert a location:

```sql
INSERT INTO stores
VALUES (
    1,
    'Chicago Store',
    ST_SetSRID(
        ST_MakePoint(-87.6298, 41.8781),
        4326
    )
);
```

---

## 7. Spatial queries

Suppose you want locations within approximately 10 km of a point:

```sql
SELECT
    store_id,
    name
FROM stores
WHERE ST_DWithin(
    location,
    ST_SetSRID(
        ST_MakePoint(-87.6298, 41.8781),
        4326
    )::geography,
    10000
);
```

PostGIS is commonly used for:

```text
GIS applications
mapping
delivery applications
fleet tracking
telecommunications
real estate
location-based searches
logistics
geofencing
```

Spatial indexes are especially important.

For example:

```sql
CREATE INDEX idx_stores_location
ON stores
USING GIST(location);
```

Without proper spatial indexes, geographic searches over large datasets can become very expensive.

---

# 8. `pg_cron`

`pg_cron` adds cron-style job scheduling inside PostgreSQL.

It runs as a PostgreSQL extension and uses a background worker to manage scheduled jobs. ([GitHub][6])

Examples include:

```text
delete old audit records
refresh materialized views
run partition maintenance
archive historical rows
collect statistics
perform periodic cleanup
```

Enable it with:

```sql
CREATE EXTENSION pg_cron;
```

A simple scheduled job might look like:

```sql
SELECT cron.schedule(
    'nightly-cleanup',
    '0 2 * * *',
    $$
    DELETE FROM audit_log
    WHERE created_at < now() - interval '90 days'
    $$
);
```

Meaning:

```text
0 2 * * *
| | | | |
| | | | +-- day of week
| | | +---- month
| | +------ day of month
| +-------- hour
+---------- minute
```

So this runs at:

```text
02:00 every day
```

`pg_cron` permits jobs to run in parallel, while preventing multiple concurrent executions of the same job; if the next execution arrives before the current one finishes, it is queued. ([GitHub][6])

---

# 9. Extension upgrades

Extensions have their own versions.

Check them with:

```sql
SELECT
    extname,
    extversion
FROM pg_extension;
```

An extension can provide migration scripts such as:

```text
myextension--1.0--1.1.sql
myextension--1.1--1.2.sql
```

Then PostgreSQL can run:

```sql
ALTER EXTENSION myextension UPDATE;
```

or:

```sql
ALTER EXTENSION myextension
UPDATE TO '1.2';
```

PostgreSQL uses the extension's update scripts to migrate its objects while tracking the installed version. ([PostgreSQL][2])

---

# 10. PostgreSQL upgrade vs extension upgrade

This is one of the most important operational concepts.

Suppose you have:

```text
PostgreSQL 16
PostGIS 3.x
pg_cron
```

and want:

```text
PostgreSQL 18
```

You must check whether the versions of PostGIS, `pg_cron`, and every other installed extension support the PostgreSQL target version.

Think of it as:

```text
Application
     |
     v
PostgreSQL version
     |
     +---- Extension A version
     +---- Extension B version
     +---- Extension C version
```

All layers must be compatible.

You should therefore create an extension compatibility matrix before a major PostgreSQL upgrade.

Example:

| Extension          | Current |  Target | PG target supported?                     | Action              |
| ------------------ | ------: | ------: | ---------------------------------------- | ------------------- |
| PostGIS            |     3.x |     3.x | Verify                                   | Upgrade if required |
| pg_cron            |     x.x |     x.x | Verify                                   | Test                |
| pg_stat_statements | bundled | bundled | Yes for matching PostgreSQL distribution | Reconfigure         |
| custom_ext         |     1.4 |       ? | Verify                                   | Rebuild/test        |

Never assume that because PostgreSQL itself upgrades successfully, every binary extension will automatically work.

As of the current PostgreSQL 18 documentation, supported PostgreSQL major releases are 14 through 18, while PostgreSQL 19 is still a development release. ([PostgreSQL][3])

---

# 11. Why binary extensions need extra attention

Some extensions contain only SQL.

Others contain compiled C code.

Conceptually:

```text
SQL-only extension
      |
      +-- functions
      +-- views
      +-- operators

Binary extension
      |
      +-- SQL objects
      |
      +-- shared library
              |
              +-- PostgreSQL server API
```

A compiled extension may need to be rebuilt when PostgreSQL is upgraded.

Examples of things to verify:

```text
CPU architecture
Operating system
PostgreSQL major version
extension release
compiler/runtime dependencies
package repository
cloud-provider support
```

---

# 12. Managed PostgreSQL compatibility

Extensions become particularly important with services such as managed PostgreSQL.

A self-managed server may allow:

```text
install arbitrary operating-system packages
compile C extensions
modify shared_preload_libraries
restart PostgreSQL
```

A managed service may restrict some or all of those actions.

Therefore:

```sql
CREATE EXTENSION something;
```

does **not** guarantee that `something` can be used on every PostgreSQL platform.

The provider must first make the extension available and may restrict:

```text
extension versions
configuration parameters
superuser privileges
shared libraries
background workers
server restarts
```

This should be checked before choosing an extension for application architecture.

---

# 13. Extension security

Extensions should effectively be treated as executable database software.

Running:

```sql
CREATE EXTENSION some_extension;
```

may create:

```text
functions
triggers
operators
data types
background workers
shared-library code
privileged objects
```

Many extensions therefore require superuser-level privileges. 

PostgreSQL also supports **trusted extensions**, which can be installed by users having `CREATE` privilege on a database when the extension's control file declares it trusted. ([PostgreSQL][1])

From a security standpoint:

```text
Untrusted extension package
            |
            v
Potential privileged code
            |
            v
Database/server compromise
```

So extensions should come from approved and reviewed sources.

---

# 14. Extension governance

For enterprise environments, don't allow extensions to become:

```text
Developer finds extension
        ↓
CREATE EXTENSION
        ↓
Production dependency
```

Instead use something like:

```text
Extension request
       ↓
Architecture review
       ↓
Security review
       ↓
License review
       ↓
Compatibility review
       ↓
Performance testing
       ↓
Approval
       ↓
Controlled deployment
```

Maintain an **extension inventory** such as:

| Extension          | Purpose                | Owner    | Version            | Databases  | Risk   | Approved |
| ------------------ | ---------------------- | -------- | ------------------ | ---------- | ------ | -------- |
| pg_stat_statements | Performance monitoring | DBA      | PostgreSQL bundled | All prod   | Low    | Yes      |
| PostGIS            | GIS                    | GIS Team | 3.x                | geo_prod   | Medium | Yes      |
| pg_cron            | Scheduling             | DBA      | x.x                | operations | Medium | Yes      |

---

# 15. Production governance rules

A good architecture standard is:

```text
1. No extension without a business/technical reason.
2. Use trusted, maintained projects.
3. Pin extension versions.
4. Verify PostgreSQL-version compatibility.
5. Test extension upgrades before PostgreSQL upgrades.
6. Review privileges and security impact.
7. Track licenses.
8. Monitor extension performance.
9. Include extensions in backup/restore testing.
10. Document an exit strategy.
```

The last point matters.

If an application stores millions of rows as:

```sql
geometry
```

then PostGIS is no longer just a convenience. It has become part of the application's data architecture.

---

# 16. Useful administration queries

Installed extensions:

```sql
SELECT
    extname,
    extversion
FROM pg_extension
ORDER BY extname;
```

Available extensions:

```sql
SELECT
    name,
    default_version,
    installed_version,
    comment
FROM pg_available_extensions
ORDER BY name;
```

Extensions where installed and available versions differ:

```sql
SELECT
    name,
    installed_version,
    default_version
FROM pg_available_extensions
WHERE installed_version IS NOT NULL
  AND installed_version <> default_version;
```

Be careful: a different `default_version` does **not automatically mean you should upgrade immediately**. Compatibility and release notes still need review.

---

# 17. Architecture comparison

Think of the three key examples like this:

| Extension            | Role                             | Architectural impact               |
| -------------------- | -------------------------------- | ---------------------------------- |
| `pg_stat_statements` | Observability                    | Mostly operational                 |
| PostGIS              | Geospatial database capabilities | Strong application/data dependency |
| `pg_cron`            | Database job scheduler           | Operational/application dependency |

This distinction is useful.

`pg_stat_statements` is usually something the DBA platform enables.

PostGIS can fundamentally change the application's data model.

`pg_cron` introduces scheduling and operational behavior directly into the database.

---

## Interview-style summary

If someone asks **"What is a PostgreSQL extension?"**, a strong answer is:

> A PostgreSQL extension is a packaged collection of database functionality that PostgreSQL can install and manage as one unit using `CREATE EXTENSION`.
> Extensions can add functions, types, operators, indexes, background workers, or complete capabilities such as PostGIS.
> In production, we also need to manage extension versions, PostgreSQL compatibility, privileges, security, upgrades, and provider support.

And the main commands to remember are:

```sql
CREATE EXTENSION extension_name;

SELECT * FROM pg_extension;

SELECT * FROM pg_available_extensions;

ALTER EXTENSION extension_name UPDATE;

DROP EXTENSION extension_name;
```

The architecture takeaway is:

```text
PostgreSQL Extension Strategy
          |
          +-- Functionality
          |      +-- PostGIS
          |      +-- pg_cron
          |
          +-- Observability
          |      +-- pg_stat_statements
          |
          +-- Compatibility
          |      +-- PostgreSQL version
          |      +-- extension version
          |      +-- OS/cloud platform
          |
          +-- Governance
                 +-- Security
                 +-- Ownership
                 +-- Licensing
                 +-- Upgrade testing
                 +-- Lifecycle management
```

**The main point:** extensions are one of PostgreSQL's biggest strengths, but once an extension becomes part of production, it should be managed like any other software dependency—not treated as just another SQL command.

[1]: https://www.postgresql.org/docs/current/sql-createextension.html "PostgreSQL: Documentation: 18: CREATE EXTENSION"
[2]: https://www.postgresql.org/docs/current/extend-extensions.html "PostgreSQL: Documentation: 18: 36.17. Packaging Related Objects into an Extension"
[3]: https://www.postgresql.org/docs/current/pgstatstatements.html "PostgreSQL: Documentation: 18: F.32. pg_stat_statements — track statistics of SQL planning and execution"
[4]: https://postgis.net/?utm_source=chatgpt.com "PostGIS"
[5]: https://postgis.net/documentation/getting_started/?utm_source=chatgpt.com "Getting Started"
[6]: https://github.com/citusdata/pg_cron/blob/main/README.md "pg_cron/README.md at main · citusdata/pg_cron · GitHub"
