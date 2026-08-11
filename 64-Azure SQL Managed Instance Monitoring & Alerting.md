# Lesson 96 — Azure SQL Managed Instance Monitoring & Alerting

This lesson is about turning raw telemetry into **actionable DBA decisions**.

A production DBA shouldn't just know that CPU is 90%.

They should be able to answer:

> **Why is CPU 90%, which workload caused it, is the business affected, and what should I do next?**

---

# 1. Monitoring vs alerting

These are related but different.

### Monitoring

Continuously observe system behavior.

```text
SQL MI
 ↓
Metrics
 ↓
Logs
 ↓
Queries
 ↓
Dashboards
```

### Alerting

Notify someone when a condition requires attention.

```text
Metric/log
   ↓
Threshold/rule
   ↓
Alert
   ↓
DBA
```

---

# 2. The monitoring stack

Think in layers:

```text
                  MONITORING
                      |
       +--------------+--------------+
       |              |              |
    Platform       Database      Application
       |              |              |
   CPU/storage      waits       errors
   connectivity     queries     latency
   availability     blocking    transactions
```

You need all three.

---

# 3. Azure Monitor

Azure Monitor is a major component for monitoring Azure resources.

Conceptually:

```text
SQL MI
  ↓
Azure Monitor
  ↓
Metrics / Logs / Alerts
  ↓
Dashboards / Operations
```

It helps centralize operational visibility.

---

# 4. Metrics vs logs

### Metrics

Numerical time-series data.

Examples:

```text
CPU
Storage
IO
Connections
```

Great for:

```text
Trend analysis
Threshold alerts
Dashboards
```

### Logs

Detailed event/query information.

Examples:

```text
Errors
Auditing events
Diagnostic information
Query-related telemetry
```

Great for:

```text
Investigation
Correlation
Historical analysis
```

---

# 5. The most important DBA metrics

At minimum monitor:

```text id="m1c7v9"
CPU
Memory
Storage
I/O
Connections
Query duration
Throughput
Waits
Blocking
Deadlocks
Errors
Availability
Backup/DR health
```

---

# 6. CPU utilization

CPU tells you how much processing capacity is being consumed.

Example:

```text id="r4x8p2"
CPU = 95%
```

This is a warning sign.

But:

> High CPU is a symptom, not a root cause.

---

# 7. High CPU investigation

Don't immediately scale up.

Ask:

```text id="k6n2m8"
Which queries?
Which database?
Which application?
When did it start?
What changed?
```

Then investigate Query Store and execution plans.

---

# 8. CPU trend matters

Compare:

```text id="v3p7c5"
Yesterday
CPU = 45%

Today
CPU = 90%
```

versus:

```text id="q8m4x1"
Every weekday
CPU = 90%
```

The first suggests a change or incident.

The second might be a predictable workload pattern.

---

# 9. Storage monitoring

Monitor:

```text id="c5n8r3"
Data space
Log space
TempDB usage
Growth
Free capacity
```

Storage problems can become severe quickly.

---

# 10. Storage alert

Suppose:

```text id="a7k2m9"
Free storage = 5%
```

Don't wait until:

```text id="z4x8p6"
Free storage = 0%
```

Alert early enough to investigate.

---

# 11. Transaction log monitoring

A transaction log that keeps growing needs investigation.

Possible causes:

```text id="n3m7c1"
Long-running transaction
Missing log backup where applicable
Replication-related issue
Large transaction
Blocked process
Application behavior
```

Don't simply keep increasing log storage.

---

# 12. I/O monitoring

Look at:

```text id="p8r4v2"
Read latency
Write latency
IOPS
Throughput
```

High latency can make queries slow even when CPU is normal.

---

# 13. Example

```text id="x5k9m3"
CPU = 35%
IO latency = very high
Application = slow
```

This suggests:

```text id="h7q2c8"
CPU isn't the primary bottleneck.
```

Investigate storage/I/O behavior.

---

# 14. Memory

Memory pressure can lead to:

```text id="m6v3n8"
More physical reads
Reduced cache effectiveness
Query slowdowns
Increased I/O
```

Monitor memory behavior rather than reacting to one instantaneous value.

---

