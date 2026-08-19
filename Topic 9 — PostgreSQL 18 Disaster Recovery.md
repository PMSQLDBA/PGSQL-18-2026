# Topic 9 — PostgreSQL 18 Disaster Recovery

Disaster Recovery (DR) is the design and process used to restore PostgreSQL service after a **major failure** such as loss of a data center, cloud region, storage platform, entire PostgreSQL cluster, or significant data corruption.

PostgreSQL 18 provides the technical building blocks for DR through **physical backups, WAL archiving, Point-in-Time Recovery (PITR), streaming standbys, failover, recovery targets, timelines, and `pg_rewind`**. ([PostgreSQL][1])

The key difference from HA is:

```text
HA
→ Keep service running through local failures.

DR
→ Recover service after a major site/system/data-loss event.
```

---

## 1. HA vs DR

These concepts overlap, but they should not be treated as the same thing.

| High Availability               | Disaster Recovery                               |
| ------------------------------- | ----------------------------------------------- |
| Usually protects local service  | Protects against major/site-wide failure        |
| Low RTO focus                   | RPO + RTO focus                                 |
| Standby often nearby            | DR copy usually geographically separated        |
| Automatic failover common       | DR activation may be controlled/manual          |
| Protects server/AZ failures     | Protects region/site/storage disasters          |
| Replication is central          | Replication + backup + WAL + recovery           |
| Seconds/minutes recovery target | Minutes/hours depending on business requirement |

Example:

```text
                 REGION A
        ┌────────────────────┐
        │ Primary            │
        │ Local Standby      │
        └──────────┬─────────┘
                   │
                   │ Asynchronous WAL
                   ▼
                 REGION B
        ┌────────────────────┐
        │ DR Standby         │
        │ Backup Repository  │
        └────────────────────┘
```

Here:

```text
Primary + Local Standby
        =
HA

Region B Standby + Backups
        =
DR
```

---

# 2. DR Starts with RPO and RTO

Before designing PostgreSQL DR, define:

```text
RPO = Recovery Point Objective
RTO = Recovery Time Objective
```

### RPO

How much data can the business afford to lose?

Example:

```text
RPO = 15 minutes
```

means the organization accepts up to approximately 15 minutes of data loss in the defined disaster scenario.

### RTO

How long can the application/database remain unavailable?

Example:

```text
RTO = 60 minutes
```

means service should be recovered within about one hour.

The architecture must be built around these requirements.

---

# 3. Example DR Tiers

You can classify applications into DR tiers.

### Tier 1 — Critical

```text
RPO: 0–seconds/minutes
RTO: minutes
```

Possible architecture:

```text
Remote streaming standby
+
Continuous WAL archiving
+
Automatic/controlled DR orchestration
+
Off-site backup
```

### Tier 2 — Important

```text
RPO: 15–60 minutes
RTO: 1–4 hours
```

Possible architecture:

```text
Remote asynchronous standby
+
Backups
+
PITR
```

### Tier 3 — Standard

```text
RPO: hours/day
RTO: several hours
```

Possible architecture:

```text
Off-site physical backup
+
Archived WAL
```

There is no universal PostgreSQL DR tiering standard; the business should define it.

---

# 4. Main PostgreSQL DR Methods

A PostgreSQL DR strategy can use:

```text
1. Backup + Restore

2. Backup + WAL + PITR

3. Remote Streaming Standby

4. Remote Standby + WAL Archive

5. Logical Replication DR

6. Cloud/object-storage recovery
```

For most critical PostgreSQL systems, the strongest architecture combines several.

---

# 5. Backup-Based DR

The simplest DR model:

```text
Production
     │
     ▼
Physical Backup
     │
     ▼
Off-site Repository
```

After disaster:

```text
Provision new server
      │
      ▼
Restore backup
      │
      ▼
Start PostgreSQL
      │
      ▼
Application reconnects
```

Advantages:

```text
Simple
Low infrastructure cost
Good protection against corruption
```

Weakness:

```text
Potentially large RPO
Potentially long RTO
```

---

# 6. Backup + WAL DR

A stronger design:

```text
PostgreSQL
    │
    ├────────► Base Backup
    │
    └────────► WAL Archive
                    │
                    ▼
             Remote Repository
```

After disaster:

```text
Base Backup
     +
Archived WAL
     │
     ▼
PITR
     │
     ▼
Recovered PostgreSQL
```

This is PostgreSQL's continuous archiving architecture. PostgreSQL 18 documents PITR as replaying archived WAL from a valid base backup to reconstruct database state. ([PostgreSQL][2])

---

# 7. Why PITR Is Important for DR

Suppose:

```text
08:00 Backup

11:00 Normal
12:00 Normal
12:37 Data corruption occurs
12:45 Corruption detected
```

Restoring only the 08:00 backup means losing:

```text
4+ hours of transactions
```

With WAL:

```text
08:00 Base Backup
       │
       +
WAL 08:00 → 12:36
       │
       ▼
Recover to 12:36
```

This reduces the logical data-loss window dramatically.

---

# 8. Remote Standby DR

Instead of waiting for backup restore, you can continuously maintain a DR standby:

```text
              REGION A

               Primary
                  │
                  │ WAL
                  │ WAN
                  ▼

              REGION B

              DR Standby
```

PostgreSQL's standby architecture allows a standby to continuously recover WAL streamed from an upstream PostgreSQL server. ([PostgreSQL][1])

If Region A is lost:

```text
Region A unavailable
        │
        ▼
Validate disaster
        │
        ▼
Promote Region B standby
        │
        ▼
Redirect applications
```

RTO can be dramatically shorter than restoring a large database from backup.

---

# 9. Why Remote Replication Is Usually Asynchronous

Suppose:

```text
Region A ↔ Region B
Round-trip latency = 80 ms
```

If every transaction must synchronously wait for Region B:

```text
Application
   │
   ▼
Primary
   │
   ▼
Remote region
   │
   ▼
Acknowledgement
   │
   ▼
COMMIT completes
```

application latency can increase significantly.

Therefore a common architecture is:

```text
Local standby
    → synchronous

Remote DR standby
    → asynchronous
```

This balances:

```text
Local RPO
Performance
Regional protection
```

---

# 10. Local HA + Remote DR

One of the most practical enterprise PostgreSQL designs:

```text
                         Applications
                              │
                              ▼
                          Endpoint
                              │
                              ▼
                           Primary
                              │
                  ┌───────────┴────────────┐
                  │                        │
                  ▼                        ▼
        Local Sync Standby         Remote Async Standby
             AZ/Local HA               DR Region
```

Responsibilities:

```text
Local Standby
→ Fast HA

Remote Standby
→ Regional disaster recovery

Backups
→ Historical/data corruption recovery
```

This is generally stronger than trying to make one standby solve every requirement.

---

# 11. Remote DR Standby Is Not Enough

A DR standby alone does not protect against every disaster.

Suppose:

```sql
DROP DATABASE finance;
```

PostgreSQL generates WAL.

The remote standby then receives:

```text
DROP DATABASE finance
```

So:

```text
Primary corrupted
      │
      ▼
Replication
      │
      ▼
DR standby corrupted
```

Therefore:

```text
DR Standby
+
Backup
+
WAL History
```

is stronger than replication alone.

---

# 12. Geographic Separation

Your DR copy should not share the same failure domain as production.

Poor DR:

```text
Rack 1
 ├── Primary
 └── DR Standby
```

Better:

```text
Data Center A
 └── Primary

Data Center B
 └── DR Standby
```

Cloud example:

```text
Region A
 └── Primary + HA

Region B
 └── DR
```

Think in terms of:

```text
Power
Network
Storage
Rack
Building
AZ
Region
Provider
```

depending on the disaster scenarios you're protecting against.

---

# 13. DR Storage Separation

Backups should also be isolated.

Poor architecture:

```text
Primary PostgreSQL
      │
      └── Backup on same filesystem
```

If storage fails:

```text
Database lost
+
Backup lost
```

Better:

```text
PostgreSQL
    │
    ├── Local Backup
    │
    └── Remote Backup Repository
```

For critical systems:

```text
Production storage
      ≠
Primary backup storage
      ≠
Off-site backup copy
```

---

# 14. WAL Archive for DR

A strong DR architecture archives WAL somewhere independent of the active database.

```text
Primary
   │
   ├── WAL streaming → DR standby
   │
   └── WAL archive   → Object/backup store
```

This creates two recovery paths:

```text
Fast path
→ Promote DR standby

Fallback path
→ Restore backup + WAL
```

That redundancy is valuable.

---

# 15. Streaming + Archive Fallback

A standby can use streaming replication and archived WAL.

Conceptually:

```text
Primary
   │
   ├──────── Streaming WAL ────────┐
   │                               ▼
   │                           DR Standby
   │                               ▲
   └────► WAL Archive ─────────────┘
```

If streaming is interrupted, archived WAL can be another source of required recovery records depending on the standby/recovery configuration. PostgreSQL's HA chapter covers continuous archiving in standby environments. ([PostgreSQL][1])

---

# 16. `restore_command`

During archive recovery PostgreSQL needs a way to retrieve WAL.

Conceptually:

```conf
restore_command = 'restore-wal %f %p'
```

Where the recovery system retrieves the requested WAL segment from the archive and places it where PostgreSQL expects it.

Production systems commonly integrate this through backup products rather than raw shell-copy commands.

---

# 17. `recovery.signal`

For archive recovery:

```text
PGDATA
  │
  ├── recovery.signal
  │
  └── recovery settings
```

PostgreSQL starts and enters archive recovery.

Conceptually:

```text
Start PostgreSQL
      │
      ▼
Detect recovery.signal
      │
      ▼
Restore WAL
      │
      ▼
Replay
      │
      ▼
Reach recovery target
```

---

# 18. DR Recovery Targets

You may need to recover to:

```text
Exact time
Transaction ID
LSN
Named restore point
End of available WAL
```

For example:

```conf
recovery_target_time = '2026-08-19 12:36:50'
```

This can be used to recover immediately before an unwanted transaction.

The exact recovery point should be verified carefully during the DR procedure.

---

# 19. Restore Points

Before major production changes:

```sql
SELECT pg_create_restore_point('before_release_2026_08');
```

Conceptually:

```text
Normal workload
      │
      ▼
Restore Point
      │
      ▼
Application Upgrade
```

If recovery is later required:

```text
Backup
   +
WAL
   │
   ▼
Recover to restore point
```

Again, the restore point itself is not a backup.

---

# 20. Disaster Scenarios

A good DR design should explicitly cover scenarios.

### Scenario 1 — Complete primary server loss

Recovery:

```text
HA standby
or
DR standby
```

### Scenario 2 — Entire data-center loss

Recovery:

```text
Remote DR standby
```

### Scenario 3 — Database corruption

Recovery:

```text
Backup + PITR
```

### Scenario 4 — Accidental DROP

Recovery:

```text
PITR
```

### Scenario 5 — Backup repository corruption

Recovery:

```text
Second/off-site repository
```

### Scenario 6 — Regional cloud outage

Recovery:

```text
Cross-region DR
```

One DR architecture should not be assumed to solve every case automatically.

---

# 21. DR Activation

DR activation should follow a controlled process.

```text
Disaster detected
      │
      ▼
Declare DR event
      │
      ▼
Confirm primary cannot safely serve
      │
      ▼
Validate DR standby
      │
      ▼
Check replication position
      │
      ▼
Promote
      │
      ▼
Redirect client traffic
      │
      ▼
Validate database
      │
      ▼
Validate application
      │
      ▼
Declare service restored
```

Do not promote merely because a monitoring probe briefly failed.

---

# 22. DR Declaration

Organizations usually need a clear authority for DR activation.

For example:

```text
DBA
     │
Infrastructure
     │
Application Owner
     │
Incident Commander
     │
     ▼
DR Declaration
```

This matters because cross-region failover can have major consequences.

Technical automation is only one part of the DR process.

---

# 23. Preventing Split Brain During DR

Suppose Region A loses communication with Region B.

Region B should not blindly assume:

```text
Region A is destroyed.
```

Region A might still be serving applications.

