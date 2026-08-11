# Lesson 82 — Azure SQL Managed Instance: Backup & Restore Deep Dive

This is one of the most important areas for a SQL Server DBA.

A production DBA should be able to answer:

> **"If someone accidentally deletes critical data at 2:15 PM, how do you recover it without making the situation worse?"**

Let's build that skill from the fundamentals to real production scenarios.

---

# 1. First: What are we protecting against?

Backups aren't only for:

```text
SQL Server crashes
```

They protect against many failure types:

```text
                 BACKUP / RECOVERY
                       |
       +---------------+----------------+
       |               |                |
 Accidental       Corruption        Disaster
 deletion
       |
 Bad UPDATE
       |
 Bad deployment
       |
 Application bug
       |
 Security incident
```

---

# 2. HA vs Backup — again

This distinction is so important that it's worth repeating.

### HA

Protects primarily against:

```text
Infrastructure/service availability failures
```

### Backup

Protects against:

```text
Logical data loss
Corruption
Accidental deletion
Point-in-time recovery
Longer-term recovery requirements
```

For example:

```sql
DELETE FROM Orders;
```

HA doesn't magically know that the statement was a mistake.

Backup/recovery mechanisms are what give you a recovery path.

---

# 3. Backup types — SQL Server fundamentals

Traditional SQL Server backup terminology includes:

```text
Full
Differential
Transaction Log
```

Let's understand each.

---

# 4. Full backup

A full backup represents the database data needed to restore the database to a particular backup point.

Conceptually:

```text
Database
   ↓
FULL BACKUP
   ↓
Database backup
```

Example:

```text
Sunday 12:00 AM
       ↓
Full backup
```

---

# 5. Differential backup

A differential backup captures changes since the relevant full backup.

Example:

```text
Sunday
Full backup
   |
   +--- Monday changes
   |
   +--- Tuesday changes
   |
   +--- Wednesday changes
```

Wednesday's differential represents changes since the full backup.

---

# 6. Transaction log backup

For databases using the FULL or BULK_LOGGED recovery model in traditional SQL Server, transaction log backups are central to point-in-time recovery.

Conceptually:

```text
Full backup
     +
Log backups
     +
Target time
     ↓
Point-in-time recovery
```

Example:

```text
Full
 ↓
Log 1
 ↓
Log 2
 ↓
Log 3
 ↓
Log 4
```

---

# 7. Important Azure SQL MI distinction

With **Azure SQL Managed Instance**, Azure provides automated backup management as part of the managed service.

You don't operate it exactly like an on-premises SQL Server where you build your own SQL Agent jobs to copy `.bak` files every night.

The service handles automated backups and backup storage according to the configured service capabilities and retention settings.

Your job is to understand:

```text
What recovery capabilities exist?
What retention do we have?
How do we restore?
What RPO/RTO does this support?
```

---

# 8. Point-in-time restore

This is one of the most important recovery capabilities.

Suppose:

```text
12:00 PM → Normal
12:30 PM → Developer runs bad UPDATE
12:35 PM → Problem discovered
```

You don't necessarily want:

```text
Restore yesterday's backup
```

You want:

```text
Restore to a point before 12:30 PM
```

That's point-in-time recovery.

---

# 9. PITR concept

Conceptually:

```text
              Database history
                    |
12:00 ─── 12:10 ─── 12:20 ─── 12:30 ─── 12:40
                              ↑
                         Bad UPDATE
                              |
                         Choose recovery
                         point before this
```

You restore to a new database at the desired recovery point, rather than simply overwriting the original database blindly.

---

# 10. Why restore to a new database?

Suppose the production database is:

```text
SalesDB
```

A safer recovery approach is often:

```text
SalesDB
   |
   +---- SalesDB_Restore_20260810
```

Now you can inspect the recovered database.

For example:

```sql
SELECT *
FROM SalesDB_Restore_20260810.dbo.Orders;
```

You compare the recovered data with production.

This is much safer than immediately replacing production.

---

# 11. Real production scenario

At:

```text
2:15 PM
```

a developer runs:

```sql
DELETE FROM dbo.Orders;
```

No `WHERE`.

Production:

```text
Orders = 50 million rows
```

The transaction commits.

At:

```text
2:17 PM
```

the application team notices.

At:

```text
2:20 PM
```

they call you.

---

