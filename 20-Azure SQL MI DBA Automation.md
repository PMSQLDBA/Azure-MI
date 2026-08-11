# Lesson 25 — Azure SQL MI DBA Automation

A production DBA should not manually check the same 20 things every morning.

The goal is:

```text
Manual work
    ↓
Script
    ↓
Automated health check
    ↓
Alert
    ↓
DBA investigates only when needed
```

---

## 1. What should you automate?

For SQL Managed Instance, common candidates are:

```text
Health checks
Performance reports
Database inventory
Storage monitoring
Failed jobs
Blocking detection
Long-running queries
Security reviews
Permission reports
Migration validation
```

A simple architecture:

```text
              Scheduler
                  |
                  v
             Health Script
                  |
       +----------+----------+
       |          |          |
       v          v          v
     SQL MI     Azure      Logs
       |          |          |
       +----------+----------+
                  |
                  v
               Alert
                  |
                  v
                 DBA
```

---

# 2. T-SQL automation

Let's start with something simple.

### Find currently running requests

```sql
SELECT
    session_id,
    status,
    command,
    cpu_time,
    total_elapsed_time,
    logical_reads,
    blocking_session_id
FROM sys.dm_exec_requests
WHERE session_id <> @@SPID;
```

This gives you an immediate snapshot of activity.

---

# 3. Find blocking

```sql
SELECT
    session_id,
    blocking_session_id,
    wait_type,
    wait_time,
    status
FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0;
```

If you see:

```text
Session 100 → blocked by 57
Session 101 → blocked by 57
Session 102 → blocked by 57
```

you should investigate:

```text
Session 57
```

That's potentially your head blocker.

---

# 4. Find long-running queries

```sql
SELECT
    session_id,
    start_time,
    status,
    command,
    total_elapsed_time,
    cpu_time
FROM sys.dm_exec_requests
WHERE total_elapsed_time > 60000;
```

Here:

```text
60000 ms = 60 seconds
```

In production, the appropriate threshold depends on the workload.

Don't automatically kill every query exceeding one minute.

---

# 5. Find expensive queries

Query Store is particularly useful for historical analysis.

You want to identify:

```text
Highest CPU
Highest duration
Highest execution count
Highest logical reads
Worst regressions
```

Conceptually:

```text
Query Store
    |
    +-- CPU
    +-- Duration
    +-- Reads
    +-- Executions
    +-- Plans
```

---

# 6. Why Query Store is so valuable

Suppose a query was slow yesterday but is no longer running.

A DMV snapshot may not tell you enough.

Query Store can preserve historical workload information.

So:

```text
DMVs
=
What's happening now?
```

while:

```text
Query Store
=
What happened over time?
```

That distinction is extremely important.

---

# 7. Database inventory script

A DBA may want a quick inventory:

```sql
SELECT
    name,
    state_desc,
    recovery_model_desc,
    compatibility_level
FROM sys.databases
ORDER BY name;
```

This lets you quickly identify:

```text
Database
State
Recovery model
Compatibility level
```

---

# 8. Database size monitoring

You need to understand database growth.

For example:

```text
Today
SalesDB = 2.0 TB

Next month
SalesDB = 2.2 TB

Six months
SalesDB = 3.2 TB
```

That trend is more valuable than today's number alone.

Think:

```text
Current size
+
Growth rate
+
Business forecast
=
Capacity plan
```

---

# 9. Automated health report

Imagine every morning at 7 AM:

```text
SQL MI DAILY HEALTH REPORT
--------------------------------

Availability:      Healthy
CPU:               42%
Storage:           63%
Blocked sessions:  0
Deadlocks:         0
Failed jobs:       1
Long queries:      2
Critical alerts:   0
```

The DBA receives:

```text
GREEN
```

or:

```text
ACTION REQUIRED
```

This is much better than manually checking everything.

---

# 10. PowerShell

PowerShell can automate Azure administration.

Typical pattern:

```text
PowerShell
   |
   +-- Authenticate
   |
   +-- Query Azure
   |
   +-- Retrieve resource information
   |
   +-- Run operational task
   |
   +-- Export/report
```

For example, an enterprise may automate:

```text
Resource inventory
Configuration checks
Monitoring configuration
Operational reporting
```

---

# 11. Azure CLI

Azure CLI provides another automation approach.

Conceptually:

```text
az
 |
 +-- resource operations
 +-- SQL operations
 +-- monitoring
 +-- networking
 +-- identity
```

You don't need to use PowerShell and CLI simultaneously for every task.

