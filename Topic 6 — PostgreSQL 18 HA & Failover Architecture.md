# Topic 6 — PostgreSQL 18 HA & Failover Architecture

High Availability (HA) in PostgreSQL is the design used to keep the database service available when a server, process, network path, or data-center component fails. PostgreSQL provides the core building blocks such as streaming replication, synchronous replication, standby promotion, WAL-based recovery, and tools like `pg_rewind`; however, a complete HA solution also needs failure detection, client routing, fencing, and orchestration. ([PostgreSQL][1])

## 1. What HA Actually Means

A highly available PostgreSQL system should answer four questions:

```text
1. What happens when the primary fails?
2. How quickly can another node take over?
3. How much data can we afford to lose?
4. How will applications reach the new primary?
```

That leads directly to two key business measures:

```text
RPO = Recovery Point Objective
      How much data loss is acceptable?

RTO = Recovery Time Objective
      How long can the service be unavailable?
```

For example:

```text
RPO = 0
RTO = 60 seconds
```

means:

```text
No committed transaction loss desired
Database service should recover within ~60 seconds
```

HA design should start from these requirements, not from a tool choice.

---

# 2. Basic PostgreSQL HA Architecture

The simplest HA topology is:

```text
                Applications
                     │
                     ▼
              Connection Endpoint
                     │
                     ▼
                  PRIMARY
                     │
                     │ WAL
                     ▼
                  STANDBY
```

The primary accepts writes.

The standby continuously receives and replays WAL.

If the primary fails:

```text
Primary Failure
      │
      ▼
Standby Promotion
      │
      ▼
New Primary
      │
      ▼
Applications reconnect
```

PostgreSQL's HA documentation describes standby servers, streaming replication, synchronous replication, failover, and load-balancing mechanisms as the core building blocks for these designs. ([PostgreSQL][1])

---

# 3. HA Is More Than Replication

A common mistake is:

```text
We have streaming replication
        =
We have HA
```

That is incomplete.

Replication provides:

```text
Data synchronization
Standby copy
Potential failover target
```

But a complete HA architecture also needs:

```text
Failure detection
Promotion decision
Leader election
Fencing
Client routing
Split-brain prevention
Old-primary reintegration
Monitoring
Operational procedures
```

So:

```text
Replication
      +
Failover Management
      +
Routing
      +
Fencing
      +
Monitoring
      =
HA Architecture
```

---

# 4. Asynchronous HA

In asynchronous replication:

```text
Application
    │
    ▼
Primary
    │
    ▼
COMMIT locally
    │
    ▼
Client receives success
    │
    ▼
WAL later reaches standby
```

This provides lower write latency.

But there is a possible failure window:

```text
COMMIT
  │
  ▼
Primary confirms success
  │
  ▼
Primary fails
  │
  ▼
WAL had not yet reached standby
```

Result:

```text
Potential data loss
```

So asynchronous HA typically gives:

```text
Better performance
Lower dependency on standby/network
RPO may be greater than zero
```

---

# 5. Synchronous HA

With synchronous replication, PostgreSQL can make transactions wait for confirmation from one or more standbys before completing commit. PostgreSQL 18 controls this through `synchronous_standby_names`, together with `synchronous_commit`. ([PostgreSQL][2])

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
Send WAL to standby
    │
    ▼
Standby confirms required state
    │
    ▼
Primary confirms COMMIT
```

This can reduce the possible data-loss window.

Trade-off:

```text
Better durability
      vs
Higher transaction latency
```

---

# 6. Synchronous Standby Selection

PostgreSQL 18 allows multiple potential synchronous standbys.

Example concept:

```conf
synchronous_standby_names = 'FIRST 1 (standby1, standby2)'
```

Priority architecture:

```text
Primary
  │
  ├── standby1  ← preferred synchronous
  └── standby2  ← replacement if standby1 fails
```

PostgreSQL states that in priority-based synchronous replication, earlier standbys in the list are preferred, and later ones can take over if a current synchronous standby fails. ([PostgreSQL][3])

---

# 7. Quorum-Based Synchronous Replication

PostgreSQL also supports quorum-based synchronous replication using `ANY`.

Conceptually:

```text
Primary
 │
 ├── Standby A
 ├── Standby B
 └── Standby C
