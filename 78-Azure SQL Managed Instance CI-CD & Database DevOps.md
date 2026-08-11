# Lesson 110 — Azure SQL Managed Instance CI/CD & Database DevOps

This lesson takes automation one step further:

> **How do you safely change a production database continuously without turning every release into a potential outage?**

For a senior Azure SQL DBA, this is one of the most important skills because databases are **stateful systems**. Application code can often be replaced quickly; database changes can alter persistent data and structure.

---

# 1. What is Database DevOps?

Database DevOps applies software-engineering practices to database changes:

```text
Version control
      ↓
Code review
      ↓
Automated testing
      ↓
Deployment pipeline
      ↓
Production
      ↓
Monitoring
      ↓
Feedback
```

The objective is:

```text
Fast changes
+
Safe changes
+
Repeatable changes
```

---

# 2. Why databases are different

Application deployment:

```text
Old binary
   ↓
New binary
```

Database deployment:

```text
Old schema
   ↓
Data already exists
   ↓
New schema
   ↓
Existing applications
   ↓
Existing queries
   ↓
Existing dependencies
```

You cannot treat database deployment exactly like application deployment.

---

# 3. Database change categories

Most changes fall into:

```text
Schema
Data
Programmability
Security
Performance
Configuration
```

Examples:

```text
CREATE TABLE
ALTER TABLE
Data migration
Stored procedure change
Role/permission change
Index change
Configuration change
```

---

# 4. Schema changes

Examples:

```text id="f1p4m7"
CREATE TABLE
ALTER TABLE
CREATE INDEX
ALTER COLUMN
DROP COLUMN
```

Some are easy to deploy.

Others can be expensive or disruptive.

---

# 5. Easy vs dangerous changes

Potentially low-risk:

```text id="x7k2m5"
Add nullable column
Create compatible object
Add stored procedure
```

Potentially high-risk:

```text id="q9v3c8"
Drop column
Change data type
Large table rewrite
Massive data migration
Blocking index operation
```

Risk depends on workload, size, dependencies, and implementation.

---

# 6. Backward compatibility

A central DevOps principle:

> **New application and old application should coexist safely during deployment transitions.**

For example:

```text
Application v1
      |
      v
Old schema
```

You introduce:

```text
New schema
```

but don't immediately remove what v1 needs.

---

# 7. Expand-and-contract

This is one of the most useful patterns.

```text
EXPAND
 ↓
DEPLOY
 ↓
MIGRATE
 ↓
SWITCH
 ↓
CONTRACT
```

---

# 8. Example — Renaming a column

Suppose:

```text id="m8c2v6"
CustomerName
```

must eventually become:

```text id="p5x9k3"
DisplayName
```

Don't immediately:

```text id="q7n4m1"
DROP CustomerName
```

because old application versions may still reference it.

---

# 9. Safe migration

### Phase 1 — Expand

Add:

```text id="v2m6r8"
DisplayName
```

Keep:

```text id="j4p9x1"
CustomerName
```

---

# 10. Phase 2 — Application compatibility

Deploy application code capable of using the new field while the old field still exists.

Potentially:

```text id="r6k3m8"
Write both
Read new
```

depending on the application's consistency requirements.

---

# 11. Phase 3 — Data migration

Backfill:

```text id="x8q2p5"
CustomerName
      ↓
DisplayName
```

For large tables, avoid assuming a single massive transaction is safe.

---

# 12. Phase 4 — Switch

Once validated:

```text id="m3v7n9"
Application
 ↓
DisplayName
```

---

# 13. Phase 5 — Contract

After all consumers no longer need the old field:

```text id="k5x8c2"
Remove CustomerName
```

This may happen in a later release.

---

# 14. Why this works

At no point do you require:

```text id="q2m9v4"
Old app
AND
new schema only
```

Instead you maintain compatibility during transition.

---

# 15. Database migration styles

Two major approaches are common.

### State-based

Describe what the final schema should look like.

Example:

```text id="v6p3m8"
Target schema
```

