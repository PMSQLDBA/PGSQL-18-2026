# Topic 27 — PostgreSQL Containers & Kubernetes

These notes are verified against the current PostgreSQL official documentation. As of **August 19, 2026**, the PostgreSQL `current` documentation points to **PostgreSQL 18**, and the official documentation page currently identifies the manual as **PostgreSQL 18.4**. ([PostgreSQL][1])

One important distinction: PostgreSQL itself documents database behavior such as WAL, replication, backup, recovery, failover, and configuration. Kubernetes deployment patterns such as StatefulSets, PersistentVolumes, and operators come from Kubernetes and operator documentation rather than the PostgreSQL core manual.

---

## 1. Why PostgreSQL is different from a normal containerized application

A normal stateless application can usually be treated like this:

```text
Application Pod dies
       ↓
Kubernetes creates another Pod
       ↓
Application continues
```

PostgreSQL is different because its important state is stored on disk.

```text
PostgreSQL
   |
   +-- Data files
   +-- WAL
   +-- Configuration
   +-- Transaction state
   +-- System catalogs
   +-- Indexes
```

If a PostgreSQL container disappears and its database files disappear with it, the database is gone.

Kubernetes explicitly describes container filesystems as ephemeral and provides persistent volumes so data can survive container or Pod replacement. ([Kubernetes][2])

Therefore the basic rule is:

> **The PostgreSQL process may live inside a container, but PostgreSQL data must live outside the container's ephemeral filesystem.**

---

# 2. PostgreSQL inside a container

A simple PostgreSQL container looks conceptually like this:

```text
┌─────────────────────────────┐
│ PostgreSQL Container        │
│                             │
│ postgres server process     │
│                             │
│ /var/lib/postgresql/data ────────┐
└─────────────────────────────┘    │
                                   │
                            Persistent Storage
```

The container contains:

* PostgreSQL binaries
* operating-system libraries
* startup scripts
* configuration/environment handling

The persistent volume contains the PostgreSQL database cluster.

A **PostgreSQL database cluster** in PostgreSQL terminology means the collection of databases managed by one PostgreSQL server instance, stored under the PostgreSQL data directory. `initdb` creates this cluster and initializes system catalogs plus the default `postgres`, `template0`, and `template1` databases. ([PostgreSQL][3])

---

# 3. Containers are replaceable; PostgreSQL data is not

A good Kubernetes architecture assumes:

```text
Pod       → disposable
Container → disposable
Node      → potentially disposable
Storage   → persistent
Database  → recoverable
```

Never design PostgreSQL so that recovering a failed Pod depends on recovering that exact container.

Instead:

```text
PostgreSQL Pod dies
       ↓
Kubernetes schedules replacement
       ↓
PVC reconnects to PersistentVolume
       ↓
PostgreSQL reads existing PGDATA
       ↓
Crash recovery / startup
       ↓
Database returns
```

PostgreSQL uses **Write-Ahead Logging (WAL)** to maintain database consistency and perform crash recovery.

---

# 4. Why a Kubernetes Deployment is usually insufficient

A Kubernetes `Deployment` works especially well for stateless workloads.

For example:

```text
Deployment
   ├── app-65df7
   ├── app-a83bc
   └── app-f93da
```

Those Pods are interchangeable.

PostgreSQL replicas are not completely interchangeable.

Suppose we have:

```text
postgres-0 = primary
postgres-1 = standby
postgres-2 = standby
```

Each instance has:

* its own database storage
* replication state
* WAL position
* server identity
* possibly replication slots and other role-specific state

That is why **StatefulSets or database operators** are normally used.

---

# 5. Kubernetes StatefulSet

Kubernetes describes StatefulSets as useful when applications require:

* stable unique network identities
* stable persistent storage
* ordered deployment/scaling
* ordered rolling updates. ([Kubernetes][4])

A StatefulSet might create:

```text
postgres-0
postgres-1
postgres-2
```

Rather than random Pod names.

Each Pod keeps a stable identity.

```text
StatefulSet: postgres

postgres-0
   |
   └── PVC: data-postgres-0

postgres-1
   |
   └── PVC: data-postgres-1

postgres-2
   |
   └── PVC: data-postgres-2
```

If `postgres-1` dies:

```text
postgres-1 dies
      ↓
replacement postgres-1 created
      ↓
same PVC attached
      ↓
same PostgreSQL data available
```

This is one major reason StatefulSets fit database workloads.

---

# 6. StatefulSet does NOT automatically give PostgreSQL HA

This is an important interview and architecture point.

A StatefulSet provides Kubernetes identity and lifecycle behavior.

It does **not** understand:

```text
PostgreSQL primary
PostgreSQL standby
WAL position
replication lag
promotion
split brain
timeline changes
rewind
backup
PITR
```

For example:

```text
StatefulSet
 replicas: 3
```

does **not** automatically mean:

```text
1 PostgreSQL primary
2 PostgreSQL replicas
automatic failover
```

Something still needs to manage PostgreSQL replication and leadership.

That might be:

```text
CloudNativePG
Patroni
Crunchy PostgreSQL Operator
Zalando Postgres Operator
other PostgreSQL operators
```

---

# 7. PersistentVolume, PVC and StorageClass

Three Kubernetes concepts are central.

## PersistentVolume — PV

A PersistentVolume represents durable storage available to Kubernetes.

Conceptually:

```text
PersistentVolume
       ↓
Cloud disk / SAN / CSI volume
```

Kubernetes defines a PV as a storage resource in the cluster. ([Kubernetes][5])

Examples of underlying storage could include:

```text
AWS EBS
Azure Managed Disk
Google Persistent Disk
Ceph RBD
SAN
other CSI-managed storage
```

---

# 8. PersistentVolumeClaim — PVC

A Pod usually doesn't directly request a particular physical disk.

Instead it creates a request:

```text
PersistentVolumeClaim
```

Example conceptually:

```yaml
resources:
  requests:
    storage: 500Gi
```

Kubernetes then finds or dynamically creates a matching PersistentVolume.

Architecture:

```text
PostgreSQL Pod
      |
      v
PVC
      |
      v
PV
      |
      v
Physical / cloud storage
```

---

# 9. StorageClass

A `StorageClass` tells Kubernetes how storage should be provisioned.

For example:

```text
StorageClass: postgres-fast

Provisioner:
    cloud CSI driver

Characteristics:
    SSD
    encrypted
    particular performance tier
```

Then:

```text
PVC
 |
 | requests StorageClass postgres-fast
 ↓
PV dynamically created
```

This avoids manually provisioning a disk for every PostgreSQL instance.

---

# 10. PostgreSQL storage characteristics matter

A PostgreSQL database does a lot more than simply store files.

Its workload involves:

```text
random reads
random writes
sequential WAL writes
fsync
checkpoints
background writer activity
temporary files
VACUUM I/O
```

Storage therefore strongly affects performance.

Important properties include:

```text
IOPS
latency
throughput
fsync latency
durability
failure behavior
snapshot behavior
volume attachment time
```

