# Topic 10 — PostgreSQL Connection Management

PostgreSQL connection management is a major production architecture topic because PostgreSQL uses a **process-per-connection model**. 

A system may have plenty of CPU and memory and still become unstable because too many application connections reach the database at once.

The key architecture principle is:

> **Application concurrency should not translate directly into PostgreSQL backend concurrency.**

A good production design controls connections at several layers:

**Application → connection pool → PgBouncer → PostgreSQL**

---

## 1. How PostgreSQL connections work

When a client connects to PostgreSQL:

```text
Application
    |
    | TCP connection
    v
PostgreSQL Listener
    |
    | authenticate
    | create backend
    v
PostgreSQL Backend Process
    |
    +--> parse SQL
    +--> execute queries
    +--> manage transaction
    +--> access shared buffers
    +--> return results
```

Unlike systems that use a small number of worker threads for thousands of clients, PostgreSQL normally creates **one backend process for each database connection**.

For example:

```text
500 application connections
        ↓
~500 PostgreSQL backend processes
```

You can see them with:

```sql
SELECT
    pid,
    usename,
    datname,
    client_addr,
    state,
    backend_start,
    xact_start,
    query_start
FROM pg_stat_activity;
```

This architecture works very well, but it means connections are **not free resources**.

---

# 2. Why PostgreSQL connections are expensive

Each backend consumes resources.

At a high level:

```text
Connection
   |
   +-- Backend OS process
   +-- Backend-local memory
   +-- Shared-memory participation
   +-- File descriptors
   +-- Locks / transaction state
   +-- Kernel resources
   +-- CPU scheduling
```

Memory is particularly important.

A common mistake is thinking:

```text
max_connections = 1000
```

means PostgreSQL simply reserves a small amount of memory for 1,000 sockets.

Actual workload memory can be much larger because active sessions can perform operations using settings such as:

```text
work_mem
temp_buffers
maintenance_work_mem
```

`work_mem` is especially important because it is **not simply per connection**. Multiple execution nodes in one query can each consume work memory.

For example, conceptually:

```text
Connection
   |
   +-- Sort       → work_mem
   +-- Hash Join  → work_mem
   +-- Sort       → work_mem
```

Parallel query can increase the picture further.

Therefore:

```text
500 connections × 64 MB work_mem
```

is not a safe way to calculate memory, but it does demonstrate why large connection limits can become dangerous.

---

# 3. `max_connections`

PostgreSQL controls the maximum number of concurrent database connections primarily through:

```conf
max_connections = 100
```

Check it with:

```sql
SHOW max_connections;
```

or:

```sql
SELECT setting
FROM pg_settings
WHERE name = 'max_connections';
```

A common architectural mistake is:

```text
Application hits connection limit
        ↓
Increase max_connections
        ↓
Problem appears solved
```

Often the real problem is:

```text
Too many application instances
×
oversized application pools
×
poor connection reuse
```

Increasing `max_connections` can simply move the failure from:

```text
connection rejected
```

to:

```text
CPU saturation
memory pressure
latency explosion
database instability
```

So treat `max_connections` as a **capacity boundary**, not a performance knob.

---

# 4. Reserved PostgreSQL connections

You should never design production capacity assuming applications can consume every connection.

PostgreSQL needs connection capacity for administration and recovery.

The familiar setting is:

```conf
superuser_reserved_connections
```

For example:

```conf
max_connections = 200
superuser_reserved_connections = 5
```

The intent is that ordinary users cannot consume the final reserved slots.

This matters during incidents.

Imagine:

```text
max_connections = 200
application connections = 200
```

Now the DBA tries:

```bash
psql
```

and cannot connect.

That is exactly when administrative access is most important.

Newer PostgreSQL versions also support a more flexible reserved-connection mechanism through the `pg_use_reserved_connections` role and `reserved_connections`.

The architectural idea is the important part:

```text
Total DB capacity
      |
      +-- Application capacity
      +-- Operational / privileged reserve
      +-- Background / infrastructure allowance
```

Do not give 100% of database connection capacity to the application tier.

---

# 5. PostgreSQL connection lifecycle

A simplified connection lifecycle looks like this:

```text
Client needs database
        |
        v
DNS resolution
        |
        v
TCP connection
        |
        v
TLS negotiation (if enabled)
        |
        v
PostgreSQL authentication
        |
        v
Backend established
        |
        v
Session initialization
        |
        v
Execute SQL
        |
        v
Commit / rollback
        |
        v
Connection stays idle
        |
        v
Reused or disconnected
```

Creating connections repeatedly adds latency and load.

Bad application behavior:

```text
HTTP request
   |
   +--> Open DB connection
   +--> Query
   +--> Close connection
```

Repeated thousands of times.

Better:

```text
Application
     |
     v
Connection Pool
  | | | | |
  v v v v v
PostgreSQL
```

The application borrows an existing connection and returns it after use.

---

# 6. Connection states in `pg_stat_activity`

A useful production query:

```sql
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;
```

Common states include:

```text
active
idle
idle in transaction
idle in transaction (aborted)
```

These mean very different things.

### `active`

The backend is executing work.

```text
Application
    |
    v
Backend → executing query
```

A large active count may indicate real database concurrency.

---

### `idle`

The connection exists but currently isn't running a query.

```text
Application Pool
      |
      v
 PostgreSQL backend
      |
      +-- waiting
```

Some idle connections are normal.

Thousands of them may indicate oversized pools.

---

### `idle in transaction`

This is much more dangerous.

Example:

```sql
BEGIN;

SELECT * FROM orders WHERE id = 100;
```

Then the application does nothing.

The session remains:

```text
idle in transaction
```

It can retain transaction state and interfere with vacuum cleanup.

Check:

```sql
SELECT
    pid,
    usename,
    now() - xact_start AS transaction_age,
    state,
    query
FROM pg_stat_activity
WHERE state LIKE 'idle in transaction%'
ORDER BY xact_start;
```

---

# 7. Connection storms

A **connection storm** happens when many clients attempt to create database connections simultaneously.

A common Kubernetes example:

```text
                 PostgreSQL restart
                        |
                        v
              Connections disappear
                        |
        +---------------+---------------+
        |               |               |
      Pod 1           Pod 2           Pod 3
        |               |               |
      retry           retry           retry
        |               |               |
        +---------------+---------------+
                        |
                        v
                  PostgreSQL
```

Now imagine 100 pods.

Each pod has:

```text
pool size = 20
```

Potential reconnect attempts:

```text
100 × 20 = 2,000
```

The database may have:

```text
max_connections = 300
```

This can overwhelm the server immediately after recovery.

---

# 8. Why connection storms are dangerous

A connection storm creates multiple forms of pressure at the same time:

```text
Connection Storm
      |
      +--> TCP setup
      +--> TLS negotiation
      +--> Authentication
      +--> Backend creation
      +--> Memory allocation
      +--> CPU scheduling
      +--> Application retries
```

Then something worse can happen.

```text
DB becomes slow
     ↓
application timeout
     ↓
application retries
     ↓
more connections
     ↓
DB becomes slower
     ↓
more retries
```

This is a **positive feedback loop**.

Production systems therefore need:

```text
connection pooling
+
retry backoff
+
retry jitter
+
connection limits
```

---

# 9. Retry backoff and jitter

Do not let every application reconnect immediately.

Bad:

```text
Connection failed

retry 1 → immediately
retry 2 → immediately
retry 3 → immediately
```

Better:

```text
retry 1 → ~1 sec
retry 2 → ~2 sec
retry 3 → ~4 sec
retry 4 → ~8 sec
```

Add random jitter:

```text
Pod A → 1.1 sec
Pod B → 1.7 sec
Pod C → 1.3 sec
Pod D → 1.9 sec
```

Without jitter, every pod can wake up together:

```text
1 sec  → 500 retries
2 sec  → 500 retries
4 sec  → 500 retries
```

With jitter, the reconnect load is distributed.

---

# 10. Why connection pooling exists

Suppose you have:

```text
100 application instances
```

and each instance has:

```text
30 DB connections
```

Potential database connections:

```text
100 × 30 = 3,000
```

But maybe only 80–150 database operations are actually executing simultaneously.

Without pooling architecture:

```text
3,000 client connections
        |
        v
3,000 PostgreSQL backends
```

With an intermediary pooler:

```text
3,000 client-side connections
        |
        v
     PgBouncer
        |
        v
100–200 PostgreSQL connections
```

