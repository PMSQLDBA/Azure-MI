**Lesson 10 — Azure SQL Managed Instance Networking**. 

This is one of the most important topics for real-world MI work because many "SQL problems" are actually **network problems**.

# Lesson 10 — Azure SQL Managed Instance Networking

Imagine this production complaint:

> **"SQL Managed Instance is healthy, but the application cannot connect."**

A SQL DBA who only checks the database may get stuck.

A cloud DBA thinks:

```text
Application
    ↓
DNS
    ↓
Network
    ↓
Firewall / NSG
    ↓
Routing
    ↓
MI endpoint
    ↓
SQL authentication
    ↓
Database
```

We need to troubleshoot **every layer**.

---

# 1. Why networking is different with MI

Traditional SQL Server might look like:

```text
Application
    |
    v
Windows Server
    |
    v
SQL Server
```

Managed Instance is a PaaS service integrated into an Azure virtual network.

Conceptually:

```text
                    Azure
                      |
                    VNet
                      |
                   Subnet
                      |
             SQL Managed Instance
                      |
                  Database
```

This is why networking is fundamental to MI.

---

# 2. The VNet

A **Virtual Network (VNet)** is your private network boundary in Azure.

Think of it like:

```text
Your corporate network
        ≈
Azure VNet
```

Inside the VNet you can have:

```text
VNet
│
├── Application subnet
├── Integration subnet
├── Other resources
└── MI subnet
```

---

# 3. MI subnet

Managed Instance is deployed into a dedicated subnet.

Conceptually:

```text
VNet: ABC-Production-VNet

10.20.0.0/16
        |
        +----------------------+
        |                      |
        v                      v
App subnet                 MI subnet
10.20.1.0/24              10.20.2.0/24
                                |
                                v
                           SQL MI
```

The MI subnet has specific requirements and configuration.

This is not the same as putting a normal VM anywhere you want.

---

# 4. Subnet delegation

The subnet used for Managed Instance needs the appropriate Azure service delegation.

Conceptually:

```text
MI subnet
    |
    v
Delegated to
Microsoft.Sql/managedInstances
```

Why?

Because Azure needs to manage the network resources required by the Managed Instance service.

If the subnet isn't configured correctly, deployment and connectivity can fail.

---

# 5. Network Security Groups — NSG

Now imagine:

```text
Application
    |
    v
MI subnet
```

An **NSG** can control network traffic.

Conceptually:

```text
Application
    |
    | TCP traffic
    v
+----------------+
|      NSG       |
|                |
| Allow?         |
| Deny?          |
+----------------+
        |
        v
      MI
```

If the NSG blocks required traffic:

```text
Application
    |
    X
   NSG
```

The database may be completely healthy.

---

# 6. Important troubleshooting lesson

Suppose:

```text
Azure Portal:
MI = Healthy
```

But:

```text
Application:
Connection timeout
```

Don't conclude:

> "SQL Server is broken."

Check:

```text
DNS
NSG
Route
Firewall
Endpoint
Port
Authentication
```

---

# 7. DNS

This is a huge one.

Application might use:

```text
mi-prod.company.internal
```

The application tries:

```text
DNS lookup
      ↓
?
```

If DNS doesn't resolve:

```text
Application
    |
    v
DNS
    |
    X
No address
```

The application can't even reach the MI endpoint.

---

# 8. DNS scenario

Developer says:

> "I can connect using the IP address, but the application hostname doesn't work."

That immediately suggests:

```text
DNS
```

Investigate:

```text
DNS resolution
Private DNS zones
Conditional forwarding
DNS servers
Hostname configuration
```

depending on your architecture.

---

# 9. DNS is often misunderstood

A connection problem can happen before SQL authentication even begins.

Think:

```text
Step 1
Can hostname resolve?
       ↓
Step 2
Can network route?
       ↓
Step 3
Can TCP connection establish?
       ↓
Step 4
Can TLS/authentication succeed?
       ↓
Step 5
Can SQL login?
       ↓
Step 6
Can database be accessed?
```

If Step 1 fails, don't waste time investigating SQL permissions.

---

