# Lesson 78 — Migrating SQL Server Agent Jobs to Azure SQL Managed Instance

This is one of the most useful topics if you're preparing for a **SQL Server → Azure SQL Managed Instance migration**.

The key lesson is:

> **You are not migrating just databases. You are migrating the entire operational ecosystem around those databases.**

That includes jobs, scripts, credentials, integrations, monitoring, maintenance, and dependencies.

---

# 1. The migration picture

A traditional SQL Server environment might look like:

```text
                    ON-PREMISES
                        |
                SQL Server
                        |
       +----------------+----------------+
       |                |                |
    Databases       SQL Agent        File System
       |                |                |
       |          +-----+------+         |
       |          |     |      |         |
       |        Jobs  Jobs   Jobs        |
       |                               Scripts
       |                               Backups
       |                               CSV
       |                               Logs
       |
       +---- Applications
```

After migration:

```text
                     AZURE
                       |
              SQL Managed Instance
                       |
       +---------------+---------------+
       |               |               |
   Databases       SQL Agent       Azure Services
       |               |               |
       |            Jobs             Storage
       |            Steps            Key Vault
       |            Schedules        Automation
       |                             Monitoring
       |
   Applications
```

The challenge is deciding what belongs where.

---

# 2. First principle: inventory everything

Before migrating jobs, create an inventory.

For every SQL Agent job, capture:

```text
Job name
Owner
Enabled/disabled
Schedule
Frequency
Steps
Commands
Database
Credentials
Proxies
External dependencies
File paths
Network paths
PowerShell
SSIS
Notifications
Runtime
Failure behavior
Business criticality
```

---

# 3. Example inventory

Suppose you discover these jobs:

| Job               | Purpose           | Frequency | Dependency       |
| ----------------- | ----------------- | --------: | ---------------- |
| IndexMaintenance  | Index maintenance |     Daily | SQL              |
| UpdateStats       | Statistics        |     Daily | SQL              |
| ETL_Orders        | Load warehouse    |    Hourly | SQL + files      |
| BackupJob         | Backup            |     Daily | SQL + storage    |
| ExportCSV         | Export data       |     Daily | Local filesystem |
| PowerShellCleanup | Cleanup           |     Daily | PowerShell       |
| PaymentProcess    | Process payments  |    15 min | SQL + API        |
| ReportRefresh     | Refresh reports   |     Daily | SQL + reporting  |

Now classify them.

---

# 4. Job classification

A useful migration classification is:

```text
                JOB
                 |
       +---------+---------+
       |         |         |
      KEEP     MODIFY    REDESIGN
       |         |         |
   SQL-native  Azure     Cloud-native
```

Let's go through each.

---

# 5. Category 1 — SQL-native jobs

Example:

```sql
EXEC dbo.UpdateStatistics;
```

This is primarily SQL work.

These jobs are generally good candidates for continuing to use SQL Server Agent on SQL Managed Instance, assuming the required functionality is supported.

Conceptually:

```text
SQL Agent
   ↓
T-SQL
   ↓
Database
```

Simple.

---

# 6. Category 2 — Jobs requiring modification

Example:

```text
SQL Agent
   ↓
PowerShell
   ↓
C:\Exports
```

The SQL portion may be fine.

But the infrastructure assumptions need to be examined.

Instead of:

```text
C:\Exports
```

you may need:

```text
Azure Blob Storage
```

or another supported storage/integration architecture.

---

# 7. Category 3 — Jobs that should be redesigned

Example:

```text
SQL Agent
   ↓
Call external REST API
   ↓
Download files
   ↓
Transform
   ↓
Upload files
   ↓
Send notification
```

That's no longer simply a SQL job.

You might use a cloud orchestration service such as:

```text
Azure Data Factory
Azure Functions
Logic Apps
Azure Automation
```

depending on the workload.

---

# 8. Real migration scenario #1 — Index maintenance

On-prem job:

```sql
EXEC dbo.IndexMaintenance;
```

Migration question:

> "Can we migrate this job?"

Probably yes, assuming the procedure and required functionality are supported.

But don't blindly migrate the schedule.

Why?

Azure changes your infrastructure and performance characteristics.

You should reassess:

```text
CPU
I/O
Storage
Workload
Maintenance duration
Business hours
Database size
```

