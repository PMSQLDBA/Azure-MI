# Lesson 92 — Azure SQL Managed Instance Monitoring & Performance

This is where your DBA skills become very practical.

A production SQL MI can be:

* **Healthy but slow**
* **Healthy but blocked**
* **Healthy but overloaded**
* **Healthy while one query is consuming most resources**
* **Healthy while the application is timing out**

So:

> **"SQL MI is healthy" does not mean "the workload is healthy."**

---

# 1. The performance troubleshooting mindset

When an application says:

> "SQL is slow."

Don't immediately change indexes.

First determine:

```text
Is the problem:
    CPU?
    Memory?
    I/O?
    Blocking?
    Deadlocks?
    Query plan?
    Storage?
    Network?
    Application?
```

---

# 2. The performance stack

Think from the bottom upward:

```text
Azure infrastructure
       ↓
SQL MI resources
       ↓
CPU / memory / I/O
       ↓
SQL waits
       ↓
Queries
       ↓
Execution plans
       ↓
Tables / indexes
       ↓
Application behavior
```

---

# 3. The first question

Always ask:

> **"What changed?"**

For example:

```text
Yesterday:
Query = 2 seconds

Today:
Query = 45 seconds
```

Possible changes:

```text
New deployment
New index
Index removed
Statistics changed
Data volume increased
Query plan changed
More users
Different parameters
Resource pressure
Azure configuration change
```

---

# 4. Baseline

Before troubleshooting an incident, you need to know what **normal** looks like.

Example:

```text
Normal CPU:
40–55%

Normal data I/O:
moderate

Normal average query duration:
800 ms

Normal blocking:
minimal
```

Then today:

```text
CPU:
95%

Query duration:
12 seconds

Blocking:
high
```

Now you have evidence.

---

# 5. What should you monitor?

At minimum:

```text
CPU
Memory
Data I/O
Log I/O
Storage
Connections
Sessions
Requests
Waits
Blocking
Deadlocks
Query duration
Query failures
```

---

# 6. Azure Monitor

Azure Monitor provides a centralized way to observe Azure resources and workloads.

For SQL MI, you can use metrics and diagnostic capabilities to understand resource behavior.

Think:

```text
SQL MI
   |
   ↓
Azure Monitor
   |
   +---- Metrics
   +---- Logs
   +---- Alerts
```

---

# 7. Metrics

Metrics give you time-series measurements.

Examples:

```text
CPU utilization
Storage utilization
IO-related metrics
Connection-related metrics
```

You can look at:

```text
Last 5 minutes
Last hour
Last 24 hours
Last 7 days
```

---

# 8. Why historical graphs matter

Suppose someone says:

> "The database is slow at 2 PM every day."

You inspect CPU:

```text
1 PM   40%
1:30   45%
2 PM   95%
2:30   90%
3 PM   45%
```

That suggests a recurring workload.

Now investigate:

```text
Scheduled jobs
ETL
Reporting
Index maintenance
Batch processing
```

---

# 9. CPU

CPU pressure can be caused by:

```text
Expensive queries
Poor execution plans
Large scans
Sorts
Aggregations
High concurrency
Parallelism
Inefficient application queries
```

But:

> High CPU does not automatically mean "add more vCores."

First identify why CPU is high.

---

# 10. Example

CPU:

```text
98%
```

You find:

```sql id="3b8cvx"
SELECT *
FROM Sales
WHERE YEAR(OrderDate) = 2026;
```

Potential issue:

```text
Function applied to indexed column
```

This can make efficient index usage more difficult.

You may need to rewrite the query or improve indexing.

---

# 11. Memory

Memory pressure can affect:

```text
Buffer cache
Query execution
Sorts
Hash operations
Plan cache
Concurrency
```

A database workload that repeatedly reads the same data benefits from having sufficient memory available for caching.

---

# 12. Don't diagnose memory only from "memory percentage"

You need context.

Ask:

```text id="h1r0lo"
What workload is running?
Are queries waiting for memory?
Is the system paging or under resource pressure?
Is the query doing large sorts/hashes?
```

A high memory utilization number alone isn't enough to prove a memory problem.

