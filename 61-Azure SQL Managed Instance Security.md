# Lesson 93 — Azure SQL Managed Instance Security

This lesson is about securing SQL MI as a **real production DBA**, not just memorizing security features.

The most important principle is:

> **Network security, authentication, authorization, encryption, auditing, and privileged access are different security layers.**

---

# 1. SQL MI security model

Think of security as layers:

```text
                    SQL MI SECURITY
                          |
       +------------------+------------------+
       |                  |                  |
    Network         Authentication      Authorization
       |                  |                  |
       ↓                  ↓                  ↓
  Who can reach?     Who are you?      What can you do?
       |
       +------------------+------------------+
                          |
                     Data Protection
                          |
                  +-------+-------+
                  |               |
               Encryption      Auditing
```

---

# 2. Authentication vs authorization

This distinction is fundamental.

### Authentication

> **Who are you?**

Examples:

```text
Azure AD / Microsoft Entra identity
SQL login
Managed identity
```

### Authorization

> **What are you allowed to do?**

Examples:

```text
SELECT
INSERT
UPDATE
EXECUTE
ALTER
CONTROL
```

So:

```text
Authentication → Identity
Authorization  → Permissions
```

---

# 3. Microsoft Entra ID

Microsoft Entra ID is Microsoft's cloud identity platform.

SQL MI can integrate with Microsoft Entra authentication.

This allows users and applications to authenticate using organizational identities instead of relying exclusively on SQL usernames/passwords.

---

# 4. Why Entra authentication matters

Traditional SQL authentication:

```text
Username
+
Password
```

creates secrets that must be managed.

Microsoft Entra authentication can provide:

```text
Central identity
MFA
Conditional Access
Group-based access
Identity lifecycle management
```

This can significantly improve enterprise security.

---

# 5. SQL authentication

SQL MI also supports SQL logins.

Example:

```text id="s5m2cr"
Login:
app_user

Password:
********
```

This can still be useful for:

```text
Legacy applications
Non-Entra-integrated applications
Specific compatibility scenarios
```

But passwords become a security responsibility.

---

# 6. Preferred enterprise model

Where practical:

```text id="g8o2f1"
Human users
     ↓
Microsoft Entra
     ↓
Groups
     ↓
SQL permissions
```

Instead of:

```text id="j5p6ny"
50 individual SQL logins
```

Group-based access is much easier to manage.

---

# 7. Example

Create an Entra group:

```text id="6g7v5m"
SQL-Production-ReadOnly
```

Members:

```text id="5ly3r3"
Alice
Bob
Carlos
```

Grant the database permission to the group.

Now:

```text id="l5q3k2"
Alice leaves company
      ↓
Remove Alice from group
      ↓
Access automatically disappears
```

You don't need to modify every database permission manually.

---

# 8. Microsoft Entra administrator

SQL MI needs an identity configured as the Microsoft Entra administrator.

Conceptually:

```text id="3t2h4z"
Microsoft Entra
      |
      ↓
Entra administrator
      |
      ↓
SQL MI
```

This establishes the administrative identity relationship for Entra authentication.

---

# 9. DBA responsibility

The Entra administrator is powerful.

Don't casually assign:

```text id="g4o5n7"
Random developer
```

Use an appropriate privileged identity or group according to organizational policy.

---

# 10. Managed identities

For applications running in Azure, managed identities are extremely useful.

Instead of:

```text id="b8f7k0"
Application
   ↓
Password stored in config
   ↓
SQL
```

you can use:

```text id="v6k1x4"
Azure workload
   ↓
Managed identity
   ↓
Microsoft Entra
   ↓
SQL MI
```

---

# 11. Why managed identity is safer

Suppose an application has:

```text id="y1p4r9"
DB_PASSWORD=SuperSecretPassword
```

If that secret leaks:

```text id="v5h7k2"
Attacker
   ↓
Password
   ↓
Database
```

With managed identity:

```text id="n6z0h3"
No application-managed database password
```

