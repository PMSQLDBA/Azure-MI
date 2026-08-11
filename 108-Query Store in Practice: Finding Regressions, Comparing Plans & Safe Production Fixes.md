# Lesson 141 — Query Store in Practice: Finding Regressions, Comparing Plans & Safe Production Fixes

Today we turn Query Store from a **concept** into a practical troubleshooting workflow.

The goal is simple:

> **When a production query becomes slower, use historical evidence to determine exactly what changed.**

---

## 1. Start with the symptom

Suppose users report:

> "The application became slow after lunch."

Don't immediately touch SQL Server configuration.

First establish:

```text
What is slow?
Which query?
When did it start?
How much slower is it?
Is it one query or many?
```

---

# 2. Identify the affected workload

Imagine the application has:

```text
Checkout
Search
Login
Reporting
Inventory
Billing
```

Users report:

```text
Checkout → slow
```

Your first goal is to isolate the database workload associated with checkout.

---

# 3. Find the query in Query Store

You can investigate queries by:

```text
Average duration
Total duration
CPU
Logical reads
Execution count
Memory-related metrics
```

Choose the dimension matching the incident.

---

# 4. Average duration can mislead

Suppose:

```text id="a6v8r2"
Query A
Average = 10 sec
Executions = 5

Query B
Average = 50 ms
Executions = 2,000,000
```

Query A looks worse per execution.

But Query B may dominate total system workload.

Therefore ask:

> **Are we optimizing the slowest query or the biggest workload consumer?**

Those aren't always the same.

---

# 5. Total resource consumption

For capacity problems, total consumption is often more useful.

For example:

```text id="j4s0m7"
Query A:
CPU = 2 hours/day

Query B:
CPU = 20 minutes/day
```

Even if Query B has a worse average execution time, Query A may be the bigger CPU problem.

---

# 6. Establish a baseline

Before changing anything, record:

```text id="p2w8k5"
Query
Plan
Execution count
Average duration
Total duration
CPU
Logical reads
Logical writes
```

You want a baseline that allows you to say:

> "This change improved the workload by X."

---

# 7. Identify the regression window

Suppose:

```text id="m8r3x1"
08:00 → 100 ms
09:00 → 110 ms
10:00 → 95 ms
11:00 → 105 ms
12:00 → 8,000 ms
```

You have a clear regression around:

```text id="n5v1z9"
12:00
```

Now investigate what changed around that time.

---

# 8. Compare plans

Suppose Query Store shows:

```text id="q8s2c4"
Plan 101:
Average = 120 ms

Plan 205:
Average = 8 sec
```

This is a strong clue.

But remember:

> **Correlation is not yet causation.**

You still need to understand the plan difference.

---

# 9. Plan comparison checklist

Compare:

```text id="c0j6k8"
Access methods
Join algorithms
Join order
Estimated rows
Actual rows
Indexes
Parallelism
Sorts
Aggregations
Memory grants
Spills
```

You are looking for the mechanism that explains the performance change.

---

# 10. Example: seek becomes scan

Old plan:

```text id="n6v3a8"
Index Seek
      ↓
Nested Loops
```

New plan:

```text id="z7x1q4"
Index Scan
      ↓
Hash Match
```

This isn't automatically bad.

Ask:

> Why did the optimizer decide that scanning was cheaper?

---

# 11. Compare cardinality estimates

Suppose:

```text id="t3p8m1"
Old plan:
Estimated = 100,000
Actual    = 95,000

New plan:
Estimated = 100
Actual    = 5,000,000
```

Now the new plan's estimate is obviously suspicious.

You have a potential root cause.

---

# 12. Follow the first major error

Remember Lesson 138.

Don't simply inspect the operator with the highest duration.

Find:

> **The first major divergence between estimated and actual rows.**

For example:

```text id="q1f5z7"
Operator A:
Estimate 10,000
Actual 11,000

Operator B:
Estimate 8,000
Actual 9,000

Operator C:
Estimate 7,000
Actual 4,000,000   ← investigate here
```

---

# 13. Statistics investigation

Once you identify an estimate problem, inspect:

```text id="h8w3r6"
Statistics age
Modification count
Rows sampled
Distribution
Histogram
Column correlation
```

