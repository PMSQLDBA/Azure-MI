# Lesson 139 — Statistics Deep Dive: Auto-Update, Sampling, FULLSCAN, Incremental & Filtered Statistics

Today we go from **understanding cardinality estimation** to understanding the machinery that feeds it: **SQL Server statistics**.

The key idea:

> **Statistics are not just maintenance objects. They are optimizer information.**

---

## 1. What statistics actually do

Suppose a table has:

```text
100,000,000 rows
```

SQL Server doesn't want to inspect every row every time it compiles a query.

Instead, statistics provide a compact representation of data distribution.

Conceptually:

```text
100M rows
   ↓
Statistics
   ↓
"Here's what the data roughly looks like"
   ↓
Optimizer
   ↓
Estimated cardinality
```

---

# 2. Statistics influence plan selection

Statistics can influence:

```text
Index Seek vs Scan
Nested Loops vs Hash Match
Join order
Memory grants
Parallelism
Sort strategy
Aggregation strategy
```

So statistics can indirectly determine almost the entire execution plan.

---

# 3. Where statistics come from

SQL Server can have statistics associated with:

### Indexes

An index can have statistics describing its key distribution.

### Auto-created statistics

SQL Server can create statistics for columns when the optimizer determines they are useful.

### User-created statistics

You can explicitly create statistics for specific columns/column combinations.

Conceptually:

```text
Index statistics
Auto statistics
User-created statistics
```

---

# 4. Statistics object vs index

Do not confuse:

```text
INDEX
```

with:

```text
STATISTICS
```

An index is primarily an access structure.

Statistics are primarily information used for estimation.

For example:

```text
Index:
"Here's an efficient way to find these rows."

Statistics:
"Here's roughly how many rows you'll find."
```

Both are important, but they solve different problems.

---

# 5. Statistics have a lifecycle

Think of statistics as:

```text
Created
  ↓
Used
  ↓
Data changes
  ↓
Potentially becomes less representative
  ↓
Updated
  ↓
Used again
```

The important part is that statistics must remain useful as the data distribution evolves.

---

# 6. Auto-create statistics

When appropriate, SQL Server can automatically create statistics.

For example, suppose a query frequently filters on:

```sql id="0r0v6k"
WHERE Region = 'West'
```

SQL Server may create statistics for:

```text id="5y8s0p"
Region
```

This reduces the amount of manual statistics management required.

---

# 7. Auto-create doesn't mean perfect

Automatic creation is helpful, but SQL Server cannot predict every possible statistical relationship.

For example:

```text id="i1s2z8"
State
City
CustomerType
Region
Date
```

may have complicated relationships.

A single-column statistic cannot perfectly describe every combination.

---

# 8. Auto-update statistics

SQL Server can automatically update statistics when it determines that enough underlying data has changed.

Conceptually:

```text id="afl5ka"
Data changes
     ↓
Change tracking / thresholds
     ↓
Statistics become eligible
     ↓
Statistics update
```

The exact behavior depends on SQL Server version, compatibility level, table size, and configuration.

---

# 9. "Old" doesn't necessarily mean "bad"

This is an important rule.

Suppose statistics were updated:

```text id="0f3p9q"
30 days ago
```

That doesn't automatically mean they are bad.

If the table is stable:

```text id="9v1q6m"
Data distribution ≈ unchanged
```

the statistics may still be perfectly useful.

---

# 10. Conversely, new doesn't guarantee accurate

Imagine:

```text id="9k0v5n"
Statistics updated yesterday
```

but today:

```text id="5j3m8w"
50 million rows were inserted
```

The statistics may already be less representative of the current data.

The relevant question is:

> **How different is the current data distribution from the information captured by the statistics?**

---

# 11. Statistics modification counters

When investigating statistics, one useful piece of information is how much the underlying data has changed since the statistics were created or updated.

You can inspect metadata such as:

```text id="7svf7p"
last_updated
modification_counter
rows
rows_sampled
```

This helps determine whether a statistics problem is plausible.

---

# 12. Inspect statistics metadata

A common tool is:

```sql id="5r2xwq"
sys.dm_db_stats_properties
```

For example, you can inspect statistics associated with a table/index and see information about:

```text id="3g5q0r"
Last update
Rows
Rows sampled
Modification count
```

This is much better than guessing.

---

# 13. Sampling

When statistics are updated, SQL Server doesn't necessarily scan every row.