The identity is controlled through Azure identity and access management.

---

# 12. Human vs application identities

Use different identity strategies.

### Humans

```text id="3x6s7f"
Microsoft Entra users/groups
```

### Azure applications

```text id="j4p2z8"
Managed identity
```

### Legacy systems

```text id="q7d3s1"
SQL authentication where necessary
```

Don't use one shared DBA login for everyone.

---

# 13. Shared accounts are dangerous

Bad:

```text id="8k4j2z"
DBAUser
Password123
```

used by:

```text
Alice
Bob
Carlos
David
```

You lose individual accountability.

Better:

```text id="m5n8x1"
Alice → individual identity
Bob   → individual identity
Carlos → individual identity
```

---

# 14. Least privilege

The principle:

> **Give users only the permissions they need to perform their job.**

Example:

A reporting user needs:

```sql id="r5j6v2"
SELECT
```

Don't give:

```text id="c8m4n1"
db_owner
```

just because it's easier.

---

# 15. Database roles

Roles make permission management easier.

Examples include:

```text id="y4q2s9"
db_datareader
db_datawriter
```

But don't blindly use broad built-in roles.

For sensitive systems, custom roles can provide tighter control.

---

# 16. Custom role example

Suppose reporting users need:

```text id="p8j4m2"
SELECT on:
Sales
Customers
Products
```

Create a reporting role conceptually:

```text id="x7w2k6"
ReportingRole
   |
   +-- SELECT Sales
   +-- SELECT Customers
   +-- SELECT Products
```

Then add users/groups to the role.

---

# 17. RBAC vs SQL permissions

This is one of the most important Azure concepts.

### Azure RBAC

Controls access to Azure resources.

Example:

```text id="8m5f2s"
Who can modify the SQL MI resource?
Who can configure Azure networking?
Who can change Azure settings?
```

### SQL database permissions

Control what a principal can do **inside the database**.

Example:

```text id="0c6p8y"
SELECT
INSERT
UPDATE
EXECUTE
ALTER
```

---

# 18. Don't confuse them

Someone can have:

```text id="n5r7v1"
Azure Contributor
```

and still not automatically have every database-level permission you might expect.

Conversely, someone can have database permissions without being an Azure resource administrator.

Think:

```text id="d1w8z2"
Azure RBAC
     ↓
Azure resource

SQL permissions
     ↓
Database objects
```

---

# 19. Example

Developer:

```text id="a3f5q7"
Azure Contributor
```

This doesn't mean:

```text id="h8x2j9"
Developer should be db_owner
```

Keep the control planes separate.

---

# 20. Network security

Authentication is not the first line of defense.

You should also control:

> **Who can reach SQL MI?**

Think:

```text id="9n4p6s"
Internet / Network
       ↓
Network controls
       ↓
SQL MI
       ↓
Authentication
       ↓
Authorization
```

---

# 21. Private networking

For production workloads, private connectivity is commonly preferred.

Conceptually:

```text id="m4s8c1"
Application
     |
Private network
     |
     ↓
SQL MI
```

This reduces unnecessary public exposure.

---

# 22. Defense in depth

Don't rely on one security mechanism.

Use multiple layers:

```text id="0m6v8a"
Network isolation
      +
Identity
      +
Least privilege
      +
Encryption
      +
Auditing
      +
Monitoring
```

If one layer fails, others still provide protection.

---

# 23. Encryption in transit

When data travels between:

```text id="w7k1m4"
Application
      ↓
Network
      ↓
SQL MI
```

you want encrypted communication.

TLS protects data while it is moving.

---

# 24. Encryption at rest

Data stored on disk should also be protected.

Think:

```text id="z9c3n8"
Data
 ↓
Storage
 ↓
Encryption
```

SQL MI supports encryption at rest through **Transparent Data Encryption (TDE)**.

---

# 25. TDE

TDE protects database files at rest.

Conceptually:

```text id="q1b5k7"
Database files
     ↓
TDE
     ↓
Encrypted storage
```

The goal is to protect against someone obtaining the underlying storage without appropriate authorization.

---

# 26. TDE does NOT solve everything

TDE does not mean:

```text id="r2h6v9"
User queries encrypted data
```

and magically prevents the user from seeing it.

If a user has:

```sql id="z7m3c2"
SELECT Salary
FROM Employees;
```

and permission to execute it, TDE doesn't hide Salary from that user.

TDE primarily protects **data at rest**.

---

# 27. Encryption layers

Remember:

```text id="t9w2p5"
At rest
   ↓
TDE

In transit
   ↓
TLS

Highly sensitive columns
   ↓
Additional application/data-level encryption options
```

Different problems require different controls.

---

# 28. Always Encrypted

For highly sensitive information, you may need stronger protection.

Examples:

```text id="v5k3x1"
Social Security numbers
Credit-card-related information
Highly sensitive personal data
```

Always Encrypted is designed to keep certain sensitive values protected from database administrators and the database engine in specific usage models.

---

# 29. Why this matters

With ordinary database encryption:

```text id="u4c8q2"
DBA
 ↓
SQL Server
 ↓
Data
```

The database engine can process plaintext values.

With client-side encryption approaches such as Always Encrypted:

```text id="h7d1s5"
Application
 ↓
Encryption
 ↓
SQL
 ↓
Encrypted value
```

The database may not have access to the plaintext value.

---

# 30. Encryption trade-offs

Stronger encryption can introduce:

```text id="x3k6r8"
Application complexity
Query limitations
Indexing considerations
Performance implications
Key-management requirements
```

Don't encrypt everything blindly.

Classify the data first.

---

# 31. Key management

Encryption is only useful if keys are properly protected.

Think:

```text id="m6p2q9"
Encrypted data
      +
Protected keys
      =
Useful encryption
```

If attackers obtain both:

```text
Data
+
Keys
```

the protection can be compromised.

---

# 32. Key rotation

Organizations may require periodic key rotation.

You need to understand:

```text id="r5n7t1"
Who owns the key?
Where is it stored?
Who can access it?
How is it rotated?
What happens during rotation?
```

---

# 33. Auditing

Auditing answers:

> **What happened?**

For example:

```text id="y2c6v8"
Who accessed data?
Who changed data?
Who changed permissions?
When did it happen?
From where?
```

---

# 34. Monitoring vs auditing

### Monitoring

> "Is something wrong?"

Example:

```text id="k4m9x2"
CPU = 95%
```

### Auditing

> "Who did what?"

Example:

```text id="q7j3p6"
User Alice executed UPDATE on table X.
```

Different purposes.

---

# 35. Defender for SQL

Microsoft Defender for SQL provides additional security capabilities around SQL workloads.

It can help identify:

```text id="p3z6w9"
Security risks
Suspicious activity
Vulnerability findings
Potential threats
```

Think of it as an additional security layer—not a replacement for proper permissions and network controls.

---

# 36. Vulnerability management

Security isn't just about attackers.

You should also identify:

```text id="f2m7c4"
Weak configurations
Excessive permissions
Outdated components
Potential vulnerabilities
```

Then remediate according to organizational risk.

---

# 37. SQL injection

A DBA must understand SQL injection even if application developers own the code.

Bad application behavior:

```text id="h8k2p4"
String concatenation
        ↓
SQL query
```

Example conceptually:

```text id="a7m5r1"
SELECT ...
WHERE Name = 'user input'
```

If input isn't handled safely, attackers may manipulate the SQL statement.

---

# 38. Parameterized queries

Preferred approach:

```text id="d4n8x3"
Application
   ↓
Parameterized SQL
   ↓
SQL MI
```

This reduces SQL injection risk.

DBAs should work with application teams to ensure secure database access patterns.

---

# 39. Secrets management

Don't put production passwords in:

```text id="s9j3v7"
Source code
Git repositories
Plain text config files
Documentation
Chat messages
```

