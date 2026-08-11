# Lesson 35 — Complete Azure SQL Managed Instance Production Architecture

You're now the lead DBA for a real company.

## Business

**ABC E-Commerce**

They have:

```text
15,000 transactions/minute
24/7 application
Customer PII
On-premises SQL Server
Multiple applications
Reporting
ETL
```

Requirements:

```text
RPO ≤ 5 minutes
RTO ≤ 30 minutes
Minimal migration downtime
Production + DR
Strong security
Centralized monitoring
```

---

# 1. Target architecture

A reasonable high-level design is:

```text
                         INTERNET
                            |
                            v
                       APPLICATION
                            |
                            v
                    IDENTITY / SECURITY
                            |
                            v
                    AZURE NETWORK
                            |
              +-------------+-------------+
              |                           |
              v                           v
         Application Subnet          MI Subnet
                                          |
                                          v
                                  SQL MANAGED INSTANCE
                                          |
                    +---------------------+---------------------+
                    |                     |                     |
                    v                     v                     v
                   HA                  BACKUP              MONITORING
                    |                     |                     |
                    v                     v                     v
                Failover                 PITR              Azure Monitor
                                          |                     |
                                          v                     v
                                      Recovery               Alerts
                                          |
                                          v
                                           DR
                                          |
                                          v
                                   SECONDARY REGION
```

This is the architecture you should be able to explain in an interview.

---

# 2. Networking

The MI environment should have appropriate Azure networking.

Conceptually:

```text
                Azure VNet
                    |
       +------------+------------+
       |                         |
       v                         v
 Application subnet          MI subnet
       |                         |
       v                         v
 Application                SQL MI
```

For hybrid connectivity:

```text
ON-PREMISES
     |
     v
VPN / ExpressRoute
     |
     v
AZURE VNET
     |
     v
SQL MI
```

The exact connectivity design depends on the organization's requirements.

---

# 3. Why networking matters during migration

Suppose:

```text
On-prem SQL Server
       |
       | migration traffic
       v
SQL MI
```

If the connection is:

```text
Slow
Unstable
High latency
```

your migration may become:

```text
Slow
Unpredictable
Difficult to validate
```

So network readiness is part of migration readiness.

---

# 4. Security architecture

For a production system handling PII:

```text
                    Identity
                       |
                       v
               Microsoft Entra ID
                       |
                       v
                    SQL MI
                       |
            +----------+----------+
            |          |          |
            v          v          v
          Users      Roles     Permissions
            |          |          |
            +----------+----------+
                       |
                       v
                     Data
```

Use:

> **Least privilege**

as the default security principle.

---

# 5. Application identity

Prefer identity-based access where appropriate.

Instead of:

```text
Application
    |
    +-- SQL username
    +-- password
```

use:

```text
Application
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

This reduces credential-management complexity.

---

# 6. Protecting customer data

For sensitive data, consider multiple security layers:

```text
Network security
       +
Authentication
       +
Authorization
       +
Encryption
       +
Auditing
       +
Monitoring
```

Don't think:

> "We enabled encryption, therefore the database is secure."

Security is layered.

---

# 7. HA design

Your first objective:

> **Keep the database service available when individual infrastructure failures occur.**

Managed Instance provides platform-managed availability mechanisms.

Your responsibility is to ensure:

```text
Application
     |
     v
Can tolerate transient connectivity
     |
     v
Failover
     |
     v
Reconnect
```

---

# 8. Application retry logic

Suppose failover occurs.

Application gets:

```text
Transient connection error
```

A resilient application can:

```text
Retry
  ↓
Reconnect
  ↓
Continue
```

Without retry handling:

```text
Failover
   ↓
Application error
   ↓
User sees failure
```

So HA is not just a database concern.

---

# 9. DR architecture

For a major regional failure:

```text
                 REGION A
                    |
                 Production
                    |
                    X
              Regional outage
                    |
                    v
                 REGION B
                    |
                  DR
