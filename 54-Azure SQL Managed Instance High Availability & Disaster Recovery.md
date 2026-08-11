# Lesson 86 — Azure SQL Managed Instance: High Availability & Disaster Recovery

This is a **core production DBA topic**.

When people say:

> "We have HA."

they may actually be talking about something very different from:

> "We have DR."

You need to understand the difference clearly.

---

# 1. HA vs DR

## High Availability — HA

HA protects you from **component or infrastructure failures** while keeping the service available with minimal interruption.

Example:

```text
Primary SQL MI
     |
     ↓
Failure
     |
     ↓
Automatic failover
     |
     ↓
Service continues
```

The goal is:

> **Keep the database service available.**

---

## Disaster Recovery — DR

DR protects you from a **larger disaster**, such as:

```text
Azure region outage
Major regional incident
Large-scale infrastructure failure
```

Conceptually:

```text
Region A
   |
   X
Regional disaster
   |
   ↓
Region B
   |
   ↓
SQL MI / restored database
```

The goal is:

> **Recover the workload somewhere else.**

---

# 2. Simple mental model

Memorize:

```text
HA = "Keep running"

DR = "Recover elsewhere"
```

---

# 3. RTO

**RTO = Recovery Time Objective**

It answers:

> **How long can the business tolerate the service being unavailable?**

Example:

```text
RTO = 30 minutes
```

Means:

> The business requires recovery within 30 minutes.

---

# 4. RPO

**RPO = Recovery Point Objective**

It answers:

> **How much data loss can the business tolerate?**

Example:

```text
RPO = 5 minutes
```

Means:

> In the worst case, the business can tolerate losing up to approximately 5 minutes of data, depending on the selected architecture and recovery mechanism.

---

# 5. RTO vs RPO

This is an interview favorite.

| Term | Question                     |
| ---- | ---------------------------- |
| RTO  | How quickly must we recover? |
| RPO  | How much data can we lose?   |

Example:

```text
RTO = 30 minutes
RPO = 5 minutes
```

means:

```text
Service recovery ≤ 30 min
Potential data loss ≤ 5 min
```

The exact achievable objectives depend on the architecture and Azure service capabilities.

---

# 6. Why SQL MI has built-in HA

Azure SQL Managed Instance is a managed PaaS database service.

Microsoft manages much of the underlying infrastructure and platform availability.

Conceptually:

```text
             SQL MI
                |
       Managed platform
                |
        +-------+-------+
        |               |
   Compute/storage   HA mechanisms
        |               |
        +-------+-------+
                |
         Automatic recovery
```

This is very different from managing SQL Server on an Azure VM.

---

# 7. SQL Server on VM vs SQL MI

With SQL Server on Azure VM, you may be responsible for designing and managing:

```text
Windows
SQL Server
Failover Cluster
Storage
Patching
HA architecture
```

With SQL MI:

```text
Azure
  |
  +-- Infrastructure management
  +-- Platform HA
  +-- SQL service management
```

Microsoft manages much more of the underlying platform.

---

# 8. But managed doesn't mean "no DR required"

This is a critical concept.

Suppose Azure provides:

```text
Excellent HA
```

That doesn't automatically mean your application is protected from:

```text
Regional disaster
Accidental deletion
Logical corruption
Bad deployment
Human error
```

You still need a business continuity and DR strategy.

---

# 9. HA architecture — conceptual

Think:

```text
              SQL MI
                 |
       +---------+---------+
       |                   |
   Active platform      HA mechanisms
       |                   |
       +---------+---------+
                 |
          Automatic recovery
```

The platform handles infrastructure-level availability mechanisms.

You don't manually build a traditional Windows Failover Cluster around SQL MI.

---

# 10. What happens during an infrastructure failure?

Imagine:

```text
SQL MI
  |
Compute problem
  X
```

The managed platform detects the failure and performs recovery/failover mechanisms as designed by the service.

From the application's perspective:

```text
Connection
   ↓
Temporary interruption
   ↓
Recovery/failover
   ↓
Reconnect
```

This is why application retry logic matters.

---

# 11. Connection interruption

A very important point:

> **HA does not necessarily mean zero interruption.**

During failover/recovery:

```text
Existing connection
      ↓
May be disconnected
      ↓
Application reconnects
```

Therefore applications should handle transient connection failures appropriately.

---

# 12. Retry logic

Good applications typically implement transient fault handling.

Conceptually:

```text
Application
   |
Connect
   |
Failure
   |
Wait
   |
Retry
   |
Retry
   |
Reconnect
```

But retries must be designed carefully.

You don't want:

```text
1,000 application threads
       ↓
retry immediately
       ↓
SQL MI
       ↓
massive connection storm
```

That's called a **retry storm**.

---

# 13. Exponential backoff

A better pattern is:

```text
Attempt 1 → fail
     ↓
wait 1 sec

Attempt 2 → fail
     ↓
wait 2 sec

Attempt 3 → fail
     ↓
wait 4 sec
```

with appropriate limits and jitter.

This reduces the chance of overwhelming the recovering service.

---

# 14. Availability Zones

Azure regions can support **Availability Zones**.

Conceptually:

```text
Azure Region
+----------------------------------+
|                                  |
| Zone 1      Zone 2      Zone 3   |
|   |           |           |      |
|   +-----------+-----------+      |
|               |                  |
|          SQL workload            |
|                                  |
+----------------------------------+
```

Availability Zones are physically separate locations within an Azure region, designed to protect against certain datacenter-level failures.

Availability-zone support and configuration depend on the specific SQL MI deployment options and Azure region.

---

# 15. Zone failure vs region failure

This distinction is important.

### Availability Zone failure

```text
Zone 1
   X
```

Other zones in the region may remain available.

### Region failure

```text
Region A
   X X X
```

Now you need a strategy involving another Azure region.

So:

```text
AZ protection ≠ Regional DR
```

---

# 16. Regional DR

Imagine your production environment:

```text
Region A
   |
   +---- SQL MI
   +---- Application
```

A major regional disaster occurs:

```text
Region A
   X
```

You need:

```text
Region B
   |
   +---- SQL MI / recovery environment
   +---- Application
```

---

# 17. Cross-region architecture

A conceptual enterprise design:

```text
             Users
               |
          Global DNS / Traffic
               |
        +------+------+
        |             |
        ↓             ↓
    Region A       Region B
      Primary        DR
        |             |
      SQL MI        SQL MI
```

Region A is normally active.

Region B is your recovery environment.

---

# 18. How do you get data to Region B?

There are several Azure SQL MI-supported business continuity and disaster recovery approaches.

Depending on requirements, you may use mechanisms such as:

```text
Backup-based recovery
Geo-restore
Failover groups
Managed instance link / supported replication scenarios
```

The correct option depends on:

```text
RPO
RTO
Database size
Workload
Application architecture
Cost
Recovery requirements
```

---

# 19. Backup-based DR

One common model is:

```text
Primary SQL MI
      |
      ↓
Backups
      |
      ↓
Azure storage/service-managed backup infrastructure
      |
      ↓
Secondary region
      |
      ↓
Restore SQL MI/database
```

This can provide regional recovery, but recovery time depends on:

```text
Database size
Backup availability
Restore time
Infrastructure provisioning
Configuration
```

So it may not meet a very aggressive RTO.

---

# 20. Geo-restore

Geo-restore allows recovery from geo-redundant backups into another region, subject to the service's supported capabilities.

Conceptually:

```text
Region A
SQL MI
 |
 | Backups
 ↓
Geo-redundant backup
 |
 ↓
Region B
 |
 ↓
Restore
 |
 ↓
New/recovered SQL MI
```

This is useful for disaster recovery scenarios.

---

# 21. Important limitation of backup-based DR

Suppose:

```text
RPO requirement = 5 minutes
```

and your recovery method only gives you a recovery point much older than that.

