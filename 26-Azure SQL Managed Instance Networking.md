# Lesson 53 — Azure SQL Managed Instance Networking

This is a **very important Azure SQL MI topic** because many real production incidents that look like "SQL Server is down" are actually **networking or authentication problems**.

The key mental model is:

```text
Application
    |
    v
DNS
    |
    v
Network path
    |
    v
Azure SQL Managed Instance
    |
    v
Authentication
    |
    v
Authorization
    |
    v
Database
```

You need to troubleshoot each layer separately.

---

## 1. Where does Managed Instance live?

Azure SQL Managed Instance is deployed into an **Azure virtual network (VNet)** and uses a dedicated subnet.

Conceptually:

```text
Azure
 |
 +-- VNet
      |
      +-- MI subnet
      |      |
      |      +-- Managed Instance
      |
      +-- Application subnet
             |
             +-- Application
```

This is one of the major architectural differences from simply connecting to a standalone public database endpoint.

---

# 2. Real-world architecture

Imagine an e-commerce company:

```text
Internet
   |
   v
Application Gateway
   |
   v
Web/App tier
   |
   v
Application subnet
   |
   | Private network
   v
SQL MI subnet
   |
   v
Azure SQL Managed Instance
   |
   +-- OrdersDB
   +-- CustomerDB
   +-- InventoryDB
```

The application communicates with MI through the configured network path.

---

# 3. Why subnet design matters

The MI subnet isn't just another random subnet.

You need to plan:

```text
Address space
Routing
Network security
DNS
Connectivity
Delegation/configuration requirements
```

Before deploying MI, networking must be designed correctly.

This is one reason MI deployments are more infrastructure-aware than Azure SQL Database deployments.

---

# 4. Application and MI in different VNets

Suppose:

```text
VNet-A
Application
```

and:

```text
VNet-B
SQL MI
```

They aren't automatically able to communicate just because they're both in Azure.

You need an appropriate connectivity mechanism, such as:

```text
VNet peering
```

or another supported network architecture.

Conceptually:

```text
VNet-A
Application
    |
    | Peering / connectivity
    |
    v
VNet-B
SQL MI
```

---

# 5. On-premises application

Now consider a hybrid company:

```text
On-premises
Application
     |
     |
VPN / ExpressRoute
     |
     v
Azure VNet
     |
     v
SQL MI
```

This is a very common enterprise scenario.

For example:

> A company wants to migrate the database to Azure first but keep the application running in its data center temporarily.

MI can participate in that hybrid architecture when network connectivity and name resolution are correctly configured.

---

# 6. DNS — the hidden problem

Suppose the application says:

```text
"Cannot connect to SQL Server."
```

You check MI:

```text
MI = Healthy
```

Next question:

> Can the application resolve the MI hostname?

The path is:

```text
Application
    |
    v
DNS lookup
    |
    +---- SUCCESS → continue troubleshooting
    |
    +---- FAILURE → networking/DNS issue
```

If DNS resolution fails, SQL authentication hasn't even started.

---

# 7. Troubleshooting sequence

When an application cannot connect:

### Step 1 — DNS

```text
Can hostname resolve?
```

### Step 2 — Network

```text
Can the application reach the destination?
```

### Step 3 — Port/connectivity

```text
Can it establish the required connection?
```

### Step 4 — TLS/encryption

```text
Can the connection negotiate successfully?
```

### Step 5 — Authentication

```text
Are credentials/identity valid?
```

### Step 6 — Authorization

```text
Does the identity have permission?
```

### Step 7 — Database

```text
Does the requested database exist and accept the connection?
```

This sequence prevents you from debugging SQL permissions when DNS is broken.

---

# 8. Real incident

At 8:00 AM:

```text
Application deployment
       ↓
Connection failures
```

DBA checks:

```text
CPU = 15%
MI = healthy
Database = online
```

Application team says:

> "SQL is broken."

You test the connection path and discover:

```text
DNS resolution
      ↓
FAIL
```

The deployment changed the hostname.

Old:

```text
old-mi-endpoint
```

New:

```text
new-mi-endpoint
```

But the application's DNS configuration wasn't updated.

The database was perfectly healthy.

---

# 9. Network Security Groups

NSGs can affect network traffic.

Conceptually:

```text
Application
    |
    v
NSG/routing/security controls
    |
    v
SQL MI
```

If a required traffic path is blocked, the application can fail before SQL Server authentication is reached.

So when troubleshooting connectivity, don't only ask:

> "Is SQL MI running?"

Ask:

> **"Can packets actually reach the service?"**

---

# 10. Route problems

You can have:

```text
DNS = successful
```

but:

```text
Network path = broken
```

For example:

```text
Application
    |
    v
DNS resolves MI
    |
    v
Wrong route
    |
    X
    |
    v
MI
```

This produces a very different troubleshooting path.

---

# 11. Authentication vs authorization

This distinction is extremely important.

### Authentication

> **Who are you?**

Examples:

```text
SQL login
Microsoft Entra identity
Managed identity
```

### Authorization

> **What are you allowed to do?**

Example:

```text
Login succeeds
      ↓
Database connection succeeds
      ↓
SELECT succeeds
      ↓
UPDATE fails
```

That's an authorization problem, not an authentication problem.

