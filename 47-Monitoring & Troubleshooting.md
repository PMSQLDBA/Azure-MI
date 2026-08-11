# Lesson 79 — Azure SQL Managed Instance Monitoring & Troubleshooting

This is one of the **most important DBA lessons**.

In real production, people rarely tell you:

> "The query has excessive PAGEIOLATCH waits."

They tell you:

> **"The application is slow."**

Your job is to go from that vague symptom to the actual root cause.

---

# 1. The DBA troubleshooting mindset

Never jump straight to:

> "The database is slow."

Break it down.

```text
Application is slow
       |
       v
Is SQL involved?
       |
       v
Can we connect?
       |
       v
Is SQL MI under resource pressure?
       |
       v
Are queries waiting?
       |
       v
Is there blocking?
       |
       v
Which query?
       |
       v
Why is the query slow?
       |
       v
What changed?
```

That is the basic troubleshooting chain.

---

# 2. The four major resource areas

Start with:

```text id="qvby6y"
             SQL MI
               |
      +--------+--------+
      |        |        |
     CPU      I/O      Memory
      |
    Network / connectivity
```

You need to understand all four.

---

# 3. CPU

High CPU means SQL Server is spending significant processing time executing workloads.

Example:

```text id="jqw8h0"
10:00 AM → CPU 40%
10:10 AM → CPU 50%
10:15 AM → CPU 95%
```

Something changed.

Possible causes:

```text
Bad execution plan
New query
Missing index
Large data increase
Parameter sensitivity
Parallel queries
Poorly written query
Maintenance workload
```

---

# 4. Don't immediately scale the instance

This is a common mistake.

Someone sees:

```text id="8ak3fg"
CPU = 95%
```

and says:

> "Increase the service tier."

Maybe.

But first determine:

> **Why did CPU suddenly increase?**

If one query is consuming 80% of CPU because of a bad plan, scaling might only hide the problem temporarily.

---

# 5. Example

Normal:

```text id="0rv6yc"
CPU = 35%
```

Deployment:

```text id="u3lh2n"
New application version
```

After deployment:

```text id="0r2d1f"
CPU = 95%
```

That's a strong clue.

Look for:

```text id="1n2f84"
New queries
Execution-plan changes
Query frequency changes
Parameter changes
```

---

# 6. I/O

I/O concerns data and log access.

Conceptually:

```text id="e0q7l8"
Query
 ↓
Buffer/cache
 ↓
Storage
```

If required data isn't already available in memory, SQL Server may need to read from storage.

High I/O can result from:

```text id="pl9b6m"
Large scans
Poor indexes
Large sorts
Hash operations
Heavy ETL
Maintenance
Large writes
```

---

# 7. Memory

SQL Server uses memory heavily for:

```text id="p8r2n4"
Data pages
Execution plans
Query workspace
Caches
Internal structures
```

Memory pressure can cause performance degradation.

But remember:

> **High memory utilization doesn't automatically mean a problem.**

SQL Server is designed to use available memory.

You need to determine whether there is actual memory pressure.

---

# 8. Network

Sometimes SQL is healthy but the application experiences network problems.

Example:

```text id="a4k6c9"
Application
      |
      | network latency
      X
      |
     SQL MI
```

Potential causes:

```text
DNS
Routing
VPN
ExpressRoute
Network security
Application-side connection pooling
Network latency
```

So:

> **Application slowness ≠ SQL slowness.**

---

# 9. Wait statistics

This is one of the most powerful SQL Server troubleshooting concepts.

Instead of asking:

> "Why isn't the query running?"

Ask:

> **"What is the query waiting for?"**

Conceptually:

```text id="9b3s1v"
Query
 |
 +-- CPU
 |
 +-- I/O
 |
 +-- Lock
 |
 +-- Memory
 |
 +-- Network
```

The wait tells you where SQL Server is spending time waiting.

---

# 10. Common wait categories

