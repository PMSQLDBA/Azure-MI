# Lesson 16 — Advanced Azure SQL Managed Instance Administration

The goal now is to move from:

> **"I understand MI"**

to:

> **"I can operate and troubleshoot MI in production."**

---

## 1. What does an MI DBA actually manage?

Think of your responsibilities in five layers:

```text
                 AZURE SQL MI DBA
                       |
      +----------------+----------------+
      |                |                |
      v                v                v
   Database        Performance       Security
      |                |                |
      v                v                v
   Recovery        Troubleshooting    Access
      |
      v
   Operations
```

You don't manage the underlying physical server the way you would with traditional SQL Server.

Instead, you manage the **database workload and service configuration**.

---

# 2. Database administration

Your daily work can include:

```text
CREATE DATABASE
ALTER DATABASE
DROP DATABASE
Database properties
Compatibility level
Users
Roles
Permissions
Schemas
Indexes
Statistics
Stored procedures
Views
Triggers
```

For example:

```sql
CREATE DATABASE SalesDB;
```

Then:

```sql
USE SalesDB;
GO
```

Create a schema:

```sql
CREATE SCHEMA Sales;
GO
```

Create a table:

```sql
CREATE TABLE Sales.Orders
(
    OrderID     bigint        NOT NULL,
    CustomerID  bigint        NOT NULL,
    OrderDate   datetime2     NOT NULL,
    Amount      decimal(18,2) NOT NULL
);
```

---

# 3. Production database design

Suppose you're building:

```text
SalesDB
```

with:

```text
Orders
Customers
Products
Payments
```

A poor design might put everything into one generic schema.

A cleaner design could be:

```text
SalesDB
 |
 +-- Sales
 |    +-- Orders
 |    +-- OrderItems
 |
 +-- Customer
 |    +-- Customers
 |
 +-- Product
 |    +-- Products
 |
 +-- Payment
      +-- Payments
```

Schemas help with:

* Organization
* Security
* Ownership
* Maintainability

---

# 4. Database users vs logins

This is a very common interview question.

### Login

Instance-level security principal.

Conceptually:

```text
SQL MI
 |
 +-- LoginA
 +-- LoginB
```

### User

Database-level principal.

```text
SalesDB
 |
 +-- UserA
 +-- UserB
```

Relationship:

```text
Login
  |
  v
User
  |
  v
Database permissions
```

Don't confuse them.

---

# 5. Roles

Instead of granting permissions individually:

```sql
GRANT SELECT ON Sales.Orders TO UserA;
GRANT SELECT ON Sales.Customers TO UserA;
GRANT SELECT ON Sales.Products TO UserA;
```

You can create a role:

```sql
CREATE ROLE ReportingReader;
```

Then:

```sql
GRANT SELECT ON SCHEMA::Sales TO ReportingReader;
```

And:

```sql
ALTER ROLE ReportingReader
ADD MEMBER UserA;
```

Much easier to manage.

---

# 6. Least privilege

Suppose a reporting application only needs:

```text
SELECT
```

Don't give:

```text
db_owner
```

Instead:

```text
Reporting App
      |
      v
ReportingReader
      |
      v
SELECT permissions
```

This is:

> **Least privilege.**

A production DBA should constantly think:

> "What is the minimum permission this account needs?"

---

# 7. Microsoft Entra authentication

Modern Azure environments often integrate SQL authentication with Microsoft Entra ID.

Conceptually:

```text
User
 |
 v
Microsoft Entra ID
 |
 v
SQL MI
 |
 v
Database
 |
 v
Permissions
```

Advantages include:

* Centralized identity
* Integration with Azure identity
* Reduced dependence on SQL passwords
* Better enterprise access management

---

# 8. Managed identity

Suppose an Azure application needs database access.

Instead of:

```text
Application
   |
   +-- username
   +-- password
```

you can use an identity-based approach:

```text
Azure Application
       |
       v
Managed Identity
       |
       v
Microsoft Entra
       |
       v
SQL MI
```

This avoids embedding database passwords in application configuration.

---

# 9. Secret management

Never put production credentials directly into:

```text
Source code
Git repository
Plain-text config
```

A common Azure pattern is:

```text
Application
    |
    v
Identity
    |
    v
Key Vault / identity-based access
    |
    v
Secret or credential
```

The exact architecture depends on the application.

---

# 10. Encryption

You should understand three concepts separately.

### Encryption at rest

Protects stored data.

```text
Disk/storage
    |
    v
Encrypted
```

### Encryption in transit

Protects traffic:

```text
Application
    |
    | encrypted connection
    v
SQL MI
```

