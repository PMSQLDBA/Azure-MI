# Lesson 138 — Cardinality Estimation Deep Dive: Histograms, Density, Correlation, Skew, Selectivity & Why SQL Server Gets Row Counts Wrong

Today we go deeper into **why SQL Server sometimes estimates the wrong number of rows**—and why that single mistake can cascade into a terrible execution plan.

---

## 1. What is cardinality estimation?

Cardinality estimation (CE) is the optimizer's attempt to answer:

> **"How many rows will this operator produce?"**

For example:

```text
Table:
10,000,000 rows

Predicate:
CustomerId = 42

Optimizer estimate:
500 rows
```

If reality is:

```text
Actual:
450 rows
```

that's a reasonable estimate.

But if reality is:

```text
Actual:
2,000,000 rows
```

the optimizer has a serious information problem.

---

# 2. Why row estimates matter so much

The optimizer uses estimated cardinality to choose:

```text
Join algorithm
Join order
Index access
Scan vs seek
Memory grant
Parallelism
Aggregation strategy
Sort strategy
```

So:

```text
Bad estimate
    ↓
Bad cost calculation
    ↓
Potentially bad plan
```

This is one of the most important chains in SQL Server performance tuning.

---

# 3. The optimizer doesn't know the future

SQL Server generally doesn't execute every possible plan and compare them.

Instead it estimates.

Conceptually:

```text
Possible Plan A → estimated cost = 10
Possible Plan B → estimated cost = 100
Possible Plan C → estimated cost = 20

Choose A
```

If the estimates are wrong:

```text
Actual Plan A = terrible
Actual Plan B = excellent
```

The optimizer may still have made the rational decision based on the information it had.

---

# 4. Where does the information come from?

A major source is:

> **Statistics**

Statistics contain information about data distribution that helps SQL Server estimate selectivity.

Think of statistics as the optimizer's **map of the data**.

---

# 5. Statistics aren't the data

This distinction is important.

The optimizer usually doesn't need to inspect every row just to estimate a predicate.

Instead it can use statistical summaries such as:

```text
Histogram
Density information
Distinct-value information
Sampling-derived information
```

Conceptually:

```text
Millions of rows
      ↓
Statistics
      ↓
Compact description of distribution
      ↓
Optimizer estimates
```

---

# 6. Histogram

For statistics on a column, SQL Server can maintain a histogram describing the distribution of values.

Imagine:

```text
Value range       Approx rows
-----------       -----------
1–100                 900
101–200               850
201–300               920
301–400               870
...
```

The histogram gives the optimizer information about how values are distributed.

---

# 7. Why histograms matter

Suppose a column contains:

```text
CustomerId
```

and most customers have approximately:

```text
1,000 orders
```

but one customer has:

```text
50,000,000 orders
```

A uniform assumption would be dangerously inaccurate.

A histogram can provide information that the distribution isn't uniform.

---

# 8. Data skew

This is called **skew**.

Example:

```text
Customer A → 10 rows
Customer B → 12 rows
Customer C → 15 rows
Customer D → 50,000,000 rows
```

The distribution is highly skewed.

A simple average can hide this completely.

---

# 9. Average is not enough

Suppose:

```text
Total rows = 10,000,000
Distinct customers = 10,000
```

Average:

```text
1,000 rows/customer
```

But maybe the actual distribution is:

```text
9,999 customers → ~10 rows each
1 customer      → ~9,900,000 rows
```

The average says:

```text
1,000
```

while one customer actually has:

```text
9,900,000
```

That difference can completely change the best plan.

---

# 10. Selectivity

Selectivity describes how much of the data satisfies a predicate.

For example:

```text
10,000,000 total rows
100,000 qualifying rows
```

Selectivity is approximately:

```text
100,000 / 10,000,000
= 1%
```

Low percentage means a highly selective predicate.

---

# 11. High selectivity vs low selectivity

A predicate returning:

```text
10 rows out of 10 million
```

is highly selective.

A predicate returning:

```text
8 million out of 10 million
```

is not.

This influences whether an index seek is attractive.

---

# 12. Why selectivity matters for access paths

Consider:

```text
10,000,000 rows
```

Query A:

```text
WHERE OrderId = 12345
```

Maybe:

```text
1 row
```

A seek is likely attractive.

Query B:

```text
WHERE Status = 'Active'
```

Maybe:

```text
9,000,000 rows
```

A scan may be more efficient.

The optimizer needs cardinality estimates to make that decision.

---

# 13. Density

Density is another statistical concept used in cardinality estimation.

At a high level, density represents information about how frequently values repeat / how many distinct combinations exist.

For a column with many distinct values:

```text
CustomerId
```

density may indicate that individual values are relatively uncommon.

For a column with few distinct values:

```text
Status
```

density is much higher.

---

# 14. Low-cardinality columns

Consider:

```text
Status
------
Active
Inactive
Pending
```

Only three values exist.

An index on `Status` may not be very selective for:

```sql
WHERE Status = 'Active'
```

if 90% of rows are active.

---

# 15. High-cardinality columns

Now consider:

```text
OrderId
```

with:

```text
10,000,000 distinct values
```

A predicate:

```sql
WHERE OrderId = 1234567
```

is highly selective.

This distinction matters enormously for index design.

---

# 16. Histograms aren't perfect

A histogram is a summary, not a complete copy of the column.

SQL Server has finite space for histogram information.

Therefore it cannot perfectly represent every possible distribution.

This becomes important with:

```text
Huge data sets
+
Highly skewed values
+
Frequent changes
```

---

# 17. Histogram steps

Conceptually, a histogram divides the value domain into ranges/steps.

For example:

```text
Step 1 → 1–100
Step 2 → 101–1,000
Step 3 → 1,001–10,000
Step 4 → 10,001–100,000
```

The optimizer uses information from these steps when estimating predicates.

---

# 18. Boundary values matter

Histogram estimation behaves differently depending on whether the requested value lies:

```text
Inside a known histogram step
```

or:

```text
Outside / beyond known information
```

This is one reason newly inserted values can sometimes be difficult to estimate accurately when statistics don't yet represent the new distribution.

---

# 19. Ascending-key problem

Imagine an ever-increasing column:

```text
OrderId
1
2
3
...
10,000,000
```

Statistics were built when the maximum value was:

```text
10,000,000
```

Then millions of new rows arrive:

```text
10,000,001
...
12,000,000
```

The statistics may not fully represent this new range until they are refreshed appropriately.

This can lead to estimation challenges.

---

# 20. Why automatic statistics updates matter

SQL Server can automatically update statistics under applicable conditions.

The goal is to keep the optimizer's information reasonably current.

But:

> **"Automatically maintained" does not mean "always perfectly current."**

Large, volatile, skewed tables can still present estimation challenges.

---

# 21. Stale statistics

Suppose yesterday:

```text
Status = 'Active' → 10%
```

Today:

```text
Status = 'Active' → 90%
```

If statistics don't reflect the new distribution sufficiently, the optimizer may make poor estimates.

That can affect:

```text
Join selection
Memory grants
Parallelism
Index selection
```

---

# 22. Statistics update is not just about age

People sometimes think:

> "Statistics older than X days are stale."

Age alone isn't the right metric.

The important question is:

> **Has the underlying data distribution changed enough to make the statistics less useful?**

A statistic can be weeks old and still useful.

Another can become problematic much sooner on a heavily changing table.

---

# 23. Sampling

Statistics can be built or updated using sampled data rather than necessarily inspecting every row.

Conceptually:

```text
Large table
 ↓
Sample
 ↓
Estimate distribution
 ↓
Statistics
```

Sampling improves efficiency but introduces approximation.

---

# 24. Full scan

A full-scan statistics operation examines the entire relevant data set.

Conceptually:

```text
All rows
 ↓
Exact distribution information
 ↓
Statistics
```

This can improve statistical accuracy but may cost substantially more resources.

---

# 25. Sampling tradeoff

Think:

```text
More sampling
   ↓
Potentially better statistics
   ↓
More work
```

versus:

```text
Less sampling
   ↓
Less maintenance cost
   ↓
Potentially less precise distribution information
```

