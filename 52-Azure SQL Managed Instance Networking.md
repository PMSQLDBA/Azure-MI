# Lesson 84 — Azure SQL Managed Instance Networking

This is one of the **most important Azure SQL MI topics for a SQL Server DBA**.

A DBA may know SQL perfectly and still struggle in Azure if they don't understand networking.

The key mental model is:

```text
Application
     |
     ↓
DNS
     |
     ↓
Network route
     |
     ↓
NSG / network controls
     |
     ↓
SQL MI endpoint
     |
     ↓
TCP connection
     |
     ↓
SQL authentication
     |
     ↓
Database authorization
```

When SQL MI connectivity fails, **don't jump directly to SQL authentication**.

Troubleshoot from the bottom up.

---

# 1. What is a VNet?

A **Virtual Network (VNet)** is the fundamental private networking boundary in Azure.

Think of it like your company's internal network.

On-premises:

```text
Corporate Network
       |
       +---- Server A
       +---- Server B
       +---- SQL Server
```

Azure:

```text
Azure VNet
       |
       +---- Application
       +---- VM
       +---- SQL MI
```

---

# 2. Typical SQL MI architecture

Conceptually:

```text
                 Azure VNet
        +---------------------------+
        |                           |
        |   App Subnet              |
        |      |                    |
        |      |                    |
        |   Application             |
        |                           |
        |                           |
        |   SQL MI Subnet           |
        |      |                    |
        |    SQL MI                 |
        |                           |
        +---------------------------+
```

SQL Managed Instance uses Azure virtual networking and requires a properly configured subnet.

---

# 3. Why SQL MI needs networking knowledge

With a traditional on-prem SQL Server:

```text
Application
    |
Corporate LAN
    |
SQL Server
```

Your network team controls:

```text
Switches
Routers
Firewalls
DNS
Subnets
```

With SQL MI:

```text
Application
    |
Azure VNet
    |
Subnet
    |
SQL MI
```

You need to understand Azure networking concepts as well.

---

# 4. Subnet

A subnet divides a VNet into smaller network segments.

Example:

```text
VNet: 10.10.0.0/16

       |
       +-------------------+
       |                   |
       ↓                   ↓
App subnet             SQL MI subnet
10.10.1.0/24           10.10.2.0/24
```

The SQL MI subnet is dedicated to the managed instance.

---

# 5. Why subnet design matters

You shouldn't randomly place everything into one subnet.

A production architecture might separate:

```text
Application subnet
Database subnet
Management subnet
Integration subnet
```

This allows you to apply appropriate network controls.

---

# 6. SQL MI subnet

SQL Managed Instance requires a subnet configured specifically for SQL MI.

Azure manages infrastructure inside that subnet.

Think:

```text
Your VNet
    |
    +---- SQL MI delegated subnet
                |
                +---- Managed Instance
```

You shouldn't treat it like a normal VM subnet where you manually install a SQL Server.

---

# 7. Delegated subnet concept

Azure supports subnet delegation, which allows certain Azure services to be deployed into a subnet under service-specific rules.

For SQL MI:

```text
VNet
  |
  ↓
Dedicated subnet
  |
  ↓
SQL Managed Instance
```

The subnet needs to meet the SQL MI networking requirements.

---

# 8. Don't put random workloads in the SQL MI subnet

A common design mistake is treating the MI subnet as a general-purpose subnet.

Think:

```text
Bad:

SQL MI subnet
   |
   +-- SQL MI
   +-- VM
   +-- Random service
```

Instead:

```text
Better:

SQL MI subnet
   |
   +-- SQL MI infrastructure
```

Follow Microsoft's current SQL MI networking requirements when designing the subnet.

---

# 9. IP addressing

Suppose:

```text
VNet
10.10.0.0/16
```

You might have:

```text
10.10.1.0/24 → Application
10.10.2.0/24 → SQL MI
10.10.3.0/24 → Management
```

The exact address plan depends on the environment.

The important concept is:

```text
VNet
 ↓
Subnet
 ↓
IP address
 ↓
Routing
```

---

# 10. DNS — extremely important

DNS converts names into IP addresses.

Example:

```text
myserver.database.windows.net
```

conceptually becomes:

```text
DNS
 ↓
IP / endpoint information
```

Applications normally connect using names rather than manually configured IP addresses.

---

# 11. Why DBAs need DNS knowledge

Imagine your application says:

```text
Cannot resolve database hostname
```

SQL Server might be completely healthy.

The problem may be:

```text
DNS
```

not SQL.

This distinction saves hours of troubleshooting.

---

# 12. Connection path

Suppose:

```text
Application
```

uses:

```text
Server=sqlmi-prod.company.internal;
```

The connection process is approximately:

```text
Application
   ↓
DNS lookup
   ↓
IP address
   ↓
Route
   ↓
Network security
   ↓
TCP connection
   ↓
SQL protocol
   ↓
Authentication
```

Each layer can fail independently.

---

# 13. Layer 1 — DNS failure

Example:

```text
nslookup sqlmi-prod.company.internal
```

returns:

```text
Server can't find ...
```

Then:

```text
Application ❌
```

but:

```text
SQL MI ✓
```

Your first problem is DNS.

---

# 14. Layer 2 — routing failure

DNS works:

```text
Hostname → IP
```

but traffic cannot reach the destination.

Possible causes include:

```text
Missing route
Incorrect route table
Network topology problem
Peering issue
VPN problem
ExpressRoute issue
```

---

# 15. Layer 3 — network security

Suppose routing works but traffic is blocked.

Possible causes:

```text
NSG
Firewall
Network appliance
Security policy
```

The application may see:

```text
Connection timeout
```

---

# 16. Layer 4 — TCP

Now the application tries to establish a TCP connection.

If TCP can't connect:

```text
Application
   |
   X
SQL MI
```

Typical symptom:

```text
Connection timeout
```

or:

```text
Network-related error
```

---

# 17. Layer 5 — SQL authentication

Suppose TCP works.

Now SQL receives the connection.

Then authentication occurs.

Example:

```text
Application
    ↓
TCP ✓
    ↓
SQL authentication ❌
```

Now you might get:

```text
Login failed
```

This is very different from:

```text
Connection timeout
```

---

# 18. Layer 6 — authorization

Suppose authentication succeeds:

```text
Login ✓
```

but the user doesn't have permission:

```text
Database access ❌
```

Example:

```text
SELECT permission denied
```

So:

```text
Network works
Authentication works
Authorization fails
```

---

# 19. Memorize this troubleshooting ladder

```text
1. DNS
   ↓
2. Routing
   ↓
3. Network security
   ↓
4. TCP
   ↓
5. SQL endpoint
   ↓
6. Authentication
   ↓
7. Authorization
   ↓
8. Application
```

This is your core troubleshooting sequence.

---

# 20. Real production scenario

Application team calls:

> "SQL Server is down!"

You check SQL MI:

```text
Status = Healthy
```

Don't conclude:

> "Application team is wrong."

Instead:

```text
Application
    ↓
DNS?
    ↓
Route?
    ↓
TCP?
    ↓
Authentication?
```

---

# 21. Step 1 — DNS test

From a machine that should connect:

```powershell
nslookup sqlmi-prod.example.internal
```

or appropriate DNS tools.

You want:

```text
Hostname
   ↓
Resolvable address
```

If DNS fails:

```text
Stop here.
```

Fix DNS before troubleshooting SQL authentication.

---

# 22. Step 2 — TCP test

From a suitable client host, you can use:

```powershell
Test-NetConnection <sql-hostname> -Port <port>
```

For example, if your environment uses TCP port 1433:

```powershell
Test-NetConnection sqlmi-prod.example.internal -Port 1433
```

The result helps determine whether the TCP connection can be established.

---

# 23. Important: ping isn't enough

People often do:

```text
ping sqlmi-prod
```

and say:

> "Ping works, so SQL works."