You may encounter waits related to:

```text id="4qz8nt"
CPU / scheduling
I/O
Locks
Memory grants
Parallelism
Log flushes
Network/client consumption
```

Don't memorize wait names without understanding the category.

Start with:

> **What resource is this query waiting for?**

---

# 11. Blocking

This is extremely common.

Imagine:

```text id="m5c8x1"
Transaction A
   |
   | locks row
   v
Customer 1001
```

Then:

```text id="j9q2r7"
Transaction B
   |
   | wants same row
   v
WAIT
```

That's blocking.

---

# 12. Blocking chain

Real systems can become:

```text id="2a6f7c"
Session 51
   ↓ blocks
Session 72
   ↓ blocks
Session 84
   ↓ blocks
Session 95
```

Now you have a blocking chain.

The first question becomes:

> **Who is the head blocker?**

---

# 13. Head blocker

Example:

```text id="z7v4m3"
Session 51
   |
   +---- Session 72
   |
   +---- Session 84
   |
   +---- Session 95
```

Session 51 is the head blocker.

Don't randomly kill:

```text id="f1y8k4"
Session 95
```

because it's merely waiting.

Find the root.

---

# 14. Real scenario

At 10:15 AM:

```text id="8d3gk0"
Application response time
↑
```

You check SQL MI.

CPU:

```text id="w6q9a2"
45%
```

CPU isn't the problem.

I/O:

```text id="2m5xv7"
Normal
```

Memory:

```text id="j8r4n3"
Normal
```

Then you discover:

```text id="n3y6p8"
Heavy blocking
```

Now the investigation changes.

---

# 15. Find the blocker

Suppose:

```text id="x1q7c5"
Session 74
```

has been running for:

```text id="r9m2v6"
47 minutes
```

and holding locks.

Question:

> What is Session 74 doing?

You inspect:

```text id="p4k8z1"
Query text
Database
Login
Transaction start
Application
Host
```

---

# 16. The real root cause

You discover:

```sql id="0j5x8s"
BEGIN TRANSACTION;

UPDATE Orders
SET Status = 'Processing'
WHERE ...
```

Then the application crashed before:

```sql id="8q3m7n"
COMMIT
```

or:

```sql id="7w1z4k"
ROLLBACK
```

Now:

```text id="m8n2v5"
Open transaction
      ↓
Locks
      ↓
Blocking
      ↓
Application slowness
```

That's a real production incident.

---

# 17. Don't kill sessions blindly

You might eventually need to terminate a problematic session.

But first understand:

```text id="q4v9c2"
What transaction is active?
What work has been done?
Will rollback be large?
Who owns the transaction?
What business operation is affected?
```

Killing a session can trigger rollback.

A huge rollback can itself consume resources and take time.

---

# 18. Deadlocks

Blocking:

```text id="b9x4m1"
A → waiting for B
```

Deadlock:

```text id="p7c2n5"
A → waiting for B
B → waiting for A
```

Graphically:

```text id="w2r6k8"
Session A
    |
    | waits
    v
Session B
    |
    | waits
    v
Session A
```

SQL Server detects the cycle and chooses a deadlock victim.

---

# 19. Real deadlock example

Transaction A:

```text id="9h6x3m"
UPDATE Customer
UPDATE Order
```

Transaction B:

```text id="k4n8p2"
UPDATE Order
UPDATE Customer
```

Potential sequence:

```text id="e3c7v1"
A locks Customer
B locks Order

A wants Order
B wants Customer
```

Now:

```text id="j5q2z8"
A ←→ B
```

Deadlock.

---

# 20. How do you solve deadlocks?

Don't simply increase the SQL MI size.

Investigate:

```text id="2n8m4v"
Access order
Transaction length
Indexes
Isolation level
Query design
Application retry logic
```

A common solution is to make transactions access resources in a consistent order.

For example:

```text id="h7r1q5"
Always:
Customer → Order
```

