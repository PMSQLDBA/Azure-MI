# Lesson 89 — Azure SQL Managed Instance Networking

Networking is one of the areas where Azure SQL Managed Instance differs significantly from Azure SQL Database.

If you're working as a SQL Server DBA, don't think only:

> "What server name and port do I use?"

With Managed Instance, you need to understand **VNet, subnet, DNS, routing, NSGs, connectivity, and hybrid networking**.

---

# 1. The big picture

A typical production architecture looks like:

```text
                         Internet
                            |
                            X
                            |
                    +----------------+
                    | Azure Network  |
                    +----------------+
                            |
                         VNet
                            |
             +--------------+--------------+
             |                             |
             ↓                             ↓
       Application                    SQL MI Subnet
          Subnet                            |
             |                              ↓
             ↓                       +-------------+
       App Service / VM              | SQL Managed |
             |                       |  Instance   |
             |                       +-------------+
             |
             +-------- Private connectivity --------+
```

The key idea:

> **SQL Managed Instance is deployed into an Azure virtual network.**

---

# 2. Why does SQL MI need networking?

Traditional SQL Server:

```text
Application
    |
    ↓
SQL Server
    |
    ↓
Network
```

SQL MI:

```text
Application
    |
    ↓
Azure networking
    |
    ↓
VNet
    |
    ↓
SQL MI
```

Azure networking determines whether the application can actually reach the database.

---

# 3. The major networking components

You should know these:

```text
VNet
Subnet
IP address
DNS
NSG
Route table
UDR
Private connectivity
VPN
ExpressRoute
Peering
Firewall
Ports
```

We'll go through each.

---

# 4. VNet

A **Virtual Network (VNet)** is the private network boundary in Azure.

Think of it like a corporate network.

Example:

```text
10.10.0.0/16
```

Inside:

```text
+--------------------------------------+
| Azure VNet                           |
|                                      |
|  App Subnet                          |
|  10.10.1.0/24                       |
|                                      |
|  SQL MI Subnet                       |
|  10.10.2.0/24                       |
|                                      |
+--------------------------------------+
```

---

# 5. Subnet

A subnet divides the VNet into smaller network segments.

For example:

```text
VNet
10.10.0.0/16
        |
        +---- App subnet
        |     10.10.1.0/24
        |
        +---- SQL MI subnet
              10.10.2.0/24
```

SQL Managed Instance requires a dedicated subnet for the managed instances.

This is an important point:

> **You don't simply deploy SQL MI into the same general-purpose subnet as random application resources.**

---

# 6. Why dedicated subnet?

SQL MI has infrastructure requirements that are different from ordinary VMs.

The dedicated subnet allows Azure to manage the networking and underlying service infrastructure appropriately.

Think:

```text
SQL MI subnet
      |
      +-- SQL MI infrastructure
      +-- Required networking
      +-- Service operations
```

---

# 7. Production example

Suppose your company has:

```text
VNet: 10.20.0.0/16
```

Architecture:

```text
10.20.0.0/16
       |
       +-----------------------+
       |                       |
       ↓                       ↓
App subnet               SQL MI subnet
10.20.1.0/24             10.20.10.0/24
       |                       |
       ↓                       ↓
App/VMs                  Managed Instance
```

The application needs to reach SQL MI privately.

---

# 8. SQL MI is not just a VM

This is an important mindset.

You might think:

```text
SQL MI = VM with SQL Server installed
```

Not exactly.

Azure manages much of the underlying infrastructure.

You interact with:

```text
SQL MI endpoint
SQL databases
SQL configuration
Networking
Security
```

rather than managing the operating system like a normal SQL Server VM.

---

# 9. DNS

DNS is one of the most common causes of connection problems.

Application connects using a hostname such as:

```text
myserver.database.windows.net
```

The hostname needs to resolve correctly.

Conceptually:

```text
Application
    |
    ↓
DNS lookup
    |
    ↓
SQL MI hostname
    |
    ↓
IP address
    |
    ↓
SQL MI
```

If DNS fails:

```text
Application
    |
    ↓
DNS
    X
```

the application won't even reach the database.

---

# 10. Real production incident

Application team says:

> "SQL Managed Instance is down."

You check SQL MI.

It is healthy.

