# Lesson 63 — Blocking, Deadlocks & Locking Internals in Azure SQL Managed Instance

This is one of the most important SQL Server production topics.

A database can have:

```text
CPU = 20%
Memory = normal
Storage = normal
MI health = healthy
```

…and the application can still be **completely stuck**.

Why?

> **Blocking.**

---

# 1. Real production incident

Imagine this alert at 2:15 AM:

```text
Application response time
       ↓
45 seconds
```

Developers say:

> "SQL MI is slow."

You check:

```text
CPU          18%
IO           25%
Connections  300
MI health    Healthy
```

Nothing looks overloaded.

But users can't complete transactions.

You investigate active requests and discover:

```text
200 sessions
       ↓
WAITING
       ↓
LCK_M_...
```

Now you have a direction:

> **Lock contention.**

---

# 2. What is a lock?

Imagine two transactions accessing the same row.

Transaction A:

```sql id="j0y3f9"
BEGIN TRANSACTION;

UPDATE dbo.Account
SET Balance = Balance - 100
WHERE AccountID = 100;
```

Transaction A modifies the row and keeps its transaction open.

Transaction B tries:

```sql id="x2m7q4"
UPDATE dbo.Account
SET Balance = Balance + 50
WHERE AccountID = 100;
```

Transaction B may have to wait.

Conceptually:

```text id="5w0m8q"
Transaction A
     |
     | holds lock
     v
 AccountID = 100
     ^
     |
     | waiting
     |
Transaction B
```

A is the **blocker**.

B is the **blocked session**.

---

# 3. Blocking isn't automatically a bug

This is important.

SQL Server uses locking to maintain transaction correctness.

Some blocking is normal.

For example:

```text id="5n6q1z"
Transaction A
UPDATE
COMMIT quickly

Transaction B
waits briefly
then continues
```

That's normal database behavior.

The problem is:

```text id="4f8x2p"
Long-running transaction
        ↓
Locks held for a long time
        ↓
Many sessions blocked
```

---

# 4. The head blocker

Imagine:

```text id="a7x2m4"
Session 51
   |
   +---- blocks Session 62
   |
   +---- blocks Session 63
   |
   +---- blocks Session 64
   |
   +---- blocks Session 65
```

Session 51 is the **head blocker**.

You shouldn't blindly kill:

```text id="b9q4v1"
62
63
64
65
```

because those are victims of the blocking chain.

First investigate:

> **Why is Session 51 holding the lock?**

---

# 5. Find the blocker

A useful starting query:

```sql id="q8v1n5"
SELECT
    r.session_id,
    r.blocking_session_id,
    r.status,
    r.command,
    r.wait_type,
    r.wait_time,
    r.wait_resource,
    DB_NAME(r.database_id) AS database_name,
    t.text AS sql_text
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE r.blocking_session_id <> 0
ORDER BY r.wait_time DESC;
```

Suppose you find:

```text id="f7r2k9"
Session 51
Blocking: 62,63,64,65,66...
```

Now investigate session 51.

---

# 6. The real problem

You discover:

```text id="6q1v8h"
Session 51
Application: ReportingApp
Transaction started: 11:02 PM
Current time: 2:20 AM
```

That's suspicious.

The transaction has been open for more than three hours.

Maybe the application did:

```text id="7g3d2w"
BEGIN TRANSACTION
       ↓
UPDATE
       ↓
Process 100,000 records
       ↓
Generate report
       ↓
Call external API
       ↓
Wait
       ↓
COMMIT
```

This is a terrible transaction boundary.

The transaction is holding locks while doing work that doesn't need to happen inside the transaction.

---

# 7. The real fix

Don't just kill the session every night.

Fix the application:

Bad:

```text id="0h3j9m"
BEGIN TRANSACTION
    |
    +-- Update rows
    |
    +-- Generate report
    |
    +-- Call external API
    |
    +-- Send email
    |
COMMIT
```

Better:

```text id="u9s1h5"
BEGIN TRANSACTION
    |
    +-- Required database changes
    |
COMMIT
    |
    +-- Generate report
    +-- Call external API
    +-- Send email
```