```

But remember:

> **DR is an entire solution, not merely a database copy.**

You need to consider:

```text
Database
Application
Networking
Identity
Secrets
DNS
Monitoring
Security
Operational runbook
```

---

# 10. RPO

Business requirement:

```text
RPO ≤ 5 minutes
```

Meaning:

> The business can tolerate at most approximately five minutes of data loss in the defined disaster scenario.

Your architecture and recovery mechanism must actually satisfy that requirement.

Don't simply write:

```text
RPO = 5 minutes
```

in a design document.

You must **test it**.

---

# 11. RTO

Requirement:

```text
RTO ≤ 30 minutes
```

Meaning:

> The service should be restored within approximately 30 minutes after the defined disaster event.

Again:

```text
Design
  ↓
Test
  ↓
Measure
```

If recovery takes:

```text
47 minutes
```

then your architecture does not meet the requirement.

---

# 12. Backup architecture

Think in terms of:

```text
Production database
       |
       v
Automated backup
       |
       v
Recovery capability
       |
       v
Point-in-time recovery
```

But backup alone isn't enough.

Test:

```text
Backup
 ↓
Restore
 ↓
Validate
 ↓
Measure RTO
```

---

# 13. Monitoring architecture

A production system should feed telemetry into centralized monitoring.

Conceptually:

```text
SQL MI
 |
 +-- Metrics
 +-- Logs
 +-- Diagnostic information
 |
 v
Azure Monitor
 |
 +-- Alerts
 +-- Dashboards
 +-- Log Analytics
 |
 v
Operations team
```

---

# 14. What should trigger an alert?

Examples:

```text
High CPU sustained
Storage approaching capacity
Availability issue
Large blocking chain
Critical errors
Security events
Failed operational jobs
Unexpected workload changes
```

Avoid creating:

```text
500 noisy alerts
```

Instead create:

```text
Actionable alerts
```

---

# 15. Performance architecture

Your performance stack:

```text
Azure metrics
      |
      v
CPU / IO / storage
      |
      v
SQL workload
      |
      v
DMVs
      |
      v
Query Store
      |
      v
Execution plans
      |
      v
Root cause
```

This is your troubleshooting chain.

---

# 16. Query Store strategy

Query Store gives you historical visibility.

For example:

```text
Monday
Query = 200 ms

Tuesday
Query = 250 ms

Wednesday
Query = 7 sec
```

You can investigate:

```text
Plan change
Statistics
Data growth
Index change
Parameter behavior
Application deployment
```

That's why Query Store is extremely valuable for production DBAs.

---

# 17. Migration architecture

Now move the existing SQL Server workload.

```text
ON-PREMISES
SQL Server
    |
    | Assessment
    v
Compatibility analysis
    |
    v
Network connectivity
    |
    v
Test migration
    |
    v
Application testing
    |
    v
Performance testing
    |
    v
Production migration
    |
    v
Cutover
    |
    v
SQL MI
```

---

# 18. Dependency assessment

You have:

```text
120 SQL Agent jobs
80 logins
5 linked servers
15 SSIS packages
```

These must be assessed.

Your migration isn't complete simply because:

```text
8 TB databases
```

are successfully moved.

You also need to understand:

```text
Jobs
Security
ETL
External systems
Reporting
Application connections
```

---

# 19. Migration testing

Create a test environment.

Test:

```text
Database restore/migration
Application connectivity
Authentication
Jobs
ETL
Reports
Performance
Security
```

Then document issues.

For example:

```text
Issue #1
Linked server dependency

Issue #2
Application login

Issue #3
SSIS connectivity

Issue #4
Slow reporting query
```

Fix these **before production**.

---

# 20. Production cutover

A good runbook:

```text
T-7 days
Final rehearsal

T-1 day
Change freeze

T-1 hour
Readiness checks

T-15 min
Application maintenance mode

T
Final synchronization

T+5
Data validation

T+10
Application redirect

