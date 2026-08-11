# Lesson 126 — Query Store Architecture & Historical Performance Analysis — Expert Level

The goal of this lesson is to understand **Query Store as the historical memory of your SQL workload**.

If Lesson 125 taught you:

> **“What is SQL Server waiting for right now?”**

Query Store helps answer:

> **“Which queries have been running, how have they performed over time, and what changed?”**

---

# 1. The fundamental problem

Imagine production performance changes:

```text
Monday
Query = 200 ms

Tuesday
Query = 220 ms

Wednesday
Query = 4,000 ms
```

The incident is discovered Wednesday afternoon.

Without historical information, you may ask:

> "What changed?"

Query Store can provide the historical evidence needed to investigate.

---

# 2. What Query Store captures

At a high level, Query Store maintains information about:

```text
Query
  ↓
Query text
  ↓
Execution
  ↓
Execution plan
  ↓
Runtime statistics
  ↓
Historical performance
```

This allows you to analyze query behavior across time.

---

# 3. Query Store mental model

Think:

```text id="c5m8x2"
                 QUERY
                   |
             +-----+-----+
             |           |
          PLAN A       PLAN B
             |           |
          Runtime      Runtime
          history      history
             |           |
             +-----+-----+
                   |
              Time series
```

This is particularly valuable for identifying **plan regressions**.

---

# 4. Why historical data matters

Traditional real-time monitoring answers:

> "What is happening now?"

Query Store helps answer:

> "What happened yesterday?"

and:

> "What happened after the deployment?"

and:

> "Was this query always slow?"

---

# 5. Query performance over time

Imagine:

```text id="v9k3p1"
Execution time

5 sec |                 *
4 sec |                *
3 sec |               *
2 sec | *
1 sec | *
0 sec +---------------------
       Mon Tue Wed Thu
```

Something changed around Wednesday.

Now investigate:

```text id="m2r7x5"
Plan change?
Statistics?
Index?
Data volume?
Application release?
Parameter behavior?
```

---

# 6. Query Store isn't just a slow-query list

This is an important distinction.

Query Store can help analyze:

```text id="q6x1n8"
Query frequency
Duration
CPU
Logical reads
Physical reads
Execution count
Plans
Plan changes
Runtime trends
```

The objective is not simply:

> "Find queries taking more than one second."

It's to understand workload behavior.

---

# 7. Query identification

A production system may execute thousands or millions of queries.

Query Store helps organize them into query identities.

Think:

```text id="a8p4s2"
Application workload
        ↓
Thousands of executions
        ↓
Queries
        ↓
Historical statistics
```

You can then rank workload by different dimensions.

---

# 8. Top CPU queries

Suppose:

```text id="j5m8r3"
Query A → 60% CPU
Query B → 15%
Query C → 8%
```

Query A becomes an obvious candidate for investigation.

But don't automatically optimize it.

Ask:

> Is Query A supposed to consume that CPU?

A legitimate analytical workload may naturally be expensive.

---

# 9. Top duration queries

You might discover:

```text id="w2c7v9"
Query A → 30 sec
Query B → 12 sec
Query C → 8 sec
```

Again, context matters.

A query running once per day for 30 seconds may matter less than:

```text id="p6n1x4"
Query D → 200 ms
executed 500,000 times/hour
```

---

# 10. Total resource consumption

A useful principle:

```text id="r8m3q5"
Cost per execution
        ×
Execution count
        =
Total workload cost
```

A moderately expensive query executed millions of times can dominate the system.

---

# 11. Query frequency

Example:

```text id="u4k9s1"
Query A:
100 ms × 1,000 executions

Query B:
2 sec × 10 executions
```

Total execution time:

```text id="g7p2v6"
A = 100 sec
B = 20 sec
```

Query A consumes more aggregate execution time despite being individually faster.

---

# 12. Query Store and baselines

Create a normal-performance baseline.

For example:

```text id="x5c8m2"
Query
P50 = 80 ms
P95 = 150 ms
CPU = 20 ms
Reads = 2,000
```

Later:

```text id="n9r4k7"
P50 = 300 ms
P95 = 2 sec
CPU = 100 ms
Reads = 50,000
```

That's a measurable regression.

---

# 13. Plan history

One of Query Store's most valuable capabilities is historical plan information.

A query can have:

```text id="b6q1t8"
Plan A
Plan B
Plan C
```

over its lifetime.

The key question becomes:

> **Did the execution plan change when performance changed?**