For transactional databases, low latency is often more important than simply having large throughput numbers.

---

# 11. PostgreSQL WAL

WAL means:

> **Write-Ahead Log**

The core principle is:

```text
Change database page
       ↓
Write WAL first
       ↓
Then modified data page may reach disk
```

This allows PostgreSQL to recover after crashes.

PostgreSQL 18's default `wal_level` is `replica`, which records enough information to support WAL archiving and physical replication. ([PostgreSQL][6])

Conceptually:

```text
Application transaction
        ↓
PostgreSQL
        ↓
WAL generated
        ↓
WAL flushed
        ↓
COMMIT acknowledged
        ↓
Data pages eventually written
```

WAL underpins:

* crash recovery
* physical replication
* continuous archiving
* Point-in-Time Recovery
* standby creation

---

# 12. PostgreSQL replication inside Kubernetes

A common architecture is:

```text
                 ┌──────────────┐
Writes ─────────▶│ Primary      │
                 │ postgres-1   │
                 └──────┬───────┘
                        │
                  WAL streaming
                  ┌─────┴─────┐
                  ▼           ▼
             postgres-2   postgres-3
              Standby      Standby
```

PostgreSQL documentation describes streaming replication using a primary that sends WAL and standby servers that receive it. A cascading standby may itself receive and forward WAL. ([PostgreSQL][7])

---

# 13. Physical streaming replication

Physical replication works at the WAL/storage level.

Primary:

```text
INSERT
UPDATE
DELETE
   ↓
WAL records
   ↓
WAL sender
```

Standby:

```text
WAL receiver
   ↓
receive WAL
   ↓
replay WAL
   ↓
database replica
```

This makes the standby essentially a physical copy of the primary database cluster.

---

# 14. Asynchronous replication

Default PostgreSQL streaming replication architectures frequently use asynchronous replication.

```text
Application
    |
    v
Primary
    |
 COMMIT
    |
Response returned
    |
    +--------> WAL later reaches standby
```

Advantage:

```text
lower transaction latency
```

Disadvantage:

```text
possible data loss during sudden primary failure
```

Example:

```text
Primary commits transaction T100
         ↓
Client receives success
         ↓
Primary crashes before standby gets T100
         ↓
Standby promoted
         ↓
T100 may be missing
```

That is an RPO consideration.

---

# 15. Synchronous replication

With synchronous replication:

```text
Client
  |
  v
Primary
  |
send WAL
  |
  v
Synchronous standby
  |
acknowledge
  |
  v
Primary acknowledges commit
```

Potential benefit:

```text
much smaller / potentially zero replication-loss window
```

Trade-off:

```text
increased write latency
```

If synchronous replicas become unavailable, availability behavior depends on how synchronous replication is configured.

Architecture always involves a trade-off:

```text
Consistency
Availability
Latency
```

---

# 16. Hot standby

PostgreSQL standby servers can often accept read-only queries.

Architecture:

```text
                PostgreSQL Primary
                      |
               physical replication
              ┌───────┴───────┐
              ▼               ▼
         Read replica     Read replica
              ↑               ↑
              │               │
          read queries     read queries
```

Useful for:

* reports
* analytics
* read scaling
* some backup-related activities

But replicas should not automatically be treated as infinitely scalable read nodes because long-running standby queries can interact with WAL replay and recovery.

---

# 17. Service routing in Kubernetes

Applications should not normally connect directly to:

```text
postgres-0
```

because the primary might change.

Instead:

```text
Application
      |
      v
Service: postgres-rw
      |
      v
Current primary
```

Read workloads could use:

```text
Application
      |
      v
Service: postgres-ro
      |
      +------> replica-1
      +------> replica-2
```

This routing model is used by PostgreSQL Kubernetes operators.

CloudNativePG, for example, exposes `-rw` routing to the current primary and `-ro` routing for replicas. During failover, the operator redirects the read-write service to the newly promoted instance. ([CloudNativePG][8])

---

# 18. What is a Kubernetes Operator?

An operator adds application-specific control logic to Kubernetes.

Normal Kubernetes understands:

```text
Pods
Services
Volumes
Nodes
Scheduling
```

A PostgreSQL operator understands database concepts such as:

```text
PostgreSQL primary
standby
replication
failover
backup
restore
PITR
rolling update
switchover
connection routing
```

Conceptually:

```text
             Custom Resource
                   |
                   v
          PostgreSQL Operator
                   |
              reconcile()
                   |
       ┌───────────┼─────────────┐
       ▼           ▼             ▼
     Pods       Services        PVCs
       |
       v
 PostgreSQL cluster
```

---

# 19. Reconciliation loop

Kubernetes operators use the controller/reconciliation model.

Suppose you declare:

```text
Desired state:
3 PostgreSQL instances
```

Actual state:

```text
2 instances running
```

The operator sees:

```text
Desired ≠ Actual
```

and reconciles:

```text
create/rebuild another PostgreSQL instance
```

This is much more powerful than shell scripts continuously checking processes.

---

# 20. Custom Resource Definitions

Operators normally introduce a Kubernetes CRD.

Instead of manually creating:

```text
3 Pods
3 PVCs
services
replication configuration
credentials
failover scripts
backup Jobs
```

you might declare something conceptually like:

```yaml
apiVersion: postgresql.example.io/v1
kind: Cluster

spec:
  instances: 3

  storage:
    size: 500Gi
```

The exact API varies by operator.

The operator converts that high-level desired state into the required Kubernetes and PostgreSQL resources.

---

# 21. CloudNativePG pattern

CloudNativePG is a Kubernetes operator designed specifically for PostgreSQL lifecycle management. Current PostgreSQL community news describes it as managing bootstrapping, configuration, HA, routing, backup, and disaster recovery. ([PostgreSQL][9])

Typical architecture:

```text
                    Kubernetes
┌────────────────────────────────────────────┐
│                                            │
│          CloudNativePG Operator            │
│                    │                       │
│              reconciliation                │
│                    │                       │
│          PostgreSQL Cluster                │
│                                            │
│     ┌────────┐   ┌────────┐   ┌────────┐  │
│     │Primary │──▶│Replica │   │Replica │  │
│     │ Pod    │──▶│ Pod    │   │ Pod    │  │
│     └───┬────┘   └───┬────┘   └───┬────┘  │
│         │            │            │         │
│        PVC          PVC          PVC        │
│                                            │
└────────────────────────────────────────────┘
```

The fundamental idea is:

> Kubernetes handles infrastructure orchestration; CloudNativePG adds PostgreSQL-aware orchestration.

---

# 22. CloudNativePG primary/replica model

A typical cluster might contain:

```text
cluster-1  primary
cluster-2  standby
cluster-3  standby
```

Physical PostgreSQL replication keeps them synchronized.

CloudNativePG uses PostgreSQL native physical replication for HA and read scaling. ([CloudNativePG][10])

Applications use operator-managed services:

```text
mydb-rw
     ↓
current primary

mydb-ro
     ↓
replicas
```

This separates database clients from physical Pod identity.

---

