# Lesson 51 — Azure SQL Managed Instance Interview Masterclass

Let's start with one of the **most important Azure SQL MI interview questions**.

## Question 1: Why choose Azure SQL Managed Instance instead of Azure SQL Database?

A weak answer is:

> "Managed Instance is like SQL Server in Azure."

That's directionally correct, but not enough for a senior interview.

The better answer is about **compatibility, features, migration effort, and operational model**.

---

# 1. The three options

Think of the Azure SQL family like this:

```text
                         Azure SQL
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
       SQL Database    Managed Instance   SQL Server
       PaaS database   PaaS instance      IaaS/on-prem
```

The key difference is **how much of the SQL Server environment you want Microsoft to manage**.

---

# 2. SQL Server on Azure VM

This is closest to traditional SQL Server.

You control:

```text
OS
SQL Server installation
SQL configuration
Patching strategy
VM
Storage
Networking
```

Think:

> **"I want SQL Server, but running on Azure infrastructure."**

Useful when you need extensive OS-level control or specific SQL Server/third-party requirements.

---

# 3. Azure SQL Database

This is more database-focused.

You typically don't manage:

```text
Windows OS
SQL Server instance
Instance-level infrastructure
```

You focus on:

```text
Database
Schema
Queries
Indexes
Security
Application
```

It's excellent for applications designed around a cloud database platform.

But traditional SQL Server workloads may require changes when moving to SQL Database.

---

# 4. Azure SQL Managed Instance

Managed Instance sits between those two models.

Conceptually:

```text
Traditional SQL Server
        |
        | compatibility
        v
Managed Instance
        |
        | PaaS management
        v
Microsoft manages infrastructure
```

You get a SQL Server instance-like environment while Microsoft manages much of the underlying platform.

---

# 5. Why migration teams love MI

Imagine a company has:

```text
SQL Server 2019
300 databases
```

and applications rely on:

```text
SQL Agent
Cross-database queries
Instance-level features
Multiple databases
Traditional SQL Server behavior
```

Migrating everything directly to SQL Database might require substantial application redesign.

MI can reduce that migration gap.

The strategy becomes:

```text
On-prem SQL Server
       ↓
Compatibility assessment
       ↓
Azure SQL Managed Instance
       ↓
Application modernization later
```

This is often called a **modernize gradually** approach.

---

# 6. Real-world scenario

Suppose an insurance company has:

```text
PolicyDB
ClaimsDB
CustomerDB
ReportingDB
```

Applications perform:

```sql
SELECT *
FROM PolicyDB.dbo.Policies p
JOIN CustomerDB.dbo.Customers c
    ON p.CustomerID = c.CustomerID;
```

That's a cross-database workload.

The company wants to move to Azure quickly without immediately redesigning the application.

MI can be attractive because its instance-oriented model is much closer to traditional SQL Server.

---

# 7. Another important scenario: SQL Agent

Suppose an existing environment has hundreds of SQL Agent jobs:

```text
01:00 → ETL
02:00 → Index maintenance
03:00 → Reporting refresh
04:00 → Data cleanup
```

If the organization has a large dependency on SQL Server Agent-style scheduling, MI can reduce migration friction compared with moving the workload to a database service that doesn't provide the same instance-level SQL Server experience.

But always validate the **specific feature and version compatibility** rather than assuming every SQL Server feature behaves identically.

---

# 8. Managed Instance is still PaaS

This is an important interview point.

You aren't managing the underlying operating system like you would with SQL Server on an Azure VM.

Conceptually:

```text
                    YOU
                     |
        +------------+-------------+
        |                          |
    Database                  SQL workload
    Security                  Performance
    Schema                    Query tuning
    Permissions               Application
        |
        v
Microsoft
    Infrastructure
    OS
    Platform management
```

That means:

> **Less infrastructure management, more focus on database workloads.**

---

# 9. When would you choose SQL Database?

Imagine a modern application:

```text
Web API
   ↓
One database
   ↓
Modern application architecture
```

The application doesn't depend on:

```text
Cross-database architecture
SQL Agent
Instance-level SQL Server features
Legacy SQL Server behaviors
```

Then Azure SQL Database may be a better fit.

You don't choose MI simply because it is "more powerful."

You choose based on **requirements**.

---

# 10. When would you choose SQL Server on Azure VM?

Suppose you need:

```text
OS-level access
Third-party SQL extensions
Specialized agents
Specific SQL Server configuration
Full infrastructure control
```

Then SQL Server on Azure VM may make more sense.

Think:

```text
Maximum control
       ↑
SQL Server VM
       |
Managed Instance
       |
Azure SQL Database
       ↓
Maximum platform abstraction
```

