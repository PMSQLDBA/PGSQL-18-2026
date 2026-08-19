# Topic 1 — PostgreSQL 18 Installation & Configuration

This topic should cover more than simply installing the PostgreSQL binaries. From a DBA/architecture perspective, it should explain the complete flow from **software installation → cluster initialization → directory structure → configuration → authentication → startup → validation → production baseline**.

The content below is aligned with the official PostgreSQL 18 documentation.

[PostgreSQL 18 Official Documentation](https://www.postgresql.org/docs/18/?utm_source=chatgpt.com)

---

## 1. PostgreSQL Installation Overview

A PostgreSQL installation has two distinct concepts that are important to separate:

**PostgreSQL software installation** installs executables and utilities such as:

```text
postgres
psql
initdb
pg_ctl
pg_dump
pg_restore
pg_basebackup
createdb
createuser
```

**Database cluster initialization** creates the actual database storage area (`PGDATA`) used by a PostgreSQL server instance.

In PostgreSQL terminology, a **database cluster** means a collection of databases managed by one PostgreSQL server instance. `initdb` creates this cluster and initially creates `postgres`, `template1`, and `template0`. ([PostgreSQL][1])

Conceptually:

```text
Operating System
      │
      ├── PostgreSQL 18 Software
      │       ├── postgres
      │       ├── psql
      │       ├── pg_ctl
      │       ├── initdb
      │       └── other utilities
      │
      └── PostgreSQL Cluster / Instance
              │
              ├── PGDATA
              ├── postgresql.conf
              ├── pg_hba.conf
              ├── pg_ident.conf
              │
              ├── postgres database
              ├── template0
              └── template1
```

This distinction becomes particularly important when operating **multiple PostgreSQL instances on the same server**.

---

# 2. PostgreSQL 18 Installation Methods

PostgreSQL can generally be installed using:

1. Operating-system packages
2. PostgreSQL community packages
3. Source-code compilation
4. Containers/images
5. Cloud-managed PostgreSQL services

The official PostgreSQL documentation has separate instructions for binary packages and source installation. PostgreSQL specifically notes that Chapter 17 applies to source-code installation; packaged distributions such as RPM or Debian packages should use the binary-package guidance instead. ([PostgreSQL][2])

For enterprise Linux environments, package-based installation is usually preferable because it simplifies:

```text
installation
patching
minor-version upgrades
dependency management
service management
standardization
```

Source installation is more appropriate when there is a specific requirement for custom compilation options or build-time features.

Official reference:

[PostgreSQL 18 Installation from Source Code](https://www.postgresql.org/docs/18/installation.html?utm_source=chatgpt.com)

---

# 3. PostgreSQL Operating-System Account

A PostgreSQL server should run under a dedicated operating-system account.

A typical account is:

```text
postgres
```

The architecture looks like:

```text
Linux Server
   │
   └── postgres OS account
          │
          └── PostgreSQL server processes
                 │
                 └── PGDATA
```

The PostgreSQL server **must not run as root**. Likewise, `initdb` must be executed by the operating-system account that will own the PostgreSQL server process. ([PostgreSQL][1])

For example:

```bash
mkdir -p /pgdata/18/data
chown -R postgres:postgres /pgdata
chmod 700 /pgdata/18/data
```

Then initialize the cluster as the PostgreSQL OS user.

```bash
su - postgres
```

This separation is important because the OS account controls access to the PostgreSQL data files, while PostgreSQL roles control access **inside the database**.

---

# 4. PostgreSQL Database Cluster Initialization

After installing PostgreSQL binaries, initialize the database cluster using:

```bash
initdb
```

Example:

```bash
initdb -D /pgdata/18/data
```

Or:

```bash
export PGDATA=/pgdata/18/data
initdb
```

`PGDATA` identifies the database cluster's data directory, and `-D` can override it. ([PostgreSQL][1])

You can also initialize through `pg_ctl`:

```bash
pg_ctl -D /pgdata/18/data initdb
```

PostgreSQL documents both approaches. ([PostgreSQL][3])

---

# 5. What `initdb` Actually Does

`initdb` is not just creating an empty directory.

It creates the internal structure PostgreSQL requires, including shared system catalogs and three initial databases:

```text
postgres
template0
template1
```

Their purposes are:

| Database    | Purpose                                             |
| ----------- | --------------------------------------------------- |
| `postgres`  | Default administrative/general-purpose database     |
| `template1` | Default template used when creating databases       |
| `template0` | Clean template that normally should not be modified |

For example:

```sql
CREATE DATABASE application_db;
```

By default, PostgreSQL creates the new database based on `template1`.

The official `initdb` documentation describes this initialization process. ([PostgreSQL][1])

---

# 6. Production Initialization Considerations

For production systems, cluster initialization deserves architectural planning.

Important decisions include:

```text
Data directory
Locale
Encoding
Authentication
Checksums
Filesystem
Storage layout
OS ownership
Backup design
WAL architecture
```

For example, rather than accepting settings without review, the DBA should explicitly understand what initialization options are appropriate for the organization's standard build.

One particularly important security point is authentication. PostgreSQL's documentation recommends avoiding an unsafe `trust` configuration when local users cannot all be trusted, and gives `scram-sha-256` as one possible authentication choice during initialization. ([PostgreSQL][3])

Example:

```bash
initdb \
  -D /pgdata/18/data \
  -A scram-sha-256 \
  -W
```

The exact production command should be determined by your organization's security, locale, encoding, and operational requirements rather than copied blindly.

---

# 7. Understanding `PGDATA`

`PGDATA` is one of the most important concepts for PostgreSQL administration.

Example:

```bash
export PGDATA=/pgdata/18/data
```

Conceptually:

```text
PostgreSQL Instance
       │
       ▼
     PGDATA
       │
       ├── base/
       ├── global/
       ├── pg_wal/
       ├── pg_tblspc/
       ├── pg_stat/
       ├── pg_stat_tmp/
       ├── pg_multixact/
       ├── pg_subtrans/
       ├── pg_xact/
       │
       ├── postgresql.conf
       ├── pg_hba.conf
       ├── pg_ident.conf
       ├── PG_VERSION
       └── postmaster.pid
```

PostgreSQL refers to the cluster data directory commonly as `PGDATA`, although configuration files can be relocated. ([PostgreSQL][4])

---

# 8. Important PGDATA Directories

A DBA should understand the important directories rather than treating `PGDATA` as a black box.

### `base/`

Contains per-database files for databases stored in the default tablespace.

Conceptually:

```text
base/
 ├── 1/
 ├── 5/
 └── 16384/
```

The directory names correspond internally to database identifiers.

---

### `global/`

Contains cluster-wide tables and information rather than information belonging to only one database.

---

### `pg_wal/`

Contains Write-Ahead Log files.

```text
Transaction
    │
    ▼
WAL generation
    │
    ▼
pg_wal/
```

WAL is fundamental to:

```text
crash recovery
physical replication
backup/recovery
PITR
```

The detailed WAL architecture belongs in your separate **WAL & Checkpoints** topic.

---

### `pg_tblspc/`

Contains symbolic links associated with user-defined tablespaces.

---

### `pg_xact/`

Contains transaction commit-status information.

---

### `PG_VERSION`

Identifies the PostgreSQL major version associated with the cluster.

For PostgreSQL 18, the cluster will identify itself as version 18.

PostgreSQL documents the physical cluster layout in its storage documentation. ([PostgreSQL][4])

[PostgreSQL 18 Database File Layout](https://www.postgresql.org/docs/18/storage-file-layout.html?utm_source=chatgpt.com)

---

# 9. Three Critical Configuration Files

The three files every PostgreSQL administrator should understand are:

```text
postgresql.conf
pg_hba.conf
pg_ident.conf
```

By default, PostgreSQL traditionally keeps these configuration files with the database cluster, although PostgreSQL supports storing them separately. ([PostgreSQL][5])

Their responsibilities are different:

| File              | Responsibility                        |
| ----------------- | ------------------------------------- |
| `postgresql.conf` | PostgreSQL server behavior            |
| `pg_hba.conf`     | Client authentication rules           |
| `pg_ident.conf`   | External-to-database username mapping |

This distinction is fundamental.

---

# 10. `postgresql.conf`

`postgresql.conf` is the primary server configuration file.

It controls areas including:

```text
connections
memory
WAL
checkpoints
replication
query planning
logging
statistics
vacuum
locks
SSL
worker processes
I/O
```

PostgreSQL 18 organizes these settings under its Server Configuration chapter. ([PostgreSQL][6])

[PostgreSQL 18 Server Configuration](https://www.postgresql.org/docs/18/runtime-config.html?utm_source=chatgpt.com)

---

# 11. Important Connection Parameters

A basic configuration might contain:

```conf
listen_addresses = '*'
port = 5432
max_connections = 200
```

These settings have different purposes.

### `listen_addresses`

Controls which network interfaces PostgreSQL listens on.

For example:

```conf
listen_addresses = 'localhost'
```

means PostgreSQL accepts TCP connections only through the local host interfaces.

A configuration such as:

```conf
listen_addresses = '*'
```

allows PostgreSQL to listen on all available interfaces, but **does not by itself authorize every remote client**. Client access must still satisfy authentication rules such as those in `pg_hba.conf`.

That distinction is important:

```text
listen_addresses
       │
       ▼
Where PostgreSQL listens
       │
       ▼
pg_hba.conf
       │
       ▼
Who can authenticate
       │
       ▼
PostgreSQL privileges
       │
       ▼
What authenticated role can do
```

---

# 12. PostgreSQL Port

The standard PostgreSQL port is:

```text
5432
```

Configuration:

```conf
port = 5432
```

For multiple PostgreSQL instances on one host, each TCP-listening instance normally needs its own port.

Example:

```text
PostgreSQL 18 PROD     → 5432
PostgreSQL 18 TEST     → 5433
PostgreSQL 18 DEV      → 5434
```

The port alone should not be treated as a security control.

---

# 13. `max_connections`

Example:

```conf
max_connections = 200
```

This controls the maximum number of concurrent database connections, subject to PostgreSQL's reserved connection mechanisms.

A common mistake is simply increasing this parameter whenever applications run out of connections.

For example:

```text
Application
   │
   ├── 1000 application threads
   │
   ▼
1000 DB connections
   │
   ▼
PostgreSQL
```

That architecture can create unnecessary process and memory pressure.

A better design may use connection pooling:

```text
Applications
     │
     │ 1000 logical requests
     ▼
Connection Pooler
     │
     │ controlled DB connections
     ▼
PostgreSQL
```

Connection sizing should therefore be handled as an architecture/capacity decision rather than a single configuration number.

---

# 14. Memory Configuration

Important PostgreSQL parameters include:

```conf
shared_buffers
work_mem
maintenance_work_mem
effective_cache_size
```

But these should **not** be configured using a universal set of magic percentages.

They depend on:

```text
server RAM
concurrent sessions
query workload
number of databases
connection pooling
maintenance activity
operating system
other processes
HA architecture
```

A particularly important point is that `work_mem` is not simply a fixed amount allocated once per connection. Queries can perform multiple memory-consuming operations, and concurrent queries multiply the potential memory demand.

Therefore:

```text
work_mem = 64MB
```

does **not** mean:

```text
100 connections × 64MB
```

is necessarily the maximum relevant memory consumption.

Memory configuration deserves workload-based testing.

PostgreSQL 18 groups these parameters under resource consumption settings. ([PostgreSQL][6])

---

# 15. `pg_hba.conf`

`pg_hba.conf` controls client authentication.

HBA means:

**Host-Based Authentication**

A typical entry follows the structure:

```text
TYPE   DATABASE   USER   ADDRESS   METHOD
```

Example:

```conf
host    appdb    appuser    10.10.20.0/24    scram-sha-256
```

Meaning roughly:

```text
TCP connection
      │
      ├── Database = appdb
      ├── User     = appuser
      ├── Network  = 10.10.20.0/24
      │
      └── Authentication = SCRAM-SHA-256
```

PostgreSQL evaluates `pg_hba.conf` records in order, so rule ordering matters. ([PostgreSQL][7])

---

# 16. Authentication Architecture

When an application connects:

```text
Application
    │
    ▼
Network / PostgreSQL listener
    │
    ▼
pg_hba.conf
    │
    ▼
Authentication method
    │
    ▼
PostgreSQL Role
    │
    ▼
Database/Object privileges
```

These are separate layers.

For example, successful authentication does not automatically mean the user can:

```sql
SELECT * FROM payroll;
```

The PostgreSQL role must also have appropriate privileges.

PostgreSQL 18 explains that authentication establishes the client's identity, while PostgreSQL role privileges determine access to database objects. ([PostgreSQL][8])

[PostgreSQL 18 Client Authentication](https://www.postgresql.org/docs/18/client-authentication.html?utm_source=chatgpt.com)

---

# 17. Authentication Methods

PostgreSQL supports multiple authentication approaches.

Depending on platform and requirements, these include mechanisms such as:

```text
SCRAM password authentication
peer
certificate
LDAP
GSSAPI
SSPI
PAM
OAuth
```

PostgreSQL 18 notably added support for **OAuth authentication** through `pg_hba.conf` and associated client/server configuration. ([PostgreSQL][9])

For normal password-based environments, SCRAM is the modern PostgreSQL password authentication mechanism.

An example rule is:

```conf
host    appdb    appuser    10.10.20.0/24    scram-sha-256
```

Avoid broad rules such as unrestricted network access unless there is a carefully reviewed reason.

---

# 18. `pg_ident.conf`

`pg_ident.conf` provides username mapping.

It is useful when an external authentication identity does not directly match the desired PostgreSQL role.

Conceptually:

```text
External Identity
       │
       ▼
pg_ident.conf
       │
       ▼
PostgreSQL Role
```

This is especially relevant to authentication methods where external identities are mapped to database roles.

---

# 19. Configuration File Locations

PostgreSQL allows configuration and data files to be separated.

Relevant parameters include:

```conf
data_directory
hba_file
ident_file
```

and PostgreSQL can also be told where its main configuration file is located.

For example, an organization could design:

```text
/etc/postgresql/18/prod/
        │
        ├── postgresql.conf
        ├── pg_hba.conf
        └── pg_ident.conf

/pgdata/18/prod/
        │
        └── database files
```

PostgreSQL explicitly supports separating configuration files from the data directory. ([PostgreSQL][5])

[PostgreSQL 18 Configuration File Locations](https://www.postgresql.org/docs/18/runtime-config-file-locations.html?utm_source=chatgpt.com)

This can be useful for configuration management and backup processes, but the design must ensure that configuration files are backed up and restored consistently with operational procedures.

---

# 20. Starting PostgreSQL

The actual database server executable is:

```text
postgres
```

A manual start could look like:

```bash
postgres -D /pgdata/18/data
```

More commonly for direct administration, `pg_ctl` can manage the instance:

```bash
pg_ctl -D /pgdata/18/data start
```

PostgreSQL documents both approaches, while also recommending use of the operating system's package/service infrastructure when PostgreSQL was installed through a package. ([PostgreSQL][10])

Official reference:

[PostgreSQL 18 Starting the Database Server](https://www.postgresql.org/docs/18/server-start.html?utm_source=chatgpt.com)

---

# 21. PostgreSQL Service Management

In a Linux production environment, PostgreSQL will commonly be managed through the OS service manager.

Conceptually:

```text
systemd
   │
   ▼
PostgreSQL service
   │
   ▼
postgres
   │
   ▼
PostgreSQL child/background processes
```

Depending on the package/distribution, commands may look similar to:

```bash
systemctl start <postgresql-service>
systemctl stop <postgresql-service>
systemctl restart <postgresql-service>
systemctl status <postgresql-service>
```

The **exact service name and filesystem locations are package/distribution specific**, so they should not be hard-coded into a generic PostgreSQL architecture standard.

PostgreSQL's official documentation explicitly notes that pre-packaged versions normally provide OS-specific mechanisms for running the server as a background service. ([PostgreSQL][10])

---

# 22. Reload vs Restart

This distinction is critical for DBAs.

Some configuration changes can be applied with a reload:

```bash
pg_ctl reload
```

or from SQL:

```sql
SELECT pg_reload_conf();
```

Other parameters require PostgreSQL to restart.

Conceptually:

```text
Configuration Change
        │
        ▼
Does parameter require restart?
       / \
     No   Yes
     │     │
  Reload Restart
```

Before changing production configuration, DBAs should determine whether the specific parameter is dynamically reloadable or requires restart.

---

# 23. Validate the Running Configuration

Do not assume that the value written into a file is necessarily the effective PostgreSQL value.

Use PostgreSQL itself to check.

For example:

```sql
SHOW data_directory;
```

```sql
SHOW config_file;
```

```sql
SHOW hba_file;
```

```sql
SHOW port;
```

```sql
SHOW max_connections;
```

And:

```sql
SELECT version();
```

You can inspect PostgreSQL settings through:

```sql
SELECT name, setting, unit, context
FROM pg_settings
ORDER BY name;
```

This is very useful because it shows the **effective runtime configuration**.

---

# 24. Find the Actual Configuration Files

On systems installed through packages, avoid assuming:

```text
postgresql.conf must be under PGDATA
```

Instead ask PostgreSQL:

```sql
SHOW config_file;
SHOW hba_file;
SHOW data_directory;
```

For example, it is completely valid for PostgreSQL to have:

```text
Configuration:
    /etc/postgresql/18/prod/postgresql.conf

Data:
    /data/postgresql/18/prod/
```

The PostgreSQL 18 documentation explicitly supports this separation. ([PostgreSQL][5])

---

# 25. Basic Installation Validation

After installation, verify the software:

```bash
postgres --version
```

```bash
psql --version
```

Then connect:

```bash
psql -d postgres
```

Inside PostgreSQL:

```sql
SELECT version();
```

Check databases:

```sql
\l
```

Check roles:

```sql
\du
```

Check configuration:

```sql
SHOW data_directory;
SHOW config_file;
SHOW hba_file;
SHOW port;
```

The expected architecture is:

```text
Client / psql
      │
      ▼
TCP / Unix Socket
      │
      ▼
PostgreSQL Listener
      │
      ▼
Authentication
      │
      ▼
PostgreSQL Backend
      │
      ▼
Database
      │
      ▼
Storage
```

---

# 26. Production Directory Design

For enterprise PostgreSQL, directory design should be planned before installation.

A conceptual design could be:

```text
/pgdata/
   └── 18/
       └── prod/
           └── data/

/pgwal/
   └── 18/
       └── prod/

/pgbackup/
   └── prod/

/pgarchive/
   └── prod/

/pglog/
   └── prod/
```

This is an **architecture example, not a PostgreSQL-required directory standard**.

The actual design depends on:

```text
storage platform
backup solution
HA architecture
I/O requirements
filesystem
cloud/on-prem environment
security requirements
operational standards
```

One particularly important warning: moving or separating WAL storage should be done deliberately. `pg_wal` is part of PostgreSQL's core durability and recovery architecture.

---

# 27. Configuration Management Strategy

For production environments, avoid treating `postgresql.conf` as an unmanaged file that DBAs edit differently on every server.

A stronger architecture is:

```text
Enterprise PostgreSQL Standard
             │
             ▼
Baseline Configuration
             │
       ┌─────┼─────┐
       ▼     ▼     ▼
      DEV   TEST   PROD
                    │
                    ▼
            Environment overrides
```

Parameters can be classified as:

### Global baseline

```text
logging standards
security standards
SSL requirements
monitoring
audit requirements
```

### Environment-specific

```text
memory
connections
WAL
replication
backup
storage
```

### Workload-specific

```text
work_mem
parallelism
autovacuum tuning
planner parameters
```

This makes configuration reproducible and auditable.

---

# 28. PostgreSQL 18 Installation Architecture

A good production installation can be viewed as several layers:

```text
                 Applications
                      │
                      ▼
              Connection Pooler
                      │
                      ▼
              PostgreSQL 18
                      │
        ┌─────────────┼──────────────┐
        │             │              │
        ▼             ▼              ▼
 Configuration    Authentication    Logging
postgresql.conf   pg_hba.conf
        │
        ▼
   PostgreSQL
   Processes
        │
        ▼
      PGDATA
        │
   ┌────┼─────────┐
   ▼    ▼         ▼
 Data   WAL    Tablespaces
        │
        ▼
 Backup / Recovery / Replication
```

This is why **installation and configuration should be treated as an architecture topic**, rather than simply documenting `yum install` or `apt install`.

---

# 29. DBA Installation Checklist

For a production PostgreSQL 18 build, I would verify at least:

| Area               | Validation                                 |
| ------------------ | ------------------------------------------ |
| PostgreSQL version | Correct PostgreSQL 18 release installed    |
| OS account         | Dedicated PostgreSQL service account       |
| PGDATA             | Correct storage location                   |
| Ownership          | PostgreSQL account owns required files     |
| Permissions        | Data directory appropriately restricted    |
| Cluster            | Successfully initialized                   |
| Locale/encoding    | Matches application requirements           |
| Authentication     | Reviewed `pg_hba.conf`                     |
| Password auth      | Secure authentication strategy             |
| Network            | Correct listener interfaces                |
| Port               | Correct instance port                      |
| Connections        | Capacity planned                           |
| Memory             | Sized for workload/server                  |
| WAL                | Storage and recovery requirements reviewed |
| Logging            | Production logging configured              |
| SSL/TLS            | Configured where required                  |
| Backup             | Backup strategy established                |
| Monitoring         | Monitoring integration configured          |
| Service            | Automatic startup configured               |
| Validation         | Test connection completed                  |
| Documentation      | Configuration baseline recorded            |

---

# 30. Recommended Enterprise Build Flow

For an architecture document, I would define the installation process like this:

```text
1. Validate OS prerequisites
          │
          ▼
2. Provision storage
          │
          ▼
3. Create PostgreSQL OS account
          │
          ▼
4. Install PostgreSQL 18 software
          │
          ▼
5. Create directory structure
          │
          ▼
6. Initialize PostgreSQL cluster
          │
          ▼
7. Configure postgresql.conf
          │
          ▼
8. Configure pg_hba.conf
          │
          ▼
9. Configure TLS/security
          │
          ▼
10. Configure OS service
          │
          ▼
11. Start PostgreSQL
          │
          ▼
12. Validate configuration
          │
          ▼
13. Configure monitoring
          │
          ▼
14. Configure backup/recovery
          │
          ▼
15. Perform connectivity test
          │
          ▼
16. Capture configuration baseline
          │
          ▼
17. Production readiness review
```

This is much closer to how **PostgreSQL installation should be handled in an enterprise architecture** than simply documenting package installation commands.

### Primary PostgreSQL 18 references

The most useful official sections for this topic are [PostgreSQL 18 Server Setup and Operation](https://www.postgresql.org/docs/18/runtime.html?utm_source=chatgpt.com), [PostgreSQL 18 Server Configuration](https://www.postgresql.org/docs/18/runtime-config.html?utm_source=chatgpt.com), [PostgreSQL 18 initdb](https://www.postgresql.org/docs/18/app-initdb.html?utm_source=chatgpt.com), [PostgreSQL 18 Client Authentication](https://www.postgresql.org/docs/18/client-authentication.html?utm_source=chatgpt.com), and [PostgreSQL 18 Database File Layout](https://www.postgresql.org/docs/18/storage-file-layout.html?utm_source=chatgpt.com). These collectively cover the core installation, cluster initialization, configuration, authentication, startup, and physical-layout concepts used above. ([PostgreSQL][11])

[1]: https://www.postgresql.org/docs/18/app-initdb.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: initdb"
[2]: https://www.postgresql.org/docs/18/installation.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 17. Installation from Source Code"
[3]: https://www.postgresql.org/docs/current/creating-cluster.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 18.2. Creating a Database Cluster"
[4]: https://www.postgresql.org/docs/18/storage-file-layout.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 66.1. Database File Layout"
[5]: https://www.postgresql.org/docs/18/runtime-config-file-locations.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.2. File Locations"
[6]: https://www.postgresql.org/docs/18/runtime-config.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 19. Server Configuration"
[7]: https://www.postgresql.org/docs/current/auth-pg-hba-conf.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 20.1. The pg_hba.conf File"
[8]: https://www.postgresql.org/docs/18/client-authentication.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 20. Client Authentication"
[9]: https://www.postgresql.org/docs/18/release-18.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: E.5. Release 18"
[10]: https://www.postgresql.org/docs/18/server-start.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 18.3. Starting the Database Server"
[11]: https://www.postgresql.org/docs/18/runtime.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 18. Server Setup and Operation"