That is the core value of tools such as [PgBouncer](https://www.pgbouncer.org/?utm_source=chatgpt.com).

---

# 11. PgBouncer architecture

PgBouncer sits between applications and PostgreSQL.

```text
Application Pods
       |
       | many client connections
       v
+-------------------+
|     PgBouncer     |
|                   |
| client connections|
|        ↓          |
| server pool       |
+---------+---------+
          |
          | fewer connections
          v
+-------------------+
|    PostgreSQL     |
+-------------------+
```

Applications connect to PgBouncer rather than directly to PostgreSQL.

Example conceptually:

```text
Applications:       2,000 connections
PgBouncer → DB:       150 connections
```

The exact ratio depends on workload characteristics.

---

# 12. PgBouncer terminology

It helps to distinguish:

### Client connection

```text
Application → PgBouncer
```

### Server connection

```text
PgBouncer → PostgreSQL
```

These numbers do not have to be equal.

That is the key architectural benefit.

```text
Client concurrency
        ≠
Database backend concurrency
```

---

# 13. PgBouncer pooling modes

PgBouncer has three major pooling modes:

```text
session
transaction
statement
```

For architecture discussions, **session vs transaction pooling** is usually the most important comparison.

---

# 14. Session pooling

With session pooling:

```text
Client connects
      |
      v
PgBouncer assigns DB connection
      |
      v
Client keeps same DB connection
      |
      v
Client disconnects
      |
      v
DB connection returns to pool
```

Conceptually:

```text
App Session A ───────────────→ DB Connection 1

App Session B ───────────────→ DB Connection 2
```

The backend is effectively dedicated for the lifetime of the client session.

### Advantages

It has high PostgreSQL compatibility because session state behaves naturally.

Things such as:

```sql
SET search_path = ...
```

temporary objects and other session-scoped behavior are easier to reason about.

### Disadvantage

Pooling efficiency is lower.

If 1,000 application clients remain connected, session pooling can still require a large backend pool depending on configuration and demand.

---

# 15. Transaction pooling

Transaction pooling is much more aggressive.

A PostgreSQL server connection is assigned only for a transaction.

```text
Client A
   |
   | BEGIN
   v
DB Connection 1
   |
   | COMMIT
   v
returned to pool
```

Then:

```text
Client B
   |
   | BEGIN
   v
DB Connection 1
```

So one PostgreSQL connection can serve many application sessions over time.

Example:

```text
Client A ----TX----\
Client B ------TX---\
Client C --TX-------->  PgBouncer → DB connection pool
Client D --------TX-/
```

This provides much stronger multiplexing.

---

# 16. Session vs transaction pooling

| Area                        | Session Pooling       | Transaction Pooling  |
| --------------------------- | --------------------- | -------------------- |
| Backend assignment          | Entire client session | Transaction duration |
| Pooling efficiency          | Lower                 | Higher               |
| Backend reuse               | Moderate              | High                 |
| Session-state compatibility | Strong                | More restrictions    |
| Connection reduction        | Moderate              | Often significant    |
| App compatibility testing   | Easier                | Essential            |
| Good for high client counts | Sometimes             | Often                |

A useful mental model:

```text
Session pooling
Client : Backend
roughly more persistent

Transaction pooling
Many clients : smaller backend pool
```

---

# 17. Transaction pooling compatibility

Transaction pooling changes an important assumption:

> The application cannot assume that consecutive transactions use the same PostgreSQL backend.

For example:

```text
Transaction 1
Client → Backend 17

Transaction 2
Client → Backend 42
```

Therefore session-dependent behavior needs careful review.

Examples include:

```text
session-level SET state
temporary tables
session advisory locks
LISTEN / NOTIFY usage patterns
some prepared-statement behaviors depending on configuration/version
```

Do not simply place PgBouncer in transaction mode in front of an application and assume everything will work.

Test application behavior explicitly.

---

# 18. Application connection pools

Even with PgBouncer, applications often have their own pool.

For example:

```text
Java Application
       |
       v
     HikariCP
       |
       v
    PgBouncer
       |
       v
   PostgreSQL
```

Or:

```text
Python App
    |
 SQLAlchemy pool
    |
 PgBouncer
    |
 PostgreSQL
```

This means you actually have **multiple queues/pools**.

```text
Request
   |
   v
Application pool
   |
   v
PgBouncer client queue
   |
   v
PgBouncer server pool
   |
   v
PostgreSQL
```

All layers must be sized together.

---

# 19. The classic connection-sizing mistake

Suppose:

```text
50 Kubernetes pods
```

Each has:

```text
minimum pool = 10
maximum pool = 50
```

Worst-case potential:

```text
50 × 50
= 2,500 connections
```

If PostgreSQL has:

```text
max_connections = 500
```

the architecture clearly does not fit.

Adding autoscaling makes this more important.

Suppose:

```text
Current pods = 20
Maximum pods = 100

Pool per pod = 20
```

At maximum scale:

```text
100 × 20 = 2,000
```

You must size for the **deployment's possible state**, not only today's pod count.

---

# 20. Connection-budget design

A better architecture starts with the database.

Assume, purely as an example:

```text
max_connections = 300
```

Create a budget:

```text
300 total
 |
 +-- 20 administration / emergency
 +-- 20 monitoring / infrastructure
 +-- 10 replication / operational allowance
 +-- 50 safety / capacity margin
 |
 +-- 200 application connections
```

Now the application architecture must fit inside:

```text
Application DB budget = 200
```

Not 300.

---

# 21. Application connection sizing

Suppose the application tier can scale to:

```text
40 pods
```

and the database application budget is:

```text
200 connections
```

A simple upper bound would be:

```text
200 / 40
= 5 connections per pod
```

But that is only the starting point.

You also need to consider:

```text
Number of services
Number of replicas
Autoscaling maximum
Background workers
Scheduled jobs
Read replicas
Failover behavior
Pooler topology
Actual query concurrency
Query latency
```

A useful architecture formula is:

```text
Total potential connections
=
Σ(instance count × pool max per instance)
```

across all services reaching the database.

---

# 22. Size from concurrency, not traffic volume

A service may handle:

```text
5,000 requests/sec
```

That does **not** automatically mean it needs thousands of DB connections.

Suppose each request spends only:

```text
20 ms
```

actively using the database.

Approximate concurrent DB demand:

```text
5,000 × 0.020
= 100
```

This is closely related to Little's Law:

```text
Concurrency ≈ Throughput × Service Time
```

For example:

```text
2,000 DB operations/sec
×
0.025 sec average DB hold time

≈ 50 concurrent DB operations
```

This is far more useful for sizing than:

```text
We have 100 pods, so let's give every pod 50 connections.
```

---

# 23. More connections do not automatically mean more throughput

Suppose PostgreSQL runs best around:

```text
100 actively competing DB operations
```

Then increasing concurrency to:

```text
500
```

may not give:

```text
5× throughput
```

Instead:

```text
More active queries
      |
      +--> CPU contention
      +--> lock contention
      +--> memory pressure
      +--> cache pressure
      +--> I/O queueing
      +--> scheduler overhead
      |
      v
Higher latency
```

This can create a latency curve like:

```text
Latency
  ^
  |                         /
  |                      __/
  |                   __/
  |                __/
  |______________/
  +----------------------------> Concurrency
```

Eventually, adding concurrency makes everything slower.

The goal is not:

```text
maximum number of DB connections
```

It is:

```text
maximum sustainable throughput
at acceptable latency
```

---

# 24. Connection queueing can be healthy

Teams sometimes think:

```text
Waiting for a connection = bad
```

Not necessarily.

Controlled queueing can protect PostgreSQL.

```text
1,000 incoming requests
         |
         v
Application / PgBouncer queue
         |
         | controlled concurrency
         v
100 DB backends
         |
         v
PostgreSQL
```

This can be much healthier than:

```text
1,000 incoming requests
         |
         v
1,000 active DB backends
         |
         v
CPU / memory / I/O collapse
```

The pool acts as a **concurrency control mechanism**, not merely a connection cache.

---

# 25. Important PostgreSQL timeouts

Connection management is also about making sure bad or abandoned work does not consume capacity forever.

Important settings include:

```text
connect_timeout
statement_timeout
lock_timeout
idle_in_transaction_session_timeout
idle_session_timeout
```

Some are server settings and some are client/libpq parameters.

---

# 26. `connect_timeout`

This controls how long a client waits while establishing a connection.

Conceptually:

```text
Application
    |
    | attempting connection
    |
    |---- timeout ---->
```

Without a reasonable timeout, requests can wait too long when the database or network is unavailable.

For libpq-style connection strings:

```text
connect_timeout=5
```

Example only; production values should match the environment.

---

# 27. `statement_timeout`

This limits how long a SQL statement may run.

```sql
SET statement_timeout = '30s';
```

A runaway query can otherwise hold a connection for a long time.

```text
Long query
    |
    +--> holds backend
    +--> consumes resources
    +--> potentially holds locks
    +--> reduces available pool capacity
```

A timeout protects more than query latency—it also protects **connection capacity**.

---

# 28. `lock_timeout`

Suppose:

```text
Transaction A
    |
    +-- holds lock

Transaction B
    |
    +-- waits
```

Without a timeout, B may wait for an unacceptable amount of time.

Example:

```sql
SET lock_timeout = '5s';
```

Again, choose values based on workload requirements rather than copying generic numbers.

---

# 29. `idle_in_transaction_session_timeout`

This is one of the most useful protections.

```conf
idle_in_transaction_session_timeout = '60s'
```

It protects against applications doing:

```text
BEGIN
   |
query
   |
application stops doing DB work
   |
transaction remains open
```

These sessions are especially problematic because long-running transactions can interfere with PostgreSQL vacuum behavior and retain resources/state.

---

# 30. `idle_session_timeout`

This can terminate sessions that remain idle beyond a configured duration.

However, use it carefully with connection pools.

If PostgreSQL closes idle connections underneath a pool, the pool needs to detect and replace them cleanly.

You therefore need coordination between:

```text
Database timeout
Pool connection lifetime
Pool idle timeout
Health checks
```

---

# 31. Pool timeout vs SQL timeout

These solve different problems.

### Pool acquisition timeout

```text
Request
   |
   v
Waiting for available connection
   |
   X timeout
```

Meaning:

> I could not obtain a database connection quickly enough.

### Statement timeout

```text
Request
   |
   v
Got connection
   |
   v
SQL executing
   |
   X timeout
```

Meaning:

> The SQL itself took too long.

These should be monitored separately.

A high pool timeout rate may indicate:

```text
pool too small
DB too slow
connections held too long
traffic spike
connection leak
```

Increasing the pool size blindly may make the database problem worse.

---

# 32. Connection leaks

A classic application bug:

```text
connection = pool.get()

execute query

exception occurs

connection never returned
```

Over time:

```text
Pool size = 20

available:
20
19
18
17
...
2
1
0
```

Then requests start waiting for connections.

Applications should follow patterns that guarantee release:

```text
Acquire
   |
   v
try
   |
execute
   |
finally
   |
   v
Release
```

Framework-specific mechanisms should normally handle this.

---

# 33. Monitor pool utilization

Do not monitor only PostgreSQL.

Monitor the entire path.

```text
Application
    |
    +-- pool active
    +-- pool idle
    +-- pool pending
    +-- acquisition latency
    +-- acquisition timeout
    |
    v
PgBouncer
    |
    +-- client active
    +-- client waiting
    +-- server active
    +-- server idle
    +-- wait time
    |
    v
PostgreSQL
    |
    +-- connections
    +-- active sessions
    +-- idle sessions
    +-- idle transactions
    +-- query latency
    +-- locks
    +-- CPU
    +-- memory
```

This tells you **where queueing occurs**.

---

# 34. Useful PostgreSQL connection queries

Connections by state:

```sql
SELECT
    state,
    count(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;
```

Connections by application:

```sql
SELECT
    application_name,
    state,
    count(*)
FROM pg_stat_activity
GROUP BY application_name, state
ORDER BY count(*) DESC;
```

Connections by user:

```sql
SELECT
    usename,
    count(*)
FROM pg_stat_activity
GROUP BY usename
ORDER BY count(*) DESC;
```

Connections by database:

```sql
SELECT
    datname,
    count(*)
FROM pg_stat_activity
GROUP BY datname
ORDER BY count(*) DESC;
```

Long transactions:

```sql
SELECT
    pid,
    usename,
    application_name,
    state,
    now() - xact_start AS transaction_age,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

---

# 35. HA changes connection management

Now consider a highly available PostgreSQL architecture:

```text
             Applications
                  |
                  v
              PgBouncer
                  |
                  v
          Primary PostgreSQL
                  |
               streaming
                  |
                  v
             Standby
```

If the primary fails:

```text
Primary X
    |
Standby promoted
    |
New Primary
```

The connection layer must discover and use the new primary.

That makes connection management an **HA problem**, not merely a performance problem.

---

# 36. What happens during failover?

Imagine:

```text
Old Primary
    |
    X failure

Applications
    |
    +--> existing connections break
    +--> transactions fail
    +--> reconnect attempts start
```

Meanwhile:

```text
Standby
   |
   v
Promotion
   |
   v
New Primary
```

The application needs to reconnect through a mechanism that points it to the correct writable node.

Possible architectures include:

```text
DNS
Load balancer
HAProxy
PgBouncer
Kubernetes Service
Cloud DB endpoint
PostgreSQL HA manager
```

The exact solution depends on the platform.

---

# 37. HA-aware pooling

A robust design might look like:

```text
                  Applications
                       |
                       v
                +-------------+
                |  PgBouncer  |
                +------+------+
                       |
                       v
                HA routing layer
                  /         \
                 /           \
                v             v
           PostgreSQL      PostgreSQL
             Primary         Standby
```

Or the managed database platform may provide a stable writer endpoint:

```text
Application
     |
 PgBouncer
     |
Writer Endpoint
     |
Current Primary
```

The key requirement is:

> The pool must not remain permanently attached to the old primary after failover.

---

# 38. Connection lifetime matters in HA

Long-lived connections are efficient but can complicate failover and infrastructure changes.

A pool may support concepts such as:

```text
max connection lifetime
idle timeout
health check
connection validation
```

For example:

```text
Connection created
      |
      v
Used repeatedly
      |
      v
Lifetime threshold
      |
      v
Retire connection
      |
      v
Create fresh connection
```

This can help applications gradually pick up:

```text
new DNS targets
new database nodes
certificate changes
routing changes
```

But aggressive recycling can itself create excessive connection churn.

---

# 39. Failover retry design

A dangerous pattern:

```text
Primary fails
    |
10,000 requests fail
    |
Every request retries immediately
    |
10,000 connection attempts
    |
New primary becomes overloaded
```

Better:

```text
Primary fails
      |
      v
connections fail
      |
      v
controlled retries
      |
      +-- exponential backoff
      +-- jitter
      +-- retry limit
      |
      v
new primary
```

The recovery path must be capacity-tested.

---

# 40. PgBouncer itself needs HA thinking

If the architecture is:

```text
Application
     |
     v
PgBouncer
     |
     v
PostgreSQL
```

and there is only one PgBouncer instance:

```text
PgBouncer failure
      |
      X
Applications cannot reach DB
```

You have created another single point of failure.

Possible approaches include:

```text
PgBouncer per application pod
```

or:

```text
multiple PgBouncer instances
behind service/load balancing
```

or platform-managed pooling.

Each architecture has different operational characteristics.

---

# 41. Sidecar/local PgBouncer architecture

One pattern is:

```text
Pod
+----------------------------+
|                            |
| Application → PgBouncer    |
|                            |
+--------------+-------------+
               |
               v
           PostgreSQL
```

Advantages can include:

```text
local connection hop
isolated pool behavior
application-specific configuration
```

But if there are many pods, each PgBouncer instance may maintain its own server pool.

For example:

```text
100 pods
×
PgBouncer default_pool_size 20
=
potentially large DB connection footprint
```

So PgBouncer does not automatically solve connection scaling if its own pools are multiplied across many instances.

---

# 42. Centralized PgBouncer architecture

Another design:

```text
       App Pods
      /   |   \
     /    |    \
    v     v     v
+---------------------+
| PgBouncer Cluster   |
+----------+----------+
           |
           v
      PostgreSQL
```

This can provide stronger global multiplexing.

But now PgBouncer becomes shared infrastructure and needs:

```text
HA
capacity planning
monitoring
network design
security
deployment strategy
```

There is no universally correct topology.

---

# 43. Read/write connection architecture

Some systems separate:

```text
writes → primary
reads  → replicas
```

Example:

```text
                 Application
                  /       \
                 /         \
                v           v
        Writer Pool      Reader Pool
             |               |
             v               v
          Primary         Replicas
```

Now connection budgeting must be done independently.

Example:

```text
Primary:
  max_connections = X
  application budget = Y

Replica:
  max_connections = A
  application budget = B
```

Do not treat the PostgreSQL cluster as having one global connection limit. Each PostgreSQL server has its own capacity.

---

# 44. Production capacity design example

Consider:

```text
80 application pods
3 services
PostgreSQL primary
2 read replicas
PgBouncer
```

Suppose testing shows the primary can sustain the required workload with:

```text
120 active server connections
```

Instead of configuring:

```text
80 pods × 30 connections
= 2,400 PostgreSQL connections
```

you could design:

```text
Applications
    |
    | thousands of logical/client connections
    v
PgBouncer
    |
    | ~controlled server pool
    v
PostgreSQL
```

For illustration:

```text
PostgreSQL max_connections = 250

Operational reserve       = 30
Infrastructure            = 20
Safety margin             = 50
Application server budget = 150
```

Then PgBouncer might be designed so application traffic cannot exceed that server-side budget.

The exact values must come from load testing and production behavior.

---

# 45. Capacity should be based on workload testing

Do not choose:

```text
pool_size = 100
```

because another company or blog uses 100.

Test:

```text
DB connections = 20
      ↓
throughput / latency

DB connections = 40
      ↓
throughput / latency

DB connections = 80
      ↓
throughput / latency

DB connections = 120
      ↓
throughput / latency

DB connections = 200
      ↓
throughput / latency
```

You may discover:

```text
Connections     Throughput      p95 latency
------------------------------------------------
20              low             good
40              better          good
80              strong          good
120             slightly better rising
200             same            poor
400             worse           terrible
```

Then perhaps the database's useful concurrency region is around:

```text
80–120
```

not 400.

This is why **load testing determines pool sizing**.

---

# 46. Connection headroom

Never plan normal production usage at:

```text
99% of max_connections
```

You need headroom for:

```text
traffic bursts
deployment overlap
autoscaling
maintenance
monitoring
administration
failover
background workers
unexpected workload
```

A deployment can temporarily double application instances.

For example:

```text
Normal:
50 pods

Rolling deployment:
50 old pods
+
50 new pods
```

If both generations establish pools:

```text
connection demand temporarily increases
```

Connection capacity design must account for deployment behavior.

---

# 47. Autoscaling trap

Suppose:

```text
10 pods
pool max = 20

DB potential = 200
```

Everything looks fine.

Then CPU autoscaling changes the application to:

```text
50 pods
```

Potential:

```text
50 × 20 = 1,000
```

Autoscaling the application may therefore accidentally **DDoS your own database**.

Architecture must align:

```text
HPA max replicas
×
pool max
≤
safe connection architecture
```

or place a global concurrency boundary such as PgBouncer between them.

---

# 48. Background jobs are often forgotten

Teams calculate:

```text
API service connections
```

but forget:

```text
worker processes
ETL
Airflow
reporting
cron jobs
migration jobs
monitoring
admin tools
BI tools
```

Production connection budget should include all consumers:

```text
                   PostgreSQL
                       |
        +--------------+-------------+
        |              |             |
       API          Workers          BI
        |              |             |
     Reports         ETL           Admin
```

Inventory every connection source.

---

# 49. Separate users can improve control

Instead of every workload using one database role:

```text
app_user
```

consider workload-specific roles where appropriate:

```text
api_user
worker_user
reporting_user
migration_user
monitoring_user
```

Then PostgreSQL can provide clearer visibility and controls.

For example:

```sql
ALTER ROLE reporting_user CONNECTION LIMIT 20;
```

This can prevent one workload from consuming the entire connection budget.

---

# 50. Database-level connection limits

PostgreSQL can also constrain connections to a database:

```sql
ALTER DATABASE reporting CONNECTION LIMIT 30;
```

And roles can have limits:

```sql
ALTER ROLE analytics_user CONNECTION LIMIT 10;
```

These are useful guardrails, though they should be designed carefully because connection pooling can change how connections map to users and databases.

---

# 51. Connection observability

At minimum, production dashboards should show:

```text
PostgreSQL
-----------
total connections
% max_connections used
active
idle
idle in transaction
long transactions
connection rate
connection failures

Application pool
----------------
active
idle
pending
max
acquisition latency
timeouts

PgBouncer
---------
client active
client waiting
server active
server idle
pool wait time
connection creation rate
```

A connection count alone is not enough.

For example:

```text
150 connections
```

could mean:

```text
145 idle + 5 active
```

or:

```text
150 active
```

Those are completely different situations.

---

# 52. Useful utilization metric

A simple capacity metric:

```text
Connection utilization
=
current connections / max_connections × 100
```

Example:

```text
180 / 250 × 100
= 72%
```

But also track:

```text
active / application connection budget
```

because reserved and operational connections should not be treated as normal application capacity.

---

# 53. Connection rate is important

Suppose total connections remain around:

```text
100
```

You might assume everything is healthy.

But imagine applications are constantly doing:

```text
connect
disconnect
connect
disconnect
```

Thousands of times per second.

The snapshot still shows approximately:

```text
100 connections
```

while the system experiences heavy connection churn.

Therefore monitor:

```text
connection count
AND
connection creation rate
```

---

# 54. Production anti-patterns

### Anti-pattern 1 — Huge `max_connections`

```text
Problem:
connections exhausted

Fix:
max_connections = 5000
```

This can convert a controlled rejection into database resource exhaustion.

---

### Anti-pattern 2 — Huge pool on every pod

```text
200 pods
×
100 connections
=
20,000 possible connections
```

Pool configuration must consider fleet size.

---

### Anti-pattern 3 — Pool size equals thread count

```text
200 application threads
→ 200 DB connections
```

Most threads may not use the database simultaneously.

---

### Anti-pattern 4 — No connection acquisition timeout

Requests wait indefinitely for pool capacity.

---

### Anti-pattern 5 — Immediate retry loops

```text
DB unavailable
→ retry
→ retry
→ retry
```

Creates connection storms.

---

### Anti-pattern 6 — `idle in transaction`

Applications hold transactions open while doing unrelated work.

---

### Anti-pattern 7 — No HA testing

Failover works at the PostgreSQL level, but applications cannot reconnect correctly.

---

### Anti-pattern 8 — Pooler becomes SPOF

```text
Highly available PostgreSQL
+
single PgBouncer
=
not actually highly available
```

---

# 55. Recommended production architecture

For a high-scale application, a common architecture is:

```text
                     Clients
                        |
                        v
                Load Balancer/API
                        |
                        v
               Application Pods
              /       |        \
             /        |         \
            v         v          v
       Local/App Connection Pools
             \        |         /
              \       |        /
               v      v       v
              PgBouncer Layer
                     |
                     |
             controlled DB
               concurrency
                     |
                     v
            PostgreSQL Primary
                  /      \
                 /        \
                v          v
           Replica 1    Replica 2
```

The layers solve different problems:

```text
Application pool
    → reuse connections locally

PgBouncer
    → multiplex clients and control DB concurrency

PostgreSQL
    → execute actual database work
```

---

# 56. Capacity design workflow

A practical architecture process is:

```text
1. Measure workload
       |
       v
2. Determine sustainable DB concurrency
       |
       v
3. Set PostgreSQL connection budget
       |
       v
4. Reserve operational headroom
       |
       v
5. Size PgBouncer server pools
       |
       v
6. Size application pools
       |
       v
7. Account for autoscaling
       |
       v
8. Account for deployments
       |
       v
9. Test failover
       |
       v
10. Load test connection storms
```

This order is important.

Do **not** start with:

```text
How many connections should each pod have?
```

Start with:

```text
How much concurrent database work can PostgreSQL safely sustain?
```

---

# 57. Interview scenario

### Question

You have:

```text
100 Kubernetes pods
pool max = 30
PostgreSQL max_connections = 500
```

What concerns you?

### Strong answer

Potential application connections are:

```text
100 × 30 = 3,000
```

which is far above PostgreSQL's connection limit.

I would first determine actual DB concurrency from workload measurements rather than simply increasing `max_connections`.

I would then:

```text
1. Establish a PostgreSQL connection budget.
2. Reserve connections for operations/admin/infrastructure.
3. Reduce per-pod pool sizing.
4. Consider PgBouncer, especially transaction pooling if compatible.
5. Account for HPA maximum replicas.
6. Add connection acquisition timeouts.
7. Implement exponential retry backoff and jitter.
8. Test connection storms and database failover.
9. Monitor pool wait time and active DB sessions.
```

That demonstrates architecture thinking rather than configuration memorization.

---

# 58. Senior-level scenario: failover

### Question

PostgreSQL fails over successfully, but the application remains unavailable for two minutes. Why?

Possible causes:

```text
stale connections
DNS caching
pool not recycling connections
connection retries too aggressive
connection retries too slow
PgBouncer still targeting old primary
load balancer health detection delay
application connection validation issue
new primary overloaded by reconnect storm
```

The investigation should follow:

```text
Application
    |
    v
Application pool
    |
    v
PgBouncer
    |
    v
HA endpoint / routing
    |
    v
New PostgreSQL primary
```

Do not investigate PostgreSQL alone.

---

# 59. Senior-level scenario: pool exhaustion

Suppose:

```text
Pool size = 50
Active = 50
Pending requests = 300
PostgreSQL CPU = 40%
```

Do you immediately increase pool size?

**No.**

First determine why connections are occupied.

Check:

```text
query latency
lock waits
idle-in-transaction sessions
network latency
application connection leaks
long transactions
slow downstream work inside transactions
pool instrumentation
```

For example, application code may do:

```text
BEGIN

SELECT ...

call external API   ← 5 seconds

UPDATE ...

COMMIT
```

The DB connection remains occupied while the application waits for an unrelated API.

Better:

```text
call external API

BEGIN
SELECT ...
UPDATE ...
COMMIT
```

Keep database connection hold time as short as practical.

---

# 60. Senior-level mental model

Think of connections as a controlled resource:

```text
                  Incoming Requests
                         |
                         v
                 Application Queue
                         |
                         v
                 Application Pool
                         |
                         v
                   PgBouncer
                         |
                         v
              PostgreSQL Connections
                         |
                         v
             CPU / Memory / I/O / Locks
```

Every layer controls concurrency.

The objective is:

```text
High application concurrency
        |
        v
controlled DB concurrency
        |
        v
stable PostgreSQL latency
```

not:

```text
High application concurrency
        |
        v
equally high DB concurrency
```

---

# 61. Production design rules to remember

1. **A PostgreSQL connection maps to a backend process.**
2. **Connections consume real database and OS resources.**
3. **`max_connections` is a capacity boundary, not a performance target.**
4. **Keep operational/reserved connection headroom.**
5. **Size pools using actual DB concurrency, not pod/thread count.**
6. **Always calculate pool size against maximum autoscaling capacity.**
7. **PgBouncer can separate client concurrency from PostgreSQL backend concurrency.**
8. **Transaction pooling gives stronger multiplexing but requires application compatibility testing.**
9. **`idle in transaction` sessions deserve immediate attention.**
10. **Use sensible connection, statement, lock, pool, and idle-transaction timeouts.**
11. **Use exponential backoff and jitter during reconnects.**
12. **Test connection storms—not just normal load.**
13. **Design the connection layer for PostgreSQL failover.**
14. **Do not make PgBouncer a single point of failure.**
15. **Monitor waiting connections, not just connection counts.**
16. **Include workers, BI, ETL, migrations, and admin tools in the connection budget.**
17. **Account for rolling deployments and temporary replica overlap.**
18. **More connections do not necessarily produce more throughput.**
19. **Controlled queueing is often healthier than unlimited DB concurrency.**
20. **Determine the database's sustainable concurrency through realistic load testing.**

The architecture-level takeaway is:

```text
              Application concurrency
                       |
                       v
              [ Application Pool ]
                       |
                       v
                  [ PgBouncer ]
                       |
             concurrency boundary
                       |
                       v
                 [ PostgreSQL ]
                       |
              finite DB resources
          CPU + RAM + I/O + Locks
```

**Good PostgreSQL connection management is not about allowing as many connections as possible. 
It is about allowing exactly enough database concurrency to achieve the required throughput while keeping latency, memory, CPU, failover behavior, and operational headroom predictable.**