# 10. TCP connectivity

Suppose DNS works.

Now:

```text
Application
    |
    v
MI hostname
    |
    v
Network
    |
    v
TCP
```

You need to establish a TCP connection to the appropriate SQL endpoint/port.

A timeout can indicate:

```text
NSG
Firewall
Routing
DNS
Network appliance
Endpoint issue
```

An authentication error is different.

---

# 11. Timeout vs authentication error

This distinction is extremely useful.

### Timeout

Example:

```text
Connection timed out
```

Think:

```text
DNS
Network
NSG
Firewall
Routing
Endpoint
```

### Login failed

Example:

```text
Login failed for user
```

Network connectivity probably exists.

Now think:

```text
Username
Password
Authentication type
Login
User
Permissions
```

This distinction can save hours.

---

# 12. Network path

Imagine:

```text
On-premises Application
          |
          v
      Corporate
       Network
          |
          v
       VPN/ExpressRoute
          |
          v
      Azure VNet
          |
          v
       MI subnet
          |
          v
          MI
```

Every section can introduce a problem.

---

# 13. VPN vs ExpressRoute

For hybrid environments:

```text
On-premises
     |
     +---- VPN ---- Azure VNet
     |
     +---- ExpressRoute ---- Azure VNet
```

### VPN

Encrypted connectivity over an existing network path.

### ExpressRoute

Private connectivity through an ExpressRoute circuit/provider architecture.

For production enterprise environments, the choice depends on:

```text
Security
Latency
Reliability
Bandwidth
Cost
Business requirements
```

---

# 14. Example: on-premises application cannot connect

Architecture:

```text
On-prem SQL client
       |
       v
Corporate DNS
       |
       v
VPN
       |
       v
Azure VNet
       |
       v
MI
```

Application gets:

```text
Connection timeout
```

Your troubleshooting order could be:

```text
1. DNS resolution
2. Network route
3. VPN/ExpressRoute health
4. NSG
5. Firewall/network appliance
6. TCP connectivity
7. SQL authentication
```

---

# 15. NSG vs SQL authentication

Don't confuse them.

### NSG

Controls network traffic.

```text
"Can this network connection get through?"
```

### SQL authentication

Controls database access.

```text
"Who are you?"
```

### SQL permissions

Controls authorization.

```text
"What are you allowed to do?"
```

So:

```text
Network
   ↓
Authentication
   ↓
Authorization
```

---

# 16. Authentication example

Suppose the application connects successfully but gets:

```text
Login failed for user 'appuser'
```

That means you should investigate:

```text
Is login present?
Password correct?
Authentication method correct?
Login enabled?
Database user mapped?
Permissions correct?
```

Don't start modifying NSGs.

The network already worked well enough to reach SQL authentication.

---

# 17. Azure AD / Microsoft Entra authentication

Modern Azure architectures often use **Microsoft Entra ID** authentication.

Conceptually:

```text
Application
    |
    v
Microsoft Entra ID
    |
    v
Token
    |
    v
SQL Managed Instance
```

Instead of:

```text
Application
    |
    v
SQL username/password
```

This can improve centralized identity management.

But it introduces additional dependencies:

```text
Identity configuration
Permissions
Token acquisition
Network connectivity
Managed identity/service principal
```

---

# 18. Application architecture

A production architecture could look like:

```text
                 Users
                   |
                   v
              Web Application
                   |
                   v
              App Subnet
                   |
                   v
             Azure VNet
                   |
                   v
              MI Subnet
                   |
                   v
          SQL Managed Instance
```

And for on-premises:

```text
On-Premises
    |
    v
VPN / ExpressRoute
    |
    v
Azure VNet
    |
    v
MI subnet
    |
    v
SQL MI
```

---

# 19. Network Security Group troubleshooting

Suppose:

```text
Application = 10.20.1.50
MI = 10.20.2.x
```

Traffic:

```text
10.20.1.50
    |
    | TCP
    v
NSG
    |
    X
Blocked
```

Application:

```text
Connection timeout
```

MI:

```text
Healthy
```

The DBA might waste time checking:

```text
Indexes
CPU
Query Store
```

when the actual problem is:

```text
NSG rule
```

---

# 20. Routing

Even if NSG allows traffic, routing can fail.

Example:

```text
Application
   |
   v
Route table
   |
   X
Wrong route
```

Or traffic may be sent through:

```text
Azure Firewall
Network Virtual Appliance
Proxy
```

and that component may block or alter traffic.

So you need to understand the full network path.

---

# 21. The classic production scenario

ABC Retail:

```text
Application
  |
  v
Azure VNet
  |
  v
SQL MI
```

Everything worked yesterday.

Today:

```text
Application → MI = timeout
```

You check:

```text
MI health = Healthy
CPU = 20%
Storage = Healthy
```

You then test:

```text
DNS = resolves
TCP = fails
```

Now you've narrowed the problem.

Potential causes:

```text
NSG
Firewall
Routing
Network appliance
Port
```

That's a much better diagnosis than:

> "SQL MI is down."

---

# 22. Another scenario: DNS failure

Application:

```text
Server=mi-prod.company.com
```

DNS:

```text
mi-prod.company.com
     ↓
NXDOMAIN
```

Application:

```text
Cannot resolve host
```

MI itself:

```text
Healthy
```

Root cause:

```text
DNS configuration
```

Again:

> **Database healthy ≠ application connectivity healthy.**

---

# 23. Network troubleshooting flow

Memorize this:

```text
Application cannot connect
          |
          v
      DNS works?
       /     \
     NO       YES
     |          |
  Fix DNS      TCP works?
                /    \
              NO      YES
              |        |
        Network path   Authentication
        / NSG / FW       |
                          v
                     Authorization
```

This is a very useful real-world flow.

---

# 24. Don't troubleshoot randomly

A weak troubleshooting approach:

```text
Check CPU
Restart app
Change password
Change NSG
Restart MI
Change query
```

That's guesswork.

A strong approach:

```text
Application error
      ↓
Classify error
      ↓
DNS?
      ↓
TCP?
      ↓
Authentication?
      ↓
Authorization?
      ↓
Database?
      ↓
Query?
```

Each step eliminates entire categories of problems.

---

# 25. Interview question

### "The Managed Instance is healthy, but the application can't connect. What do you check?"

A strong answer:

> "I'd first classify the connection failure. If it's a timeout, I'd check DNS resolution, network routing, NSGs, firewalls, private connectivity and endpoint reachability. If TCP connectivity succeeds but authentication fails, I'd investigate the login/authentication configuration and database user mapping. If the connection succeeds but queries fail or are slow, I'd move into SQL-level troubleshooting."

That's an excellent cloud DBA answer.

---

# 26. Full MI architecture

Now combine everything we've learned:

```text
                         INTERNET / USERS
                                |
                                v
                         APPLICATION
                                |
                         +------+------+
                         |             |
                         v             v
                       DNS       Authentication
                         |             |
                         +------+------+
                                |
                                v
                              VNet
                                |
                    +-----------+-----------+
                    |                       |
                    v                       v
              Application subnet       MI subnet
                                            |
                                            v
                                  SQL Managed Instance
                                            |
                     +----------------------+----------------+
                     |                      |                |
                     v                      v                v
                    HA                   Backup             DR
                     |                      |                |
                     v                      v                v
                Availability          Data recovery      Region recovery
```

That's the architecture you should have in your head.

---

# Lesson 10 challenge 🧑‍💻

Production incident:

> **ABC Retail's application suddenly cannot connect to SQL Managed Instance.**

You discover:

```text
MI health:              Healthy
DNS resolution:         SUCCESS
TCP connectivity:       FAIL
Application error:      Connection timeout
CPU:                    30%
Storage:                Healthy
```

Answer these:

**1. Would you investigate SQL queries first? Why or why not?**

**2. What are your next 3–5 areas of investigation?**

**3. Why does the fact that DNS succeeds matter?**

**4. Why does a timeout point you toward networking rather than SQL authentication?**

**5. If TCP connectivity starts working but you then receive `Login failed for user`, what layer has the problem moved to?**
