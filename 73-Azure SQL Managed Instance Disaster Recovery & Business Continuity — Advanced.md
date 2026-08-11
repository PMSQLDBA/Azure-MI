# Lesson 105 — Azure SQL Managed Instance Disaster Recovery & Business Continuity — Advanced

This lesson is about designing SQL MI so that a **failure does not automatically become a business disaster**.

The core principle:

> **High availability keeps a service running through many failures. Disaster recovery gets the business running again when the original environment or region cannot be relied upon.**

---

# 1. HA vs DR

This distinction is fundamental.

### High Availability

Protects against failures such as:

```text id="ha8k21"
Node failure
Infrastructure failure
Certain platform failures
```

Goal:

> Keep the service available or restore it quickly.

### Disaster Recovery

Protects against larger events:

```text id="dr5m47"
Regional outage
Major platform failure
Data corruption
Operational disaster
```

Goal:

> Restore business operations within an acceptable timeframe.

---

# 2. Business Continuity

Business continuity is broader than databases.

It asks:

```text id="c1n8q4"
Can the business continue operating?
```

That includes:

```text id="b7m3x9"
Database
Application
Identity
Networking
DNS
Secrets
Monitoring
People
Processes
```

A perfect database DR plan is useless if the application can't connect to the recovered database.

---

# 3. RPO

**Recovery Point Objective** answers:

> **How much data can the business afford to lose?**

Example:

```text id="rpo7x2"
RPO = 15 minutes
```

Means the business accepts the possibility of losing up to approximately 15 minutes of data, depending on the recovery mechanism and scenario.

---

# 4. RTO

**Recovery Time Objective** answers:

> **How quickly must the service be restored?**

Example:

```text id="rto4k8"
RTO = 1 hour
```

means the recovery process should restore service within the defined one-hour objective.

---

# 5. RPO vs RTO

Memorize:

```text id="z6p3m1"
RPO → How much data can we lose?

RTO → How much time can we be unavailable?
```

---

# 6. Example

Business requirement:

```text id="q9v2d5"
RPO = 5 minutes
RTO = 30 minutes
```

Your architecture must support both.

A solution that restores in 10 minutes but loses two hours of data:

```text id="n3k7s1"
RTO ✓
RPO ✗
```

is not compliant with the requirement.

---

# 7. HA does not equal DR

Suppose SQL MI remains highly available inside a region.

But:

```text id="u8f4c2"
Entire region unavailable
```

Local HA cannot solve that.

You need a regional recovery strategy.

---

# 8. Failure domains

Think in layers:

```text id="f3m8q7"
Disk failure
   ↓
Host failure
   ↓
Zone/infrastructure failure
   ↓
Regional failure
   ↓
Global/business disaster
```

The larger the failure domain, the more deliberate your recovery architecture must be.

---

# 9. Single-region architecture

Basic:

```text id="s7d4p2"
Users
  ↓
Application
  ↓
SQL MI
```

Simple, but regional disaster recovery may be limited.

---

# 10. Regional DR architecture

Conceptually:

```text id="k4m9x6"
             Users
                |
             Application
                |
        +-------+-------+
        |               |
   Primary region    DR region
        |               |
      SQL MI          SQL MI
```

The DR region provides an alternative location for recovery.

---

# 11. Active vs passive DR

### Active-active

Both environments actively serve workload.

```text id="p8c2m5"
Region A ← users
Region B ← users
```

More complex.

### Active-passive

Primary handles normal traffic.

```text id="v5n7q3"
Region A → PRIMARY
Region B → STANDBY
```

During disaster:

```text id="m2x8k4"
Region B → PRIMARY
```

The right choice depends on business requirements and architecture.

---

# 12. Recovery strategies

You can think of recovery options as a spectrum:

```text id="r6c3v9"
Backup restore
      ↓
Warm standby
      ↓
Replicated secondary
      ↓
Active/active
```

Generally:

```text id="j7m4p2"
Faster recovery
↑
More cost/complexity
```

---

# 13. Backup-based DR

The simplest approach:

```text id="h2q8w6"
Primary
  ↓
Backups
  ↓
Disaster
  ↓
Provision/restore
  ↓
Recover
```

Advantages:

```text id="n4k1s8"
Lower cost
Simple conceptual model
Useful for corruption/recovery
```

Trade-off:

```text id="x3p7d5"
Recovery may take longer
```

---

# 14. Replication-based DR

Conceptually:

```text id="c9v5m2"
Primary
  ↓
Replication
  ↓
Secondary
```

If the primary region fails:

```text id="a8k3q6"
Secondary
  ↓
Become primary
```

This can significantly reduce recovery time compared with building everything from scratch.

---

# 15. Data replication ≠ application replication

You can have:

```text id="f7w2n9"
Database ready
```

but:

```text id="q3m8k1"
Application unavailable
```

because the DR environment lacks:

```text id="s5d6v4"
Compute
Networking
Identity
Secrets
Configuration
DNS
```

Therefore DR must cover the entire dependency chain.

---

# 16. Application dependency map

Build:

```text id="e8x2m7"
Users
 ↓
DNS
 ↓
Load balancer
 ↓
Application
 ↓
Identity
 ↓
SQL MI
 ↓
External services
```

For each dependency ask:

> **What happens if this component fails?**

---

# 17. Connection strings

Applications must know where to connect after failover.

Avoid hardcoding:

```text id="n2v6c9"
Primary server only
```

Prefer an architecture where the application can use an appropriate failover-aware endpoint or configuration mechanism.

---

# 18. DNS considerations

If the application uses DNS to reach database infrastructure:

```text id="x4p8m1"
Application
   ↓
DNS
   ↓
Database endpoint
```

During DR:

```text id="d7k3q5"
DNS/endpoint
   ↓
DR database
```

DNS TTL and caching behavior must be considered.

---

# 19. Failover

Failover means moving workload responsibility from the primary to another available environment.

Conceptually:

```text id="g5m9r2"
Primary
  ↓
Failure
  ↓
Failover decision
  ↓
Secondary
  ↓
Application reconnects
```

---

# 20. Failover is not just a database event

A complete failover may require:

```text id="c2k8v6"
Database
Application
DNS
Authentication
Networking
Secrets
Monitoring
Background jobs
```

---

# 21. Failover types

Operationally, distinguish:

```text id="q7m3x8"
Planned failover
```

from:

```text id="v9k4p2"
Unplanned failover
```

Planned failover is often used for:

```text id="h6d1r5"
DR testing
Maintenance
Migration
Validation
```

---

# 22. Planned failover

Ideal process:

```text id="s8p3n7"
Notify
 ↓
Pause/coordinate workloads if needed
 ↓
Fail over
 ↓
Reconnect
 ↓
Validate
```

This provides a controlled opportunity to test DR.

---

# 23. Unplanned disaster

Example:

```text id="a3x6m9"
Regional outage
 ↓
Primary unavailable
 ↓
DR decision
 ↓
Failover
 ↓
Application recovery
 ↓
Business validation
```

Time matters.

---

# 24. RPO during failover

Suppose:

```text id="z5n8q2"
Last synchronized data:
12:00

Failure:
12:05
```

Potential data exposure:

```text id="m7v4k1"
Up to ~5 minutes
```

Whether this is acceptable depends on the architecture and business RPO.

---

# 25. Data loss categories

Understand:

```text id="p4c9x6"
Zero/near-zero data loss
```

versus:

```text id="j8m2v5"
Some recent transactions may be lost
```

The architecture and replication mode determine the actual behavior.

Never promise:

> "No data loss."

unless the architecture and operational process genuinely support that guarantee.

---

# 26. Point-in-time restore

PITR answers:

> **Can I restore the database to a previous point in time?**

Useful for:

```text id="y3r7k2"
Accidental deletion
Bad deployment
Corruption
Operator error
```

Example:

```text id="n6m4p8"
09:00 healthy
09:30 bad deployment
09:45 discovered

Restore to:
09:29
```

---

# 27. DR vs corruption recovery

Replication can replicate bad changes.

Example:

```text id="u5c8j3"
DELETE mistake
   ↓
Primary
   ↓
Replicated
   ↓
Secondary
```

Now both environments may contain the mistake.

Therefore:

> **Replication protects availability, while backups provide an important recovery path for logical corruption.**

---

# 28. Backup strategy

A mature recovery strategy considers:

```text id="q8x4m1"
Full recovery capability
Retention
Restore points
Geo-redundancy where appropriate
Restore testing
```

---

# 29. Backup testing

A backup isn't truly trusted until restoration has been tested.

Test:

```text id="d2k7p5"
Can restore?
How long?
How much data?
Does application connect?
Are permissions intact?
Are jobs/configuration recoverable?
```

---

# 30. Restore time measurement

