# Lesson 111 — Azure SQL Managed Instance High Availability, Failover & Disaster Recovery — Advanced

This lesson focuses on the question every production DBA must be able to answer:

> **“If the database fails—or the entire Azure region becomes unavailable—how quickly can we recover, how much data could we lose, and how do we prove it?”**

---

# 1. HA vs DR

These are related but different.

### High Availability

Protects against failures **within the normal operating environment**.

```text
Primary workload
      ↓
HA mechanism
      ↓
Service continues
```

Goal:

> Minimize interruption.

### Disaster Recovery

Protects against **larger-scale failures**.

```text
Primary region
      ↓
Major disaster
      ↓
Secondary region
      ↓
Recovery
```

Goal:

> Restore service after a major failure.

---

# 2. The four questions

A strong DR design answers:

```text id="7p4m2x"
1. What can fail?
2. How much data can we lose?
3. How quickly must service return?
4. How do we prove recovery works?
```

---

# 3. RPO

**Recovery Point Objective**

RPO answers:

> How much data loss can the business tolerate?

Example:

```text id="m8c3q7"
RPO = 5 minutes
```

means the recovery strategy should target no more than approximately five minutes of acceptable data loss.

---

# 4. RTO

**Recovery Time Objective**

RTO answers:

> How long can the service be unavailable?

Example:

```text id="x5n9v2"
RTO = 30 minutes
```

means the service should be restored within that target.

---

# 5. RPO vs RTO

Remember:

```text id="q3m7k8"
RPO → Data loss
RTO → Downtime
```

A system can have:

```text
Low RPO
High RTO
```

or:

```text
Low RTO
Higher RPO
```

depending on architecture and cost.

---

# 6. Business requirements first

Don't begin with:

> "Which Azure feature should I use?"

Begin with:

```text id="v6p2m4"
Business criticality
      ↓
RPO
      ↓
RTO
      ↓
Architecture
      ↓
Technology
```

---

# 7. Example

Payment database:

```text id="n8r3x5"
RPO = near-zero/very low
RTO = minutes
```

A development database:

```text id="k4m9p1"
RPO = hours
RTO = hours
```

These should not necessarily receive identical DR architecture.

---

# 8. Failure domains

Think about increasingly larger failures:

```text id="x7c2m8"
Query/process failure
        ↓
Database failure
        ↓
Instance/service failure
        ↓
Zone/infrastructure failure
        ↓
Region failure
        ↓
Wider Azure/service incident
```

Your architecture should match the failures you're required to survive.

---

# 9. High availability

HA is about maintaining service during expected infrastructure/service failures.

Conceptually:

```text id="r5m8q3"
Application
     |
     v
SQL MI
     |
   HA
     |
Healthy service
```

The exact underlying platform implementation is managed by Azure.

---

# 10. Don't confuse HA with backup

Backup:

```text id="p3x7m9"
Recovery mechanism
```

HA:

```text id="c8n2v5"
Service continuity mechanism
```

Backups are still essential even when HA exists.

---

# 11. Why HA doesn't replace backups

Imagine:

```text id="m6q4x8"
Accidental DELETE
```

HA may faithfully preserve the bad transaction across the service.

You need:

```text id="n9v3p2"
Point-in-time recovery
```

or another appropriate recovery mechanism.

---

# 12. Logical corruption

Consider:

```text id="x4m8c7"
Bad deployment
 ↓
Incorrect UPDATE
 ↓
Data corruption
```

HA does not necessarily protect you from this.

DR strategy must include **logical recovery**, not only infrastructure failure.

---

# 13. Backup strategy

A recovery strategy generally considers:

```text id="q7p2n5"
Full recovery capability
Point-in-time recovery
Long-term retention requirements
Geo-redundancy
Restore testing
```

Exact options depend on the Azure SQL MI configuration and service capabilities.

---

# 14. Point-in-time recovery

Conceptually:

```text id="v3m8k1"
Database
   |
   +-- Full/base backups
   |
   +-- Log information
   |
   ↓
Restore to desired point
```

The important idea:

> Recover the database to a known valid point before the incident.

---

# 15. Example

Suppose:

```text id="r6x2m9"
09:00 → healthy
09:30 → bad deployment
10:00 → corruption discovered
```

If your recovery mechanism supports the required point-in-time target, you may restore to a point before the bad change.

---

# 16. Geo-redundancy

For regional disaster scenarios, recovery data needs to exist outside the affected region.

Conceptually:

```text id="m5n8c3"
Region A
   |
   | replication / backup
   ↓
Region B
```

This protects against regional failure.

---

# 17. Secondary region

A DR architecture often looks like:

```text id="p8q3v6"
PRIMARY REGION
     |
     | replication
     ↓
SECONDARY REGION
```