Before promoting DR:

```text
Fence old primary
or
ensure application traffic cannot reach it.
```

Desired state:

```text
Only one writable PostgreSQL primary
```

at any given time.

---

# 24. DNS-Based DR Routing

A simple DR approach:

```text
app-db.company.com
        │
        ▼
Region A PostgreSQL
```

During disaster:

```text
DNS update
        │
        ▼
Region B PostgreSQL
```

Potential issue:

```text
DNS TTL
Caching
Client behavior
```

can affect actual RTO.

So DNS failover must be tested rather than assumed.

---

# 25. Global Load Balancer / Service Endpoint

A more advanced approach:

```text
Applications
     │
     ▼
Global Endpoint
     │
 ┌───┴────┐
 ▼        ▼
Region A Region B
Active   DR
```

When Region A is unavailable:

```text
Global Endpoint
      │
      ▼
Region B
```

Again, routing is external infrastructure; PostgreSQL itself does not provide global application traffic management.

---

# 26. Active/Passive DR

The most common PostgreSQL DR model:

```text
Region A
Primary
   │
   ▼
Region B
Passive Standby
```

Region B normally does not accept writes.

Advantages:

```text
Simple data ownership
Lower split-brain risk
Clear promotion path
```

This is usually safer than trying to maintain independent multi-writer PostgreSQL regions.

---

# 27. Active/Read-Only DR

The DR standby can sometimes also serve reads:

```text
Region A
Primary
   │
   ▼
Region B
Hot Standby
   │
   └── Reporting
```

Benefits:

```text
DR hardware is used
Replication can be continuously validated
Reporting workload offloaded
```

But:

```text
Reporting workload
must not prevent DR readiness.
```

The DR node's primary purpose is recovery.

---

# 28. DR Capacity Planning

The DR environment needs enough capacity to actually run production.

Bad design:

```text
Production:
64 CPU
512 GB RAM

DR:
8 CPU
32 GB RAM
```

The DR database may successfully promote but fail under production traffic.

You should validate:

```text
CPU
RAM
Storage IOPS
Network
Connection capacity
Backup throughput
WAL replay
Application throughput
```

in the DR environment.

---

# 29. DR Standby Lag

For asynchronous DR:

```text
Primary current WAL
       │
       ▼
DR replay position
```

The gap is your potential replication RPO exposure.

Monitor:

```text
Sent LSN
Write LSN
Flush LSN
Replay LSN
Time lag
Byte lag
```

A DR standby that is hours behind may not meet the intended RPO even though replication technically says "streaming."

---

# 30. WAN Capacity

Remote replication requires network bandwidth sufficient for WAL generation.

Example:

```text
Peak WAL generation
= 100 MB/sec
```

but:

```text
WAN capacity available
= 30 MB/sec
```

Then:

```text
WAL generation
    >
WAN transfer rate
```

lag will grow continuously.

Capacity planning should therefore include peak WAL rates, not only average application traffic.

---

# 31. WAL Burst Planning

Certain operations can dramatically increase WAL:

```text
Bulk UPDATE
CREATE INDEX
Large COPY
VACUUM FULL
Data migration
Batch loads
Schema operations
```

So design:

```text
WAN capacity
WAL archive capacity
DR storage
```

for bursts as well as normal workload.

---

# 32. DR Storage Capacity

The DR environment needs space for:

```text
PostgreSQL data
pg_wal
Temporary files
Logs
Backup staging
Growth
Replay bursts
```

Also monitor retained WAL due to:

```text
Replication slots
Standby downtime
Network problems
```

A DR outage should not cause the primary's `pg_wal` filesystem to fill.

---

# 33. Replication Slots in DR

A physical replication slot can ensure WAL needed by a DR standby remains available.

But:

```text
DR standby offline
      │
      ▼
Slot retains WAL
      │
      ▼
Primary storage grows
```

Therefore slot-based DR requires monitoring and limits.

You need operational answers to:

```text
How long can DR be offline?

How much WAL may be retained?

When should we rebuild rather than retain?
```

---

# 34. DR Standby Rebuild

Sometimes lag/recovery problems make it safer to rebuild.

