# Lesson 108 — Azure SQL Managed Instance Monitoring & Observability — Advanced

This lesson moves from **troubleshooting after something breaks** to **observability that tells you something is going wrong before users notice**.

The key idea:

> **Monitoring tells you what is happening. Observability helps you understand why it is happening.**

---

# 1. Monitoring vs observability

### Monitoring

You track known signals:

```text id="m8q2v6"
CPU
Memory
Storage
Connections
Latency
Errors
```

### Observability

You combine multiple signals to understand system behavior:

```text id="x4n7k1"
Metrics
+
Logs
+
Traces
+
Query telemetry
+
Events
+
Business signals
```

---

# 2. The three pillars

Traditional observability uses:

```text id="c6p9r3"
Metrics
Logs
Traces
```

For SQL MI, add:

```text id="v2m8x5"
Query Store
Database telemetry
Azure platform signals
```

---

# 3. Metrics

Metrics are numerical measurements over time.

Examples:

```text id="q7k3n9"
CPU utilization
Storage consumption
I/O
Connections
Query duration
Transaction rate
```

Metrics are excellent for:

```text id="p5x8m2"
Dashboards
Alerts
Capacity planning
Trend analysis
```

---

# 4. Logs

Logs provide event-level information.

Examples:

```text id="n4c7v1"
Errors
Authentication events
Administrative activity
Configuration events
Security events
```

Logs help answer:

> **What happened?**

---

# 5. Traces

Distributed applications may look like:

```text id="r8m3q6"
User
 ↓
API
 ↓
Service
 ↓
SQL MI
 ↓
External service
```

A database query may be slow because the application spent time elsewhere.

Distributed tracing helps identify where the latency actually occurred.

---

# 6. SQL Query Store

Query Store is one of the most valuable SQL performance tools.

It provides historical information about:

```text id="x2p7m5"
Queries
Plans
Execution statistics
Runtime trends
Regressions
```

This is particularly valuable because a DMV snapshot only shows what's happening now.

---

# 7. Current vs historical data

Think:

```text id="j6n9c3"
DMVs
→ What is happening now?

Query Store
→ What happened over time?

Azure Monitor
→ What is happening at platform/resource level?
```

Use them together.

---

# 8. Azure Monitor

Azure Monitor provides the broader monitoring platform around Azure resources.

Conceptually:

```text id="q3v8m1"
SQL MI
 ↓
Telemetry
 ↓
Azure Monitor
 ↓
Metrics / Logs / Alerts
```

---

# 9. Log Analytics

Log Analytics provides a centralized environment for querying supported log data.

Conceptually:

```text id="m5x2k8"
Multiple resources
      ↓
Central logs
      ↓
Log Analytics
      ↓
Queries
      ↓
Dashboards / Alerts
```

This becomes particularly valuable in larger environments.

---

# 10. Why centralization matters

Suppose you have:

```text id="r7c4n2"
20 SQL MIs
30 applications
Multiple regions
```

Checking each resource manually is inefficient.

Centralized telemetry allows:

```text id="v9p3m6"
Environment-wide investigation
```

---

# 11. Metrics dashboard

A production SQL MI dashboard might show:

```text id="k2x8q5"
CPU
Storage
I/O
Connections
Query latency
Errors
Deadlocks
Blocking
```

But don't put 100 charts on one screen.

The dashboard should answer:

> **Is the platform healthy?**

---

# 12. Executive dashboard vs DBA dashboard

### Executive

```text id="n6m3r8"
Availability
SLO
Major incidents
Business impact
```

### DBA

```text id="p4x7k1"
CPU
I/O
Waits
Blocking
Query regressions
Storage
Connections
```

Different audiences need different views.

---

# 13. SLI

**Service Level Indicator** is the measurement.

Examples:

```text id="c8v2m5"
Availability
Latency
Error rate
Successful transactions
```

---

# 14. SLO

**Service Level Objective** defines the target.

Example:

```text id="q9m4x7"
99.95% monthly availability
```

Another:

```text id="j3p8n2"
99% of critical queries < 500 ms
```

---

# 15. SLA

SLA is a formal service commitment.

Don't confuse:

```text id="x5k1v9"
SLI → measurement
SLO → target
SLA → formal commitment
```

---

# 16. Error budget

Suppose your SLO is:

```text id="m7r2c6"
99.9% availability
```

You have a small amount of allowable unavailability.