Use appropriate secret-management mechanisms and managed identities where possible.

---

# 40. Password rotation

If SQL authentication must be used:

```text id="w5c8m2"
Password
   ↓
Secure storage
   ↓
Rotation
   ↓
Controlled access
```

A forgotten shared password can become a major security problem.

---

# 41. Privileged access

DBA privileges should be tightly controlled.

Example hierarchy:

```text id="b3n7k5"
Read-only
   ↓
Developer
   ↓
DBA
   ↓
Security administrator
   ↓
Platform administrator
```

Not every person needs every level.

---

# 42. Break-glass account

Some organizations maintain emergency administrative access.

A break-glass account should be:

```text id="q8v2n6"
Highly protected
Rarely used
Monitored
Audited
Tested
```

It should not become the normal DBA account.

---

# 43. Privileged identity management

For highly privileged operations, organizations may use temporary elevation.

Conceptually:

```text id="z5m1c7"
Normal user
     ↓
Request elevation
     ↓
Approval
     ↓
Temporary privilege
     ↓
Perform task
     ↓
Privilege removed
```

This reduces standing administrative privileges.

---

# 44. Production access

A mature environment may separate:

```text id="n6r4t8"
Development
Testing
Production
```

and require stronger controls for production.

For example:

```text id="s3j7q1"
Production DBA access
        ↓
MFA
        ↓
Privileged identity
        ↓
Auditing
```

---

# 45. Database ownership

Be careful with database owners and highly privileged accounts.

A good security review checks:

```text id="f7k2x5"
Who owns the database?
Who has db_owner?
Who has CONTROL?
Who has ALTER?
Who can create users?
Who can grant permissions?
```

---

# 46. Permission review

Perform periodic access reviews.

Example:

```text id="v4m8c2"
Every quarter:
    ↓
List privileged users
    ↓
Remove departed users
    ↓
Remove unnecessary access
    ↓
Review service identities
    ↓
Review production permissions
```

---

# 47. Orphaned access

An employee may leave the company but a database permission may remain.

That's why identity lifecycle management matters.

Ideal flow:

```text id="j5q7x9"
Employee leaves
      ↓
Entra account disabled
      ↓
Group membership removed
      ↓
Database access disappears
```

This is another advantage of centralized identity management.

---

# 48. Incident example — suspicious login

Suppose monitoring reports:

```text id="r3m6w8"
Unexpected login
```

Don't immediately reset every password.

Investigate:

```text id="x7p1c4"
Which identity?
When?
Source?
Authentication method?
Successful or failed?
What SQL was executed?
What permissions did identity have?
```

Then contain the incident.

---

# 49. Security incident workflow

```text id="k8m4s2"
Detect
  ↓
Validate
  ↓
Contain
  ↓
Investigate
  ↓
Eradicate
  ↓
Recover
  ↓
Review
```

---

# 50. Example security incident

At 1:30 AM:

```text id="b5n7v2"
Unexpected production login
```

Investigation:

```text id="q9x3m1"
Identity = developer account
Source = unexpected location
```

Then:

```text id="r4k8c6"
Review audit logs
      ↓
Check executed commands
      ↓
Check privilege changes
      ↓
Check data access
```

If compromised:

```text id="t2m6p9"
Disable identity
Revoke access
Investigate activity
Rotate affected credentials
Validate database integrity
```

---

# 51. Security incident: excessive permissions

Suppose:

```text id="w6c3n8"
ReportingUser
```

has:

```text id="a4q7z1"
db_owner
```

That's excessive.

Change to:

```text id="p8m2r5"
ReportingRole
    ↓
SELECT only
```

This reduces blast radius.

---

# 52. Blast radius

Security engineers care deeply about blast radius.

If an account is compromised:

### Broad privilege

```text id="f3j9k2"
Compromised account
      ↓
db_owner
      ↓
Huge database impact
```

### Least privilege

