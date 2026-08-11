# Lesson 91 — Azure SQL Managed Instance Backup & Restore

This lesson is critical because **HA/DR does not replace backups**.

As a DBA, you need to know not only that SQL MI is protected automatically, but also **what happens when someone deletes data, a deployment corrupts data, or the business asks for a restore to a specific point in time.**

---

# 1. The core idea

Think of SQL MI protection as three separate layers:

```text
                 SQL MI Protection
                       |
        +--------------+--------------+
        |              |              |
        ↓              ↓              ↓
       HA           Backups           DR
        |              |              |
 Availability      Data recovery   Disaster recovery
```

Each solves a different problem.

---

# 2. What is a backup?

A backup is a recoverable copy of database data that can be used to restore the database.

For example:

```text
Production DB
     |
     ↓
Backup
     |
     ↓
Recovery
```

Backups protect against things such as:

```text
Accidental DELETE
Bad UPDATE
Application bugs
Corruption
Bad deployment
Human error
```

---

# 3. HA is not backup

Suppose someone runs:

```sql
DELETE FROM Customers;
```

and commits it.

If the database is protected by HA:

```text
DELETE
  ↓
Primary
  ↓
HA infrastructure
```

The legitimate transaction can be reflected through the HA mechanism.

HA doesn't understand:

> "That DELETE was a mistake."

That's why you need backups.

---

# 4. Point-in-time restore

One of the most important recovery capabilities is:

> **Restore a database to a specific point in time within the available retention window.**

Imagine:

```text
10:00  Normal
10:10  Deployment
10:15  Bad SQL
10:20  Data corruption discovered
```

You may want to recover the database to:

```text
10:14:59
```

instead of restoring only the most recent state.

---

# 5. Example

Before incident:

```text
Customers = 1,000,000 rows
```

Bad deployment:

```sql
DELETE FROM Customers;
```

Now:

```text
Customers = 0 rows
```

You don't want:

```text
Restore today's database
```

if today's backup reflects the bad state.

You want:

```text
Restore to a point before the DELETE.
```

That's the value of point-in-time recovery.

---

# 6. PITR

PITR means:

> **Point-In-Time Restore**

Conceptually:

```text
Backup history
|
|---- 10:00
|---- 10:05
|---- 10:10
|---- 10:15
|---- 10:20
|
+---- Restore here
```

You select the desired recovery point.

---

# 7. Why transaction logs matter

SQL Server recovery relies heavily on transaction log information.

Conceptually:

```text
Full backup
    +
Log history
    ↓
Point-in-time recovery
```

The exact managed implementation is handled by Azure SQL Managed Instance, so you don't manage the underlying backup infrastructure the way you would for a self-managed SQL Server.

---

# 8. SQL Server DBA perspective

On traditional SQL Server, you might manage:

```text
Full backups
Differential backups
Transaction-log backups
Backup jobs
Backup storage
Backup retention
Backup verification
Restore jobs
```

With SQL MI, Azure manages much of the automated backup infrastructure.

Your responsibilities shift toward:

```text
Recovery requirements
Retention
Restore testing
Recovery procedures
Business validation
```

---

# 9. Full backup concept

A full backup represents a recoverable copy of the database at a particular point.

Conceptually:

```text
Database
   |
   ↓
Full backup
   |
   ↓
Restore
```

You don't need to manually run traditional SQL Agent backup jobs just to maintain the standard automated backup chain provided by the managed service.

---

# 10. Differential backups

Traditional SQL Server DBAs know:

```text
Full
  +
Differential
  +
Logs
```

For SQL MI, don't assume you need to manually reproduce your traditional SQL Server backup-job architecture.

Azure SQL Managed Instance provides managed backup capabilities.

The important DBA question becomes:

> **Can I meet the required recovery objectives?**

---

# 11. Transaction-log backups

Transaction-log backups support recovery to a more precise point in time in traditional SQL Server architectures.

In SQL MI, Azure manages the automated backup mechanism that supports point-in-time recovery.

You generally don't manage the underlying backup scheduling yourself.

---

# 12. Why automation is useful

Imagine managing:

```text
500 databases
```

Traditional approach:

```text
500 backup jobs
500 schedules
500 destinations
500 retention policies
```

Managed service:

```text
Azure-managed backup infrastructure
```

This removes a significant amount of operational work.

---

# 13. But automation doesn't eliminate responsibility

A DBA still needs to know:

```text
How long is data recoverable?
Where can it be restored?
How do I restore it?
How long will recovery take?
What data can be lost?
How do I validate the restored database?
```

---

