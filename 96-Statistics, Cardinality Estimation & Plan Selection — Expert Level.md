# Lesson 129 — Statistics, Cardinality Estimation & Plan Selection — Expert Level

Today we connect three concepts that control a huge portion of SQL Server query performance:

> **Statistics → Cardinality Estimates → Execution Plan**

If the optimizer badly estimates how many rows a query will produce, it can choose a plan that is fundamentally inappropriate for the workload.

---

## 1. The core chain

Memorize this:

```text
Data distribution
      ↓
Statistics
      ↓
Cardinality estimation
      ↓
Cost estimation
      ↓
Execution plan
      ↓
Query performance
```

A problem near the beginning can propagate all the way to execution.

---

# 2. What are statistics?

SQL Server statistics describe aspects of the distribution of values in columns or indexes.

Conceptually:

```text
Table
 ↓
Column values
 ↓
Statistics
 ↓
Optimizer learns about distribution
```

The optimizer uses this information to estimate how many rows predicates will return.

---

# 3. Why statistics exist

Imagine:

```text
Orders = 100 million rows
```

Query:

```sql
SELECT *
FROM dbo.Orders
WHERE CustomerId = 12345;
```

SQL Server needs to estimate:

> "How many rows will CustomerId = 12345 return?"

It can't inspect every row during optimization.

Statistics provide an approximation.

---

# 4. Cardinality estimation

Cardinality means, roughly:

> **How many rows are expected at a particular point in a query?**

Example:

```text
Estimated rows = 100
Actual rows    = 100
```

Excellent.

But:

```text
Estimated rows = 100
Actual rows    = 5,000,000
```

That's a serious mismatch.

---

# 5. Why estimates matter

The optimizer may choose differently depending on estimated row count.

For example:

```text
10 rows
 ↓
Nested Loops may be excellent
```

versus:

```text
10 million rows
 ↓
Hash Join or another strategy may be more appropriate
```

A bad estimate can therefore lead to a bad plan.

---

# 6. Statistics histogram

For suitable statistics, SQL Server maintains a histogram describing value distribution.

Conceptually:

```text
Value range        Approx rows
-----------        -----------
1–100              50,000
101–200            40,000
201–300            10,000
301–400             1,000
...
```

The optimizer uses this distribution to estimate predicates.

---

# 7. Data distribution matters

Consider:

```text
CustomerId
```

with:

```text
Customer A → 10 rows
Customer B → 10 rows
Customer C → 10 rows
```

Very uniform.

Now imagine:

```text
Customer X → 50 million rows
Everyone else → tiny counts
```

Highly skewed.

A single simplistic assumption may not accurately represent both cases.

---

# 8. Data skew

Skew means values aren't distributed evenly.

Example:

```text
Status
------
Active   → 95%
Closed   → 4%
Pending  → 1%
```

A query for:

```sql
WHERE Status = 'Active'
```

has very different selectivity from:

```sql
WHERE Status = 'Pending'
```

Statistics help the optimizer understand that difference.

---

# 9. The danger of stale statistics

Suppose statistics were created when:

```text
Rows = 1 million
```

Then the table grows to:

```text
Rows = 500 million
```

The distribution may have changed substantially.

If statistics don't adequately reflect current data, estimates may deteriorate.

---

# 10. Statistics updates

SQL Server can update statistics automatically under appropriate conditions.

You can also explicitly update them when justified.

Conceptually:

```text
Data changes
   ↓
Statistics become less representative
   ↓
Statistics update
   ↓
Optimizer gets newer information
```

But statistics updates themselves can have operational implications.

---

# 11. Don't blindly update statistics constantly

A common beginner strategy is:

> "Statistics are important, so update all statistics every hour."

That's not automatically good.

Frequent large statistics updates can cause:

```text
I/O
CPU
Compilation/recompilation effects
Maintenance overhead
```

Use evidence and workload requirements.

---

# 12. Statistics sampling

Statistics may be built or refreshed using a sample of the data rather than scanning every row.

Sampling can be much cheaper.

But for highly skewed data, the quality of the sample can matter.

The goal is:

```text
Useful estimates
+
Reasonable maintenance cost
```

---

# 13. FULLSCAN

A full scan provides more complete information for statistics creation/update.

Conceptually:

```text
Entire table/index
      ↓
Statistics
```

This can be useful in selected cases, but it can also be expensive on large tables.

Don't make FULLSCAN your universal solution.

---

# 14. Auto statistics

SQL Server can automatically create certain statistics for columns when useful.

This can help the optimizer discover data distribution information without manually creating every statistic.

But auto-created statistics are still part of a broader statistics-management strategy.

---

# 15. Column statistics vs index statistics