---

# 13. I/O

Database workloads frequently depend on storage performance.

Think:

```text
Query
  ↓
Reads data
  ↓
Storage I/O
```

If storage is slow or overloaded:

```text
Query duration ↑
```

---

# 14. Data I/O vs log I/O

Conceptually:

### Data I/O

Reads/writes database data pages.

### Log I/O

Writes transaction-log records.

A workload can be:

```text
Read-heavy
```

or:

```text
Write-heavy
```

or both.

---

# 15. Example

Reporting workload:

```text
SELECT
JOIN
GROUP BY
ORDER BY
```

may generate significant:

```text
Data reads
CPU
Memory usage
```

Transaction processing:

```text
INSERT
UPDATE
DELETE
```

may generate significant:

```text
Log activity
Data writes
Locks
```

---

# 16. Wait statistics

One of the most valuable SQL performance concepts is:

> **Waits tell you what SQL Server is waiting for.**

Imagine a query takes:

```text
30 seconds
```

Instead of simply saying:

> "The query is slow."

ask:

> **"What did it spend those 30 seconds waiting for?"**

---

# 17. Common wait categories

You may encounter waits related to:

```text
CPU scheduling
I/O
Locks
Memory
Parallelism
Networking
```

The exact wait names matter, but the category is often the first clue.

---

# 18. Blocking

Blocking occurs when one session prevents another from acquiring a resource it needs.

Example:

```text id="c7b3k7"
Session 51
    |
    | holds lock
    ↓
Table/row
    ↑
    |
Session 72
    |
    X
Waiting
```

---

# 19. Real-world blocking example

Session 51:

```sql id="bqk7m3"
BEGIN TRANSACTION;

UPDATE Orders
SET Status = 'Processing'
WHERE OrderID = 1001;

-- Transaction remains open
```

Session 72:

```sql id="ozp99b"
UPDATE Orders
SET Status = 'Completed'
WHERE OrderID = 1001;
```

Session 72 waits.

---

# 20. The real problem

People often say:

> "Session 72 is blocked."

But that's not the root cause.

The root cause is:

```text
Session 51
```

holding the resource and not completing its transaction.

So:

> **Find the blocker, not just the victim.**

---

# 21. Blocking chain

A production system can look like:

```text id="m1l7c4"
Session 51
   ↓ blocks
Session 72
   ↓ blocks
Session 83
   ↓ blocks
Session 91
   ↓ blocks
Session 110
```

One poorly behaved transaction can create a large queue.

---

# 22. Blocking troubleshooting

Ask:

```text
Who is blocked?
Who is blocking?
What SQL is running?
How long has the transaction been open?
What resource is locked?
Why hasn't the transaction committed/rolled back?
```

---

# 23. Don't kill sessions blindly

A DBA sees:

```text
Blocking session = 51
```

and immediately:

```sql
KILL 51;
```

Dangerous.

First determine:

```text
What is session 51 doing?
Is it a legitimate business transaction?
Is it stuck?
Is the application holding a transaction open?
Will rollback be large?
```

---

# 24. Rollback can take time

Suppose session 51 modified millions of rows.

You kill it.

Now SQL Server has to undo the transaction.

So:

```text
KILL
  ↓
Rollback
  ↓
Potentially significant duration
```

Don't assume `KILL` means the work disappears instantly.

---

# 25. Deadlocks

Blocking and deadlocks are related but different.

### Blocking

```text
A waits for B
```

### Deadlock

```text
A waits for B
B waits for A
```

Neither can proceed.

---

# 26. Deadlock example

Transaction A:

```text id="xjyd5j"
Locks Customer
    ↓
Wants Order
```

Transaction B:

```text id="g6k4pw"
Locks Order
    ↓
Wants Customer
```

Result:

```text id="8r5x6q"
Customer ← A
   ↑       ↓
   |       |
   B ← Order
```

Circular dependency.

---

# 27. SQL Server resolves deadlocks

SQL Server detects the deadlock and chooses a victim.

Conceptually:

```text id="5j31g4"
Transaction A
Transaction B
      |
      ↓
Deadlock detector
      |
      ↓
Choose victim
      |
      ↓
Rollback victim
      |
      ↓
Other transaction continues
```