The secondary may be:

```text id="x2m7k9"
Active
Warm
Passive
Backup-based
```

depending on requirements.

---

# 18. Active-active vs active-passive

### Active-passive

```text id="c5v9n2"
Primary → serving traffic
Secondary → waiting
```

### Active-active

```text id="r7m3x8"
Region A → traffic
Region B → traffic
```

Active-active is more complex, particularly around writes and data consistency.

---

# 19. Failover

Failover means moving service from the current primary to an alternative location or healthy service.

Conceptually:

```text id="n4k8p2"
PRIMARY
   X
   ↓
SECONDARY
   ↓
APPLICATION
```

---

# 20. Planned vs unplanned failover

### Planned

Used for:

```text id="m9x2c5"
DR exercises
Maintenance
Architecture testing
Regional migration
```

You control the timing.

### Unplanned

Used when:

```text id="q6v3n8"
Primary becomes unavailable
```

The priority is restoring service.

---

# 21. Failover isn't just a database operation

You must consider:

```text id="x8m4p1"
Database
+
Application
+
DNS/routing
+
Authentication
+
Secrets
+
Monitoring
```

A database can be healthy in the secondary region while the application remains unable to connect.

---

# 22. Connection handling

Applications should handle transient connectivity failures.

Conceptually:

```text id="c3n7v5"
Connection failure
      ↓
Retry policy
      ↓
Reconnect
      ↓
Continue
```

Retries must be carefully designed.

---

# 23. Retry strategy

Bad:

```text id="r2m8x6"
Retry immediately
Retry immediately
Retry immediately
Retry immediately
```

This can create a retry storm.

Better:

```text id="p5q3n9"
Retry
 ↓
Backoff
 ↓
Retry
 ↓
Backoff
 ↓
Retry
```

Use bounded retries and appropriate jitter where supported by the application stack.

---

# 24. Idempotency

Retries can create duplicate operations.

Example:

```text id="v8m2c4"
Payment request
 ↓
Timeout
 ↓
Application retries
```

Was the first request committed?

If yes, retrying blindly may create a duplicate.

Critical operations should use appropriate idempotency mechanisms.

---

# 25. Connection strings

A DR-capable application should use an appropriate connection strategy that can continue operating after failover.

Avoid architectures where:

```text id="n7x4m9"
Application
   ↓
Hardcoded primary IP
```

is the recovery mechanism.

Use supported service endpoints and application connection strategies designed for failover.

---

# 26. DNS and endpoint considerations

After a regional recovery event:

```text id="q3m8p6"
Where does application traffic go?
```

You need a deliberate answer.

Potential mechanisms include service-provided endpoints, DNS-based routing, or application configuration depending on the architecture.

---

# 27. DNS caching

Even if DNS is updated correctly, clients may cache records.

Therefore:

```text id="m5v9x2"
DNS change
≠
Instant global application switch
```

Test actual application behavior.

---

# 28. Failover groups

For Azure SQL workloads requiring cross-region failover, failover groups can provide a structured mechanism for managing primary/secondary relationships and listener endpoints.

The important concept is:

```text id="x6p3k8"
Primary
   |
Failover relationship
   |
Secondary
```

Applications can use the appropriate listener endpoint rather than embedding a specific primary instance address.

---

# 29. Why listener endpoints matter

Application configuration should ideally remain stable across failover.

Conceptually:

```text id="r8m4n2"
Application
      |
      v
Listener
   /     \
Primary  Secondary
```

Failover changes the destination behind the abstraction.

---

# 30. DR application dependencies

Don't only test SQL.

List:

```text id="c7x2m5"
Application servers
Identity
DNS
Secrets
Certificates
Queues
Storage
External APIs
Monitoring
```

DR fails if a critical dependency remains only in the primary region.

---

# 31. Dependency mapping

Build a diagram:

```text id="v3q9m6"
Users
  |
DNS / Front Door
  |
Application
  |
SQL MI
  |
Other services
```

Then mark:

```text id="p8m4x1"
Primary region
Secondary region
Global
```

---

# 32. DR readiness checklist

Before calling a system "DR ready":

```text id="k2n7c5"
☐ Database recovery mechanism tested
☐ Secondary region available
☐ Application deployed there
☐ Secrets available
☐ Identity works
☐ Network works
☐ Monitoring works
☐ DNS/routing works
☐ Runbook exists
☐ People know the procedure
```

---

# 33. DR runbook

A useful runbook contains:

```text id="m9x3p7"
Detection
 ↓
Assessment
 ↓
Decision authority
 ↓
Failover steps
 ↓
Application validation
 ↓
Communication
 ↓
Monitoring
 ↓
Recovery / failback
```

---

# 34. Decision authority