rather than:

```text id="x3m8c2"
Transaction A:
Customer → Order

Transaction B:
Order → Customer
```

---

# 21. Query Store

**Query Store** is one of the most useful tools for SQL performance analysis.

It helps you understand:

```text id="d8n2m5"
Which queries ran?
How often?
How long?
How much CPU?
Which plans were used?
Did performance change?
```

Think:

```text id="p5v7x3"
Query Store
    |
    +-- Query
    +-- Plan
    +-- Runtime statistics
    +-- Historical behavior
```

---

# 22. Why Query Store is powerful

Suppose yesterday:

```text id="6g1p8r"
Query duration = 100 ms
```

Today:

```text id="x4m9q2"
Query duration = 8 seconds
```

Query Store can help you investigate what changed.

For example:

```text id="c7z3n1"
Plan A → 100 ms
Plan B → 8 sec
```

Now you have a strong clue:

> **Plan regression.**

---

# 23. Execution plans

An execution plan shows how SQL Server intends to execute a query.

Example:

```text id="r5w2m8"
SELECT
   ↓
JOIN
   ↓
Index Seek
```

versus:

```text id="v8k3p1"
SELECT
   ↓
JOIN
   ↓
Table Scan
```

A scan isn't automatically bad.

But if you're querying:

```text id="q2m7x9"
10 rows
```

from:

```text id="z6c4n8"
500 million rows
```

a large scan deserves investigation.

---

# 24. Seek vs scan

### Seek

SQL Server can navigate to the relevant data.

```text id="a9r4c7"
Index
 ↓
Target rows
```

### Scan

SQL Server examines a large portion of the structure.

```text id="k1m8p3"
Index/table
 ↓
Many rows/pages
```

The right choice depends on query selectivity, indexing, statistics, and other factors.

---

# 25. Missing index recommendations

SQL Server may identify potential missing indexes.

But don't blindly create every suggested index.

Why?

Each additional index can increase:

```text id="w3n7q2"
Storage
INSERT cost
UPDATE cost
DELETE cost
Maintenance
```

So index recommendations require DBA judgment.

---

# 26. Parameter sensitivity

Suppose:

```sql id="6q9c2p"
SELECT *
FROM Orders
WHERE CustomerID = @CustomerID;
```

For:

```text id="m5x8n1"
CustomerID = 100
```

there are:

```text id="g7p2r4"
5 rows
```

For:

```text id="j9v3k6"
CustomerID = 999
```

there are:

```text id="s2c8m5"
20 million rows
```

One execution plan may not be optimal for both cases.

This can produce sudden performance changes depending on parameters.

---

# 27. Plan regression scenario

Production:

```text id="f4m7q1"
10:00 AM → Query = 150 ms
```

At:

```text id="p8x3n6"
10:05 AM
```

a plan changes.

Now:

```text id="w2k9r5"
Query = 12 seconds
```

Application becomes slow.

Query Store shows:

```text id="c6v1m8"
Old plan → fast
New plan → slow
```

Now investigate:

```text id="y3q7n2"
Statistics
Data distribution
Index changes
Parameter behavior
Plan choice
```

---

# 28. Extended Events

Extended Events are useful for capturing detailed SQL Server events.

They can help investigate:

```text id="5v9m2k"
Deadlocks
Long-running queries
Errors
Specific SQL events
Performance problems
```

Think of them as:

> **A targeted event collection mechanism.**

---

# 29. Don't capture everything forever

A common mistake:

```text id="g2q8p4"
Capture everything
24/7
```

That creates unnecessary overhead and enormous volumes of data.

Instead:

```text id="n7m3x5"
Define incident
 ↓
Capture relevant events
 ↓
Analyze
 ↓
Stop/adjust collection
```

Use targeted diagnostics.

---

# 30. Azure Monitor

Azure Monitor gives you the Azure-level monitoring view.

Think:

```text id="p9x4r6"
Azure Monitor
     |
     +-- Metrics
     +-- Logs
     +-- Alerts
```

You can correlate SQL behavior with other Azure resources.

For example:

```text id="w6m2n8"
Application
   ↓
Network
   ↓
SQL MI
   ↓
Storage
```

---

# 31. Log Analytics

Log Analytics can provide centralized log analysis when configured for the relevant Azure resources and diagnostics.

Conceptually:

```text id="j3q8v1"
SQL MI
 |
 +---- Logs
 |
 +---- Metrics
 |
 +---- Diagnostics
       |
       v
Log Analytics
```

This becomes particularly useful in enterprise environments where many services must be correlated.

---

# 32. Alerts

You can create alerts for meaningful conditions.

Examples:

```text id="m8c2p7"
CPU unusually high
Storage concern
Failed job
Connectivity issue
Performance threshold
```

But again:

> Alert on something actionable.

---

# 33. Real-time incident

Let's simulate a production incident.

### 10:15 AM

Business reports:

> "Orders are taking 30 seconds instead of 1 second."

You are the DBA.

What do you do?

---

# 34. Step 1 — Confirm the symptom

Check:

```text id="q7v1n4"
Is SQL involved?
Is the issue global or one application?
Is it all databases?
Is it one query?
```

Don't assume.

---

# 35. Step 2 — Check SQL MI health/resources

You examine:

```text id="x9m3c8"
CPU
I/O
Memory
Connections
Other relevant platform metrics
```

Suppose:

```text id="b5k7p2"
CPU = 48%
I/O = normal
Memory = normal
```

No obvious platform resource saturation.

---

# 36. Step 3 — Check waits

You discover:

```text id="r2n8v6"
Large amount of lock-related waiting
```

Now investigate blocking.

---

# 37. Step 4 — Find head blocker

You discover:

```text id="k4m9x1"
Session 82
```

has held a transaction for:

```text id="w7p3c5"
31 minutes
```

The application team says:

> "That session belongs to the reporting service."

---

# 38. Step 5 — Inspect the query

You find:

```sql id="z6q2m8"
BEGIN TRANSACTION;

UPDATE Orders
SET Status = 'Pending'
WHERE CustomerID = @CustomerID;
```

The transaction then performs a slow external operation before committing.

That's bad transaction design.

---

# 39. Root cause

```text id="e4v8n2"
Long transaction
       ↓
Locks held too long
       ↓
Blocking
       ↓
Queries wait
       ↓
Application latency
```

Notice:

```text id="a1m6x9"
CPU = normal
```

yet the application is extremely slow.

That's why CPU alone is not enough.

---

# 40. Remediation

Potential remediation:

```text id="t7q3m5"
Shorten transaction
       ↓
Don't hold SQL locks during external calls
       ↓
Commit quickly
       ↓
Move external processing outside transaction
```

Then investigate whether existing blocked sessions need to be handled carefully.

---

# 41. Another incident — CPU spike

Different day:

```text id="p2r9x6"
CPU = 97%
```

You check top queries.

One query suddenly consumes:

```text id="g5m8c2"
65% of CPU
```

Query Store shows:

```text id="z8v4n1"
Plan changed 15 minutes ago
```

Now your investigation becomes:

```text id="n3q7m5"
Why did plan change?
```

Possible causes:

```text
Statistics update
Data distribution
Index change
Parameter behavior
Engine behavior
```

---

# 42. Another incident — I/O spike

Suppose:

```text id="h6x2p9"
CPU = 30%
I/O = extremely high
```

Top query:

```sql id="j7m4q1"
SELECT *
FROM Transactions
WHERE CONVERT(date, TransactionDate) = @Date;
```

Potential issue:

```text id="b3n8v6"
Function applied to column
       ↓
Poor index utilization
       ↓
Large scan
       ↓
I/O pressure
```