# 23. CloudNativePG failure flow

Suppose:

```text
Primary = pod-1
Replica = pod-2
Replica = pod-3
```

Then `pod-1` fails.

Conceptually:

```text
pod-1 unavailable
        ↓
operator detects failure
        ↓
evaluate replicas
        ↓
promote suitable replica
        ↓
pod-2 becomes primary
        ↓
rw Service points to pod-2
        ↓
old primary rebuilt/rejoined
```

This is PostgreSQL-aware HA, not simply Kubernetes restarting a container.

---

# 24. Patroni pattern

Patroni is another widely used PostgreSQL HA framework.

The Patroni documentation describes it as a template for creating HA PostgreSQL solutions. The current Patroni documentation available during verification is **4.1.5**, released August 12, 2026. ([Patroni][11])

Conceptually:

```text
             Distributed coordination
                       |
            ┌──────────┼──────────┐
            |          |          |
         Patroni    Patroni    Patroni
            |          |          |
            v          v          v
         PG-1       PG-2       PG-3
        Primary    Replica    Replica
```

Patroni manages:

* leader election
* PostgreSQL promotion
* demotion
* failover
* replication configuration
* cluster membership

---

# 25. Patroni and Distributed Configuration Store

Traditional Patroni installations commonly use a DCS such as:

```text
etcd
Consul
ZooKeeper
```

Conceptually:

```text
Postgres node 1 ─┐
Postgres node 2 ─┼── Patroni ── DCS
Postgres node 3 ─┘
```

The DCS stores information used for cluster coordination and leadership.

Patroni's documentation recommends properly fault-tolerant consensus deployments—for example 3 or 5 DCS nodes for etcd/ZooKeeper/Consul-style deployments. ([Patroni][11])

---

# 26. Patroni running directly on Kubernetes

Patroni can also use Kubernetes itself as its coordination mechanism.

Patroni documentation says it can use Kubernetes objects to manage cluster state and the leader key, avoiding the need for a separate etcd deployment in this mode. ([Patroni][12])

Architecture:

```text
             Kubernetes API
                   |
          Endpoints / objects
                   |
       ┌───────────┼───────────┐
       │           │           │
    Patroni     Patroni     Patroni
       │           │           │
      PG1         PG2         PG3
```

Patroni currently recommends Kubernetes Endpoints mode over ConfigMaps for leader coordination because leader metadata and endpoint addresses can be updated together. ([Patroni][12])

---

# 27. CloudNativePG vs Patroni — conceptual difference

They overlap in functionality, but the architectural approach is different.

| CloudNativePG                                        | Patroni                                                   |
| ---------------------------------------------------- | --------------------------------------------------------- |
| Kubernetes-native PostgreSQL operator                | PostgreSQL HA framework                                   |
| Operator reconciles Kubernetes resources             | Patroni agents coordinate PostgreSQL instances            |
| Kubernetes CRDs central to lifecycle                 | Patroni configuration + DCS/Kubernetes coordination       |
| Handles provisioning/lifecycle/HA/backup integration | Primarily centered on PostgreSQL HA and leader management |
| Very Kubernetes-oriented                             | Can run both inside and outside Kubernetes                |

You should understand the patterns rather than simply memorize product names.

---

# 28. The biggest PostgreSQL HA problem: split brain

Suppose:

```text
Node A = primary
Node B = standby
```

Network failure occurs.

Both sides cannot communicate.

Dangerous scenario:

```text
Node A thinks:
"I am still primary"

Node B thinks:
"A is dead, therefore I become primary"
```

Now:

```text
Primary A accepts writes
Primary B accepts writes
```

This is:

> **split brain**

The databases diverge.

```text
Timeline A:
T1 → T2 → A3 → A4

Timeline B:
T1 → T2 → B3 → B4
```

You cannot simply combine physical database files afterward.

This is why leader election, fencing, consensus and failover coordination are critical.

---

# 29. Failover

PostgreSQL documentation defines the basic standby failover situation as:

```text
Primary fails
      ↓
Standby starts failover procedure
      ↓
Standby becomes primary
```

The current PostgreSQL manual explicitly discusses failover in this context. ([PostgreSQL][13])

But PostgreSQL alone doesn't provide a complete Kubernetes cluster manager.

That's where systems such as:

```text
CloudNativePG
Patroni
```

become important.

---

# 30. Failover vs switchover

These terms are often confused.

### Failover

Unexpected problem:

```text
Primary crashes
       ↓
Replica promoted
```

Usually automatic.

### Switchover

Planned role change:

```text
Primary A healthy
Replica B healthy
       ↓
controlled synchronization
       ↓
B promoted
A demoted
```

Used during:

* maintenance
* node replacement
* some upgrades
* planned infrastructure operations

Switchover is generally safer because both servers are available during the operation.

---

# 31. Rejoining an old primary

Suppose:

```text
A = original primary
B = replica
```

A fails.

```text
B promoted
```

Then A returns.

You cannot simply start A as though nothing happened because timelines may have diverged.

PostgreSQL provides:

```text
pg_rewind
```

for synchronizing a PostgreSQL cluster with another copy after their timelines have diverged. ([PostgreSQL][14])

Conceptually:

```text
Old primary A
     |
 pg_rewind
     |
     v
New primary B's history
     |
     v
A rejoins as standby
```

Otherwise the instance may need a new base backup.

---

# 32. Kubernetes node failure

Example:

```text
Node 1
  └── PostgreSQL Primary Pod

Node 2
  └── PostgreSQL Replica

Node 3
  └── PostgreSQL Replica
```

If Node 1 disappears:

```text
Node 1 unavailable
      ↓
Primary unavailable
      ↓
operator/Patroni detects problem
      ↓
replica promoted
      ↓
database Service redirected
```

Eventually Kubernetes may reschedule the old instance elsewhere.

But database availability should not necessarily wait for an entire disk to be detached, attached and PostgreSQL restarted.

That is why database replication provides a faster HA path.

---

# 33. Storage HA is not database HA

Suppose PostgreSQL uses a highly durable cloud disk.

That protects storage.

It does not necessarily provide instant database availability.

```text
Primary node dies
       ↓
detach disk
       ↓
find another node
       ↓
attach disk
       ↓
mount filesystem
       ↓
start PostgreSQL
       ↓
crash recovery
```

This can take time.

Compare with replication:

```text
Primary dies
      ↓
standby promoted
      ↓
traffic redirected
```

So production architectures often combine:

```text
Durable persistent storage
+
PostgreSQL replicas
+
Automated failover
+
Backups
```

---

# 34. Backups are NOT replicas

This must be remembered.

```text
Replication ≠ Backup
```

If someone executes:

```sql
DROP TABLE customers;
```

the operation is replicated.

```text
Primary:
customers dropped

Replica:
customers dropped
```

Similarly:

```text
DELETE without WHERE
application bug
logical corruption
malicious action
```

may all replicate.

Replication protects mainly against server/instance failure.

Backup protects against data loss and supports historical recovery.

---

# 35. PostgreSQL official backup categories