```text id="x8m5c4"
Compromised account
      ↓
SELECT on reporting schema
      ↓
Limited impact
```

Least privilege reduces damage.

---

# 53. Defense in depth example

A production SQL MI might have:

```text id="d7k2p9"
Private networking
       ↓
Network restrictions
       ↓
Microsoft Entra authentication
       ↓
MFA for humans
       ↓
Managed identity for applications
       ↓
Least-privilege SQL roles
       ↓
TDE
       ↓
TLS
       ↓
Auditing
       ↓
Threat detection
```

That's much stronger than:

```text id="g3x6m1"
Username + password
```

---

# 54. Security checklist

```text id="r5c8v2"
NETWORK
☐ Private connectivity where appropriate
☐ Minimize exposure
☐ Network access reviewed

AUTHENTICATION
☐ Entra authentication
☐ MFA for humans
☐ Managed identities
☐ SQL authentication only where necessary

AUTHORIZATION
☐ Least privilege
☐ Group-based access
☐ Custom roles where appropriate
☐ Privileged accounts reviewed

ENCRYPTION
☐ TDE
☐ TLS
☐ Sensitive-data encryption where required
☐ Key management

AUDITING
☐ Auditing enabled
☐ Logs retained appropriately
☐ Security events monitored

OPERATIONS
☐ Access reviews
☐ Credential rotation
☐ Break-glass procedures
☐ Incident response process
```

---

# 55. Interview question

### "What's the difference between authentication and authorization?"

Strong answer:

> "Authentication establishes the identity of the user or application. Authorization determines what that authenticated identity is allowed to access or execute."

---

# 56. Interview question

### "RBAC vs SQL permissions?"

Answer:

> "Azure RBAC controls access to Azure resources and management operations, while SQL permissions control access to databases, schemas, tables, procedures, and other database objects."

---

# 57. Interview question

### "Why use managed identity?"

Answer:

> "Managed identity allows an Azure workload to authenticate using an Azure-managed identity rather than storing database credentials in application configuration, reducing secret-management risk."

---

# 58. Interview question

### "What does TDE protect?"

Answer:

> "TDE protects database files and backups at rest through encryption. It doesn't replace authorization or protect data from a user who already has permission to query it."

---

# 59. Interview question

### "What does TLS protect?"

Answer:

> "TLS protects data in transit between the client/application and SQL service."

---

# 60. Interview question

### "What is least privilege?"

Answer:

> "Users and applications receive only the permissions necessary to perform their required tasks, reducing the potential impact if an identity is compromised."

---

# 61. Interview question

### "How would you secure an application connecting to SQL MI?"

A strong answer:

```text id="c4n8v2"
1. Prefer private connectivity.
2. Use managed identity where supported.
3. Use Microsoft Entra authentication.
4. Grant only required database permissions.
5. Encrypt connections with TLS.
6. Enable encryption at rest.
7. Avoid hard-coded secrets.
8. Audit access.
9. Monitor suspicious activity.
10. Review permissions periodically.
```

---

# 62. Final security mental model

Memorize this:

```text id="z7q3m1"
                SECURE SQL MI
                     |
     +---------------+---------------+
     |               |               |
  Network       Authentication   Authorization
     |               |               |
 Private          Entra          Least privilege
 connectivity     Managed ID     Roles
     |               |               |
     +---------------+---------------+
                     |
                Data Protection
                     |
             +-------+-------+
             |               |
          Encryption      Auditing
             |               |
          TDE/TLS        Monitoring
             |
             ↓
          Compliance
```

And the most important principle:

> **Security is not one feature. It is a layered system.**

---

## Next — Lesson 94: Azure SQL Managed Instance Disaster Recovery & Business Continuity

We'll go deeper into **RPO/RTO, failover groups, geo-replication concepts, cross-region recovery, planned vs unplanned failover, DNS/application connectivity, DR runbooks, DR testing, region failure scenarios, backup-based recovery vs replication-based DR, and a complete production-region outage exercise.**
