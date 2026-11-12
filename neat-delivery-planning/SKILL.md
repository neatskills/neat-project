---
name: neat-delivery-planning
description: Use when requirements and a solution direction are defined — breaks them into features and tasks.
neatskills:
  requires: neat-discovery-solutioning
---

# Planning

**Role:** Product owner breaking down requirements into features and tasks in dependency order — producing build-ready specs.

Structure of this run: workspace setup → issue processing → requirements loading → extraction → wave selection → feature breakdown → task breakdown → wrap-up → retro.

Run when requirements and solution direction are available. Hand off to `neat-delivery-building` after plan reviewed.

## Configuration

### Workspace

Follow [../references/neat-project/delivery-rules.md](../references/neat-project/delivery-rules.md) — artifact formats: [references/planning-artifacts.md](references/planning-artifacts.md).

| File | Operation |
| --- | --- |
| `specs/delivery/delivery.json` | `write`; `patch status` on subsequent runs |
| `specs/delivery/issues.md` | `write` if absent; `patch status` open→addressed; `append` new gap entries |
| `{base}/wave{NN}-{name}.md` | `write` — consolidated wave file containing requirements and features |
| `{base}/plan.md` | `write` — cross-wave summary |
| `{kb}/decisions.md` | `write` |
| `{kb}/adrs/adr-{NNN}-{slug}.md` | `write` — cross-scope architectural decisions only |
| `specs/discovery/{prd,solution,requirements{NN}}.md` | `read — do not write` |
| `specs/discovery/modernization.md` | `read` ({track} = modernization only) |
| `specs/system/system.md` | `read — do not write` |
| `specs/system/modules/*.md` | `read — do not write` |
| `{base}/build-history.md` | `read` (written by building; read in Phase 5 for audit staleness) |
| `CLAUDE.md` | `read — do not write` |
| `{base}/review.md` | `write` if absent (Phase 1 Step 1); `read — do not write` thereafter |

**Concurrent sessions:** re-read `delivery.json` before each write — patch only your entry. Never edit waves with `status=building|built` — defer to later wave or log as `gap` in `issues.md`.

**Artifact status:** Follow [../references/neat-project/artifact-status.md](../references/neat-project/artifact-status.md). **Schema checks:** verify every artifact write against [references/planning-artifacts.md](references/planning-artifacts.md).

### Routing

Resolved Phase 1 Step 1 — held as session state.

| Route | Options |
| --- | --- |
| `{track}` | `standard` \| `modernization` |
| `{base}` | `specs/delivery/` \| `specs/delivery/{product}/` |
| `{kb}` | `specs/system/` \| `specs/system/{product}/` |
| `{product_col}` | `yes` (tables include a `Product` column) \| `no` |

### Definitions

- **Escalate:** stop and present conflict for user decision — triggers: unmatched title, cycle, circular `depends_on`, cross-wave task reference, mid-execution cross-feature dependency, subagent review gap.
- **No-discovery stop:** `No discovery documents found in ./specs/discovery. Run neat-discovery-assessing first.` — when required discovery artifact absent.

### Checkpoints

**Resume-or-reset:** when partial output found — `{context} Resume or reset? [1] Resume (Recommended: preserves work) [2] Reset` (phase: Phase 1 Step 2; wave: Phase 5 Step 1).

## Phase 1: Setup

Detect project structure (track, paths, products), populate routing variables, create workspace artifacts if absent, and check for partial work to resume.

**Step 1 — Detect and populate routing:**

**`{track}`:** `modernization.md` exists + `prd.md` absent → `modernization`; else → `standard`.

**`{base}`, `{kb}`, `{product_col}`:** Check `delivery.json`:
- **Present** → read `planning_model` and `product_col` field:
  - `phase-first` → shared paths
  - `product-first` → ask product, use namespaced paths
- **Absent** → detect:
  - `{track} = modernization` → shared, `{product_col} = no` — write `delivery.json` (`planning_model: phase-first`, `product_col: false`).
  - `{track} = standard`:
    - No-discovery stop if `prd.md` absent
    - Load — check `Product` column:
      - No column or single value → shared, `{product_col} = no`
      - Multiple values → ask `Delivery scope? [1] Shared path (Recommended for simple multi-product) [2] Per-product path (Recommended for isolated product lifecycles)`
        - Shared: `{product_col} = yes`
        - Per-product: ask product, namespaced, `{product_col} = yes`
  - Write `delivery.json` with `planning_model` + `product_col` — verify schema.

