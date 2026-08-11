# Lesson 120 — Azure SQL Managed Instance Automation & Infrastructure as Code — Expert Level

The goal is to move from:

> **“I can create a SQL Managed Instance.”**

to:

> **“I can reproducibly build, secure, configure, deploy, validate, and operate an enterprise SQL MI platform entirely through controlled automation.”**

---

# 1. Infrastructure as Code

**Infrastructure as Code (IaC)** means defining infrastructure declaratively rather than manually creating resources through the portal.

Instead of:

```text
Portal
 ↓
Click
 ↓
Configure
 ↓
Create
```

you want:

```text
Git
 ↓
IaC definition
 ↓
Review
 ↓
Pipeline
 ↓
Azure
```

---

# 2. Why IaC matters

Manual infrastructure creates:

```text
Configuration drift
Human error
Inconsistent environments
Poor auditability
Difficult disaster recovery
```

IaC gives you:

```text
Repeatability
Version control
Reviewability
Automation
Consistency
```

---

# 3. Declarative vs imperative

### Imperative

You describe **how** to do something.

```text
Create network
Create subnet
Create route
Create SQL MI
Configure setting
```

### Declarative

You describe **what the final state should be**.

```text
I want:
VNet
Subnet
SQL MI
Private connectivity
Specific configuration
```

The IaC engine determines how to reach that state.

---

# 4. Azure IaC choices

Common approaches include:

```text
Azure Bicep
ARM templates
Terraform
Azure CLI / PowerShell
```

A mature organization may use more than one, but standardization is usually preferable.

---

# 5. Bicep

Microsoft Bicep is Azure's domain-specific language for declaratively deploying Azure resources.

Conceptually:

```text
Bicep
 ↓
Azure Resource Manager
 ↓
Azure resources
```

It provides Azure-native infrastructure modeling.

---

# 6. ARM templates

ARM templates represent Azure infrastructure declaratively in JSON.

They are powerful but can become verbose.

Conceptually:

```text
Bicep
   ↓ compile
ARM template
   ↓
Azure Resource Manager
```

---

# 7. Terraform

Terraform uses declarative configuration and a state model.

Conceptually:

```text
Terraform configuration
        ↓
Terraform state
        ↓
Provider
        ↓
Azure
```

Terraform is particularly useful in organizations managing infrastructure across multiple platforms or clouds.

---

# 8. The state problem

Terraform needs to understand:

```text
What I declared
vs
What already exists
```

That's where state becomes important.

A production Terraform setup should carefully manage:

```text
State storage
State locking
Access control
Secrets
Backups
Recovery
```

Never casually store production state on an engineer's laptop.

---

# 9. State drift

Suppose IaC says:

```text
SKU = X
```

but someone manually changes Azure to:

```text
SKU = Y
```

Now:

```text
Declared state ≠ Actual state
```

That's **configuration drift**.

---

# 10. Drift detection

A mature platform regularly asks:

```text
Desired state
      vs
Actual state
```

Then determines whether differences are:

```text
Expected
Unauthorized
Temporary
Required remediation
```

---

# 11. Git as the source of truth

A common architecture:

```text
Git repository
      ↓
Pull request
      ↓
Code review
      ↓
Validation
      ↓
Pipeline
      ↓
Azure
```

This creates an auditable infrastructure history.

---

# 12. Repository structure

A production repository might conceptually look like:

```text
infra/
 ├── modules/
 │    ├── network/
 │    ├── sqlmi/
 │    ├── monitoring/
 │    └── security/
 │
 ├── environments/
 │    ├── dev/
 │    ├── test/
 │    └── prod/
 │
 └── pipelines/
```

The exact structure depends on the IaC technology and organization.

---

# 13. Modules

Avoid repeating giant infrastructure definitions.

Instead:

```text
SQL MI module
Network module
Monitoring module
Identity module
```

Then compose them.

---

# 14. Reusable module design

A good module exposes meaningful inputs:

```text
environment
region
network
subnet
compute configuration
storage configuration
identity
tags
monitoring configuration
```

and hides implementation details that consumers shouldn't manipulate.

---

# 15. Environment separation

