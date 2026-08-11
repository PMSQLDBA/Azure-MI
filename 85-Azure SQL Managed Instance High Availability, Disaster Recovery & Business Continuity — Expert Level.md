# Lesson 117 — Azure SQL Managed Instance High Availability, Disaster Recovery & Business Continuity — Expert Level

The goal is to move from:

> **“We have a backup and a secondary region.”**

to:

> **“We have a tested business-continuity architecture with defined RPO/RTO, clear failure domains, controlled failover, and a documented recovery process.”**

---

# 1. HA vs DR

These are not the same.

### High Availability

Protects primarily against failures **within the normal service/region architecture**.

```text
Component failure
      ↓
Service continues
```

### Disaster Recovery

Protects against larger failures such as:

```text
Region failure
Major platform failure
Large-scale corruption
Operational disaster
```

Think:

```text
HA → keep running
DR → recover elsewhere
```

---

# 2. Business continuity

Business continuity asks:

> **How does the business continue operating when something goes wrong?**

It includes more than SQL MI.

```text
Database
+
Application
+
Networking
+
Identity
+
DNS
+
People
+
Runbooks
```

---

# 3. The four critical questions

Every production design should answer:

```text id="8y6d7j"
1. What can fail?
2. How much data can we lose?
3. How long can we be down?
4. How do we recover?
```

These determine the architecture.

---

# 4. RPO

**Recovery Point Objective** answers:

> **How much data loss is acceptable?**

Example:

```text id="bq6b6x"
RPO = 15 minutes
```

means the business may tolerate losing up to roughly 15 minutes of recent changes, depending on the recovery mechanism and actual guarantees.

---

# 5. RTO

**Recovery Time Objective** answers:

> **How quickly must service return?**

Example:

```text id="m0w5p9"
RTO = 1 hour
```

means the recovery process must target restoring service within one hour.

---

# 6. RPO + RTO drive architecture

Suppose the business requires:

```text id="f3x2k6"
RPO < 5 minutes
RTO < 15 minutes
```

A simple backup-only recovery strategy may not satisfy those objectives.

You need an architecture designed around the required recovery characteristics.

---

# 7. Failure domains

Think about failure at multiple levels:

```text id="n7u8q3"
Query
 ↓
Database
 ↓
SQL MI
 ↓
Availability zone / infrastructure
 ↓
Region
 ↓
Organization
```

Your protection should match the failure you're trying to survive.

---

# 8. HA architecture

A managed database service such as SQL MI provides built-in high-availability capabilities.

The important operational principle is:

> **Don't recreate the service's internal HA architecture manually unless the product architecture explicitly requires it.**

Instead, understand what failures the service is designed to tolerate.

---

# 9. Zone redundancy

Where supported and appropriate, zone redundancy can improve resilience against failures affecting an availability zone.

Conceptually:

```text id="8c0x8p"
Region
 ├── Zone 1
 ├── Zone 2
 └── Zone 3
```

The service distributes appropriate components across failure domains.

---

# 10. Zone failure vs region failure

These are very different.

### Zone failure

```text id="4q2y4w"
Region remains available
```

### Region failure

```text id="6a0g3z"
Entire regional service becomes unavailable
```

Zone redundancy does **not** automatically mean regional disaster recovery.

---

# 11. Regional DR

A typical multi-region architecture:

```text id="p7q5l2"
Primary Region
     |
     | Replication / DR mechanism
     ↓
Secondary Region
```

The secondary region should be sufficiently independent to survive the primary region's failure.

---

# 12. Don't put both sides in the same failure domain

This defeats the purpose:

```text id="r8v3k1"
Primary
   ↓
Secondary
   ↓
Same region
```

If the disaster affects the whole region, both may be impacted.

---

# 13. Active-passive

A common DR model:

```text id="u9q0x2"
Primary → serving traffic
Secondary → standby
```

During disaster:

```text id="j3m6w8"
Primary fails
 ↓
Promote secondary
 ↓
Redirect application
```

