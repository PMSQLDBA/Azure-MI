# Lesson 13 — Real-World SQL Managed Instance Troubleshooting

The most important DBA skill isn't memorizing 100 DMV queries.

It's being able to go from:

> **"SQL is slow."**

to:

> **"This specific workload is waiting on this specific resource because of this specific root cause."**

---

# 1. Your DBA troubleshooting framework

When someone says:

> "The database is slow."

Don't immediately run random commands.

Use:

```text
Incident
   ↓
What exactly is failing?
   ↓
When did it start?
   ↓
Is it SQL or application?
   ↓
CPU / Memory / IO / Blocking / Waits?
   ↓
Which query/session?
   ↓
Execution plan?
   ↓
What changed?
   ↓
Fix
   ↓
Validate
```

This becomes your standard operating procedure.

---

# 2. Scenario #1 — CPU is 98%

Production alert:

```text
MI CPU = 98%
```

Application:

```text
Response time = 15 seconds
```

You investigate.

Top queries:

```text
Query A = 65% CPU
Query B = 20% CPU
Query C = 8% CPU
```

Immediately:

```text
CPU
 ↓
Top queries
 ↓
Execution plan
```

Don't resize the MI yet.

---

# 3. Query A

Suppose Query A is:

```sql id="m5c2sv"
SELECT *
FROM Orders
WHERE CustomerName = @CustomerName;
```

Execution plan:

```text id="3p1gdx"
Table Scan
     ↓
100 million rows
```

That's suspicious.

You investigate indexes.

Suppose there is no useful index on:

```text id="fh2y84"
CustomerName
```

Potential solution:

```text id="x6kj7h"
Appropriate index
+
Query optimization
```

But you must consider:

```text id="qxg7gu"
Selectivity
Write overhead
Storage
Existing indexes
Actual workload
```

Don't blindly create one.

---

# 4. Scenario #2 — CPU is only 30%, but everything is slow

This is a classic interview scenario.

Metrics:

```text id="7r0g2p"
CPU = 30%
Memory = Normal
IO = Normal
```

Application:

```text id="4h6x2a"
Timeouts
```

You investigate active requests.

You discover:

```text id="1s3k3r"
Blocking sessions = 150
```

Now the problem is:

```text id="2j3z4k"
Blocking
```

Not CPU.

---

# 5. Find the head blocker

Conceptually:

```text id="jy3h4f"
Session 51
    |
    | Holds lock
    v
Table
    ^
    |
Session 72
    |
Session 73
    |
Session 74
    |
Session 75
```

Session 51 is the **head blocker**.

Your questions:

```text id="8t20gx"
What query is Session 51 running?
How long has it been running?
Is it in a transaction?
Why is the transaction open?
What resource is locked?
What business operation owns it?
```

---

# 6. Don't blindly kill the blocker

This is a very important production DBA principle.

Someone might say:

> "Kill session 51!"

You should first understand the consequence.

Suppose Session 51 modified:

```text id="q6z9x4"
500,000 rows
```

and you kill it.

SQL Server may need to roll back the transaction.

Now:

```text id="t6w4pk"
Transaction
   ↓
ROLLBACK
   ↓
Large amount of work
```

The blocking may continue while rollback occurs.

So:

> **KILL is not always an instant solution.**

---

# 7. Scenario #3 — Deadlock

Application error:

```text id="m26z3j"
Transaction (Process ID 84)
was deadlocked...
```

Don't treat this as a random SQL failure.

Investigate:

```text id="l0d1q4"
Which queries?
Which tables?
Which locks?
Which transaction order?
Why did they conflict?
```

---

# 8. Classic deadlock

Transaction A:

```sql id="xk3k9s"
BEGIN TRANSACTION;

UPDATE Customers
SET Status = 'Active'
WHERE CustomerID = 1;

UPDATE Orders
SET Status = 'Processing'
WHERE OrderID = 100;
```

Transaction B:

```sql id="8k9r2m"
BEGIN TRANSACTION;

UPDATE Orders
SET Status = 'Processing'
WHERE OrderID = 100;

UPDATE Customers
SET Status = 'Active'
WHERE CustomerID = 1;
```

The order differs.

A:

```text id="7m1x3y"
Customer → Order
```

B:

```text id="b7v5p2"
Order → Customer
```

Potential deadlock:

```text id="3h2v8w"
A locks Customer
B locks Order

A waits for Order
B waits for Customer
```

