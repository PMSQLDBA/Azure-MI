# Lesson 102 — Azure SQL Managed Instance Maintenance, Patching & Upgrades

This lesson is about keeping SQL Managed Instance **secure, supported, stable, and predictable** while minimizing business disruption.

The key principle:

> **Maintenance is successful when the system is changed safely, users remain within agreed service objectives, and the DBA can prove the change worked.**

---

# 1. Why maintenance matters

Production databases need ongoing:

```text
Security updates
Platform updates
Engine updates
Configuration changes
Performance improvements
Capacity changes
```

If you never maintain the environment, you accumulate:

```text
Security risk
Technical debt
Compatibility risk
Operational risk
```

---

# 2. Managed service ≠ zero responsibility

Azure manages substantial infrastructure and platform operations for SQL MI.

But the DBA still owns important areas such as:

```text
Application compatibility
Database configuration
Performance
Permissions
Workload behavior
Testing
Change management
Validation
```

Think:

```text
Azure manages platform operations
          +
You manage workload/data operations
```

---

# 3. Planned maintenance

Maintenance can involve platform changes that may affect service behavior.

A production DBA should understand:

```text
When?
What?
Expected impact?
How long?
What dependencies?
How will we validate?
```

---

# 4. Maintenance windows

Where supported, maintenance scheduling can help align planned platform maintenance with periods of lower business impact.

For example:

```text
Business peak:
09:00–18:00

Preferred maintenance:
02:00–04:00
```

The exact available scheduling capabilities depend on the SQL MI service and configuration.

---

# 5. Maintenance isn't only about downtime

Even if an event doesn't create a long outage, it can cause:

```text
Brief connectivity interruption
Failover
Connection resets
Transient errors
Performance changes
```

Applications must therefore be designed for transient infrastructure events.

---

# 6. Application retry behavior

During a maintenance-related interruption:

```text
Connection
   ↓
Transient failure
   ↓
Retry
   ↓
Reconnect
```

A well-designed application can recover automatically.

A poorly designed application may expose:

```text
"Database connection failed"
```

to every user.

---

# 7. Retry design

Good retry behavior generally includes:

```text
Exponential backoff
Jitter
Limited retries
Appropriate timeout
Transient-error detection
```

Avoid:

```text
Immediate infinite retry
```

because that can produce a connection storm.

---

# 8. Connection pooling

Connection pooling can improve application performance by reusing connections.

But after failover/maintenance, pools may contain stale connections.

Applications should handle connection failures correctly and establish fresh connections when needed.

---

# 9. Maintenance communication

A production maintenance process should communicate:

```text
Maintenance event
Start/end window
Expected impact
Affected systems
Owner
Escalation path
Validation plan
```

Don't rely solely on:

> "Azure is doing maintenance."

Your application and business teams still need to know what to expect.

---

# 10. Change management

Use a controlled process:

```text
Request
 ↓
Assess
 ↓
Test
 ↓
Approve
 ↓
Schedule
 ↓
Execute
 ↓
Validate
 ↓
Document
```

---

# 11. Risk assessment

Before a significant change, ask:

```text
What can break?
Who is affected?
Can we detect failure quickly?
Can we recover?
What is the rollback strategy?
```

---

# 12. Production vs non-production

Never make production the first environment where you discover compatibility problems.

Use:

```text
Development
   ↓
Test
   ↓
Staging
   ↓
Production
```

The exact number of environments depends on organizational needs.

---

# 13. Upgrade testing

Suppose a platform/engine change is planned.

Test:

```text
Application connectivity
Authentication
Stored procedures
Queries
Jobs
ETL
Reports
Integrations
Performance
Security
```

---

# 14. Compatibility testing

A database can technically upgrade successfully while an application still breaks.

For example:

```text
Database:
Healthy

Application:
Stored procedure behavior changed
```

Therefore:

> **Successful platform maintenance does not automatically mean successful application maintenance.**

---

# 15. Query regression

After an upgrade or configuration change, check:

```text
Query duration
CPU
I/O
Waits
Execution plans
Application latency
```

Query Store can be especially useful for comparing workload behavior over time.

---

# 16. Baseline before maintenance

Before a major change, record:

```text id="j7c4fd"
CPU
Memory
I/O
Latency
Top queries
Waits
Blocking
Connections
Error rate
```

This becomes your baseline.

---

# 17. Why baselines matter

After maintenance:

```text
Before:
Average query = 300 ms

After:
Average query = 900 ms
```

You now have evidence of a regression.

Without a baseline, someone might say:

> "It seems fine."

---

# 18. Maintenance validation

Validation should occur at multiple levels.

### Infrastructure

```text
SQL MI healthy
```

### Database

```text
Database accessible
Queries execute
```

### Application

```text
Application connects
```

### Business

