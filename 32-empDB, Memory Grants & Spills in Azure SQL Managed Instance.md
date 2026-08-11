# Lesson 64 — TempDB, Memory Grants & Spills in Azure SQL Managed Instance

This is an advanced performance topic, and it's especially useful when you see a strange situation like:

> **"CPU is only 40%, there is little blocking, but one query takes 2 minutes."**

A common clue is an execution-plan warning around **sorts or hash operations**.

---

## 1. What is TempDB?

`tempdb` is a system database used for temporary/workspace operations.

SQL Server can use it for things such as:

```text
Temporary tables
Table variables
Sorting
Hash operations
Row-versioning
Intermediate query results
Worktables
Workfiles
```

Think of it as a shared workspace.

```text
              SQL MI
                |
        +-------+-------+
        |       |       |
      Query A Query B Query C
        |       |       |
        +-------+-------+
                |
              tempdb
```

---

# 2. Real production scenario

At 11:30 AM:

```text
Report query
   ↓
Normally: 5 seconds
Today:    2 minutes
```

You check:

```text
CPU = 40%
Blocking = minimal
Connections = normal
```

So why is the query slow?

You inspect the execution plan and see:

```text
Sort
 ⚠ Warning
```

or:

```text
Hash Match
 ⚠ Spill to tempdb
```

Now you've found a major clue.

---

# 3. What is a spill?

Suppose SQL Server needs memory for a sort.

It estimates:

```text
Required memory ≈ 100 MB
```

and receives enough memory.

Everything stays in memory:

```text
Data
 ↓
Sort
 ↓
Memory
 ↓
Result
```

Fast.

But suppose it needs:

```text
2 GB
```

and doesn't have enough workspace memory available.

It may spill intermediate data to `tempdb`.

```text
Data
 ↓
Sort
 ↓
Memory insufficient
 ↓
tempdb
 ↓
Read/write intermediate data
 ↓
Result
```

Disk/tempdb activity makes the operation much more expensive.

---

# 4. Why did the query need so much memory?

Common reasons include:

```text
Large sort
Large hash join
Poor cardinality estimate
Huge intermediate result set
Missing/ineffective index
Bad execution plan
```

This is why memory-grant problems often lead you back to **query optimization**.

---

# 5. Memory grants

Certain operators request workspace memory before execution.

Examples:

```text
Sort
Hash Match
```

Conceptually:

```text
Query
  |
  v
Optimizer estimates rows
  |
  v
Memory grant requested
  |
  v
Execution
```

If the estimate is wrong, the grant can also be wrong.

---

# 6. Under-grant

Suppose:

```text
Estimated rows = 10,000
Actual rows    = 10,000,000
```

SQL Server may request memory based on the smaller estimate.

Result:

```text
Not enough memory
      ↓
Spill
      ↓
tempdb
      ↓
Slow query
```

So again:

> **Cardinality estimation is extremely important.**

---

# 7. Over-grant

The opposite can happen.

Suppose SQL Server estimates:

```text
100 million rows
```

but actual result:

```text
1,000 rows
```

It may request a large memory grant unnecessarily.

Now imagine:

```text
100 concurrent queries
```

Each requesting a large amount of memory.

You can create memory pressure even though individual queries aren't actually processing much data.

---

# 8. Under-grant vs over-grant

Memorize this:

| Problem                 | Consequence                          |
| ----------------------- | ------------------------------------ |
| Too little memory grant | Spill to tempdb                      |
| Too much memory grant   | Memory pressure / concurrency impact |

So:

```text
Bad estimate
    ↓
Bad memory grant
    ↓
Performance problem
```

---

# 9. Example: large sort

Query:

```sql id="k7x2m1"
SELECT
    CustomerID,
    OrderDate,
    TotalAmount
FROM Sales.Orders
ORDER BY OrderDate DESC;
```

Suppose:

```text id="v4n8c6"
Orders = 500 million rows
```

Sorting all those rows can require substantial workspace.

If the application only needs:

```text id="x3m6q9"
TOP (100)
```

you should also ask whether the query and indexing strategy allow SQL Server to avoid sorting a huge dataset.

For example:

```sql id="q9p2s7"
SELECT TOP (100)
    CustomerID,
    OrderDate,
    TotalAmount
FROM Sales.Orders
ORDER BY OrderDate DESC;
```

An appropriate index may allow a much more efficient access pattern.

---

# 10. Indexes can reduce sorting

Suppose you frequently execute:

```sql id="r5m8c3"
ORDER BY OrderDate DESC
```

and have an appropriate index beginning with `OrderDate`.

SQL Server may be able to retrieve rows in useful order instead of performing a large explicit sort.

Conceptually:

### Without useful ordering

```text id="a6k1w4"
Table
 ↓
Read millions of rows
 ↓
Sort
 ↓
Return results
```

### With useful index

```text id="j8n3v5"
Index ordered by OrderDate
 ↓
Read required rows
 ↓
Return
```

Potentially much less work.

---

# 11. Hash join spills

Now consider:

```sql id="c5q7m9"
SELECT ...
FROM Sales.Orders o
JOIN Sales.Customers c
    ON o.CustomerID = c.CustomerID;
```

A hash join may require substantial workspace memory.

If memory is insufficient:

```text id="b4r8x2"
Hash operation
      ↓
Memory insufficient
      ↓
Spill
      ↓
tempdb
```

The execution plan may show a warning.

---

# 12. Don't automatically blame TempDB

This is critical.

If you see:

```text id="p1s6v8"
tempdb activity = high
```

don't immediately conclude:

> "TempDB is too small."

Ask:

> **"Why is the workload using so much tempdb?"**

Potential root causes:

```text id="d8k3w7"
Sort spill
Hash spill
Large temporary tables
Row-versioning workload
Poor query design
Large intermediate results
Long transactions
```

Fixing the workload can be more effective than simply increasing capacity.

---

# 13. Row versioning

TempDB can also support row-versioning mechanisms.

This becomes important in concurrency scenarios.

Conceptually:

```text id="w4p9k1"
Transaction A
changes row
     |
     v
Previous row version
     |
     v
tempdb/version store
     |
     v
Transaction B
can access appropriate version
```

This can reduce reader/writer blocking in suitable configurations.

But long-running transactions can cause version information to remain necessary for longer.

---

# 14. Long transactions + version store

Suppose:

```text id="m8c2v6"
Transaction A starts
       ↓
Changes data
       ↓
Transaction stays open
       ↓
More changes occur
       ↓
Version information accumulates
```

Now you have another reason to monitor transaction duration.

A transaction that's open for hours isn't merely a locking problem.

It can have broader resource implications.

---

# 15. How to investigate an expensive query

Start with:

```text id="t5q9w2"
1. Query Store
2. Execution plan
3. Actual vs estimated rows
4. Memory grant information
5. Sort/hash warnings
6. Logical reads
7. CPU
8. Duration
9. Waits
10. tempdb activity
```

You're trying to answer:

> **"What work is the query actually doing?"**

---

# 16. Example diagnosis

Suppose the query is:

```text id="n4g7s2"
Duration:       120 sec
CPU:            35 sec
Logical reads:  80 million
Memory grant:   2 GB
Spills:         Yes
```

Plan:

```text id="y3m8p6"
Large Scan
   ↓
Hash Join
   ↓
Sort
   ⚠ spill
   ↓
Result
```

Estimated rows:

```text id="c6w1r9"
100,000
```

Actual rows:

```text id="z8q4v2"
15,000,000
```

This screams:

> **Cardinality estimation problem.**

Don't start by resizing the MI.

---

# 17. What would you investigate?

### A. Statistics

```text id="2f7m9c"
Are statistics current?
```

### B. Data distribution

```text id="5x8n1q"
Is the data highly skewed?
```

### C. Query predicates

```text id="j4r6v8"
Are predicates selective?
```

### D. Indexes

```text id="7p3k2m"
Can the query avoid scanning/sorting?
```

### E. Parameter sensitivity

```text id="q9w1e6"
Does performance vary dramatically by parameter?
```

### F. Query design

```text id="s2m5k8"
Is the query processing far more data than necessary?
```

---

# 18. A common bad fix

Developer:

> "The query spills to tempdb. Let's add more indexes everywhere."

