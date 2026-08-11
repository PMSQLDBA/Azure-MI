# Lesson 124 — Azure SQL Managed Instance Performance Architecture — Expert Level

The goal of this lesson is to move beyond:

> **“The database is slow.”**

and toward:

> **“I can systematically identify the performance bottleneck, prove its cause with telemetry, fix the correct layer, and demonstrate measurable improvement.”**

---

## 1. Performance is a system property

Don't look at SQL MI in isolation.

```text id="q8v7a2"
User
 ↓
Application
 ↓
Connection pool
 ↓
Network
 ↓
SQL MI
 ↓
Query
 ↓
Storage
```

A slow request doesn't automatically mean SQL MI is the problem.

---

# 2. The performance stack

Think about five major layers:

```text id="4w2s0m"
1. Application
2. Connectivity
3. SQL engine
4. Compute / memory
5. Storage
```

And across all five:

```text id="k2f5n8"
Observability
```

---

# 3. Start with the symptom

Suppose users report:

> "The application is slow."

That's not enough.

Convert it into a measurable statement:

```text id="7c3r5p"
API latency
Normal: 200 ms
Current: 4.2 seconds
```

Now you have something you can investigate.

---

# 4. Separate latency from throughput

### Latency

How long does one operation take?

```text id="n8x2s1"
Query = 2 seconds
```

### Throughput

How much work can the system process?

```text id="m5v7q4"
10,000 requests/minute
```

You can have:

```text id="0g1k6p"
Low latency + low throughput
```

or:

```text id="x4z8b3"
High latency + high throughput
```

They require different analysis.

---

# 5. CPU bottleneck

Example:

```text id="p9j3r0"
CPU
████████████████████ 95%
```

Possible causes:

```text id="w4n8t2"
Expensive queries
Too much concurrency
Poor indexing
Inefficient plans
Large scans
Excessive parallelism
Workload growth
```

Don't immediately conclude:

> "Add more CPU."

First determine **why CPU is being consumed**.

---

# 6. Memory pressure

Memory is critical for SQL workloads because SQL Server uses memory for:

```text id="y6q1v9"
Data pages
Execution plans
Query workspace
Caches
Internal structures
```

Memory pressure can cause increased I/O and degraded query performance.

---

# 7. Storage performance

A query may be CPU-light but storage-heavy.

Conceptually:

```text id="c7t2m5"
Query
 ↓
Needs pages
 ↓
Storage read
 ↓
Latency
```

Investigate:

```text id="u3p6d8"
I/O latency
Read/write workload
Data-file behavior
Log-file behavior
TempDB activity
```

---

# 8. Network latency

Not every slow query is a slow SQL execution.

Example:

```text id="s4f9k2"
SQL execution = 50 ms
Network transfer = 2 seconds
```

The application experiences:

```text id="m1d7x5"
2.05 seconds
```

Always distinguish:

```text id="r8k3v0"
Server execution time
vs
End-to-end request time
```

---

# 9. Query execution time

Break the request down:

```text id="e6n2q1"
Application
   ↓ 20 ms
Network
   ↓ 10 ms
SQL execution
   ↓ 500 ms
Result transfer
   ↓ 100 ms
Application processing
   ↓ 50 ms
Total = 680 ms
```

Now you know where to focus.

---

# 10. Wait statistics

Wait statistics answer:

> **What is SQL Server waiting for?**

Conceptually:

```text id="h5v9c2"
Query
 ↓
CPU?
I/O?
Lock?
Memory?
Network?
Parallelism?
```

They are extremely useful for diagnosing bottlenecks.

---

# 11. CPU-bound workload

A CPU-bound query may show:

```text id="z7m1q8"
High CPU
+
Low I/O wait
+
Expensive execution
```

Potential causes:

* Poor query design
* Missing/inefficient indexes
* Large aggregations
* Complex joins
* Excessive scalar computation

---

# 12. I/O-bound workload

Typical pattern:

```text id="d3k6v1"
High I/O waits
+
Large reads
+
Storage latency
```

Potential causes:

```text id="p8r4y2"
Table scans
Poor indexes
Large data access
Memory pressure
Inefficient plans
```

---

# 13. Blocking-bound workload

Example:

```text id="n4x8c7"
Session A
   |
 holds lock
   ↓
Session B
   |
 waiting
```

CPU could be completely normal while users experience severe latency.

---

# 14. Concurrency

A query that runs in:

```text id="b6m2q9"
100 ms
```

may be fine at:

```text id="g3v8k1"
10 requests/sec
```

but problematic at:

```text id="t5p1r4"
10,000 requests/sec
```

Performance is not only about individual query speed.

It's also about **concurrent workload**.

---