---

# 9. Migration scenario #2 — Local backup

Old job:

```text
BACKUP DATABASE Sales
TO DISK = 'D:\SQLBackups\Sales.bak';
```

This is a classic migration issue.

The on-prem server owns:

```text
D:\SQLBackups
```

A managed service isn't a traditional Windows server where you manage arbitrary local disks.

Therefore, you need to rethink the backup architecture using the supported Azure SQL Managed Instance backup capabilities and appropriate storage/recovery options.

The important lesson:

> **Don't migrate infrastructure assumptions; migrate business requirements.**

---

# 10. Migration scenario #3 — CSV export

Old job:

```text
SQL Agent
   ↓
bcp
   ↓
C:\Exports\Orders.csv
```

After migration:

```text
C:\Exports
```

may no longer be the right architecture.

Instead:

```text
SQL MI
   ↓
Supported data movement/export mechanism
   ↓
Azure Storage
```

Then other applications can consume the file.

---

# 11. Why Blob Storage is useful

Azure Blob Storage is useful for:

```text
Files
CSV
JSON
Archives
Exports
Data exchange
Large objects
```

Architecture:

```text
SQL MI
   |
   v
Azure Storage
   |
   +-- Orders.csv
   +-- Customers.csv
   +-- Archive/
```

This is more cloud-native than treating the database server as a file server.

---

# 12. Migration scenario #4 — PowerShell

Old:

```text
SQL Agent
   ↓
PowerShell
   ↓
C:\Scripts\Cleanup.ps1
```

Ask:

```text
What does the script actually do?
```

If it only executes SQL:

```text
Rewrite as T-SQL
```

If it manages Azure resources:

```text
Azure Automation / appropriate Azure tooling
```

If it processes files:

```text
Azure storage + orchestration
```

The correct destination depends on the responsibility of the script.

---

# 13. Migration scenario #5 — ETL

Suppose:

```text
SQL Agent
   ↓
SSIS package
   ↓
Extract
   ↓
Transform
   ↓
Load
```

This is more complex.

You need to evaluate:

```text
SSIS compatibility
Package dependencies
File paths
Credentials
Data sources
Destinations
Network connectivity
Runtime
Scheduling
```

Depending on requirements, you may retain supported SSIS architecture or modernize the pipeline using Azure-native data integration services.

---

# 14. Important architecture question

Don't ask:

> "How do I make this exact old job run in Azure?"

Ask:

> **"What business function does this job perform, and what's the best Azure architecture for that function?"**

That's a much stronger cloud migration approach.

---

# 15. Migration scenario #6 — Payment processing

Old:

```text
SQL Agent
   ↓
Every 15 minutes
   ↓
Process payments
```

Maybe this is fine.

But investigate:

```text
Does it call an external payment API?
Does it need local files?
Does it require Windows credentials?
Does it rely on server-specific configuration?
Is it idempotent?
```

If it is purely SQL:

```text
SQL Agent
   ↓
Stored procedure
```

may be perfectly reasonable.

---

# 16. Migration scenario #7 — Application notification

Old job:

```text
SQL Agent
   ↓
Send email
```

Don't assume your old SQL Server mail configuration can simply be copied.

Review:

```text
SMTP configuration
Authentication
Security
Network restrictions
Notification requirements
```

Cloud environments often require redesigned notification paths.

---

# 17. Migration scenario #8 — Network share

Old job:

```text
SQL Server
    |
    +---- \\FileServer\Reports
```

After migration:

```text
SQL MI
    |
    X
Old file server
```

Maybe network connectivity can be established depending on architecture, but don't assume it.

You need to evaluate:

```text
VPN/ExpressRoute
DNS
Routing
Authentication
SMB requirements
Firewall
Network security
```

Or redesign the file workflow around Azure Storage.

---

# 18. Migration scenario #9 — Credentials

Old job may use:

```text
Windows credential
SQL credential
Proxy
Service account
```

In Azure, you should reassess the identity model.

Possible architecture:

```text
Managed Identity
      ↓
Azure resource
      ↓
Target service
```

or:

```text
Key Vault
      ↓
Secret
      ↓
Application/automation
```

Avoid copying old service-account practices into the cloud without evaluation.

---

# 19. Migration scenario #10 — Job owner

SQL Agent jobs have owners.

