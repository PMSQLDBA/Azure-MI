# Lesson 67 — Azure SQL Managed Instance Networking & Connectivity

This is one of the most important areas for an Azure SQL DBA because a database can be perfectly healthy while **applications cannot connect to it**.

The key principle:

> **"SQL is healthy" does not mean "the application can reach SQL."**

---

# 1. Real production incident

At 9:15 AM:

```text id="r7x2k1"
Application
    ↓
Cannot connect to SQL MI
```

Developer says:

> "SQL Managed Instance is down."

You check Azure:

```text id="m4q8v2"
SQL MI status = Healthy
CPU           = 30%
Storage       = Normal
Databases     = Online
```

So where is the problem?

You need to troubleshoot the connection path **layer by layer**.

---

# 2. The connection path

Think about the journey:

```text id="a9p3c7"
Application
     |
     v
DNS
     |
     v
Network routing
     |
     v
VNet
     |
     v
Subnet
     |
     v
Network security controls
     |
     v
SQL MI endpoint
     |
     v
Authentication
     |
     v
SQL database
```

A failure anywhere can look like:

```text id="x2m8n4"
"SQL connection failed"
```

That's why the error message alone isn't enough.

---

# 3. Azure VNet

A Virtual Network is the logical network boundary for Azure resources.

Conceptually:

```text id="j5r9w3"
Azure VNet
+------------------------------+
|                              |
| Application                  |
|      |                       |
|      | network               |
|      v                       |
| SQL Managed Instance         |
|                              |
+------------------------------+
```

The exact architecture depends on the service and networking design, but the key idea is:

> **SQL MI networking is tightly integrated with Azure virtual networking.**

---

# 4. Subnet

Resources are placed into subnets.

Conceptually:

```text id="v8k3q1"
VNet
 |
 +-- App subnet
 |
 +-- SQL MI subnet
 |
 +-- Other subnets
```

For SQL MI, subnet configuration and delegated networking requirements are important.

You don't treat the MI subnet like an arbitrary application subnet.

---

# 5. Why subnet design matters

Imagine your organization deploys:

```text id="z3x7m2"
Application
    ↓
SQL MI
```

Then someone changes networking configuration in the MI subnet.

Suddenly:

```text id="b6q1r9"
Application → SQL MI
             X
```

The database didn't change.

The network path did.

This is why DBAs working in Azure need enough networking knowledge to troubleshoot connectivity.

---

# 6. NSG — Network Security Group

An NSG controls network traffic according to rules.

Conceptually:

```text id="q4n8m1"
Application
    |
    | traffic
    v
   NSG
    |
    +---- ALLOW
    |
    +---- DENY
    |
    v
SQL MI
```

If the traffic is denied:

```text id="s8k2v5"
Application
      |
      X
     NSG
```

The application can't connect.

---

# 7. Example

Suppose your SQL MI connection uses the expected SQL port.

Application:

```text id="w7p3n6"
10.20.1.10
```

SQL MI:

```text id="c5m8q2"
10.20.5.10
```

Security rule accidentally changes from:

```text id="g1r4x9"
ALLOW
10.20.1.0/24
→ SQL subnet
```

to:

```text id="j6n2p8"
DENY
```

Result:

```text id="m9q3w1"
Application
   ↓
Network
   ↓
DENIED
```

SQL itself may be completely healthy.

---

# 8. DNS

This is another extremely common problem.

The application doesn't normally connect by saying:

```text
10.20.5.10
```

It may use a hostname.

For example:

```text id="y5v8r2"
my-sql-mi.company.internal
```

The application needs DNS to resolve:

```text id="h3k7m1"
Hostname
   ↓
IP address
```

If DNS is wrong:

```text id="p9x2c6"
Application
   ↓
DNS lookup
   X
```

No SQL connection occurs.

---

# 9. Classic DNS incident

Developer:

> "Ping works to the server!"

You ask:

> "What hostname does the application use?"

They say:

```text id="z8m4q1"
sql-prod.company.com
```

You test DNS.

It resolves to:

```text id="r5n7v3"
OLD SQL SERVER
```

instead of the new MI endpoint.

The database is healthy.

The application is simply going to the wrong destination.

---

# 10. DNS troubleshooting

From the application environment, verify:

```text id="k2w6p9"
nslookup <hostname>
```

or appropriate DNS tools.

You want:

```text id="c8m3x7"
Application hostname
       ↓
Expected DNS resolution
       ↓
Expected SQL endpoint
```

Don't test DNS from your laptop and assume the application sees the same DNS environment.

This is critical.

---

# 11. "Works from my laptop"

One of the most dangerous statements in production troubleshooting:

> **"It works from my laptop."**

Your laptop may have:

```text id="m7q1v5"
Different DNS
Different network
VPN
Different firewall rules
Different identity
Different route
```

The application server may have a completely different path.

Always test **from the source environment that is failing**.

---

# 12. Port connectivity

After DNS:

```text id="v4n8p2"
Can the application reach the endpoint?
```

Then:

```text id="f6x3m9"
Can it reach the required TCP port?
```

Conceptually:

```text id="s2k7q4"
DNS works
    ↓
Routing works
    ↓
Port reachable?
    ↓
SQL handshake
```

A port-level test helps distinguish:

```text id="n9c1x5"
Network problem
```

from:

```text id="b8m4q2"
SQL authentication problem
```

---

# 13. Connection timeout vs login failure

This distinction is extremely useful.

### Timeout

Something like:

```text id="a3v8k1"
Connection timed out
```

Often suggests investigating:

```text id="q6m2r9"
DNS
Routing
Firewall
NSG
Port
Network path
```

### Login failed

Something like:

```text id="w5p7n3"
Login failed for user
```

means you likely reached SQL successfully.

Now investigate:

```text id="j9k4x2"
Authentication
Identity
Password/token
SQL user
Permissions
```

---

# 14. Your troubleshooting tree

Memorize this:

```text id="c1r8m4"
Application cannot connect
          |
          v
      DNS resolve?
       /      \
     NO       YES
     |          |
   DNS fix    Port reachable?
                /       \
              NO        YES
              |           |
         Network fix   SQL handshake?
                           |
                           v
                     Authentication?
                      /          \
                    NO           YES
                    |              |
              Identity/login    Authorization
              investigation    investigation
```

This saves enormous amounts of time.

---

# 15. TLS

After network connectivity, encryption and protocol negotiation matter.

Applications connecting securely to SQL Server/Azure SQL services need appropriate TLS support.

Suppose an old application uses an outdated driver.

Then:

```text id="y4n8c2"
Application
    |
    | outdated TLS/driver behavior
    X
SQL MI
```

You might see connection failures even though:

```text id="m7p2x9"
DNS = good
Network = good
Port = good
Credentials = good
```

The client driver can still be the problem.

---

# 16. Driver compatibility

A production DBA should know what client is connecting.

Ask:

```text id="g8v3k5"
.NET?
JDBC?
ODBC?
Python?
Java?
Legacy application?
SQLCMD?
PowerShell?
```

Then determine:

```text id="r2m6q9"
Driver version
TLS support
Authentication support
Connection-string behavior
Encryption settings
```

A database migration can expose old-client compatibility problems.

---

# 17. Connection string

Another classic incident:

Application connection string:

```text id="p3n7x1"
Server=old-server;
Database=Sales;
```

After migration, SQL MI is:

```text id="k8q2m5"
new-mi-endpoint
```

Application still uses:

```text id="z5r9v3"
old-server
```

Result:

```text id="h4c7m1"
Connection failure
```

The database team says:

> "MI is healthy."

Application team says:

> "SQL is down."

Neither side has yet checked the connection string.

---

# 18. Authentication after networking

Let's say:

```text id="n2x8k4"
DNS ✓
Routing ✓
Port ✓
TLS ✓
```

But:

```text id="q7m3v1"
Login failed
```

Now move to identity.

Questions:

```text id="a6r9p2"
Which identity?
SQL login?
Entra identity?
Managed identity?
Was password rotated?
Did service principal change?
Does SQL user exist?
```

This is why troubleshooting must be layered.

---

# 19. Managed identity + networking

Suppose an Azure application uses managed identity.

The connection process is roughly:

```text id="m8w2c5"
Application
    |
    +-- Network connection to SQL MI
    |
    +-- Obtain Entra token
    |
    v
SQL MI
    |
    +-- Validate identity
    |
    +-- Check database authorization
```

All these steps need to work.

You can have:

```text id="k1q6v9"
Network = perfect
```

but:

```text id="b4n8r2"
Identity authorization = broken
```

Or:

```text id="x7p3m5"
Identity = correct
```

but:

```text id="v9c2k6"
Network = blocked
```

---

# 20. Real production incident

Let's walk through one.

### 08:55

Deployment completed.

### 09:00

Application reports:

```text id="e5m9x3"
SQL connection timeout
```

### Step 1 — SQL MI health

```text id="u7r2p8"
Healthy
```

No issue.

### Step 2 — DNS

Application host:

```text id="j3n6q1"
sql-prod.company.internal
       ↓
10.20.5.10
```

Correct.

### Step 3 — Port

```text id="s8k4m2"
Port test = FAIL
```

Now stop investigating SQL credentials.

You haven't even reached SQL.

---

# 21. Find the network change

Check recent changes.

You discover:

```text id="g9v1c5"
08:50
NSG rule deployment
```

Previous:

```text id="w2m7x4"
ALLOW application subnet
```

New:

```text id="p6r3n8"
DENY
```

Root cause:

> **Network security rule change blocked the application subnet.**

Fix:

```text id="h8q2k6"
Restore intended network rule
       ↓
Retest port
       ↓
Retest SQL connection
       ↓
Validate application
```

---

# 22. Why DBAs need networking skills

Imagine you're a DBA who knows:

```text id="v6x1m9"
SELECT
JOIN
INDEX
BACKUP
RESTORE
```

but doesn't understand:

```text id="q3n7p5"
DNS
VNet
Subnet
NSG
Routing
Ports
TLS
```

Then when the application can't connect, you're dependent on another team for every investigation.

A senior Azure DBA should be able to say:

> "The request is failing before it reaches SQL."

That's powerful.

---

# 23. Connectivity troubleshooting sequence

Use this exact order:

```text id="a7q4m8"
1. Application error
       ↓
2. Connection string
       ↓
3. DNS resolution
       ↓
4. Routing
       ↓
5. Port connectivity
       ↓
6. TLS/driver
       ↓
7. SQL authentication
       ↓
8. SQL authorization
       ↓
9. Database availability
       ↓
10. Application behavior
```

Don't jump around randomly.

---

# 24. Interview question

### "Application can't connect to SQL MI. What do you check?"

A strong answer:

> "I'd first classify the failure as a timeout, network error, TLS/driver error, authentication failure, or authorization failure. Then I'd validate the connection string, DNS resolution from the application environment, routing and network security controls, required port connectivity, TLS and client-driver compatibility, and finally SQL authentication and authorization. I'd also check recent Azure networking or application configuration changes."

Excellent.

---

# 25. Interview question

### "SQL MI is healthy but the application can't connect. What does that tell you?"

Answer:

> "It tells me the managed instance health status alone doesn't prove end-to-end application connectivity. The failure could exist anywhere between the application and SQL MI, including DNS, routing, network security, TLS, client configuration, authentication, or authorization."

---

# 26. Enterprise architecture

A simplified enterprise setup might look like:

```text id="n4p8x2"
                  Users
                    |
                    v
              Application
                    |
              App subnet
                    |
          +---------+---------+
          | Network controls |
          +---------+---------+
                    |
                    v
              SQL MI subnet
                    |
                    v
             Azure SQL MI
                    |
          +---------+---------+
          |                 |
       Identity          Database
          |                 |
     Entra / MI       SQL permissions
          |                 |
          +--------+--------+
                   |
                Auditing
                   |
              Monitoring
```

Notice how many layers can fail.

---

# 27. Senior DBA mental model

When a connection fails, don't think:

> "SQL Server problem."

Think:

```text id="z6m2r8"
Can the client find SQL?
       ↓
Can it reach SQL?
       ↓
Can it negotiate securely?
       ↓
Can it authenticate?
       ↓
Is it authorized?
       ↓
Can it access the database?
```

That sequence will make you much faster at production troubleshooting.

---

# Next — Lesson 68

Next we'll cover **Azure SQL Managed Instance monitoring & alerting**, including a complete production monitoring design:

* Azure Monitor
* Log Analytics
* diagnostic settings
* metrics vs logs
* Query Store
* SQL Agent/job monitoring
* blocking alerts
* CPU/memory/storage alerts
* failed login detection
* deadlock monitoring
* backup monitoring
* and how to build a **2 AM production alert system that tells you what actually went wrong**, rather than simply saying **"SQL is slow."**
