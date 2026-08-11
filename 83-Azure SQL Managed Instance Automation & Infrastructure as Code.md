# Lesson 115 — Azure SQL Managed Instance Automation & Infrastructure as Code

The goal is to move from:

> **“I can deploy and configure SQL MI manually.”**

to:

> **“I can create a consistent, secure, repeatable SQL MI platform from code, validate it automatically, and safely promote it across environments.”**

---

# 1. Why automation matters

Manual deployment creates:

```text
Configuration drift
Human error
Inconsistent environments
Slow recovery
Poor auditability
```

Automation gives:

```text
Code
 ↓
Review
 ↓
Deploy
 ↓
Validate
 ↓
Repeat
```

---

# 2. Infrastructure as Code

Infrastructure as Code, or IaC, means defining infrastructure declaratively in files.

Instead of:

```text
Portal
 ↓
Click
 ↓
Click
 ↓
Click
```

you have:

```text
Code
 ↓
Version control
 ↓
Pipeline
 ↓
Azure
```

---

# 3. Common Azure IaC approaches

Two major choices are:

```text
Bicep
Terraform
```

You may also use:

```text
Azure CLI
Azure PowerShell
ARM APIs
```

for automation and operational tasks.

---

# 4. Bicep

Bicep is Azure-focused infrastructure-as-code tooling.

Conceptually:

```text
Bicep
 ↓
Azure Resource Manager
 ↓
Azure resources
```

It is particularly useful when your organization is heavily Azure-centric.

---

# 5. Terraform

Terraform uses declarative configuration and maintains state about infrastructure.

Conceptually:

```text
Terraform configuration
        ↓
Terraform plan
        ↓
Terraform apply
        ↓
Azure
```

It is especially useful for organizations managing multiple cloud platforms or standardizing on Terraform.

---

# 6. Declarative thinking

You describe:

> **What the infrastructure should look like.**

Rather than:

> **Which individual clicks should I perform?**

Example concept:

```text
SQL MI should have:
- specified subnet
- specified compute configuration
- specified networking
- specified identity
- specified security configuration
```

---

# 7. Desired state

Think:

```text id="7u4jbt"
Desired state
     ↓
Actual state
     ↓
Compare
     ↓
Difference
     ↓
Reconcile
```

This is the foundation of IaC.

---

# 8. Configuration drift

Suppose your code says:

```text
Environment = Production
Setting A = X
Setting B = Y
```

Someone manually changes:

```text
Setting B = Z
```

Now:

```text
Code ≠ Azure
```

That's configuration drift.

---

# 9. Why drift is dangerous

You may believe:

> "Production matches our approved configuration."

But reality says:

```text
Production
   ↓
Manual modification
   ↓
Undocumented state
```

This makes incidents harder to reproduce.

---

# 10. Source control

Store IaC in version control.

Typical structure:

```text id="0m2hmb"
repo/
 ├── infrastructure/
 ├── modules/
 ├── environments/
 ├── pipelines/
 └── documentation/
```

Exact structure depends on organizational standards.

---

# 11. Pull requests

Infrastructure changes should go through the same engineering discipline as application code:

```text id="1q4q0v"
Developer
 ↓
Branch
 ↓
Change
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

---

# 12. Why code review matters

An IaC change can be as dangerous as application code.

For example:

```text id="9p7j2v"
Firewall change
Network change
Identity change
Deletion
Storage change
```

One incorrect setting can affect an entire production environment.

---

# 13. Modules

Don't copy-paste an entire SQL MI deployment for every environment.

Instead:

```text id="b0i0y7"
Reusable module
      ↓
Dev
Test
Stage
Prod
```

Parameterize environment-specific values.

---

# 14. Example conceptual module

```text id="c2q9e3"
SQL MI module
 ├── networking
 ├── identity
 ├── compute
 ├── security
 ├── monitoring
 └── backup/DR configuration
```

The module defines the standard platform.

---

# 15. Environment configuration

Keep environment-specific values separate.

Example:

```text id="s8k3p9"
dev
 └── smaller capacity

test
 └── representative capacity

prod
 └── production capacity
