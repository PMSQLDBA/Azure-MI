# Lesson 144 — TempDB: Architecture, Contention, Spills & Performance Troubleshooting

Today we move into **TempDB**, one of the most important SQL Server system databases for performance troubleshooting.

The key idea:

> **TempDB is shared infrastructure. Problems in TempDB can affect many seemingly unrelated queries.**

---

## 1. What is TempDB?

SQL Server uses TempDB for temporary and intermediate work.

Examples include:

```text
Temporary tables
Table variables
Worktables
Workfiles
Sort operations
Hash operations
Row versioning
Some index operations
Intermediate query results
```

It is recreated when SQL Server starts.

---

# 2. TempDB is not just for `#temp` tables

A common misconception is:

> "TempDB is only used when I create a temporary table."

Not true.

A query such as:

```sql
SELECT *
FROM LargeTable
ORDER BY SomeColumn;
```

may require TempDB if SQL Server needs to spill intermediate data.

Likewise, hash operations can use TempDB when memory isn't sufficient.

---

# 3. The basic architecture

Think of TempDB as:

```text id="k2v8m1"
                TEMPDB
                   │
        ┌──────────┼──────────┐
        ↓          ↓          ↓
     Temp       Work       Version
    objects     space        store
```

Multiple sessions can use it simultaneously.

That shared nature is what makes TempDB problems potentially widespread.

---

# 4. Why TempDB matters

Suppose:

```text id="z4r7p2"
Query A → needs TempDB
Query B → needs TempDB
Query C → needs TempDB
Query D → needs TempDB
```

If TempDB becomes a bottleneck, all four workloads can suffer.

You may initially think:

> "Four different queries suddenly became slow."

The common dependency may be TempDB.

---

# 5. TempDB files

TempDB can have multiple data files.

Conceptually:

```text id="c8m3x5"
tempdb.mdf
tempdb_2.ndf
tempdb_3.ndf
tempdb_4.ndf
```

The purpose isn't simply:

> "More files = more performance."

The goal is to reduce allocation contention and distribute TempDB activity appropriately.

---

# 6. Don't blindly create dozens of files

A common piece of outdated advice is:

> "Always create eight TempDB files."

That's too simplistic.

The appropriate number depends on:

```text id="n7q1v6"
CPU count
Workload
Allocation contention
Existing configuration
Version
Environment
```

Start reasonably, measure, and adjust based on evidence.

---

# 7. Equal file sizing

TempDB data files should generally be configured consistently.

For example:

```text id="u1k6r9"
File 1 → 8 GB
File 2 → 8 GB
File 3 → 8 GB
File 4 → 8 GB
```

rather than:

```text id="d3p8x2"
File 1 → 8 GB
File 2 → 2 GB
File 3 → 20 GB
File 4 → 1 GB
```

Uneven sizing can create undesirable allocation behavior.

---

# 8. Autogrowth

TempDB should not constantly need to grow during normal workload.

Bad scenario:

```text id="h6w4m9"
TempDB = 10 GB

Workload grows
↓
File fills
↓
Autogrowth
↓
More workload
↓
Autogrowth again
↓
Performance disruption
```

Pre-size TempDB appropriately.

---

# 9. Autogrowth is not capacity planning

This distinction is important.

Autogrowth is a **safety mechanism**, not your primary sizing strategy.

Think:

```text id="q3j8v5"
Pre-size for expected workload
+
Reasonable growth setting
```

rather than:

```text id="x9m2c7"
Start tiny
+
Let SQL Server grow forever
```

---

# 10. TempDB contention

One classic TempDB problem is allocation contention.

SQL Server has internal allocation structures that can become hotspots under heavy temporary-object activity.

Conceptually:

```text id="v5r1n8"
Many sessions
     ↓
TempDB allocations
     ↓
Same internal structures
     ↓
Contention
     ↓
Waiting
```

---

# 11. Symptoms of TempDB contention

Possible symptoms include:

```text id="s7k3p9"
Many concurrent sessions
TempDB-related waits
High temporary-object activity
Unexpected latency
Workloads slowing together
```

But again:

> A TempDB-related wait does not automatically prove TempDB is the root cause.

---

# 12. TempDB contention vs TempDB capacity

These are different problems.

### Capacity problem

