# Lesson 74 — High Availability & Disaster Recovery in Azure SQL Managed Instance

Now we're moving from **performance** into one of the most important production DBA responsibilities:

> **What happens when the database or Azure region has a failure?**

For Azure SQL Managed Instance, you need to clearly understand **HA, DR, failover, RPO, and RTO**.

---

# 1. First: HA vs DR

These are often confused.

### High Availability — HA

HA protects you from failures **within the same Azure region/service architecture**.

Think:

```text id="h3v8q1"
Primary SQL MI
      |
      ↓
HA mechanism
      |
      ↓
Another healthy instance/replica
```

Goal:

> **Keep the service available despite infrastructure/component failures.**

---

### Disaster Recovery — DR

DR addresses a larger disaster, such as:

```text id="m7x2p9"
Entire Azure region unavailable
```

Conceptually:

```text id="q4n8c3"
Region A
SQL MI
   X
   |
   | Disaster
   ↓
Region B
SQL MI / DR architecture
```

Goal:

> **Recover the workload in another location.**

---

# 2. Simple mental model

Memorize:

```text id="w5r2k8"
HA  → "Keep running"
DR  → "Recover elsewhere"
```

---

# 3. RPO

RPO = **Recovery Point Objective**

It answers:

> **How much data can the business afford to lose?**

Example:

```text id="j8m3v6"
RPO = 5 minutes
```

The business is saying:

> "In a disaster, our recovery design should limit potential data loss to roughly five minutes, according to the supported architecture and actual conditions."

---

# 4. RTO

RTO = **Recovery Time Objective**

It answers:

> **How long can the business tolerate the service being unavailable?**

Example:

```text id="p6x1q9"
RTO = 30 minutes
```

Meaning:

```text id="r3n7m2"
Failure
 ↓
Recovery
 ↓
Application operational
```

should be within the required target.

---

# 5. RPO vs RTO

This table is worth memorizing:

| Concept | Question                        |
| ------- | ------------------------------- |
| RPO     | How much data can we lose?      |
| RTO     | How long can we be unavailable? |

Think:

```text id="z4m8c1"
RPO → DATA
RTO → TIME
```

---

# 6. Real production requirement

Imagine an online banking system.

Business requirements:

```text id="t7p2x5"
RPO = near-zero / very low
RTO = 15 minutes
```

You cannot simply say:

> "We'll restore yesterday's backup."

That may violate both requirements.

Your architecture must be designed around the business objectives.

---

# 7. Azure SQL Managed Instance HA

Azure SQL Managed Instance is a managed platform.

Azure handles many underlying infrastructure responsibilities, including the platform's high-availability architecture.

This is a major benefit compared with running SQL Server yourself on unmanaged virtual machines.

Conceptually:

```text id="q8v3m6"
Azure SQL MI
      |
      +---- Managed infrastructure
      |
      +---- HA architecture
      |
      +---- Automated platform operations
```

You still need to understand what happens during failures and how your application responds.

---

# 8. What HA protects against

Think about infrastructure failures such as:

```text id="f4n9x2"
Host failure
Hardware failure
Underlying infrastructure issue
Platform maintenance
```

The managed service architecture is designed to maintain availability through supported mechanisms.

But:

> **HA does not protect you from every logical problem.**

---

# 9. Logical failure vs infrastructure failure

Suppose:

```text id="k2m7p4"
Developer:
DELETE FROM Orders;
```

HA doesn't understand:

> "This DELETE is wrong."

If the transaction is valid, the database system treats it as valid work.

So:

```text id="s9x3n6"
HA ≠ logical data protection
```

That's why we previously discussed:

```text id="v5q1r8"
Backups
PITR
Long-term retention
```

---

# 10. Another example

Infrastructure failure:

```text id="m8r4x2"
Host fails
```

HA:

```text id="j6n9p3"
Service continues/recover automatically
```

Logical failure:

```text id="q2c7v5"
Bad UPDATE
```

HA:

```text id="a4x8m1"
Doesn't magically undo it
```

PITR/recovery:

```text id="w7n3k9"
Potential solution
```

---

# 11. Availability Zones

Azure regions can support multiple Availability Zones.

Conceptually:

```text id="p8m3q7"
Azure Region
 |
 +-- Zone 1
 |
 +-- Zone 2
 |
 +-- Zone 3
```

Availability Zones are physically separate locations within an Azure region designed to reduce the impact of localized datacenter-level failures.

The exact capabilities and configuration options depend on the Azure service and region.

---

# 12. Why zones matter

Without zone-level resilience:

```text id="v4n8m2"
Zone failure
    ↓
Service impact
```

With an appropriate zone-resilient architecture:

```text id="x6q1r9"
Zone 1 ──┐
         ├── Resilient service
Zone 2 ──┘
```

The goal is to reduce the blast radius of a localized infrastructure failure.

---

# 13. Zone failure vs regional failure

This distinction is critical.

### Zone failure

```text id="c9m4p2"
Zone 1 fails
```

Region still exists.

### Regional failure

```text id="r7x3n8"
Entire region unavailable
```

Now you need a DR strategy involving another region.

---

# 14. HA and DR architecture

A simplified enterprise architecture:

```text id="h5q8m1"
             PRIMARY REGION
                  |
             SQL Managed
               Instance
                  |
                HA
                  |
        +---------+---------+
        |                   |
    Zone A               Zone B
     
                  |
                  | DR
                  ↓
          SECONDARY REGION
                  |
             SQL MI / DR
             Architecture
```

This gives you two layers:

```text id="w3n9x6"
Within-region resilience
+
Cross-region recovery
```

---

# 15. Planned failover

A planned failover is controlled.

Example:

```text id="m2r7c4"
Maintenance
    ↓
Prepare
    ↓
Fail over
    ↓
Secondary becomes primary
```

This can be tested during a controlled window.

---

# 16. Unplanned failover

An unplanned failover occurs because something has failed.

Example:

```text id="k8x1p5"
Primary unavailable
      ↓
Failure detected
      ↓
Recovery/failover mechanism
      ↓
New primary
```

There may be some interruption.

Active connections may need to reconnect.

---

# 17. Application connection behavior

This is extremely important.

Suppose SQL fails over.

The application has:

```text id="q4m8x2"
Connection
   ↓
SQL MI
```

After failover, existing connections may be broken.

The application needs appropriate:

```text id="z6p3r9"
Connection retry
Transient fault handling
Timeout handling
Connection pooling behavior
```

---

# 18. Transient failures

Azure applications should be designed to handle transient faults.

A transient fault means:

```text id="v7n2c5"
Temporary connectivity failure
```

Examples:

```text id="j3m8q1"
Failover
Network interruption
Maintenance event
Temporary service issue
```

A well-designed application can retry appropriate operations.

---

# 19. Retry is not simply "run everything again"

This is very important.

Imagine:

```sql id="f5q9m2"
INSERT INTO Orders(...)
VALUES (...);
```

The application sends the command.

The server processes it.

Then the network connection fails before the client receives the response.

Application thinks:

```text id="x7c3n8"
"Transaction failed."
```

But the server might have actually committed it.

If the application blindly retries:

```text id="r4m1p6"
INSERT again
```

you could create a duplicate order.

---

# 20. Idempotency

This is why robust applications use idempotency techniques.

For example:

```text id="p8n2v7"
OrderRequestID = ABC123
```

If the same request arrives twice:

```text id="w5x9m3"
ABC123
ABC123
```

the application/database can recognize that the request was already processed.

This is an application-design issue, not something Azure SQL MI automatically solves for you.

---

# 21. DR scenario — regional outage

Let's simulate a serious production event.

Your primary region is:

```text id="c6r2x8"
East US
```

Your DR region is:

```text id="m9v4q1"
Central US
```

At 3:15 PM:

```text id="n7p3x5"
Primary region outage
```

Application errors begin.

---

# 22. Incident response

### Step 1

Confirm:

```text id="h2m8q6"
Is this a database problem?
Application problem?
Network problem?
Regional Azure problem?
```

Don't fail over just because one application server is unhealthy.

---

# 23. Step 2 — Assess scope

Suppose:

```text id="q5x1r9"
Multiple Azure services unavailable
Multiple workloads affected
```

Now the regional-disaster hypothesis becomes stronger.

