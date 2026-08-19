# Topic 16 — PostgreSQL Transactions & Concurrency

This topic is critical for PostgreSQL architecture and production troubleshooting. The main idea is simple:

> **Many users can read and modify the same database at the same time, while PostgreSQL tries to keep the data correct and minimize unnecessary blocking.**

The key concepts are **ACID, MVCC, transactions, snapshots, isolation levels, locks, blocking, deadlocks, and long-running transactions**.

---

## 1. What is a Transaction?

A transaction is a group of database operations treated as one logical unit.

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 500
WHERE account_id = 101;

UPDATE accounts
SET balance = balance + 500
WHERE account_id = 202;

COMMIT;
```

Think of this as a bank transfer.

Either **both updates succeed**, or neither should remain.

If something goes wrong:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 500
WHERE account_id = 101;

-- Something failed

ROLLBACK;
```

`ROLLBACK` cancels the changes made by the transaction.

---

# 2. ACID Properties

PostgreSQL transactions follow the **ACID** model.

| Property    | Meaning                                            | Example                              |
| ----------- | -------------------------------------------------- | ------------------------------------ |
| Atomicity   | All or nothing                                     | Both sides of a transfer succeed     |
| Consistency | Data remains valid                                 | Constraints continue to hold         |
| Isolation   | Concurrent transactions don't improperly interfere | Two users updating data concurrently |
| Durability  | Committed data survives failures                   | `COMMIT` persists the transaction    |

### Atomicity

Suppose:

```text
Account A = $1,000
Account B = $2,000
```

Transfer $500 from A → B.

PostgreSQL should never permanently leave the system at:

```text
A = $500
B = $2,000
```

The transaction either becomes:

```text
A = $500
B = $2,500
```

or rolls back to:

```text
A = $1,000
B = $2,000
```

---

# 3. Why Concurrency Is Difficult

Imagine two sessions working simultaneously.

Initial value:

```text
product quantity = 10
```

Session A:

```sql
UPDATE inventory
SET quantity = quantity - 3
WHERE product_id = 100;
```

Session B simultaneously runs:

```sql
UPDATE inventory
SET quantity = quantity - 4
WHERE product_id = 100;
```

PostgreSQL must coordinate these operations.

It needs to answer questions such as:

* What data can each transaction see?
* Can both transactions update the same row?
* Should one transaction wait?
* When should locks be released?
* What happens if transactions wait for each other?

This is where **MVCC + snapshots + isolation + locking** work together.

---

# 4. MVCC — Multi-Version Concurrency Control

MVCC is one of the most important PostgreSQL concepts.

**MVCC = Multi-Version Concurrency Control.**

Instead of immediately overwriting an existing row, PostgreSQL can maintain different row versions.

Conceptually:

```text
Original row
employee_id = 10
salary      = 80000

Transaction A updates it

Old version:
salary = 80000

New version:
salary = 90000
```

Different transactions may temporarily see different versions depending on their snapshots.

This is a major reason PostgreSQL can provide high concurrency.

### Key principle

> **Readers generally don't block writers, and writers generally don't block readers.**

But:

> **Writers can block other writers when they modify the same rows.**

This distinction is extremely important.

---

# 5. MVCC Example

Suppose:

```sql
SELECT salary
FROM employees
WHERE employee_id = 10;
```

returns:

```text
80000
```

Transaction A starts:

```sql
BEGIN;

UPDATE employees
SET salary = 90000
WHERE employee_id = 10;
```

but doesn't commit.

At the same time Transaction B runs:

```sql
SELECT salary
FROM employees
WHERE employee_id = 10;
```

Transaction B doesn't normally have to wait for Transaction A's update.

It can continue seeing the appropriate previously committed row version:

```text
80000
```

After Transaction A commits, later snapshots may see:

```text
90000
```

This is fundamentally different from systems/designs where readers must wait behind writers.

---

# 6. PostgreSQL Tuple Metadata

