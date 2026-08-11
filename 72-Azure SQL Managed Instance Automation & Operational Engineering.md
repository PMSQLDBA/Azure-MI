# Lesson 104 — Azure SQL Managed Instance Automation & Operational Engineering

This lesson moves from **manually operating SQL MI** to building systems that are **repeatable, observable, safe, and automated**.

The key principle:

> **If a task is repetitive, predictable, and rules-based, it should be a candidate for automation.**

But:

> **Automation without guardrails simply makes mistakes happen faster.**

---

# 1. Why automate DBA operations?

Imagine doing this every week:

```text
Check health
Check storage
Check backups
Check alerts
Review performance
Generate report
```

Manually:

```text
Human
 ↓
Portal
 ↓
Query
 ↓
Copy results
 ↓
Spreadsheet
 ↓
Email
```

Automated:

```text
Scheduler
 ↓
Health checks
 ↓
Analysis
 ↓
Alert/report
```

Automation improves:

```text
Consistency
Speed
Repeatability
Auditability
```

---

# 2. What should be automated?

Good candidates:

```text id="9u6x1b"
Health checks
Monitoring
Reporting
Configuration validation
Deployment
Routine maintenance
Backup verification
Capacity checks
Alert routing
```

Poor candidates for blind automation:

```text id="k6k4xa"
Destructive production changes
Unknown remediation
Data deletion
Unreviewed security changes
```

---

# 3. Automation maturity

Think in levels:

```text id="6x8j3f"
Level 1
Manual

Level 2
Scripted

Level 3
Scheduled

Level 4
Event-driven

Level 5
Self-healing
```

---

# 4. Level 1 — Manual

Example:

```text id="a7m2p1"
Open Azure portal
Check CPU
Check storage
Check health
```

Easy, but inconsistent.

---

# 5. Level 2 — Scripted

Create a repeatable script:

```text id="0l7h6v"
Script
 ↓
Collect health data
 ↓
Output result
```

Now every DBA runs the same checks.

---

# 6. Level 3 — Scheduled

Run automatically:

```text id="r6e4d8"
Every day at 06:00
       ↓
Health script
       ↓
Report
```

No human needs to remember.

---

# 7. Level 4 — Event-driven

Example:

```text id="7f3n4w"
Storage threshold exceeded
        ↓
Alert
        ↓
Automation
        ↓
Notify DBA
```

---

# 8. Level 5 — Self-healing

Example:

```text id="3z7g1p"
Known transient condition
        ↓
Automated remediation
        ↓
Validate
        ↓
Escalate if unsuccessful
```

Self-healing should be used carefully.

---

# 9. Azure CLI

Azure CLI is useful for scripting Azure resource operations.

Conceptually:

```text id="h3w8z2"
CLI
 ↓
Azure authentication
 ↓
Resource operation
 ↓
Result
```

Typical operations include:

```text
Create
Read
Update
Delete
List
Configure
```

---

# 10. PowerShell

PowerShell is particularly useful in Microsoft-heavy environments.

It can automate:

```text id="p4s6q1"
Azure resources
SQL administration
Reporting
Scheduled tasks
Operational workflows
```

---

# 11. Azure Resource Manager

Azure resources are managed through Azure Resource Manager (ARM).

Conceptually:

```text id="z8y2x4"
Your script
    ↓
Azure API / ARM
    ↓
SQL MI resource
```

This allows automation to interact with Azure resources programmatically.

---

# 12. Infrastructure as Code

Instead of manually creating infrastructure:

```text id="n7c2m8"
Click
Click
Click
Configure
Configure
```

define the desired environment as code.

Conceptually:

```text id="1r6v9p"
Code
 ↓
Deployment engine
 ↓
Azure resources
```

---

# 13. IaC benefits

Infrastructure as Code provides:

```text id="q3k5s7"
Repeatability
Version control
Peer review
Consistency
Auditability
Disaster recovery
```

---

# 14. Declarative vs imperative

### Imperative

You tell the system:

```text id="w5d8a2"
Create this
Then configure that
Then change this
```

### Declarative

You describe:

```text id="m2k7p4"
"I want the environment to look like this."
```

The deployment system determines what changes are required.

---

# 15. Desired state

Think:

```text id="8t3v1c"
Desired configuration
        ↓
Compare
        ↓
Actual configuration
        ↓
Apply difference
```

This is powerful for maintaining consistency.

---

# 16. Idempotency

One of the most important automation concepts.

An operation is idempotent when running it multiple times produces the same desired state.

Example:

```text id="9j2f5s"
Ensure setting = X
```

Running it:

```text id="d6r8w1"
1 time → X
2 times → X
10 times → X
```

