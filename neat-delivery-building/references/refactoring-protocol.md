# Standalone Refactoring Protocol

**Purpose:** Guide ad-hoc CRAP evaluation and refactoring outside the building workflow.

**Use case:** Quick cleanups when you spot messy code and want to refactor immediately. Supports rolling parallel execution for multiple files (max 5 concurrent agents).

**Not for:** Large-scale refactorings (50+ files) or systematic legacy cleanup — use the full NEAT workflow (assessing → solutioning → building) for those.

---

## When to Invoke

User says any of:

- "Refactor X" (file, class, module)
- "Check CRAP on X"
- "Clean up X"
- "Evaluate quality of X"
- "Fix complexity in X"

Where X is:

- **File:** `auth.py`, `user-service.ts`
- **Class:** `UserService`, `AuthController`
- **Module/Directory:** `src/domain/`, `adapters/`

---

## Execution Steps

**Step 1 — Resolve Scope:**

**Parse user intent:**

- Explicit path provided → use as-is
- Class name → find file containing class
- Directory → find all source files (exclude tests, configs, node_modules)

**Confirm scope:**

```
Refactoring scope: {scope_description}
Files: {N} files
  - {file1}
  - {file2}
  ...

Proceeding with autonomous refactoring:
  Method pass: rolling parallel, max 5 agents
  File pass: sequential
```

If >50 files → warn: "This is a large refactoring. Consider using the full building workflow for better tracking."

---

**Step 2 — Method pass (rolling parallel):**

**Goal:** Autonomous method refactoring with rolling parallel execution (max 5 concurrent agents).

**Load references:**

- [quality-gates.md § Method-Level CRAP Score](quality-gates.md § Method-Level CRAP Score)
- [quality-gates.md § File-Level Quality Check](quality-gates.md § File-Level Quality Check)
- [quality-gates.md § Refactoring Strategies](quality-gates.md § Refactoring Strategies)

**Rolling parallel execution:**

**If N ≤ 5 files:** Spawn one agent per file in parallel

**If N > 5 files:**

```
Refactoring {N} files with rolling parallel execution (max 5 agents at a time).

Queue: [file1, file2, ..., file{N}]
Active: max 5 agents at any time
Completion: as each agent finishes, start next from queue
```

**Algorithm:**

1. Create queue: `queue = [file1, file2, ..., fileN]`
2. Start first 5 agents: spawn agents for `queue[0:5]`, remove from queue
3. **When any agent completes:**
   - Aggregate its results
   - If queue not empty: spawn next agent from queue, remove from queue
   - Maintain concurrent count ≤ 5
4. Repeat step 3 until queue empty and all agents complete
5. Aggregate all results

**Agent prompt (autonomous per-file refactoring):**

````
Refactor all methods in {file_path} that have CRAP ≥ 15.

**For each method in the file:**

1. **Calculate complexity:** Count decision points (if, for, while, case, catch, ?, &&, ||, and, or) + 1
2. **Measure coverage:** Run coverage tool (pytest --cov / npm test --coverage), parse report
3. **Calculate CRAP:** complexity² × (1 - coverage)³ + complexity
4. **If CRAP ≥ 15:** Refactor this method

**CRITICAL - Correct measurement:**
- Complexity: Count decision points + 1, NOT LOC
- Coverage: Run tool and parse report, NOT hardcoded 0.0
- Sanity check: If all coverage = 0 or complexity = LOC → stop and investigate

**Auto-refactor flagged methods:**

For each method with CRAP ≥ 15, apply strategies SEQUENTIALLY until CRAP < 15:

**Root cause analysis:**
- Coverage < 0.5 → Coverage-driven: [Add tests, Guard clauses, Extract method, Simplify conditionals]
- Coverage ≥ 0.5 → Complexity-driven: [Guard clauses, Extract method, Simplify conditionals, Add tests]

**After EACH strategy:**
- Run tests → If fail: auto-revert, try next strategy
- Recalculate CRAP (re-run coverage, get new %)
- If CRAP < 15 → DONE, move to next method
- If CRAP improved but ≥ 15 → continue with next strategy
- If CRAP didn't improve → try next strategy

**Stop conditions per method:**
- CRAP < 15 (success)
- CRAP didn't improve after trying all strategies (stuck)
- All strategies exhausted (report final CRAP)

**Report results:**

Return structured report:

```jsonc
{
  "file": null,                // file path
  "methods_scanned": null,     // count
  "methods_refactored": [
    { "name": null, "crap_before": null, "crap_after": null, "status": null }   // status: done | stuck | exhausted
  ],
  "methods_clean": null        // count
}
```

Compare your report against this schema — confirm all required keys are present.

````

**Aggregate results after all agents complete:**

```

Method pass complete

Files processed: {N}
Methods scanned: {total}
Methods refactored: {count}

- Done (CRAP < 15): {count}
- Stuck (no improvement): {count}
- Clean (already < 15): {count}

Per-file results:
  file1.py: 3 methods refactored (CRAP 18→8, 22→11, 19→9), 2 clean
  file2.py: 2 methods refactored (CRAP 17→7, 21→12), 4 clean
  ...

```

---

**Step 3 — File pass analysis:**

**Goal:** Recalculate file metrics after method refactoring and tag files that still need refactoring.

**Recalculate file-level metrics:**

For each file:
1. **LOC:** Count lines of code in the file
2. **CRAP Sum:** Sum of all method CRAP scores (use post-refactoring scores from the method pass)
3. **Method count:** Count of methods/functions in the file

