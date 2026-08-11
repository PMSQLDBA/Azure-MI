# Lesson 145 — Indexing Deep Dive: SARGability, Key Order, Covering Indexes & Included Columns

Today we go deeper into one of the most important SQL Server performance skills:

> **Designing indexes that match how queries actually search, join, sort, and return data.**

A good index can transform a query from seconds to milliseconds. A bad index can add write overhead without solving anything.

---

## 1. What an index really does

Without a useful index, SQL Server may need to inspect a large portion of a table:

```text id="7q2m4x"
Table
 ↓
Scan many pages
 ↓
Find matching rows
```

With an appropriate index:

```text id="k8p5r1"
Index
 ↓
Locate matching keys
 ↓
Find required rows
```

The goal isn't simply:

> "Create an index."

The goal is:

> **Create an index that reduces the amount of unnecessary work for an important workload.**

---

# 2. The three major jobs of an index

An index can help SQL Server with:

```text id="m4c9v7"
1. Finding rows
2. Ordering rows
3. Covering required columns
```

For example:

```sql id="q3w8n1"
SELECT OrderId, OrderDate
FROM Orders
WHERE CustomerId = 100
ORDER BY OrderDate;
```

A well-designed index might help with both:

```text id="z6r2k5"
WHERE CustomerId = 100
```

and:

```text id="p9v4x8"
ORDER BY OrderDate
```

---

# 3. Clustered vs nonclustered index

A simplified mental model:

### Clustered index

Determines the physical/logical organization of the table's data rows around the clustered key.

### Nonclustered index

A separate B-tree structure containing indexed keys and row-location information.

Think:

```text id="b7n3m9"
Table
  ↑
Clustered index

Separate nonclustered indexes
  ↓
Index A
Index B
Index C
```

---

# 4. The clustered key matters everywhere

For a clustered table, the clustered key becomes part of nonclustered index row locators.

Therefore a very wide clustered key can make all nonclustered indexes larger.

Example:

```text id="x5k1q8"
Clustered key:
BIGINT
```

is generally much narrower than:

```text id="r2m7c4"
Clustered key:
Large composite string key
```

This doesn't mean every table should blindly use an integer identity column, but clustered-key design has consequences beyond the clustered index itself.

---

# 5. Composite indexes

Suppose:

```sql id="f8v3q2"
CREATE INDEX IX_Orders_Customer_Date
ON Orders(CustomerId, OrderDate);
```

The key order is:

```text id="n1x6m9"
CustomerId
    ↓
OrderDate
```

This ordering matters enormously.

---

# 6. The leftmost-prefix concept

An index beginning with:

```text id="t7p4k2"
(CustomerId, OrderDate)
```

is naturally useful for predicates involving:

```text id="c9m5v1"
CustomerId
```

and:

```text id="a2q8r6"
CustomerId + OrderDate
```

But it is not automatically ideal for a query filtering only on:

```text id="h4n7x3"
OrderDate
```

This is the **leftmost-prefix** principle.

---

# 7. Example

Index:

```text id="j6w2p9"
(CustomerId, OrderDate)
```

### Query A

```sql id="v1k8c5"
WHERE CustomerId = 100
```

Good match.

### Query B

```sql id="e3r7m2"
WHERE CustomerId = 100
AND OrderDate >= '2026-01-01'
```

Also a strong match.

### Query C

```sql id="u9q4x6"
WHERE OrderDate >= '2026-01-01'
```

Potentially much less useful as a seek predicate.

---

# 8. Equality before range

A very useful indexing heuristic is:

> **Equality predicates often come before range predicates in a composite index.**

For:

```sql id="s8m2k5"
WHERE CustomerId = @CustomerId
  AND OrderDate >= @StartDate
```

a natural candidate is:

```text id="p3v7n1"
(CustomerId, OrderDate)
```

because:

```text id="y6c9r4"
CustomerId = exact value
       ↓
OrderDate = range within that customer
```

---

# 9. But indexing isn't a formula

Don't blindly apply:

```text id="k1m5z8"
Equality → Range → Everything else
```

You must consider:

```text id="q7r2x4"
Join predicates
ORDER BY
GROUP BY
Selectivity
Query frequency
Data distribution
Covering requirements
Other workload
```

Index design is workload-specific.

---

# 10. Selectivity

Selectivity describes how effectively a predicate narrows the data.

Suppose:

```text id="w9c3p6"
Gender = 'M'
```

returns 50% of a table.

That's not very selective.

But:

```text id="d2h8k1"
CustomerId = 983742
```

might return 10 rows from 100 million.

That is highly selective.

---

# 11. High selectivity often favors seeks

If a predicate returns a tiny fraction of the table:

```text id="m5q1v7"
Seek
 ↓
Few rows
```

can be very efficient.

But remember:

> **High selectivity isn't the only factor in index design.**

---

# 12. Covering indexes

A query is said to be **covered** by an index when the index contains all the information required to satisfy the query, so SQL Server doesn't need to access the base table/clustered index for additional columns.

Example:

```sql id="n8c4x2"
SELECT OrderId, OrderDate, TotalAmount
FROM Orders
WHERE CustomerId = @CustomerId;
```

Potential index:

```text id="y7p3m9"
(CustomerId)
INCLUDE (OrderId, OrderDate, TotalAmount)
```

---

# 13. Why covering helps

Without coverage:

```text id="r4k8c1"
Index Seek
    ↓
Key Lookup
    ↓
Base table / clustered index
```

With coverage:

```text id="u2m6x5"
Index Seek
    ↓
Required columns already available
```

That can eliminate many lookups.

---

# 14. Key Lookup

A Key Lookup occurs when SQL Server uses a nonclustered index to locate rows but needs additional columns from the clustered index.

Example:

```text id="f9q3v7"
Seek
 ↓
100 rows
 ↓
100 Key Lookups
```

That might be fine.

But:

```text id="c5x1m8"
Seek
 ↓
1,000,000 rows
 ↓
1,000,000 lookups
```

can be extremely expensive.

---

# 15. Lookup tipping point

There is a general principle:

> As the number of qualifying rows grows, repeated lookups can become more expensive than scanning.

Therefore:

```text id="k2v9r5"
Small result
→ Seek + Lookup

Large result
→ Scan may be better
```

This is another reason not to assume that a seek is always superior.

---

# 16. Included columns

SQL Server allows:

```sql id="m7p4c2"
CREATE INDEX IX_Orders_Customer
ON Orders(CustomerId)
INCLUDE (OrderDate, TotalAmount);
```

Here:

```text id="n5x8q1"
Key:
CustomerId
```

and:

```text id="r3k6v9"
Included:
OrderDate
TotalAmount
```

---

# 17. Why use INCLUDE?

Included columns can help cover queries without making those columns part of the index's search/order key.

This can make the index more efficient than putting everything into the key.

---

# 18. Key columns vs included columns

### Key columns

Can help with:

```text id="j4w8m2"
Seek predicates
Ordering
Index navigation
```

### Included columns

Primarily provide:

```text id="s6p1x7"
Additional data for coverage
```

They aren't used the same way as key columns for navigating the B-tree.

---

# 19. Don't put every column in the key

Suppose you need:

```text id="b8r2m5"
CustomerId
OrderDate
Status
TotalAmount
SalesRep
Region
ProductDescription
```

Don't automatically create:

```text id="q1n7c4"
(CustomerId, OrderDate, Status, TotalAmount,
 SalesRep, Region, ProductDescription)
```

That can create a huge index.

Instead ask:

```text id="x9m3v6"
Which columns determine search/order?
Which columns are only needed for output?
```

Then consider:

```text id="e5k8p2"
Key columns
+
INCLUDE columns
```

---

# 20. SARGability

Now we reach a critical concept:

> **SARGability determines whether SQL Server can efficiently use a predicate to navigate an index.**

"SARGable" essentially means the predicate can be used effectively as a search argument.

---

# 21. A classic non-SARGable predicate

Suppose:

```sql id="u7c3m1"
WHERE YEAR(OrderDate) = 2026
```

The function is applied to the column.

This can make it harder for SQL Server to perform an efficient seek on an index beginning with `OrderDate`.

---

