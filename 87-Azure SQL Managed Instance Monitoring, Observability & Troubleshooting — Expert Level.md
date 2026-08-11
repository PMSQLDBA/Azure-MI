# Lesson 119 — Azure SQL Managed Instance Monitoring, Observability & Troubleshooting — Expert Level

The goal is to move from:

> **“I can see CPU and memory.”**

to:

> **“I can detect abnormal behavior, correlate database and application signals, isolate the bottleneck, determine root cause, and prove the remediation worked.”**

---

# 1. Monitoring vs observability

These terms are related but different.

### Monitoring

Answers:

> **Is something wrong?**

Example:

```text
CPU > 90%
```

### Observability

Answers:

> **Why is it wrong?**

Example:

```text
CPU > 90%
 ↓
Top query consumes 65% CPU
 ↓
Plan changed yesterday
 ↓
New plan performs large scan
```

---

# 2. The three pillars

A useful observability model is:

```text id="4p6m9a"
Metrics
Logs
Traces
```

For SQL MI, add database-specific telemetry:

```text id="8s3j1x"
Query Store
DMVs
Wait statistics
Execution plans
```

---

# 3. Four questions observability must answer

A mature platform should answer:

```text id="c2x7m5"
1. Is SQL MI healthy?
2. Is the application healthy?
3. What changed?
4. What is causing the problem?
```

---

# 4. The observability architecture

Conceptually:

```text id="r7n3q8"
                  SQL MI
                    |
       +------------+------------+
       |            |            |
     Metrics       Logs      Query Store
       |            |            |
       +------------+------------+
                    |
              Azure Monitor
                    |
              Log Analytics
                    |
             Alerts / Dashboards
                    |
              Incident Response
```

---

# 5. Azure Monitor

Microsoft Azure Monitor is the central Azure monitoring platform.

Think of it as the ecosystem around:

```text id="3x5q9p"
Metrics
Logs
Alerts
Dashboards
Workbooks
Diagnostic data
```

---

# 6. Metrics

Metrics are generally time-series measurements.

Examples conceptually:

```text id="5a8m2k"
CPU
Storage
IO
Connections
Workload activity
```

They are useful for:

```text id="9p3c7v"
Dashboards
Threshold alerts
Capacity planning
Trend analysis
```

---

# 7. Metrics vs logs

### Metric

```text id="2f7n8m"
CPU = 82%
```

Fast and compact.

### Log

```text id="1q6v4s"
Timestamp
Query
User
Database
Duration
Action
Result
```

Much richer context.

---

# 8. Diagnostic settings

Diagnostic settings can route supported resource telemetry to destinations such as:

```text id="7x2m5c"
Log Analytics workspace
Storage account
Event Hubs
```

The exact categories available depend on the resource and current Azure capabilities.

---

# 9. Why centralize logs?

Suppose you have:

```text id="q1w7e9"
SQL MI
Application
Network
Identity
```

Centralized telemetry lets you correlate events.

For example:

```text id="8h3p5v"
10:01 Application latency increases
10:01 SQL query duration increases
10:01 Blocking appears
10:01 Deployment occurred
```

Now you have a timeline.

---

# 10. Log Analytics

Microsoft Log Analytics provides centralized log querying using Kusto Query Language (KQL).

Conceptually:

```text id="6s9k1b"
Telemetry
   ↓
Log Analytics
   ↓
KQL
   ↓
Investigation
```

---

# 11. KQL mindset

KQL is designed around:

```text id="0m4n8x"
Get data
 ↓
Filter
 ↓
Project
 ↓
Aggregate
 ↓
Sort
```

A simple conceptual query:

```kusto
SomeTable
| where TimeGenerated > ago(1h)
| summarize count() by bin(TimeGenerated, 5m)
```

---

# 12. Why KQL matters

Instead of manually opening multiple dashboards:

```text id="w7p3c6"
Query logs
 ↓
Filter incident window
 ↓
Group by resource
 ↓
Find anomaly
```

This makes incident investigation repeatable.

---

# 13. Time correlation

Always establish:

```text id="2v6j8q"
Incident start
Incident end
```

Then investigate only the relevant window first.

For example:

```text id="4g8n1y"
09:55 → healthy
10:03 → latency spike
10:07 → deployment
10:09 → CPU spike
```

This is much more useful than looking at a random 24-hour chart.

---

# 14. Baselines

You need to know what "normal" means.

Example:

```text id="h5c2m9"
Normal CPU:
35–55%

Incident:
92%
```

But perhaps:

```text id="q4x8r1"
CPU = 92%
```

is normal during a scheduled nightly batch.

Context matters.

---

# 15. Static thresholds

Simple alert:

```text id="d8f2m5"
CPU > 90%
```

Useful, but limited.

Why?

Because:

```text id="u6r1p7"
90% for 30 seconds
```

is different from:

```text id="e3k9x2"
90% for 2 hours
```

---

# 16. Dynamic thresholds

Better alerting considers historical behavior.

For example:

```text id="a5n8q4"
Normal at 2 PM:
40%

Today:
85%
```

Even though 85% might not cross a static emergency threshold, it could be anomalous for that workload.

---

# 17. Alert fatigue

If every minor event creates a page:

```text id="f7m3k8"
100 alerts/day
 ↓
Nobody trusts alerts
```

This is operational failure.

Alerts should be:

```text id="x2v9q6"
Actionable
Prioritized
Context-rich
```

---

# 18. Alert severity

A useful conceptual model:

```text id="k3p6w1"
Informational
Warning
High
Critical
```

Only critical issues should wake someone immediately.

---

# 19. SLI

**Service Level Indicator**

A measurable signal.

Examples:

```text id="j7q4m8"
Database query latency
Connection success rate
Transaction success rate
```

---

# 20. SLO

**Service Level Objective**

The target.

Example:

```text id="m5x8r2"
99.9% successful requests
```

or:

```text id="b1n6v4"
p95 query latency < 300 ms
```

---

# 21. SLA

An SLA is a contractual/service commitment.

Don't confuse:

```text id="z3k7p9"
SLI → measurement
SLO → target
SLA → commitment
```

---

# 22. Error budgets

Suppose:

```text id="r8m2q5"
SLO = 99.9%
```

You have a finite amount of acceptable unreliability.

This creates a useful operational principle:

> Reliability work and feature delivery should be balanced against the remaining error budget.

---

# 23. SQL MI health dashboard

A production dashboard should expose important signals such as:

```text id="q6w4n1"
CPU
Storage
Connections
Query latency
Query throughput
Failed operations
Blocking
Deadlocks
Waits
Backup status
HA/DR health
```

Exact metrics depend on the SQL MI capabilities and telemetry sources available in your environment.

---

# 24. Query Store

Query Store is your historical database performance memory.

It helps answer:

```text id="v2p8m4"
Which queries are expensive?
Which queries changed?
Which plans were used?
When did performance regress?
```

---

# 25. DMVs

Dynamic Management Views provide runtime and internal SQL Server information.

Conceptually:

```text id="c4n7x9"
Current state
Sessions
Requests
Waits
Indexes
Query execution
```

DMVs are extremely useful during live troubleshooting.

---

# 26. Query Store vs DMVs

Think:

```text id="s8y2j5"
DMVs
→ current/runtime state

Query Store
→ historical workload behavior
```

Both are valuable.

---

# 27. Wait analysis

Suppose users report:

> "Everything is slow."

Check waits.

You might discover:

```text id="e5m8q1"
Lock waits dominate
```

Then investigate:

```text id="k7r3v6"
Blocking
 ↓
Long transaction
 ↓
Application code
```

That's much more actionable than:

```text id="t1n9p4"
CPU = 50%
```

---

# 28. Blocking investigation

Use a chain:

```text id="c8m2w7"
Victim session
    ↓
Waiting on lock
    ↓
Blocking session
    ↓
Transaction
    ↓
Application
```

Find the **root blocker**, not just the victim.

---

# 29. Deadlock monitoring

Deadlocks are concurrency failures.

Monitor:

```text id="x4p9k3"
Frequency
Involved queries
Tables/indexes
Transaction order
Application pattern
```

A deadlock occurring once isn't necessarily equivalent to hundreds per minute.

---

# 30. Root-cause analysis

A good RCA should distinguish:

### Symptom

```text id="j5v8n2"
API latency increased
```

### Immediate cause

```text id="m4q7c1"
Database requests blocked
```

### Root cause

```text id="r6x2p9"
New deployment introduced long transaction
```

### Contributing factors

```text id="w8n3k5"
Missing alert
No transaction-duration monitoring
Insufficient load testing
```

---

# 31. Five Whys

Example:

```text id="u2k7m9"
Why was API slow?
→ Database requests blocked.

Why blocked?
→ Long transaction held locks.

Why long transaction?
→ New application code.

Why wasn't it caught?
→ Production-like concurrency wasn't tested.

Why?
→ Performance test didn't model transaction behavior.
```