# 15. Wait statistics

Wait statistics answer:

> **What is the SQL engine waiting for?**

This is one of the most useful DBA diagnostic tools.

Conceptually:

```text id="r9p5k2"
Query
 ↓
Needs resource
 ↓
Waits
 ↓
Eventually continues
```

---

# 16. Common wait categories

You may encounter waits related to:

```text id="q3x7m1"
CPU
I/O
Locks
Memory
Parallelism
Network
Transaction log
```

The important skill is connecting the wait to the underlying bottleneck.

---

# 17. Blocking

Blocking occurs when one session holds a resource that another session needs.

Example:

```text id="c8n4v6"
Session A
   |
   | Holds lock
   ↓
Table

Session B
   |
   | Wants lock
   ↓
WAITING
```

---

# 18. Blocking chain

It can become:

```text id="k2m8p5"
Session A
   ↓
blocks B
   ↓
blocks C
   ↓
blocks D
```

One problematic transaction can affect many users.

---

# 19. Blocking investigation

Ask:

```text id="w7q3n9"
Who is blocking?
What are they running?
How long?
What lock?
Why is the transaction open?
Is it expected?
```

---

# 20. Deadlocks

A deadlock is different from ordinary blocking.

Example:

```text id="p4x8c2"
Session A holds Resource 1
Session B holds Resource 2

A needs Resource 2
B needs Resource 1
```

Result:

```text id="z6m3k7"
Circular dependency
```

SQL Server detects the deadlock and chooses a victim.

---

# 21. Deadlock monitoring

Track:

```text id="n8v2r5"
Frequency
Queries involved
Objects involved
Applications involved
Timing
```

One deadlock can be an isolated event.

Thousands indicate a design problem.

---

# 22. Query Store

Query Store is one of the DBA's best tools for historical query analysis.

It can help answer:

```text id="f5k9m1"
Which query is slow?
When did it become slow?
Did its plan change?
How much CPU does it consume?
What was its historical behavior?
```

---

# 23. Query Store workflow

```text id="c7x2p8"
Alert
 ↓
Identify affected period
 ↓
Query Store
 ↓
Find top queries
 ↓
Compare plans
 ↓
Identify regression
 ↓
Fix
```

---

# 24. Top CPU query

Suppose:

```text id="r4m8n3"
Query A = 70% of CPU
Query B = 10%
Query C = 5%
```

Investigate Query A first.

Don't spend an hour tuning Query C because its code is easier.

Prioritize by impact.

---

# 25. Query duration

Monitor:

```text id="v2k6q9"
Average duration
95th percentile
Maximum duration
Execution count
```

Average alone can hide problems.

---

# 26. Example

Suppose:

```text id="y8p3m5"
Average = 100 ms
```

But:

```text id="g6r2c7"
5% of executions = 10 seconds
```

Users may still experience serious latency.

Percentiles can provide a better picture.

---

# 27. Connections

Monitor connection behavior:

```text id="m4x7k1"
Connection count
Failed connections
Connection spikes
Authentication failures
```

A sudden connection spike could indicate:

```text id="p9n3c6"
Application deployment
Connection pool problem
Retry storm
Attack
Misconfiguration
```

---

# 28. Connection storm

Imagine:

```text id="j5q8r2"
Normal:
2,000 connections

Suddenly:
20,000 attempts
```

Potential causes:

```text id="w3m7x9"
Application retry loop
Pool misconfiguration
Service restart
Network failure
```

This can create a cascading failure.

---

# 29. Error monitoring

Monitor database errors.

Look for:

```text id="q6k2n8"
Authentication failures
Timeouts
Deadlocks
Resource exhaustion
Connection failures
Permission errors
```

Error spikes often provide an early warning.

---

# 30. Availability monitoring

A healthy monitoring system should answer:

```text id="x4v8m2"
Is SQL MI reachable?
Are applications connecting?
Are databases operational?
Are critical transactions succeeding?
```

Availability isn't just:

```text
Azure resource = green
```

---

# 31. Business monitoring

This is the most mature level.

Example:

```text id="r8c5p1"
Orders created per minute
Payments processed
Invoices generated
```

If:

```text id="m7n3q9"
CPU = normal
```