**Paths:** Shared: `{base} = specs/delivery/`, `{kb} = specs/system/`; Namespaced: `{base} = specs/delivery/{product}/`, `{kb} = specs/system/{product}/`.

**Workspace setup:** one-time if absent:
- `{base}/review.md` absent → create empty-tables skeleton per [references/planning-artifacts.md](references/planning-artifacts.md).
- Load `CLAUDE.md` from root (skip if absent) — hold as context, never write.

**Step 2 — Resume check:**

`delivery.json` has `status=planned` wave → Resume-or-reset (`Wave {wave} is already planned.`): Resume → skip `planned|building|built` in Phase 5; Reset → `planned` → `pending` (preserve `building|built`).

Proceed to Phase 2: Issue Processing.

## Phase 2: Issue Processing

Seed issues from discovery artifacts (prerequisites, technical debt). Present blockers. Mark selected issues for extraction.

**Step 1 — Seed issues:**

Read `issues.md` (create if absent) — skip entries already present.

1. **Both tracks** — scan `solution.md § Technical Risks`: `**Discovered Prerequisite:**` rows → `issues.md` entry (`Type: gap`, `Status: open`, `Blocker: true`, `Source: solution.md`). `solution.md` absent → skip.
2. **Modernization track** — `modernization.md ## Legacy System Analysis / ### Improvements` rows → `issues.md` (`Type: debt`, `Status: open`, `Blocker: false`, `Source: modernization.md`).
3. No discovery artifacts → skip (No-discovery stop fires Phase 3).

Present new entries before Step 2.

**Step 2 — Issues check:**

- **Blockers present** → `{N} blocker issue(s) found — planning will focus on blockers only. Continue?` — Yes: only blockers → requirements, skip PRD load Phase 3 (still ask iteration), present non-blockers for awareness only; No: stop.
- **Non-blockers only** → present — user may skip.
- **None** → Phase 3.

Selected issues: mark for extraction. Set `Resolution: pending (REQ-NNN to be assigned)`. ID assignment Phase 4 per [references/planning-artifacts.md](references/planning-artifacts.md).

Proceed to Phase 3: Requirements Loading.

## Phase 3: Requirements Loading

Load requirements from discovery artifacts (PRD or modernization doc), validate structure, and scan for iteration prerequisites.

Blocker-focus (Phase 2 Step 2) → ask iteration, skip to Phase 4 (requirements from blockers only). Otherwise:

- **Modernization** — Initial: load `modernization.md` (`## Executive Summary`, `## Requirements`, `## Constraint Checks`) + `solution.md` (continue if absent) — validate: Requirements table + Constraint Checks present (stop if missing). Iteration: scan `requirements*.md`, ask which, load `requirementsNN.md` + `solution.md`.
- **Standard** — `prd.md` absent → No-discovery stop. Initial: load `prd.md` + `solution.md` (continue if absent) — validate: Requirements table + Risks + `## Delivery Estimate` (stop if missing). Iteration: scan `requirements*.md`, ask which, load `requirementsNN.md` + `solution.md`.

**Iteration prerequisite scan:** scan `solution.md § Technical Risks` for `**Discovered Prerequisite:**` — seed new `issues.md` entries (`Type: gap`, `Blocker: true`). If found: `Discovered prerequisites: {list}. [1] Continue as additional requirements (Recommended: prerequisites absorbed as requirements) [2] Stop to re-plan`.

`{product_col}` → include all products, preserve column.

Namespaced paths → filter to product.

**Iteration validate:** `## Active` section + rows + risk annotations + dependency map present (stop if missing).

Proceed to Phase 4: Requirements Extraction.

## Phase 4: Requirements Extraction

Extract requirements from loaded documents with IDs and dependencies. Batch them into waves using topological sort. Register waves in delivery.json.

**Step 1 — Extract:**

Extract to memory (not file — embedded in wave files Phase 6).

