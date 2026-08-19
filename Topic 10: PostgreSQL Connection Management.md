## Topic 10: PostgreSQL Connection Management

PostgreSQL connection management is about controlling **how applications connect to the database, how many connections are allowed, how connections are reused, and how idle or unhealthy connections are cleaned up**.

Poor connection management can cause high memory usage, CPU overhead, slow queries, connection errors, and in severe cases database outages.

### 1. PostgreSQL Connection Limits

PostgreSQL uses a **process-per-connection model**. Each client connection normally gets its own PostgreSQL backend process.

The main server parameter is:

```sql
SHOW max_connections;
```

Example:

```text
max_connections = 200
```

This means PostgreSQL can support approximately 200 concurrent database connections, although some slots are reserved for administrative purposes.

It can be configured in `postgresql.conf`:

```conf
max_connections = 200
```

Increasing `max_connections` is not always the correct solution because every active PostgreSQL connection consumes memory and operating-system resources.

For example:

```text
Application 1 → 100 connections
Application 2 → 100 connections
Reporting    → 50 connections
Monitoring   → 20 connections
--------------------------------
Requested    → 270 connections
PostgreSQL   → max_connections = 200
```

Eventually applications can receive errors such as:

```text
FATAL: remaining connection slots are reserved
for non-replication superuser connections
```

The better architecture is usually:

```text
Applications
      |
      v
Connection Pool
      |
      v
PostgreSQL
```

rather than allowing every application instance to create a large number of direct database connections.

---

## 2. Why Too Many PostgreSQL Connections Are a Problem

A common misconception is:

```text
More connections = better performance
```

That is not necessarily true.

Suppose:

```text
50 application servers
20 DB connections each
```

The total becomes:

```text
50 × 20 = 1,000 PostgreSQL connections
```

Even if most connections are idle, PostgreSQL still needs to maintain backend processes and associated memory structures.

Too many connections can lead to:

```text
Higher memory consumption
More context switching
CPU overhead
Lock contention
Cache pressure
Slower query processing
Connection exhaustion
```

Therefore connection count should be based on **actual database concurrency**, not simply the number of application users.

---

# 3. Connection Pooling

A connection pool keeps a smaller number of database connections open and allows application requests to reuse them.

Without pooling:

```text
Request
   |
Create DB connection
   |
Authenticate
   |
Execute SQL
   |
Close DB connection
```

With pooling:

```text
Request
   |
Borrow connection
   |
Execute SQL
   |
Return connection
```

The second approach avoids repeated connection setup and authentication overhead.

Typical architecture:

```text
              +----------------+
App Server 1--|                |
App Server 2--|   PgBouncer    |---- PostgreSQL
App Server 3--|                |
              +----------------+
```

For example:

```text
Application connections → 1,000
PgBouncer connections   → 100
PostgreSQL connections  → 100
```

PgBouncer multiplexes many client connections across a smaller number of PostgreSQL server connections.

---

# 4. PgBouncer

**PgBouncer** is a lightweight external connection pooler commonly placed between applications and PostgreSQL.

Architecture:

```text
Application
     |
     | 500 client connections
     v
+-------------+
|  PgBouncer  |
+-------------+
     |
     | 50 PostgreSQL connections
     v
+-------------+
| PostgreSQL  |
+-------------+
```

Applications connect to PgBouncer instead of connecting directly to PostgreSQL.

Example application connection:

```text
postgresql://user:password@pgbouncer-host:6432/mydb
```

PgBouncer commonly listens on:

```text
6432
```

while PostgreSQL commonly listens on:

```text
5432
```

PgBouncer helps with:

```text
Connection reuse
Connection spikes
Large application fleets
Serverless workloads
Microservices
Short-lived application connections
Protecting PostgreSQL from connection exhaustion
```

---

# 5. PgBouncer Pooling Modes

PgBouncer provides three important pooling modes:

```text
Session pooling
Transaction pooling
Statement pooling
```