# 14. Retention

Backup retention determines how far back you can recover.

Think:

```text
Today
 |
 +-----------------------------+
 | Available recovery window  |
 +-----------------------------+
 |
Older data eventually expires
```

Your organization's retention requirement might be:

```text
7 days
30 days
Months
Years
```

The appropriate mechanism depends on the requirement.

---

# 15. Operational retention vs long-term retention

These are different business needs.

### Operational recovery

> "We accidentally broke the database yesterday."

Use point-in-time recovery within the available recovery window.

### Long-term compliance

> "We need to retain data for several years."

That is a different requirement and may require **long-term retention (LTR)** or another archival strategy.

---

# 16. LTR

LTR stands for:

> **Long-Term Retention**

It is intended for keeping backups for extended periods.

Example:

```text
Weekly backup
     |
     ↓
Long-term retention
     |
     +---- 1 year
     +---- 3 years
     +---- 7 years
```

Exact retention options depend on the supported service configuration.

---

# 17. Why LTR matters

Suppose your company has a regulatory requirement:

> "Financial database backups must be retained for 7 years."

Normal operational PITR retention isn't necessarily the right solution.

You need a deliberate long-term retention strategy.

---

# 18. Backup vs archive

Don't confuse:

```text
Backup
```

with:

```text
Archive
```

Backup is primarily about recovery.

Archive may be about:

```text
Compliance
Historical access
Records retention
Legal requirements
```

The architecture and storage requirements can differ.

---

# 19. Restore to a new database

A very useful recovery pattern is:

```text
ProductionDB
     |
     ↓
Restore
     |
     ↓
ProductionDB_RestoreTest
```

Why?

You can inspect the recovered database without immediately replacing production.

---

# 20. This is extremely useful for DBAs

Suppose someone says:

> "We accidentally deleted 50,000 customer records."

Instead of immediately destroying the current database and restoring over it, you can:

```text
Production
    |
    ↓
Point-in-time restore
    |
    ↓
Temporary database
    |
    ↓
Compare data
```

Then recover the required records using a controlled process.

---

# 21. Example recovery workflow

Current database:

```text
SalesDB
```

Bad operation occurred at:

```text
14:32
```

You restore to:

```text
14:31
```

as:

```text
SalesDB_Recovery
```

Then compare:

```text
SalesDB
vs
SalesDB_Recovery
```

You identify the missing rows.

Then use an appropriate controlled data-recovery process.

---

# 22. Why not always restore over production?

Because you could lose:

```text
Valid transactions
```

that occurred after the incident.

Example:

```text
14:32 Bad DELETE
14:35 Valid customer orders
14:40 Incident discovered
```

Restoring production to 14:31 would potentially remove valid 14:35 transactions.

That's why a separate restore is often safer for logical data recovery.

---

# 23. Real production incident

At 2:00 PM:

```text
Deployment starts
```

At 2:10 PM:

```text
Bad migration executes
```

At 2:20 PM:

```text
Users report incorrect data
```

At 2:25 PM:

```text
DBA identifies deployment as cause
```

The DBA should preserve evidence and determine the appropriate recovery point.

Possible approach:

```text
Restore to temporary database
        ↓
Validate expected state
        ↓
Identify affected rows
        ↓
Determine recovery strategy
```

---

# 24. Accidental DELETE scenario

Suppose:

```sql
DELETE FROM Orders
WHERE OrderDate < '2025-01-01';
```

The developer intended:

```text
Test environment
```

but executed against production.

Committed.

Now:

```text
Orders
↓
Data missing
```

First questions:

```text
When did the DELETE happen?
How many rows were affected?
What other transactions occurred afterward?
Can we restore to a temporary database?
```

---

# 25. Don't immediately restore

A common junior DBA reaction:

> "Restore the latest backup!"

Wrong approach.

First determine:

```text
Incident time
Recovery point
Business impact
Data generated after incident
Required recovery scope
```

Then choose the recovery method.

---

# 26. Recovery Point Selection

Suppose:

```text
10:00  Normal
10:15  Bad UPDATE
10:20  Bad UPDATE discovered
```

Potential recovery point:

```text
10:14:59
```

But you should validate whether there were legitimate transactions between 10:15 and 10:20 that need to be preserved.

---

# 27. Recovery Time

RTO is not simply:

> "How fast can SQL MI restore?"

It includes the entire recovery workflow:

```text
Incident detected
       ↓
Determine recovery point
       ↓
Initiate restore
       ↓
Database available
       ↓
Validate database
       ↓
Validate application
       ↓
Business sign-off
```

