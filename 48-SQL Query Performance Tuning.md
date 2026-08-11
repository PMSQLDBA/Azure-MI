# Lesson 80 — SQL Query Performance Tuning in Azure SQL Managed Instance

Now we're moving from **"How do I diagnose slowness?"** to **"How do I actually fix a slow SQL query?"**

This is core SQL Server DBA knowledge and transfers very well to Azure SQL Managed Instance.

---

# 1. The performance-tuning flow

When you receive a slow query, don't immediately create an index.

Use this sequence:

```text
Slow query
   ↓
Reproduce / measure
   ↓
Execution plan
   ↓
Waits + runtime statistics
   ↓
Identify bottleneck
   ↓
Check indexes/statistics
   ↓
Check query design
   ↓
Make ONE meaningful change
   ↓
Test again
   ↓
Compare before vs after
```

The most important principle:

> **Measure → change → measure again.**

---

# 2. Example: 8-second query

Suppose the application runs:

```sql
SELECT
    OrderID,
    CustomerID,
    OrderDate,
    TotalAmount
FROM dbo.Orders
WHERE CustomerID = 5001;
```

The application reports:

```text
Average duration = 8 seconds
```

Business expectation:

```text
< 1 second
```

Our job is to determine why.

---

# 3. First question: how much data?

Suppose:

```text
Orders = 100 million rows
```

and:

```text
CustomerID = 5001
```

has only:

```text
2,500 orders
```

You ideally want SQL Server to locate those 2,500 rows efficiently rather than examine 100 million rows.

---

# 4. Execution plan

You inspect the execution plan.

Suppose you see:

```text
Table Scan
     ↓
100,000,000 rows
     ↓
Filter CustomerID = 5001
     ↓
2,500 rows
```

That's a huge clue.

SQL Server is effectively doing:

> "Look through everything and find the matching rows."

---

# 5. Why is it scanning?

One likely reason:

```text
No suitable index on CustomerID
```

You might consider:

```sql
CREATE INDEX IX_Orders_CustomerID
ON dbo.Orders(CustomerID);
```

Now SQL Server can potentially navigate directly to rows for that customer.

---

# 6. Seek vs scan

Think of a seek like using an index in a book.

You want:

```text
CustomerID = 5001
```

Instead of:

```text
Read every page
```

you go:

```text
Index
 ↓
5001
 ↓
Relevant rows
```

That's an index seek.

---

# 7. But a seek isn't automatically better

This is important.

Suppose your query asks for:

```text
90 million of 100 million rows
```

An index seek may not be efficient.

A scan could actually be cheaper.

Therefore:

> **Don't optimize for "seek"; optimize for the workload.**

---

# 8. Key lookup

Now suppose you create:

```sql
CREATE INDEX IX_Orders_CustomerID
ON dbo.Orders(CustomerID);
```

The query needs:

```text
OrderID
CustomerID
OrderDate
TotalAmount
```

The index contains:

```text
CustomerID
```

but not necessarily all requested columns.

SQL Server might do:

```text
Index Seek
    ↓
Find CustomerID
    ↓
Key Lookup
    ↓
Fetch remaining columns
```

---

# 9. Why key lookups can become expensive

Suppose:

```text
2,500 matching rows
```

SQL Server might perform thousands of lookups into the base table.

Conceptually:

```text
Index Seek
   |
   +-- Lookup 1
   +-- Lookup 2
   +-- Lookup 3
   +-- ...
   +-- Lookup 2500
```

That can become expensive.

---

# 10. Covering index

You can potentially create a covering index:

```sql
CREATE INDEX IX_Orders_CustomerID
ON dbo.Orders(CustomerID)
INCLUDE
(
    OrderDate,
    TotalAmount
);
```

Depending on the query and table design, the index can provide the needed columns without repeatedly going back to the base table.

`OrderID` is commonly already available as the clustered key if the table is clustered on `OrderID`, but this depends on the actual schema.

---

# 11. What does INCLUDE mean?

The key columns determine the index's search/order structure.

Included columns are additional payload stored with the index to help cover queries.

Think:

```text
Index key:
CustomerID

Included:
OrderDate
TotalAmount
```

The included columns aren't used the same way as key columns for ordering/searching.

---

# 12. Don't create giant covering indexes

You might think:

> "I'll include every column."

Bad idea.

A massive index increases:

```text
Storage
INSERT cost
UPDATE cost
DELETE cost
Maintenance
Backup/storage footprint
```

Indexes are not free.

---

# 13. Index design is a trade-off

You are balancing:

```text
READ PERFORMANCE
       ↕
WRITE COST
       ↕
STORAGE
       ↕
MAINTENANCE
```

An index that makes one SELECT fast may make a high-volume INSERT workload slower.

---

# 14. Real production example

Suppose:

```text
Orders
100 million rows
```

Workload:

```text
SELECT = 10 million/day
INSERT = 5 million/day
UPDATE = 20 million/day
```

Adding 20 indexes isn't automatically a good idea.

Every write may need to maintain those indexes.

You need to optimize for the **whole workload**, not one query.

---

# 15. SARGability

This is one of the most important SQL tuning concepts.

A predicate is generally considered more index-friendly when SQL Server can use the column directly to seek/filter efficiently.

Bad example:

```sql
WHERE YEAR(OrderDate) = 2026
```

You are applying a function to the column.

A potentially more index-friendly form is:

```sql
WHERE OrderDate >= '20260101'
  AND OrderDate <  '20270101';
```

Now SQL Server can potentially use an index on `OrderDate` more effectively.

---

# 16. Another example

Less index-friendly:

```sql
WHERE LEFT(LastName, 3) = 'Cha'
```

Potentially better:

```sql
WHERE LastName >= 'Cha'
  AND LastName < 'Chb';
```

But be careful: string range logic depends on collation and desired semantics.

The broader lesson is:

> **Avoid unnecessarily wrapping indexed columns in functions in predicates.**

---

# 17. Implicit conversions

Another common performance issue:

```sql
WHERE CustomerID = @CustomerID
```

Suppose:

```text
CustomerID = INT
@CustomerID = NVARCHAR
```

SQL Server may need to perform conversions.

Depending on data types and precedence, the conversion can affect index usage.

Always try to match parameter and column data types appropriately.

---

# 18. `SELECT *`

This:

```sql
SELECT *
FROM Orders
WHERE CustomerID = 5001;
```

can be problematic.

You're requesting every column.

Maybe the application only needs:

```text
OrderID
OrderDate
TotalAmount
```

Prefer:

```sql
SELECT
    OrderID,
    OrderDate,
    TotalAmount
FROM Orders
WHERE CustomerID = 5001;
```

Benefits can include:

```text
Less data
Less I/O
Smaller result set
Better opportunity for covering indexes
```

---

# 19. Statistics

Indexes aren't the whole story.

SQL Server needs statistics to estimate:

```text
How many rows will this query return?
```

For example:

```text
CustomerID = 5001
```

Actual:

```text
2,500 rows
```

Estimated:

```text
2 rows
```

That's a huge estimation error.

Bad estimates can lead to poor plans.

---

# 20. Cardinality estimation

Think:

```text
Query optimizer
      ↓
Statistics
      ↓
Estimate rows
      ↓
Choose plan
```

If the estimate is badly wrong:

```text
Wrong estimate
      ↓
Wrong operator choices
      ↓
Poor execution plan
      ↓
Slow query
```

---

# 21. Statistics become especially important after data changes

Suppose your table originally contains:

```text
1 million rows
```

Then six months later:

```text
500 million rows
```

Data distribution may have changed substantially.

Old statistics may no longer represent the workload accurately.

SQL Server has mechanisms for maintaining statistics, but DBAs still need to monitor situations where statistics become stale or estimation quality degrades.

---

# 22. Parameter sensitivity

Consider:

```sql
CREATE PROCEDURE dbo.GetOrders
    @CustomerID INT
AS
BEGIN
    SELECT *
    FROM dbo.Orders
    WHERE CustomerID = @CustomerID;
END;
```

Customer A:

```text
CustomerID = 100
Rows = 5
```

Customer B:

```text
CustomerID = 200
Rows = 10 million
```

One plan may not be ideal for both.

---

# 23. Why this matters

Suppose SQL Server compiles the procedure using:

```text
CustomerID = 100
```

and chooses a plan optimized for very few rows.

Later:

```text
CustomerID = 200
```

returns 10 million rows.

The previously compiled plan may perform poorly for that parameter.

This is broadly referred to as **parameter sensitivity**; the historical term "parameter sniffing" is commonly used too.

---

# 24. Query Store helps here

Query Store can show:

```text
Query
 ↓
Multiple plans
 ↓
Different runtime performance
```

