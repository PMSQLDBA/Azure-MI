# Lesson 71 — Azure SQL Managed Instance Performance Troubleshooting

This is one of the most important lessons for a SQL DBA.

In production, someone rarely tells you:

> "Query 742 has a plan regression caused by a cardinality-estimation problem."

They usually tell you:

> **"SQL is slow."**

Your job is to turn that vague statement into a measurable root cause.

---

# 1. First rule: "SQL is slow" is not a diagnosis

When an application is slow, many different things could be happening:

```text
Application
    |
    v
SQL connection
    |
    +--> Connection delay
    |
    +--> Blocking
    |
    +--> CPU pressure
    |
    +--> IO latency
    |
    +--> Memory/resource pressure
    |
    +--> Bad execution plan
    |
    +--> Statistics problem
    |
    +--> Deadlock/retries
    |
    +--> Application problem
```

Never start by randomly rebuilding indexes.

Start by **classifying the problem**.

---

# 2. The performance troubleshooting framework

Use this sequence:

```text
1. Is the problem real?
        ↓
2. When did it start?
        ↓
3. Is it all queries or specific queries?
        ↓
4. Is SQL waiting or consuming resources?
        ↓
5. Which resource?
        ↓
6. Which query/session?
        ↓
7. Why is that query behaving differently?
        ↓
8. What changed?
        ↓
9. Fix
        ↓
10. Validate
```

This is your production framework.

---

# 3. Scenario #1 — CPU suddenly reaches 100%

At 10:05 AM:

```text
CPU = 35%
```

At 10:15:

```text
CPU = 98%
```

Application latency increases.

First question:

> **What changed at 10:15?**

Don't immediately resize the MI.

Investigate workload first.

---

# 4. CPU investigation

Look for:

```text
Top CPU-consuming queries
Query execution count
Query duration
Parallelism
Plan changes
New deployment
Scheduled jobs
Unexpected workload
```

Conceptually:

```text
CPU 100%
   |
   +-- Query A = 60%
   +-- Query B = 20%
   +-- Query C = 10%
   +-- Everything else = 10%
```

Now you have a target.

---

# 5. Example

Suppose Query A normally consumes:

```text
50 ms CPU/execution
```

Today:

```text
500 ms CPU/execution
```

and runs:

```text
20,000 times/hour
```

Even a small per-execution increase becomes huge.

This is why:

> **CPU problems can be caused by query efficiency OR workload volume.**

---

# 6. Two ways CPU can increase

### Case A — Query became inefficient

```text
Same workload
      ↓
More CPU per execution
      ↓
CPU ↑
```

### Case B — Workload increased

```text
Same query
      ↓
10x executions
      ↓
CPU ↑
```

The fixes are completely different.

---

# 7. Scenario #2 — Query went from 200 ms to 30 seconds

This is classic **query regression**.

Yesterday:

```text id="n1m8r4"
Query
200 ms
```

Today:

```text id="z6q2p7"
Query
30 sec
```

First question:

> **Did the execution plan change?**

This is where Query Store becomes extremely valuable.

---

# 8. Query Store comparison

Conceptually:

```text id="c7x4m9"
Query 742

Plan 101
   ↓
200 ms

Plan 204
   ↓
30 sec
```

Now you have a strong hypothesis:

> **Plan regression.**

---

# 9. Why execution plans matter

Imagine the query needs:

```text id="p5r8n2"
100 rows
```

Plan A:

```text id="w3m7k1"
Index Seek
   ↓
100 rows
```

Very efficient.

But Plan B estimates:

```text id="q9v2c6"
1,000,000 rows
```

and chooses:

```text id="h4x8m3"
Large scan
```

Now SQL performs much more work.

---

# 10. Estimated vs actual rows

One of the most useful things to inspect:

```text id="j2n6p8"
Estimated rows
vs
Actual rows
```

Example:

```text id="r5m1x9"
Estimated = 10
Actual    = 500,000
```

That's a huge mismatch.

Possible causes include:

```text id="c8q4v2"
Stale statistics
Data distribution change
Parameter-sensitive behavior
Correlated predicates
Poor cardinality estimation
```

---

# 11. Scenario #3 — Blocking

Now consider:

```text id="s7x3m5"
Application request
       |
       v
SELECT
       |
      WAIT
```