### Transparent Data Encryption

TDE protects database files at rest.

Remember:

> **TDE does not mean the network connection is automatically protected.**

Those are different security layers.

---

# 11. Auditing

Suppose someone changes:

```sql
UPDATE Customers
SET CreditLimit = 0;
```

Management asks:

> "Who did this?"

Without appropriate auditing, you may have difficulty answering.

Auditing helps establish:

```text
Who?
What?
When?
Where?
```

A production security architecture should define:

```text
Audit policy
Retention
Monitoring
Alerting
Review process
```

---

# 12. SQL Agent administration

MI supports SQL Agent capabilities, which makes it familiar to traditional SQL Server DBAs.

Example job:

```text
Nightly Maintenance
       |
       +-- Check statistics
       +-- Perform approved maintenance
       +-- Cleanup staging data
       +-- Send notification
```

But don't blindly implement old SQL Server maintenance scripts.

In a managed environment, always ask:

> "Is this task still necessary?"

---

# 13. Example: index maintenance

A legacy environment might have:

```text
Every night:
Rebuild every index
```

That's often a bad blanket strategy.

Instead:

```text
Index fragmentation
       |
       v
Is it meaningful?
       |
       v
Is the index actually used?
       |
       v
Would maintenance help?
```

You should base maintenance on workload and evidence.

---

# 14. Statistics

Statistics help the optimizer estimate:

```text
How many rows will this query return?
```

Suppose:

```sql
SELECT *
FROM Orders
WHERE CustomerID = 5000;
```

The optimizer needs to estimate the number of matching rows.

If statistics are inaccurate:

```text
Bad estimate
    ↓
Bad execution plan
    ↓
Slow query
```

So statistics are extremely important to performance.

---

# 15. Example of a bad estimate

Actual:

```text
CustomerID = 5000
Rows = 5,000,000
```

Optimizer estimates:

```text
Rows = 5
```

Now the optimizer may choose a plan appropriate for five rows.

But it actually processes millions.

Result:

```text
Poor plan
   ↓
High IO
   ↓
High duration
```

This is why execution plans matter.

---

# 16. Lesson 17 — Execution Plans

Now let's go deeper.

Suppose:

```sql
SELECT OrderID, OrderDate, Amount
FROM Sales.Orders
WHERE CustomerID = 100;
```

SQL Server has to decide:

```text
How should I find these rows?
```

Possible choices:

```text
Table Scan
Index Scan
Index Seek
```

---

# 17. Index Seek

Conceptually:

```text
Index
 |
 +-- CustomerID = 100
       |
       v
   matching rows
```

Usually efficient when the predicate is selective and the index supports the query.

---

# 18. Table Scan

Conceptually:

```text
Table
 |
 +-- Row 1
 +-- Row 2
 +-- Row 3
 ...
 +-- Row 100,000,000
```

SQL Server examines a large portion of the table.

A scan isn't automatically bad.

This is important.

If you request:

```text
90% of a table
```

a scan may actually be the correct strategy.

So:

> **Seek = good and scan = bad is an oversimplification.**

---

# 19. Key Lookup

Imagine:

```text
Index Seek
    |
    v
Find CustomerID
    |
    v
Need additional columns
    |
    v
Key Lookup
```

If there are only a few rows:

```text
Excellent
```

If there are millions:

```text
Potentially expensive
```

This is where a covering index may help.

---

# 20. Covering index

Query:

```sql
SELECT OrderDate, Amount
FROM Sales.Orders
WHERE CustomerID = @CustomerID;
```

Potential index:

```sql
CREATE INDEX IX_Orders_CustomerID
ON Sales.Orders(CustomerID)
INCLUDE(OrderDate, Amount);
```

Conceptually:

```text
Index
 |
 +-- CustomerID
 +-- OrderDate
 +-- Amount
```

The engine may obtain everything it needs from the index.

But again:

> Don't create indexes without workload analysis.

---

# 21. Join performance

Suppose:

```sql
SELECT c.CustomerName,
       o.OrderDate,
       o.Amount
FROM Customer.Customers c
JOIN Sales.Orders o
    ON c.CustomerID = o.CustomerID;
```

Possible join algorithms include:

```text
Nested Loops
Hash Match
Merge Join
```

Each has situations where it makes sense.

---

# 22. Nested Loops

Conceptually:

```text
Small input
    |
    v
Nested Loops
    |
    +----> Search second table
    +----> Search second table
    +----> Search second table
```

Excellent when:

```text
Outer input = small
Inner side = efficiently indexed
```

Can become expensive when the outer input is huge.

---

# 23. Hash Match