# 15. The concurrency multiplier

Think:

```text id="y8c4s6"
Query cost
   ×
Concurrent executions
   =
Resource demand
```

A small inefficiency multiplied by thousands of requests can become a major incident.

---

# 16. Connection storms

During an application restart:

```text id="u2r7m9"
Application instances restart
       ↓
Thousands of connections
       ↓
SQL MI connection pressure
       ↓
Queries slow
       ↓
Retries begin
       ↓
More connections
```

This creates a feedback loop.

---

# 17. Retry storms

Bad architecture:

```text id="h1d5q8"
Failure
 ↓
Immediate retry
 ↓
Failure
 ↓
Immediate retry
 ↓
Failure
 ↓
Repeat
```

Better:

```text id="c9v3x6"
Failure
 ↓
Backoff
 ↓
Jitter
 ↓
Retry limit
```

Performance engineering must include application behavior.

---

# 18. Query Store

Query Store provides historical workload information.

Think:

```text id="r5m8w2"
Query
 ↓
Execution
 ↓
Plan
 ↓
Historical performance
```

This helps answer:

> "Was this query always slow?"

or:

> "When did its performance change?"

---

# 19. Performance regression

Example:

```text id="f4q2k9"
Monday:
Query = 100 ms

Tuesday:
Query = 110 ms

Wednesday:
Query = 3,000 ms
```

Something changed.

Potential causes:

```text id="p6s1v7"
Execution plan
Statistics
Data distribution
Index
Workload
Schema
Application
```

---

# 20. Baselines

Before tuning, establish normal behavior.

Record:

```text id="a7k3d2"
CPU
Memory
I/O
Latency
Throughput
Connections
Waits
Top queries
```

Then compare:

```text id="e8m5x1"
Current
  vs
Baseline
```

---

# 21. Performance investigation workflow

Use this sequence:

```text id="z2v6n9"
Symptom
 ↓
Measure
 ↓
Identify affected workload
 ↓
Check resource utilization
 ↓
Check waits
 ↓
Inspect queries
 ↓
Inspect execution plans
 ↓
Change one thing
 ↓
Measure again
```

Avoid random tuning.

---

# 22. Query plan analysis

Look for:

```text id="n6r1c5"
Scans
Seeks
Lookups
Sorts
Hash joins
Nested loops
Parallelism
Spills
Cardinality estimates
```

But don't assume any operator is automatically bad.

Context matters.

---

# 23. Table scan

A table scan reads many or all rows/pages.

Potentially appropriate when:

```text id="u9k2m4"
Table is small
Most rows are required
```

Potentially problematic when:

```text id="x5c7p8"
Huge table
Few rows required
High frequency
```

---

# 24. Index seek

A seek can efficiently locate relevant data.

Conceptually:

```text id="w1d8q3"
Index
 ↓
Target range
 ↓
Required rows
```

But a seek isn't automatically faster than every alternative.

---

# 25. Key lookup

A key lookup can occur when SQL Server uses a nonclustered index to find rows and then retrieves additional columns.

One lookup:

```text id="g6y2p4"
Fine
```

Millions of lookups:

```text id="j9r5s7"
Potentially expensive
```

Always consider frequency.

---

# 26. Cardinality estimation

Suppose SQL Server estimates:

```text id="m4n8q2"
100 rows
```

but actual result is:

```text id="v7c1x9"
10,000,000 rows
```

The optimizer may select a poor strategy.

Investigate:

```text id="q5p3k8"
Statistics
Predicates
Data distribution
Parameterization
```

---

# 27. Statistics

Statistics help the optimizer estimate data distribution.

They become particularly important as data changes.

A plan that worked well against:

```text id="d1x6r9"
1 million rows
```

may behave differently after:

```text id="s8m2v4"
500 million rows
```

---

# 28. Parameter sensitivity

A query may perform differently depending on parameter values.

Example:

```text id="k3f7p1"
Customer A → 5 rows
Customer B → 5 million rows
```

One execution plan may not be ideal for both.

Diagnose parameter sensitivity before applying a workaround.

---

# 29. Index strategy

Ask:

```text id="b4n9w2"
Which queries need optimization?
Which predicates are common?
Which joins are expensive?
Which columns are returned?
```

Then design indexes around actual workload.

---

# 30. Over-indexing

Too many indexes cause:

```text id="p2s6d8"
More storage
More writes
More maintenance
More optimizer choices
Potentially higher write latency
```

Therefore:

> Index for workload, not for column count.

---

# 31. Query anti-pattern

Example:

```sql id="q7m3x1"
SELECT *
FROM Orders;
```

If the application needs only:

```text id="z5c8v4"
OrderId
CustomerId
OrderDate
```