```text
DR standby unusable
      │
      ▼
Take fresh base backup
      │
      ▼
Transfer to DR
      │
      ▼
Start standby
      │
      ▼
Catch up WAL
```

`pg_basebackup` can create full physical base backups from a primary and, under documented conditions, from a standby. ([PostgreSQL][3])

For very large databases, rebuild time should be included in DR risk planning.

---

# 35. Backup from DR / Standby

One useful design:

```text
Primary
   │
   ▼
DR Standby
   │
   ▼
Backup
```

This can reduce backup I/O on the production primary.

PostgreSQL 18 documents that `pg_basebackup` can run against a standby, with specific requirements and limitations. ([PostgreSQL][3])

However, backups should not create a workload that compromises standby catch-up or DR readiness.

---

# 36. DR and `pg_rewind`

After Region B is promoted:

```text
Region A Primary
      X

Region B DR Standby
      │
      ▼
Promoted
      │
      ▼
New Primary
```

Later Region A returns.

Its history may now have diverged.

A common reintegration approach is:

```text
Old Region A primary
       │
       ▼
pg_rewind
       │
       ▼
Resynchronize
       │
       ▼
Region A becomes standby
```

PostgreSQL 18 documents `pg_rewind` specifically for synchronizing diverged PostgreSQL clusters, including the typical post-failover old-primary scenario. ([PostgreSQL][4])

---

# 37. `pg_rewind` vs Full Rebuild

### `pg_rewind`

Potentially faster when divergence is manageable.

```text
Compare changed blocks
      │
      ▼
Synchronize differences
```

### Full rebuild

```text
Delete/reinitialize target
      │
      ▼
Fresh base backup
      │
      ▼
Start standby
```

Use the appropriate method based on:

```text
WAL availability
Cluster state
Timeline divergence
Backup availability
Operational confidence
```

---

# 38. DR Failback

Failover to DR is only half the story.

Eventually you may want production back in the original region.

Example:

```text
Before disaster:

Region A = Primary
Region B = DR


After disaster:

Region B = Primary


After Region A restored:

Region B = Primary
Region A = Standby
```

Then later:

```text
Controlled switchover

Region A = Primary
Region B = DR
```

This is **failback**.

---

# 39. Failback Workflow

A safer workflow:

```text
1. Restore Region A infrastructure
        │
        ▼
2. Build Region A as standby
        │
        ▼
3. Synchronize from Region B
        │
        ▼
4. Verify lag = acceptable
        │
        ▼
5. Schedule maintenance window
        │
        ▼
6. Stop/drain writes
        │
        ▼
7. Ensure Region A caught up
        │
        ▼
8. Promote Region A
        │
        ▼
9. Redirect applications
        │
        ▼
10. Reconfigure Region B as DR
```

Do not simply restore an old backup in Region A and point applications back to it.

---

# 40. DR Timelines

After failover, PostgreSQL creates a new WAL timeline.

Conceptually:

```text
Timeline 1
     │
     ├──────── old Region A
     │
     ▼
Failover
     │
     └──────── Timeline 2
                  │
                  ▼
              Region B
```

Timelines allow PostgreSQL recovery mechanisms to distinguish different histories after promotion.

This matters during:

```text
PITR
Failback
pg_rewind
Archive management
```

---

# 41. Backup Retention for DR

Your DR strategy should maintain enough history to recover from incidents that are discovered late.

Example:

```text
Corruption introduced:
Monday

Detected:
Friday
```

If retention is:

```text
2 days
```

the clean recovery point may already be gone.

Therefore backup retention should reflect:

```text
Detection time
Compliance
Business recovery window
Storage cost
```

not simply backup frequency.

---

# 42. Off-Site Backup

A minimum DR principle:

```text
Production failure domain
        ≠
Backup failure domain
```

For example:

```text
Region A PostgreSQL
     │
     ▼
Region B/Object Storage Backup
```

For very critical data, additional controls may include:

```text
Immutable backup
Separate credentials
Separate account/subscription
Encryption
Retention locks
```

Those are infrastructure/security controls rather than PostgreSQL core functionality.

---

