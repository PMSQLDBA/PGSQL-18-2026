# Topic 5 — PostgreSQL 18 Replication

PostgreSQL replication is the process of keeping one or more PostgreSQL servers synchronized with another server. In PostgreSQL 18, the two major replication models are **physical replication** and **logical replication**. Physical replication works at the WAL/block level, while logical replication works at the table/change level. PostgreSQL also supports synchronous, asynchronous, cascading, hot standby, replication slots, and streaming replication. ([PostgreSQL][1])

## 1. High-Level Replication Architecture

A simple physical streaming replication setup looks like this:

```text
                    PRIMARY
                       │
                       │ WAL
                       ▼
                  WAL Sender
                       │
                       │ Streaming
                       ▼
                  WAL Receiver
                       │
                       ▼
                    STANDBY
                       │
                       ▼
                  WAL Replay
```

The primary accepts read/write traffic. The standby receives WAL generated on the primary and replays it to maintain a physical copy of the database cluster. PostgreSQL's HA documentation describes standby servers as systems that track changes from a primary, with hot standbys able to serve read-only queries. ([PostgreSQL][1])

---

## 2. Physical vs Logical Replication

This is the first distinction a DBA should understand.

| Physical Replication                  | Logical Replication                         |
| ------------------------------------- | ------------------------------------------- |
| Replicates WAL changes                | Replicates logical row changes              |
| Cluster-level                         | Database/table-oriented                     |
| Binary/physical copy                  | Publisher/subscriber model                  |
| Same PostgreSQL architecture required | More flexible object selection              |
| Common for HA/DR                      | Common for migration/integration            |
| Standby generally read-only           | Subscriber can have more flexible use cases |
| Uses WAL sender/receiver              | Uses logical decoding and apply workers     |

Logical replication in PostgreSQL uses a **publish/subscribe model**, where subscribers pull changes from publications on the publisher. ([PostgreSQL][2])

---

# 3. Physical Streaming Replication

Physical streaming replication sends WAL records continuously from the primary to the standby.

Conceptually:

```text
Application
    │
    ▼
 PRIMARY
    │
    │ UPDATE / INSERT / DELETE
    ▼
Generate WAL
    │
    ▼
WAL Sender
    │
    │ Network
    ▼
WAL Receiver
    │
    ▼
Write WAL
    │
    ▼
Replay WAL
    │
    ▼
 STANDBY
```

This is called **streaming replication** because the WAL is streamed directly between PostgreSQL servers rather than waiting only for archived WAL files. PostgreSQL's replication protocol includes dedicated replication connections and WAL streaming messages. ([PostgreSQL][3])

---

# 4. WAL Is the Foundation

Physical replication depends on PostgreSQL's Write-Ahead Log.

Whenever the primary changes data:

```text
Transaction
    │
    ▼
Generate WAL
    │
    ├── Used locally for recovery
    │
    ├── Can be archived
    │
    └── Can be streamed to standby
```

This makes WAL central to:

```text
Crash recovery
Physical replication
Point-in-time recovery
Backup recovery
Standby synchronization
```

We will go deeper into the WAL internals in **Topic 15 — WAL & Checkpoints**.

---

# 5. WAL Sender Process

On the primary, PostgreSQL uses a:

```text
walsender
```

process for replication connections.

Architecture:

```text
Primary PostgreSQL
       │
       ├── Client backend
       ├── Client backend
       │
       └── WAL Sender
                │
                ▼
             Standby
```

The replication protocol puts a backend into physical `walsender` mode when a replication connection is initiated. ([PostgreSQL][3])

---

# 6. WAL Receiver Process

On the standby, PostgreSQL uses:

```text
walreceiver
```

to receive streamed WAL.

Conceptually:

```text
Primary
   │
   │ WAL stream
   ▼
WAL Receiver
   │
   ▼
Standby WAL
   │
   ▼
Recovery / Replay
```

You can monitor the receiver using:

```sql
SELECT *
FROM pg_stat_wal_receiver;
```

This is particularly useful when troubleshooting standby connectivity or streaming issues.

---

# 7. Standby Startup and Recovery

A standby is essentially running PostgreSQL in continuous recovery mode.

It:

```text
Receives WAL
     │
     ▼
Writes/reads WAL
     │
     ▼
Replays changes
     │
     ▼
Keeps database synchronized
```

