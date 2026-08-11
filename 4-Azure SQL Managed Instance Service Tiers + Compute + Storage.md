**Lesson 2: Azure SQL Managed Instance Service Tiers + Compute + Storage**.

## 1. The two main MI architectures

For your DBA mental model, start here:

```text
                 SQL Managed Instance
                         |
             +-----------+-----------+
             |                       |
             v                       v
      General Purpose        Business Critical
             |
             v
       Next-gen General
          Purpose
```

The tier you choose affects **performance, availability architecture, storage behavior, and cost**.

---

# 2. General Purpose

Think of General Purpose as:

> **"Good general-purpose SQL Server workload without requiring the highest performance architecture."**

Conceptually:

```text
             SQL MI Compute
                  |
                  |
                  v
           Remote / managed
              storage
                  |
       +----------+----------+
       |          |          |
      Data       Log       TempDB
```

Typical workloads:

* Internal business applications
* ERP
* CRM
* Moderate OLTP
* Development/test
* Applications where extreme I/O performance isn't required

### Example

ABC Retail has:

```text
SalesDB
500 GB
500 users
Moderate transaction volume
```

The application isn't extremely latency-sensitive.

General Purpose may be a reasonable starting point.

---

# 3. Business Critical

Now imagine a banking application:

```text
Transactions/sec: Very high
Latency requirement: Very low
Availability requirement: Very high
```

You don't want your architecture optimized primarily for cost.

You want:

```text
                SQL MI
                  |
          +-------+-------+
          |               |
          v               v
       Primary         Secondary
       replica          replica
          |
          v
      Local SSD
```

Business Critical is designed for demanding workloads and uses local SSD storage with multiple replicas. Microsoft documents a built-in secondary replica and read scale-out capability for Business Critical.

---

# 4. The important performance concept

Suppose your query does this:

```sql
SELECT *
FROM Orders
WHERE CustomerID = 100;
```

The query needs to access:

```text
SQL Engine
    ↓
Memory
    ↓
Storage
```

If storage latency is high:

```text
CPU:      30%
Memory:   40%
Storage:  Slow
```

Your application can still be slow.

So:

> **Low CPU does NOT automatically mean the database is healthy.**

This is a very important DBA lesson.

---

# 5. CPU vs IO — real production scenario

Suppose the application team tells you:

> "SQL MI is slow."

You check:

```text
CPU = 35%
```

You might think:

> "CPU isn't the problem."

Good.

Now check:

```text
IO latency = Very high
```

Now you've found a possible bottleneck.

The problem could be:

```text
Poor query
     ↓
Table scan
     ↓
Millions of pages read
     ↓
Storage pressure
     ↓
Application slowdown
```

Not:

```text
CPU shortage
```

This is why an Azure DBA needs to understand **workload characteristics**, not just vCores.

---

# 6. What are vCores?

Think of a vCore as a unit of allocated compute capacity.

Very simplified:

```text
2 vCores
   ↓
Small workload

8 vCores
   ↓
Medium workload

32+ vCores
   ↓
Large workload
```

But don't make this mistake:

> "My SQL Server has 32 CPUs, therefore I need 32 vCores."

Not necessarily.

You need to analyze:

```text
CPU utilization
Peak CPU
Concurrency
Query duration
IO
Memory pressure
Workload pattern
```

---

# 7. Real production sizing example

Your existing SQL Server:

```text
32 CPU
128 GB RAM
4 TB database
```

Average CPU:

```text
25%
```

Peak CPU:

```text
55%
```

Someone says:

> "Let's provision 32 vCores in Azure."

Stop.

You should first investigate:

```text
Why was the existing server sized at 32 CPU?

Was it actually using them?

Are there memory constraints?

Are queries IO-bound?

Is there batch processing?

Are there monthly peaks?

```

Maybe:

```text
16 vCores
```

is enough.

Or perhaps:

```text
32 vCores
```

is necessary during peak periods.

**Sizing should be workload-driven.**

---

# 8. Storage is another major consideration

Suppose:

```text
Database = 4 TB
```

Don't think:

> "I need 4 TB storage."

You need to consider:

```text
Data files
Log files
TempDB
Growth
Indexes
Maintenance
Future growth
Backup requirements
```

For example:

