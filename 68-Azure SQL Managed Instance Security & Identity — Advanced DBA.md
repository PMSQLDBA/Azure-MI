# Lesson 100 — Azure SQL Managed Instance Security & Identity — Advanced DBA

🎉 **Lesson 100** — this is a major milestone.

Now we move from availability and recovery into **production security architecture**.

The most important principle:

> **Authentication answers "Who are you?" Authorization answers "What are you allowed to do?"**

---

# 1. The security model

Think of SQL MI security as layers:

```text
User / Application
       ↓
Authentication
       ↓
Authorization
       ↓
Database permissions
       ↓
Data protection
       ↓
Auditing & monitoring
```

---

# 2. Authentication vs authorization

### Authentication

Determines identity.

Examples:

```text
Microsoft Entra identity
SQL login
Managed identity
Service principal
```

### Authorization

Determines permissions.

Examples:

```text
SELECT
INSERT
UPDATE
EXECUTE
ALTER
CONTROL
```

Remember:

```text
Authentication = WHO?
Authorization  = WHAT?
```

---

# 3. Microsoft Entra authentication

For Azure SQL Managed Instance, Microsoft Entra authentication can provide centralized identity management.

Conceptually:

```text
User
 ↓
Microsoft Entra ID
 ↓
SQL Managed Instance
 ↓
Database permissions
```

This can reduce dependence on individual SQL passwords.

---

# 4. Why centralized identity matters

Suppose an employee leaves the organization.

With centralized identity:

```text
Employee account disabled
        ↓
Access can be revoked centrally
```

Instead of hunting through multiple databases for passwords and logins.

---

# 5. Human users vs applications

Don't treat them identically.

### Human

Usually:

```text
Microsoft Entra identity
```

### Application

Potential options include:

```text
Managed identity
Service principal
Other supported authentication mechanism
```

The preferred choice depends on the application architecture.

---

# 6. Managed identity

A managed identity allows an Azure resource to authenticate without you manually storing a password or secret in application code.

Conceptually:

```text
Azure Application
       ↓
Managed Identity
       ↓
Microsoft Entra
       ↓
SQL MI
```

This is powerful because credentials are managed by the platform rather than embedded in source code.

---

# 7. Why managed identity is valuable

Avoid:

```text
Application
   ↓
Hard-coded SQL password
```

Prefer:

```text
Application
   ↓
Managed Identity
   ↓
Microsoft Entra
   ↓
Database
```

This reduces secret-management burden.

---

# 8. Service principals

A service principal represents an application identity in Microsoft Entra.

Conceptually:

```text
Application
    ↓
Service principal
    ↓
Microsoft Entra
    ↓
SQL MI
```

Use application identities rather than sharing human credentials.

---

# 9. Never share DBA credentials

Bad:

```text
app1 → DBA login
app2 → DBA login
developer → DBA login
```

Better:

```text
app1 → app identity
app2 → app identity
developer → individual identity
DBA → individual privileged identity
```

This improves:

```text
Accountability
Auditing
Least privilege
Revocation
```

---

# 10. RBAC vs SQL permissions

This is extremely important.

Azure RBAC controls access to **Azure resources**.

SQL permissions control access to **SQL objects/data**.

Think:

```text
Azure RBAC
     ↓
Azure resource management
```

versus:

```text
SQL permissions
     ↓
Database objects and data
```

They are related but not interchangeable.

---

# 11. Example

A user may have Azure permissions allowing them to:

```text
Manage SQL MI resource
```

That does **not automatically mean** they should have:

```text
SELECT * FROM SensitiveCustomers
```

Database permissions still matter.

---

# 12. Least privilege

The principle:

> **Give identities only the permissions they actually need.**

Suppose an application only reads data.

Don't grant:

```text
db_owner
```

when it only needs:

```text
SELECT
```

---

# 13. Why db_owner is dangerous

`db_owner` provides extremely broad control over the database.

If an application account is compromised:

```text
Compromised application
        ↓
db_owner
        ↓
Huge blast radius
```

Least privilege limits the damage.

---

# 14. Database roles

Instead of assigning permissions individually to dozens of users:

```text
User A → SELECT
User B → SELECT
User C → SELECT
```

Create a role:

```text
ReportingReaders
       ↓
SELECT permissions
```

Then:

```text
User A → ReportingReaders
User B → ReportingReaders
User C → ReportingReaders
```

---

# 15. Role-based access

Conceptually:

```text
Identity
   ↓
Role
   ↓
Permissions
   ↓
Objects
```

This is easier to manage and audit.

---

# 16. Example

Suppose reporting users need:

```text
SELECT
```

on:

```text
Sales
Customers
Products
```

Create a role such as:

```text
ReportReader
```

Grant the required permissions to that role.

Assign users to the role.

---

# 17. EXECUTE permissions

Applications frequently execute stored procedures.

Instead of giving broad table permissions:

```text
Application
   ↓
EXECUTE
   ↓
Stored Procedure
```

This can provide a controlled interface to data.

For example:

```sql
GRANT EXECUTE
ON OBJECT::dbo.GetCustomerOrders
TO AppReader;
```

---

# 18. Application access pattern

A good design might look like:

```text
Application identity
        ↓
AppRole
        ↓
EXECUTE
        ↓
Stored procedures
        ↓
Required tables
```

rather than:

```text
Application
        ↓
db_owner
```

---

# 19. SQL logins

SQL authentication can still be useful in some scenarios.

But understand the trade-offs:

```text
SQL login
   ↓
Username + password
```

You now have to manage:

```text
Password
Rotation
Storage
Expiration policy
Access review
Secret leakage
```

Use stronger identity mechanisms where appropriate.

---

# 20. Password management

Never put passwords in:

```text
Source code
Git repositories
Configuration files
Scripts
Chat messages
Documentation
```

Instead use appropriate secret-management mechanisms.

---

# 21. Secrets architecture

Conceptually:

```text
Application
    ↓
Managed identity
    ↓
Secret store
    ↓
Credential
```

or, when supported by the architecture:

```text
Application
    ↓
Managed identity
    ↓
Microsoft Entra
    ↓
SQL MI
```

The second pattern can eliminate a stored database password entirely.

---

# 22. Privileged access

DBAs need elevated permissions.

But privileged access should be:

```text
Controlled
Audited
Time-limited where appropriate
Individually attributable
```

Avoid:

```text
Shared DBA account
```

because then you can't easily answer:

> "Who performed this change?"

---

# 23. Break-glass access

Organizations sometimes maintain emergency administrative access.

The principle is:

```text
Normal operations
     ↓
Least privilege

Emergency situation
     ↓
Controlled break-glass access
     ↓
Heavy auditing
```

Emergency accounts should be tightly protected and monitored.

---

# 24. Auditing

Auditing answers:

> **What happened, when, and who performed it?**

Potential audit events include:

```text
Login
Failed login
DDL change
Permission change
Data access
Administrative activity
```

The exact audit configuration should match compliance and security requirements.

---

# 25. Audit vs monitoring

### Monitoring

Asks:

> Is something wrong?

Example:

```text
CPU high
Connections high
Latency high
```

### Auditing

Asks:

> What happened?

Example:

```text
User X changed permissions
User Y accessed table Z
```

Different objectives.

---

# 26. Security monitoring

Watch for:

```text
Repeated login failures
Unexpected privilege changes
Unknown identities
Suspicious queries
Unexpected data access
Unusual administrative activity
```

---

# 27. Encryption at rest

Encryption protects stored data.

Conceptually:

```text
Database files
     ↓
Encrypted storage
```

Azure SQL services support encryption at rest mechanisms such as Transparent Data Encryption (TDE).

---

# 28. TDE

Transparent Data Encryption protects database files at rest.

The key idea:

```text
Database data
     ↓
Encryption
     ↓
Stored encrypted
```

Applications generally don't need to change their SQL queries simply because TDE is enabled.

---

# 29. TDE vs application encryption

Don't confuse:

```text
TDE
```

with:

```text
Application-level encryption
```

TDE protects the storage layer.

Application-level encryption can protect specific sensitive fields/data.

They solve different problems.

---

# 30. Encryption in transit

Data moving between:

```text
Application
      ↓
SQL MI
```

should be protected using encrypted connections such as TLS.

Think:

```text
At rest → TDE/encryption
In transit → TLS
```

---

# 31. Defense in depth

A strong security design has multiple layers:

```text
Identity
   ↓
Network security
   ↓
Authentication
   ↓
Authorization
   ↓
Encryption
   ↓
Auditing
   ↓
Monitoring
```

If one layer fails, other layers still provide protection.

---

# 32. Network security

Don't rely on database permissions alone.

Use appropriate network controls such as:

```text
Private networking
Firewall controls
Network security controls
Restricted routing
```