That's the real recovery time.

---

# 28. Recovery validation

Never say:

> "Restore completed, therefore we're done."

Check:

```text
Database state
Tables
Critical row counts
Stored procedures
Users
Permissions
Application connectivity
Business transactions
```

---

# 29. Example validation

Suppose:

```sql
SELECT COUNT(*) FROM Orders;
```

Before incident:

```text
15,000,000
```

After restore:

```text
14,950,000
```

That's a signal to investigate.

You need to verify that the restored point corresponds to the intended recovery state.

---

# 30. Database consistency

After recovery, validate important business objects:

```text
Customers
Orders
Payments
Inventory
Invoices
```

For example:

```text
Orders ↔ OrderDetails
Orders ↔ Customers
Payments ↔ Orders
Inventory ↔ Products
```

A database can be technically online but still be logically wrong for the application.

---

# 31. Backup and security

Backups contain sensitive data.

Think:

```text
Production database
       |
       ↓
Backup
       |
       ↓
Sensitive information
```

Therefore consider:

```text
Encryption
Access control
Retention
Compliance
Auditing
```

---

# 32. Who should be allowed to restore?

Not every developer should have permission to:

```text
Restore production database
```

A controlled process might be:

```text
Developer
   ↓
Incident ticket
   ↓
DBA approval
   ↓
Restore
   ↓
Validation
```

---

# 33. Restore testing

One of the most important DBA rules:

> **A backup that has never been successfully restored is not a fully proven recovery strategy.**

Test restores regularly.

---

# 34. Restore drill

Example monthly test:

```text
1. Select production database
2. Choose recovery point
3. Restore to test environment
4. Validate schema
5. Validate row counts
6. Validate application connectivity
7. Measure restore duration
8. Document results
```

---

# 35. Why restore duration matters

Suppose:

```text
Business RTO = 30 minutes
```

You test a restore and discover:

```text
Restore = 45 minutes
```

Your backup strategy technically works.

But:

```text
Business requirement = NOT MET
```

You need to redesign or optimize the recovery process.

---

# 36. RPO and backups

Suppose:

```text
Business RPO = 15 minutes
```

Your recovery architecture must be capable of meeting that requirement.

Don't say:

> "We have backups."

Ask:

> **"Can the available recovery mechanism actually meet the required RPO?"**

---

# 37. RTO and backups

Similarly:

```text
Business RTO = 1 hour
```

You must test:

```text
Detection
Restore
Validation
Application recovery
```

not just database restore speed.

---

# 38. Backup retention planning

Ask the business:

```text
How much operational recovery history do you need?
How long must backups be retained?
Are there regulatory requirements?
Do we need long-term retention?
How often should restores be tested?
```

---

# 39. Example policy

A company might define:

```text
Operational recovery:
30 days

Long-term retention:
7 years

DR:
Secondary region

Restore testing:
Monthly

DR testing:
Quarterly
```

The exact values should come from business and compliance requirements.

---

# 40. SQL MI vs traditional SQL Server

| Traditional SQL Server         | SQL Managed Instance                               |
| ------------------------------ | -------------------------------------------------- |
| DBA commonly schedules backups | Azure manages automated backup infrastructure      |
| DBA manages backup jobs        | Less infrastructure management                     |
| DBA manages backup destination | Managed service handles standard automated backups |
| DBA performs restores          | DBA initiates/controls recovery                    |
| DBA validates backups          | DBA still must test recovery                       |
| DBA defines retention          | DBA/business define requirements                   |

The DBA role moves from:

```text
Backup operator
```

toward:

```text
Recovery architect
```

---

# 41. What Azure manages

Azure manages much of:

```text
Backup scheduling
Backup infrastructure
Backup storage management
Backup automation
```

You focus on:

```text
Retention requirements
Recovery strategy
Restore operations
Testing
Compliance
```

---

# 42. What you still manage

As a DBA, you still need to understand:

```text
Which database needs recovery?
When did the incident happen?
What recovery point is required?
Where should it be restored?
How will data be validated?
How will the application reconnect?
```

---

# 43. Production disaster example

Imagine:

```text
3:00 PM
Production database corrupted

3:05 PM
Application team reports errors

3:10 PM
DBA identifies bad deployment

3:15 PM
Recovery point selected

3:20 PM
Restore initiated

3:35 PM
Restore completed

3:40 PM
Validation complete

3:45 PM
Application restored
```

Actual RTO:

```text
45 minutes
```

If the requirement was:

```text
RTO ≤ 60 minutes
```

you succeeded.

---

# 44. Data corruption scenario

