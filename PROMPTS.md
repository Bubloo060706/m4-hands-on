# Module 4 – AI Debugging and Refactoring

**Course:** CS5013 – Programming with AI  
**Student:** RATHAN BALAJI P R  
**Roll No:** CS26E009  
**Repository:** m4-hands-on  
**Branch:** `feat/m4-ai-debug-refactor`

---

# Session 4A – Debugging with AI

## Part A – Stack Trace Investigation

### Initial failing test

The initial test run found 10 tests:
- 9 successful
- 1 failed

The failing test was `quotesDiscountForLoyalCustomer()`.

The important failure was:

```text
5-year loyalty customer should get 10% off (tier 3)
Expected: 90.0
Actual: 95.0
```

The failure occurred at `PriceEngineTest.java:92`.

## Stack Trace Investigation Prompt

```text
Explain this Java stack trace in plain English. What is the most likely root cause? Do NOT propose a patch yet.
```

### AI diagnosis

The progressive stack-trace investigation identified an off-by-one error in the loyalty-tier boundary. A customer with exactly 5 loyalty years was being treated as belonging to the 3–4 year tier and receiving a 5% discount instead of the expected 10% discount.

The most specific diagnosis was:

> The test itself ran normally, but its assertion failed... root cause is an off-by-one error in loyalty-tier condition. The code checks years > 5 for the 10% tier, so exactly 5 years falls through to 5% tier (years >=3).

### My assessment

I agree with the AI's diagnosis because the expected 10% discount for exactly 5 loyalty years is consistent with the failing test, while the current boundary condition sends 5 years to the 5% tier.

The full stack trace is useful for locating the failure, but the assertion message and the relevant test/source lines were enough to identify this particular root cause.

---

## Part B – Minimal Bug Fix

### Root cause

The 10% loyalty tier was selected only when:

```java
years > 5
```

Therefore, a customer with exactly 5 loyalty years incorrectly received the 5% discount instead of the intended 10% discount.

### Prompt

```text
Given root cause: The 10% loyalty tier is selected only when years > 5, so a customer with exactly 5 loyalty years incorrectly falls into the 5% tier. Write the smallest patch that fixes the bug without changing behaviour on any other test. Show only the diff.
```

### AI-proposed patch

```diff
-        if (years > 5) {
+        if (years >= 5) {
```

The patch was reviewed before being applied.

### Verification

```text
10 tests found
10 tests successful
0 tests failed
```

### Commit

```text
Fix loyalty discount boundary at five years

Why: Customers with exactly five loyalty years were incorrectly receiving 5% instead of the intended 10% discount.
```

Commit: `1bd7958 Fix loyalty discount boundary at five years`

## Part C – 4A Reflection

The most useful AI response was the final diagnosis because it connected the expected/actual values directly to the `years > 5` condition in `PriceEngine.java`. The least useful response was the initial short-stack-trace analysis because it had less information about the assertion and source location. A complete stack trace is not always necessary when the assertion message and relevant source/test lines already identify the failure. However, the additional stack-trace lines helped confirm the exact test location and expected-versus-actual values while the framework frames themselves added mostly contextual information.

---

# Session 4B – Refactoring with AI

## Part A – Refactoring Priority

### Prompt

```text
Rank the methods in this file by refactoring priority.
For each, give:
(a) current cyclomatic complexity estimate,
(b) the single refactor move you would apply first (extract-method, flatten-guard-clauses, replace-loop-with-pipeline, or rename),
(c) one sentence on the expected complexity reduction.
Do not propose code yet.
```

### AI response

```text
| Priority | Method | Complexity estimate | First refactor move | Expected reduction |
|---|---|---:|---|---|
| 1 | `quote` | ~14-18 | `extract-method` | Extracting the loyalty-tier calculation would remove roughly 3 decision points from `quote`, reducing its top-level complexity while isolating the pricing rule. |

`quote` is the only method in this file, so it is the sole refactoring target. The complexity range reflects whether compound boolean conditions are counted as separate predicates.
```

---

## Part B – Loyalty Discount Refactoring

### Prompt

```text
Extract the loyalty-tier discount block into a private helper method ‘applyLoyaltyDiscount(...)‘. Do NOT change public method signatures. Show the diff.
```

The loyalty calculation was extracted into the private helper:

```java
private static Money applyLoyaltyDiscount(
        Money running,
        Money subtotal,
        Customer customer)
```

The main method delegates the loyalty calculation through:

```java
running = applyLoyaltyDiscount(running, subtotal, customer);
```

The loyalty thresholds were preserved:

```java
int years = customer.loyaltyYears();
double loyaltyRate;

if (years >= 5) {
    loyaltyRate = 0.10;
} else if (years >= 3) {
    loyaltyRate = 0.05;
} else if (years >= 1) {
    loyaltyRate = 0.02;
} else {
    loyaltyRate = 0.0;
}
```

