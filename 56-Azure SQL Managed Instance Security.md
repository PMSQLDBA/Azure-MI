# Lesson 88 — Azure SQL Managed Instance Security

Security is one of the most important areas for an Azure SQL Managed Instance DBA.

A production DBA needs to think about security at **multiple layers**, not just usernames and passwords.

```text id="sec001"
                    SQL MI SECURITY
                          |
       +------------------+------------------+
       |                  |                  |
    Network          Authentication     Authorization
       |                  |                  |
    Private VNet       Who are you?      What can you do?
       |                  |                  |
    Firewall          Entra ID/SQL      Roles/Permissions
       |
       +------------------+
       |
    Data Protection
       |
    Encryption / TDE / Always Encrypted
       |
    Monitoring
       |
    Auditing / Defender
```

---

# 1. Authentication vs Authorization

This is the first concept you must master.

## Authentication

Answers:

> **Who are you?**

Examples:

```text id="sec002"
Username/password
Microsoft Entra identity
Managed identity
```

---

## Authorization

Answers:

> **What are you allowed to do?**

Examples:

```text id="sec003"
SELECT
INSERT
UPDATE
DELETE
EXECUTE
ALTER
CREATE
```

So:

```text id="sec004"
Authentication
      ↓
Who are you?
      ↓
Authorization
      ↓
What can you do?
```

---

# 2. Real-world example

Imagine an application called:

```text id="sec005"
OrderAPI
```

It needs to execute:

```sql id="sec006"
SELECT *
FROM Orders
WHERE CustomerID = @CustomerID;
```

Authentication:

```text id="sec007"
OrderAPI → Microsoft Entra identity
```

Authorization:

```text id="sec008"
OrderAPI → SELECT permission on Orders
```

The application should **not** automatically receive:

```text id="sec009"
DROP DATABASE
ALTER SERVER
CREATE LOGIN
```

This is least privilege.

---

# 3. SQL Authentication

Traditional SQL Server authentication uses:

```text id="sec010"
Login
  +
Password
```

Example conceptually:

```text id="sec011"
Application
    |
    ↓
SQL username
    +
password
    |
    ↓
SQL MI
```

This works, but creates credential-management challenges.

---

# 4. Problem with passwords

Suppose your application stores:

```text id="sec012"
SQLUser = appuser
Password = P@ssw0rd...
```

Where does it live?

Possibilities:

```text id="sec013"
appsettings.json
Environment variable
Configuration file
CI/CD variable
Developer machine
Source repository
```

That's risky.

A leaked password can become a major security incident.

---

# 5. Better approach: Microsoft Entra authentication

Microsoft Entra ID provides identity-based authentication.

Conceptually:

```text id="sec014"
Application
    |
    ↓
Microsoft Entra ID
    |
    ↓
Token
    |
    ↓
SQL MI
```

Instead of sending a permanent SQL password, the application can authenticate using an identity/token-based mechanism.

---

# 6. Managed Identity

This is one of the most important Azure concepts.

A managed identity gives an Azure resource an identity that can be used to authenticate to supported services.

For example:

```text id="sec015"
Azure App Service
      |
      ↓
Managed Identity
      |
      ↓
Microsoft Entra ID
      |
      ↓
SQL MI
```

No application-managed password is required for that identity.

---

# 7. Real production scenario

Your application runs on:

```text id="sec016"
Azure App Service
```

It needs access to:

```text id="sec017"
SQL Managed Instance
```

Old design:

```text id="sec018"
App
 |
 +-- SQL username
 +-- SQL password
 |
 ↓
SQL MI
```

Better design:

```text id="sec019"
App Service
     |
     ↓
Managed Identity
     |
     ↓
Microsoft Entra
     |
     ↓
SQL MI
```

This is a major security improvement.

---

# 8. Why managed identity is useful

You avoid having to manually manage:

```text id="sec020"
Password creation
Password rotation
Password storage
Password exposure
Password expiration
```

Azure manages the identity lifecycle for the Azure resource.

---

# 9. System-assigned vs user-assigned identity

There are two major managed identity types.

### System-assigned

Identity is tied to the Azure resource.

