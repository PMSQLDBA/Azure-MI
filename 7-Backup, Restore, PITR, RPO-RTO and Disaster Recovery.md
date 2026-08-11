Previous: 
Correct answers are **B and E**. 
Since DNS, network, MI health, database state, and username are all confirmed, I'd next investigate the **credential/authentication configuration** and whether the **login is disabled or otherwise unable to authenticate**.


# Lesson 5 — Backup, Restore, PITR, RPO/RTO & Disaster Recovery

For a production DBA, this topic is critical because:

> **HA keeps the service available. Backups help you recover data. DR helps you survive larger failures.**

These are related, but they're **not the same thing**.

---

# 1. First understand the three concepts

Think about ABC Retail:

```text
                 PRODUCTION
                     |
             SQL Managed Instance
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
      HA          Backups          DR
       |             |              |
       v             v              v
 Infrastructure   Data recovery   Region failure
    failure       / mistakes
```

### HA

Protects primarily against service/infrastructure failures.

### Backup

Protects against:

```text
Accidental DELETE
Bad UPDATE
Data corruption
Recovery requirements
```

### DR

Protects against:

```text
Regional disaster
Major service disruption
Business continuity events
```

---

# 2. Automated backups

With Azure SQL Managed Instance, Azure manages automated database backups.

Conceptually:

```text
SalesDB
   |
   +---- Full backup
   |
   +---- Differential/log backups
   |
   v
Azure-managed backup storage
```

Microsoft states that MI automatically manages database backups and supports point-in-time restore within the configured retention period, with retention up to **35 days**.

This means you don't have to build your traditional:

```text
SQL Agent
   ↓
BACKUP DATABASE
   ↓
.bak file
   ↓
Backup server
   ↓
Tape/object storage
```

architecture for the normal automated backup capability.

---

# 3. But don't misunderstand this

Some DBAs hear:

> "Azure manages backups."

and think:

> "I don't need to think about backups anymore."

Wrong.

You still need to understand:

```text
Backup retention
Restore process
PITR
Long-term retention requirements
DR
Recovery testing
RPO
RTO
Application consistency
```

The DBA responsibility moves from:

> **"Run every backup job manually."**

toward:

> **"Design, validate, monitor and test recovery."**

---

# 4. Point-in-time restore

This is one of the most useful features.

Imagine:

```text
10:00 AM
Database healthy

10:15 AM
Database healthy

10:27 AM
Developer runs DELETE

10:30 AM
Someone notices

10:35 AM
Incident declared
```

Timeline:

```text
10:00 ─── 10:15 ─── 10:27 ─── 10:30 ─── 10:35
                         X
                       DELETE
```

You can restore the database to a point before the mistake, assuming it's within the available retention window.

For example:

```text
ProductionDB
     |
     |
     +--------------------+
                          |
                          ↓
                  Point-in-time
                     restore
                          |
                          ↓
                ProductionDB-Restore
```

**Important:** You generally restore to a **new database**, then validate it before deciding how to use it for recovery.

---

# 5. Real production scenario

Developer executes:

```sql
DELETE FROM Orders;
```

without a `WHERE` clause.

😱

Millions of rows disappear.

Your first instinct should **not** be:

```sql
ROLLBACK;
```

because if the transaction has already committed, you can't simply roll it back.

Instead, determine:

```text
When did the mistake occur?
What data was affected?
Do we need the entire database restored?
Can we recover only the affected data?
```

If a point-in-time restore is appropriate:

```text
Current Production
       |
       |
       v
Restore to 10:26 AM
       |
       v
SalesDB-Recovered
```

Then:

```text
Validate
   ↓
Identify missing rows
   ↓
Extract required data
   ↓
Repair production
```

This is often safer than blindly replacing the entire production database.

---

# 6. Very important DBA lesson

**Backup is not the same as recovery.**

You can have:

```text
Backups = SUCCESS
```

and still have:

```text
Recovery = FAILED
```

if you don't know:

* How to restore
* How long restoration takes
* Where the restored database goes
* How applications reconnect
* How permissions are restored
* How data is validated

That's why:

> **Recovery testing is more important than simply checking that backups completed.**

---

# 7. RPO

Now let's understand **RPO — Recovery Point Objective**.

Ask:

> **"How much data can the business afford to lose?"**

Example:

```text
RPO = 5 minutes
```

means, conceptually:

> In a disaster, the business can tolerate losing up to about 5 minutes of data.

Example:

```text
12:00 ─── 12:05 ─── 12:10 ─── 12:15
                         X
                       Failure
```

If the recovery point is around 12:10:

```text
Potential data loss
≈ 5 minutes
```

RPO is a **business requirement**, not simply a database setting.

---

# 8. RTO

RTO means:

> **How long can the application be unavailable?**

Example:

```text
RTO = 30 minutes
```

If disaster occurs:

```text
2:00 PM
   |
   X
Failure
   |
   |
   v
2:30 PM
   |
   v
Application restored
```

The business wants recovery within approximately 30 minutes.

---

# 9. RPO vs RTO

Memorize this:

```text
RPO = DATA
      "How much data can I lose?"

RTO = TIME
      "How long can I be down?"
```

Or:

> **RPO = how far backward.**

> **RTO = how fast forward.**

---

# 10. Real business requirement

ABC Retail tells you:

```text
RPO = 5 minutes
RTO = 30 minutes
```

That means:

```text
Data loss tolerance
        ↓
~5 minutes

Downtime tolerance
        ↓
~30 minutes
```

Now your architecture must be designed around those requirements.

---

# 11. Geo-DR architecture

Suppose ABC Retail has:

```text
PRIMARY REGION
East US
   |
   v
SQL Managed Instance
   |
   |
   | Replication
   |
   v
SECONDARY REGION
West US
   |
   v
SQL Managed Instance
```

