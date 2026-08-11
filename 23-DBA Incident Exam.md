# Lesson 38 — Final DBA Incident Exam

## Incident 1 — Production MI at 96% CPU

### Environment

```text
Azure SQL Managed Instance
Production
24/7 e-commerce application

Databases:
OrdersDB
CustomerDB
ReportingDB
```

At **10:05 AM**, Azure Monitor sends:

```text
CPU utilization: 96%
```

Five minutes later, the application team reports:

> "Checkout is extremely slow. Some requests are timing out."

You check the MI:

```text
CPU:              97%
Storage:          54%
Memory:           Normal
IO:               Normal
Blocked sessions: 3
Deadlocks:        0
```

You check currently executing requests:

```text
Session  CPU     Duration    Reads       Database
------   ------  ----------  ----------  ----------
71       85%     8 sec       8,500,000   OrdersDB
82       3%      200 ms      12,000      OrdersDB
91       2%      500 ms      25,000      CustomerDB
```

Session 71 is running:

```sql
SELECT
    CustomerID,
    SUM(OrderAmount)
FROM Sales.Orders
WHERE OrderDate >= @StartDate
GROUP BY CustomerID;
```

Query Store shows:

```text
Yesterday:
Average duration = 180 ms

Today:
Average duration = 8.2 sec
```

And Query Store shows two plans:

```text
Plan A
Index Seek
Average duration: 180 ms

Plan B
Clustered Index Scan
Average duration: 8.2 sec
```

Plan B started appearing at **9:55 AM**.

---

## Your task

You're the production DBA.

Explain **exactly what you would do**, in order.

Don't just say:

> "I would check the execution plan."

I want your complete troubleshooting thought process:

```text
1. What do you verify first?
2. What evidence do you collect?
3. What is your likely root cause?
4. What would you check before changing anything?
5. What immediate mitigation would you consider?
6. Would you restart/scale the MI?
7. How would you validate your fix?
8. What would you do to prevent recurrence?
```

### Important

There may be **more than one valid answer**. I'm evaluating your **DBA reasoning**, not whether you memorize a particular command.

Reply with your diagnosis and action plan. Then I'll grade it **out of 10 like a senior Azure SQL DBA interview**, correct anything missing, and give you **Incident 2 — 300-session blocking storm**.

Let's continue with **Incident 2**. This one tests whether you can distinguish **blocking, deadlocks, CPU pressure, and a long-running transaction**.

# Lesson 39 — Incident 2: 300 Blocked Sessions

### Production environment

```text
Azure SQL Managed Instance
Database: OrdersDB
Business: E-commerce
Traffic: 24/7
```

At **2:15 PM**, the application monitoring system reports:

```text
Checkout failures:        Increasing
API response time:        12 seconds
Database CPU:             32%
Database IO:              Normal
```

Azure Monitor shows:

```text
CPU:                 32%
Storage:             61%
Memory:              Normal
Deadlocks:           0
```

But when you inspect active requests:

```text
Total active sessions:       410
Blocked sessions:            302
```

You identify this chain:

```text
Session 57
    |
    +---- Session 82
    +---- Session 91
    +---- Session 104
    +---- Session 127
    +---- Session 155
    +---- Session 201
    +---- ...
```

Session 57 has:

```text
Status:              sleeping
Open transaction:     YES
Transaction age:     52 minutes
```

The application team tells you:

> "We have not deployed anything today."

You investigate the transaction and find:

```sql
BEGIN TRANSACTION;

UPDATE Sales.Orders
SET Status = 'Processing'
WHERE OrderID BETWEEN 100000 AND 900000;

-- Application has not committed
```

The transaction modified approximately:

```text
750,000 rows
```

---

# Your DBA challenge

You are on-call.

Explain your response in this order:

### 1. Diagnosis

What exactly is happening?

Is this:

```text
A. High CPU
B. Deadlock
C. Blocking
D. Storage problem
```

And why?

---

### 2. Immediate investigation

What would you check about **Session 57**?

Think about:

```text
Who owns it?
What application created it?
What transaction is open?
Is it still doing work?
How many locks does it hold?
What resources are blocked?
Why hasn't it committed?
```

---

### 3. Would you KILL session 57?

This is the important question.

If you decide:

```sql
KILL 57;
```

what could happen afterward?

Think about:

```text
Rollback
Rollback duration
Additional blocking
Application impact
Data consistency
```

---

### 4. Business decision

The application team says:

> "Checkout is down. Kill it now!"

But the DBA discovers the transaction is rolling back potentially hundreds of thousands of modifications.

What information do you need before deciding?

---

### 5. Permanent fix

Assuming the application forgot to commit the transaction, what would you recommend changing?

Think beyond:

> "Kill the session."

Possible areas:

```text
Transaction design
Batch size
Application error handling
Connection handling
Timeouts
Monitoring
Long-running transaction alerts
```

---

# Senior DBA bonus question

Why can this incident happen even though:

```text
CPU = 32%
IO = Normal
Deadlocks = 0
```

This is an important concept.

A database can be **very slow for users without having high CPU**.

For example:

```text
Application request
       |
       v
Waiting for lock
       |
       v
No significant CPU consumption
       |
       v
User experiences 12-second response
```

So remember:

> **Performance problems are not synonymous with CPU problems.**

---

## Your answer should look something like this

Don't worry about exact syntax yet. Explain your reasoning:

```text
1. Identify the head blocker
2. Confirm the transaction and locks
3. Determine why it is open
4. Assess business impact
5. Decide whether termination is safe
6. Understand rollback implications
7. Resolve the immediate blocking
8. Fix the application/transaction design
9. Add monitoring to detect it earlier
10. Validate that blocking has disappeared
```

After this one, we'll move to **Incident 3: Deadlock storm**, where you'll have to analyze two transactions fighting over the same resources.

# Lesson 40 — Incident 3: Deadlock Storm