PostgreSQL 18's official manual identifies three fundamental backup approaches:

1. SQL dump
2. file-system-level backup
3. continuous archiving. ([PostgreSQL][15])

In Kubernetes production HA environments, continuous physical backup plus WAL archiving is especially important.

---

# 36. Logical backups

Common tools:

```text
pg_dump
pg_dumpall
pg_restore
```

Example:

```text
PostgreSQL
    |
 pg_dump
    |
    v
logical SQL/archive backup
```

Advantages:

* portable
* can restore specific databases or objects
* useful for migrations
* useful for logical recovery

Limitations:

* potentially slow for huge systems
* recovery time can be long
* not equivalent to continuous physical PITR protection

---

# 37. Physical base backup

PostgreSQL provides:

```text
pg_basebackup
```

It creates a base backup of a running PostgreSQL cluster.

Typical architecture:

```text
Primary
   |
   | replication protocol
   v
pg_basebackup
   |
   v
Base backup
```

The current PostgreSQL documentation notes that replication slots are generally preferred during `pg_basebackup` WAL streaming because they prevent required WAL from being removed before the backup has consumed it. ([PostgreSQL][16])

---

# 38. Base backup + WAL

A powerful PostgreSQL backup model is:

```text
Base backup
+
continuous WAL archive
```

Example:

```text
Sunday 00:00
     |
     v
Base Backup
     |
     + WAL001
     + WAL002
     + WAL003
     + WAL004
     + WAL005
```

Recovery reconstructs the database by:

```text
restore base backup
       ↓
replay WAL
       ↓
reach desired recovery position/time
```

---

# 39. Point-in-Time Recovery — PITR

Suppose:

```text
10:00 database healthy
10:01 transaction
10:02 transaction
10:03 accidental DELETE
10:04 more activity
```

You want:

```text
database state at 10:02:59
```

With:

```text
Base backup
+
archived WAL
```

PostgreSQL can recover to a chosen point.

```text
Base backup
    ↓
Replay WAL
    ↓
10:00
10:01
10:02
STOP
```

PostgreSQL's official continuous archiving documentation explains that recovery requires a continuous sequence of archived WAL files. ([PostgreSQL][17])

---

# 40. Why continuous WAL archiving matters in Kubernetes

Imagine:

```text
Primary Pod
Replica Pod
Replica Pod
```

A cluster-wide accident deletes the Kubernetes namespace.

If your only copies of data were:

```text
the three PVCs
```

you may lose everything depending on storage lifecycle policies.

Better architecture:

```text
Kubernetes PostgreSQL cluster
             |
             | WAL archive
             | base backups
             v
      Independent object storage
```

For example:

```text
PostgreSQL cluster
      |
      +---- base backups ----+
      |                      |
      +---- WAL archive -----+---> external object storage
```

The backup failure domain should ideally differ from the database failure domain.

---

# 41. CloudNativePG backup pattern

CloudNativePG supports online physical backup and WAL archiving and supports PITR-style recovery. ([CloudNativePG][18])

Conceptually:

```text
                 PostgreSQL cluster
                        |
              ┌─────────┴──────────┐
              │                    │
        Base backups          WAL archive
              │                    │
              └─────────┬──────────┘
                        ▼
                  Object Storage
```

Examples of object-storage-style targets in architecture can include:

```text
S3-compatible storage
Azure Blob-like storage
Google cloud object storage
```

depending on operator integration.

---

# 42. Backup should leave the Kubernetes cluster

A weak design:

```text
PostgreSQL PVC
       ↓
Backup PVC
       ↓
same cluster
same storage platform
same region
```

A disaster affecting that infrastructure could destroy both.

A stronger design:

```text
Kubernetes Cluster
       |
 PostgreSQL
       |
 backup/WAL
       |
       v
Independent object storage
       |
 optional replication
       v
Another region/account
```

Think in terms of **failure domains**.

---

# 43. RPO and RTO

Every PostgreSQL HA/DR design should start with these two questions.

## RPO — Recovery Point Objective

How much data can we afford to lose?

Example:

```text
RPO = 5 minutes
```

means:

```text
up to approximately 5 minutes of committed data loss may be acceptable
```

Desired RPO drives decisions about:

```text
synchronous replication
WAL archiving
backup frequency
cross-region replication
```

---

# 44. RTO — Recovery Time Objective

How long can the database remain unavailable?

Example:

```text
RTO = 10 minutes
```

means recovery should ideally complete within ten minutes.

Architecture choices affect RTO:

```text
automatic failover
number of replicas
storage attach latency
backup restoration speed
WAL replay volume
cross-region architecture
automation
```

---

# 45. Availability Zones

A bad HA configuration:

```text
Availability Zone A

Node1 → Primary
Node2 → Replica
Node3 → Replica
```

If AZ A goes down:

```text
entire database cluster unavailable
```

Better:

```text
AZ A            AZ B            AZ C

Primary         Replica         Replica
```

Now one zone failing doesn't necessarily remove every PostgreSQL instance.

Operators can use Kubernetes scheduling rules such as:

```text
node affinity
pod anti-affinity
topology spread constraints
```

to distribute instances.

---

# 46. Pod anti-affinity

Without placement controls, Kubernetes could do:

```text
Node 1
 ├── PostgreSQL primary
 ├── PostgreSQL replica 1
 └── PostgreSQL replica 2
```

Node 1 failure destroys all three database processes.

Better:

```text
Node 1
 └── primary

Node 2
 └── replica

Node 3
 └── replica
```

This is one reason pod anti-affinity or topology spreading matters for PostgreSQL.

---

# 47. Topology awareness

Ideal architecture:

```text
Kubernetes cluster

Zone A
   Node A1
      PostgreSQL primary

Zone B
   Node B1
      PostgreSQL replica

Zone C
   Node C1
      PostgreSQL replica
```

Now failure domains become:

```text
Pod
Node
Availability Zone
```

rather than everything sharing one physical dependency.

---

# 48. Readiness and liveness probes

Kubernetes needs to know whether a Pod is usable.

Two concepts:

```text
Liveness
Readiness
```

Liveness asks roughly:

> Is this container/process alive enough that Kubernetes should keep it running?

Readiness asks:

> Should this Pod currently receive traffic?

For PostgreSQL, readiness must be role-aware in sophisticated HA setups.

For example:

```text
PostgreSQL running
but
replication broken
or
Pod isn't current primary
```

may mean it should not receive certain application traffic.

Database operators generally implement richer health logic than simply checking whether TCP port 5432 accepts connections.

---

# 49. Don't use aggressive liveness probes

Database startup can legitimately take time.

Example:

```text
PostgreSQL starts
      ↓
crash recovery
      ↓
replays WAL
      ↓
becomes available
```

If Kubernetes repeatedly kills the process because a simplistic liveness probe expires during recovery:

```text
startup
 ↓
recovery
 ↓
probe fails
 ↓
container killed
 ↓
startup again
```

you create a failure loop.

Health probes must understand realistic PostgreSQL recovery times.

---

# 50. Graceful shutdown

