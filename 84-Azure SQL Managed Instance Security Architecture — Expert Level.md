# Lesson 116 — Azure SQL Managed Instance Security Architecture — Expert Level

The goal is to move from:

> **“SQL MI is secured.”**

to:

> **“We have a layered security architecture with explicit identity, network, data, monitoring, governance, and recovery controls.”**

---

# 1. Security is a system

Don't think:

```text
Firewall = security
```

Think:

```text
Identity
   +
Network
   +
Data protection
   +
Authorization
   +
Monitoring
   +
Governance
   +
Recovery
```

A weakness in one layer should not automatically compromise everything.

---

# 2. Defense in depth

A strong architecture looks like:

```text
Internet / Users
       ↓
Identity controls
       ↓
Network controls
       ↓
SQL authentication
       ↓
Database authorization
       ↓
Data protection
       ↓
Auditing / detection
```

---

# 3. Threat model first

Before choosing controls, ask:

```text
What are we protecting?
Who can access it?
How can they access it?
What happens if credentials are stolen?
What happens if an admin account is compromised?
What must be audited?
How quickly must we detect abuse?
```

---

# 4. CIA triad

Security architecture traditionally protects:

### Confidentiality

Prevent unauthorized disclosure.

### Integrity

Prevent unauthorized modification.

### Availability

Keep systems usable.

```text
Confidentiality
Integrity
Availability
```

All three matter for SQL MI.

---

# 5. Identity is the first major boundary

A mature architecture prefers strong identity-based access.

Conceptually:

```text
Human / Application
        ↓
Identity provider
        ↓
Authentication
        ↓
Authorization
        ↓
SQL MI
```

For Azure environments, Microsoft Entra ID is central to this model.

---

# 6. Authentication vs authorization

This distinction is critical.

### Authentication

> **Who are you?**

### Authorization

> **What are you allowed to do?**

Example:

```text
Alice
 ↓
Authenticated
 ↓
Database user
 ↓
SELECT only
```

---

# 7. Microsoft Entra authentication

Microsoft Entra authentication can reduce dependence on traditional SQL usernames/passwords.

Conceptually:

```text
User/Application
       ↓
Microsoft Entra
       ↓
Token
       ↓
SQL MI
```

This enables centralized identity management.

---

# 8. Why identity-based authentication is powerful

It allows organizations to leverage:

```text
MFA
Conditional Access
Identity lifecycle
Centralized access policies
Managed identities
RBAC
```

rather than independently managing database passwords everywhere.

---

# 9. Managed identities

Managed identities are especially useful for Azure-hosted applications.

Instead of:

```text
Application
 ↓
Stored SQL password
 ↓
SQL MI
```

prefer, where supported:

```text
Application
 ↓
Managed identity
 ↓
Microsoft Entra
 ↓
SQL MI
```

---

# 10. System-assigned vs user-assigned identity

Conceptually:

### System-assigned

Identity lifecycle follows the Azure resource.

```text
Resource created
 ↓
Identity created
```

Delete resource:

```text
Identity removed
```

### User-assigned

Identity exists independently and can be associated with multiple resources.

Choose based on lifecycle and reuse requirements.

---

# 11. Service principals vs managed identity

A service principal is an application identity.

A managed identity is an Azure-managed identity associated with an Azure resource.

For Azure-hosted workloads, managed identities often reduce credential-management complexity.

---

# 12. Least privilege

The fundamental rule:

> **Give identities only the permissions they actually need.**

Avoid:

```text
Application → sysadmin
```

when the application only requires:

```text
SELECT
INSERT
UPDATE
```

---

# 13. RBAC vs SQL permissions

This is a frequent interview topic.

### Azure RBAC

Controls access to Azure resources.

Examples conceptually:

```text
Who can manage SQL MI?
Who can modify Azure networking?
Who can change Azure configuration?
```

### SQL authorization

Controls access inside SQL.

Examples:

```text
Who can SELECT?
Who can EXECUTE?
Who can ALTER?
Who can CREATE?
```

These are different authorization layers.

---

# 14. Example

An engineer may have Azure permissions to inspect an SQL MI resource but not necessarily:

```text
SELECT sensitive_customer_data
```

Conversely, a database user may have:

```text
SELECT
```

inside SQL but no ability to modify the Azure infrastructure.

This separation is desirable.

---

# 15. Privileged identities

