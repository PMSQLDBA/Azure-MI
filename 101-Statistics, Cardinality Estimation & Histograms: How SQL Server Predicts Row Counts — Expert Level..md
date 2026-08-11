# Lesson 134 — Statistics, Cardinality Estimation & Histograms: How SQL Server Predicts Row Counts — Expert Level

Today we go deeper into one of the most important ideas in SQL Server performance:

> **SQL Server has to predict how many rows each operation will produce before it chooses an execution plan.**

Those predictions are called **cardinality estimates**.

When the prediction is good, the optimizer can often choose an excellent plan.

When the prediction is badly wrong, everything downstream can go wrong.

---

## 1. The optimizer is making predictions

Suppose you run:

```sql
SELECT *
FROM dbo.Orders
WHERE CustomerId = 123;
```

Before execution, SQL Server needs to estimate:

```text
How many rows will CustomerId = 123 return?
```

Maybe the optimizer predicts:

```text
Estimated = 500 rows
```

But reality is:

```text
Actual = 2,000,000 rows
```

That is a major estimation error.

---

# 2. Why cardinality matters so much

The estimated row count influences many decisions:

```text
Cardinality estimate
        ↓
Join strategy
        ↓
Access method
        ↓
Memory grant
        ↓
Parallelism
        ↓
Sort strategy
        ↓
Aggregation strategy
```

One bad estimate can therefore affect the entire plan.

---

# 3. Statistics are the optimizer's evidence

SQL Server maintains **statistics** describing data distribution.

Conceptually:

```text
Table data
   ↓
Statistics
   ↓
Distribution information
   ↓
Optimizer
   ↓
Cardinality estimate
```

Statistics aren't simply:

> "How many rows are in the table?"

They contain information about how values are distributed.

---

# 4. Basic statistics components

A statistics object contains information such as:

```text
Total rows
Rows sampled
Density information
Histogram
```

The exact internal details depend on the SQL Server version and statistics type, but these are the key concepts to understand.

---

# 5. Histogram

A histogram describes the distribution of values for a statistics leading column.

Conceptually:

```text
Value range       Approximate distribution

1 – 100           ███████
101 – 200         ███
201 – 300         ███████████
301 – 400         ██
...
```

This lets the optimizer reason about selectivity.

---

# 6. Why a histogram matters

Imagine:

```text
CustomerId
```

has:

```text
1,000,000 distinct values
```

but one customer owns:

```text
30% of all orders
```

while most customers own only:

```text
0.0001%
```

A simple average would be misleading.

The histogram helps SQL Server recognize distribution patterns.

---

# 7. Statistics are not indexes

This distinction is essential.

### Index

Primarily provides an efficient way to access data.

### Statistics

Provide information about data distribution for optimization.

You can have:

```text
Statistics without a useful index
```

and:

```text
An index whose associated statistics aren't sufficient
```

They serve different purposes.

---

# 8. Indexes often create statistics

When you create an index, SQL Server maintains statistics associated with the index key.

For example:

```sql
CREATE INDEX IX_Orders_CustomerId
ON dbo.Orders(CustomerId);
```

This provides an access path and statistics useful for optimization.

---

# 9. Auto-created statistics

SQL Server can automatically create column statistics when appropriate if the relevant database option is enabled.

Conceptually:

```text
Query references column
        ↓
No suitable statistics
        ↓
Auto-create statistics
        ↓
Optimizer gets distribution information
```

---

# 10. Auto-update statistics

Statistics can become stale as data changes.

SQL Server can automatically update statistics under appropriate conditions.

Conceptually:

```text
Data changes
    ↓
Statistics age
    ↓
Threshold reached
    ↓
Statistics update
```

---

# 11. Why stale statistics matter

Imagine statistics were created when:

```text
Rows = 1 million
```

Later:

```text
Rows = 500 million
```

The old distribution information may no longer represent reality accurately.

That can lead to:

```text
Bad estimates
 ↓
Poor plan choices
```

---

# 12. Statistics sampling

Statistics don't necessarily need to inspect every row every time.

SQL Server can sample data.

Conceptually:

```text
500 million rows
       ↓
Representative sample
       ↓
Statistics
```

This makes statistics maintenance more practical.

---

# 13. Sampling tradeoff

More sampling can provide better distribution information but can require more resources.

Less sampling can reduce maintenance cost but may miss important distribution characteristics.

