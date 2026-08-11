# Lesson 62 — Query Store + Execution Plans: Finding a 45-Second Query

This is a classic **senior SQL DBA production incident**.

### Incident

A business application reports:

> "The Order Search page suddenly takes 45 seconds."

Monitoring shows:

```text
CPU              35%
Storage          Normal
Connections      Normal
Blocking         None
MI health        Normal
```

Before the incident:

```text
Query duration ≈ 300 ms
```

Now:

```text
Query duration ≈ 45 sec
```

This is exactly where **Query Store + execution plans** become extremely useful.

---

# 1. First question: What changed?

Don't start by changing the database.

Ask:

```text
When did it become slow?
```

Timeline:

```text
09:00
Query = 300 ms

10:00
Query = 320 ms

10:15
Application deployment

10:17
Query = 45 sec
```

That correlation is valuable.

But remember:

> **Correlation is a clue, not proof.**

---

# 2. Check Query Store

Query Store lets you investigate query performance historically.

Conceptually:

```text
Query Store
     |
     +-- Query
     |
     +-- Plans
     |
     +-- Runtime statistics
     |
     +-- Performance over time
```

You want to compare:

```text
Old plan
vs
New plan
```

---

# 3. Imagine Query Store shows this

```text
Query ID: 742

Plan 101
Average duration: 310 ms
Executions: 80,000

Plan 204
Average duration: 44.8 sec
Executions: 1,200
```

This is a huge clue.

The query isn't necessarily inherently bad.

It may be:

> **Using a different execution plan.**

---

# 4. Compare the plans

### Plan 101

```text
Index Seek
     |
     v
Nested Loops
     |
     v
Small number of rows
```

Runtime:

```text
~300 ms
```

### Plan 204

```text
Index Scan
     |
     v
Hash Join
     |
     v
Millions of rows
```

Runtime:

```text
~45 sec
```

Now ask:

> **Why did the optimizer choose Plan 204?**

That's the real investigation.

---

# 5. Look at estimated vs actual rows

Suppose Plan 204 shows:

```text
Estimated rows: 50
Actual rows:    3,500,000
```

That's a massive estimation error.

The optimizer thought:

```text
50 rows
```

but reality was:

```text
3.5 million
```

That can completely change the optimal execution strategy.

---

# 6. Why estimates matter

Imagine the optimizer estimates:

```text
10 rows
```

A nested-loop strategy might be reasonable.

But if the actual result is:

```text
5 million rows
```

then repeatedly looking up rows can become extremely expensive.

Conversely, if the optimizer expects millions of rows when only a handful actually qualify, it may choose a large scan/hash strategy that isn't appropriate for that execution.

---

# 7. Investigate statistics

One possible cause:

```text
Statistics
      ↓
Stale or inaccurate distribution information
      ↓
Bad cardinality estimate
      ↓
Poor plan
```

You investigate:

```text
When were statistics last updated?
How much did the table change?
Did data distribution change?
Is the affected column highly skewed?
```

---

# 8. Data growth can trigger a problem

Suppose last month:

```text
Orders = 10 million rows
```

Today:

```text
Orders = 500 million rows
```

The query hasn't changed.

But the workload characteristics have.

A plan that worked well at:

```text
10 million rows
```

may no longer be optimal at:

```text
500 million rows
```

This is why performance tuning is not a one-time activity.

---

# 9. Parameter sensitivity

Now suppose the query is:

```sql id="7w0jz2"
SELECT
    OrderID,
    CustomerID,
    OrderDate,
    TotalAmount
FROM Sales.Orders
WHERE CustomerID = @CustomerID;
```

Customer distribution:

```text id="q0p7km"
Customer A → 20 rows

Customer B → 4,000,000 rows
```

That's highly skewed.

A plan optimized for Customer A might be terrible for Customer B.

---

# 10. A realistic sequence

First execution:

```text id="q5v6s3"
@CustomerID = 100
```

Returns:

```text
15 rows
```

Optimizer chooses:

```text
Index Seek
```

Excellent.

Later:

```text id="7d3n8h"
@CustomerID = 500000
```

Returns:

```text
4 million rows
```

But the previously selected strategy may no longer be ideal.

You can get dramatic runtime differences between parameter values.

---

# 11. Modern SQL Server considerations

Don't oversimplify parameter sensitivity as merely:

> "Parameter sniffing is bad."

The optimizer and modern SQL Server/Azure SQL capabilities include mechanisms intended to handle parameter-sensitive workloads more intelligently.

Your senior-level answer should be:

> **"I would investigate parameter sensitivity and the plans selected for different parameter values rather than assuming every plan regression is caused by parameter sniffing."**

That's a much better answer.

---

# 12. Check indexes

Suppose the query filters by:

```text id="y9q2mf"
CustomerID
```

