## Lesson 87 — Azure SQL Managed Instance: Backup, Restore & Point-in-Time Recovery

This is one of the most important areas for a SQL Server DBA because **HA and DR are not enough**.

If someone accidentally runs:

```sql
DELETE FROM Customers;
```

and commits it, HA doesn't magically save you.

You need **backup and recovery**.

---

# 1. First understand the 4 different protections

Think of production protection like this:

```text
                 SQL MI Protection
                       |
       +---------------+---------------+
       |               |               |
       ↓               ↓               ↓
      HA              DR            Backup
       |               |               |
Availability      Regional        Data recovery
failure           disaster        / mistakes
```

And backup gives you something HA doesn't:

```text
"What did the database look like at 11:41 AM?"
```

---

# 2. HA vs Backup

Suppose this happens:

```text
11:40 AM → Database healthy
11:42 AM → Developer runs DELETE
11:43 AM → DELETE commits
11:44 AM → HA continues normally
```

HA sees:

```text
Database = healthy
```

So HA has done its job.

But your data is wrong.

You need:

```text
Point-in-time recovery
```

to recover the database to a point before the accidental deletion.

---

# 3. What is a backup?

A backup is a recoverable copy of database data/state that can be used to restore the database.

At a high level:

```text
Production Database
        |
        ↓
     Backups
        |
        ↓
     Restore
        |
        ↓
Recovered Database
```

---

# 4. Types of SQL Server backups

If you're coming from traditional SQL Server DBA work, know these concepts:

### Full backup

Contains a full database backup at a particular point.

### Differential backup

Contains changes since the relevant full backup.

### Transaction log backup

Captures transaction-log records needed for point-in-time recovery in traditional full/bulk-logged recovery scenarios.

But here's an important Azure SQL Managed Instance distinction:

> **SQL Managed Instance is a managed service, so Azure manages the automated backup infrastructure for you.**

You don't operate backups exactly the same way you would on a self-managed SQL Server VM.

---

# 5. Automated backups

Azure SQL Managed Instance provides automated backups for database recovery.

The platform manages much of the backup infrastructure.

Conceptually:

```text
                 SQL MI
                   |
                   ↓
           Automated backups
                   |
        +----------+----------+
        |                     |
   Full backups          Log backups
        |                     |
        +----------+----------+
                   |
                   ↓
          Point-in-time restore
```

The exact backup retention and redundancy options depend on the SQL MI configuration and Azure capabilities available for your deployment.

---

# 6. Why automated backups are important

Imagine you have:

```text
500 databases
```

Manually managing:

```text
Full
Differential
Log
Copy
Retention
Cleanup
Verification
```

would be operationally expensive.

With Azure SQL MI, Microsoft manages the automated backup infrastructure.

Your DBA responsibility shifts toward:

```text
Recovery requirements
Retention configuration
Restore testing
DR design
Monitoring
Operational validation
```

---

# 7. Point-in-time restore — PITR

This is the most important concept in this lesson.

**Point-in-time restore** means recovering a database to a specific point in time within the available recovery window.

Example:

```text
11:00 AM
Database healthy

11:15 AM
Application deployment

11:42 AM
Accidental DELETE

11:50 AM
Problem discovered
```

You could potentially restore the database to:

```text
11:41 AM
```

instead of restoring it to the current damaged state.

---

# 8. Why not simply restore the latest backup?

Suppose the latest recoverable state is:

```text
11:50 AM
```

But the bad DELETE happened at:

```text
11:42 AM
```

Restoring to 11:50 may include the bad DELETE.

You need:

```text
Before bad transaction
```

Therefore:

```text
PITR
```

is the correct concept.

---

# 9. Real production scenario

Let's say:

```text
Database:
SalesDB
```

At:

```text
11:40 AM
```

everything is good.

Developer executes:

```sql
DELETE FROM CustomerOrders;
```

No WHERE clause.

Suppose the command affects:

```text
2,000,000 rows
```

It commits.

---

# 10. DBA gets alerted

At:

```text
11:50 AM
```

the DBA receives an alert.

First rule:

> **Don't panic and don't immediately overwrite the production database.**

