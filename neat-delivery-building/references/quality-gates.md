# Quality Gates: Method & File Level

Two-tier quality gate system for implementation tasks with automated refactoring strategies.

**Contents:**

1. [Method-Level CRAP Score](#method-level-crap-score) — Task-level quality gate (after each implementation task)
2. [File-Level Quality Check](#file-level-quality-check) — Feature-level quality gate (after all tasks in a feature)
3. [Refactoring Strategies](#refactoring-strategies) — Auto-refactoring techniques

CRAP (Change Risk Anti-Patterns) measures complexity and test coverage risk. Results stored in progress.json during the build. Written to build-history.md when the wave completes (Phase 6).

**Quick Reference — CRAP formula and threshold:**
`CRAP(m) = comp(m)² × (1 − cov(m))³ + comp(m)` where `comp(m)` is cyclomatic complexity and `cov(m)` is decimal branch coverage (0.0–1.0). **Threshold: < 15** (strict inequality: CRAP < 15 passes, CRAP ≥ 15 triggers refactoring). See SKILL.md § Quick Reference § Quality Gates for at-a-glance rules.

---

## Method-Level CRAP Score

Run after EACH implementation task completes (Phase 2 Step 4 quality gate #4). Skip for `configuration`, `iac`, and `cicd` tasks.

### CRAP Formula

`CRAP(m) = comp(m)² × (1 − cov(m))³ + comp(m)`

- `comp(m)` = cyclomatic complexity
- `cov(m)` = decimal branch coverage (0.0–1.0)

**Counting complexity:**
- Each `if`, `for`, `while`, `case`, `catch`, `?`, `&&`, `||`, `and`, `or` = +1, then add 1
- Logical operators count per McCabe 1976 (compound predicates contribute complexity)
- If counting/estimation infeasible: treat as CRAP > 30 (conservative) and present blocker prompt

### Threshold

**CRAP < 15** passes (strict inequality: CRAP ≥ 15 triggers refactoring)

For each method created or modified:

- **CRAP < 15** → Pass silently; append to `quality.methods` with `decision: "accepted"`

- **CRAP ≥ 15** → Attempt auto-refactoring:
  1. Analyze root cause (coverage vs complexity)
  2. Apply refactoring strategies sequentially (see § Refactoring Strategies)
  3. Re-run tests after each strategy
  4. If CRAP < 15: mark `decision: "refactored"`
  5. If all strategies fail: prompt user (see § Refactoring Strategies § Tier 1 User Prompt)

**Strategy order by root cause:**

**Coverage-driven (coverage < 0.5):**
1. Add test coverage (target > 0.8)
2. Guard clauses
3. Extract method
4. Simplify conditionals

**Complexity-driven (coverage ≥ 0.5):**
1. Guard clauses
2. Extract method
3. Simplify conditionals
4. Add test coverage

**After each strategy:**
- Before code-changing strategy: if branches it touches are uncovered, first add characterization tests pinning current behavior
- Apply strategy, re-run feature-scoped tests
- Tests fail → auto-revert, try next strategy
- Tests pass → recalculate CRAP. CRAP < 15 → done. Otherwise → next strategy

**All strategies exhausted and CRAP ≥ 15:** prompt user with three options (see § Refactoring Strategies § Tier 1 User Prompt).

### Rationale

**Why single threshold:** Empirical data showed:
- CRAP < 15: maintainable
- CRAP ≥ 15: accumulates debt regardless of severity
- Auto-refactoring safe for moderate violations (15-30)
- Three-tier thresholds added complexity without value

**Why auto-refactor:** Immediate automated fixes prevent debt accumulation. Most violations (15-25) can be auto-fixed safely with guard clauses or test additions.

**Why task-level:** Immediate feedback prevents debt accumulation. Catching CRAP > 15 right after writing is easier than discovering 5 such methods at feature completion.

### Storage

After resolution, append to `features.{current-feature}.quality.methods` in progress.json (schema in building-artifacts.md § progress.json). Results written to build-history.md when wave completes (Phase 6).

---

## File-Level Quality Check

File-level quality gate applied after all tasks in a feature complete (Phase 2 Step 5 quality gate #5). Checks each file touched by implementation tasks using two independent dimensions.

### Metrics

For each file:

1. **LOC** = Lines of code in the file
2. **CRAP Sum** = Σ CRAP (sum of all method CRAP scores) — reuses method-level scores
3. **Methods** = Count of methods/functions in the file

### Two Independent Thresholds

**Threshold 1: LOC < 200** (file size)  
**Threshold 2: CRAP Sum < 50** (total complexity)

### 2D Diagnosis Matrix

| LOC | CRAP Sum | Diagnosis | Strategy |
|-----|----------|-----------|----------|
| < 200 | < 50 | Clean | None (skip prompt) |
| < 200 | ≥ 50 | File too complex | Refactor top CRAP contributors |
| ≥ 200 | < 50 | File too big | Check cohesion; split by responsibility if mixed |
| ≥ 200 | ≥ 50 | Both problems | Refactor → Split |

### Analysis & Prompt

**When LOC ≥ 200 OR CRAP Sum ≥ 50:**

> File `{file}` quality issue:
> • LOC: {loc} (threshold: 200)
> • CRAP Sum: {crap_sum} (threshold: 50)
> • Methods: {methods}
>
> Diagnosis: {diagnosis_message}
>
> Structure:
> Method breakdown (sorted by CRAP):
>
> 1. {method_1}: CRAP {score_1}
> 2. {method_2}: CRAP {score_2}
> ...
>
> Recommendations:
> {strategy recommendations based on diagnosis}
>
> [1] {primary recommendation}
> [2] {alternative strategy}
> [3] Accept current structure — write `debt` entry

**Diagnosis messages:**

- **LOC < 200, CRAP Sum ≥ 50:**
  - "File is too complex (methods have high CRAP scores)"
  - Primary: Refactor top CRAP contributors
  - Alternative: Remove duplication if near-copies detected

- **LOC ≥ 200, CRAP Sum < 50:**
  - "File is too big (many methods, but low complexity)"
  - Primary: Split by responsibility (SRP) if methods serve unrelated jobs
  - Alternative: Keep if cohesive (record reason)

- **LOC ≥ 200, CRAP Sum ≥ 50:**
  - "File is big AND complex"
  - Primary: Refactor complex methods first, then split file
  - Alternative: Split first, refactor after

### Rationale: Why 2D Thresholds?

**Advantages over combined formula:**
- Clear diagnosis: LOC tells size, CRAP Sum tells complexity
- Actionable strategy: Each dimension maps to specific fix
- Simple thresholds: 200 LOC, 50 CRAP Sum (easy to remember)
- Reuses data: LOC from codebase, CRAP from method-level checks
- No arbitrary weights: Each dimension stands alone

**Empirical validation:**
- LOC 200: Standard file size limit (many style guides)
- CRAP Sum 50: Validated from sibling project data
  - Clean files: 9-50 CRAP Sum
  - Problematic files before refactoring: 53-106 CRAP Sum
  - After refactoring: all under 50

### Storage

After resolution, append to `features.{current-feature}.quality.files` in progress.json (schema in building-artifacts.md § progress.json).

---

## Refactoring Strategies

Auto-refactoring techniques for method-level (Tier 1) and file-level (Tier 2) quality gates.

### Tier 1: Method-Level Refactoring

**Trigger:** CRAP ≥ 15 (after each implementation task)

**Approach:** Auto-refactor with sequential strategies, stop when CRAP < 15. Strategy order based on root cause (see § Method-Level CRAP Score § Threshold).

#### Root Cause Analysis

**Why coverage first when low:** Adding tests is safest (no code changes), makes subsequent refactorings safer, and may reduce CRAP enough without touching code.

**Characterization tests before changing code:** Before guard clauses, extract method, or simplify conditionals, check branches it touches. If any uncovered, first pin current behavior (including errors) with tests — even in complexity-driven order.

#### Strategy 1: Add Test Coverage

**When:** Coverage < 0.5 (primary) or as fallback (secondary)

**Technique:** Run coverage tool with branch coverage; find uncovered decision points. Add one test per uncovered branch (edge cases: null/empty/boundary; negative: invalid inputs). Target coverage > 0.8 (80%) — may reduce CRAP below 15 alone.

#### Strategy 2: Guard Clauses

**When:** Complexity-driven (coverage ≥ 0.5) or as secondary

**Technique:** Convert nested conditions to early returns

**Impact:** Reduces complexity by 1-3 points per nested level removed

#### Strategy 3: Extract Method

**When:** Complexity-driven (coverage ≥ 0.5)

**Technique:** Extract cohesive code blocks (e.g., validation/payment/inventory steps) into separate methods

**Impact:** Distributes complexity; each extracted method typically CRAP < 10

#### Strategy 4: Simplify Conditionals

**When:** Complex boolean logic

**Techniques:**
1. **Extract boolean to named variable** — assign compound condition to descriptive name, test the name
2. **De Morgan's laws** — rewrite negated compounds (`not (a and b)` → `not a or not b`)
3. **Replace nested ternary** — use if/elif/else chain

**Impact:** Reduces complexity by 1-2 points per simplification

#### Tier 1 User Prompt (After All Strategies Fail)

**When:** All auto-refactoring strategies exhausted and CRAP ≥ 15

> Method `{method_name}` in `{file}` has CRAP {score}:
> • Complexity: {complexity}
> • Coverage: {coverage}
>
> Auto-refactoring failed:
> ✗ Add test coverage — {reason}
> ✗ Guard clauses — {reason}
> ✗ Extract method — {reason}
> ✗ Simplify conditionals — {reason}
>
> [1] Manual refactor now
> [2] Refactor later — write `debt` entry (ISS-NNN)
> [3] Accept as-is — architecture requires this complexity

**Options:** (1) Manual refactor now: refactor inline, re-run tests, verify CRAP < 15; (2) Refactor later: write `debt` entry (ISS-NNN), mark `decision: "accepted"`; (3) Accept as-is: architecturally necessary (e.g., state machine, parser), mark `decision: "accepted"`

---

### Tier 2: File-Level Refactoring

**Trigger:** LOC ≥ 200 OR CRAP Sum ≥ 50 (after all tasks in a feature complete)

#### For "File Too Complex" (CRAP Sum ≥ 50)

Identify top CRAP contributors (methods sorted by CRAP descending). Apply Tier 1 strategies to top N methods. Recalculate after each refactoring. Goal: CRAP Sum < 50.

#### For "File Too Big" (LOC ≥ 200)

Analyze method groups by responsibility — if cohesive, keep and record reason. Otherwise, split by SRP: extract cohesive modules to new files. Find all usage of moved symbols repo-wide (grep or AST); update imports (or re-export at old path temporarily); run callers' tests. Goal: each new file has one responsibility.

#### For "Both Problems" (LOC ≥ 200 AND CRAP Sum ≥ 50)

**Refactor → Split:** Refactor complex methods first (reduces CRAP Sum, clarifies responsibilities), then split file by responsibility (reduces LOC). Verify each new file: LOC < 200 AND CRAP Sum < 50. Re-run feature-scoped tests. Under thresholds → proceed. Still over → offer to continue or accept with `debt` entry.

---

## See Also

- building-artifacts.md § progress.json — Quality storage schema
- building-artifacts.md § build-history.md — Quality reporting format