If East US experiences a major regional failure:

```text
                 USERS
                   |
                   v
               Application
                   |
             Primary unavailable
                   |
                   X
                   |
                   v
              Secondary MI
```

The application can be redirected/fail over according to the DR architecture.

---

# 12. Failover groups

For supported Managed Instance scenarios, **failover groups** can provide a way to manage geo-failover between instances.

Conceptually:

```text
              Application
                   |
                   v
             Failover endpoint
                   |
             +-----+-----+
             |           |
             v           v
         Primary      Secondary
          MI             MI
```

The advantage is that applications can use a stable endpoint rather than hard-coding a specific primary instance.

Microsoft documents failover groups as a business continuity/DR mechanism for Managed Instance.

---

# 13. DR is not just replication

This is a common interview trap.

Someone says:

> "We have replication, so we have DR."

Not necessarily.

A real DR plan needs:

```text
Replication
+
Failover procedure
+
Application recovery
+
DNS/connection strategy
+
Security
+
Monitoring
+
Testing
+
Failback procedure
```

Imagine your database is available in the DR region but:

```text
Application → ❌
Authentication → ❌
DNS → ❌
Secrets → ❌
```

You don't have a functioning DR environment.

You have a replicated database.

That's different.

---

# 14. DR drill

Let's simulate an incident.

ABC Retail:

```text
Primary Region: East US
DR Region: West US
```

At 3:00 PM:

> Azure region outage.

Your DBA team receives an alert.

What should happen?

### Step 1 — Confirm incident

```text
Is this actually a regional outage?
```

### Step 2 — Check primary

```text
MI status
Connectivity
Application health
```

### Step 3 — Evaluate RPO

```text
How current is secondary?
```

### Step 4 — Initiate failover according to the approved runbook

```text
Primary
   ↓
Secondary
```

### Step 5 — Application validation

Test:

```text
Login
Read
Insert
Update
Transactions
Background jobs
```

### Step 6 — Business validation

```text
Orders working?
Payments working?
Reports working?
```

Only then declare recovery successful.

---

# 15. Failback

Suppose East US becomes healthy again.

Don't immediately switch production back.

First:

```text
East US
   ↓
Healthy?
   ↓
Data synchronized?
   ↓
Application tested?
   ↓
Failback approved?
```

Then:

```text
West US
   ↓
Production
```

becomes:

```text
East US
   ↓
Production
```

Failback should be treated as another controlled change.

---

# 16. Backup vs HA — very important

Imagine someone accidentally executes:

```sql
DELETE FROM Customer;
```

Your HA replica might also receive that change.

So:

```text
Primary
   |
   | DELETE
   v
Secondary
   |
   ↓
Bad data replicated
```

That's why:

> **HA does not replace backups.**

Backups give you a recovery path to an earlier point in time.

---

# 17. Backup vs DR

Suppose the entire Azure region fails.

A point-in-time restore may not be your fastest recovery mechanism.

You may need:

```text
Secondary Region
      ↓
Existing MI
      ↓
Failover
```

That's where DR architecture becomes important.

Think:

```text
Accidental data deletion
        ↓
Backup / PITR

Infrastructure failure
        ↓
HA

Regional disaster
        ↓
DR
```

---

# 18. Production architecture

Now our ABC Retail design becomes:

```text
                         USERS
                           |
                           v
                      APPLICATION
                           |
                           v
                    PRIMARY REGION
                           |
                    SQL Managed Instance
                           |
              +------------+------------+
              |                         |
              v                         v
          Automated                  HA
           Backups
              |
              |
              +-----------------------------+
                                            |
                                      DR / Replication
                                            |
                                            v
                                   SECONDARY REGION
                                            |
                                    SQL Managed Instance
```

This is the mental model you need.

---

# 19. Recovery testing

A very strong DBA doesn't say:

> "Our backups are successful."

They say:

> **"We successfully performed a restore test and measured recovery time."**

For example:

```text
Backup status       SUCCESS
Restore test        SUCCESS
Restore duration    18 minutes
Data validation     SUCCESS
Application test    SUCCESS
```

Now you have evidence.

---

# 20. Interview questions

### Q1. What is RPO?

> The maximum acceptable amount of data loss measured in time.

### Q2. What is RTO?

> The maximum acceptable time required to restore service.

### Q3. Does HA replace backups?

> No. HA primarily improves availability; backups provide recovery from data loss, corruption, and user mistakes.

### Q4. Does replication guarantee DR?

> No. A complete DR solution also requires failover, application connectivity, authentication, validation, monitoring, and tested recovery procedures.

### Q5. Developer accidentally deleted data. What do you do?

Think:

```text
Stop further damage
       ↓
Determine exact incident time
       ↓
Assess affected data
       ↓
Use appropriate recovery mechanism
       ↓
Restore/PITR if appropriate
       ↓
Validate
       ↓
Repair production
```

---

# Your Lesson 5 challenge 🧑‍💻

Here's your production incident:

> **ABC Retail's developer accidentally deleted 500,000 customer records at 2:15 PM. The transaction committed successfully. The primary and secondary MI replicas are both healthy. The business discovers the issue at 2:25 PM.**

Answer these:

**1. Would HA/secondary replica help recover the deleted rows? Why or why not?**

**2. What Azure recovery capability would you investigate first?**

**3. Why wouldn't you immediately replace the entire production database with the restored database?**

**4. What does RPO = 5 minutes mean?**

**5. What does RTO = 30 minutes mean?**

After this, we'll move to **Lesson 6 — Migration from SQL Server 2019 → Managed Instance**, where we'll actually design the migration plan: **assessment → compatibility → networking → migration method → pre-cutover → cutover → validation → rollback**.