If the standby supports read-only sessions during recovery, it is called a **hot standby**. PostgreSQL's HA documentation distinguishes hot standby from a warm standby, which is not serving client queries until promotion. ([PostgreSQL][1])

---

# 8. Hot Standby

Hot standby allows users to execute read-only queries while the server is still recovering.

Architecture:

```text
                     PRIMARY
                        │
                        │ WAL
                        ▼
                     STANDBY
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
          WAL Replay         Read-only Queries
```

This enables architectures such as:

```text
Writes  → Primary

Reports → Standby
Reads   → Standby
```

However, the standby is still replaying WAL, so long-running read queries and recovery activity can interact with each other.

---

# 9. Asynchronous Replication

By default, physical streaming replication is commonly operated asynchronously.

Simplified flow:

```text
Primary Transaction
       │
       ▼
Local COMMIT
       │
       ▼
Client receives success
       │
       ▼
WAL continues to standby
```

The primary does not necessarily wait for the standby to acknowledge the transaction before confirming commit to the application.

Benefit:

```text
Lower write latency
```

Trade-off:

```text
Potential data loss during failover
```

because some recently committed WAL might not yet have reached the standby.

---

# 10. Synchronous Replication

With synchronous replication, PostgreSQL can make committing transactions wait for confirmation from selected standby servers.

Conceptually:

```text
Application
    │
    ▼
Primary
    │
    ▼
Generate WAL
    │
    ▼
Send to Standby
    │
    ▼
Standby acknowledges
    │
    ▼
Primary COMMIT completes
    │
    ▼
Application receives success
```

PostgreSQL determines active synchronous standbys using `synchronous_standby_names`; selected standbys must be connected and streaming. ([PostgreSQL][4])

---

# 11. Synchronous Replication Trade-Off

Synchronous replication gives stronger durability guarantees but increases dependency on network and standby health.

Conceptually:

```text
Asynchronous
--------------
Fast commit
Potential data loss

Synchronous
--------------
Safer failover
Higher commit latency
Standby/network dependency
```

Therefore this is an **RPO vs latency** decision, not simply "sync is better."

---

# 12. `synchronous_standby_names`

Example:

```conf
synchronous_standby_names = 'standby1'
```

More advanced configurations can specify multiple standby choices.

Conceptually:

```text
Primary
 │
 ├── Standby1
 ├── Standby2
 └── Standby3
```

PostgreSQL can select one or more synchronous standbys depending on how `synchronous_standby_names` is configured. ([PostgreSQL][4])

---

# 13. Synchronous Commit Levels

PostgreSQL allows different synchronous durability behaviors through:

```text
synchronous_commit
```

Conceptually, confirmation can be based on different stages such as WAL reaching:

```text
Primary local WAL
Standby WAL write
Standby WAL flush
Standby WAL replay
```

The exact choice affects:

```text
Durability
Commit latency
Read-after-write behavior
```

This should be designed based on application requirements.

---

# 14. Replication Slots

Replication slots help PostgreSQL retain WAL needed by a replication consumer.

Conceptually:

```text
Primary WAL
   │
   ▼
Replication Slot
   │
   ▼
Tracks consumer progress
   │
   ▼
Required WAL retained
```

Without a slot, WAL required by a disconnected standby could be recycled if it is no longer otherwise required.

A slot protects the consumer—but introduces an important operational risk.

---

# 15. Replication Slot Risk

Suppose the consumer goes offline:

```text
Standby offline
      │
      ▼
Slot remains active/inactive
      │
      ▼
Primary retains required WAL
      │
      ▼
pg_wal grows
      │
      ▼
Filesystem can fill
```

Therefore replication slots must be monitored.

Use:

```sql
SELECT *
FROM pg_replication_slots;
```

Important items include:

```text
slot_name
slot_type
active
restart_lsn
confirmed_flush_lsn
wal_status
```

depending on slot type and status.

---

# 16. Physical Replication Slot

Example:

```sql
SELECT pg_create_physical_replication_slot('standby1_slot');
```

Then configure the standby to use that slot.

Conceptually:

```text
Primary
   │
   └── Physical Slot: standby1_slot
                 │
                 ▼
              Standby1
```

This protects the WAL needed by that standby.

---

# 17. Streaming Replication Configuration

The primary must be configured to support replication.

Important areas include:

```text
wal_level
max_wal_senders
max_replication_slots
pg_hba.conf
```

and potentially:

```text
synchronous_standby_names
```

for synchronous replication.