Typical environments:

```text
Development
Testing
Staging
Production
```

You want:

```text
Same architecture
Different parameters
```

rather than:

```text
Completely different infrastructure
```

---

# 16. Configuration hierarchy

Think:

```text
Global defaults
      ↓
Environment configuration
      ↓
Regional configuration
      ↓
Resource-specific overrides
```

Avoid duplicating the same configuration everywhere.

---

# 17. Naming conventions

Automated infrastructure needs deterministic names.

Example concept:

```text
<org>-<environment>-<region>-<resource>
```

For example:

```text
contoso-prod-eastus-sqlmi
```

The exact convention should be standardized.

---

# 18. Tags

Tags are extremely useful for governance.

Examples:

```text
Environment
Owner
Application
CostCenter
DataClassification
BusinessUnit
Criticality
```

---

# 19. Azure Policy

Policy lets organizations enforce rules.

Conceptually:

```text
Developer wants resource
        ↓
Policy evaluation
        ↓
Allowed / denied / audited
```

Examples:

```text
Require tags
Restrict regions
Require approved SKUs
Enforce security settings
```

---

# 20. Policy vs IaC

These solve different problems.

### IaC

> **What should we deploy?**

### Policy

> **What are we allowed to deploy?**

Use both.

---

# 21. Resource locks

Resource locks can protect critical resources against accidental deletion.

Conceptually:

```text
Production SQL MI
       ↓
Deletion protection
```

But locks can also interfere with automation.

Therefore:

> Use them deliberately, not blindly.

---

# 22. Identity

Automation needs Azure permissions.

Avoid:

```text
Pipeline
 ↓
Global administrator credentials
```

Prefer:

```text
Pipeline identity
 ↓
Least-privilege permissions
 ↓
Required resources only
```

---

# 23. Managed identities

Managed identities reduce the need to store long-lived credentials.

Conceptually:

```text
Azure workload
      ↓
Managed identity
      ↓
Azure resource
```

This is preferable to embedding secrets in scripts whenever supported.

---

# 24. Secrets

Never put secrets directly into:

```text
Git
IaC files
Pipeline YAML
Shell scripts
```

Use an appropriate secret-management solution such as:

Azure Key Vault.

---

# 25. Secret flow

A better architecture:

```text
Application / Pipeline
        ↓
Identity
        ↓
Key Vault
        ↓
Secret
```

The secret isn't permanently embedded in source code.

---

# 26. SQL authentication

Where possible, prefer strong identity-based authentication patterns.

If SQL authentication is required:

```text
Credential
 ↓
Secure secret store
 ↓
Controlled retrieval
```

Rotate credentials according to organizational requirements.

---

# 27. Networking as IaC

Remember:

```text
SQL MI
```

doesn't exist in isolation.

Infrastructure automation may need to create/configure:

```text
VNet
Subnet
NSGs
Route configuration
DNS
Private connectivity
Firewall/security controls
```

Networking dependencies must be modeled explicitly.

---

# 28. Dependency graph

Think of deployment as:

```text
Resource Group
      ↓
VNet
      ↓
Subnet
      ↓
Network configuration
      ↓
SQL MI
      ↓
Databases
      ↓
Monitoring
```

The deployment engine must understand these dependencies.

---

# 29. SQL MI subnet

SQL MI requires specific networking architecture.

Therefore:

> **Don't treat the subnet as an ordinary application subnet.**

Its configuration and delegated requirements should be encoded and validated as part of the platform module.

---

# 30. Production topology

A production architecture may conceptually look like:

```text
Hub
 |
 +---- Shared services
 |
Spoke VNet
 |
 +---- SQL MI subnet
 |
 +---- Application subnet
 |
 +---- Private endpoints/services
```

Exact topology depends on organizational architecture.

---

# 31. Network security

Infrastructure code should express:

```text
Allowed connectivity
Routing
Security controls
DNS dependencies
Private access
```

rather than leaving critical security decisions to manual portal configuration.

---

# 32. Immutable vs mutable infrastructure

### Mutable

```text
Create resource
 ↓
Change resource manually
 ↓
Change again
```