You might discover:

```text
Plan A
Average = 50 ms

Plan B
Average = 12 sec
```

Now you know there is a plan/performance difference worth investigating.

---

# 25. Plan regression

Imagine:

```text
Monday
Query = 100 ms
```

Then:

```text
Tuesday
Query = 10 sec
```

No application code changed.

Query Store shows:

```text
Plan 1 → fast
Plan 2 → slow
```

This is a classic plan regression investigation.

---

# 26. What caused the new plan?

Possible factors include:

```text
Statistics changes
Data distribution
Index changes
Schema changes
Query changes
Parameter behavior
Engine/compatibility changes
```

Don't immediately force the old plan without understanding why the optimizer changed.

---

# 27. Plan forcing

Query Store can support plan forcing in appropriate circumstances.

Conceptually:

```text
Bad plan
   ↓
Identify known-good plan
   ↓
Force plan
   ↓
Stabilize workload
```

But plan forcing should be treated as a controlled mitigation, not a substitute for root-cause analysis.

---

# 28. `EXISTS` vs `IN`

Consider:

```sql
SELECT *
FROM Customers c
WHERE c.CustomerID IN
(
    SELECT CustomerID
    FROM Orders
);
```

You could potentially write:

```sql
SELECT *
FROM Customers c
WHERE EXISTS
(
    SELECT 1
    FROM Orders o
    WHERE o.CustomerID = c.CustomerID
);
```

Which is faster?

There is no universal answer.

The optimizer can transform logically equivalent queries into similar plans.

So:

> **Don't use a blanket rule like "`EXISTS` is always faster than `IN`."**

Compare actual execution plans and runtime.

---

# 29. JOIN performance

Suppose:

```sql
SELECT
    c.CustomerName,
    o.OrderDate,
    o.TotalAmount
FROM Customers c
JOIN Orders o
    ON c.CustomerID = o.CustomerID
WHERE c.CustomerID = 5001;
```

Potentially useful indexes could exist on:

```text
Customers.CustomerID
Orders.CustomerID
```

But the correct index design depends on:

```text
Join pattern
Filtering
Selected columns
Data distribution
Existing indexes
Workload
```

---

# 30. Join algorithms

SQL Server can use different physical join strategies, including:

```text
Nested Loops
Hash Match
Merge Join
```

### Nested Loops

Often useful when one input is relatively small and the other has an efficient access path.

```text
Small input
   ↓
Repeated lookup
   ↓
Large input
```

### Hash Join

Often useful for larger unsorted inputs.

```text
Build hash
   +
Probe hash
```

### Merge Join

Works well when both inputs are appropriately ordered.

The optimizer chooses based on estimates and cost.

---

# 31. Why estimates matter

Suppose SQL Server estimates:

```text
100 rows
```

but actual is:

```text
10 million rows
```

It might choose a nested loops plan that is excellent for 100 rows but terrible for 10 million.

So when you see:

```text
Estimated rows ≠ Actual rows
```

investigate.

---

# 32. Temp tables vs CTEs

A common interview question:

> "Are temp tables faster than CTEs?"

There is no universal answer.

A CTE is primarily a query-expression construct.

A temporary table physically stores intermediate results in `tempdb`.

They have different behavior.

Use a temp table when you need things such as:

```text
Materialize intermediate results
Reuse intermediate data
Create indexes on intermediate data
Break a complex process into stages
```

Use a CTE when it makes the query simpler and materialization isn't necessary.

---

# 33. TempDB

TempDB can be involved in:

```text
Sorting
Hash operations
Temporary tables
Table variables
Versioning
Intermediate operations
```

Heavy workloads can therefore put pressure on TempDB.

In SQL Managed Instance, TempDB is managed as part of the service, but DBAs still need to understand workloads that generate substantial TempDB activity.

---

# 34. Example — expensive sort

Query:

```sql
SELECT *
FROM Orders
ORDER BY OrderDate DESC;
```

If the query must sort a very large result set, it may require significant memory and potentially TempDB work.

You need to determine:

```text
How many rows?
Does an appropriate index exist?
Does the application really need all rows?
Can pagination be used?
```

---

# 35. Pagination

Bad:

```sql
SELECT *
FROM Orders
ORDER BY OrderDate DESC;
```

returning millions of rows to the application.

Better application design might use pagination:

```text
Page 1 → 50 rows
Page 2 → 50 rows
Page 3 → 50 rows
```