Then:

```text
Architecture ❌
```

even though technically you have a DR solution.

DR must be designed from business requirements.

---

# 22. Failover groups

SQL Managed Instance supports **failover groups** for supported cross-region scenarios.

Conceptually:

```text
Region A
SQL MI
Primary
   |
   | replication
   ↓
Region B
SQL MI
Secondary
```

During a regional disaster:

```text
Primary
   X
   ↓
Failover
   ↓
Secondary
```

The application can use a listener endpoint designed for failover scenarios rather than hard-coding a specific primary server.

---

# 23. Why listener endpoints matter

Bad design:

```text
Application
    |
    ↓
primary-mi-specific-hostname
```

If primary changes:

```text
Application
    X
```

Better:

```text
Application
    |
    ↓
Failover-aware listener
    |
    ↓
Current primary
```

The application doesn't need to know which region is currently primary.

---

# 24. Application architecture

A DR design isn't complete if only the database is protected.

Suppose:

```text
Region A
   |
   +-- SQL MI
   +-- Application
```

and Region A fails.

If Region B only contains:

```text
SQL MI
```

but not the application infrastructure:

```text
Region B
   |
   +-- SQL MI
```

then users still can't use the system.

You need to consider:

```text
Application
Database
Networking
DNS
Identity
Secrets
Certificates
Storage
Monitoring
```

---

# 25. Full DR architecture

A more complete design:

```text
                 Users
                   |
            Global routing
                   |
          +--------+--------+
          |                 |
       Region A          Region B
       Primary             DR
          |                 |
      Application        Application
          |                 |
        SQL MI            SQL MI
          |                 |
          +------Data-------+
```

And both regions need:

```text
VNet
DNS
Security
Identity
Monitoring
Secrets
```

---

# 26. Scenario: region failure

Imagine:

```text
Friday 2:00 PM
```

Primary region:

```text
Healthy
```

At:

```text
2:15 PM
```

major regional outage.

Users cannot connect.

Your DR plan says:

```text
Fail over to Region B
```

The team executes:

```text
1. Confirm incident
2. Declare disaster
3. Validate secondary
4. Fail over database
5. Redirect application
6. Validate application
7. Monitor
```

---

# 27. RTO calculation

Suppose:

```text
2:15 PM → outage
2:20 PM → DR declared
2:28 PM → database available
2:35 PM → application available
```

Recovery time:

```text
20 minutes
```

If requirement:

```text
RTO = 30 minutes
```

then:

```text
20 < 30
```

You met the RTO.

---

# 28. RPO calculation

Suppose last recoverable data point is:

```text
2:11 PM
```

Outage:

```text
2:15 PM
```

Potential data gap:

```text
4 minutes
```

If:

```text
RPO = 5 minutes
```

then:

```text
4 < 5
```

The recovery point meets the target.

Again, actual RPO depends on the specific replication/backup architecture and timing.

---

# 29. Interview scenario

Question:

> "Our business requires RTO 30 minutes and RPO 5 minutes. How would you design SQL MI DR?"

Don't immediately answer:

> "Use backups."

Instead ask:

```text
How large is the database?
How much transaction volume?
What is the expected failover frequency?
Is cross-region replication required?
Can the application run in the secondary region?
How quickly can infrastructure be provisioned?
How will DNS fail over?
How will secrets/authentication work?
How will we test DR?
```

Then choose the architecture.

---

# 30. RTO depends on more than database recovery

Suppose database failover takes:

```text
5 minutes
```

but application startup takes:

```text
25 minutes
```

Total:

```text
30 minutes
```

If your RTO is:

```text
30 minutes
```

you have no margin.

Therefore:

```text
RTO =
DB recovery
+
Application recovery
+
Networking
+
Validation
```

---

# 31. DR runbook

A production DR plan should be documented.

Example:

```text
DR RUNBOOK

1. Detect regional outage
2. Confirm scope
3. Declare disaster
4. Freeze deployments
5. Validate secondary SQL MI
6. Validate network
7. Validate identity
8. Fail over database
9. Update application configuration/routing
10. Start/activate application
11. Run smoke tests
12. Validate critical transactions
13. Monitor
14. Communicate recovery
```

---

# 32. DR testing

This is extremely important.

A DR plan that has never been tested is only a theory.

You should perform:

```text
DR drills
Failover tests
Application validation
Connection tests
DNS tests
Recovery timing
Data validation
```

Measure:

```text
Actual RTO
Actual RPO
```

---

# 33. Example DR test

Business requirement:

```text
RTO = 30 min
RPO = 5 min
```

Test result:

```text
Database recovery = 8 min
Application recovery = 17 min
DNS/routing = 5 min
```

Total:

```text
30 min
```

You barely meet the requirement.

That's valuable information.

---

# 34. DR test failure

Suppose actual result:

```text
Recovery = 47 min
```

Requirement:

```text
30 min
```

Then:

```text
DR plan ❌
```

You investigate:

```text
Why did it take 47 minutes?
```

Maybe:

```text
Manual provisioning
Manual DNS changes
Missing secrets
Slow restore
Application startup
```

Now improve the runbook/architecture.

---

# 35. Human error scenario

DR isn't only for regional disasters.

Suppose DBA accidentally executes:

```sql id="t0tzzx"
DELETE FROM Orders;
```

without a proper WHERE clause.

This is a **logical data loss** scenario.

HA doesn't necessarily protect you.

Why?

Because the bad change can potentially propagate through HA/replication mechanisms.

You need:

```text id="j2y6is"
Point-in-time recovery
Backup
Restore
Data recovery procedures
```

This is a critical HA vs backup distinction.

---

# 36. HA doesn't protect against bad SQL

Suppose:

```text id="hsp0m5"
DELETE 10 million rows
```

The database remains:

```text id="xjtr4b"
Healthy
```

HA works perfectly.

But your data is wrong.

Therefore:

```text id="3t6d0u"
HA ≠ protection from logical mistakes
```

---

# 37. Backup strategy

For production SQL MI, you should understand:

```text id="i8x65p"
Automated backups
Point-in-time restore
Long-term retention where configured/supported
Geo-redundancy
Restore procedures
```

The exact retention and redundancy capabilities depend on the SQL MI configuration and Azure offerings in use.

---

# 38. Point-in-time restore

Suppose:

```text id="9l9p5q"
11:00 AM → normal
11:15 AM → accidental DELETE
11:20 AM → discovered
```

You may need to restore to:

```text id="y9n83l"
11:14 AM
```

instead of restoring the database to its latest state.

That's the concept of point-in-time recovery.

---

# 39. PITR vs HA

### HA

```text id="9r5g2s"
Infrastructure failure
        ↓
Automatic recovery/failover
```

### PITR

```text id="m2f9uk"
Logical mistake
       ↓
Restore earlier point
```

Different problems.

---

# 40. Geo-restore vs failover

Broadly:

### Geo-restore

```text id="a7m0uj"
Backup
 ↓
Restore in another region
```

Potentially slower, depending on database size and recovery architecture.

### Failover group

```text id="6l9w1q"
Primary
   ↓
Secondary
   ↓
Failover
```

Designed for faster business continuity in supported scenarios.

The right choice depends on your RTO/RPO requirements.

---

# 41. Cost consideration

DR isn't free.

You may need:

```text id="o1f7yd"
Secondary SQL MI
Application infrastructure
Networking
Monitoring
Storage
Data transfer
Operational support
```

You need to balance:

```text id="k4id2m"
Business risk
      ↕
Recovery requirements
      ↕
Cost
```

---

# 42. Active-active vs active-passive

### Active-passive

```text id="0z4gtd"
Region A
ACTIVE

Region B
STANDBY
```

Simpler for many DR designs.

### Active-active

```text id="5bgpwp"
Region A
ACTIVE

Region B
ACTIVE
```

Both regions serve traffic.