# 12. First rule: STOP MAKING IT WORSE

Do not immediately start random restores.

First:

```text
Stop affected application activity if appropriate
Identify exactly what happened
Preserve evidence
Determine transaction/time
Determine scope
```

For example:

```text
Was it:
DELETE?
UPDATE?
DROP TABLE?
DROP DATABASE?
Schema deployment?
```

---

# 13. Establish the timeline

You want:

```text
Last known good
       ↓
Bad operation
       ↓
Discovery
       ↓
Current state
```

Example:

```text
2:10 PM
Last known good

2:15 PM
DELETE executed

2:17 PM
Application detects issue

2:20 PM
DBA notified
```

Now your recovery target is approximately:

```text
2:14:59 PM
```

or another appropriate point immediately before the destructive transaction.

---

# 14. Don't restore blindly

Imagine the developer says:

> "Restore the database to 2 PM."

You should ask:

> "Why 2 PM?"

If the bad operation happened at:

```text
2:15 PM
```

then 2 PM is unnecessarily old.

You could lose legitimate transactions between:

```text
2:00 → 2:15
```

That's why timeline analysis matters.

---

# 15. Better approach

Restore to:

```text
Just before the bad operation
```

Then:

```text
Recovered DB
     ↓
Identify missing rows
     ↓
Extract required data
     ↓
Apply controlled correction
     ↓
Validate
```

This minimizes collateral data loss.

---

# 16. The recovery workflow

```text
Incident
   ↓
Stop/contain application activity
   ↓
Identify bad operation
   ↓
Determine recovery point
   ↓
Perform point-in-time restore
   ↓
Validate restored database
   ↓
Extract/correct data
   ↓
Reconcile
   ↓
Resume application
```

---

# 17. Why not simply restore production?

Suppose legitimate transactions happened after 2:15 PM.

Example:

```text
2:16 PM → New order
2:17 PM → Payment
2:18 PM → Shipment
```

If you restore production to 2:14 PM:

```text
All those legitimate transactions disappear.
```

That's why recovery should often be surgical.

---

# 18. Recovery options

Depending on the incident, you might:

### Option A

Restore entire database to a new database.

### Option B

Restore and extract specific missing rows.

### Option C

Restore to a point in time and use it as the new production database.

### Option D

Use another recovery mechanism if the architecture provides a more appropriate path.

The correct approach depends on:

```text
Scope
Data loss
Application state
RPO
RTO
Transaction volume
```

---

# 19. Scenario: entire database deleted

Suppose:

```text
DROP DATABASE SalesDB;
```

Now the problem is different.

You aren't recovering individual rows.

You're recovering the database itself.

You need to determine:

```text
When was the last valid recovery point?
What backup/recovery capability is available?
What is the required RTO?
What dependencies does the application have?
```

---

# 20. Scenario: database corruption

Suppose application errors begin:

```text
Msg ...
Corruption detected
```

Now:

```text
Don't assume the newest backup is automatically the best recovery point.
```

You may need to determine:

```text
When did corruption begin?
Which backups are clean?
Can the issue be isolated?
What recovery point is safe?
```

This is where monitoring and historical evidence become extremely valuable.

---

# 21. Scenario: bad UPDATE

This is perhaps more common than database deletion.

Developer executes:

```sql
UPDATE Customers
SET CreditLimit = 0;
```

Oops.

No `WHERE`.

Now every customer has:

```text
CreditLimit = 0
```

Recovery challenge:

```text
Production contains legitimate changes
```

between the last backup and the incident.

You don't want to blindly restore the entire database.

Instead:

```text
PITR restore
      ↓
Recover original customer values
      ↓
Compare
      ↓
Generate corrective UPDATE
      ↓
Validate
```

---

# 22. Example recovery

Production:

```text
CustomerID | CreditLimit
-----------|------------
101        | 5000
102        | 7000
103        | 9000
```

Bad UPDATE:

```sql
UPDATE Customers
SET CreditLimit = 0;
```

Restore a copy to just before the bad statement.

Recovered:

```text
CustomerID | CreditLimit
-----------|------------
101        | 5000
102        | 7000
103        | 9000
```

You can then build a controlled reconciliation.

---

# 23. Why backup validation matters

A backup that has never been restored is an assumption.

A mature DBA asks:

> **"When was the last successful restore test?"**

Not:

> "Do we have backups?"

---

