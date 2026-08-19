# Topic 7 — PostgreSQL HA-DR Tools & Ecosystem

PostgreSQL itself provides the core pieces for HA and DR: streaming replication, synchronous replication, standby promotion, WAL archiving, recovery, logical replication, and monitoring views. A complete production platform usually adds external tools around PostgreSQL for **failover orchestration, consensus, connection routing, pooling, backup, restore, and monitoring**. PostgreSQL 18’s HA chapter is the right base reference for the database-side behavior. ([PostgreSQL][1])

## 1. High-Level Tool Architecture

A common enterprise stack looks like this:

```text
                         Applications
                              │
                              ▼
                        PgBouncer
                  Connection Pooling Layer
                              │
                              ▼
                         HAProxy / VIP
                     Database Routing Layer
                              │
                  ┌───────────┴───────────┐
                  ▼                       ▼
               Primary                 Standby
             PostgreSQL 18           PostgreSQL 18
                  │                       │
                  └──────── WAL ──────────┘
                              │
                              ▼
                            Patroni
                      HA Orchestration
                              │
                              ▼
                    etcd / Consul / DCS
                   Consensus / Cluster State
                              │
                 ┌────────────┴────────────┐
                 ▼                         ▼
            pgBackRest                  Barman
         Backup / Restore           Backup / DR Mgmt
```

The important point is that each tool solves a different problem. You should avoid treating the stack as one product.

---

## 2. Responsibilities by Layer

| Tool / Layer        | Main Responsibility                            |
| ------------------- | ---------------------------------------------- |
| PostgreSQL 18       | Replication, WAL, standby, recovery, promotion |
| Patroni             | PostgreSQL HA orchestration                    |
| etcd / Consul       | Distributed cluster state / consensus          |
| HAProxy             | Route traffic to correct database node         |
| PgBouncer           | PostgreSQL connection pooling                  |
| pgBackRest          | Physical backup, restore, PITR, WAL archive    |
| Barman              | Centralized PostgreSQL backup/DR management    |
| Prometheus/Grafana  | Monitoring and dashboards                      |
| OS / Infrastructure | Storage, network, compute, fencing             |

The separation matters because, for example, **PgBouncer is not a failover manager**, and **Patroni is not a backup solution**.

---

# 3. Patroni

Patroni is a PostgreSQL HA orchestration framework. It manages PostgreSQL nodes and coordinates which node is allowed to act as leader/primary. Patroni uses a distributed configuration store (DCS) such as etcd, Consul, ZooKeeper, or Kubernetes-backed state, depending on the deployment model. ([Patroni][2])

A simple architecture:

```text
                 Applications
                      │
                      ▼
                 HAProxy / VIP
                      │
            ┌─────────┴─────────┐
            ▼                   ▼
        PostgreSQL A        PostgreSQL B
          Patroni             Patroni
            │                   │
            └─────────┬─────────┘
                      ▼
                    etcd
```

Patroni's main responsibilities typically include:

```text
Leader election
PostgreSQL role management
Promotion
Demotion
Replication configuration
Health checking
Cluster membership
Failover coordination
Switchover operations
REST API for node state
```

Patroni does **not replace PostgreSQL replication**. It orchestrates PostgreSQL's native capabilities.

---

# 4. Patroni + PostgreSQL Relationship

Think of it this way:

```text
PostgreSQL
    │
    ├── Generates WAL
    ├── Streams WAL
    ├── Runs standby
    ├── Promotes standby
    └── Handles database transactions

Patroni
    │
    ├── Decides which node should be leader
    ├── Coordinates failover
    ├── Updates PostgreSQL configuration
    ├── Manages promotion/demotion
    └── Advertises node health/state
```

So:

```text
PostgreSQL = Database engine

Patroni = HA control layer
```

---

# 5. Distributed Configuration Store — DCS

Patroni needs shared cluster state.

This is normally provided by a DCS such as:

```text
etcd
Consul
ZooKeeper
Kubernetes API
```

The DCS stores information such as:

```text
Current leader
Cluster membership
Leader lock
Configuration
Failover state
```

Conceptually:

```text
Patroni Node A ─┐
Patroni Node B ─┼──► DCS
Patroni Node C ─┘
                    │
                    ▼
             Current Leader State
```

Patroni's documentation recommends running a fault-tolerant DCS with an odd number of members, commonly 3 or 5, for proper consensus. ([Patroni][2])

