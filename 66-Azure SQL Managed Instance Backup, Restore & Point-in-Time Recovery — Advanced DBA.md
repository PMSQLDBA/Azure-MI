# Lesson 98 — Azure SQL Managed Instance Backup, Restore & Point-in-Time Recovery

This lesson covers one of the most important DBA responsibilities:

> **When something goes wrong, how do you recover the database correctly, quickly, and with acceptable data loss?**

The key distinction is:

```text
Backup = copy of data for recovery
Restore = process of recovering from that backup
PITR = recovering to a specific point in time
RPO = how much data loss is acceptable
RTO = how quickly service must be restored
```

---

# 1. Why backups matter

A highly available database is **not automatically a backup**.

For example:

```text
Primary SQL MI
      |
      +---- High availability
      |
      +---- Automatic failover
```

This protects against certain infrastructure failures.

But if someone executes:

```sql
DELETE FROM Customers;
```

and commits it, high availability does not magically undo the transaction.

You need a recovery mechanism.

---

# 2. Backup vs high availability vs disaster recovery

Keep these separate.

### Backup

Protects against:

```text
Accidental deletion
Logical corruption
Some data-loss scenarios
Recovery to an earlier point
```

### High availability

Protects primarily against:

```text
Infrastructure/component failure
```

### Disaster recovery

Protects against:

```text
Larger-scale regional/service disruption
```

Think:

```text
Backup
   ≠
HA
   ≠
DR
```

A mature architecture may use all three.

---

# 3. Azure SQL Managed Instance backups

Azure SQL Managed Instance provides managed automated backups.

This means you don't normally operate SQL Server backup infrastructure the same way you would on a self-managed SQL Server VM.

But the DBA still needs to understand:

```text
Retention
Recovery window
Restore behavior
Recovery objectives
Backup validation
Business requirements
```

---

# 4. Recovery objectives

Two terms are essential.

## RPO — Recovery Point Objective

> **How much data can the business afford to lose?**

Example:

```text
RPO = 15 minutes
```

means the business accepts up to approximately 15 minutes of data loss in the defined recovery scenario.

---

# 5. RTO — Recovery Time Objective

> **How long can the service be unavailable?**

Example:

```text
RTO = 1 hour
```

means the recovery process should restore service within approximately one hour.

---

# 6. RPO vs RTO

Memorize:

```text
RPO → How much data can we lose?

RTO → How long can we be down?
```

Example:

```text
RPO = 5 minutes
RTO = 30 minutes
```

This means:

```text
Maximum acceptable data loss ≈ 5 minutes
Maximum acceptable recovery time ≈ 30 minutes
```

---

# 7. Business requirements come first

Don't start by asking:

> "What backup setting should I use?"

Start with:

```text
What data loss is acceptable?
How quickly must the system return?
How long must data be retained?
What regulations apply?
```

Then design the recovery strategy.

---

# 8. Point-in-time restore

PITR means:

> **Restore a database to a specific point within the available recovery window.**

Example:

```text
14:00
14:10
14:20
14:30
14:40
14:50
15:00
```

Suppose an accidental deletion happened at:

```text
14:37
```

You may want to recover to:

```text
14:36:59
```

or another suitable point before the destructive operation.

---

# 9. Important PITR concept

You generally don't "rewind" the existing production database like a video.

Instead, point-in-time recovery typically creates a restored database from the available backup history.

Conceptually:

```text
Production DB
     |
     | backup history
     ↓
Restore operation
     |
     ↓
New restored DB
```

---

# 10. Why restore to a new database?

Suppose production is:

```text
SalesDB
```

and someone accidentally deleted important rows.

You can restore a copy:

```text
SalesDB
SalesDB-Recovered
```

Then investigate the recovered data.

This can be much safer than immediately replacing production.

---

# 11. Recovery workflow

A typical logical recovery process:

```text
Incident
   ↓
Stop further damage
   ↓
Identify target recovery point
   ↓
Restore
   ↓
Validate recovered database
   ↓
Determine recovery method
   ↓
Reconcile data if needed
   ↓
Return application to service
```

---

# 12. First rule during accidental deletion

Suppose someone says:

> "We accidentally deleted 500,000 customers."

Don't immediately start restoring.

First:

```text
Stop the destructive process
Preserve evidence
Identify exact incident time
Determine affected objects
Determine whether the transaction committed
```

---

# 13. Why timing matters

Suppose:

```text
14:00 — Normal
14:30 — Bad deployment
14:35 — DELETE executed
14:40 — Incident detected
```

You need to know:

```text
When did the bad change occur?
```

A restore target of:

```text
14:34
```

might be appropriate.

A restore target of:

```text
14:29
```

may unnecessarily lose valid transactions.

---

# 14. Recovery point selection

Don't automatically choose:

> "The latest possible restore point."

Instead choose:

> **The latest known-good point before the destructive event.**

That minimizes unnecessary data loss.

---

# 15. Example

Timeline:

```text
10:00  Normal
10:15  Normal
10:30  Bad deployment
10:35  Bad DELETE
10:40  Incident detected
```

Potential recovery target:

```text
10:34:59
```

Then:

```text
Recover database
      ↓
Validate deleted records exist
      ↓
Identify valid transactions after 10:35
      ↓
Reconcile if required
```

---

# 16. Recovery is not always "restore everything"

Suppose only one table was damaged.

You might restore a copy of the database and extract the required data.

Conceptually:

```text
Production
   |
   +---- Restore copy
             |
             +---- Extract missing rows
             |
             +---- Validate
             |
             +---- Reinsert/reconcile
```

This can avoid losing unrelated transactions.

---

# 17. Logical corruption

Backup recovery is especially valuable for logical corruption.

Examples:

```text
Accidental DELETE
Incorrect UPDATE
Bad migration
Application bug
Wrong batch job
```

The infrastructure may be completely healthy.

The **data** is wrong.

---

# 18. Physical/infrastructure failure

A different scenario:

```text
Database infrastructure problem
```

High availability and managed service capabilities may provide automated recovery/failover mechanisms.

The DBA's job is to understand:

```text
What protection applies?
How quickly does it recover?
Does the application reconnect?
Is manual action required?
```

---

# 19. Backup retention

Retention answers:

> **How far back can I recover using the available backup history?**

Business requirements may say:

```text
Recover last 7 days
Recover last 35 days
Retain historical data for years
```

These are different requirements.

---

# 20. Short-term recovery vs long-term retention

Think:

```text
Operational recovery
        |
        +-- Recent accidental deletion
        +-- Recent corruption
        |
        ↓
Point-in-time recovery

Long-term retention
        |
        +-- Compliance
        +-- Historical requirements
        |
        ↓
Long-term backup retention strategy
```

Don't confuse the two.

---

# 21. Long-term retention

Some organizations require backups retained for much longer than the normal operational recovery period.

Examples:

```text
Financial records
Regulatory requirements
Legal requirements
Audit requirements
```

The exact retention period should come from business/compliance requirements.

---

# 22. Backup security

Backups contain sensitive database information.

Therefore consider:

```text
Encryption
Access control
Retention
Auditing
Administrative permissions
Compliance
```

A backup isn't "safe" merely because it isn't the live database.

---

# 23. Restore testing

One of the most important DBA rules:

> **A backup you have never successfully restored is an assumption, not proven recoverability.**

You should periodically test restores.

---

# 24. Restore test workflow

```text
Select backup
    ↓
Restore
    ↓
Database integrity validation
    ↓
Application validation
    ↓
Performance validation
    ↓
Document results
```

---

# 25. What should you validate?

After restoring:

```text
Database accessible
Tables present
Expected data present
Users/permissions appropriate
Critical queries work
Application can connect
Stored procedures work
Indexes present
Jobs/dependencies understood
```

---

# 26. Recovery drill

A mature organization periodically performs:

```text
Disaster recovery exercise
```

For example:

```text
Simulated database loss
       ↓
Execute runbook
       ↓
Restore
       ↓
Connect application
       ↓
Validate business transactions
       ↓
Measure RTO/RPO
```

---

# 27. Why drills matter

A recovery runbook may say:

```text
Step 1: Restore database
Step 2: Validate application
```

But during a real incident you may discover:

```text
Missing permissions
Unknown connection string
Broken DNS
Unclear application owner
Unexpected restore duration
Missing credentials
```

Testing exposes these problems before the real disaster.

---

# 28. Recovery runbook

A good runbook contains:

```text
Incident criteria
Recovery owner
Escalation contacts
Target recovery point
Restore procedure
Validation steps
Application steps
Communication plan
Rollback/recovery alternatives
Success criteria
```

---

# 29. Recovery communications

During a serious incident, stakeholders need concise information.