No.

An index may help, but you need to understand the operator causing the spill.

Similarly:

> "Let's increase memory."

Also incomplete.

If the query is badly estimating 15 million rows as 100, giving it more resources may only hide the problem temporarily.

---

# 19. Another real scenario: SELECT *

Suppose:

```sql id="y6n2r4"
SELECT *
FROM Orders
WHERE CustomerID = @CustomerID;
```

The application actually uses only:

```text id="m3p7x9"
OrderID
OrderDate
TotalAmount
```

But `SELECT *` retrieves every column.

If the table contains:

```text id="c5j8w2"
50+ columns
large strings
JSON
LOB data
```

you're moving much more data than needed.

A targeted projection:

```sql id="h2k9q4"
SELECT
    OrderID,
    OrderDate,
    TotalAmount
FROM Orders
WHERE CustomerID = @CustomerID;
```

can reduce:

```text id="n8v3m1"
IO
Memory
Network traffic
Intermediate data
```

depending on the plan and schema.

---

# 20. The production DBA mindset

When you see a spill:

Don't ask only:

> "How do I give this query more memory?"

Ask:

> **"Why did this query need that much memory in the first place?"**

When you see high tempdb usage:

Don't ask only:

> "How do I increase tempdb?"

Ask:

> **"Which workload is generating the tempdb activity?"**

When you see a large memory grant:

Don't ask only:

> "Do we need more memory?"

Ask:

> **"Is the optimizer estimating the workload correctly?"**

---

# 21. Interview question

### Interviewer:

> "A query is spilling to tempdb. What would you investigate?"

Strong answer:

> "I'd first identify the operator causing the spill, typically a sort or hash operation, and compare estimated versus actual rows. I'd investigate statistics, data distribution, parameter sensitivity, indexing, and query shape because inaccurate cardinality estimates can result in inappropriate memory grants. I'd also look at whether concurrent queries are creating memory pressure. If the workload is valid and properly optimized, I'd then evaluate resource capacity. I wouldn't assume the root cause is simply insufficient tempdb."

Excellent senior-level answer.

---

# 22. Another interview question

> **"What is the difference between a memory grant that's too small and one that's too large?"**

Answer:

```text id="v8q2m6"
Too small
   ↓
Spills
   ↓
tempdb
   ↓
Slower query

Too large
   ↓
Memory reserved unnecessarily
   ↓
Other queries may wait for memory
   ↓
Concurrency suffers
```

---

# 23. Complete performance model

At this point, your mental model should be:

```text id="z4m7p2"
                 QUERY
                   |
        +----------+----------+
        |          |          |
       CPU        IO        MEMORY
        |          |          |
        |          |      Memory Grant
        |          |          |
        |          |       +--+--+
        |          |       |     |
        |          |    Too low Too high
        |          |       |     |
        |          |     Spill  Pressure
        |          |       |
        |          |     tempdb
        |          |
        +----------+
             |
           WAITS
             |
       +-----+-----+
       |           |
    Blocking    Other waits
```

Everything is connected.

---

# 24. Your senior DBA troubleshooting framework

When a query is slow:

```text
SYMPTOM
   ↓
WHEN did it start?
   ↓
WHO is affected?
   ↓
CURRENT workload
   ↓
CPU / IO / waits
   ↓
Blocking / deadlocks
   ↓
Query Store
   ↓
Execution plan
   ↓
Estimated vs actual rows
   ↓
Statistics
   ↓
Indexes
   ↓
Memory grants / spills
   ↓
Parameter sensitivity
   ↓
Application behavior
   ↓
ROOT CAUSE
   ↓
TARGETED FIX
   ↓
VALIDATION
```

That is the framework I want you to internalize.

---

# Next — Lesson 65

We'll move into **High Availability & Disaster Recovery for Azure SQL Managed Instance**.

We'll build a realistic enterprise architecture and cover:

* **HA vs DR**
* zone redundancy
* automatic failover
* regional disaster
* failover groups
* RPO/RTO
* backups vs DR
* primary-region outage
* accidental deletion vs regional disaster
* and a complete **"Azure region went down at 2 AM"** incident scenario.