---

# 6. Why 3 DCS Nodes Are Better Than 2

Suppose you have:

```text
etcd1
etcd2
```

If communication between them fails:

```text
etcd1     X     etcd2
```

there is no clear majority.

A 3-member setup:

```text
etcd1
etcd2
etcd3
```

requires:

```text
2 of 3
```

for a majority.

If one node fails:

```text
etcd1
etcd2
etcd3 X
```

the remaining two can continue making consensus decisions.

This is why production consensus layers are generally designed around odd-sized clusters.

---

# 7. Patroni Leader Lock

A simplified Patroni model is:

```text
DCS
 │
 └── Leader Lock
         │
         ▼
     PostgreSQL A
```

As long as A owns and successfully renews the leader state, A can remain primary.

If A fails or can no longer maintain valid leadership:

```text
Leader lock expires / changes
          │
          ▼
Other Patroni nodes evaluate state
          │
          ▼
New leader selected
```

This mechanism is part of how Patroni avoids arbitrary multiple-primary behavior.

---

# 8. Patroni Failover Flow

A simplified failover:

```text
Primary PostgreSQL fails
          │
          ▼
Patroni detects failure
          │
          ▼
DCS leadership evaluated
          │
          ▼
Best standby selected
          │
          ▼
Standby promoted
          │
          ▼
DCS leader updated
          │
          ▼
HAProxy routes traffic
          │
          ▼
Applications reconnect
```

This is much more complete than manually running:

```bash
pg_ctl promote
```

because orchestration also has to manage cluster state and routing.

---

# 9. Patroni Switchover vs Failover

### Switchover

Planned:

```text
Primary healthy
      │
      ▼
Operator requests role change
      │
      ▼
Patroni promotes chosen replica
```

Useful for:

```text
OS patching
PostgreSQL maintenance
Hardware changes
Planned testing
```

### Failover

Unplanned:

```text
Primary unavailable
      │
      ▼
Patroni elects/promotes replacement
```

These should be tested separately.

---

# 10. `patronictl`

Patroni provides:

```text
patronictl
```

for administrative operations.

Typical operational areas include:

```text
View cluster state
Switchover
Failover
Restart PostgreSQL
Reload configuration
Edit cluster configuration
Reinitialize replicas
```

Conceptually:

```text
DBA
 │
 ▼
patronictl
 │
 ▼
Patroni Cluster
```

This becomes an important DBA tool in a Patroni-managed environment.

---

# 11. Patroni REST API

Patroni exposes node health and role information through its REST API.

This is commonly useful for:

```text
Health checks
HAProxy routing
Monitoring
Automation
```

Example logic:

```text
HAProxy
   │
   ▼
Patroni REST API
   │
   ├── Is node primary?
   ├── Is node replica?
   └── Is node healthy?
```

Then HAProxy can route based on the result.

---

# 12. HAProxy

HAProxy is commonly placed between applications and PostgreSQL nodes to provide a stable database endpoint and route connections based on node health/role.

Conceptually:

```text
              Applications
                   │
                   ▼
                HAProxy
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
     Primary               Replica
    Read/Write             Read-only
```

In a Patroni architecture, HAProxy commonly checks Patroni's REST API to determine which node currently owns the primary role.

HAProxy itself does not understand PostgreSQL WAL or replication state unless external health checks expose that information.

---

# 13. Writer Endpoint

A common HAProxy design provides:

```text
postgres-write.company.local
```

which always routes to:

```text
Current Primary
```

Architecture:

```text
Application
    │
    ▼
postgres-write
    │
    ▼
HAProxy
    │
    ▼
Current Primary
```

After failover:

```text
Old Primary X
New Primary B
```

the application still connects to:

```text
postgres-write
```

and HAProxy changes the backend destination.

---

# 14. Reader Endpoint

You can also create:

```text
postgres-read.company.local
```

routing to healthy replicas.

Example:

```text
              Reader Endpoint
                    │
                    ▼
                 HAProxy
            ┌───────┼───────┐
            ▼       ▼       ▼
         Replica1 Replica2 Replica3
```

This supports:

```text
Reporting
Analytics
Read scaling
```

But you must account for replication lag.

A read immediately after a write may not yet be visible on an asynchronous replica.

---

# 15. HAProxy Is Not Connection Pooling

This distinction is important:

```text
HAProxy
   │
   └── Routes network/database connections

PgBouncer
   │
   └── Pools PostgreSQL connections
```

They solve different problems.

You may use:

```text
Application
    │
    ▼
PgBouncer
    │
    ▼
HAProxy
    │
    ▼
PostgreSQL
```

or sometimes:

```text
Application
    │
    ▼
HAProxy
    │
    ▼
PgBouncer
    │
    ▼
PostgreSQL
```

The correct order depends on architecture and ownership of pooling/routing.

---

# 16. PgBouncer

PgBouncer is a lightweight connection pooler for PostgreSQL. Applications connect to PgBouncer as though it were PostgreSQL, while PgBouncer manages a smaller pool of actual server connections. ([PgBouncer][3])

Architecture:

```text
Applications
     │
     │ 2,000 client connections
     ▼
  PgBouncer
     │
     │ 200 PostgreSQL connections
     ▼
 PostgreSQL
```

This can significantly reduce PostgreSQL backend-process pressure.

---

# 17. Why PgBouncer Matters in HA

After failover, many applications may reconnect simultaneously.

Without pooling:

```text
Primary fails
    │
    ▼
Thousands of clients reconnect
    │
    ▼
New primary overloaded
```

This is called a:

```text
connection storm
```

PgBouncer can help control the number of actual PostgreSQL connections.

```text
5,000 application sessions
          │
          ▼
      PgBouncer
          │
          ▼
200 database connections
```

This can make failover behavior much more predictable.

---

# 18. PgBouncer Pooling Modes

PgBouncer supports different pooling modes:

```text
session
transaction
statement
```

The official PgBouncer feature documentation notes that transaction pooling can break some PostgreSQL session-based behavior, so application compatibility must be checked. ([PgBouncer][4])

### Session pooling

```text
Client connects
     │
     ▼
Gets server connection
     │
     ▼
Keeps it for client session
```

Most PostgreSQL-session semantics are preserved.

---

# 19. Transaction Pooling

With transaction pooling:

```text
Client
  │
  ▼
BEGIN
  │
  ▼
Gets server connection
  │
  ▼
COMMIT
  │
  ▼
Connection returned to pool
```

This provides much higher pooling efficiency.

But session-level features need careful review.

For example, application behavior involving session state can become problematic if that state is expected to remain tied to one PostgreSQL backend.

PgBouncer explicitly documents compatibility differences between pooling modes. ([PgBouncer][4])

---

# 20. Statement Pooling

Statement pooling is even more aggressive:

```text
Each SQL statement
      │
      ▼
May use pooled connection
```

This has significant transaction restrictions and is not suitable for many normal transactional applications.

For most enterprise applications, the meaningful design comparison is usually:

```text
Session Pooling
       vs
Transaction Pooling
```

---

# 21. PgBouncer Capacity Planning

Important concepts include:

```text
max_client_conn
default_pool_size
min_pool_size
reserve_pool_size
pool_mode
```

Conceptually:

```text
Maximum client sessions
       │
       ▼
PgBouncer
       │
       ▼
Controlled server pool
       │
       ▼
PostgreSQL max_connections
```

Do not configure them independently.

You want:

```text
PgBouncer server pool
   <
PostgreSQL safe backend capacity
```

while still leaving room for:

```text
DBA connections
Monitoring
Replication
Maintenance
Administrative sessions
```

---

# 22. PgBouncer Deployment Options

### Local PgBouncer

```text
Application Host
      │
   PgBouncer
      │
      ▼
PostgreSQL
```

Benefits can include local pooling and lower connection setup overhead.

### Central PgBouncer

```text
Applications
     │
     ▼
Central PgBouncer
     │
     ▼
PostgreSQL
```

Simpler central management, but PgBouncer itself must then be made highly available.

---

# 23. HAProxy + PgBouncer Example

One practical design:

```text
Applications
     │
     ▼
PgBouncer Pair
     │
     ▼
HAProxy Pair
     │
     ▼
Patroni PostgreSQL Cluster
```

Responsibilities:

```text
PgBouncer
   → control connections

HAProxy
   → find current primary

Patroni
   → manage primary/standby roles

etcd
   → consensus

PostgreSQL
   → database + replication
```

---

# 24. pgBackRest

pgBackRest is a PostgreSQL-focused physical backup and restore solution. Its official documentation supports full, differential, and incremental backup models, restores, WAL archiving, multiple repositories, and remote/cloud repository options. ([pgBackRest][5])

