# Lesson 118 — Azure SQL Managed Instance Performance Engineering — Expert Level

The goal is to move from:

> **“The query is slow.”**

to:

> **“We can identify the bottleneck, prove its cause, change one variable, measure the result, and prevent regression.”**

---

# 1. Performance engineering is not guesswork

A production performance investigation should follow:

```text
Observe
  ↓
Measure
  ↓
Hypothesize
  ↓
Test
  ↓
Change
  ↓
Measure again
  ↓
Validate
```

Avoid:

```text
"CPU is high → add more CPU."
```

That may treat the symptom rather than the cause.

---

# 2. The performance equation

Think of database performance as:

```text
Performance
=
Query efficiency
+
Data access efficiency
+
Concurrency
+
Resource capacity
+
Application behavior
```

A slow database isn't necessarily an under-sized database.

---

# 3. First question: what is actually slow?

Separate:

```text
Application latency
Database execution time
Network latency
Connection acquisition
Lock waiting
CPU execution
I/O waiting
```

Example:

```text
App response = 5 seconds

Database query = 100 ms
```

The database may not be the bottleneck.

---

# 4. End-to-end latency

A useful mental model:

```text
User
 ↓
Frontend
 ↓
API
 ↓
Connection pool
 ↓
SQL MI
 ↓
Query
 ↓
Storage
```

Measure each layer.

---

# 5. Establish a baseline

Before optimizing, capture:

```text
CPU
Memory pressure
I/O
Query duration
Query CPU time
Logical reads
Physical reads
Waits
Concurrency
Throughput
```

Without a baseline, you cannot prove improvement.

---

# 6. The three big resource dimensions

Most database performance investigations eventually involve:

```text
CPU
I/O
Concurrency
```

But memory, network, and configuration can also be critical.

---

# 7. CPU-bound workload

Symptoms:

```text
High CPU
High query CPU time
Queries doing expensive computation
```

Potential causes:

```text
Bad execution plan
Missing index
Large scans
Expensive joins
Scalar functions
Poor cardinality estimates
```

---

# 8. I/O-bound workload

Symptoms:

```text
High data reads
Slow storage access
Large scans
Queries retrieving unnecessary data
```

Potential causes:

```text
Missing/inefficient indexes
Poor query predicates
Large result sets
Table scans
Data locality
```

---

# 9. Concurrency-bound workload

Symptoms:

```text
Queries are individually fast
But users experience long waits
```

Potential causes:

```text
Blocking
Lock contention
Resource contention
Connection pressure
Worker/thread contention
```

---

# 10. Query execution lifecycle

Conceptually:

```text
SQL text
 ↓
Parse
 ↓
Bind/resolve
 ↓
Optimize
 ↓
Execution plan
 ↓
Execute
 ↓
Return rows
```

Performance problems can arise at multiple stages.

---

# 11. The optimizer

SQL Server's optimizer attempts to choose an efficient execution plan.

It considers:

```text
Query structure
Statistics
Indexes
Data distribution
Estimated costs
Available physical operators
```

---

# 12. Estimated vs actual plan

This distinction is crucial.

### Estimated plan

```text
What optimizer expected
```

### Actual plan

```text
What actually happened
```

Comparing them can reveal cardinality estimation problems.

---

# 13. Cardinality estimation

Suppose the optimizer estimates:

```text
100 rows
```

but execution produces:

```text
10,000,000 rows
```

That's a massive estimation error.

It can lead to poor choices such as:

```text
Nested loops
Small memory grant
Repeated lookups
```

when another strategy would have been better.

---

# 14. Execution plan operators

Common operators include:

```text
Index Seek
Index Scan
Table Scan
Nested Loops
Hash Match
Merge Join
Sort
Key Lookup
Aggregate
Filter
```

Learn what each means rather than memorizing "good" and "bad" labels.

---

# 15. Index Seek

A seek navigates to a relevant portion of an index.

Conceptually:

```text
Index
 ↓
Locate matching range
 ↓
Read required entries
```

Often efficient for selective predicates.

---

# 16. Index Scan

A scan reads many or all index pages.

That isn't automatically bad.

For example:

```text
SELECT COUNT(*)
FROM LargeTable;
```

may legitimately require a large amount of data to be read.

---

# 17. Scan ≠ automatically bad

A common beginner mistake:

> "I see an Index Scan, therefore the query is slow."

Not necessarily.

Ask:

```text
How many rows?
How many pages?
How selective is the query?
What alternative exists?
What is the total cost?
```

---

# 18. Key Lookup

A common pattern:

```text
Index Seek
   ↓
Key Lookup
   ↓
Base table
```

If only a few rows are returned, this can be efficient.

If millions of rows require lookups:

```text
Seek
 ↓
Millions of lookups
```

performance can degrade dramatically.

---

# 19. Covering index

An index can include the columns required by the query.

Conceptually:

```text
Index key:
CustomerId

Included:
Name
Status
CreatedDate
```

Then the query may retrieve everything from the index without repeatedly accessing the base table.

---

# 20. Composite indexes

Suppose the query frequently uses:

```sql
WHERE TenantId = ?
  AND Status = ?
  AND CreatedDate >= ?
```

A composite index may be appropriate.

But ordering matters.

```text
(TenantId, Status, CreatedDate)
```

is not equivalent to:

```text
(Status, CreatedDate, TenantId)
```

---

# 21. Index design principle

Index based on workload, not intuition.

Ask:

```text
What queries run?
What predicates are used?
What joins occur?
What columns are returned?
How selective are predicates?
How often is the table modified?
```

---

# 22. Too many indexes

Indexes improve reads but create costs.

Every insert/update/delete may need index maintenance.

Therefore:

```text
More indexes
≠
Always faster
```

You must balance:

```text
Read performance
vs
Write overhead
vs
Storage
vs
Maintenance
```

---

# 23. Statistics

Statistics describe data distribution to the optimizer.

They help answer:

> **How many rows will this predicate probably return?**

Example:

```text
Status = 'Active'
```

If 99% of rows are active, that's very different from:

```text
Status = 'Cancelled'
```

if only 0.1% are cancelled.

---

# 24. Stale statistics

Data changes over time.

Therefore statistics can become less representative of current data.

Potential consequence:

```text
Bad estimate
 ↓
Bad plan
 ↓
Poor performance
```

Statistics maintenance is therefore part of performance engineering.

---

# 25. Parameter sensitivity

Consider:

```sql
WHERE CustomerId = @CustomerId
```

If some customers have:

```text
10 rows
```

while another has:

```text
10,000,000 rows
```

a plan optimized for one value may perform poorly for another.

This is a classic parameter-sensitivity scenario.

---

# 26. Why plan reuse can hurt

SQL Server often reuses compiled plans.

That's usually good.

But:

```text
Parameter A
 ↓
Plan optimized for A
 ↓
Plan reused for Parameter B
 ↓
B has radically different data distribution
 ↓
Poor performance
```

---

# 27. Parameter-sensitive plan optimization

Modern SQL Server versions include mechanisms intended to handle certain parameter-sensitive workloads more effectively.

The important engineering lesson is:

> **Don't automatically assume recompilation is the answer.**

First understand:

```text
Data distribution
Plan choice
Parameter values
SQL Server version/compatibility level
Available optimization features
```

---

# 28. Query Store

Query Store is one of the most important tools for performance engineering.

Conceptually:

```text
Queries
 ↓
Plans
 ↓
Runtime statistics
 ↓
Historical comparison
```

It helps answer:

> **What changed?**

---

# 29. Plan regression

Suppose:

```text
Monday:
Query = 100 ms
```

Then:

```text
Tuesday:
Query = 5 seconds
```

Query Store can help determine whether the execution plan changed.

---

# 30. Performance regression workflow

