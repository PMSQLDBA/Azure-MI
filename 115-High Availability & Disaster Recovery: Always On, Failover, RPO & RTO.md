# Lesson 148 — High Availability & Disaster Recovery: Always On, Failover, RPO & RTO

Today we're moving from **performance and concurrency** into one of the most important production DBA areas:

> **What happens when the database server, storage, or entire site fails?**

A production SQL Server environment needs more than backups. You need a strategy for **availability, recovery, and business continuity**.

---

## 1. HA vs DR

These terms are related but different.

### High Availability — HA

Goal:

> **Keep the service available when a component fails.**

Example:

```text
Primary SQL Server
      ↓ failure
Secondary SQL Server
      ↓
Application continues
```

### Disaster Recovery — DR

Goal:

> **Recover the service after a major failure or disaster.**

Examples:

```text
Data center failure
Region outage
Major corruption
Ransomware
Infrastructure destruction
```

---

# 2. RPO

**RPO = Recovery Point Objective**

It answers:

> **"How much data can we afford to lose?"**

Example:

```text
RPO = 5 minutes
```

means the business can tolerate losing approximately five minutes of recent transactions in the worst-case recovery scenario.

---

# 3. RTO

**RTO = Recovery Time Objective**

It answers:

> **"How long can the service be unavailable?"**

Example:

```text
RTO = 30 minutes
```

means the business expects service to be restored within roughly 30 minutes.

---

# 4. RPO vs RTO

Remember:

```text
RPO
→ How much data can we lose?

RTO
→ How much downtime can we tolerate?
```

They're independent.

You could have:

```text
RPO = 1 minute
RTO = 4 hours
```

or:

```text
RPO = 1 hour
RTO = 5 minutes
```

depending on the architecture and business requirements.

---

# 5. Start with business requirements

Don't begin with:

> "Let's configure Always On."

Start with:

```text
Business requirements
       ↓
RPO
RTO
       ↓
Architecture
       ↓
Implementation
       ↓
Testing
```

Technology should support the requirement.

---

# 6. SQL Server Always On

SQL Server provides several technologies under the **Always On** umbrella.

The two major ones you'll encounter are:

```text
Always On Availability Groups
Failover Cluster Instances
```

They solve different problems.

---

# 7. Availability Groups

An Availability Group provides replicated databases across SQL Server instances.

Conceptually:

```text
             Availability Group
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
      Primary              Secondary
       Server                Server
          │                   │
       Database             Replica
```

---

# 8. Primary replica

The primary replica handles the normal read/write workload.

```text
Application
     ↓
Primary replica
     ↓
Read + Write
```

---

# 9. Secondary replica

A secondary replica maintains a copy of the database.

Depending on configuration, it can potentially support:

```text
Read-only workloads
Backups
Disaster recovery
Failover
```

---

# 10. Synchronous vs asynchronous commit

This distinction is critical.

### Synchronous commit

The primary waits for the secondary to harden the transaction log before completing the commit.

Conceptually:

```text
Application
    ↓
Primary
    ↓
Log
    ↓
Secondary hardens log
    ↓
Commit completes
```

This provides strong protection against data loss between synchronized replicas, assuming the appropriate configuration and failure scenario.

---

# 11. Asynchronous commit

With asynchronous commit:

```text
Primary
   ↓
Commit
   ↓
Continue

Secondary
   ↓
Receives log later
```

The primary doesn't wait for the secondary to confirm hardening.

This is useful when replicas are geographically distant or network latency makes synchronous operation impractical.

---

# 12. The trade-off

### Synchronous

Pros:

```text
Very low data-loss exposure
```

Cons:

```text
Potential commit latency
Network dependency
```

### Asynchronous

Pros:

```text
Lower primary commit latency
Good for remote DR
```

Cons:

```text
Potential data loss
```

---

# 13. Synchronous doesn't mean "no possible data loss"

Be careful with absolute statements.

Synchronous replication protects against certain failure scenarios when the secondary has hardened the log.