Don't assume:

```text id="f9m3v6"
"Restore should take 20 minutes."
```

Measure it.

Example:

```text id="h7x2q4"
Restore start: 01:00
Restore complete: 01:37

RTO contribution = 37 minutes
```

This is operational evidence.

---

# 31. DR drill

A proper DR exercise should simulate:

```text id="s4k8n2"
Primary unavailable
       ↓
Declare incident
       ↓
Activate DR
       ↓
Recover database
       ↓
Recover application
       ↓
Redirect traffic
       ↓
Validate business
```

---

# 32. DR test objectives

Measure:

```text id="p6m1x9"
Actual RTO
Actual RPO
Failover duration
Application recovery
Data validation
Communication
Runbook accuracy
```

---

# 33. DR runbook

Example:

```text id="x8q3m5"
1. Declare disaster
2. Identify failure scope
3. Confirm primary unavailable
4. Determine latest recoverable point
5. Activate DR database
6. Validate database
7. Configure application
8. Redirect traffic
9. Validate critical transactions
10. Monitor
11. Communicate recovery
12. Document incident
```

---

# 34. Recovery priority

Not every application is equally important.

Classify:

```text id="m3v7p1"
Tier 0 → Mission critical
Tier 1 → Critical
Tier 2 → Important
Tier 3 → Non-critical
```

Then define:

```text id="r5x9k4"
RTO
RPO
Recovery order
```

for each tier.

---

# 35. Example

### Payment system

```text id="q2f8m6"
RTO = 15 min
RPO = near-zero/very low
```

### Internal reporting

```text id="v6n1x3"
RTO = 8 hours
RPO = 4 hours
```

Different workloads deserve different DR investments.

---

# 36. Cost vs resilience

A more resilient architecture generally costs more.

Think:

```text id="z9m4c7"
Low cost
   ↓
Backup restore

Higher cost
   ↓
Warm standby

Higher cost
   ↓
Replicated secondary

Highest complexity
   ↓
Active/active
```

Choose based on business requirements—not technology fashion.

---

# 37. Security during DR

Don't forget:

```text id="e5x8n2"
Encryption
Identity
Secrets
Certificates
Firewall/network rules
Private connectivity
Auditing
```

A DR environment should not become a security exception.

---

# 38. Identity recovery

Suppose the database is recovered.

Application still fails because:

```text id="k7m3p9"
Managed identity
   ↓
Missing permissions
```

Therefore validate identity as part of DR.

---

# 39. Monitoring recovery

After failover, monitoring must follow the new primary.

Verify:

```text id="c8v2x5"
Alerts
Dashboards
Logs
Query Store
On-call routing
```

Don't recover the database and forget observability.

---

# 40. Background jobs

A common DR trap:

```text id="j4n8m2"
Primary application
   ↓
Scheduled job
```

After failover, both primary and DR environments may run the job.

Result:

```text id="w6q3p7"
Duplicate processing
```

Design jobs carefully for DR.

---

# 41. Split-brain risk

You must avoid having two environments both believing they are primary.

Conceptually:

```text id="a9m5x3"
Region A → Primary
Region B → Primary
```

This can cause:

```text id="p2k7v8"
Conflicting writes
Duplicate processing
Data inconsistency
```

Your failover architecture must define authoritative ownership.

---

# 42. DR communication

During a real disaster:

```text id="q6x1m4"
Technical recovery
+
Business communication
```

Stakeholders need to know:

```text id="n8v3k5"
What happened?
Is service available?
What functionality is restored?
What data point was recovered?
Are there known limitations?
```

---

# 43. Incident command

Large incidents benefit from clear roles:

```text id="r3m7x2"
Incident commander
DBA
Application lead
Network lead
Security
Communications
Business owner
```

One person shouldn't try to coordinate everything while simultaneously debugging SQL.

---

# 44. DR success criteria

A DR exercise succeeds only if:

```text id="k5x8q1"
Database recovered
+
Application recovered
+
Users can connect
+
Critical transactions work
+
RPO met
+
RTO met
+
Monitoring works
```

---

# 45. Example DR exercise

Assume:

```text id="d4n9m2"
RPO = 15 min
RTO = 60 min
```

At:

```text id="g7p3x8"
10:00 → simulated regional outage
```

---

### 10:05

Incident declared.

```text id="v6k2q4"
5 min elapsed
```

---

### 10:10

DR database activated.

```text id="h8m4c1"
10 min
```