SQL Server detects the cycle.

---

# 9. How do you fix deadlocks?

Potential solutions:

```text id="5u5q8a"
Consistent access order
Shorter transactions
Better indexes
Reduced lock duration
Query optimization
Appropriate isolation strategy
Application retry logic
```

The correct fix depends on the actual deadlock graph.

---

# 10. Scenario #4 — Query suddenly became slow

Monday:

```text id="z5j6s4"
Query duration = 300 ms
```

Tuesday:

```text id="p0j7u9"
Query duration = 12 seconds
```

Same query.

What changed?

This is where:

# Query Store

becomes extremely valuable.

---

# 11. Query Store investigation

Conceptually:

```text id="s5d0m8"
Query
  |
  +---- Plan A
  |       |
  |       +---- 300 ms
  |
  +---- Plan B
          |
          +---- 12 sec
```

You investigate:

```text id="2z8w0e"
Why did Plan B appear?
```

Possible causes:

```text id="y6v4x0"
Statistics
Data distribution
Parameter sensitivity
Index changes
Schema changes
Optimizer behavior
```

---

# 12. Parameter sniffing / parameter sensitivity

Suppose:

```sql id="t4v0h8"
SELECT *
FROM Orders
WHERE CustomerID = @CustomerID;
```

For one customer:

```text id="4f6yq8"
CustomerID = 10
Rows = 5
```

For another:

```text id="p2k8v5"
CustomerID = 5000
Rows = 5 million
```

A plan optimized for one distribution may perform badly for another.

You might see:

```text id="o3w1s9"
Parameter A
   ↓
Excellent plan

Parameter B
   ↓
Terrible plan
```

This is why execution plans and workload patterns matter.

---

# 13. Scenario #5 — IO is high

Suppose:

```text id="8q6h4k"
CPU = 40%
IO = High
```

You identify:

```text id="3g7j1p"
Top query
```

Execution plan:

```text id="a7q3zs"
Clustered Index Scan
       ↓
200 million rows
       ↓
Filter
```

Maybe the query is reading far more data than necessary.

Potential investigation:

```text id="1l5k0x"
Indexes
Query predicates
Statistics
Data volume
Execution plan
Selectivity
```

---

# 14. Scenario #6 — Memory grant problem

Query:

```sql id="7q2z1p"
SELECT CustomerID, SUM(OrderAmount)
FROM Orders
GROUP BY CustomerID
ORDER BY SUM(OrderAmount) DESC;
```

Execution requires:

```text id="3u9f4e"
Hash/aggregate
+
Sort
```

Suppose SQL Server requests a large memory grant.

Other queries now wait.

You might see:

```text id="4r7z1d"
Query A
   ↓
Large memory grant
   ↓
Query B waiting
   ↓
Query C waiting
```

The solution isn't necessarily:

```text id="2b6v5j"
Add memory
```

First investigate the query and its execution plan.

---

# 15. Scenario #7 — TempDB pressure

Suppose queries perform large:

```text id="9r3c2m"
Sorts
Hash joins
Temporary objects
```

and spill operations to tempdb.

You may observe:

```text id="3z4x1h"
TempDB activity ↑
IO ↑
Query duration ↑
```

Investigate:

```text id="e7w2q8"
Execution plan
Spills
Memory grants
TempDB usage
Query design
```

---

# 16. Scenario #8 — Application timeout but SQL query is fast

Application says:

> "SQL query is taking 10 seconds."

You measure SQL execution:

```text id="h5x0m9"
SQL = 200 ms
```

Then:

```text id="3t1c7a"
Network = 100 ms
Application processing = 9 sec
```

Conclusion:

```text id="f7n5h2"
SQL isn't the bottleneck.
```

This is why you should always correlate:

```text id="m6h9e1"
Application telemetry
+
Azure monitoring
+
SQL monitoring
```

---

# 17. DMV thinking

SQL Server exposes Dynamic Management Views.

Think of DMVs as:

> **Windows into the current state of the SQL engine.**

Useful categories include:

```text id="n1h2q3"
Sessions
Requests
Connections
Waits
Indexes
Query execution
Memory
IO
```

For example:

```sql id="2v9p4m"
sys.dm_exec_requests
```

helps investigate currently executing requests.

And:

```sql id="8k3r5x"
sys.dm_exec_sessions
```

helps investigate sessions.

---

# 18. Current vs historical troubleshooting

