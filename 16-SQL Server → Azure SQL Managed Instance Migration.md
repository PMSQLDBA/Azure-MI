**Lesson 14 — Migrating SQL Server to Azure SQL Managed Instance**.

This is one of the most valuable real-world skills for an Azure SQL DBA because companies commonly have existing SQL Server workloads they want to move to Azure without redesigning the entire application.

# Lesson 14 — SQL Server → Azure SQL Managed Instance Migration

Imagine this environment:

```text
ON-PREMISES
┌─────────────────────────────┐
│ Windows Server              │
│                             │
│ SQL Server 2019             │
│                             │
│ SalesDB                     │
│ CustomerDB                  │
│ OrderDB                     │
└──────────────┬──────────────┘
               │
               │ Migration
               ▼
          Azure VNet
               │
               ▼
       SQL Managed Instance
```

The goal isn't simply:

> "Move the database."

The real goal is:

> **Move the workload while preserving functionality, performance, security, and business continuity.**

---

# 1. Why choose Managed Instance?

Suppose the company has a traditional SQL Server environment using:

```text
SQL Server
├── SQL Agent jobs
├── Logins
├── Databases
├── Stored procedures
├── Linked servers
├── SQL Server features
├── SSIS workloads
└── Maintenance processes
```

They don't want to rewrite everything for a cloud-native database platform.

MI can be attractive because it provides a managed SQL Server-compatible environment while reducing much of the infrastructure management burden.

Think:

```text
Traditional SQL Server
        ↓
    Migration
        ↓
Azure SQL Managed Instance
        ↓
Less infrastructure management
```

---

# 2. Migration isn't one step

A professional migration has stages:

```text
Assessment
    ↓
Compatibility analysis
    ↓
Dependency discovery
    ↓
Target design
    ↓
Network setup
    ↓
Migration method
    ↓
Testing
    ↓
Performance validation
    ↓
Cutover
    ↓
Validation
    ↓
Decommission
```

Never start with:

> "Let's restore the database to Azure."

First understand the workload.

---

# 3. Step 1 — Assessment

Suppose you have:

```text
SQL Server 2019
CPU: 70%
Memory: 80%
Storage: 8 TB
Databases: 25
```

You need to understand:

```text
Database size
Growth
CPU
Memory
IO
Connections
Query performance
SQL Agent jobs
Logins
Linked servers
SSIS
Replication
CLR
Service Broker
Features
Dependencies
```

---

# 4. Compatibility assessment

You need to ask:

> "Will this workload behave correctly on Managed Instance?"

You investigate:

```text
T-SQL compatibility
SQL Server features
Instance-level features
Database-level features
Security model
External dependencies
Application dependencies
```

Some features may require changes or alternative implementations.

This is why assessment comes **before migration**.

---

# 5. Example assessment

Suppose:

```text
Legacy SQL Server
```

contains:

```text
25 databases
120 SQL Agent jobs
50 logins
8 linked servers
3 SSIS packages
2 replication configurations
```

You can't just say:

> "All databases are compatible."

You need to map the **entire workload**.

For example:

```text
Database
    |
    +-- Stored procedures
    +-- Users
    +-- Jobs
    +-- Linked servers
    +-- External applications
    +-- Reporting
```

---

# 6. Database compatibility vs application compatibility

A database may restore successfully.

But the application can still fail.

Example:

```text
Database
   ↓
Successfully migrated
```

but:

```text
Application
   ↓
Hard-coded server name
   ↓
Connection failure
```

Or:

```text
Application
   ↓
Old authentication method
   ↓
Failure
```

So:

> **Database migration success ≠ application migration success.**

---

# 7. Step 2 — Target architecture

Before migrating, design the Azure environment.

Example:

```text
                 Azure
                   |
                 VNet
                   |
        +----------+----------+
        |                     |
        v                     v
 Application subnet       MI subnet
                              |
                              v
                       SQL Managed Instance
```

Then consider:

```text
DNS
NSGs
Routing
Firewall
VPN / ExpressRoute
Identity
Monitoring
Backup
DR
```

This connects everything we've already learned.

---

# 8. Hybrid connectivity

During migration, the source may still be on-premises.

So:

```text
ON-PREMISES
SQL Server
    |
    |
VPN / ExpressRoute
    |
    |
AZURE
    |
    v
SQL MI
```

This network path must be reliable.

Otherwise your migration process may fail or become extremely slow.

