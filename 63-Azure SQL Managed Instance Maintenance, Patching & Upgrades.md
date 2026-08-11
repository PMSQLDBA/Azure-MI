# Lesson 95 — Azure SQL Managed Instance Maintenance, Patching & Upgrades

This lesson is about keeping a production SQL MI **healthy, supported, secure, and current** without creating unnecessary downtime or application problems.

The key idea:

> **Patching is not just a database task. It is a change-management process involving SQL MI, Azure, networking, applications, monitoring, and business stakeholders.**

---

# 1. Why maintenance matters

Production systems need regular maintenance because of:

```text
Security fixes
Bug fixes
Platform improvements
Performance improvements
Reliability improvements
New features
End-of-support requirements
```

Ignoring maintenance creates technical debt.

---

# 2. Azure SQL MI is a managed service

With SQL Server on a traditional VM, you might manage:

```text
Windows patching
SQL Server patching
OS configuration
Storage
Hardware
Cluster infrastructure
```

With Azure SQL Managed Instance, Microsoft manages much of the underlying platform.

Your responsibility shifts toward:

```text
Database configuration
Security
Application compatibility
Workload validation
Change management
Monitoring
```

---

# 3. Shared responsibility

Think:

```text id="x8s2kd"
Microsoft
   |
   +-- Azure infrastructure
   +-- Managed platform
   +-- Underlying service maintenance
   |
   ↓
You
   |
   +-- Databases
   +-- Permissions
   +-- Application behavior
   +-- Configuration
   +-- Change validation
```

Managed service does **not** mean zero DBA responsibility.

---

# 4. Maintenance categories

Think about maintenance in several groups:

```text
Platform maintenance
Security patching
SQL engine updates
Configuration changes
Application changes
Schema changes
Performance maintenance
DR maintenance
```

Each has different risks.

---

# 5. Planned maintenance

Planned maintenance is controlled.

Typical flow:

```text id="2n7f8k"
Notification
   ↓
Impact assessment
   ↓
Testing
   ↓
Change window
   ↓
Maintenance
   ↓
Validation
```

The objective is predictable operations.

---

# 6. Maintenance window

A maintenance window is a period selected for planned maintenance activities.

For production systems, choose a period with:

```text id="k3q9v5"
Low business traffic
Low batch activity
Low transaction volume
Available support staff
Monitoring coverage
```

Don't choose a window just because:

> "Nobody is in the office."

Some overnight jobs may be running.

---

# 7. Maintenance dependencies

Before a maintenance event, identify:

```text id="c5m8r2"
Applications
ETL
Reporting
SQL Agent jobs
Integration services
External APIs
Backup processes
Monitoring
Security tools
```

Maintenance can affect more than the database engine.

---

# 8. Notifications

Production DBAs should monitor Azure maintenance notifications.

The goal is to know:

```text id="v4x7p1"
What is changing?
When?
Which resource?
Expected impact?
Required action?
```

Don't wait until the maintenance starts.

---

# 9. Patch testing

Never assume:

> "It's only a patch, so testing isn't necessary."

For critical workloads:

```text id="m7q2z9"
Development
    ↓
Test
    ↓
Staging
    ↓
Production
```

Validate application behavior before production changes where practical.

---

# 10. What should you test?

At minimum:

```text id="a5n8c3"
Application connectivity
Authentication
Critical queries
Stored procedures
ETL
Reports
SQL Agent jobs
Transactions
Performance
Monitoring
```

---

# 11. Compatibility testing

A database engine update can expose application assumptions.

For example:

```text id="p3k6v8"
Application
    ↓
Old SQL behavior
    ↓
Engine update
    ↓
Different behavior
```

The application may technically connect but behave differently.

---

# 12. Compatibility level

SQL Server databases have a **database compatibility level**.

This can influence certain database-engine behaviors.

Conceptually:

```text id="d8m2r4"
SQL engine version
        +
Compatibility level
        ↓
Database behavior
```

Don't casually change compatibility level in production.

---

# 13. Engine version vs compatibility level

These are different concepts.

### Engine version