It can use a sample.

Conceptually:

```text id="f5v3kq"
1 billion rows
      ↓
Sample subset
      ↓
Build distribution estimate
      ↓
Statistics
```

Sampling reduces the cost of statistics maintenance.

---

# 14. Why sampling exists

Imagine a huge table:

```text id="4p8c7x"
10 billion rows
```

Scanning the entire table solely to refresh statistics can be expensive.

If a representative sample provides sufficiently good estimates:

```text id="3q1c0r"
Small maintenance cost
+
Good enough statistics
```

may be a better tradeoff.

---

# 15. But sampling can miss rare values

Suppose:

```text id="p8z5x1"
99.99% → common values
0.01%  → rare values
```

A sample might not fully capture the rare distribution.

That can matter if a query specifically searches for those uncommon values.

---

# 16. FULLSCAN

You can request statistics to be built from the entire relevant data set.

Conceptually:

```text id="m3b8zv"
FULLSCAN
   ↓
All rows examined
   ↓
More complete distribution information
```

This can improve statistical accuracy.

But:

> **FULLSCAN is not automatically the best solution.**

---

# 17. Why FULLSCAN can be expensive

On a very large table:

```text id="k72r2n"
10 billion rows
```

a full scan may cause significant:

```text id="t3k8xq"
I/O
CPU
Duration
Storage activity
```

If done too frequently, statistics maintenance itself can become a workload problem.

---

# 18. The tradeoff

Think:

```text id="4u2q5n"
More accurate statistics
        ↕
More maintenance cost
```

The goal is not:

> "Make every statistic FULLSCAN."

The goal is:

> **Use enough statistical accuracy to produce good plans without creating unnecessary maintenance overhead.**

---

# 19. Sampling can actually be excellent

For many distributions, a sufficiently representative sample can provide excellent estimates.

If the data is:

```text id="9q1j5x"
large
stable
reasonably distributed
```

sampling may be completely adequate.

---

# 20. When FULLSCAN becomes more interesting

FULLSCAN can be worth investigating when:

```text id="j8v1s3"
Estimates are consistently wrong
+
The column has difficult/skewed distribution
+
Sampling appears inadequate
+
The statistics are important to a critical workload
```

But always test before assuming.

---

# 21. Statistics update options

You may encounter commands such as:

```sql id="d7k4r2"
UPDATE STATISTICS dbo.Orders;
```

or:

```sql id="4f6q1a"
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;
```

The first allows the engine to choose an update strategy.

The second explicitly requests a full scan.

---

# 22. Don't FULLSCAN everything

This is a classic beginner mistake.

Imagine:

```text id="e7f0r3"
500 large tables
```

and you run:

```text id="3bq7jz"
FULLSCAN on everything
```

You could create a huge maintenance workload.

Better:

```text id="7h1k5a"
Identify problematic statistics
        ↓
Target those statistics
        ↓
Test improvement
```

---

# 23. Statistics update timing

Statistics updates can happen in contexts where SQL Server needs current enough information for compilation.

There are also configuration options affecting whether updates happen synchronously or asynchronously.

These settings have different tradeoffs.

---

# 24. AUTO_UPDATE_STATISTICS_ASYNC

With asynchronous statistics updates, a query may compile using existing statistics while an update happens in the background.

Conceptually:

```text id="g3v0yb"
Query arrives
   ↓
Statistics need update
   ↓
Use existing stats
   ↓
Background update begins
```

This can reduce compilation waits.

But the first compilation may still use the older statistics.

---

# 25. Synchronous statistics update

With synchronous behavior:

```text id="d0w7yr"
Query arrives
   ↓
Statistics need update
   ↓
Wait for statistics update
   ↓
Compile using updated stats
```

The query may pay the update cost immediately.

But it can compile with fresher information.

---

# 26. There is no universal winner

Async vs synchronous depends on workload.

Consider:

```text id="7f1bqv"
High compilation latency sensitivity
```

versus:

```text id="8g9x3m"
Highly sensitive to estimate accuracy
```

You need to evaluate the workload rather than applying a blanket rule.

---

# 27. Statistics and plan cache

Remember:

```text id="h0u8r6"
Statistics change
        ↓
Plan may become eligible for recompilation
```

The exact recompilation behavior depends on the scenario.

But statistics are closely tied to plan compilation.

---

# 28. Why this matters