Example:

```text
Incident:
Customer database accidentally modified.

Current state:
Production access restricted.

Recovery:
Restoring to the last known-good point.

Target:
Restore to 14:34 UTC.

Validation:
DBA + application team.

Next update:
After restore validation.
```

Good communication reduces confusion.

---

# 30. Scenario — accidental DELETE

Suppose:

```sql
DELETE FROM Orders;
```

was accidentally executed without a proper filter.

Millions of rows are affected.

### Step 1

Stop the process.

```text
Application/job/session
```

if still running.

### Step 2

Record:

```text
Time
Database
Table
Transaction
User
Application
```

### Step 3

Determine whether the transaction committed.

### Step 4

Identify the last known-good point.

### Step 5

Restore a copy.

### Step 6

Validate.

### Step 7

Determine how to reconcile production.

---

# 31. Why restoring over production immediately can be dangerous

Suppose valid transactions happened after the accidental DELETE:

```text
14:35 — Bad DELETE
14:36 — Legitimate order
14:37 — Legitimate payment
14:38 — Legitimate invoice
```

If you restore production to:

```text
14:34
```

you also lose the legitimate transactions.

Therefore:

> **Recovery must consider both damaged data and valid post-incident transactions.**

---

# 32. Better recovery pattern

Often:

```text
Production
    |
    | continues carefully
    |
Restore copy
    |
    +-- Recover missing data
    |
    +-- Compare with production
    |
    ↓
Reconcile
```

The exact strategy depends on the incident.

---

# 33. Scenario — bad UPDATE

Suppose:

```sql
UPDATE Customers
SET CreditLimit = 0;
```

was accidentally committed.

Potential strategy:

```text
Restore copy to point before UPDATE
       ↓
Identify correct values
       ↓
Compare with production
       ↓
Generate targeted correction
       ↓
Validate
```

This can be safer than replacing the whole database.

---

# 34. Scenario — bad schema deployment

Suppose a deployment:

```text
Drops a column
Changes data types
Changes stored procedures
```

and causes application failure.

Recovery may involve:

```text
Application rollback
+
Database rollback/recovery
```

The application and database must be considered together.

---

# 35. Database rollback vs application rollback

A deployment can produce:

```text
Application version 2
Database schema version 2
```

If the application is rolled back to version 1, it may not work with schema version 2.

Therefore deployments should consider:

```text
Backward compatibility
Forward compatibility
Rollback strategy
```

---

# 36. Zero-downtime deployment thinking

A safer schema deployment pattern is often:

```text
Old application
      ↓
Backward-compatible schema change
      ↓
New application
      ↓
Data migration
      ↓
Remove obsolete schema later
```

Avoid coupling an irreversible schema change tightly to one application release when possible.

---

# 37. Recovery performance

Restoring a database isn't instantaneous.

Recovery time can depend on:

```text
Database size
Restore mechanism
Target configuration
Network
Validation work
Application dependencies
```

This is why RTO must be **tested**, not guessed.

---

# 38. RTO testing

Suppose management says:

```text
RTO = 30 minutes
```

You perform a recovery drill.

Actual result:

```text
Restore + validation = 55 minutes
```

You have discovered:

> **The architecture does not currently meet the business requirement.**

That is valuable information.

---

# 39. RPO testing

Suppose business requirement:

```text
RPO = 15 minutes
```

During a recovery exercise, you discover:

```text
Potential data loss = 40 minutes
```

Again:

> The recovery design doesn't meet the stated requirement.

You need to adjust the architecture, process, or business expectation.

---

# 40. Recovery decision tree

```text
             DATA PROBLEM
                  |
          Is service available?
             /          \
           YES           NO
            |             |
     Logical damage     HA/DR path
            |
     Identify damage
            |
     Need historical data?
          /       \
        YES        NO
         |          |
       PITR      Targeted fix
         |
     Restore copy
         |
      Validate
         |
      Reconcile
```

---

# 41. Backup monitoring

Don't just configure backups and forget them.

Monitor:

```text
Backup health
Retention configuration
Restore availability
Recovery history
Failures/errors where applicable
```

---

# 42. Backup validation

Good validation includes:

```text
Can we restore?
Can the database open?
Can users connect?
Can critical queries execute?
Can the application work?
Can we meet RTO?
```

---

# 43. Backup documentation

Document:

```text
What is protected?
Retention period
Recovery window
Restore procedure
RPO
RTO
Owners
Testing frequency
Last successful restore test
```

