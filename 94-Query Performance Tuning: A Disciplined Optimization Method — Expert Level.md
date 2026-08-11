# Lesson 127 — Query Performance Tuning: A Disciplined Optimization Method — Expert Level

Today we move from **diagnosis** to **controlled optimization**.

The biggest difference between a beginner and an expert isn't knowing more SQL syntax.

It's knowing **how to change a query without guessing**.

---

## 1. The optimization principle

Never start with:

> "What index should I create?"

Start with:

> **"What is the workload doing, and what makes it expensive?"**

Use:

```text
Measure
  ↓
Understand
  ↓
Hypothesize
  ↓
Change
  ↓
Measure again
```

---

# 2. Define the performance objective

Before tuning, establish the target.

Example:

```text
Current P95 = 2.5 sec
Target P95  = <500 ms
```

Or:

```text
CPU/query = 200 ms
Target     = <75 ms
```

Without a target, you don't know whether optimization succeeded.

---

# 3. Capture the baseline

Record:

```text
Duration
CPU time
Logical reads
Physical reads
Execution count
Waits
Execution plan
Rows returned
```

For example:

```text
Before optimization

Duration      = 1,800 ms
CPU           = 1,200 ms
Logical reads = 450,000
```

Now you have something to compare against.

---

# 4. Understand the query

Before changing it, understand:

```text
What does it return?
Why is it needed?
How frequently is it executed?
What parameters does it receive?
How much data should it process?
```

Sometimes the best optimization is discovering that the application doesn't need the query at all.

---

# 5. Query optimization hierarchy

Use this order:

```text
1. Remove unnecessary work
2. Reduce rows
3. Reduce columns
4. Improve predicates
5. Improve joins
6. Improve indexing
7. Improve plan stability
8. Optimize resource usage
```

This is usually more effective than blindly adding hardware.

---

# 6. SELECT *

Consider:

```sql id="q3k8m1"
SELECT *
FROM Customers
WHERE CustomerId = @CustomerId;
```

If the application only needs:

```text id="v7p2r5"
CustomerId
Name
Status
```

use:

```sql id="m4x9c2"
SELECT CustomerId, Name, Status
FROM Customers
WHERE CustomerId = @CustomerId;
```

You're reducing unnecessary data movement.

---

# 7. Why fewer columns matter

Returning unnecessary columns increases potentially:

```text id="n6r1q8"
I/O
Memory
Network traffic
Serialization
Client processing
```

A query doesn't become efficient merely because it returns the correct rows.

It should return **only what is required**.

---

# 8. Filter early

Consider:

```sql id="p8c2v5"
SELECT ...
FROM LargeTable
WHERE Status = 'Active';
```

The goal is to reduce the working dataset as early as practical.

Think:

```text id="x1m7k4"
10 million rows
      ↓
Filter
      ↓
100,000 rows
      ↓
Join / aggregate
```

rather than carrying millions of unnecessary rows through later operations.

---

# 9. SARGability

One of the most important query-tuning concepts is **SARGability**.

A predicate is generally more index-friendly when SQL Server can efficiently use an index to locate qualifying rows.

Example:

```sql id="r5n9x3"
WHERE OrderDate >= @StartDate
  AND OrderDate < @EndDate
```

This can be much more index-friendly than applying a function to the indexed column.

---

# 10. Function on a column

Potentially problematic pattern:

```sql id="c7v2m8"
WHERE YEAR(OrderDate) = 2026
```

You are transforming the column before comparison.

A range predicate is often preferable:

```sql id="h3q8p1"
WHERE OrderDate >= '20260101'
  AND OrderDate <  '20270101'
```

The exact design depends on the workload and data types.

---

# 11. Implicit conversions

Another common performance problem:

```text id="j6m1r9"
Column datatype
      ≠
Parameter datatype
```

SQL Server may need to perform conversions.

Potential consequences include:

```text id="w4x8c2"
Poor index usage
Extra CPU
Unexpected plans
```

Ensure application parameters use appropriate data types.

---

# 12. Leading wildcard searches

Consider:

```sql id="s9p3k7"
WHERE Name LIKE '%smith%'
```

A traditional B-tree index may not efficiently support searching for an arbitrary substring.

The problem isn't simply:

> "The index is missing."

The access pattern itself may not be compatible with a normal index strategy.

---

# 13. Trailing wildcard

Compare:

```sql id="d2v6m8"
WHERE Name LIKE 'Smith%'
```

with:

```sql id="q7n4x1"
WHERE Name LIKE '%Smith%'
```

The first pattern may be much more index-friendly.

This is an example of **query shape affecting index usability**.

---

# 14. Joins

Suppose:

```sql id="k8m2p5"
SELECT ...
FROM Orders o
JOIN Customers c
  ON o.CustomerId = c.CustomerId;
```