retrieving everything increases:

```text id="a1d6p9"
I/O
Network traffic
Memory
Processing
```

Prefer targeted projections.

---

# 32. Pagination

Bad pagination can become expensive:

```sql id="n3x9k2"
ORDER BY OrderDate
OFFSET 1000000 ROWS
FETCH NEXT 50 ROWS ONLY;
```

For very large datasets, appropriate keyset/seek-based pagination may be more efficient.

---

# 33. N+1 query problem

Application:

```text id="v8p2m5"
1 query → customers
1000 queries → customer orders
```

Total:

```text id="j4q7c1"
1001 database calls
```

Even if each query is fast, the overall workload can be expensive.

---

# 34. Set-based processing

Prefer, when appropriate:

```text id="h6m1s8"
One set-based operation
```

over:

```text id="r9t4w3"
Thousands of row-by-row operations
```

SQL engines are optimized for set-based relational processing.

---

# 35. Transaction duration

Long transactions hold resources longer.

Example:

```text id="c2v8p6"
BEGIN TRAN
   ↓
Long application processing
   ↓
UPDATE
   ↓
COMMIT
```

If locks remain held throughout unnecessary application work, blocking can increase.

Keep transactions appropriately scoped.

---

# 36. Deadlocks and performance

Deadlocks aren't merely correctness problems.

Frequent deadlocks create:

```text id="u4n7x2"
Retries
Failed transactions
Extra workload
Latency
User-visible errors
```

Fix the underlying access pattern rather than relying on retries alone.

---

# 37. TempDB performance

TempDB may become important when workloads perform:

```text id="f8q3m1"
Large sorts
Hash operations
Temporary tables
Versioning
Complex queries
```

Investigate TempDB when query behavior suggests temporary workspace pressure.

---

# 38. Parallelism

SQL Server can execute parts of a query concurrently.

Potential benefit:

```text id="w5d2n9"
Large analytical query
 ↓
Multiple workers
 ↓
Faster execution
```

But excessive parallelism can cause:

```text id="k7p4c8"
CPU pressure
Worker contention
Reduced throughput
```

Don't treat parallelism as automatically good or bad.

---

# 39. OLTP vs analytical workloads

### OLTP

```text id="m1q8z4"
Small transactions
High concurrency
Low latency
```

### Analytics

```text id="r6t2y9"
Large scans
Aggregations
Longer execution
High resource consumption
```

Mixing them without workload planning can create contention.

---

# 40. Reporting workload problem

Imagine:

```text id="n8c3x5"
09:00–17:00
OLTP workload
```

Then a report runs:

```text id="p4v7k2"
SELECT huge dataset
JOIN many tables
GROUP BY everything
```

The report may consume resources needed by transactional users.

---

# 41. Workload isolation

Possible architectural approaches include:

```text id="s3m9d6"
Separate reporting architecture
Read-oriented workloads
Scheduling
Application-level throttling
Dedicated compute/resources
```

The right solution depends on the SQL MI/application design.

---

# 42. Performance and security

Security controls can affect performance.

Examples:

```text id="g5x1v8"
Encryption
Auditing
Inspection
Network controls
Identity operations
```

But never disable security simply because:

> "It makes things faster."

Measure first and optimize safely.

---

# 43. Performance and networking

For applications accessing SQL MI through private networking, investigate:

```text id="h9k2m7"
Network path
Latency
DNS
Routing
Connection establishment
Throughput
```

A database can execute quickly while the end-to-end application remains slow.

---

# 44. DNS latency

If connection establishment is slow:

```text id="y6p3q8"
Application
 ↓
DNS resolution
 ↓
Connection
 ↓
Authentication
 ↓
SQL
```

Measure each stage.

Don't automatically blame query execution.

---

# 45. Connection pooling

Connection pooling reduces repeated connection establishment.

But:

```text id="v2m8r5"
Pool too small
→ Requests wait

Pool too large
→ Excessive DB concurrency
```

The correct size depends on workload and application architecture.

---

# 46. Performance testing

A good performance test includes:

```text id="j7x4c9"
Realistic data volume
Realistic concurrency
Realistic query mix
Realistic transactions
Failure scenarios
Peak load
```

A test with:

```text id="s1d6n8"
10 rows
1 user
```

doesn't prove production performance.

---

# 47. Load testing

Measure:

```text id="b8q5m2"
Requests/sec
Queries/sec
P50 latency
P95 latency
P99 latency
Errors
CPU
Memory
I/O
```

Tail latency is especially important.

---

# 48. Why P99 matters

Suppose:

```text id="c3n7x1"
P50 = 100 ms
P95 = 300 ms
P99 = 8 seconds
```