---

# 28. Application behavior

The application should handle deadlock errors appropriately.

A retry can be reasonable when the operation is safe to retry.

But blindly retrying everything can make problems worse.

---

# 29. Query Store

One of the most important tools for SQL performance troubleshooting is **Query Store**.

Think of Query Store as:

> **Historical performance memory for queries.**

It can help answer:

```text id="t3d0an"
Which queries are expensive?
Which queries became slower?
Did the execution plan change?
What queries consume CPU?
What queries have high duration?
```

---

# 30. Why Query Store is powerful

Suppose:

```text id="7ld1fb"
Monday:
Query = 1 sec

Tuesday:
Query = 25 sec
```

The query text didn't change.

Query Store can help you investigate whether:

```text id="2h1j1y"
Execution plan changed
```

---

# 31. Query plan regression

This is a classic DBA incident.

Yesterday:

```text id="6kj6t2"
Plan A
Index Seek
1 second
```

Today:

```text id="z4i2xq"
Plan B
Large Scan
30 seconds
```

Same query.

Different plan.

That's called a **plan regression**.

---

# 32. Why can plans change?

Possible reasons include:

```text
Statistics changes
Data distribution changes
Parameter sensitivity
Index changes
Schema changes
Optimizer decisions
```

---

# 33. Query Store investigation

Look for:

```text id="l8axq7"
High CPU queries
High duration queries
High execution count
Queries with plan changes
Queries with increasing duration
```

Then inspect:

```text id="f1tbyq"
Query text
Execution plan
Runtime statistics
Wait information where available
```

---

# 34. Top CPU query

Suppose Query Store shows:

```text id="v5tr3j"
Query A = 65% CPU
Query B = 10%
Query C = 5%
```

Start with Query A.

But ask:

> Is it expensive because it runs once, or because it runs 10 million times?

---

# 35. Total cost vs individual cost

Example:

```text id="4yq0t0"
Query A:
10 seconds × 1 execution
= 10 seconds

Query B:
100 ms × 100,000 executions
= 10,000 seconds
```

Query B is much more important overall.

Therefore monitor:

```text
Average duration
Total duration
Execution count
CPU
Logical reads
```

---

# 36. Query optimization

Suppose you find:

```sql id="o6i7ap"
SELECT *
FROM Orders
WHERE CustomerID = @CustomerID;
```

Possible investigation:

```text id="qefr77"
Is CustomerID indexed?
How many rows match?
Is SELECT * necessary?
What does the execution plan show?
Are statistics current?
```

Don't automatically create an index before examining evidence.

---

# 37. Execution plan

The execution plan tells you how SQL Server intends to execute the query.

You may see:

```text id="2g6qzq"
Index Seek
Index Scan
Table Scan
Nested Loops
Hash Match
Sort
Aggregate
```

Each operator gives you clues.

---

# 38. Seek vs scan

Simplified:

### Seek

```text id="6m4y0r"
Index
 ↓
Find matching rows
```

Often efficient for selective predicates.

### Scan

```text id="yb8q6v"
Read large portion of structure
```

Can be perfectly valid for large result sets.

Important:

> **A scan is not automatically bad.**

---

# 39. Example of a good scan

Query:

```sql id="q74xg1"
SELECT COUNT(*)
FROM Orders;
```

If the query needs almost every row, scanning can be appropriate.

Never use:

> "Scan = bad."

Use:

> "Is this access method appropriate for the workload?"

---

# 40. Statistics

Statistics help the optimizer estimate:

```text id="0frd6y"
How many rows will match?
```

Suppose SQL Server estimates:

```text id="jv0t4x"
10 rows
```

but actual result is:

```text id="bqfdub"
10,000,000 rows
```

That estimation error can lead to a poor execution plan.

---

# 41. Parameter sensitivity

A query may perform differently depending on parameter values.

Example:

```text id="zhd9lr"
Customer A
→ 5 rows

Customer B
→ 5,000,000 rows
```

The same query can need different strategies.

This can produce performance issues related to parameter sensitivity/parameter sniffing.