```text id="a4m9x2"
TempDB is running out of space
```

### Allocation contention

```text id="j6p1r7"
TempDB has space
but sessions contend for allocation structures
```

### I/O problem

```text id="w8c5n3"
TempDB storage is too slow
```

### Query spill problem

```text id="b2v7k4"
Queries are writing intermediate data to TempDB
```

Treat them separately.

---

# 13. TempDB space usage

A critical troubleshooting question:

> **Who is using TempDB?**

Potential consumers include:

```text id="e5x8q1"
User objects
Internal objects
Version store
```

You need to distinguish these categories.

---

# 14. User objects

Examples:

```text id="m3r9v6"
#Temporary tables
##Global temporary tables
Table-like temporary structures
```

These are explicitly created by workloads.

---

# 15. Internal objects

SQL Server itself may use TempDB for:

```text id="p7c2y8"
Sorts
Hashes
Spools
Worktables
Intermediate operations
```

A query can therefore consume substantial TempDB without ever executing:

```sql id="g4t1n6"
CREATE TABLE #Something
```

---

# 16. Version store

Row versioning can consume TempDB.

This becomes especially important with:

```text id="q8v3m5"
Snapshot-based isolation
Read committed snapshot
Online operations
Long-running transactions
```

---

# 17. Row versioning concept

Suppose a row changes:

```text id="n2k7r4"
Old version
    ↓
stored for readers
    ↓
New version
```

A transaction that still needs the older version can read it.

This improves concurrency in many scenarios.

But version storage requires resources.

---

# 18. Long transactions and version store

Imagine:

```text id="z5c1p8"
Transaction begins
       ↓
Keeps running for hours
       ↓
Other transactions modify rows
       ↓
Old row versions must remain available
       ↓
Version store grows
```

Long-running transactions can therefore contribute to TempDB pressure.

---

# 19. This creates an important connection

Earlier we discussed:

```text id="u7m3q2"
Blocking
```

Now we have:

```text id="r9v5k1"
Long transaction
      ↓
Locks
      +
Version retention
      ↓
Concurrency + TempDB impact
```

One transaction can create multiple performance problems.

---

# 20. TempDB spills

A spill occurs when an operator needs more memory than it received and writes intermediate data to TempDB.

Typical operators include:

```text id="d8x2m6"
Sort
Hash Match
Hash Aggregate
```

---

# 21. Example: sort spill

Suppose SQL Server expects:

```text id="c3v7q9"
10,000 rows
```

but actually receives:

```text id="k1p8r4"
10,000,000 rows
```

The memory grant may be inadequate.

The sort can spill:

```text id="j5m2x8"
Memory
  ↓
Insufficient
  ↓
TempDB
  ↓
Additional I/O
```

---

# 22. Why spills are important

A spill can cause:

```text id="v6n3c1"
More I/O
More TempDB activity
Longer duration
Potential concurrency effects
```

A spill is therefore often a symptom of a deeper estimation or memory-grant issue.

---

# 23. Don't just "make TempDB faster"

Suppose a query spills because the optimizer estimated:

```text id="s2r7k5"
100 rows
```

but the actual result is:

```text id="f9x1m4"
10 million rows
```

Buying faster TempDB storage may reduce the symptom.

But fixing the cardinality problem could prevent the spill altogether.

Root cause matters.

---

# 24. Memory grants

Before execution, SQL Server estimates how much memory an operator may need.

Conceptually:

```text id="y4k8q2"
Estimate rows
    +
Row width
    +
Operation type
    ↓
Memory grant
```

If the estimate is wrong, the grant can be wrong.

---

# 25. Under-grant

Too little memory:

```text id="c6p2w9"
Insufficient grant
     ↓
Spill
     ↓
TempDB activity
     ↓
Slower query
```

---

# 26. Over-grant

Too much memory is also a problem.

Suppose one query requests:

```text id="m8v1x5"
10 GB
```

and 20 such queries run concurrently.

Potentially:

```text id="r3q7k2"
20 × 10 GB
=
200 GB requested
```

That can create severe memory pressure.

So:

> **Both under-granting and over-granting are performance problems.**

---

# 27. Memory grant feedback

Modern SQL Server versions include mechanisms that can learn from previous execution behavior and adjust memory grants for future executions in supported scenarios.