Conceptually:

```text id="sec021"
App Service
   |
   +---- System-assigned identity
```

If the resource is deleted, the identity is also removed.

---

### User-assigned

Identity exists as a separate Azure resource.

```text id="sec022"
User-assigned identity
        |
        +------ App A
        |
        +------ App B
        |
        +------ App C
```

This can be useful when multiple resources need the same identity.

---

# 10. When would you choose user-assigned identity?

Suppose you have:

```text id="sec023"
Application A
Application B
Application C
```

All need access to the same SQL MI resources.

You might create:

```text id="sec024"
Shared managed identity
```

and assign it to the required resources.

But don't automatically share identities everywhere.

Least privilege still applies.

---

# 11. Microsoft Entra users in SQL MI

Microsoft Entra authentication doesn't mean:

> "Everyone in the company can access SQL MI."

You still need to create appropriate database principals/users and grant permissions.

Conceptually:

```text id="sec025"
Entra Identity
      |
      ↓
SQL MI database user
      |
      ↓
Database roles/permissions
```

---

# 12. Login vs user

This is a classic SQL Server concept.

A **login** operates at the server/instance level.

A **database user** exists inside a database.

Conceptually:

```text id="sec026"
SQL MI
 |
 +-- Login
      |
      +-- Database A → User
      |
      +-- Database B → User
```

A login can be mapped to a user in a database.

---

# 13. Contained users

Contained database users reduce dependency on instance-level login mappings.

Conceptually:

```text id="sec027"
Database
   |
   +-- User
   |
   +-- Permissions
```

This can simplify database-level access management, especially for Microsoft Entra identities.

---

# 14. Server roles vs database roles

Don't confuse these.

### Server/instance-level

Examples include permissions related to:

```text id="sec028"
Logins
Server configuration
Instance-level administration
```

### Database-level

Examples:

```text id="sec029"
SELECT
INSERT
UPDATE
DELETE
EXECUTE
```

---

# 15. Least privilege

This is perhaps the most important security principle.

> Give a user/application only the permissions it actually needs.

Suppose the application only needs:

```sql id="sec030"
SELECT
INSERT
UPDATE
```

Don't give it:

```text id="sec031"
db_owner
```

unless there is a legitimate reason.

---

# 16. Why db_owner is dangerous

`db_owner` effectively gives very broad control over a database.

If an application account is compromised:

```text id="sec032"
Application credential stolen
        ↓
db_owner
        ↓
Attacker can potentially modify/delete sensitive data
```

The blast radius becomes much larger.

---

# 17. Better role design

Create a role such as:

```sql id="sec033"
CREATE ROLE OrderAppRole;
```

Then grant only required permissions.

For example:

```sql id="sec034"
GRANT SELECT, INSERT, UPDATE
ON dbo.Orders
TO OrderAppRole;
```

Then add the application user to the role.

The exact permission model should be designed according to the application's operations.

---

# 18. Network security

Even if authentication is perfect, you should still restrict network access.

Think:

```text id="sec035"
Internet
   |
   X
SQL MI
```

Prefer:

```text id="sec036"
Application
    |
    ↓
Private network
    |
    ↓
SQL MI
```

---

# 19. SQL MI and virtual networks

SQL Managed Instance is designed to operate within an Azure virtual network environment.

Conceptually:

```text id="sec037"
Azure VNet
+----------------------------------+
|                                  |
| Application Subnet               |
|       |                          |
|       ↓                          |
| SQL MI Subnet                    |
|                                  |
+----------------------------------+
```

This provides a strong network boundary compared with exposing a database directly to the public internet.

---

# 20. Defense in depth

A secure design doesn't depend on one control.

Use multiple layers:

```text id="sec038"
Network isolation
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

# 21. Encryption in transit

When data moves between:

```text id="sec039"
Application
     ↓
Network
     ↓
SQL MI
```

you want encrypted communication.

TLS is used to protect data in transit.

Conceptually:

```text id="sec040"
Plain text
   X
Network attacker
   X