That becomes the:

```text id="p8n3x5"
Error budget
```

If you repeatedly consume it, reliability work should take priority.

---

# 17. Why DBAs should understand SLOs

Because:

```text id="v4m8q2"
CPU = 90%
```

is not automatically a business incident.

But:

```text id="n6c3x7"
Checkout success rate drops
```

is directly meaningful.

Monitoring should connect infrastructure signals to business outcomes.

---

# 18. Alert design

Bad alert:

```text id="q2x9m4"
CPU > 70%
```

for every five-minute interval.

This can create noise.

Better:

```text id="k7m3p8"
CPU > 90%
AND
sustained for 15 minutes
AND
application latency elevated
```

Context matters.

---

# 19. Alert fatigue

If operators receive:

```text id="r5n8v2"
500 alerts/day
```

they eventually stop trusting alerts.

Good alerting is:

```text id="x3c7m1"
Few
Relevant
Actionable
Prioritized
```

---

# 20. Actionable alert

Bad:

> CPU is high.

Better:

> SQL MI CPU has remained above 90% for 15 minutes and critical application latency has exceeded its SLO. Top CPU workload should be investigated.

The second tells the operator **why it matters**.

---

# 21. Alert severity

Example:

```text id="p9k4x6"
Critical
High
Medium
Low
Informational
```

Severity should map to operational action.

---

# 22. Alert ownership

Every important alert should have:

```text id="m2v8c5"
Owner
Runbook
Escalation path
Expected response
```

An alert without an owner is just noise.

---

# 23. Alert → runbook

Example:

```text id="n7q3x9"
High blocking
      ↓
Runbook
      ↓
Identify head blocker
      ↓
Check transaction
      ↓
Determine safe action
```

This reduces mean time to recovery.

---

# 24. MTTD

**Mean Time to Detect**

How long it takes to detect an incident.

Goal:

```text id="c5x2m8"
Detect important failures quickly.
```

---

# 25. MTTR

**Mean Time to Recovery/Repair**

How long it takes to restore service.

Observability should reduce:

```text id="v8p4n1"
MTTD
+
MTTR
```

---

# 26. Example

Without good monitoring:

```text id="q6m9x3"
Issue starts → 45 min later discovered
```

With proactive monitoring:

```text id="r2k7p5"
Issue starts → 3 min later detected
```

The monitoring system has directly improved reliability.

---

# 27. Proactive monitoring

Don't wait for:

```text id="x9c4m6"
Storage = 100%
```

Alert earlier:

```text id="n3v8k2"
Storage growth predicts capacity exhaustion
```

---

# 28. Capacity monitoring

Track:

```text id="m5p2q7"
Current usage
Growth rate
Historical trend
Forecast
```

Then ask:

> When will the current capacity become insufficient?

---

# 29. Example forecast

Current free storage:

```text id="k8x3r4"
120 GB
```

Average daily growth:

```text id="v2m7n5"
12 GB/day
```

Approximate:

```text id="q6p9c1"
120 / 12 = 10 days
```

Now you can act before the incident.

---

# 30. Baselines

A number isn't necessarily bad.

Example:

```text id="j4m8x2"
CPU = 80%
```

Could be normal for a batch workload.

Instead compare against:

```text id="p7n3c9"
Normal behavior
Same time of day
Same workload
Historical baseline
```

---

# 31. Anomaly detection

Instead of:

```text id="x5q2m8"
CPU > 80%
```

consider:

```text id="m9v4k1"
CPU significantly above normal baseline
```

This can detect unusual behavior even when an absolute threshold isn't crossed.

---

# 32. Business-hour baselines

Workloads often vary:

```text id="r3c8p6"
08:00 → low
12:00 → high
18:00 → medium
02:00 → batch processing
```

A single static threshold may generate unnecessary alerts.

---

# 33. Correlated monitoring

Consider:

```text id="n6x1m7"
CPU ↑
Query latency ↑
Error rate ↑
```

This is much stronger evidence than CPU alone.

---

# 34. Golden signals

For services, monitor:

```text id="v8m2q5"
Latency
Traffic
Errors
Saturation
```

For SQL MI, translate these into database-relevant signals:

```text id="p4x7c9"
Query latency
Transaction/query volume
SQL errors
CPU/I/O/storage/concurrency saturation
```

---

# 35. Saturation

Saturation means a resource is approaching its useful limit.

Examples:

```text id="k2m8v3"
CPU saturation
Storage saturation
I/O saturation
Connection saturation
```

Monitoring saturation helps predict failure.

---

# 36. Connection monitoring

Track:

```text id="q7x4n1"
Connection count
Connection failures
Connection duration
Connection pool behavior
```

A sudden connection spike can indicate:

```text id="m3p9c6"
Application issue
Connection leak
Traffic spike
Retry storm
```

---

# 37. Retry storms

A dangerous pattern:

```text id="r8v2k5"
SQL request fails
 ↓
Application retries
 ↓
Retry fails
 ↓
Application retries again
 ↓
Thousands of requests
```

This can make an existing outage worse.

Observability should correlate:

```text id="x6m3q9"
Application retries
+
Database errors
+
Connection spikes
```

---

# 38. Query latency monitoring

Track not only averages.

Consider:

```text id="p5n8c2"
Average
P50
P95
P99
```

---

# 39. Why averages can mislead

Suppose:

```text id="j7x3m9"
99 queries = 10 ms
1 query = 10 seconds
```

Average may hide the user experience.

Percentiles expose tail latency.

---

# 40. P95 and P99

### P95

95% of observations are at or below this value.

### P99

99% are at or below this value.

For customer-facing systems, tail latency can be extremely important.

---

# 41. Slow query alerts

Don't necessarily alert on:

```text id="v4k9m2"
Any query > 1 second
```

You may generate thousands of alerts.

Better:

```text id="n8p3x6"
Critical query
+
P95 latency exceeds SLO
+
Sustained over defined period
```

---

# 42. Deadlock monitoring

Deadlocks are not always catastrophic.

But a sudden increase is important.

Monitor:

```text id="c5m7x1"
Deadlocks/hour
Trend
Affected workload
```

---

# 43. Blocking monitoring

Track:

```text id="x2q8v4"
Blocked sessions
Block duration
Head blocker
Affected application
```

An occasional short block may be normal.

A 20-minute blocking chain is different.

---

# 44. Storage alerting

Use multiple levels:

```text id="m9k3p7"
Warning → capacity approaching threshold
Critical → capacity dangerously low
```

Also alert on **growth rate**, not only current percentage.

---

# 45. Security monitoring

Monitor:

```text id="r6v2n8"
Authentication failures
Unexpected identities
Privilege changes
Sensitive-data access
Security alerts
```

Security telemetry should be integrated with the operational monitoring strategy.

---

# 46. Monitoring architecture

A mature environment might look like:

```text id="p8m4x2"
                   SQL MI
                     |
        +------------+------------+
        |            |            |
      Metrics       Logs       Query Store
        |            |            |
        +------------+------------+
                     |
               Azure Monitor
                     |
              Log Analytics
                     |
          +----------+----------+
          |                     |
      Dashboards              Alerts
          |                     |
      DBA / SRE             Incident Mgmt
```

---

# 47. Centralized monitoring

For multiple environments:

```text id="v3n7q5"
Production
Staging
Development
DR
```

centralize important telemetry while maintaining appropriate access controls.

---

# 48. Environment separation

Don't let development noise overwhelm production monitoring.

Use:

```text id="k5x9m3"
Environment
Application
Service
Database
Region
```

as useful dimensions/tags.

---

# 49. Naming standards

Good names:

```text id="q2m8v6"
prod-sqlmi-orders-eastus
prod-sqlmi-payments-westus
```

Consistent naming improves:

```text id="r7p3c1"
Search
Alerting
Automation
Incident response
```

---

# 50. Dashboard design

A DBA dashboard might have:

```text id="m8x4n2"
TOP
→ Availability / critical alerts

SECOND
→ CPU / memory / I/O / storage

THIRD
→ Query latency / waits / blocking

FOURTH
→ Errors / deadlocks / connections

BOTTOM
→ Capacity trends
```

Prioritize information.

---

# 51. Monitoring the monitor

Your monitoring system can fail too.

Verify:

```text id="c9v2m5"
Telemetry is arriving
Alerts are firing
Alert routing works
Dashboards update
Logs are retained
```

A silent monitoring system is dangerous.

---

# 52. Alert testing

Don't assume an alert works because it exists.

Test:

```text id="x6p3k8"
Trigger condition
 ↓
Alert generated
 ↓
Notification sent
 ↓
On-call receives
 ↓
Runbook accessible
```

---

# 53. Observability during DR

