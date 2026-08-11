**Lesson 9 — SQL Server Features on Azure Managed Instance**.

This lesson is especially important if you're coming from **SQL Server 2019 administration**, because one of the biggest reasons companies choose Azure SQL Managed Instance is that it feels much more like a traditional SQL Server environment than Azure SQL Database.

# Lesson 9 — SQL Agent, Linked Servers, Cross-Database Queries & Instance-Level Features

Imagine this existing environment:

```text
SQL Server 2019
│
├── 25 databases
├── SQL Agent jobs
├── Linked servers
├── Cross-database queries
├── Logins
├── Credentials
├── Database Mail
├── Maintenance jobs
├── ETL
└── Reporting
```

The company wants to move it to:

```text
Azure SQL Managed Instance
```

The big question is:

> **"How much of my SQL Server ecosystem can I keep?"**

The answer is: **a lot**, but you still have to assess individual features and dependencies.

---

# 1. SQL Server vs Managed Instance

Think of the Azure SQL family like this:

```text
                    Azure SQL
                       │
          ┌────────────┼────────────┐
          │            │            │
          v            v            v
     SQL Database      MI      SQL Server on VM
          │            │            │
     PaaS database   PaaS       IaaS
                     instance
```

For a traditional SQL Server DBA:

### Azure SQL Database

Feels more like:

```text
Database as a service
```

### Azure SQL Managed Instance

Feels more like:

```text
SQL Server instance
+
Azure managed infrastructure
```

That's why MI is often attractive for SQL Server migrations.

---

# 2. SQL Server Agent

Suppose your existing SQL Server has:

```text
SQL Agent
│
├── Full backup
├── Index maintenance
├── ETL
├── Data cleanup
├── Reporting refresh
└── Alerting
```

Managed Instance supports SQL Server Agent functionality, although there are MI-specific limitations and differences you need to validate.

For example:

```text
Job: Load Sales

01:00 AM
   ↓
Execute ETL
   ↓
Load staging
   ↓
Transform
   ↓
Load warehouse
```

This is a familiar model for SQL Server DBAs.

---

# 3. But here's the migration trap

Your SQL Agent job might contain:

```text
Step 1:
EXEC dbo.LoadData

Step 2:
C:\Scripts\ProcessFiles.ps1

Step 3:
Copy file from D:\Exports

Step 4:
Send email
```

The SQL portion may migrate easily.

But the infrastructure dependencies may not.

For every job ask:

```text
What does this job actually touch?
```

---

# 4. SQL Agent dependency assessment

For each job, inspect:

```text
Job name
Owner
Schedule
Steps
Credentials
Proxy
PowerShell
CmdExec
SSIS
File system
Network shares
Linked servers
Database dependencies
Email notifications
```

Build a migration matrix:

| Job             | SQL only? | External dependency | Action   |
| --------------- | --------- | ------------------- | -------- |
| Nightly cleanup | Yes       | None                | Migrate  |
| ETL             | No        | File share          | Assess   |
| PowerShell job  | No        | Server OS           | Redesign |
| Report refresh  | Yes       | SSRS                | Assess   |
| Data export     | No        | Local disk          | Redesign |

This is how you approach migration professionally.

---

# 5. Linked Servers

Suppose SQL Server 2019 has:

```sql
SELECT *
FROM [ERP-SERVER].ERPDB.dbo.Customers;
```

This means your SQL Server is reaching another data source.

Architecture:

```text
SQL MI
   |
   | Linked server
   v
ERP database
```

Before migration, ask:

```text
Where is ERP?
How is authentication performed?
What provider is being used?
Can MI reach it over the network?
Is TLS supported?
Are credentials valid?
```

---

# 6. Network problem

Suppose:

```text
MI
 |
 X
 |
ERP Server
```

The linked server configuration may be correct, but networking could still prevent communication.

You need:

```text
MI
 ↓
VNet
 ↓
Network routing
 ↓
Firewall
 ↓
DNS
 ↓
ERP
```

This is another reason networking is so important for MI.

---

# 7. Cross-database queries

This is one of MI's major conveniences.

Suppose you have:

```text
SalesDB
CustomerDB
ReportingDB
```

Your application might execute:

```sql
SELECT
    c.CustomerName,
    SUM(o.OrderAmount)
FROM CustomerDB.dbo.Customers c
JOIN SalesDB.dbo.Orders o
    ON c.CustomerID = o.CustomerID
GROUP BY c.CustomerName;
```

This is a cross-database query.

Because these databases are on the same MI, this type of architecture can be much easier to preserve than moving everything to a database-per-service architecture.

---

# 8. Why this matters in migrations

Imagine your application has:

```text
50 databases
```

and hundreds of queries like:

```text
DatabaseA.dbo.Table
DatabaseB.dbo.Table
DatabaseC.dbo.Table
```

Moving to an architecture where each database is isolated could require major application redesign.

Managed Instance can reduce that migration effort because it provides instance-level capabilities and supports multiple databases within the instance.

---

# 9. Cross-database transactions

Suppose:

```text
OrderDB
PaymentDB
```

An application performs:

```text
Create Order
      +
Create Payment
```

and expects both operations to succeed or fail together.

That's a transaction design issue.

Before migration, identify:

```text
Cross-database transactions
Distributed transactions
Linked-server transactions
Application transaction scopes
```

Don't assume that every distributed transaction pattern will behave exactly like your old environment.

Assess it explicitly.

---

# 10. Logins vs users

This is a classic SQL Server concept.

### Login

Server/instance-level security principal.

```text
Login
  ↓
SQL Managed Instance
```

### User

Database-level security principal.

```text
Login
  ↓
Database User
  ↓
Permissions
```

Example:

```text
Login:
app_login

Database:
SalesDB

User:
app_user
```

Migration requires understanding both levels.

---

# 11. Orphaned users

Suppose you restore a database.

You might have:

```text
SalesDB
   |
   v
User: app_user
```

But the corresponding login isn't correctly mapped on the target.

You can end up with an orphaned user/security mapping problem.

So after migration, validate:

```text
Logins
Users
SID mappings
Roles
Permissions
Authentication
```

---

# 12. Least privilege

Don't solve migration issues by giving the application:

```text
sysadmin
```

That's dangerous.

Instead:

```text
Application
   ↓
Login
   ↓
User
   ↓
Required role/permissions
```

Use the minimum permissions necessary.

---

# 13. Managed Identity

In Azure environments, you may also use Azure identity mechanisms rather than relying exclusively on SQL authentication.

Conceptually:

```text
Application
     |
     v
Managed Identity
     |
     v
Azure authentication
     |
     v
SQL Managed Instance
```

This can reduce the need to store SQL passwords in application configuration.

The exact authentication design depends on the application and Azure identity setup.

---

# 14. Database Mail

Traditional SQL Server environments often have:

```text
SQL Agent
   |
   v
Database Mail
   |
   v
SMTP
```

For example:

```text
Job fails
   ↓
SQL Agent
   ↓
Email DBA
```

When migrating, validate:

```text
Mail configuration
SMTP connectivity
Authentication
Security
Job notifications
```

Don't assume an existing SMTP setup will work unchanged.

---

# 15. Maintenance jobs

Traditional SQL Server often has jobs like:

```text
01:00 AM
↓
Index maintenance

02:00 AM
↓
Statistics update

03:00 AM
↓
Cleanup
```

In Azure SQL Managed Instance, you need to determine which maintenance tasks are still appropriate.

Don't blindly migrate every old SQL Agent maintenance job.

Why?

Because Azure services already manage portions of:

```text
Infrastructure
Storage
Patching
Backups
Platform maintenance
```

Your old on-premises maintenance strategy may have been designed around responsibilities that Azure now handles.

---

# 16. Example: old maintenance job

On-premises:

```sql
ALTER INDEX ALL ON dbo.Orders
REBUILD;
```

Every Sunday.

You migrate the job unchanged.

But your database is:

```text
4 TB
```

and the rebuild causes:

```text
CPU ↑
IO ↑
Log generation ↑
Blocking ↑
```

Now production users complain Monday morning.

The lesson:

> **Migration is not "copy everything exactly."**

It's:

> **Preserve required functionality while redesigning what no longer makes sense.**

---

# 17. SQL Server Agent + Azure architecture

A useful mental model:

```text
                 SQL Managed Instance
                        |
        +---------------+---------------+
        |               |               |
        v               v               v
   SQL Agent        Databases       Security
        |
        +---- Jobs
        +---- Schedules
        +---- Alerts
        +---- Notifications
```

