# Lesson 85 — Azure SQL Managed Instance Monitoring & Performance

Now we move into one of the **most important areas for a SQL Server DBA**:

> **How do you know SQL Managed Instance is healthy, and how do you find the root cause when the application becomes slow?**

A good DBA doesn't just look at CPU.

You need to correlate:

```text
Application
    ↓
SQL requests
    ↓
Queries
    ↓
CPU / Memory / IO
    ↓
Waits
    ↓
Blocking / Deadlocks
    ↓
Storage / Transaction Log
    ↓
Azure infrastructure
```

---

# 1. Monitoring vs Performance Troubleshooting

These are related but different.

### Monitoring

Answers:

> **"Is something wrong?"**

Example:

```text
CPU > 90%
```

### Performance troubleshooting

Answers:

> **"Why is it wrong?"**

Example:

```text
CPU > 90%
     ↓
Which queries?
     ↓
Which application?
     ↓
Why are those queries expensive?
     ↓
Bad execution plan?
     ↓
Missing index?
     ↓
Large scan?
```

This distinction is extremely important.

---

# 2. What should you monitor?

For SQL MI, think about these major categories:

```text
┌─────────────────────────────┐
│ SQL MI MONITORING            │
├─────────────────────────────┤
│ CPU                          │
│ Memory                       │
│ Storage                      │
│ Data IO                      │
│ Log IO                       │
│ Connections                  │
│ Query performance            │
│ Wait statistics              │
│ Blocking                     │
│ Deadlocks                    │
│ Errors                       │
│ Availability                │
│ Backup/recovery              │
│ Security events              │
└─────────────────────────────┘
```

---

# 3. Azure Monitor

Azure Monitor is the broader Azure monitoring platform.

Conceptually:

```text
SQL MI
   |
   +---- Metrics
   |
   +---- Logs
   |
   +---- Alerts
   |
   +---- Diagnostics
             |
             ↓
        Azure Monitor
```

You can use Azure Monitor capabilities to monitor Azure resources and build alerting/observability around them.

---

# 4. Metrics vs Logs

This distinction is important.

### Metrics

Usually numerical time-series data.

Example:

```text
CPU
80%
82%
85%
92%
95%
```

Good for:

```text
Dashboards
Alerts
Trend analysis
```

### Logs

Detailed event/activity information.

Example:

```text
Login event
Error
Query-related diagnostic information
Audit event
```

Good for:

```text
Investigation
Correlation
Historical analysis
```

---

# 5. Real-time monitoring architecture

A typical monitoring flow:

```text
                SQL MI
                  |
        +---------+---------+
        |                   |
      Metrics              Logs
        |                   |
        ↓                   ↓
 Azure Monitor        Log Analytics
        |                   |
        +---------+---------+
                  |
               Alerts
                  |
                  ↓
             DBA / SOC
```

---

# 6. CPU monitoring

CPU is one of the first things DBAs check.

Suppose:

```text
09:00 → 30%
10:00 → 35%
11:00 → 40%
12:00 → 45%
13:00 → 92%
```

Something changed.

But remember:

> **High CPU is a symptom, not necessarily the root cause.**

---

# 7. High CPU scenario

Application becomes slow.

You see:

```text
CPU = 95%
```

Don't immediately scale the instance.

First ask:

```text
Which queries are consuming CPU?
```

Then:

```text
Why?
```

Possible causes:

```text
Missing index
Bad execution plan
Large table scan
Poor query design
Parameter sensitivity
Data growth
Unexpected workload
Parallelism
```

---

# 8. CPU troubleshooting flow

```text
High CPU
   ↓
Identify time window
   ↓
Identify expensive queries
   ↓
Identify application/user
   ↓
Check execution plan
   ↓
Check waits
   ↓
Check indexes/statistics
   ↓
Determine root cause
```

---

# 9. Example

At 2:30 PM:

```text
CPU = 98%
```

Top query:

```sql
SELECT *
FROM Orders
WHERE CustomerID = @CustomerID;
```

Suppose `Orders` has:

```text
500 million rows
```

and there is no useful index on `CustomerID`.

The database may perform a huge scan.