---

# 42. Don't immediately disable parameter sniffing

Another common mistake:

> "Query is slow. Parameter sniffing!"

Maybe.

But first collect evidence.

Look at:

```text id="f9y30v"
Execution plans
Parameters
Runtime statistics
Row estimates
Actual rows
```

Then decide.

---

# 43. CPU troubleshooting workflow

If CPU is 95%:

```text id="v7e3q0"
1. Confirm sustained CPU pressure.
2. Identify top CPU consumers.
3. Check Query Store.
4. Inspect execution plans.
5. Check recent changes.
6. Check concurrency.
7. Optimize inefficient queries.
8. Consider scaling only after understanding workload.
```

---

# 44. I/O troubleshooting workflow

If I/O is high:

```text id="p5h5n8"
1. Identify read/write pressure.
2. Identify top I/O queries.
3. Check execution plans.
4. Check indexes.
5. Check table size/data growth.
6. Check statistics.
7. Check workload changes.
8. Consider resource scaling if justified.
```

---

# 45. Blocking troubleshooting workflow

If blocking spikes:

```text id="e0ay4d"
1. Find blocked sessions.
2. Find root blocker.
3. Identify SQL.
4. Check transaction duration.
5. Identify application behavior.
6. Determine whether intervention is safe.
7. Fix transaction design.
```

---

# 46. Deadlock troubleshooting workflow

```text id="d3t2k4"
1. Capture deadlock information.
2. Identify participating sessions.
3. Identify resources.
4. Determine lock acquisition order.
5. Standardize access order if possible.
6. Reduce transaction scope.
7. Add appropriate indexes where they reduce lock duration.
8. Implement safe retry logic.
```

---

# 47. Long-running transactions

A transaction should generally be as short as practical.

Bad pattern:

```text id="80ufl4"
BEGIN TRANSACTION

Do lots of application work

Wait for user

Call external service

More SQL

COMMIT
```

This can hold locks unnecessarily.

Better:

```text id="y16i83"
Prepare work
    ↓
BEGIN
    ↓
Required SQL
    ↓
COMMIT
```

---

# 48. TempDB

TempDB is used for temporary database operations.

Workloads involving:

```text id="g5ngq6"
Sorting
Hashing
Temporary tables
Versioning
Intermediate results
```

can put pressure on TempDB.

As a DBA, monitor TempDB-related symptoms when troubleshooting memory/I/O/query performance.

---

# 49. Connection storms

Imagine an application has:

```text id="af6e6c"
100 connections
```

Then a deployment bug causes:

```text id="5i8d2w"
10,000 connections
```

Suddenly:

```text
Connection pressure
CPU pressure
Memory pressure
Worker pressure
```

The database may appear "slow" even though individual queries aren't terrible.

---

# 50. Connection pooling

Applications should generally use appropriate connection pooling.

Bad pattern:

```text id="rvk9y5"
Open connection
Run tiny query
Close
Repeat thousands of times
```

A properly managed pool can reuse connections efficiently.

---

# 51. Alerts

Monitoring isn't useful if nobody notices problems.

Create alerts for meaningful conditions.

For example:

```text id="h0h3f4"
CPU sustained above threshold
Storage approaching capacity
Connection pressure
Critical failure events
```

Don't alert on every minor fluctuation.

---

# 52. Alert fatigue

Suppose you receive:

```text id="9z5g83"
500 alerts/day
```

DBA eventually ignores them.

That's dangerous.

Good alerts should be:

```text id="9x7qki"
Actionable
Relevant
Meaningful
Prioritized
```

---

# 53. Severity

Example:

```text id="2uk8k0"
Critical:
Production database unavailable

High:
Sustained resource exhaustion

Medium:
Storage approaching threshold

Low:
Noncritical performance deviation
```

---

# 54. Azure Monitor + Query Store

These tools complement each other.

```text id="q8k1z0"
Azure Monitor
    |
    ↓
"CPU is high"
    |
    ↓
Query Store
    |
    ↓
"Query X is consuming CPU"
    |
    ↓
Execution plan
    |
    ↓
"Plan changed"
    |
    ↓
Root cause
```