```

Don't hard-code production values into the reusable module.

---

# 16. Naming standards

Automation should enforce predictable names.

For example:

```text id="6e5g4t"
<org>-<app>-<env>-<region>-<resource>
```

The exact convention should come from your organization's naming standard.

Consistency helps:

```text
Search
Automation
Operations
Governance
Billing
```

---

# 17. Resource groups

A resource group is a logical management boundary.

Think about grouping resources based on:

```text id="7w5v1m"
Lifecycle
Ownership
Application
Environment
```

Avoid arbitrary organization.

---

# 18. Tagging

Automation should apply standard tags such as:

```text id="3h7f0s"
Environment
Application
Owner
CostCenter
Criticality
DataClassification
```

Tags improve governance and cost analysis.

---

# 19. Managed identity

Prefer managed identities where supported instead of storing credentials in scripts.

Conceptually:

```text id="6x5s3c"
Azure resource
      ↓
Managed Identity
      ↓
Azure resource/service
```

The platform manages the identity lifecycle.

---

# 20. Why avoid passwords in IaC?

This is dangerous:

```text id="7fz1w5"
adminPassword = "..."
```

because secrets can accidentally enter:

```text
Git
Logs
Pipeline output
Artifacts
State
```

---

# 21. Secrets management

Use an appropriate secrets-management system, such as:

Microsoft Azure

services designed for secure secret storage.

Conceptually:

```text
Pipeline
 ↓
Identity
 ↓
Secret store
 ↓
Temporary access
```

---

# 22. Never print secrets

A common automation mistake:

```text id="5v4jv0"
echo $PASSWORD
```

or logging command arguments containing credentials.

Your pipeline should treat secrets as sensitive values.

---

# 23. Azure Key Vault

For Azure environments, Azure Key Vault is commonly used to store secrets, keys, and certificates.

The principle is:

```text
Application/pipeline
       ↓
Identity
       ↓
Key Vault
       ↓
Secret
```

rather than:

```text
Application
       ↓
Hard-coded password
```

---

# 24. Network automation

SQL MI networking should also be defined as code where practical.

Think:

```text id="k4st2k"
VNet
 ↓
Subnet
 ↓
Delegation / required configuration
 ↓
NSG / routing considerations
 ↓
SQL MI
```

Networking mistakes can prevent deployment or connectivity.

---

# 25. Dependency management

Infrastructure isn't independent.

Example:

```text id="t5f2a1"
VNet
 ↓
Subnet
 ↓
SQL MI
 ↓
Database/application
```

Automation must understand these relationships.

---

# 26. Deployment ordering

Some resources must exist before others.

Conceptually:

```text id="j8r0vl"
Network
   ↓
Identity
   ↓
SQL MI
   ↓
Database configuration
   ↓
Monitoring
```

IaC tools usually model dependencies, but engineers still need to understand them.

---

# 27. Pipeline stages

A mature pipeline might be:

```text id="r5j3qy"
Validate
   ↓
Lint
   ↓
Plan / What-if
   ↓
Security checks
   ↓
Approval
   ↓
Deploy
   ↓
Post-deployment validation
```

---

# 28. Bicep validation

Before deployment, validate the template.

Conceptually:

```text id="2m9e4f"
Bicep
 ↓
Build
 ↓
Lint
 ↓
What-if
 ↓
Review
```

The goal is to identify problems before changing Azure.

---

# 29. Terraform plan

Terraform's equivalent safety mechanism is the plan.

Conceptually:

```text id="4at7wx"
Configuration
     +
Current state
     ↓
Terraform plan
     ↓
Proposed changes
```

Review the plan carefully before applying it.

---

# 30. Destructive changes

Be extremely careful with changes that can:

```text id="v2i4j6"
Delete resources
Destroy data
Change networking
Break connectivity
Change authentication
```

A production pipeline should include appropriate protections.

---

# 31. Approval gates

Production changes may require:

```text id="7ppnq3"
Automated validation
+
Human approval
```

especially for high-risk infrastructure changes.

---

# 32. Immutable vs mutable infrastructure

### Mutable

Change an existing resource repeatedly.

### Immutable

Create a new known-good resource/version and transition to it when architecture permits.

SQL MI is not simply an immutable compute artifact in the same way as a stateless VM/container image, so practical automation often combines declarative infrastructure management with controlled in-place changes.

---

# 33. Database schema deployment

Infrastructure provisioning and database schema deployment are related but different concerns.

Think:

```text id="0t1jfd"
Infrastructure
   ↓
SQL MI exists
   ↓
Database exists
   ↓
Schema migration
   ↓
