# Lesson 131 — Execution Plan Internals: Operators, Costs, Memory Grants & Spills — Expert Level

Today we're going inside the execution plan itself.

The goal is not merely to recognize operators like **Seek**, **Scan**, or **Hash Match**.

The expert skill is understanding:

> **Why SQL Server chose those operators, what resources they consume, and where the real bottleneck is.**

---

## 1. The execution plan is the optimizer's strategy

Think of an execution plan as:

```text
Query
 ↓
Optimizer
 ↓
Chosen strategy
 ↓
Operators
 ↓
Data movement
 ↓
CPU / I/O / Memory / Parallelism
```

The plan tells you **how SQL Server intends to execute the query**.

---

# 2. Estimated vs actual execution plans

There are two major perspectives.

### Estimated plan

Shows what SQL Server expects to happen.

### Actual plan

Shows what happened during execution, including runtime information.

For performance troubleshooting, actual execution plans are often extremely valuable because you can compare:

```text
Estimated rows
vs
Actual rows
```

---

# 3. Execution-plan operators

Common operators include:

```text
Index Seek
Index Scan
Table Scan
Key Lookup
Nested Loops
Hash Match
Merge Join
Sort
Filter
Stream Aggregate
Hash Aggregate
Compute Scalar
Concatenation
Parallelism
```

Each operator represents a particular operation.

---

# 4. Seek vs scan

A common misconception:

> "Seek = good, Scan = bad."

Not necessarily.

If you need:

```text
10 rows
out of
100 million
```

a seek can be excellent.

If you need:

```text
90 million rows
out of
100 million
```

a scan may be the more efficient strategy.

The question is:

> **How much data actually needs to be processed?**

---

# 5. Index Seek

A seek navigates an index to locate a relevant range or set of keys.

Conceptually:

```text
Index
 ↓
Navigate to target
 ↓
Read relevant range
```

This can dramatically reduce I/O for selective predicates.

---

# 6. Index Scan

A scan reads a substantial portion or all of an index.

That isn't automatically inefficient.

For a large analytical query, scanning a narrow covering index may be cheaper than repeatedly looking up rows in a wider table.

---

# 7. Table Scan

A table scan reads the table's underlying storage structure.

It can be perfectly reasonable when:

```text
Most rows are required
```

or when no useful selective access path exists.

---

# 8. Key Lookup

A Key Lookup usually appears when SQL Server uses a nonclustered index to find qualifying rows but needs additional columns from the base table.

Conceptually:

```text
Nonclustered Index
       ↓
Find row
       ↓
Key Lookup
       ↓
Base table
```

---

# 9. Why Key Lookup can become expensive

Suppose:

```text
Index returns = 5 rows
```

Five lookups may be trivial.

But:

```text
Index returns = 5 million rows
```

Five million lookups can generate significant random I/O and CPU work.

This is why **estimated/actual row counts matter**.

---

# 10. Nested Loops

Nested Loops conceptually does:

```text
For each row from outer input:
    find matching rows in inner input
```

Example:

```text
Outer = 10 rows
Inner lookup = indexed
```

Excellent scenario.

But:

```text
Outer = 10 million rows
```

can make repeated inner operations extremely expensive.

---

# 11. Nested Loops mental model

Think:

```text
Outer rows
   ×
Inner work per row
   =
Total work
```

If the outer input is tiny, this can be wonderful.

If the outer input is unexpectedly huge, performance can collapse.

---

# 12. Hash Match

Hash joins typically build a hash structure from one input and probe it with another.

Conceptually:

```text
Input A
 ↓
Build hash table

Input B
 ↓
Probe hash table
```

Hash-based operations are often useful for larger, less-selective inputs.

---

# 13. Hash Match isn't automatically bad

You may see:

```text
Hash Match
```

and think:

> "I need to eliminate this."

No.

A hash join may be exactly the right strategy for a large analytical join.

The real question is:

> **Is the hash operation appropriate for the cardinalities and workload?**

---

# 14. Hash spills

Hash operations may require memory.

If insufficient memory is available, portions of the operation can spill to TempDB.

Conceptually:

```text
Hash operation
      ↓
Needs memory
      ↓
Insufficient grant
      ↓
Spill
      ↓
TempDB I/O
```

Spills can significantly hurt performance.

---

# 15. Merge Join

Merge Join works efficiently when both inputs are appropriately ordered.

Conceptually:

```text
Input A → sorted
Input B → sorted
          ↓
      Merge Join
```

It can be highly efficient for suitable workloads.

---

# 16. Why ordering matters

If inputs are already ordered appropriately, Merge Join can avoid expensive sorting.

But if SQL Server must first sort huge inputs:

```text
Sort
 ↓
Sort
 ↓
Merge Join
```

the overall strategy may become more expensive.

---

# 17. Sort operator

Sorting can consume substantial:

```text
CPU
Memory
TempDB
```

especially for large datasets.

A query like:

```sql id="4e5e0e"
ORDER BY OrderDate;
```

may require a sort unless the access path already provides the needed order.

---

# 18. Indexes can eliminate sorts

Suppose:

```sql id="r1j9v6"
WHERE CustomerId = @CustomerId
ORDER BY OrderDate;
```

An index such as:

```text id="k8x3m2"
(CustomerId, OrderDate)
```

may provide rows in a useful order.

This can reduce or eliminate an explicit Sort.

---

# 19. Stream Aggregate

A Stream Aggregate generally works on appropriately ordered input.

Conceptually:

```text
Sorted input
     ↓
Stream Aggregate
     ↓
Groups/results
```

It can be efficient because it doesn't necessarily need the same type of hash structure used by Hash Aggregate.

---

# 20. Hash Aggregate

Hash Aggregate builds a structure for grouping.

Conceptually:

```text
Rows
 ↓
Hash by grouping key
 ↓
Aggregate
```

It can be effective for large, unordered inputs but may require substantial memory.

---

# 21. Aggregate spills

If a hash aggregate doesn't have enough memory:

```text id="8z8w1q"
Hash Aggregate
      ↓
Memory insufficient
      ↓
TempDB spill
```

Again, cardinality and memory estimates matter.

---

# 22. Compute Scalar

Compute Scalar performs expressions such as:

```sql id="9g7q1m"
Price * Quantity
```

or:

```sql id="3r2k8p"
CASE WHEN Status = 'A' THEN 1 ELSE 0 END
```

It isn't inherently expensive.

But complex expressions repeated over millions of rows can matter.

---

# 23. Filter

A Filter applies a predicate after rows reach that point.

Example:

```text id="x4v2q7"
Input
 ↓
Filter
 ↓
Qualifying rows
```

An important question is:

> **Could the filtering have happened earlier through a better access path?**

---

# 24. Predicate pushdown

If SQL Server can apply filtering closer to the data source, it may reduce downstream work.

Conceptually:

```text
Bad:
Read many rows
 ↓
Filter
 ↓
Process many rows

Better:
Filter early
 ↓
Process fewer rows
```

This is one reason SARGable predicates and good indexes matter.

---

# 25. Residual predicates

A seek can locate a broad range and then apply an additional predicate to rows within that range.

Conceptually:

```text
Seek predicate
 ↓
Candidate rows
 ↓
Residual predicate
 ↓
Final rows
```

A seek doesn't necessarily mean every examined row qualifies.

---

# 26. Seek predicate vs residual predicate

This distinction can be useful when diagnosing unexpectedly high reads.

You may see:

```text
Seek
Estimated output = 100 rows
```

but the storage engine may examine many more candidate rows before applying residual filtering.

---

# 27. Predicate selectivity again

Suppose:

```text id="x9w4c7"
Candidate rows = 10 million
Final rows = 10
```

Even though the final output is tiny, the query may still do substantial work.

The ideal index may allow SQL Server to narrow the candidate set earlier.

---

# 28. Operator cost percentages

Execution plans often show relative estimated operator costs.

For example:

```text
Index Scan     70%
Hash Match     20%
Sort           10%
```

These percentages are useful clues.

But:

> **They are not actual runtime percentages.**

Don't blindly optimize the operator with the largest displayed percentage.

---

# 29. Actual elapsed time matters more

You ultimately care about:

```text id="7j0m2n"
Duration
CPU
Logical reads
Physical reads
Memory
Waits
Concurrency
```

The estimated plan is a prediction.

Production behavior is the reality.

---

# 30. Memory grants

Some operators request memory during query execution.

Typical consumers include:

```text id="c5n8r1"
Sort
Hash Join
Hash Aggregate
```

SQL Server estimates how much memory is needed.

That estimate depends heavily on cardinality.

---

# 31. Memory-grant pipeline

Think:

```text id="2m7x4q"
Cardinality estimate
       ↓
Estimated row size
       ↓
Memory requirement
       ↓
Memory grant
       ↓
Sort / Hash / Aggregate
```

An error early in the chain can create a bad grant.

---

# 32. Under-grant

Suppose:

```text id="6k1p9m"
Estimated rows = 1,000
Actual rows = 5,000,000
```

The optimizer may request insufficient memory.

Potential result:

```text id="8q3v7c"
Spill to TempDB
```

---

# 33. Over-grant

Now reverse it:

```text id="4n6m2x"
Estimated rows = 50,000,000
Actual rows = 100
```

