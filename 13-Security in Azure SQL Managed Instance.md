**Lesson 11 — Security in Azure SQL Managed Instance**.

This is a major real-world DBA topic because in production, the question isn't only:

> "Can the application connect?"

It's also:

> **"Who can connect, what can they access, and how do we protect the data?"**

# Lesson 11 — Azure SQL Managed Instance Security 🔐

Think about security in layers:

```text
                    SQL MI SECURITY
                          |
       +------------------+------------------+
       |                  |                  |
       v                  v                  v
   Identity            Network             Data
       |                  |                  |
       v                  v                  v
 Authentication       NSG/VNet          Encryption
       |
       v
 Authorization
```

And then:

```text
Monitoring
Auditing
Threat detection
Secrets management
```

---

# 1. Authentication vs Authorization

This is one of the most important concepts.

### Authentication

> **Who are you?**

Example:

```text
app-user
```

### Authorization

> **What are you allowed to do?**

Example:

```text
SELECT → Yes
INSERT → Yes
DROP TABLE → No
```

So:

```text
User
 ↓
Authentication
 ↓
Authorization
 ↓
Database access
```

---

# 2. Traditional SQL Authentication

You can have:

```text
Login:
app_login

Password:
********
```

Then:

```text
app_login
    |
    v
SQL Managed Instance
    |
    v
SalesDB user
    |
    v
Permissions
```

This is familiar from traditional SQL Server.

But there is a major problem:

> **Where do you store the password?**

Maybe:

```text
appsettings.json
Environment variable
Secret store
Configuration file
```

You don't want credentials scattered throughout your environment.

---

# 3. Microsoft Entra authentication

Azure provides integration with **Microsoft Entra ID**.

Conceptually:

```text
Application
    |
    v
Microsoft Entra ID
    |
    v
Authentication token
    |
    v
SQL Managed Instance
```

Instead of:

```text
Application
    |
    v
username + password
```

you can use an identity-based model.

---

# 4. Why this is useful

Imagine 100 applications.

Traditional model:

```text
App1 → SQL password
App2 → SQL password
App3 → SQL password
...
App100 → SQL password
```

Now password rotation becomes painful.

With identity-based authentication:

```text
App
 ↓
Managed Identity
 ↓
Microsoft Entra
 ↓
SQL MI
```

You reduce the need to manage long-lived SQL passwords.

---

# 5. Managed Identity

This is an important Azure concept.

A managed identity allows an Azure resource to have an identity managed by Azure.

For example:

```text
Azure App Service
       |
       v
Managed Identity
       |
       v
Microsoft Entra
       |
       v
SQL MI
```

The application doesn't necessarily need:

```text
SQL username
+
SQL password
```

stored in its configuration.

---

# 6. Real production scenario

ABC Retail has:

```text
Azure App Service
       |
       v
Customer API
       |
       v
SQL MI
```

Old design:

```text
Connection string
User = appuser
Password = ********
```

Security team says:

> "We don't want database passwords stored in application configuration."

You redesign:

```text
App Service
    |
    v
Managed Identity
    |
    v
Microsoft Entra authentication
    |
    v
SQL MI
```

Now identity management becomes much cleaner.

---

# 7. Login → User → Role

Let's revisit SQL Server security.

Suppose:

```text
Login:
app_login
```

Inside:

```text
SalesDB
```

you create/map:

```text
User:
app_user
```

Then assign:

```text
db_datareader
db_datawriter
```

Conceptually:

```text
Login
  ↓
User
  ↓
Database Role
  ↓
Permissions
```

---

# 8. Why don't we give db_owner?

Someone says:

> "The application needs access to the database. Give it db_owner."

That's usually a bad design.

`db_owner` is powerful.

The application might then be able to perform operations far beyond what it actually needs.

Instead:

```text
Application
   |
   v
app_user
   |
   +-- SELECT
   +-- INSERT
   +-- UPDATE
```

and nothing unnecessary.

This is called:

> **Least privilege**

---

# 9. Least privilege example

Suppose an application only needs:

```sql id="7dbs32"
SELECT
INSERT
UPDATE
```

