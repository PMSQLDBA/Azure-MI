# Lesson 103 — Azure SQL Managed Instance Performance Tuning — Advanced DBA

This lesson is about moving from **“the database is slow”** to a disciplined method for finding and fixing the actual bottleneck.

> **Performance tuning is not “make queries faster.” It is finding the limiting resource or inefficient workload, changing the right thing, and proving the change improved the system without creating a worse problem elsewhere.**

---

# 1. The performance-tuning hierarchy

Use this order:

```text
Application symptom
        ↓
Measure
        ↓
Identify workload
        ↓
Identify bottleneck
        ↓
Find root cause
        ↓
Apply smallest safe change
        ↓
Measure again
```

Don't start with:

```text
"Let's add an index."
```

---

# 2. The four major bottlenecks

Most SQL performance problems eventually involve one or more of:

```text
CPU
I/O
Memory
Concurrency
```

And sometimes:

```text
Network
Application
Configuration
```

---

# 3. CPU-bound workload

Typical symptoms:

```text
High CPU
Queries consume large CPU time
Plans contain expensive operators
```

Potential causes:

```text
Bad execution plan
Large scans
Complex joins
Poor predicates
High concurrency
Missing/inefficient indexes
```

---

# 4. I/O-bound workload

Typical symptoms:

```text
High reads
High I/O latency
Long-running queries
Storage pressure
```

Potential causes:

```text
Large scans
Poor indexing
Inefficient queries
Working set larger than available memory
```

---

# 5. Memory pressure

Potential symptoms:

```text
Memory-related waits
Large memory grants
Spills
Poor cache efficiency
Slow queries
```

Don't simply conclude:

> "Memory is high, therefore memory is the problem."

Determine whether the workload is actually experiencing memory pressure.

---

# 6. Concurrency problems

Examples:

```text
Blocking
Deadlocks
Lock contention
Connection storms
```

A query can be perfectly optimized and still appear slow because it is waiting behind another transaction.

---

# 7. The most important question

When a query takes 10 seconds, ask:

> **Did it spend 10 seconds doing work, or 10 seconds waiting?**

This distinction is fundamental.

---

# 8. Query duration

Suppose:

```text id="q5b7q0"
Duration = 10 seconds
CPU = 1 second
```

That means:

```text
~1 sec CPU
~9 sec waiting
```

Investigate waits, blocking, I/O, or other external factors.

---

# 9. Compare with CPU

Another query:

```text id="d9b2n2"
Duration = 10 sec
CPU = 9.5 sec
```

Now it is likely doing substantial work.

Investigate:

```text
Execution plan
Indexes
Joins
Predicates
Cardinality
```

---

# 10. Query Store first

For historical performance issues, Query Store is often your starting point.

Look for:

```text id="m0g2vz"
High CPU
High duration
High execution count
Recent regressions
Plan changes
```

---

# 11. Top queries

A useful investigation list:

```text id="x9i7w6"
Top CPU queries
Top duration queries
Top logical reads
Top execution count
Recently regressed queries
```

Don't only investigate the single slowest query.

A query taking 100 ms but executing 1 million times may matter more.

---

# 12. Total resource consumption

Consider:

```text
Total CPU =
CPU per execution × execution count
```

Example:

```text
Query A:
1 sec × 10 executions = 10 sec

Query B:
10 ms × 100,000 executions = 1,000 sec
```

Query B is a much larger workload.

---

# 13. Execution plans

Execution plans explain how SQL Server intends to execute a query.

Conceptually:

```text
Query
 ↓
Optimizer
 ↓
Execution Plan
 ↓
Operators
 ↓
Rows processed
```

---

# 14. Common operators

You'll frequently encounter:

```text
Index Seek
Index Scan
Table Scan
Nested Loops
Hash Match
Merge Join
Sort
Aggregate
Filter
Key Lookup
```

Understanding these is essential for advanced tuning.

---

# 15. Index Seek

A seek can efficiently locate qualifying rows.

Conceptually:

```text
Index
 ↓
Specific key range
 ↓
Required rows
```

But:

> **Seek does not automatically mean fast.**

A seek can still process a huge number of rows.

---

# 16. Index Scan

A scan reads a larger portion of an index.