Do not think of replication setup as one parameter. It involves:

```text
WAL generation
Network access
Authentication
Standby initialization
Replication connection
Retention
Monitoring
```

---

# 18. Replication Authentication

Replication connections are also controlled through `pg_hba.conf`.

A conceptual entry could be:

```conf
host  replication  repl_user  10.10.20.10/32  scram-sha-256
```

Meaning:

```text
Replication connection
      │
      ├── User: repl_user
      ├── Source: 10.10.20.10
      └── Authentication: SCRAM
```

You should not give unnecessary login/database permissions to replication accounts.

---

# 19. Replication Role

A replication role can be created like:

```sql
CREATE ROLE repl_user
WITH LOGIN REPLICATION
PASSWORD 'secure-password';
```

In production, the password should come from an approved secrets-management process instead of being hard-coded.

The important privilege is:

```text
REPLICATION
```

which allows the role to initiate replication connections.

---

# 20. Creating a Standby with `pg_basebackup`

One common way to initialize a physical standby is:

```bash
pg_basebackup
```

Conceptually:

```text
Primary Cluster
      │
      ▼
pg_basebackup
      │
      ▼
Standby PGDATA
      │
      ▼
Start standby
```

A common pattern is:

```bash
pg_basebackup \
  -h primary-host \
  -U repl_user \
  -D /pgdata/18/data \
  -R \
  -X stream \
  -P
```

The exact flags should be chosen for your build standard rather than copied blindly.

---

# 21. `standby.signal`

Modern PostgreSQL uses:

```text
standby.signal
```

to indicate standby operation.

When present, PostgreSQL starts in standby mode and uses recovery-related configuration to connect to the upstream server.

Conceptually:

```text
PGDATA
   │
   ├── standby.signal
   │
   └── PostgreSQL config
              │
              ▼
         Start as standby
```

---

# 22. Primary Connection Information

The standby commonly uses:

```conf
primary_conninfo = 'host=primary-host port=5432 user=repl_user ...'
```

to identify how it should connect to the primary.

If using a slot:

```conf
primary_slot_name = 'standby1_slot'
```

Conceptually:

```text
Standby
  │
  ├── primary_conninfo
  │       └── where to connect
  │
  └── primary_slot_name
          └── which slot to use
```

---

# 23. Monitoring Primary Replication

On the primary:

```sql
SELECT *
FROM pg_stat_replication;
```

A useful query:

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

This helps determine:

```text
Is standby connected?
Is it streaming?
How far has WAL been sent?
Written?
Flushed?
Replayed?
Is it synchronous?
```

---

# 24. Understanding Replication LSNs

LSN means:

```text
Log Sequence Number
```

It identifies a position in WAL.

Important replication positions include:

```text
sent_lsn
write_lsn
flush_lsn
replay_lsn
```

Conceptually:

```text
Primary WAL Current Position
            │
            ▼
         sent_lsn
            │
            ▼
         write_lsn
            │
            ▼
         flush_lsn
            │
            ▼
         replay_lsn
```

The gaps between these positions help locate replication lag.

---

# 25. Replication Lag Layers

Replication lag is not one single number.

There can be:

```text
Send lag
Write lag
Flush lag
Replay lag
```

Architecture:

```text
Primary
   │
   │ Send
   ▼
Network
   │
   ▼
Standby receive
   │
   │ Write
   ▼
Filesystem
   │
   │ Flush
   ▼
Durable WAL
   │
   │ Replay
   ▼
Database state
```

If you only monitor one "lag seconds" value, you may miss where the actual bottleneck exists.

---

# 26. Calculate WAL Byte Lag

You can also compare LSN positions.

For example:

```sql
SELECT
    application_name,
    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_current_wal_lsn(),
            replay_lsn
        )
    ) AS replay_byte_lag
FROM pg_stat_replication;
```

This tells you how much WAL separates the primary's current position from what the standby has replayed.

This can be more useful than only looking at time-based lag.

---

# 27. Monitoring the Standby

On the standby:

```sql
SELECT *
FROM pg_stat_wal_receiver;
```

Useful checks include:

```text
Connection to upstream
Sender host/port
Received LSN
Latest message time
Slot information
```

Also check:

```sql
SELECT pg_is_in_recovery();
```

On a standby:

```text
true
```

On a normal primary:

```text
false
```

---

# 28. Replay Position on Standby

Useful functions include:

```sql
SELECT pg_last_wal_receive_lsn();
```