The right choice depends on workload and data characteristics.

---

# 26. Correlated columns

This is where cardinality estimation gets really interesting.

Suppose a table contains:

```text
State
City
```

and you query:

```sql
WHERE State = 'Louisiana'
  AND City = 'New Orleans'
```

These columns aren't independent.

Knowing:

```text
State = Louisiana
```

strongly affects the probability of:

```text
City = New Orleans
```

---

# 27. Independence assumption

A simplistic model might reason as though predicates were independent:

```text
P(State = LA AND City = New Orleans)
≈
P(State = LA) × P(City = New Orleans)
```

But real-world data often violates independence.

That can produce bad estimates.

---

# 28. Correlation example

Suppose:

```text
State = Louisiana → 5%
City = New Orleans → 4%
```

A naive independence calculation gives:

```text
5% × 4%
= 0.2%
```

But perhaps almost all "New Orleans" rows are in Louisiana.

The true result could be closer to:

```text
4%
```

That's a 20× difference.

---

# 29. Why multi-column statistics can help

If the workload frequently filters on:

```text
State
+
City
```

statistics that capture multi-column relationships can provide better information than independent single-column statistics.

This is one reason column combinations matter in database design.

---

# 30. But don't create statistics blindly

More statistics aren't automatically better.

They introduce:

```text
Maintenance cost
Storage
Update overhead
```

and don't necessarily solve every estimation problem.

Statistics should support actual query patterns.

---

# 31. Correlation also appears in joins

Suppose:

```text
Orders.CustomerId
Customers.CustomerId
```

are naturally correlated through the relationship.

Join cardinality depends heavily on:

```text
Distinct values
Distribution
Nulls
Duplicates
Filters
```

A bad estimate on either side can distort the join estimate.

---

# 32. Join cardinality

Suppose:

```text
Customers = 1,000,000 rows
Orders    = 50,000,000 rows
```

and:

```text
Average orders/customer = 50
```

A join:

```sql
Customers c
JOIN Orders o
  ON o.CustomerId = c.CustomerId
```

could produce approximately:

```text
50,000,000 matching rows
```

before additional filters.

But if the optimizer estimates only:

```text
500,000
```

the plan selection may be very different.

---

# 33. Many-to-many joins

Joins can become particularly difficult when both sides contain duplicates for the join key.

Example:

```text
A:
Key = 10 → 100 rows

B:
Key = 10 → 200 rows
```

The join can produce:

```text
100 × 200
= 20,000 rows
```

for that key.

This multiplication can surprise people.

---

# 34. Filtered predicates

Consider:

```sql
WHERE Status = 'Active'
  AND Region = 'West'
```

If:

```text
Status
```

and:

```text
Region
```

are correlated, estimating each independently can be inaccurate.

The more predicates interact, the harder cardinality estimation can become.

---

# 35. Functions and expressions

Predicates involving expressions can complicate estimation.

For example:

```sql
WHERE YEAR(OrderDate) = 2026
```

The optimizer isn't simply looking at:

```text
OrderDate = a known boundary
```

It is dealing with a transformed value.

This can make both access and estimation more difficult.

---

# 36. SARGability revisited

Compare:

```sql
WHERE OrderDate >= '20260101'
  AND OrderDate <  '20270101'
```

with:

```sql
WHERE YEAR(OrderDate) = 2026
```

The first directly describes a range on the column.

That is generally much easier to use with an index and statistics.

---

# 37. Implicit conversion revisited

Suppose:

```text
Column = INT
Parameter = VARCHAR
```

The conversion can affect both:

```text
Access path
+
Cardinality estimation
```

So data-type mismatches can create problems beyond simple CPU overhead.

---

# 38. NULLs

NULL introduces another layer of complexity.

Remember:

```text
NULL ≠ 0
NULL ≠ ''
NULL ≠ NULL
```

SQL's three-valued logic means predicates involving NULL behave differently from ordinary equality comparisons.

Statistics and estimates must account for the presence of NULL values.

---

# 39. Three-valued logic

SQL predicates can evaluate to:

```text
TRUE
FALSE
UNKNOWN
```

For example:

```sql
NULL = NULL
```

is not TRUE.

It is UNKNOWN.

This matters when reasoning about:

```text
WHERE
JOIN
NOT IN
```

and cardinality.

---

# 40. NOT IN and NULL

A classic trap:

```sql
WHERE CustomerId NOT IN (
    SELECT CustomerId
    FROM SomeTable
)
```

If the subquery contains NULL, three-valued logic can produce surprising results.

This isn't just a syntax issue.

It can influence both correctness and optimization.

---

# 41. Parameter sniffing

When a parameterized statement is compiled, SQL Server can use the parameter value available during compilation to estimate cardinality.

This is commonly called:

> **Parameter sniffing.**

The concept itself isn't bad.

It can be extremely beneficial.

---

# 42. Why parameter sniffing can become a problem

Suppose:

```text
CustomerId = 1
```

returns:

```text
10 rows
```

while:

```text
CustomerId = 999
```

returns:

```text
10,000,000 rows
```

A plan optimized for one value may be inappropriate for the other.

---

# 43. Parameter-sensitive workloads

The underlying issue is often:

```text
One query shape
+
Very different data distributions
+
One reused plan
```

This is why parameter-sensitive workloads deserve special attention.

---

# 44. Cardinality errors can be multiplicative

Imagine:

```text
Operator 1:
Estimate = 100
Actual   = 10,000
```

Then a join uses that estimate.

The next operator estimates:

```text
1,000 rows
```

but reality becomes:

```text
1,000,000 rows
```

Errors can propagate through the plan.

So a small initial estimation error can become a massive downstream problem.

---

# 45. The domino effect

Think of it like:

```text
Bad statistic
     ↓
Bad selectivity estimate
     ↓
Bad row estimate
     ↓
Bad join choice
     ↓
Bad memory grant
     ↓
Spill
     ↓
TempDB activity
     ↓
Slow query
```

This is why cardinality estimation is central to performance tuning.

---

# 46. Memory grant feedback

Modern SQL Server features can adjust memory-grant behavior based on observed execution characteristics in applicable scenarios.

Conceptually:

```text
First execution
      ↓
Observed memory behavior
      ↓
Feedback
      ↓
Future execution adjustment
```

This can help mitigate some estimation-related memory problems.

---

# 47. Cardinality estimation isn't always wrong

An important mindset:

> **An estimate doesn't have to exactly match reality to produce a good plan.**

For example:

```text
Estimated = 10,000
Actual    = 12,000
```

The plan may be perfectly fine.

The concern is when errors are large enough to materially change plan decisions.

---

# 48. Error magnitude matters

Compare:

```text
Estimate = 10,000
Actual   = 12,000
```

with:

```text
Estimate = 10
Actual   = 10,000,000
```

The second is much more likely to cause downstream problems.

---

# 49. A practical way to inspect estimates

When reading an actual plan, mentally calculate:

```text
Actual / Estimated
```

For example:

```text
Actual = 1,000,000
Estimated = 100
```

Then:

```text
1,000,000 / 100
= 10,000×
```

That's a huge estimation error.

---

# 50. Don't only inspect the final operator

The final SELECT operator might show:

```text
Actual = 100 rows
Estimated = 100 rows
```

and look perfect.

But somewhere earlier:

```text
Actual = 10 million
Estimated = 100
```

may have caused massive unnecessary work.

Always inspect the intermediate operators.

---

# 51. Find the first major estimation error

A powerful technique:

> **Find where the estimate first diverges dramatically from reality.**

For example:

```text
Operator A
Estimate 1,000
Actual 1,100     ← fine

Operator B
Estimate 10,000
Actual 9,000     ← fine

Operator C
Estimate 20,000
Actual 5,000,000 ← major divergence
```

Operator C becomes a prime investigation target.

---

# 52. Why the first error matters

If an operator suddenly gets:

```text
5,000,000 actual rows
```

instead of:

```text
20,000 estimated
```

everything downstream is now operating on a bad assumption.

The downstream operators may be innocent.

---

# 53. Statistics update isn't always the solution

Suppose you find:

```text
Estimate ≠ Actual
```