For large datasets, **keyset/seek pagination** can sometimes be much more efficient than very large `OFFSET` values.

---

# 36. Real tuning exercise

Let's return to:

```sql
SELECT
    OrderID,
    CustomerID,
    OrderDate,
    TotalAmount
FROM dbo.Orders
WHERE CustomerID = 5001;
```

Initial performance:

```text
8 seconds
```

---

# 37. Step 1 — Execution plan

We see:

```text
Table Scan
```

and:

```text
100 million rows examined
```

Potential issue:

```text
No suitable index
```

---

# 38. Step 2 — Create candidate index

For example:

```sql
CREATE INDEX IX_Orders_CustomerID
ON dbo.Orders(CustomerID)
INCLUDE (OrderDate, TotalAmount);
```

Now test again.

---

# 39. Step 3 — Compare

Before:

```text
Duration = 8 sec
Logical reads = very high
```

After:

```text
Duration = 200 ms
Logical reads = dramatically lower
```

That's a successful tuning change.

But don't stop there.

---

# 40. Step 4 — Check write workload

Ask:

```text
How often is Orders inserted?
How often updated?
How large is the index?
Does it increase maintenance cost?
```

If the table receives millions of writes per hour, the index has a cost.

You need to verify the net benefit.

---

# 41. Step 5 — Production validation

Don't just test:

```text
Query = 200 ms
```

Also check:

```text
CPU
I/O
Blocking
Writes
Other queries
Index size
Maintenance duration
```

A "faster query" that severely damages writes is not necessarily a successful optimization.

---

# 42. The tuning triangle

Always consider:

```text
              QUERY
              /   \
             /     \
        INDEX ----- DATA
             \     /
              WORKLOAD
```

Query performance depends on more than SQL syntax.

---

# 43. Another classic issue — functions

Suppose:

```sql
WHERE CONVERT(date, OrderDate) = '2026-08-10'
```

Potentially better:

```sql
WHERE OrderDate >= '20260810'
  AND OrderDate <  '20260811';
```

Now an index on:

```text
OrderDate
```

can potentially be used more effectively.

This is a classic SARGability improvement.

---

# 44. Another classic issue — leading wildcard

Consider:

```sql
WHERE LastName LIKE '%smith%'
```

A traditional B-tree index generally can't efficiently seek to arbitrary text preceded by `%`.

But:

```sql
WHERE LastName LIKE 'smith%'
```

has a prefix and may be able to use an index more effectively.

For true substring search requirements, consider whether full-text search or another search architecture is more appropriate.

---

# 45. Another classic issue — implicit conversion

Suppose:

```text
Column:
VARCHAR(20)

Parameter:
NVARCHAR(20)
```

Depending on data type precedence and expression shape, SQL Server may convert one side.

That can affect index usage and CPU.

Make data types consistent.

---

# 46. Don't trust estimated cost percentages blindly

Execution plans may show:

```text
Operator A = 80%
Operator B = 20%
```

Those percentages are optimizer estimates.

They don't mean:

> "Operator A literally consumed 80% of the CPU."

Use actual runtime statistics and other evidence.

---

# 47. Actual vs estimated execution plan

### Estimated

```text
What SQL Server expects
```

### Actual

```text
What actually happened
```

Comparing them can expose estimation problems.

Look for:

```text
Estimated rows
Actual rows
Join choice
Memory grant
Spills
Operators
```

---

# 48. Memory grants

Some queries request memory for:

```text
Sort
Hash
Other workspace operations
```

If the estimate is wrong, the memory grant can be inappropriate.

Too little:

```text
Spill
 ↓
TempDB activity
 ↓
Slower query
```

Too much:

```text
Memory reserved unnecessarily
 ↓
Other workloads affected
```

---

# 49. Query spills

A sort or hash operation may spill to TempDB when it doesn't get enough memory.

In an actual execution plan, spill warnings can be an important clue.

Potential causes:

```text
Bad cardinality estimate
Large operation
Insufficient memory grant
Data distribution
```

Don't blindly "add memory" before understanding the underlying problem.

---

# 50. Query Store workflow

For a slow production query:

```text
Query Store
   ↓
Find query
   ↓
Review duration/CPU
   ↓
Review plans
   ↓
Compare historical performance
   ↓
Identify regression
   ↓
Investigate root cause
   ↓
Tune or stabilize
```

