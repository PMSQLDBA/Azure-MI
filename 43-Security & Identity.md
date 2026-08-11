# Lesson 75 — Security & Identity in Azure SQL Managed Instance

This lesson is extremely important for **Azure SQL Managed Instance + SQL Server DBA** roles.

The biggest thing to understand is that Azure security has **multiple layers**.

A common mistake is to think:

> "I gave the user Azure RBAC, so they can access the database."

Not necessarily.

---

# 1. The security layers

Think about access to Azure SQL MI like this:

```text
                    User / Application
                           |
                           v
                    Azure / Network
                           |
             +-------------+-------------+
             |                           |
         Azure RBAC                Network controls
             |                           |
             +-------------+-------------+
                           |
                           v
                    SQL Managed Instance
                           |
                    Authentication
                           |
                           v
                    Authorization
                           |
                           v
                     Database
                           |
                    Objects / Data
```

There are several separate questions:

```text
1. Can I reach the server?
2. Who am I?
3. Can SQL Server authenticate me?
4. What am I allowed to do?
5. What data can I access?
```

---

# 2. Authentication vs authorization

Memorize this:

> **Authentication = Who are you?**

> **Authorization = What are you allowed to do?**

Example:

```text
Alice
 ↓
Authentication
 ↓
"Yes, this is Alice."
 ↓
Authorization
 ↓
"She can SELECT Orders."
```

But perhaps:

```text
Alice
 ↓
DELETE Orders
 ↓
DENIED
```

because she doesn't have DELETE permission.

---

# 3. Two major authentication approaches

Azure SQL Managed Instance supports SQL Server authentication as well as Microsoft Entra authentication.

Conceptually:

```text
SQL Authentication
       |
       +-- SQL login
       +-- Password

Microsoft Entra
       |
       +-- User
       +-- Group
       +-- Managed identity
       +-- Service principal scenarios
```

---

# 4. SQL Authentication

Traditional SQL Server model:

```text
Username
+
Password
```

Example:

```text id="5a7c3d"
Application
    |
    | username/password
    v
SQL Managed Instance
```

This can work, but credential management becomes an operational concern.

---

# 5. Microsoft Entra authentication

Microsoft Entra ID provides identity-based authentication.

Conceptually:

```text id="8n4k2p"
Application/User
       |
       v
Microsoft Entra ID
       |
       | token
       v
Azure SQL MI
```

The database validates the identity/token rather than requiring a traditional SQL password for that authentication flow.

---

# 6. Why organizations prefer identity-based authentication

Imagine 200 applications.

Traditional approach:

```text id="r3m8x1"
App 1 → password
App 2 → password
App 3 → password
...
App 200 → password
```

Now you have:

```text id="j7q2m5"
Password rotation
Secret storage
Credential leakage risk
Expiration problems
Operational overhead
```

Identity-based access can reduce the need to distribute long-lived database passwords.

---

# 7. Managed Identity

This is one of the most important Azure concepts.

Suppose you have:

```text id="m6p1x8"
Azure App Service
       |
       |
       v
Azure SQL Managed Instance
```

Instead of storing:

```text id="c9r4n2"
SQL username
SQL password
```

the application can use an Azure managed identity where supported.

Conceptually:

```text id="v5q8k3"
Application
     |
     | Managed Identity
     v
Microsoft Entra ID
     |
     | Token
     v
SQL MI
```

---

# 8. Why Managed Identity is powerful

The application doesn't need to carry a database password around.

That can eliminate a class of problems such as:

```text id="x2m7p4"
Password stored in config
Password stored in source code
Password accidentally logged
Password rotation
Secret expiration
```

It doesn't eliminate security work, but it changes the model.

---

# 9. User-assigned vs system-assigned identity

Azure managed identities generally come in two forms.

### System-assigned

Identity is tied to the Azure resource.

Conceptually:

```text id="q8n3r6"
App Service
    |
    +-- System-assigned identity
```

If the resource is deleted, the identity lifecycle is tied to that resource.

---

### User-assigned

Identity is a separate Azure resource.

```text id="k4m9x2"
Managed Identity
      |
      +---- App A
      |
      +---- App B
```

This can be useful when multiple resources need the same identity.

---

# 10. Example: production application

Imagine:

```text id="z7p3c5"
Azure App Service
       |
       v
Order API
       |
       v
SQL MI
```

Security requirement:

> "The Order API should read and update Orders, but it must not access Payroll."

You could create an identity-based database user and grant only the required permissions.

Conceptually:

```text id="m2x8q4"
Order API identity
       |
       v
SQL user
       |
       +---- SELECT Orders
       +---- INSERT Orders
       +---- UPDATE Orders
       |
       X---- Payroll
```

That's least privilege.

---

# 11. Azure RBAC vs SQL permissions

This is one of the most important distinctions.

