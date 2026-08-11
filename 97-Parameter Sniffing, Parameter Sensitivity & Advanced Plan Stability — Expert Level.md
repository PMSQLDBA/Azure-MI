# Lesson 130 — Parameter Sniffing, Parameter Sensitivity & Advanced Plan Stability — Expert Level

This lesson focuses on one of the most misunderstood SQL Server performance problems:

> **The same query can be fast for one parameter value and painfully slow for another.**

The issue is often not that SQL Server "forgot" how to optimize the query.

It is that **one execution plan may not be appropriate for every parameter value**.

---

## 1. The core problem

Consider:

```sql
SELECT *
FROM dbo.Orders
WHERE CustomerId = @CustomerId;
```

Customer distribution:

```text
Customer A → 10 rows
Customer B → 20 rows
Customer C → 15 rows
Customer X → 10,000,000 rows
```

One plan may work beautifully for Customer A but poorly for Customer X.

---

# 2. Parameter sniffing

When SQL Server compiles a parameterized query, it can use the parameter value available during compilation to help optimize the plan.

Conceptually:

```text
Query
  +
@CustomerId = A
      ↓
Compilation
      ↓
Plan optimized around A
      ↓
Plan cached
```

Later:

```text
@CustomerId = X
      ↓
Same cached plan
      ↓
Potentially poor performance
```

---

# 3. Parameter sniffing isn't inherently bad

This is extremely important.

Parameter sniffing can be beneficial.

If:

```text id="d5r8n2"
@CustomerId = A
```

returns 10 rows, SQL Server may choose an efficient seek-based plan.

Without using the available parameter information, it might make a less informed decision.

So the problem isn't:

> "SQL Server sniffed the parameter."

The problem is:

> **"The cached plan isn't suitable for the range of parameter values being executed."**

---

# 4. Parameter sensitivity

The more useful modern concept is:

> **Parameter sensitivity**

It occurs when different parameter values have dramatically different optimal execution strategies.

Example:

```text
Parameter A
→ 5 rows
→ Nested Loops + Seek

Parameter B
→ 20 million rows
→ Hash Join + Scan
```

One plan cannot necessarily be optimal for both.

---

# 5. Data skew creates the problem

The root often looks like:

```text id="x2m7p9"
Highly skewed data
       ↓
Different parameter selectivity
       ↓
Different ideal plans
       ↓
Plan reuse
       ↓
Performance variation
```

---

# 6. Example workload

Imagine:

```text
CustomerId = 100
Rows = 12

CustomerId = 200
Rows = 8

CustomerId = 999
Rows = 25,000,000
```

If the query is optimized for `CustomerId = 100`, a lookup-heavy plan may be excellent.

For `999`, it could become disastrous.

---

# 7. Why the optimizer can't simply choose the perfect plan every time

Compilation has a cost.

If SQL Server compiled every execution independently:

```text
Execution
 ↓
Compile
 ↓
Execute
```

you could waste significant CPU and increase latency.

Plan reuse exists for a reason:

```text
Compile once
 ↓
Reuse many times
```

The challenge is balancing:

```text
Plan reuse
vs
Parameter-specific optimization
```

---

# 8. How to recognize parameter sensitivity

Look for:

```text
Same query
+
Different parameter values
+
Large runtime differences
```

For example:

```text
Customer A:
80 ms

Customer B:
90 ms

Customer X:
15,000 ms
```

If the query text and plan are effectively the same, investigate parameter sensitivity.

---

# 9. Query Store is extremely useful

Query Store can help reveal:

```text id="q8v3m6"
Query history
Plan history
Runtime statistics
Performance variation
Plan changes
```

This lets you investigate whether performance varies across time or plans.

---

# 10. Compare runtime distributions

Suppose Query Store shows:

```text
Average duration = 2 sec
```

That's not enough.

Look deeper:

```text
P50 = 50 ms
P95 = 3 sec
P99 = 20 sec
```

That distribution strongly suggests inconsistent execution behavior.

---

# 11. Plan comparison

Suppose you find two plans:

```text
Plan A
→ Index Seek
→ Nested Loops
→ Key Lookup

Plan B
→ Scan
→ Hash Join
```

Now ask:

> Which parameter values correspond to each plan?

That can reveal parameter sensitivity.

---

# 12. The classic symptom

You execute:

```sql
EXEC dbo.GetOrders @CustomerId = 10;
```

and get:

```text
50 ms
```

Then:

```sql
EXEC dbo.GetOrders @CustomerId = 999;
```

and get:

```text
30 seconds
```

Then reverse the order.

If the performance pattern changes depending on which parameter is compiled first, that is a strong clue.

---

# 13. Stored procedures

Parameter sensitivity frequently appears with stored procedures:

```sql
CREATE PROCEDURE dbo.GetOrders
    @CustomerId int
AS
BEGIN
    SELECT ...
    FROM dbo.Orders
    WHERE CustomerId = @CustomerId;
END;
```

The procedure may reuse a compiled plan across executions.

---

# 14. Local variables aren't a magical fix

A common old-school workaround is replacing parameters with local variables.

That can change estimation behavior, but it may also remove useful parameter information.

Potential result:

```text
Parameter-aware estimate
        ↓
becomes
Generic estimate
```

That is not automatically better.

---

# 15. OPTION (RECOMPILE)

One possible strategy is:

```sql
SELECT ...
FROM dbo.Orders
WHERE CustomerId = @CustomerId
OPTION (RECOMPILE);
```

Conceptually:

```text
Each execution
      ↓
Fresh compilation
      ↓
Current parameter considered
      ↓
Potentially better plan
```

---

# 16. The cost of RECOMPILE

The downside is compilation overhead.

If a query executes:

```text
100 times/day
```

the cost may be insignificant.

If it executes:

```text
100,000 times/second
```

recompiling every execution may be unacceptable.

Therefore:

> **Use recompilation when the benefit outweighs compilation cost.**

---

# 17. Dynamic SQL

Another strategy can be parameterized dynamic SQL that produces different query shapes when appropriate.

For example:

```text
Small result workload
→ seek-oriented query

Large result workload
→ scan-oriented query
```

This can give the optimizer more flexibility.

But dynamic SQL should be designed carefully to avoid:

```text
SQL injection
Plan-cache fragmentation
Compilation overhead
Maintenance complexity
```

---

# 18. Query hints

SQL Server supports query hints that can influence optimization behavior.

Hints can be powerful.

They can also become technical debt.

Use them when:

```text
Root cause is understood
Behavior is predictable
Business impact is significant
Alternative solutions are inadequate
```

---

# 19. "Optimize for unknown"

Some scenarios use an approach that asks SQL Server to optimize without relying heavily on the current parameter value.

Conceptually:

```text
Specific parameter
       ↓
Generic optimization assumption
```

This can stabilize behavior for skewed workloads in certain cases.

But generic estimates can also be worse than parameter-specific ones.

---

# 20. OPTIMIZE FOR

SQL Server can also be directed to optimize around a specified parameter value in appropriate situations.

Conceptually:

```text
Optimize around representative value
```

This can be useful when one value represents the dominant or desired workload.

But hardcoding a value can become fragile as data changes.

---

# 21. Plan forcing

Query Store can be used for controlled plan management.

Suppose:

```text
Plan A = 100 ms
Plan B = 10 sec
```

and Plan B is a clear regression.

A forced plan can sometimes stabilize the workload while the root cause is investigated.

---

# 22. Plan forcing isn't permanent truth

Imagine:

```text
2025:
Plan A = excellent
```

Then data grows dramatically:

```text
2027:
Plan A = poor
```

If Plan A remains forcibly selected, you may prevent SQL Server from adapting.

Therefore:

> **Plan forcing should be monitored and revisited.**

---

# 23. Parameter Sensitive Plan optimization

Modern SQL Server versions include features designed to address parameter-sensitive workloads more intelligently.

The important conceptual idea is:

```text
One query
   ↓
Multiple parameter-sensitive plan variants
```

instead of forcing one plan to serve every parameter value.

---

# 24. The idea behind multiple variants

Imagine:

```text
Small parameter result
      ↓
Plan Variant A
      ↓
Seek-oriented

Large parameter result
      ↓
Plan Variant B
      ↓
Scan/hash-oriented
```

This is much closer to the actual workload reality.

---

# 25. Why this matters

Historically, the choice often felt like:

```text
Reuse one plan
       OR
Compile repeatedly
```

Modern parameter-sensitive optimization features can provide more nuanced behavior in supported environments.

---

# 26. Compatibility level matters

Whether a particular optimizer capability is available and how it behaves can depend on:

```text
SQL Server version
Database compatibility level
Configuration
Query shape
Workload
```