Internally, PostgreSQL row versions—tuples—contain transaction visibility information.

Two important system columns are conceptually:

```text
xmin
xmax
```

`xmin` identifies the transaction that created the tuple version.

`xmax` can identify the transaction that deleted it or made that version obsolete.

You can inspect them:

```sql
SELECT xmin, xmax, *
FROM employees;
```

Simplified example:

```text
xmin | xmax | id | salary
-----+------+----+-------
100  | 105  | 10 | 80000
105  | 0    | 10 | 90000
```

Conceptually:

```text
Transaction 100
    |
    v
salary = 80000
    |
 UPDATE
    |
    v
salary = 90000
    ^
    |
Transaction 105
```

The exact visibility logic is more sophisticated, but `xmin/xmax` are useful for understanding PostgreSQL's MVCC model.

---

# 7. What Is a Snapshot?

A **snapshot** determines which row versions a transaction or statement is allowed to see.

Think of it as:

> "From my transaction's point of view, which database changes are visible right now?"

For example:

```text
T1 starts
|
| sees salary = 80K
|
T2 updates salary → 90K
|
T2 commits
|
T1 reads again
```

Whether T1 sees `80K` or `90K` depends on its isolation level and when PostgreSQL takes the relevant snapshot.

---

# 8. PostgreSQL Isolation Levels

PostgreSQL supports:

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

But there is an important PostgreSQL-specific detail:

**READ UNCOMMITTED behaves like READ COMMITTED.**

So practically, focus on:

```text
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

---

# 9. READ COMMITTED

This is PostgreSQL's default isolation level.

```sql
BEGIN;

SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

The important behavior is:

> **Each statement gets a new snapshot.**

Example:

Transaction A:

```sql
BEGIN;

SELECT salary
FROM employees
WHERE id = 10;
```

Result:

```text
80000
```

Transaction B:

```sql
UPDATE employees
SET salary = 90000
WHERE id = 10;

COMMIT;
```

Transaction A runs the query again:

```sql
SELECT salary
FROM employees
WHERE id = 10;
```

It can now return:

```text
90000
```

So within the same transaction:

```text
SELECT #1 → 80000

another transaction commits

SELECT #2 → 90000
```

That is expected under `READ COMMITTED`.

---

# 10. REPEATABLE READ

Start with:

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

Now Transaction A reads:

```sql
SELECT salary
FROM employees
WHERE id = 10;
```

Result:

```text
80000
```

Transaction B changes it:

```sql
UPDATE employees
SET salary = 90000
WHERE id = 10;

COMMIT;
```

Transaction A queries again:

```sql
SELECT salary
FROM employees
WHERE id = 10;
```

It still sees:

```text
80000
```

Conceptually:

```text
Transaction A snapshot
        |
        v
+------------------------+
| Database as visible    |
| when snapshot was made |
+------------------------+

Meanwhile:

Transaction B
80000 → 90000 → COMMIT

Transaction A continues
seeing its older snapshot.
```

This gives a much more stable view of the database.

---

# 11. SERIALIZABLE

Serializable provides the strongest isolation.

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

PostgreSQL attempts to make concurrent transactions behave **as though they had executed one after another**.

Example:

```text
T1 ────────┐
           ├── concurrent
T2 ────────┘
```

PostgreSQL tries to ensure the outcome is equivalent to something like:

```text
T1 → T2
```

or:

```text
T2 → T1
```

If PostgreSQL cannot safely serialize the transactions, one transaction may fail with a serialization error.

Applications therefore need retry logic.

Conceptually:

```text
BEGIN
   |
business operations
   |
COMMIT
   |
serialization failure?
   |
   +--- YES ---> retry whole transaction
   |
   +--- NO ----> success
```

---

# 12. Isolation-Level Comparison

| Isolation       | Snapshot behavior       | Concurrent changes may appear? | Typical use                |
| --------------- | ----------------------- | -----------------------------: | -------------------------- |
| Read Committed  | Per statement           |                            Yes | Normal OLTP                |
| Repeatable Read | Stable transaction view |                   Generally no | Reports / consistent reads |
| Serializable    | Strongest isolation     |                     Controlled | Critical business rules    |

