# Topic 26 — PostgreSQL Cloud Architecture

These notes are based on the **current PostgreSQL 18 documentation** and current official AWS, Microsoft Azure, and Google Cloud documentation. 

As of August 19, 2026, the PostgreSQL documentation marks **PostgreSQL 18 as the current supported major version**, with **18.4** as the latest production release shown in the official docs. 

PostgreSQL 19 is still a development/beta line, so PostgreSQL 18 documentation is the correct production reference. ([PostgreSQL][1])

---

## 1. What is PostgreSQL Cloud Architecture?

PostgreSQL Cloud Architecture means designing and operating PostgreSQL databases using cloud infrastructure.

There are two main approaches:

**Managed PostgreSQL**

The cloud provider operates much of the database infrastructure for you.

Examples:

* AWS Amazon RDS for PostgreSQL
* AWS Amazon Aurora PostgreSQL-Compatible Edition
* Azure Database for PostgreSQL
* Google Cloud SQL for PostgreSQL

**Self-managed PostgreSQL**

You install PostgreSQL yourself on:

* AWS EC2
* Azure Virtual Machines
* Google Compute Engine
* Kubernetes
* physical or private-cloud servers

The major architectural decision is therefore:

> **How much database infrastructure responsibility should the cloud provider handle, and how much control does the organization need?**

---

# 2. PostgreSQL Architecture Refresher

Before comparing cloud services, understand what the cloud platforms are managing underneath.

A traditional PostgreSQL architecture can be represented as:

```text
Application
     |
     v
PostgreSQL Server
     |
     +---------------------+
     |                     |
Shared Memory        Background Processes
     |
     v
Database Files
     |
     v
Persistent Storage
```

Important components include:

```text
Client Connections
       |
       v
PostgreSQL Backend Processes
       |
       +---- Shared Buffers
       +---- WAL Buffers
       +---- Work Memory
       |
       v
PostgreSQL Storage
       |
       +---- Tables
       +---- Indexes
       +---- WAL
       +---- System Catalogs
```

In cloud architecture, the cloud provider usually manages some or all of:

```text
Infrastructure
Storage
Server provisioning
OS patching
PostgreSQL binaries
Backups
Monitoring
Replication
Failover
Hardware replacement
```

---

# 3. PostgreSQL WAL and Cloud Architecture

A central PostgreSQL concept for cloud HA and disaster recovery is **Write-Ahead Logging — WAL**.

Instead of immediately relying on modified database files for recovery, PostgreSQL first records changes into WAL.

Conceptually:

```text
Transaction
    |
    v
Generate WAL
    |
    v
WAL becomes durable
    |
    v
Database pages written later
```

WAL supports:

* crash recovery
* physical replication
* standby databases
* Point-in-Time Recovery
* backup recovery
* logical decoding

The current PostgreSQL documentation confirms that streaming replication transfers WAL records from a primary server to standby servers as the WAL is generated. ([PostgreSQL][2])

---

# 4. PostgreSQL Replication Architecture

A normal architecture looks like:

```text
             Writes
Application ---------> PostgreSQL Primary
                           |
                           | WAL
                           |
             +-------------+-------------+
             |                           |
             v                           v
        Standby 1                     Standby 2
        Read only                     Read only
```

PostgreSQL describes these standby models as **warm standby** and **hot standby**.

A hot standby can accept read-only queries while continuously applying WAL from the primary. ([PostgreSQL][3])

---

# 5. Synchronous vs Asynchronous Replication

This is critical for cloud architecture.

## Synchronous replication

The primary waits until the standby acknowledges replication before considering the transaction fully committed according to the configured synchronous policy.

Conceptually:

```text
Application
    |
    | COMMIT
    v
Primary
    |
    | WAL
    v
Standby
    |
    | ACK
    v
Primary
    |
    v
Application receives success
```

Advantages:

* very low potential data loss
* useful for high availability

Disadvantages:

* additional write latency
* standby/network problems can affect write availability depending on configuration

---

## Asynchronous replication

The primary commits without waiting for the replica.

```text
Application
    |
    v
Primary
    |
    | COMMIT successful
    |
    +------ WAL ------> Replica
```

Advantages:

* lower latency
* better geographic replication

Disadvantage:

A failure can occur before all WAL reaches the replica.

This means:

```text
Possible RPO > 0
```

PostgreSQL supports both synchronous and asynchronous approaches for standby architectures. ([PostgreSQL][4])

---

# 6. High Availability vs Read Scaling

One common architecture mistake is treating an HA standby and a read replica as the same thing.

They solve different problems.

### High-availability standby

Purpose:

```text
Primary failure
      |
      v
Standby promoted
      |
      v
Application continues
```

Main goal:

**availability**

---

### Read replica

Purpose:

```text
                    Primary
                  /    |     \
                 /     |      \
                v      v       v
             Reader  Reader  Reader
```

Main goal:

**read scaling**

The replica may also contribute to disaster recovery, but its primary architectural purpose is usually offloading reads.

---

# 7. Point-in-Time Recovery — PITR

PITR allows the database to recover to a particular moment.

For example:

```text
10:00  Database OK
10:10  Database OK
10:15  Developer accidentally deletes table
10:20  Problem discovered
```

Instead of restoring only last night's backup, PITR could restore close to:

```text
10:14:59
```

Native PostgreSQL PITR combines:

```text
Base Backup
     +
Archived WAL
     =
Point-in-Time Recovery
```

PostgreSQL's current documentation explicitly describes continuous WAL archiving and PITR, and notes that recovery requires the necessary archived WAL sequence going back to the backup. ([PostgreSQL][5])

---

# 8. PostgreSQL Managed Services

The three major cloud platforms offer managed PostgreSQL.

```text
                     Managed PostgreSQL

          +----------------+----------------+
          |                |                |
          v                v                v
         AWS             Azure             GCP
          |                |                |
    RDS PostgreSQL    Azure Database    Cloud SQL
          |
    Aurora PostgreSQL
```

---

# 9. AWS PostgreSQL Architecture

AWS has two major PostgreSQL choices:

```text
AWS
 |
 +-- Amazon RDS for PostgreSQL
 |
 +-- Amazon Aurora PostgreSQL-Compatible
```

These should not be treated as identical services.

---

# 10. Amazon RDS for PostgreSQL

Amazon RDS PostgreSQL is managed PostgreSQL where AWS manages much of the underlying infrastructure.

AWS manages areas such as:

* infrastructure provisioning
* storage
* backups
* replacement of failed infrastructure
* monitoring integration
* patching workflows
* Multi-AZ failover mechanisms