Wrong.

Ping tests ICMP.

SQL connectivity uses TCP.

You need to test the relevant TCP port.

---

# 24. TCP success

Suppose:

```text
Test-NetConnection
TcpTestSucceeded : True
```

Now you know:

```text
DNS likely works
TCP path works
```

You move upward:

```text
Authentication
```

---

# 25. TCP failure

Suppose:

```text
TcpTestSucceeded : False
```

Now don't waste time changing SQL logins.

Investigate:

```text
Routing
NSG
Firewall
Network appliance
VPN
ExpressRoute
Subnet
DNS
```

---

# 26. Login failure

Suppose:

```text
TCP ✓
```

but:

```text
Login failed for user...
```

Now investigate:

```text
Authentication method
Username
Password/token
Microsoft Entra configuration
SQL login
User mapping
```

Not NSGs.

---

# 27. Permission failure

Suppose login succeeds:

```text
Authentication ✓
```

but:

```text
SELECT permission denied
```

Now investigate:

```text
User
Database role
Schema permissions
Object permissions
```

Again:

```text
Not a network problem.
```

---

# 28. Connection timeout vs login failure

This is an important interview distinction.

### Timeout

Usually points toward:

```text
Network
DNS
Routing
Firewall
Port
Endpoint
```

### Login failed

Usually points toward:

```text
Authentication
Credentials
Identity
Login configuration
```

### Permission denied

Usually points toward:

```text
Authorization
```

---

# 29. On-premises → Azure SQL MI

Now let's make the scenario more realistic.

Company:

```text
On-premises Data Center
       |
       |
       ↓
Azure
       |
    SQL MI
```

The application is still on-premises.

How does it reach SQL MI?

You need network connectivity between on-premises and Azure.

Common approaches include:

```text
VPN
ExpressRoute
```

---

# 30. Site-to-site VPN

Conceptually:

```text
On-prem
   |
VPN Gateway
   |
Internet
   |
Azure VPN Gateway
   |
VNet
   |
SQL MI
```

Traffic travels through an encrypted VPN connection.

This is commonly used for hybrid connectivity.

---

# 31. ExpressRoute

ExpressRoute provides private connectivity between your on-premises environment and Microsoft cloud through supported connectivity providers.

Conceptually:

```text
Corporate Network
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

It's commonly used for enterprise hybrid architectures requiring private connectivity and predictable networking characteristics.

---

# 32. VPN vs ExpressRoute

Very simplified:

| Feature                   | VPN              | ExpressRoute                          |
| ------------------------- | ---------------- | ------------------------------------- |
| Connectivity              | Encrypted tunnel | Private connectivity                  |
| Uses public internet path | Generally yes    | No                                    |
| Setup                     | Usually simpler  | More involved                         |
| Enterprise use            | Common           | Common for larger hybrid environments |
| Cost                      | Typically lower  | Typically higher                      |

The exact performance, availability, and cost depend on the architecture.

---

# 33. VNet peering

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

You can connect VNets using supported Azure networking mechanisms such as VNet peering.

Conceptually:

```text
VNet A
 App
  |
  +==========+
             |
             |
          VNet B
             |
            SQL MI
```

Now the application can potentially reach the SQL MI network through the connected topology, assuming routing and security are correctly configured.

---

# 34. Hub-and-spoke architecture

Large organizations often use:

```text
                  Hub VNet
                     |
          +----------+----------+
          |                     |
       Spoke A                Spoke B
       App                     Data
                               |
                              SQL MI
```

The hub might contain shared:

```text
Firewall
DNS
VPN Gateway
ExpressRoute Gateway
Security services
```

Spokes contain workloads.

This is common in enterprise Azure designs.

---

# 35. DNS in hub-and-spoke

Here's where things get interesting.

Suppose:

```text
Application
   |
   ↓
sqlmi-prod.company.internal
```

The application needs to resolve the database name correctly.

Your DNS architecture might include:

```text
Corporate DNS
      |
Azure DNS / Private DNS
      |
