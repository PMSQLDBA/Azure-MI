# Lesson 90 — Azure SQL Managed Instance: High Availability & Disaster Recovery

This is one of the **most important topics for a SQL Server DBA** moving to Azure.

The first thing to understand is:

> **High Availability (HA) and Disaster Recovery (DR) are not the same thing.**

---

# 1. HA vs DR

### High Availability

HA answers:

> **"What happens if something fails inside my normal production environment?"**

Examples:

```text
SQL instance failure
Host failure
Hardware failure
Availability-zone issue
```

Goal:

```text
Keep the application running
with minimal interruption.
```

---

### Disaster Recovery

DR answers:

> **"What happens if the entire production environment or Azure region becomes unavailable?"**

Examples:

```text
Regional outage
Major infrastructure failure
Large-scale disaster
Business continuity event
```

Goal:

```text
Recover the application
in another location.
```

---

# 2. Simple picture

```text
                 HA
                  |
        +---------+---------+
        |                   |
   Same region          Same workload
        |
        ↓
   Fast recovery


                 DR
                  |
        +---------+---------+
        |                   |
 Primary region       Secondary region
        |                   |
        ↓                   ↓
    Production          Recovery
```

---

# 3. SQL MI HA architecture

SQL Managed Instance is a managed PaaS service.

Azure handles many of the underlying infrastructure responsibilities required to provide high availability.

You don't manage SQL Server Failover Cluster Instances the same way you would with traditional Windows Server SQL Server.

Think:

```text
Traditional SQL Server
        |
        ↓
You manage cluster
You manage nodes
You manage OS
You manage SQL installation


SQL Managed Instance
        |
        ↓
Azure manages infrastructure
        |
        ↓
You manage databases/workloads/configuration
```

---

# 4. What does HA protect against?

Imagine:

```text
Application
     |
     ↓
SQL MI
     |
     ↓
Underlying infrastructure
```

If an underlying component fails, Azure can perform recovery/failover mechanisms without you manually rebuilding the SQL Server.

The objective is:

```text
Failure
  ↓
Detection
  ↓
Recovery/failover
  ↓
Application reconnects
```

---

# 5. Important DBA concept

Do **not** think:

> "HA means zero downtime."

HA means:

> **The service is designed to remain available or recover quickly from failures.**

There can still be:

```text
Connection interruption
Failover
Transient errors
Active transaction rollback
Application retry requirements
```

---

# 6. Application retry is important

Suppose:

```text
Application
     |
     ↓
SQL MI
```

A failover occurs.

The existing TCP connection may break.

Application sees:

```text
Connection error
```

A resilient application should:

```text
Detect transient failure
       ↓
Wait
       ↓
Retry
       ↓
Reconnect
```

---

# 7. Exponential backoff

Don't do:

```text
Retry
Retry
Retry
Retry
Retry
Retry
```

immediately.

That can make an outage worse.

Instead:

```text
Attempt 1
   ↓
Wait
   ↓
Attempt 2
   ↓
Longer wait
   ↓
Attempt 3
```

This is called **exponential backoff** when the delay increases progressively.

---

# 8. Real-time scenario

Your e-commerce application receives:

```text
10,000 requests/minute
```

SQL MI experiences a failover.

Without retry:

```text
SQL connection fails
      ↓
Application errors
      ↓
Customers see failures
```

With resilient retry logic:

```text
SQL connection fails
      ↓
Transient error detected
      ↓
Retry
      ↓
New connection
      ↓
Request succeeds
```

The application may experience a short interruption, but recovery is much smoother.

---

# 9. What happens to an active transaction?

Suppose:

```sql
BEGIN TRANSACTION;

UPDATE Orders
SET Status = 'Completed'
WHERE OrderID = 1001;
```

Then a failover occurs before the transaction completes.

The transaction may be rolled back.

The application should be designed to safely retry operations when appropriate.

This is why **idempotency** is important.

---

# 10. Idempotency

Imagine the application sends:

```text
Create Order #1001
```

The connection breaks.

The application doesn't know:

> "Did SQL Server receive the request?"

If it blindly retries:

```text
Create Order #1001
Create Order #1001
```

you might create duplicates.

A better design uses an idempotency key/order ID or appropriate transaction/application logic.

---

# 11. HA is not backup

This is extremely important.