First investigate.

---

# 11. Step 1 — Determine exactly what happened

Ask:

```text
When did the bad operation occur?
Which database?
Which table?
How many rows?
Was it committed?
What application/session executed it?
Are users still actively modifying the database?
```

Suppose:

```text
Database = SalesDB
Incident = 11:42 AM
Discovered = 11:50 AM
```

---

# 12. Step 2 — Stop further damage

If the bad application process is still running:

```text
Stop it.
```

If a deployment caused the problem:

```text
Pause deployment.
```

If another process is modifying the affected data:

```text
Control the workload.
```

The goal is:

```text
Prevent additional damage
```

---

# 13. Step 3 — Don't immediately restore over production

This is a major DBA principle.

Instead, initially restore to a **separate database** where practical.

For example:

```text
Production
SalesDB
    |
    ↓
Point-in-time restore
    |
    ↓
SalesDB_Recovery
```

Why?

Because now you can compare:

```text
Production
vs
Recovery copy
```

without making the incident worse.

---

# 14. Step 4 — Choose the recovery point

Suppose:

```text
Bad DELETE:
11:42:17 AM
```

You may choose a point immediately before the bad operation, such as:

```text
11:42:00 AM
```

The exact usable granularity depends on the recovery mechanism and available recovery points.

The principle is:

```text
Choose the latest safe point before corruption/error.
```

---

# 15. Why choose the latest safe point?

Suppose:

```text
11:00 → Good
11:15 → Good
11:30 → Good
11:41 → Good
11:42 → Bad DELETE
```

If you restore to:

```text
11:00
```

you lose everything between 11:00 and 11:42.

If you can recover to:

```text
11:41
```

you minimize unrelated data loss.

This is why PITR is so powerful.

---

# 16. The recovery workflow

Conceptually:

```text
             Production
                 |
                 ↓
         Identify bad time
                 |
                 ↓
        Select safe recovery
              point
                 |
                 ↓
        Point-in-time restore
                 |
                 ↓
          Recovery database
                 |
                 ↓
         Validate data
                 |
                 ↓
        Extract required data
                 |
                 ↓
         Correct production
```

---

# 17. Very important: restoring the entire database vs recovering rows

Suppose only one table was damaged:

```text
CustomerOrders
```

but 99 other tables continued receiving valid transactions.

If you restore the entire production database to 11:41 AM, you may lose legitimate transactions from:

```text
11:41 → 11:50
```

That could be unacceptable.

So a sophisticated DBA asks:

> **Do I need to restore the entire database, or can I use the restored copy to recover only the affected data?**

---

# 18. Safer recovery pattern

Create:

```text
SalesDB_Recovery
```

at the safe point.

Then compare:

```text
SalesDB
      vs
SalesDB_Recovery
```

Find the missing rows.

Then selectively recover them into production.

Conceptually:

```text
SalesDB_Recovery
       |
       ↓
Correct rows
       |
       ↓
Production SalesDB
```

This can minimize collateral data loss.

---

# 19. Example

Production currently has:

```text
OrderID  Customer  Status
1001     Alice     Shipped
1002     Bob       Paid
1003     John      Cancelled
```

The accidental DELETE removed:

```text
1001
1002
1003
```

But after the incident, new legitimate orders were created:

```text
1004
1005
1006
```

If you overwrite production with an older full database state, you might lose:

```text
1004
1005
1006
```

Instead:

```text
Restore historical copy
       ↓
Extract 1001–1003
       ↓
Insert/recover them
       ↓
Keep 1004–1006
```

This is often a better recovery strategy.

---

# 20. But be careful with identity/keys

Suppose you recover rows manually.

You need to consider:

```text
Primary keys
Foreign keys
Identity values
Unique constraints
Triggers
Computed columns
Audit columns
Dependencies
```

You can't simply copy data blindly.

---

# 21. Referential integrity

Suppose:

```text
Orders
   |
   ↓
OrderItems
```

If the DELETE removed:

```text
Orders
```

but `OrderItems` still exist—or vice versa—you may have consistency problems.

You need to understand the schema relationship before recovering data.