The deployment system determines differences.

### Migration-based

Describe incremental changes:

```text id="x4n8k2"
Migration 001
Migration 002
Migration 003
```

Both have legitimate use cases.

---

# 16. DACPAC-style deployment

A DACPAC represents a database schema model.

Conceptually:

```text id="m9r2x7"
Desired schema
      ↓
Compare
      ↓
Existing database
      ↓
Deployment changes
```

This is useful for schema-centric database projects.

---

# 17. Migration-based deployment

Example:

```text id="c7m4p1"
001_CreateCustomer.sql
002_AddDisplayName.sql
003_CreateIndex.sql
004_CreateProcedure.sql
```

The system tracks which migrations have already executed.

---

# 18. DACPAC vs migrations

Think:

| Approach           | Strength                                  |
| ------------------ | ----------------------------------------- |
| DACPAC/state-based | Schema model and desired-state deployment |
| Migrations         | Explicit ordered database changes         |

Neither is universally better.

Choose based on:

```text
Team workflow
Database complexity
Application architecture
Deployment tooling
Migration requirements
```

---

# 19. Database project

A repository might contain:

```text id="n3x8m5"
Database/
 ├── Tables/
 ├── Views/
 ├── Procedures/
 ├── Functions/
 ├── Security/
 ├── Scripts/
 └── Tests/
```

The exact structure can vary by tooling.

---

# 20. Version control

Never keep production database changes only in:

```text id="r4p8k2"
Someone's laptop
```

Instead:

```text id="m6x2v9"
Git
 ↓
Review
 ↓
Pipeline
```

Every change becomes traceable.

---

# 21. Pull request review

Reviewers should check:

```text id="q8c3m7"
Correctness
Performance
Security
Backward compatibility
Data impact
Locking
Deployment duration
Rollback/recovery
```

---

# 22. Migration review example

Someone proposes:

```text id="v5m9x1"
ALTER TABLE Orders
ALTER COLUMN CustomerId ...
```

The reviewer should ask:

```text id="p7n2k4"
How large is Orders?
What indexes depend on this?
What locks occur?
How long could it run?
Will application traffic continue?
Can it be done incrementally?
```

---

# 23. Automated validation

Before deployment:

```text id="c2x8m5"
Syntax
 ↓
Schema validation
 ↓
Security checks
 ↓
Unit tests
 ↓
Integration tests
 ↓
Performance checks
```

---

# 24. Database unit tests

Test database logic such as:

```text id="n6p3r9"
Stored procedures
Functions
Constraints
Business rules
```

Example:

```text id="m8v4k2"
Given customer exists
When procedure runs
Then expected order is created
```

---

# 25. Integration testing

Integration testing validates:

```text id="x3q7m5"
Application
+
Database
```

This catches issues that isolated SQL tests may miss.

---

# 26. Compatibility testing

Check:

```text id="r9m2c6"
Old application + new schema
New application + new schema
```

During rolling deployments, both may temporarily exist.

---

# 27. Performance testing

A migration can be functionally correct but operationally dangerous.

Example:

```text id="v7p4n8"
CREATE INDEX
```

may improve one query while increasing:

```text id="k3x9m2"
Write cost
Storage
Maintenance
```

Test the whole workload.

---

# 28. Large data migrations

Avoid:

```text id="q5m8c3"
UPDATE 500 million rows
```

as one enormous transaction without carefully understanding its consequences.

Potential issues include:

```text id="n2r7x4"
Lock duration
Transaction log growth
Blocking
I/O
Recovery time
```

---

# 29. Batched migration

Instead:

```text id="m6p3v8"
Batch 1
 ↓
Commit
 ↓
Batch 2
 ↓
Commit
 ↓
Batch 3
```

This can reduce transaction size and improve operational control.

Exact batch sizing should be tested against the workload.

---

# 30. Migration checkpointing

A migration should ideally know:

```text id="x8c4m1"
What has completed?
What remains?
```

This makes recovery from interruption easier.

---

# 31. Idempotent migrations