Encrypted TLS channel
```

---

# 22. Encryption at rest

Data stored on disk should also be protected.

This is:

> **Encryption at rest**

Azure SQL services provide encryption mechanisms such as **Transparent Data Encryption (TDE)**.

---

# 23. TDE

TDE encrypts database storage so that data files and related database storage aren't simply readable as plaintext if the underlying storage is accessed improperly.

Conceptually:

```text id="sec041"
Database
   |
   ↓
TDE
   |
   ↓
Encrypted storage
```

---

# 24. What TDE protects

Think:

```text id="sec042"
Data at rest
     ↓
Encrypted
```

It is particularly useful for protecting against scenarios involving unauthorized access to the underlying storage.

But TDE doesn't solve:

```text id="sec043"
Authorized user executes DELETE
```

Because the database engine decrypts data for authorized operations.

Again:

> Encryption ≠ authorization.

---

# 25. TDE vs Always Encrypted

This is an important interview question.

### TDE

Protects:

```text id="sec044"
Data at rest
```

### Always Encrypted

Can protect sensitive column values from the database engine in scenarios where the client-side encryption architecture supports it.

Conceptually:

```text id="sec045"
Application
   |
   ↓
Encrypted sensitive column
   |
   ↓
SQL MI
```

The database may not have access to plaintext for appropriately configured Always Encrypted columns.

---

# 26. Example: salary data

Suppose table:

```text id="sec046"
Employees
-----------------------
EmployeeID
Name
Salary
SSN
```

You might want additional protection around:

```text id="sec047"
SSN
```

TDE protects the database at rest.

Always Encrypted can provide stronger column-level confidentiality for supported client/application scenarios.

---

# 27. Important limitation of encryption

Suppose application has permission:

```text id="sec048"
SELECT Salary
```

Encryption doesn't automatically prevent the authorized application from reading the value.

Security must combine:

```text id="sec049"
Identity
+
Permissions
+
Encryption
```

---

# 28. Azure Key Vault

Secrets and cryptographic keys often need secure centralized management.

Azure Key Vault can be used to manage:

```text id="sec050"
Secrets
Keys
Certificates
```

Conceptually:

```text id="sec051"
Application
    |
    ↓
Managed Identity
    |
    ↓
Key Vault
    |
    ↓
Secret / Key
```

This avoids putting secrets directly into application source code.

---

# 29. Key Vault + SQL MI

For encryption-related scenarios, Key Vault can participate in key management architectures.

Conceptually:

```text id="sec052"
SQL MI
  |
  ↓
Encryption
  |
  ↓
Key management
  |
  ↓
Azure Key Vault
```

The exact configuration depends on the encryption feature and SQL MI capabilities being used.

---

# 30. Managed Identity + Key Vault

This is a very common enterprise pattern.

Instead of:

```text id="sec053"
App
 |
 +-- Key Vault username
 +-- Key Vault password
```

use:

```text id="sec054"
App
 |
 ↓
Managed Identity
 |
 ↓
Key Vault
```

Again:

> **No hard-coded credentials.**

---

# 31. Auditing

Authentication answers:

> Who are you?

Authorization answers:

> What can you do?

Auditing answers:

> **What actually happened?**

For example:

```text id="sec055"
User: Alice
Time: 14:32
Database: SalesDB
Action: UPDATE
Object: Customers
```

This is extremely valuable for security investigations.

---

# 32. Security incident scenario

Suppose someone says:

> "Customer records were modified yesterday."

You need to determine:

```text id="sec056"
Who?
When?
What?
From where?
Which database?
Which operation?
```

Auditing and related logging can provide evidence for investigation.

---

# 33. Auditing vs monitoring

### Monitoring

```text id="sec057"
CPU
IO
Memory
Performance
```

### Auditing

```text id="sec058"
Who accessed what?
Who modified what?
When?
```

Different purposes.

---

# 34. Defender for SQL

Microsoft Defender for SQL provides additional security capabilities for SQL workloads.

It can help identify:

```text id="sec059"
Potential threats
Suspicious activities
Vulnerabilities
Security recommendations
```

Think:

```text id="sec060"
SQL MI
   |
   ↓
Defender for SQL
   |
   ↓
