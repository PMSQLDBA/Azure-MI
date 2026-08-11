# Lesson 113 — Azure SQL Managed Instance Performance Engineering — Expert Level

This lesson is about moving from:

> **“The query is slow.”**

to:

> **“I can identify the bottleneck, prove its cause, fix it safely, and demonstrate the improvement.”**

That is the difference between basic troubleshooting and **expert database performance engineering**.

---

# 1. The performance hierarchy

When a production SQL MI workload is slow, investigate in this order:

```text
Business symptom
      ↓
Workload
      ↓
Query
      ↓
Execution plan
      ↓
Waits / bottleneck
      ↓
Resource
      ↓
Root cause
      ↓
Fix
      ↓
Measure
```

Don't immediately create an index.

---

# 2. The four major resources

Most SQL performance problems ultimately involve:

```text
CPU
Memory
I/O
Concurrency
```

A query can also be affected by:

```text
Network
TempDB
Compilation
Plan quality
External dependencies
```

---

# 3. CPU-bound workload

Symptoms:

```text
CPU consistently high
Queries consuming significant CPU
Little evidence of I/O saturation
```

Potential causes:

```text
Bad execution plan
Missing/inefficient indexes
Large scans
Expensive joins
Functions
High compilation/recompilation
Poor cardinality estimates
```

---

# 4. I/O-bound workload

Symptoms:

```text
High reads
Storage latency
Large scans
Slow physical access
```

Potential causes:

```text
Missing index
Inefficient index
Large table scan
Poor data locality
Memory pressure
```

---

# 5. Memory pressure

Symptoms may include:

```text
Poor cache efficiency
Large physical reads
Memory grant waits
Plan instability
```

Potential causes:

```text
Large queries
Bad cardinality estimates
Concurrent memory-intensive queries
Insufficient available memory
```

---

# 6. Concurrency problems

The database may have enough CPU and I/O capacity but still be slow because sessions are waiting on one another.

Examples:

```text
Blocking
Deadlocks
Lock contention
Long transactions
```

---

# 7. Wait statistics

One of the most powerful performance concepts:

> **When a worker isn't actively running, what is it waiting for?**

Think:

```text
Running
  ↓
Waiting
  ↓
Resource / synchronization
```

---

# 8. Why waits matter

Suppose CPU is only:

```text
35%
```

but users complain about severe latency.

That doesn't mean:

> "SQL has plenty of CPU, so the database is fine."

The workload could be waiting on:

```text
Locks
I/O
Memory grants
Parallelism
```

---

# 9. Common wait categories

Conceptually, classify waits into:

```text
CPU / scheduler
I/O
Locking
Memory
Parallelism
Network / client
Log
External resources
```

The exact wait names matter, but **categorizing the bottleneck first** is more important.

---

# 10. Blocking

Blocking occurs when one session prevents another from acquiring a required lock/resource.

Example:

```text
Session A
   |
Long transaction
   |
Lock
   |
Session B → WAITING
Session C → WAITING
Session D → WAITING
```

One transaction can therefore create a large latency cascade.

---

# 11. Blocking chain

Imagine:

```text
A
↓
B
↓
C
↓
D
↓
E
```

Don't automatically kill E.

Find the **head blocker**.

---

# 12. Head blocker

The head blocker is the session at the beginning of the blocking chain.

Conceptually:

```text
HEAD BLOCKER
      ↓
    Blocked
      ↓
    Blocked
      ↓
    Blocked
```

Fixing the root blocker may release the entire chain.

---

# 13. Long transactions

Long transactions are dangerous because they can hold locks for extended periods.

Example:

```text
BEGIN TRANSACTION
 ↓
Large operation
 ↓
Application delay
 ↓
COMMIT
```

If the operation takes 30 minutes, locks may remain relevant for much of that time.

---

# 14. Transaction design

Good transactions should generally be:

```text
Focused
Predictable
As short as practical
```

Don't perform unnecessary application work while a database transaction is open.

---

# 15. Deadlocks

Deadlock:

```text
Session A holds Lock 1
       ↓
needs Lock 2

Session B holds Lock 2
       ↓
needs Lock 1
```

Neither can proceed.

SQL Server detects the cycle and chooses a victim.

---

# 16. Deadlock is not simply "slow SQL"

A deadlock is a **coordination problem**.

Typical fixes include:

```text
Consistent object access order
Shorter transactions
Better indexes
Reduced lock duration
Application retry strategy
```

---

# 17. Deadlock example

