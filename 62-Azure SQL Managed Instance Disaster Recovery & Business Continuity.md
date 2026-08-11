# Lesson 94 — Azure SQL Managed Instance Disaster Recovery & Business Continuity

This lesson connects everything we've covered so far.

A production DBA must be able to answer:

> **"What happens if the entire Azure region hosting our SQL MI becomes unavailable?"**

That is a different problem from a server failure, query problem, or accidental data deletion.

---

# 1. The three failure levels

Keep these separate:

```text
Failure
   |
   +-- Database/query problem
   |      ↓
   |   Backup / PITR
   |
   +-- Instance/infrastructure problem
   |      ↓
   |   HA
   |
   +-- Regional disaster
          ↓
          DR
```

---

# 2. HA vs DR

### High Availability

Protects against localized failures.

```text
Primary
   ↓
HA infrastructure
   ↓
Automatic recovery/failover
```

Goal:

> Keep the service available.

### Disaster Recovery

Protects against a larger disaster such as:

```text
Region outage
Major regional incident
Large-scale infrastructure failure
```

Goal:

> Recover the workload in another location.

---

# 3. The most important DR terms

You must know:

```text
RPO
RTO
Failover
Failback
DR region
Replication
Recovery
Runbook
DR test
```

---

# 4. RPO

**RPO = Recovery Point Objective**

It answers:

> **How much data can the business afford to lose?**

Example:

```text
RPO = 15 minutes
```

Means the business can tolerate approximately 15 minutes of data loss in the defined disaster scenario.

---

# 5. RTO

**RTO = Recovery Time Objective**

It answers:

> **How long can the business be unavailable?**

Example:

```text
RTO = 1 hour
```

Means the business expects recovery within approximately one hour.

---

# 6. RPO and RTO together

Suppose:

```text
RPO = 15 minutes
RTO = 1 hour
```

Your DR architecture needs to support:

```text
Data loss ≤ approximately 15 minutes
Recovery time ≤ approximately 1 hour
```

Don't confuse them.

```text
RPO → Data
RTO → Time
```

---

# 7. Simple example

Disaster happens:

```text
10:00
10:15
10:30
10:45
11:00
```

If:

```text
RPO = 15 minutes
```

you need a recovery design capable of limiting the data-loss window to roughly that level.

If:

```text
RTO = 1 hour
```

the service should be restored within the agreed recovery period.

---

# 8. DR architecture

A typical cross-region design looks conceptually like:

```text
Region A
   |
   | Primary
   ↓
SQL MI
   |
   | Replication / DR mechanism
   ↓
Region B
   |
   ↓
Secondary SQL MI
```

Region B exists to provide recovery capability if Region A suffers a major outage.

---

# 9. Why a second region?

Suppose both your:

```text
SQL MI
Backup-dependent recovery process
```

are effectively tied to the same regional disaster boundary.

A severe regional incident can affect your recovery options.

Cross-region DR provides geographic separation.

---

# 10. DR is not just SQL

This is a very common mistake.

You might have:

```text
SQL MI in Region A
SQL MI in Region B
```

and still have a broken DR architecture.

Why?

Because the application may depend on:

```text
Application servers
DNS
Key Vault
Storage
Identity
Networking
Load balancers
Secrets
Monitoring
External APIs
```

If those aren't recoverable, the application may still be unavailable.

---

# 11. Complete DR architecture

Think:

```text
                REGION A
                  |
        +---------+---------+
        |         |         |
       App       SQL      Storage
        |         |         |
        +---------+---------+
                  |
             DR mechanism
                  |
                  ↓
                REGION B
                  |
        +---------+---------+
        |         |         |
       App       SQL      Storage
```

Everything required for the application needs a recovery strategy.

---

# 12. Business continuity

Business continuity is broader than database DR.

It asks:

> **How does the business continue operating during a major disruption?**

For example:

```text
Database
Application
Users
Network
Identity
Operations
Support
Business processes
```

all need consideration.

---

