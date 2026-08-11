Previous: 

Exactly. For the incident, the key answer is: 

**the secondary replica doesn't save you from a committed bad transaction because the change is replicated too.** 

You would investigate **point-in-time recovery**, restore to a point before 2:15 PM, validate the affected rows, and then repair production rather than blindly replacing the whole database.

Now let's get into the part that ties everything together.

# Lesson 6 — SQL Server 2019 → Azure SQL Managed Instance Migration

We'll use the same production scenario:

```text
ON-PREMISES
SQL Server 2019
       |
       | 4 TB
       |
       v
Azure SQL Managed Instance
```

Business requirements:

```text
Database size:       4 TB
SQL Server:          2019
Downtime:            < 10 minutes
Application:         24/7
DR:                  Required
SQL Agent:           Required
Cross-database:      Required
```

This is a realistic MI migration.

---

# 1. Don't start by copying the database

A common junior DBA approach is:

```text
SQL Server
    |
    ↓
Backup
    |
    ↓
Azure MI
```

That is only part of the migration.

A production migration should look more like:

```text
Assessment
    ↓
Compatibility analysis
    ↓
Architecture
    ↓
Networking
    ↓
Security
    ↓
Migration method
    ↓
Test migration
    ↓
Performance testing
    ↓
Cutover planning
    ↓
Production migration
    ↓
Validation
    ↓
Monitoring
```

This is the mindset of a migration DBA.

---

# 2. Phase 1 — Assessment

Before migration, collect information.

### SQL Server inventory

```text
SQL Server version
Edition
Build
Databases
Database sizes
Compatibility levels
Recovery models
```

### Instance-level objects

```text
Logins
SQL Agent jobs
Credentials
Linked servers
Endpoints
Server-level permissions
Certificates
Database mail
```

### Database-level objects

```text
Tables
Indexes
Stored procedures
Functions
Views
Triggers
Users
Roles
Permissions
```

---

# 3. Workload assessment

You also need to understand how the server behaves.

Collect:

```text
CPU
Memory
IOPS
IO latency
Throughput
Connections
Transactions/sec
Wait statistics
Blocking
Deadlocks
Top queries
Query duration
Database growth
```

Why?

Because you're not simply migrating:

> **a database**

You're migrating:

> **a workload.**

---

# 4. Compatibility assessment

Now ask:

> **"Will everything work on MI?"**

SQL Managed Instance has high SQL Server compatibility, but **high compatibility does not mean every feature/configuration is automatically identical**.

You should identify:

```text
Unsupported features
Changed behavior
Version differences
Configuration dependencies
Application dependencies
```

Microsoft provides migration/assessment tooling for evaluating SQL Server workloads before moving them to Azure SQL targets.

---

# 5. Example compatibility issue

Suppose your SQL Server has:

```text
Linked Server
```

and the application does:

```sql id="0n8p4v"
SELECT *
FROM [LinkedServer01].OtherDB.dbo.Customers;
```

You shouldn't assume:

> "It worked on SQL Server, so it will automatically work the same way on MI."

You need to assess:

```text
Linked server configuration
Authentication
Network connectivity
Provider compatibility
Distributed queries
```

This is why migration assessment matters.

---

# 6. Another dependency: SQL Agent

ABC Retail has:

```text
SQL Agent Jobs
```

For example:

```text id="jz7k0p"
Job: Nightly ETL
Schedule: 1:00 AM

Job:
  EXEC dbo.LoadSalesData
```

If SQL Agent is an important workload component, this is one reason MI can be attractive compared with Azure SQL Database: MI supports SQL Agent capabilities that are familiar to SQL Server DBAs, subject to MI-specific limitations.

But don't simply say:

> "SQL Agent exists, therefore migration is automatic."

You need to review:

```text
Job owner
Credentials
Proxies
PowerShell
CmdExec
Schedules
Linked servers
File-system dependencies
Job steps
```

A job that executes:

```text
C:\Scripts\load.ps1
```

on your on-premises server may need architectural changes.

