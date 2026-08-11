# Lesson 137 — Execution Plans Deep Dive: Operators, Cost, Properties, Parallelism & Reading Plans Like a Performance Engineer

Today we move from **index structures** to the thing SQL Server actually builds to execute a query:

> **The execution plan.**

The goal is not merely to recognize operators. The expert skill is learning to look at a plan and ask:

> **Where is SQL Server doing unnecessary work, and why did the optimizer choose to do it?**

---

## 1. What is an execution plan?

An execution plan describes how SQL Server intends to execute a query.

Conceptually:

```text
SQL Query
   ↓
Parser / Algebrizer
   ↓
Optimizer
   ↓
Execution Plan
   ↓
Storage Engine
```

The plan contains operators such as:

```text
Scan
Seek
Join
Sort
Aggregate
Filter
Lookup
Spool
```

---

# 2. Estimated vs actual execution plan

There are two important concepts.

### Estimated plan

Shows what SQL Server **expects** to happen.

### Actual plan

Shows what SQL Server **actually did**, along with runtime information.

Conceptually:

```text
Estimated:
"SQL Server expects 100 rows"

Actual:
"SQL Server processed 1,000,000 rows"
```

That difference can be extremely important.

---

# 3. The most important comparison

When investigating performance, compare:

```text
Estimated Rows
        vs
Actual Rows
```

A major mismatch can indicate a cardinality-estimation problem.

For example:

```text
Estimated: 10
Actual:    500,000
```

That's not a small discrepancy.

It can completely change the optimal strategy.

---

# 4. Why estimates matter

Suppose SQL Server estimates:

```text
10 rows
```

It may choose:

```text
Nested Loops
+
Key Lookup
```

But if reality is:

```text
500,000 rows
```

that strategy may become extremely expensive.

The optimizer isn't necessarily "stupid."

It may simply have been given an inaccurate picture of the data.

---

# 5. Execution plan is a decision tree

A plan can be viewed conceptually as:

```text
             Query
               ↓
        ┌──────┴──────┐
     Access          Access
       ↓                ↓
     Join  ←────────── Join
       ↓
    Filter
       ↓
   Aggregate
       ↓
    Result
```

Each operator performs a piece of the work.

---

# 6. Reading direction

Graphical execution plans can be visually confusing.

For SQL Server plans, the data flow is generally understood from:

```text
Right → Left
```

The right side often contains the initial data-access operators.

The left side tends toward the final output.

Don't assume the operator positioned furthest left is where the work began.

---

# 7. Scan vs Seek

You already learned that:

```text
Seek ≠ always good
Scan ≠ always bad
```

The same principle applies when reading plans.

Ask:

```text
How many rows?
How many pages?
What percentage of the structure?
What is the estimated cost?
```

A scan of 20 pages can be excellent.

A seek followed by 5 million lookups can be terrible.

---

# 8. Table Scan

A table scan generally means SQL Server is reading the table's pages from a heap.

Conceptually:

```text
Heap
 ↓
Read pages
 ↓
Evaluate rows
```

This can be perfectly reasonable when most rows are needed.

---

# 9. Clustered Index Scan

If the table has a clustered index, scanning the table often appears as:

```text
Clustered Index Scan
```

The leaf level contains the table's rows.

So conceptually:

```text
Clustered Index Scan
       ↓
Table data
```

Again, this isn't automatically a problem.

---

# 10. Index Seek

A seek uses the index's ordering to navigate toward relevant keys.

Conceptually:

```text
Root
 ↓
Intermediate pages
 ↓
Target leaf range
```

This can drastically reduce the amount of data examined.

---

# 11. Seek predicates vs residual predicates

This is an advanced but important distinction.

A seek may use part of a predicate to navigate the index:

```text
Seek Predicate
```

Then apply additional filtering after reaching the relevant range:

```text
Residual Predicate
```

Therefore:

> **Seeing "Index Seek" doesn't necessarily mean SQL Server only touched the rows you ultimately wanted.**

---

# 12. Example

Index:

```text
(CustomerId, OrderDate)
```

Query:

```sql id="n4q6t1"
WHERE CustomerId = 10
  AND YEAR(OrderDate) = 2026;
```

SQL Server may navigate using:

```text
CustomerId = 10
```

but still evaluate the expression against many rows for that customer.

That can create substantial residual work.

---

# 13. Filter operator

A Filter operator evaluates a predicate against rows produced by another operator.

Conceptually:

```text
Input rows
   ↓
Filter
   ↓
Rows that pass
```

If a plan reads millions of rows and then filters almost all of them away, investigate whether the predicate could have been pushed into a better access path.

---

# 14. Predicate pushdown

Ideally, filtering happens as early as practical.

Compare:

```text
Read 10 million
 ↓
Filter to 100
```

with:

```text
Use index
 ↓
Read approximately 100
```

The second is usually much more efficient.

This is one reason good indexing and SARGability matter.

---

# 15. Compute Scalar

A Compute Scalar calculates expressions.

For example:

```sql id="t6y2p8"
SELECT Quantity * UnitPrice
```

may involve a Compute Scalar operator.

Don't automatically blame Compute Scalar when a query is slow.

The displayed operator cost may not tell the entire story of where CPU is actually being spent.

---

# 16. Sort

Sort orders rows.

For example:

```sql id="w7k3m5"
ORDER BY OrderDate
```

may produce:

```text
Scan
 ↓
Sort
 ↓
Result
```

Sorts can consume:

```text
CPU
Memory
TempDB
```

depending on the operation and data volume.

---

# 17. Avoiding unnecessary Sorts

A useful index may already provide the required order.

Instead of:

```text
Index Scan
 ↓
Sort
 ↓
Output
```

you may get:

```text
Ordered Index Access
 ↓
Output
```

This is one reason index key order matters.

---

# 18. Sort spills

If a sort doesn't have enough memory, SQL Server may spill work to TempDB.

Conceptually:

```text
Memory insufficient
       ↓
TempDB spill
       ↓
Additional I/O
```

Spills can significantly hurt performance.

---

# 19. Hash Match

Hash Match is used for several operations, especially joins and aggregation.

For a hash join:

```text
Build input
    ↓
Hash table
    ↓
Probe input
    ↓
Matches
```

It can be excellent for large, unsorted datasets.

---

# 20. Hash join memory

Hash operations often require memory to build internal structures.

If the memory grant is insufficient:

```text
Hash operation
      ↓
Spill
      ↓
TempDB
```

Again, memory grants matter.

---

# 21. Nested Loops

Nested Loops is conceptually:

```text
For each row from outer input:
    find matching rows in inner input
```

If outer input has:

```text
10 rows
```

this can be extremely efficient.

If it actually has:

```text
10 million rows
```

and the inner side requires expensive work each time, it can become disastrous.

---

# 22. Nested Loops is not "bad"

Never use:

> "Nested Loops = bad."

Instead ask:

```text
How many outer rows?
What does the inner operation cost?
Is the inner side indexed?
How many executions?
```

A loop executed 20 times can be excellent.

A loop executed 20 million times may be a problem.

---

# 23. Merge Join

Merge Join works best when both inputs are appropriately ordered.

Conceptually:

```text
Input A → sorted
              \
               Merge
              /
Input B → sorted
```

It can be very efficient for large joins when the required ordering already exists.

---

# 24. Three classic join algorithms

Remember:

```text
Nested Loops
Hash Match
Merge Join
```

The optimizer chooses among them based on estimated:

```text
Rows
Costs
Ordering
Indexes
Memory
Join predicates
```

---

# 25. Join algorithm selection

A simplified mental model:

### Small outer input

```text
Nested Loops
```

may be attractive.

### Large unsorted inputs

```text
Hash Match
```

may be attractive.

### Large already-ordered inputs

```text
Merge Join
```

may be attractive.

But actual optimizer decisions are more nuanced.

---

# 26. Key Lookup

A Key Lookup occurs when a nonclustered index finds rows but doesn't contain all required columns.

Conceptually:

```text
Nonclustered Index Seek
        ↓
Key Lookup
        ↓
Clustered row
```

If repeated many times, this can become expensive.

---

# 27. RID Lookup

On a heap, the equivalent concept is a RID Lookup.

```text
Nonclustered Index
       ↓
RID
       ↓
Heap row
```

So:

```text
Clustered table → Key Lookup
Heap            → RID Lookup
```

---

# 28. Spool operators

Spools store intermediate rows temporarily so they can be reused.

Conceptually:

```text
Input
 ↓
Spool
 ↓
Reuse
```

Spools can be useful.

Don't automatically assume:

> "Spool = bad."

The important question is **why SQL Server introduced it and how expensive it is.**

---

# 29. Table Spool

A Table Spool can materialize rows for later reuse.

Potential reasons include:

```text
Avoid repeated work
Support nested-loop behavior
Protect against certain data-access patterns
```

Sometimes a spool is a symptom of an underlying inefficiency.

Sometimes it is an intentional optimization.

---

# 30. Eager vs Lazy Spool

An expert-level distinction:

### Eager Spool

Consumes and stores its input proactively.

### Lazy Spool

Obtains/stores rows as they are requested.

The difference matters when analyzing when work occurs and how much data gets materialized.

---

# 31. Stream Aggregate

Stream Aggregate works on ordered input.

Conceptually:

```text
Sorted rows
   ↓
Stream Aggregate
   ↓
Groups
```

If the input is already appropriately ordered, this can be highly efficient.

---

# 32. Hash Aggregate

Hash Aggregate builds hash structures to group rows.

Conceptually:

```text
Rows
 ↓
Hash table
 ↓
Groups
```

It can work well without pre-sorted input but requires memory.

---

# 33. Stream vs Hash Aggregate

A simplified comparison:

| Operator         | Strength                     | Potential concern    |
| ---------------- | ---------------------------- | -------------------- |
| Stream Aggregate | Ordered input                | May require ordering |
| Hash Aggregate   | Doesn't require sorted input | Memory / spills      |

The optimizer chooses based on estimated costs.

---

# 34. Concatenation

For operations involving:

```sql id="j8m2x6"
UNION ALL
```

SQL Server may use a Concatenation operator.

Conceptually:

```text
Query A ─┐
         ├──→ Concatenation → Output
Query B ─┘
```

---

# 35. Parallelism

Large queries may use multiple worker threads.

Conceptually:

```text
          Query
            ↓
      Distribute Work
       ↙   ↓   ↘
    Worker Worker Worker
       ↘   ↓   ↙
       Gather
          ↓
       Result
```

Parallelism isn't automatically good or bad.

---

# 36. Why parallelism exists

Parallelism can reduce elapsed time by dividing work across multiple workers.

For example:

```text
Serial:
100 units → one worker

Parallel:
100 units → 4 workers
```

But real scaling is not perfectly linear.

---

# 37. Parallelism has overhead

Parallel execution introduces coordination costs:

```text
Worker scheduling
+
Data exchange
+
Synchronization
+
Memory
```

For a tiny query, parallelism may cost more than it saves.

---

# 38. Exchange operators

Parallel plans commonly contain exchange operators such as:

```text
Distribute Streams
Repartition Streams
Gather Streams
```

These operators move rows between worker threads.

They are essential to understanding parallel plans.

---

# 39. Repartition Streams

Suppose rows need to be distributed based on a join key.

SQL Server may hash the key and distribute rows:

```text
Rows
 ↓
Hash distribution
 ↙ ↓ ↘
Workers
```

This allows workers to process relevant subsets.

---

# 40. Gather Streams

At the end:

```text
Worker 1 ─┐
Worker 2 ─┤
Worker 3 ─┼→ Gather Streams → Result
Worker 4 ─┘
```

The results are combined.

This can sometimes become a bottleneck.

---

# 41. Parallelism skew

Suppose four workers receive:

```text
Worker 1 → 25%
Worker 2 → 25%
Worker 3 → 25%
Worker 4 → 25%
```

Great.

But imagine:

```text
Worker 1 → 5%
Worker 2 → 5%
Worker 3 → 5%
Worker 4 → 85%
```

Now one worker does most of the work.

That's **parallelism skew**.

---

# 42. Why skew happens

Often due to data distribution.

For example, suppose one customer represents:

```text
70% of all rows
```

and the distribution strategy doesn't spread that workload evenly.

Parallelism exists, but the work isn't balanced.

---

# 43. Degree of Parallelism

DOP means approximately how many worker threads participate in a parallel region.

Higher DOP isn't automatically faster.

You can encounter:

```text
Too little parallelism
```

or:

```text
Too much parallelism
```

depending on workload and server capacity.

---

# 44. Parallelism and concurrency

Suppose:

```text
100 concurrent queries
```

and every query aggressively uses many workers.

You can get worker pressure.

One query being faster in isolation doesn't necessarily mean the system is healthier under concurrency.

---

# 45. Cost threshold for parallelism

SQL Server uses configuration and optimizer costing to determine when parallel plans may be considered.

The important conceptual point:

> **The cost threshold is not "the number of milliseconds a query must run before parallelism."**

It is an optimizer cost concept.

---

# 46. Estimated operator cost

Execution plans often display estimated percentages.