```

Configuration concept:

```text
ANY 2
```

means:

```text
Commit may continue after
ANY 2 eligible standbys
acknowledge the WAL
```

This can improve resilience compared with depending on one exact standby.

Conceptually:

```text
         Primary
            │
      ┌─────┼─────┐
      ▼     ▼     ▼
      A     B     C

Need acknowledgements
from any 2
```

PostgreSQL's synchronous replication implementation distinguishes priority-based and quorum-based modes. ([PostgreSQL][2])

---

# 8. Priority vs Quorum

### Priority Mode

```text
FIRST 1 (A, B, C)
```

Preference:

```text
A first
B if A unavailable
C if A and B unavailable
```

Useful when standbys have clear roles.

For example:

```text
A = same data center
B = nearby AZ
C = remote region
```

### Quorum Mode

```text
ANY 2 (A, B, C)
```

Any two can satisfy synchronous commit.

This can be useful when all candidate standbys are reasonably equivalent.

---

# 9. Commit Durability Levels

`synchronous_commit` affects how far WAL must progress before PostgreSQL considers the commit complete. PostgreSQL 18 documents that when synchronous standbys are configured, this setting controls whether transactions wait for standby processing. ([PostgreSQL][4])

At a simplified level:

```text
Primary generates WAL
       │
       ▼
Standby receives
       │
       ▼
Standby writes
       │
       ▼
Standby flushes
       │
       ▼
Standby replays
```

Different commit modes can require different points.

This creates a durability/latency spectrum:

```text
Lower wait
    │
    ▼
Lower latency
Potentially weaker remote guarantee

Higher wait
    │
    ▼
Stronger remote confirmation
Higher latency
```

---

# 10. HA Across Availability Zones

A common cloud or data-center design:

```text
                   Region
                     │
          ┌──────────┴──────────┐
          │                     │
         AZ1                   AZ2
          │                     │
       Primary               Standby
          │                     ▲
          └──────── WAL ────────┘
```

This protects against:

```text
Server failure
Storage failure
AZ-level infrastructure failure
```

depending on infrastructure design.

For lower RPO, synchronous replication may be used across nearby zones if latency is acceptable.

---

# 11. Multi-Site HA/DR Architecture

For larger environments:

```text
                    Site A
                      │
                   Primary
                      │
            ┌─────────┴─────────┐
            │                   │
            ▼                   ▼
       Local Standby       Remote Standby
            │                   │
            │                   ▼
            │                Site B
            │
            ▼
          Fast HA
```

Typical design:

```text
Local standby
    → low RTO

Remote standby
    → disaster recovery
```

Not every standby needs the same replication mode.

For example:

```text
Local standby  → synchronous
Remote standby → asynchronous
```

This balances:

```text
Durability
Latency
Geographic protection
```

---

# 12. Failover Sequence

A controlled failover process normally looks like:

```text
1. Detect primary failure
        │
        ▼
2. Confirm failure is real
        │
        ▼
3. Fence/isolate old primary
        │
        ▼
4. Select best standby
        │
        ▼
5. Promote standby
        │
        ▼
6. Redirect clients
        │
        ▼
7. Validate service
        │
        ▼
8. Rebuild/rejoin old primary
```

The exact order matters.

Especially:

```text
Fencing
before
allowing multiple writable primaries
```

---

# 13. Failure Detection

HA systems need to determine:

```text
Is the primary really down?
```

Possible signals:

```text
PostgreSQL process unavailable
TCP connection failure
Database health-check failure
Server unreachable
Storage failure
Node heartbeat loss
Replication heartbeat loss
```

But a failed health check does not always mean the primary itself is dead.

Example:

```text
Network partition
```

may make the primary unreachable to the HA manager while the database itself is still accepting application writes.

That is why failover must be designed carefully.

---

# 14. Network Partition Problem

Suppose:

```text
Site A                Site B
Primary               Standby
   │                     │
   └──── network X ──────┘
