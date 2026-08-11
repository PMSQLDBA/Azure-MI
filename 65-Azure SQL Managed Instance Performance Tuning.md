# Lesson 97 — Azure SQL Managed Instance Performance Tuning

This is one of the most important DBA lessons.

Monitoring tells you:

> **Something is slow.**

Performance tuning answers:

> **Why is it slow, and what is the safest way to make it faster?**

The most important rule:

> **Never tune from symptoms alone. Find the bottleneck first.**

---

# 1. Performance tuning framework

Use this sequence:

```text
Slow workload
    ↓
Scope
    ↓
Measure
    ↓
Identify bottleneck
    ↓
Find root cause
    ↓
Apply targeted fix
    ↓
Measure again
```

Don't jump straight to:

```text
Add index
Increase CPU
Restart SQL
Update statistics
```

without evidence.

---

# 2. What does "slow" mean?

Different users may mean different things:

```text
Application response = slow
Query duration = slow
Report = slow
Database = slow
Login = slow
Batch = slow
```

First determine exactly what is slow.

---

# 3. Performance dimensions

Think about:

```text
CPU
Memory
I/O
Locks
Concurrency
Network
Query design
Indexes
Statistics
Application behavior
```

A query can be slow for completely different reasons.

---

# 4. CPU-bound workload

Example:

```text
CPU = 95%
I/O = normal
Blocking = low
```

Possible causes:

```text
Expensive query
Poor execution plan
Large scans
Complex calculations
High parallelism
Too much workload
```

---

# 5. I/O-bound workload

Example:

```text
CPU = 30%
I/O latency = high
```

Potential causes:

```text
Large table scans
Missing/inefficient indexes
Poor data access pattern
Insufficient I/O capacity
```

---

# 6. Lock-bound workload

Example:

```text
CPU = 30%
I/O = normal
Blocking = severe
```

Possible cause:

```text
Long-running transaction
```

The query isn't necessarily computationally expensive.

It's waiting.

---

# 7. Wait statistics

Waits provide clues about what SQL Server is waiting on.

Think:

```text
Query
 ↓
Needs resource
 ↓
Can't get it immediately
 ↓
WAIT
```

The wait category helps identify the bottleneck.

---

# 8. Don't optimize waits blindly

A wait statistic isn't automatically a problem.

For example:

```text
High wait
```

doesn't mean:

```text
Bad query
```

You need context:

```text
Which workload?
How much?
During what period?
Is application performance affected?
```

---

# 9. Query Store

Query Store should be one of your first tools for historical query analysis.

It helps identify:

```text
High CPU queries
High duration queries
High I/O queries
Execution-count-heavy queries
Plan regressions
```

---

# 10. Query Store investigation

Suppose users complain at:

```text id="1aj47s"
14:00
```

Look at the same period.

Find:

```text id="e3k9c1"
Queries with increased duration
Queries with increased CPU
Plan changes
```

Now you can compare before vs after.

---

# 11. Execution plans

An execution plan shows how SQL Server intends to execute a query.

Conceptually:

```text id="q4n7m2"
Query
 ↓
Parse
 ↓
Optimize
 ↓
Execution plan
 ↓
Operators
 ↓
Results
```

The plan is one of the most important tuning artifacts.

---

# 12. Scan vs seek

One of the most famous DBA concepts.

### Index seek

SQL Server can navigate efficiently to relevant rows.

```text id="x7m3p8"
Index
 ↓
Target rows
```

### Scan

SQL Server reads a much larger portion of an object.

```text id="v2k8c5"
Table/index
 ↓
Many/all rows
```

But:

> **A scan is not automatically bad.**

Scanning a tiny table can be perfectly appropriate.

---

# 13. Example

Table:

```text id="c8r5m1"
Customers = 1 million rows
```

Query:

```sql id="3qf6s9"
SELECT *
FROM Customers
WHERE CustomerID = 12345;
```

If `CustomerID` is indexed, an efficient seek may be appropriate.

---

# 14. Another example

Table:

```text id="n7p2x4"
Customers = 100 rows
```

Query:

```sql id="b5m9k3"
SELECT *
FROM Customers;
```

A scan is completely reasonable.

Don't optimize plans based on operator names alone.

---

# 15. Look for expensive operators

Potentially important operators include:

```text id="r4x8c2"
Sort
Hash Match
Large scans
Key Lookup
Nested Loops with huge row counts
Spills
Parallelism
```

But again:

> **Operator cost must be interpreted in context.**

---

# 16. Key Lookup

A common pattern:

```text id="k6m3q9"
Index Seek
    ↓
Key Lookup
    ↓
Base table
```

If thousands or millions of lookups occur, performance may suffer.