You still manage:

* schemas
* tables
* indexes
* SQL
* users/roles
* database design
* queries
* performance tuning
* application connection management
* supported PostgreSQL parameters
* security configuration at the database/application level

---

# 11. Basic RDS PostgreSQL Architecture

```text
             Application
                  |
                  v
           RDS Endpoint
                  |
                  v
        RDS PostgreSQL Instance
                  |
                  v
          Managed Storage
```

Usually the database should run inside private networking rather than being directly exposed to the public internet.

A typical architecture is:

```text
Internet
   |
Load Balancer
   |
Application Servers
   |
Private Network
   |
RDS PostgreSQL
```

---

# 12. RDS Multi-AZ Architecture

Traditional Multi-AZ architecture:

```text
Availability Zone A        Availability Zone B

   Primary DB  -----------> Standby DB
        |
        |
    Application
```

The standby exists primarily for high availability.

AWS documentation distinguishes the HA standby from a normal read replica: the Multi-AZ standby uses synchronous replication and, in the traditional DB-instance design, isn't used to serve normal read traffic. ([AWS Documentation][6])

If failure occurs:

```text
Primary fails
     |
     v
AWS initiates failover
     |
     v
Standby becomes primary
```

Applications normally continue using the database DNS endpoint instead of manually switching IP addresses.

---

# 13. RDS Multi-AZ DB Cluster

AWS also provides a newer Multi-AZ DB cluster architecture.

Conceptually:

```text
                 Application
                     |
                     v
                   Writer
                     |
        +------------+------------+
        |                         |
        v                         v
     Reader 1                  Reader 2
       AZ-B                      AZ-C
```

AWS currently describes a Multi-AZ DB cluster as:

* one writer
* two readable instances
* spread across three Availability Zones
* semisynchronous HA replication

This architecture provides both higher availability and additional read capacity. ([AWS Documentation][7])

---

# 14. RDS Read Replicas

Read replicas are normally asynchronous.

```text
                   RDS Primary
                       |
             +---------+----------+
             |                    |
             v                    v
       Read Replica 1       Read Replica 2
```

Use cases:

* analytics
* reports
* dashboards
* read-heavy APIs
* separating reporting queries from transactional workload

AWS supports both same-Region and cross-Region PostgreSQL read replicas. ([AWS Documentation][8])

Important:

```text
Replica = eventually consistent
```

Replication lag can exist.

Therefore, don't send queries that require immediate read-after-write consistency to an asynchronous replica without designing for the lag.

---

# 15. AWS Cross-Region DR

Example:

```text
AWS Region A
-------------------------------
Primary RDS PostgreSQL
        |
        | asynchronous
        |
AWS Region B
-------------------------------
Cross-Region Read Replica
```

If Region A becomes unavailable:

```text
Cross-region replica
        |
        v
Promote replica
        |
        v
Redirect applications
```

Promotion turns a read replica into an independent writable database.

AWS documents cross-Region read replicas and promotion as separate operations rather than the same automatic Multi-AZ failover mechanism. ([AWS Documentation][9])

---

# 16. Amazon Aurora PostgreSQL

Aurora PostgreSQL is different from RDS PostgreSQL.

It is:

> an AWS-designed distributed relational database architecture that is PostgreSQL-compatible.

AWS describes Aurora as a fully managed relational engine compatible with PostgreSQL and MySQL. ([AWS Documentation][10])

The major architectural difference is storage.

---

# 17. Traditional RDS vs Aurora Storage

Simplified RDS model:

```text
PostgreSQL Instance
       |
       v
Instance Storage
```

Aurora separates compute more strongly from distributed shared cluster storage:

```text
        Writer
          |
          |
   +------+------+
   |             |
 Reader        Reader
   |             |
   +------+------+
          |
          v
   Aurora Cluster Storage
```

Multiple Aurora database instances access the same logical cluster storage.

AWS describes Aurora storage as a **distributed shared storage architecture**. ([AWS Documentation][11])

---

# 18. Aurora Distributed Storage

Aurora copies cluster data across multiple Availability Zones.

Conceptually:

```text
AZ-A             AZ-B             AZ-C

Storage copies   Storage copies   Storage copies
      \              |              /
       \             |             /
        +------ Aurora Volume ----+
```

AWS documentation states that Aurora synchronously replicates data across **six storage nodes across Availability Zones** for the cluster volume. ([AWS Documentation][12])

This is an important interview difference:

> RDS PostgreSQL primarily manages PostgreSQL using RDS infrastructure patterns, while Aurora PostgreSQL changes the underlying storage architecture significantly while maintaining PostgreSQL compatibility.

---

# 19. Aurora Writer and Readers

```text
             Application Writes
                    |
                    v
              Aurora Writer
                    |
               Shared Storage
              /      |       \
             /       |        \
            v        v         v
        Reader 1  Reader 2  Reader 3
```

Read replicas do not each need an entirely independent storage copy in the same way traditional PostgreSQL replication architectures do.

---

# 20. Aurora Failover

If the writer fails:

```text
Writer
  X
  |
  v
Aurora Replica
  |
Promoted
  |
  v
New Writer
```

For production, having Aurora replicas available reduces dependency on creating a replacement database compute node during the outage.

Aurora exposes cluster endpoint concepts so the application does not need to hard-code individual instance addresses.

---

# 21. Aurora Use Cases

Aurora PostgreSQL is attractive when you need:

* strong AWS integration
* managed high availability
* multiple readers
* fast storage growth
* reduced infrastructure administration
* high read scalability
* cross-region database architectures
* Aurora-specific capabilities

But always check PostgreSQL compatibility requirements.

Because:

```text
Aurora PostgreSQL-Compatible
             !=
Self-managed PostgreSQL in every implementation detail
```

Engine versions, extensions, configuration options, upgrade timing, and operational capabilities can differ.

---

# 22. Azure Database for PostgreSQL

The current Azure managed PostgreSQL product is centered on:

**Azure Database for PostgreSQL — Flexible Server.**

Microsoft describes it as a fully managed PostgreSQL Database-as-a-Service offering. ([Microsoft Learn][13])

---

# 23. Basic Azure PostgreSQL Architecture

```text
Azure Application
      |
      v
Azure Database for PostgreSQL
 Flexible Server
      |
      v
Managed Storage
```

Typical enterprise architecture:

```text
Users
  |
Application Gateway / Load Balancer
  |
AKS / App Service / VM
  |
Private Network
  |
Azure Database for PostgreSQL
```

---