Application deployment
```

---

# 34. Migration scripts

Schema changes should be:

```text id="h1y8z3"
Versioned
Reviewed
Repeatable where possible
Tested
Auditable
```

Examples:

```text id="4i2d9j"
001_create_customer
002_create_order
003_add_index
004_add_column
```

---

# 35. Database migrations in CI/CD

A mature pipeline may look like:

```text id="3d8g8j"
Application code
      +
Database migration
      ↓
Build
      ↓
Test
      ↓
Deploy
```

Database changes should not be an undocumented manual step.

---

# 36. Backward compatibility

Be careful with schema changes.

Suppose application version 1 expects:

```text id="s9z4yw"
Column A
```

and version 2 expects:

```text id="m1n0gq"
Column B
```

Immediately deleting A can break rolling deployments.

---

# 37. Expand-and-contract migration

A safer pattern:

```text id="n9m7c4"
Expand
 ↓
Add new structure
 ↓
Deploy compatible application
 ↓
Migrate data
 ↓
Switch usage
 ↓
Contract
 ↓
Remove old structure
```

This supports safer rolling deployments.

---

# 38. Example

Instead of:

```text id="9f5k8d"
DROP old_column
```

immediately:

```text id="4w0q7r"
ADD new_column
 ↓
Application writes both
 ↓
Backfill
 ↓
Application reads new
 ↓
Stop old writes
 ↓
Remove old column later
```

---

# 39. Automated validation

After deployment, don't simply say:

> "Pipeline succeeded."

Validate:

```text id="4x8n0a"
Resource exists
Connectivity works
Expected configuration exists
Monitoring works
Authentication works
Critical query works
Application health is normal
```

---

# 40. Smoke tests

A simple smoke test:

```text id="t7h4y0"
Connect
 ↓
Run safe query
 ↓
Verify expected result
 ↓
Return success/failure
```

---

# 41. Health checks

Post-deployment checks could include:

```text id="c7q4vl"
Connectivity
Database availability
Expected schema version
Critical stored procedure
Expected permissions
Monitoring ingestion
```

---

# 42. Drift detection

Regularly compare:

```text id="7f6f1j"
Desired configuration
        VS
Actual configuration
```

If different:

```text id="0j2j4z"
Investigate
 ↓
Approve
 ↓
Reconcile
```

---

# 43. Policy as Code

Azure Policy can enforce organizational standards.

Conceptually:

```text id="3n9n6k"
Resource
 ↓
Policy
 ↓
Compliant?
 ↙       ↘
Yes       No
```

---

# 44. Policy examples

Organizations may enforce requirements such as:

```text id="0y3f7g"
Approved regions
Required tags
Security configurations
Allowed resource types
Diagnostic settings
```

Exact policy definitions depend on organizational requirements.

---

# 45. Governance hierarchy

Think:

```text id="0v7f0v"
Organization
 ↓
Management groups
 ↓
Subscriptions
 ↓
Resource groups
 ↓
Resources
```

Policies can be applied at appropriate scopes.

---

# 46. Policy vs IaC

They solve different problems.

### IaC

> **How should we deploy this resource?**

### Policy

> **What configurations are allowed or required?**

Together:

```text id="5g7h8f"
IaC + Policy
     ↓
Consistent + governed infrastructure
```

---

# 47. Azure CLI automation

CLI is useful for:

```text id="z7n1h4"
Deployment
Inspection
Scripting
Operational tasks
CI/CD
```

Typical pattern:

```text id="t1p9n2"
Authenticate
 ↓
Select subscription
 ↓
Execute command
 ↓
Validate result
```

---

# 48. PowerShell automation

PowerShell is particularly useful when your organization already has a Windows/Microsoft automation ecosystem.

It can orchestrate:

```text id="m8f6v1"
Azure
SQL
Files
APIs
CI/CD
```

---

# 49. Idempotency

A good automation script should ideally be safe to run repeatedly.

Conceptually:

```text id="h2v6qa"
Run once → desired state
Run again → no unnecessary changes
```

This is a core IaC principle.

---

# 50. Bad automation

```text id="0y3jrm"
Create resource
 ↓
If exists → fail
```

Better:

```text id="4z0f1w"
Ensure desired state
```

---

# 51. Environment promotion

Don't manually rebuild production.

Promote the same tested infrastructure logic:

```text id="v7z4a8"
Dev
 ↓
Test
 ↓
Stage
 ↓