A robust migration should avoid accidentally doing the same destructive operation twice.

Conceptually:

```text id="r3n7p5"
If required state doesn't exist
    apply change
else
    skip/validate
```

But don't make every migration blindly conditional; correctness and explicit failure are often preferable to silently hiding a broken state.

---

# 32. Deployment pipeline

A mature pipeline might be:

```text id="k9m2x6"
Commit
 ↓
Build
 ↓
Static validation
 ↓
Database tests
 ↓
Integration tests
 ↓
Security checks
 ↓
Staging deployment
 ↓
Smoke tests
 ↓
Approval
 ↓
Production deployment
 ↓
Post-deployment validation
```

---

# 33. Smoke tests

After deployment, execute a small set of critical checks:

```text id="v4p8c2"
Can application connect?
Can critical query execute?
Can critical transaction complete?
Are permissions correct?
Are errors normal?
```

---

# 34. Deployment gates

Production deployment can require:

```text id="m7x3n9"
Tests passed
+
Security passed
+
Approver approved
+
No active incident
```

This creates a controlled release process.

---

# 35. Canary deployment

Canary releases expose a new version to a small portion of traffic.

Conceptually:

```text id="q2c8m5"
95% → old
5%  → new
```

Observe:

```text id="n6v3p7"
Errors
Latency
Database load
```

Then expand if healthy.

---

# 36. Blue/green deployment

Conceptually:

```text id="r5m9x2"
BLUE  → current
GREEN → new
```

Validate GREEN.

Then switch traffic.

This can reduce application deployment risk, although database state synchronization remains a key challenge.

---

# 37. Database challenge with blue/green

You can't always simply create:

```text id="x8p4k1"
Database B
```

and switch instantly because:

```text id="m3q7v9"
Data changes
+
Transactions
+
Replication
+
Schema compatibility
```

must be handled.

---

# 38. Zero-downtime mindset

"Zero downtime" doesn't mean:

> Nothing changes.

It means:

> Changes are designed so users experience no unacceptable interruption.

This usually requires:

```text id="c9n2m6"
Backward compatibility
Incremental changes
Careful migration
Monitoring
Fast mitigation
```

---

# 39. Rollback

Application rollback:

```text id="v5x8q3"
Version 4
 ↓
Version 3
```

Database rollback may be much harder.

Example:

```text id="p2m7k9"
Column dropped
Data transformed
```

You may not have a trivial reverse operation.

---

# 40. Therefore:

Before deployment, ask:

> **If this fails halfway through, what state will the database be in?**

This question is more important than simply asking:

> "What's the rollback command?"

---

# 41. Recovery strategy

For risky changes, define:

```text id="n8c3x5"
Backup/recovery option
Forward fix
Rollback strategy
Validation
Escalation
```

---

# 42. Deployment observability

During deployment monitor:

```text id="m4p9v2"
CPU
I/O
Latency
Blocking
Deadlocks
Errors
Transaction log
Connections
```

A migration can be syntactically successful while damaging production performance.

---

# 43. Post-deployment comparison

Compare:

```text id="x7q3m8"
Before
vs
After
```

For:

```text id="c5m2n9"
Query duration
CPU
Reads
Errors
Deadlocks
Blocking
```

---

# 44. Deployment success criteria

Don't define success as:

```text id="r8v4p1"
"Pipeline returned green."
```

Define:

```text id="j3x7m5"
Schema correct
+
Application healthy
+
Latency within SLO
+
No unexpected errors
+
No unacceptable resource increase
```

---

# 45. Release metadata

Record:

```text id="q6m8c2"
Application version
Database version
Migration version
Commit SHA
Deployment time
Environment
```

This makes incident correlation much easier.

---

# 46. Example incident

At 14:00:

```text id="v9p2k5"
Migration deployed
```

At 14:05:

```text id="m3x8n1"
P95 latency doubles
```

Query Store shows:

```text id="c7q4m9"
Plan regression
```

Deployment metadata links the regression to:

```text id="x5r2p8"
Migration + index change
```