For example:

```text
Index Scan   5%
Hash Match  80%
Sort        15%
```

Do not interpret these as:

> "Hash Match consumes exactly 80% of CPU."

They are optimizer cost estimates within that plan.

---

# 47. Estimated cost is not runtime

A plan can estimate:

```text
Operator A = 90%
```

while runtime reality differs significantly.

Why?

Because:

```text
Estimates
≠
Actual resource consumption
```

especially when cardinality estimates are wrong.

---

# 48. The plan tooltip is valuable

When examining an operator, inspect properties such as:

```text
Estimated Rows
Actual Rows
Estimated Number of Executions
Actual Number of Executions
Estimated Data Size
Seek Predicates
Predicate
Ordered
Parallelism details
Warnings
```

The properties often reveal more than the operator name itself.

---

# 49. Actual number of executions

This is particularly useful with Nested Loops.

Suppose:

```text
Inner Seek
Actual executions = 2,000,000
```

Now you know why that seemingly cheap seek may actually be expensive.

Cost per execution × executions can become enormous.

---

# 50. Rows per execution

Consider:

```text
Actual executions = 1,000,000
Rows per execution = 1
```

versus:

```text
Actual executions = 10
Rows per execution = 1,000,000
```

These are fundamentally different workloads.

Always inspect both dimensions.

---

# 51. Cardinality estimation errors

Suppose the plan says:

```text
Estimated rows: 100
Actual rows:    10,000,000
```

Now ask:

```text
Why?
```

Potential causes include:

```text
Stale statistics
Data skew
Correlated predicates
Parameter sensitivity
Non-SARGable expressions
Complex predicates
Insufficient statistics
```

---

# 52. A bad estimate can cascade

Imagine:

```text
Actual = 5 million
Estimated = 10
```

The optimizer may choose:

```text
Nested Loops
```

Then:

```text
5 million × inner operation
```

causes huge work.

So the visible problem:

```text
Nested Loops is slow
```

may not be the root cause.

The root cause could be:

```text
Bad cardinality estimate
```

---

# 53. Memory grants

Some operators need memory.

Examples:

```text
Sort
Hash Join
Hash Aggregate
```

SQL Server may request a memory grant before execution.

Conceptually:

```text
Query
 ↓
Estimate memory requirement
 ↓
Request memory
 ↓
Execute
```

---

# 54. Underestimated memory grant

If SQL Server expects:

```text
10,000 rows
```

but gets:

```text
10 million
```

the memory grant may be insufficient.

That can cause:

```text
Sort spill
Hash spill
TempDB activity
```

---

# 55. Overestimated memory grant

The opposite is also dangerous.

If SQL Server expects:

```text
100 million rows
```

but actually gets:

```text
100 rows
```

it may request far more memory than necessary.

Under concurrency, many such queries can create memory pressure.

---

# 56. Memory grant is a concurrency issue

Imagine:

```text
Query A → 2 GB
Query B → 2 GB
Query C → 2 GB
Query D → 2 GB
```

Even if each query is individually reasonable, concurrent grants can put pressure on available memory.

So performance tuning must consider:

```text
Single-query efficiency
+
System-wide concurrency
```

---

# 57. Spill warnings

Execution plans can expose warnings indicating that operations spilled to TempDB.

A spill is often worth investigating.

But don't automatically assume:

> "Any spill means the query is broken."

The severity depends on:

```text
Spill size
Frequency
Duration
TempDB impact
Concurrency
```

---

# 58. TempDB appears everywhere

TempDB can be involved in:

```text
Sort spills
Hash spills
Spools
Temporary objects
Versioning
Internal worktables
```

So a query plan can reveal why TempDB is suddenly busy.

---

# 59. Implicit conversions

One of the most useful plan warnings to recognize is an implicit conversion.

Example:

```sql id="e3k9p2"
WHERE IntColumn = @varcharParameter
```

SQL Server may need to convert one side.

Depending on data types and precedence, this can interfere with efficient index access.

---

# 60. Why data types matter

Suppose:

```text id="r8m4x1"
Column = INT
Parameter = VARCHAR
```

Even if the query looks simple:

```sql id="q6p7c3"
WHERE CustomerId = @CustomerId
```

the mismatch can change how SQL Server evaluates the predicate.

Best practice:

> **Use compatible data types across joins, predicates, parameters, and application code.**

---

# 61. Missing-index suggestions

Execution plans may sometimes suggest a missing index.

Treat these as:

> **Hints for investigation, not automatic instructions.**

A missing-index suggestion doesn't know the entire database workload.

It may recommend several overlapping indexes.

---

# 62. Why missing-index suggestions can be misleading

Suppose SQL Server recommends:

```text
INDEX(A)
INDEX(A,B)
INDEX(A,C)
```

Creating all three might be unnecessary.

A carefully designed:

```text
INDEX(A,B)
INCLUDE(C)
```

or another composite design could potentially serve the workload more efficiently.

Always consolidate recommendations with existing indexes.

---

# 63. Actual vs estimated plan: expert workflow

When investigating:

```text
1. Capture actual plan
2. Find highest-work operators
3. Compare estimated vs actual rows
4. Check executions
5. Check warnings
6. Check predicates
7. Check joins
8. Check memory/spills
9. Check parallelism
10. Identify root cause
```

Don't simply chase the operator with the largest displayed percentage.

---

# 64. Read from the data sources outward

A powerful technique is to start at:

```text
Scans
Seeks
Lookups
```

and ask:

```text
How much data entered the plan?
```

Then follow the data through:

```text
Filter
Join
Sort
Aggregate
```

This helps you understand where unnecessary rows are introduced.

---

# 65. The "too many rows too early" principle

One of the strongest tuning ideas is:

> **Don't carry unnecessary rows farther into the plan than necessary.**

Bad pattern:

```text
10 million rows
 ↓
Join
 ↓
Sort
 ↓
Filter
 ↓
100 rows
```

Potentially better:

```text
10 million
 ↓
Filter
 ↓
100,000
 ↓
Join
 ↓
Sort
 ↓
100
```

The exact optimizer behavior varies, but the principle is extremely useful.

---

# 66. The "too many columns too early" principle

The same applies to row width.

Carrying:

```text
50 columns
```

through a large join is more expensive than carrying only:

```text
5 required columns
```

where possible.

Wider rows can increase:

```text
Memory
I/O
Sort cost
Hash memory
Spill risk
```

---

# 67. Join order

The optimizer determines a join strategy and ordering based on estimates and cost.

A bad cardinality estimate can lead to a poor join order.

For example:

```text
A JOIN B JOIN C
```

might perform very differently from:

```text
B JOIN C JOIN A
```

depending on intermediate result sizes.

---

# 68. Intermediate result size

This is critical.

Imagine:

```text
A JOIN B → 100 rows
```

then:

```text
100 JOIN C → 1,000 rows
```

That's very different from:

```text
A JOIN B → 50 million rows
```

then:

```text
50 million JOIN C → 1,000 rows
```

The second plan may perform enormous unnecessary work.

---

# 69. Predicate placement

For inner joins, predicates can often be logically rearranged by the optimizer.

But the practical goal remains:

```text
Reduce rows early
```

especially before expensive joins, sorts, and aggregates.

---

# 70. Plan warnings deserve attention

Common warning categories include:

```text
Implicit conversions
Spills
No join predicate
Cardinality issues
Memory-related warnings
```

A warning doesn't automatically mean "fix this."

It means:

> **Investigate this operator.**

---

# 71. No join predicate

A join without an appropriate join condition can result in a Cartesian product.

Conceptually:

```text
A rows × B rows
```

If:

```text
A = 100,000
B = 100,000
```

the intermediate result can become enormous.

---

# 72. Cartesian product

For example:

```sql id="c4w9n2"
SELECT *
FROM Customers c
CROSS JOIN Products p;
```

produces combinations of every customer with every product.

This can be intentional.

But an accidental missing join predicate can create catastrophic workload.

---

# 73. Operator cost hierarchy

Don't memorize a universal list like:

```text
Sort = bad
Hash = bad
Scan = bad
Lookup = bad
```

Instead think:

```text
Operator
+
Rows
+
Executions
+
Data size
+
Memory
+
I/O
+
Concurrency
```

The same operator can be cheap in one plan and disastrous in another.

---

# 74. Example

Plan A:

```text
Index Seek
Actual rows = 5
Lookup executions = 5
```

Plan B:

```text
Index Seek
Actual rows = 5,000,000
Lookup executions = 5,000,000
```

The operator names look identical.

The performance implications are completely different.

---

# 75. Plan regression

A query that was fast yesterday can become slow today because:

```text
Data changed
Statistics changed
Plan changed
Indexes changed
Parameters changed
Server resources changed
Concurrency changed
```

Therefore performance troubleshooting must compare:

```text
Old plan
vs
New plan
```

when possible.

---

# 76. Parameter sensitivity

Consider:

```sql id="q8m5r1"
WHERE CustomerId = @CustomerId;
```

Customer A may have:

```text
20 rows
```

Customer B may have:

```text
20 million rows
```

One plan may not be optimal for both.

This is a classic parameter-sensitivity scenario.

---

# 77. Why parameter sensitivity matters to plans

A plan optimized for:

```text
20 rows
```

may favor:

```text
Nested Loops + Lookup
```

while a plan optimized for:

```text
20 million rows
```

may favor:

```text
Scan + Hash
```

The same query text can therefore behave very differently depending on parameter values and plan reuse.

---

# 78. Plan cache

SQL Server can cache execution plans.

Conceptually:

```text
Query text
 ↓
Compiled plan
 ↓
Plan cache
 ↓
Reuse
```

Plan reuse can avoid repeated compilation.

But reuse also means an unsuitable cached plan can sometimes create performance problems.

---

# 79. Compilation vs execution

A query has two broad phases:

```text
Compilation
 ↓
Execution
```

Compilation determines the plan.

Execution performs the work.

A query can be slow because of:

```text
Compilation
```

or:

```text
Execution
```

or both.

---

# 80. Don't tune only the visible operator

Suppose you see:

```text
Hash Match = expensive
```

Ask:

```text
Why did it receive so many rows?
Why did optimizer choose Hash Match?
Was the memory grant sufficient?
Was cardinality correct?
Was the join order appropriate?
```

The expensive operator may be the **victim**, not the root cause.

---

# 81. Root-cause thinking

A strong performance engineer thinks:

```text
Symptom
 ↓
Operator
 ↓
Why this operator?
 ↓
Why these rows?
 ↓
Why this estimate?
 ↓
Why this access path?
 ↓
Underlying cause
```

This prevents superficial tuning.

---

# 82. Execution-plan tuning checklist

When opening an actual execution plan, inspect:

```text
□ Estimated vs actual rows
□ Actual executions
□ Scans/seeks
□ Lookups
□ Join algorithms
□ Sorts
□ Aggregates
□ Spools
□ Memory grants
□ Spill warnings
□ Implicit conversions
□ Parallelism
□ Predicate placement
□ Data volume
```

---

# 83. A realistic investigation

Suppose:

```text
Query duration = 18 seconds
```

You see:

```text
Nested Loops
  ↓
Index Seek
  ↓
Key Lookup
```

Don't immediately add INCLUDE columns.

First inspect:

```text
Estimated outer rows = 50
Actual outer rows    = 4,000,000
```

Now the likely story becomes:

```text
Bad estimate
 ↓
Nested Loops selected
 ↓
Millions of lookups
 ↓
18-second query
```

That is much more useful than simply saying:

> "The Key Lookup is expensive."

---

# 84. Possible root causes of the estimate error

Now investigate:

```text
Statistics
Data distribution
Predicate shape
Parameter sensitivity
Correlated columns
Implicit conversion
Filtered data patterns
```

The fix may have nothing to do with the Key Lookup itself.

---

# 85. The most important execution-plan skill

Don't ask:

> "Which operator is bad?"

Ask:

> **"Why did SQL Server think this was the cheapest way to produce the result?"**

That question leads directly into optimizer reasoning.

---

# 86. Lesson 137 takeaway

Your execution-plan mental model should now be:

```text
QUERY
 ↓
ESTIMATES
 ↓
ACCESS PATH
 ↓
ROWS PRODUCED
 ↓
JOIN STRATEGY
 ↓
SORT / AGGREGATE
 ↓
MEMORY / TEMPDB
 ↓
PARALLELISM
 ↓
FINAL RESULT
```

And when actual performance is poor:

```text
Estimated rows
       ↓
Actual rows
       ↓
Executions
       ↓
Rows flowing between operators
       ↓
Resource usage
       ↓
Root cause
```

### The expert rule

> **Never diagnose an execution plan from operator names alone.**

A `Seek`, `Scan`, `Nested Loops`, `Hash Match`, `Sort`, or `Spool` is neither inherently good nor bad.

Its value depends on:

**how many rows, how many executions, how much data, how much memory, how much I/O, and why the optimizer chose it.**

**Next: Lesson 138 — Cardinality Estimation Deep Dive: Histograms, Density, Correlation, Skew, Selectivity & Why SQL Server Gets Row Counts Wrong.**
