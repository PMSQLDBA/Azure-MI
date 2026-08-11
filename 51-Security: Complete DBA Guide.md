# Lesson 83 — Azure SQL Managed Instance Security: Complete DBA Guide

Security is one of the biggest differences between a **development SQL Server** and a **production SQL Server**.

A production DBA must answer:

> **Who can connect? What can they access? How is the data protected? How do we know what they did?**

For Azure SQL Managed Instance, think about security in **layers**:

```text
                    SQL MI SECURITY
                          |
       +------------------+------------------+
       |                  |                  |
   Identity           Network            Data
       |                  |                  |
Authentication       Private access      Encryption
Authorization        NSG/Firewall        TDE
RBAC                 DNS                 Key management
       |                  |                  |
       +------------------+------------------+
                          |
                    Monitoring/Audit
```

---

# 1. Authentication vs Authorization

This is the first concept you must master.

### Authentication

Answers:

> **"Who are you?"**

Examples:

```text
John
Application identity
Service principal
Managed identity
SQL login
```

### Authorization

Answers:

> **"What are you allowed to do?"**

Example:

```text
John
   ↓
Can SELECT
Cannot UPDATE
Cannot DROP
```

Remember:

```text
Authentication = WHO
Authorization  = WHAT
```

---

# 2. Real-world example

Imagine an application:

```text
E-commerce Application
        |
        ↓
    SQL MI
        |
        +---- Orders
        +---- Customers
        +---- Payments
```

You don't want every application connection to have:

```text
sysadmin
```

That would be extremely dangerous.

Instead:

```text
Application identity
       ↓
SQL user
       ↓
Required permissions only
```

This is **least privilege**.

---

# 3. SQL Authentication

Traditional SQL Server supports SQL logins such as:

```text
app_user
report_user
dba_user
```

Conceptually:

```text
Username + Password
        ↓
SQL MI
```

Example:

```text id="4v7d0n"
Application
    |
    | username/password
    v
SQL Managed Instance
```

This works, but password management creates operational and security challenges.

---

# 4. Microsoft Entra authentication

Azure environments strongly benefit from Microsoft Entra-based identity.

Conceptually:

```text
User/Application
      |
      ↓
Microsoft Entra ID
      |
      ↓
SQL Managed Instance
```

Instead of managing a separate SQL password for every identity, you can use centralized identity.

This enables things such as:

```text
Centralized identity
Conditional access
MFA for human users
Identity lifecycle management
Group-based access
```

---

# 5. Human user scenario

Suppose your DBA team has:

```text
Alice
Bob
Charlie
```

Instead of giving each person an independent SQL login and password, you can use an Entra security group:

```text
SQL-Production-DBA
       |
       +-- Alice
       +-- Bob
       +-- Charlie
```

Then grant the appropriate SQL permissions to the group.

This makes onboarding/offboarding much easier.

---

# 6. Employee leaves the company

Without centralized identity:

```text
Employee leaves
      ↓
Find every SQL login
      ↓
Disable/remove access
```

With Entra-based access:

```text
Employee leaves
      ↓
Identity disabled
      ↓
Access through group membership disappears
```

This is much easier to manage centrally.

---

# 7. Managed identity

This is extremely important for Azure applications.

Suppose:

```text
Azure App Service
       |
       ↓
SQL Managed Instance
```

A bad design might be:

```text
Application
    |
    +-- username
    +-- password
```

And then someone stores the password in:

```text
appsettings.json
```

or:

```text
source code
```

That's a security risk.

---

# 8. Managed identity concept

Instead:

```text
Azure Application
       |
Managed Identity
       |
       ↓
Azure identity platform
       |
       ↓
SQL MI
```

The application can authenticate using its Azure identity rather than embedding a long-lived password.

---

# 9. Why managed identity is powerful

Imagine a production application.

Without managed identity:

```text
Password
   ↓
Stored somewhere
   ↓
Rotated periodically
   ↓
Application configuration updated
```

With managed identity:

```text
Azure workload
      ↓
Managed identity
      ↓
Token-based authentication
      ↓
SQL MI
```

You eliminate a major category of secret-management problems.

---

# 10. System-assigned vs user-assigned managed identity

Azure managed identities generally come in two forms:

```text
System-assigned
User-assigned
```

### System-assigned

Identity lifecycle is tied to the Azure resource.

```text
Resource created
   ↓
Identity created

Resource deleted
   ↓
Identity deleted
```

### User-assigned

Identity is a separate Azure resource.

```text
Managed Identity
      |
      +---- Application A
      +---- Application B
```

This can be useful when the same identity needs to be associated with multiple Azure resources.

---

# 11. Application architecture

A common pattern:

```text
Azure App
    |
    | Managed Identity
    ↓
SQL Managed Instance
    |
    ↓
Application database
```

The SQL permissions should still be restricted.

Managed identity does **not** mean:

> "The application can do everything."

You still implement authorization.

---

# 12. Authentication + authorization together

Think:

```text
Managed Identity
      ↓
Authentication
      ↓
"Who is this?"
      ↓
SQL User
      ↓
Authorization
      ↓
"What can it do?"
```

---

# 13. RBAC vs SQL permissions

This confuses many DBAs.

### Azure RBAC

Controls access to **Azure resources**.

Examples:

```text
SQL Managed Instance resource
Azure configuration
Monitoring
Management operations
```

### SQL permissions

Control access **inside SQL Server**.

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

---

# 14. Example

Suppose I give someone Azure permissions to manage the SQL Managed Instance resource.

That doesn't automatically mean:

```text
SELECT * FROM dbo.Customers;
```

inside a database.

And the reverse is also true.

A person can have SQL permissions without being an Azure subscription administrator.

---

# 15. Mental model

```text
Azure layer
     |
     ↓
Azure RBAC
     |
     ↓
Can manage Azure resource

SQL layer
     |
     ↓
SQL permissions
     |
     ↓
Can access database objects
```

---

# 16. Login vs user

Another critical SQL Server concept.

A **login** is a server-level security principal.

A **user** is a database-level security principal.

Conceptually:

```text
SQL Instance
    |
    +---- Login
           |
           +---- Database User
                    |
                    +---- Permissions
```

---

# 17. Example

Suppose:

```text
Login = AppLogin
```

Database:

```text
SalesDB
```

contains:

```text
User = AppUser
```

Then:

```text
AppLogin
   ↓
AppUser
   ↓
SalesDB
   ↓
Permissions
```

---

# 18. Why this matters

You might have:

```text
Login
```

but no user in a particular database.

Then the login may successfully authenticate to SQL but still not have access to that database.

This leads to the classic:

> "Login works, but I can't access the database."

---

# 19. Least privilege

This is perhaps the most important security principle.

> **Give users and applications only the permissions they actually need.**

Bad:

```text
Application
   ↓
sysadmin
```

Better:

```text
Application
   ↓
SalesDB
   ↓
EXECUTE on required procedures
```

---

# 20. Why application should not be sysadmin

Imagine the application has:

```text
sysadmin
```

Then an application vulnerability could potentially become a database disaster.

An attacker could potentially:

```text
DROP DATABASE
ALTER security
Read sensitive data
Modify data
Create logins
```

The blast radius becomes enormous.

---

# 21. Better application security

Instead of:

```text
Application → sysadmin
```

use:

```text
Application
   ↓
Application user
   ↓
Specific database
   ↓
Specific schemas/objects
   ↓
Required permissions
```

---

# 22. Stored procedure approach

For sensitive applications, you may expose operations through stored procedures.

Example:

```sql
CREATE PROCEDURE dbo.CreateOrder
    @CustomerID INT,
    @Amount DECIMAL(18,2)
AS
BEGIN
    ...
END;
```