# 22. Rewrite it as a range

Instead:

```sql id="p6n2x8"
WHERE OrderDate >= '20260101'
  AND OrderDate <  '20270101'
```

Now SQL Server can potentially seek into the date range.

Conceptually:

```text id="w4r9k5"
OrderDate
   ↓
Start at 2026-01-01
   ↓
Read until 2027-01-01
```

---

# 23. Another example

Avoid:

```sql id="g2m7v4"
WHERE LEFT(LastName, 1) = 'S'
```

as a general indexing strategy.

The expression transforms the column before comparison.

Depending on the requirement, alternative designs may include:

```text id="n8x1c6"
Range predicates
Computed columns
Appropriate indexes
Different search design
```

---

# 24. Functions aren't always forbidden

Don't memorize:

> "Functions make queries bad."

That's too simplistic.

The real question is:

> **Can SQL Server still use an efficient access path for this predicate?**

Some expressions and scenarios can still be optimized effectively.

---

# 25. Implicit conversions

Another common SARGability issue is implicit conversion.

Suppose:

```text id="k5p2z9"
Column = VARCHAR
Parameter = NVARCHAR
```

SQL Server may need to convert values.

Depending on data types and precedence, the conversion can interfere with efficient index access.

---

# 26. Always match data types

If a column is:

```sql id="r7m3x1"
CustomerCode VARCHAR(20)
```

then your application should ideally send:

```text id="v9c6q4"
VARCHAR-compatible
```

rather than unnecessarily forcing:

```text id="j2n8p5"
NVARCHAR
```

The exact behavior depends on the types involved, but matching types avoids an entire class of problems.

---

# 27. Leading wildcard

Consider:

```sql id="d4x7m2"
WHERE LastName LIKE '%smith'
```

A traditional B-tree index generally cannot efficiently seek to:

```text id="a8q3n6"
anything ending in "smith"
```

because the beginning of the value is unknown.

---

# 28. Trailing wildcard

Compare:

```sql id="c1v9r5"
WHERE LastName LIKE 'smith%'
```

Now the beginning is known.

This can potentially support an index seek.

So:

```text id="y6m2k8"
'smith%'   → potentially seekable
'%smith'   → generally not a normal B-tree seek
```

---

# 29. Search requirements matter

If the application genuinely needs:

```text id="p3x7n1"
Contains arbitrary text
```

then simply adding a normal B-tree index may not solve the problem.

Depending on the requirement, consider appropriate search technologies/features.

---

# 30. Composite index and ORDER BY

Suppose:

```sql id="n5r8c2"
SELECT *
FROM Orders
WHERE CustomerId = @CustomerId
ORDER BY OrderDate;
```

An index such as:

```text id="u1m6x9"
(CustomerId, OrderDate)
```

can potentially support both filtering and ordering.

The database may not need a separate sort.

---

# 31. Avoiding a Sort

Without a suitable ordering:

```text id="g7k3p5"
Seek/filter
    ↓
Sort
    ↓
Return
```

With appropriate index order:

```text id="q9v2m8"
Index access
    ↓
Rows already ordered
    ↓
Return
```

Eliminating a large sort can be valuable.

---

# 32. GROUP BY

Indexes can sometimes help aggregation.

Suppose:

```sql id="w4c8r1"
SELECT CustomerId, COUNT(*)
FROM Orders
GROUP BY CustomerId;
```

An index beginning with:

```text id="s2n7x5"
CustomerId
```

may provide useful ordering/access characteristics.

But the optimizer still chooses the actual plan based on cost.

---

# 33. JOIN predicates

Suppose:

```sql id="j6m3q8"
SELECT ...
FROM Orders o
JOIN Customers c
  ON o.CustomerId = c.CustomerId;
```

Useful indexes may exist on:

```text id="e8p1v4"
Customers(CustomerId)
Orders(CustomerId)
```

The optimal design depends on:

```text id="b5x9k2"
Join direction
Cardinality
Filtering
Data size
Existing clustered keys
```

---

# 34. Indexes are workload-dependent

There is no universal:

> "Best index for Orders."

The correct index depends on queries such as:

```text id="r3c7m1"
Query A → CustomerId
Query B → OrderDate
Query C → Status
Query D → CustomerId + OrderDate
```

You need to understand the workload as a whole.

---

# 35. Every index has a cost

Indexes improve reads but increase:

```text id="v8n2x6"
INSERT cost
UPDATE cost
DELETE cost
Storage
Backup size
Maintenance
Statistics maintenance
```

Therefore:

> **An index is not free.**

---

# 36. Write-heavy table

Imagine:

```text id="m1q7c5"
10 million INSERTs/day
```

and you create:

```text id="k8r3v9"
20 nonclustered indexes
```

Every insert potentially has to maintain those indexes.

Your SELECT workload may improve while your write workload becomes significantly more expensive.

---

# 37. Unused indexes

An index that never helps important queries still costs resources.

Periodically review:

```text id="x4p6m2"
Usage
Writes
Storage
Business importance
```

But don't drop an index solely because it appears unused during a short observation period.

A monthly or quarterly report might rely on it.

---

# 38. Duplicate indexes

You may discover:

```text id="a7v2n8"
IX_A(CustomerId)
IX_B(CustomerId)
```

These may be redundant.

Redundant indexes can increase:

```text id="c5m9x1"
Write cost
Storage
Maintenance
```

Index consolidation can sometimes improve overall system health.

---

# 39. Nearly duplicate indexes

Consider:

```text id="z8q3r6"
IX_A(CustomerId, OrderDate)

IX_B(CustomerId, OrderDate, Status)
```

They aren't identical.

But they may overlap significantly.

Whether both are justified depends on the workload.

---

# 40. Filtered indexes

A filtered index can target a subset of rows.

For example:

```sql id="p4x7m2"
CREATE INDEX IX_Orders_Open
ON Orders(CustomerId, OrderDate)
WHERE Status = 'Open';
```

This can be powerful when:

```text id="k6n1v8"
Only a small subset of rows
is frequently queried
```

---

# 41. Why filtered indexes can be efficient

Instead of indexing:

```text id="r2m9c5"
100 million rows
```

you may index:

```text id="u7x3q1"
500,000 relevant rows
```

Potential benefits:

```text id="n5v8k4"
Smaller index
Less storage
Less maintenance
Efficient targeted access
```

But the query predicate must align appropriately with the filter.

---

# 42. Unique indexes

If a column combination is logically unique, enforcing uniqueness through an index can provide both:

```text id="j9c2m6"
Data integrity
+
Indexing
```

Example:

```text id="e1x7p4"
EmailAddress
```

if the business rule truly requires uniqueness.

---

# 43. Statistics come with indexes

SQL Server maintains statistics associated with indexes and can also maintain column statistics separately.

This matters because the optimizer needs estimates to choose plans.

Therefore indexes influence performance not only through physical access paths but also through information available to the optimizer.

---

# 44. Index maintenance isn't "rebuild everything"

Another common mistake:

> "The server is slow. Rebuild all indexes."

That is not a diagnostic strategy.

First establish:

```text id="w8m3q7"
Is fragmentation relevant?
Which indexes matter?
What is the workload?
Is maintenance causing more harm than benefit?
```

---

# 45. Fragmentation

Fragmentation can matter in some workloads, especially for large range scans.

But modern SSD-backed systems and random-access workloads may behave differently from old disk-based assumptions.

Don't treat a fragmentation percentage as a universal performance diagnosis.

---

# 46. Index design workflow

Use this process:

```text id="q2x6n9"
1. Identify important query
2. Inspect WHERE
3. Inspect JOIN
4. Inspect ORDER BY
5. Inspect GROUP BY
6. Check returned columns
7. Check current indexes
8. Compare estimated/actual plan
9. Design candidate index
10. Test read + write impact
11. Deploy carefully
12. Monitor
```

---

# 47. Example

Query:

```sql id="f7m1c4"
SELECT OrderId, OrderDate, TotalAmount
FROM Orders
WHERE CustomerId = @CustomerId
  AND OrderDate >= @StartDate
ORDER BY OrderDate;
```

Candidate:

```text id="s8q3v6"
KEY:
(CustomerId, OrderDate)

INCLUDE:
(OrderId, TotalAmount)
```

