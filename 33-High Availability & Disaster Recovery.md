# Lesson 65 — Azure SQL Managed Instance: High Availability & Disaster Recovery

Now we're moving from **performance** into one of the most important responsibilities of a production DBA:

> **What happens when the database, VM infrastructure, or even an entire Azure region has a failure?**

The first concept you need to separate is:

**HA ≠ DR ≠ Backup**

They solve different problems.

---

# 1. HA vs DR vs Backup

Think of three different failures.

### Failure A — infrastructure failure

```text
SQL workload
    ↓
Underlying infrastructure failure
    ↓
Service recovers/fails over
```

You want:

> **High Availability (HA)**

---

### Failure B — entire region unavailable

```text
Azure Region A
      ↓
Major outage
      ↓
Region unavailable
```

You want:

> **Disaster Recovery (DR)**

---

### Failure C — developer deletes data

```text
2:17 PM
DELETE
   ↓
Data damaged
```

You want:

> **Backup / Point-in-Time Recovery / logical recovery**

---

# 2. The easiest way to remember

| Requirement | Primary purpose                         |
| ----------- | --------------------------------------- |
| HA          | Survive infrastructure/service failures |
| DR          | Survive major/regional disasters        |
| Backup/PITR | Recover data                            |

So:

```text
HA → availability
DR → disaster
Backup → data recovery
```

Don't tell a business:

> "We have backups, therefore we have DR."

That's not necessarily true.

---

# 3. Real enterprise architecture

Imagine a company operating in the US.

Primary:

```text id="x1q7a2"
Azure Region A
    |
    +-- Application
    |
    +-- SQL Managed Instance
```

DR:

```text id="r9m4k6"
Azure Region B
    |
    +-- DR architecture
```

Conceptually:

```text id="w3p8n1"
             Users
               |
               v
          Application
               |
               v
       Primary Region
               |
               v
          SQL MI
               |
               | Replication / DR mechanism
               |
               v
         Secondary Region
```

The exact architecture depends on your required RPO/RTO and supported MI capabilities.

---

# 4. What is RPO?

RPO:

> **Recovery Point Objective**

It answers:

> "How much data can the business afford to lose?"

Example:

```text id="m2s8v4"
RPO = 5 minutes
```

The business says:

> "If disaster happens, we can tolerate losing at most approximately five minutes of transactions."

---

# 5. What is RTO?

RTO:

> **Recovery Time Objective**

It answers:

> "How quickly must the service come back?"

Example:

```text id="q7n3p5"
RTO = 30 minutes
```

Meaning:

> The business expects service restoration within approximately 30 minutes.

---

# 6. RPO and RTO are not the same

Imagine:

```text id="b4k9w2"
Disaster
  |
  v
12:00 PM
```

If your recovered data is:

```text id="c8m1r6"
11:55 AM
```

you lost about:

```text id="j3x7v9"
5 minutes
```

That's an **RPO** discussion.

If service comes back at:

```text id="p5q2z8"
12:20 PM
```

that's a **20-minute recovery time**.

That's an **RTO** discussion.

---

# 7. Business requirements come first

Don't begin by asking:

> "Which Azure feature should we use?"

Begin with:

```text id="d6f1y4"
How much data loss is acceptable?
How much downtime is acceptable?
```

For example:

### System A — internal reporting

```text id="e8m3q7"
RPO = 1 hour
RTO = 4 hours
```

### System B — payments

```text id="k2r9s5"
RPO = near-zero/very low
RTO = minutes
```

They should not necessarily have the same DR architecture.

---

# 8. HA inside the primary environment

For HA, you want the service to tolerate certain infrastructure failures without requiring a full regional recovery process.

Conceptually:

```text id="n7c4x2"
Primary Azure Region
        |
        +-- SQL MI
             |
             +-- HA infrastructure
```

The platform handles much of the underlying availability architecture.

Your application should therefore be designed to tolerate a database connection interruption/failover appropriately.

---

# 9. Important application behavior