---

# 9. Migration strategy

There are several broad strategies.

### Offline migration

```text
Stop application
      ↓
Migrate database
      ↓
Validate
      ↓
Start application
```

Simple conceptually.

But downtime may be significant.

---

# 10. Online migration

For systems requiring minimal downtime:

```text
Production SQL Server
        |
        | Continuous migration
        v
SQL Managed Instance
        |
        | Keep synchronized
        v
Cutover
```

Conceptually:

```text
Source
  |
  |---- data ---->
  |
  |---- ongoing changes ---->
  |
  v
Target MI
```

At cutover:

```text
Stop writes
     ↓
Final synchronization
     ↓
Validate
     ↓
Switch application
```

This can significantly reduce downtime compared with a simple offline migration.

---

# 11. Real production scenario

ABC Retail has:

```text
SalesDB = 4 TB
```

Business says:

> "We can only have 15 minutes of downtime."

A simple:

```text
Backup
 ↓
Copy
 ↓
Restore
```

might not meet the requirement.

You would investigate an approach that supports:

```text
Initial data movement
+
Ongoing synchronization
+
Short final cutover
```

---

# 12. Cutover planning

This is where experienced DBAs spend a lot of time.

You need a runbook.

Example:

```text
23:00  Freeze deployments
23:05  Notify business
23:10  Stop application writes
23:12  Final synchronization
23:15  Validate target
23:18  Update application connection
23:20  Start application
23:25  Smoke test
23:30  Business validation
```

The exact times depend on the environment.

---

# 13. Connection string change

Before migration:

```text
Server=OLD-SQL-SERVER
Database=SalesDB
```

After migration:

```text
Server=SQL-MI-ENDPOINT
Database=SalesDB
```

This looks simple.

But production applications may have:

```text
Connection pools
Secrets
DNS aliases
Configuration files
Environment variables
Load balancers
Deployment pipelines
```

You need to account for all of them.

---

# 14. DNS strategy

One useful approach is to avoid hard-coding infrastructure-specific names throughout the application.

For example:

```text
Application
    |
    v
db.company.internal
    |
    v
SQL Server
```

After migration:

```text
Application
    |
    v
db.company.internal
    |
    v
SQL MI
```

The application doesn't necessarily need to know the physical backend changed.

This can make cutover cleaner, depending on the architecture.

---

# 15. Login migration

Here's a classic migration problem.

You migrate:

```text
Database
```

but forget:

```text
Logins
```

Then:

```text
Application
    |
    v
SQL MI
    |
    X
Login failed
```

Why?

Database users and instance-level logins are related but distinct security objects.

You need to assess:

```text
Logins
Users
Roles
Permissions
Authentication method
Service accounts
```

---

# 16. SQL Agent jobs

Another classic mistake.

Database migrated:

```text
SalesDB → MI
```

But the old server had:

```text
SQL Agent
├── Nightly ETL
├── Index maintenance
├── Data cleanup
├── Reporting refresh
└── Backup-related jobs
```

You need to determine how those jobs will operate in the target environment.

Don't assume:

> "The database is there, so the whole application is migrated."

---

# 17. Linked servers

Suppose:

```text
SQL Server A
      |
      v
Linked Server
      |
      v
SQL Server B
```

After migration:

```text
SQL MI
      |
      X
Linked server dependency
```

You need to evaluate:

```text
Connectivity
Authentication
Provider compatibility
DNS
Network routing
Security
Performance
```

---

# 18. SSIS dependency

Maybe your workload uses:

```text
SQL Server
   |
   v
SSIS
   |
   +---- Files
   +---- APIs
   +---- Databases
   +---- Data warehouse
```

Moving the SQL database doesn't automatically move the entire ETL architecture.

You must map those dependencies.

---

# 19. Performance baseline

Before migration, capture:

```text
CPU
IO
Memory
Query duration
Transactions/sec
Wait statistics
Top queries
Connection count
```

Why?

Because after migration, someone will ask:

> "Is MI faster or slower?"

Without a baseline:

```text
Before = ?
After = ?
```

You can't answer objectively.

---

# 20. Example baseline

Before:

```text
Average query duration = 450 ms
CPU = 65%
IO latency = 8 ms
Orders/min = 12,000
```

After:

```text
Average query duration = 280 ms
CPU = 45%
IO latency = 5 ms
Orders/min = 14,000
```