Security signals
   |
   ↓
Security team / DBA
```

---

# 35. Security posture

A mature production environment combines:

```text id="sec061"
Identity
Network
Permissions
Encryption
Auditing
Threat detection
Vulnerability management
```

No single product is enough.

---

# 36. RBAC vs SQL permissions

This is a very common Azure interview question.

### Azure RBAC

Controls access to **Azure resources**.

For example:

```text id="sec062"
SQL Managed Instance resource
Azure configuration
Resource management
```

### SQL permissions

Control what someone can do **inside SQL Server/databases**.

For example:

```text id="sec063"
SELECT
INSERT
UPDATE
DELETE
EXECUTE
```

So:

```text id="sec064"
Azure RBAC ≠ SQL database permissions
```

---

# 37. Example

A user may have Azure permissions to:

```text id="sec065"
View SQL MI resource
```

but that doesn't automatically mean they can execute:

```sql id="sec066"
SELECT *
FROM CustomerData;
```

inside a database.

You need appropriate SQL authentication/authorization as well.

---

# 38. Another example

A DBA may have SQL permissions:

```text id="sec067"
SELECT
ALTER
EXECUTE
```

but lack Azure permissions to:

```text id="sec068"
Change VNet configuration
Delete the SQL MI resource
Modify Azure networking
```

Again:

```text id="sec069"
Azure control plane
       ≠
SQL data plane
```

---

# 39. Control plane vs data plane

This distinction is very important.

### Control plane

Managing the Azure resource:

```text id="sec070"
Create SQL MI
Resize
Configure networking
Change Azure settings
Delete resource
```

### Data plane

Working with the database:

```text id="sec071"
SELECT
INSERT
UPDATE
DELETE
EXECUTE
```

---

# 40. Real production architecture

Suppose you have:

```text id="sec072"
Azure App Service
       |
       ↓
Managed Identity
       |
       ↓
Microsoft Entra ID
       |
       ↓
SQL MI
```

Inside SQL MI:

```text id="sec073"
Application User
       |
       ↓
Application Role
       |
       ↓
SELECT / INSERT / UPDATE
```

Network:

```text id="sec074"
Private VNet
```

Data:

```text id="sec075"
TDE
```

Monitoring:

```text id="sec076"
Auditing
+
Defender for SQL
+
Azure Monitor
```

That's a strong enterprise architecture.

---

# 41. Real-time scenario: no passwords allowed

Business requirement:

> "Our security team doesn't allow database passwords to be stored in application configuration."

Application:

```text id="sec077"
Azure App Service
```

Database:

```text id="sec078"
SQL Managed Instance
```

Solution:

```text id="sec079"
App Service
    |
    ↓
Managed Identity
    |
    ↓
Microsoft Entra ID
    |
    ↓
SQL MI
    |
    ↓
Database User
    |
    ↓
Least-privilege Role
```

No permanent SQL password is required for this authentication pattern.

---

# 42. Step-by-step design

### Step 1

Enable/configure Microsoft Entra authentication for the SQL MI environment.

### Step 2

Give the application resource a managed identity.

### Step 3

Create the appropriate Microsoft Entra database principal.

### Step 4

Create a database role.

### Step 5

Grant only required permissions.

For example:

```sql id="sec080"
CREATE ROLE OrderApplicationRole;
```

Then:

```sql id="sec081"
GRANT SELECT, INSERT, UPDATE
ON dbo.Orders
TO OrderApplicationRole;
```

Then associate the application identity with the appropriate database principal/role according to the chosen Entra authentication model.

---

# 43. What you should NOT do

Don't give the application:

```text id="sec082"
sysadmin
```

or broad administrative permissions simply because:

> "The application needs SQL access."

It usually doesn't.

---

# 44. Separate identities

A mature enterprise environment may use:

```text id="sec083"
Production App Identity
Development App Identity
Reporting Identity
ETL Identity
DBA Identity
```

rather than:

```text id="sec084"
One SQL account for everything
```

Why?

Because it improves:

```text id="sec085"
Auditability
Least privilege
Incident investigation
Credential isolation
```

---

# 45. ETL scenario

Suppose an ETL service needs:

```text id="sec086"
SELECT source tables
INSERT staging tables
EXECUTE procedures
```

Don't give:

```text id="sec087"
db_owner
```

Instead:

```text id="sec088"
ETL identity
    |
    ↓
