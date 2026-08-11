# Lesson 133 — TempDB Internals, Version Store, Spills & TempDB Contention — Expert Level

Today we focus on **TempDB**, one of the most important and most misunderstood parts of SQL Server.

The key idea:

> **TempDB is not simply a place where developers create temporary tables. It is a shared workspace used by the SQL Server engine for many internal operations.**

---

## 1. What is TempDB?

TempDB is a system database recreated when SQL Server starts.

It is used for temporary and intermediate work such as:

```text
Temporary tables
Table variables
Worktables
Sorts
Hash operations
Spills
Row versioning
Internal query operations
```

Unlike ordinary databases, TempDB is designed to be disposable.

---

# 2. TempDB lifecycle

When SQL Server starts:

```text
SQL Server starts
       ↓
TempDB recreated
       ↓
Files initialized
       ↓
System begins using TempDB
```

When SQL Server shuts down:

```text
TempDB contents disappear
```

Persistent business data should never depend on TempDB.

---

# 3. Why TempDB performance matters

Because so many workloads can use TempDB:

```text
Query A ─┐
Query B ─┤
Query C ─┼→ TempDB
Query D ─┤
Versioning┘
```

A TempDB bottleneck can affect many otherwise unrelated queries.

---

# 4. TempDB isn't necessarily "temporary tables"

This misconception causes many bad diagnoses.

You can have heavy TempDB usage without a single:

```sql
CREATE TABLE #Temp
```

statement.

For example:

```text
Large Sort
     ↓
Needs workspace
     ↓
Insufficient memory
     ↓
TempDB spill
```

No temporary table was explicitly created.

---

# 5. Worktables and workfiles

SQL Server may internally create temporary structures for operations such as:

```text
Sort
Hash
Cursor-related processing
Intermediate query results
Other internal operations
```

These structures can consume TempDB resources.

---

# 6. The TempDB mental model

Think of TempDB as:

```text
                  TempDB
                    │
       ┌────────────┼─────────────┐
       ↓            ↓             ↓
 Temporary      Query          Row
 Objects        Workspace      Versions
       │            │             │
       ↓            ↓             ↓
 #tables        Sort/Hash      Isolation
```

This explains why TempDB problems can have very different symptoms.

---

# 7. TempDB spills

One of the most common performance-related TempDB uses is a spill.

A query operator expects:

```text
1,000 rows
```

but actually processes:

```text
10,000,000 rows
```

The memory grant isn't sufficient.

Then:

```text
Operator
 ↓
Memory insufficient
 ↓
Spill
 ↓
TempDB
```

---

# 8. Operators that commonly spill

Pay particular attention to:

```text
Sort
Hash Match
Hash Aggregate
```

These operators can require significant workspace memory.

---

# 9. Why spills happen

Common causes include:

```text
Poor cardinality estimates
Insufficient memory grant
Large data volumes
Concurrent memory pressure
Unexpected row widths
```

The most interesting case is often:

```text
Bad estimate
    ↓
Bad memory grant
    ↓
Spill
```

---

# 10. Spills are symptoms

Don't immediately conclude:

> "TempDB is too slow."

A spill may be caused by the query receiving insufficient memory because the optimizer estimated the workload incorrectly.

Therefore:

```text
Spill
 ↓
Check memory grant
 ↓
Check cardinality estimate
```

---

# 11. Example

Estimated:

```text
Rows = 500
```

Actual:

```text
Rows = 5,000,000
```

SQL Server might allocate memory based on the much smaller estimate.

Result:

```text
Hash Match
 ↓
Memory exhausted
 ↓
TempDB spill
```

The root cause may be the **cardinality estimate**, not TempDB itself.

---

# 12. Memory grant and TempDB

This relationship is extremely important:

```text
Cardinality estimate
        ↓
Memory grant
        ↓
Sort / Hash
        ↓
Enough memory?
      /     \
    Yes      No
     ↓        ↓
Continue    Spill
              ↓
           TempDB
```

---

# 13. Version store

TempDB also stores row versions for row-versioning features.