Suppose a failover occurs.

The application may see:

```text id="z8v3m5"
Connection lost
```

That doesn't necessarily mean:

```text id="g1q6r4"
Database permanently unavailable
```

It may mean:

```text id="s5k9w2"
Transient interruption
       ↓
Reconnect
       ↓
Continue
```

This is why application connection logic matters.

---

# 10. Retry logic

A production application should generally be designed to handle appropriate transient failures.

Conceptually:

```text id="r3n7p8"
Request
  |
  v
Database connection
  |
  X transient failure
  |
  v
Short backoff
  |
  v
Retry
  |
  v
Success
```

But be careful.

This is bad:

```text id="f8m2q1"
Failure
 ↓
Retry immediately
 ↓
Failure
 ↓
Retry immediately
 ↓
Failure
 ↓
1000 clients retry simultaneously
```

That's a **retry storm**.

---

# 11. Better retry strategy

Use concepts such as:

```text id="k6p1v4"
Limited retries
Exponential backoff
Jitter
Timeouts
Circuit breaker
Idempotency where applicable
```

For example:

```text id="w4x9m7"
Retry 1 → 100 ms
Retry 2 → 300 ms
Retry 3 → 1 sec
```

The exact values depend on the application and service behavior.

---

# 12. Regional disaster scenario

Now the serious incident.

At 2:00 AM:

```text id="b2m7q9"
Azure Region A
       ↓
Major regional outage
```

Your primary application is unavailable.

Monitoring shows:

```text id="x8c3p1"
SQL MI unavailable
Application unavailable
```

The DBA's first question should **not** be:

> "Can I restart SQL?"

The region itself is down.

---

# 13. Disaster recovery runbook

Your runbook should look roughly like:

```text id="q9v2m5"
1. Confirm regional outage
        ↓
2. Declare incident
        ↓
3. Assess business impact
        ↓
4. Validate DR readiness
        ↓
5. Initiate failover according to architecture
        ↓
6. Redirect application traffic
        ↓
7. Validate database
        ↓
8. Validate application
        ↓
9. Validate business transactions
        ↓
10. Monitor secondary environment
```

---

# 14. Don't forget DNS/application routing

Suppose the application points directly to the primary database endpoint.

After disaster:

```text id="c4n8y1"
Application
    |
    X
Primary region
```

Your DR design needs a mechanism for the application to connect to the appropriate secondary environment after failover.

Depending on the architecture, this can involve supported failover mechanisms, application configuration, DNS/routing, or other traffic-management components.

The important principle is:

> **Database DR is useless if the application doesn't know where to go.**

---

# 15. DR testing

This is extremely important.

Imagine management says:

> "We have a DR environment."

Ask:

> "When was the last successful failover test?"

If nobody knows:

```text id="g5v2m9"
DR status = assumption
```

not proof.

A good DR program periodically tests:

```text id="p1r6x3"
Failover
Application connectivity
Authentication
Jobs
Reports
Integrations
Performance
Data consistency
Monitoring
Failback
```

---

# 16. DR test example

Suppose the DR exercise is scheduled for Saturday.

Before:

```text id="v4m8q2"
Primary → Healthy
Secondary → Ready
```

Test:

```text id="s7k1n5"
Failover
   ↓
Secondary becomes active
   ↓
Application redirected
   ↓
Smoke tests
   ↓
Business validation
```

Then:

```text id="c3x9p6"
Failback
```

You don't just test:

> "Can the database start?"

You test:

> **"Can the business operate?"**

---

# 17. Backup vs DR — real scenario

Developer accidentally runs:

```sql id="j6m2r8"
DELETE FROM Orders;
```

At 3:00 PM.

DR replica:

```text id="n5q8v1"
also contains the deletion
```

Why?

Because replication/DR protects availability, but it can also replicate **logical mistakes**.

That's why:

```text id="a4p7x2"
DR ≠ backup
```

You still need backups/PITR for accidental data modification.

---

# 18. Regional disaster vs logical disaster

### Regional disaster