That's a useful mental model, although actual feature boundaries depend on the specific requirement.

---

# 11. Interview scenario

### Interviewer:

> "We have a 10-year-old SQL Server application with 100 databases and many cross-database queries. Management wants to migrate to Azure quickly. Which service would you consider?"

A strong response:

> "I'd first perform a compatibility and dependency assessment. If the workload depends heavily on instance-level SQL Server capabilities and cross-database behavior, Managed Instance would be a strong candidate because it provides a managed PaaS environment while maintaining a SQL Server-compatible instance model. I'd validate the specific features and application dependencies before committing to the architecture."

Notice that you're not saying:

> "Always use MI."

You're saying:

> **Assess → identify requirements → select platform.**

---

# 12. The migration decision matrix

| Requirement                   | SQL Database                  | Managed Instance | SQL Server VM |
| ----------------------------- | ----------------------------- | ---------------- | ------------- |
| PaaS                          | ✅                             | ✅                | ❌             |
| OS management                 | ❌                             | ❌                | ✅             |
| SQL Server-like instance      | Limited                       | Strong           | Full          |
| Legacy migration friendliness | Medium                        | High             | Very high     |
| OS-level customization        | ❌                             | ❌                | ✅             |
| Cross-database workloads      | More limited/design-dependent | Strong fit       | Strong        |
| Full infrastructure control   | ❌                             | ❌                | ✅             |

**Important:** exact feature availability and limitations should always be checked against the current Azure documentation before an architecture decision.

---

# 13. The senior-level answer

If asked:

> **"Why Managed Instance?"**

Say:

> "I would choose Azure SQL Managed Instance when I need a PaaS service but the workload has significant SQL Server instance-level compatibility requirements. It can reduce migration effort for applications using multiple databases, traditional SQL Server features, and other instance-oriented capabilities. Compared with SQL Server on Azure VM, I give up OS-level control in exchange for Microsoft-managed infrastructure. Compared with Azure SQL Database, MI generally provides a closer SQL Server compatibility model. The final choice should be based on a detailed compatibility and dependency assessment."

That's a strong interview answer.

---

# Lesson 52 — HA vs DR: The Question Interviewers Love

A very common question:

> **"What's the difference between High Availability and Disaster Recovery?"**

### High Availability

Goal:

> **Keep the service available when a component fails.**

Think:

```text
Primary
   |
   X
   |
Secondary
   |
   v
Service continues
```

Examples include platform-supported high-availability mechanisms within the service.

---

### Disaster Recovery

Goal:

> **Recover the service after a major failure, such as a regional disaster.**

Think:

```text
Primary Region
      X
      |
      v
Secondary Region
      |
      v
Application restored
```

---

# 14. Simple example

### HA

```text
Server/component failure
       ↓
Failover
       ↓
Service continues
```

Potentially very small interruption.

### DR

```text
Region unavailable
       ↓
Activate secondary environment
       ↓
Recover data
       ↓
Start application
       ↓
Validate
       ↓
Service restored
```

Usually a larger operational process.

---

# 15. RPO + RTO

Always connect these concepts.

```text
HA
 ↓
Availability

DR
 ↓
RPO + RTO
```

### RPO

How much data can we lose?

```text
Failure:       10:00
Recoverable:    9:57

RPO ≈ 3 minutes
```

### RTO

How long can the service be unavailable?

```text
Failure:       10:00
Service back:  10:22

RTO = 22 minutes
```

---

# 16. Interview trap

Interviewer:

> "We have geo-replication, so our RTO is zero."

❌ Wrong.

Replication doesn't automatically equal zero RTO.

You must account for:

```text
Detection
Decision
Failover
DNS/network
Application startup
Authentication
Validation
```

That's why you **test RTO instead of guessing it**.

---

# 17. Another trap

> "Our RPO is zero because we have synchronous HA."

Don't automatically make that claim either.

RPO depends on the **specific architecture and failure scenario**.

A local high-availability mechanism and a regional disaster-recovery strategy solve different problems.

Always ask:

```text
What failure are we protecting against?
Where is the secondary?
How is data synchronized?
What happens during the actual disaster?
```

---

# 18. Senior DBA architecture picture

Keep this in your head:

```text
                 APPLICATION
                      |
                      v
              +---------------+
              | Azure SQL MI   |
              |   PRIMARY      |
              +---------------+
                      |
                 HA mechanism
                      |
              +---------------+
              | Local/paired   |
              | HA protection  |
              +---------------+

                      +

              Regional DR
                      |
                      v
              +---------------+
              | Azure SQL MI   |
              |   SECONDARY    |
              +---------------+
                      |
                      v
                 DR APP
```

**HA protects availability.**

**DR protects business continuity from larger failures.**

---

