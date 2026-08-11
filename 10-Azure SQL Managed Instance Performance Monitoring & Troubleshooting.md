**Lesson 8 — Azure SQL Managed Instance Performance Monitoring & Troubleshooting**.

This is where your SQL Server DBA skills become especially valuable.

# Lesson 8 — "SQL Managed Instance Is Slow" 🔎

Imagine the production support ticket:

> **"The application is extremely slow. SQL Managed Instance must be overloaded."**

Your job isn't to immediately increase vCores.

Your job is to answer:

> **What exactly is slow, and why?**

---

# 1. Start with the four big questions

When SQL MI is reported as slow, investigate:

```text
                    PERFORMANCE
                         |
          +--------------+--------------+
          |              |              |
         CPU            Memory          IO
          |              |              |
          +--------------+--------------+
                         |
                       Queries
                         |
                 Blocking / Locks
                         |
                       Waits
```

And don't forget:

```text
Application
    |
    v
Network latency
    |
    v
SQL MI
```

The database might not actually be the bottleneck.

---

# 2. First question: Is CPU high?

Suppose:

```text
CPU = 95%
```

Now you have evidence that compute is under pressure.

But don't immediately conclude:

> "Add more vCores."

First ask:

```text
What is consuming CPU?
```

Possibilities:

```text
Bad query
Missing index
Large scan
Poor execution plan
Too many concurrent queries
Parallelism
Reporting workload
Application regression
```

---

# 3. Example: bad query

Suppose the application executes:

```sql id="ydv4xw"
SELECT *
FROM Orders
WHERE CustomerName = 'John';
```

But `CustomerName` isn't indexed.

SQL Server may need to scan:

```text id="7d84tz"
Orders
 |
 +-- Row 1
 +-- Row 2
 +-- Row 3
 ...
 +-- Row 10,000,000
```

That's potentially a lot of work.

You might see:

```text id="w6j7bq"
CPU ↑
IO ↑
Query duration ↑
```

Adding compute might temporarily hide the problem, but fixing the query/index may be the real solution.

---

# 4. CPU troubleshooting mindset

If CPU is high:

```text id="3n3r8p"
High CPU
   |
   v
Find top CPU queries
   |
   v
Inspect execution plans
   |
   v
Check indexes/statistics
   |
   v
Check concurrency
   |
   v
Determine root cause
```

Don't jump straight to:

```text
Increase vCores
```

---

# 5. Second question: Is memory under pressure?

SQL Server is designed to use memory aggressively for:

```text id="xhj5q5"
Data pages
Execution plans
Query workspace
Caching
```

High memory utilization by itself isn't automatically a problem.

The question is:

> **Is there memory pressure affecting workload performance?**

Look for symptoms such as:

```text id="bj9gde"
Queries waiting for memory
Excessive physical reads
Memory grant pressure
Queries spilling to tempdb
```

---

# 6. Memory grant scenario

Imagine:

```sql id="x4lqxf"
SELECT CustomerID, SUM(OrderAmount)
FROM Orders
GROUP BY CustomerID;
```

The optimizer may request a memory grant for operations such as:

```text id="8n6yko"
Sort
Hash join
Hash aggregate
```

If the query requests too much memory:

```text id="iq8m6a"
Query A
   |
   +---- Large memory grant
   |
   v
Other queries wait
```

Now a single poorly designed query can affect other workloads.

---

# 7. TempDB spills

Suppose the query expects enough memory for a sort but doesn't get it.

Then:

```text id="i9sh70"
Query
  ↓
Sort
  ↓
Memory insufficient
  ↓
TempDB
  ↓
Disk IO
```

This can create:

```text id="40rj5b"
Query slow
IO increased
TempDB activity increased
```

So when investigating performance, **tempdb matters**.

---

# 8. Third question: Is storage/IO the bottleneck?

Suppose:

```text id="z7i4kw"
CPU = 25%
Memory = Normal
IO latency = High
```

The database can still be slow.

For example:

```text id="b8a8k2"
Query
  ↓
Needs 1 million pages
  ↓
Storage reads
  ↓
High IO latency
  ↓
Query takes longer
```

This is why:

> **Low CPU does not mean low database workload.**

---

# 9. Query waits

Now we get to one of the most useful SQL Server concepts:

# Wait statistics

Instead of asking:

> "What is SQL Server doing?"

ask:

> **"What is SQL Server waiting for?"**

Conceptually:

```text id="x3qyt4"
Query
  |
  +-- CPU
  |
  +-- IO
  |
  +-- Lock
  |
  +-- Memory
  |
  +-- Network
```

A query often spends significant time **waiting** rather than actively using CPU.

---

# 10. Example: PAGEIOLATCH

Suppose you see a lot of IO-related waits.

Conceptually:

```text id="5t6v2f"
Query
  ↓
Needs data page
  ↓
Page isn't available in memory
  ↓
Storage read
  ↓
WAIT
```

This can indicate storage/data access pressure.

But don't diagnose based on the wait name alone.

You need to correlate:

```text id="h5m3p6"
Waits
+
Query
+
Execution plan
+
IO metrics
+
Workload
```

---

# 11. Example: blocking

This is one of the most common production incidents.

Session 1:

```sql id="ypv6xj"
BEGIN TRANSACTION;

UPDATE Orders
SET Status = 'Processing'
WHERE OrderID = 100;
```

But the transaction remains open.

Now Session 2:

```sql id="f5t5rd"
SELECT *
FROM Orders
WHERE OrderID = 100;
```

Session 2 may have to wait.

Conceptually:

```text id="5n1rkl"
Session 1
   |
   | Holds lock
   v
Orders
   ^
   |
   | Waiting
Session 2
```

Application sees:

```text
Query timeout
```

Developer says:

> "SQL MI is slow."

But CPU might be:

```text
20%
```

The real problem is:

> **Blocking.**

---

# 12. Blocking chain

It can get worse.

```text id="df29dr"
Session 1
   ↓
Blocks Session 2
         ↓
      Blocks Session 3
               ↓
            Blocks Session 4
```

You now have a blocking chain.

Example:

```text id="d7w8f3"
Head blocker
    |
    +---- Session 2
    |
    +---- Session 3
    |
    +---- Session 4
    |
    +---- Session 5
```

One poorly managed transaction can cause a large application outage.

---

# 13. How do you investigate blocking?

Start with:

```text id="q1qj8k"
Who is blocking?
Who is being blocked?
What SQL is running?
How long has the transaction been open?
What object/resource is locked?
```

You can investigate through DMVs and Azure monitoring tools.

For example, SQL Server provides views such as:

```sql id="8q7j1g"
sys.dm_exec_requests
sys.dm_exec_sessions
```

These can help identify active requests and sessions.

---

# 14. Deadlocks

Blocking can become a deadlock.

Imagine:

```text id="c9bqxy"
Transaction A
   |
   | locks Customer
   |
   v

Transaction B
   |
   | locks Order
   |
   v
```

Then:

```text id="7ctv1j"
A needs Order
B needs Customer
```

Now:

```text id="j7j2oa"
A → waiting for B

B → waiting for A
```

Nobody can proceed.

SQL Server detects the deadlock and chooses a victim.

```text id="6l0jpn"
Transaction A
     ↘
      DEADLOCK
     ↗
Transaction B

        ↓
SQL chooses victim
```

Application receives an error.

---

# 15. Why deadlocks happen

Common causes include:

```text id="7u1kvl"
Different access order
Long transactions
Missing indexes
Large updates
Poor transaction design
High concurrency
```

Example:

Transaction A:

```text id="qdy9hc"
Customer → Order
```

Transaction B:

```text id="q8u2b6"
Order → Customer
```

They acquire locks in different orders.

That's a classic deadlock pattern.

---

# 16. Query Store

Now we get to one of your most important performance tools:

# Query Store

Think of Query Store as:

> **A historical record of query performance and execution plans.**

Instead of only asking:

> "What is happening right now?"

you can ask:

> **"What changed?"**

---

# 17. Real incident

Monday:

```text
Query duration = 200 ms
```

Tuesday:

```text
Query duration = 8 seconds
```

Developer says:

> "Nothing changed."

Query Store may show:

```text id="2x4h2h"
Monday
  ↓
Plan A
  ↓
200 ms

Tuesday
  ↓
Plan B
  ↓
8 seconds
```

Now you investigate:

```text
Why did the plan change?
```

Possible causes include:

```text
Statistics
Data distribution
Parameter sensitivity
Index changes
Compatibility changes
Optimizer behavior
```

This is much more powerful than guessing.

---

# 18. Execution plans

Suppose Query Store identifies:

```text
Top expensive query
```

Now inspect its execution plan.

You might see:

```text id="r6w2f3"
Index Seek
    ↓
Nested Loops
    ↓
Key Lookup
```

Or:

```text id="8j6qfk"
Table Scan
    ↓
Hash Match
    ↓
Sort
```

The plan tells you how SQL Server is executing the query.

---

# 19. Table Scan vs Index Seek

Imagine:

```text id="k8evm7"
Orders = 100 million rows
```

Query:

```sql id="jbrh6s"
SELECT *
FROM Orders
WHERE OrderID = 12345;
```

If `OrderID` is properly indexed, SQL Server may use:

```text id="q3vylw"
Index Seek
```

Instead of:

```text id="4j4k9j"
Scan 100 million rows
```

That's why indexing matters.

---

# 20. But don't blindly add indexes

This is another DBA trap.

Someone sees a slow query and says:

> "Add an index."

