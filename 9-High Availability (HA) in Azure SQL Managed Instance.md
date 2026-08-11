Great. Let's continue with **Lesson 7 — High Availability (HA) in Azure SQL Managed Instance**.

This is where you'll start thinking like a production DBA during an actual outage.

# Lesson 7 — Azure SQL Managed Instance High Availability

First, keep this distinction clear:

```text
HA  → Keep the service available
DR  → Recover from a larger/regional disaster
Backup → Recover lost/corrupted/deleted data
```

They solve different problems.

---

# 1. What happens when a SQL Server fails?

Traditional SQL Server might look like:

```text
SQL Server
     |
     X  Server failure
     |
     v
Application DOWN
```

To avoid this, traditional environments might use:

```text
Failover Cluster Instance
Always On Availability Groups
Log Shipping
Database Mirroring
```

With Azure SQL Managed Instance, Azure provides a managed HA architecture so you don't have to build and operate the underlying cluster infrastructure yourself.

---

# 2. Think in terms of replicas

Conceptually:

```text
                 SQL Managed Instance
                        |
              +---------+---------+
              |                   |
              v                   v
           Primary             Secondary
              |                   |
              |   Replication     |
              +-------------------+
```

Your application connects to the MI endpoint rather than directly managing individual underlying servers.

If the underlying infrastructure fails, Azure can perform failover according to the service's HA architecture.

---

# 3. What does "managed" really mean?

This is important.

With traditional SQL Server, you might have to manage:

```text
Windows Server
Storage
Cluster
Patching
Failover configuration
Network interfaces
SQL Server installation
```

With MI:

```text
Azure
  ↓
Infrastructure
  ↓
Platform HA
  ↓
Failover
  ↓
Patching
```

Azure handles much of the underlying infrastructure management.

You still manage:

```text
Databases
Schemas
Queries
Indexes
Users
Permissions
Applications
Workload
```

So:

> **Managed Instance removes infrastructure work; it doesn't eliminate DBA work.**

---

# 4. General Purpose HA

Conceptually:

```text
              MI
               |
        Managed HA layer
               |
        +------+------+
        |             |
        v             v
     Compute       Storage
```

The architecture is designed to provide high availability while keeping costs lower than the highest-performance architecture.

The exact underlying implementation differs from Business Critical.

---

# 5. Business Critical HA

Business Critical is designed for more demanding workloads.

Conceptually:

```text
                   Application
                        |
                        v
                     Primary
                        |
               +--------+--------+
               |                 |
               v                 v
           Secondary 1       Secondary 2
```

Business Critical uses multiple replicas and local SSD storage.

This provides a stronger architecture for workloads where:

```text
Low latency
High performance
High availability
Read scale-out
```

are particularly important.

---

# 6. Why replicas matter

Imagine:

```text
PRIMARY
   |
   | Transactions
   v
SECONDARY
```

The secondary maintains a synchronized copy of the database according to the HA architecture.

If the primary infrastructure fails:

```text
Primary
   |
   X
Failure
   |
   v
Secondary
   |
   v
New primary
```

The application can reconnect through the managed endpoint.

---

# 7. Planned vs unplanned failover

This is an important distinction.

## Planned failover

You intentionally initiate it.

Example:

```text
Saturday maintenance
        |
        v
Planned failover
        |
        v
Secondary becomes primary
```

You might do this for:

* Maintenance testing
* DR testing
* Infrastructure changes
* Operational validation

---

## Unplanned failover

Something fails unexpectedly.

For example:

```text
Primary infrastructure
       |
       X
Hardware / platform failure
       |
       v
Azure detects failure
       |
       v
Failover
```

The purpose is to minimize downtime.

---

# 8. Production outage simulation

Let's simulate ABC Retail.

At 10:15 AM:

```text
Application
    |
    v
SQL MI Primary
```

Everything is normal.

At 10:16 AM:

```text
Primary infrastructure failure
```

The DBA sees:

```text
Application errors
```

What should you do?

**Don't immediately start modifying the database.**

First determine:

```text
Is the failure application-wide?
Is MI experiencing an outage?
Is the issue regional?
Is failover occurring?
Can the application reconnect?
```

---

# 9. Automatic failover

Suppose Azure detects an underlying failure.

Conceptually:

```text
10:16:00
Primary fails

10:16:xx
Azure detects problem

10:16:xx
Failover begins

10:16:xx
Secondary becomes primary

10:16:xx
Application reconnects
```

The exact duration depends on the failure scenario and architecture.

Don't promise a specific number of seconds unless you're working from the documented SLA/behavior for the exact configuration.

---

# 10. Important DBA lesson

When HA fails over, the database isn't necessarily "broken."

You may see:

```text
Existing connections
        ↓
Disconnected
```

Applications using connection pooling should generally be designed to reconnect/retry appropriately.

This is why application retry logic matters.

---

# 11. Connection retry

Imagine:

```text
Application
    |
    v
SQL MI
    |
    X
Failover
```

The existing TCP connection may be broken.

The application needs to establish a new connection:

```text
Application
    |
    X
Old connection

Application
    |
    v
New connection
    |
    v
New primary
```

This is why production applications should use appropriate:

```text
Connection retry
Command timeout
Connection timeout
Transient fault handling
```

---

# 12. A common developer mistake

Developer says:

> "The database failed over, but our application crashed."

You investigate.

MI:

```text
Healthy
```

Application:

```text
Connection errors
```

Possible problem:

```text
Application doesn't retry connections.
```

This is an **application resilience problem**, not necessarily a database availability problem.

---

# 13. Read scale-out

Now let's revisit Business Critical.

Suppose:

```text
OLTP workload
+
Heavy reporting
```

If reporting runs against the primary:

```text
Primary
  |
  +-- Customer transactions
  +-- Orders
  +-- Payments
  +-- Reporting queries
```

Heavy reporting can compete for:

```text
CPU
Memory
IO
Locks
```

With read scale-out:

```text
                    SQL MI
                      |
              +-------+-------+
              |               |
              v               v
           Primary        Readable secondary
              |               |
              v               v
             OLTP         Reporting
```

That can help isolate read-heavy workloads.

---

# 14. But don't confuse HA secondary with backup

This is worth repeating.

Suppose:

```sql id="s7f8z2"
UPDATE Customers
SET CreditLimit = 0;
```

was accidentally committed.

That transaction can be replicated to the HA secondary.

So:

```text
Primary
  |
  | Bad UPDATE
  v
Secondary
  |
  v
Bad UPDATE
```

Therefore:

> **HA replicas aren't a substitute for point-in-time recovery.**

---

# 15. HA vs DR

Here's the easiest way to visualize it:

```text
                  DATABASE
                     |
          +----------+----------+
          |          |          |
          v          v          v
         HA       Backup       DR
          |          |          |
          v          v          v
       Server/    Data loss   Region/
       platform   recovery    major disaster
       failure
```

### Example

Underlying infrastructure fails:

```text
HA
```

Developer deletes rows:

```text
PITR / Backup
```

Entire Azure region becomes unavailable:

```text
DR
```

---

# 16. Regional DR

Now:

```text
                 PRIMARY REGION
                     East US
                        |
                        v
                       MI
                        |
                        |
                     Replication
                        |
                        v
                 SECONDARY REGION
                     West US
                        |
                        v
                       MI
```

This is different from normal local HA.

Local HA handles failures **within the primary deployment architecture**.

Geo-DR protects against a much larger failure domain.

---

# 17. Production DR incident

At 3 PM:

```text
East US
   |
   X
Regional outage
```

Your secondary is:

```text
West US
```

The incident response becomes:

```text
Detect
  ↓
Confirm regional issue
  ↓
Evaluate secondary
  ↓
Check replication/RPO
  ↓
Fail over
  ↓
Redirect application
  ↓
Validate
```

Now your business requirement matters.

Suppose:

```text
RPO = 5 minutes
RTO = 30 minutes
```

You need to determine whether your architecture can realistically meet those requirements.

---

# 18. SLA vs your application RTO

This is another interview trap.

Someone says:

> "Azure provides high availability, so our application has zero downtime."

No.

Your real outage duration can include:

```text
Azure failover
+
Connection retry
+
Application recovery
+
DNS/endpoint behavior
+
Connection pool recovery
+
Business validation
```

Therefore:

> **Platform availability doesn't automatically equal application availability.**

You have to test the complete system.

---

# 19. HA testing

A professional DBA doesn't wait for a production failure to discover that the application doesn't reconnect.

You should test:

```text
Failover
   ↓
Application reconnect
   ↓
Transaction retry
   ↓
Connection pooling
   ↓
Monitoring
   ↓
Alerting
```

For example:

```text
Test:
10:00 AM — normal
10:01 AM — planned failover
10:02 AM — application reconnects
10:03 AM — transactions verified
10:05 AM — test complete
```

Document:

```text
Downtime
Errors
Reconnect time
Lost transactions
Application behavior
```

---

# 20. What should you monitor?

For a production MI environment, monitor at least:

```text
CPU
Memory
Storage
IO
Connections
Query performance
Blocking
Deadlocks
Failed connections
Database availability
Failover events
Replication/DR health
Backup health
```

And don't forget the application.

Because:

```text
SQL MI = Healthy
Application = Broken
```

is absolutely possible.

---

# 21. Real incident: "SQL MI is down"

Imagine your monitoring alerts:

```text
Application error rate = HIGH
```

Developer:

> "SQL Managed Instance is down."

You investigate:

```text
MI health       = Healthy
CPU             = 20%
Storage         = Healthy
```

Then you discover:

```text
Connection pool exhausted
```

The actual architecture is:

```text
Application
    |
    X
Connection pool
    |
    v
SQL MI
```

The database wasn't the problem.

This is why production troubleshooting must be **end-to-end**.

---

# 22. Interview question

### "What happens during MI failover?"

A strong answer:

> "The platform detects the failure and promotes another available replica according to the Managed Instance HA architecture. Existing connections may be interrupted, so applications must reconnect. I would monitor the failover event, validate application recovery, and check for transient errors or transaction retry requirements."

That's much better than:

> "Azure automatically changes the server."

---

# 23. Your complete HA mental model

Memorize this:

```text
                    SQL MI
                      |
            +---------+---------+
            |                   |
            v                   v
          HA                    DR
     Same deployment        Other region
            |                   |
            v                   v
      Infrastructure       Regional failure
        failure
```

And:

```text
Backup
   ↓
Data recovery
```

These are three separate protection mechanisms.

---

# Lesson 7 challenge 🧑‍💻

ABC Retail has:

```text
Business Critical MI
Heavy OLTP
Heavy reporting
DR region configured
```

At 11:30 AM:

> The primary experiences an infrastructure failure.

The application reports connection errors for a short period and then reconnects successfully.

### Answer these:

**1. Is this necessarily a database corruption event? Why?**

**2. What kind of event likely occurred?**

**3. Why might existing application connections fail even though MI eventually becomes available?**

**4. Why should the application have retry/reconnect logic?**

**5. If the entire Azure region fails, is local HA alone enough? What additional architecture is required?**

Next we'll tackle **Lesson 8 — Monitoring & Performance Troubleshooting**, where I'll teach you how to diagnose a real complaint like:

> **"Azure SQL Managed Instance is slow."**

We'll work through **CPU, memory, IO, waits, blocking, deadlocks, Query Store, execution plans, and Azure monitoring** step-by-step.