---

# 22. Another scenario — bad UPDATE

Suppose someone runs:

```sql
UPDATE Customers
SET CreditLimit = 0;
```

instead of:

```sql
UPDATE Customers
SET CreditLimit = 0
WHERE CustomerID = 123;
```

Now:

```text
10 million customers
```

are affected.

Again:

```text
HA won't save you.
```

PITR becomes extremely useful.

---

# 23. Another scenario — accidental DROP TABLE

Developer runs:

```sql
DROP TABLE CustomerOrders;
```

At:

```text
3:17 PM
```

Discovered:

```text
3:20 PM
```

Possible recovery strategy:

```text
Point-in-time restore
       ↓
Restore before 3:17
       ↓
Recovery database
       ↓
Validate table
       ↓
Recover required schema/data
```

---

# 24. Another scenario — bad deployment

Suppose application deployment changes:

```text
Stored procedure
Index
Schema
Data transformation
```

At:

```text
10:00 AM
```

Application is deployed.

At:

```text
10:15 AM
```

data starts becoming incorrect.

At:

```text
10:30 AM
```

DBA discovers it.

The database itself may be healthy.

But the application has introduced logical corruption.

PITR may help recover data to a known-good state.

---

# 25. Backup retention

Retention determines how far back you can recover using the available automated recovery mechanism.

Example conceptually:

```text
Today
 |
 +---- Recent recovery points
 |
 +---- Older recovery points
 |
 +---- Retention boundary
```

If a problem is discovered **after the available recovery window**, your normal PITR option may no longer be sufficient.

This is where longer-term retention/other backup strategies may matter.

---

# 26. Long-term retention

Some businesses have requirements such as:

```text
Keep data for:
7 years
```

for regulatory or business reasons.

That's different from:

```text
Operational recovery
```

Operational recovery might be:

```text
"Restore yesterday's database."
```

Long-term retention might be:

```text
"Keep historical backups for years."
```

These are different requirements.

---

# 27. Operational backup vs compliance retention

Think:

```text
Operational recovery
        |
        ↓
"What happened yesterday?"
```

versus:

```text
Long-term retention
        |
        ↓
"What records must we retain for years?"
```

Don't confuse them.

---

# 28. Geo-redundant backup

For regional resilience, backup data can be stored using supported geo-redundancy options.

Conceptually:

```text
Region A
   |
   ↓
Backup
   |
   ↓
Geo-redundant storage
   |
   ↓
Region B
```

This can help when the primary region becomes unavailable.

The exact redundancy option must be checked against the SQL MI configuration and region/service support.

---

# 29. Geo-restore

If your primary region is unavailable:

```text
Region A
   X
```

you may use an appropriate geo-recovery mechanism to restore into another region:

```text
Region B
   |
   ↓
Recovered SQL MI/database
```

This is useful for DR but generally has different RTO characteristics from a pre-provisioned synchronized secondary.

---

# 30. Backup and DR are connected

Think:

```text
BACKUP
  |
  +---- PITR
  |
  +---- Accidental deletion
  |
  +---- Corruption
  |
  +---- Regional recovery
```

So backup isn't just:

> "Something we configure because Microsoft recommends it."

It's part of the overall business continuity architecture.

---

# 31. Backup vs replication

This distinction is very important.

### Replication / failover

Keeps another copy closer to current state.

Good for:

```text
Availability
Business continuity
Fast failover
```

### Backup

Provides historical recovery points.

Good for:

```text
Accidental deletion
Logical corruption
Point-in-time recovery
Longer-term recovery
```

---

# 32. Why replication doesn't replace backup

Suppose:

```text
Primary
   |
   ↓
Bad DELETE
   |
   ↓
Secondary
```

If the bad operation is replicated:

```text
Secondary
   ↓
Also has bad DELETE
```

Therefore:

```text
Replication ≠ backup
```

---

# 33. Recovery time

PITR recovery time depends on multiple factors.

For example:

```text
Database size
Recovery mechanism
Target environment
Infrastructure availability
Data volume
Validation
Application startup
```

Therefore you should never assume:

> "Restore will take 5 minutes."