---

# 24. Step 3 — Check DR readiness

Determine:

```text id="v8m3n7"
Secondary healthy?
Replication/recovery status?
Last known synchronization point?
Application configuration ready?
DNS/connectivity ready?
```

---

# 25. Step 4 — Understand RPO

Suppose the DR architecture has a potential recovery point of:

```text id="k4r9p2"
3:12 PM
```

Incident:

```text id="x7m1c5"
3:15 PM
```

Potential data gap:

```text id="z3n8q6"
~3 minutes
```

The exact behavior depends on the architecture and failure state, but this illustrates why RPO matters.

---

# 26. Step 5 — Fail over

Once the incident team decides:

```text id="s5p2m8"
Fail over
```

the DR environment becomes the production target.

Then:

```text id="n9x4r7"
Application
    ↓
DR database
```

---

# 27. Step 6 — Validate application

Don't stop at:

> "Failover succeeded."

Test:

```text id="w6m3q1"
Application login
Create order
Read order
Update order
Payment workflow
Reports
Background jobs
Critical APIs
```

This is business validation.

---

# 28. Step 7 — Communicate

A good DBA incident update might say:

> "The primary region is unavailable. The DR environment is healthy and within the defined recovery objectives. Failover has been initiated. We are currently validating application connectivity and critical transaction workflows."

That's much better than:

> "Database failed."

---

# 29. Failback

After the primary region recovers, you don't necessarily immediately move everything back.

You need a controlled process:

```text id="e4x7m2"
DR production
     ↓
Stabilize
     ↓
Synchronize/reconcile
     ↓
Validate primary
     ↓
Planned failback
```

Failback is a change.

Treat it like one.

---

# 30. DR testing

A DR strategy that has never been tested is risky.

Test:

```text id="h8q3n5"
Failover
Application connectivity
Authentication
Transactions
Background jobs
Monitoring
DNS/network dependencies
Failback
```

Record:

```text id="p2m7x9"
Actual RTO
Actual recovery point
Problems
Manual steps
Application issues
```

---

# 31. The "database is available" trap

Suppose the secondary SQL MI is healthy.

But:

```text id="r5n8c2"
Application cannot reach it.
```

Your DR isn't successful.

Or:

```text id="x3q7m1"
SQL works
Authentication fails
```

Still not successful.

Or:

```text id="v9p4k6"
SQL works
But application background workers still point to primary
```

Still not successful.

DR is an **end-to-end system**, not just a database.

---

# 32. DR dependency mapping

A real production application may depend on:

```text id="j6m2r8"
Web/API
   ↓
Application services
   ↓
SQL MI
   ↓
Storage
   ↓
Messaging
   ↓
Identity
   ↓
DNS
   ↓
Monitoring
```

If only SQL moves to another region:

```text id="k8x3p5"
Application
      ↓
Old region
      X
```

the application may still fail.

---

# 33. Connection strings

Your application needs a supported connection strategy that allows it to connect to the correct SQL MI endpoint after failover.

Don't design:

```text id="m3n7q1"
Application
    ↓
Hard-coded primary infrastructure assumption
```

without considering failover.

Use appropriate Azure SQL MI connectivity patterns and application retry/failover behavior.

---

# 34. Planned vs unplanned failover

| Feature               | Planned             | Unplanned                 |
| --------------------- | ------------------- | ------------------------- |
| Trigger               | DBA/change          | Failure                   |
| Preparation           | High                | Limited                   |
| Testing               | Easy                | Real incident             |
| Expected interruption | Controlled          | Potentially unpredictable |
| Goal                  | Maintenance/DR test | Restore service           |

---

# 35. RPO/RTO example

Business says:

```text id="p7m2x8"
RPO = 5 minutes
RTO = 30 minutes
```

This means:

```text id="q9n4c6"
Maximum acceptable data loss
≈ 5 minutes

Maximum acceptable recovery duration
≈ 30 minutes
```

Your architecture and operational process need to be capable of meeting those requirements.

---

# 36. Architecture decision

Suppose management says:

> "We need zero downtime and zero data loss."

Your response shouldn't be:

> "Sure."

You should ask:

```text id="w4x8m1"
What does "zero" mean?
Is synchronous replication required?
What is the budget?
What failure types are included?
Does the application support retries?
What dependencies exist?
What is acceptable during planned maintenance?
```

Business requirements must be translated into technical architecture.

---

# 37. HA does not equal backup

This is worth memorizing:

```text id="c5m9r2"
HA
↓
Availability

Backup
↓
Recovery

DR
↓
Regional disaster recovery
```

You generally need all three concepts.

---

# 38. HA does not equal security

Another misconception:

> "We have HA, so we're protected."

No.

HA doesn't solve:

```text id="j7p3n8"
SQL injection
Bad permissions
Ransomware
Accidental DELETE
Credential compromise
Bad application deployment
```

Those require other controls.

---

# 39. HA does not equal performance

You can have:

```text id="z8q2v6"
Highly available SQL MI
```

with:

```text id="m4r9x1"
Terrible query performance
```

Availability and performance are separate dimensions.

---

# 40. Four pillars

Think of production SQL architecture as:

```text id="s6n3p8"
             SQL PRODUCTION
                   |
      +------------+------------+
      |            |            |
     HA           DR         Backup
      |            |            |
 Availability   Disaster     Logical
                             recovery
                   |
                Security
```

And performance sits across all of them.

---

# 41. Interview question

### "What's the difference between HA and DR?"

Answer:

> "HA is designed to maintain service availability through failures within the supported service architecture, typically within a region. DR addresses larger disasters, such as regional outages, by providing a recovery capability in another region or location. HA primarily focuses on availability; DR focuses on recovery from major disasters."

---

# 42. Interview question

### "What are RPO and RTO?"

Answer:

> "RPO defines the maximum acceptable amount of data loss, while RTO defines the maximum acceptable time to restore service."

Simple.

---

# 43. Interview question

### "Does HA protect against accidental DELETE?"

Answer:

> "No. HA protects availability, not logical correctness. An accidental committed DELETE can be propagated through the availability architecture. Backup and point-in-time recovery are used for logical data recovery."

Excellent answer.

---

# 44. Interview question

### "What do you test during DR?"

Answer:

```text id="u2m8x6"
Database failover
Application connectivity
Authentication
Critical transactions
Background jobs
DNS/network dependencies
Monitoring
RPO
RTO
Failback
```

---

# 45. Senior DBA mental model

When management says:

> **"What happens if Azure goes down?"**

Don't answer:

> "Azure has HA."

Instead think:

```text id="p9r4m2"
Failure
  ↓
What failed?
  ↓
Zone or region?
  ↓
What is our RPO?
  ↓
What is our RTO?
  ↓
Where is the recovery target?
  ↓
How does application reconnect?
  ↓
How do we validate?
  ↓
How do we fail back?
```

That's production thinking.

---

# 46. Complete regional-disaster flow

Memorize this:

```text id="w7x2m9"
REGIONAL FAILURE
       ↓
Detect
       ↓
Confirm scope
       ↓
Declare incident
       ↓
Assess DR readiness
       ↓
Determine recovery point
       ↓
Fail over
       ↓
Redirect/reconnect application
       ↓
Validate critical transactions
       ↓
Monitor
       ↓
Communicate
       ↓
Recover primary
       ↓
Synchronize
       ↓
Controlled failback
       ↓
Post-incident review
```

---

# 47. The most important lesson

A mature SQL DBA doesn't say:

> **"We have a secondary database, therefore we're protected."**

Instead:

> **"Our HA/DR design is aligned with business RPO/RTO requirements, the application supports failover and transient-fault handling, and we regularly test the entire recovery process."**

That's the difference between knowing Azure features and designing a production system.

---

## Next — Lesson 75: Security & Identity for Azure SQL Managed Instance

Next we'll cover **production SQL MI security**, including:

* Microsoft Entra ID authentication
* SQL authentication
* managed identities
* service principals
* least privilege
* server-level vs database-level permissions
* contained users
* RBAC vs SQL permissions
* TDE
* encryption in transit
* Key Vault concepts
* auditing
* Defender for SQL
* private endpoints/network security
* common DBA security mistakes
* and a real scenario where an application suddenly loses database access after a deployment.