You test connectivity from a VM.

It works.

Then you test from the application environment.

It fails.

This strongly suggests:

```text
SQL MI itself
     ✓
Network path
     ?
DNS
     ?
Application environment
     ?
```

Don't immediately restart SQL.

---

# 11. First troubleshooting question

Ask:

> **Can the application resolve the SQL MI hostname?**

Test conceptually:

```text
nslookup <SQL-MI-hostname>
```

or appropriate DNS tools from the application's network environment.

If DNS doesn't resolve:

```text
DNS problem
```

If it resolves:

```text
Continue to network connectivity.
```

---

# 12. Connectivity troubleshooting hierarchy

Use this order:

```text
1. DNS resolution
       ↓
2. Network route
       ↓
3. NSG / firewall
       ↓
4. Port connectivity
       ↓
5. TLS
       ↓
6. Authentication
       ↓
7. SQL authorization
       ↓
8. Application configuration
```

This is a very useful DBA troubleshooting model.

---

# 13. Why this order matters

Suppose the error is:

```text
Login failed for user
```

That's different from:

```text
Connection timeout
```

### Login failed

Network probably worked.

### Connection timeout

Authentication may never have happened.

So don't troubleshoot SQL permissions when the network cannot reach SQL MI.

---

# 14. NSG

**Network Security Group (NSG)** controls network traffic based on rules.

Conceptually:

```text
Application
     |
     ↓
NSG
     |
     +---- Allow?
     |
     +---- Deny?
     |
     ↓
SQL MI
```

A badly configured NSG can block traffic.

---

# 15. Example NSG problem

Application:

```text
10.20.1.20
```

SQL MI subnet:

```text
10.20.10.0/24
```

Application tries to connect.

But NSG has:

```text
Deny traffic
```

Result:

```text
Application
    |
    ↓
NSG
    X
SQL MI
```

The SQL engine may be completely healthy.

---

# 16. Important: don't blindly change NSGs

A common junior DBA mistake:

> "Connection failed. I'll allow everything."

Never do:

```text
Allow:
0.0.0.0/0
Any port
Any protocol
```

just to make the error disappear.

Instead determine:

```text
Source
Destination
Port
Protocol
Required direction
```

Then create the minimum required rule.

---

# 17. Routing

Suppose:

```text
VNet A
   |
   ↓
VNet B
   |
   ↓
SQL MI
```

Azure needs a valid route between them.

Routes determine:

> **Where should this packet go?**

Conceptually:

```text
Application
     |
     ↓
Route table
     |
     ↓
Network path
     |
     ↓
SQL MI
```

---

# 18. User Defined Routes — UDR

A custom route can influence how traffic flows.

Example:

```text
10.20.10.0/24
      |
      ↓
Firewall
      |
      ↓
SQL MI
```

A badly designed UDR can send traffic somewhere it shouldn't go.

Result:

```text
Application
   |
   ↓
Unexpected route
   |
   X
SQL MI
```

---

# 19. Hub-and-spoke architecture

Enterprise Azure environments commonly use:

```text
                 Hub VNet
                    |
        +-----------+-----------+
        |                       |
        ↓                       ↓
    Spoke A                  Spoke B
        |                       |
        ↓                       ↓
Application                 SQL MI
```

The hub might contain:

```text
Azure Firewall
VPN Gateway
ExpressRoute Gateway
DNS services
Network monitoring
```

---

# 20. VNet peering

Suppose:

```text
VNet A
Application
```

and:

```text
VNet B
SQL MI
```

You can establish connectivity using appropriate Azure networking patterns such as VNet peering.

Conceptually:

```text
VNet A
App
 |
 +------ Peering ------+
                       |
                       ↓
                    VNet B
                      SQL MI
```

But peering alone isn't enough.

You still need:

```text
Routing
NSGs
DNS
Ports
```

to work.

---

# 21. Hybrid scenario

This is extremely important for real SQL Server environments.

Company has:

```text
On-premises
SQL Server
10.1.0.0/16
```

Azure has:

```text
Azure VNet
10.20.0.0/16
```

SQL MI:

```text
10.20.10.0/24
```

The company wants on-premises applications to access SQL MI.

Architecture:

```text
On-premises
    |
    ↓
VPN / ExpressRoute
    |
    ↓
Azure VNet
    |
    ↓
SQL MI
```

---

# 22. VPN

A site-to-site VPN creates an encrypted connection between networks.

Conceptually:

```text
Corporate Network
       |
       |
    VPN tunnel
       |
       |
Azure VNet
       |
       ↓
SQL MI
```

Good for many hybrid scenarios.

---

# 23. ExpressRoute

ExpressRoute provides private connectivity between on-premises infrastructure and Azure through supported connectivity providers.

Conceptually:

```text
Corporate DC
     |
     ↓
ExpressRoute
     |
     ↓
Azure
     |
     ↓
VNet
     |
     ↓
SQL MI
```

Enterprises may prefer this for predictable private connectivity and network architecture requirements.

---

# 24. VPN vs ExpressRoute

Simplified:

| VPN                                                   | ExpressRoute                             |
| ----------------------------------------------------- | ---------------------------------------- |
| Encrypted tunnel over supported internet connectivity | Private connectivity through provider    |
| Generally simpler                                     | More enterprise network integration      |
| Often lower complexity/cost                           | Usually more involved                    |
| Good for many hybrid cases                            | Useful for large enterprise connectivity |

The right choice depends on business requirements and network architecture.

---

# 25. Real-time hybrid migration scenario

Company currently has:

```text
On-prem SQL Server
```

They want to migrate to:

```text
Azure SQL Managed Instance
```

But the application remains on-premises initially.

Architecture:

```text
                 Azure
                  |
             SQL Managed
               Instance
                  |
                  |
          VPN / ExpressRoute
                  |
                  |
              On-prem
                  |
                  ↓
             Application
```

During the transition:

```text
Application
     |
     ↓
SQL MI
```

must work reliably across the hybrid network.

---

# 26. Migration troubleshooting

Application team says:

> "Migration succeeded, but the application can't connect."

Don't immediately assume SQL migration failed.

Check:

```text
DNS
Routing
VPN/ExpressRoute
NSGs
Ports
TLS
Authentication
Connection string
```

---

# 27. Ports

SQL Server commonly uses:

```text
TCP 1433
```

But don't assume every SQL MI connectivity scenario can be reduced to:

> "Open 1433 and everything works."

You must verify the specific endpoint and connectivity model you're using.

For SQL MI, connectivity can involve different endpoints and ports depending on configuration/use case.

---

# 28. Public vs private connectivity

SQL MI is designed around VNet-based networking.

Depending on configuration and supported features, you may have:

```text
Private connectivity
```

and potentially:

```text
Public endpoint
```

for specific access scenarios.

From a security perspective, prefer private connectivity where appropriate.

---

# 29. Public endpoint scenario

Imagine an organization has a controlled external application that needs database access.

Rather than exposing the entire network, the organization may configure an appropriate SQL MI public connectivity mechanism and tightly control access.

But ask:

```text
Do we actually need public connectivity?
```

If not:

```text
Keep traffic private.
```

---

# 30. Private connectivity scenario

Preferred enterprise pattern:

```text
Application
     |
     ↓
Private network
     |
     ↓
SQL MI
```

No direct public exposure is required for the application-to-database path.

---

# 31. DNS in hybrid environments

Here's a very common problem.

Azure SQL MI hostname:

```text
my-mi.xxxxxx.database.windows.net
```

Application is on-premises.

On-prem DNS doesn't know how to resolve the Azure hostname correctly.

Result:

```text
Application
   |
   ↓
DNS
   X
SQL MI
```

The database is healthy.

The network may be healthy.

But name resolution fails.

---

# 32. Split DNS

Enterprise environments may use DNS forwarding/resolution architecture so that:

```text
On-prem DNS
      |
      ↓
Azure DNS / resolver
      |
      ↓
Azure private name resolution
```

works correctly.

The exact implementation depends on your enterprise DNS architecture.

---

# 33. DNS troubleshooting

When connection fails:

```text
nslookup SQL-MI-hostname
```

Then ask:

```text
Does it resolve?
What IP does it return?
Is that IP reachable?
Is the response coming from the expected DNS infrastructure?
```

---

# 34. Connection test

After DNS:

```text
Test-NetConnection <hostname> -Port <port>
```

