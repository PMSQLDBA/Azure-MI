# Lesson 109 — Azure SQL Managed Instance Automation & Infrastructure as Code

This lesson is about turning DBA operations from **manual repetition** into **repeatable, auditable, version-controlled automation**.

The senior-level principle is:

> **If you perform the same operational task repeatedly, it should eventually become code, policy, or a pipeline.**

---

# 1. Why automate SQL MI?

Manual operations create:

```text
Human error
Configuration drift
Inconsistent environments
Slow deployments
Poor auditability
```

Automation gives:

```text
Consistency
Repeatability
Speed
Traceability
Scalability
```

---

# 2. The automation spectrum

Think of maturity as:

```text
Manual
  ↓
Script
  ↓
Reusable script
  ↓
Pipeline
  ↓
Infrastructure as Code
  ↓
Policy-driven automation
```

---

# 3. Manual provisioning

Example:

```text
Create SQL MI
Configure networking
Configure security
Create databases
Configure monitoring
Configure backup/DR
```

Doing this manually for every environment creates drift.

---

# 4. Scripted provisioning

You can automate with:

```text
Azure CLI
PowerShell
SQL scripts
```

Conceptually:

```text
Script
 ↓
Azure resources
 ↓
SQL configuration
```

Better than manual, but scripts can still become difficult to manage as infrastructure grows.

---

# 5. Infrastructure as Code

IaC describes the desired infrastructure declaratively.

Conceptually:

```text
Desired state
      ↓
IaC definition
      ↓
Deployment engine
      ↓
Azure resources
```

Examples include:

```text
Bicep
ARM templates
Terraform
```

---

# 6. Imperative vs declarative

### Imperative

You tell the system **how** to do something.

```text
Create resource
Configure property
Create database
Add setting
```

### Declarative

You describe **what state you want**.

```text
SQL MI should exist
with these properties
and these dependencies
```

The platform/tool determines how to achieve the desired state.

---

# 7. Why declarative infrastructure helps

Suppose production and staging should be identical except for approved environment-specific values.

IaC lets you define:

```text
Common configuration
+
Environment parameters
```

rather than manually recreating the environment.

---

# 8. Bicep

Bicep is Azure-focused infrastructure-as-code syntax.

Conceptually:

```text
Bicep
 ↓
Azure Resource Manager
 ↓
Azure resources
```

It is designed specifically for Azure resource deployment.

---

# 9. ARM templates

ARM templates represent Azure infrastructure declaratively using JSON.

They are powerful but can become verbose.

Bicep provides a cleaner authoring experience while targeting Azure Resource Manager.

---

# 10. Terraform

Terraform is a broader infrastructure-as-code platform.

Conceptually:

```text
Terraform
 ↓
Provider
 ↓
Azure
```

It can be useful when your organization manages multiple infrastructure platforms through a common IaC workflow.

---

# 11. IaC repository

A mature repository might look like:

```text id="r4m8x2"
infra/
 ├── modules/
 │    ├── sqlmi/
 │    ├── networking/
 │    ├── monitoring/
 │    └── security/
 │
 ├── environments/
 │    ├── dev/
 │    ├── staging/
 │    └── production/
 │
 └── pipelines/
```

Exact structure depends on organizational standards.

---

# 12. Modules

Don't duplicate entire SQL MI definitions everywhere.

Create reusable components:

```text id="k7p3n9"
SQL MI module
Network module
Monitoring module
Security module
```

Then compose them.

---

# 13. Parameterization

Avoid hardcoding:

```text id="x2m8q5"
Environment
Region
SKU
Subnet
Tags
```

Instead:

```text id="v9c4r1"
Environment parameters
        ↓
Reusable module
```

---

# 14. Environment separation

For example:

```text id="m5q8x3"
DEV
 → small resources

STAGING
 → production-like

PRODUCTION
 → production configuration
```

The same architecture can be reused with controlled parameters.

---

# 15. Configuration drift

Suppose IaC says:

```text id="p7n3m6"
Setting = A
```

Someone manually changes Azure to:

```text id="c4x8q2"
Setting = B
```

Now:

```text id="j9m2v5"
Desired state ≠ Actual state
```

That's configuration drift.

---

# 16. Why drift is dangerous

Drift can produce:

```text id="r3k7n8"
Environment inconsistency
Unexpected behavior
Security gaps
Difficult troubleshooting
Deployment surprises
```

---

# 17. Drift detection

A mature process regularly compares:

```text id="x5m9p1"
Desired configuration
        vs
Actual configuration
```

Then reports differences.

---

# 18. Git as the source of truth

A common pattern:

```text id="q8v4m3"
Git
 ↓
IaC
 ↓
Pipeline
 ↓
Azure
```

Changes should ideally be reviewed before reaching production.

---

# 19. Pull requests

Production infrastructure changes should typically follow:

```text id="n6x2p8"
Developer/DBA
 ↓
Git branch
 ↓
Pull request
 ↓
Review
 ↓
Validation
 ↓
Merge
 ↓
Deployment
```

This gives you traceability.

---

# 20. CI/CD

CI/CD means automating validation and delivery.

Conceptually:

```text id="m3r7k5"
Commit
 ↓
Build
 ↓
Validate
 ↓
Test
 ↓
Deploy
 ↓
Verify
```

---

# 21. Database CI/CD

Database changes need special care because schema changes affect stateful systems.

Typical flow:

```text id="v8c2n6"
Migration script
 ↓
Validation
 ↓
Test database
 ↓
Integration testing
 ↓
Staging
 ↓
Production
```

---

# 22. Expand-and-contract deployment

A safer database deployment pattern:

```text id="p5m9x3"
Expand
 ↓
Deploy compatible application
 ↓
Backfill/migrate
 ↓
Switch application behavior
 ↓
Contract
```

---

# 23. Example

Old application:

```text id="j2k8v4"
Name
```

New requirement:

```text id="q6m3x9"
FirstName
LastName
```

Don't immediately delete `Name`.

Instead:

```text id="r8p4n1"
1. Add FirstName
2. Add LastName
3. Deploy compatible application
4. Migrate data
5. Switch application
6. Validate
7. Remove Name later
```

This reduces deployment risk.

---

# 24. Idempotency

An automation script should ideally be safe to run repeatedly.

Bad:

```text id="x7m2c5"
CREATE something
```

every time without checking whether it already exists.

Better:

```text id="n4v8q1"
Ensure desired state
```

---

# 25. Idempotent mindset

Instead of:

> "Create X."

Think:

> "Ensure X exists with these properties."

That's a fundamental IaC concept.

---

# 26. Azure CLI

Azure CLI is useful for:

```text id="m8k3p6"
Provisioning
Querying resources
Automation
Pipelines
Operational scripts
```

Typical conceptual flow:

```text
az ...
   ↓
Azure Resource Manager
   ↓
Resource
```

---

# 27. PowerShell

PowerShell is useful when you need:

```text id="v2x7m4"
Complex automation
Object-oriented command output
Windows-oriented administration
Azure automation
Database scripting
```

---

# 28. CLI vs PowerShell

Don't treat one as universally superior.

Choose based on:

```text id="c9m5p2"
Team expertise
Existing tooling
Pipeline environment
Operating system
Script complexity
```

---

# 29. SQL automation

Azure infrastructure automation isn't enough.

Database-level operations can also be automated:

```text id="q4n8x1"
Schema deployment
Role creation
Permission grants
Stored procedure deployment
Index maintenance
Validation
```

---

# 30. Security automation

Automate security controls where practical.

Examples:

```text id="r6m2v9"
Standard roles
Auditing
Monitoring
Private networking
Policy checks
Access reviews
```

---

# 31. Policy as code

Security and governance rules can be expressed as policies.

Example concept:

```text id="x3p7k5"
Production SQL MI
MUST:
→ use approved network
→ have required monitoring
→ have required tags
→ meet security baseline
```

---

# 32. Why policy automation matters

Without policy:

```text id="n9c4m6"
Developer creates resource
 ↓
Forgets monitoring
 ↓
Production resource exists
 ↓
Nobody notices
```

With automated policy:

```text id="m5x8q2"
Non-compliant resource
 ↓
Detected/blocked/remediated
```

depending on the policy configuration.

---

# 33. Tags

Standardize tags such as:

```text id="v7p3n8"
Environment
Application
Owner
CostCenter
Criticality
DataClassification
```

Tags improve:

```text
Cost management
Ownership
Automation
Governance
Incident response
```

---

# 34. Monitoring automation

Don't manually configure every alert.

Automate:

```text id="q2m9x4"
Alerts
Dashboards
Diagnostic settings
Log routing
Action groups
```

where appropriate.

---

# 35. Backup/DR automation

Automate validation of:

```text id="c6x8p1"
Backup configuration
Retention
DR configuration
Replication health
Failover readiness
```