Another session:

```text id="k9p2r8"
UPDATE
   |
Transaction still open
```

The SELECT is not necessarily slow because the query itself is inefficient.

It is **waiting for another transaction**.

---

# 12. Blocking chain

Example:

```text id="m4r8q1"
Session 51
   |
   | blocks
   v
Session 72
   |
   | blocks
   v
Session 89
   |
   | blocks
   v
Session 101
```

The most important session may be:

```text id="n6x2p7"
Session 51
```

because it is the **head blocker**.

---

# 13. Why killing the last session is wrong

Suppose:

```text id="v8m3c6"
51 → 72 → 89 → 101
```

You kill session 101.

What happens?

```text id="q2r7n9"
51 → 72 → 89
```

The problem remains.

You need to understand the root blocker and whether terminating it is safe.

---

# 14. Head blocker investigation

Ask:

```text id="j5x9p3"
Who owns the transaction?
What SQL is executing?
How long has it been open?
Which application?
Which user?
Why is the transaction open?
Is it safe to terminate?
```

Potential causes:

```text id="c1m8r4"
Application bug
Long transaction
Missing index
Large update
Open connection
Batch process
Human session
```

---

# 15. Scenario #4 — Deadlocks

Blocking is:

```text
A → B
```

Deadlock is:

```text
A → B
↑   ↓
+---+
```

Two or more sessions are waiting on each other.

SQL Server detects the cycle and chooses a victim so the system can continue.

---

# 16. Example deadlock

Transaction A:

```text id="w8q2n6"
Locks Customer
↓
Needs Order
```

Transaction B:

```text id="p4m7x1"
Locks Order
↓
Needs Customer
```

Result:

```text id="z9c3r8"
Customer ← A
   ↑       |
   |       ↓
   B → Order
```

Cycle.

---

# 17. Deadlock doesn't always mean "SQL is broken"

Often the underlying problem is application design.

For example:

Transaction 1:

```text id="n2x6m9"
UPDATE Customer
UPDATE Order
```

Transaction 2:

```text id="j7r4p3"
UPDATE Order
UPDATE Customer
```

Different lock acquisition order creates risk.

A better consistent order can eliminate the deadlock.

---

# 18. Scenario #5 — IO latency

Suppose CPU is only:

```text id="a5v9q2"
40%
```

but queries are slow.

Don't conclude:

> "CPU isn't high, so SQL isn't the problem."

Investigate IO.

You might see:

```text id="k8m3r6"
Read latency ↑
Write latency ↑
Query duration ↑
```

A query may be waiting on storage-related work.

---

# 19. CPU vs IO

Simple mental model:

### CPU problem

```text id="p1x7c4"
SQL is actively doing too much computation
```

### IO problem

```text id="q6m2n8"
SQL is waiting for data movement/storage operations
```

### Blocking problem

```text id="v4r9k1"
SQL is waiting for another transaction/session
```

These require different troubleshooting paths.

---

# 20. Scenario #6 — Memory/resource pressure

Suppose queries suddenly become slower after workload increases.

Potentially investigate:

```text id="h7m3q9"
Memory pressure
Large queries
Poor plans
Excessive grants
Concurrent workload
```

One query may request a very large memory grant.

Example:

```text id="s5n8x2"
Query A
Memory grant = huge

Query B
waiting

Query C
waiting
```

This can create a cascading performance problem.

---

# 21. Scenario #7 — Parameter-sensitive query

This is a very important real-world problem.

Suppose:

```sql id="6axc9u"
SELECT *
FROM Orders
WHERE CustomerID = @CustomerID;
```

Customer A:

```text id="p3n7m2"
10 rows
```

Customer B:

```text id="w8q1x5"
5,000,000 rows
```

The optimizer may choose a plan that works very well for one parameter but poorly for another.

---

# 22. Example

For Customer A:

```text id="f6m2r9"
Index Seek
```

Excellent.

For Customer B:

```text id="k4x8p1"
Large data access
```

The same cached plan may not be optimal.

Result:

```text id="c9q3v7"
Customer A → fast
Customer B → slow
```

This is why you can hear:

> "The query is fast for me."

while another user says:

> "It's incredibly slow."

---

# 23. Why parameter sensitivity matters in production

