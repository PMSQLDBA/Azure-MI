# Lesson 112 — Azure SQL Managed Instance Security — Advanced

This lesson moves from basic security configuration into **defense-in-depth, identity architecture, least privilege, encryption, auditing, privileged access, and security operations**.

The senior DBA mindset is:

> **Security isn't one feature. It's a chain of controls that must remain secure from identity → network → database → data → monitoring → recovery.**

---

# 1. SQL MI security layers

Think in layers:

```text
Internet / User
      ↓
Identity
      ↓
Network
      ↓
SQL MI
      ↓
Database
      ↓
Objects
      ↓
Data
      ↓
Audit / Monitoring
```

A weakness at any layer can undermine the whole system.

---

# 2. Defense in depth

Don't rely on one control.

For example:

```text id="m3x8p7"
Identity
+
Network isolation
+
Authentication
+
Authorization
+
Encryption
+
Auditing
+
Monitoring
```

If one control fails, another still provides protection.

---

# 3. Authentication vs authorization

This distinction is fundamental.

### Authentication

> **Who are you?**

### Authorization

> **What are you allowed to do?**

Example:

```text id="q7n2c5"
User authenticates
      ↓
Identity verified
      ↓
Permissions evaluated
      ↓
Access allowed/denied
```

---

# 4. Microsoft Entra authentication

Azure SQL workloads can integrate with Microsoft Entra identity.

This allows organizations to use centralized identity controls rather than relying exclusively on SQL authentication.

Conceptually:

```text id="v8m4p1"
User / Application
       ↓
Microsoft Entra
       ↓
SQL MI
```

---

# 5. Why Entra authentication matters

It can provide:

```text id="x5c9m3"
Centralized identity
MFA
Conditional Access
Identity lifecycle management
Centralized auditing
Reduced password dependency
```

---

# 6. SQL authentication

SQL authentication uses a SQL-specific login and password.

Conceptually:

```text id="r6p2n8"
Username
+
Password
      ↓
SQL MI
```

It remains useful in appropriate scenarios, but credentials must be protected carefully.

---

# 7. Password problem

Long-lived passwords create risks:

```text id="m9x3v7"
Secret leakage
Password reuse
Rotation overhead
Credential sprawl
```

Where supported and appropriate, identity-based authentication can reduce these risks.

---

# 8. Managed identities

Managed identities allow Azure resources to authenticate without storing traditional credentials in application configuration.

Conceptually:

```text id="k4n8c2"
Azure workload
      ↓
Managed Identity
      ↓
Microsoft Entra
      ↓
SQL MI
```

---

# 9. Why managed identity is powerful

Instead of:

```text id="p7m3x9"
Application
 ↓
Connection string
 ↓
Password
```

you can move toward:

```text id="x2v8q5"
Application
 ↓
Identity
 ↓
Database permissions
```

This reduces secret management complexity.

---

# 10. Least privilege

The principle:

> **Give identities only the permissions they actually need.**

Bad:

```text id="n5c8m2"
Application → sysadmin
```

Better:

```text id="r3x7p9"
Application
 ↓
Specific database role
 ↓
Required objects only
```

---

# 11. RBAC vs SQL permissions

This is a common interview topic.

### Azure RBAC

Controls access to **Azure resources/control plane**.

Example concepts:

```text id="m6v2k8"
Manage SQL MI resource
Read Azure configuration
Modify Azure resources
```

### SQL permissions

Control access **inside SQL Server/database/data plane**.

Example:

```text id="q9p3x5"
SELECT
INSERT
UPDATE
EXECUTE
ALTER
```

---

# 12. Control plane vs data plane

Think:

```text id="v4m8n2"
Azure Resource
      |
Control Plane
      ↓
Resource management
```

versus:

```text id="c7x3p6"
Database
      |
Data Plane
      ↓
Queries / data
```

These are different security domains.

---

# 13. Example

An Azure administrator may have permission to manage the SQL MI resource but should not automatically be granted unrestricted access to every row of application data.

That separation is important.

---

# 14. Database roles

Use roles to group permissions.

Conceptually:

```text id="x8m4q2"
Role
 ↓
Permissions
 ↓
Users / applications
```

Instead of assigning permissions individually to every user.

---

# 15. Role-based design

Example:

```text id="p5n9c3"
AppReadRole
AppWriteRole
ReportingRole
DBAAdminRole
```

Then:

```text id="m7x2v8"
Application → AppReadRole
Application → AppWriteRole
```

depending on its responsibilities.

---

# 16. Avoid direct privilege sprawl

Bad:

```text id="k3m8p6"
User A → SELECT
User A → UPDATE
User B → SELECT
User B → UPDATE
User C → SELECT
...
```

Better:

```text id="q6v3n9"
Role
 ↓
Permissions
 ↓
Members
```

---

# 17. Privileged access

