# Lesson 140 — Query Store Deep Dive: Capturing Queries, Plans, Regressions, Runtime Statistics & Production Troubleshooting

Today we move from **statistics** into one of the most important production troubleshooting tools in modern SQL Server:

> **Query Store**

If execution plans are the evidence, Query Store is your **historical evidence locker**.

---

## 1. Why Query Store exists

Without Query Store, troubleshooting a slow query can be difficult because you may only see the **current** execution plan.

But production problems often look like:

```text
Monday:
Query = 2 seconds
Plan A

Tuesday:
Query = 2 seconds
Plan A

Wednesday:
Query = 45 seconds
Plan B
```

If you inspect only Wednesday's plan, you may not immediately know that the query used to perform well.

Query Store preserves historical information.

---

# 2. What Query Store captures

At a high level, Query Store can retain information about:

```text
Query text
Query identity
Execution plans
Runtime statistics
Plan history
Compilation/runtime information
```

This lets you investigate:

```text
What changed?
When did it change?
Which plan was used?
How did performance change?
```

---

# 3. The fundamental Query Store model

Think:

```text
Query
  ↓
Query Store identifies it
  ↓
Plans
  ↓
Runtime statistics
  ↓
Historical performance
```

This gives you a timeline rather than a single snapshot.

---

# 4. Query Store vs execution plan

An execution plan tells you:

> "How is SQL Server executing this query?"

Query Store can help answer:

> "How has SQL Server executed this query over time?"

That historical dimension is enormously valuable.

---

# 5. Why historical information matters

Imagine:

```text
Current plan:
Hash Match
```

You might think:

> "The Hash Match is causing the problem."

But Query Store reveals:

```text
Previous plan:
Nested Loops

Previous average duration:
800 ms

Current average duration:
25 sec
```

Now you know the important event:

> **A plan change occurred.**

That changes your investigation completely.

---

# 6. Query Store and performance regression

A regression is typically:

```text
Same query
+
Performance becomes worse
```

For example:

```text
Before:
CPU = 500 ms
Duration = 700 ms

After:
CPU = 8,000 ms
Duration = 12 sec
```

Query Store can help correlate that regression with plan changes.

---

# 7. A plan regression example

Suppose Query Store shows:

```text
Plan 101:
Avg duration = 400 ms
Executions = 100,000

Plan 205:
Avg duration = 9,000 ms
Executions = 10,000
```

You now have strong evidence that Plan 205 deserves investigation.

But don't automatically force Plan 101.

First understand **why** Plan 205 was selected.

---

# 8. Why the new plan may actually be correct

Imagine data changed:

```text
Orders:
10 million → 500 million rows
```

The old plan may no longer be appropriate.

So:

```text
Old plan ≠ automatically good
New plan ≠ automatically bad
```

The question is:

> **Which plan is appropriate for the current workload and data distribution?**

---

# 9. Query Store runtime statistics

Query Store can aggregate runtime information such as:

```text
Execution count
Duration
CPU time
Logical reads
Logical writes
Memory-related metrics
```

The exact available metrics depend on SQL Server version and configuration.

This allows you to identify expensive queries by different dimensions.

---

# 10. Duration vs CPU

Suppose:

```text id="j0t5x4"
Duration = 30 sec
CPU      = 500 ms
```

The query spent most of its time waiting.

Possible areas:

```text
Blocking
I/O
Memory
External waits
Concurrency
```

Now consider:

```text id="4q7f8m"
Duration = 30 sec
CPU      = 29 sec
```

That's very different.

The query is consuming substantial CPU.

---

# 11. Logical reads

Suppose:

```text id="x9m3p7"
Logical reads = 50,000,000
```

That immediately tells you the query is touching a tremendous amount of buffer-cache data.

Possible causes include:

```text
Missing/ineffective index
Poor predicate selectivity
Large scan
Bad join strategy
Incorrect cardinality estimate
```

Again, Query Store gives you historical evidence.

---

