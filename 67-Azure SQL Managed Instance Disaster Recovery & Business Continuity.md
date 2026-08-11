# Lesson 99 — Azure SQL Managed Instance Disaster Recovery & Business Continuity

This lesson connects **high availability, backups, disaster recovery, failover, RPO/RTO, applications, networking, and business continuity** into one operational model.

The key principle:

> **Disaster recovery is not simply having another database. It is having a tested way to restore the entire business service.**

---

# 1. HA vs DR

These are different objectives.

### High Availability

Protects against failures while keeping service available with minimal interruption.

```text id="m1"
Primary SQL MI
      |
   HA architecture
      |
      ↓
Automatic/managed recovery
```

### Disaster Recovery

Protects against a larger failure such as:

```text id="m2"
Region outage
Major platform disruption
Large-scale infrastructure failure
```

Conceptually:

```text id="m3"
Region A
  |
  | DR relationship
  ↓
Region B
```

---

# 2. Backup + HA + DR

A mature architecture often looks like:

```text id="m4"
                 PROTECTION
                     |
       +-------------+-------------+
       |             |             |
     Backup          HA            DR
       |             |             |
   Data recovery   Local        Regional
   / PITR          failure      failure
```

No single mechanism covers every failure mode.

---

# 3. RPO

Recovery Point Objective asks:

> **How much data can the business afford to lose?**

Examples:

```text id="m5"
RPO = 5 minutes
RPO = 15 minutes
RPO = 1 hour
```

Lower RPO generally requires stronger synchronization/replication and operational design.

---

# 4. RTO

Recovery Time Objective asks:

> **How quickly must the service be restored?**

Example:

```text id="m6"
RTO = 30 minutes
```

This includes more than database startup.

It may include:

```text id="m7"
Database failover
Network
DNS
Authentication
Application restart/reconnect
Validation
```

---

# 5. The real DR chain

Think:

```text id="m8"
Database
   ↓
Network
   ↓
Identity
   ↓
Application
   ↓
Integration
   ↓
Users
```

If any critical component cannot recover, the business service may remain unavailable.

---

# 6. Example

Suppose SQL MI fails over successfully.

But the application uses:

```text id="m9"
Hard-coded private IP address
```

The database is healthy.

The application still can't connect.

Therefore:

> **Database DR succeeded, but application DR failed.**

---

# 7. DR architecture

A typical conceptual design:

```text id="m10"
              PRIMARY REGION
              ┌──────────────┐
              │ Application  │
              │ SQL MI       │
              └──────┬───────┘
                     │
                Replication
                     │
                     ↓
              SECONDARY REGION
              ┌──────────────┐
              │ Application  │
              │ SQL MI       │
              └──────────────┘
```

The exact Azure implementation depends on the SQL MI capabilities and architecture you select.

---

# 8. Synchronous vs asynchronous thinking

Replication can involve trade-offs.

### Synchronous-style protection

Potentially:

```text id="m11"
Lower data-loss exposure
+
Higher dependency on network latency
```

### Asynchronous replication

Potentially:

```text id="m12"
Greater geographic flexibility
+
Possible replication lag
+
Potential data loss during disaster
```

For regional DR, understand the RPO implications of replication lag.

---

# 9. Replication lag

Suppose:

```text id="m13"
Primary:
Transaction 1000 committed

Secondary:
Transaction 980 received
```

The secondary is behind.

Difference:

```text id="m14"
20 transactions
```

If the primary region suddenly disappears, those transactions may not yet exist on the secondary.

That's an RPO concern.

---

# 10. DR is a business decision

Suppose management says:

> "We can't lose any data."

Your next questions should be:

```text id="m15"
What does "any" mean?
Which transactions?
What latency is acceptable?
What is the budget?
What downtime is acceptable?
```

Perfect zero-loss regional DR may have significant architectural and operational implications.

---

# 11. DR tiers

Organizations often classify workloads.

Example:

### Tier 0

```text id="m16"
Mission critical
Very low RPO
Very low RTO
```

### Tier 1

```text id="m17"
Business critical
Low RPO
Low RTO
```

### Tier 2

```text id="m18"
Important
Moderate recovery objectives
```

### Tier 3

```text id="m19"
Non-critical
Longer recovery acceptable
```

Not every database needs the same DR architecture.

---

# 12. Application connection handling

One of the most important DR considerations:

> **Can the application reconnect after failover?**