Transaction A:

```text
UPDATE Customer
UPDATE Order
```

Transaction B:

```text
UPDATE Order
UPDATE Customer
```

Different acquisition order can create a cycle.

---

# 18. Execution plans

The execution plan answers:

> **How did SQL Server actually execute this query?**

It shows operations such as:

```text
Seek
Scan
Join
Sort
Aggregate
Lookup
Filter
Parallelism
```

---

# 19. Estimated vs actual plan

### Estimated plan

What the optimizer expects.

### Actual plan

What happened during execution, including runtime information.

Comparing the two can reveal major estimation problems.

---

# 20. Cardinality estimation

The optimizer needs to estimate:

```text
How many rows will this operation produce?
```

Example:

```text
Expected: 100 rows
Actual:   10,000,000 rows
```

That difference can radically change the execution plan.

---

# 21. Why estimates matter

Suppose optimizer expects:

```text
10 rows
```

It might choose:

```text
Nested Loops
```

But actual rows are:

```text
10 million
```

That plan may become disastrous.

---

# 22. Statistics

Statistics help the optimizer understand data distribution.

They describe information such as:

```text
Row counts
Data distribution
Column relationships, to the extent captured
```

Stale or insufficient statistics can contribute to bad plans.

---

# 23. Statistics maintenance

Don't blindly assume:

> "Statistics are automatically handled, therefore statistics can never be the problem."

Monitor:

```text
Modification patterns
Data distribution changes
Query behavior
Plan quality
```

---

# 24. Indexes

Indexes exist to make data access more efficient.

But:

> **Every index is a performance tradeoff.**

Indexes can improve:

```text
Reads
Searches
Joins
Ordering
```

while increasing:

```text
Storage
INSERT cost
UPDATE cost
DELETE cost
Maintenance
```

---

# 25. Missing-index recommendations

Treat automated missing-index suggestions as:

> **Hypotheses, not commands.**

Why?

Because the recommendation may:

```text
Overlap existing indexes
Create excessive write overhead
Ignore workload diversity
Miss broader indexing strategy
```

---

# 26. Index overlap

Suppose you already have:

```text
Index A
(A, B, C)
```

and a tool recommends:

```text
Index B
(A, B)
```

You shouldn't automatically create Index B.

First evaluate whether the existing index already supports the workload.

---

# 27. Composite indexes

Column order matters.

Example:

```text
INDEX (CustomerId, OrderDate)
```

is different from:

```text
INDEX (OrderDate, CustomerId)
```

because the optimizer's ability to efficiently use the index depends on predicates, ordering, and query shape.

---

# 28. SARGability

A predicate is generally more optimizer-friendly when it allows efficient index access.

Example:

```text
WHERE OrderDate >= @StartDate
```

is typically more index-friendly than applying a function directly to the indexed column in a way that prevents efficient seeking.

Concept:

```text
Column
 ↓
Search efficiently
```

---

# 29. Function on indexed column

Potentially problematic pattern:

```text
WHERE YEAR(OrderDate) = 2026
```

versus a range predicate:

```text
WHERE OrderDate >= '2026-01-01'
AND OrderDate <  '2027-01-01'
```

The second form can often be much more index-friendly.

---

# 30. Key lookups

A plan may perform:

```text
Index Seek
    ↓
Key Lookup
    ↓
Base table
```

A few lookups may be fine.

Millions of lookups may be expensive.

---

# 31. Covering index

An index can sometimes contain enough information to satisfy the query without repeatedly accessing the base table.

Conceptually:

```text
Query
 ↓
Index
 ↓
Complete result
```

But don't automatically include every column.

Large indexes create their own problems.

---

# 32. Parameter sensitivity

This is a senior-level problem.

Suppose:

```text
@CustomerId = 1
```

returns:

```text
10 rows
```

but:

```text
@CustomerId = 999999
```

returns:

```text
5 million rows
```

One plan may be excellent for one value and terrible for another.

---

# 33. Why parameter sensitivity happens

The optimizer may compile a plan based on one parameter value.

That plan can later be reused for a very different value.

Conceptually:

```text
First execution
      ↓
Plan compiled
      ↓
Plan cached
      ↓
Different parameter
      ↓
Poor performance
```

---

# 34. Diagnosing parameter sensitivity

Look for:

```text
Different runtime durations
Different row counts
Different execution characteristics
Same query/plan context
```

Query Store can be extremely useful here.

---

# 35. Query Store

Query Store provides historical visibility into query execution and plan behavior.