Don't immediately run:

```sql
UPDATE STATISTICS ...
```

First determine **why** the estimate is wrong.

Possible causes include:

```text
Statistics freshness
Data skew
Correlation
Expression
Parameter sensitivity
Join complexity
Data-type mismatch
```

Updating statistics may help—but it isn't guaranteed.

---

# 54. Histogram limitations

A histogram is primarily associated with a leading statistics column.

This means the order of columns in a multi-column statistic matters.

For example:

```text
(A, B, C)
```

is not statistically equivalent to:

```text
(B, A, C)
```

for all estimation purposes.

---

# 55. Leading column concept

Suppose you have:

```text
Statistics:
(A, B)
```

The histogram is primarily based on:

```text
A
```

The additional information about:

```text
B
```

helps in other ways, but the histogram isn't simply an independent full histogram for both columns.

This is one reason composite statistics/indexes require careful design.

---

# 56. Statistics and indexes

Indexes and statistics are related but distinct.

An index provides:

```text
Data access structure
```

Statistics provide:

```text
Distribution information
```

An index can have associated statistics, but SQL Server can also maintain statistics that aren't simply the same thing as an index structure.

---

# 57. Auto-created statistics

SQL Server can automatically create statistics in appropriate situations.

This is useful because developers don't need to manually predict every column that the optimizer may need statistical information about.

But automatic statistics don't eliminate the need for understanding statistics behavior.

---

# 58. Statistics names

You may encounter names resembling:

```text
_WA_Sys_...
```

These commonly indicate automatically created statistics.

Don't interpret the name as meaning:

> "This statistic is useless."

Its existence often means SQL Server determined it could be useful for optimization.

---

# 59. Filtered statistics

Filtered statistics can describe a subset of a table.

Conceptually:

```text
Entire table
████████████████████

Filtered subset
████
```

This can be valuable when a particular subset has a very different distribution.

---

# 60. Example of filtered distribution

Suppose:

```text
Status = Active
```

represents 95% of the table.

But:

```text
Status = Deleted
```

represents only 0.1%.

The distribution is highly uneven.

A filtered strategy can sometimes provide more targeted information for a particular workload.

---

# 61. Histograms and inequality predicates

Equality:

```sql
WHERE CustomerId = 100
```

and range:

```sql
WHERE CustomerId >= 100
```

require different estimation reasoning.

Range predicates depend on the distribution across values, making histogram information especially important.

---

# 62. Range query example

Suppose:

```text
OrderDate
```

contains ten years of data.

Query:

```sql
WHERE OrderDate >= '2026-01-01'
```

The optimizer needs to estimate how many rows fall in that date range.

A useful histogram can make this estimate much more accurate.

---

# 63. Date columns are often tricky

Date distributions frequently have patterns such as:

```text
Older years → many rows
Recent days → rapidly growing rows
Future dates → none
```

This means the distribution may not be uniform.

Assuming:

```text
Every date has the same number of rows
```

can be badly wrong.

---

# 64. Data distribution is everything

Consider:

```text
1 billion rows
```

If values are evenly distributed:

```text
Average ≈ Reality
```

If values are heavily skewed:

```text
Average ≠ Reality
```

This is why statistics are fundamentally about **distribution**, not merely row counts.

---

# 65. Cardinality estimation and indexes

An index doesn't guarantee a good estimate.

You can have:

```text
Excellent index
+
Bad statistics
=
Bad plan
```

And conversely:

```text
No ideal index
+
Excellent estimates
=
Optimizer may still choose the best available plan
```

Both access structures and statistical information matter.

---

# 66. Expert troubleshooting scenario

Suppose a query suddenly changes from:

```text
1 second
```

to:

```text
45 seconds
```

You compare plans.

Old plan:

```text
Nested Loops
```

New plan:

```text
Hash Match
```

Don't immediately say:

> "Hash Match is bad."

Instead compare:

```text
Estimated rows
Actual rows
Statistics
Parameters
Data volume
Memory grants
```

---

# 67. What if the new plan is actually correct?

Suppose the data changed from:

```text
100,000 rows
```

to:

```text
100,000,000 rows
```