Architecture:

```text
            PostgreSQL
                 │
                 ├── Data
                 │
                 └── WAL
                     │
                     ▼
                 pgBackRest
                     │
             ┌───────┴────────┐
             ▼                ▼
       Backup Repository   WAL Archive
```

---

# 25. pgBackRest Backup Types

Common backup types include:

```text
Full
Differential
Incremental
```

### Full

Copies the complete required database-cluster backup set.

### Differential

Copies changes since the latest full backup.

### Incremental

Copies changes since the previous backup in the chain.

pgBackRest's official user guide describes full backup and restore behavior and supports multiple backup strategies. ([pgBackRest][5])

---

# 26. pgBackRest and WAL Archiving

For PITR:

```text
Database Backup
      +
Archived WAL
      =
Point-in-Time Recovery
```

Architecture:

```text
Primary
   │
   ├── Backup
   │      ▼
   │   Repository
   │
   └── WAL
          ▼
      WAL Archive
```

You might restore:

```text
Backup at 01:00
      │
      +
WAL until 10:47:25
      │
      ▼
Database state at 10:47:25
```

This is one reason backup and WAL archiving belong in the overall DR ecosystem.

---

# 27. pgBackRest from Standby

pgBackRest supports backup architectures that can use a standby for backup operations. ([pgBackRest][6])

Conceptually:

```text
                Primary
                   │
                   │ WAL
                   ▼
                Standby
                   │
                   ▼
              pgBackRest
                   │
                   ▼
            Backup Repository
```

Benefit:

```text
Reduce backup read workload
on primary
```

But the complete backup architecture must still account for WAL consistency, cluster topology, and failover.

---

# 28. Multiple Backup Repositories

pgBackRest supports multiple repositories. ([pgBackRest][6])

Example:

```text
                  PostgreSQL
                      │
                  pgBackRest
                 /          \
                ▼            ▼
          Repository 1   Repository 2
          Local/DC        Remote/Cloud
```

This is useful for:

```text
Local fast restores
+
Off-site disaster protection
```

---

# 29. Barman

Barman, Backup and Recovery Manager, is an open-source PostgreSQL backup and disaster-recovery management tool designed to manage backups and recovery for multiple PostgreSQL servers, often from a centralized backup host. ([PgBarman Documentation][7])

Architecture:

```text
 PostgreSQL A ─┐
 PostgreSQL B ─┼──► Barman Server
 PostgreSQL C ─┘        │
                        ▼
                 Backup Repository
```

This can be useful in estates where one team centrally manages many PostgreSQL clusters.

---

# 30. Barman Capabilities

Barman documentation covers areas including:

```text
Physical backups
WAL archiving
WAL streaming
Point-in-time recovery
Backup catalog management
Recovery
Multiple PostgreSQL servers
```

([PgBarman Documentation][7])

A useful mental model is:

```text
PostgreSQL Fleet
      │
      ▼
Central Barman
      │
      ▼
Backup/Recovery Operations
```

---

# 31. pgBackRest vs Barman

Both are serious PostgreSQL backup/DR tools, so you generally do **not** need both for the same backup responsibility unless there is a clear architectural reason.

| Area                            | pgBackRest                               | Barman                                           |
| ------------------------------- | ---------------------------------------- | ------------------------------------------------ |
| PostgreSQL physical backup      | Yes                                      | Yes                                              |
| WAL archive                     | Yes                                      | Yes                                              |
| PITR                            | Yes                                      | Yes                                              |
| Central multi-server model      | Possible                                 | Strong focus                                     |
| Full/diff/incremental           | Strong                                   | Depends on selected method/features              |
| Cloud/object repository support | Strong                                   | Available options depend on architecture/version |
| Standby backup                  | Supported                                | Supported workflows available                    |
| Typical use                     | High-performance PostgreSQL backup stack | Centralized backup/DR management                 |

Choose based on:

```text
Operational model
Existing standards
Database sizes
Restore requirements
Repository design
Cloud/on-prem architecture
Team skills
Automation
```

Do not choose just because one tool has more features.

---

# 32. Backup Tool Is Not HA

Important distinction:

```text
Patroni
   → Failover / HA

pgBackRest / Barman
   → Backup / Recovery / DR
```

If primary crashes:

```text
Patroni
   │
   ▼
Promote standby
```

