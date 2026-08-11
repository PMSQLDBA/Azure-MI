# Lesson 147 — Deadlocks: Detection, Graphs, Root Causes & Prevention

Today we're covering one of the most important SQL Server concurrency topics:

> **Deadlocks are not the same thing as ordinary blocking.**

A deadlock occurs when two or more sessions are waiting for resources held by each other, creating a cycle where none can proceed.

---

## 1. Blocking vs deadlocking

### Blocking

```text
Session A
   ↓
holds lock

Session B
   ↓
waits for A
```

A can eventually finish, release the lock, and B continues.

### Deadlock

```text
Session A
   ↓
holds Resource 1
   ↓
waits for Resource 2

Session B
   ↓
holds Resource 2
   ↓
waits for Resource 1
```

Neither can continue.

---

# 2. The classic deadlock

Imagine two tables:

```text
Customers
Orders
```

Session A:

```sql
UPDATE Customers
SET Name = 'Alice'
WHERE CustomerId = 1;
```

Then:

```sql
UPDATE Orders
SET Status = 'Open'
WHERE CustomerId = 1;
```

Session B does the reverse:

```sql
UPDATE Orders
SET Status = 'Closed'
WHERE CustomerId = 1;
```

Then:

```sql
UPDATE Customers
SET Name = 'Bob'
WHERE CustomerId = 1;
```

Now:

```text
A holds Customers
A wants Orders

B holds Orders
B wants Customers
```

Cycle.

---

# 3. What SQL Server does

SQL Server has deadlock detection.

When it detects a deadlock, it chooses one session as the **deadlock victim**.

That transaction is rolled back so the other transaction can continue.

---

# 4. Error 1205

The victim typically receives an error similar to:

```text
Transaction (Process ID ...)
was deadlocked on lock resources...
```

The important SQL Server error number is:

```text id="d9x3k7"
1205
```

Applications should be prepared to handle deadlock victims appropriately.

---

# 5. Deadlock does NOT mean SQL Server is broken

Deadlocks can occur in correctly functioning systems.

They are often caused by application transaction behavior.

The database engine is doing its job by detecting the cycle and breaking it.

The goal is to:

> **Reduce unnecessary deadlocks and handle unavoidable ones safely.**

---

# 6. The deadlock graph

The most valuable diagnostic artifact is the **deadlock graph**.

Conceptually:

```text
       Session A
          │
          │ holds
          ↓
      Resource 1
          │
          │ requested by
          ↓
       Session B
          │
          │ holds
          ↓
      Resource 2
          │
          │ requested by
          ↓
       Session A
```

That graph shows the cycle.

---

# 7. What the graph tells you

A deadlock graph can reveal:

```text id="x5m2q8"
Victim
Sessions
Transactions
Resources
Locks held
Locks requested
Statements
Database objects
Indexes
Execution context
```

This is far more useful than simply knowing:

> "We got error 1205."

---

# 8. Extended Events

A modern way to capture deadlocks is Extended Events.

SQL Server provides deadlock information through events such as:

```text id="p7c3n9"
xml_deadlock_report
```

This can capture the deadlock graph for analysis.

---

# 9. Why Query Store alone isn't enough

Query Store is excellent for:

```text id="m4r8x1"
Query performance
Plans
Duration
CPU
Reads
Regressions
```

But deadlocks are fundamentally about:

```text id="n6k2v5"
Concurrency
Locks
Resources
Transaction ordering
```

So combine Query Store with:

```text id="q9x3m7"
Extended Events
Lock information
Transaction analysis
```

---

# 10. The most common deadlock pattern

One of the simplest patterns is:

```text id="u8p4c2"
Transaction A:
Table A → Table B

Transaction B:
Table B → Table A
```

Different resource acquisition order creates the cycle.

---

# 11. Consistent ordering

A powerful prevention technique is:

> **Access shared resources in a consistent order.**

For example, always:

```text id="a6m1x9"
Customers
   ↓
Orders
```

instead of allowing some code paths to do:

```text id="r2c8k5"
Customers → Orders
```

and others:

```text id="v7n3p4"
Orders → Customers
```

Consistency reduces the possibility of circular waits.

---

# 12. Transaction scope

Large transactions hold locks longer.

Consider:

```text id="w5q9m2"
BEGIN TRAN

Update 1
Update 2
Update 3
Update 4
Update 5

COMMIT
```

If this takes 30 seconds, locks may remain held for much longer than necessary.

---

# 13. Short transactions

Generally:

```text id="k3x8p1"
Begin
 ↓
Required work
 ↓
Commit
```

should be preferred over:

```text id="n6m4r7"
Begin
 ↓
Lots of unrelated processing
 ↓
Network call
 ↓
Application logic
 ↓
More processing
 ↓
Commit
```

The latter can unnecessarily extend lock duration.

---

# 14. Never hold transactions across external work unnecessarily

A dangerous pattern is:

```text id="q8c2v5"
BEGIN TRAN

UPDATE database

Call external API
   ↓
wait 5 seconds

More database work

COMMIT
```

During that external call, locks may remain held.

Now other sessions can become blocked or participate in deadlocks.

---

# 15. Application transaction design matters

Deadlocks often aren't caused by a single SQL statement.

They can emerge from:

```text id="m7p1x4"
Transaction A
+
Transaction B
+
Different execution order
+
Concurrency
```

So the fix may belong in application code rather than only in SQL.

---

# 16. Isolation level matters

Different isolation levels produce different locking/versioning behavior.

For example:

```text id="c4r8n2"
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
SNAPSHOT
```

Higher locking isolation can increase concurrency pressure in some workloads.

---

# 17. SERIALIZABLE

SERIALIZABLE provides strong isolation but can hold more restrictive locks for longer.

This can increase:

```text id="p5x9m3"
Blocking
Deadlock risk
Concurrency limitations
```

It isn't "bad," but it should be used intentionally.

---

# 18. Snapshot-based isolation

Snapshot-based approaches can reduce reader/writer blocking by using row versions.

This can improve concurrency.

But as we discussed with TempDB:

```text id="u2k6r8"
Versioning
   ↓
TempDB usage
```

So every concurrency strategy has trade-offs.

---

# 19. Deadlocks aren't only table locks

Deadlocks can involve:

```text id="e9v4q1"
Rows
Keys
Pages
Tables
Metadata
Application locks
Other resources
```

The deadlock graph tells you what actually happened.

---

# 20. Indexes can influence deadlocks

Suppose one query accesses rows through:

```text id="j3m8x6"
Index A
```

while another accesses them through:

```text id="r5p1n7"
Index B
```

Different access paths can produce different lock acquisition patterns.

Index design can therefore affect deadlock behavior.

---

# 21. Large scans can make concurrency worse

Suppose a query scans a huge amount of data while holding locks.

That can increase the opportunity for another transaction to interact with those resources.

Better indexing can sometimes reduce:

```text id="v8c4m2"
Rows touched
Lock duration
Lock footprint
```

and therefore reduce deadlock probability.

---

# 22. But don't add indexes just for deadlocks

Again:

> **Diagnose first.**

You need to know:

```text id="n6q2x9"
Which resources?
Which statements?
Which access paths?
Which transaction order?
```

Then decide whether indexing is relevant.

---

# 23. Deadlock victim selection

SQL Server chooses a victim based on deadlock resolution considerations, including deadlock priority and rollback cost.

You can influence priority with:

```sql id="s4r7m1"
SET DEADLOCK_PRIORITY ...
```

For example, applications can mark less important work as a lower-priority victim candidate.

---

# 24. DEADLOCK_PRIORITY

Conceptually:

```text id="a7k3p8"
Critical transaction
    ↓
HIGH priority

Background cleanup
    ↓
LOW priority
```

If a deadlock occurs, SQL Server can favor the lower-priority transaction as the victim.

Use this deliberately.

---

# 25. Deadlock retry

Applications should generally be prepared for deadlock victim errors.

A robust pattern is:

```text id="q5m9x2"
Execute transaction
      ↓
Deadlock 1205?
      ↓
Retry after short delay
      ↓
Try again
```