You need to check:

```text
Who owns the job?
Does that login still exist?
Will it be available after migration?
Does the job require that identity?
```

A common migration problem is:

```text
Job imported
   ↓
Owner/login doesn't exist
   ↓
Job behaves unexpectedly
```

Ownership and execution context should be validated after migration.

---

# 20. Migration scenario #11 — Disabled jobs

Don't migrate blindly.

Suppose inventory shows:

```text
250 jobs
```

but:

```text
80 are disabled
```

Why?

Maybe they're:

```text
Old
Unused
Temporary
Deprecated
Emergency jobs
Development jobs
```

Don't automatically migrate technical debt.

---

# 21. Migration principle

Before migration:

```text
DISCOVER
```

Then:

```text
CLASSIFY
```

Then:

```text
CLEAN UP
```

Then:

```text
MIGRATE
```

Then:

```text
VALIDATE
```

---

# 22. Migration workflow

Here's a good enterprise process:

```text
                    JOB MIGRATION
                         |
                         v
                    Inventory
                         |
                         v
                    Dependency
                      analysis
                         |
                         v
                     Classify
                         |
              +----------+----------+
              |          |          |
             Keep      Modify    Redesign
              |          |          |
              +----------+----------+
                         |
                         v
                    Test in Azure
                         |
                         v
                  Performance test
                         |
                         v
                    Production
```

---

# 23. Testing

Never say:

> "The job exists, so migration is complete."

Test:

```text
Does it start?
Does every step work?
Does the schedule work?
Does authentication work?
Does it produce the expected output?
Does it handle failure?
Does retry work?
Does notification work?
Does it affect performance?
```

---

# 24. Performance testing

Suppose old job:

```text
On-prem
Duration = 45 minutes
```

After migration:

```text
SQL MI
Duration = 2 hours
```

The job technically works.

But migration isn't successful if your business requires:

```text
Complete by 6 AM
```

You need to investigate.

---

# 25. Job runtime analysis

Track:

```text
Start time
End time
Duration
CPU
I/O
Waits
Blocking
Rows processed
Failures
Retries
```

Compare:

```text
Before migration
       vs
After migration
```

---

# 26. Dependency mapping

Create a dependency map.

Example:

```text
Job: DailySalesReport

SQL MI
 |
 +-- SalesDB
 |
 +-- Stored Procedure
 |
 +-- Azure Storage
 |
 +-- Reporting System
 |
 +-- Email/notification
```

Now you know that migrating the SQL job alone isn't enough.

---

# 27. Real enterprise example

Suppose a company has:

```text
300 SQL Agent jobs
```

After assessment:

```text
180 → Keep in SQL Agent
60  → Modify
40  → Move to Azure Automation
15  → Move to Data Factory
5   → Retire
```

That's a realistic migration strategy.

You don't need to force every job into one platform.

---

# 28. Why retiring jobs matters

Suppose a job:

```text
ArchiveOldCustomers
```

hasn't run successfully in 18 months.

Don't waste migration effort.

Ask:

```text
Is it still required?
Who owns it?
What business process depends on it?
```

Sometimes the best migration is:

> **Delete the obsolete job.**

---

# 29. SQL Agent vs Azure Data Factory

A simple comparison:

| Requirement            | SQL Agent | Azure Data Factory  |
| ---------------------- | --------- | ------------------- |
| T-SQL scheduling       | Excellent | Good                |
| SQL maintenance        | Excellent | Not primary purpose |
| Multi-system ETL       | Limited   | Excellent           |
| File movement          | Depends   | Excellent           |
| Cloud orchestration    | Limited   | Excellent           |
| SQL Server familiarity | Excellent | Different model     |

Don't interpret this as "ADF replaces SQL Agent."

They solve different problems.

---

# 30. SQL Agent vs Azure Automation

Azure Automation is useful when you need broader automation around Azure resources and scripts.

Conceptually:

```text
SQL Agent
   ↓
SQL-centric automation
```

while:

```text
Azure Automation
   ↓
Azure / infrastructure / scripting automation
```

There can be overlap, but the architectural intent differs.

---

# 31. Real scenario — database maintenance

Suppose you need:

```text
Every Sunday
3 AM
Run maintenance across 50 databases
```

