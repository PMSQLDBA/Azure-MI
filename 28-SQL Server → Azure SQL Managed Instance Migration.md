# Lesson 58 — SQL Server → Azure SQL Managed Instance Migration: What Actually Breaks

Now let's do a **real migration project** from start to finish.

Imagine a company has an on-premises SQL Server environment and wants to move it to Azure SQL Managed Instance.

The database migration tool reports:

> **Migration successful.**

But two days later, production is broken.

Why?

Because **database migration and application/workload migration are not the same thing.**

---

# 1. The fictional company

Let's use a realistic example.

**Company:** RetailMart

Current environment:

```text
On-premises
     |
     +-- SQL Server PROD01
           |
           +-- SalesDB
           +-- CustomerDB
           +-- InventoryDB
           +-- ReportingDB
```

Applications:

```text
Web Application
Order Processing
Reporting
ETL
Finance
```

SQL Server has:

```text
250 databases
1200 SQL Agent jobs
40 linked servers
Cross-database queries
SQL logins
Windows/AD authentication
SSIS packages
Reporting dependencies
```

Management says:

> "Move everything to Azure SQL Managed Instance."

---

# 2. Step 1 — Don't start migrating yet

Your first job is:

> **Discovery.**

Build an inventory.

```text
SQL Server
    |
    +-- Databases
    +-- Logins
    +-- Jobs
    +-- Linked servers
    +-- Credentials
    +-- Certificates
    +-- SSIS
    +-- Applications
    +-- Reports
    +-- Interfaces
```

For every database, determine:

```text
Database size
Compatibility level
SQL Server version
Recovery model
HA dependencies
Application owner
Criticality
Peak workload
Business hours
```

---

# 3. Dependency mapping

This is where senior DBAs differentiate themselves.

Suppose:

```text
SalesDB
   |
   +---- Application A
   |
   +---- SQL Agent Job 17
   |
   +---- ReportingDB
   |
   +---- LinkedServer FinanceSQL
```

If you migrate only `SalesDB`, you haven't migrated the workload.

You need:

```text
SalesDB
 + application
 + jobs
 + dependencies
 + security
 + integrations
```

---

# 4. Compatibility assessment

Now ask:

> **Can this workload run on MI as-is?**

Look for:

```text
Unsupported features
Deprecated features
SQL Server version differences
Instance-level dependencies
External dependencies
Application assumptions
```

This is where Microsoft's migration assessment tooling can help identify compatibility issues.

But don't treat a tool report as the entire migration plan.

A tool can identify technical dependencies.

It doesn't know all your business dependencies.

---

# 5. Compatibility example

Suppose the assessment reports:

```text
Database:
SalesDB

Compatibility:
Mostly compatible

Issue:
Linked Server dependency
```

The junior DBA might say:

> "Only one issue."

The senior DBA asks:

> "What does the linked server do?"

Investigation reveals:

```text
SalesDB
   |
   v
Linked Server
   |
   v
Finance SQL Server
   |
   v
Tax calculation
```

That's not a minor technical issue.

It's a **business dependency**.

---

# 6. Linked servers

Imagine:

```sql id="3qkqj5"
SELECT
    s.OrderID,
    f.TaxAmount
FROM SalesDB.dbo.Orders s
JOIN FinanceServer.FinanceDB.dbo.Tax f
    ON s.OrderID = f.OrderID;
```

This query crosses server boundaries.

During migration, you need to determine:

```text
Does the linked server remain?
Where is the target?
Is authentication supported?
Is network connectivity available?
Is the query performance acceptable?
```

Don't assume:

> "It worked on-premises, therefore it will work in Azure."

---

# 7. Cross-database queries

Now imagine:

```sql id="c0v3x6"
SELECT *
FROM SalesDB.dbo.Orders o
JOIN CustomerDB.dbo.Customers c
    ON o.CustomerID = c.CustomerID;
```

This is an important dependency.

You need to know:

```text
Are both databases moving to the same MI?
Are they moving together?
Will the application still use the same names?
```

If both databases land on the same MI, the architecture can remain much closer to the existing SQL Server model.

---

# 8. SQL Agent migration

Suppose this job exists:

```text id="q6n2l3"
Job: NightlySalesLoad

Schedule:
Every day at 1:00 AM

Steps:
1. Execute ETL
2. Run stored procedure
3. Export report
4. Send notification
```

Migration requires more than copying the database.

You must verify:

```text
Job exists
Schedule exists
Owner exists
Credentials exist
Job steps work
Paths exist
External systems reachable
Notifications work
```

---

# 9. The classic migration failure

Database migration:

```text id="6t5l1b"
SUCCESS
```

Application testing:

```text id="0f8x2e"
SUCCESS
```

Go-live:

```text id="8h2y7s"
SUCCESS
```

Next morning:

```text id="j6l4a0"
Reports missing
```

Why?

Because:

```text id="y0j7z1"
SQL Agent Job
       ↓
Export file
       ↓
Old on-prem file share
```

The database moved.

The file share didn't.

---

# 10. Security migration

This is another huge issue.