When Kubernetes terminates a Pod:

```text
SIGTERM
   ↓
PostgreSQL shutdown
   ↓
connections close
   ↓
database process exits
```

Give PostgreSQL enough termination grace time.

Do not design database Pods as if they are stateless web servers that can simply be hard-killed immediately.

Planned shutdown reduces unnecessary crash recovery.

---

# 51. PodDisruptionBudget

Kubernetes maintenance might voluntarily evict Pods.

Suppose:

```text
3 PostgreSQL Pods
```

Without appropriate protection, maintenance could temporarily remove too many database instances.

A PodDisruptionBudget can help restrict voluntary disruptions.

Conceptually:

```text
At least 2 database Pods available
```

But a PDB alone does not make PostgreSQL HA.

It complements:

```text
replication
operator failover
topology placement
storage
```

---

# 52. Resource requests and limits

Databases require predictable resources.

Important resources:

```text
CPU
RAM
disk
IOPS
network bandwidth
```

Kubernetes scheduling should provide realistic requests.

For example:

```text
resources:
  requests:
    cpu: ...
    memory: ...
```

Under-requesting resources can place PostgreSQL on overloaded nodes.

Overly tight memory limits can cause PostgreSQL to be killed by the container runtime when memory usage exceeds the cgroup limit.

---

# 53. PostgreSQL memory in containers

Important settings include:

```text
shared_buffers
work_mem
maintenance_work_mem
autovacuum_work_mem
```

But Kubernetes memory includes more than `shared_buffers`.

Think roughly:

```text
Total memory =
shared_buffers
+ per-session memory
+ maintenance workers
+ autovacuum
+ backend overhead
+ kernel/filesystem behavior
+ extensions
+ process overhead
```

PostgreSQL's official docs currently note that `shared_buffers` defaults to 128 MB in many installations, although production systems normally require explicit tuning appropriate for available memory and workload. ([PostgreSQL][19])

---

# 54. Connection explosion in Kubernetes

A Kubernetes environment may contain:

```text
100 application Pods
```

Suppose each opens:

```text
20 PostgreSQL connections
```

Then:

```text
100 × 20 = 2,000 connections
```

Scale to 300 Pods:

```text
300 × 20 = 6,000 connections
```

PostgreSQL uses a process-per-connection model, so huge connection counts can be expensive.

That is why connection pooling is frequently deployed.

Common example:

```text
PgBouncer
```

Architecture:

```text
Application Pods
       |
       v
    PgBouncer
       |
       v
   PostgreSQL
```

---

# 55. Secrets

Never hard-code passwords inside:

```text
Docker image
Git repository
ConfigMap
plain manifest
```

Use Kubernetes Secrets or an external secrets system.

Typical secrets include:

```text
database password
replication password
backup credentials
object-storage credentials
TLS private keys
```

Operators can often automate PostgreSQL credential generation and management.

---

# 56. TLS

Consider encryption for:

```text
application → PostgreSQL
replica → primary
operator → management endpoints
backup → object storage
```

PostgreSQL supports TLS connections.

In production, certificate rotation and secret management must be part of the operational architecture rather than a one-time deployment task.

---

# 57. Monitoring PostgreSQL on Kubernetes

You need visibility into both layers.

### Kubernetes layer

Monitor:

```text
Pod health
restart count
node status
PVC capacity
storage latency
CPU
memory
network
operator status
```

### PostgreSQL layer

Monitor:

```text
connections
transactions
locks
long-running queries
replication lag
WAL generation
checkpoints
autovacuum
dead tuples
cache behavior
database growth
backup status
```

Monitoring only Kubernetes is not sufficient.

A Pod can be:

```text
Running
```

while PostgreSQL is operationally unhealthy.

---

# 58. Replication lag

A standby receives and replays WAL.

You can think of lag as:

```text
Primary WAL position
      -
Standby replay position
```

If lag becomes large:

```text
Primary
 WAL LSN 100000

Replica
 replay LSN 70000
```

then the standby is significantly behind.

Promoting a highly lagging asynchronous standby may increase data-loss exposure.

Operators must consider replica health during failover selection.

---

# 59. Replication slots

A physical replication slot tells PostgreSQL:

> WAL required by this consumer must not yet be removed.

Useful because a replica temporarily disconnected from the primary may still need older WAL.

But there is a major danger.

```text
Replica disconnected
       ↓
slot remains
       ↓
Primary retains WAL
       ↓
pg_wal grows
       ↓
disk fills
```

So monitor:

```text
replication slot activity
retained WAL
disk usage
```

Replication slots improve reliability but can create a disk-capacity risk if unmanaged.

---

# 60. WAL archive vs replication slot

They solve related but different problems.

### Replication slot

Primarily protects WAL required by a replication consumer.

```text
Primary
   |
 replication slot
   |
Replica
```

### WAL archive

Creates durable historical WAL copies.

```text
Primary
   |
archive WAL
   |
Object storage
```

Used for:

```text
PITR
backup
disaster recovery
```

A robust environment may use both.

---

# 61. `pg_basebackup`

For physical backup:

```bash
pg_basebackup
```

PostgreSQL can stream a base backup from a running server using the replication protocol. ([PostgreSQL][16])

It's used to:

```text
create physical backup
initialize standby
support PITR strategy
```

PostgreSQL 17 added incremental backup capability around the physical backup toolset, which continues into PostgreSQL 18. ([PostgreSQL][20])

---

# 62. `pg_rewind`

Useful after failover.

Scenario:

```text
A primary
B standby

A fails
B promoted

A returns
```

Their histories have now diverged.

Instead of always copying the entire cluster:

```text
pg_rewind
```

can synchronize the old copy with the current cluster under appropriate conditions. PostgreSQL explicitly describes `pg_rewind` as synchronizing clusters after timeline divergence. ([PostgreSQL][14])

---

# 63. Kubernetes snapshots are not automatically PostgreSQL backups

This is a subtle but important architecture point.

A CSI volume snapshot gives you:

```text
block/storage snapshot
```

but PostgreSQL backup consistency still matters.

A storage snapshot taken without database coordination may represent a crash-consistent image rather than a carefully managed PostgreSQL backup.

Operators that integrate PostgreSQL-aware backup workflows with snapshot technology are preferable to assuming:

```text
snapshot = guaranteed database backup
```

Always test recovery.

---

# 64. Restore testing

A backup is only useful if it can be restored.

Production practice should include:

```text
backup created
       ↓
backup verified
       ↓
restore test
       ↓
database consistency validation
       ↓
application validation
```

Important metrics:

```text
Backup success rate
Restore success rate
Actual recovery time
Available PITR window
Oldest recoverable point
```

Do not stop at:

```text
"Backup Job succeeded."
```

---

# 65. Kubernetes-level restore pattern

Typical disaster-recovery flow:

```text
Original cluster lost
       ↓
Create new PostgreSQL cluster
       ↓
Restore base backup
       ↓
Retrieve archived WAL
       ↓
Replay WAL
       ↓
Reach recovery target
       ↓
Promote
       ↓
Application reconnects
```

