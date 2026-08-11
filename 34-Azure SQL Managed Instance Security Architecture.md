# Lesson 66 — Azure SQL Managed Instance Security Architecture

Now we move into **security**, one of the areas where SQL Server DBAs are expected to understand both the **SQL layer** and the **Azure layer**.

The biggest mistake is treating these as the same thing.

> **Azure RBAC and SQL permissions are different security systems.**

Let's build the model from the ground up.

---

# 1. The security layers

Think of an Azure SQL Managed Instance environment like this:

```text
                Azure
                  |
        +---------+---------+
        |                   |
   Azure RBAC          Networking
        |                   |
        +---------+---------+
                  |
              SQL MI
                  |
        +---------+---------+
        |                   |
 Authentication       Authorization
        |                   |
   Who are you?       What can you do?
        |                   |
        +---------+---------+
                  |
              Database
                  |
          Object permissions
```

There are multiple security boundaries.

---

# 2. Authentication vs authorization

This is fundamental.

### Authentication

> **Who are you?**

Examples:

```text
SQL login
Microsoft Entra user
Microsoft Entra service principal
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
CREATE DATABASE
```

So:

```text id="r2q7k8"
Authentication
       ↓
"Who are you?"
       ↓
Authorization
       ↓
"What can you do?"
```

---

# 3. Real production incident

Imagine an application deployment at 10 AM.

Immediately afterward:

```text
Application
    ↓
SQL connection
    X
Login failed
```

Developers say:

> "SQL MI is down."

You check:

```text
MI status = healthy
Network = healthy
CPU = normal
```

So what happened?

You investigate authentication.

---

# 4. SQL authentication

Traditional SQL Server authentication uses:

```text
Username
Password
```

For example:

```text id="f6w3m1"
ApplicationLogin
**************
```

The login exists in SQL Server.

Advantages:

```text
Simple
Compatible with legacy applications
```

But it introduces credential-management concerns:

```text
Passwords
Rotation
Secret storage
Credential leakage
```

---

# 5. Microsoft Entra authentication

Azure environments can use Microsoft Entra identities.

Conceptually:

```text
Application
    |
    v
Microsoft Entra
    |
    | token
    v
SQL Managed Instance
```

The application doesn't necessarily need a traditional SQL password.

This is particularly valuable for Azure-native workloads.

---

# 6. Managed identity

A managed identity lets an Azure resource authenticate to supported services without you embedding a secret/password in application configuration.

Conceptually:

```text
Azure Application
       |
       | Managed Identity
       v
Microsoft Entra
       |
       | token
       v
SQL MI
```

This reduces credential-management overhead.

---

# 7. Why managed identity is valuable

Bad architecture:

```text id="c7m2v9"
Application
   |
   +-- SQL username
   +-- SQL password
```

Now someone must manage:

```text
Password storage
Rotation
Secret exposure
Deployment configuration
```

Better:

```text id="m8x1q4"
Application
   |
   +-- Managed Identity
            |
            v
       Entra token
            |
            v
           SQL
```

The identity is managed through Azure rather than hardcoding a database password into the application.

---

# 8. But managed identity alone doesn't grant SQL permissions

This is important.

People sometimes think:

> "The VM/App Service/Function has a managed identity, therefore it can access every database."

No.

You need both:

```text id="j4p9w2"
Identity
   +
SQL authorization
```

Conceptually:

```text
Managed Identity
       ↓
Authentication
       ↓
SQL user/principal
       ↓
Database permissions
       ↓
SELECT/EXECUTE/etc.
```

---

# 9. SQL permissions

Suppose the application only needs:

```sql id="v5m7c1"
SELECT
```

Don't give it:

```text id="a8k2q6"
db_owner
```

just because it's convenient.

Instead follow:

> **Least privilege.**

For example:

```text id="n3q9r5"
ApplicationUser
      |
      +-- SELECT on required tables
      +-- EXECUTE on required procedures
```

rather than:

```text id="z6w1m4"
ApplicationUser
      |
      +-- db_owner
```

---

# 10. Roles

SQL Server has built-in database roles such as:

```text id="p8c4n7"
db_datareader
db_datawriter
db_ddladmin
db_owner
```

But don't automatically use broad fixed roles.

For production applications, custom roles can provide better control.

Example:

```sql id="h2m6x8"
CREATE ROLE AppReadOnly;
```

Then grant only what's required.

---

# 11. RBAC vs SQL permissions

This is one of the most common interview questions.

### Azure RBAC

Controls access to Azure resources.

Examples:

```text id="g3n8p2"
Azure subscription
Resource group
SQL MI resource
Networking resources
Monitoring resources
```

### SQL permissions

Control database operations.

Examples:

```text id="v7x1k5"
SELECT
INSERT
UPDATE
DELETE
EXECUTE
ALTER
```

So:

> **Azure RBAC does not replace SQL database authorization.**

---

# 12. Example

