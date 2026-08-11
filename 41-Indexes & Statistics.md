# Lesson 73 — Indexes & Statistics in Azure SQL Managed Instance

This is one of the most important SQL Server DBA topics because **indexes can make a query dramatically faster—but poorly designed indexes can make the whole system worse.**

We'll learn this from a production perspective rather than just memorizing definitions.

---

# 1. Start with the problem

Imagine this table:

```text
Orders
--------------------------------
OrderID
CustomerID
OrderDate
Status
TotalAmount
SalesPersonID
ShippingAddress
```

And it contains:

```text
100 million rows
```

Your application runs:

```sql
SELECT OrderID, OrderDate, TotalAmount
FROM Orders
WHERE CustomerID = 1001;
```

Without a useful index, SQL Server may need to inspect a huge amount of data.

Conceptually:

```text
100,000,000 rows
        ↓
       Scan
        ↓
Find CustomerID = 1001
```

That's expensive.

---

# 2. What an index does

Think of an index like a book's index.

Without an index:

```text
Read entire book
      ↓
Find "Azure SQL"
```

With an index:

```text
Index
  ↓
Azure SQL → page 742
```

SQL Server can navigate directly toward relevant data.

---

# 3. Clustered index

A clustered index determines how the table's data is organized around its clustered key.

A common example:

```sql
CREATE CLUSTERED INDEX IX_Orders_OrderID
ON Orders(OrderID);
```

Conceptually:

```text
Orders
  |
Clustered Index
  |
OrderID
  |
Rows
```

There can be **one clustered index per table**.

Why?

Because the table's data can have only one clustered ordering structure.

---

# 4. Nonclustered index

A nonclustered index is a separate structure containing indexed keys and pointers/locators to the underlying row data.

Example:

```sql
CREATE INDEX IX_Orders_CustomerID
ON Orders(CustomerID);
```

Now SQL Server can quickly locate rows for:

```sql
WHERE CustomerID = 1001
```

---

# 5. Real production scenario

Suppose:

```text
Orders = 100 million rows
```

Application query:

```sql
SELECT *
FROM Orders
WHERE CustomerID = @CustomerID;
```

Without an appropriate index:

```text
Large scan
↓
High IO
↓
High duration
```

After creating:

```text
IX_Orders_CustomerID
```

you might get:

```text
Index Seek
↓
Relevant rows
```

Performance can improve dramatically.

But there's a catch.

---

# 6. Indexes aren't free

Every index has a maintenance cost.

Suppose:

```text
Orders
  |
  +-- Clustered index
  +-- IX_CustomerID
  +-- IX_Status
  +-- IX_OrderDate
  +-- IX_SalesPerson
  +-- IX_TotalAmount
```

When you insert a row:

```text
INSERT
  ↓
Table data
  ↓
Index 1
  ↓
Index 2
  ↓
Index 3
  ↓
Index 4
  ↓
Index 5
```

Every additional index can increase write work and storage.

---

# 7. The key DBA trade-off

Indexes improve:

```text
SELECT performance
```

but can increase:

```text
INSERT cost
UPDATE cost
DELETE cost
Storage
Maintenance
Potential write contention
```

Therefore:

> **The goal isn't maximum indexes. The goal is the right indexes.**

---

# 8. Covering index

Now consider:

```sql
SELECT OrderDate, TotalAmount
FROM Orders
WHERE CustomerID = 1001;
```

An index only on:

```text
CustomerID
```

may find the rows, but SQL Server still needs:

```text
OrderDate
TotalAmount
```

from the underlying data.

You can potentially create:

```sql
CREATE INDEX IX_Orders_CustomerID
ON Orders(CustomerID)
INCLUDE (OrderDate, TotalAmount);
```

Now the index contains the columns needed by the query.

This is a **covering index** for that query pattern.

---

# 9. Why INCLUDE matters

Think:

```text
Index key:
CustomerID

Included:
OrderDate
TotalAmount
```

The key is used for navigation/filtering.

Included columns provide additional data without becoming part of the index key.

This distinction is extremely important.

---

# 10. Composite index

Suppose the query is:

```sql
SELECT *
FROM Orders
WHERE CustomerID = 1001
  AND Status = 'Shipped';
```

You may consider:

```sql
CREATE INDEX IX_Orders_Customer_Status
ON Orders(CustomerID, Status);
```

Now the index has two key columns.

But order matters.

---

# 11. Index key order