But a complete disaster involving multiple replicas, corruption, operator error, or other conditions can require backups and additional DR controls.

That's why:

> **HA is not a replacement for backups.**

---

# 14. Backups remain essential

Even with Availability Groups, maintain appropriate:

```text
Full backups
Differential backups
Transaction log backups
```

according to your recovery model and business requirements.

---

# 15. Why replicas don't replace backups

Imagine:

```text
Accidental DELETE
```

The bad transaction can be replicated to every secondary.

Now:

```text
Primary
   ↓
DELETE replicated
   ↓
Secondary
```

Replication doesn't know:

> "That DELETE was a mistake."

Backups provide a historical recovery path.

---

# 16. Human error

Suppose someone runs:

```sql
DELETE FROM Customers;
```

without a proper WHERE clause.

An HA secondary won't magically protect you from the logical mistake.

You need recovery mechanisms such as:

```text
Backups
Point-in-time restore
Operational controls
```

---

# 17. Point-in-time recovery

If you have:

```text
Full backup
+
Transaction log backups
```

you can potentially restore to a specific point in time within the available log-backup chain.

Conceptually:

```text
Full backup
    ↓
Log 1
    ↓
Log 2
    ↓
Log 3
    ↓
Bad DELETE
```

You can restore to a point immediately before the unwanted action, assuming the required backup chain is intact.

---

# 18. Recovery model

For point-in-time recovery with regular transaction log backups, SQL Server commonly uses:

```text
FULL recovery model
```

The recovery model is therefore directly connected to your backup and DR strategy.

---

# 19. Full recovery does not mean "automatic backups"

This is an important distinction.

Setting:

```text
FULL recovery
```

doesn't automatically give you point-in-time recovery.

You still need a functioning log-backup schedule.

---

# 20. Log chain

Think of transaction log backups as a chain:

```text
Full
 ↓
Log A
 ↓
Log B
 ↓
Log C
 ↓
Log D
```

If the required chain is broken, your restore options may be affected.

---

# 21. Tail-log backup

In certain disaster scenarios, a **tail-log backup** can capture the most recent transaction-log records that haven't yet been backed up.

This can be important for minimizing data loss during recovery.

The exact feasibility depends on the failure scenario and database state.

---

# 22. Failover

Failover means the workload moves from one replica/server to another.

Conceptually:

```text
Before:

Application
    ↓
Primary A

After:

Application
    ↓
Primary B
```

---

# 23. Automatic vs manual failover

Depending on architecture and configuration, failover can be:

```text
Automatic
```

or:

```text
Manual
```

Automatic failover can reduce downtime, but requires careful configuration and health detection.

---

# 24. Failover is not the same as recovery

A failover may take:

```text
Seconds / minutes
```

while restoring a large database from backups could take:

```text
Hours
```

They're different mechanisms serving different failure scenarios.

---

# 25. Failover Cluster Instance

An FCI protects the SQL Server instance using clustered infrastructure.

Conceptually:

```text
SQL Server Instance
       ↓
Shared storage
       ↓
Cluster nodes

Node A → active
Node B → standby
```

If the active node fails, the instance can fail over to another node.

---

# 26. FCI vs Availability Group

A simplified comparison:

| Feature            | Availability Group         | Failover Cluster Instance   |
| ------------------ | -------------------------- | --------------------------- |
| Protection         | Database-level             | Instance-level              |
| Copies             | Separate database replicas | Shared storage architecture |
| Readable secondary | Possible                   | Not the same model          |
| Storage model      | Separate replica storage   | Shared storage              |
| DR flexibility     | Strong                     | Different use case          |

The exact capabilities depend on SQL Server edition/version and architecture.

---

# 27. Availability Groups are not "just backups"

An AG maintains synchronized/replicated copies of databases.

That's primarily about:

```text
Availability
Failover
Read scaling possibilities
DR
```

Backups are still a separate recovery mechanism.

---

# 28. Listener

An Availability Group can use a listener endpoint so applications don't have to hard-code a specific replica server.

Conceptually:

```text
Application
     ↓
AG Listener
     ↓
Current primary
```

After failover:

```text
Application
     ↓
Same listener
     ↓
New primary
```

This simplifies application connectivity.

---

# 29. Read-only routing

In appropriate configurations, read-only workloads can be routed to readable secondary replicas.

For example:

```text
Write traffic
     ↓
Primary

Read-only reporting
     ↓
Secondary
```

This can reduce load on the primary.

---

# 30. But secondaries aren't free

A secondary consumes:

```text
CPU
Memory
Storage
Network
SQL Server resources
```

If it's also serving reports, you must account for that workload.

---

# 31. Reporting on a secondary

Imagine:

```text
Primary
→ OLTP

Secondary
→ Huge reporting queries
```

The reporting workload can consume significant resources.

So "offloading reads" doesn't mean:

> "No performance impact."

It means moving the impact elsewhere.

---

# 32. Synchronization health

For Availability Groups, monitor things such as:

```text
Synchronization state
Synchronization health
Redo lag
Log send queue
Redo queue
Connection health
```

The exact metrics available depend on configuration/version.

---

# 33. Log send queue

Conceptually:

```text
Primary
   ↓
Log generated
   ↓
Not yet sent
   ↓
Log send queue
```

A growing queue can indicate the secondary is falling behind.

---

# 34. Redo queue

A secondary may receive log records but still need to apply them.

Conceptually:

```text
Received
   ↓
Waiting to be redone
   ↓
Redo queue
```

A large redo queue indicates apply lag.

---

# 35. Data loss exposure

For asynchronous replication:

```text
Primary log
    ↓
Secondary is behind
```

If the primary is permanently lost, the unsent/unhardened portion may be unavailable on the secondary.

That is your potential data-loss window.

---

# 36. Network latency

Geographic DR introduces:

```text
Distance
+
Network latency
+
Bandwidth constraints
```

For asynchronous replication, this is easier to tolerate.

For synchronous replication, network latency can directly affect transaction commit performance.

---

# 37. RPO/RTO architecture example

Suppose the business says:

```text
RPO ≤ 1 minute
RTO ≤ 5 minutes
```

You might consider:

```text
Local HA
+
Synchronous secondary
+
Remote DR
+
Frequent backups
+
Automated monitoring
```

But the final architecture depends on budget, workload, geography, and operational requirements.

---

# 38. Cost matters

A highly available architecture may require:

```text
Additional SQL Server licenses
Additional servers
Storage
Network bandwidth
Monitoring
Operational expertise
Testing
```

The correct design balances:

```text
Business value
vs
Complexity
vs
Cost
```

---

# 39. DR testing

This is critical:

> **A backup that has never been restored is an assumption, not proof of recoverability.**

You need restore testing.

---

# 40. Restore drills

A proper DR test might verify:

```text
Backup exists
   ↓
Restore succeeds
   ↓
Database consistency verified
   ↓
Application connects
   ↓
Data validated
   ↓
Performance acceptable
   ↓
RTO measured
```

---

# 41. Measure the RTO

Don't say:

> "We can recover in 30 minutes."

Measure it.

Example:

```text
Start: 02:00
Restore begins: 02:05
Database online: 02:23
Application validated: 02:31
```

Actual RTO:

```text
31 minutes
```

Now you know whether you're meeting the requirement.

---

# 42. Measure the RPO

Likewise:

```text
Last recoverable transaction:
10:58:42
Failure:
11:00:00
```

Potential data-loss window:

```text
~1 minute 18 seconds
```

Measure it rather than assuming it.

---

# 43. Backup monitoring

A backup job saying:

```text
"Success"
```

isn't enough.

Monitor:

```text
Backup age
Backup duration
Backup size
Failures
Log backup continuity
Restore testing
Storage availability
```

---

# 44. Encryption and security

DR copies must be protected too.

Consider:

```text
Encryption at rest
Encryption in transit
Backup encryption
Credential security
Access control
Key management
```

A secondary server containing production data is still production data.

---