Your DR environment needs monitoring too.

Don't discover during a disaster:

```text id="n4m8q2"
"Nobody can see the DR database."
```

Monitor:

```text id="v7x2c5"
Primary
+
DR
```

continuously at the required level.

---

# 54. Observability during failover

After failover verify:

```text id="p3k9m6"
New primary metrics
Application latency
Errors
Connections
Query performance
Security telemetry
Alerts
```

---

# 55. Example incident

Alert:

```text id="j5m2x8"
P95 query latency > SLO
```

Investigation:

```text id="r8c4n3"
Query Store
 ↓
Plan changed
 ↓
Logical reads ↑
 ↓
CPU ↑
```

Then:

```text id="q6v1p7"
Deployment occurred 10 minutes earlier
```

Now the telemetry has connected:

```text id="m3x8k2"
User symptom
+
Database behavior
+
Plan regression
+
Deployment
```

That is observability.

---

# 56. Observability maturity

### Level 1

```text id="v9p2m4"
"We have some metrics."
```

### Level 2

```text id="k4x7n1"
"We have dashboards."
```

### Level 3

```text id="c8m3q6"
"We have alerts."
```

### Level 4

```text id="r5v9p2"
"We correlate metrics, logs, and query telemetry."
```

### Level 5

```text id="m7x2k8"
"We proactively detect anomalies, measure SLOs, forecast capacity, and continuously improve reliability."
```

---

# 57. Monitoring checklist

```text id="q3n8v5"
PLATFORM
☐ CPU
☐ Memory
☐ I/O
☐ Storage
☐ Connections

DATABASE
☐ Query latency
☐ Waits
☐ Blocking
☐ Deadlocks
☐ Query Store

APPLICATION
☐ Request latency
☐ Error rate
☐ Retries
☐ Throughput

SECURITY
☐ Authentication
☐ Authorization
☐ Audit
☐ Security alerts

OPERATIONS
☐ Dashboards
☐ Alerts
☐ Runbooks
☐ Escalation
☐ Alert testing

RELIABILITY
☐ SLI
☐ SLO
☐ Error budget
☐ Capacity forecasting
☐ DR monitoring
```

---

# 58. Interview question

### "What's the difference between monitoring and observability?"

> "Monitoring tells us whether known conditions are healthy by tracking predefined signals. Observability combines metrics, logs, traces, and system telemetry to help us understand the internal causes of unexpected behavior."

---

# 59. Interview question

### "How would you design SQL MI alerts?"

> "I would start from business and reliability objectives, then alert on actionable symptoms such as sustained resource saturation, SLO violations, connection failures, blocking, deadlocks, storage risk, and security events. I'd avoid excessive threshold-based alerts, correlate signals where possible, assign ownership, and attach runbooks."

---

# 60. Interview question

### "Why is Query Store important?"

> "It provides historical query and plan information, allowing us to identify performance trends and regressions that may not be visible from a current DMV snapshot."

---

# 61. Interview question

### "CPU is normal, but users report high latency. What do you do?"

> "I wouldn't conclude the database is healthy based on CPU alone. I'd check query latency, waits, blocking, I/O, memory pressure, connection behavior, and application-side latency to determine whether the system is doing work or waiting."

---

# 62. Final observability model

```text id="x8m4q2"
                 USERS
                   |
             APPLICATION
                   |
             SQL MI
                   |
      +------------+------------+
      |            |            |
    Metrics       Logs       Queries
      |            |            |
      +------------+------------+
                   |
            OBSERVABILITY
                   |
       +-----------+-----------+
       |           |           |
    Dashboard    Alert      Analysis
       |           |           |
       +-----------+-----------+
                   |
              RUNBOOK
                   |
                ACTION
                   |
              VALIDATION
```

---

# 63. Golden rule

> **Good monitoring tells you that something is wrong. Great observability tells you what changed, why it matters, and where to look next.**

The advanced DBA should aim for:

```text id="m6p3v8"
Detect early
↓
Understand quickly
↓
Respond safely
↓
Recover fast
↓
Prevent recurrence
```

---

## Next — Lesson 109: Azure SQL Managed Instance Automation & Infrastructure as Code

We'll move into **Azure CLI, PowerShell, ARM/Bicep concepts, Terraform concepts, automated database provisioning, configuration management, monitoring automation, security automation, CI/CD integration, drift detection, and production DBA automation patterns.**