You inspect indexes and discover:

```text id="0x7r4c"
IX_Orders_CustomerID
```

exists.

So why is the query scanning?

Possible reasons:

```text id="4i1k6s"
Poor selectivity
Statistics
Residual predicates
Implicit conversion
Function on indexed column
Plan choice
Parameter sensitivity
Covering/index design
```

The existence of an index does **not** guarantee an Index Seek.

---

# 13. Implicit conversion

Here's a classic problem.

Column:

```sql id="r2f7km"
CustomerID VARCHAR(20)
```

Application sends:

```text id="3u6q9b"
NVARCHAR
```

SQL Server may need to perform a conversion.

Depending on the expression and data types, this can interfere with efficient index access.

So always inspect:

```text id="1g8m3c"
Data types
Parameters
Predicates
Conversions
```

when a supposedly indexed query unexpectedly scans.

---

# 14. Functions on columns

Another classic:

```sql id="2h5w9n"
WHERE YEAR(OrderDate) = 2026
```

This can make it harder to use an index efficiently on `OrderDate` than a range predicate such as:

```sql id="9k3p4x"
WHERE OrderDate >= '20260101'
  AND OrderDate <  '20270101'
```

The general lesson:

> **Make predicates searchable/sargable where possible.**

---

# 15. Query Store remediation

Suppose you establish:

```text id="e6f3x2"
Plan 101 = good
Plan 204 = bad
```

One possible mitigation is to use Query Store's plan management capabilities to force the known-good plan, **when appropriate**.

But don't treat plan forcing as the permanent solution automatically.

You still need to understand:

```text id="v4m6w2"
Why did the bad plan appear?
Will the old plan remain good?
Did the data change?
Is the query changing?
```

Think:

```text id="6d0w1q"
Plan forcing
    ↓
Possible immediate stabilization
    ↓
Root-cause investigation
    ↓
Permanent solution
```

---

# 16. When should you NOT force a plan?

Suppose:

```text id="x7k3m5"
Old plan = good
```

but the data volume has fundamentally changed.

Forcing the old plan forever may simply postpone the problem.

Likewise, if the query has multiple very different parameter patterns, one plan may not be ideal for all executions.

So:

> **Plan forcing is a tool, not a substitute for understanding the workload.**

---

# 17. The production fix

Suppose your investigation finds:

```text id="f0h4x7"
Statistics became inaccurate
+
Data distribution changed
+
New plan was selected
```

You might:

```text id="z1n5c8"
1. Correct statistics issue
2. Re-evaluate execution plan
3. Validate runtime
4. Monitor Query Store
5. Remove temporary mitigation if appropriate
```

Or you might discover:

```text id="b3x8n2"
Query design problem
```

Then rewrite the query.

Or:

```text id="m8k4v2"
Index design problem
```

Then evaluate an appropriate index.

The fix depends on evidence.

---

# 18. Always validate after the fix

Suppose before:

```text id="w2q6m9"
45 sec
```

After:

```text id="k7d3r1"
280 ms
```

Excellent.

But don't stop there.

Check:

```text id="q6s8n4"
CPU
Logical reads
Physical reads
Duration
Execution count
Concurrency
Blocking
Plan stability
```

A query that went from:

```text
45 sec → 300 ms
```

but now consumes:

```text
10× CPU
```

may still create another production problem under concurrency.

---

# 19. Complete investigation

Your incident now looks like:

```text id="5c4r8n"
User complaint
      ↓
45-second query
      ↓
MI health normal
      ↓
CPU normal
      ↓
No blocking
      ↓
Query Store
      ↓
Plan changed
      ↓
Old plan = 300 ms
New plan = 45 sec
      ↓
Execution-plan comparison
      ↓
Cardinality estimate problem
      ↓
Statistics/data distribution investigation
      ↓
Parameter sensitivity investigated
      ↓
Root cause identified
      ↓
Targeted remediation
      ↓
Validation
      ↓
Monitoring
```

That's a **senior DBA troubleshooting story**.

---

# 20. Interview question

### Interviewer:

> "A query was running in 300 ms yesterday and 45 seconds today. CPU isn't high and there is no blocking. What do you check?"

A strong answer:

> "I'd first establish when the regression started and use Query Store to compare historical runtime statistics and execution plans. If the plan changed, I'd compare the good and bad plans, especially cardinality estimates, access methods, joins, and memory grants. I'd then investigate statistics, data-distribution changes, parameter sensitivity, indexes, implicit conversions, and recent application or schema changes. If appropriate, I could temporarily stabilize the workload using Query Store plan management, but I'd still investigate the root cause. Finally I'd validate the fix using duration, reads, CPU, concurrency, and plan stability."

That's the level expected from a strong senior SQL DBA.

---