Suppose your SQL MI has excellent HA.

Someone accidentally executes:

```sql
DELETE FROM Customers;
```

What happens?

HA does **not** magically restore the deleted data.

The deletion may be replicated to the HA infrastructure because it is a legitimate database operation.

Therefore:

```text
HA ≠ Backup
```

---

# 12. HA vs backup

| Feature | Purpose                                               |
| ------- | ----------------------------------------------------- |
| HA      | Keep service available during infrastructure failures |
| Backup  | Recover data                                          |
| DR      | Recover service after major disaster                  |

You need all three concepts.

---

# 13. Example

Scenario:

```text
Database accidentally deleted
```

Use:

```text
Backup / point-in-time restore
```

Scenario:

```text
Underlying infrastructure failure
```

Use:

```text
HA
```

Scenario:

```text
Azure region unavailable
```

Use:

```text
DR strategy
```

---

# 14. RPO

RPO means:

> **Recovery Point Objective**

It answers:

> **"How much data can we afford to lose?"**

Example:

```text
RPO = 5 minutes
```

Means the business accepts potentially losing up to about five minutes of data, depending on the chosen architecture and actual recovery behavior.

---

# 15. RTO

RTO means:

> **Recovery Time Objective**

It answers:

> **"How quickly must the system be restored?"**

Example:

```text
RTO = 30 minutes
```

Business says:

> "The order system must be operational within 30 minutes."

---

# 16. RPO vs RTO

Memorize:

```text
RPO = How much data can we lose?

RTO = How much time can we be down?
```

---

# 17. Business example

Company says:

> "Our banking application cannot lose more than 1 minute of transactions and must be operational within 5 minutes."

Therefore:

```text
RPO ≤ 1 minute
RTO ≤ 5 minutes
```

Now you design the DR architecture around those requirements.

Don't start by selecting a technology first.

Start with:

```text
Business requirement
       ↓
RPO / RTO
       ↓
Architecture
```

---

# 18. Region failure

Imagine your primary SQL MI is in:

```text
East US
```

And a major regional outage occurs.

Your architecture could have:

```text
East US
   |
   ↓
Primary SQL MI


West US
   |
   ↓
Secondary SQL MI
```

Conceptually:

```text
          Application
               |
        +------+------+
        |             |
     Primary       Secondary
     Region         Region
        |             |
      SQL MI         SQL MI
```

---

# 19. Why have a secondary region?

Because HA inside one region doesn't protect you from every regional disaster.

For example:

```text
Primary region
      |
      X
Major regional outage
```

You need a recovery architecture elsewhere.

---

# 20. DR architecture

A simplified architecture:

```text
                 Global Application
                        |
             +----------+----------+
             |                     |
             ↓                     ↓
       Primary Region       Secondary Region
             |                     |
          SQL MI                SQL MI
             |
          Primary
             |
        Data replication
             |
             ↓
          Secondary
```

The exact SQL MI DR mechanism depends on the architecture and Azure features selected.

---

# 21. Failover

Failover means:

> Move the workload from the primary to the secondary.

Normal:

```text
Application
     |
     ↓
Primary SQL MI
```

After failover:

```text
Application
     |
     ↓
Secondary SQL MI
```

---

# 22. Planned failover

A planned failover is intentional.

Example:

```text
Scheduled Azure maintenance
```

You know it is coming.

You coordinate:

```text
DBA
Application team
Network team
Business team
```

Then perform a controlled failover according to the DR design.

---

# 23. Unplanned failover

Example:

```text
Primary region outage
```

You don't have the luxury of a scheduled maintenance window.

The DR mechanism must react according to the configured architecture.

This is why DR must be tested **before** a disaster.

---

# 24. Manual vs automatic failover

Depending on the selected architecture, failover can be:

```text
Automatic
```

or:

```text
Manual/controlled
```

The choice depends on business requirements.

Automatic sounds attractive, but uncontrolled automatic failover can sometimes have business consequences.

---

# 25. Why automatic failover isn't always ideal

Suppose the application has a temporary network issue.

If your architecture incorrectly interprets that as:

> "Primary database is dead."

it could trigger an unnecessary failover.

This is why failover policies need careful design.

---

# 26. Failover Groups

For supported Azure SQL architectures, **failover groups** can provide a logical mechanism for managing relationships between primary and secondary databases/instances and application connectivity.