Administrative access should be tightly controlled.

Consider:

```text
Privileged identity
 ↓
Strong authentication
 ↓
Time-limited access
 ↓
Auditing
```

Avoid permanent broad access whenever practical.

---

# 16. Just-in-time administration

For sensitive administrative operations:

```text
Normal user
 ↓
Request elevated access
 ↓
Approval / policy
 ↓
Temporary privilege
 ↓
Perform operation
 ↓
Privilege expires
```

This reduces standing privilege.

---

# 17. Break-glass accounts

Organizations often maintain emergency administrative access.

But:

```text
Break-glass
≠
Daily account
```

It should be:

```text
Highly protected
Rarely used
Strongly monitored
Regularly tested
```

---

# 18. Network security

Even perfect identity controls don't mean:

> "Expose SQL MI everywhere."

Network access should be restricted.

Think:

```text
Application network
       ↓
Private connectivity
       ↓
SQL MI
```

---

# 19. Network isolation

A common architecture places SQL MI inside controlled Azure networking.

Conceptually:

```text
                 Internet
                    X
                    |
              Application
                    |
                  VNet
                    |
                 Subnet
                    |
                  SQL MI
```

The exact topology depends on application and connectivity requirements.

---

# 20. Private connectivity

For Azure PaaS services, private networking patterns can reduce public exposure.

For SQL MI specifically, understand the difference between:

```text
SQL MI's own VNet/subnet deployment model
```

and:

```text
Private Endpoint
```

They are not interchangeable concepts.

---

# 21. Private endpoint

A private endpoint provides a private IP-based connection to a supported Azure service.

Conceptually:

```text
Client VNet
    ↓
Private endpoint
    ↓
Azure service
```

Whether and how it applies depends on the specific service architecture.

---

# 22. Don't blindly copy architecture patterns

A common mistake is:

> "Every Azure database needs a private endpoint."

Architecture must follow the actual connectivity model of the service.

For SQL MI, understand its native network placement and supported connectivity options before designing additional private connectivity.

---

# 23. Firewall philosophy

The goal isn't:

```text
Allow everything
```

It is:

```text
Allow only required sources
```

Prefer:

```text
Known network
Known identity
Known port
Known purpose
```

---

# 24. Network Security Groups

NSGs can help control network traffic at supported Azure networking layers.

But don't assume:

> "NSG = SQL authorization."

They solve different problems.

```text
NSG
 ↓
Network traffic

SQL permissions
 ↓
Database operations
```

---

# 25. Segmentation

Separate workloads where appropriate:

```text
Web
 ↓
Application
 ↓
Database
```

Avoid unnecessary lateral connectivity.

If the application tier is compromised, segmentation limits what the attacker can reach.

---

# 26. Egress control

Security isn't only inbound.

Ask:

> **Where can SQL MI-related workloads communicate outward?**

Control and monitor outbound connectivity where architecture and service capabilities permit.

---

# 27. Encryption in transit

Data moving across networks should be protected using appropriate encrypted protocols.

Conceptually:

```text
Application
   ↓
Encrypted connection
   ↓
SQL MI
```

Transport encryption protects against network interception.

---

# 28. Encryption at rest

Encryption at rest protects stored data.

A common baseline is:

```text
Database files
Backup data
Storage
```

protected through platform-supported encryption mechanisms.

---

# 29. Transparent Data Encryption

TDE protects database files at rest.

Conceptually:

```text
Application
   ↓
Plain logical data
   ↓
SQL engine
   ↓
Encrypted storage
```

The application doesn't need to manually encrypt every query result just to benefit from TDE.

---

# 30. TDE limitation

TDE primarily protects:

> **Data at rest**

It does not automatically solve:

```text
Authorized user reading data
Application compromise
Incorrect permissions
Data exfiltration
```

That's why defense in depth matters.

---

# 31. Customer-managed keys

Organizations with stronger key-management requirements may use customer-managed key patterns where supported.

Conceptually:

```text
Customer-controlled key
        ↓
Azure key-management service
        ↓
Database encryption
```

This can provide additional control over cryptographic key lifecycle.

---

# 32. Key rotation

Key management includes:

```text
Creation
Storage
Access
Rotation
Revocation
Recovery
Auditing
```

A key isn't secure merely because it exists.

---

# 33. Always Encrypted

Always Encrypted addresses a different problem.