Investigate:

```text id="v1r7x9"
Join columns
Filtering columns
Cardinality
Indexes
Data distribution
```

Don't assume that adding an index to every join column automatically solves the problem.

---

# 15. Join cardinality

If one side produces:

```text id="n5c9q2"
100 rows
```

and the other:

```text id="f4m8x6"
10 million rows
```

the join strategy matters enormously.

Bad estimates can produce poor plans.

---

# 16. Nested loops

Nested loops can be excellent when:

```text id="a7p2m9"
Outer input = small
Inner lookup = efficient
```

But potentially expensive when:

```text id="r3x8k1"
Outer input = millions of rows
Inner side repeatedly accessed
```

Never label an operator "bad" without understanding its inputs.

---

# 17. Hash joins

Hash joins can be useful for larger inputs.

They may require substantial memory.

Potential problems include:

```text id="m6q1v8"
Large memory grants
Spills
CPU consumption
```

Again, context determines whether the operator is appropriate.

---

# 18. Sort operations

Sorts can become expensive when processing large datasets.

Investigate:

```text id="x9v4c3"
Why does the query need sorting?
Can an index provide the required order?
How many rows are being sorted?
Is the memory grant sufficient?
```

---

# 19. Memory spills

A query may request insufficient workspace memory for operations such as:

```text id="p2n7m5"
Sort
Hash
```

The operation may spill to TempDB.

Symptoms can include:

```text id="g8c1r6"
Extra I/O
Longer execution
TempDB activity
```

The solution may involve fixing cardinality estimates rather than simply increasing memory.

---

# 20. Covering indexes

Suppose a query needs:

```text id="u5k9x2"
WHERE CustomerId = ?
```

and returns:

```text id="d3m7q8"
Name
Status
Email
```

An index can potentially be designed so the required data can be obtained efficiently without excessive lookups.

This is the idea behind a **covering index**.

---

# 21. But don't over-cover

A huge covering index can create:

```text id="h6r2p4"
Large index size
Write overhead
Storage consumption
Maintenance cost
```

Design indexes based on important workloads.

---

# 22. Composite indexes

Suppose queries commonly filter by:

```text id="j1x8n5"
CustomerId
Status
OrderDate
```

Index column order matters.

An index isn't simply:

> "These three columns exist."

Its key ordering affects which predicates and access patterns it can support efficiently.

---

# 23. Equality vs range predicates

A common design consideration:

```text id="q4m7v9"
Equality
   +
Range
   +
Additional output columns
```

For example:

```text id="c8p2x6"
CustomerId = ?
Status = ?
OrderDate >= ?
```

The best index structure depends on the actual workload and selectivity.

---

# 24. Selectivity

Suppose:

```text id="n7k3m1"
IsActive
```

has:

```text
99% = true
1%  = false
```

An index on `IsActive` alone may not provide much filtering benefit for:

```text
WHERE IsActive = 1
```

Compare that with a highly selective customer identifier.

Index usefulness depends on data distribution.

---

# 25. Statistics matter

Indexes aren't enough.

The optimizer also needs useful information about data distribution.

Think:

```text id="r5x9p4"
Data
 ↓
Statistics
 ↓
Cardinality estimate
 ↓
Plan choice
```

Bad estimates can cause bad plans even when indexes exist.

---

# 26. Parameter sensitivity

Consider:

```text id="v3m8q1"
@CustomerId = 10
```

returns:

```text
5 rows
```

while:

```text
@CustomerId = 500000
```

returns:

```text
5 million rows
```

A plan ideal for one parameter may be poor for another.

---

# 27. Don't immediately rewrite the query

If the query is slow, first determine whether the issue is:

```text id="k6p1x7"
Query syntax
Indexing
Statistics
Plan selection
Parameter sensitivity
Data volume
Concurrency
Blocking
```

Sometimes the SQL text itself is perfectly reasonable.

---

# 28. Execution plan comparison

Before:

```text id="w2q9m5"
Index Seek
   ↓
Key Lookup × 10
```

After:

```text id="c7r4x8"
Covering Index Seek
```

Then compare:

```text id="m1n6p3"
CPU
Reads
Duration
Execution count
```

The plan looking "cleaner" isn't enough.

Measure the actual result.

---

# 29. Estimated plan vs actual plan

### Estimated plan

Shows what the optimizer expects.

### Actual execution plan

Shows what happened during execution, including actual row counts and runtime information where available.

For difficult performance problems, comparing:

```text
Estimated rows
vs
Actual rows
```

can reveal major estimation problems.

---

# 30. Cardinality mismatch

Example:

```text id="s4x8m2"
Estimated = 50
Actual    = 5,000,000
```

That's a huge mismatch.