You should measure it.

---

# 34. Restore testing

A backup is not truly trustworthy until you can successfully recover from it.

Your DBA team should periodically test:

```text
Can we restore?
Can we connect?
Can applications connect?
Is data consistent?
Can users access required objects?
How long did it take?
```

---

# 35. The 3 questions every DBA should ask

For every production database:

### Question 1

> Can we recover it?

### Question 2

> How far back can we recover it?

### Question 3

> How long does recovery actually take?

If you can't answer these, your recovery strategy isn't mature.

---

# 36. RPO and backup

Suppose the business says:

```text
RPO = 5 minutes
```

You need to verify that your recovery architecture can actually provide a recovery point within that target.

Don't say:

> "We have backups, so RPO is 5 minutes."

Instead:

> "We have validated the configured backup/recovery mechanism and measured the achievable recovery point."

That's a much stronger DBA answer.

---

# 37. RTO and restore

Suppose:

```text
RTO = 30 minutes
```

You perform a restore test.

Result:

```text
Provisioning = 10 min
Restore = 35 min
Validation = 10 min
```

Total:

```text
55 minutes
```

Your architecture does **not** meet the RTO.

This is why testing matters.

---

# 38. Recovery validation

After restoring a database, don't simply say:

> "Restore completed successfully."

Check:

```text
Database accessible
Tables present
Expected row counts
Critical stored procedures
Users/permissions
Application connectivity
Data consistency
Recent transactions
```

---

# 39. Production recovery checklist

During a real incident:

```text
☐ Stop ongoing damage
☐ Determine incident time
☐ Identify affected objects
☐ Identify safe recovery point
☐ Check recovery options
☐ Restore to isolated environment
☐ Validate restored data
☐ Compare with production
☐ Determine minimal recovery set
☐ Recover required data
☐ Validate application
☐ Monitor
☐ Document incident
```

---

# 40. Why isolated restore is powerful

Suppose:

```text
Production = 5 TB
```

You restore a recovery copy.

Now you can investigate safely:

```text
Production
    |
    +----------------+
    |                |
    ↓                ↓
Current data    Recovery data
                    |
                    ↓
               Compare
```

This prevents you from making an irreversible recovery decision too early.

---

# 41. Recovery is not just a technical operation

You also need:

```text
Business owner
Application owner
DBA
Security
Operations
```

Why?

Because restoring data can affect business transactions.

Suppose finance says:

> "Do not lose today's invoices."

While the DBA says:

> "The safest restore point is 3 hours ago."

You need a business decision about acceptable data loss.

---

# 42. Example incident

Let's make it realistic.

### Timeline

```text
09:00
Normal workload

11:42
Bad DELETE

11:43
Transaction committed

11:45
Application continues processing

11:50
Incident discovered
```

There were legitimate transactions from:

```text
11:43 → 11:50
```

Therefore a full rollback to 11:41 may lose legitimate business data.

Better approach:

```text
Restore PITR copy to 11:41
        ↓
Identify deleted rows
        ↓
Recover only deleted rows
        ↓
Preserve legitimate 11:43–11:50 transactions
```

This is exactly the kind of reasoning interviewers want to hear.

---

# 43. Another important issue: cascading deletes

Suppose:

```text
Customer
   |
   ↓
Orders
   |
   ↓
OrderItems
```

A DELETE on `Customer` may trigger cascading deletion.

You might think:

```text
"Only one row was deleted."
```

But it could actually remove:

```text
1 Customer
50 Orders
400 OrderItems
```

Therefore you must understand:

```text
Foreign keys
Cascade behavior
Triggers
Stored procedures
```

before recovering.

---

# 44. Backup security

Backups contain sensitive data.

Therefore consider:

```text
Encryption
Access control
Identity
Retention
Auditing
Compliance
```

A backup should not become an easy way to bypass production security.

---

# 45. DBA responsibility

For Azure SQL MI, Microsoft handles much of the underlying managed-service infrastructure.

But **you still own the recovery strategy**.

Your responsibility includes:

```text
Business requirements
Retention
DR
Recovery testing
Restore procedures
Application coordination
Security
Monitoring
```