Result:

```text
CPU ↑
IO ↑
Latency ↑
```

---

# 10. Memory

Memory is another major resource.

Think:

```text
SQL Server
   |
   ↓
Buffer pool
   |
   ↓
Data pages cached in memory
```

SQL Server uses memory heavily to avoid repeatedly reading data from storage.

---

# 11. Why memory matters

Suppose a frequently accessed table is:

```text
100 GB
```

but only a small portion can remain cached.

The system may need more physical reads.

Conceptually:

```text
Insufficient effective memory
        ↓
More reads from storage
        ↓
IO pressure
        ↓
Query latency
```

Memory problems can therefore appear as IO problems.

---

# 12. Don't blindly say "memory is high"

SQL Server intentionally uses memory.

Seeing:

```text
Memory = 90%
```

doesn't automatically mean there's a problem.

You need to determine:

```text
Is memory pressure occurring?
Are queries waiting?
Is the workload affected?
Is there external pressure?
```

---

# 13. Data IO

SQL workloads read and write data pages.

Think:

```text
Query
  ↓
Buffer pool
  ↓
Data pages
  ↓
Storage
```

If a query needs many pages from storage:

```text
Data IO ↑
```

---

# 14. Example: table scan

Suppose:

```text
Customer table = 200 GB
```

Query:

```sql
SELECT *
FROM Customers
WHERE Email = @Email;
```

If there is no suitable index:

```text
Query
 ↓
Scan 200 GB
 ↓
Huge IO
 ↓
Slow query
```

An appropriate index could dramatically reduce the amount of data accessed.

---

# 15. Log IO

The transaction log is different from data files.

Think:

```text
UPDATE
INSERT
DELETE
    ↓
Transaction Log
```

SQL Server must maintain transaction durability and recovery information.

Heavy write workloads can produce substantial log activity.

---

# 16. Example

Suppose an ETL process runs:

```text
01:00 AM
```

and updates:

```text
500 million rows
```

You may see:

```text
Log generation ↑
Log IO ↑
Transaction log usage ↑
```

This can affect the workload and potentially cause operational problems if log space becomes constrained.

---

# 17. Transaction log misconception

A common mistake:

> "The transaction log is 80% used, so SQL is broken."

Not necessarily.

You need to ask:

```text
Why is the log not truncating?
```

Possible causes include:

```text
Long-running transaction
Availability/replication-related conditions
Backup/recovery considerations
Other SQL Server log reuse blockers
```

The correct investigation is more important than the percentage itself.

---

# 18. Wait statistics

This is one of the **most powerful SQL Server performance concepts**.

A query can be slow because it is:

```text
Running
```

or because it is:

```text
Waiting
```

Wait statistics help answer:

> **What is SQL Server waiting for?**

---

# 19. Example

Suppose a query takes:

```text
30 seconds
```

You might discover:

```text
CPU time = 2 seconds
Wait time = 28 seconds
```

Then the question becomes:

> What was it waiting for?

---

# 20. Common wait categories

You don't need to memorize every wait type immediately.

Understand broad categories:

```text
CPU-related
IO-related
Locking/blocking
Memory-related
Parallelism
Network/client-related
Log-related
```

Examples you may encounter include:

```text
PAGEIOLATCH_*
LCK_M_*
WRITELOG
RESOURCE_SEMAPHORE
CXPACKET
CXCONSUMER
ASYNC_NETWORK_IO
```

The exact interpretation depends on workload and context.

---

# 21. PAGEIOLATCH

Broadly, PAGEIOLATCH waits are associated with waiting for data pages to be read from storage.

Conceptually:

```text
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

If these waits dominate, investigate:

```text
IO performance
Query access patterns
Indexes
Memory
Large scans
```

---

# 22. LCK_M_* waits

These indicate lock-related waits.

Example:

```text
Transaction A
    |
    ↓
Updates row
    |
    +---- holds lock

Transaction B
    |
    ↓
Needs same row
    |
    ↓