and:

```sql
SELECT pg_last_wal_replay_lsn();
```

Conceptually:

```text
Received WAL
      │
      ▼
pg_last_wal_receive_lsn()

Replayed WAL
      │
      ▼
pg_last_wal_replay_lsn()
```

A large gap between receive and replay positions suggests the standby can receive WAL but cannot replay fast enough.

---

# 29. Cascading Replication

PostgreSQL supports cascading replication.

Instead of:

```text
Primary
 ├── Standby1
 ├── Standby2
 ├── Standby3
 └── Standby4
```

you could use:

```text
Primary
   │
   ▼
Standby1
   │
   ├── Standby2
   └── Standby3
```

A cascading standby can receive WAL from an upstream server and send it further downstream. PostgreSQL documents this as a way to reduce direct connections to the primary and reduce inter-site bandwidth usage. ([PostgreSQL][5])

---

# 30. Cascading Replication Use Case

Imagine:

```text
Data Center A
    Primary
       │
       │ WAN
       ▼
Data Center B
   Standby1
      │
      ├── Standby2
      └── Standby3
```

Instead of sending three separate streams across the WAN:

```text
Primary → B1
Primary → B2
Primary → B3
```

you can send one stream:

```text
Primary → B1
```

then replicate locally:

```text
B1 → B2
B1 → B3
```

This can reduce WAN utilization.

---

# 31. WAL Archive + Streaming Replication

Streaming replication and WAL archiving can be used together.

Architecture:

```text
             PRIMARY
               │
        ┌──────┴──────┐
        ▼             ▼
 Streaming WAL     WAL Archive
        │             │
        ▼             ▼
     Standby      Archive Store
        │
        └──── fallback/recovery
```

This provides more resilience if streaming is interrupted and the required WAL can still be obtained from archive.

---

# 32. Hot Standby Conflict

Because a standby is replaying changes from the primary while also serving read queries, conflicts can occur.

Example:

```text
Standby Query
     │
     │ Needs old row version
     ▼
Long SELECT

At same time:

Primary VACUUM/change
     │
     ▼
WAL arrives on standby
     │
     ▼
Recovery wants cleanup
```

PostgreSQL may need to delay recovery or cancel the standby query depending on configuration and circumstances.

This is one reason read replicas are not simply "free read capacity."

---

# 33. `hot_standby_feedback`

One configuration related to standby query conflicts is:

```conf
hot_standby_feedback = on
```

This can help reduce query cancellations on the standby by informing the primary about snapshots needed on the standby.

Trade-off:

```text
Fewer standby query conflicts
          vs
Potential additional bloat on primary
```

This should be evaluated based on workload.

---

# 34. Failover

Failover means promoting a standby so it becomes a primary.

Conceptually:

```text
BEFORE

Primary
   │
   ▼
Standby


Primary fails
   │
   ▼

Standby
   │
   ▼
PROMOTE
   │
   ▼
New Primary
```

PostgreSQL includes promotion capabilities, but deciding **which standby to promote and how applications reconnect** is an HA orchestration problem.

PostgreSQL's HA documentation treats failover as a separate architectural concern. ([PostgreSQL][1])

---

# 35. Promotion

A standby can be promoted using tools/functions such as:

```bash
pg_ctl promote
```

or PostgreSQL mechanisms designed for promotion.

After promotion:

```text
Standby
   │
   ▼
Recovery ends
   │
   ▼
Read/write database
   │
   ▼
New WAL timeline
```

After this happens, the old primary cannot simply be assumed to rejoin automatically.

---

# 36. Switchover vs Failover

These terms should be distinguished.

### Switchover

Planned role change.

```text
Primary healthy
     │
     ▼
Controlled transition
     │
     ▼
Standby promoted
```

Used for:

```text
Maintenance
Patch work
Data center testing
Planned migration
```

### Failover

Unplanned transition caused by failure.

```text
Primary fails unexpectedly
       │
       ▼
Standby promoted
```

Failover generally carries more operational risk.

---

# 37. Split Brain Risk

A serious HA problem occurs if both nodes accept writes independently:

```text
       Network Problem

Primary A          Primary B
    │                  │
    ▼                  ▼
Writes A            Writes B
```

Now both sides diverge.

This is:

```text
split brain
```

Preventing it requires more than replication. It requires:

```text
Failover coordination
Leader election
Fencing
Client routing
Operational controls
```