The SQL Server engine/platform version running the instance.

### Compatibility level

A database-level setting controlling compatibility behavior.

So:

```text id="y6p9t2"
Engine version ≠ compatibility level
```

This distinction is frequently tested in interviews.

---

# 14. Example

Suppose:

```text id="n5c7x1"
SQL MI engine updated
```

but:

```text id="j4m8q6"
Database compatibility level remains unchanged
```

You can potentially separate:

```text id="r2v5k9"
Platform upgrade
```

from:

```text id="s8p3d7"
Application behavior changes
```

That can make upgrades safer.

---

# 15. Query behavior

After an engine update, monitor important queries:

```text id="q7n2m4"
Execution plans
CPU
Duration
Logical reads
Wait statistics
Blocking
```

A query that was fast before maintenance can sometimes behave differently afterward.

---

# 16. Query Store

Query Store is extremely useful for upgrade validation.

Before maintenance:

```text id="m3v7x9"
Capture baseline
```

After maintenance:

```text id="w5k1c8"
Compare performance
```

Look for:

```text id="r6p2j4"
Regressed queries
Plan changes
CPU increases
Duration increases
```

---

# 17. Establish a baseline

Before a major change, capture:

```text id="k9c4n2"
Average CPU
Peak CPU
DTU/vCore utilization as applicable
Query duration
Throughput
Waits
Blocking
Failed connections
```

Then compare after maintenance.

Without a baseline, you can't confidently say:

> "Performance changed."

---

# 18. Maintenance validation

After maintenance:

```text id="v3m8x7"
1. SQL MI reachable?
2. Databases online?
3. Applications connecting?
4. Authentication working?
5. Jobs running?
6. Queries performing normally?
7. Monitoring healthy?
8. Replication/DR healthy?
9. Backups healthy?
```

---

# 19. Application smoke testing

Do not stop at:

```text id="f6k2p1"
SELECT @@VERSION;
```

That's not enough.

Run actual business operations:

```text id="j8q4w3"
Login
Create transaction
Read record
Update record
Run critical report
Process batch
```

---

# 20. Maintenance incident

Imagine maintenance completes.

SQL MI reports healthy.

But the application reports:

```text id="z5m7c2"
Login failures
```

Possible causes:

```text id="n4p8x1"
Authentication configuration
Network connectivity
Certificate/TLS behavior
Application driver
Connection settings
Identity permissions
```

This is why end-to-end validation matters.

---

# 21. Upgrade planning

For a major engine upgrade:

```text id="b6r3v9"
Inventory
   ↓
Compatibility review
   ↓
Test
   ↓
Performance baseline
   ↓
Change plan
   ↓
Production upgrade
   ↓
Validation
   ↓
Monitoring
```

---

# 22. Inventory before upgrade

Know what exists:

```text id="q3x8m6"
Databases
Logins
Users
Roles
Linked servers
Jobs
Credentials
External dependencies
Applications
Connection drivers
```

A surprising number of upgrade failures come from undocumented dependencies.

---

# 23. Linked servers

If your environment uses linked servers, test them.

Example:

```text id="c7p2n5"
SQL MI
   ↓
Linked Server
   ↓
External SQL Server
```

After an upgrade, authentication, provider, or connectivity behavior may need validation.

---

# 24. SQL Agent jobs

SQL Agent jobs are operational dependencies.

Review:

```text id="m8v4r1"
Schedules
Owners
Credentials
Proxies
PowerShell steps
SSIS steps
External commands
Notifications
```

After maintenance:

```text id="w2k7q9"
Confirm critical jobs execute successfully.
```

---

# 25. Application drivers

A database engine can be upgraded while an old client driver remains.

Potential architecture:

```text id="x5j3p8"
Application
   ↓
Old driver
   ↓
SQL MI
```

Compatibility issues may arise.

Test the actual application stack, not just the database.

---

# 26. TLS and certificates

Secure connections can involve:

```text id="y9c4k2"
TLS
Certificates
Drivers
Trust stores
Encryption settings
```

If an application suddenly cannot connect after a maintenance event, don't assume the database itself is down.

---

