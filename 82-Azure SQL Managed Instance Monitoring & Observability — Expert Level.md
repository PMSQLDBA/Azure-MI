# Lesson 114 — Azure SQL Managed Instance Monitoring & Observability — Expert Level

The goal of this lesson is to move from:

> **“We have monitoring enabled.”**

to:

> **“We can detect, diagnose, correlate, and predict SQL MI problems before they become major incidents.”**

---

# 1. Monitoring vs observability

These terms are related but not identical.

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
CPU increased
 ↓
Query X changed plan
 ↓
Logical reads increased
 ↓
Latency increased
```

---

# 2. Three pillars

A useful model is:

```text
Metrics
Logs
Traces
```

For database operations, add:

```text
Query performance
Audit/security telemetry
Events
```

---

# 3. SQL MI observability layers

Think:

```text
Azure platform
      ↓
SQL MI
      ↓
Database
      ↓
Query
      ↓
Application
      ↓
User experience
```

Monitoring only one layer leaves blind spots.

---

# 4. Azure Monitor

Azure Monitor provides the broader monitoring platform around Azure resources.

Conceptually:

```text
SQL MI
 ↓
Azure Monitor
 ↓
Metrics / Logs / Alerts
```

---

# 5. Metrics

Metrics are numeric time-series measurements.

Examples include:

```text
CPU utilization
Storage usage
I/O-related measurements
Connections
Workload activity
```

Metrics are excellent for:

```text
Trend detection
Threshold alerts
Capacity planning
Dashboards
```

---

# 6. Logs

Logs provide richer event information.

Examples:

```text
Errors
Auditing
Diagnostic events
Security activity
```

Logs are useful when you need context rather than a single number.

---

# 7. Diagnostic settings

Diagnostic settings can route supported resource logs/metrics to destinations such as:

```text
Log Analytics
Storage
Event Hubs
```

The exact categories available depend on the Azure service and current platform capabilities.

---

# 8. Why centralize telemetry?

Imagine:

```text
SQL MI A
SQL MI B
SQL MI C
SQL MI D
```

Instead of checking every instance manually:

```text
SQL MI
   ↓
Central telemetry
   ↓
Queries / dashboards / alerts
```

This is much easier to operate at scale.

---

# 9. Log Analytics

A Log Analytics workspace can act as a central location for supported Azure logs.

Conceptually:

```text
Azure resources
      ↓
Log Analytics
      ↓
KQL
      ↓
Analysis
```

---

# 10. KQL mindset

Kusto Query Language is designed for querying telemetry.

Conceptually:

```text
Source
| Filter
| Project
| Summarize
| Order
```

Example pattern:

```text
Telemetry
| where TimeGenerated > ago(...)
| summarize count() by ...
```

The exact table and column names depend on the telemetry source.

---

# 11. Don't treat logs as a dumping ground

A mature logging strategy defines:

```text
What to collect
Why collect it
Where to store it
How long to retain it
Who can access it
What alert depends on it
```

---

# 12. Retention

Retention should reflect:

```text
Operational troubleshooting
Security requirements
Compliance
Cost
Incident investigation
```

Keeping everything forever is usually neither necessary nor cost-effective.

---

# 13. Query Store

Query Store is one of your most valuable database-performance observability tools.

It helps capture historical information around:

```text
Queries
Plans
Runtime performance
Plan changes
```

---

# 14. Why historical data matters

Without history:

```text
Query is slow now
```

You may not know:

```text
Was it always slow?
What changed?
When did it become slow?
Which plan was previously good?
```

Query Store helps answer those questions.

---

# 15. DMVs

Dynamic Management Views provide current operational information.

Examples of categories:

```text
Sessions
Requests
Locks
Indexes
Query execution
Memory
Waits
```

Think:

```text
DMVs → current/near-current state
Query Store → historical query behavior
```

They complement one another.

---

# 16. Wait statistics

Waits help explain:

> **What is the workload waiting for?**

For example:

```text
CPU
I/O
Locks
Memory
Parallelism
Network
```

Don't treat a wait name in isolation.

Interpret it within workload context.

---

# 17. Blocking monitoring

A useful dashboard should reveal:

```text
Current blockers
Blocked sessions
Blocking duration
Blocking chain
Transaction age
```

---

# 18. Head blocker alert

A useful operational alert might detect:

```text
Long-running blocking session
```

But avoid blindly paging on every short block.

Use thresholds based on workload expectations.

---

# 19. Deadlock monitoring

Deadlocks should be captured and analyzed.

Monitor:

```text
Deadlock frequency
Affected workload
Victim queries
Objects involved
Time of occurrence
```

A repeated deadlock is a design problem, not simply a monitoring event.

---

# 20. CPU alerts

Simple alert:

```text
CPU > 90%
```

But CPU threshold alone can create false positives.

A better approach considers:

```text
CPU
+
Duration
+
Baseline
+
Business impact
```

---

# 21. Dynamic thresholds

Suppose normal CPU is:

```text
20–40%
```

A sudden:

```text
80%
```

may be significant.

But another workload may normally run at:

```text
75–85%
```

A fixed threshold isn't equally useful in both environments.

---

# 22. Baseline monitoring

Learn normal behavior:

```text
Monday 09:00
Monday 12:00
Monday 18:00
Weekend
Month-end
Quarter-end
```

Then detect deviations.

---

# 23. Seasonality

Some databases naturally have:

```text
Morning spikes
End-of-month processing
Nightly ETL
Weekend reporting
```

Don't alert simply because a known scheduled workload occurs.

---

# 24. SLI

A **Service Level Indicator** measures actual service behavior.

Examples:

```text
Transaction latency
Successful transaction percentage
Database availability
Query error rate
```

---

# 25. SLO

A **Service Level Objective** defines the target.

Example:

```text
99.9% successful transactions
```

or:

```text
P95 checkout latency < 500 ms
```

---

# 26. SLA

An SLA is a formal service commitment, often involving contractual terms.

Don't confuse:

```text
SLI → measurement
SLO → target
SLA → formal commitment
```

---

# 27. Database SLO examples

For a critical OLTP system:

```text
Availability ≥ 99.9%
P95 critical-query latency < target
Error rate < target
```

The actual values must come from business requirements.

---

# 28. User-centric monitoring

Database metrics alone don't tell the entire story.

Suppose:

```text
SQL CPU = 40%
```

Looks healthy.

But:

```text
Checkout P95 = 8 seconds
```

The user is still experiencing an incident.

---

# 29. Application correlation

You want:

```text
User request
   ↓
