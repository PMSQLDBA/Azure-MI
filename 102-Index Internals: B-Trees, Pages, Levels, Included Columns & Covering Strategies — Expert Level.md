# Lesson 135 — Index Internals: B-Trees, Pages, Levels, Included Columns & Covering Strategies — Expert Level

Today we're going underneath the usual **"create an index on this column"** advice.

The expert question is:

> **How does SQL Server physically navigate an index, and why does a particular index reduce work?**

---

## 1. What an index really is

At a high level, a rowstore index is an organized structure that helps SQL Server locate rows without scanning an entire table or index.

A useful mental model is a **B-tree**:

```text
                 Root
                  ↓
             Intermediate
              ↙        ↘
          Intermediate  Intermediate
             ↓              ↓
                 Leaf level
```

The exact implementation details are more nuanced, but this model is fundamental.

---

# 2. The three important levels

Think of a typical B-tree as:

```text
Root
 ↓
Intermediate levels
 ↓
Leaf level
```

The root helps SQL Server find the appropriate branch.

Intermediate pages narrow the search.

The leaf level contains the index's actual row-level entries.

---

# 3. Why the tree is useful

Suppose an index contains millions of keys.

SQL Server doesn't need to inspect every key.

Instead:

```text
Root
 ↓
Find branch
 ↓
Find narrower branch
 ↓
Reach leaf
 ↓
Read relevant range
```

This is the fundamental reason an index seek can be efficient.

---

# 4. Index pages

SQL Server stores data in pages.

For traditional rowstore structures, the standard data-page size is:

```text
8 KB
```

A useful hierarchy is:

```text
Database
 ↓
File
 ↓
Page
 ↓
Row
```

Indexes are themselves stored across pages.

---

# 5. Page size and row width

If rows are narrow:

```text
More rows/page
```

If rows are wide:

```text
Fewer rows/page
```

This matters because reading 1,000 pages is very different from reading 100,000 pages.

---

# 6. Why row width matters to indexes

Imagine an index contains:

```text id="x2z6c7"
10 million rows
```

If each entry is relatively narrow, the index may occupy far fewer pages than the base table.

That makes an index scan potentially much cheaper than scanning the table.

---

# 7. Narrow indexes can be powerful

Suppose the table contains:

```text id="6d0m5q"
100 columns
```

but your query only needs:

```text id="f7r3p2"
CustomerId
OrderDate
```

A narrow index containing those relevant columns may be dramatically smaller than the table.

---

# 8. Clustered index

A clustered index determines the organization of the table's data rows.

The clustered index's leaf level contains the table rows themselves.

Conceptually:

```text id="4r8c1v"
Clustered Index
       ↓
Leaf level
       ↓
Actual table rows
```

That's why a clustered table and a heap behave differently.

---

# 9. Heap

A heap is a table without a clustered index.

Conceptually:

```text id="0m3h9x"
Table
 ↓
Unordered data pages
```

A heap isn't inherently bad.

Some workloads can use heaps appropriately.

But heaps have different access and maintenance characteristics from clustered tables.

---

# 10. Clustered index vs nonclustered index

Think:

### Clustered

```text id="4p2v8m"
Leaf
 ↓
Actual row data
```

### Nonclustered

```text id="z6k1r3"
Leaf
 ↓
Index key
+
Included/indexed information
+
Row locator
```

The row locator allows SQL Server to find the underlying row when additional data is needed.

---

# 11. Row locator

For a clustered table, a nonclustered index entry generally identifies the row using the clustered key.

For a heap, the row locator is a RID.

Conceptually:

```text id="m9q4x2"
Nonclustered Index
      ↓
Row locator
      ↓
Underlying row
```

---

# 12. Why clustered-key width matters

Suppose your clustered key is:

```text id="c8m5y1"
INT
```

versus:

```text id="p3r7k9"
VARCHAR(200)
```

A wide clustered key can increase the size of nonclustered indexes because the clustered key is used as the row locator in nonclustered indexes on clustered tables.

This is an important design consideration.

---

# 13. The clustered key isn't just about clustering

When selecting a clustered key, consider:

```text id="q6v1n8"
Width
Uniqueness
Stability
Insert pattern
Access patterns
Impact on nonclustered indexes
```

A seemingly innocent choice can affect the entire indexing strategy.

---

# 14. Composite index

Consider:

```sql id="g0v5x4"
CREATE INDEX IX_Orders_Customer_Date
ON dbo.Orders(CustomerId, OrderDate);
```

The key order is:

```text id="4s2m8q"
CustomerId
    ↓
OrderDate
```

That order matters enormously.