```text
Critical transaction succeeds
```

---

# 19. Four-level validation

Use:

```text id="7snq2k"
Platform
   ↓
Database
   ↓
Application
   ↓
Business
```

Example:

```text
SQL MI = healthy
SQL query = works
Application = works
Order processing = works
```

That's much stronger than simply checking the Azure portal.

---

# 20. Maintenance runbook

A useful runbook:

```text
1. Confirm maintenance window
2. Notify stakeholders
3. Verify backups/recovery posture
4. Capture baseline
5. Verify monitoring
6. Confirm application retry behavior
7. Execute/allow maintenance
8. Monitor during event
9. Validate database
10. Validate application
11. Validate business transaction
12. Compare metrics
13. Close incident/change
```

---

# 21. Backup verification

Before significant changes, verify your recovery capability.

Ask:

```text
Are backups healthy?
Can required recovery points be reached?
Is retention appropriate?
Has restoration been tested?
```

A backup that has never been restored is an assumption, not proof.

---

# 22. Rollback

Not every Azure-managed platform operation can simply be "rolled back."

Therefore distinguish:

```text
Rollback
```

from:

```text
Recovery / remediation
```

For some platform changes, your practical recovery plan may involve:

```text
Restore
Failover
Configuration reversal
Application rollback
Data recovery
```

---

# 23. Application rollback

Suppose a deployment is the actual source of a regression.

Database platform:

```text
Healthy
```

Application:

```text
Broken
```

The right rollback may be:

```text
Application version N
   ↓
Revert to version N-1
```

not a database rollback.

---

# 24. Schema changes

Schema changes deserve special care.

Example:

```text
ALTER TABLE
ADD COLUMN
DROP COLUMN
ALTER COLUMN
CREATE INDEX
DROP INDEX
```

Before deployment, consider:

```text
Locking
Duration
Storage
Query plans
Application compatibility
Rollback/recovery
```

---

# 25. Backward compatibility

A useful deployment strategy:

```text
Old application
      ↓
New database schema
      ↓
New application
```

The new schema should ideally remain compatible with the old application during the transition.

This is often called an **expand-and-contract** style approach.

---

# 26. Expand-and-contract

### Phase 1 — Expand

Add new structures without breaking the old application.

```text
Old schema + new schema
```

### Phase 2 — Migrate

Move application behavior/data.

### Phase 3 — Contract

Remove old structures only after they're no longer needed.

```text
New schema only
```

This reduces deployment coupling.

---

# 27. Example

Old:

```text
Customer.Name
```

New design:

```text
Customer.FirstName
Customer.LastName
```

Don't immediately delete `Name`.

Instead:

```text
1. Add FirstName/LastName
2. Populate them
3. Update application
4. Validate
5. Stop using Name
6. Remove Name later
```

This is safer for rolling deployments.

---

# 28. Index maintenance

Indexes also require careful change management.

Before creating an index:

```text
Why?
Which query?
What benefit?
What storage cost?
What write overhead?
```

Don't create indexes simply because:

> "Indexes are good."

---

# 29. Statistics

Query performance can change when:

```text
Data distribution changes
Statistics change
Execution plans change
```

After significant workload changes, investigate query behavior rather than assuming the database engine will behave identically.

---

# 30. Maintenance and performance

After maintenance, compare:

```text
CPU
Duration
I/O
Waits
Plans
Blocking
Deadlocks
```

A maintenance event should have an explicit validation period.

---

# 31. Maintenance monitoring

During the event, monitor:

```text
Availability
Connections
Error rate
Latency
CPU
I/O
Application errors
```

You don't want to discover a problem 30 minutes later from a customer complaint.

---

# 32. Maintenance incident example

Suppose:

```text
02:00 — Maintenance begins
02:05 — Brief failover
02:06 — Connections recover
02:10 — Application healthy
```

That's likely a successful event if service objectives were met.

But suppose:

```text
02:05 — Failover
02:06 — Application connection failures
02:20 — Still failing
```

Now you have an operational incident.

---

# 33. Troubleshooting maintenance-related connectivity

Check:

```text
Application connection pool
DNS
Authentication
Network path
TLS
Endpoint
Transient errors
```

Don't immediately assume:

> "SQL MI is down."

---

# 34. Upgrade compatibility matrix

For major changes, maintain something like:

```text
Component          Tested?
--------------------------------
Application API       ✓
ORM                   ✓
Drivers               ✓
Stored procedures     ✓
ETL                   ✓
Reporting             ✓
Authentication        ✓
Monitoring            ✓
Backup/recovery       ✓
```

This makes readiness measurable.

---

# 35. Drivers matter

Database engine compatibility isn't the only concern.

Applications also depend on:

```text
ODBC
JDBC
.NET drivers
ORMs
Connection libraries
```

