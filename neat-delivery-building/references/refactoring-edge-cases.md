# Refactoring Edge Cases

Load on-demand when encountering issues during standalone refactoring.

---

## Case 1: No Coverage Tool Available

**Problem:** Can't calculate CRAP without coverage data

**WARNING:** This is LAST RESORT ONLY. Do NOT use manual estimation when a coverage tool is available.

**Why this is dangerous:**
- Hardcoding coverage = 0.0 makes well-tested methods look broken
- Flags 2-3× more methods than necessary
- Wastes refactoring effort on code that's already fine

**Solution (only when truly unavailable):**

1. **First, try to install coverage tool:**
   - Python: `pip install pytest-cov`
   - JavaScript: `npm install --save-dev jest` (has built-in coverage)
   - Go: built-in (`go test -cover`)
   - Java: `maven-surefire-plugin` with JaCoCo

2. **If installation blocked, manual estimation:**
   - Count decision points for complexity
   - Inspect tests for coverage estimate (0.3-0.7 range if tests exist)
   - Conservative: If no tests found → coverage = 0.0

3. **Document the limitation:**
   - Record in "Not Checked" section: "Coverage estimated without tooling"
   - Results may be overly pessimistic

---

## Case 2: Tests Fail After Refactoring

**Problem:** Applied strategy broke tests

**Solution:**

1. Show test failures
2. Auto-revert the last refactoring
3. Try next strategy OR
4. Ask user: "Continue debugging or skip this method?"

---

## Case 3: Method Stuck or Strategies Exhausted

**Problem:** CRAP still ≥ 15 after all strategies OR CRAP stopped improving

**Solution:**

1. Automatic continuation already tried all applicable strategies
2. Show final state: `{method}: CRAP {original} → {current} (exhausted strategies)`
3. Prompt per method:

   > {method}: CRAP still {score} after all strategies
   > [1] Accept current state (improved from {original})
   > [2] Accept as debt (record in issues.md if exists)
   > [3] Continue manually (guide me through next steps)
   > [4] Revert this method (keep other refactorings)

**Note:** This prompt appears only for stuck/exhausted methods. Methods that reached CRAP < 15 via automatic continuation don't need user input.