The public method signature was not changed.

### Verification

```text
10 tests found
10 tests successful
0 tests failed
```

Commit: `4b1ffee Refactor loyalty discount calculation`

---

# Part C – SpotBugs

## SpotBugs finding

SpotBugs reported:

```text
M V EI2: new Order(long, List, boolean, String) may expose internal representation by storing an externally mutable object into lines
At Order.java:[line 4]
```

A related finding was:

```text
M V EI: Order.lines() may expose internal representation by returning lines
At Order.java:[line 4]
```

### Explanation prompt

```text
Explain this SpotBugs finding in plain English and identify the smallest safe patch:

M V EI2: new Order(long, List, boolean, String) may expose internal representation by storing an externally mutable object into lines at Order.java:[line 4]

Do not propose a patch yet. Explain:
1. What the warning means.
2. Why it can cause a real bug.
3. What part of Order.java is responsible.
4. What the smallest safe fix would need to accomplish.
```

### AI explanation

1. `Order` stores the caller's `List<Line>` directly, so external code can mutate the list after the `Order` has been created.
2. This can cause the order contents to change unexpectedly after construction.
3. The record component `List<Line> lines` uses the generated constructor/accessor, which stores and returns the same list.
4. The smallest safe fix is a defensive unmodifiable copy of the list. Protecting the stored list also protects the accessor from exposing a mutable internal representation.

### Patch prompt

```text
Given that diagnosis, write the smallest safe patch for the SpotBugs finding.

Requirements:
- Prevent external mutation of Order.lines().
- Do not change the public API or record components.
- Preserve the existing behavior of Order.
- Show only the diff.
```

### Applied patch

```java
public Order {
    lines = lines == null ? null : List.copyOf(lines);
}
```

This prevents external mutation of the stored list while preserving the existing record component and public API.

### Verification

```text
10 tests found
10 tests successful
0 tests failed
```

SpotBugs was rerun with the JUnit dependency on the auxiliary classpath and reported no remaining findings.

Commit: `99a30c3 Fix SpotBugs mutable Order representation`

---

# Part D – Rejected Refactoring

### Prompt

```text
We already refactored the loyalty-tier discount calculation into the private helper applyLoyaltyDiscount(...).

Suggest ONE alternative refactoring for the current PriceEngine.java that could improve readability or reduce complexity, but that you would NOT recommend applying in this assignment.

Do not modify the code.
Explain:
1. What the proposed refactor is.
2. Why it could be useful.
3. Why it may be inappropriate or unnecessary here.
```

### AI response

> 1. **Proposed refactor:** Extract the regional tax logic into a private helper such as `applyRegionalTax(...)`.
>
> 2. **Why useful:** It would isolate the region-to-tax-rate decisions, make `quote(...)` read more like a sequence of pricing steps, and reduce its conditional complexity.
>
> 3. **Why inappropriate here:** The assignment focuses on the loyalty-tier refactor, and the tax block is short and already clear. Extracting it would expand the change without addressing a current defect or requirement, increasing scope and review overhead.

### Decision

I rejected this refactoring because the assignment specifically required the loyalty-tier discount extraction. The regional tax block was already short and understandable, so extracting it would increase the scope of the change without addressing a current requirement or defect.

---

# PMD Complexity Check

PMD was run to inspect the cyclomatic complexity of `quote()`.

The post-refactoring result was:

```text
src\PriceEngine.java:14: CyclomaticComplexity:
The method 'quote(Order, Customer)' has a cyclomatic complexity of 19.
```

Therefore:

```text
quote() cyclomatic complexity after refactoring = 19
```

The PMD result did not show a numerical reduction after the refactoring, so no reduction is claimed. The refactoring was evaluated through the requested structural change and full test-suite verification.

---

# Final 4B Reflection

The loyalty-tier extraction made the pricing method more modular by moving the loyalty-specific decision logic into a private helper while preserving the public API. The SpotBugs finding required understanding defensive copying because storing the caller's mutable list directly could allow the `Order` state to change unexpectedly; the `List.copyOf(...)` fix addresses that issue without changing the record component. I rejected the proposed extraction of the regional tax logic because it was outside the required refactoring scope and the existing block was already short and clear. PMD reported a cyclomatic complexity of 19 for `quote()` after the refactoring, so I did not claim a numerical complexity reduction that was not demonstrated by the tool.

---

# Final Verification

Git history:

```text
99a30c3 Fix SpotBugs mutable Order representation
4b1ffee Refactor loyalty discount calculation
1bd7958 Fix loyalty discount boundary at five years
e9e5ab1 Initial Module 4 starter
```

Final test result:

```text
10 tests successful
0 tests failed
```

Branch:

```text
feat/m4-ai-debug-refactor
```

Pull Request:

```text
Module 4 – AI Debugging and Refactoring
```