Most users are fine.

But 1% of requests are extremely slow.

For high-volume applications, that can represent thousands of users.

---

# 49. Performance regression testing

After changing:

```text id="z4m8p6"
Index
Query
Schema
SQL configuration
Application version
```

run performance tests again.

Compare:

```text id="u2y5k9"
Before
vs
After
```

---

# 50. Change one variable

Bad experiment:

```text id="e7q1m3"
Change index
+
Change query
+
Change schema
+
Change application
```

Now you don't know what helped.

Better:

```text id="k9r4v8"
Baseline
 ↓
One controlled change
 ↓
Measure
```

---

# 51. Performance incident example

Users report:

> "Checkout takes 5 seconds."

Investigation:

```text id="x3p7c2"
API latency → 5 sec
SQL duration → 4.7 sec
CPU → 40%
I/O → normal
Blocking → low
```

Then Query Store reveals:

```text id="d8m1s6"
New execution plan
```

Now the investigation is focused.

---

# 52. Another performance incident

```text id="q5v9k3"
API latency → 5 sec
SQL duration → 100 ms
```

SQL MI isn't the primary bottleneck.

Investigate:

```text id="r7c2x8"
Application
Network
Serialization
External APIs
Connection setup
```

This is why end-to-end observability matters.

---

# 53. Performance architecture review

Ask:

```text id="n1h6p4"
What is the workload?
What is normal?
What is slow?
Which queries dominate?
What are the waits?
Where is resource pressure?
What changed?
What is the bottleneck?
```

Then act.

---

# 54. Capacity planning

Track trends:

```text id="w8m3q6"
Database size
CPU
Memory
I/O
Connections
Throughput
Latency
```

Example:

```text id="s2k7v5"
CPU:
Jan 40%
Feb 50%
Mar 65%
Apr 78%
```

You can anticipate capacity problems before they become incidents.

---

# 55. Performance SLO

Define measurable targets.

Example:

```text id="p4x9c1"
P95 checkout latency < 500 ms
P99 < 1 second
```

Then monitor against them.

---

# 56. Performance budget

A request might have:

```text id="g6v2n8"
Total budget = 1 second

Application = 200 ms
Network = 100 ms
SQL = 500 ms
Other = 200 ms
```

If SQL grows to:

```text id="u9k5r3"
900 ms
```

the total budget is exceeded.

Performance becomes a resource allocation problem.

---

# 57. Performance optimization hierarchy

Use this order:

```text id="m8q2d5"
1. Fix incorrect behavior
2. Fix inefficient queries
3. Fix data access patterns
4. Fix indexing/statistics
5. Optimize concurrency
6. Optimize resources
7. Scale infrastructure
```

Don't jump directly to scaling.

---

# 58. Scaling isn't always optimization

Suppose a query takes:

```text id="b7x4n9"
20 seconds
```

You double resources and get:

```text id="e1p6k3"
15 seconds
```

You may have spent more money without solving the underlying problem.

If query optimization can reduce it to:

```text id="j5m8q2"
500 ms
```

that's a much stronger solution.

---

# 59. Expert performance framework

Use:

```text id="r3k7v1"
OBSERVE
   ↓
MEASURE
   ↓
LOCALIZE
   ↓
EXPLAIN
   ↓
CHANGE
   ↓
VALIDATE
   ↓
BASELINE AGAIN
```

Never skip **validate**.

---

# 60. Final architecture

```text id="x6n2p8"
                       APPLICATION
                            |
                    Connection Pool
                            |
                         Network
                            |
                         SQL MI
                            |
          +-----------------+-----------------+
          |                 |                 |
         CPU             Memory              I/O
          |                 |                 |
          +-----------------+-----------------+
                            |
                       SQL Workload
                            |
             +--------------+--------------+
             |              |              |
           Queries        Locks         TempDB
             |              |              |
             +--------------+--------------+
                            |
                       Query Store
                            |
                    Performance Analysis
                            |
             +--------------+--------------+
             |                             |
          Baseline                    Optimization
             |                             |
             +-------------+---------------+
                           |
                       Validation
                           |
                     SLO / Capacity
```

---

## Lesson 124 takeaway

The expert mindset is:

> **Don't ask "How do I make SQL MI faster?" Ask "Where is the latency coming from, what resource or workload is responsible, what evidence proves the cause, and how do I verify that my change actually improved the system?"**

The core workflow to remember:

```text
Symptom
  ↓
Measure
  ↓
Find bottleneck
  ↓
Find root cause
  ↓
Change
  ↓
Measure again
  ↓
Prove improvement
```

**Next: Lesson 125 — SQL Server Wait Statistics & Bottleneck Analysis — Expert Level.**
