# TDD Protocol

Test-Driven Development protocol for implementation and refactoring tasks.

## Purpose

Enforce test-driven development: tests from requirements BEFORE implementation.

## Scope

Implementation and refactoring tasks (not configuration, iac, cicd).

## Parameters

- `mode`: `implementation` | `refactoring`
  - `implementation`: Full TDD cycle (steps 1-6)
  - `refactoring`: Regression cycle only (step 6 before → modify → step 6 after)

## Input

- `test_source`: `acceptance_criteria` | `root_cause` | `task_spec` (varies by caller)
- `task_outputs`: Files to be implemented
- `task_type`: From delivery-rules.md (implementation | refactoring | configuration | iac | cicd)

**Input validation (blocker if fails):**
- If `mode = implementation` AND `test_source` is empty/missing → blocker: "Cannot derive tests without test source"
- If `task_type` not in [implementation, refactoring] → skip this protocol (delegate to task-type-specific rules)

## Steps

**If mode = refactoring:** Skip to step 6 (run existing tests before changes), then modify code, then step 6 again (run tests after). Done.

**If mode = implementation:**

### 1. Extract test requirements

From the test source:
- **acceptance_criteria**: What behavior must each criteria verify?
- **root_cause**: What invariant was broken?
- **task_spec**: What contract must hold?

### 2. Write test stubs

- Create test files/functions that will verify each requirement
- Use test names that map clearly to requirements
- Include edge cases, boundary conditions, error cases

### 3. Verify coverage

For each requirement, name which test(s) verify it:
- **Gap → blocker:** If any requirement lacks a test, cannot proceed
- **Ambiguity → blocker:** If requirement is untestable, cannot derive implementation

**Coverage mapping example:**
```
Acceptance criteria 1: "User can log in with email and password"
  → test_login_with_valid_credentials()
  → test_login_with_invalid_password()
  → test_login_with_nonexistent_email()

Acceptance criteria 2: "Session expires after 24 hours"
  → test_session_expires_after_24_hours()
  → test_session_valid_before_expiration()
```

### 4. Run tests (expect fail)

All new tests must fail (no implementation exists yet):
- **If any pass → broken stub:** Test doesn't actually verify new behavior; revise it
- Tests should fail with clear error messages indicating missing implementation

### 5. Implementation

Execute per task_type using delivery-rules.md § Task Types § Execution:
- **delivery-rules.md** is a shared protocol defining how to implement each task type
- Located at: [delivery-rules.md](delivery-rules.md)
- Implement minimal code to make tests pass (no gold-plating)

### 6. Run tests (expect pass)

All tests must now pass:
- **Failure → fix once, re-run:** Still failing → stop (blocker or failed status)
- Verify implementation meets all requirements from step 1

## Output

- Tests written and passing
- Implementation complete
- Coverage verified (each requirement → test mapping exists)

## Error Conditions

**Untestable requirement** → blocker with question for user:
> Requirement cannot be tested: "{requirement}"
> Context: {why it's untestable}
> [1] Clarify requirement
> [2] Escalate to planning

**Test stub passes before implementation** → broken stub:
> Test `{test_name}` passes before implementation.
> This means the test doesn't verify new behavior.
> Action: Revise test to actually verify the requirement.

**Tests fail after implementation** → implementation incomplete:
> Tests still failing after implementation:
> {failing test names and error messages}
> [1] Fix inline
> [2] Blocker (for subagent) or escalate (for main session)
