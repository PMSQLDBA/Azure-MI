# Lesson 3 — Azure SQL Managed Instance Networking

For an Azure DBA, **networking is one of the most important MI topics**.

A database can be perfectly healthy, but if networking is wrong:

> **Application → ❌ → SQL Managed Instance**

Let's understand the complete connection path.

---

## 1. Our architecture

For ABC Retail, imagine:

```text
                    ON-PREMISES
                        |
                 SQL Server 2019
                        |
                        |
                 VPN / ExpressRoute
                        |
                        v
                 +-------------+
                 | Azure VNet  |
                 |             |
                 | App Subnet  |
                 |     |       |
                 |     v       |
                 | Application |
                 |             |
                 | MI Subnet   |
                 |     |       |
                 |     v       |
                 | SQL MI      |
                 +-------------+
```

Azure SQL Managed Instance is deployed into an **Azure Virtual Network**, using a subnet dedicated to the Managed Instance.

---

# 2. What is a VNet?

If you're coming from traditional networking, think of an Azure VNet as your **private network inside Azure**.

For example:

```text
VNet: 10.10.0.0/16
```

Inside it:

```text
10.10.1.0/24 → Application subnet
10.10.2.0/24 → MI subnet
10.10.3.0/24 → Other services
```

Conceptually:

```text
10.10.0.0/16
      |
      +── 10.10.1.0/24
      |       |
      |       +── Application
      |
      +── 10.10.2.0/24
              |
              +── SQL Managed Instance
```

---

# 3. Why does MI need a dedicated subnet?

Managed Instance isn't just a single database endpoint.

Azure deploys and manages multiple underlying components required for:

* SQL engine
* Management
* High availability
* Networking
* Service operations

Therefore, MI has specific subnet requirements.

Your architecture should be thought of as:

```text
VNet
 |
 +-- Application subnet
 |
 +-- MI dedicated subnet
```

**Don't treat the MI subnet like a normal VM subnet.**

There are specific delegation, routing, NSG, DNS and subnet configuration requirements.

This is one reason MI networking can feel more complicated than Azure SQL Database networking.

---

# 4. Application → MI connection

Let's follow one request.

A customer opens your website:

```text
Customer
   |
   v
Web Application
   |
   v
Application Server
   |
   v
SQL Managed Instance
```

The application might have a connection string conceptually like:

```text
Server=tcp:<managed-instance-endpoint>,1433;
Database=SalesDB;
...
```

The application doesn't need to know:

```text
Which physical server?
Which storage?
Which replica?
Which underlying host?
```

That's Azure's responsibility.

The application connects to the MI endpoint.

---

# 5. DNS becomes important

Imagine your MI endpoint is something like:

```text
mycompany-mi.<unique-domain>
```

The application needs to resolve that hostname.

So:

```text
Application
    |
    | DNS query
    v
DNS
    |
    | IP address
    v
Managed Instance
```

If DNS isn't configured correctly:

```text
Application
    |
    | "Where is my SQL Server?"
    |
    X
   DNS failure
```

You can have a perfectly healthy SQL MI and still have:

> **Application cannot connect.**

---

# 6. The classic DBA troubleshooting question

Developer says:

> "SQL MI is down."

Don't immediately believe that.

Test systematically:

```text
1. Can DNS resolve the MI hostname?
2. Can the application reach the network?
3. Is routing correct?
4. Is the required port reachable?
5. Are NSG/firewall rules correct?
6. Is authentication working?
7. Is SQL MI actually healthy?
```

That's a much better DBA approach.

---

# 7. NSG — Network Security Group

Think of an NSG as a network traffic filter.

Conceptually:

```text
Application
     |
     | TCP traffic
     v
   NSG
     |
     +---- ALLOW → SQL MI
     |
     +---- DENY  → Unauthorized traffic
```

For example:

```text
Application subnet
       |
       | TCP 1433
       |
       v
MI subnet
```