Goal:

> **Reduce who can even reach the database.**

---

# 33. Identity + network

Strong architecture:

```text
Untrusted Internet
       X
       |
   Network controls
       |
Trusted application network
       |
     SQL MI
       |
   SQL permissions
```

This creates multiple barriers.

---

# 34. Data classification

Not all data has equal sensitivity.

Example:

```text
Public
Internal
Confidential
Highly sensitive
```

Database security should reflect data classification.

---

# 35. Sensitive data

Examples:

```text
Customer identity
Financial information
Credentials
Personal information
Business secrets
```

For sensitive data, consider:

```text
Encryption
Restricted access
Auditing
Data masking where appropriate
Minimal retention
```

---

# 36. Dynamic Data Masking

Data masking can help prevent certain users from seeing full sensitive values.

Conceptually:

```text
Privileged user:
1234-5678-9012

Restricted user:
XXXX-XXXX-9012
```

But remember:

> **Masking is not the same as encryption or authorization.**

It should not be treated as a complete security boundary.

---

# 37. Row-Level Security

Sometimes different users should see different rows.

Example:

```text
Salesperson A → Region A
Salesperson B → Region B
Manager → All regions
```

Row-level security can enforce filtering at the database layer.

Conceptually:

```text
User
 ↓
Security predicate
 ↓
Allowed rows
```

---

# 38. Why database-level controls matter

If access restrictions exist only in application code:

```text
Application
   ↓
Filter rows
```

another client might bypass the application.

If appropriate database-level controls are used:

```text
Any client
   ↓
Database security policy
   ↓
Allowed rows
```

the protection is closer to the data.

---

# 39. SQL injection

An application accepts:

```text
CustomerName
```

and builds SQL using string concatenation.

Dangerous pattern:

```text
"SELECT ... WHERE Name = '" + userInput + "'"
```

Use parameterized queries instead.

Conceptually:

```text
SQL template
+
Parameter
```

rather than constructing executable SQL from raw input.

---

# 40. Why DBA should care about SQL injection

SQL injection isn't only a developer issue.

A DBA should ensure:

```text
Application accounts have minimal privileges.
```

If injection occurs:

```text
Compromised query
      ↓
Limited database permissions
      ↓
Reduced blast radius
```

---

# 41. Security baseline

For a production SQL MI, think:

```text
☐ Strong identity
☐ Least privilege
☐ Network restriction
☐ Encryption at rest
☐ TLS in transit
☐ Auditing
☐ Monitoring
☐ Secret management
☐ Individual admin accounts
☐ Regular access review
```

---

# 42. Access review

Permissions should not be permanent by default.

Periodically ask:

```text
Who has access?
Why?
Do they still need it?
What permissions do they have?
When was it last reviewed?
```

---

# 43. Joiner / mover / leaver

A mature identity process handles:

### Joiner

```text
New employee
 ↓
Appropriate access
```

### Mover

```text
Role changes
 ↓
Permissions adjusted
```

### Leaver

```text
Employee leaves
 ↓
Access revoked
```

---

# 44. Application identity lifecycle

Do the same for applications:

```text
Application created
 ↓
Identity created
 ↓
Minimum permissions
 ↓
Periodic review
 ↓
Application retired
 ↓
Identity removed
```

Unused service principals and credentials are security risks.

---

# 45. Security incident example

Suppose an application identity is compromised.

It has:

```text
db_owner
```

Attacker can potentially perform broad database operations.

Now imagine it only has:

```text
EXECUTE
```

on a controlled set of procedures.

The blast radius can be substantially smaller.

This is why least privilege matters.

---

# 46. Security incident workflow

```text
Suspicious activity
       ↓
Detect
       ↓
Identify identity
       ↓
Contain access
       ↓
Preserve audit evidence
       ↓
Investigate
       ↓
Remediate
       ↓
Rotate/revoke credentials if needed
       ↓
Review permissions
```

---

# 47. Example — compromised application identity

Application:

```text
OrdersAPI
```

Identity:

```text
OrdersAPI-MI
```

Normal permissions:

```text
EXECUTE dbo.CreateOrder
EXECUTE dbo.GetOrder
```

Attacker attempts:

```text
DROP TABLE Customers;
```

The identity doesn't have the necessary permission.

That's defense in depth.

---

# 48. Example — excessive DBA privilege

Suppose every DBA uses:

```text
sysadmin
```

for every task.

This creates unnecessary risk.