```

Site B loses connectivity to Site A.

Site B may think:

```text
Primary is dead
```

while Site A still sees:

```text
Primary is healthy
```

If B promotes itself:

```text
Primary A          New Primary B
    │                   │
    ▼                   ▼
 Writes              Writes
```

This creates split brain.

---

# 15. Split Brain

Split brain means more than one node believes it is the writable primary.

```text
                NETWORK PARTITION

        Primary A             Primary B
            │                     │
            ▼                     ▼
        Write Set A           Write Set B
```

Now data diverges.

This is one of the most serious HA failure modes.

PostgreSQL's replication mechanisms by themselves do not solve arbitrary external leader-election/fencing problems; the architecture must ensure only one writable primary is allowed to serve the application.

---

# 16. Fencing

Fencing means ensuring the old primary cannot continue accepting writes after another node is promoted.

Conceptually:

```text
Old Primary
    │
    ▼
Fence
    │
    ├── Shut down server
    ├── Disable network
    ├── Remove storage access
    ├── Power off VM
    └── Block client routing
```

Then:

```text
Promote standby
```

The architecture objective is:

```text
Only ONE writable primary
```

at all times.

---

# 17. STONITH Concept

You may hear the term:

```text
STONITH
```

which means roughly:

```text
Shoot The Other Node In The Head
```

It is an infrastructure-clustering term for forcibly ensuring a failed/isolated node cannot continue operating.

In database HA, the underlying principle is:

```text
Do not rely only on "I cannot see the old primary."

Ensure the old primary cannot write.
```

---

# 18. Quorum Concept

Quorum is commonly used by HA control systems to avoid two independent sides both making leadership decisions.

Example:

```text
3 voting members

Node A
Node B
Node C
```

A majority is:

```text
2 of 3
```

If the environment splits:

```text
A + B          C
```

A+B have quorum.

C does not.

Conceptually:

```text
Only side with majority
may make leadership decision
```

PostgreSQL's core synchronous replication has quorum-based commit semantics, while cluster-manager quorum/leader election is generally provided by external HA coordination systems.

---

# 19. Witness / Consensus Node

Some HA architectures use a third node for consensus.

```text
Primary
    │
Standby
    │
Witness / Consensus node
```

The third system does not necessarily store PostgreSQL data.

Its purpose may be:

```text
Leader election
Quorum
Cluster state
Split-brain prevention
```

This pattern is common with external coordination systems.

---

# 20. Primary Election

When the primary fails and multiple standbys exist:

```text
Standby A
Standby B
Standby C
```

the HA layer should choose the safest candidate.

Important factors include:

```text
Replication position
Health
Location
Priority
Synchronous status
WAL availability
Timeline
Application-routing design
```

Normally:

```text
Most suitable + most current standby
```

should be promoted.

---

# 21. Promotion

PostgreSQL supports promoting a standby to primary.

Example:

```bash
pg_ctl promote
```

Conceptually:

```text
Standby
   │
   ▼
Promotion
   │
   ▼
End recovery
   │
   ▼
New timeline
   │
   ▼
Read/write primary
```

The promotion operation is only one part of failover.

HA also needs:

```text
Decision
Fencing
Routing
Validation
Rejoin
```

---

# 22. Timeline Changes

After failover:

```text
Old Primary Timeline
         │
         ├────────── old path
         │
         └────────── new primary timeline
```

PostgreSQL creates a new timeline when recovery is promoted.

This allows PostgreSQL to distinguish the new recovery history from the old primary's history.

This matters when:

```text
Rejoining old primary
Recovering from archive
Using pg_rewind
```

---

# 23. Old Primary After Failover

Suppose:

```text
Primary A fails
Standby B promoted
```

Then:

```text
B = new primary

A = old primary
```

You should **not** simply restart A and assume it can continue as a standby.

Their timelines may have diverged.

Architecture:

```text
         Common history
              │
        ┌─────┴─────┐
        ▼           ▼
    Old A        New B
```

The old primary must be resynchronized.

---

# 24. `pg_rewind`

PostgreSQL provides `pg_rewind` specifically for synchronizing a PostgreSQL cluster with another copy of the same cluster after their timelines diverge, commonly when bringing an old primary back as a standby after failover. ([PostgreSQL][5])

Conceptually:

```text
Old Primary A
     │
     │ diverged
     ▼
