# Lesson 31 — Azure SQL Managed Instance Hands-On DBA Lab

We'll build your skills through **real production-style incidents**.

## Lab 1: Daily Health Check

Imagine you log in Monday morning and need to answer:

> **"Is my SQL Managed Instance healthy?"**

You should check these areas:

```text
1. Availability
2. CPU
3. Memory
4. IO
5. Storage
6. Blocking
7. Long-running queries
8. Deadlocks
9. Failed SQL Agent jobs
10. Query regressions
11. Security events
12. Backup/recovery status
```

---

# 1. Check active requests

Start with:

```sql
SELECT
    session_id,
    status,
    command,
    cpu_time,
    total_elapsed_time,
    logical_reads,
    writes,
    blocking_session_id
FROM sys.dm_exec_requests
WHERE session_id <> @@SPID
ORDER BY cpu_time DESC;
```

### How to read it

Suppose you get:

```text
session_id   cpu_time   elapsed   logical_reads   blocking
-----------  ---------  --------  --------------  --------
72           85000      90000     8000000         0
91           12000      50000     2000000         72
```

Session 72 is suspicious because:

```text
CPU high
Reads high
```

Session 91 is being blocked by:

```text
72
```

So you investigate **72 first**.

---

# 2. Find blocking

```sql
SELECT
    session_id,
    blocking_session_id,
    wait_type,
    wait_time,
    wait_resource
FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0
ORDER BY wait_time DESC;
```

Imagine:

```text
Session 100 → blocked by 57
Session 101 → blocked by 57
Session 102 → blocked by 57
Session 103 → blocked by 57
```

Then:

```text
57 = potential head blocker
```

But don't immediately kill it.

Investigate first.

---

# 3. Find the SQL text

You can combine request information with SQL text:

```sql
SELECT
    r.session_id,
    r.status,
    r.cpu_time,
    r.total_elapsed_time,
    r.logical_reads,
    r.blocking_session_id,
    t.text
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE r.session_id <> @@SPID
ORDER BY r.cpu_time DESC;
```

Now you can answer:

> **"What is this session actually running?"**

That's much more useful than just seeing a session ID.

---

# 4. Find long-running transactions

A transaction that stays open too long can cause major blocking.

You should investigate:

```text
Transaction
    ↓
Locks
    ↓
Blocking
    ↓
Application timeouts
```

Typical causes include:

* Application bugs
* Missing COMMIT
* Large batch operations
* Slow client processing
* Network interruption
* Poor transaction design

---

# 5. Real scenario

You discover:

```text
Session 57
Transaction duration: 48 minutes
Rows affected: 750,000
```

and:

```text
350 sessions
```

are waiting.

### Your thought process should be:

```text
Is session 57 still making progress?
          |
          +-- YES → investigate why it is taking so long
          |
          +-- NO  → determine safe mitigation
```

Then determine:

```text
Application?
User?
Transaction?
Business impact?
Rollback impact?
```

---

# 6. Don't make this DBA mistake

Never think:

> "Blocking = kill the blocker."

Instead:

> **"Blocking is a symptom. Find out why the transaction is holding locks."**

Killing a session may only hide the problem.

---

# Lab 2 — High CPU

Now imagine:

```text
CPU = 96%
```

You execute your request query.

You discover:

```text
Query A
CPU = 85% of total CPU
Executions = 20,000/min
Duration = 4 seconds
```

First conclusion:

> Query A deserves investigation.

---

# 7. Find expensive queries

For currently executing workload, investigate:

```text
CPU
Logical reads
Elapsed time
Execution count
```

For historical analysis, use:

> **Query Store**

Think:

```text
DMVs
 ↓
Current workload

Query Store
 ↓
Historical workload
```

---

# 8. Query Store investigation

Suppose:

```text
Yesterday:
Query = 200 ms

Today:
Query = 6 seconds
```

Query Store shows:

```text
Plan A → 200 ms
Plan B → 6 sec
```

This immediately suggests:

> **Plan regression.**

Now investigate why.

---

# 9. Compare plans

Plan A:

```text
Index Seek
     ↓
Nested Loops
```

Plan B:

```text
Table Scan
     ↓
Hash Match
     ↓
Sort
```

Don't immediately force Plan A.

First investigate:

```text
Statistics
Data distribution
Parameter values
Index changes
Schema changes
Recent deployment
Cardinality estimates
```

---

# Lab 3 — Indexing

Suppose this query is slow:

```sql
SELECT
    OrderDate,
    OrderAmount
FROM Sales.Orders
WHERE CustomerID = @CustomerID;
```

Current table:

```text
Orders = 300 million rows
```

Existing index:

```text
IX_Orders_OrderID
```

That's not useful for this predicate.

You might investigate:

```sql
CREATE INDEX IX_Orders_CustomerID
ON Sales.Orders(CustomerID)
INCLUDE(OrderDate, OrderAmount);
```

But again:

**Don't deploy blindly.**

---

# 10. Why?

Because the index creates costs:

```text
INSERT
   ↓
Index must be maintained

UPDATE
   ↓
Index may need maintenance

DELETE
   ↓
Index must be maintained
```

Plus:

```text
Storage
Maintenance
Memory/cache usage
```

Therefore:

> **Index based on workload evidence.**

---

# Lab 4 — Statistics

Suppose execution plan says:

```text
Estimated rows: 10
Actual rows:    5,000,000
```

That's a huge discrepancy.

Ask:

```text
Why is the optimizer estimating incorrectly?
```

Possible reasons:

```text
Stale statistics
Data distribution
Parameter sensitivity
Expression/predicate complexity
Correlation between columns
```

Don't assume statistics are always the answer—but they're one of the first things to investigate.

---

# Lab 5 — Deadlock

You receive:

```text
Transaction (Process ID 84)
was deadlocked on lock resources.
```

Deadlock graph:

```text
Transaction A
    |
    | owns Customer
    |
    v
waiting for Order


Transaction B
    |
    | owns Order
    |
    v
waiting for Customer
```

This creates:

```text
A → B
↑   ↓
└───┘
```

That's a cycle.

---

# 11. Permanent fix

One approach is to make transactions access resources consistently:

```text
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

Other possibilities:

```text
Shorter transactions
Better indexing
Reduced lock duration
Appropriate isolation strategy
Application retry logic
```

---

# Lab 6 — Accidental DELETE

Developer executes:

```sql
DELETE FROM Sales.Orders;
```

No `WHERE`.

Millions of rows disappear.

Your response:

```text
1. Determine exact time
2. Determine transaction state
3. Determine whether rollback is possible
4. If committed, identify recovery point
5. Recover appropriately
6. Validate data
7. Restore business operation
```

Don't immediately restore the entire production database without considering the business impact.

Sometimes you need to recover into a separate environment and extract the missing data.

For example:

```text
Production
    |
    +---- Current database
    |
    v
Temporary recovered database
    |
    v
Identify missing rows
    |
    v
Reinsert/repair
```

That can avoid unnecessary downtime.

---

# Lab 7 — Storage alert

Alert:

```text
Storage = 88%
```

Don't immediately increase capacity.

Investigate:

```text
Which database?
Which files?
Growth rate?
Data or log?
Expected growth?
Unexpected growth?
Large tables?
Retention problem?
```

Then determine:

```text
Immediate mitigation
+
Root cause
+
Long-term capacity plan
```

---

# Lab 8 — Migration validation

After migrating SQL Server → MI, don't just run:

```text
SELECT COUNT(*)
```

on one table.

Validate important business objects.

Example:

```text
Customer count
Order count
Revenue totals
Recent orders
Important reports
Stored procedures
Users
Jobs
Linked servers
Application connectivity
```

For example:

```text
Source:
Orders = 85,000,000

Target:
Orders = 85,000,000
```

That's useful, but not sufficient.

You should also validate important business aggregates.

---

# Lab 9 — Production cutover

A realistic cutover:

```text
T-7 days
Final migration rehearsal

T-1 day
Freeze structural changes

T-1 hour
Final readiness check

T-15 min
Application enters maintenance mode

T
Final synchronization

T+5 min
Validate database

T+10 min
Redirect application

T+15 min
Smoke testing

T+20 min
Business validation
```

If the business says:

> **"We have a 15-minute downtime requirement."**

your runbook must be designed around that requirement.

---

# Lab 10 — Rollback

Suppose:

```text
Application switched to MI
```

Then you discover:

```text
Critical payment workflow failing
```

You don't simply say:

> "Switch back."

First determine:

```text
What data was written to MI?
Can those changes be reconciled?
Is the old environment still consistent?
What is the business impact?
```

Rollback is a **data consistency problem**, not just a DNS problem.

---

# Lesson 32 — Build Your DBA Runbook

You should eventually have a document like this:

```text
PRODUCTION SQL MI RUNBOOK
=========================

1. Health check
2. CPU investigation
3. IO investigation
4. Blocking investigation
5. Deadlock investigation
6. Query regression
7. Storage issue
8. Security incident
9. Backup/recovery
10. Failover
11. DR activation
12. Migration cutover
13. Rollback
14. Post-incident review
```

This becomes extremely valuable in real operations.

---

# Lesson 33 — Senior-Level Thinking

At this stage, stop thinking only in terms of commands.

Think in this sequence:

```text
SYMPTOM
   ↓
EVIDENCE
   ↓
HYPOTHESIS
   ↓
TEST
   ↓
ROOT CAUSE
   ↓
MITIGATION
   ↓
PERMANENT FIX
   ↓
VALIDATION
   ↓
PREVENTION
```

Example:

```text
Application slow
      ↓
CPU 95%
      ↓
Query A consumes CPU
      ↓
Execution plan shows scan
      ↓
Statistics/index investigation
      ↓
Fix
      ↓
CPU falls to 45%
      ↓
Monitor
      ↓
Automate detection
```

That is the mindset I want you to develop.

---

# Lesson 34 — Final Capstone Begins

Now we're going to build a complete production environment mentally.

### Company

**ABC E-Commerce**

### Requirements

```text
24/7 application
15,000 transactions/minute
Customer PII
On-premises SQL Server today
Azure migration required
Minimal downtime
RPO <= 5 minutes
RTO <= 30 minutes
Production + DR
Strong auditing
Centralized monitoring
```

### Existing SQL Server

```text
CustomerDB      1 TB
OrdersDB        5 TB
ReportingDB     2 TB

120 SQL Agent jobs
80 logins
5 linked servers
15 SSIS packages
```

---

## Your architecture should contain

```text
                    USERS
                      |
                      v
                APPLICATION
                      |
                      v
               Azure Network
                      |
                      v
               SQL Managed
                 Instance
                      |
       +--------------+--------------+
       |              |              |
       v              v              v
      HA           Backup         Monitoring
       |              |              |
       v              v              v
   Failover         PITR        Azure Monitor
                      |
                      v
                     DR
                      |
                      v
               Secondary Region
```

And across the architecture:

```text
Identity
Security
Auditing
Networking
Automation
Capacity Planning
```

---

### Completed

✅ MI fundamentals
✅ Architecture
✅ Networking
✅ Security
✅ Backup/recovery
✅ HA/DR
✅ Monitoring
✅ Performance
✅ Query Store
✅ Execution plans
✅ Indexes
✅ Statistics
✅ Blocking
✅ Deadlocks
✅ Migration
✅ Cutover/rollback
✅ Automation concepts
✅ Production troubleshooting
