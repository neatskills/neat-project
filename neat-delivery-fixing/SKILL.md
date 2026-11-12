---
name: neat-delivery-fixing
description: Use when addressing issues with unclear root causes — change requests, reviewer findings, or bugs.
---

# Fixing

**Role:** You are a software engineer who resolves issues by stepping back to understand root causes before fixing — producing verified fixes that address the real problem, not just symptoms.

Structure of this run: input → **RESIST direct fix** → triage → step back → reframe → root cause → resolve → verify → retro.

## When to Use

Run when addressing change requests, code review findings, or bugs — anything requiring investigation. Skip for truly trivial changes (typo fix, constant update). Scope: one issue, one root cause, one fix — no knowledge synchronization (that's building's job).

## Configuration

**Reframe** (mental model): step back when new information emerges — don't fix symptoms blindly. **The trap:** input looks clear and actionable, so the instinct is to fix it directly — but "clear" information is often just a symptom. **Why it's hard:** input gives narrow scope; reframe demands big picture — why did this happen, what else is affected, what is the root cause. **Guard against:** symptom-fixing, taking "clear" information at face value, skipping context gathering, quick patches that miss related issues. **Mental check:** Have I reframed this as the real problem?

## Phase 1: Input Understanding

Recognize the input type (change request, reviewer finding, or bug) and resist the trap of fixing symptoms directly.

**Step 1 — Recognize input type and THE TRAP:**

Input types: **Change request** (user wants something changed), **Reviewer finding** (code review/scan flagged issue), **Bug** (unexpected behavior/test failure).

**THE TRAP** — consider big picture:

- Not "fix this bug" → "why did this happen? what else breaks this way?"
- Not "address this finding" → "what pattern allowed this? where else?"
- Not "implement this request" → "why changed? what assumptions broke?"

**Commitment:** Full reframing. No shortcuts.

Proceed to Phase 2: Step Back & Reframe.

## Phase 2: Step Back & Reframe

Gather context, resist the urge to fix immediately, reframe the problem to identify root cause, and decide on resolution scope.

**Step 1 — Quick context scan:**

Recent changes (`git log --oneline -10`, `git blame`), related issues (`specs/delivery/issues.md`), test coverage.

**Step 2 — Context expansion (resist the fix):**

**Change requests:** Why did requirements change? What assumptions broke? What else is affected? Read original requirements (`specs/discovery/prd.md` or `specs/discovery/requirements{NN}.md`), solution design (`specs/discovery/solution.md`), related features in delivery plan.

**Reviewer findings:** Why did code allow this? Where else could this pattern exist? Why didn't tests/reviews catch it earlier?

**Bugs:** What actually broke vs symptom? When (`git log`, `git bisect`)? What changed recently? Related instances/patterns? Read test suite for this area.

**Load context:** `specs/system/system.md` (all sections), `specs/system/modules/*.md` (per [../references/neat-project/delivery-rules.md](../references/neat-project/delivery-rules.md) § Module Loading), `../references/neat-project/delivery-rules.md`, `CLAUDE.md` (if present), requirement/solution files from above. **Questions:** Follow [../references/neat-project/knowledge-protocol.md](../references/neat-project/knowledge-protocol.md) § Main Session Usage.

**Step 3 — Reframe the problem:**

**Change request → What's the real need?** Is the stated request the actual solution needed? What problem is the user really trying to solve? Could there be a better approach given our architecture?

**Reviewer finding → Why did this happen?** What made this code seem correct when written? What assumption or pattern led to this? One-off or systematic issue?

**Bug → What actually broke?** Symptom vs root cause? What invariant was violated? What assumption proved false?

**Gate:** Articulate the reframed problem clearly — if not, gather more context or ask.

**Step 4 — Root cause analysis:**

**Questions:** Why did this happen (5 Whys)? What assumption broke? Where else could this occur? What tests/reviews would have caught this? Is this revealing a design gap?

**Evidence required:** Don't guess — find evidence, reproduce if it's a bug, trace failure path, identify exact divergence point.

**Step 5 — Scope decision:**