During an incident, people shouldn't argue:

> "Who is allowed to initiate failover?"

Define this beforehand.

Example:

```text id="q6v2n8"
Incident Commander
       |
Database Lead
       |
Application Lead
```

---

# 35. Failover criteria

Define objective conditions.

For example:

```text id="x4m8c3"
Primary unavailable
+
Estimated recovery exceeds RTO
+
Secondary healthy
+
Business approval
```

Avoid making critical decisions entirely from intuition during an incident.

---

# 36. DR test

A DR test should simulate:

```text id="r7n3p5"
Primary unavailable
      ↓
Failover
      ↓
Application reconnect
      ↓
Transaction test
      ↓
Monitoring validation
      ↓
Business validation
```

---

# 37. Don't just test failover

Test the whole chain:

```text id="m2x8q6"
User
 ↓
DNS/routing
 ↓
Application
 ↓
Authentication
 ↓
SQL MI
 ↓
Critical transaction
```

---

# 38. RTO measurement

Don't claim:

```text id="c9p3v7"
RTO = 30 minutes
```

because the architecture theoretically supports it.

Measure:

```text id="n6m4x2"
Incident start
        ↓
Failover initiated
        ↓
Database ready
        ↓
Application ready
        ↓
First successful transaction
```

Actual elapsed time is your evidence.

---

# 39. RPO measurement

Similarly, measure actual recovery point.

Determine:

```text id="x8v2m5"
Last known committed transaction
vs
Recovered transaction point
```

Then compare with the business RPO.

---

# 40. DR test evidence

Record:

```text id="p4m7c9"
Test date
Participants
Start time
Failover time
RTO achieved
RPO achieved
Issues
Corrective actions
```

This turns DR from a claim into evidence.

---

# 41. Failback

After recovering the original region:

```text id="k5x9n3"
Secondary
   ↓
Current production
```

Do you immediately return to the original primary?

Not necessarily.

First determine:

```text id="v7m2p8"
Original region healthy?
Data synchronized?
Application ready?
Business window appropriate?
```

---

# 42. Failback risk

Every additional transition introduces risk.

Therefore:

> **Don't fail back simply because the original region is available.**

Fail back when it is operationally safe.

---

# 43. Split-brain concern

In distributed systems, you must avoid two locations independently believing they are the authoritative write primary.

Conceptually:

```text id="r3c8m6"
Region A → writes
Region B → also writes
```

This can create conflicting state.

Your architecture must define authoritative write ownership.

---

# 44. DR and data consistency

Ask:

```text id="m8q2x5"
Is replication synchronous?
Is it asynchronous?
What lag exists?
What happens during network interruption?
```

The answers directly affect RPO.

---

# 45. Network partition

Suppose:

```text id="x4n7p2"
Region A  X  Region B
```

Replication stops.

You need an explicit policy for:

```text id="c6m3v9"
Who becomes primary?
Can writes continue?
How is divergence handled?
```

---

# 46. DR cost

DR architecture isn't free.

You may pay for:

```text id="p9v2m5"
Secondary compute
Storage
Networking
Replication
Monitoring
Operations
Testing
```

The design should reflect business criticality.

---

# 47. Tiered DR

Not every database needs identical recovery.

Example:

```text id="k7m3x8"
Tier 0
→ Mission critical

Tier 1
→ Business critical

Tier 2
→ Important

Tier 3
→ Non-critical
```

Each tier gets an appropriate RPO/RTO and cost profile.

---

# 48. Example recovery matrix

| Tier     |      Example RTO |      Example RPO | Strategy                |
| -------- | ---------------: | ---------------: | ----------------------- |
| Critical |          Minutes |         Very low | Cross-region HA/DR      |
| High     |         < 1 hour |              Low | Regional recovery       |
| Medium   |            Hours |            Hours | Backup-based recovery   |
| Low      | Business-defined | Business-defined | Cost-optimized recovery |

These are **illustrative targets**, not Azure guarantees.

---

# 49. Backup-only DR

For lower-criticality workloads:

```text id="m4x8p1"
Primary
   |
Backups
   ↓
Secondary region/storage
```

During disaster:

```text id="q7n2c5"
Restore
 ↓
Validate
 ↓
Application reconnect
```

This can have a much higher RTO than active replication.

---

# 50. DR vs backup retention

Retention answers:

> How far back can I recover?

DR answers:

> How quickly can I restore service after a major failure?

These are different requirements.

---

# 51. Long-term retention

Some businesses need historical recovery for:

```text id="v5m8x3"
Compliance
Legal requirements
Financial records
Audit
```

That doesn't necessarily mean those backups provide rapid DR.

---

# 52. Security during DR

Don't forget:

```text id="x2p7m9"
Encryption
Identity
Secrets
Certificates
Firewall/network rules
Private endpoints
Audit
```

A secondary environment must meet the same security baseline.

---

# 53. Monitoring DR

Monitor:

```text id="n8c4m6"
Replication health
Lag
Backup health
Secondary availability
Storage
Connectivity
```

A DR environment shouldn't be invisible until disaster day.

---

# 54. DR alerting

Important alerts include:

```text id="m3v9q2"
Replication stopped
Replication lag exceeds threshold
Backup failure
Secondary unavailable
Authentication failure
Network connectivity failure
```

---

# 55. DR automation

Automate safe checks:

```text id="r6x2k8"
Verify secondary
Verify connectivity
Verify application deployment
Verify monitoring
Verify configuration
Verify recovery readiness
```

Automate failover only when the risk and business policy justify it.

---

# 56. Chaos / resilience testing

Mature teams deliberately test failure.

Conceptually:

```text id="p4n8m3"
Introduce controlled failure
       ↓
Observe
       ↓
Recover
       ↓
Measure
       ↓
Improve
```

The goal isn't to cause outages.

The goal is to discover weaknesses **before** a real outage does.

---

# 57. Common DR mistakes

### Mistake 1

> "We have backups, so we're covered."

Backups without tested restoration aren't a proven DR strategy.

---

### Mistake 2

> "We have a secondary region."

A secondary environment that can't serve the application isn't effective DR.

---

### Mistake 3

> "Failover is automated, so we're safe."

Automation can fail too.

---

### Mistake 4

> "Our RTO is 15 minutes."

Unless measured, it's an assumption.

---

### Mistake 5

> "The database failed over, so we're done."

Application dependencies may still be broken.

---

# 58. Interview question

### "What's the difference between RPO and RTO?"

> "RPO defines the maximum acceptable amount of data loss, while RTO defines the maximum acceptable time to restore service."

---

# 59. Interview question

### "Do backups provide high availability?"

> "No. Backups primarily provide recovery capability. High availability is designed to minimize service interruption during supported failures. Backups remain necessary for logical corruption, accidental deletion, and other recovery scenarios."

---

# 60. Interview question

### "How would you design DR for SQL MI?"

Strong answer:

```text id="c8m3x7"
1. Gather business RPO/RTO
2. Identify failure scenarios
3. Select appropriate HA/DR architecture
4. Establish cross-region recovery
5. Configure application connectivity
6. Protect dependencies
7. Monitor replication/backups
8. Create runbooks
9. Test failover
10. Measure actual RPO/RTO
11. Fix gaps
12. Retest
```

---

# 61. Interview question

### "What happens to an application during database failover?"

> "Existing connections may be interrupted, so the application must detect transient failures, reconnect through the appropriate endpoint, and retry safe operations according to a bounded retry policy. Critical operations should also be designed to avoid duplicate effects."

---

# 62. Interview question

### "How do you prove your DR strategy works?"

Best answer:

> "By testing it. I measure the time from failure to successful application transaction, determine the recovered data point, validate dependencies and monitoring, document the results, and compare measured RTO/RPO against business targets."

---

# 63. Production DR architecture

```text id="m7p2x9"
                    USERS
                      |
                 GLOBAL ENTRY
                      |
              +-------+-------+
              |               |
          REGION A         REGION B
          PRIMARY          SECONDARY
              |               |
             SQL MI  <---->  SQL MI
              |               |
          Monitoring      Monitoring
              |               |
              +-------+-------+
                      |
                Central Ops
                      |
              Alerts / Runbooks
```

The exact architecture depends on the SQL MI features, application design, and required RPO/RTO.

---

# 64. Incident sequence

```text id="x3n8m5"
Failure detected
       ↓
Assess severity
       ↓
Estimate recovery time
       ↓
Compare with RTO
       ↓
Decision: recover/fail over
       ↓
Failover
       ↓
Application reconnect
       ↓
Validate transactions
       ↓
Business validation
       ↓
Monitor
       ↓
Stabilize
```

---

# 65. Golden rule

> **A DR design is not real until it has been successfully tested under realistic conditions.**

Remember:

```text id="q5m9c2"
RPO → How much data can we lose?

RTO → How long can we be down?

HA → How do we minimize interruption?

DR → How do we recover from major failure?

Backup → How do we recover data?

Testing → How do we prove the design works?
```

---

## Next — Lesson 112: Azure SQL Managed Instance Security — Advanced

We'll go deep into **identity, Microsoft Entra authentication, RBAC vs SQL permissions, managed identities, least privilege, networking security, private endpoints, encryption, TDE, Always Encrypted concepts, auditing, Defender for SQL, vulnerability management, privileged access, security monitoring, and production security architecture.**