This is one of the most valuable workflows to master.

---

# 51. Azure SQL MI-specific thinking

Remember that SQL Managed Instance is a managed service.

You still perform:

```text
Query tuning
Index design
Statistics analysis
Blocking investigation
Execution-plan analysis
Query Store analysis
```

But you don't manage the underlying physical server in the same way you would on-premises.

You focus more on:

```text
Workload
Configuration exposed by the service
Azure resource capacity
Networking
Managed-service behavior
```

---

# 52. When should you scale SQL MI?

Consider scaling when:

```text
Resource pressure is sustained
AND
queries/workload are reasonably optimized
AND
capacity is genuinely insufficient
```

Example:

```text
CPU:
90–95%
for hours every day

Top queries:
already optimized

Workload:
growing
```

Scaling becomes reasonable.

---

# 53. When should you NOT scale first?

Example:

```text
CPU jumped from 40% → 98%
```

because:

```text
One query
   ↓
Bad plan
   ↓
Huge scan
```

Tune the query first.

Otherwise:

```text
Bigger MI
   ↓
Bad query still bad
```

You may simply spend more money to run inefficient SQL.

---

# 54. Interview question

### "A query takes 10 seconds. What do you do?"

Strong answer:

> "First I'd reproduce or measure the query and capture its actual execution plan and runtime statistics. I'd look at waits, logical reads, CPU, row estimates versus actual rows, joins, scans, lookups, spills, and memory grants. Then I'd review indexes and statistics and check Query Store for historical behavior or plan regression. I'd make a targeted change and compare the before-and-after metrics."

That's a strong DBA answer.

---

# 55. Interview question

### "What is a covering index?"

Answer:

> "A covering index contains the columns needed by a query, either as key columns or included columns, so SQL Server can satisfy the query from the index without needing additional lookups to the base table."

---

# 56. Interview question

### "Does an index always improve performance?"

Absolutely not.

An index can improve reads while increasing:

```text
INSERT
UPDATE
DELETE
storage
maintenance
```

So:

> **Index for workload, not for individual queries in isolation.**

---

# 57. Interview question

### "What is SARGability?"

A good interview definition:

> **SARGability describes whether a search predicate is written in a form that allows the optimizer to efficiently use an index/search argument.**

Example:

Less friendly:

```sql
WHERE YEAR(OrderDate) = 2026
```

More index-friendly:

```sql
WHERE OrderDate >= '20260101'
AND OrderDate < '20270101'
```

---

# 58. Your performance-tuning checklist

When a query is slow:

```text
☐ Reproduce
☐ Capture actual plan
☐ Check Query Store
☐ Check duration
☐ Check CPU
☐ Check logical reads
☐ Check waits
☐ Check blocking
☐ Compare estimated vs actual rows
☐ Check scans
☐ Check lookups
☐ Check joins
☐ Check indexes
☐ Check statistics
☐ Check implicit conversions
☐ Check SARGability
☐ Check spills
☐ Check memory grants
☐ Check recent changes
☐ Test one change
☐ Compare before/after
```

---

# 59. The golden rule

Don't say:

> "This query is slow because it has a table scan."

Instead say:

> **"The query is scanning 100 million rows to return 2,500 rows, resulting in excessive logical reads and CPU. The execution plan and workload suggest a selective access path on CustomerID may be beneficial, so I'll test an appropriate index and measure the before/after impact."**

That's the difference between **guessing** and **performance engineering**.

---

# 60. What you've learned so far

You now have the foundation for:

```text
SQL Server
     ↓
Azure SQL Managed Instance
     ↓
Architecture
     ↓
Migration
     ↓
SQL Agent
     ↓
Monitoring
     ↓
Troubleshooting
     ↓
Query tuning
```

The next important step is **HA/DR**, because production DBAs need to answer:

> **"What happens when the primary database or SQL Managed Instance becomes unavailable?"**

---

## Next — Lesson 81: Azure SQL Managed Instance High Availability & Disaster Recovery

We'll cover:

* HA vs DR
* Availability Groups concepts
* automatic failover
* zone redundancy
* backups
* point-in-time restore
* RPO
* RTO
* failover groups
* geo-replication concepts
* regional disaster scenarios
* planned vs unplanned failover
* application connection behavior
* real production DR architecture
* and the classic interview question:

> **"Your primary SQL MI region is completely unavailable. Walk me through exactly what you do."**