Compare:

```text
Index A:
(CustomerID, Status)
```

versus:

```text
Index B:
(Status, CustomerID)
```

They are not equivalent for every workload.

Suppose most queries filter by:

```sql
WHERE CustomerID = ?
```

Then:

```text
(CustomerID, Status)
```

may be a more natural leading-key choice.

But the correct order depends on the actual query workload, selectivity, predicates, joins, sorting, and other requirements.

---

# 12. The leftmost-key concept

Suppose:

```text
Index:
(CustomerID, Status, OrderDate)
```

A query filtering by:

```sql
WHERE CustomerID = 1001
```

can potentially use the leading key.

A query filtering by:

```sql
WHERE Status = 'Shipped'
```

doesn't get the same direct benefit from the leading `CustomerID` key.

Think:

```text
CustomerID
    ↓
Status
    ↓
OrderDate
```

The beginning of the index matters.

---

# 13. SARGability

This is a major interview topic.

A predicate is generally more **SARGable** when SQL Server can efficiently use an index to search for qualifying values.

Example:

```sql
WHERE CustomerID = 1001
```

is straightforward.

But consider:

```sql
WHERE YEAR(OrderDate) = 2026
```

The function is applied to the column.

That can make efficient index use more difficult.

A range predicate is often preferable:

```sql
WHERE OrderDate >= '20260101'
  AND OrderDate <  '20270101'
```

The exact best form depends on data types and business requirements, but the principle is:

> **Avoid unnecessarily transforming the indexed column in the predicate.**

---

# 14. Another SARGability example

Potentially problematic:

```sql
WHERE UPPER(CustomerName) = 'CHINNU'
```

If the database/index collation already provides the needed case behavior, applying `UPPER()` may be unnecessary.

Better:

```sql
WHERE CustomerName = 'Chinnu'
```

when semantically appropriate.

---

# 15. Implicit conversions

Another common production problem.

Suppose:

```text
CustomerID = INT
```

but the application sends:

```text
NVARCHAR
```

Now SQL Server may need to perform an implicit conversion.

That can interfere with efficient index usage depending on the data types and conversion direction.

This is why application and database data types should align.

---

# 16. Filtered indexes

Suppose only a small portion of orders are active:

```text
100 million orders
↓
2 million active
```

Your workload frequently asks:

```sql
WHERE Status = 'Active'
```

A filtered index can potentially be useful:

```sql
CREATE INDEX IX_Orders_Active
ON Orders(CustomerID, OrderDate)
WHERE Status = 'Active';
```

The idea:

```text
Instead of indexing everything
        ↓
Index only qualifying rows
```

This can reduce index size and maintenance compared with a full-table equivalent in appropriate scenarios.

---

# 17. Missing-index recommendations

SQL Server can suggest indexes based on observed query patterns.

These recommendations can be useful.

But:

> **Never blindly create every missing-index recommendation.**

Why?

Suppose SQL Server recommends:

```text
Index A
Index B
Index C
Index D
```

You might already have:

```text
Index E
```

that partially or completely overlaps them.

Blindly adding everything creates index explosion.

---

# 18. Duplicate indexes

Suppose you have:

```text
Index 1:
(CustomerID)

Index 2:
(CustomerID, Status)

Index 3:
(CustomerID, Status, OrderDate)
```

These may have overlapping purposes.

You need to understand the workload before removing or consolidating anything.

Never drop an index simply because another index starts with the same column.

---

# 19. Index usage analysis

For every index, ask:

```text
Reads?
Writes?
Size?
Purpose?
Which queries depend on it?
```

Imagine:

```text
Index A
Reads: 50 million
Writes: 2 million
```

Likely valuable.

But:

```text
Index B
Reads: 10
Writes: 100 million
```

That's a candidate for investigation.

Not automatic deletion.

---

# 20. Production scenario — index makes SELECT faster

Before:

```text
Query duration = 8 sec
CPU = high
IO = high
```

You create a suitable index.

After:

```text
Query duration = 100 ms
CPU ↓
IO ↓
```

Great.

But now your application has heavy writes.

```text
INSERT rate = 100,000/min
```

Every insert must maintain the new index.

Potential result:

```text
SELECT performance ↑
INSERT overhead ↑
```

You need to evaluate the **overall workload**, not one query.

---

# 21. Statistics

Indexes aren't the whole story.

SQL Server also needs information about data distribution.

That's where **statistics** come in.