SQL MI endpoint
```

If DNS forwarding is incorrectly configured:

```text
Application
   |
   X
DNS
```

SQL may be healthy while the application is broken.

---

# 36. Private DNS

Private DNS allows internal name resolution within private networking architectures.

Conceptually:

```text
Private DNS
     |
     +---- SQL endpoint
     |
     +---- Internal services
```

This helps applications use names rather than hard-coded IP addresses.

---

# 37. Why don't we hard-code SQL IPs?

Suppose:

```text
SQL MI
IP = 10.10.2.15
```

Application:

```text
Server=10.10.2.15
```

Later infrastructure changes.

Now:

```text
10.10.2.15
```

may no longer represent the correct endpoint.

Your application breaks.

Use the supported DNS/service endpoint instead.

---

# 38. NSG

**Network Security Groups (NSGs)** provide network traffic filtering at Azure network interfaces/subnets according to supported rules.

Conceptually:

```text
Application
    |
    ↓
NSG
    |
    ↓
SQL MI subnet
```

You can control allowed/denied traffic according to source, destination, protocol, and port within the supported networking model.

---

# 39. Example NSG logic

Suppose application subnet is:

```text
10.10.1.0/24
```

SQL MI subnet:

```text
10.10.2.0/24
```

You may have a rule allowing required SQL traffic:

```text
Source:
10.10.1.0/24

Destination:
10.10.2.0/24

Protocol:
TCP

Port:
required SQL port
```

The exact SQL MI networking requirements include additional platform-specific rules, so don't create an NSG from a simplistic example and assume it's production-ready.

---

# 40. Critical warning

With managed services, **you cannot treat network security rules like arbitrary VM rules**.

SQL MI has documented networking requirements for:

```text
Azure platform communication
Management
High availability
Health
Service functionality
```

If you block required traffic:

```text
SQL MI
   ↓
Unexpected behavior
```

Always follow the current Microsoft SQL MI networking requirements when modifying NSGs, route tables, and related controls.

---

# 41. User-defined routes

Enterprise networks may use custom routing.

Conceptually:

```text
Application
    ↓
Route table
    ↓
Firewall/NVA
    ↓
SQL MI
```

This can introduce another failure point.

---

# 42. Network Virtual Appliance

Some enterprises force traffic through a firewall/NVA.

Example:

```text
Application
    |
    ↓
Azure Firewall / NVA
    |
    ↓
SQL MI
```

If the firewall doesn't allow the required traffic:

```text
Application
    ↓
Firewall ❌
    ↓