---

# 44. Common DBA mistake

> "Azure takes automatic backups, so we're fully protected."

That's incomplete.

You still need to know:

```text
What can be recovered?
How far back?
How quickly?
How do we restore?
Who performs it?
Has it been tested?
```

---

# 45. Common DBA mistake #2

> "We have a replica, so we don't need backups."

Incorrect.

A replica/HA mechanism can replicate logical mistakes too.

If the application performs:

```sql
DELETE FROM Customers;
```

the deletion can propagate through the HA architecture.

Backups provide a different recovery capability.

---

# 46. Common DBA mistake #3

> "Restore succeeded, so we're done."

Not necessarily.

You still need:

```text
Database validation
Data validation
Application validation
Business validation
```

---

# 47. Production recovery checklist

```text
INCIDENT
☐ Identify incident
☐ Stop further damage
☐ Record timeline
☐ Identify affected database
☐ Determine scope

RECOVERY
☐ Determine RPO requirement
☐ Select recovery point
☐ Restore
☐ Validate restored database
☐ Compare data
☐ Determine reconciliation strategy

APPLICATION
☐ Validate connectivity
☐ Validate critical transactions
☐ Validate integrations
☐ Validate jobs

CLOSURE
☐ Confirm business recovery
☐ Document data loss, if any
☐ Measure RTO
☐ Measure RPO
☐ Root-cause analysis
☐ Prevent recurrence
```

---

# 48. Interview question

### "What is the difference between RPO and RTO?"

Strong answer:

> **RPO defines the maximum acceptable amount of data loss, while RTO defines the maximum acceptable time to restore service.**

Example:

```text
RPO = 15 minutes
RTO = 1 hour
```

---

# 49. Interview question

### "What is point-in-time restore?"

> "Point-in-time restore allows a database to be recovered to a selected time within the available recovery window, which is especially useful for recovering from accidental data changes or logical corruption."

---

# 50. Interview question

### "Why restore to a new database?"

> "Restoring to a separate database allows us to inspect and validate recovered data without immediately overwriting the production database. We can then selectively reconcile the required data or determine whether a broader recovery is necessary."

---

# 51. Interview question

### "Are backups enough for disaster recovery?"

Strong answer:

> "No. Backups provide a recovery mechanism, but disaster recovery also requires an appropriate architecture, recovery procedures, application dependencies, networking, identity, monitoring, and tested RPO/RTO objectives."

---

# 52. Interview question

### "How do you know backups are usable?"

> "I periodically perform restore tests and validate both the database and application. I also measure the actual restore time against the required RTO."

---

# 53. Interview question

### "Someone accidentally deleted data. What do you do?"

A strong production answer:

```text
1. Stop further destructive activity.
2. Determine whether the transaction committed.
3. Record the exact incident timeline.
4. Identify affected data.
5. Determine the latest known-good recovery point.
6. Restore a separate copy.
7. Validate the recovered data.
8. Compare it with production.
9. Reconcile only the required data where possible.
10. Validate the application and business process.
11. Document the incident and prevent recurrence.
```

---

# 54. The DBA recovery model

Memorize this:

```text
                 DATA INCIDENT
                       |
                  STOP DAMAGE
                       |
                 IDENTIFY TIME
                       |
                DEFINE RPO/RTO
                       |
                CHOOSE RECOVERY
                  /          \
               PITR          HA/DR
                 |
            RESTORE COPY
                 |
              VALIDATE
                 |
             RECONCILE
                 |
          APPLICATION TEST
                 |
          BUSINESS VALIDATION
                 |
              RESUME
                 |
             POSTMORTEM
```

---

# 55. Final takeaway

The most important lesson is:

> **Backup configuration is not the same as recoverability.**

A production DBA should be able to answer all five:

```text
1. What are we protecting?
2. How far back can we recover?
3. How much data can we lose?
4. How quickly can we recover?
5. Have we actually tested it?
```

If you can answer those confidently, you're thinking like a production DBA.

---

## Next — Lesson 99: Azure SQL Managed Instance Disaster Recovery & Business Continuity

We'll connect **backup + HA + failover + geo-disaster recovery** into one architecture, including **RPO/RTO design, failover groups/managed-instance link concepts, regional outages, planned vs forced failover, application connection handling, DNS/network dependencies, DR runbooks, failback, and a full regional-outage scenario.**