If your network security rules don't allow the required communication:

```text
Application
    |
    X
 NSG / routing
    |
    X
SQL MI
```

Again:

> SQL itself can be completely healthy.

---

# 8. SQL port

Traditional SQL Server commonly uses:

```text
TCP 1433
```

The important concept is not to memorize one port and assume every Azure configuration works identically.

Instead:

> **Know which endpoint and port your specific MI deployment/application configuration requires, and verify network access accordingly.**

For troubleshooting, tools such as:

```text
nslookup
Test-NetConnection
```

can help determine whether the problem is DNS or TCP connectivity.

For example:

```powershell
Test-NetConnection <MI-hostname> -Port 1433
```

If:

```text
TcpTestSucceeded : False
```

you investigate networking before investigating SQL queries.

---

# 9. On-premises → Azure MI

Now let's make it more realistic.

ABC Retail still has:

```text
ON-PREMISES
|
+-- Active Directory
+-- Application
+-- SQL Server
```

They're migrating gradually.

Azure:

```text
AZURE
|
+-- VNet
    |
    +-- Application
    +-- SQL MI
```

They need:

```text
On-premises
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

This creates a **hybrid architecture**.

---

# 10. VPN vs ExpressRoute

At a high level:

### VPN

Uses encrypted connectivity over the internet.

```text
On-prem
   |
   | Encrypted VPN
   |
   v
Azure
```

### ExpressRoute

Provides private connectivity through an ExpressRoute provider.

```text
On-prem
   |
   | Private connectivity
   |
   v
Azure
```

For enterprise production systems, the choice depends on:

```text
Security
Latency
Bandwidth
Reliability
Cost
Existing network architecture
Business requirements
```

Don't think:

> "ExpressRoute is always required."

It isn't.

---

# 11. Real production scenario

Suppose the application is still on-premises:

```text
On-prem Application
        |
        |
        v
      Azure
        |
        v
      SQL MI
```

The application makes:

```text
SELECT
INSERT
UPDATE
EXEC stored procedure
```

across the network.

Now latency matters.

Imagine:

```text
Application → MI
Network latency = 2 ms
```

Excellent.

But imagine:

```text
Application → MI
Network latency = 80 ms
```

Now an application that performs thousands of small database round trips could become significantly slower.

For example:

```text
Application
   |
   +--> SELECT
   |
   +--> SELECT
   |
   +--> SELECT
   |
   +--> UPDATE
   |
   +--> SELECT
```

Each trip crosses the network.

So:

> **Database performance isn't only about the database. Application/network architecture matters.**

---

# 12. The N+1 query problem

Imagine your application wants 1,000 customers.

Instead of:

```sql
SELECT *
FROM Customer
WHERE CustomerID IN (...);
```

it does:

```text
SELECT customer 1
SELECT customer 2
SELECT customer 3
...
SELECT customer 1000
```

Now you have:

```text
1000 network round trips
```

If latency is significant:

```text
1000 × network latency
```

can become painful.

This is why moving SQL Server to Azure can expose application architecture problems that weren't obvious when the application and database were sitting on the same LAN.

---

# 13. Private connectivity

Enterprise architecture generally prefers keeping database traffic private.

Conceptually:

```text
Internet
   |
   X
   |
   |      No direct public path
   |
Application
   |
Private VNet
   |
   v
SQL MI
```

This provides network isolation.

Microsoft's architecture guidance emphasizes VNet integration and private networking for Managed Instance.

---

# 14. DNS architecture

Here's a scenario you'll encounter in real projects.

On-premises DNS knows:

```text
company.local
```

Azure uses:

```text
Azure DNS / custom DNS
```

Your application needs to resolve:

```text
my-mi.database.windows.net
```

You need to ensure DNS resolution works correctly across the hybrid network.

Conceptually:

```text
Application
     |
     v
