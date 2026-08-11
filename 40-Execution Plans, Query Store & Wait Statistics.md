# Lesson 72 — Execution Plans, Query Store & Wait Statistics

This is the lesson where the pieces start connecting.

Previously, we learned:

```text
Application is slow
       ↓
Find the query
       ↓
Find the resource/problem
```

Now we go one level deeper:

```text
Why is this particular query expensive?
```

The three major tools/concepts are:

```text
Query Store
Execution Plans
Wait Statistics
```

Think of them as three different cameras looking at the same incident.

---

# 1. The three-camera model

```text
                 SQL Performance Incident
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
     Query Store     Execution Plan   Wait Stats
          |              |              |
     Historical       How SQL        What SQL
     behavior         executes       is waiting for
```

### Query Store

Answers:

> **"What changed over time?"**

### Execution plan

Answers:

> **"How is SQL executing this query?"**

### Wait statistics

Answers:

> **"What is SQL waiting on?"**

Combine all three and your diagnosis becomes much stronger.

---

# 2. Example query

Consider:

```sql id="m8q2x4"
SELECT
    CustomerID,
    OrderDate,
    TotalAmount
FROM Orders
WHERE CustomerID = 1001;
```

SQL Server has to decide:

```text id="r4n7p2"
How should I find these rows?
```

Possible approaches include:

```text id="x6c3m9"
Index Seek
Index Scan
Table/Clustered Scan
```

The optimizer chooses based on estimates and available indexes/statistics.

---

# 3. What is an execution plan?

An execution plan is essentially SQL Server's chosen strategy for executing a query.

Conceptually:

```text id="j9w3k6"
Query
  ↓
Optimizer
  ↓
Execution Plan
  ↓
Operators
  ↓
Data access
```

A plan can contain operators such as:

```text id="v5m1r8"
Index Seek
Index Scan
Key Lookup
Nested Loops
Hash Match
Sort
Aggregate
Filter
```

---

# 4. Index Seek

A seek means SQL Server can navigate directly to relevant portions of an index.

Conceptually:

```text id="p3x8n2"
Index
+----------------------+
| Customer 1000        |
| Customer 1001  ←──── SQL finds this
| Customer 1002        |
+----------------------+
```

If only a small percentage of rows are needed, this can be efficient.

---

# 5. Index Scan

A scan examines a larger portion or all of an index.

Conceptually:

```text id="c7m2r9"
Index
 ↓
Read many/all entries
 ↓
Find matching rows
```

A scan is **not automatically bad**.

This is extremely important.

---

# 6. Seek isn't always good

Suppose a table has:

```text id="n5q8x1"
100 million rows
```

and your query needs:

```text id="a4r7m3"
90 million rows
```

A seek followed by millions of lookups could be much worse than a scan.

So don't use:

> "Seek = good, Scan = bad."

Instead ask:

> **"Is the chosen access method appropriate for the amount of data being processed?"**

---

# 7. The optimizer cares about estimated cost

The optimizer evaluates possible strategies.

Simplified:

```text id="u8p3k5"
Plan A → estimated cost 100
Plan B → estimated cost 10
Plan C → estimated cost 50
```

It chooses the plan it estimates to be best.

But:

> **The optimizer can only make good decisions when its information is reasonably accurate.**

That's where statistics become important.

---

# 8. Estimated vs actual rows

This is one of the most important execution-plan concepts.

Suppose an operator says:

```text id="w2m7c9"
Estimated rows = 10
Actual rows    = 500,000
```

That's a massive difference.

SQL Server planned as though it would process:

```text id="k6x1r4"
10 rows
```

but actually processed:

```text id="q9n3v7"
500,000 rows
```

That can lead to a poor downstream strategy.

---

# 9. Why estimates matter

Imagine the optimizer expects:

```text id="s4p8m1"
10 rows
```

It might choose:

```text id="h7c2x5"
Nested Loops
```

But if reality is:

```text id="z3n9q6"
5 million rows
```

a different join strategy might have been more appropriate.

Now you can get:

```text id="b8m4r2"
Huge CPU
Huge IO
Long execution time
```

---

# 10. Common causes of row-estimate errors

Investigate:

```text id="f6q1x8"
Stale statistics
Data distribution changes
Correlated predicates
Parameter sensitivity
Complex expressions
Implicit conversions
Poor cardinality assumptions
```

Don't automatically assume:

> "Statistics are stale."

Prove it.

---

# 11. Key Lookup

Suppose you have an index on:

```text id="j5m8r3"
CustomerID
```

but your query needs:

```text id="v2x7n9"
CustomerID
OrderDate
TotalAmount
Status
```

The index finds the matching rows, but SQL Server needs additional columns from the underlying table.

It may perform:

```text id="q4c1m7"
Index Seek
   ↓
Key Lookup
   ↓
Base table
```

---

# 12. When Key Lookup becomes expensive

Suppose:

```text id="p8r3x6"
Index seek finds:
20 rows
```

Fine.

But:

```text id="n7m2q9"
Index seek finds:
5 million rows
```

Now:

```text id="z1c8v4"
5 million lookups
```

can become extremely expensive.

This is why you should inspect:

> **How many times does this operator execute?**

not just:

> "The plan contains a Key Lookup."

---

# 13. Nested Loops

Nested Loops conceptually does:

```text id="m4x9q2"
For each row from input A
    find matching rows in B
```

Excellent when:

```text id="w6p1r8"
Outer input is small
Inner access is efficient
```

Potentially expensive when:

```text id="c3n7v5"
Outer input becomes huge
```

---

# 14. Hash Match

Hash joins are often useful for larger datasets.

Conceptually:

```text id="s8r2m6"
Build hash table
       ↓
Probe hash table
       ↓
Matching rows
```

They can consume significant memory.

So when you see a Hash Match, investigate:

```text id="q5x9n1"
Data volume
Memory grant
Spills
Row estimates
```

Again:

> Hash Match isn't inherently bad.

---

# 15. Sort operator

Sorting can be expensive.

Example:

```sql id="9b3r6v"
SELECT *
FROM Orders
ORDER BY OrderDate;
```

If SQL Server doesn't already have useful ordering available:

```text id="h8m2p7"
Read rows
   ↓
Sort
   ↓
Return
```

For millions of rows, this can consume significant CPU/memory.

---

# 16. Memory grants

Some operators require memory to perform work such as:

```text id="k1x7q4"
Sort
Hash
Aggregation
```

SQL Server may request a memory grant.

Example:

```text id="r6m3n8"
Requested = 8 GB
Used      = 200 MB
```

That may indicate overestimation.

Or:

```text id="v9p2c5"
Requested = 200 MB
Needed    = 8 GB
```

That can cause spills to disk.

Both situations deserve investigation.

---

# 17. Spills

Imagine:

```text id="j7q4m1"
Sort needs:
8 GB

Available grant:
2 GB
```

SQL Server may spill work to tempdb.

Conceptually:

```text id="n3x8r6"
Query
 ↓
Sort
 ↓
Memory insufficient
 ↓
Spill
 ↓
tempdb
 ↓
Additional IO
 ↓
Slower query
```

A spill can be a symptom of estimation or memory-grant problems.

---

# 18. Wait statistics

Now we switch cameras.

Suppose a query takes:

```text id="c5m9q2"
30 seconds
```

That number alone doesn't tell you why.

Wait information can help.

Conceptually:

```text id="z8r3x6"
Query
  |
  +-- CPU
  |
  +-- IO wait
  |
  +-- Lock wait
  |
  +-- Memory wait
  |
  +-- Parallelism-related wait
```

---

# 19. `LCK_M_*` waits

These indicate lock-related waiting.

Conceptually:

```text id="p7n2v4"
Session A
   |
   | holds lock
   v
Session B
   |
   X
 waits
```

If you see significant lock waits, investigate:

```text id="h4x8m1"
Blocking
Transaction duration
Isolation behavior
Indexes
Application transaction scope
```

---

# 20. `PAGEIOLATCH_*`

These waits generally indicate waiting for data pages to be read from storage into memory.

Conceptually:

```text id="r6m3q9"
Query
 ↓
Needs data page
 ↓
Page not available in memory
 ↓
Storage read
 ↓
Wait
```

If these are significant, investigate:

```text id="v8p1c5"
IO latency
Data access patterns
Query efficiency
Working set
Storage/resource pressure
```

Don't simply conclude:

> "Storage is broken."

The query may simply be reading far too much data.

---

# 21. `RESOURCE_SEMAPHORE`

This type of wait is associated with waiting for query execution memory grants.

Conceptually:

```text id="n2q7m5"
Query A → huge memory grant
Query B → waiting
Query C → waiting
```

Investigate:

```text id="x4r9p1"
Large sorts
Hash operations
Bad estimates
Large concurrent queries
Memory grant behavior
```

---

# 22. `CXPACKET` and `CXCONSUMER`

Parallel query execution can produce parallelism-related waits.

Important:

> **Do not automatically treat CXPACKET as a problem.**

Parallelism itself isn't bad.

Suppose a query uses multiple workers:

```text id="w7m3n8"
Coordinator
   |
   +-- Worker 1
   +-- Worker 2
   +-- Worker 3
   +-- Worker 4
```

Some waiting is normal.

The question is:

> **Is parallelism contributing to a real performance problem?**

You need context.

---

# 23. Wait stats are clues, not verdicts

If you see:

```text id="k5p9r2"
PAGEIOLATCH
```

don't immediately say:

> "Increase storage."

If you see:

```text id="m7x3c8"
LCK_M_*
```

don't immediately say:

> "Kill the blocking session."

If you see:

```text id="q2n8v6"
CXPACKET
```

don't immediately say:

> "Disable parallelism."

Always connect:

```text id="j4r1p7"
Wait
 ↓
Query
 ↓
Plan
 ↓
Resource
 ↓
Root cause
```

---

# 24. Full production example

Let's investigate this:

> **"The Orders API is taking 20 seconds."**

---

## Step 1 — Identify the SQL query

Application telemetry identifies:

```text id="c8m2r5"
Query ID = 742
```

---

## Step 2 — Query Store

Historical average:

```text id="n7q3x9"
250 ms
```

Current:

```text id="p4r8m1"
20 sec
```

Something changed.

---

## Step 3 — Compare plans

Old:

```text id="w6x2k8"
Index Seek
Nested Loops
```

New:

```text id="z9m4c7"
Index Scan
Hash Match
Sort
```

Huge change.

---

## Step 4 — Check estimated vs actual rows

New plan:

```text id="j3n8v5"
Estimated = 50
Actual    = 2,000,000
```

Very significant mismatch.

---

## Step 5 — Check waits

You see substantial:

```text id="r7q2m6"
PAGEIOLATCH_*
```

and some CPU.

This is consistent with the query reading much more data than expected.

---

## Step 6 — Investigate why estimate is wrong

Check:

```text id="x5m1p9"
Statistics
Parameter values
Data distribution
Recent data load
Query changes
```

Suppose you discover:

```text id="v8c3r4"
A large data load happened yesterday.
```

Statistics no longer represent the new distribution well enough for this query.

---

# 25. Root-cause chain

Now you can explain the incident:

```text id="q6n2m8"
Large data load
      ↓
Cardinality estimate became inaccurate
      ↓
Optimizer chose different plan
      ↓
Scan + Hash + Sort
      ↓
Much more data processed
      ↓
IO increased
      ↓
Query duration increased
      ↓
Application latency increased
```

That's a real root-cause explanation.

---

# 26. Notice the difference

Weak DBA explanation:

> "SQL is slow because IO is high."

Better:

> "IO increased because Query 742 changed plans and began scanning a much larger portion of the Orders data."

Even better:

> "A large data-distribution change caused inaccurate cardinality estimates, leading the optimizer to choose a scan/hash plan. The query consequently processed substantially more data, increasing IO and execution time."

That's the level you should aim for.

---

# 27. Query Store workflow

When a query is slow:

```text id="y2m7p4"
Find Query ID
     ↓
Check historical duration
     ↓
Check CPU
     ↓
Check execution count
     ↓
Check plans
     ↓
Compare good vs bad plan
     ↓
Investigate what changed
```

This should become muscle memory.

---

# 28. Execution-plan workflow

When looking at a plan:

```text id="n8r3x5"
1. Look at overall shape
        ↓
2. Find expensive operators
        ↓
3. Compare estimated vs actual rows
        ↓
4. Look for scans/lookups
        ↓
5. Check joins
        ↓
6. Check sorts/hashes
        ↓
7. Check warnings/spills
        ↓
8. Check memory grants
```