---

# 7. Migration methods

Now we reach the important question:

> **How do we actually move the data?**

There isn't one universal method.

The correct method depends on:

```text
Database size
Downtime requirement
Network bandwidth
Source configuration
Target configuration
Migration tooling
Business requirements
```

---

# 8. Backup and restore

For a suitable workload, you can use:

```text
SQL Server
   |
   | Full backup
   v
Backup storage
   |
   v
Azure SQL MI
```

Conceptually:

```text id="1wyj2s"
SQL Server
    |
    | BACKUP
    v
Azure Storage
    |
    | RESTORE
    v
SQL Managed Instance
```

This is straightforward and familiar to DBAs.

But there's a problem with our requirement:

```text
Database = 4 TB
Downtime = <10 minutes
```

A full backup/restore process could take too long depending on:

```text
Backup throughput
Network bandwidth
Storage
Restore throughput
Database size
```

Therefore, don't automatically choose backup/restore just because it's familiar.

---

# 9. Online migration

For a 24/7 production system, you may investigate an **online migration approach**.

Conceptually:

```text
              SQL Server
                  |
                  | Continuous data movement
                  |
                  v
              Azure MI
                  |
                  |
             Keep syncing
                  |
                  v
              Cutover
```

During most of the migration:

```text
Production application
        |
        v
   SQL Server
```

continues running.

Meanwhile:

```text
SQL Server
    |
    +-----------------> MI
```

keeps the target synchronized.

Then at cutover:

```text
Application
     |
     X
SQL Server
     
Application
     |
     v
SQL MI
```

The objective is to make the final synchronization/cutover window small.

---

# 10. Why online migration is attractive

Our business says:

```text
24/7 application
<10 minutes downtime
```

So:

```text
Offline migration
     ↓
Long outage
     ↓
Potentially unacceptable
```

Whereas:

```text
Online migration
     ↓
Initial copy
     ↓
Continuous synchronization
     ↓
Small final cutover
```

may better fit the requirement.

The exact tooling and supported migration path should be selected after assessing the source workload and current Azure capabilities.

---

# 11. Cutover

This is the most stressful part of a migration.

Imagine:

```text
Saturday 11:00 PM

Migration team ready
Application team ready
Business owner ready
Rollback plan ready
```

You enter the cutover window.

### Step 1

Stop application writes.

```text
Application
     |
     X
No more writes
```

### Step 2

Allow final synchronization.

```text
SQL Server
     |
     ↓
MI
```

### Step 3

Verify synchronization.

```text
Source position
      =
Target position
```

### Step 4

Switch application connection.

```text
OLD

Application
    ↓
SQL Server
```

to:

```text
NEW

Application
    ↓
SQL MI
```

### Step 5

Run smoke tests.

---

# 12. Smoke testing

Don't just ask:

> "Can I connect?"

Test actual business functionality.

For example:

```text
Login
   ↓
Customer search
   ↓
Create order
   ↓
Update order
   ↓
Payment transaction
   ↓
Reporting
```

Database validation:

```text
SELECT COUNT(*)
FROM Customers;
```

Compare critical counts between source and target.

Check:

```text
Tables
Rows
Stored procedures
Indexes
Users
Permissions
Jobs
```

---

# 13. Application connection string

This is an easy place to make a mistake.

Old:

```text
Server=ONPREM-SQL01;
Database=SalesDB;
```

New:

```text
Server=<MI-endpoint>;
Database=SalesDB;
```

But production applications may have:

```text
Connection pooling
TLS settings
Authentication method
Timeouts
Retry logic
Failover settings
Secrets
```

So the connection string change needs testing.

---

# 14. The rollback plan

A professional migration always asks:

> **"What if the cutover fails?"**

Suppose:

```text
11:00 PM
Cutover

11:05 PM
Application errors

11:07 PM
Problem identified
```

You need a decision tree:

```text
Migration
   |
   v
Validation
   |
   +---- PASS → Continue
   |
   +---- FAIL
           |
           v
       Is issue fixable
       within window?
         /       \
       YES        NO
       |           |
     Fix        Rollback
```

