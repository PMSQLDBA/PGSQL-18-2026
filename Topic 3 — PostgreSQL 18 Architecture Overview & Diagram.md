# Topic 3 — PostgreSQL 18 Architecture Overview & Diagram

PostgreSQL uses a **client/server, process-based architecture**. A PostgreSQL server instance manages a database cluster, uses shared memory for common data structures, starts background/auxiliary processes for database operations, and normally creates a dedicated backend process for each client connection. ([PostgreSQL][1])

[PostgreSQL 18 — Architectural Fundamentals](https://www.postgresql.org/docs/18/tutorial-arch.html?utm_source=chatgpt.com)

---

## 1. High-Level PostgreSQL Architecture

At a high level:

```text
                        CLIENT LAYER
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
     Application           psql             BI / ETL
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                    TCP/IP / Unix Socket
                             │
                             ▼
                ┌─────────────────────────┐
                │      PostgreSQL 18      │
                │   Primary Server Process│
                │       (postgres)        │
                └────────────┬────────────┘
                             │
              ┌──────────────┼───────────────┐
              │              │               │
              ▼              ▼               ▼
        Backend          Shared Memory    Background /
        Processes                        Auxiliary Processes
              │              │               │
              │        ┌─────┼──────┐        │
              │        │     │      │        │
              │     Shared   WAL   Other     │
              │     Buffers Buffers Shared   │
              │                     Data     │
              │                              │
              └──────────────┬───────────────┘
                             │
                             ▼
                       STORAGE LAYER
                             │
              ┌──────────────┼─────────────┐
              ▼              ▼             ▼
           Data Files      pg_wal       Tablespaces
```

The key architectural areas are therefore:

**Client → Connection → Backend Process → Shared Memory → Background Processes → WAL/Data Storage**

---

# 2. Client/Server Architecture

PostgreSQL follows a client/server model.

Clients can include:

```text
Web Application
Java Application
.NET Application
Python Application
psql
ETL Tool
Reporting Tool
Microservice
Monitoring Tool
```

They connect through:

```text
TCP/IP
or
Unix-domain socket
```

Conceptually:

```text
Application
     │
     │ SQL
     ▼
PostgreSQL Server
     │
     │ Result
     ▼
Application
```

The PostgreSQL server handles database files on behalf of clients; applications do not directly read PostgreSQL's physical table files.

---

# 3. PostgreSQL Instance vs Cluster vs Database

These terms are important.

### Server/Host

The physical machine, VM, container, or other computer environment where PostgreSQL runs. PostgreSQL's glossary explicitly allows "server" to refer to physical hardware, a container, or a VM. ([PostgreSQL][1])

### PostgreSQL Instance

A running PostgreSQL server managing one database cluster.

### Database Cluster

A collection of databases managed by one PostgreSQL server instance.

### Database

An individual database inside the cluster.

Architecture:

```text
Server / VM
│
├── PostgreSQL Instance
│      │
│      └── Cluster / PGDATA
│             │
│             ├── postgres
│             ├── appdb
│             ├── reportingdb
│             └── inventorydb
│
└── PostgreSQL Processes
```

A single host can run multiple PostgreSQL clusters/instances. PostgreSQL's storage documentation explicitly notes that multiple clusters managed by different server instances can exist on the same machine. ([PostgreSQL][2])

---

# 4. PostgreSQL Is Process-Based

One of the most important PostgreSQL architecture concepts is its **process model**.

When you inspect PostgreSQL at OS level, you will see multiple `postgres` processes.

For example:

```bash
ps -ef | grep postgres
```

Conceptually:

```text
postgres
   │
   ├── checkpointer
   ├── background writer
   ├── WAL writer
   ├── autovacuum launcher
   ├── other auxiliary/background processes
   │
   ├── backend → Application connection 1
   ├── backend → Application connection 2
   └── backend → DBA connection
```

PostgreSQL's documentation describes the first process as the primary server process, followed by background processes and then server processes handling individual client connections. ([PostgreSQL][3])

---

# 5. Primary Server Process

The main PostgreSQL executable is:

```text
postgres
```

It is responsible for coordinating the instance, including accepting new client connections and managing PostgreSQL child processes.

Think of it as:

```text
                  postgres
                     │
          ┌──────────┼──────────┐
          │          │          │
       Client     Background   Shared
       Backends    Processes   Resources
```

You may still encounter the historical term **postmaster** in PostgreSQL documentation, configuration/file names, and DBA discussions.

---

# 6. Backend Processes

A backend is the PostgreSQL process that acts on behalf of a client session and processes its requests. ([PostgreSQL][1])

For example:

```text
Application Connection 1
          │
          ▼
    Backend Process 1

Application Connection 2
          │
          ▼
    Backend Process 2

psql Connection
          │
          ▼
    Backend Process 3
```

So conceptually:

```text
100 database connections
          │
          ▼
roughly 100 client backend processes
```

plus PostgreSQL's auxiliary/background processes.

This is one reason connection management matters so much in PostgreSQL.

---

# 7. Why Connection Pooling Matters

Suppose an application architecture contains:

```text
50 Application Servers
        ×
100 Connections
        =
5,000 PostgreSQL connections
```

Directly opening thousands of PostgreSQL sessions can create significant resource overhead.

A pooler can provide:

```text
Applications
     │
     │ Many logical/client connections
     ▼
┌──────────────┐
│   PgBouncer  │
└──────┬───────┘
       │
       │ Smaller controlled connection pool
       ▼
┌──────────────┐
│ PostgreSQL   │
└──────────────┘
```

We will cover this properly under **Topic 10 — PostgreSQL Connection Management**.

---

# 8. PostgreSQL Shared Memory

PostgreSQL processes need common memory areas to coordinate their work.

PostgreSQL defines shared memory as RAM used by processes common to an instance. It holds cached database pages, transient WAL information, and other shared state. ([PostgreSQL][1])

High-level view:

```text
               PostgreSQL Shared Memory
                        │
          ┌─────────────┼──────────────┐
          │             │              │
          ▼             ▼              ▼
    Shared Buffers   WAL-related    Other Shared
                       Memory          State
```

The largest part is normally:

```text
shared_buffers
```

PostgreSQL 18 defines `shared_buffers` as the amount of memory used by the database server for shared memory buffers. ([PostgreSQL][4])

---

# 9. Shared Buffers

Shared buffers cache database pages in PostgreSQL memory.

Architecture:

```text
SQL Query
    │
    ▼
Backend Process
    │
    ▼
Shared Buffers
    │
    ├── Page already available
    │         │
    │         └── Use cached page
    │
    └── Page not available
              │
              ▼
          Storage
              │
              ▼
       Load page into
       Shared Buffers
```

This reduces the need for PostgreSQL to repeatedly request the same pages from storage.

---

# 10. Dirty Pages

Suppose PostgreSQL modifies a row.

The corresponding page in shared buffers becomes **dirty**.

```text
SELECT
   │
   ▼
Read Page
   │
   ▼
Shared Buffer


UPDATE
   │
   ▼
Modify Page
   │
   ▼
Dirty Buffer
```

A dirty page means the in-memory version has changes that need eventually to reach persistent storage.

PostgreSQL's glossary defines a dirty page as a modified shared-buffer page that has not yet been written back to the filesystem. ([PostgreSQL][1])

---

# 11. Background Writer

PostgreSQL has a background writer process.

Its purpose is to write dirty shared buffers to the filesystem gradually, helping spread I/O work rather than forcing large bursts. ([PostgreSQL][4])

```text
Shared Buffers
      │
      │ Dirty pages
      ▼
Background Writer
      │
      ▼
Operating System
      │
      ▼
Storage
```

The background writer is **not the same thing as the checkpointer**.

That distinction is important.

---

# 12. Checkpointer

PostgreSQL periodically performs checkpoints.

At a high level:

```text
             Shared Buffers
                  │
                  │ Dirty pages
                  ▼
             Checkpoint
                  │
                  ▼
             Data Files
```

Checkpoints create recovery points and ensure dirty data pages are eventually written toward durable storage.

The relationship among:

```text
WAL
Checkpoints
Dirty buffers
Crash recovery
```

is critical enough that we will cover it separately under **Topic 15 — PostgreSQL WAL & Checkpoints**.

---

# 13. WAL — Write-Ahead Logging

PostgreSQL uses **Write-Ahead Logging (WAL)**.

The fundamental idea is:

> The WAL describing a change must reach durable storage before the changed data page itself is allowed to reach durable storage.

Conceptually:

```text
UPDATE customer
        │
        ▼
Database Change
        │
        ├───────────────┐
        ▼               ▼
   WAL Record       Modified Page
        │               │
        ▼               ▼
      WAL             Shared
     Buffer           Buffers
        │               │
        ▼               │
     pg_wal             │
        │               │
        └──── before ────┘
                    eventual
                    data write
```

This architecture is fundamental to PostgreSQL durability and crash recovery.

---

# 14. WAL Writer

PostgreSQL has a WAL writer auxiliary process.

Its job is associated with writing WAL buffers toward persistent WAL storage.

Simplified architecture:

```text
Transactions
     │
     ▼
WAL Records
     │
     ▼
WAL Buffers
     │
     ▼
WAL Writer / backend WAL flushes
     │
     ▼
pg_wal
```

Do not confuse:

```text
Background Writer → data buffers

WAL Writer        → WAL
```

They perform different jobs.

---

# 15. Autovacuum Architecture

PostgreSQL's MVCC architecture generates old row versions as data changes.

PostgreSQL therefore includes autovacuum infrastructure.

Simplified:

```text
Autovacuum Launcher
        │
        ▼
Determine databases needing work
        │
        ▼
Autovacuum Workers
        │
        ▼
Tables
        │
        ├── VACUUM-related work
        └── ANALYZE-related work
```

This is fundamental to PostgreSQL operations and will be covered deeply under **Topic 12 — Vacuum & Autovacuum**.

---

# 16. Other Auxiliary and Background Processes

Depending on configuration and workload, PostgreSQL can have processes related to:

```text
Checkpointer
Background Writer
WAL Writer
Autovacuum Launcher
Autovacuum Workers
WAL Archiver
WAL Sender
WAL Receiver
Startup/Recovery
Logical Replication
Parallel Query Workers
Custom Background Workers
```

PostgreSQL 18's glossary identifies auxiliary processes including the autovacuum launcher, background writer, checkpointer, logger, startup process, WAL archiver, WAL receiver, WAL summarizer, and WAL writer. ([PostgreSQL][1])

Some processes only exist when the relevant feature is enabled.

---

# 17. PostgreSQL 18 Worker Architecture

PostgreSQL also supports worker processes.

For example:

```text
Backend
   │
   ▼
Parallel Query
   │
   ├── Worker 1
   ├── Worker 2
   └── Worker 3
```

Important parameters include:

```text
max_worker_processes
max_parallel_workers
max_parallel_workers_per_gather
max_parallel_maintenance_workers
```

PostgreSQL 18 documents `max_worker_processes` as the maximum number of background processes supported by the cluster and explains how parallel workers consume that worker pool. ([PostgreSQL][4])

---

# 18. Query Processing Architecture

A SQL query does not go directly from the application to a table.

The simplified internal path is:

```text
Client
   │
   ▼
Backend Process
   │
   ▼
Parser
   │
   ▼
Analyzer / Rewrite
   │
   ▼
Planner / Optimizer
   │
   ▼
Execution Plan
   │
   ▼
Executor
   │
   ▼
Shared Buffers / Storage
   │
   ▼
Result
   │
   ▼
Client
```

PostgreSQL's internals documentation describes the backend path from receiving a query through parsing, planning/optimization, execution, and returning results. ([PostgreSQL][5])

[PostgreSQL 18 — Overview of PostgreSQL Internals](https://www.postgresql.org/docs/18/overview.html?utm_source=chatgpt.com)

---

# 19. Parser

Suppose the application sends:

```sql
SELECT customer_name
FROM customer
WHERE customer_id = 100;
```

The parser checks the SQL syntax and constructs PostgreSQL's internal representation of the statement.

Conceptually:

```text
SQL Text
   │
   ▼
Parser
   │
   ▼
Parse Tree
```

---

# 20. Analyzer / Rewrite Stage

PostgreSQL then resolves things such as:

```text
Table names
Column names
Data types
Operators
Functions
Permissions-related information
```

and applies PostgreSQL's rewrite machinery where required.

Simplified:

```text
Parse Tree
    │
    ▼
Analysis / Rewrite
    │
    ▼
Internal Query Tree
```

---

# 21. Planner / Optimizer

The optimizer determines how PostgreSQL should execute the query.

For example:

```sql
SELECT *
FROM orders
WHERE customer_id = 100;
```

Possible strategies might include:

```text
Sequential Scan
        OR
Index Scan
        OR
Bitmap Scan
```

The planner evaluates possible plans and estimates their costs.

```text
Query
  │
  ▼
Planner
  │
  ├── Plan A → estimated cost
  ├── Plan B → estimated cost
  └── Plan C → estimated cost
            │
            ▼
      Chosen Plan
```

This is why statistics and indexes matter so much to PostgreSQL performance.

---

# 22. Executor

Once a plan is selected:

```text
Execution Plan
      │
      ▼
Executor
      │
      ▼
Access rows/pages
      │
      ▼
Return result
```

For example:

```text
Index Scan
    │
    ▼
Index
    │
    ▼
Heap/Table Page
    │
    ▼
Matching Row
```

This becomes important when learning:

```sql
EXPLAIN
```

and:

```sql
EXPLAIN ANALYZE
```

which we will cover under performance tuning.

---

# 23. PostgreSQL Storage Architecture

The physical storage starts with the PostgreSQL cluster's data directory:

```text
PGDATA
```

PostgreSQL 18 documents the cluster layout in detail. ([PostgreSQL][2])

[PostgreSQL 18 — Database File Layout](https://www.postgresql.org/docs/18/storage-file-layout.html?utm_source=chatgpt.com)

A simplified layout:

```text
PGDATA
│
├── base/
│     └── Per-database files
│
├── global/
│     └── Cluster-wide relations
│
├── pg_wal/
│     └── WAL files
│
├── pg_tblspc/
│     └── Tablespace links
│
├── pg_replslot/
│     └── Replication slot state
│
├── pg_stat/
│     └── Statistics files
│
├── pg_xact/
│     └── Transaction commit status
│
├── postgresql.conf
├── pg_hba.conf
├── pg_ident.conf
├── postgresql.auto.conf
└── PG_VERSION
```

Configuration files can also be stored outside `PGDATA`. ([PostgreSQL][6])

---

# 24. Database Physical Layout

Inside `PGDATA/base`, PostgreSQL normally has a directory for each database, identified by the database's OID. ([PostgreSQL][2])

Conceptually:

```text
PGDATA
   │
   └── base
        │
        ├── 1
        ├── 5
        └── 16384
              │
              ├── Table files
              ├── Index files
              └── Other relation files
```

You can map a database name to its OID:

```sql
SELECT oid, datname
FROM pg_database;
```

---

# 25. PostgreSQL Pages

PostgreSQL stores tables and indexes in pages.

The normal PostgreSQL block/page size is commonly:

```text
8 KB
```

although block size is a build-time choice.

Conceptually:

```text
Table
 │
 ├── Page 1
 │     ├── Row
 │     ├── Row
 │     └── Row
 │
 ├── Page 2
 │
 └── Page N
```

Pages are fundamental because PostgreSQL's buffer manager operates on database pages rather than arbitrary individual rows.

PostgreSQL's physical-storage chapter covers database page layout along with free-space maps, visibility maps, TOAST and HOT. ([PostgreSQL][7])

---

# 26. Table Storage — Heap

A normal PostgreSQL table is commonly referred to as a **heap**.

Simplified:

```text
Table / Heap
     │
     ├── Page
     │    ├── Tuple
     │    ├── Tuple
     │    └── Tuple
     │
     └── Page
          └── Tuples
```

A tuple is essentially PostgreSQL's internal row representation.

Understanding heap storage becomes particularly useful for:

```text
MVCC
VACUUM
Bloat
HOT updates
Index scans
Visibility
```

---

# 27. TOAST Architecture

Large values may not fit conveniently inside normal table rows.

PostgreSQL uses **TOAST** mechanisms for large attributes.

Conceptually:

```text
Table Row
   │
   ├── Normal columns
   │
   └── Large value
          │
          ▼
        TOAST
          │
          ▼
   Compressed and/or
   out-of-line storage
```

PostgreSQL's physical-storage chapter specifically covers both on-disk and in-memory TOAST behavior. ([PostgreSQL][7])

---

# 28. Free Space Map

PostgreSQL tracks available space in relation pages using the **Free Space Map (FSM)**.

Conceptually:

```text
Table
 │
 ├── Page 1 → almost full
 ├── Page 2 → free space
 ├── Page 3 → full
 └── Page 4 → free space
             ▲
             │
             FSM
```

This helps PostgreSQL find pages with room for new tuples.

---

# 29. Visibility Map

PostgreSQL also maintains a **Visibility Map (VM)**.

It tracks information useful for determining whether pages contain tuples visible to all transactions and whether pages need freezing-related work.

This becomes particularly important for:

```text
VACUUM
Index-only scans
MVCC maintenance
```

Both the free-space map and visibility map are documented as part of PostgreSQL 18's physical storage architecture. ([PostgreSQL][7])

---

# 30. PostgreSQL Memory Architecture

A practical way to think about PostgreSQL memory is:

```text
                  PostgreSQL Memory
                         │
             ┌───────────┴───────────┐
             │                       │
        Shared Memory           Process-local Memory
             │                       │
       Shared Buffers              work_mem
       WAL-related memory          maintenance_work_mem
       Locks/shared state          query execution memory
       Other shared state          local backend state
```

This distinction is important.

### Shared memory

Used across PostgreSQL processes.

### Process-local memory

Used by an individual backend/worker for operations such as:

```text
Sort
Hash
Join
Aggregation
Maintenance
```

This explains why memory sizing cannot be based only on `shared_buffers`.

---

# 31. Operating-System Cache

PostgreSQL does not operate in isolation from the OS.

A simplified read path is:

```text
Backend
   │
   ▼
Shared Buffers
   │
   ▼
Operating-System Cache
   │
   ▼
Storage
```

Therefore PostgreSQL memory architecture should be designed together with:

```text
Server RAM
OS filesystem cache
Concurrent connections
work_mem
maintenance workloads
shared_buffers
```

rather than assigning all RAM to PostgreSQL shared buffers.

---

# 32. PostgreSQL and OS IPC

Because PostgreSQL uses multiple processes, it relies on operating-system inter-process communication mechanisms.

PostgreSQL 18 documents use of:

```text
Shared memory
Semaphores
Dynamic shared memory
```

and related kernel resources. ([PostgreSQL][8])

This matters especially for:

```text
Large servers
High connection counts
Parallel processing
Multiple PostgreSQL instances
```

---

# 33. Transaction Architecture

At a simplified level:

```text
BEGIN
  │
  ▼
SQL Operations
  │
  ├── Generate WAL
  ├── Modify buffers
  └── Maintain transaction state
  │
  ▼
COMMIT
  │
  ▼
Commit becomes durable
```

PostgreSQL combines:

```text
Transactions
MVCC
WAL
Locks
Snapshots
```

to provide transactional consistency.

We'll cover this deeply in **Topic 16 — Transactions & Concurrency**.

---

# 34. MVCC Architecture

PostgreSQL uses **Multi-Version Concurrency Control (MVCC)**.

Instead of simply overwriting a row version in place from the logical transaction perspective, PostgreSQL can maintain different tuple versions.

Simplified:

```text
Original Row
Customer = ABC
     │
     │ UPDATE
     ▼
Old Tuple Version
     +
New Tuple Version
Customer = XYZ
```

Different transactions may temporarily see different versions according to their snapshots.

This architecture helps readers and writers operate concurrently, but also creates the need for vacuuming old/dead tuple versions.

---

# 35. Replication Architecture

PostgreSQL's architecture also supports replication.

A basic physical streaming-replication model is:

```text
               PRIMARY
                  │
                  │ WAL
                  ▼
             WAL Sender
                  │
             Network
                  │
                  ▼
             WAL Receiver
                  │
                  ▼
               STANDBY
```

WAL is therefore used not only for crash recovery but also as the foundation for physical replication.

Detailed replication belongs in **Topic 5**.

---

# 36. HA Architecture

PostgreSQL itself provides the core replication mechanisms, but complete HA architectures normally require additional orchestration/design around them.

Conceptually:

```text
              Applications
                   │
                   ▼
             Endpoint / Proxy
                   │
          ┌────────┴────────┐
          ▼                 ▼
       Primary           Standby
          │                 ▲
          │       WAL       │
          └─────────────────┘
                   │
                   ▼
            HA Management
```

HA design must address:

```text
Failure detection
Leader/primary selection
Promotion
Client routing
Fencing
Split-brain prevention
RPO/RTO
```

Those are covered in **Topic 6 — HA & Failover Architecture**.

---

# 37. Complete PostgreSQL Architecture Diagram

Putting the pieces together:

```text
 ┌─────────────────────────────────────────────────────────────┐
 │                       CLIENT LAYER                          │
 │                                                             │
 │   Application     Microservice      psql       BI / ETL    │
 └────────┬───────────────┬──────────────┬─────────────┬───────┘
          │               │              │             │
          └───────────────┴──────┬───────┴─────────────┘
                                 │
                         TCP / Unix Socket
                                 │
                                 ▼
 ┌─────────────────────────────────────────────────────────────┐
 │                     POSTGRESQL 18 INSTANCE                  │
 │                                                             │
 │                    Primary postgres process                 │
 │                              │                              │
 │             ┌────────────────┼────────────────┐             │
 │             │                │                │             │
 │             ▼                ▼                ▼             │
 │        Backend          Backend          Backend            │
 │        Process          Process          Process            │
 │             │                │                │             │
 │             └────────────────┼────────────────┘             │
 │                              │                              │
 │                              ▼                              │
 │                     QUERY PROCESSING                        │
 │                                                             │
 │       Parser → Rewrite/Analysis → Planner → Executor        │
 │                              │                              │
 │                              ▼                              │
 │                       SHARED MEMORY                         │
 │                                                             │
 │           ┌──────────────────┼──────────────────┐           │
 │           ▼                  ▼                  ▼           │
 │     Shared Buffers      WAL-related        Shared State     │
 │                           Memory                            │
 │                                                             │
 │                     BACKGROUND PROCESSES                    │
 │                                                             │
 │   Checkpointer   Background Writer   WAL Writer             │
 │   Autovacuum     WAL Archiver        Replication Workers    │
 │   Other Auxiliary / Background Workers                      │
 └──────────────────────────────┬──────────────────────────────┘
                                │
                                ▼
 ┌─────────────────────────────────────────────────────────────┐
 │                       STORAGE LAYER                         │
 │                                                             │
 │                           PGDATA                            │
 │                                                             │
 │      base/       global/       pg_wal/       pg_tblspc/    │
 │        │                           │                        │
 │        ▼                           ▼                        │
 │    Data Files                    WAL Files                  │
 │        │                                                    │
 │        ├── Heap                                             │
 │        ├── Indexes                                          │
 │        ├── TOAST                                            │
 │        ├── FSM                                              │
 │        └── Visibility Map                                   │
 └─────────────────────────────────────────────────────────────┘
```

---

# 38. PostgreSQL 18 Architecture-Specific Point: Asynchronous I/O

One PostgreSQL 18 architectural change worth knowing is the introduction of an **asynchronous I/O (AIO) subsystem**. PostgreSQL 18's release notes state that AIO can improve operations including sequential scans, bitmap heap scans, and vacuum operations. ([PostgreSQL][9])

This is important when studying PostgreSQL 18 because the traditional simplified model:

```text
PostgreSQL Process
       │
       ▼
Synchronous I/O
       │
       ▼
Storage
```

is no longer sufficient for understanding every I/O path.

At a higher level PostgreSQL 18 can perform supported I/O through its newer asynchronous infrastructure, allowing useful work and I/O operations to overlap more effectively.

This should be revisited in detail under **Performance Tuning** and **Storage & Capacity Planning**.

---

# 39. Architecture Troubleshooting View

Understanding architecture helps a DBA quickly identify which layer is having trouble.

| Problem              | Architecture area to investigate          |
| -------------------- | ----------------------------------------- |
| Cannot connect       | Network / listener / authentication       |
| Too many connections | Backend processes / pooling               |
| High CPU             | Queries / planner / executor              |
| High memory          | Shared + backend/worker memory            |
| Slow reads           | Buffers / OS cache / storage / query plan |
| High disk I/O        | Queries / checkpoints / vacuum / storage  |
| WAL growth           | Transactions / replication / archiving    |
| Replication lag      | WAL sender/receiver/replay/network        |
| Dead tuples          | MVCC / autovacuum                         |
| Blocking             | Transactions / locks                      |
| Database won't start | PGDATA/config/WAL/storage/OS resources    |

This is why architecture knowledge is foundational for troubleshooting.

---

# 40. Architecture View for a DBA

A DBA should be able to mentally follow this path:

```text
                  SQL Request
                      │
                      ▼
                  Connection
                      │
                      ▼
               Backend Process
                      │
                      ▼
              Parser / Planner
                      │
                      ▼
                  Executor
                      │
                      ▼
               Shared Buffers
                      │
             ┌────────┴────────┐
             ▼                 ▼
          WAL Path          Data Path
             │                 │
             ▼                 ▼
           pg_wal          Data Files
             │                 │
             └────────┬────────┘
                      ▼
                   Storage
```

If you understand this flow, many later PostgreSQL topics become much easier.

---

# 41. Key Architecture Concepts to Remember

| Component                  | Main purpose                              |
| -------------------------- | ----------------------------------------- |
| `postgres` primary process | Manages server instance and processes     |
| Backend process            | Handles a client session                  |
| Shared buffers             | Caches database pages                     |
| WAL                        | Records changes for durability/recovery   |
| WAL writer                 | Helps write WAL                           |
| Background writer          | Gradually writes dirty buffers            |
| Checkpointer               | Coordinates checkpoint activity           |
| Autovacuum                 | MVCC maintenance/statistics work          |
| Parser                     | Interprets SQL                            |
| Planner                    | Chooses execution strategy                |
| Executor                   | Executes the selected plan                |
| `PGDATA`                   | Cluster's main data directory             |
| `base/`                    | Per-database physical files               |
| `pg_wal/`                  | WAL files                                 |
| `pg_tblspc/`               | Tablespace links                          |
| Heap                       | Normal table storage                      |
| TOAST                      | Large-value storage mechanism             |
| FSM                        | Tracks available page space               |
| Visibility Map             | Tracks page visibility/freeze information |

---

## PostgreSQL 18 official references

The most important official documentation for this topic is [PostgreSQL 18 Architectural Fundamentals](https://www.postgresql.org/docs/18/tutorial-arch.html?utm_source=chatgpt.com) for the basic client/server and process architecture.

For the internal SQL path, use [PostgreSQL 18 Overview of PostgreSQL Internals](https://www.postgresql.org/docs/18/overview.html?utm_source=chatgpt.com). It covers connection establishment and the parser, planner/optimizer, and executor stages. ([PostgreSQL][5])

For memory, background writer, I/O and worker-process configuration, use [PostgreSQL 18 Resource Consumption](https://www.postgresql.org/docs/18/runtime-config-resource.html?utm_source=chatgpt.com). ([PostgreSQL][4])

For physical architecture, use [PostgreSQL 18 Database Physical Storage](https://www.postgresql.org/docs/18/storage.html?utm_source=chatgpt.com) and [PostgreSQL 18 Database File Layout](https://www.postgresql.org/docs/18/storage-file-layout.html?utm_source=chatgpt.com). These cover `PGDATA`, database files, TOAST, free-space maps, visibility maps and page layout. ([PostgreSQL][2])

Finally, the [PostgreSQL 18 Release Notes](https://www.postgresql.org/docs/18/release-18.html?utm_source=chatgpt.com) are important for the PostgreSQL 18-specific asynchronous I/O architecture. ([PostgreSQL][9])

**Tracker: 3 / 29 completed — 26 pending.** Topic 3, **PostgreSQL Architecture Overview & Diagram**, is now completed.

[1]: https://www.postgresql.org/docs/18/glossary.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Appendix M. Glossary"
[2]: https://www.postgresql.org/docs/18/storage-file-layout.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 66.1. Database File Layout"
[3]: https://www.postgresql.org/docs/18/monitoring-ps.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 27.1. Standard Unix Tools"
[4]: https://www.postgresql.org/docs/18/runtime-config-resource.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.4. Resource Consumption"
[5]: https://www.postgresql.org/docs/current/overview.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 51. Overview of PostgreSQL Internals"
[6]: https://www.postgresql.org/docs/18/runtime-config-file-locations.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.2. File Locations"
[7]: https://www.postgresql.org/docs/18/storage.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 66. Database Physical Storage"
[8]: https://www.postgresql.org/docs/18/kernel-resources.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 18.4. Managing Kernel Resources"
[9]: https://www.postgresql.org/docs/18/release-18.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: E.5. Release 18"