---

# 15. Leftmost-prefix concept

An index on:

```text id="0j6p7w"
(CustomerId, OrderDate)
```

is naturally organized first by:

```text id="2r9x3k"
CustomerId
```

and then by:

```text id="n7c5m1"
OrderDate
```

This makes it especially useful for predicates involving the leading key.

---

# 16. Example

Excellent match:

```sql id="w4d1q8"
WHERE CustomerId = 100
  AND OrderDate >= '2026-01-01';
```

The index can narrow by:

```text id="8k3z5m"
CustomerId
```

then navigate within that customer's date range.

---

# 17. Less ideal example

Consider:

```sql id="3b8nq7"
WHERE OrderDate >= '2026-01-01';
```

with only:

```text id="q2m6x9"
(CustomerId, OrderDate)
```

The leading key isn't being constrained.

SQL Server may still use the index in some ways, but it doesn't provide the same efficient ordered access that an index beginning with `OrderDate` would.

---

# 18. Key order should follow workload

Don't memorize:

> "Put the most selective column first."

That's an oversimplification.

Consider:

```text id="h9v2m4"
Predicates
Join conditions
Sort requirements
Grouping
Range conditions
Selectivity
Frequency
```

The best order depends on how the index will actually be used.

---

# 19. Equality before range — useful heuristic

A common design heuristic is:

```text id="5p7x1r"
Equality columns
        ↓
Range columns
```

For example:

```text id="3j8c6n"
(CustomerId, OrderDate)
```

can be excellent for:

```sql id="g5y2q9"
WHERE CustomerId = @CustomerId
  AND OrderDate >= @Date;
```

But this is a heuristic, not a universal law.

---

# 20. ORDER BY and index order

An index can sometimes provide rows already ordered for:

```sql id="m6z8w1"
ORDER BY CustomerId, OrderDate;
```

This can avoid an explicit Sort.

Therefore, indexes can reduce both:

```text id="3q7x2p"
I/O
```

and:

```text id="j9n4v5"
CPU/memory spent sorting
```

---

# 21. GROUP BY and index order

Appropriate index ordering can sometimes help aggregation operations by providing useful ordering.

For example:

```sql id="y1c5k8"
GROUP BY CustomerId
```

may benefit from an index ordered by:

```text id="h7m3q2"
CustomerId
```

depending on the rest of the query and chosen plan.

---

# 22. Included columns

Non-key columns can be added using `INCLUDE`.

Example:

```sql id="e9p2w6"
CREATE INDEX IX_Orders_Customer
ON dbo.Orders(CustomerId)
INCLUDE (OrderDate, TotalAmount, Status);
```

The key is:

```text id="b8k4r1"
CustomerId
```

The included columns are stored at the leaf level but aren't part of the index's search ordering.

---

# 23. Why INCLUDE exists

Suppose the query is:

```sql id="c7n1m5"
SELECT OrderDate, TotalAmount, Status
FROM dbo.Orders
WHERE CustomerId = @CustomerId;
```

The index can use:

```text id="t2r6q9"
CustomerId
```

to find rows and already contain:

```text id="w4m8p3"
OrderDate
TotalAmount
Status
```

This can avoid lookups.

---

# 24. Covering index

An index is **covering** for a query when it contains all information required to execute that query without retrieving additional columns from the underlying table.

Conceptually:

```text id="0q7z3n"
Query
 ↓
Index
 ↓
Everything required
```

No extra base-table lookup is needed.

---

# 25. Covering doesn't mean "include everything"

A terrible strategy is:

```text id="7n3m5p"
CREATE INDEX
(
Key
)
INCLUDE
(
Column1,
Column2,
Column3,
...
Column100
);
```

This can create a huge index.

Consequences:

```text id="e5k8x1"
More storage
More writes
More logging
More maintenance
Larger memory footprint
More page splits
```

Cover strategically.

---

# 26. The covering-index tradeoff

You are trading:

```text id="u1c8m5"
Extra index size
```

for:

```text id="w6q3n9"
Less lookup work
```

The goal is not maximum coverage.

The goal is:

> **Enough coverage to eliminate expensive work.**

---

# 27. Key vs INCLUDE

This is one of the most important index-design distinctions.

### Key column

Affects:

```text id="c9v4m2"
Ordering
Seekability
Range navigation
Join/group/order capabilities
```

### Included column

Primarily provides:

```text id="x7n1q5"
Payload at the leaf level
```

It doesn't provide the same ordering/search semantics as a key column.

---

# 28. Example

Index:

```text id="r5m8k3"
(CustomerId, OrderDate)
INCLUDE (TotalAmount)
```