Application receives:

```text
EXECUTE
```

permission rather than broad direct table modification.

Conceptually:

```text
Application
    |
    ↓
EXECUTE procedure
    |
    ↓
Controlled database operation
```

This can reduce unnecessary direct table permissions.

---

# 23. Example permission design

Application needs:

```text
Read Products
Create Orders
Read Orders
```

Don't give:

```text
db_owner
```

Instead, consider permissions such as:

```text
SELECT on Products
EXECUTE on CreateOrder
SELECT on Orders
```

The exact permission model depends on your schema and security requirements.

---

# 24. Reporting user

Suppose the BI team needs read-only access.

Bad:

```text
report_user → db_datawriter
```

They don't need write access.

Better:

```text
report_user
     ↓
db_datareader
```

or more restrictive object/schema-level permissions when appropriate.

---

# 25. DBA access

A DBA may legitimately need elevated privileges.

But even DBA access should be:

```text
Controlled
Audited
Time-bound where possible
Approved
Monitored
```

Production access should not be casual.

---

# 26. Privileged access

Imagine:

```text
Developer
```

needs production access for 30 minutes.

Don't automatically give:

```text
Permanent db_owner
```

A stronger approach is:

```text
Request
  ↓
Approval
  ↓
Temporary access
  ↓
Perform work
  ↓
Access removed
  ↓
Audit
```

This reduces privilege accumulation.

---

# 27. Network security

Even perfect database permissions aren't enough if your SQL endpoint is unnecessarily exposed.

Think:

```text
Internet
   |
   X
SQL MI
```

You want controlled connectivity:

```text
Application VNet
      |
Private network
      |
SQL MI
```

---

# 28. Private connectivity

Azure SQL Managed Instance is designed for private network integration.

A typical architecture:

```text
                    Azure
                     |
          +----------+----------+
          |                     |
       App subnet          SQL MI subnet
          |                     |
          +------ Private ------+
                 connectivity
```

The goal is to avoid exposing database traffic unnecessarily to the public internet.

---

# 29. Network security layers

Think:

```text
User/Application
       |
       ↓
DNS
       |
       ↓
Private connectivity
       |
       ↓
Network security controls
       |
       ↓
SQL MI
       |
       ↓
SQL authentication
       |
       ↓
SQL authorization
```

Security should be layered.

---

# 30. Encryption in transit

When data travels:

```text
Application
      |
      | encrypted connection
      ↓
SQL MI
```

you want appropriate TLS protection.

This protects the communication channel from unauthorized interception.

---

# 31. Encryption at rest

Now consider:

```text
SQL MI
  ↓
Storage
```

If someone somehow gets physical access to the underlying storage, you don't want readable database files.

Encryption at rest protects stored data.

---

# 32. TDE

**Transparent Data Encryption (TDE)** protects SQL Server database files at rest through encryption.

Conceptually:

```text
Database
   ↓
Encryption
   ↓
Storage
```

The application generally doesn't have to rewrite SQL queries because of TDE.

That's why it's called **transparent**.

---

# 33. What TDE protects

TDE primarily addresses:

```text
Data at rest
```

It is not a replacement for:

```text
Authentication
Authorization
Network encryption
Auditing
Application security
```

---

# 34. TDE does NOT solve everything

Suppose:

```text
Attacker
   ↓
Valid application credentials
   ↓
SELECT * FROM Customers
```

TDE doesn't say:

> "This user isn't allowed to see the data."

The data is decrypted for authorized database operations.

So:

```text
TDE = protection of stored data
Permissions = protection of access
```

---

# 35. Key management

Encryption requires keys.

In Azure environments, key management can involve Azure Key Vault depending on the security architecture and requirements.

Think:

```text
SQL MI
  |
  ↓
Encryption
  |
  ↓
Key management
  |
  ↓
Key Vault / managed key architecture
```

