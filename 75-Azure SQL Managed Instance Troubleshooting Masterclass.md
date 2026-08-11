# Lesson 107 — Azure SQL Managed Instance Troubleshooting Masterclass

This is where the previous lessons come together.

The goal is not to memorize hundreds of commands. The goal is to develop a **repeatable incident-response method** that lets you move from:

```text
"SQL is slow."
```

to:

```text
"Here is the bottleneck, here is the root cause, here is the safest fix, and here is the evidence that it worked."
```

---

# 1. The DBA troubleshooting framework

Use this sequence:

```text
1. Confirm
2. Scope
3. Measure
4. Classify
5. Isolate
6. Hypothesize
7. Test
8. Remediate
9. Validate
10. Document
```

Never jump directly from:

```text
Symptom → Fix
```

---

# 2. Step 1 — Confirm the problem

Someone says:

> "The database is down."

Don't immediately assume SQL MI is down.

Check:

```text id="q4m7x2"
Is the SQL MI reachable?
Can a known-good client connect?
Can another database connect?
Is the application failing or the database?
Is the problem current?
```

---

# 3. Step 2 — Scope the incident

Determine whether the issue affects:

```text id="n8c3v6"
One query
One database
One application
Multiple applications
Entire SQL MI
Network path
Regional environment
```

This dramatically narrows the search.

---

# 4. Example

Suppose:

```text id="k6p2r9"
Application A → slow
Application B → normal
Application C → normal
```

Likely:

```text id="x7m4q1"
Application A
or
its workload
```

rather than immediately blaming SQL MI.

---

# 5. Scope by time

Ask:

```text id="v3n8m5"
When did it start?
Was it sudden?
Was it gradual?
Is it intermittent?
Is it still happening?
```

Then correlate with:

```text id="m9c2k7"
Deployment
Configuration change
Data load
Traffic spike
Maintenance
Security change
```

---

# 6. Step 3 — Measure

Collect evidence.

For performance:

```text id="f5x8p3"
CPU
Memory
I/O
Duration
Reads
Writes
Waits
Blocking
Connections
```

For availability:

```text id="r2k6m9"
Connectivity
Errors
Health
Failover events
Network path
Authentication
```

---

# 7. Step 4 — Classify

Most incidents fall into categories:

```text id="c8m4v1"
Availability
Performance
Concurrency
Capacity
Security
Deployment
Data
Network
```

Classification helps prevent random investigation.

---

# 8. Availability incident

Symptoms:

```text id="a7q3m8"
Connection failures
Timeouts
Application cannot connect
```

Start with:

```text id="n5v9k2"
Can anyone connect?
Is SQL MI reachable?
Are credentials valid?
Is DNS working?
Is networking working?
Is the issue application-specific?
```

---

# 9. Performance incident

Symptoms:

```text id="x4m7p1"
Queries slower
CPU higher
Latency increased
```

Ask:

```text id="k9c2v6"
Doing work or waiting?
```

Then inspect:

```text id="r3n8q5"
Query Store
Execution plans
Waits
Blocking
CPU
I/O
Memory
```

---

# 10. Concurrency incident

Symptoms:

```text id="v8p2m4"
Sudden latency
Low CPU
Queries waiting
```

Investigate:

```text id="j5x7c1"
Blocking
Locks
Deadlocks
Long transactions
```

---

# 11. Capacity incident

Symptoms:

```text id="m2k6r9"
Persistent resource saturation
```

Look at:

```text id="w7n3p5"
CPU
Storage
Memory
Connections
I/O
Workload growth
```

---

# 12. Security incident

Symptoms:

```text id="q8x4m2"
Unexpected login
Unauthorized access
Permission failure
Suspicious activity
```

Investigate:

```text id="c5m9v7"
Identity
Authentication
Authorization
Audit logs
Network activity
Security alerts
```

---

# 13. Deployment incident

Symptoms:

```text id="p3k7x1"
Performance changed immediately after release
Errors after schema change
Application/database incompatibility
```

Correlate:

```text id="n6m2q8"
Deployment timestamp
Schema changes
Plan changes
Application version
```

---

# 14. Network incident

Symptoms:

```text id="v4r8m3"
Connection timeout
Intermittent connection
High application latency
```

Separate:

```text id="j9c2x6"
Can SQL MI be reached?
```

from:

```text id="s5m7p1"
Can authentication succeed?
```

from:

```text id="a8n4k3"
Can the application execute SQL?
```

---

# 15. The troubleshooting ladder

Use:

```text id="e7m3q8"
Layer 1 → Application
Layer 2 → Network
Layer 3 → Authentication
Layer 4 → SQL connectivity
Layer 5 → Query/workload
Layer 6 → Database
Layer 7 → SQL MI/platform
Layer 8 → Azure regional infrastructure
```

Don't skip layers.

---

# 16. Scenario 1 — CPU suddenly reaches 95%

Incident:

```text id="b2x8m4"
CPU = 95%
Application latency = high
```

First question:

> Is CPU genuinely the bottleneck?

---

# 17. Find top CPU consumers

Identify:

```text id="r6k3p9"
Queries
Applications
Sessions
Time period
```

Then compare against baseline.

---

# 18. Check recent changes

Ask:

```text id="x9m2c5"
Was there a deployment?
New query?
New index?
Statistics change?
Traffic spike?
Batch job?
```

---

# 19. Inspect execution plan

Look for:

```text id="p4v8n1"
Large scans
Expensive joins
Bad cardinality estimates
Parallelism
Repeated lookups
Sort/hash operations
```

---

# 20. Possible resolution

Example:

```text id="j7c3m8"
New query
 ↓
Table scan
 ↓
High CPU
```

Solution might be:

```text id="q2n6x4"
Query rewrite
+
Appropriate index
```

Not:

```text id="z8m1p5"
Immediately scale everything
```

---

# 21. Scenario 2 — Everything is slow but CPU is only 25%

This is a classic clue.

Possible causes:

```text id="m3k7v9"
Blocking
I/O waits
Memory waits
Network/application latency
```

Start with waits and blocking.

---

# 22. Blocking chain

Suppose:

```text id="h8x4q2"
Session 101
   ↓ blocks
Session 205
   ↓ blocks
Session 307
   ↓ blocks
Session 401
```

Session 101 is the head blocker.

Investigate it.

---

# 23. Head blocker investigation

Ask:

```text id="r5m9c3"
What query is running?
How long has transaction been open?
Who owns the session?
Why is the transaction open?
What locks are held?
```

Don't kill sessions blindly.

---

# 24. Scenario 3 — Deadlocks increase after deployment

Symptoms:

```text id="v7p2k8"
Deadlocks increased
Application errors
```

Compare:

```text id="x3m9q4"
Before deployment
vs
After deployment
```

Look for:

```text id="j8c4n6"
Changed transaction order
New query
Missing index
Longer transactions
New access pattern
```

---

# 25. Deadlock reasoning

Example:

```text id="n2q7x5"
Transaction A
locks Customer
then Order

Transaction B
locks Order
then Customer
```

Different lock order creates a cycle.

Solution:

```text id="m6p3v8"
Consistent access order
```

where appropriate.

---

# 26. Scenario 4 — Query was fast yesterday, slow today

Think:

```text id="r4x8k2"
Plan regression
Statistics
Data distribution
Parameter sensitivity
Index change
```

Start with Query Store.

---

# 27. Query Store comparison

Find:

```text id="w5n7m1"
Old plan
New plan
Runtime
CPU
Reads
```

If the plan changed:

```text id="c8q2v6"
Investigate why.
```

---

# 28. Parameter sensitivity

Example:

```text id="k3m9p5"
Customer A → 10 rows
Customer B → 10 million rows
```

A plan optimized for A may perform badly for B.

Look for:

```text id="x7r2c4"
Different parameter values
Large runtime variation
Plan changes
```

---

# 29. Scenario 5 — Storage warning

Suppose:

```text id="v8m4q1"
Storage = 88%
```

Don't wait until:

```text id="j5n2c7"
100%
```

Investigate:

```text id="m3x9p6"
Growth trend
Largest databases
Data growth
Log growth
Retention
Temporary usage
```

---

# 30. Capacity forecast

Suppose:

```text id="p6k2m8"
Current free capacity = 100 GB
Growth = 10 GB/day
```

Approximate remaining time:

```text id="q9x4c3"
100 / 10 = 10 days
```

Now you have a planning problem instead of an emergency.