or an equivalent network connectivity test.

This tells you whether the TCP connection can be established.

---

# 35. Interpret results

### DNS fails

```text
DNS problem
```

### DNS succeeds, TCP fails

Likely:

```text
Routing
NSG
Firewall
Port
```

### TCP succeeds, SQL login fails

Likely:

```text
Authentication
```

### Login succeeds, query fails

Likely:

```text
Authorization
```

This troubleshooting flow is extremely useful.

---

# 36. Complete troubleshooting tree

```text
Application can't connect
          |
          ↓
Can hostname resolve?
      /          \
    NO            YES
    |              |
   DNS             ↓
               Can TCP connect?
                 /       \
               NO         YES
               |           |
        Network path       ↓
        NSG/firewall    Authentication
                            |
                            ↓
                       Authorization
                            |
                            ↓
                       Application
                        behavior
```

Memorize this.

---

# 37. Real scenario — timeout

Error:

```text
Connection timeout
```

DBA checks:

```text
SQL MI status = Healthy
```

Then:

```text
DNS = Successful
TCP = Failed
```

Conclusion:

> Don't troubleshoot SQL users yet.

Investigate:

```text
NSG
Firewall
Route
VPN
ExpressRoute
VNet peering
Port
```

---

# 38. Real scenario — login failed

Error:

```text
Login failed for user
```

DBA tests TCP:

```text
Successful
```

Therefore:

```text
Network path = working
```

Now investigate:

```text
Username
Password
Microsoft Entra token
Managed identity
SQL user
Database user
Authentication method
```

---

# 39. Real scenario — SELECT denied

Error conceptually:

```text
The SELECT permission was denied
```

Network:

```text
✓
```

Authentication:

```text
✓
```

Authorization:

```text
❌
```

Fix:

```text
Database role
Permissions
User mapping
```

Don't modify networking.

---

# 40. Network Security Groups

When troubleshooting NSGs, understand:

```text
Source
Destination
Protocol
Port
Direction
Priority
```

For example:

```text
Source:
10.20.1.0/24

Destination:
SQL MI subnet

Protocol:
TCP

Port:
required SQL MI endpoint port

Action:
Allow
```

---

# 41. NSG priority

NSG rules have priorities.

A lower numerical priority is evaluated before a higher numerical priority.

For example:

```text
100 → Deny
200 → Allow
```

The deny can prevent the later allow from being effective.

So don't just look for an "Allow" rule.

Look at the complete rule evaluation.

---

# 42. Azure Firewall

Large enterprises may place Azure Firewall in the traffic path.

Example:

```text
Application
    |
    ↓
Azure Firewall
    |
    ↓
SQL MI
```

Then you need to validate:

```text
Firewall policy
Network rule
Application rule where relevant
Routing
DNS
```

---

# 43. Firewall troubleshooting

Suppose:

```text
App → SQL MI
```

fails.

SQL MI is healthy.

NSG is correct.

Routes look correct.

Then:

```text
Azure Firewall
```

may be dropping traffic.

Check firewall logs and network rules.

---

# 44. Service endpoints vs private endpoints

These are commonly confused.

### Service endpoint

Extends VNet identity to supported Azure services.

### Private endpoint

Provides a private IP-based interface into a supported Azure service.

However, **SQL Managed Instance has its own VNet-integrated architecture**, so don't blindly apply generic Azure SQL Database networking patterns to SQL MI.

This distinction is important in interviews.

---

# 45. SQL MI DNS zones

SQL MI networking relies heavily on correct DNS configuration.

When integrating with custom DNS infrastructure, you need to ensure Azure and SQL MI-related names can resolve correctly.

A custom DNS setup that works for your VMs can still be incorrectly configured for SQL MI.

---

# 46. Production architecture

A mature enterprise design may look like:

```text
                     ON-PREMISES
                         |
                   ExpressRoute
                         |
                         ↓
                    HUB VNET
                +----------------+
                | Firewall       |
                | DNS            |
                | Gateway        |
                +-------+--------+
                        |
                     Peering
                        |
                        ↓
                 SPOKE VNET
                +---------------+
                |               |
                ↓               ↓
          App subnet       SQL MI subnet
                |               |
                ↓               ↓
          Application       SQL MI
```