---

# 14. Plan regression

Example:

```text id="s3m8x5"
Plan A
 ↓
100 ms

Plan B
 ↓
5 seconds
```

That's a classic plan-regression investigation.

---

# 15. Why plans change

Possible causes include:

```text id="k2p7v9"
Statistics changes
Data distribution
Index changes
Schema changes
Parameter behavior
Optimizer changes
Database changes
Application changes
```

Don't assume the optimizer simply "made a mistake."

Understand the circumstances.

---

# 16. The optimizer's job

The SQL Server optimizer tries to find an efficient execution strategy.

Conceptually:

```text id="v8c4n1"
SQL query
   ↓
Optimizer
   ↓
Possible plans
   ↓
Estimated cost
   ↓
Chosen plan
```

If its assumptions change, the chosen plan may change.

---

# 17. Estimated vs actual reality

The optimizer makes estimates.

Example:

```text id="d5x9m3"
Estimated rows = 100
Actual rows    = 1,000,000
```

That mismatch can lead to a poor plan.

Query Store tells you the historical outcome; execution plans help explain why.

---

# 18. Query Store + execution plans

Use them together:

```text id="q7r2m6"
Query Store
   ↓
Find regression
   ↓
Compare plans
   ↓
Inspect plan operators
   ↓
Find cause
```

This is much stronger than looking at a plan in isolation.

---

# 19. Query Store + wait statistics

Lesson 125:

```text id="j3n8x4"
Wait statistics
```

Lesson 126:

```text id="p6v1k9"
Query Store
```

Together:

```text id="r4m7c2"
Wait
 ↓
Affected workload
 ↓
Query Store
 ↓
Query
 ↓
Plan
 ↓
Root cause
```

This creates a powerful troubleshooting workflow.

---

# 20. Query Store + application deployment

Suppose:

```text id="w8x3q5"
10:00 → Application release
10:15 → Query performance changes
10:20 → CPU increases
10:30 → User complaints
```

Query Store may reveal that a frequently executed query changed behavior after the release.

Now you have a strong investigation path.

---

# 21. Query Store and ORM-generated SQL

Modern applications often use ORMs.

Developers may not write every SQL statement manually.

The application could generate:

```text id="m1q9v6"
Complex JOIN
+
Multiple predicates
+
Large projection
+
Parameterization
```

Query Store allows database engineers to see the actual workload reaching SQL Server.

---

# 22. Hidden workload

An application team might say:

> "Our application only performs simple reads."

Query Store may reveal:

```text id="c5p2n8"
Huge joins
Repeated queries
Excessive executions
Large result sets
```

Observability replaces assumptions with evidence.

---

# 23. Query Store and execution count

A query's frequency can reveal architectural problems.

Example:

```text id="z7r4m1"
SELECT Customer
```

executed:

```text id="h2k8v5"
1,000,000 times/hour
```

That could indicate:

```text id="q9m3x6"
N+1 behavior
Missing caching
Poor batching
Application design issue
```

---

# 24. Query Store and N+1

Suppose an API retrieves:

```text id="s4v8p2"
100 customers
```

and then runs:

```text id="a6n1k7"
100 separate order queries
```

Query Store may show the same query executing repeatedly.

That can expose an application-level performance problem.

---

# 25. Query Store and data growth

A query may be fast today:

```text id="e8q2m5"
100 ms
```

but after data grows:

```text id="r3x7c9"
2 seconds
```

Query Store's historical trend can reveal gradual degradation.

This is valuable for capacity planning.

---

# 26. Query Store as a performance historian

Think of Query Store as:

```text id="n5k1v8"
                   QUERY STORE
                        |
          +-------------+-------------+
          |             |             |
        Queries       Plans         Runtime
          |             |             |
          +-------------+-------------+
                        |
                     Timeline
                        |
               Performance history
```

---

# 27. Query Store retention

Historical information consumes storage.

Therefore, Query Store itself needs operational consideration:

```text id="u6p4m9"
Retention
Size
Capture behavior
Cleanup
Monitoring
```

You don't want unlimited historical collection without understanding its resource implications.

---

# 28. Query Store capture strategy

Not every workload should necessarily be captured with identical granularity.

You need to balance:

```text id="x2m8q5"
Diagnostic value
        vs
Storage/resource overhead
```

The exact configuration should be appropriate to the environment and workload.

---

# 29. Query Store isn't a substitute for monitoring

Query Store is powerful, but you still need:

```text id="k7v3n1"
Azure monitoring
Database metrics
Application telemetry
Logs
Wait analysis
Alerts
```

Think of Query Store as one layer in the observability stack.

---

# 30. Performance investigation example

### Symptom

Users report:

> "Search became slow."

Metrics:

```text id="g4p9c2"
CPU ↑
Latency ↑
```

Query Store:

```text id="t6x1m8"
Search query duration increased
```

Plan history:

```text id="w3r7n5"
Plan A → index seek
Plan B → large scan
```

Now investigate:

```text id="q8m2v6"
Why did Plan B appear?
```

---

# 31. Plan regression investigation

Use this sequence:

```text id="d5k9x3"
1. Identify regression
       ↓
2. Identify affected query
       ↓
3. Compare historical plans
       ↓
4. Compare runtime statistics
       ↓
5. Examine statistics/index/data changes
       ↓
6. Test candidate fix
       ↓
7. Validate performance
```

---

# 32. Query Store and forced plans

SQL Server environments can support mechanisms for plan forcing in appropriate scenarios.

Conceptually:

```text id="p2x7m4"
Good plan
   ↓
Known regression
   ↓
Temporarily prefer stable plan
```

But plan forcing should not become:

> "Force every query to the plan that worked yesterday."

---

# 33. Why forced plans can be dangerous

Data changes.

Workloads change.

Indexes change.

A plan that was good for:

```text id="r8c3v6"
1 million rows
```

may be poor for:

```text id="m1q7k9"
500 million rows
```

Therefore:

> Plan forcing is a tool for controlled stabilization, not a replacement for root-cause analysis.

---

# 34. Query Store and change management

When deploying changes, record:

```text id="v5n2x8"
Deployment time
Schema changes
Index changes
Application version
Configuration changes
```

Then correlate with Query Store.

This dramatically improves incident investigation.

---

# 35. Performance timeline

A useful mental model:

```text id="h7q4m2"
Deployment
    |
    v
Plan change
    |
    v
CPU increases
    |
    v
Latency increases
    |
    v
Users complain
```

Query Store can help reconstruct this sequence.

---

# 36. Query Store and incident response

During an incident:

```text id="n3x8p5"
Current metrics
      +
Current waits
      +
Query Store history
      +
Recent deployments
```

Together they help answer:

> **What changed?**

---

# 37. Top resource-consuming queries

Don't only look for the slowest query.

Analyze:

```text id="q6m1v9"
Highest CPU
Highest logical reads
Highest physical reads
Longest duration
Most executions
Largest total resource consumption
```

Different rankings reveal different problems.

---

# 38. Example

Suppose:

| Query | Avg duration | Executions | Total time |
| ----- | -----------: | ---------: | ---------: |
| A     |       10 sec |          5 |     50 sec |
| B     |       100 ms |    100,000 | 10,000 sec |
| C     |        2 sec |         20 |     40 sec |

Query B is much more important from an aggregate workload perspective.

---

# 39. Average isn't enough

Suppose:

```text id="y4r8p2"
Average = 100 ms
```

But:

```text id="c7m3x9"
P50 = 50 ms
P95 = 100 ms
P99 = 5 sec
```

Average hides tail latency.

Use distributions and percentiles where possible.

---

# 40. Query Store and workload classification

Separate workloads conceptually:

```text id="m9p2v6"
OLTP
Reporting
ETL
Maintenance
Administrative
```

A 20-second analytical query may be completely normal in one workload and disastrous in another.

---

# 41. Query Store and maintenance

Maintenance operations can affect query performance through:

```text id="f3x8n1"
Statistics changes
Index operations
Resource consumption
Plan changes
```

If performance changes after maintenance, investigate the relationship.

---

# 42. Query Store and statistics updates

Statistics influence plan selection.

Therefore:

```text id="u5k1q7"
Statistics change
      ↓
Optimizer estimates change
      ↓
Plan may change
      ↓
Runtime may change
```

This is one reason historical plan information is so valuable.

---

# 43. Query Store and parameter sensitivity

A query may behave differently depending on parameter values.

Historical runtime information can reveal patterns such as:

```text id="r2m7c4"
Same query
 ↓
Different execution characteristics
```

Then investigate:

```text id="x9p3v6"
Parameter values
Plan choice
Data distribution
```

---

# 44. Query Store and regression detection

A mature platform can establish automated or operational processes for detecting:

```text id="k8q4m1"
Duration regression
CPU regression
Read regression
Execution-count changes
Plan changes
```