Statistics can be associated with:

```text
Index keys
```

and can also be created independently on columns.

The optimizer may use multiple statistics to estimate a query.

---

# 16. Multi-column statistics

This is an advanced concept.

Suppose:

```text
Country
City
```

are correlated.

For example:

```text
Country = USA
City = New Orleans
```

is not an independent random combination.

If the optimizer assumes independence incorrectly, cardinality estimates can be poor.

Multi-column statistics can sometimes provide more useful correlation information.

---

# 17. Correlated columns

Example:

```text
State = Louisiana
City = New Orleans
```

These values are related.

But if the optimizer treats predicates as independent without sufficient information, it may underestimate or overestimate the result.

This can affect join and memory decisions.

---

# 18. Cardinality estimation models

SQL Server has evolved its cardinality-estimation behavior over time.

Different compatibility levels and database configurations can affect which estimation model is used.

The important lesson is:

> **Changing the optimizer environment can change plan selection.**

---

# 19. Compatibility level matters

A database's compatibility level can influence:

```text
Query optimization behavior
Cardinality estimation
Available optimizer features
```

Therefore, after changing compatibility levels, performance testing is important.

---

# 20. Query optimizer isn't deterministic in the simplistic sense

People sometimes imagine:

```text
Same SQL
=
Always same plan
```

In reality, plan selection can depend on:

```text
Statistics
Parameter values
Schema
Indexes
Compatibility level
Optimizer behavior
SET options
Database state
```

---

# 21. Estimated rows vs actual rows

This is one of the most valuable execution-plan diagnostics.

Look for:

```text
Estimated: 10
Actual:    10
```

Great.

But:

```text
Estimated: 10
Actual:    1,000,000
```

is a major warning sign.

---

# 22. Where to look

Don't only compare the final query result.

Inspect important operators:

```text
Index Seek
Index Scan
Join
Sort
Aggregate
Filter
Key Lookup
```

Look for operators where estimated and actual row counts diverge dramatically.

---

# 23. Example

Suppose:

```text
Table Scan
Estimated rows = 1,000
Actual rows    = 20,000,000
```

The optimizer thought it was dealing with a small result but actually processed a huge dataset.

That can cascade into:

```text
Bad join choice
Insufficient memory
Spills
CPU increase
Long duration
```

---

# 24. Memory grants

Cardinality estimates influence memory grants.

Suppose SQL Server expects:

```text
100 rows
```

but gets:

```text
5 million rows
```

The query may receive too little memory for operations such as:

```text
Sort
Hash join
Hash aggregate
```

Potentially causing spills.

---

# 25. Excessive memory grants

The opposite can happen.

Suppose SQL Server estimates:

```text
50 million rows
```

but actual is:

```text
50 rows
```

The query might request far more memory than necessary.

That can reduce memory available to other concurrent queries.

So estimation errors can cause both:

```text
Under-grant
```

and:

```text
Over-grant
```

---

# 26. Join selection

Cardinality estimates strongly influence join strategy.

Example:

```text
Estimated small input
        ↓
Nested Loops
```

But actual:

```text
Millions of rows
```

The nested-loop strategy may become extremely expensive.

---

# 27. Parallelism decisions

Estimated cost also influences whether SQL Server chooses parallel execution.

A significant underestimation can contribute to a serial plan that performs poorly for a large workload.

An overestimation can contribute to unnecessary parallelism.

Again:

> Estimates influence the plan; they don't directly guarantee a particular outcome.

---

# 28. Predicate selectivity

The optimizer needs to estimate how selective a predicate is.

Example:

```sql
WHERE CustomerId = 100
```

might return:

```text
5 rows
```

while:

```sql
WHERE CustomerId > 0
```

might return:

```text
100 million rows
```

The difference is fundamental to plan selection.

---

# 29. Expressions can complicate estimation

Consider:

```sql
WHERE YEAR(OrderDate) = 2026
```

The optimizer may have less direct statistical information about the transformed expression than about the underlying column.

A range predicate:

```sql
WHERE OrderDate >= '20260101'
  AND OrderDate <  '20270101'
```

can often make the filtering intent clearer and more index-friendly.

---

# 30. Functions and estimation

Functions can make both:

```text
Index access
```

and:

```text
Cardinality estimation
```

more difficult.

This is another reason to avoid unnecessary transformations of indexed columns.

---

# 31. Parameter sensitivity

Consider:

```sql
WHERE CustomerId = @CustomerId;
```

Suppose:

```text
Customer A → 5 rows
Customer B → 20 million rows
```

The optimizer may compile a plan based on one parameter value that isn't ideal for another.

This is a major advanced tuning topic.