Now you've found a process problem, not merely a SQL problem.

---

# 32. Change correlation

When investigating incidents, always ask:

> **What changed?**

Potential changes:

```text id="x9m4q7"
Application deployment
Schema change
Index change
Statistics update
Configuration
Traffic increase
Network change
Identity change
```

Many incidents are change-related.

---

# 33. Deployment markers

A mature observability system correlates deployments with telemetry.

Conceptually:

```text id="w1c6n8"
Deployment
     |
     | timestamp
     ↓
Performance telemetry
```

Then you can quickly see:

```text id="h4p7y2"
Deploy
 ↓
Latency increases
```

---

# 34. Database/application correlation

Suppose:

```text id="q3n8v6"
API p95 = 2 seconds
```

Correlate with:

```text id="r5m1x9"
SQL p95 = 1.8 seconds
```

Strong evidence points toward SQL.

But if:

```text id="t6k4p2"
SQL p95 = 100 ms
API p95 = 2 seconds
```

the bottleneck is elsewhere.

---

# 35. Distributed tracing

For distributed applications:

```text id="f7x2m8"
Request
 ↓
API
 ↓
Service A
 ↓
Service B
 ↓
SQL MI
```

Tracing helps identify which segment consumes the latency budget.

---

# 36. Correlation IDs

Use a request identifier:

```text id="a1q6z8"
Request ID = ABC123
```

Then correlate:

```text id="p4r9m2"
Application logs
+
Database-related telemetry
+
Downstream services
```

This dramatically improves incident investigation.

---

# 37. Slow query detection

Create a process to identify:

```text id="n8c3v7"
High-duration queries
High-CPU queries
High-read queries
Frequently executed queries
Regressed queries
```

Don't focus only on the single slowest query.

---

# 38. Cost of a query

A query executed:

```text id="e6m2w9"
1 time/hour
```

at 5 seconds may be less important than:

```text id="q7v4k1"
100,000 times/hour
```

at 100 ms.

Performance prioritization should consider:

```text id="j2x8p5"
Cost per execution
×
Execution frequency
```

---

# 39. Throughput matters

Suppose:

```text id="s9m3q6"
Latency = 50 ms
```

but throughput is:

```text id="w5k7r2"
10 requests/sec
```

while the business requires:

```text id="c8n1v4"
500 requests/sec
```

Low latency alone doesn't mean the system meets requirements.

---

# 40. Capacity monitoring

Track trends:

```text id="g4p6x8"
CPU growth
Storage growth
Query volume
Connection growth
Transaction volume
```

Then forecast:

```text id="m7r2n5"
"When will we exceed capacity?"
```

---

# 41. Capacity planning

Don't wait for:

```text id="q1w8e4"
CPU = 99%
```

Plan based on:

```text id="h3m6k9"
Current utilization
Growth rate
Seasonality
Upcoming releases
Business forecasts
```

---

# 42. Seasonal workloads

Example:

```text id="8p2r5v"
Normal:
40% CPU

Black Friday:
85% CPU
```

85% isn't necessarily an incident.

You need workload context.

---

# 43. Scheduled workloads

Batch processing can create predictable spikes:

```text id="f1x7m3"
01:00 → ETL starts
01:15 → CPU spikes
02:00 → returns to normal
```

Instead of alerting:

```text id="n6q8w2"
CPU > 80%
```

you may design alerts around unexpected behavior during known batch windows.

---

# 44. Storage monitoring

Monitor:

```text id="r3m9v7"
Current usage
Growth rate
Free capacity
Data/log behavior
```

Storage exhaustion can become a severe operational incident.

---

# 45. Connection monitoring

Watch for:

```text id="u5x1k8"
Unexpected connection growth
Connection failures
Pool exhaustion
Sudden application connection storms
```

---

# 46. Connection storm

A common failure pattern:

```text id="j9q4m6"
Application outage
 ↓
Requests retry
 ↓
More connections
 ↓
Database pressure
 ↓
More failures
 ↓
More retries
```

This is a feedback loop.

---

# 47. Retry storms

Retries should use:

```text id="w2n7p5"
Exponential backoff
Jitter
Maximum attempts
Circuit breaking
```

Blind immediate retries can amplify a database incident.

---

# 48. Alert dependency awareness

Don't create:

```text id="m4x8r2"
1 incident
 ↓
500 identical alerts
```

Correlate related alerts.

Example:

```text id="c7v3n9"
Primary failure
 ↓
Connection failures
 ↓
API errors
 ↓
User errors
```