You may observe:

```text id="m7r2n5"
Same query
Same database
Different execution time
```

That should make you investigate:

```text id="q4x8c1"
Parameters
Execution plans
Data distribution
Plan reuse
Statistics
```

Don't automatically blame infrastructure.

---

# 24. Scenario #8 — Statistics problem

Suppose a table changes from:

```text id="z3p7m9"
100K rows
```

to:

```text id="h6n2x4"
100M rows
```

The optimizer needs accurate information about data distribution.

If statistics are stale or not representative:

```text id="r8c1m5"
Estimated rows
      ≠
Actual rows
```

This can lead to poor plan choices.

---

# 25. Don't confuse statistics with indexes

A common beginner answer:

> "Query is slow → create an index."

Sometimes correct.

Sometimes completely wrong.

The problem could be:

```text id="w2q9k6"
Blocking
Bad plan
Statistics
Parameter sensitivity
IO
CPU
Memory
Application behavior
```

You need evidence.

---

# 26. Scenario #9 — Query slow only during business hours

This is extremely common.

At:

```text id="s5m8r1"
2 AM → 200 ms
```

At:

```text id="c7n3x9"
10 AM → 20 sec
```

Why?

Potentially:

```text id="p4q8m2"
Higher concurrency
Blocking
CPU contention
Different query mix
Memory pressure
More IO
Different parameters
```

The query itself hasn't necessarily changed.

The **environment around it** changed.

---

# 27. Performance timeline

Build a timeline.

Example:

```text id="j9x3m6"
09:55
Normal

10:00
Application deployment

10:03
CPU ↑

10:05
Query duration ↑

10:06
Blocking ↑

10:07
Application timeouts
```

Now ask:

> What changed at 10:00?

This is often more useful than staring at today's CPU graph.

---

# 28. Query Store as your historical memory

Suppose you have:

```text id="v6m2q8"
Monday
Query = 200 ms

Tuesday
Query = 210 ms

Wednesday
Query = 220 ms

Thursday
Query = 30 sec
```

Query Store helps establish that this is a **regression**, not a permanently slow query.

That's a huge distinction.

---

# 29. Performance troubleshooting example

Production alert:

```text id="n4p8x2"
Application latency > 5 sec
```

You investigate.

### CPU

```text id="q7m3r9"
92%
```

### Blocking

```text id="x5c1n8"
Minimal
```

### Deadlocks

```text id="m2v9k4"
None
```

### Query Store

```text id="j8r3p6"
Query 742
Plan changed
```

### Query performance

```text id="s1n7x5"
Previous:
300 ms

Current:
18 sec
```

### Execution plan

```text id="k6m4q2"
Index Seek
       ↓
became
       ↓
Large scan
```

Now you have a very strong diagnosis:

> **A plan regression caused by the query using a less efficient access strategy.**

---

# 30. What NOT to do first

Don't immediately:

```text id="w8x2m5"
Restart SQL
Rebuild all indexes
Update every statistic
Kill random sessions
Scale compute
Clear caches
```

These can hide evidence or create new problems.

---

# 31. The "five questions"

When SQL is slow, ask:

### 1. What is slow?

```text
One query?
All queries?
Application?
Connection?
```

### 2. When did it become slow?

```text
Exact time
```

### 3. What changed?

```text
Deployment?
Data volume?
Plan?
Workload?
Configuration?
```

### 4. What is SQL doing?

```text
CPU?
IO?
Waiting?
Blocking?
Memory?
```

### 5. What is the evidence?

```text
Query Store
Execution plan
Wait information
Monitoring
Logs
Application telemetry
```

---

# 32. Production incident — full walkthrough

Let's simulate one.

### 10:00 AM

Application normal.

```text
Latency = 250 ms
```

### 10:12 AM

New application release.

### 10:15 AM

Latency:

```text
8 seconds
```

---

## Step 1 — Check whether everything is slow

Only one API is affected.

Therefore:

```text
Not necessarily an MI-wide resource issue.
```

---

## Step 2 — Identify SQL query

Application telemetry identifies:

```text
Query 742
```

---

## Step 3 — Query Store

Previous:

```text
300 ms
```

Current:

```text
7.8 sec
```

---

## Step 4 — Plan comparison

Old:

```text
Index Seek
```

New:

```text
Large Scan
```

---

## Step 5 — Why did the plan change?

Investigate:

```text
Statistics
Parameters
Indexes
Query text
Deployment changes
```

---

## Step 6 — Stabilize

Depending on the specific cause and available supported features, you may consider a controlled plan-stabilization approach, query/application correction, index/statistics remediation, or rollback.

The key point:

> **Stabilize first, then perform deeper root-cause analysis.**

---

# 33. Stabilization vs root cause

These are different.

### Stabilization

Goal:

> Get production healthy.

Example:

```text
Reduce workload
Rollback problematic release
Apply an appropriate plan correction
Resolve blocking
Scale resources if justified
```

### Root cause

Goal:

> Prevent recurrence.

Example:

```text
Why did plan change?
Why were statistics inaccurate?
Why did application change parameters?
Why was the index missing?
Why did transaction stay open?
```

A temporary workaround is not the root cause.

---

# 34. Performance incident lifecycle

Memorize:

```text id="e2v7k1"
DETECT
  ↓
CLASSIFY
  ↓
MEASURE
  ↓
IDENTIFY
  ↓
STABILIZE
  ↓
ROOT CAUSE
  ↓
FIX
  ↓
VALIDATE
  ↓
PREVENT
```

This is a professional troubleshooting process.

---

# 35. Interview question

### "SQL MI CPU is at 100%. What do you do?"

Strong answer:

> "I would first determine whether the CPU increase is sustained and whether the impact is application-visible. Then I'd identify the top CPU-consuming queries and compare their current behavior with historical Query Store data. I'd determine whether the increase is caused by higher execution volume, increased CPU per execution, a plan regression, or a broader workload change. I'd also check blocking, waits, recent deployments, scheduled jobs, and resource trends before deciding whether query remediation or scaling is appropriate."

That's much stronger than:

> "Increase the CPU."

---

# 36. Interview question

### "A query is suddenly slow. What do you check?"

Answer:

```text
Query Store
Execution plan
Waits
Blocking
CPU
IO
Statistics
Parameter sensitivity
Recent changes
Data volume
```

Then explain:

> "I'd compare current behavior with the historical baseline to determine what changed."

---

# 37. Interview question

### "What is the difference between blocking and a slow query?"

A slow query may be:

```text
Doing expensive work
```

A blocked query may be:

```text
Waiting for another session
```

Example:

```text
Slow:
SELECT scans 500M rows

Blocked:
SELECT is waiting for an UPDATE transaction
```

Same user-visible symptom:

> "The page is slow."

Completely different root cause.

---

# 38. Your performance cheat sheet

When someone says:

> **SQL is slow**

think:

```text id="b5q8m3"
                 SQL SLOW
                     |
       +-------------+-------------+
       |             |             |
      CPU           IO          BLOCKING
       |             |             |
    Queries       Reads       Transactions
       |
    Plans
       |
 Statistics
       |
 Parameters
       |
 Workload
```

Then investigate:

```text id="r6n2x9"
Query Store
Execution plans
Waits
Blocking
Resource metrics
Application telemetry
Timeline of changes
```

---

# 39. Most important lesson

Never troubleshoot SQL performance by guessing.

Don't say:

> "It must be indexes."

Don't say:

> "CPU is high, so increase compute."

Don't say:

> "The database is slow."

Instead say:

> **"The application latency increased at 10:15. Query 742's average duration increased from 300 ms to 18 seconds after a plan change. CPU increased concurrently, while blocking remained low. I'm investigating why the optimizer selected the new plan and will stabilize production while determining the underlying cause."**

That is how a senior DBA communicates.

---

## Next — Lesson 72

Next we'll go deeper into **Execution Plans + Query Store + Wait Statistics**, including a real query-performance investigation from start to finish.

We'll cover:

* how to read an execution plan
* Index Seek vs Index Scan
* Key Lookup
* joins
* Sort operators
* Hash Match
* Nested Loops
* estimated vs actual rows
* memory grants
* expensive operators
* wait statistics
* `PAGEIOLATCH`
* `LCK_M_*`
* `CXPACKET` / `CXCONSUMER`
* `RESOURCE_SEMAPHORE`
* and how to connect **waits → query → execution plan → root cause**.