This distinction is crucial.

### DMV

Generally useful for:

```text id="f0y3w1"
What is happening now?
```

### Query Store

Useful for:

```text id="p6z8t2"
What happened historically?
What changed?
```

So:

```text id="m4n7k2"
Current incident
     ↓
DMVs

Performance regression
     ↓
Query Store
```

Use both.

---

# 19. Wait statistics

Suppose the query is slow.

You find:

```text id="x7j2m5"
CPU isn't high.
```

Ask:

> What is the query waiting on?

Conceptually:

```text id="r8p3v1"
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
 +-- Other resources
```

Wait statistics help identify where execution time is being spent.

But remember:

> **A wait type is a clue, not automatically a root cause.**

Always correlate it with the workload.

---

# 20. A senior DBA's thought process

Junior approach:

> "CPU is high. Increase vCores."

Senior approach:

> "CPU is high. Which workload is consuming CPU? Is the workload expected? Did a query or execution plan change? Can we reduce the work before scaling?"

---

# 21. Scaling vs optimization

You have two broad approaches:

### Vertical scaling

```text id="q9h7v4"
More vCores
More resources
```

### Workload optimization

```text id="z5c3k8"
Better query
Better index
Better plan
Less contention
Better application behavior
```

Sometimes you need both.

For example:

```text id="5d2p1v"
Current MI
   ↓
Optimize queries
   ↓
Still insufficient
   ↓
Scale MI
```

That's often better than scaling blindly.

---

# 22. Real production incident

At 9:00 AM:

```text id="b6q3r1"
CPU = 40%
```

At 9:15 AM:

```text id="k5n8s2"
CPU = 95%
```

You discover:

```text id="p3v7m1"
New application release
```

Query Store:

```text id="w8r2k4"
New query
↓
Huge scan
↓
CPU consumption ↑
```

Root cause:

```text id="q1s6d9"
Application deployment
```

Not:

```text id="r5h2n8"
Azure infrastructure
```

This is why correlating events by time is powerful.

---

# 23. Production troubleshooting checklist

When an MI performance alert arrives:

```text id="q3f8s1"
☐ Confirm incident
☐ Determine start time
☐ Check CPU
☐ Check memory
☐ Check IO
☐ Check connections
☐ Check blocking
☐ Check deadlocks
☐ Check waits
☐ Identify expensive queries
☐ Review Query Store
☐ Compare execution plans
☐ Check recent deployments
☐ Check schema/index/statistics changes
☐ Check application behavior
☐ Apply safest fix
☐ Validate
☐ Document root cause
```

---

# 24. Incident report example

After fixing an outage, don't just write:

> "Fixed SQL performance."

Write:

```text id="e6f8q2"
Incident:
Order processing latency increased.

Start:
09:15 AM

Impact:
Order API latency > 10 seconds.

Root cause:
New query introduced by application deployment performed a large table scan.

Evidence:
Query Store showed increased CPU and a new execution plan.

Resolution:
Application query/index was optimized.

Validation:
Average latency returned to normal.

Preventive action:
Added performance testing to deployment process.
```

That's professional incident management.

---

# 25. The complete DBA troubleshooting model

Memorize this:

```text id="s7x4n1"
                USER REPORT
                    |
                    v
              "SQL IS SLOW"
                    |
          +---------+---------+
          |                   |
          v                   v
      SQL issue          App/network issue
          |
          v
 +--------+--------+
 |        |        |
CPU     IO     Blocking
 |        |        |
 v        v        v
Query   Query    Locks
 |        |        |
 +--------+--------+
          |
          v
      Execution Plan
          |
          v
       Root Cause
          |
          v
         Fix
          |
          v
       Validate
```

This is the mindset I want you to develop.

---

# Final practical challenge 🧑‍💻

You are the production DBA for ABC Retail.

At **10:00 AM**, the application team reports:

> "Order processing has become extremely slow."

You collect:

```text
MI CPU:                 35%
MI Memory:              Normal
MI IO:                  Normal

Application latency:    15 seconds

Blocking sessions:      0

Deadlocks:              0

Top query duration:     200 ms

Network latency:        12 seconds
```

### Your task:

Answer these **5 questions**:

**1. Is SQL MI the likely root cause?**

**2. What evidence supports your conclusion?**

**3. What would you investigate next?**

**4. Would you scale the MI?**

**5. How would you explain the incident to the application team?**