The exact configuration should be designed according to your organization's security requirements.

---

# 36. Why Key Vault?

You don't want encryption keys handled casually.

Key management needs:

```text
Access control
Auditing
Rotation
Protection
Lifecycle management
```

A centralized key-management service helps enforce these controls.

---

# 37. Customer-managed keys

Some organizations have stronger compliance requirements.

They may want control over encryption keys rather than relying solely on service-managed keys.

This is where **customer-managed keys** can become relevant.

Conceptually:

```text
Company
   |
   ↓
Key Vault
   |
   ↓
Customer-managed key
   |
   ↓
Azure SQL encryption
```

---

# 38. Auditing

Security isn't only:

> "Can they access it?"

You also need:

> **"What did they do?"**

Auditing helps capture relevant database activity for investigation, compliance, and operational monitoring.

Example:

```text
User
 ↓
SELECT
 ↓
Sensitive table
```

You may want evidence that the action occurred.

---

# 39. Real-world audit scenario

Security team asks:

> "Who accessed the Customer SSN data yesterday?"

Without auditing:

```text
???
```

With appropriate auditing and monitoring:

```text
User
↓
Time
↓
Database
↓
Activity
```

You can investigate.

---

# 40. Security incident

Suppose:

```text
3:00 AM
```

Someone accesses a sensitive table.

Your monitoring detects:

```text
Unexpected login
+
Unexpected source
+
Sensitive query
```

Now security can investigate.

This is why:

```text
Authentication
+
Authorization
+
Auditing
```

must work together.

---

# 41. Microsoft Defender for SQL

Microsoft Defender for SQL provides additional security capabilities around SQL workloads.

It can help with areas such as:

```text
Threat detection
Security alerts
Vulnerability assessment
Security posture
```

It's an additional security layer—not a replacement for good SQL permissions.

---

# 42. Vulnerability assessment

A vulnerability assessment can identify security weaknesses such as:

```text
Weak configurations
Excessive permissions
Potential security issues
```

Think of it as:

```text
Security scanner
      ↓
Find weaknesses
      ↓
Remediate
```

---

# 43. Security baseline

A production SQL MI should have a baseline.

For example:

```text
☑ Strong authentication
☑ Entra integration
☑ Least privilege
☑ Private connectivity
☑ Encryption
☑ Auditing
☑ Monitoring
☑ Backup protection
☑ Key management
☑ Vulnerability assessment
```

---

# 44. Real scenario — developer needs read-only access

This is a classic interview question.

Requirement:

> "Developer needs read-only access to Production SalesDB."

Don't give:

```text
db_owner
```

Instead:

```text
Developer
   ↓
Entra group
   ↓
SQL user
   ↓
SalesDB
   ↓
Read-only permissions
```

For broad read-only access, a suitable database role can be used. For tighter control, grant only the required schemas/objects.

---

# 45. Why groups are better

Suppose 20 developers need access.

Bad:

```text
Developer1 → SQL permission
Developer2 → SQL permission
Developer3 → SQL permission
...
```

Better:

```text
SQL-Prod-Sales-ReadOnly
       |
       +-- Dev1
       +-- Dev2
       +-- Dev3
       ...
```

Then:

```text
Group
  ↓
SQL permissions
```

Access becomes much easier to manage.

---

# 46. Employee leaves

If:

```text
Developer3
```

leaves the company:

```text
Remove Developer3
from Entra group
```

You don't need to hunt through every SQL database looking for individual permissions if access was designed centrally.

---

# 47. Application secret problem

Bad design:

```text
appsettings.json

Password=SuperSecretPassword123
```

Even worse:

```text
Git repository
   ↓
Password
```

Now potentially:

```text
Developers
CI/CD
Git history
Build agents
```

may expose the credential.

Avoid storing long-lived secrets in source code.

---

# 48. Better approach

Use:

```text
Managed Identity
```

where supported.