This can improve utilization but dramatically increases architectural complexity.

For databases, multi-region write architectures require careful consideration of consistency, conflict handling, application behavior, and supported Azure capabilities.

---

# 43. DNS failover

Suppose:

```text id="b2y4xj"
app.company.com
```

normally points to:

```text id="b9lqkz"
Region A
```

During disaster:

```text id="b9lqkz"
Region A
   X
```

Traffic must move to:

```text id="l6j1pd"
Region B
```

DNS/global routing is therefore part of the DR design.

---

# 44. DNS TTL

Suppose DNS TTL is very long.

Clients may continue using stale information.

That can delay failover.

Therefore, DNS strategy must be considered when designing RTO.

---

# 45. Secrets and identity

Imagine Region B has:

```text id="7o8o1c"
SQL MI ✓
Application ✓
```

but the application cannot authenticate because:

```text id="k3b6x1"
Secret missing
Managed identity configuration incomplete
Certificate missing
```

Your DR test fails.

Therefore:

```text id="1y4v2p"
Identity
Secrets
Certificates
```

are part of DR.

---

# 46. Network DR

You also need:

```text id="n7a2wq"
VNet
Subnets
Routes
Firewall
DNS
VPN
ExpressRoute
NSGs
```

A database can be healthy in Region B and still be unreachable.

---

# 47. Full enterprise DR diagram

```text id="0h2yq7"
                         USERS
                           |
                    Global Routing
                           |
             +-------------+-------------+
             |                           |
             ↓                           ↓
        REGION A                    REGION B
        PRIMARY                     DR/SECONDARY
             |                           |
      +------+-------+             +-----+------+
      |              |             |            |
 Application      SQL MI      Application    SQL MI
      |              |             |            |
      +------+-------+             +-----+------+
             |                           |
             +------ Replication --------+
             
             Both regions:
             - VNet
             - DNS
             - Identity
             - Security
             - Monitoring
             - Secrets
```

---

# 48. Planned vs unplanned failover

### Planned failover

You intentionally move workload.

Example:

```text
Regional maintenance
DR testing
Migration
```

### Unplanned failover

Something unexpected happens.

Example:

```text
Regional outage
Infrastructure failure
```

The operational process is different.

---

# 49. Failback

After Region A recovers:

```text
Region A
   |
   ↓
Healthy again
```

Do you immediately move production back?

Not necessarily.

You need to consider:

```text
Data synchronization
Application readiness
Business impact
Failback risk
Maintenance window
```

Failback should be planned.

---

# 50. DR lifecycle

Think:

```text id="h4pql4"
Design
  ↓
Build
  ↓
Test
  ↓
Measure
  ↓
Improve
  ↓
Retest
```

Not:

```text
Build once
  ↓
Forget
```

---

# 51. Interview question

### "What is RPO?"

Answer:

> "Recovery Point Objective defines the maximum acceptable amount of data loss, measured in time."

Example:

```text
RPO = 5 minutes
```

means the business can tolerate approximately five minutes of data loss under the defined recovery scenario.

---

# 52. Interview question

### "What is RTO?"

Answer:

> "Recovery Time Objective defines the maximum acceptable time to restore the service after a disaster."

Example:

```text
RTO = 30 minutes
```

means recovery should complete within 30 minutes.

---

# 53. Interview question

### "Does HA protect against accidental DELETE?"

Answer:

> "No. HA protects availability, not necessarily logical correctness. An accidental DELETE can be replicated through the HA mechanism. Backups and point-in-time recovery are needed for logical recovery."

Excellent interview answer.

---

# 54. Interview question

### "Does Availability Zone protection equal DR?"

Answer:

> "No. Availability Zones protect against certain datacenter-level failures within a region. Regional DR protects against a broader regional outage and normally requires a recovery strategy in another region."

---

# 55. Interview question

### "Why does application retry logic matter?"

Answer:

> "Database failover can cause existing connections to be interrupted. Proper transient fault handling and retry logic allow applications to reconnect automatically after temporary failures."