High-privilege accounts should be tightly controlled.

Examples:

```text id="r4x8m2"
sysadmin
db_owner
security administration
Azure resource administration
```

Use privileged access only when necessary.

---

# 18. Break-glass access

Organizations may maintain emergency access for severe incidents.

That access should be:

```text id="n8p3c5"
Rarely used
Strongly protected
Audited
Reviewed
```

It should not become the normal DBA account.

---

# 19. Privileged Identity Management

Where applicable, privileged access can be made:

```text id="m2x7v9"
Just-in-time
Approval-based
Time-limited
Audited
```

This reduces standing administrative privileges.

---

# 20. Network security

Even a perfectly configured database can be exposed unnecessarily through poor network architecture.

Defense starts with:

```text id="p8c4m1"
Network isolation
```

---

# 21. Network segmentation

Separate:

```text id="x3n9k5"
Application subnet
Database subnet
Management access
```

according to architecture and security requirements.

---

# 22. Private connectivity

For sensitive workloads, prefer private network paths where appropriate.

Conceptually:

```text id="q7m2v8"
Application
    |
Private network
    |
SQL MI
```

rather than exposing database traffic unnecessarily to public paths.

---

# 23. Network security groups

NSGs can control network traffic at supported network boundaries.

Principle:

```text id="r5x8n3"
Allow only required traffic
```

Don't create broad rules such as:

```text id="m9c4p2"
Any source
Any destination
Any port
```

unless there is an exceptional, justified requirement.

---

# 24. Firewall mindset

Ask:

```text id="v6p3x9"
Who needs access?
From where?
To what?
On which port?
For what purpose?
```

Security rules should answer those questions explicitly.

---

# 25. Encryption in transit

Data moving between application and database should be protected using secure transport.

Conceptually:

```text id="k2m8q4"
Application
   |
Encrypted connection
   |
SQL MI
```

---

# 26. Encryption at rest

Data stored on disk should be protected.

A key Azure SQL control is:

> **Transparent Data Encryption (TDE)**

TDE encrypts database files at rest without requiring application-level encryption logic.

---

# 27. TDE concept

Without encryption:

```text id="x7c3m5"
Database files
   ↓
Readable storage
```

With TDE:

```text id="p9n2v8"
Database files
   ↓
Encrypted
```

Applications continue to query normally.

---

# 28. TDE vs Always Encrypted

Important distinction.

### TDE

Protects:

```text id="m4x8q2"
Data at rest
```

### Always Encrypted

Designed to protect sensitive column values from certain database-side access scenarios, with encryption/decryption handled by authorized clients.

---

# 29. Why TDE isn't enough for every threat

TDE helps if someone obtains:

```text id="n6p3c9"
Underlying database storage
```

But it doesn't prevent an authorized database account from running:

```text id="q8m2v5"
SELECT SensitiveColumn
```

The database engine must decrypt data for normal database operations.

---

# 30. Always Encrypted concept

Conceptually:

```text id="r3x7m9"
Application
   |
Encrypt/decrypt
   |
Encrypted column
   |
SQL MI
```

The database stores ciphertext for protected values.

---

# 31. When to consider stronger column-level protection

Examples:

```text id="v5k8n2"
Credit card information
National identifiers
Highly sensitive personal information
```

The exact architecture depends on application requirements and supported encryption features.

---

# 32. Key management

Encryption introduces another question:

> **Who controls the keys?**

Depending on the security architecture, keys may be managed using Azure-supported key-management services and customer-controlled keys.

---

# 33. Key rotation

Keys should have:

```text id="m7p2x6"
Ownership
Rotation process
Access controls
Monitoring
Recovery strategy
```

Key management is part of database availability too.

---

# 34. Key loss scenario

If encryption depends on a key that becomes unavailable:

```text id="x4c8m1"
Database
   ↓
Encryption key unavailable
   ↓
Access problems
```

Therefore security controls must be included in DR planning.

---

# 35. Auditing

Auditing answers:

> **What happened?**

Examples:

```text id="q6n3v8"
Who connected?
What operation occurred?
When?
From where?
What object was accessed?
```

---

# 36. Auditing vs monitoring

Monitoring:

```text id="p8m4x2"
CPU
Latency
Errors
```

Auditing:

```text id="r5c7n9"
Who accessed data?
Who changed permissions?
Who executed sensitive operations?
```

Different purposes.

---

# 37. Security monitoring

Security telemetry can identify:

```text id="m3x8q5"
Repeated login failures
Unusual access
Privilege escalation
Suspicious queries
Unexpected network behavior
```

---

# 38. Defender for SQL

Azure provides security capabilities through Microsoft Defender for SQL.

Conceptually:

```text id="v9k2m6"
SQL MI
 ↓
Security assessment / detection
 ↓
Alerts / recommendations
```

