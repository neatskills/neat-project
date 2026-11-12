---
name: neat-delivery-building
description: Use when a planned wave is ready to build OR when resuming a building session from progress tracking.
neatskills:
  requires: neat-delivery-planning
---

# Building

**Role:** You are a software engineer who executes a planned wave by implementing task specs in dependency order — producing verified code and a build report.

## Overview

Structure of this run: plan load → [per wave: feature implementation in dependency order → regression tests → subagent review → knowledge update → history recording → runbook generation → finalization] → retro.

## When to Use

Run when:
- One or more waves have a complete, reviewed plan
- Resuming a building session from progress tracking
- Ad-hoc CRAP evaluation or refactoring of 1–5 files outside a wave build

Not for planning — use `neat-delivery-planning` for that. When execution hits issues that require reconsidering the problem beyond the immediate task scope, transition to `neat-delivery-fixing` (Reframe mental model).

## Configuration

### Workspace

Path variables — Follow [../references/neat-project/delivery-rules.md](../references/neat-project/delivery-rules.md). **Artifact formats:** [references/building-artifacts.md](references/building-artifacts.md).

| File | Operation |
| --- | --- |
| `specs/delivery/delivery.json` | `patch status` — wave only |
| `specs/delivery/issues.md` | `append` |
| `{base}/wave{NN}-{name}.md` | `read` — wave file containing requirements, features, and tasks (no writes, status tracked in progress.json) |
| `{base}/wave{NN}-{name}-progress.json` | `write` if absent; `patch wave`; `patch features.{feature}` |
| `{base}/runbook.md` | `write` if absent; `append` thereafter |
| `{base}/build-history.md` | `append` |
| `specs/system/system.md` | `append` — see Constraints |
| `specs/system/modules/{unit}.md` | `patch source_files` — see Constraints |
| `specs/system/modules/index.md` | `read — do not write` |
| `specs/discovery/modernization.md` | `read` (modernization track only — `legacy_path` frontmatter key) |
| `{kb}/decisions.md` | `read — do not write` |
| `specs/system/modules/shared-modules.md` | `read — do not write` |
| `specs/discovery/{prd,solution,requirements{NN}}.md` | `read — do not write` |
| `CLAUDE.md` | `read — do not write` (Phase 1 Step 4); `append` (Phase 9 retro, conventions-capture Step 2 — project conventions only) |
| `{base}/review.md` | `read — do not write` (Phase 4) |
| `{base}/wave{NN}-{name}-coordinator-state.json` | `write` — coordinator mode only; deleted at wave finalization |

**Artifact status:** Follow [../references/neat-project/artifact-status.md](../references/neat-project/artifact-status.md) for artifact status marker semantics.

**Constraints:**

- `system.md`: append only to `§ Foundation` `§ Architecture` `§ Contracts` — never `§ Code Structure` or `§ Conventions`
- `modules/{unit}.md`: never create a new module file. Unmapped files follow `delivery-rules.md` § Task Types § Execution (Module map update).
- `decisions.md`: read — never writes (planning is the authoritative writer)

**Session state (not files):**

- `touched_files`: paths written by implementation, cicd, and iac tasks; reset per wave

**Schema checks:** After writing any artifact, verify it against the relevant schema in `references/building-artifacts.md` — confirm all required keys are present. (`modules/{unit}.md` and `modules/index.md` schemas are defined in `../references/neat-project/shared-artifacts.md` — loaded on demand per Phase 1 Step 4.)

### Routing

Resolved in Phase 1 Step 1 and held as session state. `{wave}`, `{wave-file}`, `{progress-file}`, and `{coordinator-file}` are set in Step 2 when the user selects a wave or feature.

| Route | Options |
| --- | --- |
| `{base}` | `specs/delivery/` (shared paths) \| `specs/delivery/{product}/` (per-product paths) |
| `{kb}` | `specs/system/` (shared paths) \| `specs/system/{product}/` (per-product paths) |
| `{wave-file}` | `{base}/wave{NN}-{name}.md` — consolidated wave file containing requirements, features, and tasks |
| `{progress-file}` | `{base}/wave{NN}-{name}-progress.json` — batch-mode build state for `{wave}` |
| `{coordinator-file}` | `{base}/wave{NN}-{name}-coordinator-state.json` — coordinator-mode build state for `{wave}` |