---

# 14. Active-active

Another model:

```text id="s4y2d8"
Region A → traffic
Region B → traffic
```

Both regions actively serve workloads.

This can improve availability but is substantially more complex for stateful databases and application consistency.

---

# 15. DR isn't just a database

Suppose the database exists in Region B.

But:

```text id="6h2t4n"
Application unavailable
DNS points to Region A
Identity unavailable
Network unavailable
Secrets unavailable
```

You still don't have functional DR.

---

# 16. Full DR stack

Think:

```text id="a9r4j0"
DNS
 ↓
Application
 ↓
Identity
 ↓
Networking
 ↓
SQL MI
 ↓
Secrets
 ↓
Monitoring
```

All critical dependencies need a recovery strategy.

---

# 17. Application failover

Database failover alone may not redirect users.

Typical flow:

```text id="4b8v2s"
Database failure
 ↓
DR database promoted
 ↓
Application endpoint changes
 ↓
Traffic redirected
 ↓
Application reconnects
```

The mechanism depends on your application architecture.

---

# 18. DNS considerations

If DNS is part of failover:

```text id="0h8r1g"
Application endpoint
      ↓
DNS
      ↓
Healthy region
```

Consider:

```text id="4u7n3j"
TTL
Caching
Propagation behavior
Health checks
Client behavior
```

DNS is not instantaneous magic.

---

# 19. Connection strings

Applications should avoid hard-coding a single disaster-prone endpoint.

Prefer an architecture where connection routing/failover can be managed appropriately.

For SQL MI, use the supported connectivity and failover patterns for the specific HA/DR architecture rather than inventing application-side database switching logic.

---

# 20. Planned vs unplanned failover

### Planned

```text id="5f0q7n"
Maintenance
Testing
Migration
Controlled event
```

### Unplanned

```text id="r5p8c1"
Regional outage
Infrastructure failure
Major incident
```

Your runbook must handle both.

---

# 21. Planned failover

A planned failover should generally involve:

```text id="8u4s9w"
Validate secondary
 ↓
Check replication
 ↓
Confirm application readiness
 ↓
Initiate failover
 ↓
Validate
 ↓
Redirect/confirm traffic
```

---

# 22. Unplanned failover

During an actual disaster:

```text id="2y6d1m"
Detect failure
 ↓
Confirm scope
 ↓
Declare incident
 ↓
Promote DR
 ↓
Redirect application
 ↓
Validate business transactions
 ↓
Communicate status
```

---

# 23. Failover is not recovery

This distinction matters.

### Failover

Move service to another healthy replica/region.

### Recovery

Restore service/data using backups or other recovery mechanisms.

```text id="0n3p8r"
Failover → switch
Recovery → rebuild/restore
```

---

# 24. Backup-based DR

Backups provide another recovery path.

Conceptually:

```text id="k1m4s7"
Backup
 ↓
Recovery environment
 ↓
Restore
 ↓
Validate
 ↓
Application reconnect
```

This can be slower than a continuously synchronized DR architecture but is extremely important as an independent recovery mechanism.

---

# 25. Why backups still matter with replication

Replication doesn't protect against every problem.

Consider:

```text id="6h1p4y"
Bad application change
 ↓
Data corrupted
 ↓
Replication copies corruption
```

A historical backup may provide a recovery point before the corruption.

---

# 26. Logical corruption

Examples:

```text id="z7r8s3"
DELETE without WHERE
Bad migration
Application bug
Incorrect UPDATE
```

HA/replication may preserve the bad change.

This is why point-in-time recovery and backup strategy remain important.

---

# 27. Recovery hierarchy

Think:

```text id="0f6m2k"
HA
 ↓
Failover
 ↓
DR
 ↓
Point-in-time restore
 ↓
Long-term backup/archive
```

Each addresses different failure scenarios.

---

# 28. RPO tradeoff

Generally:

```text id="2x8h1m"
Lower RPO
   ↓
More synchronization / infrastructure complexity
   ↓
Potentially higher cost
```

The business must decide what data loss is acceptable.

---

# 29. RTO tradeoff

Similarly:

```text id="4v3g7z"
Lower RTO
   ↓
More pre-positioned infrastructure
   ↓
More automation
   ↓
More testing
```

Faster recovery generally requires more preparation.

---

# 30. DR readiness

A secondary isn't necessarily "DR-ready."

Verify:

```text id="p5q8m4"
Replication healthy
Configuration correct
Application compatible
Network ready
Identity ready
Secrets available
Monitoring configured
Runbook tested
```

---

# 31. Configuration drift in DR

Danger:

```text id="g3x6n8"
Primary
 ↓
Manual change
```

while:

```text id="f4h2q0"
Secondary
 ↓
Old configuration
```

During disaster:

> The secondary doesn't behave like production.

IaC helps reduce this risk.

---

# 32. DR infrastructure as code

Maintain both environments through the same automation model:

```text id="n2j7v6"
Reusable IaC
     ↓
Primary parameters
Secondary parameters
```

This reduces configuration divergence.

---

# 33. Data consistency

A critical DR question:

> **What exactly does the secondary contain at failover time?**

Understand:

```text id="w0x8k5"
Synchronous vs asynchronous behavior
Replication lag
Last known good point
Potential data loss
```

Never promise zero data loss unless the architecture and service guarantees actually support it.

---

# 34. Replication lag

Suppose:

```text id="q7k2m3"
Primary:
Transaction 1000

Secondary:
Transaction 980
```

The secondary is behind.

If failover occurs now:

```text id="e5w9t1"
Transactions 981–1000
```

may be affected depending on the replication mechanism and failure scenario.

---

# 35. Monitor lag

Don't merely configure replication.

Monitor:

```text id="x8q4v6"
Replication health
Lag
Last synchronization
Errors
Secondary availability
```

---

# 36. Application consistency

A database can be healthy while the application isn't.

After failover, validate:

```text id="a6j0y4"
Authentication
Reads
Writes
Transactions
Background jobs
Queues
External integrations
```

---

# 37. Background jobs

A common DR failure:

```text id="9x7n2c"
Primary fails
 ↓
DR application starts
 ↓
Both regions execute scheduled job
 ↓
Duplicate processing
```

DR architecture must account for schedulers and workers.

---

# 38. Message queues

If the application uses queues:

```text id="f3j5p8"
Database DR
≠
Queue DR
```

You need a strategy for:

```text id="k6v1r9"
Messages
Offsets
Consumers
Duplicates
Ordering
Replay
```

---

# 39. External dependencies

Your application may depend on:

```text id="5s2m8q"
Payment gateway
Identity service
Storage
Third-party APIs
DNS
Email
```

DR must include the critical dependency chain.

---

# 40. DR runbook

A runbook should specify:

```text id="r1m9v6"
Trigger
Owner
Decision authority
Failover steps
Validation steps
Communication
Rollback/failback
Post-incident actions
```

---

# 41. Example runbook structure

```text id="6t4w8p"
1. Declare incident
2. Confirm primary failure
3. Verify secondary health
4. Check replication status
5. Initiate failover
6. Update application routing
7. Run smoke tests
8. Validate critical transactions
9. Notify stakeholders
10. Monitor recovery
```

---

# 42. Don't improvise during a disaster

Bad:

```text id="k8p0s1"
"Let's figure it out now."
```

Better:

```text id="c4n7d2"
Documented
+
Tested
+
Automated
```

The less cognitive load during an incident, the better.

---

# 43. DR testing

A DR plan that has never been tested is theoretical.

Test:

```text id="0r6h4v"
Failover
Application reconnection
Data validation
DNS/routing
Authentication
Monitoring
Rollback/failback
```

---

# 44. Tabletop exercise

You don't always need to immediately cause a production outage.

Start with:

```text id="3w8p6k"
Scenario:
"Primary region unavailable."
```

Then walk through:

```text id="9d1f7q"
Who acts?
What command?
What dependency?
What decision?
What communication?
```

---

# 45. Technical DR test

Eventually perform controlled technical failover testing where supported.

Measure:

```text id="n4q7s2"
Actual RTO
Actual RPO
Application downtime
Data loss
Human intervention
Unexpected dependencies
```

---

# 46. Compare actual vs target

Suppose:

```text id="4j6x8m"
Target RTO = 30 minutes
Actual RTO = 74 minutes
```

Your DR architecture does not meet the business requirement.

The test exposed a design gap.

---

# 47. Failback

People often forget the return journey.

After primary recovery:

```text id="7m2p5d"
Primary restored
 ↓
Synchronize
 ↓
Validate
 ↓
Plan failback
 ↓
Switch traffic
 ↓
Validate
```

---

# 48. Never rush failback

A recovered primary isn't automatically safe.

Verify:

```text id="u6r1q9"
Infrastructure
Database
Replication
Application
Networking
Monitoring
```

before switching back.

---

# 49. Split-brain risk

Distributed systems must avoid two regions believing simultaneously:

> **“I am the primary writer.”**

That can cause:

```text id="h4y8n2"
Conflicting writes
Data divergence
Duplicate processing
Severe recovery complexity
```

Use supported failover mechanisms and clear authority.

---

# 50. DR monitoring

Create a dedicated DR dashboard:

```text id="e3k7m0"
Primary health
Secondary health
Replication status
Lag
Backup health
Last successful sync
Network readiness
Application readiness
```

---

# 51. DR alerting

Examples:

```text id="d7q5k1"
Replication unhealthy
Lag above threshold
Secondary unavailable
Backup failure
DR configuration drift
```

These should be detected **before** a disaster.

---

# 52. DR security

The secondary must be secured as strongly as the primary.

Don't create:

```text id="v8x4m6"
Production = hardened
DR = forgotten
```

An attacker may specifically target the weaker recovery environment.

---

# 53. DR secrets

Ensure the DR environment can securely access:

```text id="z1r5q8"
Secrets
Certificates
Managed identities
Configuration
```

without embedding emergency credentials into a runbook.

---

# 54. DR observability

During failover you need visibility into:

```text id="0p7n3s"
Database
Application
Network
Identity
Replication
User traffic
```

Otherwise you can successfully fail over the database but still have a broken service.

---

# 55. Business continuity tiers

Not every workload deserves identical DR.

Example:

```text id="k5v2j9"
Tier 0 → mission critical
Tier 1 → critical
Tier 2 → important
Tier 3 → noncritical
```

Then assign different:

```text id="1m8x4q"
RPO
RTO
DR architecture
Testing frequency
Cost
```

---

# 56. Cost optimization

A low-priority workload may use:

```text id="f9w2c6"
Backup-based recovery
```

while a mission-critical workload may require:

```text id="h6q1n8"
Continuously available DR architecture
+
Automated failover
+
Frequent testing
```

Don't over-engineer every database.

---

# 57. Disaster categories

Consider at least:

```text id="u8s4m1"
Hardware/platform failure
Zone failure
Region failure
Network failure
Identity failure
Application failure
Human error
Data corruption
Security incident
```

Different failures require different recovery paths.

---

# 58. Human error

This is especially important.

Example:

```text id="a3k7w9"
Engineer runs bad DELETE
```

Regional failover doesn't solve it.

You need:

```text id="p0n6r2"
Backup
+
Point-in-time recovery
+
Access controls
+
Operational safeguards
```

---

# 59. Cyberattack scenario

Suppose an attacker gains privileged access.

You may need:

```text id="m2q8v4"
Contain identity
 ↓
Block access
 ↓
Preserve evidence
 ↓
Determine blast radius
 ↓
Recover from trusted data
 ↓
Rotate credentials
 ↓
Validate integrity
```