# 45. Ransomware consideration

If an attacker gains access to the production environment, blindly replicating everything can spread damage.

Therefore:

```text
HA
≠
Security isolation
```

Backups should be designed with resilience against malicious deletion or tampering where appropriate.

---

# 46. Immutable/offline backup considerations

For serious disaster recovery planning, organizations may maintain backups with additional protection against alteration or deletion.

The architecture depends on the organization's security requirements and infrastructure.

The important principle:

> **Your recovery mechanism should survive the failure you're designing it for.**

---

# 47. Multi-region DR

A remote replica can protect against:

```text
Site failure
Regional outage
Infrastructure disaster
```

But introduces:

```text
Network latency
Bandwidth cost
Operational complexity
Potential data-loss window
```

---

# 48. HA doesn't protect every failure

Suppose someone executes:

```sql id="r1w8k3"
DROP DATABASE Production;
```

If the command succeeds and replicates through the HA mechanism, your secondary isn't necessarily a historical recovery system.

Again:

```text
HA → availability
Backup → recovery
```

You generally need both.

---

# 49. Application-level failover

Database failover isn't enough if the application can't reconnect.

The application needs:

```text
Retry behavior
Connection timeout strategy
Appropriate listener configuration
Transient-failure handling
```

---

# 50. Connection retries

After failover, existing connections may be broken.

Applications should be designed to:

```text
Detect transient failure
 ↓
Reconnect
 ↓
Retry safely where appropriate
```

This is especially important for connection pools.

---

# 51. Transaction retry

Suppose failover occurs during a transaction.

The application may receive an error.

It may need to:

```text
Rollback/abandon current transaction
Reconnect
Re-execute safely
```

Again:

> **Idempotency matters.**

---

# 52. Common HA/DR mistake

Bad mindset:

> "We have an Availability Group, so we're covered."

Better:

```text
HA
+
Backups
+
Restore testing
+
Monitoring
+
Application recovery
+
DR plan
```

---

# 53. Another common mistake

Bad:

> "Our backups are successful."

Better:

> "We have successfully restored our backups and measured the recovery process."

---

# 54. Interview question

### "What is RPO?"

Strong answer:

> "Recovery Point Objective defines the maximum acceptable amount of data loss, expressed in time. For example, a five-minute RPO means the organization aims to recover with no more than approximately five minutes of recent data potentially lost."

---

# 55. Interview question

### "What is RTO?"

Strong answer:

> "Recovery Time Objective defines the maximum acceptable time to restore service after a failure."

---

# 56. Interview question

### "Does Always On replace backups?"

Strong answer:

> "No. Availability technologies improve availability and failover, but they don't provide historical recovery from logical errors such as accidental deletes or corruption. Backups remain essential."

---

# 57. Interview question

### "Synchronous vs asynchronous replication?"

Strong answer:

> "Synchronous commit waits for the secondary to harden the transaction log before the primary completes the commit, providing strong protection against certain data-loss scenarios at the cost of potential latency. Asynchronous commit allows the primary to proceed without waiting, which is useful for remote replicas but introduces a potential replication/data-loss window."

---

# 58. Expert architecture model

Think in layers:

```text
                 BUSINESS
                    │
             RPO / RTO
                    │
                    ↓
             HA + DR DESIGN
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
       HA        Backups       DR
        │           │           │
      Failover    Restore     Remote site
        │           │           │
        └───────────┼───────────┘
                    ↓
              TEST + MONITOR
```

---

# 59. Lesson 148 takeaway

Remember these distinctions:

```text
HA
→ Keep service running

DR
→ Recover from major failure

RPO
→ How much data can we lose?

RTO
→ How long can we be down?

Availability Group
→ Replicated database availability

Backup
→ Historical recovery
```

The strongest production design isn't:

> **"We have a replica."**

It's:

> **"We know our RPO and RTO, have an architecture that meets them, have tested failover and restore procedures, and know exactly how the application behaves during recovery."**

---
**Next:** **Lesson 149**
**Final:** **Lesson 150**
