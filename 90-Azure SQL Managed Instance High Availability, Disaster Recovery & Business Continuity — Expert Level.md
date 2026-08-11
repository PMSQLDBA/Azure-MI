
# Lesson 123 — Azure SQL MI Backup & Restore Architecture

Backup strategy:

```text
Full backups
   +
Differential backups
   +
Transaction-log backups
```

Key concepts:

* Point-in-time restore
* Backup retention
* Restore targets
* Recovery objectives
* Geo-redundant backup considerations
* Restore testing
* Backup monitoring

### Expert principle

> A backup that has never been restored is an assumption, not a proven recovery mechanism.

---

# Lesson 124 — Azure SQL MI Performance Architecture

Performance begins with architecture:

```text
Application
    ↓
Connection layer
    ↓
SQL MI
    ↓
Compute
    ↓
Memory / Storage
```

Analyze:

* CPU
* Memory pressure
* I/O latency
* Query duration
* Throughput
* Blocking
* Wait statistics
* TempDB behavior
* Storage performance

### Key principle

> Don't optimize SQL queries before determining where the bottleneck actually is.

---

# Lesson 125 — SQL Server Wait Statistics

Wait statistics help answer:

> **What is SQL Server waiting for?**

Common categories include:

```text
CPU-related waits
I/O waits
Locking waits
Memory waits
Parallelism-related waits
Network waits
```

A useful investigation:

```text
High latency
   ↓
Wait analysis
   ↓
Identify bottleneck
   ↓
Validate with workload
   ↓
Optimize
```

---

# Lesson 126 — Query Store Architecture

Query Store provides historical query-performance information.

Think:

```text
Query
 ↓
Execution
 ↓
Performance history
 ↓
Plan history
 ↓
Regression analysis
```

Use it to identify:

* Query regressions
* Plan changes
* High-duration queries
* High-resource queries
* Changing workloads

---

# Lesson 127 — Query Performance Tuning

A disciplined tuning process:

```text
Identify query
 ↓
Measure baseline
 ↓
Inspect execution plan
 ↓
Check statistics
 ↓
Check indexes
 ↓
Check waits
 ↓
Change
 ↓
Measure again
```

Never assume an index is the solution.

---

# Lesson 128 — Execution Plans

Execution plans explain how SQL Server executes a query.

Look for:

```text
Table scans
Index scans
Index seeks
Large sorts
Hash operations
Key lookups
Excessive joins
Cardinality estimation problems
```

### Expert rule

> A visually complicated execution plan isn't automatically a bad execution plan.

Measure actual cost and workload impact.

---

# Lesson 129 — Indexing Strategy

Indexes improve reads but introduce costs:

```text
Storage
INSERT overhead
UPDATE overhead
DELETE overhead
Maintenance
Memory consumption
```

Think:

```text
Query workload
     ↓
Access patterns
     ↓
Indexes
```

Avoid creating indexes simply because a column "looks important."

---

# Lesson 130 — Statistics & Cardinality Estimation

SQL Server needs estimates of:

```text
Rows
Selectivity
Data distribution
```

Bad estimates can produce poor plans.

Symptoms:

```text
Expected: 10 rows
Actual:   10 million rows
```

This can radically change execution behavior.

---

# Lesson 131 — Parameter Sniffing

SQL Server may compile a plan using parameter values observed during compilation.

A plan optimized for:

```text
Customer A → 5 rows
```

may perform badly for:

```text
Customer B → 50 million rows
```

Diagnose before attempting fixes.

Potential approaches depend on workload and SQL Server behavior.

---

# Lesson 132 — Blocking & Locking

Blocking occurs when one session prevents another from proceeding.

Conceptually:

```text
Transaction A
     |
     | holds lock
     ↓
Transaction B
     |
    WAIT
```

Investigate:

```text
Who is blocking?
What resource?
Why is the transaction open?
How long?
```

---

# Lesson 133 — Deadlocks