But retries need safeguards.

---

# 26. Don't retry forever

Bad:

```text id="u3x8n6"
while error:
    retry forever
```

Better:

```text id="r6p1k4"
Retry 1
Retry 2
Retry 3
Then fail gracefully
```

Use an appropriate backoff strategy.

---

# 27. Exponential backoff

If multiple clients immediately retry:

```text id="j8c4m2"
Deadlock
 ↓
Everyone retries instantly
 ↓
Same contention
 ↓
Another deadlock
```

Backoff helps spread retries.

Conceptually:

```text id="v1n7q5"
Retry 1 → short delay
Retry 2 → longer delay
Retry 3 → longer delay
```

Add jitter when appropriate so clients don't synchronize their retries.

---

# 28. Idempotency matters

Retries are only safe when the operation is designed to tolerate retry behavior.

For example:

```text id="p4x9m6"
"Send $100"
```

cannot blindly be executed twice.

Application-level idempotency is therefore important for transactional retry logic.

---

# 29. Deadlock vs timeout

These are different.

### Deadlock

SQL Server detects a cycle and chooses a victim.

### Lock timeout

A session waits too long and the timeout occurs.

Conceptually:

```text id="z2c8r5"
Deadlock
→ cycle detected

Timeout
→ waited too long
```

Don't confuse the two.

---

# 30. Deadlock vs blocking

Also:

```text id="m7q3x1"
Blocking
→ someone is waiting

Deadlock
→ circular waiting
```

A system can have heavy blocking without having a single deadlock.

---

# 31. A practical deadlock investigation

Suppose production reports:

> "Users occasionally get transaction deadlock errors."

Start with:

```text id="k9r4p2"
1. Capture deadlock graph
2. Identify victim
3. Identify other participants
4. Identify resources
5. Identify SQL statements
6. Determine lock order
7. Determine transaction boundaries
8. Find why both transactions overlap
9. Fix ordering/scope/access pattern
10. Add safe retry handling
```

---

# 32. Example deadlock graph

Imagine:

```text id="b6x1m8"
Transaction A
  Holds: Customer 10
  Wants: Order 500

Transaction B
  Holds: Order 500
  Wants: Customer 10
```

This immediately tells you:

```text id="c3n7q5"
A → Customer → Order
B → Order → Customer
```

The likely structural fix is consistent ordering.

---

# 33. Lock duration

Deadlock probability is influenced by how long resources remain locked.

Suppose:

```text id="w4p8m1"
Transaction A
lock acquired
      ↓
5 seconds
      ↓
commit
```

Another transaction has a large window to collide.

If you reduce the transaction to:

```text id="s7x2c9"
lock acquired
 ↓
100 ms
 ↓
commit
```

the collision opportunity decreases dramatically.

---

# 34. Lock footprint

Another useful concept is:

> **How many resources does the transaction touch?**

Compare:

```text id="h5n1q8"
Transaction A
→ 10 rows
```

with:

```text id="j2m6v4"
Transaction B
→ 10 million rows
```

The second has a much larger concurrency footprint.

---

# 35. SARGability and deadlocks

This connects directly to Lesson 145.

A non-SARGable query may scan a large structure instead of efficiently locating a small set of rows.

That can increase:

```text id="p8q3m7"
Rows touched
Duration
Lock duration
Concurrency pressure
```

So query tuning can sometimes be concurrency tuning.

---

# 36. Covering indexes and deadlocks

A covering index can reduce the need for additional access to the base table.

Potentially:

```text id="x5v9c2"
Less I/O
↓
Shorter execution
↓
Shorter lock duration
↓
Lower contention
```

Again, this isn't guaranteed, but it's an important connection.

---

# 37. Deadlocks and foreign keys

Foreign-key relationships can participate in locking behavior.

Updates/deletes involving parent and child rows may require multiple resources.

This makes consistent transaction ordering particularly important.

---

# 38. Deadlocks and triggers

Triggers can make transactions touch additional tables that aren't obvious from the original statement.

For example:

```sql id="m3r8x1"
UPDATE Orders ...
```