The query may reserve far more memory than needed.

Under concurrency:

```text id="j7r1p8"
Query A → huge grant
Query B → huge grant
Query C → huge grant
...
```

This can contribute to memory pressure.

---

# 34. Row width matters

Memory isn't determined only by row count.

Suppose:

```text id="u3m8q5"
Rows = 1,000,000
```

with:

```text id="x7c2p1"
Row width = 50 bytes
```

versus:

```text id="v9n4k6"
Row width = 2,000 bytes
```

The memory requirements can be dramatically different.

---

# 35. Memory Grant Feedback

Modern SQL Server versions can use runtime information to improve future memory-grant decisions in supported scenarios.

Conceptually:

```text
Initial execution
      ↓
Grant too small/large
      ↓
Runtime feedback
      ↓
Adjust future grant
```

This is an example of SQL Server adapting based on observed execution behavior.

---

# 36. Spills aren't always catastrophic

A spill deserves investigation.

But don't assume:

> "Any spill = emergency."

Consider:

```text
Small spill
+
Rare query
+
Minimal latency impact
```

versus:

```text
Huge spill
+
High-frequency query
+
Heavy TempDB I/O
+
Production latency
```

Context matters.

---

# 37. Parallelism

Parallel plans divide work among multiple threads.

Conceptually:

```text
Input
  ↓
Distribute
 ↙ ↓ ↘
T1 T2 T3
 ↘ ↓ ↙
 Gather
```

Parallelism can reduce elapsed time for sufficiently expensive workloads.

---

# 38. Parallelism isn't automatically good

Parallel execution has overhead:

```text
Thread coordination
Data exchange
Synchronization
CPU consumption
```

A tiny query may be slower when parallelized.

---

# 39. Parallelism operators

You'll often encounter exchange operators such as:

```text id="t2y6m8"
Distribute Streams
Repartition Streams
Gather Streams
```

These move data between parallel workers.

---

# 40. Repartition Streams

Rows may need to be redistributed based on a key.

For example:

```text id="k8n3q5"
Worker 1 → many keys
Worker 2 → many keys
Worker 3 → many keys
```

SQL Server may repartition rows so related values reach the same worker.

---

# 41. Parallelism imbalance

Suppose:

```text id="w5m1r9"
Worker 1 → 100 rows
Worker 2 → 100 rows
Worker 3 → 100 rows
Worker 4 → 10 million rows
```

The query may still be constrained by the heavily loaded worker.

This is sometimes called **parallelism skew**.

---

# 42. Data skew can become parallelism skew

The same underlying data distribution problem can affect:

```text id="c9v4p7"
Cardinality estimation
+
Parallel distribution
```

If one key dominates the data, repartitioning can produce uneven workloads.

---

# 43. Serial vs parallel plans

Don't ask:

> "Why isn't this query parallel?"

Ask:

> "Would parallel execution actually improve this workload?"

A query processing only 100 rows probably doesn't need it.

A massive analytical scan may.

---

# 44. Row mode vs batch mode

SQL Server can execute workloads using different processing modes.

### Row mode

Processes rows individually.

### Batch mode

Processes groups/batches of rows more efficiently for suitable analytical workloads.

Columnstore is strongly associated with batch-mode processing, though modern SQL Server versions can support batch mode in additional scenarios.

---

# 45. Why batch mode can be powerful

Instead of:

```text id="7q4n2x"
Row
Row
Row
Row
Row
...
```

think:

```text id="p8m3v1"
Batch
 ↓
Vectorized processing
 ↓
More efficient CPU utilization
```

This can significantly improve analytical workloads.

---

# 46. Plan shape matters

Consider:

```text id="a1m8q6"
Seek
 ↓
Nested Loops
 ↓
Lookup
```

versus:

```text id="n4x7c2"
Scan
 ↓
Hash Join
 ↓
Aggregate
```

Don't judge by individual operators.

Analyze the **entire plan shape**.

---

# 47. Find the first major divergence

A powerful troubleshooting technique is:

> Find where the estimated and actual row counts first diverge significantly.

Example:

```text id="s2k9v5"
Operator 1
Estimated 1,000
Actual 1,050  ← good

Operator 2
Estimated 900
Actual 950    ← good

Operator 3
Estimated 800
Actual 8,000,000 ← problem
```

Operator 3 is a strong starting point.

---

# 48. Why the first bad estimate matters

Once an estimate becomes wildly wrong:

```text id="h6p3r8"
Bad estimate
 ↓
Bad downstream assumptions
 ↓
Bad join choice
 ↓
Bad memory grant
 ↓
Potential spill
```

Fixing downstream symptoms without understanding the original estimate can lead to ineffective tuning.

---