Suppose:

```text id="j1d4vz"
Old statistics
      ↓
Plan A
```

Then statistics update:

```text id="e3k6m0"
New statistics
      ↓
Optimizer recompiles
      ↓
Plan B
```

Plan B might be:

```text id="8l2s7x"
Much better
```

or, in some circumstances:

```text id="w6n9ce"
Unexpectedly worse
```

This is one reason plan changes sometimes correlate with statistics updates.

---

# 29. Incremental statistics

For partitioned tables, **incremental statistics** can be particularly useful.

Imagine:

```text id="3l6f8w"
Orders
├── 2023
├── 2024
├── 2025
├── 2026
```

Without incremental statistics, maintaining statistics across the entire structure can be expensive.

---

# 30. Incremental statistics concept

With incremental statistics, SQL Server can maintain statistical information at partition level and combine it into a global picture.

Conceptually:

```text id="a7r9cx"
Partition 1 → statistics
Partition 2 → statistics
Partition 3 → statistics
Partition 4 → statistics
       ↓
Global statistical view
```

This can be valuable for large partitioned tables where only some partitions change.

---

# 31. Why partitioning helps here

Suppose:

```text id="3f1m8j"
2023 = static
2024 = static
2025 = static
2026 = constantly changing
```

You don't want every small change in 2026 to require expensive work across all historical data.

Incremental statistics can make maintenance more targeted.

---

# 32. Partitioning doesn't automatically solve estimation

Important distinction:

```text id="p8j7s1"
Partitioning
≠
Good statistics
```

Partitioning can improve manageability and certain query patterns.

Statistics still need to accurately describe the relevant data.

---

# 33. Filtered statistics

Filtered statistics describe a subset of rows.

Conceptually:

```text id="u4t8f0"
Table
████████████████████████

Filtered statistic
█████
```

For example:

```sql id="x6j4b0"
CREATE STATISTICS
    Stats_ActiveOrders
ON dbo.Orders(OrderDate)
WHERE Status = 'Active';
```

This can give the optimizer targeted information about that subset.

---

# 34. When filtered statistics can help

Suppose:

```text id="v2b7g6"
Active = 95%
Cancelled = 4.9%
Pending = 0.1%
```

A query targeting:

```text id="z5q0x1"
Pending
```

may benefit from more targeted statistical information.

---

# 35. Filtered index vs filtered statistics

Don't confuse:

```text id="j0h3c9"
Filtered Index
```

with:

```text id="5j1m4v"
Filtered Statistics
```

A filtered index provides:

```text id="f6v7n2"
An access structure for a subset
```

Filtered statistics provide:

```text id="9x3d2k"
Distribution information for a subset
```

They can sometimes be used together, but they solve different problems.

---

# 36. Multi-column statistics

Suppose a query frequently uses:

```sql id="x7m8b2"
WHERE CustomerType = 'Business'
  AND Region = 'West'
```

A multi-column statistic can provide information about the relationship between:

```text id="5q8f9v"
CustomerType
Region
```

rather than treating them as completely unrelated attributes.

---

# 37. Column order matters

Consider:

```text id="4n1z7w"
(CustomerType, Region)
```

versus:

```text id="0y8k5m"
(Region, CustomerType)
```

These are not interchangeable from the optimizer's perspective.

The leading column strongly affects the histogram and usefulness of the statistic for certain predicates.

---

# 38. Statistics on expressions

Modern SQL Server capabilities can support statistics/indexing strategies around computed expressions in appropriate designs.

The general principle remains:

> **If the query repeatedly filters on a transformed value, make that transformation visible to the optimizer in a way it can reason about.**

This is one reason persisted computed columns can sometimes be useful.

---

# 39. Example

Suppose you frequently query:

```sql id="a6t9q2"
WHERE YEAR(OrderDate) = 2026
```

A possible design strategy is to expose the relevant expression through a computed column and index/statistics strategy appropriate to the workload.

Then the optimizer can potentially reason about:

```text id="g7s1x3"
OrderYear = 2026
```

more directly.

This must be evaluated against the actual workload rather than applied automatically.

---

# 40. Statistics and skew

Highly skewed columns deserve special attention.

Example:

```text id="f5u9v1"
Status = 'Active' → 99%
Status = 'Archived' → 0.9%
Status = 'Pending' → 0.1%
```

A query for:

```sql id="8s6w0q"
WHERE Status = 'Pending'
```

has very different selectivity from:

```sql id="m0x5e4"
WHERE Status = 'Active'
```

The optimizer needs distribution information to distinguish them.

---

# 41. One statistic can represent very different values

This is why parameter-sensitive queries are interesting.

```text id="z8p2c7"
@Status = 'Active'
```

and:

```text id="m6r1x4"
@Status = 'Pending'
```

can require completely different plans.

Statistics may know the values have different frequencies.

The challenge becomes whether the compiled/reused plan is appropriate for the specific parameter values.

---

# 42. Statistics and filtered workloads

Suppose an application almost always queries:

```text id="b2q5k8"
Status = 'Active'
```

but the rare maintenance process queries:

```text id="v8x1d0"
Status = 'Archived'
```

Those two workloads may have radically different cardinality and optimal plans.

This is where understanding distribution is much more valuable than simply knowing "there is an index."

---

# 43. Statistics maintenance strategy

A mature database environment shouldn't use:

```text id="z1c9y5"
"Update all statistics every night."
```

as its only strategy.

Instead think:

```text id="d7r3p1"
Which tables change?
Which statistics change?
Which queries are sensitive?
Which statistics are inaccurate?
What is the maintenance cost?
```

---

# 44. Targeted maintenance

A better strategy may be:

```text id="7j2k9v"
Stable table
→ little intervention

Highly volatile table
→ more frequent attention

Critical skewed statistic
→ targeted update

Huge partitioned table
→ incremental strategy
```

This is workload-aware maintenance.

---

# 45. Rebuild vs reorganize vs statistics

Another common misconception:

> "Index rebuild automatically fixes all statistics problems."

Be careful.

Index operations can affect associated statistics, but:

```text id="r7f1z8"
Index maintenance
≠
Universal statistics maintenance
```

And manually created statistics are a separate concern.

Always know which statistics you're actually maintaining.

---

# 46. Rebuild and statistics

An index rebuild can update the statistics associated with that index using the full index data in applicable circumstances.

But this does not mean:

```text id="m8q0v6"
"Rebuild every index to refresh all statistics."
```

That can be extremely expensive and unnecessary.

---

# 47. Reorganize

Index reorganize is primarily an index-maintenance operation.

Don't use:

```text id="f9j4w0"
REORGANIZE
```

as a generic statistics-refresh command.

Again:

> **Index maintenance and statistics maintenance are related, but distinct tasks.**

---

# 48. Statistics maintenance cost

Statistics updates consume resources.

Potential costs include:

```text id="w4y7p2"
I/O
CPU
Locks / concurrency effects
Compilation activity
Storage
```

On very large tables, this can be significant.

---

# 49. Over-maintenance

Imagine a table that changes constantly.

If you repeatedly run:

```text id="6r2m9x"
FULLSCAN
```

after every small change, you may spend more resources maintaining statistics than the workload gains from the extra accuracy.

That's a bad tradeoff.

---

# 50. Under-maintenance

The opposite is:

```text id="c8v1q4"
Data changes massively
Statistics remain stale
Queries use poor estimates
Plans deteriorate
```

The goal is somewhere between these extremes.

---

# 51. Statistics quality is workload-dependent

The "best" statistics configuration depends on:

```text id="y3p5v8"
Table size
Data volatility
Distribution
Query patterns
Parameter sensitivity
Partitioning
Maintenance window
Concurrency
```

There is no universal setting that is optimal for every database.

---

# 52. Practical diagnostic query

When investigating a suspicious statistic, start by looking at metadata such as:

```sql id="g5p1n2"
SELECT
    s.name,
    sp.last_updated,
    sp.rows,
    sp.rows_sampled,
    sp.modification_counter
FROM sys.stats AS s
CROSS APPLY sys.dm_db_stats_properties(
    s.object_id,
    s.stats_id
) AS sp
WHERE s.object_id = OBJECT_ID('dbo.Orders');
```

This gives you a starting point for understanding the statistics state.

---

# 53. What to look for

Pay attention to:

```text id="x1c7v4"
last_updated
rows
rows_sampled
modification_counter
```

Ask:

```text id="0f8m2q"
Has the table changed substantially?
How much was sampled?
When were stats last updated?
Does the workload show estimate errors?
```

---

# 54. Don't diagnose from modification count alone

A large modification count is a clue—not proof.

For example:

```text id="3r7k9m"
1 million modifications
```

may be irrelevant if the modifications don't meaningfully change the distribution relevant to the query.

Again:

> **Distribution matters more than raw modification count.**

---

# 55. Statistics on volatile columns

Suppose:

```text id="n2m5z8"
CreatedDate
```

is constantly increasing.

New rows are continually added.

This is a classic situation where:

```text id="j9x1c6"
Statistics freshness
+
Ascending values
```

can become important.

---

# 56. Statistics and data loading

Bulk loads can radically change distributions.

For example:

```text id="b8v4q0"
Before load:
10M rows

After load:
500M rows
```

If the new rows have very different characteristics, estimates may change dramatically.

After major data-loading events, statistics should be considered part of the performance review.

---

# 57. ETL workloads

Data warehouses often have:

```text id="5y0x3m"
Large batch loads
```

followed by:

```text id="r8q6n2"
Analytical queries
```

The data distribution after the load can be significantly different from the previous state.

Statistics strategy therefore becomes especially important.

---

# 58. Incremental statistics for large partitions

Consider:

```text id="h4k8s1"
Partition 2024 → unchanged
Partition 2025 → unchanged
Partition 2026 → loaded today
```

Incremental statistics can allow maintenance to focus more efficiently on changing partitions in supported configurations.

This is especially valuable for large time-based tables.

---

# 59. Statistics and partition elimination

If a query filters on a partitioning key:

```text id="m1r5v7"
WHERE OrderDate >= '2026-01-01'
```

partition elimination can reduce the amount of data involved.

But statistics still matter for estimating the rows within the relevant partitions.

So:

```text id="7c4n0q"
Partition elimination
+
Good statistics
```

can be a powerful combination.

---

# 60. Filtered statistics vs partition statistics

They solve different problems.

### Filtered statistics

Target a logical subset based on a predicate.

### Incremental statistics

Help manage statistics across partitions.

Think:

```text id="q8m3w1"
Filtered → predicate-based subset
Incremental → partition-based maintenance
```

---

# 61. A statistics investigation example

Suppose:

```text id="2y5v9x"
Query:
WHERE CustomerId = @CustomerId
```

Actual plan:

```text id="n3b7q4"
Estimated = 20 rows
Actual = 8,000,000 rows
```

You inspect statistics.

You discover:

```text id="h7m1x9"
CustomerId is highly skewed
```

One customer owns most of the data.

Now you have a plausible explanation.

---

# 62. But statistics may already be current

Suppose:

```text id="a0c6j2"
last_updated = today
```

You might conclude:

> "Statistics are fine."

Not necessarily.

The problem could instead be:

```text id="x4r9k7"
Parameter sensitivity
Correlation
Predicate shape
Histogram limitations
```

Fresh statistics don't guarantee accurate estimates.

---

# 63. Testing a statistics hypothesis

A disciplined approach:

```text id="4m8z1q"
1. Capture current plan
2. Record estimated/actual rows
3. Inspect statistics
4. Update targeted statistics
5. Re-run
6. Compare plan
7. Compare runtime
```

If nothing improves:

```text id="c1v5x8"
The statistics hypothesis may have been wrong.
```

That's valuable information.

---

# 64. Statistics aren't magic

A common misconception is:

> "If the query has bad estimates, update statistics."

A better statement is:

> **"If statistics are inaccurate or insufficient to represent the relevant distribution, updating or redesigning statistics may improve estimates."**

That distinction matters.

---

# 65. The statistics decision tree

When you see a large estimate error:

```text id="w7z2c1"
Is the statistic stale?
       ↓
     Yes → update/test
       ↓
      No
       ↓
Is distribution highly skewed?
       ↓
     Yes → investigate histogram/filtering
       ↓
      No
       ↓
Are predicates correlated?
       ↓
     Yes → investigate multi-column information
       ↓
      No
       ↓
Is parameter sensitivity involved?
       ↓
     Yes → investigate plan behavior
       ↓
      No
       ↓
Investigate expressions, joins, conversions, etc.
```

---

# 66. Advanced concept: CE model

SQL Server has evolved its cardinality-estimation models over different engine/compatibility-level generations.

The important idea is:

```text id="s7x2p0"
Same query
+
Different CE behavior
=
Potentially different estimates
=
Potentially different plans
```

This is one reason compatibility-level changes can affect query performance even when query text doesn't change.