# 43. Protection Against Ransomware / Credential Compromise

If the same credentials can:

```text
Delete production
and
Delete all backups
```

your DR design has a serious weakness.

A stronger architecture separates:

```text
Database credentials
Backup writer credentials
Backup delete privileges
Cloud account privileges
```

and may use immutable or retention-locked storage.

---

# 44. DR Monitoring

Monitor both production and DR.

Important DR indicators:

```text
DR standby reachable
Streaming active
Replay position
Replication lag
WAL receive rate
WAL replay rate
Archive status
Backup success
Backup age
Repository capacity
DR filesystem capacity
DR PostgreSQL health
Network latency
WAN throughput
```

DR should not be treated as:

```text
"we'll check it during a disaster"
```

---

# 45. DR Alerting

Important alerts include:

| Condition                           | Why it matters                 |
| ----------------------------------- | ------------------------------ |
| DR standby disconnected             | DR protection reduced          |
| Replication lag exceeds RPO         | Recovery target at risk        |
| WAL receiver stopped                | DR not receiving changes       |
| WAL replay lag increasing           | DR cannot keep up              |
| WAL archive failure                 | PITR/recovery chain risk       |
| Backup too old                      | Backup RPO risk                |
| Backup repository near full         | Future backups may fail        |
| Replication slot retaining huge WAL | Primary disk risk              |
| DR filesystem near full             | Standby may stop               |
| DR node unhealthy                   | Failover candidate unavailable |
| Restore test overdue                | Recoverability unverified      |

---

# 46. DR Runbook

A production DR runbook should contain:

```text
Incident declaration criteria

Current primary identification

DR standby validation

Replication lag check

Old-primary fencing procedure

DR promotion procedure

Application-routing procedure

Database validation queries

Application smoke tests

Backup restore procedure

PITR procedure

Rollback strategy

Old-site reintegration

pg_rewind procedure

Failback procedure

Contact/escalation list
```

A DR plan existing only in one DBA's memory is not a reliable DR plan.

---

# 47. Example DR Activation Runbook

```text
1. Incident commander declares DR
2. Confirm Region A unavailable
3. Stop/fence Region A write path
4. Verify Region B PostgreSQL health
5. Check latest replay LSN
6. Record estimated RPO loss
7. Promote Region B
8. Confirm pg_is_in_recovery() = false
9. Confirm read/write transaction succeeds
10. Change global DB endpoint
11. Restart/reconnect application pools
12. Run application validation
13. Verify monitoring
14. Declare recovery complete
```

The exact steps depend on your environment and HA tooling.

---

# 48. Validate the DR Primary

After promotion:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
false
```

Then basic validation:

```sql
SELECT now();
```

Test controlled writes:

```sql
CREATE TABLE dr_validation_test
(
    id integer,
    test_time timestamptz
);
```

or preferably use a pre-approved application-level validation process rather than creating arbitrary objects in production.

---

# 49. Application Validation

Database promotion alone is not a successful DR test.

Validate:

```text
Application login
Read transactions
Write transactions
Background jobs
Messaging integrations
Batch processes
Reporting
External APIs
Connection pools
Secrets
DNS/network routing
```

You can have:

```text
PostgreSQL DR = healthy
```

while:

```text
Application = unavailable
```

because of routing or external dependencies.

---

# 50. DR Dependencies

Document external dependencies such as:

```text
DNS
Load balancers
Application servers
Secrets vault
IAM
Certificates
Storage
Message queues
ETL
Schedulers
Monitoring
Backup repository
Network routes
Firewall rules
```

A DR architecture should cover the **service**, not only the PostgreSQL process.

---

# 51. DR Testing Types

### Tabletop test

Walk through procedure without actual failover.

Useful for:

```text
Runbook review
Roles/responsibilities
Communication gaps
```

### Technical test

Actually restore a backup or promote a test standby.

### Full DR simulation

```text
Primary site unavailable
      │
      ▼
Activate secondary site
      │
      ▼