WAIT
```

This is blocking.

---

# 23. Blocking

Blocking is not automatically bad.

Short blocking can be completely normal.

Problematic blocking is typically:

```text
Long duration
High impact
Repeated
Causing user-visible delays
```

---

# 24. Blocking example

```text
Session 51
   |
   | UPDATE Orders
   |
   +---- transaction remains open
             |
             ↓
          lock held
             |
             ↓
Session 72
   |
   | SELECT Orders
   |
   ↓
WAITING
```

Now users report:

> "The application is slow."

CPU may be only:

```text
25%
```

So CPU monitoring alone would miss the problem.

---

# 25. Root blocker

Imagine:

```text
Session 51
    ↓
blocks 72
    ↓
blocks 73
    ↓
blocks 74
    ↓
blocks 75
```

Session 51 is the **root blocker**.

You want to identify:

```text
Root blocker
    ↓
Why is it holding the lock?
    ↓
How long?
    ↓
Who started the transaction?
```

---

# 26. Common blocking causes

```text
Long transaction
Uncommitted transaction
Large UPDATE
Large DELETE
Poor transaction design
Application connection issue
Missing indexes
Unexpected batch process
```

---

# 27. Deadlocks

Blocking:

```text
A → waits for B
```

Deadlock:

```text
A waits for B
B waits for A
```

Conceptually:

```text
Transaction A
    |
    | holds Resource 1
    ↓
waits for Resource 2
          ↑
          |
holds Resource 2
    |
Transaction B
```

SQL Server detects the cycle and chooses a victim.

---

# 28. Deadlock example

Transaction A:

```sql
UPDATE Customers
SET Name = 'A'
WHERE CustomerID = 1;
```

Then:

```sql
UPDATE Orders
SET Status = 'X'
WHERE CustomerID = 1;
```

Transaction B may acquire locks in the opposite order.

Now:

```text
A holds Customer
A wants Order

B holds Order
B wants Customer
```

Deadlock.

---

# 29. Deadlock troubleshooting

Don't simply say:

> "Increase timeout."

That doesn't solve the underlying problem.

Investigate:

```text
Deadlock graph
Queries involved
Resources involved
Lock order
Indexes
Transaction scope
Application retry behavior
```

---

# 30. Query Store

**Query Store** is one of your best tools for query-performance analysis.

It helps preserve query-related performance information so you can analyze changes over time.

Think:

```text
Query Store
     |
     +---- Query
     +---- Plan
     +---- Runtime statistics
     +---- Historical performance
```

---

# 31. Why Query Store is valuable

Suppose:

```text
Monday
Query = 500 ms
```

Tuesday:

```text
Query = 20 seconds
```

You want to know:

```text
What changed?
```

Query Store can help you investigate:

```text
Plan change
Runtime change
Regression
Query frequency
Duration
CPU
```

---

# 32. Query performance regression

Classic production incident:

```text
10:00 AM
Query = 300 ms

10:30 AM
Query = 15 sec
```

Possible reason:

```text
Execution plan changed
```

Query Store can help identify this type of regression.

---

# 33. Execution plans

Suppose query:

```sql
SELECT *
FROM Orders
WHERE CustomerID = 12345;
```

Plan A:

```text
Index Seek
   ↓
Small number of rows
```

Plan B:

```text
Table Scan
   ↓
Millions of rows
```

Plan B may be dramatically slower.

---

# 34. Why plans change

Possible reasons include:

```text
Statistics changes
Data distribution changes
Parameter sensitivity
Schema changes
Index changes
Engine behavior
Compilation/recompilation
```

Query Store helps you correlate plan and performance history.

---

# 35. Parameter sensitivity

Suppose:

```sql
CREATE PROCEDURE GetOrders
    @CustomerID INT
AS
SELECT *
FROM Orders
WHERE CustomerID = @CustomerID;
```

Customer 1:

```text
10 rows
```

Customer 2:

```text
10 million rows
```

A plan optimized for one value may not be ideal for another.

This can cause performance instability.

---

# 36. Missing indexes

Suppose execution plan shows:

```text
Table Scan
```

and your query filters on:

```text
CustomerID
```

You might investigate an index such as:

```sql
CREATE INDEX IX_Orders_CustomerID
ON dbo.Orders(CustomerID);
```

But don't blindly create every suggested index.

Indexes have costs:

```text
Storage
INSERT overhead
UPDATE overhead
DELETE overhead
Maintenance
Compilation considerations
```

A DBA must balance read performance with write cost.

---

# 37. Statistics

Statistics help the optimizer estimate data distribution.

Conceptually:

```text
Statistics
     ↓