SQL MI
```

SQL itself can be completely healthy.

---

# 43. Real troubleshooting scenario

Application team says:

> "Production application cannot connect to SQL MI."

You ask:

### Question 1

Can the hostname resolve?

```text
nslookup
```

### Question 2

Can the host reach the required TCP port?

```text
Test-NetConnection
```

### Question 3

Does the route exist?

### Question 4

Is traffic blocked by NSG/firewall/NVA?

### Question 5

Does SQL receive the connection?

### Question 6

Is authentication failing?

### Question 7

Does the database user have permissions?

This is systematic troubleshooting.

---

# 44. Scenario — DNS failure

Application:

```text
Server=sqlmi-prod.internal
```

Error:

```text
Cannot resolve host
```

You run:

```powershell
nslookup sqlmi-prod.internal
```

Result:

```text
NXDOMAIN
```

Diagnosis:

```text
DNS problem
```

Don't:

```text
Reset SQL password
Restart SQL
Create login
```

Those are irrelevant.

---

# 45. Scenario — DNS works, TCP fails

```text
nslookup ✓
```

but:

```text
Test-NetConnection ... -Port <required-port>
❌
```

Now investigate:

```text
Routing
NSG
Firewall
NVA
VPN
ExpressRoute
Subnet
```

---

# 46. Scenario — TCP works, login fails

```text
DNS ✓
TCP ✓
SQL endpoint ✓
```

but:

```text
Login failed
```

Investigate:

```text
SQL authentication
Entra authentication
Credential
Token
Login/user
Database mapping
```

---

# 47. Scenario — login works, SELECT fails

```text
DNS ✓
TCP ✓
Authentication ✓
```

but:

```text
SELECT permission denied
```

Now:

```text
Check user
Check role
Check schema
Check object permissions
```

---

# 48. Scenario — only one application fails

Suppose:

```text
App A → SQL MI ✓
App B → SQL MI ✓
App C → SQL MI ❌
```

SQL MI is probably not globally unavailable.

Focus on:

```text
App C
DNS
Identity
Connection string
Network path
Firewall rules
```

---

# 49. Scenario — every application fails

Suppose:

```text
App A ❌
App B ❌
App C ❌
```

Now investigate broader infrastructure:

```text
SQL MI status
VNet
DNS
Network routing
Firewall
NSG
Regional Azure issue
```

This is why comparing multiple clients is useful.

---

# 50. SQL port

Traditional SQL Server commonly uses:

```text
TCP 1433
```

SQL Managed Instance supports SQL connectivity through its documented endpoints and ports.

Don't blindly assume every SQL MI connection scenario is identical to a VM-hosted SQL Server.

Always verify the current SQL MI connection requirements for the specific endpoint you're using.

---

# 51. Connection string

Example conceptually:

```text
Server=tcp:<managed-instance-endpoint>,<port>;
Database=SalesDB;
...
```

The application should use the supported SQL MI endpoint rather than a hard-coded internal IP.

---

# 52. Application connection pooling

Suppose SQL MI is healthy but:

```text
Application
   ↓
Connection pool exhausted
```

Users see:

```text
Timeout
```

This may look like SQL connectivity.

But the real issue is:

```text
Application connection pool
```

Again:

> Don't assume every "SQL connection error" is a SQL Server problem.

---

# 53. Timeout types

A DBA should distinguish:

### DNS timeout

```text
Cannot resolve server name
```

### TCP connection timeout

```text
Cannot establish connection
```

### SQL login timeout

```text
Server reachable, but connection/login negotiation isn't completing
```

### Query timeout

```text
Connection established
Query is taking too long
```

These are completely different problems.

---

# 54. Query timeout ≠ network failure

Example:

```text
Application
    ↓
SQL MI ✓
    ↓
SELECT ...
    ↓
Query running 10 minutes
    ↓
Application timeout
```

The network worked.

The query may be suffering from:

```text
Blocking
Poor execution plan
Missing index
CPU
IO
Locking
```

Don't troubleshoot DNS for a query timeout.

---

# 55. SQL MI + on-prem application

Let's build a real enterprise architecture:

```text
                   Corporate DC
              +-------------------+
              | Application       |
              | SQL tools         |
              +---------+---------+
                        |
                  ExpressRoute
                        |
                        ↓
                  Azure Hub VNet
                        |
              +---------+---------+
              |                   |
           Firewall             DNS
              |                   |
              +---------+---------+
                        |
                    Spoke VNet
                        |
                     SQL MI
```

This is a realistic hybrid design.

---

# 56. Troubleshooting this architecture

If the app cannot connect:

```text
1. Corporate DNS
2. DNS forwarding
3. ExpressRoute
4. Azure routing
5. Firewall
6. NSG
7. SQL endpoint
8. TCP
9. Authentication
10. Authorization
```

That's how a senior DBA approaches it.

---

# 57. VPN outage scenario

Suppose:

```text
SQL MI ✓
Azure App ✓
On-prem App ❌
```

What changed?

Maybe:

```text
VPN tunnel ❌
```

Azure SQL itself doesn't need troubleshooting.

Your hybrid network path does.

---

# 58. ExpressRoute scenario

Suppose all on-prem applications suddenly lose access:

```text
App1 ❌
App2 ❌
App3 ❌
```

But:

```text
Azure App → SQL MI ✓
```

This strongly suggests investigating:

```text
ExpressRoute
On-prem routing
Firewall
DNS
```

rather than SQL MI itself.

---

# 59. Monitoring network health

A production environment should monitor:

```text
Connectivity
Latency
Packet loss where applicable
DNS
VPN
ExpressRoute
Firewall
SQL connectivity
```

You don't want users to report the outage before your monitoring system does.

---

# 60. Latency matters

Imagine:

```text
Application
   |
   | 100 ms
   |