# 49. Execution plan troubleshooting hierarchy

Use this:

```text id="m7x2q9"
1. What is the query actually doing?
2. How many rows are estimated?
3. How many actually occur?
4. Where does divergence begin?
5. What access path was selected?
6. What joins were selected?
7. Are memory grants appropriate?
8. Are there spills?
9. Is parallelism appropriate?
10. What does runtime telemetry show?
```

---

# 50. Example: slow query

Suppose:

```text id="v5q1n8"
Duration = 30 seconds
CPU = 4 seconds
```

That tells you:

```text id="k3m9r6"
A lot of elapsed time
is not CPU.
```

You should investigate waits, I/O, blocking, memory, or other external factors rather than blindly optimizing CPU.

---

# 51. Example: CPU-heavy query

Suppose:

```text id="c8p4x2"
Duration = 10 seconds
CPU = 9.8 seconds
```

This points more strongly toward:

```text id="j1n7m5"
Excessive CPU work
```

Potential causes include:

```text
Large scans
Expensive expressions
Poor joins
Repeated lookups
Sorting
Aggregation
Parallelism overhead
```

---

# 52. Logical reads

Logical reads indicate pages read from the buffer pool.

High logical reads often mean:

```text id="r9x3k7"
A lot of data was touched.
```

This is frequently more useful for diagnosing query I/O efficiency than simply looking at elapsed time.

---

# 53. Physical reads

Physical reads involve fetching pages from storage.

They can be influenced by:

```text id="w2m6p8"
Buffer-pool state
Storage performance
Working-set size
Concurrent workload
```

Therefore:

```text
Logical reads ≠ physical reads
```

---

# 54. The execution-plan + runtime model

Expert tuning combines:

```text id="y4c8n1"
Execution Plan
      +
Actual Rows
      +
Logical Reads
      +
CPU
      +
Duration
      +
Waits
      +
Memory
```

No single metric tells the entire story.

---

# 55. Expert scenario

Plan:

```text id="x6p2m9"
Index Seek
 ↓
Nested Loops
 ↓
Key Lookup
```

Estimated:

```text id="n8v4c1"
100 rows
```

Actual:

```text id="r3m7x5"
8 million rows
```

Runtime:

```text id="q1k9p6"
45 seconds
```

Diagnosis:

```text id="w5c2m8"
The operators themselves aren't inherently wrong.
The cardinality estimate is wrong,
causing an inappropriate plan shape.
```

---

# 56. Expert scenario #2

Plan:

```text id="s7x4n1"
Scan
 ↓
Hash Match
 ↓
Hash Aggregate
```

Runtime:

```text id="m3p8q6"
200 ms
```

Someone says:

> "There is a scan and hash join. Optimize it."

Wrong conclusion.

If:

```text id="k9c1v5"
200 ms
+
low CPU
+
low reads
+
no spills
```

the plan may already be excellent.

---

# 57. Expert scenario #3

Plan:

```text id="a8n2r7"
Sort
 ↓
Hash Match
```

Memory grant:

```text id="f4m9x3"
2 GB
```

Actual usage:

```text id="p6c1v8"
50 MB
```

Under high concurrency, this could become a serious problem even if the query itself is fast.

The issue isn't necessarily elapsed time.

It may be **resource consumption under concurrency**.

---

# 58. The biggest execution-plan mistake

Don't ask:

> "Which operator is bad?"

Ask:

> **"Which operator is doing more work than it should, and why?"**

That shift in thinking is crucial.

---

# 59. Expert mental model

```text id="z3m7q2"
Operator
   ↓
Input rows
   ↓
Row width
   ↓
Estimated cost
   ↓
Memory requirement
   ↓
Actual work
   ↓
CPU / I/O / waits
```

Every operator is part of a larger resource pipeline.

---

# 60. Lesson 131 takeaway

Remember these five principles:

### 1.

**Seek doesn't automatically mean good.**

### 2.

**Scan doesn't automatically mean bad.**

### 3.

**Estimated cost isn't actual runtime.**

### 4.

**Cardinality errors can cause downstream plan problems.**

### 5.

**The real bottleneck is the operator doing unnecessary work, not simply the operator with the scariest name.**

Your execution-plan workflow:

```text id="b4x8m1"
PLAN
 ↓
ROWS
 ↓
ESTIMATE vs ACTUAL
 ↓
FIRST MAJOR DIVERGENCE
 ↓
OPERATOR WORK
 ↓
MEMORY / SPILLS
 ↓
PARALLELISM
 ↓
CPU / I/O / WAITS
 ↓
ROOT CAUSE
```

**Next: Lesson 132 — Wait Statistics, Blocking, Latches & Locks — Expert Level.**