And most importantly:

> **Test recovery, not just configuration.**

---

# 36. Automated health checks

A pipeline can verify:

```text id="m3v7k9"
SQL MI reachable
Database accessible
Expected schema exists
Expected permissions exist
Critical query succeeds
Monitoring telemetry exists
```

---

# 37. Deployment gates

Before production:

```text id="r8n2x5"
Security check
 ↓
IaC validation
 ↓
Database migration test
 ↓
Performance test
 ↓
Approval
 ↓
Production
```

---

# 38. Automated rollback

Not every database change can be safely rolled back.

This is critical.

Application binaries may be easy to roll back:

```text id="x5m8c2"
Version 3
 ↓
Version 2
```

Database schema changes can be much harder:

```text id="q7p3n9"
Data migration
 ↓
Irreversible state change
```

Design deployment strategies accordingly.

---

# 39. Forward-fix mindset

For many database migrations, instead of trying to reverse every data change:

```text id="v4m6x8"
Detect problem
 ↓
Deploy corrective migration
```

may be safer than destructive rollback.

---

# 40. Schema migration tracking

A mature deployment process tracks:

```text id="n8c3p5"
Migration ID
Version
Timestamp
Environment
Result
```

This makes database state auditable.

---

# 41. Secrets

Never put passwords directly into:

```text id="j5m9x2"
Git
IaC files
Scripts
Pipeline logs
```

Use an appropriate secret-management mechanism and preferably identity-based authentication where supported.

---

# 42. Secret scanning

CI pipelines can scan repositories for accidental secrets.

Conceptually:

```text id="r2x7m4"
Commit
 ↓
Secret scan
 ↓
Secret found?
 ↓
FAIL
```

---

# 43. Managed identity in automation

Automation itself can use identities.

Instead of:

```text id="m6q3v8"
Pipeline
 ↓
Long-lived Azure password
```

prefer an identity-based mechanism appropriate to your CI/CD platform.

---

# 44. Least privilege for pipelines

A deployment pipeline shouldn't automatically have:

```text id="p9k2x5"
Subscription Owner
```

if it only needs:

```text id="c4m8n1"
Deploy SQL resources
Deploy database schema
Read monitoring state
```

Give it only what it needs.

---

# 45. Production approval

A common pattern:

```text id="x7v3m9"
DEV
 ↓
AUTOMATED TEST
 ↓
STAGING
 ↓
AUTOMATED TEST
 ↓
APPROVAL
 ↓
PRODUCTION
```

Automation does not mean removing governance.

---

# 46. Infrastructure testing

IaC should be tested before deployment.

Examples:

```text id="q5m8c2"
Syntax validation
Schema validation
Policy validation
Security validation
Dependency validation
```

---

# 47. Database testing

Test:

```text id="n3x7p6"
Schema
Permissions
Stored procedures
Critical queries
Data migration
Performance impact
Application compatibility
```

---

# 48. Production validation

After deployment:

```text id="m8r2k4"
Resource healthy?
Database accessible?
Application healthy?
Query latency normal?
Errors normal?
Monitoring working?
```

Deployment isn't complete until validation succeeds.

---

# 49. Automation architecture

A mature system can look like:

```text id="v6c9p3"
             Git
              |
        Pull Request
              |
       Validation / Tests
              |
        Approval / Policy
              |
            CI/CD
              |
       +------+------+
       |             |
     IaC          DB Migration
       |             |
       +------+------+
              |
             Azure
              |
       +------+------+
       |             |
    SQL MI       Monitoring
       |             |
       +------+------+
              |
          Validation
```

---

# 50. GitOps mindset

The desired state lives in version control.

Conceptually:

```text id="p2x8m5"
Git
 ↓
Desired state
 ↓
Automation
 ↓
Azure
```

Changes are reviewed and auditable.

---

# 51. What should NOT be automated blindly?

Automation can amplify mistakes.

Be careful with:

```text id="j4m7v2"
DROP DATABASE
Mass permission removal
Production failover
Destructive migrations
Mass index changes
Credential rotation
```

Use:

```text
Approval
Validation
Guardrails
Rollback/recovery plan
```

---

# 52. Safe automation pattern

For risky operations:

```text id="x8q3m6"
Detect
 ↓
Preview
 ↓
Validate
 ↓
Approve
 ↓
Execute
 ↓
Verify
```

---

# 53. Dry runs

Where supported, preview changes before applying them.