An old client driver can become an unexpected failure point.

---

# 36. TLS and encryption changes

Security-related maintenance may affect:

```text
TLS versions
Certificates
Authentication
Encryption requirements
Client compatibility
```

Test both:

```text
Application → SQL MI
```

and:

```text
Administrative tools → SQL MI
```

---

# 37. Authentication testing

After a security or platform change, verify:

```text
Microsoft Entra users
Managed identities
Service principals
SQL logins where applicable
```

An application may fail even though the database itself is healthy.

---

# 38. Maintenance and DR

Maintenance should not accidentally invalidate your DR strategy.

Verify:

```text
Replication
Secondary health
Backups
Failover readiness
Application DR
```

---

# 39. Maintenance + DR drill

A mature organization may test:

```text
Maintenance
   ↓
Failover
   ↓
Application recovery
```

This demonstrates that the environment remains recoverable through operational changes.

---

# 40. Emergency maintenance

Sometimes you cannot wait for a perfect change window.

For example:

```text
Critical security issue
Critical reliability problem
Severe performance issue
```

Then use:

```text
Emergency change process
```

Still maintain:

```text
Approval
Risk assessment
Communication
Monitoring
Validation
Documentation
```

Emergency does not mean uncontrolled.

---

# 41. Maintenance checklist

```text
BEFORE
☐ Understand change
☐ Confirm maintenance window
☐ Notify stakeholders
☐ Verify recovery capability
☐ Capture baseline
☐ Confirm monitoring
☐ Confirm application retry behavior
☐ Test dependencies

DURING
☐ Monitor availability
☐ Monitor connections
☐ Monitor errors
☐ Monitor application
☐ Record timeline

AFTER
☐ Validate SQL MI
☐ Validate database
☐ Validate application
☐ Validate business transaction
☐ Compare performance
☐ Check Query Store
☐ Check alerts
☐ Document results
```

---

# 42. Interview question

### "Does Azure-managed SQL mean I don't need a maintenance plan?"

Strong answer:

> "No. Azure manages substantial platform operations, but the DBA still needs to manage workload compatibility, application behavior, configuration, performance, security, testing, monitoring, and change validation."

---

# 43. Interview question

### "How would you prepare for a major database platform change?"

Answer:

```text
1. Understand the change.
2. Identify affected dependencies.
3. Test in non-production.
4. Capture production baseline.
5. Verify recovery capabilities.
6. Confirm application compatibility.
7. Schedule the change.
8. Monitor execution.
9. Validate platform/database/application/business behavior.
10. Compare post-change performance.
```

---

# 44. Interview question

### "How do you know maintenance succeeded?"

Don't say:

> "The Azure portal says healthy."

Instead:

```text
SQL MI healthy
+
Database accessible
+
Queries healthy
+
Application connected
+
Critical transactions successful
+
Performance within baseline
+
No unexpected errors
```

---

# 45. Interview question

### "What is your rollback plan?"

Strong answer:

> "First I determine whether the change supports an actual rollback. For managed platform changes, rollback may not be a simple reversal, so I define a recovery/remediation strategy using the available mechanisms such as configuration reversal, application rollback, failover, or database recovery where appropriate."

---

# 46. Production maintenance architecture

```text
             CHANGE REQUEST
                    |
                    ↓
              RISK ASSESSMENT
                    |
                    ↓
                  TEST
                    |
                    ↓
                 APPROVE
                    |
                    ↓
                SCHEDULE
                    |
                    ↓
                MAINTENANCE
                    |
          +---------+---------+
          |                   |
      MONITORING          RUNBOOK
          |                   |
          +---------+---------+
                    |
                    ↓
                VALIDATION
                    |
        +-----------+-----------+
        |           |           |
     Platform    Database    Business
        |           |           |
        +-----------+-----------+
                    |
                    ↓
              DOCUMENT RESULT
```

---

# 47. Golden rule

> **Never define success as "the change completed."**

Define success as:

> **"The change completed and the service still meets its technical and business objectives."**

---

# 48. Final mental model

```text
                 MAINTENANCE
                      |
       +--------------+--------------+
       |              |              |
     Security       Platform      Application
       |              |              |
       +--------------+--------------+
                      |
                   Testing
                      |
                   Baseline
                      |
                  Change
                      |
                 Monitoring
                      |
                 Validation
                      |
               Business outcome
```

That is the production DBA mindset: **control the change, understand the risk, observe the system, and prove the outcome.**

---

## Next — Lesson 103: Azure SQL Managed Instance Performance Tuning — Advanced DBA

We'll go deeper into **execution plans, Query Store, indexing strategy, statistics, parameter sniffing, cardinality estimation, SARGability, memory grants, tempdb, waits, blocking, deadlocks, plan regression, and a full end-to-end performance troubleshooting case.**