Operators can automate much of this workflow.

---

# 66. Region disaster recovery

HA inside one Kubernetes cluster is not necessarily DR.

Example:

```text
Region A
┌────────────────────────┐
│ Kubernetes             │
│ primary + 2 replicas   │
└────────────────────────┘
```

All three replicas may disappear if the region fails.

For DR:

```text
Region A                       Region B

Primary K8s                    DR K8s
cluster                        cluster
    |                             ^
    | replication / backup        |
    +-----------------------------+
```

CloudNativePG documentation discusses replica/distributed topologies across Kubernetes clusters as a DR architecture. ([CloudNativePG][21])

---

# 67. Why automatic cross-region failover is dangerous

Consider:

```text
Region A     X network X      Region B

Primary                      Standby
```

Region B cannot tell whether:

```text
Region A is dead
```

or:

```text
only the network is broken
```

If Region B automatically promotes while A is still accepting writes:

```text
split brain
```

Patroni's multi-datacenter guidance explicitly discusses this problem and recommends manual promotion in certain two-site asynchronous standby-cluster architectures because the remote site cannot prove the original site is down. ([Patroni][22])

---

# 68. Backup HA and database HA are separate

A mature architecture has several protection layers.

```text
Layer 1
PostgreSQL WAL/crash recovery

Layer 2
Local replicas

Layer 3
Automated failover

Layer 4
Persistent volumes

Layer 5
Base backups + WAL archive

Layer 6
Off-site / cross-region backup

Layer 7
Disaster recovery environment
```

No single layer replaces all the others.

---

# 69. Typical production PostgreSQL Kubernetes architecture

```text
                     Applications
                          |
                          v
                ┌──────────────────┐
                │ PostgreSQL RW    │
                │ Service          │
                └────────┬─────────┘
                         |
                         v
                ┌──────────────────┐
                │ Primary          │
                │ PostgreSQL       │
                └────────┬─────────┘
                         |
                  WAL replication
                ┌────────┴────────┐
                ▼                 ▼
        ┌──────────────┐    ┌──────────────┐
        │ Replica      │    │ Replica      │
        │ Zone B       │    │ Zone C       │
        └──────┬───────┘    └──────┬───────┘
               │                   │
              PVC                 PVC
               │                   │
              PV                  PV

Primary also has:
        │
       PVC
        │
       PV

                       WAL / backups
                             |
                             v
                       Object Storage
                             |
                             v
                         DR copy
```

Above everything:

```text
CloudNativePG Operator
or
Patroni/operator architecture
```

handles database-specific lifecycle and HA decisions.

---

# 70. Operator failure does not necessarily mean database failure

An important architecture principle:

```text
Operator
   ↓
manages PostgreSQL
```

The operator should not sit in the database query path.

Application traffic normally goes:

```text
Application
     ↓
Service
     ↓
PostgreSQL
```

not:

```text
Application
     ↓
Operator
     ↓
PostgreSQL
```

Therefore temporary operator unavailability should not automatically stop existing PostgreSQL traffic, although automated management actions such as failover or reconciliation may be affected.

---

# 71. Major PostgreSQL upgrades

Container image replacement does not magically upgrade PostgreSQL's on-disk data format between major releases.

For example:

```text
PostgreSQL 17 data directory
```

cannot simply be assumed to become PostgreSQL 18 because you changed:

```text
image: postgres:17
```

to:

```text
image: postgres:18
```

PostgreSQL officially documents three major-upgrade approaches:

```text
pg_dump/pg_dumpall
pg_upgrade
replication-based upgrade
```

([PostgreSQL][23])

Operators need explicit major-upgrade strategies.

---

# 72. Minor version updates

Within a PostgreSQL major version:

```text
18.x → newer 18.x
```

generally involves replacing the binaries/container image and restarting instances.

A HA operator may perform something conceptually like:

```text
update replicas
       ↓
verify replica
       ↓
controlled switchover
       ↓
update former primary
```

The exact process depends on the operator.

---

# 73. Rolling update consideration

For stateless apps:

```text
replace any Pod
```

For PostgreSQL:

```text
which Pod is primary?
how far are replicas behind?
does the update require restart?
should we switchover first?
are enough replicas healthy?
```

Database-aware operators therefore provide value beyond basic StatefulSet rolling updates.

---

# 74. Scaling PostgreSQL is not the same as scaling stateless Pods

For web applications:

```text
replicas: 3 → replicas: 20
```

might work easily.

For PostgreSQL:

```text
3 replicas → 20 replicas
```

creates:

```text
20 full copies of database storage
20 replication streams
more WAL consumers
more network traffic
more storage cost
```

Kubernetes itself warns that not every stateful application scales simply by increasing StatefulSet replicas. ([Kubernetes][24])

---

# 75. Scaling writes

Physical PostgreSQL replication normally has:

```text
one writable primary
multiple read replicas
```

Therefore adding replicas scales:

```text
reads
```

more naturally than:

```text
writes
```

Write scaling may require architectural techniques such as:

```text
partitioning
sharding
application-level data distribution
distributed PostgreSQL solutions
```

depending on requirements.

---

# 76. Kubernetes database anti-patterns

Avoid architectures like:

```text
PostgreSQL Deployment
+
emptyDir
```

because Pod replacement can remove the database.

Also avoid:

```text
single PostgreSQL Pod
+
single PVC
+
no backup
```

for a critical production system.

Other common problems:

```text
all replicas on same node
all replicas in same AZ
backup stored beside database
no PITR
no restore testing
automatic unsafe cross-region failover
unmonitored replication slots
no connection pooling
latest/unpinned container images
```

---

# 77. Never rely only on `latest` container tags

Avoid production configuration like:

```yaml
image: postgres:latest
```

because the meaning of `latest` can change.

Prefer an explicitly controlled version.

Conceptually:

```yaml
image: postgres:18.x
```

or an organization-approved immutable digest.

This makes upgrades deliberate and reproducible.

---

# 78. Production design checklist

For an enterprise PostgreSQL Kubernetes deployment, examine these areas:

* **Database topology:** primary plus at least appropriate standby capacity.
* **Placement:** replicas distributed across nodes and preferably availability zones.
* **Storage:** reliable persistent CSI storage with known latency/IOPS characteristics.
* **HA controller:** CloudNativePG, Patroni-based architecture, or another mature PostgreSQL operator.
* **Replication:** decide asynchronous versus synchronous according to RPO and latency.
* **Routing:** applications connect through role-aware services rather than Pod identities.
* **Backups:** physical base backup plus WAL archiving for PITR where appropriate.
* **Backup isolation:** keep recoverable copies outside the database failure domain.
* **DR:** explicitly define cross-region recovery instead of assuming local HA equals DR.
* **Observability:** PostgreSQL metrics plus Kubernetes/operator metrics.
* **Connections:** use pooling where application Pod counts create excessive backend connections.
* **Security:** Secrets, TLS, RBAC and restricted backup credentials.
* **Maintenance:** controlled minor updates, explicit major-version upgrade process.
* **Testing:** routinely test failover and restore procedures.