Or use a secure secret-management system such as:

```text
Azure Key Vault
```

for secrets that genuinely need to exist.

---

# 49. Secret rotation

Suppose your application uses a SQL password.

Security policy says:

```text
Rotate every 90 days
```

You now have:

```text
Password rotation
       ↓
Update secret store
       ↓
Application picks up new secret
       ↓
Connection validated
```

This introduces operational complexity.

Managed identity can eliminate many password-based authentication scenarios.

---

# 50. Security architecture example

A strong production architecture could look conceptually like:

```text
                         Users
                           |
                    Microsoft Entra ID
                           |
                    Authentication
                           |
                           ↓
                 +-------------------+
                 |   Application     |
                 +-------------------+
                           |
                    Managed Identity
                           |
                    Private Network
                           |
                           ↓
                 +-------------------+
                 |   SQL MI          |
                 +-------------------+
                    |      |      |
                    |      |      |
                  TDE    Audit   Permissions
                    |
                Key management
```

---

# 51. Another real scenario — reporting team

Business Intelligence team says:

> "We need access to production data."

Don't immediately give them:

```text
db_owner
```

Ask:

```text
What data?
Which tables?
Read only?
How frequently?
Can sensitive columns be exposed?
```

Maybe the correct design is:

```text
Reporting identity
      ↓
Read-only role
      ↓
Reporting schema
      ↓
Approved objects
```

---

# 52. Sensitive data

Suppose you have:

```text
Customers
```

with:

```text
Name
Email
Phone
SSN
CreditCard
```

A reporting user may need:

```text
Name
Email
```

but not:

```text
SSN
CreditCard
```

Don't give broad table access if it exposes sensitive information unnecessarily.

---

# 53. Data classification

A mature environment classifies data:

```text
Public
Internal
Confidential
Highly sensitive
```

Then security controls are aligned to the classification.

For SQL Server workloads, this can influence:

```text
Permissions
Encryption
Auditing
Masking
Access review
Retention
```

---

# 54. Defense in depth

The biggest security principle:

> **Never depend on one security control.**

For example:

```text
Private network
      +
Strong identity
      +
Least privilege
      +
Encryption
      +
Auditing
      +
Monitoring
      +
Backups
```

If one layer fails, other layers still provide protection.

---

# 55. Common DBA security mistakes

### Mistake 1

Giving applications:

```text
sysadmin
```

### Mistake 2

Sharing one DBA account.

### Mistake 3

Putting passwords in source control.

### Mistake 4

Giving everyone `db_owner`.

### Mistake 5

No auditing.

### Mistake 6

Allowing unnecessary network exposure.

### Mistake 7

Never reviewing permissions.

### Mistake 8

Never testing security controls.

---

# 56. Permission review

A production DBA should periodically ask:

```text
Who has access?
Why?
Do they still need it?
What can they do?
When was access last used?
```

This is particularly important for:

```text
Developers
Contractors
Service accounts
DBAs
Automation identities
```

---

# 57. Break-glass access

Production environments sometimes need emergency access.

For example:

```text
Production incident
       ↓
Normal DBA unavailable
       ↓
Emergency privileged account
```

This should be:

```text
Highly protected
Audited
Rarely used
Monitored
Tested
```

Not:

```text
Everyone knows the password
```

---

# 58. SQL injection

Security isn't only Azure configuration.

Suppose the application constructs SQL like:

```text
"SELECT * FROM Users WHERE Name = '" + userInput + "'"
```

An attacker may manipulate the input.

Better application design uses:

```text
Parameterized queries
```

or safe ORM/query mechanisms.

The DBA should understand this because SQL security is shared between:

```text
Application
+
Database
```

---

# 59. Production incident example

Imagine an attacker compromises an application credential.

They can connect to SQL MI.

What happens next depends heavily on your design.

### Bad design:

```text
Application identity
       ↓
db_owner
```

Attacker potentially has broad database control.