SQL MI
```

A single query might be fine.

But imagine:

```text
Application
   |
   ↓
100 ms
   |
SQL MI
```

and the application makes:

```text
100 database round trips
```

Network latency can become a major performance factor.

---

# 61. Chatty applications

Bad architecture:

```text
App
 ↓
SQL query
 ↓
App
 ↓
SQL query
 ↓
App
 ↓
SQL query
...
```

Hundreds of round trips amplify network latency.

Better:

```text
App
   ↓
Fewer, more efficient calls
   ↓
SQL
```

This is where SQL performance and network architecture intersect.

---

# 62. Connection pooling

Connection pooling reduces the overhead of repeatedly establishing database connections.

Conceptually:

```text
Application
   |
Connection Pool
   |
   +-- Connection 1
   +-- Connection 2
   +-- Connection 3
   +-- Connection 4
             |
             ↓
           SQL MI
```

This can significantly improve application efficiency.

---

# 63. Don't confuse connection pooling with SQL connection limits

If an application creates excessive concurrent connections:

```text
Application
   ↓
10,000 connections
   ↓
SQL MI
```

you may see resource pressure.

The right response may involve:

```text
Application pool configuration
Connection management
Workload tuning
SQL resource analysis
```

not just increasing network rules.

---

# 64. Network security and least privilege

You want both:

```text
Network restriction
+
SQL authorization
```

For example:

```text
Only App subnet
      ↓
Can reach SQL MI
      ↓
Application identity
      ↓
Can only read/write required objects
```

If the application credential is compromised, the attacker has fewer options.

---

# 65. Interview question

### "Application cannot connect to SQL MI. What do you check?"

Strong answer:

> "I troubleshoot layer by layer. First I verify DNS resolution, then routing and network security controls, then TCP connectivity to the required SQL endpoint/port. If TCP succeeds, I move to SQL authentication and finally database authorization. I also compare connectivity from other clients to determine whether the issue is application-specific or infrastructure-wide."

That's a very strong DBA answer.

---

# 66. Interview question

### "How do you test SQL connectivity from Windows?"

A common first-level TCP test is:

```powershell
Test-NetConnection <hostname> -Port <port>
```

And DNS can be checked with:

```powershell
nslookup <hostname>
```

These don't prove the entire SQL authentication process works, but they help isolate network-layer problems.

---

# 67. Interview question

### "Ping works, but SQL connection fails. Why?"

Answer:

> "Ping uses ICMP, while SQL connectivity uses TCP. Successful ping doesn't prove that DNS, routing, the required TCP port, network security rules, or SQL authentication are working."

---

# 68. Interview question

### "DNS works but TCP fails. What does that tell you?"

Answer:

> "The hostname is resolving, so I would move to routing and network controls such as NSGs, firewalls, network appliances, VPN/ExpressRoute, and the SQL endpoint/port."

---

# 69. Interview question

### "TCP works but login fails."

Answer:

> "The basic network path is working. I'd investigate authentication—SQL credentials, Microsoft Entra authentication, token/identity configuration, login state, and database user mapping."

---

# 70. Interview question

### "Login works but SELECT fails."

Answer:

> "That's an authorization issue. I'd inspect the database user, roles, schema permissions, and object-level permissions."

---

# 71. The golden troubleshooting tree

Memorize this:

```text
              Application cannot connect
                        |
                        ↓
                 DNS resolution?
                  /           \
                NO             YES
                |               |
             DNS fix       TCP connection?
                              /      \
                            NO        YES
                            |          |
                     Network path    Login?
                     /    |    \      /   \
                   NSG Firewall Route NO   YES
                                        |     |
                                    Auth fix  Permission?
                                              /      \
                                            NO        YES
                                            |          |
                                       Auth issue   Authorization
