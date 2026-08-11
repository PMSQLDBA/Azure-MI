**Lesson 15 — Production-Grade Azure SQL Managed Instance: HA, Failover, DR, Monitoring, and Operations**.

At this point, you've learned the individual pieces. Now we're going to connect them into **one production architecture** and think like an Azure SQL DBA.

---

# Lesson 15 — Production-Grade Azure SQL Managed Instance

Imagine you're responsible for this business:

> **ABC Retail processes thousands of orders every minute. The database cannot simply "go down."**

Your job is to design for:

* High availability
* Disaster recovery
* Performance
* Security
* Monitoring
* Backup/recovery
* Controlled maintenance
* Operational troubleshooting

---

# 1. First: HA vs DR

This distinction must be crystal clear.

### High Availability

Question:

> **"What happens if the database infrastructure has a failure?"**

Example:

```text
Primary MI
    |
    X
Failure
    |
    v
HA mechanism
    |
    v
Service recovers
```

The objective is minimizing service interruption.

### Disaster Recovery

Question:

> **"What happens if an entire region or major environment becomes unavailable?"**

Example:

```text
Region A
   |
   X
Regional outage
   |
   v
Region B
   |
   v
DR environment
```

---

# 2. Production architecture

A simplified enterprise architecture:

```text
                       USERS
                         |
                         v
                 Application Layer
                         |
                         v
                 Azure Load Balancer
                         |
                         v
                    App Services
                         |
                         v
                +------------------+
                |    Azure VNet    |
                |                  |
                | App subnet       |
                |                  |
                | MI subnet        |
                +--------+---------+
                         |
                         v
                 SQL Managed Instance
                         |
              +----------+----------+
              |                     |
              v                     v
         HA / Failover          Backup
              |                     |
              |                     v
              |                  Recovery
              |
              v
         DR architecture
              |
              v
        Secondary Region
```

This is the high-level picture.

---

# 3. Why MI is attractive for HA

With traditional SQL Server on a VM, you may have to design and operate:

```text
VM
OS
SQL Server
Failover Cluster
Storage
Patching
Networking
Monitoring
```

With Managed Instance, Microsoft manages much of the underlying platform.

Your DBA responsibilities move more toward:

```text
Database
Configuration
Performance
Security
Workload
Recovery
Monitoring
Application dependencies
```

That's one of the major PaaS benefits.

---

# 4. Failover

Suppose the current primary becomes unavailable.

Conceptually:

```text
Before:

Application
    |
    v
Primary
    |
    v
Database


After failure:

Application
    |
    v
Failover
    |
    v
New healthy primary
```

Your application needs to be designed to tolerate transient connectivity interruptions.

---

# 5. Transient connection failures

After a failover, an application may temporarily receive:

```text
Connection error
Timeout
Connection reset
Transient SQL error
```

A well-designed application should use:

```text
Retry logic
Connection pooling
Appropriate command timeout
```

This is an important collaboration point between DBA and developers.

You cannot solve every availability issue solely from the database side.

---

# 6. Example: application without retry logic

Imagine:

```text
09:00:00
Application → MI
```

Failover happens:

```text
09:00:05
Connection interrupted
```

Application:

```text
ERROR
User sees failure
```

With retry logic:

```text
09:00:05
Connection fails

09:00:06
Retry

09:00:07
Retry

09:00:08
Connection succeeds
```

The user may never notice.

---

# 7. DR design

Now suppose:

```text
Primary Region
     |
     X
Complete regional outage
```

Your organization might have:

```text
Primary Region
       |
       v
DR Region
       |
       v
Secondary MI architecture
```

But DR isn't just copying the database.

You need:

```text
Database
+
Network
+
DNS
+
Identity
+
Security
+
Application
+
Secrets
+
Monitoring
+
Runbooks
```

---

# 8. DNS during DR

Application may use:

```text
db.company.com
```

During normal operations:

```text
db.company.com
       |
       v
Primary environment
```

During DR:

```text
db.company.com
       |
       v
DR environment
```

The exact implementation depends on your architecture, but the principle is important:

> **Application connectivity must be part of DR design.**

---

# 9. Monitoring architecture

A production DBA shouldn't discover problems from users.

Bad:

```text
Customer:
"Website is slow."
```

Better:

```text
Monitoring
    |
    v
Alert
    |
    v
DBA
```

You monitor:

```text
CPU
IO
Storage
Connections
Blocking
Deadlocks
Query performance
Errors
Availability
Security
```

---

# 10. Azure Monitor

Azure Monitor provides the broader monitoring platform.

Conceptually:

```text
SQL MI
  |
  v
Metrics / Logs
  |
  v
Azure Monitor
  |
  +---- Alerts
  |
  +---- Dashboards
  |
  +---- Log Analytics
```

The goal is to turn raw telemetry into operational information.

---

# 11. Alerts

Don't create alerts for everything.

Create alerts around things that matter.

Examples:

```text
CPU sustained above threshold
Storage approaching limit
Critical failures
Availability problems
Connection failures
Security events
```

And consider:

```text
Alert severity
Business impact
Duration
Noise
Escalation
```

---

# 12. Alert fatigue

Suppose the DBA receives:

```text
500 alerts/day
```

Eventually:

> Nobody pays attention.

That's dangerous.

Better:

```text
10 useful alerts
```

that represent meaningful conditions.

Good monitoring asks:

> **"If this alert fires at 3 AM, will someone know what action to take?"**

---

# 13. Dashboard

A useful production dashboard could contain:

```text
SQL MI Production
--------------------------------
CPU                  42%
Storage              61%
Connections           320
Blocked sessions        0
Deadlocks              0
Query latency         240 ms
Errors                 2
Availability          Healthy
```

This gives you an immediate operational view.

---

# 14. Performance monitoring

Don't only monitor infrastructure.

You need:

```text
Infrastructure metrics
+
Database metrics
+
Query metrics
```

For example:

```text
Azure metrics
     ↓
CPU / IO / storage

SQL monitoring
     ↓
Waits / blocking / sessions

Query Store
     ↓
Query duration / plans / regressions
```

Together they tell the real story.

---

# 15. Incident scenario

Alert:

```text
CPU = 95%
```

You investigate.

Azure metrics:

```text
CPU high
```

SQL:

```text
Blocking = 0
```

Query Store:

```text
New query appeared 10 minutes ago
```

Execution plan:

```text
Large table scan
```

Application deployment:

```text
10 minutes ago
```

Root cause:

```text
New application release
```

This is much better than simply saying:

> "CPU was high."

---

# 16. Storage monitoring

Storage issues can become serious.

Suppose:

```text
Database size
      ↓
Growing rapidly
```

You should ask:

```text
Which database?
Which tables?
Data or log?
Expected growth?
Unexpected growth?
Retention problem?
Application behavior?
```

Don't simply react after the system reaches a limit.

---

# 17. Capacity planning

A DBA should think ahead.

Suppose:

```text
Current DB size = 4 TB
Growth = 100 GB/month
```

Then:

```text
12 months
≈ 1.2 TB additional growth
```

Now you can plan.

Also consider:

```text
CPU growth
Transaction growth
Connection growth
IO workload
Peak periods
```

---

# 18. Black Friday scenario

ABC Retail normally processes:

```text
5,000 orders/hour
```

Black Friday:

```text
30,000 orders/hour
```

If you only design based on normal traffic:

```text
Production
     |
     v
Traffic spike
     |
     v
Performance degradation
```

Capacity planning must consider peak workloads.

---

# 19. Maintenance planning

Managed Instance reduces infrastructure maintenance, but DBAs still have operational responsibilities.

Examples:

```text
Index strategy
Statistics
Query tuning
Security reviews
Job management
Database configuration
Application compatibility
Capacity planning
DR testing
```

You still own the workload.

---

# 20. Change management

Never make production changes casually.

A good change process:

```text
Change request
      ↓
Impact assessment
      ↓
Testing
      ↓
Approval
      ↓
Maintenance window
      ↓
Implementation
      ↓
Validation
      ↓
Rollback if necessary
      ↓
Documentation
```

---

# 21. Example production change

You want to add an index:

```sql id="x9v4km"
CREATE INDEX IX_Orders_CustomerID
ON Orders(CustomerID);
```

Don't immediately run it in production.

First consider:

```text
Does it improve the query?
How much storage?
Write overhead?
Build duration?
Impact on workload?
Existing indexes?
```

Then:

```text
Test
 ↓
Measure
 ↓
Approve
 ↓
Deploy
 ↓
Validate
```

---

# 22. Security operations

Security isn't a one-time setup.

You should periodically review:

```text
Users
Logins
Roles
Permissions
Managed identities
Secrets
Auditing
Security alerts
Network access
```

Especially:

> **Who has administrative access?**

That's one of the most important questions.

---

# 23. Break-glass access

Production environments sometimes need emergency administrative access.

But it should be:

```text
Controlled
Audited
Limited
Monitored
```

Don't have:

```text
Everyone → sysadmin
```

That's not a security architecture.

---

# 24. Disaster recovery exercise

A mature organization schedules DR testing.

Example:

```text
Quarterly DR exercise

1. Simulate region outage
2. Activate DR
3. Redirect application
4. Validate database
5. Validate authentication
6. Validate critical workflows
7. Measure RTO
8. Measure RPO
9. Document gaps
10. Improve architecture
```

The exercise is successful only if the business workflow works.

---

# 25. Production DBA daily routine

A practical daily checklist:

```text
Morning
│
├── Availability
├── Critical alerts
├── CPU
├── IO
├── Storage
├── Blocking
├── Deadlocks
├── Failed jobs
├── Query regressions
├── Security alerts
└── Backup/recovery health
```

Then:

```text
Weekly
│
├── Capacity review
├── Performance trends
├── Security review
├── Job review
└── DR/backup checks
```

And periodically:

```text
DR test
Restore test
Access review
Architecture review
```

---

# 26. The senior DBA mindset

A junior DBA asks:

> "What failed?"

A senior DBA asks:

> **"Why did it fail, how did we detect it, how quickly did we recover, and how do we prevent it from happening again?"**

That's the difference.

---

# 27. Full production architecture

Put everything together:

```text
                         USERS
                           |
                           v
                    APPLICATION
                           |
                           v
                  Identity / Security
                           |
                           v
                    Azure Network
                           |
                  +--------+--------+
                  |                 |
                  v                 v
             App Subnet         MI Subnet
                                    |
                                    v
                           SQL MANAGED INSTANCE
                                    |
               +--------------------+--------------------+
               |                    |                    |
               v                    v                    v
             HA                  Backup                Monitoring
               |                    |                    |
               v                    v                    v
           Failover               PITR             Azure Monitor
                                    |                    |
                                    v                    v
                               Recovery              Alerts
                                    |
                                    v
                                  DR
                                    |
                                    v
                            Secondary Region
```

And around all of it:

```text
Security
Networking
Identity
Auditing
Capacity Planning
Change Management
```

---

# 28. What you should now be able to explain

At this point, you should be comfortable explaining:

### Azure SQL MI fundamentals

* What MI is
* Why companies use it
* PaaS vs SQL Server on VM

### Networking

* VNet
* MI subnet
* NSG
* DNS
* VPN
* ExpressRoute
* Routing

### Security

* Authentication
* Authorization
* Microsoft Entra ID
* Managed identity
* Least privilege
* TDE
* TLS
* Key Vault
* Auditing

### Recovery

* Backup
* PITR
* RPO
* RTO
* DR
* Restore testing

### Performance

* CPU
* IO
* Blocking
* Deadlocks
* Waits
* DMVs
* Query Store
* Execution plans
* Indexes

### Migration

* Assessment
* Compatibility
* Dependencies
* Online migration
* Cutover
* Rollback
* Validation

That's already a strong foundation for an **Azure SQL DBA** role.

---

# Final architecture challenge 🧑‍💻

Imagine you're interviewing for a senior Azure SQL DBA role.

The interviewer says:

> **"Design a production Azure SQL Managed Instance environment for an e-commerce company."**

Requirements:

```text
24/7 application
10,000 transactions/minute
Sensitive customer data
Minimal downtime
RPO ≤ 5 minutes
RTO ≤ 30 minutes
Hybrid connectivity to on-premises
Production + DR
Strong auditing
```

Give me your architecture under these headings:

```text
1. Network
2. SQL Managed Instance
3. Security
4. HA
5. Backup
6. DR
7. Monitoring
8. Performance
9. Migration
10. Operational procedures
```

Don't worry if you don't know everything yet.

**Try designing it yourself first.** 