Conceptually:

```text
Input A
   |
   v
Hash table
   ^
   |
Input B
```

Often useful for larger, unsorted inputs.

But it can require significant memory.

If memory isn't sufficient:

```text
Hash operation
      ↓
Spill to tempdb
      ↓
Performance degradation
```

---

# 24. Sort

Sort operations can also consume memory.

Example:

```sql
SELECT *
FROM Sales.Orders
ORDER BY OrderDate DESC;
```

Potentially:

```text
Read rows
   ↓
Sort
   ↓
Return rows
```

Large sorts can become expensive.

---

# 25. Execution-plan investigation

When you see a slow query:

Don't ask:

> "Which operator is red?"

Ask:

```text
What is the query trying to do?
How many rows are expected?
How many rows are actually processed?
Which operator dominates cost/work?
Are there scans?
Are there lookups?
Are there spills?
Are estimates wrong?
Are joins appropriate?
```

That's real plan analysis.

---

# 26. Lesson 18 — Indexing strategy

Indexes solve specific access patterns.

Imagine:

```text
Orders
100 million rows
```

Query:

```sql
WHERE CustomerID = 123
```

Without useful indexing:

```text
Potential large scan
```

With appropriate indexing:

```text
Seek → matching rows
```

But indexes have a cost.

Every index means:

```text
Storage
+
INSERT overhead
+
UPDATE overhead
+
DELETE overhead
+
Maintenance
```

So:

> **More indexes ≠ better performance.**

---

# 27. Index decision process

Before creating an index:

```text
1. Identify expensive query
2. Inspect execution plan
3. Determine missing/inefficient access path
4. Check existing indexes
5. Estimate benefit
6. Estimate write/storage cost
7. Test
8. Deploy
9. Measure
```

---

# 28. Lesson 19 — Real production troubleshooting lab

Now you're going to act as the DBA.

### Incident

At 11:00 AM:

```text
Order API latency:
300 ms
```

At 11:15 AM:

```text
Order API latency:
8 seconds
```

Azure metrics:

```text
CPU = 45%
IO = Normal
Memory = Normal
```

Blocking:

```text
0
```

Deadlocks:

```text
0
```

Query Store shows:

```text
Query:
SELECT ...
FROM Orders
WHERE CustomerID = @CustomerID

Before:
250 ms

Now:
7.5 seconds
```

Execution plan changed from:

```text
Index Seek
```

to:

```text
Table Scan
```

---

## Your diagnosis

This is most likely a:

> **Query plan regression.**

Not:

```text
CPU saturation
Blocking
MI infrastructure failure
```

---

# 29. What changed?

Now investigate:

```text
Statistics?
Index?
Data distribution?
Parameter sensitivity?
Plan compilation?
Application deployment?
```

Suppose you discover:

```text
Statistics became stale
```

Then:

```text
Bad cardinality estimate
      ↓
Different plan
      ↓
Table Scan
      ↓
7.5 sec
```

That's your root cause.

---

# 30. Correct production response

Don't immediately:

```text
CREATE INDEX ...
```

Instead:

```text
Confirm regression
      ↓
Compare plans
      ↓
Identify root cause
      ↓
Correct statistics / plan / query
      ↓
Retest
      ↓
Monitor
```

This is evidence-based troubleshooting.

---

# Lesson 20 — Complete Migration Project

Let's now combine migration concepts.

### Source

```text
SQL Server 2019
Windows Server
```

### Databases

```text
SalesDB = 2 TB
OrdersDB = 4 TB
CustomerDB = 500 GB
```

### Requirements

```text
24/7 application
RPO ≈ 0 / minimal data loss
RTO < 30 minutes
Hybrid connectivity
```

---

## Phase 1 — Discovery

Inventory:

```text
Databases
Users
Logins
Jobs
Linked servers
SSIS
Applications
Reports
ETL
Network dependencies
```

---

## Phase 2 — Assessment

Determine:

```text
Compatibility
Feature support
Database size
Growth
Performance
Dependencies
Security
```

---

## Phase 3 — Design

Design:

```text
Azure subscription
Resource groups
VNet
Subnet
MI
Networking
Identity
Security
Monitoring
DR
```

---

## Phase 4 — Test migration

Do not make the first migration your production migration.

Instead:

```text
Production SQL Server
       |
       v
Test migration
       |
       v
SQL MI test environment
       |
       v
Application testing
```

Measure:

```text
Performance
Functionality
Compatibility
Data correctness
```

---

# 31. Production cutover

Example:

```text
22:00  Freeze application changes
22:05  Stop writes
22:10  Final synchronization
22:15  Validate target
22:18  Redirect application
22:20  Start application
22:25  Smoke tests
22:30  Business validation
```

If successful:

```text
Production → MI
```

If critical failure:

```text
Rollback according to runbook
```

---

# Lesson 21 — Automation

A modern Azure DBA shouldn't manually perform everything.

Think:

```text
Manual
   ↓
Script
   ↓
Automation
   ↓
Repeatable operation
```

Examples:

```text
Database inventory
Health checks
Permission reports
Performance reports
Deployment
Monitoring
Alerting
Migration validation
```

You may use:

```text
T-SQL
PowerShell
Azure CLI
Infrastructure-as-code tools
Automation pipelines
```

The exact tooling depends on the organization's standards.

---

# 32. Example health-check concept

A daily health report could collect:

```text
Database
Size
CPU
Top queries
Blocking
Deadlocks
Failed jobs
Storage
Security events
```

Then:

```text
Automation
    |
    v
Daily report
    |
    v
DBA
```

Instead of manually checking everything.

---

# Lesson 22 — Interview Preparation

Now let's start preparing you for actual interviews.

### Question 1

**What is Azure SQL Managed Instance?**

Strong answer:

> Azure SQL Managed Instance is a fully managed Azure PaaS database service based on the SQL Server engine. It provides high SQL Server compatibility while Microsoft manages much of the underlying infrastructure, patching, availability, and platform operations. It is useful for organizations migrating existing SQL Server workloads to Azure without requiring the level of application changes often associated with more database-platform-specific PaaS offerings.

---

### Question 2

**Why choose MI instead of SQL Database?**

Answer around:

```text
SQL Server compatibility
Instance-level capabilities
Migration requirements
Existing applications
SQL Agent
Cross-database scenarios
Networking
Enterprise workloads
```

But don't claim every SQL Server feature is supported identically.

Always verify feature compatibility for the specific workload.

---

### Question 3

**How do you troubleshoot high CPU?**

Answer:

```text
1. Confirm CPU pattern
2. Identify top CPU-consuming queries
3. Check Query Store
4. Review execution plans
5. Check recent changes
6. Check statistics/indexes
7. Optimize workload
8. Scale if necessary
9. Validate
```

---

### Question 4

**How do you troubleshoot blocking?**

Answer:

```text
Identify head blocker
      ↓
Identify transaction
      ↓
Determine why it remains open
      ↓
Identify affected sessions
      ↓
Assess business impact
      ↓
Resolve safely
      ↓
Prevent recurrence
```

---

### Question 5

**HA vs DR?**

```text
HA = availability during failures
DR = recovery from major disasters
```

---

# 33. Your current skill level

After these modules, your knowledge map looks like:

```text
AZURE SQL MI
│
├── Fundamentals              ✅
├── Architecture              ✅
├── Networking                ✅
├── Security                  ✅
├── Backup / PITR             ✅
├── HA / DR                   ✅
├── Monitoring                ✅
├── Troubleshooting           ✅
├── Migration                 ✅
├── Query Store               ✅
├── Execution plans           ✅
├── Indexing                  ✅
├── Statistics                ✅
├── Production operations     ✅
└── Interview preparation     🔄
```

We're now entering the **hands-on DBA phase**.

---

# Next: Lesson 23 — Your First Real DBA Lab

I'm going to stop giving you the answer first.

I'll give you a **production incident**, and **you diagnose it**.

### Incident #1

Your monitoring system reports:

```text
Production SQL MI

CPU:                 92%
Memory:              Normal
Storage:             61%
IO:                  Normal

Application latency: 9 seconds
Blocking:            0
Deadlocks:           0
```

Query Store shows:

```text
Query A
Average duration:    8.4 sec
CPU:                 Very High
Executions:          12,000/min

Query B
Average duration:    150 ms
CPU:                 Low

Query C
Average duration:    300 ms
CPU:                 Low
```

Query A:

```sql
SELECT
    CustomerID,
    SUM(OrderAmount)
FROM Sales.Orders
WHERE OrderDate >= @StartDate
GROUP BY CustomerID;
```

Execution plan shows:

```text
Clustered Index Scan
        ↓
  250 million rows
        ↓
     Hash Match
        ↓
      Sort
```

### You are the DBA.

Tell me:

**1. What is your first conclusion?**

**2. What would you investigate?**

**3. Would you immediately scale the MI?**

**4. What indexing or query changes would you consider?**

**5. What information would you want before making the production change?**

Answer those five, and I'll grade your response like a **real senior SQL/Azure DBA interview** and then give you Incident #2, which will involve **blocking and a head blocker**.