Production
```

with controlled environment-specific parameters.

---

# 52. Why promotion matters

If production uses completely different manual steps:

```text id="w2q7k9"
Dev ≠ Test ≠ Prod
```

then testing becomes less meaningful.

The closer the deployment process is, the lower the surprise factor.

---

# 53. GitOps concept

Git becomes the source of truth:

```text id="a6x4mz"
Git
 ↓
Approved configuration
 ↓
Automation
 ↓
Azure
```

Changes begin with code rather than portal clicks.

---

# 54. Disaster recovery benefit

IaC is extremely valuable during disaster recovery.

Instead of:

```text id="7x4h6j"
"What settings did production have?"
```

you have:

```text id="0n8x9e"
Version-controlled configuration
```

You still need to account for data recovery and service-specific recovery architecture, but infrastructure reproducibility is dramatically improved.

---

# 55. Documentation through code

Good IaC communicates:

```text id="r4u2l0"
What exists
How it is configured
Dependencies
Standards
Ownership
```

Code becomes part of operational documentation.

---

# 56. Security scanning

CI/CD should scan infrastructure changes for:

```text id="9u4h5k"
Public exposure
Weak configuration
Secrets
Overly broad permissions
Policy violations
```

This is an important DevSecOps layer.

---

# 57. Least privilege

Automation identities should have only the permissions they need.

Avoid:

```text id="2m5q3k"
Everyone → Owner
```

Prefer:

```text id="1f7d8s"
Pipeline identity
 ↓
Minimum required permissions
```

---

# 58. Separate deployment identities

A mature environment may use different identities for:

```text id="5n8j6f"
Development
Testing
Production
```

This reduces blast radius.

---

# 59. Pipeline secrets

Never embed credentials directly into:

```text id="w1v4k9"
YAML
Bicep
Terraform
Shell scripts
Git repositories
```

Use secure identity and secret mechanisms.

---

# 60. State management

Terraform introduces an important concept:

> **Terraform state**

State helps Terraform understand:

```text id="k0p7s4"
What it believes exists
How resources map to configuration
What changes are required
```

Protect the state carefully.

---

# 61. Terraform state security

State can contain sensitive information depending on resources/providers.

Therefore:

```text id="4w6x2e"
Protect state
Control access
Encrypt storage
Avoid public exposure
Use appropriate locking
```

---

# 62. Bicep vs Terraform

| Area               | Bicep                         | Terraform        |
| ------------------ | ----------------------------- | ---------------- |
| Azure focus        | Excellent                     | Excellent        |
| Multi-cloud        | Limited focus                 | Strong           |
| Azure-native model | Strong                        | Strong           |
| State management   | ARM handles deployment state  | Terraform state  |
| Ecosystem          | Azure-centric                 | Broad ecosystem  |
| Learning curve     | Often simpler for Azure teams | Broader concepts |

The correct choice depends on organizational standards and platform strategy.

---

# 63. Automation maturity

### Level 1

```text
Manual portal deployment
```

### Level 2

```text
Scripts
```

### Level 3

```text
IaC
```

### Level 4

```text
CI/CD + policy + validation
```

### Level 5

```text
Self-service platform
+
Governance
+
Observability
+
Automated remediation
```

---

# 64. Self-service SQL MI platform

At mature organizations, developers might request:

```text id="m7d3s1"
Application
Environment
Region
Criticality
Data classification
```

The platform automatically provisions the approved architecture.

Conceptually:

```text id="b7g4w5"
Developer request
       ↓
Approved template
       ↓
IaC pipeline
       ↓
SQL MI platform
       ↓
Monitoring + security + governance
```

---

# 65. Automated remediation

Not every issue requires a human.

Safe automation might:

```text id="1x4v9m"
Detect
 ↓
Validate condition
 ↓
Execute approved remediation
 ↓
Verify
 ↓
Notify
```

But destructive or ambiguous actions should generally require stronger controls.

---

# 66. Example remediation

A monitoring system detects:

```text id="e4t7q2"
Storage threshold approaching
```

Automation can:

```text id="y9v6f3"
Create incident
 ↓
Notify owner
 ↓
