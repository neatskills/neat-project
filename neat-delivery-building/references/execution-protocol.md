# Execution Protocol

**Role:** Build features from a wave — producing verified code and reports.

**Execution model:**
- **Main session mode:** Building directly in coordinator (one batch in round)
- **Subagent mode:** Autonomous execution in isolated worktree

**Never (when running as subagent):**

- Write `progress.json`, `delivery.json`, `issues.md`, `coordinator-state.json`, `specs/system/**`, or any spec file — report instead.
- Write files outside your tasks' `files` and the tests for them.
- Run project-wide tests, type checks, or lints — the coordinator runs those as finisher.
- Write under `legacy_path` when it is set.
- Create git commits — you run in an isolated worktree; the coordinator merges your files back to main uncommitted.

## Step 1 — Load

**Main session mode:**

- Context already loaded from coordinator Phase 1 Step 1
- Features from `coordinator-state.json` batch

**Subagent mode:**

- Parse "Subagent manifest:" from your prompt (see [subagent-manifest.md](subagent-manifest.md))
- Read every path in manifest `rules` and `knowledge`, plus `claude_md` if set
- Initialize `touched_files` session state (empty)

## Step 2 — Build Features

For each feature (manifest order for subagent; batch order for main session), run its tasks in step order, starting at `resume_from` (skip tasks before it).

### 2.1 — Progressive Feature Loading

**Main session mode:**

- Feature sections already loaded in coordinator Phase 1 Step 1

**Subagent mode:**

- Read feature's `### {name}` section in `wave_file § Features`, plus sections named in `depends_on`
- Read `wave_file § Requirements` rows for the feature's requirement IDs
- Load `solution_path`
- Task touching a module not in `knowledge.modules` → read `specs/system/modules/{module}.md` (longest prefix match per SKILL.md § Quick Reference § Module Loading)

### 2.2 — Pre-execution Checks (before first task)

Follow SKILL.md Phase 2 Step 3 § Pre-execution checks with mode-specific handling:

**Naming alignment** (config/iac/cicd tasks):
- Main session: records in `wave.pre_implementation.values`
- Subagent: substitutes from manifest `pre_implementation.values`

**Open callouts**:
- Main session: asks user [1] Resolve [2] Escalate
- Subagent: substitutes from manifest `pre_implementation.callouts`, unresolved → blocker (§ 2.5)

**Completeness**:
- Main session: asks [1] Clarify [2] Escalate
- Subagent: blocker (§ 2.5)

**Spec drift**:
- Both modes: follow codebase, record in deviations
- Main session: writes issues.md
- Subagent: records in feature `issues` list

### 2.3 — TDD Phase (BEFORE executing implementation tasks)

Follow [tdd-protocol.md](tdd-protocol.md) with `mode=implementation`, `test_source=acceptance_criteria`.

**When TDD protocol blocker triggers:**

- **Main session:** follow protocol's user prompt (typically [1] Clarify [2] Escalate)
- **Subagent:**
  - Set feature `status: blocked`
  - Set `blocker.task` = current task slug
  - Set `blocker.question` = blocker message from protocol
  - Set `blocker.scope` = "feature"
  - Continue with next feature in batch

### 2.4 — Execute Implementation Tasks

Per task, Follow [delivery-rules.md](../../references/neat-project/delivery-rules.md) § Task Types § Execution for its type.

**BEFORE each task:**

- **Module loading** (implementation tasks): Follow SKILL.md § Quick Reference § Module Loading
- **Preserve check**: Follow SKILL.md § Quick Reference § Preserve mental check
- **Placement** (implementation tasks): Follow SKILL.md § Quick Reference § Placement rules OR [structure-checks.md](structure-checks.md) § Placement for complex cases
- **Modernization:** `{legacy_path}/` files read-only (never write)

**Decision application (subagent mode only):**

- Apply every `decision_log` answer from manifest

**User questions and blockers:**

When delivery-rules.md says to ask or surface something to the user (missing input, undecidable shape, unresolved callout, second structural-gate failure, new dependency):

- **Main session:** prompt user immediately per delivery-rules.md; record answers in coordinator state
- **Subagent:**
  - Stop this feature with `status: blocked`
  - Set `blocker.task` = current task slug
  - Set `blocker.question` = the question
  - Set `blocker.context` = relevant context
  - Set `blocker.scope` = "feature"
  - Set `resume_from` to current task
  - Continue with next feature in batch

**Module map and issues:**

- **Main session:**
  - Patch `modules/{unit}.md` per delivery-rules.md § Module map update
  - Write to issues.md per building SKILL.md § Escalate to planning
