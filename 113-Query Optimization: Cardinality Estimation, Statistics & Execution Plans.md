# Lesson 146 — Query Optimization: Cardinality Estimation, Statistics & Execution Plans

Today we connect several topics you've already learned:

**statistics → cardinality estimates → execution plans → memory grants → query performance**

The central idea:

> **SQL Server can only choose a good plan if it has a reasonably accurate idea of how many rows each operation will produce.**

---

## 1. What is cardinality estimation?

Cardinality estimation is SQL Server's attempt to answer:

> **"How many rows will this operation return?"**

For example:

```sql
SELECT *
FROM Orders
WHERE CustomerId = 100;
```

SQL Server might estimate:

```text
Estimated rows = 25
```

But reality could be:

```text
Actual rows = 20
```

That's reasonably accurate.

---

## 2. A bad estimate

Suppose SQL Server estimates:

```text
Estimated = 25
```

but execution produces:

```text
Actual = 2,000,000
```

That's a huge mismatch.

The optimizer may have chosen a plan assuming it only needed to process a tiny number of rows.

---

## 3. Why estimates matter

Cardinality estimates influence:

```text id="k8x4m2"
Join algorithm
Index access method
Join order
Memory grant
Parallelism
Aggregation strategy
Sort strategy
Hash strategy
```

One bad estimate can therefore affect the entire execution plan.

---

# 4. Execution plans are estimates + decisions

Conceptually:

```text id="r5n9c3"
Statistics
    ↓
Cardinality estimate
    ↓
Cost calculation
    ↓
Plan alternatives
    ↓
Chosen execution plan
```

The optimizer doesn't simply ask:

> "What's fastest?"

It evaluates possible strategies using estimated costs.

---

# 5. Estimated vs actual rows

One of the most useful things you can inspect in an actual execution plan is:

```text id="p3v7m1"
Estimated Rows
vs
Actual Rows
```

Example:

```text id="x6q2k8"
Estimated: 100
Actual:    105
```

Excellent.

Another:

```text id="m9c4r7"
Estimated: 100
Actual:    10,000,000
```

Potentially serious.

---

# 6. Estimation errors compound

Suppose:

```text id="a8k3n5"
Table A
Estimated = 100
Actual    = 1000
```

Then SQL Server joins that result with another table.

The next operator may now receive a very different cardinality than expected.

Conceptually:

```text id="q1v6x9"
Estimate wrong
     ↓
Next estimate wrong
     ↓
Next estimate wrong
     ↓
Plan quality deteriorates
```

This is why an error early in a plan can have a huge downstream effect.

---

# 7. Statistics

SQL Server uses statistics to estimate data distribution.

Statistics provide information about:

```text id="w4m8p2"
Distinct values
Data distribution
Density/selectivity
Histogram information
```

They help the optimizer predict how many rows a predicate may return.

---

# 8. Histograms

For a column such as:

```text id="j7c2r5"
CustomerId
```

the histogram provides information about ranges of values and their distribution.

Conceptually:

```text id="n5x9k1"
Value range      Frequency
---------        ---------
1–1000           High
1001–5000        Medium
5001–100000      Low
```

The actual statistics structure is more nuanced, but the mental model is useful.

---

# 9. Data skew

Suppose:

```text id="b6m3q8"
CustomerId = 1
→ 50 million rows

CustomerId = 2
→ 3 rows

CustomerId = 3
→ 5 rows
```

That's highly skewed data.

A predicate for CustomerId 1 behaves very differently from CustomerId 2.

This connects directly to our previous lesson on:

> **Parameter-sensitive queries.**

---

# 10. Statistics don't have to be perfect

The optimizer doesn't need exact counts for every value.

It needs estimates accurate enough to choose a good plan.

Think:

```text id="r8k1v4"
Good estimate
→ good plan likely

Bad estimate
→ bad plan more likely
```

---

# 11. Statistics updates

Statistics can become stale as data changes.

Imagine:

```text id="x2m7c9"
Statistics created
→ 1 million rows
```

Then:

```text id="p4n8q1"
50 million new rows inserted
```

The old statistics may no longer represent the current distribution accurately.

---

# 12. Auto-update statistics

SQL Server can automatically update statistics under appropriate conditions.

This is useful, but don't assume:

> "Auto-update statistics means statistics can never be a problem."

Large, rapidly changing tables can still present estimation challenges.

---

# 13. Fullscan statistics

Sometimes you may consider refreshing statistics with a more comprehensive sampling approach.

For example:

```sql
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;
```

This reads the entire relevant data to build the statistics.

But:

> **FULLSCAN is not something you should blindly schedule for every table.**

It can be expensive.

---