Think:

```text
Query
 ↓
Plans
 ↓
Performance over time
```

It helps answer:

> "When did this query become slow?"

---

# 36. Query Store and regression

Example:

```text
Monday
Plan A → 100 ms

Tuesday
Plan B → 4 seconds
```

Query Store helps establish that a plan change corresponds to the regression.

---

# 37. Plan regression

A plan can regress after:

```text
Statistics changes
Data growth
Index changes
Configuration changes
Engine changes
Parameter distribution changes
```

Therefore:

> **A query that was fast yesterday can become slow without any application code change.**

---

# 38. Plan forcing

In appropriate situations, a known-good plan can be forced.

But think of forcing as:

```text
Stabilization
```

not:

```text
Permanent root-cause solution
```

Investigate why the optimizer chose the bad plan.

---

# 39. Query tuning workflow

A disciplined workflow:

```text
1. Identify slow query
2. Establish baseline
3. Capture actual plan
4. Check waits
5. Check row estimates
6. Check indexes
7. Check statistics
8. Check parameter behavior
9. Apply one change
10. Measure
11. Validate under representative workload
```

---

# 40. Never tune without a baseline

Before changing anything record:

```text
Duration
CPU
Logical reads
Physical reads
Rows
Executions
Plan
Wait profile
```

Then compare afterward.

---

# 41. Example

Before:

```text
Duration: 4.2 sec
CPU:      3.1 sec
Reads:    800,000
```

After:

```text
Duration: 350 ms
CPU:      200 ms
Reads:    20,000
```

Now you can demonstrate the improvement objectively.

---

# 42. CPU vs elapsed time

Important distinction:

```text
Elapsed time
```

is what the user experiences.

```text
CPU time
```

is processor consumption.

A query can have:

```text
High elapsed
Low CPU
```

because it's waiting.

Or:

```text
High CPU
High elapsed
```

because it is computationally expensive.

---

# 43. Parallelism

SQL Server can execute parts of a query in parallel.

Conceptually:

```text
Query
 ↓
Multiple workers
 ↓
Combine results
```

Parallelism isn't automatically bad.

---

# 44. The wrong question

Don't ask:

> "How do I eliminate parallelism?"

Ask:

> "Why is this query using parallelism, and is the parallel plan appropriate?"

---

# 45. Parallelism overhead

Parallel execution has coordination costs.

For small queries:

```text
Serial
```

may be better.

For large analytical queries:

```text
Parallel
```

may be beneficial.

---

# 46. Memory grants

Queries sometimes request memory for operations such as:

```text
Sort
Hash join
Hash aggregate
```

If estimates are wrong, memory grants can be badly sized.

---

# 47. Excessive memory grant

Example:

```text
Expected rows: 1,000
Actual rows: 10
```

The query may receive more memory than necessary.

That memory could be unavailable to other queries.

---

# 48. Insufficient memory grant

Opposite scenario:

```text
Expected rows: 10
Actual rows: 10 million
```

The operation may not receive enough memory and could spill to tempdb.

---

# 49. TempDB

TempDB is used for many operations, including workloads involving:

```text
Sorts
Hash operations
Temporary objects
Versioning
Intermediate results
```

Performance problems here can affect many sessions simultaneously.

---

# 50. TempDB bottlenecks

Potential symptoms:

```text
High TempDB activity
Spills
Allocation contention
Heavy temporary object usage
```

Investigate the workload rather than treating TempDB as an isolated component.

---

# 51. Query spills

A spill occurs when an operation can't keep all required working data in memory and uses TempDB.

Examples:

```text
Sort spill
Hash spill
```

Large or unexpected spills can significantly hurt performance.

---

# 52. Why spills happen

Common causes:

```text
Poor cardinality estimate
Insufficient memory grant
Large intermediate result
Data growth
```

---

# 53. Query anti-patterns

Watch for:

```text
SELECT *
Functions on indexed columns
Implicit conversions
Unnecessary DISTINCT
Large sorts
Unbounded result sets
Non-SARGable predicates
RBAR-style processing
```

---

# 54. SELECT *

Why can it be problematic?

```text
More data
More I/O
Larger network payload
Potentially less effective covering indexes
```

Explicitly request needed columns.

---

# 55. Implicit conversions

Suppose:

```text
Column = varchar
Parameter = nvarchar
```

The engine may need conversion.

Depending on types and data distribution, this can interfere with efficient index access.

---

# 56. Query design matters