### IaC-driven

```text
Desired definition
 ↓
Controlled change
 ↓
Deployment
```

The second model gives you a much clearer history.

---

# 33. Infrastructure changes should be reviewed

A production change should ideally follow:

```text
Developer
 ↓
Pull request
 ↓
Automated validation
 ↓
Peer review
 ↓
Approval
 ↓
Deployment
```

---

# 34. Terraform plan

Terraform can show intended changes before applying them.

Conceptually:

```text
Current
  ↓
Plan
  ↓
+
-
~
```

Where the symbols represent creation, deletion, or modification.

The important idea is:

> **Review the change before execution.**

---

# 35. Bicep validation

Bicep deployments can similarly be validated and previewed through Azure deployment mechanisms.

The principle remains:

```text
Validate
 ↓
Preview
 ↓
Review
 ↓
Deploy
```

---

# 36. CI pipeline

A useful infrastructure pipeline:

```text
Commit
 ↓
Lint
 ↓
Validate
 ↓
Security scan
 ↓
Plan / What-if
 ↓
Tests
```

Production deployment happens only after appropriate approvals.

---

# 37. CD pipeline

Deployment:

```text
Dev
 ↓
Test
 ↓
Staging
 ↓
Production
```

Promotion should use the same infrastructure patterns.

---

# 38. Environment promotion

Avoid manually rebuilding production.

Instead:

```text
Same module
+
Production parameters
```

This reduces environment divergence.

---

# 39. Infrastructure tests

Test:

```text
Resource existence
Network configuration
Security settings
Required tags
Identity configuration
Monitoring
Policy compliance
```

---

# 40. Negative tests

Don't test only:

> "Does the resource exist?"

Also test:

> "Can an unauthorized configuration be created?"

For example:

```text
Missing required tag
Unauthorized region
Unsupported configuration
Public exposure
```

Expected result:

```text
Deployment blocked
```

---

# 41. SQL configuration automation

Infrastructure automation can establish supported SQL MI configuration such as:

```text
Instance configuration
Security settings
Networking
Monitoring
```

Database-level objects may require a separate database deployment layer.

---

# 42. Infrastructure vs database deployment

Separate:

```text
Infrastructure
```

from:

```text
Database schema/application
```

Conceptually:

```text
Infrastructure pipeline
       ↓
SQL MI available
       ↓
Database deployment
       ↓
Schema
       ↓
Stored procedures
       ↓
Indexes
```

---

# 43. Database migrations

Use a migration framework/process for:

```text
Tables
Columns
Indexes
Procedures
Functions
Views
Data migrations
```

Avoid arbitrary manual production SQL.

---

# 44. Migration ordering

Example:

```text
1. Add new nullable column
2. Deploy application
3. Backfill data
4. Validate
5. Enforce constraint
6. Remove old column later
```

This supports safer rolling changes.

---

# 45. Expand-and-contract

This is a powerful deployment pattern.

### Expand

Add new capability without removing the old one.

```text
Old schema
+
New schema
```

### Migrate

Move application/data usage.

### Contract

Remove obsolete components.

```text
New schema
```

This reduces deployment coupling.

---

# 46. Why destructive changes are dangerous

Suppose version 1 expects:

```text
OldColumn
```

and deployment immediately removes it.

If an old application instance is still running:

```text
Old app
 ↓
OldColumn
 ↓
ERROR
```

Expand-and-contract avoids this class of failure.

---

# 47. Rollback

Infrastructure rollback and database rollback are different.

Infrastructure:

```text
Restore previous configuration
```

Database:

```text
Schema/data may have changed
```

You cannot assume:

> "Just roll back the deployment."

---

# 48. Database rollback planning

Before a migration, determine:

```text
Can schema changes be reversed?
Can data changes be reversed?
How long would rollback take?
Is restore required?
Can application rollback occur independently?
```

---

# 49. Backward compatibility

A production deployment should ideally support:

```text
Old application
+
New database
```

for a transition period.

Then:

```text
New application
+
New database
```

This is especially valuable in distributed systems.

---

# 50. Deployment gates

Production deployment can require:

```text
Tests passed
Security scan passed
Policy compliance
Plan reviewed
Approval obtained
Backup/recovery conditions satisfied
```

---

# 51. Automated validation after deployment

Don't stop when Azure says:

```text
Deployment succeeded
```

Run functional and infrastructure checks:

```text
SQL MI reachable
Authentication works
Expected databases exist
Expected configuration exists
Monitoring is active
Application connectivity works
```

---

# 52. Smoke tests

A simple smoke test might:

```text
Connect
 ↓
Run lightweight query
 ↓
Validate expected result
```

For example:

```sql id="4p8z1k"
SELECT 1;
```

Then application-level health checks should validate actual functionality.

---

# 53. Post-deployment performance validation

Compare:

```text
Before deployment
vs
After deployment
```

Metrics:

```text
Latency
CPU
Reads
Errors
Throughput
Waits
```

---

# 54. Canary deployment

For appropriate architectures:

```text
Small traffic
 ↓
Validate
 ↓
Increase traffic
 ↓
Validate
 ↓
Full rollout
```

This limits blast radius.

---

# 55. Blue/green architecture

Conceptually:

```text
Blue = current
Green = new
```

Deploy and validate Green:

```text
Blue
 ↓
Green ready
 ↓
Switch traffic
```

Useful when the architecture supports independent environments.

---

# 56. Feature flags

Feature flags can decouple:

```text
Deployment
```

from:

```text
Feature activation
```

Example:

```text
New database capability deployed
 ↓
Feature disabled
 ↓
Validate
 ↓
Enable gradually
```

---

# 57. Disaster recovery and IaC

IaC becomes extremely valuable in disaster recovery.

Instead of:

```text
"Remember how production was configured?"
```

you have:

```text
Git
 ↓
Infrastructure definition
 ↓
Rebuild environment
```

---

# 58. But IaC isn't your data backup

Critical distinction:

```text
IaC
→ rebuild infrastructure

Backup
→ recover data
```

You need both.

---

# 59. Recovery architecture

Conceptually:

```text
IaC repository
      +
Database backups
      +
Secrets/configuration
      +
DNS/network configuration
```

Together they enable disaster recovery.

---

# 60. Recovery testing

A DR plan isn't proven because documentation exists.

Test:

```text
Deploy
 ↓
Restore
 ↓
Configure
 ↓
Connect application
 ↓
Validate
```

Measure:

```text
RTO
RPO
```

---

# 61. RTO

**Recovery Time Objective**

> How quickly must service be restored?

Example:

```text
RTO = 2 hours
```

---

# 62. RPO

**Recovery Point Objective**

> How much data loss is acceptable?

Example:

```text
RPO = 15 minutes
```

---

# 63. IaC and compliance

Infrastructure code can encode standards such as:

```text
Required tags
Approved regions
Security configuration
Monitoring
Identity requirements
Network restrictions
```

This turns compliance into repeatable automation.

---

# 64. Policy as code

A mature organization moves from:

```text
Security checklist
```

toward:

```text
Automated policy
 ↓
Deployment validation
 ↓
Continuous compliance
```

---

# 65. Separation of duties

A production pipeline may enforce:

```text
Engineer writes code
 ↓
Reviewer approves
 ↓
Pipeline deploys
```

rather than allowing one person to make unreviewed production changes.

---

# 66. Least privilege for pipelines

Don't give:

```text
Owner on entire subscription
```

unless there is a compelling, controlled reason.

Prefer:

```text
Specific scope
+
Specific actions
+
Specific identity
```

---

# 67. Pipeline security

Protect:

```text
Source repository
Pipeline credentials
Terraform state
Deployment identities
Secrets
Artifacts
```

The pipeline itself is production infrastructure.

---

# 68. Terraform state security

State may contain sensitive configuration information.

Therefore:

```text
Secure backend
Encryption
Access control
Versioning/recovery
Locking
```

are important.

---

# 69. Secrets in Terraform

Be careful:

Even if a secret isn't visible in your `.tf` file, it may still appear in state depending on how the resource/provider handles it.

Design secret flows intentionally.

---

# 70. Drift response