Investigate:

```text id="p9v3k7"
Statistics
Predicates
Data distribution
Expressions
Parameter behavior
```

---

# 31. Query rewriting

Sometimes a query can be made more efficient by restructuring it.

For example:

```text id="g2m6q1"
Unnecessary subquery
        ↓
Simpler relational expression
```

But don't rewrite SQL simply because it looks complicated.

First prove that the structure causes measurable cost.

---

# 32. CTEs

A common misconception:

> "CTEs are automatically faster."

They're not inherently a performance optimization.

A CTE is primarily a query expression mechanism.

Evaluate the resulting execution plan and runtime behavior.

---

# 33. Temporary tables

Temporary tables can sometimes help by:

```text id="x5n8r3"
Breaking a complex operation
Materializing intermediate results
Improving reuse
Changing optimization boundaries
```

But they also introduce:

```text id="j7p2m9"
TempDB work
Additional writes
Statistics/compilation considerations
```

Use them deliberately.

---

# 34. EXISTS vs JOIN

Don't blindly follow rules like:

> "EXISTS is always faster."

or:

> "JOIN is always faster."

The optimizer may produce equivalent or different strategies depending on the query.

Compare the actual plans and workload.

---

# 35. DISTINCT as a performance smell

Sometimes developers add:

```sql id="r8c4v2"
SELECT DISTINCT ...
```

to remove duplicate rows created by an inefficient join.

That can introduce:

```text id="n2m7x9"
Sort
Hash aggregate
Extra memory
Extra CPU
```

Ask:

> **Why are duplicates being generated in the first place?**

---

# 36. ORDER BY

Sorting can be expensive.

If an application requires:

```sql id="p6k1x8"
ORDER BY OrderDate DESC
```

consider whether an appropriate index can help satisfy the ordering efficiently.

But verify the actual execution plan.

---

# 37. Pagination optimization

Large OFFSET values can require substantial work.

Example:

```sql id="v3q8m5"
OFFSET 500000 ROWS
FETCH NEXT 50 ROWS ONLY
```

For very large datasets, keyset/seek pagination can sometimes be more efficient.

Conceptually:

```text id="c9r2n7"
LastSeenId = 500000
        ↓
WHERE Id > LastSeenId
        ↓
Fetch next 50
```

---

# 38. Row-by-row processing

This pattern is often expensive:

```text id="m4x7p1"
WHILE
   process one row
END
```

or application code that repeatedly submits individual statements.

Prefer set-based operations when they match the business requirement.

---

# 39. Batch operations

Instead of:

```text id="q8n3v6"
1,000,000 individual operations
```

a well-designed batch operation may dramatically reduce:

```text id="f5r1m9"
Round trips
Compilation overhead
Transaction overhead
Logging overhead
```

But batch size must be chosen carefully.

---

# 40. Parameterization

Parameterized queries can help with:

```text id="w7k2c4"
Plan reuse
Security
Reduced compilation
```

But parameterization can also interact with parameter sensitivity.

Therefore:

> Parameterization is not automatically a performance solution.

---

# 41. Compilation overhead

If SQL Server repeatedly compiles similar statements instead of effectively reusing plans, CPU can be wasted.

Investigate:

```text id="z3m8q5"
Ad hoc workload
Query text variation
Parameterization
Plan cache behavior
```

---

# 42. Dynamic SQL

Dynamic SQL can be useful when designed correctly.

But careless dynamic SQL can cause:

```text id="h1p6x9"
Plan cache fragmentation
Compilation overhead
Security vulnerabilities
Hard-to-diagnose workload
```

Use parameterized dynamic SQL where appropriate.

---

# 43. ORM optimization

ORMs can generate inefficient SQL through:

```text id="k4v9r2"
Lazy loading
N+1 queries
Over-fetching
Unnecessary joins
Poor pagination
```

Query tuning sometimes requires changing ORM behavior rather than hand-editing SQL.

---

# 44. Query tuning isn't always database tuning

Example:

```text id="n6x2m8"
Database query = 20 ms
Application executes it 100,000 times
```

The real problem may be:

```text id="r5p7c1"
Application architecture
```

A database engineer should recognize this.

---

# 45. Caching

If the same expensive result is requested repeatedly and doesn't need real-time freshness, caching may reduce database workload.

Think:

```text id="u8m3q6"
Application
 ↓
Cache hit
 ↓
No DB query
```

But caching introduces:

```text id="s2k9v4"
Invalidation
Staleness
Memory use
Consistency considerations
```

---

# 46. Query tuning vs caching

Ask:

```text id="g7x1n5"
Can the query be made cheap?
Should the query exist at all?
Can the result be reused?
```

Sometimes the best database query is the one you don't execute.

---

# 47. Optimization experiment

