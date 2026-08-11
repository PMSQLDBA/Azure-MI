# Lesson 143 — Query Store + Parameter Sniffing & Parameter-Sensitive Queries

Today we tackle one of the most common reasons a query can be **fast for one parameter and painfully slow for another**.

The key concept is:

> **One execution plan may not be equally good for every parameter value.**

---

## 1. What is parameter sniffing?

Consider:

```sql
SELECT *
FROM Orders
WHERE CustomerId = @CustomerId;
```

SQL Server needs an execution plan.

During compilation, it may use the parameter value available at compilation to estimate how many rows will be returned.

That information can influence the chosen plan.

This behavior is commonly called **parameter sniffing**.

---

# 2. Why can that be a problem?

Imagine the data looks like this:

```text
Customer A → 10 orders
Customer B → 10,000,000 orders
```

One plan might be excellent for Customer A:

```text
Index Seek
    ↓
Nested Loops
```

But that same strategy could be terrible for Customer B.

For millions of rows, SQL Server may prefer something closer to:

```text
Scan
   ↓
Hash-based processing
```

---

# 3. The fundamental problem

The issue isn't:

```text
Parameter sniffing = bad
```

The real issue is:

```text
Different parameter values
        ↓
Very different data distributions
        ↓
One plan isn't ideal for all values
```

Parameter sniffing can actually be beneficial when the first compiled value produces a good plan for most executions.

---

# 4. A simple example

Suppose:

```sql
EXEC dbo.GetOrders @CustomerId = 10;
```

Customer 10 has:

```text
15 rows
```

SQL Server chooses:

```text
Index Seek
```

Later:

```sql
EXEC dbo.GetOrders @CustomerId = 99999;
```

Customer 99999 has:

```text
20 million rows
```

The previously compiled plan may now be inefficient.

---

# 5. The classic symptom

You may hear:

> "The query is sometimes fast and sometimes slow."

That's a major clue.

Especially when:

```text
Same query
Same database
Same code
Different parameter values
```

produce dramatically different performance.

---

# 6. But don't jump to parameter sniffing

Intermittent performance can also result from:

```text
Blocking
I/O pressure
CPU pressure
Plan changes
Cache behavior
Concurrent workload
Data growth
```

So parameter sensitivity is a hypothesis—not a conclusion.

---

# 7. Query Store is extremely useful here

Suppose Query Store shows:

```text
Query X

Plan 101:
Average duration = 50 ms

Plan 205:
Average duration = 15 sec
```

And both plans have been used for the same query.

That's a major clue.

Now investigate:

```text
Which parameters were associated with the executions?
Why are multiple plans appearing?
How different are the plans?
```

---

# 8. One query, multiple plans

Conceptually:

```text
                Query
                  ↓
        ┌─────────┴─────────┐
        ↓                   ↓
      Plan A              Plan B
        ↓                   ↓
     Fast values        Large values
```

Multiple plans don't automatically mean something is broken.

They may reflect genuinely different optimal strategies.

---

# 9. Data skew

The underlying issue is often **skew**.

For example:

```text
CustomerId
---------
1        → 20,000,000 rows
2        → 5 rows
3        → 8 rows
4        → 12 rows
```

Customer 1 is dramatically different from most customers.

A single generic plan may struggle to serve both extremes efficiently.

---

# 10. Histograms and skew

Recall that column statistics contain a histogram.

The optimizer uses statistics to estimate selectivity.

But highly skewed data can create situations where:

```text
Value A
→ extremely common

Value B
→ extremely rare
```

and the best execution strategy differs substantially.

---

# 11. Why estimates matter

Suppose the optimizer expects:

```text
Estimated = 10 rows
```

but the parameter actually produces:

```text
Actual = 20,000,000 rows
```

That can affect:

```text
Join choice
Memory grant
Access method
Parallelism
Aggregation strategy
```

A small estimation mistake can therefore cascade into a large plan difference.

---

# 12. Seek vs scan

One of the classic parameter-sensitive differences:

### Small result

```text
Index Seek
```

### Huge result

```text
Index Scan
```

A seek isn't inherently better.

A scan isn't inherently worse.

The right choice depends on how much data must actually be processed.

---

# 13. Why a seek can become expensive

Suppose:

```text
1 row returned
```

A seek is excellent.

But imagine:

```text
20 million rows returned
```

A seek followed by millions of lookups may be much worse than reading a large contiguous portion of the table/index.

This is why:

> **"Seek = good, scan = bad" is an incorrect rule.**

---

# 14. Parameter-sensitive query example

Suppose:

```sql
CREATE PROCEDURE dbo.GetOrders
    @CustomerId int
AS
BEGIN
    SELECT OrderId, OrderDate, TotalAmount
    FROM Orders
    WHERE CustomerId = @CustomerId;
END;
```

Customer distribution:

```text
Customer 1 → 25 million rows
Customer 2 → 10 rows
```

A single cached plan may not serve both efficiently.

---

# 15. How the problem appears in production

You might observe:

```text
User A:
50 ms

User B:
12 sec

User C:
40 ms

User D:
15 sec
```

The application team says:

> "It's random."

But perhaps the behavior is correlated with customer IDs.

That is not random.

It's parameter-sensitive.

---

# 16. Query Store investigation

Start with:

```text
Query
↓
Runtime statistics
↓
Plan history
↓
Performance distribution
```

Look for:

```text
Multiple plans
Large duration variation
Large CPU variation
Different read patterns
```

---

# 17. Compare Plan A and Plan B

Suppose:

### Plan A

```text
Index Seek
Nested Loops
Logical reads = 500
Duration = 20 ms
```

### Plan B

```text
Index Scan
Hash Join
Logical reads = 5,000,000
Duration = 10 sec
```

Now ask:

> Which parameter values produce each behavior?

---

# 18. Don't assume Plan A is universally better

Suppose Plan A is used for:

```text
Customer 2 → 10 rows
```

and Plan B for:

```text
Customer 1 → 25 million rows
```

Plan B may actually be the correct plan for Customer 1.

The goal isn't:

```text
One plan for everything
```

The goal is:

> **Efficient plans for the relevant parameter patterns.**

---

# 19. Parameter-sensitive optimization

Modern SQL Server versions include features designed to handle certain parameter-sensitive workloads more intelligently.

The important idea is:

```text
One query
     ↓
Recognize different cardinality ranges
     ↓
Potentially use different plan variants
```

Instead of forcing one compromise plan onto every parameter value.

---

# 20. The conceptual model

Traditional behavior:

```text
Parameters
    ↓
One cached plan
    ↓
All executions
```

Parameter-sensitive optimization can allow a model closer to:

```text
Parameters
       ↓
┌──────┼──────┐
↓      ↓      ↓
Small Medium Large
↓      ↓      ↓
Plan A Plan B Plan C
```

The exact implementation depends on SQL Server version and feature availability.

---

# 21. Why this is important

Historically, DBAs often had to choose between techniques such as:

```text
RECOMPILE
OPTIMIZE FOR
Dynamic SQL
Local variables
Plan forcing
Query hints
```

Modern SQL Server capabilities can reduce the need for some manual workarounds in supported scenarios.

---

# 22. OPTION (RECOMPILE)

One classic approach is:

```sql
OPTION (RECOMPILE)
```

Conceptually:

```text
Each execution
      ↓
Compile using current parameter
      ↓
Potentially better plan
```

This can help parameter-sensitive queries.

But it has a cost.

---

# 23. The downside of RECOMPILE

Compilation itself consumes resources.

If a query executes:

```text
100 times/day
```

recompilation may be reasonable.

If it executes:

```text
100,000 times/sec
```

constant recompilation could become expensive.

Therefore:

> **RECOMPILE is a tool, not a default solution.**

---

# 24. OPTIMIZE FOR

Another approach is to influence optimization toward a particular parameter behavior.

Conceptually:

```text
OPTIMIZE FOR (@CustomerId = ...)
```

This can be useful in carefully understood workloads.

But it can also hard-code an assumption.

---

# 25. Why hard-coded assumptions are risky

Suppose today:

```text
Customer 100
→ representative workload
```

Six months later:

```text
Customer 100
→ unusual workload
```

Your optimization assumption may become stale.

This is why targeted fixes should be monitored.

---

# 26. Local variables aren't magic

A common historical workaround is:

```sql
DECLARE @LocalCustomerId int = @CustomerId;
```

and then using:

```sql
WHERE CustomerId = @LocalCustomerId
```

This can change estimation behavior.

But it doesn't magically make the query optimal.

It may simply trade:

```text
Parameter-specific estimate
```

for:

```text
More generic estimate
```

That can help one workload and hurt another.

---

# 27. Dynamic SQL

Parameterized dynamic SQL can sometimes allow different statements or optimization contexts for substantially different scenarios.

But:

> **Dynamic SQL should be designed carefully.**

You must consider:

```text
Security
Parameterization
Plan reuse
Maintainability
Complexity
```

Don't use dynamic SQL merely because someone says "parameter sniffing."

---

# 28. Query Store hints

In supported SQL Server versions, Query Store can provide a way to apply targeted query-level guidance without changing application code.

This can be useful when:

```text
Application release cannot happen immediately
```

or:

```text
Production needs a controlled intervention
```

But again:

> Use the smallest appropriate intervention.

---

# 29. Plan forcing

Suppose Plan A is consistently good for the dominant workload.

You might consider forcing it.

But if:

```text
Plan A → excellent for small customers
Plan B → excellent for huge customers
```

forcing Plan A globally may simply move the problem.

This is why parameter-sensitive workloads are particularly dangerous candidates for blind plan forcing.

---

# 30. A real troubleshooting sequence

When you suspect parameter sensitivity:

```text
1. Identify query
2. Check Query Store history
3. Look for multiple plans
4. Compare runtime characteristics
5. Identify parameter/data distribution
6. Compare estimates vs actual rows
7. Determine whether plans correspond to different parameter ranges
8. Choose an appropriate strategy
9. Test with representative parameter values
10. Verify production behavior
```

---

# 31. Representative testing is critical

Never test only:

```text
@CustomerId = 1
```

if your workload contains:

```text
1 → huge customer
2 → tiny customer
3 → medium customer
```

Test across the distribution.

For example:

```text
Small
Medium
Large
Extreme
```

---

# 32. Why one successful test proves little

Suppose you optimize for:

```text
Customer 2
```

and the query becomes:

```text
20 ms
```

Great.

But if:

```text
Customer 1
```

becomes:

```text
60 seconds
```

you haven't solved the workload.

You've optimized one point.

---

# 33. Think in workload classes

A better approach is:

```text
Small result set
Medium result set
Large result set
Huge result set
```

Then ask:

> What execution strategy is appropriate for each class?

This is the heart of parameter-sensitive optimization.

---

# 34. Parameter sniffing vs stale statistics

These can interact.

Suppose:

```text
Statistics inaccurate
+
Parameter-sensitive data
```

The optimizer may produce a particularly poor plan.

Therefore don't stop at:

> "It's parameter sniffing."

Ask:

> **Why are the estimates wrong, and why does the chosen plan fail for this parameter?**

---

# 35. Parameter sniffing vs blocking

These can also coexist.

A query may have:

```text
Good plan
+
Heavy blocking
```

and appear slow.

Don't attribute every intermittent slowdown to parameter sensitivity.

---

# 36. Parameter sniffing vs plan cache

Remember:

```text
Plan cache
```

determines whether a compiled plan can be reused.

But:

```text
Query Store
```

helps you understand historical plans and runtime behavior.

Use both concepts appropriately.

---

# 37. A useful diagnostic clue

Suppose:

```text
Same query
Same plan
Different parameter
Different duration
```

This doesn't automatically prove parameter sniffing.

The same plan can behave differently simply because it processes different amounts of data.

You need to inspect:

```text
Actual rows
Logical reads
CPU
Waits
```

---

# 38. Stronger evidence

Suppose:

```text
Parameter A:
Estimated = 10
Actual = 12
Duration = 20 ms

Parameter B:
Estimated = 10
Actual = 20,000,000
Duration = 15 sec
```

Now the estimate mismatch is significant.

That strengthens the parameter-sensitivity diagnosis.

---

# 39. A particularly useful pattern

Look for:

```text
Parameter A
→ plan optimized for tiny result

Parameter B
→ same plan
→ enormous result
→ huge CPU/reads
```

This is the classic shape.

---

# 40. Parameter-sensitive workload and indexes

Sometimes the right index can reduce the difference between parameter classes.

For example:

```text
CustomerId
```

may need an index that makes small-result queries efficient.

But again, an index is not automatically the answer.

For very large result sets, a scan may still be appropriate.

---

# 41. Covering indexes

If the query needs:

```sql
SELECT OrderId, OrderDate, TotalAmount
```

an index on:

```text id="b1c7m4"
CustomerId
```

with appropriate included columns may reduce lookups for selective queries.

But the index must be evaluated against:

```text id="z5m9q2"
Write cost
Storage
Maintenance
Large-result behavior
```

---

# 42. Why key lookups matter here

Suppose a small customer returns:

```text id="r4h8c2"
20 rows
```

A seek + lookup may be excellent.

But for:

```text id="k6p3v1"
20 million rows
```

the same lookup strategy could become disastrous.

This is another reason parameter-sensitive workloads can produce radically different optimal plans.

---

# 43. Query Store as the evidence source

For these cases, Query Store can answer:

```text id="w2s7n8"
How many plans?
How often used?
How fast?
How much CPU?
How many reads?
When did each plan appear?
```

Then the execution plan answers:

```text id="j8m4x5"
What is each plan actually doing?
```

---

# 44. Production fix hierarchy

A useful way to think about interventions:

### Level 1

Correct the underlying data/statistics/query/index problem.

### Level 2

Use an appropriate optimizer feature.

### Level 3

Use targeted query guidance.

### Level 4

Use plan forcing as tactical containment.

The exact order can vary, but don't jump immediately to permanent overrides.

---

# 45. Monitoring after the fix

After implementing a solution, track:

```text id="x6r1p9"
Average duration
P95/P99 if available
CPU
Reads
Execution count
Plan distribution
Error/timeout rate
```

For parameter-sensitive queries, **plan distribution** is particularly interesting.

---

# 46. What success looks like

Before:

```text id="e7k2n5"
Small values → 20 ms
Large values → 20 sec
```

After:

```text id="s4q8m1"
Small values → 20 ms
Large values → 500 ms
```

That's a meaningful improvement.

---

# 47. What failure looks like

You optimize the large values:

```text id="n9c3v6"
Large → 500 ms
```

but:

```text id="a1r7k4"
Small → 5 sec
```

You've simply moved the problem.

Always test the full parameter distribution.

---

# 48. Interview question

### "What is parameter sniffing?"

Strong answer:

> "Parameter sniffing is SQL Server's behavior of using parameter values available during compilation to help optimize a parameterized query. It can become problematic when different parameter values have very different data distributions and therefore require substantially different execution strategies."

---

# 49. Interview question

### "How would you diagnose parameter sniffing?"

Strong answer:

> "I'd look for inconsistent performance across parameter values, then use Query Store to identify multiple plans or significant runtime variation. I'd compare estimates and actual rows, inspect data distribution and statistics, and verify whether different parameter ranges genuinely require different access strategies."

---

# 50. Interview question

### "Would you use RECOMPILE immediately?"

Strong answer:

> "Not automatically. RECOMPILE can solve some parameter-sensitive cases by optimizing for the current parameter, but compilation has a cost. I'd first understand the workload and consider whether a more sustainable solution is appropriate."

---

# 51. Interview question

### "Is parameter sniffing always bad?"

Strong answer:

> "No. Parameter sniffing can improve performance because the optimizer can use actual parameter information to choose a more selective plan. It becomes a problem when the plan produced for one parameter value performs poorly for other values."

---

# 52. Expert mental model

Remember:

```text
Parameter
    ↓
Data distribution
    ↓
Cardinality estimate
    ↓
Cost calculation
    ↓
Plan choice
    ↓
Runtime behavior
```

If different parameters produce dramatically different cardinalities:

```text
Different parameters
       ↓
Different optimal plans
```

That is where parameter-sensitive behavior becomes important.

---

# 53. Lesson 143 takeaway

The key lesson is:

> **Don't ask whether parameter sniffing is "bad." Ask whether one execution plan can efficiently serve the actual distribution of parameter values.**

Your troubleshooting chain should be:

```text
Query Store
   ↓
Runtime variation
   ↓
Plan variation
   ↓
Parameter distribution
   ↓
Estimate accuracy
   ↓
Appropriate strategy
   ↓
Measure again
```

---