Ask:

> Does the statistical information reasonably represent today's data?

---

# 14. Query changes

Not every regression is caused by SQL Server.

Maybe the application deployment changed:

```sql id="e9r2w6"
WHERE CustomerId = @CustomerId
```

into:

```sql id="s7c1x4"
WHERE CustomerId = @CustomerId
  AND IsArchived = 0
```

A small query change can produce a completely different plan.

Always compare the query text as well as the plan.

---

# 15. Parameter sensitivity

Suppose the query is:

```sql id="k2f9m7"
SELECT ...
FROM Orders
WHERE CustomerId = @CustomerId;
```

Customer A:

```text id="h5q8v2"
10 rows
```

Customer B:

```text id="x3m7n1"
20 million rows
```

A plan optimized for Customer A may be terrible for Customer B.

Query Store can reveal unusual performance patterns around such queries.

---

# 16. Data growth

Maybe nothing changed in the query or indexes.

The data did.

Example:

```text id="w1z6k3"
January:
10 million orders

August:
500 million orders
```

A plan that worked well in January may no longer be appropriate.

This is a critical distinction:

```text id="6r2j9p"
Plan regression
```

versus:

```text id="p0c4x8"
Workload/data evolution
```

---

# 17. Index changes

Check whether someone recently:

```text id="q6s3v1"
Created an index
Dropped an index
Disabled an index
Changed an index
Changed included columns
Changed partitioning
```

A seemingly unrelated index change can alter plan selection.

---

# 18. Statistics changes

Similarly, investigate:

```text id="z8k2r5"
Statistics update
```

near the regression window.

Sometimes a new statistical picture causes a plan change.

That doesn't necessarily mean the update was wrong.

It means you should understand why the new plan performs differently.

---

# 19. Compatibility-level changes

If the regression occurred after:

```text id="r3v7m9"
Database upgrade
```

or:

```text id="u4x6q1"
Compatibility-level change
```

investigate optimizer/CE behavior.

A different optimizer model can produce:

```text id="j2n8c5"
Different estimates
→ Different cost
→ Different plan
```

---

# 20. Memory grant changes

Compare plans for:

```text id="v8x4m2"
Requested memory
Granted memory
Used memory
Spills
```

A new plan may request substantially more memory.

Or it may receive too little and spill to TempDB.

---

# 21. Spills

Suppose a sort or hash operation spills.

Conceptually:

```text id="d4p1s7"
Memory insufficient
      ↓
TempDB
      ↓
Additional I/O
      ↓
Slower query
```

A cardinality-estimation error can contribute to an inappropriate memory grant.

---

# 22. Same query, same plan, slower execution

This is one of the most important scenarios.

Suppose Query Store shows:

```text id="n9c2v5"
Plan = unchanged
```

but:

```text id="x4m7j8"
Duration:
100 ms → 5 sec
```

Don't blame the plan yet.

Investigate:

```text id="u1f6r3"
Blocking
CPU pressure
I/O pressure
Memory pressure
Concurrency
Network/client behavior
Data volume
```

---

# 23. Blocking example

Query:

```text id="a2r5n8"
Same plan
Same estimated rows
Same logical work
```

But:

```text id="q7v0c1"
Duration = 30 sec
CPU = 200 ms
```

That strongly suggests much of the time was spent waiting rather than executing CPU work.

Blocking is one possible explanation.

---

# 24. CPU-bound example

Now:

```text id="s5k1m9"
Duration = 30 sec
CPU = 28 sec
```

This is a fundamentally different problem.

Now investigate:

```text id="e7v3q6"
Plan efficiency
CPU-intensive operators
Parallelism
Scans
Expressions
Concurrency
```

---

# 25. I/O-bound example

Suppose:

```text id="r2n8f4"
CPU = 500 ms
Duration = 20 sec
Logical reads = enormous
```

Potential areas include:

```text id="t0k5m2"
Large scans
Poor indexes
Storage pressure
Cache misses
```

Again, the numbers tell the story.

---

# 26. Query Store isn't a substitute for live monitoring

Query Store tells you historical workload behavior.

For an active incident you may also need:

```text id="v5j1x8"
DMVs
Blocking information
Wait information
Active requests
Resource utilization
```