But external dependencies may sit outside:

```text
SQL MI
  |
  +---- Azure Storage
  |
  +---- File share
  |
  +---- ERP
  |
  +---- API
  |
  +---- SMTP
```

Those dependencies need network and security validation.

---

# 18. Real migration scenario

ABC Retail has a job:

```text
Job: NightlyOrderExport

Step 1
EXEC SalesDB.dbo.PrepareExport

Step 2
bcp SalesDB.dbo.Orders out D:\Exports\Orders.csv

Step 3
PowerShell uploads file to partner server

Step 4
Send email
```

Don't say:

> "SQL Agent supports jobs, so we're done."

Break it apart.

### Step 1

```text
Stored procedure
```

Likely straightforward.

### Step 2

```text
Local D:\ drive
```

Needs redesign because you're moving from your own Windows server infrastructure.

### Step 3

```text
External partner
```

Requires:

```text
Network
Authentication
Firewall
DNS
```

### Step 4

```text
SMTP
```

Needs validation.

This is real migration analysis.

---

# 19. Instance-level thinking

When moving SQL Server → MI, categorize objects.

### Database-level

```text
Tables
Views
Procedures
Functions
Triggers
Users
Roles
```

### Instance-level

```text
Logins
SQL Agent
Linked servers
Credentials
Server configurations
```

### External infrastructure

```text
File shares
Applications
SMTP
ETL servers
Monitoring
DNS
Network
```

This categorization makes migration planning much easier.

---

# 20. Real DBA migration checklist

Before production cutover:

```text
DATABASE
☐ Compatibility checked
☐ Database migrated
☐ Row counts validated
☐ Objects validated
☐ Indexes validated
☐ Statistics strategy reviewed

SECURITY
☐ Logins
☐ Users
☐ Roles
☐ Permissions
☐ Authentication

SQL AGENT
☐ Jobs
☐ Schedules
☐ Credentials
☐ Proxies
☐ Notifications
☐ External dependencies

NETWORK
☐ DNS
☐ Firewall
☐ VNet
☐ Routing
☐ Private connectivity
☐ Linked server connectivity

APPLICATION
☐ Connection string
☐ Authentication
☐ Retry behavior
☐ Transaction testing
☐ Performance testing

OPERATIONS
☐ Monitoring
☐ Alerts
☐ Backup
☐ DR
☐ Runbooks
☐ Rollback
```

---

# 21. Interview question

If an interviewer asks:

> **"Why choose Managed Instance instead of Azure SQL Database for a SQL Server migration?"**

A good answer is:

> "Managed Instance provides a higher level of instance compatibility with traditional SQL Server, including capabilities such as multiple databases, cross-database scenarios, SQL Server Agent, and other instance-level features. This can reduce application and operational changes when migrating an existing SQL Server workload. However, I would still perform a compatibility and dependency assessment rather than assuming full feature equivalence."

That's a strong DBA answer.

---

# 22. One more important concept: PaaS vs IaaS

Compare:

```text
SQL Server on Azure VM
```

with:

```text
Azure SQL Managed Instance
```

### SQL Server on VM

You manage more:

```text
OS
SQL installation
Patching
VM
Storage
HA infrastructure
Backups
```

### Managed Instance

Azure manages much more of the platform:

```text
VM infrastructure
OS
Patching
Platform HA
Storage infrastructure
```

You focus more on:

```text
Database
Security
Queries
Performance
Application
Data
```

So:

> **MI is a compromise between compatibility and managed service.**

That's one of the most important concepts to remember.

---

# Lesson 9 challenge 🧑‍💻

ABC Retail has this SQL Agent job:

```text
Job: Nightly Customer Export

Step 1:
EXEC CustomerDB.dbo.PrepareExport

Step 2:
bcp CustomerDB.dbo.Customers
out D:\Exports\Customers.csv

Step 3:
PowerShell uploads Customers.csv
to an external partner

Step 4:
Send email notification
```

They migrate the SQL Server to Managed Instance.

The job starts failing.

### Tell me what you would investigate for each step:

**1. Step 1 — stored procedure**

**2. Step 2 — `D:\Exports`**

**3. Step 3 — external partner**

**4. Step 4 — email**

And the most important question:

**Would you simply recreate the job exactly as it existed on SQL Server 2019? Why or why not?**