This supports:

```text id="q2v6x9"
Search by CustomerId
Navigate by OrderDate
Return TotalAmount
```

But `TotalAmount` isn't part of the index ordering.

---

# 29. Key lookup and covering indexes

Consider:

```text id="j8x2m7"
Nonclustered Seek
      ↓
Key Lookup
```

If the lookup is expensive, one possibility is:

```text id="b5q9r1"
Add required columns to INCLUDE
```

Then:

```text id="m7c3v8"
Nonclustered Seek
      ↓
Return required columns
```

But first verify that the lookup is actually a bottleneck.

---

# 30. When a Key Lookup is fine

Suppose:

```text id="s3n6p8"
Estimated rows = 5
Actual rows = 5
```

Five lookups are trivial.

Adding several wide columns to an index just to eliminate five lookups could make the overall system worse.

---

# 31. When a Key Lookup is dangerous

Suppose:

```text id="d1k7x4"
Estimated = 10
Actual = 5,000,000
```

Now:

```text id="f8q2m6"
5 million lookups
```

could be extremely expensive.

The underlying issue may be:

```text id="y4r9n3"
Bad cardinality estimate
```

or:

```text id="k2m5p7"
Index not sufficiently covering/selective
```

---

# 32. Index width

Every additional index column increases index size.

This affects:

```text id="a8x3m6"
Pages
I/O
Cache usage
Maintenance
Write cost
```

A narrow index is generally easier to maintain and cache.

---

# 33. Write amplification

Every INSERT/UPDATE/DELETE may need to maintain multiple indexes.

Suppose a table has:

```text id="f7m2q9"
1 clustered index
+
10 nonclustered indexes
```

An insert may need to modify multiple structures.

Therefore:

> **Indexes accelerate reads but can make writes more expensive.**

---

# 34. Update cost

If an indexed column changes:

```text id="k4v8x1"
UPDATE indexed column
```

SQL Server may need to modify index entries.

If a non-key included column changes, the leaf-level index payload may also need updating.

The cost depends on the index and operation.

---

# 35. Index maintenance

Over time, index structures may experience page-level changes and fragmentation depending on workload.

Maintenance decisions should be based on:

```text id="p9m3r7"
Actual fragmentation
Page density
Workload
Query behavior
Maintenance cost
```

Don't rebuild every index simply because a percentage is above some arbitrary threshold.

---

# 36. Page splits

Suppose an index page is full.

A new key needs to be inserted between existing values.

SQL Server may need to split the page:

```text id="n7q1x5"
Full Page
   ↓
Split
 ↙   ↘
Page A  Page B
```

This can increase:

```text id="z4m8c2"
I/O
Logging
Page count
Fragmentation
```

---

# 37. Random vs sequential inserts

Consider an ever-increasing key:

```text id="a2m7p9"
1
2
3
4
5
...
```

New rows generally arrive toward the end of the index.

With a random key:

```text id="c8v4n1"
83921
12
5017
223
...
```

new rows may need insertion throughout the tree.

This can create different page-split behavior.

---

# 38. Sequential clustered keys

A monotonically increasing clustered key can reduce many page-split patterns associated with random insertion.

But it can also create a hot insertion point at the end under very high concurrency.

So:

> **Sequential isn't universally perfect either.**

---

# 39. Fill factor

Fill factor controls how full index pages are initially populated during certain index creation/rebuild operations.

Conceptually:

```text id="w6p2q8"
100% fill
→ maximum density

Lower fill
→ more free space
```

The idea is to leave room for future inserts.

---

# 40. Don't automatically lower fill factor

A common mistake:

> "Fragmentation? Set fill factor to 70%."

That may create:

```text id="x8m4k1"
More pages
More I/O
Larger index
Lower cache efficiency
```

Fill factor should address a demonstrated page-split pattern, not be applied blindly.

---

# 41. Fragmentation isn't automatically a performance problem

An index can show high logical fragmentation while queries remain fast.

What matters is:

```text id="j7q3n9"
Does fragmentation materially increase I/O or degrade the workload?
```

Don't confuse a maintenance metric with actual user impact.

---

# 42. Page density

Low page density means fewer rows per page.

That can increase:

```text id="e5r1m8"
Number of pages
I/O
Memory footprint
```

Sometimes page density matters more than fragmentation percentage.

---

# 43. Index seek isn't always the goal

Remember:

```text id="z9x2c6"
Seek ≠ automatically faster
```

If a query needs most of an index:

```text id="m5q7v3"
Scan may be cheaper.
```

The optimizer chooses based on estimated total cost.

---

