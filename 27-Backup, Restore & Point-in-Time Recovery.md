# Lesson 55 — Azure SQL Managed Instance: Backup, Restore & Point-in-Time Recovery

This is one of the most important areas for a production SQL DBA.

The scenario:

> **A developer accidentally deletes 2 million customer records at 2:17 PM.The business wants the data as it existed at 2:16 PM, but they do NOT want to lose legitimate transactions that occurred after 2:16 PM.**

That last requirement changes the recovery strategy completely.

---

# 1. First understand the recovery problem

Timeline:

```text
2:00 PM
Normal operations
     |
     v
2:10 PM
Normal transactions
     |
     v
2:16 PM
Good state
     |
     v
2:17 PM
Developer accidentally deletes
2 million rows
     |
     v
2:18 PM
Application continues working
     |
     v
2:30 PM
Incident discovered
```

The business says:

> "Give me the records from 2:16 PM, but don't throw away everything that happened from 2:16 to 2:30."

You cannot simply restore the production database backward and pretend nothing happened.

Why?

Because you'd potentially lose valid transactions between:

```text
2:16 → 2:30
```

---

# 2. PITR — Point-in-Time Restore

Point-in-time restore means:

> **Restore a database to a specific point in time within the available recovery window.**

Conceptually:

```text
Backups + transaction history
          |
          v
      Recovery
          |
          v
   2:16 PM state
```

You can create a restored copy representing the database at approximately the desired point in time, subject to the service's supported recovery capabilities and retention configuration.

---

# 3. Critical DBA principle

Don't immediately restore production.

Instead:

```text
Production
    |
    | remains available
    |
    v
Restore a separate copy
    |
    v
Recover required data
    |
    v
Validate
    |
    v
Merge/reconcile
    |
    v
Production
```

This is often much safer.

---

# 4. Real-world recovery strategy

Suppose the accidental DELETE was:

```sql id="a9v4m0"
DELETE FROM Customer.Contacts;
```

At:

```text id="1y5m7h"
2:17 PM
```

You discover it at:

```text id="t9e8q4"
2:30 PM
```

A safe approach can be:

```text id="b1qf4e"
Production
   |
   +----------------------+
   |                      |
   v                      v
Continue normal work   PITR restore
                        to 2:16 PM
                             |
                             v
                       Recovered data
                             |
                             v
                       Validate missing
                       customer records
                             |
                             v
                    Reinsert/reconcile
                             |
                             v
                         Production
```

This avoids unnecessarily rolling the whole production database backward.

---

# 5. Why this is better

Suppose legitimate transactions occurred after 2:17 PM:

```text id="n0f8g3"
2:20 PM → New customer
2:22 PM → Order created
2:25 PM → Payment updated
```

If you restore production to:

```text id="kj3g58"
2:16 PM
```

those valid transactions disappear from the restored state.

Instead, recover the **missing data** from the restored copy.

---

# 6. The hardest part: reconciliation

Suppose the deleted table contained:

```text id="4w9sbc"
CustomerID
Name
Email
Phone
Address
```

You restore the database to:

```text id="2:16 PM"
```

Now you have the deleted records.

But some customers may have legitimate changes after 2:17 PM.

For example:

```text id="b1g7mt"
Customer 1001

2:10 PM
Email = old@example.com

2:17 PM
DELETE

2:25 PM
Customer legitimately recreated/updated
```

You can't blindly insert the old record.

You must determine:

```text id="j0r2pu"
What existed before deletion?
What changed afterward?
What should the final production state be?
```

That's a **data reconciliation problem**.

---

# 7. Don't confuse backup with recovery

A backup answers:

> **Can I reconstruct data?**

Recovery asks:

> **How do I safely restore the business to a correct state?**

These aren't identical.

For example:

```text id="50gh9h"
Backup available
      ↓
Great
```

doesn't mean:

```text id="8kq4jk"
Recovery procedure is safe
```

You need to test the actual recovery workflow.

---

# 8. Full restore vs PITR

### Full restore

Conceptually:

```text id="7g5r3j"
Full backup
   ↓
Restore database
```

Useful when you need a full database recovery based on a backup.

### Point-in-time restore

Conceptually:

```text id="49w4wt"
Recovery history
      ↓
Choose time
      ↓
Restore database state
      ↓
Specific point in time
```

Useful when you need:

> "What did the database look like immediately before the incident?"

---

# 9. PITR isn't a time machine

This is important.

Suppose the database was corrupted at:

```text id="5s3p9w"
2:17:03 PM
```

and you request:

```text id="3h4x7z"
2:17:00 PM
```

You need to understand the service's actual recovery granularity, available recovery window, and supported restore behavior.

Don't promise the business:

> "We can restore to exactly any millisecond."

Instead say:

> "We can restore to a supported point in the configured recovery window, subject to the service's recovery capabilities."

---

# 10. RPO enters the conversation

Suppose the business says:

> "We cannot lose more than 5 minutes of transactions."

That's an RPO requirement.

But PITR is about recovering to a desired point in time.

These are related but different concepts.

```text id="j43z0p"
RPO
↓
Maximum tolerable data loss

PITR
↓
Recovery mechanism to a selected time
```

---

# 11. Real production recovery plan

For accidental deletion:

```text id="u3j3vy"
1. Stop further destructive activity
2. Identify exact incident time
3. Determine scope of deletion
4. Preserve production state
5. Restore a separate copy to pre-incident time
6. Validate recovered records
7. Compare with production
8. Identify legitimate post-incident changes
9. Reconcile data
10. Validate application behavior
11. Restore business service
12. Document root cause
```

---

# 12. Should you stop the application?

Sometimes.