might trigger:

```text id="k7p2v5"
Audit table update
```

Now the transaction's resource footprint is larger than the original statement suggests.

Always inspect triggers when investigating unexpected locking behavior.

---

# 39. Deadlocks and cascading actions

Cascading foreign-key actions can similarly expand the set of affected resources.

For example:

```text id="q1n6c9"
DELETE parent
    ↓
DELETE child rows
    ↓
More locks
```

This can influence concurrency.

---

# 40. Deadlock prevention principles

Remember these:

```text id="v8m3r6"
1. Keep transactions short.
2. Access resources in consistent order.
3. Use appropriate indexes.
4. Avoid unnecessary work inside transactions.
5. Avoid unnecessary high isolation.
6. Keep data access patterns predictable.
7. Implement safe retries.
```

---

# 41. What NOT to do

Don't respond to deadlocks by immediately:

```text id="a5q9x2"
Increasing timeout
```

A longer timeout doesn't eliminate a deadlock.

The deadlock detector will still identify the cycle.

---

# 42. Don't simply add NOLOCK

Another common reaction:

> "Use NOLOCK everywhere."

That's not a deadlock solution.

It changes read semantics and can introduce:

```text id="r7c1m4"
Dirty reads
Missing rows
Duplicate rows
Inconsistent results
```

and doesn't solve writer-writer deadlocks.

---

# 43. Don't blindly change isolation

Changing isolation level without understanding the workload can simply move the problem.

For example:

```text id="u6p2k8"
Less locking
→ potentially more version-store usage
```

or:

```text id="n4x9c3"
Stronger isolation
→ potentially more blocking
```

Concurrency design requires trade-off analysis.

---

# 44. Extended Events + application logs

The best investigation often combines:

```text id="m8q5v1"
Deadlock graph
+
Application request ID
+
Transaction context
+
Query Store
```

This lets you connect:

```text id="j3r7x9"
Database deadlock
        ↓
Application operation
        ↓
Specific code path
```

---

# 45. Interview question

### "What is a deadlock?"

Strong answer:

> "A deadlock occurs when two or more transactions form a circular wait, where each holds resources needed by another transaction. SQL Server detects the cycle and chooses a victim transaction to roll back."

---

# 46. Interview question

### "How do you prevent deadlocks?"

Strong answer:

> "I first capture and analyze the deadlock graph. Common prevention techniques include consistent resource access order, shorter transactions, reducing unnecessary work inside transactions, appropriate indexing, and appropriate isolation levels. I also make the application resilient by safely retrying deadlock victims."

---

# 47. Interview question

### "How is a deadlock different from blocking?"

Strong answer:

> "Blocking is a dependency where one session waits for another. A deadlock is a cycle of dependencies where the sessions cannot make progress without one of them being terminated."

---

# 48. Interview question

### "Should an application retry deadlocks?"

Strong answer:

> "Yes, when the transaction is safely retryable. Deadlocks can occur transiently even in well-designed systems, so applications should typically handle error 1205 with a bounded retry and backoff strategy. The retry must be designed to be safe and idempotent."

---

# 49. Expert mental model

Think:

```text
Transaction A
    │
    │ holds
    ↓
 Resource 1
    │
    │ waits
    ↓
 Resource 2
    ↑
    │ holds
    │
Transaction B
```

Then ask:

```text
Why did A acquire Resource 1 first?
Why did B acquire Resource 2 first?
Why were they overlapping?
Why did they hold the locks so long?
```

Those questions lead you toward the root cause.

---

# 50. Lesson 147 takeaway

Remember:

```text
BLOCKING
A waits for B

DEADLOCK
A waits for B
B waits for A
```

And the troubleshooting chain:

```text
Deadlock
   ↓
Capture graph
   ↓
Identify resources
   ↓
Identify statements
   ↓
Identify transaction order
   ↓
Reduce overlap / standardize order
   ↓
Add safe retry
   ↓
Monitor
```

> **The goal isn't merely to make deadlocks disappear. The goal is to make the system's concurrency behavior predictable, efficient, and resilient.**

---