---

# 46. Interview question

### "What is point-in-time restore?"

Strong answer:

> "Point-in-time restore allows a database to be recovered to a specific point within the available recovery window, which is especially useful for accidental data modification, deletion, or logical corruption."

---

# 47. Interview question

### "A developer accidentally deletes 2 million rows. What do you do?"

A strong answer:

> "First I stop further damage and determine the exact incident time. I would identify the latest safe recovery point before the deletion, restore a separate copy of the database to that point, validate the missing data, and then recover only the affected data into production where possible. I would avoid blindly overwriting production because legitimate transactions may have occurred after the incident."

That is a **very strong production DBA answer**.

---

# 48. Interview question

### "Why restore to another database first?"

Answer:

> "It allows us to inspect and validate the recovered state without risking the current production database. We can compare the historical copy with production and determine the minimum amount of data that needs to be restored."

---

# 49. Interview question

### "Does HA replace backups?"

Answer:

> **"No. HA primarily addresses availability. Backups provide historical recovery and protection against logical errors such as accidental DELETE, UPDATE, DROP, or corruption."**

---

# 50. Interview question

### "Does replication replace backup?"

Answer:

> "No. Replication can maintain a current secondary copy, but a logical mistake can potentially propagate to the secondary. Historical backups are required for point-in-time recovery."

---

# 51. Interview question

### "How do you validate a backup strategy?"

Don't say:

> "The backup job says successful."

Instead:

```text
Backup success
      ↓
Restore test
      ↓
Database validation
      ↓
Application validation
      ↓
Measure RTO/RPO
```

---

# 52. The complete protection model

Memorize this:

```text
                         PRODUCTION
                             |
             +---------------+---------------+
             |               |               |
             ↓               ↓               ↓
             HA              DR            BACKUP
             |               |               |
       Availability       Region          Historical
         failure          disaster         recovery
             |               |               |
             ↓               ↓               ↓
        Failover         Secondary       PITR / Restore
                                             |
                                             ↓
                                    Logical recovery
```

---

# 53. The DBA golden rule

> **Never confuse "the database is available" with "the data is recoverable."**

Those are different requirements.

A production system can be:

```text
Available ✓
```

while:

```text
Data = wrong ❌
```

Backups and point-in-time recovery address that second problem.

---

# 54. Lesson 87 checklist

```text
BACKUP
☐ Automated backups
☐ Backup retention
☐ Backup redundancy
☐ Long-term retention concepts

RECOVERY
☐ Point-in-time restore
☐ Geo-restore
☐ Restore to separate database
☐ Data validation
☐ Selective data recovery

BUSINESS CONTINUITY
☐ RPO
☐ RTO
☐ HA
☐ DR
☐ Backup

INCIDENT RESPONSE
☐ Accidental DELETE
☐ Accidental UPDATE
☐ DROP TABLE
☐ Logical corruption
☐ Cascading changes

OPERATIONS
☐ Restore testing
☐ Recovery runbook
☐ Measure recovery time
☐ Validate application
```

---

## One scenario you should be able to explain in an interview

```text
11:42 AM
Developer accidentally deletes 2M rows
        ↓
11:50 AM
DBA discovers issue
        ↓
Stop further damage
        ↓
Identify exact incident time
        ↓
Find latest safe recovery point
        ↓
PITR → separate recovery database
        ↓
Validate missing rows
        ↓
Compare production vs recovery
        ↓
Recover only required data
        ↓
Validate application
        ↓
Monitor
        ↓
Document incident
```

That's the practical difference between **knowing Azure SQL MI** and **being able to operate it as a production DBA**.

---

### Next: Lesson 88 — Azure SQL Managed Instance Security

We'll cover **Microsoft Entra ID authentication, SQL authentication, logins/users, contained users, managed identities, Key Vault integration, TDE, Always Encrypted, Dynamic Data Masking, auditing, Defender for SQL, firewall/network security, private endpoints, least privilege, RBAC vs SQL permissions**, and then work through a real scenario where **a production application needs to access SQL MI without storing a SQL username/password anywhere**.