This belongs mainly in **Topic 6 — HA & Failover Architecture**.

---

# 38. Replication and RPO

Replication mode affects Recovery Point Objective.

Example:

```text
Asynchronous replication

Primary accepts commit
        │
        ▼
Primary crashes before standby receives WAL
        │
        ▼
Some committed transactions potentially lost
```

So:

```text
RPO may be > 0
```

With an appropriately designed synchronous architecture, the data-loss window can be reduced significantly, potentially toward zero for specified failure assumptions.

---

# 39. Replication and RTO

Replication also affects Recovery Time Objective.

Without standby:

```text
Failure
  │
  ▼
Restore backup
  │
  ▼
Replay WAL
  │
  ▼
Service restored
```

Potentially long.

With a ready standby:

```text
Failure
  │
  ▼
Detect
  │
  ▼
Promote
  │
  ▼
Redirect application
```

Potentially much faster.

But replication alone does **not** guarantee low RTO. Detection, promotion, DNS/VIP/proxy routing, fencing and application recovery all matter.

---

# 40. Logical Replication

PostgreSQL logical replication operates using:

```text
Publisher
Publication
Subscriber
Subscription
```

Architecture:

```text
             PUBLISHER
                 │
                 ▼
             Publication
                 │
          Logical Changes
                 │
                 ▼
             Subscription
                 │
                 ▼
             SUBSCRIBER
```

PostgreSQL describes logical replication as a publish/subscribe model where subscribers pull data from publications. ([PostgreSQL][2])

---

# 41. Publication

A publication defines which changes are available for logical replication.

Example:

```sql
CREATE PUBLICATION sales_pub
FOR TABLE orders, customers;
```

Conceptually:

```text
Publisher Database
      │
      ├── orders
      ├── customers
      ├── inventory
      │
      ▼
Publication
      │
      ├── orders
      └── customers
```

A publication is defined within one database and represents a set of table changes. ([PostgreSQL][6])

---

# 42. Subscription

On the target:

```sql
CREATE SUBSCRIPTION sales_sub
CONNECTION 'host=source dbname=sales user=repl_user'
PUBLICATION sales_pub;
```

Conceptually:

```text
Publisher
    │
    │ sales_pub
    ▼
Subscription
    │
    ▼
Subscriber
```

PostgreSQL creates/uses a logical replication slot to track subscription progress in the normal subscription architecture.

---

# 43. Logical Replication Initial Copy

When a subscription is created, PostgreSQL can initially copy existing table data and then continue applying changes.

Conceptually:

```text
Step 1
Initial Table Copy
      │
      ▼
Subscriber gets existing rows

Step 2
Logical Changes
      │
      ▼
INSERT / UPDATE / DELETE continue
```

This makes logical replication useful for migration and selective data replication.

---

# 44. Logical Replication Use Cases

Logical replication is useful for:

```text
Major-version migration
Selective table replication
Data distribution
Reporting systems
Integration platforms
Near-zero-downtime migration
Database consolidation/splitting
```

Example:

```text
Source PostgreSQL
     │
     ├── table_a
     ├── table_b
     └── table_c

Logical Replication
     │
     └── only table_a + table_b

Target PostgreSQL
```

Physical replication cannot selectively replicate only specific normal tables in the same way.

---

# 45. Logical Replication and DDL

A critical operational point:

Logical replication focuses mainly on logical data changes.

You should not assume that all schema/DDL changes will automatically appear on the subscriber.

For migrations:

```text
Schema preparation
      │
      ▼
Logical replication
      │
      ▼
Ongoing DDL coordination
      │
      ▼
Cutover
```

Schema compatibility must be managed.

---

# 46. Replica Identity

For logical replication, PostgreSQL needs to identify rows for `UPDATE` and `DELETE`.

Normally this uses a primary key or suitable replica identity.

Conceptually:

```text
UPDATE customer
SET name = ...
WHERE customer_id = 100
```

Logical replication needs a reliable way to identify:

```text
Which row should be updated on subscriber?
```

PostgreSQL's publication documentation explicitly includes Replica Identity as part of logical replication behavior. ([PostgreSQL][6])

---

# 47. Logical Replication Slots

Logical replication also uses replication slots.

A logical slot tracks:

```text
How far the consumer has processed logical changes
```

Conceptually:

```text
WAL
 │
 ▼
Logical Decoding
 │
 ▼
Logical Slot
 │
 ▼
Subscriber / Consumer
```