`READ COMMITTED` is the normal PostgreSQL default.

Don't automatically switch everything to `SERIALIZABLE`. Stronger isolation has concurrency and retry implications.

---

# 13. Locks

MVCC does **not** mean PostgreSQL doesn't use locks.

PostgreSQL uses many types of locks.

At a high level:

```text
PostgreSQL locking
      |
      +--- Table-level locks
      |
      +--- Row-level locks
      |
      +--- Advisory locks
      |
      +--- Internal/lightweight synchronization
```

For application/database troubleshooting, table and row locking are especially important.

---

# 14. Row-Level Locking

Suppose Transaction A runs:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE account_id = 1;
```

PostgreSQL must protect that row from conflicting modifications.

Transaction B then runs:

```sql
UPDATE accounts
SET balance = balance - 200
WHERE account_id = 1;
```

Transaction B may have to wait.

Conceptually:

```text
Transaction A
UPDATE row 1
    |
    +---- holds row lock
              |
              v
Transaction B
UPDATE row 1
    |
    +---- WAITING
```

When A executes:

```sql
COMMIT;
```

B can continue.

This is **blocking**.

---

# 15. SELECT FOR UPDATE

Sometimes the application wants to explicitly lock rows before changing them.

```sql
BEGIN;

SELECT *
FROM accounts
WHERE account_id = 1
FOR UPDATE;
```

Now another transaction trying to perform a conflicting modification/lock on that row may need to wait.

Example workflow:

```sql
BEGIN;

SELECT balance
FROM accounts
WHERE account_id = 1
FOR UPDATE;

-- application validates balance

UPDATE accounts
SET balance = balance - 500
WHERE account_id = 1;

COMMIT;
```

Useful when implementing logic like:

```text
Read
  ↓
Validate
  ↓
Modify
```

where another transaction must not modify the selected row in between.

---

# 16. FOR UPDATE vs FOR SHARE

Common row-locking clauses include:

```sql
FOR UPDATE
FOR NO KEY UPDATE
FOR SHARE
FOR KEY SHARE
```

They have different conflict strengths.

For architecture interviews, know that PostgreSQL supports **multiple row-lock modes rather than simply "locked/unlocked."**

You should then be able to explain why:

```sql
SELECT ... FOR UPDATE
```

is commonly used when the application intends to modify selected rows.

---

# 17. NOWAIT

Sometimes waiting is undesirable.

```sql
SELECT *
FROM jobs
WHERE id = 100
FOR UPDATE NOWAIT;
```

If another transaction already holds a conflicting lock, PostgreSQL returns an error rather than waiting.

Conceptually:

```text
Normal:
lock unavailable → wait → wait → continue

NOWAIT:
lock unavailable → error immediately
```

Useful for systems where fast failure is better than unpredictable waiting.

---

# 18. SKIP LOCKED

`SKIP LOCKED` is extremely useful for worker/queue patterns.

Suppose a table contains jobs:

```text
job_id | status
-------+---------
1      | READY
2      | READY
3      | READY
4      | READY
```

Worker A:

```sql
SELECT *
FROM jobs
WHERE status = 'READY'
ORDER BY job_id
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

It locks Job 1.

Worker B runs the same query.

Instead of waiting for Job 1, it can skip the locked row and take Job 2.

```text
Worker A → Job 1
Worker B → Job 2
Worker C → Job 3
Worker D → Job 4
```

This can be very useful for implementing concurrent database-backed work queues.

---

# 19. Table-Level Locks

PostgreSQL also has table-level lock modes.

For example:

```sql
LOCK TABLE orders IN ACCESS EXCLUSIVE MODE;
```

`ACCESS EXCLUSIVE` is a very strong table lock.

Operations such as DDL can require stronger locks than normal queries.

Example:

```sql
ALTER TABLE orders
ADD COLUMN source text;
```

In production, this matters because a seemingly simple schema change may need to acquire a table lock.

If another transaction is holding a conflicting lock:

```text
Application transaction
        |
        | holding lock
        |
ALTER TABLE
        |
        +---- waiting
```

And sessions behind that operation can potentially create a blocking chain.

---

# 20. Blocking

Blocking means:

> One database session cannot proceed because another transaction holds a conflicting lock.

Example:

Session A:

```sql
BEGIN;

UPDATE customers
SET status = 'ACTIVE'
WHERE id = 100;
```

Session A then remains open.

Session B:

```sql
UPDATE customers
SET status = 'INACTIVE'
WHERE id = 100;
```

Session B waits.

The major operational question is:

> **Who is blocking whom?**

---

# 21. Detecting Blocking

A very useful PostgreSQL function is:

```sql
pg_blocking_pids(pid)
```

For example:

```sql
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    pg_blocking_pids(pid) AS blocking_pids,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

This helps identify blocked sessions and their blockers.

You can also inspect locks using:

```sql
SELECT *
FROM pg_locks;
```

In real production troubleshooting, you commonly combine:

```text
pg_stat_activity
pg_locks
pg_blocking_pids()
```

---

# 22. Wait Events

`pg_stat_activity` exposes useful fields:

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity;
```

Example:

```text
pid   state   wait_event_type   wait_event
----  ------  ----------------  -----------
501   active  Lock              transactionid
```

This tells you that the backend is waiting on something lock-related.

Don't assume every slow query is CPU or disk related.

Sometimes:

```text
Query runtime = 30 seconds
```

actually means:

```text
29 seconds waiting for lock
1 second executing
```

---

# 23. Blocking Chain

Production incidents often look more like this:

```text
Session A
   |
   | blocks
   v
Session B
   |
   | blocks
   v
Session C
   |
   | blocks
   v
Session D
```

The most important session may therefore not be the slow query at the bottom.

The real problem can be **Session A**, the root blocker.

Example:

```text
Session A:
idle in transaction
holding lock

        ↓

Session B:
UPDATE waiting

        ↓

Session C:
ALTER TABLE waiting

        ↓

many application sessions waiting
```

This is why finding the **root blocker** is critical.

---

# 24. What Is a Deadlock?

Blocking and deadlocking are different.

Blocking:

```text
A holds resource
B waits for A
```

That can resolve when A commits.

Deadlock:

```text
Transaction A
holds Row 1
waits for Row 2

Transaction B
holds Row 2
waits for Row 1
```

Diagram:

```text
        waits for
   A --------------> B
   ^                 |
   |                 |
   +-----------------+
       waits for
```

Neither can continue.

PostgreSQL detects the deadlock and aborts one transaction.

---

# 25. Deadlock Example

Transaction A:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

-- later

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;
```

Transaction B:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 50
WHERE id = 2;

-- later

UPDATE accounts
SET balance = balance + 50
WHERE id = 1;
```

Sequence:

```text
T1 locks row 1

T2 locks row 2

T1 wants row 2
→ waits for T2

T2 wants row 1
→ waits for T1

DEADLOCK
```

PostgreSQL eventually breaks the cycle by aborting one participant.

---

# 26. Preventing Deadlocks

One of the best techniques is **consistent lock ordering**.

Bad pattern:

```text
Transaction A: lock 1 → lock 2
Transaction B: lock 2 → lock 1
```

Better:

```text
Transaction A: lock 1 → lock 2
Transaction B: lock 1 → lock 2
```

For example, always process accounts by ascending ID.

```sql
SELECT *
FROM accounts
WHERE id IN (1, 2)
ORDER BY id
FOR UPDATE;
```

Other good practices:

* Keep transactions short.
* Avoid unnecessary user/network interaction inside transactions.
* Access resources in consistent order.
* Index queries properly so operations don't touch unnecessary rows.
* Build retry logic for deadlocks.
* Monitor lock waits.