The important idea is:

```text
Application
     |
     ↓
Stable failover endpoint
     |
     +------ Primary
     |
     +------ Secondary
```

The application shouldn't need to be manually reconfigured every time the primary/secondary role changes.

---

# 27. Why endpoint abstraction matters

Bad architecture:

```text
Application
   |
   ↓
Hard-coded primary server
```

During disaster:

```text
Primary unavailable
      ↓
Application still points there
      ↓
Failure
```

Better:

```text
Application
     |
     ↓
Logical failover endpoint
     |
     ↓
Current primary
```

The abstraction makes disaster recovery easier.

---

# 28. DNS during DR

Suppose:

```text
Primary:
mi-east.example...

Secondary:
mi-west.example...
```

If the application is hard-coded to the primary hostname:

```text
Application
    |
    ↓
Primary
    X
```

After failover, you want the application to reach the correct active endpoint.

This is one reason application connection architecture matters as much as database replication.

---

# 29. DR isn't just database replication

A common mistake is:

> "We have a secondary database, therefore DR is complete."

No.

A complete DR architecture includes:

```text
Database
Application
Networking
DNS
Identity
Secrets
Certificates
Monitoring
Deployment
```

---

# 30. Example: complete disaster

Primary Azure region fails.

What does the application need?

```text
Application servers
     ↓
Networking
     ↓
DNS
     ↓
Identity
     ↓
SQL MI
     ↓
Secrets/configuration
```

If only the SQL database exists in the secondary region but your application can't reach it, DR still fails.

---

# 31. DR runbook

A production DR runbook might say:

```text
1. Detect regional incident
2. Confirm primary is unavailable
3. Declare DR event
4. Notify stakeholders
5. Validate secondary environment
6. Initiate/approve failover
7. Validate SQL MI
8. Validate DNS/endpoint
9. Validate application connectivity
10. Run smoke tests
11. Validate transactions
12. Notify business
13. Monitor closely
```

---

# 32. DR smoke test

After failover, don't just say:

> "SQL MI is online."

Test:

```text
SELECT
INSERT
UPDATE
Application login
Order creation
Payment workflow
Reporting
Critical stored procedures
```

The business cares about **application functionality**, not merely database status.

---

# 33. Real scenario — e-commerce

Imagine:

```text
Online Store
     |
     ↓
Order API
     |
     ↓
SQL MI
```

Business requirement:

```text
RPO = near-zero/very low
RTO = 15 minutes
```

You design:

```text
Primary region
     |
     ↓
SQL MI
     |
     ↓
DR architecture
     |
     ↓
Secondary region
```

Then test:

```text
Primary unavailable
       ↓
Failover
       ↓
Application reconnects
       ↓
Orders work
```

---

# 34. Scenario — reporting system

Reporting has different requirements.

Business says:

> "Reports can be unavailable for 2 hours."

Then:

```text
RTO = 2 hours
```

You may choose a less aggressive DR architecture than for a payment database.

This is why:

> **DR design must follow business criticality.**

---

# 35. Tier applications

Example:

```text
Tier 1:
Payments
Orders
Authentication

Tier 2:
Customer service

Tier 3:
Analytics
Historical reporting
```

Each tier can have different:

```text
RPO
RTO
DR architecture
```

---

# 36. Backup strategy

Even with HA and DR, maintain backups.

Why?

Because you need protection from:

```text
Accidental DELETE
Bad deployment
Data corruption
Application bugs
Malicious activity
Logical errors
```

Replication can replicate bad changes.

Backups provide a different recovery path.

---

# 37. Point-in-time restore

Imagine:

```text
10:00 — Normal
10:15 — Bad deployment
10:20 — Data corruption discovered
```

You may want:

```text
Restore database
to a point before the bad deployment
```

This is a backup/recovery use case.

Not an HA failover problem.

---

# 38. HA + Backup + DR

Think of them as three safety nets:

```text
                 SQL MI Protection
                       |
        +--------------+--------------+
        |              |              |
        ↓              ↓              ↓
       HA            Backup           DR
        |              |              |
   Availability     Data recovery   Regional recovery
```

---

# 39. Zone redundancy

Azure services can use availability zones where supported.

Conceptually:

```text
Region
+----------------------------------+
|                                  |
| Zone 1      Zone 2      Zone 3   |
|   |           |           |      |
|   +-----------+-----------+      |
|             SQL service           |
|                                  |
+----------------------------------+
```

The purpose is to reduce impact from a failure affecting one availability zone.

Whether and how zone redundancy applies depends on the specific SQL MI configuration/tier/region capabilities.

---

# 40. Zone failure vs region failure

Don't confuse:

```text
Availability Zone failure
```

with:

```text
Entire Azure region failure
```

A zone-redundant design can improve resilience within a region.

It is not automatically a complete cross-region DR solution.

---

# 41. Real-world failure matrix

| Failure                              | Typical protection                          |
| ------------------------------------ | ------------------------------------------- |
| SQL infrastructure component failure | HA                                          |
| Host/infrastructure issue            | HA                                          |
| Availability-zone disruption         | Zone-resilient architecture where supported |
| Accidental DELETE                    | Backup/PITR                                 |
| Bad deployment                       | Backup/PITR                                 |
| Regional outage                      | Cross-region DR                             |
| Application bug                      | Backup + application recovery               |
| Credential compromise                | Security controls                           |
| Human error                          | Backup + governance                         |

---

# 42. Failback

After the primary region recovers:

```text
Secondary
   |
   ↓
Current production
```

Do you immediately switch back?

Not necessarily.

A controlled failback might involve:

```text
Validate primary
      ↓
Synchronize/validate data
      ↓
Validate application
      ↓
Schedule maintenance
      ↓
Fail back
      ↓
Smoke test
```

---

# 43. Why failback matters

Many organizations test:

```text
Primary → Secondary
```

but never test:

```text
Secondary → Primary
```

Then disaster recovery works, but they don't know how to safely return to normal operations.

That's a major operational gap.

---

# 44. DR testing

A DR plan that has never been tested is only a document.

Test:

```text
DNS
Connectivity
Authentication
Database
Application
Failover
Failback
Monitoring
```

---

# 45. DR exercise

Imagine a quarterly DR drill:

```text
Saturday 01:00
       |
       ↓
Simulate primary outage
       |
       ↓
Failover
       |
       ↓
Application validation
       |
       ↓
Business validation
       |
       ↓
Failback
       |
       ↓
Postmortem
```

Record:

```text
Actual RTO
Actual RPO
Failures
Unexpected dependencies
Manual steps
```

---

# 46. RTO calculation example

Business requirement:

```text
Application must recover within 20 minutes.
```

Breakdown:

```text
Detection             3 min
Decision               2 min
Failover               5 min
DNS/connection         2 min
Application restart    3 min
Validation             5 min
---------------------------
Total                 20 min
```

Now you can see why every component matters.

---

# 47. RPO calculation example

Suppose the architecture can potentially have:

```text
Replication lag = 30 seconds
```

Business requirement:

```text
RPO ≤ 5 minutes
```

Then the architecture may satisfy the business requirement, subject to the actual service behavior and documented guarantees.

---

# 48. Don't promise "zero data loss"

A common interview mistake:

> "Geo-replication gives zero data loss."

Don't make blanket statements.

Actual RPO depends on:

```text
Replication architecture
Service behavior
Commit/replication semantics
Network conditions
Failover timing
```

Always relate your answer to the documented SLA/architecture.

---

# 49. Application design matters

Database DR is only part of the story.

Application should support:

```text
Retry
Timeout
Connection recovery
Idempotency
Failover endpoint
Transaction handling
```

Otherwise even a well-designed SQL MI DR architecture may not produce a seamless application recovery.

---

# 50. Example — payment processing

Suppose:

```text
Payment request
      |
      ↓
Application
      |
      ↓
SQL MI
```

Database connection breaks after payment processing begins.

Application retries.

If the application isn't idempotent, it could accidentally process the same payment twice.

So DR design must include:

```text
Database resilience
+
Application resilience
```

---

# 51. Interview question

### "What is RPO?"

Answer:

> "Recovery Point Objective is the maximum acceptable amount of data loss measured in time."

Example:

```text
RPO = 10 minutes
```

means the business is willing to potentially lose up to approximately 10 minutes of data, depending on the recovery architecture.

---

# 52. Interview question

### "What is RTO?"

Answer:

> "Recovery Time Objective is the maximum acceptable time required to restore the service after an outage."

Example:

```text
RTO = 30 minutes
```