Don't give:

```text id="n8xq4s"
DELETE
ALTER
DROP
CONTROL
db_owner
```

unless there is a real requirement.

Security should be based on:

> **What does the application actually need?**

---

# 10. Data encryption at rest

Now suppose someone steals the physical storage.

You don't want:

```text id="3imkq4"
Database files
      ↓
Readable customer data
```

Azure SQL services use encryption mechanisms to protect data at rest.

A key concept you'll encounter is:

# Transparent Data Encryption — TDE

Conceptually:

```text id="trwq9z"
Database
   |
   v
Encryption
   |
   v
Encrypted storage
```

Applications don't need to manually encrypt every query just because TDE is enabled.

That's why it's called **transparent**.

---

# 11. TDE protects what?

Think:

```text id="6k2qlm"
Data at rest
     ↓
Database files
     ↓
Encrypted
```

It helps protect stored database data if the underlying storage is accessed improperly.

But TDE doesn't mean:

```text id="9s3cqa"
"Everything is encrypted everywhere."
```

You still need to think about:

```text Network encryption
Application secrets
Backups
Sensitive columns
Logs
Exports
```

---

# 12. Encryption in transit

Now imagine:

```text id="16q5kw"
Application
     |
     | Network
     v
SQL MI
```

You want the traffic protected.

This is where TLS/encrypted connections matter.

Think:

```text id="0drr2m"
Application
    |
    | encrypted traffic
    v
SQL MI
```

So:

```text id="5y3wzx"
TDE
 ↓
Protection at rest

TLS
 ↓
Protection in transit
```

These solve different problems.

---

# 13. Column-level protection

Suppose your database contains:

```text id="zkw3qu"
Customer
├── Name
├── Address
├── Phone
└── CreditCardNumber
```

Encrypting the entire database doesn't necessarily satisfy every sensitive-data requirement.

You might need stronger protection for specific sensitive fields.

This is where technologies such as **Always Encrypted** can be relevant.

Conceptually:

```text id="8sv3r1"
Application
   |
   | Encrypt/decrypt
   v
Sensitive column
   |
   v
SQL MI
```

The exact architecture depends on the application and encryption model.

---

# 14. Azure Key Vault

Now comes another important Azure service.

Where should encryption keys/secrets be managed?

A common answer:

> **Azure Key Vault**

Conceptually:

```text id="g4s0z3"
             Key Vault
                 |
       +---------+---------+
       |                   |
     Keys                Secrets
       |                   |
       +---------+---------+
                 |
                 v
               SQL MI
```

Key Vault helps centralize:

```text
Secrets
Keys
Certificates
```

and control access to them.

---

# 15. Customer-managed keys

Organizations sometimes require greater control over encryption keys.

Conceptually:

```text id="k9m2o0"
Your organization
       |
       v
Key Vault
       |
       v
Customer-managed key
       |
       v
SQL encryption
```

Why might an enterprise want this?

```text id="9x2g9m"
Compliance
Key ownership/control
Security policies
Regulatory requirements
```

But this adds operational responsibility.

---

# 16. Auditing

Now imagine security asks:

> "Who accessed the customer table yesterday?"

You need more than encryption.

You need:

```text id="f98u1b"
Audit
   ↓
Record activity
   ↓
Investigate
```

For example:

```text id="n3xk7s"
User: app_user
Action: SELECT
Object: Customer
Time: 14:05
```

Auditing helps establish an activity trail.

---

# 17. Why auditing matters

Imagine a sensitive table:

```text id="f7j6g0"
CustomerCreditCards
```

Security discovers suspicious access.

They ask:

> Who accessed it?

Without auditing:

```text id="e3k3ar"
????
```

With auditing:

```text id="k0g7g7"
User
 ↓
Time
 ↓
Action
 ↓
Object
```

You can investigate.

---

# 18. Defender for SQL

Azure also provides security capabilities through **Microsoft Defender for SQL**.

Think of it as another security layer:

```text id="q5g3u4"
SQL MI
 |
 +---- Threat detection
 |
 +---- Security recommendations
 |
 +---- Vulnerability assessment
 |
 +---- Alerts
```