That's safer than:

```text id="k1m4p7"
Increment setting by 1
```

which produces a different result every time.

---

# 17. Why idempotency matters

Automation can retry.

Suppose:

```text id="s4c9y2"
Automation
 ↓
Timeout
 ↓
Did operation succeed?
 ↓
Retry
```

If the operation is idempotent, retries are safer.

---

# 18. Automation authentication

Avoid putting credentials directly into scripts.

Bad:

```text id="x5v7q0"
$username = "admin"
$password = "..."
```

Better:

```text id="h2n8k3"
Managed identity
Service principal
Secret management
Federated identity
```

Use least privilege.

---

# 19. Managed identity

Managed identity allows Azure resources/workloads to authenticate to supported Azure services without embedding long-lived credentials in code.

Conceptually:

```text id="a6c3v8"
Automation
   ↓
Managed Identity
   ↓
Azure authorization
   ↓
Target resource
```

This reduces credential-management risk.

---

# 20. Least privilege for automation

An automation job that only needs to:

```text id="f1z5c7"
Read metrics
```

should not have:

```text id="k8m3r2"
Full administrator permissions
```

Give automation only the permissions it needs.

---

# 21. Secrets

If a secret is genuinely required:

```text id="q2v9s4"
Store securely
 ↓
Retrieve at runtime
 ↓
Never print it
 ↓
Rotate it
```

Never commit secrets into source control.

---

# 22. Health-check automation

Create a standard health check.

Example:

```text id="w7p3n1"
SQL MI reachable?
Databases accessible?
Storage healthy?
CPU healthy?
Connections normal?
Backup status healthy?
Blocking present?
Deadlocks increasing?
Critical alerts?
```

---

# 23. Automated health report

Output something like:

```text id="r3f7k2"
SQL MI Health Report

Availability: PASS
CPU:          PASS
Storage:      WARNING
Connections:  PASS
Blocking:     PASS
Backups:      PASS
Alerts:       WARNING
```

This is much easier to consume than raw logs.

---

# 24. Health status model

Use simple statuses:

```text id="g6n4x8"
PASS
WARNING
CRITICAL
UNKNOWN
```

**UNKNOWN** is important.

If the monitoring system can't retrieve data, don't automatically assume:

> "Everything is healthy."

---

# 25. Automated capacity monitoring

Suppose:

```text id="v8m2q5"
Storage > 80%
```

Automation can:

```text id="d3k7p1"
Detect
 ↓
Record
 ↓
Notify
 ↓
Create ticket
```

It doesn't necessarily need to modify the database automatically.

---

# 26. Automated performance reporting

Every morning:

```text id="m5r8t2"
Top CPU queries
Top duration queries
Top reads
Plan regressions
Blocking
Deadlocks
```

Send the report to the DBA team.

Now performance review becomes proactive.

---

# 27. Alert-driven automation

Architecture:

```text id="x9c4b6"
Metric / Log
     ↓
Alert rule
     ↓
Automation
     ↓
Action
```

Possible actions:

```text id="v2p8n5"
Notify
Create incident
Run diagnostic
Collect logs
Start remediation
```

---

# 28. Don't automatically fix everything

Suppose:

```text id="s8w4j1"
CPU > 90%
```

Don't create:

```text id="u6k3p2"
IF CPU > 90%
THEN randomly kill sessions
```

That could make the incident worse.

---

# 29. Safe automation pattern

Use:

```text id="z5x7c9"
Detect
 ↓
Gather evidence
 ↓
Classify
 ↓
Apply approved action
 ↓
Validate
 ↓
Escalate if unsuccessful
```

---

# 30. Guardrails

Every production automation should consider:

```text id="c4m9s2"
Scope
Permissions
Timeout
Retry limit
Change limit
Validation
Rollback/recovery
Escalation
```

---

# 31. Dry run

Before making changes, support a mode that reports:

```text id="r7h3k1"
"What would change?"
```

without actually changing anything.

This is extremely useful for:

```text id="b5n8q4"
Infrastructure
Configuration
Permissions
Maintenance
```

---

# 32. Logging automation

Every automation should record:

```text id="p3v6y9"
Timestamp
Actor
Target
Action
Input
Result
Duration
Error
Correlation ID
```

This gives you an audit trail.

---

# 33. Example automation log

```text id="x2k8m4"
2026-08-10 06:00
Job: SQLMI-HealthCheck
Target: MI-PROD-01
Result: WARNING
Reason: Storage = 87%
Action: Notification sent
```

Now another DBA can understand what happened.

---

# 34. Automation failure

Automation itself can fail.

Example:

```text id="f7c1m5"
Scheduler
 ↓
Script
 ↓
Authentication failure
 ↓
No health report
```

If nobody notices, you have **false confidence**.

Therefore:

> **Monitor the monitoring system.**

---

# 35. Automation heartbeat

A scheduled job can emit a heartbeat:

```text id="a8v3d2"
Health check started
Health check completed
```

If no heartbeat arrives for the expected interval:

```text id="j6n9q4"
Automation failure alert
```

---

# 36. Runbooks

A runbook describes how to handle a known operational situation.

Example:

```text id="m4s8p2"
High CPU Runbook

1. Confirm alert
2. Check duration
3. Identify top queries
4. Check Query Store
5. Check waits
6. Check blocking
7. Check recent deployments
8. Escalate/tune/scale
9. Validate
10. Document
```

---

# 37. Automated runbooks

A runbook can be partly automated:

```text id="w1k7x3"
Alert
 ↓
Collect diagnostics automatically
 ↓
Attach evidence to incident
 ↓
Human investigates
```

This is often safer than full autonomous remediation.

---

# 38. Human-in-the-loop

For high-risk changes:

```text id="q5d9v6"
Automation
 ↓
Recommend action
 ↓
Human approval
 ↓
Execute
 ↓
Validate
```

This is a strong production pattern.

---

# 39. CI/CD for database changes

Database changes can be integrated into deployment pipelines.

Conceptually:

```text id="z3m8c1"
Developer
 ↓
Source control
 ↓
Build
 ↓
Validation
 ↓
Test
 ↓
Approval
 ↓
Production deployment
```

---

# 40. Database deployment validation

Automated pipeline checks might include:

```text id="h7p2v5"
Syntax
Schema compatibility
Unit tests
Migration order
Permissions
Performance checks
Rollback/recovery plan
```

---

# 41. Migration safety

A deployment should know:

```text id="v9q4k6"
What changes?
How long?
What locks?
What dependencies?
Can old app version coexist?
```

This connects automation with the expand-and-contract approach from the previous lesson.

---

# 42. Version control

Store:

```text id="c6m1r8"
SQL scripts
Infrastructure definitions
Automation code
Runbooks
Configuration
```

in version control.

Then you know:

```text id="b8x2q5"
Who changed it?
What changed?
When?
Why?
```

---

# 43. Code review

Production automation should receive peer review.

Review:

```text id="u4n7k3"
Permissions
Destructive operations
Error handling
Retry logic
Secrets
Logging
Scope
```

---

# 44. Testing automation

Test automation at multiple levels:

```text id="s3v8d1"
Unit
 ↓
Integration
 ↓
Non-production
 ↓
Production-safe validation
```

Don't test a destructive script for the first time against production.

---

# 45. Failure injection

Mature teams test:

```text id="r8m2p6"
What if authentication fails?
What if Azure API times out?
What if SQL MI is unreachable?
What if the script runs twice?
What if validation fails?
```

This reveals hidden automation assumptions.

---

# 46. Retry logic

Retries should be:

```text id="j2k5v7"
Limited
Delayed
Context-aware
Logged
```

Avoid:

```text id="n8c4x1"
while failure:
    retry forever
```

---

# 47. Exponential backoff

Conceptually:

```text id="w6q3m9"
Retry 1 → wait 1 sec
Retry 2 → wait 2 sec
Retry 3 → wait 4 sec
Retry 4 → wait 8 sec
```

Add jitter where appropriate to avoid synchronized retry storms.

---

# 48. Timeout handling

Every remote operation should have a reasonable timeout.

Without one:

```text id="p7s2d4"
Automation
 ↓
Network issue
 ↓
Process waits indefinitely
 ↓
Next scheduled job starts
 ↓
Multiple stuck jobs
```

---

# 49. Concurrency control

Scheduled automation should avoid overlapping runs.

Example:

```text id="v4m8n1"
06:00 job starts
06:30 still running
07:00 new job starts
```

Now two copies operate simultaneously.

Use locking or execution controls where appropriate.

---

# 50. Automation architecture

A mature design:

```text id="k3p7s9"
             Scheduler/Event
                    |
                    ↓
               Automation
                    |
             Authentication
                    |
                    ↓
              SQL MI / Azure
                    |
          +---------+---------+
          |                   |
       Evidence            Action
          |                   |
          +---------+---------+
                    |
                 Validate
                    |
          +---------+---------+
          |                   |
        Success             Failure
          |                   |
       Record              Escalate
```

---

# 51. Example — automated storage warning

```text id="f8c2m5"
Storage = 87%
```

Automation:

```text id="d7n4q1"
1. Confirm measurement
2. Check trend
3. Check growth rate
4. Record evidence
5. Create warning
6. Notify DBA
```