Suppose application code executes:

```sql
UPDATE Accounts
SET Balance = 0;
```

without a proper WHERE clause.

The database is still:

```text
Healthy
Online
Responsive
```

But the data is logically corrupted.

HA doesn't solve this.

Backup/PITR becomes essential.

---

# 45. Logical vs physical failure

This distinction is important.

### Physical/infrastructure failure

```text
Host failure
Infrastructure failure
Zone disruption
```

Think:

```text
HA
```

### Logical failure

```text
Bad DELETE
Bad UPDATE
Bad deployment
Application bug
```

Think:

```text
Backup / PITR
```

### Regional disaster

```text
Region unavailable
```

Think:

```text
DR
```

---

# 46. Interview question

### "Does Azure SQL Managed Instance automatically back up databases?"

Strong answer:

> "Yes. SQL Managed Instance provides automated backup capabilities managed by Azure, including support for point-in-time recovery. The DBA still needs to understand retention, recovery requirements, restore procedures, and testing."

---

# 47. Interview question

### "What would you do if a developer accidentally deleted data?"

Answer:

```text
1. Stop further destructive activity.
2. Determine exact incident time.
3. Identify affected data.
4. Determine required recovery point.
5. Restore to a separate database where possible.
6. Validate the recovered data.
7. Recover only the required information or perform the approved database recovery.
8. Validate the application.
9. Document the incident.
```

That's much stronger than:

> "Restore the backup."

---

# 48. Interview question

### "What is point-in-time restore?"

Answer:

> "It allows a database to be recovered to a specific point in time within the available recovery window, which is particularly useful for accidental data changes or logical errors."

---

# 49. Interview question

### "What is long-term retention?"

Answer:

> "LTR is used to retain backups for extended periods to satisfy business, regulatory, or compliance requirements beyond normal operational recovery needs."

---

# 50. Interview question

### "How do you validate backups?"

Don't say:

> "The backup job succeeded."

Instead:

```text
Backup exists
   ↓
Restore it
   ↓
Database opens
   ↓
Integrity/consistency checks
   ↓
Business validation
```

A successful backup operation doesn't prove the entire recovery process works.

---

# 51. Interview question

### "How do you calculate recovery readiness?"

Think:

```text
Recovery readiness =
Backup availability
+
Correct retention
+
Successful restore
+
Known RPO
+
Known RTO
+
Documented procedure
+
Regular testing
```

---

# 52. Real DBA exercise

Imagine you're on call.

At 11:00 AM:

```text
Production data corruption reported.
```

You immediately ask:

```text
When did it start?
What changed?
Which tables are affected?
Is corruption still occurring?
Can we isolate the application?
What is the last known good point?
What RPO/RTO applies?
```

Then:

```text
Restore to temporary database
       ↓
Compare
       ↓
Validate
       ↓
Choose recovery method
```

This is mature DBA behavior.

---

# 53. Recovery checklist

```text
INCIDENT
☐ Identify incident time
☐ Stop further damage
☐ Identify affected objects
☐ Identify business impact

RECOVERY
☐ Determine recovery point
☐ Confirm retention
☐ Select restore destination
☐ Initiate restore
☐ Monitor recovery

VALIDATION
☐ Database available
☐ Schema correct
☐ Row counts
☐ Critical data
☐ Permissions
☐ Application connection
☐ Business transaction tests

POST-RECOVERY
☐ Document incident
☐ Measure RTO
☐ Determine actual RPO
☐ Root-cause analysis
☐ Improve controls
```

---

# 54. The DBA golden rule

When a disaster occurs:

> **Don't confuse "database is online" with "data is correct."**

You need both.

```text
Database online
      +
Correct data
      +
Application works
      =
Successful recovery
```

---

# 55. Final mental model

Keep these four scenarios in your head:

```text
                    FAILURE
                       |
       +---------------+---------------+
       |               |               |
       ↓               ↓               ↓
 Infrastructure     Bad data        Region
       |               |               |
       ↓               ↓               ↓
      HA              PITR             DR
       |               |               |
       ↓               ↓               ↓
 Availability       Recovery       Business
```

And remember:

```text
HA ≠ Backup
Backup ≠ DR
DR ≠ Application resilience
```

You need all of them.

---

## Next — Lesson 92: Azure SQL Managed Instance Monitoring & Performance

We'll cover **CPU, memory, I/O, waits, Query Store, Azure Monitor, metrics, alerts, DMVs, blocking, deadlocks, long-running queries, storage performance, tempdb, troubleshooting methodology, baseline creation, and a complete production performance incident.**