# 24. Azure High Availability Architecture

Azure PostgreSQL HA uses:

```text
        Primary
           |
           | synchronous replication
           |
        Standby
```

Microsoft states that HA provisions physically separate primary and standby replicas and synchronously commits data to the primary and standby. ([Microsoft Learn][14])

---

# 25. Azure Zone-Redundant HA

A production pattern is:

```text
Availability Zone 1        Availability Zone 2

       Primary  -----------> Standby
          |
          v
     Applications
```

If the primary infrastructure or zone fails:

```text
Primary unavailable
       |
       v
Standby promoted
       |
       v
Service reconnects
```

This protects against Availability Zone failure within the region.

Azure also supports same-zone HA depending on configuration and regional requirements. ([Microsoft Learn][15])

---

# 26. Azure Read Replicas

Read replicas provide horizontal read scaling.

```text
                     Primary
                       |
              asynchronous
           +-----------+-----------+
           |                       |
           v                       v
       Replica 1               Replica 2
```

Microsoft currently documents Azure PostgreSQL read replicas as using PostgreSQL native physical replication, normally streaming through replication slots, with asynchronous updates. ([Microsoft Learn][16])

Use them for:

```text
Primary
 |
 +---- Transactions
 |
 +---- Replica ---> Reporting
 |
 +---- Replica ---> Analytics
 |
 +---- Replica ---> Read API
```

---

# 27. Azure Cross-Region Replication

Azure supports read replicas in another geographical region.

```text
Azure Region A
   |
 PostgreSQL Primary
   |
   | asynchronous
   v
Azure Region B
   |
 Read Replica
```

Use cases include:

* disaster recovery
* geographic read scaling
* reducing read latency for remote applications

Microsoft specifically documents same-region and cross-geography read replicas for these scenarios. ([Microsoft Learn][17])

---

# 28. Azure Networking

For enterprise database architectures, avoid exposing the database directly to the internet where possible.

Azure provides private networking options including:

* VNet integration
* Private Link
* Private Endpoint
* Private DNS

Example:

```text
Azure VNet
|
+-- Application subnet
|       |
|       v
|   Application
|
+-- Private connectivity
        |
        v
Azure PostgreSQL
```

With VNet private access, Microsoft states that the server can use private IP connectivity without an internet-accessible public endpoint. ([Microsoft Learn][18])

Azure Private Link/private endpoints provide another private connectivity model over Microsoft's network. ([Microsoft Learn][19])

---

# 29. Azure Backups and PITR

Azure PostgreSQL manages:

* database file snapshots
* WAL/transaction-log backups
* backup retention
* PITR operations

Conceptually:

```text
Snapshots
    +
WAL backups
    |
    v
Point-in-Time Restore
```

Microsoft says Flexible Server backs up data files and transaction logs and allows restoration to a point within the configured retention period. ([Microsoft Learn][20])

An important architecture point:

A PITR generally creates a restored server rather than overwriting your live production server.

For example:

```text
Production Server
       |
       | PITR
       v
New Restored Server
```

Then applications can be validated before cutover.

---

# 30. Google Cloud SQL for PostgreSQL

Google Cloud's managed PostgreSQL offering is:

**Cloud SQL for PostgreSQL**

Google provides management around:

* provisioning
* backups
* patching
* replication
* high availability
* monitoring
* storage
* failover

The official Cloud SQL PostgreSQL documentation covers HA, backups, read replicas, PITR and database administration. ([Google Cloud Documentation][21])

---

# 31. Basic Cloud SQL Architecture

```text
Application
    |
    v
Cloud SQL for PostgreSQL
    |
    v
Managed Storage
```

A preferred private design might be:

```text
Users
 |
Load Balancer
 |
GKE / Compute Engine / Cloud Run
 |
VPC
 |
Private connectivity
 |
Cloud SQL PostgreSQL
```

---

# 32. Cloud SQL High Availability

Simplified architecture:

```text
Zone A                    Zone B

Primary ---------------- Standby
   |
   v
Application
```

The standby protects the service from zonal infrastructure failures.

If the primary becomes unavailable:

```text
Failure
   |
   v
Standby takes over
   |
   v
Database service resumes
```

Google provides a dedicated HA configuration for Cloud SQL PostgreSQL. ([Google Cloud Documentation][22])

---

# 33. Cloud SQL Read Replicas

```text
                      Primary
                         |
             +-----------+----------+
             |                      |
             v                      v
         Replica A              Replica B
```

Typical workload separation:

```text
Primary ---> OLTP
Replica ---> Reporting
Replica ---> Analytics
Replica ---> Read APIs
```

Again:

> read replica ≠ HA standby.

HA solves availability.

Read replicas primarily solve read scaling and can also participate in DR architectures.

---

# 34. Cloud SQL Cross-Region DR

Google documents disaster recovery designs using cross-region read replicas.

```text
Region A
---------
Primary
   |
   | replication
   v

Region B
---------
Read Replica
```

During a regional disaster, the replica can form part of the promotion/failover process.

Google's Cloud SQL DR documentation explicitly describes cross-region read replicas as part of a disaster-recovery strategy. ([Google Cloud Documentation][23])

---

# 35. Cloud SQL Backups

Cloud SQL supports:

```text
Automated backups
On-demand backups
PITR
```

Google states that Cloud SQL backups can be scheduled or created on demand and are incremental. ([Google Cloud Documentation][24])

---

# 36. Cloud SQL PITR

PITR uses PostgreSQL WAL underneath.

Google specifically notes that PostgreSQL write-ahead logging provides the transaction logs required for Cloud SQL PITR. ([Google Cloud Documentation][25])

Example:

```text
12:00 Backup
 |
 | WAL
 |
 | WAL
 |
13:27 bad transaction
 |
Restore to
13:26:59
```

Cloud SQL PITR creates a **new instance**, rather than restoring into the existing instance. ([Google Cloud Documentation][26])

---

# 37. AWS vs Azure vs GCP — Conceptual Comparison

