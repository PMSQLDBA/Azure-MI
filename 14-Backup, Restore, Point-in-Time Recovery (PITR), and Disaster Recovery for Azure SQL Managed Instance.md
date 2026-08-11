**Lesson 12 — Backup, Restore, Point-in-Time Recovery (PITR), and Disaster Recovery for Azure SQL Managed Instance**.

This is one of the most important areas for a DBA because **HA keeps services available, but backups let you recover data**.

---

# Lesson 12 — Backup & Restore in Azure SQL Managed Instance

Start with this mental model:

```text
                 DATA PROTECTION
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
       HA            Backup           DR
        |              |              |
   Service         Recover          Region /
   availability    deleted data    major outage
```

Don't mix these three.

---

# 1. HA is NOT backup

Imagine:

```text
Primary
   |
   | Replication
   v
Secondary
```

A DBA accidentally runs:

```sql
DELETE FROM Customers;
```

and commits it.

The change can propagate through the HA architecture.

So:

```text
Primary
  |
  | DELETE
  v
Secondary
```

Both may contain the unwanted change.

Therefore:

> **HA doesn't protect you from logical mistakes.**

That's where backup/PITR comes in.

---

# 2. What does backup protect against?

Examples:

```text
Accidental DELETE
Accidental UPDATE
Dropped table
Dropped database
Application bug
Data corruption
Operational mistakes
```

For example:

```text
10:00 AM
Customers = 10 million rows

10:15 AM
Developer runs bad DELETE

10:16 AM
Business discovers problem
```

You may need to recover the database to a point before the mistake.

---

# 3. Point-in-Time Restore

This is called:

# PITR

Point-in-Time Restore means:

> Restore a database to a specific point in time within the available recovery window.

Example:

```text
10:00      10:05      10:10      10:15      10:20
 |           |          |           |          |
 |           |          |        BAD DELETE    |
 |           |          |           |          |
 +-----------+----------+-----------+
                         ^
                    Restore here
```

You don't necessarily restore only to the latest backup.

You can restore to an appropriate point before the incident.

---

# 4. Real production incident

ABC Retail has:

```text
CustomerDB
```

At:

```text
2:00 PM
```

Everything is fine.

At:

```text
2:17 PM
```

A deployment contains a bug:

```sql
UPDATE Customers
SET Email = NULL;
```

Unfortunately, it updates millions of rows.

At:

```text
2:20 PM
```

Business discovers the problem.

You don't want:

```text
Restore from yesterday
```

because you'd lose an entire day's legitimate transactions.

Instead:

```text
Restore CustomerDB
to a point shortly before
2:17 PM
```

That's the power of PITR.

---

# 5. Important question: Do you restore over the existing database?

Usually, you should **not immediately overwrite the production database**.

A safer recovery process is:

```text
Production CustomerDB
          |
          v
PITR restore
          |
          v
CustomerDB_Restore
```

Then compare:

```text
Production
    vs
Restored copy
```

You can determine:

```text
What data was lost?
What transactions occurred after the recovery point?
What needs to be merged?
```

This is often much safer.

---

# 6. Why restore to a separate database?

Suppose:

```text
2:00 PM → good
2:17 PM → bad update
2:20 PM → discovered
```

Restore:

```text
CustomerDB_Restore
```

to:

```text
2:16 PM
```

Now:

```text
Production
   |
   +---- Bad data
   |
Restore
   |
   +---- Good data
```

You can compare them.

---

# 7. Selective recovery

Imagine only 5,000 customers were incorrectly modified.

You may not need to replace the entire production database.

You could:

```text
PITR restore
     ↓
Find affected rows
     ↓
Validate
     ↓
Generate correction script
     ↓
Apply carefully
```

This can minimize business disruption.

---

# 8. But there's a catch

Suppose between:

```text
2:00 PM
```

and:

```text
2:17 PM
```

the application processed:

```text
10,000 legitimate orders
```

If you restore the entire production database to 2:00 PM, those legitimate transactions disappear from the restored copy.

Therefore:

> **Recovery isn't simply "restore and replace."**

You need to understand the business transaction timeline.

---

# 9. RPO

Now we introduce:

# Recovery Point Objective

RPO answers:

> **How much data can the business afford to lose?**

Example:

```text
RPO = 5 minutes
```

Means the business is designed around potentially losing up to roughly five minutes of recent data in the relevant failure scenario.

---

# 10. RTO

RTO answers:

> **How quickly must the service be restored?**

Example:

```text
RTO = 30 minutes
```

Business says:

> "We can tolerate the application being unavailable for up to 30 minutes."

So:

```text
RPO = How much data?
RTO = How much time?
```

Memorize this.

---

# 11. Backup vs DR

Suppose:

```text
Database accidentally deleted
```

Use:

```text
Backup/PITR
```

But suppose:

```text
Entire Azure region becomes unavailable
```

Now you need:

```text
DR architecture
```

Conceptually:

```text
East US
   |
   v
Primary MI
   |
   | Replication / DR architecture
   v
West US
   |
   v
Secondary
```

---

# 12. Geo-DR

Regional disaster scenario:

```text
Primary Region
      |
      X
Regional outage
```

Your organization can fail over to a secondary region, depending on the DR architecture you've configured.

The application then needs to connect to the DR environment.

---

# 13. DR is not automatic just because backups exist

This is an important interview point.

Imagine:

```text
Primary region
    |
    v
Backups
```

Backups help recover data.

But they don't automatically mean:

```text
Application
    ↓
Secondary region
    ↓
Instant recovery
```

A real DR design includes:

```text
Data protection
+
Secondary environment
+
Network
+
Identity
+
Application
+
DNS/connection strategy
+
Failover process
+
Testing
```

---

# 14. DR architecture

A realistic architecture:

```text
             Users
               |
               v
          Application
               |
        +------+------+
        |             |
        v             v
     Region A       Region B
      Primary       Secondary
        |             |
        v             v
      SQL MI        SQL MI
        |             |
        +------+------+
               |
            DR process
```

The application must know how to reach the correct environment during failover.

---

# 15. DR test

A common mistake is:

> "We have DR because we configured it."

No.

You have DR when you **test it**.

A DR exercise might be:

```text
09:00
Start DR test

09:05
Confirm secondary health

09:10
Fail over

09:15
Application connects

09:20
Run business validation

09:30
Declare test successful
```

Measure:

```text
Actual RTO
Actual RPO
Connection recovery
Data consistency
Application behavior
```

---

# 16. Backup retention

Backup retention matters.

Suppose your organization says:

> "We need to recover data from three months ago."

But your available PITR window doesn't cover that period.

PITR alone won't solve it.

That's where longer-term retention strategies may be required.

Think:

```text
Short-term recovery
        ↓
PITR

Long-term recovery
        ↓
Long-term retention
```

The exact retention capabilities and limits depend on the MI configuration and Azure service offering.

---

# 17. Full recovery strategy

A mature DBA environment might have:

```text
                    DATA PROTECTION
                           |
        +------------------+------------------+
        |                  |                  |
        v                  v                  v
       HA                 PITR               DR
        |                  |                  |
   Availability        Logical error       Region outage
                           |
                           v
                    Long-term retention
```

Each solves a different failure scenario.

---

# 18. Real-world disaster scenario

Let's say ABC Retail processes:

```text
$5 million/day
```

Requirements:

```text
RPO = 5 minutes
RTO = 30 minutes
```

Your architecture must be evaluated against those requirements.

You cannot simply say:

> "We use Azure, so we're protected."

Instead ask:

```text
How quickly can we detect failure?
How quickly can we fail over?
How much data can be lost?
How does the application reconnect?
Does DNS change?
Does authentication work?
Is the secondary region ready?
Have we tested it?
```