T+15
Smoke testing

T+20
Business validation
```

You should define:

```text
Success criteria
Rollback criteria
Owners
Communication
Escalation
```

---

# 21. Rollback

Suppose payment processing fails after cutover.

You need to know:

```text
What data was written to MI?
What data exists only in MI?
Can changes be reconciled?
Is source still usable?
```

Rollback should be designed **before** cutover.

Not invented during an incident.

---

# 22. Post-migration operations

After successful migration:

```text
Day 1
  ↓
Intensive monitoring

Week 1
  ↓
Performance comparison

Month 1
  ↓
Capacity review

Ongoing
  ↓
Security
Performance
DR
Backup
Operations
```

Compare against your pre-migration baseline.

---

# Lesson 36 — Your Senior DBA Mental Model

When you're given any SQL MI problem, use:

```text
              PROBLEM
                 |
                 v
             SYMPTOM
                 |
                 v
              EVIDENCE
                 |
                 v
             HYPOTHESIS
                 |
                 v
               TEST
                 |
                 v
            ROOT CAUSE
                 |
                 v
             MITIGATION
                 |
                 v
          PERMANENT FIX
                 |
                 v
             VALIDATE
                 |
                 v
             PREVENT
```

Example:

```text
Users report slow application
        ↓
CPU = 95%
        ↓
Query A = 80% CPU
        ↓
Plan regression
        ↓
Compare plans
        ↓
Statistics caused bad estimate
        ↓
Correct statistics/plan strategy
        ↓
CPU = 45%
        ↓
Monitor
        ↓
Create regression alert
```

This is how you should answer scenario-based interview questions.

---

# Lesson 37 — Final Mock Interview Begins

I'm now going to treat you like a candidate for an **Azure SQL DBA / SQL Server DBA with Azure experience**.

### Question 1

> **Your production Azure SQL Managed Instance suddenly reaches 95% CPU. Users report that the application is slow. How would you troubleshoot this?**

A strong answer should cover:

```text
1. Confirm the issue
2. Determine whether CPU is sustained
3. Identify top CPU-consuming queries
4. Check Query Store
5. Compare execution plans
6. Check recent deployments/changes
7. Investigate statistics/indexes
8. Check waits and workload behavior
9. Implement the least risky fix
10. Validate the result
11. Prevent recurrence
```

---

### Question 2

> **Your MI has 300 blocked sessions. What do you do?**

Think:

```text
Find head blocker
      ↓
Identify transaction
      ↓
Determine why locks are held
      ↓
Assess business impact
      ↓
Mitigate safely
      ↓
Fix root cause
```

---

### Question 3

> **Your company wants to migrate an 8-TB SQL Server database to MI with less than 15 minutes downtime. How do you approach it?**

Your answer should include:

```text
Assessment
Compatibility
Network
Migration method
Initial data movement
Continuous synchronization
Cutover
Validation
Rollback
```

---

### Question 4

> **The business says RPO is 5 minutes and RTO is 30 minutes. What do those mean?**

Your answer:

```text
RPO
= maximum acceptable data loss measured in time

RTO
= maximum acceptable time to restore service
```

Then explain how you would **test** both.

---

### Question 5

> **A query that used to run in 200 ms now takes 8 seconds, but CPU and IO are normal. What do you investigate?**

Think:

```text
Query Store
   ↓
Plan regression
   ↓
Execution plan comparison
   ↓
Statistics
Indexes
Parameter sensitivity
Recent changes
```

---

# Where we are now

You have completed the **core learning curriculum**.

The remaining work is no longer learning new Azure SQL MI concepts. It's **practice and assessment**:

```text
                  AZURE SQL MI
                       |
        +--------------+--------------+
        |              |              |
     THEORY        OPERATIONS      INTERVIEW
        |              |              |
     COMPLETE       PRACTICE         NEXT
```


You'll diagnose each one, and I'll score your answers like a senior DBA interviewer.