SQL Agent can be a natural fit when the operation is SQL-centric.

Architecture:

```text
SQL Agent
   ↓
Maintenance procedure
   ↓
Databases
```

---

# 32. Real scenario — copy files from SFTP

Requirement:

```text
Every hour
 ↓
Connect to SFTP
 ↓
Download files
 ↓
Transform
 ↓
Load SQL MI
```

This is not primarily a SQL maintenance problem.

A data integration/orchestration service may be more appropriate.

Conceptually:

```text
SFTP
 ↓
Data integration
 ↓
Transform
 ↓
SQL MI
```

---

# 33. Real scenario — Azure resource cleanup

Requirement:

```text
Every night
Find unused Azure resources
 ↓
Tag/report them
```

That's not really a database job.

An Azure automation mechanism is more appropriate.

---

# 34. The biggest migration mistake

The biggest mistake is:

> **Lift-and-shift everything without understanding dependencies.**

You copy:

```text
Jobs
Scripts
Credentials
Schedules
```

and assume everything will work.

Then production discovers:

```text
Missing file path
Missing credential
Network failure
Broken notification
Unsupported command
Different permissions
Different timing
```

---

# 35. Better approach

Use:

```text
Assess
 ↓
Simplify
 ↓
Modernize
 ↓
Migrate
 ↓
Validate
```

Not:

```text
Copy
 ↓
Hope
```

---

# 36. Interview question

### "How would you migrate 500 SQL Agent jobs to SQL MI?"

Strong answer:

> "I'd first inventory the jobs and classify them by SQL-only functionality, external dependencies, security context, file-system requirements, SSIS or PowerShell dependencies, and business criticality. SQL-centric jobs that are supported could remain in SQL Agent. Jobs with Azure or cross-service dependencies might need modification or redesign using services such as Azure Automation or Data Factory. I'd test scheduling, execution context, dependencies, performance, failure handling, and notifications before production cutover."

That's a very good interview response.

---

# 37. Interview question

### "Would you migrate every SQL Agent job?"

Answer:

> "No. I'd assess whether each job is still required and whether SQL Agent is still the appropriate execution platform. Some jobs can remain unchanged, some require modification, some are better redesigned using Azure-native services, and obsolete jobs should be retired."

---

# 38. Interview question

### "What is a cloud-native migration?"

It doesn't necessarily mean:

> "Rewrite everything."

It means:

> **Use managed cloud capabilities where they provide operational, security, scalability, or reliability benefits while retaining appropriate SQL-native capabilities where they remain the best fit.**

---

# 39. Your migration checklist

For every SQL Agent job:

```text
☐ Job name
☐ Business purpose
☐ Owner
☐ Enabled?
☐ Schedule
☐ Steps
☐ T-SQL
☐ PowerShell
☐ SSIS
☐ External executables
☐ File paths
☐ Network paths
☐ Credentials
☐ Proxy
☐ Notifications
☐ Dependencies
☐ Runtime
☐ Failure behavior
☐ Retry behavior
☐ Business criticality
☐ Target architecture
☐ Test result
```

---

# 40. The complete mental model

Remember:

```text
          ON-PREM JOB
               |
               v
          What does it do?
               |
       +-------+-------+
       |       |       |
      SQL    Files   External
       |       |       |
       v       v       v
   SQL Agent Storage  Azure
                     Service
```

Don't migrate the **implementation** blindly.

Migrate the **business capability**.

---

# 41. Senior DBA takeaway

When you hear:

> "We are migrating SQL Server to Azure SQL Managed Instance."

Don't think only:

```text
Database
```

Think:

```text
Database
+
Security
+
Network
+
Jobs
+
Backups
+
Monitoring
+
ETL
+
Applications
+
File dependencies
+
Credentials
+
DR
```

That's the complete workload.

---

## Next — Lesson 79: SQL MI Monitoring & Troubleshooting

We'll start one of the most important DBA areas:

**"The application is slow — how do you find the root cause?"**

We'll cover:

* CPU
* memory
* I/O
* waits
* blocking
* deadlocks
* long-running queries
* Query Store
* execution plans
* Extended Events
* DMVs
* Azure Monitor
* Log Analytics
* alerts
* real-time troubleshooting
* and a complete **production incident where SQL MI suddenly becomes slow at 10:15 AM**.
