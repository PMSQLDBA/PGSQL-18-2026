# Topic 28 — PostgreSQL Capacity & Scalability

Verified this against the current official PostgreSQL documentation. 

As of August 2026, the current major release documented by PostgreSQL is **PostgreSQL 18**. The notes below use PostgreSQL 18 behavior and terminology. ([PostgreSQL][1])

Official references: 
· [PostgreSQL 18 — High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html) 
· [Table Partitioning](https://www.postgresql.org/docs/18/ddl-partitioning.html) 
· [Connections and Authentication](https://www.postgresql.org/docs/current/runtime-config-connection.html) 
· [Resource Consumption](https://www.postgresql.org/docs/current/runtime-config-resource.html)

---

## 1. What does PostgreSQL scalability mean?

Scalability is the ability of a PostgreSQL platform to handle increasing demand without unacceptable degradation in:

* throughput / transactions per second
* query latency
* concurrent connections
* data volume
* read/write volume
* CPU, memory, storage and I/O utilization
* availability

Think of database growth across four dimensions:

```text
                PostgreSQL Growth

                    Workload
                       |
        +--------------+--------------+
        |              |              |
      Data          Traffic       Connections
      Growth         Growth          Growth
        |              |              |
        +--------------+--------------+
                       |
                 Need to Scale
                       |
           +-----------+-----------+
           |                       |
     Vertical Scaling        Horizontal Scaling
     Bigger DB server        More DB servers
```

A good PostgreSQL architecture normally does **not** jump immediately to horizontal scaling. The usual progression is:

```text
Optimize SQL / indexes
        ↓
Tune PostgreSQL
        ↓
Scale server resources
        ↓
Introduce connection pooling
        ↓
Partition very large tables where useful
        ↓
Add read replicas
        ↓
Distribute/shard only if necessary
```

That sequence matters because distributed databases introduce considerably more operational and application complexity.

---

# 2. Vertical Scaling

Vertical scaling means increasing the resources available to one PostgreSQL server.

For example:

```text
Before

PostgreSQL
8 CPU
32 GB RAM
500 GB SSD


After vertical scaling

PostgreSQL
32 CPU
128 GB RAM
2 TB NVMe
```

No additional PostgreSQL primary is introduced.

## What can be vertically scaled?

### CPU

More CPU cores can help workloads containing:

* many concurrent queries
* CPU-heavy queries
* aggregations
* joins
* sorting
* parallel queries
* index creation
* maintenance activity

But:

> More CPUs do not automatically make every individual query faster.

A poorly indexed query performing a huge sequential scan can remain slow even after adding CPUs.

---

### Memory

Increasing RAM can be extremely valuable because PostgreSQL and the operating system can keep more frequently accessed data in memory.

Important PostgreSQL memory-related settings include:

```text
shared_buffers
work_mem
maintenance_work_mem
effective_cache_size
huge_pages
```

`shared_buffers` controls PostgreSQL's shared buffer cache. PostgreSQL 18 documentation suggests **25% of system RAM as a reasonable starting point** for a dedicated database server with at least 1 GB of RAM, while also noting that PostgreSQL relies heavily on the OS filesystem cache. The documentation says allocations above roughly 40% are unlikely to perform better than smaller values in most cases. This is guidance, not a universal sizing formula. ([PostgreSQL][2])

Example:

```text
Server RAM = 128 GB

Possible starting point:

shared_buffers = 32GB
```

Do not blindly use:

```text
shared_buffers = 100GB
```

just because memory exists.

---

### Storage

Storage performance is often one of the most important capacity factors.

Important characteristics include:

```text
IOPS
latency
throughput
storage capacity
fsync performance
WAL write performance
```

For write-heavy PostgreSQL workloads, storage latency can become a major bottleneck.

Typical improvement:

```text
HDD
 ↓
SSD
 ↓
NVMe SSD
 ↓
Higher-performance provisioned storage
```

Depending on the environment, storage and WAL may also be placed on storage designed for their different access patterns.

---

## Advantages of vertical scaling

Vertical scaling is operationally simple:

```text
Application
     |
     v
 PostgreSQL
     |
 Bigger Machine
```

Benefits include:

* minimal application changes
* no distributed data ownership
* no replica routing logic
* no sharding logic
* transactions remain straightforward
* easier backup/recovery architecture
* easier troubleshooting

Therefore, vertical scaling is often the **first infrastructure scaling option**.

---

## Vertical scaling limitations

Eventually, a single machine reaches limits.

Examples:

```text
CPU limit
RAM limit
storage capacity limit
storage throughput limit
network limit
connection/process overhead
single-primary write throughput
hardware/cloud instance maximum
```

There is also an economic limit.

Going from:

```text
16 → 32 CPUs
```

may be reasonable.

Going from:

```text
128 → 256 → 512 CPUs
```

may become expensive while producing progressively smaller improvements for the workload.

So vertical scaling has a practical ceiling.

---

# 3. Horizontal Scaling

Horizontal scaling means adding database servers rather than continuously increasing one server.

Example:

```text
                  Application
                       |
                 Load / Routing
                  /    |     \
                 /     |      \
                v      v       v
             Primary Replica Replica
```

PostgreSQL's built-in capabilities provide several pieces that can support horizontal architectures, but an important distinction is:

> Native PostgreSQL replication does not magically turn multiple PostgreSQL servers into one unlimited multi-write database.

The official documentation specifically points out that read-only database servers are relatively easy to combine, whereas read/write servers are considerably harder because writes must be synchronized between systems. ([PostgreSQL][3])

Horizontal scaling therefore has different forms.

### Read scaling

```text
1 Primary
+
N Read Replicas
```

This is common.

### Data distribution

```text
Shard 1
Shard 2
Shard 3
...
```

This requires considerably more architecture.

### Logical data distribution

Logical replication can selectively replicate tables/data between PostgreSQL systems.

### Foreign data access

`postgres_fdw` allows one PostgreSQL server to access tables on another PostgreSQL server. It can be useful in distributed architectures, although FDW itself should not be confused with a complete transparent sharding solution. ([PostgreSQL][4])

---

# 4. Read Replicas

Read replicas are one of PostgreSQL's most important horizontal scaling techniques.

Architecture:

```text
                       Writes
Application --------------------------+
                                     |
                                     v
                                 PRIMARY
                                    |
                           WAL / Replication
                         /          |          \
                        v           v           v
                    Replica 1   Replica 2   Replica 3
                        ^           ^           ^
                        |           |           |
                    Reporting    Search      Analytics
```

PostgreSQL physical streaming replication transfers WAL changes from the primary to standby servers.

A standby configured for **Hot Standby** can accept connections and execute read-only queries while recovery is occurring. ([PostgreSQL][5])

---

## Why replicas improve scalability

Suppose the primary receives:

```text
2,000 write TPS
20,000 read QPS
```

If most reads can move to replicas:

```text
Primary
  Writes
  critical reads

Replica 1
  application reads

Replica 2
  reporting

Replica 3
  analytics
```

the primary has significantly less query work.

---

# 5. Read/Write Splitting

Applications can route operations differently.

```text
INSERT ─┐
UPDATE ─┼────────> Primary
DELETE ─┘

SELECT ──────────> Read Replica
```

But the application must understand consistency requirements.

For example:

```sql
UPDATE account
SET balance = 500
WHERE account_id = 100;
```

Immediately afterward:

```sql
SELECT balance
FROM account
WHERE account_id = 100;
```

If the `SELECT` goes to an asynchronous replica, it might temporarily return the old balance.

Why?

```text
Primary
  |
  | Commit
  |
  +------ WAL ------> Replica
                        |
                      Replay
```

There can be a delay between primary commit and replica replay.

This is **replication lag**.

---

# 6. Synchronous vs Asynchronous Replication

PostgreSQL supports different replication durability approaches.

## Asynchronous

Conceptually:

```text
Client
  |
  v
Primary
  |
Commit
  |
Return SUCCESS
  |
  +--------> Replica
```

Advantages:

* lower commit latency
* primary doesn't normally wait for standby acknowledgement

Trade-off:

* replica can lag
* recent transactions may be unavailable on a replica
* depending on failure timing/configuration, failover can lose transactions that had not reached the standby

The PostgreSQL documentation explicitly describes this availability/performance trade-off. ([PostgreSQL][3])

---

## Synchronous

Conceptually:

```text
Client
   |
Primary
   |
Generate WAL
   |
   +-------> Synchronous Standby
                   |
               acknowledge
                   |
                   v
Primary completes commit
```

This improves durability guarantees but increases write latency because commit can depend on communication with a synchronous standby.

The correct architecture depends on:

```text
RPO requirement
latency requirement
network latency
failure model
business criticality
```

---

# 7. Replicas Do Not Solve Write Scaling

This is an important architecture/interview point.

Consider:

```text
Primary
   |
   +------ Replica
   |
   +------ Replica
   |
   +------ Replica
```

You now have more **read capacity**.

But:

```text
INSERT
UPDATE
DELETE
```

still normally go to the primary in a standard physical replication architecture.

Therefore:

```text
Read scalability  → improved

Write scalability → largely unchanged
```

In fact, replication introduces additional WAL/network/replication work on the primary.

---

# 8. Cascading Replication

With many replicas, having every standby connect directly to the primary is not always desirable.

Instead:

```text
                  Primary
                 /       \
                /         \
          Replica A      Replica B
            /   \
           /     \
    Replica C   Replica D
```

C and D receive replication through A.

PostgreSQL supports **cascading replication** as part of its standby architecture. ([PostgreSQL][3])

This can reduce some direct replication fan-out from the primary.

Trade-off:

```text
Primary
  ↓
Replica A
  ↓
Replica C
```

adds another replication stage, so downstream lag and failure topology need consideration.

---

# 9. Replication Slots and Capacity

Replication slots help PostgreSQL retain resources needed by replication consumers.

Relevant PostgreSQL settings include:

```text
max_wal_senders
max_replication_slots
wal_keep_size
```

PostgreSQL 18 currently documents defaults of **10** for both `max_wal_senders` and `max_replication_slots`. These are configuration defaults, not recommended production replica counts. ([PostgreSQL][6])

A critical operational risk is:

```text
Replica / consumer stops
        ↓
Replication slot remains active
        ↓
Required WAL retained
        ↓
pg_wal grows
        ↓
Disk consumption increases
```

Therefore capacity monitoring should include replication slots and WAL retention, not only table/database size.

---

# 10. Partitioning

Partitioning splits one logical table into multiple physical tables called partitions.

Example:

```text
orders
 |
 +-- orders_2026_01
 +-- orders_2026_02
 +-- orders_2026_03
 +-- orders_2026_04
```

Applications can still query the partitioned parent:

```sql
SELECT *
FROM orders
WHERE order_date >= DATE '2026-08-01';
```

PostgreSQL routes/plans access against the appropriate partitions.

---

# 11. PostgreSQL Partitioning Types

PostgreSQL supports built-in declarative:

```text
RANGE
LIST
HASH
```

partitioning. ([PostgreSQL][7])

---

## RANGE partitioning

Good for time-series or naturally ordered ranges.

```sql
CREATE TABLE orders (
    order_id bigint,
    order_date date NOT NULL,
    customer_id bigint,
    amount numeric
)
PARTITION BY RANGE (order_date);
```

Partitions:

```sql
CREATE TABLE orders_2026_07
PARTITION OF orders
FOR VALUES FROM ('2026-07-01')
TO ('2026-08-01');

CREATE TABLE orders_2026_08
PARTITION OF orders
FOR VALUES FROM ('2026-08-01')
TO ('2026-09-01');
```

Architecture:

```text
orders
 |
 +-- July
 +-- August
 +-- September
```

Common uses:

* audit records
* transactions
* events
* telemetry
* logs
* historical data

---

# 12. LIST Partitioning

Useful for discrete known values.

Example:

```text
customers
 |
 +-- US
 +-- Canada
 +-- Europe
```

SQL:

```sql
CREATE TABLE customers (
    customer_id bigint,
    region text
)
PARTITION BY LIST (region);
```

Then:

```sql
CREATE TABLE customers_us
PARTITION OF customers
FOR VALUES IN ('US');
```

Useful when data has clear logical categories.

---

# 13. HASH Partitioning

Useful when you want rows distributed across partitions using a hash of the partition key.

```text
customer_id
     |
    hash
     |
+----+----+----+----+
P0   P1   P2   P3
```

Example:

```sql
CREATE TABLE customer_events (
    event_id bigint,
    customer_id bigint,
    payload jsonb
)
PARTITION BY HASH (customer_id);
```

This can distribute rows relatively evenly when range/list semantics are not appropriate.

---

# 14. Partition Pruning

Partition pruning is one of the main performance reasons to partition.

Suppose:

```text
orders
 |
 +-- 2024
 +-- 2025
 +-- 2026
```

Query:

```sql
SELECT *
FROM orders
WHERE order_date >= DATE '2026-08-01';
```

Without useful pruning, many partitions could need consideration/scanning.

With pruning:

```text
2024  → prune
2025  → prune
2026  → scan relevant partition(s)
```

PostgreSQL uses partition bounds to determine which partitions can be excluded. Partition pruning can happen during planning and, in some situations, during execution. ([PostgreSQL][7])

Verify with:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE order_date >= DATE '2026-08-01';
```

---

# 15. Partitioning Is Not the Same as Sharding

This distinction is essential.

## Partitioning

Usually:

```text
              PostgreSQL Server

                   orders
                     |
        +------------+------------+
        |            |            |
      Part 1       Part 2       Part 3
```

All partitions may remain inside the same PostgreSQL cluster/server infrastructure.

## Sharding

```text
Application / Routing
       |
 +-----+-----+-----+
 |           |     |
Shard 1   Shard 2 Shard 3
Server A   Server B Server C
```

Data is distributed across independent database servers.

Therefore:

> Partitioning primarily divides a logical table. Sharding distributes data across independent database nodes.

Partitioning alone does **not** eliminate the capacity ceiling of a single PostgreSQL server.

---

# 16. Why Partition Large Tables?

PostgreSQL's official documentation lists several potential benefits:

* faster access in suitable workloads
* improved data locality
* smaller working indexes
* efficient bulk data removal
* easier movement of old data
* avoiding vacuum overhead from huge bulk deletes

For example, instead of:

```sql
DELETE FROM events
WHERE event_date < DATE '2025-01-01';
```

potentially deleting hundreds of millions of rows, an old partition can sometimes simply be detached or dropped.

Conceptually:

```sql
ALTER TABLE events
DETACH PARTITION events_2024;
```

or:

```sql
DROP TABLE events_2024;
```

This can be dramatically more efficient than deleting rows individually. ([PostgreSQL][7])

---

# 17. Too Many Partitions Can Hurt

A common misconception is:

```text
More partitions = more performance
```

That is false.

Too many partitions can increase:

* planning time
* metadata overhead
* session memory usage
* maintenance complexity
* DDL complexity

PostgreSQL 18 says the planner can generally handle hierarchies containing **up to a few thousand partitions reasonably well when typical queries prune all but a small number**, but planning time and memory can grow when many partitions remain after pruning. PostgreSQL explicitly recommends testing the intended workload rather than assuming more partitions are better. ([PostgreSQL][7])

So:

```text
100 partitions
```

may outperform:

```text
10,000 partitions
```

depending entirely on workload and pruning.

---

# 18. Connection Scaling

Connections are another major PostgreSQL capacity constraint.

PostgreSQL uses a process-based architecture for client sessions.

Conceptually:

```text
Client 1 ──> PostgreSQL backend
Client 2 ──> PostgreSQL backend
Client 3 ──> PostgreSQL backend
...
Client N ──> PostgreSQL backend
```

PostgreSQL controls the maximum using:

```text
max_connections
```

PostgreSQL 18 documentation says its typical default is **100**, although the actual initialized value can depend on kernel support. Increasing `max_connections` also increases allocation of certain resources, including shared memory. ([PostgreSQL][8])

---

# 19. Why Not Simply Set max_connections = 10,000?

Because accepting more connections is not the same as processing more useful database work.

Imagine:

```text
10,000 application connections
          |
          v
      PostgreSQL
```

Thousands of backends can create:

* memory pressure
* process scheduling overhead
* context switching
* lock contention
* CPU contention
* cache pressure
* slower overall throughput

The better architecture is often:

```text
10,000 application-side requests/connections
              |
              v
       Connection Pooler
              |
         200 DB connections
              |
              v
          PostgreSQL
```

The exact numbers are workload-dependent; `200` is only an illustration.

---

# 20. Connection Pooling

A pooler lets many application clients share a smaller number of database connections.

Architecture:

```text
Application Instances
 | | | | | | | | | |
          |
          v
   Connection Pooler
     | | | | | |
          |
          v
      PostgreSQL
```

Instead of repeatedly creating expensive database sessions:

```text
connect
authenticate
execute
disconnect
```

connections can be reused.

This is especially valuable for:

* microservices
* serverless applications
* high-concurrency APIs
* applications with many mostly-idle sessions
* connection bursts

PostgreSQL itself provides the server-side connection limit and session model; production architectures commonly place an external pooler or platform-provided pooling layer in front when connection fan-in becomes large.

---

# 21. Connection Capacity Formula — Important Caution

A simplistic estimate sometimes used is:

```text
memory ≈ connections × work_mem
```

Do **not** use this as an exact PostgreSQL memory formula.

`work_mem` is not allocated once per connection. It can be used by individual sort/hash operations, and a single query can contain multiple such operations; parallel workers can further change memory demand.

For example:

```text
100 active queries
× several memory-consuming plan nodes
× work_mem
```

can consume far more memory than expected.

Therefore capacity planning should model **concurrent memory-consuming operations**, not simply connection count.

---

# 22. Workload Growth

Database growth normally occurs across several dimensions simultaneously.

```text
Users
  ↓
Requests
  ↓
Connections
  ↓
Queries
  ↓
CPU / Memory
  ↓
Reads / Writes
  ↓
WAL
  ↓
Storage
  ↓
Replication
  ↓
Backup volume
```

Capacity planning therefore cannot be based only on:

```text
Database size = 2 TB
```

Two 2-TB databases can have completely different capacity requirements.

---

# 23. Important Capacity Metrics

I would monitor at least these categories.

### Compute

```text
CPU utilization
load
CPU saturation
parallel worker usage
```

### Memory

```text
RAM utilization
swap activity
shared buffer behavior
OS cache pressure
temporary operations
```

### Storage

```text
database size
table/index growth
disk utilization
IOPS
read latency
write latency
throughput
temporary file growth
WAL volume
```

### Connections

```text
active connections
idle connections
idle-in-transaction connections
connection utilization
connection wait time
pool utilization
```

### SQL workload

```text
TPS
QPS
query latency
slow queries
lock waits
deadlocks
temporary files
sequential scans
index usage
```

### Replication

```text
replication lag
WAL generation
WAL retention
replication slots
standby replay
standby conflicts
```

### Maintenance

```text
dead tuples
autovacuum activity
VACUUM duration
ANALYZE
table/index bloat indicators
checkpoint behavior
```

---

# 24. Capacity Is About Saturation, Not Just Utilization

This is an architecture concept worth remembering.

Suppose:

```text
CPU = 60%
```

That does not necessarily mean:

```text
40% capacity available
```

The workload could already be bottlenecked on storage latency or lock contention.

Example:

```text
CPU           60%
Memory        70%
Disk capacity 50%

Disk latency  VERY HIGH
```

The database is effectively saturated on I/O even though CPU and storage capacity look healthy.

Therefore:

```text
Capacity != one metric
```

Instead:

```text
Effective capacity =
minimum remaining capacity
across critical bottleneck resources
```

---

# 25. Scaling Read-Heavy Workloads

Suppose an application has:

```text
90% SELECT
10% writes
```

A sensible architecture may be:

```text
                 Application
                     |
          +----------+----------+
          |                     |
        Writes                 Reads
          |                     |
          v              +------+------+
       Primary           |      |      |
                         v      v      v
                        R1     R2     R3
```

Potential benefits:

* reporting isolated from primary
* analytics isolated
* more aggregate read throughput
* HA replicas may also provide read capacity

But replication lag must be understood.

---

# 26. Scaling Write-Heavy Workloads

Suppose:

```text
20% reads
80% writes
```

Adding ten read replicas will not solve the main bottleneck.

The scaling sequence should focus first on things such as:

```text
SQL optimization
        ↓
index optimization
        ↓
transaction design
        ↓
batching where appropriate
        ↓
WAL/storage performance
        ↓
checkpoint/autovacuum tuning
        ↓
vertical scaling
        ↓
partitioning where beneficial
        ↓
data distribution/sharding if required
```

Write scaling is fundamentally harder than read scaling.

---

# 27. PostgreSQL Write Scaling Limitation

Standard PostgreSQL physical replication is fundamentally:

```text
            Primary
              |
        all primary writes
              |
      +-------+-------+
      |               |
   Standby         Standby
```

It is not:

```text
Write
 ↓
Any PostgreSQL node
 ↓
Automatically globally consistent
```

This is one of the most important scaling limitations to understand.

The PostgreSQL documentation itself explains that read/write systems are harder to combine because modifications must be propagated while maintaining consistency. ([PostgreSQL][3])

---

# 28. Logical Replication for Scaling

Logical replication operates at a different level from physical streaming replication.

It can be useful for:

* selective table replication
* data distribution
* migrations
* upgrades
* integration architectures
* consolidating selected data

Simplified architecture:

```text
Publisher
 |
 | logical changes
 |
 +----------> Subscriber
```

PostgreSQL 18 requires `wal_level = logical` for a logical-replication publisher and capacity planning needs to account for replication slots, WAL senders and logical replication workers. ([PostgreSQL][9])

Logical replication introduces its own operational considerations and is not simply a drop-in replacement for physical replicas.

---

# 29. Partitioning + Replicas

These techniques solve different problems and can be combined.

```text
                     Primary
                        |
        +---------------+---------------+
        |               |               |
     Orders Q1       Orders Q2       Orders Q3
        |
        | streaming replication
        v
                     Replica
                        |
        +---------------+---------------+
        |               |               |
     Orders Q1       Orders Q2       Orders Q3
```

Partitioning helps manage:

```text
large-table access
data lifecycle
maintenance
index/data locality
```

Replicas help with:

```text
read scaling
reporting isolation
HA
```

They are complementary, not competing solutions.

---

# 30. Partitioning Does Not Automatically Improve Every Query

Suppose:

```text
orders PARTITION BY order_date
```

Query:

```sql
SELECT *
FROM orders
WHERE customer_id = 12345;
```

If there is no useful `order_date` condition, PostgreSQL may need to consider many partitions.

Compare:

```sql
SELECT *
FROM orders
WHERE customer_id = 12345
  AND order_date >= DATE '2026-08-01'
  AND order_date < DATE '2026-09-01';
```

Now the partition key condition can allow PostgreSQL to prune unrelated date partitions.

Therefore the partition key should strongly align with:

```text
query access patterns
+
data lifecycle
+
uniqueness requirements
```

PostgreSQL specifically recommends choosing partition keys that commonly appear in query `WHERE` clauses when appropriate. ([PostgreSQL][7])

---

# 31. Scaling Limitations — Summary

PostgreSQL has several practical scaling boundaries.

| Area         | Limitation                                                 |
| ------------ | ---------------------------------------------------------- |
| CPU          | Single server has finite compute                           |
| Memory       | Server RAM is finite                                       |
| Storage      | Capacity/IOPS/latency limits                               |
| Connections  | Excessive backends create overhead                         |
| Writes       | Standard physical replication retains a primary write node |
| Replicas     | Mainly increase read capacity                              |
| Replication  | Lag and WAL retention must be managed                      |
| Partitioning | Too many partitions increase planning/memory overhead      |
| Sharding     | Introduces distributed-system complexity                   |
| Network      | Replication/distributed queries depend on network          |
| Locks        | Contention can limit scaling before CPU does               |
| Transactions | Cross-node transactions are harder                         |
| Operations   | More nodes increase monitoring/failover/backup complexity  |

---

# 32. Scaling Decision Framework

For architecture work, use this flow:

```text
Database workload increasing?
           |
           v
Are queries optimized?
     |           |
    No          Yes
     |           |
Optimize         v
SQL/indexes   Resource bottleneck?
                 |
       +---------+---------+
       |         |         |
      CPU      Memory      I/O
       |         |         |
       +---------+---------+
                 |
          Vertical scaling
                 |
                 v
        Connections high?
           |          |
          Yes         No
           |
      Connection Pool
                 |
                 v
        Read workload high?
           |          |
          Yes         No
           |
       Read Replicas
                 |
                 v
       Very large tables?
           |          |
          Yes         No
           |
       Partitioning
                 |
                 v
     Write capacity exhausted?
           |          |
          Yes         No
           |
 Data distribution /
 sharding evaluation
```

---

# 33. Example Production Evolution

### Stage 1 — Small

```text
Application
    |
PostgreSQL
```

Use:

* correct indexes
* query tuning
* backups
* monitoring
* autovacuum

---

### Stage 2 — Growing application

```text
Applications
     |
Connection Pool
     |
PostgreSQL
```

Focus on:

```text
connection management
memory
CPU
storage
query performance
```

---

### Stage 3 — Read-heavy growth

```text
                    +--> Read Replica
Application -> Pool -> Primary
                    +--> Read Replica
```

Use replicas for suitable read traffic.

---

### Stage 4 — Large data

```text
Primary
 |
 +-- partition 2025
 +-- partition 2026_01
 +-- partition 2026_02
 +-- partition 2026_03
```

Introduce partitioning when table size/access/lifecycle actually justify it.

---

### Stage 5 — Large distributed workload

```text
                    Router/Application
                         |
          +--------------+--------------+
          |              |              |
       Shard A         Shard B        Shard C
          |              |              |
       Replica         Replica        Replica
```

At this stage you are dealing with distributed-system concerns:

```text
shard-key selection
routing
rebalancing
cross-shard queries
distributed transactions
failure handling
backup/recovery
operational automation
```

This should normally be justified by actual capacity requirements rather than introduced prematurely.

---

# 34. Capacity Planning Example

Suppose today's workload is:

```text
Database size       = 2 TB
Growth               = 80 GB/month
Peak connections     = 350
Peak TPS             = 8,000
Reads                = 80%
Writes               = 20%
CPU peak             = 70%
Storage peak         = 65%
Replication lag      = normally low
```

Expected growth:

```text
Traffic growth = 50% over 12 months
```

Do not simply calculate:

```text
8,000 × 1.5 = 12,000 TPS
```

Also project:

```text
connections
CPU
RAM
storage
IOPS
WAL generation
replica load
backup size
backup duration
restore duration
autovacuum workload
table/index growth
```

Architecture response might be:

```text
1. Tune expensive SQL
2. Introduce/validate connection pooling
3. Increase primary CPU/RAM/storage capacity
4. Move eligible reads to replicas
5. Partition the largest time-series tables
6. Benchmark expected 12–24 month workload
7. Maintain operational headroom
```

---

# 35. Benchmark Before Scaling

PostgreSQL includes `pgbench`, which can run benchmark workloads using multiple concurrent sessions and report transaction rates and latency. Important options include client count, transaction count/time and custom scripts. ([PostgreSQL][10])

For example:

```bash
pgbench -c 100 -j 8 -T 300 mydb
```

But production capacity testing should preferably use transactions representative of the real application.

Better:

```text
Production SQL patterns
+
production-like data volume
+
production-like concurrency
+
realistic read/write ratio
+
realistic indexes
+
realistic connection pooling
```

Then test:

```text
100 users
500 users
1,000 users
2,000 users
```

and observe where latency begins increasing disproportionately.

---

# 36. Capacity Headroom

Do not design production so that normal peak usage is:

```text
CPU      95%
Storage  95%
Connections 98%
```

That leaves little room for:

```text
traffic spikes
failover
maintenance
VACUUM
index creation
backups
replication catch-up
unexpected queries
```

The required headroom is organization/workload-specific, so there is no universal PostgreSQL percentage.

Architecturally, the objective is:

```text
Normal Peak Load
      <
Safe Operating Capacity
      <
Maximum Capacity
```

You especially need to consider **failure capacity**.

For example:

```text
Normal:

Replica 1 = 40%
Replica 2 = 40%
Replica 3 = 40%
```

If one replica fails, the remaining replicas must absorb its traffic.

Capacity planning therefore needs:

```text
normal-state capacity
+
degraded-state capacity
```

---

# 37. Architecture Example

A mature PostgreSQL architecture could look like:

```text
                   Applications
                        |
                        v
                 Load Balancer/API
                        |
                        v
                Connection Pooling
                        |
              +---------+---------+
              |                   |
              | Writes            | Reads
              v                   v
          PostgreSQL          Read Routing
            Primary           /          \
              |              /            \
              |             v              v
              |         Replica 1      Replica 2
              |
              | WAL
              +-----------------------------+
              |
        Partitioned Tables
              |
       +------+------+------+
       |      |      |      |
      P1     P2     P3     P4
```

Each layer solves a different capacity problem:

```text
Pooling
   → connection scalability

Vertical scaling
   → CPU/RAM/I/O capacity

Partitioning
   → large-table management and suitable query performance

Read replicas
   → read scalability + HA possibilities

Sharding/data distribution
   → scale beyond one server's data/write boundaries
```

---

# 38. Key Architecture Rules to Remember

1. **Scale SQL before scaling hardware.** A missing index can waste far more capacity than additional CPUs can recover.

2. **Vertical scaling is usually simpler than horizontal scaling.** Use the simpler architecture while it meets requirements.

3. **Read replicas scale reads, not primary writes.**

4. **Connection count is not throughput.** 5,000 connections do not mean 5,000 useful concurrent transactions.

5. **Use pooling for connection fan-in.**

6. **Partition for a reason.** Large tables, pruning, lifecycle management and maintenance are good reasons; "the table has many rows" alone isn't enough.

7. **Partitioning is not sharding.**

8. **Replication lag matters for read-after-write consistency.**

9. **More replicas create more operational work.**

10. **Capacity planning must include degraded operation.** Ask what happens when a replica/node is unavailable.

11. **Benchmark the actual workload.** PostgreSQL's `pgbench` is useful, but custom production-like workloads provide much better capacity evidence. ([PostgreSQL][10])

12. **Do not scale based on CPU alone.** PostgreSQL can be constrained by I/O, locks, memory, WAL, connections, storage or maintenance long before CPU reaches 100%.

---

# 39. Interview / Architecture Answer

If asked:

**"How would you scale PostgreSQL?"**

A strong answer is:

> I first identify the actual bottleneck rather than immediately adding servers. I check query performance, indexes, CPU, memory, I/O, connection pressure, locking, WAL generation and table growth. I optimize SQL and database configuration first, then vertically scale the primary where appropriate. For high connection counts, I introduce connection pooling. For read-heavy workloads, I use hot standby/read replicas and route suitable read traffic to them while accounting for replication lag. For very large tables, I consider declarative partitioning when the partition key aligns with query and data-retention patterns. If the workload eventually exceeds the write or storage capacity of a single primary, I evaluate logical distribution or sharding, understanding that this introduces routing, cross-shard transaction, rebalancing and operational complexity.

That demonstrates the important principle:

```text
Optimize
   ↓
Pool
   ↓
Scale Up
   ↓
Scale Reads Out
   ↓
Partition
   ↓
Distribute only when required
```

## PostgreSQL 18 official documentation used

These notes were checked against the current PostgreSQL 18 documentation, particularly:

* [High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html?utm_source=chatgpt.com) — replication architecture, standby concepts and scaling trade-offs. ([PostgreSQL][3])
* [Hot Standby](https://www.postgresql.org/docs/18/hot-standby.html?utm_source=chatgpt.com) — running read-only workloads on standby servers. ([PostgreSQL][5])
* [Replication Configuration](https://www.postgresql.org/docs/18/runtime-config-replication.html?utm_source=chatgpt.com) — `max_wal_senders`, replication slots, WAL retention and synchronous replication settings. ([PostgreSQL][6])
* [Table Partitioning](https://www.postgresql.org/docs/18/ddl-partitioning.html?utm_source=chatgpt.com) — RANGE/LIST/HASH partitioning, pruning, maintenance and partition-count considerations. ([PostgreSQL][7])
* [Connections and Authentication](https://www.postgresql.org/docs/current/runtime-config-connection.html?utm_source=chatgpt.com) — `max_connections` and reserved connections. ([PostgreSQL][8])
* [Resource Consumption](https://www.postgresql.org/docs/current/runtime-config-resource.html?utm_source=chatgpt.com) — memory/resource configuration including `shared_buffers`. ([PostgreSQL][2])
* [Logical Replication Configuration](https://www.postgresql.org/docs/18/logical-replication-config.html?utm_source=chatgpt.com) — publisher/subscriber capacity settings. ([PostgreSQL][9])
* [postgres_fdw](https://www.postgresql.org/docs/18/postgres-fdw.html?utm_source=chatgpt.com) — accessing PostgreSQL data on remote servers. ([PostgreSQL][4])
* [pgbench](https://www.postgresql.org/docs/current/pgbench.html?utm_source=chatgpt.com) — workload and concurrency benchmarking. ([PostgreSQL][10])

**Topic 28 takeaway:** PostgreSQL scalability is not one feature. It is a progression of **query optimization → resource tuning → connection management → vertical scaling → partitioning/read scaling → distributed data architecture**, chosen according to the actual bottleneck.

[1]: https://www.postgresql.org/docs/current/runtime-config-wal.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.5. Write Ahead Log"
[2]: https://www.postgresql.org/docs/current/runtime-config-resource.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.4. Resource Consumption"
[3]: https://www.postgresql.org/docs/current/high-availability.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 26. High Availability, Load Balancing, and Replication"
[4]: https://www.postgresql.org/docs/18/postgres-fdw.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: F.38. postgres_fdw — access data stored in external PostgreSQL servers"
[5]: https://www.postgresql.org/docs/18/hot-standby.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 26.4. Hot Standby"
[6]: https://www.postgresql.org/docs/18/runtime-config-replication.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.6. Replication"
[7]: https://www.postgresql.org/docs/18/ddl-partitioning.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 5.12. Table Partitioning"
[8]: https://www.postgresql.org/docs/current/runtime-config-connection.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.3. Connections and Authentication"
[9]: https://www.postgresql.org/docs/18/logical-replication-config.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 29.12. Configuration Settings"
[10]: https://www.postgresql.org/docs/current/pgbench.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: pgbench"