# 12. Execution count matters

Consider two queries:

```text
Query A:
Duration = 10 seconds
Executions = 1

Query B:
Duration = 100 ms
Executions = 1,000,000
```

Query B may consume far more total resources.

Approximate total duration:

```text
A = 10 sec

B = 100 ms × 1,000,000
  = 100,000 sec
```

So don't rank queries only by average duration.

---

# 13. Average vs total

These are different questions.

### Average duration

> "How expensive is one execution?"

### Total duration

> "How much time did this query consume across all executions?"

Both can be useful.

---

# 14. Example

```text
Query A
Average = 10 seconds
Executions = 10

Query B
Average = 100 ms
Executions = 1,000,000
```

Total execution time:

```text
A ≈ 100 sec

B ≈ 100,000 sec
```

Query B may be the much bigger workload problem.

---

# 15. CPU ranking

You can similarly investigate:

```text
Top CPU-consuming queries
```

This can reveal queries that are individually fast but executed extremely frequently.

For example:

```text
5 ms × 10 million executions
```

can become a major CPU consumer.

---

# 16. Read-heavy workloads

For data-access problems, ranking by:

```text
Total logical reads
```

can reveal queries generating enormous buffer-cache activity.

This is particularly useful for identifying:

```text
Repeated scans
Inefficient lookups
Large joins
Poor filtering
```

---

# 17. Query Store as a baseline

Before changing anything, establish:

```text
Current average duration
Current CPU
Current reads
Execution count
Current plan
Historical plan
```

Then make your change.

Afterward:

```text
Compare
```

This turns tuning into measurement rather than opinion.

---

# 18. The "before and after" methodology

Use:

```text
Baseline
   ↓
Change
   ↓
Observe
   ↓
Compare
```

For example:

```text
Before:
CPU       = 2,000 ms
Reads     = 500,000
Duration  = 3 sec

After:
CPU       = 400 ms
Reads     = 20,000
Duration  = 500 ms
```

That's compelling evidence.

---

# 19. Query Store and plan history

Suppose a query has:

```text
Plan A
Plan B
Plan C
```

Query Store can help you determine:

```text
When each plan appeared
How frequently each was used
How each performed
```

This is extremely useful for plan instability.

---

# 20. Plan instability

A query may alternate between plans:

```text
A → B → A → C → B → A
```

Performance may therefore fluctuate.

This is often more difficult to diagnose than a query that is simply always slow.

---

# 21. Why plan instability matters

A query could produce:

```text
Fast execution
Fast execution
Slow execution
Fast execution
Very slow execution
```

Users may describe this as:

> "The database is randomly slow."

But the underlying problem could be:

```text
Plan variability
```

---

# 22. Parameter sensitivity

Query Store can be particularly useful when investigating parameter-sensitive behavior.

Imagine:

```text
@CustomerId = 1
→ 5 rows

@CustomerId = 999
→ 20,000,000 rows
```

One plan may not be ideal for both.

Query Store can reveal different execution patterns and plans associated with the query.

---

# 23. Query Store doesn't replace the actual plan

Query Store is a historical tool.

When you identify a problematic plan, you still need to inspect the execution plan itself.

Think:

```text
Query Store
    ↓
Find the suspect
    ↓
Execution plan
    ↓
Understand the mechanism
```

---

# 24. Finding the expensive query

A practical workflow:

```text
1. Identify workload symptom
2. Open Query Store
3. Rank queries by relevant metric
4. Find regression or outlier
5. Compare plans
6. Inspect estimates
7. Investigate root cause
8. Test fix
```

---

# 25. What should you rank by?

Depending on the problem:

```text
CPU problem
→ Total CPU

I/O problem
→ Total logical reads

Slow individual executions
→ Average duration

High workload volume
→ Execution count

Overall workload impact
→ Total duration
```

Choose the metric that matches the problem.

---

# 26. Query Store isn't only for emergencies

It is also useful for:

```text
Capacity planning
Performance baselines
Release validation
Index changes
Statistics changes
Database upgrades
Compatibility-level changes
```

It turns performance into something you can observe historically.

---

# 27. Query Store and deployments

Suppose you deploy a new application version:

```text
Version 1.0
```

then:

```text
Version 1.1
```

After deployment:

```text
CPU ↑ 40%
Reads ↑ 60%
Duration ↑ 25%
```

Query Store can help identify which queries changed behavior.

---

# 28. Database upgrade scenario

Suppose:

```text
SQL Server upgrade
+
Compatibility level change
```

Afterward, one query becomes slow.

Query Store may show:

```text
Old plan
New plan
Performance difference
```

This can dramatically shorten the investigation.

---

# 29. Compatibility-level changes

Remember our previous CE discussion.

Changing compatibility level can change optimizer behavior.

Therefore:

```text
Compatibility change
       ↓
Potential CE behavior change
       ↓
Different estimates
       ↓
Different plans
```

Query Store gives you a way to observe these effects.

---

# 30. Query Store and forced plans

SQL Server can allow a plan to be forced for a query in Query Store.

Conceptually:

```text
Query
 ↓
Known good Plan A
 ↓
Force Plan A
```

This can be a useful tactical response to a regression.

But:

> **Plan forcing is usually a containment mechanism, not the root-cause fix.**

---

# 31. When plan forcing makes sense

Suppose:

```text
Plan A:
Fast and stable

Plan B:
Suddenly terrible

Production incident:
NOW
```

Forcing Plan A can potentially restore acceptable performance while you investigate.

That can be a legitimate operational strategy.

---

# 32. Why forcing can be dangerous

Data changes.

Statistics change.

Indexes change.

Workload changes.

Therefore the plan that was optimal yesterday may become inappropriate tomorrow.

Plan forcing should be monitored.

---

# 33. The correct mindset

Don't think:

> "Force the plan and we're done."

Think:

```text
Incident
 ↓
Contain
 ↓
Investigate root cause
 ↓
Fix underlying issue
 ↓
Remove temporary workaround when appropriate
```

---

# 34. Plan forcing isn't magic

If the forced plan depends on:

```text
Index
Statistics
Data distribution
Memory
Hardware
```

and those conditions change, the forced plan may eventually become inefficient.

So every forced plan should have a reason.

---

# 35. Query Store hints

Modern SQL Server versions also provide Query Store-based mechanisms for influencing query behavior without necessarily changing application code.

These capabilities can be useful for targeted production interventions.

But the principle remains:

> **Use the least invasive mechanism that solves the actual problem.**

---

# 36. Query Store configuration matters

Query Store itself consumes resources.

It has to:

```text
Capture
Aggregate
Persist
Maintain
```

historical information.

Therefore, configuration should be appropriate for the workload.

---

# 37. Query Store storage

Query Store stores its information inside the database context.

Therefore you should monitor:

```text
Query Store size
Growth
Capture behavior
Flush/storage behavior
Retention
```

You don't want your performance-history system to become an operational problem.

---

# 38. Retention

Historical information is useful, but keeping unlimited history isn't always practical.

You need a balance between:

```text
Longer history
+
More troubleshooting context
```

and:

```text
Storage
+
Maintenance
```

---

# 39. Capture policy

Not every query necessarily deserves identical capture treatment.

Modern SQL Server versions provide configuration options that can control how aggressively Query Store captures workload information.

The goal is:

```text
Useful history
+
Reasonable overhead
```

---

# 40. Query Store "noise"

High-volume systems may generate huge numbers of query executions.

If capture isn't managed carefully, the store can contain a lot of low-value information.

The DBA's job is to extract:

```text
Signal
```

from:

```text
Noise
```

---

# 41. Query identity

Query Store groups executions based on query identity rather than simply treating every execution as a completely unrelated event.

This allows you to see:

```text
Same logical query
+
Different plans
+
Different performance
```

over time.

---

# 42. Query text changes