Optimizer estimates rows
     ↓
Chooses execution plan
```

If estimates are poor:

```text
Bad estimate
   ↓
Bad plan
   ↓
Poor performance
```

---

# 38. Example of bad estimation

Actual:

```text
1,000,000 rows
```

Optimizer estimates:

```text
10 rows
```

The optimizer may choose an access strategy suitable for a tiny result.

But the query actually processes:

```text
1,000,000 rows
```

Performance suffers.

---

# 39. CPU + IO together

Suppose:

```text
CPU = 95%
IO = 90%
```

Possible causes:

```text
Large scans
High workload
Poor indexing
Expensive queries
```

But if:

```text
CPU = 25%
IO = 95%
```

you might focus more heavily on:

```text
IO bottlenecks
Storage
Query access patterns
Memory/cache effectiveness
```

Don't interpret metrics in isolation.

---

# 40. Production scenario — application suddenly slow

Let's do a complete DBA investigation.

At:

```text
2:30 PM
```

users say:

> "The application is very slow."

---

## Step 1 — Confirm scope

Ask:

```text
Is everything slow?
One page?
One transaction?
One database?
One application?
All applications?
```

This immediately narrows the investigation.

---

# 41. Step 2 — Check SQL MI health

Check:

```text
Instance availability
CPU
Memory
Storage
IO
Connections
```

Suppose:

```text
SQL MI = healthy
CPU = 92%
```

Now investigate CPU.

---

# 42. Step 3 — Identify top queries

Find:

```text
Top CPU-consuming queries
Top duration queries
Top execution-count queries
```

Suppose:

```text
Query A
CPU = 70%

Query B
CPU = 15%

Query C
CPU = 5%
```

Query A deserves immediate attention.

---

# 43. Step 4 — Look at execution plan

Query A:

```sql
SELECT *
FROM Orders
WHERE CustomerID = @CustomerID;
```

Plan:

```text
Clustered Index Scan
```

Table:

```text
500 million rows
```

Now you have a strong lead.

---

# 44. Step 5 — Check index

There is no suitable index on:

```text
CustomerID
```

You evaluate:

```sql
CREATE INDEX IX_Orders_CustomerID
ON dbo.Orders(CustomerID);
```

But before creating it, you consider:

```text
How often is query executed?
How selective is CustomerID?
What is the write workload?
Will the index materially reduce IO?
How much storage will it consume?
```

---

# 45. Step 6 — Validate

After implementing an approved fix, measure:

```text
CPU
IO
Query duration
Execution count
Application response time
```

Before:

```text
Query = 15 sec
CPU = 92%
```

After:

```text
Query = 300 ms
CPU = 45%
```

Now you have evidence.

---

# 46. Don't fix without measuring

A common junior DBA mistake:

```text
Problem
 ↓
Create index
 ↓
Done
```

Senior approach:

```text
Problem
 ↓
Baseline
 ↓
Hypothesis
 ↓
Change
 ↓
Measure
 ↓
Validate
```

---

# 47. Azure Monitor alerts

You can configure alerts around important metrics/conditions.

Examples:

```text
CPU sustained above threshold
Storage approaching limit
Unexpected resource behavior
Availability-related conditions
```

But avoid alerting on everything.

If you create:

```text
500 alerts
```

your DBA team may start ignoring alerts.

---

# 48. Good alert design

An alert should answer:

```text
What happened?
How serious?
Who should respond?
What action should they take?
```

Example:

```text
Production SQL MI
CPU > 90%
for 10 minutes
```

is more useful than:

```text
CPU = 90%
```

for one transient spike.

---

# 49. Alert fatigue

Bad:

```text
CPU > 70%
```

for every five-minute spike.

Result:

```text
100 alerts/day
```

Eventually:

```text
DBA ignores alerts
```

Better:

```text
CPU > 90%
AND sustained
AND production
```

with appropriate severity.

---

# 50. Log Analytics

Log Analytics can centralize and query logs from supported Azure resources.

Conceptually:

```text
SQL MI
  |
  ↓