Use the capabilities appropriate to your organization's security posture and licensing.

---

# 39. Vulnerability management

A mature security program doesn't stop at detecting attacks.

It also identifies:

```text id="x7p3n8"
Weak configuration
Security recommendations
Potential vulnerabilities
Unsupported components
Excessive permissions
```

---

# 40. Security baseline

Create a standard:

```text id="c4m8v2"
Authentication
Network
Encryption
Auditing
Logging
RBAC
SQL permissions
Monitoring
```

Then apply it consistently.

---

# 41. Security as code

Your baseline can be represented through:

```text id="n6x2p9"
IaC
Policies
Pipeline checks
Automated tests
```

This prevents security from depending entirely on manual DBA memory.

---

# 42. Example security pipeline

```text id="m8q3v5"
Infrastructure change
       ↓
Security policy check
       ↓
Identity check
       ↓
Network check
       ↓
Encryption check
       ↓
Monitoring check
       ↓
Deploy
```

---

# 43. Data classification

Not all data has equal sensitivity.

Example:

```text id="r7m2c8"
Public
Internal
Confidential
Highly sensitive
```

Controls should increase with sensitivity.

---

# 44. Sensitive data discovery

Know where sensitive data exists:

```text id="x5n9p3"
Customer identifiers
Financial data
Health-related data
Credentials/secrets
Business secrets
```

Then apply appropriate controls.

---

# 45. Don't store secrets in SQL

Avoid using database tables as an informal secret vault for:

```text id="k3v8m1"
API passwords
Private keys
Cloud credentials
Tokens
```

Use an appropriate secret-management system.

---

# 46. Service principals and applications

Application identities should have:

```text id="p6m4x9"
Dedicated identity
Least privilege
No shared credentials
Auditable ownership
```

Avoid:

```text id="q2n7c5"
One account
   ↓
Every application
```

because compromise becomes much more damaging.

---

# 47. Shared accounts

Shared DBA accounts make auditing difficult.

Instead:

```text id="v8x3m6"
Individual identity
      ↓
Privileged role
```

Then logs can identify the responsible operator.

---

# 48. Separation of duties

For highly sensitive environments:

```text id="m5p9k2"
Developer
DBA
Security
Operations
```

may have deliberately different privileges.

This reduces the chance that one compromised identity can perform every action.

---

# 49. Production DBA access

A mature pattern:

```text id="x4c7n8"
Normal account
   ↓
Routine work

Privileged elevation
   ↓
Administrative task
```

rather than using permanent high privilege.

---

# 50. Authentication logging

Monitor:

```text id="r9m3v5"
Successful logins
Failed logins
Unexpected identities
Unusual locations
Unusual times
```

A sudden spike in failures can indicate:

```text id="p7x2m6"
Misconfiguration
Credential expiration
Brute-force attempt
Application failure
```

---

# 51. SQL injection

Database security also depends on applications.

Bad:

```text id="n8q4c2"
SQL string + raw user input
```

Better:

```text id="m3v7x9"
Parameterized queries
```

Database administrators should understand the application-side risks because SQL MI security doesn't exist in isolation.

---

# 52. Stored procedures aren't automatically secure

Using stored procedures can help centralize access patterns, but poorly designed procedures can still expose data or execute unsafe dynamic SQL.

Security must be evaluated at the complete query boundary.

---

# 53. Dynamic SQL

Dynamic SQL requires careful handling.

Risks include:

```text id="x6m2p8"
SQL injection
Unexpected object access
Privilege abuse
```

Use safe parameterization and strict input validation.

---

# 54. Auditing privileged changes

Especially monitor:

```text id="c8v3n5"
CREATE LOGIN
ALTER LOGIN
CREATE USER
ALTER ROLE
GRANT
DENY
REVOKE
```

These can change the security boundary.

---

# 55. Permission review

Regularly ask:

```text id="q5m8x2"
Who has access?
Why?
Do they still need it?
Is privilege excessive?
```

This is access governance.

---

# 56. Dormant accounts

An account that hasn't been used for months may be:

```text id="m7n3c9"
Unnecessary
Forgotten
Compromised
```

Establish an appropriate lifecycle:

```text id="v2x8p5"
Create
→ Review
→ Use
→ Revalidate
→ Disable/remove
```

---

# 57. Security incident response

If suspicious activity occurs:

```text id="r4m9k3"
Detect
 ↓
Contain
 ↓
Investigate
 ↓
Eradicate
 ↓
Recover
 ↓
Learn
```

DBAs participate heavily in the investigation and recovery stages.

---

# 58. Example security incident

Suppose:

```text id="x8p2m6"
Failed login rate spikes
```

Then:

```text id="q3n7v5"
Unexpected account
+
Unusual source
+
Successful login
```

You should correlate:

```text id="m6c4x8"
Identity logs
+
SQL audit
+
Network telemetry
+
Application telemetry
```