```text id="w9k3m6"
Primary region unavailable
        ↓
DR/failover
```

### Accidental DELETE

```text id="r2v8q5"
Database available
but data is wrong
        ↓
PITR / logical recovery
```

Very different problems.

---

# 19. What if DR is behind?

Suppose:

```text id="h4n7p2"
Primary = 10:00:00
Secondary = 09:58:30
```

Your effective data-loss exposure could be approximately:

```text id="m8q1x6"
90 seconds
```

That's part of your RPO discussion.

You need monitoring around replication/DR health rather than assuming:

> "Secondary exists, so we're protected."

---

# 20. DR architecture must match business requirements

Imagine the business requires:

```text id="x7p3m9"
RPO = 1 minute
RTO = 10 minutes
```

But your design realistically provides:

```text id="q5n8v2"
RPO = 30 minutes
RTO = 2 hours
```

That's not a technical success.

It's a **business requirement failure**.

---

# 21. Interview question

### "What is the difference between HA and DR?"

Strong answer:

> "HA is designed to maintain service availability through failures within the primary environment or service architecture, minimizing interruption. DR is designed to recover the workload from a larger disaster, potentially including regional loss. HA primarily addresses availability; DR addresses disaster recovery. Neither replaces backups for logical data corruption or accidental data changes."

---

# 22. Interview question

### "Can DR replace backups?"

Answer:

> **No.**

Because:

```text id="c8r2m5"
DR protects availability
Backup protects recoverability
```

A bad transaction can be replicated to the DR environment.

---

# 23. Interview question

### "What are RPO and RTO?"

Use this answer:

> "RPO is the maximum amount of data loss the business can tolerate, measured in time. RTO is the maximum acceptable time to restore the service. I use those business requirements to determine the appropriate HA, DR, backup, and failover architecture."

---

# 24. The 2 AM regional outage

Let's put everything together.

### 02:00

```text id="z3m8k1"
Region A outage
```

### 02:03

```text id="j6p4x9"
Incident declared
```

### 02:05

```text id="w7q2n5"
DR readiness validated
```

### 02:07

```text id="b8v3m6"
Failover initiated
```

### 02:10

```text id="r4n9p1"
Database available in DR
```

### 02:12

```text id="k5x7q3"
Application redirected
```

### 02:15

```text id="m2c8v4"
Smoke tests passed
```

### 02:20

```text id="s6j1n9"
Business transactions validated
```

### 02:30

```text id="h7q3m5"
Service restored
```

You achieved approximately:

```text id="e9x2k7"
RTO ≈ 30 minutes
```

Then you assess the actual data synchronization state to determine whether the RPO requirement was met.

---

# 25. The senior DBA mindset

When someone says:

> "We have HA."

Ask:

> **"What failure does it protect us from?"**

When someone says:

> "We have DR."

Ask:

> **"What's our tested RPO and RTO?"**

When someone says:

> "We have backups."

Ask:

> **"When did we last perform a successful restore?"**

Those three questions will expose a lot of weak disaster-recovery designs.

---

# 26. Your HA/DR mental model

Memorize:

```text
                    BUSINESS
                       |
                 RPO / RTO
                       |
          +------------+------------+
          |                         |
         HA                         DR
          |                         |
   Primary availability      Regional/major
                              disaster recovery
          |                         |
          +------------+------------+
                       |
                    BACKUPS
                       |
              Data recovery
              / PITR / restore
```

And:

```text
HA ≠ DR
DR ≠ Backup
Backup ≠ Application recovery
```

All four layers need to work together.

---

# Next — Lesson 66

We'll go into **Azure SQL Managed Instance security architecture**, including:

* Microsoft Entra authentication
* managed identities
* SQL logins vs Entra identities
* server principals vs database users
* least privilege
* RBAC vs SQL permissions
* Key Vault
* encryption
* TDE
* Always Encrypted
* auditing
* Defender/security monitoring
* and a real incident where an application suddenly gets **"Login failed"** after a deployment.
