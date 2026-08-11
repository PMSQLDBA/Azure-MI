# Lesson 101 — Azure SQL Managed Instance Monitoring, Alerting & Observability

Now we move from **security** into **production observability**.

The key principle:

> **Monitoring tells you that something changed. Observability helps you understand why.**

A production DBA shouldn't wait for users to say:

> "The database is slow."

You should ideally detect the problem, identify the affected workload, understand the cause, and know whether it requires action.

---

# 1. Monitoring vs observability

### Monitoring

You watch known signals:

```text
CPU
Memory
Storage
Connections
Latency
Errors
```

### Observability

You combine multiple signals to answer:

```text
What happened?
Why did it happen?
What changed?
Who is affected?
Is it getting worse?
```

Think:

```text
Monitoring
    ↓
Detect

Observability
    ↓
Understand
```

---

# 2. The four major observability areas

For SQL MI, think about:

```text
1. Infrastructure/resource health
2. Database/query health
3. Application behavior
4. Security/audit activity
```

Together:

```text
Application
    ↓
SQL workload
    ↓
Database
    ↓
SQL MI resources
    ↓
Azure platform
```

---

# 3. Azure Monitor

Azure Monitor provides the broader Azure monitoring framework.

Conceptually:

```text
SQL MI
  ↓
Azure Monitor
  ↓
Metrics / Logs / Alerts
  ↓
Dashboards / Automation
```

It helps you monitor both the Azure resource and the surrounding environment.

---

# 4. Metrics vs logs

This distinction is important.

### Metrics

Usually numeric/time-series information:

```text
CPU utilization
Storage utilization
Connections
```

Useful for:

```text
Dashboards
Threshold alerts
Trend analysis
```

### Logs

Detailed event/query/activity information.

Useful for:

```text
Investigation
Correlation
Historical analysis
Security
Troubleshooting
```

---

# 5. Think in signals

A useful production model:

```text
        Observability
             |
     +-------+-------+
     |       |       |
  Metrics  Logs    Traces*
     |       |       |
     +-------+-------+
             |
          Context
```

*Application tracing is usually outside SQL MI itself, but correlating application telemetry with database telemetry is extremely valuable.

---

# 6. Resource metrics

Start with:

```text
CPU
Memory
Storage
I/O
Connections
```

But don't interpret any single metric in isolation.

Example:

```text
CPU = 95%
```

could mean:

```text
Heavy legitimate workload
Bad query
Plan regression
Batch job
Scaling need
```

---

# 7. CPU monitoring

High CPU can indicate:

```text
Expensive queries
High concurrency
Parallel workloads
Poor execution plans
Insufficient capacity
```

Ask:

> **Which workload is consuming the CPU?**

Don't stop at:

> "CPU is high."

---

# 8. CPU investigation

Use:

```text
CPU metric
    ↓
Time window
    ↓
Query Store
    ↓
Top CPU queries
    ↓
Execution plan
    ↓
Root cause
```

This connects infrastructure metrics to actual SQL workload.

---

# 9. Memory monitoring

Memory pressure can affect:

```text
Caching
Query execution
Memory grants
Sorts
Hash operations
Overall responsiveness
```

But high memory utilization alone isn't necessarily a problem.

The important question is:

> **Is there evidence of memory pressure or degraded workload behavior?**

---

# 10. Storage monitoring

Watch:

```text
Used capacity
Available capacity
Growth trends
I/O behavior
```

A database with:

```text
95% storage used
```

may not be immediately failing, but it's a warning.

You need enough time to react.

---

# 11. Capacity planning

Don't wait for:

```text
Storage = 100%
```

Instead monitor the trend:

```text
Month 1 → 60%
Month 2 → 67%
Month 3 → 74%
Month 4 → 81%
```

Now you can estimate when action will be required.

---

# 12. I/O monitoring

I/O-related performance problems may show up as:

```text
High latency
Slow queries
High read/write activity
Storage pressure
```

Correlate:

```text
I/O
+
Query duration
+
Wait statistics
+
Execution plan
```

---

# 13. Connections

Monitor connection behavior.

Potential problems include:

```text
Too many connections
Connection leaks
Connection storms
Poor pooling
Unexpected application behavior
```

Example:

```text
Normal:
500 connections

Suddenly:
5,000 connections
```

That is an investigation trigger.

---

# 14. Connection storm

Imagine an application loses connectivity.

All workers retry simultaneously:

```text
Failure
 ↓
Retry × 10,000
 ↓
Connection storm
 ↓
Database pressure
 ↓
More failures
 ↓
More retries
```

This is a feedback loop.

Application retry behavior belongs in database observability too.

---

# 15. Query Store

Query Store is one of your most valuable database-level observability tools.

Use it to understand:

```text
Query performance
Plan history
Regressions
CPU
Duration
Execution frequency
```

---

# 16. Why historical data matters

Suppose:

```text
Monday:
Query = 200 ms

Tuesday:
Query = 200 ms

Wednesday:
Query = 3 seconds
```

Without history, you know:

> "It's slow."

With Query Store, you can ask:

> **What changed?**

---

# 17. Plan regression

Example:

```text
Old plan
    ↓
Index Seek
    ↓
200 ms

New plan
    ↓
Large Scan
    ↓
3 seconds
```

This is a classic performance regression pattern.

---

# 18. Query Store investigation

When a query becomes slow:

```text
1. Identify query
2. Check duration history
3. Check CPU history
4. Compare plans
5. Check execution frequency
6. Check plan changes
7. Identify root cause
```

---

# 19. Wait statistics

Waits tell you what SQL Server is spending time waiting for.

Conceptually:

```text
Query
 ↓
Needs resource
 ↓
Resource unavailable
 ↓
WAIT
```

Waits can help distinguish:

```text
CPU pressure
I/O pressure
Blocking
Memory issues
Parallelism
```

---

# 20. Blocking

Blocking occurs when one session prevents another from proceeding.

Example:

```text
Session A
   |
   | holds lock
   ↓
Session B
   |
   | waits
   ↓
Session C
```

This can create a blocking chain.

---

# 21. Blocking chain

Example:

```text
Head blocker
    ↓
Session 2
    ↓
Session 3
    ↓
Session 4
```

If you only investigate Session 4, you may waste time.

Find the **head blocker**.

---

# 22. Deadlocks

Deadlock:

```text
Transaction A
   ↓
Needs resource B

Transaction B
   ↓
Needs resource A
```

Neither can continue.

SQL Server detects the deadlock and chooses a victim to terminate.

---

# 23. Deadlock monitoring

Don't just ask:

> "Did a deadlock happen?"

Ask:

```text
Which queries?
Which tables?
Which indexes?
Which application?
What transaction pattern?
Why?
```

Repeated deadlocks usually indicate a workload/design problem worth investigating.

---

# 24. Alerting

Monitoring without alerting can still leave you blind.

An alert should answer:

```text
What happened?
How serious is it?
Who needs to know?
What action is expected?
```

---

# 25. Bad alert

```text
CPU > 70%
```

with no context.

This may generate constant noise.

---

# 26. Better alert

Consider:

```text
CPU > 90%
AND
sustained for 15 minutes
```

Then route it to the appropriate operational team.

---

# 27. Alert severity

A useful model:

### Informational

```text
Storage > 70%
```

No immediate action.

### Warning

```text
Storage > 85%
```

Plan action.

### Critical

```text
Storage > 95%
```

Immediate response.

---

# 28. Alert fatigue

If you receive:

```text
500 alerts/day
```

people stop paying attention.

That's dangerous.

Good monitoring aims for:

> **High signal, low noise.**

---

# 29. Alert on symptoms vs causes

Example:

```text
CPU > 90%
```

is a symptom.

A more actionable alert might identify:

```text
Critical resource saturation
```

while dashboards and query telemetry help determine the cause.

Don't try to encode every possible root cause into alert thresholds.

---

# 30. SLOs

Service Level Objectives define expected service behavior.

Example:

```text
Availability = 99.9%
```

or:

```text
95% of database requests < 500 ms
```

The exact SLO should reflect business requirements.

---

# 31. SLI vs SLO

### SLI

The measurement.

Example:

```text
Request latency
```

### SLO

The target.

Example:

```text
95% of requests under 500 ms
```

Think:

```text
SLI = What we measure
SLO = What we target
```

---

# 32. Error budget

If SLO is:

```text
99.9% availability
```

the organization has some allowed unavailability.

That allowance is commonly called an:

> **Error budget**

It helps balance:

```text
Reliability
vs
Change velocity
```

---

# 33. Dashboards

A DBA dashboard should answer:

```text
Is the system healthy?
What changed?
What is currently under pressure?
Which workload is responsible?
```

---

# 34. Useful dashboard sections