Indexes can improve reads but create costs for:

```text id="qj7vqb"
INSERT
UPDATE
DELETE
Storage
Index maintenance
Memory
```

Example:

```text id="6fy8jp"
Orders
 |
 +-- Clustered index
 +-- Customer index
 +-- Date index
 +-- Status index
 +-- Amount index
 +-- ...
```

Too many indexes can hurt write-heavy workloads.

The goal is:

> **The right indexes for the actual workload.**

---

# 21. Application latency vs database latency

This is critical in Azure.

Suppose:

```text id="0mrykr"
Application response = 5 seconds
```

That doesn't necessarily mean:

```text
SQL query = 5 seconds
```

Maybe:

```text id="6g7vyo"
Application
   ↓
Network = 2 sec
   ↓
SQL = 1 sec
   ↓
Application processing = 2 sec
```

Total:

```text
5 seconds
```

So always separate:

```text
Application time
Network time
SQL execution time
```

---

# 22. Real troubleshooting example

Developer:

> **"Customer search takes 10 seconds."**

You investigate.

### Step 1

SQL duration:

```text
150 ms
```

### Step 2

Network:

```text
50 ms
```

### Step 3

Application processing:

```text
9.8 seconds
```

Conclusion:

```text
SQL MI = NOT the bottleneck
```

This prevents an unnecessary Azure resize.

---

# 23. Another scenario

Developer says:

> **"SQL MI is slow."**

You find:

```text
CPU = 30%
IO = Normal
Memory = Normal
```

But:

```text
Blocking = 200 sessions
```

Now you know:

```text
Performance problem
      ↓
Blocking
```

not:

```text
CPU shortage
```

---

# 24. Performance troubleshooting decision tree

Memorize this:

```text id="1o6k7u"
"SQL MI is slow"
        |
        v
Is CPU high?
   /          \
 YES           NO
 |              |
Find top CPU    Check IO
queries         |
                v
             IO high?
             /      \
           YES       NO
            |         |
         IO/query    Check
         analysis    blocking
                       |
                       v
                    Blocking?
                    /       \
                  YES        NO
                   |          |
                Locks       Check
                /TXNs       waits/
                             queries/
                             network
```

This is a **starting framework**, not a rigid diagnostic script.

---

# 25. What Azure monitoring adds

In Azure, you also need to think about platform-level monitoring.

Typical tools include:

```text id="2x2hfa"
Azure Monitor
Metrics
Diagnostic settings
Log Analytics
Alerts
Activity logs
SQL monitoring tools
Query Store
DMVs
```

Think of the layers:

```text id="x5p8vw"
Azure platform
      ↓
MI metrics
      ↓
SQL engine
      ↓
Query Store / DMVs
      ↓
Individual query
      ↓
Execution plan
```

You want to move from:

> **"Something is slow."**

to:

> **"Query X increased from 200 ms to 8 seconds because its execution plan changed and it is now scanning a large table."**

That's a DBA-quality diagnosis.

---

# 26. Your production incident

Let's make this realistic.

ABC Retail sends you an alert:

> **"Order processing is extremely slow."**

You collect:

```text id="x4j5rd"
CPU:             42%
Memory:          Normal
IO latency:      Normal
Connections:     Normal
```

Then you discover:

```text id="9smx5e"
Blocking sessions: 187
```

The head blocker has been running for:

```text
45 minutes
```

and is holding locks from a transaction.

### What do you investigate next?

Think about:

```text id="6aqh8n"
Who owns the transaction?
What SQL is running?
Why has it been open for 45 minutes?
Is the application waiting for user input?
Was a transaction left open?
What sessions are blocked?
What business operation is affected?
```

**Don't immediately kill the session.**

First understand what it is doing.

Killing a session can trigger rollback, and a large rollback can itself take time.

---

# 27. The DBA performance hierarchy

When troubleshooting, think:

```text id="e2x3wk"
1. Is the problem real?
        ↓
2. Is it application or database?
        ↓
3. Is it CPU / memory / IO / blocking?
        ↓
4. Which query/session?
        ↓
5. What execution plan?
        ↓
6. What changed?
        ↓
7. What is the safest fix?
        ↓
8. Did the fix actually work?
```

This approach will take you much further than memorizing individual DMV queries.

---

# Lesson 8 challenge 🧑‍💻

Production incident:

```text
Application response time: 12 seconds

MI CPU:                    35%
Memory:                    Normal
IO latency:                Normal
Blocking sessions:         250

Head blocker:
Transaction duration:      52 minutes
```

The developer says:

> "Increase MI vCores."

### Would you increase vCores immediately?

**Yes or No?**

And tell me:

**1. What is your first area of investigation?**

**2. Why might increasing vCores not fix the problem?**

**3. What would you inspect on the head-blocking session?**

**4. Why should you be careful before killing the session?**