# 14. Sampling

Statistics are often built using sampled data.

Sampling is a performance trade-off:

```text id="u7m3k8"
Less data read
     ↓
Faster statistics creation
     ↓
Potentially less precise distribution information
```

For many workloads, sampling is perfectly adequate.

---

# 15. When statistics become suspicious

Investigate statistics when you see:

```text id="c5q2n9"
Large estimated/actual mismatch
Unexpected join strategy
Unexpected scans
Unexpected memory grants
Parameter-sensitive behavior
Sudden plan degradation
```

Statistics are one possible root cause.

---

# 16. Don't blame statistics automatically

Suppose:

```text id="h8x4m1"
Estimated = 100
Actual = 10,000
```

You shouldn't immediately conclude:

> "Update statistics."

First ask:

```text id="v2r7k5"
Why is the estimate wrong?
```

Possibilities include:

```text id="m6q1p8"
Stale statistics
Correlated predicates
Expressions
Data skew
Implicit conversion
Complex predicates
Join estimation
Parameter sensitivity
```

---

# 17. Correlated columns

This is an advanced but important problem.

Suppose:

```text id="a9n3x6"
State = 'LA'
City = 'New Orleans'
```

These columns aren't independent.

If the optimizer treats predicates too independently, estimates can become inaccurate.

For example:

```text id="s4k8m2"
WHERE State = 'LA'
AND City = 'New Orleans'
```

The actual relationship between the columns matters.

---

# 18. Independence assumptions

A simplified model might assume:

```text id="p7c2v9"
Selectivity(A AND B)
≈
Selectivity(A) × Selectivity(B)
```

But real-world columns may be correlated.

That can produce poor estimates.

---

# 19. Cardinality estimation versions

SQL Server has evolved its cardinality estimation models over time.

This matters because:

```text id="w5m1q7"
Older compatibility behavior
```

can produce different estimates from:

```text id="j3x8n4"
Newer cardinality estimation behavior
```

Changing database compatibility level can therefore change execution plans.

---

# 20. Compatibility level

A production change such as:

```text id="r6k2p9"
ALTER DATABASE ... SET COMPATIBILITY_LEVEL = ...
```

can affect optimizer behavior.

Therefore after a compatibility-level upgrade:

```text id="n8v4c1"
Don't only test application functionality.
```

Also test:

```text id="b2q7m5"
Query performance
Plan changes
CPU
I/O
Regressions
```

---

# 21. Query Store is extremely valuable here

Before a database upgrade, Query Store can provide historical performance information.

After the upgrade:

```text id="x9k3m6"
Compare:
Old plan
vs
New plan
```

and:

```text id="q4r8n1"
Old duration
vs
New duration
```

This helps identify regressions.

---

# 22. Plan regression

Suppose before an upgrade:

```text id="v1m6c8"
Duration = 100 ms
```

After:

```text id="d7p2x5"
Duration = 8 sec
```

If the execution plan changed substantially, you have a potential plan regression.

This is exactly where Query Store becomes useful.

---

# 23. Join estimation

Consider:

```sql
SELECT *
FROM Orders o
JOIN Customers c
  ON o.CustomerId = c.CustomerId;
```

SQL Server needs to estimate:

```text id="k8r3q6"
Rows from Orders
+
Rows from Customers
+
Join selectivity
```

An incorrect estimate can lead to the wrong join strategy.

---

# 24. Nested Loops

Nested Loops is often effective when the outer input is relatively small.

Conceptually:

```text id="p5x2m9"
Outer rows
   ↓
For each row
   ↓
Find matching inner rows
```

If the optimizer expects:

```text id="a7c4n1"
100 outer rows
```

but gets:

```text id="z3m8q5"
10 million outer rows
```

Nested Loops may become disastrous.

---

# 25. Hash Join

Hash joins are often useful for larger inputs.

Conceptually:

```text id="n6r1v8"
Build hash table
      ↓
Probe with other input
```

If SQL Server incorrectly believes the inputs are tiny, it may choose a strategy that doesn't fit reality.

---

# 26. Merge Join

Merge Join works especially well when inputs are appropriately ordered.

Conceptually:

```text id="c2x7m4"
Input A → sorted
Input B → sorted
       ↓
Merge matching keys
```

Again, estimates and ordering influence whether it is attractive.

---

# 27. Join choice isn't "which is best?"

The three major join algorithms are:

```text id="q8m3v6"
Nested Loops
Hash Join
Merge Join
```

None is universally best.

The optimizer chooses based on estimated workload characteristics.

---

# 28. Memory grant estimation

Cardinality estimation also affects memory grants.

Suppose SQL Server expects:

```text id="u5k1r9"
1,000 rows
```