Applications should generally use an appropriate connection mechanism that supports the intended failover architecture rather than relying on a fixed database IP.

---

# 13. Connection strings

A robust application design considers:

```text id="m20"
Server/endpoint
Authentication
Encryption
Retry behavior
Connection timeout
Command timeout
Failover behavior
```

---

# 14. Retry logic

Imagine a transient failover:

```text id="m21"
Application request
      ↓
Connection failure
      ↓
Failover
      ↓
Retry
      ↓
Success
```

Good retry handling can turn a brief infrastructure event into a minor user-visible interruption.

Bad retry logic can create a:

```text id="m22"
Retry storm
```

---

# 15. Retry storm

Suppose 10,000 application threads simultaneously retry:

```text id="m23"
Failure
 ↓
Retry immediately
 ↓
Failure
 ↓
Retry immediately
 ↓
Failure
```

This can overwhelm the recovering system.

Use controlled retry strategies such as:

```text id="m24"
Backoff
Jitter
Retry limits
Transient-error detection
```

---

# 16. Failover types

Conceptually distinguish:

### Planned failover

Used for:

```text id="m25"
DR testing
Maintenance
Planned regional movement
```

### Unplanned/forced failover

Used when:

```text id="m26"
Primary region is unavailable
```

Planned failover should generally be tested and coordinated.

---

# 17. Planned failover

A controlled sequence:

```text id="m27"
Notify users
   ↓
Stop/change application traffic
   ↓
Confirm replication health
   ↓
Fail over
   ↓
Validate secondary
   ↓
Redirect/confirm application
   ↓
Run smoke tests
```

---

# 18. Forced failover

During a major outage:

```text id="m28"
Primary unavailable
       ↓
Assess secondary
       ↓
Initiate DR
       ↓
Application reconnect
       ↓
Validate service
```

You may have less ability to preserve the newest transactions.

That is the RPO trade-off.

---

# 19. DR runbook

A production DR runbook should contain:

```text id="m29"
1. Incident declaration
2. Decision authority
3. Recovery target
4. Failover procedure
5. Application procedure
6. Networking procedure
7. Authentication procedure
8. Validation
9. Communication
10. Failback procedure
```

---

# 20. Decision authority

Don't let five teams independently decide:

> "Let's fail over."

Define who has authority.

Example:

```text id="m30"
Incident Commander
       |
       +-- DBA
       +-- Application
       +-- Network
       +-- Security
```

---

# 21. Failover decision

Before failing over, ask:

```text id="m31"
Is primary truly unavailable?
Is secondary healthy?
Is replication sufficiently current?
What data loss is expected?
Can the application run from secondary?
```

---

# 22. The danger of premature failover

Suppose primary is experiencing:

```text id="m32"
Temporary network issue
```

but is still processing transactions.

If you immediately force DR:

```text id="m33"
Primary transactions
       ↓
Not yet replicated
       ↓
Secondary becomes primary
```

Potential data loss or split-brain-style operational confusion can occur depending on the architecture.

Failover is a controlled decision.

---

# 23. Split-brain concept

The dangerous situation is:

```text id="m34"
Region A thinks:
"I am primary."

Region B thinks:
"I am primary."
```

Now both sides can accept writes.

This can create conflicting data.

A well-designed managed DR architecture uses mechanisms to prevent or control this scenario, but your operational runbook must still account for application traffic and decision authority.

---

# 24. DNS

Many DR architectures rely on stable endpoints, DNS, or application configuration.

After failover, verify:

```text id="m35"
DNS resolution
Endpoint
Private DNS
Application configuration
Network routes
```

Don't assume DNS automatically solves everything.

---

# 25. Private networking

If SQL MI uses private networking, DR may require:

```text id="m36"
VNet configuration
Private endpoints/connectivity as applicable
DNS
Routing
Firewall/security rules
ExpressRoute/VPN
NSGs where relevant
```

Database replication alone doesn't reproduce network architecture.

---

# 26. Identity

The application may authenticate using:

```text id="m37"
Managed identity
Service principal
SQL authentication
Microsoft Entra authentication
```

During DR, verify that required identities and permissions work in the secondary environment.

---

# 27. Secrets

Applications may depend on:

```text id="m38"
Passwords
Certificates
Keys
Connection strings
Secrets
```

If those aren't available in the DR environment:

```text id="m39"
Database = healthy
Application = broken
```

---

# 28. External dependencies

Your application may depend on:

```text id="m40"
Payment gateway
Message queue
API
Storage
Identity provider
Third-party service
```

DR must include these dependencies where they are critical.

---

# 29. DR architecture diagram

Think beyond SQL:

```text id="m41"
        PRIMARY REGION
 ┌─────────────────────────┐
 │ App                     │
 │   ↓                     │
 │ SQL MI                  │
 │   ↓                     │
 │ External services       │
 └──────────┬──────────────┘
            │
         Replication
            │
            ↓
       SECONDARY REGION
 ┌─────────────────────────┐
 │ App                     │
 │   ↓                     │
 │ SQL MI                  │
 │   ↓                     │
 │ External services       │
 └─────────────────────────┘
```

---

# 30. DR testing

A DR strategy that has never been tested is theoretical.

Test:

```text id="m42"
Failover
Application reconnection
DNS
Authentication
Transactions
Monitoring
Alerts
Backups
Failback
```

---

# 31. Measure the test

Don't just say:

> "DR worked."

Record:

```text id="m43"
Detection time
Decision time
Failover time
Application recovery time
Validation time
Total RTO
Estimated data loss
```

---

# 32. Example DR drill

Target:

```text id="m44"
RTO = 30 minutes
RPO = 5 minutes
```

Actual:

```text id="m45"
Detection = 5 min
Decision = 4 min
Failover = 6 min
Application = 8 min
Validation = 10 min
```

Total:

```text id="m46"
33 minutes
```

Result:

> RTO target was missed by 3 minutes.

That's actionable.

---

# 33. DR monitoring

During normal operations, monitor:

```text id="m47"
Replication health
Replication lag
Secondary health
Connectivity
Capacity
Application readiness
Backup health
```

Don't wait for the disaster to discover replication is broken.

---

# 34. Secondary capacity

A common mistake:

```text id="m48"
Primary:
Large capacity

Secondary:
Tiny capacity
```

Then disaster occurs.

Secondary becomes primary and immediately struggles.

DR capacity must satisfy the recovery workload.

---

# 35. Cost vs recovery objectives

Better DR usually costs more.

You may need:

```text id="m49"
Additional compute
Storage
Networking
Monitoring
Operational effort
Testing
```

Therefore:

> **DR architecture should be driven by business criticality and RPO/RTO, not by technology enthusiasm.**

---

# 36. Failback

After the primary region recovers, don't immediately switch back.

First:

```text id="m50"
Primary restored
      ↓
Replication re-established
      ↓
Data synchronized
      ↓
Application validated
      ↓
Planned failback
```

---

# 37. Why failback can be harder than failover

After DR activation:

```text id="m51"
Secondary is now serving writes.
```

When primary returns, you need to ensure:

```text id="m52"
New transactions
       ↓
Synchronize back
       ↓
Primary becomes ready
```

Never assume the original primary can simply be switched back on.

---

# 38. Failback planning

Define:

```text id="m53"
Who decides?
When?
How is synchronization verified?
How is application traffic moved?
How is downtime minimized?
How is success validated?
```

---

# 39. Business continuity

Disaster recovery is one part of business continuity.

Business continuity asks:

> **How does the organization continue operating during disruption?**

That can include:

```text id="m54"
Technology
People
Processes
Facilities
Vendors
Communications
```

---

# 40. DR vs business continuity

```text id="m55"
Business Continuity
       |
       +-- People
       +-- Process
       +-- Technology
       +-- Vendors
       |
       +-- Disaster Recovery
```

DR is therefore a component of the larger continuity strategy.

---

# 41. Scenario — regional outage

Imagine:

```text id="m56"
Region A
SQL MI + application
```

suddenly becomes unavailable.

---

## Step 1 — Declare incident

```text id="m57"
Confirm outage
Open incident
Assign incident commander
```

---

## Step 2 — Assess secondary

Check:

```text id="m58"
Replication health
Last replicated data
Capacity
Connectivity
Application readiness
```

---

## Step 3 — Estimate RPO

Suppose last confirmed replicated data:

```text id="m59"
19:12
```

Incident begins:

```text id="m60"
19:15
```

Potential data-loss exposure:

```text id="m61"
~3 minutes
```

That should be communicated.

---

# 42. Step 4 — Fail over

Execute the approved DR procedure.

```text id="m62"
Secondary
   ↓
Become active
```

---

# 43. Step 5 — Application

Confirm:

```text id="m63"
Connection endpoint
DNS
Authentication
Network
Application startup
```

---

# 44. Step 6 — Business validation

Test:

```text id="m64"
Login
Create order
Update customer
Process payment
Generate report
```

Don't stop at:

```text id="m65"
"SQL connection succeeded."
```

---

# 45. Step 7 — Monitor

After failover:

```text id="m66"
CPU
Memory
Storage
Latency
Connections
Errors
Blocking
Application throughput
```

The secondary may have a different workload profile.

---

# 46. Step 8 — Communicate

Tell stakeholders:

```text id="m67"
DR activated
Current service status
Estimated data-loss window
Known limitations
Next update
```

---

# 47. Step 9 — Stabilize before failback

Don't immediately return to Region A because:

> "It's back online."

First determine:

```text id="m68"
Root cause
Infrastructure health
Replication
Data consistency
Capacity
```

Then plan failback.

---

# 48. DR checklist

```text id="m69"
ARCHITECTURE
☐ Primary defined
☐ Secondary defined
☐ Replication configured
☐ Network available
☐ Identity available
☐ Application available

MONITORING
☐ Replication monitored
☐ Lag monitored
☐ Alerts configured
☐ Secondary capacity monitored

RUNBOOK
☐ Failover steps
☐ Validation steps
☐ Communication steps
☐ Failback steps

TESTING
☐ Planned failover tested
☐ Application reconnect tested
☐ DNS tested
☐ Authentication tested
☐ Business transactions tested
☐ RTO measured
☐ RPO measured
```

---

# 49. Interview question

### "What's the difference between HA and DR?"

Strong answer:

> "High availability is primarily designed to maintain service through localized failures with minimal interruption. Disaster recovery addresses larger failures, such as a regional outage, and provides a strategy for restoring service elsewhere."

---

# 50. Interview question

### "What is RPO?"

> "Recovery Point Objective is the maximum acceptable amount of data loss, usually expressed as a time interval."

Example:

```text id="m70"
RPO = 10 minutes
```

---

# 51. Interview question

### "What is RTO?"

> "Recovery Time Objective is the maximum acceptable time to restore the service after an outage."

---

# 52. Interview question

### "What should you test during DR?"

Strong answer:

```text id="m71"
1. Database failover.
2. Replication health.
3. Application connectivity.
4. DNS/networking.
5. Authentication.
6. Critical transactions.
7. Monitoring and alerting.
8. External dependencies.
9. Failback.
10. Actual RPO/RTO.
```

---

# 53. Interview question

### "The secondary database is healthy. Is DR ready?"

Not necessarily.

You also need:

```text id="m72"
Application
Network
DNS
Identity
Secrets
Dependencies
Monitoring
Runbook
People
```

A database alone isn't the business service.

---

# 54. Interview question

### "Why test failback?"

Because:

```text id="m73"
Failover succeeds
      ↓
Secondary becomes production
      ↓
Writes occur there
      ↓
Original primary recovers
```

You must ensure the original primary can safely rejoin and receive current data before moving production back.

---

# 55. The complete protection model

Memorize:

```text id="m74"
                   BUSINESS SERVICE
                         |
          +--------------+--------------+
          |              |              |
       DATABASE       APPLICATION     NETWORK
          |              |              |
     +----+----+         |              |
     |         |         |              |
 Backup       HA         |            DNS
     |         |         |          Identity
     |         |         |              |
     +---------+---------+--------------+
                         |
                         ↓
                        DR
                         |
                   Secondary Region
                         |
                         ↓
                    Failover
                         |
                         ↓
                    Validation
                         |
                         ↓
                    Business
                    Continuity
```

---

# 56. Final mental model

When someone asks:

> **"Are we protected?"**

Don't answer:

> "Yes, we have backups."

Instead ask:

```text id="m75"
Can we recover the data?
Can we recover the application?
Can we recover the network?
Can users authenticate?
Can critical transactions execute?
How much data can we lose?
How long will recovery take?
Have we tested it?
```

That is the difference between **having DR technology** and **having a real DR capability**.

---

## Next — Lesson 100: Azure SQL Managed Instance Security & Identity — Advanced DBA

We'll cover **Microsoft Entra authentication, managed identities, SQL authentication, RBAC vs database permissions, least privilege, contained users, server/database roles, auditing, Defender-related security concepts, encryption, TDE, TLS, secrets, privileged access, service principals, and a complete production security design.**
