# Lesson 68 — Monitoring & Alerting for Azure SQL Managed Instance

Now we're moving from **troubleshooting after something breaks** to something more senior:

> **Detecting the problem before users report it.**

A production DBA should not rely on:

> "Users will tell us when SQL is slow."

You want observability that answers:

```text
What happened?
When did it happen?
Which database?
Which query?
Which resource?
Which application?
What changed?
Is it getting worse?
```

---

# 1. Monitoring has multiple layers

For Azure SQL Managed Instance, think in layers:

```text
                    Monitoring
                        |
       +----------------+----------------+
       |                |                |
    Azure            SQL MI          Application
    Metrics          Telemetry         Metrics
       |                |                |
 CPU / storage      Queries / waits    Response time
 Connections        Blocking           Errors
 Health             Deadlocks          Throughput
       |                |                |
       +----------------+----------------+
                        |
                   Correlation
                        |
                    Root cause
```

A single metric rarely tells the whole story.

---

# 2. Metrics vs logs

This distinction is important.

### Metrics

Usually numeric/time-series measurements.

Examples:

```text id="v8q2m5"
CPU percentage
Storage consumption
Connections
IO-related metrics
```

They are excellent for:

```text id="q3m7x1"
Dashboards
Threshold alerts
Trend analysis
```

---

### Logs

Detailed event/diagnostic information.

Examples:

```text id="j5n9r2"
Errors
Auditing events
Diagnostic information
Query-related telemetry
Security events
```

Logs are useful when you need to answer:

> **"What exactly happened?"**

---

# 3. Azure Monitor

Microsoft Azure provides Azure Monitor capabilities for collecting and analyzing resource telemetry.

Think:

```text id="x4p7m1"
SQL MI
   |
   +---- Metrics
   |
   +---- Diagnostic logs
   |
   +---- Alerts
   |
   v
Azure Monitor
   |
   +---- Dashboard
   +---- Log Analytics
   +---- Alerting
```

---

# 4. Log Analytics

Log Analytics is useful for centralized querying and investigation of collected logs.

Conceptually:

```text id="m6w2k9"
SQL MI
   |
   v
Diagnostic settings
   |
   v
Log Analytics workspace
   |
   v
KQL queries
```

This becomes especially useful when you have:

```text id="n8c4r3"
Multiple SQL MIs
Multiple applications
Multiple environments
Multiple Azure resources
```

---

# 5. Why centralized logging matters

Imagine you have:

```text id="p7x1m5"
Production MI 1
Production MI 2
Production MI 3
Production MI 4
```

Without centralized analysis:

```text id="k2q8v4"
Check MI 1
Check MI 2
Check MI 3
Check MI 4
```

With centralized logs:

```text id="g9m3r6"
All telemetry
     ↓
Central workspace
     ↓
Search/filter/correlate
```

Much easier for enterprise operations.

---

# 6. Query Store

Remember our earlier lessons?

Query Store is your historical SQL performance memory.

It can help answer:

```text id="c4r7n2"
Which query became slower?
When?
What plan was used?
Did the plan change?
How often does it execute?
```

That's different from simply watching:

```text id="s5x1m8"
CPU = 90%
```

CPU tells you **something is happening**.

Query Store can help tell you **which SQL workload is responsible**.

---

# 7. Build alerts around symptoms AND causes

Bad monitoring:

```text id="f8q2m4"
CPU > 80%
```

That's useful, but incomplete.

Better:

```text id="w3n7p9"
CPU > threshold
+
Query duration increasing
+
Blocking increasing
+
Deadlocks detected
```

Now you have more context.

---

# 8. Example: CPU alert

Suppose:

```text id="a6v9x2"
CPU > 85%
for 10 minutes
```

This is much better than:

```text id="j1k5r8"
CPU > 85%
for 5 seconds
```

Why?

A short spike may be normal.

A sustained spike is more likely to indicate:

```text id="p4m8c6"
Workload increase
Bad query
Plan regression
Batch job
Resource pressure
```

---

# 9. Alert fatigue

This is a huge production problem.

Suppose your monitoring generates:

```text id="z8q3n7"
500 alerts/day
```

Operations team starts ignoring alerts.

That's dangerous.

Your objective should be:

> **Alert on actionable conditions, not every unusual metric.**

---

# 10. Good vs bad alert

### Bad

```text id="x7c2m9"
CPU = 70%
```

Maybe nothing is wrong.

### Better

```text id="q5n8r1"
CPU > 90%
for 15 minutes
AND
application latency elevated
```

Now there's stronger evidence of impact.

---

# 11. Blocking alert

Imagine:

```text id="m4w9p2"
Blocked sessions > 50
```

This is actionable.

But an even better investigation alert might identify:

```text id="s6k3v8"
Head blocker
Transaction age
Database
Application
Wait duration
```

Because:

> **"50 blocked sessions"** is a symptom.

> **"One application transaction has been open for 45 minutes and is blocking 80 sessions"** is an actionable diagnosis.

---

# 12. Deadlock monitoring