But don't automatically shut everything down.

Ask:

```text id="v6c8z1"
Is the destructive operation still running?
Can more data be affected?
Is production still usable?
Can we safely isolate the affected functionality?
```

For example:

```text id="w4t3gc"
Only customer-maintenance API
       ↓
Disable that API

Orders
Payments
Reporting
       ↓
Continue operating
```

That can be safer than shutting down the entire business.

---

# 13. Accidental UPDATE scenario

Suppose instead:

```sql id="3x9d5s"
UPDATE Customer.Customers
SET CreditLimit = 0;
```

and:

```text id="w7f6qx"
1.8 million customers
```

The recovery strategy is similar.

Restore a copy to the point before the bad UPDATE:

```text id="9p7n1c"
PITR copy
   ↓
Correct values
   ↓
Compare with production
   ↓
Generate recovery changes
   ↓
Apply carefully
```

You may not need to replace the entire production database.

---

# 14. What if the entire database is corrupted?

That's different.

Suppose:

```text id="1qv9a5"
Multiple tables corrupted
Application cannot operate
Database integrity compromised
```

Now full database restore may be more appropriate.

The recovery decision becomes:

```text id="m2w3pz"
Small logical mistake
        ↓
Logical data recovery

Large database-level failure
        ↓
Database restore
```

---

# 15. Test your backups

This is one of the most important DBA lessons:

> **A backup you haven't successfully restored is an assumption, not proven recoverability.**

You should periodically test:

```text id="x3p5q6"
Restore
Connectivity
Application validation
Data validation
Permissions
Performance
Recovery duration
```

And measure:

```text id="f4t8v2"
Actual restore time
```

because that affects RTO.

---

# 16. Interview question

### Interviewer:

> "A developer accidentally deletes a large amount of data. Do you restore production to yesterday's backup?"

Strong answer:

> "Not automatically. First I'd determine the exact incident time and scope. If the goal is to recover only the affected data while preserving legitimate transactions after the incident, I'd generally restore a separate copy to a point before the destructive operation, validate the affected records, and reconcile them back into production. If the entire database is compromised or business requirements make logical recovery impractical, then a full database restore may be appropriate. The recovery plan should preserve as much valid post-incident work as possible."

That's a **production DBA answer**.

---

# Lesson 56 — Backup/Restore Interview Questions

## Question 1

**What is RPO?**

> Maximum acceptable amount of data loss measured in time.

Example:

```text
RPO = 5 minutes
```

means the business can tolerate losing up to approximately five minutes of data, subject to the actual recovery architecture.

---

## Question 2

**What is RTO?**

> Maximum acceptable time to restore the business service.

Example:

```text
RTO = 30 minutes
```

means the business expects service restoration within 30 minutes.

---

## Question 3

**Why is a backup strategy not enough?**

Because:

```text
Backup exists
    ≠
Recovery works
```

You must test:

```text
Backup
 ↓
Restore
 ↓
Validation
 ↓
Application
```

---

## Question 4

**What's the difference between backup retention and PITR?**

Think:

```text
Retention
=
How long recovery information is available

PITR
=
Recover to a selected point within the supported recovery window
```

---

## Question 5

**Would you use PITR for every data mistake?**

No.

For example:

```text
Small accidental DELETE
→ PITR copy + logical recovery

Entire database corruption
→ Potential full restore

Regional disaster
→ DR strategy
```

Choose the recovery mechanism based on the **failure scope**.

---

# Lesson 57 — SQL Agent on Managed Instance

Now let's talk about a feature that is extremely important during SQL Server migrations:

## SQL Server Agent

Imagine an existing SQL Server environment has:

```text id="v2z6bg"
300 SQL Agent jobs
```

Examples:

```text id="j2v8q9"
01:00 → ETL
02:00 → Index maintenance
03:00 → Data cleanup
04:00 → Reporting refresh
05:00 → Archive
```

The company migrates the database to MI.

Then developers say:

> "The database migrated successfully, but none of our jobs are running."

This is a **migration dependency failure**.

---

# 1. Database migration ≠ environment migration

A common mistake is to migrate:

```text id="c9d7f2"
Tables
Views
Procedures
Data
```

and assume:

> "Everything is migrated."

But the workload may depend on:

```text id="i0b2m3"
SQL Agent jobs
Credentials
Operators
Alerts
Schedules
Linked servers
SSIS
External scripts
Certificates
Logins
```

You need a **dependency inventory**.

---

# 2. Example

Suppose:

```text id="u1q9oe"
OrdersDB
```

has a nightly process:

```text id="k5l0pd"
SQL Agent Job
    ↓
Execute stored procedure
    ↓
Export file
    ↓
Upload to external system
```

The database itself migrated perfectly.

But the job was never migrated.

At 2:00 AM:

```text id="f0g8qa"
No export
```

At 8:00 AM:

```text id="5j1wzr"
Business reporting missing
```

The database team gets blamed for a "database problem."

It wasn't.

It was a **migration dependency problem**.

---

# 3. Senior DBA migration checklist

Before migration:

```text id="x8m4vq"
☐ Databases
☐ Logins
☐ Users
☐ Roles
☐ SQL Agent jobs
☐ Linked servers
☐ Credentials
☐ Certificates
☐ SSIS dependencies
☐ Reporting
☐ ETL
☐ Application connection strings
☐ External integrations
☐ Monitoring
```

After migration:

```text id="7n6l9s"
☐ Jobs running
☐ Schedules correct
☐ Permissions correct
☐ External integrations working
☐ Application tests passed
```

---

# The big lesson

When migrating SQL Server to Azure SQL MI:

> **Don't migrate only the database. Migrate the workload and its dependencies.**