This moves performance management from reactive to proactive.

---

# 45. Performance alert example

Imagine:

```text id="s5n9x2"
Query P95:
150 ms → 800 ms
```

Your monitoring detects the deviation.

Then:

```text id="v1m6r8"
Query Store
 ↓
Plan changed
 ↓
Investigate
```

You may resolve the issue before users report it.

---

# 46. Query Store and capacity planning

Historical data can reveal:

```text id="g7c2p5"
Query volume ↑
CPU ↑
Reads ↑
Latency ↑
```

This indicates workload growth.

You can then decide whether to:

```text id="n4x8m3"
Optimize
Scale
Redesign
Cache
Archive
Partition
Change workload architecture
```

---

# 47. Don't blame SQL Server automatically

Suppose Query Store shows:

```text id="q2p7v9"
Query execution = 50 ms
```

but application latency is:

```text id="d8m3x1"
2.5 sec
```

The database isn't responsible for most of the latency.

Always correlate database and application telemetry.

---

# 48. Query Store architecture

Conceptually:

```text id="b6v1k9"
                 APPLICATION
                      |
                      ↓
                 SQL QUERIES
                      |
                      ↓
                 SQL SERVER
                      |
             +--------+--------+
             |                 |
          Execution         Query Store
             |                 |
             |          +------+------+ 
             |          |             |
             |        Plans        Runtime
             |          |             |
             +----------+-------------+
                        |
                   Historical
                    analysis
```

---

# 49. Expert operational workflow

During a performance incident:

```text id="w4x9c2"
1. Establish incident window
2. Identify affected workload
3. Check Query Store trends
4. Identify top resource consumers
5. Compare plans
6. Correlate waits
7. Correlate deployments
8. Form hypothesis
9. Test fix
10. Validate
```

---

# 50. The most important distinction

Remember:

```text id="r7m2k5"
Monitoring
    ↓
"What is happening?"

Query Store
    ↓
"What has been happening?"

Execution plan
    ↓
"How is the query executing?"

Wait statistics
    ↓
"What is it waiting for?"

Application tracing
    ↓
"How does this affect the user?"
```

Together they create a complete performance picture.

---

# 51. Expert interview question

### "What is Query Store?"

Strong answer:

> "Query Store is a SQL Server database-level feature that captures historical query execution information, including query text, runtime statistics, and execution plans. It allows us to analyze workload behavior over time, identify regressions, compare plans, and investigate changes that affect performance."

---

# 52. Expert interview question

### "How would you investigate a query that suddenly became slow?"

Strong answer:

> "I would identify when the regression began, use Query Store to compare historical runtime statistics and execution plans, correlate the change with statistics, indexes, schema or application deployments, inspect the affected plans and waits, test the suspected cause, and then validate the improvement."

---

# 53. Expert interview question

### "Would you immediately force the previous plan?"

Strong answer:

> "Not automatically. I would first determine why the plan changed and whether the previous plan remains appropriate for the current data and workload. Plan forcing can be useful as a controlled stabilization mechanism, but it shouldn't replace root-cause analysis."

---

# 54. Expert mindset

The junior question is:

> **"Which query is slow?"**

The senior question is:

> **"Which workload consumes the most resources?"**

The expert question is:

> **"What changed in the workload or execution strategy, why did it change, and how can we prevent the regression from recurring?"**

---

# 55. The complete Query Store investigation chain

```text id="c8x4n7"
USER REPORT
     ↓
PERFORMANCE METRICS
     ↓
INCIDENT WINDOW
     ↓
QUERY STORE
     ↓
AFFECTED QUERY
     ↓
PLAN HISTORY
     ↓
RUNTIME HISTORY
     ↓
WAIT STATISTICS
     ↓
SCHEMA / DATA / DEPLOYMENT CHANGES
     ↓
ROOT CAUSE
     ↓
FIX
     ↓
VALIDATION
     ↓
BASELINE
```

---

# 56. Lesson 126 takeaway

Remember these four questions:

```text id="m2v7q9"
1. Which query?
2. How often?
3. How expensive?
4. What changed?
```

And remember the core relationship:

> **Query Store tells you what happened over time; execution plans help explain how it happened; wait statistics help explain where time was spent.**

That combination is one of the most important foundations of expert SQL Server performance engineering.

**Next: Lesson 127 — Query Performance Tuning: A Disciplined Optimization Method — Expert Level.**
