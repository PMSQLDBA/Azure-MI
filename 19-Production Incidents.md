
# Lesson 23 — Production Incident #1: High CPU

### Incident

Production Azure SQL Managed Instance:

```text
CPU                  92%
Memory               Normal
Storage              61%
IO                   Normal
Blocking              0
Deadlocks             0
```

Application latency:

```text
9 seconds
```

Query Store:

```text
Query A
Average duration:     8.4 sec
CPU:                  Very High
Executions:           12,000/min

Query B
Average duration:     150 ms

Query C
Average duration:     300 ms
```

Query A:

```sql
SELECT
    CustomerID,
    SUM(OrderAmount)
FROM Sales.Orders
WHERE OrderDate >= @StartDate
GROUP BY CustomerID;
```

Execution plan:

```text
Clustered Index Scan
        ↓
250 million rows
        ↓
Hash Match
        ↓
Sort
```

---

# 1. First diagnosis

The first conclusion should be:

> **The MI is experiencing workload-driven CPU pressure caused primarily by Query A.**

Why?

Because we have strong evidence:

```text
CPU = 92%
       ↓
Query A = dominant CPU consumer
       ↓
12,000 executions/min
       ↓
250M-row scan
       ↓
Hash + Sort
```

This is much more useful than simply saying:

> "CPU is high."

---

# 2. Would I immediately scale the MI?

### No — not as the first action.

Scaling may eventually be appropriate, but first determine whether the workload can be made more efficient.

Imagine Query A currently does:

```text
250M rows × 12,000 executions/min
```

That's potentially an enormous amount of work.

If you optimize the query and reduce the work dramatically, scaling may not be necessary.

Think:

```text
Bad query
   ↓
Optimize
   ↓
Measure
   ↓
Scale if still required
```

---

# 3. What would I investigate?

I'd investigate:

### Query behavior

```text
How frequently is it executed?
Which parameters are used?
Is it really necessary 12,000 times/minute?
```

### Execution plan

```text
Why is there a scan?
Why is there a large hash?
Why is there a sort?
Are there spills?
Are cardinality estimates accurate?
```

### Indexes

The predicate is:

```sql
WHERE OrderDate >= @StartDate
```

So I'd inspect existing indexes involving:

```text
OrderDate
CustomerID
OrderAmount
```

and determine whether an index could reduce the amount of data read.

---

# 4. Possible indexing strategy

One candidate to investigate might be:

```sql
CREATE INDEX IX_Orders_OrderDate
ON Sales.Orders(OrderDate)
INCLUDE(CustomerID, OrderAmount);
```

Why?

The query needs:

```text
Filter:
OrderDate

Output/grouping:
CustomerID

Aggregation:
OrderAmount
```

A suitable covering strategy could potentially reduce the amount of base-table data that must be accessed.

But **do not blindly create this index**.

Before deploying it, investigate:

```text
Existing indexes
Table size
Write workload
Index storage
Selectivity of OrderDate
Actual execution plan
Query frequency
```

---

# 5. There's another important question

Why is this query running:

```text
12,000 times/minute
```

That's roughly:

```text
200 executions/second
```

That's enormous.

So I would involve the application team.

Maybe the application is doing:

```text
Request
  ↓
Run aggregation
  ↓
Request
  ↓
Run aggregation
  ↓
Request
  ↓
Run aggregation
```

when it could instead:

```text
Calculate once
     ↓
Cache result
     ↓
Reuse result
```

Sometimes the best database optimization is:

> **Stop executing the query unnecessarily.**

---

# 6. Another possibility: pre-aggregation

Suppose the application constantly asks:

> "Give me total order amount per customer since yesterday."

Running this against 250 million rows repeatedly may be inefficient.

A different architecture could use:

```text
Orders
   ↓
Aggregation process
   ↓
Summary table
   ↓
Application
```

Then the application queries a much smaller dataset.

This is an architectural optimization rather than simply an index change.

---

# 7. What would I check before changing production?

I'd establish a baseline:

```text
Current CPU
Current query duration
Executions/min
IO
Waits
Logical reads
Actual execution plan
Data volume
```

Then test the proposed change.

For example:

```text
Before
CPU = 92%
Query = 8.4 sec

After index/test
CPU = 45%
Query = 300 ms
```

Now we have evidence.

---

# 8. Important senior DBA principle

Don't say:

> "The execution plan has a scan, therefore the scan is wrong."

A scan may be perfectly valid.

The real question is:

> **Is SQL Server doing an unreasonable amount of work for this workload?**

Here, the combination of:

```text
250M rows
+
12,000 executions/min
+
8.4 sec
+
92% CPU
```

makes the workload highly suspicious.

---

# Incident #2 — Blocking

Now let's make it harder.

At 2:30 PM:

```text
CPU:                 38%
IO:                  Normal
Memory:              Normal
```

But the application reports:

> **"Orders are timing out."**

You investigate:

```text
Active requests:       480
Blocked requests:      372
Deadlocks:               0
```

You find:

```text
Session 57
    |
    | Blocking
    +------------------ Session 82
    +------------------ Session 91
    +------------------ Session 104
    +------------------ Session 127
    +------------------ Session 205
    +------------------ ...
```

Session 57 has been running for:

```text
47 minutes
```

Its transaction is:

```sql
BEGIN TRANSACTION;

UPDATE Sales.Orders
SET Status = 'Processing'
WHERE OrderID BETWEEN 100000 AND 800000;

-- Application hasn't committed yet
```

---

# 9. Diagnosis

This is a classic:

> **Head blocker.**

Session 57 is holding locks.

Other sessions are waiting.

Conceptually:

```text
Session 57
   |
   | locks
   v
Orders
   ^
   |
   +---- Session 82 waiting
   +---- Session 91 waiting
   +---- Session 104 waiting
   +---- Session 127 waiting
```

The problem isn't CPU.

The problem is:

```text
Long-running transaction
        ↓
Locks held
        ↓
Blocking
        ↓
Application timeouts
```

---

# 10. Should you immediately KILL session 57?

Not automatically.

First investigate:

```text
Who owns session 57?
What application is it?
What transaction is running?
How many rows changed?
Is it still making progress?
Why hasn't it committed?
What happens if it is killed?
```

This matters because killing it may cause:

```text
ROLLBACK
```

And a large rollback can itself take time.

---

# 11. What is the real fix?

Immediate mitigation might be:

```text
Resolve/terminate problematic transaction
```

depending on the business situation.

But the permanent fix is:

```text
Why was the transaction open for 47 minutes?
```

Possible causes:

```text
Application forgot COMMIT
User left transaction open
Batch too large
Slow client processing
Poor transaction design
Network interruption
Incorrect error handling
```

---

# 12. Better transaction design

Instead of:

```text
BEGIN TRANSACTION

Update 700,000 rows

Wait for application processing

COMMIT
```

you may need a design such as:

```text
Small batch
   ↓
Commit
   ↓
Small batch
   ↓
Commit
   ↓
Small batch
```

depending on business consistency requirements.

The goal is often to reduce:

```text
Lock duration
Transaction size
Rollback impact
Blocking
```

---

# Incident #3 — Deadlock

Now:

```text
CPU: 35%
Blocking: Low
Application errors: Increasing
```

Error:

```text
Transaction (Process ID 84)
was deadlocked on lock resources.
```

You capture the deadlock graph.

It shows:

```text
Transaction A
    |
    | locks Customer
    |
    v
waits for Order


Transaction B
    |
    | locks Order
    |
    v
waits for Customer
```

So:

```text
A → Customer → waiting for Order
B → Order → waiting for Customer
```

Cycle detected.

---

# 13. Deadlock fix

The first thing I'd investigate is whether both transactions can access resources in the same order.

For example:

```text
Always:

Customer
   ↓
Order
```

instead of:

```text
Transaction A:
Customer → Order

Transaction B:
Order → Customer
```