---

# 27. Blocking vs Deadlock

| Blocking                          | Deadlock                                      |
| --------------------------------- | --------------------------------------------- |
| One session waits                 | Transactions wait in a cycle                  |
| Can be normal                     | Cannot resolve naturally without intervention |
| Blocker may eventually commit     | PostgreSQL aborts a participant               |
| Often caused by long transactions | Often caused by inconsistent lock ordering    |

Important interview statement:

> **Every deadlock involves blocking, but not every blocking situation is a deadlock.**

---

# 28. Long-Running Transactions

Long-running transactions are dangerous in PostgreSQL.

Example:

```sql
BEGIN;

SELECT * FROM orders;

-- application does nothing for 2 hours

COMMIT;
```

Even if the transaction isn't consuming CPU, keeping an old transaction/snapshot around can create operational problems.

Why?

Because MVCC needs to preserve row versions that may still be visible to active transactions.

Conceptually:

```text
Old transaction snapshot
        |
        v
------------------------------------------------>
Database keeps changing

UPDATE
UPDATE
DELETE
UPDATE
DELETE

Old row versions
may not yet be removable
```

---

# 29. MVCC and Dead Tuples

Suppose:

```sql
UPDATE employees
SET salary = 90000
WHERE id = 10;
```

Conceptually PostgreSQL creates a new tuple version.

```text
Old:
id=10 salary=80000

New:
id=10 salary=90000
```

The old version eventually becomes unnecessary.

PostgreSQL's `VACUUM` machinery can reclaim/reuse space associated with obsolete tuples when it is safe.

But an old transaction may still have a snapshot that could potentially need old versions.

Therefore:

```text
Long-running transaction
        ↓
Old snapshot remains relevant
        ↓
VACUUM cleanup horizon held back
        ↓
Dead tuples accumulate
        ↓
Table/index bloat pressure
```

This is one reason transaction age matters greatly in PostgreSQL.

---

# 30. Idle in Transaction

A particularly bad state is:

```text
idle in transaction
```

Example application behavior:

```sql
BEGIN;

UPDATE orders
SET status = 'PROCESSING'
WHERE id = 100;
```

Then the application waits for something:

```text
API call
user input
network response
bug
sleep
```

without executing:

```sql
COMMIT;
```

or:

```sql
ROLLBACK;
```

The transaction remains open.

Depending on what it did, it can retain locks and/or an old transaction horizon.

Find such sessions:

```sql
SELECT
    pid,
    usename,
    xact_start,
    state,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

This should be part of production monitoring.

---

# 31. Finding Long-Running Transactions

Useful query:

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    xact_start,
    now() - xact_start AS transaction_age,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

You are looking for transactions whose age is unexpectedly high.

Example:

```text
PID   Transaction Age
----  ---------------
101   00:00:02
102   00:00:05
103   03:47:12   <-- investigate
```

---

# 32. `statement_timeout`

You can prevent statements from running indefinitely.

```sql
SET statement_timeout = '30s';
```

A statement exceeding the configured time can be cancelled.

This is useful, but remember:

```text
statement timeout
≠
transaction timeout
```

An application can still leave transactions open through bad transaction handling.

---

# 33. `lock_timeout`

You can also limit how long PostgreSQL waits to acquire a lock.

```sql
SET lock_timeout = '5s';
```

Example:

```text
ALTER TABLE
    |
waiting for lock
    |
5 seconds
    |