The optimizer and statistics mechanisms balance accuracy and cost.

---

# 14. Fullscan

A statistics update can be performed using a full scan when appropriate.

Conceptually:

```text
Entire table/index
      ↓
Statistics
```

This can provide highly detailed information but may be expensive on very large objects.

---

# 15. Histogram limitations

A histogram is powerful, but it isn't a perfect representation of every value.

It contains a finite number of steps.

Therefore:

```text
Millions of distinct values
        ↓
Finite histogram representation
```

The optimizer must sometimes infer selectivity.

---

# 16. Histogram steps

A histogram contains boundaries representing ranges of values.

Conceptually:

```text
Step 1 → 1–100
Step 2 → 101–500
Step 3 → 501–5,000
Step 4 → 5,001–50,000
```

The optimizer uses these ranges to estimate predicates.

---

# 17. Equality predicates

For:

```sql
WHERE CustomerId = 123
```

SQL Server may use histogram information to estimate how many rows match.

If the value is represented accurately in the relevant statistics information, the estimate can be quite good.

---

# 18. Range predicates

For:

```sql
WHERE OrderDate >= '2026-01-01'
  AND OrderDate < '2026-02-01'
```

the optimizer needs to estimate the number of rows within a range.

The histogram becomes especially important here.

---

# 19. Data skew

Suppose:

```text
Status = 'Completed' → 95%
Status = 'Pending'   → 4%
Status = 'Cancelled' → 1%
```

A query for:

```sql
WHERE Status = 'Cancelled'
```

has very different selectivity from:

```sql
WHERE Status = 'Completed'
```

Good statistics help the optimizer recognize this.

---

# 20. Why averages can fail

Suppose:

```text
Average rows/customer = 1,000
```

But reality is:

```text
Customer A = 2
Customer B = 4
Customer C = 500
Customer D = 50,000,000
```

An average estimate is useless for some values.

This connects directly to the parameter-sensitivity lesson.

---

# 21. Statistics and parameter sensitivity

The relationship is:

```text
Data skew
 ↓
Different selectivity
 ↓
Different cardinality estimates
 ↓
Different ideal plans
```

If the same cached plan is reused across very different values, performance can vary dramatically.

---

# 22. Cardinality estimation models

SQL Server has evolved its cardinality estimation behavior over time.

The broad distinction you'll encounter is:

```text
Legacy CE
vs
Newer CE
```

The newer cardinality estimator introduced major changes in how SQL Server models certain data distributions and predicate relationships.

---

# 23. Why CE changes matter

A query can produce:

```text
Old CE:
Estimated = 10,000

New CE:
Estimated = 100,000
```

The actual number might be:

```text
Actual = 90,000
```

The newer estimate may lead to a substantially better plan.

---

# 24. Compatibility level

Cardinality-estimation behavior can depend on:

```text
SQL Server version
Database compatibility level
Query-level configuration/hints
```

Therefore, when a plan changes after an upgrade, CE behavior should be part of the investigation.

---

# 25. Correlated columns

This is a classic estimation challenge.

Suppose:

```text
State
City
```

are strongly related.

For example:

```text
State = Louisiana
City = New Orleans
```

is a realistic combination.

But:

```text
State = Louisiana
City = Seattle
```

isn't.

If SQL Server treats predicates as more independent than they actually are, estimates can become inaccurate.

---

# 26. Independence assumption

Conceptually:

```text
Predicate A selectivity
        ×
Predicate B selectivity
        ↓
Combined estimate
```

This works better when predicates are reasonably independent.

It can fail when columns are highly correlated.

---

# 27. Example

Suppose:

```text
WHERE State = 'Louisiana'
  AND City = 'New Orleans'
```

If the optimizer estimates each predicate independently without adequately capturing their relationship, the combined estimate may be inaccurate.

That can affect:

```text
Join choice
Memory grants
Parallelism
Access methods
```

---

# 28. Multi-column statistics

When columns are commonly queried together, multi-column statistics can provide additional distribution information.

Example:

```sql
CREATE STATISTICS ST_Customer_State_City
ON dbo.Customer(State, City);
```

This can help the optimizer understand relationships between columns in appropriate scenarios.

---

# 29. Column order matters

For multi-column statistics:

```text
(State, City)
```

is not conceptually identical to:

```text
(City, State)
```