Application request ID
   ↓
SQL operation
   ↓
Query
   ↓
Database resource usage
```

This allows end-to-end troubleshooting.

---

# 30. Incident correlation

Imagine:

```text
14:03 Application latency ↑
14:04 SQL query duration ↑
14:04 CPU ↑
14:04 Plan changed
```

Now you have a strong correlation chain.

---

# 31. Time synchronization

Correlation requires accurate timestamps.

When analyzing incidents, align:

```text
Application logs
Database telemetry
Azure telemetry
Deployment logs
Security logs
```

using consistent timestamps/time zones.

---

# 32. Deployment correlation

Every production deployment should ideally have identifiable metadata:

```text
Version
Commit
Migration
Deployment time
Environment
```

Then monitoring can ask:

> **Did performance change immediately after deployment?**

---

# 33. Change correlation

Major performance incidents often follow changes such as:

```text
Index deployment
Schema change
Statistics change
Application release
Configuration change
Data-volume increase
```

Monitor changes alongside performance.

---

# 34. Golden signals

A useful general monitoring model is:

```text
Latency
Traffic
Errors
Saturation
```

For databases, translate them appropriately.

### Latency

```text
Query duration
Transaction latency
```

### Traffic

```text
Queries/sec
Transactions/sec
Connections
```

### Errors

```text
Failed requests
SQL errors
Deadlocks
Timeouts
```

### Saturation

```text
CPU
I/O
Memory
Storage
Concurrency
```

---

# 35. Dashboard design

Avoid dashboards containing 100 unrelated charts.

Create focused views.

### Executive

```text
Availability
SLO
Critical transaction latency
Major incidents
```

### DBA

```text
CPU
I/O
Waits
Blocking
Deadlocks
Query regressions
Storage
```

### Security

```text
Authentication
Audit events
Privileged changes
Security alerts
```

---

# 36. Critical workload dashboard

For a payment database:

```text
Checkout success rate
Checkout P95 latency
Transaction errors
Database CPU
Blocking
Deadlocks
Top queries
```

This is more useful than a generic "SQL health" screen.

---

# 37. Alert severity

Not every event should wake an engineer.

Example:

```text
INFO
Normal event

WARNING
Investigate

CRITICAL
Immediate action
```

---

# 38. Alert fatigue

If every alert is:

```text
CRITICAL
```

then eventually:

```text
CRITICAL = ignored
```

Alert quality matters as much as alert quantity.

---

# 39. Actionable alerts

Every page-worthy alert should answer:

```text
What happened?
Why does it matter?
What should I check?
Who owns it?
```

---

# 40. Example

Bad alert:

```text
CPU high
```

Better:

```text
SQL MI CPU > 90% for 10 minutes.
Critical checkout latency has exceeded SLO.
Top CPU query: Query X.
Deployment detected 7 minutes ago.
```

Now the engineer can act immediately.

---

# 41. Alert runbook links

Each critical alert should ideally map to a runbook:

```text
Alert
 ↓