| Area                                   | AWS                                                                         | Azure                                         | Google Cloud                   |
| -------------------------------------- | --------------------------------------------------------------------------- | --------------------------------------------- | ------------------------------ |
| Standard managed PostgreSQL            | RDS PostgreSQL                                                              | Azure Database for PostgreSQL Flexible Server | Cloud SQL PostgreSQL           |
| Cloud-specific PostgreSQL architecture | Aurora PostgreSQL                                                           | Flexible Server                               | Cloud SQL                      |
| HA                                     | Multi-AZ                                                                    | Same-zone / Zone-redundant HA                 | Regional HA                    |
| Read scaling                           | Read replicas                                                               | Read replicas                                 | Read replicas                  |
| Cross-region DR                        | Cross-region replicas / backup replication / Aurora Global Database options | Geo-replicas / geo-redundant backup options   | Cross-region read replica / DR |
| PITR                                   | Yes                                                                         | Yes                                           | Yes                            |
| Automated backups                      | Yes                                                                         | Yes                                           | Yes                            |
| Private networking                     | VPC                                                                         | VNet / Private Link                           | VPC/private connectivity       |
| Infrastructure patching                | Managed                                                                     | Managed                                       | Managed                        |
| OS access                              | No                                                                          | No                                            | No                             |
| Full PostgreSQL superuser              | Restricted                                                                  | Restricted                                    | Restricted                     |
| Infrastructure control                 | Limited                                                                     | Limited                                       | Limited                        |

Exact feature availability, PostgreSQL versions, extensions and regional support can change, so check the provider's current service documentation before making a production architecture decision.

---

# 38. Managed vs Self-Managed PostgreSQL

This is one of the most important architecture discussions.

## Managed architecture

```text
Application
    |
    v
Managed PostgreSQL
    |
Cloud provider manages infrastructure
```

You primarily manage:

```text
Data model
Schemas
Queries
Indexes
Roles
Database configuration
Performance
Application connections
```

Provider manages much of:

```text
Servers
Storage
Hardware
Operating system
Database provisioning
Backups
HA infrastructure
Patching workflow
```

---

# 39. Self-Managed Architecture

Example on AWS:

```text
             Application
                  |
                  v
            Load / DB Proxy
                  |
        +---------+---------+
        |                   |
        v                   v
EC2 PostgreSQL Primary   EC2 Standby
        |                   |
        +---- WAL ----------+
```

You are responsible for:

```text
PostgreSQL installation
Linux
Storage
Replication
Failover
Backups
Patching
Monitoring
Certificates
Security
Upgrades
PostgreSQL configuration
HA software
DR
```

---

# 40. Managed vs Self-Managed Responsibility

| Task                     | Managed PostgreSQL     | Self-Managed                  |
| ------------------------ | ---------------------- | ----------------------------- |
| Hardware                 | Provider               | Customer/provider VM platform |
| OS installation          | Provider               | Customer                      |
| PostgreSQL installation  | Provider               | Customer                      |
| OS patching              | Provider               | Customer                      |
| PostgreSQL patching      | Mostly provider-driven | Customer                      |
| Backup infrastructure    | Provider               | Customer                      |
| HA infrastructure        | Provider               | Customer                      |
| Failover tooling         | Provider               | Customer                      |
| PostgreSQL schema        | Customer               | Customer                      |
| SQL tuning               | Customer               | Customer                      |
| Index tuning             | Customer               | Customer                      |
| Data lifecycle           | Customer               | Customer                      |
| Application architecture | Customer               | Customer                      |
| Security design          | Shared                 | Customer-heavy                |
| Extension flexibility    | Restricted             | High                          |
| OS access                | No                     | Yes                           |
| Filesystem access        | No                     | Yes                           |

---

# 41. Why Choose Managed PostgreSQL?

Advantages include:

### Less operational work

Instead of maintaining:

```text
Linux
PostgreSQL packages
Storage
Backups
Replication
Failover
```

the database team can focus more on:

```text
SQL
Schema design
Performance
Security
Data architecture
```

### Easier HA

Provision:

```text
Primary + Standby
```

through cloud configuration rather than manually building:

```text
Streaming replication
Failover automation
Virtual IP/DNS switching
Health checks
Fencing
Rebuild procedures
```

### Automated backup services

Managed environments normally integrate:

```text
Snapshots
+
Transaction logs
+
Retention
+
PITR
```

### Monitoring integration

For example:

```text
AWS -> CloudWatch
Azure -> Azure Monitor
GCP -> Cloud Monitoring
```

---

# 42. Managed PostgreSQL Limitations

Managed doesn't mean unlimited PostgreSQL access.

Common restrictions include:

* no root OS access
* restricted filesystem access
* restricted true superuser permissions
* only supported extensions
* provider-controlled patching
* provider-controlled engine builds
* limited low-level replication configuration
* limited access to PostgreSQL binaries
* restrictions on certain parameters

Therefore:

```text
Managed PostgreSQL
       =
PostgreSQL capability
       +
Cloud provider guardrails
```

---

# 43. Why Choose Self-Managed PostgreSQL?

Self-managed PostgreSQL makes sense when you require:

### Full operating-system control

For example:

```text
custom kernel tuning
custom storage configuration
custom filesystem
special monitoring agents
```

### Full PostgreSQL superuser capability

### Unsupported extensions

Example architectural requirement:

```text
Application
   |
PostgreSQL
   |
Special native extension
   |
Custom operating system library
```

A managed platform may not permit it.

### Custom HA architecture

For example:

```text
Patroni
+
etcd/Consul
+
HAProxy
+
PostgreSQL
```

### Specialized backup systems

Such as:

```text
pgBackRest
Barman
custom WAL archive
custom object-storage pipelines
```

---

# 44. Self-Managed HA Example

A mature self-managed architecture could look like:

```text
                    Applications
                         |
                         v
                     HAProxy
                         |
               +---------+---------+
               |                   |
               v                   v
           Primary              Standby
         PostgreSQL            PostgreSQL
               |                   ^
               +------ WAL --------+
               |
               v
          WAL Archive
               |
               v
          Object Storage
```

A coordination layer might determine who should be primary.

For example:

```text
        Patroni
       /   |   \
      /    |    \
Primary  Standby  Standby
```

This gives enormous control but also increases operational responsibility.

---

# 45. Backup Is Not High Availability

A critical architecture principle:

```text
Backup != HA
```

If the production server fails and the only recovery mechanism is:

```text
restore backup
```

your RTO might be too long.

HA:

```text
Primary fails
   |
Standby takes over
```

Backup:

```text
Data lost/corrupted
   |
Restore historical copy
```

You normally need **both**.

---

# 46. Replication Is Not Backup

Another important principle:

```text
Replication != Backup
```

Suppose:

```sql
DROP TABLE customers;
```

is executed.

Replication faithfully reproduces it:

```text
Primary:
DROP customers

      |
      v

Replica:
DROP customers
```

The replica doesn't automatically protect against logical corruption.

A backup/PITR system might.

---

# 47. HA vs DR

Also distinguish:

```text
HA = High Availability
DR = Disaster Recovery
```