- **Subagent:** add unmapped files and any `gap`/`debt`/`defect` to feature's `issues` list (report in Step 4)

### 2.5 — Quality Gates (after each implementation task)

Both modes run these 4 checks:

1. **Feature-scoped tests** — all must pass; fix failures before proceeding
2. **Scoped type/lint** (tsc, eslint, mypy on this feature's files only — not project-wide)
3. **Mutation adequacy** — Follow [../references/neat-project/mutation-adequacy.md](../references/neat-project/mutation-adequacy.md) on each method created/changed
4. **Method CRAP check** — Follow [quality-gates.md § Method-Level CRAP Score](quality-gates.md § Method-Level CRAP Score)

**When method CRAP user prompt would trigger (all strategies exhausted):**

- **Main session:** present user prompt per quality-gates.md § Method-Level CRAP Score
- **Subagent:**
  - Record method with `decision: "pending"` in feature's `quality.methods`
  - Continue

**Test failures:**

- Both modes: fix once and re-run
- **Main session:** still failing → present user with [1] Fix [2] Escalate [3] Accept partial
- **Subagent (still failing):**
  - Stop this feature with `status: failed`
  - Set `failure.task` = current task slug
  - Set `failure.error` = "tests failed after fix attempt"
  - Set `failure.details` = failing tests and output
  - Set `resume_from` to current task
  - Continue with next feature

### 2.6 — Task Completion Tracking

**Main session mode:**

- Re-read coordinator-state.json
- Append completed task to feature's `completed` with building's prefixes (`deferred:{slug}`, `iac:{slug}`)
- Append implementation/cicd/iac task written files to feature's `touched_files`
- Update feature's `last_updated`
- Save coordinator-state.json (patch feature only)

**Subagent mode:**

- Append completed task to feature's `completed` (in report state)
- Append written files to feature's `touched_files` (in report state)

## Step 3 — Verify

For each feature that finished all tasks:

### 3.1 — Preserve Verification

Follow [preserve-protocol.md](preserve-protocol.md) § Checkpoint 2:
- Run feature-scoped tests
- Check drift/missed criteria/unintended changes
- Fix inline
- Record deviations

**Feature-level test failures:**

- **Main session:** follow preserve-protocol.md user prompts
- **Subagent:** set `status: failed` per § 2.5 Test failures

### 3.2 — File-Level Quality Check

Follow [quality-gates.md § File-Level Quality Check](quality-gates.md § File-Level Quality Check) for each file touched by implementation tasks.

**When user prompt would trigger (file over threshold):**

- **Main session:** present user prompt per quality-gates.md § File-Level Quality Check
- **Subagent:** record in feature's `quality.files` with `decision: "pending"` instead of prompting

### 3.3 — Finisher Check (main session only)

**Main session:** Follow [wave-rules.md](wave-rules.md) § Finisher Check

**Subagent:** skip (coordinator runs finisher phases, not subagent)

## Step 4 — Report

**Main session mode:**

- Update coordinator-state.json:
  - Set feature `status: done`
  - Final `last_updated` timestamp
- Save coordinator-state.json
- Continue to next feature in batch

**Subagent mode:**

- Return the report as your final message — nothing else
- Compare against schema — confirm all required keys are present for every feature in the batch
- Do not commit

### Report Schema (Subagent Mode Only)

```jsonc
{
  "features": {
    "auth": {
      "status": "done",              // done | blocked | failed
      "completed": ["auth-001", "deferred:config-001", "iac:infra-001"],
      "touched_files": ["src/auth/service.ts"],
      "quality": {                   // progress.json quality schema; decision "pending" = coordinator asks the user
        "methods": [],
        "files": []
      },
      "deviations": [],
      "issues": [                    // coordinator writes these to issues.md
        { "type": "gap", "description": "no module found for `src/x.ts` — verify solutioning completed for this track" }
      ],
      "resume_from": null,           // task slug to resume from when blocked or failed; null when done
      "blocker": null,               // when blocked: { "task": "slug", "question": "...", "context": "...", "scope": "feature | cross-feature" }
      "failure": null                // when failed: { "task": "slug", "error": "...", "details": "failing tests and output" }
    }
  }
}
```

**Status rules:**

- `done`: all tasks completed, pass criteria met
- `blocked`: stopped at a decision that requires user input
- `failed`: tests failed after fix attempt OR unrecoverable error

**resume_from:** null when done, task slug when blocked/failed (enables coordinator to resume)