Never enter production cutover without knowing:

```text
Who makes rollback decision?
What triggers rollback?
How do we redirect application?
How do we handle writes?
How do we protect data consistency?
```

---

# 15. Critical rollback problem

Suppose the application starts writing to MI:

```text
11:10 PM
Application → MI
```

Then you discover a serious problem and say:

> "Let's switch back to SQL Server."

Wait.

What happened to the writes made after 11:00 PM?

```text
SQL MI
   |
   +-- New orders
   +-- New customers
   +-- Updates
```

If you simply switch back:

```text
Application → SQL Server
```

you could lose those transactions.

Therefore:

> **Rollback is not just changing a connection string.**

You need a carefully designed data-consistency strategy.

This is one of the most important real-world migration lessons.

---

# 16. Migration validation checklist

After cutover:

```text
☐ DNS resolution
☐ Application connectivity
☐ Authentication
☐ Database accessibility
☐ Row counts
☐ Critical tables
☐ Stored procedures
☐ Jobs
☐ Linked servers
☐ Permissions
☐ Application transactions
☐ Performance
☐ Monitoring
☐ Backup status
☐ DR status
```

---

# 17. Performance validation

Don't assume:

> "Migration successful = performance successful."

You might discover:

```text
Before:
Query = 500 ms

After:
Query = 3 seconds
```

Then investigate:

```text
Execution plan
Statistics
Indexes
Compatibility level
CPU
IO
Memory
Waits
Network latency
Parameter sensitivity
```

A migration is successful only when the **business workload** works correctly and acceptably.

---

# 18. Complete migration architecture

Our ABC Retail project now looks like:

```text id="5c7qpp"
                     ON-PREMISES
                SQL Server 2019
                      |
                      |
              Assessment / Testing
                      |
                      |
             Migration connectivity
                      |
                      v
              +---------------+
              | Azure VNet    |
              |               |
              | Application   |
              |      |        |
              |      v        |
              | SQL MI        |
              +---------------+
                      |
             +--------+--------+
             |                 |
             v                 v
        Automated             DR
         Backups
```

Migration flow:

```text
Assessment
    ↓
Compatibility
    ↓
Network
    ↓
Security
    ↓
Initial migration
    ↓
Synchronization
    ↓
Cutover
    ↓
Validation
    ↓
Monitoring
    ↓
DR validation
```

---

# 19. The DBA interview answer

If an interviewer asks:

> **"How would you migrate a 4-TB SQL Server 2019 production database to Azure SQL Managed Instance with less than 10 minutes downtime?"**

A strong answer would sound like:

> "First I would assess the workload, compatibility, dependencies, database size, performance characteristics, logins, SQL Agent jobs, linked servers, and application connectivity. Then I'd design the MI networking, security and target sizing. Because the workload is 4 TB and the downtime requirement is under 10 minutes, I'd investigate an online migration approach rather than relying solely on an offline backup/restore. I'd perform a test migration, validate application functionality and performance, establish a detailed cutover and rollback plan, synchronize the target, stop writes during the final cutover, validate synchronization, redirect the application, perform smoke tests, and monitor the environment. Finally, I'd validate backup and DR recovery."

That's the type of answer that demonstrates **real DBA thinking**, rather than just knowing Azure terminology.

---

# Your Lesson 6 challenge

You are planning the ABC Retail migration.

You have:

```text
SQL Server 2019
Database: 4 TB
24/7 application
Downtime: <10 minutes
SQL Agent jobs
Linked servers
Heavy reporting
DR required
```

Answer these 5 questions:

**1. Why might a simple full backup → restore approach be risky for the <10-minute downtime requirement?**

**2. What would you assess before choosing the migration method?**

**3. Why is a test migration necessary even if the database restores successfully?**

**4. During cutover, why do you need to stop application writes before final synchronization?**

**5. Why is rollback more complicated after the application has started writing to MI?**