means the service should be recovered within 30 minutes.

---

# 53. Interview question

### "Is HA the same as DR?"

Strong answer:

> "No. HA primarily addresses failures within the normal service environment and aims for rapid recovery or continuity. DR addresses larger disasters such as regional outages and generally involves recovery in another location."

---

# 54. Interview question

### "Does HA replace backups?"

Answer:

> "No. HA is designed for availability, while backups provide data recovery from logical problems such as accidental deletion, corruption, or bad changes."

---

# 55. Interview question

### "What happens to an application connection during failover?"

Answer:

> "Existing connections can be interrupted. A resilient application should handle transient failures and reconnect using appropriate retry logic and the correct database endpoint."

---

# 56. Interview question

### "Why should applications use a failover-aware endpoint?"

Because:

```text
Primary role
     ↓
can change
     ↓
Secondary becomes primary
```

The application shouldn't require a manual configuration change every time database roles change.

---

# 57. Interview question

### "How would you design SQL MI DR?"

A strong answer:

```text
1. Understand business RPO/RTO.
2. Identify critical databases/applications.
3. Select an appropriate cross-region architecture.
4. Ensure network connectivity to the secondary.
5. Plan DNS/endpoint behavior.
6. Configure authentication/identity.
7. Maintain backups independently.
8. Make the application retry/failover aware.
9. Create a DR runbook.
10. Test failover and failback regularly.
```

---

# 58. Production architecture

A mature design might look like:

```text
                    USERS
                      |
                      ↓
              Global Application
                      |
           +----------+----------+
           |                     |
           ↓                     ↓
      PRIMARY REGION       DR REGION
           |                     |
      App Services           App Services
           |                     |
           ↓                     ↓
        SQL MI  <==========>  SQL MI
           |                     |
           +---------+-----------+
                     |
                  Backups
                     |
              Recovery storage
```

The exact replication mechanism and endpoint design depend on your SQL MI architecture.

---

# 59. HA/DR checklist

```text
HA
☐ Understand managed HA architecture
☐ Understand failover behavior
☐ Application retry
☐ Transaction handling
☐ Zone resilience where appropriate

BACKUP
☐ Automated backups
☐ PITR
☐ Long-term retention where required
☐ Restore testing

DR
☐ Secondary region
☐ RPO defined
☐ RTO defined
☐ Replication architecture
☐ Failover procedure
☐ Failback procedure
☐ DNS
☐ Networking
☐ Identity

TESTING
☐ DR drill
☐ Application smoke test
☐ Actual RTO measured
☐ Actual RPO measured
☐ Runbook updated
```

---

# 60. Final real-world scenario

Your company runs:

```text
E-commerce platform
```

Architecture:

```text
Users
  |
  ↓
Application
  |
  ↓
SQL MI - East US
```

Business requirements:

```text
RPO = 5 minutes
RTO = 30 minutes
```

Your DR plan:

```text
                PRIMARY
                East US
                   |
                SQL MI
                   |
                   ↓
             DR architecture
                   |
                   ↓
               West US
                   |
                SQL MI
```

But you don't stop there.

You also ensure:

```text
Secondary networking
Secondary application environment
DNS/endpoint strategy
Identity
Secrets
Monitoring
Backups
Runbook
```

Then you perform regular DR exercises.

---

# The most important mental model

When you hear:

> **"Make SQL MI highly available."**

Think:

```text
HA
 ↓
Infrastructure resilience
 ↓
Failover
 ↓
Application reconnect
```

When you hear:

> **"Protect us from a regional disaster."**

Think:

```text
DR
 ↓
Secondary region
 ↓
Replication/recovery
 ↓
Failover
 ↓
Application redirection
```

When you hear:

> **"Someone deleted our data."**

Think:

```text
Backup
 ↓
Point-in-time recovery
```

And when you hear:

> **"Our application must survive database failover."**

Think:

```text
SQL resilience
+
Retry logic
+
Idempotency
+
Correct endpoint
+
DR testing
```

---

## Next — Lesson 91: Azure SQL Managed Instance Backup & Restore

We'll cover **automated backups, full/differential/log concepts, point-in-time restore, long-term retention, restore scenarios, accidental deletion, corruption, migration-related backups, backup/restore differences between SQL Server and SQL MI, RPO implications, and a complete production restore exercise.**
