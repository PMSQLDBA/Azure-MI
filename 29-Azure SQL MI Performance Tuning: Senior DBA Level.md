# Lesson 61 — Azure SQL MI Performance Tuning: Senior DBA Level

Now we're moving into the part that usually separates a **junior SQL DBA** from a **senior production DBA**:

> **Finding the real cause of a performance problem instead of treating the symptom.**

---

# 1. The golden rule

When someone says:

> "SQL MI is slow."

That is **not a diagnosis**.

"Slow" could mean:

```text
Application
   ↓
Network latency
   ↓
Connection establishment
   ↓
SQL execution
   ↓
Blocking
   ↓
CPU
   ↓
IO
   ↓
Memory
   ↓
Application processing
```

Your first job is to locate **where the latency actually occurs**.

---

# 2. Real production incident

It's 2:00 AM.

Monitoring sends:

```text
ALERT:
Application response time > 10 seconds
```

Developers report:

> "SQL is extremely slow."

You connect to the MI.

You see:

```text
CPU:       94%
Storage:   40%
Connections: normal
MI:        healthy
```

What do you do?

### Don't do this:

```text
Scale compute immediately
```

Why?

Because CPU may be the **symptom**, not the root cause.

---

# 3. Step 1 — Find the expensive queries

Start with current workload.

```sql id="q4j5f8"
SELECT TOP (20)
    r.session_id,
    r.status,
    r.command,
    DB_NAME(r.database_id) AS database_name,
    r.cpu_time,
    r.total_elapsed_time,
    r.logical_reads,
    r.writes,
    r.wait_type,
    r.wait_time,
    r.blocking_session_id,
    t.text AS sql_text
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE r.session_id <> @@SPID
ORDER BY r.cpu_time DESC;
```

Suppose you find:

```text id="v5p8q2"
Session 82
CPU = extremely high
Reads = extremely high
Query = OrderSearch
```

Now you have a suspect.

---

# 4. Step 2 — Is CPU actually the problem?

Suppose:

```text id="n0r4u8"
CPU = 95%
```

Look at waits.

Why?

Because:

```text
High CPU
```

can result from:

```text
Bad query plan
Large scan
Poor join strategy
Missing/ineffective index
Huge data volume
High concurrency
```

You need to know **what is consuming CPU**.

---

# 5. Step 3 — Query Store

Now check Query Store.

You want to answer:

> "Was this query always expensive?"

Imagine:

```text id="h2z6w9"
Yesterday:
Duration = 200 ms

Today:
Duration = 12 seconds
```

That is a huge clue.

The query itself may not have changed.

Its **execution plan may have changed**.

---

# 6. Plan regression

Imagine two plans:

### Plan A

```text id="2p4q7s"
Index Seek
    ↓
Nested Loops
```

Performance:

```text
200 ms
```

### Plan B

```text id="6n1v8c"
Table Scan
    ↓
Hash Join
```

Performance:

```text
12 seconds
```

If Query Store shows the query switched from A → B, you have a **plan regression investigation**.

---

# 7. Why would a plan change?

Possible causes:

```text id="y8r3c2"
Statistics changed
Data distribution changed
Index changed
Parameter sensitivity
Cardinality estimation
Database compatibility changes
Schema changes
Different parameter values
Optimizer decisions
```

Don't assume:

> "SQL Server randomly chose a bad plan."

Find out **why the optimizer chose it**.

---

# 8. Parameter sensitivity

This is one of the most common advanced SQL performance topics.

Suppose:

```sql id="q0y2x8"
SELECT *
FROM Orders
WHERE CustomerID = @CustomerID;
```

Customer 1:

```text id="4h3z9p"
10 rows
```

Customer 2:

```text id="j8x1m5"
5,000,000 rows
```

Same query.

Very different data distribution.

An execution strategy that's excellent for:

```text id="r8q6v1"
10 rows
```

may be terrible for:

```text id="d2n7w4"
5 million rows
```

This is where parameter-sensitive plan behavior becomes important.

---

# 9. What might you see?

For a small customer:

```text id="3m7p2k"
Index Seek
→ 10 rows
→ excellent
```

For a huge customer:

```text id="s8j4q6"
Index Seek
→ millions of lookups
→ terrible
```

The optimizer's plan choice must account for the data distribution and parameter values.

This is why parameter sensitivity can create seemingly inconsistent performance.

---

# 10. Don't immediately add an index

Suppose someone tells you:

> "The query is slow. Create an index."

That's incomplete reasoning.

First ask:

```text id="x7f5n2q"
Does the predicate have a useful index?
Is the current index actually being used?
Are statistics accurate?
Is the plan appropriate?
Is the query returning too many rows?
Is the bottleneck CPU, IO, or blocking?
```

An index isn't a universal performance solution.

---

# 11. Statistics

Think of statistics as information that helps the optimizer understand data distribution.

For example:

```text id="m3w8k1"
CustomerID
```

might have:

```text
10 million rows
```

But only:

```text
1 customer → 4 million rows
remaining customers → small number of rows
```

That's highly skewed data.

The optimizer's estimates matter enormously.

---

# 12. Estimated vs actual rows

Suppose execution plan shows:

```text id="q1s6y9"
Estimated rows = 100
Actual rows    = 4,500,000
```

That's a massive discrepancy.

Now investigate:

```text id="f3d8p0"
Statistics
Predicates
Data distribution
Expressions
Parameter sensitivity
```

If the optimizer expects 100 rows but gets millions, downstream join and access-method choices can be very poor.

---

# 13. Blocking vs deadlocks

