# Lesson 106 — Azure SQL Managed Instance Security — Advanced DBA

This lesson focuses on securing SQL MI as a **production data platform**, not merely enabling a few security settings.

The central principle:

> **Security is layered: identity, authorization, network, encryption, data protection, auditing, monitoring, and operational controls must work together.**

---

# 1. The security model

Think in layers:

```text
User / Application
        ↓
Identity
        ↓
Authentication
        ↓
Authorization
        ↓
Network
        ↓
Database
        ↓
Data
        ↓
Audit / Monitoring
```

A weakness in one layer can undermine the others.

---

# 2. Authentication vs authorization

This distinction must be crystal clear.

### Authentication

> **Who are you?**

Examples:

```text
Microsoft Entra identity
SQL login
Managed identity
```

### Authorization

> **What are you allowed to do?**

Examples:

```text
SELECT
INSERT
EXECUTE
ALTER
CONTROL
```

---

# 3. Authentication flow

Conceptually:

```text
Application
    ↓
Identity provider
    ↓
Authentication
    ↓
SQL MI
    ↓
Database permissions
```

Authentication proves identity.

Authorization determines access.

---

# 4. Microsoft Entra authentication

Microsoft Entra ID provides centralized identity management.

Benefits include:

```text
Centralized identity
MFA integration
Conditional access
Identity lifecycle management
Reduced password dependency
```

For enterprise environments, Entra-based authentication is often preferable where supported.

---

# 5. SQL authentication

SQL authentication uses:

```text
Login
+
Password
```

It remains useful in certain scenarios, but password-based credentials introduce management responsibilities.

You must consider:

```text
Password storage
Rotation
Exposure
Account lifecycle
Auditing
```

---

# 6. Managed identity

Managed identity is particularly useful for Azure-hosted applications.

Conceptually:

```text
Application
    ↓
Managed Identity
    ↓
Entra
    ↓
SQL MI
```

The application doesn't need to store a database password.

---

# 7. Why managed identity is powerful

Instead of:

```text
Connection string
+
username
+
password
```

you can use an identity-based authentication model.

Benefits:

```text
No hardcoded password
Centralized identity
Lifecycle tied to Azure resource
Better secret hygiene
```

---

# 8. Authorization layers

Permissions can exist at different levels:

```text
Azure RBAC
       ↓
SQL MI
       ↓
Database
       ↓
Schema
       ↓
Object
       ↓
Column / Row controls
```

Don't confuse Azure RBAC with database permissions.

---

# 9. Azure RBAC

Azure RBAC controls access to Azure resources.

Examples:

```text
Who can manage the SQL MI resource?
Who can change networking?
Who can view Azure configuration?
```

It does not automatically mean:

> "This user can SELECT every table."

---

# 10. SQL permissions

SQL permissions control database-level operations.

Examples:

```text
SELECT
INSERT
UPDATE
DELETE
EXECUTE
ALTER
CONTROL
```

This is the layer where database authorization becomes important.

---

# 11. Least privilege

The goal is:

> Give identities only the permissions they actually need.

Bad:

```text id="a2q7m4"
Application → db_owner
```

Better:

```text id="k8x3p1"
Application
   ↓
Specific database role
   ↓
Required permissions
```

---

# 12. Database roles

Use roles to group permissions.

Conceptually:

```text
ApplicationRole
   ↓
SELECT on required tables
EXECUTE on required procedures
```

Then assign users/applications to the role.

This is easier to manage than granting permissions individually.

---

# 13. Don't use dbo casually

Giving an application broad ownership rights is dangerous.

If the application is compromised:

```text id="m5r8v2"
Compromised application
       ↓
Broad DB permissions
       ↓
Large blast radius
```

Least privilege reduces that blast radius.

---

# 14. Separation of duties

Different responsibilities should be separated where practical.

Example:

```text id="q6n3w8"
Developer
→ deploy application

DBA
→ manage database

Security
→ review access

Application identity
→ execute application workload
```

Avoid one identity having unrestricted access to everything.

---

# 15. Privileged accounts

Administrative identities deserve extra protection.

Use:

```text id="j4c9p7"
Strong authentication
MFA where applicable
Privileged access controls
Auditing
Limited usage
```