pg_rewind
     │
     ▼
Synchronize with
New Primary B
     │
     ▼
A rejoins as standby
```

This can be much faster than rebuilding the entire standby from a fresh base backup when conditions allow.

---

# 25. Rejoin Workflow

Typical workflow:

```text
Primary A
   │
   X failure

Standby B
   │
   ▼
Promoted
   │
   ▼
New Primary B

Old A returns
   │
   ▼
Ensure A stopped
   │
   ▼
pg_rewind / rebuild
   │
   ▼
Configure A as standby
   │
   ▼
A follows B
```

---

# 26. Client Routing

After failover, applications must know where the new primary is.

You do not want application connection strings hard-coded like:

```text
db-host-01
```

if HA requires the writable role to move.

Common abstraction:

```text
Application
     │
     ▼
Logical Database Endpoint
     │
     ▼
Current Primary
```

Possible mechanisms include:

```text
Proxy
VIP
DNS
Service discovery
Load balancer
Cluster-managed endpoint
```

These are infrastructure decisions rather than PostgreSQL core replication features.

---

# 27. Proxy-Based Routing

Conceptually:

```text
             Applications
                  │
                  ▼
                Proxy
                  │
          ┌───────┴────────┐
          ▼                ▼
       Primary          Standby
       Read/Write        Read-only
```

After failover:

```text
Proxy
  │
  ▼
New Primary
```

The application connection target remains stable.

---

# 28. Read/Write vs Read-Only Endpoints

A mature architecture may expose:

```text
Writer Endpoint
      │
      ▼
Current Primary

Reader Endpoint
      │
      ├── Standby1
      ├── Standby2
      └── Standby3
```

This separates:

```text
OLTP writes
from
read scaling
```

But reader routing must consider replication lag.

A query sent to a lagging standby may not immediately see a transaction committed on the primary.

---

# 29. Failover vs Switchover

These should not be confused.

### Failover

Unplanned.

```text
Unexpected Primary Failure
          │
          ▼
Promote Standby
```

### Switchover

Planned.

```text
Healthy Primary
      │
      ▼
Prepare Standby
      │
      ▼
Controlled Role Exchange
```

Switchover is useful for:

```text
Maintenance
OS patching
Hardware replacement
Testing
Planned migration
```

---

# 30. Planned Switchover Flow

Example:

```text
1. Verify standby healthy
2. Verify replication caught up
3. Stop or drain writes
4. Confirm final WAL received
5. Demote/stop old primary
6. Promote standby
7. Redirect applications
8. Reconfigure old primary as standby
9. Validate
```

A controlled switchover should minimize both:

```text
Data risk
Downtime
```

---

# 31. Automatic vs Manual Failover

### Manual Failover

Human/operator decides:

```text
Failure detected
      │
      ▼
DBA validates
      │
      ▼
DBA promotes standby
```

Advantages:

```text
Lower risk of false promotion
Human judgment
```

Disadvantages:

```text
Higher RTO
Operator dependency
```

### Automatic Failover

HA system detects and acts automatically.

```text
Failure
  │
  ▼
Health check
  │
  ▼
Consensus
  │
  ▼
Fence
  │
  ▼
Promote
```

Advantages:

```text
Lower RTO
```

but requires very strong design around:

```text
Quorum
Fencing
False positives
Network partitions
```

---

# 32. RPO Design Examples

### RPO: minutes

Potential architecture:

```text
Asynchronous remote standby
```

Suitable where some data loss is acceptable.

### RPO: seconds

```text
Low-lag asynchronous standby
+
fast monitoring
```

### RPO: near zero / zero under defined failure assumptions

```text
Synchronous standby
```

assuming commit confirmation requirements and failure scenarios are correctly defined.

There is always an infrastructure assumption behind "zero data loss."

---

# 33. RTO Design Examples

### RTO: hours

```text
Backup restore
```

might be enough.

### RTO: 15 minutes

```text
Warm standby
+
manual failover
```

may work.

### RTO: 1 minute

```text
Streaming standby
+
automatic failure detection
+
promotion
+
routing automation
```

may be required.

The tighter the RTO:

```text
More automation
More complexity
More testing
```

---

# 34. One Standby HA Design

```text
          Application
               │
               ▼
             Proxy
               │
               ▼
            Primary
               │
               ▼
            Standby