Understanding these modes is very important in PostgreSQL architecture.

---

## 5.1 Session Pooling

In session pooling, one PostgreSQL server connection is assigned to a client for the entire client session.

```text
Client connects
      |
      v
PgBouncer
      |
      v
PostgreSQL connection
      |
Client session continues
      |
Client disconnects
      |
Connection returned to pool
```

Example:

```text
Client A → PostgreSQL connection 1
Client B → PostgreSQL connection 2
Client C → PostgreSQL connection 3
```

The PostgreSQL connection is released only when the client disconnects.

### Advantages

Supports nearly all PostgreSQL session-level features.

For example:

```sql
SET search_path = myschema;

CREATE TEMP TABLE temp_orders (...);

LISTEN order_events;
```

### Disadvantage

Pooling efficiency is lower because long-running client sessions hold PostgreSQL connections.

Use session pooling when applications depend heavily on **session state**.

---

# 5.2 Transaction Pooling

Transaction pooling is usually the most commonly used PgBouncer mode.

A PostgreSQL connection is assigned only for the duration of a transaction.

```text
Client
  |
BEGIN
  |
PgBouncer assigns connection
  |
Queries
  |
COMMIT
  |
Connection returned to pool
```

The next transaction from the same application could use another PostgreSQL backend.

For example:

```text
Transaction 1

Client A
   |
   v
PostgreSQL Connection 5
```

Later:

```text
Transaction 2

Client A
   |
   v
PostgreSQL Connection 12
```

This provides much better connection reuse.

### Important limitation

Applications should not assume that session-level state remains available between transactions.

For example, designs depending on:

```sql
SET some_setting = 'value';
```

temporary session state, session-scoped advisory locks, or certain session-specific behaviors can require additional consideration.

Transaction pooling is normally a strong choice for:

```text
Web applications
REST APIs
Microservices
High-concurrency workloads
Short transactions
```

---

# 5.3 Statement Pooling

Statement pooling releases the PostgreSQL connection after every SQL statement.

Conceptually:

```text
SELECT ...
   |
connection returned

UPDATE ...
   |
connection returned
```

This provides very aggressive multiplexing.

However, multi-statement transactions are not supported in the normal way because:

```sql
BEGIN;

UPDATE accounts ...;

INSERT INTO transactions ...;

COMMIT;
```

requires the same backend connection across the transaction.

Therefore statement pooling is relatively uncommon.

---

# 6. Pooling Mode Comparison

| Feature              | Session              | Transaction           | Statement             |
| -------------------- | -------------------- | --------------------- | --------------------- |
| Connection released  | Client disconnect    | Transaction completes | Statement completes   |
| Pool efficiency      | Low                  | High                  | Very high             |
| Session state        | Supported            | Limited               | Very limited          |
| Transactions         | Supported            | Supported             | Restricted            |
| Typical applications | Legacy/session-heavy | APIs/web apps         | Specialized workloads |
| Recommended commonly | Sometimes            | Yes                   | Rarely                |

For most modern stateless applications, **transaction pooling is often the preferred starting point**.

---

# 7. Connection Sizing

Connection sizing should not be based only on:

```text
number of users
```

or:

```text
number of application servers
```

It should be based primarily on:

```text
Concurrent database work
CPU capacity
Memory
Query complexity
Transaction duration
Workload type
```

Consider:

```text
PostgreSQL server
CPU = 16 cores
max_connections = 200
```

You have:

```text
20 application pods
```

Giving every pod:

```text
50 connections
```

would produce:

```text
20 × 50 = 1,000 connections
```

which is clearly incompatible with the server limit.

A better design could be:

```text
20 application pods
       |
       | Up to 1,000 client connections
       v
    PgBouncer
       |
       | 80 server connections
       v
   PostgreSQL
```

The exact value requires workload testing rather than relying on a fixed universal formula.

---

# 8. Connection Budget

A good architecture creates a **connection budget**.