but actual rows are:

```text id="h2p8m3"
10 million
```

The query may receive insufficient memory.

That can cause:

```text id="v7c4n6"
Sort spill
Hash spill
TempDB usage
```

This connects directly to Lesson 144.

---

# 29. Overestimation

The opposite can also happen.

Suppose SQL Server expects:

```text id="e9m2x5"
10 million rows
```

but actual is:

```text id="b6q1k8"
100 rows
```

The query may receive a large memory grant it doesn't really need.

Under concurrency, that can cause:

```text id="r4n7p2"
Memory pressure
Reduced concurrency
RESOURCE_SEMAPHORE waits
```

---

# 30. The estimate-to-resource connection

Remember this chain:

```text id="m8x3c7"
Cardinality estimate
       ↓
Plan choice
       ↓
Memory grant
       ↓
Execution behavior
       ↓
TempDB / CPU / I/O
```

This is one of the most important chains in SQL Server performance tuning.

---

# 31. Reading an execution plan

When you open an actual execution plan, don't stare at the entire diagram at once.

Start with:

```text id="p1v6n9"
1. Highest-cost operators
2. Estimated vs actual rows
3. Warnings
4. Scans/seeks
5. Join algorithms
6. Memory grants
7. Sorts/hashes/spills
```

---

# 32. Cost percentages

You may see:

```text id="k7m2q5"
Operator A = 70%
Operator B = 20%
Operator C = 10%
```

Don't interpret these as:

> "Operator A consumes 70% of server CPU."

They are optimizer cost estimates relative to the plan, not direct production resource measurements.

---

# 33. Actual execution time

If you need runtime truth, look at actual runtime metrics.

For example:

```text id="x4n8c1"
CPU
Elapsed time
Actual rows
Logical reads
Waits
```

The plan is a diagnostic artifact, not a replacement for runtime measurements.

---

# 34. Plan warnings

Pay attention to warnings such as:

```text id="r5m1v7"
Spill
Implicit conversion
Missing statistics
No join predicate
Excessive memory grant
```

A warning isn't automatically a problem, but it deserves investigation.

---

# 35. Missing index recommendations

SQL Server may show:

> Missing Index

Treat this as a **suggestion**, not an instruction.

It doesn't necessarily account for:

```text id="j2q8m4"
Existing overlapping indexes
Write cost
Storage
Other queries
Index maintenance
Workload trade-offs
```

Never blindly create every missing-index recommendation.

---

# 36. A better approach

Suppose SQL Server recommends:

```text id="n7c3x5"
CREATE INDEX ...
```

Ask:

```text id="v8m1q6"
Which query generated it?
How often does that query run?
How expensive is it?
Do I already have a similar index?
What will writes cost?
Will the index help other important queries?
```

Then test.

---

# 37. Query hints

Hints can influence optimizer decisions.

Examples include:

```text id="q4k9m2"
RECOMPILE
OPTIMIZE FOR
MAXDOP
MAX_GRANT_PERCENT
```

But hints should generally be treated as targeted interventions.

They can become technical debt if used without understanding the root problem.

---

# 38. MAXDOP

MAXDOP controls the maximum degree of parallelism available to a query.

But:

> **MAXDOP is not a magic "make query faster" switch.**

Too much parallelism can create:

```text id="w3p7x1"
CPU pressure
Worker contention
Parallelism overhead
```

Too little can make large queries unnecessarily slow.

---

# 39. Parallelism and estimates

Cardinality estimates can influence whether parallel execution is considered worthwhile.

If SQL Server thinks a query processes:

```text id="a6m2v9"
50 rows
```

parallelism may not make sense.

If reality is:

```text id="c8q4r1"
50 million rows
```

the choice becomes very different.

Again:

```text id="k5n7x3"
Estimate → plan choice
```

---

# 40. Parameter-sensitive estimates

Connect today's lesson with Lesson 143.

Suppose:

```text id="u2r8m4"
Parameter A
Estimated = 10
Actual = 10

Parameter B
Estimated = 10
Actual = 20 million
```

The second execution can experience:

```text id="x7c1p5"
Bad join strategy
Bad memory grant
Spills
Huge I/O
```

This is why cardinality estimation is central to parameter-sensitive performance.

---

# 41. A practical diagnostic example

Query:

```sql
SELECT *
FROM Orders
WHERE CustomerId = @CustomerId
  AND Status = 'Open';
```

You observe:

```text id="m3q8v1"
Estimated rows = 50
Actual rows = 2,000,000
```

Don't immediately create an index.

Investigate:

```text id="j6n2x9"
Statistics
Data distribution
CustomerId skew
Status distribution
Parameter value
Existing indexes
```