---

# 31. Scenario 6 — Application cannot connect

Break it into layers:

```text id="r7m3v8"
DNS?
 ↓
Network?
 ↓
Port/path?
 ↓
TLS?
 ↓
Authentication?
 ↓
Authorization?
 ↓
Database?
```

This prevents guessing.

---

# 32. Authentication failure

If network connectivity succeeds but authentication fails:

Investigate:

```text id="c5x8n2"
Credential
Identity
Token
Entra configuration
SQL login
Password
Account status
```

Don't start changing firewall rules if the connection already reaches SQL MI.

---

# 33. Authorization failure

If:

```text id="m8q2v4"
Login succeeds
```

but:

```text id="j3n7p6"
SELECT fails
```

Investigate:

```text id="x4c9m1"
Database user
Role membership
Object permissions
Schema permissions
```

This is an authorization problem, not an authentication problem.

---

# 34. Scenario 7 — Tempdb pressure

Symptoms may include:

```text id="v2p8k5"
Spills
Heavy tempdb usage
Slow sorts
Slow hash operations
Version-store pressure
```

Investigate workload first.

Find:

```text id="n6m3x9"
Which queries are using tempdb?
Why?
Are they spilling?
Are transactions long?
```

---

# 35. Scenario 8 — Memory grant issue

Suppose:

```text id="q4r7m2"
Query requests huge memory grant
```

Other queries suffer.

Investigate:

```text id="c8x1p5"
Execution plan
Cardinality estimates
Sorts
Hash operations
Concurrent grants
```

Possible root cause:

```text id="j5m9v3"
Bad cardinality estimate
```

---

# 36. Scenario 9 — I/O latency

Suppose:

```text id="x7k2n4"
CPU = normal
I/O latency = high
```

Look for:

```text id="p3m8q6"
Large scans
Heavy reads
Memory pressure
Storage behavior
Queries generating excessive I/O
```

---

# 37. Don't confuse reads with bad SQL

A query that reads 1 GB may be appropriate if it genuinely needs that data.

Ask:

```text id="w4n9c2"
How many rows are returned?
How many are processed?
Can the query read less?
Can indexing reduce unnecessary reads?
```

---

# 38. Scenario 10 — Failed deployment

Application errors after schema deployment.

First:

```text id="r6x3m8"
Identify exact failure
```

Then:

```text id="k2p7v4"
Compare schema
Compare application version
Check migration
Check permissions
Check dependencies
```

---

# 39. Compatibility failure

Example:

```text id="q8m1c5"
Application v2
expects column NewStatus

Database
doesn't have NewStatus
```

This is a deployment compatibility problem.

Use controlled migration strategies such as:

```text id="v3n7x9"
Expand
 ↓
Deploy
 ↓
Migrate
 ↓
Contract
```

---

# 40. Scenario 11 — Security incident

Unexpected identity:

```text id="m5k8p2"
user/service account
accesses sensitive table
```

Response:

```text id="c7x3n6"
Detect
 ↓
Contain
 ↓
Preserve evidence
 ↓
Investigate
 ↓
Remove unauthorized access
 ↓
Recover
 ↓
Review controls
```

---

# 41. Scenario 12 — Regional outage

Primary region becomes unavailable.

Don't improvise.

Use the DR runbook:

```text id="z4m9q1"
Declare incident
 ↓
Assess outage
 ↓
Activate DR
 ↓
Validate database
 ↓
Recover application
 ↓
Redirect traffic
 ↓
Validate business transactions
 ↓
Monitor
```

---

# 42. The 5-question incident method

When under pressure, ask:

### 1. What changed?

```text id="a6r3m8"
Deployment?
Traffic?
Configuration?
Data?
Platform?
```

### 2. What is affected?

```text id="j9k2p5"
One query?
Database?
Application?
Entire region?
```

### 3. Is it doing work or waiting?

```text id="x4v7c1"
CPU?
Wait?
Blocking?
I/O?
```

### 4. What evidence proves the hypothesis?

```text id="m8q3n6"
Metrics
Plans
Logs
Query Store
Audit
```

### 5. What is the safest reversible action?

```text id="p2c7r5"
Mitigate first.
Then fix root cause.
```

---

# 43. Mitigation vs root-cause fix

This distinction is critical.