As with physical slots, inactive logical slots can retain WAL and must be monitored.

---

# 48. Logical Decoding

Logical decoding converts WAL changes into a logical form.

Architecture:

```text
WAL Records
    │
    ▼
Logical Decoding
    │
    ▼
Logical Change
    │
    ├── INSERT
    ├── UPDATE
    └── DELETE
```

This is the foundation not only of PostgreSQL logical replication but also many CDC systems.

---

# 49. Physical vs Logical Migration Example

Suppose you want to migrate:

```text
PostgreSQL 16
      │
      ▼
PostgreSQL 18
```

Physical replication is tied closely to physical cluster compatibility and is not a general cross-major-version migration mechanism.

Logical replication, however, can be used between suitable PostgreSQL versions because it transfers logical changes rather than requiring the target to replay the source's physical storage state.

This makes logical replication important for upgrade/migration architectures.

---

# 50. `pg_createsubscriber`

PostgreSQL 18 includes the `pg_createsubscriber` utility, which can help transform a physical standby into a logical replication subscriber as part of certain migration workflows. PostgreSQL documents options for publications, subscriptions and logical replication slots. ([PostgreSQL][7])

This is a more advanced tool and should be used only after understanding the topology and operational constraints.

---

# 51. Replication Monitoring Dashboard

A useful production replication dashboard should contain:

```text
PRIMARY
  │
  ├── WAL generation rate
  ├── WAL sender count
  ├── Connected standbys
  ├── sent/write/flush/replay positions
  ├── replication byte lag
  ├── replication time lag
  ├── slot status
  └── retained WAL

STANDBY
  │
  ├── Recovery status
  ├── WAL receiver status
  ├── receive position
  ├── replay position
  ├── replay lag
  ├── recovery conflicts
  └── read workload

LOGICAL
  │
  ├── subscription status
  ├── worker status
  ├── slot status
  └── synchronization health
```

---

# 52. Key Replication Alerts

I would configure alerts around:

| Condition                       | Why it matters                     |
| ------------------------------- | ---------------------------------- |
| Standby disconnected            | HA protection reduced              |
| Replication lag above SLA       | RPO/read consistency risk          |
| WAL receiver stopped            | Standby not receiving WAL          |
| Replay lag growing              | Standby can't keep up              |
| Replication slot inactive       | WAL retention risk                 |
| Excessive retained WAL          | Disk-full risk                     |
| Synchronous standby unavailable | Commit latency/availability impact |
| Archive failures                | Recovery/PITR risk                 |
| Recovery conflicts              | Read replica workload impact       |
| Subscriber stopped              | Logical replication data drift     |
| Replication worker failure      | Logical replication interrupted    |

The alert threshold should be based on your RPO/RTO and workload baseline.

---

# 53. Replication Troubleshooting Flow

A useful sequence is:

```text
Replication problem
       │
       ▼
Is standby connected?
       │
       ▼
Check pg_stat_replication
       │
       ▼
Is WAL being sent?
       │
       ▼
Check network/authentication
       │
       ▼
Check pg_stat_wal_receiver
       │
       ▼
Is WAL received but not replayed?
       │
       ▼
Check standby CPU / I/O
       │
       ▼
Check recovery conflicts
       │
       ▼
Check WAL/archive availability
       │
       ▼
Check replication slots
       │
       ▼
Check PostgreSQL logs
```

---

# 54. Production Physical Replication Architecture

A common enterprise pattern:

```text
                         Applications
                              │
                              ▼
                            Proxy
                              │
                              ▼
                          PRIMARY
                              │
                 ┌────────────┼────────────┐
                 │            │            │
                 ▼            ▼            ▼
             Standby1      Standby2     WAL Archive
             Local HA       DR Site         │
                 │            │             ▼
                 │            │        Backup Storage
                 │            │
             Read-only     DR / Failover
```

This combines:

```text
HA
Read scaling
DR
Backup recovery
```

but those are different objectives and should be designed separately.

---

# 55. Important Architecture Principle

Replication is **not backup**.

If someone runs:

```sql
DROP TABLE customers;
```

on the primary:

```text
Primary
   │
   │ WAL
   ▼
Standby
   │
   ▼
DROP TABLE customers
```

The standby faithfully replicates the mistake.

Similarly:

```text
Bad UPDATE
Bad DELETE
Application corruption
```

can all be replicated.

Therefore you still need:

```text
Backups
WAL archive
PITR
Retention
Restore testing
```

Replication protects primarily against certain infrastructure failures; backup protects against additional recovery scenarios.

---

# 56. Replication Best Practices

For production PostgreSQL, I would standardize:

```text
Use dedicated replication roles
Use secure authentication
Monitor replication continuously
Monitor replication slots
Monitor retained WAL
Define expected replication lag
Align replication mode with RPO
Test failover regularly
Test standby promotion
Protect against split brain
Combine replication with backup/PITR
Document topology
Understand sync vs async trade-offs
Avoid reporting workloads overwhelming standby
Monitor network latency
Monitor standby storage performance
```

---

# 57. Physical Replication Checklist

| Area                      | Validation                   |
| ------------------------- | ---------------------------- |
| Primary WAL configuration | Correct                      |
| `max_wal_senders`         | Capacity planned             |
| Replication slots         | Correctly sized/monitored    |
| Replication user          | Least privilege              |
| `pg_hba.conf`             | Restricted                   |
| Base backup               | Successfully created         |
| `primary_conninfo`        | Correct                      |
| Slot                      | Correctly configured if used |
| Standby mode              | Confirmed                    |
| Streaming state           | Healthy                      |
| Lag                       | Within SLA                   |
| Archive                   | Healthy if configured        |
| Read-only traffic         | Tested                       |
| Promotion                 | Tested                       |
| Monitoring                | Alerts configured            |
| Backup                    | Independent backup exists    |

---

# 58. Logical Replication Checklist

| Area             | Validation              |
| ---------------- | ----------------------- |
| Publisher        | Correct                 |
| Publication      | Correct table selection |
| Subscriber       | Ready                   |
| Schema           | Compatible              |
| Replica identity | Valid                   |
| Initial copy     | Completed               |
| Subscription     | Enabled                 |
| Slot             | Healthy                 |
| Apply workers    | Running                 |
| DDL changes      | Coordinated             |
| Sequences        | Migration plan reviewed |
| Conflicts        | Monitored               |
| Lag              | Monitored               |
| Cutover          | Tested                  |
| Rollback plan    | Documented              |

---

## PostgreSQL 18 Official References

The main PostgreSQL 18 reference is **Chapter 26 — High Availability, Load Balancing, and Replication**, which covers standby operation, streaming replication, replication slots, cascading replication, synchronous replication, failover and hot standby. ([PostgreSQL][1])

For streaming replication configuration, PostgreSQL 18's replication settings document parameters such as `synchronous_standby_names` and related replication controls. ([PostgreSQL][4])

For protocol-level understanding of WAL streaming and replication connections, use the PostgreSQL 18 **Streaming Replication Protocol** documentation. ([PostgreSQL][3])

For logical replication, use **Chapter 29 — Logical Replication**, which describes PostgreSQL's publisher/publication/subscriber/subscription model. ([PostgreSQL][2]) The publication documentation explains that publications represent sets of table changes and covers replica identity. ([PostgreSQL][6])

**Tracker: 5 / 29 completed — 24 pending.**

**Next: Topic 6 — PostgreSQL HA & Failover Architecture**, covering RPO/RTO, synchronous vs asynchronous HA design, automatic failover, quorum, fencing, split-brain prevention, switchover, client routing, multi-AZ/multi-site patterns, and production HA reference architectures.

[1]: https://www.postgresql.org/docs/current/high-availability.html?utm_source=chatgpt.com "18: Chapter 26. High Availability, Load Balancing, and ..."
[2]: https://www.postgresql.org/docs/current/logical-replication.html?utm_source=chatgpt.com "Documentation: 18: Chapter 29. Logical Replication"
[3]: https://www.postgresql.org/docs/current/protocol-replication.html?utm_source=chatgpt.com "Documentation: 18: 54.4. Streaming Replication Protocol"
[4]: https://www.postgresql.org/docs/current/runtime-config-replication.html?utm_source=chatgpt.com "Documentation: 18: 19.6. Replication"
[5]: https://www.postgresql.org/docs/current/warm-standby.html?utm_source=chatgpt.com "Documentation: 18: 26.2. Log-Shipping Standby Servers"
[6]: https://www.postgresql.org/docs/current/logical-replication-publication.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 29.1. Publication"
[7]: https://www.postgresql.org/docs/current/app-pgcreatesubscriber.html?utm_source=chatgpt.com "Documentation: 18: pg_createsubscriber"