Runbook
 ↓
Diagnosis
 ↓
Mitigation
 ↓
Escalation
```

---

# 42. Monitoring database connections

Monitor:

```text
Connection count
Connection failures
Connection duration
Connection saturation
```

Sudden changes can indicate:

```text
Application deployment
Connection leak
Pool configuration problem
Traffic spike
```

---

# 43. Connection storms

After an outage, thousands of clients may reconnect simultaneously.

```text
Database recovers
       ↓
All clients reconnect
       ↓
Connection storm
       ↓
Resource pressure
```

This is why application retry/backoff design is part of database observability.

---

# 44. Storage monitoring

Track:

```text
Used storage
Growth rate
Available capacity
I/O behavior
```

Don't wait until storage is nearly exhausted.

---

# 45. Capacity forecasting

Suppose:

```text id="growth-example"
January → 2 TB
February → 2.2 TB
March → 2.4 TB
April → 2.6 TB
```

Trend:

```text
~200 GB/month
```

You can forecast when capacity constraints may become relevant.

---

# 46. Query performance trend

Track important queries over time:

```text
Week 1 → 100 ms
Week 2 → 120 ms
Week 3 → 150 ms
Week 4 → 800 ms
```

This is more informative than looking only at today's average.

---

# 47. P95 vs average

Suppose:

```text
Average = 100 ms
P95 = 700 ms
```

Most queries are fast, but the tail is problematic.

For user-facing workloads, percentile monitoring is often more useful than average alone.

---

# 48. Error budgets

If an SLO allows a certain amount of failure, the difference between target and actual performance creates an **error budget**.

Conceptually:

```text
SLO
 ↓
Allowed failure
 ↓
Error budget
```

When the budget is being consumed rapidly, prioritize reliability work.

---

# 49. Proactive monitoring

Reactive:

```text
User complains
 ↓
Investigate
```

Proactive:

```text
Trend detected
 ↓
Capacity forecast
 ↓
Action before incident
```

That's the goal of mature observability.

---

# 50. Predictive signals

Potential early warnings:

```text
Storage growth accelerating
Query duration trending upward
CPU baseline increasing
Blocking becoming more frequent
Replication lag increasing
Connection count rising
```

---

# 51. Monitoring replication/DR

For DR-enabled architectures, monitor:

```text
Replication health
Lag
Secondary availability
Last successful synchronization
```

A DR secondary that silently falls behind is a major risk.

---

# 52. Monitoring backups

Don't assume:

> "Backups are configured."

Monitor:

```text
Backup success
Backup failures
Retention
Recovery capability
```

And periodically test restoration.

---

# 53. Security monitoring

Security observability should include:

```text
Authentication failures
Unusual login patterns
Privilege changes
Suspicious access
Audit events
Security recommendations
```

---

# 54. Audit vs operational logs

Keep these concepts separate:

```text
Operational logs
→ Is the service working?

Audit logs
→ Who did what?
```

Both are valuable but serve different purposes.

---

# 55. Centralized monitoring architecture

A mature environment might look like:

```text
                 SQL MI
                   |
          +--------+--------+
          |        |        |
       Metrics   Logs    Query Store
          |        |        |
          +--------+--------+
                   |
             Azure Monitor
                   |
             Log Analytics
                   |
          +--------+--------+
          |        |        |
      Dashboards Alerts  Analytics
          |        |        |
          +--------+--------+
                   |
               On-call
```

---

# 56. Application + database observability

Extend the architecture:

```text
User
 ↓
Application telemetry
 ↓
Request trace
 ↓
SQL query
 ↓
SQL MI telemetry
 ↓
Azure platform telemetry
```

This gives end-to-end visibility.

---

# 57. Incident example

Users report:

> "The website is slow."

Monitoring shows:

```text
Application P95 ↑
       ↓
SQL query duration ↑
       ↓
CPU normal
       ↓
Blocking ↑
       ↓
One long transaction
```

Root cause:

```text
Long transaction
```

Not:

```text
"Azure SQL is slow."
```

---

# 58. Another incident

Monitoring shows:

```text
Application latency ↑
SQL CPU ↑
```

Query Store:

```text
Query X
Plan A → 200 ms
Plan B → 5 sec
```

Deployment:

```text
Plan B appeared shortly after index/schema change
```

Now you have a strong investigation path.

---

# 59. Observability during deployments

Monitor:

```text
Before deployment
      ↓
During deployment
      ↓
Immediately after
      ↓
15 min
      ↓
1 hour
      ↓