This is a **senior DBA-level scenario** because the database may look healthy while the application is repeatedly failing.

## Production situation

It's **11:40 AM**.

The application team reports:

> "Orders occasionally fail with SQL deadlock errors. Retrying the request sometimes works."

Azure Monitor:

```text
CPU:                  41%
Storage:              48%
IO:                   Normal
Memory:               Normal
Blocked sessions:     Low
```

But application errors have increased:

```text
Deadlock errors:
11:00 → 2
11:15 → 7
11:30 → 23
11:40 → 51
```

---

# 1. The deadlock graph

You capture a deadlock graph.

It shows two transactions.

### Transaction A

```sql
BEGIN TRANSACTION;

UPDATE Sales.Customers
SET LastOrderDate = GETDATE()
WHERE CustomerID = @CustomerID;

UPDATE Sales.Orders
SET Status = 'Processing'
WHERE OrderID = @OrderID;

COMMIT;
```

### Transaction B

```sql
BEGIN TRANSACTION;

UPDATE Sales.Orders
SET Status = 'Completed'
WHERE OrderID = @OrderID;

UPDATE Sales.Customers
SET LastOrderDate = GETDATE()
WHERE CustomerID = @CustomerID;

COMMIT;
```

---

# 2. What is happening?

Transaction A acquires:

```text id="x3xv09"
Customer lock
```

Then wants:

```text id="z2d1a1"
Order lock
```

Transaction B already owns:

```text id="t5y0fj"
Order lock
```

and wants:

```text id="4o3y7h"
Customer lock
```

So:

```text id="u2zv9q"
Transaction A
    |
    | owns Customer
    |
    | waits for Order
    |
    v

Transaction B
    |
    | owns Order
    |
    | waits for Customer
    |
    +------------------+
```

That's a cycle.

SQL Server detects the deadlock and chooses one transaction as the **deadlock victim**.

---

# 3. Important distinction

This is **not ordinary blocking**.

Blocking:

```text id="ryg8tm"
A owns resource
B waits
```

Deadlock:

```text id="8qj8d2"
A owns Resource 1
A waits for Resource 2

B owns Resource 2
B waits for Resource 1
```

So:

```text id="x8xg5h"
Blocking
=
Waiting

Deadlock
=
Circular waiting
```

---

# 4. Your first question

Should you increase the MI size because CPU is 41%?

### No.

The evidence doesn't point toward resource exhaustion.

The problem is:

```text id="hlg4qv"
Transaction ordering
        ↓
Circular locking
        ↓
Deadlock
        ↓
Victim transaction
        ↓
Application failure
```

Adding CPU or memory doesn't fix that logical locking pattern.

---

# 5. What is the likely permanent fix?

Look at the order in which resources are accessed.

Transaction A:

```text id="m8y08q"
Customer
   ↓
Order
```

Transaction B:

```text id="bx8z8j"
Order
   ↓
Customer
```

The application should ideally use a consistent ordering.

For example:

```text id="v8p0wu"
Customer
   ↓
Order
```

for both transactions, assuming that ordering works with the application's business logic.

Then:

```text id="m8o9d5"
Transaction A:
Customer → Order

Transaction B:
Customer → Order
```

The circular dependency can be eliminated.

---

# 6. But don't stop there

As DBA, investigate:

### Transaction duration

Are transactions unnecessarily long?

```text id="n8d6vi"
BEGIN
  ↓
Many operations
  ↓
External call
  ↓
More operations
  ↓
COMMIT
```

That's dangerous.

Ideally, don't hold database locks while waiting on external services unnecessarily.

---

# 7. Check indexes

Poor indexes can make a transaction hold locks longer.

For example:

```text id="s5r0kw"
UPDATE
   ↓
Large scan
   ↓
Many rows examined
   ↓
Longer execution
   ↓
Locks held longer
   ↓
Higher deadlock probability
```

So investigate whether the affected statements have appropriate access paths.

---

# 8. Application retry

Deadlocks are transient by nature.

A well-designed application should often retry the victim transaction.

Conceptually:

```text id="q1k1fq"
Transaction
    ↓
Deadlock
    ↓
Retry
    ↓
Success
```

But:

> **Retry logic is a mitigation, not necessarily the root-cause fix.**

If you have:

```text id="l1g8gv"
1 deadlock/hour
```

retry may be acceptable.

If you have:

```text id="6zkwb9"
5,000 deadlocks/hour
```

you have an architectural problem to solve.

---

# 9. What should you collect?

For a production investigation:

```text id="drt4y6"
Deadlock graph
Victim session
Resource types
Objects involved
SQL statements
Transaction order
Indexes
Isolation level
Execution plans
Transaction duration
Application behavior
```

The deadlock graph is particularly valuable because it shows the relationship between the sessions and resources.

---

# 10. Real-world example

Imagine an e-commerce checkout.

Transaction A:

```text id="whu1zt"
Update Customer
Reserve Order
```

Transaction B:

```text id="0zrv2m"
Update Order
Update Customer
```

During heavy traffic:

```text id="whu7g2"
Checkout #1001
     ↓
Customer locked
```

while:

```text id="0e9q76"
Order processing #9001
     ↓
Order locked
```

Then both need the resource owned by the other.

The result:

```text id="a5u5nq"
DEADLOCK
```

This is why deadlocks often appear during traffic spikes even when the system looked fine during testing.

---

# 11. Production response

Your runbook could be:

```text id="o0l6zo"
DEADLOCK ALERT
      ↓
Capture deadlock graph
      ↓
Identify victim
      ↓
Identify participating statements
      ↓
Determine resource order
      ↓
Check transaction duration
      ↓
Check indexes/plans
      ↓
Review application retry
      ↓
Fix inconsistent ordering
      ↓
Deploy
      ↓
Monitor deadlock frequency
```

---

# 12. Interview answer

If an interviewer asks:

> **"How do you troubleshoot deadlocks in Azure SQL Managed Instance?"**