Run application workload
```

This is the strongest validation.

---

# 52. PITR Test

A useful recurring exercise:

```text
1. Select backup
2. Provision isolated PostgreSQL
3. Restore base backup
4. Retrieve WAL
5. Recover to chosen time
6. Start database
7. Validate data
8. Record total recovery duration
```

This validates:

```text
Backup
WAL archive
Recovery settings
Staff knowledge
RTO
```

---

# 53. DR Drill Metrics

Capture:

```text
Failure detection time
Decision time
Promotion time
DNS/routing time
Application reconnect time
Data-validation time
Total RTO
Observed RPO
```

Example:

```text
Detection       5 min
Decision        5 min
Promotion       2 min
Routing         3 min
Validation     10 min
----------------------
Actual RTO     25 min
```

Now you can compare:

```text
Required RTO = 30 min

Actual RTO = 25 min

PASS
```

---

# 54. DR Testing Frequency

Frequency depends on business criticality.

Possible models:

```text
Tier 1
→ Quarterly DR activation tests

Tier 2
→ Semiannual

Tier 3
→ Annual
```

This is organizational guidance, not a PostgreSQL requirement.

Restore testing may happen more frequently than a full site failover.

---

# 55. DR and Logical Replication

Logical replication can also be used in some DR designs:

```text
Publisher
    │
    ▼
Logical Replication
    │
    ▼
Subscriber in DR
```

Advantages:

```text
Selective replication
Version flexibility
Migration flexibility
```

But logical replication has different behavior from physical standby replication and requires consideration of:

```text
DDL
Sequences
Conflicts
Replica identity
Subscription state
```

For core PostgreSQL HA/DR, physical replication is usually simpler when maintaining a full binary standby.

---

# 56. DR Across PostgreSQL Versions

A physical standby normally belongs to the same PostgreSQL major-version architecture.

For migration between major versions, logical mechanisms may be more appropriate.

So:

```text
Same-version DR
→ Physical streaming replication

Cross-version migration
→ Logical replication / pg_dump / upgrade methods
```

Do not confuse DR topology with upgrade topology.

---

# 57. DR Backup Architecture

A strong design might be:

```text
                     PRIMARY REGION
                           │
                    PostgreSQL 18
                           │
            ┌──────────────┼───────────────┐
            │              │               │
            ▼              ▼               ▼
      Local Standby      Backup         WAL Archive
            │              │               │
            │              └──────┬────────┘
            │                     ▼
            │             Remote/Object Store
            │
            │ WAN Replication
            ▼
                      DR REGION
                           │
                     DR Standby
```

This gives three independent recovery mechanisms:

```text
1. Local failover
2. Remote standby
3. Backup + PITR
```

---

# 58. Example Enterprise DR Architecture

```text
                    ┌─────────────────────┐
                    │    Applications     │
                    └──────────┬──────────┘
                               │
                     Global DB Endpoint
                               │
             ┌─────────────────┴────────────────┐
             │                                  │
             ▼                                  ▼
        REGION A                           REGION B
     ACTIVE REGION                        DR REGION
             │                                  │
      ┌──────┴───────┐                   ┌──────┴───────┐
      │ Primary      │                   │ DR Standby   │
      │ Sync Standby │──── Async WAL ───►│ PostgreSQL   │
      └──────┬───────┘                   └──────────────┘
             │
             ├────────► WAL Archive
             │
             └────────► Physical Backup
                             │
                             ▼
                     Remote Repository