Normal business cycle
```

Some regressions don't appear immediately.

---

# 60. Black-box vs white-box monitoring

### Black-box

Monitor from the user's perspective:

```text
Can I complete the transaction?
```

### White-box

Monitor internal components:

```text
CPU
Waits
Locks
Queries
Storage
```

You need both.

---

# 61. Synthetic transactions

A synthetic monitor can periodically execute a safe representative transaction:

```text
Login test
 ↓
Read test
 ↓
Critical workflow test
```

This detects problems before real users report them.

Be careful that synthetic workloads don't distort production behavior.

---

# 62. Monitoring ownership

Every metric should ideally have an owner.

For example:

```text
Database health → DBA
Application latency → App team
Identity → IAM/security
Network → Network team
```

Cross-team ownership must be explicit.

---

# 63. Observability cost

Telemetry has cost.

Optimize:

```text
Collection
Retention
Sampling
Destination
Query frequency
```

Collect enough information to diagnose incidents without creating unnecessary cost.

---

# 64. Monitoring anti-patterns

### Anti-pattern 1

```text
CPU dashboard only
```

### Anti-pattern 2

```text
100 alerts
```

### Anti-pattern 3

```text
No historical data
```

### Anti-pattern 4

```text
No application correlation
```

### Anti-pattern 5

```text
Alerts without runbooks
```

### Anti-pattern 6

```text
No DR/backup monitoring
```

---

# 65. Expert monitoring checklist

```text
PLATFORM
☐ CPU
☐ Storage
☐ I/O
☐ Connections
☐ Availability

DATABASE
☐ Query Store
☐ Waits
☐ Blocking
☐ Deadlocks
☐ Long-running queries
☐ TempDB behavior

APPLICATION
☐ Request latency
☐ Error rate
☐ Throughput
☐ Critical transactions

SECURITY
☐ Authentication
☐ Audit
☐ Privilege changes
☐ Security alerts

DR
☐ Replication health
☐ Lag
☐ Backup health
☐ Recovery testing

OPERATIONS
☐ Alerts
☐ Dashboards
☐ Runbooks
☐ SLOs
☐ Capacity forecasting
```

---

# 66. Interview question

### "What is the difference between monitoring and observability?"

> "Monitoring tells us whether known conditions are healthy or unhealthy. Observability gives us enough telemetry to investigate why the system behaves the way it does, including unexpected failure modes."

---

# 67. Interview question

### "What would you monitor on SQL MI?"

Strong answer:

```text
CPU
Storage
I/O
Connections
Waits
Blocking
Deadlocks
Query Store
Long-running queries
Errors
Backup health
DR/replication
Security events
Application latency
```

Then add:

> "I would correlate these with business SLOs rather than monitoring infrastructure metrics in isolation."

---

# 68. Interview question

### "How do you avoid alert fatigue?"

> "I prioritize actionable alerts, use appropriate thresholds and durations, distinguish warning from paging severity, account for normal workload patterns, and attach runbooks to critical alerts. I regularly review alerts to remove noisy ones."

---

# 69. Interview question

### "How would you detect a performance regression?"

> "I'd establish a baseline, monitor Query Store and application latency over time, correlate changes with deployments or data changes, compare execution plans and runtime statistics, and alert when critical workload SLOs or statistically meaningful performance thresholds regress."

---

# 70. Interview question

### "What is the most important dashboard?"

Not necessarily CPU.

For a business-critical workload:

```text
Business transaction success
Business transaction latency
Error rate
Database availability
Critical query health
```

Then drill down into infrastructure metrics.

---

# 71. Golden rule

> **Monitor what users care about, then collect enough database telemetry to explain why those outcomes change.**

Think:

```text
USER EXPERIENCE
      ↓
APPLICATION
      ↓
DATABASE WORKLOAD
      ↓
QUERY
      ↓
RESOURCE
      ↓
ROOT CAUSE
```

---

# 72. Final mental model

```text
                 BUSINESS SLO
                      ↓
               USER EXPERIENCE
                      ↓
              APPLICATION TRACE
                      ↓
                 SQL QUERY
                      ↓
          +-----------+-----------+
          |           |           |
        PLAN        WAITS       ERRORS
          |           |           |
          +-----------+-----------+
                      ↓
               SQL MI RESOURCES
                      ↓
              AZURE MONITOR
                      ↓
              LOG ANALYTICS
                      ↓
              ALERT / ANALYSIS
                      ↓
                 RUNBOOK
                      ↓
                 RESPONSE
```

That's the foundation of **expert SQL MI observability**.

---

## Next — Lesson 115: Azure SQL Managed Instance Automation & Infrastructure as Code

We'll cover **Bicep, Terraform concepts, Azure CLI/PowerShell automation, policy, deployment pipelines, configuration drift, automated provisioning, database initialization, environment promotion, secrets, managed identities, automated validation, and building a repeatable SQL MI platform from code.**