---

# 59. Security and observability

The same observability principles from the previous lesson apply.

You need:

```text id="v9m2k5"
Metrics
Logs
Audit
Alerts
Investigation
```

Security without visibility is difficult to operate.

---

# 60. Security alert design

Bad:

```text id="p4x8n3"
Every failed login → Critical
```

This creates noise.

Better:

```text id="k7m3q9"
Repeated failures
+
Unexpected identity
+
Unusual source
```

Use context.

---

# 61. Network + identity defense

Suppose an attacker steals credentials.

Network isolation can still limit:

```text id="x2v6m8"
Where they can connect from
```

Suppose a network rule is misconfigured.

Strong identity controls can still limit:

```text id="r5p9c3"
What they can do
```

This is defense in depth.

---

# 62. Zero Trust mindset

Don't assume:

```text id="n8m4x2"
"Internal network = trusted."
```

Instead evaluate:

```text id="c6p3v7"
Identity
Device/workload
Context
Network
Resource
Permission
```

---

# 63. Security architecture

A mature SQL MI environment might look like:

```text id="q9x2m5"
              USER / APP
                   |
              IDENTITY
                   |
            NETWORK CONTROLS
                   |
                SQL MI
                   |
       +-----------+-----------+
       |           |           |
    DATABASE    ENCRYPTION   ROLES
       |           |           |
       +-----------+-----------+
                   |
             AUDIT / LOGS
                   |
           SECURITY MONITORING
                   |
               RESPONSE
```

---

# 64. Security checklist

```text id="m5v8c2"
IDENTITY
☐ Entra authentication where appropriate
☐ Managed identities where appropriate
☐ MFA for human access
☐ Least privilege
☐ No shared privileged accounts

AUTHORIZATION
☐ RBAC reviewed
☐ SQL roles reviewed
☐ Excess permissions removed
☐ Privileged access controlled

NETWORK
☐ Private connectivity where appropriate
☐ Network segmentation
☐ Minimal inbound access
☐ NSG/network rules reviewed

ENCRYPTION
☐ Encryption at rest
☐ Encryption in transit
☐ Key management
☐ Key rotation strategy

DATA
☐ Sensitive data identified
☐ Appropriate column/data protection
☐ Secrets not stored improperly

AUDITING
☐ Database auditing
☐ Security logs
☐ Privilege-change monitoring
☐ Authentication monitoring

OPERATIONS
☐ Vulnerability assessment
☐ Security alerts
☐ Incident response
☐ Access reviews
☐ DR security validation
```

---

# 65. Interview question

### "What's the difference between RBAC and SQL permissions?"

> "Azure RBAC primarily controls access to Azure resources and management operations, while SQL permissions control access inside the database engine, such as schemas, tables, procedures, and data. They represent different authorization layers."

---

# 66. Interview question

### "Why use managed identities?"

> "Managed identities allow supported Azure workloads to authenticate using Azure-managed identities instead of embedding long-lived credentials in application configuration. This reduces secret-management risk and supports centralized identity governance."

---

# 67. Interview question

### "What does TDE protect?"

> "TDE primarily protects database storage at rest by encrypting database files. It doesn't by itself prevent an authorized database principal from querying decrypted data."

---

# 68. Interview question

### "How would you secure a production SQL MI?"

Strong answer:

```text id="x3m7p9"
Identity
 ↓
Least privilege
 ↓
Private network architecture
 ↓
Encryption
 ↓
Auditing
 ↓
Security monitoring
 ↓
Vulnerability management
 ↓
Access reviews
 ↓
Incident response
```

And I'd enforce repeatable controls through IaC and policy where possible.

---

# 69. Interview question

### "How do you protect highly sensitive columns?"

> "I would first classify the data and understand the threat model. If database-level access itself must not expose plaintext values, I'd evaluate client-side/column-level encryption such as Always Encrypted, while considering application compatibility, key management, querying limitations, and operational requirements."

---

# 70. Security golden rule

> **Never ask only, "Is the database secure?" Ask, "Secure against which threat, using which control, and how do we know that control is working?"**

Think:

```text id="k8m3x5"
IDENTITY
   ↓
NETWORK
   ↓
AUTHENTICATION
   ↓
AUTHORIZATION
   ↓
ENCRYPTION
   ↓
AUDITING
   ↓
MONITORING
   ↓
RESPONSE
   ↓
RECOVERY
```

That is **defense in depth**.

---

## Next — Lesson 113: Azure SQL Managed Instance Performance Engineering — Expert Level

We'll go into **wait statistics, execution plans, Query Store, parameter sensitivity, cardinality estimation, indexing strategy, statistics, memory grants, tempdb, concurrency, blocking, deadlocks, CPU/I/O bottlenecks, plan regression, workload tuning, and a systematic production performance methodology.**