Pick the tool that fits your team's automation standards.

---

# 12. Infrastructure as Code

For repeatable environments:

```text
Development
Test
Production
DR
```

you don't want four completely manual builds.

Instead:

```text
Infrastructure definition
        |
        v
Automation pipeline
        |
        +---- Dev
        +---- Test
        +---- Prod
```

Common infrastructure-as-code approaches include:

```text
ARM/Bicep
Terraform
```

The important concept is:

> **Infrastructure becomes repeatable and version-controlled.**

---

# 13. Example enterprise deployment

Without automation:

```text
Engineer manually creates:

VNet
Subnet
NSG
MI
Monitoring
Alerts
Security
```

Potential problem:

```text
Dev ≠ Test ≠ Prod
```

With automation:

```text
Infrastructure definition
        ↓
Dev
        ↓
Test
        ↓
Production
```

Configuration becomes much more consistent.

---

# 14. Monitoring automation

Suppose:

```text
CPU > 90%
```

Instead of:

```text
CPU > 90%
   ↓
Nobody notices
   ↓
Customer complains
```

you want:

```text
CPU > 90%
      ↓
Azure Monitor
      ↓
Alert
      ↓
Notification
      ↓
DBA
```

For more advanced environments:

```text
Alert
  ↓
Automation
  ↓
Diagnostic data
  ↓
Incident system
```

---

# 15. Don't automate dangerous actions blindly

This is important.

Bad automation:

```text
CPU > 90%
   ↓
Restart database
```

Why?

Because CPU may be high due to:

```text
Critical business workload
```

Restarting could make the situation worse.

Better:

```text
CPU > 90%
    ↓
Alert
    ↓
Collect diagnostic information
    ↓
DBA / automated decision
```

Automation should reduce risk, not create it.

---

# Lesson 26 — Azure SQL MI Security Operations

Now let's go deeper into security.

A production security model should look something like:

```text
                   Identity
                      |
                      v
             Microsoft Entra ID
                      |
                      v
                 SQL MI
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
      Logins        Users         Roles
        |             |             |
        +-------------+-------------+
                      |
                      v
                 Permissions
                      |
                      v
                   Data
```

---

# 16. Authentication vs authorization

Another interview favorite.

### Authentication

> **Who are you?**

Example:

```text
User → Login
```

### Authorization

> **What are you allowed to do?**

Example:

```text
User
 ↓
ReportingReader
 ↓
SELECT
```

Remember:

```text
Authentication ≠ Authorization
```

---

# 17. Application security

Suppose an application needs to execute:

```sql
SELECT OrderID, OrderDate
FROM Sales.Orders;
```

Don't give it:

```text
sysadmin
```

Instead:

```text
Application identity
      ↓
Database user
      ↓
Specific role
      ↓
Required permissions
```

This limits damage if the application credential is compromised.

---

# 18. Auditing scenario

Imagine:

```text
Customer table
```

contains sensitive information.

Someone executes:

```sql
DELETE FROM Customer.Customers;
```

Your organization needs to know:

```text
Who?
When?
What?
```

This is where auditing and centralized monitoring become important.

---

# 19. Security incident

Suppose an account suddenly starts querying:

```text
Millions of customer records
```

at 3 AM.

Your security architecture should make it possible to:

```text
Detect
   ↓
Investigate
   ↓
Identify identity
   ↓
Determine scope
   ↓
Respond
```

This is why database security and monitoring can't be separated.

---

# Lesson 27 — Backup and Recovery Drill

Now let's simulate a real emergency.

At:

```text
10:15 AM
```

a developer accidentally executes:

```sql
DELETE FROM Sales.Orders;
```

without a `WHERE` clause.

Millions of rows disappear.

The application team says:

> **"Recover the data immediately."**

What do you do?

---

# 20. First: Don't panic

Determine:

```text
When did it happen?
What exactly was deleted?
Was the transaction committed?
Which database?
How much data?
```

If the transaction is still active:

```text
ROLLBACK
```

may be relevant.

But if committed:

```text
Recovery strategy
```

is required.

---

# 21. Point-in-time recovery

Suppose:

```text
10:00
Good database

10:15
Accidental DELETE

10:30
Incident discovered
```

You may need to recover to a point shortly before the mistake, depending on the available recovery capabilities and your architecture.

Conceptually:

```text
Backups
   |
   +---- 10:00
   +---- 10:05
   +---- 10:10
   +---- 10:15
   +---- ...
```

The objective is:

```text
Recover
   ↓
Validate
   ↓
Restore business data
```

---

# 22. Never assume backup = recovery

This is a very important DBA principle.

You can have:

```text
Backup = SUCCESS
```

but still have:

```text
Recovery = FAILURE
```

if:

* The backup cannot be restored
* Recovery point is wrong
* Permissions are missing
* Application dependencies aren't available
* Data validation fails

Therefore:

> **Test restores.**

---

# 23. Recovery testing

A mature organization periodically performs:

```text
Backup
  ↓
Restore
  ↓
Validate
  ↓
Measure recovery time
```

Record:

```text
RTO achieved
RPO achieved
Data validation
Issues discovered
```

---

# Lesson 28 — End-to-End Production Scenario

Now let's put everything together.

### Company

```text
ABC E-Commerce
```

### Workload

```text
15,000 transactions/min
24/7
Sensitive customer information
Hybrid environment
```

### Requirements

```text
RPO: minimal
RTO: < 30 min
Strong security
Monitoring
DR
```

---

# 24. Your architecture

You could design:

```text
                         Internet
                            |
                            v
                       Application
                            |
                            v
                     Azure Network
                            |
                            v
                     SQL Managed
                       Instance
                            |
          +-----------------+-----------------+
          |                 |                 |
          v                 v                 v
         HA              Backup           Monitoring
          |                 |                 |
          v                 v                 v
      Failover             PITR         Azure Monitor
                            |
                            v
                            DR Region
```

Around everything:

```text
Identity
Security
Auditing
Networking
Automation
```

---

# 25. Migration approach

Source:

```text
On-prem SQL Server
```

Target:

```text
Azure SQL MI
```

Process:

```text
Assessment
   ↓
Compatibility analysis
   ↓
Dependency mapping
   ↓
Network connectivity
   ↓
Test migration
   ↓
Application testing
   ↓
Performance testing
   ↓
Production migration
   ↓
Cutover
   ↓
Validation
```

---

# 26. After migration

Your work isn't finished.

You now monitor:

```text
Performance
Security
Capacity
Availability
Jobs
Backups
Query regressions
Application behavior
```

Then optimize continuously.

---

# Lesson 29 — Senior DBA Interview Questions

These are the questions I'd expect you to practice.

### Architecture

1. What is Azure SQL Managed Instance?
2. Why would you choose MI instead of SQL Database?
3. MI vs SQL Server on Azure VM?
4. What are the networking requirements?
5. How would you design production and DR?

### Performance

6. How do you troubleshoot high CPU?
7. How do you troubleshoot blocking?
8. What causes deadlocks?
9. What is Query Store?
10. How do you investigate a plan regression?
11. What is a covering index?
12. What causes tempdb spills?

### Security

13. Authentication vs authorization?
14. How would you implement least privilege?
15. Why use managed identity?
16. How would you audit sensitive data access?

### Recovery

17. What is RPO?
18. What is RTO?
19. How do you perform point-in-time recovery?
20. How do you test backups?

### Migration

21. How do you migrate SQL Server to MI?
22. How do you achieve minimal downtime?
23. What dependencies do you assess?
24. How do you plan rollback?
25. How do you validate after cutover?

---

# Lesson 30 — The Final DBA Challenge

This is your **capstone project**.

You're the lead Azure SQL DBA.

A company gives you:

```text
ON-PREMISES

SQL Server 2019

Databases:
    CustomerDB       1 TB
    OrdersDB         5 TB
    ReportingDB      2 TB

SQL Agent jobs:
    120

Logins:
    80

Linked servers:
    5

SSIS:
    15 packages

Application:
    24/7

Traffic:
    15,000 transactions/min

Requirements:
    RPO ≤ 5 minutes
    RTO ≤ 30 minutes
    Minimal downtime
    Strong security
    DR required
    Monitoring required
```

Your job is to produce a design covering:

```text
1. Assessment
2. MI architecture
3. Network
4. Identity
5. Security
6. Migration method
7. HA
8. DR
9. Backup/recovery
10. Performance monitoring
11. Query optimization
12. Automation
13. Cutover
14. Rollback
15. Post-migration operations
```

If you can explain all 15 areas clearly, you're no longer just learning Azure SQL MI terminology—you are thinking like a **production Azure SQL DBA**.

### One important correction to our course status

We have now covered the **remaining conceptual areas**, but I would still consider the training **not fully complete until you've done the hands-on/capstone exercises**. That's where the knowledge becomes interview-ready and production-ready.