The principle:

> **Know what automation will change before allowing it to change production.**

---

# 54. Automation logging

Every important automation job should record:

```text id="m5v9c2"
Who/identity
What changed
When
Environment
Result
Errors
Version/commit
```

---

# 55. Automation failure

Suppose:

```text id="r7x2n8"
Pipeline fails after schema migration
```

Don't simply rerun blindly.

Determine:

```text id="c3m6p9"
What completed?
What failed?
Is migration partially applied?
Is it idempotent?
Can it safely resume?
```

---

# 56. Partial state

Stateful systems make partial failures particularly important.

Example:

```text id="v4k8m2"
Step 1 → completed
Step 2 → completed
Step 3 → failed
Step 4 → never executed
```

The automation must understand current state before retrying.

---

# 57. State management

IaC tools may maintain state describing deployed resources.

Protect that state carefully because it can contain sensitive infrastructure information.

Use appropriate:

```text id="n9p3x5"
Access control
Storage
Locking
Versioning
Backup
```

---

# 58. Cost automation

Automation can also control cost.

Examples:

```text id="q6m2v8"
Non-production schedules
Unused resource detection
Capacity reviews
Tag-based cost allocation
```

Be careful not to automate shutdowns that violate business requirements.

---

# 59. Automated capacity reporting

Generate reports:

```text id="x3r7m5"
Current capacity
Growth
Forecast
Top consumers
Potential exhaustion date
```

This turns reactive DBA work into proactive planning.

---

# 60. Automation maturity model

### Level 1

```text id="p8c2m6"
Manual DBA
```

### Level 2

```text id="v5x9n3"
Scripts
```

### Level 3

```text id="j7m4q1"
Reusable automation
```

### Level 4

```text id="r2k8p5"
CI/CD + IaC
```

### Level 5

```text id="m6x3v9"
Policy-driven, observable, self-service platform
```

---

# 61. Interview question

### "Why use Infrastructure as Code?"

> "IaC makes infrastructure reproducible, reviewable, version-controlled, and consistent across environments. It also reduces configuration drift and makes infrastructure changes auditable."

---

# 62. Interview question

### "What's the difference between Bicep and Terraform?"

> "Bicep is Azure-native infrastructure-as-code syntax that compiles to Azure Resource Manager deployments. Terraform is a broader IaC platform using providers to manage resources across Azure and other platforms. The right choice depends on organizational architecture and tooling requirements."

---

# 63. Interview question

### "How would you automate SQL MI provisioning?"

Strong answer:

```text id="x4m8q2"
IaC module
 ↓
Network configuration
 ↓
SQL MI configuration
 ↓
Security baseline
 ↓
Monitoring
 ↓
Database provisioning
 ↓
Roles/permissions
 ↓
Validation
```

Everything should be version-controlled and environment-parameterized.

---

# 64. Interview question

### "How do you prevent deployment from breaking production?"

> "I use version-controlled migrations, automated validation, staging, compatibility-focused schema changes, security checks, deployment gates, monitoring, and post-deployment validation. For risky changes, I use an expand-and-contract approach rather than assuming rollback is always possible."

---

# 65. Interview question

### "How do you handle configuration drift?"

> "I define the desired state in IaC, regularly compare it with actual infrastructure, detect unauthorized changes, and either reconcile or investigate the drift according to operational policy."

---

# 66. Final automation model

```text id="n7q3m8"
              GIT
               |
          CODE REVIEW
               |
         VALIDATION
               |
       SECURITY / POLICY
               |
            CI/CD
               |
       +-------+-------+
       |               |
      IaC          DB MIGRATIONS
       |               |
       +-------+-------+
               |
             AZURE
               |
             SQL MI
               |
          MONITORING
               |
           VALIDATE
               |
            REPORT
```

---

# 67. Golden rule

> **Automate the process, not just the command.**

A mature DBA doesn't merely create a script that says:

```text
"Run this command."
```

They build a workflow that handles:

```text
Plan
→ Validate
→ Approve
→ Execute
→ Observe
→ Verify
→ Record
```

That is the difference between **scripting** and **production-grade automation**.

---

## Next — Lesson 110: Azure SQL Managed Instance CI/CD & Database DevOps

We'll go deeper into **database migration strategies, DACPAC vs migration-based deployment, schema versioning, Git workflows, automated testing, blue/green and canary patterns, zero/low-downtime deployments, backward compatibility, deployment gates, rollback strategy, and production database release engineering.**
