# Lesson 70 — Backup, Restore & Point-in-Time Recovery in Azure SQL Managed Instance

This is one of the most important DBA topics.

If you understand only one principle from this lesson, remember:

> **A backup strategy is not about taking backups. It is about being able to recover the business data within the required RPO and RTO.**

Let's use a realistic production incident.

---

# 1. The 2:17 PM incident

Your company runs an order-processing system.

At:

```text
2:17 PM
```

a developer accidentally executes:

```sql
DELETE FROM Orders;
```

without the intended `WHERE` clause.

Millions of rows are affected.

Application:

```text
Orders
   ↓
Empty / damaged
```

Management immediately asks:

> "Can you restore the database to 2:15 PM?"

Your answer cannot simply be:

> "Yes, we have backups."

You need to know:

```text
What recovery options exist?
What is the recovery point?
How much downtime?
How much data loss?
Can we preserve the current state?
How do we validate the recovery?
```

---

# 2. Backup vs restore

These are different operations.

### Backup

Creates a recoverable copy/state of data.

```text id="e5q7r2"
Database
   ↓
Backup
   ↓
Recovery data
```

### Restore

Uses recovery data to recreate database state.

```text id="m8n3v6"
Backup
   ↓
Restore
   ↓
Database
```

A backup that has never been successfully restored is a risk.

---

# 3. Three important recovery concepts

For Azure SQL Managed Instance, understand:

```text id="p2x9k4"
Point-in-Time Restore
Long-term retention
Geo-disaster recovery
```

They solve different problems.

---

# 4. Point-in-Time Restore — PITR

PITR means:

> Recover the database to a specific point in time within the available recovery window.

Example:

```text id="r7m1c8"
2:00 PM ───── 2:15 PM ───── 2:17 PM
                           ↑
                     Accidental DELETE
```

You want:

```text id="q4n8v2"
2:15 PM
```

Conceptually:

```text id="x6p3m9"
Backup/recovery chain
       ↓
Select target time
       ↓
Restore
       ↓
Database state at target time
```

---

# 5. PITR is perfect for accidental data modification

This is the classic use case.

```text id="w3k8n1"
Developer mistake
      ↓
Data deleted
      ↓
Database still available
      ↓
PITR
```

But there is an important operational question:

> **Do you overwrite the production database, or restore to a new database first?**

Usually, you should think carefully before destroying your current evidence/state.

---

# 6. Safer recovery pattern

Suppose:

```text id="a8q2m6"
ProductionDB
```

is damaged.

Instead of immediately replacing it, you can consider restoring to:

```text id="c5n9r3"
ProductionDB_Recovery
```

Then:

```text id="v7m1x4"
Compare
   ↓
Validate
   ↓
Extract missing data
   ↓
Repair production
```

This can be much safer depending on the incident.

---

# 7. Why not immediately restore over production?

Because you may lose:

```text id="j4p8q2"
Valid transactions after the target time
```

Example:

```text id="s6m3n9"
2:15 PM → good
2:17 PM → DELETE
2:18 PM → legitimate customer order
2:19 PM → legitimate payment
```

If you restore production directly to 2:15 PM:

```text id="k1x7r5"
2:18 transaction
2:19 transaction
```

may also disappear from the restored database.

That can create another data-loss problem.

---

# 8. Better question

Instead of:

> "Restore the database."

Ask:

> **"What exact data needs recovery, and how can we minimize loss of legitimate transactions?"**

This is where DBA judgment matters.

---

# 9. Recovery options

For the accidental DELETE scenario:

### Option A

Restore entire database to a new database.

```text id="m8q3v7"
ProductionDB
     |
     +---- PITR
             ↓
      ProductionDB_Recovery
```

Then recover only the affected data.

### Option B

If the incident is severe and continuing, potentially perform a controlled application/database recovery according to the organization's DR procedure.

The correct choice depends on business impact and recovery architecture.

---

# 10. RPO connection

Suppose your recovery design allows:

```text id="x2n9k5"
RPO = 5 minutes
```

The business understands that, under a disaster scenario, approximately five minutes of data could potentially be lost depending on the recovery mechanism.

But for PITR, you may be able to target a much more precise point in the supported recovery window.

That's why:

```text id="v4m7q1"
PITR
```

and:

```text id="c8p2x6"
DR replication
```

shouldn't be confused.

---

# 11. RTO connection

Now management says:

> "We need the application back within 15 minutes."

That's:

```text id="q9m3w8"
RTO = 15 minutes
```

If your recovery procedure takes:

```text id="h6x1p4"
90 minutes
```

then your design does not meet the requirement.

This is why restore procedures must be **tested**, not just documented.

---

# 12. Backup retention

Recovery isn't useful if the target date is outside your available retention.

Imagine:

```text id="r5n8k2"
Incident:
August 10
```

But the required recovery point is:

```text id="s3m7v1"
May 15
```

If your retention doesn't cover May 15:

```text id="z8q2x6"
PITR cannot magically recover it.
```

That's where longer-term retention requirements become important.

---

# 13. Short-term vs long-term recovery

Think:

```text id="j4n9m3"
Recent accidental mistake
        ↓
PITR / short-term recovery
```

versus:

```text id="p7x2q8"
Older historical recovery requirement
        ↓
Long-term retention / archival recovery design
```

The retention period should come from business, legal, compliance, and operational requirements—not simply:

> "30 days sounds good."

---

# 14. Real example

Suppose your business requires:

```text id="w6c1r4"
Operational recovery:
35 days

Compliance:
7 years
```

You shouldn't assume your normal operational PITR mechanism alone satisfies the seven-year requirement.

You need a separate retention/archive strategy appropriate to the requirement.

---

# 15. Restore validation

This is one of the most overlooked DBA tasks.

Suppose restore completes:

```text id="n2m8q5"
Restore = SUCCESS
```

Does that mean you're finished?

**No.**

You need to validate:

```text id="c7r3x9"
Database accessible?
Tables present?
Expected row counts?
Critical procedures present?
Users/permissions correct?
Application connectivity?
Data consistency?
Business transactions?
```

---

# 16. Technical restore success ≠ business recovery success

Imagine:

```text id="g8p4m1"
Restore completed
```

But:

```text id="v5q9x2"
Application login fails
```

or:

```text id="r1n7c6"
Critical table missing
```

or:

```text id="k3m8w4"
Application points to old database
```

Then the business isn't recovered.

---

# 17. Restore validation checklist

After recovery:

```text id="s9x2m5"
☐ Database online
☐ Expected database objects present
☐ Critical tables accessible
☐ Row counts checked
☐ Recent critical transactions checked
☐ Security principals/permissions validated
☐ Application connection tested
☐ Smoke tests passed
☐ Business owner validates functionality
```

---

# 18. Recovery scenario — step by step

Let's return to:

```text
2:17 PM
DELETE FROM Orders
```

### Step 1 — Stop further damage

Determine whether the application is still writing to the damaged data.

```text id="f7m3q1"
Application
     ↓
Orders
     ↓
Still processing?
```

If necessary, follow your incident process to pause affected functionality.

---

### Step 2 — Preserve evidence

Don't immediately start destroying/replacing databases.

Capture:

```text id="p8r2n6"
Time of incident
SQL statement
Affected database
Affected tables
Application/user
Approximate row impact
```

---

### Step 3 — Determine target recovery time

Management says:

```text id="w4q9x7"
Restore to 2:15 PM
```

Now you have a recovery target.

---

### Step 4 — Determine recovery method

PITR is considered because:

```text id="m1n8c3"
Target = specific time
```

---

### Step 5 — Restore safely

Potentially:

```text id="v6x2p9"
ProductionDB
      |
      | PITR
      v
ProductionDB_Recovery
```

---

### Step 6 — Validate

Check:

```text id="j5k8m2"
Orders
Customers
Payments
Inventory
```

---

### Step 7 — Recover only what is needed

If the damage is isolated to:

```text id="q3r7n1"
Orders
```

you may be able to extract the correct records from the recovered database rather than replacing the entire production database.

This is often operationally safer.

---

# 19. A subtle issue: transactions

Suppose the developer's mistake occurred inside a transaction.

You need to understand whether:

```text id="x8m4c6"
Transaction committed?
Transaction rolled back?
Partially processed?
```

This can significantly change recovery strategy.

Never assume:

> "The DELETE statement ran, therefore exactly X rows are permanently gone."

Investigate the actual transaction state.

---

# 20. Another scenario — corrupted application data

Imagine a bug runs:

```text id="r2n7v5"
UPDATE Customer
SET CreditLimit = 0;
```

for thousands of customers.

The database is online.

Queries work.

CPU is normal.

The application is technically healthy.

But the **data is wrong**.

This is still a recovery incident.

The database doesn't have to be "down" for backup/restore to matter.

---

# 21. Backup monitoring

Don't only monitor:

```text id="k6m3q9"
Database available
```

Also monitor:

```text id="p1x8r4"
Recovery capability
```

Ask:

```text id="v7n2c5"
Can we recover?
How far back?
When was recovery last tested?
Are backups completing?
Is retention sufficient?
```

---

# 22. Restore testing

A mature organization periodically performs restore tests.

Example:

```text id="w9m4q2"
Production backup
      ↓
Restore to test environment
      ↓
Validate
      ↓
Measure duration
      ↓
Document result
```

Record:

```text id="c8x3n7"
Restore start
Restore completion
Database size
Validation result
Issues
```

This gives you real RTO evidence.

---

# 23. Why restore time matters

Suppose database size:

```text id="m5r8q1"
2 TB
```

You tell management:

> "RTO is 30 minutes."

But your actual tested restore takes:

```text id="z4n7p2"
2 hours
```

Then your RTO isn't 30 minutes.

Your documentation is simply wrong.

---

# 24. Geo-disaster recovery

Now change the scenario.

The problem isn't accidental deletion.

The entire primary Azure region becomes unavailable.

```text id="h3q9x6"
Region A
   X
Major outage
```

Now you need:

```text id="s7m2p4"
DR architecture
```

Potentially involving another region and supported SQL MI disaster-recovery capabilities.

This is different from PITR.

---

# 25. PITR vs DR

| Situation                            | Typical recovery concept |
| ------------------------------------ | ------------------------ |
| Accidental DELETE                    | PITR / targeted recovery |
| Corrupted data                       | PITR / backup recovery   |
| Need older historical recovery       | Long-term retention      |
| Regional outage                      | DR/failover              |
| Need exact historical database state | Point-in-time recovery   |

Remember:

```text id="q2n8r5"
PITR → "Give me the database as it was at time X."
DR → "Give me service continuity after disaster."
```

---

# 26. The dangerous misconception

Someone says:

> "Our secondary region is replicated, so we're protected from accidental DELETE."

Not necessarily.

If the bad transaction is replicated:

```text id="v8m3x1"
Primary
DELETE
   ↓
Replication
   ↓
Secondary
DELETE
```

Both environments can contain the logical mistake.

Therefore:

```text id="p4q9k7"
DR ≠ protection from every logical error
```

---

# 27. Backup + DR architecture

A mature design looks more like:

```text id="n6x2r8"
                    Production
                        |
                  SQL Managed Instance
                        |
          +-------------+-------------+
          |                           |
       Recovery                    DR
       capability                capability
          |                           |
        PITR                    Secondary region
          |                           |
          +-------------+-------------+
                        |
                     Testing
                        |
                    Validation
```

You need both.

---

# 28. Recovery decision tree

When an incident happens:

```text id="j7p3m9"
What happened?
      |
      +-- Logical data mistake?
      |       |
      |       +-- PITR / targeted recovery
      |
      +-- Regional disaster?
      |       |
      |       +-- DR/failover
      |
      +-- Older historical requirement?
      |       |
      |       +-- Long-term retention
      |
      +-- Unknown?
              |
              +-- Investigate before acting
```

---

# 29. Senior DBA rule

When someone says:

> "Restore it."

Your first questions should be:

```text id="x5m8q2"
What database?
What exact target time?
Why?
What data is affected?
Can production continue?
What transactions occurred afterward?
What is the RTO?
What is the safest recovery target?
```

This prevents a DBA from turning one incident into a bigger one.

---

# 30. Interview questions

### Q: What is Point-in-Time Restore?

> It allows recovery of a database to a specific point within the available recovery period, making it useful for accidental data modifications and similar logical-recovery scenarios.

### Q: Why restore to another database first?

> It can preserve the current production state while allowing validation and selective recovery. Replacing production immediately can unnecessarily discard legitimate transactions that occurred after the desired recovery point.

### Q: Does DR replace backups?

> No. DR primarily addresses availability and disaster scenarios. Backups and point-in-time recovery are essential for logical data recovery such as accidental deletes or corrupt changes.

### Q: How do you prove an RTO?

> By performing controlled recovery tests and measuring the actual time required to restore and validate the workload.

### Q: What should you validate after a restore?

> Database availability, objects, critical data, security, application connectivity, and business functionality—not just whether the restore command completed successfully.

---

# 31. The recovery mental model

Memorize this:

```text id="r3k9m1"
                RECOVERY
                    |
       +------------+------------+
       |            |            |
      PITR         DR        Long-term
       |            |          retention
       |            |            |
  Logical error   Disaster    Historical
       |            |          recovery
       |            |            |
       +------------+------------+
                    |
                 TESTING
                    |
                 VALIDATION
                    |
                  BUSINESS
                 RECOVERY
```

And the most important principle:

> **A successful restore is not the same as a successful recovery.**

The real success criterion is:

```text
Business can safely continue
+
Required data is recovered
+
RPO/RTO requirements are met
```

---

# Next — Lesson 71

Next we'll go into **Azure SQL Managed Instance performance troubleshooting in real production scenarios**.

We'll work through actual DBA incidents such as:

1. **CPU suddenly reaches 100%**
2. **A query that took 200 ms now takes 30 seconds**
3. **Blocking chain causes application timeout**
4. **Deadlocks suddenly increase**
5. **IO latency spikes**
6. **Memory/resource pressure**
7. **Parameter-sensitive query problems**
8. **Execution-plan regression**
9. **Statistics problems**
10. **A production query is slow only during business hours**

We'll build a systematic **"SQL is slow" troubleshooting framework** that you can use in interviews and real Azure SQL MI production support.