A Hash Match may now be exactly what the optimizer should choose.

The query became slower because the workload became larger—not necessarily because SQL Server chose the wrong plan.

---

# 68. Performance is relative to work

Always ask:

```text
How much work is the query actually doing?
```

A query processing:

```text
1 billion rows in 20 seconds
```

may be more efficient than one processing:

```text
100,000 rows in 5 seconds
```

depending on the workload.

Raw duration alone doesn't identify the root cause.

---

# 69. Logical reads + estimates

A useful combination is:

```text
Estimated rows
+
Actual rows
+
Logical reads
+
CPU time
+
Elapsed time
```

Together they tell a much stronger story than any single metric.

---

# 70. The cardinality-estimation debugging workflow

Use this sequence:

```text
1. Capture actual execution plan
        ↓
2. Find major estimate/actual mismatches
        ↓
3. Locate first major divergence
        ↓
4. Inspect predicate/statistics
        ↓
5. Check data distribution
        ↓
6. Check parameter values
        ↓
7. Check joins
        ↓
8. Check memory grants/spills
        ↓
9. Test a hypothesis
        ↓
10. Measure again
```

---

# 71. What NOT to do

Avoid this workflow:

```text
Query slow
   ↓
Rebuild all indexes
   ↓
Update all statistics
   ↓
Add five indexes
   ↓
Change MAXDOP
   ↓
Hope
```

That's not tuning.

That's changing multiple variables without understanding causality.

---

# 72. Better approach

Instead:

```text
Observe
 ↓
Hypothesize
 ↓
Change ONE meaningful variable
 ↓
Measure
 ↓
Compare
 ↓
Keep or revert
```

This is how you build confidence in a tuning decision.

---

# 73. Interview question

### "What is cardinality estimation?"

Strong answer:

> "Cardinality estimation is the optimizer's prediction of how many rows an operator or predicate will produce. SQL Server uses statistics, data distribution, predicates, and other information to estimate row counts, and those estimates influence join algorithms, access paths, memory grants, and other plan decisions."

---

# 74. Interview question

### "Why can stale statistics cause a bad execution plan?"

Strong answer:

> "If statistics no longer represent the current data distribution, the optimizer may estimate the selectivity of predicates incorrectly. Those incorrect estimates can lead to inappropriate access paths, join algorithms, memory grants, or join ordering."

---

# 75. Interview question

### "What is data skew?"

Strong answer:

> "Data skew means values are distributed unevenly. A small number of values may account for a very large portion of the rows, while many other values are rare. Skew can make average or independence-based estimates inaccurate and can produce parameter-sensitive plans."

---

# 76. Interview question

### "Why isn't updating statistics always enough?"

Strong answer:

> "Because estimation errors can come from more than stale statistics. They can result from skew, correlated predicates, expressions, parameter sensitivity, implicit conversions, complex joins, or limitations in the available statistical information."

---

# 77. Expert mental model

Think of SQL Server's optimizer as asking:

```text
How many rows?
      ↓
How expensive to access?
      ↓
How many rows after filtering?
      ↓
How expensive to join?
      ↓
How much memory?
      ↓
Serial or parallel?
      ↓
Which complete plan has the lowest estimated cost?
```

The entire decision tree depends heavily on the first question:

> **How many rows?**

---

# 78. Lesson 138 takeaway

The central chain is:

```text
DATA DISTRIBUTION
       ↓
STATISTICS
       ↓
CARDINALITY ESTIMATION
       ↓
COST ESTIMATION
       ↓
EXECUTION PLAN
       ↓
ACTUAL PERFORMANCE
```

If the estimate is dramatically wrong:

```text
Bad estimate
   ↓
Wrong assumptions
   ↓
Potentially wrong plan
```

### The expert rule

> **When an execution plan looks wrong, find the first major cardinality-estimation error before blaming the operator that happens to be expensive.**

That habit alone will dramatically improve your SQL Server troubleshooting.

**Next: Lesson 139 — Statistics Deep Dive: Auto-Update, Sampling, FULLSCAN, Incremental Statistics, Filtered Statistics & Advanced Maintenance Strategies.**