It can help identify security risks and suspicious behavior.

But remember:

> Security tools don't replace secure design.

---

# 19. Network security + database security

Let's combine everything.

```text id="6t3k7n"
Application
     |
     v
VNet / Network
     |
     v
NSG / Firewall
     |
     v
SQL MI
     |
     v
Authentication
     |
     v
Authorization
     |
     v
Database
     |
     v
Encrypted data
     |
     v
Audit / Monitoring
```

Security is **defense in depth**.

---

# 20. Real production scenario

ABC Retail has:

```text
Customer API
     |
     v
SQL MI
```

Requirements:

```text
1. No SQL passwords in application config
2. Only application identity can access DB
3. App can read/write customer orders
4. App cannot modify schema
5. Sensitive data must be encrypted
6. Security team needs audit records
```

A reasonable architecture might be:

```text id="h4kqv0"
Customer API
     |
     v
Managed Identity
     |
     v
Microsoft Entra
     |
     v
SQL MI
     |
     +---- app_user
     |
     +---- Least privilege
     |
     +---- TDE
     |
     +---- Auditing
     |
     +---- Defender for SQL
     |
     v
Key Vault where applicable
```

This is much stronger than:

```text id="kzqg7d"
app
 |
 v
sa
 |
 v
password123
```

---

# 21. Security incident scenario

Suppose someone says:

> **"The application login has db_owner because it wasn't working otherwise."**

Red flag 🚩.

Your response should be:

```text
1. Determine required permissions
2. Remove unnecessary privileges
3. Create appropriate database role
4. Grant only required permissions
5. Test application
6. Monitor for authorization failures
```

Don't simply leave `db_owner` permanently because it's convenient.

---

# 22. Another important distinction

### Authentication failure

```text
Who are you?
     ↓
FAILED
```

Example:

```text
Login failed
```

### Authorization failure

```text
Who are you?
     ↓
SUCCESS

What can you do?
     ↓
FAILED
```

Example:

```text
SELECT permission denied
```

This distinction is very useful during troubleshooting.

---

# 23. Security troubleshooting example

Application receives:

```text
Login failed for user 'appuser'
```

Investigate:

```text
Authentication
```

But application receives:

```text
SELECT permission denied on object 'Orders'
```

Now:

```text
Authentication = successful
Authorization = problem
```

This immediately narrows the investigation.

---

# 24. Security architecture to memorize

Use this mental model:

```text
                    USERS / APPLICATIONS
                            |
                            v
                    IDENTITY / AUTH
                            |
                            v
                       NETWORK
                            |
                            v
                    SQL MANAGED INSTANCE
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
        AUTHORIZATION    ENCRYPTION      AUDITING
             |              |              |
             v              v              v
         Roles/ACLs     TDE/TLS/etc.     Logs
                            |
                            v
                       KEY MANAGEMENT
```

---

# 25. Interview question

### "How would you secure an Azure SQL Managed Instance used by a production application?"

A strong answer:

> "I'd use private network connectivity where appropriate, restrict network access with the appropriate Azure networking controls, use Microsoft Entra authentication or managed identities where supported to reduce long-lived credentials, enforce least-privilege database permissions, use encryption at rest and encrypted connections, protect keys and secrets appropriately, enable auditing and security monitoring, and regularly review access and security recommendations."

That's a strong cloud DBA answer.

---

# Lesson 11 challenge 🧑‍💻

ABC Retail has this architecture:

```text
Azure App Service
       |
       v
SQL Managed Instance
```

The application currently uses:

```text
SQL Login: appuser
Password: stored in application configuration
Database role: db_owner
```

Security team gives you these requirements:

```text
1. Remove stored SQL passwords where possible
2. Application must read/write Orders
3. Application must NOT alter database schema
4. Protect data at rest
5. Audit access to sensitive customer data
```

### Design the improved security architecture.

Tell me:

**1. What would you use instead of storing the SQL password?**

**2. What database permissions would you give the application?**

**3. Would you keep `db_owner`? Why?**

**4. What protects data at rest?**

**5. What would you use to track access to sensitive data?**