Conceptually:

```text id="n4c8y1"
Execution 1
   ↓
Grant too large/small
   ↓
Observe actual usage
   ↓
Adjust future grant
```

This can reduce repeated memory-grant problems.

---

# 28. TempDB and indexes

Some index operations can use TempDB.

For example, index creation or rebuild operations can require significant temporary workspace depending on the operation and configuration.

Therefore:

```text id="p1x6m9"
Maintenance window
+
Large index operation
+
TempDB workload
```

can create temporary resource pressure.

---

# 29. TempDB and sorting

Queries such as:

```sql id="t8m3v2"
ORDER BY
```

can trigger sorting.

If the sort can't fit in memory, TempDB may be involved.

This is why queries that look simple syntactically can still create substantial TempDB activity.

---

# 30. TempDB and hashing

Hash joins and hash aggregates can also require workspace.

If the memory grant is insufficient:

```text id="z6q1r5"
Hash operation
   ↓
Spill
   ↓
TempDB
```

Again, investigate why the grant was inadequate.

---

# 31. Spool operators

Execution plans may contain spool operators.

Spools can materialize intermediate results, potentially using TempDB depending on the operation.

Don't automatically treat every spool as bad.

Ask:

```text id="x3k8n7"
Why was the spool introduced?
How much data does it process?
Is it expensive?
Can the underlying query design be improved?
```

---

# 32. TempDB storage

TempDB performance depends heavily on storage characteristics.

Important considerations include:

```text id="a5m2q8"
Latency
Throughput
IOPS
Concurrency
Storage configuration
```

But don't jump to storage tuning until you've established that TempDB I/O is actually limiting performance.

---

# 33. Separate TempDB from user database I/O

If a query is slow, determine:

```text id="u9r4c6"
Is it reading user data?
Is it writing TempDB?
Is it waiting on TempDB?
Is it doing both?
```

The distinction can completely change the fix.

---

# 34. TempDB file layout

A common production configuration includes:

```text id="e7n1v5"
Multiple equally sized data files
+
Appropriate growth settings
+
Fast storage
```

The exact number should be based on workload and contention evidence.

---

# 35. Log file

TempDB also has a transaction log.

Don't forget it.

A TempDB workload can experience log-related pressure as well as data-file pressure.

Monitor:

```text id="k2x7m4"
Data file usage
Log usage
Growth events
I/O latency
```

---

# 36. TempDB sizing

There is no universal:

> "TempDB should be exactly X GB."

Sizing depends on workload.

Consider:

```text id="p8v3q1"
Peak temporary tables
Largest sorts
Hash spills
Version store
Maintenance operations
Concurrent workload
```

Size for realistic peak demand rather than average demand.

---

# 37. Capacity planning

A useful process:

```text id="c5m9x2"
Observe peak usage
      ↓
Identify major consumers
      ↓
Estimate growth
      ↓
Pre-size
      ↓
Monitor
      ↓
Adjust
```

---

# 38. Don't clear TempDB blindly

If TempDB suddenly grows, don't immediately restart SQL Server just to "clear it."

A restart recreates TempDB, but it may:

```text id="w1r6k8"
Mask the cause
Disrupt production
Lose useful diagnostic evidence
```

First understand why it grew.

---

# 39. Common causes of unexpected TempDB growth

Possible causes:

```text id="q4n7p3"
Large temporary tables
Large sorts
Hash spills
Version store
Long-running transactions
Index operations
Unexpected workload
Application changes
```

---

# 40. A practical investigation

Suppose TempDB reaches:

```text id="j6c2v9"
95% used
```

First ask:

```text id="r8m1x5"
What is consuming it?
```

Then classify:

```text id="v3q7k2"
User objects?
Internal objects?
Version store?
```

Only then choose a fix.

---

# 41. Example: version store problem

You discover:

```text id="z9p4c1"
Version store = huge
```

Then investigate:

```text id="m2k7x6"
Longest-running transaction
Isolation level
Snapshot usage
Application transaction behavior
```

Maybe the root cause is one transaction that has been open for hours.

---

# 42. Example: internal-object problem

Suppose:

```text id="s5n8r2"
Internal objects = huge
```

Now investigate queries performing:

```text id="f1v6q9"
Large sorts
Large hashes
Spills
Complex intermediate processing
```

Execution plans become particularly important.

---

# 43. Example: user-object problem

Suppose:

```text id="h7c3m5"
User objects = huge
```

Look for:

```text id="x2p8v4"
Large #temp tables
Temporary staging
Application-generated temporary objects
Sessions holding temporary objects
```

Now the application/query design may be the correct place to investigate.

---

# 44. Query Store connection

Query Store can identify queries with:

```text id="n6k1q8"
High duration
High reads
High execution count
Spill-prone plans
```

Then execution plans can show:

```text id="u4r9m2"
Sort
Hash
Spool
Memory grant
```

Together they help identify TempDB consumers.

---

# 45. TempDB troubleshooting workflow

Memorize this:

```text id="b8x5j1"
TempDB problem
      ↓
Space or performance?
      ↓
What is consuming it?
      ↓
User / Internal / Version
      ↓
Which sessions/queries?
      ↓
Why are they consuming it?
      ↓
Fix root cause
      ↓
Verify
```

---

# 46. TempDB contention workflow

For contention:

```text id="y3m7q6"
Observe waits
      ↓
Confirm TempDB allocation contention
      ↓
Review number/sizing of data files
      ↓
Check workload pattern
      ↓
Adjust configuration if justified
      ↓
Measure again
```

Don't configure blindly.

---

# 47. TempDB spill workflow

For spills:

```text id="r2k8n5"
Find spilling operator
      ↓
Check estimated vs actual rows
      ↓
Check memory grant
      ↓
Check statistics
      ↓
Check query/index design
      ↓
Fix estimation/grant/workload
      ↓
Verify spill disappears or becomes insignificant
```

---

# 48. Version-store workflow

For version-store growth:

```text id="c7x1m9"
Identify version-store growth
      ↓
Find long-running transactions
      ↓
Inspect isolation/versioning behavior
      ↓
Identify application transaction
      ↓
Shorten/fix transaction behavior
      ↓
Monitor recovery
```

---

# 49. Interview question

### "What is TempDB used for?"

Strong answer:

> "TempDB is SQL Server's shared temporary workspace. It supports temporary objects, internal worktables and workfiles, sorts and hashes that spill, row versioning, and certain maintenance operations."

---

# 50. Interview question

### "Why use multiple TempDB data files?"

Strong answer:

> "Multiple appropriately sized TempDB data files can reduce allocation contention by distributing activity across files. The number should be based on workload and observed contention rather than a universal fixed rule."

---

# 51. Interview question

### "What causes TempDB to grow unexpectedly?"

Strong answer:

> "Common causes include large temporary objects, query spills, internal worktables, row-versioning activity, long-running transactions, index operations, and sudden workload changes."

---

# 52. Interview question

### "How would you troubleshoot a TempDB problem?"

A strong sequence:

```text id="q8w4m1"
1. Determine whether the issue is space, I/O, or contention.
2. Identify what category consumes space.
3. Identify the sessions/queries involved.
4. Inspect plans and memory grants.
5. Check transactions and versioning.
6. Correct the underlying cause.
7. Validate TempDB behavior afterward.
```

---

# 53. The biggest misconception

Don't think:

> **"TempDB is slow, so add more TempDB files."**

Instead think:

> **"What exactly is wrong with TempDB, and what evidence proves it?"**

More files can help a specific allocation-contention problem.

They won't automatically solve:

```text id="m3q9x6"
Bad query design
Spills
Version-store growth
Insufficient storage
Long transactions
Huge temp tables
```

---

# 54. Lesson 144 takeaway

Keep these four TempDB problem categories separate:

```text id="j5v2k8"
1. SPACE
   TempDB is filling up.

2. CONTENTION
   Sessions compete for allocation resources.

3. I/O
   TempDB storage is too slow.

4. WORKLOAD
   Queries are generating excessive temporary work.
```

And remember:

> **TempDB problems are often symptoms of another problem.**

A spill may point to bad cardinality estimates.

Version-store growth may point to a long transaction.

Excessive temporary objects may point to application design.

Allocation contention may point to insufficiently distributed TempDB files.

---
**Next:** **Lesson 145**
**Final:** **Lesson 150**