Suppose:

```text
max_connections = 300
```

Do not allocate all 300 connections to applications.

Example:

| Consumer          | Connections |
| ----------------- | ----------: |
| Main application  |         180 |
| Reporting         |          30 |
| ETL               |          20 |
| Monitoring        |          10 |
| DBA/admin reserve |          20 |
| Buffer            |          40 |
| Total             |         300 |

This prevents one workload from consuming every available connection.

---

# 9. PgBouncer Pool Size

Important PgBouncer settings include concepts such as:

```text
default_pool_size
min_pool_size
reserve_pool_size
max_client_conn
```

For example:

```ini
[pgbouncer]

pool_mode = transaction

max_client_conn = 2000

default_pool_size = 50
```

Conceptually this means PgBouncer may accept many application-side connections while maintaining a much smaller number of server-side PostgreSQL connections.

Remember:

```text
PgBouncer client connection count
              !=
PostgreSQL connection count
```

That separation is one of the main reasons PgBouncer is useful.

---

# 10. Application-Side Connection Pools

Applications usually have their own connection pools as well.

Examples include:

```text
Java      → HikariCP
Python    → psycopg pool / SQLAlchemy
Node.js   → pg Pool
.NET      → Npgsql pooling
Go        → database/sql
```

Example configuration:

```text
minimumPoolSize = 5
maximumPoolSize = 20
```

With Kubernetes:

```text
20 pods × maximumPoolSize 20
```

could potentially create:

```text
400 database connections
```

Therefore always calculate:

```text
Total possible connections
=
Application instances
×
Maximum pool size
```

For multiple applications:

```text
Total =
(App A instances × App A pool)
+
(App B instances × App B pool)
+
(App C instances × App C pool)
```

This calculation is extremely important during capacity planning.

---

# 11. Double Pooling

An environment can contain both:

```text
Application connection pool
          +
PgBouncer
```

For example:

```text
Application
   |
HikariCP
   |
PgBouncer
   |
PostgreSQL
```

This is common and can work well, but the application pool should not be unnecessarily large.

Example:

```text
50 Kubernetes pods
20 HikariCP connections each
```

means PgBouncer may receive:

```text
1,000 client connections
```

while PgBouncer may only keep:

```text
100 PostgreSQL connections
```

The application pool controls application-side concurrency, while PgBouncer protects PostgreSQL-side connection capacity.

---

# 12. Connection Timeouts

Several different timeouts exist, and they solve different problems.

They should not be treated as the same setting.

Important categories include:

```text
Connection timeout
Statement timeout
Lock timeout
Idle transaction timeout
Idle session timeout
Pool acquisition timeout
TCP/network timeout
```

---

# 13. Connection Timeout

A connection timeout controls how long an application waits while establishing a database connection.

Conceptually:

```text
Application
     |
Attempt connection
     |
PostgreSQL unavailable
     |
Wait...
     |
Timeout
```

Example application setting:

```text
connectionTimeout = 5 seconds
```

Without a reasonable timeout, an application can remain stuck waiting for an unhealthy database or network path.

---

# 14. statement_timeout

PostgreSQL provides:

```sql
SHOW statement_timeout;
```

It limits how long a SQL statement can run.

Example:

```sql
SET statement_timeout = '30s';
```

If a query exceeds the limit, PostgreSQL cancels it.

For example:

```sql
SELECT very_expensive_query();
```

after the configured timeout can result in an error such as:

```text
ERROR: canceling statement due to statement timeout
```

This protects the system from unexpectedly long-running queries.

However, timeout values should match workload requirements. A reporting query and a user-facing API may need very different limits.

---

# 15. lock_timeout

`lock_timeout` controls how long PostgreSQL waits to acquire a lock.

Example:

```sql
SET lock_timeout = '5s';
```

Consider:

```text
Transaction A
UPDATE customer ...
      |
holds row lock
      |
Transaction B
UPDATE same customer ...
      |
waiting
```