```text
Current data       4 TB
Annual growth      500 GB
Indexes/temp data  Additional capacity
Future requirement 3 years
```

Your storage planning must account for growth.

---

# 9. TempDB matters

Many SQL Server DBAs focus heavily on:

```text
User databases
```

and forget:

```text
tempdb
```

But tempdb can become heavily used by:

```text
Sorts
Hash joins
Temporary tables
Table variables
Version store
Online operations
Large queries
```

Imagine:

```sql
SELECT *
FROM Orders
ORDER BY OrderDate;
```

If SQL Server needs a large sort and doesn't have enough memory, it can spill to tempdb.

Then:

```text
Query
 ↓
Sort
 ↓
Memory insufficient
 ↓
TempDB spill
 ↓
Storage IO
 ↓
Query becomes slow
```

This is a classic production troubleshooting scenario.

---

# 10. Business Critical read scale-out

Here's an interesting production scenario.

Your primary database handles:

```text
OLTP
```

Meanwhile the reporting team runs:

```sql
SELECT
    CustomerID,
    SUM(OrderAmount)
FROM Orders
GROUP BY CustomerID;
```

Potentially millions of rows.

You don't want reporting queries fighting with OLTP.

Business Critical can provide a readable secondary replica for read-only workloads.

Conceptually:

```text
                 Application
                     |
                     v
                  Primary
                     |
             +-------+-------+
             |               |
             v               v
           OLTP          Read-only
                         Reporting
```

That's a very useful architecture.

---

# 11. But here's an interview trap

Interviewer:

> "Business Critical has a secondary replica. Can I use it for everything?"

No.

You need to understand:

**Secondary replicas used for read scale-out are intended for read-only workloads.**

Also, replicated data can have some lag.

Therefore:

```text
Critical transaction
      ↓
Must see latest data
      ↓
Primary
```

But:

```text
Analytics/reporting
      ↓
Can tolerate slight replication lag
      ↓
Readable secondary
```

That's the correct mental model.

---

# 12. Next-gen General Purpose

Microsoft has introduced **Next-gen General Purpose**, which changes the underlying architecture to provide improved performance and scalability compared with the classic General Purpose architecture.

For interviews, remember:

```text
Classic GP
     ↓
Remote storage architecture

Next-gen GP
     ↓
Newer architecture
     ↓
Improved performance/scalability
```

When choosing a tier in a real deployment, always check the **current regional availability, limits, pricing, and supported configurations**, because Azure capabilities evolve.

---

# 13. How I want you to think as a DBA

Don't start with:

> "Which tier is cheapest?"

Start with:

```text
               WORKLOAD
                  |
       +----------+----------+
       |          |          |
      CPU        IO       Latency
       |          |          |
       +----------+----------+
                  |
             Availability
                  |
                  ↓
              Service Tier
```

Then:

```text
Service Tier
     ↓
vCores
     ↓
Memory/compute
     ↓
Storage
     ↓
Performance
     ↓
Cost
```

---

# 14. Real project decision

Let's return to ABC Retail.

We have:

```text
Database:       4 TB
SQL Server:     2019
Users:          5,000
Application:    24/7
Downtime:       <10 minutes
DR:             Required
Reporting:      Heavy
Transactions:   High
```

I would **not** immediately choose a tier.

I'd first collect:

```text
CPU utilization
Memory utilization
IO latency
IOPS
Throughput
Database growth
Peak workload
Connection count
Top queries
Wait statistics
Blocking
Deadlocks
```

Then compare the workload against MI tiers.

That's how a production DBA makes the decision.

---

# Your Lesson 2 challenge 🧑‍💻

You are the DBA for ABC Retail.

You discover:

```text
CPU average:        35%
CPU peak:           85%

Memory:             Heavy usage

IO latency:         High

Database:           4 TB

Reporting queries:  Very heavy

Application:        24/7

Downtime allowed:   10 minutes

Availability:       Very high
```

### Tell me:

**1. Would you initially investigate General Purpose or Business Critical?**

**2. Why might Business Critical be attractive for this workload?**

**3. Would you immediately increase vCores because CPU peaks at 85%? Why/why not?**

**4. What would you investigate if IO latency is high?**

**5. Why might the reporting workload be moved to a readable secondary?**