This is fundamental to:

```text
READ COMMITTED SNAPSHOT
SNAPSHOT ISOLATION
```

and other engine behaviors involving row versioning.

---

# 14. Why row versions exist

Suppose:

```text
Transaction A
updates a row
```

A reader may need access to an earlier committed version.

Conceptually:

```text
Current row
    ↓
New version

Older version
    ↓
Version store
```

The reader can access the appropriate version rather than waiting for the writer in certain row-versioned isolation scenarios.

---

# 15. Version-store architecture

Conceptually:

```text
User Database
     ↓
Row modification
     ↓
Version information
     ↓
TempDB version store
     ↓
Reader accesses appropriate version
```

The exact internals are more sophisticated, but this is the core mental model.

---

# 16. Version-store cleanup

Old row versions aren't kept forever.

SQL Server must determine when versions are no longer needed.

Conceptually:

```text
Version created
      ↓
Readers may need it
      ↓
Reader finishes
      ↓
Version becomes unnecessary
      ↓
Cleanup
```

---

# 17. Long-running transactions and version store

A long-running transaction can prevent older row versions from being removed as quickly as they otherwise could.

Therefore:

```text
Long transaction
      ↓
Old versions remain useful
      ↓
Version store grows
```

This can contribute to TempDB pressure.

---

# 18. Long-running readers matter too

You don't necessarily need a massive UPDATE to cause version-store retention.

A transaction that keeps a consistent snapshot open for a long time can keep old versions relevant.

This is another reason transaction duration matters.

---

# 19. RCSI and TempDB

If Read Committed Snapshot Isolation is enabled:

```text
Readers
 ↓
Versioned reads
 ↓
TempDB versioning activity
```

This can reduce blocking but increases version-store workload.

It's a concurrency tradeoff.

---

# 20. The tradeoff

Traditional locking:

```text
Reader ↔ Writer
Potential blocking
```

Row versioning:

```text
Reader
 ↓
Version store
Less reader/writer blocking
```

But now:

```text
TempDB
+
Version maintenance
```

become part of the workload.

---

# 21. TempDB file configuration

TempDB consists of one or more data files and a log file.

The files need appropriate sizing and configuration for the workload.

A common goal is to avoid repeated file-growth events during normal operation.

---

# 22. Multiple TempDB data files

Multiple data files can help with certain TempDB allocation-contention scenarios.

But don't blindly apply:

> "Always create eight files."

The appropriate configuration depends on:

```text
CPU count
Workload
Version
Existing contention
File sizes
Growth behavior
```

---

# 23. Equal-sized data files

When multiple TempDB data files are used, they are generally configured with equal initial sizes and growth settings.

This helps SQL Server distribute allocations more predictably.

---

# 24. Autogrowth

Suppose TempDB is:

```text
100 GB
```

and needs more space.

If it grows by:

```text
10 MB
```

repeatedly, that can produce many growth events.

Better sizing aims to make growth events less frequent and predictable.

---

# 25. Pre-sizing

If you know TempDB routinely requires substantial space, pre-sizing it can avoid repeated runtime growth.

Conceptually:

```text
Bad:
Small → grow → grow → grow → grow

Better:
Appropriate initial size
       ↓
Normal workload
       ↓
Occasional growth
```

---

# 26. Instant File Initialization

Instant File Initialization can reduce zero-initialization overhead for data-file growth under supported conditions.

Important distinction:

```text
Data files
vs
Log file
```

The behavior and requirements are different.

Don't assume a setting that helps data files automatically applies to the transaction log.

---

# 27. TempDB log

TempDB also has a transaction log.

Even though TempDB is recreated after restart, its log still matters during runtime.

Heavy activity can require substantial log throughput.

---

# 28. TempDB log pressure

Potential causes include:

```text
Large temporary modifications
Large temporary objects
Long-running transactions
Versioning activity
Heavy internal operations
```

Again, identify the workload before changing configuration.

---

# 29. TempDB contention

Contention occurs when many workers compete for shared TempDB resources.

Conceptually:

```text
Worker 1 ─┐
Worker 2 ─┤
Worker 3 ─┼→ TempDB shared structures
Worker 4 ─┤
Worker 5 ─┘
```

Potential symptoms include elevated waits and reduced throughput.

---

# 30. Allocation contention

Historically, certain TempDB allocation structures could become hotspots under heavy concurrent workloads.

Modern SQL Server versions include improvements that reduce some of these problems.

But workload-specific contention can still exist.

---

# 31. Don't confuse space pressure with contention

These are different.

### Space pressure

```text
TempDB needs more space
```

### Contention

```text
Many workers are competing for access
to the same TempDB resources
```

You can have:

```text
Plenty of free space
+
Severe contention
```

or:

```text
No contention
+
TempDB nearly full
```

---

# 32. TempDB full vs TempDB slow

Another critical distinction.

### TempDB full

There isn't enough allocated/free space.

### TempDB slow

Operations are taking too long because of:

```text
I/O
Contention
Spills
Versioning
Concurrency
```

The remedies are different.

---

# 33. Temporary tables

Temporary tables:

```sql
CREATE TABLE #Orders
(
    OrderId int,
    CustomerId int
);
```

are physically backed by TempDB.

They can be very useful.

Don't avoid them simply because they use TempDB.

---

# 34. Temporary tables can improve performance

A temp table can sometimes:

```text
Break a complex query
Materialize intermediate results
Provide statistics
Reduce repeated work
Improve cardinality estimation
Create useful indexes
```

So:

> **TempDB usage is not automatically bad.**

---

# 35. Temp tables vs table variables

Historically, table variables had weaker cardinality estimation behavior in many scenarios.

Modern SQL Server versions have improved table-variable estimation in some cases, including deferred compilation under appropriate configurations.

Still, temp tables and table variables have different behavior and should be selected based on workload.

---

# 36. Materialization can be useful

Suppose a complex query repeatedly generates the same expensive intermediate result.

You might choose:

```text id="u3d6x9"
Complex computation
      ↓
#Temp table
      ↓
Index
      ↓
Multiple downstream queries
```

Instead of recomputing the intermediate result repeatedly.

---

# 37. Temp table indexing

A temp table can have indexes:

```sql
CREATE INDEX IX_Orders_CustomerId
ON #Orders(CustomerId);
```

This can make downstream processing significantly more efficient.

But indexing itself costs:

```text id="p7v1c5"
CPU
I/O
TempDB space
```

Again:

> Measure the tradeoff.

---

# 38. TempDB and CTEs

A common misconception:

> "CTEs don't use TempDB."

A CTE is primarily a query-expression construct; it does not automatically mean materialization into TempDB.

However, the execution plan may still use TempDB for operations such as:

```text id="f2r6m8"
Sorts
Spills
Worktables
Other intermediate processing
```

The CTE itself isn't equivalent to a temp table.

---

# 39. TempDB and table variables

Table variables do use TempDB internally despite their syntax looking memory-oriented.

Therefore:

```text id="y5c8q1"
DECLARE @T TABLE (...)
```

does not mean:

> "This object is guaranteed to live entirely in RAM."

---

# 40. TempDB and spills

Consider:

```text id="a7q3m6"
Query
 ↓
Sort 500 million rows
 ↓
Memory insufficient
 ↓
TempDB
```

You may see substantial TempDB usage even though the query contains no temporary table.

---

# 41. Diagnosing TempDB problems

Start with:

```text id="x9v2k5"
1. Is TempDB growing?
2. Is it running out of space?
3. Is there high I/O?
4. Is there contention?
5. Is the version store large?
6. Which queries are spilling?
7. Which sessions use large temp objects?
8. Are long transactions retaining versions?
```

---

# 42. Identify the workload

Don't simply say:

> "TempDB is huge."

Find the source.

Potential sources:

```text id="m6p4w8"
Large sorts
Hash spills
Temp tables
Table variables
Row versioning
Index operations
ETL workloads
Reporting queries
Application-generated temporary objects
```

---