---

# 32. Parameter sniffing

During compilation, SQL Server may use parameter values available at compilation to help optimize the query.

This is often beneficial.

But when data distribution is highly uneven, the compiled plan may be poorly suited to later executions with very different parameter values.

---

# 33. Parameter sensitivity isn't always a problem

This distinction matters.

Parameter sniffing is not inherently bad.

It can produce excellent plans when the workload is reasonably uniform.

The problem occurs when:

```text
Parameter distribution
+
Plan reuse
+
Data skew
```

combine to produce poor plan choices.

---

# 34. Diagnosing parameter sensitivity

Look for:

```text
Same query
+
Large runtime variation
+
Different parameter values
+
Plan remains reused
```

Query Store can be particularly useful for spotting these patterns historically.

---

# 35. Potential strategies

Depending on the workload, SQL Server offers different mechanisms and query-design approaches to handle parameter sensitivity.

Possible approaches include:

```text
Query hints
Recompilation strategies
Dynamic SQL
Different query shapes
Plan management
Engine features designed for parameter sensitivity
```

The correct solution depends on the workload.

---

# 36. Don't jump to OPTION(RECOMPILE)

`OPTION (RECOMPILE)` can produce a plan optimized for the current parameter values.

But recompiling has a cost.

Potential trade-off:

```text
Better per-execution plan
        vs
More compilation work
```

Use it when the workload justifies it.

---

# 37. Statistics and plan regression

Suppose:

```text
Monday:
Plan A → 100 ms
```

Statistics update:

```text
Tuesday:
Plan B → 4 sec
```

This does not automatically mean:

> "Statistics update was bad."

The new statistics may have caused the optimizer to make a different decision based on new information.

The real question is:

> **Why did the new plan perform worse?**

---

# 38. Data changes can make yesterday's plan obsolete

Example:

```text
January:
Active orders = 10,000
```

By August:

```text
Active orders = 50 million
```

A plan optimized for the January distribution may no longer be appropriate.

This is why plan management must account for changing data.

---

# 39. Ascending-key challenges

Time-based columns can have continuously increasing values:

```text
2026-08-01
2026-08-02
2026-08-03
...
```

New values may appear beyond the range represented by existing statistics.

SQL Server has mechanisms to address estimation challenges around changing data distributions, but time-based workloads still require careful monitoring.

---

# 40. Statistics on partitioned tables

Large partitioned tables introduce additional considerations.

You need to think about:

```text
Partition-level data distribution
Statistics maintenance
Incremental statistics where appropriate
Partition elimination
```

The goal is to keep estimates useful without unnecessarily processing the entire table.

---

# 41. Incremental statistics

For appropriate partitioned-table workloads, incremental statistics can allow statistics maintenance to be managed more efficiently across partitions.

Conceptually:

```text
Partition 1 → stats
Partition 2 → stats
Partition 3 → stats
...
```

rather than treating the entire dataset as one undifferentiated block for every maintenance operation.

---

# 42. Statistics and filtered indexes

A filtered index can have statistics reflecting its filtered subset.

That can be very valuable when the subset has a distinct distribution from the rest of the table.

Example:

```text
Entire table:
500 million rows

Open orders:
2 million rows
```

Statistics focused on the relevant subset can improve estimation for appropriate queries.

---

# 43. Histograms are not magic

Statistics don't store every possible fact about the data.

They are compact representations.

Therefore, estimation can still be wrong when:

```text
Data is highly skewed
Columns are correlated
Expressions are complex
Predicates are unusual
Data changes rapidly
```

Statistics improve estimates—they don't guarantee perfection.

---

# 44. Cardinality estimation errors compound

Suppose:

```text
Operator 1:
Estimated 100
Actual 10,000
```

Then the next operator receives the wrong expectation.

That can cause:

```text id="x8m4p2"
Wrong join choice
Wrong memory grant
Wrong parallelism
Wrong access method
```

Errors can propagate through the entire plan.

---

# 45. A plan can look reasonable but still be wrong

You may see:

```text
Index Seek
Nested Loops
Key Lookup
```

and think:

> "Looks efficient."

But if the seek actually returns:

```text
20 million rows
```

the plan may be terrible for the actual workload.

Always inspect cardinality.

---

# 46. Cost estimates aren't runtime measurements

The optimizer's cost model is useful for choosing between plans.

But:

```text
Estimated cost ≠ actual execution time
```

A plan with a lower estimated cost isn't automatically faster in the real world.

Measure actual runtime.

---

# 47. Statistics maintenance strategy

A mature environment considers:

```text
Automatic statistics
Manual updates where justified
Sampling
Large-table behavior
Partitioning
Workload timing
Plan stability
```