Diagnostic data
  |
  ↓
Log Analytics workspace
  |
  ↓
KQL queries
```

This becomes especially useful when you want to correlate SQL-related events with other Azure resources.

---

# 51. Why correlation matters

Suppose:

```text
2:30 PM
SQL CPU spikes
```

At the same time:

```text
2:30 PM
Application deployment occurred
```

And:

```text
2:31 PM
Traffic doubled
```

The root cause might be an application release rather than an infrastructure failure.

Cross-resource monitoring helps you see the bigger picture.

---

# 52. Performance timeline

Build a timeline:

```text
2:00 PM
Normal

2:20 PM
Deployment

2:25 PM
Traffic ↑

2:30 PM
CPU ↑

2:31 PM
Query duration ↑

2:35 PM
Users report slowness
```

This is much stronger than simply saying:

> "CPU was high."

---

# 53. Baselines

You need to know what "normal" means.

For example:

```text
Normal CPU:
35–55%

Normal IO:
40–60%

Normal query duration:
200 ms
```

Then:

```text
Today:
CPU = 95%
IO = 90%
Query = 12 sec
```

The deviation is obvious.

---

# 54. Daily workload patterns

Production databases often have predictable patterns.

Example:

```text
00:00 → ETL
06:00 → Users start
09:00 → High workload
12:00 → Moderate
18:00 → Batch processing
23:00 → Backup/maintenance
```

If CPU hits:

```text
90%
```

during a known nightly batch, it may be expected.

If it hits:

```text
90%
```

at 3:00 AM unexpectedly, investigate.

---

# 55. Blocking scenario

Let's analyze another incident.

Users report:

> "Orders page is frozen."

Metrics:

```text
CPU = 30%
```

You might initially think:

> SQL looks healthy.

But blocking shows:

```text
Session 51
   ↓
holds lock for 20 minutes
   ↓
Session 72
   ↓
Session 73
   ↓
Session 74
```

The problem is blocking, not CPU.

---

# 56. Blocking investigation

Find:

```text
Root blocker
Blocking duration
Blocked sessions
SQL text
Transaction start time
Application/user
```

Then ask:

```text
Why has the transaction remained open?
```

Possible answer:

```text
Application began transaction
User abandoned request
Connection remained open
Transaction never committed
```

Now the application design becomes part of the investigation.

---

# 57. Deadlock scenario

At 3 PM:

```text
Deadlocks suddenly increase
```

Query Store shows:

```text
UPDATE Customers
UPDATE Orders
```

You inspect deadlock information and find:

```text
Transaction A:
Customer → Order

Transaction B:
Order → Customer
```

Both acquire locks in different orders.

A likely fix is to make transaction access order consistent, alongside other appropriate query/index/transaction changes.

---

# 58. High log usage scenario

Suppose:

```text
Transaction log usage = very high
```

Don't immediately shrink the log.

First investigate:

```text
Why can't log space be reused?
```

Possible cause:

```text
Long-running transaction
```

Find:

```text
Transaction start time
Session
SQL statement
Application
```

Then address the root cause.

---

# 59. Why repeated log shrinking is bad

A common anti-pattern:

```text
Log grows
 ↓
Shrink
 ↓
Log grows
 ↓
Shrink
 ↓
Log grows
```

This treats the symptom.

You need to understand:

```text
Normal workload
Log sizing
Growth configuration
Log reuse
Transaction behavior
```

---

# 60. Storage monitoring

Monitor:

```text
Data storage
Transaction log
Available capacity
Growth trends
```

Example:

```text
January → 2 TB
February → 2.2 TB
March → 2.5 TB
April → 2.8 TB
```

You can forecast:

> "At this growth rate, we'll approach the storage limit in approximately X months."

That's proactive DBA work.

---

# 61. Capacity planning

Don't wait until:

```text
Storage = 99%
```

Instead:

```text
Current usage
      ↓
Growth rate
      ↓