but:

```text id="k2x6v4"
Orders = 0
```

you have a serious production incident.

---

# 32. Technical health vs business health

```text id="y5q8c3"
Technical health
       |
       +-- CPU
       +-- memory
       +-- storage
       +-- waits

Business health
       |
       +-- orders
       +-- payments
       +-- transactions
```

A mature DBA monitors both.

---

# 33. Alerting principles

Bad alert:

```text id="j9m3x7"
CPU > 70%
```

for one minute.

This can create noise.

Better:

```text id="c4k8p2"
CPU > 85%
for 10+ minutes
AND
business/application impact detected
```

The exact thresholds should be based on your workload baseline.

---

# 34. Avoid alert fatigue

If the DBA receives:

```text id="v7n2m5"
500 alerts/day
```

they'll eventually ignore alerts.

Good monitoring has:

```text id="x3q8r1"
Fewer
Relevant
Actionable
Prioritized
```

alerts.

---

# 35. Severity levels

Example:

### Critical

```text id="a5m7k9"
Service unavailable
Data loss risk
Severe production outage
```

### High

```text id="p2x6c4"
Sustained resource saturation
Major application degradation
```

### Warning

```text id="r8v3n1"
Storage approaching threshold
Increasing latency
```

---

# 36. Alert routing

Different alerts may go to different teams.

```text id="k6q4m8"
Database alert
    ↓
DBA

Network alert
    ↓
Network team

Application error
    ↓
Application team

Security event
    ↓
Security team
```

Don't send everything to everyone.

---

# 37. Alert payload

A useful alert should tell you:

```text id="z3p7x5"
What happened?
Which resource?
When?
Severity?
Current value?
Threshold?
Link to investigation?
```

Bad:

> "CPU alert."

Good:

> "Production SQL MI CPU exceeded 90% for 15 minutes beginning 14:32 UTC."

---

# 38. Dashboards

A production dashboard should answer quickly:

```text id="n4c8m2"
Is everything healthy?
If not, where?
How severe?
What changed?
```

---

# 39. DBA dashboard

Recommended sections:

```text id="r7m3q9"
Availability
CPU
Memory
Storage
I/O
Connections
Blocking
Deadlocks
Top queries
Errors
Backups
DR
```

---

# 40. Dashboard design

Don't put 100 charts on one screen.

Start with:

```text id="x8p2k5"
Health
 ↓
Capacity
 ↓
Performance
 ↓
Errors
 ↓
Workload
```

Then drill down.

---

# 41. Log Analytics

Log Analytics can centralize and query logs from Azure resources.

Conceptually:

```text id="m6v9c3"
SQL MI
Application
Azure resources
     ↓
Log Analytics
     ↓
Queries
     ↓
Correlation
```

This is particularly useful for cross-service troubleshooting.

---

# 42. Correlation

Suppose at:

```text id="q3n7x8"
14:20
```

SQL latency increases.

Log Analytics can help correlate:

```text id="f5m2r9"
Application deployment
+
SQL performance change
+
Authentication errors
```

Now you have a likely timeline.

---

# 43. Timeline analysis

One of the most powerful troubleshooting techniques is:

```text id="p8c4v6"
What changed immediately before the incident?
```

Example:

```text id="j7n3k2"
14:00 deployment
14:05 connection spike
14:07 CPU spike
14:10 timeouts
```

The deployment becomes an important suspect.

---

# 44. Baseline monitoring

Normal workload:

```text id="w5m8x3"
CPU = 40–60%
```

Today:

```text id="r2q7c9"
CPU = 85%
```

This is more meaningful than an arbitrary global threshold.

Know your environment's normal.

---

# 45. Capacity planning

Monitoring isn't only for incidents.

Track trends:

```text id="n8p4m6"
Storage growth
CPU growth
Connection growth
Transaction growth
```

Then predict:

```text id="c5x2k9"
When will capacity become a problem?
```

---

# 46. Example capacity trend

Suppose storage usage:

```text id="z7m3q1"
January  = 40%
March    = 50%
May      = 60%
July     = 70%
```

The right question isn't:

> "Are we okay today?"

It's:

> "When will we hit our operational threshold?"

---

# 47. Performance incident framework

When users report:

> "The database is slow."

Use:

```text id="q6n8r4"
1. Scope
2. Timeline
3. Symptoms
4. Resource utilization
5. Waits
6. Blocking
7. Query Store
8. Recent changes
9. Root cause
10. Remediation
```

---

# 48. Step 1 — Scope

Ask:

```text id="x2m7p5"
One user?
One application?
One database?
All databases?
Entire SQL MI?
```

This immediately narrows the investigation.

---

# 49. Step 2 — Timeline

Determine:

```text id="v9k3c6"
When did it start?
Is it continuous?
Intermittent?
Exactly coincides with a deployment?
```

---

# 50. Step 3 — Resource check

Look at:

```text id="j4n8m2"
CPU
Memory
Storage
I/O
Connections
```

Then:

```text id="p7x5r1"
Is there a clear bottleneck?
```

---

# 51. Step 4 — Waits and blocking

If CPU isn't saturated:

```text id="m8c3q7"
Check waits
Check blocking
Check deadlocks
```

You may find:

```text id="z2k6n9"
Lock contention
```

instead of a CPU problem.

---

# 52. Step 5 — Query Store

Find:

```text id="f3v7m5"
Top resource consumers
Recently regressed queries
Plan changes
```

---

# 53. Step 6 — Recent changes

Ask:

```text id="c9n4x8"
Deployment?
Schema change?
Index change?
Statistics change?
Configuration?
Maintenance?
Traffic increase?
```

---

# 54. Step 7 — Root cause

Example:

```text id="r6m2p9"
New application deployment
       ↓
Query lost useful index usage
       ↓
CPU increased
       ↓
Request latency increased
```

Now you have a causal chain.

---

# 55. Step 8 — Remediation

Possible actions:

```text id="k5x8c3"
Query tuning
Index change
Plan correction
Application fix
Configuration adjustment
Scaling
```

Use the least disruptive appropriate fix.

---

# 56. Step 9 — Verify

After remediation:

```text id="m7q2v4"
CPU ↓
Latency ↓
Errors ↓
Query duration ↓
Business transactions ↑
```

If metrics improve, validate that the business problem is actually resolved.

---

# 57. Example incident

Alert:

```text id="j8c3n5"
CPU > 90% for 15 minutes
```

Investigation:

```text id="q4m7x2"
Query Store:
Query 123 = huge CPU increase
```

Execution plan:

```text id="p6v9k1"
Index seek → table scan
```

Recent change:

```text id="z3r8c5"
Index removed during deployment
```

Root cause:

```text id="w5n2m7"
Deployment removed required index.
```

Remediation:

```text id="c7x4p9"
Restore appropriate index
```

---

# 58. Don't blindly create indexes

Indexing has costs:

```text id="r8m3k6"
Storage
Write overhead
Maintenance
Insert/update/delete cost
Plan complexity
```

Tune based on workload evidence.

---

# 59. Blocking incident

Alert:

```text id="n2c7v5"
Blocking duration > threshold
```

Investigate:

```text id="x9m4k1"
Blocking session
Blocked sessions
Transaction duration
SQL statement
Application
Lock type
```

Then determine whether to:

```text id="p6r8q3"
Wait
Optimize transaction
Fix application
Terminate session
```

Killing a session should be a deliberate decision.

---

# 60. Deadlock incident

Suppose:

```text id="m3v7c9"
Deadlocks increased from
2/day → 500/day
```

Don't just monitor them.

Investigate:

```text id="f8k2n5"
Queries
Lock order
Transactions
Indexes
Application behavior
```

This is an application/database design problem.

---

# 61. Storage incident

Alert:

```text id="q5x9m3"
Storage = 90%
```

Investigate:

```text id="r7c2k8"
What is consuming space?
Data?
Log?
TempDB?
Recent growth?
```

Then determine the appropriate remediation.

---

# 62. Monitoring doesn't replace backups

This is critical.

Monitoring tells you:

```text id="x4m8p2"
Something happened.
```

Backups allow:

```text id="v7n3c6"
Recovery from certain data-loss scenarios.
```

You need both.

---