Don't stare at every operator randomly.

---

# 29. What does "expensive operator" mean?

Execution-plan percentages are estimates relative to the plan.

For example:

```text id="m5x8q2"
Index Scan = 70%
```

That does **not** automatically mean:

> "Delete the index scan."

It means:

> "This operator represents a large estimated portion of the plan's work."

Investigate why it exists and whether it is appropriate.

---

# 30. Real-life trap

You see:

```text id="v4p7n1"
Table Scan = 95%
```

You immediately create an index.

After deployment:

```text id="k9m2r6"
Performance gets worse.
```

Why?

The query may legitimately need:

```text id="c3x8q5"
80% of the table
```

An index may add overhead without meaningfully reducing work.

Always understand the access pattern.

---

# 31. Another trap — "rebuild all indexes"

Query is slow.

DBA runs:

```text id="q7m3n8"
Rebuild all indexes
```

Query remains slow.

Why?

Because the root cause was:

```text id="h2r9v5"
Blocking
```

or:

```text id="w6x1c4"
Bad plan
```

or:

```text id="p8m4q2"
Parameter sensitivity
```

Maintenance is not a substitute for diagnosis.

---

# 32. Performance investigation template

When you investigate a query, document:

```text id="x9q3m7"
Query ID:
Database:
Application:
Start time:
Baseline duration:
Current duration:
Execution count:
CPU:
IO:
Waits:
Plan ID:
Estimated rows:
Actual rows:
Recent changes:
Root cause:
Immediate mitigation:
Permanent fix:
Validation:
```

This is an excellent production incident template.

---

# 33. Interview question

### "How do Query Store, execution plans, and wait statistics work together?"

A strong answer:

> "Query Store gives me historical behavior and helps identify regressions and plan changes. The execution plan shows how SQL Server is accessing and processing the data, including operators, estimates, joins, lookups, and memory-related behavior. Wait statistics show where execution is spending time waiting. I correlate all three rather than treating any one metric as the root cause."

Excellent answer.

---

# 34. Interview question

### "Is an Index Scan always bad?"

Correct answer:

> **No.**

A scan can be appropriate when a large portion of the data is required.

You evaluate:

```text id="d2q8m6"
Rows needed
Table size
Selectivity
Available indexes
Query shape
Actual workload
```

---

# 35. Interview question

### "What does a large estimated-vs-actual row mismatch tell you?"

Answer:

> "It indicates that the optimizer's cardinality estimate differs substantially from reality. I'd investigate statistics, data distribution, parameter sensitivity, predicates, and other factors that could cause inaccurate estimates."

---

# 36. Interview question

### "What do PAGEIOLATCH waits mean?"

A good answer:

> "They generally indicate sessions waiting for data pages to be read into memory. I would correlate those waits with the affected queries, IO latency, data access patterns, and execution plans before concluding that the storage subsystem itself is the root cause."

---

# 37. Your mental model

Put everything together:

```text id="r4m8x2"
                  Application Slow
                         |
                         v
                    Find Query
                         |
                         v
                    Query Store
                         |
                +--------+--------+
                |                 |
          Plan regression?     No regression
                |                 |
                v                 v
          Compare plans       Check workload
                |                 |
                +--------+--------+
                         |
                         v
                   Execution Plan
                         |
             +-----------+-----------+
             |           |           |
          Estimates     IO         CPU
             |           |           |
        Statistics      Waits     Operators
             |           |           |
             +-----------+-----------+
                         |
                         v
                   Root Cause
```

---

# 38. The golden rule

When investigating SQL performance:

> **Don't optimize what you haven't measured.**

And:

> **Don't treat an execution-plan operator or wait type as a root cause by itself.**

Your job is to connect the evidence.

---

# Next — Lesson 73

Next we'll take this further into **indexes and statistics in Azure SQL Managed Instance**.

We'll cover:

* clustered vs nonclustered indexes
* included columns
* covering indexes
* composite indexes
* index key order
* SARGability
* filtered indexes
* duplicate/redundant indexes
* write overhead
* fragmentation
* statistics
* auto-update statistics
* missing-index recommendations
* and a real production case where **adding an index makes the application faster but makes the database overall worse**.