---

# 67. Legacy vs newer CE behavior

Historically, SQL Server had an older cardinality estimation model.

Newer versions introduced a newer model with different assumptions around things such as:

```text id="f5j8x0"
Predicate independence
Correlation
Join estimation
Data distribution
```

The newer model is not simply:

> "Always better."

It is a different estimation model that can improve many workloads while changing behavior for others.

---

# 68. Why upgrades can change plans

Suppose:

```text id="e8p4v2"
SQL Server version changes
```

or:

```text id="r1z6c9"
Database compatibility level changes
```

Then:

```text id="q6m3s1"
CE behavior
 ↓
Estimates
 ↓
Costs
 ↓
Plan choice
```

may change.

This is why performance testing should be part of major database upgrades.

---

# 69. Query Store and statistics investigations

When available, Query Store can help correlate:

```text id="x4k9q1"
Plan changes
+
Performance changes
+
Execution statistics
```

This makes it easier to determine whether a performance regression corresponds to a plan change.

---

# 70. A mature workflow

For a production regression:

```text id="0p4w8c"
Query Store
    ↓
Identify regression
    ↓
Compare plans
    ↓
Compare estimates
    ↓
Inspect statistics
    ↓
Identify cause
    ↓
Test targeted change
    ↓
Monitor
```

This is far more reliable than blindly rebuilding indexes.

---

# 71. Statistics maintenance hierarchy

Think in levels:

### Level 1 — Automatic

Let SQL Server manage statistics normally.

### Level 2 — Targeted

Manually update specific statistics/tables when evidence supports it.

### Level 3 — Advanced

Use strategies such as:

```text
Filtered statistics
Incremental statistics
Specific sampling
Workload-specific maintenance
```

Don't jump to Level 3 without a reason.

---

# 72. Common mistakes

### Mistake 1

> "Statistics are old, therefore they're bad."

False.

### Mistake 2

> "FULLSCAN is always better."

Not necessarily.

### Mistake 3

> "More statistics are always better."

False.

### Mistake 4

> "Index rebuild fixes all statistics."

False.

### Mistake 5

> "Estimate mismatch automatically means stale statistics."

False.

---

# 73. Interview question

### "What is the difference between statistics and indexes?"

Strong answer:

> "An index primarily provides an efficient access structure for retrieving rows, while statistics provide information about data distribution that the optimizer uses to estimate cardinality and cost. Both influence plan selection, but they serve different purposes."

---

# 74. Interview question

### "Why shouldn't you use FULLSCAN everywhere?"

Strong answer:

> "FULLSCAN can provide more complete distribution information, but it can be expensive on large tables. The goal is to maintain statistics accurately enough for good plans while keeping statistics-maintenance overhead reasonable."

---

# 75. Interview question

### "What is data skew?"

Strong answer:

> "Data skew occurs when values are distributed unevenly. A small number of values may represent a disproportionate number of rows, which can make average or independence-based estimates inaccurate and can contribute to parameter-sensitive plans."

---

# 76. Interview question

### "What are incremental statistics?"

Strong answer:

> "Incremental statistics are designed for partitioned tables and allow statistical information to be maintained more granularly by partition, which can reduce the cost of maintaining statistics when only some partitions change."

---

# 77. The expert mental model

Think of statistics as the optimizer's **belief system** about the database.

```text id="h7n4p1"
Reality
   ↓
Data distribution
   ↓
Statistics
   ↓
Optimizer's belief
   ↓
Estimated cardinality
   ↓
Plan
```

The optimizer can only make decisions based on the information available to it.

---

# 78. Lesson 139 takeaway

The key chain is:

```text id="2h5k9q"
DATA CHANGES
     ↓
STATISTICS REPRESENTATION
     ↓
CARDINALITY ESTIMATION
     ↓
COST ESTIMATION
     ↓
PLAN SELECTION
```

Your job as a performance engineer is not to keep every statistic "perfect."

Your job is to ensure that **important workloads have sufficiently accurate information to produce reliable plans at an acceptable maintenance cost.**

### The expert rule

> **Don't ask, "When should I update statistics?" Ask, "When has the optimizer's picture of the data become inaccurate enough to affect important plans?"**

**Next: Lesson 140 — Query Store Deep Dive: Capturing Queries, Plans, Regressions, Runtime Statistics, Forced Plans & Production Performance Troubleshooting.**
