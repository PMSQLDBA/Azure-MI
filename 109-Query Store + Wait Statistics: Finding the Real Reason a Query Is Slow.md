# Lesson 142 — Query Store + Wait Statistics: Finding the Real Reason a Query Is Slow

Today we connect two powerful ideas:

> **Query Store tells you what happened over time.**
> **Wait statistics help explain what SQL Server was waiting for.**

This combination is extremely useful because a query being slow does **not** automatically mean its execution plan is bad.

---

## 1. The first question: CPU or waiting?

Suppose a query takes:

```text
Duration = 20 seconds
```

That number alone tells you very little.

Now compare:

### Case A

```text
CPU      = 19 seconds
Duration = 20 seconds
```

Likely CPU-heavy.

### Case B

```text
CPU      = 200 ms
Duration = 20 seconds
```

The query spent most of its lifetime waiting.

That distinction changes your investigation.

---

# 2. The basic timing relationship

Conceptually:

```text
Elapsed Time
    =
CPU time
+
waiting time
```

Not every scenario maps perfectly to a simple subtraction because of parallelism and other accounting details, but as a troubleshooting model it is extremely useful.

Think:

```text
CPU
+
WAIT
=
ELAPSED
```

---

# 3. Why waits matter

SQL Server frequently has to wait for something:

```text
CPU
I/O
Locks
Memory
Parallel workers
Network/client consumption
Other resources
```

Waiting isn't automatically bad.

The important question is:

> **What is the query waiting for, and is that wait abnormal for this workload?**

---

# 4. Query Store gives historical context

Suppose:

```text
Monday:
Duration = 200 ms

Tuesday:
Duration = 220 ms

Wednesday:
Duration = 8 sec
```

Query Store can help show the regression.

Now suppose historical wait information shows:

```text
Monday:
Mostly CPU

Wednesday:
Large increase in lock-related waits
```

Now you have a strong direction for the investigation.

---

# 5. A slow query isn't necessarily a bad query

This is one of the most important DBA concepts.

Imagine:

```text
Query execution:
100 ms CPU

+
30 sec blocked by another transaction
```

The query itself may be perfectly reasonable.

The problem is:

```text
Concurrency / blocking
```

not necessarily:

```text
Query plan
```

---

# 6. CPU-bound workload

Suppose Query Store shows:

```text
Duration = 10 sec
CPU     = 9.5 sec
```

Now investigate:

```text
Expensive operators
Large scans
Complex expressions
Poor join strategies
Excessive parallelism
High concurrency
CPU pressure
```

The execution plan becomes especially important.

---

# 7. I/O-bound workload

Suppose:

```text
Duration = 15 sec
CPU      = 300 ms
```

and the query performs enormous amounts of data access.

Possible causes:

```text
Large scans
Poor indexing
Storage latency
Buffer-cache misses
Inefficient joins
```

Now inspect reads and I/O-related waits.

---

# 8. Blocking-bound workload

Suppose:

```text
Duration = 30 sec
CPU      = 100 ms
```

and the query spends most of its time waiting for locks.

You might discover:

```text
Transaction A
    ↓
holds lock
    ↓
Transaction B
    ↓
waiting
    ↓
Application request becomes slow
```

Adding an index to Transaction B may not solve the fundamental problem.

---

# 9. Memory-related waits

Memory pressure can manifest through waits and query behavior.

You may see:

```text
Large memory grants
Grant contention
Spills
Poor concurrency
```

A query that requests an unnecessarily large memory grant can affect not only itself but other queries.

---

# 10. Parallelism waits

Parallel execution introduces coordination.

Conceptually:

```text
Worker 1 ─┐
Worker 2 ─┤
Worker 3 ─┼→ Exchange → next operator
Worker 4 ─┘
```

Workers may have to wait for one another.

This doesn't mean:

> "Parallelism is bad."

Parallelism is often exactly what SQL Server should use.

The question is whether the parallel plan is appropriate.

---

# 11. Parallelism isn't automatically the culprit

A common beginner mistake:

> "I see a parallel plan, therefore parallelism is causing the problem."

Not necessarily.

A large analytical query may legitimately need parallel execution.