The goal is to protect sensitive column data so that plaintext is not exposed to the database engine in the same way as ordinary encrypted-at-rest storage.

Conceptually:

```text
Application
 ↓
Encryption/decryption
 ↓
Encrypted column data
 ↓
SQL MI
```

---

# 34. TDE vs Always Encrypted

| Feature                               | TDE                | Always Encrypted                         |
| ------------------------------------- | ------------------ | ---------------------------------------- |
| Protects storage                      | Yes                | Yes, but different purpose               |
| Transparent to application            | Mostly             | Requires client/application support      |
| Protects against privileged DB access | Limited            | Stronger for supported sensitive columns |
| Main goal                             | At-rest encryption | Sensitive data confidentiality           |

---

# 35. Dynamic Data Masking

Data masking can reduce exposure of sensitive values to certain users.

Example:

```text
Actual:
123-45-6789

Masked:
XXX-XX-6789
```

But remember:

> Masking is not the same as encryption.

It should not be treated as a complete security boundary.

---

# 36. Row-level security

Row-level security controls which rows a user can access.

Example:

```text
Tenant A → Tenant A rows
Tenant B → Tenant B rows
```

This is useful in multi-tenant architectures.

---

# 37. Column-level permissions

Instead of granting broad table access:

```text
SELECT on Customer
```

you may need more granular access depending on the design.

Sensitive columns deserve particular attention.

---

# 38. Views as security boundaries

A common pattern:

```text
Application
 ↓
Secure view
 ↓
Underlying tables
```

The application gets only the columns and rows it needs.

---

# 39. Stored procedures as security boundaries

Another pattern:

```text
Application
 ↓
EXECUTE procedure
 ↓
Controlled data access
```

This can reduce direct table permissions when designed correctly.

---

# 40. Don't rely on obscurity

Bad security:

```text
"Users won't know the table name."
```

Good security:

```text
Explicit permissions
+
Authentication
+
Authorization
+
Network controls
```

---

# 41. SQL injection

Database security includes application security.

Bad:

```text
"SELECT ... WHERE name = '" + userInput + "'"
```

Prefer parameterized queries.

The database cannot compensate for fundamentally unsafe application query construction.

---

# 42. Application identity separation

Don't let every application use one shared database identity.

Instead:

```text
App A → Identity A
App B → Identity B
Reporting → Reporting identity
Admin → Admin identity
```

This improves:

```text
Auditing
Least privilege
Incident investigation
Revocation
```

---

# 43. Shared accounts

Shared accounts are dangerous because:

```text
Alice
Bob
Service
Automation
```

may all appear as:

```text
dbadmin
```

You lose accountability.

---

# 44. Auditing

Auditing answers:

> **Who did what, when, and from where?**

Capture appropriate events such as:

```text
Authentication
Data access
Schema changes
Permission changes
Administrative activity
```

The exact audit categories depend on the service and configuration.

---

# 45. Audit retention

Audit data should have:

```text
Retention policy
Access controls
Integrity protection
Monitoring
```

Don't store security logs somewhere administrators can casually modify without trace.

---

# 46. Security monitoring

Security telemetry should feed centralized monitoring where appropriate:

```text
SQL MI
 ↓
Audit / diagnostic telemetry
 ↓
Central monitoring
 ↓
Detection
 ↓
Incident response
```

---

# 47. Microsoft Defender for SQL

Microsoft Defender for Cloud provides security capabilities for Azure resources, including SQL-related protection and recommendations through its Defender plans.

Use it as an additional security layer, not as a replacement for identity and authorization design.

---

# 48. Vulnerability management

Security posture should be continuously assessed.

Look for:

```text
Weak configuration
Missing controls
Excessive privileges
Potential vulnerabilities
Unsupported components
```

Then:

```text
Detect
 ↓
Prioritize
 ↓
Remediate
 ↓
Verify
```

---

# 49. Security recommendations

Not every recommendation has equal importance.

Prioritize based on:

```text
Exploitability
Data sensitivity
Exposure
Business impact
Likelihood
```

---

# 50. Data classification

Not all data deserves identical controls.

Example:

```text
Public
Internal
Confidential
Highly sensitive
```

The more sensitive the data:

```text
More restrictive access
+
Stronger encryption
+
More monitoring
+
Stricter retention
```

---

# 51. Data discovery

Know where sensitive data exists.