Suppose a DBA has Azure permission to manage the SQL Managed Instance resource.

That doesn't automatically mean:

```text id="q5m9w3"
SELECT * FROM Customer
```

is allowed for every database user.

Conversely, a database user may be able to query data without having permission to modify Azure infrastructure.

Different security layers.

---

# 13. SQL login vs database user

Another important SQL Server concept.

At the instance level:

```text id="x8r4m2"
LOGIN
```

At the database level:

```text id="k1p7v9"
USER
```

Conceptually:

```text
SQL Login
    |
    v
Database User
    |
    v
Database permissions
```

This is why you can have an authenticated principal but still receive:

```text
Permission denied
```

---

# 14. Authentication success but authorization failure

Suppose application connects successfully:

```text id="m4q8c1"
Login succeeded
```

Then executes:

```sql id="y7p2k9"
UPDATE dbo.Orders
SET Status = 'Completed';
```

and receives:

```text id="s3v6n8"
UPDATE permission denied
```

This is **not an authentication problem**.

Authentication worked.

Authorization failed.

---

# 15. Real incident

Application deployment:

```text id="j9w3p5"
New application identity
       ↓
Authentication works
       ↓
Stored procedure call
       ↓
Permission denied
```

Developer says:

> "SQL authentication is broken."

Your investigation should ask:

```text id="k2r7m4"
Who is the application authenticating as?
Does the database user exist?
What roles is it in?
Does it have EXECUTE?
Is the object in another schema?
Did deployment create a new identity?
```

---

# 16. Managed identity deployment failure

Suppose production application originally used:

```text id="q6m1x8"
ManagedIdentity-A
```

Deployment changes it to:

```text id="v3n9p2"
ManagedIdentity-B
```

But SQL permissions still exist for:

```text id="z8c4w7"
ManagedIdentity-A
```

Result:

```text id="n5k2r6"
New identity
    ↓
Authentication token
    ↓
SQL MI
    ↓
No appropriate SQL principal/permissions
    ↓
Access denied
```

The database isn't broken.

The deployment changed the identity.

---

# 17. Encryption at rest

Now let's discuss data protection.

Azure SQL Managed Instance supports encryption-at-rest capabilities, including **Transparent Data Encryption (TDE)**.

Conceptually:

```text id="u4j8m1"
Application
    ↓
SQL MI
    ↓
Encrypted database files
    ↓
Encrypted storage
```

If someone obtains the underlying encrypted storage, the data isn't simply readable as plaintext database files.

---

# 18. TDE

Transparent Data Encryption protects database files at rest.

The application doesn't normally need to change its SQL queries.

That's why it's called:

> **Transparent**

Conceptually:

```text id="k8r3v6"
INSERT
 ↓
SQL Server
 ↓
Encryption
 ↓
Storage
```

and:

```text id="p2m7x9"
Storage
 ↓
Decryption
 ↓
SQL Server
 ↓
Application
```

---

# 19. TDE isn't the same as column-level encryption

Suppose you have:

```text id="w6n2q4"
Customer
---------
Name
SSN
CreditCardNumber
```

TDE protects the database at rest.

But an application user with legitimate SQL access can still query:

```sql id="e9v3k7"
SELECT CreditCardNumber
FROM Customer;
```

TDE isn't designed to prevent authorized database users from reading data through SQL.

That's where other mechanisms, such as **Always Encrypted**, can be relevant for specific sensitive-column requirements.

---

# 20. Always Encrypted — conceptual difference

With appropriate Always Encrypted architecture:

```text id="c7p1m9"
Sensitive column
       ↓
Encrypted
       ↓
Database stores ciphertext
```

The client/application driver participates in encryption/decryption.

This can protect sensitive values from exposure to database administrators in scenarios designed for that security model.

It has important limitations and application/query implications, so you don't enable it casually.

---

# 21. Key management

Encryption requires keys.

Azure environments commonly use services such as:

Microsoft Azure

and Azure Key Vault for centralized key management scenarios.

Conceptually:

```text id="m2x8q4"
SQL MI
   |
   | encryption key dependency
   v
Key management service
   |
   v
Protected keys
```

Key management becomes part of your operational responsibility:

```text id="g9n4r1"
Who can access keys?
Who can rotate them?
What happens if access is lost?
How is access audited?
```

---

# 22. Network security

Database security isn't only usernames and passwords.

You also need to consider:

```text id="h3v7p8"
Network access
   ↓
Private connectivity
   ↓
Firewall/access controls
   ↓
Identity
   ↓
SQL authorization
```

A secure architecture generally minimizes unnecessary public exposure and restricts network paths according to the workload's requirements.

---

# 23. Defense in depth

Think:

```text id="z5q1m8"
                    Internet
                       |
                 Network controls
                       |
                 Private access
                       |
                    SQL MI
                       |
                 Authentication
                       |
                 Authorization
                       |
                    Database
                       |
                  Encryption
                       |
                    Auditing
```