**Stay in fixing** if: root cause is code-level and manageable, fix can be verified with tests, no architectural changes.

**Escalate to solutioning** if: root cause reveals design flaws, architecture assumptions are wrong, requires rethinking component boundaries.

**Transition to building** if: resolution requires planned multi-step rework, multiple features need coordinated changes (runs under `neat-delivery-building`'s Preserve mental model).

**Escalate to assessing** if: change spans multiple systems.

**If escalating/transitioning:** Document findings in `specs/delivery/issues.md` per schema from `../neat-delivery-planning/references/planning-artifacts.md`, confirm all required keys present, proceed to Phase 4: Retro.

Otherwise, proceed to Phase 3: Resolve.

## Phase 3: Resolve

Design the fix addressing root cause, write tests first (TDD), implement comprehensively, verify resolution, and document lessons learned.

**Step 1 — Design the fix:**

What needs to change to fix root cause? How to prevent this class of issue? Identify files/functions, interfaces/signatures/integration points, related instances.

**Choice order (only after root cause is established):** Fix root cause once in shared code, not separately in each caller. Then prefer: helper/pattern already in codebase → standard library/platform → dependency already in manifest → new code. New dependency = design change (escalate per Phase 2 Step 5). Smallest *complete* fix: covers root cause and every related instance — small never means symptom patch.

**Design review:** Spawn subagent to independently review. Follow [references/design-review.md](references/design-review.md) — present derived design (root cause, files, interfaces, signatures, integration points, related instances).

**After subagent returns:** Present design and findings to user, flag decisions requiring input. Wait for explicit confirmation. If user requests changes, revise and re-present. If user stops/rejects, summarize unresolved and proceed to Phase 4: Retro.

**Step 2 — Write tests first (TDD):**

Follow [../references/neat-project/tdd-protocol.md](../references/neat-project/tdd-protocol.md) with `mode=implementation`, `test_source=root_cause`. **Test requirements:**

- Change request → test real need, not stated request
- Reviewer finding → test root cause violation, not symptom
- Bug → test broken invariant, not visible failure

**Step 3 — Implement the fix:**

Implement changes to fix root cause, fix all related instances, run tests (all must pass), **mutation adequacy** per [../references/neat-project/mutation-adequacy.md](../references/neat-project/mutation-adequacy.md) for changed methods.

**Step 4 — Verify comprehensive resolution:**

**Gate questions:** Did we fix root cause or just symptom? All related instances fixed? Will this prevent recurrence?

**Regression check:** Run existing test suite, verify no unintended side effects, check related functionality still works.

**Step 5 — Prevention & learning:**

**Document lessons:**

- Change request → what made requirements change, can we catch earlier, what assumptions to validate upfront, update `specs/system/system.md` if concepts changed
- Reviewer finding → why wasn't this obvious, what pattern to watch for, add to CLAUDE.md if project convention, strengthen tests/reviews
- Bug → what assumption broke, what class of issues, how to prevent similar bugs, update module documentation if behavior changed

**Prevention actions:** Add lightest guardrails (linting rules, type constraints, validation) that prevent recurrence, strengthen tests for this class, update documentation, log to `specs/delivery/issues.md` if reveals a pattern.

**Batch summary:** If knowledge-protocol queries used in Phase 2, follow [../references/neat-project/knowledge-protocol.md](../references/neat-project/knowledge-protocol.md) § Batch Summary with `{checkpoint}` = `Fix complete`.

Proceed to Phase 4: Retro.

## Phase 4: Retro

Reflect on the fixing process and generate retro output for skill improvement.

**Reflection prompts (fold into report):** Did I resist the trap and step back before fixing? Did I reframe the problem or just fix the symptom? Did I find root cause with evidence or guess? Did I fix comprehensively or just the visible issue? What would I do differently next time?

Follow [../references/neat-project/retro-template.md](../references/neat-project/retro-template.md) § Simple — `{skill-name}` = `neat-delivery-fixing`; `{execution-overhead}` = `Execution overhead: symptom-fixing without reframing, missed root cause, rework from insufficient context gathering`.

Done.