The goal isn't:

> "Statistics must always be perfectly current."

The goal is:

> **"Statistics must be sufficiently accurate for good optimization at acceptable maintenance cost."**

---

# 48. Troubleshooting bad estimates

Use this sequence:

```text id="m7q2v9"
Bad estimate
   ↓
Which operator?
   ↓
Which predicate?
   ↓
Which column?
   ↓
Which statistics?
   ↓
Is data skewed?
   ↓
Are columns correlated?
   ↓
Are statistics current?
   ↓
Is parameter sensitivity involved?
   ↓
What plan resulted?
```

---

# 49. Expert scenario

Query:

```sql
SELECT *
FROM dbo.Orders
WHERE CustomerId = @CustomerId;
```

Statistics suggest:

```text
Estimated = 20 rows
```

But for a particular customer:

```text
Actual = 8,000,000 rows
```

The plan uses:

```text
Nested Loops
+
Repeated lookups
```

Performance collapses.

The real problem isn't simply:

> "Nested Loops are bad."

The deeper issue is:

```text
Data skew
 ↓
Poor estimate
 ↓
Plan inappropriate for large parameter
```

---

# 50. Expert scenario #2

A report query suddenly starts spilling.

You discover:

```text
Estimated rows = 100
Actual rows = 2 million
```

The memory grant was too small.

Possible investigation:

```text
Statistics
Cardinality estimation
Predicate shape
Parameter behavior
```

Increasing server memory may not solve the underlying problem.

---

# 51. Expert scenario #3

After a large data load:

```text
Query latency ↑
CPU ↑
```

Query Store shows:

```text
Plan changed
```

Execution plan shows:

```text
Estimated rows drastically different
```

Investigate:

```text
Statistics
Data distribution
Plan selection
```

rather than immediately changing indexes.

---

# 52. Expert diagnostic hierarchy

When estimates are wrong:

```text
1. Check data distribution
2. Check statistics
3. Check predicates
4. Check correlations
5. Check parameter sensitivity
6. Check compatibility/optimizer behavior
7. Check resulting plan
```

---

# 53. Statistics don't exist in isolation

Remember:

```text
Statistics
    +
Indexes
    +
Data distribution
    +
Query predicates
    +
Parameters
    +
Optimizer model
    ↓
Execution plan
```

A plan is the result of many interacting factors.

---

# 54. Interview question

### "What are SQL Server statistics?"

Strong answer:

> "Statistics provide the optimizer with information about data distribution so it can estimate predicate and operator cardinalities. Those estimates influence cost calculations and execution-plan selection. Poor or stale statistics can contribute to inaccurate estimates and suboptimal plans."

---

# 55. Interview question

### "What is cardinality estimation?"

Strong answer:

> "Cardinality estimation is the optimizer's prediction of how many rows an operator or predicate will produce. Those estimates influence join algorithms, access methods, memory grants, and parallelism decisions."

---

# 56. Interview question

### "Estimated rows are very different from actual rows. What does that tell you?"

Strong answer:

> "It indicates a cardinality-estimation problem at that operator. I would investigate the predicate, relevant statistics, data skew, column correlation, parameter sensitivity, and whether statistics adequately represent the current data."

---

# 57. Interview question

### "Are stale statistics always the cause of a bad plan?"

Strong answer:

> "No. Poor estimates can result from stale statistics, but also from skew, correlations, expressions, parameter sensitivity, optimizer assumptions, or query shape. I would verify the specific estimation problem before changing statistics."

---

# 58. The complete optimization chain

Connect the last three lessons:

```text id="v3m8q1"
QUERY
  ↓
INDEX
  ↓
STATISTICS
  ↓
CARDINALITY ESTIMATE
  ↓
PLAN
  ↓
RESOURCE USAGE
  ↓
WAIT
  ↓
USER EXPERIENCE
```

This is the heart of SQL Server performance engineering.

---

# 59. Lesson 129 takeaway

Remember:

> **The optimizer can only make a good decision from the information and assumptions available to it.**

Your diagnostic chain should be:

```text id="p6x2n9"
BAD PERFORMANCE
      ↓
BAD PLAN?
      ↓
BAD ESTIMATE?
      ↓
WHY BAD ESTIMATE?
      ↓
STATISTICS?
DATA SKEW?
CORRELATION?
PARAMETER?
PREDICATE?
OPTIMIZER MODEL?
      ↓
CORRECT ROOT CAUSE
```

And the golden rule:

> **Don't fix the plan before understanding why the optimizer chose it.**

**Next: Lesson 130 — Parameter Sniffing, Parameter Sensitivity & Advanced Plan Stability — Expert Level.**