---

# 47. Why hub-and-spoke?

Centralize:

```text
Firewall
DNS
Connectivity
Monitoring
Security controls
```

while keeping workloads separated.

For example:

```text
Production spoke
Development spoke
Testing spoke
```

---

# 48. Production vs development networking

Don't necessarily put everything into one flat network.

Better:

```text
Production VNet
       |
       ↓
Production SQL MI

Development VNet
       |
       ↓
Development SQL MI
```

This reduces accidental cross-environment access.

---

# 49. Network segmentation

Think:

```text
Internet
   X
   |
   ↓
Application layer
   |
   ↓
Database layer
```

The database should not be directly reachable by every workload.

Only required applications should have access.

---

# 50. Real-time scenario — developer laptop

A developer says:

> "I can connect to production SQL MI from the application server, but not from my laptop."

This may actually be correct.

Why?

Production SQL MI may be intentionally private.

Architecture:

```text
Developer laptop
      X
      |
Private SQL MI
```

while:

```text
Application
    |
    ↓
Private network
    |
    ↓
SQL MI
```

works.

---

# 51. How should developers connect?

Instead of exposing SQL MI publicly just for developer convenience, enterprises may use controlled connectivity such as:

```text
VPN
Bastion/jump host
Private connectivity
Approved administrative network
```

depending on the organization's architecture.

---

# 52. DBA access architecture

A secure DBA model:

```text
DBA Laptop
    |
    ↓
VPN / Bastion
    |
    ↓
Private network
    |
    ↓
SQL MI
```

rather than:

```text
Internet
   |
   ↓
SQL MI
```

---

# 53. Connection strings

A typical application connection string contains information such as:

```text
Server
Database
Authentication method
Encryption settings
```

But avoid embedding:

```text
Permanent passwords
```

where managed identity/Entra authentication can be used.

---

# 54. TLS troubleshooting

Suppose:

```text
DNS ✓
TCP ✓
Authentication ❌
```

But credentials appear correct.

Possible issue:

```text
TLS/certificate
Driver compatibility
Encryption configuration
Authentication library
```

Check the application's SQL driver and supported authentication configuration.

---

# 55. Old SQL drivers

A legacy application may use an old SQL client library.

You migrate from:

```text
SQL Server 2016
```

to:

```text
Azure SQL MI
```

Then the application fails.

Potential cause:

```text
Old driver
```

especially when modern authentication or encryption requirements are involved.

Don't assume the SQL MI is broken.

---

# 56. Network vs SQL troubleshooting matrix

| Symptom                        | First area to check                        |
| ------------------------------ | ------------------------------------------ |
| DNS name doesn't resolve       | DNS                                        |
| Timeout                        | Routing / NSG / firewall                   |
| TCP connection refused/fails   | Network/endpoint/port                      |
| Login failed                   | Authentication                             |
| Permission denied              | Authorization                              |
| TLS error                      | Encryption/driver                          |
| Query timeout after connection | SQL/performance                            |
| Intermittent connectivity      | Network/routing/DNS/load/resource behavior |

---

# 57. SQL Server DBA mindset

Traditional DBA:

```text
Check SQL Server
Check port
Check listener
```

Azure SQL MI DBA:

```text
Check SQL MI
   +
VNet
   +
Subnet
   +
DNS
   +
Route
   +
NSG
   +
Firewall
   +
Identity
```

Your scope becomes broader.

---

# 58. Interview question

### "Where is SQL Managed Instance deployed?"

Answer:

> "SQL Managed Instance is deployed into a dedicated Azure virtual network subnet, allowing it to integrate with Azure networking and private enterprise connectivity."

---

# 59. Interview question

### "Application cannot connect to SQL MI. What do you check?"

Strong answer:

> "I start by checking DNS resolution, then TCP connectivity and routing, followed by NSGs/firewalls and the required endpoint/port. If network connectivity succeeds, I move to TLS, authentication, SQL authorization, and finally application configuration."

That answer demonstrates structured troubleshooting.

---

# 60. Interview question

### "DNS works but connection times out. What next?"

Answer:

> "I would check routing, NSGs, Azure Firewall or other network security controls, VPN/ExpressRoute if hybrid, VNet peering, and whether the required endpoint/port is reachable."