### Definitions

- **Execution order:** task step numbers are canonical within a feature; feature `depends_on` fields are canonical across features.
- **Preserve** (this skill's mental model): break down the work without losing details, context, or intent. Used by the Preserve check (Phase 2 Step 4), Preserve verification (Phase 2 Step 5), and the final gate (Phase 4).
  - **Guard against:** drifting from original requirements. Elaborating beyond what was asked. Inventing features not requested. Forgetting details during decomposition. Losing sight of original intent.
  - **Mental check:** Am I preserving original intent?
- **Escalate to planning** (Phases 1–5): write an entry to `specs/delivery/issues.md` (create if absent) with `status: open`. This is the only durable channel to planning — a verbal note to the user does not survive to the next planning run.
  - Schema: `references/building-artifacts.md` § issues.md — load if not yet loaded.
  - ID: read existing entries first, then increment from the highest ISS-NNN.
  - `Type`: `defect` for wrong/missing behavior, `debt`/`gap` otherwise.
  - `Blocker`: Follow [references/building-artifacts.md](references/building-artifacts.md) § issues.md (Blocker field semantics).
  - Compare your output against the schema — confirm all required keys are present.
  - Planning's issue processing (Phase 2) picks it up and sets `status: addressed` once converted to a requirement (Phase 4).

## Quick Reference

**Progress.json update pattern** (after each task, Phase 2 Step 4):
1. Re-read progress.json
2. Append task slug to `features.{feature}.completed` (prefix `deferred:` for configuration, `iac:` for iac)
3. Update `features.{feature}.task` to next slug (or null if done)
4. For implementation/cicd/iac: append paths to `features.{feature}.touched_files`
5. Update `features.{feature}.last_updated`
6. Save (patch feature only)

**Feature claiming pattern** (Phase 1 Step 3):
1. Re-read progress.json
2. If `status: "in-progress"` + recent `last_updated` → race, offer other features
3. Write feature entry: `{status: "in-progress", task: "<first-slug>", completed: [], touched_files: [], deviations: [], quality: {methods: [], files: []}, last_updated: "<timestamp>"}`
4. Save

**Module Loading** (implementation tasks only):
1. Read task `files` (outputs)
2. For each output path, map to a module using longest prefix match against loaded modules' `path` fields
3. If the path maps to a module NOT in the loaded set → load `specs/system/modules/{module}.md`, add to loaded set
4. If no module match found → note for Phase 6 build-history Deviations

**Preserve mental check** (before each task):
- Am I elaborating beyond what was asked? → NO required
- Am I inventing features not specified? → NO required
- Have I forgotten any requirement details? → NO required
- If any answer is YES or uncertain → STOP, re-read requirement

**Placement rules** (implementation tasks, structure validation):
- Layer-based: Domain → domain-layer; Service → application-layer; Repository → infrastructure-layer; Controller → api-layer
- Feature-based: Feature code → feature-area/{component-type}; Shared → shared-code-path
- Parent directory missing → escalate to planning

**Conditional reference loading**:
- `refactoring-protocol.md` — when user selects refactor option
- `coordinator-protocol.md` — when user selects coordinator mode
- `execution-protocol.md` — loaded by coordinator-protocol (both main session and subagent modes)
- `preserve-protocol.md` — Phase 2 Step 5 only
- `knowledge-sync.md` — Phase 5 only
- `implementation-review.md` — Phase 4 only
- Modernization files — when `specs/discovery/modernization.md` exists

**Quality Gates** (see [quality-gates.md](references/quality-gates.md) for CRAP formula details and complete strategies):
- **Method-level threshold:** CRAP < 15 (run after each implementation task; CRAP ≥ 15 triggers auto-refactoring)
- **File-level thresholds:** LOC < 200 AND CRAP Sum < 50 (run after feature completes; exceeding either triggers file-level refactoring)
- **Complexity counting:** Each `if`, `for`, `while`, `case`, `catch`, `?`, `&&`, `||`, `and`, `or` = +1, then add 1
- **Auto-refactoring order:** Coverage-driven (cov < 0.5): [1] Add tests [2] Guard clauses [3] Extract method [4] Simplify conditionals; Complexity-driven (cov ≥ 0.5): [1] Guard clauses [2] Extract method [3] Simplify conditionals [4] Add tests
- **Storage:** Append to `quality.methods` and `quality.files` in progress.json; reported in build-history.md

## Phase 1: Setup

Populates the routing table, confirms the build wave, and pre-scans inputs for completeness.

**Step 1 — Detect and populate routing table:**

- Ad-hoc CRAP evaluation or refactoring of 1–5 files outside a wave build → skip the phases and Follow [references/refactoring-protocol.md](references/refactoring-protocol.md).

Otherwise read `specs/delivery/delivery.json`:

- Absent → stop: tell the user to run planning first.
- `planning_model` absent or unrecognised → stop: `delivery.json` is malformed — ask the user to check planning's output.
- `phase-first` (shared paths), single-product → `{base}` = `./specs/delivery/`, `{kb}` = `./specs/system/`.
- `phase-first` (shared paths), multi-product → list products, ask which. `{base}` = `./specs/delivery/`, `{kb}` = `./specs/system/`.
- `product-first` (per-product paths) → list products, ask which. `{base}` = `./specs/delivery/{product}/`, `{kb}` = `./specs/system/{product}/`.

Set `{wave-file}` per `### Routing` once `{wave}` is known from Step 2.

**Step 2 — Select wave or feature:** Follow [references/wave-selection.md](references/wave-selection.md) — loads decision matrix, topological sort rules, build mode selection, and user routing. Sets `{wave}`, `{wave-file}`, `{feature}` (if applicable). Outcome determines next step: feature selection → Step 3 (claim feature); new wave + batch mode → Step 3 (claim wave); new wave + coordinator mode → Step 4 then [coordinator-protocol.md](references/coordinator-protocol.md); refactor → [refactoring-protocol.md](references/refactoring-protocol.md) then return to Step 2; coordinator resume → Step 4 then [coordinator-protocol.md](references/coordinator-protocol.md).

**Step 3 — Claim:**

**Wave (new build):** Re-read delivery.json. If wave still `planned` → set `status=building`, patch entry. If claimed → tell user, pick next wave. Create/update progress-file per [building-artifacts.md](references/building-artifacts.md) § progress.json (absent → initialize wave state; exists → wave claimed, proceed to feature).

**Feature:** Re-read progress-file. If feature `status: "in-progress"` + recent `last_updated` (<2hrs) → race detected, re-run safety check, offer free features. If stale (>2hrs) → ask takeover. If free → write feature entry per Quick Reference (Feature claiming pattern), save (patch feature only).

**Step 4 — Load plan:** Auto-load: wave-file (requirements table + features metadata), system.md (all), modules/*.md per [delivery-rules.md](../references/neat-project/delivery-rules.md) § Module Loading, delivery-rules.md, CLAUDE.md. Session state: if modernization.md exists → read frontmatter, store `legacy_path`; initialize `touched_files` (empty or reconstruct from done features). On-demand refs per Quick Reference (Conditional reference loading).

**Step 5 — Pre-implementation validation:**

**(a) Decisions** (if `wave.pre_implementation.decisions_reviewed` absent):

1. Load `{kb}/decisions.md` (if absent, skip this check)
2. For each decision, check if it: adds requirement/constraint not in feature specs (gap), contradicts specs (conflict), or aligns (no issue)
3. **No gaps/conflicts:** Report "Checked {N} decisions — all align" and skip to step 6
4. **Gaps found:** For each gap, recommend "Fix inline" (implementation detail, no AC/output change) or "Escalate to planning" (changes AC, adds outputs, or uncertain)
5. **Conflicts found:** Recommend "Escalate to planning"
6. Record `wave.pre_implementation.decisions_reviewed = {timestamp}` in progress.json
7. Drop decisions.md and feature specs from context

**(b) Codebase inputs:** Verify `inputs` not in plan outputs exist; `{legacy_path}/` inputs are read-only; if in `outputs` → escalate.

Then proceed to Phase 2.

## Phase 2: Implementation

Implements each feature in dependency order, task by task, with unit tests gating each task before the next begins.

**Step 1 — Establish Structure Skeleton:** First wave only (skip if resuming) — Follow [structure-checks.md](references/structure-checks.md) § Skeleton Creation. Record `structure_skeleton_created = true`.

**Step 2 — Initialize and resume:** Load progress-file. Read `features.{current-feature}` for task/completed. Progressive feature loading — load current feature section from wave-file (full spec + tasks); load `depends_on` features for context. Resume from `features.{current-feature}.task`.

**Step 3 — Pre-execution checks (before first task):**

- **Naming alignment** (if config/iac/cicd tasks):
  - Load prior values (progress.json, runbook.md Values table)
  - Scan tasks for operator values
  - Resolve conflicts (user decides)
  - Record new values in `wave.pre_implementation.values`
  - Unknown → `{KEY}` placeholder
- **Open callouts:**
  - Grep `{open:` in feature section
  - Check `wave.pre_implementation.callouts`
  - If new → ask user [1] Resolve [2] Escalate
  - Record resolutions
- **Completeness:** If type/interface/signature cannot be derived → ask [1] Clarify [2] Escalate
- **Spec drift:** If task path/name conflicts with codebase → follow codebase, record in deviations, log `gap` to issues.md

**Step 4 — Execute:** Execute feature tasks in step order. **Implementation tasks** — Follow [tdd-protocol.md](../references/neat-project/tdd-protocol.md) (`mode=implementation`, `test_source=acceptance_criteria`), then [delivery-rules.md](../references/neat-project/delivery-rules.md) § Task Types § Execution. **Per task BEFORE execution** — **Module loading & Preserve check** (Quick Reference); **implementation only** — **Placement** (Quick Reference or [structure-checks.md](references/structure-checks.md) for complex cases). **Modernization** — `{legacy_path}/` files read-only, never write. **Questions** — follow [knowledge-protocol.md](../references/neat-project/knowledge-protocol.md) § Main Session.

**After each task:** **Implementation only — quality gates** — (1) run feature-scoped tests (all pass); (2) run scoped type/lint (tsc/eslint/mypy on feature paths only, not project-wide); (3) mutation adequacy per [mutation-adequacy.md](../references/neat-project/mutation-adequacy.md) (method-level in outputs); (4) method CRAP per [quality-gates.md](references/quality-gates.md) → append to `features.{feature}.quality.methods`. **Update progress.json** — per Quick Reference (Progress.json update pattern). **Legacy check** — verify no output has `{legacy_path}/` prefix (if found → escalate).

**Step 5 — Verify:** Follow [preserve-protocol.md](references/preserve-protocol.md) § Checkpoint 2 — run feature-scoped tests; check drift/missed criteria/unintended changes; fix inline; record deviations. **File-level quality** — per [quality-gates.md](references/quality-gates.md) → append to `features.{feature}.quality.files`. **Mark done:** re-read progress.json; set `features.{feature}.status = "done"`, `task = null`, update `last_updated`; save. **Finisher check:** Follow [wave-rules.md](references/wave-rules.md) § Finisher Check. Then proceed to Phase 3.

## Phase 3: Regression Tests

Run project-wide verification tests and validate wave structure after all features complete.

**Finisher only** (Phases 3-8 run once per wave by finisher; Phase 2 Step 5 sets `wave.status = "closing"` to block concurrent sessions). **Prepare wave-wide context** — union across features: `touched_files`, `quality`, `deviations`, `completed`.

**Step 1 — Verify:** Run project-wide checks (full test suite, type check, lint, integration) — first time since features built (Phase 2 ran scoped only). Follow [structure-checks.md](references/structure-checks.md) § Wave Structure Validation. **Failures** — single-task: fix/re-run (max 2 cycles) → if unresolved ask [1] Fix [2] Escalate [3] Accept; manual integration: ask [1] Fix [2] Accept [3] Escalate; contract: fix if code bug → if unresolved ask [1] Re-plan (stop build, log ISS-NNN) [2] Accept. Accept → record in deviations for Phase 6. Then proceed to Phase 4.

## Phase 4: Subagent Review

Spawn independent review subagent to verify implementation correctness and flag issues.

**Preserve final gate** — before spawning, verify you preserved original intent (no drift/elaborate/invent).

**Step 1 — Spawn:** Load `{base}/review.md` (absent = standard dimensions only). Load issues.md (open/accepted only). Follow [implementation-review.md](references/implementation-review.md) — fill paths: wave-file, touched-files, system.md, solution.md, issues.md. Instruct subagent to check drift. If review.md present → append additional passes/severity/exclusions.

**Step 2 — Review issues:** Write findings to issues.md per Escalate to planning (Definitions). Present grouped by Type: `defect` (fix inline → `accepted` "fixed inline"; out of scope → `open`), `debt`/`gap` (defer to planning; user accepts/dismisses → `accepted` with rationale; else `open`). Building writes `open` or `accepted` only, never `addressed`. Then proceed to Phase 5.

## Phase 5: Knowledge Update

Synchronize knowledge base with implementation changes and record batch summary.

**Step 1 — Batch summary:** If knowledge-protocol used in Phase 2 → Follow [knowledge-protocol.md](../references/neat-project/knowledge-protocol.md) § Batch Summary (`{checkpoint}` = "Wave complete").

**Step 2 — Knowledge sync:** Follow [knowledge-sync.md](references/knowledge-sync.md) — module rescan (on request), update system.md sections per `touched_files`, report divergences/gaps to issues.md, greenfield transition (first wave → brownfield mode). Then proceed to Phase 6.

## Phase 6: History Recording

Records build outcomes, quality metrics, and deviations to build-history.md.

Append to `{base}/build-history.md` (create if absent) per [building-artifacts.md](references/building-artifacts.md) § build-history.md — include Quality field rules, Architecture Changes, Deviations (union from features + unmapped notes). Verify schema. Then proceed to Phase 7.

## Phase 7: Runbook Generation

Generates runbook entries for deferred configuration and deployment steps.

**Step 1 — Wave runbook:** Use `completed` union (from Phase 3). Collect `deferred:{slug}` and `iac:{slug}` tasks. If none → skip. Append to `{base}/runbook.md` (create with header + Values table if absent) per [building-artifacts.md](references/building-artifacts.md).

**Step 2 — Build & Deploy + Verification:** Re-read delivery.json. Skip unless every other wave `status=built` (last wave). Create/update runbook: append Build & Deploy and Verification sections per [building-artifacts.md](references/building-artifacts.md) § runbook.md. Set `status: final` in frontmatter if written/updated.

Then proceed to Phase 8.

## Phase 8: Finalization

Marks wave complete and reports final status to the user.

**Mark wave complete:** Re-read progress-file → set `wave.status = "built"`, update `wave.last_updated`, save. Re-read delivery.json → set wave `status = "built"`, patch entry. **Stop rule (if session stops before finisher)** — no features started (progress.json absent/empty) → set wave `status=planned`; some features done/in-progress → keep `status=building`. **Exit:** Tell user then proceed to Phase 9. If runbook written/updated → "Wave complete. All features implemented, tests green. Configuration deferred to runbook — see `{base}/runbook.md`." Else → "Wave complete. All features implemented, tests green. No configuration required."

## Phase 9: Retro

Captures improvements for this skill based on execution experience.

Follow [retro-template.md](../references/neat-project/retro-template.md) § With Conventions Capture (`{skill-name}` = "neat-delivery-building").

Done.