```

This is a strong reference model for critical PostgreSQL workloads.

---

# 59. DR Decision Matrix

| Failure                    | Preferred Recovery     |
| -------------------------- | ---------------------- |
| PostgreSQL process failure | Local HA               |
| Primary server failure     | Local standby          |
| AZ failure                 | Standby in another AZ  |
| Data-center failure        | Remote DR standby      |
| Regional failure           | Cross-region DR        |
| Accidental DELETE          | PITR / object recovery |
| Accidental DROP            | PITR                   |
| Logical corruption         | Backup/PITR            |
| Lost primary + standby     | Backup restore         |
| Old primary after failover | `pg_rewind` or rebuild |
| DR standby too far behind  | Evaluate PITR/rebuild  |
| Backup corruption          | Secondary repository   |

---

# 60. Disaster Recovery Checklist

| Area                   | Validation              |
| ---------------------- | ----------------------- |
| Business RPO           | Defined                 |
| Business RTO           | Defined                 |
| DR site                | Separate failure domain |
| DR standby             | Healthy                 |
| Replication mode       | Defined                 |
| Replication lag        | Within RPO              |
| WAL archiving          | Healthy                 |
| Physical backups       | Current                 |
| Off-site copies        | Available               |
| Backup retention       | Adequate                |
| Immutable protection   | Considered              |
| Network capacity       | Validated               |
| DR compute capacity    | Production-capable      |
| DR storage capacity    | Adequate                |
| Promotion procedure    | Tested                  |
| Fencing                | Tested                  |
| Client routing         | Tested                  |
| Application validation | Tested                  |
| PITR                   | Tested                  |
| `pg_rewind`            | Tested where applicable |
| Failback               | Documented              |
| Monitoring             | Enabled                 |
| Alerts                 | Enabled                 |
| Runbook                | Current                 |
| DR drill               | Successfully completed  |

---

# 61. Core Rules to Remember

```text
HA and DR are not the same.

Replication is not backup.

DR must start with RPO and RTO.

Keep DR in a separate failure domain.

Use local HA for fast local recovery.

Use remote standby for site/region recovery.

Use backup + WAL for historical recovery.

Monitor DR continuously.

Asynchronous DR must be monitored for lag.

Protect WAL archives.

Test PITR.

Test actual promotion.

Test application routing.

Plan failback before a disaster occurs.

A standby that has never been tested is not a proven DR solution.
```

---

## PostgreSQL 18 Official References

The main official PostgreSQL 18 reference is **Chapter 26 — High Availability, Load Balancing, and Replication**. It covers standby operation, streaming replication, synchronous replication, continuous archiving in standbys, failover, and hot standby behavior. ([PostgreSQL][1])

For failover, PostgreSQL 18 documents promotion through `pg_ctl promote` or `pg_promote()`, along with considerations around synchronized logical replication slots during failover. ([PostgreSQL][5])

For backup-driven DR and PITR, PostgreSQL 18's Server Administration documentation places **Continuous Archiving and Point-in-Time Recovery** alongside HA and replication as the core recovery mechanisms. ([PostgreSQL][2]) `wal_level = replica` or higher is required for continuous archiving and physical streaming replication; `minimal` WAL is insufficient for PITR. ([PostgreSQL][6])

For rebuilding or creating DR standbys, `pg_basebackup` can produce full or incremental physical base backups and can write standby recovery configuration with `-R`. PostgreSQL 18 also supports taking a base backup from a properly configured standby, subject to documented limitations. ([PostgreSQL][3])

For returning an old primary after DR activation, PostgreSQL 18 documents **`pg_rewind`** as a way to synchronize clusters whose timelines diverged after failover; otherwise, a fresh base backup may be required. ([PostgreSQL][4])

**Tracker: 9 / 29 completed — 20 pending.**

**Next: Topic 10 — PostgreSQL Connection Management**, covering PostgreSQL backend connections, `max_connections`, reserved connections, connection lifecycle, connection storms, PgBouncer, session vs transaction pooling, timeouts, application connection sizing, HA-aware pooling, and production capacity design.

[1]: https://www.postgresql.org/docs/current/high-availability.html?utm_source=chatgpt.com "18: Chapter 26. High Availability, Load Balancing, and ..."
[2]: https://www.postgresql.org/docs/current/admin.html?utm_source=chatgpt.com "Documentation: 18: Part III. Server Administration"
[3]: https://www.postgresql.org/docs/current/app-pgbasebackup.html?utm_source=chatgpt.com "Documentation: 18: pg_basebackup"
[4]: https://www.postgresql.org/docs/current/app-pgrewind.html?utm_source=chatgpt.com "Documentation: 18: pg_rewind"
[5]: https://www.postgresql.org/docs/current/warm-standby-failover.html?utm_source=chatgpt.com "Documentation: 18: 26.3. Failover"
[6]: https://www.postgresql.org/docs/current/runtime-config-wal.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.5. Write Ahead Log"