Deadlock:

```text
Transaction A
   ↓ locks X
   ↓ waits for Y

Transaction B
   ↓ locks Y
   ↓ waits for X
```

SQL Server detects the cycle and chooses a victim.

Prevention strategies include:

* Consistent access order
* Short transactions
* Appropriate indexing
* Reduced lock duration
* Correct isolation strategy

---

# Lesson 134 — Transaction Isolation Levels

Isolation determines how transactions interact.

Common levels:

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SNAPSHOT
SERIALIZABLE
```

Higher isolation can provide stronger consistency but may increase contention or resource requirements.

### Expert principle

> Choose isolation based on business correctness, not simply "maximum safety."

---

# Lesson 135 — TempDB Architecture

TempDB supports many operations:

```text
Temporary objects
Sorts
Hash operations
Version stores
Intermediate results
```

Investigate TempDB when you see:

```text
Unexpected spills
Heavy temporary-object usage
Version-store pressure
Contention
```

---

# Lesson 136 — SQL MI Resource Governance

Enterprise environments may contain:

```text
Application A
Application B
Reporting
ETL
Administrative workloads
```

One workload shouldn't blindly consume resources needed by another.

Use appropriate SQL/application architecture and workload controls.

---

# Lesson 137 — Connection Pooling

Opening a database connection for every request can be expensive.

Prefer:

```text
Application
   ↓
Connection pool
   ↓
Reusable connections
   ↓
SQL MI
```

But oversized pools can create excessive concurrency.

---

# Lesson 138 — Connection Pool Failure Modes

A badly configured application can cause:

```text
Connection storm
      ↓
SQL MI overloaded
      ↓
Requests slow
      ↓
More retries
      ↓
More connections
      ↓
System collapse
```

This is a classic cascading failure.

---

# Lesson 139 — Application Retry Architecture

Retries should use:

```text
Bounded retries
Exponential backoff
Jitter
Transient-error detection
Idempotency
```

Avoid:

```text
while(true)
    retry();
```

---

# Lesson 140 — SQL MI Monitoring Architecture

Monitor multiple layers:

```text
Azure platform
      ↓
SQL instance
      ↓
Database
      ↓
Query
      ↓
Application
```

Metrics alone aren't enough.

Combine:

```text
Metrics
Logs
Query Store
Traces
Alerts
```

---

# Lesson 141 — Observability & Distributed Tracing

A request may travel:

```text
User
 ↓
API
 ↓
Service
 ↓
SQL MI
 ↓
External dependency
```

A database slowdown must be correlated back to the original request.

Use correlation IDs and distributed tracing where supported by the application stack.

---

# Lesson 142 — SQL MI Alerting Strategy

Don't alert on everything.

Separate:

### Warning

```text
Elevated CPU
Increasing latency
```

### Critical

```text
Service unavailable
Severe connection failure
Security incident
```

Alerts should have:

```text
Owner
Severity
Runbook
Escalation
```

---

# Lesson 143 — Capacity Planning

Capacity planning asks:

> **When will the workload outgrow the current architecture?**

Track:

```text
CPU trend
Memory trend
Storage growth
IOPS
Throughput
Connections
Query latency
Database size
```

Trend data is more useful than one day's snapshot.

---

# Lesson 144 — Performance Baselines

Create a baseline:

```text
Normal CPU
Normal latency
Normal throughput
Normal waits
Normal connections
Normal I/O
```

Then detect:

```text
Current
  ↓
Baseline
  ↓
Deviation
```

Without a baseline, "high CPU" is difficult to interpret.

---

# Lesson 145 — SQL MI Cost Optimization

Cost optimization should consider:

```text
Compute
Storage
Backup
Networking
Monitoring
Secondary environments
Operational overhead
```

Don't optimize only the Azure invoice.

A cheaper database that causes major application latency may increase total business cost.

---

# Lesson 146 — FinOps for SQL MI

A mature FinOps model connects:

```text
Azure cost
     ↓