Structure: `ID`, `Type`, `Requirement`, `Requires`, `Risks` (+ `Product` if `{product_col}`).

- **Initial** — from PRD: `Type = feature`.
  - Derive `Requires` from `## Delivery Estimate` Mermaid (REQ-IDs → titles)
  - Derive `Risks` from `## Risks` by ID
  - IDs: PRD loaded → continue from highest PRD ID (or REQ-001); blocker-focus → REQ-001
- **Iteration** — from `requirementsNN.md ## Active`:
  - `Type = feature` (PRD-sourced), `bug` (defect issues), `chore` (debt/gap)
  - Derive `Requires` from `## Active` Mermaid
  - Derive `Risks` from `## Active`
  - IDs: copy from `## Active`; blocker-focus → highest REQ-ID across wave files (or REQ-001); assign missing IDs from highest

Issue-derived rows: once REQ-NNN assigned → set `Status: addressed`, `Resolution: REQ-NNN` in `issues.md`.

**Discovered prerequisites:** `solution.md` loaded → scan `## Technical Risks` for `**Discovered Prerequisite:**` — add to `Requires` column.

**Feasibility check:** read health annotations from `solution.md ## Technical Risks` — blocked/uncertain/infeasible → surface, ask how to proceed. Deferred: `Wave` blank, exclude Step 2 batching, note in `Risks`. NFRs unachievable → flag uncertain with rationale.

**Step 2 — Batch:**

1. `{product_col}` → partition by `Product`.
2. Resolve `Requires` titles → IDs — build ID→ID graph.
   - Unmatched/cycle → escalate
   - Check direction: use `solution.md § Architecture / Interfaces`
   - Consumer uses producer's output? If requirement provides infra its "dependencies" need (transport/runtime), flag edge as possibly inverted — ask before sorting
3. **Classify edges:**
   - `contract` (consumer reads artifact/message with written-down shape per `solution.md § Contracts / Data` or schema)
   - `code` (consumer imports/modifies/copies producer code/patterns; default)
   - Present — confirm
   - Contract edges: `Requires` cell = `{title} (contract: {schema ref})` — exclude from ordering, becomes fixture task Phase 7 Step 1
4. Topological sort per partition (code edges only) — no dependencies between → one wave.
5. **Wave naming:** derive from titles (e.g. "booking") — slug `wave{NN}-{name}`.
   - NN 2-digit, sequential across runs — check `{base}/` for next NN
   - User provided names → use; else ask confirm
6. Assign wave slug — register in `delivery.json`.
   - Preserve `planned|building|built`
   - Register new as `pending`
   - Present — confirm

Proceed to Phase 5: Waves.

## Phase 5: Waves

Select which pending waves to work on, load context (system, solution, modules), and scan for deviations and gaps before feature breakdown.

**Step 1 — Order:**

Collect `status=pending` from `delivery.json` — none → stop: `All waves planned. Run neat-delivery-building.`

Existing `wave{NN}-{name}.md` with `status: draft` → Resume-or-reset (`Found partial work for wave {wave}.`): Resume → keep file, resume Phase 6 from first incomplete feature (`## Features`) or Phase 7 if all complete; Reset → delete, restart Phase 6 Step 1. Wait per wave.

Check required waves via `Requires` → IDs → assignments (code edges only). Required wave not `status=built` → `Depends on wave {wave} not yet built — continue?` — Yes: note risk (applies to session); No: drop wave, offer remaining (none → stop).

Present order — confirm which to work on.

**Step 2 — Load context:**

Warnings: `build-history.md` entries not in `system.md` → `system.md may not reflect recent builds — verify or re-run solutioning`; `system.md` absent/`mode: greenfield` → note limited context; `solution.md` has `codebase_analysis: skipped` → `Solution drafted without codebase analysis — reduced architectural context`.

| Context | File | Sections |
| --- | --- | --- |
| Knowledge base | `system.md` | All |
| Solution design | `solution.md` | All |
| Requirements | in-memory (Phase 4) | Details + risks |
| Build history | `build-history.md` | Tiered (Step 3) |
| Module map | `modules/index.md` + full files | Index-based per [../references/neat-project/delivery-rules.md](../references/neat-project/delivery-rules.md) § Module Loading; `shared-modules.md` read-only |
| Legacy module map | `modernization.md ## Legacy System Analysis / ### Module Map` (modernization only) | All — for migration/replacement/adapter tasks |