### Better design:

```text
Application identity
       ↓
Limited permissions
       ↓
Specific database/schema/procedures
```

The blast radius is much smaller.

---

# 60. Interview question

### "What is the difference between Azure RBAC and SQL permissions?"

Answer:

> "Azure RBAC controls management access to Azure resources, while SQL permissions control access to databases and database objects. Having Azure permissions on the Managed Instance does not automatically grant unrestricted access to data inside SQL databases."

---

# 61. Interview question

### "What is a managed identity?"

Answer:

> "A managed identity is an Azure-managed identity that allows supported workloads to authenticate to Azure resources without storing long-lived credentials in application code or configuration."

---

# 62. Interview question

### "Why would you prefer managed identity over SQL authentication for an Azure application?"

Strong answer:

> "Managed identity reduces the need to store and rotate long-lived passwords, integrates with Azure identity, and allows access to be controlled through identity and SQL permissions."

---

# 63. Interview question

### "What is TDE?"

Answer:

> "Transparent Data Encryption encrypts database data at rest. It protects the underlying database storage, but it doesn't replace authentication, authorization, network security, or auditing."

---

# 64. Interview question

### "A developer needs read-only access to one production database. What would you do?"

Excellent answer:

> "I'd use centralized identity such as a Microsoft Entra group where appropriate, create or map a SQL user for that identity, and grant only the required read permissions on the target database. I would avoid `db_owner` or server-level administrative permissions, and I'd review and audit the access periodically."

---

# 65. Interview question

### "How would you secure SQL MI from the network?"

Think:

```text
Private connectivity
+
VNet architecture
+
Network security controls
+
Restricted access
+
DNS
+
TLS
```

Answer:

> "I'd keep database connectivity private where the architecture supports it, restrict network paths to only required application and administrative sources, use appropriate network security controls, and require encrypted connections."

---

# 66. Your production security checklist

```text
IDENTITY
☐ Microsoft Entra authentication
☐ Managed identities
☐ Strong authentication
☐ Group-based access

AUTHORIZATION
☐ Least privilege
☐ Database roles
☐ Schema/object permissions
☐ No unnecessary db_owner
☐ No unnecessary sysadmin

NETWORK
☐ Private connectivity
☐ Restricted network paths
☐ Secure DNS
☐ TLS

DATA
☐ TDE
☐ Key management
☐ Sensitive-data controls

MONITORING
☐ Auditing
☐ Defender/security monitoring
☐ Vulnerability assessment
☐ Access reviews

OPERATIONS
☐ Secret management
☐ Rotation
☐ Break-glass procedure
☐ Periodic access review
☐ Security incident runbook
```

---

# 67. The complete security mental model

When someone asks:

> **"How do you secure Azure SQL Managed Instance?"**

Don't answer with just:

> "Enable encryption."

Think in this order:

```text
                 SECURITY
                    |
        +-----------+-----------+
        |           |           |
     Identity    Network      Data
        |           |           |
     Entra       Private       TDE
     Managed     access        Keys
     Identity
        |
   Authorization
        |
   SQL permissions
        |
     Auditing
        |
    Monitoring
```

That's the production DBA answer.

---

## Next — Lesson 84: Azure SQL Managed Instance Networking

This is a **very important DBA + Azure topic**, especially for interviews.

We'll cover:

* VNet
* Subnets
* Delegated subnet concepts
* NSGs
* Route tables
* DNS
* Private connectivity
* Ports
* Application → SQL MI connectivity
* On-premises → SQL MI
* VPN
* ExpressRoute
* Private DNS
* Connectivity troubleshooting
* `Test-NetConnection`
* DNS troubleshooting
* Login timeout vs authentication failure
* Real production incidents
* and the classic interview scenario:

> **"The application cannot connect to SQL Managed Instance. SQL is healthy. Walk me through your troubleshooting process from DNS to TCP to authentication."**