Forecast
      ↓
Plan capacity
```

This is especially important for production systems.

---

# 62. Query Store + Azure Monitor

These tools answer different questions.

### Azure Monitor

```text
How is the instance behaving?
```

### Query Store

```text
Which queries are behaving badly?
```

### SQL diagnostic tools

```text
What is happening inside SQL right now?
```

Use them together.

---

# 63. Real-time vs historical

For a live incident:

```text
What's happening RIGHT NOW?
```

you investigate:

```text
Active requests
Blocking
Current waits
Current CPU/IO
```

For historical analysis:

```text
What happened yesterday at 2:30 PM?
```

you rely more on:

```text
Query Store
Azure Monitor
Logs
Historical metrics
```

---

# 64. Performance troubleshooting framework

Memorize this:

```text
              SLOW APPLICATION
                     |
                     ↓
              Define the scope
                     |
                     ↓
              Check SQL health
                     |
          +----------+----------+
          |          |          |
         CPU        IO       Blocking
          |          |          |
          +----------+----------+
                     |
                     ↓
               Wait statistics
                     |
                     ↓
              Top queries
                     |
                     ↓
             Execution plans
                     |
                     ↓
          Indexes / statistics
                     |
                     ↓
               Validate fix
```

---

# 65. What NOT to do

When users say SQL is slow:

### Don't immediately:

```text
Restart SQL MI
```

### Don't blindly:

```text
Create indexes
```

### Don't blindly:

```text
Scale compute
```

### Don't blindly:

```text
Kill sessions
```

### Don't blindly:

```text
Shrink database/log
```

First determine the root cause.

---

# 66. Scaling vs tuning

Suppose CPU is 95%.

You have two choices:

### Scale

```text
More compute
```

### Tune

```text
Fix inefficient query
```

Sometimes scaling is appropriate.

Sometimes tuning is the better long-term solution.

---

# 67. Example

Before:

```text
CPU = 95%
```

Bad query:

```text
Table scan
500M rows
```

You scale the instance.

Now:

```text
CPU = 70%
```

But the query is still inefficient.

Tomorrow:

```text
Data grows
```

CPU goes back to:

```text
95%
```

The root problem remains.

---

# 68. Better approach

```text
Identify expensive query
       ↓
Fix plan/index/query
       ↓
Reduce workload
       ↓
Then reassess capacity
```

This is sustainable performance engineering.

---

# 69. Interview question

### "CPU is 95%. What do you do?"

Strong answer:

> "I wouldn't immediately scale the instance. I'd identify the time window, examine top CPU-consuming queries, inspect their execution plans and wait statistics, check for recent plan regressions, and evaluate indexes/statistics and workload changes. If the workload is legitimately larger than the provisioned capacity after tuning, then I'd consider scaling."

---

# 70. Interview question

### "CPU is normal but users say SQL is slow. What could be happening?"

Excellent answer:

> "I would investigate blocking, deadlocks, IO latency, memory pressure, waits, network/client waits, query regressions, and application-side issues. CPU alone doesn't determine database performance."

---

# 71. Interview question

### "What is Query Store used for?"

Answer:

> "Query Store captures historical query and execution-plan performance information, making it useful for identifying expensive queries, plan changes, and performance regressions over time."

---

# 72. Interview question

### "What is the difference between blocking and deadlocking?"

### Blocking

```text
A holds lock
B waits
```

### Deadlock

```text
A waits for B
B waits for A
```

SQL Server detects a deadlock and terminates one transaction as the victim.

---

# 73. Interview question

### "Why shouldn't you automatically shrink the transaction log?"

Because shrinking doesn't solve the reason the log keeps growing.

Correct approach:

```text
Find why log reuse is blocked
       ↓
Fix root cause
       ↓
Size log appropriately
       ↓