```

Advantages:

```text
Simple
Lower infrastructure cost
Easy to understand
```

Weakness:

```text
Only one failover candidate
```

During maintenance or another failure:

```text
redundancy may temporarily be zero
```

---

# 35. Two-Standby HA Design

```text
                Primary
               /       \
              /         \
             ▼           ▼
       Standby A      Standby B
```

Possible roles:

```text
Standby A → local synchronous HA
Standby B → asynchronous DR
```

This is a strong enterprise pattern.

---

# 36. Three-Node HA Design

```text
                 Primary
                /       \
               ▼         ▼
          Standby1    Standby2
```

plus:

```text
Consensus/quorum infrastructure
```

This can give more flexibility for:

```text
Maintenance
Node failures
Quorum
Synchronous standby selection
```

---

# 37. Multi-Region Design

Example:

```text
                 Region A
              ┌─────────────┐
              │ Primary     │
              │ Standby A   │
              └──────┬──────┘
                     │
                     │ asynchronous WAL
                     ▼
                 Region B
              ┌─────────────┐
              │ Standby B   │
              └─────────────┘
```

Why asynchronous cross-region?

Because WAN latency can significantly affect synchronous commit latency.

Example:

```text
Cross-region RTT = 60 ms
```

A synchronous transaction may need to incorporate remote network delay into commit latency depending on the selected durability mode.

---

# 38. Local Sync + Remote Async Pattern

A very common architecture:

```text
                    PRIMARY
                       │
            ┌──────────┴──────────┐
            │                     │
            ▼                     ▼
   Local Synchronous        Remote Asynchronous
       Standby                  Standby
            │                     │
            ▼                     ▼
      Local HA                Regional DR
```

Advantages:

```text
Low data-loss risk locally
Fast local failover
Protection from regional disaster
Avoid WAN latency on every commit
```

---

# 39. Standby Health

Before promoting a standby, validate:

```text
Streaming state
Replication lag
WAL availability
Replay position
Storage health
Database health
Timeline
Server health
```

A standby that is:

```text
2 hours behind
```

may technically be alive but may be a poor failover candidate.

---

# 40. HA Monitoring

Important HA monitoring includes:

```text
Primary availability
Standby availability
Replication state
Replication lag
WAL sender status
WAL receiver status
Replication slots
Synchronous standby state
WAL archive status
Node health
Network health
Disk health
Failover manager health
```

Key PostgreSQL views include:

```sql
pg_stat_replication
pg_stat_wal_receiver
pg_replication_slots
```

These should feed your monitoring platform.

---

# 41. Synchronous Standby Failure

Suppose:

```text
Primary
   │
   ▼
Synchronous Standby
```

If the standby disappears, behavior depends on the synchronous configuration and availability of alternate eligible standbys.

PostgreSQL recommends keeping enough potential synchronous standbys configured so the requested synchronous set can be maintained when one fails. ([PostgreSQL][3])

Otherwise applications may experience:

```text
COMMIT waiting
```

while PostgreSQL waits for required synchronous acknowledgement.

---

# 42. HA Availability vs Durability Trade-Off

Consider:

```text
Require synchronous acknowledgement
```

This protects data.

But if every synchronous standby becomes unavailable:

```text
Primary is alive
but
commits may block
```

So there is a classic trade-off:

```text
Data durability
        vs
Write availability
```

This decision must be made with business owners, not only DBAs.

---

# 43. HA and Backup

HA is not backup.

Example:

```sql
DELETE FROM customer;
```

If correctly committed:

```text
Primary
    │
    │ WAL
    ▼