# 44. Narrow scan vs wide lookup

Suppose:

```text id="p8m3r5"
Index A = 100 MB
Table = 100 GB
```

A query requiring 70% of rows might scan Index A and avoid reading the entire table.

This can be much better than:

```text id="w4k6n2"
Index seek
+
Millions of lookups
```

---

# 45. Index intersection

SQL Server can sometimes combine information from multiple indexes.

Conceptually:

```text id="t7x1m9"
Index A
  +
Index B
  ↓
Combined result
```

But this isn't a reason to create dozens of single-column indexes.

Purpose-built composite indexes are often more effective for important workloads.

---

# 46. Filtered indexes

A filtered index contains only rows satisfying a predicate.

Example:

```sql id="b6n2x8"
CREATE INDEX IX_Orders_Open
ON dbo.Orders(CustomerId, OrderDate)
WHERE Status = 'Open';
```

This can be much smaller than indexing every row.

---

# 47. Why filtered indexes can be powerful

Suppose:

```text id="j3v8q5"
100 million total rows
2 million Open rows
```

A filtered index covering only:

```text id="n1m7c4"
Status = 'Open'
```

may be dramatically smaller.

Benefits can include:

```text id="p5r2x9"
Less storage
Less I/O
Smaller memory footprint
Lower maintenance cost
```

But only queries compatible with the filter can benefit.

---

# 48. Computed columns

If a query frequently searches an expression:

```sql id="h4q8m2"
WHERE FirstName + LastName = 'JohnSmith';
```

one option can be a persisted computed column with an appropriate index.

Conceptually:

```text id="v6x1n7"
Expression
 ↓
Computed column
 ↓
Index
 ↓
Efficient access
```

This can transform a difficult expression into an indexable access path in appropriate scenarios.

---

# 49. SARGability and indexes

A predicate like:

```sql id="q9c4w6"
WHERE OrderDate >= @StartDate
```

is naturally suited to range navigation.

Compare:

```sql id="m2x7p1"
WHERE YEAR(OrderDate) = 2026
```

The function changes how the predicate relates to the indexed column.

A rewritten range predicate is often more index-friendly.

---

# 50. Index design is query-driven

Don't begin with:

> "Which columns have low cardinality?"

Begin with:

```text id="c7n3m8"
Which queries matter?
What predicates do they use?
How do they join?
How do they sort?
What columns do they return?
How many rows do they process?
```

Then design the access path.

---

# 51. One index can serve multiple queries

A good composite index can support several related query patterns.

For example:

```text id="r4x8q2"
(CustomerId, OrderDate)
```

may support:

```text id="k3m7p1"
Customer lookup
Customer + date range
Customer ordering
Customer aggregation
```

The exact benefit depends on query shape.

---

# 52. Duplicate indexes

A database may contain:

```text id="x9m2k7"
IX_A(CustomerId)
IX_B(CustomerId)
```

with almost identical definitions.

This can produce unnecessary:

```text id="v5q1n8"
Storage
Write overhead
Maintenance
```

Index inventory should be reviewed periodically.

---

# 53. Redundant indexes

Suppose:

```text id="z6c3m9"
IX_A(CustomerId, OrderDate)
```

and:

```text id="w2r8p4"
IX_B(CustomerId)
```

The second may be redundant for some workloads, but **not automatically**.

The narrower index might still be useful because it is:

```text id="q4n7x1"
Smaller
Cheaper to scan
Better cached
```

Index redundancy requires workload analysis.

---

# 54. Index usage statistics

SQL Server exposes usage information that can help identify:

```text id="k8m1p6"
Seeks
Scans
Lookups
Updates
```

This can help determine whether an index is actually being used.

But usage statistics are not permanent historical truth; they have lifecycle/reset considerations.

---

# 55. Don't drop an index solely because seeks = 0

Maybe it supports:

```text id="e2x6r9"
Rare but critical query
```

or:

```text id="a5m8c3"
Constraint enforcement
```

or its usage statistics were reset.

Always understand why the index exists before removing it.

---

# 56. Index design and constraints

A primary key or unique constraint may automatically create an index depending on configuration.

Those indexes can serve both:

```text id="y7q2m4"
Integrity
```

and:

```text id="f9x3n8"
Query performance
```

Don't treat every index as purely a tuning artifact.

---

# 57. Covering strategy

A good process is:

```text id="r2v7k5"
Identify expensive query
       ↓
Find predicates
       ↓
Choose key columns
       ↓
Consider ORDER BY/GROUP BY
       ↓
Identify required output columns
       ↓
Add only useful INCLUDE columns
       ↓
Measure
```