Deadlocks should be monitored separately from ordinary blocking.

Why?

Because a deadlock is a cyclic dependency.

You want to capture:

```text id="h2r7m5"
Time
Database
Application
Victim
Participating statements
Resources
Deadlock graph
```

The graph is especially valuable because it tells you **why the cycle occurred**.

---

# 13. Failed-login monitoring

Security teams often care about:

```text id="c8m4x1"
Repeated failed authentication
```

Example:

```text id="v7n2q6"
10:01 → failed login
10:02 → failed login
10:03 → failed login
...
```

Potential causes:

```text id="j9p3r8"
Wrong password
Expired credential
Bad application configuration
Password spray
Compromised account
```

You need correlation with:

```text id="s4w8m2"
Source
Identity
Application
Time
```

---

# 14. Storage monitoring

Storage monitoring isn't just:

> "Are we full?"

You want trend analysis.

Suppose:

```text id="n6x1q4"
January → 40%
February → 48%
March → 56%
April → 67%
May → 78%
```

At the current growth rate, you'll eventually hit a capacity problem.

This is:

> **Capacity planning**

not emergency troubleshooting.

---

# 15. Storage growth calculation

Imagine:

```text id="k3r8m5"
Current usage = 700 GB
Free = 300 GB
Growth = 50 GB/month
```

Approximate runway:

```text id="x9q2v6"
300 / 50 = 6 months
```

So you have roughly six months of capacity at the current growth rate.

But a senior DBA asks:

```text id="m5p7n1"
Is growth linear?
Are there seasonal peaks?
Are backups affected?
Is there unexpected data growth?
```

Never assume the trend will remain constant.

---

# 16. Connection monitoring

Suppose connection count is:

```text id="w2c9r4"
Normal = 200
```

Suddenly:

```text id="g6m1x8"
1,500
```

Possible causes:

```text id="q4n7p2"
Connection leak
Application deployment
Connection pooling problem
Retry storm
Traffic spike
Slow queries causing connections to remain busy
```

Again:

> A metric is a clue, not a root cause.

---

# 17. Retry storm scenario

Remember our networking lesson?

Suppose SQL experiences a brief transient issue.

Application clients do:

```text id="h8r3m6"
Failure
 ↓
Retry
 ↓
Failure
 ↓
Retry
 ↓
Retry
 ↓
Retry
```

Now 500 application instances suddenly create thousands of connection attempts.

Monitoring shows:

```text id="b5x9q1"
Connections ↑
CPU ↑
Errors ↑
Latency ↑
```

The original problem may have been tiny.

The retry behavior amplified it into an incident.

---

# 18. Monitoring the application matters

A database-only dashboard isn't enough.

Suppose:

```text id="p2m7x4"
SQL CPU = 30%
```

but:

```text id="n9c3w8"
Application latency = 20 sec
```

Maybe the application is:

```text id="q6v1r5"
Waiting on external API
Processing data locally
Experiencing thread-pool starvation
```

Without application telemetry, you may incorrectly blame SQL.

---

# 19. Correlation is the goal

A strong monitoring system lets you correlate:

```text id="s4k8m2"
02:00
Application latency ↑
       |
02:01
SQL query duration ↑
       |
02:01
CPU ↑
       |
02:02
Plan changed
       |
02:02
Query Store shows regression
```

Now you have a story.

That's much more useful than:

```text id="r7x2p9"
CPU = 95%
```

---

# 20. Production dashboard

A practical DBA dashboard might include:

### Availability

```text id="z3m8q1"
MI health
Database availability
Connection failures
```

### Capacity

```text id="h6p2w4"
CPU
Storage
Memory/resource indicators
Connections
```

### Performance

```text id="k9v4n1"
Query duration
Top CPU queries
Top logical-read queries
Waits
Query Store regressions
```

### Concurrency

```text id="m7x2c5"
Blocked sessions
Deadlocks
Long-running transactions
```

### Security

```text id="q1n8r3"
Failed logins
Unexpected principals
Auditing events
```

---

# 21. Alert severity

Not everything should be a critical alert.

A useful model:

### Critical

```text id="p5x9m2"
Service unavailable
Severe connection failure
Major storage issue
Regional failure
```

### High

```text id="c7r4n8"
Sustained CPU saturation
Large blocking chain
Repeated deadlocks
Severe query regression
```

### Medium

```text id="v2m6q1"
Storage growth
Increasing latency
Capacity trend
```

### Informational

```text id="g8n3p5"
Normal failover
Configuration change
Routine events
```

---

# 22. Alert → runbook

Every important alert should ideally have a runbook.

For example:

```text id="x5q1m8"
ALERT:
>50 blocked sessions
```

Runbook:

```text id="j7r4n2"
1. Find head blocker
2. Identify application
3. Check transaction age
4. Check SQL text
5. Determine business impact
6. Decide whether intervention is safe
7. Capture evidence
8. Fix root cause
9. Validate
```

Now the alert is operationally useful.

---

# 23. Another alert

```text id="n4v8c2"
ALERT:
Query duration regression
```