With `lock_timeout`, Transaction B can fail instead of waiting indefinitely.

This is especially useful for:

```text
Deployments
DDL operations
Interactive applications
Workloads sensitive to blocking
```

---

# 16. idle_in_transaction_session_timeout

This setting is extremely useful operationally.

Example:

```sql
BEGIN;

UPDATE orders ...;

-- application does nothing
```

The transaction remains open.

This can cause problems because an idle transaction may continue holding:

```text
Locks
Old transaction snapshots
Database resources
```

PostgreSQL can terminate these sessions using:

```conf
idle_in_transaction_session_timeout = '5min'
```

Conceptually:

```text
BEGIN
 |
query
 |
application becomes idle
 |
5 minutes
 |
PostgreSQL terminates session
```

---

# 17. idle_session_timeout

An idle session is different from an idle transaction.

```text
Idle session:

Connection open
No active transaction
No queries
```

PostgreSQL can terminate sessions that stay unused for too long with:

```conf
idle_session_timeout = '30min'
```

This can help clean up abandoned direct client connections, although it should be used carefully when connection pooling is involved.

---

# 18. Pool Acquisition Timeout

Consider an application connection pool containing:

```text
20 connections
```

All 20 are busy.

Request 21 arrives:

```text
Request
   |
Connection pool
   |
No free connection
   |
Wait
```

A pool acquisition timeout determines how long Request 21 waits.

For example:

```text
5 seconds
```

If no connection becomes available:

```text
Connection acquisition timeout
```

This prevents application threads from waiting forever.

---

# 19. Connection Lifecycle

A typical database connection lifecycle looks like:

```text
Application starts
      |
Create connection/pool
      |
Connect
      |
Authenticate
      |
Connection becomes available
      |
Borrow connection
      |
Execute transaction
      |
Return connection
      |
Reuse
      |
Connection becomes old/unhealthy
      |
Close connection
```

With PgBouncer:

```text
Application connection
        |
        v
PgBouncer client session
        |
        v
Acquire PostgreSQL connection
        |
Execute transaction
        |
Release PostgreSQL connection
        |
Reuse by another client
```

Understanding the lifecycle helps diagnose connection leaks and pool exhaustion.

---

# 20. Connection Leaks

A connection leak occurs when an application borrows a database connection but does not return it properly.

Example:

```text
Pool size = 20

Request 1 → takes connection
Request 2 → takes connection
...
Request 20 → takes connection
```

If connections are never returned:

```text
Request 21
   |
   v
No connection available
```

The database may be perfectly healthy, but the application appears unable to access it.

Application code should therefore follow patterns such as:

```text
Acquire
   |
try
   |
Execute
   |
finally
   |
Release
```

Many modern frameworks handle this automatically when their recommended connection-management patterns are followed.

---

# 21. Long Transactions and Pool Exhaustion

One of the biggest enemies of connection pooling is long-running transactions.

For example:

```text
Pool size = 20

20 requests
each transaction = 60 seconds
```

The pool remains fully occupied for long periods.

New requests wait even though PostgreSQL might not have reached `max_connections`.

Therefore monitor:

```text
Transaction duration
Query duration
Pool wait time
Pool utilization
```

not just PostgreSQL connection count.

---

# 22. Monitoring PostgreSQL Connections

The main PostgreSQL view is:

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
    xact_start,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity;