---

# 58. Example

Query:

```sql id="s4n8q1"
SELECT OrderDate, TotalAmount, Status
FROM dbo.Orders
WHERE CustomerId = @CustomerId
  AND OrderDate >= @StartDate
ORDER BY OrderDate;
```

Potential index:

```text id="p6m2x9"
(CustomerId, OrderDate)
INCLUDE (TotalAmount, Status)
```

Why?

```text id="c8r4m1"
CustomerId
 → filtering

OrderDate
 → range + ordering

TotalAmount, Status
 → returned payload
```

This is a classic covering design.

---

# 59. But don't stop at the index definition

Ask:

```text id="w3k9p2"
How many rows does CustomerId return?
```

If one customer owns:

```text id="v6m1x8"
40 million rows
```

the query may still need substantial work.

A perfect index cannot make 40 million required rows disappear.

---

# 60. Indexing cannot fix excessive result sets

If the application requests:

```text id="m4x7q3"
10 million rows
```

the database still has to:

```text id="n9p2c6"
Read
Process
Transmit
Consume
```

An index may reduce finding those rows, but it cannot eliminate the cost of actually returning them.

---

# 61. Indexing and write-heavy systems

Imagine:

```text id="b8q4m1"
10,000 INSERTs/sec
```

Adding 20 nonclustered indexes may improve a few reads but dramatically increase write cost.

This is an architecture tradeoff:

```text id="h7r2x5"
Read performance
      ↕
Write performance
```

---

# 62. Indexing and memory

Indexes consume buffer-pool memory when actively used.

A database with:

```text id="s3m9v7"
Huge number of wide indexes
```

can reduce memory available for:

```text id="c1x6q8"
Data pages
Other indexes
Working sets
```

More indexes don't automatically mean better performance.

---

# 63. Index tuning should be workload-aware

A good index recommendation asks:

```text id="n5p8k2"
Frequency?
Duration?
CPU?
Logical reads?
Rows returned?
Write cost?
Storage cost?
Concurrency?
```

An index helping a query that runs once per month may be less valuable than one improving a query that runs 10,000 times per minute.

---

# 64. Expert scenario

You see:

```text id="f6m1r8"
Index Seek
+
Key Lookup
```

Someone says:

> "Make it covering."

Don't immediately do that.

First check:

```text id="p9x3c5"
Actual lookup count
Lookup cost
Index size
Query frequency
Write workload
```

Maybe five lookups are harmless.

---

# 65. Expert scenario #2

A query scans:

```text id="z2q7m4"
20 million rows
```

but returns:

```text id="k8c1n6"
100 rows
```

Potential issue:

```text id="y5r3p9"
Poor filtering/access path
```

Now indexing could be highly valuable.

---

# 66. Expert scenario #3

A query scans:

```text id="v4m8q2"
80% of a narrow index
```

and runs in:

```text id="b6n1x7"
150 ms
```

Don't create a seek-oriented index simply because you see "Scan."

The scan may already be the optimal strategy.

---

# 67. Interview question

### "What is the difference between clustered and nonclustered indexes?"

Strong answer:

> "The clustered index determines the organization of the table's row data at its leaf level. A nonclustered index is a separate structure whose leaf entries contain the index key, any included columns, and a locator to the underlying row when needed."

---

# 68. Interview question

### "What are included columns?"

Strong answer:

> "Included columns are non-key columns stored at the leaf level of a nonclustered index. They can allow an index to cover a query without participating in the index's search ordering."

---

# 69. Interview question

### "Why does clustered-key width matter?"

Strong answer:

> "On a clustered table, the clustered key is generally used as the row locator in nonclustered indexes. A wide clustered key can therefore increase the size of every nonclustered index."

---

# 70. Lesson 135 takeaway

The key idea:

> **A good index doesn't merely make a query "seek." It reduces the amount of unnecessary work required to produce the result.**

Your index-design model:

```text id="q4m8x2"
QUERY
 ↓
Predicates
 ↓
JOIN conditions
 ↓
ORDER BY / GROUP BY
 ↓
Choose KEY columns
 ↓
Choose INCLUDE columns
 ↓
Cover strategically
 ↓
Measure reads / CPU / duration
 ↓
Measure write + storage cost
```

And remember:

```text id="n7c3p9"
More indexes
≠
More performance
```

The expert goal is:

> **The smallest, most useful set of indexes that efficiently supports the actual workload without imposing unacceptable write, storage, memory, and maintenance costs.**

**Next: Lesson 136 — Index Fragmentation, Page Splits, Fill Factor & Index Maintenance — Expert Level.**