ERROR
```

This can be particularly valuable for controlled production deployments where you would rather fail a migration than allow it to wait indefinitely and contribute to a queue of blocked sessions.

---

# 34. `idle_in_transaction_session_timeout`

PostgreSQL provides a useful protection:

```sql
SET idle_in_transaction_session_timeout = '5min';
```

This helps terminate sessions that sit idle while a transaction remains open beyond the configured threshold.

It's a valuable guardrail against poorly behaved applications.

---

# 35. Transaction IDs and Wraparound

PostgreSQL uses transaction IDs as part of MVCC.

Transaction IDs are finite and eventually wrap around.

This is why vacuuming isn't merely:

> "Cleaning unused space."

It also has a critical role in preventing **transaction ID wraparound problems**.

This is where concepts such as:

```text
VACUUM
autovacuum
FREEZE
transaction age
```

become operationally important.

A PostgreSQL architect should never casually disable autovacuum globally.

---

# 36. Transaction Lifecycle

Think about a normal transaction as:

```text
             BEGIN
               |
               v
        Obtain snapshot
               |
               v
           Execute SQL
               |
       +-------+-------+
       |               |
       v               v
   Read rows       Modify rows
       |               |
     MVCC            Locks
       |               |
       +-------+-------+
               |
               v
        COMMIT / ROLLBACK
               |
               v
       Release resources
```

This connects the major concepts.

---

# 37. How MVCC and Locks Work Together

A common misunderstanding is:

> "PostgreSQL uses MVCC, therefore PostgreSQL doesn't need locks."

Wrong.

Think of it this way:

```text
MVCC
 |
 +--- determines what row versions you can SEE

Locks
 |
 +--- coordinate conflicting OPERATIONS
```

Example:

```text
Reader vs Writer
    ↓
often MVCC allows concurrency

Writer vs same Writer target
    ↓
locks coordinate access
```

That distinction is important.

---

# 38. Production Scenario

Suppose users report:

```text
Checkout API normally: 200 ms
Current latency:       25 seconds
```

CPU:

```text
20%
```

Disk:

```text
normal
```

Connections:

```text
high
```

You inspect:

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    pg_blocking_pids(pid),
    query
FROM pg_stat_activity;
```

You discover:

```text
PID 100
idle in transaction

PID 200
waiting for PID 100

PID 300
waiting for PID 200

PID 400
waiting for PID 200
```

Root cause:

```text
Application bug
      ↓
transaction left open
      ↓
locks retained
      ↓
other transactions blocked
      ↓
connection pool fills
      ↓
API latency explodes
```

This is a classic production PostgreSQL incident.

---

# 39. Architecture Fix

Don't solve it by simply increasing:

```text
max_connections
```

That can make the situation worse.

Fix the actual transaction behavior:

```text
Application
   |
   +-- transactions short
   |
   +-- always commit/rollback
   |
   +-- retry deadlocks
   |
   +-- retry serialization failures where appropriate
   |
   +-- consistent locking order
   |
   +-- sensible timeouts
   |
   +-- monitor transaction age
   |
   +-- monitor blocking
```

Then configure guardrails such as:

```text
statement_timeout
lock_timeout
idle_in_transaction_session_timeout
```

according to workload requirements.

---

# 40. Interview Scenario

### Question

**"Our PostgreSQL application suddenly has hundreds of active sessions, but CPU usage is only 25%. What would you check?"**

A strong answer:

> I would first determine whether those sessions are actually executing or waiting.
> I would inspect `pg_stat_activity`, including `wait_event_type` and `wait_event`, and check blocking relationships with `pg_blocking_pids()`.
> If sessions are waiting on locks, I would identify the root blocker rather than killing random blocked sessions.
> I would also look for old transactions and sessions that are `idle in transaction`, because they may hold locks or old snapshots.
> After resolving the immediate blocker, I would investigate the application transaction boundaries and add suitable transaction and lock timeout protections.

That answer demonstrates much more PostgreSQL understanding than:

> "I would check slow queries."

---

# 41. Another Interview Scenario

### Question

**"Why doesn't an UPDATE normally block SELECT in PostgreSQL?"**

Good answer:

> PostgreSQL uses MVCC. An UPDATE creates a new row version, while existing transactions can continue reading the appropriate previously committed version according to their snapshot.

Therefore normal readers generally don't need to wait for writers. However, two transactions attempting conflicting modifications to the same row can block because PostgreSQL still uses locks for write coordination.