# 24. Restore testing

A good process:

```text
Backup
  ↓
Restore test
  ↓
Database accessible?
  ↓
Data valid?
  ↓
Application validation
  ↓
Record recovery time
```

Measure:

```text
Actual restore duration
Actual data availability
Actual RPO
Actual RTO
```

---

# 25. Large database recovery

Suppose:

```text
Database = 20 TB
```

Management says:

> "Restore it in 10 minutes."

Don't just say:

> "Sure."

Calculate whether the architecture can physically support the required recovery time.

Consider:

```text
Database size
Storage throughput
Network
Compute
Backup location
Restore mechanism
Database initialization
Application startup
Validation
```

RTO must be **tested**, not guessed.

---

# 26. Recovery time isn't just restore time

Suppose:

```text
Restore = 20 minutes
```

But:

```text
Application configuration = 10 minutes
DNS = 5 minutes
Validation = 10 minutes
```

Total:

```text
45 minutes
```

Your actual RTO is closer to:

```text
45 minutes
```

not 20.

---

# 27. Backup security

Backups contain your data.

Therefore consider:

```text
Encryption
Access control
Identity
Retention
Audit
Network security
```

A backup isn't harmless simply because it's not the production database.

---

# 28. Encryption

Azure SQL Managed Instance supports encryption capabilities for data at rest, and backup protection is part of the platform's security model.

From a DBA perspective, you need to understand:

```text
Who can access backups?
How are encryption keys managed?
What happens during restore?
What permissions are required?
```

---

# 29. Separation of duties

Imagine:

```text
Developer
   ↓
Production database access
```

and:

```text
Developer
   ↓
Backup deletion
```

That's dangerous.

A mature environment separates responsibilities.

For example:

```text
Developer
   → application/data access

DBA
   → database operations

Security
   → privileged access

Platform team
   → Azure infrastructure
```

Exact roles vary by organization.

---

# 30. Backup retention

Imagine the company says:

> "We need to recover data from 60 days ago."

Your normal operational PITR window may not be enough.

You need a retention strategy that meets the business requirement.

Ask:

```text
How far back must we recover?
```

Not merely:

```text
How often do we back up?
```

---

# 31. Regulatory example

Suppose you're working with financial records.

Requirement:

```text
Historical recovery / retention
```

You might need:

```text
Short-term operational recovery
+
Long-term retention
```

These solve different problems.

---

# 32. Migration using backup/restore

Backup/restore isn't only disaster recovery.

It can also be useful in migration scenarios.

For example:

```text
On-prem SQL Server
        ↓
Backup
        ↓
Azure-compatible recovery/migration process
        ↓
SQL Managed Instance
```

The exact migration method depends on database size, downtime tolerance, compatibility, networking, and the Azure migration tooling/approach selected.

---

# 33. Migration scenario

Suppose:

```text
On-prem SQL Server
Database = 4 TB
```

Business requirement:

```text
Downtime < 1 hour
```

A traditional:

```text
Backup
↓
Copy 4 TB
↓
Restore
```

may or may not fit the requirement.

You need to calculate:

```text
Backup time
Transfer time
Restore time
Validation
Application cutover
```

---

# 34. Migration is an RTO problem too

Think:

```text
Migration downtime
=
Backup
+
Transfer
+
Restore
+
Validation
+
Cutover
```

If this exceeds the business window, choose another migration architecture.

---

# 35. Backup and DR are related but different

A clean mental model:

```text
                 DATA PROTECTION
                       |
          +------------+------------+
          |                         |
       Backup                     DR
          |                         |
   Recover data              Recover service
          |                         |
   PITR / restore            Secondary region
```

Backup may help you rebuild.

DR aims to restore service quickly.

---

# 36. Real-world ransomware scenario

Imagine:

```text
10:00 AM
Normal

10:20 AM
Suspicious activity

10:30 AM
Encryption attack discovered
```

You need to think about:

```text
Which systems are compromised?
Are production credentials compromised?
Are backups accessible to attacker?
What recovery points are trustworthy?
Can you restore safely?
```

This becomes both:

```text
Security incident
+
Database recovery incident
```

---

# 37. Don't immediately trust the latest recovery point

If an attacker has altered data for several hours, the latest backup may contain compromised data.

You need:

```text
Incident timeline
+
Known-good recovery point
```

This is why security monitoring and database recovery planning must work together.