---

### 10:25

Application redirected.

```text id="q3x7n5"
25 min
```

---

### 10:35

Business transaction succeeds.

```text id="z1p6m8"
35 min
```

---

### Result

```text id="b9k2r4"
RTO target = 60 min
Actual = 35 min
```

Good.

Now verify:

```text id="s5v8x2"
RPO target = 15 min
Actual data gap = 8 min
```

Also good.

---

# 46. What if DR fails?

Suppose:

```text id="c7m3q9"
Database recovered
Application cannot authenticate
```

Then:

```text id="j4x8n1"
DR test = FAILED
```

That's not bad.

It's valuable information.

The purpose of a DR drill is to discover weaknesses **before** a real disaster.

---

# 47. DR maturity

### Level 1

```text id="v5k2m8"
"We have backups."
```

### Level 2

```text id="r9x3q1"
"We have documented restore steps."
```

### Level 3

```text id="n6m8c4"
"We have tested restore."
```

### Level 4

```text id="p2v7k5"
"We have tested regional recovery."
```

### Level 5

```text id="z8m1x6"
"We regularly prove RPO/RTO through realistic exercises."
```

---

# 48. DR checklist

```text id="t4q9m3"
BUSINESS
☐ RPO defined
☐ RTO defined
☐ Application priorities defined

DATABASE
☐ Backup strategy
☐ Restore strategy
☐ Replication/secondary where required
☐ PITR tested
☐ Recovery tested

APPLICATION
☐ Connection strategy
☐ Authentication
☐ Secrets
☐ DNS
☐ Background jobs

OPERATIONS
☐ DR runbook
☐ Monitoring
☐ Incident roles
☐ Communication plan
☐ Escalation

TESTING
☐ Failover tested
☐ Restore tested
☐ Application tested
☐ Business transaction tested
☐ RPO measured
☐ RTO measured
```

---

# 49. Interview question

### "What's the difference between RPO and RTO?"

> "RPO defines the maximum acceptable amount of data loss, while RTO defines the maximum acceptable recovery time."

---

# 50. Interview question

### "Are backups enough for DR?"

> "Not necessarily. Backups provide an important recovery mechanism, especially for logical corruption, but backup-based recovery may not meet a strict RTO. The appropriate architecture depends on the business's RPO/RTO requirements."

---

# 51. Interview question

### "Why isn't replication a replacement for backups?"

Because:

```text id="n2x7p4"
Replication
    ↓
Copies changes
```

If an operator makes a bad change:

```text id="m8c3k5"
Bad DELETE
   ↓
Replicated
   ↓
Secondary also receives it
```

Backups/PITR can provide a recovery point before the mistake.

---

# 52. Interview question

### "How would you test DR?"

Strong answer:

```text id="x4p7m9"
1. Define RPO/RTO.
2. Simulate the appropriate failure.
3. Execute the documented recovery process.
4. Recover the database.
5. Recover dependent application components.
6. Redirect traffic.
7. Execute critical business transactions.
8. Measure actual RPO/RTO.
9. Verify monitoring and security.
10. Document gaps and improve the runbook.
```

---

# 53. Interview question

### "What is the biggest DR mistake?"

One of the biggest:

> **Assuming that because replication or backups exist, recovery will work.**

The only way to gain confidence is to **test recovery**.

---

# 54. Final DR architecture

```text id="w6m2r9"
                  USERS
                    |
               APPLICATION
                    |
          +---------+---------+
          |                   |
       PRIMARY              DR
       REGION             REGION
          |                   |
        SQL MI              SQL MI
          |                   |
          +---------+---------+
                    |
              Recovery Data
                    |
             Backup / Restore
                    |
              DR Procedures
                    |
              Business Test
```

---

# 55. Golden rule

> **A DR strategy is not a diagram. It is a tested ability to recover the business.**

And remember:

```text id="p9x3m7"
RPO → How much data?
RTO → How much time?
HA  → Stay available
DR  → Recover from disaster
Backup → Recover data
Replication → Improve availability/recovery
Testing → Prove the plan
```

---

## Next — Lesson 106: Azure SQL Managed Instance Security — Advanced DBA

We'll cover **Microsoft Entra authentication, RBAC vs SQL permissions, managed identities, TDE, Always Encrypted, dynamic data masking, row-level security, auditing, Defender for SQL, private endpoints, network isolation, key management, privileged access, and a complete production security architecture.**