Monitor growth
```

---

# 74. Production monitoring dashboard

A useful DBA dashboard might show:

```text
┌──────────────────────────────────┐
│ SQL MI PRODUCTION                │
├──────────────────────────────────┤
│ CPU                 48%          │
│ Memory              Healthy      │
│ Storage             62%          │
│ Log usage           31%          │
│ Active connections  1,250        │
│ Blocking sessions   0            │
│ Deadlocks           0            │
│ Top CPU query       Query #123   │
│ Top duration        Query #456   │
│ Alerts              0            │
└──────────────────────────────────┘
```

This gives you a quick health view.

---

# 75. The most important production metrics

If you're preparing for a SQL DBA role, prioritize:

```text
1. CPU
2. Memory
3. Data IO
4. Log IO
5. Storage
6. Wait statistics
7. Blocking
8. Deadlocks
9. Query duration
10. Query CPU
11. Query execution count
12. Connection count
13. Errors
```

You don't need to memorize every Azure metric name immediately.

Understand what the metrics mean.

---

# 76. Complete incident example

Let's combine everything.

### 2:30 PM

Users report:

> "Orders are slow."

### Azure Monitor

```text
CPU = 40%
```

So CPU isn't the obvious problem.

### IO

```text
IO latency elevated
```

### Waits

```text
PAGEIOLATCH-related waits elevated
```

### Query Store

Top query:

```sql
SELECT *
FROM Orders
WHERE CustomerID = @CustomerID;
```

### Execution plan

```text
Large table scan
```

### Investigation

No useful index exists.

### Solution

After appropriate testing/change approval:

```sql
CREATE INDEX IX_Orders_CustomerID
ON dbo.Orders(CustomerID);
```

### Result

```text
IO ↓
Query duration ↓
Application latency ↓
```

That's a proper performance investigation.

---

# 77. Another complete incident

### Users report

> "Application completely hangs."

Metrics:

```text
CPU = 25%
```

You inspect active sessions.

Find:

```text
Session 100
transaction open for 45 minutes
```

Blocking tree:

```text
100
 |
 +-- 101
 |
 +-- 102
 |
 +-- 103
 |
 +-- 104
```

Root blocker:

```text
Session 100
```

You investigate the application transaction.

Root cause:

```text
Application failed to commit/rollback after an exception.
```

The SQL MI infrastructure wasn't the problem.

---

# 78. Senior-level thinking

A junior DBA asks:

> "Which metric is high?"

A senior DBA asks:

> **"What changed, when did it change, which workload was affected, and what evidence connects that change to the user impact?"**

That's the mindset you want.

---

# 79. Your Lesson 85 checklist

```text
MONITORING
☐ Azure Monitor
☐ Metrics
☐ Logs
☐ Alerts
☐ Log Analytics

RESOURCE
☐ CPU
☐ Memory
☐ Data IO
☐ Log IO
☐ Storage

SQL
☐ Wait statistics
☐ Blocking
☐ Deadlocks
☐ Active requests
☐ Query Store
☐ Execution plans
☐ Statistics
☐ Indexes

OPERATIONS
☐ Baselines
☐ Capacity planning
☐ Alert thresholds
☐ Historical analysis
☐ Root-cause analysis
```

---

# 80. The golden rule

When a production application becomes slow:

```text
Don't ask only:

"Is SQL CPU high?"
```

Ask:

```text
WHAT changed?
     ↓
WHEN did it change?
     ↓
WHO/WHAT is consuming resources?
     ↓
WHAT is SQL waiting for?
     ↓
WHICH query is responsible?
     ↓
WHY is that query slow?
     ↓
WHAT changed in the plan/data/index/workload?
     ↓
WHAT fix will address the root cause?
     ↓
DID performance improve after the fix?
```

That is the **production DBA troubleshooting mindset**.

---

## Next — Lesson 86: Azure SQL Managed Instance High Availability & Disaster Recovery

We'll cover:

* HA vs DR
* Built-in SQL MI high availability
* Service-level resilience
* Availability zones
* Failover behavior
* Business continuity
* RPO
* RTO
* Automated backups
* Point-in-time restore
* Geo-restore
* Failover groups
* Cross-region architecture
* Regional disaster scenarios
* Planned vs unplanned failover
* What happens to connections during failover
* Application retry logic
* Real production DR architecture
* **Full interview scenario: "The Azure region hosting your SQL MI goes down. Your application has to be restored within 30 minutes with no more than 5 minutes of data loss. Design the solution."**