Suppose production drift is detected.

Possible policies:

### Audit

```text
Notify only
```

### Correct

```text
Restore desired state
```

### Block

```text
Prevent unauthorized change
```

Choose based on resource criticality.

---

# 71. GitOps-style model

A mature workflow can resemble:

```text
Git desired state
       ↓
Controller/pipeline
       ↓
Azure
       ↓
Observed state
       ↓
Drift detection
       ↓
Correction
```

The exact implementation varies.

---

# 72. Disaster recovery exercise

Imagine the production SQL MI environment is lost.

Can your team answer:

```text
Where is the infrastructure definition?
Where are backups?
Where are secrets?
How is networking rebuilt?
How is DNS restored?
How is the application connected?
Who executes recovery?
How long does it take?
```

If not, the architecture isn't truly automated.

---

# 73. Expert interview question

### "Why use IaC for SQL MI?"

Strong answer:

> "IaC makes the SQL MI platform reproducible, reviewable, auditable, and recoverable. It lets us standardize networking, security, monitoring, identity, and instance configuration across environments while reducing manual configuration drift."

---

# 74. Interview question

### "Terraform state was changed manually. What problem does that create?"

> "Terraform uses state to map declared resources to real infrastructure. Unexpected state changes can cause incorrect plans, resource replacement, drift, or loss of Terraform's understanding of the environment. State therefore needs controlled storage, access, locking, backup, and recovery."

---

# 75. Interview question

### "How would you deploy a database schema safely?"

> "I'd use versioned migrations and backward-compatible expand-and-contract patterns where possible. I'd validate migrations in lower environments, control production deployment through CI/CD, monitor performance after deployment, and have a documented recovery strategy for both schema and data changes."

---

# 76. Interview question

### "IaC says the resource is correct, but Azure differs. What do you do?"

> "I'd treat it as potential drift. First determine whether the difference is intentional and identify how it occurred. Then reconcile the source of truth and actual state through a reviewed change rather than blindly overwriting production."

---

# 77. Production automation blueprint

```text id="x8m3q1"
                    Git
                     ↓
                Pull Request
                     ↓
          +----------+----------+
          |                     |
       Validation           Security
          |                  Scanning
          +----------+----------+
                     ↓
              Plan / What-if
                     ↓
                  Review
                     ↓
                  Approval
                     ↓
              Development
                     ↓
                  Testing
                     ↓
                 Staging
                     ↓
               Production
                     ↓
             Smoke Testing
                     ↓
          Monitoring / Validation
                     ↓
              Drift Detection
```

---

# 78. The complete platform model

```text id="r4n7v2"
                    GIT
                     |
             +-------+-------+
             |               |
           IaC          DB Migrations
             |               |
             ↓               ↓
         Azure Platform   SQL MI Schema
             |               |
      +------+-------+       |
      |      |       |       |
    VNet   Identity  Monitor |
      |      |       |       |
      +------+-------+-------+
                     |
                 Application
                     |
                 Telemetry
                     |
               Observability
                     |
              Incident Response
                     |
              Continuous Change
```

---

# 79. Golden rule

> **If production infrastructure cannot be rebuilt predictably from version-controlled definitions, it isn't fully automated.**

And one more:

> **Automation should reduce human error—not automate human mistakes.**

---

# 80. Final mental model

```text id="m6p2x8"
DEFINE
  ↓
VERSION
  ↓
VALIDATE
  ↓
REVIEW
  ↓
DEPLOY
  ↓
TEST
  ↓
OBSERVE
  ↓
DETECT DRIFT
  ↓
IMPROVE
```

That is the foundation of **expert Azure SQL Managed Instance Infrastructure as Code and automation**.

---

## Next — Lesson 121: Azure SQL Managed Instance Security Architecture — Expert Level

We'll go deeper into **Microsoft Entra authentication, managed identities, RBAC vs database permissions, TDE, Always Encrypted, Microsoft Defender for SQL, auditing, vulnerability assessment, private networking, NSGs, identity boundaries, secrets, key management, data classification, least privilege, threat modeling, compliance, and designing a defense-in-depth SQL MI security architecture.**