Suppose on-premises has:

```text id="r7y9m2"
DOMAIN\SalesUsers
DOMAIN\FinanceUsers
DOMAIN\DBAUsers
```

You migrate the databases.

But permissions depend on identities.

You need to validate:

```text
Logins
Users
Roles
Permissions
Authentication method
Service identities
Application identities
```

Remember:

```text id="x2e7k9"
Login
   ↓
Database User
   ↓
Role
   ↓
Permissions
```

A database user without the correct identity mapping can create application failures.

---

# 11. Connection strings

After migration, applications need to point to the MI endpoint.

Suppose old:

```text id="2r4x7m"
Server=PROD01
```

New:

```text id="5h3c8q"
Server=<MI endpoint>
```

You need to update:

```text id="9q1p4f"
Connection strings
Secrets
DNS
Application configuration
Deployment pipelines
Monitoring
```

Don't hard-code the new endpoint in dozens of applications if your architecture can provide a better abstraction.

---

# 12. Cutover strategy

Don't think:

```text id="y6n8s4"
Migration = one switch
```

Think:

```text id="7j3q5e"
Preparation
     ↓
Initial migration
     ↓
Validation
     ↓
Data synchronization
     ↓
Testing
     ↓
Cutover
     ↓
Validation
     ↓
Monitoring
     ↓
Decommission
```

---

# 13. Why migration testing matters

Suppose your migration takes:

```text id="9j4h6k"
10 hours
```

but the business allows only:

```text id="u3k8x1"
30 minutes downtime
```

You cannot simply stop production and copy everything.

You need a migration strategy that separates:

```text
Bulk data movement
```

from:

```text
Final synchronization/cutover
```

This is where migration tooling and appropriate online migration approaches become important.

---

# 14. Pre-cutover checklist

Before switching production:

```text id="5d0k3s"
☐ Database migrated
☐ Data validated
☐ Row counts checked
☐ Application tested
☐ Queries tested
☐ SQL Agent jobs tested
☐ Linked servers tested
☐ Security tested
☐ DNS tested
☐ Network tested
☐ Monitoring configured
☐ Backup/recovery validated
☐ Performance baseline captured
☐ Rollback plan documented
```

---

# 15. Rollback plan

This is one of the most important interview topics.

Interviewer:

> "What's your rollback plan?"

Weak answer:

> "We'll restore the old server."

Better answer:

> "Before cutover, I define explicit rollback criteria, preserve the source environment, validate synchronization state, document the exact application and DNS changes, and define how we would redirect the application back to the source if post-cutover validation fails."

Rollback isn't:

```text id="o5z0fa"
"Something goes wrong → panic."
```

It's a **predefined decision process**.

---

# 16. Post-cutover monitoring

The first few hours are critical.

Monitor:

```text id="j3m7vd"
CPU
IO
Query duration
Blocking
Deadlocks
Connections
Errors
Application latency
Job execution
Database growth
```

Compare against the baseline you captured before migration.

---

# 17. Migration success criteria

Don't define success as:

> "The database is online."

Define it as:

```text id="e1n4y6"
Database online
      +
Applications working
      +
Business transactions working
      +
Jobs working
      +
Reports working
      +
Security working
      +
Performance acceptable
      +
Recovery validated
```

That's a **production migration**.

---

# Lesson 59 — Migration Interview Scenario

### Interviewer:

> "You migrated a 2 TB SQL Server database to Managed Instance. The migration tool says successful, but users complain the application is slower. What do you do?"

Don't immediately say:

> "Increase MI compute."

Use this sequence:

```text id="1v8q3c"
1. Establish baseline
        ↓
2. Identify affected queries
        ↓
3. Compare execution plans
        ↓
4. Check Query Store
        ↓
5. Check CPU/IO/waits
        ↓
6. Check blocking
        ↓
7. Check statistics/indexes
        ↓
8. Check application/network latency
        ↓
9. Identify root cause
        ↓
10. Apply targeted fix
```

Possible causes include:

```text id="l2x7q9"
Different execution plan
Statistics
Indexing
Compatibility behavior
Parameter sensitivity
Application connection behavior
Network latency
Resource sizing
Concurrency
```

Only after evidence should you consider scaling.

---

# Lesson 60 — Your Migration Architecture

A mature SQL MI migration project looks like:

```text id="p7v4x2"
                 ON-PREMISES
                     |
             +-------+-------+
             |               |
        SQL Server       Applications
             |               |
             +-------+-------+
                     |
                 Discovery
                     |
                     v
              Compatibility
                 Assessment
                     |
                     v
               Remediation
                     |
                     v
              Initial Migration
                     |
                     v
               Data Sync
                     |
                     v
                 Testing
                     |
                     v
                 Cutover
                     |
                     v
              Azure SQL MI
                     |
          +----------+----------+
          |          |          |
       Monitor     Backup      DR
```

The most important takeaway:

> **Migration is not a database-copy exercise. It's a workload transformation project.**

---

* TempDB problems
* and a **real 2 AM production performance incident where you have to diagnose the problem step-by-step**.