A strong answer is:

> "I first capture and analyze the deadlock graph to identify the participating sessions, resources, SQL statements, and victim. Then I determine the locking order and transaction boundaries. I check whether inconsistent resource-access order, long transactions, inefficient plans, or missing indexes are contributing. For immediate resilience I verify that the application handles the deadlock victim with appropriate retry logic. For the permanent fix, I try to make resource access consistent and reduce transaction duration. Finally, I validate the deadlock rate after the change."

That's a **senior-level answer**.

---

# Incident 4 — Query Suddenly Became 40× Slower

Now we're moving into **query tuning and Query Store**, one of the most important areas for an Azure SQL DBA.

## Situation

At 9:00 AM:

```text
Query duration = 180 ms
```

At 9:30 AM:

```text
Query duration = 7.5 seconds
```

Application team:

> "Nothing changed in the application."

MI metrics:

```text id="e9as11"
CPU:          38%
IO:           Normal
Memory:       Normal
Blocking:     0
Deadlocks:    0
```

So infrastructure looks healthy.

You open Query Store.

You find:

```text id="q7d5jp"
Query ID: 1842

Plan 17:
Average duration = 180 ms
Executions       = 80,000

Plan 23:
Average duration = 7.5 sec
Executions       = 6,000
```

Plan 17:

```text id="h9k2av"
Index Seek
     ↓
Nested Loops
     ↓
Small result
```

Plan 23:

```text id="7ayj1k"
Clustered Index Scan
     ↓
Hash Match
     ↓
Large result
```

---

# Your diagnosis

This is strongly pointing toward:

> **Query execution plan regression.**

But don't immediately say:

> "Force Plan 17."

First investigate **why Plan 23 was selected**.

---

## Things to investigate

### 1. Statistics

Did the data distribution change?

```text id="8y6n5f"
Yesterday:
Customer has 50 orders

Today:
Customer has 5 million orders
```

A plan that's good for one distribution might be bad for another.

---

### 2. Parameter sensitivity

Suppose:

```sql id="1g7qts"
SELECT *
FROM Sales.Orders
WHERE CustomerID = @CustomerID;
```

Customer A:

```text
3 rows
```

Customer B:

```text
10 million rows
```

A plan optimized for Customer A might be terrible for Customer B.

This is one reason parameter-sensitive workloads deserve careful investigation.

---

### 3. Index changes

Was an index:

```text id="7qu0fz"
Dropped?
Disabled?
Changed?
```

Or did a schema deployment change the workload?

---

### 4. Data growth

Maybe the table grew from:

```text id="5e7p9w"
50 million
```

to:

```text id="8v7e8d"
500 million rows
```

A plan that was acceptable before may no longer be ideal.

---

# Immediate mitigation

If you've confirmed:

```text id="b5of4x"
Plan 17 = known good
Plan 23 = known bad
```

and the business impact is severe, a plan forcing strategy may be considered as a **temporary mitigation**, depending on the circumstances.

But document:

```text id="2gytpk"
Why it was forced
When it was forced
What evidence supported it
Who approved it
What permanent investigation remains
```

Don't turn plan forcing into:

> "Our permanent performance strategy."

---

# Validate

After mitigation:

```text id="0e3g2g"
Query duration
CPU
Logical reads
Execution count
Application latency
```

Compare:

```text id="t6w4n9"
Before
7.5 sec

After
200 ms
```

Then continue investigating the root cause.

---

# Lesson 40 takeaway

You've now seen four critical production patterns:

```text
1. High CPU
   → Find expensive workload

2. Blocking
   → Find head blocker and transaction cause

3. Deadlock
   → Find circular resource dependency

4. Query regression
   → Compare historical execution plans
```

The key is to **identify the type of problem before choosing the solution**.

Next comes **Incident 5 — Storage is 96% full and growing rapidly**, where we'll deal with capacity, data/log growth, and what a production DBA should—and should not—do when the database is approaching a critical storage threshold.

# Lesson 41 — Incident 5: Storage at 96%

This is a very realistic production incident because **running out of storage is different from running out of CPU**.

## Production alert

At **3:20 AM**, Azure Monitor sends:

```text
CRITICAL

SQL Managed Instance storage utilization: 96%
```

At the same time:

```text
CPU:          34%
Memory:       Normal
IO:           Increasing
Blocking:     0
Deadlocks:    0
```

Application team reports:

> "Some batch jobs are failing with database errors."

---

# 1. First question: What's consuming the storage?

Don't immediately assume:

> "The database is too large."

You need to determine **what is growing**.

Think:

```text
MI storage
    |
    +---- Data files
    |
    +---- Log files
    |
    +---- TempDB / system usage
    |
    +---- Other database files
```

You need to identify the actual consumer.

---

# 2. Database-level investigation

Start by identifying database sizes and growth.

For example:

```sql
SELECT
    name,
    state_desc,
    recovery_model_desc
FROM sys.databases
ORDER BY name;
```

Then investigate file-level information.

A useful view is:

```sql
SELECT
    DB_NAME(database_id) AS database_name,
    name AS logical_file_name,
    type_desc,
    size * 8.0 / 1024 AS size_mb,
    physical_name
FROM sys.master_files
ORDER BY size DESC;
```

This gives you a starting point for determining:

```text
Database
File
File type
Current size
```

---

# 3. Suppose you discover this

```text
Database       File type     Size
-------------  ------------  --------
OrdersDB       ROWS          7.2 TB
OrdersDB       LOG           1.1 TB
CustomerDB     ROWS          1.0 TB
ReportingDB    ROWS          2.0 TB
```

Now the investigation becomes much more interesting.

The log is:

```text
1.1 TB
```

That's potentially suspicious.

But **don't shrink it immediately**.

---

# 4. Important DBA principle: Don't blindly shrink the log

A common junior DBA reaction is:

```sql
DBCC SHRINKFILE(...);
```

to get space back.