Database performance isn't just DBA infrastructure work.

A query can be technically valid and still be inefficient.

Performance requires collaboration between:

```text
DBA
Developers
Architects
```

---

# 57. Application behavior

Sometimes the database is healthy but the application sends:

```text
10,000 tiny queries
```

instead of:

```text
1 efficient query
```

This is a workload design issue.

---

# 58. N+1 query pattern

Example:

```text
Get 1,000 customers
       ↓
Run another query for each customer
       ↓
1,001 queries
```

Potentially much worse than a properly designed set-based query.

---

# 59. Set-based thinking

Prefer:

```text
Process a set
```

over:

```text
Process one row at a time
```

when appropriate.

SQL engines are optimized for set-oriented operations.

---

# 60. Performance methodology

When someone says:

> "The database is slow."

Ask:

```text
When?
Which workload?
Which queries?
How much slower?
Compared with what baseline?
What changed?
```

This turns a vague complaint into an engineering problem.

---

# 61. The five-question diagnostic

### 1. Is it CPU?

```text
CPU pressure?
```

### 2. Is it I/O?

```text
Reads/storage latency?
```

### 3. Is it waiting?

```text
Locks/memory/network/etc.?
```

### 4. Is the plan wrong?

```text
Estimates/indexes/joins?
```

### 5. What changed?

```text
Code?
Data?
Statistics?
Plan?
Indexes?
Workload?
```

---

# 62. Performance incident example

Users report:

> "Checkout is slow."

Start with:

```text
Business transaction
      ↓
Relevant application request
      ↓
SQL queries
      ↓
Query Store
      ↓
Execution plan
      ↓
Waits
```

Don't start by resizing the MI.

---

# 63. Scenario

You discover:

```text
CPU = 35%
I/O = normal
```

but:

```text
Lock waits = high
```

Root cause may be:

```text
Long-running transaction
```

Increasing CPU won't solve it.

---

# 64. Another scenario

You discover:

```text
CPU = 90%
```

Top query:

```text
Huge table scan
```

Execution plan:

```text
Scan → Hash Join → Sort
```

Possible solution:

```text
Index/query rewrite/statistics
```

—not necessarily a larger SQL MI.

---

# 65. Another scenario

Query Store shows:

```text
100 ms
 ↓
100 ms
 ↓
120 ms
 ↓
4 seconds
```

Plan changed at exactly the same time.

Now investigate:

```text
Plan regression
```

---

# 66. Another scenario

Query is fast for:

```text
CustomerId = 10
```

but terrible for:

```text
CustomerId = 500000
```

Suspect:

```text
Parameter sensitivity
```

---

# 67. Production tuning rule

> **Change one meaningful variable at a time whenever practical.**

Otherwise:

```text
Index changed
+
Statistics updated
+
Query rewritten
+
Configuration changed
```

and you don't know what fixed the issue.

---

# 68. Performance experiment

Use:

```text
Baseline
 ↓
Hypothesis
 ↓
Change
 ↓
Measure
 ↓
Compare
 ↓
Keep/revert
```

That's scientific performance engineering.

---

# 69. Don't optimize one query at the expense of the workload

Suppose an index makes:

```text
SELECT
```

20× faster.

But it causes:

```text
INSERT
```

to become 3× slower across millions of rows.

The correct question is:

> **Did overall workload performance improve?**

---

# 70. Read vs write optimization

Every optimization has a workload context.

```text
Read-heavy system
→ indexing may dominate

Write-heavy system
→ excessive indexing can hurt

Analytics
→ large scans/parallelism may be acceptable

OLTP
→ predictable low-latency access often matters more
```

---

# 71. OLTP vs analytics

### OLTP

```text
Small transactions
Low latency
High concurrency
Frequent writes
```

### Analytics

```text
Large scans
Aggregations
Longer queries
High memory usage
```

Don't tune them using identical assumptions.

---

# 72. Performance budgets

For critical operations define targets:

```text
Checkout P95 < X ms
Login P95 < Y ms
Report P95 < Z sec
```

Then database performance becomes measurable against business requirements.

---

# 73. Percentiles

Average latency can hide problems.

Example:

```text
Average = 200 ms
P95 = 2 sec
P99 = 10 sec
```

Most users may be fine while a significant tail suffers.

Monitor:

```text
P50
P95
P99
```

where appropriate.

---

# 74. Performance regression testing

Before releasing a database change:

```text
Baseline workload
 ↓
Apply change
 ↓
Replay representative workload
 ↓
Compare
```