Better:

```text
Routine task
 ↓
Minimum required permission

Emergency task
 ↓
Elevated controlled access
```

---

# 49. Security architecture

A good conceptual design:

```text
                 USERS
                   |
            Microsoft Entra
                   |
          +--------+--------+
          |                 |
       Normal             Admin
       access              access
          |                 |
       DB roles       Controlled privilege
          |                 |
          +--------+--------+
                   |
                 SQL MI
                   |
        +----------+----------+
        |          |          |
       TDE        TLS       Audit
        |          |          |
        +----------+----------+
                   |
              Monitoring
```

---

# 50. Interview question

### "What's the difference between Azure RBAC and SQL permissions?"

Strong answer:

> "Azure RBAC controls access to Azure resources and management operations, while SQL permissions control access within the SQL environment, such as databases, schemas, tables, procedures, and data. Having Azure resource permissions does not automatically grant unrestricted database access."

---

# 51. Interview question

### "Why use managed identity?"

> "Managed identity allows an Azure resource to authenticate using its platform-managed identity rather than embedding a database password or client secret in application configuration. This reduces credential-management and secret-exposure risks."

---

# 52. Interview question

### "Why shouldn't applications use db_owner?"

> "Because it violates least privilege and creates a large blast radius. A compromised application could potentially modify schema, data, permissions, or other database resources. The application should receive only the permissions required for its workload."

---

# 53. Interview question

### "What is TDE?"

> "Transparent Data Encryption protects database storage by encrypting data at rest. It addresses storage-level protection and is different from TLS, which protects data in transit."

---

# 54. Interview question

### "How would you secure a production application connecting to SQL MI?"

Strong answer:

```text
1. Use managed identity or an appropriate Microsoft Entra-based identity where possible.
2. Avoid hard-coded credentials.
3. Restrict network connectivity.
4. Grant minimum required database permissions.
5. Use roles rather than excessive individual grants.
6. Encrypt connections.
7. Enable appropriate auditing and monitoring.
8. Regularly review access.
9. Protect sensitive data.
10. Test incident and credential-revocation procedures.
```

---

# 55. Interview question

### "How do you investigate suspicious database activity?"

```text
Identity
   ↓
Authentication logs
   ↓
Audit records
   ↓
Executed operations
   ↓
Permission changes
   ↓
Timeline
   ↓
Containment
```

The key is to establish:

> **Who did what, when, from where, and with what privileges?**

---

# 56. Production security checklist

```text
IDENTITY
☐ Microsoft Entra authentication where appropriate
☐ Managed identities for applications where appropriate
☐ Individual admin accounts
☐ No shared credentials

AUTHORIZATION
☐ Least privilege
☐ Database roles
☐ Minimal application permissions
☐ Regular access reviews

NETWORK
☐ Restricted connectivity
☐ Private networking where appropriate
☐ Firewall/security controls

DATA
☐ Encryption at rest
☐ TLS in transit
☐ Sensitive-data controls
☐ Appropriate retention

MONITORING
☐ Auditing
☐ Security alerts
☐ Failed-login monitoring
☐ Privilege-change monitoring

OPERATIONS
☐ Credential rotation/revocation procedures
☐ Incident response
☐ Break-glass procedure
☐ Periodic security review
```

---

# 57. The security mental model

Memorize this:

```text
              WHO?
               |
        Authentication
               |
               ↓
             CAN DO
               |
        Authorization
               |
               ↓
             WHERE?
               |
         Network controls
               |
               ↓
            DATA
               |
       Encryption / RLS
               |
               ↓
         WHAT HAPPENED?
               |
            Auditing
               |
               ↓
         DETECT / RESPOND
```

---

# 58. Final takeaway

A secure SQL MI environment isn't created by one checkbox.

It is a combination of:

> **Strong identity + least privilege + network restriction + encryption + auditing + monitoring + tested incident response.**

And the most important principle to remember:

> **Assume an identity will eventually be compromised. Design permissions so that compromise does not automatically become total database compromise.**

---

## Next — Lesson 101: Azure SQL Managed Instance Monitoring, Alerting & Observability — Advanced DBA

We'll build a complete **production monitoring strategy**, covering **Azure Monitor, metrics, logs, Log Analytics, diagnostic settings, Query Store, waits, blocking, deadlocks, CPU/memory/I/O, alert design, dashboards, SLOs, alert fatigue, incident correlation, and a full production troubleshooting exercise.**