Think:

```text id="a8k3p6"
Historical evidence
+
Current evidence
=
Complete investigation
```

---

# 27. Plan forcing as containment

Suppose:

```text id="f1q6s9"
Plan A = good
Plan B = terrible
```

Production is currently suffering.

If Plan A is known to be safe and appropriate, plan forcing can potentially restore performance quickly.

But treat it as:

```text id="m4z7c2"
Emergency containment
```

not:

```text id="w8r3n5"
Root-cause resolution
```

---

# 28. Document every forced plan

If you force a plan, record:

```text id="c6y1k9"
Why it was forced
Which plan was chosen
When it was forced
What incident triggered it
What the expected benefit is
What follow-up investigation is required
```

Otherwise six months later someone may ask:

> "Why is this plan forced?"

and nobody knows.

---

# 29. Monitor forced plans

After forcing:

```text id="x3q7m1"
Did performance improve?
Is CPU lower?
Are reads lower?
Is duration stable?
Did data distribution change?
```

A forced plan should be treated as something that needs observation.

---

# 30. Root-cause fix examples

If the cause is stale statistics:

```text id="k7m2p4"
Targeted statistics strategy
```

If the cause is missing index:

```text id="r1v8c3"
Appropriate index
```

If the cause is parameter sensitivity:

```text id="n5q9x2"
Appropriate parameter-sensitive strategy
```

If the cause is a bad query expression:

```text id="d6m1s8"
Rewrite/query design
```

If the cause is blocking:

```text id="u3f7j5"
Transaction/concurrency fix
```

The solution depends on the cause.

---

# 31. Don't optimize the symptom

Suppose the query is slow because it is blocked.

Adding an index may:

```text id="e8q4v1"
Change nothing
```

or even make the workload more complicated.

If the real issue is:

```text id="y2k6m9"
Long-running transaction
```

fix the transaction behavior.

---

# 32. Production-safe testing

When possible:

```text id="c5n8r2"
Production evidence
       ↓
Reproduce safely
       ↓
Test candidate fix
       ↓
Compare metrics
       ↓
Deploy
```

Don't experiment blindly on a busy production server.

---

# 33. A/B comparison

For a candidate change, compare:

```text id="h3j7w0"
Before
vs
After
```

Metrics:

```text id="p9m4x2"
Duration
CPU
Reads
Writes
Execution count
Memory
Spills
```

You want objective evidence.

---

# 34. Example

Before:

```text id="z7r2c5"
Avg duration = 4,200 ms
CPU          = 3,800 ms
Reads        = 2,000,000
```

After:

```text id="q1m6v8"
Avg duration = 300 ms
CPU          = 250 ms
Reads        = 40,000
```

That is a strong improvement across multiple dimensions.

---

# 35. Beware of averages

Suppose:

```text id="s4x9n1"
Before:
Average = 100 ms
```

After:

```text id="d8k3q7"
Average = 90 ms
```

Looks better.

But perhaps:

```text id="m1v6c4"
P95:
Before = 2 sec
After  = 10 sec
```

The average improved while the worst user experience became much worse.

For latency-sensitive applications, distribution matters.

---

# 36. Percentiles

When available, think beyond averages:

```text id="f7q2k5"
P50 → typical execution
P95 → slow tail
P99 → extreme tail
```

A query can have a good average but terrible tail latency.

---

# 37. Why tail latency matters

Imagine:

```text id="g8n3w6"
99% of requests = 50 ms
1% of requests  = 30 sec
```

Average may still look acceptable.

But that 1% can represent thousands of users in a large application.

Production engineering often cares heavily about the tail.

---

# 38. Query Store and release validation

Before deployment:

```text id="r5c1x8"
Capture baseline
```

After deployment:

```text id="n9v4k2"
Compare workload
```

Look for:

```text id="j7m2q6"
New expensive queries
Regressions
Plan changes
CPU increases
Read increases
```

This creates a feedback loop between development and database operations.

---

# 39. Query Store and database upgrades

Before upgrading:

```text id="c4x8p1"
Capture workload baseline
```

After upgrading:

```text id="v6m3z9"
Compare plans and runtime
```

This is particularly important because optimizer behavior can change.