Azure RBAC controls access to Azure resources.

Examples:

```text id="r5n1v8"
SQL Managed Instance resource
Networking
Monitoring
Azure configuration
```

SQL permissions control access **inside SQL Server**.

Examples:

```text id="c7m3q9"
SELECT
INSERT
UPDATE
DELETE
EXECUTE
ALTER
CONTROL
```

So:

```text id="x8p2k6"
Azure RBAC
      ≠
SQL database permissions
```

---

# 12. Example

Suppose you give someone an Azure role allowing them to manage the SQL Managed Instance resource.

That doesn't automatically mean:

```sql id="5gk1qz"
SELECT * FROM CustomerData;
```

will work.

They still need appropriate database authentication and authorization.

---

# 13. Server-level vs database-level permissions

SQL Server has multiple permission scopes.

Conceptually:

```text id="y4n8m2"
SQL Managed Instance
       |
       +-- Server-level
       |
       +-- Database
             |
             +-- Schema
                   |
                   +-- Object
```

Examples:

```text id="q9r3x7"
Server-level:
LOGIN
ALTER ANY DATABASE
```

Database-level:

```text id="m5p8c1"
CONNECT
CREATE TABLE
SELECT
EXECUTE
```

Object-level:

```text id="v2k7n4"
SELECT on dbo.Orders
```

---

# 14. Least privilege

The rule:

> **Give the minimum permissions required to perform the job.**

Bad:

```text id="s8x4m2"
Application
   ↓
db_owner
```

Better:

```text id="j6p1q9"
Application
   ↓
Specific database user
   ↓
Specific role
   ↓
Required permissions
```

---

# 15. Why `db_owner` is dangerous

Suppose an application only needs:

```text id="f3m7x2"
SELECT
INSERT
UPDATE
```

But you give:

```text id="c9q5n8"
db_owner
```

Now the application may have far more control than necessary.

If the application is compromised:

```text id="r4x8p1"
Attacker
 ↓
Application credentials
 ↓
db_owner
 ↓
Huge database impact
```

Least privilege reduces blast radius.

---

# 16. Database roles

Instead of assigning permissions individually to every user, use roles.

Conceptually:

```text id="n7m2c5"
OrderAppRole
   |
   +-- SELECT Orders
   +-- INSERT Orders
   +-- UPDATE Orders
```

Then:

```text id="q8x3v1"
Order API identity
      ↓
OrderAppRole
```

This is much easier to manage.

---

# 17. Example SQL

Conceptually:

```sql id="j4n8r2"
CREATE ROLE OrderAppRole;

GRANT SELECT, INSERT, UPDATE
ON dbo.Orders
TO OrderAppRole;
```

Then assign the appropriate database principal to that role.

The exact identity/user creation mechanism depends on whether you're using SQL authentication or Microsoft Entra authentication.

---

# 18. Schema-level permissions

Suppose an application needs access to all objects under:

```text id="p6x2m9"
Sales
```

Instead of granting permissions object by object, you can sometimes grant at schema scope.

Conceptually:

```text id="w3r8q5"
Sales schema
    |
    +-- Orders
    +-- Customers
    +-- Invoices
```

Then permissions can be managed at the appropriate scope.

This reduces administration overhead.

---

# 19. Contained database users

A contained database user can be associated directly with a database rather than requiring a traditional server-level login model for every scenario.

This is especially useful in modern identity-oriented architectures.

Conceptually:

```text id="k5m1x7"
Database
   |
   +-- User
   +-- Permissions
```

This can simplify certain database access models and migrations.

---

# 20. Application security scenario

Imagine a developer says:

> "The application can't connect to SQL MI. Give the application `sysadmin`."

🚨 Stop.

That's not a troubleshooting strategy.

First investigate:

```text id="v8q2n4"
Network?
DNS?
Authentication?
Token?
Identity?
Database user?
Permissions?
Connection string?
Firewall/private connectivity?
```

---

# 21. Real production incident

At 10 AM:

```text id="z3m7p1"
Order API
     ↓
SQL MI
```

works.

At 10:30 AM:

```text id="c8x4q6"
Deployment
```

After deployment:

```text id="n5r2m9"
Login failed
```

Developer says:

> "Database is down."

You investigate.

---

# 22. Step 1 — Check SQL MI availability

SQL MI is healthy.

So:

```text id="m4p8x2"
Database infrastructure
       ✓
```

---

# 23. Step 2 — Check network

Application can reach the SQL MI endpoint.

```text id="q7n3v6"
Network
       ✓
```

---

# 24. Step 3 — Check authentication

Application now uses a managed identity.

Previously it used:

```text id="w2k9c5"
SQL username/password
```

Deployment changed the authentication model.

Now:

```text id="r6x1m8"
Managed Identity
       ↓
Microsoft Entra
       ↓
SQL MI
```

---