DNS Resolver
     |
     +---- On-prem DNS
     |
     +---- Azure DNS
     |
     v
MI endpoint
```

Incorrect DNS can produce confusing symptoms:

```text
Ping works?      Maybe irrelevant.
DNS resolves?   Critical.
TCP works?      Critical.
SQL login works? Authentication issue.
```

---

# 15. Important troubleshooting sequence

Suppose developer says:

> **"The application cannot connect to SQL MI."**

You should follow this order:

```text
                    Connection Failure
                           |
                           v
                    DNS resolution?
                       /       \
                     NO         YES
                     |           |
                  Fix DNS        v
                            TCP connectivity?
                              /       \
                            NO         YES
                            |           |
                         Network       v
                          issue    Authentication?
                                      /      \
                                    NO        YES
                                    |          |
                                Fix auth      SQL/application
                                              investigation
```

This is much better than immediately opening SSMS and staring at the database.

---

# 16. Authentication

Once network connectivity works:

```text
Application
     |
     v
SQL MI
     |
     v
Authentication
```

You may use:

```text
SQL authentication
Microsoft Entra authentication
```

Then authorization:

```text
Login
  ↓
User
  ↓
Role
  ↓
Permission
  ↓
Database object
```

Example:

```text
Application login
       |
       v
SalesDB
       |
       +-- SELECT
       +-- INSERT
       +-- UPDATE
```

---

# 17. A very common production mistake

Someone says:

> "The login exists, but the application still gets permission denied."

Check:

```text
Login exists?
      ↓
User mapped?
      ↓
Correct database?
      ↓
Role membership?
      ↓
Object permission?
```

For example:

```text
Login
  ↓
User
  ↓
SalesDB
  ↓
db_datareader
```

The login existing at the instance level doesn't automatically mean the user has every database permission.

That's standard SQL Server security thinking, and it remains important in MI.

---

# 18. Let's simulate a production incident

ABC Retail has just migrated to MI.

At 9:00 AM:

```text
Application → SQL MI
```

Everything works.

At 10:00 AM:

> **Users cannot log in.**

You check MI.

```text
MI status: Healthy
CPU: 30%
Memory: Normal
Storage: Normal
```

Developer says:

> "SQL is down."

What do you do?

### Step 1

Check DNS.

```text
nslookup <MI-endpoint>
```

### Step 2

Check TCP.

```text
Test-NetConnection <MI-endpoint> -Port 1433
```

### Step 3

Check network path.

```text
Application
   ↓
VNet
   ↓
Subnet
   ↓
NSG
   ↓
MI
```

### Step 4

Check authentication.

### Step 5

Check whether the application changed its connection string.

This prevents you from wasting an hour troubleshooting SQL queries when the real problem is networking.

---

# 19. The architecture you should memorize

For an enterprise MI deployment:

```text
                  USERS
                    |
                    v
              APPLICATION
                    |
                    v
              PRIVATE NETWORK
                    |
              +-----+------+
              |            |
              v            v
          Azure VNet    On-Prem
              |            |
              |       VPN/ExpressRoute
              |
              v
          MI Subnet
              |
              v
       SQL Managed Instance
              |
       +------+------+
       |             |
       v             v
    Databases      HA/DR
```

This is the mental picture I want you to have.

---

# 20. Your DBA challenge

Now imagine this production incident:

> **ABC Retail migrated SQL Server to Azure SQL Managed Instance. The database is healthy. SSMS from a DBA workstation can connect. But the application cannot connect.**

You discover:

```text
DNS resolution from DBA laptop: SUCCESS
DNS resolution from application server: FAIL
SQL MI status: HEALTHY
Database status: ONLINE
CPU: 25%
```

### Question:

Where would you investigate first?

A. Rebuild indexes
B. Increase vCores
C. Query Store
D. DNS/network configuration on the application side

**Choose A, B, C, or D and explain why in one or two sentences.**