Don't use an administrative account for routine application activity.

---

# 16. Network security

Database security isn't just permissions.

Consider:

```text id="v7m2x5"
Who can reach SQL MI?
From where?
Over what path?
Is traffic private?
Are unnecessary paths exposed?
```

---

# 17. Network isolation

A common enterprise architecture is:

```text id="s3k8n1"
Internet
   X
   |
Private network
   |
Application
   |
SQL MI
```

The database shouldn't be directly exposed simply because connectivity is convenient.

---

# 18. Private connectivity

Private networking helps keep database traffic within controlled network paths.

Typical design:

```text id="p9x4m6"
Users
 ↓
Application
 ↓
Private network
 ↓
SQL MI
```

This reduces unnecessary public exposure.

---

# 19. Network vs authorization

These are different controls.

Network security answers:

> **Can you reach the service?**

Authorization answers:

> **Once connected, what can you do?**

You need both.

---

# 20. Encryption in transit

Traffic between clients and SQL MI should use encrypted connections.

Conceptually:

```text id="c7m2v9"
Application
   ↓
Encrypted connection
   ↓
SQL MI
```

Encryption protects data while it travels across the network.

---

# 21. Encryption at rest

Data stored on infrastructure should also be protected.

Think:

```text id="r4x8k3"
Data at rest
    ↓
Encryption
    ↓
Storage
```

This protects against certain scenarios involving unauthorized access to underlying storage.

---

# 22. TDE

**Transparent Data Encryption (TDE)** protects database files at rest.

Conceptually:

```text id="n6q2p8"
Database
   ↓
TDE
   ↓
Encrypted storage
```

Applications generally don't need to change their SQL statements to use TDE.

---

# 23. TDE and application encryption are different

TDE protects storage.

It does **not** mean:

> "The application can never see plaintext data."

Once authorized, SQL Server can decrypt data for processing.

For highly sensitive fields, additional controls may be required.

---

# 24. Always Encrypted

Always Encrypted is designed to protect sensitive data so that encryption/decryption can be controlled outside the database engine for supported scenarios.

Useful for highly sensitive fields such as:

```text id="x8m4q1"
Government identifiers
Financial identifiers
Highly sensitive personal data
```

---

# 25. TDE vs Always Encrypted

Think:

```text id="w2k7p5"
TDE
→ Protects database storage

Always Encrypted
→ Protects selected sensitive columns from broader database-side access
```

They solve different problems.

---

# 26. Encryption key management

Security depends on protecting encryption keys.

Think:

```text id="g5n8r2"
Data
 ↓
Encryption key
 ↓
Key management
```

If key management is weak, encryption protection is weakened.

---

# 27. Key rotation

Keys may need to be rotated according to organizational policy.

Automation can help:

```text id="v3q6m9"
Scheduled policy
 ↓
Key rotation
 ↓
Validation
 ↓
Audit
```

But key rotation must be carefully tested.

---

# 28. Dynamic Data Masking

Dynamic Data Masking can limit how sensitive data appears to certain users.

Example:

```text id="k4m8x2"
Actual:
555-123-4567

Masked:
XXXX-XXX-4567
```

---

# 29. Important limitation of masking

Masking is **not equivalent to encryption**.

It controls presentation/access behavior in supported scenarios.

Don't use masking as your only protection for highly sensitive data.

---

# 30. Row-Level Security

Row-Level Security can restrict which rows a user can access.

Example:

```text id="q7x3p5"
Salesperson A
→ Region A rows

Salesperson B
→ Region B rows
```

Same table:

```text id="j8m2v4"
Sales
```

but different visible rows.

---

# 31. RLS architecture

Conceptually:

```text
User
 ↓
Security predicate
 ↓
Allowed rows
```

This can be useful in multi-tenant or departmental systems.

---

# 32. RLS considerations

Be careful with:

```text id="f6p9c2"
Performance
Complex predicates
Testing
Administrative access
Application behavior
```

Security logic must be tested just as seriously as business logic.

---

# 33. Auditing

Auditing answers:

> **What happened?**

You may need to track:

```text id="m3v7k8"
Who
What
When
Where
Which object
```

---

# 34. Audit examples

Potential events:

```text id="r8x2n5"
Login activity
Permission changes
Schema changes
Sensitive queries
Data access
Administrative operations
```

The exact events collected should align with security requirements.

---

# 35. Audit vs monitoring

### Monitoring

> "Is something wrong?"

### Auditing

> "What happened and who did it?"

You often need both.

---

# 36. Threat detection

Security monitoring should look for unusual behavior:

```text id="x5k9p1"
Unusual login
Unexpected location
Privilege escalation
Abnormal query behavior
Repeated failed authentication
Suspicious access patterns
```

---

# 37. Defender for SQL

Security tooling can provide additional monitoring and threat-detection capabilities around SQL workloads.

The important architectural point is:

```text id="n7m4c8"
Database
   ↓
Security telemetry
   ↓
Detection
   ↓
Alert
   ↓
Investigation
```

Don't treat security tooling as a replacement for proper permissions.

---

# 38. Vulnerability management

Security should also identify:

```text id="p2q8m6"
Weak configuration
Unsupported software
Excessive privileges
Potential vulnerabilities
```

Then remediate systematically.

---

# 39. Access review

Permissions change over time.

Example:

```text id="w9m3x4"
Employee joins
 ↓
Gets access
 ↓
Changes role
 ↓
Still retains old access
```

This creates privilege accumulation.

Regular access reviews are essential.

---

# 40. Orphaned access

Look for:

```text id="c5k8n2"
Inactive users
Unused service identities
Old database users
Former employees
Unnecessary roles
```

Remove access that is no longer justified.

---

# 41. Service identities

Applications should ideally use dedicated identities.

Bad:

```text id="h2v6q9"
10 applications
   ↓
Same SQL admin account
```

If compromised:

```text id="j4p7x1"
All 10 applications
   ↓
Potentially compromised
```

Better:

```text id="m8r3c5"
App A → Identity A
App B → Identity B
App C → Identity C
```

---

# 42. Credential rotation

If passwords/secrets are required:

```text id="v5n9k2"
Generate
 ↓
Store securely
 ↓
Deploy
 ↓
Validate
 ↓
Rotate
```

Rotation should be designed into the architecture rather than treated as an emergency procedure.

---

# 43. SQL injection

Database security also depends on application coding.

Bad:

```text id="q1w8e5"
SQL = "SELECT ... WHERE Name = '" + userInput + "'"
```

Better:

```text id="z6x3m9"
Parameterized query
```

or appropriately designed stored procedures.

---

# 44. Why DBA should care about SQL injection

Because an application vulnerability can become:

```text id="p7k4c2"
Application compromise
 ↓
Database access
 ↓
Data exposure
```

Least privilege limits the blast radius.

---

# 45. Stored procedures

Stored procedures can help centralize controlled database operations.

For example:

```text id="r2m8v5"
Application
   ↓
EXECUTE procedure
   ↓
Controlled database operation
```

But stored procedures don't automatically make an application secure.

They must still be designed correctly.

---

# 46. Dynamic SQL

Dynamic SQL requires special care.

Badly constructed dynamic SQL can introduce injection risks.

Use:

```text id="x9c4n7"
Parameterization
Safe identifier handling
Strict validation
Least privilege
```

---

# 47. Production security architecture

A mature design might look like:

```text id="f8m3q1"
                    USERS
                      |
                Entra Identity
                      |
                 Application
                      |
              Managed Identity
                      |
                Private Network
                      |
                    SQL MI
          +-----------+-----------+
          |           |           |
         TDE        RLS      DB Roles
          |           |           |
          +-----------+-----------+
                      |
               Audit / Monitoring
                      |
                Security Alerts
```

---

# 48. Security incident example

Suppose:

```text id="n4x7k2"
Unexpected account
accesses sensitive table
```

Investigation:

```text id="c8m5p9"
1. Identify identity
2. Review authentication
3. Review permissions
4. Review audit logs
5. Determine accessed objects
6. Determine data exposure
7. Revoke/contain access
8. Rotate credentials if necessary
9. Investigate root cause
10. Document incident
```

---

# 49. Security incident response

Use a lifecycle:

```text id="q6v2m8"
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

---

# 50. Containment

Possible actions depend on the incident:

```text id="m7p3x5"
Disable compromised identity
Remove excessive permissions
Block network path
Rotate credentials
Restrict application
```

Do not blindly destroy evidence.

---

# 51. Evidence

Preserve:

```text id="x2n8k4"
Audit logs
Authentication events
Query activity
Configuration changes
Deployment records
Network/security telemetry
```

This supports investigation.

---

# 52. Security baseline

Create a standard SQL MI security baseline.

Example:

```text id="w5r9m3"
☐ Entra authentication configured
☐ Least privilege
☐ Private networking
☐ Encryption enabled
☐ Audit enabled
☐ Monitoring enabled
☐ Sensitive data controls
☐ Credential management
☐ Access reviews
☐ Security alerts
```

---

# 53. Continuous security

Security isn't:

```text id="a8k4q1"
Configure once
 ↓
Forget
```

It's:

```text id="s3m7v9"
Configure
 ↓
Monitor
 ↓
Review
 ↓
Test
 ↓
Improve
```

---

# 54. Security checklist

```text id="r4x8m2"
IDENTITY
☐ Entra authentication
☐ Managed identity
☐ Least privilege
☐ MFA for privileged users where applicable
☐ Access reviews

AUTHORIZATION
☐ Database roles
☐ Minimal permissions
☐ Separation of duties
☐ No unnecessary db_owner

NETWORK
☐ Private connectivity
☐ Restricted access paths
☐ Network segmentation

DATA
☐ TDE
☐ Appropriate key management
☐ Sensitive-column protection
☐ RLS where appropriate
☐ Masking where appropriate

MONITORING
☐ Auditing
☐ Security alerts
☐ Threat detection
☐ Vulnerability assessment

OPERATIONS
☐ Credential rotation
☐ Incident response
☐ Security baseline
☐ Regular reviews
```

---

# 55. Interview question

### "What's the difference between Azure RBAC and SQL permissions?"

> "Azure RBAC governs access to Azure resources and management operations, while SQL permissions govern what an identity can do inside SQL Server databases. They are complementary layers."

---

# 56. Interview question

### "Is TDE enough to secure sensitive data?"

> "No. TDE primarily protects data at rest. Sensitive-data security also requires identity, authorization, network controls, auditing, and potentially column-level protections such as Always Encrypted depending on the threat model."

---

# 57. Interview question

### "Why use managed identity?"

> "It allows supported Azure workloads to authenticate using an Azure-managed identity rather than storing long-lived database credentials in application configuration. It reduces secret-management risk and works well with least-privilege access."

---

# 58. Interview question

### "What's the difference between masking and encryption?"

> "Masking changes how data is presented to users in supported scenarios, while encryption protects the underlying data cryptographically. Masking should not be treated as a replacement for encryption."

---

# 59. Interview question

### "How would you secure an application connecting to SQL MI?"

Strong answer:

```text id="z1m6p8"
Managed identity
        +
Private connectivity
        +
Least-privilege database role
        +
Encrypted connection
        +
TDE
        +
Auditing/monitoring
        +
Secure secret management where required
```

---

# 60. Final security model

Memorize:

```text id="h4q8m1"
WHO?
 ↓
Authentication

CAN DO WHAT?
 ↓
Authorization

CAN REACH FROM WHERE?
 ↓
Network

IS DATA PROTECTED?
 ↓
Encryption

WHAT HAPPENED?
 ↓
Auditing

IS SOMETHING SUSPICIOUS?
 ↓
Threat detection

CAN WE RECOVER?
 ↓
Backup + DR
```

---

# 61. Golden rule

> **Never rely on a single security control.**

A secure SQL MI environment uses **defense in depth**:

```text
Identity
+
Least privilege
+
Private networking
+
Encryption
+
Data-level controls
+
Auditing
+
Monitoring
+
Incident response
```

That is the mindset expected from a senior/advanced Azure SQL DBA.

---

## Next — Lesson 107: Azure SQL Managed Instance Troubleshooting Masterclass

We'll put everything together and troubleshoot **realistic production incidents** involving **CPU spikes, blocking, deadlocks, memory pressure, I/O latency, tempdb problems, query regressions, connection failures, authentication issues, storage exhaustion, failed deployments, and regional outages** using a structured DBA incident methodology.