The exact fix requires examining the execution plan and schema/indexes, but the pattern is important.

---

# 43. Another incident — connection failures

Application says:

> "SQL is down."

You check:

```text id="w9p3k2"
Queries
CPU
I/O
```

Everything looks normal.

Then:

```text id="c4x8m1"
Application
     |
     X
SQL MI
```

You investigate:

```text id="n7q2v5"
DNS
Network route
Private endpoint/connectivity architecture
Authentication
Connection string
TLS/encryption requirements
Connection pool
```

Again:

> **Don't assume every database incident is a query-performance problem.**

---

# 44. Performance troubleshooting hierarchy

Use this order:

```text id="z5r8m3"
1. Confirm symptom
        ↓
2. Check platform/resource health
        ↓
3. Check waits
        ↓
4. Check blocking/deadlocks
        ↓
5. Identify expensive queries
        ↓
6. Inspect execution plans
        ↓
7. Check recent changes
        ↓
8. Remediate
        ↓
9. Validate
        ↓
10. Monitor
```

---

# 45. "What changed?" is one of your best questions

When performance suddenly changes, ask:

```text id="a6m2v9"
Was there a deployment?
Index change?
Statistics update?
Data volume increase?
Configuration change?
Azure scaling change?
New job?
New query?
Network change?
```

Sudden problems usually have a trigger.

---

# 46. Baselines

You need to know what "normal" looks like.

For example:

```text id="y8q3m5"
Normal CPU: 35–55%
Normal latency: <1 sec
Normal ETL: 25 min
Normal I/O: X
```

Then:

```text id="k4p7n2"
Today:
CPU 80%
Latency 8 sec
ETL 70 min
```

Now you can identify deviations.

---

# 47. Performance baseline

A good DBA tracks:

```text id="r6m9x1"
CPU
I/O
Memory pressure
Query duration
Transaction rate
Blocking
Deadlocks
Failed jobs
Connections
Storage
```

The exact metrics depend on workload.

---

# 48. Why historical data matters

Suppose a user says:

> "The database was slow yesterday at 3 PM."

Real-time monitoring can't show you everything that happened yesterday unless you retained the relevant telemetry.

That's why:

```text id="p3x8m7"
Query Store
Azure Monitor
Log Analytics
Monitoring history
```

are valuable.

---

# 49. Performance troubleshooting tools

Your toolbox should include:

```text id="j8n4q2"
Azure Portal
Azure Monitor
Log Analytics
Query Store
DMVs
Execution plans
Extended Events
SQL Agent history
Application telemetry
```

No single tool answers every question.

---

# 50. DMV concept

Dynamic Management Views provide runtime information about SQL Server state.

They can help investigate:

```text id="x5m2v8"
Active sessions
Requests
Waits
Locks
Indexes
Queries
Memory
```

Think:

```text id="q7r3n9"
DMV
 ↓
"What is happening right now?"
```

while Query Store is especially valuable for:

```text id="c6p8m1"
"What has been happening over time?"
```

---

# 51. DMV vs Query Store

Memorize this distinction:

| Tool            | Best for                           |
| --------------- | ---------------------------------- |
| DMVs            | Current/runtime state              |
| Query Store     | Historical query behavior          |
| Execution Plan  | How a query executes               |
| Extended Events | Targeted event investigation       |
| Azure Monitor   | Azure resource/platform monitoring |
| Log Analytics   | Centralized telemetry/log analysis |

---

# 52. Senior troubleshooting question

Suppose someone tells you:

> "CPU is 90%, therefore the database is slow."

Your response should be:

> "High CPU is a symptom, not necessarily the root cause. I'd identify the top CPU-consuming queries, review waits and execution plans, correlate the timing with recent changes, and determine whether query optimization or resource scaling is appropriate."

That's a strong answer.

---

# 53. Scaling vs tuning

This distinction is extremely important.

### Tune

When:

```text id="x1c8m5"
One bad query
Bad plan
Missing index
Blocking
Inefficient SQL
```

### Scale

When:

```text id="q7m3n9"
Workload legitimately exceeds capacity
Sustained resource pressure
Business growth
```

Sometimes you need both.

---

# 54. Example

Current:

```text id="w2n6p8"
SQL MI capacity
CPU = 95%
```

Top query:

```text id="r4x9m3"
Consumes 70% CPU
```

Fix query:

```text id="k7c2v5"
CPU → 50%
```

No scale-up required.

---

# 55. Opposite scenario

Suppose you optimize everything and still have:

```text id="v8m3q1"
CPU = 90%
24/7
```

and workload is growing.

Then:

```text id="p4x7n2"
Scale
```

may be appropriate.

The important part is having evidence.

---

# 56. The production incident framework

Whenever production is slow:

```text id="c8m2r7"
SYMPTOM
  ↓
SCOPE
  ↓
RESOURCE
  ↓
WAIT
  ↓
QUERY
  ↓
PLAN
  ↓
ROOT CAUSE
  ↓
FIX
  ↓
VALIDATE
  ↓
PREVENT
```

The last step matters.

---

# 57. Prevention after incident

Suppose you fixed a bad query.

Don't stop.

Ask:

```text id="n6q3x9"
Why wasn't it detected earlier?
```

Maybe add:

```text id="p1r8m4"
Performance baseline
Alert
Query Store monitoring
Regression detection
Code review
Index review
Load testing
```

Then the next incident may never happen.

---

# 58. Interview scenario

### "Production SQL MI is suddenly slow. What are your first five checks?"

A strong answer:

```text id="v7m3q1"
1. Confirm scope and reproduce/measure the slowdown.
2. Check SQL MI resource/platform metrics.
3. Examine waits, blocking and deadlocks.
4. Identify top expensive queries using runtime data/Query Store.
5. Compare execution plans and recent changes.
```

Then continue with:

```text id="x2p8n5"
6. Determine root cause.
7. Remediate.
8. Validate.
9. Add preventive monitoring.
```

---

# 59. Your troubleshooting cheat sheet

```text id="j9c4m7"
HIGH CPU?
    ↓
Top CPU queries
    ↓
Execution plan
    ↓
Recent changes

HIGH I/O?
    ↓
Large scans?
    ↓
Indexes?
    ↓
Query plan?

HIGH LATENCY?
    ↓
Blocking?
    ↓
Waits?
    ↓
Network?

DEADLOCK?
    ↓
Capture deadlock graph
    ↓
Find conflicting transactions
    ↓
Fix access order/query design

CONNECTION FAILURE?
    ↓
Network
    ↓
DNS
    ↓
Authentication
    ↓
Permissions
    ↓
Connection configuration
```

---

# 60. Final mental model

When someone says:

> **"Azure SQL Managed Instance is slow."**

Don't accept that statement as the diagnosis.

Translate it into:

```text id="u2m8p5"
What exactly is slow?
        ↓
When did it start?
        ↓
Who is affected?
        ↓
What changed?
        ↓
What resource is constrained?
        ↓
What are the queries waiting on?
        ↓
Which query is responsible?
        ↓
What does its plan show?
        ↓
What is the root cause?
        ↓
What is the safest fix?
```

That is how a production DBA thinks.

---

## Next — Lesson 80: Query Performance Tuning in SQL MI

We'll go deeper into the **actual SQL tuning side**:

* Clustered vs nonclustered indexes
* Covering indexes
* Included columns
* Index seek vs scan
* Key lookups
* Statistics
* Execution plans
* SARGability
* Parameter sniffing/sensitivity
* Query Store plan regression
* TempDB issues
* CTEs vs temp tables
* `EXISTS` vs `IN`
* `JOIN` performance
* Real production slow-query examples
* and a complete **"8-second query → 200 ms"** tuning walkthrough.