---

# 42. Another Interview Scenario

### Question

**"What's the danger of a long-running transaction?"**

Good answer:

> A long-running transaction can retain an old snapshot or transaction horizon, preventing PostgreSQL from cleaning up row versions that may still be visible to it. This can interfere with vacuum cleanup, contribute to dead tuple accumulation and bloat, and increase transaction-ID-related operational risk. If the transaction has modified data, it may also hold locks for a long time and cause blocking.

---

# 43. Another Interview Scenario

### Question

**"How would you troubleshoot a deadlock?"**

Good approach:

```text
1. Inspect PostgreSQL logs
        ↓
2. Identify participating transactions
        ↓
3. Identify resources locked
        ↓
4. Determine lock acquisition order
        ↓
5. Find inconsistent ordering
        ↓
6. Change application transaction logic
        ↓
7. Add transaction retry handling
```

The long-term solution usually isn't:

```text
increase deadlock timeout
```

Instead, fix the transaction access pattern.

---

# 44. Practical Monitoring Queries

### Active transactions

```sql
SELECT
    pid,
    usename,
    state,
    now() - xact_start AS xact_age,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

### Idle transactions

```sql
SELECT
    pid,
    usename,
    now() - xact_start AS xact_age,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

### Blocked sessions

```sql
SELECT
    pid,
    pg_blocking_pids(pid) AS blockers,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

### Lock overview

```sql
SELECT
    locktype,
    database,
    relation,
    mode,
    granted,
    pid
FROM pg_locks
ORDER BY pid;
```

---

# 45. Mental Model to Remember

Use this mental model:

```text
                PostgreSQL Concurrency
                        |
          +-------------+-------------+
          |                           |
         MVCC                       Locks
          |                           |
       Versions                  Conflicts
          |                           |
      Snapshots                     Wait
          |                           |
   Isolation Levels              Blocking
                                      |
                                   Deadlock
```

And separately:

```text
Long Transaction
       |
       +----> old snapshot
       |         |
       |         +----> VACUUM cleanup delayed
       |                    |
       |                    +----> bloat pressure
       |
       +----> locks may remain
                 |
                 +----> blocking
                           |
                           +----> application latency
```

---

# 46. Key Architecture Takeaways

For production PostgreSQL, remember these points:

1. **PostgreSQL uses MVCC for concurrency.**
2. **Readers generally don't block writers, and writers generally don't block readers.**
3. **Conflicting writers can block each other.**
4. **Snapshots determine which tuple versions a transaction can see.**
5. **READ COMMITTED is PostgreSQL's default isolation level.**
6. **READ COMMITTED normally uses a new snapshot per statement.**
7. **REPEATABLE READ gives a stable transaction view.**
8. **SERIALIZABLE provides the strongest isolation but applications must handle serialization failures.**
9. **MVCC does not eliminate locks.**
10. **`SELECT ... FOR UPDATE` explicitly locks selected rows for conflicting operations.**
11. **`NOWAIT` fails instead of waiting for a conflicting lock.**
12. **`SKIP LOCKED` is useful for concurrent worker/queue designs.**
13. **Blocking and deadlocks are different.**
14. **Consistent lock ordering is one of the best ways to prevent deadlocks.**
15. **Long-running transactions are dangerous in PostgreSQL.**
16. **`idle in transaction` should be monitored carefully.**
17. **Old transactions can prevent efficient cleanup of dead tuples.**
18. **Autovacuum is essential to healthy MVCC operation.**
19. **Use `pg_stat_activity`, `pg_locks`, and `pg_blocking_pids()` for concurrency troubleshooting.**
20. **When production is slow but CPU is low, always consider lock waits and blocking.**

### One-line architecture summary

> **MVCC controls what concurrent transactions can see; isolation controls how stable that view is; locks protect conflicting operations; blocking happens when locks conflict; deadlocks happen when waits form a cycle; and long-running transactions can interfere with both concurrency and MVCC cleanup.**
