# Lesson 48 — Azure SQL MI Advanced DBA: Real Production Troubleshooting Toolkit

Now we move from incident scenarios into the **commands and techniques you would actually use during an incident**.

The goal is to connect:

**symptom → DMV → evidence → diagnosis → action**

---

## 1. First command: What's running right now?

```sql
SELECT
    r.session_id,
    r.status,
    r.command,
    DB_NAME(r.database_id) AS database_name,
    r.cpu_time,
    r.total_elapsed_time,
    r.logical_reads,
    r.writes,
    r.wait_type,
    r.wait_time,
    r.blocking_session_id,
    t.text AS sql_text
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE r.session_id <> @@SPID
ORDER BY r.cpu_time DESC;
```

This is one of your most useful first-response queries.

It tells you:

```text
Who is running?
What database?
How much CPU?
How long?
How many reads?
What is it waiting for?
Who is blocking it?
What SQL is executing?
```

---

# 2. Find blocking

```sql
SELECT
    r.session_id,
    r.blocking_session_id,
    r.wait_type,
    r.wait_time,
    r.wait_resource,
    DB_NAME(r.database_id) AS database_name,
    t.text
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE r.blocking_session_id <> 0
ORDER BY r.wait_time DESC;
```

Suppose you see:

```text
Session 101 → blocked by 57
Session 102 → blocked by 57
Session 103 → blocked by 57
Session 104 → blocked by 57
```

You have a candidate head blocker:

```text
57
```

But remember:

> **Don't kill it merely because it is a blocker.**

---

# 3. Find the blocker

```sql
SELECT
    session_id,
    status,
    login_name,
    host_name,
    program_name,
    database_id,
    open_transaction_count
FROM sys.dm_exec_sessions
WHERE session_id = 57;
```

Now you can discover:

```text
Login
Application
Host
Open transactions
```

This is much more useful than simply knowing:

```text
SPID = 57
```

---

# 4. Find open transactions

Use transaction information to determine whether the blocker has an active transaction.

Conceptually:

```text
Session
   ↓
Transaction
   ↓
Locks
   ↓
Blocking
```

Suppose:

```text
Transaction age = 2 hours
```

That's a strong clue.

---

# 5. Find locks

You can inspect lock information using:

```sql
SELECT
    request_session_id,
    resource_type,
    resource_database_id,
    request_mode,
    request_status
FROM sys.dm_tran_locks
WHERE request_session_id <> @@SPID
ORDER BY request_session_id;
```

Now you can ask:

```text
What resource is locked?
Which session owns it?
What lock mode?
Is another session waiting?
```

---

# 6. CPU troubleshooting

Suppose:

```text
CPU = 95%
```

Don't immediately increase compute.

First establish:

```text
Which database?
Which queries?
Which sessions?
Which application?
Which execution plans?
```

A useful current-workload query:

```sql
SELECT TOP (20)
    r.session_id,
    r.cpu_time,
    r.total_elapsed_time,
    r.logical_reads,
    DB_NAME(r.database_id) AS database_name,
    t.text
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
ORDER BY r.cpu_time DESC;
```

---

# 7. Historical CPU investigation

Current DMVs answer:

> **"What's happening now?"**

Query Store answers:

> **"What has been happening over time?"**

This distinction is critical.

Imagine:

```text
09:00 → Query = 200 ms
10:00 → Query = 200 ms
11:00 → Query = 8 sec
```

The current DMV may only show the latest execution.

Query Store can help you see the historical change.

---

# 8. Query Store mindset

When investigating a slow query, ask:

```text
Has it always been slow?
```

versus:

```text
Did it become slow recently?
```

If it became slow recently:

```text
Plan regression
Statistics
Index change
Data growth
Parameter sensitivity
Application change
```

become important investigation paths.

---

# 9. Execution plan investigation

Suppose:

```text
Estimated rows = 100
Actual rows    = 5,000,000
```

That's a major cardinality estimation problem.

Investigate:

```text
Statistics
Predicates
Data distribution
Parameter values
Expressions
Joins
```

Don't immediately blame the optimizer.

The optimizer can only make decisions based on the information available to it.

---

# 10. Index investigation

Suppose a query does:

```sql
SELECT
    OrderDate,
    OrderAmount
FROM Sales.Orders
WHERE CustomerID = @CustomerID;
```

but performs a large scan.

You investigate:

```text
Existing indexes
Predicate columns
Included columns
Selectivity
Write workload
Index usage
```

Potential solution:

```sql
CREATE INDEX IX_Orders_CustomerID
ON Sales.Orders(CustomerID)
INCLUDE(OrderDate, OrderAmount);
```

But again:

> **Never create an index solely because an execution plan suggests it.**

Consider:

```text
Read benefit
+
Write overhead
+
Storage
+
Maintenance
+
Existing indexes
```

---

# 11. Storage investigation

Start at:

```text
MI
 ↓
Database
 ↓
File
 ↓
Table
 ↓
Growth rate
```

For example:

```sql
SELECT
    DB_NAME(database_id) AS database_name,
    name,
    type_desc,
    size * 8.0 / 1024 AS size_mb
FROM sys.master_files
ORDER BY size DESC;
```

Then determine whether the problem is:

```text
Data
Log
TempDB
Unexpected growth
Normal business growth
```

---

# 12. Log troubleshooting

This is a very important query:

```sql
SELECT
    name,
    recovery_model_desc,
    log_reuse_wait_desc
FROM sys.databases;
```

Suppose:

```text
OrdersDB
log_reuse_wait_desc = ACTIVE_TRANSACTION
```

Then investigate long-running transactions.

Don't respond with:

```sql
DBCC SHRINKFILE(...)
```

as your first thought.

Instead ask:

> **Why is the log not becoming reusable?**

---

# 13. Wait statistics

One of the most powerful performance concepts is:

> **What is SQL Server waiting for?**

Think:

```text
CPU problem
      vs
Waiting problem
```

For example:

```text
High CPU
→ CPU pressure

High PAGEIOLATCH-type waits
→ storage/I/O investigation

High lock waits
→ blocking investigation

High network-related waits
→ application/network investigation
```

Wait statistics help point your investigation in the right direction.

---

# 14. A critical production principle

Never interpret one metric in isolation.

For example:

```text
CPU = 30%
```

does **not** mean:

> "Database is healthy."

You could have:

```text
CPU = 30%
Blocking = severe
```

and users can still experience terrible performance.

Likewise:

```text
CPU = 95%
```

doesn't automatically mean:

> "Increase compute."

The CPU could be caused by one poorly performing query.

---

# 15. The 5-question troubleshooting method

Whenever someone says:

> **"SQL MI is slow."**

Ask:

### Question 1

**Slow compared with what?**

```text
Baseline?
Normal workload?
Yesterday?
Last week?
```

### Question 2

**When did it start?**

```text
Suddenly?
Gradually?
After deployment?
After data growth?
```

### Question 3

**Who is affected?**

```text
All applications?
One application?
One database?
One query?
```

### Question 4

**What changed?**

```text
Code
Data
Index
Statistics
Configuration
Traffic
Network
```

### Question 5

**What is the evidence?**

```text
CPU
Waits
Blocking
Query Store
Execution plan
DMVs
Logs
```

This five-question approach prevents random troubleshooting.

---

# Lesson 49 — Production Change Management

A senior DBA isn't only good at fixing problems.

They're also good at **not creating new problems**.

Suppose someone requests:

> "Create this index in production immediately."

Your response shouldn't automatically be:

```sql
CREATE INDEX ...
```

Ask:

```text
Why?
Evidence?
Expected benefit?
Size?
Write overhead?
Maintenance impact?
Deployment window?
Rollback plan?
```

---

## Change lifecycle

```text
Request
   ↓
Impact assessment
   ↓
Test
   ↓
Approval
   ↓
Production deployment
   ↓
Validation
   ↓
Monitoring
```

For a production index:

```text
Before
 ↓
Capture baseline
 ↓
Create index
 ↓
Monitor
 ↓
Compare performance
 ↓
Keep or roll back
```

---

# Lesson 50 — Your Azure SQL MI DBA Checklist

You should eventually be comfortable with this checklist.

## Daily

```text
☐ MI availability
☐ CPU
☐ Memory
☐ Storage
☐ IO
☐ Blocking
☐ Long-running requests
☐ Failed jobs
☐ Query regressions
☐ Security alerts
```

## Weekly

```text
☐ Capacity trend
☐ Top expensive queries
☐ Index review
☐ Statistics concerns
☐ Job failures
☐ Security review
☐ Backup/recovery validation
```

## Monthly

```text
☐ DR readiness
☐ RPO/RTO test/review
☐ Capacity forecast
☐ Permission review
☐ Patch/platform review
☐ Query performance trends
☐ Cost review
```

---

# The DBA command hierarchy

Remember this:

```text
SYMPTOM
   ↓
METRICS
   ↓
DMVs
   ↓
QUERY STORE
   ↓
EXECUTION PLAN
   ↓
ROOT CAUSE
   ↓
CHANGE
   ↓
VALIDATION
```

Don't skip from:

```text
SYMPTOM
```

straight to:

```text
CHANGE
```

That's how production incidents become worse.

---