### Mitigation

```text id="k6m3v9"
Reduce immediate impact
```

Example:

```text id="q8x1p4"
Scale resources temporarily
```

### Root-cause fix

```text id="n5r7c2"
Fix inefficient query
```

A good incident response may use both.

---

# 44. Don't make multiple changes simultaneously

Bad:

```text id="v3m8k1"
Change index
Change query
Change configuration
Scale MI
Restart application
```

Now you don't know what fixed the issue.

Better:

```text id="x7p2c5"
One controlled change
 ↓
Measure
 ↓
Next change if needed
```

---

# 45. Incident timeline

Document:

```text id="r9m4n6"
10:02 Alert
10:05 Investigation started
10:08 Root cause suspected
10:12 Mitigation applied
10:18 Latency normalized
10:30 Root-cause analysis complete
```

This is extremely valuable for postmortems.

---

# 46. Evidence collection

For every incident, preserve:

```text id="m2x8p4"
Metrics
Logs
Query Store data
Execution plans
Error messages
Deployment IDs
Configuration changes
Timeline
```

---

# 47. Five whys

Example:

### Problem

```text id="j7c3m9"
Application latency increased.
```

### Why?

```text id="v5n8q2"
Database queries became slow.
```

### Why?

```text id="x4m1p6"
Query began scanning large table.
```

### Why?

```text id="q8r3k7"
Execution plan changed.
```

### Why?

```text id="n6c2v9"
Cardinality estimate changed after data growth.
```

### Why?

```text id="m3p7x5"
Statistics/plan-management process did not account for workload change.
```

Now you've reached a systemic cause.

---

# 48. Postmortem

A good postmortem contains:

```text id="c5x9m2"
What happened?
Impact?
Timeline?
Root cause?
Contributing factors?
Why wasn't it detected earlier?
What fixed it?
What prevents recurrence?
```

Avoid a blame-oriented approach.

---

# 49. Detection improvement

Ask:

> **Could monitoring have detected this before users did?**

If yes:

```text id="r8m4p1"
Create alert
Improve dashboard
Add health check
```

---

# 50. Prevention improvement

Ask:

> **How do we stop this from happening again?**

Examples:

```text id="k2x7v5"
Query optimization
Capacity planning
Deployment validation
Access review
DR testing
Automation
```

---

# 51. Troubleshooting toolkit

A senior SQL MI DBA should be comfortable with:

```text id="v6m3q8"
Query Store
Execution plans
DMVs
Wait analysis
Blocking analysis
Deadlock analysis
Azure Monitor
Logs
Alerts
Azure CLI
PowerShell
Networking diagnostics
Application telemetry
```

The important skill is knowing **when to use each tool**.

---

# 52. Tool-selection model

```text id="p9x2m7"
Historical query problem
→ Query Store

Current query/session problem
→ DMVs

Plan problem
→ Execution plan

Blocking
→ Lock/session investigation

Platform/resource problem
→ Azure monitoring

Connectivity
→ Network + authentication investigation

Security
→ Audit + security telemetry
```

---

# 53. Troubleshooting anti-patterns

Avoid:

```text id="x4m8q2"
Restarting without evidence
Killing random sessions
Adding indexes blindly
Scaling immediately
Disabling security controls
Changing many settings simultaneously
Ignoring application behavior
Assuming every issue is SQL
```

---

# 54. Production incident example

Imagine:

```text id="n7c3m5"
09:00 normal

09:15 deployment

09:17 CPU rises to 92%

09:20 application latency increases
```

Your first hypothesis:

```text id="q2v8k4"
Deployment-related workload change
```

Check:

```text id="m5x1p9"
Query Store
Plans
CPU
Reads
Deployment changes
```

---

# 55. Investigation result

You discover:

```text id="j8r3c6"
New query
 ↓
Large scan
 ↓
High CPU
```

Then:

```text id="v4m7n2"
Existing index no longer supports new access pattern
```

---

# 56. Mitigation

Possible immediate action:

```text id="p6x2k8"
Rollback deployment
```

if safe and approved.

Then:

```text id="n3m9c5"
Optimize query/index
```

---

# 57. Validation

After fix:

```text id="x8v4q1"
CPU = 45%
Latency = normal
Query duration = 200 ms
No new blocking
No unexpected write regression
```