**Apply 2D thresholds:**

- LOC < 200 AND CRAP Sum < 50 → ✓ Clean file (skip)
- Otherwise → Tag for file-level refactoring

**Diagnosis:** Follow [quality-gates.md § File-Level Quality Check](quality-gates.md § File-Level Quality Check) § 2D Diagnosis Matrix.

**Report:**

```

File pass analysis

Files clean (no action needed): {count}

- file1.py: LOC 180, CRAP Sum 42 ✓
- file3.py: LOC 165, CRAP Sum 38 ✓

Files tagged for refactoring: {count}

  1. file2.py — Too big
     - LOC: 245 (threshold: 200)
     - CRAP Sum: 45 (under threshold)
     - Strategy: Split by responsibility
  
  2. file4.py — Too complex  
     - LOC: 190 (under threshold)
     - CRAP Sum: 62 (threshold: 50)
     - Strategy: Refactor top CRAP contributors (method_x: 18, method_y: 15)
  
  3. file5.py — Both problems
     - LOC: 280, CRAP Sum: 68
     - Strategy: Refactor methods first, then split

```

---

**Step 4 — File pass refactoring (sequential):**

**IMPORTANT: File-level refactoring is NOT safe for parallel execution.**

**Why sequential:**
- File splits create new files → potential naming conflicts
- Symbol extraction updates shared files → write conflicts  
- Import updates affect callers → merge conflicts

**For each tagged file from Step 3 (one at a time):**

**1. Prompt user per file:**

> File {N} of {total}: {file_path}
> Diagnosis: {diagnosis} — LOC {loc} (threshold: 200), CRAP Sum {sum} (threshold: 50), top CRAP contributors: {method_list}
> Recommended strategy: {strategy}
> Refactor this file?
> [1] Yes  [2] No  [3] Skip remaining

**2. If [1], apply file-level strategy:**

Follow [quality-gates.md § Refactoring Strategies](quality-gates.md § Refactoring Strategies) § Tier 2:

**For "file too complex" (CRAP Sum ≥ 50):**
- Refactor top CRAP contributors (methods with highest scores)
- Goal: reduce CRAP Sum < 50

**For "file too big" (LOC ≥ 200):**
- Analyze cohesion (do methods serve one responsibility?)
- If cohesive → keep and document reason
- If mixed → split by Single Responsibility Principle

**For "both problems":**
- Refactor complex methods first
- Then check if split still needed
- Split only if still LOC ≥ 200 after method refactoring

**3. Before changing public symbols:**

Find every usage across the repository (grep, string references, dynamic references).
Add callers' tests to test run.
Consider re-exporting at old path to avoid breaking callers.

**4. Re-run tests after each file:**

- File-specific tests
- Callers' tests (if public symbols changed)
- All must pass

**5. Continue to next file sequentially**

---

**Step 5 — Finalize:**

**Generate final summary:**

```

# Refactoring Complete

## Scope

Files processed: {N}

## Method Pass Results

Methods scanned: {total}
Methods refactored: {count}

- Done (CRAP < 15): {done_count}
- Stuck (no improvement): {stuck_count}
- Clean (already < 15): {clean_count}

Top improvements:

- {file}:{method}: CRAP {before} → {after}
- {file}:{method}: CRAP {before} → {after}

## File Pass Results

Files tagged for refactoring: {tagged_count}
Files refactored: {refactored_count}
Files skipped: {skipped_count}

File changes:

- {file1}: LOC {before} → {after}, CRAP Sum {before} → {after}
- {file2}: Split into {new_file1}, {new_file2}

## Final Metrics

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Avg CRAP | {A} | {B} | {improvement}% |
| Files over threshold | {X} | {Y} | {reduction} |
| Methods over threshold | {M} | {N} | {reduction} |

```

**Tests:** All tests passed (run throughout the method and file passes)

**Commit prompt:**

> Create commit?

```

Suggested message:
  Refactor: reduce CRAP in {scope}
  
  Method pass: {N} methods refactored
  File pass: {M} files restructured
  Avg CRAP: {before} → {after} ({improvement}% improvement)

```

---

## Integration with Building

**If user is in active building session:**

Check for any `specs/delivery/**/*-progress.json` with `wave.status = "building"`:

**Warning:**

> Active building session detected (feature: {name}). Refactoring now may conflict with in-progress work — finish the current feature first (building handles quality gates), or pause building, refactor, then resume.
> Proceed?

If user confirms → proceed but note potential conflicts.

---

## Key Differences from Building Quality Gates

| Aspect | Building (Integrated) | Standalone (Ad-hoc) |
|--------|----------------------|---------------------|
| **Timing** | After each task (auto) | On-demand (manual) |
| **Scope** | Feature tasks | User-defined (file/class/module) |
| **State** | Tracked in progress.json | No session state (debt goes to issues.md only if it exists) |
| **Passes** | Method & file in task flow | Two passes: methods first, then files |
| **Auto-refactor** | Task-level only | Method-level (autonomous per file) |
| **Parallelism** | Sequential (per feature) | Method-level: rolling parallel (max 5). File-level: sequential |
| **Resume** | Yes (via progress.json) | No (one-shot) |
| **Output** | build-history.md | Git commit (optional) |

---

## Edge Cases

Follow [refactoring-edge-cases.md](refactoring-edge-cases.md) for handling:
- No coverage tool available
- Tests fail after refactoring  
- Method stuck or strategies exhausted