---

# 12. Example

Application uses:

```text
Managed Identity:
app-prod
```

Connection reaches MI successfully.

Authentication succeeds.

But:

```sql id="3y9n5g"
SELECT *
FROM Sales.Orders;
```

returns a permission error.

Your troubleshooting chain is:

```text
Network       ✅
Authentication ✅
Authorization ❌
```

You don't need to investigate NSGs anymore.

---

# 13. Connection timeout vs login failure

These are also different clues.

### Timeout

Often investigate:

```text
DNS
Routing
Firewall/security
Network reachability
Endpoint
```

### Login/authentication error

Investigate:

```text
Credentials
Identity
Authentication configuration
Contained database user
Entra configuration
```

### Permission denied

Investigate:

```text
Database user
Roles
GRANT/DENY
Object permissions
```

The exact error matters.

---

# 14. Connection pooling

Here's a real production scenario.

Application has:

```text
500 application instances
```

Each creates many database connections.

Suddenly:

```text
Connection count → extremely high
```

The application team says:

> "SQL MI is slow."

But you discover:

```text
Queries = normal
CPU = normal
IO = normal
Connections = huge
```

The problem may be application connection management.

For example:

```text
Bad pattern:

Request
  ↓
Open connection
  ↓
Query
  ↓
Don't close/reuse properly
  ↓
Repeat
```

A properly configured connection pool can dramatically reduce connection overhead.

---

# 15. Connection storms

Imagine a service outage occurs.

At 10:00 AM:

```text
Application unavailable
```

At 10:02 AM:

```text
Application recovers
```

Now **every application instance simultaneously reconnects**.

You get:

```text
10,000 connection attempts
       ↓
Connection storm
       ↓
Database/application pressure
       ↓
More failures
```

This is why production systems should use appropriate:

```text
Connection pooling
Retry policies
Backoff
Timeouts
Circuit breakers
```

Retries without backoff can actually make an outage worse.

---

# 16. Senior DBA troubleshooting flow

Memorize this:

```text
APPLICATION CANNOT CONNECT
            |
            v
       DNS resolution?
        /          \
      NO            YES
      |              |
    DNS fix       Network reachable?
                    /       \
                  NO         YES
                  |           |
             Network fix   Authentication?
                              /      \
                            NO        YES
                            |          |
                       Identity fix  Authorization?
                                        /     \
                                      NO       YES
                                      |         |
                                  Permission   DB/application
                                  fix          investigation
```

This is a **much better troubleshooting method** than repeatedly restarting the database.

---

# Lesson 54 — Azure SQL MI Authentication & Security

Now let's go one layer deeper.

Suppose your application connects successfully to MI.

You now need to decide:

> **How should the application authenticate?**

Common enterprise choices include:

```text
SQL authentication
Microsoft Entra authentication
Managed identity
```

---

## 1. SQL authentication

Conceptually:

```text
Application
    |
    +-- username/password
    |
    v
SQL MI
```

Advantages:

```text
Simple
Widely supported
Familiar
```

But you have to manage:

```text
Password storage
Rotation
Secrets
Exposure risk
```

---

# 2. Microsoft Entra authentication

Instead of relying exclusively on SQL passwords:

```text
Application
    |
    v
Microsoft Entra identity
    |
    v
SQL MI
```

This can integrate authentication with the organization's identity platform.

---

# 3. Managed identity

For Azure-hosted applications, managed identity can reduce the need to store database passwords.

Conceptually:

```text
Azure App
    |
    | Managed Identity
    v
Microsoft Entra
    |
    v
SQL MI
```

The application doesn't have to embed a long-lived SQL password in its configuration.

That's a major security advantage when implemented correctly.

---

# 4. But identity isn't permission

This is a common interview trap.

You create:

```text
Managed Identity = app-prod
```

and authentication works.

But the application gets:

```text
SELECT permission denied
```

Why?

Because:

```text
Identity exists
       ↓
Authentication
       ↓
Database user/permissions
       ↓
Authorization
```

You still need to configure the database-side authorization appropriately.

---

# 5. Least privilege

Suppose an application only needs:

```text
SELECT
INSERT
UPDATE
```

Don't give:

```text
db_owner
```

just to make the error disappear.

Instead:

```text
Application identity
       ↓
Database user
       ↓
Appropriate role/permissions
       ↓
Required objects
```

This reduces blast radius if the application is compromised.

---

# 6. Security incident example

Suppose an attacker compromises:

```text
app-prod
```

If it has:

```text
db_owner
```

the attacker potentially has a much larger attack surface.

If it only has:

```text
SELECT
INSERT
UPDATE
```

on required tables, the blast radius can be significantly smaller.

That's why:

> **Least privilege is an operational security control, not merely a compliance checkbox.**

---

# 7. Interview question

> **"What authentication method would you recommend for an Azure-hosted application connecting to SQL MI?"**

Good answer:

> "Where supported by the application and architecture, I'd prefer Microsoft Entra-based authentication, including managed identity for Azure-hosted workloads, because it can reduce dependence on long-lived database passwords. I'd then apply least-privilege database permissions to the application identity. SQL authentication remains an option when compatibility or application requirements make it necessary."

That's a strong answer because it doesn't pretend one authentication mechanism fits every workload.

---