Now the incident is validated rather than merely assumed fixed.

---

# 58. Senior DBA incident mindset

Don't say:

> "I think the index is missing."

Say:

> "Query Store shows a regression beginning at 09:17. The new plan performs a large scan and logical reads increased by approximately X. The deployment introduced the query pattern. We can reproduce the behavior and the proposed index reduces reads without unacceptable write overhead."

That's evidence-based troubleshooting.

---

# 59. Incident severity

Example:

```text id="q7m2x9"
P1 → Business unavailable
P2 → Major degradation
P3 → Limited impact
P4 → Minor/non-urgent
```

Exact definitions should come from your organization's incident policy.

---

# 60. Troubleshooting checklist

```text id="f3k8m1"
CONFIRM
☐ Is the problem real?
☐ Is it current?

SCOPE
☐ Who is affected?
☐ What is affected?
☐ When did it start?

MEASURE
☐ CPU
☐ Memory
☐ I/O
☐ Waits
☐ Duration
☐ Reads
☐ Blocking

CORRELATE
☐ Deployment
☐ Configuration
☐ Traffic
☐ Data change
☐ Platform event

DIAGNOSE
☐ Query Store
☐ Plan
☐ DMVs
☐ Logs
☐ Network
☐ Security telemetry

FIX
☐ Safest mitigation
☐ Root-cause fix
☐ Controlled change

VALIDATE
☐ Performance recovered
☐ Errors stopped
☐ No regression

DOCUMENT
☐ Timeline
☐ Root cause
☐ Prevention
```

---

# 61. Interview question

### "The database is slow. What do you do?"

A strong senior answer:

> "First I scope the issue and establish whether the latency is caused by database work or waiting. I check current resource usage, waits, blocking, Query Store, and execution plans, then correlate the onset with deployments or workload changes. I form a testable hypothesis, apply the smallest safe mitigation, validate the result, and then address the underlying root cause."

---

# 62. Interview question

### "CPU is 95%. Is CPU necessarily the root cause?"

> "Not necessarily. High CPU is a symptom of resource consumption. I would identify the workload generating CPU, determine whether it's expected or caused by a regression, and inspect query plans, execution counts, waits, and recent changes before deciding whether to tune or scale."

---

# 63. Interview question

### "Everything is slow but CPU is low. What do you investigate?"

> "I'd strongly consider waiting rather than computation. I'd investigate blocking, locks, I/O latency, memory pressure, network/application latency, and other relevant wait categories."

---

# 64. Interview question

### "A query suddenly became slow after a deployment. What do you check?"

```text id="m4x7p2"
Deployment timing
 ↓
Query Store
 ↓
Old vs new plan
 ↓
Schema/index changes
 ↓
Statistics
 ↓
Parameter sensitivity
 ↓
Application behavior
```

---

# 65. Final troubleshooting model

Memorize this:

```text id="z8q3m6"
              SYMPTOM
                 |
                 ↓
               SCOPE
                 |
                 ↓
              MEASURE
                 |
                 ↓
       +---------+---------+
       |                   |
      WORK                WAIT
       |                   |
    Query plan          Blocking
    CPU                 I/O
    Reads               Locks
    Estimates           Memory
       |                   |
       +---------+---------+
                 |
                 ↓
              ROOT CAUSE
                 |
                 ↓
              MITIGATE
                 |
                 ↓
                FIX
                 |
                 ↓
              VALIDATE
                 |
                 ↓
             DOCUMENT
```

---

# 66. Golden rule

> **Don't troubleshoot from assumptions. Troubleshoot from evidence.**

And the senior DBA mindset is:

```text id="u5m9x2"
Observe
→ Measure
→ Correlate
→ Hypothesize
→ Test
→ Change
→ Validate
→ Learn
```

Once you can do this consistently, you're no longer just operating SQL MI—you are **engineering reliability for the database platform**.

---

## Next — Lesson 108: Azure SQL Managed Instance Monitoring & Observability — Advanced

We'll build the complete observability layer: **metrics, logs, Azure Monitor, Log Analytics, alerts, dashboards, Query Store telemetry, SLOs/SLIs, error budgets, proactive monitoring, anomaly detection, alert design, noise reduction, and an end-to-end production monitoring architecture.**