The leading column has a special role in the histogram.

This is similar to the importance of key order in composite indexes, although statistics and indexes serve different purposes.

---

# 30. Filtered statistics

Filtered statistics describe a subset of a table.

For example:

```text
Status = 'Active'
```

could represent only a small portion of a massive table.

A filtered statistics object can provide more targeted distribution information for that subset.

---

# 31. Filtered indexes vs filtered statistics

Again, don't confuse them.

### Filtered index

Provides an access path for a subset of rows.

### Filtered statistics

Provide distribution information about a subset.

You can use either, depending on the problem.

---

# 32. Ascending-key problem

Date columns are a classic example.

Imagine:

```text
OrderDate
```

continuously receives new values:

```text
Yesterday
Today
Tomorrow
...
```

Statistics may lag behind the newest values.

The optimizer may therefore have difficulty estimating queries against values beyond the known histogram boundary.

---

# 33. Why ascending keys matter

Suppose statistics were last updated through:

```text
2026-07-31
```

Today's data contains:

```text
2026-08-10
```

A query for:

```sql
WHERE OrderDate >= '2026-08-09'
```

may involve values that weren't well represented when the statistics were built.

---

# 34. Statistics maintenance strategy

For heavily changing tables, consider:

```text
Auto-update behavior
Manual updates
Appropriate sampling
Incremental statistics where applicable
Maintenance scheduling
```

Don't automatically choose the most expensive statistics operation.

---

# 35. Incremental statistics

For partitioned tables, incremental statistics can maintain statistics at the partition level in supported scenarios.

Conceptually:

```text
Partition 1 → stats
Partition 2 → stats
Partition 3 → stats
...
```

This can make statistics maintenance more scalable for large partitioned objects.

---

# 36. Partitioning doesn't automatically solve estimation

Partitioning can help with:

```text
Partition elimination
Maintenance
Data management
```

But poor statistics can still cause bad estimates within relevant partitions or across the overall workload.

Partitioning isn't a replacement for good statistics.

---

# 37. Estimated vs actual rows

When diagnosing an execution plan, look for:

```text
Estimated Rows
Actual Rows
```

Then compare them.

Example:

```text
Estimated = 1,000
Actual    = 1,100
```

Excellent.

But:

```text
Estimated = 1,000
Actual    = 10,000,000
```

is a huge warning sign.

---

# 38. Estimation error ratio

A useful diagnostic concept is:

```text
Actual / Estimated
```

Example:

```text
10,000 / 1,000 = 10×
```

or:

```text
10,000,000 / 1,000 = 10,000×
```

Large discrepancies deserve investigation.

---

# 39. Don't obsess over exact equality

The optimizer doesn't need:

```text
Estimated = Actual
```

for every operator.

Real workloads contain uncertainty.

The goal is:

> **Estimates accurate enough to support good plan choices.**

---

# 40. One estimate can be "wrong" but harmless

Suppose:

```text
Estimated = 100
Actual = 300
```

If both values still lead to the same efficient plan, the estimation difference may have little practical impact.

The important question is:

> **Did the estimation error cause a bad decision?**

---

# 41. The dangerous estimate

Example:

```text
Estimated = 10
Actual = 10,000,000
```

This can cause:

```text
Nested Loops
+
Repeated Lookups
```

when:

```text
Hash Join
+
Scan
```

would have been better.

Now the estimate error is operationally significant.

---

# 42. Join cardinality

Suppose:

```text
Table A = 1 million rows
Table B = 1 million rows
```

But the join produces:

```text
Actual = 100 million rows
```

The optimizer needs to estimate this intermediate result before choosing downstream operations.

This is why join estimation errors can compound quickly.

---

# 43. Cardinality explosion

Consider:

```text
A
 ↓
Join
 ↓
100 rows

Join B
 ↓
10,000 rows

Join C
 ↓
10 million rows
```

A small estimation error early in the plan can become enormous downstream.

---

# 44. Join selectivity

The optimizer considers how likely rows are to match.

For example:

```text
Customer.CustomerId
=
Orders.CustomerId
```

If the join key is highly selective and well understood, estimates can be good.

If distributions are unusual or statistics are weak, estimates may suffer.

---

# 45. Many-to-many joins

A join isn't always:

```text
1 customer → 1 order
```

It can be:

```text
Customer
   ↓
1-to-many
   ↓
Orders
```