Trigger approved capacity workflow
```

Rather than blindly making destructive changes.

---

# 67. Production safety

Automation must have:

```text id="h6f0q4"
Guardrails
Approvals
Rollback strategy
Logging
Audit trail
Least privilege
```

Automation without safety can simply make mistakes faster.

---

# 68. Rollback

Every deployment should answer:

> **What happens if this fails?**

Possible strategies:

```text id="8j3c6s"
Revert configuration
Restore previous application version
Reverse schema migration where safe
Restore data where necessary
Fail forward
```

Database rollback is not always as simple as application rollback.

---

# 69. Database rollback warning

Suppose migration does:

```text id="j6q9s2"
DROP COLUMN
```

You cannot reliably undo that merely by saying:

```text id="a0p8c4"
ROLLBACK
```

after the deployment transaction is gone.

Database migration design must account for reversibility and data preservation.

---

# 70. Blue/green concepts

For some application architectures:

```text id="9r4n2e"
Blue = current
Green = new
```

Test the new environment, then shift traffic.

SQL MI-specific architecture and stateful data make this more complicated than stateless application deployments.

---

# 71. Canary deployment

Deploy to:

```text id="w4m8s0"
Small percentage
```

first.

Observe:

```text id="d6k9f2"
Latency
Errors
Database load
```

Then expand.

Again, applicability depends on the architecture.

---

# 72. Change management

Automation doesn't eliminate change management.

Instead:

```text id="k4w7q3"
Automation
+
Version control
+
Approval
+
Auditability
```

makes change management stronger.

---

# 73. Audit trail

For every important production change, know:

```text id="x7y2m5"
Who
What
When
Why
Which version
Which environment
Result
```

IaC and pipelines can provide much of this automatically.

---

# 74. Expert deployment workflow

```text id="2x7r4p"
Developer
   ↓
Git commit
   ↓
Pull request
   ↓
Code review
   ↓
Lint/security validation
   ↓
IaC plan / what-if
   ↓
Automated tests
   ↓
Approval
   ↓
Deployment
   ↓
Smoke tests
   ↓
Monitoring
   ↓
Promotion
```

---

# 75. Interview question

### "Why use IaC for SQL MI?"

> "IaC provides repeatability, consistency, version control, auditability, and faster recovery. It reduces configuration drift and allows infrastructure to be tested and promoted through environments rather than manually recreated."

---

# 76. Interview question

### "Bicep or Terraform?"

Strong answer:

> "I would choose based on organizational standards and requirements. Bicep is highly Azure-native and integrates naturally with ARM. Terraform provides a broader infrastructure ecosystem and is useful when managing multiple platforms. The important part is consistent, reviewed, secure automation rather than the tool alone."

---

# 77. Interview question

### "How do you prevent secrets from entering IaC?"

> "I avoid hard-coding secrets, use managed identities where possible, store required secrets in an appropriate secret-management service, use secure pipeline variables or references, restrict access through RBAC, and prevent secrets from appearing in logs or state."

---

# 78. Interview question

### "How do you handle configuration drift?"

> "I establish IaC as the desired state, detect differences through deployment plans or compliance mechanisms, investigate unauthorized changes, and reconcile approved differences back into code. I also use policy and access controls to reduce manual drift."

---

# 79. Interview question

### "How do you safely deploy database schema changes?"

> "I version migrations, test them against representative data, favor backward-compatible expand-and-contract patterns for rolling deployments, validate after deployment, and define a recovery strategy before making destructive changes."

---

# 80. Golden rule

> **Infrastructure should be reproducible, database changes should be versioned, secrets should be externalized, and production changes should be observable and reversible whenever practical.**

---

# 81. Final mental model

```text id="z6q3s8"
                 SOURCE CONTROL
                       ↓
                 INFRASTRUCTURE
                    AS CODE
                       ↓
              VALIDATION + SECURITY
                       ↓
                PLAN / WHAT-IF
                       ↓
                   APPROVAL
                       ↓
                  DEPLOYMENT
                       ↓
             POST-DEPLOYMENT TEST
                       ↓
                OBSERVABILITY
                       ↓
             DRIFT / COMPLIANCE
                       ↓
               CONTINUOUS DELIVERY
```

That is the foundation of **expert Azure SQL Managed Instance platform automation**.

---

## Next — Lesson 116: Azure SQL Managed Instance Security Architecture — Expert Level

We'll go deep into **Microsoft Entra authentication, managed identities, RBAC vs SQL permissions, least privilege, network isolation, private endpoints, firewall architecture, TDE, Always Encrypted, auditing, Defender for SQL, vulnerability assessment, Microsoft Purview concepts, secrets, key management, threat modeling, and production security architecture.**
