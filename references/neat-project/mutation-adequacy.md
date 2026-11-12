# Mutation Adequacy

Used by building and fixing. The calling skill defines which methods are in scope.

For each in-scope method, check the three mandatory classes:

| Class | Example mutation | Gate |
|---|---|---|
| Negated condition | `if (x > 0)` → `if (x <= 0)` | A test must fail when the condition is negated |
| Off-by-one boundary | `i < n` → `i <= n` | A test at the exact boundary value must exist |
| Null / empty return | return value → return `null` or `[]` | A test must assert on the return value, not merely call the method |

If a class has no applicable mutation site in a method (e.g. no conditional logic), document it as "not applicable" and proceed.

For each uncaught mutation:

- a. Write the missing test targeting the expected behaviour
- b. Apply the mutation to the source
- c. Run the new test — it **must fail**. If it passes, the test is wrong; delete it and return to (a).
- d. Revert the mutation
- e. Run the test suite (building: the feature test suite; fixing: the full test suite) — all tests must pass

No accept-with-rationale escape — an uncaught mutation is always a correctable test gap.