```text
Regression detected
       ↓
Find query
       ↓
Compare plans
       ↓
Compare runtime statistics
       ↓
Identify changed factor
       ↓
Test remediation
       ↓
Monitor
```

---

# 31. Don't immediately force plans

Plan forcing can be useful as a tactical mitigation.

But:

```text
Bad plan
 ↓
Force old plan
```

is not always a permanent solution.

The workload may evolve.

Investigate why the optimizer selected the new plan.

---

# 32. Wait statistics

Waits answer:

> **What is the session waiting for?**

A useful conceptual model:

```text
Running
  ↓
Waiting
  ↓
Resource becomes available
  ↓
Continue
```

---

# 33. Common wait categories

Examples include waits associated with:

```text
Locks
I/O
CPU scheduling
Memory grants
Parallelism
Networking
```

Don't memorize wait names without understanding the underlying resource.

---

# 34. Blocking

Suppose:

```text
Transaction A
 ↓
Holds lock

Transaction B
 ↓
Needs same resource
 ↓
WAIT
```

The database may appear "slow," but the query isn't necessarily consuming CPU.

It's waiting.

---

# 35. Blocking chain

Imagine:

```text
Session 51
   ↓ blocks
Session 72
   ↓ blocks
Session 93
   ↓ blocks
Session 104
```

The first blocker can create a large downstream performance problem.

---

# 36. Long transactions

Long transactions can hold locks for extended periods.

Potential consequences:

```text
Blocking
Version-store pressure
Delayed cleanup
Higher resource usage
```

Transactions should be appropriately scoped.

---

# 37. Isolation levels

Isolation controls visibility and concurrency behavior.

Common concepts:

```text
Read Uncommitted
Read Committed
Repeatable Read
Serializable
Snapshot-based isolation
```

Higher isolation can provide stronger consistency but may affect concurrency depending on implementation.

---

# 38. Snapshot-based concurrency

Row-versioning approaches can reduce some reader/writer blocking scenarios.

Conceptually:

```text
Writer
 ↓
Changes data

Reader
 ↓
Reads appropriate version
```

But row versioning also consumes resources.

Don't enable a feature without understanding its workload impact.

---

# 39. Deadlocks

A deadlock occurs when sessions wait on each other.

Example:

```text
Transaction A
locks Row 1
 ↓
needs Row 2

Transaction B
locks Row 2
 ↓
needs Row 1
```

Neither can proceed.

SQL Server detects the deadlock and chooses a victim.

---

# 40. Preventing deadlocks

Common strategies:

```text
Consistent access order
Shorter transactions
Appropriate indexes
Reduced lock duration
Appropriate isolation design
```

---

# 41. Memory grants

Queries sometimes request memory for operations such as:

```text
Sort
Hash join
Hash aggregate
```

If estimates are wrong:

```text
Underestimate
 ↓
Too little memory
 ↓
Spills
```

or:

```text
Overestimate
 ↓
Large memory grant
 ↓
Other queries wait
```

---

# 42. Tempdb spills

A sort/hash operation may spill to tempdb when it can't execute entirely in memory.

Conceptually:

```text
Query
 ↓
Needs memory
 ↓
Insufficient grant
 ↓
Tempdb spill
 ↓
Additional I/O
 ↓
Slower query
```

---

# 43. Tempdb is important

Tempdb supports many database operations.

Potential workloads include:

```text
Temporary objects
Sorts
Hash operations
Row versioning
Intermediate results
```

Therefore tempdb behavior can affect overall performance.

---

# 44. Parallelism

Large queries may use multiple CPU workers.

Conceptually:

```text
Query
 ↓
Parallel plan
 ├── Worker
 ├── Worker
 ├── Worker
 └── Worker
```

Parallelism can improve large workloads.

But excessive parallelism can create CPU pressure and coordination overhead.

---

# 45. Parallelism is not inherently bad

Again:

```text
Parallel plan
≠
Bad plan
```

Ask:

```text
Why is it parallel?
How much CPU?
Is it improving elapsed time?
Are workers waiting?
```

---

# 46. CPU saturation

If CPU is consistently near capacity:

Possible causes include:

```text
Poor query plans
Missing indexes
High workload volume
Inefficient code
Excessive parallelism
Under-sized compute
```

Only one of these is:

> "We need more compute."

---

# 47. Scaling compute

Scaling can be appropriate when:

```text
Workload is genuinely CPU-bound
Queries are already reasonably optimized
Demand has increased
```

But scale **after** validating the workload.

---

# 48. Vertical scaling

Conceptually:

```text
Smaller compute
      ↓
Larger compute
```

You gain more resource capacity.

The benefit depends on the specific SQL MI service tier/configuration and workload characteristics.

---

# 49. Horizontal scaling

For some workloads, distribute work across multiple databases or services.

Examples conceptually:

```text
Primary OLTP
     +
Reporting workload
```

or:

```text
Tenant A → Database A
Tenant B → Database B
```

But sharding introduces significant application complexity.

---

# 50. Read workload separation

If reporting consumes significant resources:

```text
OLTP workload
+
Reporting workload
```

can compete.

Possible architecture:

```text
Transactional database
        ↓
Read/analytics architecture
```

Choose the appropriate supported SQL/Azure architecture rather than forcing all workloads onto one instance.

---

# 51. Connection pooling

Creating database connections repeatedly can be expensive.

Bad pattern:

```text
Request
 ↓
Open connection
 ↓
Query
 ↓
Close
```

for every tiny operation without pooling.

Better:

```text
Application
 ↓
Connection pool
 ↓
Reusable connections
```

---

# 52. Too many connections

Connection pooling doesn't mean:

> "Unlimited connections."

Excessive concurrency can cause:

```text
Resource pressure
Worker pressure
Memory pressure
Lock contention
```

Pool sizing should match workload characteristics.

---

# 53. Chatty applications

Bad application pattern:

```text
Request
 ↓
Query 1
 ↓
Query 2
 ↓
Query 3
 ↓
Query 4
 ↓
...
```

A single logical operation may generate dozens or hundreds of round trips.

Reduce unnecessary database chatter.

---

# 54. N+1 query problem

Example:

```text
SELECT customers
```

then:

```text
SELECT orders WHERE customer_id = 1
SELECT orders WHERE customer_id = 2
SELECT orders WHERE customer_id = 3
...
```

This can produce massive query volume.

Look for set-based alternatives.

---

# 55. Set-based thinking

SQL databases are optimized around sets.

Prefer conceptually:

```text
Process 10,000 rows in a set
```

over:

```text
Loop 10,000 times
  ↓
Execute query
```

when the operation can safely be expressed set-wise.

---

# 56. SARGability

A predicate is generally more index-friendly when the indexed column can be used directly.

Less desirable pattern:

```sql
WHERE YEAR(OrderDate) = 2026
```

Potentially better:

```sql
WHERE OrderDate >= '20260101'
  AND OrderDate <  '20270101'
```

The optimizer can often use the range more effectively.

---

# 57. Functions on indexed columns

Be cautious with:

```sql
WHERE UPPER(Name) = ...
```

or:

```sql
WHERE CONVERT(...) = ...
```

depending on the data type and query.

Applying transformations can make index usage less efficient.

---

# 58. Implicit conversions

Suppose:

```text
Column = VARCHAR
Parameter = NVARCHAR
```

SQL Server may need a conversion.

This can sometimes produce:

```text
Poor index usage
Higher CPU
Unexpected plans
```

Data types should be aligned deliberately.

---

# 59. SELECT *

Avoid:

```sql
SELECT *
```

when the application only needs:

```text
CustomerId
Name
Status
```

Returning unnecessary columns can increase:

```text
I/O
Network traffic
Memory
CPU
```

---

# 60. Large result sets

Sometimes the query itself is fine but the application requests:

```text
10 million rows
```

Ask:

> **Does the user actually need 10 million rows?**

Use:

```text
Pagination
Filtering
Aggregation
Streaming
Export-specific workflows
```

where appropriate.

---

# 61. Pagination

Avoid inefficient pagination patterns on huge datasets where possible.

A query repeatedly doing:

```sql
OFFSET 1000000 ROWS
```

may need to process a large amount of preceding data.

Keyset/seek pagination can be more efficient for suitable workloads.

---

# 62. Data model matters

Performance isn't only about indexes.

Poor schema design can create:

```text
Huge joins
Repeated transformations
Redundant data
Complex predicates
```

Good logical modeling can simplify query execution.

---

# 63. Partitioning

Partitioning can help manage very large tables by dividing data into logical physical partitions.

Conceptually:

```text
Orders
 ├── 2024
 ├── 2025
 ├── 2026
 └── 2027
```

But partitioning is not a universal performance solution.

---

# 64. Partition elimination

The biggest performance benefit often comes when a query can access only relevant partitions.

Example:

```text
Query:
Orders from 2026
```

Instead of scanning:

```text
2024 + 2025 + 2026 + 2027
```

it may be possible to target:

```text
2026
```

---

# 65. Partitioning isn't indexing

They solve different problems.

```text
Index
→ efficient access path

Partition
→ divide large data set
```

They can complement each other.

---

# 66. Compression

Data compression can reduce storage footprint and I/O.

But it may trade:

```text
Less I/O
vs
Additional CPU
```

Evaluate against the workload.

---

# 67. Columnstore

Columnstore technologies can be highly effective for analytical workloads.

Conceptually:

```text
OLTP:
row-oriented access

Analytics:
column-oriented access
```

For aggregation-heavy workloads over large datasets, columnstore can dramatically change performance characteristics.

---

# 68. Don't use OLTP design for analytics blindly

A classic mistake:

```text
Operational tables
       ↓
Huge reporting queries
       ↓
Production workload slows
```

Separate workload patterns when necessary.

---

# 69. Query optimization workflow

When a query is slow:

```text
1. Capture query
2. Capture duration
3. Capture CPU
4. Capture reads
5. Capture execution plan
6. Compare estimates vs actuals
7. Check waits
8. Check indexes/statistics
9. Test change
10. Measure again
```

---

# 70. Example investigation

Suppose:

```text
Query duration = 8 sec
CPU = 1 sec
```

The query is mostly waiting.

Don't immediately optimize CPU.

Now suppose:

```text
Query duration = 8 sec
CPU = 7.5 sec
```

That's a very different problem.

---

# 71. Another example

Suppose:

```text
Logical reads = 50 million
Rows returned = 500
```

That's a strong signal to investigate:

```text
Access path
Indexes
Predicates
Cardinality
```

---

# 72. Another example

Suppose:

```text
Query = 50 ms
```

but users experience:

```text
5 seconds
```

Investigate:

```text
Blocking
Connection acquisition
Application retries
Network
Serialization
Other queries
```

Don't optimize a 50 ms query into 40 ms while ignoring a 5-second application bottleneck.

---

# 73. Performance regression prevention

After fixing a query:

```text
Capture baseline
 ↓
Deploy
 ↓
Monitor
 ↓
Compare
```

Track:

```text
Duration
CPU
Reads
Plan
Throughput
Error rate
```

---

# 74. Deployment effects

Schema changes can alter:

```text
Indexes
Statistics
Cardinality
Execution plans
```

A deployment that works in test may regress in production because the data distribution is different.

---

# 75. Production-like testing

Performance testing should resemble production:

```text
Data volume
Data distribution
Concurrency
Query mix
Hardware/service configuration
```

A 1-million-row test database is not equivalent to a 5-billion-row production workload.

---

# 76. Load testing

Don't test only one query.

Use realistic workload mixes:

```text
70% reads
20% writes
10% reporting
```

or whatever reflects reality.

Measure:

```text
Throughput
p50
p95
p99
Errors
CPU
I/O
Waits
```

---

# 77. Tail latency

Average latency can hide serious problems.

Example:

```text
Average = 100 ms
p99 = 8 seconds
```

Most users are fine.

But a significant tail experiences terrible performance.

Monitor percentiles for user-facing workloads.

---

# 78. Performance budgets

Define acceptable limits:

```text
API p95 < 500 ms
Database query p95 < 200 ms
CPU < 70% sustained
```

Exact targets depend on the workload.

The key is:

> **Make performance measurable.**

---

# 79. Observability stack

A mature performance architecture combines:

```text
Application telemetry
+
SQL Query Store
+
SQL diagnostics
+
Azure monitoring
+
Wait analysis
```

No single dashboard tells the entire story.

---

# 80. Expert interview question

### "A query suddenly became slow. What do you do?"

Strong answer:

> "I'd first determine whether the latency is actually in SQL MI and capture duration, CPU, reads, waits, and the execution plan. I'd compare the current plan and runtime statistics against the previous baseline, investigate cardinality estimates, statistics, indexes, blocking, and parameter sensitivity, then test a targeted remediation and validate the result through measured before-and-after metrics."

---

# 81. Interview question

### "CPU is 95%. Do you scale the instance?"

> "Not immediately. I'd determine whether CPU is caused by increased workload, inefficient queries, a plan regression, missing indexes, excessive parallelism, or another issue. If the workload is already well optimized and demand genuinely exceeds capacity, then scaling compute is appropriate."

---

# 82. Interview question

### "Why is an Index Scan sometimes acceptable?"

> "Because a scan isn't inherently inefficient. If a query needs a large percentage of the table, scanning can be cheaper than many random lookups. I evaluate rows read, rows returned, I/O, CPU, selectivity, and the overall execution plan rather than labeling scans as automatically bad."

---

# 83. Interview question

### "What does Query Store give you?"

> "Historical visibility into query execution, plans, and runtime behavior. It's especially useful for identifying plan changes and performance regressions over time."

---

# 84. Interview question

### "How do you identify blocking?"

Think:

```text
Who is waiting?
        ↓
What are they waiting for?
        ↓
Who is blocking them?
        ↓
Why is the blocker holding the resource?
```

The root cause may be:

```text
Long transaction
Missing index
Application behavior
Poor transaction design
```

---

# 85. Performance maturity model

### Level 1

```text
User reports "slow"
```

### Level 2

```text
Metrics + execution plans
```

### Level 3

```text
Query Store + wait analysis
```

### Level 4

```text
Baselines + automated monitoring
```

### Level 5

```text
Predictive capacity planning
+
Regression detection
+
Continuous workload optimization
```

---

# 86. Golden rule

> **Never optimize based on appearance. Optimize based on measured evidence.**

---

# 87. Final mental model

```text
                  USER LATENCY
                       ↓
                APPLICATION
                       ↓
                CONNECTIONS
                       ↓
                  SQL QUERY
                       ↓
                EXECUTION PLAN
                       ↓
       +---------------+---------------+
       |               |               |
      CPU             I/O         CONCURRENCY
       |               |               |
       +---------------+---------------+
                       ↓
              STATISTICS / INDEXES
                       ↓
                 QUERY STORE
                       ↓
                 WAIT ANALYSIS
                       ↓
                CHANGE + TEST
                       ↓
                   BASELINE
                       ↓
                 MONITORING
```

That is the foundation of **expert Azure SQL Managed Instance performance engineering**.

---

## Next — Lesson 119: Azure SQL Managed Instance Monitoring, Observability & Troubleshooting — Expert Level

We'll build a **production troubleshooting framework** covering **Azure Monitor, Log Analytics, diagnostic settings, metrics, logs, Query Store, DMVs, waits, blocking/deadlocks, alerts, dashboards, KQL investigation, incident triage, root-cause analysis, and designing an end-to-end SQL MI observability platform.**