Possible solution:

```text id="x2v7n5"
Better index
Included columns
Query redesign
```

But don't automatically create a covering index without considering write/storage costs.

---

# 17. Covering index

Suppose query:

```sql id="j8c4p2"
SELECT CustomerName, City
FROM Customers
WHERE CustomerID = @id;
```

An index that contains the search key and required output columns may reduce lookups.

Conceptually:

```text id="m5r7x3"
Index
 ├── CustomerID
 ├── CustomerName
 └── City
```

---

# 18. Index trade-off

Every index has costs.

```text id="w3n8k6"
More indexes
   ↓
Faster some reads
   +
More storage
   +
More write overhead
   +
More maintenance
```

The goal isn't:

> "Maximum number of indexes."

The goal is:

> **The right indexes for the workload.**

---

# 19. SARGability

This is a very important performance concept.

A predicate is generally more efficient when SQL Server can use an index effectively.

Bad pattern:

```sql id="p7q3m1"
WHERE YEAR(OrderDate) = 2026
```

Potentially better:

```sql id="v8c5n2"
WHERE OrderDate >= '20260101'
  AND OrderDate <  '20270101'
```

Why?

The second form can allow more efficient range access.

---

# 20. Functions on columns

Another common problem:

```sql id="n4k8r6"
WHERE LOWER(CustomerName) = 'alice'
```

Depending on collation/index design, applying functions to indexed columns can make efficient index access harder.

Always examine the actual plan.

---

# 21. Implicit conversions

Suppose:

```text id="x6m2p9"
Column = INT
Parameter = VARCHAR
```

SQL Server may need to convert one side.

This can cause:

```text id="r3q7v5"
Extra CPU
Poor cardinality estimates
Index access problems
```

Keep data types aligned.

---

# 22. Example

Bad:

```sql id="f8m4k2"
WHERE CustomerID = '12345'
```

when:

```text id="c5n7x3"
CustomerID = INT
```

Better:

```sql id="z2q9m6"
WHERE CustomerID = 12345
```

The exact behavior depends on context, but avoiding unnecessary implicit conversions is good practice.

---

# 23. Parameter sniffing

This is an important SQL Server performance concept.

Suppose a stored procedure:

```sql id="a7k3m8"
CREATE PROCEDURE GetOrders
    @CustomerID INT
AS
...
```

SQL Server may compile a plan using the initial parameter value.

That plan might be excellent for one customer but poor for another.

---

# 24. Example

Customer A:

```text id="d8r4p1"
1,000,000 orders
```

Customer B:

```text id="m6x9c2"
5 orders
```

A plan optimized for B might be poor for A.

A plan optimized for A might be excessive for B.

This is one form of parameter-sensitivity problem.

---

# 25. Symptoms

You may see:

```text id="q5m8x3"
Query sometimes fast
Query sometimes slow
Same query
Different parameters
Different execution behavior
```

This is a clue.

---

# 26. Parameter sniffing investigation

Check:

```text id="v7c2n9"
Query Store
Execution plans
Parameter values
Duration variation
CPU variation
```

Don't immediately use a hint without understanding the root cause.

---

# 27. Cardinality estimation

SQL Server needs to estimate:

> **How many rows will this operation produce?**

Suppose actual:

```text id="k3x8m4"
1,000,000 rows
```

but estimated:

```text id="n7p2q5"
10 rows
```

That's a huge estimation error.

---

# 28. Why bad estimates matter

The optimizer might choose:

```text id="p5m9c2"
Nested Loops
```

when:

```text id="r8x3v6"
Hash Join
```

would be more appropriate for a large result set.

So:

```text id="w4k7n1"
Bad estimate
 ↓
Bad plan
 ↓
Poor performance
```

---

# 29. Statistics

Statistics help SQL Server estimate data distribution.

They influence:

```text id="x2n8m5"
Join choices
Index choices
Join order
Cardinality estimates
```

---

# 30. Stale statistics

If data changes significantly:

```text id="c7p4m9"
Old statistics
      ↓
New data distribution
      ↓
Bad estimates
      ↓
Poor plan
```

Statistics maintenance and automatic statistics behavior are therefore important.

---

# 31. Don't blindly update all statistics

Running:

```text id="z5m2k8"
UPDATE STATISTICS everywhere
```

is not a universal solution.

It can introduce:

```text id="r9x3v7"
Compilation overhead
Resource usage
Plan changes
Unnecessary work
```

Use evidence.

---

# 32. Joins

Poor joins can be expensive.

Example:

```sql id="q4v8m2"
SELECT *
FROM Orders o
JOIN Customers c
  ON o.CustomerID = c.CustomerID;
```

Important considerations:

```text id="m6x3k9"
Join columns
Indexes
Cardinality
Data types
Filtering
Result size
```

---

# 33. Filter early

Compare conceptually:

```text id="j8r2p5"
Join huge datasets
       ↓
Filter later
```

versus:

```text id="n3k7x4"
Filter relevant rows
       ↓
Join smaller datasets
```

The optimizer may transform queries itself, but writing clear, selective predicates helps.

---

# 34. SELECT *

Avoid:

```sql id="v5m9q3"
SELECT *
FROM Orders;
```

when you only need:

```sql id="x2c7n8"
OrderID
OrderDate
Amount
```

Use:

```sql id="f4k6r1"
SELECT OrderID, OrderDate, Amount
FROM Orders;
```

Benefits can include:

```text id="p8m3v7"
Less data
Less I/O
Less network traffic
Potentially narrower indexes
```

---

# 35. Large result sets

Sometimes the query is fast but the application is slow because it retrieves:

```text id="q7n4m2"
Millions of rows
```

Ask:

> Does the application actually need all those rows?

Pagination may help:

```text id="z3x8c5"
Page 1
Page 2
Page 3
```

---

# 36. TempDB

TempDB is heavily used for temporary operations.

Potential workloads include:

```text id="r6m2k8"
Sorts
Hash operations
Temporary tables
Versioning
Intermediate results
```

Monitor TempDB-related pressure when diagnosing performance.

---

# 37. Memory grants

Queries may request memory for operations such as:

```text id="c4n7p9"
Sort
Hash
```

If estimates are wrong:

```text id="x8m3q6"
Too little memory
 ↓
Spill to disk
 ↓
Slow query
```

Or:

```text id="j5r2v7"
Too much memory requested
 ↓
Memory pressure
```

---

# 38. Spills

A spill can occur when an operation doesn't have enough memory and uses TempDB.

Potential indicators:

```text id="m7k4c2"
Sort spill
Hash spill
TempDB activity
```

Investigate:

```text id="p9x3n6"
Cardinality estimates
Memory grants
Query shape
Indexes
```

---

# 39. Parallelism

SQL Server may execute a query using multiple CPU threads.

Parallelism isn't inherently bad.

It can help large queries.

But excessive or inefficient parallelism can contribute to:

```text id="v5q8m2"
CPU pressure
Coordination overhead
Resource contention
```

Evaluate it based on workload evidence.

---

# 40. Don't blindly change MAXDOP

A common weak tuning approach:

> "Parallelism looks scary. Change MAXDOP."

Instead:

```text id="k3m7p9"
Identify query
 ↓
Understand plan
 ↓
Understand CPU/waits
 ↓
Determine whether parallelism is actually problematic
 ↓
Then consider configuration/query-level changes
```

---

# 41. Query timeout

Application reports:

```text id="x8r2m5"
SQL timeout
```

This doesn't necessarily mean:

```text id="q6v9c3"
SQL Server is down.
```

Possible causes:

```text id="m4k7n2"
Blocking
Slow query
Network delay
Application timeout setting
Resource saturation
Deadlock/retry behavior
```

---

# 42. Blocking vs slow query

Suppose:

```text id="n5p3x8"
Query execution time = 50 ms
```

but:

```text id="j7m2c4"
Waited behind another transaction = 30 seconds
```

The user sees:

```text id="v9k6r1"
30-second response
```

The query itself isn't computationally expensive.

It's blocked.

---

# 43. Performance tuning hierarchy

Use this order:

```text id="c5m8x2"
1. Understand workload
2. Identify bottleneck
3. Fix query/application design
4. Fix indexes/statistics
5. Fix configuration
6. Scale resources if necessary
```

Scaling should not automatically be step one.

---

# 44. Scale-up vs tune

### Scale-up

Useful when:

```text id="r7n2k5"
Workload genuinely exceeds capacity
```

### Tune

Useful when:

```text id="x3m8p1"
Poor query
Bad plan
Missing index
Blocking
Bad application pattern
```

Sometimes both are needed.

---

# 45. Example

Current:

```text id="f6q2m8"
SQL MI CPU = 90%
```

Investigation finds:

```text id="p3x7c9"
One query consumes 65% CPU
```

Fixing the query reduces:

```text id="n8m4r2"
CPU = 90% → 45%
```

No scaling required.

---

# 46. Another example

Current:

```text id="v5k8m1"
CPU = 90%
```

Investigation:

```text id="j2q6x9"
Queries are well optimized
No blocking
No obvious regressions
Workload has doubled
```

Now scaling may be appropriate.

---

# 47. Performance baseline

Before tuning:

```text id="m8c3p7"
Average duration = 800 ms
CPU = 75%
```

After:

```text id="r4x7n2"
Average duration = 150 ms
CPU = 35%
```

Now you can demonstrate measurable improvement.

---

# 48. Tuning without measurement

Bad:

> "I added three indexes and it feels faster."

Good:

```text id="k6m2v9"
Before:
CPU 80%
Duration 1.2 sec

After:
CPU 45%
Duration 300 ms
```

Performance engineering needs evidence.

---

# 49. Complete slow-query exercise

Application reports:

> "Customer search became slow today."

---

## Step 1 — Scope

Only customer search is slow.

Other application functions work normally.

So:

```text id="x5n8c2"
Likely workload-specific
```

---

## Step 2 — Timeline

Yesterday:

```text id="p7m3k9"
200 ms
```

Today:

```text id="v4c8r2"
4 seconds
```

Something changed.

---

## Step 3 — Query Store

Query Store shows:

```text id="m9x2q6"
Same query
New execution plan
```

---

## Step 4 — Plan comparison

Old:

```text id="j6k3n8"
Index Seek
```

New:

```text id="q2v7p5"
Table Scan
```

---

## Step 5 — Why?

Check:

```text id="r8m4c1"
Statistics
Index
Data type
Predicate
Recent schema changes
```

Suppose:

```text id="z5x7m2"
Column data type changed in application
```

Application now passes:

```text id="n3k8q6"
VARCHAR
```

while database column is:

```text id="c7m4p9"
INT
```

---

## Step 6 — Root cause

```text id="w2r6k8"
Implicit conversion
 ↓
Poor index usage
 ↓
Table scan
 ↓
Slow query
```

---

## Step 7 — Fix

Correct the application parameter type.

Then validate:

```text id="p4n7x3"
Index usage restored
Duration reduced
CPU reduced
```

---

# 50. Performance troubleshooting checklist

```text id="m8x3q7"
☐ Define the symptom
☐ Determine scope
☐ Establish timeline
☐ Check CPU
☐ Check memory
☐ Check I/O
☐ Check waits
☐ Check blocking
☐ Check deadlocks
☐ Check Query Store
☐ Compare execution plans
☐ Check statistics
☐ Check indexes
☐ Check implicit conversions
☐ Check parameter sensitivity
☐ Check recent changes
☐ Fix root cause
☐ Measure again
```

---

# 51. Interview question

### "A query is slow. What do you check first?"

Strong answer:

> "First I establish the scope and timeline, then check resource utilization, waits, blocking, and Query Store. I identify the specific query and execution plan before deciding whether the issue is caused by indexing, statistics, query design, parameter sensitivity, contention, or resource capacity."

---

# 52. Interview question

### "Is a table scan always bad?"

> "No. A scan can be appropriate for small tables or queries that need a large percentage of the rows. The correct approach is to evaluate the scan in the context of table size, selectivity, cost, and overall workload."

---

# 53. Interview question

### "What causes parameter sniffing?"

> "SQL Server can compile a plan using the initial parameter values supplied to a parameterized query or stored procedure. If data distribution varies significantly between parameter values, that cached plan may perform poorly for other values."

---

# 54. Interview question

### "Why are statistics important?"

> "Statistics provide the optimizer with information about data distribution, helping it estimate row counts and choose appropriate join strategies, indexes, and execution plans."

---

# 55. Interview question

### "When would you scale SQL MI instead of tuning a query?"

> "When investigation shows the workload is fundamentally exceeding available capacity and the queries, indexes, statistics, and application behavior are already reasonably optimized. Scaling can also be appropriate as an immediate capacity measure while a longer-term tuning solution is developed."

---

# 56. The golden tuning model

Memorize:

```text id="v7c4m8"
                PERFORMANCE PROBLEM
                         |
                    DEFINE SCOPE
                         |
                      MEASURE
                         |
              +----------+----------+
              |          |          |
             CPU        I/O       LOCKS
              |          |          |
              +----------+----------+
                         |
                       WAITS
                         |
                    QUERY STORE
                         |
                  EXECUTION PLAN
                         |
             +-----------+-----------+
             |           |           |
          Query        Index      Statistics
          Design                    |
             |           |           |
             +-----------+-----------+
                         |
                     ROOT CAUSE
                         |
                      FIX
                         |
                      MEASURE
```

And the rule to remember:

> **Tune the bottleneck, not the metric.**

---

## Next — Lesson 98: Azure SQL Managed Instance Backup, Restore & Point-in-Time Recovery — Advanced DBA

We'll go deeper into **full/differential/log recovery concepts, PITR, long-term retention, backup validation, restore testing, accidental deletion, corruption scenarios, recovery decision trees, RPO/RTO trade-offs, and a complete production database recovery exercise.**