The goal:

> **Keep transactions as short as business correctness allows.**

---

# 8. Why long transactions are dangerous

Suppose:

```text id="l4f2w8"
Transaction starts
    ↓
Locks acquired
    ↓
5 seconds
    ↓
10 seconds
    ↓
1 minute
    ↓
10 minutes
```

Meanwhile:

```text id="c3v9k1"
Application requests
        ↓
wait
wait
wait
wait
```

You get a blocking storm.

Eventually:

```text id="e6m1r4"
Connection pool exhausted
        ↓
Application appears frozen
```

The database may still show low CPU.

---

# 9. Blocking chain example

Visualize it like this:

```text id="g7v2n9"
Session 51
   |
   +----> Session 62
   |          |
   |          +----> Session 80
   |
   +----> Session 63
   |
   +----> Session 64
   |
   +----> Session 65
```

One bad transaction can therefore create **hundreds of secondary failures**.

---

# 10. Now: Deadlocks

Blocking is:

```text id="f9x2k7"
A blocks B
```

A deadlock is:

```text id="m4q8s1"
A blocks B
B blocks A
```

For example:

Transaction A:

```text id="q3z8v1"
Lock Customer 1
       ↓
Needs Customer 2
```

Transaction B:

```text id="r7y1k5"
Lock Customer 2
       ↓
Needs Customer 1
```

Now:

```text id="5j2w9x"
A → Customer 1 → needs Customer 2
                         ↑
                         |
B → Customer 2 → needs Customer 1
```

Neither can proceed.

---

# 11. SQL Server detects the deadlock

SQL Server doesn't wait forever.

It detects the cycle and chooses a transaction to terminate.

Conceptually:

```text id="m8n2q4"
Transaction A ─────┐
                   ↓
              DEADLOCK
                   ↑
Transaction B ─────┘
                   |
                   v
             Victim selected
```

The victim receives a deadlock error.

The application should generally be designed to retry appropriate transient deadlock failures.

---

# 12. Why does SQL Server choose a victim?

It considers factors such as the estimated cost of rolling back transactions and configured deadlock priority.

You can influence priority with:

```sql id="p6x1k8"
SET DEADLOCK_PRIORITY LOW;
```

or:

```sql id="w3m7v2"
SET DEADLOCK_PRIORITY HIGH;
```

This doesn't **prevent** the deadlock.

It influences which transaction is more likely to be selected as the victim.

---

# 13. Important interview question

### "How do you prevent deadlocks?"

Don't say:

> "Kill the blocking session."

That's not prevention.

A better answer:

> "I would identify the deadlock graph and determine which resources and statements are involved. Then I'd look for inconsistent object access order, unnecessarily long transactions, excessive locking, missing indexes causing broader scans, and application transaction design problems. A common prevention technique is ensuring transactions acquire resources in a consistent order."

---

# 14. Consistent access order

Suppose Transaction A does:

```text id="9y6p2k"
Customer
   ↓
Order
```

But Transaction B does:

```text id="2x8m4v"
Order
   ↓
Customer
```

This increases deadlock risk.

Make both follow:

```text id="5g1n8c"
Customer
   ↓
Order
```

Now the locking order is consistent.

That can eliminate an entire class of deadlocks.

---

# 15. Indexes can affect locking

Suppose an UPDATE cannot efficiently locate the target rows.

Instead of:

```text id="2j5r8x"
Seek
 ↓
small number of rows
```

it performs:

```text id="v7m3q1"
Large scan
 ↓
many rows examined
 ↓
more locking/concurrency impact
```

Poor indexing can therefore contribute indirectly to concurrency problems.

This is why performance tuning and locking are connected.

---

# 16. Isolation levels

Now we need to understand isolation.

A transaction's isolation level affects what it can see and how concurrency behaves.

Common levels include:

```text id="k5n8r3"
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SNAPSHOT
SERIALIZABLE
```

---

# 17. READ UNCOMMITTED