# 25. Step 4 — Check database principal

The managed identity exists in Azure.

But does SQL Server recognize it appropriately?

You discover:

```text id="j3p7n4"
Database user/permissions
       ✗
```

The identity doesn't have the expected database access.

---

# 26. Root cause

Not:

> "Azure SQL MI is down."

Actual root cause:

```text id="v9m2x5"
Application authentication model changed
        ↓
Managed identity used
        ↓
Database authorization not configured correctly
        ↓
Login/access failure
```

This is a classic cloud DBA troubleshooting scenario.

---

# 27. Encryption at rest

Azure SQL Managed Instance provides encryption capabilities for data at rest.

A major concept is:

> **Transparent Data Encryption (TDE)**

TDE encrypts database files/logs at rest.

Conceptually:

```text id="x4n8q2"
Database files
      ↓
Encryption
      ↓
Storage
```

If someone obtains the underlying storage media, the database files aren't simply sitting there as plaintext database files.

---

# 28. TDE vs application encryption

Don't confuse them.

### TDE

Protects database storage at rest.

### Application/column-level encryption

Can protect sensitive values more specifically.

For example:

```text id="m7r3c9"
SSN
Credit card information
Highly sensitive business fields
```

The security requirement determines the appropriate mechanism.

---

# 29. Encryption in transit

Data traveling between application and SQL MI should be protected using encrypted connections.

Conceptually:

```text id="p2x6n8"
Application
     |
     | Encrypted connection
     v
SQL MI
```

Think:

```text id="j5m9q3"
At rest
   ↓
Storage protection

In transit
   ↓
Network protection
```

---

# 30. Azure Key Vault

Organizations often use Azure Key Vault to centrally manage cryptographic keys, secrets, and certificates.

Conceptually:

```text id="c8r2v6"
Key Vault
    |
    +-- Keys
    +-- Secrets
    +-- Certificates
```

Applications and Azure services can integrate with Key Vault according to the supported security architecture.

---

# 31. Why centralized key management matters

Without centralized management:

```text id="n4p7x1"
App A → secret
App B → secret
App C → secret
```

Harder to govern.

With centralized management:

```text id="w8m3q5"
               Key Vault
              /    |    \
            App A App B App C
```

You can apply stronger governance and lifecycle management.

---

# 32. Private connectivity

For enterprise environments, SQL MI is commonly deployed with private network connectivity rather than exposing database traffic unnecessarily to the public internet.

Think:

```text id="r6k2m9"
VNet
 |
 +---- Application
 |
 +---- SQL Managed Instance
```

rather than:

```text id="p4x8c1"
Internet
   |
   ↓
Database
```

Network architecture is a major part of database security.

---

# 33. Network security vs database security

These are different layers.

### Network security

Controls:

```text id="y7n3m5"
Who can reach the endpoint?
```

### Database security

Controls:

```text id="q2r8v6"
What can the authenticated identity do?
```

You need both.

---

# 34. Defense in depth

A mature design might look like:

```text id="m9x4p2"
Internet
   X
   |
Private networking
   |
Network controls
   |
SQL MI
   |
Authentication
   |
Authorization
   |
Database roles
   |
Object permissions
   |
Encryption
   |
Auditing/monitoring
```

Each layer reduces risk.

---

# 35. Auditing

You need to know:

> **Who did what?**

For example:

```text id="c5v8n1"
Who accessed Orders?
Who changed permissions?
Who modified sensitive data?
When?
From where?
```

Auditing helps with:

```text id="r7m2x9"
Security investigations
Compliance
Troubleshooting
Accountability
```

---

# 36. Defender for SQL

Microsoft Defender for SQL can provide additional security capabilities and threat detection around SQL workloads.

Conceptually:

```text id="p3q9m6"
SQL MI
  |
  +-- Security monitoring
  +-- Threat detection
  +-- Recommendations
```

Don't confuse security monitoring with database authorization.

They solve different problems.

---

# 37. Common security mistake #1

> "Everyone uses the same SQL login."

Bad.

You lose:

```text id="h8x4m2"
Individual accountability
Granular permissions
Easy offboarding
```

Prefer identity-based access and appropriately scoped database principals where feasible.

---

# 38. Common security mistake #2

> "The application needs database access, so give it db_owner."

Bad.

Use:

```text id="j6r1n8"
Specific role
   ↓
Specific permissions
```

---

# 39. Common security mistake #3

> "Store the SQL password in appsettings.json."

Potentially dangerous.

Better architecture can use:

```text id="v3m7q2"
Managed Identity
       ↓
Microsoft Entra authentication
```

or, where secrets are required:

```text id="x8p4c6"
Key Vault
       ↓
Secret retrieval
```

---

# 40. Common security mistake #4

> "Security is the Azure administrator's job."

No.

DBAs still need to understand:

```text id="n5k2m9"
Database permissions
Roles
Schemas
Authentication
Encryption
Auditing
Application identities
```

---

# 41. Common security mistake #5

Giving permissions directly to individuals everywhere.

Instead:

```text id="q7r3x8"
Users
 ↓
Groups/Roles
 ↓
Permissions
```

This makes access management more scalable.

---

# 42. Production scenario — developer access

A developer needs:

```text id="z2m8p4"
SELECT
```

on:

```text id="c6x1q9"
Sales.Orders
```

Don't give:

```text id="r5n3v7"
db_owner
```

Instead use a role with the required permission:

```text id="j8m4q2"
DeveloperReadOnly
       |
       +-- SELECT Sales.Orders
```

Now if the developer leaves:

```text id="w3p7x9"
Remove user from role
```

rather than hunting through dozens of direct grants.

---

# 43. Production scenario — application identity

Requirement:

> "Order API needs to execute stored procedures but shouldn't directly update tables."

Excellent security model:

```text id="k4n9m2"
Order API Identity
        ↓
Database User
        ↓
Application Role
        ↓
EXECUTE
        ↓
Stored Procedures
```

No direct table modification.

This can significantly reduce application blast radius.

---

# 44. Security troubleshooting flow

When an application cannot access SQL MI:

```text id="v6x2p8"
1. Is SQL MI healthy?
        ↓
2. Can application reach it?
        ↓
3. Is DNS working?
        ↓
4. Is authentication succeeding?
        ↓
5. Which identity is being used?
        ↓
6. Does database user exist?
        ↓
7. Does user have required role?
        ↓
8. Does role have required permission?
        ↓
9. Is encryption/connection configuration correct?
        ↓
10. Check audit/error logs
```

This prevents random permission escalation.

---

# 45. Interview question

### "What's the difference between Azure RBAC and SQL permissions?"

Strong answer:

> "Azure RBAC controls access to Azure resources and management operations, while SQL permissions control what an authenticated principal can do inside the SQL Server engine and databases. Having Azure permissions doesn't automatically grant database-level permissions."

---

# 46. Interview question

### "Why use Managed Identity?"

Answer:

> "Managed Identity allows an Azure resource to authenticate using an identity managed by Azure, reducing the need to store and rotate long-lived credentials in application configuration. The identity still needs appropriate authorization in the target service."

---

# 47. Interview question

### "What is least privilege?"

Answer:

> "Grant only the permissions required for a user, application, or service to perform its intended function, minimizing the potential impact if the identity is compromised."

---

# 48. Interview question

### "Does TDE protect against a malicious SQL user?"

No.

TDE protects data at rest.

If an authenticated user has:

```text id="p8m3x6"
SELECT
```

they can still query the decrypted database through SQL Server.

Think:

```text id="y4q9n2"
TDE
 ↓
Storage protection

Permissions
 ↓
User access control
```

Different layers.

---

# 49. The security mental model

Memorize:

```text id="a7m3x8"
             SECURITY
                 |
      +----------+----------+
      |          |          |
   Network   Authentication Authorization
      |          |          |
 Private      Who are you? What can you do?
 connectivity
                 |
                 v
             Encryption
                 |
          +------+------+
          |             |
       At rest       In transit
          |
       Auditing
          |
       Monitoring
```

---

# 50. Final production scenario

Imagine this architecture:

```text id="q6n1r9"
Users
  |
  v
Web Application
  |
  v
Managed Identity
  |
  v
Microsoft Entra
  |
  v
Private Network
  |
  v
Azure SQL Managed Instance
  |
  +-- Database
       |
       +-- Application Role
             |
             +-- EXECUTE
                  |
                  +-- Stored Procedures
```

This is much stronger than:

```text id="w3p8m2"
Application
   |
   | admin/password
   v
SQL MI
   |
   +-- db_owner
```

The second architecture may work technically, but it creates a much larger security blast radius.

---

## The key lesson

When someone says:

> **"The application needs access to SQL Managed Instance."**

Your DBA brain should immediately ask:

```text
Who is the identity?
       ↓
How is it authenticated?
       ↓
How does it reach SQL MI?
       ↓
Which database?
       ↓
Which schema?
       ↓
Which objects?
       ↓
Which operations?
       ↓
What is the minimum required permission?
       ↓
How is access audited?
```

That is **production-grade Azure SQL security thinking**.

### Next — Lesson 76: Backup, Point-in-Time Restore & Disaster Recovery

We'll go deep into:

* Full, differential, and log backups
* Azure SQL MI automated backups
* Point-in-time restore
* Long-term retention
* Recovery scenarios
* accidental DELETE
* dropped database
* corruption
* ransomware scenario
* restoring to another Managed Instance
* RPO/RTO calculations
* backup validation
* and a complete **"production database was accidentally deleted at 2:37 PM"** incident walkthrough.