---

# 40. Query Store and index changes

Suppose you add:

```text id="k2r7m5"
IX_Orders_CustomerId
```

Afterward, Query Store can help determine whether:

```text id="w8q1c3"
Reads decreased
CPU decreased
Duration decreased
```

across the workload.

The index should be judged by workload-level impact, not merely whether one execution got faster.

---

# 41. Query Store and statistics changes

Likewise, if you change a statistics strategy:

```text id="m3v8x0"
Before
↓
Statistics change
↓
After
```

Query Store can provide evidence about whether important queries improved.

---

# 42. Query Store and application teams

A DBA can use Query Store to communicate with developers in concrete terms:

Instead of:

> "Your query is inefficient."

say:

> "After deployment, this query's average CPU increased from 20 ms to 180 ms, and its plan changed from an index seek to a scan."

That's actionable.

---

# 43. Query Store and management

You can summarize impact:

```text id="u7p2m9"
CPU:
+35%

Logical reads:
+50%

Average duration:
+20%

Top regression:
GetCustomerOrders
```

This turns database performance into measurable business-impact evidence.

---

# 44. A complete production investigation

Let's put everything together.

### Symptom

```text id="r1c5n8"
Application endpoint is slow.
```

### Step 1

Query Store identifies:

```text id="p7m3x2"
GetCustomerOrders
```

### Step 2

Historical comparison shows:

```text id="j4v8q1"
100 ms → 5 sec
```

### Step 3

Plan changed:

```text id="c9m2w6"
Seek → Scan
```

### Step 4

Estimate changed:

```text id="z5r1k7"
100 rows estimated
5,000,000 actual
```

### Step 5

Statistics inspection reveals:

```text id="n8x4p3"
Highly skewed CustomerId
```

### Step 6

Parameter sensitivity is confirmed.

Now you have a plausible root cause.

---

# 45. The fix

Depending on the SQL Server version and workload, possible solutions might include:

```text id="v3m7q9"
Parameter-sensitive optimization
Query rewrite
Appropriate indexing
Targeted statistics
Query hint
Query Store hint
Plan forcing
```

But choose based on the actual cause.

---

# 46. Verify

After implementing the fix:

```text id="a9k2x5"
Duration ↓
CPU ↓
Reads ↓
Plan stable
```

Then continue monitoring.

---

# 47. Remove temporary containment

If you used:

```text id="w6p1r8"
Plan forcing
```

or another temporary workaround, revisit it after the root cause is fixed.

A mature environment avoids accumulating permanent emergency fixes.

---

# 48. Query Store anti-patterns

Avoid:

### 1. Looking only at average duration

You may miss workload volume.

### 2. Looking only at the current plan

You lose historical context.

### 3. Forcing every previous good plan

You create technical debt.

### 4. Ignoring data changes

A plan may have changed for a legitimate reason.

### 5. Ignoring waits

A query can be slow without having a bad plan.

---

# 49. The golden workflow

Memorize this:

```text id="y4k7p2"
SYMPTOM
   ↓
QUERY STORE
   ↓
BASELINE
   ↓
REGRESSION WINDOW
   ↓
PLAN COMPARISON
   ↓
ESTIMATE COMPARISON
   ↓
ROOT-CAUSE ANALYSIS
   ↓
TARGETED FIX
   ↓
MEASURE
   ↓
MONITOR
```

This is a repeatable production troubleshooting process.

---

# 50. Expert mental model

Query Store answers three critical questions:

### 1. What?

```text
Which query is affected?
```

### 2. When?

```text
When did performance change?
```

### 3. How?

```text
Which plan/runtime characteristics changed?
```

Then your other SQL Server tools answer:

> **Why?**

---

# 51. Lesson 141 takeaway

The most important lesson is:

> **Performance troubleshooting should be evidence-driven and historical, not based solely on the current execution plan.**

Use Query Store to establish:

```text id="b5m8r1"
WHAT changed
WHEN it changed
HOW much it changed
WHICH plan changed
```

Then use execution plans, statistics, waits, DMVs, and workload analysis to determine:

```text id="q2v7n4"
WHY it changed
```

**Next:** **Lesson 142**
**Final:** **Lesson 150**