If someone accidentally deletes production data:

```text
pgBackRest / Barman
        │
        ▼
Restore / PITR
```

These solve different failure scenarios.

---

# 33. HA Tool Is Not DR by Itself

A Patroni cluster may look like:

```text
Same Rack
Primary
Standby
Standby
```

It could survive:

```text
single server failure
```

but not necessarily:

```text
rack failure
data-center loss
regional outage
```

DR requires geographic and backup/recovery design beyond simple cluster failover.

---

# 34. HA + Backup Reference Architecture

A stronger design:

```text
                       Applications
                            │
                            ▼
                         PgBouncer
                            │
                            ▼
                         HAProxy
                            │
                            ▼
               ┌────────────────────────┐
               │    Patroni Cluster     │
               │                        │
               │ Primary  →  Standby    │
               └────────────┬───────────┘
                            │
                     etcd / DCS
                            │
                     PostgreSQL WAL
                            │
               ┌────────────┴────────────┐
               ▼                         ▼
          pgBackRest                 Remote DR
             │                         Standby
             ▼
        Backup Repository
```

This provides separate mechanisms for:

```text
Connection management
Routing
HA
Consensus
Backup
DR
```

---

# 35. Multi-Site Architecture

A more advanced design:

```text
                     SITE A
             ┌──────────────────┐
             │ Primary          │
             │ Local Standby    │
             │ Patroni          │
             └────────┬─────────┘
                      │
                      │ WAL
                      ▼
                     SITE B
             ┌──────────────────┐
             │ DR Standby       │
             │ Patroni/Standby  │
             └──────────────────┘

DCS / consensus architecture
must be designed separately.
```

Patroni's multi-data-center documentation specifically discusses synchronous/asynchronous replication and recommends odd-sized DCS deployments such as 3 or 5 nodes. ([Patroni][8])

---

# 36. DCS Placement Is Critical

Bad design:

```text
Site A
   │
   ├── PostgreSQL Primary
   ├── PostgreSQL Standby
   └── all 3 etcd nodes
```

If Site A disappears:

```text
Database nodes gone
+
DCS quorum gone
```

A multi-zone design might instead distribute DCS members:

```text
AZ1 → etcd1
AZ2 → etcd2
AZ3 → etcd3
```

Patroni's multi-DC guidance explicitly recommends spreading an odd-numbered DCS cluster across failure domains. ([Patroni][8])

---

# 37. Don't Use a Single etcd Node in Production HA

Architecture:

```text
Patroni A
Patroni B
    │
    ▼
etcd1
```

If:

```text
etcd1 fails
```

your consensus/control plane becomes unavailable.

Even though PostgreSQL itself might continue running, HA management is degraded.

Better:

```text
etcd1
etcd2
etcd3
```

with proper placement and monitoring.

---

# 38. Consul

Consul can also act as Patroni's distributed configuration store.

Conceptually:

```text
Patroni Nodes
     │
     ▼
Consul Cluster
     │
     ▼
Leader/Cluster State
```

Patroni supports Consul as one of its DCS backends. ([Patroni][2])

The exact choice between:

```text
etcd
Consul
Kubernetes
```

should usually align with your organization's existing platform architecture.

Avoid introducing a new consensus technology solely for PostgreSQL if the organization already operates a reliable supported option.

---

# 39. Kubernetes and Patroni

Patroni can operate using Kubernetes objects/API for storing cluster state and managing the leader key. ([Patroni][9])

Conceptually:

```text
Kubernetes
    │
    ├── PostgreSQL Pod A
    │      └── Patroni
    │
    ├── PostgreSQL Pod B
    │      └── Patroni
    │
    └── Kubernetes API
             │
             ▼
       Cluster state
```

For Kubernetes environments, dedicated PostgreSQL operators may also manage much of this lifecycle. We will cover that more deeply in **Topic 27 — PostgreSQL Containers & Kubernetes**.

---

# 40. Monitoring Stack

A production HA ecosystem also needs observability.

Common architecture:

```text
PostgreSQL
Patroni
PgBouncer
HAProxy
etcd
Backup Tool
OS
 │
 ▼
Metrics Collectors
 │
 ▼
Prometheus
 │
 ▼
Grafana
 │
 ▼
Alerting
```

You need visibility across the **whole stack**, not just PostgreSQL.

---

# 41. PostgreSQL Metrics