Be aware that apparently similar queries can be represented differently because of changes in:

```text
Whitespace
Comments
Literal values
Parameterization
Query structure
```

Understanding how Query Store identifies queries becomes important when doing large-scale analysis.

---

# 43. A production regression example

Imagine users report:

> "The checkout page became slow after yesterday's deployment."

You investigate.

Query Store shows:

```text
Query:
GetCustomerOrders

Before:
Plan 12
Avg duration = 80 ms

After:
Plan 19
Avg duration = 4,500 ms
```

Excellent.

Now you inspect Plan 19.

---

# 44. Plan comparison

Plan 12:

```text
Index Seek
Nested Loops
Low reads
```

Plan 19:

```text
Index Scan
Hash Match
Millions of reads
```

Now you ask:

> Why did the optimizer choose Plan 19?

Potential causes:

```text
Statistics changed
Data distribution changed
Index changed
Query changed
Compatibility level changed
Parameter sensitivity
Cardinality estimate changed
```

---

# 45. Statistics connection

Suppose you discover:

```text
Statistics updated
```

just before Plan 19 appeared.

Now you compare estimates.

Plan 19:

```text
Estimated = 100 rows
Actual = 5,000,000
```

You've found a strong lead.

The next step isn't automatically:

```text
UPDATE STATISTICS
```

You need to determine whether the statistics actually caused the bad estimate.

---

# 46. Query Store makes this timeline visible

Without historical data:

```text
Current plan
```

With Query Store:

```text
Plan A
 ↓
Statistics change
 ↓
Plan B
 ↓
Performance regression
```

This historical sequence is incredibly valuable.

---

# 47. Query Store and wait statistics

Modern Query Store capabilities can also provide historical wait-related information in supported SQL Server environments.

This helps connect:

```text
Query
+
Plan
+
Performance
+
Wait behavior
```

rather than looking at waits only as a server-wide snapshot.

---

# 48. Why historical waits matter

Suppose:

```text
Query was fast last week
```

but now:

```text
Query waits heavily on I/O
```

You can investigate whether:

```text
Plan changed
```

at the same time.

This gives you another dimension of evidence.

---

# 49. Query Store vs DMVs

Dynamic Management Views are excellent for:

```text
Current/recent state
```

Query Store is excellent for:

```text
Historical state
```

Think:

```text
DMVs → "What's happening?"
Query Store → "What has been happening?"
```

Both are important.

---

# 50. Why DMVs alone aren't enough

Many DMV statistics can be lost or reset when:

```text
SQL Server restarts
Plans are evicted
Cache changes
```

Query Store is designed to preserve historical workload information beyond the lifetime of a particular plan-cache state.

---

# 51. Query Store and plan cache

Plan cache:

```text
Fast
Current
Transient
```

Query Store:

```text
Historical
Persistent
Performance-focused
```

They complement one another.

---

# 52. Practical troubleshooting sequence

When a query suddenly becomes slow:

```text
Step 1:
Query Store → Did performance regress?

Step 2:
Did the plan change?

Step 3:
Which plan was better?

Step 4:
Why did the optimizer choose the new plan?

Step 5:
Check estimates.

Step 6:
Check statistics/data changes.

Step 7:
Check indexes/query changes.

Step 8:
Test the root-cause fix.

Step 9:
Use plan forcing only if necessary as containment.
```

---

# 53. What if the plan didn't change?

This is important.

Suppose Query Store shows:

```text
Same plan
Same query
```

but:

```text
Duration increased 10×
```

Then don't obsess over plan regression.

Investigate:

```text
Blocking
I/O pressure
CPU pressure
Memory pressure
Concurrency
Data volume
External dependencies
Infrastructure
```

The query may be executing the same plan against a different environment.

---

# 54. Same plan ≠ same runtime

For example:

```text
Plan = Index Seek → Nested Loops
```

can run in:

```text
100 ms
```

under light load and:

```text
20 sec
```

