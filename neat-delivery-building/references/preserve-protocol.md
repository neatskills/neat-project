# Preserve Protocol

Drift detection protocol - ensures implementation preserves original requirement intent.

## Purpose

Detect drift from original requirements: features added beyond spec, scope creep, missed behaviors.

## Scope

All implementation features (run at two checkpoints: before tasks start, after feature completes).

## Mental Model

See SKILL.md § Definitions for the Preserve definition (break down work without losing details, context, or intent).

## Checkpoint 1: Before Task Execution (Prevention)

Run this check before executing each task.

### Input

- Task spec (from wave file)
- Requirement (from wave file § Requirements table)

### Drift Detection (3-Item Checklist)

- Am I elaborating beyond what was asked? → NO required
- Am I inventing features not specified? → NO required
- Have I forgotten any requirement details? → NO required

**If any answer is YES or uncertain → STOP**: re-read requirement and revise task understanding before proceeding.

## Checkpoint 2: After Feature Completion (Verification)

Run this check after all tasks in a feature complete.

### Input

- Feature implementation (files written)
- Requirements for this feature (from wave file § Requirements table)
- Acceptance criteria (from wave file § Features § {feature})

### Verification Questions

1. Does implementation deliver what was asked, not what I thought was needed?
2. Did I add any features not in requirements?
3. Did I skip any required behaviors?
4. Did I change scope without noticing?
5. Are all acceptance criteria met?

### Verification Checks

**Feature-scoped tests:**
- Run all tests that cover this feature's code
- All must pass (catches cross-task regressions within the feature)

**Compare implementation against requirements:**
- Line-by-line check: requirement text vs implemented behavior
- Check for unintended changes (file diff vs task spec)
- Verify each acceptance criteria has evidence (test, file, behavior)

**Check for scope drift:**
- Any files written that weren't in task outputs?
- Any behavior implemented that wasn't in requirements?
- Any required behaviors missing from implementation?

### Output

- `deviations`: List of deviations found (structure per artifacts.md § progress.json)
- **Action:** Fix inline before marking feature done, OR escalate to planning if drift reveals spec gap

### Deviation Format

```json
{
  "type": "added_feature | missed_requirement | scope_change",
  "description": "What was added/missed/changed",
  "rationale": "Why it happened",
  "resolution": "fixed_inline | escalated | accepted"
}
```

## Error Conditions

**Drift detected (Checkpoint 2):**
- Record in deviations
- Fix before marking feature done
- If drift reveals spec gap → escalate to planning

**Missed acceptance criteria:**
- Fix before marking feature done
- Cannot mark feature complete with unmet criteria

**Scope changed unintentionally:**
- Revert to original scope OR
- Escalate to planning if the change reveals a spec issue