# 27. Rollback planning

Every significant change needs a recovery strategy.

Ask:

> **If validation fails, how do we recover?**

Possible approaches depend on the change:

```text id="p5n8v3"
Restore
Failover to DR
Revert application
Reverse migration
Correct configuration
```

Not every Azure-managed platform change can simply be "rolled back" like a local software installation.

---

# 28. Important distinction

### Rollback

Return to the previous state.

### Recovery

Restore service using another mechanism.

These aren't always the same.

---

# 29. Example

Suppose a platform update causes application problems.

You might not have:

```text id="e6q2k8"
"Uninstall patch"
```

Instead, your recovery strategy could be:

```text id="r3m7v1"
Fail over to DR
Restore application compatibility
Investigate
```

This is why DR is part of change management.

---

# 30. Maintenance + DR

Before major production changes:

```text id="c8p4m6"
Primary
   ↓
Healthy DR
```

Confirm:

```text id="n7x2q9"
Replication healthy
DR accessible
Runbook available
Application recovery path tested
```

A maintenance event is a poor time to discover that DR is broken.

---

# 31. Change management

A mature DBA doesn't simply say:

> "I'll update production at midnight."

Instead:

```text id="h4k9p2"
Change request
   ↓
Risk assessment
   ↓
Impact assessment
   ↓
Testing evidence
   ↓
Approval
   ↓
Maintenance
   ↓
Validation
   ↓
Closure
```

---

# 32. Risk assessment

Ask:

```text id="z3m8v5"
What can break?
How many users are affected?
What dependencies exist?
What is the recovery plan?
What is the expected downtime?
What monitoring is available?
```

---

# 33. High-risk changes

Examples:

```text id="q6n2x8"
Major version change
Compatibility-level change
Large schema change
Authentication change
Networking change
Storage/configuration change
```

Treat these more carefully than routine maintenance.

---

# 34. Low-risk doesn't mean zero-risk

Even a routine platform maintenance event can expose:

```text id="j5c7r3"
Connection problems
Application bugs
Driver incompatibility
Unexpected performance changes
```

So maintain appropriate validation.

---

# 35. Performance regression after upgrade

Suppose before maintenance:

```text id="w8m4k1"
Average query duration = 100 ms
```

After:

```text id="p2v7c9"
Average = 400 ms
```

Don't immediately scale the MI.

First investigate:

```text id="d6x3m8"
Query Store
Execution plans
Wait stats
CPU
IO
Blocking
Statistics
```

---

# 36. Query regression workflow

```text id="s4k8n2"
Performance degraded
       ↓
Identify affected queries
       ↓
Compare old/new plans
       ↓
Check waits
       ↓
Check statistics/indexes
       ↓
Determine root cause
       ↓
Apply targeted fix
```

---

# 37. Don't use "restart" as the diagnosis

A common weak DBA response:

> "Let's restart SQL."

That may temporarily change symptoms without fixing the cause.

Production DBA thinking is:

```text id="v7m2q5"
Symptom
 ↓
Evidence
 ↓
Root cause
 ↓
Controlled remediation
```

---

# 38. Maintenance communication

A good maintenance notice should tell stakeholders:

```text id="x8c3n7"
What is changing?
When?
Expected impact?
What users should expect?
Who owns the change?
How will success be validated?
What happens if validation fails?
```

---

# 39. Example maintenance timeline

```text id="k2q6m8"
T-7 days
Change approved

T-3 days
Testing complete

T-1 day
Final health check

T-30 min
Notify stakeholders

T-0
Maintenance

T+10 min
Connectivity validation

T+20 min
Application smoke test

T+30 min
Performance validation

T+60 min
Change closure
```

---

# 40. Maintenance checklist

```text id="m9x4p2"
BEFORE
☐ Change approved
☐ Dependencies identified
☐ Backup/recovery verified
☐ DR health verified
☐ Monitoring active
☐ Baseline captured
☐ Application team informed
☐ Runbook ready

DURING
☐ Monitor connectivity
☐ Monitor errors
☐ Monitor application
☐ Record timestamps

AFTER
☐ SQL MI healthy
☐ Databases healthy
☐ Authentication works
☐ Applications work
☐ Jobs work
☐ Performance normal
☐ DR healthy
☐ Monitoring normal
☐ Change documented
```