under severe blocking or resource contention.

The plan itself isn't necessarily the problem.

---

# 55. Query Store and blocking

Query Store can show duration increasing, but blocking investigation often requires additional tools and DMVs.

So:

```text
Query Store
+
Current wait/blocking information
```

is stronger than either alone.

---

# 56. Query Store and missing indexes

Query Store can identify expensive queries.

But don't automatically add every suggested index.

For each candidate index ask:

```text
Will it reduce meaningful workload?
How large will it be?
What write overhead will it introduce?
Does another index already cover it?
Does the query actually need optimization?
```

---

# 57. Query Store and indexing strategy

A good process:

```text
Query Store
   ↓
Find expensive query
   ↓
Inspect plan
   ↓
Identify expensive operator
   ↓
Understand why
   ↓
Design index if appropriate
   ↓
Test
   ↓
Measure Query Store afterward
```

This is much better than index-by-index guessing.

---

# 58. Query Store and statistics strategy

Similarly:

```text
Query Store
   ↓
Find regression
   ↓
Compare plans
   ↓
Find estimate error
   ↓
Inspect statistics
   ↓
Target statistics fix
   ↓
Measure
```

Now your statistics maintenance is workload-driven.

---

# 59. A mature production mindset

When someone says:

> "The database is slow."

Don't immediately ask:

> "Which index should we add?"

Ask:

```text
What changed?
When did it change?
Which workload is affected?
Is CPU higher?
Are reads higher?
Did plans change?
Did waits change?
Did data volume change?
Did statistics change?
```

That's professional troubleshooting.

---

# 60. Incident-response model

A useful production sequence:

```text
DETECT
  ↓
MEASURE
  ↓
COMPARE
  ↓
ISOLATE
  ↓
CONTAIN
  ↓
FIX
  ↓
VERIFY
  ↓
MONITOR
```

Query Store is especially powerful in the:

```text
MEASURE
COMPARE
ISOLATE
VERIFY
```

stages.

---

# 61. Interview question

### "What is Query Store?"

Strong answer:

> "Query Store is a SQL Server feature that captures historical query text, execution plans, and runtime performance information, allowing you to analyze query performance over time and identify plan changes and regressions."

---

# 62. Interview question

### "How would you investigate a query that suddenly became slow?"

Strong answer:

> "I'd first establish whether the query actually regressed using historical performance data. Then I'd compare the current plan with previously good plans, examine estimated versus actual rows, and investigate statistics, parameter sensitivity, indexes, query changes, waits, blocking, and resource pressure. I'd use plan forcing only as a tactical measure when appropriate, while addressing the underlying cause."

---

# 63. Interview question

### "Would you always force the previous good plan?"

Strong answer:

> "No. I'd first determine why the new plan was selected and whether the old plan is still appropriate for the current data and workload. Forcing can be useful as temporary containment, but it can become harmful if underlying conditions change."

---

# 64. Interview question

### "What is the difference between Query Store and DMVs?"

Strong answer:

> "DMVs are primarily useful for current or recent server and workload state, while Query Store provides historical query, plan, and runtime information that persists beyond transient plan-cache changes. They complement each other."

---

# 65. Expert mental model

Think of Query Store as a **flight recorder for SQL performance**.

```text
Query
  ↓
Plan
  ↓
Execution
  ↓
Performance
  ↓
History
```

When something goes wrong:

```text
Current behavior
      ↓
Historical comparison
      ↓
Find what changed
      ↓
Find why
      ↓
Fix
```

---

# 66. Lesson 140 takeaway

The central chain is:

```text
QUERY STORE
     ↓
HISTORICAL QUERIES
     ↓
HISTORICAL PLANS
     ↓
RUNTIME STATISTICS
     ↓
REGRESSION DETECTION
     ↓
ROOT-CAUSE ANALYSIS
```

### The expert rule

> **Don't troubleshoot production performance from a single execution plan. First determine what changed over time.**

That is the real power of Query Store.

---