That isn't automatically bad.

For example:

```text
Query needs 80% of table
```

A scan may be completely reasonable.

---

# 17. Table scan

Reading a whole table may be appropriate for:

```text
Small table
Large percentage of rows requested
No useful index
```

The mistake is assuming:

> "Table scan = bad."

Always consider the workload.

---

# 18. Nested Loops

Nested loops can be excellent when:

```text
Outer input = small
Inner input = efficiently indexed
```

Conceptually:

```text
Outer row
   ↓
Find matching inner rows
   ↓
Repeat
```

---

# 19. Hash Match

Hash joins can be useful for larger unsorted inputs.

But they can require significant memory.

Potential concern:

```text
Insufficient memory
 ↓
Spill to tempdb
 ↓
Performance degradation
```

---

# 20. Merge Join

Merge joins are particularly useful when inputs are appropriately ordered.

Conceptually:

```text
Sorted input A
+
Sorted input B
       ↓
   Merge Join
```

The optimizer chooses among join strategies based on estimates and cost.

---

# 21. Sort operator

Sorting can become expensive when large datasets are involved.

Example:

```sql id="2u3u2s"
SELECT *
FROM Orders
ORDER BY OrderDate;
```

If a useful index can provide the required ordering, SQL Server may avoid an expensive sort.

---

# 22. Key Lookup

A common pattern:

```text id="1c2k3a"
Index Seek
    ↓
Key Lookup
    ↓
Base table
```

For a small number of rows, this may be fine.

For thousands or millions of rows, repeated lookups can become expensive.

---

# 23. Covering indexes

Suppose:

```sql id="n1v9gk"
SELECT CustomerId, OrderDate, Amount
FROM Orders
WHERE CustomerId = @CustomerId;
```

An index on `CustomerId` may locate the rows but still require lookups for other columns.

A covering index can include the required columns where appropriate.

Conceptually:

```text id="j4m8w2"
Search key
+
Included columns
```

---

# 24. But don't over-index

Every index has costs:

```text id="3y0v6m"
Storage
INSERT overhead
UPDATE overhead
DELETE overhead
Maintenance
Optimization complexity
```

Therefore:

> **An index is a workload trade-off, not free performance.**

---

# 25. SARGability

One of the most important query-tuning concepts.

A predicate is generally more optimizer-friendly when it allows efficient use of an index.

Example:

```sql id="3x5r4c"
WHERE OrderDate >= '2026-01-01'
```

is generally more index-friendly than applying a function to the indexed column:

```sql id="5z6c7n"
WHERE YEAR(OrderDate) = 2026
```

The second form can make efficient index access harder.

---

# 26. Another SARGability example

Avoid unnecessarily transforming the indexed column:

```sql id="zq2t9n"
WHERE UPPER(CustomerName) = 'JOHN'
```

Depending on collation and design, this may interfere with efficient access.

Better query design depends on the actual data model and requirements.

---

# 27. Implicit conversions

A mismatch between data types can cause unwanted conversions.

Example:

```text id="u4myh8"
Column = VARCHAR
Parameter = NVARCHAR
```

Depending on the circumstances, SQL Server may need to convert values.

That can affect index usage and performance.

---

# 28. Parameter sniffing

A classic advanced SQL Server problem.

Suppose a stored procedure:

```sql id="0d7w4r"
CREATE PROCEDURE GetOrders
    @CustomerId INT
AS
...
```

The optimizer may compile a plan based on the first parameter value it sees.

If data distribution is highly uneven:

```text id="jv9a3m"
Customer A → 5 rows
Customer B → 5,000,000 rows
```

One plan may not be ideal for both.

---

# 29. Parameter-sensitive behavior

Conceptually:

```text
Small customer
     ↓
Plan A is ideal

Large customer
     ↓
Plan B is ideal
```

If the same cached plan is used inappropriately, performance can vary dramatically.

---

# 30. Detecting parameter-related problems

Look for:

```text id="9l7f9j"
Same query
Different parameter values
Large runtime differences
Plan instability
```

Query Store history can be very useful here.

---

# 31. Cardinality estimation

The optimizer needs to estimate:

> **How many rows will this operation produce?**

Example:

```text
Estimated rows = 100
Actual rows = 1,000,000
```

That's a major estimation error.

---

# 32. Why estimates matter

The optimizer chooses plans based on estimates.

If estimates are wrong:

```text id="v7f9jg"
Wrong estimate
    ↓
Wrong cost calculation
    ↓
Poor plan choice
```

Possible causes include:

```text
Stale statistics
Data skew
Correlated columns
Complex predicates
Parameter sensitivity
```

---

# 33. Statistics

Statistics describe data distribution and help the optimizer estimate cardinality.

Think:

```text id="p3b4wr"
Data
 ↓
Statistics
 ↓
Optimizer estimates
 ↓
Execution plan
```

---

# 34. Statistics problem

Suppose data changes dramatically:

```text id="mm8y43"
Yesterday:
1 million rows

Today:
50 million rows
```

The optimizer's assumptions may no longer accurately represent the workload.

This can contribute to plan regressions.

---

# 35. Execution plan regression

Classic pattern:

```text id="7i5k0r"
Plan A
 ↓
Fast

Plan B
 ↓
Slow
```

Query Store can help identify the historical plan change.

---

# 36. Plan forcing

Query Store can support plan-management techniques such as plan forcing in appropriate situations.

Conceptually:

```text id="8t2k1g"
Known-good plan
      ↓
Use while root cause is investigated
```

But:

> **Plan forcing is often a mitigation, not the final root-cause fix.**

---

# 37. Wait statistics

Waits help answer:

> **What is preventing progress?**

Common categories include waits related to:

```text id="l0u7os"
Locks
I/O
Memory
CPU scheduling
Parallelism
```

The exact interpretation depends on workload and environment.

---

# 38. Blocking investigation

When users report:

> "Everything suddenly became slow."

Check:

```text id="4h6h6y"
Blocking
```

Example:

```text id="v5i5id"
Transaction A
    ↓
holds lock
    ↓
Transaction B waits
    ↓
Transaction C waits
    ↓
Transaction D waits
```

One transaction can create a large apparent performance problem.

---

# 39. Head blocker

Find the session at the beginning of the chain.

```text id="7z1a9h"
Head blocker
    ↓
B
    ↓
C
    ↓
D
```

Fixing D doesn't solve the underlying problem.

---

# 40. Long transactions

Long transactions are frequent sources of blocking.

Example:

```text id="m1q5do"
BEGIN TRAN
   ↓
Large operation
   ↓
Application waits
   ↓
User goes to lunch
   ↓
Transaction remains open
```

Now other sessions can be blocked.

---

# 41. Transaction design

Keep transactions:

```text id="v9tq20"
As short as practical
Predictable
Consistent
```

Avoid unnecessary work inside the transaction.

---

# 42. Deadlocks

Deadlock pattern:

```text id="qk5t3a"
Transaction A
 locks Row 1
 wants Row 2

Transaction B
 locks Row 2
 wants Row 1
```

SQL Server chooses a victim.

---

# 43. Deadlock prevention

Common strategies include:

```text id="9az5r1"
Consistent object access order
Short transactions
Appropriate indexing
Reduced unnecessary locking
Application retry for deadlock victims
```

---

# 44. Tempdb

Temporary objects and internal operations may use tempdb.

Heavy usage can come from:

```text id="b7z8f2"
Sorts
Hash operations
Temporary tables
Versioning
Spills
```

Don't treat tempdb as an afterthought.

---

# 45. Memory grants

Some queries request substantial memory for:

```text id="b0e7gh"
Sorts
Hash joins
Aggregations
```

If the grant is too large:

```text id="4j2o6v"
One query
 ↓
Consumes large memory
 ↓
Other queries wait
```

If too small:

```text id="x5v6dp"
Spill
 ↓
tempdb I/O
 ↓
Slower query
```

---

# 46. Spills

A spill occurs when an operation can't complete entirely in the available memory grant and uses tempdb.

Conceptually:

```text id="u4h1hz"
Sort/Hash
   ↓
Insufficient memory
   ↓
tempdb
   ↓
Performance cost
```

Spills are worth investigating when significant.

---

# 47. Parallelism

SQL Server can execute parts of a query in parallel.