# 43. Spill investigation

For a suspected spill:

```text id="v1n7c4"
Find operator
 ↓
Check spill information
 ↓
Compare estimated vs actual rows
 ↓
Inspect memory grant
 ↓
Check row width
 ↓
Check concurrent memory pressure
```

This often points toward the root cause.

---

# 44. Example spill diagnosis

Suppose:

```text id="e9r3k2"
Hash Match
Estimated rows = 5,000
Actual rows = 8,000,000
```

Memory grant:

```text id="w4x7p1"
100 MB
```

Spill:

```text id="q8m2c6"
Yes
```

Likely investigation:

```text id="k5v9s3"
Why was cardinality estimated so badly?
```

rather than:

```text id="r6p1m4"
Immediately add more TempDB storage
```

---

# 45. Example version-store diagnosis

Suppose:

```text id="b2k8q5"
TempDB version store
      ↑
Rapid growth
```

Investigate:

```text id="m7c1x9"
Which databases use row versioning?
Which sessions have long-running transactions?
Which transaction is preventing cleanup?
```

---

# 46. Example contention diagnosis

Suppose:

```text id="h4n6p2"
TempDB space = healthy
I/O latency = healthy
But throughput is poor
```

Now investigate:

```text id="s8q3v7"
Allocation contention
Latch waits
High concurrent TempDB usage
Workload patterns
```

Space expansion alone won't necessarily solve it.

---

# 47. TempDB and concurrency

One query spilling once may be harmless.

But imagine:

```text id="d6p1m8"
1,000 concurrent queries
+
each spills heavily
```

Now:

```text id="z5q7c3"
TempDB I/O
+
contention
+
CPU
+
latency
```

can become severe.

---

# 48. The multiplicative effect

Performance problems can compound:

```text id="x3m9v1"
Bad estimate
 ↓
Small memory grant
 ↓
Spill
 ↓
TempDB I/O
 ↓
Longer query
 ↓
More concurrent queries
 ↓
More TempDB pressure
 ↓
More latency
```

This is why root-cause diagnosis matters.

---

# 49. TempDB and memory pressure

Suppose the server is under memory pressure.

Queries may receive constrained grants.

Then:

```text id="q2r8m5"
Less memory
 ↓
More spills
 ↓
More TempDB
 ↓
More I/O
```

Therefore, TempDB problems can sometimes be **downstream symptoms of memory pressure**.

---

# 50. TempDB and bad cardinality estimates

Another common chain:

```text id="v7n4k2"
Bad statistics
 ↓
Bad cardinality
 ↓
Bad memory grant
 ↓
Spill
 ↓
TempDB
```

This links:

```text Statistics
→ Execution Plans
→ Memory
→ TempDB
```

---

# 51. TempDB and transaction design

Long-running transactions can affect both:

```text id="a4m9x7"
Locking
```

and:

```text id="f8c2q1"
Version-store retention
```

So transaction design has both concurrency and TempDB implications.

---

# 52. Operational monitoring

Useful categories to monitor include:

```text id="e3p6w9"
TempDB used/free space
Data-file growth
Log usage
Version-store usage
I/O latency
Spill activity
Wait statistics
Long-running transactions
TempDB allocation contention
```

---

# 53. Don't monitor only file size

A 500 GB TempDB isn't necessarily unhealthy.

A 20 GB TempDB isn't necessarily healthy.

File size tells you:

```text id="k1r8c6"
How much space is allocated.
```

It does not tell you:

```text id="w9m4p2"
Why the space is being used
or whether performance is good.
```

---

# 54. Expert scenario

You see:

```text id="c5q8n1"
TempDB = 90% full
```

Don't immediately resize.

First ask:

```text id="z7m3v6"
What is consuming the space?
```

Possible answer:

```text id="h1r9x4"
A long-running snapshot transaction
```

In that case, blindly adding storage treats the symptom rather than the cause.

---

# 55. Expert scenario #2

TempDB:

```text id="m8k2p5"
20% used
```

Yet application latency is high.

Waits show:

```text id="v4q7n3"
TempDB-related contention
```

This tells you:

> **Low space utilization does not rule out TempDB performance problems.**

---

# 56. Expert scenario #3

A reporting query:

```text id="s6x1r8"
Sort
 ↓
Spill
 ↓
TempDB
```

The query runs every minute.

Potential improvements might involve:

```text id="c9m5q2"
Better index ordering
Better cardinality estimation
Reduced row width
Query redesign
Memory-grant improvement
```

Not necessarily faster storage.

---

# 57. TempDB troubleshooting hierarchy

Use:

```text id="u4p8k1"
TEMPDB PROBLEM
      ↓
Space?
 ├── YES → Identify consumer
 └── NO
      ↓
Performance?
      ↓
I/O?
Contention?
Version store?
Spills?
      ↓
Identify workload
      ↓
Fix root cause
```

---

# 58. Interview question

### "What is TempDB used for?"

Strong answer:

> "TempDB is SQL Server's shared temporary workspace. It supports temporary objects, internal worktables and workfiles, spills from operations such as sorts and hashes, and row-versioning mechanisms. Its health can therefore affect a wide range of workloads."

---

# 59. Interview question

### "What causes TempDB spills?"

Strong answer:

> "Spills occur when operators such as sorts or hash operations require more workspace memory than is available to them. Common contributors include inaccurate cardinality estimates, insufficient memory grants, large datasets, wide rows, and memory pressure."

---

# 60. Interview question

### "How would you troubleshoot TempDB growth?"

Strong answer:

> "First I'd determine what is consuming the space—temporary objects, version store, spills, or other internal activity. Then I'd identify the sessions or workload responsible, check for long-running transactions, inspect memory grants and spills, and determine whether the issue is space pressure, I/O, or contention."

---

# 61. Interview question

### "Do more TempDB files always improve performance?"

Strong answer:

> "No. Multiple TempDB data files can help with certain allocation-contention scenarios, but file count should be based on the SQL Server version, workload, CPU configuration, and observed contention. Adding files blindly isn't a universal solution."

---

# 62. Interview question

### "Why can RCSI increase TempDB usage?"

Strong answer:

> "RCSI uses row versioning to allow readers to access appropriate committed versions without the same reader-writer blocking behavior of locking reads. Those versions require TempDB resources, so row-versioning workloads can increase TempDB activity."

---

# 63. The expert mental model

Don't think:

```text id="q6x3v9"
TempDB = temp tables
```

Think:

```text id="b8m4r1"
                    TEMPDB
                       │
       ┌───────────────┼────────────────┐
       ↓               ↓                ↓
 Temporary          Query            Version
 Objects            Workspace        Store
       │               │                │
       ↓               ↓                ↓
 #temp tables     Sorts/Hashes       RCSI/Snapshot
                      │
                      ↓
                    Spills
```

And remember the deeper chain:

```text id="n2k7p5"
Statistics
   ↓
Cardinality
   ↓
Memory Grant
   ↓
Operator
   ↓
Spill
   ↓
TempDB
   ↓
I/O / Contention
   ↓
Query latency
```

---

# 64. Lesson 133 takeaway

The most important rule:

> **Don't treat TempDB as the root cause simply because TempDB is involved.**

A TempDB problem may actually originate from:

* Bad cardinality estimates
* Poor indexing
* Large sorts/hashes
* Memory pressure
* Long-running transactions
* Row-versioning workload
* Excessive temporary-object usage
* Allocation contention

Your expert troubleshooting sequence:

```text
TEMPDB
 ↓
SPACE?
 ↓
WHAT CONSUMES IT?
 ↓
SPILL / VERSION / TEMP OBJECT / INTERNAL?
 ↓
WHY?
 ↓
QUERY / TRANSACTION / MEMORY / CONTENTION
 ↓
FIX ROOT CAUSE
 ↓
MEASURE AGAIN
```

**Next: Lesson 134 — Statistics, Cardinality Estimation & Histograms: How SQL Server Predicts Row Counts — Expert Level.**