---

# 56. Interview question

### "Why isn't a database-only DR plan enough?"

Answer:

> "Because the application also depends on networking, DNS, identity, secrets, certificates, application infrastructure, and monitoring. If only the database is recovered, users may still be unable to access the application."

---

# 57. Interview scenario — RTO 30, RPO 5

Question:

> Your company requires RTO 30 minutes and RPO 5 minutes. The primary SQL MI is in East US. Design DR.

A strong response:

```text
1. Identify supported cross-region SQL MI DR architecture.
2. Establish secondary-region infrastructure.
3. Ensure required data synchronization/backup strategy meets the RPO.
4. Deploy application and networking dependencies in the DR region.
5. Configure failover-aware application connectivity.
6. Automate DNS/global routing changes where appropriate.
7. Configure identity, secrets, certificates, and monitoring.
8. Document failover and failback runbooks.
9. Perform regular DR drills.
10. Measure actual RTO/RPO and improve the architecture.
```

Then validate that the chosen SQL MI feature set actually meets:

```text
RPO ≤ 5 minutes
RTO ≤ 30 minutes
```

Don't promise an RPO/RTO before validating the specific architecture.

---

# 58. The most important distinction

Memorize this table:

| Problem                        | Protection                    |
| ------------------------------ | ----------------------------- |
| Compute/infrastructure failure | HA                            |
| Certain zone-level failures    | Zone-aware HA where supported |
| Regional outage                | DR                            |
| Accidental DELETE              | PITR/backup                   |
| Corrupted data                 | Backup/recovery strategy      |
| Application bug                | Deployment/recovery strategy  |
| Credential failure             | Identity/secrets management   |
| Network outage                 | Network redundancy            |
| Application failure            | Application HA/DR             |

---

# 59. Complete DBA mental model

Your production architecture should look conceptually like:

```text id="9p1yq0"
                    BUSINESS
                       |
              RTO / RPO requirements
                       |
                       ↓
              DR ARCHITECTURE
                       |
       +---------------+---------------+
       |                               |
      HA                              DR
       |                               |
   Same region                    Other region
       |                               |
       ↓                               ↓
    SQL MI                         SQL MI
       |                               |
       +---------------+---------------+
                       |
                 APPLICATION
                       |
                 NETWORK/DNS
                       |
                 IDENTITY/SECRETS
                       |
                   MONITORING
                       |
                  DR TESTING
```

---

# 60. Lesson 86 checklist

```text
HA
☐ Understand managed HA
☐ Understand failover
☐ Understand transient connections
☐ Understand retry logic

DR
☐ RTO
☐ RPO
☐ Cross-region architecture
☐ Failover groups
☐ Geo-restore
☐ Backup-based recovery
☐ Point-in-time restore

NETWORK
☐ DNS
☐ VNet
☐ Firewall
☐ Routing
☐ Identity
☐ Secrets

OPERATIONS
☐ DR runbook
☐ Failover
☐ Failback
☐ DR testing
☐ Measure actual RTO/RPO
```

---

## The one sentence to remember

> **HA keeps the service available when infrastructure fails; DR gets the business running when a larger disaster takes the primary environment away.**

And one more:

> **Backups protect your data from mistakes; HA primarily protects service availability.**

---

## Next — Lesson 87: Azure SQL Managed Instance Backup, Restore & Point-in-Time Recovery

We'll go deep into:

* Full backups
* Differential backups
* Transaction log backups
* Automated backups
* Point-in-time restore
* Retention
* Long-term retention
* Geo-redundant backups
* Restore to another region
* Accidental DELETE recovery
* Database corruption scenarios
* Recovery planning
* RPO calculation
* Backup vs HA vs DR
* What a DBA actually checks during a restore
* **Real production scenario: "A developer accidentally deleted 2 million customer records at 11:42 AM. At 11:50 AM the DBA is alerted. Walk through exactly how you recover the data without unnecessarily losing other transactions."**