HA normally protects against:

```text
server failure
storage failure
zone failure
instance failure
```

DR protects against larger events:

```text
regional outage
major corruption
security incident
operator mistake
catastrophic infrastructure event
```

Example:

```text
             Region A
          Primary + Standby
                 |
                 | async
                 v
             Region B
             DR Replica
```

---

# 48. RPO and RTO

You should know these for architecture interviews.

## RPO — Recovery Point Objective

How much data can the business afford to lose?

Example:

```text
RPO = 5 minutes
```

means losing the latest five minutes of data might be accepted.

---

## RTO — Recovery Time Objective

How long can the system be unavailable?

Example:

```text
RTO = 10 minutes
```

means service should recover within approximately ten minutes.

Architecture is then designed backwards from these requirements.

Example:

```text
RPO near zero
RTO minutes
```

may suggest:

```text
Synchronous multi-zone HA
+
PITR
+
Cross-region DR
```

rather than simply taking a nightly backup.

---

# 49. PostgreSQL Cloud Production Architecture

A good general architecture is:

```text
                      Internet
                          |
                          v
                   Load Balancer
                          |
                +---------+---------+
                |                   |
                v                   v
           App Server          App Server
                \                   /
                 \                 /
                  +-------+-------+
                          |
                          v
                    DB Connection Pool
                          |
                          v
                  PostgreSQL Primary
                          |
                +---------+---------+
                |                   |
                v                   v
            HA Standby         Read Replica
                                    |
                                    v
                                Reporting

                          |
                          v
                      Backups/PITR

                          |
                          v
                 Cross-Region DR
```

---

# 50. Private Network Architecture

Production databases should generally use private connectivity.

```text
Public Internet
      |
Load Balancer
      |
Application subnet
      |
Private network
      |
PostgreSQL
```

Avoid:

```text
Internet
   |
   v
PostgreSQL Public IP
```

unless there is a clear and controlled requirement.

---

# 51. Database Connection Pooling

Cloud PostgreSQL is still PostgreSQL, and PostgreSQL connections consume resources.

Poor architecture:

```text
10,000 application requests
         |
         v
10,000 database connections
```

Better:

```text
10,000 requests
       |
       v
Connection Pool
       |
       v
200 PostgreSQL connections
```

Possible approaches include:

```text
PgBouncer
Application connection pool
AWS RDS Proxy where appropriate
Provider-specific connection approaches
```

Connection management becomes especially important with:

* Kubernetes
* serverless applications
* microservices
* many application instances

---

# 52. Vertical Scaling

Vertical scaling means increasing database server resources.

```text
Before:

4 CPU
16 GB RAM

After:

16 CPU
64 GB RAM
```

Advantages:

* simple
* minimal application changes

Limit:

Eventually one server reaches practical or economic limits.

---

# 53. Horizontal Read Scaling

Use replicas:

```text
                 Primary
                    |
        +-----------+-----------+
        |           |           |
        v           v           v
      Read 1      Read 2      Read 3
```

Applications can route:

```text
INSERT / UPDATE / DELETE -> Primary

SELECT reporting queries -> Replicas
```

But account for replication lag.

---

# 54. Write Scaling

PostgreSQL is fundamentally built around a writable primary in traditional physical replication architecture.

Read scaling is comparatively easy.

Write scaling is harder.

Options might include:

* application-level sharding
* partition-aware architectures
* distributed PostgreSQL products
* redesigning services/data ownership
* separating workloads into multiple databases

Do not assume adding replicas increases primary write throughput.

---

# 55. Logical Replication

Physical replication copies the PostgreSQL storage/WAL changes at the server level.

Logical replication works at a logical data-change level.

Conceptually:

```text
Publisher
   |
   | INSERT / UPDATE / DELETE changes
   |
   v
Subscriber
```

Logical replication can be useful for:

* migrations
* selective table replication
* major-version transitions
* data distribution
* integration architectures

It is different from standard physical HA replication.

---

# 56. Physical vs Logical Replication

| Feature                  | Physical                           | Logical                |
| ------------------------ | ---------------------------------- | ---------------------- |
| Replication level        | PostgreSQL cluster/storage changes | Logical table changes  |
| WAL based                | Yes                                | Yes                    |
| Select tables            | Generally no                       | Yes                    |
| Different major versions | Limited                            | More flexible          |
| HA standby               | Common                             | Not usual primary use  |
| Migration                | Possible                           | Very useful            |
| Read replicas            | Common                             | Possible architecture  |
| Schema handling          | Physical cluster copy              | Requires more planning |

---

# 57. Multi-Region PostgreSQL Architecture

Example:

```text
                 Global Users
                      |
              Global Load Balancer
                 /             \
                /               \
             Region A          Region B
                |                 |
                v                 v
            App Tier           App Tier
                |
                v
         PostgreSQL Primary
                |
                | async
                v
          DR Read Replica
             Region B
```

The main question becomes:

> What happens when Region A disappears?

You need a runbook covering:

```text
detect failure
promote DR replica
change routing
validate data
restart services
protect old primary
eventually fail back
```

---

# 58. Multi-Cloud PostgreSQL

Some organizations ask for:

```text
AWS
 |
PostgreSQL
 |
Azure/GCP DR
```

This is significantly more difficult than cross-region DR inside one cloud.

Challenges include:

```text
WAN latency
network cost
security
DNS
identity
replication lag
backup portability
cloud-specific PostgreSQL differences
failover automation
operational ownership
```

Self-managed PostgreSQL or logical replication may provide more portability, but at much higher operational cost.

---

# 59. Availability Zone vs Region

Remember:

```text
Region
 |
 +---- AZ-1
 |
 +---- AZ-2
 |
 +---- AZ-3
```

Multi-AZ protects primarily against localized/zonal failures.

It does not automatically mean the architecture survives complete region loss.

For regional disaster recovery:

```text
Region A
   |
   +---- Primary
   +---- HA Standby

        |
        | cross-region
        v

Region B
   |
   +---- DR copy / replica
```

---

# 60. PostgreSQL Cloud Security Architecture

Think in layers.

```text
Layer 1: Identity
Layer 2: Network
Layer 3: TLS
Layer 4: PostgreSQL roles
Layer 5: Database privileges
Layer 6: Row-level security where needed
Layer 7: Encryption at rest
Layer 8: Audit and monitoring
Layer 9: Backup security
```

---

# 61. Network Security

Preferred design:

```text
Application
    |
Private subnet
    |
Security controls
    |
PostgreSQL private endpoint
```

Use:

* VPC/VNet firewall controls
* security groups/firewall rules
* private DNS
* private service connectivity
* TLS

Avoid unnecessarily exposing port:

```text
5432
```

to the public internet.

---

# 62. PostgreSQL Authentication

Cloud services may combine PostgreSQL authentication with provider identity systems.

Possible approaches include:

```text
PostgreSQL username/password
Cloud IAM integration
Microsoft Entra authentication
TLS certificates
Secrets manager integration
```

The exact capabilities differ by service.

Avoid hard-coded database passwords such as:

```text
db_password = "ProductionPassword123"
```

Prefer:

```text
Secrets Manager
Key Vault
Secret Manager
Workload identity/IAM mechanisms
```

---

# 63. Encryption

Think about both:

```text
Encryption in transit
Encryption at rest
```

In transit:

```text
Application
   |
 TLS
   |
PostgreSQL
```

At rest:

```text
Database storage
Snapshots
Backups
Replica storage
```

Enterprise environments may require customer-managed keys depending on regulatory requirements.

---

# 64. Monitoring Architecture

Monitor at four levels.

### Infrastructure

```text
CPU
Memory
Storage
IOPS
Network
```

### PostgreSQL

```text
Connections
Transactions
Locks
Deadlocks
Cache activity
Replication lag
Checkpoints
WAL generation
Autovacuum
```

### Query

```text
Query duration
Execution count
Rows
I/O
Slow queries
Execution plans
```

### Business/application

```text
Checkout latency
API response time
Order throughput
Error rate
```

PostgreSQL's current documentation includes the cumulative statistics system and PostgreSQL runtime monitoring views used for this type of analysis. ([PostgreSQL][27])

---

# 65. Important PostgreSQL Metrics

Common metrics:

```text
CPU utilization
Memory pressure
Storage usage
Disk latency
IOPS
Database connections
Active connections
Idle connections
Transactions/sec
Replication lag
WAL volume
Deadlocks
Lock waits
Cache hit ratio
Checkpoint behavior
Long-running transactions
Autovacuum progress
Table/index bloat indicators
```

Do not rely only on CPU.

For example:

```text
CPU = 30%
```

does not mean the database is healthy if storage latency is extremely high.

---

# 66. PostgreSQL Backup Strategy

A mature backup architecture considers:

```text
Automated backups
PITR
Manual snapshots
Long-term retention
Cross-region backup
Restore testing
Backup encryption
Backup access control
```

Most important:

> A backup is only useful if restoration works.

Therefore perform:

```text
Backup
   |
Restore test
   |
Integrity validation
   |
Application validation
```

---

# 67. PostgreSQL Native Backup Knowledge

For self-managed PostgreSQL, know:

```text
pg_dump
pg_dumpall
pg_restore
pg_basebackup
WAL archiving
PITR
```

Logical:

```text
pg_dump
```

Physical:

```text
pg_basebackup
+
WAL
```

PostgreSQL 18 documentation confirms `pg_basebackup` can take physical base backups from a primary and, under appropriate configuration, from a standby. ([PostgreSQL][28])

---

# 68. Managed Service Upgrade Strategy

Managed services simplify upgrades but do not remove upgrade planning.

Consider:

```text
Application compatibility
Driver compatibility
Extension compatibility
Parameter changes
Removed features
Execution-plan changes
Downtime
Replication
Rollback strategy
```

A good pattern:

```text
Upgrade non-production
        |
Run integration tests
        |
Run performance tests
        |
Validate extensions
        |
Schedule production upgrade
```

---

# 69. Extension Compatibility

This is especially important when moving from self-managed PostgreSQL to managed services.

Self-managed:

```text
Install almost any compatible extension
```

Managed:

```text
Install only provider-supported extensions
```

Before migration, inventory:

```sql
SELECT extname, extversion
FROM pg_extension;
```

Then check every extension against the target provider.

For example, Google Cloud maintains a specific list of PostgreSQL extensions supported by Cloud SQL rather than offering unrestricted extension installation. ([Google Cloud Documentation][29])

---

# 70. Migration Architecture

Typical migration:

```text
On-Prem PostgreSQL
       |
       | initial load
       v
Cloud PostgreSQL
       |
       | CDC / replication
       v
Catch up
       |
       v
Cutover
```

Possible technologies include:

```text
pg_dump / pg_restore
Logical replication
Cloud migration services
CDC-based tools
Physical migration methods where supported
```

Choice depends on:

```text
Database size
Allowed downtime
PostgreSQL versions
Extensions
Network bandwidth
RPO/RTO
```

---

# 71. Small Database Migration

For a smaller database:

```text
Stop application
      |
pg_dump
      |
pg_restore
      |
Test
      |
Change connection string
      |
Start application
```

Simple but requires downtime.

---

# 72. Low-Downtime Migration

For large production systems:

```text
Source PostgreSQL
      |
      | Initial copy
      v
Target PostgreSQL
      |
      ^
      |
Continuous replication
```

At cutover:

```text
Stop writes
   |
Wait for replication lag = near 0
   |
Switch connection
   |
Start application
```

---

# 73. Cost Architecture

Cloud database cost is not simply:

```text
CPU + RAM
```

Consider:

```text
Database compute
Storage
Provisioned IOPS
Backup storage
Replica compute
Cross-region replication
Data transfer
Monitoring
Proxy services
HA standby
DR infrastructure
Licensing of third-party tooling
```

For example:

```text
1 primary
+
1 HA standby
+
3 read replicas
+
1 DR replica
```

is effectively a multi-node architecture, not "one PostgreSQL server."

---

# 74. Managed Service Selection

A simple decision framework:

### Choose RDS PostgreSQL when:

```text
Need standard PostgreSQL behavior
+
AWS ecosystem
+
Managed operations
```

### Choose Aurora PostgreSQL when:

```text
AWS-native architecture
+
High availability
+
Read scaling
+
Aurora-specific distributed storage benefits
```

### Choose Azure PostgreSQL when:

```text
Azure ecosystem
+
Azure networking/identity
+
Managed PostgreSQL
```

### Choose Cloud SQL PostgreSQL when:

```text
Google Cloud ecosystem
+
Managed PostgreSQL
+
GCP networking/integration
```

### Choose self-managed when:

```text
Need low-level PostgreSQL/OS control
+
Operational team can manage complexity
```

---

# 75. Architecture Decision Example

Requirement:

```text
24x7 payment platform
RPO near zero
Low RTO
AWS hosted
Read-heavy reporting
Regional disaster recovery
```

Possible architecture:

```text
                    Applications
                         |
                         v
                  Connection Pool
                         |
                         v
                Aurora PostgreSQL
                    Writer
                       |
             +---------+---------+
             |                   |
             v                   v
          Reader 1            Reader 2
        Transaction           Reporting
          Reads

              Aurora HA Storage
                    across AZs

                       |
                       v
                Cross-Region DR
```

Plus:

```text
Backups
PITR
Private VPC
TLS
Secrets Manager
Monitoring
DR testing
```

---

# 76. Architecture Anti-Patterns

Avoid these.

### Single database without HA

```text
Application -> One PostgreSQL VM
```

Failure means outage.

---

### Database exposed publicly

```text
Internet -> 5432 -> PostgreSQL
```

Avoid unless explicitly required and tightly secured.

---

### No PITR

Nightly backup only:

```text
00:00 backup
17:00 corruption
```

Potential:

```text
17 hours data loss
```

---

### Replica treated as backup

Replication copies mistakes.

---

### No replication-lag monitoring

Read replicas can fall significantly behind.

---

### Unlimited application connections

```text
500 application containers
x
100 connections
=
50,000 potential database connections
```

Use pooling and sensible limits.

---

### No restore testing

Having snapshots listed in a console doesn't prove the complete application can recover.

---

# 77. Interview Question — RDS PostgreSQL vs Aurora PostgreSQL

Good answer:

> RDS PostgreSQL provides managed PostgreSQL using Amazon RDS infrastructure. Aurora PostgreSQL is PostgreSQL-compatible but uses an AWS-designed distributed shared-storage architecture. Aurora separates compute and cluster storage more deeply, maintains redundant storage across Availability Zones, and allows multiple database instances to use the cluster volume. The choice depends on PostgreSQL compatibility requirements, performance, availability architecture, cost and AWS-specific requirements. ([AWS Documentation][10])

---

# 78. Interview Question — Multi-AZ vs Read Replica

> Multi-AZ is mainly a high-availability mechanism. A standby or HA replica exists to take over when the primary fails. A read replica is primarily used to scale read traffic and normally uses asynchronous replication, which means replication lag is possible.

AWS explicitly documents this distinction for RDS. ([AWS Documentation][6])

---

# 79. Interview Question — How Would You Design PostgreSQL for Production?

A strong answer:

```text
1. Deploy database privately.
2. Use multi-zone HA.
3. Enable automated backups and PITR.
4. Define RPO and RTO.
5. Add read replicas if read scaling is required.
6. Add cross-region DR if regional failure must be covered.
7. Use connection pooling.
8. Encrypt traffic and storage.
9. Store credentials in a secrets system.
10. Monitor PostgreSQL, infrastructure and application metrics.
11. Test backup restoration.
12. Test failover and DR regularly.
```

---

# 80. Interview Question — Managed or Self-Managed?

Answer:

> I would normally start with managed PostgreSQL because it reduces infrastructure and operational work around provisioning, backups, patching, high availability and hardware failure. I would choose self-managed PostgreSQL when the workload requires OS-level control, unrestricted PostgreSQL extensions, custom HA/backup tooling or capabilities that the managed service doesn't provide.

---

# 81. Interview Question — How Do You Protect PostgreSQL From a Region Failure?

Architecture:

```text
Region A
--------------
Primary
+
HA Standby

       |
       | asynchronous
       v

Region B
--------------
DR Replica
```

Additionally:

```text
Cross-region backups
DNS/routing strategy
Promotion process
Application failover
Secrets/network configuration
Runbook
Regular DR test
```

Do not assume Multi-AZ alone covers a whole regional outage.

---

# 82. Interview Question — Why Is PITR Important?

Example:

```text
2:00 PM database healthy
2:05 PM accidental DELETE
2:10 PM issue discovered
```

A normal snapshot from midnight could lose fourteen hours of data.

PITR can potentially restore much closer to:

```text
1:59:59 / 2:04:59
```

depending on the target event and recovery window.

Native PostgreSQL implements this through base backups plus WAL archiving. ([PostgreSQL][5])

---

# 83. Interview Question — Can Read Replicas Guarantee Fresh Reads?

No.

With asynchronous replication:

```text
Primary commits transaction
      |
      |
Application queries replica immediately
      |
Replica might not have transaction yet
```

This is known as replication lag.

Solutions depend on application design:

```text
Read critical data from primary
Use session/read-after-write routing
Measure replica lag
Accept eventual consistency where appropriate
```

---

# 84. Interview Question — What Happens During PostgreSQL Failover?

Conceptually:

```text
Primary failure detected
        |
        v
Standby selected
        |
        v
Standby promoted
        |
        v
Connection endpoint/routing updated
        |
        v
Clients reconnect
```

For native self-managed PostgreSQL, rebuilding the redundant standby state after failover is also an operational task. PostgreSQL's documentation notes that after failover a new standby must be established to return the cluster to a redundant state. ([PostgreSQL][30])

---

# 85. Quick Memory Diagram

Remember PostgreSQL cloud architecture like this:

```text
                    PostgreSQL Cloud
                          |
          +---------------+---------------+
          |                               |
       Managed                       Self-Managed
          |                               |
   +------+------+                  VM/Kubernetes
   |      |      |                       |
  AWS   Azure   GCP                 Full Control
   |
   +-- RDS PostgreSQL
   |
   +-- Aurora PostgreSQL
```

And every production design should think about:

```text
             PostgreSQL
                 |
    +------------+------------+
    |            |            |
    v            v            v
    HA         Backup         DR
    |            |            |
 Multi-AZ       PITR       Cross-Region
    |
    +------------+
    |
Read Replicas
    |
Read Scaling
```

---

# 86. Most Important Points to Remember

1. **PostgreSQL 18 is the current production documentation line; 18.4 is the current patch release shown by PostgreSQL as of August 19, 2026.** ([PostgreSQL][1])
2. WAL is the foundation of PostgreSQL physical replication and PITR. ([PostgreSQL][2])
3. HA standby and read replica solve different problems.
4. Synchronous replication favors lower potential data loss; asynchronous replication favors lower latency and geographic flexibility.
5. RDS PostgreSQL and Aurora PostgreSQL are not the same architecture.
6. Aurora uses distributed shared cluster storage. ([AWS Documentation][11])
7. Azure PostgreSQL Flexible Server provides managed HA with primary/standby configurations. ([Microsoft Learn][14])
8. Cloud SQL provides HA, replicas, automated backups and PITR. ([Google Cloud Documentation][22])
9. Multi-AZ protects primarily against localized infrastructure/AZ failures; cross-region architecture is needed when regional DR is required.
10. Replication is **not** a replacement for backup.
11. Backup is **not** a replacement for HA.
12. Define RPO and RTO before choosing the architecture.
13. Use private networking for production databases where practical.
14. Use connection pooling.
15. Test restores and failovers instead of assuming they work.
16. Managed services reduce operational work but restrict low-level PostgreSQL and OS control.
17. Self-managed PostgreSQL offers maximum flexibility but makes your team responsible for HA, backups, upgrades, patching and recovery.