**Step 3 — Scan and review:**

**Deviation scan:** `build-history.md` tiered.
- Recent 2: all
- Older: `**Deviations:**` + `**Architecture Changes:**` only
- Surface relevant to wave

**Pre-breakdown review:**
1. `solution.md` has `{TBD: reason}` → present each: `TBD: {reason}. [1] Accept risk (note in feature acceptance_criteria) (Recommended for low-risk TBDs) [2] Defer requirement (clear Wave, exclude) [3] Block`.
2. Follow [../references/neat-project/gap-resolution.md](../references/neat-project/gap-resolution.md) § Cross-Skill Continuity to `solution.md`.
   - Surface findings, ask
   - Externally-dependent → `issues.md` as `gap`

Proceed to Phase 6: Features.

## Phase 6: Features

Break requirements into features, merge overlapping features, resolve decision points, and write wave files with features and dependencies.

**Step 1 — Breakdown:**

Name features from independent concerns across requirements — name for capability delivered (not file modified).

Slugs unique per wave — rename on collision.

**Step 2 — Merge:**

Flag pairs where one's file scope ⊂ another's — present: names, shared files, rationale.

User decides. If merged:
- Pick slug (default broader's, ask if collision)
- Step 4: `depends_on` = union excluding each other, update references
- Phase 7 Step 1: combine tasks, re-number

**Step 3 — Decisions:**

Identify decision points (exclude already in `decisions.md`) — model as nodes (`section` = `### {feature-slug}` or `decisions.md` entry, `unresolved-outcome: open`).

Follow [../references/neat-project/gap-resolution.md](../references/neat-project/gap-resolution.md) to run frontier — surface findings, ask before writing.

Unresolvable → `{open:}` callouts per gap-resolution.md (don't block).

Append to `{kb}/decisions.md` per [references/planning-artifacts.md](references/planning-artifacts.md).

Cross-scope + hard-to-reverse tech commits → write ADR (`adrs/adr-{NNN}-{slug}.md`), tag `decisions.md` entry with `ADR:` (never replaces entry).

**Step 4 — Write & validate:**

Write `{base}/wave{NN}-{name}.md`: frontmatter `status: draft`; `## Requirements` table (ID, Type, Requirement, Requires, Risks, + Product if `{product_col}`); `## Features` (`### {feature-slug}` — Description, Acceptance Criteria, Depends On); embed Phase 5 Step 3 accepted-risk notes in owning feature's Acceptance Criteria.

Check: no circular `depends_on`, all `depends_on` exist in wave. Follow schema [references/planning-artifacts.md](references/planning-artifacts.md). Fix — escalate if unresolvable.

**Step 5 — Write plan:**

`{base}/plan.md` (`status: draft`) per [references/planning-artifacts.md](references/planning-artifacts.md).

Per wave (build order):
- Slug + name
- Requirement IDs + titles
- Features (name, purpose, `depends_on` — one line)
- Decision titles from this run (with links)
- Risks + open issues

~30-60 lines/run (cross-wave overview, not copy).

Proceed to Phase 7: Tasks.

## Phase 7: Tasks

Break each feature into typed tasks with inputs/outputs, assign step numbers respecting dependencies, write tasks to wave file, and run mechanical and subagent reviews.

**Before breakdown:** requirements mention new functionality → check if exists — Follow [../references/neat-project/knowledge-protocol.md](../references/neat-project/knowledge-protocol.md) § Main Session Usage.

**Step 1 — Break down:** per feature (cross-wave task ref → escalate):

1. Identify from acceptance criteria, architecture, domain rules, codebase map — one task/unit.
   
   **Modernization — file categories:**
   - Working file (new repo, created/modified, repo-relative path)
   - Legacy reference (legacy source for context, prefixed `{legacy_path}/` from `modernization.md` frontmatter, read-only, never in `outputs`)
   - Plan output (produced earlier in wave, consumed not written)
   - Prefix legacy in `inputs` with `{legacy_path}/`
2. `type` — read `system.md § Foundation` (`Infra`, `Delivery`, `Environments`; defaults: `cli`, `manual`, `single`).
   - Classify per [../references/neat-project/delivery-rules.md](../references/neat-project/delivery-rules.md) § Task Types § Classification
3. Split tasks mixing types.
4. **Test tasks:** multi-part requirements → confirm test targets each behavior before spec.
5. **Fixture tasks:** per `contract` edge into wave (Phase 4 Step 2) → add `implementation` task writing fixture under test tree per schema.
   - `acceptance_criteria` checks fixture vs schema, tests consume
6. **End-to-end check:** last wave + any `contract` edge → add task running real producer → consumer chain (automated or `configuration` if manual only).
   - `acceptance_criteria` names each edge

**Step 2 — Assign steps:**

Within-feature numbering — `depends_on` handles cross-feature.
- Producer before consumer
- Independent → share step
- Lowest valid

Two tasks same output → sequence (creates → modifies) — second declares first's output as input.

Task needs mid-execution output from other feature → escalate.

**Step 3 — Write:**

Append `**Tasks:**` to `{base}/wave{NN}-{name}.md § Features / ### {feature-slug}` per [references/planning-artifacts.md](references/planning-artifacts.md).

Each task:
- One type (no hidden dependencies beyond `inputs`)
- Bounded (specific modules/components)
- Observable done (return/file/config)
- Single concern (no part requires another first)

**Step 4 — Review:**

Mechanical checks:
- Contract completeness: `inputs` referencing earlier step has matching `outputs`.
- Scope conflict: no concurrent tasks write same function/class.
- Cross-wave contracts: `inputs` naming other wave matches that wave's `outputs`/symbol (`## Features`).
- Parallel file overlap: compare wave's `files` with planned waves having no `code` path — shared → make edge `code` (re-run Phase 4 Step 2) or move to one wave.
- **Within-wave file overlap:** per feature pair with no `depends_on`, collect `files` — overlap → `Features {A} and {B} both modify {files}. Cannot build parallel. [1] Add depends_on (serialize) (Recommended: preserves single wave, respects dependency) [2] Split waves [3] Accept (sequential)` — [1] add `depends_on`, update steps; [2] new wave file, move feature + requirements, update `delivery.json`; [3] continue.

Escalate if fails.

Spawn subagent per [references/task-review.md](references/task-review.md) — `{wave-file-path}` = `{base}/wave{NN}-{name}.md`, `{knowledge-path}` = `system.md`.

Problem → escalate.

**Step 5 — Finalize plan:**

Task review passes → update `plan.md` if Phase 7 changed features — set `status: final`.

Proceed to Phase 8: Wrap-up.

## Phase 8: Wrap-up

Check for open escalations and callouts, finalize wave status, and decide whether to continue to next wave or stop for retro.

**Step 1 — Escalation check:** confirm no open escalations, all decisions in `{kb}/decisions.md`.

**Step 2 — Callout review:** grep `{base}/wave{NN}-{name}.md` for `{open:` — list, confirm carry forward.

Rejected → resolve (user supplies answer) or defer (remove from `## Requirements`).

Don't finalize until resolved/deferred.

All requirements deferred → remove wave from `delivery.json` before Step 3.

**Step 3 — Finalize:** knowledge-protocol used Phase 7 → Follow [../references/neat-project/knowledge-protocol.md](../references/neat-project/knowledge-protocol.md) § Batch Summary (`{checkpoint}` = `Planning complete for wave {wave}`). Count tasks — present. Wave removed Step 2 → stop. Else set `status=planned` in `delivery.json` + wave frontmatter. Pending waves remain → ask: `Wave {wave} planned. [1] Continue to next wave (Recommended: more waves pending) [2] Stop`; no pending waves → Phase 9. Continuing → Phase 5 Step 1; else Phase 9.

## Phase 9: Retro

Reflect on the planning process and generate retro output for skill improvement.

Follow [../references/neat-project/retro-template.md](../references/neat-project/retro-template.md) § With Conventions Capture — `{skill-name}` = `neat-delivery-planning`, `{title-suffix}` = `· wave {wave}`, `{step2-conditional-suffix}` = `— run once at end of full session, after all waves, not per wave`.

Done.