Parallelism isn't inherently bad.

The question is:

> **Is the amount of parallel work appropriate for this workload?**

---

# 48. Parallelism problem

A poorly chosen plan may consume many CPU resources:

```text id="a4z7yb"
One query
   ↓
Many workers
   ↓
High CPU
   ↓
Other queries suffer
```

Investigate the workload before changing global parallelism settings.

---

# 49. Scaling vs tuning

Suppose CPU is 95%.

You have two broad choices:

```text id="4u3v9w"
Scale resources
```

or:

```text id="7k0y7f"
Tune workload
```

Often the best answer is:

```text id="t3u2k4"
Immediate scale/mitigate
+
Long-term query optimization
```

---

# 50. Don't tune blindly

Bad process:

```text id="m1n0fe"
Slow
 ↓
Add index
 ↓
Still slow
 ↓
Add another index
 ↓
Write performance drops
```

Better:

```text id="q0n2d7"
Measure
 ↓
Diagnose
 ↓
Hypothesis
 ↓
Change
 ↓
Measure
```

---

# 51. Performance tuning example

Suppose:

```text
Order search:
8 seconds
```

Application reports:

> "Users are complaining."

---

## Step 1 — Query Store

Find:

```text id="w6g5sa"
CPU = 6 sec
Reads = very high
```

So this isn't primarily waiting.

---

## Step 2 — Execution plan

Plan shows:

```text id="n8h1sm"
Large table scan
```

---

## Step 3 — Predicate

Query:

```sql id="y3b2r0"
WHERE YEAR(OrderDate) = 2026
```

---

## Step 4 — Rewrite

Use a date range:

```sql id="l8d4f7"
WHERE OrderDate >= '2026-01-01'
  AND OrderDate <  '2027-01-01'
```

---

## Step 5 — Index

Evaluate whether an index on `OrderDate` is appropriate.

---

## Step 6 — Validate

Before:

```text id="1t4h4n"
8 sec
```

After:

```text id="6t4h4n"
250 ms
```

Then check:

```text id="x2q5r4"
CPU
I/O
other queries
write overhead
```

---

# 52. Performance tuning example — blocking

Application:

```text
Requests = 20 seconds
```

Database:

```text
CPU = 25%
```

Query Store:

```text
Query itself = 100 ms
```

Investigation:

```text id="o6d0a7"
Long transaction
    ↓
Blocking
    ↓
Requests wait
```

The solution isn't an index.

It's transaction/concurrency remediation.

---

# 53. Performance tuning example — plan regression

Before deployment:

```text id="8o8qyp"
Query = 300 ms
```

After:

```text id="i2v8s0"
Query = 8 sec
```

Query Store shows:

```text id="g2p3s7"
Old plan → fast
New plan → slow
```

Investigate:

```text id="l9d3qf"
Statistics
Indexes
Schema changes
Parameter distribution
Plan choice
```

Plan forcing may be an interim mitigation if appropriate.

---

# 54. Performance tuning example — excessive execution count

Query:

```text id="p1t7cs"
5 ms each
```

Sounds excellent.

But:

```text
1,000,000 executions
```

Total workload:

```text
~5,000 seconds of CPU/time contribution
```

The solution may be:

```text id="f9w0z3"
Reduce round trips
Batch operations
Cache appropriate data
Optimize application behavior
```

Not necessarily an index.

---

# 55. The tuning decision tree

```text id="m4r5n6"
Query slow?
     |
     ↓
Doing work or waiting?
     |
 +---+---+
 |       |
Work   Waiting
 |       |
 ↓       ↓
Plan    Blocking?
 |       I/O?
 |       Locks?
 ↓
Indexes?
Estimates?
Stats?
SARGability?
Memory?
```

This decision tree prevents random tuning.

---

# 56. Performance baseline

Maintain:

```text id="z2t5cv"
Normal CPU
Normal latency
Normal I/O
Normal waits
Normal connections
Top queries
Normal execution plans
```

Then compare incidents against the baseline.

---

# 57. Capacity vs query problem

### Query problem

```text
One workload suddenly consumes huge CPU
```

### Capacity problem

```text
All workloads are consistently near resource limits
```

These require different solutions.

---

# 58. Scaling signals