---

# 42. Maybe the index is wrong

Suppose current index is:

```text id="p4k7c2"
(Status, CustomerId)
```

But the query frequently searches by:

```text id="v9m1x6"
CustomerId
```

A different index order might be more appropriate.

---

# 43. Maybe the statistics are wrong

Suppose the index is good, but statistics don't represent the current distribution.

Then:

```text id="r8q3n5"
Estimate
   ↓
Wrong
   ↓
Plan
   ↓
Wrong
```

Refreshing or improving statistics may help.

---

# 44. Maybe the predicate is correlated

Suppose:

```text id="x1m7k4"
CustomerId = 100
Status = 'Open'
```

and Customer 100 has an unusually high proportion of open orders.

Independent selectivity assumptions may misestimate the combined predicate.

Now the issue may require deeper investigation than simply updating statistics.

---

# 45. Maybe the query is fundamentally broad

If the query legitimately returns:

```text id="c5v9n2"
2 million rows
```

then an index seek isn't necessarily going to make the operation cheap.

Ask whether:

```text id="u6k1p8"
2 million rows
```

actually need to be returned.

Sometimes the biggest optimization is:

> **Return less data.**

---

# 46. Result-set size matters

If the application asks for:

```sql
SELECT *
```

but only needs:

```text id="m2x8r4"
OrderId
OrderDate
TotalAmount
```

you may reduce:

```text id="p7c3n9"
I/O
Network traffic
Memory
CPU
```

and potentially enable better covering indexes.

---

# 47. Avoid SELECT *

This is both a query-design and indexing lesson.

Instead of:

```sql
SELECT *
```

prefer:

```sql
SELECT OrderId, OrderDate, TotalAmount
```

when those are the only required columns.

---

# 48. Statistics troubleshooting checklist

When estimates look wrong:

```text id="n8v4m2"
□ Compare estimated vs actual rows
□ Check statistics
□ Check last statistics update
□ Check data changes
□ Check skew
□ Check correlated predicates
□ Check implicit conversions
□ Check parameter sensitivity
□ Check expressions
□ Check compatibility level
```

---

# 49. Execution-plan troubleshooting checklist

When a query is slow:

```text id="q6m1x8"
□ What is the biggest actual cost?
□ Where are the biggest row-estimation errors?
□ Are there spills?
□ Are there excessive lookups?
□ Is there blocking?
□ Are memory grants appropriate?
□ Is the join algorithm reasonable?
□ Is the query returning too much data?
```

---

# 50. Query optimization hierarchy

A useful mental hierarchy:

```text id="v3k7p5"
1. Correct the query
2. Correct data access/indexing
3. Correct statistics/estimates
4. Correct configuration
5. Apply targeted hints if justified
```

The exact order varies, but avoid using hints to hide fundamental query problems.

---

# 51. Interview question

### "What is cardinality estimation?"

Strong answer:

> "Cardinality estimation is the optimizer's prediction of how many rows an operator or predicate will produce. Those estimates influence access methods, join algorithms, memory grants, parallelism, and many other plan decisions."

---

# 52. Interview question

### "What does an estimated-vs-actual row mismatch tell you?"

Strong answer:

> "It indicates that the optimizer's cardinality estimate differed from runtime reality. A significant mismatch can lead to poor join choices, inappropriate memory grants, spills, and inefficient access methods. I'd investigate statistics, data distribution, predicates, parameter sensitivity, and other causes."

---

# 53. Interview question

### "Are stale statistics always the cause of bad estimates?"

Strong answer:

> "No. Stale statistics are one possible cause. Other causes include data skew, correlated predicates, expressions, implicit conversions, parameter sensitivity, and limitations in the optimizer's estimation model."

---

# 54. Expert mental model

When you see a bad plan, work backward:

```text id="x8r2m6"
Bad runtime
    ↑
Bad execution strategy
    ↑
Bad optimizer decision
    ↑
Bad cardinality estimate?
    ↑
Bad statistics / predicate / distribution?
```

This is much more powerful than randomly changing indexes or server settings.

---

# 55. Lesson 146 takeaway

The most important chain is:

```text id="m4q9v1"
DATA
 ↓
STATISTICS
 ↓
CARDINALITY ESTIMATE
 ↓
OPTIMIZER DECISION
 ↓
EXECUTION PLAN
 ↓
MEMORY / CPU / I/O
 ↓
RUNTIME PERFORMANCE
```

When a query is slow, ask:

> **Where in this chain did reality diverge from the optimizer's assumptions?**

That's the mindset of a strong SQL Server performance engineer.

---
**Next:** **Lesson 147**
**Final:** **Lesson 150**