Use controlled experiments.

Example:

```text id="p3r8m1"
Baseline:
Reads = 500,000
CPU = 900 ms
Duration = 1.2 sec
```

Change:

```text
Add targeted index
```

Then measure:

```text id="c6v2x9"
Reads = 8,000
CPU = 50 ms
Duration = 70 ms
```

That's strong evidence.

---

# 48. Don't trust estimated cost percentages

Execution plan percentages are useful for orientation, but they are optimizer estimates—not proof of actual runtime cost.

Always validate with runtime measurements.

---

# 49. Optimization regression

A change that improves one query can hurt another.

Example:

```text id="m8q4p2"
Query A
1 sec → 100 ms

Query B
100 ms → 500 ms
```

Why?

Perhaps a new index increased write cost or changed plan choices.

Always consider the broader workload.

---

# 50. Index impact on writes

Suppose:

```text id="x1n7c5"
INSERT
UPDATE
DELETE
```

must maintain several indexes.

More indexes can increase write cost.

Therefore:

> **Read optimization must be evaluated against write workload.**

---

# 51. The optimization tradeoff

Think:

```text id="v5m2r8"
READ PERFORMANCE
       ↕
WRITE PERFORMANCE
       ↕
STORAGE
       ↕
MAINTENANCE
```

There is rarely a completely free optimization.

---

# 52. Query tuning checklist

Before changing a query:

```text id="q9k3x6"
☐ What is the performance target?
☐ What is the baseline?
☐ How often does it run?
☐ How many rows does it process?
☐ What does the plan show?
☐ Are estimates accurate?
☐ Are waits significant?
☐ Are indexes appropriate?
☐ Are statistics healthy?
☐ Is blocking involved?
☐ Is the application pattern contributing?
```

---

# 53. After the change

Measure the same metrics:

```text id="r4p8m1"
Duration
CPU
Reads
Writes
Waits
Execution count
Plan
```

Then compare:

```text
Before
vs
After
```

---

# 54. Production safety

Never experiment recklessly in production.

Prefer:

```text id="k7n2v5"
Development
 ↓
Test
 ↓
Staging
 ↓
Controlled production rollout
 ↓
Monitor
```

For high-risk changes, have rollback plans.

---

# 55. Query tuning workflow

Memorize this:

```text id="b3x9q6"
1. Define target
       ↓
2. Capture baseline
       ↓
3. Understand workload
       ↓
4. Inspect plan
       ↓
5. Identify bottleneck
       ↓
6. Form hypothesis
       ↓
7. Make smallest useful change
       ↓
8. Measure
       ↓
9. Compare
       ↓
10. Validate broader workload
```

---

# 56. Expert scenario

### Problem

```text
Query duration = 4 sec
CPU = 3 sec
Reads = 1 million
```

Plan shows:

```text
Large table scan
```

You discover:

```text
WHERE CustomerId = @CustomerId
```

and `CustomerId` is highly selective.

Potential solution:

```text
Targeted index
```

Then validate.

---

# 57. Expert scenario #2

### Problem

```text
Query = 5 sec
CPU = low
```

Plan shows:

```text
Waiting on lock
```

Adding an index may not solve the immediate problem.

Investigate:

```text
Blocking
Transaction duration
Lock ownership
```

---

# 58. Expert scenario #3

### Problem

```text
Query = 2 sec
```

Plan looks reasonable.

But:

```text
Execution count = 2 million/hour
```

The issue may be workload frequency rather than individual query efficiency.

Potential solutions:

```text
Caching
Batching
Reducing unnecessary calls
Application redesign
```

---

# 59. Expert scenario #4

### Problem

Query performance varies dramatically by parameter.

Investigate:

```text id="j5v8m2"
Parameter sensitivity
Plan history
Data distribution
```

Potential solutions depend on the specific workload and SQL Server behavior.

Don't blindly rewrite the query.

---

# 60. The expert optimization mindset

A beginner asks:

> **"How do I make this query faster?"**

An expert asks:

> **"What work is unnecessary, what resource is expensive, what evidence proves it, and what is the smallest safe change that reduces that cost without damaging the rest of the workload?"**

---

# Lesson 127 takeaway

The core rule is:

> **Measure first. Change one thing. Measure again.**

Your optimization loop is:

```text
BASELINE
   ↓
PLAN
   ↓
BOTTLENECK
   ↓
HYPOTHESIS
   ↓
CHANGE
   ↓
MEASURE
   ↓
VALIDATE
```

And remember:

```text
Fast query
×
Huge execution count
=
Potentially huge workload cost
```

So optimize **the workload**, not merely the prettiest execution plan.

**Next: Lesson 128 — Index Architecture, Design & Advanced Index Tuning — Expert Level.**