```

This one diagram can save you hours.

---

# 72. Production incident walkthrough

Let's say at 10:00 AM:

```text
Application users report:
"Database unavailable."
```

You check:

```text
SQL MI = Healthy
```

Don't restart anything.

### Step 1

From application server:

```powershell
nslookup sqlmi-prod.example.internal
```

Result:

```text
Success
```

### Step 2

```powershell
Test-NetConnection sqlmi-prod.example.internal -Port <required-port>
```

Result:

```text
TcpTestSucceeded = False
```

Now you know:

```text
Not an SQL login problem.
```

---

# 73. Continue investigation

Check:

```text
Route
 ↓
Firewall
 ↓
NSG
 ↓
VPN/ExpressRoute
 ↓
NVA
```

Suppose you discover:

```text
Firewall rule changed at 09:58
```

Problem identified.

No SQL restart needed.

---

# 74. Another incident

Application says:

```text
Login failed for user 'appuser'
```

You test:

```text
DNS ✓
TCP ✓
```

Then inspect:

```text
Credential
Login
Entra identity
User mapping
```

You discover the password was rotated but the application still has the old credential.

Again:

```text
SQL networking = healthy
Authentication configuration = problem
```

---

# 75. Another incident

Application reports:

```text
Execution Timeout Expired
```

You verify:

```text
DNS ✓
TCP ✓
Login ✓
```

Now investigate SQL:

```text
Blocking
CPU
IO
Query plan
Indexes
Locks
Wait statistics
```

The network isn't necessarily the problem.

---

# 76. Senior DBA mindset

When someone says:

> "SQL is down."

Your response shouldn't be:

> "Let me restart SQL."

Instead:

```text
What exactly is failing?
```

Then isolate:

```text
DNS
→ Network
→ TCP
→ Authentication
→ Authorization
→ Query
→ Application
```

That's the professional approach.

---

# 77. Your networking checklist

```text
VNET
☐ VNet architecture
☐ Address space
☐ Subnets
☐ SQL MI dedicated subnet

CONNECTIVITY
☐ Application → SQL MI
☐ On-prem → Azure
☐ VPN
☐ ExpressRoute
☐ VNet peering

DNS
☐ Name resolution
☐ Private DNS
☐ DNS forwarding
☐ Correct endpoint

NETWORK SECURITY
☐ NSG
☐ Firewall
☐ Route tables
☐ NVA
☐ Required ports

TROUBLESHOOTING
☐ nslookup
☐ Test-NetConnection
☐ Route analysis
☐ Authentication
☐ Authorization

PERFORMANCE
☐ Latency
☐ Connection pooling
☐ Chatty applications
```

---

# 78. Final mental model

When troubleshooting Azure SQL MI connectivity, remember:

```text
                CONNECTION
                    |
                    ↓
                  DNS
                    |
                    ↓
                 ROUTE
                    |
                    ↓
              NETWORK SECURITY
                    |
                    ↓
                   TCP
                    |
                    ↓
              SQL ENDPOINT
                    |
                    ↓
             AUTHENTICATION
                    |
                    ↓
             AUTHORIZATION
                    |
                    ↓
                  QUERY
                    |
                    ↓
               APPLICATION
```

**Don't skip layers.**

---

## Next — Lesson 85: Azure SQL Managed Instance Monitoring & Performance

We'll start the **performance/monitoring section**, including:

* CPU
* Memory
* Storage
* IOPS
* Data IO
* Log IO
* Wait statistics
* Blocking
* Deadlocks
* Query Store
* Azure Monitor
* Log Analytics
* Alerts
* Slow queries
* High CPU incidents
* High log usage
* Storage pressure
* Real-time production troubleshooting
* and a full scenario:

> **"Users say the application became slow at 2:30 PM. SQL MI is healthy. Walk me through exactly how you identify the root cause."**