The primary event is more important than every downstream symptom.

---

# 49. Incident severity

A useful approach:

### P1

```text id="p6r1x8"
Major business outage
```

### P2

```text id="k4m7q2"
Significant degradation
```

### P3

```text id="v9n2c5"
Limited impact
```

### P4

```text id="y3x6m1"
Minor/non-urgent
```

Organizations should define their own exact criteria.

---

# 50. Incident timeline

During an incident, maintain:

```text id="a8q5r3"
10:02 Alert fired
10:04 Engineer engaged
10:07 Root blocker identified
10:12 Mitigation applied
10:15 Latency normal
10:30 RCA investigation
```

This becomes invaluable for postmortems.

---

# 51. Runbooks

A runbook should answer:

```text id="d7m2k9"
What does the alert mean?
What should I check?
What commands/tools should I use?
What constitutes mitigation?
When should I escalate?
```

---

# 52. Example: CPU alert runbook

```text id="g1n6v4"
1. Confirm duration of CPU spike
2. Check query workload
3. Identify top CPU consumers
4. Check plan changes
5. Check waits
6. Check traffic increase
7. Check recent deployments
8. Mitigate
9. Validate recovery
```

---

# 53. Example: blocking alert runbook

```text id="x5p8m3"
1. Identify blocked sessions
2. Identify root blocker
3. Inspect transaction age
4. Identify application/request
5. Determine whether blocker is expected
6. Apply approved mitigation
7. Investigate why transaction was long
```

---

# 54. Dashboards for different audiences

### DBA dashboard

```text id="v2k7r4"
Queries
Waits
Blocking
CPU
I/O
Plans
```

### Platform dashboard

```text id="q8m3x6"
Resource health
Availability
Capacity
Networking
Alerts
```

### Business dashboard

```text id="n1p5c9"
Transaction success
Customer-facing latency
Availability
Business throughput
```

Don't overload everyone with every metric.

---

# 55. Observability cost

Logging everything forever can become expensive.

Design:

```text id="u4x7m2"
High-value telemetry
+
Appropriate retention
+
Sampling where suitable
+
Tiered storage
```

Security and compliance requirements may dictate longer retention for certain events.

---

# 56. Log retention

Define:

```text id="e6q9v1"
Hot retention
Long-term retention
Archive
Deletion
```

Retention should match:

```text id="z3m8p5"
Operational needs
Compliance
Security investigations
Cost
```

---

# 57. Security of telemetry

Monitoring data can contain sensitive information.

Protect:

```text id="r7x2k6"
Access
Retention
Export
Query permissions
```

The monitoring system itself becomes part of your security boundary.

---

# 58. Observability anti-pattern

Bad:

```text id="h9n4q2"
Collect everything
 ↓
Never analyze
```

Observability isn't valuable merely because telemetry exists.

It must support decisions.

---

# 59. Another anti-pattern

```text id="w6p1m8"
CPU dashboard
Memory dashboard
Storage dashboard
```

but no:

```text id="c2r7x5"
Query visibility
Wait analysis
Application correlation
```

That's monitoring without enough observability.

---

# 60. Another anti-pattern

Alert:

```text id="p8m3v7"
CPU > 80%
```

but no context:

```text id="y4q6n2"
Which query?
Which database?
Since when?
How abnormal?
What changed?
```

Good alerts should contain enough context to begin triage.

---

# 61. Automated anomaly detection

Advanced systems can detect:

```text id="g5n8x1"
Unexpected query latency
Unusual CPU pattern
Connection anomalies
Traffic changes
Replication anomalies
```

Automation should prioritize investigation, not blindly execute destructive remediation.

---

# 62. Automated remediation

Safe automation examples:

```text id="m2q7r4"
Create incident
Notify team
Capture diagnostics
Collect plan information
Scale according to approved policy
```

Dangerous automation:

```text id="k8v3p1"
Kill random sessions
Drop indexes
Restart production
```

Automation should have guardrails.

---

# 63. Observability maturity

### Level 1

```text id="q6m1x9"
Basic metrics
```

### Level 2

```text id="r3p7v5"
Centralized logs
```

### Level 3

```text id="t8n2k4"
Query Store + DMVs + alerts
```

### Level 4

```text id="y1c6m8"
Application/database correlation
```

### Level 5

```text id="z5q9r2"
SLOs
+
Automated anomaly detection
+
Predictive capacity planning
+
Automated evidence collection
```

---

# 64. Expert incident example

Imagine:

```text id="a7m3x8"
Users report:
"Checkout is slow."
```

Don't immediately scale SQL MI.

Start:

```text id="n2p6q9"
Checkout p95
     ↓
Database p95
     ↓
Top queries
     ↓
Waits
     ↓
Blocking
     ↓
Recent deployments
```

Suppose you discover:

```text id="w4r8m1"
New query plan
 ↓
Large scan
 ↓
CPU increase
 ↓
Checkout latency
```

Now you have a defensible root cause.

---

# 65. Post-incident review

A good postmortem asks:

```text id="v9c4x2"
What happened?
Why?
Why wasn't it detected earlier?
How was it mitigated?
What made recovery difficult?
What permanent change prevents recurrence?
```

Avoid:

> **“Who caused it?”**

Focus on system improvement.

---

# 66. Root-cause vs contributing factors

Example:

```text id="h3m7q5"
Root cause:
Plan regression

Contributors:
Outdated statistics
No regression alert
Insufficient performance test
```

This leads to better corrective actions.

---

# 67. Corrective action hierarchy

Prefer:

```text id="k1p8x6"
Automated prevention
        ↓
Automated detection
        ↓
Runbook mitigation
        ↓
Human investigation
```

Don't solve every recurring problem with:

> "Tell the DBA to watch it."

---

# 68. Final troubleshooting framework

When something breaks:

```text id="f4n9m2"
1. Define impact
        ↓
2. Establish timeline
        ↓
3. Check recent changes
        ↓
4. Confirm application vs database
        ↓
5. Check resource pressure
        ↓
6. Check waits/blocking
        ↓
7. Identify expensive queries
        ↓
8. Compare execution plans
        ↓
9. Mitigate safely
        ↓
10. Validate recovery
        ↓
11. Perform RCA
        ↓
12. Prevent recurrence
```

---

# 69. Expert interview question

### "How do you troubleshoot a production SQL MI incident?"

A strong answer:

> "I first establish the impact and incident timeline, then determine whether the latency is actually in SQL MI. I correlate application telemetry with Azure Monitor, Query Store, and SQL runtime information. I examine CPU, I/O, waits, blocking, deadlocks, query duration, execution plans, and recent changes. After identifying the most likely root cause, I apply the smallest safe mitigation, validate recovery against the baseline, and then perform a formal root-cause analysis."

---

# 70. Interview question

### "Monitoring shows CPU at 90%. What next?"

> "I wouldn't treat the CPU percentage as the root cause. I'd determine how long the spike has lasted, identify the queries consuming CPU, inspect execution plans and waits, compare the workload against its baseline, and check for traffic increases or recent changes. Only after determining whether the workload is genuinely capacity-bound would I consider scaling."

---

# 71. Interview question

### "Why use both Query Store and Azure Monitor?"

> "Azure Monitor provides broader platform-level observability, while Query Store gives database-specific historical query and plan information. Together they allow me to correlate infrastructure behavior with query-level behavior."

---

# 72. Interview question

### "What makes a good alert?"

> "It should be actionable, sufficiently specific, tied to meaningful impact or abnormal behavior, contain enough context for initial triage, and avoid generating excessive noise."

---

# 73. Golden rule

> **Don't ask only “Is the database healthy?” Ask “Can I explain the user's experience?”**

That is the difference between basic monitoring and true observability.

---

# 74. Final mental model

```text id="s6p2x9"
                    USER
                     ↓
              APPLICATION
                     ↓
              DISTRIBUTED TRACE
                     ↓
                 SQL MI
                     |
       +-------------+-------------+
       |             |             |
    METRICS         LOGS       QUERY STORE
       |             |             |
       +-------------+-------------+
                     ↓
               AZURE MONITOR
                     ↓
               LOG ANALYTICS
                     ↓
                    KQL
                     ↓
            CORRELATION / RCA
                     ↓
          INCIDENT RESPONSE
                     ↓
              REMEDIATION
                     ↓
               VALIDATION
                     ↓
            PREVENT RECURRENCE
```

That is the foundation of **expert Azure SQL Managed Instance monitoring, observability, and production troubleshooting**.

---

## Next — Lesson 120: Azure SQL Managed Instance Automation & Infrastructure as Code — Expert Level

We'll cover **Terraform/Bicep/ARM concepts, declarative infrastructure, SQL MI provisioning, networking dependencies, private connectivity, identity, secrets, Azure Policy, CI/CD, drift detection, environment promotion, automated database configuration, safe deployments, rollback, and building a production-grade SQL MI platform through IaC.**