Think:

```text
Index
 ↓
How can I find the data?

Statistics
 ↓
How much data do I expect to find?
```

Both influence query optimization.

---

# 22. Example

Suppose:

```text
CustomerID = 1001
```

returns:

```text
5 rows
```

SQL Server might choose:

```text
Index Seek
```

But if:

```text
CustomerID = 1001
```

returns:

```text
50 million rows
```

a scan or different plan might be more efficient.

Statistics help the optimizer estimate this distribution.

---

# 23. Histograms

Statistics can contain a histogram describing data distribution for the leading statistics column.

Simplified:

```text
CustomerID
|
+--- 1-1000       → many rows
+--- 1001-2000    → few rows
+--- 2001-3000    → many rows
```

The optimizer uses this information to estimate cardinality.

You don't need to memorize the internal algorithm.

Remember:

> **Statistics describe data distribution and help the optimizer estimate row counts.**

---

# 24. Stale statistics

Suppose:

```text
Monday:
Orders = 10 million
```

Then a huge load happens:

```text
Tuesday:
Orders = 200 million
```

The data distribution has changed significantly.

If the optimizer's statistics don't adequately reflect that change:

```text
Estimated rows
      ↓
Can become inaccurate
      ↓
Bad plan possible
```

---

# 25. Statistics and plan regression

This connects directly to our previous lesson.

```text
Data changes
    ↓
Statistics become less representative
    ↓
Cardinality estimate changes
    ↓
Optimizer chooses different plan
    ↓
Query becomes slower
```

This is one common route to plan regression.

---

# 26. Auto-update statistics

SQL Server can automatically update statistics under appropriate conditions.

This is helpful, but don't think:

> "Auto-update statistics means I never need to investigate statistics."

Large or unusual workloads can still produce estimation challenges.

Also remember:

```text
Auto update
≠
Perfect statistics for every query
```

---

# 27. Statistics maintenance

A DBA should understand:

```text
Are statistics current?
Are estimates accurate?
Did data distribution change?
Are important queries showing estimate errors?
```

Don't use:

> "Update all statistics every hour."

as a universal solution.

Maintenance should be workload-aware.

---

# 28. Fragmentation

Now another frequently misunderstood topic.

People often say:

> "The index is fragmented, so the query is slow."

Not necessarily.

Fragmentation can matter, but its impact depends on:

```text
Workload
Index size
Access pattern
Storage characteristics
Scan behavior
```

Don't rebuild indexes just because a percentage appears on a dashboard.

---

# 29. Reorganize vs rebuild

Conceptually:

```text
REORGANIZE
   ↓
More incremental operation

REBUILD
   ↓
Recreate the index
```

The choice depends on the index, workload, maintenance window, and operational requirements.

In Azure SQL Managed Instance, you still need to think like a SQL Server DBA even though Azure manages infrastructure.

---

# 30. The biggest misconception

> "Azure is managed, so I don't need to manage indexes."

Wrong.

Azure manages infrastructure/platform components according to the managed service model.

You still own major database-engine responsibilities such as:

```text
Query design
Index design
Statistics behavior
Schema
Transactions
Workload
Performance
Application SQL
```

Managed service ≠ managed application performance.

---

# 31. Real production case

Let's say:

```text
Orders
200 million rows
```

Query:

```sql
SELECT OrderID, OrderDate, TotalAmount
FROM Orders
WHERE CustomerID = @CustomerID
  AND Status = 'Open';
```

Current performance:

```text
5 seconds
```

---

## Step 1 — Look at the plan

You see:

```text
Large Scan
```

---

## Step 2 — Check estimates

```text
Estimated = 100 rows
Actual    = 3,000,000 rows
```

Huge mismatch.

---

## Step 3 — Check indexes

Existing:

```text
(CustomerID)
```

Potential workload:

```text
CustomerID + Status
```

---

## Step 4 — Evaluate candidate index

Potentially:

```text
(CustomerID, Status)
INCLUDE
(OrderDate, TotalAmount)
```

Now the query could potentially become covered and more selective.

---

# 32. But don't stop there

Before creating it, ask:

```text
How frequently is this query executed?
How much write activity does Orders receive?
Does a similar index already exist?
How large will the index be?
Will it help other queries?
What is the maintenance cost?
```

This is DBA thinking.

---

# 33. After creating the index

Suppose:

```text
Query:
5 sec → 80 ms
```

Excellent.

But then you notice:

```text
INSERT throughput:
100K/min → 70K/min
```

Now the index improved one workload and harmed another.

You need to determine whether the trade-off is acceptable or whether the index design can be improved.

---

# 34. Index design is workload design

Don't ask:

> "What indexes should this table have?"

Ask:

> **"What workload does this table need to support?"**

Because the answer changes with:

```text
OLTP
Reporting
ETL
Batch processing
APIs
Ad-hoc queries
```

---

# 35. OLTP vs reporting

### OLTP

Usually:

```text
Many small reads
Many writes
Low latency
```

You generally want carefully targeted indexes.

### Reporting

Potentially:

```text
Large scans
Aggregations
Joins
Sorting
```

The optimal indexing strategy can be very different.

---

# 36. Composite index example

Suppose your most frequent query is:

```sql
SELECT *
FROM Orders
WHERE CustomerID = @CustomerID
  AND OrderDate >= @StartDate
  AND OrderDate < @EndDate;
```

A potentially useful index shape could be:

```text
(CustomerID, OrderDate)
```

Why?

```text
CustomerID
   ↓
narrow to customer
   ↓
OrderDate
   ↓
range filter
```

But again, validate with the actual workload and plan.

---

# 37. Key-order thinking

A useful conceptual rule:

```text
Equality predicates
        ↓
Range predicates
        ↓
Other considerations
```

For example:

```text
WHERE CustomerID = ?
  AND OrderDate >= ?
```

often leads you to consider:

```text
(CustomerID, OrderDate)
```

But this is a design heuristic—not a universal law.

---

# 38. What should you never do?

Avoid statements like:

> "Every WHERE column needs an index."

No.

Or:

> "Every table should have 10 indexes."

No.

Or:

> "Every fragmented index should be rebuilt."

No.

Or:

> "Missing-index DMV says create it, so create it."

No.

---

# 39. A senior DBA index checklist

Before adding an index:

```text
☐ Identify the query
☐ Review execution plan
☐ Check actual vs estimated rows
☐ Check existing indexes
☐ Check Query Store
☐ Check execution frequency
☐ Check read benefit
☐ Check write cost
☐ Check storage cost
☐ Consider overlapping indexes
☐ Test
☐ Monitor after deployment
```

---

# 40. Interview question

### "What is the difference between an index and statistics?"

Strong answer:

> "An index is a physical access structure that can help SQL Server locate and retrieve data efficiently. Statistics provide information about data distribution that helps the optimizer estimate cardinality and choose an execution plan. An index can exist without solving a query if the optimizer's estimates or query shape lead to a poor plan."

---

# 41. Interview question

### "Why can too many indexes be bad?"

Answer:

> "Indexes consume storage and need maintenance during data modifications. Additional indexes can increase INSERT, UPDATE, and DELETE overhead. They can also create redundant structures and maintenance complexity. The objective is to create indexes that provide meaningful workload benefit."

---

# 42. Interview question

### "What is a covering index?"

Answer:

> "An index that contains all the columns needed by a particular query, either as key columns or included columns, allowing SQL Server to satisfy the query without needing additional lookups to the underlying table."

---

# 43. Interview question

### "Why does composite index column order matter?"

Answer:

> "Because the leading key columns determine how efficiently SQL Server can navigate the index. An index on `(CustomerID, Status)` is not equivalent to one on `(Status, CustomerID)` for all query patterns."

---

# 44. Final mental model

Remember:

```text
                QUERY
                  |
                  v
             Optimizer
                  |
       +----------+----------+
       |                     |
   Statistics              Indexes
       |                     |
 "How many?"          "How can I find it?"
       |                     |
       +----------+----------+
                  |
                  v
            Execution Plan
                  |
                  v
             Performance
```

And the golden rule:

> **Good indexing is not about creating more indexes. It is about reducing the amount of work SQL Server needs to perform while keeping write and maintenance costs under control.**

---

## Next — Lesson 74: High Availability in Azure SQL Managed Instance

Next we'll move into **HA/DR architecture**, which is critical for Azure SQL MI interviews and real production work.

We'll cover:

* HA vs DR
* Availability Zones
* automatic failover
* failover groups
* geo-replication concepts
* RPO/RTO
* planned vs unplanned failover
* regional outage scenario
* application connection behavior
* DNS/connection-string considerations
* failover testing
* what happens to active transactions
* and a complete **"primary Azure region is down" production scenario**.