---

# 79. CloudNativePG vs raw StatefulSet

A useful way to remember the difference:

```text
Raw StatefulSet
--------------------------------
Stable Pod names               ✓
Stable PVCs                    ✓
Ordered Pods                   ✓
PostgreSQL replication logic   ✗
Primary election               ✗
Safe failover                  ✗
PITR management                ✗
PostgreSQL-aware upgrades      ✗
```

With a PostgreSQL operator:

```text
Operator
--------------------------------
Stable PostgreSQL instances    ✓
Replication management         ✓
Primary/standby awareness      ✓
Failover orchestration         ✓
Service routing                ✓
Backup automation              ✓
Recovery workflow              ✓
Lifecycle automation           ✓
```

The exact capabilities depend on the chosen operator and version.

---

# 80. StatefulSet vs Operator vs Patroni

Think of the layers this way:

```text
Kubernetes StatefulSet
        |
        | solves
        v
Pod identity + persistent storage lifecycle
```

```text
Patroni
        |
        | solves
        v
PostgreSQL leadership + HA coordination
```

```text
PostgreSQL Kubernetes Operator
        |
        | solves
        v
Database lifecycle management
+
HA
+
storage orchestration
+
backup/recovery
+
Kubernetes integration
```

That mental model is much more useful than treating them as equivalent products.

---

# 81. Example failure scenario

Assume:

```text
3 PostgreSQL instances

PG-1 primary
PG-2 standby
PG-3 standby
```

PG-1 node fails.

### Step 1

Operator notices PG-1 is unavailable.

```text
PG-1 X
```

### Step 2

Operator examines standby health.

```text
PG-2 lag = very low
PG-3 lag = higher
```

### Step 3

Suitable replica is promoted.

```text
PG-2 → Primary
```

### Step 4

Read-write service changes.

```text
postgres-rw
     |
     v
PG-2
```

### Step 5

Applications reconnect.

### Step 6

PG-1 may later be:

```text
rewound
or
rebuilt from primary/base backup
```

### Step 7

Cluster returns to:

```text
1 primary
2 replicas
```

This is the essence of PostgreSQL HA orchestration.

---

# 82. Example corruption scenario

Now consider a completely different failure.

At 14:00:

```sql
DELETE FROM orders;
```

accidentally runs.

Replication immediately copies the change.

```text
Primary       orders deleted
Replica 1     orders deleted
Replica 2     orders deleted
```

Failover cannot help.

Instead:

```text
Backup
+
WAL archive
+
PITR
```

might recover to:

```text
13:59:59
```

This illustrates:

> **HA solves service failure. Backup/PITR solves historical data recovery.**

---

# 83. Example disaster scenario

Now assume the entire Kubernetes cluster disappears.

```text
Kubernetes cluster X

Primary X
Replica X
Replica X
PVCs X
```

Local PostgreSQL replication is useless.

Recovery requires something outside that failure domain:

```text
Object-storage backup
+
Archived WAL
```

Then:

```text
new Kubernetes cluster
       ↓
new PostgreSQL cluster
       ↓
restore base backup
       ↓
replay WAL
       ↓
recover
```

This is disaster recovery rather than local high availability.

---

# 84. Interview question: Why StatefulSet for PostgreSQL?

A strong answer:

> PostgreSQL is stateful, so every instance needs persistent storage and a stable identity. A StatefulSet provides stable Pod names and stable PVC association, which makes it more appropriate than a standard Deployment. However, a StatefulSet alone does not manage PostgreSQL replication, leader election or failover, so production deployments normally add PostgreSQL-aware automation such as an operator or Patroni.

---

# 85. Interview question: Why use CloudNativePG instead of manually creating StatefulSets?

A strong answer:

> StatefulSets solve Kubernetes lifecycle and identity problems, but they do not understand PostgreSQL. CloudNativePG understands primary and replica roles, streaming replication, failover, service routing, backup, recovery and cluster lifecycle. That removes a lot of custom operational logic that would otherwise have to be implemented and maintained manually.

CloudNativePG's documentation describes this reconciliation-based database lifecycle approach directly. ([CloudNativePG][25])

---

# 86. Interview question: Does Kubernetes restart equal PostgreSQL failover?

No.

```text
Kubernetes restart
```

means:

```text
restart/reschedule a Pod
```

PostgreSQL failover means:

```text
promote another database instance
+
change primary role
+
redirect write traffic
+
maintain replication topology
```

They solve different problems.

---

# 87. Interview question: What happens to a PVC when a StatefulSet Pod dies?

Normally:

```text
Pod dies
     ↓
PVC remains
     ↓
replacement Pod uses persistent storage
```

StatefulSet-created PVCs are retained by default unless retention behavior is explicitly changed. Current Kubernetes API documentation confirms that claims from `volumeClaimTemplates` are retained by default. ([Kubernetes][26])

That default protects state from disappearing merely because a Pod or StatefulSet lifecycle event occurred.

---

# 88. Interview question: Is three replicas enough for HA?

The correct response is:

> It depends on where they run and how failover works.

This:

```text
3 replicas
same node
```

is poor HA.

This:

```text
3 replicas
different nodes
same AZ
```

protects against node failure but not AZ failure.

Better:

```text
3 replicas
across 3 zones
```

if the infrastructure and latency model supports it.

Then also consider:

```text
storage failure domains
replication mode
operator reliability
backup
DR
```

---

# 89. Interview question: synchronous or asynchronous replication?

Use the business requirement.

### Asynchronous

```text
+ Lower commit latency
+ Better independence from replica latency
- Possible committed-data loss during sudden primary failure
```

### Synchronous

```text
+ Stronger durability across replicas
+ Can reduce/avoid failover data loss depending on configuration
- Higher transaction latency
- Availability can be affected if required synchronous standbys disappear
```

The decision is fundamentally driven by:

```text
RPO
latency
availability
```

---

# 90. Interview question: How do you back up PostgreSQL on Kubernetes?

A good architecture answer:

> I would separate HA from backup. Local PostgreSQL replicas provide availability, but I would also create PostgreSQL-aware physical base backups and continuously archive WAL to durable storage outside the cluster. That provides PITR and protects against logical mistakes or complete Kubernetes-cluster failure. I would also regularly test the restore process and measure actual RPO and RTO.

This aligns directly with PostgreSQL's official base-backup and continuous-archiving architecture. ([PostgreSQL][17])

---

# 91. Interview question: Why isn't a replica a backup?

Because replication copies changes, including bad ones.

```text
DROP TABLE
     ↓
Primary
     ↓
replication
     ↓
Standby also drops table
```

A historical backup + WAL archive may allow recovery before that operation.

---

# 92. Interview question: What is the biggest risk with replication slots?

WAL retention.

```text
inactive replica
      +
replication slot
      ↓
old WAL retained
      ↓
pg_wal keeps growing
      ↓
filesystem may fill
```

Therefore replication-slot lag and retained WAL must be monitored.

---