## Official documentation verified

The primary PostgreSQL reference used here is the **PostgreSQL 18.4 current documentation**, including the official sections on High Availability, Streaming Replication, Hot Standby, WAL configuration, Backup and Restore, Continuous Archiving/PITR, failover and monitoring. ([PostgreSQL][1])

For AWS, the references are the official **Amazon RDS** and **Amazon Aurora** documentation covering Aurora PostgreSQL, Aurora distributed storage/HA, RDS Multi-AZ clusters, PostgreSQL read replicas, cross-Region replicas and RDS PITR/backups. ([AWS Documentation][31])

For Azure, the references are official **Microsoft Learn** documentation for Azure Database for PostgreSQL Flexible Server, including HA, read replicas, networking, backups/PITR and geo-replication. ([Microsoft Learn][13])

For Google Cloud, the references are official **Google Cloud SQL for PostgreSQL** documentation covering HA, backups, PITR and disaster recovery. ([Google Cloud Documentation][21])

[1]: https://www.postgresql.org/docs/current/index.html?utm_source=chatgpt.com "PostgreSQL 18.4 Documentation"
[2]: https://www.postgresql.org/docs/current/warm-standby.html?utm_source=chatgpt.com "Documentation: 18: 26.2. Log-Shipping Standby Servers"
[3]: https://www.postgresql.org/docs/current/high-availability.html?utm_source=chatgpt.com "18: Chapter 26. High Availability, Load Balancing, and ..."
[4]: https://www.postgresql.org/docs/current/different-replication-solutions.html?utm_source=chatgpt.com "Documentation: 18: 26.1. Comparison of Different Solutions"
[5]: https://www.postgresql.org/docs/current/continuous-archiving.html?utm_source=chatgpt.com "25.3. Continuous Archiving and Point-in-Time Recovery ..."
[6]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html?utm_source=chatgpt.com "Working with DB instance read replicas"
[7]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/multi-az-db-clusters-concepts.html?utm_source=chatgpt.com "Multi-AZ DB cluster deployments for Amazon RDS"
[8]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PostgreSQL.Replication.ReadReplicas.Configuration.html?utm_source=chatgpt.com "Read replica configuration with PostgreSQL"
[9]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.Promote.html?utm_source=chatgpt.com "Promoting a read replica to be a standalone DB instance"
[10]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraOverview.html?utm_source=chatgpt.com "What is Amazon Aurora? - Amazon Aurora"
[11]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.StorageReliability.html?utm_source=chatgpt.com "Amazon Aurora storage"
[12]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html?utm_source=chatgpt.com "High availability for Amazon Aurora"
[13]: https://learn.microsoft.com/en-us/azure/postgresql/?utm_source=chatgpt.com "Azure Database for PostgreSQL documentation landing page"
[14]: https://learn.microsoft.com/en-us/azure/postgresql/high-availability/concepts-high-availability?utm_source=chatgpt.com "High Availability in Azure Database for PostgreSQL flexible ..."
[15]: https://learn.microsoft.com/en-us/azure/postgresql/high-availability/how-to-configure-high-availability?utm_source=chatgpt.com "Configure High Availability - Azure Database for PostgreSQL"
[16]: https://learn.microsoft.com/en-us/azure/postgresql/read-replica/concepts-read-replicas?utm_source=chatgpt.com "Read replicas in Azure Database for PostgreSQL Flexible ..."
[17]: https://learn.microsoft.com/en-us/azure/postgresql/read-replica/concepts-read-replicas-geo?utm_source=chatgpt.com "Geo-replication in Azure Database for PostgreSQL Flexible ..."
[18]: https://learn.microsoft.com/en-us/azure/postgresql/network/concepts-networking-private?utm_source=chatgpt.com "Private access (virtual network integration)"
[19]: https://learn.microsoft.com/en-us/azure/postgresql/network/concepts-networking-private-link?utm_source=chatgpt.com "Networking with Private Link Azure Database for ..."
[20]: https://learn.microsoft.com/en-us/azure/postgresql/backup-restore/concepts-backup-restore?utm_source=chatgpt.com "Backup and Restore in Azure Database for PostgreSQL ..."
[21]: https://docs.cloud.google.com/sql/docs/postgres?utm_source=chatgpt.com "Cloud SQL for PostgreSQL documentation"
[22]: https://docs.cloud.google.com/sql/docs/postgres/high-availability?utm_source=chatgpt.com "About high availability | Cloud SQL for PostgreSQL"
[23]: https://docs.cloud.google.com/sql/docs/postgres/intro-to-cloud-sql-disaster-recovery?utm_source=chatgpt.com "About disaster recovery (DR) in Cloud SQL"
[24]: https://docs.cloud.google.com/sql/docs/postgres/backup-recovery/backups?utm_source=chatgpt.com "Cloud SQL backups overview | Cloud SQL for PostgreSQL"
[25]: https://docs.cloud.google.com/sql/docs/postgres/backup-recovery/configure-pitr?utm_source=chatgpt.com "Configure point-in-time recovery (PITR)"
[26]: https://docs.cloud.google.com/sql/docs/postgres/backup-recovery/pitr?utm_source=chatgpt.com "Perform point-in-time recovery (PITR) | Cloud SQL for ..."
[27]: https://www.postgresql.org/docs/current/monitoring-stats.html?utm_source=chatgpt.com "Documentation: 18: 27.2. The Cumulative Statistics System"
[28]: https://www.postgresql.org/docs/current/app-pgbasebackup.html?utm_source=chatgpt.com "Documentation: 18: pg_basebackup"
[29]: https://docs.cloud.google.com/sql/docs/postgres/extensions?utm_source=chatgpt.com "Configure PostgreSQL extensions | Cloud SQL for ..."
[30]: https://www.postgresql.org/docs/current/warm-standby-failover.html?utm_source=chatgpt.com "Documentation: 18: 26.3. Failover"
[31]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraPostgreSQL.html?utm_source=chatgpt.com "Working with Amazon Aurora PostgreSQL"