# 63. Monitoring doesn't replace DR

Similarly:

```text id="j8q5m2"
Monitoring
≠
Disaster recovery
```

Monitoring detects problems.

DR provides recovery capability.

---

# 64. Security monitoring

Include:

```text id="p3c7n9"
Failed logins
Unexpected access
Privilege changes
Suspicious activity
Audit events
```

Database monitoring and security monitoring overlap but aren't identical.

---

# 65. Monitoring ownership

Define:

```text id="k9x2m5"
Who receives alerts?
Who responds?
Who escalates?
Who owns the application?
Who owns networking?
Who owns security?
```

An alert without an owner is not operationally useful.

---

# 66. On-call model

Example:

```text id="r5m8q3"
Alert
 ↓
Primary DBA
 ↓
Application team if needed
 ↓
Platform team if needed
 ↓
Incident commander
```

The exact structure depends on your organization.

---

# 67. Monitoring checklist

```text id="c2v7n4"
PLATFORM
☐ CPU
☐ Memory
☐ Storage
☐ I/O
☐ Connectivity

DATABASE
☐ Query duration
☐ Waits
☐ Blocking
☐ Deadlocks
☐ Connections
☐ Errors

WORKLOAD
☐ Throughput
☐ Transactions
☐ Top queries
☐ Query regressions

OPERATIONS
☐ Backups
☐ DR
☐ Jobs
☐ Maintenance

SECURITY
☐ Failed authentication
☐ Privilege changes
☐ Suspicious activity

ALERTING
☐ Thresholds
☐ Severity
☐ Routing
☐ Escalation
```

---

# 68. Interview question

### "What would you monitor on SQL MI?"

Strong answer:

> "I'd monitor availability, CPU, memory, storage, I/O, connections, waits, blocking, deadlocks, query performance, errors, backups, DR health, and security events. I'd also monitor business-level transactions for critical applications."

---

# 69. Interview question

### "CPU is 95%. What do you do?"

Don't say:

> "Scale up."

Say:

```text id="q8m3v6"
1. Confirm duration and scope.
2. Identify top CPU consumers.
3. Check Query Store.
4. Check waits.
5. Check recent changes.
6. Determine root cause.
7. Apply targeted remediation.
8. Scale if capacity is genuinely the bottleneck.
```

---

# 70. Interview question

### "What is blocking?"

> "Blocking occurs when one session holds a lock or other resource that another session needs, causing the second session to wait."

---

# 71. Interview question

### "What is a deadlock?"

> "A deadlock occurs when two or more sessions form a circular dependency by holding resources that the others need. SQL Server detects the cycle and terminates one transaction as the deadlock victim."

---

# 72. Interview question

### "Why is Query Store useful?"

> "It provides historical query-performance information, allowing DBAs to identify expensive queries, plan changes, and performance regressions over time."

---

# 73. Interview question

### "Why shouldn't you alert on every metric immediately?"

Because:

```text id="n5c8r2"
Too many alerts
   ↓
Alert fatigue
   ↓
Ignored alerts
   ↓
Missed incidents
```

Alerts should be:

```text id="m7x3p9"
Actionable
Meaningful
Prioritized
```

---

# 74. The production DBA monitoring model

Memorize:

```text id="x6k2v8"
                MONITORING
                    |
        +-----------+-----------+
        |           |           |
      Health     Capacity    Workload
        |           |           |
      Up/down    CPU/storage  Queries
      Errors     I/O          Waits
      Access     Growth       Blocking
                             Deadlocks
                    |
                    ↓
                 Alerts
                    |
                    ↓
               Investigation
                    |
                    ↓
               Root Cause
                    |
                    ↓
               Remediation
                    |
                    ↓
                Validation
```

The key principle:

> **Monitoring tells you what changed. DBA investigation determines why.**

---

## Next — Lesson 97: Azure SQL Managed Instance Performance Tuning

We'll move from **monitoring** to **actually fixing performance problems**: **wait statistics, execution plans, Query Store regressions, indexing strategy, statistics, parameter sniffing, SARGability, implicit conversions, temp tables, joins, cardinality estimation, parallelism, memory grants, and a complete slow-query troubleshooting exercise.**