Runbook:

```text id="s6m3p9"
1. Identify Query ID
2. Open Query Store
3. Compare plans
4. Check execution statistics
5. Compare estimated/actual rows
6. Check recent deployment
7. Investigate statistics/indexes
8. Stabilize if necessary
9. Validate
```

This turns monitoring into an operational system.

---

# 24. KQL mindset

When logs are centralized in Log Analytics, you'll often use **Kusto Query Language (KQL)** to search and correlate telemetry.

Conceptually:

```text id="q8r2m5"
Logs
  |
  v
KQL
  |
  +-- Filter
  +-- Aggregate
  +-- Group
  +-- Time window
  +-- Correlate
```

For example, conceptually:

```text id="m3x7v9"
Find failed authentication events
→ group by identity
→ group by source
→ count over time
```

The exact tables and schemas depend on which diagnostic categories you configure and where they're sent.

---

# 25. Why diagnostic settings matter

Collecting logs isn't automatic magic.

You need to deliberately design:

```text id="p6n1k8"
What to collect?
        ↓
Where to send it?
        ↓
How long to retain it?
        ↓
Who can query it?
```

Possible destinations and integrations depend on the telemetry type and Azure service capabilities.

For enterprise environments, centralized Log Analytics is commonly useful for cross-resource investigation.

---

# 26. Cost matters

More logging isn't always better.

If you collect everything everywhere forever:

```text id="w9m4q2"
Huge log volume
      ↓
Higher cost
      ↓
More noise
```

Instead define:

```text id="c3x7n1"
Security requirements
Compliance retention
Troubleshooting requirements
Operational needs
```

Then choose appropriate collection and retention.

---

# 27. 2 AM incident — complete example

Imagine this alert:

```text id="f8q2m5"
CRITICAL

Application latency > 10 seconds
```

You open the dashboard.

### Application

```text id="j4n7x3"
Latency ↑
Errors ↑
```

### SQL MI

```text id="r6p1w8"
CPU = 88%
```

### Query Store

```text id="m2v9k4"
Query 742
300 ms → 18 sec
```

### Plan history

```text id="c7x3n5"
Plan 101 → Plan 204
```

### Blocking

```text id="q5m8r2"
None
```

### Deadlocks

```text id="v1k6p9"
None
```

Now you have:

```text id="g8r4m1"
Application latency
      ↓
SQL query regression
      ↓
Plan changed
      ↓
No blocking
      ↓
Root-cause investigation
```

That's a high-quality monitoring outcome.

---

# 28. Compare with poor monitoring

Poor system:

```text id="z9q2m6"
CPU > 80%
```

Alert:

> SQL performance problem.

DBA spends:

```text id="p3m7x1"
45 minutes
```

checking:

```text id="n8v4c2"
CPU
IO
connections
```

Eventually discovers:

```text id="r5k1w9"
One query plan changed 10 minutes earlier.
```

Good monitoring could have surfaced that relationship much faster.

---

# 29. Senior DBA monitoring philosophy

Don't build monitoring around:

> **"What numbers can we collect?"**

Build it around:

> **"What production questions do we need to answer?"**

For example:

### Question

> Why is the application slow?

Need:

```text id="q7m3n8"
Application latency
SQL duration
Waits
Blocking
Query Store
```

### Question

> Are we running out of capacity?

Need:

```text id="w4p9c2"
CPU trend
Storage trend
Connections
Workload growth
```

### Question

> Did someone access data unexpectedly?

Need:

```text id="k6x1m7"
Authentication
Auditing
Identity
Source
Timestamp
```

---

# 30. The monitoring architecture

Memorize:

```text id="s2n8q4"
                Azure SQL MI
                     |
          +----------+----------+
          |          |          |
       Metrics     Logs      Query Store
          |          |          |
          +----------+----------+
                     |
                     v
               Azure Monitor
                     |
              +------+------+
              |             |
        Log Analytics     Alerts
              |             |
              v             v
            KQL         Notifications
                            |
                    +-------+-------+
                    |       |       |
                  Email   Teams   Incident system
```

The exact integrations vary by organization, but the architecture principle is the same.

---

# 31. What I want you to remember

A senior DBA doesn't just say:

> "CPU is high."

They say:

> "CPU increased at 02:04, coinciding with a Query Store plan regression for Query 742. The query's duration increased from 300 ms to 18 seconds, blocking is absent, and the new plan performs a large scan. The incident is therefore likely query-plan related rather than general MI capacity exhaustion."

**That is observability-driven troubleshooting.**

---

# Next — Lesson 69

Next we'll move into **SQL Server Agent and automation on Azure SQL Managed Instance**.

We'll cover:

* SQL Agent architecture on MI
* jobs
* schedules
* job steps
* proxies
* operators/alerts
* PowerShell/CmdExec considerations
* database maintenance jobs
* backup/restore automation
* index/statistics maintenance
* failed-job troubleshooting
* job history
* and a real production scenario where a **nightly SQL Agent job silently fails for 14 days** and nobody notices.