---

# 19. Backup monitoring

A DBA should not assume:

> "Azure handles backups, therefore backups are automatically fine."

You still need operational confidence.

Monitor:

```text
Backup configuration
Retention requirements
Restore capability
DR health
Alerts
Compliance
```

Most importantly:

> **A backup that has never been restored is not a recovery strategy you have proven.**

---

# 20. Restore testing

Imagine you have:

```text
Backup = YES
```

But when you try restoring:

```text
Restore = FAILED
```

because:

```text
Application dependencies missing
Permissions missing
Network unavailable
Storage unavailable
Encryption key unavailable
```

That's why restore testing matters.

A good process:

```text
Backup
 ↓
Restore
 ↓
Validate
 ↓
Document
 ↓
Automate where possible
```

---

# 21. Accidental DROP DATABASE

Production incident:

```sql
DROP DATABASE CustomerDB;
```

First:

**Don't panic.**

Then:

```text
1. Confirm what happened
2. Stop destructive automation if necessary
3. Determine exact incident time
4. Determine recovery objective
5. Identify available restore options
6. Restore to a safe target
7. Validate data
8. Plan production recovery
```

Don't blindly start multiple recovery operations.

---

# 22. Accidental DELETE

Incident:

```sql
DELETE FROM Orders
WHERE OrderDate >= '2026-08-01';
```

The DBA discovers it 3 minutes later.

Possible strategy:

```text
PITR restore
     ↓
Restore separate database
     ↓
Identify affected rows
     ↓
Compare with production
     ↓
Recover only necessary data
```

This is much safer than automatically replacing the production database.

---

# 23. Ransomware scenario

Imagine:

```text
Application credentials compromised
          |
          v
Attacker modifies/deletes data
```

HA may replicate those changes.

But a proper backup/recovery strategy provides another recovery path.

Security + backup + DR work together:

```text
Security
   +
Backup
   +
DR
   +
Monitoring
```

No single mechanism is enough.

---

# 24. Interview question

### "What is the difference between HA, backup, and DR?"

Strong answer:

> "HA is primarily about maintaining service availability during infrastructure or service failures. Backup and point-in-time recovery are designed to recover data from logical or operational mistakes such as accidental deletion or corruption. DR addresses larger failure domains such as a regional outage. A production architecture may need all three."

That's a very important answer.

---

# 25. Another interview question

### "A developer accidentally deletes 50,000 customer records. What do you do?"

Strong answer:

> "First I would determine the exact time and scope of the deletion and stop any automation that could make the situation worse. I would use point-in-time recovery to restore a copy of the database to a point before the deletion, validate the affected data, and determine whether selective data recovery or full database recovery is appropriate. I would avoid immediately overwriting production until the recovery data and business impact are validated."

Excellent DBA answer.

---

# 26. Your complete recovery mental model

Memorize this:

```text
                    FAILURE
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
   Infrastructure   Logical error   Region failure
        |              |              |
        v              v              v
       HA             PITR            DR
                       |
                       v
                Restore/Recover
```

And:

```text
RPO = How much data can I lose?

RTO = How quickly must I recover?
```

---

# Lesson 12 challenge 🧑‍💻

Production incident:

At **2:15 PM**, a developer accidentally runs:

```sql
UPDATE Customers
SET CreditLimit = 0;
```

It affects **1.2 million customers**.

The mistake is discovered at **2:20 PM**.

Meanwhile, legitimate orders continued processing between 2:15 and 2:20.

Answer:

**1. Would HA protect you from this mistake? Why?**

**2. What recovery technology would you consider first?**

**3. Why would you restore to a separate database initially?**

**4. Why might restoring the entire production database to 2:14 PM be dangerous?**

**5. What is the difference between RPO and RTO?**

**6. If the entire Azure region becomes unavailable, is PITR alone your complete DR solution? Why?**