### Resource

```text
CPU
Memory
Storage
I/O
Connections
```

### Database

```text
Queries
Duration
CPU
Waits
Blocking
Deadlocks
```

### Availability

```text
Failures
Connectivity
Failovers
```

### Security

```text
Failed authentication
Privilege changes
Suspicious activity
```

---

# 35. Don't create a dashboard with 100 graphs

More graphs ≠ better observability.

A useful dashboard has:

```text
High-value signals
Clear thresholds
Time correlation
Drill-down paths
```

---

# 36. Time correlation

Suppose:

```text
14:00 — Deployment
14:05 — CPU increases
14:07 — Query duration increases
14:08 — Blocking increases
```

That timeline is extremely valuable.

It suggests:

```text
Deployment
   ↓
Workload change
   ↓
Performance impact
```

---

# 37. Change correlation

Always ask:

> **What changed?**

Possible changes:

```text
Application deployment
Database deployment
Index change
Statistics update
Configuration change
Traffic increase
Batch job
Security change
Network change
```

Many incidents become easier once the timeline is established.

---

# 38. Observability workflow

Use:

```text
Alert
 ↓
Scope
 ↓
Timeline
 ↓
Metrics
 ↓
Logs
 ↓
Query Store
 ↓
Waits
 ↓
Blocking/deadlocks
 ↓
Execution plan
 ↓
Root cause
 ↓
Fix
 ↓
Validate
```

---

# 39. Example — sudden CPU spike

Alert:

```text
CPU > 90%
```

---

## Step 1 — Timeline

```text
12:00 normal
12:10 deployment
12:12 CPU spike
```

---

## Step 2 — Query Store

Find:

```text
Query X
CPU increased from 5% → 60%
```

---

## Step 3 — Plan

Old:

```text
Index Seek
```

New:

```text
Large Scan
```

---

## Step 4 — Investigation

Recent schema/index change.

Suppose an index was removed during deployment.

---

## Step 5 — Fix

Restore the appropriate indexing strategy.

---

## Step 6 — Validate

```text
CPU: 92% → 48%
Query: 4 sec → 300 ms
```

Incident resolved.

---

# 40. Example — high latency but low CPU

You receive:

```text
Application latency = 10 seconds
CPU = 30%
```

Don't scale CPU immediately.

Investigate:

```text
Blocking
I/O latency
Network
Waits
Connection pool
```

Suppose:

```text
Head blocker
   ↓
10-second waits
```

Now the root cause is blocking.

---

# 41. Example — high connections

Metrics:

```text
Connections:
500 → 5,000
```

CPU:

```text
40%
```

Queries:

```text
Normal
```

Investigate application behavior.

Potential root cause:

```text
Connection pool misconfiguration
```

Not a database CPU problem.

---

# 42. Example — storage growth

You observe:

```text
60%
70%
80%
90%
```

over several months.

No immediate outage.

This is a:

> **Capacity-planning issue**, not necessarily an incident.

Create a proactive action before capacity becomes critical.

---

# 43. Proactive vs reactive monitoring

### Reactive

```text
Problem occurs
 ↓
Alert
 ↓
Investigate
```

### Proactive

```text
Trend detected
 ↓
Forecast
 ↓
Action before failure
```

Good DBAs do both.

---

# 44. Alert routing

Different alerts should go to appropriate owners.

Example:

```text
Database performance → DBA
Application failures → App team
Network failure → Network team
Security event → Security team
Capacity → Platform team
```

Avoid sending every alert to everyone.

---

# 45. Incident response

A good incident process:

```text
Detect
 ↓
Triage
 ↓
Assign owner
 ↓
Mitigate
 ↓
Investigate
 ↓
Resolve
 ↓
Validate
 ↓
Document
```

---

# 46. Mitigation vs root-cause fix

Suppose CPU is 99%.

Immediate mitigation:

```text
Scale resources
```

Long-term fix:

```text
Tune bad query
```

Both may be appropriate.

Don't confuse:

```text
Mitigation
```

with:

```text
Root-cause resolution
```

---

# 47. Observability retention

You need enough historical data to answer:

```text
What happened yesterday?
Last week?
Last month?
```

Retention should match:

```text
Incident investigation
Capacity planning
Compliance
Operational needs
```

---

# 48. Log Analytics

Azure Monitor can work with Log Analytics for centralized log analysis.

Conceptually:

```text
SQL MI
   ↓
Diagnostic/logging
   ↓
Log Analytics
   ↓
Queries
   ↓
Dashboards / Alerts
```

Centralization is particularly useful when correlating database activity with other Azure resources.

---

# 49. Cross-resource correlation

Imagine:

```text
Application Gateway
      ↓
App Service
      ↓
SQL MI
```

Application latency increases.

You can correlate:

```text
Gateway metrics
+
Application telemetry
+
SQL metrics
+
Query Store
```

This is far more powerful than looking at SQL MI alone.

---

# 50. End-to-end latency

Suppose user sees:

```text
Request = 5 seconds
```

Break it down:

```text
Network = 100 ms
Application = 500 ms
Database = 4.2 sec
```

Now SQL is the primary suspect.

But if:

```text
Database = 100 ms
Application = 4.5 sec
```

SQL isn't the bottleneck.

Observability prevents incorrect blame.

---

# 51. Production monitoring architecture

Conceptually:

```text
                    USERS
                      |
                 APPLICATION
                      |
          +-----------+-----------+
          |                       |
      App telemetry          SQL workload
                                  |
                             SQL MI
                                  |
                +-----------------+----------------+
                |                 |                |
             Metrics            Logs           Query Store
                |                 |                |
                +-----------------+----------------+
                                  |
                            Azure Monitor
                                  |
                            Log Analytics
                                  |
                     Dashboards / Alerts
                                  |
                           Incident Response
```

---

# 52. Monitoring checklist

```text
RESOURCE
☐ CPU
☐ Memory
☐ Storage
☐ I/O
☐ Connections

DATABASE
☐ Query Store
☐ Query duration
☐ CPU-heavy queries
☐ Waits
☐ Blocking
☐ Deadlocks

PLATFORM
☐ Availability
☐ Failovers
☐ Errors
☐ Network

SECURITY
☐ Authentication failures
☐ Privilege changes
☐ Suspicious activity

OPERATIONS
☐ Alerts
☐ Dashboards
☐ Runbooks
☐ Capacity trends
☐ Incident process
```

---

# 53. Interview question

### "What would you monitor on SQL MI?"

Strong answer:

> "I'd monitor resource utilization such as CPU, memory, storage, I/O, and connections; database workload through Query Store, waits, blocking, and deadlocks; availability and connectivity; and security-related activity. I'd correlate these signals with application telemetry and deployment timelines."

---

# 54. Interview question

### "CPU is 95%. What do you do?"

Don't say:

> "Scale the database."

Better:

```text
1. Determine duration and scope.
2. Identify top CPU-consuming workloads.
3. Check Query Store.
4. Review execution plans.
5. Check waits and concurrency.
6. Look for recent changes.
7. Determine whether the issue is query-related or capacity-related.
8. Mitigate if necessary.
9. Apply the root-cause fix.
10. Validate CPU and application performance afterward.
```

---

# 55. Interview question

### "What causes alert fatigue?"

> "Too many low-value alerts, poorly chosen thresholds, duplicate notifications, and alerts that don't require action. Good alerting focuses on actionable conditions and uses appropriate severity and duration."

---

# 56. Interview question

### "Why correlate application and database telemetry?"

Because:

```text
User sees:
5-second request
```

doesn't tell you whether the problem is:

```text
Network
Application
Database
External service
```

Cross-system correlation identifies the actual bottleneck.

---

# 57. Final mental model

Memorize:

```text
                  OBSERVABILITY
                       |
        +--------------+--------------+
        |              |              |
      Metrics         Logs        Query Store
        |              |              |
        +--------------+--------------+
                       |
                     Context
                       |
        +--------------+--------------+
        |              |              |
       CPU            I/O          Blocking
        |              |              |
        +--------------+--------------+
                       |
                   Root Cause
                       |
                    Action
                       |
                  Validation
```

---

# 58. The golden rule

> **Don't alert because a number is interesting. Alert because a condition requires action.**

And:

> **Don't troubleshoot one metric in isolation. Correlate metrics, workload, logs, changes, and application behavior.**

That is how you move from **basic monitoring** to **production observability**.

---

## Next — Lesson 102: Azure SQL Managed Instance Maintenance, Patching & Upgrades

We'll cover **Azure-managed maintenance, planned maintenance events, versioning, compatibility, patch strategy, maintenance windows, application impact, failover behavior, pre-production testing, rollback planning, upgrade validation, and a complete production maintenance runbook.**