This allows reading data that hasn't been committed.

That can produce:

> **Dirty reads.**

Example:

```text id="c8f4p2"
Transaction A
UPDATE Balance = 1000
     |
     | not committed
     v
Transaction B
reads 1000
```

Then A rolls back.

B saw data that never actually became committed.

So you shouldn't use this casually just to "remove blocking."

---

# 18. READ COMMITTED

This is the common default isolation level in SQL Server environments.

The key idea:

> A query shouldn't normally read uncommitted changes from another transaction.

Locking behavior and row-versioning behavior can vary depending on database configuration.

This is why you need to understand the actual database settings rather than assuming all SQL Server workloads behave identically.

---

# 19. Snapshot-based isolation

With row versioning, readers can sometimes access an earlier committed version of data instead of waiting for a writer.

Conceptually:

```text id="p3q9v2"
Writer
   |
   v
Current row

Reader
   |
   v
Previous committed version
```

This can significantly reduce reader/writer blocking in suitable workloads.

But row versioning isn't "free."

It has:

```text id="x1f7m4"
Version-store overhead
TempDB considerations
Different concurrency semantics
```

You need to evaluate workload behavior.

---

# 20. The wrong fix

Application team says:

> "SELECT queries are blocked. Let's use NOLOCK everywhere."

Don't blindly do this.

`NOLOCK` / `READ UNCOMMITTED` can allow inconsistent results.

You can potentially observe:

```text id="u4m8z7"
Dirty data
Missing rows
Duplicate-looking rows
Inconsistent results
```

The database appears faster because you're weakening consistency guarantees.

That's not necessarily a successful performance fix.

---

# 21. A better question

Instead of:

> "How do I eliminate blocking?"

Ask:

> **"Which blocking is necessary for correctness, and which blocking is caused by poor transaction/query/application design?"**

That is the senior DBA mindset.

---

# 22. 2 AM incident — complete diagnosis

Let's put everything together.

### Alert

```text id="f6x2w9"
Application latency = 40 seconds
```

### MI resources

```text id="r9p4m1"
CPU = 22%
IO  = 30%
```

### Waits

```text id="d2k8s6"
LCK_M_...
```

### Blocking tree

```text id="n3y7v5"
Session 51
   |
   +--> 62
   +--> 63
   +--> 64
   +--> 65
   +--> 66
```

### Session 51

```text id="h8q2c4"
Transaction age = 2 hours
```

### Application code

```text id="x7m1p9"
BEGIN TRANSACTION

UPDATE Orders

CALL external service

Generate report

COMMIT
```

### Root cause

```text id="a5v8k2"
Transaction boundary too large
```

### Permanent fix

```text id="m9r3q6"
Shorten transaction
       +
Move external work outside transaction
       +
Review indexing/query duration
       +
Add appropriate retry handling
       +
Monitor transaction duration
```

That's a real production diagnosis.

---

# 23. Your senior DBA checklist

When users report "SQL is slow":

```text id="r6n2x8"
☐ Check CPU
☐ Check IO
☐ Check waits
☐ Check blocking
☐ Find head blocker
☐ Check transaction age
☐ Check current SQL
☐ Check Query Store
☐ Check execution plans
☐ Check deadlocks
☐ Check indexes/statistics
☐ Check application transaction design
☐ Fix root cause
☐ Validate
```

Don't jump straight to:

```text id="k4w7p1"
Restart SQL
Kill sessions
Scale up
Add random indexes
Use NOLOCK everywhere
```

Those are actions, not diagnoses.

---

# Lesson 64 — TempDB, Memory Grants & Spill Problems

Next we'll tackle another classic production mystery:

> **"CPU is only 40%, blocking is minimal, but one query suddenly takes 2 minutes and the execution plan shows yellow warning triangles."**

We'll investigate:

* TempDB
* memory grants
* sort/hash spills
* excessive workspace memory
* large intermediate result sets
* why a query can be slow even when CPU isn't saturated
* and how these problems appear in **Azure SQL Managed Instance**.