or:

```text
A
 ↕
Many-to-many
 ↕
B
```

Understanding expected join multiplicity is critical to interpreting estimates.

---

# 46. Missing statistics

If SQL Server lacks useful statistics for a predicate, it may need to use less precise estimation strategies.

This can happen with:

```text
Complex expressions
Certain computed expressions
Temporary objects
Unusual predicates
Newly introduced data
```

The solution depends on the specific scenario.

---

# 47. Functions can hide useful information

Consider:

```sql
WHERE YEAR(OrderDate) = 2026;
```

The optimizer may have less direct information about the resulting predicate than with a range:

```sql
WHERE OrderDate >= '20260101'
  AND OrderDate <  '20270101';
```

The second form also tends to be more naturally aligned with index access.

---

# 48. SARGability and estimation

SARGability isn't only about index seeks.

Predicate form can also affect how effectively SQL Server reasons about selectivity.

A search argument such as:

```text
OrderDate >= X
```

is generally easier to reason about than applying an arbitrary function to the column.

---

# 49. Expressions and estimates

Consider:

```sql
WHERE Price * Quantity > 1000;
```

Estimating such expressions can be more challenging than predicates directly involving stored columns with statistics.

Complex expressions can therefore contribute to estimation uncertainty.

---

# 50. Temporary tables and statistics

Temporary tables can have statistics.

This can make them useful in complex workflows where intermediate row counts are difficult to estimate from a single giant statement.

Example:

```text
Complex query
   ↓
#Results
   ↓
Statistics
   ↓
Next query
```

The second phase can sometimes make better decisions.

---

# 51. Why breaking a query can help

Suppose:

```text
One giant query
 ↓
Optimizer must estimate everything at once
```

versus:

```text
Step 1
 ↓
Materialize
 ↓
Actual intermediate result known
 ↓
Step 2
```

Materialization can trade:

```text
TempDB + additional I/O
```

for:

```text
Better information + simpler optimization
```

This is a strategic decision, not automatically a best practice.

---

# 52. Statistics update doesn't guarantee a better plan

You might update statistics and see:

```text
Statistics refreshed
```

but the query remains slow.

Possible reasons:

```text
Index design
Query shape
Parameter sensitivity
Join correlation
Memory pressure
Blocking
Bad access path
Other estimation limitations
```

Statistics are one component of optimization.

---

# 53. Statistics can change the plan

Conversely:

```text
Statistics updated
 ↓
Cardinality estimate changes
 ↓
Join strategy changes
 ↓
Memory grant changes
 ↓
Plan changes
```

This is why statistics updates can sometimes appear to "randomly" change query performance.

The optimizer isn't random—it is reacting to changed information.

---

# 54. Plan regression after statistics update

Suppose:

```text
Old statistics
 ↓
Plan A
 ↓
Fast
```

Then:

```text
Updated statistics
 ↓
New estimate
 ↓
Plan B
 ↓
Slow
```

Possible explanations include:

```text
Plan B is actually better for current data
```

or:

```text
The new estimate caused a poor plan choice
```

Investigate before reverting statistics.

---

# 55. Statistics and Query Store

Query Store is useful for correlating:

```text
Query
+
Plan
+
Runtime
+
Time
```

with changes in:

```text
Statistics
Compatibility level
Indexes
Data distribution
```

This creates a timeline for performance changes.

---

# 56. Expert troubleshooting sequence

When estimates look suspicious:

```text
1. Identify first major estimate error
2. Identify columns/predicates involved
3. Check statistics
4. Check histogram/distribution
5. Check statistics age
6. Check data skew
7. Check predicate form
8. Check column correlation
9. Check parameter sensitivity
10. Test updated statistics
11. Compare plans
12. Measure runtime
```

---

# 57. Example

Suppose:

```text
Predicate:
WHERE CustomerId = @CustomerId
```

Estimated:

```text
500 rows
```

Actual:

```text
4,000,000 rows
```

Check:

```text
CustomerId distribution
```

You discover:

```text
99% of customers → small
1 customer        → 30% of table
```

Now you have a plausible explanation:

> **Highly skewed data is producing parameter-sensitive cardinality behavior.**

---

# 58. Example #2

Query:

```sql
WHERE State = @State
  AND City = @City;
```

Estimate:

```text
10 rows
```

Actual:

```text
2,000,000 rows
```

Investigate:

```text
State distribution
City distribution
State/City correlation
Statistics
```

The issue may be insufficient information about the relationship between the columns.

---

# 59. Example #3

Query:

```sql
WHERE OrderDate >= @StartDate;
```

Recent dates are dramatically more common than old dates.

Statistics haven't been refreshed appropriately.

You observe:

```text
Estimated = 100
Actual = 5,000,000
```

This points toward:

```text
Data distribution
+
Statistics freshness
+
Ascending-key behavior
```

---

# 60. Statistics are evidence, not truth

This is the expert mindset.

The optimizer doesn't know the future.

It uses:

```text
Statistics
+
Metadata
+
Query predicates
+
Optimizer model
```

to make a prediction.

Sometimes the prediction will be wrong.

---

# 61. Why "update statistics" is not a universal answer

Imagine the actual problem is:

```text
Parameter sensitivity
```

Updating statistics may change the estimates but still leave the fundamental issue:

```text
Small parameter → Plan A
Large parameter → Plan B
```

Similarly, if the problem is:

```text
Blocking
```

statistics won't solve it.

Always identify the bottleneck first.

---

# 62. The estimation chain

Memorize this:

```text
DATA
 ↓
STATISTICS
 ↓
CARDINALITY ESTIMATE
 ↓
PLAN CHOICE
 ↓
MEMORY / CPU / I/O
 ↓
RUNTIME
```

This chain is one of the most important concepts in SQL Server performance tuning.

---

# 63. Expert mental model

When you see a bad plan, don't immediately ask:

> "What index is missing?"

Instead ask:

```text
What did SQL Server believe?
        ↓
Why did it believe that?
        ↓
Was the estimate reasonable?
        ↓
If not, what information was missing?
        ↓
Did the estimate cause the bad plan?
```

That is optimizer-level thinking.

---

# 64. Interview question

### "What are SQL Server statistics?"

Strong answer:

> "Statistics are distribution information maintained by SQL Server to help the optimizer estimate cardinalities and choose execution plans. They include information such as row counts, density, and histogram data."

---

# 65. Interview question

### "Why are cardinality estimates important?"

Strong answer:

> "Cardinality estimates influence access methods, join algorithms, memory grants, aggregation strategies, and parallelism. A significant estimation error can therefore cause a cascade of poor downstream decisions."

---

# 66. Interview question

### "What is a histogram?"

Strong answer:

> "A histogram is a distribution structure in a statistics object that describes how values are distributed across ranges for the statistics' leading column. The optimizer uses it to estimate predicate selectivity."

---

# 67. Interview question

### "What happens when statistics are stale?"

Strong answer:

> "The optimizer may be working with a distribution that no longer represents the underlying data, which can lead to inaccurate cardinality estimates and inappropriate execution plans."

---

# 68. Interview question

### "Are estimated and actual rows expected to be identical?"

Strong answer:

> "No. Estimates are predictions. The important question is whether the estimates are accurate enough to produce good plan decisions. A modest estimation error may be harmless, while a large error can cause major performance problems."

---

# 69. Interview question

### "Why can two predicates produce a bad combined estimate?"

Strong answer:

> "One reason is correlation between columns. If the optimizer assumes predicates are more independent than they really are, multiplying selectivities can produce a significantly inaccurate estimate. Multi-column statistics can sometimes provide better information."

---

# 70. Lesson 134 takeaway

The central concept:

> **SQL Server doesn't choose a plan based on reality—it chooses a plan based on its estimate of reality.**

The complete chain is:

```text
                 DATA
                   ↓
              STATISTICS
                   ↓
              HISTOGRAM
                   ↓
        CARDINALITY ESTIMATE
                   ↓
          OPTIMIZER DECISIONS
          /        |         \
       JOIN      MEMORY      ACCESS
      CHOICE     GRANT       METHOD
          \        |         /
                   ↓
                PLAN
                   ↓
              ACTUAL DATA
                   ↓
          ESTIMATE vs ACTUAL
                   ↓
              PERFORMANCE
```

And the expert troubleshooting question is:

> **"Where did SQL Server's model of the data diverge from reality, and did that divergence cause the bad plan?"**

**Next: Lesson 135 — Index Internals: B-Trees, Pages, Levels, Included Columns & Covering Strategies — Expert Level.**