Important database metrics include:

```text
Availability
Connections
Transactions
Locks
Wait events
WAL generation
Replication lag
Autovacuum
Database growth
Query performance
Checkpoints
I/O
```

PostgreSQL 18 provides the core monitoring views under Chapter 27. ([PostgreSQL][10])

---

# 42. Patroni Metrics

Monitor things such as:

```text
Current leader
Replica state
Failover state
PostgreSQL health
Timeline
Replication state
DCS communication
```

If PostgreSQL is healthy but Patroni cannot reach the DCS, you have an **HA control-plane issue**.

---

# 43. etcd / DCS Metrics

Monitor:

```text
Member availability
Leader
Quorum
Request latency
Storage/database size
Disk latency
Network latency
Clock health
```

The DCS is small in data volume but extremely important operationally.

A slow disk on etcd can affect cluster leadership behavior.

---

# 44. PgBouncer Metrics

Monitor:

```text
Client connections
Server connections
Pool utilization
Waiting clients
Transaction rate
Query rate
Connection errors
Pool saturation
```

Example problem:

```text
PostgreSQL healthy
      │
      ▼
PgBouncer pool full
      │
      ▼
Application sees database timeout
```

So the database may not actually be the bottleneck.

---

# 45. HAProxy Metrics

Monitor:

```text
Frontend connections
Backend availability
Backend health
Current primary route
Connection failures
Session counts
Health-check failures
```

A classic failure:

```text
PostgreSQL successfully failed over
             │
             ▼
HAProxy still routes to old node
             │
             ▼
Application remains down
```

That is why routing is part of the HA service.

---

# 46. Backup Monitoring

Monitor pgBackRest/Barman for:

```text
Last successful backup
Backup age
Backup duration
Backup size
WAL archive success
Archive queue/backlog
Repository space
Retention
Restore test
```

A backup job reporting "success" is not enough.

You need:

```text
Successful restore test
```

to validate recoverability.

---

# 47. Example Enterprise HA/DR Stack

A practical on-prem or VM design:

```text
                         APPLICATION
                              │
                              ▼
                         PgBouncer
                              │
                              ▼
                    HAProxy Pair / VIP
                              │
                ┌─────────────┴─────────────┐
                ▼                           ▼
        PostgreSQL Node A           PostgreSQL Node B
            Patroni                     Patroni
              │                           │
              └────────────┬──────────────┘
                           ▼
                       etcd Cluster
                    etcd1 etcd2 etcd3
                           │
                           ▼
                    Cluster Consensus

        PostgreSQL Node A/B
                │
                ▼
            pgBackRest
                │
        ┌───────┴────────┐
        ▼                ▼
 Local Repository    Remote Repository

All components
      │
      ▼
Prometheus → Grafana → Alerts
```

---

# 48. Component Failure Examples

### PostgreSQL primary fails

```text
Patroni detects
    │
    ▼
Replica promoted
    │
    ▼
HAProxy updates routing
```

### PgBouncer fails

Database may be healthy, but clients cannot reach it.

Therefore PgBouncer itself may need:

```text
multiple instances
+
load balancer/VIP
```

### HAProxy fails

Same issue: routing layer becomes the single point of failure.

### etcd loses quorum

PostgreSQL may still be running, but Patroni cannot safely make new leadership decisions.

### Backup repository fails

HA continues, but DR/recovery protection is degraded.

---

# 49. Avoid Single Points of Failure

Do not build:

```text
2 PostgreSQL nodes
+
1 HAProxy
+
1 PgBouncer
+
1 etcd
```

and call it highly available.

You simply moved the single points of failure.

A better architecture needs redundancy at:

```text
Database
Routing
Pooling
Consensus
Monitoring
Backup repository
Network
Storage
```

depending on the required SLA.

---

# 50. Recommended Tool Responsibility Matrix

| Requirement                      | Recommended Layer                         |
| -------------------------------- | ----------------------------------------- |
| Database transactions            | PostgreSQL                                |
| WAL generation                   | PostgreSQL                                |
| Streaming replication            | PostgreSQL                                |
| Synchronous replication          | PostgreSQL                                |
| Standby promotion primitive      | PostgreSQL                                |
| Automatic failover orchestration | Patroni                                   |
| Cluster state/consensus          | etcd / Consul / Kubernetes                |
| Writer routing                   | HAProxy / service layer                   |
| Reader routing                   | HAProxy / service layer                   |
| Connection pooling               | PgBouncer                                 |
| Physical backup                  | pgBackRest or Barman                      |
| PITR                             | PostgreSQL + backup/WAL tool              |
| Off-site backup                  | Backup tool/repository design             |
| Monitoring                       | Prometheus/Grafana or enterprise platform |
| Fencing                          | Infrastructure/HA architecture            |