That's a powerful workflow.

---

# 55. DMVs

Dynamic Management Views provide runtime/internal information.

Examples include information about:

```text id="i5jzcf"
Sessions
Requests
Locks
Indexes
Query statistics
Waits
```

For example, you might inspect active requests and blocking relationships through SQL Server DMVs.

---

# 56. Example blocking investigation

Conceptually:

```sql id="b0h9x2"
SELECT
    session_id,
    blocking_session_id,
    status,
    wait_type
FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0;
```

This can help identify blocked requests.

The exact DMV query should be adapted to the SQL MI version and investigation you're performing.

---

# 57. Example active requests

```sql id="qgcrqg"
SELECT
    session_id,
    status,
    command,
    cpu_time,
    total_elapsed_time
FROM sys.dm_exec_requests;
```

This gives you a quick look at active work.

---

# 58. Query Store vs DMV

Think:

### DMV

```text id="m2j1e9"
What is happening now?
```

### Query Store

```text id="m5q6iz"
What has been happening over time?
```

This distinction is very useful.

---

# 59. Incident example

At 2:00 PM:

```text id="whb0ja"
Users report slow checkout.
```

Azure Monitor:

```text id="q3g8um"
CPU = 96%
```

You investigate Query Store:

```text id="wz4o4v"
Query A
CPU = extremely high
```

Execution plan:

```text id="t5y1un"
Large table scan
```

Recent deployment:

```text id="f2n4bg"
Index removed
```

Root cause:

```text id="x4j0pg"
Missing index / deployment regression
```

You restore/correct the appropriate schema change and monitor recovery.

---

# 60. Another incident

CPU:

```text id="v9i6ve"
40%
```

Users report:

```text id="9f7n8y"
Queries timing out.
```

You investigate.

Blocking:

```text id="g3r5t2"
Session 52
   ↓
Blocks 100 sessions
```

Root cause:

```text id="t7w6x4"
Long-running transaction
```

CPU isn't the problem.

This demonstrates why:

> **Never diagnose performance from one metric.**

---

# 61. Another incident

CPU:

```text id="y3k3r6"
50%
```

I/O:

```text id="r0j5fv"
Very high
```

Query:

```text id="9f7u1w"
Large scan
```

Execution plan:

```text id="0h3i2a"
Table scan
```

Now investigate:

```text id="g4w1is"
Indexing
Query predicate
Statistics
Data growth
```

---

# 62. Scaling

Sometimes the workload genuinely needs more resources.

You may consider scaling when:

```text id="0u7j2f"
CPU is consistently saturated
Memory pressure is sustained
I/O requirements exceed capacity
Workload has legitimately grown
```

But:

> **Scaling is not a substitute for fixing a broken query.**

---

# 63. Example

Bad query:

```text id="5n5m4c"
Consumes 80% CPU
```

You scale up.

Now:

```text id="7q0m3w"
CPU = 40%
```

Looks fixed.

But the query is still inefficient.

If the workload doubles:

```text id="l2h4m8"
Problem returns.
```

Fix the root cause where possible.

---

# 64. Performance tuning order

A useful order:

```text id="13n9cx"
1. Confirm problem
2. Establish baseline
3. Identify resource/query
4. Find root cause
5. Optimize workload
6. Re-test
7. Scale if necessary
8. Monitor
```

---

# 65. Don't change five things at once

Suppose you:

```text id="nq4h7z"
Create index
Change query
Update statistics
Scale compute
Change configuration
```

Then performance improves.

Which change fixed it?

You don't know.

Controlled troubleshooting is better:

```text id="r0w3ct"
Change one significant variable
        ↓
Measure
        ↓
Compare
```

---

# 66. Baseline document

A useful production baseline might contain:

```text id="g8c2fs"
CPU:
Average 45%
Peak 70%

Storage:
40%

Average query latency:
500 ms

Top CPU queries:
Q1, Q2, Q3

Blocking:
Rare

Deadlocks:
0–2/day

Connections:
Average 200
Peak 450
```

Now you have something to compare against.

---

# 67. Performance dashboard

A useful DBA dashboard might show:

```text id="h1afm2"
+-----------------------------+
| SQL MI Production           |
+-----------------------------+
| CPU             62%         |
| Storage         58%         |
| Connections     310         |
| Blocking        2 sessions  |
| Deadlocks       0           |
+-----------------------------+
| Top CPU queries             |
| Top duration queries        |
| Top waits                   |
| Recent alerts               |
+-----------------------------+
```

---

# 68. Performance incident checklist

```text id="7ck1f0"
INCIDENT
☐ When did it start?
☐ What changed?
☐ Is it all users or some users?
☐ Is it one query or many?

RESOURCES
☐ CPU
☐ Memory
☐ I/O
☐ Storage
☐ Connections

SQL
☐ Waits
☐ Blocking
☐ Deadlocks
☐ Long transactions
☐ Query Store
☐ Execution plans

APPLICATION
☐ Connection pool
☐ Retry behavior
☐ Recent deployment
☐ Query changes

FIX
☐ Query optimization
☐ Index/statistics
☐ Transaction correction
☐ Scaling if required

VALIDATE
☐ Performance returned to baseline
☐ Application confirmed
☐ Monitoring continues
```

---

# 69. Interview question

### "How do you troubleshoot a slow SQL MI?"

Strong answer:

> "I first establish when the issue started and compare the current workload with the baseline. I check Azure Monitor metrics for CPU, memory, I/O, storage and connections, then use DMVs for current requests, waits and blocking. I use Query Store to identify expensive queries and plan regressions, inspect execution plans, check recent application or schema changes, fix the root cause, and only then consider scaling."

That's a strong DBA answer.

---

# 70. Interview question

### "CPU is 95%. What do you do?"

Don't say:

> "Scale up."

Say:

> "First determine whether the CPU is caused by a specific query, increased workload, plan regression, or concurrency issue. I'd use Query Store, DMVs and Azure Monitor to identify the source before deciding whether query optimization or additional capacity is appropriate."

---

# 71. Interview question

### "What is blocking?"

Answer:

> "Blocking occurs when one session holds a lock or resource that another session needs, causing the second session to wait."

Then add:

> "I identify the root blocking session rather than simply killing the blocked sessions."

---

# 72. Interview question

### "Blocking vs deadlock?"

```text id="f0vtxk"
Blocking:
A → waits for B

Deadlock:
A → waits for B
B → waits for A
```

SQL Server detects deadlocks and chooses a victim.

---

# 73. Interview question

### "What is Query Store?"

Answer:

> "Query Store captures historical query performance and execution-plan information, allowing DBAs to identify expensive queries, performance regressions, and plan changes over time."

---

# 74. Interview question

### "DMVs vs Query Store?"

Excellent short answer:

> "DMVs are especially useful for understanding current runtime activity, while Query Store is valuable for historical query and plan analysis."

---

# 75. Final mental model

When users say:

> **"SQL MI is slow."**

Don't jump to:

```text id="f2t0hz"
Increase CPU
```

Think:

```text
                SQL MI SLOW
                     |
          +----------+----------+
          |          |          |
         CPU       I/O       Blocking
          |          |          |
          ↓          ↓          ↓
       Queries     Reads      Locks
          |          |          |
          +----------+----------+
                     |
                     ↓
                Query Store
                     |
                     ↓
               Execution Plan
                     |
                     ↓
                Root Cause
                     |
                     ↓
             Fix + Validate
```

---

# 76. The DBA golden rule

> **Measure first. Change second.**

And remember:

```text
High CPU ≠ automatically need more CPU

High I/O ≠ automatically need more storage

Slow query ≠ automatically need an index

Blocking ≠ automatically kill the blocker

Healthy SQL MI ≠ healthy application
```

Your job is to identify the **root cause**, not merely silence the symptom.

---

## Next — Lesson 93: Azure SQL Managed Instance Security

We'll cover **Microsoft Entra authentication, SQL authentication, managed identities, RBAC vs database permissions, Microsoft Entra groups, least privilege, encryption, TDE, TLS, Always Encrypted concepts, auditing, Defender for SQL, firewall/network security, secrets, privileged access, and a complete production security incident.**