Now your CI/CD system has become an observability source.

---

# 47. Schema drift detection

Compare:

```text id="n7m4x2"
Git schema
vs
Production schema
```

Unexpected differences should trigger investigation.

---

# 48. Permission drift

Do the same for security:

```text id="q3p8c6"
Expected roles
vs
Actual roles
```

This catches unauthorized privilege changes.

---

# 49. Automated compliance

Pipeline can verify:

```text id="m8v2r5"
Required encryption
Required monitoring
Approved network
Required tags
Required identities
```

before production deployment.

---

# 50. Secrets in CI/CD

Never expose:

```text id="x4n9k3"
Passwords
Connection strings
Tokens
Private keys
```

in logs or source control.

Prefer managed identities and secure secret stores where supported.

---

# 51. Production change control

A mature organization combines DevOps with governance:

```text id="p7m3x8"
Code
 ↓
Automated validation
 ↓
Peer review
 ↓
Risk assessment
 ↓
Approval
 ↓
Deployment
 ↓
Evidence
```

---

# 52. Database DevOps maturity

### Level 1

```text id="j5c8n2"
Manual SQL scripts
```

### Level 2

```text id="v3m7p9"
Scripts in Git
```

### Level 3

```text id="x8q4m1"
Automated deployments
```

### Level 4

```text id="r6n2c5"
Automated testing + gates
```

### Level 5

```text id="m9p3v7"
Continuous, observable, low-risk database delivery
```

---

# 53. Interview question

### "How do you deploy database changes with minimal downtime?"

> "I design changes for backward compatibility, use an expand-and-contract strategy where appropriate, separate schema changes from data migration, perform large migrations incrementally, test them before production, monitor the workload during deployment, and validate application behavior afterward."

---

# 54. Interview question

### "DACPAC or migrations?"

> "A DACPAC-style approach models the desired schema and compares it with the target database. Migration-based deployment explicitly records ordered schema changes. I'd choose based on the team's tooling, database complexity, and whether explicit operational control over each migration is important."

---

# 55. Interview question

### "Can every database deployment be rolled back?"

> "No. Database changes can be stateful and data-destructive, so rollback may not be trivial or even possible. I design deployments to be backward compatible, use backups and recovery strategies appropriately, and prefer safe forward fixes where reversing the data transformation would be risky."

---

# 56. Interview question

### "What would you monitor during a production migration?"

```text id="k4x8m2"
Latency
CPU
I/O
Blocking
Deadlocks
Errors
Transaction log
Connections
Critical application transactions
```

Then compare against the pre-deployment baseline.

---

# 57. Complete DevOps architecture

```text id="c8m3v6"
                  GIT
                   |
              Pull Request
                   |
             Code Review
                   |
          +--------+--------+
          |                 |
      Application        Database
          |                 |
          +--------+--------+
                   |
              CI Pipeline
                   |
        +----------+----------+
        |                     |
     Tests                Security
        |                     |
        +----------+----------+
                   |
             Staging Deploy
                   |
              Smoke Tests
                   |
              Approval Gate
                   |
           Production Deploy
                   |
        +----------+----------+
        |                     |
      SQL MI              Application
        |                     |
        +----------+----------+
                   |
             Observability
                   |
          Validate SLO / Health
```

---

# 58. Golden rule

> **A database deployment isn't finished when the migration succeeds. It's finished when the application, database, performance, security, and reliability checks all confirm that the new state is healthy.**

The mature database engineer thinks:

```text
Change
 ↓
Compatibility
 ↓
Test
 ↓
Deploy
 ↓
Observe
 ↓
Validate
 ↓
Learn
```

---

## Next — Lesson 111: Azure SQL Managed Instance High Availability, Failover & Disaster Recovery — Advanced

We'll go deep into **HA architecture, failover behavior, zone/region considerations, RPO/RTO, failover groups, backup/restore recovery, planned vs unplanned failover, application connection handling, DR testing, regional disaster scenarios, and designing a production-grade recovery strategy.**