You can't protect:

> **What you don't know exists.**

Maintain an understanding of:

```text
PII
Financial data
Credentials
Health-related data
Secrets
Business-critical data
```

---

# 52. Microsoft Purview

Microsoft Purview can help organizations with data governance, discovery, classification, and compliance workflows across supported data sources.

The exact capabilities depend on the services and connectors being used.

---

# 53. Compliance is not security

A system can be:

```text
Compliant
```

yet still have poor security architecture.

Compliance provides requirements and evidence.

Security engineering provides:

```text
Protection
Detection
Response
Recovery
```

---

# 54. Threat scenario: stolen application credentials

Suppose an attacker obtains an application credential.

Defense in depth:

```text
Identity
 ↓
Least privilege
 ↓
Network restrictions
 ↓
Database permissions
 ↓
Audit
 ↓
Anomaly detection
```

The attacker should not automatically gain unrestricted database access.

---

# 55. Threat scenario: compromised DBA account

Suppose a privileged identity is compromised.

Controls include:

```text
MFA
Privileged access controls
JIT access
Audit logging
Separation of duties
Alerting
Break-glass controls
```

---

# 56. Threat scenario: ransomware

Potential protections:

```text
Identity controls
Network segmentation
Immutable/secured backups where applicable
Monitoring
Recovery testing
Least privilege
```

The key question is:

> **Can we recover even if production data is compromised?**

---

# 57. Backup security

Backups are part of the security boundary.

Consider:

```text
Who can access backups?
Can they delete them?
How long are they retained?
Can recovery be tested?
```

---

# 58. Recovery testing

A backup that has never been restored is an assumption.

Test:

```text
Backup
 ↓
Restore
 ↓
Validate database
 ↓
Validate application
```

Measure actual recovery time.

---

# 59. RPO and RTO

### RPO

How much data loss is acceptable?

```text
RPO = 15 minutes
```

means losing up to that amount of recent data may be within the defined objective.

### RTO

How quickly must service recover?

```text
RTO = 1 hour
```

means the recovery objective is one hour.

---

# 60. Security + availability

Security decisions can affect availability.

Example:

```text
Overly restrictive firewall
 ↓
Application cannot connect
```

Therefore security architecture must be tested operationally.

---

# 61. Secrets rotation

Credentials should have controlled lifecycle:

```text
Create
 ↓
Use
 ↓
Rotate
 ↓
Revoke
```

Managed identity can eliminate some credential-rotation requirements entirely.

---

# 62. Certificate management

Where certificates are used:

```text
Issuance
 ↓
Deployment
 ↓
Expiration monitoring
 ↓
Rotation
```

Automate expiration detection whenever possible.

---

# 63. Security baselines

Define a standard SQL MI baseline.

Example:

```text
Authentication → approved identity model
Networking → approved topology
Logging → enabled
Auditing → enabled where required
Encryption → required
Tags → required
RBAC → least privilege
Backup → configured
Monitoring → configured
```

Then automate compliance.

---

# 64. Policy enforcement

Combine:

```text
IaC
+
Azure Policy
+
RBAC
+
Monitoring
```

to create preventive and detective controls.

---

# 65. Preventive vs detective controls

### Preventive

Stop the bad action.

```text
RBAC
Network restrictions
Policy
```

### Detective

Detect the bad action.

```text
Audit
Alerts
Threat detection
```

### Corrective

Recover from it.

```text
Rollback
Restore
Credential revocation
Incident response
```

A mature system uses all three.

---

# 66. Security incident workflow

```text
Detect
 ↓
Triage
 ↓
Contain
 ↓
Investigate
 ↓
Eradicate
 ↓
Recover
 ↓
Validate
 ↓
Lessons learned
```

---

# 67. Example security incident

Suppose unusual database access is detected.

```text
Alert
 ↓
Identify identity
 ↓
Identify source
 ↓
Review query/activity
 ↓
Check recent privilege changes
 ↓
Contain account
 ↓
Investigate
 ↓
Restore normal access
```

Don't immediately destroy evidence.

---

# 68. Security observability

Useful signals include:

```text
Failed logins
Successful privileged logins
Permission changes
Schema changes
Unexpected query patterns
Unusual network sources
Audit anomalies
```

---

# 69. Identity lifecycle

When an employee leaves:

```text
HR event
 ↓
Identity disabled
 ↓
Database access removed
 ↓
Privileged access revoked
 ↓
Tokens/session access addressed
```

Central identity makes this much easier.

---

# 70. Service account lifecycle

For application identities:

```text
Create
 ↓
Assign minimum permissions
 ↓
Monitor
 ↓
Rotate credentials if applicable
 ↓
Disable when obsolete
```

Never allow abandoned service accounts to accumulate.

---

# 71. Separation of duties

For high-risk operations:

```text
Developer
≠
Production DBA
≠
Security approver
```

The exact model depends on organizational structure, but separating conflicting responsibilities reduces insider and accidental risk.

---

# 72. Security architecture example

```text
                    USERS
                      |
                 Entra ID
                      |
              Authentication
                      |
              Conditional Access
                      |
                  Network
                      |
              +-------+-------+
              |               |
           App Tier       Admin Access
              |               |
              +-------+-------+
                      |
                   SQL MI
                      |
        +-------------+-------------+
        |             |             |
       RBAC      SQL permissions  RLS
        |             |             |
        +-------------+-------------+
                      |
              Encryption / TDE
                      |
                 Audit Logs
                      |
              Security Monitoring
                      |
               Incident Response
```

---

# 73. Expert interview question

### "How would you secure SQL MI?"

A strong answer:

> "I'd use defense in depth: Microsoft Entra-based identity where appropriate, managed identities for Azure workloads, least-privilege Azure RBAC and SQL permissions, controlled network access, encryption in transit and at rest, stronger column-level protection for highly sensitive data where required, auditing and threat detection, centralized monitoring, secure backups, policy enforcement, and regular access and recovery reviews."

---

# 74. Interview question

### "What's the difference between RBAC and SQL permissions?"

> "Azure RBAC governs access to Azure resources and management operations, while SQL permissions govern operations inside the database. Having permission to manage an Azure SQL MI resource doesn't automatically mean having permission to read database tables."

---

# 75. Interview question

### "TDE vs Always Encrypted?"

> "TDE primarily protects database storage at rest. Always Encrypted is designed for stronger protection of selected sensitive data from access by the database service itself, using client-side cryptographic operations. They solve different threat models and can complement each other."

---

# 76. Interview question

### "How would you protect an application from SQL MI?"

> "I'd use a dedicated application identity, managed identity where practical, least-privilege database permissions, parameterized queries, restricted network connectivity, encrypted connections, auditing, and monitoring for anomalous behavior."

---

# 77. Interview question

### "What happens if an application identity is compromised?"

Strong answer:

> "The blast radius should be limited by least privilege and network restrictions. I'd investigate the identity's activity, revoke or disable access as appropriate, review audit logs, rotate any affected credentials, and determine whether data or permissions were accessed or modified."

---

# 78. Security maturity model

### Level 1

```text
Password + firewall
```

### Level 2

```text
Encryption + RBAC
```

### Level 3

```text
Entra + managed identity + auditing
```

### Level 4

```text
Policy + threat detection + centralized monitoring
```

### Level 5

```text
Zero-trust principles
+
Automated governance
+
Continuous risk assessment
+
Tested incident response
+
Tested recovery
```

---

# 79. Golden rule

> **Never depend on a single security control.**

If:

```text
Identity fails
```

network controls should still help.

If:

```text
Network control fails
```

database authorization should still help.

If:

```text
Authorization is bypassed
```

auditing and detection should still provide visibility.

---

# 80. Final mental model

```text
                 IDENTITY
                    ↓
              AUTHENTICATION
                    ↓
                NETWORK
                    ↓
              AUTHORIZATION
                    ↓
             DATA PROTECTION
                    ↓
                AUDITING
                    ↓
              THREAT DETECTION
                    ↓
              INCIDENT RESPONSE
                    ↓
                 RECOVERY
                    ↓
             CONTINUOUS REVIEW
```

That is the foundation of **expert Azure SQL Managed Instance security architecture**.

---

## Next — Lesson 117: Azure SQL Managed Instance High Availability, Disaster Recovery & Business Continuity — Expert Level

We'll cover **HA vs DR, zone redundancy, failover groups, geo-replication concepts, RPO/RTO design, regional failure, backup-based recovery, DR runbooks, DNS/application failover, planned vs unplanned failover, data consistency, testing, and designing a production-grade multi-region SQL MI architecture.**