ETL database role
    |
    +-- SELECT
    +-- INSERT
    +-- EXECUTE
```

---

# 46. Reporting scenario

A reporting application should usually need:

```text id="sec089"
SELECT
```

not:

```text id="sec090"
UPDATE
DELETE
ALTER
DROP
```

This reduces accidental and malicious modification risk.

---

# 47. DBA access

DBAs may need elevated privileges.

But even then:

```text id="sec091"
Production DBA
      |
      ↓
Privileged identity
```

should be controlled and audited.

Use:

```text id="sec092"
MFA
Privileged access controls
Auditing
Least privilege
```

where appropriate.

---

# 48. Network + identity together

A secure design isn't:

```text id="sec093"
"Only password authentication."
```

Instead:

```text id="sec094"
Private network
     +
Strong identity
     +
Least privilege
     +
Encryption
     +
Monitoring
```

---

# 49. Incident scenario — stolen credentials

Suppose attacker obtains:

```text id="sec095"
appuser/password
```

If the account has:

```text id="sec096"
db_owner
```

the blast radius can be huge.

If instead the application identity has:

```text id="sec097"
SELECT/INSERT/UPDATE
on specific tables
```

the potential damage is significantly constrained.

That's the practical value of least privilege.

---

# 50. Incident scenario — suspicious login

Security team reports:

> "There was an unexpected database access event at 2 AM."

Your investigation:

```text id="sec098"
1. Identify identity
2. Check authentication events
3. Check source/network information
4. Check audit events
5. Identify objects accessed
6. Identify changes
7. Correlate with application logs
8. Determine whether access was legitimate
```

This is where auditing and centralized monitoring become important.

---

# 51. SQL Injection

Another important SQL security topic.

Bad application design:

```text id="sec099"
query = "SELECT * FROM Users WHERE Name = '" + userInput + "'"
```

An attacker may manipulate input.

Better:

```text id="sec100"
Parameterized query
```

For example conceptually:

```sql id="sec101"
SELECT *
FROM Users
WHERE Name = @Name;
```

The application supplies the parameter separately.

---

# 52. Why SQL permissions still matter

Even if SQL injection occurs, least privilege can limit damage.

Suppose compromised application identity only has:

```text id="sec102"
SELECT on Orders
```

Then the attacker has far fewer capabilities than if the identity has:

```text id="sec103"
db_owner
```

Therefore:

```text id="sec104"
Parameterized queries
+
Least privilege
```

should work together.

---

# 53. Dynamic Data Masking

Dynamic Data Masking can help limit exposure of sensitive data to certain users.

Example:

Actual:

```text id="sec105"
123-45-6789
```

Masked presentation:

```text id="sec106"
XXX-XX-6789
```

It's useful in some scenarios, but remember:

> **Dynamic Data Masking is not a replacement for authorization or encryption.**

---

# 54. Data masking misconception

Don't say:

> "The data is encrypted because it is masked."

Masking is presentation/access behavior.

Encryption is cryptographic protection.

Different concepts.

---

# 55. Security architecture

A good production SQL MI security design looks like:

```text id="sec107"
                         USERS
                           |
                           ↓
                     Application
                           |
                    Managed Identity
                           |
                           ↓
                    Microsoft Entra
                           |
                           ↓
                     SQL Managed
                       Instance
                           |
         +-----------------+-----------------+
         |                 |                 |
       Network        Authorization      Encryption
         |                 |                 |
      Private VNet      SQL Roles           TDE
      NSGs/Firewall    Least privilege     Always Encrypted*
                                             |
                                          Key Vault*
         |
         +-------------------------+
                                   |
                              Monitoring
                                   |
                         Auditing / Defender