# 13. DR tiers

Not every workload needs the same DR design.

### Tier 1

Mission critical:

```text
Very low RPO
Very low RTO
Cross-region DR
Frequent testing
```

### Tier 2

Important:

```text
Moderate RPO
Moderate RTO
Planned recovery
```

### Tier 3

Low criticality:

```text
Longer recovery
Backup-based recovery
Lower cost
```

---

# 14. Cost vs recovery

The more aggressive your DR requirements become:

```text
Lower RPO
+
Lower RTO
```

the more you generally invest in:

```text
Replication
Secondary infrastructure
Automation
Testing
Operational complexity
```

You shouldn't build the same DR architecture for every database.

---

# 15. Backup-based DR

One recovery approach is:

```text
Primary Region
      |
      ↓
Backups
      |
      ↓
Secondary Region
      |
      ↓
Restore
```

Advantages:

```text
Lower cost
Simple concept
Useful for many workloads
```

Disadvantages:

```text
Potentially higher RPO
Potentially longer RTO
Restore process required
```

---

# 16. Replication-based DR

Another approach:

```text
Primary
   |
   ↓
Continuous/near-continuous replication
   |
   ↓
Secondary
```

Advantages:

```text
Lower recovery time
Lower potential data-loss window
```

Disadvantages:

```text
More infrastructure
More operational complexity
Higher cost
```

---

# 17. Choosing the right architecture

Ask:

```text id="k7f3p9"
What is the business RPO?
What is the business RTO?
How critical is the application?
How much can we spend?
How often can we test?
What dependencies exist?
```

Then choose the design.

---

# 18. Planned vs unplanned failover

These are different.

### Planned failover

You know the event is coming.

Example:

```text
Scheduled regional maintenance
Planned DR test
Migration
```

You can prepare.

### Unplanned failover

Something has already gone wrong.

Example:

```text
Region outage
Major infrastructure incident
```

You need emergency recovery.

---

# 19. Planned failover

Typical process:

```text
1. Notify stakeholders
2. Validate secondary
3. Stop/coordinate writes
4. Ensure synchronization
5. Initiate failover
6. Redirect application
7. Validate
```

Goal:

> Minimize data loss during the controlled transition.

---

# 20. Unplanned failover

Typical process:

```text
Disaster
   ↓
Assess primary
   ↓
Determine secondary readiness
   ↓
Initiate DR failover
   ↓
Redirect application
   ↓
Validate
   ↓
Communicate recovery
```

The exact sequence depends on the DR technology and architecture.

---

# 21. Failover doesn't mean everything is finished

Suppose SQL MI has successfully failed over.

You still need:

```text
Application connectivity
DNS
Authentication
Secrets
Network routes
Connection strings
Monitoring
```

validated.

---

# 22. Connection strings

A common DR mistake:

```text
Application
   ↓
Hard-coded primary endpoint
```

If the primary region fails:

```text
Application
   ↓
Unavailable endpoint
```

The application doesn't automatically know where to go unless the architecture supports it.

---

# 23. Better application design

Use a connection strategy that supports the intended DR architecture.

Conceptually:

```text
Application
    ↓
Stable logical endpoint
    ↓
Current primary
```

During failover:

```text
Logical endpoint
    ↓
New primary
```

This minimizes application changes.

---

# 24. DNS

DNS can be part of the DR strategy.

Conceptually:

```text
db.company.internal
        |
        ↓
Primary Region
```

After failover:

```text
db.company.internal
        |
        ↓
DR Region
```

But DNS caching/TTL behavior must be considered.

---

# 25. Don't assume DNS is instant

Clients may cache DNS results.

Therefore:

```text
DNS change
   ↓
Some clients update immediately
Others may continue using cached information
```

This can affect real-world RTO.

---

# 26. DR runbook

Every critical system should have a documented runbook.

Example:

```text
DR RUNBOOK

1. Declare disaster
2. Identify incident commander
3. Confirm primary unavailable
4. Validate secondary health
5. Confirm replication status
6. Initiate failover
7. Update application routing
8. Validate authentication
9. Run smoke tests
10. Notify business
11. Monitor
12. Document
```

---

# 27. Why runbooks matter

During a disaster:

```text
Stress ↑
Confusion ↑
Communication load ↑
```

You don't want:

> "What do we do now?"

The runbook should answer it.

---

# 28. Runbook ownership

A runbook should have:

```text
Owner
Version
Last test date
Contacts
Dependencies
Rollback/failback process
Validation steps
```

---

# 29. DR testing

A DR architecture that has never been tested is only an assumption.

Test it.

At minimum, validate:

```text
Failover
Application connectivity
Authentication
Data integrity
DNS/routing
Monitoring
Business transactions
Failback
```

---

# 30. DR drill example

Suppose:

```text
Saturday 2:00 AM
```

You simulate:

```text
Region A unavailable
```

Then:

```text
2:05
Declare DR

2:10
Validate secondary

2:15
Failover

2:25
Application redirected

2:35
Business validation
```

Measured RTO:

```text
35 minutes
```

---

# 31. Compare against requirement

Business requirement:

```text
RTO ≤ 60 minutes
```

Actual:

```text
35 minutes
```

Result:

```text
PASS
```

But also measure:

```text
RPO achieved
```

---

# 32. Measuring RPO

Suppose the primary stopped at:

```text
10:00
```

Secondary contains data through:

```text
09:57
```

Approximate data gap:

```text
3 minutes
```

If:

```text
RPO requirement = 15 minutes
```

that's within the target.

---

# 33. Don't just test the database

During DR testing:

```text
SQL recovery = successful
```

doesn't necessarily mean:

```text
Application recovery = successful
```

Test the entire stack.

---

# 34. Application smoke test

After failover, test:

```text
Login
Create order
Read customer
Process payment
Generate invoice
Run critical report
```

If the business transaction works:

```text
Database + application integration
```

has been validated.

---

# 35. Data validation

After DR failover, check:

```text
Row counts
Critical transactions
Recent orders
Payments
Inventory
Sequences/identity behavior
Application consistency
```

Don't assume replication means the business state is correct without validation.

---

# 36. Monitoring the secondary

A secondary isn't useful if it's broken.

Monitor:

```text
Replication health
Synchronization state
Latency
Storage
CPU
Memory
Connectivity
```

---

# 37. Replication lag

Suppose:

```text
Primary
  |
  | 20 minutes behind
  ↓
Secondary
```

Your effective RPO may be worse than expected.

That's why replication monitoring matters.

---

# 38. DR monitoring

You want alerts for:

```text
Replication failure
Replication lag
Secondary unavailable
Storage exhaustion
Authentication problems
Network problems
```

---

# 39. Region outage scenario

Imagine:

```text
Region A
   ↓
Major Azure outage
```

Your application:

```text
Unavailable
```

DBA sees:

```text
Primary SQL MI unreachable
```

First:

```text
Don't panic.
```

Then:

```text
1. Confirm scope
2. Check Azure service health
3. Validate DR environment
4. Determine current replication state
5. Execute approved DR procedure
```

---

# 40. Azure Service Health

For Azure-wide incidents, check the relevant Azure service-status information.

You need to distinguish:

```text
Your configuration problem
```

from:

```text
Azure regional/platform incident
```

---

# 41. Don't immediately rebuild everything

A region outage doesn't automatically mean:

```text
Delete primary
Create new MI
Restore everything
```

That could create unnecessary risk.

First understand:

```text
Current DR architecture
Replication status
Available Azure capabilities
Business requirements
```

---

# 42. Failback

After the primary region recovers:

```text
Region A
   ↓
Recovered

Region B
   ↓
Current primary
```

You may eventually want:

```text
Region B
   ↓
Failback
   ↓
Region A
```

But don't rush.

---

# 43. Failback is another migration

Think of failback as:

```text
DR event
   ↓
Secondary becomes primary
   ↓
Stabilize
   ↓
Re-establish desired topology
   ↓
Synchronize
   ↓
Planned failback
```

It requires planning.

---

# 44. Never fail back blindly

Before failback:

```text
Validate Region A
Validate networking
Validate SQL MI
Validate application dependencies
Validate synchronization
Plan outage/change window
```

Then perform controlled failback.

---

# 45. Split-brain concern

A serious DR architecture must avoid two locations independently accepting conflicting writes.

Conceptually:

```text
Region A → writes
Region B → writes
```

can produce:

```text
Conflict
Conflict
Conflict
```

The architecture must establish which site is authoritative.

---

# 46. Single-primary model

A common conceptual model:

```text
Region A
 PRIMARY
    |
    ↓
Region B
 SECONDARY
```

Only the designated primary handles normal writes.

During failover:

```text
Region A
 SECONDARY / unavailable

Region B
 PRIMARY
```

---

# 47. DR security

Don't forget security in the secondary region.

It should have:

```text
Network security
Identity configuration
Permissions
Encryption
Secrets
Monitoring
Auditing
```

You don't want:

```text
Primary = secure
DR = forgotten
```

---

# 48. DR dependency inventory

Create a dependency list:

```text
Database
Application
DNS
Identity
Key Vault
Storage
Network
Certificates
Monitoring
External APIs
Batch jobs
Messaging
```

For each:

```text
Primary location
DR location
Recovery method
Owner
Test frequency
```

---

# 49. Example DR matrix

| Component   | Primary  | DR strategy                   |
| ----------- | -------- | ----------------------------- |
| SQL MI      | Region A | Secondary region              |
| Application | Region A | Deployable in Region B        |
| DNS         | Global   | Update/routing                |
| Secrets     | Region A | Recovery/replication strategy |
| Storage     | Region A | Geo-redundant strategy        |
| Monitoring  | Central  | Multi-region                  |
| Identity    | Entra    | Cloud service                 |

The exact architecture depends on the application.

---

# 50. RPO/RTO interview question

### "What is RPO?"

> "Recovery Point Objective is the maximum acceptable amount of data loss measured in time."

### "What is RTO?"

> "Recovery Time Objective is the maximum acceptable time required to restore the service after a disruption."

---

# 51. Interview question

### "Why isn't backup enough for every DR scenario?"

Strong answer:

> "Backups can provide recovery, but depending on the recovery method they may require significant restore time and may have a larger data-loss window. Workloads with aggressive RPO/RTO requirements may need a replication-based DR architecture."

---

# 52. Interview question

### "What should you test during a DR drill?"

Answer:

```text
Replication
Failover
Application connectivity
Authentication
DNS/routing
Data integrity
Business transactions
Monitoring
RTO
RPO
Failback
```

---

# 53. Interview question

### "What is the difference between planned and unplanned failover?"

> "Planned failover is a controlled transition where synchronization and application behavior can be prepared in advance. Unplanned failover occurs after an unexpected outage and may involve some data loss depending on the replication state and DR design."

---

# 54. Interview question

### "Why can a database failover successfully while the application remains down?"

Because:

```text
Database recovered
≠
Application recovered
```

Potential causes:

```text
Connection string
DNS
Network
Identity
Secrets
Certificates
Application servers
External dependencies
```

---

# 55. DR maturity levels

### Level 1 — No DR

```text
Backups only
```

### Level 2 — Documented recovery

```text
Backups
+
Runbook
```

### Level 3 — Tested recovery

```text
Backups
+
Runbook
+
Regular DR testing
```

### Level 4 — Active DR architecture

```text
Secondary environment
+
Replication
+
Automated/controlled failover
+
Testing
```

### Level 5 — Business continuity maturity

```text
Technical DR
+
Application DR
+
People/processes
+
Regular exercises
+
Measured RPO/RTO
```

---

# 56. Complete production DR exercise

Let's simulate one.

### Situation

Production SQL MI:

```text
Region A
```

DR SQL MI:

```text
Region B
```

Business requirements:

```text
RPO = 5 minutes
RTO = 30 minutes
```

---

## 57. Incident

At:

```text
03:00
```

Region A experiences a major outage.

Application:

```text
Unavailable
```

---

## 58. Step 1 — Confirm incident

Check:

```text
Azure service status
Application monitoring
Network
SQL MI connectivity
```

Confirm this is a regional issue.

---

## 59. Step 2 — Assess secondary

Check:

```text
Secondary online?
Replication healthy?
Replication lag?
Required databases present?
Application environment ready?
```

Suppose:

```text
Replication lag = 2 minutes
```

RPO target:

```text
5 minutes
```

Good.

---

## 60. Step 3 — Fail over

Initiate the approved DR failover process.

Now:

```text
Region B
    ↓
Primary
```

---

## 61. Step 4 — Redirect application

Update the application's routing/connection mechanism.

Then:

```text
Application
    ↓
Region B
    ↓
SQL MI
```

---

## 62. Step 5 — Validate

Run:

```text
Login
Create test transaction
Read customer
Read order
Verify payment workflow
```

Everything works.

---

## 63. Step 6 — Measure

Incident:

```text
03:00
```

Application restored:

```text
03:22
```

RTO:

```text
22 minutes
```

Target:

```text
30 minutes
```

Result:

```text
PASS
```

RPO:

```text
2 minutes
```

Target:

```text
5 minutes
```

Result:

```text
PASS
```

---

# 64. Step 7 — Communicate

The DBA shouldn't just fix the technical issue.

Communicate:

```text
Incident started
Current state
Failover completed
Data-loss assessment
Application status
Known limitations
Next update
```

---

# 65. Step 8 — Stabilize

Don't immediately fail back.

First:

```text
Monitor Region B
Confirm workload stability
Confirm backups
Confirm security
Confirm application health
```

---

# 66. Step 9 — Failback planning

After Region A recovers:

```text
Validate Region A
       ↓
Re-establish synchronization
       ↓
Plan maintenance window
       ↓
Controlled failback
```

---

# 67. Post-incident review

Ask:

```text
Did we meet RPO?
Did we meet RTO?
What took the longest?
Was the runbook accurate?
Did DNS behave as expected?
Did application authentication work?
Did anyone need undocumented access?
```

Then improve the architecture.

---

# 68. DR checklist

```text
BUSINESS
☐ RPO defined
☐ RTO defined
☐ Application criticality defined

ARCHITECTURE
☐ Secondary region
☐ Database recovery method
☐ Application recovery
☐ Network recovery
☐ Identity recovery
☐ Secrets recovery
☐ DNS/routing

OPERATIONS
☐ DR runbook
☐ Owners defined
☐ Contact list
☐ Monitoring
☐ Alerts

TESTING
☐ Failover tested
☐ Application tested
☐ RPO measured
☐ RTO measured
☐ Failback tested
☐ Results documented
```

---

# 69. The most important mental model

Memorize:

```text
                    DISASTER
                       |
             +---------+---------+
             |                   |
          Local             Regional
             |                   |
             ↓                   ↓
             HA                  DR
                                 |
                      +----------+----------+
                      |                     |
                 Replication            Backup
                      |                     |
                 Lower RTO/RPO        Potentially
                                      higher RTO/RPO
```

---

# 70. DBA golden rule

> **A DR solution isn't "configured" until it has been successfully tested.**

And:

```text
RPO = How much data can we lose?
RTO = How long can we be down?
```

If you remember those two questions, you'll understand the foundation of business continuity.

---

## Next — Lesson 95: Azure SQL Managed Instance Maintenance, Patching & Upgrades

We'll cover **Azure-managed patching, maintenance windows, planned maintenance, version upgrades, compatibility levels, breaking changes, pre-production testing, rollback planning, application compatibility, maintenance notifications, and how a DBA handles a production upgrade safely.**