That may provide temporary relief in certain situations, but it does not solve the reason the log grew.

Worse, repeated shrink/grow cycles are undesirable.

Think:

```text
Log grows
   ↓
Shrink
   ↓
Log grows again
   ↓
Shrink
   ↓
Log grows again
```

That's not capacity management.

---

# 5. Why did the log grow?

Investigate the workload.

Possible causes include:

```text
Large transaction
Long-running transaction
Bulk operation
Index maintenance
Replication-related behavior
Backup/log truncation issue
Incorrect recovery expectations
Application transaction problem
```

The key question is:

> **Why hasn't the log space become reusable?**

---

# 6. Check log reuse

You can inspect database log reuse information.

For example:

```sql
SELECT
    name,
    log_reuse_wait_desc
FROM sys.databases;
```

Suppose you see:

```text
OrdersDB
log_reuse_wait_desc = ACTIVE_TRANSACTION
```

That's a major clue.

You investigate active transactions and discover:

```text
Transaction started:
11 hours ago

Status:
Active

Rows being processed:
Millions
```

Now your chain is:

```text
Long transaction
      ↓
Log cannot be reused normally
      ↓
Log grows
      ↓
Storage pressure
      ↓
96% capacity
```

---

# 7. This is different from a database being genuinely too large

Imagine:

```text
OrdersDB data:
7.2 TB

Growth:
+50 GB/day
```

That could simply be normal business growth.

But:

```text
Log:
1.1 TB

Normal log:
200 GB
```

suggests a different problem.

So distinguish:

### Expected growth

```text
Business data increases
```

from:

### Abnormal growth

```text
Something caused unexpected consumption
```

---

# 8. Another scenario: data growth

Suppose instead you discover:

```text
OrdersDB:
5 TB → 7.2 TB in 3 months
```

And the biggest tables are:

```text
Sales.Orders       4.1 TB
Sales.OrderItems   2.3 TB
Audit.Events       600 GB
```

Now the problem is likely capacity planning.

You investigate:

```text
Why is Audit.Events 600 GB?
How quickly is it growing?
Is there a retention policy?
Can old data be archived?
Is this growth expected?
```

---

# 9. Data retention

Suppose the business requirement says:

> "Keep audit data for 90 days."

But the database contains:

```text
5 years of audit data
```

Then the solution isn't simply:

> "Buy more storage."

A better solution might be:

```text
Current data
    ↓
90-day retention
    ↓
Archive older data
    ↓
Delete/archive according to policy
```

This is a **data lifecycle problem**.

---

# 10. Another dangerous scenario: autogrowth

Imagine:

```text
Data file = 5 TB
Growth setting = 1 MB
```

That's a terrible configuration for a large production database.

The database may repeatedly grow by tiny amounts.

Better growth planning uses an appropriate growth strategy based on:

```text
Database size
Workload
Growth rate
Maintenance windows
Storage availability
```

Don't blindly choose a percentage or fixed size without understanding the workload.

---

# 11. Storage at 96% — what do you do immediately?

You have two separate objectives:

### Objective A — Prevent outage

```text
Critical storage pressure
       ↓
Determine safe immediate capacity relief
```

### Objective B — Find root cause

```text
Why did storage reach 96%?
```

Don't confuse them.

You may need immediate capacity action **while** continuing the investigation.

---

# 12. What NOT to do

Don't randomly:

```text
❌ Delete database files
❌ Shrink everything
❌ Kill random sessions
❌ Drop indexes
❌ Delete audit data
❌ Restart the MI
```

Production changes should be based on evidence and business approval where appropriate.

---

# 13. Capacity planning

Suppose storage is currently:

```text
7.2 TB
```

and growing:

```text
50 GB/day
```

Then:

```text
50 GB × 30 days
≈ 1.5 TB/month
```

So in six months:

```text
~9 TB additional growth
```

That changes your conversation from:

> "We have a storage alert."

to:

> "At the current growth rate, we have approximately X months of capacity remaining."

That's senior DBA thinking.

---

# 14. Build a growth dashboard

Track:

```text
Database size
File size
Daily growth
Weekly growth
Monthly growth
Free capacity
Largest tables
Log utilization
```

Then visualize:

```text
Storage
  ^
  |                     /
  |                  /
  |               /
  |            /
  |___________/____________> Time
```

You want to know **when you will hit the next capacity threshold**, not merely when you've already hit it.

---

# 15. Production response example

Suppose you determine:

```text
Storage = 96%

Cause:
Large uncommitted transaction

Business:
Nightly batch job

Transaction:
Running for 11 hours
```

Your response becomes:

```text
1. Identify transaction owner
2. Determine whether it is progressing
3. Determine business impact of stopping it
4. Assess rollback consequences
5. Prevent storage exhaustion
6. Coordinate with application/batch owner
7. Resolve transaction
8. Monitor log reuse
9. Validate storage recovery
10. Fix batch design
```

The permanent fix might be:

```text
Large transaction
       ↓
Smaller batches
       ↓
Periodic commits
       ↓
Shorter transaction duration
```

**Only if business consistency allows it.**

---

# 16. Interview question

An interviewer asks:

> **"Your Azure SQL Managed Instance is 96% full. What do you do?"**

A weak answer:

> "Increase storage."

A better answer:

> "First I identify which databases and file types are consuming the space and determine whether the growth is data, log, tempdb, or another workload-related issue. I check growth trends and log reuse information, investigate long-running transactions and other causes of abnormal growth, and take the safest immediate capacity action if an outage is imminent. Then I address the root cause and establish capacity monitoring and forecasting."

That's much stronger.

---

# Lesson 42 — Incident 6: Migration Failure

Now let's move to one of the most important skills for an Azure SQL DBA:

> **Migrating a large SQL Server workload to Managed Instance with minimal downtime.**

## Scenario

Company:

```text
ABC Retail
```

Source:

```text
SQL Server 2019
On-premises
OrdersDB = 5 TB
```