Instead ask:

```text
Why did SQL Server choose parallelism?
Is the cost estimate reasonable?
Is CPU pressure high?
Are workers spending excessive time coordinating?
```

---

# 12. Query Store + plan comparison

Suppose:

```text
Old Plan:
CPU = 500 ms
Duration = 600 ms

New Plan:
CPU = 500 ms
Duration = 10 sec
```

The CPU stayed similar.

But elapsed time exploded.

That is a clue that the query may now be spending more time waiting.

---

# 13. Another example

Suppose:

```text
Old Plan:
Reads = 100,000
CPU = 400 ms
Duration = 500 ms

New Plan:
Reads = 10,000,000
CPU = 8 sec
Duration = 9 sec
```

Now you have a very different situation.

The new plan is doing substantially more work.

This points more strongly toward:

```text
Plan regression
+
Excessive data access
```

---

# 14. The four numbers to remember

When investigating a query, start with:

```text
1. Executions
2. Duration
3. CPU
4. Logical reads
```

Then add:

```text
5. Wait behavior
6. Plan
7. Estimates
```

This gives you a much stronger picture.

---

# 15. Example diagnostic table

| Metric       | Before | After |
| ------------ | -----: | ----: |
| Executions   |   100K |  100K |
| Avg duration | 100 ms | 5 sec |
| CPU          |  80 ms | 90 ms |
| Reads        |    20K |   20K |
| Waits        |    Low |  High |

This strongly suggests:

> The query is not suddenly doing dramatically more CPU or logical work. Something external may be making it wait.

---

# 16. Blocking investigation

If blocking is suspected, move from historical Query Store data to **current blocking information**.

Ask:

```text
Who is blocking?
Who is being blocked?
What resource is locked?
How long has the transaction been open?
What query owns the blocking transaction?
```

---

# 17. The blocking chain

Imagine:

```text
Session 51
   ↓ blocks
Session 72
   ↓ blocks
Session 84
   ↓ blocks
Session 91
```

Killing Session 91 won't necessarily fix the root problem.

You need to understand the chain.

---

# 18. Root blocker

The important question is often:

> **Which session is holding the lock that starts the chain?**

That session may be:

```text
Long transaction
Uncommitted update
Poor application transaction handling
Batch operation
Unexpected idle transaction
```

---

# 19. Why transaction design matters

A query can be perfectly optimized but still perform terribly if an application does:

```text
BEGIN TRANSACTION

UPDATE ...

-- waits for user/application work

COMMIT
```

The locks may remain held far longer than necessary.

That creates database-wide consequences.

---

# 20. Long transactions

A transaction should generally hold locks for no longer than necessary.

Bad pattern:

```text
BEGIN
  DB work
  external API call
  user interaction
  application processing
COMMIT
```

This can create serious concurrency problems.

---

# 21. Better pattern

Prefer minimizing the transaction's critical section:

```text
Prepare data
   ↓
BEGIN
   ↓
Required DB changes
   ↓
COMMIT
   ↓
External work
```

Exact design depends on the application's consistency requirements, but the principle is important.

---

# 22. I/O investigation

If I/O is suspected, examine:

```text
Logical reads
Physical reads where available
Storage latency
Data access pattern
Indexes
Scans
Lookups
```

Remember:

> Logical reads and physical I/O are not the same thing.

---

# 23. Logical reads

Logical reads represent pages accessed from the buffer pool.

A query can have:

```text
10 million logical reads
```

even when much of the data is already cached.

That still represents substantial database work.

---

# 24. Physical reads

Physical reads involve obtaining data from storage.

A workload may therefore be:

```text
High logical reads
Low physical reads
```

if the working set is cached.

Or:

```text
High logical reads
High physical reads
```

if data isn't available in memory.

---

# 25. Why reducing logical reads matters

Reducing unnecessary reads can improve:

```text
CPU
Memory bandwidth
Buffer-pool pressure
I/O
Concurrency
```

That's why an index that reduces millions of page accesses can have benefits beyond raw disk I/O.

---

# 26. Wait statistics must be interpreted carefully

Never use:

> "Wait X is high, therefore fix X."

Instead ask:

```text
What workload generated the wait?
When did it increase?
Which queries are associated with it?
Is it expected?
Is it actually limiting performance?
```

Context matters.

---

# 27. Background waits

Some waits are normal or background-related.

For example, a server can accumulate waits that aren't directly responsible for a user-visible performance problem.

Therefore:

```text
High wait count
≠
Root cause
```

---

# 28. Waits and workload context

Suppose a database is heavily used for reporting.

Some parallelism-related waits may naturally occur.

That doesn't mean:

```text
Parallelism = bug
```

You need to compare the waits against:

```text
CPU
query duration
workload characteristics
historical behavior
```

---

# 29. Historical wait changes are more useful

Suppose:

```text
Last month:
Lock waits = low

Today:
Lock waits = extremely high
```

That change is much more interesting than simply seeing:

```text
Lock waits exist
```

The trend is the evidence.

---

# 30. Regression timeline

A strong investigation might look like:

```text
12:00
Deployment

12:10
Query plan changes

12:15
Duration increases

12:20
Lock waits increase

12:30
Users report slowness
```

Now you have a timeline.

The next task is to determine whether those events are causally related.

---

# 31. Query Store doesn't explain everything

Query Store is not a magical root-cause engine.

It can show:

```text
Historical query behavior
Plans
Performance changes
```

But root-cause analysis often requires combining it with:

```text
Execution plans
DMVs
Wait statistics
Blocking information
OS metrics
Application telemetry
```

---

# 32. Think in layers

A useful model:

```text
Application
     ↓
SQL query
     ↓
Execution plan
     ↓
Operators
     ↓
Resources
     ↓
CPU / memory / I/O / locks
     ↓
Hardware / infrastructure
```

A problem can originate at any layer.

---

# 33. Example: application timeout

Suppose the application reports:

```text
Timeout after 30 seconds
```

Don't conclude:

> "The query takes 30 seconds."

Maybe:

```text
Query executes in 200 ms
+
Connection pool delay
+
Blocking
+
Application processing
```

The timeout is an application-level symptom.

---

# 34. Distinguish execution time from end-to-end latency

These are different:

```text
Database execution time
```

versus:

```text
User-perceived request latency
```

A mature performance investigation follows the entire path.

---

# 35. Query Store and application telemetry

If your application has request tracing, you can correlate:

```text
Request ID
   ↓
Endpoint
   ↓
SQL query
   ↓
Query Store
   ↓
Plan
   ↓
Runtime behavior
```

This creates an extremely powerful observability chain.

---

# 36. Example: "Only some users are slow"

Suppose:

```text
User A → 100 ms
User B → 100 ms
User C → 30 sec
```

This pattern suggests investigating:

```text
Parameter sensitivity
Data skew
Blocking
Tenant-specific workload
Different execution paths
```

A global database problem is less likely.

---

# 37. Example: "Everyone is slow"

If every user suddenly becomes slow:

```text
CPU pressure
Storage pressure
Memory pressure
Blocking
Infrastructure
Network
Deployment
```

become more interesting.

---

# 38. Query Store and workload shape

Sometimes the query itself hasn't changed.

The workload has.

For example:

```text
Normal:
10,000 executions/hour

Peak:
500,000 executions/hour
```

A query that is perfectly acceptable at low volume can become a major bottleneck at high volume.

---

# 39. Concurrency amplification

Suppose one query costs:

```text
50 ms CPU
```

At:

```text
10 executions/sec
```

that's manageable.

At:

```text
1,000 executions/sec
```

the same query can consume enormous CPU.

This is why:

> **Execution cost × execution frequency = workload impact**

---

# 40. Query Store helps expose this

You can identify:

```text
Low per-execution cost
+
Huge execution count
=
High total workload cost
```

These queries are often overlooked.

---

# 41. A common DBA mistake

Finding the query with the highest average duration and immediately optimizing it.

That can be wrong.

The biggest production opportunity may instead be:

```text
A 2-ms query
executed 50 million times
```

Always consider total workload impact.

---

# 42. Another common mistake

Seeing high CPU and immediately adding indexes.

CPU could be high because of:

```text
Bad plan
Large scans
Excessive expression evaluation
Too many executions
Parallelism
Application workload explosion
```

Measure first.

---

# 43. Another common mistake

Seeing high logical reads and adding indexes everywhere.

Indexes have costs:

```text
Storage
Writes
Maintenance
Logging
Optimization complexity
```

The right index is the one that solves a demonstrated workload problem.

---

# 44. A disciplined diagnosis

Use this decision tree:

```text
Query slow?
    ↓
Is CPU high?
    ├─ Yes → inspect plan/work
    └─ No
         ↓
     Are waits high?
         ├─ Yes → identify wait category
         └─ No → investigate broader latency
```

Then:

```text
If plan changed:
    → compare plans

If plan didn't change:
    → investigate environment/concurrency/workload
```

---

# 45. Practical case study

Suppose Query Store reports:

```text
Query:
GetInventory

Before:
Duration = 200 ms
CPU = 180 ms
```

After:

```text
Duration = 12 sec
CPU = 190 ms
```

The CPU barely changed.

But duration increased massively.

You investigate waits and discover heavy locking.

---

# 46. Finding the actual issue

You find:

```text
Batch process
   ↓
opens transaction
   ↓
updates millions of rows
   ↓
holds locks
   ↓
application queries wait
```

The application query isn't necessarily poorly optimized.

The batch transaction is creating the contention.

---

# 47. The fix

Possible approaches could include:

```text
Smaller batches
Shorter transactions
Better indexing
Improved transaction design
Appropriate isolation strategy
Scheduling changes
```

The correct solution depends on the workload.

---

# 48. Why plan tuning wouldn't solve this

Imagine optimizing the blocked query from:

```text
200 ms CPU
```

to:

```text
100 ms CPU
```

If it still waits:

```text
12 sec → 12 sec
```

The user may see almost no improvement.

This demonstrates why root-cause analysis matters.

---

# 49. Another case study: CPU regression

Query Store:

```text
Before:
CPU = 200 ms
Duration = 250 ms

After:
CPU = 8 sec
Duration = 9 sec
```

Plan changed:

```text
Index Seek
→
Index Scan
```

Logical reads increased:

```text
10,000
→
8,000,000
```

Now the evidence strongly points toward a plan/access-path regression.

---

# 50. The fix path

Investigate:

```text
Statistics
Indexes
Cardinality estimates
Parameter sensitivity
Query changes
Compatibility level
```

Then implement a targeted fix.

Afterward, Query Store confirms whether performance recovered.

---

# 51. Three categories of slow queries

A useful classification:

### Type 1 — Doing too much work

```text
High CPU
High reads
Bad/expensive plan
```

### Type 2 — Waiting for resources

```text
Low CPU
High duration
Significant waits
```

### Type 3 — Executed too often

```text
Low cost each time
Huge execution count
Large total workload
```

This classification makes troubleshooting faster.

---

# 52. Expert troubleshooting checklist

When a query is reported as slow:

```text
□ What is the duration?
□ What is the CPU?
□ How many executions?
□ How many logical reads?
□ Did the plan change?
□ Did estimates change?
□ What are the major waits?
□ Is there blocking?
□ Did workload volume change?
□ Did data volume change?
□ Did statistics change?
□ Did indexes change?
□ Did the application change?
□ Did compatibility level change?
```

---

# 53. The most important distinction

Always separate:

> **"The query is expensive."**

from:

> **"The query is waiting."**

These are not the same problem.

---

# 54. Lesson 142 takeaway

Your mental model should now be:

```text
                QUERY STORE
                    ↓
            Historical behavior
                    ↓
          ┌─────────┴─────────┐
          ↓                   ↓
       PLAN                  WAITS
          ↓                   ↓
     What work?          Waiting for what?
          ↓                   ↓
          └─────────┬─────────┘
                    ↓
              ROOT CAUSE
                    ↓
                  FIX
                    ↓
                VERIFY
```

### Golden rule

> **A slow query does not automatically mean a bad execution plan.**

Use Query Store to identify **what changed**, and wait/resource information to understand **why the elapsed time increased**.

---
**Next:** **Lesson 143**
**Final:** **Lesson 150**