No single security control is enough.

---

# 24. Auditing

Suppose security asks:

> "Who changed the customer address yesterday?"

You need evidence.

Auditing can capture database activity for security, compliance, and investigation purposes.

Your design should answer:

```text id="r8x4m2"
Who?
What?
When?
Where/from what context?
Was it successful?
```

---

# 25. Security incident

At 3:00 PM:

```text id="s7k2n5"
Security alert:
Unexpected UPDATE activity
```

You investigate.

Audit trail indicates:

```text id="q4m8v1"
Principal = ApplicationIdentity
Operation = UPDATE
Time = 14:57
Database = Sales
```

Now you know:

```text id="j6p3w9"
Which identity
Which database
Which operation
When it happened
```

That is dramatically more useful than:

> "Someone changed the data."

---

# 26. Least privilege in practice

Imagine an application only calls:

```text id="c2x7m4"
dbo.GetCustomer
dbo.CreateOrder
dbo.GetOrderStatus
```

Don't grant:

```text id="y9n3p6"
SELECT/INSERT/UPDATE/DELETE
on every table
```

Prefer an authorization design where the application receives only the capabilities it needs.

For example:

```text id="m5q1r8"
EXECUTE on required stored procedures
```

with carefully controlled underlying permissions.

---

# 27. Production security checklist

For an Azure SQL MI workload:

```text id="a8v4k2"
☐ Microsoft Entra integration where appropriate
☐ Managed identities where appropriate
☐ No hardcoded secrets
☐ Least privilege
☐ Separate application identities
☐ SQL roles/users reviewed
☐ Azure RBAC reviewed
☐ Network access restricted
☐ Encryption at rest
☐ Key management
☐ Auditing
☐ Security monitoring
☐ Regular access reviews
☐ Backup/restore security
```

---

# 28. The 10 AM deployment incident — complete diagnosis

Application deployment occurs:

```text id="p7x2m9"
10:00
```

At:

```text id="f4n8c3"
10:03
```

application gets:

```text id="j1k6v5"
Permission denied
```

### Step 1

Check MI:

```text id="u3m7q8"
Healthy
```

### Step 2

Check authentication:

```text id="x8c2n4"
Token acquisition successful
```

### Step 3

Identify principal:

```text id="r5p9w1"
ManagedIdentity-B
```

### Step 4

Compare previous deployment:

```text id="k2v6m8"
Old identity = ManagedIdentity-A
New identity = ManagedIdentity-B
```

### Step 5

Check database principal:

```text id="s9q3x7"
ManagedIdentity-B
not present / not correctly authorized
```

### Root cause:

```text id="m4n8p2"
Deployment changed application identity
without corresponding SQL authorization
```

### Fix:

```text id="w7c1r5"
Create/configure appropriate database principal
+
Grant least-privilege permissions
+
Test
+
Document identity dependency
```

That's a real-world Azure DBA troubleshooting pattern.

---

# 29. Senior interview questions

### Q: What is the difference between Azure RBAC and SQL permissions?

> Azure RBAC controls access to Azure resources. SQL permissions control what a database principal can do inside SQL Server. They operate at different layers.

### Q: What is managed identity?

> A Microsoft Entra identity managed by Azure that allows supported Azure resources to authenticate to services without embedding traditional credentials in application configuration.

### Q: Does managed identity automatically give database access?

> No. Authentication and SQL authorization are separate. The identity still needs an appropriate SQL principal and permissions.

### Q: Does TDE protect against a malicious DBA querying data?

> No. TDE protects data at rest. An authorized SQL user who can query the database can still retrieve data. Additional controls such as Always Encrypted may be appropriate for certain sensitive-data requirements.

### Q: Why is least privilege important?

> It limits the blast radius if an application identity or credential is compromised and reduces accidental or unauthorized database modifications.

---

# 30. The security mental model

Memorize this:

```text
                 USER / APPLICATION
                         |
                         v
                AUTHENTICATION
                         |
              "WHO ARE YOU?"
                         |
                         v
                 AUTHORIZATION
                         |
              "WHAT CAN YOU DO?"
                         |
          +--------------+--------------+
          |              |              |
        SQL          Database       Object
       roles          roles       permissions
          |
          v
       DATA ACCESS
          |
     +----+----+
     |         |
 Encryption  Auditing
     |         |
     +----+----+
          |
      Monitoring
```

And above SQL:

```text
Azure RBAC
Network security
Managed identities
Key management
```

That's the complete picture.

---

# Next — Lesson 67

Next we'll tackle **Azure SQL Managed Instance connectivity and networking**, including a very common production scenario:

> **"The SQL MI is healthy, the database is healthy, credentials are correct — but the application suddenly cannot connect."**

We'll break down:

**VNet → subnet → NSG → DNS → private connectivity → ports → routing → private endpoints → connection strings → authentication → TLS → troubleshooting from application to SQL MI.**