Why?

```text id="n4p9x2"
CustomerId
→ equality filter

OrderDate
→ range + ordering

OrderId / TotalAmount
→ output only
```

This is a strong starting hypothesis.

---

# 48. But test it

Check:

```text id="x6r2m8"
Logical reads
CPU
Duration
Execution plan
Insert/update cost
Index size
```

Don't stop because the plan says:

```text id="c9v5k1"
Index Seek
```

Measure the actual workload.

---

# 49. The "seek" trap

Suppose before:

```text id="t1m7q4"
Table Scan
5 sec
```

After adding an index:

```text id="p8x3n6"
Index Seek
4 sec
```

You improved the access method on paper.

But the query is still slow.

Why?

Maybe:

```text id="z2r6c9"
Millions of lookups
```

or:

```text id="m5k1v8"
Poor cardinality estimate
```

or:

```text id="q7x4n3"
Blocking
```

A seek alone isn't proof of success.

---

# 50. The "scan" trap

Suppose after tuning:

```text id="h8c2m5"
Index Scan
80 ms
```

Don't complain:

> "It's scanning!"

If the index scan reads a small, appropriate structure and returns results quickly, it may be exactly what the optimizer should do.

Performance matters more than operator labels.

---

# 51. SARGability checklist

When examining a WHERE clause, ask:

```text id="r4n8x1"
□ Is a function applied to the column?
□ Is there an implicit conversion?
□ Is there a leading wildcard?
□ Is the predicate written as a useful range?
□ Do data types match?
□ Does the index key align with the predicate?
```

---

# 52. Index key checklist

For a composite index:

```text id="u5m2k7"
□ What are the equality predicates?
□ What are the range predicates?
□ What columns support ordering?
□ What columns support joins?
□ What output columns need coverage?
□ How selective are the predicates?
```

---

# 53. Index cost checklist

Before adding an index:

```text id="j3v9p6"
□ How often is the query executed?
□ How important is it?
□ How much will the index cost writes?
□ How much storage?
□ Does another index already cover it?
□ Could an existing index be modified instead?
```

---

# 54. Interview question

### "What is SARGability?"

Strong answer:

> "A SARGable predicate is written in a way that allows the optimizer to efficiently use an index for searching or range navigation. Applying functions to indexed columns, incompatible data types, and certain expression patterns can prevent efficient seeks."

---

# 55. Interview question

### "What is a covering index?"

Strong answer:

> "A covering index contains all columns needed by a query, either as key columns or included columns, allowing SQL Server to satisfy the query without additional lookups to the base table or clustered index."

---

# 56. Interview question

### "How do you choose column order?"

Strong answer:

> "I start from the actual workload and consider equality predicates, range predicates, joins, ordering, grouping, selectivity, and coverage. Equality predicates often precede range predicates, but the final order must be validated against the execution plans and workload."

---

# 57. Interview question

### "Are scans bad?"

Strong answer:

> "No. A scan can be the optimal operation when a large percentage of rows is required or when scanning a relatively small structure is cheaper than repeated lookups. I evaluate the cost and actual work rather than judging the operator name."

---

# 58. Expert mental model

Think of index design as:

```text id="k7m2x4"
QUERY
  ↓
What rows?
  ↓
WHERE / JOIN
  ↓
How are they ordered?
  ↓
ORDER BY / GROUP BY
  ↓
What data must be returned?
  ↓
Coverage
  ↓
What does the workload cost?
  ↓
Index candidate
  ↓
Measure
```

---

# 59. Lesson 145 takeaway

The most important ideas today are:

```text id="c3p8n5"
SARGability
    ↓
Can SQL Server navigate efficiently?

Key order
    ↓
Does the index match filtering/join/order?

Coverage
    ↓
Can we avoid lookups?

Selectivity
    ↓
How much data qualifies?

Index cost
    ↓
What happens to writes and storage?
```

And remember:

> **The best index is not the index that makes the execution plan look impressive. It's the index that improves the important workload at an acceptable total system cost.**

---
**Next:** **Lesson 146**
**Final:** **Lesson 150**