DR must include security incidents, not only infrastructure failures.

---

# 60. Recovery validation

Don't just check:

```text id="x5n9c7"
SQL connection works
```

Validate business operations:

```text id="j1k6p8"
Create customer
Create order
Process transaction
Read reporting data
```

Use safe synthetic or controlled tests.

---

# 61. DR architecture example

```text id="p3r8t6"
             GLOBAL ENTRY
                  |
           +------+------+
           |             |
       REGION A       REGION B
       Primary         DR
           |             |
        SQL MI         SQL MI
           |             |
           +------DR-----+
                  |
            Central Monitor
                  |
            Incident Team
```

The exact replication/failover mechanism should be chosen based on SQL MI's currently supported capabilities and business RPO/RTO requirements.

---

# 62. Expert interview question

### "What's the difference between HA and DR?"

> "HA is designed to keep the service available through failures within the normal service architecture, while DR addresses larger disasters such as regional outages or major data-center failures. HA reduces downtime; DR provides a recovery path when the primary environment itself is unavailable."

---

# 63. Interview question

### "How do RPO and RTO affect SQL MI architecture?"

> "RPO determines how much data loss the business can tolerate, while RTO determines how quickly service must return. Lower RPO generally requires stronger data synchronization, while lower RTO requires more pre-positioned infrastructure and automation. These objectives drive the choice of HA, DR, replication, backup, and failover strategies."

---

# 64. Interview question

### "Why isn't replication enough?"

> "Replication can protect against infrastructure failure, but it may also replicate logical mistakes or corrupted changes. For example, an accidental DELETE can propagate to the secondary. Independent backups and point-in-time recovery are therefore still important."

---

# 65. Interview question

### "How do you test DR?"

> "I define measurable RPO and RTO targets, perform controlled failover tests, validate database and application functionality, verify routing and identity dependencies, measure actual recovery time and data loss, document gaps, and repeat the exercise periodically."

---

# 66. Interview question

### "What would you monitor for a DR environment?"

Strong answer:

```text id="8n3q6r"
Replication health
Replication lag
Secondary availability
Backup success
Last synchronization
Network connectivity
Identity readiness
Application readiness
Failover status
```

---

# 67. DR maturity model

### Level 1

```text id="5f8m1w"
Backups exist
```

### Level 2

```text id="9q3v7k"
Documented recovery procedure
```

### Level 3

```text id="6h2x4p"
Secondary environment
```

### Level 4

```text id="1j9r5n"
Automated failover + tested RPO/RTO
```

### Level 5

```text id="7c4m8s"
Regular chaos/failure testing
+
Automated recovery
+
Business continuity across dependencies
```

---

# 68. Golden rule

> **A DR architecture is not proven by its diagram. It is proven by a successful recovery test.**

---

# 69. Final mental model

```text id="m5x8q2"
                BUSINESS
                   |
             RPO + RTO
                   |
          FAILURE ANALYSIS
                   |
       +-----------+-----------+
       |                       |
      HA                      DR
       |                       |
   Local failure          Regional disaster
       |                       |
       +-----------+-----------+
                   |
              SQL MI
                   |
        +----------+----------+
        |                     |
      Backup              Replication
        |                     |
        +----------+----------+
                   |
              APPLICATION
                   |
              NETWORK/DNS
                   |
              IDENTITY/SECRETS
                   |
             MONITORING
                   |
             RUNBOOK + TEST
                   |
                RECOVER
```

That is the foundation of **expert Azure SQL Managed Instance HA, DR, and business continuity architecture**.

---

## Next — Lesson 118: Azure SQL Managed Instance Performance Engineering — Expert Level

We'll go deep into **query optimization, execution plans, cardinality estimation, statistics, indexing strategy, parameter sensitivity, memory grants, waits, CPU pressure, I/O bottlenecks, tempdb, concurrency, plan regression, Query Store, workload isolation, and systematic performance troubleshooting.**
