**Hands-on Azure SQL Managed Instance DBA Notes** using one realistic production environment throughout.

## Lesson 1 — Our production project

We'll use this company:

```text
ABC Retail
```

Current environment:

```text
ON-PREMISES DATA CENTER
│
└── SQL Server 2019
     │
     ├── SalesDB
     ├── CustomerDB
     ├── InventoryDB
     └── ReportingDB
```

The server also has:

```text
SQL Agent Jobs
Linked Servers
Logins
Cross-database queries
Stored Procedures
Backups
```

The company wants:

```text
                    AZURE
                      │
             SQL Managed Instance
                      │
          ┌───────────┼───────────┐
          │           │           │
       SalesDB    CustomerDB   InventoryDB
          │
          ↓
      Reporting
```

### Business requirements

| Requirement            | Target               |
| ---------------------- | -------------------- |
| Migration downtime     | Minimal              |
| HA                     | Required             |
| DR                     | Required             |
| Backups                | Automated            |
| Security               | Private connectivity |
| Application changes    | Minimal              |
| SQL Agent              | Required             |
| Cross-database queries | Required             |
| Future scaling         | Required             |

---

# Lesson 1A — First decision: Why MI?

Imagine you're the DBA and your manager asks:

> "Why aren't we migrating these databases to Azure SQL Database?"

Your answer should be:

**Because this is an existing SQL Server workload with instance-level and cross-database dependencies. Azure SQL Managed Instance provides much greater SQL Server compatibility while removing much of the underlying infrastructure administration.**

That's the first architectural decision.

```text
Existing SQL Server
        │
        │
        ├── Cross-database queries
        ├── SQL Agent
        ├── Logins
        ├── Linked servers
        ├── Existing T-SQL
        │
        ↓
Azure SQL Managed Instance
```

---

# Lesson 1B — Draw the network

Before migrating **one database**, you need networking.

Our architecture will eventually look like:

```text
                 ON-PREMISES
                     │
               SQL Server 2019
                     │
                     │
             VPN / ExpressRoute
                     │
                     ↓
              Azure VNet
                     │
              ┌──────┴──────┐
              │             │
              ↓             ↓
         App Subnet      MI Subnet
              │             │
          Application       │
              │             │
              └──────→ SQL Managed
                        Instance
```

This is important:

> **Don't start an MI migration by thinking about the database first. Start by thinking about connectivity.**

If the application can't reach MI, the database migration doesn't matter.

---

# Lesson 1C — DBA questions before migration

Before touching Azure, I'd ask:

### Database questions

```text
How many databases?
How large are they?
What SQL Server version?
What compatibility levels?
What recovery models?
What are the largest tables?
What are the busiest databases?
```

### Application questions

```text
Which applications connect?
Which connection strings?
Which applications use cross-database queries?
Are there linked servers?
Are there distributed transactions?
```

### SQL Server questions

```text
How many logins?
How many SQL Agent jobs?
Any credentials?
Any certificates?
Any server-level triggers?
Any CLR?
Any Service Broker?
Any replication?
```

### Performance questions

```text
CPU?
Memory?
IOPS?
Throughput?
Peak connections?
Peak workload?
Top queries?
Blocking?
Deadlocks?
```

### Business questions

```text
What is the maximum downtime?
What is the RPO?
What is the RTO?
What is the DR region?
```

This is **migration assessment**.

---

# Your first DBA challenge 🧑‍💻

Imagine I am your manager.

I tell you:

> **"We have a 4-TB SQL Server 2019 production database. The application runs 24/7. We want to migrate it to Azure SQL Managed Instance. We can tolerate only 10 minutes of downtime. We also need DR."**

Before we talk about tools, answer these **5 questions**:

1. **Would you choose General Purpose or Business Critical? Why?**
2. **Would you use backup/restore for this migration? Why or why not?**
3. **What migration approach would you investigate for only 10 minutes of downtime?**
4. **What information would you collect from the existing SQL Server before migration?**
5. **What is the difference between RPO and RTO?**

Don't worry about getting them all right.