Standby
```

The delete is replicated.

HA protects against:

```text
Server failures
Infrastructure failures
```

Backups/PITR protect against additional scenarios such as:

```text
Human errors
Logical corruption
Accidental DROP
Bad application deployment
Historical recovery
```

You need both.

---

# 44. HA and Disaster Recovery

HA and DR are related but different.

### HA

Usually focuses on:

```text
Fast recovery from local failures
```

### DR

Usually focuses on:

```text
Large-scale site/region failure
```

For example:

```text
AZ1 + AZ2
     │
     ▼
HA

Region A → Region B
     │
     ▼
DR
```

That distinction will be expanded under **Topic 9 — PostgreSQL Disaster Recovery**.

---

# 45. Failure Scenarios to Test

Do not test only:

```text
kill PostgreSQL process
```

A proper HA validation should include:

```text
PostgreSQL process failure
Server reboot
Server crash
Storage failure
Network failure
Network partition
Primary-to-standby link failure
Standby failure
Proxy failure
Consensus service failure
Disk full
WAL archive failure
Loss of entire availability zone
```

Each can produce different behavior.

---

# 46. Failover Testing

A production HA design should regularly test:

```text
Detection time
Promotion time
Client reconnection
RPO
RTO
Split-brain protection
Old-primary fencing
Rejoin process
pg_rewind process
Application recovery
Monitoring alerts
```

A failover solution that has never been tested is not a validated HA solution.

---

# 47. Example RTO Measurement

Suppose:

```text
Primary failure detected = 10 sec
Consensus/fencing         = 5 sec
Promotion                 = 5 sec
Routing update            = 10 sec
Application reconnection  = 10 sec
```

Then:

```text
Approximate RTO = 40 sec
```

This shows why RTO includes much more than PostgreSQL promotion time.

---

# 48. Example RPO Measurement

Suppose asynchronous replication lag is:

```text
2 MB WAL
```

at the time primary storage is permanently lost.

Then the actual possible data-loss window relates to:

```text
transactions represented in
that missing WAL
```

not simply "replication was configured."

You should monitor both:

```text
Time lag
and
Byte lag
```

---

# 49. Controlled Failover Architecture

A strong sequence is:

```text
               Failure Detected
                      │
                      ▼
                Validate Failure
                      │
                      ▼
                  Quorum Check
                      │
                      ▼
               Fence Old Primary
                      │
                      ▼
            Select Best Standby
                      │
                      ▼
                   Promote
                      │
                      ▼
              Update Routing
                      │
                      ▼
             Validate Read/Write
                      │
                      ▼
               Restore Redundancy
```

This is a safer mental model than:

```text
"Primary unavailable → immediately promote something"
```

---

# 50. Production Reference Architecture

A robust enterprise pattern could be:

```text
                     Applications
                          │
                          ▼
               Database Service Endpoint
                          │
                          ▼
                    Proxy / Router
                          │
                          ▼
                       PRIMARY
                          │
              ┌───────────┴────────────┐
              │                        │
              ▼                        ▼
      Synchronous Standby       Asynchronous DR
          Same Region              Remote Site
              │                        │
              └──────────┬─────────────┘
                         │
                  HA Coordination
                         │
            ┌────────────┼─────────────┐
            ▼            ▼             ▼
         Health       Quorum        Fencing
         Checks       State         Control
```

This separates:

```text
Database replication
HA decision-making
Client routing
DR
```

which is a healthy architecture boundary.

---

# 51. PostgreSQL Native vs External HA Responsibilities

### PostgreSQL provides

```text
Streaming replication
Synchronous replication
Standby recovery
Promotion
WAL
Replication slots
pg_rewind
Monitoring views
```

### External HA layer typically provides

```text
Health checking
Leader election
Quorum
Automatic failover
Fencing
Endpoint management
Proxy configuration
Cluster-state coordination
```

This distinction is critical.

---

# 52. Important `pg_rewind` Design Point

`pg_rewind` is not the failover tool itself.

It is mainly a **post-failover reintegration tool**.

Workflow:

```text
Failover
   │
   ▼
New Primary
   │
   ▼
Old Primary Returns
   │
   ▼
pg_rewind
   │
   ▼