---

# 38. Recovery validation

After restoring:

```text
Database starts
```

doesn't mean:

```text
Recovery successful
```

Validate:

```text
Tables
Row counts
Critical records
Constraints
Stored procedures
Permissions
Application connectivity
Business transactions
```

---

# 39. Business validation

Suppose the database is online.

The DBA says:

> "Recovery complete."

Business says:

> "Payments are failing."

Not complete.

Your validation should cover the entire application path:

```text
User
 ↓
Application
 ↓
Network
 ↓
SQL MI
 ↓
Database
```

---

# 40. Recovery checklist

For a serious recovery:

```text
☐ Identify incident
☐ Stop/contain impact
☐ Establish timeline
☐ Determine recovery point
☐ Verify recovery capability
☐ Restore to safe location
☐ Validate data
☐ Compare against production
☐ Correct required data
☐ Validate application
☐ Resume traffic
☐ Monitor
☐ Document incident
☐ Review root cause
```

---

# 41. Interview question

### "A developer accidentally deleted a table. What do you do?"

Strong answer:

> "First I'd stop or contain further changes if necessary and determine exactly when and what was deleted. I'd identify the latest known-good recovery point and restore a copy of the database to a point immediately before the destructive operation. I'd validate the restored data, extract or reconstruct the required object/data, and then perform a controlled recovery in production. I would avoid blindly restoring production because legitimate transactions may have occurred after the incident."

Excellent DBA answer.

---

# 42. Interview question

### "What is point-in-time restore?"

> "Point-in-time restore allows recovery of a database to a specific time within the available recovery window, which is especially useful for accidental data modifications or other logical errors."

---

# 43. Interview question

### "Why do you restore to a new database?"

Because it allows:

```text
Production
      +
Recovered copy
      ↓
Compare
Validate
Extract
```

without immediately destroying current production state.

It's a safer incident-response pattern.

---

# 44. Interview question

### "Are backups enough for DR?"

Strong answer:

> "Not necessarily. Backups provide recovery capability, but restoring a large database may take too long for a strict RTO. For aggressive RTO/RPO requirements, a separate DR architecture may be required."

---

# 45. Interview question

### "How do you prove backups work?"

Don't say:

> "The backup job says successful."

Say:

> **"I perform periodic restore tests and measure the actual recovery time and data recovery point, then validate the database and application."**

That's the senior-level answer.

---

# 46. The DBA recovery mindset

When something goes wrong:

```text
Don't panic
     ↓
Don't make changes blindly
     ↓
Establish timeline
     ↓
Determine scope
     ↓
Choose recovery point
     ↓
Restore safely
     ↓
Validate
     ↓
Recover selectively
```

This discipline prevents a bad incident from becoming a catastrophic one.

---

# 47. One scenario to remember forever

Developer runs:

```sql
UPDATE Orders
SET Status = 'Cancelled';
```

At:

```text
2:15 PM
```

They realize at:

```text
2:20 PM
```

Your thinking should immediately become:

```text
2:15 = incident time
       ↓
Find last known-good point
       ↓
Restore a copy
       ↓
Recover original values
       ↓
Reconcile with post-incident transactions
       ↓
Controlled production correction
```

**Do not simply restore production to 2:14 PM.**

That could destroy legitimate transactions from:

```text
2:15 → 2:20
```

---

# 48. Final mental model

You should now have:

```text
                    SQL MI
                      |
       +--------------+---------------+
       |              |               |
      HA            Backup            DR
       |              |               |
 Availability       PITR          Regional recovery
       |              |
 Zones              Logical recovery
```

And the three questions:

```text
RPO → How much data can we lose?

RTO → How much downtime can we tolerate?

Recovery → How do we actually prove it works?
```

---

## Next — Lesson 83: Azure SQL Managed Instance Security

We'll move into **production security**, including:

* Microsoft Entra authentication
* SQL authentication
* Managed identities
* RBAC vs SQL permissions
* Logins vs users
* Least privilege
* TDE
* Always Encrypted concepts
* Key Vault
* Microsoft Defender for SQL
* Auditing
* Vulnerability assessment
* Private endpoints/network security
* Firewall concepts
* Service principals
* Secret management
* Real-world security incidents
* and interview scenarios such as:

> **"A developer needs read-only access to one database on production SQL MI. How would you design the access?"**