No destructive action.

---

# 52. Example — automated health check

Every 15 minutes:

```text id="u2m6k8"
Check SQL MI
   ↓
Check connectivity
   ↓
Check critical databases
   ↓
Check CPU
   ↓
Check storage
   ↓
Check blocking
   ↓
Publish status
```

If everything is healthy:

```text id="e3v7n9"
PASS
```

If not:

```text id="q6r1p4"
WARNING/CRITICAL
```

---

# 53. Example — automated diagnostic collection

Alert:

```text id="b9k5s2"
High CPU
```

Automation automatically collects:

```text id="x7m3c6"
Timestamp
Top CPU queries
Query Store data
Wait information
Blocking information
Recent alerts
Recent deployment ID
```

Then attaches the evidence to the incident.

This can dramatically reduce triage time.

---

# 54. Example — safe remediation

Known condition:

```text id="n5v2q8"
Specific transient issue
```

Automation:

```text id="p8c4m7"
Detect
 ↓
Verify condition
 ↓
Execute approved remediation
 ↓
Wait
 ↓
Validate
```

If validation fails:

```text id="j1s6d3"
STOP
 ↓
Escalate
```

---

# 55. What NOT to automate blindly

Avoid automatic actions such as:

```text id="y4q8m2"
Kill random sessions
Delete data
Drop indexes
Change production permissions
Force failover
Scale repeatedly
Restart resources
```

unless the condition, authorization, safety, and validation are exceptionally well-defined.

---

# 56. Operational engineering mindset

A strong DBA asks:

> "How can we make this process reliable enough that it doesn't depend on one person's memory?"

That leads to:

```text id="t7k3p5"
Automation
Runbooks
Monitoring
Version control
Testing
Documentation
```

---

# 57. Automation checklist

```text id="r5c8m1"
DESIGN
☐ Clear purpose
☐ Defined scope
☐ Least privilege
☐ Idempotent where possible

SAFETY
☐ Dry run
☐ Validation
☐ Retry limits
☐ Timeouts
☐ Guardrails
☐ Escalation

SECURITY
☐ No embedded secrets
☐ Managed identity where appropriate
☐ Secret storage
☐ Audit trail

OPERATIONS
☐ Logging
☐ Monitoring
☐ Heartbeat
☐ Failure alert
☐ Runbook

DELIVERY
☐ Source control
☐ Peer review
☐ Testing
☐ Change management
```

---

# 58. Interview question

### "What DBA tasks would you automate?"

Strong answer:

> "I'd automate repetitive, predictable tasks such as health checks, monitoring, reporting, configuration validation, capacity checks, and deployment workflows. For remediation, I'd use automation cautiously with clear guardrails, validation, least privilege, logging, and escalation."

---

# 59. Interview question

### "What is idempotency?"

> "An idempotent operation can be executed multiple times without producing unintended cumulative changes. It's important because automation often retries after timeouts or transient failures."

---

# 60. Interview question

### "Would you give an automation account sysadmin?"

> "No, not by default. I'd apply least privilege and give the automation only the permissions required for its specific task. If elevated privileges are genuinely required, I'd constrain the scope, protect the identity, audit its use, and review the design."

---

# 61. Interview question

### "How would you make automation safe?"

Answer:

```text id="x8p4m6"
Least privilege
+
Idempotency
+
Timeouts
+
Limited retries
+
Dry-run capability
+
Validation
+
Logging
+
Monitoring
+
Human approval for high-risk changes
```

---

# 62. Final mental model

```text id="m6r2x8"
                  AUTOMATION
                      |
       +--------------+--------------+
       |              |              |
    Trigger        Execute        Observe
       |              |              |
 Scheduler        Safe action     Logging
 Alert            Least privilege Metrics
 Event            Idempotency      Heartbeat
       |              |              |
       +--------------+--------------+
                      |
                  Validate
                      |
              +-------+-------+
              |               |
           Success          Failure
              |               |
           Record          Escalate
```

---

# 63. Golden rule

> **Automate the process, not just the command.**

A production-quality automation isn't:

```text
Run command.
```

It's:

```text
Detect
→ Verify
→ Authorize
→ Execute
→ Validate
→ Record
→ Escalate if needed
```

That is the difference between **a script** and **operational engineering**.

---

## Next — Lesson 105: Azure SQL Managed Instance Disaster Recovery & Business Continuity — Advanced

We'll cover **RPO/RTO, HA vs DR, failover architecture, failover groups, geo-replication concepts, backups, point-in-time restore, regional disaster scenarios, DR testing, application recovery, DNS/connection considerations, runbooks, and a complete DR exercise.**