Old Primary becomes Standby
```

PostgreSQL explicitly documents this as a typical use case. ([PostgreSQL][5])

---

# 53. PostgreSQL 18 Failover-Slot Improvement

PostgreSQL 18 includes continued improvements around logical replication failover support. For example, the PostgreSQL 18 release notes include `pg_recvlogical --enable-failover` support for failover slots. ([PostgreSQL][6])

This is particularly relevant when designing:

```text
Logical replication
+
HA
```

because logical replication state must also survive failover scenarios.

---

# 54. HA Architecture Checklist

| Area                   | Validation                        |
| ---------------------- | --------------------------------- |
| RPO                    | Defined                           |
| RTO                    | Defined                           |
| Primary                | Monitored                         |
| Standby                | Healthy                           |
| Replication mode       | Sync/async selected intentionally |
| Replication lag        | Alerted                           |
| Synchronous standbys   | Redundancy planned                |
| Failure detection      | Configured                        |
| Quorum                 | Designed                          |
| Fencing                | Tested                            |
| Split-brain prevention | Tested                            |
| Promotion              | Tested                            |
| Client routing         | Automatic or documented           |
| Old-primary handling   | Defined                           |
| `pg_rewind`            | Tested where used                 |
| Backups                | Independent of HA                 |
| DR                     | Separate strategy                 |
| Monitoring             | Complete                          |
| Runbook                | Documented                        |
| Failover drills        | Scheduled                         |

---

# 55. Core Design Principles

For PostgreSQL HA, remember these rules:

```text
Replication is not HA by itself.

HA is not backup.

Failover without fencing is dangerous.

Lower RPO usually costs latency or infrastructure.

Lower RTO requires automation.

Automatic failover requires strong quorum logic.

A standby must be monitored, not assumed healthy.

Client routing is part of HA.

Old-primary reintegration is part of HA.

Failover must be tested regularly.
```

---

## PostgreSQL 18 Official References

The main official reference is **PostgreSQL 18 — Chapter 26: High Availability, Load Balancing, and Replication**, which covers standby servers, replication approaches, synchronous replication, failover, and HA design concepts. ([PostgreSQL][1])

For synchronous replication configuration, use **PostgreSQL 18 — Replication Settings**, especially `synchronous_standby_names`. PostgreSQL documents that only connected, real-time streaming standbys in that configuration can serve as active synchronous standbys. ([PostgreSQL][2])

For standby behavior, cascading replication, and synchronous standby selection, use **PostgreSQL 18 — Log-Shipping Standby Servers**. PostgreSQL specifically recommends configuring enough potential synchronous standbys to maintain the requested synchronous set through failures. ([PostgreSQL][3])

For post-failover reintegration, use **PostgreSQL 18 — `pg_rewind`**, which is designed to synchronize diverged clusters and commonly allows an old primary to return as a standby after failover. ([PostgreSQL][5])

For the PostgreSQL 18-specific logical replication failover improvements, see the **PostgreSQL 18 Release Notes**. ([PostgreSQL][6])

**Tracker: 6 / 29 completed — 23 pending.**

**Next: Topic 7 — PostgreSQL HA/DR Tools & Ecosystem**, covering Patroni, etcd/Consul, HAProxy, PgBouncer, pgBackRest, Barman, monitoring components, how they fit together, and recommended production reference architectures.

[1]: https://www.postgresql.org/docs/current/high-availability.html?utm_source=chatgpt.com "18: Chapter 26. High Availability, Load Balancing, and ..."
[2]: https://www.postgresql.org/docs/current/runtime-config-replication.html?utm_source=chatgpt.com "Documentation: 18: 19.6. Replication"
[3]: https://www.postgresql.org/docs/current/warm-standby.html?utm_source=chatgpt.com "Documentation: 18: 26.2. Log-Shipping Standby Servers"
[4]: https://www.postgresql.org/docs/current/runtime-config-wal.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.5. Write Ahead Log"
[5]: https://www.postgresql.org/docs/current/app-pgrewind.html?utm_source=chatgpt.com "Documentation: 18: pg_rewind"
[6]: https://www.postgresql.org/docs/current/release-18.html?utm_source=chatgpt.com "Documentation: 18: E.5. Release 18"