```

This helps identify:

```text
Active connections
Idle connections
Idle-in-transaction sessions
Long-running queries
Long-running transactions
Blocked sessions
Application sources
```

---

# 23. Connection Count by State

Useful query:

```sql
SELECT
    state,
    count(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;
```

Example result:

```text
state                 count
--------------------  -----
idle                    85
active                  18
idle in transaction      3
```

A large number of `idle` connections may simply represent connection pools.

A large number of:

```text
idle in transaction
```

is much more concerning.

---

# 24. Connections by Application

```sql
SELECT
    application_name,
    count(*)
FROM pg_stat_activity
GROUP BY application_name
ORDER BY count(*) DESC;
```

Example:

```text
order-service       80
payment-service     40
reporting-service   20
monitoring           5
```

This is very useful when determining which application is consuming connection capacity.

---

# 25. Connections by Database

```sql
SELECT
    datname,
    count(*)
FROM pg_stat_activity
GROUP BY datname
ORDER BY count(*) DESC;
```

Example:

```text
orders_db       120
reporting_db     40
postgres          5
```

---

# 26. Detecting Idle Transactions

```sql
SELECT
    pid,
    usename,
    application_name,
    xact_start,
    state,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

These sessions deserve attention because they may retain locks or prevent normal cleanup of older row versions.

---

# 27. Detecting Long-Running Transactions

```sql
SELECT
    pid,
    usename,
    application_name,
    now() - xact_start AS transaction_duration,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Long transactions can cause:

```text
Locking problems
Pool exhaustion
MVCC cleanup problems
Table bloat
Replication impact
```

depending on the workload.

---

# 28. Terminating Problem Connections

PostgreSQL provides:

```sql
SELECT pg_cancel_backend(pid);
```

to request cancellation of the currently running query.

For stronger intervention:

```sql
SELECT pg_terminate_backend(pid);
```

terminates the session.

A common operational approach is:

```text
1. Investigate
2. Identify PID
3. Understand transaction impact
4. Try pg_cancel_backend()
5. Use pg_terminate_backend() if necessary
```

Do not blindly terminate production sessions without understanding what transaction they are running.

---

# 29. Connection Sizing Example

Consider this architecture:

```text
Kubernetes

Order Service
20 pods
10 connections each

Payment Service
10 pods
10 connections each

Customer Service
10 pods
10 connections each
```

Potential application connections:

```text
Orders
20 × 10 = 200

Payments
10 × 10 = 100

Customers
10 × 10 = 100

Total = 400
```

Suppose:

```text
PostgreSQL max_connections = 200
```

Direct connectivity is risky.

Better:

```text
                 400 app connections
                         |
                         v
                    PgBouncer
                         |
                    80 connections
                         |
                         v
                    PostgreSQL
```

Then create a connection budget such as:

```text
PgBouncer application pool = 80
Reporting                   = 20
ETL                         = 10
Monitoring                  = 5
Admin                       = 5
Reserve                     = 30
--------------------------------
Total                       = 150
```

while PostgreSQL might have:

```text
max_connections = 200
```

leaving additional safety margin.

---

# 30. Recommended Enterprise Architecture

A common architecture is:

```text
                     Kubernetes
                         |
          +--------------+--------------+
          |              |              |
      Service A      Service B      Service C
          |              |              |
          +--------------+--------------+
                         |
                         v
                  +-------------+
                  |  PgBouncer  |
                  +-------------+
                         |
                Controlled number
                 of DB connections
                         |
                         v
                  +-------------+
                  | PostgreSQL  |
                  +-------------+
```

For high availability:

```text
Applications
      |
      v
Load Balancer / DB Endpoint
      |
      v
PgBouncer instances
      |
      v
PostgreSQL Primary
```

The exact placement of PgBouncer depends on the PostgreSQL platform, failover design, service discovery, and application architecture.

---

# 31. Connection Management Best Practices

1. **Do not blindly increase `max_connections`.** First understand why connections are being consumed.

2. **Use connection pooling** for applications with many clients or application instances.

3. **Size application pools globally**, not one pod at a time.

   Always calculate:

   ```text
   replicas × pool size
   ```

4. **Keep transactions short.**

5. **Avoid idle-in-transaction connections.**

6. **Configure sensible timeouts** for connections, queries, locks, and transactions.

7. **Keep administrative connection capacity available.**

8. **Monitor `pg_stat_activity`.**

9. **Separate reporting and OLTP connection budgets** where appropriate.

10. **Load test connection settings** before production.

---

# 32. Important Interview Scenario

### Question

Your PostgreSQL server has:

```text
max_connections = 300
```

You run:

```text
50 Kubernetes pods
```

and each pod has:

```text
maximumPoolSize = 20
```

What is the problem?

### Answer

Potential application connections are:

```text
50 × 20 = 1,000
```

but PostgreSQL supports only around:

```text
300
```

connections.

The application fleet could exhaust PostgreSQL's connection capacity.

Possible architecture:

```text
50 pods
   |
1,000 possible application connections
   |
PgBouncer
   |
100 controlled PostgreSQL connections
   |
PostgreSQL
```

Also consider reducing individual application pool sizes based on actual workload concurrency.

---

# 33. Another Important Scenario

### Problem

Users complain that the API is timing out.

PostgreSQL CPU is only:

```text
30%
```

and connections show:

```text
120 / 300
```

Does this mean PostgreSQL has enough capacity?

Not necessarily.

Suppose the application pool is:

```text
20 / 20 connections busy
```

Then:

```text
Application request
       |
       v
Application pool
       |
       | No free connection
       v
Waits
```

The bottleneck is the **application connection pool**, not PostgreSQL's global `max_connections`.

Therefore connection troubleshooting should look at the full chain:

```text
Application thread
      ↓
Application connection pool
      ↓
PgBouncer
      ↓
PostgreSQL connection
      ↓
Locks / query execution
```

---

# 34. Architecture Troubleshooting Flow

When an application reports database connection problems, check:

```text
Application
   |
   | Is application pool exhausted?
   v
Application Pool
   |
   | Is PgBouncer queueing clients?
   v
PgBouncer
   |
   | Is server pool exhausted?
   v
PostgreSQL
   |
   | max_connections reached?
   | long-running transaction?
   | locks?
   | slow queries?
   v
Database workload
```

Do not immediately assume:

```text
Database is slow
```

The problem could be at any layer.

---

# 35. Key Metrics to Monitor

For production PostgreSQL connection management, monitor at least:

| Area        | Metrics                  |
| ----------- | ------------------------ |
| PostgreSQL  | Current connections      |
| PostgreSQL  | Active connections       |
| PostgreSQL  | Idle connections         |
| PostgreSQL  | Idle-in-transaction      |
| PostgreSQL  | Connection utilization % |
| PostgreSQL  | Transaction duration     |
| PostgreSQL  | Query duration           |
| PgBouncer   | Client connections       |
| PgBouncer   | Server connections       |
| PgBouncer   | Waiting clients          |
| PgBouncer   | Pool utilization         |
| Application | Active pool connections  |
| Application | Idle pool connections    |
| Application | Connection wait time     |
| Application | Pool acquisition timeout |

The most useful mental model is:

```text
Connection count
        +
Connection utilization
        +
Connection wait time
        +
Transaction duration
```

A connection count alone rarely explains the complete problem.

---

## Final Architecture Summary

```text
                 Application Requests
                         |
                         v
              Application Connection Pool
                         |
                         v
                    PgBouncer
              Transaction Pooling
                         |
                         v
                    PostgreSQL
                         |
        +----------------+----------------+
        |                |                |
  max_connections   Timeouts       pg_stat_activity
        |                |                |
        +----------------+----------------+
                         |
                         v
               Stable DB Connectivity
```

### Key takeaway

PostgreSQL connection management is **not about maximizing the number of connections**. It is about controlling concurrency.

The target architecture is:

```text
Many application requests
         ↓
Controlled application pools
         ↓
PgBouncer when required
         ↓
Limited PostgreSQL connections
         ↓
Short transactions
         ↓
Proper timeouts
         ↓
Continuous monitoring
```

For an architecture/interview answer, remember this line:

> **Size PostgreSQL connections based on database concurrency and capacity, not on the number of application users. Use pooling to multiplex application demand onto a controlled number of database connections.**