Application
     ↓
Business unit
     ↓
Customer / product
```

Use tagging and cost allocation to understand:

> **Who consumes the database platform and why?**

---

# Lesson 147 — Enterprise SQL MI Landing Zone

A mature SQL MI platform can be standardized:

```text
Management groups
      ↓
Subscriptions
      ↓
Networking
      ↓
Identity
      ↓
Security policies
      ↓
SQL MI
      ↓
Monitoring
      ↓
Governance
```

Teams should consume a **standard platform**, rather than reinventing database infrastructure.

---

# Lesson 148 — SQL MI Platform Engineering

Platform engineering turns repeated DBA/cloud work into reusable services.

For example:

```text
Developer
   ↓
Platform request
   ↓
Standard SQL MI deployment
   ↓
Security automatically applied
   ↓
Monitoring automatically enabled
   ↓
Database ready
```

The platform team provides **guardrails plus self-service**.

---

# Lesson 149 — Enterprise SQL MI Operating Model

A mature operating model defines ownership for:

```text
Infrastructure
Database
Security
Application
Networking
Identity
Monitoring
Backup
DR
Cost
Compliance
```

Use clear responsibility boundaries.

A useful model is:

```text
Platform team
    +
DBA team
    +
Application team
    +
Security team
    +
SRE/Operations
```

---

# Lesson 150 — The Complete Azure SQL MI Expert Architecture

This is the culmination of the previous lessons.

```text
                         USERS
                           |
                    Global Traffic
                           |
                    Application Tier
                           |
                +----------+----------+
                |                     |
             Identity              Network
                |                     |
          Entra / MI          Private connectivity
                |                     |
                +----------+----------+
                           |
                      Azure SQL MI
                           |
        +------------------+------------------+
        |                  |                  |
     Security          Performance        Data
        |                  |                  |
   Encryption          Query Store       Backups
   Auditing            Wait stats        PITR
   Defender            Indexes           DR
   Least privilege     Statistics        Recovery
        |                  |                  |
        +------------------+------------------+
                           |
                     Observability
                           |
              +------------+------------+
              |            |            |
            Logs         Metrics      Traces
              |            |            |
              +------------+------------+
                           |
                      Operations
                           |
              +------------+------------+
              |            |            |
             IaC         CI/CD        Policy
              |            |            |
              +------------+------------+
                           |
                       Governance
                           |
             +-------------+-------------+
             |             |             |
          Security       Cost        Compliance
             |             |             |
             +-------------+-------------+
                           |
                    Business Continuity
                           |
                    +------+------+
                    |             |
                   HA            DR
                    |             |
               Local failure   Regional failure
                    |             |
                    +------+------+
                           |
                    Tested Recovery
```

## The expert mental model

At this point, don't think of SQL MI as simply:

```text
"SQL Server in Azure"
```

Think of it as:

```text
                SQL MI PLATFORM

       ┌────────────────────────────┐
       │       GOVERNANCE           │
       ├────────────────────────────┤
       │        SECURITY            │
       ├────────────────────────────┤
       │     IDENTITY / NETWORK     │
       ├────────────────────────────┤
       │      SQL MI PLATFORM       │
       ├────────────────────────────┤
       │ PERFORMANCE / DATA         │
       ├────────────────────────────┤
       │ OBSERVABILITY / OPERATIONS │
       ├────────────────────────────┤
       │       BACKUP / DR          │
       ├────────────────────────────┤
       │       APPLICATION          │
       └────────────────────────────┘
```

The important transition is:

> **Junior:** "How do I configure SQL MI?"

> **Intermediate:** "How do I operate SQL MI?"

> **Senior:** "How do I design SQL MI for reliability, security, performance and cost?"

> **Expert:** **"How do I build a standardized, automated SQL MI platform that remains secure, observable, performant, recoverable, governable and cost-effective throughout its entire lifecycle?"**

---