Consider scaling when:

```text id="6w8g3b"
Workload is legitimate
Queries are reasonably optimized
Resource saturation is persistent
Business growth is expected
```

Don't scale simply to hide a bad query.

---

# 59. The performance hierarchy

Use this order:

```text
1. Fix obviously inefficient SQL
2. Fix indexing/statistics problems
3. Fix concurrency issues
4. Fix application behavior
5. Optimize configuration where justified
6. Scale resources when workload genuinely requires it
```

The exact order can vary by incident, but don't use scaling as the default answer.

---

# 60. Production performance checklist

```text
QUERY
☐ Duration
☐ CPU
☐ Logical reads
☐ Execution count
☐ Execution plan

OPTIMIZER
☐ Cardinality estimates
☐ Statistics
☐ Parameter sensitivity
☐ Plan regression

INDEXING
☐ Seek/scan behavior
☐ Key lookups
☐ Covering opportunities
☐ Write overhead

CONCURRENCY
☐ Blocking
☐ Deadlocks
☐ Long transactions
☐ Lock waits

MEMORY/I/O
☐ Memory grants
☐ Spills
☐ I/O latency
☐ tempdb behavior

APPLICATION
☐ Connection pooling
☐ Round trips
☐ Retry behavior
☐ Query patterns
```

---

# 61. Interview question

### "A query is taking 10 seconds. What do you check first?"

Strong answer:

> "I first determine whether the time is spent doing work or waiting. Then I inspect Query Store and the execution plan, looking at CPU, reads, waits, blocking, cardinality estimates, indexing, and recent plan changes. I also correlate the problem with application and deployment activity before changing anything."

---

# 62. Interview question

### "Are table scans always bad?"

> "No. A scan can be appropriate when a query needs a large percentage of a table or the table is small. I evaluate the number of rows processed, cost, workload frequency, and alternative access paths rather than treating scans as inherently bad."

---

# 63. Interview question

### "Why can adding an index make things worse?"

Because indexes have costs:

```text
Storage
Write overhead
Maintenance
Additional optimizer choices
```

An index that helps one query may hurt write-heavy workloads.

---

# 64. Interview question

### "What is parameter sniffing?"

> "It's a situation where a plan compiled using one parameter value may perform poorly for other parameter values, particularly when data distribution is highly uneven. I would investigate plan history, parameter sensitivity, and workload characteristics before selecting a mitigation."

---

# 65. Interview question

### "What is the difference between blocking and deadlocking?"

### Blocking

```text
A waits for B
```

Eventually B can finish and A proceeds.

### Deadlock

```text
A waits for B
B waits for A
```

Neither can proceed, so the database detects the cycle and chooses a victim.

---

# 66. Interview question

### "Would you scale up immediately when CPU reaches 95%?"

> "I'd first determine whether the CPU is caused by legitimate workload growth or an inefficient query. If the business is being impacted, I may scale as an immediate mitigation, but I'd still investigate the underlying workload so that scaling doesn't simply hide a query or application problem."

---

# 67. Final performance model

Memorize this:

```text id="4y0m2x"
                 USER REPORT
                      |
                      ↓
                  MEASURE
                      |
          +-----------+-----------+
          |                       |
        WORK                    WAIT
          |                       |
     Execution plan          Blocking
     Indexes                 I/O
     Statistics              Locks
     Estimates               Memory
     SARGability             Concurrency
          |                       |
          +-----------+-----------+
                      |
                 ROOT CAUSE
                      |
                    CHANGE
                      |
                  MEASURE
                      |
                  VALIDATE
```

---

# 68. Golden rule

> **Never optimize what you haven't measured.**

And remember:

> **The fastest query is often the query that doesn't need to run, the row that doesn't need to be read, or the round trip that doesn't need to happen.**

That is the advanced DBA mindset: **optimize the workload, not just the database.**

---

## Next — Lesson 104: Azure SQL Managed Instance Automation & Operational Engineering

We'll cover **PowerShell/Azure CLI automation, infrastructure-as-code concepts, scheduled DBA tasks, automated health checks, backup/restore automation, alert-driven remediation, maintenance automation, runbooks, idempotency, safe automation, and building a production DBA automation framework.**