Now you can demonstrate improvement.

---

# 21. Post-migration validation

After cutover, don't simply check:

```text
SQL MI = Healthy
```

Test the actual business workflow.

For example:

```text
Login
 ↓
Create customer
 ↓
Create order
 ↓
Payment processing
 ↓
Order lookup
 ↓
Reporting
```

Database validation:

```text
Row counts
Checksums where appropriate
Important business totals
Stored procedures
Jobs
Permissions
Application connectivity
Performance
```

---

# 22. Smoke testing

Immediately after cutover:

```text
Application connects?
        ↓
YES
        ↓
Can login?
        ↓
YES
        ↓
Can create order?
        ↓
YES
        ↓
Can retrieve order?
        ↓
YES
        ↓
Reporting works?
        ↓
YES
```

This is your initial smoke test.

---

# 23. Rollback plan

Every production migration needs:

> **"What if this fails?"**

Example:

```text
New MI
   |
   X
Critical problem
   |
   v
Rollback
   |
   v
Old SQL Server
```

But rollback isn't trivial.

Suppose the application processed:

```text
1,000 new orders
```

on MI after cutover.

If you simply switch back to the old server:

```text
Those 1,000 orders
     ↓
Potentially missing
```

Therefore:

> **Rollback planning must account for data written after cutover.**

---

# 24. This is why cutover is a business process

Migration isn't just:

```text
DBA → database
```

It involves:

```text
DBA
Application team
Network team
Security team
Infrastructure team
Business owners
Testing team
```

Everyone needs to understand:

```text
Start time
Freeze window
Validation
Success criteria
Rollback criteria
```

---

# 25. Real migration failure

Imagine migration succeeds.

At 10:00 AM:

```text
Application → MI
```

At 10:05 AM:

```text
Application reports errors
```

You discover:

```text
SQL login works
Database exists
But:
Linked server dependency fails
```

The database migration itself succeeded.

The **application dependency migration** failed.

This is why dependency assessment matters.

---

# 26. Migration runbook

A professional runbook might look like:

```text
PRE-MIGRATION

☐ Inventory SQL Server
☐ Assess compatibility
☐ Identify dependencies
☐ Baseline performance
☐ Design MI
☐ Configure network
☐ Configure security
☐ Test connectivity
☐ Test migration
☐ Validate application

CUTOVER

☐ Freeze changes
☐ Stop writes
☐ Final synchronization
☐ Validate data
☐ Change application endpoint
☐ Start application
☐ Smoke test
☐ Business validation

POST-MIGRATION

☐ Monitor CPU
☐ Monitor IO
☐ Monitor waits
☐ Check errors
☐ Validate jobs
☐ Validate security
☐ Compare performance
☐ Keep rollback window
☐ Document results
```

---

# 27. Migration mindset

Don't think:

> **"How do I move this database?"**

Think:

> **"How do I move this business workload safely?"**

That's the difference between a junior DBA and a senior migration engineer.

---

# 28. Interview question

### "How would you migrate a production SQL Server database to Azure SQL Managed Instance with minimal downtime?"

Strong answer:

> "I'd begin with workload and compatibility assessment, including databases, SQL Agent jobs, logins, linked servers, SSIS and application dependencies. I'd design the MI networking and security architecture and establish reliable connectivity between the source and target. I'd choose an online or low-downtime migration approach where appropriate, perform a test migration, validate data and performance, and create a detailed cutover and rollback runbook. During cutover I'd stop writes, complete final synchronization, validate the target, redirect the application, perform smoke and business validation, and closely monitor the environment afterward."

That's the kind of answer interviewers look for.

---

# Your migration challenge 🧑‍💻

You're migrating this production environment:

```text
SQL Server 2019
|
+-- CustomerDB       2 TB
+-- OrdersDB         4 TB
+-- ReportingDB      1 TB
|
+-- 75 SQL Agent jobs
+-- 40 logins
+-- 3 linked servers
+-- SSIS packages
+-- 24/7 application
```

Business requirements:

```text
Maximum downtime = 15 minutes
Data loss = unacceptable
```

Answer these:

**1. What would you assess before migration?**

**2. Would you choose a simple offline backup/restore approach? Why or why not?**

**3. What dependencies besides the databases must you migrate or redesign?**

**4. What would you measure before and after migration?**

**5. What would your rollback plan look like?**

**6. What would you validate immediately after cutover?**