These are often confused.

## Blocking

Session A:

```text id="v6n2p4"
UPDATE
```

holds a lock.

Session B:

```text id="j9r3k7"
SELECT/UPDATE
```

waits.

```text
A
|
| lock
v
Resource
^
|
B waiting
```

Eventually A commits.

B continues.

---

# 14. Deadlock

Now:

```text id="d3k7x1"
Session A
   |
   | locks Resource 1
   |
   v
Resource 1

Session B
   |
   | locks Resource 2
   |
   v
Resource 2
```

Then:

```text id="n8p4y6"
A wants Resource 2
B wants Resource 1
```

Both wait forever.

SQL Server detects the cycle and chooses a victim.

```text
A ─────> B
^        |
|        v
+--------+
```

That's a deadlock.

---

# 15. Interview question

> **"What's the difference between blocking and deadlock?"**

Answer:

> "Blocking is when one session waits for another session to release a resource. It can be normal depending on the workload. A deadlock is a cyclic dependency where two or more sessions are waiting on each other, so the database engine must terminate one transaction to break the cycle."

Excellent.

---

# 16. How to investigate blocking

Use:

```sql id="x6m9w2"
SELECT
    r.session_id,
    r.blocking_session_id,
    r.wait_type,
    r.wait_time,
    r.wait_resource,
    DB_NAME(r.database_id) AS database_name,
    t.text
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE r.blocking_session_id <> 0
ORDER BY r.wait_time DESC;
```

Then investigate the blocker:

```text id="g7t3n5"
Who?
What application?
What SQL?
Open transaction?
How long?
What resource?
```

---

# 17. The dangerous blocker

Suppose:

```text id="f4j2k8"
Session 57
```

is blocking:

```text id="p8m3z1"
Sessions 81
82
83
84
85
```

You investigate and discover:

```text id="w6y4r2"
Application = old batch process
Transaction = open
Duration = 3 hours
```

Now you've found a likely root cause.

The solution may be:

```text
Fix transaction handling
```

rather than:

```text
Kill five blocked sessions
```

---

# 18. Wait statistics

This is where senior troubleshooting becomes much more systematic.

Ask:

> **What is SQL Server spending time waiting for?**

Conceptually:

```text id="y4n7m1"
Query
 |
 +-- CPU
 |
 +-- IO
 |
 +-- Locks
 |
 +-- Memory
 |
 +-- Network
 |
 +-- Other resources
```

Wait statistics help classify the bottleneck.

---

# 19. Example

Suppose:

```text id="k3v8p0"
CPU = 35%
Users = complaining
```

You might think:

> "CPU isn't high, so SQL isn't the problem."

Wrong.

Suppose queries are spending huge amounts of time waiting for locks.

Then:

```text id="a1x9s6"
CPU = moderate
Blocking = severe
```

The database can still feel extremely slow.

---

# 20. Performance troubleshooting hierarchy

Memorize this:

```text id="w3k6t9"
User complaint
     ↓
When did it start?
     ↓
Who is affected?
     ↓
Current workload
     ↓
CPU / IO / waits
     ↓
Blocking / deadlocks
     ↓
Top queries
     ↓
Query Store
     ↓
Execution plan
     ↓
Statistics / indexes / query design
     ↓
Root cause
     ↓
Targeted fix
     ↓
Validate
```

This is the process you should describe in senior interviews.

---

# 21. The 2 AM incident — continue

We started with:

```text
CPU = 94%
```

You find:

```text
Query = OrderSearch
CPU = extremely high
```

Query Store shows:

```text
Yesterday = 200 ms
Today = 12 sec
```

Execution plans show:

```text
Old plan = Index Seek
New plan = Large Scan
```

Now you have a strong hypothesis:

> **Plan regression.**

Next, investigate:

```text
Statistics
Data changes
Index changes
Parameter values
Plan history
```

Only after establishing the cause should you choose a remediation.

Possible remediation paths can include:

```text
Statistics correction
Index/query tuning
Query Store plan forcing where appropriate
Query rewrite
Parameter-sensitivity mitigation
Compute scaling if the workload genuinely requires it
```

---

# 22. Why "just scale up" is not enough

Suppose:

```text
Current compute
CPU = 95%
```

You double the compute.

CPU becomes:

```text
50%
```

Great?

Maybe.

But if the real problem is:

```text
Bad execution plan
```

you just spent more money to execute the same inefficient query faster.

A good DBA asks:

> **"Can I remove the unnecessary work before I buy more capacity?"**

---

# 23. When scaling IS the right answer

Scaling is appropriate when evidence shows:

```text id="k7q2m5"
Workload genuinely increased
+
Queries are reasonably optimized
+
No major blocking issue
+
No obvious plan regression
+
Resource saturation is sustained
```

For example:

```text
10 AM → 40% CPU
11 AM → 55%
12 PM → 70%
1 PM → 82%
2 PM → 95%
```

and business traffic has doubled.

That's a strong capacity-planning signal.

---

# 24. Senior interview answer

### Question:

> "Production SQL MI CPU is at 95%. What do you do?"

A strong answer:

> "I wouldn't immediately scale the instance. First I'd establish whether the CPU increase is sustained and identify the workload responsible using DMVs and Query Store. I'd examine top CPU-consuming queries, execution plans, waits, blocking, and recent changes. If I find a plan regression or inefficient query, I'd address that first. If the workload is correctly optimized but the resource demand has genuinely increased, then I'd evaluate scaling based on measured capacity requirements and validate the impact afterward."

That's exactly the mindset you want.

---