Other fixes may include:

```text
Shorter transactions
Better indexes
Reduced locking
Application retry
Isolation-level changes where appropriate
```

---

# Incident #4 — Query regression

Yesterday:

```text
Query duration = 200 ms
```

Today:

```text
Query duration = 8 seconds
```

Infrastructure:

```text
CPU = 30%
IO = Normal
Blocking = 0
```

Query Store:

```text
Plan A
   ↓
200 ms

Plan B
   ↓
8 sec
```

This is a classic:

> **Plan regression.**

---

# 14. Investigation

Compare the plans.

Suppose:

### Plan A

```text
Index Seek
   ↓
Nested Loops
```

### Plan B

```text
Table Scan
   ↓
Hash Match
```

Then investigate:

```text
Statistics
Data distribution
Parameter sensitivity
Index changes
Schema changes
Plan compilation
Recent deployments
```

---

# 15. Parameter sensitivity scenario

Suppose:

```sql
SELECT *
FROM Sales.Orders
WHERE CustomerID = @CustomerID;
```

Customer 1:

```text
Rows = 3
```

Customer 5000:

```text
Rows = 20 million
```

One execution plan may not be ideal for both.

So you may see:

```text
Small customer
   ↓
Seek = excellent

Large customer
   ↓
Same plan = terrible
```

This requires careful investigation rather than blindly forcing an index.

---

# Incident #5 — TempDB / spill

Query:

```sql
SELECT
    CustomerID,
    SUM(OrderAmount)
FROM Sales.Orders
GROUP BY CustomerID
ORDER BY SUM(OrderAmount) DESC;
```

Execution plan shows:

```text
Hash Match
    ↓
Sort
    ↓
Spill warning
```

The operation didn't receive enough memory for its work efficiently, so part of the operation spilled to tempdb.

Potential consequences:

```text
More IO
More latency
TempDB pressure
Poor application performance
```

Investigate:

```text
Memory grant
Cardinality estimates
Statistics
Query design
Data volume
Concurrent workload
```

---

# 16. The troubleshooting hierarchy

When you get a production performance problem, use this:

```text
                    INCIDENT
                       |
                       v
              What changed?
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
       CPU             IO          Blocking
        |              |              |
        +--------------+--------------+
                       |
                       v
                     Waits
                       |
                       v
                Expensive queries
                       |
                       v
                Query Store
                       |
                       v
                Execution plan
                       |
                       v
                  Root cause
                       |
                       v
                     Fix
                       |
                       v
                   Validate
```

---

# 17. Your DBA toolkit

You should become comfortable with these areas:

### Current activity

```text
sys.dm_exec_requests
sys.dm_exec_sessions
```

### Query execution

```text
sys.dm_exec_sql_text
sys.dm_exec_query_stats
```

### Index information

```text
sys.indexes
sys.index_columns
```

### Database information

```text
sys.databases
```

And:

> **Query Store for historical workload and plan analysis.**

Don't memorize everything at once. Learn what question each tool answers.

---

# 18. The most important questions

When troubleshooting, ask:

### CPU

> Which workload is consuming CPU?

### IO

> Which query is reading/writing the most data?

### Blocking

> Who is the head blocker and why is the transaction open?

### Deadlock

> What resource ordering created the cycle?

### Query regression

> What changed between the good and bad plans?

### TempDB

> What operation is spilling and why?

### Memory

> Are memory grants appropriate?

These questions will take you much further than memorizing commands.

---

# Lesson 24 — Production DBA Incident Drill

Now we're going to simulate an actual on-call shift.

You receive **three alerts simultaneously**:

### Alert A

```text
CPU = 95%
```

### Alert B

```text
Storage = 88%
```

### Alert C

```text
300 blocked sessions
```

The application team says:

> "Everything is slow!"

You are the on-call DBA.

You **cannot investigate everything equally at the same time**.

Your first job is to prioritize.

### What would you investigate first?

Think about:

```text
A. CPU
B. Storage
C. Blocking
D. All three equally
```