---

# 61. Interview question

### "SQL login failed. Is networking broken?"

Not necessarily.

If you receive an actual SQL authentication failure, the network path may already be working.

Investigate:

```text
Authentication
```

rather than immediately changing firewall rules.

---

# 62. Interview question

### "Can an on-premises application connect to SQL MI?"

Yes, with appropriate hybrid network connectivity and configuration.

Typical architecture:

```text
On-premises
     |
     ↓
VPN / ExpressRoute
     |
     ↓
Azure VNet
     |
     ↓
SQL MI
```

You must configure:

```text
Routing
DNS
Network security
Authentication
```

correctly.

---

# 63. The most important troubleshooting sequence

Memorize:

```text
                 CONNECTION FAILURE
                         |
                         ↓
                    DNS resolve?
                    /          \
                  NO            YES
                  |              |
                 DNS             ↓
                          TCP connection?
                           /          \
                         NO            YES
                         |              |
                 Routing/NSG/Firewall   ↓
                                  Authentication?
                                    /       \
                                  NO         YES
                                  |           |
                              Identity       ↓
                                          Authorization?
                                            /      \
                                          NO        YES
                                          |          |
                                       SQL perms   Application/
                                                  SQL behavior
```

This is one of the most useful mental models for Azure SQL MI support.

---

# 64. Networking checklist

```text
VNET
☐ Correct VNet
☐ Dedicated SQL MI subnet
☐ Correct address space
☐ No overlapping ranges

DNS
☐ Hostname resolves
☐ Correct DNS servers
☐ Hybrid DNS configured
☐ Custom DNS doesn't break resolution

ROUTING
☐ Route exists
☐ VNet peering configured
☐ VPN/ExpressRoute routes work
☐ UDRs reviewed

SECURITY
☐ NSG rules
☐ Azure Firewall
☐ Required port
☐ Correct source/destination

APPLICATION
☐ Correct hostname
☐ Correct port
☐ TLS
☐ SQL driver
☐ Authentication
☐ Connection timeout

HYBRID
☐ VPN/ExpressRoute
☐ DNS forwarding
☐ Routes
☐ Firewall
```

---

# 65. Real production incident — complete walkthrough

Let's say:

```text
Application:
OrderAPI

SQL:
SQL Managed Instance

Environment:
Production
```

At 10:30 AM:

```text
OrderAPI
   X
SQL MI
```

Application reports:

```text
Connection timeout
```

### Step 1 — SQL MI health

```text
SQL MI = Healthy
```

So don't restart SQL MI.

### Step 2 — DNS

```text
nslookup SQL-MI-hostname
```

Result:

```text
Success
```

Continue.

### Step 3 — TCP

```text
Test-NetConnection
```

Result:

```text
Failed
```

Now you know:

```text
DNS ✓
Network connectivity ❌
```

### Step 4 — Check route

Route is correct.

### Step 5 — Check NSG

A new NSG rule was deployed yesterday:

```text
Deny application subnet → SQL MI subnet
```

There's your problem.

### Step 6

Correct the rule according to security policy.

### Step 7

Retest TCP.

```text
Success
```

### Step 8

Application reconnects.

Incident resolved.

---

# 66. What made this a good DBA investigation?

You didn't:

```text
Restart SQL MI
Change SQL passwords
Grant db_owner
Open the internet
```

Instead you followed:

```text
DNS
 ↓
TCP
 ↓
Network security
 ↓
SQL
```

That's disciplined troubleshooting.

---

# 67. The key lesson

When someone says:

> **"SQL Managed Instance is not reachable."**

Don't immediately think:

> "SQL problem."

Think:

```text
DNS
 ↓
Routing
 ↓
NSG
 ↓
Firewall
 ↓
Port
 ↓
TLS
 ↓
Authentication
 ↓
Authorization
 ↓
SQL
```

That mindset will save you enormous amounts of troubleshooting time.

---

## Next — Lesson 90: Azure SQL Managed Instance HA & DR

We'll go deep into **automatic failover, service tiers, zone redundancy, failover groups, geo-replication concepts, RPO/RTO, primary/secondary regions, planned vs unplanned failover, application connection behavior, DNS during failover, disaster scenarios, and a full production DR exercise**.