Always verify the environment before assuming a feature is available.

---

# 27. Don't confuse plan instability with parameter sensitivity

These are different concepts.

### Plan instability

The optimizer repeatedly chooses different plans over time.

### Parameter sensitivity

Different parameter values have different ideal plans.

They can overlap, but they aren't identical.

---

# 28. What can cause plan changes?

Possible causes include:

```text
Statistics updates
Index changes
Schema changes
Data distribution changes
Compatibility-level changes
Optimizer changes
Memory/resource conditions
Parameter sensitivity
```

Don't blame parameter sniffing whenever a plan changes.

---

# 29. Query Store investigation

A disciplined investigation looks like:

```text
Query
 ↓
Runtime history
 ↓
Plan history
 ↓
Parameter behavior
 ↓
Data distribution
 ↓
Estimate quality
 ↓
Plan suitability
```

---

# 30. Example investigation

Suppose:

```text
Query:
GetCustomerOrders

P50 = 40 ms
P99 = 18 sec
```

Query Store shows:

```text
Plan A:
Seek + Nested Loops

Plan B:
Scan + Hash Join
```

Now correlate:

```text
Plan A
→ small customers

Plan B
→ large customers
```

This is strong evidence of parameter-sensitive behavior.

---

# 31. Don't fix it by adding random indexes

This is an important lesson.

If the underlying issue is:

```text
Small parameter
→ seek is ideal

Large parameter
→ scan is ideal
```

adding another index may not solve the fundamental issue.

You need to address **plan selection across parameter values**.

---

# 32. Parameter sensitivity and indexes

Indexes still matter.

A seek-oriented plan requires a useful access path.

But an index that is excellent for tiny result sets may not be appropriate when millions of rows qualify.

Thus:

> **A good index does not guarantee a good plan for every parameter value.**

---

# 33. Parameter sensitivity and joins

Suppose:

```text
Small parameter
→ 10 rows
```

The optimizer might select:

```text
Nested Loops
```

For:

```text
Large parameter
→ 20 million rows
```

a:

```text
Hash Join
```

may be much more appropriate.

This is why parameter-sensitive plans can differ dramatically.

---

# 34. Parameter sensitivity and memory grants

Different parameter values can produce different cardinalities.

That affects:

```text
Memory grant
```

So one parameter may cause:

```text
Under-grant
→ spills
```

while another may cause:

```text
Over-grant
→ memory pressure
```

---

# 35. Parameter sensitivity and parallelism

A tiny result may be best handled serially.

A huge result may benefit from parallelism.

If the same cached plan is reused incorrectly, you can get:

```text
Small workload
→ unnecessary parallelism
```

or:

```text
Large workload
→ insufficient parallelism
```

---

# 36. Parameter sensitivity and reporting

Reporting workloads are especially susceptible when parameters represent highly variable ranges.

For example:

```text
@StartDate / @EndDate
```

One request might cover:

```text
1 day
```

another:

```text
5 years
```

The optimal plans may be dramatically different.

---

# 37. Date-range example

Query:

```sql
SELECT ...
FROM dbo.Sales
WHERE SaleDate >= @StartDate
  AND SaleDate < @EndDate;
```

Request A:

```text
1 day
```

Request B:

```text
5 years
```

One access strategy may not be optimal for both.

---

# 38. Parameter sensitivity and multi-tenant systems

Imagine a SaaS database:

```text
Tenant A → 100 rows
Tenant B → 10 million rows
```

Every query contains:

```text
WHERE TenantId = @TenantId
```

This is a classic scenario where tenant data distribution can create different optimal strategies.

---

# 39. Detecting tenant skew

Look for:

```text
Tenant
+
Execution count
+
Rows returned
+
Duration
```

If one tenant is dramatically larger than the rest, parameter sensitivity deserves investigation.

---

# 40. Application-level solutions

Sometimes the best fix is outside SQL Server.

Examples:

```text
Caching
Separate workloads
Tenant isolation
Pre-aggregation
Read replicas
Different APIs
Different query patterns
```

The database shouldn't be forced to solve every architectural problem.

---

# 41. When recompilation is attractive

Consider:

```text
Query is expensive
Parameter selectivity varies dramatically
Execution frequency is moderate
Optimal plan strongly depends on parameter
```

A fresh compilation may be worthwhile.

---

# 42. When recompilation is unattractive

Consider:

```text
Query executes extremely frequently
Compilation is expensive
Parameter variation isn't significant
Plan reuse is already effective
```

Then recompilation could cost more than it saves.

---

# 43. When plan forcing is attractive

Consider:

```text
Known regression
Stable workload
Known-good plan
Immediate production impact
Root cause requires longer investigation
```

Use it as controlled stabilization.

---

# 44. When plan forcing is unattractive

Avoid using it as:

```text
Permanent emergency button
```

when:

```text
Data changes frequently
Workload evolves
Plan is not consistently superior
```

---

# 45. When generic optimization is attractive

A generic strategy can be useful when:

```text
No single parameter value is representative
Performance differences are moderate
Plan stability matters
```

But always validate whether the generic plan is actually good enough.

---

# 46. Parameter sensitivity decision tree

```text
Performance varies by parameter?
          |
         Yes
          ↓
Is data distribution highly skewed?
          |
         Yes
          ↓
Do different plans perform best?
          |
      +---+---+
     Yes      No
      |        |
Multiple      Investigate
plans useful  estimation
      |
      ↓
Choose appropriate
plan-management strategy
```

---

# 47. Don't optimize from one execution

This is one of the most important rules.

Never conclude:

> "This query is fast."

after one test.

Instead test representative parameter values:

```text
Small
Medium
Large
Extreme
```

---

# 48. Build a parameter test matrix

For a customer query:

| Scenario        | Expected rows | Duration |
| --------------- | ------------: | -------: |
| Tiny customer   |            10 |        ? |
| Small customer  |         1,000 |        ? |
| Medium customer |       100,000 |        ? |
| Large customer  |           10M |        ? |

This reveals whether the query behaves consistently.

---

# 49. Production validation

A query that works beautifully in staging may fail in production because production has:

```text
Different data distribution
Different concurrency
Different statistics
Different indexes
Different parameter patterns
```

Always validate against realistic workload characteristics.

---

# 50. Advanced diagnostic sequence

When you suspect parameter sensitivity:

```text
1. Identify query
2. Capture parameter values
3. Measure runtime by parameter
4. Compare estimated vs actual rows
5. Inspect plan
6. Check Query Store history
7. Check data distribution
8. Compare alternative plans
9. Test mitigation
10. Measure CPU/IO/latency
11. Validate representative parameters
```

---

# 51. Interview question

### "What is parameter sniffing?"

Strong answer:

> "Parameter sniffing is the optimizer's use of parameter values available during compilation to help produce an execution plan. It is normally beneficial, but when parameter values have significantly different data distributions, the resulting cached plan may perform poorly for other values."

---

# 52. Interview question

### "How do you diagnose parameter sniffing?"

Strong answer:

> "I would look for large runtime variation across parameter values, compare Query Store runtime and plan history, inspect estimated versus actual cardinalities, examine data skew, and determine whether different parameter values require fundamentally different plans."

---

# 53. Interview question

### "Would you always use RECOMPILE?"

Strong answer:

> "No. RECOMPILE can produce parameter-specific plans but adds compilation overhead. I would use it only when the performance benefit justifies that overhead and after confirming parameter sensitivity is actually the problem."

---

# 54. Interview question

### "Would you force the fastest plan?"

Strong answer:

> "Not necessarily. I would determine whether that plan is appropriate across the workload. A plan that's fastest for one parameter value may be disastrous for another."

---

# 55. The expert mental model

Don't think:

```text
Query → Plan
```

Think:

```text
Query
 ↓
Parameters
 ↓
Statistics
 ↓
Cardinality estimates
 ↓
Cost model
 ↓
Candidate plans
 ↓
Chosen plan
 ↓
Plan reuse
 ↓
Actual workload
```

The mismatch often happens between:

```text
Chosen plan
```

and:

```text
Actual parameter distribution
```

---

# 56. Lesson 130 takeaway

The golden rule is:

> **A cached plan is only good if it is appropriate for the workload that reuses it.**

Remember:

```text
Data skew
   ↓
Different parameter selectivity
   ↓
Different ideal plans
   ↓
Plan reuse
   ↓
Potential performance regression
```

And your diagnostic approach:

```text
Measure by parameter
       ↓
Compare plans
       ↓
Check estimates
       ↓
Understand skew
       ↓
Choose the least invasive
appropriate solution
```

**Next: Lesson 131 — Execution Plan Internals: Operators, Costs, Memory Grants & Spills — Expert Level.**