```

`*` where the specific scenario requires/supports it.

---

# 56. Interview question

### "Authentication vs authorization?"

Answer:

> Authentication verifies who the identity is. Authorization determines what that identity is allowed to do.

---

# 57. Interview question

### "Why use managed identity?"

Answer:

> "Managed identity allows Azure resources to authenticate to supported services using an Azure-managed identity instead of storing long-lived credentials in application configuration."

---

# 58. Interview question

### "What is the difference between RBAC and SQL permissions?"

Answer:

> "Azure RBAC controls access to Azure resources and management operations, while SQL permissions control access to databases and database objects. Having Azure permissions doesn't automatically grant SELECT or UPDATE permissions inside a database."

---

# 59. Interview question

### "TDE vs Always Encrypted?"

Strong answer:

> "TDE protects database storage at rest. Always Encrypted is designed for protecting sensitive column data from exposure to the database engine in supported client-side encryption scenarios. They solve different security problems."

---

# 60. Interview question

### "Why shouldn't an application use db_owner?"

Answer:

> "It violates least privilege and increases the blast radius if the application is compromised or contains a bug. The application should receive only the database permissions required for its operations."

---

# 61. Interview question

### "How would you secure SQL MI from an application?"

A strong answer:

```text id="sec108"
1. Keep SQL MI within the appropriate private network architecture.
2. Use Microsoft Entra authentication where practical.
3. Prefer managed identity for Azure-hosted applications.
4. Create a dedicated database principal.
5. Grant least-privilege permissions.
6. Encrypt data at rest and in transit.
7. Protect encryption keys/secrets appropriately.
8. Enable auditing/security monitoring.
9. Monitor suspicious activity.
10. Regularly review permissions.
```

---

# 62. Security checklist

```text id="sec109"
IDENTITY
☐ Microsoft Entra ID
☐ SQL authentication where required
☐ Managed identity
☐ System-assigned identity
☐ User-assigned identity

AUTHORIZATION
☐ SQL users
☐ Roles
☐ Least privilege
☐ Application-specific permissions
☐ Avoid unnecessary db_owner

NETWORK
☐ VNet
☐ Private connectivity
☐ Firewall/network controls
☐ Network segmentation

DATA
☐ TLS
☐ TDE
☐ Always Encrypted where appropriate
☐ Key management

MONITORING
☐ Auditing
☐ Azure Monitor
☐ Defender for SQL
☐ Security alerts

GOVERNANCE
☐ RBAC
☐ Separation of duties
☐ Access reviews
☐ MFA
```

---

# 63. Final production scenario

Imagine this requirement:

> **"We have an Azure App Service running an e-commerce application. It needs read/write access to the Orders database in SQL Managed Instance. Security prohibits storing database passwords. The database contains customer PII. Design the security architecture."**

Your answer should look like:

```text id="sec110"
                    App Service
                        |
                        ↓
                Managed Identity
                        |
                        ↓
                 Microsoft Entra ID
                        |
                        ↓
                  SQL Managed
                    Instance
                        |
                 Database User
                        |
                 Application Role
                        |
              +---------+---------+
              |                   |
           SELECT               UPDATE
           INSERT                etc.
```

Network:

```text id="sec111"
App Service
     |
     ↓
Private network connectivity
     |
     ↓
SQL MI
```

Data protection:

```text id="sec112"
TLS
 +
TDE
 +
Always Encrypted where appropriate
 +
Key management
```

Monitoring:

```text id="sec113"
Auditing
 +
Defender for SQL
 +
Azure Monitor
```

And most importantly:

```text id="sec114"
NO hard-coded SQL password
NO db_owner
NO unnecessary permissions
```

---

## The security principle to remember

> **A secure SQL Managed Instance isn't created by one feature. It's created by combining network isolation, strong identity, least-privilege authorization, encryption, auditing, threat detection, and continuous access review.**

### Next — Lesson 89: Azure SQL Managed Instance Networking

We'll go deep into **VNet, subnets, delegated subnet concepts, NSGs, route tables, DNS, private connectivity, connectivity from App Service/VMs/on-premises, VPN vs ExpressRoute, hybrid SQL Server-to-MI scenarios, ports, troubleshooting connection failures, and a real production scenario where the application can reach Azure resources but cannot connect to SQL MI.**