---

# 51. Tool Selection Guidance

For a new self-managed PostgreSQL HA environment, a common choice is:

```text
PostgreSQL 18
+
Patroni
+
etcd
+
HAProxy
+
PgBouncer
+
pgBackRest
+
Prometheus/Grafana
```

This is **a reference architecture, not a PostgreSQL requirement**.

If your company already has:

```text
Consul
enterprise load balancer
enterprise monitoring
central backup platform
```

then reuse those where technically suitable.

The goal should be:

```text
fewer well-operated components
```

rather than:

```text
maximum number of PostgreSQL tools
```

---

# 52. When Barman May Be a Better Fit

Barman can be attractive when:

```text
Many PostgreSQL clusters
Central DBA backup team
Centralized recovery catalog
Remote backup architecture
Existing Barman operational knowledge
```

Barman's current official documentation presents it as a PostgreSQL backup and disaster-recovery manager with support for managing multiple remote servers. ([PgBarman Documentation][7])

---

# 53. When pgBackRest May Be a Better Fit

pgBackRest can be attractive when you need:

```text
High-performance parallel backup/restore
Full/differential/incremental strategy
Multiple repositories
Cloud/object storage
Encryption/compression
Backup from standby
Large databases
```

Those capabilities are documented in the project's current official material. ([pgBackRest][11])

---

# 54. Important PostgreSQL 18 Logical Failover Point

PostgreSQL 18 has native support relevant to **logical replication failover**. The PostgreSQL 18 documentation explains that logical replication slots associated with subscriptions can be synchronized to a physical standby when failover support is enabled, so logical replication can continue after publisher failover. ([PostgreSQL][12])

That means modern HA design may need to consider not only:

```text
physical standby continuity
```

but also:

```text
logical subscription continuity
```

This is especially important during migrations and CDC designs.

---

# 55. HA/DR Failure-Domain Design

Think about failure domains explicitly:

```text
Process
   ↓
Server
   ↓
VM/Hypervisor
   ↓
Storage
   ↓
Rack
   ↓
Availability Zone
   ↓
Data Center
   ↓
Region
```

Your architecture should identify which failures each component protects against.

For example:

```text
Patroni + local standby
    → server failure

Multi-AZ standby
    → AZ failure

Remote asynchronous standby
    → regional/site DR

pgBackRest off-site backup
    → corruption / deletion / historical recovery
```

---

# 56. HA/DR Runbook

Your production runbook should document:

```text
How to check Patroni cluster
How to perform switchover
How to perform emergency failover
How to confirm current primary
How to check etcd quorum
How to verify HAProxy routing
How to check PgBouncer pools
How to rejoin failed node
How to use pg_rewind
How to rebuild replica
How to restore pgBackRest/Barman backup
How to perform PITR
How to activate DR site
How to fail back
```

Automation is valuable, but DBAs still need to understand the underlying process.

---

# 57. Production Testing

Test the stack component by component.

```text
Test 1 → PostgreSQL process failure
Test 2 → Primary server failure
Test 3 → Replica failure
Test 4 → Network isolation
Test 5 → etcd member failure
Test 6 → DCS quorum loss
Test 7 → HAProxy failure
Test 8 → PgBouncer failure
Test 9 → WAL archive failure
Test 10 → Backup repository failure
Test 11 → Restore
Test 12 → PITR
Test 13 → Full DR activation
```

A successful primary promotion alone does not validate the whole platform.

---

# 58. Recommended Production Architecture

For most traditional self-managed enterprise deployments, this is a strong conceptual model:

```text
                  ┌─────────────────────┐
                  │    Applications     │
                  └──────────┬──────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │     PgBouncer       │
                  │ Connection Pooling  │
                  └──────────┬──────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │      HAProxy        │
                  │  Primary Routing    │
                  └──────────┬──────────┘
                             │
                 ┌───────────┴───────────┐
                 ▼                       ▼
        ┌─────────────────┐     ┌─────────────────┐
        │ PostgreSQL 18 A │     │ PostgreSQL 18 B │
        │ PRIMARY         │ WAL │ STANDBY         │
        │ Patroni         │────►│ Patroni         │
        └────────┬────────┘     └────────┬────────┘
                 │                       │
                 └───────────┬───────────┘
                             ▼
                  ┌─────────────────────┐
                  │     etcd Cluster    │
                  │  etcd1/etcd2/etcd3 │
                  └─────────────────────┘

 PostgreSQL
      │
      ▼
 pgBackRest
      │
 ┌────┴───────────┐
 ▼                ▼
Local Backup   Off-site Backup

All components
      │
      ▼
Prometheus / Enterprise Monitoring
      │
      ▼
Grafana / Alerts / On-call
```

---

# 59. Core Rules to Remember

```text
PostgreSQL handles replication.

Patroni handles HA orchestration.

etcd/Consul handles shared cluster state.

HAProxy handles routing.

PgBouncer handles pooling.

pgBackRest/Barman handle backup and recovery.

Monitoring must cover every layer.

Replication is not backup.

Patroni is not backup.

PgBouncer is not failover.

HAProxy is not consensus.

DCS must itself be highly available.

Every added component can become a new failure point.
```

---

## Official References

For the PostgreSQL 18 foundation, use **Chapter 26 — High Availability, Load Balancing and Replication** and the PostgreSQL 18 replication settings documentation. These define the native HA and replication mechanisms that external tooling builds upon. ([PostgreSQL][1])

For Patroni, use the official **Patroni 4.1.3 documentation**, including its DCS guidance and multi-data-center HA recommendations. Patroni currently documents DCS options including etcd and Consul and recommends 3 or 5 DCS nodes for consensus/fault tolerance. ([Patroni][2])

For connection pooling, use the official **PgBouncer documentation**. PgBouncer describes itself as a lightweight PostgreSQL connection pooler and documents the compatibility trade-offs between pooling modes. ([PgBouncer][3])

For backup and recovery, use the official **pgBackRest User Guide** and **Barman User Guide**. pgBackRest documents physical backup/restore, WAL archiving, multiple repositories and advanced backup features, while Barman focuses on centralized PostgreSQL backup and DR management. ([pgBackRest][5])

**Tracker: 7 / 29 completed — 22 pending.**

**Next: Topic 8 — PostgreSQL Backup, Restore & Recovery Activities.** That topic will go deeply into `pg_dump`, `pg_dumpall`, `pg_restore`, `pg_basebackup`, physical vs logical backup, full/differential/incremental strategies, WAL archiving, PITR, backup validation, retention, restore scenarios, and PostgreSQL 18 recovery architecture.

[1]: https://www.postgresql.org/docs/current/high-availability.html?utm_source=chatgpt.com "18: Chapter 26. High Availability, Load Balancing, and ..."
[2]: https://patroni.readthedocs.io/en/latest/README.html?utm_source=chatgpt.com "Introduction — Patroni 4.1.3 documentation"
[3]: https://www.pgbouncer.org/?utm_source=chatgpt.com "PgBouncer - lightweight connection pooler for PostgreSQL"
[4]: https://www.pgbouncer.org/features.html?utm_source=chatgpt.com "Features"
[5]: https://pgbackrest.org/user-guide.html?utm_source=chatgpt.com "pgBackRest User Guide - Debian & Ubuntu"
[6]: https://pgbackrest.org/news.html?utm_source=chatgpt.com "pgBackRest - News"
[7]: https://docs.pgbarman.org/?utm_source=chatgpt.com "User Guide — Barman 3.19.1 documentation"
[8]: https://patroni.readthedocs.io/en/latest/ha_multi_dc.html?utm_source=chatgpt.com "HA multi datacenter — Patroni 4.1.3 documentation"
[9]: https://patroni.readthedocs.io/en/latest/kubernetes.html?utm_source=chatgpt.com "Using Patroni with Kubernetes"
[10]: https://www.postgresql.org/docs/current/index.html?utm_source=chatgpt.com "PostgreSQL 18.4 Documentation"
[11]: https://pgbackrest.org/?utm_source=chatgpt.com "pgBackRest - Reliable PostgreSQL Backup & Restore"
[12]: https://www.postgresql.org/docs/current/logical-replication-failover.html?utm_source=chatgpt.com "Documentation: 18: 29.3. Logical Replication Failover"