Target:

```text
Azure SQL Managed Instance
```

Requirement:

```text
Maximum application downtime = 15 minutes
```

The team performs a test migration.

Initial data copy:

```text
5 TB
```

Migration tool reports:

```text
Initial copy: SUCCESS
```

But during validation:

```text
Orders:
Source = 82,451,233
Target = 82,451,233
```

Looks good.

Then you discover:

```text
Orders created during migration:
Source = 1,250,000
Target = 0
```

The application is still actively writing to the source database.

---

# 17. The problem

You performed:

```text
Initial copy
```

but not continuous synchronization of changes.

So:

```text
SOURCE
82M rows
   |
   | Initial copy
   v
TARGET
82M rows

Meanwhile...

SOURCE receives:
+1.25M new/changed rows
```

Therefore target is behind.

---

# 18. Why can't you simply stop the application for hours?

Because business requirement says:

```text
Downtime ≤ 15 minutes
```

You can't do:

```text
Stop application
     ↓
Copy 5 TB again
     ↓
Wait several hours
     ↓
Start application
```

That violates the requirement.

---

# 19. The migration strategy

For a minimal-downtime migration, conceptually:

```text
Phase 1
Assess

Phase 2
Initial copy

Phase 3
Continuous change synchronization

Phase 4
Validate synchronization

Phase 5
Application freeze

Phase 6
Final synchronization

Phase 7
Cutover

Phase 8
Validation
```

The critical idea is:

> **Move most of the data while the application continues operating, then synchronize the remaining changes during the short cutover window.**

---

# 20. Cutover example

Suppose:

```text
10:00 PM
Initial synchronization complete

10:05 PM
Application enters maintenance mode

10:06 PM
No new writes to source

10:07 PM
Final changes synchronized

10:09 PM
Target validated

10:10 PM
Application points to MI

10:15 PM
Smoke testing complete
```

Downtime:

```text
~10 minutes
```

That meets:

```text
≤ 15 minutes
```

assuming the migration mechanism and validation actually achieve those timings.

---

# 21. But what else must be tested?

Database rows aren't enough.

You need:

```text
Logins/users
Permissions
Stored procedures
Functions
Views
SQL Agent jobs
Linked servers
SSIS
Reporting
Application connection strings
External dependencies
Security configuration
Performance
```

And especially:

> **Application behavior.**

A database can look perfect while the application still fails.

---

# 22. Migration validation

Use multiple layers.

### Data validation

```text
Row counts
Checksums/hashes where appropriate
Important aggregates
Recent transactions
```

### Functional validation

```text
Login
Create order
Update order
Cancel order
Payment workflow
Reporting
ETL
```

### Performance validation

```text
Query latency
CPU
IO
Concurrency
Transaction throughput
```

### Security validation

```text
Users
Roles
Permissions
Authentication
Auditing
```

---

# 23. Rollback planning

Suppose cutover succeeds.

Then 10 minutes later:

```text
Payment processing fails.
```

You cannot simply say:

> "Go back to the old SQL Server."

You must ask:

```text
What writes happened on MI?
Can they be reconciled?
Is the source still synchronized?
Can we safely reverse traffic?
```

This is why:

> **Rollback must be designed before cutover.**

---

# 24. Senior DBA interview question

> **"How would you migrate a 5-TB SQL Server database to Azure SQL Managed Instance with only 15 minutes of downtime?"**

A strong answer:

> "I would begin with compatibility and dependency assessment and validate network connectivity and throughput. I'd perform an initial bulk migration while the source remains online, then use a supported change-synchronization approach to keep the target current. Before cutover I'd validate data, application dependencies, security, jobs, and performance. During the cutover window I'd stop application writes, complete final synchronization, validate the target, redirect the application, and execute smoke tests. I'd also have predefined rollback criteria and a rollback plan before production cutover."

---

# Next: Incident 7 — Disaster Recovery Failure

This one is especially important because the business tells you:

> **"Our RPO is 5 minutes and RTO is 30 minutes."**

But during a DR test, you discover recovery actually takes **52 minutes**.

We'll investigate **why the DR design failed, how to measure RPO/RTO properly, and how to build a production DR runbook**.


# Lesson 43 — Incident 7: DR Test Failure

This is one of the most important Azure SQL DBA scenarios because **having a DR architecture does not automatically mean you meet the business RPO/RTO**.

## Production requirement

ABC Retail has documented:

```text
RPO ≤ 5 minutes
RTO ≤ 30 minutes
```

Their production SQL Managed Instance is in:

```text
Primary Region
```

They have a DR environment in:

```text
Secondary Region
```

The architecture looks healthy.

---

# 1. The DR test

The company performs a planned disaster recovery exercise.

At:

```text id="2ynp5k"
10:00 AM
```

the DBA team simulates a regional outage.

They record:

```text id="h6x0g2"
10:00
Disaster declared

10:05
DR team starts recovery

10:18
Database becomes available

10:35
Application starts

10:52
Business validation completed
```

Total recovery time:

```text id="n4czw0"
52 minutes
```

But the requirement is:

```text id="2v0p4q"
30 minutes
```

### Result:

❌ **RTO FAILED**

---

# 2. But there's another problem

During validation, the team discovers:

```text id="1nq5js"
Last transaction in primary:
10:00:00

Last transaction available in DR:
09:56:30
```

Data gap:

```text id="8s74iq"
3 minutes 30 seconds
```

Requirement:

```text id="d8xv2u"
RPO ≤ 5 minutes
```

So:

✅ RPO passed

❌ RTO failed

This distinction is extremely important.

---

# 3. What does RPO actually measure?

RPO means:

> **How much data loss the business can tolerate, measured in time.**

If:

```text id="f0p8h2"
Primary failure = 10:00

Latest recoverable data = 09:56
```

then approximately:

```text id="r1aj2w"
RPO = 4 minutes
```

If the business requirement is:

```text id="k5x5bn"
≤ 5 minutes
```

you meet it.

---

# 4. What does RTO measure?

RTO means:

> **How long the business can tolerate the service being unavailable.**

If:

```text id="b7f6g8"
Failure = 10:00
Business service restored = 10:52
```

then:

```text id="8w4q1e"
RTO = 52 minutes
```

Requirement:

```text id="7y2w4f"
≤ 30 minutes
```

Therefore:

❌ RTO failed.

---

# 5. Where did the 52 minutes go?

Break the recovery into components.

```text id="9p4y8v"
10:00
Disaster declared
       |
       +-- 5 min --> Decision/approval
       |
       +-- 13 min --> Database recovery
       |
       +-- 17 min --> Application startup
       |
       +-- 17 min --> Business validation
       |
       v
10:52
Service restored
```

Now you can see the problem.

It isn't necessarily the database alone.

---

# 6. DR is an end-to-end system

Your DR architecture includes:

```text id="5u0c1f"
Database
   +
Application
   +
Network
   +
Identity
   +
Secrets
   +
DNS
   +
Monitoring
   +
Operational procedures
```

If your database becomes available in 18 minutes but the application requires another 34 minutes, your business RTO is still 52 minutes.

---

# 7. Investigate the application

Suppose the application team tells you:

> "We have 15 microservices, and each one has to be manually restarted."

You discover:

```text id="p8lq8a"
Service 1 → 2 min
Service 2 → 3 min
Service 3 → 4 min
...
Service 15 → 20 min
```

This is an operational bottleneck.

Your DBA shouldn't simply say:

> "The database DR worked."

The business cares about:

> **When can customers use the application again?**

---

# 8. Improve the RTO

Possible improvements:

```text id="0r4t1q"
Automate application startup
Automate DNS changes
Preconfigure DR networking
Preconfigure identity/access
Automate database failover/recovery steps
Create scripted validation
Reduce manual approvals
Create clear ownership
```

The goal is:

```text id="k7u8v5"
Manual recovery
       ↓
Automated recovery
```

---

# 9. DR runbook

A good runbook might look like:

```text id="b7o7dj"
STEP 1
Declare disaster

STEP 2
Activate DR

STEP 3
Confirm database availability

STEP 4
Validate latest recoverable data

STEP 5
Activate networking

STEP 6
Start application services

STEP 7
Validate authentication

STEP 8
Run database smoke tests

STEP 9
Run application smoke tests

STEP 10
Business owner validates

STEP 11
Open service to customers
```

Each step should have:

```text id="4gk4pn"
Owner
Expected duration
Success criteria
Failure action
```

---

# 10. Measure, don't guess

A common mistake:

> "Our architecture should give us a 30-minute RTO."

That's not enough.

You need:

```text id="2z3f7h"
Design
   ↓
DR Test
   ↓
Measured RTO
   ↓
Compare with requirement
```

For example:

```text id="7h2j3k"
Target RTO:       30 min
Actual RTO:       52 min

Gap:              22 min
```

Now you have an engineering problem to solve.

---

# 11. RPO testing

Likewise:

```text id="7n5e1q"
Write known transaction
       ↓
Record timestamp
       ↓
Simulate failure
       ↓
Recover DR
       ↓
Find latest transaction
       ↓
Calculate data gap
```

Suppose:

```text id="j1t6xq"
Primary last commit: 10:00:00
DR latest commit:    09:56:30
```

RPO:

```text id="6k5p9n"
3m 30s
```

Requirement:

```text id="t3v7y9"
5m
```

Result:

✅ Pass.

---

# 12. Important interview trap

Interviewer:

> **"If the DR database is available in 20 minutes, is your RTO 20 minutes?"**

Answer:

**Not necessarily.**

Your business RTO is based on **service restoration**, not merely database availability.

For example:

```text id="6xq2hf"
Database available       20 min
Application available    40 min
Business validation      45 min
```

Business RTO is approximately:

```text id="c3x5ut"
45 minutes
```

---

# 13. Failback

Another senior-level consideration:

After the primary region is restored, what happens?

You need a **failback plan**.

Conceptually:

```text id="0h4vda"
PRIMARY
   X
   |
   v
DR REGION
   |
   | operates application
   |
   v
PRIMARY RECOVERED
   |
   v
Resynchronize
   |
   v
Validate
   |
   v
Controlled failback
```

Don't rush failback simply because the primary region is back online.

---

# 14. DR test frequency

A serious organization should periodically test DR.

Why?

Because:

```text id="n8c2hh"
Configuration changes
Application changes
Network changes
Security changes
Personnel changes
```

can silently invalidate an old DR plan.

A runbook that worked six months ago may fail today.

---

# 15. Senior interview answer

If asked:

> **"How do you validate RPO and RTO for Azure SQL Managed Instance?"**

A strong answer:

> "I define measurable business requirements first, then perform controlled DR exercises. For RPO, I record known transaction timestamps and compare the last recoverable transaction in the DR environment to the failure time. For RTO, I measure the complete time from disaster declaration through database recovery, application recovery, connectivity, and business validation. I compare the measured values against the requirements and remediate any gap. I also regularly retest because infrastructure, applications, networking, and operational procedures change."

That's the answer of someone thinking beyond just the database.

---

# Lesson 44 — Incident 8: Security Incident

Now the scenario gets more serious.

At **1:20 AM**, your security team alerts you:

```text id="x3b8g8"
Suspicious database login activity detected.
```

They report:

```text id="t4k7e0"
User: reporting_user
Multiple failed logins
Then successful login
Unexpected source
```

Shortly afterward, you discover:

```text id="4h6r6f"
SELECT activity against customer tables
```

The account normally accesses:

```text id="r7p5q0"
ReportingDB
```

but is now accessing:

```text id="s6c8v1"
CustomerDB
```

This is your next incident.

You need to determine:

```text id="d1z4se"
1. Is the account compromised?
2. What permissions does it have?
3. What data was accessed?
4. When did the activity start?
5. How do you contain it?
6. What evidence must be preserved?
7. How do you prevent recurrence?
```

The key lesson will be:

> **A DBA's job during a security incident is not simply to disable the login. You must contain the threat while preserving evidence and understanding the scope of access.**

# Lesson 45 — Incident 8: Suspected SQL Security Breach

This is a **security + DBA incident**, so your priorities change slightly.

Your objectives are:

```text
1. Contain
2. Preserve evidence
3. Determine scope
4. Protect data
5. Restore secure operation
6. Prevent recurrence
```

---

## Scenario

Security Operations reports:

```text
01:20 AM
Multiple failed logins

01:23 AM
Successful login

01:25 AM
Unexpected queries against CustomerDB
```

Account:

```text
reporting_user
```

Normally:

```text
reporting_user
       |
       v
ReportingDB
```

But tonight:

```text
reporting_user
       |
       +---- ReportingDB
       |
       +---- CustomerDB   ← unexpected
```

---

# 1. Don't immediately delete the account

Your first instinct might be:

```sql
DROP LOGIN reporting_user;
```

Don't rush.

You need to coordinate with your security/incident-response process because deleting or modifying the account can interfere with investigation.

First determine:

```text
Who authenticated?
When?
From where?
What did they execute?
What permissions did they have?
What data could they access?
```

---

# 2. Identify the authentication source

Investigate authentication activity through your available auditing and monitoring data.

You want:

```text
Timestamp
Login
Source
Database
Success/failure
Authentication method
```

Build a timeline:

```text
01:20
Failed login

01:21
Failed login

01:22
Failed login

01:23
Successful login

01:24
CustomerDB connection

01:25
SELECT against customer data
```

That timeline becomes important evidence.

---

# 3. Determine permissions

Find out what the account can actually do.

Think:

```text
Login
  ↓
User
  ↓
Database roles
  ↓
Object permissions
```

For example:

```text
reporting_user
       |
       +-- db_datareader
       |
       +-- SELECT on ReportingDB
```

But perhaps someone accidentally granted:

```text
db_owner
```

That would dramatically increase the risk.

---

# 4. Least privilege

Suppose you discover:

```text
reporting_user → db_owner
```

but the application only needs:

```text
SELECT
```

That's a major security design issue.

A reporting account shouldn't normally have broad administrative privileges merely because it was convenient.

The principle is:

> **Give identities only the permissions required to perform their job.**

---

# 5. Determine what data was accessed

This is critical.

You need to distinguish:

```text
Attempted access
```

from:

```text
Successful access
```

and:

```text
Data modification
```

For example:

```text
01:25
SELECT Customer.Name

01:26
SELECT Customer.Email

01:27
UPDATE Customer...
```

The last event is significantly more serious than read-only access.

Your investigation should establish:

```text
Read?
Write?
Delete?
Schema change?
Permission change?
```

---

# 6. Containment

Once your incident-response process authorizes containment, possible actions can include:

```text
Disable/revoke compromised credentials
Restrict network access
Remove excessive permissions
Terminate malicious sessions
Block suspicious access paths
Rotate affected secrets
```

But these actions should be coordinated with the security team and documented.

---

# 7. What if the attacker is still connected?

Suppose you find:

```text
Session 84
Login = reporting_user
Database = CustomerDB
Status = running
```

and it is executing suspicious queries.

Now you have an active threat.

You may need to terminate the session as part of containment.

But preserve the relevant evidence first where possible:

```text
Session ID
Login
Source
SQL text
Timestamp
Database
Application name
Client information
```

---

# 8. Audit trail

For a security incident, you want enough logging to answer:

> **Who did what, when, and from where?**

Depending on your Azure SQL MI design, use appropriate SQL/Azure auditing and monitoring capabilities.

Think:

```text
SQL activity
     ↓
Audit/logging
     ↓
Centralized monitoring
     ↓
Security team
```

Centralized security monitoring makes correlation easier.

---

# 9. Example investigation

Suppose security finds:

```text
reporting_user
Source: unexpected IP
Authentication: successful
Time: 01:23

Queries:
SELECT CustomerName...
SELECT Email...
SELECT Phone...
```

Then:

```text
01:28
UPDATE Customer...
```

You now have evidence suggesting:

```text
Unauthorized read access
+
Potential unauthorized modification
```

The incident severity increases.

---

# 10. Check for privilege escalation

Don't stop at the compromised account.

Ask:

> **Could the attacker have used this account to obtain additional privileges?**

Investigate:

```text
Role changes
Permission grants
New users/logins
New credentials
Configuration changes
Unexpected stored procedures
Schema changes
```

Security incidents can spread through privilege escalation.

---

# 11. Check application secrets

Suppose `reporting_user` credentials were stored in:

```text
Application configuration
```

You need to consider:

```text
Where is the credential stored?
Who can access it?
Was it exposed?
Should it be rotated?
```

A strong architecture reduces long-lived SQL passwords where possible and uses managed identity / Microsoft Entra-based authentication appropriately.

---

# 12. Root cause

After containment, determine:

> **How did this happen?**

Possibilities:

```text
Credential leaked
Password reused
Secret stored insecurely
Overprivileged account
Compromised application
Unexpected network exposure
Human error
```

If the root cause is:

```text
reporting_user
password stored in source code
```

then simply changing the password isn't enough.

You need to fix the credential-management process.

---

# 13. Prevention

A mature solution might include:

```text
Least privilege
Managed identities where appropriate
Microsoft Entra authentication
Secret management
Network restrictions
Auditing
Centralized security monitoring
Alerting
Periodic access reviews
Credential rotation
```

---

# 14. Interview question

### Interviewer:

> "A SQL login is suspected to be compromised. What do you do?"

### Strong answer:

> "I first coordinate with the security incident-response process and preserve relevant evidence. I establish a timeline of authentication and database activity, identify the source and active sessions, determine the account's effective permissions, and establish whether data was read or modified. I then contain the account and any active malicious sessions using the approved incident process, rotate affected credentials or secrets, and investigate privilege escalation and related identities. Finally, I remediate the root cause through least privilege, stronger authentication, secret management, auditing, and monitoring."

That demonstrates **security awareness + DBA operational thinking**.

---

# Lesson 46 — Incident 9: Application Can't Connect to SQL MI

Now imagine this:

At **8:30 AM**, developers deploy a new application version.

Immediately:

```text
Application errors:
"Cannot connect to database"
```

Database team checks:

```text
MI status:        Healthy
CPU:              22%
Storage:          45%
Database state:   Online
```

Developers say:

> "The database is down."

But you know:

> **Database health and application connectivity are two different things.**

---

## Connection path

Think about the entire path:

```text
Application
     |
     v
DNS
     |
     v
Network
     |
     v
Firewall / security controls
     |
     v
SQL endpoint
     |
     v
Authentication
     |
     v
Authorization
     |
     v
Database
```

Any layer can fail.

---

# 1. First question

Did the application change?

Yes.

Therefore:

```text
Recent deployment
       ↓
Connection failure
```

is an important clue.

Don't automatically change the MI.

---

# 2. Check connection configuration

Compare:

```text
Previous connection
        vs
New connection
```

Check:

```text
Server/endpoint
Port
Database name
Authentication method
Username/identity
Encryption settings
Connection properties
```

A common real-world mistake is:

```text
Production application
       ↓
Wrong database endpoint
```

---

# 3. Authentication

Suppose the new application uses Microsoft Entra authentication.

You discover:

```text
Application identity:
app-prod-mi
```

But that identity doesn't have the required database user/permissions.

Then:

```text
Network connection
        ↓
Successful

Authentication
        ↓
Successful/partial

Authorization
        ↓
FAILED
```

The application may report this simply as:

> "Database connection failed."

Your job is to identify the actual layer.

---

# 4. Network troubleshooting

If authentication isn't even reached, investigate connectivity:

```text
Application
   ↓
DNS resolution
   ↓
Network path
   ↓
Endpoint/port
   ↓
SQL MI
```

Ask:

```text
Can the application resolve the endpoint?
Can it reach the endpoint?
Are required network controls correct?
Did deployment change subnet/network configuration?
```

---

# 5. Don't restart SQL MI

If:

```text
MI healthy
Other applications connected
Existing sessions normal
```

and:

```text
Only newly deployed application fails
```

then restarting the MI is usually the wrong first response.

You have evidence pointing toward:

```text
Application configuration
Authentication
Authorization
Network path
```

not MI failure.

---

# 6. Compare working application

This is a powerful troubleshooting technique.

Suppose:

```text
Application A → works
Application B → fails
```

Compare:

```text
Connection string
Identity
Network location
Database
Authentication
Permissions
TLS settings
```

The differences often reveal the problem quickly.

---

# 7. Interview answer

> "I would first establish whether the MI itself is healthy and whether other clients can connect. Since the failure immediately followed an application deployment, I'd compare the old and new connection configuration and then trace the connection path through DNS, network connectivity, authentication, authorization, and database access. I wouldn't restart the MI without evidence that the MI itself is unhealthy."

Excellent DBA thinking.

---

# Lesson 47 — Final Incident: Production Disaster

This is the **final capstone incident**.

You are now the senior DBA.

At:

```text id="n9v1x2"
09:15 AM
```

Azure reports a major regional outage.

Your production MI is unavailable.

Business says:

> **"We need the application back within 30 minutes and cannot lose more than 5 minutes of data."**

Your DR environment is available in the secondary region.

You have:

```text id="7q2z4w"
DR database
DR application
Network
Identity
Monitoring
Runbook
```

---

## Your task

You must execute the incident mentally:

```text id="u3p6z9"
09:15
Disaster declared
      ↓
?
      ↓
?
      ↓
?
      ↓
?
      ↓
Application restored
```

You need to explain:

### Step 1 — Declare and assess

```text
What confirms this is a regional disaster?
Who declares DR?
Who owns the decision?
```

### Step 2 — Determine RPO

```text
What is the latest recoverable data?
How do you determine whether the 5-minute RPO is met?
```

### Step 3 — Activate DR

```text
Database
Network
Identity
Application
```

What happens first?

### Step 4 — Validate database

Check:

```text
Connectivity
Database state
Recent transactions
Critical tables
Users/permissions
```

### Step 5 — Start application

Validate:

```text
Connection
Authentication
Business workflows
```

### Step 6 — Measure RTO

Record:

```text
Disaster time
Database available
Application available
Business validation complete
```

### Step 7 — Communicate

Tell management:

```text
Current status
Estimated recovery
Actual RPO
Actual RTO
Known limitations
```

### Step 8 — After recovery

Don't immediately celebrate.

You need:

```text
Incident review
Root cause
Data validation
Performance monitoring
Failback planning
DR runbook updates
```

---

# The complete DBA framework

You've now covered the major production scenarios:

```text
                    SQL MI DBA
                        |
       +----------------+----------------+
       |                |                |
   PERFORMANCE       OPERATIONS        RESILIENCE
       |                |                |
       v                v                v
   CPU              Blocking             HA
   Queries          Deadlocks             DR
   Plans            Storage               RPO
   Indexes          Migration              RTO
   Statistics       Security             Failover
       |                |                |
       +----------------+----------------+
                        |
                        v
                  ROOT CAUSE
                        |
                        v
                 PERMANENT FIX
```

## The most important habit

Whenever production breaks, don't jump directly to a command.

Think:

> **What is the evidence telling me?**

Then:

> **What is the safest action that restores service while preserving data and allowing me to find the root cause?**

That's the mindset that separates a **SQL developer who knows Azure** from a **production Azure SQL DBA**.