# 93. Interview question: What does Patroni solve?

A concise answer:

> Patroni provides PostgreSQL HA coordination. It manages PostgreSQL node roles, leader election, promotion, failover and cluster state using a distributed coordination system such as etcd/Consul/ZooKeeper or Kubernetes objects when operating natively on Kubernetes. ([Patroni][12])

---

# 94. Interview question: CloudNativePG or Patroni?

Don't answer with "X is always better."

Use architecture requirements.

Choose based on:

```text
Kubernetes-native lifecycle requirements
existing Patroni expertise
backup integration
upgrade automation
observability
supported PostgreSQL versions
cloud/storage integration
operational model
community/vendor support
organizational standards
```

CloudNativePG is naturally Kubernetes/operator-centered.

Patroni is strongly HA-centered and works beyond Kubernetes too.

---

# 95. Mental model to remember the whole topic

Use this stack:

```text
┌───────────────────────────────┐
│ Applications                  │
├───────────────────────────────┤
│ PostgreSQL Service / Pooler   │
├───────────────────────────────┤
│ Primary + replicas            │
├───────────────────────────────┤
│ CloudNativePG / Patroni       │
├───────────────────────────────┤
│ Kubernetes Stateful Workload  │
├───────────────────────────────┤
│ PVC                           │
├───────────────────────────────┤
│ PersistentVolume / CSI        │
├───────────────────────────────┤
│ Cloud / physical storage      │
└───────────────────────────────┘

                 +
        Base Backup + WAL
                 |
                 v
        Independent Backup
            Storage
```

---

# 96. Final architecture rule

For production PostgreSQL on Kubernetes, think of five independent concerns:

```text
1. COMPUTE
   Pods / containers / nodes

2. STORAGE
   PVC / PV / CSI

3. DATABASE HA
   Primary / standby / streaming replication

4. ORCHESTRATION
   Operator / Patroni / failover logic

5. DATA PROTECTION
   Base backup / WAL / PITR / DR
```

Do **not** assume one solves another.

A clean summary is:

> **StatefulSet keeps identity and storage stable. PostgreSQL streaming replication keeps database copies synchronized. CloudNativePG or Patroni provides database-aware HA orchestration. PersistentVolumes protect data across Pod replacement. Base backups plus WAL archiving provide PITR and disaster recovery.**

### Official documentation verification

For this topic I verified the core database statements against the **PostgreSQL 18 current documentation**, including the official chapters for Backup and Restore, Continuous Archiving/PITR, High Availability, Streaming Replication, replication configuration, `pg_basebackup`, `pg_rewind`, and WAL configuration. The PostgreSQL site currently lists the PostgreSQL 18 manual and release series, with the indexed current documentation showing **18.4** at the time of verification. ([PostgreSQL][1])

For the Kubernetes-specific pieces, I cross-checked the official Kubernetes documentation for StatefulSets, volumes and PersistentVolumes. ([Kubernetes][4]) For the implementation patterns, I also verified the current CloudNativePG and Patroni documentation; Patroni's latest documentation retrieved is **4.1.5, released August 12, 2026**. ([Patroni][27])

[1]: https://www.postgresql.org/docs/current/index.html?utm_source=chatgpt.com "PostgreSQL 18.4 Documentation"
[2]: https://kubernetes.io/docs/concepts/storage/volumes/?utm_source=chatgpt.com "Volumes"
[3]: https://www.postgresql.org/docs/current/app-initdb.html?utm_source=chatgpt.com "Documentation: 18: initdb"
[4]: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/?utm_source=chatgpt.com "StatefulSets"
[5]: https://kubernetes.io/docs/concepts/storage/persistent-volumes/?utm_source=chatgpt.com "Persistent Volumes"
[6]: https://www.postgresql.org/docs/current/runtime-config-wal.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.5. Write Ahead Log"
[7]: https://www.postgresql.org/docs/current/runtime-config-replication.html?utm_source=chatgpt.com "Documentation: 18: 19.6. Replication"
[8]: https://cloudnative-pg.io/documentation/1.18/architecture/?utm_source=chatgpt.com "Architecture"
[9]: https://www.postgresql.org/about/news/cloudnativepg-1290-released-3266/?utm_source=chatgpt.com "CloudNativePG 1.29.0 Released!"
[10]: https://cloudnative-pg.io/documentation/1.18/replication/?utm_source=chatgpt.com "Replication"
[11]: https://patroni.readthedocs.io/en/latest/README.html?utm_source=chatgpt.com "Introduction — Patroni 4.1.5 documentation"
[12]: https://patroni.readthedocs.io/en/latest/kubernetes.html?utm_source=chatgpt.com "Using Patroni with Kubernetes"
[13]: https://www.postgresql.org/docs/current/warm-standby-failover.html?utm_source=chatgpt.com "Documentation: 18: 26.3. Failover"
[14]: https://www.postgresql.org/docs/current/app-pgrewind.html?utm_source=chatgpt.com "Documentation: 18: pg_rewind"
[15]: https://www.postgresql.org/docs/current/backup.html?utm_source=chatgpt.com "Documentation: 18: Chapter 25. Backup and Restore"
[16]: https://www.postgresql.org/docs/current/app-pgbasebackup.html?utm_source=chatgpt.com "Documentation: 18: pg_basebackup"
[17]: https://www.postgresql.org/docs/current/continuous-archiving.html?utm_source=chatgpt.com "25.3. Continuous Archiving and Point-in-Time Recovery ..."
[18]: https://cloudnative-pg.io/documentation/current/appendixes/backup_barmanobjectstore/?utm_source=chatgpt.com "Appendix B - Backup on object stores"
[19]: https://www.postgresql.org/docs/current/runtime-config-resource.html?utm_source=chatgpt.com "Documentation: 18: 19.4. Resource Consumption"
[20]: https://www.postgresql.org/docs/17/release-17.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 17: E.11. Release 17"
[21]: https://cloudnative-pg.io/documentation/1.24/replica_cluster/?utm_source=chatgpt.com "Replica clusters - CloudNativePG v1.24"
[22]: https://patroni.readthedocs.io/en/latest/ha_multi_dc.html?utm_source=chatgpt.com "HA multi datacenter — Patroni 4.1.5 documentation"
[23]: https://www.postgresql.org/docs/current/upgrading.html?utm_source=chatgpt.com "Documentation: 18: 18.6. Upgrading a PostgreSQL Cluster"
[24]: https://kubernetes.io/docs/tasks/run-application/scale-stateful-set/?utm_source=chatgpt.com "Scale a StatefulSet"
[25]: https://cloudnative-pg.io/documentation/1.20/faq/?utm_source=chatgpt.com "Frequently Asked Questions (FAQ)"
[26]: https://kubernetes.io/docs/reference/kubernetes-api/apps/stateful-set-v1/?utm_source=chatgpt.com "StatefulSet"
[27]: https://patroni.readthedocs.io/en/latest/releases.html?highlight=1.6.5&utm_source=chatgpt.com "Release notes — Patroni 4.1.5 documentation"