---

# 41. Interview question

### "Who is responsible for patching SQL MI?"

Strong answer:

> "Azure SQL Managed Instance is a managed service, so Microsoft manages the underlying platform maintenance and much of the patching infrastructure. The DBA remains responsible for understanding maintenance impact, configuring appropriate settings where available, validating application compatibility, monitoring the workload, and managing the change process."

---

# 42. Interview question

### "What should you do before a production upgrade?"

Answer:

```text id="r6v2n8"
1. Inventory dependencies.
2. Establish performance baseline.
3. Test in non-production.
4. Review compatibility.
5. Validate applications and drivers.
6. Confirm backup/DR readiness.
7. Prepare rollback/recovery plan.
8. Schedule maintenance window.
9. Notify stakeholders.
10. Prepare post-upgrade validation.
```

---

# 43. Interview question

### "What would you check after an upgrade?"

Answer:

```text id="j3m8q5"
SQL MI health
Databases
Connectivity
Authentication
Applications
Jobs
Queries
Performance
Query Store
Blocking
Errors
Backups
DR
Monitoring
```

---

# 44. Interview question

### "Why use Query Store before and after upgrades?"

> "Query Store provides historical query-performance information, allowing us to compare query behavior and identify performance regressions or plan changes following an upgrade."

---

# 45. Interview question

### "What's the difference between engine version and compatibility level?"

> "The engine version describes the SQL engine/platform version running the instance. Compatibility level is a database-level setting that controls compatibility behavior for certain database-engine features. They should be treated as separate concepts."

---

# 46. Production scenario

Your manager says:

> "SQL MI maintenance finished. Users say the application is slow."

What do you do?

### Bad response:

```text id="y6n2p8"
Scale up immediately.
```

### Better response:

```text id="v4c7m3"
1. Confirm timing.
2. Compare against baseline.
3. Check Query Store.
4. Identify regressed queries.
5. Compare execution plans.
6. Check CPU/IO/waits.
7. Check blocking.
8. Determine root cause.
9. Apply targeted remediation.
10. Continue monitoring.
```

---

# 47. Production scenario — application won't connect

After maintenance:

```text id="s8k3x5"
SQL MI = healthy
Application = connection failures
```

Investigate:

```text id="n2m7q9"
DNS
Network
TLS
Certificate
Driver
Authentication
Login/user permissions
Connection string
```

Don't immediately conclude:

> "SQL is broken."

---

# 48. Production scenario — SQL Agent jobs fail

Check:

```text id="c6p4r8"
Job history
Owner
Credentials
Proxy
Schedule
Permissions
External dependencies
```

Then determine whether the failure is:

```text id="w3x7m1"
Platform-related
Configuration-related
Application-related
```

---

# 49. Golden rule for upgrades

> **Never measure upgrade success solely by whether SQL MI is online.**

True success is:

```text id="q5k8n2"
SQL MI healthy
+
Applications healthy
+
Performance acceptable
+
Security intact
+
Backups healthy
+
DR healthy
```

---

# 50. Final mental model

```text id="m7c2v9"
             PRODUCTION MAINTENANCE
                     |
          +----------+----------+
          |                     |
       BEFORE                 AFTER
          |                     |
       Baseline              Validate
       Testing               Connectivity
       Dependencies          Applications
       DR check              Performance
       Runbook               Jobs
       Approval              Backup
                             DR
                              |
                              ↓
                         Monitor
```

And remember:

> **A managed database reduces infrastructure work; it does not eliminate operational responsibility.**

---

## Next — Lesson 96: Azure SQL Managed Instance Monitoring & Alerting

We'll build a **real DBA monitoring framework** covering **CPU, memory, storage, I/O, waits, blocking, deadlocks, Query Store, Azure Monitor metrics, Log Analytics, alerts, dashboards, actionable thresholds, incident triage, and a production monitoring checklist.**