Look for:

```text
Latency
CPU
Reads
Writes
Locking
```

---

# 75. Capacity planning

Performance engineering also asks:

> "What happens when the workload doubles?"

Monitor trends:

```text
CPU growth
Storage growth
IOPS growth
Data size
Query volume
Connection volume
```

---

# 76. Scaling decision

When workload grows, consider:

```text
Query optimization
Indexing
Schema design
Caching
Application changes
Scaling compute
Architecture changes
```

Scaling the instance should not be the automatic first answer.

---

# 77. Vertical scaling

More resources:

```text id="n5m8x2"
Larger compute capacity
```

Useful when the workload genuinely requires more resources.

---

# 78. Horizontal/application scaling

Depending on architecture, you may distribute:

```text
Read workload
Application workload
Regional workload
Analytical workload
```

But SQL MI architecture and workload semantics determine what is practical.

---

# 79. Performance observability stack

Think:

```text
Query Store
   +
DMVs
   +
Wait statistics
   +
Execution plans
   +
Azure monitoring
   +
Application telemetry
```

The strongest diagnosis correlates multiple layers.

---

# 80. Expert troubleshooting flow

```text
USER REPORTS SLOWNESS
          ↓
     Define symptom
          ↓
     Identify workload
          ↓
    Check time window
          ↓
   Check resource pressure
          ↓
      Check waits
          ↓
    Identify top queries
          ↓
     Inspect plans
          ↓
 Compare estimates/actuals
          ↓
 Check indexes/statistics
          ↓
 Check concurrency
          ↓
 Form hypothesis
          ↓
       Change
          ↓
       Measure
          ↓
      Validate
```

---

# 81. Interview question

### "How do you troubleshoot a slow query?"

> "I first establish the business symptom and baseline. Then I identify the query and examine its runtime metrics, waits, and actual execution plan. I compare estimated versus actual row counts, inspect indexes and statistics, evaluate parameter sensitivity and blocking, form a hypothesis, make the smallest appropriate change, and measure the result against the baseline."

---

# 82. Interview question

### "CPU is only 30%, but the application is slow. Why?"

Possible causes:

```text
Blocking
I/O waits
Memory grants
Network/client waits
Concurrency
External dependencies
```

CPU utilization alone doesn't determine database health.

---

# 83. Interview question

### "Would you always create an index when SQL recommends one?"

> "No. I'd evaluate the recommendation against existing indexes and the overall workload. I would consider read improvement, write overhead, storage, overlap, selectivity, and whether the query itself should be rewritten."

---

# 84. Interview question

### "What is parameter sniffing?"

> "Parameter sniffing is the optimizer compiling a reusable plan based on parameter values observed at compilation. When data distribution is highly skewed, that plan may perform poorly for later executions using substantially different parameter values."

---

# 85. Interview question

### "What is your first response to a production performance incident?"

Not:

> "Increase the MI size."

Instead:

```text
Define symptom
 ↓
Check current health
 ↓
Identify waits
 ↓
Find affected workload
 ↓
Correlate recent changes
 ↓
Mitigate
 ↓
Root cause
 ↓
Permanent fix
```

---

# 86. Golden rule

> **Performance tuning is not guesswork. It is measurement → hypothesis → controlled change → measurement.**

The expert DBA doesn't say:

> "I think this index will help."

They say:

> "The workload is spending most of its time doing X because of Y. The proposed change should address Y. Here is the baseline, and here is the measured improvement."

---

# 87. Final mental model

```text
                 USER EXPERIENCE
                       ↓
                  APPLICATION
                       ↓
                    QUERY
                       ↓
                 QUERY PLAN
                       ↓
        +--------------+--------------+
        |              |              |
       CPU            I/O         CONCURRENCY
        |              |              |
        +--------------+--------------+
                       ↓
                  WAIT PROFILE
                       ↓
                 ROOT CAUSE
                       ↓
                  OPTIMIZATION
                       ↓
                  MEASUREMENT
                       ↓
                  VALIDATION
```

That is the foundation of **expert-level Azure SQL Managed Instance performance engineering**.

---

## Next — Lesson 114: Azure SQL Managed Instance Monitoring & Observability — Expert Level

We'll build a complete observability strategy covering **Azure Monitor, Log Analytics, diagnostic settings, metrics, DMVs, Query Store, alerts, dashboards, KQL concepts, SLOs/SLIs, proactive detection, capacity forecasting, incident correlation, and production monitoring architecture.**
