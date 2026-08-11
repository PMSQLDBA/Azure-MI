# Lesson 149 — SQL Server Security: Authentication, Authorization, Encryption & Auditing

Today we're covering the security layer that ties everything together:

> **Who are you? What are you allowed to do? What data can you see? And how do we prove what happened?**

---

## 1. The four security questions

A strong SQL Server security model answers four questions:

```text
WHO are you?
    ↓
Authentication

WHAT can you do?
    ↓
Authorization

HOW is data protected?
    ↓
Encryption

WHAT happened?
    ↓
Auditing
```

---

# 2. Authentication

Authentication answers:

> **"Who are you?"**

SQL Server commonly supports:

```text
Windows / Microsoft Entra authentication
SQL Server authentication
```

The exact available methods depend on SQL Server deployment/version.

---

# 3. Windows authentication

With Windows authentication, SQL Server relies on the user's Windows identity.

Conceptually:

```text
User
 ↓
Windows identity
 ↓
SQL Server
 ↓
Login
```

This can integrate with organizational identity management and centralized account policies.

---

# 4. SQL Server authentication

SQL Server can also authenticate users with SQL logins:

```text
Username
+
Password
```

For example:

```text id="w7m3q9"
CREATE LOGIN AppLogin
WITH PASSWORD = '...';
```

In real production systems, credentials should be securely managed rather than embedded in scripts or application source code.

---

# 5. Authentication ≠ authorization

This distinction is fundamental.

Authentication:

> **Who are you?**

Authorization:

> **What are you allowed to do?**

Example:

```text id="m4p8x2"
Login succeeds
    ↓
User authenticated
    ↓
Permission check
    ↓
SELECT allowed?
```

A valid login does not automatically mean the user can access everything.

---

# 6. Server-level vs database-level security

SQL Server security exists at multiple scopes.

Conceptually:

```text id="q8c1v5"
SQL Server
   ↓
Login
   ↓
Database user
   ↓
Role
   ↓
Permissions
```

---

# 7. Login

A **login** is primarily a server-level security principal.

Example:

```sql id="5h3q9m"
CREATE LOGIN ReportingLogin
FROM WINDOWS;
```

The login can then be mapped to users in databases.

---

# 8. Database user

A database user represents the identity inside a particular database.

Conceptually:

```text id="n6r2k8"
Login
   ↓
Database User
   ↓
Database Permissions
```

A single login can potentially map to users in multiple databases.

---

# 9. Roles

Roles simplify permission management.

Instead of giving permissions individually to hundreds of users:

```text id="v3m7p1"
User A
User B
User C
User D
```

you can create/manage a role:

```text id="x9q4k6"
ReportingRole
```

and assign permissions to the role.

Then:

```text id="a1c8r5"
Users
  ↓
Role
  ↓
Permissions
```

---

# 10. Principle of least privilege

One of the most important security principles:

> **Give users and applications only the permissions they actually need.**

Bad:

```text id="h6p2m9"
Application
→ sysadmin
```

Better:

```text id="w4k8c1"
Application
→ specific database user
→ required permissions only
```

---

# 11. Why sysadmin is dangerous

A `sysadmin` login has extremely broad control.

If an application uses a sysadmin credential and the application is compromised:

```text id="r7m3x5"
Application compromise
      ↓
Database credential compromise
      ↓
Potentially entire SQL Server compromise
```

That's an enormous blast radius.

---

# 12. Custom roles

Create permissions around job responsibilities.

For example:

```text id="q2v9n6"
ReportingRole
→ SELECT on reporting objects

DataEntryRole
→ INSERT/UPDATE on required tables

SupportRole
→ Limited operational permissions
```

This is much easier to manage than individual grants everywhere.

---

# 13. GRANT

`GRANT` provides permission.

Example:

```sql id="a8m4k7"
GRANT SELECT ON dbo.Orders
TO ReportingRole;
```

---

# 14. DENY

`DENY` explicitly blocks a permission.

For example:

```sql id="p5x1c8"
DENY DELETE ON dbo.Orders
TO ReportingRole;
```

But permission precedence can become complex when users belong to multiple roles.

Don't build a security model casually around layers of conflicting grants and denies.

---

# 15. REVOKE

`REVOKE` removes a previously granted or denied permission in the applicable security context.

Think:

```text id="n3q7v2"
GRANT
→ allow

DENY
→ explicitly block

REVOKE
→ remove that explicit permission assignment
```

---

# 16. Fixed database roles

SQL Server provides predefined database roles such as:

```text id="x6m2p9"
db_datareader
db_datawriter
db_owner
```

Be careful with broad roles.

For example:

```text id="j8r4c1"
db_owner
```

is extremely powerful.

---

# 17. db_datareader

Conceptually:

```text id="v5q9m3"
db_datareader
→ SELECT across user data
```

This can be useful for certain reporting scenarios, but it may be broader than necessary.

Least privilege still applies.

---

# 18. db_datawriter

Conceptually:

```text id="k1x7n4"
db_datawriter
→ INSERT
→ UPDATE
→ DELETE
```

Again, that's broad.

An application may only need:

```text id="p8m3r6"
INSERT on one table
UPDATE on two procedures
```

A custom role can be safer.

---

# 19. Stored procedures as a security boundary

Instead of granting direct table access:

```text id="q6v2c9"
Application
→ execute procedure
→ procedure accesses tables
```

This can reduce direct data access and provide a controlled interface.

For example:

```sql id="h3n8x5"
GRANT EXECUTE
ON dbo.CreateOrder
TO AppRole;
```

---

# 20. Views as security boundaries

A view can expose only required columns/rows.

Instead of:

```text id="m9p4k1"
SELECT *
FROM Customers;
```

you might expose:

```text id="r7c2v8"
CustomerId
CustomerName
City
```

while hiding sensitive fields.

---

# 21. Row-Level Security

SQL Server supports **Row-Level Security (RLS)**.

It allows access to rows based on the executing user's context.

Conceptually:

```text id="x4m8q2"
User A
→ sees Region A

User B
→ sees Region B
```

while both query the same table.

---

# 22. Why RLS is useful

RLS can help with multi-tenant or departmental data isolation.

For example:

```text id="c7p1n5"
Salesperson A
→ only their customers

Salesperson B
→ only their customers
```

The filtering is enforced at the database layer.

---

# 23. Dynamic Data Masking

Dynamic Data Masking can obscure sensitive values from users who don't need to see the full data.

Example concept:

```text id="u5k9m3"
Actual:
555-12-3456

Masked:
XXX-XX-3456
```

Important:

> **Dynamic Data Masking is not encryption and should not be treated as a complete security boundary.**

---

# 24. Encryption in transit

Data moving between client and SQL Server should be protected with encrypted connections where required.

Conceptually:

```text id="n2x6r8"
Application
   ↓
Encrypted connection
   ↓
SQL Server
```

This protects data from network interception.

---

# 25. Encryption at rest

Encryption at rest protects stored data.

SQL Server features can include:

```text id="m7q3c1"
Transparent Data Encryption (TDE)
Backup encryption
Always Encrypted
```

These solve different problems.

---

# 26. TDE

**Transparent Data Encryption** primarily protects database files and related storage from being read directly if someone obtains the underlying files.

Conceptually:

```text id="p4x8n2"
Database files
      ↓
Encrypted at rest
```

Applications generally don't need to change their queries just because TDE is enabled.

---

# 27. TDE does not solve everything

TDE doesn't mean:

> "Users can't read sensitive data."

If a user has legitimate database access:

```text id="w1m6k9"
SELECT SensitiveColumn
```

TDE doesn't inherently prevent that.

TDE primarily addresses protection of data at rest.

---

# 28. Always Encrypted

Always Encrypted is designed for scenarios where sensitive column values should remain encrypted beyond the database engine's ordinary access model.

Conceptually:

```text id="c5q9r2"
Application
   ↓
Encryption/decryption handling
   ↓
SQL Server stores ciphertext
```

It is especially relevant for highly sensitive columns.

---

# 29. Key management

Encryption is only as strong operationally as its key management.

Consider:

```text id="v8m3x6"
Where are keys stored?
Who can access them?
How are they rotated?
How are they backed up?
What happens during DR?
```

Losing required keys can make encrypted data unrecoverable.

---

# 30. Backup encryption

Backups can contain the entire database.

Therefore:

> **Protecting production data while leaving backups unprotected is a serious security gap.**

Backup encryption can help protect those files.

---

# 31. Auditing

Security isn't only about preventing access.

You may also need to answer:

> **"Who accessed or changed this data?"**

That's where auditing comes in.

---

# 32. SQL Server Audit

SQL Server Audit can capture security-related events and database activity depending on configuration.

For example:

```text id="n4x7p2"
Login activity
Permission changes
Object access
Data changes
Security events
```

The exact events captured depend on the audit configuration.

---

# 33. Audit vs application logging

Application logs may tell you:

```text id="k9m3v6"
User clicked "Update"
```

Database auditing can tell you:

```text id="q1x8c5"
Database principal
SQL statement/action
Object
Timestamp
```

For sensitive systems, both layers can be valuable.

---

# 34. Audit trail design

For important systems, think about:

```text id="r6p2n8"
Who?
What?
When?
Where?
Success/failure?
What changed?
```

This supports investigations and compliance requirements.

---

# 35. Sensitive data discovery

Before securing data, identify it.

Examples:

```text id="m5c8q1"
PII
Financial information
Credentials
Health-related information
Government identifiers
Business secrets
```

Then classify according to your organization's policies and applicable regulations.

---

# 36. Don't store secrets in plain text

Bad:

```text id="w7n4x2"
Password = 'MyPassword123'
```

inside application configuration, SQL scripts, or source control.

Use appropriate secret-management mechanisms.

---

# 37. Service accounts

Applications and jobs should use dedicated identities where practical.

For example:

```text id="p3k8m6"
Web application
→ AppIdentity

ETL process
→ ETLIdentity

Reporting service
→ ReportingIdentity
```

This improves traceability and limits permissions.

---

# 38. Credential separation

Don't use one giant credential for:

```text id="x6r2q9"
Application
Backups
Reporting
Maintenance
Deployment
Monitoring
```

Separate responsibilities when practical.

Then compromise of one identity doesn't automatically compromise everything.

---

# 39. SQL injection

Application-generated dynamic SQL can introduce SQL injection vulnerabilities.

Dangerous conceptual pattern:

```text id="v8m1c5"
"SELECT ... WHERE Name = '" + userInput + "'"
```

An attacker may manipulate the SQL syntax.

---

# 40. Parameterized queries

Prefer parameterized SQL:

```text id="j4q9n2"
SELECT ...
WHERE Name = @Name;
```

Then bind the value separately.

This is one of the most important application/database security practices.

---

# 41. Stored procedures don't automatically prevent SQL injection

A stored procedure can still be vulnerable if it constructs unsafe dynamic SQL internally.

For example:

```sql id="s7x2m8"
SET @sql = 'SELECT ... WHERE Name = ''' + @Name + '''';
EXEC(@sql);
```

Stored procedures are not magical security shields.

---

# 42. Dynamic SQL safely

When dynamic SQL is genuinely necessary, use appropriate parameterization mechanisms such as `sp_executesql`.

Conceptually:

```sql id="c9n4p6"
EXEC sys.sp_executesql
    @sql,
    N'@Name nvarchar(100)',
    @Name = @InputName;
```

---

# 43. Public-facing SQL Server

If SQL Server is exposed to networks outside the trusted application environment, security becomes even more important.

Minimize:

```text id="b5m8r1"
Network exposure
Open ports
Unused services
Excessive permissions
Weak authentication
```

Use appropriate network controls and encrypted connections.

---

# 44. Patch management

Security isn't only permissions.

Keep SQL Server and its underlying systems appropriately patched according to your organization's maintenance and security process.

An unpatched server can have vulnerabilities no permission model can fully compensate for.

---

# 45. Principle of defense in depth

A strong security design doesn't rely on one control.

Think:

```text id="q8c2m7"
Network controls
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
      ↓
Backup protection
```

If one layer fails, others still provide protection.

---

# 46. Production security review

A useful checklist:

```text id="m4x9p1"
□ Least privilege
□ No unnecessary sysadmin access
□ Strong authentication
□ Encrypted connections
□ Encryption at rest where required
□ Protected backups
□ Secure key management
□ Auditing
□ Parameterized queries
□ Secrets management
□ Patch management
□ Network restrictions
□ Security monitoring
```

---

# 47. Security and performance

Security controls can introduce overhead.

For example:

```text id="r2k7v5"
Encryption
Audit logging
Complex security predicates
```

can have performance implications.

But:

> **Don't disable security simply because it has a cost.**

Instead measure and design appropriately.

---

# 48. Security and HA/DR

Connect this with Lesson 148.

Your DR architecture must preserve security.

For example:

```text id="x6p1m8"
Primary
 ↓
Encrypted replication
 ↓
Secondary
```

And:

```text id="n3q8c4"
Primary backup
 ↓
Encrypted backup
 ↓
DR storage
```

Security must follow the data wherever it goes.

---

# 49. Security incident response

If a credential is compromised:

```text id="v5m2r9"
Identify
 ↓
Contain
 ↓
Disable/rotate credential
 ↓
Investigate audit trail
 ↓
Assess data exposure
 ↓
Recover
 ↓
Improve controls
```

Good auditing makes the investigation much easier.

---

# 50. Interview question

### "What's the difference between authentication and authorization?"

Strong answer:

> "Authentication establishes who the principal is. Authorization determines what that authenticated principal is permitted to do."

---

# 51. Interview question

### "What is least privilege?"

Strong answer:

> "Least privilege means giving a user, application, or service only the permissions necessary to perform its required tasks, minimizing the potential impact of credential compromise or misuse."

---

# 52. Interview question

### "Does TDE protect against SQL injection?"

No.

TDE primarily protects data at rest.

SQL injection is an application/query-construction vulnerability.

The appropriate defense is primarily:

```text id="k4r8m2"
Parameterized queries
+
Safe dynamic SQL
+
Appropriate application validation
```

---

# 53. Interview question

### "Does TDE replace backup encryption?"

No.

You should explicitly consider backup protection.

The database's at-rest encryption strategy and the security of backup files are related but distinct concerns.

---

# 54. Expert mental model

Think about every database identity like this:

```text id="z7m3q1"
WHO?
 ↓
Authentication

WHY?
 ↓
Business role

WHAT?
 ↓
Authorization

WHERE?
 ↓
Scope / row access

HOW?
 ↓
Encryption

WHAT HAPPENED?
 ↓
Audit
```

---

# 55. Lesson 149 takeaway

The security hierarchy to remember is:

```text id="p9x4c6"
AUTHENTICATION
      ↓
Who are you?

AUTHORIZATION
      ↓
What can you do?

ENCRYPTION
      ↓
Can unauthorized parties read the data?

AUDITING
      ↓
What happened?
```

And the golden principle:

> **Never give an application or user more database power than the business operation actually requires.**

---
